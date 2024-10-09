原创 Ryan OPPO内核工匠

_2024年02月23日 17:30_ _广东_

在系统工程师的日常工作中，最苦恼的事情之一就是分析问题所依赖的可观测性数据出现了错误。“这该死的玩意儿又出错了！” 在面对新工具出现的新问题时，工程师们在愤懑之余免不了怀念旧时的荣光：那时的调试工具设计精巧，API 简明易用，如老伙计般地可靠。

然而随着新系统、新编程语言和新编程框架的不断发展，可观测性工具也在不断地推陈出新，"good old days" 早已一去不复返了。可观测性领域的技术虽并未产生大的革新，但是工程师们在观测数据的采集方式和分析方式上做了大量的工作，Perfetto 就是 Android 领域的后起之秀之一。

Perfetto 为自己标榜了开源、稳定且高效的跨领域系统跟踪和分析平台这一头衔，也为其自身的架构设计指出了明确的目标。在这篇文章我们暂且放下“观测工具”们自身的发展历史不谈，谈谈其数据编码与传输（后文简称 Data Flow）的架构设计，并从这个角度解释其可能存在数据丢失的诸多原因，并提出相应的解决（或者规避）建议。

Note：对 Perfetto 架构不感兴趣的伙伴可以直接跳转至 PART 4 章节，获得减少 Perfetto 使用故障的具体建议。

## **前言 - 数据传输系统的设计理念**

如何将数据从一侧搬至另一侧的基本设计理念从来都不是秘密，这就如同我们在现实生活中订购产品送到家的过程。

通常我们在使用这类服务的时候只需要考虑 3 方面的因素：

- WHAT：订阅什么产品

- WHEN：什么时候收到

- WHERE：在什么地点收货

除此之外的诸多细节我们统统都不关心，我们提供了必要的信息来描述需求，供应商、物流公司设法为我们解决过程中需要处理的诸多麻烦事儿。

大多数时候机制都是运转良好的：商品总是供应充足，快递总能按时到家，很难遇到意外的情况发生，我们不需要操心整个过程是如何完成的。但是当错误的事情开始发生时（例如产品没有送到家，或是收到了错误的产品），我们就会陷入一种沮丧的情绪中：我们知道有问题，但是不知道该找谁的麻烦。这种沮丧的情绪就如同工程师遇到了不太合作的系统可观测工具，此时人类的情感是共同的。

为什么 trace 会漏掉了我们关注的时间发生的系统状态？为什么 Perfetto UI 上的 trace event 会层层叠叠地延伸至屏幕的两端？造成这一切的根源都来自于不同程度的“权衡利弊”。

毕竟可使用的资源总归是有限的，而在系统里有不止一个生产者、物流和消费者，任何一方都有可能成为瓶颈。“物流公司”的运输可能会延迟后丢失，“产品供应商”也可能会爆单，来自消费者的需求也可能不足。只要我们设法触达了这套系统的中的诸多瓶颈，那么权衡利弊的策略就会在各个环节发生。

