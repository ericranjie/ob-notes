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

作者：[linuxer](http://www.wowotech.net/author/3 "linuxer") 发布于：2017-3-10 18:50 分类：[进程管理](http://www.wowotech.net/sort/process_management)

一、前言  
Linux调度器神秘而充满诱惑，每个Linux工程师都想深入其内部一探究竟。不过中国有一句古话叫做“相由心生”，一个模块精巧的内部逻辑（也就是所谓的“心”）其外延就是简洁而优雅的接口（我称之为“相”）。通过外部接口的定义，其实我们也可以收获百分之六七十的该模块的内部信息。因此，本文主要描述Linux调度器开放给用户空间的接口，希望可以通过用户空间的调度器接口来理解Linux调度器的行为。

二、nice函数

nice函数用来修改调用进程的nice value，其接口定义如下：

>       #include <unistd.h>  
>        int nice(int inc);

为了方便说明该接口的作用，我们还是举实际的例子说明。程序调用nice(3)，则将当前进程的nice value增加3，这也就是意味着该进程的优先级降低3个level（提升nice value也就是对别人更加nice，自己的优先级就会低）。如果程序调用nice(-5)，则将当前进程的nice value减去5，这也就是意味着该进程的优先级提升5个level。当调用错误的时候返回-1，调用成功会稍微有一些歧义。POSIX标准规定了nice函数返回新的nice value，但是linux的系统调用和c库都是采用了操作成功返回0的方式。这样的处理方式使得在调用nice函数的时候无法得到当前的优先级，如果想要得到当前优先级，需要调用getpriority函数，我们在下一小节描述。

虽然说nice函数是用来调整优先级，实际上调整nice value就是调整调度器分配给该进程的CPU时间，具体是如何影响cpu time的呢？我们在后面描述内核代码的时候再详聊。此外，需要注意的是：根据POSIX标准，nice value是一个per process的设定，但是在linux中，nice value没有遵从这个标准，它是per-thread的一个属性。

三、getpriority/setpriority函数

从上节的描述中，我们了解到了nice的函数的限制，例如只能修改自己的nice value，无法获取当前的nice value值等，为此我们给出加强版本的nice接口，也就是getpriority/setpriority函数了。getpriority/setpriority函数定义如下：

> #include <sys/time.h>  
> #include  <sys/resource.h>
> 
> int getpriority(int which, int who);  
> int setpriority(int which, int who, int prio);

你说接口增加功能是好事，怎么就把名字也改了呢？为何不是getnice/setnice呢？其实从上节的描述也看出稍许端倪，我们并没有区分调度优先级和nice value这两个值，历史上，首先被使用的是nice value，很快大家觉得这个词不是那么好理解，特别是对于初学者，因此改成优先级（priority）这样的名词可以让用户更好的理解这个API的作用，当然，事实证明这个改动并不是非常理想，我们后面会描述。

getpriority/setpriority功能比较强大，能处理多种请求，不同的请求通过which和who这两个参数来制定。当which等于PRIO_PROCESS的时候，who需要传入一个process id的参数，getpriority将返回指定进程的nice value。当which等于PRIO_PGRP的时候，who需要传入一个process group id的参数，此时getpriority将返回指定进程组中优先级最高的那个（BTW，nice value是最小的）。当which等于PRIO_USER的时候，who需要user id的信息，这时候，getpriority将返回属于该user的所有进程中nice value最小的那个。who等于0说明要get或者set的对象是当前进程（或者当前进程组，或者当前的user）。

setpriority类似与nice，当然功能要强那么一点点，因为它可以接收PRIO_PROCESS，PRIO_PGRP或者PRIO_USER参数用来设定一组进程的nice value。setpriority的返回值和其他函数类似，0表示成功，-1表示操作失败，不过getpriority就稍微有一点绕了。作为linux程序员，我们都知道的nice value是[-20, 19]，如果getpriority返回这个范围，那么这里的-1优先级就有点尴尬了，因为一般的linux c库接口函数返回-1表示调用错误，我们是如何区分-1调用错误的返回还是优先级-1的返回值呢？getpriority是少数返回-1也是有可能正确的接口函数：在调用getpriority之前，我们需要首先将errno清零，调用getpriority之后，如果返回-1，我们需要看看errno是否还是保持0值，如果是，那么说明返回的是优先级-1，否则说明发生了错误。

四、操作rt priority的接口

传统的类unix内核，调度器是采用round-robin time-sharing的算法：如果有若干个进程是runnable的，那么不着急，大家排排队、吃果果，每个进程分配一个cpu时间片，大家轮流按照分配的时间片来获取cpu资源，所有的时间片用完，那么就重新一轮的分配。在这样的模型下面，间接影响cpu时间片的nice接口函数就够用了。当然，分配了更多的时间片也就是意味着有更高的优先级，因此nice vlaue也被称为进程的优先级。

但是，新的需求层出不穷（人类的欲望是无穷D），特别是实时性方面的需求，因此，POSIX标准（2008版本）增加了实时调度的内容，并且提供了POSIX realtime scheduling API来让用户空间来修改调度策略和调度优先级。这下子有点尴尬了，原来的nice value大家已经习惯称之为进程优先级了，现在真正的进程优先级登场了，怎么区分？为了解决这个问题，我们引入一个新的名词叫做调度策略（scheduling policy）。调度器在运作的时候往往设定一组规则来决定何时，选择哪一个进程进入执行状态，执行多长的时间。那些“规则”就是调度策略。

好的调度策略依赖于对进程的分类，有一类进程是大家都灰常的熟悉了就是普通进程，使用时间片轮转算法的那些进程。当然这类进程还可以细分，例如运算密集型进程（SCHED_BATCH，调度器最好不要太经常的唤醒这种进程），例如idle类进程（SCHED_IDLE），idle类进程优先级非常低，也就是说如果系统有其他事情要处理就去干别的事情（调度其他进程执行），实在没有活干了，再考虑IDLE类型的进程。不论哪一种普通进程，其优先级使用nice value这样一个调度参数来描述就OK了。

除了普通进程，还有一类是严格按照优先级来调度的进程，如果熟悉RTOS的话，对priority-base的调度器应该不会陌生，官大一级压死人，只要优先级高的进程是runnable的，那么优先级低的进程是根本没有机会执行的。这里的优先级才是真正意义的优先级，但是nice value已经被称为进程优先级了，因此这里的优先级被叫做rt priority。rt进程的调度又被细分成两类：SCHED_FIFO和SCHED_RR。这两种调度策略在相同rt priority的时候稍有差别，SCHED_FIFO是谁先到谁先获取cpu资源，并且一直占用，直到主动让出cpu或者退出，相同rt priority的进程才有机会执行。SCHED_RR稍微人性化了一点，相同rt priority的进程有时间片，大家轮流执行。对于实时进程而言，rt priority这个调度参数就描述了全部。

介绍到这里，是时候总结一下了：进程优先级有两个范围，一个是nice value，用前两个小节的API来set或者get。另外一个优先级是rt priority，完全碾压nice value这种优先级，操作rt priority的接口就在这一小节描述。

OK，经过漫长的铺垫过程，我们终于可以介绍realtime process scheduling API了，具体API定义如下：

> #include <sched.h>
> 
> int sched_setscheduler(pid_t pid, int policy, const struct sched_param *param);
> 
> int sched_getscheduler(pid_t pid);
> 
> int sched_get_priority_max(int policy);－－返回指定policy的最大的rt priority  
> int sched_get_priority_min(int policy);－－返回指定policy的最小的rt priority
> 
> int sched_setparam(pid_t pid, const struct sched_param *param);  
> int sched_getparam(pid_t pid, struct sched_param *param);

sched_get_priority_max和sched_get_priority_min分别返回了指定调度策略的最大和最小的rt priority，不同的操作系统实现不同的优先级数量。在linux中，实时进程（SCHED_FIFO和SCHED_RR）的rt priority共计99个level，最小是1，最大是99。对于其他的调度策略，这些函数返回0。

sched_getscheduler函数可以获取指定进程的scheduling policy（如果pid等于0，那么是获取调用进程的调度策略）。sched_setscheduler函数是用来设定指定进程的scheduling policy，对于实时进程，该接口函数还可以设定rt priority。如果设定进程的调度策略是非实时的调度策略的时候（例如SCHED_NORMAL），那么param参数是没有意义的，其sched_priority成员必须设定为0。sched_setparam/sched_getparam非常简单，大家自己看man page好了。

五、一统江湖的接口

看起来前面小节描述的API已经够用了，然而，故事并未结束。经过前面关于调度接口的讨论，基本上我们对调度器的行为也已经有了了解：调度器就是按照优先级（指rt priority）来工作，优先级高的永远是优先调度。范围落在[1,99]的rt priority是实时进程，而rt priority等于0的是普通进程。对于普通进程，调度器还要根据nice value（这个也曾经被称为优先级，不要和rt priority弄混了）来进行调整。用户空间的进程可以通过各种前面描述的接口API来修改调度策略、nice value以及rt priority。一切看上去已经完美，CFS类型的调度器处理普通的运算密集形（例如编译内核）和用户交互形的应用（例如vi编辑文件）。如果有应用有实时需求，可以考虑让rt类型的调度器来运筹帷幄。但是，如何混合了一些realtime的应用以及有一些timing要求的应用的时候，SCHED_FIFO和SCHED_RR并不能解决问题，因为在这种调度策略下，高优先级的任务会永远的delay低优先级的任务，如果低优先级的任务有一些timing的需求，这时候，你根本控制不了调度延迟时间。

为了解决上一节中描述的问题，一类新的进程被定义出来，这类进程的优先级比实时进程和普通进程的优先级都要高，这类进行有自己的特点，参考下图：

[![deadline](http://www.wowotech.net/content/uploadfile/201703/fb0b9a063cdff9803954f28276bff6bb20170310105036.gif "deadline")](http://www.wowotech.net/content/uploadfile/201703/5d416ba5e672e51561cb36cbf80ed9b320170310105036.gif)

这类进程的特点就是每隔固定的周期都会起来干活，需要一定的时间来处理事务。这类进程很牛，一上来就告诉调度器，我可是有点脾气的进程，和其他的那些妖艳的进程不一样的，我每隔一段时间（period）你就得固定分配给我一定的cpu资源（computer time），当然，分配的cpu time必须在该周期内执行完毕，因此就有deadline的概念。为了应对这种需求，3.14内核引入了一类新的进程叫做deadline进程，这类进程的调度策略是SCHED_DEADLINE。调度器对这类进程也会高看一眼，每当一个周期的开始时间到来的时候（也就是该deadline进程被唤醒的时间），调度器要优先处理这个deadline进程对cpu timer的需求，并且在某个指定的deadline时间内调度该进程执行。执行了指定的cpu time后，可以考虑调度走该进行，不过，当下一个周期到来的时候，调度器仍然要奋不顾身的在deadline时间内，再次调度该deadline进程执行。

虽然deadline进程优先级高于其他两类进程，但是用“优先级”来描述这类进程当然是不合理的，应该使用下面的三个参数来描述：

（1）周期时间（上图中的period）

（2）deadline时间（上图中的relative deadline）

（3）一次调度周期内分配多少的cpu时间（上图中的comp. time）

至此，估计您也已经发现，前面描述的接口其实都是不适合设定这些参数的，因此，GNU/linux操作系统中增加了下面的接口API：

> #include <sched.h>
> 
> int sched_setattr(pid_t pid, const struct sched_attr *attr, unsigned int flags);  
> int sched_getattr(pid_t pid, const struct sched_attr *attr, unsigned int size, unsigned int flags);

attr这个参数的数据类型是struct sched_attr，这个数据结构囊括了一切你想要的关于调度的控制参数：policy，nice value，rt priority，period，deadline等等。用这个接口可以完成所有前面几个小节描述API能完成的任务，唯一的不好的地方就是这个接口是linux特有的，不是posix标准，是否应用这个接口就是见仁见智了。更细节的知识这里就不描述了，大家还是参考man page好了。

六、其他

上面描述的接口API都是和调度器参数相关，其实Linux调度器还有两类接口。一个是sched_getaffinity和sched_setaffinity，用于操作一个线程的CPU affinity。另外一个接口是sched_yield，该接口可以让出CPU资源，让Linux调度器选择一个合适的线程执行。这些接口很简单，大家仔细学习就OK了。

参考文档：

1、POSIX标准2008

2、linux下的各种man page

3、linux 4.4.6内核源代码

_原创文章，转发请注明出处。蜗窝科技_

标签: [调度器](http://www.wowotech.net/tag/%E8%B0%83%E5%BA%A6%E5%99%A8) [进程管理](http://www.wowotech.net/tag/%E8%BF%9B%E7%A8%8B%E7%AE%A1%E7%90%86)

[![](http://www.wowotech.net/content/uploadfile/201605/ef3e1463542768.png)](http://www.wowotech.net/support_us.html)

« [玩转BLE(3)_使用微信蓝牙精简协议伪造记步数据](http://www.wowotech.net/bluetooth/weixin_ble_1.html) | [Linux MMC framework(2)_host controller driver](http://www.wowotech.net/comm/mmc_host_driver.html)»

**评论：**

**icy_river**  
2023-12-25 11:14

// glibc实现  
int  
__getpriority (enum __priority_which which, id_t who)  
{  
  int res;  
  
  res = INLINE_SYSCALL (getpriority, 2, (int) which, who);  
  if (res >= 0)  
    res = PZERO - res;  // PZERO==20  
  return res;  
}

[回复](http://www.wowotech.net/process_management/scheduler-API.html#comment-8854)

**MarcusKin**  
2019-09-22 14:25

getpriority看内核代码应该是[40...1]，为了解决返回-1的问题，请确认下我的理解是否正确。

[回复](http://www.wowotech.net/process_management/scheduler-API.html#comment-7660)

**MarcusKin**  
2019-09-22 14:28

@MarcusKin：内核系统调用返回是[40...1]，C库又转成[-20...19]了？

[回复](http://www.wowotech.net/process_management/scheduler-API.html#comment-7661)

**发表评论：**

 昵称

 邮件地址 (选填)

 个人主页 (选填)

![](http://www.wowotech.net/include/lib/checkcode.php) 

- ### 站内搜索
    
       
     蜗窝站内  互联网
    
- ### 功能
    
    [留言板  
    ](http://www.wowotech.net/message_board.html)[评论列表  
    ](http://www.wowotech.net/?plugin=commentlist)[支持者列表  
    ](http://www.wowotech.net/support_list)
- ### 最新评论
    
    - ja  
        [@dream：我看完這段也有相同的想法，引用 @dream ...](http://www.wowotech.net/kernel_synchronization/spinlock.html#8922)
    - 元神高手  
        [围观首席power managerment专家](http://www.wowotech.net/pm_subsystem/device_driver_pm.html#8921)
    - 十七  
        [内核空间的映射在系统启动时就已经设定好，并且在所有进程的页表...](http://www.wowotech.net/process_management/context-switch-arch.html#8920)
    - lw  
        [sparse模型和disconti模型没看出来有什么本质区别...](http://www.wowotech.net/memory_management/memory_model.html#8919)
    - 肥饶  
        [一个没设置好就出错](http://www.wowotech.net/linux_kenrel/516.html#8918)
    - orange  
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
    
    - [CFS任务的负载均衡（load balance）](http://www.wowotech.net/process_management/load_balance_detail.html)
    - [Linux的时钟](http://www.wowotech.net/timer_subsystem/clock-id-in-linux.html)
    - [中断唤醒系统流程](http://www.wowotech.net/irq_subsystem/irq_handle_procedure.html)
    - [计算机科学基础知识（四）: 动态库和位置无关代码](http://www.wowotech.net/basic_subject/pic.html)
    - [Linux设备模型(9)_device resource management](http://www.wowotech.net/device_model/device_resource_management.html)
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