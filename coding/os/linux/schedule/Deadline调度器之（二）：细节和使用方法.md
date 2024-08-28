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

作者：[linuxer](http://www.wowotech.net/author/3 "linuxer") 发布于：2018-2-22 18:23 分类：[进程管理](http://www.wowotech.net/sort/process_management)

一、前言

Linux内核的DL调度器是一个全局EDF调度器，它主要针对有deadline限制的sporadic任务。注意：这些术语已经在本系列文章的[第一部分](http://www.wowotech.net/process_management/deadline-scheduler-1.html)中说明了，这里不再赘述。在这本文中，我们将一起来看看Linux DL调度器的细节以及如何使用它。另外，本文对应的英文原文是https://lwn.net/Articles/743946/，感谢lwn和Daniel Bristot de Oliveira的分享。

二、细节

DL调度器是根据任务的deadline来确定调度的优先顺序的：deadline最早到来的那个任务最先调度执行。对于有M个处理器的系统，优先级最高的前M个deadline任务（即deadline最早到来的前M个任务）将被选择在对应M个处理器上运行。

Linux DL调度器还实现了constant bandwidth server（CBS）算法，该算法是一种CPU资源预留协议。CBS可以保证每个任务在每个period内都能收到完整的runtime时间。在一个周期内，DL进程的“活”来的时候，CBS会重新补充该任务的运行时间。在处理“活”的时候，runtime时间会不断的消耗；如果runtime使用完毕，该任务会被DL调度器调度出局。在这种情况下，该任务无法再次占有CPU资源，只能等到下一次周期到来的时候，runtime重新补充之后才能运行。因此，CBS一方面可以用来保证每个任务的CPU时间按照其定义的runtime参数来分配，另外一方面，CBS也保证任务不会占有超过其runtime的CPU资源，从而防止了DL任务之间的互相影响。

为了避免DL任务造成系统超负荷运行，DL调度器有一个准入机制，在任务配置好了period、runtime和deadline参数之后并准备加入到系统的时候，DL调度器会对该任务进行评估。这个准入机制保证了DL任务将不会使用超过系统的CPU时间的最大值。这个最大值在kernel.sched_rt_runtime_us和kernel.sched_rt_period_us sysctl参数中指定。默认值是950000和1000000，表示在1s的周期内，CPU用于执行实时任务（DL任务和RT任务）的最大时间值是950000µs。对于单个核心系统，这个测试既是必要的，也是充分的。这意味着：既然接受了该DL任务，那么CPU有信心可以保证其在截止日期之前能够分配给它需要的runtime长度的CPU时间。

然而，值得注意的是，准入测试对于多处理器系统的全局调度算法是必要的，但不是充分的。Dhall效应（在[Deadline调度器之原理](http://www.wowotech.net/process_management/deadline-scheduler-1.html)部分描述）说明了全局deadline调度器即便是接受了该任务，但是在每个CPU利用率未达100％的情况下（有可分配的CPU资源），也不能保证能该DL任务的deadline的需求得到满足。因此，在多处理器系统中，准入测试并不保证一旦接受，任务将能够在截止日期之前分配并使用其指定的运行时间。对于被接受的DL任务而言，调度器最多能做到的是“有界延迟“，对于软实时系统而言，这已经是一个不错的保证了。如果用户希望保证所有任务都能满足他们的最后期限，用户就必须使用分区方法（即使用partitioned scheduler），或者使用下面的准入测试（是必要且充分的）：

> Σ(WCETi / Pi) <= M - (M - 1) x Umax

把上面的公式用一句话表示就是：每个任务的（运行时间/周期）的总和应该小于或等于处理器的数目M，减去最大的利用率Umax乘以（M-1）。Umax是所有DL任务中，（运行时间/周期）值最大的那个（即对CPU资源需求最大）。事实证明，在低负荷情况下（即Umax比较小），系统容易进行调度处理。

对于那些cpu利用率很高的任务而言，一个很好的策略是将系统进行区域划分。即将一些高负载任务隔离开来，从而使“小活”（cpu使用率不高）和“大活”各自在一组不同的CPU上进行调度。目前，DL调度器不允许用户设置一个线程的亲和性，不过可以使用control group cpusets来对系统进行分区。

三、使用方法

例如，考虑一个有八个CPU的系统。一个“大活”的CPU利用率接近90%（单核场景下），而组内其他任务的利用率都较低。在这种场景下，一个推荐的设置是这样的：CPU0运行CPU利用率高的那个“大活”任务，让其他任务运行在其余的CPU上。要想实现这样的系统配置，用户可以执行以下步骤：

> # cd /sys/fs/cgroup/cpuset/
> 
> # mkdir cluster
> 
> # mkdir partition

首先进入cpuset目录，创建两个cpuset，然后执行下面的命令：

> # echo 0 > cpuset.sched_load_balance

上面的操作在root cpuset中disable了负载均衡，从而让新创建的cluster和partition这两个cpuset变成root domain。下面我们将对cluster进行配置，具体操作如下：

> # cd cluster/
> 
> # echo 1-7 > cpuset.cpus
> 
> # echo 0 > cpuset.mems
> 
> # echo 1 > cpuset.cpu_exclusive

上面的操作设定了cluster中的任务可以使用1～7这些系统中的CPU，cpuset.mems那一行操作和memory node相关（即设定该cpuset可以使用的memory node），如果系统不是NUMA的话，echo 0就OK了。cpuset.cpu_exclusive 是配置cpuset.cpus中的cpu们是否是该cpuset独占的cpu。在这个场景中，CPU 1~7只是分配给cluster这个cpu set，因此是独占的。OK，现在需要把各个task加入到该cluster这个cpu set中了，具体操作如下：

> # ps -eLo lwp | while read thread; do echo $thread > tasks ; done

上面的命令把系统中所有的LWP加入到cluster cpuset中。下面我们开始配置partition cpuset：

> # cd ../partition/
> 
> # echo 1 > cpuset.cpu_exclusive
> 
> # echo 0 > cpuset.mems
> 
> # echo 0 > cpuset.cpus

这里的配置过程和配置cluster的过程是一样的，这里就不再具体解释了。现在我们需要把shell移到partition这个cpuset中，操作命令如下：

> # echo $$ > tasks

完成上面的准备工作之后，最后一步就是在shell中启动deadline任务。

四、程序员视角

我们在这一章讨论使用DL调度器的场景。我们提供了三个例子：

（1）固定占有CPU资源的服务器程序

（2）按照固定的周期重新分配CPU资源的任务

（3）等待外部事件的服务器程序（外部事件可以周期性的，也可以使sporadic形态的）

周期是DL调度中最基本的参数，它定义了一个任务是以什么样子的频繁程度被激活。当一个任务没有固定的激活模式时，也可以使用DL调度器，但是这时候往往是仅仅使用其CBS特性。

我们首先举一个仅仅使用DL调度器CBS特性的例子。假设一个task，没有固定pattern，但是我们不想让它占用太多的CPU资源，仅仅是想让它最多占有20％的CPU资源。这时候，我们可以设定周期为1S，runtime是200ms，sched_setattr() 接口函数可以用来设定DL调度参数，具体的实现可以参考下面的代码：

> int main (int argc, char **argv)
> 
> {
> 
>     int ret;
> 
>     int flags = 0;
> 
>     struct sched_attr attr;
> 
>     memset(&attr, 0, sizeof(attr));
> 
>     attr.size = sizeof(attr);
> 
>     /* This creates a 200ms / 1s reservation */
> 
>     attr.sched_policy = SCHED_DEADLINE;
> 
>     attr.sched_runtime = 200000000;
> 
>     attr.sched_deadline = attr.sched_period = 1000000000;
> 
>     ret = sched_setattr(0, &attr, flags);
> 
>     if (ret < 0) {
> 
>         perror("sched_setattr failed to set the priorities");
> 
>         exit(-1);
> 
>     }
> 
>     do_the_computation_without_blocking();
> 
>     exit(0);
> 
> }

在非周期性（aperiodic ）的情况下，任务不需要知道周期何时开始，它只管运行就好了，反正在该任务消耗完指定的运行时间之后，DL调度器会对其进行节流（throttle ）。这种场景下，应用程序没有deadline的需求（deadline等于period），仅仅使用CBS特性。

我们再来一个DL调度器应用场景的例子：这次是一个有固定激活模式的任务，即该任务会在固定的时间间隔上醒来，进行事务处理，而该任务处理完之后就睡眠，直到下一个周期到来。这时候在新的周期中，runtime会重新恢复，该任务会再次被DL调度器调度，然后周而复始。具体的代码和上一段代码类似，只是具体计算部分的代码如下：

> for(;;) {
> 
>     do_the_computation();
> 
>     sched_yield();
> 
> }

具体的调度参数和上一个代码示例是一样的（即事件到来的周期是1S），虽然给出了200ms的runtime设定，但是实际上的处理不会超过200ms，一旦处理完事件，程序会调用sched_yield告知DL调度器：我已经处理完事件了，到下一个周期再给我分配资源吧，我没有什么事情需要处理了。顺便说一句，处理时间超过200ms是没有意义的，这时候CBS会throttle该任务。还有一个比较有意思的知识点就是DL调度器对yield的处理和CFS调度器不一样，DL task yield之后会阻塞该进程，直到下一个调度周期到来。

上面的例子有点类似定时任务，即每个固定的时间间隔就起来处理一些日常性事务，不过真实的实时进程往往是外部事件驱动的具体代码如下（DL参数是一样的）：

> for(;;) {
> 
>     process_the_data();
> 
>     produce_the_result()
> 
>     block_waiting_for_the_next_event();
> 
> }

在这个场景下，该任务是阻塞在系统调用中。当外部事件发生的时候，该任务被唤醒。外部事件并不是以严格的周期来唤醒该任务，但是会有一个最小的周期，也就是说这是一个sporadic task。一旦任务被激活，它将执行计算并提供响应，当该任务完成计算，提供了输出，它将由于等待下一个事件而进入休眠状态。

五、结论

deadline调度器是仅仅根据实时任务的时序约束进行调度的，从而保证实时任务正确的逻辑行为。虽然在多核系统中，全局deadline调度器会面临Dhall效应，不过我们仍然可以对系统进行分区来解决这个问题。具体的做法是采用cpusets的方法把CPU利用率高的任务放置到指定的cpuset上。开发人员也可以受益于deadline调度器：他们可以通过设计其应用程序与DL调度器交互，从而简化任务的时序控制行为。

在linux中，DL任务比实时任务（RR和FIFO）具有更高的优先级。这意味着即使是最高优先级的实时任务也会被DL任务延迟执行。因此，DL任务不需要考虑来自实时任务的干扰，但实时任务必须考虑DL任务的干扰。

DL调度器和PREEMPT_RT补丁在改善Linux实时性方面发挥着不同的作用。DL调度器让任务的调度以一种更可预测的方式进行，而PREEMPT_RT补丁集的目标是减少和限制较低优先级的任务对实时任务的调度延迟。具体的做法是通过减少下列内核中的不可抢占时间来完成的：（1）关闭抢占（2）disable IRQ（3）低优先级任务持锁。

例如，当一个实时任务运行在非实时内核上的时候，从该任务被唤醒到真正调度执行可能会有高达5ms的调度延迟。在这样的系统中，内核是无法处理deadline小于5ms的任务。相反，在实时内核的情况下，调度延迟可能不会超过150µs。这时候，那些更短的deadline的任务（例如小于5ms）也能被轻松处理。

_原创翻译整理文章，转发请注明出处。蜗窝科技_

标签: [scheduler](http://www.wowotech.net/tag/scheduler) [Deadline](http://www.wowotech.net/tag/Deadline)

[![](http://www.wowotech.net/content/uploadfile/201605/ef3e1463542768.png)](http://www.wowotech.net/support_us.html)

« [图解slub](http://www.wowotech.net/memory_management/426.html) | [KASAN实现原理](http://www.wowotech.net/memory_management/424.html)»

**评论：**

**[JiangX](http://sketch2sky.com/)**  
2019-11-26 20:33

多核deadline本质是一个装箱问题，但本文中的：  
  
或者使用下面的准入测试（是必要且充分的）：  
Σ(WCETi / Pi) <= M - (M - 1) x Umax  
  
我理解这个条件不是充要条件，而是个充分不必要条件，即通过了准入测试，可以满足deadline(period)需求，但满足deadline(period)需求的，不一定满足这个条件。不知博主验证了这个公式没，可以指点下

[回复](http://www.wowotech.net/process_management/dl-scheduler-2.html#comment-7763)

**hmayer**  
2022-04-06 14:02

@JiangX：作者虽然描述的是DL，但是DL和RT两者有些许的关系，在有些概念的解释中，读者容易混淆。  
关于多核调度能力分析，请参考 A Survey of Hard Real-Time Scheduling for Multiprocessor Systems。可以完美解释你的问题。

[回复](http://www.wowotech.net/process_management/dl-scheduler-2.html#comment-8583)

**hahahah**  
2019-04-17 20:20

main.c: In function ‘main’:  
main.c:23:23: error: storage size of ‘attr’ isn’t known  
     struct sched_attr attr;  
                       ^  
main.c:37:11: warning: implicit declaration of function ‘sched_setattr’ [-Wimplicit-function-declaration]  
     ret = sched_setattr(0, &attr, flags);  
请问你有遇到这个报错吗

[回复](http://www.wowotech.net/process_management/dl-scheduler-2.html#comment-7358)

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
    
    - [linux cpufreq framework(3)_cpufreq core](http://www.wowotech.net/pm_subsystem/cpufreq_core.html)
    - [Linux TTY framework(3)_从应用的角度看TTY设备](http://www.wowotech.net/tty_framework/application_view.html)
    - [X-006-UBOOT-pinctrl driver移植(Bubblegum-96平台)](http://www.wowotech.net/x_project/bubblegum_uboot_pinctrl.html)
    - [Linux电源管理(1)_整体架构](http://www.wowotech.net/pm_subsystem/pm_architecture.html)
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