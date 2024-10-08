作者：[itrocker](http://www.wowotech.net/author/295) 发布于：2015-11-24 15:01 分类：[电源管理子系统](http://www.wowotech.net/sort/pm_subsystem)

**1** **什么是进程冻结**

进程冻结技术（freezing of tasks）是指在系统hibernate或者suspend的时候，将用户进程和部分内核线程置于“可控”的暂停状态。

**2** **为什么需要冻结技术**

假设没有冻结技术，进程可以在任意可调度的点暂停，而且直到cpu_down才会暂停并迁移。这会给系统带来很多问题：

(1)有可能破坏文件系统。在系统创建hibernate image到cpu down之间，如果有进程还在修改文件系统的内容，这将会导致系统恢复之后无法完全恢复文件系统；

(2)有可能导致创建hibernation image失败。创建hibernation image需要足够的内存空间，但是在这期间如果还有进程在申请内存，就可能导致创建失败；

(3)有可能干扰设备的suspend和resume。在cpu down之前，device suspend期间，如果进程还在访问设备，尤其是访问竞争资源，就有可能引起设备suspend异常；

(4)有可能导致进程感知系统休眠。系统休眠的理想状态是所有任务对休眠过程无感知，睡醒之后全部自动恢复工作，但是有些进程，比如某个进程需要所有cpu online才能正常工作，如果进程不冻结，那么在休眠过程中将会工作异常。

**3** **代码实现框架**

冻结的对象是内核中可以被调度执行的实体，包括用户进程、内核线程和work_queue。用户进程默认是可以被冻结的，借用信号处理机制实现；内核线程和work_queue默认是不能被冻结的，少数内核线程和work_queue在创建时指定了freezable标志，这些任务需要对freeze状态进行判断，当系统进入freezing时，主动暂停运行。

kernel threads可以通过调用kthread_freezable_should_stop来判断freezing状态，并主动调用__refrigerator进入冻结；work_queue通过判断max_active属性，如果max_active=0，则不能入队新的work，所有work延后执行。

![](http://www.wowotech.net/content/uploadfile/201511/29551448348577.png)

  

标记系统freeze状态的有三个重要的全局变量：pm_freezing、system_freezing_cnt和pm_nosig_freezing，如果全为0，表示系统未进入冻结；system_freezing_cnt>0表示系统进入冻结，pm_freezing=true表示冻结用户进程，pm_nosig_freezing=true表示冻结内核线程和workqueue。它们会在freeze_processes和freeze_kernel_threads中置位，在thaw_processes和thaw_kernel_threads中清零。

fake_signal_wake_up函数巧妙的利用了信号处理机制，只设置任务的TIF_SIGPENDING位，但不传递任何信号，然后唤醒任务；这样任务在返回用户态时会进入信号处理流程，检查系统的freeze状态，并做相应处理。

任务主动调用try_to_freeze的代码如下：

1. static inline bool try_to_freeze_unsafe(void)
2. {
3. 	if (likely(!freezing(current))) //检查系统是否处于freezing状态
4. 		return false;
5. 	return __refrigerator(false); //主动进入冻结
6. }

8. static inline bool freezing(struct task_struct *p)
9. {
10. 	if (likely(!atomic_read(&system_freezing_cnt))) //系统总体进入freezing
11. 		return false;
12. 	return freezing_slow_path(p);
13. }

15. bool freezing_slow_path(struct task_struct *p)
16. {
17. 	if (p->flags & PF_NOFREEZE)  //当前进程是否允许冻结
18. 		return false;

20. 	if (pm_nosig_freezing || cgroup_freezing(p))  //系统冻结kernel threads
21. 		return true;

23. 	if (pm_freezing && !(p->flags & PF_KTHREAD)) //系统冻结用户进程
24. 		return true;

26. 	return false;
27. }

进入冻结状态直到恢复的主要函数:

bool __refrigerator(bool check_kthr_stop)

1. {
2. ...
3. 	for (;;) {
4. 		set_current_state(TASK_UNINTERRUPTIBLE);  //设置进程为UNINTERRUPTIBLE状态

6. 		spin_lock_irq(&freezer_lock);
7. 		current->flags |= PF_FROZEN;  //设置已冻结状态
8. 		if (!freezing(current) ||
9. 		    (check_kthr_stop && kthread_should_stop())) //判断系统是否还处于冻结
10. 			current->flags &= ~PF_FROZEN;  //如果系统已解冻，则取消冻结状态
11. 		spin_unlock_irq(&freezer_lock);

13. 		if (!(current->flags & PF_FROZEN))  //如果已取消冻结，跳出循环，恢复执行
14. 			break;
15. 		was_frozen = true;
16. 		schedule();
17. 	}
18. ......
19. }

  

**4** **参考文献**

(1) [http://www.wowotech.net/linux_kenrel/suspend_and_resume.html](http://www.wowotech.net/linux_kenrel/suspend_and_resume.html)

(2) [http://www.wowotech.net/linux_kenrel/std_str_func.html](http://www.wowotech.net/linux_kenrel/std_str_func.html)

(3) kenrel document: freezing-of-tasks.txt

  

标签: [Linux](http://www.wowotech.net/tag/Linux) [freeze](http://www.wowotech.net/tag/freeze)

[![](http://www.wowotech.net/content/uploadfile/201605/ef3e1463542768.png)](http://www.wowotech.net/support_us.html)

« [ARM64的启动过程之（六）：异常向量表的设定](http://www.wowotech.net/armv8a_arch/238.html) | [显示技术介绍(1)_概述](http://www.wowotech.net/display/display_tech_overview.html)»

**评论：**

**[bsp](http://www.wowotech.net/)**  
2021-07-20 17:18

提个问题：  
为什么UNINTERRUPTIBLE的task设计成 不能freeze，即freeze_task()为什么调用wake_up_state(p, TASK_INTERRUPTIBLE)而不是 wake_up_state(p, TASK_UNINTERRUPTIBLE);  
  
自问自答下：  
系统suspend的流程肯定是先freeze所有的task，然后suspend driver，类似人睡眠时 先停止跳舞/吃饭的活动，再躺下闭上眼睛 停止身体；  
但是一个 UNINTERRUPTIBLE的task，比如一个file write的操作，给disk发送了指令 并等待IO完成；如果他被freeze了，在从free_task到suspend_disk中间，disk的IO完成了，此时 之前的task已经freeze无法响应此complete或者需要重新唤醒来响应此IO的complete。  
所有，最好还是设计成UNINTERRUPTIBLE的task在freeze时失败，并循环检测几次（freeze_timeout_msecs = 20s）等待IO完成，如果IO一直没有完成，退出suspend，一个task处于suspend时间过长也是有问题的。

[回复](http://www.wowotech.net/pm_subsystem/237.html#comment-8256)

**rikeyone**  
2019-11-20 17:40

hibernate的流程是不是有点问题，我看到的代码是先执行freeze，然后在生成snapshot  
  
error = freeze_processes();  
if (error)  
    goto Exit;  
  
lock_device_hotplug();  
/* Allocate memory management structures */  
error = create_basic_memory_bitmaps();  
if (error)  
    goto Thaw;  
  
error = hibernation_snapshot(hibernation_mode == HIBERNATION_PLATFORM);  
  
  
否则，snapshot和 freeze之间是可能导致进程修改文件系统数据的

[回复](http://www.wowotech.net/pm_subsystem/237.html#comment-7759)

**nowar**  
2019-08-27 14:52

有两个问题请帮忙解答，先行谢过  
1）因为只有指明了freezable的内核线程才会进入freeze, 意思是在suspend后，其实系统中默认的内核线程原来处于running状态的依然在running状态吗？  
2）那些本来就已经在uninterruptable 状态的用户线程是否也要响应freeze请求进入freeze状态呢？如果是的话，这些状态的线程没有机会来响应这个请求吧？  
---------------------------------  
static int try_to_freeze_tasks(bool user_only)  
48                 do_each_thread(g, p) {   //这里应该遍历所有状态的线程，不管是否已经sleep了  
49                         if (p == current || !freeze_task(p))  
50                                 continue;  
--------------------------

[回复](http://www.wowotech.net/pm_subsystem/237.html#comment-7618)

**finics**  
2021-03-10 10:51

@nowar：可惜发布者可能太忙了，好像很久没有更新了。  
因为最近也在看这个部分，就借用这个平台一起讨论下吧。  
---------------------------------------------------------  
（1）我理解是的，内核线程如果没有刻意配置为freezable by set_freezable()，这个内核thread就不会被freeze  
  
(2) 其实对于这一点，我也觉得很奇怪，我就基于目前4.19 kernel的实现谈一下我的理解，希望高人来确认吧。  
基于如下的调用路径，在wake_up_state(task A)中，如果task A目前具有mask flag TASK_INTERRUPTIBLE的话，这个task A就可以被加入到running queue等待下一次的调度；如果task A 的mask flag 没有 TASK_INTERRUPTIBLE的话，不管A目前是TASK_UNINTERRUPTABLE或者TASK_NORMAL，都会调用kick_process，通过一个ipi软中断触发了当前另外一个CPU上正在运行的user mode task A 进行freezing。这里就有两个疑问，如果task A是TASK_UNINTERRUPTABLE flag，理论上不应该kick process，因为当前运行的task并不是A；另外一个疑问是是不是只有正在另外一个CPU上运行的user mode application才可能被freeze，另外一些user mode task，虽然也在running queue上，但是正好还没有被调度到，是不是就没有机会被freeze了。  
freeze_processes  
try_to_freeze_tasks  
  freeze_task  
   fake_signal_wake_up  
    signal_wake_up  
     signal_wake_up_state  
       wake_up_state

[回复](http://www.wowotech.net/pm_subsystem/237.html#comment-8203)

**finics**  
2021-03-10 11:14

@finics：对于疑问“是不是只有正在另外一个CPU上运行的user mode application才可能被freeze，另外一些user mode task，虽然也在running queue上，但是正好还没有被调度到，是不是就没有机会被freeze了。”  
自己又看了下如下代码。我觉得应该这样理解：所有的user mode task都会被添加sigpending标志，所以只要这个user mode task被调度，当系统调用ret_to_user时候，这个task就会被freeze。如果这个user mode task正在执行的话，就应该立刻kick_process，让这个task进行freezing. 感觉这里面是不是有个bug，如果这个task没有正在执行，并且时uninterrutable flag时，不应该直接调用kick_process吧。  
void signal_wake_up_state(struct task_struct *t, unsigned int state)  
{  
    set_tsk_thread_flag(t, TIF_SIGPENDING);---》这个task添加sigpending标志，  
  
    if (!wake_up_state(t, state | TASK_INTERRUPTIBLE))  
        kick_process(t);  
}

[回复](http://www.wowotech.net/pm_subsystem/237.html#comment-8204)

**finics**  
2021-03-10 11:28

@finics：突然灵光一闪，再来自问自答一下，抱歉。  
对于问题“如果这个task没有正在执行，并且是uninterrutable flag时，不应该直接调用kick_process吧”  
  
如果这个task处于TASK_UNINTERRUPTIBLE状态，如下代码并不会kick_process。见kick_process（）代码，只有当前task正在运行时候才去kick。所以可以这样理解，如果task处于TASK_UNINTERRUPTIBLE状态,除了加个sig_pending啥也不做，等这个user mode task自己去完成自己的事情；如果task处于TASK_INTERRUPTIBLE状态,把这个task直接唤醒，然后最终通过ret_to_user再把这个task freeze；如果这个task正在运行的话，kick_process，通过ipi中断让这个task别干活了，快点去freeze.  
void signal_wake_up_state(struct task_struct *t, unsigned int state)  
{  
    set_tsk_thread_flag(t, TIF_SIGPENDING);---》这个task添加sigpending标志，  
  
    if (!wake_up_state(t, state | TASK_INTERRUPTIBLE))  
        kick_process(t);  
}  
  
void kick_process(struct task_struct *p)  
{  
    int cpu;  
  
    preempt_disable();  
    cpu = task_cpu(p);  
    if ((cpu != smp_processor_id()) && task_curr(p))  
        smp_send_reschedule(cpu);  
    preempt_enable();  
}

[回复](http://www.wowotech.net/pm_subsystem/237.html#comment-8205)

**[bsp](http://www.wowotech.net/)**  
2021-07-20 17:03

@finics：是的，TASK_UNINTERRUPTIBLE的task 不会去wakeup;  
kernel-4.14中：  
static int try_to_freeze_tasks(bool user_only)  
        while (true) {  
                todo = 0;  
                for_each_process_thread(g, p) {  
                        if (p == current || !freeze_task(p))  
                                continue;  
  
                        if (!freezer_should_skip(p))  
                                todo++;  
                }  
}  
这里freeze_task会通过wake_up_state(p, TASK_INTERRUPTIBLE)，发送信号给interruptable的task，p->flags |= PF_FREEZER_SKIP，并返回true；  
后面freezer_should_skip检测p->flags，UNINTERRUPTIBLE的task->flags没有置1，返回false，todo++，表示这个task freeze失败。  
后面就会打印error并退出suspend。  
        } else if (todo) {  
                pr_cont("\n");  
                pr_err("Freezing of tasks failed after %d.%03d seconds"  
                       " (%d tasks refusing to freeze,

[回复](http://www.wowotech.net/pm_subsystem/237.html#comment-8255)

**禅机子**  
2023-03-13 00:49

@bsp：想问一下，最近我遇到一个问题，就是user_task，存在扫描文件的操作，如果刚好在suspend流程中，会出现freezing of tasks。  
有什么解决思路嘛，麻烦啦

[回复](http://www.wowotech.net/pm_subsystem/237.html#comment-8751)

**test**  
2018-06-01 15:52

从3.10的内核来看,内核线程的冻结这一块描述的不太准确,调查发现,对于内核线程的冻结,  
一部分是忽略了绝大多数的内核线程,一部分是主动调用try_to_freeze()进行的冻结操作,  
但是并没有调用kthread_freeable_should_stop.  
  
请参考一下  
https://www.kernel.org/doc/Documentation/power/freezing-of-tasks.txt  
中的"III. Which kernel threads are freezable?"  
  
相关的代码分析参考如下,  
  
static int try_to_freeze_tasks(bool user_only)  
48                 do_each_thread(g, p) {  
49                         if (p == current || !freeze_task(p))  
50                                 continue;  
51  
52                         if (!freezer_should_skip(p))  
53                                 todo++;  
54                 } while_each_thread(g, p);  
  
在冻结用户空间的进程时,内核线程是直接忽略过去的  
也就是freeze_task(p)返回的是false,  
  
bool freeze_task(struct task_struct *p)  
125         spin_lock_irqsave(&freezer_lock, flags);  
126         if (!freezing(p) || frozen(p)) {  
127                 spin_unlock_irqrestore(&freezer_lock, flags);  
128                 return false;    
129         }        
因为查看freezing(),  
34 static inline bool freezing(struct task_struct *p)  
35 {  
36         if (likely(!atomic_read(&system_freezing_cnt)))  
37                 return false;  
38         return freezing_slow_path(p);  
39 }    
而freezing_slow_path的实现如下,  
34 bool freezing_slow_path(struct task_struct *p)  
35 {  
36         if (p->flags & PF_NOFREEZE)  
37                 return false;  
38  
39         if (pm_nosig_freezing || cgroup_freezing(p))  
40                 return true;  
41  
42         if (pm_freezing && !(p->flags & PF_KTHREAD))  
43                 return true;  
44  
45         return false;  
46 }  
对于绝大多数内核线程都是PF_NOFREEZE的.  
而对于冻结用户空间进程的过程中,pm_freezing=true,所以对于内核线程,!(p->flags & PF_KTHREAD)=0,  
所以freezing()返回false,进一步讲,freeze_task()返回false,所以在try_to_freeze_tasks()的循环中直接就continue了,没有让todo++,  
  
而另一方面,对于冻结内核空间的进程而言,除去绝大多数的PF_NOFREEZE的内核线程以外,由于pm_nosig_freezing=true, freezing_slow_path()返回true,  
freezing()返回true,  
在freeze_task()中,  
bool freeze_task(struct task_struct *p)  
125         spin_lock_irqsave(&freezer_lock, flags);  
126         if (!freezing(p) || frozen(p)) {  
127                 spin_unlock_irqrestore(&freezer_lock, flags);  
128                 return false;    
129         }        
需要等待 frozen(p)返回1,而进程的冻结就是通过进程自己主动调用try_to_freeze()来完成的.  
典型的例子就是kswapd%d进程和file-storage进程.

[回复](http://www.wowotech.net/pm_subsystem/237.html#comment-6777)

**Rafe**  
2018-04-09 18:28

@wowo : applicaton被freeze，然后系统resume回来，对于app来说，像是什么都没发生一样。  
请问，application有方法知道，系统是resume回来的吗？

[回复](http://www.wowotech.net/pm_subsystem/237.html#comment-6654)

**nemo**  
2020-04-16 19:06

@Rafe：app 的 watchdog？

[回复](http://www.wowotech.net/pm_subsystem/237.html#comment-7956)

**hzm**  
2018-01-11 22:17

@wowo:  
你在下面回答的问题，我有个疑问  
a.关键点在于：freeze task的过程是“请求freeze”，也就是说给要被freeze的thread发一个信号，接下来的freeze过程是thread自行处理的。  
=>看了代码的流程，没看到有发信号的地方呀，能指出一下吗？  
b.因此kthread_freezable_should_stop是被freeze thread自己调用的。  
=>同样，没进程处理signal的时候有调用到这个函数，能一起把代码指出一下吗？  
  
非常感谢~

[回复](http://www.wowotech.net/pm_subsystem/237.html#comment-6464)

**hzm**  
2018-01-15 11:43

@hzm：看到了，应该是这里  
2138int get_signal(struct ksignal *ksig)  
2139{  
2140    struct sighand_struct *sighand = current->sighand;  
2141    struct signal_struct *signal = current->signal;  
2142    int signr;  
2143  
2144    if (unlikely(current->task_works))  
2145        task_work_run();  
2146  
2147    if (unlikely(uprobe_deny_signal()))  
2148        return 0;  
2149  
2150    /*  
2151     * Do this once, we can't return to user-mode if freezing() == T.  
2152     * do_signal_stop() and ptrace_stop() do freezable_schedule() and  
2153     * thus do not need another check after return.  
2154     */  
2155    try_to_freeze();  
2156

[回复](http://www.wowotech.net/pm_subsystem/237.html#comment-6468)

**[wowo](http://www.wowotech.net/)**  
2018-01-16 09:38

@hzm：找到就行了，最近太忙，没来得及回复，抱歉哈~~

[回复](http://www.wowotech.net/pm_subsystem/237.html#comment-6477)

**wmzjzwlzs**  
2017-11-04 09:27

1.没有设置为可冻结状态的内核线程有可能在device suspend之后还在运行？  
2.如果一个内核线程设置成可冻结的，是不是系统会等待这个线程主动冻结自己后才不调度它，如果这个线程就是不主动冻结自己，是不是系统就睡不下去了

[回复](http://www.wowotech.net/pm_subsystem/237.html#comment-6169)

**[wowo](http://www.wowotech.net/)**  
2017-11-06 09:10

@wmzjzwlzs：是的。这个问题其实很好验证，了解了kernel的机制之后，你可以写一个不可冻结的应用程序，实际测试一下。

[回复](http://www.wowotech.net/pm_subsystem/237.html#comment-6171)

**wmzjzwlzs**  
2017-11-06 15:23

@wowo：谢谢wowo大神，测试了下  
1.没有设置为可冻结状态的内核线程确实在device suspend之后还在运行  
2.我把一个内核线程设置成可冻结的，然后就是不主动冻结自己，系统确实就睡不下去了

[回复](http://www.wowotech.net/pm_subsystem/237.html#comment-6173)

**[wowo](http://www.wowotech.net/)**  
2017-11-06 16:55

@wmzjzwlzs：实践是检验真理的唯一标准啊，赞！！！～～～

[回复](http://www.wowotech.net/pm_subsystem/237.html#comment-6174)

**yinjian**  
2017-10-23 15:23

你好，对于内核线程freeze的过程，还有点疑问：执行完wake_up_state函数后，是如何跑到kthread_freezable_should_stop函数的？ ：）

[回复](http://www.wowotech.net/pm_subsystem/237.html#comment-6136)

**[wowo](http://www.wowotech.net/)**  
2017-10-24 09:27

@yinjian：关键点在于：freeze task的过程是“请求freeze”，也就是说给要被freeze的thread发一个信号，接下来的freeze过程是thread自行处理的。  
因此kthread_freezable_should_stop是被freeze thread自己调用的。  
kernel中有对应的例子，你可以看看。

[回复](http://www.wowotech.net/pm_subsystem/237.html#comment-6140)

**杰少**  
2022-03-17 20:56

@wowo：你这回答看不懂呀？感觉跟没回答一样

[回复](http://www.wowotech.net/pm_subsystem/237.html#comment-8576)

**废柴**  
2017-04-01 15:15

@wo神，  
  
有个疑惑，android6.0以后搞出来一个Doze模式，请问Doze和PM suspend有关系么，还是说Doze仅仅只是android层的一种低功耗模式，和PM suspend没毛线关系？  
  
谢谢  
  
废柴

[回复](http://www.wowotech.net/pm_subsystem/237.html#comment-5412)

**[wowo](http://www.wowotech.net/)**  
2017-04-01 16:43

@废柴：Doze是应用层的节电行为，和kernel中的PM没有关系。

[回复](http://www.wowotech.net/pm_subsystem/237.html#comment-5415)

**comlee**  
2017-02-18 14:41

@wowo  
你好，请问一个弱弱的问题。  
如果应用释放的wake_lock锁是系统中的最后一把wake_lock锁的话，那么系统会马上进入suspend吗？还是要系统中的所有应用不再跑或进入阻塞等状态才进suspend?

[回复](http://www.wowotech.net/pm_subsystem/237.html#comment-5229)

**[wowo](http://www.wowotech.net/)**  
2017-02-18 16:11

@comlee：一般情况下，负责电源管理的进度或者线程，优先级应该是最低的。  
另外，休眠的过程中，执行freeze的过程，也会给应用机会，所以休眠的时候，应该应该不跑了。

[回复](http://www.wowotech.net/pm_subsystem/237.html#comment-5230)

**comlee**  
2017-02-19 12:33

@wowo：@wowo:我看到/proc/power/wake_unlock这个节点的内核实现会在unlock时check并进入suspend流程。另外，我自己写了两个应用进行测试:  
1.    main  
     {  
         while(1)  
           ;  
     }  
发现这个系统是可以进入suspend的。  
  
但是  
    main()  
    {  
         system("echo 'mylock'>/proc/power/wake_lock");  
         while(1)  
           ;  
     }  
跑这个的话，系统进不了suspend(两个测试程序分别都是在init rc里面启动）。  
  
所以看上去系统不会等应用层的程序阻塞或休眠才suspend? 只要应用释放的是系统最后一把wake_lock就走suspend流程？

[回复](http://www.wowotech.net/pm_subsystem/237.html#comment-5231)

**[wowo](http://www.wowotech.net/)**  
2017-02-19 21:18

@comlee：是的，wakeup lock的思路就是，假如你不想让系统睡，你就要持一个锁。既然你没有保持锁，系统就假设可以睡了。

[回复](http://www.wowotech.net/pm_subsystem/237.html#comment-5232)

1 [2](http://www.wowotech.net/pm_subsystem/237.html/comment-page-2#comments)

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
    
    - Shiina  
        [一个电路（circuit）中，由于是回路，所以用电势差的概念...](http://www.wowotech.net/basic_subject/voltage.html#8926)
    - Shiina  
        [其中比较关键的点是相对位置概念和点电荷的静电势能计算。](http://www.wowotech.net/basic_subject/voltage.html#8925)
    - leelockhey  
        [你这是哪个内核版本](http://www.wowotech.net/pm_subsystem/generic_pm_architecture.html#8924)
    - ja  
        [@dream：我看完這段也有相同的想法，引用 @dream ...](http://www.wowotech.net/kernel_synchronization/spinlock.html#8922)
    - 元神高手  
        [围观首席power managerment专家](http://www.wowotech.net/pm_subsystem/device_driver_pm.html#8921)
    - 十七  
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
    
    - [per-entity load tracking](http://www.wowotech.net/process_management/PELT.html)
    - [小printf大作用（用日志打印的方式调试程序）](http://www.wowotech.net/soft/7.html)
    - [Device Tree（四）：文件结构解析](http://www.wowotech.net/device_model/dt-code-file-struct-parse.html)
    - [Binder从入门到放弃（下）](http://www.wowotech.net/binder2.html)
    - [以太网驱动的流程浅析(四)-以太网驱动probe流程](http://www.wowotech.net/linux_kenrel/469.html)
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