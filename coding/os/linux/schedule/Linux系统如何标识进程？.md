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
作者：[linuxer](http://www.wowotech.net/author/3 "linuxer") 发布于：2017-2-23 19:10 分类：[进程管理](http://www.wowotech.net/sort/process_management)

一、前言

其实两年前，本站已经有了一篇关于[进程标识](http://www.wowotech.net/process_management/process_identification.html)的文档，不过非常的简陋，而且代码是来自2.6内核。随着linux container、pid namespace等概念的引入，进程标识方面已经有了天翻地覆的变化，因此我们需要对这部分的内容进行重新整理。

本文主要分成四个部分来描述进程标识这个主题：在初步介绍了一些入门的各种IDs基础知识后，在第三章我们描述了pid、pid number、pid namespace等基础的概念。第四章重点描述了内核如何将这些基本概念抽象成具体的数据结构，最后我们简单分析了内核关于进程标识的源代码（代码来自linux4.4.6版本）。

二、各种ID概述

所谓进程其实就是执行中的程序而已，和静态的程序相比，进程是一个运行态的实体，拥有各种各样的资源：地址空间（未必使用全部地址空间，而是排布在地址空间上的一段段的memory mappings）、打开的文件、pending的信号、一个或者多个thread of execution，内核中数据实体（例如一个或者多个task_struct实体），内核栈（也是一个或者多个）等。针对进程，我们使用进程ID，也就是pid（process ID）。通过getpid和getppid可以获取当前进程的pid以及父进程的pid。

进程中的thread of execution被称作线程（thread），线程是进程中活跃状态的实体。一方面进程中所有的线程共享一些资源，另外一方面，线程又有自己专属的资源，例如有自己的PC值，用户栈、内核栈，有自己的hw context、调度策略等等。我们一般会说进程调度什么的，但是实际上线程才是是调度器的基本单位。对于Linux内核，线程的实现是一种特别的存在，和经典的unix都不一样。在linux中并不区分进程和线程，都是用task_struct来抽象，只不过支持多线程的进程是由一组task_struct来抽象，而这些task_struct会共享一些数据结构（例如内存描述符）。我们用thread ID来唯一标识进程中的线程，POSIX规定线程ID在所属进程中是唯一的，不过在linux kernel的实现中，thread ID是全系统唯一的，当然，考虑到可移植性，Application software不应该假设这一点。在用户空间，通过gettid函数可以获取当前线程的thread ID。对于单线程的进程，process ID和thread ID是一样的，对于支持多线程的进程，每个线程有自己的thread ID，但是所有的线程共享一个PID。

为了方便shell进行Job controll，我们需要把一组进程组织起来形成进程组。关于这方面的概念，在[进程和终端](http://www.wowotech.net/process_management/process-tty-basic.html)文档中描述的很详细，这里就不赘述了。为了标识进程组，我们需要引入进程组ID的概念。我们一般把进程组中的第一个进程的ID作为进程组的ID，进程组中的所有进程共享一个进程组ID。在用户空间，通过setpgid、getpgid、setpgrp和getpgrp等接口函数可以访问process group ID。

经过thread ID、process ID、process group ID的层层递进，我们终于来到最顶层的ID，也就是session ID，这个ID实际上是用来标识计算机系统中的一次用户交互过程：用户登录入系统，不断的提交任务（即Job或者说是进程组）给计算机系统并观察结果，最后退出登录，销毁该session。关于session的概念，在[进程和终端](http://www.wowotech.net/process_management/process-tty-basic.html)文档中描述的也很详细，大家可以参考那份文档，这里就不赘述了。在用户空间，我们可以通过getsid、setsid来操作session ID。

三、基础概念

1、用户空间如何看到process ID

我们用下面这个block diagram来描述用户空间和内核空间如何看待process ID的：

[![getpid](http://www.wowotech.net/content/uploadfile/201702/5f093198ce1a99e8c632b2cf0e47355320170223111014.gif "getpid")](http://www.wowotech.net/content/uploadfile/201702/122dff61618dc06cc1fec3649030d47f20170223111013.gif)

从用户空间来看，每一个进程都可以调用getpid来获取标识该进程的ID，我们称之PID，其类型是pid_t。因此，我们知道在用户空间可以通过一个正整数来唯一标识一个进程（我们称这个正整数为pid number）。在引入容器之后，事情稍微复杂一点，pid这个正整数只能是唯一标识容器内的进程。也就是说，如果有容器1和容器2存在于系统中，那么可以同时存在两个pid等于a的进程，分别位于容器1和容器2。当然，进程也可以不在容器里，例如进程x和进程y，它们就类似传统的linux系统中的进程。当然，你也可以认为进程x和进程y位于一个系统级别的顶层容器0，其中包括进程x和进程y以及两个容器。同样的概念，容器2中也可以嵌套一个容器，从而形成了一个container hierarchy。

容器（linux container）是一个OS级别的虚拟化方法，基本上是属于纯软件的方法来实现虚拟化，开销小，量级轻，当然也有自己的局限。linux container主要应用了内核中的cgroup和namespace隔离技术，当然这些内容不是我们这份文档关心的，我们这里主要关心pid namespace。

当一个进程运行在linux OS之上的时候，它拥有了很多的系统资源，例如pid、user ID、网络设备、协议栈、IP以及端口号、filesystem hierarchy。对于传统的linux，这些资源都是全局性的，一个进程umount了某一个文件系统挂载点，改变了自己的filesystem hierarchy视图，那么所有进程看到的文件系统目录结构都变化了（umount操作被所有进程感知到了）。有没有可能把这些资源隔离开呢？这就是namespace的概念，而PID namespace就是用来隔离pid的地址空间的。

进程是感知不到pid namespace的，它只是知道能够通过getpid获取自己的ID，并不知道自己实际上被关在一个pid namespace的牢笼。从这个角度看，用户空间是简单而幸福的，内核空间就没有这么幸运了，我们需要使用复杂的数据结构来抽象这些形成层次结构的PID。

最后顺便说一句，上面的描述是针对pid而言的，实际上，tid、pgid和sid都是一样的概念，原来直接使用这些ID就可以唯一标识一个实体，现在我们需要用（pid namespace，ID）来唯一标识一个实体。

2、内核空间如何看到process ID

虽然从用户空间看，一个pid用一个正整数表示就足够了，但是在内核空间，一个正整数肯定是不行的，我们用一个2个层次的pid namespace来描述（也就是上面图片的情形）。pid namespace 0是pid namespace 1和2的parent namespace，在pid namespace 1中的pid等于a的那进程，对应pid namespace 0中的pid等于m的那进程，也就是说，内核态实际需要两个不同namespace中的正整数来记录一个进程的ID信息。推广开来，我们可以这么描述，在一个n个level的pid namespace hieraray中，位于x level的进程需要x个正整数ID来表示该该进程。

除此之外，内核还有记录pid namespace之间的关系：谁是根，谁是叶，父子关系……

四、内核态的数据抽象

1、如何抽象pid number？

> struct upid {  
>     int nr;  
>     struct pid_namespace *ns;  
>     struct hlist_node pid_chain;  
> };

虽然用户空间使用一个正整数来表示各种IDs，但是对于内核，我们需要使用（pid namespace，ID number）这样的二元组来表示，因为单纯的pid number是没有意义的，必须限定其pid namespace，只有这样，那个ID number才是唯一的。这样，upid中的nr和ns成员就比较好理解了，分别对应ID number和pid namespace。此外，当userspace传递ID number参数进入内核请求服务的时候（例如向某一个ID发送信号），我们必须需要通过ID number快速找到其对应的upid数据对象，为了应对这样的需求，内核将系统内所有的upid保存在哈希表中，pid_chain成员是哈希表中的next node。

2、如何抽象tid、pid、sid、pgid？

> struct pid  
> {  
>     atomic_t count;  
>     unsigned int level;  
>     struct hlist_head tasks[PIDTYPE_MAX];  
>     struct rcu_head rcu;  
>     struct upid numbers[1];  
> };

虽然其名字是pid，不过实际上这个数据结构抽象了不仅仅是一个thread ID或者process ID，实际上还包括了进程组ID和session ID。由于多个task struct会共享pid（例如一个session中的所有的task struct都会指向同一个表示该session ID的struct pid数据对象），因此存在count这样的成员也就不奇怪了，表示该数据对象的引用计数。

在了解了pid namespace hierarchy之后，level成员也不难理解，任何一个系统分配的PID都是隶属于某一个namespace的，而这个namespace又是位于整个pid namespace hierarchy的某个层次上，pid->level指明了该PID所属的namespace的level。由于pid对其parent pid namespace也是可见的，因此，这个level值其实也就表示了这个pid对象在多少个pid namespace中可见。

在多少个pid namespace中可见，就会有多少个（pid namespace，pid number）对，numbers就是这样的一个数组，表示了在各个level上的pid number。tasks成员和使用该struct pid的task们关联，我们在下一节描述。

3、进程描述符中如何体现tid、pid、sid、pgid？

由于多个task共享ID（泛指上面说的四种ID），因此在设计数据结构的时候我们要考虑两种情况：

（1）从task struct快速找到对应的struct pid

（2）从struct pid能够遍历所有使用该pid的task

在这样的要求下，我们设计了一个辅助数据结构：

> struct pid_link  
> {  
>     struct hlist_node node;  
>     struct pid *pid;  
> };

其中node是将task串接到struct pid的task struct链表中的节点，而pid指向具体的struct pid。这时候，我们可以在task struct中嵌入一个pid_link的数组：

> struct task_struct {  
> ……  
> struct pid_link pids[PIDTYPE_MAX];  
> ……  
> }

Task struct中的pids成员是一个数组，分别表示该task的tid（pid）、pgid和sid。我们定义pid的类型如下：

> enum pid_type  
> {  
>     PIDTYPE_PID,  
>     PIDTYPE_PGID,  
>     PIDTYPE_SID,  
>     PIDTYPE_MAX  
> };

一直以来我们都是说四种type，tid、pid、sid、pgid，为何这里少定义一种呢？其实开始版本的内核的确是定义了四种type的pid，但是后来为了节省内存，tid和pid合二为一了。OK，现在已经引入太多的数据结构，下面我们用一幅图片来描述数据结构之间的关系：

[![pid-arch](http://www.wowotech.net/content/uploadfile/201702/d68660154f41b3235110853e0176b9a820170223111023.gif "pid-arch")](http://www.wowotech.net/content/uploadfile/201702/38ef0d410917067a562efbdaf834094c20170223111019.gif)

对于一个进程中的多个线程而言，每一个线程都可以通过task->pids[PIDTYPE_PID].pid找到该线程对应的表示thread ID的那个struct pid数据对象。当然，任何一个线程都有其所属的进程，也就是有表示其process id的那个struct pid数据对象。如何找到它呢？这需要一个桥梁，也就是task struct中定义的thread group 成员（task->group_leader），通过该指针，一个线程总是很容易的找到其对应的线程组leader，而线程组leader对应的pid就是该线程的process ID。因此，对于一个线程，其task->group_leader->pids[PIDTYPE_PID].pid就指向了表示其process id的那个struct pid数据对象。当然，对于线程组leader，其thread ID和process ID的struct pid数据对象是一个实体，对于非线程组leader的那些普通线程，其thread ID和process ID的struct pid数据对象指向不同的实体。

struct pid有三个链表头，如果该pid仅仅是标识一个thread ID，那么其pid链表头指向的链表中只有一个元素，就是使用该pid的task struct。如果该pid表示的是一个process ID，那么pid链表头指向的链表中多个task struct，每一个元素表示了属于该进程的线程的task struct，链表中第一个task struct是thread group leader。如果该pid并不表示一个process group ID或者session ID，那么struct pid中的pgid链表头和session链表头都是指向null。如果该pid表示一个process group ID的时候，其结构如下图所示：

[![pgid-arch](http://www.wowotech.net/content/uploadfile/201702/de6614bb9b99b259190103309d5c229d20170223111025.gif "pgid-arch")](http://www.wowotech.net/content/uploadfile/201702/5182e99ec9cd45d44cdb9aaa990e278520170223111024.gif)

对于那些multi-thread进程，内核有若干个task struct和进程对应，不过为了简单，在上面图片中，进程x 对应的task struct实际上是thread group leader对应的那个task struct。这些task struct的pgid指针（task->pids[PIDTYPE_PGID].pid）指向了该进程组对应的struct pid数据对象。而该pid中的pgid链表头串联了所有使用该pid的task struct（仅仅是串联thread group leader对应的那些task struct），而链表中的第一个节点就是进程组leader。

session pid的概念是类似的，大家可以自行了解学习。

4、如何抽象 pid namespace？

好吧，这个有点复杂，暂时TODO吧。

五、代码分析

1、如何根据一个task struct得到其对应的thread ID？

> static inline struct pid *task_pid(struct task_struct *task)  
> {  
>     return task->pids[PIDTYPE_PID].pid;  
> }

同样的道理，我们也可以很容易得到一个task对应的pgid和sid。process ID有一点绕，我们首先要找到该task的thread group leader对应的task，其实一个线程的thread group leader对应的那个task的thread ID就是该线程的process ID。

2、如何根据一个task struct得到当前的pid namespace？

> struct pid_namespace *task_active_pid_ns(struct task_struct *tsk)  
> {  
>     return ns_of_pid(task_pid(tsk));  
> }

这个操作可以分成两步，第一步首先找到其对应的thread ID，然后根据thread ID找到当前的pid namespace，代码如下：

> static inline struct pid_namespace *ns_of_pid(struct pid *pid)  
> {  
>     struct pid_namespace *ns = NULL;  
>     if (pid)  
>         ns = pid->numbers[pid->level].ns;  
>     return ns;  
> }

一个struct pid实体是有层次的，对应了若干层次的（pid namespace，pid number）二元组，最顶层是root pid namespace，最底层（叶节点）是当前的pid namespace，pid->level表示了当前的层次，因此pid->numbers[pid->level].ns说明的就是当前的pid namespace。

3、getpid是如何实现的？

当陷入内核后，我们很容易获取当前的task struct（根据sp_svc的值），这是起点，后续的代码如下：

> static inline pid_t task_tgid_vnr(struct task_struct *tsk)  
> {  
>     return pid_vnr(task_tgid(tsk));  
> }

通过task_tgid可以获取该task对应的thread group leader的thread ID，其实也就是process ID。此外，通过task_active_pid_ns亦可以获取当前的pid namespace，有了这两个参数，可以调用pid_nr_ns获取该task对应的pid number：

> pid_t pid_nr_ns(struct pid *pid, struct pid_namespace *ns)  
> {  
>     struct upid *upid;  
>     pid_t nr = 0;
> 
>     if (pid && ns->level <= pid->level) {  
>         upid = &pid->numbers[ns->level];  
>         if (upid->ns == ns)  
>             nr = upid->nr;  
>     }  
>     return nr;  
> }

一个pid可以贯穿多个pid namespace，但是并非所有的pid namespace都可以检视pid，获取相应的pid number。因此，在代码的开始会进行验证，如果pid namespace的层次（ns->level）低于pid当前的pid namespace的层次，那么直接返回0。如果pid namespace的level是OK的，那么要检查该namespace是不是pid当前的那个pid namespace，如果是，直接返回对应的pid number，否则，返回0。

对于gettid和getppid这两个接口，整体的概念是和getpid类似的，不再赘述。

4、给定线程ID number的情况下，如何找对应的task struct？

这里给定的条件包括ID number、当前的pid namespace，在这样的条件下如何找到对应的task呢？我们分成两个步骤，第一个步骤是先找到对应的struct pid，代码如下：

> struct pid *find_pid_ns(int nr, struct pid_namespace *ns)  
> {  
>     struct upid *pnr;
> 
>     hlist_for_each_entry_rcu(pnr,  
>             &pid_hash[pid_hashfn(nr, ns)], pid_chain)  
>         if (pnr->nr == nr && pnr->ns == ns)  
>             return container_of(pnr, struct pid,  
>                     numbers[ns->level]);
> 
>     return NULL;  
> }

整个系统有那么多的struct pid数据对象，每一个pid又有多个level的（pid namespace，pid number）对，通过pid number和namespace来找对应的pid是一件非常耗时的操作。此外，这样的操作是一个比较频繁的操作，一个简单的例子就是通过kill向指定进程（pid number）发送信号。正是由于操作频繁而且耗时，系统建立了一个全局的哈希链表来解决这个问题，pid_hash指向了若干（具体head的数量和内存配置有关）哈希链表头。这个哈希表用来通过一个指定pid namespace和id number，来找到对应的struct upid。一旦找了upid，那么通过container_of找到对应的struct pid数据对象。

第二步是从struct pid找到task struct，代码如下：

> struct task_struct *pid_task(struct pid *pid, enum pid_type type)  
> {  
>     struct task_struct *result = NULL;  
>     if (pid) {  
>         struct hlist_node *first;  
>         first = rcu_dereference_check(hlist_first_rcu(&pid->tasks[type]),  
>                           lockdep_tasklist_lock_is_held());  
>         if (first)  
>             result = hlist_entry(first, struct task_struct, pids[(type)].node);  
>     }  
>     return result;  
> }

5、getpgid是如何实现的？

> SYSCALL_DEFINE1(getpgid, pid_t, pid)  
> {  
>     struct task_struct *p;  
>     struct pid *grp;  
>     int retval;
> 
>     rcu_read_lock();  
>     if (!pid)  
>         grp = task_pgrp(current);  
>     else {  
>         retval = -ESRCH;  
>         p = find_task_by_vpid(pid);  
>         if (!p)  
>             goto out;  
>         grp = task_pgrp(p);  
>         if (!grp)  
>             goto out;
> 
>         retval = security_task_getpgid(p);  
>         if (retval)  
>             goto out;  
>     }  
>     retval = pid_vnr(grp);  
> out:  
>     rcu_read_unlock();  
>     return retval;  
> }

当传入的pid number等于0的时候，getpgid实际上是获取当前进程的process groud ID number，通过task_pgrp可以获取该进程的使用的表示progress group ID对应的那个pid对象。如果调用getpgid的时候给出了非0的process ID number，那么getpgid实际上是想要获取指定pid number的gpid。这时候，我们需要调用find_task_by_vpid找到该pid number对应的task struct。一旦找到task struct结构，那么很容易得到其使用的pgid（该实体是struct pid类型）。至此，无论哪一种参数情况（传入的参数pid number等于0或者非0），我们都找到了该pid number对应的struct pid数据对象（pgid）。当然，最终用户空间需要的是pgid number，因此我们需要调用pid_vnr找到该pid在当前namespace中的pgid number。

getsid的代码逻辑和getpid是类似的，不再赘述。

_原创文章，转发请注明出处。蜗窝科技_

标签: [进程ID](http://www.wowotech.net/tag/%E8%BF%9B%E7%A8%8BID)

[![](http://www.wowotech.net/content/uploadfile/201605/ef3e1463542768.png)](http://www.wowotech.net/support_us.html)

« [Debian8 内核升级实验](http://www.wowotech.net/linux_application/debian8-upgrade-kernel.html) | [ARM Linux上的系统调用代码分析](http://www.wowotech.net/process_management/syscall-arm.html)»

**评论：**

**perr**  
2017-07-09 03:20

现在主副线程并不是通过二图中PID NODE串联, 而是通过task->thread_group和task->thread_node,出现两个这样的组织链表是历史问题,还未有人改进. 见[_do_fork()->copy_process()]

[回复](http://www.wowotech.net/process_management/pid.html#comment-5790)

**[codingbelief](http://www.wowotech.net/)**  
2017-03-02 17:27

想了解下哪些子系统或者场景如何使用这些 id，有没有计划加一个章节描述下这几个 id 在内核的应用场景 ~ ~  
另外，期待 cgroup、namespace、虚拟化这块的文章 ~ ~

[回复](http://www.wowotech.net/process_management/pid.html#comment-5282)

**[linuxer](http://www.wowotech.net/)**  
2017-03-03 09:30

@codingbelief：内核的各个子系统并不使用这些ID，主要是用户空间程序利用这些ID做进程间通信、job control什么的。  
  
虚拟化我的确也很感兴趣，完成进程和内存管理部分，准备向虚拟化进军。

[回复](http://www.wowotech.net/process_management/pid.html#comment-5283)

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
    
    - [X-019-KERNEL-串口驱动开发之数据收发](http://www.wowotech.net/x_project/serial_driver_porting_4.html)
    - [Linux I2C framework(2)_I2C provider](http://www.wowotech.net/comm/i2c_provider.html)
    - [Linux电源管理(15)_PM OPP Interface](http://www.wowotech.net/pm_subsystem/pm_opp.html)
    - [linux 串口调试方法](http://www.wowotech.net/159.html)
    - [Meltdown论文翻译](http://www.wowotech.net/basic_subject/meltdown.html)
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