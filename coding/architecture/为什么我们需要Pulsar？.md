# 

原创 林琳 博文视点Broadview

_2021年12月02日 19:34_

👆点击“博文视点Broadview”，获取更多书讯

![图片](https://mmbiz.qpic.cn/mmbiz_png/PW0wIHxgg3nr1VNxfeqxVOw2nPJHVH4xeZibzPY5F4ibOuOZLMsUMrzIibGB6KMw7EurSKv6DkrtLzuhYdBa30A9Q/640?wx_fmt=png&wxfrom=13&tp=wxpic)

随着互联网的高速发展，用户规模与业务并发量开始急剧增加，海量的请求需要接收和存储，业务需要中间件来实现削峰填谷；业务也在不断发展，企业内部的系统数量也在不断地增长，不同语言开发出来的系统需要统一的事件驱动；大数据、AI已经成为很多业务中不可或缺的技术，它们都需要统一的数据源。越来越多的场景离不开消息队列，**稍具规模的业务，消息队列都是“标配”**。

有的人可能会问，现在消息队列已经非常成熟了，我们可以使用Kafka、RabbitMQ等满足日常的业务需求，**为什么还会出现Pulsar这个消息队列，并且迅速发展呢？**

理由有很多，由于篇幅问题，我们不能一一列举，下面列出几个日常使用中比较关注的方面。我们会发现，Pulsar不仅仅是一个消息队列。

**1** **云原生环境的适配**

基于Kubernetes的整个生态，已经成为事实上的云原生标准，大量的服务都开始与这个标准适配。借助于Kubernetes的动态伸缩能力，企业能够更好地管理计算资源，降低成本、提升效率。

对于无状态的服务，移植进云原生环境是很容易的。而传统的消息队列，在设计之初并没有考虑云原生的情况，大多是有状态的。因此，传统消息队列的运维成本相对较高，在适配云原生环境的问题上，需要研发人员投入一定的时间。

而Pulsar是**计算与存储分离的架构**，天然适配云原生环境。适配云原生环境是一个比较虚的概念，其实包含了很多的方面的内容。例如，利用容器化环境动态扩/缩容量、高可用，等等。Pulsar的Broker是无状态的，并且不存储任何数据。而负责存储的BookKeeper，也只是负责存储数据，数据的组织方式与规则等有状态的元信息都存储在ZooKeeper中。因此，Bookie也可以水平扩容，新的数据能存入新的Bookie。至于高可用等其他能力，我们在后续的章节中再详细讲解。

**2** **支持多租户和海量Topic**

Pulsar**天然支持多租户**，每个租户下还支持多Namespace（命名空间），非常适合做共享大集群，方便维护。

此外，Pulsar**天然支持租户之间资源的逻辑隔离**，只要用户的运营管控后台和监控足够强大，便可以做到动态隔离大流量租户，防止互相干扰，还能实现大集群资源的充分利用。另外，单集群的Pulsar现在已经支持几十万个Topic，这部分能力还在持续优化中，后续支持百万级别的Topic会很轻松。

**3** **可靠消息与性能**

现有的消息队列让很多人都形成了一个惯性思维，即如果一个消息队列的消息是可靠的，那么它的性能肯定很差，因为同步刷盘、数据同步等会消耗时间。

而Pulsar**基本做到了可靠消息与性能的平衡**。即使在很高吞吐量的场景下，也能保证消息的可靠性，还能保证单点的性能。由于受到每条消息大小的影响，用QPS来计算性能可能不太合适，用每秒的流量来计算性能可能更准确。

在保证消息可靠的前提下，把单台机器的网卡带宽用满还是很容易的，最终的性能瓶颈要么在硬盘，要么在网卡。

Pulsar的性能是我们平常使用的一些消息队列无法比拟的。这其实得益于Pulsar的Quorum机制与条带化写入策略。

**4** **低延迟**

很多业务对消息延迟有很高的要求，现有的一些队列，要么延迟很小但吞吐量低，要么延迟很大但吞吐量高，而Pulsar则是一个两者可以兼得的消息队列。

在笔者编写《深入解析Apache Pulsar》一书的时候，Pulsar官方做了一个Pulsar与Kafka全方位对比的测试，对比文章中显示，Kafka在100个Partition时，99%的延迟小于18.75ms，在5000个Partition的时候消息延迟开始大幅上升，99%的延迟小于79236ms。Pulsar从100、5000一直到10000个Partition，在单个Broker同步返回算成功的情况下，延迟一直保持在10ms以下；在两个Broker同步返回算成功的场景下，延迟一直保持在20ms以下。可见，Pulsar**在低延迟方面的表现还是非常优秀的**。

**5** **高可用的分布式消息队列**

Pulsar是**一个真正意义上的分布式消息队列，自带了各种容灾方案**。我们可以根据不同业务的RTO（Recovery Time Objective）、RPO（Recovery Point Objective）来决定使用哪一种。

首先，异步的跨地域复制（Geo Replication）可以实现两个集群之间的主备复制、互备复制等。这种方式在一个集群完全宕机的情况下，另外一个集群可以继续提供服务，但数据有一定的时间间隔的损失，这取决于网络延迟和异步复制延迟。另外一种是强一致的复制方式，这种方式在官方的文档中没有写出来，但我们可以通过BookKeeper的跨机架配置，或者配置Quorum的全ACK方式来实现。这种开箱即用的方式，让我们在架构设计上省去了很多的工夫。

**6** **轻量函数式计算（Pulsar Function）**

我们**可以使用函数创建复杂的处理逻辑，无须部署单独的处理系统**。

函数是Pulsar消息传递系统的计算基础结构，我们来看一个常见的使用场景：把Topic-1中的数据读出来，经过中间处理，然后把数据存入Topic-2，通过上传Java、Go、Python代码，用户可以自定义中间的处理过程。在这个场景的基础上衍生，读取的数据并不一定是Pulsar中的Topic，可以是外部其他系统；写入的也不一定是Pulsar的Topic，可以是外部的系统，从而实现Sink、Source语义。

**7** **流批一体**

随着业务的不断发展，流计算和批处理越来越常见，通常我们需要分别维护一套流计算平台和批处理平台以满足不断发展的业务需求。而Pulsar**可以同时支持两种计算方式，只需要维护一套中间件即可实现流批一体**。

完整的历史数据可以让我们做批计算，数据在某段时间内可以变为流。流和批本来就是硬币的两面，随着业务的不断发展，单纯使用流计算或者批处理都无法满足业务的需求。Pulsar使用Segment分片存储可以很方便地支持流计算，使用分层存储又可以很好地支持批处理。我们再也不用把数据从不同的存储中迁移、转换了，Pulsar天然支持流批融合。再基于函数的能力，Pulsar可以很容易和其他流计算和批计算平台对接，成为它们的数据源或者消息存储节点。

**8** **多协议、功能丰富**

Pulsar是一个融合的消息系统。

**除了自己的通信协议，还支持其他消息队列的协议**，比如支持Kafka协议的KOP、AMQP、MQTT协议等。这让其他消息队列迁移到Pulsar的成本大大降低，方便内部统一消息队列。

对比其他消息队列，Pulsar的**功能非常丰富**，比如延迟队列、死信队列、顺序消息、主题压缩（相同Key消息只保留最新一条）、多租户、认证授权、分层存储（冷热分离）、跨地域复制等。基本上日常业务需要的能力，Pulsar都能满足。这就让一套消息队列能支持众多的业务，不会因为无法提供某些业务能力而又要维护另外一类消息队列，降低了内部团队的运维成本。

现在很多公司的业务场景非常复杂，Kafka有很多功能的缺失，如果要使用死信队列，则可能还要部署一套RabbitMQ，最终可能市面上所有的消息队列都会维护一套，导致消息队列的维护成本急剧上升。如果使用Pulsar，那么只需要维护一套集群就能解决所有的业务问题。现有的业务也可以借助Pulsar对多协议的支持，无缝切换到Pulsar。

了解了Pulsar的一些优势，下面讲解Pulsar架构演进的历程。Pulsar在国内首次大规模落地是在智联招聘。Pulsar演进过程中的能力里程碑如下。

**1** **Pulsar的诞生**

Pulsar诞生于2012年，最初的目的是在Yahoo内部整合其他消息系统，构建统一逻辑、支撑大集群和跨区域的消息平台。当时的其他消息系统（包括 Kafka）都不能满足Yahoo的业务需求，比如大集群多租户、稳定可靠的I/O服务质量、百万级Topic、跨地域复制等，因此Pulsar应运而生。

**2** **可扩展存储**

基于Apache BookKeeper，Pulsar实现了数据流的可扩展存储。BookKeeper是一个分布式天然可扩展的日志系统，所有的数据以追加的方式写入，已经写入的数据不能再修改。基于BookKeeper，Pulsar的消息以日志的方式追加写入分布式节点，读取则从reader定义的起始偏移量连续读取。

**3** **计算与存储分离**

Pulsar正式分离出Broker与Bookie两层架构，Broker为无状态服务，用于发布和消费消息，而BookKeeper专注于存储。此时的Pulsar正式具备了云原生能力，在这种架构下，可以实现容量的动态伸缩。

**4** **统一消息模型与API**

在数据业务场景中，数据的表现形式通常可以分为两类：队列和流。队列通常用于构建核心业务应用程序服务，流通常用于构建实时数据服务，如数据管道。Pulsar通过定义Topic消息模型，实现了队列和流语义的统一。Pulsar Topic成为数据的来源。消息只需要在Topic上存储一次，就可以以不同的订阅方式消费数据。

**5** **支持Schema**

通过建立内部的Schema Registry，让消息的序列化与反序列化变得更加简单且规范。消息序列化方式从生产方与消费方的人为约定变为Schema维度的强约束。

**6** **Function与Pulsar IO**

基于Function与IO，Pulsar奠定了流式计算、Serverless化的基础。

**7** **实现无限存储能力**

如果Pulsar存储中的数据量不断增加，那么磁盘空间最终会不足。因此Pulsar实现了分层存储（Offloader），把老旧、冷数据自动迁移到外部廉价存储中。但整个过程对用户透明，用户还可以像直接在Pulsar中一样读取这些数据。

**8** **支持多协议扩展**

Pulsar支持多协议扩展能力，进而演化出了KOP、ROP、AOP等多协议插件。

**9** **Exactly-Once语义支持**

随着Pulsar 2.8.0的发布，基于最新的事务消息能力，Pulsar实现了Exactly-Once（刚好一次）的语义，这对于Function和Pulsar I/O来说，计算准确性得到了保证。

Pulsar服务端的基本结构可以分为三层，分别是代理层、Broker层和Bookie层。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

Pulsar的总体架构

**其中，代理层并不是必需的。**

如果没有代理层，那么生产者/消费者会直接与每个要生产/消费的Broker建立一个连接。但很多时候，我们并不方便直接暴露Broker的IP地址，这时可以使用一个代理层，只暴露代理层的IP地址，由代理层做相应的请求转发。

如果启用了代理层，那么客户端做Broker的服务发现，也是通过代理完成的。客户端不直接与Broker建立连接，而是与Proxy先建立连接，后续的请求也是先经过Proxy，然后由Proxy转发给对应的Broker，适用于对网络、安全等有要求的场景。

**Broker和一般MQ不同**，数据并不直接存储在Broker中，而是保存在最下面的BookKeeper集群中。由于Broker是无状态的，所以它可以很方便地在容器环境中快速扩/缩容。

Broker主要负责整个Pulsar的业务逻辑，BookKeeper只负责数据的存储。如果没有使用代理层，那么客户端会先连接到任意一个Broker拉取元数据，然后直接与Broker建立连接。

Broker除了可以处理常见的数据流请求，比如发送消息、接收消息，还提供了管理流相关的接口。这些接口分为租户（Tenant）级别、命名空间（Namespace）级别、主题（Topic）级别等，比如创建租户、删除命名空间、查询主题列表等。管理流的接口都基于RESTful的HTTP，数据流的接口则基于Pulsar自定义的二进制协议，使用ProtoBuf作为序列化工具。

\*\*BookKeeper是一个可扩展、容错、低延迟、只可追加数据的存储服务。\*\*Pulsar使用它来存储数据，它不包含任何业务逻辑，我们可以把它看作数据库。整个Pulsar的存储逻辑都由BookKeeper负责，它拥有动态伸缩、自动容错恢复、读/写分离等能力，我们会在后面的存储章节中重点讲解BookKeeper。

Broker和BookKeeper都会用到ZooKeeper，Broker主要用它存储元数据、选主，使用它的分布式锁，BookKeeper的使用场景也类似。Broker除了使用一个本地的ZooKeeper，还可能用到一个Global ZooKeeper，通常是在多个集群之间需要相互通信的场景中，比如跨地域复制等，多个集群之间的Broker使用Global ZooKeeper来共享相关的元数据配置。

本文摘自《深入解析Apache Pulsar》一书，欢迎阅读此书了解更多关于Apache Pulsar的内容！

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E "深入解析Apache Pulsar.jpg")

