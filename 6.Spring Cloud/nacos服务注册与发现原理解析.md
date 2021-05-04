# 参考网站 #
Nacos注册中心源码分析 : [https://www.processon.com/view/link/5ea27ca15653bb6efc68eb8c](https://www.processon.com/view/link/5ea27ca15653bb6efc68eb8c)  
Nacos实战 : [https://xie.infoq.cn/article/d342a914b8754dd52f5709b43](https://xie.infoq.cn/article/d342a914b8754dd52f5709b43)  
Nacos官网 :  [https://nacos.io/zh-cn/docs/quick-start.html](https://nacos.io/zh-cn/docs/quick-start.html)  
注册中心源码 : [https://www.processon.com/view/link/5ec668cf7d9c08156c574471](https://www.processon.com/view/link/5ec668cf7d9c08156c574471)

# Nacos注册中心原理 #
## Nacos服务注册表结构 ##

	ServiceManager结构 : Map<namespace, Map<group::serivceName, Service>>  
	Service结构：Map<clusterName, Cluster>  
	Cluster结构：Set<Instance>    

![](https://static001.geekbang.org/infoq/bb/bb5f7b911382512c154890fe394a37ec.png)

## AP模式(Distro协议) ##
1. 接收到注册请求后首先判断是CP还是AP，AP模式下是临时节点，CP模式是持久化节点（持久化到磁盘文件）
2. 服务注册后将变化的事件信息放入队列（BlockingQueue）中
3. 会有后台线程不断从队列中取出事件信息，根据不同的事件，如注册节点是CHANGE事件，从注册表的map中取出服务名称对应的服务节点List<Instance>将新节点加入该List中；下线节点是DELETE事件，也就是将该节点从服务节点List移除，此处的事件类似Reactor模型的思想。
4. 更新节点不管是注册还是删除，在nacos中走的其实都是同一个方法updateIps，使用copy on write的方式，更新时首先new一个新的HashMap<服务名称，List<Instance>>，将旧注册列表中该服务名称对应的所有节点信息put到这个map中，之后所有的修改都是更新该map中的List<Instance>，最终将Cluster中的临时节点集合指针直接指向这个更新后的集合
5. 队列本身解决了多服务注册时的并发问题，无需加锁也不会发生写操作覆盖。
6. 其核心思想就是防止读写并发冲突，把原数据复制一份，操作完成后在合并回真正的注册表内存中
7. 本服务完成注册后，会将本次注册封装成一个延迟task放入nacos延迟任务类中的taskMap中，延迟时间为1s，如果有同服务名称多节点注册会将其合并为一个task，合并完成后如果更新节点数超过指定数量（默认1000），则会直接取消掉这个task的延迟时间，将其变为立即触发
8. 后台会有线程每隔100ms从这个ConcurrentHashMap中拿出所有需要执行的task进行处理，此时向集群内其他注册中心通过http同步更新数据
9. 服务节点默认5s向注册中心发送一次心跳，如果注册中心超过15s没有收到心跳就会将该服务节点标记为不健康，如果超过30s将该服务直接下线

总结：nacos的AP模式其实和eureka本身差别不大，但是一方面使用**copy on write**思想取代了eureka的三级缓存，大幅度提升了服务发现速度（从几十秒缩短到几秒），另一方面nacos采用的是多节点批量更新，解决了上万服务节点场景集中上线时，eureka各节点间频繁进行上下线同步更新通信，导致注册中心QPS飙升的问题，以此可以支持大规模服务节点的注册

## CP模式（Raft协议） ##
1. 注册时先判断当前节点是否是leader节点，如果不是leader会将本次注册请求转发到leader节点，只有leader节点才可以写数据
2. leader节点首先会将数据放入队列，之后通过队列消费将本次注册写入内存中
3. 同时nacos也会将本次注册信息写入磁盘，通过磁盘保证数据持久化
4. 主节点写入数据成功后，再将数据同步到其他从节点，此时使用CountDownLatch，构造参数为所有节点数 / 2 + 1，如果超过半数节点返回成功即通知客户端写入成功
5. RaftCore类中init方法，在初始化时会监测本机是否有已经持久化的磁盘文件，如果有会直接从其中恢复数据
6. 选主
	- 服务在等待一定随机时间后向其他节点发送投票信息，如果被超过半数以上节点选中，当前节点就会成为leader节点。
	- 此时其他节点可能还在等待随机时间。
	- leader节点会向其他节点发送心跳，告知我已经成为leader节点。其他节点停止等待，变成flower节点
	- leader节点会不断向flower节点发送心跳，flower节点会等待一定时间，如果超过等待时间没有收到leader节点的心跳，则认为leader节点挂掉进而重新选举

> Raft算法   
> 演示网址 [http://thesecretlivesofdata.com/raft/](http://thesecretlivesofdata.com/raft/)  
> 参考网站 [https://blog.csdn.net/gonghaiyu/article/details/108425747](https://blog.csdn.net/gonghaiyu/article/details/108425747)

# 源码分析 #
## 服务注册 ##
首先需要引入spring-cloud-starter-alibaba-nacos-discovery包，本文的引入的版本是2.2.1

	<dependency>
	    <groupId>com.alibaba.cloud</groupId>
	    <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
	    <version>2.2.1.RELEASE</version>
	</dependency>

根据sprin.factories配置来完成相关类的自动注册  
![](https://img2020.cnblogs.com/blog/2259594/202101/2259594-20210106215517879-1249858455.png)

我们重点来看这几个类，看名称可猜到是用来服务注册的，NacosServiceRegistryAutoConfiguration用来注册管理这几个bean

![](https://img2020.cnblogs.com/blog/2259594/202101/2259594-20210106220308566-878073783.png)

NacosServiceRegistry：完成服务注册，实现ServiceRegistry

NacosRegistration：用来注册时存储nacos服务端的相关信息

NacosAutoServiceRegistration 继承spring中的AbstractAutoServiceRegistration，AbstractAutoServiceRegistration继承ApplicationListener<WebServerInitializedEvent>，通过事件监听来发起服务注册，到时候会调用NacosServiceRegistry.register(registration)

来看具体如何注册

	/******************************************************NacosServiceRegistry******************************************************/
	public void register(Registration registration) {
	    if (StringUtils.isEmpty(registration.getServiceId())) {
	        log.warn("No service to register for nacos client...");
	    } else {
	        String serviceId = registration.getServiceId();
	        String group = this.nacosDiscoveryProperties.getGroup();
	        Instance instance = this.getNacosInstanceFromRegistration(registration);
	        try {
	            this.namingService.registerInstance(serviceId, group, instance);
	        } 
	    }
	}
	
	/******************************************************NacosNamingService******************************************************/
	public void registerInstance(String serviceName, String groupName, Instance instance) throws NacosException {
	
	    if (instance.isEphemeral()) {
	        // 添加心跳检测
	        beatReactor.addBeatInfo(NamingUtils.getGroupedName(serviceName, groupName), beatInfo);
	    }
	    // 完成服务注册
	    serverProxy.registerService(NamingUtils.getGroupedName(serviceName, groupName), groupName, instance);
	}
	
	/******************************************************NacosNamingService******************************************************/
	public void addBeatInfo(String serviceName, BeatInfo beatInfo) {
	
	    String key = buildKey(serviceName, beatInfo.getIp(), beatInfo.getPort());
	    // 发起一个心跳检测任务
	    executorService.schedule(new BeatTask(beatInfo), beatInfo.getPeriod(), TimeUnit.MILLISECONDS);
	    MetricsMonitor.getDom2BeatSizeMonitor().set(dom2Beat.size());
	}
	
	/******************************************************BeatTask******************************************************/
	class BeatTask implements Runnable {
	
	    @Override
	    public void run() {
	        if (beatInfo.isStopped()) {
	            return;
	        }
	        long nextTime = beatInfo.getPeriod();
	        try {
	            // 向nacos服务发起心跳检测
	            JSONObject result = serverProxy.sendBeat(beatInfo, BeatReactor.this.lightBeatEnabled);
	            long interval = result.getIntValue("clientBeatInterval");
	            boolean lightBeatEnabled = false;
	            if (result.containsKey(CommonParams.LIGHT_BEAT_ENABLED)) {
	                lightBeatEnabled = result.getBooleanValue(CommonParams.LIGHT_BEAT_ENABLED);
	            }
	            BeatReactor.this.lightBeatEnabled = lightBeatEnabled;
	            if (interval > 0) {
	                nextTime = interval;
	            }
	            int code = NamingResponseCode.OK;
	            if (result.containsKey(CommonParams.CODE)) {
	                code = result.getIntValue(CommonParams.CODE);
	            }
	            if (code == NamingResponseCode.RESOURCE_NOT_FOUND) {
	                // 未注册 先完成注册
	                try {
	                    serverProxy.registerService(beatInfo.getServiceName(),
	                        NamingUtils.getGroupName(beatInfo.getServiceName()), instance);
	                } 
	            }
	        } 
	        // 发起下一次心跳检测
	        executorService.schedule(new BeatTask(beatInfo), nextTime, TimeUnit.MILLISECONDS);
	    }
	}


服务提供者向nacos server发起服务注册前，先向nacos server建立起心跳检测机制，nacos server那边也有一个心跳检测，服务提供者不停的向nacos server发起心跳检测 告知自己的健康状态，nacos serve发现该服务心跳检测时间超时会发布超时事件来告知服务消费者

## 服务发现 ##
当第一次请求时候，才会去获取服务，也就是懒加载

1.在本地查找实例缓存信息，如果缓存为空，则开启定时任务请求服务端获取实例信息列表来更新缓存到本地
 
1. NacosDiscoveryClient.getInstances()

		@Override
		public List<ServiceInstance> getInstances(String serviceId) {
		    try {
		        return serviceDiscovery.getInstances(serviceId);
		    }
		    ...省略
		}

2. NacosDiscoveryClient.getInstances()

		public List<ServiceInstance> getInstances(String serviceId) throws NacosException {
		    String group = discoveryProperties.getGroup();
		    List<Instance> instances = namingService().selectInstances(serviceId, group, true);
		    return hostToServiceInstanceList(instances, serviceId);
		}

3. NacosNamingService.selectInstances()  

		public List<Instance> selectInstances(String serviceName, String groupName, List<String> clusters, boolean healthy,
		        boolean subscribe) throws NacosException {
		    ServiceInfo serviceInfo;
		    // 默认订阅服务
		    if (subscribe) {
		        serviceInfo = hostReactor.getServiceInfo(NamingUtils.getGroupedName(serviceName, groupName),
		                StringUtils.join(clusters, ","));
		    } else {
		        serviceInfo = hostReactor
		                .getServiceInfoDirectlyFromServer(NamingUtils.getGroupedName(serviceName, groupName),
		                        StringUtils.join(clusters, ","));
		    }
		    return selectInstances(serviceInfo, healthy);
		}  


4. HostReactor.getServiceInfo()

		public ServiceInfo getServiceInfo(final String serviceName, final String clusters) {
		    NAMING_LOGGER.debug("failover-mode: " + failoverReactor.isFailoverSwitch());
		    String key = ServiceInfo.getKey(serviceName, clusters);
		    if (failoverReactor.isFailoverSwitch()) {
		        return failoverReactor.getService(key);
		    }
		    // 本地缓存找
		    ServiceInfo serviceObj = getServiceInfo0(serviceName, clusters);
		    if (null == serviceObj) {
		        serviceObj = new ServiceInfo(serviceName, clusters);
		        serviceInfoMap.put(serviceObj.getKey(), serviceObj);
		        updatingMap.put(serviceName, new Object());
		        // 从服务端拉取服务信息
		        updateServiceNow(serviceName, clusters);
		        updatingMap.remove(serviceName);
		    } else if (updatingMap.containsKey(serviceName)) {
		        ...省略
		    }
		　　 //添加定时任务
		    scheduleUpdateIfAbsent(serviceName, clusters);
		    return serviceInfoMap.get(serviceObj.getKey());
		}

5. 更新服务信息 -- HostReactor.updateServiceNow()

		// 更新服务信息
		private void updateServiceNow(String serviceName, String clusters) {
		    try {
		        updateService(serviceName, clusters);
		    } catch (NacosException e) {
		        NAMING_LOGGER.error("[NA] failed to update serviceName: " + serviceName, e);
		    }
		}

6. 添加定时任务 -- HostReactor.scheduleUpdateIfAbsent()

		//如果不存在，则添加定时任务
		public void scheduleUpdateIfAbsent(String serviceName, String clusters) {
		    if (futureMap.get(ServiceInfo.getKey(serviceName, clusters)) != null) {
		        return;
		    }
		    synchronized (futureMap) {
		        if (futureMap.get(ServiceInfo.getKey(serviceName, clusters)) != null) {
		            return;
		        }
		        //添加定时任务
		        ScheduledFuture<?> future = addTask(new UpdateTask(serviceName, clusters));
		        futureMap.put(ServiceInfo.getKey(serviceName, clusters), future);
		    }
		}
	
7. 从服务端拉取服务信息 -- HostReactor.updateService()

		// 从服务端拉取服务信息
		public void updateService(String serviceName, String clusters) throws NacosException {
		    ServiceInfo oldService = getServiceInfo0(serviceName, clusters);
		    try {
		        String result = serverProxy.queryList(serviceName, clusters, pushReceiver.getUdpPort(), false);
		        if (StringUtils.isNotEmpty(result)) {
		            // 将服务端拉取到的服务信息缓存在本地
		            processServiceJson(result);
		        }
		    } finally {
		        if (oldService != null) {
		            synchronized (oldService) {
		                oldService.notifyAll();
		            }
		        }
		    }
		}

8. 查询列表 -- NamingProxy.queryList()

		//查询列表
		public String queryList(String serviceName, String clusters, int udpPort, boolean healthyOnly)
		    throws NacosException {
		    final Map<String, String> params = new HashMap<String, String>(8);
		    params.put(CommonParams.NAMESPACE_ID, namespaceId);
		    params.put(CommonParams.SERVICE_NAME, serviceName);
		    params.put("clusters", clusters);
		    params.put("udpPort", String.valueOf(udpPort));
		    params.put("clientIP", NetUtils.localIP());
		    params.put("healthyOnly", String.valueOf(healthyOnly));
		    //最终调用Http请求拉取服务器服务信息列表 
		    return reqApi(UtilAndComs.nacosUrlBase + "/instance/list", params, HttpMethod.GET);
		} 

9. Http入口 -- InstanceController.list()

		@GetMapping("/list")
		@Secured(parser = NamingResourceParser.class, action = ActionTypes.READ)
		public ObjectNode list(HttpServletRequest request) throws Exception {
		    //获取参数并校验
		    String namespaceId = WebUtils.optional(request, CommonParams.NAMESPACE_ID, Constants.DEFAULT_NAMESPACE_ID);
		    String serviceName = WebUtils.required(request, CommonParams.SERVICE_NAME);
		    NamingUtils.checkServiceNameFormat(serviceName);
		    String agent = WebUtils.getUserAgent(request);
		    String clusters = WebUtils.optional(request, "clusters", StringUtils.EMPTY);
		    String clientIP = WebUtils.optional(request, "clientIP", StringUtils.EMPTY);
		    int udpPort = Integer.parseInt(WebUtils.optional(request, "udpPort", "0"));
		    String env = WebUtils.optional(request, "env", StringUtils.EMPTY);
		    boolean isCheck = Boolean.parseBoolean(WebUtils.optional(request, "isCheck", "false"));
		    String app = WebUtils.optional(request, "app", StringUtils.EMPTY);
		    String tenant = WebUtils.optional(request, "tid", StringUtils.EMPTY);
		    boolean healthyOnly = Boolean.parseBoolean(WebUtils.optional(request, "healthyOnly", "false"));
		    // 根据命名空间id,服务名获取实例信息
		    return doSrvIpxt(namespaceId, serviceName, agent, clusters, clientIP, udpPort, env, isCheck, app, tenant,
		            healthyOnly);
		}

10. InstanceController.doSrvIpxt()
		
		public ObjectNode doSrvIpxt(String namespaceId, String serviceName, String agent, String clusters, String clientIP,
		        int udpPort, String env, boolean isCheck, String app, String tid, boolean healthyOnly) throws Exception {
		    ClientInfo clientInfo = new ClientInfo(agent);
		    ObjectNode result = JacksonUtils.createEmptyJsonNode();
		    //获取服务
		    Service service = serviceManager.getService(namespaceId, serviceName);
		    long cacheMillis = switchDomain.getDefaultCacheMillis();
		    // 尝试启用推送
		    try {
		        if (udpPort > 0 && pushService.canEnablePush(agent)) {
		            pushService.addClient(namespaceId, serviceName, clusters, agent, new InetSocketAddress(clientIP, udpPort),
		                            pushDataSource, tid, app);
		            cacheMillis = switchDomain.getPushCacheMillis(serviceName);
		        }
		    } catch (Exception e) {
		        Loggers.SRV_LOG.error("[NACOS-API] failed to added push client {}, {}:{}", clientInfo, clientIP, udpPort, e);
		        cacheMillis = switchDomain.getDefaultCacheMillis();
		    }
		    if (service == null) {
		        if (Loggers.SRV_LOG.isDebugEnabled()) {
		            Loggers.SRV_LOG.debug("no instance to serve for service: {}", serviceName);
		        }
		        result.put("name", serviceName);
		        result.put("clusters", clusters);
		        result.put("cacheMillis", cacheMillis);
		        result.replace("hosts", JacksonUtils.createEmptyArrayNode());
		        return result;
		    }
		    //检查服务是否禁用
		    checkIfDisabled(service);
		    List<Instance> srvedIPs;
		    // 获取所有永久和临时服务实例
		    srvedIPs = service.srvIPs(Arrays.asList(StringUtils.split(clusters, ",")));
		    // 选择器过滤服务
		    if (service.getSelector() != null && StringUtils.isNotBlank(clientIP)) {
		        srvedIPs = service.getSelector().select(clientIP, srvedIPs);
		    }
		    ...如果找不到服务则返回当前服务
		}

11. Service.srvIPs()


		//获取所有的永久实例和临时实例
		public List<Instance> srvIPs(List<String> clusters) {
		    if (CollectionUtils.isEmpty(clusters)) {
		        clusters = new ArrayList<>();
		        clusters.addAll(clusterMap.keySet());
		    }
		    return allIPs(clusters);
		}

		public List<Instance> allIPs() {
		    List<Instance> allInstances = new ArrayList<>();
		    allInstances.addAll(persistentInstances);
		    allInstances.addAll(ephemeralInstances);
		    return allInstances;
		}

12. 推送服务实例信息 -- PushService.addClient()

		public void addClient(String namespaceId, String serviceName, String clusters, String agent,
		        InetSocketAddress socketAddr, DataSource dataSource, String tenant, String app) {
		    // 初始化推送客户端
		    PushClient client = new PushClient(namespaceId, serviceName, clusters, agent, socketAddr, dataSource, tenant, app);
		    addClient(client);
		}

		// 添加推送目标客户端。
		public void addClient(PushClient client) {
		    // 客户端由键“ serviceName”存储，因为通知事件由serviceName更改驱动
		    String serviceKey = UtilsAndCommons.assembleFullServiceName(client.getNamespaceId(), client.getServiceName());
		    ConcurrentMap<String, PushClient> clients = clientMap.get(serviceKey);
		    // 如果获取不到推送客户端则新建推送客户端，并缓存
		    if (clients == null) {
		        clientMap.putIfAbsent(serviceKey, new ConcurrentHashMap<>(1024));
		        clients = clientMap.get(serviceKey);
		    }
		    // 刷新或者缓存
		    PushClient oldClient = clients.get(client.toString());
		    if (oldClient != null) {
		        oldClient.refresh();
		    } else {
		        PushClient res = clients.putIfAbsent(client.toString(), client);
		        if (res != null) {
		            Loggers.PUSH.warn("client: {} already associated with key {}", res.getAddrStr(), res.toString());
		        }
		        Loggers.PUSH.debug("client: {} added for serviceName: {}", client.getAddrStr(), client.getServiceName());
		    }
		}


**返回指定命名空间下内存注册表中所有的永久实例和临时实例给客户端**

