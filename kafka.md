## kafka ##
kafka最早是由LinkedIn公司开发的,作为其自身业务消息处理的基础,后LinkedIn公司将kafka捐赠给Apache,现在已经成为Apache的一个顶级项目了,kafka作为一个高吞吐的分布式消息系统,目前已经被很多公司应用在实际的业务中了,并且与许多数据处理架构相结合,比如hadoop,spark等.  
kafka是一个分布式消息队列.具有高性能,持久性,多副本备份,横向扩展能力.生产者往队列里push消息,消费者从队列里pull消息进行业务员逻辑.一般在架构设计中起到解耦,削峰,异步处理的作用

### 参考网址 ###
[http://kafka.apache.org/](http://kafka.apache.org/)  
[https://www.infoq.cn/article/kafka-analysis-part-1](https://www.infoq.cn/article/kafka-analysis-part-1)

### 基本概念介绍 ###
kafka是运行在一个集群上,所以它可以拥有一个或多个服务节点,kafka集群将消息存储在特定的文件中,对外表现为Topics,每条消息记录都包含一个key,消息内容以及时间戳.  

![](https://i.imgur.com/wVYOoZ6.png)  

如上图所示,一个典型的kafka集群包含若干producer,若干braker,若干consumer Gruop,以及一个zookeeper集群.kafka通过zookeeper管理集群配置,选举leader,以及在consumer group发生变化是进行rebalance,producer使用push模式将消息发布到broker,consumer使用pull模式从broker订阅并消费消息.  

- broker : 可以简单理解为一个kafka的节点,多个broker节点构成整个kafka集群;
- producer : 生产者,通过broker发布新的消息到某个topic中
- consumer : 消费者,通过broker从某个topic 获取消息
- consumer Group : 每个consumer属于一个特定的consumer Group(可为每个consumer指定group name,若不指定group name 则属于默认的group)  

	![](http://kafka.apache.org/21/images/log_anatomy.png)
- topic : 某种类型的消息合集;  
	- Partition : 它是topic在物理上的分组,多个Partition会被分散地存储在不同的kafka节点上,单个partition的消息是保证有序的,但是整个topic的消息就不一定是有34序的;
	- segment : 包含消息内容的指定大小的文件,由index文件和log文件组成;一个partition由多个segment组成
		- offset : segment文件中消息的索引值,从0开始计数
	- replica(N) : 消息的冗余备份,表现为每个partition都会有N个完全相同的冗余备份,这些备份被尽量分散存储在不同的机器上;
	
### Topic & Partition ###
topic在逻辑上可以被认为是一个queue,每条消息都必须指定她的topic,可以简单理解为必须指明吧这条消息放进哪个queue里.为了使得kafka的吞吐率可以线性提高物理上把topic分成一个或多个partition,每个partition在物理上对应一个文件夹,该文件件下存储这个partition的所有消息合索引文件.

kafka提供两种策略删除旧数据,一是基于时间,二是基于partition文件大小,例如可以通过配置$KAFKA_HOME/config/server.properties,让kafka删除一周之前的数据,也可以在partition文件超过1GB时删除旧数据,配置如下
	
	# The minimum age of a log file to be eligible for deletion
	log.retention.hours=168
	# The maximum size of a log segment file. When this size is reached a new log segment will be created.
	log.segment.bytes=1073741824
	# The interval at which log segments are checked to see if they can be deleted according to the retention policies
	log.retention.check.interval.ms=300000
	# If log.cleaner.enable=true is set the cleaner will be enabled and individual logs can then be marked for log compaction.
	log.cleaner.enable=false
	

### Producer 消息路由 ###
producer发送消息到broker时,会根据partition机制选择将其存储到哪个partition,如果partition机制设置合理,所有消息可以均匀分布在不同的partition里,这样就实现了负载均衡.	
如果一个topic对应一个文件,那这个文件所在的机器I/O将会成为这个topic的性能瓶颈,而有了partition后,不同的消息可以并行写入不同的broker的不同的partition中,极大的提高了吞吐率,可以在$KAFKA_HOME/config/server.properties中通过配置项num.partitions来指定新建Topic的默认Partition数量，也可在创建Topic时通过参数指定，同时也可以在Topic创建之后通过Kafka提供的工具修改。

### Push vs. Pull ###
作为一个消息系统，Kafka 遵循了传统的方式，选择由 Producer 向 broker push 消息并由 Consumer 从 broker pull 消息。一些 logging-centric system，比如 Facebook 的Scribe和 Cloudera 的Flume，采用 push 模式。事实上，push 模式和 pull 模式各有优劣。

push 模式很难适应消费速率不同的消费者，因为消息发送速率是由 broker 决定的。push 模式的目标是尽可能以最快速度传递消息，但是这样很容易造成 Consumer 来不及处理消息，典型的表现就是拒绝服务以及网络拥塞。而 pull 模式则可以根据 Consumer 的消费能力以适当的速率消费消息。

对于 Kafka 而言，pull 模式更合适。pull 模式可简化 broker 的设计，Consumer 可自主控制消费消息的速率，同时 Consumer 可以自己控制消费方式——即可批量消费也可逐条消费，同时还能选择不同的提交方式从而实现不同的传输语义。