**▊**\*\*《\*\***深入解析Apache Pulsar》**

林琳 著

- 详解ApachePulsar源码，深入分析背后原理与实现，实战Pulsar线上问题处理

本书由浅入深地讲解了Apache Pulsar中各个组件的使用方式及内部实现原理，通过阅读本书，读者可以快速、轻松地了解Apache Pulsar内部的运行机制。

第1章介绍Apache Pulsar的背景，以及如何快速部署一个Apache Pulsar服务。第2章介绍Apache Pulsar客户端的实现机制与原理，包括生产者、消费者、管理流客户端等。第3章介绍Apache Pulsar中最重要的逻辑组件—Broker，读者通过这部分内容可以了解Broker所有的特性。除了最基础的收发消息，Apache Pulsar还能进行轻量级的函数计算、数据流转。第4章详细介绍Apache Pulsar的Function和Pulsar IO （Connector）。第5章介绍Apache Pulsar的存储层—BookKeeper，通过对本章的学习，读者可以了解Apache Pulsar的数据存储模型及流程实现。第6章介绍线上实战的一些经验，包括高可用、扩/缩容、资源隔离等。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E "pulsar二维码 (2).png")

（京东满100减50，快快扫码抢购吧！）

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

如果喜欢本文

欢迎 **在看**丨**留言**丨**分享至朋友圈** 三连

