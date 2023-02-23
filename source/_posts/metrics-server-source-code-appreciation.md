---
title: metrics-server源码赏析
date: 2023/02/20
updated: 2023/02/20
type:
comments:
description: 最近刚好在做HPA，就想着先把kubernetes使用metrics-server的源码先看看，毕竟这也是HPA能够实现的依据，metrics-server主要也是通过aggregate api向其他组件提供集群当中的pod和node的cpu跟内存监控指标的。
keywords: metrics-server
---


## 1.前言

最近刚好在做HPA，就想着先把kubernetes使用metrics-server的源码先看看，毕竟这也是HPA能够实现的依据，metrics-server主要也是通过aggregate api向其他组件提供集群当中的pod和node的cpu跟内存监控指标的。



## 2.数据结构

抓取接口是通过

```go
// https://github.com/kubernetes-sigs/metrics-server/blob/83b2e01f9825849ae5f562e47aa1a4178b5d06e5/pkg/scraper/interface.go#L24
type Scraper interface {
	Scrape(ctx context.Context) *storage.MetricsBatch
}
```

从接口我们可以看到返回的是，storage.MetricsBatch，我们可以从源码当中的mock数据窥探出数据的大概模样

```go
// https://github.com/kubernetes-sigs/metrics-server/blob/83b2e01f9825849ae5f562e47aa1a4178b5d06e5/pkg/server/server_test.go#L47
BeforeEach(func() {
		resolution = 60 * time.Second
		scraper = &scraperMock{
			result: &storage.MetricsBatch{
				Nodes: map[string]storage.MetricsPoint{
					"node1": {
						Timestamp:         time.Now(),
						CumulativeCpuUsed: 0,
						MemoryUsage:       0,
					},
				},
			},
		}
		store = &storageMock{}
		server = NewServer(nil, nil, nil, store, scraper, resolution)
	})
```



## 3.启动流程

![整体流程图](/static/image/notes/2023/02/20/整体流程图.png)

`main`函数就不画出来了，跟其他kubernetes系列项目一样，都是使用cobra来启动的，然后导向到NewMetricsServerCommand

```go
// NewMetricsServerCommand provides a CLI handler for the metrics server entrypoint
func NewMetricsServerCommand(stopCh <-chan struct{}) *cobra.Command {
   opts := options.NewOptions()
   cmd := &cobra.Command{
      Short: "Launch metrics-server",
      Long:  "Launch metrics-server",
      RunE: func(c *cobra.Command, args []string) error {
         if err := runCommand(opts, stopCh); err != nil {
            return err
         }
         return nil
      },
   }
   fs := cmd.Flags()
   nfs := opts.Flags()
   for _, f := range nfs.FlagSets {
      fs.AddFlagSet(f)
   }
   local := flag.NewFlagSet(os.Args[0], flag.ExitOnError)
   logs.AddGoFlags(local)
   nfs.FlagSet("logging").AddGoFlagSet(local)

   usageFmt := "Usage:\n  %s\n"
   cols, _, _ := term.TerminalSize(cmd.OutOrStdout())
   cmd.SetUsageFunc(func(cmd *cobra.Command) error {
      fmt.Fprintf(cmd.OutOrStderr(), usageFmt, cmd.UseLine())
      cliflag.PrintSections(cmd.OutOrStderr(), nfs, cols)
      return nil
   })
   cmd.SetHelpFunc(func(cmd *cobra.Command, args []string) {
      fmt.Fprintf(cmd.OutOrStdout(), "%s\n\n"+usageFmt, cmd.Long, cmd.UseLine())
      cliflag.PrintSections(cmd.OutOrStdout(), nfs, cols)
   })
   fs.AddGoFlagSet(local)
   return cmd
}
```

真正的运行都是包含在`runCommand`当中，其他就是关于参数的解析校验类的。`runCommand`当中主要就是三步

1. 根据配置文件、启动参数、默认参数生成启动的配置信息
2. 根据配置信息完成启动的一些对象实例化
3. 真正启动服务

