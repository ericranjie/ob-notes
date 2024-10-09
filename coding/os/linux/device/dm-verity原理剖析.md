# 

Original Mcgrady_wu OPPO内核工匠

_2021年09月10日 17:00_

**一、技术模块简介**

Dm-verity 是 device-mapper 架构下的一个目标设备类型， 通过它来保障设备或者设备分区的完整性。

Dm-verity类型的目标设备有两个底层设备，一个是数据设备(data device), 是用来存储实际数据的，另一个是hash设备(hash device), 用来存储hash数据的，这个是用来校验data device数据的完整性的。

简单的架构图如下：

![Image](https://mmbiz.qpic.cn/mmbiz_png/d4hoYJlxOjOGz8hI5WJKqy9FNqicb0cgmOrQ7kzTnzrYhg5CTJHadozRMhk2nDSD1NWrFPDjDuEzRnkSdV5UFPw/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

图中映射设备(Mapper Device)和目标设备(Target Device)是一对一关系，对映射设备的读操作被映射成对目标设备的读操作，在目标设备中，dm-verity又将读操作映射为数据设备（Data Device）的读操作。但是在读操作的结束处，dm-verity加了一个额外的校验操作，对读到的数据计算一个hash值，用这个哈希值和存储在哈希设备(Hash Device)

**二、设计原理**

对于本文要介绍的dm-verity功能模块，笔者选择在当前移动终端应用的角度来展开讲解，也就是Android平台在dm-verity的应用。

Android 端主要是在镜像启动时验证这个功能场景上使用到了 dm-verity 技术，该技术可以对块存储设备进行完整性检查，有助于阻止某些恶意程序对镜像的修改，有助于Android用户在启动设备时确认设备状态与上次使用时是否相同。在系统镜像(比如 system、vendor等)启动时以及运行时可以实时性监测当前镜像是否被篡改。

通过dm-verity技术，可以确认块设备内容是否跟预期一致，具体的实现原理是利用哈希树(hashtree)做到的。用以下图来形象说明

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

(图来自：https://source.android.com/security/verifiedboot/dm-verity)

简单说明一下这个插图背后的原理：

在编译阶段，首先会对系统镜像(比如system.img、vendor.img)按照每4k大小计算对应hash，将这些hash信息存起来，形成上面图中的layer 0层，紧接着会对 layer 0 层同样按照每4k大小计算hash，并将这层的hash信息存起来，形成layer 1层，以此类推，直至最后的hash信息存放在一个4k大小的块中(未填满使用0填充)，这里边存储的hash信息称为 root hash。

在运行阶段，对于镜像里的文件进行访问时，操作对应所在块设备的存储区域时，会计算当前存储块(按4k大小)的hash值，然后会跟存储在哈希树上对应块的hash值进行比较，如果匹配不上，则认为当前该文件在底层存储被篡改或是损坏了。

为了更形象的描述下镜像运行时如何利用哈希树做校验的，下面以一个1G大小的镜像为例，来说明一下这个过程：

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

根据 hashtree 的生成方式，以 1G 的镜像为例：

1）按照 4K 大小划分，将1G 大小的镜像依顺序划分可得到 262144 个 4k 大小的块

2）对这 262144 块数据块进行第一层(Level 0) hash 计算，由于 SHA256(具体的hash算法可配置，此例以SHA256为参考) 计算出来的hash值占 256 个字节，一个 4K 的块可以存储 128 个hash值，所以存储这 262144 块数据块的hash值需要花费 2048 块

3）对第一层存储 hash 值的数据块进行第二层(Level 1) 的 hash 计算，同理，计算这 2048 块hash数据块需要花费 16 块

4）对第二层存储 hash 值的数据块进行第三层(Level 2) 的 hash 计算，由于第二层的hash数据块小于128块，所以第三层是最后一层，直接计算得到 root hash 数据块(不够4K大小补齐0)。

细心的读者可能已经发现了，Level 0 层其实已经包含了所有raw data数据块的hash信息了，也就说明Level 0 层已经具备可以验证 raw data 的能力了，为何还需要在 Level 0 的基础上继续算hash组装下一个 Level 层级呢？

这里要引入一个安全策略设计问题，镜像raw data数据块是由对应的hash信息来校验保证的，为了保证镜像raw data是我们“想要的”，我们还需要对 hash 信息进行合法性验证，简单理解就是要确认这个hash信息是我们“想要的”， 方法就是对这个hash信息再次计算一次hash值，这里称为 root hash，然后添加一些类似于签名保护并保存起来，主要是为了防止 root hash 被非法篡改。在一次 raw data 数据块的校验过程中，需要对 hash 信息计算一次 hash，然后跟保存的 root hash 进行比较，验证了 hash 信息的合法性之后，再来校验对应的 raw data 数据块。

有了以上背景，再回到刚刚这个“为何还需要在 Level 0 的基础上继续算hash组装下一个 Level 层级”的问题，按照上面的安全策略，如果 hash 信息只有 Level 0 一层的话，接下来对 raw data 数据块的校验将会是这样：每操作一个raw data数据块，都需要计算一次 Level 0 的 hash 值，跟 root hash 进行比较，验证合法之后再对 raw data 数据块进行校验。本文中举例是 1G 的镜像，Level 0用来保存 hash 信息的数据块已经达到 2048 块，如果对面对更大的镜像，Level 0 所占的数据块也会更大，如果是按照上面的计算方法，对 raw data 数据块的校验效率将会非常非常低。

作为对比，哈希树机制是如何体现出效率呢，下面以具体某一块 raw data 数据块的读取过程来说明其设计原理：

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

1）假设目前正需要读取第 200000 块的数据块，通过前面哈希树的构造，可以比较快速的计算出在 Level 0 层，也就是直接对应这个数据块的hash值存储位置，通过对 128 相除以及求余的方式就可以分别计算出该 hash 值存储在具体某一块(块A)以及这一块上的偏移(偏移A)。

2）确定了 Level 0 层的具体块A后，利用相同的方式可以得到 Level 0 块A在 Level 1 层存储其 hash 值的块B位置以及块B上的偏移(偏移B)

3）同上述原理，可以最终定位到 Level 1 层块 B 在 Level 2 上的数据块(只有一块)上偏移(偏移C)

4）在获取到该数据块对应hashtree关联的各个层级的块以及偏移后，接下来就是做一层一层的验证：

步骤1. 优先验证的是Level 2 中的数据块(只有一个4k的数据块)， 计算这个数据块的hash，跟保存的 root hash进行比对，验证Level 2的数据块块是否正确。

步骤2. 在Level 2中的数据块得到验证后，由上面计算到的Level 2中数据块上的偏移 C 去校验 Level 1 层的 B 块。

步骤3. 在 Level 1 层的 B 数据块得到验证后，由上面计算到的 Level 1 中 B 数据块上的偏移B去验证 Level 0 层的 A块。

步骤4. Level 0层中的 A 数据块得到验证后，最终会由 Level 0 层的偏移A来校验最终的 raw data数据块(第 200000 块).

可以看到，在使用哈希树的设计之后，对数据块验证整个过程中，涉及到的数据块hash计算只有3块(有N层就计算N块)，相比于一层校验模式，效率要高很多。

**三、应用层面**

读者到目前为止应该大致了解到了移动终端在镜像完整性校验上相关的设计原理，接下来会从应用端层面来说明如何使用 dm-verity 底层提供的接口来初始化 dm 设备，并为后续实时校验做好准备，内核绝大多数功能模块存在的意义都要靠跟应用端的交互来体现，作为对内核某个模块的研究，初步从应用层出发不乏是一个好办法。

应用层主要有以下3个步骤：

步骤一. Create dm device

创建 dm 设备主要有如下小步骤：

1\) open  /dev/device-mapper 设备节点

2\) 传入逻辑分区name、随机生成的uuid参数，调用  DM_DEV_CREATE ioctl 命令

