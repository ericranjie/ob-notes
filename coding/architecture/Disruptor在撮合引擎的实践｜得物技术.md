原创 天佑 得物技术

_2024年07月03日 18:30_ _上海_

**目录**

一、Disruptor的简介

1\. Disruptor的使用场景

2\. Disruptor和ArrayBlockingQueue性能对比

3\. Disruptor快速接入指南

4\. Disruptor消费者等待策略

5\. Disruptor灵活的消费者模式

二、Disruptor的核心概念

1\. Disruptor内部组件交互图

2\. 核心概念

三、Disruptor的特点

1\. 环形数组结构

2\. 无锁化设计

3\. 独占缓存行的方式消除伪共享

4\. 预分配内存

四、Disruptor在撮合引擎中的应用

1\. 数字货币交易系统的简介

2\. 撮合引擎流程图

3\. 撮合引擎之Disruptor代码

五、总结

**一**

**Disruptor的简介**

Disruptor是基于事件异步驱动模型实现的，采用了RingBuffer数据结构，支持高并发、低延时、高吞吐量的高性能工作队列，它是由英国外汇交易公司LMAX开发的，研发的初衷是解决内存队列的延迟问题，不同于我们常用的分布式消息中间件RocketMQ、Kafaka，而Disruptor是单机的、本地内存队列，类似JDK的ArrayBlockingQueue等队列。

**Disruptor的使用场景**

- 加密货币交易撮合引擎

- Log4j2基于Disruptor实现的异步日志处理

- Canal+Disruptor实现高效的数据同步

- 知名开源框架Apache Strom

2010年在QCon的演讲，介绍了基于Disruptor开发的系统单线程能支撑每秒600万订单，由此可见该组件可以大幅提升系统的TPS，所以对于一些需要大幅提升单机应用的吞吐量的场景可以考虑使用Disruptor。

**Disruptor和ArrayBlockingQueue性能对比**

- ArrayBlockingQueue是基于数组ArrayList实现的，通过ReentrantLock独占锁保证线程安全；

- Disruptor是基于环形数组队列RingBuffer实现的，通过CAS乐观锁保证线程安全。在多种生产者-消费者模式下的性能对比。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

Figure 1. Unicast: 1P–1C

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

Figure 2. Three Step Pipeline: 1P–3C

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

Figure 3. Sequencer: 3P–1C

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

Figure 4. Multicast: 1P–3C

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

Figure 5. Diamond: 1P–3C

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**Disruptor快速接入指南**

**引入Maven依赖**

<dependency>

<groupld>com.lmax</groupld>

<artifactld>disruptor</artifactld>

<version>4.0.0</version>

</dependency>

**自定义事件和事件工厂**

```
public class LongEvent {
```

**定义事件处理器，即消费者**

```
public class LongEventHandler implements EventHandler<LongEvent> {
```

**定义事件生产者**

```
import com.lmax.disruptor.RingBuffer;
```

**编写启动类**

```
public class LongEventMain {
```

**Disruptor消费者等待策略**

等待策略WaitStrategy是一种决定一个消费者如何等待生产者将event对象放入Disruptor的方式/策略。

下面是常见的4种消费者等待策略：

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**Disruptor灵活的消费者模式**

**支持单生产者和多生产者**

构造Disruptor时指定生产者类型即可：ProducerType.SINGLE 和 ProducerType.MULTI

**单消费者**

```
//注册单个消费者
```

**多消费者：并行的、广播模式**

同一个事件会同时被所有消费者处理，同组内消费者之间不存在竞争关系。

```
//注册多个消费者
```

**多消费者：并行的、消费者组模式**

同组内消费者之间互斥，一个事件只会被同组内单个消费者处理，但可以支持多个消费者组，消费者组之间完全隔离，互不影响，代码实现方式有两点不同之处：

- 消费者需要实现WorkHandler接口，而不是 EventHandler 接口；

- 使用handleEventsWithWorkerPool设置Disruptor的消费者，而不是handleEventsWith方法。

```
public class LongWorkHandler  implements WorkHandler<LongEvent> {
```

- 多个消费者组之间并行模式

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

```
//注册消费者组1
```

- 多个消费者组之间航道执行模式

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

```
//注册消费者
```