```go
func runCommand(o *options.Options, stopCh <-chan struct{}) error {
   if o.ShowVersion {
      fmt.Println(version.Get().GitVersion)
      os.Exit(0)
   }

   errors := o.Validate()
   if len(errors) > 0 {
      return errors[0]
   }
   // 生成metric-server的server端配置
   config, err := o.ServerConfig()

   if err != nil {
      return err
   }

   // 完成配置注入
   s, err := config.Complete()

   if err != nil {
      return err
   }
   // 真正启动服务
   return s.RunUntil(stopCh)
}
```



### 3.1 ServerConfig

`ServerConfig` 就是生成配置文件的，其中有个Client端的配置生成，重点主要看两个

- APIServer的配置生成
- Client端的配置生成

```go
func (o Options) ServerConfig() (*server.Config, error) {
   apiserver, err := o.ApiserverConfig()
   if err != nil {
      return nil, err
   }
   restConfig, err := o.restConfig()
   if err != nil {
      return nil, err
   }
   return &server.Config{
      Apiserver: apiserver,
      Rest:      restConfig,
      // 生成Client端的配置
      Kubelet:          o.KubeletClient.Config(restConfig),
      MetricResolution: o.MetricResolution,
      ScrapeTimeout:    o.KubeletClient.KubeletRequestTimeout,
      NodeSelector:     o.KubeletClient.NodeSelector,
   }, nil
}
```

**APIServer的配置生成**

```go
func (o Options) ApiserverConfig() (*genericapiserver.Config, error) {
   // 检查证书是否可以读取，如果不可以则尝试生成自签名证书
   if err := o.SecureServing.MaybeDefaultWithSelfSignedCerts("localhost", nil, []net.IP{net.ParseIP("127.0.0.1")}); err != nil {
      return nil, fmt.Errorf("error creating self-signed certificates: %v", err)
   }

   serverConfig := genericapiserver.NewConfig(api.Codecs)
   if err := o.SecureServing.ApplyTo(&serverConfig.SecureServing, &serverConfig.LoopbackClientConfig); err != nil {
      return nil, err
   }

   if !o.DisableAuthForTesting {
      if err := o.Authentication.ApplyTo(&serverConfig.Authentication, serverConfig.SecureServing, nil); err != nil {
         return nil, err
      }
      if err := o.Authorization.ApplyTo(&serverConfig.Authorization); err != nil {
         return nil, err
      }
   }

   if err := o.Audit.ApplyTo(serverConfig); err != nil {
      return nil, err
   }

   versionGet := version.Get()
   serverConfig.Version = &versionGet
   // enable OpenAPI schemas
   // 暴露OpenAPI断点
   serverConfig.OpenAPIConfig = genericapiserver.DefaultOpenAPIConfig(generatedopenapi.GetOpenAPIDefinitions, openapinamer.NewDefinitionNamer(api.Scheme))
   serverConfig.OpenAPIConfig.Info.Title = "Kubernetes metrics-server"
   serverConfig.OpenAPIConfig.Info.Version = strings.Split(serverConfig.Version.String(), "-")[0] // TODO(directxman12): remove this once autosetting this doesn't require security definitions

   return serverConfig, nil
}
```

**Client端的配置生成**

关于`metrics-server`访问`node`的优先级配置，就在`Client`端的配置里面了。