步骤二. Load verity table

这里需要引入一个 verity table 的概念，先简单介绍以下 verity table 所包含的内容：

1）Verity target version(verity target 版本号)

2）Data block device(存储实际待校验数据的块设备)

3）Hash block device(存储校验使用到hash的块设备，一般情况跟data block device是同一个)

4）Data block size(数据块设备的每块存储size)

5）Hash block size（hash块设备的每块存储size）

6）Num data block(数据块设备占用的块数量)

7）Hash start block(hash设备在存储设备的起始位置)

8）Hash algo(hash算法)

9）root digest(对应上面说的 root hash)

10）Salt(用于计算hash的盐值)

这些信息主要是跟最终数据块在校验计算过程中会被使用到的，比如说 hash 设备的起始位置、hash算法、root hash、salt，这些都在实际运行时校验数据块时会用到，这些信息是存储是镜像的固定位置上，这些信息在编译阶段构建镜像的时候就已经计算好的，并存储在镜像的固定位置。

Verity table 初始化代码具体如下：

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

上层通过从镜像固定位置获取到信息并初始化好 verity table， 通过调用DM_TABLE_LOAD

Ioctl 命令将 verity table 传递至kernel。

步骤三. Active dm device

调用DM_DEV_SUSPEND ioctl 指令激活 dm device，对应底层，该 cmd 对应 suspend & resume的实现，如果不设置 DM_SUSPEND_FLAG 标志位，默认走 resume 流程。

