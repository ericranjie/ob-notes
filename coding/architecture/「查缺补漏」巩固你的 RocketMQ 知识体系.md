# 

悟空聊架构

 _2021年12月06日 18:20_

以下文章来源于是Kerwin啊 ，作者柯小贤

[

![](http://wx.qlogo.cn/mmhead/Q3auHgzwzM7canx0dj479ZOt5If239T5XOX2GUWWiaMyJXQN5icb7Srw/0)

**是Kerwin啊**.

一切都是有可能的，甚至那些不可能的也是。

](https://mp.weixin.qq.com/s?__biz=MzAwMjI0ODk0NA==&mid=2451961588&idx=1&sn=2fb0de461f7c3cfb9ac71663557fa9f2&chksm=8d1c0f6bba6b867d413a6775b8b7a025e3af418d2ffb5e4eb536825feb953b2c499a72187b3c&mpshare=1&scene=24&srcid=1206b4MDQTJ3Ury1l1Cvan2n&sharer_sharetime=1638786522938&sharer_shareid=5fb9813bfe9ffc983435bfc8d8c5e9ca&key=daf9bdc5abc4e8d095029538793dcc599f84b8dac6b261631da122a81d4c1a01797a43d02b544b70a32fdb230c1c27903e5f27f04af7608c86daf02cb82204f35bff7cf78c61a7f66bff2f9387839c549fd614550ded4ed38ff264ebe37c263fa36f51e540b48d94ccc28a444f6b493890a0de62b8366b5f2fa9bbd7e84c85b7&ascene=0&uin=MTEwNTU1MjgwMw%3D%3D&devicetype=Windows+11+x64&version=63090b19&lang=zh_CN&countrycode=CN&exportkey=n_ChQIAhIQ4NSptsWNZEKbZTQuzob3JRLmAQIE97dBBAEAAAAAAKfrDnk2vO4AAAAOpnltbLcz9gKNyK89dVj09O0ivYAVQP81lzD9fD3%2BshWUxAn5QqDvBwcZ606A2edXGrji2V%2F9BGGywejV7u5jIDkfxkvdtz4wTQBmmguZZ4OjVB1VEzIM%2FD7WIZABez8XzPQnMctgASzca69pUB%2FytvP7Kmzq3l6OYqbi9t6JgkzXv0i9uMayyMAv1iIFNT%2FJA%2Fj2Sagv6Yl%2FloM6xi%2FPmKPru5uZPMv4KcFfTXlK9QqNiUL8hR0ooGUSVAu0mbNYdx3pPqgqCrrlkK8WkZwM&acctmode=0&pass_ticket=LATXYhlWK%2FI7Epx4Va%2FUak7PYduVxUn9pOqV5v%2FC4kZ%2Bbv8sYnIafOwNTc03t1zO&wx_header=1&fasttmpl_type=0&fasttmpl_fullversion=7351805-zh_CN-zip&fasttmpl_flag=1#)

## 大家好，我是悟空呀。

## 本期带来的内容是 RocketMQ 核心知识点。建议收藏起来慢慢看~

## Windows安装部署

```
下载地址：
```

选择‘Binary’进行下载

解压已下载工程

### 配置

> 新增系统变量 ROCKETMQ_HOME  ->  F:\RocketMQ\rocketmq-4.5.2
> 
> JAVA_HOME             ->  F:\Java_JDK\JDK1.8
> 
> Path 系统变量新增：Maven/bin目录
> 
> PS：RocketMQ 消息存储在C:\Users\Administrator\store store目录中  `文件占用较大，注意删除不必要的内容`

### 启动

> start mqnamesrv.cmd
> 
> start mqbroker.cmd -n 127.0.0.1:9876 autoCreateTopicEnable=true

### Rocket集成可视化监控插件

```
任意目录（拉取项目，随便哪里都行）git clone https://github.com/apache/rocketmq-externals.git
```

`Rocket可视化监控插件 增加Topic | 自动增加Topic（4.5.2版本）`

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

> 4.5.2 版本 支持自动创建Topic
> 
> 4.3.0 版本 必须通过监控程序配置Topic，否则执行程序报错，没有此路由

### SpringBoot集成 RocketMQ

```
<!--RocketMQ-->
```

## RocketMQ基本概念

  

### 概览

基于RocketMQ的分布式系统，一般可以分为四个集群：Name server、broker、producer、consumer

1. name server
    

- 提供轻量级的服务发现和路由服务；
    
- 每个节点都存放了全部的路由信息和对应的读写服务；
    
- 存储支持水平扩展
    

3. broker
    

- 提供满足TOPIC和QUEUE机制的消息存储服务；
    
- 有推和拉两种模式；
    
- 通过2或3拷贝实现高可用；
    
- 提供上亿消息的堆积能力；
    
- 提供故障恢复、统计功能和告警功能；
    

5. producer
    

- 支持分布式部署，通过负载平衡模块给broker发消息
    
- 支持快速失败
    
- 低延迟
    

7. consumer
    

1. 支持推和拉两种模式
    
2. 支持集群消费和广播消费
    

  

### 核心模块

  

#### Name Server

> 提供Broker管理；Routing管理（路由管理）

NameServer，很多时候称为命名发现服务，其在RocketMQ中起着中转承接的作用，是一个无状态的服务，多个NameServer之间不通信。任何Producer，Consumer，Broker与所有NameServer通信，向NameServer请求或者发送数据。而且都是单向的，Producer和Consumer请求数据，Broker发送数据。正是因为这种单向的通信，RocketMQ水平扩容变得很容易

- 提供轻量级的服务发现和路由服务；
    
- 每个节点都存放了全部的路由信息和对应的读写服务；
    
- 存储支持水平扩展
    

总结：相比于ZooKeeper提供的分布式锁，发布和订阅，数据一致性，选举等，在RocketMQ是不适用的，因此重写了一套更加轻量级的发现服务，主要用以存储 Broker相关信息以及当前Broker上的topic信息，路由信息等

#### Broker Server

> 提供Remoting Module、客户端管理、存储服务、HA服务(主从)、索引服务

- 提供满足TOPIC和QUEUE机制的消息存储服务；
    
- 有推和拉两种模式；
    
- 通过2或3拷贝实现高可用；
    
- 提供上亿消息的堆积能力；
    
- 提供故障恢复、统计功能和告警功能；
    

#### producer

- 支持分布式部署，通过负载平衡模块给broker发消息
    
- 支持快速失败
    
- 低延迟
    

#### consumer

- 支持推和拉两种模式
    
- 支持集群消费和广播消费
    

  

### 核心角色介绍

  

#### 生产者

生产者发送业务系统产生的消息给broker, RocketMQ提供了多种发送方式：同步的、异步的、单向的

  

#### 生产者组

具有相同角色的生产者被分到一组, 假如原始的生产者在事务后崩溃，broker会联系 同一生产者组中的不同生产者实例，继续提交或回滚事务

  

#### 消费者

一个消费者从broker拉取信息，并将信息返还给应用。为了我们应用的正确性，提供了两种消费者类型：

拉式消费者：拉式消费者从broker拉取消息，一旦一批消息被拉取，用户应用系统将发起消费过程。

推式消费者：推式消费者，从另一方面讲，囊括了消息的拉取、消费过程，并保持了内部的其他工作，留下了一个回调 接口给终端用户去实现，实现在消息到达时要执行的内容。

  

#### 消费者组

具有相同角色的消费者被组在一起，称为消费者组，它完成了负载均衡和容错的目标

一个消费组中的消费者实例必须有确定的相同的订阅topic

  

#### Topic（主题）

Topic是一个消息的目录，在这个目录中，生产者传送消息，消费者拉取消息，可以多个消费者订阅同一个topic，一个生产者也可以发送多个topic

PS：RocketMQ 基于发布订阅模式，发布订阅的核心即 Topic 主题

  

#### Message（消息）

消息是被传递的信息。一个消息必须有一个Topic，它可以理解为信件上的地址。一个消息也可以有一个可选的tag，和额外的key-value对。例如：你可以设置业务中的键到你的消息中，在broker服务中查找消息，以便在开发期间诊断问题

  

#### 消息队列

Topic被分割成一个或多个消息队列。队列分为3中角色：异步主、同步主、从。如果你不能容忍消息丢失，我们建议你部署同步主，并加一个从队列。如果你容忍丢失，但你希望队列总是可用，你可以部署异步主和从队列。如果你想最简单，你只需要一个异步主，不需要从队列。消息保存磁盘的方式也有两种，推荐使用的是异步保存，同步保存是昂贵的并会导致性能损失，如果你想要可靠性，我们推荐你使用同步主+从的方式。

  

#### Tag（标签）

标签，用另外一个词来说，就是子主题，为用户提供额外的灵活性。具有相同Topic的消息可以有不同的tag。

  

#### Broker（队列）

Broker是RocketMQ的一个主要组件，它接收生产者发送的消息，存储它们并准备处理消费者的拉取请求。它也存储消息相关的元数据， 包括消费组，消费成功的偏移量，主题、队列的信息。

  

#### 名称服务

名称服务主要提供路由信息。生产者/消费者客户端寻找topic，并找到通信的队列列表。

  

#### 消息的顺序

当`DefaultMQPushConsumer`被使用，你就要决定消费消息时，是顺序消费还是同时消费

- 顺序消费
    

顺序消费消息的意思是 消息将按照生产者发送到队列时的顺序被消费掉。如果你被强制要求使用全局的顺序，你要确保你的topic只有一个消息队列。

如果指定顺序消费，消息被同时消费的数量就是订阅这个topic的消费组的数量。

- 同时消费
    

当同时消费消息时，消息同时消费的最大数量取决于消费客户端指定的线程池的大小。

  

#### 最佳实践

##### **Producer最佳实践**

1. 一个应用尽可能用一个 Topic，消息子类型用 tags 来标识，tags 可以由应用自由设置。**只有发送消息设置了tags，消费方在订阅消息时，才可以利用 tags 在 broker 做消息过滤。**
    
2. 每个消息在业务层面的唯一标识码，要设置到 keys 字段，方便将来定位消息丢失问题。由于是哈希索引，请务必保证 key 尽可能唯一，这样可以避免潜在的哈希冲突。
    
    消息发送成功或者失败，要打印消息日志，务必要打印 sendresult 和 key 字段。
    
3. **对于消息不可丢失应用，务必要有消息重发机制。例如：消息发送失败，存储到数据库，能有定时程序尝试重发或者人工触发重发。**
    
4. 某些应用如果不关注消息是否发送成功，请直接使用sendOneWay方法发送消息。
    

##### **Consumer最佳实践**

1. **消费过程要做到幂等（即消费端去重）**
    
2. 尽量使用批量方式消费方式，可以很大程度上提高消费吞吐量。
    
3. 优化每条消息消费过程
    

## MQ核心问题

  

### 1.消息队列适合解决的问题

解决的核心问题主要是：异步、解耦、削峰

但是引入消息队列也会有很多额外的问题，比如系统复杂性会大大增加，同时需要解决重复下发，重复消费，消费顺序，消息丢失，重试机制等等问题，因此不能滥用，合适的场景用合适的技术

  

### 2.消息模型：主题和队列的区别

_**一、消息队列的演进**_

**1、初始阶段**

最初的消息队列，就是一个严格意义上的队列。队列是一种数据结构，先进先出，在消息入队出队过程中，保证这些消息严格有序。**早期的消息队列就是按照“队列”的数据结构设计的**。

队列模型：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

生产者（Producer）发消息就是入队操作，消费者（Consumer）收消息就是出队也就是删除操作，服务端存放消息的容器自然就称为“队列”。

- 如果有多个**生产者往同一个队列里面发送**消息，这个队列中可以消费到的消息，就是这些生产者生产的所有消息的合集。消息的顺序就是这些生产者**发送消息的自然顺序**。
    
- 如果有**多个消费者接收同一个队列**的消息，这些消费者之间实际上是**竞争的关系**，每个消费者只能收到队列中的一部分消息，也就是说任何**一条消息只能被其中的一个消费者收到**。
    

**2、发布 - 订阅模型阶段**

如果需要将**一份消息数据分发给多个消费者**，要求**每个消费者都能收到全量的消息**，例如，对于一份订单数据，风控系统、分析系统、支付系统等都需要接收消息。

这个时候，单个队列就满足不了需求，一个可行的解决方式是，为**每个消费者创建一个单独的队列，让生产者发送多份**。但是同样的一份消息数据被复制到多个队列中会**浪费资源**，更重要的是，生产者必须知道有多少个消费者。为每个消费者单独发送一份消息，这实际上**违背了消息队列“解耦”**这个设计初衷。

为了解决这个问题，演化出了另外一种消息模型：**发布 - 订阅模型**（Publish-Subscribe Pattern）

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

消息的发送方称为发布者（Publisher），消息的接收方称为订阅者（Subscriber），服务端存放消息的容器称为主题（Topic）。

- 发布者将消息发送到主题中，订阅者在接收消息之前需要先“订阅主题”。
    
- 每份订阅中，订阅者都可以接收到主题的所有消息。
    

**3、总结：**

- 在很长的一段时间，队列模式和发布 - 订阅模式是并存的。
    
- 有些消息队列同时支持这两种消息模型，比如 ActiveMQ。
    
- 对比这两种模型，生产者就是发布者，消费者就是订阅者，队列就是主题，并没有本质的区别。它们最大的区别是：**一份消息数据能不能被消费多次的问题**。
    
- 实际上，在这种发布 - 订阅模型中，如果只有一个订阅者，那它和队列模型就基本是一样的了。也就是说，发布 - 订阅模型在功能层面上是可以兼容队列模型的。
    

_**二、RabbitMQ 的消息模型**_

少数依然坚持**使用队列模型**的产品之一。

RabbitMQ 使用 **Exchange 模块解决多个消费者的问题**。Exchange 位于生产者和队列之间，生产者并不关心将消息发送给哪个队列，而是**将消息发送给 Exchange**，由 Exchange 上**配置的策略**来决定将消息投递到哪些队列中。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

- 同一份消息如果需要被多个消费者来消费，需要**配置 Exchange 将消息发送到多个队列**，每个队列中都**存放一份完整的消息数据**，可以为一个消费者提供消费服务。
    

_**三、RocketMQ 的消息模型**_

RocketMQ 使用的消息模型是标准的**发布 - 订阅模型**。在 RocketMQ 也有队列（Queue）这个概念。

**消息队列的消费机制：**

几乎所有的消息队列产品都使用一种非常朴素的“**请求 - 确认”机制**，确保消息不会在传递过程中由于网络或服务器故障丢失。

在生产端，生产者先将消息发送给服务端，也就是 Broker，服务端在收到消息并将消息写入主题或者队列中后，会**给生产者发送确认的响应**。如果生产者没有收到服务端的确认或者收到失败的响应，则会**重新发送消息**。

在消费端，消费者在收到消息并完成自己的消费业务逻辑（比如，将数据保存到数据库中）后，也会**给服务端发送消费成功的确认**，服务端只有收到消费确认后，才认为一条消息被成功消费，否则它会给消费者**重新发送这条消息**，直到收到对应的消费成功确认。

这个确认机制很好地**保证了消息传递过程中的可靠性**，但是，引入这个机制在消费端带来了一个问题：**为了确保消息的有序性，在某一条消息被成功消费之前，下一条消息是不能被消费的**，也就是说，每个主题在任意时刻，**至多只能有一个消费者实例在进行消费**，那就**没法通过水平扩展消费者的数量来提升消费端总体的消费性能**。

**为了解决这个问题，RocketMQ 在主题下面增加了队列的概念：**

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

- **每个主题包含多个队列**，通过多个队列来实现多实例并行生产和消费。需要注意的是，RocketMQ 只在队列上保证消息的有序性，主题层面是无法保证消息的严格顺序的。
    
- **生产者会往所有队列发消息**，但不是“同一条消息每个队列都发一次”，**每条消息只会往某个队列里面发送一次**。
    
- **一个消费组，每个队列上只能串行消费，多个队列加一起就是并行消费了**，并行度就是队列数量，队列数量越多并行度越大，所以水平扩展可以**提升消费性能。**
    
- **每队列每消费组维护一个消费位置（offset）**，记录这个消费组在这个队列上消费到哪儿了。
    
- 订阅者是通过消费组（Consumer Group）来体现的。**每个消费组都消费主题中一份完整的消息**，不同消费组之间消费进度彼此不受影响，也就是说，一条消息被 Consumer Group1 消费过，也会再给 Consumer Group2 消费。
    
- **消费组中包含多个消费者，同一个组内的消费者是竞争消费的关系**，每个消费者负责消费组内的一部分消息。如果一条消息被消费者 Consumer1 消费了，那同组的其他消费者就不会再收到这条消息。
    
- 由于消息需要被不同的组进行多次消费，所以消费完的消息并不会立即被删除，这就需要 RocketMQ **为每个消费组在每个队列上维护一个消费位置（Consumer Offset）**，这个位置之前的消息都被消费过，之后的消息都没有被消费过，每成功消费一条消息，消费位置就加一。我们在使用消息队列的时候**，丢消息的原因大多是由于消费位置处理不当导致的**。
    

_**四、Kafka 的消息模型**_

Kafka 的消息模型和 RocketMQ 是完全一样的，唯一的区别是，在 Kafka 中，队列这个概念的名称不一样，**Kafka 中对应的名称是“分区（Partition）”**，含义和功能是没有任何区别的。

_**五、总结**_

- 常用的消息队列中，**RabbitMQ 采用的是队列模型**，但是它一样可以实现发布 - 订阅的功能。**RocketMQ 和 Kafka 采用的是发布 - 订阅模型**，并且二者的消息模型是基本一致的。
    

  

### 3.消息丢失怎么办? 如何保证消息的可靠性传输?

**首先如何验证消息是否丢失？**

- 如果是 IT 基础设施比较完善的公司，一般都有分布式链路追踪系统，使用类似的追踪系统可以很方便地追踪每一条消息。
    
- 如果没有这样的追踪系统，我们可以利用消息队列的有序性来验证是否有消息丢失
    

即保证消息消费顺序的情况下，根据消息的序号，在消费段判断是否连续

解决方案：

**消息从生产到消费的过程中，可以划分三个阶段：**

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**1、生产阶段**

消息队列通过最常用的**请求确认机制**，来保证消息的可靠传递：当你代码调用发消息方法时，消息队列客户端会把消息发送到Broker，Broker收到消息后，会给客户端返回一个确认响应，表明消息已收到。客户端收到响应后，完成了一次正常消息的发送。

有些消息队列在长时间没收到发送确认响应后，会自动重试，如果重试失败，就会以返回值或者异常的方式告知用户。在编写发送消息的代码时，需要注意，**正确处理返回值或者捕获异常**，就可以保证这个阶段的消息不会丢失。

**同步发送时，只要注意捕获异常即可。**

**异步发送时，则需要在回调方法里进行检查。这个地方需要特别注意，很多丢消息的原因就是，我们使用了异步发送，却没有在回调中检查发送结果。**

**2、存储阶段**

在存储阶段正常情况下，只要Broker在正常运行，就不会出现丢消息的问题；但是如果Broker出现故障，比如进程死掉或者服务器宕机，还是可能会丢失消息的。

如果对消息的可靠性要求非常高，可以通过配置Broker参数来避免因为宕机丢消息：

- 对于单个节点的 Broker，需要配置 Broker 参数，**在收到消息后，将消息写入磁盘后再给 Producer 返回确认响应**，这样即使发生宕机，由于消息已经被写入磁盘，就不会丢失消息，恢复后还可以继续消费。例如，在 RocketMQ 中，需要将刷盘方式 flushDiskType 配置为 SYNC_FLUSH 同步刷盘。
    
- 对于 Broker 是由多个节点组成的集群，需要将 Broker 集群配置成：**至少将消息发送到 2 个以上的节点，再给客户端回复发送确认响应**。这样当某个 Broker 宕机时，其他的 Broker 可以替代宕机的 Broker，也不会发生消息丢失。
    

**3、消息阶段**

消费阶段采用和生产阶段类似的**确认机制**来保证消息的可靠传递，客户端从 Broker 拉取消息后，执行用户的消费业务逻辑，**成功后，才会给 Broker 发送消费确认响应**。如果 Broker 没有收到消费确认响应，下次拉消息的时候还会返回同一条消息，确保消息不会在网络传输过程中丢失，也不会因为客户端在执行消费逻辑中出错导致丢失。

在编写消费代码时需要注意的是：不要在收到消息后就立即发送消费确认，而是应该**在执行完所有消费业务逻辑之后，再发送消费确认**。

  

### 4.处理消费过程中的重复消息

在消息传递过程中，如果出现**传递失败**的情况，发送方会**执行重试**，重试过程中就有可能**产生重复的消息**。如果没有对重复消息进行处理，就可能导致系统的数据出现错误。

比如，一个消费订单消息，统计下单金额的微服务，如果没有正确处理重复消息，那就会出现重复统计，导致统计结果错误。

_**一、消息重复的情况必然存在**_

在MQTT协议中，给出了三种传递消息时能够提供的服务质量标准：

- **At most once**：至多一次。最多会被送达一次，也就是说没有消息可靠性保证，允许丢消息。一般都是一些对消息可靠性要求不高的监控场景使用，比如每分钟上报一次机房温度数据，可以接受数据少量丢失。
    
- **At least once**：至少一次。至少会被送达一次，也就是说**不允许丢消息**，但是**允许有少量重复消息出现**。
    
- **Exactly once**：恰好一次。只会被送达一次，不允许丢失也不允许重复，这个是最高等级。
    

这个服务质量标准不仅适用于 MQTT，对所有的消息队列都是适用的。常用的**绝大部分消息队列提供的服务质量都是 At least once，包括 RocketMQ、RabbitMQ 和 Kafka** 。也就是说，消息队列很难保证消息不重复。

注意：Kafka 支持的“Exactly once”和我们刚刚提到的消息传递的服务质量标准“Exactly once”是不一样的，它是 Kafka 提供的另外一个特性，Kafka 中支持的事务也和我们通常意义理解的事务有一定的差异。在 Kafka 中，事务和 Excactly once 主要是为了配合流计算使用的特性。

_**二、用幂等性解决重复消息问题**_

幂等本来是一个数学上的概念，它的定义是：如果一个函数f(x)满足：f(f(x)) = f(x)，则函数f(x)满足米幂等性。扩展到计算机领域，被用来描述一个操作、方法或者服务。

- 一个幂等操作的特点是，其**任意多次执行所产生的影响均与一次执行的影响相同**。
    
- 一个幂等方法，使用同样的参数，对它进行多次调用和一次调用，对系统产生的影响是一样的。所以不用担心重复执行会对系统造成任何改变。
    

举例：

1、在不考虑并发的情况下，“将账户 X 的余额设置为 100 元”，执行一次后对系统的影响是，账户 X 的余额变成了 100 元。只要提供的参数 100 元不变，那即使再执行多少次，账户 X 的余额始终都是 100 元，不会变化，这个操作就是一个幂等的操作。

2、“将账户 X 的余额加 100 元”，这个操作它就不是幂等的，每执行一次，账户余额就会增加 100 元，执行多次和执行一次对系统的影响（也就是账户的余额）是不一样的。

如果消费消息的业务逻辑具备幂等性，那就不用担心消息重复的问题，因为同一条消息，**消费一次和消费多次对系统的影响是完全一样的**。消费多次等于消费一次。从对系统的影响结果来说：At least once + 幂等消费 = Exactly once。

实现幂等操作最好的方式是，**从业务逻辑设计上入手，将消费的业务逻辑设计成具备幂等性的操作**。

**常用的设计幂等操作的方法**：

（1）利用**数据库的唯一约束**实现幂等

上面提到的那个不具备幂等特性的转账的例子：将账户 X 的余额加 100 元。在这个例子中，我们可以通过改造业务逻辑，让它具备幂等性。

首先，我们可以限定，对于每个转账单每个账户只可以执行一次变更操作，在分布式系统中，这个限制实现的方法非常多，最简单的是我们在数据库中建一张转账流水表，这个表有三个字段：转账单 ID、账户 ID 和变更金额，然后给转账单 ID 和账户 ID 这两个字段联合起来创建一个唯一约束，这样对于相同的转账单 ID 和账户 ID，表里至多只能存在一条记录。

这样，我们消费消息的逻辑可以变为：“在转账流水表中**增加一条转账记录，然后再根据转账记录，异步操作更新用户余额即可**。”在转账流水表增加一条转账记录这个操作中，由于我们在这个表中预先定义了“账户 ID 转账单 ID”的唯一约束，**对于同一个转账单同一个账户只能插入一条记录，后续重复的插入操作都会失败**，这样就实现了一个幂等的操作。

基于这个思路，不光是可以使用关系型数据库，只要是支持类似“INSERT IF NOT EXIST”语义的存储类系统都可以用于实现幂等，比如，你可以用 Redis 的 SETNX 命令来替代数据库中的唯一约束，来实现幂等消费。

（2）为更新的数据设置前置条件

给数据变更设置一个前置条件，如果满足条件就更新数据，否则拒绝更新数据，在更新数据的时候，同时变更前置条件中需要判断的数据。这样，重复执行这个操作时，由于第一次更新数据的时候已经变更了前置条件中需要判断的数据，不满足前置条件，则不会重复执行更新数据操作。

比如，“将账户 X 的余额增加 100 元”这个操作并不满足幂等性，我们可以把这个操作加上一个前置条件，变为：“如果账户 X 当前的余额为 500 元，将余额加 100 元”，这个操作就具备了幂等性。对应到消息队列中的使用时，可以在发消息时在消息体中带上当前的余额，在消费的时候进行判断数据库中，当前余额是否与消息中的余额相等，只有相等才执行变更操作。

但是，如果我们要更新的数据不是数值，或者我们要做一个比较复杂的更新操作怎么办？用什么作为前置判断条件呢？更加通用的方法是，给你的**数据增加一个版本号属性**，每次更数据前，**比较当前数据的版本号是否和消息中的版本号一致**，如果不一致就拒绝更新数据，**更新数据的同时将版本号 +1**，一样可以实现幂等更新。

（3）记录并检查操作

如果上面提到的两种实现幂等方法都不能适用于你的场景，还有一种通用性最强，适用范围最广的实现幂等性方法：记录并检查操作，也称为“Token 机制或者 GUID（全局唯一 ID）机制”，实现的思路特别简单：**在执行数据更新操作之前，先检查一下是否执行过这个更新操作**。这种方法适用范围最广，但是实**现难度和复杂度也比较高，一般不推荐使用**。

具体的实现方法是，在发送消息时，给每条消息**指定一个全局唯一的 ID**，消费时，先根据这个 ID 检查这条消息是否有被消费过，如果没有消费过，才更新数据，然后**将消费状态置为已消费**。

在分布式系统中，这个方法其实是非常难实现的。首先，给每个消息指定一个全局唯一的 ID 就是一件不那么简单的事儿，方法有很多，但都不太好同时满足简单、高可用和高性能，或多或少都要有些牺牲。更加麻烦的是，在“检查消费状态，然后更新数据并且设置消费状态”中，**三个操作必须作为一组操作保证原子性**，才能真正实现幂等，否则就会出现 Bug。

比如说，对于同一条消息：“全局 ID 为 8，操作为：给 ID 为 666 账户增加 100 元”，有可能出现这样的情况：

- t0 时刻：Consumer A 收到条消息，检查消息执行状态，发现消息未处理过，开始执行“账户增加 100 元”；
    
- t1 时刻：Consumer B 收到条消息，检查消息执行状态，发现消息未处理过，因为这个时刻，Consumer A 还未来得及更新消息执行状态。
    

这样就会导致账户被错误地增加了两次 100 元，这是一个在分布式系统中非常容易犯的错误，一定要引以为戒。对于这个问题，当然我们可以用事务来实现，也可以用锁来实现，但是在分布式系统中，无论是分布式事务还是分布式锁都是比较难解决问题。

  

### 5.利用事务消息实现分布式事务

_**一、消息事务**_

其实很多场景下，我们“发消息”这个过程，目的往往是**通知另外一个系统或者模块去更新数据**，消息队列中的“事务”，主要解决**消息生产者和消息消费者的数据一致性问题**。

用户在电商APP上购物时，先把商品加到购物车里，然后几件商品一起下单，最后支付，完成购物流程。

这个过程中有一个需要用到消息队列的步骤，订单系统创建订单后，发消息给购物车系统，将已下单的商品从购物车中删除。因为从购物车删除已下单商品这个步骤，并不是用户下单支付这个主要流程中必要的步骤，使用消息队列来异步清理购物车是更加合理。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

对于订单系统，它创建订单的过程实际执行了2个步骤的操作：

- 在订单库中插入一条订单数据，创建订单；
    
- 发消息给消息队列，消息的内容就是刚刚创建的订单
    

对于购物车系统：

- 订阅相应的主题，接收订单创建的消息，然后清理购物车，在购物车中删除订单的商品。
    

在分布式系统中，上面提到的步骤，任何一个都有可能失败，如果不做任何处理，那就有可能出现订单数据与购物车数据不一致的情况，比如：

- 创建了订单，没有清理购物车；
    
- 订单没创建成功，购物车里面的商品却被清掉了。
    

所以我们需要解决的问题为：在上述任意步骤都有可能失败的情况下，还要保证订单库和购物车库这两个库的数据一致性。

_**二、分布式事务**_

分布式事务就是要在分布式系统中实现事务。在分布式系统中，在保证可用性和不严重牺牲性能的前提下，光是要**实现数据的一致性就已经非常困难了**，显然实现严格的分布式事务是更加不可能完成的任务。所以目前大家所说的分布式事务，更多情况下，是在**分布式系统中事务的不完整实现**，在不同的应用场景中，有不同的实现，目的都是通过一些妥协来解决实际问题。

常见的分布式事务实现：

- 2PC（Two-phase Commit，也叫二阶段提交）
    
- TCC（Try-Confirm-Cancel）
    
- 事务消息
    

每一种实现都有其特定的使用场景，也有各自的问题，都不是完美的解决方案。

**事务消息适用的场景**主要是那些需要**异步更新数据**，并且**对数据实时性要求不太高**的场景。比如在创建订单后，如果出现短暂的几秒，购物车里的商品没有被及时情况，也不是完全不可接受的，只要最终购物车的数据和订单数据保持一致就可。

_**三、消息队列实现分布式事务**_

**事务消息需要消息队列提供相应的功能才能实现，kafka和RocketMQ都提供了事务相关功能。**

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

对于订单系统：

- 首先，订单系统在消息队列上开启一个事务。
    
- 然后订单系统给消息服务器发送一个“半消息”，这个半消息不是说消息内容不完整，它包含的内容就是完整的消息内容，半消息和普通消息的唯一区别是，在事务提交之前，对于消费者来说，这个消息是不可见的。
    
- 半消息发送成功后，订单系统就可以执行本地事务了，在订单库中创建一条订单记录，并提交订单库的数据库事务。
    
- 然后根据本地事务的执行结果决定提交或者回滚事务消息。如果订单创建成功，那就提交事务消息，购物车系统就可以消费到这条消息继续后续的流程。如果订单创建失败，那就回滚事务消息，购物车系统就不会收到这条消息。这样就基本实现了“要么都成功，要么都失败”的一致性要求。
    

对于购物车系统：

- 对于购物车系统收到订单创建成功消息清理购物车这个操作来说，失败的处理比较简单，**只要成功执行购物车清理后再提交消费确认即可**，如果失败，**由于没有提交消费确认，消息队列会自动重试**。
    

**如果在第四步提交事务消息时失败了怎么办？Kafka 和 RocketMQ 给出了 2 种不同的解决方案：**

1、Kafka 的解决方案：

直接抛出异常，让用户自行处理。我们可以在业务代码中反复重试提交，直到提交成功，或者删除之前创建的订单进行补偿。

2、RocketMQ 的解决方案：

在 RocketMQ 中的事务实现中，增加了**事务反查的机制**来解决事务消息提交失败的问题。如果 Producer 也就是订单系统，在提交或者回滚事务消息时发生网络异常，RocketMQ 的 Broker 没有收到提交或者回滚的请求，Broker 会定期去 Producer 上**反查这个事务对应的本地事务的状态**，然后根据反查结果决定提交或者回滚这个事务。为了支撑这个事务反查机制，我们的业务代码需要**实现一个反查本地事务状态的接口**，告知 RocketMQ 本地事务是成功还是失败。

综合上面讲的通用事务消息的实现和 RocketMQ 的事务反查机制，使用 **RocketMQ 事务消息功能实现分布式事务的流程**如下图：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

### 6.消息队列中的顺序问题

当我们说顺序时，我们在说什么？

日常思维中，顺序大部分情况会和时间关联起来，即时间的先后表示事件的顺序关系。

比如事件A发生在下午3点一刻，而事件B发生在下午4点，那么我们认为事件A发生在事件B之前，他们的顺序关系为先A后B。

上面的例子之所以成立是因为他们有相同的参考系，即他们的时间是对应的同一个物理时钟的时间。如果A发生的时间是北京时间，而B依赖的时间是东京时间，那么先A后B的顺序关系还成立吗？

_如果没有一个绝对的时间参考，那么A和B之间还有顺序吗，或者说怎么断定A和B的顺序？_

显而易见的，如果A、B两个事件之间如果是有因果关系的，那么A一定发生在B之前（前因后果，有因才有果）。相反，在没有一个绝对的时间的参考的情况下，若A、B之间没有因果关系，那么A、B之间就没有顺序关系。

**那么，我们在说顺序时，其实说的是：**

- **有绝对时间参考的情况下，事件的发生时间的关系；**
    
- **和没有时间参考下的，一种由因果关系推断出来的happening before的关系；**
    

在分布式环境中讨论顺序

当把顺序放到分布式环境（多线程、多进程都可以认为是一个分布式的环境）中去讨论时：

- 同一线程上的事件顺序是确定的，可以认为他们有相同的时间作为参考
    
- 不同线程间的顺序只能通过因果关系去推断
    

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

_（点表示事件，波浪线箭头表示事件间的消息）_

上图中，进程P中的事件顺序为p1->p2->p3->p4（时间推断）。而因为p1给进程Q的q2发了消息，那么p1一定在q2之前（因果推断）。但是无法确定p1和q1之间的顺序关系。

推荐阅读《Time, Clocks, and the Ordering of Events in a Distributed System》，会透彻的分析分布式系统中的顺序问题。

消息中间件中的顺序消息

什么是顺序消息

有了上述的基础之后，我们回到本篇文章的主题中，聊一聊消息中间件中的顺序消息。

> 顺序消息（FIFO 消息）是 MQ 提供的一种严格按照顺序进行发布和消费的消息类型。顺序消息由两个部分组成：顺序发布和顺序消费。
> 
> 顺序消息包含两种类型：
> 
> 分区顺序：一个Partition内所有的消息按照先进先出的顺序进行发布和消费
> 
> 全局顺序：一个Topic内所有的消息按照先进先出的顺序进行发布和消费

这是阿里云上对顺序消息的定义，把顺序消息拆分成了顺序发布和顺序消费。那么多线程中发送消息算不算顺序发布？

如上一部分介绍的，多线程中若没有因果关系则没有顺序。那么用户在多线程中去发消息就意味着用户不关心那些在不同线程中被发送的消息的顺序。即多线程发送的消息，不同线程间的消息不是顺序发布的，同一线程的消息是顺序发布的。这是需要用户自己去保障的。

而对于顺序消费，则需要保证哪些来自同一个发送线程的消息在消费时是按照相同的顺序被处理的（为什么不说他们应该在一个线程中被消费呢？）。

全局顺序其实是分区顺序的一个特例，即使Topic只有一个分区（以下不在讨论全局顺序，因为全局顺序将面临性能的问题，而且绝大多数场景都不需要全局顺序）。

如何保证顺序

在MQ的模型中，顺序需要由3个阶段去保障：

1. 消息被发送时保持顺序
    
2. 消息被存储时保持和发送的顺序一致
    
3. 消息被消费时保持和存储的顺序一致
    

发送时保持顺序意味着对于有顺序要求的消息，用户应该在同一个线程中采用同步的方式发送。存储保持和发送的顺序一致则要求在同一线程中被发送出来的消息A和B，存储时在空间上A一定在B之前。而消费保持和存储一致则要求消息A、B到达Consumer之后必须按照先A后B的顺序被处理。

如下图所示：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

对于两个订单的消息的原始数据：a1、b1、b2、a2、a3、b3（绝对时间下发生的顺序）：

- 在发送时，a订单的消息需要保持a1、a2、a3的顺序，b订单的消息也相同，但是a、b订单之间的消息没有顺序关系，这意味着a、b订单的消息可以在不同的线程中被发送出去
    
- 在存储时，需要分别保证a、b订单的消息的顺序，但是a、b订单之间的消息的顺序可以不保证
    
-   
    

- a1、b1、b2、a2、a3、b3是可以接受的
    
- a1、a2、b1、b2、a3、b3也是可以接受的
    
- a1、a3、b1、b2、a2、b3是不能接受的
    

- 消费时保证顺序的简单方式就是“什么都不做”，不对收到的消息的顺序进行调整，即只要一个分区的消息只由一个线程处理即可；当然，如果a、b在一个分区中，在收到消息后也可以将他们拆分到不同线程中处理，不过要权衡一下收益
    

开源RocketMQ中顺序的实现

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

上图是RocketMQ顺序消息原理的介绍，将不同订单的消息路由到不同的分区中。文档只是给出了Producer顺序的处理，Consumer消费时通过一个分区只能有一个线程消费的方式来保证消息顺序，具体实现如下。

**Producer端**

Producer端确保消息顺序唯一要做的事情就是将消息路由到特定的分区，在RocketMQ中，通过MessageQueueSelector来实现分区的选择。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

- Listmqs：消息要发送的Topic下所有的分区
    
- Message msg：消息对象
    
- 额外的参数：用户可以传递自己的参数
    

比如如下实现就可以保证相同的订单的消息被路由到相同的分区：

```
long orderId = ((Order) object).getOrderId;
```

**Consumer端**

RocketMQ消费端有两种类型：MQPullConsumer和MQPushConsumer。

MQPullConsumer由用户控制线程，主动从服务端获取消息，每次获取到的是一个MessageQueue中的消息。PullResult中的List msgFoundList自然和存储顺序一致，用户需要再拿到这批消息后自己保证消费的顺序。

对于PushConsumer，由用户注册MessageListener来消费消息，在客户端中需要保证调用MessageListener时消息的顺序性。RocketMQ中的实现如下：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

1. PullMessageService单线程的从Broker获取消息
    
2. PullMessageService将消息添加到ProcessQueue中（ProcessMessage是一个消息的缓存），之后提交一个消费任务到ConsumeMessageOrderService
    
3. ConsumeMessageOrderService多线程执行，每个线程在消费消息时需要拿到MessageQueue的锁
    
4. 拿到锁之后从ProcessQueue中获取消息
    

保证消费顺序的核心思想是：

- 获取到消息后添加到ProcessQueue中，单线程执行，所以ProcessQueue中的消息是顺序的
    
- 提交的消费任务时提交的是“对某个MQ进行一次消费”，这次消费请求是从ProcessQueue中获取消息消费，所以也是顺序的（无论哪个线程获取到锁，都是按照ProcessQueue中消息的顺序进行消费）
    

顺序和异常的关系

顺序消息需要Producer和Consumer都保证顺序。Producer需要保证消息被路由到正确的分区，消息需要保证每个分区的数据只有一个线程消息，那么就会有一些缺陷：

- 发送顺序消息无法利用集群的Failover特性，因为不能更换MessageQueue进行重试
    
- 因为发送的路由策略导致的热点问题，可能某一些MessageQueue的数据量特别大
    
- 消费的并行读依赖于分区数量
    
- 消费失败时无法跳过
    

不能更换MessageQueue重试就需要MessageQueue有自己的副本，通过Raft、Paxos之类的算法保证有可用的副本，或者通过其他高可用的存储设备来存储MessageQueue。

热点问题好像没有什么好的解决办法，只能通过拆分MessageQueue和优化路由方法来尽量均衡的将消息分配到不同的MessageQueue。

消费并行度理论上不会有太大问题，因为MessageQueue的数量可以调整。

消费失败的无法跳过是不可避免的，因为跳过可能导致后续的数据处理都是错误的。不过可以提供一些策略，由用户根据错误类型来决定是否跳过，并且提供重试队列之类的功能，在跳过之后用户可以在“其他”地方重新消费到这条消息。

![](http://mmbiz.qpic.cn/mmbiz_png/GfOyjHGwCoN5Ky9IZgsdyZgM3lbEC5Ixe3ZgxBu5MdzvjWFyC7uqrsffcyhtrD8r8t0qicxbGmNxKYr4l4mANKQ/300?wx_fmt=png&wxfrom=19)

**面试突击**

大厂面试突击，专注分享面试题，如计算机基础、计算机网络、Java后端、前端Vue。

16篇原创内容

公众号

阅读 1816

​

写留言

**留言 3**

- 悟空呀
    
    2021年12月6日
    
    赞
    
    建议收藏起来慢慢看
    
    置顶
    
- GIY
    
    2021年12月6日
    
    赞
    
    大佬牛逼，加入待学计划![[666]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
    悟空聊架构
    
    作者2021年12月6日
    
    赞
    
    roll slowly
    

已无更多数据

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/SfAHMuUxqJ2JicQpKzblLHz64qoibVa3ATNA4rH8mIYXAF3OErAzxFKHzf5qiaiblb4rAMuAXXMJHEcKcvaHv4ia9rA/300?wx_fmt=png&wxfrom=18)

悟空聊架构

825

3

写留言

**留言 3**

- 悟空呀
    
    2021年12月6日
    
    赞
    
    建议收藏起来慢慢看
    
    置顶
    
- GIY
    
    2021年12月6日
    
    赞
    
    大佬牛逼，加入待学计划![[666]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
    悟空聊架构
    
    作者2021年12月6日
    
    赞
    
    roll slowly
    

已无更多数据