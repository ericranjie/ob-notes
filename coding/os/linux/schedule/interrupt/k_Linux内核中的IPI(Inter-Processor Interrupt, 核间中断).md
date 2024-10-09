原创 TrickBoy CodeTrap
_2024年03月30日 00:00_ _江苏_

在Linux内核中，IPI(Inter-Processor Interrupt, 核间中断)是在多处理器系统下，一种常用的CPU间的通信机制。

该机制允许一个CPU向其余一个或多个CPU发送中断，从而触发目标CPU上相应的处理函数。

本文将从**为什么要使用IPI**，**如何使用IPI**以及**IPI的实现原理**来分析Linux中的IPI机制。

**为什么要使用IPI**

昨天，有个朋友问了我这样一个问题：

他有个任务，需要使用一个内核模块定时去获取某个CPU上的curr进程信息，应该怎么去实现代码呢？

我的**第一个想法**是：直接去遍历进程链表，依据task_struct的on_cpu字段去进行判断，不就OK了吗？

代码大致是这样：

```c
typedef struct {     char comm[TASK_COMM_LEN];     int pid;     int cpu;     int exist; } task_info_t;  static int __init my_init(void) {     int cpu, cpu_nums;     struct task_struct *task;     task_info_t *task_array, *task_info_pos;      cpu_nums = num_possible_cpus();      task_array = kzalloc(sizeof(task_info_t) * cpu_nums, GFP_KERNEL);     if (!task_array) {         printk(KERN_ERR "alloc task array failed!");         return -ENOMEM;     }      for_each_process(task) {         if (task->on_cpu) {             cpu = task_cpu(task);              task_info_pos = &task_array[cpu];              memcpy(&task_info_pos->comm, task->comm, TASK_COMM_LEN);             task_info_pos->pid = task->pid;             task_info_pos->cpu = cpu;             task_info_pos->exist = 1;         }     }      for (cpu = 0; cpu < cpu_nums; cpu++) {         task_info_pos = &task_array[cpu];          if (!task_info_pos->exist)             continue;                  printk(KERN_INFO "cpu %d pid %d comm: %s\n", cpu, task_info_pos->pid, task_info_pos->comm);     }    kfree(task_array);    return 0; }
```

输出结果如下：\
!\[\[Pasted image 20240906122829.png\]\]
!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

1. 为什么只有3个CPU上的curr信息？

因为0号进程（idle进程）不在进程链表中。其余CPU都处于idle状态，因此无法通过进程链表找到它们的curr进程信息。

2. 为什么判断依据是task_struct的on_cpu成员，而非是task->state == TASK_RUNNING呢？

因为task_struct在加入rq之后，state都是TASK_RUNNING的，无法作为进程是否在CPU上运行的证据。而on_cpu字段的更新是在context_switch（实际进程切换的前后）去设置的。PS：进程切换前去设置next（task_struct）的on_cpu为1，进程切换后去设置prev（task_struct）的on_cpu为0。

但使用这个方法明显存在一定的问题：

1. 每次都需要完整地去遍历一遍进程链表，即该任务的性能表现是与系统中的进程数量强相关的。

1. on_cpu字段的更新机制可能会导致出现在遍历时发现一个CPU上有两个task_struct的on_cpu字段皆为1的情况。

因此，**第二个想法**是去找一找内核里面现有的API可以做到这个事情吗？（指定CPU，返回CPU上当前运行的进程）

的确是有这样的函数：

```c
DECLARE_PER_CPU_SHARED_ALIGNED(struct rq, runqueues);  #define cpu_rq(cpu)    (&per_cpu(runqueues, (cpu))) #define this_rq()    this_cpu_ptr(&runqueues) #define task_rq(p)    cpu_rq(task_cpu(p)) #define cpu_curr(cpu)    (cpu_rq(cpu)->curr) #define raw_rq()    raw_cpu_ptr(&runqueues)
```

但这些函数的定义是在/kernel/sched/sched.h中的，没有办法被内核模块引用到。

**第三个想法**是一个“骚操作”，写一个kprobe函数，默认是disable的，当需要去获得信息的时候再enable这个kprobe，获取到信息后再enable它。

