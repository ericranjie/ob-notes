
Linux内核之旅 _2024年03月06日 21:16_ _陕西_

以下文章来源于嵌入式ARM和Linux ，作者tupeloshen

- 1 执行信号的默认动作
- 2 捕获信号
- 3 系统调用的重新执行
- 4 x86_64架构-do_signal()

前面我们已经介绍了内核注意到信号的到来，调用相关函数更新进程描述符以便进程接收处理信号。但是，如果目标进程此时没有运行，内核则推迟传递信号。现在，我们看看内核如何处理进程挂起的信号。

正如第4章的`从中断和异常返回`一节中提到的，内核允许在进程返回到用户态执行之前，检查进程的`TIF_SIGPENDING`标志。因此，内核每次完成中断或异常的处理后，都会检查挂起信号是否存在。为了处理非阻塞的挂起信号，内核调用`do_signal()`函数，其接受2个参数：

- `regs`
  `current`当前进程的用户态寄存器内容在内核栈中保存位置的地址。
- `oldset`
  用来保存阻塞信号位掩码数组的变量地址。

对`do_signal()`的描述，主要集中在信号传递的通用机制；真实的代码中涵盖了许多细节，比如处理竞态条件和其它特殊情况（如`冻结系统`、`生成核心转储`、`停止和杀死整个线程组`等等。我们将忽略这些细节。

如前所述，`do_signal()`函数通常只在`CPU`打算返回到用户态时才会被调用。所以，如果中断处理程序里调用`do_signal()`，函数直接返回：

`if ((regs->xcs & 3) != 3)       return 1;   `

如果`oldset`是`NULL`，使用`current->blocked`的地址初始化它：

`if (!oldset)       oldset = &current->blocked;   `

`do_signal()`的核心是一段循环代码，重复调用`dequeue_signal()`函数，直到私有和共享挂起信号队列没有被阻塞的挂起信号。`dequeue_signal()`的返回值存储在局部变量`signr`中，如果等于`0`，意味着所有挂起信号都被处理完，则`do_signal()`完成；如果返回非零，就有挂起信号等待被处理。`do_signal()`处理完这个信号后，会再次调用`dequeue_signal()`函数。

`dequeue_signal()`首先考虑私有挂起信号队列中的所有信号（数字从小到大），然后是共享挂起队列中的信号。它会更新相应的数据结构，标识该信号不再挂起并返回信号值。其中，涉及到清除`current->pending.signal`或`current->signal->shared_pending.signal`中的相应位，并调用`recalc_sigpending()`更新`TIF_SIGPENDING`的值。

让我们看一下`do_signal()`如何处理`dequeue_signal()`返回的每个挂起信号。首先，检查当前接收进程是否正在被其它进程监控；如果是这种情况，`do_signal()`调用`do_notify_parent_cldstop()`和`schedule()`使监控线程意识到该信号处理。

然后，`do_signal()`将待处理信号的`k_sigaction`数据结构的地址加载到局部变量`ka`中：

`ka = &current->sig->action[signr-1];   `

依赖具体内容，可能执行三类动作：忽略信号，执行默认动作或执行信号处理程序。

当传递的信号被忽略，`do_signal()`则继续处理其它挂起信号：

`if (ka->sa.sa_handler == SIG_IGN)       continue;   `

接下来的两节，我们将描述如何执行默认动作和信号处理程序。

# 1 执行信号的默认动作

如果`ka->sa.sa_handler`等于`SIG_DFL`，`do_signal()`执行信号的默认动作。唯一的例外是，当接收进程是`init`时，这种情况下，信号会被抛弃：

`if (current->pid == 1)       continue;   `

对于其它进程，忽略信号的默认处理也非常简单：

`if (signr==SIGCONT || signr==SIGCHLD ||           signr==SIGWINCH || signr==SIGURG)       continue;   `

对于默认动作是`stop`的信号，则会停止线程组中所有进程。为此，`do_signal()`将它们的状态设置为`TASK_STOPPED`，然后调用`schedule()`调度进程：

