# [蜗窝科技](http://www.wowotech.net/)

### 慢下来，享受技术。

[![](http://www.wowotech.net/content/uploadfile/201401/top-1389777175.jpg)](http://www.wowotech.net/)

- [博客](http://www.wowotech.net/)
- [项目](http://www.wowotech.net/sort/project)
- [关于蜗窝](http://www.wowotech.net/about.html)
- [联系我们](http://www.wowotech.net/contact_us.html)
- [支持与合作](http://www.wowotech.net/support_us.html)
- [登录](http://www.wowotech.net/admin)

﻿

## 

作者：[wowo](http://www.wowotech.net/author/2 "runangaozhong@163.com") 发布于：2017-12-18 16:18 分类：[USB](http://www.wowotech.net/sort/usb)

## 1. 前言

从1996年1月USB1.0正式发布至今（2017年9月 USB3.2发布），USB已经走过了21个年头。在这21年的时间了，USB标准化组织（_USB_ Implementers Forum，USB-IF）折腾出来了各式各样、五花八门的接口形态：Type A、Type A SuperSpeed、Type B、Type B SuperSpeed、Mini-A、Mini-B、Micro-A、Micro-B、Micro-B SuperSpeed、Type C等等。

另外，USB接口主要由插座（Receptacle）、插头（Plug）和线缆（Cable）三部分组成，再叠加上这些奇奇怪怪的规范，灾难就发生了：

> A产品喜欢用Type A的插座，B产品偏偏喜欢Type B，连接它们的线缆就悲剧了，只能变成A-to-B的了。以此类推，A-to-A、B-to-B、A-to-MicroA、等等，于是我们的抽屉就挤满了各种不明用途的USB线……

好吧，吐槽时间结束，因为本文的主角不是过去的那些奇奇怪怪的接口，而是最新的、红到发紫的USB-C（也称作USB Type C）规范。提起typec，它还真和它的A、B前辈们不太一样：

> 因为它有自己独立的、自行演化的规范文件----USB Type-C Specification（2014年发8月布1.0版本，2017年7月发布1.3版本）。而前辈们就没有这样的待遇了，它们都依附于具体的USB规范（USB 1.0、USB 1.1、USB 2.0、等等）。

为什么会这样的呢？当然是因为它有独特之处了，具体请参考本文后续的描述。

## 2. 概述

我们接着上面的问题讲。

Type C之前的规范（Type A、Type B、等等），偏重于USB接口的“硬”的特性，如信号的个数、接口的形态、电气特性、等等，这些特性一旦固定，就没有更改的需求了，这就导致了：

> 1）这些接口规范不需要单独存在（因为没有更新、演化的要求），“随便”在USB规范的哪个章节交代一下就行了。
>
> 2）同时存在五花八门、种类繁多的接口（因为不能更新、演化啊，一旦新需求出现，只能再搞一个新的了）。

到USB Type C的时候，USB标准化组织的这些家伙突然开窍了（管他主动开窍还是被动开窍，反正是开窍了），在定义USB接口“硬”的特性的基础上，增加了一些“软”的内容，一下子就海阔天空了。至此，USB接口（仅仅指Type C）摆脱了和USB的从属关系，变成了一个可以和USB规范平起平坐的新规范。

大家估计会很好奇，这家伙到底Get了什么新技能，从而成功上位了呢？让我们简单总结一下（注意其中黄色高亮部分）：

> ▲ 定义一套新的接口形态（Receptacle/Plug/Cable）
>
> ▲ 插座（Receptacle）可以用在很薄的电子设备上，因为它的高度只有3mm
>
> ▲ 插头（Plug）更容易使用了，可以正着插、反着插、随便插、想怎么插怎么插，终于不再反人类了（想想之前，插一个U盘到电脑中：哦，好像插不进去，反过来试试；嗯？还是插不进去，再反过来试试；噢！终于插进去了……流汗中……）
>
> ▲ 线缆（Cable）也更容易使用了，两端一模一样（当然，为了兼容、转接旧有规范的除外），也是想怎么插就怎么插
>
> ▲ 插头（Plug）和线缆（Cable）的改进，并不是一个空手套白狼的买卖，是要付出代价的，因为需要一个称作“Configuration Process”的过程解决如下的两个问题：\
> □ 插头可以随便插，因而需要一套检测插入方向的机制，并可以通过插入方向动态的map管脚信号以便进行后续的通信\
> □ 线缆的两端一模一样，就无法区分所连接的两个USB设备的角色（Host or Device、等等），因而需要一套协商机制，以便让两端的USB设备进行角色的沟通
>
> ▲ 以上的“Configuration Process”是使用两个称作CC（CC1和CC2）的管脚进行的，利用不同电压，传递一些简单的信息，以满足上面的需求。\
> □ 后来，一个称作USB PD（Power Delivery）\[3\]的规范出现了，它在这两个管脚上实现了一种简单的、半双工的通信协议，以完成USB power供给有关协商（有关USB PD，可参考相应的规范\[3\]以及本站后续的文章）。可以说，这个通信协议就是打开新世界的钥匙。基于它，更多有意义的事情出现了（因此，USB Type C可以单独存在了），例如\
> □ 支持扩展功能。通过扩展功能，USB Type C接口拥有了无线的想象空间，可以摇身变成任意其它协议的物理接口，例如配件接口、音频接口、视频接口、debug接口等等，大有一统天下之势。从这个角度看，USB Type C不仅仅是成功上位（从USB规范中独立出来），而是成功逆袭（凌驾于USB规范之上），格局啊！！

对USB Type C有个基本的了解之后，我们再简单分析一下它的主要特性（主要从软件的角度，纯电气方面的内容直接插规范就行了，这里不再罗嗦）。

## 3. 主要特性

#### 3.1 接口形态（Receptacle/Plug/Cable）

为了实现自己的理想和抱负，USB Type C定义了新的接口形态。另外，为了兼容旧的接口以及一些特殊功能，它定义了不同形态的插座、插头、线缆。主要包括：

1）定义了2种Type-C的插座

> a）全功能的Type-C插座，可以用于支持USB2.0、USB3.1、等特性的平台和设备。\
> b）USB 2.0 Type-C插座，只可以用在支持USB2.0的平台和设备上。

2）定义了3种Type-C插头

> a）全功能的Type-C插头，可以用于支持USB2.0、USB3.1、等特性的平台和设备。\
> b）USB 2.0 Type-C插头，只可以用在支持USB2.0的平台和设备上。\
> c）USB Type-C Power-Only插头，用在那些只需要供电设备上（如充电器）。

3）定义了3种标准的Type-C线缆

> a）两端都是全功能Type-C插头的全功能Type-C线缆。\
> b）两端都是USB 2.0 Type-C插头的USB 2.0 Type-C线缆。\
> c）只有一端是Type-C插头（全功能Type-C插头或者USB 2.0 Type-C插头）的线缆。

4）为兼容旧设备而定义的线缆或者适配器

> a）一种线缆，一端是全功能的Type-C插头，另一端是USB 3.1 Type-A插头。\
> b）一种线缆，一端是USB 2.0 Type-C插头，另一端是USB 2.0 Type-A插头。\
> c）一种线缆，一端是全功能的Type-C插头，另一端是USB 3.1 Type-B插头。\
> d）一种线缆，一端是USB 2.0 Type-C插头，另一端是USB 2.0 Type-B插头。\
> e）一种线缆，一端是USB 2.0 Type-C插头，另一端是USB 2.0 Mini-B插头。\
> f）一种线缆，一端是全功能的Type-C插头，另一端是USB 3.1 Micro-B插头。\
> g）一种线缆，一端是USB 2.0 Type-C插头，另一端是USB 2.0 Micro-B插头。\
> h）一种适配器，一端是全功能的Type-C插头，另一端是USB 3.1 Type-A插座。\
> i）一种适配器，一端是USB 2.0 Type-C插头，另一端是USB 2.0 Micro-B插座。

注1：再吐槽一下，看了上面的a-i，应该能感受到之前的USB接口规范是多么的能折腾了吧？

#### 3.2 管脚及信号的定义

USB Type-C接口有24个管脚，插座和插头在管脚信号的定义上有一点点的不同，分别如下：

|   |   |   |   |   |   |   |   |   |   |   |   |
|---|---|---|---|---|---|---|---|---|---|---|---|
|A1|A2|A3|A4|A5|A6|A7|A8|A9|A10|A11|A12|

|   |   |   |   |   |   |   |   |   |   |   |   |
|---|---|---|---|---|---|---|---|---|---|---|---|
|GND|TX1+|TX1-|VBUS|CC1|D+|D-|SBU1|VBUS|RX2-|RX2+|GND|

|   |
|---|
||

|   |   |   |   |   |   |   |   |   |   |   |   |
|---|---|---|---|---|---|---|---|---|---|---|---|
|GND|RX1+|RX1-|VBUS|SBU2|D-|D+|CC2|VBUS|TX2-|TX2+|GND|

|   |   |   |   |   |   |   |   |   |   |   |   |
|---|---|---|---|---|---|---|---|---|---|---|---|
|B12|B11|B10|B9|B8|B7|B6|B5|B4|B3|B2|B1|

表格1：USB Type-C Receptacle Interface (Front View)

|   |   |   |   |   |   |   |   |   |   |   |   |
|---|---|---|---|---|---|---|---|---|---|---|---|
|A12|A11|A10|A9|A8|A7|A6|A5|A4|A3|A2|A1|

|   |   |   |   |   |   |   |   |   |   |   |   |
|---|---|---|---|---|---|---|---|---|---|---|---|
|GND|RX2+|RX2-|VBUS|SBU1|D-|D+|CC|VBUS|TX1-|TX1+|GND|

|   |
|---|
||

|   |   |   |   |   |   |   |   |   |   |   |   |
|---|---|---|---|---|---|---|---|---|---|---|---|
|GND|TX2+|TX2-|VBUS|VCONN|||SBU2|VBUS|RX1-|RX1+|GND|

|   |   |   |   |   |   |   |   |   |   |   |   |
|---|---|---|---|---|---|---|---|---|---|---|---|
|B1|B2|B3|B4|B5|B6|B7|B8|B9|B10|B11|B12|

表格2：USB Full-Featured Type-C Plug Interface (Front View)

以上信号按照功能可以分为5类：

> 1）Power有关的信号，包括\
> a）VBUS，USB线缆的bus power（和我们通常意义上VBUS保持一致）。\
> b）VCONN（只有在插头上才会有该信号），用于向插头供电（由此可以推测出有些插头中可能会有电路）。\
> c）GND，接地。
>
> 2）USB 2.0数据线，D+/D-。它们在插头端只有一对，和旧的USB 2.0规范一致。但为了支持正反随意插。在插座端定义了两组，这样插座端可以根据实际情况进行合适的mapping。
>
> 3）USB3.1数据线，TX+/-和RX+/-，用于高速的数据传输。插头和插座端都有两组，用于支持正反随意插。
>
> 4）用于Configuration的信号，对插头来说，只有一个CC，对插座来说，有两个CC1和CC2。
>
> 5）扩展功能所需的信号，具体使用场景由相应的扩展功能决定。
>
> 注2：对于3.1中所描述的不同类型的插座和插头，这24个管脚以及信号不一定全部使用，具体可参考USB Type-C的规范\[2\]。

注3：大家可能注意到了，USB Type-C 24个管脚信号中，Power类（GND/VBUS）和数据类（D+/D-/TX/RX）是完全对称的（对Power来说，无论怎么插，都是一样；对数据线来说，简单的路由一下，就可以工作）。剩下的，包括CC、SBU和VCONN，用于方向、线类型等检测，具体可参考后面的介绍。

#### 3.3 USB port的Data Role

在USB 2.0及以前的时代，根据功能的不同，USB端口分为Host、Device、OTG（Host和Device二者皆可）、Hub等。在Type C时代，事情的本质还在，不过名字稍微有些调整，分别称作：

> DFP（Downstream Facing Port），一般作为Host或者Hub，在初始配置下通过VBUS或者VCONN向device供电。
>
> UFP（Upstream Facing Port），一般作为Device或者Hub，连接到Host上，初始配置下通过VBUS或者VCONN由Host供电。
>
> DRD(Dual-Role-Data)，类似于以前的OTG，既可以作为DFP，也可以作为UFP。设备刚连接时作为哪一种角色，由端口的Power Role（参考后面的介绍）决定；后续也可以通过data role switch过程更改（如果支持USB PD协议的话）。

#### 3.4 USB port的Power Role

根据USB port的供电（或者受电）情况，USB type c将port划分为Source、Sink等power角色，具体如下：

> Source，通过VBUS或者VCONN供电。
>
> Sink，通过VBUS或者VCONN接受供电。
>
> DRP(Dual-Role-Power)，既可以作为Source，也可以作为Sink。到底作为Source还是Sink，由设备连接后的Configuration Process（具体可参考后面的介绍）以及后续的power role switch过程决定。

#### 3.5 连接检测

USB Type-C的连接检测包括3部分的内容：

> 连接检测；
>
> 连接方向检测；
>
> Power Role检测。

这三部分都是通过USB Type-C接口的CC（CC1和CC2）管脚进行的，原理总结如下（具体细节请参考\[2\]中的说明，本文不详细罗列了）：

1）参考3.2小节的说明，USB Type-C的插座有两个CC（CC1和CC2），而插头（或者说USB cable），有两种情况：

> ▲ 正常情况下，只有一个CC，两个Type-C端口连接到一起之后，只会存在一个CC连接。通过检测哪一个CC有连接，就可以判断连接的方向。
>
> ▲ 如果是可供电的USB cable（Powered cable），一个用做CC，另一个用作Vconn，检测方式参考后面的介绍。

2）不同功能的USB port，需要在CC1和CC2管脚上接不同的上拉或者下拉电阻，规则如下：

> Source需要在CC1和CC2管脚接上拉电阻Rp（也可以使用电流源取代，本文就不介绍了），Rp的值可参考\[2\]中“Table 4-20 Source CC Termination (Rp) Requirements”的介绍。
>
> Sink需要在CC1和CC2管脚接下拉电阻Rd，Rd的值可参考\[2\]中“Table 4-21 Sink CC Termination (Rd) Requirements”的介绍。
>
> 可以通过VCONN供电的USB电缆（Powered cable）为了让Source检测到（以便通过VCONN为它供电），需要在CC脚接下拉电阻Rd，并在Vconn串接一个电阻Ra（代表Vconn的负载）。
>
> 其它配件的规则，不再介绍了（可参考\[2\]中有关的章节）。

3）基于上面的规则，Source需要根据CC1和CC2的状态（阻值）检测Sink的连接及方向，规则如下：

> 如果CC1和CC2均为Open状态，表示没有Sink连接。
>
> 如果CC1和CC2中的一个的阻值为Rd（除去自己的Rp或者电流源的阻值）、另一个为Open，表示有Sink连接，连接的方向由哪一个CC为Rd决定。
>
> 如果CC1和CC2中的一个的阻值为Ra（除去自己的Rp或者电流源的阻值）、另一个为Open，表示有不带Sink的Powered cable连接，连接的方向由哪一个CC为Ra决定。
>
> 如果一个CC的阻值为Ra，另一个为Rd，则表示有带SInk的Powered cable连接，连接方向由哪个CC为Ra（Rd）决定。

4）Source检测到Sink的连接（及方向）后，会向Vbus和Vconn供电。之后：

> Sink通过检测Vbus来确定Source的连接。
>
> Sink检测到连接后，同样根据CC管脚的状态检测连接方向。

5）Source检测到不带Sink的Powered cable的连接后，不会向Vbus或者Vconn供电。

6）Source检测到带Sink的Powered cable的连接（及方向）后，会向Vbus和Vconn供电，Sink的行为后上面4)类似。

#### 3.6 Power role检测

上面的过程，是基于某个port作为Source，另一个port作为Sink为假设的。实际上，某一个Type-C port可能是DRP port，怎么办呢？简单：

> 它可以在Source和Sink两个状态来回切换，在某一个状态下的检测逻辑，和上面3.5描述的完全一样。
>
> 当在Source状态下，成功的检测到Sink的连接时，则自己作为Source；当在Sink状态下成功检测到Source的连接时，则自己作为Sink。

另外，Type-C规范也提供了一个后悔药，虽然我是DRP（Source或者Sink都可以支持），但我更倾向于作为Source（或Sink）怎么办？好办：

> 当在动态检测的过程中，自己变成了一个自己不喜欢的角色，还可以执行一个Try.SRC或者Try.SNK的过程，尝试变成自己喜欢的角色。如果碰巧对方认可你作为这个角色，OK，万事大吉。

注4：到目前为止（在没有USB PD协议参与的情况下），USB Type-C port的data role必须和power role保持一致（因为Type-C规范没有提供多余的机制去单独的协商data role）。

#### 3.7 USB PD协议

正确连接后，双方通过CC管脚，使用USB PD（Power Delivery）协议，可以进行后续配置和操作，包括：

> Data Role的切换（类似USB OTG的切换）；
>
> Power Role的切换；
>
> Vconn role的切换（谁通过Vconn向Powered cable供电）；
>
> 供电的调整；
>
> 等等（具体可参考\[3\]以及后续有关USB PD协议的介绍）。

## 4. 参考文档

\[1\] [https://en.wikipedia.org/wiki/USB#2.0](https://en.wikipedia.org/wiki/USB#2.0 "https://en.wikipedia.org/wiki/USB#2.0")

\[2\] USB Type-C Specification Release 1.3.pdf

\[3\] USB_PD_R3_0 V1.1 20170112.pdf

原创文章，转发请注明出处。蜗窝科技，[www.wowotech.net](http://www.wowotech.net/usb/usb_type_c_overview.html)。

标签: [Power](http://www.wowotech.net/tag/Power) [USB](http://www.wowotech.net/tag/USB) [typec](http://www.wowotech.net/tag/typec) [type-c](http://www.wowotech.net/tag/type-c) [3.1](http://www.wowotech.net/tag/3.1) [pd](http://www.wowotech.net/tag/pd)

[![](http://www.wowotech.net/content/uploadfile/201605/ef3e1463542768.png)](http://www.wowotech.net/support_us.html)

« [中断唤醒系统流程](http://www.wowotech.net/irq_subsystem/irq_handle_procedure.html) | [进程切换分析（3）：同步处理](http://www.wowotech.net/process_management/scheudle-sync.html)»

**评论：**

**hczx123**\
2020-06-04 15:06

本来想在论坛发贴求助的，发现帐号忘了，想重新注册个帐号，没想到现在不允许注册了。。

最近在搞UAC这块，感觉介绍这方面的文档不多，不知道蜗窝能不能写写这方面的，还有\
就是想找音视频相关的知识，发现这里没有这个版块，提个建议加下这个版块。

最后再表示感谢，这里对我的帮助真的很大，讲的比较深入，然后又容易懂。

[回复](http://www.wowotech.net/usb/usb_type_c_overview.html#comment-8013)

**[wowo](http://www.wowotech.net/)**\
2020-06-06 16:18

@hczx123：现在没有多少时间能写文章，抱歉！~

[回复](http://www.wowotech.net/usb/usb_type_c_overview.html#comment-8014)

**Ryang**\
2019-06-20 17:27

写的真好，现在很期待关于PD协议的文章，不知道什么时候有啊。求发~

[回复](http://www.wowotech.net/usb/usb_type_c_overview.html#comment-7480)

**Cardinal**\
2019-03-17 21:35

看了你给的plug和receptacle的图，我把自己给绕进去了。\
plug上一端的TX在对应的另外一端就是RX， 这么想是对的吧？

[回复](http://www.wowotech.net/usb/usb_type_c_overview.html#comment-7314)

**Jagger**\
2018-10-02 23:21

Wowo, 我看了你的文章，觉得你的思路和表达比其他博主，清晰太多了。

而且你应该都是直接从英文标准里面自己总结出来的吧！赞赞赞！

千万不要停啊， 我自己看英文资料实在是理解不了。。。

[回复](http://www.wowotech.net/usb/usb_type_c_overview.html#comment-6967)

**长夜将尽**\
2018-07-31 16:28

wowo，你好！非常幸运能发现你的这个小窝，获益良多。 关于USB和显示这块，不知道后续什么时候还能再更新？

[回复](http://www.wowotech.net/usb/usb_type_c_overview.html#comment-6860)

**[wowo](http://www.wowotech.net/)**\
2018-08-01 08:54

@长夜将尽：谢谢鼓励啊~\
其实最近几个月更新都很少（不止USB和显示），郭大侠和我这一段时间都太忙了，挤不出时间写。等再过一段时间稍微不忙一些了，我们再继续~

[回复](http://www.wowotech.net/usb/usb_type_c_overview.html#comment-6862)

**oldman**\
2018-05-09 16:20

在某个unix系统上做了10+年的USB driver，从libusb到device class，到host，基本上所有主要的设备都研究过。再换工作的时候发现，没人需要这个玩意，只好果断放弃10多年的积累，投入Linux kernel的怀抱。没想到在这里还能看到需求！

[回复](http://www.wowotech.net/usb/usb_type_c_overview.html#comment-6742)

**小明**\
2018-01-17 17:03

找了半天没找到，原来是没写...快写一个吧，可以由浅入深，由框架到细节。期待

[回复](http://www.wowotech.net/usb/usb_type_c_overview.html#comment-6483)

**lehand**\
2018-01-08 13:26

期待USB的后续内容

[回复](http://www.wowotech.net/usb/usb_type_c_overview.html#comment-6458)

**EE**\
2017-12-20 16:54

wowo,你肚子到底有多少货，牛，蓝牙不知能否还有后续，其实好期待的：）

[回复](http://www.wowotech.net/usb/usb_type_c_overview.html#comment-6392)

**[wowo](http://www.wowotech.net/)**\
2017-12-21 09:09

@EE：哪里哪里，我都是边学边写。其实这些文章不是写给大伙看的，而是给自己看得。蓝牙的话，应该会接着写，但没有多么明确的目标。

[回复](http://www.wowotech.net/usb/usb_type_c_overview.html#comment-6397)

**EE**\
2017-12-21 09:40

@wowo：恩恩：），期待：）

[回复](http://www.wowotech.net/usb/usb_type_c_overview.html#comment-6400)

**sunracle**\
2017-12-19 17:04

看来窝窝的手已经伸向了usb，期待usb子系统大作

[回复](http://www.wowotech.net/usb/usb_type_c_overview.html#comment-6390)

**[wowo](http://www.wowotech.net/)**\
2017-12-21 09:08

@sunracle：我是随便写的哈，最近时间不是特别多，USB需要写好久（好好写的话，估计得一年半载），不好办啊:-(

[回复](http://www.wowotech.net/usb/usb_type_c_overview.html#comment-6396)

1 [2](http://www.wowotech.net/usb/usb_type_c_overview.html/comment-page-2#comments)

**发表评论：**

昵称

邮件地址 (选填)

个人主页 (选填)

![](http://www.wowotech.net/include/lib/checkcode.php)

- ### 站内搜索

  蜗窝站内  互联网

- ### 功能

  [留言板\
  ](http://www.wowotech.net/message_board.html)[评论列表\
  ](http://www.wowotech.net/?plugin=commentlist)[支持者列表\
  ](http://www.wowotech.net/support_list)

- ### 最新评论

  - ja\
    [@dream：我看完這段也有相同的想法，引用 @dream ...](http://www.wowotech.net/kernel_synchronization/spinlock.html#8922)
  - 元神高手\
    [围观首席power managerment专家](http://www.wowotech.net/pm_subsystem/device_driver_pm.html#8921)
  - 十七\
    [内核空间的映射在系统启动时就已经设定好，并且在所有进程的页表...](http://www.wowotech.net/process_management/context-switch-arch.html#8920)
  - lw\
    [sparse模型和disconti模型没看出来有什么本质区别...](http://www.wowotech.net/memory_management/memory_model.html#8919)
  - 肥饶\
    [一个没设置好就出错](http://www.wowotech.net/linux_kenrel/516.html#8918)
  - orange\
    [点赞点赞，对linuxer的文章总结到位](http://www.wowotech.net/device_model/dt-code-file-struct-parse.html#8917)

- ### 文章分类

  - [Linux内核分析(25)](http://www.wowotech.net/sort/linux_kenrel) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=4)
    - [统一设备模型(15)](http://www.wowotech.net/sort/device_model) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=12)
    - [电源管理子系统(43)](http://www.wowotech.net/sort/pm_subsystem) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=13)
    - [中断子系统(15)](http://www.wowotech.net/sort/irq_subsystem) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=14)
    - [进程管理(31)](http://www.wowotech.net/sort/process_management) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=15)
    - [内核同步机制(26)](http://www.wowotech.net/sort/kernel_synchronization) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=16)
    - [GPIO子系统(5)](http://www.wowotech.net/sort/gpio_subsystem) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=17)
    - [时间子系统(14)](http://www.wowotech.net/sort/timer_subsystem) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=18)
    - [通信类协议(7)](http://www.wowotech.net/sort/comm) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=20)
    - [内存管理(31)](http://www.wowotech.net/sort/memory_management) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=21)
    - [图形子系统(2)](http://www.wowotech.net/sort/graphic_subsystem) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=23)
    - [文件系统(5)](http://www.wowotech.net/sort/filesystem) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=26)
    - [TTY子系统(6)](http://www.wowotech.net/sort/tty_framework) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=27)
  - [u-boot分析(3)](http://www.wowotech.net/sort/u-boot) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=25)
  - [Linux应用技巧(13)](http://www.wowotech.net/sort/linux_application) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=3)
  - [软件开发(6)](http://www.wowotech.net/sort/soft) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=1)
  - [基础技术(13)](http://www.wowotech.net/sort/basic_tech) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=6)
    - [蓝牙(16)](http://www.wowotech.net/sort/bluetooth) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=10)
    - [ARMv8A Arch(15)](http://www.wowotech.net/sort/armv8a_arch) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=19)
    - [显示(3)](http://www.wowotech.net/sort/display) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=22)
    - [USB(1)](http://www.wowotech.net/sort/usb) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=28)
  - [基础学科(10)](http://www.wowotech.net/sort/basic_subject) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=7)
  - [技术漫谈(12)](http://www.wowotech.net/sort/tech_discuss) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=8)
  - [项目专区(0)](http://www.wowotech.net/sort/project) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=9)
    - [X Project(28)](http://www.wowotech.net/sort/x_project) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=24)

- ### 随机文章

  - [Linux kernel debug技巧----开启DEBUG选项](http://www.wowotech.net/linux_application/kernel_debug_enable.html)
  - [X-025-KERNEL-Linux gpio driver的移植之基本功能](http://www.wowotech.net/x_project/kernel_gpio_driver_porting_1.html)
  - [Linux PM QoS framework(3)\_per-device PM QoS](http://www.wowotech.net/pm_subsystem/per_device_pm_qos.html)
  - [Linux DMA Engine framework(1)\_概述](http://www.wowotech.net/linux_kenrel/dma_engine_overview.html)
  - [Perf book 9.3章节翻译（下）](http://www.wowotech.net/kernel_synchronization/perfbook-9-3-rcu.html)

- ### 文章存档

  - [2024年2月(1)](http://www.wowotech.net/record/202402)
  - [2023年5月(1)](http://www.wowotech.net/record/202305)
  - [2022年10月(1)](http://www.wowotech.net/record/202210)
  - [2022年8月(1)](http://www.wowotech.net/record/202208)
  - [2022年6月(1)](http://www.wowotech.net/record/202206)
  - [2022年5月(1)](http://www.wowotech.net/record/202205)
  - [2022年4月(2)](http://www.wowotech.net/record/202204)
  - [2022年2月(2)](http://www.wowotech.net/record/202202)
  - [2021年12月(1)](http://www.wowotech.net/record/202112)
  - [2021年11月(5)](http://www.wowotech.net/record/202111)
  - [2021年7月(1)](http://www.wowotech.net/record/202107)
  - [2021年6月(1)](http://www.wowotech.net/record/202106)
  - [2021年5月(3)](http://www.wowotech.net/record/202105)
  - [2020年3月(3)](http://www.wowotech.net/record/202003)
  - [2020年2月(2)](http://www.wowotech.net/record/202002)
  - [2020年1月(3)](http://www.wowotech.net/record/202001)
  - [2019年12月(3)](http://www.wowotech.net/record/201912)
  - [2019年5月(4)](http://www.wowotech.net/record/201905)
  - [2019年3月(1)](http://www.wowotech.net/record/201903)
  - [2019年1月(3)](http://www.wowotech.net/record/201901)
  - [2018年12月(2)](http://www.wowotech.net/record/201812)
  - [2018年11月(1)](http://www.wowotech.net/record/201811)
  - [2018年10月(2)](http://www.wowotech.net/record/201810)
  - [2018年8月(1)](http://www.wowotech.net/record/201808)
  - [2018年6月(1)](http://www.wowotech.net/record/201806)
  - [2018年5月(1)](http://www.wowotech.net/record/201805)
  - [2018年4月(7)](http://www.wowotech.net/record/201804)
  - [2018年2月(4)](http://www.wowotech.net/record/201802)
  - [2018年1月(5)](http://www.wowotech.net/record/201801)
  - [2017年12月(2)](http://www.wowotech.net/record/201712)
  - [2017年11月(2)](http://www.wowotech.net/record/201711)
  - [2017年10月(1)](http://www.wowotech.net/record/201710)
  - [2017年9月(5)](http://www.wowotech.net/record/201709)
  - [2017年8月(4)](http://www.wowotech.net/record/201708)
  - [2017年7月(4)](http://www.wowotech.net/record/201707)
  - [2017年6月(3)](http://www.wowotech.net/record/201706)
  - [2017年5月(3)](http://www.wowotech.net/record/201705)
  - [2017年4月(1)](http://www.wowotech.net/record/201704)
  - [2017年3月(8)](http://www.wowotech.net/record/201703)
  - [2017年2月(6)](http://www.wowotech.net/record/201702)
  - [2017年1月(5)](http://www.wowotech.net/record/201701)
  - [2016年12月(6)](http://www.wowotech.net/record/201612)
  - [2016年11月(11)](http://www.wowotech.net/record/201611)
  - [2016年10月(9)](http://www.wowotech.net/record/201610)
  - [2016年9月(6)](http://www.wowotech.net/record/201609)
  - [2016年8月(9)](http://www.wowotech.net/record/201608)
  - [2016年7月(5)](http://www.wowotech.net/record/201607)
  - [2016年6月(8)](http://www.wowotech.net/record/201606)
  - [2016年5月(8)](http://www.wowotech.net/record/201605)
  - [2016年4月(7)](http://www.wowotech.net/record/201604)
  - [2016年3月(5)](http://www.wowotech.net/record/201603)
  - [2016年2月(5)](http://www.wowotech.net/record/201602)
  - [2016年1月(6)](http://www.wowotech.net/record/201601)
  - [2015年12月(6)](http://www.wowotech.net/record/201512)
  - [2015年11月(9)](http://www.wowotech.net/record/201511)
  - [2015年10月(9)](http://www.wowotech.net/record/201510)
  - [2015年9月(4)](http://www.wowotech.net/record/201509)
  - [2015年8月(3)](http://www.wowotech.net/record/201508)
  - [2015年7月(7)](http://www.wowotech.net/record/201507)
  - [2015年6月(3)](http://www.wowotech.net/record/201506)
  - [2015年5月(6)](http://www.wowotech.net/record/201505)
  - [2015年4月(9)](http://www.wowotech.net/record/201504)
  - [2015年3月(9)](http://www.wowotech.net/record/201503)
  - [2015年2月(6)](http://www.wowotech.net/record/201502)
  - [2015年1月(6)](http://www.wowotech.net/record/201501)
  - [2014年12月(17)](http://www.wowotech.net/record/201412)
  - [2014年11月(8)](http://www.wowotech.net/record/201411)
  - [2014年10月(9)](http://www.wowotech.net/record/201410)
  - [2014年9月(7)](http://www.wowotech.net/record/201409)
  - [2014年8月(12)](http://www.wowotech.net/record/201408)
  - [2014年7月(6)](http://www.wowotech.net/record/201407)
  - [2014年6月(6)](http://www.wowotech.net/record/201406)
  - [2014年5月(9)](http://www.wowotech.net/record/201405)
  - [2014年4月(9)](http://www.wowotech.net/record/201404)
  - [2014年3月(7)](http://www.wowotech.net/record/201403)
  - [2014年2月(3)](http://www.wowotech.net/record/201402)
  - [2014年1月(4)](http://www.wowotech.net/record/201401)

[![订阅Rss](http://www.wowotech.net/content/templates/default/images/rss.gif)](http://www.wowotech.net/rss.php "RSS订阅")

Copyright @ 2013-2015 [蜗窝科技](http://www.wowotech.net/ "wowotech") All rights reserved. Powered by [emlog](http://www.emlog.net/ "采用emlog系统")