代码大致是这样：

```c
static atomic_t cpu_get_num;  static int my_kprobe_handler(struct kprobe *p, struct pt_regs *regs) {     int cpu = smp_processor_id();      task_info_t *task_info_pos = &per_cpu(task_array, cpu);      if (task_info_pos->exist)         return 0;      memcpy(&task_info_pos->comm, current->comm, TASK_COMM_LEN);     task_info_pos->pid = current->pid;     task_info_pos->cpu = cpu;     task_info_pos->exist = 1;      atomic_inc(&cpu_get_num);      return 0; }  static struct kprobe my_kprobe = {     .symbol_name = "__schedule",     .pre_handler = my_kprobe_handler,     .flags = KPROBE_FLAG_DISABLED, };  static int __init my_init(void) {     int ret, cpu, cpu_nums, timeout = 0;     task_info_t *task_info_pos;      ret = register_kprobe(&my_kprobe);     if (ret < 0) {         printk(KERN_ERR "register_kprobe failed, ret %d\n", ret);         return -EINVAL;     }      cpu_nums = num_possible_cpus();      for (cpu = 0; cpu < cpu_nums; cpu++) {         task_info_pos = &per_cpu(task_array, cpu);         task_info_pos->exist = 0;     }     atomic_set(&cpu_get_num, 0);      ret = enable_kprobe(&my_kprobe);     if (ret) {       printk(KERN_ERR "enable_kprobe failed, ret %d\n", ret);     }          while (atomic_read(&cpu_get_num) < cpu_nums) {       timeout++;        if (timeout > 100)         goto out;     }     disable_kprobe(&my_kprobe);      for (cpu = 0; cpu < cpu_nums; cpu++) {         task_info_pos = &per_cpu(task_array, cpu);         printk(KERN_INFO "cpu: %d pid: %d comm: %s\n", task_info_pos->cpu, task_info_pos->pid, task_info_pos->comm);     }  out:     unregister_kprobe(&my_kprobe);     return 0; }
```

输出结果如下：\
!\[\[Pasted image 20240906122913.png\]\]
!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

但这种实现的问题是：

1. 注册完kprobe函数后，实际内核text中挂载函数的地址的指令已经被替换。虽然刚开始是enable的，但仍然会有跳转的开销。

1. 获得各个CPU上进程的信息的时机依赖于挂载点的选择，如果目标函数很久都没有触发，则无疑会增加等待的时间。

这时候突然想到，kprobe方式是被动等待其余CPU触发int3中断，这种被动的方式是带来这么多问题的根本原因。所以，与其被动挨打，不如主动出击。我完全可以使用主动触发其余CPU的中断，来让它们将自己的curr信息进行报告。

因此，IPI不失为一个好选择。

**如何使用IPI**

IPI机制在很多驱动以及内核逻辑的实现上被应用很多。比如在DVFS的驱动中，一个CPU上的任务要去调整其余CPU的频率，就会使用IPI机制。又或者一个高优先级的任务加入运行队列，这时候需要使用IPI机制来通知对应CPU上的任务下处理器。

依旧按照前面的需求来完成代码：

```c
static void do_get_cpu_current(void *args) {   int cpu = smp_processor_id();   struct task_struct **task_array = (struct task_struct **)args;    task_array[cpu] = get_task_struct(current);    return; }  static int __init my_init(void) {   int cpu, current_cpu, cpu_nums;   struct task_struct *task;   struct task_struct **task_array;    cpu_nums = num_possible_cpus();   current_cpu = smp_processor_id();    task_array = kmalloc(sizeof(struct task_struct *) * cpu_nums, GFP_KERNEL);    smp_call_function(do_get_cpu_current, task_array, 1);   smp_mb();    for (cpu = 0; cpu < cpu_nums; cpu++) {     if (cpu == current_cpu)       continue;      task = task_array[cpu];      if (!task) {       printk(KERN_ERR "no task in task_array");       continue;     }      printk(KERN_INFO "cpu: %d pid: %d comm: %s\n", cpu, task->pid, task->comm);     put_task_struct(task);   }    kfree(task_array);    return 0;  } 
```