`if (signr==SIGSTOP || signr==SIGTSTP ||           signr==SIGTTIN || signr==SIGTTOU) {       if (signr != SIGSTOP &&               is_orphaned_pgrp(current->signal->pgrp))           continue;       do_signal_stop(signr);   }   `

`SIGSTOP`和其它信号有些许不同：`SIGSTOP`总是停止线程组，而其它信号只有在线程组处于孤儿进程组中时才会停止该线程组。`POSIX`标准指明，只要线程组中某个进程在同一个会话的不同进程组中有一个父进程，它就不是孤儿进程组。因此，如果父进程已死，但是，发起进程的用户仍然处于登录中，该进程组就不是孤儿进程组。

`do_signal_stop()`检查当前进程是否是线程组中第一个被停止的进程。如果是，它负责停止所有进程：本质上，该函数将信号描述符中的`group_stop_count`字段设置为正值，并唤醒线程组中的每个进程。然后，每个进程依次查看此字段以识别正在进行的`组停止`，将其状态更改为`TASK_STOPPED`，并调用`schedule()`重新调度进程。`do_signal_stop()`函数还向线程组`leader`的父进程发送`SIGCHLD`信号，除非父进程设置了`SIGCHLD`的`SA_NOCLDSTOP`标志。

默认动作为`dump`的信号会在进程的工作目录中创建核心转储文件：该文件列出了进程地址空间和寄存器的完整内容。`do_signal()`创建核心转储文件之后，会杀死线程组。其余`18`个信号的默认动作是`terminate`，就是杀死进程。为此，调用`do_group_exit()`，执行一个优雅的`group exit`处理程序（可以参考第3章的`进程终止`一节）

# 2 捕获信号

如果信号指定了处理程序，则`do_signal()`执行该程序。通过调用`invoking handle_signal()`

`handle_signal(signr, &info, &ka, oldset, regs);   if (ka->sa.sa_flags & SA_ONESHOT)       ka->sa.sa_handler = SIG_DFL;   return 1;   `

如果接收的信号设置了`SA_ONESHOT`标志，则必须将其重置为默认操作，以便再次出现相同信号将不会触发信号处理程序的执行。注意`do_signal()`在处理完单个信号后是如何返回的。在下次调用`do_signal()`之前，不会考虑其他挂起的信号。这种方法确保了实时信号将按适当的顺序处理。

执行信号处理程序是一项相当复杂的任务，因为在用户态和内核态之间切换时需要小心地切换堆栈。我们在这里详细解释一下:

`信号处理程序`是由用户进程定义的函数，包含在用户代码段中。`handle_signal()`函数在内核态运行，而信号处理程序在用户态运行；这意味着当前进程必须首先在用户态执行信号处理程序，然后才能被允许恢复其“正常”执行。此外，当内核试图恢复进程的正常执行时，内核堆栈不再包含被中断程序的硬件上下文，因为内核堆栈在每次从用户态转换到内核态时都会被清空。

下图`11-2`说明了捕获信号的函数执行流程。假设非阻塞信号被发送给进程。中断或异常发生时，进程切换到内核态。在即将返回到用户态之前，内核调用`do_signal()`函数，依次处理信号（`handle_signal()`）并配置用户态栈（`setup_frame()`或`setup_rt_frame()`）。进程切换到用户态后，开始执行信号处理程序，因为该处理程序的地址被强制加载到了`PC`程序计数器中。当信号程序终止后，调用`setup_frame()`或`setup_rt_frame()`将返回代码加载到用户态栈中。这段返回代码会调用`sigreturn()`和`rt_sigreturn()`系统调用；相应的服务例程会将正常程序的硬件上下文内容拷贝到内核态栈并将用户态栈恢复到其原始状态（`restore_sigcontext()`）。当系统调用终止时，正常程序继续其执行。

![[Pasted image 20240924152400.png]]

图`11-2` 捕获一个信号

现在，让我们看一下其执行细节：

## 2.1 Setting up the frame

为了正确设置进程的用户态栈，`handle_signal()`函数既可以调用`setup_frame()`（对于那些不需要`siginfo_t`的信号），也可以调用`setup_rt_frame()`（对于那些确定需要`siginfo_t`的信号）。具体调用哪个函数，依赖于信号的`sigaction`表中`sa_flags`字段的`SA_SIGINFO`标志。

