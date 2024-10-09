作者：[linuxer](http://www.wowotech.net/author/3 "linuxer") 发布于：2014-4-23 8:39 分类：[进程管理](http://www.wowotech.net/sort/process_management)

一、前言

为什么要写一个关于进程如何创建的文档？其实用do_fork作为关键字进行索引，你会发现网上的相关文档数以万计。作为一个内核工程师，对进程以及进程相关的内容当然是非常感兴趣，但是网上的资料并不能令我非常满意（也许是我没有检索到好的文章），一个简单的例子如下：

> static void copy_flags(unsigned long clone_flags, struct task_struct \*p)\
> {\
> unsigned long new_flags = p->flags;
>
> new_flags &= ~(PF_SUPERPRIV | PF_WQ_WORKER);\
> new_flags |= PF_FORKNOEXEC;\
> p->flags = new_flags;\
> }

上面的代码是进程创建过程的一个片段，网上的解释一般都是对代码逻辑的描述：清除PF_SUPERPRIV 和PF_WQ_WORKER这两个flag的标记，设定PF_FORKNOEXEC标记。坦率的讲，这样的代码解析没有任何意义，其实c代码都已经是非常清楚了。当然，也有的文章进行了进一步的分析，例如对PF_SUPERPRIV 被清除进行了这样的解释：表明进程是否拥有超级用户权限的PF_SUPERPRIV标志被清0。很遗憾，这样的解释不能令人信服，因为如果父进程是超级用户权限，其创建的子进程是要继承超级用户权限的。

正因为如此，我想对linux kernel中进程创建涉及的方方面面的系统知识进行梳理，在我的能力范围内对进程创建的source code进行逐行解析。一言以蔽之，do_fork的source code只是索引，重要的是与其相关的各个知识点。

由于进程创建是一个大工程，因此分成若干的部分。本文是第一部分，主要内容包括：

1、从用户空间看进程创建

2、系统调用层面看进程创建

3、trace的处理

4、参数检查

5、复制thread_info和task_struct

注：本文引用的内核代码来自3.14版本的linux kernel。

二、用户空间如何创建进程

应用程序在用户空间创建进程有两种场景：

1、创建的子进程和父进程共用一个elf文件。这种情况，elf文件中的正文段中的部分代码是父进程和子进程共享，部分代码是属于父进程，部分代码属于子进程。这种情况适合于大多数的网络服务程序。

2、创建的子进程需要加载自己的elf文件。例如shell。

为了应对这些需求，linux采用了fork then exec两段式的方式来创建进程。对于场景1，程序直接fork即可，对于场景2，使用fork then exec来应对。本文主要focus在fork操作上，对于exec的操作，在进程加载文档中描述。

应用程序可以通过fork系统调用创建进程，该新创建的进程是调用fork进程的子进程。fork之后，一个进程会象细胞分裂那样变成两个进程。子进程复制了父进程（也就是调用fork的那个进程）的绝大部分的资源（文件描述符、信号处理、当前工作目录等），更细节的信息可以参考后面具体的内核代码分析。

完全复制父进程的资源的开销非常大，特别是对于场景2，所有的开销都是完全的没有任何意义，因为系统load新的elf文件后，会重建text、data等segment。不过，在引入COW(copy-on-write)技术后，fork的开销其实也不算特别大，大部分的copy都是通过share完成的，主要的开销集中在复制父进程的页表上。在某些特定的场合下，如果程序想把复制父进程页表这一点开销也节省掉，那么linux还提供了vfork函数。Vfork和fork是类似的，除了下面两点：\
1、阻塞父进程\
2、不复制父进程的页表\
之所以vfork要阻塞父进程是因为vfork后父子进程使用的是完全相同的memory descriptor, 也就是说使用的是完全相同的虚拟内存空间, 包括栈也相同。所以两个进程不能同时运行, 否则栈就乱掉了。所以vfork后, 父进程是阻塞的，直到调用了exec系列函数或者exit函数后。这时候，子进程的mm(old_mm)需要释放掉，不再与父进程共用了，这时候就可以解 除父进程的阻塞状态。\
除了fork和vfork，Linux内核还提供的clone的系统调用接口主要用于线程的创建，这个接口提供了更多的灵活性，可以让用户指定父进程和子进程（也就是创建的进程）共享的内容。其实通过传递不同的参数，clone接口可以实现fork和vfork的功能。更多细节可以参考后面具体的内核代码分析

三、系统调用相关代码分析

fork对应的系统调用代码如下：

> #ifdef \_\_ARCH_WANT_SYS_FORK\
> SYSCALL_DEFINE0(fork)\
> {\
> #ifdef CONFIG_MMU\
> return do_fork(SIGCHLD, 0, 0, NULL, NULL);\
> #else\
> /\* can not support in nommu mode \*/\
> return(-EINVAL);\
> #endif\
> }\
> #endif

对于fork的实现，在kernel中会使用COW技术，如果没有MMU的话，也就没有虚拟地址、页表这些概念，也就无法实现COW版本的fork。在这样的条件下，如果强行实现fork，那么也只能是：

1、完全复制。也就是说，内核为子进程选择适合的地址空间，并且copy完整的父进程的地址空间到子进程。

2、禁止fork，用vfork＋exec来实现fork

上面的代码已经很清楚了，内核采用了方法2。

vfork对应的系统调用代码如下：

> #ifdef \_\_ARCH_WANT_SYS_VFORK\
> SYSCALL_DEFINE0(vfork)\
> {\
> return do_fork(CLONE_VFORK | CLONE_VM | SIGCHLD, 0,\
> 0, NULL, NULL);\
> }\
> #endif

fork和vfork的实现是和CPU architecture相关的（参见source code中的\_\_ARCH_WANT_SYS_FORK和\_\_ARCH_WANT_SYS_VFORK）。在POSIX标准中对vfork描述如下：Applications are recommended to use the fork( ) function instead of this function。也就是说，标准不建议实现vfork，但是linux kernel还是保留了该系统调用，一方面是有些应用对performance特别敏感，vfork可以获得一些的性能优势。此外，在没有MMU支持的CPU上，vfork＋exec来可以用来实现fork。

clone对应的系统调用代码如下：

> #ifdef \_\_ARCH_WANT_SYS_CLONE\
> #ifdef CONFIG_CLONE_BACKWARDS\
> \*\*SYSCALL_DEFINE5(clone, unsigned long, clone_flags, unsigned long, newsp,\
> int \_\_user \*, parent_tidptr,\
> int, tls_val,\
> int \_\_user \*, child_tidptr)\
> \*\*#elif defined(CONFIG_CLONE_BACKWARDS2)\
> SYSCALL_DEFINE5(clone, unsigned long, newsp, unsigned long, clone_flags,\
> int \_\_user \*, parent_tidptr,\
> int \_\_user \*, child_tidptr,\
> int, tls_val)\
> #elif defined(CONFIG_CLONE_BACKWARDS3)\
> SYSCALL_DEFINE6(clone, unsigned long, clone_flags, unsigned long, newsp,\
> int, stack_size,\
> int \_\_user \*, parent_tidptr,\
> int \_\_user \*, child_tidptr,\
> int, tls_val)\
> #else\
> SYSCALL_DEFINE5(clone, unsigned long, clone_flags, unsigned long, newsp,\
> int \_\_user \*, parent_tidptr,\
> int \_\_user \*, child_tidptr,\
> int, tls_val)\
> #endif\
> {\
> return do_fork(clone_flags, newsp, 0, parent_tidptr, child_tidptr);\
> }\
> #endif

在我熟悉的平台上（ARM和X86），clone的实现是上面粗体的定义。不同的CPU architecture会有一些区别（例如，参数顺序不一样，stack的增长方向等），这不是本文的主题，因此暂且略过。

从上面的代码片段可以看出，无论哪一个系统调用，最终都是使用了do_fork这个内核函数，后续我们的分析主要几种在对这个函数逐行解读。

四、trace相关的处理

> if (!(clone_flags & CLONE_UNTRACED)) {\
> if (clone_flags & CLONE_VFORK)\
> trace = PTRACE_EVENT_VFORK;\
> else if ((clone_flags & CSIGNAL) != SIGCHLD)\
> trace = PTRACE_EVENT_CLONE;\
> else\
> trace = PTRACE_EVENT_FORK;
>
> if (likely(!ptrace_event_enabled(current, trace)))\
> trace = 0;\
> }

Linux的内核提供了ptrace这样的系统调用，通过它，一个进程（我们称之 tracer，例如strace、gdb）可以观测和控制另外一个进程（被trace的进程，我们称之tracee）的执行。一旦Tracer和 tracee建立了跟踪关系，那么所有发送给tracee的信号(除SIGKILL)都会汇报给Tracer，以便Tracer可以控制或者观测 tracee的执行。例如断点的操作。Tracer程序一般会提供界面，以便用户可以设定一个断点（当tracee运行到断点时，会停下来）。当用户设定 了断点后，tracer就会保存该位置的指令，然后向该位置写入SWI \_\_ARM_NR_breakpoint（这种断点是soft break point，可以设定无限多个，对于hard break point是和CPU体系结构相关，一般支持2个）。当执行到断点位置的时候，发生软中断，内核会给tracee进程发出SIGTRAP信号，当然这个信号也会被tracer捕获。对于tracee，当收到信号的时候，无论是什么信号，甚至是ignor的信号，tracee进程都会停止运行。Tracer进程可以对tracee进行各种操作，例如观察tracer的寄存器，观察变量等等。

在了解完上述的背景之后，再来看代码就比较简单了。这个代码块控制创建进程是否向tracer上报信号，如果需要上报，那么要上报哪些信号。如果用户进程 在创建的时候有携带CLONE_UNTRACED的flag，那么该进程则不能被trace。对于内核线程，在创建的时候都会携带该flag，这也就意味着，内核线程是无法被traced，也就不需要上报event给tracer。

五、参数检查和安全检查

> if ((clone_flags & (CLONE_NEWNS|CLONE_FS)) == (CLONE_NEWNS|CLONE_FS))\
> return ERR_PTR(-EINVAL);

在 2.4.19版本之前，系统中的所有进程都是共享一个mount namespace，在这种情况下，任何进程通过mount或者umount来改变mount namespace都会反应到其他的进程中。从2.4.19版本开始，linux提供了per-process的mount namespace机制，也就是说每个进程都是拥有自己私有的mount namespace（呵呵～～～是不是有点怀念过去简单而美好的日子了）。

CLONE_NEWNS这个flag就是用来控制在clone的时候，父子进程是否要共享mount namespace的。通过fork创建的进程总是和父进程共享mount namespace的（当然子进程也可以调用unshare来解除共享）。当调用clone创建进程的时候，可以有更多的灵活性，可以通过 CLONE_NEWNS这个flag可以不和父进程共享mount namespace（注意：子进程的这个private mount namespace仍然用父进程的mount namespace来初始化，只是之后，子进程和父进程的mount namespace就分道扬镳了，这时候，子进程的mount或者umount的动作将不会影响到父进程）。

CLONE_FS flag是用来控制父子进程是否共享文件系统信息（例如文件系统的root、当前工作目录等），如果设定了该flag，那么父子进程共享文件系统信息，如 果不设定该flag，那么子进程则copy父进程的文件系统信息，之后，子进程调用chroot，chdir，umask来改变文件系统信息将不会影响到 父进程。

在内核中，CLONE_NEWNS和CLONE_FS是排他的。一个进程的文件系统信息在内核中是用struct fs_struct来抽象，这个结构中就有mount namespace的信息，因此如果想共享文件系统信息，其前提条件就是要处于同一个mount namespace中。

> if ((clone_flags & (CLONE_NEWUSER|CLONE_FS)) == (CLONE_NEWUSER|CLONE_FS))\
> return ERR_PTR(-EINVAL);

CLONE_NEWUSER这个flag是和user namespace相关的标识，在通过clone函数fork进程的时候，我们可以选择clone之前的user namespace，当然也可以通过传递该标识来创建新的user namespace。user namespace是linux kernel支持虚拟化之后引入的一个机制，可以允许系统创建不同的user namespace（之前系统只有一个user namespace）。user namespace用来管理user ID和group ID的映射。一个user namespace形成一个container，该user namespace的user ID和group ID的权限被限定在container内部。也就是说，某一个user namespace中的root（UID等于0）并非具备任意的权限，他仅仅是在该user namespace中是privileges的，在该user namespace之外，该user并非是特权用户。

CLONE_NEWUSER|CLONE_FS的组合会导致一个系统漏洞，可以让一个普通用户窃取到root的权限，具体可以参考下面的连接：\
[http://www.openwall.com/lists/oss-security/2013/03/13/10](http://www.openwall.com/lists/oss-security/2013/03/13/10)

> if ((clone_flags & CLONE_THREAD) && !(clone_flags & CLONE_SIGHAND))\
> return ERR_PTR(-EINVAL);

POSIX规定一个进程内部的多个thread要共享一个PID，但是，在linux kernel中，不论是进程还是线程，都是会分配一个task struct并且分配一个唯一的PID（这时候，PID其实就是thread ID）。这样，为了满足POSIX的线程规定，linux引入了线程组的概念，一个进程中的所有线程所共享的那个PID被称为线程组ID，也就是task struct中的tgid成员。因此，在linux kernel中，线程组ID（tgid，thread group id）就是传统意义的进程ID。对于sys_getpid系统调用，linux内核返回了tgid。对于sys_gettid系统调用，本意是要求返回线 程ID，在linux内核中，返回了task struct的pid成员。一言以蔽之，POSIX的进程ID就是linux中的线程组ID。POSIX的线程ID也就是linux中的pid。

在了解了线程组ID和线程ID之后，我们来看一看CLONE_THREAD这个flag。这个flag被设定的话，则表示被创建的子进程与父进程在一个线程组中。否则会创建一个新的线程组。

如果设定CLONE_SIGHAND这个flag，则表示创建的子进程与父进程共享相同的信号处理（signal handler）表。线程组应该共享signal handler（POSIX规定），因此，当设定了CLONE_THREAD后必须同时设定CLONE_SIGHAND

> if ((clone_flags & CLONE_SIGHAND) && !(clone_flags & CLONE_VM))\
> return ERR_PTR(-EINVAL);

设定了CLONE_SIGHAND表示共享signal handler，前提条件就是要共享地址空间（也就是说必须设定CLONE_VM），否则，无法共享signal handler。因为如果不共享地址空间，即便是同样地址的handler，其物理地址都是不一样的。

> if ((clone_flags & CLONE_PARENT) &&\
> current->signal->flags & SIGNAL_UNKILLABLE)\
> return ERR_PTR(-EINVAL);

CLONE_PARENT这个flag表示新fork的进程想要和创建该进程的cloner拥有同样的父进程。

SIGNAL_UNKILLABLE这个flag是for init进程的，其他进程不会设定这个flag。

Linux kernel会静态定义一个init task，该task的pid是0，被称作swapper（其实就是idle进程，在系统没有任何进程可调度的时候会执行该进程）。系统中的所有进程（包括内核线程）由此开始。对于用户空间进程，内核会首先创建init进程，所有其他用户空间的进程都是由init进程派生出来的。因此init进程要负责为所有用户空间的进程处理后事（否则会变成僵 尸进程）。但是如果init进程想要创建兄弟进程（其父亲是swapper），那么该进程无法由init进程回收，其父亲swapper进程也不会收养用户空间创建的init的兄弟进程，这种情况下，这类进程退出都会变成zombie，因此要杜绝。

> if (clone_flags & CLONE_SIGHAND) {\
> if ((clone_flags & (CLONE_NEWUSER | CLONE_NEWPID)) ||\
> (task_active_pid_ns(current) !=\
> current->nsproxy->pid_ns_for_children))\
> return ERR_PTR(-EINVAL);\
> }

当CLONE_SIGHAND被设定的时候，父子进程应该共享signal disposition table。也就是说，一个进程修改了某一个signal的handler，另外一个进程也可以感知的到。

CLONE_NEWPID这个flag是和PID namespace相关的标识。思路同CLONE_NEWUSER。 这两个flag是和虚拟化技术相关的。虚拟化技术就需要资源隔离，也就是说，不同的虚拟主机（实际上在一台物理主机上）资源是互相不可见的。因此，linux kernel增加了若干个name space，例如user name space、PID namespace、IPC namespace、uts namespace、network namespace等。以PID namespace为例，原来的linux kernel中，PID唯一的标识了一个process，在引入PID namespace之后，不同的namespace可以拥有同样的ID，也就是说，标识一个进程的是PID namespace ＋ PID。

CLONE_NEWUSER设定的时候，就会为fork的进程创建一个新的user namespace，以便隔离USER ID。linux 系统内的一个进程和某个user namespace内的uid和gid相关。user namespace被实现成树状结构，新的user namespace中第一个进程的uid就是0，也就是root用户。这个进程在这个新的user namespace中有超级权限，但是，在其父user namespace中只是一个普通用户。

更详细的解释TODO。

> retval = security_task_create(clone_flags);\
> if (retval)\
> goto fork_out;

这一段代码是和LinuxSecurity Modules相关的。LinuxSecurity Modules是一个安全框架，允许各种安全模型插入到内核。大家熟知的一个计算机安全模型就是selinux。具体这里就不再描述。如果本次操作通过了 安全校验，那么后续的操作可以顺利进行

六、复制内核栈、thread_info和task_struct

> retval = -ENOMEM;\
> p = dup_task_struct(current);\
> if (!p)\
> goto fork_out;

每一个用户空间进程都有一个内核栈和一个用户空间的栈（对于多线程的进程，应该有多个用户空间栈和内核栈）。内核栈和thread_info数据结构共同占用了THREAD_SIZE（一般是2个page）的memory。thread_info数据结构和CPU architecture相关，thread_info数据结构的task 成员指向进程描述符（也就是task struct数据结构）。进程描述符的stack成员指向对应的thread_info数据结构。

dup_task_struct这段代码主要动作序列包括：

1、分配内核栈和thread_info数据结构所需要的memory（统一分配），分配task sturct需要的memory。

2、设定内核栈和thread_info以及task sturct之间的联系

3、将父进程的thread_info和task_struct数据结构的内容完全copy到子进程的thread_info和task_struct数据结构

4、将task_struct数据结构的usage成员设定为2。usage成员其实就是一个reference count。之所以被设定为2，因为fork之后已经存在两个reference了，一个是自己，另外一个是其父进程。

_原创文章，转发请注明出处。蜗窝科技，[www.wowotech.net](http://www.wowotech.net/linux_kenrel/Process-Creation-1.html)。_

标签: [do_fork](http://www.wowotech.net/tag/do_fork)

[![](http://www.wowotech.net/content/uploadfile/201605/ef3e1463542768.png)](http://www.wowotech.net/support_us.html)

« [Linux设备模型(7)\_Class](http://www.wowotech.net/device_model/class.html) | [Unix的历史](http://www.wowotech.net/tech_discuss/Unix%E7%9A%84%E5%8E%86%E5%8F%B2.html)»

**评论：**

**PeterChen**\
2019-09-01 17:13

确实是目前看到写得最清楚的进程创建

[回复](http://www.wowotech.net/linux_kenrel/Process-Creation-1.html#comment-7639)

**tester 111**\
2019-07-06 20:31

@tester举报自己吗

[回复](http://www.wowotech.net/linux_kenrel/Process-Creation-1.html#comment-7513)

**tester**\
2019-07-06 15:10

举报\<\<奔跑吧linux内核>>涉嫌抄袭\
\<\<奔跑吧linux内核>>一书中，第三章-进程管理，第3.1.2节，书中内容和文章内容大体雷同，只是表述变换了一下，应该是抄袭，不知道对博主有没有帮助

[回复](http://www.wowotech.net/linux_kenrel/Process-Creation-1.html#comment-7512)

**小亮**\
2020-06-17 16:47

@tester：我在研究linuxer的中断子系统中也发现了奔跑吧linux涉嫌抄袭的问题。

[回复](http://www.wowotech.net/linux_kenrel/Process-Creation-1.html#comment-8029)

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

  - [新技能get: 订阅Linux内核邮件列表](http://www.wowotech.net/linux_application/lkml.html)
  - [Linux TTY framework(5)\_System console driver](http://www.wowotech.net/tty_framework/system_console_driver.html)
  - [perfbook memory barrier（14.2章节）中文翻译（下）](http://www.wowotech.net/kernel_synchronization/perfbook-memory-barrier-2.html)
  - [小printf大作用（用日志打印的方式调试程序）](http://www.wowotech.net/soft/7.html)
  - [Linux kernel debug技巧----开启DEBUG选项](http://www.wowotech.net/linux_application/kernel_debug_enable.html)

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