输出结果如下：\
!\[\[Pasted image 20240906122939.png\]\]
!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

这里忽略掉当前内核模块所在的CPU，输出了其余CPU上的curr信息。向其它CPU发送IPI中断使用了smp_call_function这个函数。

这个函数的的定义如下：

```c
/**  * smp_call_function(): Run a function on all other CPUs.  * @func: The function to run. This must be fast and non-blocking.  * @info: An arbitrary pointer to pass to the function.  * @wait: If true, wait (atomically) until function has completed  *        on other CPUs.  *  * Returns 0.  *  * If @wait is true, then returns once @func has returned; otherwise  * it returns just before the target cpu calls @func.  *  * You must not call this function with disabled interrupts or from a  * hardware interrupt handler or from a bottom half handler.  */ void smp_call_function(smp_call_func_t func, void *info, int wait) {   preempt_disable();   smp_call_function_many(cpu_online_mask, func, info, wait);   preempt_enable(); } EXPORT_SYMBOL(smp_call_function);
```

smp_call_function()：在除自身之外的所有CPU上运行一个函数。

func：目标函数

info：目标函数的参数

wait：是否等待其余CPU上的任务完成

注意：只能用于进程上下文。

当然除了这个API外，还有其余的API，譬如：

smp_call_function_single——在指定某个CPU上运行一个函数。

smp_call_function_any——给任意给定的CPU上运行一个函数。

除了前面提到的这些同步接口，还有异步接口（注册函数后，直接返回，可以用于非进程上下文），比如：

smp_call_function_single_async——在指定某个CPU上运行一个异步函数。

**IPI的实现原理**

知道了IPI有哪些接口，现在分析一下IPI的实现原理。

直接看一次广播给多个CPU的函数的实现：smp_call_function_many。

```c
/**  * smp_call_function_many(): Run a function on a set of CPUs.  * @mask: The set of cpus to run on (only runs on online subset).  * @func: The function to run. This must be fast and non-blocking.  * @info: An arbitrary pointer to pass to the function.  * @wait: Bitmask that controls the operation. If %SCF_WAIT is set, wait  *        (atomically) until function has completed on other CPUs. If  *        %SCF_RUN_LOCAL is set, the function will also be run locally  *        if the local CPU is set in the @cpumask.  *  * If @wait is true, then returns once @func has returned.  *  * You must not call this function with disabled interrupts or from a  * hardware interrupt handler or from a bottom half handler. Preemption  * must be disabled when calling this function.  */ void smp_call_function_many(const struct cpumask *mask,           smp_call_func_t func, void *info, bool wait) {   smp_call_function_many_cond(mask, func, info, wait * SCF_WAIT, NULL); } EXPORT_SYMBOL(smp_call_function_many);
```

接着看smp_call_function_many_cond函数：