接下来，我们看一下`setup_frame()`函数的具体实现：（`Linux`内核版本是`v2.6.11`，文件位置：`arch/x86_64/kernel/signal.c`）

```cpp
/* 这些符号的定义在vsyscall内存页中，查看vsyscall-sigreturn.S文件*/   extern void __user __kernel_sigreturn;   extern void __user __kernel_rt_sigreturn;      static void setup_frame(int sig, struct k_sigaction *ka,               sigset_t *set, struct pt_regs * regs)   {       void __user *restorer;       struct sigframe __user *frame;       int err = 0;       int usig;          frame = get_sigframe(ka, regs, sizeof(*frame));          if (!access_ok(VERIFY_WRITE, frame, sizeof(*frame)))           goto give_sigsegv;          usig = current_thread_info()->exec_domain           && current_thread_info()->exec_domain->signal_invmap           && sig < 32           ? current_thread_info()->exec_domain->signal_invmap[sig]           : sig;          err = __put_user(usig, &frame->sig);       if (err)           goto give_sigsegv;          err = setup_sigcontext(&frame->sc, &frame->fpstate, regs, set->sig[0]);       if (err)           goto give_sigsegv;          if (_NSIG_WORDS > 1) {           err = __copy_to_user(&frame->extramask, &set->sig[1],                         sizeof(frame->extramask));           if (err)               goto give_sigsegv;       }          restorer = &__kernel_sigreturn;       if (ka->sa.sa_flags & SA_RESTORER)           restorer = ka->sa.sa_restorer;          /* Set up to return from userspace.  */       err |= __put_user(restorer, &frame->pretcode);               /*        * This is popl %eax ; movl $,%eax ; int $0x80        *        * WE DO NOT USE IT ANY MORE! It's only left here for historical        * reasons and because gdb uses it as a signature to notice        * signal handler stack frames.        */       err |= __put_user(0xb858, (short __user *)(frame->retcode+0));       err |= __put_user(__NR_sigreturn, (int __user *)(frame->retcode+2));       err |= __put_user(0x80cd, (short __user *)(frame->retcode+6));          if (err)           goto give_sigsegv;          /* 为信号处理程序配置寄存器 */       regs->esp = (unsigned long) frame;       regs->eip = (unsigned long) ka->sa.sa_handler;       regs->eax = (unsigned long) sig;       regs->edx = (unsigned long) 0;       regs->ecx = (unsigned long) 0;          /* 恢复用户态的段寄存器 */       set_fs(USER_DS);       regs->xds = __USER_DS;       regs->xes = __USER_DS;       regs->xss = __USER_DS;       regs->xcs = __USER_CS;          /* 在进入信号处理程序时清除TF标志，但通知正在单步跟踪的跟踪器，        * 跟踪器也可能希望在信号处理程序内部进行单步执行        */       regs->eflags &= ~TF_MASK;       if (test_thread_flag(TIF_SINGLESTEP))           ptrace_notify(SIGTRAP);       // ...省略，打印调试信息，然后返回。      give_sigsegv:       force_sigsegv(sig, current);   }   
```

`setup_frame()`接收4个参数，如下所示：

- `sig`
  信号
- `ka`
  信号的`k_sigaction`表地址
- `oldset`
  阻塞信号的位掩码组地址
- `regs`
  用户态寄存器内容在内核栈的保存位置

`setup_frame()`将一个称为`frame`的数据结构压倒用户态栈中，该数据结构存储着处理信号和能够正确返回到`sys_sigreturn()`函数的所需要信息。`frame`是一个`sigframe`表，包含以下字段（参见图`11-3`）：

- `pretcode`

  信号处理程序的返回地址。其实就是`__kernel_sigreturn`标签处的汇编代码。

- `sig`

  信号，信号处理程序需要的一个参数。

- `sc`

  包含用户态进程即将切换到内核态之前的进程上下文内容，其数据类型为`sigcontext`（这些信息是从`current`的内核态栈中拷贝而来）。另外，它还包含一个进程阻塞信号的位数组。