```go
// Config 生成metric-server的client端配置
func (o KubeletClientOptions) Config(restConfig *rest.Config) *client.KubeletClientConfig {
   config := &client.KubeletClientConfig{
      Scheme:      "https",
      DefaultPort: o.KubeletPort,
      // 节点访问优先级
      AddressTypePriority: o.addressResolverConfig(),
      UseNodeStatusPort:   o.KubeletUseNodeStatusPort,
      Client:              *rest.CopyConfig(restConfig),
   }
   if o.DeprecatedCompletelyInsecureKubelet {
      config.Scheme = "http"
      config.Client = *rest.AnonymousClientConfig(&config.Client) // don't use auth to avoid leaking auth details to insecure endpoints
      config.Client.TLSClientConfig = rest.TLSClientConfig{}      // empty TLS config --> no TLS
   }
   if o.InsecureKubeletTLS {
      config.Client.TLSClientConfig.Insecure = true
      config.Client.TLSClientConfig.CAData = nil
      config.Client.TLSClientConfig.CAFile = ""
   }
   if len(o.KubeletCAFile) > 0 {
      config.Client.TLSClientConfig.CAFile = o.KubeletCAFile
      config.Client.TLSClientConfig.CAData = nil
   }
   if len(o.KubeletClientCertFile) > 0 {
      config.Client.TLSClientConfig.CertFile = o.KubeletClientCertFile
      config.Client.TLSClientConfig.CertData = nil
   }
   if len(o.KubeletClientKeyFile) > 0 {
      config.Client.TLSClientConfig.KeyFile = o.KubeletClientKeyFile
      config.Client.TLSClientConfig.KeyData = nil
   }
   return config
}

func (o KubeletClientOptions) addressResolverConfig() []corev1.NodeAddressType {
   // 设置访问node的ip优先级（node当中保存着各种address，其中包含InternalIP、ExternalIP等）
   addrPriority := make([]corev1.NodeAddressType, len(o.KubeletPreferredAddressTypes))
   for i, addrType := range o.KubeletPreferredAddressTypes {
      addrPriority[i] = corev1.NodeAddressType(addrType)
   }
   return addrPriority
}
```

这个参数解析可以在 cmd/metrics-server/app/options/kubelet_client.go#AddFlags当中看到

```go
// 优先使用 InternalIP 来访问 kubelet，这样可以避免节点名称没有 DNS 解析记录时，通过节点名称调用节点 kubelet API 失败的情况
fs.StringSliceVar(&o.KubeletPreferredAddressTypes, "kubelet-preferred-address-types", o.KubeletPreferredAddressTypes, "The priority of node address types to use when determining which address to use to connect to a particular node")
```



### 3.2 config.Complete()

`config.Complete()` 是根据配置信息去实例化了一些对象，其中包括：

- Pod Informer
- Node Informer
- 指标抓取器
- 指标存储器（这里的存储器只是一个内存型的简单存储，只保留最近的一次指标，其他历史指标不做持久化）
- 还有kubernetes aggregate api 最常见的`genericServer`

```go
func (c Config) Complete() (*server, error) {
   var labelRequirement []labels.Requirement

   podInformerFactory, err := runningPodMetadataInformer(c.Rest)
   if err != nil {
      return nil, err
   }
   // 生成Pod的Informer
   podInformer := podInformerFactory.ForResource(corev1.SchemeGroupVersion.WithResource("pods"))
   informer, err := informerFactory(c.Rest)
   if err != nil {
      return nil, err
   }
   kubeletClient, err := resource.NewForConfig(c.Kubelet)
   if err != nil {
      return nil, fmt.Errorf("unable to construct a client to connect to the kubelets: %v", err)
   }
   // 生成Node的Informer
   nodes := informer.Core().V1().Nodes()
   ns := strings.TrimSpace(c.NodeSelector)
   if ns != "" {
      labelRequirement, err = labels.ParseToRequirements(ns)
      if err != nil {
         return nil, err
      }
   }
   // 指标抓取器
   scrape := scraper.NewScraper(nodes.Lister(), kubeletClient, c.ScrapeTimeout, labelRequirement)

   // Disable default metrics handler and create custom one
   c.Apiserver.EnableMetrics = false
   metricsHandler, err := c.metricsHandler()
   if err != nil {
      return nil, err
   }
   genericServer, err := c.Apiserver.Complete(nil).New("metrics-server", genericapiserver.NewEmptyDelegate())
   if err != nil {
      return nil, err
   }
   genericServer.Handler.NonGoRestfulMux.HandleFunc("/metrics", metricsHandler)

   store := storage.NewStorage(c.MetricResolution)
   if err := api.Install(store, podInformer.Lister(), nodes.Lister(), genericServer, labelRequirement); err != nil {
      return nil, err
   }

   s := NewServer(
      nodes.Informer(),
      podInformer.Informer(),
      genericServer,
      store,
      scrape,
      c.MetricResolution,
   )
   err = s.RegisterProbes(podInformerFactory)
   if err != nil {
      return nil, err
   }
   return s, nil
}
```