```c
static void smp_call_function_many_cond(const struct cpumask *mask,           smp_call_func_t func, void *info,           unsigned int scf_flags,           smp_cond_func_t cond_func) {   int cpu, last_cpu, this_cpu = smp_processor_id();   struct call_function_data *cfd;   bool wait = scf_flags & SCF_WAIT;   bool run_remote = false;   bool run_local = false;   int nr_cpus = 0;    lockdep_assert_preemption_disabled();    /*    * Can deadlock when called with interrupts disabled.    * We allow cpu's that are not yet online though, as no one else can    * send smp call function interrupt to this cpu and as such deadlocks    * can't happen.    */   if (cpu_online(this_cpu) && !oops_in_progress &&       !early_boot_irqs_disabled)     lockdep_assert_irqs_enabled();    /*    * When @wait we can deadlock when we interrupt between llist_add() and    * arch_send_call_function_ipi*(); when !@wait we can deadlock due to    * csd_lock() on because the interrupt context uses the same csd    * storage.    */   WARN_ON_ONCE(!in_task());    /* Check if we need local execution. */   if ((scf_flags & SCF_RUN_LOCAL) && cpumask_test_cpu(this_cpu, mask))     run_local = true;    /* Check if we need remote execution, i.e., any CPU excluding this one. */   cpu = cpumask_first_and(mask, cpu_online_mask);   if (cpu == this_cpu)     cpu = cpumask_next_and(cpu, mask, cpu_online_mask);   if (cpu < nr_cpu_ids)     run_remote = true;    if (run_remote) {     cfd = this_cpu_ptr(&cfd_data);     cpumask_and(cfd->cpumask, mask, cpu_online_mask);     __cpumask_clear_cpu(this_cpu, cfd->cpumask);      cpumask_clear(cfd->cpumask_ipi);     for_each_cpu(cpu, cfd->cpumask) {       struct cfd_percpu *pcpu = per_cpu_ptr(cfd->pcpu, cpu);       call_single_data_t *csd = &pcpu->csd;        if (cond_func && !cond_func(cpu, info))         continue;        csd_lock(csd);       if (wait)         csd->node.u_flags |= CSD_TYPE_SYNC;       csd->func = func;       csd->info = info; #ifdef CONFIG_CSD_LOCK_WAIT_DEBUG       csd->node.src = smp_processor_id();       csd->node.dst = cpu; #endif       cfd_seq_store(pcpu->seq_queue, this_cpu, cpu, CFD_SEQ_QUEUE);       if (llist_add(&csd->node.llist, &per_cpu(call_single_queue, cpu))) {         __cpumask_set_cpu(cpu, cfd->cpumask_ipi);         nr_cpus++;         last_cpu = cpu;          cfd_seq_store(pcpu->seq_ipi, this_cpu, cpu, CFD_SEQ_IPI);       } else {         cfd_seq_store(pcpu->seq_noipi, this_cpu, cpu, CFD_SEQ_NOIPI);       }     }      cfd_seq_store(this_cpu_ptr(&cfd_seq_local)->ping, this_cpu, CFD_SEQ_NOCPU, CFD_SEQ_PING);      /*      * Choose the most efficient way to send an IPI. Note that the      * number of CPUs might be zero due to concurrent changes to the      * provided mask.      */     if (nr_cpus == 1)       send_call_function_single_ipi(last_cpu);     else if (likely(nr_cpus > 1))       arch_send_call_function_ipi_mask(cfd->cpumask_ipi);      cfd_seq_store(this_cpu_ptr(&cfd_seq_local)->pinged, this_cpu, CFD_SEQ_NOCPU, CFD_SEQ_PINGED);   }    if (run_local && (!cond_func || cond_func(this_cpu, info))) {     unsigned long flags;      local_irq_save(flags);     func(info);     local_irq_restore(flags);   }    if (run_remote && wait) {     for_each_cpu(cpu, cfd->cpumask) {       call_single_data_t *csd;        csd = &per_cpu_ptr(cfd->pcpu, cpu)->csd;       csd_lock_wait(csd);     }   } }
```

实际调用的核心部分是这一部分：

```c
       if (nr_cpus == 1)       send_call_function_single_ipi(last_cpu);     else if (likely(nr_cpus > 1))       arch_send_call_function_ipi_mask(cfd->cpumask_ipi);
```

在这之前的主要工作是：

1. 设置当前CPU的per_cpu变量cfd->cpumask为目标CPU的mask

1. 依次遍历目标CPU的mask，找到对应CPU的per_cpu变量cfd->pcpu->csd，更新csd的信息存放目标函数和参数，之后将当前csd加入到对应CPU的call_single_queue队列中

至于，wait的实现，则是在下发任务前csd_lock()函数中设置csd->node.u_flags的CSD_FLAG_LOCK标记位。

```c
csd->node.u_flags |= CSD_FLAG_LOCK;
```

在任务下发之后，去检查对应的CSD_FLAG_LOCK是否被清理掉。

```c
smp_cond_load_acquire(&csd->node.u_flags, !(VAL & CSD_FLAG_LOCK));
```

那现在完成了函数的注册，应该如何去触发中断呢？

以x86架构下的广播IPI为例子，其调用是：