- `fpstate`

  用来保存用户态进程的浮点寄存器信息，数据结构类型为`_fpstate`。（参见第3章的`保存和加载FPU、MMX和XMM寄存器`）。

- `extramask`

  指定阻塞实时信号的位数组。

- `retcode`

  发起`sigreturn()`系统调用的8字节代码。在`Linux`早期版本中，这段代码用来从信号处理程序返回；但`Linux 2.6`版本以后，仅用作符号签名，以便调试器可以识别信号的栈帧。

![[Pasted image 20240924152411.png]]

图`11-3` 用户态栈上的`frame`

`setup_frame()`函数调用`get_sigframe()`计算`frame`第一个内存位置，因为该内存位置位于用户态栈上，所以函数返回的值为`(regs->esp - sizeof(struct sigframe)) & 0xfffffff8`。

> - Linux允许进程调用`signaltstack()`系统调用为它们的信号处理程序指定一个替换栈；这个特性也是`X/Open`标准要求的。如果使用的是替换栈，`get_sigframe()`函数返回的是替换栈中的一个地址。对于此特性，我们不过多讨论，从概念上讲，其与常规信号处理非常类似。

因为在`x86`架构上，栈是向下增长的，所以，`frame`的首地址等于当前栈顶位置的地址减去`frame`的大小，结果按照8字节对齐。

返回地址使用`access_ok`进行验证：如果合法，函数重复调用`__put_user()`，以便填充`frame`的所有字段。`pretcode`字段初始化为`&__kernel_sigreturn`，这是`vsyscall`内存页上的一段汇编代码地址。（参见第10章的`通过sysenter指令发起系统调用`的一节）

接下来，修改内核态栈的`regs`内容，保证当`current`切换到用户态时，`CPU`控制权能够传递给信号处理程序：

`regs->esp = (unsigned long) frame;       regs->eip = (unsigned long) ka->sa.sa_handler;       regs->eax = (unsigned long) sig;       regs->edx = regs->ecx = 0;       regs->xds = regs->xes = regs->xss = __USER_DS;       regs->xcs = __USER_CS;`

最后，`setup_frame()`函数将保存在内核态栈上的段寄存器复位成用户态默认值而终止。现在，信号处理程序所需的信息都在用户态栈顶位置了。

`setup_rt_frame()`函数与`setup_frame()`类似，但是它把一个扩展帧（数据结构为`rt_sigframe`）存放到了用户态栈中，该扩展帧还包括与信号有关的`siginfo_t`表的内容。此外，该函数还将`pretcode`字段指向`vsyscall`内存页中的`__kernel_rt_sigreturn`代码段。

## 2.2 计算信号标志

配置完用户态栈后，`handle_signal()`函数检查与该信号相关的标志。如果该信号没有设置`SA_NODEFER`标志，则在信号处理程序执行期间，`sigaction`表中的`sa_mask`字段中的所有信号必须被阻塞，以便该信号快速处理完成：

`if (!(ka->sa.sa_flags & SA_NODEFER)) {           spin_lock_irq(&current->sighand->siglock);           sigorsets(&current->blocked, &current->blocked, &ka->sa.sa_mask);           sigaddset(&current->blocked, sig);           recalc_sigpending(current);           spin_unlock_irq(&current->sighand->siglock);       }`

正如先前描述的，`recalc_sigpending()`函数检查该进程是否有非阻塞的挂起信号，并设置其相应的`TIF_SIGPENDING`标志。

完成之后，返回`do_signal()`，随即也返回。

## 2.3 启动信号处理程序

当从`do_signal()`返回后，当前进程切换到用户态执行。因为`setup_frame()`的准备工作，`eip`寄存器指向了信号处理程序的第一条指令，而`esp`指向了压入用户态栈顶的`frame`的第一个内存位置。于是，开始执行信号处理程序。

## 2.4 终止信号处理程序

当信号处理程序执行完成时，其栈顶的返回地址指向`vsyscall`内存页（`frame`中的`pretcode`字段）

