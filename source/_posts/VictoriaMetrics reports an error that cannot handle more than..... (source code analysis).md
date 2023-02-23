---
title: VictoriaMetrics报错cannot handle more than..... （源码分析）
date: 2023/02/23
updated: 2023/02/23
type:
comments:
description: 最近刚好在做HPA，就想着先把kubernetes使用metrics-server的源码先看看，毕竟这也是HPA能够实现的依据，metrics-server主要也是通过aggregate api向其他组件提供集群当中的pod和node的cpu跟内存监控指标的。
keywords: metrics-server
---

## 1.前言

之前不是在搞**VictoriaMetrics**嘛，自己在公网有两台2C4G的机器，就跑个**VictoriaMetrics**集群上去，但是我上面不止跑这玩意，结果就把**VictoriaMetrics**集群跑挂了，报了什么 cannot handle more than .... possible solutions .... increase `-insert.maxQueueDuration`, increase `-maxConcurrentInserts`, increase server capacity。虽然不知道就是队列满了，要等其他请求完成，但是作为一个开发，还是得本着知其然知其所以然，所以继续fake source code吧。

## 2.定位核心代码

直接拷贝源码搜索这段相关文本，很快就可以定位到相关位置了 `/lib/writeconcurrencylimiter/concurrencylimiter.go#Do`

可以看到这段报错其实就在一个select..case...当中，上面相关的注释也说明了：所有任务都在忙中，等待maxQueueDuration时长，如果在maxQueueDuration时间内没有获取到队列可以处理资源，那么就丢弃请求并返回错误信息。

```go
// Do calls f with the limited concurrency.
func Do(f func() error) error {
	// Limit the number of conurrent f calls in order to prevent from excess
	// memory usage and CPU thrashing.
	select {
	// 往通道写入一个数据，标识着开始处理一个请求
	case ch <- struct{}{}:
		// 阻塞进入等待请求完成
		err := f()
		// 请求处理完之后释放通道内的一个数据，释放出来之后才可以接收多一个任务
		<-ch
		return err
	default:
	}

	// All the workers are busy.
	// Sleep for up to *maxQueueDuration.
	// 达到处理的最大数，需等其他Do(f func() error)释放资源
	concurrencyLimitReached.Inc()
	// 获取time.Timer,设置超时等待时间为maxQueueDuration
	t := timerpool.Get(*maxQueueDuration)
	select {
	// 在maxQueueDuration有其他请求释放了资源
	case ch <- struct{}{}:
		// 回收time.Timer
		timerpool.Put(t)
		// 继续处理任务
		err := f()
		<-ch
		return err
	// 超时
	case <-t.C:
		// 回收time.Timer
		timerpool.Put(t)
		// pacelimiter（步长限制器）
		concurrencyLimitTimeout.Inc()
		// 超时信息提示
		return &httpserver.ErrorWithStatusCode{
			Err: fmt.Errorf("cannot handle more than %d concurrent inserts during %s; possible solutions: "+
				"increase `-insert.maxQueueDuration`, increase `-maxConcurrentInserts`, increase server capacity", *maxConcurrentInserts, *maxQueueDuration),
			StatusCode: http.StatusServiceUnavailable,
		}
	}
}
```

上面的这个方法是在`concurrencylimiter.go`当中的，这个看名字跟看源码，也没具体业务，所以我们看下有什么调用了这个func

<img src="/static/image/notes/2023/02/23/Run调用图.png" alt="image-20230222141145560" style="zoom:50%;" />

可以看到非常多的方法都调用了，我们可以看下限速器是用那个value，看看这个是限速什么的并发。

```go
concurrencyLimitReached = metrics.NewCounter(`vm_concurrent_insert_limit_reached_total`)
concurrencyLimitTimeout = metrics.NewCounter(`vm_concurrent_insert_limit_timeout_total`)
```

可以看到这个是给vminsert的使用的，`VictoriaMetrics`当中，`vminsert`负责接收`vmagent`的流量并将其转发到vmstorage当中，在`vmstorage`出错、处理过慢、卡死的情况下，有可能会导致无法转发流量，进而造成`vminsert`的CPU和内存飙升，为了防止`vminsert`过载导致组件故障，`vminsert`引入了限速器，在牺牲一部分数据的情况下保证了系统的正常运行。

但是！上面虽然那么多会调用这个方法，实际上我只有Prometheus的RemoteWrite

