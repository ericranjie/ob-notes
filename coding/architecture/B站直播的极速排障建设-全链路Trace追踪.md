# 

Original 大前端 哔哩哔哩技术

_2024年09月06日 12:02_ _上海_

\*\*一、\*\***概述**

直播业务具有实时性强，复杂度高，排查链路长，影响面大等特征，线上问题如果不能立刻排查处理，分分秒秒都在影响用户的观看体验、主播的收入。

但各端的问题可能都只是表象，例如，一个看似简单的画面卡顿问题，可能涉及到编码器配置、网络带宽分配、服务器负载等多个方面，各个团队经常在等待合作方的反馈，一整套流程下来，一个线上问题的定位可能要消耗掉数小时的人力。

**我们迫切的需要一套高效的跨端实时排障系统！**

![Image](https://mmbiz.qpic.cn/mmbiz_png/1BMf5Ir754TlMExITSehgIkOqASRby8IshxZlE3TkR2reEkxL7O52EgGXR6lqxq1WCB9j31qcw5gyevWibibRlqQ/640?wx_fmt=png&from=appmsg&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

**为此我们采取了以下措施：**

1. \*\*关键业务监控：\*\*联合各协作方，对关键业务的接口、广播和核心处理逻辑实施了实时埋点监控，并附加了相关场景信息，确保了问题定位的准确性和全面性。

1. \*\*统一追踪系统：\*\*为了实现单个业务链路所有埋点的跨端联络，我们设计了统一的trace_id字段，并在数据层进行串联，通过看板直观展示，极大地提升了问题追踪和定位的效率。

**这些措施带来了显著的成效：**

- \*\*跨部门协作效率提升：\*\*通过实时数据共享和统一追踪系统，直播移动端、PC端、Web端、服务端以及流媒体等各个团队协作效率大幅提升。在开播、视频连线等9个核心业务的故障排查中，排障率达到了91%，异常定位的平均时间从2小时缩短至仅需5分钟。兄弟部门也采纳了我们的方案，有效减轻了工作压力。

- \*\*系统稳定性增强：\*\*这些措施还帮助我们优化了开播异常断流、连麦发布订阅失败等多项关键业务问题，确保了系统的高效运行，减少了因技术问题导致的用户流失。

- \*\*用户体验改善：\*\*我们的快速响应和问题解决能力极大地提升了主播和用户的直播体验。用户和主播的正面反馈络绎不绝，间接提高了主播收入稳定性，增强了平台的吸引力。

\*\*二、\*\***技术方案详解**

**1. 方案设计**

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

如上图所示，整体全链路排障建设可以分为数据采集、数据处理&存储、可视化工具建设3大块。

在开始介绍实现方案之前，需要简单介绍一下OpenTracing，它是业内实现分布式链路追踪系统通常会采用的方案，我们在后续的埋点和上报组件设计也对它进行了一些参考。OpenTracing定义了追踪数据所需要的操作和数据结构，帮助开发人员实现分布式追踪的能力。OpenTracing里面有两个比较核心的概念，简单说明一下：

Trace：Trace代表一条追踪路径，它由多个Span组成，存在一个唯一ID

Span：Span代表追踪路径中的一个时间跨度，包括操作名称、开始时间、结束时间等信息，由SpanID作为标识。由多个 Span 可以形成一条追踪路径。Span还定义了父子、跟随两种关系。在Span上下文中，记录和维护了Trace的ID和当前Span的ID。

下图是OpenTracing的模型图，它描述了由多个Span组成一条追踪路径：

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

OpenTracing在服务端得到了广泛的使用，但是面临客户端业务现状和问题，我们调整了最终方案的实现方式

1. 直播场景用户行为一般都是即时操作，时间片段的设计并太不合适

1. OpenTracing是跨编程语言的标准，一些API的设计比较抽象，在业务中使用不友好

考虑到这些，我们借鉴了OpenTracing中核心的概念：trace_id和事件上下文，并简化了OpenTracing中Span的概念，尝试复用端上已有的埋点，扩展字段来实现全链路Trace的能力。下面我们开始介绍。

首先要确认的是必须上报的埋点字段。为了减轻理解和上报的成本，在设计上我们希望尽可能简单。实际上，这些字段的设计也是为了解决几个关键的问题：

**1.1 如何将各端的埋点关联起来？**

\*\*trace_id：\*\*一个复杂的事件链路往往并不是单端闭环的。拿邀请上麦举例，它涉及到主播客户端A -> 业务服务 -> 广播服务 -> 观众客户端B -> RTC。我们希望将这一次事件链路中相关的日志都能聚合起来呈现，而不是各端查各端的。为了解决这个问题，我们生成一个全链路都会透传的唯一ID，在每个端的上报中都会携带这个ID，然后通过这个ID，把这次事件关联的上报检索出来，一起展示。

**1.2 如何解决日志中缺失的上下文？**

\*\*extends：\*\*我们在上报中增加了扩展数据，用于携带上下文以及自己关心的信息，同时在可视化工具中展示出来。

**1.3 如何快速的找到异常的环节？**

\*\*level：\*\*我们给每一个上报定义了3种状态，正常、警告和异常。在可视化工具中，针对警告和异常状态的上报，用黄色和红色展示出来，这样可以第一时间定位到出现问题的地方，找到负责的端和同学。

**1.4 如何衡量这次的事件是否正常？**

\*\*type：\*\*我们给每一个上报节点定义了3种类型，起始、过程和结束。一次事件执行中，会有一个起始节点，一个结束节点和多个过程节点。如果这次事件执行链路里面，有结束节点，并且所有节点的状态都是正常的，那我们就认为这次事件执行是正常，否则就是异常的

到这里，最主要的埋点字段就介绍完了，接下来就只需要各端在关键路径上添加上报即可。

在上线验证阶段，移动端先通过透传trace_id的方式快速上线并打通了整条链路，验证了可行性。但是这种方式弊端很大，代码入侵严重而且健壮性差，对于业务同学来说这是非常劝退的，所以我们针对上报组件做了一些设计，目的是降低接入和维护成本，减少代码入侵。

**2. 上报组件设计**

上报组件随项目发展共迭代了三个版本，每一版都比上一版更加易用和完善。下面介绍我们的迭代过程，共分为“快速验证可行性”、“大幅提升易用性” 和 “继续增强鲁棒性”。

**2.1 快速验证可行性**

在项目初期，为了快速验证链路可行性，上报组件未做过多设计，仅实现了最基础的功能：

将上述的基础字段（trace_id, level, type等）、业务方自定义参数以及公共参数（房间信息、网络、推流、设备、外设情况、线程id等）进行上报。

在最初版中，我们将所有的非公共参数都写到了函数入参中，并在业务层透传了trace_id。如图所示：

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**2.2 大幅提升易用性**

**2.2.1 快速方案遇到的问题**

基础功能上线后，验证了我们通过trace_id串联起多个埋点的想法是可行的。随着越来越多的业务接入，显而易见的两个问题便浮上水面：

1. \*\*上报代码过于繁杂：\*\*由于需要8个参数，需要多行代码才能完成一次上报，在业务代码中插入这一块又一块的和业务无关的代码，会严重降低可读性和可维护性；

1. \*\*业务入侵性大：\*\*在低耦合的代码架构下，一个功能点的实现经常横跨1～3个模块、纵深5～10层方法调用，想要做到精确的全链路追踪，势必要将trace_id透传，这样就需要在每个方法的入参都增加一个trace_id的参数，不仅写起来麻烦，还对业务的入侵性巨大；

这里举两个实际的代码例子：

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

接下来我们就这两个问题对埋点组件进行优化：

**2.2.2 解决埋点代码冗长**

需要在业务层和上报层中间插入一个埋点聚合层，负责组装参数，并针对每一个节点向外提供一个简明的方法，在业务层就只需要一行简短的代码就可以完成上报了。

在埋点聚合层中，我们也做了一些简单的设计，旨在减少业务方的代码量：

1. 尽力减少上报方法传参的数量，将需要的参数都封装进一个事件模型类中，并针对起始节点提供便利构造方法。且将入参node_type、trace_id、level、extends字段加上默认值，这样对于大部分的节点，就不再需要携带所有的参数了。

1. 针对每个业务类型都额外做了一层封装，这样就不用每次都填写event_type字段了，进一步的减少了代码量。至此，对于普通节点，甚至只需要指定key和log两个字段就可以完成上报了。

下面是解决第一个问题（埋点代码冗长）的简单图示：

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**2.2.3 解决业务入侵性**

业内流行使用插桩的方式来进行非入侵式的埋点，但切面的形式很难获取业务上下文，无法解决方法A调用方法B的trace_id透传问题，因此并不适用于这个场景。

对于trace_id透传的问题，解决方案是把trace_id缓存一下。但是需要解决以下场景的问题：

1. 多线程并发：并行启动了多次同一个事件，且他们的完成时间也不固定，如同时上传了多张大小不一的封面。

1. 事件中断：前一次事件因为某些原因中断了，永远的停留在了某个节点。

为此，我们首先引入状态机的概念，将所有节点使用有向图进行表示，这样我们便能清晰的感知到事件的发生到结束，以及某个节点后续可以流转至哪些节点、是否发生错误中断。

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

下面举一个具体的例子，上麦流程图和其对应的有向图：

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

图中使用了虚实线来区分跨端或者跨线程的动作，其必要性可参见下文第(2)点。

**（1）自动化寻找trace_id**

当一个起始节点准备上报时：

1. 为其创建一个上报实例

1. 在实例中记录trace_id、当前的节点，以及它之后可能会流转到的节点

1. 将这个实例扔到池子中

当一个非起始节点准备上报时：

1. 组件会根据有向图去池子中查找需要流转到的节点

1. 使用实例的trace_id进行上报

1. 更新实例的时间戳

1. 将实例流转到下一个节点，若无后续节点，则移除实例

**（2）解决多线程问题**

这个方案似乎很完美，但从B到E是一个网络接口请求，如果先后很快地发出了两次请求，很可能会出现下面的情况：

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

如果遵照上述的方案，B和E之间就会被错误的关联起来。为了规避这个问题，trace_id会在跨线程时从埋点组件中外抛，需要业务方短暂记录，并传递到下一个节点。当然，针对常用的网络请求，我们也做了易用的封装，详见2.3.1。

于是之前上麦的有向图会变为：

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

在多线程问题解决之后，当一个含有trace_id的节点进入上报组件时，即会为其创建一个上报实例，按上文记录字段后放到池子里。

至于无trace_id的上报则完全相同，唯一的区别就是在查找节点时，加上线程id的校验，这样可以防止同一事件在不同的线程中同时启动。

在trace_id已经被自动化后，整体的上报流程图如下：

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**2.2.4 提升易用性后的代码架构**

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**2.3 继续增强鲁棒性**

**2.3.1 对“接口与广播”的封装**

在端上遇到的跨线程/跨端场景绝大多数都是网络请求和广播，因此，为了避免出错和降低复杂度，我们做了一套易用的封装：

1.  网络请求

（1）组件会在发起指定请求之前通过有向图自动寻路获取此次trace_id

（2）使用该trace_id上报请求事件，并会自动带上所有的业务请求参数

（3）将trace_id置于请求头，用于串联服务端节点

（4）在接口返回后，自动上报响应数据，若接口错误，会将该节点标记为error

2.  广播

（1）组件会在指定广播到达时，尝试去获取trace_id字段。

（2）若获取成功，则自动进行上报，若没有，则会走自动化上报流程。

这样，业务方即使遇到跨线程/跨端，也无需关心trace_id了，在这两种场景下彻底做到了业务无感。

**2.3.2 对“抗风险能力”的补足**

当业务链路与对应的有向图不符时，trace_id的自动化管理便会失效。为此我们设计了特殊异常case的监控：

1.  事件跟踪上下文丢失

自动化寻找trace_id失败，此种情况多发生在上报的埋点与有向图描述不符，此时埋点组件会上报一个警告埋点触达开发及时修改链路。

2.  事件跟踪超时

即上次单线程的流转还未结束，新的节点就已到达。此种情况多发生在流转过程被意外打断，如check失败后直接return。此时原事件在看板中会表现为链路中断的错误。同样的，会上报一个警告埋点。

**2.4 上报组件整体概览**

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**3. 数据处理和存储**

在处理上报的海量数据时，需要清洗掉错误的数据，并解析各个终端上报的不同数据结构，转化为统一数据模型。由于数据是逐条上报的，必须将这些离散数据串联成完整的事件链路，这样就知道用户操作了什么、经历了哪些端、哪些节点出了问题或漏了哪些节点。

**3.1 事件串联**

（1）单trace_id串联：事件由唯一的trace_id串联整个流程。

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

（2）多trace_id串联：事件由多端各自的trace_id组合而成。

为兼容业务服务和广播服务各自独立的trace追踪系统，我们实现了一套多重映射算法，且无缝兼容了单trace_id方案，最终溯源成事件开始的原始trace_id。

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**3.2 数据清洗与存储**

为应对直播的实时性要求，我们采用流计算技术。先快速筛选出trace相关事件，再清洗掉异常数据，在单次流计算执行过程中进行映射建立关系并落表存储，实现小于5min级别的数据响应处理。数据表支持灵活的定制化查询和分析需求。

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**4. 数据可视化**

可视化简化了查询过程，能快速准确地捕捉异常和关键信息。适用于开发、测试、产品、运营、客服等角色。

**4.1 覆盖场景**

从App启动到退出，从开播到关播，从上麦到下麦，从PK发起到结束等关键业务场景。

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**4.2 落地效果**

在日常业务中，已经有效解决了很多实际问题，以下是遇到的一些案例查询：

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**三、结束语**

不知何时起，“用关键链路查一下”已经成了身边同事常说的口头禅，整套排障体系的价值，也得到了验证。

未来还有许多要做的事，我们将致力于拓展业务覆盖面、建立业务健康度监控体系、提升上下文信息的有效性等。

我们相信，通过不断的技术创新和服务优化，我们的业务能够迎接更大的挑战，为用户创造更大的价值。

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

-End-

作者丨瞳舞舞、小猪、钱来、星空凛、lish、志超

**开发者问答**

**大家线上问题定位都需要多久呢？为了高效排障又做了哪些努力？**欢迎在留言区告诉我们。转发并留言，小编将选取1则最有价值的评论，送出**哔哩哔哩教师节钢笔1支**（见下图）。**9月10日中午12点开奖。如果喜欢本期内容的话，欢迎点个“在看”吧！**

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**往期精彩指路**

- [千万长连消息系统](http://mp.weixin.qq.com/s?__biz=Mzg3Njc0NTgwMg==&mid=2247498952&idx=1&sn=61316837cd685e4d20cf6931858c31a9&chksm=cf2f39edf858b0fb5b60c44a291684541af59dbf612354fcdd72c94633904f417f2256a21197&scene=21#wechat_redirect)

- [全链路压测改造之全链自动化测试实践](http://mp.weixin.qq.com/s?__biz=Mzg3Njc0NTgwMg==&mid=2247487748&idx=1&sn=c9cbcacf3bba25b478abf2a0f5c0e75f&chksm=cf2cd421f85b5d37adce8fc782151942912bf38a24be2ebd42b000505bcae60a4833d9a230cd&scene=21#wechat_redirect)

- [哔哩哔哩直播通用榜单系统](http://mp.weixin.qq.com/s?__biz=Mzg3Njc0NTgwMg==&mid=2247499176&idx=1&sn=cb639b1d8de4ebec556e2b17cc6ad842&chksm=cf2f388df858b19b364c782ad5439ee12f2db752c34acfc958d6e36e45ad8ca28c97c94c8a22&scene=21#wechat_redirect)

[通用工程](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=Mzg3Njc0NTgwMg==&action=getalbum&album_id=3289447926347317252#wechat_redirect)丨[大前端](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=Mzg3Njc0NTgwMg==&action=getalbum&album_id=2390333109742534656#wechat_redirect)丨[业务线](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=Mzg3Njc0NTgwMg==&action=getalbum&album_id=3297757408550699008#wechat_redirect)

[大数据](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=Mzg3Njc0NTgwMg==&action=getalbum&album_id=2329861166598127619#wechat_redirect)丨[AI](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=Mzg3Njc0NTgwMg==&action=getalbum&album_id=2782124818895699969#wechat_redirect)丨[多媒体](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=Mzg3Njc0NTgwMg==&action=getalbum&album_id=2532608330440081409#wechat_redirect)

![](http://mmbiz.qpic.cn/mmbiz_png/1BMf5Ir754SVM7VMibhzz9AicOz5qdQ5oaN6TgXhbVyGHQYPwBErEYQCOnhibA1Qp431JkpPKY0qGhT0EfDchicXMQ/300?wx_fmt=png&wxfrom=19)

**哔哩哔哩技术**

提供B站相关技术的介绍和讲解

272篇原创内容

公众号

![](http://mmbiz.qpic.cn/mmbiz_png/EVKwaZXNTl9OCCo7pxLHz2e2I3kV3rTPao5LlIickfJS79DNd2yjqjfYEtwtMOyVuKhJoDIq6UU4U9TQbjvOLaQ/300?wx_fmt=png&wxfrom=19)

**哔哩哔哩招聘**

生产快乐的地方

22篇原创内容

公众号

大前端50

数据可视化2

问题排障1

用户体验1

大前端 · 目录

上一篇文档画中画之跨页面播放队列

Reads 3347

​

Comment

**留言 7**

- 😀

  江苏Yesterday

  Like9

  b站太优秀了，啥方案都往外讲![[旺柴]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)

- PyKt

  广东Yesterday

  Like2

  很好奇为啥不直接使用 OpenTelemetry 这类成熟的方案来进行可观测性的建设，而是基于 OpenTracing 思路自己做的一套基于事件的链路追踪管理呢

  H.five

  北京Yesterday

  Like4

  kpi

  1条回复

- 那时刻

  北京Yesterday

  Like3

  我们排查线上问题的时候，对于刚发布的情况，定位比较快，针对版本的改动排查即可。而对于偶发的线上问题，需要漫长的时间去追踪。为了高效的排查问题，需要埋点的方式来协助追踪。文中提到的trace id值得借鉴。我有以下几个问题： 1. Span的跟随关系怎么理解，它与父子关系的区别是什么？ 2. 事件中断：前一次事件因为某些原因中断了，永远的停留在了某个节点。对于中断的节点什么移除呢？ 3. 多trace_id串联多重映射算法能够详细说下吗？

- Tian

  上海Yesterday

  Like3

  太好了，受益匪浅，像大佬们学习🌟

- 散木

  浙江Yesterday

  Like3

  相比于技术方案，更好奇你们如何做的技术选型，opentracing/opentelemetry/skywalking等开源框架

已无更多数据

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/1BMf5Ir754SVM7VMibhzz9AicOz5qdQ5oaN6TgXhbVyGHQYPwBErEYQCOnhibA1Qp431JkpPKY0qGhT0EfDchicXMQ/300?wx_fmt=png&wxfrom=18)

哔哩哔哩技术

4523217

7

Comment

**留言 7**

- 😀

  江苏Yesterday

  Like9

  b站太优秀了，啥方案都往外讲![[旺柴]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)

- PyKt

  广东Yesterday

  Like2

  很好奇为啥不直接使用 OpenTelemetry 这类成熟的方案来进行可观测性的建设，而是基于 OpenTracing 思路自己做的一套基于事件的链路追踪管理呢

  H.five

  北京Yesterday

  Like4

  kpi

  1条回复

- 那时刻

  北京Yesterday

  Like3

  我们排查线上问题的时候，对于刚发布的情况，定位比较快，针对版本的改动排查即可。而对于偶发的线上问题，需要漫长的时间去追踪。为了高效的排查问题，需要埋点的方式来协助追踪。文中提到的trace id值得借鉴。我有以下几个问题： 1. Span的跟随关系怎么理解，它与父子关系的区别是什么？ 2. 事件中断：前一次事件因为某些原因中断了，永远的停留在了某个节点。对于中断的节点什么移除呢？ 3. 多trace_id串联多重映射算法能够详细说下吗？

- Tian

  上海Yesterday

  Like3

  太好了，受益匪浅，像大佬们学习🌟

- 散木

  浙江Yesterday

  Like3

  相比于技术方案，更好奇你们如何做的技术选型，opentracing/opentelemetry/skywalking等开源框架

已无更多数据