!\[\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

在生产者 & 运输者 & 消费者 组成的一个供销系统下，每一方都会为了自身效率的最大化而添加很多非必要的中间环节。例如：物流公司可能会为货物增设中转站，以提前备货来缓解消费者的需求旺盛使得某些货品的物流压力骤增。

## **PART 1 - 基本理念：生产者，消费者，以及 IPC 通讯**

现在让我们将目光拉回到 Perfetto 本身。我们已经知道了 Perfetto 希望成为一个拥抱开源、支持跨平台跨领域的系统跟踪和分析框架，那么在这个框架下生产者、消费者和 “物流公司” 分别有哪些？从官方提供的示意图中我们可以窥知一二：

!\[\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

- 数据生产者：如图绿色框所示。生产者可以是多个进程，且每个进程可以同时供给不同的数据类型。

- 数据消费者：如图黄色框所示。在一个数据跟踪会话中只存在一个消费者进程。对于 Perfetto 来说，Traced 只能算是一个代理消费者。代理消费者仅仅将数据放置在自己的 Buffer 中，并可以与真正的消费者协商数据的处理方式：是定期读取 buffer 数据做持久化存储？还是将数据重定向至新的 IPC 通道？

- 数据传输：如图蓝色框所示。数据传输分为信号传递和数据交换两个过程，发生在生产者进程和消费者进程之间。

Perfetto 的跨平台数据跟踪能力，是以 Data source 的 ABI/API 协议来定义的。无论是 Chrome 浏览器内核，还是 Android 或 Chrome OS 操作系统，都可以在遵循这套协议的基础上将自己注册为 Perfetto 的数据生产者。

以 Linux ftrace 为例：Perfetto 在 Android 操作系统中如何将 ftrace 作为其数据源？从上图灰色框部分可以看到 ftrace 在每个 CPU core 中已有对应的 ring buffer，在兼容已有 ftrace 框架的基础上，Android 系统启动了 traced_probes 进程来定期读取 ftrace buffer 数据并将其序列化为 perfetto 支持的二进制格式。由于 traced_probes 在启动阶段已经注册为了 Perfetto 的 Data source，因此消费端在启动一个跟踪会话时只需要告知 Perfetto 订阅 linux.ftrace 这个数据源即可。

## **PART 2 - 权衡利弊：观测开销 vs 传输可靠性**

可观测工具的使用是有成本的。如下表展示了在前台随机启动应用 60 秒的过程中，相关可观测工具进程的 CPU task 运行时间的统计：

|process_name|pid|uid|cpu_time_ms|cpu_time_perccent|
|---|---|---|---|---|
|/system/bin/traced_probes|2376|9999|11013.04|2.294383|
|/system/bin/logd|1034|1036|6801.802|1.417042|
|logcat|3756|0|4545.653|0.947011|
|/system/bin/traced|2386|9999|2064.368|0.430077|
|/apex/com.android.os.statsd/bin/statsd|1506|1066|1124.588|0.234289|
|logcat|7330|\[NULL\]|11.8526|0.002469|

从上表数据可知，无论是 trace 相关的进程（tracd, tracd_probes）还是 log 的进程（logd, logcat）或是 metrics 采集进程（statsd）都会引入一定的性能开销。随着需要记录或存储的数据流量越来越大，工具本身可能引入的开销也逐渐膨胀。

我们希望观测工具所带来的 “观察者效应” 能够尽可能地消除。除了约束数据的生产者生产数据的速度，观测工具还会绞劲脑汁地优化观测数据的 data flow 中所有可能引入系统负载的代码流程。

Perfetto 采用了共享内存 Buffer 的方式来减少跨进程的数据拷贝所带来开销，并采用 buffer 的分区写入方案实现了生产者在并行生产数据时的数据同步开销。

#### **Central Buffer 映射**

如 PART 1 所述，Perfetto 为 trace 会话设置了一个代理的数据消费者，这个代理消费者位于 Traced 进程内，负责将生产者产生的数据拷贝至独立的 Central Buffer 中。Central Buffer 需要设置合适的大小，以缓冲特定生产者发送的数据包。

buffer 与 生产者可以建立明确的映射关系：               \
!\[\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

权衡利弊 1：生产者生产数据的速度是有差异的，因此依照数据生产的吞吐率来为其映射合适的 Central Buffer 可以保证 buffer 的填充速率相对一致：生产较快的数据源使用更大的 Central Buffer，而生产较慢的数据源应当映射到一个更小的 Central Buffer。同时对于更 “重要” 的生产者生产的数据，最好也为其设置独立的 Central Buffer，以免受到其它生产者的干扰。

生产速度不一致的数据生产者如果不进行 buffer 隔离，可能会出现互相挤兑现象。假使我们仅仅分配一个 Buffer 供 A，B，C 三个生产者共用，在预期的状态下，Buffer 按照 RING BUFFER 模式进行数据的存储和覆写，如下图所示：               \
!\[\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)                \
此时三位生产者生产数据的频率和数据的大小近似，在 Buffer Size 的限制窗口内我们可以观测到来自三个生产者的数据。               \
当生产者 A 的数据包大小或生产速度突然加快时，RING BUFFER 模式将会挤占 buffer 中记录的来自其它生产者的数据包并使其快速丢失，如下图：               \
!\[\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)                \
当新的数据包写入 Buffer 后，来自生产者 B 的数据将失效并无法被观测到。

#### **Shared Memory Buffer 的数据搬运（读写）**

Shared Memory Buffer 与 IPC 通道是 Perfetto 中数据搬运的实现基础，生产者与代理消费者遵循相应的协议有序地使用 Shared Memory Buffer 来完成高效的数据传输过程。下面这张图展示了其中的一些细节：               \
!\[\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

Shared Memory Buffer 并不是整块地被使用，而是被划分为不同的区域，我们暂且将每个区域称作一个 Page，Page 是进程间共享数据的最小单元。而 Page 在进程内部还会被划分为以 chunk 为最小单元的 buffer 块。每个 chunk 内的数据写入是 lock free 的，这意味着 chunk 内的数据写入是顺序的，与生产数据的线程相映射。

Chunk 的状态可以划分为 3 个阶段，每个阶段都只能被一个独立的实体来访问。简言之，chunk 是生产者与消费者之间交互的最小粒度，对每个 chunk 的访问必须是独占的。Chunk 的状态切换如下图：

!\[\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

- Free: Chunk 是空闲的，消费者不会去拷贝处于 Free 状态的 chunk，而生产者则需要申请该状态的 Chunk 来写入数据；

- BeingWritten: 此时 Chunk 正在被数据生产者使用，数据的写入正在进行并且还未完成。注：即使此时 chunk 还未填满，消费者仍然可能在结束 Trace 会话时拷贝其中的数据。

- BeingRead: Chunk 已经写满了数据，此时生产者不能再去修改其中的数据，生产者已经开始拷贝其中的数据。当数据拷贝完成后，Chunk 将被重置为 Free 状态。

权衡利弊 2：在 Producer 与 Consumer 之间共享的 Page 大小和数量不可能是无限的。在设定 Shared Memory Buffer 的总大小后，Page 的大小和 chunk 的大小需要在多个因素之间权衡利弊。

1. Page size 越大，则 Page 交换的 IPC 信号就发送得更不频繁，反之亦然；

1. Page size 和 chunk size 越大，chunk 的过程就更不经常发生，而 chunk 的状态切换需要申请同步锁；

1. Page 或 chunk 的 size 越大，则数据更不易被填满，处于 BeingWritten 状态的 chunk 数量越来越多而 Free 状态的 chunk 不足甚至降为 0，这可能使得数据生产者无法获得可用 chunk 来写入数据；

通过上文的分析，我们知道可以通过设定更大的 Shared Memory Buffer 和更大的 Central Buffer 来提高数据传输的可靠性，而这将以更大的内存空间使用为代价。Perfetto 依据生产者的生产速度与消费者的搬运速度为 Buffer 的大小设置了相应的经验值，这可能在 Pixel 的机器上运转良好，可以在可控的观测开销下获得可靠的数据传输能力，然而在其它的设备上可能并不能很好地工作。

## **PART 3 - 数据编码协议中的权衡利弊**

下面我们来谈谈 Perfetto 中的数据编码协议。Perfetto 采用了 protobuf 对 trace 的数据进行序列化，且煞费苦心地为其专门开发了 ProtoZero 库来提高 protobuf 的序列化性能以降低 perfetto trace 数据的实时序列化开销。

protobuf 存在很多优良的特性，包括空间友好的可变长编码，可通过 .proto 文件预定义的数据结构等等。这些内容与本文的主题无关，因此不再展开。我们重点谈谈 Perfetto trace 中的原子数据结构：TracePacket 以及其中存在的权衡利弊。

#### **数据原子性**

TracePacket 是 Perfetto Trace 中的最小数据单元，trace 数据是由一系列大大小小的 TracePacket 的序列化数据组成的。有趣的是，虽然 Shared Memory Buffer 被划分为 Page 和更小单元的 Chunk，但这并不会限制 TracePacket 的数据大小。TracePacket 是可以跨越多个 chunk 存储的。详情可如下图所示：               \
!\[\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

Packet 2 是一个尺寸巨大的 TracePacket，因而我们跨越了 3 个 chunk 来存储这份数据，分别是 chunk 1 -> chunk 3 -> chunk 4。为了保证后端数据能够按正确的方式还原 TracePacket ，chunk header 中会记录与之关联的前一份或后一份 chunk 的 ID

权衡利弊 3：TracePacket Size 如果太大，可能会使得 TracePacket 的原子写入还未完成时，与之关联的 chunk 已经写入了 Central Buffer，甚至已经从 Central Buffer 中递交到了真正的数据消费者（被写入文件或在 Ring Buffer 中被覆写）。

#### **TracePacket 数据写回**

我们已经了解了：TracePacket 是 Perfetto Trace 中的最小数据单元，其二进制数据是顺序写入的。如果数据的格式损毁，则 TracePacket 中的数据将无法还原。

现在我们要深入到另一个细节：TracePacket 的原子数据写入顺序。要说明这个问题首先需要了解 TracePacket 在内存中的布局，其细节如下图所示：

!\[\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

由于 protobuf 是可变字长编码，因此 TracePacket 会在其编码数据的头部预留空间用于标记数据段的字长。在通过 ProtoZero 库来进行快速的 protobuf 序列化编码时，序列化程序不会提前计算 TracePacket 的序列化字长，而是通过预留 size 字段，待 payload 部分的编码写入完毕后再返回 size 字段写回字段的长度。

目前为止一切看上去都运作良好，不过新的问题很快就会显现出来。让我们同时考虑数据写回与 数据原子性 小节中提到的权衡利弊的问题：

1. 当 TracePacket size > chunk size 时，TracePacket 的编码需要跨越多个 chunk；

1. 当最后一个 chunk 完成 TracePacket 的写入时，记录 TracePacket 的 size 字段的 chunk 可能已经写入了 Central Buffer；

1. 记录 TracePacket 头部数据的 chunk 也许已经在 Central Buffer 中消费掉了。

Perfetto 的应对策略：我们还可以通过 IPC 信号通道来通知 Trace Service 进行事后补救，只需要告知它：“请修复目标编号为XXX，来自 XX Producer 的 chunk” 便可以在问题 3 发生之前完成数据的补救措施。关于 chunk 数据的补救涉及的具体信号内容，可以参考 CommitDataRequest.ChunkToPatch 中的协议字段。

#### **增量编码**

大家可能了解过视频编解码中的 “帧间编码” 的概念，这是一个利用邻近帧之间的时域相关性来进行预测，去除相邻帧之间的冗余信息的编码过程。简单来视频由一帧一帧的图像编码组成，帧间编码在视频数据中确定了一些 关键帧，并通过算法来比较关键帧之间的差异，通过只记录 “变化区域的数据” 来实现数据压缩的效果。相对地，完整记录每一帧信息的编码方式称为“帧内编码”，关于两种编码方式的形象化说明，参见下图 帧内编码 vs 帧间编码：               \
!\[\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

Perfetto 也巧妙地利用了这一理念，通过增量编码的方式来节约数据记录的开销，这是另一个 “权衡利弊” 的例子：

1. 使用 Trace Event SDK 的 DataSource 会尽可能减少 TracePacket 中的 string 类字段记录，因为 string 字段通常不能得到很好的压缩。大多数的 string 类字段都只记录一次并创建 id -> string 的映射信息（类似于关键帧信息），后续的其它 TracePacket 在使用这些信息时就无需记录 string 而是其 id 以减少总体数据编码的大小，Trace Processor 当然可以在解码 Trace 数据的过程中还原这种映射关系。

1. 结合前文可知，这些关键帧信息可能会丢失，在这种情况下与之关联的其它 TracePacket 就无法正确地解析结果。Trace Processor 将会检测到这些错误的发生，并跳过关键帧波及的所有 TracePacket 信息。在 Central Buffer 处于 RING BUFFER 的记录模式时，这将会更经常地发生。

权衡利弊 4：为了降低数据丢失的风险，关键帧数据不应该关联太多的 TracePacket，我们都明白不要把鸡蛋全都放在同一个篮子里。定期地让增量编码失效并重新记录关键帧数据是很有必要的。               \
包含关键帧数据的 TracePacket 或许要存储在独立的 Central Buffer 区域，以在 RING BUFFER 模式下与关联的其它 Buffer 的填充速度相匹配。显然这也不是最优雅的做法，不过总能缓解问题发生的严重程度。

## **PART 4 - 应对之法**

通过 PART 1 ~ PART 3 章节的内容，我们已经深入探讨了关于 Perfetto Trace 的 DataFlow 中涉及的相关信息，其中不乏大量的 “权衡利弊”，很多时候都需要在性能开销、可靠性、易用性之间做出艰难的选择，最糟糕的是在某些极端的场景下数据丢失总会发生。

现在可以定论了：Perfetto 不是一个 100% 可靠的 tracing 平台，它只是一个拥抱开源（机制不完善）、稳定（不是 100% 稳定）且高效（并未追求极致性能）的跨领域系统跟踪和分析平台，我们可以接受现实了。如何才能正确地拥抱这个平台，并使用好这样的工具呢？

“我的剑留给能够挥舞它的人！” ————By 查理-芒格

#### **完整性检查**

即使无法避免错误，至少要能记录下错误发生的时刻。显然 Perfetto 的开发者们也深刻地理解这儿道理，于是他们特意为 Perfetto Trace 数据创建了一张独立的 SQLite 表，用于记录这些 “失败时刻” 的发生。我们可以在 Perfetto UI 的 Query(SQL) 功能栏中，通过以下 SQL 查询来获得这些信息：

select\*from stats where severity ='data_loss';

如果数据完好，则 SQL 的查询结果通常如下表所示：

|   |   |   |   |   |   |
|---|---|---|---|---|---|
|name|idx|severity|source|value|description|
|ftrace_cpu_overrun_delta|0|data_loss|trace|0|The kernel ftrace buffer cannot keep up with the rate of events produced. Indexed by CPU. This is likely a misconfiguration.|
|ftrace_cpu_overrun_delta|1|data_loss|trace|0|The kernel ftrace buffer cannot keep up with the rate of events produced. Indexed by CPU. This is likely a misconfiguration.|
|ftrace_cpu_overrun_delta|2|data_loss|trace|0|The kernel ftrace buffer cannot keep up with the rate of events produced. Indexed by CPU. This is likely a misconfiguration.|
|ftrace_cpu_overrun_delta|3|data_loss|trace|0|The kernel ftrace buffer cannot keep up with the rate of events produced. Indexed by CPU. This is likely a misconfiguration.|
|ftrace_cpu_overrun_delta|4|data_loss|trace|0|The kernel ftrace buffer cannot keep up with the rate of events produced. Indexed by CPU. This is likely a misconfiguration.|
|ftrace_cpu_overrun_delta|5|data_loss|trace|0|The kernel ftrace buffer cannot keep up with the rate of events produced. Indexed by CPU. This is likely a misconfiguration.|
|ftrace_cpu_overrun_delta|6|data_loss|trace|0|The kernel ftrace buffer cannot keep up with the rate of events produced. Indexed by CPU. This is likely a misconfiguration.|
|ftrace_cpu_overrun_delta|7|data_loss|trace|0|The kernel ftrace buffer cannot keep up with the rate of events produced. Indexed by CPU. This is likely a misconfiguration.|
|traced_buf_abi_violations|0|data_loss|trace|0||
|traced_buf_abi_violations|1|data_loss|trace|0||
|traced_buf_patches_failed|0|data_loss|trace|0||
|traced_buf_patches_failed|1|data_loss|trace|0||
|traced_buf_trace_writer_packet_loss|0|data_loss|trace|0||
|traced_buf_trace_writer_packet_loss|1|data_loss|trace|0||
|traced_final_flush_failed|NULL|data_loss|trace|0||
|traced_flushes_failed|NULL|data_loss|trace|0||
|misplaced_end_event|NULL|data_loss|analysis|0||
|truncated_sys_write_duration|NULL|data_loss|analysis|0|Count of sys_write slices that have a truncated duration to resolve nesting incompatibilities with atrace slices. Real durations can be recovered via the |raw| table.|
|perf_samples_skipped_dataloss|NULL|data_loss|trace|0||

|   |   |   |   |   |   |
|---|---|---|---|---|---|
|name|idx|severity|source|value|description|
|ftrace_cpu_overrun_delta|0|data_loss|trace|0|The kernel ftrace buffer cannot keep up with the rate of events produced. Indexed by CPU. This is likely a misconfiguration.|
|ftrace_cpu_overrun_delta|1|data_loss|trace|0|The kernel ftrace buffer cannot keep up with the rate of events produced. Indexed by CPU. This is likely a misconfiguration.|
|ftrace_cpu_overrun_delta|2|data_loss|trace|0|The kernel ftrace buffer cannot keep up with the rate of events produced. Indexed by CPU. This is likely a misconfiguration.|
|ftrace_cpu_overrun_delta|3|data_loss|trace|0|The kernel ftrace buffer cannot keep up with the rate of events produced. Indexed by CPU. This is likely a misconfiguration.|
|ftrace_cpu_overrun_delta|4|data_loss|trace|0|The kernel ftrace buffer cannot keep up with the rate of events produced. Indexed by CPU. This is likely a misconfiguration.|
|ftrace_cpu_overrun_delta|5|data_loss|trace|0|The kernel ftrace buffer cannot keep up with the rate of events produced. Indexed by CPU. This is likely a misconfiguration.|
|ftrace_cpu_overrun_delta|6|data_loss|trace|0|The kernel ftrace buffer cannot keep up with the rate of events produced. Indexed by CPU. This is likely a misconfiguration.|
|ftrace_cpu_overrun_delta|7|data_loss|trace|0|The kernel ftrace buffer cannot keep up with the rate of events produced. Indexed by CPU. This is likely a misconfiguration.|
|traced_buf_abi_violations|0|data_loss|trace|0||
|traced_buf_abi_violations|1|data_loss|trace|0||
|traced_buf_patches_failed|0|data_loss|trace|0||
|traced_buf_patches_failed|1|data_loss|trace|0||
|traced_buf_trace_writer_packet_loss|0|data_loss|trace|0||
|traced_buf_trace_writer_packet_loss|1|data_loss|trace|0||
|traced_final_flush_failed|NULL|data_loss|trace|0||
|traced_flushes_failed|NULL|data_loss|trace|0||
|misplaced_end_event|NULL|data_loss|analysis|0||
|truncated_sys_write_duration|NULL|data_loss|analysis|0|Count of sys_write slices that have a truncated duration to resolve nesting incompatibilities with atrace slices. Real durations can be recovered via the |raw| table.|
|perf_samples_skipped_dataloss|NULL|data_loss|trace|0||

stats 表中记录了各式各样用于描述 Trace 状态的元数据，这些元数据来源于 Trace 自身记录的数据以及 trace_processor 在处理 Trace 的过程中分析得到的数据。               \
通过 severity 字段，我们可以分辨不同等级的元数据记录：

- info: 常规的统计信息，通常不意味着 Trace 的内容错误或数据丢失，仅仅用于了解 Trace 自身的数据分布特性。

- error: trace 采集过程中发生的系统状态异常。此类异常通常标识了在正常情况下本不该发生的事件，例如：在 trace 会话过程中发生了时钟的校准；ATRACE 的 begin 与 end 事件无法正确地配对等等；

- data_loss:在数据传输的过程中发生了一些 “权衡利弊” 的决策，导致 trace 中的部分数据丢失。

针对一些重要的字段 description 中也做了详细的说明，可以作为我们进行错误分析的重要依据。

关于 stats 表中每个字段的详细说明，建议查看 Perfetto Document 中 stats 表的 Reference，本文将不再赘述。

#### **降低 Trace 丢失的概率**

现在是时候梳理 Perfetto Trace 的 DataFlow 架构中可能引起数据神秘失踪的诸多因素了，参见下图：               \
!\[\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

我们为可能存在故障的环节都做了红色标记，这使得 Perfetto DataFlow 看上去相当不可靠。下面让我们来一一梳理每个故障环节可以采取的措施：

##### **故障 1：ftrace buffer losses**

ftrace 的 buffer 大小是有限的，其填满的速度通常取决于来自 kernel 和 ATRACE 发送的数据量大小。这可能在不同设备上都有差异。为了在 ftrace buffer 出现数据丢失前拿走数据，我们需要 traced_probes 进程能够尽快地进行数据的读取和编码，并将其发送至一个足够维持一段时间的 Central Buffer。这里有两个建议的措施：

1. 为 linux.ftrace 的数据源映射足够大的 Central Buffer，在基于 RING BUFFER 策略的采集模式中更应如此。对于 Android 系统来说，ftrace 的数据乃是 Perfetto Trace 中最大的数据来源；

1. 保证 traced_probes 进程有合适的优先级来完成其工作。否则它可能会因为获取不到足够的 CPU 算力资源而无法及时地搬运 ftrace 中产生的数据。

##### **故障 2：Shared Memory Buffer Limit**

Shared Memory Buffer（后文简称 SMB）为数据生产者和消费者之间提供的共享内存 buffer 通常限制为 128-512 KB 大小，而单个 Page 的大小通常为 4KB~32KB。如果 SMB 中缺乏足够的 Free 状态的 chunk 来供数据生产者使用，则数据丢失将会发生。此处的建议措施：

1. 确保 traced 或其它 Trace 数据的消费者的消费速率能够大于或等于数据生产的速率。如果 traced 在一段时间内无法被 CPU 调度而不再搬运 SMB 中写满的 chunk，则数据丢失会开始发生；

1. 调整 SMB 和 Page 的大小以满足需求。然而大多数时候这需要我们在使用 Perfetto Client 库时通过设置 TracingInitArgs.shmem_size_hint_kb 与 TracingInitArgs.shmem_page_size_hint_kb 来实现。然而对于 Android 系统而言，基于 android.os.Trace (SDK) / ATrace\_\* (NDK) 的跟踪方式恐怕难以调整这两个值的大小。

##### **故障 3：Central Buffer**

Central Buffer 位于 Traced 进程中，是消费者处理 Trace 数据的数据中转站。根据 Central Buffer 的使用方式不同，其数据丢失的可能也不相同：

- RING BUFFER：Buffer 按序写入 Buffer，当 Buffer size 不足时最早写入的数据将被覆盖而丢失；

- STREAM LONG TRACE：Buffer data 将以流式的方式被读取和发送至其它消费者。Buffer 仍然可能在两次读取的间隔时段被填满并发生数据溢出；

- STOP WHEN FULL：Central Buffer 填满时停止 trace 会话。

**应对故障的可行措施：**

1.提供足够大的 Central Buffer 大小。无论是 RING BUFFER 模式还是其它模式这总是有效，在设备的 RAM SIZE 日益膨胀的今日，我们似乎不必吝啬 Buffer size 的扩张；

2.如果采用 STREAM LONG TRACE 的采集模式，可以通过降低两次 STREAM 读取之间的时间间隔来降低 buffer 数据溢出的风险；

3.为不同的 Data Source 映射合适的 Central Buffer，以减少不同数据源的数据写入速率不同而发生相互挤占的风险；

4.为了应对增量编码可能带来的数据集丢失，也可以尝试调整 “增量重置” 的间隔时间。

##### **故障 4：Trace file 存储**

数据只有真正被消费才不算丢失。无论是将其持久化至何处，保证数据落盘的实时性也是很重要的。如果 IO Block 的时间太久，Central Buffer 仍然可能发生溢出。这通常在低端机上是更容易出现的，尤其是发热发烫的机器。

#### **总结**

此处我提供了一份标准的 TraceConfig 示例，在大多数情况下这份 trace 可以连续记录 60 秒的 system trace 数据并且不会出现数据的丢失或错乱。在注释中详细说明了不同的字段如何控制各个故障点的可靠性：

buffers: {                  \
size_kb: 260096                  \
fill_policy: RING_BUFFER                  \
}                  \
buffers: {                  \
size_kb: 2048                  \
fill_policy: RING_BUFFER                  \
}                  \
data_sources: {                  \
config {                  \
name: "android.packages_list"                  \
target_buffer: 1                  \
}                  \
}                  \
data_sources: {                  \
config {                  \
name: "linux.process_stats"                  \
target_buffer: 1                  \
process_stats_config {                  \
scan_all_processes_on_start: true                  \
}                  \
}                  \
}                  \
data_sources: {                  \
config {                  \
name: "android.log"                  \
android_log_config {                  \
log_ids: LID_SYSTEM                  \
}                  \
}                  \
}                  \
data_sources: {                  \
config {                  \
name: "android.surfaceflinger.frametimeline"                  \
}                  \
}                  \
data_sources: {                  \
config {                  \
name: "linux.sys_stats"                  \
sys_stats_config {                  \
stat_period_ms: 1000                  \
stat_counters: STAT_CPU_TIMES                  \
stat_counters: STAT_FORK_COUNT                  \
cpufreq_period_ms: 1000                  \
}                  \
}                  \
}                  \
data_sources: {                  \
config {                  \
name: "linux.ftrace"                  \
ftrace_config {                  \
ftrace_events: "sched/sched_switch"                  \
ftrace_events: "power/suspend_resume"                  \
ftrace_events: "sched/sched_wakeup"                  \
ftrace_events: "sched/sched_wakeup_new"                  \
ftrace_events: "sched/sched_waking"                  \
ftrace_events: "power/cpu_frequency"                  \
ftrace_events: "power/cpu_idle"                  \
ftrace_events: "sched/sched_process_exit"                  \
ftrace_events: "sched/sched_process_free"                  \
ftrace_events: "task/task_newtask"                  \
ftrace_events: "task/task_rename"                  \
atrace_categories: "am"                  \
atrace_categories: "aidl"                  \
atrace_categories: "dalvik"                  \
atrace_categories: "binder_driver"                  \
atrace_categories: "gfx"                  \
atrace_categories: "input"                  \
atrace_categories: "pm"                  \
atrace_categories: "power"                  \
atrace_categories: "rs"                  \
atrace_categories: "res"                  \
atrace_categories: "ss"                  \
atrace_categories: "view"                  \
atrace_categories: "wm"                  \
atrace_apps: "\*"                  \
}                  \
}                  \
}

# trace 会话最长持续时间，在 LONG_TRACE 模式下有效

duration_ms: 60000

# 设置此字段后 STREAM 模式将启用，数据将定期从 Central Buffer 中读出

write_into_file: true

# 在 write_info_File = true 时有效，控制两次数据读出之间的等待间隔时间

file_write_period_ms: 2500                  \
max_file_size_bytes: 1500000000

# 要求数据源定期将数据提交至 Central Buffer，即使 SMB 中的 chunk 并未写满。常用于解决部分 Page 填满花费时间太久而无法即使刷入 Central Buffer 的情形

flush_period_ms: 30000                  \
\
incremental_state_config {                  \
\# 触发关键帧数据失效的间隔时间                  \
clear_period_ms: 5000                  \
}

无论如何，权衡利弊（trade-off）的策略都在可观测工具的各处发生着。Google 无法为所有的场景都调试出一份通用的 “甜点” 参数，所以将调整的余地开放给了使用者。

## **结语**

Perfetto 并不是一个十全十美的工具，如果想要“挥舞好这把宝剑”，我们就需要对其中的曲折原委有所了解。

希望这篇文章能够帮助到各位读者，让大家对于 Perfetto 的 Data Flow 架构有更深入的理解。当我们在下一次遇到Perfetto 无法配合工作时不会感到那么手足无措，而是能够试着通过调整一些参数来解决遇到的问题。

## **参考文献&资料**

部分参考资料、图片来源于如下网站链接，在此致谢~~

1. \[Perfetto Documents\] - https://perfetto.dev/docs/

1. \[Long GOP vs All instra\] - https://www.sonystyle.com.cn/content/dam/sonystyle/products/ilc/e-body/ilce_7m4/feature/ilce_7m4_d04_6857b92c.jpg

往

期

推

荐

[Android分区挂载原理介绍（上）](http://mp.weixin.qq.com/s?__biz=MzAxMDM0NjExNA==&mid=2247491998&idx=1&sn=8800a373e4874c7ace4cad8ad22cbcd4&chksm=9b536a73ac24e36574a32e973a615223a1d1692952d44cca73ff166de0e5127b2f919ade5e89&scene=21#wechat_redirect)

[Android分区挂载原理介绍（下）](http://mp.weixin.qq.com/s?__biz=MzAxMDM0NjExNA==&mid=2247492000&idx=1&sn=cdf4ee5146dfe8c11616becfe154c3c0&chksm=9b536a4dac24e35b2e6c71f1c18ba0153fc11f9d4d87f6d5da8b990e92ee38e686ff13ddbe13&scene=21#wechat_redirect)

[深入理解Linux内核共享内存机制- shmem&tmpfs](http://mp.weixin.qq.com/s?__biz=MzAxMDM0NjExNA==&mid=2247492060&idx=1&sn=c69f7815bf8364dd34e097d0bfb0250a&chksm=9b536a31ac24e327d426583cdf53a53aeca3c68e09dedbabc5b720595d68a61d6e01837d7089&scene=21#wechat_redirect)

!\[\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

长按关注内核工匠微信

Linux内核黑科技| 技术文章| 精选教程

阅读 2498

​