<img src="/static/image/notes/2023/02/23/Prometheus.png" alt="image-20230222160739320" style="zoom:50%;" />

那么问题肯定是出现在`vmstorage`上面了。

## 3. 链路分析

<img src="/static/image/notes/2023/02/23/调用链.png" alt="image-20230222222254285" style="zoom:50%;" />

上面是promeremotewrite的调用链，到Storage.AddRows时，有一处并发控制，也就是`addRowsConcurrencyCh`

而这个变量的初始化我们可以看看

```go
var (
   // Limit the concurrency for data ingestion to GOMAXPROCS, since this operation
   // is CPU bound, so there is no sense in running more than GOMAXPROCS concurrent
   // goroutines on data ingestion path.
   addRowsConcurrencyCh = make(chan struct{}, cgroup.AvailableCPUs())
   addRowsTimeout       = 30 * time.Second
)
```

可以看到这里是根据CPU的核数来限制并发的，假设有3个核心，则写入操作最多3个并发。

```go
// AddRows adds the given mrs to s.
func (s *Storage) AddRows(mrs []MetricRow, precisionBits uint8) error {
   if len(mrs) == 0 {
      return nil
   }

   // Limit the number of concurrent goroutines that may add rows to the storage.
   // This should prevent from out of memory errors and CPU thrashing when too many
   // goroutines call AddRows.
   select {
   // 如果可以写入addRowsConcurrencyCh，证明队列此时还没满，跳过当前的select
   case addRowsConcurrencyCh <- struct{}{}:
   // 无法写入addRowsConcurrencyCh，此时并发数已经超过CPU的核数了
   default:
      // Sleep for a while until giving up
      atomic.AddUint64(&s.addRowsConcurrencyLimitReached, 1)
      // 这里获取到一个time.Timer来控制超时
      t := timerpool.Get(addRowsTimeout)

      // Prioritize data ingestion over concurrent searches.
      // 依然是pacelimiter来进行计数
      storagepacelimiter.Search.Inc()

      select {
      // 如果在addRowsTimeout时间内，入队成功
      case addRowsConcurrencyCh <- struct{}{}:
         // 归还time.Timer对象
         timerpool.Put(t)
         // 计数器减一，然后会一直减1，直到等待数量为0了，调用cond.Broadcast()来通知select工作，可以进去Dec()看看cond.Broadcast()
         storagepacelimiter.Search.Dec()
         // 等待了addRowsTimeout时间后仍然没有CPU资源，只能报错
      case <-t.C:
         timerpool.Put(t)
         storagepacelimiter.Search.Dec()
         atomic.AddUint64(&s.addRowsConcurrencyLimitTimeout, 1)
         atomic.AddUint64(&s.addRowsConcurrencyDroppedRows, uint64(len(mrs)))
         return fmt.Errorf("cannot add %d rows to storage in %s, since it is overloaded with %d concurrent writers; add more CPUs or reduce load",
            len(mrs), addRowsTimeout, cap(addRowsConcurrencyCh))
      }
   }

   // Add rows to the storage in blocks with limited size in order to reduce memory usage.
   // 这一步才开始真正的数据插入逻辑
   var firstErr error
   ic := getMetricRowsInsertCtx()
   maxBlockLen := len(ic.rrs)
   for len(mrs) > 0 {
      mrsBlock := mrs
      if len(mrs) > maxBlockLen {
         mrsBlock = mrs[:maxBlockLen]
         mrs = mrs[maxBlockLen:]
      } else {
         mrs = nil
      }
      if err := s.add(ic.rrs, ic.tmpMrs, mrsBlock, precisionBits); err != nil {
         if firstErr == nil {
            firstErr = err
         }
         continue
      }
      atomic.AddUint64(&rowsAddedTotal, uint64(len(mrsBlock)))
   }
   putMetricRowsInsertCtx(ic)

   <-addRowsConcurrencyCh

   return firstErr
}
```



## 4. 总结

到上面那一步就可以发现，这个插入协程数跟cpu核心数的关系比较大，结合我的服务实际情况，因为在安装VictoriaMetrics时，使用的是operator部署，给他分配的cpu限制到了非常小的核心数，这样就会导致我的VictoriaMetrics会因为在太多的Prometheus remotewrite而超时丢弃信息，进而给出报错cannot handle more than .... possible solutions .... increase .....