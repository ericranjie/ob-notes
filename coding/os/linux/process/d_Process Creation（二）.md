
作者：[linuxer](http://www.wowotech.net/author/3 "linuxer") 发布于：2014-4-28 15:40 分类：[进程管理](http://www.wowotech.net/sort/process_management)

本文是[Process Creation（一）](http://www.wowotech.net/linux_kenrel/Process-Creation-1.html)的延续，主要内容包括：

1、进程描述符中Realtime Mutex相关数据结构的初始化

2、子进程如何复制父进程的credentials

3、per-task delay accounting的处理

4、子进程如何复制父进程的flag

# 七、初始化Realtime Mutex相关的成员

> static void rt_mutex_init_task(struct task_struct \*p)\
> {\
> raw_spin_lock_init(&p->pi_lock);\
> #ifdef CONFIG_RT_MUTEXES\
> p->pi_waiters = RB_ROOT;\
> p->pi_waiters_leftmost = NULL;\
> p->pi_blocked_on = NULL;\
> p->pi_top_task = NULL;\
> #endif\
> }

Mutex是一种人民群众喜闻乐见的内核同步方式，不过real time mutex是什么呢？real time mutex是被设计用来支PI-futexes的。什么是PI？什么又是futex？PI是优先级继承（Priority Inheritance），该方法是用来解决优先级翻转问题的。什么是优先级翻转（Priority Inversion）？它是一种调度延迟现象。一般而言，调度器总是优先调度到优先级高的进程（线程），但是，当同步资源被较低优先级的进程所拥有（也就是说持有锁），高优先级的进程未能获取该同步资源，这时候，高优先级进程要等到持有锁的进程释放该资源后才能被调度到。下面的图片更加详细的描述了该问题：

![[Pasted image 20241024221158.png]]

低优先级进程和高优先级进程都需要访问一个公共资源，因此需要一个mutex来保护对该公共资源的访问。

T0时刻，只有低优先级进程处于可运行状态，运行过程中，在T1时刻，低优先级进程访问公共资源，并且持有了mutex lock，T2时刻，由于外部事件，导致中优先级进程进入可运行状态，中优先级进程就绪进入可运行状态，由于优先级高于正在运行的低优先级进程，低优先级进程被抢占（没有unlock mutex），中优先级进程被调度执行。同样地T3时刻，由于外部事件，高优先级进程抢占中优先级进程。高优先级进程运行到T4时刻，需要访问公共资源，但该资源被更低优先级的进程所拥有，高优先级进程只能被挂起等待该资源，而此时处于可运行状态的线程中，中优先级进程由于优先级高而被调度执行。

上述现象中，优先级高的进程要等待优先级低的进程完之后才能被调度，更加严重的是如果中优先级进程执行很费时的操作，显然高优先级进程的被调度时机就不能保证，整个实时调度的性能就很差了。

为了解决上述由于优先级翻转引起的问题，很多操作系统引入了优先级继承（Priority Inheritance）的解决方法。优先级继承的方法是这样的，当高优先级进程在等待低优先级的进程程占用的竞争资源时，为了使低优先级的进程能够尽快获得调度运行（以便释放高优先级进程需要的竞争资源），由操作系统kernel把低优先级进程的优先级提高到等待竞争资源高优先级进程的优先级。

OK，了解完这些内容之后，我们回到了futex。linux内核提供了一个叫做快速用户空间互斥（Fast User-Space Mutexes）的锁的机制，简称futex，通过这样的机制用户空间程序可以实现对互斥资源的快速访问。为什么提供futex这样的机制？如何使用？在用户空间如何互斥？为何能够快速？问题太多了，下次我会启动一个专题来描述futex。

# 八、process credentials

> retval = -EAGAIN;\
> if (atomic_read(&p->real_cred->user->processes) >=\
> task_rlimit(p, RLIMIT_NPROC)) {\
> if (p->real_cred->user != INIT_USER &&\
> !capable(CAP_SYS_RESOURCE) && !capable(CAP_SYS_ADMIN))\
> goto bad_fork_free;\
> }\
> current->flags &= ~PF_NPROC_EXCEEDED;
>
> retval = copy_creds(p, clone_flags);\
> if (retval \< 0)\
> goto bad_fork_free;

上面的这段代码主要是处理如何复制新创建进程的credentials。在task struct数据结构中，下面的数据结构描述了进程这个对象的credentials：

> const struct cred \_\_rcu *real_cred; /* objective and real subjective task\
> \* credentials (COW) \*/\
> const struct cred \_\_rcu *cred;    /* effective (overridable) subjective task\
> \* credentials (COW) \*/

虽然本网站的[process credentials](http://www.wowotech.net/linux/19.html)描述了一些进程credentials的相关知识，但那是基于2.6.29版本之前的描述。随着内核的演进，从2.6.29版本开始，内核对于credential有了进一步的抽象。首先，credential不再是进程特有的，而是内核对象（进程、file、socket等）的一个属性集合。而这些内核对象的属性被分成两类：一类是该对象作为动作的发起者，操作其他内核对象时候需要用的credential属性，被称为subjective context。另外一类是本内核对象被其他内核对象操作时候需要用到的credential属性，被称为objective context。举一个简单的例子：对于内核中的进程对象，可能存在下面的动作：

1、该进程访问文件系统中的文件

2、在进行文件的asynchronous I/O操作的时候，文件会向进程发送信号。

对于进程这个内核对象而言，在上面的场景1中，要使用进程的subjective context，而在场景2中，使用进程的objective context。内核中，struct cred就是对credential的抽象，进程描述符（struct task_struct）、文件描述符（struct file ）这些内核对象都需要定义一个struct cred的成员来代表该对象的credential信息。OK，了解了上述信息之后，我们可以继续讲述task struct数据结构中的credentials成员。在进程对象在操作其他内核对象时候使用cred成员，而在其他对象操作该进程对象的时候，需要获取该进程的credential的时候，需要使用real_cred成员。

在copy process credential之前首先进行资源检查

1、该用户是否已经创建了太多的进程，如果没有超出了resource limit，那么继续fork

2、如果超出了resource limit，但user ID是root的话，没有问题，继续

3、如果不是root，但是有CAP_SYS_RESOURCE或者CAP_SYS_ADMIN的capbility，也OK，否则的话fork失败

检查完成之后就正式开始copy credential了（注：略去CONFIG_KEYS的代码，内核中支持密钥相关的操作值得用一篇单独的文档来描述，此外，为了降低文档长度，删除了debug相关的代码）:

> int copy_creds(struct task_struct \*p, unsigned long clone_flags)\
> {\
> struct cred \*new;\
> int ret;
>
> if ( clone_flags & CLONE_THREAD  ) {  //创建线程相关的代码\
> p->real_cred = get_cred(p->cred);\
> get_cred(p->cred);      //对credential描述符增加两个计数，因为新创建线程的cred和real_cred都指向父进程的credential描述符\
> atomic_inc(&p->cred->user->processes); //属于该user的进程/线程数目增加1\
> return 0;\
> }
>
> new = prepare_creds();       //后段的代码是和fork进程相关。prepare_creds是创建一个当前task的subjective context（task->cred）的副本。\
> if (!new)\
> return -ENOMEM;
>
> if (clone_flags & CLONE_NEWUSER) {//如果父子进程不共享user namespace，那么还需要创建一个新的user namespace\
> ret = create_user_ns(new);\
> if (ret \< 0)\
> goto error_put;\
> }
>
> atomic_inc(&new->user->processes);\
> p->cred = p->real_cred = get_cred(new); //和线程的处理类似，只不过进程需要创建一个credential的副本\
> return 0;
>
> error_put:\
> put_cred(new);\
> return ret;\
> }

对于创建线程，所谓copy，也就是共享，在之前的task struct进行copy的时候，credential相关的两个指针都已经是指向同样的struct cred数据结构，完成了共享的操作（被创建进程/线程的real_cred指向其父进程的real_cred，被创建进程/线程的cred指向其父进程的cred），这些需要做一些修正：

1、被创建进程/线程的real_cred指向其父进程的cred。具体原因TODO

2、修正credential描述符的reference count

对于创建进程，内核会分配一个新的cred描述符，copy 父进程的credentials，也就是说，不是共享cred描述符，而的确是copy的动作了。如果本次fork也携带了CLONE_NEWUSER参数，打算创建一个新的user namespace，那么父子进程的username space也需要独立开来，

# 九、进程创建总数限制

> retval = -EAGAIN;\
> if (nr_threads >= max_threads)\
> goto bad_fork_cleanup_count;

nr_threads是系统当前的线程数目；max_threads是系统允许容纳的最大的线程数。由于资源（CPU、memory）受限，系统不可能无限制的创建线程，否则，系统的memory可能会被进程的内核栈消耗掉。

在系统初始化的时候（fork_init），会根据当前系统中的memory对该值进行设定。

> max_threads = mempages / (8 * THREAD_SIZE / PAGE_SIZE);
>
> if (max_threads \< 20)  // 最小也会被设定为20\
> max_threads = 20;

max_threads可以由用户进行设定。在/proc/sys目录下保存着若干的内核参数，该目录下的kernel/threads-max文件就是对系统内的可以创建的最大线程数的限制。

# 十、模块处理

> if (!try_module_get(task_thread_info(p)->exec_domain->module))\
> goto bad_fork_cleanup_count;

struct thread_info数据结构中的exec_domain成员指向了当前进程/线程的execution domain descriptor。linux kernel有一个很好的设计就是允许在其他操作系统上编译的程序在GNU/linux上执行。对于DOS或者Windows这样的操作系统，GNU/linux支持起来有些困难，多半是通过用户空间的仿真来做。但是对于POSIX兼容的操作系统，由于接口API是相同的，GNU/linux应该不会花费太多的力气，只需要处理系统调用的具体细节问题或者各种信号的编码问题。这些信息在kernel中用struct exec_domain抽象。

既然是共享了父进程的exec_domain，那么需要通过try_module_get去增加reference count（具体的copy在copy thread info的时候已经完成了）。

# 十一、per-task delay accounting的处理

> delayacct_tsk_init(p);

delayacct是一个缩写，是指per-task delay accounting。这个feature是统计每一个task的等待系统资源的时间（例如等待CPU、memeory或者IO）。这些统计信息有助于精准的确定task访问系统资源的优先级。

一个进程/线程可能会因为下面的原因而delay：

1、该进程/线程处于runnable，但是等待调度器调度执行

2、该进程/线程发起synchronous block I/O，进程/线程处于阻塞状态，直到I/O的完成

3、进程/线程在执行过程中等待page swapping in。由于内存有限，系统不可能把进程的各个段（正文段、数据段等）都保存在物理内存中，当访问那些没有在物理内存的段的地址的时候，就会有磁盘操作，导致进程delay，这里有个专业的术语叫做capacity misses

4、进程/线程申请内存，但是由于资源受限而导致page frame reclaim的动作。

系统为何对这些delay信息进行统计呢？主要让系统确定下列的优先级的时候更有针对性：

1、task priority。如果该进程/线程长时间的等待CPU，那么调度器可以调高该任务的优先级。

2、IO priority。如果该进程/线程长时间的等待I/O，那么I/O调度器可以调高该任务的I/O优先级。

3、rss limit value。引入虚拟内存后，每个进程都拥有4G的地址空间。系统中有那么多进程，而物理内存就那么多，不可能让每一个进程虚拟地址（page）都对应一个物理地址（page frame）。没有对应物理地址的那部分虚拟地址的实际内容应该保存在文件或者swap设备上，一旦要访问该地址，系统会产生异常，并处理好分配page frame，页表映射，copy磁盘内容到page frame等一系列动作。rss的全称是resident set size，表示有物理内存对应的虚拟地址空间。由于物理内存资源有限，各个进程要合理使用。rss limit value定义了各个进程rss的上限。

> struct task_delay_info \*delays;

进程描述符中的delays成员记录了该task的delay统计信息，delayacct_tsk_init就是对该数据结构进程初始化。本文先简单描述概念，后续会有专门的文件来描述进程的统计信息。

# 十二、复制进程描述符的flag

> static void copy_flags(unsigned long clone_flags, struct task_struct \*p)\
> {\
> unsigned long new_flags = p->flags;
>
> new_flags &= ~(PF_SUPERPRIV | PF_WQ_WORKER);\
> new_flags |= PF_FORKNOEXEC;\
> p->flags = new_flags;\
> }

copy_flags函数用来copy进程的flag，大部分的flag都是直接copy，但是下面的几个是例外：

1、PF_SUPERPRIV，这个flag是标识进程曾经使用了super-user privileges（并不表示该进程有超级用户的权限）。对于新创建的进程，当然不会用到super-user privileges，因此要清掉。

2、清除PF_WQ_WORKER标识。PF_WQ_WORKER是用来标识该task是一个workqueue worker。如果新创建的内核线程的确是一个workqueue worker的话，那么在其worker thread function（worker_thread）中会进行设定的。具体worker、workqueue等概念请参考Concurrency-managed workqueues相关的描述。

3、设定PF_FORKNOEXEC标识，表明本进程/线程正在fork，还没有执行exec的动作。

_原创文章，转发请注明出处。蜗窝科技_，[www.wowotech.net](http://www.wowotech.net/linux_kenrel/process-creation-2.html)。

标签: [process](http://www.wowotech.net/tag/process) [management](http://www.wowotech.net/tag/management) [do_fork](http://www.wowotech.net/tag/do_fork)

---

« [Linux电源管理(1)\_整体架构](http://www.wowotech.net/pm_subsystem/pm_architecture.html) | [Linux设备模型(8)\_platform设备](http://www.wowotech.net/device_model/platform_device.html)»

**评论：**

**小学生**\
2019-06-06 14:54

关于 process credential 有个好奇的地方，我看了prepare_creds() 和 commit_creds() 的实现，p->cred 和 p->real_cred 似乎无论何时都会指向同一块内存，那在 struct task_struct中设置 cred 和 real_cred 的意义是什么呢？

[回复](http://www.wowotech.net/process_management/process-creation-2.html#comment-7460)

**[duak](http://www.wowotech.net/)**\
2014-11-07 09:32

你们三个大牛的对话，看后受益匪浅，现在改用delay_work。谢谢你们！Thank you！\
PS：这个网站真好，里面好多知识，小僧继续默默修行 -\_-

[回复](http://www.wowotech.net/process_management/process-creation-2.html#comment-688)

**wowo**\
2014-11-06 13:53

@duak,我觉这里面可以从3个方面考虑：

1. 定时获取电量，是否有必要开线程？\
   线程的开销是比较大的，起一个timer反而会更好。
1. 如果真的需要线程的话，suspend和resume不需要处理，就算你处理了，也是多余的。因为当执行到driver的suspend时，thread早已被PM core freeze掉，当执行到driver的resume时，kthread_start是无法生效的，还得等到PM core后续的处理。
1. 如果真的需要在在suspend和resume中start和stop线程，也没关系，什么意外都不会发生。

[回复](http://www.wowotech.net/process_management/process-creation-2.html#comment-678)

**[forion](http://www.wowotech.net/)**\
2014-11-06 13:57

@wowo：wowo是搞power的专家，一般的电量检测都是用timer来做的，开线程实在是太不按套路出牌了。

[回复](http://www.wowotech.net/process_management/process-creation-2.html#comment-679)

**[linuxer](http://www.wowotech.net/)**\
2014-11-06 14:09

@forion：并不是所有的获取电量都是那么的简单，可以在timer中完成，例如电池中的电量计是HDQ 接口或者I2C接口的，这种情况下，获取电量的函数需要调用HDQ core或者I2C core的接口，而那些接口有可能睡眠的。如果在timer中调用，会死的很难看的（这不是一个100％重现的bug，偶尔会挂掉，发生在无法获取总线访问权限的时候）。

当然，即便如此，使用线程也不是一个好的方法，可以考虑workqueue。

[回复](http://www.wowotech.net/process_management/process-creation-2.html#comment-681)

**[forion](http://www.wowotech.net/)**\
2014-11-06 14:25

@linuxer：我记得你在讲中断的时候有讲到这一点。\
IRQF_ONESHOT  one shot本身的意思的只有一次的，结合到中断这个场景，则表示中断是一次性触发的，不能嵌套。对于primary handler，当然是不会嵌套，但是对于threaded interrupt handler，我们有两种选择，一种是mask该interrupt source，另外一种是unmask该interrupt source。一旦mask住该interrupt source，那么该interrupt source的中断在整个threaded interrupt handler处理过程中都是不会再次触发的，也就是one shot了。这种handler不需要考虑重入问题。\
具体是否要设定one shot的flag是和硬件系统有关的，我们举一个例子，比如电池驱动，电池里面有一个电量计，是使用HDQ协议进行通信的，电池驱动会注册一个threaded interrupt handler，在这个handler中，会通过HDQ协议和电量计进行通信。对于这个handler，通过HDQ进行通信是需要一个完整的HDQ交互过程，如果中间被打断，整个通信过程会出问题，因此，这个handler就必须是one shot的。

[回复](http://www.wowotech.net/process_management/process-creation-2.html#comment-682)

**wowo**\
2014-11-06 15:03

@linuxer：确实，如果场景复杂的话，要考虑workqueue，无论到什么时候，driver尽量不去直接使用kthread。

[回复](http://www.wowotech.net/process_management/process-creation-2.html#comment-683)

**健康生活博客**\
2014-05-31 12:47

一看代码我就晕 根本不懂

[回复](http://www.wowotech.net/process_management/process-creation-2.html#comment-170)

**[linux_fans](http://www.wowotech.net/)**\
2014-05-07 17:43

图片不清晰，使用GIF格式会好很多。

[回复](http://www.wowotech.net/process_management/process-creation-2.html#comment-121)

**[linuxer](http://www.wowotech.net/)**\
2014-05-08 09:42

@linux_fans：已改，多谢指正！

[回复](http://www.wowotech.net/process_management/process-creation-2.html#comment-122)

**[duak](http://www.wowotech.net/)**\
2014-11-05 19:57

@linuxer：@linuxer： 你好！有个问题向你请教一下：我在driver中新开了一个线程，线程中set_current_state(TASK_UINTERRUPTABLE),之后schedule_timeout(3\* Hz)。请问在schedule出去这3秒钟内，driver中其它函数还能被调用么？

[回复](http://www.wowotech.net/process_management/process-creation-2.html#comment-655)

**[forion](http://www.wowotech.net/)**\
2014-11-05 20:15

@duak：hi duak\
首先第一点，你现在的driver你指的是什么？你指的是driver的probe函数吗？\
我的理解，你在driver中启动了一个线程，那么这个线程就是独立于你这个driver存在的啦。这个线程的调度就依赖于linux的进程调度，不依赖于之前的driver，除非有其他的信号量或者其他的通信。

[回复](http://www.wowotech.net/process_management/process-creation-2.html#comment-656)

**[duak](http://www.wowotech.net/)**\
2014-11-05 21:16

@forion：哈哈!感谢，写的不是很清楚。是的，在probe中开的线程，在suspend的时候杀死线程，之所以提上面一个问题是考虑系统睡眠时调driver对应的suspend函数kill线程时，此时若线程正好是scedule出去的，是否会对系统suspend造成影响。工作时间不长，一头雾水，希望得到帮助。

[回复](http://www.wowotech.net/process_management/process-creation-2.html#comment-662)

**[wowo](http://www.wowotech.net/)**\
2014-11-05 21:47

@duak：觉得有点奇怪，driver的suspend为什么非得主动kill线程？系统suspend过程中会freeze所有的线程的。

[回复](http://www.wowotech.net/process_management/process-creation-2.html#comment-663)

**[forion](http://www.wowotech.net/)**\
2014-11-05 21:52

@wowo：而且你kill掉线程就没有这个线程了啊？那你这个线程就只有probe开始有，然后suspend就没有了啊？你说说你要实现的目的吧？感觉可以有更好的实现方式。你这个方式太奇怪了。

**[duak](http://www.wowotech.net/)**\
2014-11-06 13:28

@wowo：那freeze线程的时候也走各个driver的freeze函数么？我在driver的freeze函数中有做打印，suspend并没有看到进了driver 的freeze函数。或者你的意思是说使用内核线程的时候只用创建使用就OK，在driver不用做相应的kthread_stop动作?

**[duak](http://www.wowotech.net/)**\
2014-11-06 13:37

@forion：就是battery driver，在probe中开了个线程来处理并上报电量，然后driver中有实现suspend函数和freeze函数，系统suspend的时候需要把线程kthread_stop掉，resume的时候又kthread_run起来。而且在线程中有相应调度睡眠机制，之前的问题是，suspend走到battery_driver中，要做kthread_stop动作，但此时若线程刚好睡眠中，这种情况下是否能kthread_stop掉线程？

[回复](http://www.wowotech.net/process_management/process-creation-2.html#comment-675)

**[duak](http://www.wowotech.net/)**\
2014-11-06 13:37

@forion：就是battery driver，在probe中开了个线程来处理并上报电量，然后driver中有实现suspend函数和freeze函数，系统suspend的时候需要把线程kthread_stop掉，resume的时候又kthread_run起来。而且在线程中有相应调度睡眠机制，之前的问题是，suspend走到battery_driver中，要做kthread_stop动作，但此时若线程刚好睡眠中，这种情况下是否能kthread_stop掉线程？

[回复](http://www.wowotech.net/process_management/process-creation-2.html#comment-676)

**[linuxer](http://www.wowotech.net/)**\
2014-11-05 22:47

@duak：作为一个系统工程师或者驱动工程师，你必须理解driver中的各种上下文：\
1、driver中创建的内核线程上下文\
2、调用进程的上下文（xxx_read, xxx_write等函数）\
3、中断上下文\
4、softirq context

内核驱动的某个函数也有可能根据运行时候的调用情况属于多个上下文，我们必须动态的来看待driver的代码。

你的问题是：请问在schedule出去这3秒钟内，driver中其它函数还能被调用么？其实，如果driver的其他函数不属于这个内核线程上下文，那么它们当然是自由的被其他进程调用，亦或由于响应中断或者softirq而调用。

[回复](http://www.wowotech.net/process_management/process-creation-2.html#comment-666)

**[duak](http://www.wowotech.net/)**\
2014-11-06 13:38

@linuxer：恩恩 ，很感谢你的提点，会多多充电的

[回复](http://www.wowotech.net/process_management/process-creation-2.html#comment-677)

**[electrlife](http://www.wowotech.net/)**\
2016-04-06 16:34

@duak：驱动会在进程的上下文吗？是不是应该表述为进程的内核栈上下文。

[回复](http://www.wowotech.net/process_management/process-creation-2.html#comment-3800)

**[hello_world](http://www.wowotech.net/)**\
2016-04-06 23:16

@electrlife：内核栈上下文？我似乎没有听说过这种说法。\
驱动当然会在进程上下文，当然可以加一个修饰：内核态的进程上下文

**[electrlife](http://www.wowotech.net/)**\
2016-04-07 09:34

@linuxer：进程的上下文没错，但我的理解进程上下文会有两个堆栈被使用，当处于用户态时使用的是用户堆栈，当陷入内核态时则会使用内核堆栈，所以驱动是在内核态运行的，我想应该也是在当前进程的内核堆栈上运行，因些驱动应该不可以使用太多的局部变量。不知我的理解是否正确，还是请大牛们来论述下，以免误导！

[回复](http://www.wowotech.net/process_management/process-creation-2.html#comment-3808)

**[hello_world](http://www.wowotech.net/)**\
2016-04-08 08:48

@electrlife：你的表述也是对的，不过有两个场景是需要分开的：\
1、A进程调用open、read、write等函数，通过系统调用陷入内核态，这时候，B驱动对应的xxx_open、xxx_read、xxx_write函数被调用，在这种情况下，驱动运行在进程A的进程上下文，当然使用了A进程的内核栈。

2、A进程在用户空间欢快的执行，但是B驱动的中断来了，打断A进程的执行，虽然这时候current是A进程，在内核态执行的是B驱动的interrupt handler，而且使用的也是A进程的内核栈，但是，这时候是中断上下文

因此，不是说驱动使用了某个进程的内核栈就能说是进程上下文，因此，我反对“内核栈上下文“这种表述

[回复](http://www.wowotech.net/process_management/process-creation-2.html#comment-3812)

**zcjfish**\
2016-06-21 18:06

@hello_world：这个说明是对上下文说的比较清楚的:D

**[动漫情报站](http://cc.yikehome.com/)**\
2014-05-07 16:06

好吧， 这些高深的东西我也只能看看

[回复](http://www.wowotech.net/process_management/process-creation-2.html#comment-120)

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

  - [Linux设备模型(8)\_platform设备](http://www.wowotech.net/device_model/platform_device.html)
  - [Linux电源管理(10)\_autosleep](http://www.wowotech.net/pm_subsystem/autosleep.html)
  - [Linux reset framework](http://www.wowotech.net/pm_subsystem/reset_framework.html)
  - [X-026-KERNEL-Linux gpio driver的移植之gpio range](http://www.wowotech.net/x_project/kernel_gpio_driver_porting_2.html)
  - [Linux kernel内核配置解析(5)\_Boot options(基于ARM64架构)](http://www.wowotech.net/linux_kenrel/kernel_config_boot_option.html)

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
