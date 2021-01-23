## ZooKeeper ##
ZooKeeper是一个开源的分布式服务协调组件,是Google Chubby的开源实现.是一个高性能的分布式数据一致性解决方案.他将那些复杂的、容易出错的分布式一致性服务封装起来，构成该一个高效可靠的原语集，并提供一系列简单易用的接口给用户使用。

### 参考网站 ###
[https://www.cnblogs.com/yuyijq/p/3391945.html](https://www.cnblogs.com/yuyijq/p/3391945.html)  
[https://draveness.me/zookeeper-chubby](https://draveness.me/zookeeper-chubby)  
[https://juejin.im/post/5b03d58a6fb9a07a9e4d8f01](https://juejin.im/post/5b03d58a6fb9a07a9e4d8f01)  
[https://www.cnblogs.com/wlwl/p/10715065.html](https://www.cnblogs.com/wlwl/p/10715065.html)

### ZooKeeper基本概念 ###
#### 角色 ####
ZooKeeper集群中的所有机器通过一个Leader 选举过程来选定一台称为 “Leader” 的机器，Leader 既可以为客户端提供写服务又能提供读服务。除了 Leader 外，Follower 和 Observer 都只能提供读服务。Follower 和 Observer 唯一的区别在于 Observer 机器不参与 Leader 的选举过程，也不参与写操作的“过半写成功”策略，因此 Observer 机器可以在不影响写性能的情况下提升集群的读性能。  
![](https://i.imgur.com/lkzz5ED.png)  

- 领导者(leader)  
	1. 事务请求的唯一调度和处理者,保证集群事务处理的顺序性;
	2. 集群内部各服务器的调度者
- 跟随者(follower) 
	1. 处理客户端非事务请求,转发事务请求给Leader服务器
	2. 参与事务请求Proposal的投票
	3. 参与Leader选举的投票
- 观察者(observer)  
    1. Follower和Observer唯一的区别在于Observer机器不参与Leader的选举过程,也不参与操作的"过半写成功"策略,因此Observer机器可以在不影响写性能的情况下提升集群的读性能
- 客户端(client)  
	1. 请求发起方
	
当 Leader 服务器出现网络中断,崩溃退出和重启等异常情况时,ZAB协议就会进入恢复模式并选举产生新的 Leader 服务器.这个过程大致是这样的:

1. Leader election(选举阶段) : 节点在一开始都处于选举阶段,只要有一个节点得到超半数节点的票数,它就可以当选准leader;
2. Discovery(发现阶段) : 在这个阶段,followers跟准leader进行通信,同步followers最近接收的事务提议,
3. Synchronization(同步阶段) : 同步阶段主要是利用leader前一阶段获得的最新提议历史,同步集群中所有的副本.同步完成之后,准leader才会成为真正的leader.
4. Broadcast(广播阶段) : 到了这个阶段,Zookeeper集群才能正式对外提供事务服务,并且leader可以进行消息广播,同时如果有新的节点加入,还需要对新节点进行同步.

#### 文件系统 ####
Zookeeper中其实并没有文件和文件夹的概念,它只有一个Znode的概念,它既能作为容器存储数据,也可以持有其他的Znode形成父子关系.  
![](https://i.imgur.com/g2CzCsW.png)  
Znode其实有 PERSISTENT、PERSISTENT_SEQUENTIAL、EPHEMERAL 和 EPHEMERAL_SEQUENTIAL 四种类型，它们是临时与持久、顺序与非顺序两个不同的方向组合成的四种类型.
临时节点是客户端在连接Zookeeper时才会保持存在的节点,一旦客户端和服务端之间的连接中断,当前节点持有的所有节点都会被删除,而持久的节点不会随着会话连接的中断而删除,它们需要被客户端主动删除;Zookeeper中另一个节点的特性就是顺序和非顺序,如果我们使用zookeeper创建了顺序的节点,那么所有节点就会在名字的末尾附加一个序列号,序列号是一个由父节点维护的单调递增计数器.

#### 协调分布式事务 ####
Zookeeper的另一个作用就是担任分布式事务中的协调者角色,在之前介绍分布式事务的文章中我们曾经介绍分布式事务本质上都是通过2PC来实现的,在两阶段提交中就需要一个协调者负责协调分布式事务的执行.  

    ZooKeeper zk = new ZooKeeper("localhost", 3000, null);
	String path = zk.create("/transfer/tx", new byte[0], ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.PERSISTENT_SEQUENTIAL);
	
	List ops = Arrays.asList(
	Op.create(path + "/cohort", new byte[0], ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.PERSISTENT_SEQUENTIAL),
	Op.create(path + "/cohort", new byte[0], ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.PERSISTENT_SEQUENTIAL),
	Op.create(path + "/cohort", new byte[0], ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.PERSISTENT_SEQUENTIAL)
	);
	zk.multi(ops);  

当前节点作为协调者在每次发起分布式事务时都会创建一个 /transfer/tx 的持久顺序节点，然后为几个事务的参与者创建几个空白的节点，事务的参与者在收到事务时会向这些空白的节点中写入信息并监听这些节点中的内容。

所有的事务参与者会向当前节点中写入提交或者终止，一旦当前的节点改变了事务的状态，其他节点就会得到通知，如果出现一个写入终止的节点，所有的节点就会回滚对分布式事务进行回滚。

使用 Zookeeper 实现强一致性的分布式事务其实还是一件比较困难的事情，一方面是因为强一致性的分布式事务本身就有一定的复杂性，另一方面就是 Zookeeper 为了给客户端提供更多的自由，对外暴露的都是比较基础的 API，对它们进行组装实现复杂的分布式事务还是比较麻烦的，对于如何使用 Zookeeper 实现分布式事务，我们可以在 ZooKeeper Recipes and Solutions 一文中找到更为详细的内容。

#### 分布式锁 ####
在数据库中,锁的概念其实是非常重要的,常见的关系型数据库就会对排他锁和共享锁进行支持,而zookeeper提供的api也可以让我们非常简单的实现分布式锁.  

	ZooKeeper zk = new ZooKeeper("localhost", 3000, null);
	final String resource = "/resource";
	
	final String lockNumber = zk
	        .create("/resource/lock-", null, ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.EPHEMERAL_SEQUENTIAL);
	
	List<String> locks = zk.getChildren(resource, false, null);
	Collections.sort(locks);
	if (locks.get(0).equals(lockNumber.replace("/resource/", ""))) {
	    System.out.println("Acquire Lock");
	    zk.delete(lockNumber, 0);
	} else {
	    zk.getChildren(resource, new Watcher() {
	        public void process(WatchedEvent watchedEvent) {
	            try {
	                ZooKeeper zk = new ZooKeeper("localhost", 3000, null);
	                List locks = zk.getChildren(resource, null, null);
	                Collections.sort(locks);
	
	                if (locks.get(0).equals(lockNumber.replace("/resource/", ""))) {
	                    System.out.println("Acquire Lock");
	                    zk.delete(lockNumber, 0);
	                }
	
	            } catch (Exception e) {}
	        }
	    }, null);
	}
   
如果多个服务同时要对某个资源进行修改，就可以使用上述的代码来实现分布式锁，假设集群中存在一个资源 /resource，几个服务需要通过分布式锁保证资源只能同时被一个节点使用，我们可以用创建临时顺序节点的方式实现分布式锁；当我们创建临时节点后，通过 getChildren 获取当前等待锁的全部节点，如果当前节点是所有节点中序号最小的就得到了当前资源的使用权限，在对资源进行处理后，就可以通过删除 /resource/lock-00000000x 来释放锁，如果当前节点不是最小值，就会注册一个 Watcher 等待 /resource 子节点的变化直到当前节点的序列号成为最小值。

上述代码在集群中争夺同一资源的服务器特别多的情况下会出现羊群效应，每次子节点改变时都会通知当前节点，造成资源的浪费，我们其实可以将 getChildren 换成 getData，让当前节点只监听前一个节点的删除事件： 
 
	Integer number = Integer.parseInt(lockNumber.replace("/resource/lock-", "")) + 1;
	String previousLock = "/resource/lock-" + String.format("%010d", number);
	
	zk.getData(previousLock, new Watcher() {
	    public void process(WatchedEvent watchedEvent) {
	        try {
	            if (watchedEvent.getType() == Event.EventType.NodeDeleted) {
	                System.out.println("Acquire Lock");
	                ZooKeeper zk = new ZooKeeper("localhost", 3000, null);
	                zk.delete(lockNumber, 0);
	            }
	        } catch (Exception e) {}
	    }
	}, null); 