**热文推荐**

- [Serverless：微服务架构的终极模式](http://mp.weixin.qq.com/s?__biz=MjM5NTk0NjMwOQ==&mid=2651143566&idx=1&sn=5e5132acc3c5c9ef8a7df9fbdadf1790&chksm=bd0157a48a76deb2e21727881b11b01aed26a5d4e38e41287f06f0cf1f580560e67e2b24090e&scene=21#wechat_redirect)

- [详解阿里开源分布式事务框架Seata](http://mp.weixin.qq.com/s?__biz=MjM5NTk0NjMwOQ==&mid=2651143524&idx=1&sn=2a80bad1d101849ce1195e665f3309d8&chksm=bd01574e8a76de58f3b164d35296a80b7dff1c09182ef6152698443d1c45bb67a72820662705&scene=21#wechat_redirect)

- [图论算法：稳定婚姻问题](http://mp.weixin.qq.com/s?__biz=MjM5NTk0NjMwOQ==&mid=2651143435&idx=1&sn=fc5603b9feb4c3e959ae282eeaefa430&chksm=bd0157218a76de37fffa48ddfa25582634f3f1f49327c84c1979427f366f8c3b10dc0974c9a7&scene=21#wechat_redirect)

- [60万字诚意续作《腾讯游戏开发精粹Ⅱ》正式发布](http://mp.weixin.qq.com/s?__biz=MjM5NTk0NjMwOQ==&mid=2651143374&idx=1&sn=d1479e02c2d85ee8a41016bd14fb33a1&chksm=bd0156e48a76dff264d5d9fbc0dda278aefe21454346db93ec49ffeb42b0504aa359a01e27ce&scene=21#wechat_redirect)

______________________________________________________________________

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

▼点击阅读原文，查看本书详情~

阅读原文

阅读 1155

​

写留言

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/PW0wIHxgg3lICy34AaLxSSBtOGFrm6eovZRP96ic72ibb6aTQiaYEeIPd1Jl7r7wia7Bh3v8HOmOQgCQUMaTicfROgQ/300?wx_fmt=png&wxfrom=18)

博文视点Broadview

813

写留言

写留言

**留言**

暂无留言
