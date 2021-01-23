# Redis #
## 参考资料 ##
[https://juejin.im/post/5b4dd82ee51d451925629622#heading-63](https://juejin.im/post/5b4dd82ee51d451925629622#heading-63)  

## 数据类型 ##
Redis最为常用的数据类型主要有以下五种：

- String
- Hash
- List
- Set
- Sorted set  

在具体描述这几种数据类型之前，我们先通过一张图了解下Redis内部内存管理中是如何描述这些不同数据类型的：  
![](https://i.imgur.com/oUkvCgh.jpg)  

首先Redis内部使用一个redisObject对象来表示所有的key和value,redisObject最主要的信息如上图所示：type:代表一个value对象具体是何种数据类型，encoding是不同数据类型在redis内部的存储方式，比如：type=string代表value存储的是一个普通字符串，那么对应的encoding可以是raw或者是int,如果是int则代表实际redis内部是按数值型类存储和表示这个字符串的，当然前提是这个字符串本身可以用数值表示，比如:"123" "456"这样的字符串。  
这里需要特殊说明一下vm字段，只有打开了Redis的虚拟内存功能，此字段才会真正的分配内存，该功能默认是关闭状态的，该功能会在后面具体描述。通过上图我们可以发现Redis使用redisObject来表示所有的key/value数据是比较浪费内存的，当然这些内存管理成本的付出主要也是为了给Redis不同数据类型提供一个统一的管理接口，实际作者也提供了多种方法帮助我们尽量节省内存使用，我们随后会具体讨论。  
下面我们先来逐一的分析下这五种数据类型的使用和内部实现方式：

- String
	1. 常用命令：set,get,decr,incr,mget 等。
	2. 应用场景：String是最常用的一种数据类型，普通的key/value存储都可以归为此类，这里就不所做解释了。
	3. 实现方式：  String在redis内部存储默认就是一个字符串，被redisObject所引用，当遇到incr,decr等操作时会转成数值型进行计算，此时redisObject的encoding字段为int。
- Hash
	1. 常用命令：hget,hset,hgetall 等。
	2. 应用场景：我们简单举个实例来描述下Hash的应用场景，比如我们要存储一个用户信息对象数据，包含以下信息：用户ID为查找的key，存储的value用户对象包含姓名，年龄，生日等信息，如果用普通的key/value结构来存储，主要有以下2种存储方式：  
第一种方式将用户ID作为查找key,把其他信息封装成一个对象以序列化的方式存储，这种方式的缺点是，增加了序列化/反序列化的开销，并且在需要修改其中一项信息时，需要把整个对象取回，并且修改操作需要对并发进行保护，引入CAS等复杂问题。  
第二种方法是这个用户信息对象有多少成员就存成多少个key-value对儿，用用户ID+对应属性的名称作为唯一标识来取得对应属性的值，虽然省去了序列化开销和并发问题，但是用户ID为重复存储，如果存在大量这样的数据，内存浪费还是非常可观的。  
那么Redis提供的Hash很好的解决了这个问题，Redis的Hash实际是内部存储的Value为一个HashMap，并提供了直接存取这个Map成员的接口。  
也就是说，Key仍然是用户ID, value是一个Map，这个Map的key是成员的属性名，value是属性值，这样对数据的修改和存取都可以直接通过其内部Map的 Key(Redis里称内部Map的key为field), 也就是通过 key(用户ID) + field(属性标签) 就可以操作对应属性数据了，既不需要重复存储数据，也不会带来序列化和并发修改控制的问题。很好的解决了问题。  
这里同时需要注意，Redis提供了接口(hgetall)可以直接取到全部的属性数据,但是如果内部Map的成员很多，那么涉及到遍历整 个内部Map的操作，由于Redis单线程模型的缘故，这个遍历操作可能会比较耗时，而另其它客户端的请求完全不响应，这点需要格外注意。  
上面已经说到Redis Hash对应Value内部实际就是一个HashMap，实际这里会有2种不同实现，这个Hash的成员比较少时Redis为了节省内存会采用类似一维数组的方式来紧凑存储，而不会采用真正的HashMap结构，对应的value redisObject的encoding为zipmap,当成员数量增大时会自动转成真正的HashMap,此时encoding为ht。  
- List
	1. 常用命令：lpush,rpush,lpop,rpop,lrange等。
	2. 应用场景：  Redis list的应用场景非常多，也是Redis最重要的数据结构之一，比如twitter的关注列表，粉丝列表等都可以用Redis的list结构来实现，比较好理解，这里不再重复。
	3. 实现方式：  Redis list的实现为一个双向链表，即可以支持反向查找和遍历，更方便操作，不过带来了部分额外的内存开销，Redis内部的很多实现，包括发送缓冲队列等也都是用的这个数据结构。
- Set
	1. 常用命令：sadd,spop,smembers,sunion 等。
	2. 应用场景：Redis set对外提供的功能与list类似是一个列表的功能，特殊之处在于set是可以自动排重的，当你需要存储一个列表数据，又不希望出现重复数据时，set 是一个很好的选择，并且set提供了判断某个成员是否在一个set集合内的重要接口，这个也是list所不能提供的。
	3. 实现方式：set 的内部实现是一个 value永远为null的HashMap，实际就是通过计算hash的方式来快速排重的，这也是set能提供判断一个成员是否在集合内的原因。
- Sorted set
	1. 常用命令：zadd,zrange,zrem,zcard等
	2. 使用场景：Redis sorted set的使用场景与set类似，区别是set不是自动有序的，而sorted set可以通过用户额外提供一个优先级(score)的参数来为成员排序，并且是插入有序的，即自动排序。当你需要一个有序的并且不重复的集合列表，那么 可以选择sorted set数据结构，比如twitter 的public timeline可以以发表时间作为score来存储，这样获取时就是自动按时间排好序的。
	3. 实现方式：Redis sorted set的内部使用HashMap和跳跃表(SkipList)来保证数据的存储和有序，HashMap里放的是成员到score的映射，而跳跃表里存放的 是所有的成员，排序依据是HashMap里存的score,使用跳跃表的结构可以获得比较高的查找效率，并且在实现上比较简单。

## Redis集群 ##
### redis-cluster 架构图 ###
 ![](https://i.imgur.com/Q8mp51D.png)  

架构细节:  

1. 所有的redis节点彼此互联(PING-PONG机制),内部使用二进制协议优化传输速度和带宽.
2. 节点的fail是通过集群中超过半数的节点检测失效时才生效.
3. 客户端与redis节点直连,不需要中间proxy层.客户端不需要连接集群所有节点,连接集群中任何一个可用节点即可
4. redis-cluster把所有的物理节点映射到[0-16383]slot上,cluster 负责维护node<->slot<->value
Redis 集群中内置了 16384 个哈希槽，当需要在 Redis 集群中放置一个 key-value 时，redis 先对 key 使用 crc16 算法算出一个结果，然后把结果对 16384 求余数，这样每个 key 都会对应一个编号在 0-16383 之间的哈希槽，redis 会根据节点数量大致均等的将哈希槽映射到不同的节点.

### redis-cluster 投票 容错 ###
![](https://i.imgur.com/Kjm7xJm.png)  
1. 集群中所有master参与投票,如果半数以上master节点与其中一个master节点通信超过(cluster-node-timeout),认为该master节点挂掉.
2. 什么时候整个集群不可用(cluster_state:fail)?
	- 如果集群任意master挂掉,且当前master没有slave，则集群进入fail状态。也可以理解成集群的[0-16383]slot映射不完全时进入fail状态。
	- 如果集群超过半数以上master挂掉，无论是否有slave，集群进入fail状态。

### reids的持久化 ###
[https://segmentfault.com/a/1190000015983518?utm_source=tag-newest](https://segmentfault.com/a/1190000015983518?utm_source=tag-newest) 
[http://redisdoc.com/topic/persistence.html](http://redisdoc.com/topic/persistence.html) 
#### 持久化方式 ####

- RDB(point-in-time snapshot):在指定的时间间断能对你的数据进行快照存储(RDB是快照文件的方式,redis通过执行save/bgsave命令,执行数据的备分,将redis当前的数据保存到*.rdb文件中,文件保存了所有的数据集合)
- AOF:记录每次对服务器写的操作,当服务器重启的时候会重新执行这些命令来恢复原始的数据(AOF是服务器通过读取配置,在指定的时间里,追加redis写操作的命令到*.aof文件中,是一种增量的持久化方式.) 
 
#### 持久化的配置 ####
1. RDB的持久化配置

		# 时间策略
		save 900 1
		save 300 10
		save 60 10000
		
		# 文件名称
		dbfilename dump.rdb
		
		# 文件保存路径
		dir /home/work/app/redis/data/
		
		# 如果持久化出错，主进程是否停止写入
		stop-writes-on-bgsave-error yes
		
		# 是否压缩
		rdbcompression yes
		
		# 导入时是否检查
		rdbchecksum yes

	
配置其实非常简单,这里说一下持久化的时间策略具体是什么意思.  

- save 900 1	
	- 表示900s内如果有1条是写入命令,就出发产生一次快照,可以理解为就是进行一次备份
- save 300 10
	- 表示300s内有10条写入,就产生快照
		
下面的类似,那么为什么需要配置这么多条规则呢?因为Redis每个时段的读写请求肯定不是均衡的，为了平衡性能与数据安全，我们可以自由定制什么情况下触发备份。所以这里就是根据自身Redis写入情况来进行合理配置。

stop-writes-on-bgsave-error yes 这个配置也是非常重要的一项配置，这是当备份进程出错时，主进程就停止接受新的写入操作，是为了保护持久化的数据一致性问题。如果自己的业务有完善的监控系统，可以禁止此项配置， 否则请开启。

关于压缩的配置 rdbcompression yes ，建议没有必要开启，毕竟Redis本身就属于CPU密集型服务器，再开启压缩会带来更多的CPU消耗，相比硬盘成本，CPU更值钱。

当然如果你想要禁用RDB配置，也是非常容易的，只需要在save的最后一行写上：save ""  
	 
2. AOF的配置  
	
		# 是否开启aof
		appendonly yes
		
		# 文件名称
		appendfilename "appendonly.aof"
		
		# 同步方式
		appendfsync everysec
		
		# aof重写期间是否同步
		no-appendfsync-on-rewrite no
		
		# 重写触发配置
		auto-aof-rewrite-percentage 100
		auto-aof-rewrite-min-size 64mb
		
		# 加载aof时如果有错如何处理
		aof-load-truncated yes
		
		# 文件重写策略
		aof-rewrite-incremental-fsync yes


还是重点解释一些关键的配置：

appendfsync everysec 它其实有三种模式:  

- always：把每个写命令都立即同步到aof，很慢，但是很安全
- everysec：每秒同步一次，是折中方案
- no：redis不处理交给OS来处理，非常快，但是也最不安全  \

一般情况下都采用 everysec 配置，这样可以兼顾速度与安全，最多损失1s的数据。

aof-load-truncated yes 如果该配置启用，在加载时发现aof尾部不正确是，会向客户端写入一个log，但是会继续执行，如果设置为 no ，发现错误就会停止，必须修复后才能重新加载。

#### 工作原理 ####
关于原理部分，我们主要来看RDB与AOF是如何完成持久化的，他们的过程是如何。

在介绍原理之前先说下Redis内部的定时任务机制，定时任务执行的频率可以在配置文件中通过 hz 10 来设置（这个配置表示1s内执行10次，也就是每100ms触发一次定时任务）。该值最大能够设置为：500，但是不建议超过：100，因为值越大说明执行频率越频繁越高，这会带来CPU的更多消耗，从而影响主进程读写性能。

定时任务使用的是Redis自己实现的 TimeEvent，它会定时去调用一些命令完成定时任务，这些任务可能会阻塞主进程导致Redis性能下降。因此我们在配置Redis时，一定要整体考虑一些会触发定时任务的配置，根据实际情况进行调整。

#### RDB的原理 ####
在Redis中RDB持久化的触发分为两种：自己手动触发与Redis定时触发。

针对RDB方式的持久化，手动触发可以使用：

- save：会阻塞当前Redis服务器，直到持久化完成，线上应该禁止使用。
- bgsave：该触发方式会fork一个子进程，由子进程负责持久化过程，因此阻塞只会发生在fork子进程的时候  

而自动触发的场景主要是有以下几点：

- 根据我们的 save m n 配置规则自动触发；
- 从节点全量复制时，主节点发送rdb文件给从节点完成复制操作，主节点会触发 bgsave；
- 执行 debug reload 时；
- 执行 shutdown时，如果没有开启aof，也会触发  

由于 save 基本不会被使用到，我们重点看看 bgsave 这个命令是如何完成RDB的持久化的。
![](https://i.imgur.com/Ys7Cq5s.png)  
这里注意的是 fork 操作会阻塞，导致Redis读写性能下降。我们可以控制单个Redis实例的最大内存，来尽可能降低Redis在fork时的事件消耗。以及上面提到的自动触发的频率减少fork次数，或者使用手动触发，根据自己的机制来完成持久化。

#### AOF的原理 ####
AOF的整个流程大体来看可以分为两步，一步是命令的实时写入（如果是 appendfsync everysec 配置，会有1s损耗），第二步是对aof文件的重写。

对于增量追加到文件这一步主要的流程是：命令写入=》追加到aof_buf =》同步到aof磁盘。那么这里为什么要先写入buf在同步到磁盘呢？如果实时写入磁盘会带来非常高的磁盘IO，影响整体性能。

aof重写是为了减少aof文件的大小，可以手动或者自动触发，关于自动触发的规则请看上面配置部分。fork的操作也是发生在重写这一步，也是这里会对主进程产生阻塞。

手动触发： bgrewriteaof，自动触发 就是根据配置规则来触发，当然自动触发的整体时间还跟Redis的定时任务频率有关系。

下面来看看重写的一个流程图:  
![](https://i.imgur.com/z1bmvgL.png)  
对于上图有四个关键点补充一下：

1. 在重写期间，由于主进程依然在响应命令，为了保证最终备份的完整性；因此它依然会写入旧的AOF file中，如果重写失败，能够保证数据不丢失。
2. 为了把重写期间响应的写入信息也写入到新的文件中，因此也会为子进程保留一个buf，防止新写的file丢失数据。
3. 重写是直接把当前内存的数据生成对应命令，并不需要读取老的AOF文件进行分析、命令合并。
4. AOF文件直接采用的文本协议，主要是兼容性好、追加方便、可读性高可认为修改修复。

> 不能是RDB还是AOF都是先写入一个临时文件，然后通过 rename 完成文件的替换工作。


#### 从持久化中恢复数据 ####
数据的备份、持久化做完了，我们如何从这些持久化文件中恢复数据呢？如果一台服务器上有既有RDB文件，又有AOF文件，该加载谁呢？

其实想要从这些文件中恢复数据，只需要重新启动Redis即可。我们还是通过图来了解这个流程:  
![](https://i.imgur.com/HEki8DR.png)  
启动时会先检查AOF文件是否存在，如果不存在就尝试加载RDB。那么为什么会优先加载AOF呢？因为AOF保存的数据更完整，通过上面的分析我们知道AOF基本上最多损失1s的数据。  

#### 性能与实践 ####
通过上面的分析，我们都知道RDB的快照、AOF的重写都需要fork，这是一个重量级操作，会对Redis造成阻塞。因此为了不影响Redis主进程响应，我们需要尽可能降低阻塞。

1. 降低fork的频率，比如可以手动来触发RDB生成快照、与AOF重写；
2. 控制Redis最大使用内存，防止fork耗时过长；
3. 使用更牛逼的硬件；
4. 合理配置Linux的内存分配策略，避免因为物理内存不足导致fork失败。

在线上我们到底该怎么做？我提供一些自己的实践经验。

1. 如果Redis中的数据并不是特别敏感或者可以通过其它方式重写生成数据，可以关闭持久化，如果丢失数据可以通过其它途径补回；
2. 自己制定策略定期检查Redis的情况，然后可以手动触发备份、重写数据；
3. 单机如果部署多个实例，要防止多个机器同时运行持久化、重写操作，防止出现内存、CPU、IO资源竞争，让持久化变为串行；
4. 可以加入主从机器，利用一台从机器进行备份处理，其它机器正常响应客户端的命令；
5. RDB持久化与AOF持久化可以同时存在，配合使用。  

本文的内容主要是运维上的一些注意点，但我们开发者了解到这些知识，在某些时候有助于我们发现诡异的bug。接下来会介绍Redis的主从复制与集群的知识。