`__kernel_sigreturn:       popl %eax       movl $__NR_sigreturn, %eax       int $0x80   `

因此，信号（也就是`frame`中的`sig`字段）被从栈中丢弃；然后，调用`sigreturn()`系统调用。

`sys_sigreturn()`函数计算`regs`（类型为`pt_regs`）的地址，其中包含用户进程的硬件上下文内容，以便完成内核态切换到用户态执行。因为我们在从内核态切换到用户态执行信号处理程序的过程中，内核态栈已经被破坏，所以需要重新建立一个临时内核态栈，数据来源就是用户态栈中配置的`frame`数据结构。

`` asmlinkage int sys_sigreturn(unsigned long __unused)   {       /* 建立进程在内核态的临时栈 */       struct pt_regs *regs = (struct pt_regs *) &__unused;       // 内核态栈中用户存储       struct sigframe __user *frame = (struct sigframe __user *)(regs->esp - 8);       sigset_t set;       int eax;          /* 验证`frame`数据结构是否正确 */       if (verify_area(VERIFY_READ, frame, sizeof(*frame)))           goto badframe;       /* 处理实时信号 */       if (__get_user(set.sig[0], &frame->sc.oldmask)           || (_NSIG_WORDS > 1           && __copy_from_user(&set.sig[1], &frame->extramask,                       sizeof(frame->extramask))))           goto badframe;          /* 将在信号处理程序期间阻塞的信号恢复挂起状态 */       sigdelsetmask(&set, ~_BLOCKABLE);       spin_lock_irq(&current->sighand->siglock);       current->blocked = set;       recalc_sigpending();       spin_unlock_irq(&current->sighand->siglock);              /* 将用户态栈中的frame中保存的用户进程硬件上下文拷贝到内核态栈，并移除frame */       if (restore_sigcontext(regs, &frame->sc, &eax))           goto badframe;       return eax;          /* 错误数据处理 */   badframe:       force_sig(SIGSEGV, current);       return 0;   }    ``

`sys_sigreturn()`函数计算出`regs`（类型为`pt_regs`）的地址，它包含用户进程的硬件上下文内容（参考第10章的`参数传递`一节。根据`regs`中的`esp`字段，就能推断出用户栈中的`frame`地址。

然后，从`frame`的`sc`字段中拷贝在调用信号处理程序之前被阻塞的信号（位数组）到当前进程`current`的`blocked`字段中。也就是将这些被阻塞的信号解除阻塞。调用`recalc_sigpending()`将这些信号重新加入到挂起信号队列中。

接下来，`sys_sigreturn()`函数需要将`frame`的`sc`字段中的进程硬件上下文拷贝到内核态栈，并从用户态栈中移除`frame`数据。这两步的完成都是`restore_sigcontext()`函数实现的。

如果信号是由系统调用发送的（比如，`rt_sigqueueinfo()`），要求信号相关的`siginfo_t`表数据，其机制与上面类似。扩展帧中的`pretcode`字段指向`__kernel_rt_sigreturn`标签处的汇编代码（位于`vsyscall`内存页），这段代码会调用`rt_sigreturn()`系统调用。相应的系统服务例程`sys_rt_sigreturn()`将扩展帧中的进程硬件上下文拷贝到内核态栈，并且将扩展帧从用户态栈中移除，以便恢复原始的用户态栈。

# 3 系统调用的重新执行

对于系统调用请求，内核有时不能立即满足。这时候，发起系统调用的进程会被置成`TASK_INTERRUPTIBLE`或`TASK_UNINTERRUPTIBLE`状态。

如果进程处于`TASK_INTERRUPTIBLE`状态且其它进程发送信号给它，内核在没有完成系统调用的情况下将进程置为`TASK_RUNNING`（参考第4章的`从中断和异常返回`一节）。当进程想要切换回用户态，同时信号传递过来时，系统调用服务例程还没有完成其工作，所以会返回错误码`EINTR`、`ERESTARTNOHAND`、`ERESTART_RESTARTBLOCK`、`ERESTARTSYS`、`ERESTARTNOINTR`。

事实上，在这种场景下，用户态进程能得到的错误码只能是`EINTR`，这意味着系统调用还没有完成。应用编程者可以检查这个错误码并决定是否重新发起系统调用。其余的错误码由内核内部使用，指定是否在信号处理程序结束之后自动重新执行系统调用。

表`11-11` 列出了未完成系统调用相关的错误码，以及它们三种信号默认行为的影响。表中的术语说明如下：

- `Terminate`

  系统调用将不会自动重新执行；进程将切换到用户态下`int $0x80`或`sysenter`之后的指令处继续执行，同时，通过寄存器`eax`返回`-EINTR`值。

- `Reexecute`

  内核强制用户进程重新加载系统调用号（`eax`），然后重新调用`int $0x80`或`sysenter`；而进程不会意识到重新执行，也不会传递错误码给它。

- `Depends`

  只有被传递的信号设置了`SA_RESTART`标志，系统调用才会被重新执行；否则，系统调用将终止并返回错误码`-EINTR`。

表`11-11` 系统调用的重新执行

|信号行为|EINTR|ERESTARTSYS|ERESTARTNOHAND  <br>ERESTART_RESTARTBLOCK\*|ERESTARTNOINTR|
|---|---|---|---|---|
|Default|`Terminate`|`Reexecute`|`Reexecute`|`Reexecute`|
|Ignore|`Terminate`|`Reexecute`|`Reexecute`|`Reexecute`|
|Catch|`Terminate`|`Depends`|`Terminate`|`Reexecute`|

> - `ERESTARTNOHAND`和`ERESTART_RESTARTBLOCK`重启系统调用的机制不同。

在传递信号时，内核必须在重新执行它之前确保进程发起了系统调用。这就是`regs`寄存器上下文的`orig_eax`字段发挥关键作用的地方。让我们回忆一下，当中断或异常处理程序启动时，这个字段是如何初始化的:

- `中断`

  该字段为中断`IRQ`减去`256`（因为中断号数量小于`224`，减去`256`表示内核使用负数表示`IRQ`）（参考第4章的`为中断处理程序保存寄存器`）。

- `0x80异常（包括sysenter）`

  该字段包含系统调用号（第10章的`进入和推出系统调用`一节）。

- `其它异常`

  该字段为`–1`（参考第4章的`为异常处理程序保存寄存器`）。

因此，`orig_eax`中的非负值意味着信号唤醒了一个在系统调用中休眠的可中断进程（`TASK_INTERRUPTIBLE`）。服务例程意识到了系统调用被中断，因此返回一个前面提到的错误码。

## 3.1 重新启动`non-caught`信号中断的系统调用

对于被忽略或执行默认动作的信号，`do_signal()`分析系统调用的错误码，判断系统调用是否自动重新执行，如表`11-1`所示。如果系统调用必须重启，则修改`regs`上下文内容：`eip-2`表示将`eip`指向`int $0x80`或`sysenter`，`eax`包含系统调用号：

`if (regs->orig_eax >= 0) {           if (regs->eax == -ERESTARTNOHAND || regs->eax == -ERESTARTSYS ||                   regs->eax == -ERESTARTNOINTR) {               regs->eax = regs->orig_eax;               regs->eip -= 2;           }           if (regs->eax == -ERESTART_RESTARTBLOCK) {               regs->eax = __NR_restart_syscall;               regs->eip -= 2;           }       }`

`regs->eax`包含着系统调用服务例程的返回码（参加第10章的`进入和退出系统调用`一节）。因为`int $0x80`和`sysreturn`指令都是2个字节长度，所以`eip-2`指向了`int $0x80`或`sysenter`，可以再次触发系统调用。

错误码`ERESTART_RESTARTBLOCK`是特殊的，因为`eax`被设置为了`restart_syscall()`系统调用号；因此，用户不会重新启动被信号中断的同一个系统调用。此错误码只有与时间有关的系统调用使用，这些系统调用重新启动时，应该调整其用户态参数。典型的例子是`nanosleep()`系统调用（参考第6章的`动态定时器的应用：nanosleep()系统调用`）：假设进程调用它来暂停执行20毫秒，随后过了10毫秒之后发生了一个信号。如果系统调用还和平常一样重启，那么，总的延时时间会超过30毫秒。

相反，如果`nanosleep()`系统调用服务例程被中断，则使用特殊服务例程的地址填充`current`进程的`thread_info`结构体的`restart_block`字段，并返回`ERESTART_RESTARTBLOCK`错误码。`sys_restart_syscall()`服务例程只执行前面特殊的服务里程，计算首次调用和重新启动之间经过的时间，从而调整延时。

## 3.2 重新启动`caught`信号中断的系统调用

如果信号需要捕获处理，`handle_signal()`分析错误码，根据`sigaction`中的`SA_RESTART`标志，判断是否需要重启：

`if (regs->orig_eax >= 0) {           switch (regs->eax) {               case -ERESTART_RESTARTBLOCK:               case -ERESTARTNOHAND:                   regs->eax = -EINTR;                   break;               case -ERESTARTSYS:                   if (!(ka->sa.sa_flags & SA_RESTART)) {                       regs->eax = -EINTR;                       break;                   }               /* fallthrough */               case -ERESTARTNOINTR:                   regs->eax = regs->orig_eax;                   regs->eip -= 2;           }       }`

如果必须重新启动系统调用，`handle_signal()`的处理方式与`do_signal()`完全相同；否则，它会向用户进程返回一个`-EINTR`错误码。

# 4 x86_64架构-do_signal()

Linux内核版本是`v2.6.11`，文件位置：`arch/x86_64/kernel/signal.c`：

`` /*    * 注意init是一个特殊进程：它不会收到不想处理的信号。所以，即使错误地发送    * `SIGKILL`信号给它，也不会杀死它。    */   int do_signal(struct pt_regs *regs, sigset_t *oldset)   {       struct k_sigaction ka;       siginfo_t info;       int signr;          /* 如果不是返回到用户态，则直接返回。 */       if ((regs->cs & 3) != 3) {           return 1;       }             // ...省略          if (!oldset)           oldset = &current->blocked;          signr = get_signal_to_deliver(&info, &ka, regs, NULL);       if (signr > 0) {           /*            * 在将信号传递到用户空间之前重新启动多有观察点。            * 如果观察点在内核内部触发，寄存器将被清除。            */           if (current->thread.debugreg7)               asm volatile("movq %0,%%db7"    : : "r" (current->thread.debugreg7));              /* 传递信号 */           handle_signal(signr, &info, &ka, oldset, regs);           return 1;       }      no_signal:       /* 是否是系统调用 */       // ...省略(见前面第3.1节的处理)       return 0;   }    ``

`x86-64`架构的`handle_signal()`函数（文件位置：`arch/x86_64/kernel/signal.c`）：

`static void   handle_signal(unsigned long sig, siginfo_t *info, struct k_sigaction *ka,           sigset_t *oldset, struct pt_regs *regs)   {       // ... 省略（调试信号信息）          // 省略（被中断的系统调用的相关处理）          // 省略（IA32_EMULATION配置）          setup_rt_frame(sig, ka, info, oldset, regs);          if (!(ka->sa.sa_flags & SA_NODEFER)) {           spin_lock_irq(&current->sighand->siglock);           sigorsets(&current->blocked,&current->blocked,&ka->sa.sa_mask);           sigaddset(&current->blocked,sig);           recalc_sigpending();           spin_unlock_irq(&current->sighand->siglock);       }   }   `

`i386`架构的`handle_signal()`函数（文件位置：`arch/i386/kernel/signal.c`）：

`static void   handle_signal(unsigned long sig, siginfo_t *info, struct k_sigaction *ka,             sigset_t *oldset, struct pt_regs * regs)   {       // 省略（被中断的系统调用的相关处理）          /* Set up the stack frame */       if (ka->sa.sa_flags & SA_SIGINFO)           setup_rt_frame(sig, ka, info, oldset, regs);       else           setup_frame(sig, ka, oldset, regs);          if (!(ka->sa.sa_flags & SA_NODEFER)) {           spin_lock_irq(&current->sighand->siglock);           sigorsets(&current->blocked,&current->blocked,&ka->sa.sa_mask);           sigaddset(&current->blocked,sig);           recalc_sigpending();           spin_unlock_irq(&current->sighand->siglock);       }   }   `

`static void setup_rt_frame(int sig, struct k_sigaction *ka, siginfo_t *info,                  sigset_t *set, struct pt_regs * regs)   {       struct rt_sigframe __user *frame;       struct _fpstate __user *fp = NULL;        int err = 0;       struct task_struct *me = current;          if (used_math()) {           fp = get_stack(ka, regs, sizeof(struct _fpstate));            frame = (void __user *)round_down((unsigned long)fp - sizeof(struct rt_sigframe), 16) - 8;              if (!access_ok(VERIFY_WRITE, fp, sizeof(struct _fpstate))) {            goto give_sigsegv;           }              if (save_i387(fp) < 0)                err |= -1;        } else {           frame = get_stack(ka, regs, sizeof(struct rt_sigframe)) - 8;       }          if (!access_ok(VERIFY_WRITE, frame, sizeof(*frame))) {           goto give_sigsegv;       }          if (ka->sa.sa_flags & SA_SIGINFO) {            err |= copy_siginfo_to_user(&frame->info, info);           if (err) {                goto give_sigsegv;       }       }                  /* Create the ucontext.  */       err |= __put_user(0, &frame->uc.uc_flags);       err |= __put_user(0, &frame->uc.uc_link);       err |= __put_user(me->sas_ss_sp, &frame->uc.uc_stack.ss_sp);       err |= __put_user(sas_ss_flags(regs->rsp),                 &frame->uc.uc_stack.ss_flags);       err |= __put_user(me->sas_ss_size, &frame->uc.uc_stack.ss_size);       err |= setup_sigcontext(&frame->uc.uc_mcontext, regs, set->sig[0], me);       err |= __put_user(fp, &frame->uc.uc_mcontext.fpstate);       if (sizeof(*set) == 16) {            __put_user(set->sig[0], &frame->uc.uc_sigmask.sig[0]);           __put_user(set->sig[1], &frame->uc.uc_sigmask.sig[1]);        } else {               err |= __copy_to_user(&frame->uc.uc_sigmask, set, sizeof(*set));       }          /* Set up to return from userspace.  If provided, use a stub          already in userspace.  */       /* x86-64 should always use SA_RESTORER. */       if (ka->sa.sa_flags & SA_RESTORER) {           err |= __put_user(ka->sa.sa_restorer, &frame->pretcode);       } else {           /* could use a vstub here */           goto give_sigsegv;        }          if (err) {            goto give_sigsegv;       }       #ifdef DEBUG_SIG       printk("%d old rip %lx old rsp %lx old rax %lx\n", current->pid,regs->rip,regs->rsp,regs->rax);   #endif          /* Set up registers for signal handler */       {            struct exec_domain *ed = current_thread_info()->exec_domain;           if (unlikely(ed && ed->signal_invmap && sig < 32))               sig = ed->signal_invmap[sig];       }        regs->rdi = sig;       /* In case the signal handler was declared without prototypes */        regs->rax = 0;            /* This also works for non SA_SIGINFO handlers because they expect the          next argument after the signal number on the stack. */       regs->rsi = (unsigned long)&frame->info;        regs->rdx = (unsigned long)&frame->uc;        regs->rip = (unsigned long) ka->sa.sa_handler;          regs->rsp = (unsigned long)frame;          set_fs(USER_DS);       if (regs->eflags & TF_MASK) {           if ((current->ptrace & (PT_PTRACED | PT_DTRACE)) == (PT_PTRACED | PT_DTRACE)) {               ptrace_notify(SIGTRAP);           } else {               regs->eflags &= ~TF_MASK;           }       }      #ifdef DEBUG_SIG       printk("SIG deliver (%s:%d): sp=%p pc=%p ra=%p\n",           current->comm, current->pid, frame, regs->rip, frame->pretcode);   #endif          return;      give_sigsegv:       force_sigsegv(sig, current);   }   `


​