应用端在实现上比较简单，主要通过 create -> load verity table -> active dm device 的流程完成了对dm设备的创建、verity table的读取以及传递以及dm设备的激活，为后续实时进行的数据块校验做好了初始化工作。

**四、内核层面**

有了以上应用层面的流程讲解，那对应内核，自然而然就是对每一步应用端的系统调用做对应的内核实现做讲解。

相应的，内核层面也有以下3个步骤：

步骤一. dev_create

对应应用端的DM_DEV_CREATE ioctl cmd，kernel端的大致实现如下：

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

这部分比较简单：

1). 检测传入参数的partition name是否合法

2). 开始尝试分配内存初始化 mapped device 结构体以及分配设备minor号（最终用于 dm 设备的设备号，比如 dm-1），使用内核提供的blk_queue_make_request函数注册该mapped device对应的请求队列dm_make_request, 该请求队列最终会在IO重定向中被使用到。并将该mapped device作为磁盘块设备注册到内核中。

3). 将创建好的mapped device插入到一个全局hash表中，该表中保存了内核中当前创建的所有mapped device。

步骤二. dm_ctl_ioctl(DM_TABLE_LOAD_CMD(table_load))

对应应用端的DM_TABLE_LOAD ioctl cmd，kernel端的大致实现可看下面的思维导图：

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

总的来说，这个步骤主要是根据入参初始化相应的dm_table、dm_target结构，并且根据参数所指定的target类型，调用相应的target类型的构建函数ctr在内存中构建target device， 在结构上形成 dm_table --> dm_target --> target type --> target device 的链路结构。

步骤三. dm_ctl_ioctl(DM_DEV_SUSPEND_CMD(dev_suspend))

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

很显然，这一个步骤主要是建立 mapped device 与 dm_table 的关联。

通过以上几个步骤，在内核中就建立一个可以提供给用户使用的mapped device逻辑块设备

综上涉及到了几个关键的数据结构： mapped device、dm_table、dm_target、target_type、target device(以dm-verity为例)

其实这几个步骤主要是对上述数据结构进行初始化，并且更重要是互相建立了关联关系。他们之间的关联关系如下：

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

内核经过了这3个步骤，一方面是创建了一个可以提供给应用端使用的 mapped device 逻辑块设备，另一方面是内部建立了 mapped device - target device 的联系， 应用层可以通过对 mapped device 进行策略逻辑操作，最终会通过 mapped device - target device 的联系作用到 对应的target device 上。

**五、核心数据流**

在上面介绍了dm-verity的设计原理、应用层面以及内核层面的实现之后，读者可能比较关心整个链路的数据流，或者说镜像在校验链路具体流程是如何的，接下来以下主要是围绕着访问镜像文件时的IO流是如何的。

上面说到，应用层在经过 create -> load verity table -> active dm device 的流程完成了dm块设备的初始化工作，之后应用层会对该逻辑块设备进行文件系统挂载，在挂载的过程中，需要访问到实际存储设备(读取文件系统的super block等)，这个过程中就需要透过这个逻辑块设备，最终操作到与其关联的target device。

在这个过程中， 对块设备的IO请求会从逻辑设备mapped device转发相应的target device上，并且会根据对应target_type描述的IO处理规则对IO请求进行处理。以本文讨论的 dm-verity 类型的 target device 来说，对于 mapped device 转发过来的IO，会在 hashtree 里找到该 IO data 对应的 hash 数据，并进行比较，完成校验，返回此次的校验结果并结束本次IO请求。

**六、总结与回顾**

本文从设计原理并结合例子出发，呈现了 dm-verity 在设计层面的实现效果，能让读者能简单、直白的了解原理核心所在，紧接着对应用层的 Ioctl 指令、kernel对应的实现细节的解读，搭建了一套用来支撑上述设计原理的软件结构，最后通过核心数据IO流的流转方式的描述，结合上述搭建的软件结构来反映了 dm-verity 设计原理的最终实现。

参考资料：

1. Linux kernel-5.4 source code

2. https://source.android.com/security/verifiedboot

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**长按关注**

**内核工匠微信**

Linux 内核黑科技 | 技术文章 | 精选教程

Reads 1984

​