**多消费者：链式、菱形、六边形执行模式**

通过多种组合方式，可实现灵活的消费者执行顺序，如下：

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

```
//链式
```

**二**

**Disruptor的核心概念**

**Disruptor内部组件交互图**

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**核心概念**

有些概念前面已经介绍过，在此不再赘述，说一说还未介绍的几个概念：

**Sequence**

Sequence本身就是一个序号管理器，它是严格顺序增长的，Disruptor通过它标识和定位RingBuffer中的每一个事件，每个Consumer都维护一个Sequence，通过Sequence可以跟踪Consumer事件处理进度，它有AtomicLong的大多数功能特性，而且它消除了CPU伪共享的问题。

**Sequencer**

Sequencer是一个接口，它有两个实现类：SingleProducerSequencer(单生产者实现)、MultiProducerSequencer(多生产者实现)，它主要作用是实现生产者和消费者之间快速、正确传递数据的并发算法。

Sequencer是生产者与缓冲区RingBuffer之间的桥梁。生产者可以通过Sequencer向RingBuffer申请数据的存放空间，并使用publish()方法通过WaiteStrategy通知消费者。

**SequenceBarrier（序列屏障）**

SequenceBarrier用于保证事件的有序性。它通过维护一组Sequence来跟踪消费者的进度，当生产者发布新的事件时，序列屏障会检查是否所有消费者都已处理完前面的事件，如果是，则通知生产者可以发布新的事件。

SequenceBarrier是消费者与RingBuffer之间的桥梁。在Disruptor中，消费者直接访问的是SequenceBarrier，而不是RingBuffer，因此SequenceBarrier能减少RingBuffer上的并发冲突，当消费者的消费速度大于生产者的生产速度时，消费者就可以通过waitFor()方法给予生产者一定的缓冲时间，从而协调了生产者和消费者的速度问题。

SequenceBarrier同时也是消费者与消费者之间消费依赖的抽象，SequenceBarrier只有一个实现类，即ProcessingSequenceBarrier。ProcessingSequenceBarrier由生产者Sequencer、消费定位cursorSequence、等待策略waitStrategy、还有一组依赖Sequence(dependentSequence)组成。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**三**

**Disruptor的特点**

**环形数组结构**

- 采用首尾相接的数组而非链表，无需担心index溢出问题，且数组对处理器的缓存机制更加友好；

- 在RingBuffer数组长度设置为2^N时，通过sequence & (bufferSize-1)加速定位元素实际下标索引，通过结合左移(\<\<)操作实现乘法；

- 结合SequenceBarrier机制，实现线程与线程之间高效的数据交互。

**无锁化设计**

每个生产者或者消费者线程，会先申请可以操作的元素在数组中的位置，申请到之后，直接在该位置写入或者读取数据，整个过程通过原子变量CAS，保证操作的线程安全，即Disruptor的Sequence的自增就是CAS的自旋自增，对应的ArrayBlockQueue的数组索引index是互斥自增。

**独占缓存行的方式消除伪共享**

**什么是伪共享**

出现伪共享问题（False Sharing）的原因：

- 一个缓存行可以存储多个变量（存满当前缓存行的字节数）；64个字节可以放8个long，16个int；

- 而CPU对缓存的修改又是以缓存行为最小单位的；不是以long 、byte这样的数据类型为单位的；

- 在多线程情况下，如果需要修改“共享同一个缓存行的其中一个变量”，该行中其他变量的状态就会失效，甚至进行一致性保护。

所以，伪共享问题（False Sharing）的本质是：

**CPU针对缓存的操作是以Cache Line为基本单位，对缓存行中的单个变量进行修改，会导致整个缓存行其他不相关的数据也都失效了，需要从主存重新加载，这个过程会带来性能损耗。**

**Disruptor是如何解决伪共享的**

Sequence是标识RingBuffer环形数组的下标，同时生产者和消费者也会维护各自的Sequence，最重要的是，**Sequence通过填充CPU缓存行避免了伪共享带来的性能损耗**，来看下其填充缓存行源码：

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**预分配内存**