```c
arch_send_call_function_ipi_mask->   smp_ops.send_call_func_ipi(mask)->     native_send_call_func_ipi->    /* apic层支持指定mask, all, allbutself三种send IPI方式 */  void native_send_call_func_ipi(const struct cpumask *mask) {   if (static_branch_likely(&apic_use_ipi_shorthand)) {     unsigned int cpu = smp_processor_id();      if (!cpumask_or_equal(mask, cpumask_of(cpu), cpu_online_mask))       goto sendmask;      if (cpumask_test_cpu(cpu, mask))       apic->send_IPI_all(CALL_FUNCTION_VECTOR);     else if (num_online_cpus() > 1)       apic->send_IPI_allbutself(CALL_FUNCTION_VECTOR);     return;   }  sendmask:   apic->send_IPI_mask(mask, CALL_FUNCTION_VECTOR); }        apic->send_IPI_mask
```

这里选择看apic_numachip.c中的代码：

```c
    .send_IPI      = numachip_send_IPI_one,   .send_IPI_mask      = numachip_send_IPI_mask,   .send_IPI_mask_allbutself  = numachip_send_IPI_mask_allbutself,   .send_IPI_allbutself    = numachip_send_IPI_allbutself,   .send_IPI_all      = numachip_send_IPI_all,   .send_IPI_self      = numachip_send_IPI_self,
```

可以看到，实现了以上这个IPI的接口。

```c
static void numachip_send_IPI_mask(const struct cpumask *mask, int vector) {   unsigned int cpu;    for_each_cpu(cpu, mask)     numachip_send_IPI_one(cpu, vector); }  static void numachip_send_IPI_one(int cpu, int vector) {   int local_apicid, apicid = per_cpu(x86_cpu_to_apicid, cpu);   unsigned int dmode;    preempt_disable();   local_apicid = __this_cpu_read(x86_cpu_to_apicid);    /* Send via local APIC where non-local part matches */   if (!((apicid ^ local_apicid) >> NUMACHIP_LAPIC_BITS)) {     unsigned long flags;      local_irq_save(flags);     __default_send_IPI_dest_field(apicid, vector,       APIC_DEST_PHYSICAL);     local_irq_restore(flags);     preempt_enable();     return;   }   preempt_enable();    dmode = (vector == NMI_VECTOR) ? APIC_DM_NMI : APIC_DM_FIXED;   numachip_apic_icr_write(apicid, dmode | vector); }
```

这里也没什么好看的了，就是写特定地址，触发中断。

起码，我看到的这里的代码，没有实现广播写的功能，在最底层还是for循环去写。

在前面看到，发送自定义IPI的时候，中断vector是CALL_FUNCTION_VECTOR，其实通过IPI机制发送的不止这一种中断，譬如还有之前提到的提醒某个CPU要进行调度的IPI。

```c
DEFINE_IDTENTRY_SYSVEC_SIMPLE(sysvec_reschedule_ipi) {   ack_APIC_irq();   trace_reschedule_entry(RESCHEDULE_VECTOR);   inc_irq_stat(irq_resched_count);   scheduler_ipi();   trace_reschedule_exit(RESCHEDULE_VECTOR); }  DEFINE_IDTENTRY_SYSVEC(sysvec_call_function) {   ack_APIC_irq();   trace_call_function_entry(CALL_FUNCTION_VECTOR);   inc_irq_stat(irq_call_count);   generic_smp_call_function_interrupt();   trace_call_function_exit(CALL_FUNCTION_VECTOR); }  DEFINE_IDTENTRY_SYSVEC(sysvec_call_function_single) {   ack_APIC_irq();   trace_call_function_single_entry(CALL_FUNCTION_SINGLE_VECTOR);   inc_irq_stat(irq_call_count);   generic_smp_call_function_single_interrupt();   trace_call_function_single_exit(CALL_FUNCTION_SINGLE_VECTOR); }
```

这里看下CALL_FUNCTION_VECTOR的处理函数，generic_smp_call_function_interrupt()。

```c
generic_smp_call_function_interrupt()->   generic_smp_call_function_single_interrupt()->     __flush_smp_call_function_queue(true)
```

\_\_flush_smp_call_function_queue函数也没有特别，就是在之前添加任务的call_single_queue队列中取出函数指针去执行，很标准的任务列表的代码。

**最后1个问题**

为什么有些接口只可以在进程上下文使用？而那个异步接口却可以在中断上下文使用？

