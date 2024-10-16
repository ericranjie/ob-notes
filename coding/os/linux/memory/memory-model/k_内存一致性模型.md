
作者：[passerby](http://www.wowotech.net/author/257) 发布于：2019-3-24 14:22 分类：[内存管理](http://www.wowotech.net/sort/memory_management)

早期的CPU是通过提高主频来提升CPU的性能，但是随着频率“红利”越来越困难的情况下，厂商开始用多核来提高CPU的计算能力。多核是指一个CPU里有多个核心，在同一时间一个CPU能够同时运行多个线程，通过这样提高CPU的并发能力。

内存一致性模型（memory consistency model）就是用来描述多线程对共享存储器的访问行为，在不同的内存一致性模型里，多线程对共享存储器的访问行为有非常大的差别。这些差别会严重影响程序的执行逻辑，甚至会造成软件逻辑问题。在后面的介绍中，我们将分析不同的一致性模型里，多线程的内存访问乱序问题。

目前有多种内存一致性模型：
顺序存储模型（sequential consistency model）
完全存储定序（total store order）
部分存储定序（part store order）
宽松存储模型（relax memory order）
在后面我们会分析这几个一致性模型的特性

在分析之前，我们先定义一个基本的内存模型，以这个内存模型为基础进行分析
![[Pasted image 20241016095759.png]]

上图是现代CPU的基本内存模型，CPU内部有多级缓存来提高CPU的load/store访问速度（因为对于CPU而言，主存的访问速度太慢了，上百个时钟周期的内存访问延迟会极大的降低CPU的使用效率，所以CPU内部往往使用多级缓存来提升内存访问效率。）

C1与C2是CPU的2个核心，这两个核心有私有缓存L1，以及共享缓存L2。最后一级存储器才是主存。后面的顺序一致性模型（SC）中，我们会以这个为基础进行描述（在完全存储定序、部分存储定序和宽松内存模型里会有所区别，后面会描述相关的部分）

为了简化描述的复杂性，在下面的内存一致性模型描述里，会先将缓存一致性（cache coherence）简单化，认为缓存一致性是完美的（假设多核cache间的数据同步与单核cache一样，没有cache引起的数据一致性问题），以减少描述的复杂性。

顺序存储模型是最简单的存储模型，也称为强定序模型。CPU会按照代码来执行所有的load与store动作，即按照它们在程序的顺序流中出现的次序来执行。从主存储器和CPU的角度来看，load和store是顺序地对主存储器进行访问。

下面分析这段代码的执行结果

![[Pasted image 20241016095814.png]]

在顺序存储器模型里，MP（多核）会严格严格按照代码指令流来执行代码

所以上面代码在主存里的访问顺序是：

S1 S2 L1 L2

通过上面的访问顺序我们可以看出来，虽然C1与C2的指令虽然在不同的CORE上运行，但是C1发出来的访问指令是顺序的，同时C2的指令也是顺序的。虽然这两个线程跑在不同的CPU上，但是在顺序存储模型上，其访问行为与UP（单核）上是一致的。

我们最终看到r2的数据会是NEW，与期望的执行情况是一致的，所以在顺序存储模型上是不会出现内存访问乱序的情况

## 3 完全存储定序

为了提高CPU的性能，芯片设计人员在CPU中包含了一个存储缓存区（store buffer），它的作用是为store指令提供缓冲，使得CPU不用等待存储器的响应。所以对于写而言，只要store buffer里还有空间，写就只需要1个时钟周期（哪怕是ARM-A76的L1 cache，访问一次也需要3个cycles，所以store buffer的存在可以很好的减少写开销），但这也引入了一个访问乱序的问题。

首先我们需要对上面的基础内存模型做一些修改，表示这种新的内存模型

相比于以前的内存模型而言，store的时候数据会先被放到store buffer里面，然后再被写到L1 cache里。

![[Pasted image 20241016095829.png]]

首先我们思考单核上的两条指令：

S1：store flag= set

S2：load r1=data

S3：store b=set

如果在顺序存储模型中，S1肯定会比S2先执行。但是如果在加入了store buffer之后，S1将指令放到了store buffer后会立刻返回，这个时候会立刻执行S2。S2是read指令，CPU必须等到数据读取到r1后才会继续执行。这样很可能S1的store flag=set指令还在store buffer上，而S2的load指令可能已经执行完（特别是data在cache上存在，而flag没在cache中的时候。这个时候CPU往往会先执行S2，这样可以减少等待时间）

这里就可以看出再加入了store buffer之后，内存一致性模型就发生了改变。

如果我们定义store buffer必须严格按照FIFO的次序将数据发送到主存（所谓的FIFO表示先进入store buffer的指令数据必须先于后面的指令数据写到存储器中），这样S3必须要在S1之后执行，CPU能够保证store指令的存储顺序，这种内存模型就叫做完全存储定序（TSO）。

我们继续看下面的一段代码

![[Pasted image 20241016095848.png]]

在SC模型里，C1与C2是严格按照顺序执行的

代码可能的执行顺序如下：

S1 S2 L1 L2

S1 L1 S2 L2

S1 L1 L2 S2

L1 L2 S1 S2

L1 S1 S2 L2

L1 S1 L2 S2

由于SC会严格按照顺序进行，最终我们看到的结果是至少有一个CORE的r1值为NEW，或者都为NEW。

在TSO模型里，由于store buffer的存在，L1和S1的store指令会被先放到store buffer里面，然后CPU会继续执行后面的load指令。Store buffer中的数据可能还没有来得及往存储器中写，这个时候我们可能看到C1和C2的r1都为0的情况。

所以，我们可以看到，在store buffer被引入之后，内存一致性模型已经发生了变化（从SC模型变为了TSO模型），会出现store-load乱序的情况，这就造成了代码执行逻辑与我们预先设想不相同的情况。而且随着内存一致性模型越宽松（通过允许更多形式的乱序读写访问），这种情况会越剧烈，会给多线程编程带来很大的挑战。

## 4 部分存储定序

芯片设计人员并不满足TSO带来的性能提升，于是他们在TSO模型的基础上继续放宽内存访问限制，允许CPU以非FIFO来处理store buffer缓冲区中的指令。CPU只保证地址相关指令在store buffer中才会以FIFO的形式进行处理，而其他的则可以乱序处理，所以这被称为部分存储定序（PSO）。

那我们继续分析下面的代码

![[Pasted image 20241016095904.png]]

S1与S2是地址无关的store指令，cpu执行的时候都会将其推到store buffer中。如果这个时候flag在C1的cahe中存在，那么CPU会优先将S2的store执行完，然后等data缓存到C1的cache之后，再执行store data=NEW指令。

这个时候可能的执行顺序：

S2 L1 L2 S1

这样在C1将data设置为NEW之前，C2已经执行完，r2最终的结果会为0，而不是我们期望的NEW，这样PSO带来的store-store乱序将会对我们的代码逻辑造成致命影响。

从这里可以看到，store-store乱序的时候就会将我们的多线程代码完全击溃。所以在PSO内存模型的架构上编程的时候，要特别注意这些问题。

## 5 宽松内存模型

丧心病狂的芯片研发人员为了榨取更多的性能，在PSO的模型的基础上，更进一步的放宽了内存一致性模型，不仅允许store-load，store-store乱序。还进一步允许load-load，load-store乱序， 只要是地址无关的指令，在读写访问的时候都可以打乱所有load/store的顺序，这就是宽松内存模型（RMO）。

我们再看看上面分析过的代码

![[Pasted image 20241016095928.png]]

在PSO模型里，由于S2可能会比S1先执行，从而会导致C2的r2寄存器获取到的data值为0。在RMO模型里，不仅会出现PSO的store-store乱序，C2本身执行指令的时候，由于L1与L2是地址无关的，所以L2可能先比L1执行，这样即使C1没有出现store-store乱序，C2本身的load-load乱序也会导致我们看到的r2为0。从上面的分析可以看出，RMO内存模型里乱序出现的可能性会非常大，这是一种乱序随可见的内存一致性模型。

## 6 内存屏障

芯片设计人员为了尽可能的榨取CPU的性能，引入了乱序的内存一致性模型，这些内存模型在多线程的情况下很可能引起软件逻辑问题。为了解决在有些一致性模型上可能出现的内存访问乱序问题，芯片设计人员提供给了内存屏障指令，用来解决这些问题。

内存屏障的最根本的作用就是提供一个机制，要求CPU在这个时候必须以顺序存储一致性模型的方式来处理load与store指令，这样才不会出现内存访问不一致的情况。

对于TSO和PSO模型，内存屏障只需要在store-load/store-store时需要（写内存屏障），最简单的一种方式就是内存屏障指令必须保证store buffer数据全部被清空的时候才继续往后面执行，这样就能保证其与SC模型的执行顺序一致。

而对于RMO，在PSO的基础上又引入了load-load与load-store乱序。RMO的读内存屏障就要保证前面的load指令必须先于后面的load/store指令先执行，不允许将其访问提前执行。

我们继续看下面的例子：

![[Pasted image 20241016100028.png]]

例如C1执行S1与S2的时候，我们在S1与S2之间加上写屏障指令，要求C1按照顺序存储模型来进行store的执行，而在C2端的L1与L2之间加入读内存屏障，要求C2也按照顺序存储模型来进行load操作，这样就能够实现内存数据的一致性，从而解决乱序的问题。

ARM的很多微架构就是使用RMO模型，所以我们可以看到ARM提供的dmb内存指令有多个选项：

LD              load-load/load-store

ST              store-store/store-load

SY              any-any

这些选项就是用来应对不同情况下的乱序，让其回归到顺序一致性模型的执行顺序上去

标签: [内存一致性模型](http://www.wowotech.net/tag/%E5%86%85%E5%AD%98%E4%B8%80%E8%87%B4%E6%80%A7%E6%A8%A1%E5%9E%8B)

---

« [浅谈Cache Memory](http://www.wowotech.net/memory_management/458.html) | [copy\_{to,from}\_user()的思考](http://www.wowotech.net/memory_management/454.html)»

**评论：**

**火耳**\
2020-05-31 00:41

赞，解答了很多疑惑

[回复](http://www.wowotech.net/memory_management/456.html#comment-8005)

**BurNIng**\
2020-02-12 10:49

大神，跪拜一下

[回复](http://www.wowotech.net/memory_management/456.html#comment-7874)

**lamb0145**\
2019-11-26 16:30

终于明白了arm64架构头文件里的dsb(sy)是个什么东西了。

[回复](http://www.wowotech.net/memory_management/456.html#comment-7761)

**lamb0145**\
2019-11-26 17:32

@lamb0145：对了，这个也只能保证单个core上的访问顺序吧？如果多核上要保证，还是需要spin_lock这些锁的吧？

[回复](http://www.wowotech.net/memory_management/456.html#comment-7762)

**googol**\
2021-04-14 15:59

@lamb0145：同问，内存屏障的例子里怎么保证core 1和core 2之间也是按代码顺序执行的？

[回复](http://www.wowotech.net/memory_management/456.html#comment-8219)

**BeDook**\
2019-08-02 10:05

博主，有没有时间举例讲解一下SCU和ACP啊，SCU是维护各个核的L1 D-Cache之间的一致性的么？ACP是维护外设，如DMA和Cache/Memory之间的一致性的么？能否举例

[回复](http://www.wowotech.net/memory_management/456.html#comment-7568)

**gnx**\
2019-05-18 17:41

写得很清晰，赞！吹毛求疵一下，tso例子中s1，s2，l1，l2命名有点点混淆

[回复](http://www.wowotech.net/memory_management/456.html#comment-7427)

**Leo**\
2019-04-16 11:35

Blog是站主才能发么？感觉有些总结挺好的，有空我也想把自己学的梳理一下，发到这里来，大家一起来讨论下

[回复](http://www.wowotech.net/memory_management/456.html#comment-7353)

**[wowo](http://www.wowotech.net/)**\
2019-04-16 20:01

@Leo：不止站主才能发，大家都可以发的，非常欢迎！要发表的话我可以给你开个账号~

[回复](http://www.wowotech.net/memory_management/456.html#comment-7354)

**walker**\
2019-04-07 16:46

作者，页面上的图有点小，看不清楚哈

[回复](http://www.wowotech.net/memory_management/456.html#comment-7346)

**蓝领笑笑生**\
2019-04-02 22:53

虽然不知道说的是什么，但看起来好厉害的样子！

[回复](http://www.wowotech.net/memory_management/456.html#comment-7340)

**nswcfd_001**\
2019-04-01 20:29

循序渐进的解释清楚了各个概念，不过好像图片的缩放有些问题。

[回复](http://www.wowotech.net/memory_management/456.html#comment-7338)

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

  - [Linux cpuidle framework(3)\_ARM64 generic CPU idle driver](http://www.wowotech.net/pm_subsystem/cpuidle_arm64.html)
  - [X-026-KERNEL-Linux gpio driver的移植之gpio range](http://www.wowotech.net/x_project/kernel_gpio_driver_porting_2.html)
  - [Linux reset framework](http://www.wowotech.net/pm_subsystem/reset_framework.html)
  - [显示技术介绍(3)\_CRT技术](http://www.wowotech.net/display/crt_intro.html)
  - [X-004-UBOOT-串口驱动移植(Bubblegum-96平台)](http://www.wowotech.net/x_project/bubblegum_uboot_serial.html)

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
