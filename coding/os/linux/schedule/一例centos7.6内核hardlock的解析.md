作者：[安庆](http://www.wowotech.net/author/539 "oppo混合云内核&虚拟化负责人，架构并孵化了oppo的云游戏，云手机等产品。") 发布于：2020-3-30 20:03 分类：[Linux内核分析](http://www.wowotech.net/sort/linux_kenrel)

一、前言

本文以一个在centos7.6内核发生的crash,描述一下常见hardlock导致panic的解bug流程。

二、故障现象

机器出现复位，收集到了crash文件，从日志中显示为hardlock。

KERNEL: /usr/lib/debug/lib/modules/3.10.0-957.27.2.el7.x86_64/vmlinux

DUMPFILE: vmcore [PARTIAL DUMP]

CPUS: 48

DATE: Wed Dec 11 23:25:55 2019

UPTIME: 80 days, 12:06:06-----------------------crash之前运行时间

LOAD AVERAGE: 67.57, 65.21, 53.49-----------死之前load 较高

TASKS: 3169

NODENAME:

RELEASE: 3.10.0-957.27.2.el7.x86_64

VERSION: #1 SMP Mon Jul 29 17:46:05 UTC 2019

MACHINE: x86_64 (2300 Mhz)

MEMORY: 382.5 GB

PANIC: "Kernel panic - not syncing: Hard LOCKUP"--------配置了hardlock_panic

PID: 0

COMMAND: "swapper/34"

TASK: ffff9f1a36998000 (1 of 48) [THREAD_INFO: ffff9f1a36994000]

CPU: 34

STATE: TASK_RUNNING (PANIC)

crash> bt

PID: 0 TASK: ffff9f1a36998000 CPU: 34 COMMAND: "swapper/34"

#0 [ffff9f487e0489f0] machine_kexec at ffffffffae063b34

#1 [ffff9f487e048a50] __crash_kexec at ffffffffae11e242

#2 [ffff9f487e048b20] panic at ffffffffae75d85b

#3 [ffff9f487e048ba0] nmi_panic at ffffffffae09859f

#4 [ffff9f487e048bb0] watchdog_overflow_callback at ffffffffae14a881

#5 [ffff9f487e048bc8] __perf_event_overflow at ffffffffae1a26b7

#6 [ffff9f487e048c00] perf_event_overflow at ffffffffae1abd24

#7 [ffff9f487e048c10] intel_pmu_handle_irq at ffffffffae00a850

#8 [ffff9f487e048e38] perf_event_nmi_handler at ffffffffae76d031

#9 [ffff9f487e048e58] nmi_handle at ffffffffae76e91c

#10 [ffff9f487e048eb0] do_nmi at ffffffffae76eb3d

#11 [ffff9f487e048ef0] end_repeat_nmi at ffffffffae76dd89 [exception RIP: tg_unthrottle_up+24]

RIP: ffffffffae0dc4d8 RSP: ffff9f487e043e08 RFLAGS: 00000046

RAX: ffff9f4a6a078400 RBX: ffff9f787be9ab80 RCX: ffff9f4a03ea9530

RDX: 0000000000000005 RSI: ffff9f787be9ab80 RDI: ffff9f4a03ea9400------task_group

RBP: ffff9f487e043e08 R8: ffff9f782c78a100 R9: 0000000000000001

R10: 0000000000004dcd R11: 0000000000000005 R12: ffff9f78790edc00

R13: ffffffffae0dc4c0 R14: 0000000000000000 R15: ffff9f4a03ea9400

ORIG_RAX: ffffffffffffffff CS: 0010 SS: 0000

--- <NMI exception stack> ---

#12 [ffff9f487e043e08] tg_unthrottle_up at ffffffffae0dc4d8

#13 [ffff9f487e043e10] walk_tg_tree_from at ffffffffae0d3b20

#14 [ffff9f487e043e60] unthrottle_cfs_rq at ffffffffae0e46e7

#15 [ffff9f487e043e98] distribute_cfs_runtime at ffffffffae0e496a

#16 [ffff9f487e043ee8] sched_cfs_period_timer at ffffffffae0e4b67

#17 [ffff9f487e043f20] __hrtimer_run_queues at ffffffffae0c71e3

#18 [ffff9f487e043f78] hrtimer_interrupt at ffffffffae0c776f

#19 [ffff9f487e043fc0] local_apic_timer_interrupt at ffffffffae05a61b

#20 [ffff9f487e043fd8] smp_apic_timer_interrupt at ffffffffae77b6e3

#21 [ffff9f487e043ff0] apic_timer_interrupt at ffffffffae777df2

--- <IRQ stack> ---

#22 [ffff9f1a36997db8] apic_timer_interrupt at ffffffffae777df2 [exception RIP: cpuidle_enter_state+87]

RIP: ffffffffae5b06c7 RSP: ffff9f1a36997e60 RFLAGS: 00000202

RAX: 0018b604313aebda RBX: ffff9f1a36997e38 RCX: 0000000000000018

RDX: 0000000225c17d03 RSI: ffff9f1a36997fd8 RDI: 0018b604313aebda

RBP: ffff9f1a36997e88 R8: 0000000000005a0e R9: 0000000000000018

R10: 0000000000004dcd R11: 0000000000000005 R12: ffffffffae7699bc

R13: ffffffffae7699c8 R14: ffffffffae7699bc R15: ffffffffae7699c8

ORIG_RAX: ffffffffffffff10 CS: 0010 SS: 0000

#23 [ffff9f1a36997e90] cpuidle_idle_call at ffffffffae5b081e

#24 [ffff9f1a36997ed0] arch_cpu_idle at ffffffffae0366de

#25 [ffff9f1a36997ee0] cpu_startup_entry at ffffffffae0fd7ba

#26 [ffff9f1a36997f28] start_secondary at ffffffffae0580d7

#27 [ffff9f1a36997f50] start_cpu at ffffffffae0000d5

三、分析过程

1、可能存在问题的原因

我们知道，hardlock一般有两种，一种是关中断时间过长，超过了阈值，系统通过NMI发送来

收集信息，另外一种就是类似于 [https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/commit/kernel/watchdog.c?id=7edaeb6841dfb27e362288ab8466ebdc4972e867](https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/commit/kernel/watchdog.c?id=7edaeb6841dfb27e362288ab8466ebdc4972e867)

这种案例的，其中以第一种案例比较常见。而常见的关中断时间长，也有多种情况，比如长时间抢spinlock

抢不到，比如只是代码bug导致运行时间过长，本文属于第二种情况。

2、具体原因的分析

/*

 * High resolution timer interrupt

 * Called with interrupts disabled

 */

void hrtimer_interrupt(struct clock_event_device *dev)

既然是时间长，就要看一下这个阈值是多少：

crash> p watchdog_thresh

watchdog_thresh = $1 = 60

默认是10s，说明这个环境上有人因为hardlock的问题加大过这个阈值。

反汇编对应的代码，分析如下：

crash> task_group ffff9f4a03ea9400

struct task_group {

css = {

cgroup = 0xffff9f4a57564000,

crash> cgroup.sibling 0xffff9f4a57564000

sibling = {

next = 0xffff9f4a84df3c10,

prev = 0xffff9f1b5b435a10

}

crash> list 0xffff9f4a84df3c10 |wc -l

4460

说明该task_group有4459个兄弟，也就是同一等级的task_group 有这么多，4459比 4460少一个是因为要减去父cgroup的作为链表串接头的计数。

  

我们知道，cfs在支持组调度之后，每个task_group创建的两个定时器，一个是周期性定时器，也就是堆栈中的sched_cfs_period_timer ，

另外一个是 slack_timer,用来归还时间给总池子的，本文出问题的就是第一个定时器。

因为hrtimer是关中断运行的，所以需要解决堆栈中这个定时器为什么会运行这么长时间。

经过分析代码，有两种可能，一种如下：

static enum hrtimer_restart sched_cfs_period_timer(struct hrtimer *timer) {

    struct cfs_bandwidth *cfs_b =

        container_of(timer, struct cfs_bandwidth, period_timer);

    ktime_t now;

    int overrun;

    int idle = 0;

    raw_spin_lock(&cfs_b->lock);

    for (;;) {-----------一直循环

        now = hrtimer_cb_get_time(timer);//其实就是ktime_get，也就是当前时间

        overrun = hrtimer_forward(timer, now, cfs_b->period);

        if (!overrun)-------假设overrun一直非0则不退出

            break;

        idle = do_sched_cfs_period_timer(cfs_b, overrun);

    }

    raw_spin_unlock(&cfs_b->lock);

    return idle ? HRTIMER_NORESTART : HRTIMER_RESTART; }

  

这是一种可能，还有一种可能如下：

static int do_sched_cfs_period_timer(struct cfs_bandwidth *cfs_b, int overrun)

{。。。

    while (throttled && cfs_b->runtime > 0 && !cfs_b->distribute_running) {////当前还是被throttled并且还有时间没有分发完毕,并且没有并发分发

        runtime = cfs_b->runtime;//剩余的时间继续分发

       cfs_b->distribute_running = 1;//设置正在分发的标志

        raw_spin_unlock(&cfs_b->lock);

        /* we can't nest cfs_b->lock while distributing bandwidth */

        runtime = distribute_cfs_runtime(cfs_b, runtime,

                         runtime_expires);

        raw_spin_lock(&cfs_b->lock);

        cfs_b->distribute_running = 0;

        throttled = !list_empty(&cfs_b->throttled_cfs_rq);//判断当前是否被throttled

        cfs_b->runtime -= min(runtime, cfs_b->runtime);//防止变成负数

    }

}

因为第二个循环受限于一种条件，也就是当前被限流的task_group不为空，同时还有时间没有分发完毕，同时没有在并发分发，否则则会break，

我们先来分析第二种可能，先看看当前被限流的task_group链表是否为空，这个是串接在父task_group中的。

crash> task_group.parent ffff9f4a03ea9400

parent = 0xffff9f78790edc00

crash> task_group.cfs_bandwidth.throttled_cfs_rq 0xffff9f78790edc00

cfs_bandwidth.throttled_cfs_rq = {

next = 0xffff9f782c78c900,

prev = 0xffff9f48362bd900

},

crash> list 0xffff9f782c78c900 |wc -l

46

除去链表头，说明还有45个task_group被限流，满足一个条件，那runtime呢？

crash> task_group.cfs_bandwidth.runtime 0xffff9f78790edc00

cfs_bandwidth.runtime = 0,

初步看起来好像不符合循环的条件，那有没有可能是前面一直循环，但是到crash的时候，这个值被修改为了0呢？

那就需要查看一下调用 distribute_cfs_runtime 的时候的 runtime是多少，runtime是第二个参数，也就是rsi，我们需要获取栈中的数据，

crash> dis -l distribute_cfs_runtime

/usr/src/debug/kernel-3.10.0-957.27.2.el7/linux-3.10.0-957.27.2.el7.x86_64/kernel/sched/fair.c: 3487

0xffffffffae0e4860 <distribute_cfs_runtime>: nopl 0x0(%rax,%rax,1) [FTRACE NOP]

0xffffffffae0e4865 <distribute_cfs_runtime+5>: push %rbp

0xffffffffae0e4866 <distribute_cfs_runtime+6>: mov %rsp,%rbp

0xffffffffae0e4869 <distribute_cfs_runtime+9>: push %r15 0xffffffffae0e486b <distribute_cfs_runtime+11>: push %r14 0xffffffffae0e486d <distribute_cfs_runtime+13>: push %r13 0xffffffffae0e486f <distribute_cfs_runtime+15>: push %r12

0xffffffffae0e4871 <distribute_cfs_runtime+17>: push %rbx

0xffffffffae0e4872 <distribute_cfs_runtime+18>: sub $0x18,%rsp

0xffffffffae0e4876 <distribute_cfs_runtime+22>: mov %rsi,-0x40(%rbp)----将初始剩余时间压栈，也就是栈中的000000003b9aca00,也就是十进制的 1000000000

#15 [ffff9f487e043e98] distribute_cfs_runtime at ffffffffae0e496a

ffff9f487e043ea0: 000000003b9aca00 ffff9f782c78a100 ----我们的rsi被压栈在此

ffff9f487e043eb0: 51afefcdce629c68 ffff9f78790edd80

ffff9f487e043ec0: ffff9f78790edd48 ffff9f78790ede40

ffff9f487e043ed0: 0018a72ed3fa2963 0000000000000001

ffff9f487e043ee0: ffff9f487e043f18 ffffffffae0e4b67

#16 [ffff9f487e043ee8] sched_cfs_period_timer at ffffffffae0e4b67

ffff9f487e043ef0: ffff9f78790edd80 ffff9f487e055960

ffff9f487e043f00: ffff9f487e0559a0 ffffffffae0e4a90

ffff9f487e043f10: ffff9f487e055a98 ffff9f487e043f70

ffff9f487e043f20: ffffffffae0c71e3

  

crash> pd 0x000000003b9aca00

$2 = 1000000000

而带宽设置的时候，参数如下：

crash> cfs_bandwidth.distribute_running,runtime,period,quota

crash> ffff9f78790edd48

distribute_running = true

runtime = 0

period = {

tv64 = 23148000

}

quota = 1000000000----------------每个period补充的时间，

所以说，能确定进入 distribute_cfs_runtime的时候，是第一次循环，因为rsi入参的值就是quota配置的值。也就是说，我们推断的第二种可能性被推翻了，因为假设是第二次循环且有限流，则不可能入参为1000000000。

这样就说明不是 do_sched_cfs_period_timer 的while 循环导致了hardlock的检测，当然我们也不能排除一次while循环耗时很长的情况，而 do_sched_cfs_period_timer 的执行时间长短取决于 throttled_cfs_rq 链表的长短，目前crash的时候是有 45 个task_group 被限流，

有没有一种可能，很多个限流的task_group，导致处理很长时间，直到crash的时刻还剩下45个未处理完毕了？通过走读代码，这种概率是存在的，

但是我们认为这种概论非常非常小，因为我们 当前对应的内核版本3.10.0.957内核，已经合入了 id=c06f04c70489b9deea3212af8375e2f0c2f0b184 这个补丁的。哪怕最极端的情况，当出现 4459个 task_group被限流，需要 distribute_cfs_runtime 来解除限流，对应的时间消耗也达不到60s之多。

所以我们需要回到 第一种可能性去，就是 sched_cfs_period_timer 出现了多次循环。

第一种可能需要出现循环的条件是：

hrtimer_forward 返回非0值，也就是出现了overrun。

overrun就是当前时间减去timer->node.expires  为 cfs_b->period 的整数倍的次数，也就是相当于本来应该在 

overrun就是当前时间减去timer->timer->node.expires  时刻回调的定时器没有及时执行

period_timer = {

node = {

node = {

__rb_parent_color = 18446637938508750208, rb_right = 0x0, rb_left = 0x0 }, expires = {

tv64 = 6955566201684000

}

},

3、复现

需要制造这样一种情况的条件就是：cfs_b→period比较小，另外待解除限流的进程足够多就行，好的，下面我们就模仿一下这种情况：通过复现，我们来验证一下猜测：

第一步，创建一个耗cpu的进程

# cat caq.c

#include <stdio.h>

int main(int argc,char* argv[])

{

int i=0;

while(1)

i++;

}

然后gcc -g -o caq.o caq.c

这个caq.o下面会用到。

#!/bin/bash

mkdir -p /sys/fs/cgroup/cpu/user.slice/1

echo 1000> /sys/fs/cgroup/cpu/user.slice/1/cpu.cfs_period_us

#因为我们有60个核，就设置54个吧，

echo 54000> /sys/fs/cgroup/cpu/user.slice/1/cpu.cfs_quota_us

for i in {1..2000}

do

mkdir -p /sys/fs/cgroup/cpu/user.slice/1/$i

temp="_caq_$i"

echo $temp

./caq.o "$temp" &

pid=$(ps -ef |grep -i $temp|grep -v grep |awk '{print $2}') echo $pid >/sys/fs/cgroup/cpu/user.slice/1/$i/cgroup.procs

done

在一个目录下创建2000个cg，嗯，很快，我们就触发了crash

如果你在线上执行了上述操作，你可以更新自己的简历了。。。。

四、结论

那么怎么解决这个问题呢？查看上游的kernel git记录，我们找到一个相关的commitid=2e8e19226，看似把问题解决了，如下：

for (;;) {

@@ -4899,6 +4902,28 @@ static enum hrtimer_restart sched_cfs_period_timer(struct hrtimer *timer)

  if (!overrun)

  break;

+ if (++count > 3) {

+ u64 new, old = ktime_to_ns(cfs_b->period);

+

+ new = (old * 147) / 128; /* ~115% */

+ new = min(new, max_cfs_quota_period);

+

+ cfs_b->period = ns_to_ktime(new);

+

+ /* since max is 1s, this is limited to 1e9^2, which fits in u64 */

+ cfs_b->quota *= new;

+ cfs_b->quota /= old;

+

+ pr_warn_ratelimited(

+ "cfs_period_timer[cpu%d]: period too short, scaling up (new cfs_period_us %lld, cfs_quota_us = %lld)\n",

+ smp_processor_id(),

+ new/NSEC_PER_USEC,

+ cfs_b->quota/NSEC_PER_USEC);

+

+ /* reset count so we don't come right back in here */

+ count = 0;

+ }

+

这个补丁通过放大period来尝试解决这个问题，但是如果仔细分析的话，也不一定能解决，比如你变态地创建更多cpu消耗型的cg，period升级也可能会触发crash，毕竟分发时间的时候，中间有一把spinlock的时间是争抢时间是不可控的。

当然比改之前确实概率降低很多，或者干脆touch_nmi_watchdog一下？这个就留给喜欢和社区打交道的同学了。

从业务的角度说，尽量不要在一个目录下创建那么多耗cpu的cg，同时，将period尽量放大一些比较好。

[![](http://www.wowotech.net/content/uploadfile/201605/ef3e1463542768.png)](http://www.wowotech.net/support_us.html)

« [一例hardened_usercopy的bug解析](http://www.wowotech.net/linux_kenrel/480.html) | [zRAM内存压缩技术原理与应用](http://www.wowotech.net/memory_management/zram.html)»

**评论：**

**[海安人才网](http://https//www.harcw.com)**  
2020-12-14 11:49

现在服务器我都用centos7

[回复](http://www.wowotech.net/linux_kenrel/479.html#comment-8157)

**[安庆](http://www.wowotech.net/)**  
2021-05-08 14:33

@海安人才网：如果不是创建那么多cgroup，没事的

[回复](http://www.wowotech.net/linux_kenrel/479.html#comment-8234)

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
    
    - [Linux电源管理(7)_Wakeup events framework](http://www.wowotech.net/pm_subsystem/wakeup_events_framework.html)
    - [蓝牙协议分析(8)_BLE安全机制之白名单](http://www.wowotech.net/bluetooth/ble_white_list.html)
    - [X-018-KERNEL-串口驱动开发之serial console](http://www.wowotech.net/x_project/serial_driver_porting_3.html)
    - [Linux I2C framework(3)_I2C consumer](http://www.wowotech.net/comm/i2c_consumer.html)
    - [X-024-OHTHERS-在windows平台下使用libusb](http://www.wowotech.net/x_project/libusb_on_windows.html)
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