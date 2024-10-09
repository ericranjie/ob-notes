原创 丁威 中间件兴趣圈

_2021年11月20日 08:32_

点击上方“中间件兴趣圈”，选择“设为星标”

越努力越幸运，唯有坚持不懈！[![图片](https://mmbiz.qpic.cn/mmbiz_png/Wkp2azia4QFtRsQop4YtGM7EiaiaXNibdElEDFKEfJmfXo7yxtC5GRPOJveia0HbBMRAW3NUV8qW77U0B5RF2uoxE8w/640?wx_fmt=png&wxfrom=13&tp=wxpic)](http://mp.weixin.qq.com/s?__biz=MzIzNzgyMjYxOQ==&mid=2247486570&idx=1&sn=25afad26c7986aaaf32e0be8de2fc334&chksm=e8c3fb9edfb4728831bd29cbc6b9805baea8b2670401cf54f21f0675c6026811bea511dac811&scene=21#wechat_redirect)

Broker端与客户端的心跳在Kafka中非常的重要，因为一旦在一个心跳过期周期内(默认10s)，Broker端的消费组组协调器(GroupCoordinator)会把消费者从消费组中移除，从而触发重平衡。在2.4.x以下其版本中，消费组一旦进入重平衡状态，该消费组内所有消费者全部暂停消费，直到重平衡完成。

本文将来探讨Kafka的心跳机制的具体实现。本文的组织结构如下：

- 源码解读Kafka心跳机制

- Kafka心跳架构设计亮点(时间轮调度算法实现原理图)

> 温馨提示：如果大家对源码阅读不感兴趣，可以直接跳到本文的第二部分，用流程图、数据结构图阐述心跳的实现机制。

## 1、源码分析Kafka心跳机制

在介绍源码分析之前介绍笔直的一条**源码分析经验：找准入口，了解调用链路**。故笔者会先寻找归纳出Kafka心跳处理的所有入口。

#### 1.1Kafka心跳入口总结

Kafka心跳包的处理流程如下图所示：

![图片](https://mmbiz.qpic.cn/mmbiz_png/Wkp2azia4QFtvCeIhCL0pZh4ibXkfibqhP4budnetAWuoOtEWiclerfEAH2vyUfDBlkSMTK1FzaMPsllxsFCnwsgqg/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

图的右边是kafka心跳在服务端的核心处理流程，而左边主要展示kafka中所有的心跳请求，根据上图得知Kafka触发心跳处理的主要请求分别如下：

1. KafkaConsume主动发送心跳包 消费者会以3s的频率向服务端发送心跳包，服务端对应的入口为 KafkaApis的handleHeartbeatRequest方法。

1. 消费者加入消费组 在消费端重平衡过程中，客户端主动向其组协调器发起Join_Group(加入消费组)时，组协调器会认为收到一个有效的心跳包，服务端对应的处理入口：KafkaApis的handleJoinGroup方法。

1. 消费者获取队列负载结果 在重平衡的第二个阶段，消费组的Leader在计算出分区负载结果后会发给组协调器，消费组中的其他成员需要发生Sync_Group请求获取负载结果，组协调器同样认为收到了一个有效的心跳包。服务端对应的处理入口：KafkaApis的handleSyncGroupRequest。

1. 消费者提交位点 消费者组协调器收到消费者提交位点请求，同样可以认定消费者是存活的。位点提交的处理入口：KafkaApis的handlerCommitOffsets方法。

1. \_\_consumers_offsets主题的ISR的Leader发生变化

   如果\_\_consumers_offsets主题中的各个分区Leader发生变化，与特定分区的组协调器需要重新选举，与此组协调器相关的消费者将触发重平衡。

> 上述任何一种请求，都能表明消费端是**存活的**，故能有效阻止服务端将客户端端心跳设置为过期，进入下一个心跳检测周期。

**上述各个入口，特别是\_\_consumers_offsets的ISR对消费组的影响，后续会专门展开研究，现在我们将重心转移到服务端是如何处理一个心跳包的。**

#### 1.2 源码分析Kafka心跳处理机制

从上面的流程图可以得出，Kafka收到一个心跳包后的处理入口为GroupCoordinator的completeAndScheduleNextExpiration方法，核心代码如下图所示：

![图片](https://mmbiz.qpic.cn/mmbiz_png/Wkp2azia4QFtvCeIhCL0pZh4ibXkfibqhP4A8KojeV2K7mpSicibamNItWkwmU69Gric54N2OQc3EjqLOleeCicggxUlw/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

在介绍该方法之前首先介绍一个该方法的入参含义：

- GroupMetadata group 消费组的元信息。

- MemberMetadata member 消费者的元信息。

- long timeoutMs 心跳超时时间，默认为10s，这个参数是由消费端的**session.timeout.ms**参数设置，默认为10s。

Step1：为消费组设置唯一标识：groupId + "-" + memberId构成。

Step2：将hearbeatSatisfied设置为true，表示该消费者收到一个有效的心跳包。

Step3：收到一个有效的心跳包，通知定时调度器停止本次的心跳过期检测。

Step4：构建一个DelayedHearbeat，进入下一个心跳检测周期。

接下来将分别对Step3、Step4展开详细介绍。

##### 1.2.1 心跳检测正常处理逻辑

在收到一个心跳包时，尝试将本次检测设置成功，具体的实现由DelayedOperation的checkAndComplete方法，代码如下：

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

Kafka使用一个数据结构来存储需要跟踪的所有消费者，在这里成为Watch机制。

实现要点：根据key获取WatchList，然后从获取的WatchList中内部的ConcurrentMap中再按照Key获取对应与当前消费者对应的Watch。

- 如果没有找到对应消费者的Watch，则直接返回，无需检测，说明已经成功检测。

- 如果找到了对应消费者的Watch，则执行被watch的tryCompleteWatched方法。

Watch的数据结构如下：

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)接下来重点关注Watches的tryCompleteWatched方法，该方法的详细调用代码如下图所示：

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

这边先重点介绍一下组协调器判断一次成功的心跳检测的三个标准中满足一个即可(GroupCoordinator的tryCompleteHeartbeat方法)：

- 如果消费组的状态处于Dead

- 如果消费组的状态为Pending(消费组在重平衡中)

- hearbeatSatisfied为true，即收到了一个有效的心跳包。

上述代码的实现比较简单，这里就不一一罗列，其核心关键点如下：

- 删除对应的Watch，表示一次心跳检测成功。

- Watchs中存储的对象是DelayedOperation(Kafka延迟类型的父类)的子类，在心跳检测中具体为DelayedHeartbeat。

- 最终执行DelayedOperation的是TimeTask的cancel方法（**取消延迟任务**），就是从延迟调度中移除自己，表示没有超时，结束本轮的超时检测，具体的存储结构，将在下文详介绍如果开启新一轮心跳检测时再详细讲解。

为了方便大家阅读源码，其主要的调用时序图如下：

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

##### 1.2.2 开启下一轮心跳检测

###### 1.2.2.1将延迟任务放入时间轮

在接受到一个新的心跳包首先用于清除上一轮设置的延迟任务，然后需要开启一个新的延迟任务，接下来我们将来具体看看Kafka如何开启新一轮心跳检测机制，\*\*其本质上是Kafka的延迟(定时)实现原理。\*\*代码入口如下图所示：

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

开启下一轮调度时首先将Member的heartbeatSatisfied设置为false。

其核心思想是创建一个心跳延迟任务DelayedHeartbeat，并对其检测是否完成或者添加Watch，启动心跳延迟或者等待下一个心跳包的到来。

其实看到这里，我们应该能得到一个关于Kafka心跳检测机制的实现思路：

- 开启一个延迟任务，延迟检查时间为心跳过期时间，一旦延迟任务执行，则意外着心跳超时。

- 当收到一个心跳包时，需要取消上一次设置的延迟任务。

- 使用循环使用延迟任务，从而实现类似定时任务的效果。

接下来我们详细探讨一下DelayedOperationPurgatory的tryCompleteElseWatch方法，其代码如下图所示：

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

Step1：尝试调用DelayedHeartbeat的tryComplete方法，判断是否可以判断完成，这里主要是消费组是否为重平衡或者状态为Dead，如果上述情况不满足，则会返回false，因为在发起下一轮心跳包时已将heartbeatSatisfied设置为false。

Step2：为该消费者添加到Watch中，表示kafka需要跟踪该消费者的心跳。

Step3：再次调用maybeTryComplete方法，再尝试判断是否该心跳检测完成。

Step4：如果没有完成，则该任务延迟任务(DelayedHeartbeat)添加到定时调度中。

接下来将进入到Kafka心跳的核心机制，即**延迟任务的实现机制**。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

每一个待执行的延迟任务被封装在TimeTaskEntry中，这个一个典型的双链表，数据结构说明说明如下：

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

并持有一个关键字段：该定时任务的过期时间，等于系统当前时间+过期时间，在心跳检测场景中默认为10s。

继续跟踪SystemTimer的addTimerTaskEntry，其代码如下：

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

addTimerTaskEntry的核心实现如下：

- 尝试将延迟任务添加到时间轮，如果已经过期，则提交到线程池，触发心跳过期的逻辑，提交到线程后，DelayedOperation的run方法会被调用，最终onExpiration方法被调用。

  !\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

接下来重点谈一下往时间轮中添加任务的具体实现，核心代码见下图所示：

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

核心实现要点：

Step1：如果任务已经被取消或者已过期，返回false。如果返回false，则会触发定时任务过期。

Step2根据过期时间，放入到时间轮中指定的位置，时间轮的数据结构如下：

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

每一个格代表一个时间间隔，例如200ms，当前指针指向的格子，代表该格子中的所有任务过期，例如现在要要插入一个700ms过期，从当前指针的下一格开始算起，放入第4格中。

另外时间轮的总格子有限，则该时间轮能计算的最大时间是有限的，例如一个8格的时间轮，每一格代表200ms,则如果要在2s后过期，显然这个时间轮无法存储，通常的解决方案是采用多级时间轮，另外一级的时间轮，其时间精度会更粗。

结合上述关于时间轮的原理，再去看上述代码，就显得容易看懂了。

Step3:就是处理第一级时间轮无法满足过期时间，则放入到第二级时间轮中。

###### 1.2.2.2 驱动时间轮

基于时间轮算法，除了数据按找时间轮到方向、触发时间存储在合适的刻度量，还需要驱动时间轮指针。Kafka中的驱动时间轮入口为：

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

具体实现代码如下：

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

具体就是将指针处的所有任务全部拉取出来，执行addTimeTaskEntry，其中过期的任务将提交到线程池触发延迟任务的执行。

上述代码看起来比较简单，就不一一介绍，为了方便大家读懂上面的代码，我们只需要了解一下kafka采用时间轮的实际存储数据结构，即能很容易理解上述代码：

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

其核心特点：**环形队列就是一个数组，每一个元素在Kafka中对应一个桶，每一个桶存储一个TimerTaskList(链表)，每次指针指向的TimerTaskList，将该链表中的元素代表的任务全部执行。**

## 2、图解Kafka心跳架构设计

读起源码来说或许比较枯燥，接下来给出Kafka心跳处理的图解，重点是阐述Kafka时间轮算法的核心数据结构。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

### 最后说一句(求关注，别白嫖我)

如果这篇文章对您有所帮助，或者有所启发的话，帮忙扫描下发二维码关注一下，您的支持是我坚持写作最大的动力。

求一键三连：点赞、转发、在看。

![](http://mmbiz.qpic.cn/mmbiz_png/Wkp2azia4QFvs5UnzjhzrS6h29QXbfK4I0GLibNP4Qlpt1ovSdmwfsoY7D4JYUZzkACtqe3wrKh6icG7oHUTMibJbA/300?wx_fmt=png&wxfrom=19)

**中间件兴趣圈**

《RocketMQ技术内幕》作者维护，主打成体系剖析JAVA主流中间件架构与设计原理，为构建完备的互联网分布式架构体系而努力，助力突破职场瓶颈。

232篇原创内容

公众号

关注公众号：「中间件兴趣圈」，在公众号中回复：「PDF」可获取大量学习资料，回复「专栏」可获取15个主流Java中间件源码分析专栏，另外回复：加群，可以跟很多BAT大厂的前辈交流和学习。

走进作者

[10年IT老兵给职场新人的一些建议](http://mp.weixin.qq.com/s?__biz=MzIzNzgyMjYxOQ==&mid=2247484618&idx=1&sn=e24d7d19006f0d66e697e8d2be4aa508&chksm=e8c3f33edfb47a286f4515c4b11e822c35eab9b6c7ada25ac2cce3d2f7e5dac0230b54c56646&scene=21#wechat_redirect)

[“我”被阿里巴巴宠幸了](http://mp.weixin.qq.com/s?__biz=MzIzNzgyMjYxOQ==&mid=2247486100&idx=1&sn=3166338465f9b4a47ad93ecf13df6e48&chksm=e8c3fd60dfb47476e0c3ff65673eee47a5b99c7455f70252d08d6d0330828ea9050b27526a7d&scene=21#wechat_redirect)

[程序员如何提高影响力](http://mp.weixin.qq.com/s?__biz=MzIzNzgyMjYxOQ==&mid=2247485407&idx=1&sn=0e0de515b3a66ac91e55fdf583be5c0d&chksm=e8c3f02bdfb4793daecebbead9c5cdf6e64da25b80f2fd3f2bcfcc52a6c3a57b2414298bd0b5&scene=21#wechat_redirect)

[优秀程序员必备技能之如何高效阅读源码](http://mp.weixin.qq.com/s?__biz=MzIzNzgyMjYxOQ==&mid=2247485573&idx=1&sn=4d936fa700b38b5158316bdaf1aeac68&chksm=e8c3ff71dfb476675613afe09c682bc5fbd454b35f8d3d6d0458360149d5f0d673965c8852c4&scene=21#wechat_redirect)

[我的另一种参与 RocketMQ 开源社区的方式](http://mp.weixin.qq.com/s?__biz=MzIzNzgyMjYxOQ==&mid=2247484822&idx=1&sn=ecaada01b1bcf73b3a9fb750872b8e9d&chksm=e8c3f262dfb47b74d6f03be903dc734953e83ee720ac5b98e7ffcd92da39df5d68308b26bf85&scene=21#wechat_redirect)

点击查看“阅读原文”，**可直接进入\[中间件兴趣圈\]文章合集。**

Kafka实战与进阶25

Kafka实战与进阶 · 目录

上一篇答读者问：Kafka顺序消费吞吐量下降该如何优化？下一篇双十一期间Kafka以这种方式丢消息让我猝不及防

阅读原文

阅读 2797

​

写留言

**留言 4**

- 棉被精

  2021年11月21日

  赞

  应届生明年入职，打听过了要用Kafka，还没接触过。能给推荐一些学习资料吗![[嘿哈]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

  中间件兴趣圈

  作者2021年11月21日

  赞1

  推荐看看 胡夕大佬的 kafka实战这本书籍

  棉被精

  2021年11月21日

  赞

  好的谢谢![[社会社会]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![[社会社会]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

- 0.0

  2021年11月20日

  赞1

  必须要给最喜欢的时间轮留言

已无更多数据

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/Wkp2azia4QFvs5UnzjhzrS6h29QXbfK4I0GLibNP4Qlpt1ovSdmwfsoY7D4JYUZzkACtqe3wrKh6icG7oHUTMibJbA/300?wx_fmt=png&wxfrom=18)

中间件兴趣圈

18310

4

写留言

**留言 4**

- 棉被精

  2021年11月21日

  赞

  应届生明年入职，打听过了要用Kafka，还没接触过。能给推荐一些学习资料吗![[嘿哈]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

  中间件兴趣圈

  作者2021年11月21日

  赞1

  推荐看看 胡夕大佬的 kafka实战这本书籍

  棉被精

  2021年11月21日

  赞

  好的谢谢![[社会社会]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![[社会社会]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

- 0.0

  2021年11月20日

  赞1

  必须要给最喜欢的时间轮留言

已无更多数据
