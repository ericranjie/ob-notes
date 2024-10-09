原创 2021 SDC 看雪学苑

_2021年11月03日 17:59_

解优化过程是在优化过程中的一个逆过程，如果隐藏了一些安全隐患在解优化过程中，往往会造成一些安全影响。相对于直接产生错误代码的优化漏洞，解优化过程的漏洞会更加的隐蔽。

刘博寒先生作为玄武实验室高级安全研究员，针对解优化漏洞的类型转换问题、其他机制的不适配等技术难点细节，分析并分享了多个实际的解优化模块漏洞原理和漏洞利用技巧。

下面就让我们来回顾[2021看雪第五届安全开发者峰会](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458396801&idx=1&sn=2d13a137f3d17766558e8689d48fd341&chksm=b18f180b86f8911d25d2ff9e0c50068c4e202187e72f9b77fe1bfcfcc14ffd662805264c7dab&scene=21#wechat_redirect)上\*\*《**Chrome 浏览器解优化过程中的漏洞安全研究**》\*\*此议题的精彩内容。

![图片](https://mmbiz.qpic.cn/mmbiz_png/ZN1xiarzMc8oCQowagb1nlL3nRfaHW67eickDGd25WCMxw3oNKouamvCg1Hg1J88CX0g7WO1qcoiaQWwnhXibibqcKQ/640?wx_fmt=png&wxfrom=13&tp=wxpic)

演讲嘉宾

![图片](https://mmbiz.qpic.cn/mmbiz_png/t8HEWljw1E7dMrqROMVthRC4Xic9NUPnS7d3Uh4X7T8aiaLpcsHyT6gTYBQdibgSAJn92ibia0oSmvTu60bg5roBt6w/640?wx_fmt=png&wxfrom=13&tp=wxpic)

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

【刘博寒-腾讯安全玄武实验室高级安全研究员】

刘博寒：腾讯安全玄武实验室高级安全研究员

主要关注浏览器相关的安全研究。曾发现并报告多项Chrome浏览器安全漏洞，Google Security Hall of Fame 当前排名92。

![图片](https://mmbiz.qpic.cn/mmbiz_png/KLoTw1Op24K7bYlV0ty3cYaXKEJ4LukvDSWzMiawENwjkzichAIcDuC1uBxuMfSj29gpevgLGPPeMeHCwKyGiaZ5w/640?wx_fmt=png&wxfrom=13&tp=wxpic)

![图片](https://mmbiz.qpic.cn/mmbiz_png/ZN1xiarzMc8oCQowagb1nlL3nRfaHW67eickDGd25WCMxw3oNKouamvCg1Hg1J88CX0g7WO1qcoiaQWwnhXibibqcKQ/640?wx_fmt=png&wxfrom=13&tp=wxpic)

演讲内容

![图片](https://mmbiz.qpic.cn/mmbiz_png/t8HEWljw1E7dMrqROMVthRC4Xic9NUPnS7d3Uh4X7T8aiaLpcsHyT6gTYBQdibgSAJn92ibia0oSmvTu60bg5roBt6w/640?wx_fmt=png&wxfrom=13&tp=wxpic)

以下为速记全文：

大家下午好，我是刘博寒，来自腾讯安全玄武实验室，今天分享的议题是 《chrome浏览器解优化过程中的漏洞安全研究》。首先对本人和团队进行介绍，玄武实验室主要开展面向实际威胁的安全研究。包括以主流操作系统和主流应用为目标的二进制类安全研究、针对Android平台的移动安全研究、区块链安全研究、硬件设备安全研究、渗透测试以及安全开发等工作。欢迎对相关领域感兴趣的同学投递简历。我主要关注浏览器相关的安全研究，曾发现并报告过多项Chrome浏览器的安全漏洞。

在今天这个议题中，我主要**分4个模块进行介绍，首先是背景知识，然后是对一些解优化模块的历史漏洞进行分析，最后提出一种解优化模块的漏洞利用技术，并对这个议题进行总结。**

01

背景知识

首先我们可以看一下 chrome的安全现状，chrome浏览器作为市场占比最高的浏览器，同时也饱受 APT攻击和0 day漏洞的困扰。

在PC端和移动端的软件安全地位上，chrome也处于一个很高的位置。

根据Google捕获或者收到的在野漏洞报告统计，在v8引擎中存在的漏洞数量是大于50%的。**在v8引擎中优化漏洞占比很大**，目前也是攻击者挖掘的主要目标。

Google对待这种优化漏洞的修复是比较激进的，在修复漏洞同时也会修复这些漏洞利用的通用方式。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

如上图所示是从2019年到2021年修复的4条typer漏洞的利用方式，在优化漏洞挖掘和利用的难度不断增加的情况下，我们开始关注解优化模块，这个模块是优化过程的逆过程，二者是息息相关的，**因为它不直接产生错误代码，所以说结构化漏洞是更加隐蔽难以发现的**。

进入背景知识部分，我们首先看一下v8中 js代码的执行流程，在初次执行 js代码的时候， js代码会解析成 bytecode，然后在ignition中按bytecode解析执行，每一个bytecode实现了一个完备的功能，同时会搜集执行的一些参数信息。

在ignition执行多次以后，会根据搜集到的参数进行推断优化，生成的 jit代码最终编译成符合我们当前输入的这些类型的汇编语言，而不去处理超过我们之前遇到的这些参数范围，这样会大大提升它的处理效率。

当JIT能处理的类型不能满足输入的要求时，JIT代码会解优化，回退到对应Bytecode中去执行。最终在Deoptmize节点处解优化，Deoptmize节点有一个framestate，其中保存了恢复到ignition执行的所需的现场环境。

V8的Turbofan优化有以下5个阶段，首先在 graph building这个阶段主要是从bytecode生成ir图，并插入一些初始的FrameState用于保存解优化环境状态。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

然后在inline阶段会将JSCall节点展开成更细致的流程节点，然后在type阶段会根据已有信息及节点的操作规则对各节点进行类型和范围标记,基于上述类型信息对Builtin函数及对象生成函数进行优化。

General Optimization这个阶段是优化环境的主阶段，与编译原理有关。然后在Code generation这个阶段，它会划分基本块，选择指令、寄存器，最终形成 jit汇编代码。

如果我们要了解解优化的问题，就需要先了解优化与解优化之间的联系，然后我下面会分享几个与解优化相关的优化阶段。

第一个阶段就是逃逸分析，逃逸分析这个阶段会检查函数中的非逃逸变量，然后优化掉分配内存的节点。在foo函数中我们可以看到a是临时变量，它的生命周期是与foo函数相同的，而在逃逸分析之前，我们可以看到右侧的这一系列节点都是为它分配内存，储存现场环境的，而在逃逸分析之后的阶段，为临时变量分配内存的这部分节点被会被消除掉，临时变量会被折叠成常量。但当从jit代码解优化到解释器执行时，bytecode需要一个js对象来作为参数进行赋值等，因此解优化时需要根据逃逸分析处优化成常量的各数据内容进行恢复对象。

在逃逸分析之后，就是SimplifiedLowering这个阶段，这个阶段是会根据Typer阶段数据等信息为各节点确定representation，并在不同类型的节点间增加转换节点。分为三个阶段，反向传播Truncation信息，即节点要求的输入。正向传播类型信息，计算节点的输出。转换阶段会转换节点类型或者增加不一致类型的转换节点。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

在SimplifiedLowering这个阶段结束之后，然后就进入 instruction selection这样一个阶段，它这个阶段主要的作用是选择所要使用的指令，而这个阶段也是最开始处理解优化节点的一个阶段。

主要将FrameState节点转换为FrameStateDescriptor结构。

当前 FrameState 的输入节点保存了函数栈所需的参数、临时变量、上下文等信息。后续会把每个frame输入的node转换成StateValue结构保存，每一个framestate形成以StateValueDescriptor为节点的树。对于普通节点，会将根据OP生成StateValueDescriptor；对于TypedObjectState等表示对象的节点，会进一步生成其自身的StateValueList，以便加入其成员数据。

然后我们进入了与解优化相关的最后一个优化阶段，是AssembleCodePhase的一部分，主要是针对之前选择的instruction，生成汇编代码。其中会根据之前绑定在 Deoptimize 的 FrameStateDescriptor 生成 Translation 对象，该对象是一个 buffer ，保存了序列化的 FrameStateDescriptor 数据。

常量会统一保存在literals中，translation 和 literals 两个数组会被保存在 JSCode 对象中，供解优化时随时可以找到。

对于解优化数据的保存，是采用序列化的方法，记录 FrameState 结构及数据存储位置，而JIT不会生成额外的解优化数据保存代码。

以上就是所有在优化段落中跟解优化相关的这些阶段。当了解了这些之后，我们就可以聊聊什么是解优化。当jit函数的输入不满足之前预期的类型时，它就会弃用之前生成的优化代码，然后基于优化代码的这部分的堆栈结构，生成回退到 igniton所需的StackFrame，跳转回检查失效位置的bytecode偏移，继续执行。

解优化会包括以下4个流程。

保存现场环境时，会把当前全部寄存器压到栈上，因为jit所用的数据仅存在于栈和寄存器中。生成Deoptimizer：主要是对input frame的初始化。生成TranslatedValue：是对translation的反序列化，生成各上下文对应的值。Materialize：重新打包对象，为其申请内存并迭代生成其中各成员信息，填充output frame。

解优化过程是由于JIT函数使用的堆栈与Interpreted Frame不一致，并且临时变量在JIT函数中并不以对象形式存在，因此**在解除优化的过程中需要重新生成Ignition所需的堆栈数据**。

以上就是关于解优化的一些基础知识的介绍，我们下面会对解优化模块的一些历史漏洞进行分析。

02

解优化模块历史漏洞分析

第一类历史漏洞，是优化中类型转换问题，主要分析的是CVE-2020-6512漏洞，这个漏洞是微信团队在2020年7月份左右发现的，这个漏洞我们可以从patch首先看起，patch采用了一个很复杂的写法，在接优化生成对象时当发现从上下文提取出的时smi类型时会为其生成heapnumber对象，其他位置使用时转换为smi。即改变了默认的使用方法。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

然后我们可以从公开的一个 testcase来分析原因，我们可以看到有两个比较有意思的点：1、使用循环变量对obj的属性赋值；2、使用obj触发解优化，将会重新生成obj对象。

首先，在逃逸分析阶段，会对各个内存可能出现的值进行统计，并为可能出现的值建立一个phi节点用于合并。在SimplifiedLowering这个阶段时候，会对加入的phi节点，精确它的赋值。

我们可以看到149节点实际上是等同于它循环变量这个节点，它会出现的范围就是从- 5000000000 ~ 1000000000。而这个151节点它分析的位置实际上是对 obj的my_property这个属性，是它会默认为其加一个Undefined。

实际上是认为当 object的生成而没有赋值的时候，它那个位置是Undefined，所以说会对151这个节点的类型打了一个 tagged的标记，就是全部对象，从而149需要产生一个转换节点。

在EffectLinearization阶段，这个转换节点将被进一步展开尝试将其转换为Int32，如果可以就使用这个值，而此处会连接一个FrameState，**当此处发生解优化时，寄存器中存储的数据并不是对象需要的数据类型**。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

所以在结构化发生的过程中，对 objects生成的时候，对my_property属性赋值的时候会把 SMI类型写入属性，产生了一个**属性类型错误**的对象。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

这个一个因为精度问题导致的类型放大，进一步转换成类型生成错误，并且难以用简单的方法规避掉这种情况。在整理过程中，我们发现还有很多同类型的漏洞。在这次议题中我们就不对其他的问题进行分析了。

然后我们对这一类问题做出一个总结就是该类问题主要出现在某些解优化发生时刻，类型转换导致与对象生成所需类型不一致；该类问题的修复一般只能根据修复特定情况下的问题，容易遗漏；漏洞本身出现在优化阶段，但需要解优化时甚至需要其他进一步才能触发效果；漏洞以类型混淆为主，通常是smi与HeapObject之间的混淆。

然后第二类漏洞我想分享的是解优化与其他机制间的兼容性，因为我们可以看到结构化其实是一个非常复杂的逻辑，这里要分享的是CVE-2021-21195，这个漏洞是我今年3月份发现的。我们可以想一个问题，**当同一个临时变量被生产两次，会发生什么问题？**

首先如果想要达到这种状态，就必须满足第一个条件，就是是否在解优化以外的位置同样可以发生对象的重新生成。

然后第二个就是说，V8的堆管理机制对指向相同内存的不同对象的处理。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

首先我们来看第一个问题，我们在exception stack trace mechanism中找到了可以重新生成对象的一个代码位置，然后这个机制实际上是在JSError发生的时候，用于追踪错误调用栈该位置会打包发生错误的Function及上下文。并且在优化函数中发生不会解优化。

当有如下条件发生的时候，是可以生成两个指向同样数据的对象。

在优化函数中发生错误，产生JSError对象；发生错误的函数上下文中包含需要重复产生的对象；该重复产生的对象同样会被解优化的MaterializeHeapObjects过程生成，最终我们构造出了这样的一个test case， test case也被写进了v8的模块测试中，在bar函数中由于饮用未声明对象导致error，会打包arr；同时正则表达式test错误会导致解优化，生成arr；最终，两次重新生成的arr指向同一块element。我们又找到了一个非常神奇的函数Array.prototype.shift。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

这个函数在如果大家对JS很熟悉的话，它就是对array的第一个参数进行弹出，Shift的实现会将所有的数据向后移动。在这一次操作之后，内存实际上是这样一个状态，JSError生成的array中element实际上是指向了一个filler对象，通过gc将filler这里释放，导致UAF，但与传统uaf不一致，由于拿不到JSError中的这个Array对象这里会使得gc的扫描器混淆。

这个问题是在之前某个版本对逃逸分析细化引入的，属于一个机制问题，影响了10+个版本。我们对这一类问题也做一个总结，该类漏洞由各机制间的兼容性导致，并不通用。但漏洞可能由一些无意的功能优化引入，可能导致各类漏洞出现，利用并不统一。

03

解优化漏洞利用技术研究

我们下面将介绍一种解优化漏洞利用技术，在解优化模块中， MaterializeHeapObjects阶段受优化类型分析的结果影响，容易生成错误的V8对象，在解优化模块漏洞中占比很高。并且这一类漏洞以**类型混淆漏洞**为主，并且缺少地址泄漏。

然后我们后面会以CVE-2020-6512漏洞为例做漏洞利用。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

然后我们可以首先来看一下漏洞的初始状态是什么，在Release版本，程序不会崩溃，而是产生一个对象。该对象DescriptorArray中标明的my_property的类型是HeapObject，但实际在对象中存储的是0x42424242，该值用于表示0x21212121这个smi。

当直接访问生成错误的属性时，并不会发生类型混淆，而是会仍以smi类型进行解析。V8对象对于smi和HeapObject的区分是因末位来进行标识的。那么是否存在不进行判断直接使用的位置呢？我们知道如果进行判断是不会产生任何问题的。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

然后我们使用了优化函数，首先预先构造这样的一个优化函数，它的图是这样的：1、在 39: CheckMaps 检查maps是否发生改变；2、在 42: CheckMaps 继续检查 obj.my_property 的map是否发生改变。**当以上检查全部通过时，会将smi作为HeapObject来访问，从而形成类型混淆，导致任意地址读写。**

但是在产生类型混淆的过程中，必会遇到两个问题：

1、smi所指向的地址不存在，程序崩溃；

2、smi所指向地址中不包括 CheckMaps 中所检查的 map ，优化函数被解优化，无法类型混淆。

因此，针对当前的漏洞状态，有两个问题亟待解决：**在没有地址泄露的情况下，如何得到稳定可控的地址？如何在上述地址布置 map ，从而构造内存读写？**

其实第一个问题对很多做二进制的人都非常的困扰，就是说怎么去bypass随机化呢？然后我们找到了一个v8的指针压缩机制，在M80之后，为了减少V8对象的内存使用，在64位架构中将V8对象存储转换为32位。通过以下方式将指针调整为32位。1、确保所有V8对象分配在4GB范围内；2、将指针表示为这个范围内的偏移量。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

在运行时将V8堆的高32位存储到r13寄存器中，当使用时，利用该寄存器加上V8对象内部存储的偏移值作为对象访问的地址。这个机制虽然说对内存是友好的，但是它降低了v8堆的随机化程度。

我们来看一下这个机制的具体实现，首先在V8 Isolate初始化的时候，会预先申请4G大小的内存，作为整个V8的对象内存空间。加入freelist。

此后，V8将该堆划分为如下5种类型进行分块管理，每种类型的存储用于存储活跃性、大小不同的对象。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

而我们可以看到OldSpace、CodeSpace、MapSpace的基类是PagedSpace，该类的内存管理模式是各Space从RegionAllocator中的 freelist 切割下所需要的大小，然后把剩余的大小再放回 freelist 。并且，在各个 PagedSpace 内部分配 V8 对象时，采取的方法也是从起始位置依次切割的方法。

那么我们可以得到一个结论，对于稳定的 PagedSpace 中越早申请的对象相对偏移越稳定。我们就根据以上推断就找到了两个非常稳定的对象地址，第一个就是静态字符串，这个对象是在解析js的源码到AST这个阶段生成的，所以它是非常稳定的。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

第二个就是对象的map，存在于MapSpace，且生成map的情况很少。利用上述对象偏移的可预测性，我们可以在静态字符串中写入map对象地址，将该字符串地址写入错误生成的对象，得到类型混淆。当拥有了一个类型混淆后，利用优化函数可以向任意地址写值，但当前仍然缺少地址泄露，向哪里写值又是一个问题。需要满足两个条件：

**1、稳定的地址；**

**2、可以进一步获得越界读写等能力。**

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

在这个问题上我们又找到了String对象，String对象拥有很多类型，其中一个是ONE_BYTE_INTERNALIZED_STRING_TYPE，其内存构造是长度+变长的字符串。当String A的长度被修改后，可以向后进行搜索，利用lastIndexOf(ArrayMap)的返回值。

利用堆喷，可以使得Victim Array 分配在String a对象内存后面，当String A的长度被修改后，可以向后进行搜索，利用lastIndexOf(ArrayMap)的返回值，可以计算出Victim Array的地址，并且也可以泄露Array element的地址。后面利用成熟的方法很容易达到RCE。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

04

总结

然后最后对议题做出总结，在优化漏洞备受关注的同时，解优化模块也是一个很好但隐蔽的攻击面。优化部分的类型转换问题、其他机制的不适配等原因，均可以在解优化时产生意想不到的问题。对于优化中类型转换导致的错误对象生成，通常可以转换为类型混淆漏洞，本议题利用压缩指针的特性预测部分对象地址，从而提出一套稳定利用方案。当然我这套利用是在83版本写的，最近的还有没有一些新的mitigation我不太确定。以上就是我演讲的全部内容，谢谢。

注意：点击**阅读原文**即可获得本次峰会演讲完整PPT！其它议题演讲PPT经讲师同意后会陆续放出！

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

2021 SDC议题回顾

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

[2021看雪SDC议题回顾 | 基于Qemu/kvm硬件加速的下一代安全对抗平台](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458400885&idx=1&sn=5d7455f419dc3329046d514b19da8305&chksm=b18f08ff86f881e923b8dedc9cfdf67c6685fd4836eaaede37f6fa56b28f7d22c011d8f19ae0&scene=21#wechat_redirect)

[2021看雪SDC议题回顾 | 代码混淆研究的新方向](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458400936&idx=1&sn=8590e661c6e1dc2919026750e2389a42&chksm=b18f082286f88134789746cbd166b5419d884e7305d12df031fa107e75fb938368802c378043&scene=21#wechat_redirect)

[2021看雪SDC议题回顾 | houdini编译机制探索](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458400936&idx=1&sn=8590e661c6e1dc2919026750e2389a42&chksm=b18f082286f88134789746cbd166b5419d884e7305d12df031fa107e75fb938368802c378043&scene=21#wechat_redirect)

[2021看雪SDC议题回顾 | Make Deep Exploit RCE Attack Popular](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458401181&idx=1&sn=46564e3062a14d7e5dc205f1cff340e2&chksm=b18f091786f88001a5e422cc5057f670a013ff2a954e58b30fde298eed23afb61f7b6b99bf61&scene=21#wechat_redirect)

[2021看雪SDC议题回顾 | 基于模拟仿真的蓝牙协议栈漏洞挖掘](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458401340&idx=1&sn=668eb21cceec544c0cf85fe13878d249&chksm=b18f0ab686f883a0236bf76e83c6acd6a084ea4927686e35162807e29f0be5bd0f61005d5d1c&scene=21#wechat_redirect)

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

- End -

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

公众号ID：ikanxue

官方微博：看雪安全

商务合作：wsc@kanxue.com

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

点个在看你最好看

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

戳“阅读原文”查看议题完整PPT吧！

阅读原文

阅读 1584

​

写留言

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/1UG7KPNHN8EGLfh77kFmnicd9WOic2ibvhCibFdB4bL4srJCgo2wnvdoXLxpIvAkfCmmcptXZB0qKWMoIP8iaibYN2FA/300?wx_fmt=png&wxfrom=18)

看雪学苑

7分享在看

写留言

写留言

**留言**

暂无留言
