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

作者：[linuxer](http://www.wowotech.net/author/3 "linuxer") 发布于：2015-4-9 18:13

一、前言

一般工程师的直觉是当提升了CPU的运行频率，那么性能应该是呈现线性的关系，例如如果CPU跑260MHz，当降低到130MHz后，其性能应该会降低一半。实际情况如何呢？我们来做一个实验看看。

二、实验

1、实验环境介绍

硬件平台当然是强大的，生命期超长，死而复生（2010年就phase out了，但仍然有一批人马顽强的在上面开发）的AD6900处理器了。除此之外，还需要一台示波器用来测量时间。

2、在kernel中测试

实验方法如下：\
（1）设置AD6900 ARM core所需的clock（260、130，65，32.5）

（2）使用linux中udelay函数（当然，kernel中udelay是和loops per jiffy相关，我们需要稍加修改，让他仅在260MHz的时候能够正确的delay）的方式进行延时

（3）通过GPIO翻转来确定起始和结束的时间点

（4）用示波器测量方波脉宽，记录实验数据。

实验结果如下：

|   |   |
|---|---|
|clock|udelay的执行时间|
|260|2.5ms|
|130|5.0ms|
|65|10ms|
|32.5|20ms|

基本上，在260MHz中执行2.501ms的程序，在130MHz上执行了5.081ms的时间，在32.5MHz上执行了20.64ms，基本上CPU运行clock和程序执行时间呈现线性的关系。

3、在bootloader中测试

同样的实验，我们在bootloader中重复进行，udelay有两个版本，一个是c语言的版本，一开始代码如下：

> void delay( void )\
> {\
> int cnt = 20000;\
> while(cnt—);\
> }

编译之后，delay函数没有任何效果。当然由于开了O2的优化，实际上上面的函数都被优化掉了，要不改成全局变量试一试：

> int cnt = 20000;\
> void delay( void )\
> {\
> while(cnt--);\
> }

情况依然，由于强大的编译器知道delay函数执行到最后cnt等于－1，因此，在delay函数中就偷懒，直接给cnt赋值－1，优化掉了--操作和while判断，当然，给cnt变量加上volatile的修饰符应该是OK了，我们采用了简单的去掉O2优化的方法，得到了下面的结果：

|   |   |
|---|---|
|clock|udelay的执行时间|
|260|2.610ms|
|130|3.720ms|
|65|4.630ms|
|32.5|9.242ms|

好象结果已经不是那么线性了。怎么回事，要不用汇编写，这样会更直接一些，代码如下：

> 400202f4 \<\_\_delay>:\
> 400202f4:    e2500001     subs    r0, r0, #1    ; 0x1\
> 400202f8:    8afffffd     bhi    400202f4 \<\_\_delay>

当然，在执行\_\_delay函数之前，要传递初值给r0寄存器。OK，代码已经完全和linux中的一致了，测试结果如下：

|   |   |
|---|---|
|clock|udelay的执行时间|
|260|2.5ms|
|130|2.5ms|
|65|2.5ms|
|32.5|5ms|

呵呵，结果多么的神奇。

三、分析

1、和程序执行相关的HW block如下图所示：

[![cpu block](http://www.wowotech.net/content/uploadfile/201504/06804468f393a81db7d21b0cb5cfe3b620150409101221.gif "cpu block")](http://www.wowotech.net/content/uploadfile/201504/056b277eb3bd5305ed2a0afd59cfa81020150409101109.gif)

上图只是画出和本次实验有关的block。

2、kernel中的程序执行过程描述

内核态的delay函数虽然代码位于SDRAM，但是由于：

（1）TLB已经保存了地址映射

（2）指令已经被加载到cacheline

因此，实际上程序的执行基本是是限制在了ARM core内，不需要通过bus访问类似SRAM或者SDRAM的bus slave设备。这时候，增加ARM core的频率可以获取线性的程序性能提升。

3、bootloader中的程序执行过程描述

由于种种原因，我们的bootload中没有打开cache，没有启动MMU，而且bootloader中的代码在System RAM中运行。具体clock配置如下：

|   |   |   |
|---|---|---|
|ARM clock|BUS clock|udelay的执行时间|
|260|65|2.5ms|
|130|65|2.5ms|
|65|65|2.5ms|
|32.5|32.5|5ms|

由于ARM core需要每次去system RAM中取指，虽然ARM core可以运行在260MHz下，但是由于BUS block限制在65MHz上，因此，大部分的时间，ARM core处于饥饿状态，这种状态下，什么流水线、什么out-of-order，什么分支预测都是浮云，性能的瓶颈来自上图中的A bus。因此，在260、130和65MHz下得到了相同的测试结果。

我们再来看看c代码版本的情况，使用objdump看看实际执行的代码是怎样的：

> 400202b0:    e59f3028     ldr    r3, \[pc, #40\]    ; 400202e0 \<.text+0x2e0>\
> 400202b4:    e5933000     ldr    r3, \[r3\]\
> 400202b8:    e2432001     sub    r2, r3, #1    ; 0x1\
> 400202bc:    e59f301c     ldr    r3, \[pc, #28\]    ; 400202e0 \<.text+0x2e0>\
> 400202c0:    e5832000     str    r2, \[r3\]\
> 400202c4:    e59f3014     ldr    r3, \[pc, #20\]    ; 400202e0 \<.text+0x2e0>\
> 400202c8:    e5933000     ldr    r3, \[r3\]\
> 400202cc:    e3730001     cmn    r3, #1    ; 0x1\
> 400202d0:    1afffff6     bne    400202b0

汇编代码就2条指令，反复执行，c版本的看起来要复杂多了。为何c版本的delay函数没有出现和汇编版本的delay一样的测试结果呢？（TODO，我们留给大家思考吧）

PS：能完成这篇文档，我首先要感谢CCTV，MTV，各种TV，还有华南区linux kernel首席团队的LGR同学，多谢他的实验数据。

_原创文章，转发请注明出处。蜗窝科技_

标签: [ARM性能](http://www.wowotech.net/tag/ARM%E6%80%A7%E8%83%BD)

[![](http://www.wowotech.net/content/uploadfile/201605/ef3e1463542768.png)](http://www.wowotech.net/support_us.html)

« [linux usb 摄像头测试](http://www.wowotech.net/162.html) | [linux 串口调试方法](http://www.wowotech.net/159.html)»

**评论：**

**泥人**\
2018-07-16 10:50

资料不公开，是否可以问个问题？！呵呵\
AD6900有片上BootROM吗？

谢谢。\
文章时间有些久，不知还有响应吗？

[回复](http://www.wowotech.net/arm-performance-test.html#comment-6847)

**1**\
2018-01-11 09:01

博主使用的AD6900是否有资料可以分享，最近正在学习其开发，但由于芯片太老，网上找不到资料。

[回复](http://www.wowotech.net/arm-performance-test.html#comment-6461)

**[linuxer](http://www.wowotech.net/)**\
2018-01-11 22:00

@1：抱歉，那些资料都是签了NDA的，不适合外传的。

[回复](http://www.wowotech.net/arm-performance-test.html#comment-6463)

**1**\
2018-01-12 11:38

@linuxer：原来是内部资料，难怪网上找不到呢，那方便请教一个关于其DSP MMR的问题吗？

[回复](http://www.wowotech.net/arm-performance-test.html#comment-6465)

**femrat**\
2015-09-17 12:46

那个。。。cnt不是等于-1吗。。。。。。

[回复](http://www.wowotech.net/arm-performance-test.html#comment-2599)

**[linuxer](http://www.wowotech.net/)**\
2015-09-17 23:17

@femrat：哪个地方cnt等于-1？能不能说的再清楚一些？

[回复](http://www.wowotech.net/arm-performance-test.html#comment-2601)

**femrat**\
2015-09-17 23:20

@linuxer：不知道怎么描述位置。。抄一段原文… “情况依然，由于强大的编译器知道delay函数执行到最后cnt等于0，因此，在delay函数中就偷懒，直接给cnt赋值0”

[回复](http://www.wowotech.net/arm-performance-test.html#comment-2602)

**[linuxer](http://www.wowotech.net/)**\
2015-09-17 23:28

@femrat：OK，知道你说的哪个地方了，这份文档的代码和思路都是另外一位同事做的，我是帮忙整理的文章。我觉得你说的是对的，不过，明天我还是用arm-linux-gcc验证一下

[回复](http://www.wowotech.net/arm-performance-test.html#comment-2604)

**femrat**\
2015-09-18 02:35

@linuxer：感谢文章~另外我真不是强迫症QAQ

[回复](http://www.wowotech.net/arm-performance-test.html#comment-2605)

**[linuxer](http://www.wowotech.net/)**\
2015-09-18 12:23

@femrat：我验证过了，你是对的。这当然不是强迫症，是你深厚的编程功力的体现，多谢拉。

另外，我review了这个错误发生的原因：\
while(--cnt);对应的汇编是 mov     r2, #0\
while(cnt--);对应的汇编是 mvn     r2, #0

我猜测当初不认真，把mvn看成mov了，^\_^

[回复](http://www.wowotech.net/arm-performance-test.html#comment-2611)

**femrat**\
2015-09-18 13:26

@linuxer：：-D  再次感谢文章～

**buhui**\
2015-08-08 14:33

希望蜗蜗首席多发表些Linux内核方面的好文章，我们都是爱技术的人。

[回复](http://www.wowotech.net/arm-performance-test.html#comment-2438)

**wowo**\
2015-08-09 17:57

@buhui：多谢鼓励，我们会尽力的，希望常来多交流。

[回复](http://www.wowotech.net/arm-performance-test.html#comment-2440)

**[Adam_QSI](http://www.wowotech.net/)**\
2015-05-04 14:14

好文章，希望能有对比较新的armv7 armv8架构的性能分析

[回复](http://www.wowotech.net/arm-performance-test.html#comment-1691)

**perr**\
2015-04-14 19:48

站长对DRM有研究否?\
最近,我一直为这个东西犯愁:\
1.不知道怎么去搭建OpenGL库(mesa)的测试环境\
2.找不到会mesa->libdrm->DRM(kernel)的华人/中文社区,发了封邮件,结果maintainer不理我\
3.缺少drm的入门导引的资料(单纯代码来看,对我来说有些复杂)\
我该肿么办呢?我想加入你们团队,自己摸索很无聊.

[回复](http://www.wowotech.net/arm-performance-test.html#comment-1501)

**[wowo](http://www.wowotech.net/)**\
2015-04-14 21:19

@perr：perr，非常抱歉，恐怕不能帮到您。虽然我的工作内容是显示，但关于GPU的理解，还是比较浅，以至于无法很好的描述出来。\
虽然后续有计划整理一下显示子系统的内容，但近期应该没有时间弄。\
我能给您的建议就是，做这件事情之前，先厘清如下的问题：

1. 您的目的是什么？爱好？学习？工作？…
1. 您拥有哪些资源（硬件环境、软件环境、等等）？能不能把这些东西以block图的形式呈现出来？
1. 您的近期目标是什么？远期目标是什么？能不能把这些目标拆分？
1. 先不要关注细节，把东西run起来之后再说。因此着重点是模块之间的连接方式。\
   最后，如果您不介意，我可以在小站上给您开一个专题，记录您做这些事情的详细步骤，遇到问题的话，大家可以一起讨论，集思广益。先开动起来，把问题一个一个抛出来，总能解决，最怕的就是，不知道问题是什么。

[回复](http://www.wowotech.net/arm-performance-test.html#comment-1502)

**[RobinHsiang](http://www.wowotech.net/)**\
2015-04-23 16:14

@wowo：请教华南区首席Linux PM\\显示专家一个问题，我现在要通过网络远程获取LCD的显示情况，我主要是要确保LCD有在正常显示。客户端是运行安卓的ARM平台。\
我现在能通过网络截图发送到服务器上供查看，但就算客户端的LCD不接上，我依然能从framebuffer里面读取数据。\
针对这个问题，首席专家有建议没？

[回复](http://www.wowotech.net/arm-performance-test.html#comment-1567)

**[wowo](http://www.wowotech.net/)**\
2015-04-23 17:15

@RobinHsiang：哈哈，什么首席不首席的，让您见笑了。我的观点是：除非你外加一个摄像头，实时监控，实时分析（人眼或者图像识别算法），才能确百分之百的万无一失。\
当然，抛开“万无一失”的解法，我们可以换个思路：\
怎样在LCD没有接上、损坏、以及其它异常的情况下，让客户端的软件先检测到？这可能需要根据实际情况，寻找具体的解决方案。

[回复](http://www.wowotech.net/arm-performance-test.html#comment-1570)

**[RobinHsiang](http://www.wowotech.net/)**\
2015-04-23 23:08

@wowo：哈哈，我这几天都跟同事吹牛，说我最近经常跟南中国地区的首席linux专家请教讨论问题。\
关于上面这个问题，我其实也硬件的软件的想了一些，我再想几天了再来跟你请教。\
这就像风清扬教令狐冲武功，师傅指点了我得思考一下。不过我没令狐冲那资质，转的慢 ^\_^

[回复](http://www.wowotech.net/arm-performance-test.html#comment-1584)

**[printk](http://www.wowotech.net/)**\
2015-04-23 17:32

@wowo：wowo也是做显示的阿。

[回复](http://www.wowotech.net/arm-performance-test.html#comment-1574)

**wowo**\
2015-04-11 11:45

骚年们，你们这一个260M的单核CPU，加上一个顶多130M的DDR，还整天调频来调频去的，汗！！！没法说你们了……哈哈。\
PS：个人意见，在个人电脑、平板、智能手机等领域硬件指标太强悍了，例如1.6G 4核，720M DDR，才有动态调频的需求，因为确认可以省很多power，而且大部分场景用不着跑这么快。\
至于其它的非主流产品，都给你省下来你能省多少啊？

[回复](http://www.wowotech.net/arm-performance-test.html#comment-1492)

**[tigger](http://www.wowotech.net/)**\
2015-04-13 16:08

@wowo：跑分的时候一定要跑得快

[回复](http://www.wowotech.net/arm-performance-test.html#comment-1496)

**wowo**\
2015-04-13 16:58

@tigger：哈哈，tigger不小心说实话了。

[回复](http://www.wowotech.net/arm-performance-test.html#comment-1497)

**[tigger](http://www.wowotech.net/)**\
2015-04-14 09:47

@wowo：大家都知道啊，无论是手机厂商还是芯片厂商都心照不宣

[回复](http://www.wowotech.net/arm-performance-test.html#comment-1498)

**[动漫资讯](http://www.mandaokuangtu.com/)**\
2015-04-10 23:07

买了一个二手电脑，cpu不好

[回复](http://www.wowotech.net/arm-performance-test.html#comment-1491)

**[tianhello](http://www.wowotech.net/)**\
2015-04-09 18:40

赞一个！\
"2、在bootloader中测试"标号是不是写错了；\
“一个是c语音的版本”，c语言误写成c语音了。\
PS：我没有强迫症。。。

[回复](http://www.wowotech.net/arm-performance-test.html#comment-1486)

**[linuxer](http://www.wowotech.net/)**\
2015-04-10 04:43

@tianhello：多谢指正！已经修改

[回复](http://www.wowotech.net/arm-performance-test.html#comment-1487)

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

  - Shiina\
    [一个电路（circuit）中，由于是回路，所以用电势差的概念...](http://www.wowotech.net/basic_subject/voltage.html#8926)
  - Shiina\
    [其中比较关键的点是相对位置概念和点电荷的静电势能计算。](http://www.wowotech.net/basic_subject/voltage.html#8925)
  - leelockhey\
    [你这是哪个内核版本](http://www.wowotech.net/pm_subsystem/generic_pm_architecture.html#8924)
  - ja\
    [@dream：我看完這段也有相同的想法，引用 @dream ...](http://www.wowotech.net/kernel_synchronization/spinlock.html#8922)
  - 元神高手\
    [围观首席power managerment专家](http://www.wowotech.net/pm_subsystem/device_driver_pm.html#8921)
  - 十七\
    [内核空间的映射在系统启动时就已经设定好，并且在所有进程的页表...](http://www.wowotech.net/process_management/context-switch-arch.html#8920)

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

  - [Linux cpufreq framework(2)\_cpufreq driver](http://www.wowotech.net/pm_subsystem/cpufreq_driver.html)
  - [Linux内核同步机制之（四）：spin lock](http://www.wowotech.net/kernel_synchronization/spinlock.html)
  - [关于java单线程经常占用cpu100%分析](http://www.wowotech.net/linux_kenrel/483.html)
  - [蓝牙协议分析(10)\_BLE安全机制之LE Encryption](http://www.wowotech.net/bluetooth/le_encryption.html)
  - [linux usb 摄像头测试](http://www.wowotech.net/162.html)

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