环形队列存放的是Event对象，而且是在Disruptor创建的时候调用EventFactory创建并一次将队列填满。Event保存生产者生产的数据，消费者也是通过Event获取数据，后续生产者只需要替换掉Event中的属性值。这种方式避免了重复创建对象，降低JVM的GC频率，带来系统性能的提升。后续我们在做编码的时候其实也可以借鉴这种实现思路。

见com.lmax.disruptor.RingBuffer.fill(EventFactoryeventFactory)

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**四**

**Disruptor在撮合引擎中的应用**

**数字货币交易系统的简介**

\*\*背景&价值\
\*\*

为用户提供数字虚拟货币的实时在线交易平台，实现盈亏。

**C端核心界面**

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

以上截图仅用于技术展示，不构成投资建议

\*\*交易系统简化交互图\
\*\*

为了便于理解，简单列举交易系统的核心服务和数据流向，见下图：

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**撮合应用的特点**

- 纯内存的、CPU密集型的

应用启动时加载数据库未处理订单、写日志、撮合成功发送消息到MQ会涉及IO操作。

- 有状态的

正因为应用是有状态的，所以需要通过Disruptor提升单机的性能和吞吐量。

为什么撮合应用不设计成无状态的？

在学习或者实际做架构设计时，一般大多数情况都建议将应用设计为无状态的，可以通过水平扩展，实现应用的高可用、高性能。而有状态的应用一般有单点故障问题，难以通过水平扩展提升应用的性能，但是做架构设计的时候，还是需要从实际的场景出发，而撮合应用场景很显然更适合设计成有状态的。在数字加密货币交易平台，每一种数字加密货币都是由唯一的“交易对”去标识的，类似股票交易中的股票代码，针对不同交易对的买卖交易单是天然隔离的，而同种交易对的买卖交易单必须是在同一个应用去处理的，否则匹配撮合的时候是有问题的。如果使用无状态的设计，那么所有的交易对都必须在一个集群内处理，而且每个应用都必须要有全量交易对的订单数据，这样就会存在两个问题：多个应用撮合匹配结果不一致，以哪个为准、热点交易对如何做隔离，所以解决方案就是根据交易对维度对订单做分片，同一个交易对的订单消息路由到同一个撮合应用进行处理，这样其实就是将撮合应用设计成有状态的。每一种交易对每个时刻有且只有一个应用能处理，然后再通过k8s的Liveness和Readiness探针做自动故障转移和恢复来解决单点故障的问题，最后通过本地缓存Caffeine+高性能队列Disruptor提升单pod的吞吐量。16C64G的配置在实际业务场景压测的结果是，单机最大TPS在200w/s左右，对于整个交易系统而言性能瓶颈已经不在撮合应用，因为极端情况下可以配置成一个pod处理一个交易对。

**撮合引擎流程图**

撮合引擎服务核心链路流程图：

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**撮合引擎之Disruptor代码**

为了便于理解，删除了和Disruptor无关的代码，只列举和Disruptor相关联的代码。

**定义事件：用户交易单**

```
@Data
```

**定义事件处理器：对用户买单和卖单进行撮合匹配**

```
//撮合事件处理器
```

**事件生产者：构建Disruptor、生产事件**

```
/**
```

```
public class ExchangeCore extends AbstractLifeCycle {
```

```
public class MatchEventPublisher {
```

**五**

**总结**

Disruptor作为一个以高性能著称的队列，它有很多优秀的设计思想值得我们学习，比如环形数组队列RingBuffer、SequenceBarrier机制、无锁化设计、预分配内存、消除伪共享、以及灵活丰富的生产者和消费者模式。本文只是介绍了一些对Disruptor的基本功能和实际使用场景，后续大家有兴趣可以结合源码去做更加深入的理解。由于本人文笔和经验有限，若有不足之处，还请及时指正，共同学习和进步。

**引用：**

https://lmax-exchange.github.io/disruptor/user-guide/#\_advanced_techniques

**往期回顾**

