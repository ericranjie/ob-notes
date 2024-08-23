
石杉的架构笔记

 _2022年01月14日 09:10_

以下文章来源于云加社区 ，作者雷洁彦

[

![](http://wx.qlogo.cn/mmhead/Q3auHgzwzM4VrfJGRic9cMlydQkzsTsDFptqtib3k4CxI3TOVia4Nmicpw/0)

**云加社区**.

腾讯云官方社区公众号，汇聚技术开发者群体，分享技术干货，打造技术影响力交流社区。

](https://mp.weixin.qq.com/s?__biz=MzU0OTk3ODQ3Ng==&mid=2247521891&idx=1&sn=48014eddba610375ea69afeff2a1d3ee&chksm=fba57e60ccd2f7765ac126738fb12c703597f5baad548e0be1e22d59f39291bcdacbe8d8237c&mpshare=1&scene=24&srcid=0114r3mw0bJJArar4326k1ic&sharer_sharetime=1642124006105&sharer_shareid=5fb9813bfe9ffc983435bfc8d8c5e9ca&key=daf9bdc5abc4e8d071156fb2d7728b35331575c7f9fd971d2361aae22b47f567e131b7119d820373e00b21849a91fac1e3922d195d069c19f6e462f3a7958224cd55d5dad8fa9ecf9f1baf7d7a566b8eab3ad6c1b122b2584c62f9b3f9f84e4ed12610dd4a99135af0f087fb86b2bff836d0ce47ffa1ef3215c3b03323253bdb&ascene=0&uin=MTEwNTU1MjgwMw%3D%3D&devicetype=Windows+11+x64&version=63090b19&lang=zh_CN&countrycode=CN&exportkey=n_ChQIAhIQpq451h2RBP2MCiMMl1RU5RLmAQIE97dBBAEAAAAAAEw6Lmwv7jMAAAAOpnltbLcz9gKNyK89dVj0JXtjyf8EayWlDrrgV2dOiZH5X1s7FtnAFMDgRMvuhvmjl7KQ%2FXd4lGtfdfkNv4IZ1dzoMAtGypOqLFwFXIKauL9G2vkY8yvfDMnkScx8Na6I%2BAGpevXGVqqS%2FK8Y%2FaCRBpdQHY0gEFQxmx4Pz7%2F29KyaZmzqpDUuv2OueOGSvpEH1UdB8ug8NGMtlrpaRrFQeDGW98adynPpPAUY2uzvBjuddlVFNpOeW%2FJgO3RbCPzzTRnYhz%2FHEisbgfrBVzfb&acctmode=0&pass_ticket=BdmrugC0oPmqQcLInWnMqOhw4UhHSfUky5I0R%2BevqdR5N0T9RBXJuL7R1XUZovPd&wx_header=1&fasttmpl_type=0&fasttmpl_fullversion=7350504-zh_CN-zip&fasttmpl_flag=1#)

**「** 关注**“石杉的架构笔记”**，大厂架构经验**倾囊相授** **」**

![](http://mmbiz.qpic.cn/mmbiz_png/1J6IbIcPCLaRRZTmbrgz4dZdQhIia64KMLmHFiaxtDjQqnhIj7Dxx99GCRl6W5B1EEQbhKBAibA9rzxevJ6QzAtvA/300?wx_fmt=png&wxfrom=19)

**石杉的架构笔记**

一线大厂架构经验倾囊相授！

234篇原创内容

公众号

**“从零开始带你成为JVM实战高手” 免费加餐啦！**

![图片](https://mmbiz.qpic.cn/mmbiz_png/1J6IbIcPCLYDicABS9TxA7JufWCbDsMnIrFKicLUCciaR7UHRiaqcCp1hR8ZpnTbhG6HPbTHmKD7tZIrdh4rQ80njw/640?wx_fmt=png&wxfrom=13&tp=wxpic)

[**点击查看 专栏目录**](https://mp.weixin.qq.com/s?__biz=MzU0OTk3ODQ3Ng==&mid=2247521891&idx=1&sn=48014eddba610375ea69afeff2a1d3ee&chksm=fba57e60ccd2f7765ac126738fb12c703597f5baad548e0be1e22d59f39291bcdacbe8d8237c&mpshare=1&scene=24&srcid=0114r3mw0bJJArar4326k1ic&sharer_sharetime=1642124006105&sharer_shareid=5fb9813bfe9ffc983435bfc8d8c5e9ca&key=daf9bdc5abc4e8d071156fb2d7728b35331575c7f9fd971d2361aae22b47f567e131b7119d820373e00b21849a91fac1e3922d195d069c19f6e462f3a7958224cd55d5dad8fa9ecf9f1baf7d7a566b8eab3ad6c1b122b2584c62f9b3f9f84e4ed12610dd4a99135af0f087fb86b2bff836d0ce47ffa1ef3215c3b03323253bdb&ascene=0&uin=MTEwNTU1MjgwMw%3D%3D&devicetype=Windows+11+x64&version=63090b19&lang=zh_CN&countrycode=CN&exportkey=n_ChQIAhIQpq451h2RBP2MCiMMl1RU5RLmAQIE97dBBAEAAAAAAEw6Lmwv7jMAAAAOpnltbLcz9gKNyK89dVj0JXtjyf8EayWlDrrgV2dOiZH5X1s7FtnAFMDgRMvuhvmjl7KQ%2FXd4lGtfdfkNv4IZ1dzoMAtGypOqLFwFXIKauL9G2vkY8yvfDMnkScx8Na6I%2BAGpevXGVqqS%2FK8Y%2FaCRBpdQHY0gEFQxmx4Pz7%2F29KyaZmzqpDUuv2OueOGSvpEH1UdB8ug8NGMtlrpaRrFQeDGW98adynPpPAUY2uzvBjuddlVFNpOeW%2FJgO3RbCPzzTRnYhz%2FHEisbgfrBVzfb&acctmode=0&pass_ticket=BdmrugC0oPmqQcLInWnMqOhw4UhHSfUky5I0R%2BevqdR5N0T9RBXJuL7R1XUZovPd&wx_header=1&fasttmpl_type=0&fasttmpl_fullversion=7350504-zh_CN-zip&fasttmpl_flag=1)

---

【文章来源】【公众号：云加社区】

导语 | 在信息流场景，内容的请求处理、原子模块调度、结果的分发等至关重要，将会直接影响到内容的外显、推荐、排序等。基于消息100%成功的要求，我对Pulsar进行了调研，并采用Pulsar实现消息的可靠处理。本文主要参考Pulsar的官方文档和技术文章，对Pulsar的特性、机制、原理等进行整理总结。

  

**一、Pulsar概述**

  

Apache Pulsar是Apache软件基金会顶级项目，是下一代云原生分布式消息流平台，集消息、存储、轻量化函数式计算为一体，采用计算与存储分离架构设计，支持多租户、持久化存储、多机房跨区域数据复制，具有强一致性、高吞吐、低延时及高可扩展性等流数据存储特性，被看作是云原生时代实时消息流传输、存储和计算最佳解决方案。

  

  

Pulsar是一个pub-sub (发布-订阅)模型的消息队列系统。

  

## **（一）Pulsar架构**

  

Pulsar由Producer、Consumer、多个Broker、一个BookKeeper集群、一个Zookeeper集群构成，具体如下图所示。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

- **Producer**：数据生成者，即发送消息的一方。生产者负责创建消息，将其投递到Pulsar中。
    

  

- **Consumer**：数据消费者，即接收消息的一方。消费者连接到 Pulsar 并接收消息，进行相应的业务处理。
    

  

- **Broker**：无状态的服务层，负责接收消息、传递消息、集群负载均衡等操作，Broker不会持久化保存元数据。
    

  

- **BookKeeper**：有状态的持久层，包含多个Bookie，负责持久化地存储消息。
    

  

- **ZooKeeper**：存储Pulsar、BookKeeper的元数据，集群配置等信息，负责集群间的协调(例如：Topic与Broker的关系)、服务发现等。
    

  

从Pulsar的架构图上可以看出，Pulsar在架构设计上采用了计算与存储分离的模式，发布/订阅相关的计算逻辑在Broker上完成，而数据的持久化存储交由BookKeeper去实现。

  

- ### **Broker扩展**
    

  

在Pulsar中Broker是无状态的，当需要支持更多的消费者或生产者时，可以简单地添加更多的Broker节点来满足业务需求。Pulsar支持自动的分区负载均衡，在Broker节点的资源使用率达到阈值时，会将负载迁移到负载较低的Broker节点，这个过程中分区也将在多个Broker节点中做平衡迁移，一些分区的所有权会转移到新的Broker节点。在后面Bundle小节会具体介绍这部分的实现。

  

- ### **Bookie扩展**
    

  

存储层的扩容，通过增加Bookie节点来实现。在BooKie扩容的阶段，由于分片机制，整个过程不会涉及到不必要的数据搬迁，即不需要将旧数据从现有存储节点重新复制到新存储节点。在后续的Bookkeeper小节中会具体介绍。

  

  

## **（二）Topic**

  

和其他消息队列类似，Pulsar中也有Topic。Topic即在生产者与消费者中传输消息的通道。消息可以以Topic为单位进行归类，生产者负责将消息发送到特定的Topic，而消费者指定特定的Topic进行消费。  

  

- ### **分区Topic（Topic-Partition）**
    

  

Pulsar的Topic可以分为非分区Topic和分区Topic。

  

普通的Topic仅仅被保存在单个Broker中，这限制了Topic的最大吞吐量。分区Topic是一种特殊类型的主题，支持被多个Broker处理，从而实现更高的吞吐量。

  

针对一个Topic，可以设置多个Topic分区来提高Topic的吞吐量。每个Topic Partition由Pulsar分配给某个Broker，该Broker称为该Topic Partition的所有者。生成者和消费者会与每个Topic分区的Broker创建链接，发送消息并消费消息。

  

如下图所示，Topic1有Partition1、Partition2、Partition3、Partition4、Partition5五个分区，Partition1和Partition4由Broker1处理，Partition2和Partition5由Broker2处理，Partition3由Broker3处理。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

从Pulsar社区版的golang-sdk可以看出，客户端的Producer和Consumer在初始化的时候，都会与每一个Topic-Partition创建链接，并且会监听是否有新的Partition，以创建新的链接。

  

- ### **非持久Topic**
    

  

默认情况下，Pulsar会保存所有没确认的消息到BookKeeper中。持久Topic的消息在Broker重启或者Consumer出现问题时保存下来。

  

除了持久Topic，Pulsar也支持非持久Topic。这些Topic的消息只存在于内存中，不会存储到磁盘。

  

因为Broker不会对消息进行持久化存储，当Producer将消息发送到Broker时，Broker可以立即将ack返回给Producer，所以非持久Topic的消息传递会比持久Topic的消息传递更快一些。相对的，当Broker因为一些原因宕机、重启后，非持久Topic的消息都会消失，订阅者将无法收到这些消息。

  

- ### **重试Topic**
    

  

由于业务逻辑处理出现异常，消息一般需要被重新消费。Pulsar支持生产者同时将消息发送到普通的Topic和重试Topic，并指定允许延时和最大重试次数。当配置了允许消费者自动重试时，如果消息没有被消费成功，会被保存到重试Topic中，并在指定延时时间后，重新被消费。

  

- ### **死信Topic**
    

  

当Consumer消费消息出错时，可以通过配置重试Topic对消息进行重试，但是，如果当消息超过了最大的重试次数仍处理失败时，该怎么办呢？Pulsar提供了死信Topic，通过配置deadLetterTopic，当消息达到最大重试次数的时候，Pulsar会将消息推送到死信Topic中进行保存。

  

  

## **（三）订阅（subscription）**

  

通过订阅的方式，我们可以指定消息如何投递给消费者。

  

- ### **订阅类型（Subscription type）**
    

  

Pulsar支持独占（Exclusive）、灾备（Failover）、共享（Shared）、Key_Shared这四种订阅类型。

  

**独占（Exclusive）SinglePartition**

  

Exclusive下，只允许Subscription存在一个消费者，如果多个消费者使用同一个订阅名称去订阅同一个Topic，则会报错。如下图，只有Consumer A-0可以消费数据。

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

**灾备（Failover）**

  

Failover下，一个Subscription中可以有多个消费者，但只有Master Consumer可以消费数据。当Master Consumer断开连接时，消息会由下一个被选中的Consumer进行消费。

  

- 分区Topic：Broker会按照消费者的优先级和消费名的顺序对消费者进行排序，将Topic均匀地分配给优先级最高的消费者。
    

  

- 非分区Topic：Broker会根据消费者订阅的非分区Topic的时间顺序选择消费者。
    

  

如下图，Consumer-B-0是Master Consumer。当Consumer-B-0发生问题与Broker断开连接时，Consumer-B-1将成为下一个Master Consumer来消费数据。

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

**共享（Shared）**

  

shared中，多个消费者可以绑定到同一个Subscription上。消息通过 round robin即轮询机制分发给不同的消费者，并且每个消息仅会被分发给一个消费者。当消费者断开连接，所有被发送给消费者但没有被确认的消息将被重新处理，分发给其它存活的消费者。

  

如下图, Consumer-C-1、Consumer-C-2、Consumer-C-3都可以订阅 Topic消费数据。

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

**Key_Shared**

  

Key_Shared中，多个Consumer可以绑定到同一个Subscription。消息在传递给Consumer时，具有相同键的消息只会传递给同一个Consumer。

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

- ### **订阅模式（Subscription modes）**
    

  

订阅模式有持久化和非持久化两种。**订阅模式取决于游标(cursor)的类型**。

  

创建订阅时，将创建一个相关的游标来记录最后使用的位置。当订阅的consumer重新启动时，它可以从它所消费的最后一条消息继续消费。

  

- Durable（持久订阅）：游标是持久性的，会保留消息并保持游标记录的位置。当Broker重新启动时，可以从BookKeeper中恢复游标，消息可以从游标上次记录的位置继续消费。默认情况下，都是持久化订阅。
    

  

- NonDurable（非持久订阅）：游标不是持久性的，当Broker宕机时，游标会丢失并无法恢复，所以消息无法继续从上次消费的位置开始继续消费。
    

  

一个订阅可以有一个或多个消费者。当使用者订阅主题时，它必须指定订阅名称。持久订阅和非持久订阅可以具有相同的名称，它们彼此独立。如果使用者指定了以前不存在的订阅，则会自动创建订阅。

  

默认情况下，没有任何持久订阅的Topic的消息将被标记为已删除。如果要防止消息被标记为已删除，可以为此Topic创建持久订阅。在这种情况下，只有被确认的消息才会被标记为已删除。

  

- ### **多主题订阅**
    

  

当Consumer订阅Topic时，默认指定订阅一个主题。从Pulsar的1.23.0-incubating的版本开始，Pulsar消费者可以同时订阅多个Topic。可以通过两种方式进行订阅：

  

- 正则表达式，例如：
    
    persistent://public/default/finance-.*
    

  

- 明确指定Topic列表。
    

  

  

**二、Pulsar生产者（Producer）**

  

Producer是连接topic的程序，它将消息发布到一个Pulsar broker上。

  

## **（一）访问模式**

  

消息生成者有多种模式访问Topic ，可以使用以下几种方式将消息发送到Topic。

  

- Shared：默认情况下，多个生成者可以将消息发送到同一个Topic。
    

  

- Exclusive：在这种模式下，只有一个生产者可以将消息发送到Topic ，当其他生产者尝试发送消息到这个Topic时，会发生错误。只有独占Topic的生产者发生宕机时（Network Partition）该生产者会被驱逐，新的生产者才能产生并向Topic发送消息。
    

  

- WaitForExclusive：在这种模式下，只有一个生产者可以将消息发送到Topic。当已有生成者和Topic建立连接时，其他生产者的创建会被挂起而不会产生错误。如果想要采用领导者选举机制来选择消费者的话，可以采用这种模式。
    

  

  

## **（二）路由模式**

  

当将消息发送到分区Topic时，需要指定消息的路由模式，这决定了消息将会被发送到哪个分区Topic。**Pulsar有以下三种消息路由模式，RoundRobinPartition为默认路由模式**。

  

- RoundRobinPartition：如果消息没有指定key，为了达到最大吞吐量，生产者会以round-robin （轮询）方式将消息发布到所有分区。请注意round-robin并不是作用于每条单独的消息，而是作用于延迟处理的批次边界，以确保批处理有效。如果消息指定了key，分区生产者会根据key的hash值将该消息分配到对应的分区。这是默认的模式。
    

  

- SinglePartition：如果消息没有指定key，生产者将会随机选择一个分区，并发布所有消息到这个分区。如果消息指定了key，分区生产者会根据key的hash值将该消息分配到对应的分区。
    

  

- CustomPartition：自定义模式，用户可以创建自定义路由模式，通过实现MessageRouter接口实现自定义路由规则。
    

  

  

## **（三）批量处理**

  

Pulsar支持对消息进行批量处理。批量处理启用后，Producer会在一次请求中累积并发送一批消息。批量处理时的消息数量取决于最大消息数（单次批量处理请求可以发送的最大消息数）和最大发布延迟（单个请求的最大发布延迟时间）决定。开启批量处理后，积压的数量是批量处理的请求总数，而不是消息总数。

  

- ### **索引确认机制**
    

  

通常情况下，只有Consumer确认了批量请求中的所有消息，这个批量请求才会被认定为已处理。当这批消息没有全部被确认的情况下，发生故障时，会导致一些已确认的消息被重复确认。

  

为了避免Consumer重复消费已确认的消息，Pulsar从Pulsar 2.6.0开始采用批量索引确认机制。如果启用批量索引确认机制，Consumer将筛选出已被确认的批量索引，并将批量索引确认请求发送给Broker。Broker维护批量索引的确认状态并跟踪每批索引的确认状态，以避免向Consumer发送已确认的消息。当该批信息的所有索引都被确认后，该批信息将被删除。

  

默认情况下，索引确认机制处于关闭状态。开启索引确认机制将产生导致更多内存开销。

  

- ### **key-based batching**
    

  

key_shared模式下，Broker会根据消息的key来分发消息，但默认的批量处理模式，无法保证将所有的相同的key都打包到同一批中，而且Consumer在接收到批数据时，会默认把第一个消息的key当作这批消息的key，这会导致消息的错乱。因此key_shared模式下，不支持默认的批量处理。

  

key-based batching能够确保Producer在打包消息时，将相同key的消息打包到同一批中，从而consumer在消费的时候，也能够消费到指定key的批数据。

  

没有指定key的消息在打包成批后，这一批数据也是没有key的，Broker在分发这批消息时，会使用NON_KEY作为这批消息的key。

  

  

## **（四）消息分块**

  

启用分块后，如果消息大小超过允许发送的最大消息大小时，Producer会将原始消息分割成多个分块消息，并将分块消息与消息的元数据按顺序发送到Broker。

  

在Broker中，分块消息会和普通消息以相同的方式存储在Ledger中。唯一的区别是，Consumer需要缓存分块消息，并在接收到所有的分块消息后将其合并成真正的消息。如果Producer不能及时发布消息的所有分块，Consumer不能在消息的过期时间内接收到所有的分块，那么Consumer已接收到的分块消息就会过期。

  

Consumer会将分块的消息拼接在一起，并将它们放入接收器队列中。客户端从接收器队列中消费消息。当Consumer消费到原始的大消息并确认后，Consumer就会发送与该大消息关联的所有分块消息的确认。

  

- ### **处理一个producer和一个订阅consumer的分块消息**
    

  

如下图所示，当生产者向主题发送一批大的分块消息和普通的非分块消息时。假设生产者发送的消息为M1，M1有三个分块M1-C1，M1-C2和M1-C3。这个Broker在其管理的Ledger里面保存所有的三个块消息，然后以相同的顺序分发给消费者（独占/灾备模式）。消费者将在内存缓存所有的块消息，直到收到所有的消息块。将这些消息合并成为原始的消息M1，发送给处理进程。

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

- ### **多个生产者和一个生产者处理块消息**
    

  

当多个生产者发布块消息到单个主题，这个Broker在同一个Ledger里面保存来自不同生产者的所有块消息。如下所示，生产者1发布的消息M1，M1 由M1-C1，M1-C2和M1-C3三个块组成。生产者2发布的消息M2，M2由M2-C1，M2-C2和M2-C3三个块组成。这些特定消息的所有分块是顺序排列的，但是其在Ledger里面可能不是连续的。这种方式会给消费者带来一定的内存负担。因为消费者会为每个大消息在内存开辟一块缓冲区，以便将所有的块消息合并为原始的大消息。

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

  

**三、Pulsar消费者（Consumer）**

  

Consumer是通过订阅关系连接Topic，接收消息的程序。

  

Consumer向Broker发送flow permit request以获取消息。在 Consumer端有一个队列，用于接收从Broker推送来的消息。

  

## **（一）消息确认**

  

Pulsar提供两种确认模式：

  

- 累积确认：消费者只需要确认最后一条收到的消息，在此之前的消息，都不会被再次发送给消费者。
    

  

- 单条确认：消费者需要确认每条消息并发送ack给Broker。
    

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

如图，上方为累积确认模式，当消费者发送M12的确认消息给Broker后，Broker会把M12之前的消息和M12一样都标记为已确认。下方为单条确认模式，当消费者发送M7的确认消息给Broker后，Broker会把M7这条消息标记为已确认。当消费者发送M12的确认消息给Broker后，Broker会把M12这条消息标记为已确认。

  

需要注意的是，订阅模式中的shared模式是不支持累积确认的。因为该订阅模式下的每个消费者都能消费数据，无法保证单个消费者的消费消息的时序和顺序。

  

- ### **AcknowledgmentsGroupingTracker**
    

  

消息的单条确认和累积确认并不是直接发送确认请求给Broker，而是把请求转交给AcknowledgmentsGroupingTracker处理。

  

为了保证消息确认的性能，并避免Broker接收到非常高并发的ack请求，Tracker默认支持批量确认，即使是单条消息的确认，也会先进入队列，然后再一批发往Broker。在创建consumer的时候，可以设置acknowledgementGroupTimeMicros，默认情况下，每100ms或者堆积超过1000时，AcknowledgmentsGroupingTracker会发送一批确认请求。如果设置为0，则每次确认消息后，Consumer都会立即发送确认请求。

  

  

## **（二）取消确认**

  

当Consumer无法处理一条消息并想重新消费时，Consumer可以发送一个取消确认的消息给Broker，Broker会重新将这条消息发送给Consumer。

  

如果启用了批量处理，那这一批中的所有消息都会重新发送给消费者。

  

消息取消确认也有单条取消模式和累积取消模式，取决于消费者使用的订阅模式。

  

在Exclusive模式和Failover订阅模式中，消费者仅仅只能对收到的最后一条消息进行取消确认。

  

在Shared和Key_Shared的订阅类型中，消费者可以单独否定确认消息。

  

如果启用了批量处理，那这一批中的所有消息都会重新发送给消费者。

  

- ### **NegativeAcksTracker**
    

  

取消确认和其他消息确认一样，不会立即请求Broker，而是把请求转交NegativeAcksTracker进行处理。Tracker中记录着每条消息以及需要延迟的时间。Tracker默认是33ms左右一个时间刻度进行检查，默认延迟时间是1分钟，抽取出已经到期的消息并触发重新投递。Tracker存在的意义是为了合并请求。另外如果延迟时间还没到，消息会暂存在内存，如果业务侧有大量的消息需要延迟消费，还是建议使用reconsumeLater接口。NegativeAck唯一的好处是不需要每条消息都指定时间，可以全局设置延迟时间。

  

  

## **（三）redelivery backoff机制**

  

通常情况下可以使用取消确认来达到处理消息失败后重新处理消息的目的，但通过redelivery backoff可以更好的实现这种目的。可以通过指定消息重试的次数、消息重发的延迟来重新消费处理失败的消息。

  

  

## **（四）确认超时**

  

除了取消确认和redelivery backoff机制外，还可以通过开启自动重传递机制来处理未确认的消息。启用自动重传递后，client会在ackTimeout时间内跟踪未确认的消息，并在消息确认超时后自动向代理重新发送未确认的消息请求。

  

- 如果开启了批量处理，那这批消息都会重新发送给Consumer。
    

  

- 与确认超时相比，取消确认会更合适。因为取消确认能更精确地控制单个消息的再交付，并避免在消息处理时引起的超过确认超时而导致无效的再重传。
    

  

  

## **（五****）消息预拉取**

  

Consumer客户端SDK会默认预先拉取消息到Consumer本地，Broker侧会把这些已经推送到Consumer本地的消息记录为pendingAck，这些消息既不会再投递给别的消费者，也不会ack超时，除非当前Consumer被关闭，消息才会被重新投递。Broker侧有一个RedeliveryTracker接口，这个Tracker会记录消息到底被重新投递了多少次，每条消息推送给消费者时，会先从Tracker的哈希表中查询一下重投递的次数，和消息一并推送给消费者。

  

  

## **（六）未确认的消息处理**

  

如果消息被消费者消费后一直没有确认怎么办？

  

unAckedMessageTracker中维护了一个时间轮，时间轮的刻度根据ackTimeout、tickDurationInMs这两个参数生成，每个刻度时间=ackTimeout/tickDurationInMs。新追踪的消息会放入最后一个刻度，每次调度都会移除队列头第一个刻度，并新增一个刻度放入队列尾，保证刻度总数不变。每次调度，队列头刻度里的消息将会被清理，unAckedMessageTracker会自动把这些消息做重投递。

  

重投递就是客户端发送一个redeliverUnacknowledgedMessages命令给Broker。每一条推送给消费者但是未ack的消息，在Broker侧都会有一个集合来记录（pengdingAck），这是用来避免重复投递的。触发重投递后，Broker会把对应的消息从这个集合里移除，然后这些消息就可以再次被消费了。

  

  

**四、Pulsar服务端** 

  

Broker是Pulsar的一个无状态组件，主要负责运行以下两个组件：

  

- http服务：提供为生产者和消费者管理任务和Topic查找的REST API。Producer通过连接到Broker来发送消息，Consumer通过连接到Broker来接收消息。
    

  

- 调度器：提供异步http服务，用于二进制数据的传输。
    

  

## **（一）消息确认与留存**

  

Pulsar Broker会默认删除已经被所有Consumer确认的消息，并以backlog的方式持久化存储所有未被确认的内消息。

  

Pulsar的message retention(消息留存) 和message expiry (消息过期)这两个特性可以调整Broker的默认设置。

  

- Message retention: 保留Consumer已确认的消息。
    

  

通过留存规则的设定，可以保证已经被确认且符合留存规则的消息持久地保存在Pulsar中，而没有被留存规则覆盖、已经被确认的消息会被删除。

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

- Message expire（消息过期）：设置未确认消息的存活时长（TTL）。
    

  

通过设置消息的TTL，有些即使还没有被确认，但已经超过TTL的消息，也会被删除。

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

  

## **（二）消息去重**

  

实现消息去重的一种方式是确保消息仅生成一次，即生产者幂等。这种方式的缺点是把消息去重的工作交由应用去做。

  

在Pulsar中，Broker支持配置开启消息去重，用户不需要为了消息去重去调整Producer的代码。启用消息去重后，即使一条消息被多次发送到Topic上，这条消息也只会被持久化到磁盘一次。

  

如下图，未开启消息去重时，Producer发送消息1到Topic后，Broker会把消息1持久化到BookKeeper，当Producer又发送消息1时，Broker会把消息1再一次持久化到BookKeeper。开启消息去重后，当Producer再次发送消息1时，Broker不会把消息1再一次持久化到磁盘。

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

- ### **去重原理**
    

  

Producer对每一个发送的消息，都会采用递增的方式生成一个唯一的sequenceID，这个消息会放在message的元数据中传递给Broker。同时，Broker也会维护一个PendingMessage队列，当Broker返回发送成功ack后，Producer会将PendingMessage队列中的对于的sequence id删除，表示Producer任务这个消息生产成功。Broker会记录针对每个 Producer接收到的最大Sequence ID和已经处理完的最大Sequence ID。

  

当Broker开启消息去重后，Broker会对每个消息请求进行是否去重的判断。收到的最新的Sequence ID是否大于Broker端记录的两个维度的最大Sequence ID，如果大于则不重复，如果小于或等于则消息重复。消息重复时，Broker端会直接返回ack，不会继续走后续的存储处理流程。

  

  

## **（三）消息延迟传递**

  

延时消息功能允许Consumer能够在消息发送到Topic后过一段时间才能消费到这条消息。在这种机制中，消息在发布到Broker后，会被存储在BookKeeper中，当到消息特定的延迟时间时，消息就会传递给Consumer。

  

下图为消息延迟传递的机制。Broker在存储延迟消息的时候不会进行特殊的处理。当Consumer消费消息的时候，如果这条消息设置了延迟时间，则会把这条消息加入DelayedDeliveryTracker中，当到了指定的发送时间时，DelayedDeliveryTracker才会把这条消息推送给消费者。

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

###   

- ### **延迟投递原理**
    

  

在Pulsar中，可以通过两种方式实现延迟投递。分别为deliverAfter和deliverAt。

  

deliverAfter可以指定具体的延迟时间戳，deliverAt可以指定消息在多长时间后消费。两种方式本质时一样的，deliverAt方式下，客户端会计算出具体的延迟时间戳发送给Broker。

  

DelayedDeliveryTracker会记录所有需要延迟投递的消息的index。index由Timestamp、Ledger ID、Entry ID三部分组成，其中Ledger ID和Entry ID用于定位该消息，Timestamp除了记录需要投递的时间，还用于延迟优先级队列排序。DelayedDeliveryTracker会根据延迟时间对消息进行排序，延迟时间最短的放在前面。当Consumer在消费时，如果有到期的消息需要消费，则根据DelayedDeliveryTracker index的Ledger ID、Entry ID找到对应的消息进行消费。  

  

如下图，Producer依次投递m1、m2、m3、m4、m5这五条消息，m2没有设置延迟时间，所以会被Consumer直接消费。m1、m3、m4、m5在DelayedDeliveryTracker会根据延迟时间进行排序，并在到达延迟时间时，依次被Consumer进行消费。

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

  

## **（四）Bundle**

  

我们知道，Topic分区会散落在不同的Broker中，那Topic分区和Broker的关系是如何维护的呢？当某个Broker负载过高时，Pulsar怎么处理呢？

  

Topic分区与Broker的关联是通过Bundle机制进行管理的。

  

每个namespace存在一个Bundle列表，在namesapce创建时可以指定Bundle的数量。Bundle其实是一个分片机制，每个Bundle拥有 namespace 整个hash范围的一部分。每个Topic (分区) 通过hash运算落到相应的Bundle区间，进而找到当前区间关联的Broker。每个Bundle绑定唯一的一个Broker，但一个Broker可以有多个Bundle。

  

如下图，T1、T2这两个Topic的hash结果落在[0x0000000L——0x4000000L]中，这个hash范围的Bundle对应Broker2，Broker2会对T1、T2进行处理。

  

同理，T4的hash结果落在[0x4000000L——0x8000000L]中，这个hash范围的Bundle对应Broker1，Broker1会对T4进行处理；

  

T5的hash结果落在[0x8000000L——0xC000000L]中，这个hash范围的Bundle对应Broker3，Broker3会对T5进行处理；

  

T3的hash结果落在[0xC000000L——0x0000000L]中，这个hash范围的Bundle对应Broker3，Broker3会对T3进行处理。

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

Bundle可以根据绑定的Broker的负载进行动态的调整、绑定。当Bundle绑定的Broker的Topic数过多、负载过高时，都会触发Bundle拆分，将原有的Bundle拆分成2个Bundle，并将其中一个Bundle重新分配给不同的Broker，以降低原Broker的Topic数或负载。

  

  

**五、Pulsar存储层（Bookkeeper）**

  

BookKeeper是Pulsar的存储组件。

  

对于Pulsar的每个Topic（分区），其数据并不会固定的分配在某个 Bookie上，具体的逻辑实现我们在Bundle一节已经讨论过，而Topic的物理存储，实际上是通过BookKeeper组件来实现的。

  

## **（一）分片存储**

  

概念：

  

- Bookie：BookKeeper的一部分，处理需要持久化的数据。
    

  

- Ledger：BookKeeper的存储逻辑单元，可用于追加写数据。
    

  

- Entry：写入BookKeeper的数据实体。当批量生产时，Entry为多条消息，当非批量生产时，Entry为单条数据。
    

  

Pulsar在物理上采用分片存储的模式，存储粒度比分区更细化、存储负载更均衡。如图，一个分区Topic-Partition2的数据由多个分片组成。每个分片作为BookKeeper中的一个Ledger，均匀的分布并存储在BookKeeper的多个Bookie节点中。

  

基于分配存储的机制，使得Bookie的扩容可以即时完成，无需任何数据复制或者迁移。当Bookie扩容时，Broker可以立刻发现并感知新的Bookie，并尝试将新的分片Segment写入新增加的Bookie中。

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

如上图，在Broker中，消息以Entry的形式追加的形式写入Ledger中，每个Topic分区都有多个非连续ID的Ledger，Topic分区的Ledger同一时刻只有一个处于可写状态。

  

Topic分区在存储消息时，会先找到当前使用的Ledger，生成Entry ID（每个Entry ID在同一个Ledger内是递增的）。当Ledger的长度或Entry个数超过阈值时，新消息会存储到新Ledger中。每个messageID由[Ledger ID，Entry ID，Partition编号，batch-index]组成。（Partition：消息所属的Topic分区，batch-index：是否为批量消息）

  

一个Ledger会根据Topic指定的副本数量存储到多个Bookie中。一个Bookie可以存放多个不连续的Ledger。

  

  

## **（二）读写数据的流程**

   

- Journals：Journals文件包含BookKeeper的事务日志信息。在对Ledger更新之前，Bookie会保证更新的事务信息已经写入Journals。当Bookie启动或者旧的Journals大小达到阈值时，就会创建一个新的Journals 。
    

  

- Entry Logs：Entry Logs管理从Bookie收到的Entry数据。来自不同Ledger的Entry会按顺序聚合并写入Entry Logs，这些Entry在Entry Logs中的偏移量会作为指针保存在Ledger Cache中，以便快速查找。当Bookie启动或者旧的Entry Logs大小达到阈值时，就会创建一个新的Entry Logs。当旧的Entry Logs没有与任何活跃的Ledger关联时，就会被垃圾回收器删除。
    

  

- Index Files：每个Ledger都会创建一个Index file，它包括一个头和几个固定长度的Index page，这些Index page记录存储在Entry Logs中的Entry的偏移量。由于更新索引文件会引入随机的磁盘I/O，所以索引文件由后台运行的同步线程延迟更新。这确保了更新的快速性能。在索引页持久化到磁盘之前，将它们聚集在Ledger Cache中以方便查找。
    

  

- Ledger Cache：Ledger Cache存放在内存池中，这样可以更高效地管理磁盘头调度。
    

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

- ### **消息的写入**
    

  

将Entry追加写入Ledger中。

  

将这次Entry的更新操作写入Journal日志中，当由多个数据写入时，可以批量提交，将数据刷到Journal磁盘中。

  

将Entry数据写入写缓存中。

  

返回写入成功响应。

  

到这里，消息写入的同步流程已经完成。

  

3-A. 内存中的Entry数据会根据Ledger和写入Ledger的时间顺序进行排序，批量写入Entry Log中。

  

3-B. Entry在Entry log中的偏移量以Index Page的方式写入Ledger Cache中，即iIdex Files。

  

Entry Log和Ledger Cache中的Index File会Flush到磁盘中。

  

- ### **消息的读取**
    

  

A.先从写缓存中以尾部读的方式读取。

  

B.如果写缓存未命中，则从读缓存中读取。

  

C.如果读缓存未命中，则从磁盘中读取。磁盘读取有三步：

  

- C-1.读取Index Disk，获取Entry的偏移量。
    
      
    
- C-2.根据Entry的偏移量，在Entry Disk中快速找到Entry。
    
      
    
- C-3.将Entry数据写入读缓存中。
    

  

**参考文献**

1.Pulsar官方文档

2.BookKeeper官方文档

3.Apache Pulsar 技术系列-客户端消息确认

4.Apache Pulsar 技术系列-Message deduplication这里的去重与你想的可能不一样

5.Apache Pulsar 技术系列-Pulsar延迟消息投递解析

6.Apache 系列—Pulsar核心特性解析

-------------  END  -------------

**扫描下方二维码，****获取更多****架****构笔记资料****！**

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

点个在看你最好看

阅读 6707

​