### 3.3 RunUntil

最终这里就是启动Node Informer、Pod Informer、指标抓取器、以及服务的启动（kubectl top node接口等等）

```go
// RunUntil starts background scraping goroutine and runs apiserver serving metrics.
// 注：GenericAPIServer主要处理apiregistration.k8s.io组下的APIService资源请求
// 如果是Metrics—Server的请求，那么也会转发到对应的服务，这里最终的一步就是启动Metrics—Server的GenericAPIServer
func (s *server) RunUntil(stopCh <-chan struct{}) error {
   ctx, cancel := context.WithCancel(context.Background())
   defer cancel()

   // Start informers
   go s.nodes.Run(stopCh)
   go s.pods.Run(stopCh)

   // Ensure cache is up to date
   ok := cache.WaitForCacheSync(stopCh, s.nodes.HasSynced)
   if !ok {
      return nil
   }
   ok = cache.WaitForCacheSync(stopCh, s.pods.HasSynced)
   if !ok {
      return nil
   }

   // Start serving API and scrape loop
   go s.runScrape(ctx)
   return s.GenericAPIServer.PrepareRun().Run(stopCh)
}
```



## 4. 指标抓取

指标抓取从上面我们可以看到是调用了`runScrape`

```go
func (s *server) runScrape(ctx context.Context) {
   ticker := time.NewTicker(s.resolution)
   defer ticker.Stop()
   s.tick(ctx, time.Now())

   for {
      select {
      case startTime := <-ticker.C:
         s.tick(ctx, startTime)
      case <-ctx.Done():
         return
      }
   }
}
```

而 `runScrape` 里面其实是调用了一个定时任务，其中启动了：

- s.scraper.Scrape(ctx) 进行指标抓取
- s.storage.Store(data) 对指标进行存储

```go
// tick 是指标抓取和存储的定时任务，通过s.scraper.Scrape(ctx)抓取，s.storage.Store(data)存储
func (s *server) tick(ctx context.Context, startTime time.Time) {
   s.tickStatusMux.Lock()
   s.tickLastStart = startTime
   s.tickStatusMux.Unlock()

   ctx, cancelTimeout := context.WithTimeout(ctx, s.resolution)
   defer cancelTimeout()

   klog.V(6).InfoS("Scraping metrics")
   data := s.scraper.Scrape(ctx)

   klog.V(6).InfoS("Storing metrics")
   s.storage.Store(data)

   collectTime := time.Since(startTime)
   tickDuration.Observe(float64(collectTime) / float64(time.Second))
   klog.V(6).InfoS("Scraping cycle complete")
}
```



### 4.1 s.scraper.Scrape

核心就是对node进行for，然后启动协程去抓取，可以看到每个协程会随机进行休眠，然后调用 collectNode 对接点进行抓取

