## mq 高可用性



![è°è°åå¸å¼ç³»ç»çCAPçè®º](https://pic3.zhimg.com/v2-2f26a48f5549c2bc4932fdf88ba4f72f_1440w.jpg)



## **CAP理论概述**

**一个分布式系统最多只能同时满足一致性（Consistency）、可用性（Availability）和分区容错性（Partition tolerance）这三项中的两项**。

## **CAP权衡**

通过CAP理论，我们知道无法同时满足一致性、可用性和分区容错性这三个特性，那要舍弃哪个呢？

> CA without P：如果不要求P（不允许分区），则C（强一致性）和A（可用性）是可以保证的。但其实分区不是你想不想的问题，而是始终会存在，因此CA的系统更多的是允许分区后各子系统依然保持CA。
> CP without A：如果不要求A（可用），相当于每个请求都需要在Server之间强一致，而P（分区）会导致同步时间无限延长，如此CP也是可以保证的。很多传统的数据库分布式事务都属于这种模式。
> AP wihtout C：要高可用并允许分区，则需放弃一致性。一旦分区发生，节点之间可能会失去联系，为了高可用，每个节点只能用本地数据提供服务，而这样会导致全局数据的不一致性。现在众多的NoSQL都属于此类。



### 消息中间件的传输模式**

**1、点对点模型**

点对点模型用于消息生产者和消息消费者之间点到点的通信。消息生产者将消息发送到由某个名字标识的特定消费者。这个名字实际上对于消费服务中的一个队列（Queue），在消息传递给消费者之前它被存储在这个队列中。队列消息可以放在内存中也可以是持久的，以保证在消息服务出现故障时仍然能够传递消息。

传统的点对点消息中间件通常由消息队列服务、消息传递服务、消息队列和消息应用程序接口API组成，其典型的结构如下图所示。

 ![img](https://leanote.com/api/file/getImage?fileId=5acf72b4ab64413fe50026c8)

 特点：

a、每个消息只用一个消费者

b、发送者和接受者没有时间依赖

c、接受者确认消息接受和处理成功

**2、发布-订阅模型(Pub/Sub)**

 发布者/订阅者模型支持向一个特定的消息主题生产消息。0或多个订阅者可能对接收来自特定消息主题的消息感兴趣。在这种模型下，发布者和订阅者彼此不知道对方。这种模式就好比是匿名公告板。这种模式被概况为：多个消费者可以获得消息，在发布者和订阅者之间存在时间依赖性。发布者需要建立一个订阅（subscription），以便能够消费者订阅。订阅者必须保持持续的活动状态及接收消息，除非订阅者建立了持久的订阅。在这种情况下，在订阅者未连接时发布的消息将在订阅者重新连接时重新发布。

 **发布订阅模型特性**

1）每个消息有多个订阅者
2）客户端只有订阅后才能接受到消息
如下图，当有消息更新后会同时更新订阅者，并且一个消息会有多个订阅者，如下图：

![img](https://www.linuxea.com/action/Watermark?mark=L3Vzci91cGxvYWRzLzIwMTYvMTIvMzA1MzQ5MTcyOS5wbmc=)

 3）持久订阅和非持久订阅
如下图，如果此时持久订阅出现故障，在故障后重启则会重新发送一次消息，如果非持久订阅则不会，消息则错过，如下图:
![img](https://www.linuxea.com/action/Watermark?mark=L3Vzci91cGxvYWRzLzIwMTYvMTIvMzQ1MDI5NTY5Ni5wbmc=)

消费者订阅后，中间件会投递给消费者，消费者确认后在返回给中间件，如下图:

 

![img](https://leanote.com/api/file/getImage?fileId=5acf72c9ab644141db0021f9)



### 参考模板:

1. 消息的可靠性(暂时不考虑)
   1. ack机制
   2. 重发
2. 镜像(replication) 集群 (横向扩容, 不考虑)
   1. 消息的复制(异步ack)
   2. 消息的存储(内存或者异步刷硬盘)
3. consumer从哪里读取数据(暂时不考虑))
4. broker leader 宕机.  通过**选举机制**选取新的leader
   1. 消息会不会丢失
   2. 消息重复投递
   3. 消息的顺序性
5. 宕机的机器重启后(或者新加入的机器), 重新连接leader. 同步数据  
6. 消费者消费数据(暂时不考虑)
   1. 数据的消费特性(at-least-once delivery, 可重复消费等)



### RabbitMQ

2. 采用镜像集群模式,   环形结构 保证消息传递
   1. 消息的生产和消费会在master和slave之间进行同步。同步由master发起，通过GM来确保所有slave完成同步
   2. BQ队列.. 初始默认情况下，非持久化消息直接进入内存队列，此时效率最高，当内存占用逐渐达到一个阈值时，消息和消息索引逐渐往磁盘中移动，随着消费者的不断消费，内存占用的减少，消息逐渐又从磁盘中被转到内存队列中。
   
3. 消费者只依附于master. 消费者确认后消息会删除

4. 当master节点挂掉时，最老的slave节点(根基于 slave 加入 cluster 的时间排序)将被提升为master
   1. 因为最老的 slave 与旧的 master 之间的同步状态应该是最好的。**如果此时所有 slave 处于未同步状态，则未同步的消息会丢失。**
   2. 新的master节点requeue所有unack消息，因为这个新节点无法区分这些unack消息是否已经到达客户端，亦或是ack消息丢失在老的master的链路上，亦或者是丢在master组播ack消息到所有slave的链路上。所以处于消息可靠性的考虑，重新入队所有unack 的消息，**不过此时客户端可能会有重复消息**。
   
5. 每当一个节点加入或者重新加入(例如从网络分区中恢复回来)镜像队列，之前保存的队列内容会被清空。

   

## AMQP 简介

RabbitMQ 是 AMQP（高级消息队列协议）的标准实现：

<img src="https://pic2.zhimg.com/80/v2-95679b747c214fcc1b30b5592177acb1_720w.jpg" alt="img"  />

- Producer：消息生产者，即投递消息的程序。

- Broker：消息队列服务器实体。

- - Exchange：消息交换机，它指定消息按什么规则，路由到哪个队列。
  - Binding：绑定，它的作用就是把 Exchange 和 Queue 按照路由规则绑定起来。
  - Queue：消息队列载体，每个消息都会被投入到一个或多个队列。

- Consumer：消息消费者，即接受消息的程序。

![img](https://pic4.zhimg.com/80/v2-c735dc7d3e0ea74acc7f933abeaa40b7_720w.jpg)



RabbitMQ是比较有代表性的，因为是基于**主从**做高可用性的，我们就以他为例子讲解第一种MQ的高可用性怎么实现。

----

镜像集群模式

这种模式，才是rabbitmq提供是真正的高可用模式，跟普通集群不一样的是，你创建的queue，无论元数据还是queue里面是消息数据都存在多个实例当中，然后每次写消息到queue的时候，都会自动把消息到多个queue里进行消息同步。

![img](https://img-blog.csdn.net/20180514175222427)



这样的话，好处在于，你任何一个机器宕机了，没事儿，别的机器都可以用。

坏处在于，

第一，这个性能开销也太大了吧，消息同步所有机器，导致网络带宽压力和消耗很重！

第二，这么玩儿，就没有扩展性可言了，如果某个queue负载很重，你加机器，新增的机器也包含了这个queue的所有数据，并没有办法线性扩展你的queue

---

## 镜像队列结构

镜像队列中各节点的进程是独立的，进程间通过消息传递来通信。

![img](https://static.oschina.net/uploads/space/2016/1205/201326_LvxE_3088236.png)

BQ: 备份队列.  master , slave 都是BQ的实现

消费者只依附于master。因此，主服务器负责通知所有从服务器何时从bq获取消息、何时断开消息以及何时重新请求消息。

gm: 一个类似于Raft的算法, 但是没有经过理论验证.... 



镜像队列中每个节点都有一个amqqueue_process队列进程，这些进程中有一个master，其余是slave；每个队列进程会启动一个gm（guaranteed multicast）进程，镜像队列中所有gm进程组成一个gm组（gm group）。集群中每个有客户端连接的节点都会启动若干个channel进程，channel进程中记录着镜像队列中master和所有slave进程的Pid，以便直接与队列进程通信。

消息的生产和消费会在master和slave之间进行同步。同步由master发起，通过GM来确保所有slave完成同步

GM保证镜像队列进程组中的进程可以动态添加和删除，发送到进程组的消息在消息的生命周期中，到达进程组中的每个进程。消息的生命周期从消息发送时算起，直到消息发送者了解到消息已经抵达了进程组中的所有进程为止。

## 可靠多播 (Guaranteed Multicast)

广播通常的实现方法是发送者将消息发给集群中的每个节点，这要求集群节点是全联通的（fully connected）。当发送节点挂掉时，消息可能没有发到集群中的每个节点，这就引入了集群中哪些节点要为已挂掉节点负责、继续发送消息的问题。

为简化网络拓扑和实现复杂度，GM组将集群中的节点组成一个环。环中节点挂掉时，很容易知道哪些节点已收到消息、哪些节点没有收到：如果一个节点挂掉，其左右邻居节点将获悉其退出信息，然后最近的上游节点（upstream）负责将挂掉节点未转发的消息（in-flight messages）继续发给最近的下游节点（downstream）。

这就要求GM组中每个节点缓存收到的消息（gm缓存队列），以便当下游节点挂掉时重新转发该消息。消息的最初发送者在收到自身发出的消息时，必须能够再次发出该消息的确认信息（gm ack message）在GM组中同步，这样其他节点才能够将缓存消息删除。当消息最初发送者再次收到确认消息时，说明整个集群已经同步该消息。此时新加入GM组的节点，将不会再收到该消息。

![img](https://static.oschina.net/uploads/space/2016/1205/201344_RxSO_3088236.jpg)

​																												**图2  GM消息传递流程**



GM通过[监控进程](http://www.erlang.org/doc/man/erlang.html#monitor-2)来获悉进程退出：A进程监控B进程，当B不存在时，A直接收到B退出消息（退出原因是进程不存在）；当B运行一段时间后退出时，A同样能收到B退出消息（退出原因是进程退出）。GM组中各节点监控其上下游节点进程。

GM实现中，主要包括以下3方面。

  1. 对于GM组中节点加入和退出的操作，如果这些操作是一个有序的序列，GM就能够判断哪些节点应该继承已挂掉节点的消息。如A->B->C变为A->X->C，如果不确定顺序，将无法判断应该是X继承B的消息，还是C继承B的消息。

     如果顺序确定为A->B->C->D变为A->C->D，B退出时，A和C将获悉退出消息，A将自己的状态（包括缓存的消息、确认消息）发给C，C结合自身状态就能知道哪些消息是B退出时未转发的。

     ![img](https://static.oschina.net/uploads/space/2016/1205/201358_dnKU_3088236.png)

     ​																												**图3  节点退出处理流程**

     2. 对于GM组中新加入的节点，必须先与上游节点同步通信，初始化自身状态（与最近上游节点保持状态一致）。这样这个新节点才能够正确处理后面接收到的确认消息；同时当下游节点退出时，也能够把正确的状态发给新的下游节点。

        如A->B->C变为A->D->B->C，D向A发送加入集群请求，A发送自身状态到D，D将自身状态初始化与A一致，并通知B邻居信息更新。

        ![img](https://static.oschina.net/uploads/space/2016/1205/201410_tsEW_3088236.png)

        ​																											**图4  节点加入处理流程**

        3. 集群中的节点状态（view）通常是保存在数据库中，为了确保节点变化是一个有序的序列，节点每收到一条消息都需要查询数据库来验证当前view是否失效，是否要更新view。这将急剧降低系统性能。

        　　GM的实现中，每个节点都缓存一份view，使用缓存失效机制来确保view的正确性。当节点变化时，其他节点能够根据收到的消息获悉该变化。由于GM将节点变化当做有序序列来处理，view的变化就不能通过GM的机制来确保：将view变化的消息传给环中一个节点时，不能保证该节点是否会挂掉。同时，节点的变化可能导致每个节点拥有不同的view。

        　　所以GM规定当view更新时，会原子性的增加view值（同时用事务更新数据库中节点状态）。节点间传输消息时，会包含节点的view，只有当接收节点持有的view值不小于发送节点的view值时，接收节点才能够正确处理消息；否则，接收节点要先更新其持有的view。

        　　如A->B->C->D->E变为A->E，假设A、B、C、D、E最初都有view值x。B、C、D同时退出时，A获悉B的退出消息，view值更新为x+1；E获悉D的退出消息，view值更新为x+2；然后A监控C，获悉C退出，view值更新为x+3；A继续获取下游节点E，向E发送自身状态，E的view值小于A，所以E先更新自身view（view值将更新为x+3），然后处理消息。（这是一种可能的顺序，真实的顺序按照节点收到退出消息顺序确定。）

     ## Master提升

     GM保证了集群中slave节点挂掉时，消息不会丢失。当master节点挂掉时，最老的slave节点将被提升为master，此时channel发给原master的消息可能并没有通过原master的GM广播出来，所以该场景下需要额外的机制来确保消息不丢失。

     ![img](https://static.oschina.net/uploads/space/2016/1205/201423_qRsw_3088236.png)

     ​																												**图5  生产消息传递路径**

     RabbitMQ通过以下方法解决该问题：channel进程在收到生产者消息后，将消息发送给所有的队列进程，队列进程缓存从channel收到的消息（amqqueue缓存队列）。当队列进程从gm收到该消息后，将消息入BQ队列，并删除缓存消息。当master挂掉时，新的master通过对比gm缓存和amqqueue缓存，获取上述channel已发出、原master未通过gm转发的消息。新master此时会将未确认（consume ack）的消费消息退回BQ队列（requeue），然后将上面获取的原master未转发消息入BQ队列（同时在gm广播进行同步）。

     对于消费消息，channel只用与master通信，master将消费信息在集群中同步。当master挂掉时，新提升的master将未确认的消费消息退回BQ队列（由于新master不知道消费者是否ack，所以只能将消息退回队列）；已确认的消费消息由新master继续在集群中同步，不用退回队列。

　　该机制确保了RabbitMQ的可靠交付（at-least-once delivery）。

### 镜像队列消息同步

​		新节点加入集群时处于未同步状态，默认情况下镜像队列不会自动在集群中同步，所以该情况下可以等待消费者将老的消息取走，直到集群中master队列的内容和新节点一致，新节点才会处于同步状态；或者在节点加入后，手动调用同步命令，对镜像队列强制同步。手动同步时，队列处于阻塞状态，不响应客户端调用。

　　处于未同步状态的节点，收到gm同步的消费信息时，直接传递消息，而不从后端BQ队列取消息。处于同步状态的节点，收到gm同步的消费信息时，先从BQ队列取消息，而后传递。

　　Master节点重启后将变为slave，如果在重启时，集群仍有消息的生产和消费，原master将处于非同步状态，RabbitMQ实现中将该场景当做新节点加入来处理，直接将原master已保存的数据清空；如果在重启时，集群无消息生产和消费，原master将保持同步状态。

　　所以，如果一个集群中节点相继退出，则最后一个退出的节点是master，该节点保存有最新的数据，集群再起启动时，须先启动该节点。

　　需要注意的是镜像队列中没有已同步节点时，如果Master正常退出(rabbitmqctl stop)，队列将处于Down状态，服务不可用；如果Master异常退出(crash、kill、宕机)时，RabbitMQ提升最老的Slave为Master，此时数据可能有丢失。



初始默认情况下，非持久化消息直接进入内存队列，此时效率最高，当内存占用逐渐达到一个阈值时，消息和消息索引逐渐往磁盘中移动，随着消费者的不断消费，内存占用的减少，消息逐渐又从磁盘中被转到内存队列中。

![img](https://images2018.cnblogs.com/blog/1253350/201806/1253350-20180608102522017-95525574.jpg)





### 文章链接: https://my.oschina.net/hackandhike/blog/800334

---

## kafka

1. 消息的可靠性(暂时不考虑)
   1. ack机制
   2. 重发
2. 镜像(replication) 集群 (横向扩容, 不考虑)
   1. 消息的复制 同步或者异步ack,   HW, LEO
   2. 消息的存储 顺序写磁盘
3. 只会从 leader 去读，但是只有当一个消息已经被所有 follower 都同步成功返回 ack 的时候，这个消息才会被消费者读到。
   1. Kafka并不删除已消费的消息
4. broker leader 宕机.  通过**选举机制**选取新的leader.  Kafka在Zookeeper中动态维护了一个ISR（in-sync replicas），这个ISR里的所有Replica都跟上了leader，只有ISR里的成员才有被选为Leader的可能。
   1. 消息会不会丢失  (异步ack会丢失数据, 但是如果生产者有重试机制, 则不会丢失)
   2. 消息会重复投递  (理论上会, 因为ISR不一定最新)
   3. 消息的顺序性  (partion保证)
5. 宕机的机器重启后(或者新加入的机器), 重新连接leader. 同步数据  
   1. 根据leader epocher 来处理是否截断
6. 消费者消费数据(暂时不考虑)
   1. 数据的消费特性(保证 at-least-once delivery)



![img](http://cdn.17coding.info/WeChat%20Screenshot_20190325215237.png)

**Producer**：Producer即生产者，消息的产生者，是消息的入口。
　　**kafka cluster**：
　　　　**Broker**：Broker是kafka实例，每个服务器上有一个或多个kafka的实例，我们姑且认为每个broker对应一台服务器。每个kafka集群内的broker都有一个**不重复**的编号，如图中的broker-0、broker-1等……
　　　　**Topic**：消息的主题，可以理解为消息的分类，kafka的数据就保存在topic。在每个broker上都可以创建多个topic。
　　　　**Partition**：Topic的分区，每个topic可以有多个分区，分区的作用是做负载，提高kafka的吞吐量。同一个topic在不同的分区的数据是不重复的，partition的表现形式就是一个一个的文件夹！
　　　　**Replication**:每一个分区都有多个副本，副本的作用是做备胎。当主分区（Leader）故障的时候会选择一个备胎（Follower）上位，成为Leader。在kafka中默认副本的最大数量是10个，且副本的数量不能大于Broker的数量，follower和leader绝对是在不同的机器，同一机器对同一个分区也只可能存放一个副本（包括自己）。
　　　　**Message**：每一条发送的消息主体。
　　**Consumer**：消费者，即消息的消费方，是消息的出口。
　　**Consumer Group**：我们可以将多个消费组组成一个消费者组，在kafka的设计中同一个分区的数据只能被消费者组中的某一个消费者消费。同一个消费者组的消费者可以消费同一个topic的不同分区的数据，这也是为了提高kafka的吞吐量！
　　**Zookeeper**：kafka集群依赖zookeeper来保存集群的的元信息，来保证系统的可用性。



Kafka 一个最基本的架构认识：由多个 broker 组成，每个 broker 是一个节点；你创建一个 topic，这个 topic 可以划分为多个 partition，每个 partition 可以存在于不同的 broker 上，每个 partition 就放一部分数据。这就是**天然的分布式消息队列**，就是说一个 topic 的数据，是**分散放在多个机器上的，每个机器就放一部分数据**。

## 为何需要Replication

　　在Kafka在0.8以前的版本中，是没有Replication的，一旦某一个Broker宕机，则其上所有的Partition数据都不可被消费，这与Kafka数据持久性及Delivery Guarantee的设计目标相悖。同时Producer都不能再将数据存于这些Partition中。

- 如果Producer使用同步模式则Producer会在尝试重新发送`message.send.max.retries`（默认值为3）次后抛出Exception，用户可以选择停止发送后续数据也可选择继续选择发送。而前者会造成数据的阻塞，后者会造成本应发往该Broker的数据的丢失。
- 如果Producer使用异步模式，则Producer会尝试重新发送`message.send.max.retries`（默认值为3）次后记录该异常并继续发送后续数据，这会造成数据丢失并且用户只能通过日志发现该问题。

　　由此可见，在没有Replication的情况下，一旦某机器宕机或者某个Broker停止工作则会造成整个系统的可用性降低。随着集群规模的增加，整个集群中出现该类异常的几率大大增加，因此对于生产系统而言Replication机制的引入非常重要。 　　



Kafka 0.8 以后，提供了 HA 机制，就是 replica（复制品） 副本机制。每个 partition 的数据都会同步到其它机器上，形成自己的多个 replica 副本。所有 replica 会选举一个 leader 出来，那么生产和消费都跟这个 leader 打交道，然后其他 replica 就是 follower。写的时候，leader 会负责把数据同步到所有 follower 上去，读的时候就直接读 leader 上的数据即可。只能读写 leader？很简单，**要是你可以随意读写每个 follower，那么就要 care 数据一致性的问题**，系统复杂度太高，很容易出问题。Kafka 会均匀地将一个 partition 的所有 replica 分布在不同的机器上，这样才可以提高容错性。

- **高可用性**：因为如果某个 broker 宕机了，没事儿，那个 broker上面的 partition 在其他机器上都有副本的。如果这个宕机的 broker 上面有某个 partition 的 leader，那么此时会从 follower中**重新选举**一个新的 leader 出来，大家继续读写那个新的 leader 即可。这就有所谓的高可用性了。
- **写数据**：生产者就写 leader，然后 leader 将数据落地写本地磁盘，接着其他 follower 自己主动从 leader 来 pull 数据。一旦所有 follower 同步好数据了，就会发送 ack 给 leader，leader 收到所有 follower的 ack 之后，就会返回写成功的消息给生产者。（当然，这只是其中一种模式，还可以适当调整这个行为）
- **消费**：只会从 leader 去读，但是只有当一个消息已经被所有 follower 都同步成功返回 ack 的时候，这个消息才会被消费者读到。



一个topic为一类消息，每条消息必须指定一个topic。物理上，一个topic分成一个或多个partition，每个partition有多个副本分布在不同的broker中，如下图3.1。

![img](https://pic3.zhimg.com/80/v2-604f846886c49517144f971cdf9c24c2_720w.jpg)



###### 选择Partition写入

1. hash

   ![image-20200622115348372](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20200622115348372.png)

2. 顺序

![image-20200622115401036](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20200622115401036.png)



每个partition在存储层面是一个append log文件，发布到此partition的消息会追加到log文件的尾部，为顺序写人磁盘（顺序写磁盘比随机写内存的效率还要高）。每条消息在log文件中的位置成为offset（偏移量），offset为一个long型数字，唯一标记一条消息。如下图3.2

![img](https://pic4.zhimg.com/80/v2-05e568493577dea782a1241db8b827cf_720w.jpg)

Topic -> Partition 物理上

```shell
drwxr-xr-x 2 root root 4096 Apr 10 16:10 topic_vms_test-0
drwxr-xr-x 2 root root 4096 Apr 10 16:10 topic_vms_test-1
drwxr-xr-x 2 root root 4096 Apr 10 16:10 topic_vms_test-2
drwxr-xr-x 2 root root 4096 Apr 10 16:10 topic_vms_test-3
```



每个partition上

上面提到 partition 还可以细分为 segment，这个 segment 又是什么？如果就以 partition 为最小存储单位，我们可以想象当 Kafka producer 不断发送消息，必然会引起 partition 文件的无限扩张，这样对于消息文件的维护以及已经被消费的消息的清理带来严重的影响，所以这里以 segment 为单位又将 partition 细分。每个 partition(目录) 相当于一个巨型文件被平均分配到多个大小相等的 segment(段) 数据文件中（每个 segment 文件中消息数量不一定相等）这种特性也方便 old segment 的删除，即方便已被消费的消息的清理，提高磁盘的利用率。每个 partition 只需要支持顺序读写就行，segment 的文件生命周期由服务端配置参数（log.segment.bytes，log.roll.{ms,hours}等若干参数）决定。

segment 文件由两部分组成，分别为“.index”文件和“.log”文件，分别表示为 segment 索引文件和数据文件。这两个文件的命令规则为：partition 全局的第一个 segment 从 0 开始，后续每个 segment 文件名为上一个 segment 文件最后一条消息的 offset 值，数值大小为 64 位，20 位数字字符长度，没有数字用 0 填充，如下：

```shell
00000000000000000000.index
00000000000000000000.log
00000000000000000000.timeindex（时间的索引）
00000000000000170410.index
00000000000000170410.log
00000000000000239430.index
00000000000000239430.log
```



![Kafkaæ°æ®å¯é æ§æ·±åº¦è§£è¯»](https://static001.infoq.cn/resource/image/e4/96/e4367c6b0a775b9b1c2862756a0bb896.jpg)

如上图，“.index”索引文件存储大量的元数据，“.log”数据文件存储大量的消息，索引文件中的元数据指向对应数据文件中 message 的物理偏移地址。其中以“.index”索引文件中的元数据 [3, 348] 为例，在“.log”数据文件表示第 3 个消息，即在全局 partition 中表示 170410+3=170413 个消息，该消息的物理偏移地址为 348。

那么如何从 partition 中通过 offset 查找 message 呢？以上图为例，读取 offset=170418 的消息，首先查找 segment 文件，其中 00000000000000000000.index 为最开始的文件，第二个文件为 00000000000000170410.index（起始偏移为 170410+1=170411），而第三个文件为 00000000000000239430.index（起始偏移为 239430+1=239431），所以这个 offset=170418 就落到了第二个文件之中。其他后续文件可以依次类推，以其实偏移量命名并排列这些文件，然后根据二分查找法就可以快速定位到具体文件位置。其次根据 00000000000000170410.index 文件中的 [8,1325] 定位到 00000000000000170410.log 文件中的 1325 的位置进行读取。

###### 要是读取 offset=170418 的消息，从 00000000000000170410.log 文件中的 1325 的位置进行读取，那么怎么知道何时读完本条消息，否则就读到下一条消息的内容了？这个就需要联系到消息的物理结构了，消息都具有固定的物理结构，包括：offset（8 Bytes）、消息体的大小（4 Bytes）、crc32（4 Bytes）、magic（1 Byte）、attributes（1 Byte）、key length（4 Bytes）、key（K Bytes）、payload(N Bytes) 等等字段，可以确定一条消息的大小，即读取到哪里截止。

默认情况下，有个参数log.index.interval.bytes限定了在日志文件写入多少数据，就要在索引文件写一条索引，默认是4KB，写4kb的数据然后在索引里写一条索引，所以索引本身是稀疏格式的索引，不是每条数据对应一条索引的。而且索引文件里的数据是按照位移和时间戳升序排序的，所以kafka在查找索引的时候，会用二分查找，时间复杂度是O(logN)，找到索引，就可以在.log文件里定位到数据了。

![img](https://user-gold-cdn.xitu.io/2019/11/27/16ea9b355ecdc779?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

上面的0，2039···这些代表的是物理位置。为什么稀松索引会比直接一条条读取速度快，它不是每一条数据都会记录，是相隔几条数据的记录方式，但是就比如现在要消费偏移量为7的数据，就直接先看这个稀松索引上的记录，找到一个6时，7比6大，然后直接看后面的数据，找到8，8比7大，再看回来，确定7就是在6~8之间，而6的物理位置在9807,8的物理位置在12345，直接从它们中间去找。就提升了查找物理位置的速度。就类似于普通情况下的二分查找。



### 复制原理和同步方式

Kafka 中 topic 的每个 partition 有一个预写式的日志文件，虽然 partition 可以继续细分为若干个 segment 文件，但是对于上层应用来说可以将 partition 看成最小的存储单元（一个有多个 segment 文件拼接的“巨型”文件），每个 partition 都由一些列有序的、不可变的消息组成，这些消息被连续的追加到 partition 中。

![Kafkaæ°æ®å¯é æ§æ·±åº¦è§£è¯»](https://static001.infoq.cn/resource/image/3c/fc/3cf328620b54a8f3e7e8cb8af35898fc.jpg)

上图中有两个新名词：HW 和 LEO。这里先介绍下 LEO，LogEndOffset 的缩写，表示每个 partition 的 log 最后一条 Message 的位置。HW 是 HighWatermark 的缩写，是指 consumer 能够看到的此 partition 的位置，这个涉及到多副本的概念，这里先提及一下，下节再详表。

言归正传，为了提高消息的可靠性，Kafka 每个 topic 的 partition 有 N 个副本（replicas），其中 N(大于等于 1) 是 topic 的复制因子（replica fator）的个数。Kafka 通过多副本机制实现故障自动转移，当 Kafka 集群中一个 broker 失效情况下仍然保证服务可用。在 Kafka 中发生复制时确保 partition 的日志能有序地写到其他节点上，N 个 replicas 中，其中一个 replica 为 leader，其他都为 follower, leader 处理 partition 的所有读写请求，与此同时，follower 会被动定期地去复制 leader 上的数据

如下图所示，Kafka 集群中有 4 个 broker, 某 topic 有 3 个 partition, 且复制因子即副本个数也为 3：

![img](https://pic3.zhimg.com/80/v2-604f846886c49517144f971cdf9c24c2_720w.jpg)

Kafka 提供了数据复制算法保证，如果 leader 发生故障或挂掉，一个新 leader 被选举并被接受客户端的消息成功写入。Kafka 确保从同步副本列表中选举一个副本为 leader，或者说 follower 追赶 leader 数据。leader 负责维护和跟踪 ISR(In-Sync Replicas 的缩写，表示副本同步队列，具体可参考下节) 中所有 follower 滞后的状态。当 producer 发送一条消息到 broker 后，leader 写入消息并复制到所有 follower。消息提交之后才被成功复制到所有的同步副本。消息复制延迟受最慢的 follower 限制，重要的是快速检测慢副本，如果 follower“落后”太多或者失效，leader 将会把它从 ISR 中删除

### ISR

上节我们涉及到 ISR (In-Sync Replicas)，这个是指副本同步队列。副本数对 Kafka 的吞吐率是有一定的影响，但极大的增强了可用性。默认情况下 Kafka 的 replica 数量为 1，即每个 partition 都有一个唯一的 leader，为了确保消息的可靠性，通常应用中将其值 (由 broker 的参数 offsets.topic.replication.factor 指定) 大小设置为大于 1，比如 3。 所有的副本（replicas）统称为 Assigned Replicas，即 AR。

ISR 是 AR 中的一个子集，由 leader 维护 ISR 列表，follower 从 leader 同步数据有一些延迟（包括延迟时间 replica.lag.time.max.ms 和延迟条数 replica.lag.max.messages 两个维度, 当前最新的版本 0.10.x 中只支持 replica.lag.time.max.ms 这个维度），任意一个超过阈值都会把 follower 剔除出 ISR, 存入 OSR（Outof-Sync Replicas）列表，新加入的 follower 也会先存放在 OSR 中。AR=ISR+OSR。

(Kafka 0.10.x 版本后移除了 replica.lag.max.messages 参数，只保留了 replica.lag.time.max.ms 作为 ISR 中副本管理的参数。为什么这样做呢？replica.lag.max.messages 表示当前某个副本落后 leaeder 的消息数量超过了这个参数的值，那么 leader 就会把 follower 从 ISR 中删除...... 于是就会出现它们不断地剔出 ISR 然后重新回归 ISR，这无疑增加了无谓的性能损耗    ...这里省略)

上面一节还涉及到一个概念，即 HW。**HW 俗称高水位，HighWatermark 的缩写，取一个 partition 对应的 ISR 中最小的 LEO 作为 HW，consumer 最多只能消费到 HW 所在的位置。另外每个 replica 都有 HW,leader 和 follower 各自负责更新自己的 HW 的状态。对于 leader 新写入的消息，consumer 不能立刻消费，leader 会等待该消息被所有 ISR 中的 replicas 同步后更新 HW，此时消息才能被 consumer 消费。这样就保证了如果 leader 所在的 broker 失效，该消息仍然可以从新选举的 leader 中获取。**对于来自内部 broKer 的读取请求，没有 HW 的限制。

下图详细的说明了当 producer 生产消息至 broker 后，ISR 以及 HW 和 LEO 的流转过程：

###### (保证4,5 完整写入, 需要Ack给生产者)

![Kafkaæ°æ®å¯é æ§æ·±åº¦è§£è¯»](https://static001.infoq.cn/resource/image/52/f2/5287520018eb8aea0bbcd5049a080df2.jpg)

---

**HW 是如何更新的呢? **

follower在和leader同步数据的时候，同步过来的数据会带上LEO的值，可是在**实际情况中有可能p0的副本可能不仅仅只有2个**。此时我画多几个follower(p0)，它们也向leader partition同步数据，带上自己的LEO。**leader partition就会记录这些follower同步过来的LEO，然后取最小的LEO值作为HW值**

这个做法是保证了如果leader partition宕机，集群会从其它的follower partition里面选举出一个新的leader partition。这时候无论选举了哪一个节点作为leader，都能保证存在此刻待消费的数据，保证数据的安全性。

那么follower自身的HW的值如何确定，那就是**follower获取数据时也带上leader partition的HW的值，然后和自身的LEO值取一个较小的值作为自身的HW值**。

---

现在你再回想一下之前提到的ISR，是不是就更加清楚了。**follower如果超过10秒没有到leader这里同步数据，就会被踢出ISR**。它的作用就是帮助我们在leader宕机时快速再选出一个leader，因为在ISR列表中的follower都是和leader同步率高的，就算丢失数据也不会丢失太多。

而且我们之前没提到什么情况下follower可以返回ISR中，现在解答，当**follower的LEO值>=leader的HW值，就可以回到ISR**。

![img](https://user-gold-cdn.xitu.io/2019/11/30/16ebbe829bfedb8b?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)



Kafka 的 ISR 的管理最终都会反馈到 Zookeeper 节点上。具体位置为：/brokers/topics/[topic]/partitions/[partition]/state。目前有两个地方会对这个 Zookeeper 的节点进行维护：

1. Controller 来维护：Kafka 集群中的其中一个 Broker 会被选举为 Controller，主要负责 Partition 管理和副本状态管理，也会执行类似于重分配 partition 之类的管理任务。在符合某些特定条件下，Controller 下的 LeaderSelector 会选举新的 leader，ISR 和新的 leader_epoch 及 controller_epoch 写入 Zookeeper 的相关节点中。同时发起 LeaderAndIsrRequest 通知所有的 replicas。
2. **leader 来维护：leader 有单独的线程定期检测 ISR 中 follower 是否脱离 ISR, 如果发现 ISR 变化，则会将新的 ISR 的信息返回到 Zookeeper 的相关节点中。**

### 数据可靠性和持久性保证

当 producer 向 leader 发送数据时，可以通过 request.required.acks 参数来设置数据可靠性的级别：

- 1（默认）：这意味着 producer 在 ISR 中的 leader 已成功收到的数据并得到确认后发送下一条 message。如果 leader 宕机了，则会丢失数据。
- 0：这意味着 producer 无需等待来自 broker 的确认而继续发送下一批消息。这种情况下数据传输效率最高，但是数据可靠性确是最低的。
- -1：producer 需要等待 ISR 中的所有 follower 都确认接收到数据后才算一次发送完成，可靠性最高。但是这样也不能保证数据不丢失，比如当 ISR 中只有 leader 时（前面 ISR 那一节讲到，ISR 中的成员由于某些情况会增加也会减少，最少就只剩一个 leader），这样就变成了 acks=1 的情况

如果要提高数据的可靠性，在设置 request.required.acks=-1 的同时，也要 min.insync.replicas 这个参数 (可以在 broker 或者 topic 层面进行设置) 的配合，这样才能发挥最大的功效。min.insync.replicas 这个参数设定 ISR 中的最小副本数是多少，默认值为 1，当且仅当 request.required.acks 参数设置为 -1 时，此参数才生效。如果 ISR 中的副本数少于 min.insync.replicas 配置的数量时，客户端会返回异常：org.apache.kafka.common.errors.NotEnoughReplicasExceptoin: Messages are rejected since there are fewer in-sync replicas than required。



接下来对 acks=1 和 -1 的两种情况进行详细分析：

**1. request.required.acks=1**

producer 发送数据到 leader，leader 写本地日志成功，返回客户端成功；此时 ISR 中的副本还没有来得及拉取该消息，leader 就宕机了，那么此次发送的消息就会丢失。

![Kafkaæ°æ®å¯é æ§æ·±åº¦è§£è¯»](https://static001.infoq.cn/resource/image/fc/bd/fcc432e0e30c1a815a09123952145cbd.jpg)



**2. request.required.acks=-1**

同步（Kafka 默认为同步，即 producer.type=sync）的发送模式，replication.factor>=2 且 min.insync.replicas>=2 的情况下，不会丢失数据。

有两种典型情况。acks=-1 的情况下（如无特殊说明，以下 acks 都表示为参数 request.required.acks），数据发送到 leader, ISR 的 follower 全部完成数据同步后，leader 此时挂掉，那么会选举出新的 leader，数据不会丢失。

![Kafkaæ°æ®å¯é æ§æ·±åº¦è§£è¯»](https://static001.infoq.cn/resource/image/28/45/28ab66a12966fee279381f83c67b2745.jpg)

acks=-1 的情况下，数据发送到 leader 后 ，部分 ISR 的副本同步，leader 此时挂掉。比如 follower1h 和 follower2 都有可能变成新的 leader, producer 端会得到返回异常，producer 端会重新发送数据，数据可能会重复。

![Kafkaæ°æ®å¯é æ§æ·±åº¦è§£è¯»](https://static001.infoq.cn/resource/image/68/97/68fc7c2fb0c4032e119133db5abc8897.jpg)

当然上图中如果在 leader crash 的时候，follower2 还没有同步到任何数据，而且 follower2 被选举为新的 leader 的话，这样消息就不会重复。

### 3.5 关于 HW 的进一步探讨

考虑上图（即 acks=-1, 部分 ISR 副本同步）中的另一种情况，如果在 Leader 挂掉的时候，follower1 同步了消息 4,5，follower2 同步了消息 4，与此同时 follower2 被选举为 leader，那么此时 follower1 中的多出的消息 5 该做如何处理呢？

这里就需要 HW 的协同配合了。如前所述，**一个 partition 中的 ISR 列表中，leader 的 HW 是所有 ISR 列表里副本中最小的那个的 LEO。**类似于木桶原理，水位取决于最低那块短板。

![Kafkaæ°æ®å¯é æ§æ·±åº¦è§£è¯»](https://static001.infoq.cn/resource/image/3a/67/3a63cea8900ea96235171a3d7432d667.jpg)

如上图，某个 topic 的某 partition 有三个副本，分别为 A、B、C。A 作为 leader 肯定是 LEO 最高，B 紧随其后，C 机器由于配置比较低，网络比较差，故而同步最慢。这个时候 A 机器宕机，这时候如果 B 成为 leader，假如没有 HW，在 A 重新恢复之后会做同步 (makeFollower) 操作，在宕机时 log 文件之后直接做追加操作，而假如 B 的 LEO 已经达到了 A 的 LEO，会产生数据不一致的情况，所以使用 HW 来避免这种情况。A 在做同步操作的时候，先将 log 文件截断到之前自己的 HW 的位置，即 3，之后再从 B 中拉取消息进行同步。

如果失败的 follower 恢复过来，它首先将自己的 log 文件截断到上次 checkpointed 时刻的 HW 的位置，之后再从 leader 中同步消息。leader 挂掉会重新选举，新的 leader 会发送“指令”让其余的 follower 截断至自身的 HW 的位置然后再拉取新的消息。

###### 当 ISR 中的个副本的 LEO 不一致时，如果此时 leader 挂掉，选举新的 leader 时并不是按照 LEO 的高低进行选举，而是按照 ISR 中的顺序选举。


### 文章链接:https://www.infoq.cn/article/depth-interpretation-of-kafka-data-reliability/

##### 

### 高水位和Leader epoch 文章链接: http://zhongmingmao.me/2019/09/20/kafka-high-watermark-leader-epoch/

###### 高水位更新机制

#### 远程副本

<img src="https://kafka-1253868755.cos.ap-guangzhou.myqcloud.com/geek-time/kafka-high-watermark-update.png" alt="img" style="zoom:50%;" />

1. 每个副本对象都保存了一组高水位和LEO值，**Leader副本所在的Broker**还保存了***其它Follower副本的LEO值***
2. Kafka把Broker 0上保存的Follower副本又称为**远程副本**（**Remote** Replica）
3. Kafka副本机制在运行过程中
   - 会更新
     - Broker 1上Follower副本的高水位和LEO值
     - Broker 0上Leader副本的高水位和LEO以及**所有远程副本的LEO**
   - 不会更新
     - Broker 0**所有远程副本的高水位值**，即图中标记为**灰色**的部分
4. Broker 0保存远程副本的作用
   - **帮助**Leader副本**确定**其**高水位**，即**分区高水位**

#### 更新时机

| 更新对象                       | 更新时机                                                     |
| :----------------------------- | :----------------------------------------------------------- |
| Broker 0上Leader副本的LEO      | Leader副本**接收**到生产者发送的消息，**写入到本地磁盘**后，会更新其LEO值 |
| Broker 1上Follower副本的LEO    | Follower副本从Leader副本**拉取**消息，**写入本地磁盘**后，会更新其LEO值 |
| Broker 0上远程副本的LEO        | Follower副本从Leader副本**拉取**消息时，会告诉Leader副本**从哪个位移开始拉取**， Leader副本会使用这个位移值来更新远程副本的LEO |
| Broker 0上Leader副本的高水位   | 两个更新时机：一个是Leader副本更新其LEO之后，一个是更新完远程副本LEO之后 具体算法：取Leader副本和所有与Leader**同步**的远程副本LEO中的**最小值** |
| Broker 1上Follower副本的高水位 | Follower副本成功更新完LEO后，会比较其LEO与**Leader副本发来的高水位值**， 并用两者的**较小值**去更新自己的高水位 |

1. 与Leader副本保持同步，需要满足两个条件
   - 该远程Follower副本在**ISR**中
   - 该远程Follower副本LEO值**落后**Leader副本LEO值的时间**不超过**参数`replica.lag.time.max.ms`（**10秒**）
2. 某个副本能否进入ISR是由第二个条件判断的
   - 2个条件判断是为了应对意外情况：**Follower副本已经追上Leader，却不在ISR中**
   - 假设Kafka只判断第1个条件，副本F刚刚重启，并且已经具备进入ISR的资格，但此时尚未进入到ISR
     - 由于缺少了副本F的判断，**分区高水位有可能超过真正ISR中的副本LEO**，而**高水位>LEO**是**不允许**的

#### Leader副本

1. 处理生产者请求
   - 写入消息到本地磁盘，更新**LEO**
   - 更新分区高水位值
     - 获取Leader副本所在Broker端保存的所有远程副本LEO值`{LEO-1, LEO-2,... LEO-n}`
     - 获取Leader副本的LEO值：`currentLEO`
     - 更新**`currentHW = min(currentLEO, LEO-1, LEO-2,... LEO-n)`**
2. 处理Follower副本拉取消息
   - 读取**磁盘**（或**页缓存**）中的消息数据
   - 使用Follower副本发送请求中的位移值来更新远程副本的**LEO**值
   - 更新**分区高水位**值（与上面一致）

#### Follower副本

1. 从Leader拉取消息
   - 写入消息到本地磁盘
   - 更新**LEO**
   - 更新高水位值
     - 获取**Leader发送**的高水位值：`currentHW`
     - 获取步骤2中更新的LEO值：`currentLEO`
     - 更新高水位**`min(currentHW, currentLEO)`**

### 副本同步样例

主题是**单分区两副本**，首先是初始状态，所有值都是0

<img src="https://kafka-1253868755.cos.ap-guangzhou.myqcloud.com/geek-time/kafka-high-watermark-example-1.png" alt="img" style="zoom:80%;" />

当生产者向主题分区发送一条消息后，状态变更为

<img src="https://kafka-1253868755.cos.ap-guangzhou.myqcloud.com/geek-time/kafka-high-watermark-example-2.png" alt="img" style="zoom:67%;" />

此时，Leader副本成功将消息写入到**本地磁盘**，将**LEO**值更新为1（更新高水位值为0，并把结果发送给Follower副本）
Follower再次尝试从Leader拉取消息，此时有消息可以拉取，Follower副本也成功更新**LEO**为1（并将高水位更新为0）
此时，Leader副本和Follower副本的**LEO**都是1，但各自的**高水位依然是0**，需要等到**下一轮**的拉取中被更新

<img src="https://kafka-1253868755.cos.ap-guangzhou.myqcloud.com/geek-time/kafka-high-watermark-example-3.png" alt="img" style="zoom:67%;" />

**在新一轮的拉取请求中**(这个是什么时候呢?)，由于位移值为0的消息已经拉取成功，因此Follower副本这次拉取请求的位移值为**1**
Leader副本接收到此请求后，更新**远程副本LEO**为**1**，然后更新**Leader高水位**值为**1**
最后，**Leader副本**会将**当前更新过的高水位**值1发送给**Follower副本**，Follower副本接收到后，也会将自己的高水位值更新为**1**

<img src="https://kafka-1253868755.cos.ap-guangzhou.myqcloud.com/geek-time/kafka-high-watermark-example-4.png" alt="img" style="zoom: 67%;" />

## Leader Epoch (follower重启导致的截断)

### 基本概念

1. 上面的副本同步过程中，Follower副本的高水位更新需要一轮额外的拉取请求才能实现
   - 如果扩展到**多个Follower副本**，可能需要**多轮拉取请求**
   - 即Leader副本高水位更新和Follower副本高水位更新在时间上存在错配
     - 这种错配是很多**数据丢失**或**数据不一致**问题的根源
     - 因此，社区在**0.11**版本正式引入了`Leader Epoch`概念，来规避**高水位更新错配**导致的各种**不一致**问题
2. Leader Epoch可以大致认为是Leader版本，由两部分数据组成
   - Epoch
     - 一个**单调递增**的版本号
     - 每当**副本领导权发生变更**时，都会增加该版本号
     - 小版本号的Leader被认为是**过期Leader**，不能再行使Leader权利
   - 起始位移（Start Offset）
     - **Leader副本**在该Epoch值上写入的**首条消息**的位移
3. 两个Leader Epoch，<0,0>和<1,120>  `<0,0>`表示版本号为0，该版本的Leader从位移0开始保存消息，一共保存了120条消息
   - 之后Leader发生了**变更**，版本号增加到1，新版本的起始位移是120
4. Broker在内存中为每个分区都缓存Leader Epoch数据，同时还会定期地将这些数据持久化到一个checkpoint文件中
   - 当**Leader副本写入消息到磁盘**时，Broker会尝试更新这部分缓存
   - 如果Leader是**首次**写入消息，那么Broker会向缓存中**增加Leader Epoch条目**，否则不做更新
   - 这样每次有Leader变更时，新的Leader副本会查询这部分缓存，取出对应的Leader Epoch的起始位移
     - 然后进行相关的逻辑判断，避免**数据丢失**和**数据不一致**的情况

<img src="https://kafka-1253868755.cos.ap-guangzhou.myqcloud.com/geek-time/kafka-leader-epoch-example-1.png" alt="img" style="zoom: 33%;" />

1. 开始时，副本A和副本B都处于正常状态，A是Leader副本
2. 某个的生产者（**默认acks设置**）向A发送了两条消息，A全部写入成功，Kafka会通知生产者说两条消息全部发送成功
3. 假设Leader和Follower都写入了这两条消息，而且Leader副本的高水位也更新了，但***Follower副本的高水位还未更新***
4. 此时副本B所在的Broker宕机，当它重启回来后，副本B会执行**日志截断!!**
   - **将LEO值调整为之前的高水位值!!**，也就是1
   - 位移值为1的那条消息被副本B**从磁盘中删除**，此时副本B的**底层磁盘文件**中只保留1条消息，即位移为0的消息
5. 副本B执行完日志截断操作后，开始从A拉取消息，此时恰好副本A所在的Broker也宕机了，副本B自然成为新的Leader
   - 当A回来后，需要执行相同的**日志截断**操作，但**不能超过新Leader**，即**将高水位调整与B相同的值**，也就是1
   - 操作完成后，位移值为1的那条消息就从两个副本中被**永远抹掉**，造成了**数据丢失**

### Leader Epoch规避数据丢失

<img src="https://kafka-1253868755.cos.ap-guangzhou.myqcloud.com/geek-time/kafka-leader-epoch-example-2.png" alt="img" style="zoom: 33%;" />

1. Follower副本B重启后，需要向A发送一个特殊的请求去获取**Leader的LEO值**，该值为2

2. 当获知Leader LEO后，B发现该LEO值大于等于自己的LEO，而且缓存中也没有保存任何起始位移值>2的Epoch条目

   - **B无需执行任何日志截断操作**
   - 明显改进：***副本是否执行日志截断不再依赖于高水位进行判断***

3. A宕机，B成为Leader，当A重启回来后，执行与B相同的逻辑判断，发现同样

   不需要执行日志截断

   - 至此位移值为1的那条消息在两个副本中**均得到保留**
   - 后面生产者向B**写入新消息**后，副本B所在的Broker缓存中会生成新的Leader Epoch条目：**`[Epoch=1, Offset=2]`**

## 小结

1. 高水位在界定Kafka消息对外可见性以及实现副本机制方面起到非常重要的作用
   - 但设计上的缺陷给Kafka留下了很多**数据丢失**或**数据不一致**的潜在风险
2. 为此，社区引入了**`Leader Epoch`**机制，尝试规避这类风险，并且效果不错



# Broker Failover过程

## Controller对Broker failure的处理过程

1. Controller在Zookeeper的`/brokers/ids`节点上注册Watch。一旦有Broker宕机（本文用宕机代表任何让Kafka认为其Broker die的情景，包括但不限于机器断电，网络不可用，GC导致的Stop The World，进程crash等），其在Zookeeper对应的Znode会自动被删除，Zookeeper会fire Controller注册的Watch，Controller即可获取最新的幸存的Broker列表。

2. Controller决定set_p，该集合包含了宕机的所有Broker上的所有Partition

3. 对set_p中的每一个Partition：
   　　3.1 从`/brokers/topics/[topic]/partitions/[partition]/state`读取该Partition当前的ISR。
      　　3.2 决定该Partition的新Leader。如果当前ISR中有至少一个Replica还幸存，则选择其中一个作为新Leader，新的ISR则包含当前ISR中所有幸存的Replica。否则选择该Partition中任意一个幸存的Replica作为新的Leader以及ISR（该场景下可能会有潜在的数据丢失）。如果该Partition的所有Replica都宕机了，则将新的Leader设置为-1。

   ​		3.3 将新的Leader，ISR和新的`leader_epoch`及`controller_epoch`写入`/brokers/topics/[topic]/partitions/[partition]/state`。注意，该操作只有Controller版本在3.1至3.3的过程中无变化时才会执行，否则跳转到3.1。

4. 直接通过RPC向set_p相关的Broker发送LeaderAndISRRequest命令。Controller可以在一个RPC操作中发送多个命令从而提高效率

   <img src="http://www.jasongj.com/img/kafka/KafkaColumn2/kafka_broker_failover.png" alt="broker failover sequence diagram " style="zoom: 67%;" />

## LeaderAndIsrRequest响应过程

　　对于收到的LeaderAndIsrRequest，Broker主要通过ReplicaManager的becomeLeaderOrFollower处理，流程如下：

1. 若请求中controllerEpoch小于当前最新的controllerEpoch，则直接返回ErrorMapping.StaleControllerEpochCode。

2. 对于请求中partitionStateInfos中的每一个元素，即（(topic, partitionId), partitionStateInfo)：
   　　2.1 若partitionStateInfo中的leader epoch大于当前ReplicManager中存储的(topic, partitionId)对应的partition的leader epoch，则：
      　　　　2.1.1 若当前brokerid（或者说replica id）在partitionStateInfo中，则将该partition及partitionStateInfo存入一个名为partitionState的HashMap中
      　　　　2.1.2否则说明该Broker不在该Partition分配的Replica list中，将该信息记录于log中
      　　2.2否则将相应的Error code（ErrorMapping.StaleLeaderEpochCode）存入Response中

3. 筛选出partitionState中Leader与当前Broker ID相等的所有记录存入partitionsTobeLeader中，其它记录存入partitionsToBeFollower中。

4. 若partitionsTobeLeader不为空，则对其执行makeLeaders方。

5. 若partitionsToBeFollower不为空，则对其执行makeFollowers方法。

6. 若highwatermak线程还未启动，则将其启动，并将hwThreadInitialized设为true。

7. 关闭所有Idle状态的Fetcher。

   <img src="http://www.jasongj.com/img/kafka/KafkaColumn3/LeaderAndIsrRequest_Flow_Chart.png" alt="LeaderAndIsrRequest Flow Chart" style="zoom: 67%;" />

## 创建/删除Topic

1. Controller在Zookeeper的`/brokers/topics`节点上注册Watch，一旦某个Topic被创建或删除，则Controller会通过Watch得到新创建/删除的Topic的Partition/Replica分配。
2. 对于删除Topic操作，Topic工具会将该Topic名字存于`/admin/delete_topics`。若`delete.topic.enable`为true，则Controller注册在`/admin/delete_topics`上的Watch被fire，Controller通过回调向对应的Broker发送StopReplicaRequest；若为false则Controller不会在`/admin/delete_topics`上注册Watch，也就不会对该事件作出反应，此时Topic操作只被记录而不会被执行。
3. 对于创建Topic操作，Controller从`/brokers/ids`读取当前所有可用的Broker列表，对于set_p中的每一个Partition：
   　　3.1 从分配给该Partition的所有Replica（称为AR）中任选一个可用的Broker作为新的Leader，并将AR设置为新的ISR（因为该Topic是新创建的，所以AR中所有的Replica都没有数据，可认为它们都是同步的，也即都在ISR中，任意一个Replica都可作为Leader）
      　　3.2 将新的Leader和ISR写入`/brokers/topics/[topic]/partitions/[partition]`
4. 直接通过RPC向相关的Broker发送LeaderAndISRRequest。

<img src="http://www.jasongj.com/img/kafka/KafkaColumn2/kafka_create_topic.png" alt="create topic sequence diagram" style="zoom:67%;" />

## Controller Failover

Controller也需要Failover。每个Broker都会在Controller Path (`/controller`)上注册一个Watch。当前Controller失败时，对应的Controller Path会自动消失（因为它是Ephemeral Node），此时该Watch被fire，所有“活”着的Broker都会去竞选成为新的Controller（创建新的Controller Path），但是只会有一个竞选成功（这点由Zookeeper保证）。竞选成功者即为新的Leader，竞选失败者则重新在新的Controller Path上注册Watch。因为[Zookeeper的Watch是一次性的，被fire一次之后即失效](http://zookeeper.apache.org/doc/trunk/zookeeperProgrammers.html#ch_zkWatches)，所以需要重新注册。

Broker成功竞选为新Controller后会触发KafkaController.onControllerFailover方法，并在该方法中完成如下操作：

1. 读取并增加Controller Epoch。
2. 在ReassignedPartitions Path(`/admin/reassign_partitions`)上注册Watch。
3. 在PreferredReplicaElection Path(`/admin/preferred_replica_election`)上注册Watch。
4. 通过partitionStateMachine在Broker Topics Patch(`/brokers/topics`)上注册Watch。
5. 若`delete.topic.enable`设置为true（默认值是false），则partitionStateMachine在Delete Topic Patch(`/admin/delete_topics`)上注册Watch。
6. 通过replicaStateMachine在Broker Ids Patch(`/brokers/ids`)上注册Watch。
7. 初始化ControllerContext对象，设置当前所有Topic，“活”着的Broker列表，所有Partition的Leader及ISR等。
8. 启动replicaStateMachine和partitionStateMachine。
9. 将brokerState状态设置为RunningAsController。
10. 将每个Partition的Leadership信息发送给所有“活”着的Broker。
11. 若`auto.leader.rebalance.enable`配置为true（默认值是true），则启动partition-rebalance线程。
12. 若`delete.topic.enable`设置为true且Delete Topic Patch(`/admin/delete_topics`)中有值，则删除相应的Topic



# High Level Consumer

很多时候，客户程序只是希望从Kafka读取数据，不太关心消息offset的处理。同时也希望提供一些语义，例如同一条消息只被某一个Consumer消费（单播）或被所有Consumer消费（广播）。因此，Kafka Hight Level Consumer提供了一个从Kafka消费数据的高层抽象，从而屏蔽掉其中的细节并提供丰富的语义。

## Consumer Group

High Level Consumer将从某个Partition读取的最后一条消息的offset存于Zookeeper中([Kafka从0.8.2版本](https://archive.apache.org/dist/kafka/0.8.2.0/RELEASE_NOTES.html)开始同时支持将offset存于Zookeeper中与[将offset存于专用的Kafka Topic中](https://issues.apache.org/jira/browse/KAFKA-1012))。这个offset基于客户程序提供给Kafka的名字来保存，这个名字被称为Consumer Group。Consumer Group是整个Kafka集群全局的，而非某个Topic的。每一个High Level Consumer实例都属于一个Consumer Group，若不指定则属于默认的Group。

<img src="http://www.jasongj.com/img/kafka/KafkaColumn4/KafkaColumn4-consumers.png" alt="Consumer Zookeeper Structure" style="zoom:80%;" />

很多传统的Message Queue都会在消息被消费完后将消息删除，一方面避免重复消费，另一方面可以保证Queue的长度比较短，提高效率。而如上文所述，Kafka并不删除已消费的消息，为了实现传统Message Queue消息只被消费一次的语义，Kafka保证每条消息在同一个Consumer Group里只会被某一个Consumer消费。与传统Message Queue不同的是，Kafka还允许不同Consumer Group同时消费同一条消息，这一特性可以为消息的多元化处理提供支持。

<img src="http://www.jasongj.com/img/kafka/KafkaAnalysis/consumer_group.png" alt="kafka consumer group" style="zoom: 67%;" />





每个消费者唯一保存的元数据是offset值，这个位置完全为消费者控制，因此消费者可以采用任何顺序来消费记录，如下图3.3

![img](https://pic1.zhimg.com/80/v2-d16bfd7504e51a93592e6dd9b7c26a80_720w.jpg)

**kafka中只能保证partition中记录是有序的，而不保证topic中不同partition的顺序**

一个消费组由一个或多个消费者实例组成，便于扩容与容错。kafka是发布与订阅模式，这个订阅者是消费组，而不是消费者实例。每一条消息只会被同一个消费组里的一个消费者实例消费，不同的消费组可以同时消费同一条消息，

![img](https://pic2.zhimg.com/80/v2-9d834d37fe52e7aa9fecf368a42540ad_720w.jpg)

为了实现传统的消息队列中消息只被消费一次的语义，kafka保证同一个消费组里只有一个消费者会消费一条消息，kafka还允许不同的消费组同时消费一条消息，这一特性可以为消息的多元化处理提供了支持，kafka的设计理念之一就是同时提供离线处理和实时处理，因此，可以使用Storm这种实时流处理系统对消息进行实时在线处理，同时使用Hadoop这种批处理系统进行离线处理，还可以同时将数据实时备份到另一个数据中心，只需要保证这三个操作的消费者实例在不同consumer group 即可



## Data Replication

　　Kafka的Data Replication需要解决如下问题：

- 怎样Propagate消息
- 在向Producer发送ACK前需要保证有多少个Replica已经收到该消息
- 怎样处理某个Replica不工作的情况
- 怎样处理Failed Replica恢复回来的情况

### Propagate消息

Producer在发布消息到某个Partition时，先通过 Metadata （通过 Broker 获取并且缓存在 Producer 内） 找到该 Partition 的Leader，然后无论该Topic的Replication Factor为多少（也即该Partition有多少个Replica），Producer只将该消息发送到该Partition的Leader。Leader会将该消息写入其本地Log。每个Follower都从Leader pull数据。这种方式上，Follower存储的数据顺序与Leader保持一致。Follower在收到该消息并写入其Log后，向Leader发送ACK。一旦Leader收到了ISR中的所有Replica的ACK，该消息就被认为已经commit了，Leader将增加HW并且向Producer发送ACK。

为了提高性能，每个Follower在接收到数据后就立马向Leader发送ACK，而非等到数据写入Log中。因此，对于已经commit的消息，Kafka只能保证它被存于多个Replica的内存中，而不能保证它们被持久化到磁盘中，也就不能完全保证异常发生后该条消息一定能被Consumer消费。但考虑到这种场景非常少见，可以认为这种方式在性能和数据持久化上做了一个比较好的平衡。在将来的版本中，Kafka会考虑提供更高的持久性。
Consumer读消息也是从Leader读取，只有被commit过的消息（offset低于HW的消息）才会暴露给Consumer。

### ACK前需要保证有多少个备份

和大部分分布式系统一样，Kafka处理失败需要明确定义一个Broker是否“活着”。对于Kafka而言，Kafka存活包含两个条件，一是它必须维护与Zookeeper的session(这个通过Zookeeper的Heartbeat机制来实现)。二是Follower必须能够及时将Leader的消息复制过来，不能“落后太多”。

Leader会跟踪与其保持同步的Replica列表，该列表称为ISR（即in-sync Replica）。如果一个Follower宕机，或者落后太多，Leader将把它从ISR中移除。这里所描述的“落后太多”指Follower复制的消息落后于Leader后的条数超过预定值（该值可在$KAFKA_HOME/config/server.properties中通过`replica.lag.max.messages`配置，其默认值是4000）或者Follower超过一定时间（该值可在$KAFKA_HOME/config/server.properties中通过`replica.lag.time.max.ms`来配置，其默认值是10000）未向Leader发送fetch请求。。
　　Kafka的复制机制既不是完全的同步复制，也不是单纯的异步复制。事实上，同步复制要求所有能工作的Follower都复制完，这条消息才会被认为commit，这种复制方式极大的影响了吞吐率（高吞吐率是Kafka非常重要的一个特性）。而异步复制方式下，Follower异步的从Leader复制数据，数据只要被Leader写入log就被认为已经commit，这种情况下如果Follower都复制完都落后于Leader，而如果Leader突然宕机，则会丢失数据。而Kafka的这种使用ISR的方式则很好的均衡了确保数据不丢失以及吞吐率。Follower可以批量的从Leader复制数据，这样极大的提高复制性能（批量写磁盘），极大减少了Follower与Leader的差距。
　　需要说明的是，Kafka只解决fail/recover，不处理“Byzantine”（“拜占庭”）问题。一条消息只有被ISR里的所有Follower都从Leader复制过去才会被认为已提交。这样就避免了部分数据被写进了Leader，还没来得及被任何Follower复制就宕机了，而造成数据丢失（Consumer无法消费这些数据）。而对于Producer而言，它可以选择是否等待消息commit，这可以通过`request.required.acks`来设置。这种机制确保了只要ISR有一个或以上的Follower，一条被commit的消息就不会丢失。 

### Leader 选举

上文说明了Kafka是如何做Replication的，另外一个很重要的问题是当Leader宕机了，怎样在Follower中选举出新的Leader。因为Follower可能落后许多或者crash了，所以必须确保选择“最新”的Follower作为新的Leader。一个基本的原则就是，如果Leader不在了，新的Leader必须拥有原来的Leader commit过的所有消息。这就需要作一个折衷，如果Leader在标明一条消息被commit前等待更多的Follower确认，那在它宕机之后就有更多的Follower可以作为新的Leader，但这也会造成吞吐率的下降.

Kafka在Zookeeper中动态维护了一个ISR（in-sync replicas），这个ISR里的所有Replica都跟上了leader，只有ISR里的成员才有被选为Leader的可能。在这种模式下，对于f+1个Replica，一个Partition能在保证不丢失已经commit的消息的前提下容忍f个Replica的失败。在大多数使用场景中，这种模式是非常有利的。事实上，为了容忍f个Replica的失败，Majority Vote和ISR在commit前需要等待的Replica数量是一样的，但是ISR需要的总的Replica的个数几乎是Majority Vote的一半。(这里有问题,  因为ISR的失败是10秒不同步, 所以可能会有未完全跟上的副本)

### 如何处理所有Replica都不工作

上文提到，在ISR中至少有一个follower时，Kafka可以确保已经commit的数据不丢失，但如果某个Partition的所有Replica都宕机了，就无法保证数据不丢失了。这种情况下有两种可行的方案：

- 等待ISR中的任一个Replica“活”过来，并且选它作为Leader
- 选择第一个“活”过来的Replica（不一定是ISR中的）作为Leader

　　这就需要在可用性和一致性当中作出一个简单的折衷。如果一定要等待ISR中的Replica“活”过来，那不可用的时间就可能会相对较长。而且如果ISR中的所有Replica都无法“活”过来了，或者数据都丢失了，这个Partition将永远不可用。选择第一个“活”过来的Replica作为Leader，而这个Replica不是ISR中的Replica，那即使它并不保证已经包含了所有已commit的消息，它也会成为Leader而作为consumer的数据源（前文有说明，所有读写都由Leader完成）。Kafka0.8.*使用了第二种方式。根据Kafka的文档，在以后的版本中，Kafka支持用户通过配置选择这两种方式中的一种，从而根据不同的使用场景选择高可用性还是强一致性。 　

### 如何选举Leader

它在所有broker中选出一个controller，所有Partition的Leader选举都由controller决定。controller会将Leader的改变直接通过RPC的方式（比Zookeeper Queue的方式更高效）通知需为此作出响应的Broker。同时controller也负责增删Topic以及Replica的重新分配。



### 文章链接: http://www.jasongj.com/2015/04/24/KafkaColumn2/



---

## RocketMQ

1. 消息的可靠性(暂时不考虑)

   1. ack机制
   2. 重发

2. 镜像(replication) 集群 (横向扩容, 不考虑)

   1. RocketMQ通过主从结构来实现消息冗余，master接收来自producer发送来的消息，然后同步消息到slave，根据master的role不同，同步的时机可分为两种不同的情况：

      - SYNC_MASTER：如果master是这种角色，每次master在将producer发送来的消息写入内存（磁盘）的时候会同步等待master将消息传输到slave
      - ASYNC_MASTER：这种角色下消息会异步复制到slave

      这里注意的是master传输到slave只有CommitLog的物理文件。

   2. 消息的存储 顺序写磁盘(CommitLog)

3. producer只能发送消息到master，而不能发送到slave，这也说明了**master负责读“写”，而slave只负责读**

4. 消息的存储是一直存在于CommitLog中的。而由于CommitLog是以文件为单位（而非消息）存在的，CommitLog的设计是只允许顺序写的，且每个消息大小不定长，所以这决定了消息文件几乎不可能按照消息为单位删除（否则性能会极具下降，逻辑也非常复杂）。所以消息被消费了，消息所占据的物理空间并不会立刻被回收。

5. broker leader 宕机.  Dledger方案

   1. 消息会不会丢失  raft协议保证了写入成功的消息存在多数节点上，最终选举出来的主节点的当前复制进度一定是比绝大多数的从节点要大，并且也会等于承偌给客户端的已提交偏移量，故不会丢消息
   2. 消息会重复消费  消息消费进度的同步是slave定时向master拉取进行更新，存在时延，master挂掉后，消费者继续消费，此时消费的broker不一定就能选举成为master，因此消费进度可能丢失，存在重复消费的可能性
   3. 消息的顺序性  (partion保证)

6. 宕机的机器重启后(或者新加入的机器), 重新连接leader. 同步数据  

   1. 

7. 消费者消费数据(暂时不考虑)

   1. 数据的消费特性(保证 at-least-once delivery)



## Topic的存储

### Topic

标识一类消息的逻辑名字，消息的逻辑管理单位。无论消息生产还是消费，都需要指定Topic。

### Tag

RocketMQ支持给在发送的时候给topic打tag，同一个topic的消息虽然逻辑管理是一样的。但是消费topic1的时候，如果你订阅的时候指定的是tagA，那么tagB的消息将不会投递。

### Message Queue

简称Queue或Q。消息物理管理单位。一个Topic将有若干个Q。若Topic同时创建在不同的Broker，则不同的broker上都有若干Q，消息将物理地存储落在不同Broker结点上，具有水平扩展的能力。

无论生产者还是消费者，实际的生产和消费都是针对Q级别。例如Producer发送消息的时候，会预先选择（默认轮询）好该Topic下面的某一条Q地发送；Consumer消费的时候也会负载均衡地分配若干个Q，只拉取对应Q的消息。

每一条message queue均对应一个文件，这个文件存储了实际消息的索引信息。

Message queue存储消息的偏移量。读消息先读message queue，根据偏移量到commit log读消息本身

并且即使文件被删除，也能通过实际纯粹的消息文件（commit log）恢复回来。



![img](https://upload-images.jianshu.io/upload_images/6302559-5693e4bec15216b5.png?imageMogr2/auto-orient/strip|imageView2/2/w/837/format/webp)





![img](https://images2018.cnblogs.com/blog/846961/201805/846961-20180505144614692-1825592300.png)

在RocketMQ里面有一个概念broker set，一个broker set由一个master和多个slave组成，一个broker set内的每个broker的brokerName相同。

在broker集群中每个master相互之间是独立，master之间不会有交互，每个master维护自己的CommitLog、自己的ConsumeQueue，但是每一个master都有可能收到同一个topic下的producer发来的消息



# 1. 概述

本文主要解析 `Namesrv`、`Broker` 如何实现高可用，`Producer`、`Consumer` 怎么与它们通信保证高可用。

# 2. Namesrv 高可用

**启动多个 `Namesrv` 实现高可用。**
相较于 `ZooKeeper`、`Consul`、`Etcd` 等，`Namesrv` 是一个**超轻量级**的注册中心，提供**命名服务**。

## 2.1 broker启动的时候会向namesrv注册自己的信息

**多个 `Namesrv` 之间，没有任何关系（不存在类似 `ZooKeeper` 的 `Leader`/`Follower` 等角色），不进行通信与数据同步。通过 `Broker` 循环注册多个 `Namesrv`。**

```java
【BrokerOuterAPI.java】
List<String> nameServerAddressList = this.remotingClient.getNameServerAddressList();
if (nameServerAddressList != null) {
	for (String namesrvAddr : nameServerAddressList) { // 循环多个 Namesrv
	 	RegisterBrokerResult result = this.registerBroker(namesrvAddr, clusterName, brokerAddr, brokerName, brokerId,
                     haServerAddr, topicConfigWrapper, filterServerList, oneway, timeoutMills);
        if (result != null) {
            registerBrokerResult = result;
        }
	}
return registerBrokerResult;
```

所以，namesrv通过将broker注册来的信息构造成自己的数据结构：

- 每个cluster有哪些broker set
- 每个broker set包括哪些broker，brokerId和broker的ip:port
- 每个broker的存活情况，根据每次broker上报来的信息，清除可能下线的broker
- 每个topic的消息队列信息，几个读队列，几个写队列

namesrv汇总所有的broker的这些信息，然后供consumer和producer拉取

## 2.2 Producer、Consumer 访问 Namesrv

**`Producer`、`Consumer` 从 `Namesrv`列表选择一个可连接的进行通信。**

```java
【NettyRemotingClient.java】
// 返回已选择、可连接Namesrv
String addr = this.namesrvAddrChoosed.get();
if (addr != null) {
	 ChannelWrapper cw = this.channelTables.get(addr);
	 if (cw != null && cw.isOK()) {
	 	return cw.getChannel();
	  }
 } 
... 省略没有查找到的情况
```

### producer发送消息的时候知道发送到哪一个master

producer发送消息的时候发往哪一个broker是由MessageQueue决定的，所以我们先要搞清楚producer发送消息时候的MessageQueue怎么来的。producer维护了一个topicPublishInfoTable，里面包含了每个topic对应的MessageQueue，所以问题就变成了topicPublishInfoTable怎么构造的。

producer发送消息之前都会获取topic对应的队列信息，当topicPublishInfoTable中没有的时候会从namesrv获取

```java
TopicRouteData topicRouteData;
// 从manesrv获取topic的路由信息，namesrv从topicQueueTable获取到该topic对应的所有的QueueData
// 然后将每个brokerName下的BrokerData返回
topicRouteData = this.mQClientAPIImpl.getTopicRouteInfoFromNameServer(topic, 1000 * 3);
for (BrokerData bd : topicRouteData.getBrokerDatas()) {
    // 每个broker set下所有的broker地址(ip:port)
    this.brokerAddrTable.put(bd.getBrokerName(), bd.getBrokerAddrs());
}
// 将从namesrv获取到的路由信息转换为TopicPublishInfo
// 期间会将没有master的broker set的queue信息去除
TopicPublishInfo publishInfo = topicRouteData2TopicPublishInfo(topic, topicRouteData);
```

到此，producer也知道自己可以向哪些MessageQueue发送消息了，接下来就是producer的负载均衡算法选出其中一个MessageQueue发送消息（org.apache.rocketmq.client.impl.producer.DefaultMQProducerImpl#selectOneMessageQueue，这个暂时不详表），MessageQueue包含的信息有topic、brokerName、queueId，但是producer发送的时候得知道broker的ip:port信息，而且一个brokerName对应的是一个broker set，并不能确定具体的broker，所以接下来应该找到具体的broker

```java
// org.apache.rocketmq.client.impl.factory.MQClientInstance#findBrokerAddressInPublish
public String findBrokerAddressInPublish(final String brokerName) {
    // 上面updateTopicRouteInfoFromNameServer方法将broker set下的broker地址信息保存到brokerAddrTable
    // 再次重申：一个broker set下的broker的brokerName相同
    HashMap<Long/* brokerId */, String/* address */> map = this.brokerAddrTable.get(brokerName);
    if (map != null && !map.isEmpty()) {
        // 没有花样，就是直接返回brokerId时MixAll.MASTER_ID的broker的ip:port信息
        // 前面说过master的brokerId就是MixAll.MASTER_ID，所以获取到的broker是broker set中的master
        return map.get(MixAll.MASTER_ID);
    }

    return null;
}
```

producer只会向是master的broker发送消息，也就是一个broker set中brokerId是0的broker。

producer只能发送消息到master，而不能发送到slave，这也说明了master负责读“写”，而slave只负责读



# 3. Broker 高可用

**启动多个 `Broker分组` 形成 `集群` 实现高可用。**
**`Broker分组` = `Master节点`x1 + `Slave节点`xN。**
类似 `MySQL`，`Master节点` 提供**读写**服务，`Slave节点` 只提供**读**服务。

## 3.2 Broker 主从

- **每个分组，`Master`节点 不断发送新的 `CommitLog` 给 `Slave`节点。 `Slave`节点 不断上报本地的 `CommitLog` 已经同步到的位置给 `Master`节点。**

- **`Broker分组` 与 `Broker分组` 之间没有任何关系，不进行通信与数据同步。**

集群内，`Master`节点 有**两种**类型：`Master_SYNC`、`Master_ASYNC`：前者在 `Producer` 发送消息时，等待 `Slave`节点 存储完毕后再返回发送结果，而后者不需要等待。

### 3.1.2 组件

再看具体实现代码之前，我们来看看 `Master`/`Slave`节点 包含的组件：

![HAç»ä»¶å¾.png](http://static.iocoder.cn/images/RocketMQ/2017_05_14/04.png)

- ```
  Master
  ```

  节点

  - `AcceptSocketService` ：接收 `Slave`节点 连接。

  - ```
    HAConnection
    ```

    - `ReadSocketService` ：**读**来自 `Slave`节点 的数据。
    - `WriteSocketService` ：**写**到往 `Slave`节点 的数据。

- ```
  Slave
  ```

  节点

  - `HAClient` ：对 `Master`节点 连接、读写数据。

### 3.1.3 通信协议

`Master`节点 与 `Slave`节点 **通信协议**很简单，只有如下两条。

| 对象          | 用途                                      | 第几位 | 字段          | 数据类型 | 字节数 | 说明                  |
| :------------ | :---------------------------------------- | :----- | :------------ | :------- | :----- | :-------------------- |
| Slave=>Master | 上报CommitLog**已经**同步到的**物理**位置 |        |               |          |        |                       |
|               |                                           | 0      | maxPhyOffset  | Long     | 8      | CommitLog最大物理位置 |
| Master=>Slave | 传输新的 `CommitLog` 数据                 |        |               |          |        |                       |
|               |                                           | 0      | fromPhyOffset | Long     | 8      | CommitLog开始物理位置 |
|               |                                           | 1      | size          | Int      | 4      | 传输CommitLog数据长度 |
|               |                                           | 2      | body          | Bytes    | size   | 传输CommitLog数据     |

### ASYNC_MASTER同步数据到slave

1. salve连接到master，向master上报slave当前的offset
2. master收到后确认给slave发送数据的开始位置
3. master查询开始位置对应的MappedFIle
4. master将查找到的数据发送给slave
5. slave收到数据后保存到自己的CommitLog

```java
// org.apache.rocketmq.store.ha.HAService.HAClient#run

// slave上报过来的offset说明offset之前的数据slave都已经收到
HAConnection.this.slaveAckOffset = readOffset;
if (HAConnection.this.slaveRequestOffset < 0) {
    // 如果是刚刚和slave建立连接，需要知道slave需要从哪里开始接收commitLog
    HAConnection.this.slaveRequestOffset = readOffset;
    log.info("slave[" + HAConnection.this.clientAddr + "] request offset " + readOffset);
}
// 如果收到来自slave的确认之后，唤醒等待同步到slave的线程(如果是SYNC_MASTER)
HAConnection.this.haService.notifyTransferSome(HAConnection.this.slaveAckOffset);
```

通过上面slave和master的通信，master已经知道第一次从哪里（slaveRequestOffset）开始给slave传输数据

```java
// 说明还没收到来自slave的offset，等10ms重试 
if (-1 == HAConnection.this.slaveRequestOffset) {
    Thread.sleep(10);
    continue;
}
// 如果是第一次发送数据需要计算出从哪里开始给slave发送数据
if (0 == HAConnection.this.slaveRequestOffset) {
     this.nextTransferFromWhere = masterOffset;
}else{
     this.nextTransferFromWhere = HAConnection.this.slaveRequestOffset;
}

// 如果上一次transfer完成了才进行下一次transfer
if (this.lastWriteOver) {
    ...
    this.lastWriteOver = this.transferData();
}else{
    // 说明上一次的数据还没有传输完成，这里继续上一次的传输
}
```

上面master已经通过网络将MappedFile数据发送给slave，接下来就是slave收到master的数据，然后保存到自己的CommitLog。(省略)

### SYNC_MASTER同步数据到slave

SYNC_MASTER和ASYNC_MASTER传输数据到salve的过程是一致的，只是时机上不一样。SYNC_MASTER接收到producer发送来的消息时候，会同步等待消息也传输到salve。

1. master将需要传输到slave的数据构造为GroupCommitRequest交给GroupTransferService
2. 唤醒传输数据的线程（如果没有更多数据需要传输的的时候HAClient.run会等待新的消息）
3. 等待当前的传输请求完成



## RocketMQ高可用方案现状

### 3.1 Master/Slave方案

![img](https://pic2.zhimg.com/80/v2-89677183690ab97b9b3a7c8d3fb1aa7d_720w.jpg)

**核心概念：**

- Name Server：无状态集群(数据可能暂时不一致)，提供命名服务，更新和发现Topic-Broker服务。
- Broker：负责存储、转发消息，分为Master和Slave。Broker启动后需要将自己的Topic和Group等路由信息注册至所有Name Server；随后每隔30s定期向Name Server上报路由信息。
- 生产者：与 Name Server 任一节点建立长链接，定期从 Name Server 读取路由信息，并与提供 Topic 服务的对应 Master Broker 建立长链接。
- 消费者：与 Name Server 任一节点建立长链接，定期从 Name Server 拉取路由信息，并与提供 Topic 服务的 Master Broker、Slave Broker 建立长连接。Consumer 既可以从 Master Broker 订阅消息，也可以从 Slave Broker 订阅消息，订阅规则由 Broker 配置决定。支持Pull和Push两种模型。
- Topic：表示一类消息的集合，每个主题包含若干条消息，每条消息只能属于一个主题，是RocketMQ进行消息订阅的基本单位。

### Master/Slave方案

部署架构图可参考本文第二章概念介绍图，核心是broker部署的时候指定角色，通过主从复制实现，支持同步和异步两种模式。

![img](https://pic3.zhimg.com/80/v2-dff8ed5c912db053a51933e09b7c3dd6_720w.jpg)

- Broker目前仅支持主备复制，Master宕机不可修复需要手动修改备的配置文件(此时Slave是Read-Only)然后重启，不具备宕机自动failover能力
- Master/Slave复制模式分为同步和异步，用户可以根据业务场景进行trade-off，在性能和数据可靠性之间选择一个
- RocketMQ的Topic可以分片在不同的broker上面，一个broker挂了，生产者可以继续投递消息到其他broker，已经挂的broker上的消息必须等人工重启才能进一步处理

### 3.2 Dledger方案

![img](https://pic2.zhimg.com/80/v2-0ec71d9e5eece3caccd523d6fcc53b7d_720w.jpg)

核心是实现了基于raft协议的DLedgerCommitLog，保证Broker内commitLog文件的一致性。

### RocketMQ-on-DLedger-Group

- RocketMQ-on-DLedger-Group 是指一组相同名称的 Broker，至少需要 3 个节点，通过 Raft 自动选举出一个 Leader，其余节点 作为 Follower，并在 Leader 和 Follower 之间复制数据以保证高可用
- RocketMQ-on-DLedger-Group能自动容灾切换，并保证数据一致
- RocketMQ-on-DLedger-Group是可以水平扩展的，也即可以部署任意多个RocketMQ-on-DLedger-Group同时对外提供服务

### 数据可靠性

DLedger机制只保证commitlog文件的一致性，主从切换过程中是否会有数据丢失和消费进度错乱呢？

- 消息消费进度的同步是slave定时向master拉取进行更新，存在时延，master挂掉后，消费者继续消费，此时消费的broker不一定就能选举成为master，因此消费进度可能丢失，存在重复消费的可能性
- raft协议保证了写入成功的消息存在多数节点上，最终选举出来的主节点的当前复制进度一定是比绝大多数的从节点要大，并且也会等于承偌给客户端的已提交偏移量，故不会丢消息



### https://zhuanlan.zhihu.com/p/108256762

### 参考链接: https://www.cnblogs.com/sunshine-2015/p/8998549.html

### https://blog.csdn.net/notOnlyRush/article/details/80358426



网易云音乐

**消息队列高可用概要设计：**



![img](https://pic3.zhimg.com/80/v2-4bceddc80b4f7e39a40b13611e58fcaa_720w.jpg)

针对Level0，Level1的QOS，我们提供异步复制异步双写模式优先保证业务可用性，在机器宕机时能快速恢复。针对Level2的QOS要求，我们提供了同步复制同步双写，优先保证消息不丢失。

云音乐基于开源的Rplication，修正Replication复制的问题，增加CommitlogOffset上报，Failover自动探测选主的策略实现了根据QOS可配置的Broker高可用。

![img](https://pic4.zhimg.com/80/v2-4961d5778ab590d33a4b22890be63007_720w.jpg)

据图分析和理解，该方案和阿里HAController有限状态机变化类似，区别在于增加了服务QOS，根据设定的QOS达到指定的终态。

1） 第一个节点启动后，Controller控制状态机切换为单主状态，通知启动节点以AsyncMaster角色提供服务。

2） Controller发现启动新的Slave后，将全局状态改为异步复制状态，主节点AsyncMaster，从节点Slave模式运行，判断QOS如果是Level0和Level1则进入终态

3） 如果QOS是Lever2, Controller根据Slave同步Master的进度，如主备之间offset相差50条消息，全局状态进入同步复制状态，此时主节点按照SyncMaster模式运行，从节点依然是Slave。

4）QOS是level0或level1，AsyncMaster挂掉的情况下，Controller根据之前主备上报的CommitlogOffset提升Slave为AsyncMaster，此时可能会丢失部分消息(取决于异步复制进度)

5）QOS是level2，SyncMaster宕机，因为是同步复制，所以可以直接提升Slave为AsyncMaster单节点运行，等待新的节点加入

6）Slave宕机，告警即可，无需切换；单节点运行状态下如果宕机也是直接告警。



基础知识

在讲一些思想之前，我们需要先了解一些概念。

一个 messaging system 最重要的事情莫过于消息送达的模式：at least once 或者 at most once。at least once 是指同一个消息会被传输 1 到 n 次，而 at most once 是指同一个消息会被传输 0 到 1 次。这很好理解，如果 messaging system 内建了重传机制，并且将消息持久化到磁盘中以保证即便进程崩溃消息依旧能够送达，那么这就是 at least once。反之，如果没有构建任何上述的机制，消息送出后就并不理会，这是 at most once。很可惜，ZeroMQ 并非严格意义上的 at least once 或者 at most once，以其 Pub/Sub 模式来说，ZeroMQ 构建了消息确认和重传机制，却未对消息进行持久化，那么内存耗尽或者进程崩溃都会造成消息丢失，而重传则可能会造成消息被发送 1 到 n 次。这也是为何我认为 ZeroMQ 并非真正意义上的 Message Queue，当然，它可以用来构建一个真正的 MQ。注意，在一个网络环境中，消息的送达只能是上述两种情况，不可能 exactly once，如果有人这么说，那么一定是在误导。

讲到消息重传，细心的同学可能会疑虑：TCP 内建有重传机制，为何 ZeroMQ 在消息层面还要多此一举？这是因为 TCP 的重传只保证了网络层面报文的重传，而 ZeroMQ 通过消息层面的重传，保证了一个消息一旦送达，一定是完整送达。

at least once 的使用场景很容易理解，我们发送一条消息，自然是为了接受者能够保证接收到。至于保证接收的副作用 —— 重传的副本，只要消息的处理是幂等（Idempotent）的，就不会有问题。at most once 的使用场景让人比较困惑，什么时候我们发了一条消息，丢了也就丢了，并不可惜呢？比如说这些场景：

- 各种网络拓扑下的 heart beat（当然，大部分场合下 heart beat 可以直接用 IP/UDP，不必使用 messaging），偶尔丢几个消息无关痛痒
- 密集的 status report message 或者 tracking event。丢失的消息对全局并不构成威胁。

最后一个概念是 back pressure。但凡一个 messaging system 里，消息的两端，生产者和消费者间，都会产生处理速度不一致的问题。如果生产者发送消息的速度过快，消费者赶不及处理，就会造成消息的拥塞，进而不断把压力回溯给上游，最终一层层回溯到消息的生产者，使其停止产生更多的内容。



# Pulsar



## **Pulsar 的分层架构**

从数据库到消息系统，大多数分布式系统采用了数据处理和数据存储共存于同一节点的方法。这种设计减少了网络上的数据传输，可以提供更简单的基础架构和性能优势，但其在系统可扩展性和高可用性上会大打折扣。

Pulsar 架构中数据服务和数据存储是单独的两层：数据服务层由无状态的 “Broker” 节点组成，而数据存储层则由 “Bookie” 节点组成。

![img](https://pic3.zhimg.com/80/v2-d1620cf298a5d5d5626f2273ac6d9c26_720w.jpg)

这种存储和计算分离的架构给 Pulsar 带来了很多优势。首先，在 Pulsar 这种分层架构中，服务层和存储层都能够独立扩展，可以提供灵活的弹性扩容。特别是在弹性环境（例如云和容器）中能够自动扩容缩容，并动态适应流量的峰值。并且， Pulsar 这种分层架构显著降低了集群扩展和升级的复杂性，提高了系统可用性和可管理性。此外，这种设计对容器是非常友好的，这使 得Pulsar 也成为了流原生平台的理想选择。

Pulsar 系统架构的优势也包括 Pulsar 分片存储数据的方式。Pulsar 将主题分区按照更小的分片粒度来存储，然后将这些分片均匀打散分布在存储层的 “bookie” 节点上。这种以分片为中心的数据存储方式，将主题分区作为一个逻辑概念，分为多个较小的分片，并均匀分布和存储在存储层中。这种架构设计为 Pulsar 带来了更好的性能，更灵活的扩展性和更高的可用性。

Pulsar 架构中的每层都可以单独设置大小，进行扩展和配置。根据其在不同服务中的作用不同，可灵活配置集群。对于需要长时间保留的用户数据，无需重新配置 broker，只要调整存储层的大小。如果要增加处理资源，不用重新强制配置存储层，只需扩展处理层。此外，可根据每层的需求优化硬件或容器配置选择，根据存储优化存储节点，根据内存优化服务节点，根据计算资源优化处理节点。

![img](https://pic1.zhimg.com/80/v2-ae3a723b400016fb47c1288bd5c3fc20_720w.jpg)

## **IO 访问模式的优势**

传统消息系统（图 3 左侧图）中，每个 Broker 只能利用本地磁盘提供的存储容量，这会给系统带来一些限制：

\1. Broker 可以存储和服务的数据量受限于单个节点的存储容量。因此，一旦 Broker 节点的存储容量耗尽，它就不能再提供写请求，除非在写入前先清除现有的部分数据。

\2. 对于单个分区，如果需要在多个节点中存储多个备份，容量最小的节点将决定分区的最终大小。

![img](https://pic3.zhimg.com/80/v2-14e617d0740dd25f6ea7935a98e350fa_720w.jpg)

 相比之下，在 Apache Pulsar（图 3 右侧图）中，数据服务和数据存储是分离的，Pulsar 服务层的任意 Broker 都可以访问存储层的所有存储节点，并利用所有节点的整体存储容量。在服务层，从系统可用性的角度来看，这也有着深远的影响，只要任一个 Pulsar 的 Broker 还在运行，用户就可以通过这个 Broker 读取先前存储在集群中的任何数据，并且还能够继续写入数据。



流系统中通常有三种 IO 访问模式：

\1. **写（Writes）**：将新数据写入系统中；

\2. **追尾读（Tailing Reads）**：读取最近写入的数据；

\3. **追赶读（Catch-up Reads）**：读取历史的数据。例如当一个新消费者想要从较早的时间点开始访问数据，或者当旧消费者长时间离线后又恢复时。

和大多数其他消息系统不同，Pulsar 中这些 IO 访问模式中的每一种都与其他模式隔离。在同样 IO 访问模式下，我们来对比下 Pulsar 和其他传统消息系统（存储和服务绑定在单个节点上，如 Apache Kafka）的不同。

**写**

在传统消息系统架构中，一个分区的所有权会分配给 Leader Broker。对于写请求，该 Leader Broker 接受写入并将数据复制到其他 Broker。如图 4 左侧所示，数据首先写入 Leader Broker 并复制给其他 followers。数据的一次持久化写入的过程需要两次网络往返。

在 Pulsar 系统架构中，数据服务由无状态 Broker 完成，而数据存储在持久存储中。数据会发送给服务该分区的 Broker，该 Broker 并行写入数据到存储层的多个节点中。一旦存储层成功写入数据并确认写入，Broker 会将数据缓存在本地内存中以提供追尾读（Tailing Reads）。

![img](https://pic1.zhimg.com/80/v2-033160982a7d2ac120369ca425db66e0_720w.jpg)

如图 4 所示，和传统的系统架构相比，Pulsar 的系统架构并不会在写入的 IO 路径上引入额外的网络往返或带宽开销。 而存储和服务的分离则会显著提高系统的灵活性和可用性。

**追尾读**

对于读取最近写入的数据场景，在传统消息系统架构中，消费者从 Leader Broker 的本地存储中读取数据；在 Pulsar 的分层架中，消费者从 Broker 就可以读取数据，由于 Broker 已经将数据缓存在内存中，并不需要去访问存储层。

![img](https://pic1.zhimg.com/80/v2-02803cb122363cd68035b32a03af290c_720w.jpg)

这两种架构只需要一次网络往返就可以读取到数据。由于 Pulsar 在系统中自己管理缓存中的数据，没有依赖文件系统缓存，这样 Tailing Reads 很容易在缓存中命中，而无需从磁盘读取。传统的系统架构一般依赖于文件系统的缓存，读写操作不仅会相互竞争资源（包括内存），还会与代理上发生的其他处理任务竞争。因此，在传统的单片架构中实现缓存并扩展非常困难。

**追赶读**

追赶读（**Catch-up Reads**）非常有趣。传统的系统架构对 Tailing reads 和 Catch-up reads 两种访问模式进行了同样的处理。即使一份数据存在多个 Broker 中，所有的 Catch-up reads 仍然只能发送给 Leader Broker。

Pulsar 的分层架构中历史（旧）数据存储在存储层中。Catch-up 读可以通过存储层并行读取数据，而不会与 Write 和 Tailing Reads 两种 IO 模式竞争或干扰。

**三种 IO 模式放在一起看**

最有趣的是当你把这些不同的模式放在一起时，也就是实际发生的情况。这也正是单体架构的局限性最令人痛苦的地方。传统的消息系统架构中，所有不同的工作负载都被发送到一个中心（Leader Broker）位置，几乎不可能在工作负载之间提供任何隔离。

然而，Pulsar 的分层架构可以很容易地隔离这些 IO 模式：服务层的内存缓存为 Tailing Reads 这种消费者提供最新的数据；而存储层则为历史处理和数据分析型的消费者提供数据读取服务。

![img](https://pic3.zhimg.com/80/v2-32848cb286ab0adff0a524f131f46552_720w.jpg)

这种 IO 隔离是 Pulsar 和传统消息系统的根本差异之一，也是 Pulsar 可用于替换多个孤立系统的关键原因之一。Apache Pulsar 的存储架构读、写分离，能保证性能的一致性，不会引起数据发布和数据消费间的资源竞争。已发布数据的写入传递到存储层进行处理，而当前数据直接从 broker 内存缓存中读取，旧数据直接从存储层读取。



## **超越传统消息系统**

上面讨论了 Pulsar 的分层架构如何为不同类型的工作负载提供高性能和可扩展性。Pulsar 分层架构带来的好处远远不止这些。我举几个例子。

**无限的流存储**

并行访问流式计算中的最新数据和批量计算中的历史数据，是业界一个普遍的需求。

由于 Pulsar 基于分片的架构，Pulsar 的一个主题在理论上可以达到无限大小。当容量不足时，用户只需要添加容器或存储节点即可轻松扩展存储层，而无需重新平衡数据；新添加的存储节点会被立即用于新的分片或者分片副本的存储。

Pulsar 将无界的数据看作是分片的流，分片分散存储在分层存储（tiered storage）、BookKeeper 集群和 Broker 节点上，而对外提供一个统一的、无界数据的视图。其次，不需要用户显式迁移数据，减少存储成本并保持近似无限的存储。因此，Pulsar 不仅可以存储当前数据，还可以存储完整的历史数据。

![img](https://pic4.zhimg.com/80/v2-f88e6021b2bf14e6e751be20f997fe23_720w.jpg)



### 链接 https://zhuanlan.zhihu.com/p/99800206

Pulsar采用“存储和服务分离”的两层架构（这是Pulsar区别于其他MQ系统最重要的一点，也是所谓的“下一代消息系统”的核心）：

- Broker：提供发布和订阅的服务（Pulsar的组件）
- Bookie：提供存储能力（BookKeeper的存储组件）

优势是Broker成为了stateless的组件，可以水平扩容（RocketMQ的Broker是包含存储的，是有状态的，Broker的扩容更像是“拆分”）。高可靠，一致性等通过BookKeeper去保证。

<img src="http://ifeve.com/wp-content/uploads/2018/07/pulsar-system-architecture.png" alt="img" style="zoom:67%;" />

上图是Pulsar Cluster的架构：

- 采用ZooKeeper存储元数据，集群配置，作为coordination
  - local zk负责Pulsar Cluster内部的配置等
  - global zk则用于Pulsar Cluster之间的数据复制等
- 采用Bookie作为存储设备(大多数MQ系统都采用本地磁盘或者DB作为存储设备)
- Broker负责负载均衡和消息的读取、写入等
- Global replicators负责集群间的数据复制

### 链接 [http://ifeve.com/apache-pulsar%E4%BB%8B%E7%BB%8D/](http://ifeve.com/apache-pulsar介绍/)



### 下面2篇文章 讲了设计!!!!(有兴趣研究下..)

在这篇文章中，我们将介绍Apache Pulsar的设计，这篇文章不适合想要了解如何使用Apache Pulsar的读者，适合想要了解Apache Pulsar是如何工作的读者。

## **设计核心**

- 保证不丢失消息(使用正确的配置且不是整个数据中心故障)
- 强顺序性保证
- 可预测的读写延迟

Apache Pulsar选择一致性而不是可用性就像BookKeeper和Zookeeper一样。Apache Pulsar尽一切努力保持一致性。

## **多层抽象**

Apache Pulsar在上层具有高级别的Topic(主题)和Subscription(订阅)的概念，在底层数据存储在二进制文件中，这些数据交叉分布在多个服务器上的多个Topic。在其中包含很多的细节部分。我个人认为把它分成不同的抽象层更容易理解Apache Pulsar的架构设计，所以这就是我在这篇文章中要做的事情。



接下来我们按照下图，一层一层的进行分析。

![img](https://mmbiz.qpic.cn/mmbiz_png/4ibRRsibIGr0aNVate6WTldDhAUgkPwmh6oLMrTicSWtJlbf3icsiadlm4EdpEPueTCr3OcibIlwfRzRibquHpYmof27w/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

### 第一层 - Topic、Subscription和Cursors

我们将要简要介绍Topic(主题)、Subsription(订阅)和Cursors(游标)的基本概念，不会包含深层次的消息传递方式。

<img src="https://mmbiz.qpic.cn/mmbiz_jpg/4ibRRsibIGr0al9tFBy2SftYhbxaYRsicWFRFya3noD8wvdpa1AG7F7wPvb4lsLRcddxcLEySE4z9r4pfibR3lzaBA/640?wx_fmt=jpeg&amp;tp=webp&amp;wxfrom=5&amp;wx_lazy=1&amp;wx_co=1" alt="img" style="zoom:67%;" />

消息存储在Topic中。逻辑上一个Topic是日志结构，每个消息都在这个日志结构中有一个偏移量。Apache Pulsar使用游标来跟踪偏移量。生产者将消息发送到一个指定的Topic，Apache Pulsar保证消息一旦被确认就不会丢失(正确的配置和非整个集群故障的情况下)。

消费者通过订阅来消费Topic中的消息。订阅是游标(跟踪偏移量)的逻辑实体，并且还根据不同的订阅类型提供一些额外的保证

- Exclusive(独享) - 一个订阅只能有一个消息者消费消息
- Shared(共享) - 一个订阅中同时可以有多个消费者，多个消费者共享Topic中的消息
- Fail-Over(灾备) - 一个订阅同时只有一个消费者，可以有多个备份消费者。一旦主消费者故障则备份消费者接管。不会出现同时有两个活跃的消费者。

一个Topic可以添加多个订阅。订阅不包含消息的数据，只包含元数据和游标。

Apache Pulsar通过允许消费者将Topic看做在消费者消费确认后删除消息的队列，或者消费者可以根据游标的回放来提供队列和日志的语义。在底层都使用日志作为存储模型。



### **第二层(1) - 逻辑存储模型**

现在该介绍Apache BookKeeper了。我将在Apache Pulsar的背景下讨论BookKeeper，尽管BookKeeper是一个通用的分布式日志存储解决方案。

首先，BookKeeper将数据存储至集群中的节点上，每个BookKeeper节点称为Bookie。其次，Pulsar和BookKeeper都使用Apache Zookeeper来存储元数据和监控节点健康状况。

<img src="https://mmbiz.qpic.cn/mmbiz_png/4ibRRsibIGr0aNVate6WTldDhAUgkPwmh6KBUxJQPMeb66UzicA9PQia4iasCQEA3fw83A2JuasUjE0LibiaXTHiciczA1w/640?wx_fmt=png&amp;tp=webp&amp;wxfrom=5&amp;wx_lazy=1&amp;wx_co=1" alt="img" style="zoom:80%;" />







### 链接 https://mp.weixin.qq.com/s/CIpCLCxqpLoQVUKz6QeDJQ



![img](https://alexstocks.github.io/pic/pulsar/pulsar_vs_kafka.webp)







### 链接 https://alexstocks.github.io/html/pulsar.html



## Broker层–无状态服务层

Broker集群在Apache Pulsar中形成无状态服务层。服务层是“无状态的”，因为Broker实际上并不在本地存储任何消息数据。有关Pulsar主题的消息，都被存储在分布式日志存储系统（Apache BookKeeper）中。我们将在下一节中更多地讨论BookKeeper。
每个主题分区（Topic Partition）由Pulsar分配给某个Broker，该Broker称为该主题分区的所有者。 Pulsar生产者和消费者连接到主题分区的所有者Broker，以向所有者代理发送消息并消费消息

如果一个Broker失败，Pulsar会自动将其拥有的主题分区移动到群集中剩余的某一个可用Broker中。这里要说的一件事是：由于Broker是无状态的，当发生Topic的迁移时，Pulsar只是将所有权从一个Broker转移到另一个Broker，在这个过程中，不会有任何数据复制发生。

下图显示了一个拥有4个Broker的Pulsar集群，其中4个主题分区分布在4个Broker中。每个Broker拥有并为一个主题分区提供消息服务。

<img src="https://img-blog.csdnimg.cn/20190308113029895.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzgzMjg0Ng==,size_16,color_FFFFFF,t_70" alt="å¨è¿éæå¥å¾çæè¿°" style="zoom: 50%;" />

## BookKeeper层–持久化存储层

Apache BookKeeper是Apache Pulsar的持久化存储层。 Apache Pulsar中的每个主题分区本质上都是存储在Apache BookKeeper中的分布式日志。
每个分布式日志又被分为Segment分段。 每个Segment分段作为Apache BookKeeper中的一个Ledger，均匀分布并存储在BookKeeper群集中的多个Bookie（Apache BookKeeper的存储节点）中。Segment的创建时机包括以下几种：基于配置的Segment大小；基于配置的滚动时间；或者当Segment的所有者被切换。

通过Segment分段的方式，主题分区中的消息可以均匀和平衡地分布在群集中的所有Bookie中。 这意味着主题分区的大小不仅受一个节点容量的限制； 相反，它可以扩展到整个BookKeeper集群的总容量。

下面的图说明了一个分为x个Segment段的主题分区。 每个Segment段存储3个副本。 所有Segment都分布并存储在4个Bookie中。

<img src="https://img-blog.csdnimg.cn/2019030811304473.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzgzMjg0Ng==,size_16,color_FFFFFF,t_70" alt="å¨è¿éæå¥å¾çæè¿°" style="zoom: 50%;" />

## Segment为中心的存储

存储服务的分层的架构 和 以Segment为中心的存储 是Apache Pulsar（使用Apache BookKeeper）的两个关键设计理念。 这两个基础为Pulsar提供了许多重要的好处：
无限制的主题分区存储
即时扩展，无需数据迁移
无缝Broker故障恢复
无缝集群扩展
无缝的存储（Bookie）故障恢复
独立的可扩展性

下面我们分别展开来看着几个好处。

**无限制的主题分区存储**
由于主题分区被分割成Segment并在Apache BookKeeper中以分布式方式存储，因此主题分区的容量不受任何单一节点容量的限制。 相反，主题分区可以扩展到整个BookKeeper集群的总容量，只需添加Bookie节点即可扩展集群容量。 这是Apache Pulsar支持存储无限大小的流数据，并能够以高效，分布式方式处理数据的关键。 使用Apache BookKeeper的分布式日志存储，对于统一消息服务和存储至关重要。

**即时扩展，无需数据迁移**
由于消息服务和消息存储分为两层，因此将主题分区从一个Broker移动到另一个Broker几乎可以瞬时内完成，而无需任何数据重新平衡（将数据从一个节点重新复制到另一个节点）。 这一特性对于高可用的许多方面至关重要，例如集群扩展；对Broker和Bookie失败的快速应对。 我将使用例子在下文更详细地进行解释。

**无缝Broker故障恢复**
下图说明了Pulsar如何处理Broker失败的示例。 在例子中Broker 2因某种原因（例如停电）而断开。 Pulsar检测到Broker 2已关闭，并立即将Topic1-Part2的所有权从Broker 2转移到Broker 3。在Pulsar中数据存储和数据服务分离，所以当代理3接管Topic1-Part2的所有权时，它不需要复制Partiton的数据。 如果有新数据到来，它立即附加并存储为Topic1-Part2中的Segment x + 1。 Segment x + 1被分发并存储在Bookie1, 2和4上。因为它不需要重新复制数据，所以所有权转移立即发生而不会牺牲主题分区的可用性。

<img src="https://img-blog.csdnimg.cn/20190308113118156.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzgzMjg0Ng==,size_16,color_FFFFFF,t_70" alt="å¨è¿éæå¥å¾çæè¿°" style="zoom:50%;" />

**无缝集群容量扩展**
下图说明了Pulsar如何处理集群的容量扩展。 当Broker 2将消息写入Topic1-Part2的Segment X时，将Bookie X和Bookie Y添加到集群中。 Broker 2立即发现新加入的Bookies X和Y。然后Broker将尝试将Segment X + 1和X + 2的消息存储到新添加的Bookie中。 新增加的Bookie立刻被使用起来，流量立即增加，而不会重新复制任何数据。 除了机架感知和区域感知策略之外，Apache BookKeeper还提供资源感知的放置策略，以确保流量在群集中的所有存储节点之间保持平衡。

<img src="https://img-blog.csdnimg.cn/20190308113125537.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzgzMjg0Ng==,size_16,color_FFFFFF,t_70" alt="å¨è¿éæå¥å¾çæè¿°" style="zoom:50%;" />



**无缝的存储（Bookie）故障恢复**
下图说明了Pulsar（通过Apache BookKeeper）如何处理bookie的磁盘故障。 这里有一个磁盘故障导致存储在bookie 2上的Segment 4被破坏。Apache BookKeeper后台会检测到这个错误并进行复制修复。

<img src="https://img-blog.csdnimg.cn/20190308113133356.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzgzMjg0Ng==,size_16,color_FFFFFF,t_70" alt="å¨è¿éæå¥å¾çæè¿°" style="zoom:50%;" />

Apache BookKeeper中的副本修复是Segment（甚至是Entry）级别的多对多快速修复，这比重新复制整个主题分区要精细，只会复制必须的数据。 这意味着Apache BookKeeper可以从bookie 3和bookie 4读取Segment 4中的消息，并在bookie 1处修复Segment 4。所有的副本修复都在后台进行，对Broker和应用透明。
即使有Bookie节点出错的情况发生时，通过添加新的可用的Bookie来替换失败的Bookie，所有Broker都可以继续接受写入，而不会牺牲主题分区的可用性。

**独立的可扩展性**
由于消息服务层和持久存储层是分开的，因此Apache Pulsar可以独立地扩展存储层和服务层。这种独立的扩展，更具成本效益：
当您需要支持更多的消费者或生产者时，您可以简单地添加更多的Broker。主题分区将立即在Brokers中做平衡迁移，一些主题分区的所有权立即转移到新的Broker。
当您需要更多存储空间来将消息保存更长时间时，您只需添加更多Bookie。通过智能资源感知和数据放置，流量将自动切换到新的Bookie中。 Apache Pulsar中不会涉及到不必要的数据搬迁，不会将旧数据从现有存储节点重新复制到新存储节点。

## 和Kafka的对比

Apache Kafka和Apache Pulsar都有类似的消息概念。 客户端通过主题与消息系统进行交互。 每个主题都可以分为多个分区。 然而，Apache Pulsar和Apache Kafka之间的根本区别在于Apache Kafka是以分区为存储中心，而Apache Pulsar是以Segment为存储中心。

![å¨è¿éæå¥å¾çæè¿°](https://img-blog.csdnimg.cn/20190308113144324.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzgzMjg0Ng==,size_16,color_FFFFFF,t_70)

上图显示了以分区为中心和以Segment为中心的系统之间的差异。
在Apache Kafka中，分区只能存储在单个节点上并复制到其他节点，其容量受最小节点容量的限制。这意味着容量扩展需要对分区重新平衡，这反过来又需要重新复制整个分区，以平衡新添加的代理的数据和流量。
重新传输数据非常昂贵且容易出错，并且会消耗网络带宽和I/O。维护人员在执行此操作时必须非常小心，以避免破坏生产系统。
Kafka中分区数据的重新拷贝不仅发生在以分区为中心的系统中的群集扩展上。许多其他事情也会触发数据重新拷贝，例如副本故障，磁盘故障或计算机的故障。在数据重新复制期间，分区通常不可用，直到数据重新复制完成。例如，如果您将分区配置为存储为3个副本，这时，如果丢失了一个副本，则必须重新复制完整个分区后，分区才可以再次可用。
在用户遇到故障之前，通常会忽略这种缺陷，因为许多情况下，在短时间内仅是对内存中缓存数据的读取。当数据被保存到磁盘后，用户将越来越多地不可避免地遇到数据丢失，故障恢复的问题，特别是在需要将数据长时间保存的场合。

相反，在Apache Pulsar中，同样是以分区为逻辑单元，但是以Segment为物理存储单元。分区随着时间的推移会进行分段，并在整个集群中均衡分布，旨在有效地迅速地扩展。
Pulsar是以Segment为中心的，因此在扩展容量时不需要数据重新平衡和拷贝，旧数据不会被重新复制，这要归功于在Apache BookKeeper中使用可扩展的以Segment为中心的分布式日志存储系统。
通过利用分布式日志存储，Pulsar可以最大化Segment放置选项，实现高写入和高读取可用性。 例如，使用BookKeeper，副本设置等于2，只要任何2个Bookie启动，就可以对主题分区进行写入。 对于读取可用性，只要主题分区的副本集中有1个处于活动状态，用户就可以读取它，而不会出现任何不一致。

## 总结

总之，Apache Pulsar这种独特的基于分布式日志存储的以Segment为中心的发布/订阅消息系统可以提供许多优势，例如可靠的流式系统，包括无限制的日志存储，无需分区重新平衡的即时扩展，快速复制修复以及通过最大化数据放置实现高写入和读取可用性选项。

### 链接 : https://my.oschina.net/xiaominmin/blog/4287127/print



## ZeroMQ

## ActiveMQ 

## CMQ(腾讯的, 参考了RabbitMQ)

腾讯云 CMQ 基于 RabbitMQ/AMQP 高可靠的原理，并使用 Raft 协议重新设计实现，在可靠性、吞吐量和性能上有了大幅提升。

#### 比较:

RabbitMQ 使用生产消息确认、消费者确认机制来提供可靠交付功能。

- 生产消息确认：生产者向MQ发送消息后，等待 MQ 回复确认成功；否则生产者向 MQ 重发该消息。此过程可以异步进行，生产者持续发送消息，MQ 将消息批量处理后再回复确认；生产者通过识别确认返回中的 ID 来确定哪些消息被成功处理。
- 消费者确认：MQ 向消费者投递消息后，等待消费者回复确认成功；否则 MQ 重新向消费者投递该消息。该过程同样可以异步处理，MQ 持续投递消息，消费者批量处理完后回复确认。

可以看出 RabbitMQ/AMQP 提供的是“至少一次交付”（at-least-once delivery），异常情况下，消息会被重复投递或消费。

### 消息存储

为提高消息的可靠性，保证在 RabbitMQ 重启服务不可用时，要对收到的消息持久化写入磁盘。在收到消息时 RabbitMQ 将消息写入文件中，当写入达到一定数量或一定时间周期后 RabbitMQ 将文件落盘存储。

生产消息确认就是在消息落盘存储后，MQ向生产者回复已落盘存储的消息 ID。

----

### 可用性提升

CMQ 和 RabbitMQ 都能够使用多台机器进行热备份，提高可用性。CMQ 基于 Raft 算法实现，简单易维护。RabbitMQ 使用自创的 GM 算法（Guaranteed Multicast），学习难度较高。

Raft 协议中，Log 复制只要大多数节点向 Leader 返回成功，Leader 就可以应用该请求，向客户端返回成功：

![img](https://main.qcloudimg.com/raw/9427c939705d656ba71a54c0d9a31f5f.png)
　　　
GM 可靠多播将集群中所有节点组成一个环。Log 复制依次从 Leader 向后继节点传播，当 Leader 再次收到该请求时，发出确认消息在环中传播，直至 Leader 再次收到该确认消息，表明 Log 在环中所有节点同步完成。

![img](https://main.qcloudimg.com/raw/c1602a66ea72818122693805b674b9a1.png)

GM 算法要求 Log 在集群所有节点同步之后才能向客户端返回成功；Raft 算法则只要求大多数节点同步完成。Raft 算法在同步路径上比 GM 算法减少了近一半的等待时间。



### MongoDB

MongoDB 本身就拥有高可用及分区的解决方案，分别为副本集(Replica Set)和分片(sharding)

**副本集**

![1570700521288917](C:\Users\Administrator\Pictures\Saved Pictures\1570700521288917.png)



首先，我们看一下 MongoDB 副本集的各种角色。

- Primary：主服务器，只有一组，处理客户端的请求，一般是读写
- Secondary：从服务器，有多组，保存主服务器的数据副本，主服务器出问题时其中一个从服务器可提升为新主服务器，可提供只读服务
- Hidden：一般只用于备份节点，不处理客户端的读请求
- Secondary-Only：不能成为 primary 节点，只能作为 secondary 副本节点，防止一些性能不高的节点成为主节点
- Delayed：slaveDelay 来设置，为不处理客户端请求，一般需要隐藏
- Non-Voting：没有选举权的 secondary 节点，纯粹的备份数据节点。
- Arbiter：仲裁节点，不存数据，只参与选举，可用可不用

然后我们思考一下 MongoDB 副本集是通过什么方式去进行同步数据的，我们了解 Oracle 的 DataGuar 同步模式，我们也了解 MySQL 主从同步模式，他们都是传输日志到备库然后应用的方法，那么不难想象，MongoDB 的副本集基本也是这个路子，这里就不得不提到同步所依赖的核心 Oplog。Oplog 其实就像 MySQL 的 Binlog 一样，记录着主节点上执行的每一个操作，而 Secondary 通过复制 Oplog 并应用的方式来进行数据同步。

大家还可能会问 MongoDB 副本集是实时同步吗？这其实也是在问数据库一致性的问题。MySQL 的半同步复制模式保证数据库的强一致，Oracle DataGuard 的最大保护模式也能够保证数据库的强一致，而 MongoDB 可以通过 getLastError 命令来保证写入的安全，但其毕竟不是事务操作，无法做到数据的强一致。

MongoDB 副本集 Secondary 通常会落后几毫秒，如果有加载问题、配置错误、网络故障等原因，延迟可能会更大。



###### 官方文档

# 复制集成员

MongoDB的 *复制集* 是由一组 [`mongod`](https://mongoing.com/docs/reference/program/mongod.html#bin.mongod) 实例所组成的，并提供了数据冗余与高可用性。复制集中的成员有以下几种：

[*Primary*](https://mongoing.com/docs/core/replica-set-members.html#replica-set-primary-member). 

The primary receives all write operations. (主节点接受所有的写操作)

[*Secondaries*](https://mongoing.com/docs/core/replica-set-members.html#replica-set-secondary-members).

从节点通过应用主节点传来的数据变动操作来保持其数据集与主节点的一致。从节点也可以通过增加额外的参数配置来对应特殊的需求。例如，从节点可以是 [*non-voting*](https://mongoing.com/docs/core/replica-set-elections.html#replica-set-non-voting-members) 或是 [*priority 0*](https://mongoing.com/docs/core/replica-set-priority-0-member.html#replica-set-secondary-only-members) 。

我们也可以为复制集设置一个 [*投票节点*](https://mongoing.com/docs/core/replica-set-members.html#replica-set-arbiters) 。投票节点其本身并不包含数据集。但是，一旦当前的主节点不可用时，投票节点就会参与到新的主节点选举的投票中。

一个复制集至少需要这几个成员：一个 [*主节点*](https://mongoing.com/docs/core/replica-set-members.html#replica-set-primary-member) ，一个 [*从节点*](https://mongoing.com/docs/core/replica-set-members.html#replica-set-secondary-members) ，和一个 [*投票节点*](https://mongoing.com/docs/core/replica-set-members.html#replica-set-arbiters) 。但是在大多数情况下，我们会保持3个拥有数据集的节点：一个 [*主节点*](https://mongoing.com/docs/core/replica-set-members.html#replica-set-primary-member) 和两个 [*从节点*](https://mongoing.com/docs/core/replica-set-members.html#replica-set-secondary-members) 。



# 复制集主节点

在复制集中，主节点是唯一能够接收写请求的节点。MongoDB在 [*主节点*](https://mongoing.com/docs/reference/glossary.html#term-primary) 上进行写操作，并会将这些操作记录到主节点的 [*oplog*](https://mongoing.com/docs/core/replica-set-oplog.html) 中。 [*从节点*](https://mongoing.com/docs/core/replica-set-members.html#replica-set-secondary-members) 会将oplog复制到其本机并将这些操作应用到其自己的数据集上。

在拥有下述三个成员的复制集中，主节点将接收所有的写请求，而从节点会将oplog复制到本机并在其自己的数据集上应用这些操作。

![Diagram of default routing of reads and writes to the primary.](https://mongoing.com/docs/_images/replica-set-read-write-operations-primary.png)

复制集中任何成员都可以接收读请求。但是默认情况下，应用程序会直接连接到在主节点上进行读操作。阅读 [*复制集读选项*](https://mongoing.com/docs/core/read-preference.html) 可以获得更多关于更改默认读请求目标的信息。

复制集最多只能拥有一个主节点。一旦当前的主节点不可用了，复制集就会选举出新的主节点。参见 [*复制集选举*](https://mongoing.com/docs/core/replica-set-elections.html) 获得更多信息。

# 复制集从节点

从节点的数据集与 [*主节点*](https://mongoing.com/docs/reference/glossary.html#term-primary) 中的一致。从节点将主节点上的 [*oplog*](https://mongoing.com/docs/core/replica-set-oplog.html) 复制到本机，并异步的将这些操作记录应用在其自己的数据集上。每个复制集可以拥有多个从节点。

下述由三个成员组成的复制集拥有两个从节点。这些从节点将主节点上的oplog复制过来并应用在其自己的数据集上。

![Diagram of a 3 member replica set that consists of a primary and two secondaries.](https://mongoing.com/docs/_images/replica-set-primary-with-two-secondaries.png)

客户端虽然无法在从节点上进行写操作，但却可以进行读操作。阅读 [*复制集读选项*](https://mongoing.com/docs/core/read-preference.html) 可以获得更多有关如何让客户端直接对复制集节点进行读操作的信息。

从节点是可以升职为主节点的。一旦现有的主节点不可用了，那么复制集将会发起 [*election*](https://mongoing.com/docs/reference/glossary.html#term-election) 来选择将哪个从节点提升为新的主节点。在拥有下述三个成员的复制集中，一旦当前主节点不可用了，就会触发选举机制，并将在剩下的从节点中选举出一个新的主节点。

包括:

![image-20200624145810241](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20200624145810241.png)

# 投票节点

投票节点 **并不含有** 复制集中的数据集副本，且也 **无法** 升职为主节点。复制集中可能会有多个投票节点来为 [*选举出新的主节点*](https://mongoing.com/docs/core/replica-set-elections.html#replica-set-elections) 进行投票。投票节点的存在使得复制集可以以偶数个节点存在，而无需为复制集再新增节点。

仅仅在复制集成员为偶数个的时候加入投票节点。如果在拥有奇数个复制集成员的复制集中新增了一个投票节点，复制集可能会遇到 [*选举*](https://mongoing.com/docs/reference/glossary.html#term-election) 僵局。关于如何新增一个投票节点请参考 [*为复制集添加投票节点*](https://mongoing.com/docs/tutorial/add-replica-set-arbiter.html) 

# 复制集Oplog

The [*oplog*](https://mongoing.com/docs/reference/glossary.html#term-oplog) (operations log) is a special [*capped collection*](https://mongoing.com/docs/reference/glossary.html#term-capped-collection) that keeps a rolling record of all operations that modify the data stored in your databases. MongoDB applies database operations on the [*primary*](https://mongoing.com/docs/reference/glossary.html#term-primary) and then records the operations on the primary’s oplog. The [*secondary*](https://mongoing.com/docs/reference/glossary.html#term-secondary) members then copy and apply these operations in an asynchronous process. All replica set members contain a copy of the oplog, in the [`local.oplog.rs`](https://mongoing.com/docs/reference/local-database.html#local.oplog.rs) collection, which allows them to maintain the current state of the database.

```
oplog（操作日志）是一个特殊的，有上限的集合，该记录保持所有修改存储在数据库中的数据的操作的滚动记录。 MongoDB在主数据库上应用数据库操作，然后在主数据库的操作日志中记录这些操作。然后，从节点成员将这些操作复制并应用到异步过程中。所有副本集成员都在local.oplog.rs集合中包含操作日志的副本，这使他们可以维护数据库的当前状态。
```

为了提高复制的效率，复制集中所有节点之间会互相进行心跳检测（通过ping）。每个节点都可以从任何其他节点上获取oplog。

Each operation in the oplog is [*idempotent*](https://mongoing.com/docs/reference/glossary.html#term-idempotent). That is, oplog operations produce the same results whether applied once or multiple times to the target dataset.

```
操作日志中的每个操作都是幂等的。也就是说，oplog操作会产生相同的结果，无论是一次还是多次应用于目标数据集。
```



## 故障发现与恢复

对于复制集而言，选举是相对独立的操作，但是也需要时间来全部完成的。当选举开始的时候，复制集中没有主节点也不能接收处理写请求。除非遇到必要的情况，Mongodb是尽量不进行选举的。

### 2.1 故障发现

### 复制集成员每两秒向复制集中其他成员进行心跳检测。如果某个节点在10秒内没有返回，那么它将被标记为不可用。

副本集的每个节点之间会维持着两秒一次的心跳检测，当从节点与主节点的通信时间超过配置的 electionTimeoutMillis 时间 (默认为10秒) 时，则认为该主节点出现故障，此时副本集会进行新的主节点选举

如果复制集中的某个节点不能连接上其他多数节点，那么它将不能升职为主节点。在选举中，多数是指多数 *投票* 而不是多数节点个数。

### 2.2 优先选举

MongoDB 的选举算法会尝试让高优先级的节点优先发起选举，从而更容易在选举中胜出。如果某一个优先级较低的节点在短时间内被选举为新的主节点，这时副本集仍然会继续发起选举，直至可用的最高优先级的节点成为新的主节点。需要特别注意的是在这个过程当中，优先级为 0 的成员不能寻求选举，也不能成为主节点。

![img](https://user-gold-cdn.xitu.io/2020/1/7/16f7f1ce9f64edd9?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

##### 如果复制集是由三个节点组成的，且三个节点均可投票，只要其中两个节点能够互相沟通那么复制集就能选举出新的主节点。如果有两个节点不可用了，那么剩下的节点将为 [*从节点*](https://mongoing.com/docs/reference/glossary.html#term-secondary) ，因为它不能与复制集中多数节点进行沟通。 如果两个从节点不可用了，剩下的 [*主节点*](https://mongoing.com/docs/reference/glossary.html#term-primary) 将降职为从节点。

### 2.3 投票成员

节点发起选举后，需要具有投票权的节点进行投票，当获得半数以上选票时，该备用节点可以成为新的主节点。对于一个复制集，只有处于以下状态的节点拥有投票权，这些节点统称为投票成员：

- **PRIMARY**：副本集的主节点。
- **SECONDARY**：处于复制状态的从节点。
- **STARTUP2**：mongod 完成配置加载后，副本集的每个成员都进入 STARTUP2 状态，此时它成为副本集的活动成员并且有资格投票。然后该成员决定是否进行初始同步。如果成员开始初始同步，则成员将保留在 STARTUP2 状态，直到所有数据复制完成并构建好索引。之后成员过渡到 RECOVERING。
- **RECOVERING**：当副本集的成员尚未准备好接受读取操作时，它将进入 RECOVERING 状态。处于恢复状态的成员有资格在选举中投票，但没有资格成为主节点。
- **ARBITER**：ARBITER 状态的成员不复制数据或接受写入操作，仲裁者通常处于这一状态。
- **ROLLBACK**：如果成员正在进行数据回滚，它就处于 ROLLBACK 状态。

除了投票成员外，那些持有副本集数据的副本，并且可以接受来自客户端应用程序的读取操作，但没有投票权的成员统称为非投票成员。出于网络通讯成本的考虑，MongoDB 的副本集最多有 50 个节点（成员），默认情况下最多可包含7个投票成员。投票成员和非投票成员的要求如下：

- 非投票成员的优先级 `members[n].priority` 必须为 0，优先级为 0 的成员不能寻求选举，也不能成为主节点。
- 优先级大于 0 的节点持有的票数 `members[n].votes` 不能为 0，该参数默认值为 1，可选值为 1 或 0。

如下示例是一个 9 个成员的副本集，包含 7 个投票成员和 2 个无投票成员：

 ![img](https://user-gold-cdn.xitu.io/2020/1/7/16f7f1caa314865b?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

# 故障切换时的回滚

之前的 [*主节点*](https://mongoing.com/docs/reference/glossary.html#term-primary) 在 a [*故障切换*](https://mongoing.com/docs/reference/glossary.html#term-failover) 后重新回归 [*复制集*](https://mongoing.com/docs/reference/glossary.html#term-replica-set) 时将会发生写操作的回滚。回滚只会发生在主节点的写操作 **没能** 成功在 [*从节点*](https://mongoing.com/docs/reference/glossary.html#term-secondary) 上应用就Down 掉的情况下。当主节点重新以一个从节点加入复制集，它将进行 “回滚” ，其上得写操作将与复制集中其他成员的保持一致。

MongoDB会尽量避免回滚的发生。回滚如果确实发生了，往往是由于网络导致的。从节点如果无法跟上之前主节点上的写操作的吞吐，那么将会加剧回滚的影响范围。

当主节点在从节点完成写操作的复制后才Down 掉的或是主节点一直是可用的或是与多数节点可以沟通的，将 *不会* 发生回滚。

## 选取回滚的数据

When a rollback does occur, MongoDB writes the rollback data to [*BSON*](https://mongoing.com/docs/reference/glossary.html#term-bson) files in the `rollback/` folder under the database’s [`dbPath`](https://mongoing.com/docs/reference/configuration-options.html#storage.dbPath) directory. The names of rollback files have the following form:

发生回滚后，MongoDB会将回滚数据写入数据库dbPath目录下rollback /文件夹中的BSON文件中。回滚文件的名称具有以下形式：

```shell
<database>.<collection>.<timestamp>.bson
```

例如：

```shell
records.accounts.2011-05-09T18-10-04.0.bson
```

To read the contents of the rollback files, use [*bsondump*](https://mongoing.com/docs/reference/program/bsondump.html). Based on the content and the knowledge of their applications, administrators can decide the next course of action to take.

要读取回滚文件的内容，请使用[* bsondump *]（https://mongoing.com/docs/reference/program/bsondump.html）。根据其应用程序的内容和知识，管理员可以决定下一步要采取的行动。

## 避免复制集的回滚

我们可以通过设置 *复制集的安全写级别* 确保写操作应用到了整个复制集来避免回滚。

To prevent rollbacks of data that have been acknowledged to the client, run all voting members with journaling enabled and use [*w: majority write concern*](https://mongoing.com/docs/reference/write-concern.html#wc-w) to guarantee that the write operations propagate to a majority of the replica set nodes before returning with acknowledgement to the issuing client.

为防止回滚已确认给客户端的数据，请在启用log功能的情况下运行所有有表决权的成员，并使用w：多数写入问题来确保写入操作传播到多数副本集节点，然后返回确认给发行方客户端。



## Raft (...)