1. [Apache Flink类型及序列化研读&生产应用｜得物技术](http://mp.weixin.qq.com/s?__biz=MzkxNTE3ODU0NA==&mid=2247527101&idx=1&sn=bee193b981bb15d10f65446a8f25e2a7&chksm=c1613fe2f616b6f42e89d413fcbb2ff6ec37837abf0de0b8b201720f6dcb8900e8a42441566f&scene=21#wechat_redirect)\
2. [可视化流量录制规则探索和实践｜得物技术](http://mp.weixin.qq.com/s?__biz=MzkxNTE3ODU0NA==&mid=2247525938&idx=1&sn=f72b5d3399c0abfbefc4a66367b849a9&chksm=c161336df616ba7bb5131cd3c3cbf2ffbd8b544e56c14cac9ad841f5b9054062c1f95c35b782&scene=21#wechat_redirect)\
3. [客服测试流水线编排设计思路和准入准出应用｜得物技术](http://mp.weixin.qq.com/s?__biz=MzkxNTE3ODU0NA==&mid=2247525842&idx=1&sn=8ef60b41b54efe9db929266934a7905d&chksm=c161308df616b99ba004dadd99239b94f163d7f51d5082fae34ca429ea68da56b35bc60260bb&scene=21#wechat_redirect)\
4. [深入剖析时序Prophet模型：工作原理与源码解析｜得物技术](http://mp.weixin.qq.com/s?__biz=MzkxNTE3ODU0NA==&mid=2247525652&idx=1&sn=9ad2289419f427b572cef7623dc11ab3&chksm=c161304bf616b95d181e3c40e29a05ee7491f6f8f92fbc3365eb137d838a1d9d2c4eda836860&scene=21#wechat_redirect)\
5. [在得物的小程序生态实践](http://mp.weixin.qq.com/s?__biz=MzkxNTE3ODU0NA==&mid=2247525886&idx=1&sn=2dbea387abfb6c9e77dcd17a75cc5623&chksm=c16130a1f616b9b7d9afb7d55e9ff1d88f40d1dfd9b3bc3be8e0dcbb0b053021b279271f3e49&scene=21#wechat_redirect)

文 / 天佑

关注得物技术，每周一、三、五更新技术干货

要是觉得文章对你有帮助的话，欢迎评论转发点赞～

未经得物技术许可严禁转载，否则依法追究法律责任。

“

**扫码添加小助手微信**

如有任何疑问，或想要了解更多技术资讯，请添加小助手微信：

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**线下活动推荐**

**快快点击下方图片报名吧！**

[!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)](https://mp.weixin.qq.com/s?__biz=MzkxNTE3ODU0NA==&mid=2247527397&idx=1&sn=b504919a60da99052d1bd6cd26c9be96&chksm=c1613ebaf616b7ac41c12de414621bfe6fb413f254c9af864cc637c7b790977668ad3aad68c1&mpshare=1&scene=24&srcid=0703zmGnHjD1tNwzQXceeRaK&sharer_shareinfo=b164dcb13bdae4923de7d282ef816f75&sharer_shareinfo_first=b164dcb13bdae4923de7d282ef816f75&key=daf9bdc5abc4e8d038b457ebc0bcbf6784267024406a167c57317c82a531d5ee520137ff28bce7e994ac32738eeee8e230d206fb3b098d4089eb3c71b19f2528eceb70d2cb9146a4d3bd3edad5a431bf050ec49a6d43b2dd5d7b4b7722c22dcad9cf8d6938df549271a32446693615bcdf7a379581fdf872986e1f6ae08b43e4&ascene=0&uin=MTEwNTU1MjgwMw%3D%3D&devicetype=Windows+11+x64&version=63090621&lang=zh_CN&countrycode=CN&exportkey=n_ChQIAhIQr%2FxuytkBQoHFLl01eBdN%2BRLmAQIE97dBBAEAAAAAAKyrF4%2FG1KgAAAAOpnltbLcz9gKNyK89dVj0wtVcIgVYFeiPEpnju%2FGrL5yiZnqPwh9zZd1eXVJJEJ7XQ9JACffrUdVbXGl7JokTqvJMYkDQos4OpIcMnQhIU5INsNjYCLdOnMjCNvBjlptVjwehtawI6ghFH8pVWKXazs2Wb3%2BkN2YUX%2FTapX6lNPrP5ViB1%2BXIGVzsE0hqPKzUCR0AK6I4LWCEC16tFirtlml8uKfOP7rGt1jE26pM5T26vgv6PMSZRGcimPWk2itVM89HihbIf6PXLmV3HYjs&acctmode=0&pass_ticket=gB%2B5NulOcFdFRW8bJPVNgapKpdwjRF9VoNI5XG299%2BzkSBOLAddw7b4ygeY3IEDJ&wx_header=1&fasttmpl_type=0&fasttmpl_fullversion=7350504-zh_CN-zip&fasttmpl_flag=1)

Java18

disruptor1

高性能4

Java · 目录

上一篇浅析Spring中Async注解底层异步线程池原理｜得物技术下一篇探索BPMN—工作流技术的理论与实践｜得物技术

阅读 9779

​

写留言

**留言 13**

- Jungle

  四川7月3日

  赞3

  这个框架不好，一旦有一个消费者阻塞，最后导致生产阻塞

  得物技术

  作者7月4日

  赞1

  框架还是需要结合自己的实际场景去考虑。

- 杨建®

  广东7月4日

  赞

  多消费者的情况下 一个消费者阻塞 会影响整个链路数据发送吧

  得物技术

  作者7月4日

  赞1

  支持多个消费者组，消费者组之间完全隔离，互不影响。

- 影子

  上海7月8日

  赞

  有几个问题请教: 1、消费者应该异步线程处理吧。 2、消费者处理失败，可以通知生产者么？ 3、可以结合数据库当成简易 MQ 使用么？![[捂脸]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

  得物技术

  作者7月8日

  赞

  1. 是的。 2. 生产者可以作为一个bean注册到容器中或者自己维护一个HashMap保存该对象，消费者可以找到对应的生产者重新投递。 3. 可以。

- 狂妄之徒

  四川7月3日

  赞

  按照我的理解，这个框架不适合穿插io操作的场景，即使框架再快，io拖慢了，最终还是io的瓶颈

  得物技术

  作者7月4日

  赞

  撮合应用可以理解为纯内存的操作，io的问题可以结合线程池+异步化去处理。

- Regulus

  上海7月3日

  赞

  这配置单机tps两百万？？？

  得物技术

  作者7月4日

  赞

  可以参考官网数据，纯内存操作200w没什么压力。

- Bob

  上海7月3日

  赞

  这个框架真是出来这么多年了，却很少有人知道

  ICD

  陕西8月11日

  赞

  没那么多公司有撮合的业务

  1条回复

已无更多数据

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/AAQtmjCc74AlsHDtoVyU8hqzNTGS26fV9PmHAcZ8uib1GWNJibIuBiavPdAXw9IOzjlEAYRJUNjOEme5geMNPoZ1Q/300?wx_fmt=png&wxfrom=18)

得物技术

12790158

13

写留言

**留言 13**

- Jungle

  四川7月3日

  赞3

  这个框架不好，一旦有一个消费者阻塞，最后导致生产阻塞

  得物技术

  作者7月4日

  赞1

  框架还是需要结合自己的实际场景去考虑。

- 杨建®

  广东7月4日

  赞

  多消费者的情况下 一个消费者阻塞 会影响整个链路数据发送吧

  得物技术

  作者7月4日

  赞1

  支持多个消费者组，消费者组之间完全隔离，互不影响。

- 影子

  上海7月8日

  赞

  有几个问题请教: 1、消费者应该异步线程处理吧。 2、消费者处理失败，可以通知生产者么？ 3、可以结合数据库当成简易 MQ 使用么？![[捂脸]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

  得物技术

  作者7月8日

  赞

  1. 是的。 2. 生产者可以作为一个bean注册到容器中或者自己维护一个HashMap保存该对象，消费者可以找到对应的生产者重新投递。 3. 可以。

- 狂妄之徒

  四川7月3日

  赞

  按照我的理解，这个框架不适合穿插io操作的场景，即使框架再快，io拖慢了，最终还是io的瓶颈

  得物技术

  作者7月4日

  赞

  撮合应用可以理解为纯内存的操作，io的问题可以结合线程池+异步化去处理。

- Regulus

  上海7月3日

  赞

  这配置单机tps两百万？？？

  得物技术

  作者7月4日

  赞

  可以参考官网数据，纯内存操作200w没什么压力。

- Bob

  上海7月3日

  赞

  这个框架真是出来这么多年了，却很少有人知道

  ICD

  陕西8月11日

  赞

  没那么多公司有撮合的业务

  1条回复

已无更多数据