```go
func (c *scraper) Scrape(baseCtx context.Context) *storage.MetricsBatch {
   nodes, err := c.nodeLister.List(c.labelSelector)
   if err != nil {
      // report the error and continue on in case of partial results
      klog.ErrorS(err, "Failed to list nodes")
   }
   klog.V(1).InfoS("Scraping metrics from nodes", "nodes", klog.KObjSlice(nodes), "nodeCount", len(nodes), "nodeSelector", c.labelSelector)

   responseChannel := make(chan *storage.MetricsBatch, len(nodes))
   defer close(responseChannel)

   startTime := myClock.Now()

   // TODO(serathius): re-evaluate this code -- do we really need to stagger fetches like this?
   delayMs := delayPerSourceMs * len(nodes)
   if delayMs > maxDelayMs {
      delayMs = maxDelayMs
   }

   for _, node := range nodes {
      go func(node *corev1.Node) {
         // Prevents network congestion.
         // 每个协程随机sleep一段时间，防止几个协程同事发起请求而造成网络堵塞
         sleepDuration := time.Duration(rand.Intn(delayMs)) * time.Millisecond
         time.Sleep(sleepDuration)
         // make the timeout a bit shorter to account for staggering, so we still preserve
         // the overall timeout
         ctx, cancelTimeout := context.WithTimeout(baseCtx, c.scrapeTimeout)
         defer cancelTimeout()
         klog.V(2).InfoS("Scraping node", "node", klog.KObj(node))
         m, err := c.collectNode(ctx, node)
         if err != nil {
            if errors.Is(err, context.DeadlineExceeded) {
               klog.ErrorS(err, "Failed to scrape node, timeout to access kubelet", "node", klog.KObj(node), "timeout", c.scrapeTimeout)
            } else {
               klog.ErrorS(err, "Failed to scrape node", "node", klog.KObj(node))
            }
         }
         responseChannel <- m
      }(node)
   }
	
   res := &storage.MetricsBatch{
      Nodes: map[string]storage.MetricsPoint{},
      Pods:  map[apitypes.NamespacedName]storage.PodMetricsPoint{},
   }
	 // 这里是对指标重新分类筛选，Node纬度、Pod纬度	
   for range nodes {
      srcBatch := <-responseChannel
      if srcBatch == nil {
         continue
      }
      for nodeName, nodeMetricsPoint := range srcBatch.Nodes {
         if _, nodeFind := res.Nodes[nodeName]; nodeFind {
            klog.ErrorS(nil, "Got duplicate node point", "node", klog.KRef("", nodeName))
            continue
         }
         res.Nodes[nodeName] = nodeMetricsPoint
      }
      for podRef, podMetricsPoint := range srcBatch.Pods {
         if _, podFind := res.Pods[podRef]; podFind {
            klog.ErrorS(nil, "Got duplicate pod point", "pod", klog.KRef(podRef.Namespace, podRef.Name))
            continue
         }
         res.Pods[podRef] = podMetricsPoint
      }
   }

   klog.V(1).InfoS("Scrape finished", "duration", myClock.Since(startTime), "nodeCount", len(res.Nodes), "podCount", len(res.Pods))
   return res
}
```

collectNode 实际上就是调用kubeletClient的GetMetrics

```go
// 实际上就是调用GetMetrics接口
func (c *scraper) collectNode(ctx context.Context, node *corev1.Node) (*storage.MetricsBatch, error) {
   startTime := myClock.Now()
   defer func() {
      requestDuration.WithLabelValues(node.Name).Observe(float64(myClock.Since(startTime)) / float64(time.Second))
      lastRequestTime.WithLabelValues(node.Name).Set(float64(myClock.Now().Unix()))
   }()
   ms, err := c.kubeletClient.GetMetrics(ctx, node)

   if err != nil {
      requestTotal.WithLabelValues("false").Inc()
      return nil, err
   }
   requestTotal.WithLabelValues("true").Inc()
   return ms, nil
}

// GetMetrics implements client.KubeletMetricsGetter
func (kc *kubeletClient) GetMetrics(ctx context.Context, node *corev1.Node) (*storage.MetricsBatch, error) {
	port := kc.defaultPort
	nodeStatusPort := int(node.Status.DaemonEndpoints.KubeletEndpoint.Port)
	if kc.useNodeStatusPort && nodeStatusPort != 0 {
		port = nodeStatusPort
	}
	addr, err := kc.addrResolver.NodeAddress(node)
	if err != nil {
		return nil, err
	}
	url := url.URL{
		Scheme: kc.scheme,
		Host:   net.JoinHostPort(addr, strconv.Itoa(port)),
		Path:   "/metrics/resource",
	}
	return kc.getMetrics(ctx, url.String(), node.Name)
}

//nolint:staticcheck // to disable SA6002 (argument should be pointer-like to avoid allocations)
func (kc *kubeletClient) getMetrics(ctx context.Context, url, nodeName string) (*storage.MetricsBatch, error) {
	req, err := http.NewRequest("GET", url, nil)
	if err != nil {
		return nil, err
	}
	requestTime := time.Now()
	response, err := kc.client.Do(req.WithContext(ctx))
	if err != nil {
		return nil, err
	}
	defer response.Body.Close()
	if response.StatusCode != http.StatusOK {
		return nil, fmt.Errorf("request failed, status: %q", response.Status)
	}
	b := kc.buffers.Get().([]byte)
	buf := bytes.NewBuffer(b)
	buf.Reset()
	_, err = io.Copy(buf, response.Body)
	if err != nil {
		kc.buffers.Put(b)
		return nil, fmt.Errorf("failed to read response body - %v", err)
	}
	b = buf.Bytes()
	ms, err := decodeBatch(b, requestTime, nodeName)
	kc.buffers.Put(b)
	if err != nil {
		return nil, err
	}
	return ms, nil
}
```