参见前面的代码（在进程上下文使用的接口）其做wait的接口是使用per_cpu的一个全局变量去进行任务下发。如果上一个任务卡住，那么现在进行任务下发时也会卡住。

那如果将wait设置为false呢？因为需要依靠这个全局变量去下发，在set之前，默认做了一次wait的操作。

而异步的接口的实现是自己识别到csd（输入）被占用立即退出，因此不会有wait导致卡死的问题出现：

```c
int smp_call_function_single_async(int cpu, struct __call_single_data *csd) {   int err = 0;    preempt_disable();    if (csd->node.u_flags & CSD_FLAG_LOCK) {     err = -EBUSY;     goto out;   }    csd->node.u_flags = CSD_FLAG_LOCK;   smp_wmb();    err = generic_exec_single(cpu, csd);  out:   preempt_enable();    return err; } EXPORT_SYMBOL_GPL(smp_call_function_single_async); 
```

我对IPI的理解也并不算深入。

但对开发者来说，这种手段在特定场景下的效果可能也蛮不错的!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)。

![](https://mmbiz.qlogo.cn/mmbiz_jpg/TibnEdoonBKJXf2P0sGG5JBWKUIMIjlib4YekFs5IhJyEluzwzPibd7nl2IZjn7nbhEcEov3x4IrWYAoWv6HNziaGQ/0?wx_fmt=jpeg)

TrickBoy

![赞赏二维码](https://mp.weixin.qq.com/s?__biz=Mzg3MDYzOTQyNg==&mid=2247483697&idx=1&sn=60a1f9aca4382ab2c25bf24b2d9e0cbb&chksm=ce8bf31cf9fc7a0a84599e80eed1c7e850a5fcbf499d29c9462594b56654d64e4eae5369e896&mpshare=1&scene=24&srcid=0330o9kaFJmqUWH4ovHYRBl2&sharer_shareinfo=1e9468b27cfc62d78ac5873f4c0ac31b&sharer_shareinfo_first=1e9468b27cfc62d78ac5873f4c0ac31b&key=daf9bdc5abc4e8d0ecfc8525ad5670367b54640c22c5212335774d14ec26e8b41d7d371002e5c352160c37f0cbb074a524496f8051a59640605d38d5f8ee0e83e1567653bbb67bbc20a914020a5670cfb3f20853f574fb21e97c320185bcddbdcb7cf2a00973f1dc619cbd127f6e77983672d3a506bae74404b2eb517f997abb&ascene=0&uin=MTEwNTU1MjgwMw%3D%3D&devicetype=Windows+11+x64&version=63090b19&lang=zh_CN&countrycode=CN&exportkey=n_ChQIAhIQM84yUEIQLWhsciz%2BFQgTyRLmAQIE97dBBAEAAAAAADQYNQXp7MMAAAAOpnltbLcz9gKNyK89dVj0F1jTdSOOQs6V7jebQpwauQXQkq6WUxCjp5eVophDew8Im%2Bi%2Bh1Abn007YWU3kYP8H0Y8VAEOIq9mCGozvmsVk0JpsRQDBuxxDj9HtPkGDCeRX%2FjXLroNZY7d0PiFVF4YRItRyKtxL5NlRTW%2BUg2azO9aZgB4035tsprfY5FG5J07DeFoYXmOHHbeNIKjmq7bQSn9B1ZJXWH%2B6nvKTfutlInscQMq6usXOTNFTYW5kKTiQITa0OqGyzYgKwtalCI2&acctmode=0&pass_ticket=%2BheWQ8duYrNY1ThuaO7lLUMoYbdW1Elm9lVQlYdUhsSiPER8TfEzdYBx3OVeaBWK&wx_header=1&fasttmpl_type=0&fasttmpl_fullversion=7350504-zh_CN-zip&fasttmpl_flag=1)喜欢作者

阅读 86

修改于2024年03月30日

​

发消息

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/Dz7JZlP0rjyQkC3MaOvFTPggGQhbvqy1ibY6AtwjYXC4w9kBXBXCouwL0q7W9TlzsZ9giarG2VId58QzffAeF3JQ/300?wx_fmt=png&wxfrom=18)

CodeTrap

117在看

发消息
