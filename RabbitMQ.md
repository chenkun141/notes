## 概念 ##
RabbitMQ 是一个由 Erlang 语言开发的 AMQP 的开源实现。  
AMQP ：Advanced Message Queue，高级消息队列协议。它是应用层协议的一个开放标准，为面向消息的中间件设计，基于此协议的客户端与消息中间件可传递消息，并不受产品、开发语言等条件的限制。  
RabbitMQ 最初起源于金融系统，用于在分布式系统中存储转发消息，在易用性、扩展性、高可用性等方面表现不俗。具体特点包括：  

1. 可靠性（Reliability）：RabbitMQ 使用一些机制来保证可靠性，如持久化、传输确认、发布确认。
2. 灵活的路由（Flexible Routing）：在消息进入队列之前，通过 Exchange 来路由消息的。对于典型的路由功能，RabbitMQ 已经提供了一些内置的 Exchange 来实现。针对更复杂的路由功能，可以将多个 Exchange 绑定在一起，也通过插件机制实现自己的 Exchange 。
3. 消息集群（Clustering）：多个 RabbitMQ 服务器可以组成一个集群，形成一个逻辑 Broker 。
4. 高可用（Highly Available Queues）：队列可以在集群中的机器上进行镜像，使得在部分节点出问题的情况下队列仍然可用。
5. 多种协议（Multi-protocol）：RabbitMQ 支持多种消息队列协议，比如 STOMP、MQTT 等等。
6. 多语言客户端（Many Clients）：RabbitMQ 几乎支持所有常用语言，比如 Java、.NET、Ruby 等等。
7. 管理界面（Management UI）:RabbitMQ 提供了一个易用的用户界面，使得用户可以监控和管理消息 Broker 的许多方面。
8. 跟踪机制（Tracing）:如果消息异常，RabbitMQ 提供了消息跟踪机制，使用者可以找出发生了什么。
9. 插件机制（Plugin System）:RabbitMQ 提供了许多插件，来从多方面进行扩展，也可以编写自己的插件。

## RabbitMQ使用场景 ##
1. 跨系统的异步通信，所有需要异步交互的地方都可以使用消息队列。就像我们除了打电话（同步）以外，还需要发短信，发电子邮件（异步）的通讯方式。

2. 多个应用之间的耦合，由于消息是平台无关和语言无关的，而且语义上也不再是函数调用，因此更适合作为多个应用之间的松耦合的接口。基于消息队列的耦合，不需要发送方和接收方同时在线。在企业应用集成（EAI）中，文件传输，共享数据库，消息队列，远程过程调用都可以作为集成的方法。

3. 应用内的同步变异步，比如订单处理，就可以由前端应用将订单信息放到队列，后端应用从队列里依次获得消息处理，高峰时的大量订单可以积压在队列里慢慢处理掉。由于同步通常意味着阻塞，而大量线程的阻塞会降低计算机的性能。

4. 消息驱动的架构（EDA），系统分解为消息队列，和消息制造者和消息消费者，一个处理流程可以根据需要拆成多个阶段（Stage），阶段之间用队列连接起来，前一个阶段处理的结果放入队列，后一个阶段从队列中获取消息继续处理。

5. 应用需要更灵活的耦合方式，如发布订阅，比如可以指定路由规则。

6. 跨局域网，甚至跨城市的通讯（CDN行业），比如北京机房与广州机房的应用程序的通信。