### 4.2 s.storage.Store

存储是将Node、Pod区分开来存储的，因为不需要记录历史继续，所以也比较简单，基本逻辑就是遍历指标，查看是否存在，不存在就直接怼进去新的，存在的话就比较一下指标的时间，如果是新的，就覆盖原来的值。

```go
func (s *storage) Store(batch *MetricsBatch) {
   s.mu.Lock()
   defer s.mu.Unlock()
   s.nodes.Store(batch)
   s.pods.Store(batch)
}
```

节点存储的数据结构，看到last和prev，其实就知道这玩意就是链表存储方式

```go
// nodeStorage stores last two node metric batches and calculates cpu & memory usage
//
// This implementation only stores metric points if they are newer than the
// points already stored and the cpuUsageOverTime function used to handle
// cumulative metrics assumes that the time window is different from 0.
type nodeStorage struct {
   // last stores node metric points from last scrape
   last map[string]MetricsPoint
   // prev stores node metric points from scrape preceding the last one.
   // Points timestamp should proceed the corresponding points from last.
   prev map[string]MetricsPoint
}

func (s *nodeStorage) Store(batch *MetricsBatch) {
	lastNodes := make(map[string]MetricsPoint, len(batch.Nodes))
	prevNodes := make(map[string]MetricsPoint, len(batch.Nodes))
	for nodeName, newPoint := range batch.Nodes {
		if _, exists := lastNodes[nodeName]; exists {
			klog.ErrorS(nil, "Got duplicate node point", "node", klog.KRef("", nodeName))
			continue
		}
		lastNodes[nodeName] = newPoint

		if lastNode, found := s.last[nodeName]; found {
			// If new point is different then one already stored
			if newPoint.Timestamp.After(lastNode.Timestamp) {
				// Move stored point to previous
				prevNodes[nodeName] = lastNode
			} else if prevPoint, found := s.prev[nodeName]; found {
				if prevPoint.Timestamp.Before(newPoint.Timestamp) {
					// Keep previous point
					prevNodes[nodeName] = prevPoint
				} else {
					klog.V(2).InfoS("Found new node metrics point is older than stored previous, drop previous",
						"node", nodeName,
						"previousTimestamp", prevPoint.Timestamp,
						"timestamp", newPoint.Timestamp)
				}
			}
		}
	}
	s.last = lastNodes
	s.prev = prevNodes

	// Only count last for which metrics can be returned.
  // 只计算最后可以返回的指标
	pointsStored.WithLabelValues("node").Set(float64(len(prevNodes)))
}
```

Pod的也是大同小异

