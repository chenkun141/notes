## kafka ##
kafka最早是由LinkedIn公司开发的,作为其自身业务消息处理的基础,后LinkedIn公司将kafka捐赠给Apache,现在已经成为Apache的一个顶级项目了,kafka作为一个高吞吐的分布式消息系统,目前已经被很多公司应用在实际的业务中了,并且与许多数据处理架构相结合,比如hadoop,spark等.  
kafka是一个分布式消息队列.具有高性能,持久性,多副本备份,横向扩展能力.生产者往队列里push消息,消费者从队列里pull消息进行业务员逻辑.一般在架构设计中起到解耦,削峰,异步处理的作用

### 参考网址 ###
[http://kafka.apache.org/](http://kafka.apache.org/)  
[https://www.infoq.cn/article/kafka-analysis-part-1](https://www.infoq.cn/article/kafka-analysis-part-1)  
[https://mp.weixin.qq.com/s/fMcRtuX_qfLwI0f8HXiGiQ](https://mp.weixin.qq.com/s/fMcRtuX_qfLwI0f8HXiGiQ)

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

### kafka数据存储设计 ###
#### partition的数据文件(offset,messageSize,data) ####
partition中的每条Message包含了一下三个属性:offset,MessageSize,data,其中offset表示Message在这个partition中的偏移量,offset不是该Message在partition数据文件中的实际存储位置,而是逻辑上一个值,它唯一确定了partition中的一条Message的具体内容.
#### 数据文件分段(顺序读写,分段命令,二分查找) ####
partition物理上由多个segment文件组成,每个segment大小相等,顺序读写,每个segment数据文件以该段中最小的offset命名,文件扩展名为.log.这样在查找指定offset的Message的时候,用二分查找就可以定位到该Message在哪个segment数据文件中.
#### 数据文件索引(分段索引,稀疏存储) ####
kafka为每个分段后的数据文件建立了索引文件,文件名与数据文件的名字是一样的,只是文件扩展名为.index.index文件中并没有为数据文件中的每条Message建立索引,而是采用了稀疏存储的方式,每隔一定字节的数据建立一条索引.这样避免了索引文件占用过多的空间,从而可以将索引文件保留在内存中.
![](https://i.imgur.com/2o6cb8T.png)

### 生产者设计 ###
#### 负载均衡(partition会均衡分布到不同broker上) ####
由于消息topic由多个partition组成,且partition会均衡分布到不同的broker上,因此,为了有效利用broker集群的性能,提高消息的吞吐量,producer可以通过随机或者hash等方式,将消息平均发送到多个partition上,以实现负载均衡.
![](https://i.imgur.com/EaQfAmR.png)
#### 批量发送 ####
是提高消息吞吐量重要的方式,producer端可以在内存中合并多条消息后,以一次请求的方式发送了批量的消息给broker.从而大大减少broker存储消息的IO操作次数.但也一定程度上影响了消息的实时性,相当于以时延代价,换取更好的吞吐量.
#### 压缩(Gzip或snappy) ####
producer端可以通过gzip或snappy格式对消息集合进行压缩.producer端进行压缩之后,在consumer端需进行压缩.压缩的好处就是减少传输的数据量,减轻对网络传输的压力,在对大数据处理上,瓶颈往往体现在网络上而不是CPU(压缩和解压会耗掉部分CPU资源).

### 消费者设计 ###
![](https://i.imgur.com/WrO0pJF.png)
#### consumer group ####
同一consumer group中的多个consumer实例,不同时消费同一个partition,等效于队列模式.partition内消息是有序的,consumer通过pull方式消费消息,卡法咖不删除已消费的消息,对于partition,顺序读写磁盘数据,以时间复杂度O(1)方式提供消息持久化能力.


### kafka安装 ###
#### 安装java环境 ####
不做详细介绍
#### 安装zookeeper环境 ####
kafka的底层使用zookeeper存储元数据,确保一致性,所以安装kafka前需要先安装zookeeper,kafka的发行版自带了zookeeper,可以直接使用脚本启动,不过安装一个zookeeper也不费劲  

#### zookeeper单机搭建 ####
1. 在官网下载tar.gz包,直接解压,解压完成后
	
		cd /usr/local/zookeeper/zookeeper-3.4.10,
2. 创建一个data文件夹,然后进入到conf文件夹下,使用 mv zoo_sample.cfg zoo.cfg 进行重命名操作,
3. 然后使用 vi 打开 zoo.cfg ，更改一下dataDir = /usr/local/zookeeper/zookeeper-3.4.10/data,保存。  
4. 进入bin目录,启动服务输入命令
	
		./zkServer.sh start 
输出下面内容表示大件成功
![](https://i.imgur.com/TPNRCj1.jpg)  
5. 关闭服务输入命令,
	
		./zkServer.sh stop
![](https://i.imgur.com/rkqbbMi.jpg)  
6. 查看状态信息
		
		./zkServer.sh status

#### zookeeper集群搭建 ####
安装三个虚拟机,为各自的虚拟机安装zookeeper
> 注:在每个虚拟机搭建好,新建两个文件夹,分别是data和log文件夹


1. 设置集群  
编辑文件conf/zoo.cfg,三个文件的内容如下:

		tickTime=2000
		initLimit=10
		syncLimit=5
		#创建data目录
		dataDir=/usr/local/zookeeper/zookeeper-3.4.10/data
		#创建的log目录
		dataLogDir=/usr/local/zookeeper/zookeeper-3.4.10/log
		clientPort=12181
		#虚拟机服务器ip地址
		server.1=192.168.1.7:12888:13888
		server.2=192.168.1.8:12888:13888
		server.3=192.168.1.9:12888:13888



	> server.1 中的这个1表示的是服务器的表示也可以是其他数字,表示这是第几号服务器,这个标识要喝下面我们配置的myid的标识一致可以.
192.168.1.7:12888:13888为集群中的IP地址,第一个端口表示的是master与slave之间的通信接口,默认是2888,第二个端口是leader选举的端口,集群刚启动的时候选举或者leader挂掉之后进行新的选举的端口,默认是3888

	配置解释:  

	- **tickTime**:这个时间是作为zookeeper服务器之间或客户端与服务器之间维持心跳的时间间隔,也就是每个tickTime时间就会发送一个心跳.  
	- **initLimit**:这个配置项是用来配置zookeeper接受客户端(这里所说的客户端不是用户连接zookeeper服务器的客户端,而是zookeeper服务器集群中连接到leader的follower服务器)初始化连接是最长能忍受多少个心跳时间间隔数.当已经超过5个心跳的时间(也就是tickTime)长度后zookeeper服务器还没有收到客户端的返回信息,那么表明这个客户端连接失败.总的时间长度就是5*2000 = 10秒  
	- **syncLimit**: 这个配置项标识leader和follower之间发送消息,请求和应答时间长度,最长不能超过多少个tickTime的时间长度,总的时间长度就是5*2000=10秒; 
	- **dataDir**:快照日志的存储路径  
	- **dataLogDir**:事务日志的存储路径,如果不配置这个,那么事务日志默认存储到dataDir指定的目录,这样会严重影响zk的性能,当zk吞吐量较大的时候,产生的事务日志,快照日志太多
	- **clientPort**:这个端口就是客户端连接zookeeper服务器的端口,zookeeper会监听这个端口,接受客户端的访问请求.



2. 创建myid文件  
在了解其配置文件后,现在来常见每个集群节点的myid,我们上面说过,这个myid就是server.1的这个1,类似的,需要为集群中的每个服务都指定标识,使用echo命令进行创建

		# server.1
		echo "1" > /usr/local/zookeeper/zookeeper-3.4.10/data/myid
		# server.2
		echo "2" > /usr/local/zookeeper/zookeeper-3.4.10/data/myid
		# server.3
		echo "3" > /usr/local/zookeeper/zookeeper-3.4.10/data/myid

3. 启动服务并测试
配置完成,为每个zk服务启动并测试,我在Windows电脑的测试结果如下  
启动服务(每台都需要执行)
	
		cd /usr/local/zookeeper/zookeeper-3.4.10/bin
		./zkServer.sh start

检查服务状态

	 ./zkServer.sh status   

zk集群一般只有一个leader，多个follower，主一般是相应客户端的读写请求，而从主同步数据，当主挂掉之后就会从follower里投票选举一个leader出来。
	
#### kafka集群搭建 ####
1. 准备条件
 - 搭建好zookeeper集群
 - kafka压缩包

2. 安装kafka  
在/usr/local下新建kafka文件夹,然后把下载完成的tar.gz包一到/usr/local/kafaka目录下,使用tar -zxvf压缩包进行解压,解压完成后,进入到kafka_2.12-2.3.0 目录下,新建log文件夹,进入到config目录下  
我们可以看到有很多properties配置文件,这里主要关注server.properties这个文件即可.
![](https://i.imgur.com/Qo1Eylo.jpg)
	 		
- kafka 启动方式有两种  

		- 一种是使用 kafka 自带的 zookeeper 配置文件来启动（可以按照官网来进行启动，并使用单个服务多个节点来模拟集群http://kafka.apache.org/quickstart#quickstart_multibroker）
		- 一种是通过使用独立的zk集群来启动，这里推荐使用第二种方式，使用 zk 集群来启动	

- 修改配置项  
需要为每个服务都修改一下配置项,也就是server.properties,需要更新和添加的内容有

			broker.id=0 //初始是0，每个 server 的broker.id 都应该设置为不一样的，就和 myid 一样 我的三个服务分别设置的是 1,2,3
			log.dirs=/usr/local/kafka/kafka_2.12-2.3.0/log
	
			#在log.retention.hours=168 下面新增下面三项
			message.max.byte=5242880
			default.replication.factor=2
			replica.fetch.max.bytes=5242880
			
			#设置zookeeper的连接端口
			zookeeper.connect=192.168.1.7:2181,192.168.1.8:2181,192.168.1.9:2181

		配置项的含义

			broker.id=0  #当前机器在集群中的唯一标识，和zookeeper的myid性质一样
			port=9092 #当前kafka对外提供服务的端口默认是9092
			host.name=192.168.1.7 #这个参数默认是关闭的，在0.8.1有个bug，DNS解析问题，失败率的问题。
			num.network.threads=3 #这个是borker进行网络处理的线程数
			num.io.threads=8 #这个是borker进行I/O处理的线程数
			log.dirs=/usr/local/kafka/kafka_2.12-2.3.0/log #消息存放的目录，这个目录可以配置为“，”逗号分割的表达式，上面的num.io.threads要大于这个目录的个数这个目录，如果配置多个目录，新创建的topic他把消息持久化的地方是，当前以逗号分割的目录中，那个分区数最少就放那一个
			socket.send.buffer.bytes=102400 #发送缓冲区buffer大小，数据不是一下子就发送的，先回存储到缓冲区了到达一定的大小后在发送，能提高性能
			socket.receive.buffer.bytes=102400 #kafka接收缓冲区大小，当数据到达一定大小后在序列化到磁盘
			socket.request.max.bytes=104857600 #这个参数是向kafka请求消息或者向kafka发送消息的请请求的最大数，这个值不能超过java的堆栈大小
			num.partitions=1 #默认的分区数，一个topic默认1个分区数
			log.retention.hours=168 #默认消息的最大持久化时间，168小时，7天
			message.max.byte=5242880  #消息保存的最大值5M
			default.replication.factor=2  #kafka保存消息的副本数，如果一个副本失效了，另一个还可以继续提供服务
			replica.fetch.max.bytes=5242880  #取消息的最大直接数
			log.segment.bytes=1073741824 #这个参数是：因为kafka的消息是以追加的形式落地到文件，当超过这个值的时候，kafka会新起一个文件
			log.retention.check.interval.ms=300000 #每隔300000毫秒去检查上面配置的log失效时间（log.retention.hours=168 ），到目录查看是否有过期的消息如果有，删除
			log.cleaner.enable=false #是否启用log压缩，一般不用启用，启用的话可以提高性能
			zookeeper.connect=192.168.1.7:2181,192.168.1.8:2181,192.168.1.9:2181 #设置zookeeper的连接端口

- 启动Kafka集群并测试  

		- 启动服务,进入到**/usr/local/kafka/kafka_2.12-2.3.0/bin**目录下
			
				# 启动后台进程
				./kafka-server-start.sh -daemon ../config/server.properties
		
		- 检查服务是否启动
				
				# 执行命令 jps
				6201 QuorumPeerMain
				7035 Jps
				6972 Kafka
		
		- kafka已经启动
- 创建topic来验证是否创建成功
		
			# cd .. 往回退一层 到 /usr/local/kafka/kafka_2.12-2.3.0 目录下
			bin/kafka-topics.sh --create --zookeeper 192.168.1.7:2181 --replication-factor 2 --partitions 1 --topic cxuan
		
		对上面的解释  

			--replication-factor 2   复制两份  
			--partitions 1 创建1个分区  
			--topic 创建主题  
		
		查看我们的主题是否创建成功
		
			bin/kafka-topics.sh --list --zookeeper 192.168.1.7:2181

- 启动一个服务就能把集群启动起来

		- 在一台机器上创建一个发布者
		
				# 创建一个broker，发布者
				./kafka-console-producer.sh --broker-list 192.168.1.7:9092 --topic cxuantopic
		
		- 在一台服务器上创建一个订阅者
		
				# 创建一个consumer， 消费者
				bin/kafka-console-consumer.sh --bootstrap-server 192.168.1.7:9092 --topic cxuantopic --from-beginning

		> 注意：这里使用 --zookeeper 的话可能出现 zookeeper is not a recognized option 的错误，这是因为 kafka 版本太高，需要使用 --bootstrap-server 指令

- 测试结果
		- 发布
	![](https://i.imgur.com/7s8Mu3j.png)
		- 消费
	![](https://i.imgur.com/nTe1C6n.png)

- 其他命令  

		- 显示topic
		
				bin/kafka-topics.sh --list --zookeeper 192.168.1.7:2181
				
				# 显示
				cxuantopic
		
		- 查看topic状态
		
				bin/kafka-topics.sh --describe --zookeeper 192.168.1.7:2181 --topic cxuantopic
				
				# 下面是显示的详细信息
				Topic:cxuantopic PartitionCount:1 ReplicationFactor:2 Configs:
				Topic: cxuantopic Partition: 0 Leader: 1 Replicas: 1,2 Isr: 1,2
				
				# 分区为为1  复制因子为2   主题 cxuantopic 的分区为0 
				# Replicas: 0,1   复制的为1，2
		
		- **Leader** 负责给定分区的所有读取和写入的节点，每个节点都会通过随机选择成为 leader。  
		- **Replicas** 是为该分区复制日志的节点列表，无论它们是 Leader 还是当前处于活动状态。  
		- **Isr** 是同步副本的集合。它是副本列表的子集，当前仍处于活动状态并追随Leader。  
至此，kafka 集群搭建完毕。
		 

- 验证多节点接收数据   
	刚刚我们都使用的是 相同的ip服务，下面使用其他集群中的节点，验证是否能够接受到服务在另外两个节点上使用

			bin/kafka-console-consumer.sh --bootstrap-server 192.168.1.7:9092 --topic cxuantopic --from-beginning
然后再使用 broker 进行消息发送，经测试三个节点都可以接受到消息。
	
#### 配置详解 ####

在搭建 Kafka 的时候我们简单介绍了一下 server.properties 中配置的含义，现在我们来详细介绍一下参数的配置和概念  

1. 常规配置  
这些参数是kafka中最基本的配置  

- broker.id  
每个 broker 都需要有一个标识符，使用 broker.id 来表示。它的默认值是 0，它可以被设置成其他任意整数，在集群中需要保证每个节点的 broker.id 都是唯一的。
- port  
如果使用配置样本来启动 kafka ，它会监听 9092 端口，修改 port 配置参数可以把它设置成其他任意可用的端口。
- zookeeper.connect  
用于保存 broker 元数据的地址是通过 zookeeper.connect 来指定。localhost:2181 表示运行在本地 2181 端口。该配置参数是用逗号分隔的一组 hostname:port/path 列表，每一部分含义如下：
hostname 是 zookeeper 服务器的服务名或 IP 地址
port 是 zookeeper 连接的端口
/path 是可选的 zookeeper 路径，作为 Kafka 集群的 chroot 环境。如果不指定，默认使用跟路径
- log.dirs  
Kafka 把消息都保存在磁盘上，存放这些日志片段的目录都是通过 log.dirs 来指定的。它是一组用逗号分隔的本地文件系统路径。如果指定了多个路径，那么 broker 会根据 "最少使用" 原则，把同一分区的日志片段保存到同一路径下。要注意，broker 会向拥有最少数目分区的路径新增分区，而不是向拥有最小磁盘空间的路径新增分区。
- num.recovery.threads.per.data.dir  
对于如下 3 种情况，Kafka 会使用可配置的线程池来处理日志片段
服务器正常启动，用于打开每个分区的日志片段；
服务器崩溃后启动，用于检查和截断每个分区的日志片段；
服务器正常关闭，用于关闭日志片段
默认情况下，每个日志目录只使用一个线程。因为这些线程只是在服务器启动和关闭时会用到，所以完全可以设置大量的线程来达到井行操作的目的。特别是对于包含大量分区的服务器来说，一旦发生崩愤，在进行恢复时使用井行操作可能会省下数小时的时间。设置此参数时需要注意，所配置的数字对应的是 log.dirs 指定的单个日志目录。也就是说，如果 num.recovery.threads.per.data.dir 被设为 8，并且 log.dir 指定了 3 个路径，那么总共需要 24 个线程。
- auto.create.topics.enable  
默认情况下，Kafka 会在如下 3 种情况下创建主题
当一个生产者开始往主题写入消息时
当一个消费者开始从主题读取消息时
当任意一个客户向主题发送元数据请求时
- delete.topic.enable  
如果你想要删除一个主题，你可以使用主题管理工具。默认情况下，是不允许删除主题的，delete.topic.enable 的默认值是 false 因此你不能随意删除主题。这是对生产环境的合理性保护，但是在开发环境和测试环境，是可以允许你删除主题的，所以，如果你想要删除主题，需要把 delete.topic.enable 设为 true。

2. 主题默认配置  
Kafka 为新创建的主题提供了很多默认配置参数，下面就来一起认识一下这些参数
- num.partitions  
num.partitions 参数指定了新创建的主题需要包含多少个分区。如果启用了主题自动创建功能（该功能是默认启用的），主题分区的个数就是该参数指定的值。该参数的默认值是 1。要注意，我们可以增加主题分区的个数，但不能减少分区的个数。
- default.replication.factor  
这个参数比较简单，它表示 kafka保存消息的副本数，如果一个副本失效了，另一个还可以继续提供服务default.replication.factor 的默认值为1，这个参数在你启用了主题自动创建功能后有效。
- log.retention.ms  
Kafka 通常根据时间来决定数据可以保留多久。默认使用 log.retention.hours 参数来配置时间，默认是 168 个小时，也就是一周。除此之外，还有两个参数 log.retention.minutes 和 log.retentiion.ms 。这三个参数作用是一样的，都是决定消息多久以后被删除，推荐使用 log.retention.ms。
- log.retention.bytes  
另一种保留消息的方式是判断消息是否过期。它的值通过参数 log.retention.bytes 来指定，作用在每一个分区上。也就是说，如果有一个包含 8 个分区的主题，并且 log.retention.bytes 被设置为 1GB，那么这个主题最多可以保留 8GB 数据。所以，当主题的分区个数增加时，整个主题可以保留的数据也随之增加。
- log.segment.bytes  
上述的日志都是作用在日志片段上，而不是作用在单个消息上。当消息到达 broker 时，它们被追加到分区的当前日志片段上，当日志片段大小到达 log.segment.bytes 指定上限（默认为 1GB）时，当前日志片段就会被关闭，一个新的日志片段被打开。如果一个日志片段被关闭，就开始等待过期。这个参数的值越小，就越会频繁的关闭和分配新文件，从而降低磁盘写入的整体效率。
- log.segment.ms  
上面提到日志片段经关闭后需等待过期，那么log.segment.ms 这个参数就是指定日志多长时间被关闭的参数和，log.segment.ms 和 log.retention.bytes 也不存在互斥问题。日志片段会在大小或时间到达上限时被关闭，就看哪个条件先得到满足。
- message.max.bytes  
broker 通过设置 message.max.bytes 参数来限制单个消息的大小，默认是 1000 000， 也就是 1MB，如果生产者尝试发送的消息超过这个大小，不仅消息不会被接收，还会收到 broker 返回的错误消息。跟其他与字节相关的配置参数一样，该参数指的是压缩后的消息大小，也就是说，只要压缩后的消息小于 mesage.max.bytes，那么消息的实际大小可以大于这个值
这个值对性能有显著的影响。值越大，那么负责处理网络连接和请求的线程就需要花越多的时间来处理这些请求。它还会增加磁盘写入块的大小，从而影响 IO 吞吐量。
		