## RabbitMQ架构 ##
![](https://i.imgur.com/XOakoMt.png)  

- 重要角色 :
	- **生产者(publisher)**:消息的创建者，负责创建和推送数据到消息服务器,也是一个向交换器发布消息的客户端应用程序；
	- **消费者(consumer)**: 消息的接收方，用于处理数据和确认消息,表示一个从消息队列中取得消息的客户端应用程序;
	- **代理**:就是 RabbitMQ 本身，用于扮演“快递”的角色，本身不生产消息，只是扮演“快递”的角色

- 重要组件 :
	- **ConnectionFactory(连接管理器)** ：应用程序与Rabbit之间建立连接的管理器，程序代码中使用;
	- **Channel(信道)** ：消息推送使用的通道,多路复用连接中的一条独立的双向数据流通道.信道是建立在真实的TCP连接内地虚拟连接,AMQP命令都是通过信道发出去的,不管是发布消息,订阅队列,还是接受消息,这些动作都是通过信道完成.因为对于操作系统来说建立和销毁TCP都是非常昂贵的开销,所以引入了信道的概念,以复用一条TCP连接;
	- **exchange(交换器)** : 用来接收生产者发送的消息并将这些消息路由给服务器的队列;
	- **queue(消息队列)** : 用来保存消息知道发送给消费者.它是消息的容器,也是消息的终点.一个消息可投入一个或多个队列.消息一直在队列里面.等待消费者连接到这个队列将其取走;	
	- **RoutingKey(路由键)** ：用于把生成者的数据分配到交换器上;
	- **BindingKey(绑定键)** : 消息队列和交换器之间的关联.一个绑定就是基于路由键将交换器和消息队列连接起来的路由规则,所以可以将交换器理解成一个由绑定构成的路由表;
	- **Message(消息)** : 消息是不具名的,它由消息头和消息体组成.消息体是不透明的,而消息头则由一系列的可选属性组成,这些属性包括routing-key(路由键),priority(相对于其他消息的优先权),delivery-mode(指出该消息可能需要持久性存储)等;
	- **virtual host(虚拟主机)** : 表示一批交换器,消息队列和相关对象.虚拟主机是共享相同的身份认证和加密环境的独立服务器域;
	- **broker** : 表示消息对类服务器实体.

### RabbitMQ的消息是怎么发送的 ###
首先客户端必须连接到RabbitMQ服务器才能发布和消费消息,客户端和Rabbit Server 之间会创建一个tcp连接,一旦tcp打开并通过了认证(认证就是你发送给rabbit服务器的用户名和密码),你的客户端和RabbitMQ就创建了一条amqp信道(channel),信道是创建在"真实"tcp上的虚拟连接,amqp命令都是通过信道发送出去的,每个信道都会有一个唯一的id,不论是发布消息,订阅队列都是通过这个信道完成的.

### RabbitMQ怎么避免消息丢失 ###
- 消息持久化
	- 保证消息持久化成功必须满足以下条件:  
		1. 声明队列必须设置持久化durable设置为true.
		2. 消息推送投递模式必须设置持久化,deliverMode设置为2(持久)
		3. 消息已经到达持久化交换器
		4. 消息已经到达持久化队列
	- 持久化的缺点  
		1. 降低了服务器的吞吐量,因为使用的是磁盘而非内存存储,从而降低了吞吐量.可尽量使用ssd硬盘来缓解吞吐量的问题
- ack确认机制
- 设置集群镜像模式
- 消息补偿机制


### RabbitMQ中virtual host(vhost)的作用   ###
vhost可以理解为虚拟broker即mini-RabbitMQ server。其内部均含有独立的queue、exchange和binding等，但最最重要的是，其拥有独立的权限系统，可以做到vhost范围的用户控制。当然，从RabbitMQ的全局角度，vhost可以作为不同权限隔离的手段(一个典型的例子就是不同的应用可以跑在不同的vhost中)

### rabbitmq 怎么保证消息的稳定性 ###
通过将channel设置为confirm(确认)模式,保证消息的稳定性


## Exchange Types(RabbitMQ广播类型) ##
Procuder Publish的Message进入了Exchange.接着通过"routing keys",RabbitMQ会找到应该把这个Message放到哪个queue里.queue也是通过这个"routing keys"来做的绑定.  
有三种类型的Exchanges:direct,fanout,topic.每个实现不同的路由算法(routing algorithm).  	
	
- Direct exchange：如果 routing key 匹配，那么Message就会被传递到相应的queue中。其实在queue创建时，它会自动的以queue的名字作为routing key来绑定那个exchange。
- Fanout exchange： 会向响应的queue广播。
- Topic exchange：对key进行模式匹配，比如ab可以传递到所有ab的queue;   

1. direct键(routing key)分布:  
direct类型的Exchange路由规则也很简单，它会把消息路由到那些Binding key与Routing key完全匹配的Queue中.  
![](https://i.imgur.com/cXPkqoH.png) 
以上图的配置为例，我们以routingKey="error"发送消息到Exchange，则消息会路由到Queue1（amqp.gen-S9b…，这是由RabbitMQ自动生成的Queue名称）和Queue2（amqp.gen-Agl…）；如果我们以Routing Key="info"或routingKey="warning"来发送消息，则消息只会路由到Queue2。如果我们以其他Routing Key发送消息，则消息不会路由到这两个Queue中.

2. fanout(广播分发)  
fanout类型的Exchange路由顾泽非常简单,它会把所有发送到该Exchange的消息路由到所有与它绑定的Queue中.    
![](https://i.imgur.com/VjbfcfP.png)  
上图中,生产者(p)发送到EXchange(x)的所有消息都会路由到图中的两个Queue,并最终被两个消费者(C1与C2)消费.

3. topic交换器(模式匹配)  
前面讲到direct类型的Exchange路由规则是完全匹配Binding Key与Routing Key，但这种严格的匹配方式在很多情况下不能满足实际业务需求。topic类型的Exchange在匹配规则上进行了扩展，它与direct类型的Exchage相似，也是将消息路由到Binding Key与Routing Key相匹配的Queue中，但这里的匹配规则有些不同，它约定： 
Routing Key为一个句点号"."分隔的字符串(我们将被句点号"."分隔开的每一段独立的字符串称为一个单词)，如"stock.usd.nyse"、"nyse.vmw"、"quick.orange.rabbit".Binding Key与Routing Key一样也是句点号"."分隔的字符串。 
Binding Key中可以存在两种特殊字符"*"与"#"，用于做模糊匹配，其中"*"用于匹配一个单词，"#"用于匹配多个单词(可以是零个).
![](https://i.imgur.com/Czpa6Bl.png)  
以上图中的配置为例，routingKey="quick.orange.rabbit"的消息会同时路由到Q1与Q2，routingKey="lazy.orange.fox"的消息会路由到Q1，routingKey="lazy.brown.fox"的消息会路由到Q2，routingKey="lazy.pink.rabbit"的消息会路由到Q2(只会投递给Q2一次，虽然这个routingKey与Q2的两个bindingKey都匹配) routingKey="quick.brown.fox"、routingKey="orange"、routingKey="quick.orange.male.rabbit"的消息将会被丢弃，因为它们没有匹配任何bindingKey

4. headers  
headers类型的Exchange不依赖于Routing Key与Binding Key的匹配规则来路由消息，而是根据发送的消息内容中的headers属性进行匹配.  
在绑定Queue与Exchange时指定一组键值对；当消息发送到Exchange时，RabbitMQ会取到该消息的headers（也是一个键值对的形式），对比其中的键值对是否完全匹配Queue与Exchange绑定时指定的键值对。如果完全匹配则消息会路由到该Queue，否则不会路由到该Queue.
该类型的Exchange没有用到过（不过也应该很有用武之地），所以不做介绍  

## RPC ##
MQ本身是基于异步的消息处理，前面的示例中所有的生产者(P)将消息发送到RabbitMQ后不会知道消费者（C）处理成功或者失败(甚至连有没有消费者来处理这条消息都不知道)  
但实际的应用场景中，我们很可能需要一些同步处理，需要同步等待服务端将我的消息处理完成后再进行下一步处理。这相当于RPC（Remote Procedure Call，远程过程调用）。在RabbitMQ中也支持RPC 
   
![](https://i.imgur.com/D96xyE4.png)   
 
**RabbitMQ中实现PRC的机制是:**  
客户端发送请求(消息)时,在消息的属性(Message Properties,在AMQP协议中定义了14种properties,这些属性会随着消息一起发送的)中设置两个值replyTo(一个Queue名称,用于告诉服务器处理完成后将通知我的消息发送到这个Queue中)和correlationld(此次请求的标识号,服务器处理完成后需要将次属性返还,客户端将根据这个id了解哪条请求被成功执行了或执行失败).服务器端收到消息处理完后,将生成一条应答消息到replyTo指定的Queue,同时带上correlationld属性.饥饿护短之前已订阅replyTo指定的Queue,从中收到服务器的应答消息后,根据其中的correlationld属性分析哪条请求被执行了,根据执行结果进行后续业务处理.

## 使用ACK确认Message的正确传递 ##
默认情况下，如果Message已经被某个Consumer正确的接收到了，那么该Message就会被从Queue中移除。当然也可以让同一个Message发送到很多的Consumer.

如果一个Queue没被任何的Consumer Subscribe(订阅),当有数据到达时，这个数据会被cache，不会被丢弃。当有Consumer时，这个数据会被立即发送到这个Consumer。这个数据被Consumer正确收到时，这个数据就被从Queue中删除.

那么什么是正确收到呢？通过ACK。每个Message都要被acknowledged（确认，ACK）。我们可以显示的在程序中去ACK，也可以自动的ACK。如果有数据没有被ACK，那么RabbitMQ Server会把这个信息发送到下一个Consumer.

如果这个APP有bug，忘记了ACK，那么RabbitMQ Server不会再发送数据给它，因为Server认为这个Consumer处理能力有限。而且ACK的机制可以起到限流的作用(Benefitto throttling)：在Consumer处理完成数据后发送ACK，甚至在额外的延时后发送ACK，将有效的balance Consumer的load.

当然对于实际的例子，比如我们可能会对某些数据进行merge，比如merge 4s内的数据，然后sleep 4s后再获取数据。特别是在监听系统的state，我们不希望所有的state实时的传递上去，而是希望有一定的延时。这样可以减少某些IO，而且终端用户也不会感觉到

## RabbitMQ实现延迟消息队列 ##
通过消息过期后进入死信交换器,再由交换器转发到延迟消费队列,实现延迟功能,使用RabbitMQ-delayed-message-exchange 插件实现延迟功能.

## RabbitMQ集群的作用 ##
- 高可用 : 某个服务器出现问题,整个RabbitMQ还可以继续使用;  
- 高容量 : 集群可以承载更多的消息量.

## RabbitMQ节点的类型 ##
-磁盘节点 : 消息会存储到磁盘
-内存节点 : 消息都存储到内存中,重启服务消息丢失,性能高于磁盘类型

## RabbitMQ集群搭建需要注意问题 ##
各节点之间使用"-link"连接,此属性不能忽略;  
各节点使用的erlang cookie值必须相同,此值相当于"秘钥"的功能,用于各节点的认证.整个集群中必须包含一个磁盘节点

## RabbitMQ每个节点是其他接待你的完整拷贝吗? ##
不是,原因有一下两个:  
存储空间的考虑 : 如果每个节点都拥有所有队列的完全拷贝,这样新增节点不但没有新增存储空间,反而增加了更多的冗余数据;  
性能的考虑 : 如果每条消息都需要完整拷贝到每一个集群节点,那新增节点并没有提升处理消息的能力,最多是保持和单节点相同的性能甚至是更糟.

## RabbitMQ集群中唯一一个磁盘节点崩溃了会发生什么情况 ##
如果唯一磁盘的磁盘节点崩溃了,不能进行以下操作:  
- 不能创建队列
- 不能创建交换器
- 不能创建绑定
- 不能添加用户
- 不能更改权限
- 不能添加和删除集群节点

唯一磁盘节点崩溃了,集群是可以保证运行的,但你不能更改任何东西.

## RabbitMQ对集群节点停止顺序有要求吗? ##
RabbitMQ对集群的停止的顺序是有要求的,应该先关闭内存节点,最后在关闭磁盘节点.如果顺序恰好相反的话,可能会造成消息的丢失.