```go
// nodeStorage stores last two pod metric batches and calculates cpu & memory usage
//
// This implementation only stores metric points if they are newer than the
// points already stored and the cpuUsageOverTime function used to handle
// cumulative metrics assumes that the time window is different from 0.
type podStorage struct {
	// last stores pod metric points from last scrape
	last map[apitypes.NamespacedName]PodMetricsPoint
	// prev stores pod metric points from scrape preceding the last one.
	// Points timestamp should proceed the corresponding points from last and have same start time (no restart between them).
	prev map[apitypes.NamespacedName]PodMetricsPoint
	// scrape period of metrics server
	metricResolution time.Duration
}
```

可能是出于Pod的数量可能较多，还需要判断一下指标的抓取周期，也就是字段 metricResolution，使用位置是在对新的 Point 的时间戳以及起始时间的差值进行比较。

```go
newPoint.Timestamp.Sub(newPoint.StartTime) < s.metricResolution
```

```go
func (s *podStorage) Store(newPods *MetricsBatch) {
   lastPods := make(map[apitypes.NamespacedName]PodMetricsPoint, len(newPods.Pods))
   prevPods := make(map[apitypes.NamespacedName]PodMetricsPoint, len(newPods.Pods))
   var containerCount int
   for podRef, newPod := range newPods.Pods {
      podRef := apitypes.NamespacedName{Name: podRef.Name, Namespace: podRef.Namespace}
      if _, found := lastPods[podRef]; found {
         klog.ErrorS(nil, "Got duplicate pod point", "pod", klog.KRef(podRef.Namespace, podRef.Name))
         continue
      }

      newLastPod := PodMetricsPoint{Containers: make(map[string]MetricsPoint, len(newPod.Containers))}
      newPrevPod := PodMetricsPoint{Containers: make(map[string]MetricsPoint, len(newPod.Containers))}
      for containerName, newPoint := range newPod.Containers {
         if _, exists := newLastPod.Containers[containerName]; exists {
            klog.ErrorS(nil, "Got duplicate Container point", "container", containerName, "pod", klog.KRef(podRef.Namespace, podRef.Name))
            continue
         }
         newLastPod.Containers[containerName] = newPoint
         if newPoint.StartTime.Before(newPoint.Timestamp) && newPoint.Timestamp.Sub(newPoint.StartTime) < s.metricResolution && newPoint.Timestamp.Sub(newPoint.StartTime) >= freshContainerMinMetricsResolution {
            copied := newPoint
            copied.Timestamp = newPoint.StartTime
            copied.CumulativeCpuUsed = 0
            newPrevPod.Containers[containerName] = copied
         } else if lastPod, found := s.last[podRef]; found {
            // Keep previous metric point if newPoint has not restarted (new metric start time < stored timestamp)
            if lastContainer, found := lastPod.Containers[containerName]; found && newPoint.StartTime.Before(lastContainer.Timestamp) {
               // If new point is different then one already stored
               if newPoint.Timestamp.After(lastContainer.Timestamp) {
                  // Move stored point to previous
                  newPrevPod.Containers[containerName] = lastContainer
               } else if prevPod, found := s.prev[podRef]; found {
                  if prevPod.Containers[containerName].Timestamp.Before(newPoint.Timestamp) {
                     // Keep previous point
                     newPrevPod.Containers[containerName] = prevPod.Containers[containerName]
                  } else {
                     klog.V(2).InfoS("Found new containerName metrics point is older than stored previous , drop previous",
                        "containerName", containerName,
                        "pod", klog.KRef(podRef.Namespace, podRef.Name),
                        "previousTimestamp", prevPod.Containers[containerName].Timestamp,
                        "timestamp", newPoint.Timestamp)
                  }
               }
            }
         }
      }
      containerPoints := len(newPrevPod.Containers)
      if containerPoints > 0 {
         prevPods[podRef] = newPrevPod
      }
      lastPods[podRef] = newLastPod

      // Only count containers for which metrics can be returned.
      containerCount += containerPoints
   }
   s.last = lastPods
   s.prev = prevPods

   pointsStored.WithLabelValues("container").Set(float64(containerCount))
}
```