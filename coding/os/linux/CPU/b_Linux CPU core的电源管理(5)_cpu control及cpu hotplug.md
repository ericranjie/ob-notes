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

作者：[wowo](http://www.wowotech.net/author/2 "runangaozhong@163.com") 发布于：2015-9-19 21:39 分类：[电源管理子系统](http://www.wowotech.net/sort/pm_subsystem)

#### 1. 前言

由“[Linux CPU core的电源管理(1)\_概述](http://www.wowotech.net/pm_subsystem/cpu_core_pm_overview.html)”的描述可知，kernel cpu control位于“.\\kernel\\cpu.c”中，是一个承上启下的模块，负责屏蔽arch-dependent的实现细节，向上层软件提供控制CPU core的统一API（主要包括cpu_up/cpu_down等接口的实现）。本文将基于这些API，从上到下，分析CPU core从启动到关闭的整个过程（主要是CPU hotplug），进一步理解系统运行过程中CPU core电源管理相关的行为。

注1：其实这一部分已经不属于电源管理的范畴了，而是系统级的软件行为（boot、调度、电源管理等等），之所以放到这里讲述，主要原因是，这些复杂行为的背后，目的只有一个----节电。因此，本文只会focus在CPU core power状态切换的过程上，涉及到得其它知识，如进程调度，只会一笔带过。

#### 2. possible/present/active/online cpus

前面文章提到过，kernel使用4个bitmap，来保存分别处于4种状态的CPU core：possible、present、active和online。这四个状态的意义到底是什么？下面我们根据相关的代码逻辑，来解答这个问题。

开始之前，先看一下kernel中对他们的注释：

1: /\* include/linux/cpumask.h \*/

2:

3:

4: /\*

5:  * The following particular system cpumasks and operations manage

6:  * possible, present, active and online cpus.

7:  \*

8:  *     cpu_possible_mask- has bit 'cpu' set iff cpu is populatable

9:  *     cpu_present_mask - has bit 'cpu' set iff cpu is populated

10:  *     cpu_online_mask  - has bit 'cpu' set iff cpu available to scheduler

11:  *     cpu_active_mask  - has bit 'cpu' set iff cpu available to migration

12:  \*

13:  *  If !CONFIG_HOTPLUG_CPU, present == possible, and active == online.

14:  \*

15:  *  The cpu_possible_mask is fixed at boot time, as the set of CPU id's

16:  *  that it is possible might ever be plugged in at anytime during the

17:  *  life of that system boot.  The cpu_present_mask is dynamic(\*),

18:  *  representing which CPUs are currently plugged in.  And

19:  *  cpu_online_mask is the dynamic subset of cpu_present_mask,

20:  *  indicating those CPUs available for scheduling.

21:  \*

22:  *  If HOTPLUG is enabled, then cpu_possible_mask is forced to have

23:  *  all NR_CPUS bits set, otherwise it is just the set of CPUs that

24:  *  ACPI reports present at boot.

25:  \*

26:  *  If HOTPLUG is enabled, then cpu_present_mask varies dynamically,

27:  *  depending on what ACPI reports as currently plugged in, otherwise

28:  *  cpu_present_mask is just a copy of cpu_possible_mask.

29:  \*/

30:

> 大意是这样的：
>
> possible状态的CPU意味着是“populatable（觉得这个单词还没有possible易懂）”的，可理解为存在这个CPU资源，但还没有纳入Kernel的管理范围；
>
> present状态的CPU，是已经“populated”的CPU，可理解为已经被kernel接管；
>
> online状态的CPU，表示可以被调度器使用；
>
> active状态的CPU，表示可以被migrate（什么意思？）；
>
> 如果系统没有使能CPU Hotplug功能，则present等于possible，active等于online。
>
> 还真不是很容易理解，不急，我们一个一个分析。

##### 2.1 possible CPU

possible的CPUs，代表了系统中可被使用的所有的CPU，在boot阶段确定之后，就不会再修改。以ARM64为例，其初始化的过程如下。

1）系统上电后，boot CPU启动，执行start_kernel（init/main.c），并分别调用boot_cpu_init和setup_arch两个接口，进行possible CPU相关的初始化。

2）boot_cpu_init负责将当前的boot CPU放到possible CPU的bitmap中，同理，boot CPU也是present、oneline、active CPU（因此，后续的描述，都是针对非boot CPU的）。如下：

1: /\* init/main.c \*/

2:

3: static void \_\_init boot_cpu_init(void)

4: {

5:         int cpu = smp_processor_id();

6:         /\* Mark the boot cpu "present", "online" etc for SMP and UP case \*/

7:         set_cpu_online(cpu, true);

8:         set_cpu_active(cpu, true);

9:         set_cpu_present(cpu, true);

10:         set_cpu_possible(cpu, true);

11: }

> smp_processor_id用于获取当前的CPU id；
>
> set_cpu_xxx接口，可以将指定的CPU设置为（或者清除）指定的状态。

3）setup_arch负责根据MPIDR寄存器，以及DTS配置，解析并设置其它的possible CPU，如下：

1: /\* arch/arm64/kernel/setup.c \*/

2:

3: void \_\_init setup_arch(char \*\*cmdline_p)

4: {

5:         ...

6:         cpu_logical_map(0) = read_cpuid_mpidr() & MPIDR_HWID_BITMASK;

7:         cpu_read_bootcpu_ops();

8: #ifdef CONFIG_SMP

9:         smp_init_cpus();

10:         smp_build_mpidr_hash();

11: #endif

12:         ...

13: }

14:

3a）cpu_logical_map数组

kernel使用一个整形数组（cpu_logical_map，定义如下），保存物理CPU（由ID标示）和逻辑CPU（数组的index）之间的映射，该数组的长度由NR_CPUS决定。

1: /\* arch/arm64/include/asm/smp_plat.h \*/

2:

3: /\*

4:  * Logical CPU mapping.

5:  \*/

6: extern u64 \_\_cpu_logical_map\[NR_CPUS\];

7: #define cpu_logical_map(cpu)    \_\_cpu_logical_map\[cpu\]

上面setup_arch代码的第六行，通过read_cpuid_mpidr接口，读取当前CPU（boot CPU）的ID（物理ID），并保存在map表的第一个位置。

3b）smp_init_cpus

如果使能了SMP，则调用smp_init_cpus接口，完成如下事情：

从DTS中解析其它CPU的HW ID（通过‘reg’关键字，如下），并保存在cpu_logical_map数组中；

对所有cpu_logical_map数组中的CPU，执行set_cpu_possible操作，将它们设置为possible状态。

1: {

2:        ...

3:        cpus {

4:                 #address-cells = \<2>;

5:                 #size-cells = \<0>;

6:

7:                 cpu@0 {

8:                         device_type = "cpu";

9:                         compatible = "arm,cortex-a53", "arm,armv8";

10:                         reg = \<0x0 0x0>;

11:                         enable-method = "psci";

12:                         cpu-idle-states = \<&CPU_SLEEP_0 &CPU_SLEEP_1>;

13:                 };

14:

15:                 cpu@1 {

16:                         device_type = "cpu";

17:                         compatible = "arm,cortex-a53", "arm,armv8";

18:                         reg = \<0x0 0x1>;

19:                         enable-method = "psci";

20:                         cpu-idle-states = \<&CPU_SLEEP_0 &CPU_SLEEP_1>;

21:                 };

22:                 ...

23:        };

24:        ...

25: }

26:

> CPU DTS文件示例。

4）总结

> 对ARM64来说，possible的CPU，就是在DTS中指定了的，物理存在的CPU core。

##### 2.2 present CPU

还是以ARM64为例，“start_kernel—>setup_arch”成功执行之后，继续执行“start_kernel—>rest_init—>kernel_init（pid 1，init task）—>kernel_init_freeable”，在kernel_init_freeable中会调用arch-dependent的接口：smp_prepare_cpus，该接口主要的主要功能有两个：

1）构建系统中CPU的拓扑结构，具体可参考“[Linux CPU core的电源管理(2)\_cpu topology](http://www.wowotech.net/pm_subsystem/cpu_topology.html)”。

2）拓扑结构构建完成后，根据CPU的拓扑，初始化系统的present CPU mask，代码如下：

1: void \_\_init smp_prepare_cpus(unsigned int max_cpus)

2: {

3:         ...

4:         /\* Don't bother if we're effectively UP \*/

5:         if (max_cpus \<= 1)

6:                 return;

7:

8:         /\*

9:          * Initialise the present map (which describes the set of CPUs

10:          * actually populated at the present time) and release the

11:          * secondaries from the bootloader.

12:          \*

13:          * Make sure we online at most (max_cpus - 1) additional CPUs.

14:          \*/

15:         max_cpus--;

16:         for_each_possible_cpu(cpu) {

17:                 if (max_cpus == 0)

18:                         break;

19:

20:                 if (cpu == smp_processor_id())

21:                         continue;

22:

23:                 if (!cpu_ops\[cpu\])

24:                         continue;

25:

26:                 err = cpu_ops\[cpu\]->cpu_prepare(cpu);

27:                 if (err)

28:                         continue;

29:

30:                 set_cpu_present(cpu, true);

31:                 max_cpus--;

32:         }

33: }

> 4~6行：当然，如果CPU个数不大于1，则不是SMP系统，就没有后续的概念，直接返回。
>
> 16~32行，轮询所有的possible CPU，如果某个CPU core满足一些条件，则调用set_cpu_present，将其设置为present CPU，满足的条件包括：具备相应的cpu_ops指针（有关cpu ops请参考“[Linux CPU core的电源管理(3)\_cpu ops](http://www.wowotech.net/pm_subsystem/cpu_ops.html)”）；cpu ops的.cpu_prepare回调成功执行。

由“[Linux CPU core的电源管理(3)\_cpu ops](http://www.wowotech.net/pm_subsystem/cpu_ops.html)”中有关CPU ops的解释可知，.cpu_prepare回调主要用于检查某个CPU是否具备执行的条件。如果.cpu_prepare执行成功，则说明该CPU是可以启动的。因此，present CPU的意义是：

> 该CPU已经被kernel识别到，并且具备执行代码的条件，后续可以在需要的时候（如hotpulg的时候），启动该CPU。

##### 2.3 online CPU

由前面present CPU可知，如果某个CPU是present的，则说明该CPU具备boot的条件，但是否已经boot还是未知数。

由“[Linux CPU core的电源管理(3)\_cpu ops](http://www.wowotech.net/pm_subsystem/cpu_ops.html)”的介绍可知，所谓的CPU boot，就是让CPU执行（取指译码/执行）代码（这里为linux kernel）。而CPU是否boot，则反映到online mask上，已经boot的CPU，会在secondary_start_kernel中，调用set_cpu_online接口，将其设置为online状态。反之，会在\_\_cpu_disable中将其从online mask中清除。

有关CPU boot的流程，请参考下面的介绍。

##### 2.4 active CPU

在单核时代，调度器（scheduler）的职责很单纯：主要负责管理、调教一帮调皮捣蛋的task，尽量以“公平公正”的原则，为它们分配有限的CPU资源。

但在SMP系统中，特别是支持CPU hotplug的系统中，调度器需要多操一份心，即：

> CPU资源可以在任何时候增加或者删除。增加的时候，需要将新增的资源分配给等待的task。删除的时候，需要将那些运行在这些CPU上的task，转移到其它尚存的CPU上（这个过程称作migration）。

要达到上面的目的，调度器需要监视CPU hotplug有关的每一个风吹草动。由于调度器和[CPU控制](http://www.wowotech.net/pm_subsystem/cpu_ops.html)两个独立的模块，kernel通过notifier机制（“[Linux CPU core的电源管理(1)\_概述](http://www.wowotech.net/pm_subsystem/cpu_core_pm_overview.html)”中有提及，但没有过多介绍）实现这一功能。

简言之，每当系统的CPU资源有任何变动，[kernel CPU control](http://www.wowotech.net/pm_subsystem/cpu_ops.html)模块就会通知调度器，调度器根据相应的event（CPU_DOWN_FAILED、CPU_DOWN_PREPARE等），调用set_cpu_active接口，将某个CPU添加到active mask或者移出active mask。这就是active CPU的意义：

> 从调度器的角度，CPU的状态，即是否对调度器可见，或者说，调度器是否可以把task分配到这个CPU上运行。

注2：由此可知，active状态，只是为了方便调度器操作，抽象出的状态，和CPU电源管理之间没有耦合，后面就不在涉及这部分内容。

#### 3. CPU的控制流程

CPU的控制流程，可以总结为up和down两种行为（和“.\\kernel\\cpu.c”中的cpu_up、cpu_down两个接口对应），up指CPU的启动过程，down指相反地过程。

根据CPU的发展过程，up和down的行为又可以分为三类：单核CPU的up/down；多核CPU的up/down；hotplugable CPU的up/down。下面让我们对这几种情况做一下简单的介绍。

##### 3.1 单核CPU的控制流程

单核时代，只有一个CPU core，因此CPU的up/down，就是软件的整个生命周期（也就无所谓up/down了），如下：

> 1）系统上电，CPU从ROM代码执行，经bootloader（非必须），将控制权交给linux kernel。这就是cpu up的过程。
>
> 2）系统运行（一大堆省略号）。
>
> 3）由linux kernel及其进程调度算法所决定，不允许系统在没有CPU资源的情况下运行（这也是boot CPU的由来），所以系统的整个运行过程中，CPU都是up状态。
>
> 4）系统关闭，cpu down。

##### 3.2 多核CPU的控制流程

linux kernel对待SMP系统的基本策略是：指定一个boot CPU，完成系统的初始化，然后再启动其它CPU。过程如下：

> 1）boot CPU启动，其up/down的控制流程和生命周期，和单核CPU一样。
>
> 2）boot CPU启动的过程中，调用cpu_up接口，启动其它CPU（称作secondary CPUs），使它们变成online状态（具体可参考“[Linux CPU core的电源管理(3)\_cpu ops](http://www.wowotech.net/pm_subsystem/cpu_ops.html)”）。这就是secondary CPUs的up过程。
>
> 3）由于CPU不支持hotplug功能，因此所有CPU只能up，不能down。直到系统关闭，才是cpu down。

#### 3.3 CPU hotplug的控制流程

对于支持CPU hotplug功能的平台来说，可以在系统启动后的任意时刻，关闭任意一个secondary CPU（对ARM平台来说，CPU0或者说boot CPU，是不可以被关闭的），并在需要的时候，再次打开它。因此，相应的CPU控制流程如下：

> 1）boot CPU启动，其up/down的控制流程和生命周期，和单核CPU一样。
>
> 2）boot CPU启动的过程中，调用cpu_up接口，启动secondary CPU，使它们变成online状态，这是secondary CPUs的up过程的一种。
>
> 3）在系统负荷较低、或者不需要使用的时候，调用cpu_down接口，关闭不需要使用的secondary CPU，这是secondary CPUs的down过程。
>
> 4）在需要的时候，再次调用cpu_up接口，启动处于down状态的CPU，这是secondary CPUs的up过程的另一种。

有关CPU hotplug的具体说明，可参考后面描述。

#### 4. CPU hotplug

##### 4.1 CPU hotplug的时机

在kernel/cpu.c中，cpu_up接口，只会在使能了CONFIG_SMP配置项（意味着是SMP系统）后才会提供。而cpu_down接口，则只会在使能了CONFIG_HOTPLUG_CPU配置项（意味着支持CPU hotplug）后才会提供。

在当前kernel实现中，只支持通过sysfs的形式，关闭或打开CPU（当然，如果需要可以自定义一些方法，实现动态开关核的功能，本文就不在描述了），例如：

> echo 0 > /sys/devices/system/cpu/cpuX/online      # 关闭CPU
>
> echo 1 > /sys/devices/system/cpu/cpuX/online      # 打开CPU

另外，CPU hotplug还受“maxcpus”命令行参数影响：

系统启动的时候，可以通过命令行参数“maxcpus”，告知kernel本次启动所使用的CPU个数，该个数可以小于等于possible CPU的个数。系统初始化时，只会把“maxcpus”所指定个数的CPU置为present状态，具体可参考上面2.2小节所描述的smp_prepare_cpus的代码实现。

因此，CPU hotplug只能管理“maxcpus”所指定个数的CPU，具体可参考后面_cpu_up的流程分析。

注3：蜗蜗对这部分的理解，和“Documentation\\cpu-hotplug.txt”中的描述有出入，文档是这样描述的：

> maxcpus=n    Restrict boot time cpus to n. Say if you have 4 cpus, using\
> maxcpus=2 will only boot 2. You can choose to bring the\
> other cpus later online, read FAQ's for more info.

它说其它CPU可以在后边被online，但从代码逻辑来说，没有机会online啊！先存疑吧！！

##### 4.2 CPU hotplug的过程分析

CPU online的软件流程如下：

> echo 0 > /sys/devices/system/cpu/cpuX/online\
> online_store(drivers/base/core.c)\
> device_online(drivers/base/core.c)\
> cpu_subsys_online(drivers/base/cpu.c)\
> cpu_up(kernel/cpu.c)\
> \_cpu_up(kernel/cpu.c)

CPU offline的流程和online类似，不再详细介绍。这两个操作，最终是由cpu_up/cpu_down（也即_cpu_up/\_cpu_down）两个接口实现的，下面我们重点分析这两个接口。

注4：内核中经常有这样的函数，xxx、\_xxx或者\_\_xxx，区别是一个或者两个下划线，其中的含义是：

> xxx接口，通常需要由某个锁保护，一般提供给其它模块调用。它会直接调用_xxx接口；
>
> \_xxx接口，则不需要保护，一般由模块内部在确保安全的情况下调用。有时，外部模块确信可行（不需要保护），也可能会直接调用；
>
> \_\_xxx接口，一般提供给arch-dependent的软件层实现，比如这里的arch/arm64/kernel/xxx.c。
>
> 理解这些含义后，会加快我们阅读代码的速度，另外，如果直接写代码，也尽量遵守这样的原则，以便使自己的代码更规范、更通用。

##### 4.3 cpu_up流程分析

cpu_up的基本流程如下所示：

[![cpu_up_overview](http://www.wowotech.net/content/uploadfile/201509/4884e749f6105f737f304ecc0885b9e520150919133910.gif "cpu_up_overview")](http://www.wowotech.net/content/uploadfile/201509/675b05f65d1bb3321e511f41f1d1b7e620150919133910.gif)

其要点包括：

> 1）up前后，发送PREPARE、ONLINE、STARTING等notify，以便让关心者作出相应的动作，例如调度器、RCU、workqueue等模块，都需要关注CPU的hotplug动作，以便进行任务的重新分配等操作。
>
> 2）执行Arch-specific相关的boot操作，将CPU boot起来，最终通过secondary_start_kernel接口，停留在per-CPU的idle线程上。

下面我们结合代码，对上述过程做一个简单的分析。

###### 4.3.1 per-CPU的idle线程

我们在“[linux cpuidle framework](http://www.wowotech.net/tag/cpuidle)”的系列文章中，已经分析过linux cpuidle有关的工作原理，但却没有提及cpuidle的源头，这里我们补充回来。

首先，boot CPU在执行初始化动作的时候，会通过“smp_init—>idle_threads_init—>idle_init”的调用，为每个CPU创建一个idle线程，如下：

1: /\* kernel/smpboot.c \*/

2: static inline void idle_init(unsigned int cpu)

3: {

4:         struct task_struct \*tsk = per_cpu(idle_threads, cpu);

5:

6:         if (!tsk) {

7:                 tsk = fork_idle(cpu);

8:                 if (IS_ERR(tsk))

9:                         pr_err("SMP: fork_idle() failed for CPU %u\\n", cpu);

10:                 else

11:                         per_cpu(idle_threads, cpu) = tsk;

12:         }

13: }

> 该接口的本质是，为每个CPU fork一个idle thread（由struct task_struct结构表示），并保存在一个per-CPU的全局变量（idle_threads）中。
>
> 此时，idle thread只是一个task结构，并没有执行。那它最终怎么执行的呢？我们继续往后面看。

###### 4.3.2 arch-specific CPU boot

\_cpu_up接口会在完成一些准备动作之后，调用平台相关的\_\_cpu_up接口，由平台代码完成具体的up操作，如下：

1: static int \_cpu_up(unsigned int cpu, int tasks_frozen)

2: {

3:         int ret, nr_calls = 0;

4:         void \*hcpu = (void \*)(long)cpu;

5:         unsigned long mod = tasks_frozen ? CPU_TASKS_FROZEN : 0;

6:         struct task_struct \*idle;

7:

8:         cpu_hotplug_begin();

9:

10:         if (cpu_online(cpu) || !cpu_present(cpu)) {

11:                 ret = -EINVAL;

12:                 goto out;

13:         }

14:

15:         idle = idle_thread_get(cpu);

16:         if (IS_ERR(idle)) {

17:                 ret = PTR_ERR(idle);

18:                 goto out;

19:         }

20:

21:         ret = smpboot_create_threads(cpu);

22:         if (ret)

23:                 goto out;

24:

25:         ret = \_\_cpu_notify(CPU_UP_PREPARE | mod, hcpu, -1, &nr_calls);

26:         if (ret) {

27:                 nr_calls--;

28:                 pr_warn("%s: attempt to bring up CPU %u failed\\n",

29:                         __func__, cpu);

30:                 goto out_notify;

31:         }

32:

33:         /\* Arch-specific enabling code. \*/

34:         ret = \_\_cpu_up(cpu, idle);

35:         if (ret != 0)

36:                 goto out_notify;

37:         BUG_ON(!cpu_online(cpu));

38:

39:         /\* Wake the per cpu threads \*/

40:         smpboot_unpark_threads(cpu);

41:

42:         /\* Now call notifier in preparation. \*/

43:         cpu_notify(CPU_ONLINE | mod, hcpu);

44:

45: out_notify:

46:         if (ret != 0)

47:                 \_\_cpu_notify(CPU_UP_CANCELED | mod, hcpu, nr_calls, NULL);

48: out:

49:         cpu_hotplug_done();

50:

51:         return ret;

52: }

> 准备动作包括：
>
> 1）获取idle thread的task指针，该指针最终会以参数的形式传递给arch-specific代码。
>
> 2）创建一个用于管理CPU hotplug动作的线程（smpboot_create_threads），该线程的具体意义，后面会再说明。
>
> 3）发送CPU_UP_PREPARE notify。

以ARM64为例，\_\_cpu_up的内部实现如下：

1: /\* arch/arm64/kernel/smp.c \*/

2: int \_\_cpu_up(unsigned int cpu, struct task_struct \*idle)

3: {

4:         int ret;

5:

6:         /\*

7:          * We need to tell the secondary core where to find its stack and the

8:          * page tables.

9:          \*/

10:         secondary_data.stack = task_stack_page(idle) + THREAD_START_SP;

11:         \_\_flush_dcache_area(&secondary_data, sizeof(secondary_data));

12:

13:         /\*

14:          * Now bring the CPU into our world.

15:          \*/

16:         ret = boot_secondary(cpu, idle);

17:         if (ret == 0) {

18:                 /\*

19:                  * CPU was successfully started, wait for it to come online or

20:                  * time out.

21:                  \*/

22:                 wait_for_completion_timeout(&cpu_running,

23:                                             msecs_to_jiffies(1000));

24:

25:                 if (!cpu_online(cpu)) {

26:                         pr_crit("CPU%u: failed to come online\\n", cpu);

27:                         ret = -EIO;

28:                 }

29:         } else {

30:                 pr_err("CPU%u: failed to boot: %d\\n", cpu, ret);

31:         }

32:

33:         secondary_data.stack = NULL;

34:

35:         return ret;

36: }

> 该接口以idle thread的task指针为参数，完成如下动作：
>
> 1）将idle线程的堆栈，保存在一个名称为secondary_data的全局变量中（这地方很重要，后面再介绍其中的奥妙）。
>
> 2）执行boot_secondary接口，boot CPU，具体的流程，可参考“[Linux CPU core的电源管理(3)\_cpu ops](http://www.wowotech.net/pm_subsystem/cpu_ops.html)”中的描述。
>
> 3）boot_secondary返回后，等待对应的CPU切换为online状态。

###### 4.3.3 secondary_startup

“[Linux CPU core的电源管理(3)\_cpu ops](http://www.wowotech.net/pm_subsystem/cpu_ops.html)” 4.1小节，分析了使用SPIN TABLE cpu ops的情况下，boot_secondary到secondary_startup的流程（其它cpu ops类似），本文继续secondary_startup的分析。

该接口位于arch/arm64/kernel/head.S中，负责secondary CPU启动后的后期操作，如下：

1: ENTRY(secondary_startup)

2:         /\*

3:          * Common entry point for secondary CPUs.

4:          \*/

5:         mrs     x22, midr_el1                   // x22=cpuid

6:         mov     x0, x22

7:         bl      lookup_processor_type

8:         mov     x23, x0                         // x23=current cpu_table

9:         cbz     x23, \_\_error_p                  // invalid processor (x23=0)?

10:

11:         pgtbl   x25, x26, x28                   // x25=TTBR0, x26=TTBR1

12:         ldr     x12, \[x23, #CPU_INFO_SETUP\]

13:         add     x12, x12, x28                   // \_\_virt_to_phys

14:         blr     x12                             // initialise processor

15:

16:         ldr     x21, =secondary_data

17:         ldr     x27, =\_\_secondary_switched      // address to jump to after enabling the MMU

18:         b       \_\_enable_mmu

19: ENDPROC(secondary_startup)

20:

21: ENTRY(\_\_secondary_switched)

22:         ldr     x0, \[x21\]                       // get secondary_data.stack

23:         mov     sp, x0

24:         mov     x29, #0

25:         b       secondary_start_kernel

26: ENDPROC(\_\_secondary_switched)

> 我们重点关注上面16~17行，以及21~26行的\_\_secondary_switched，\_\_secondary_switched会将保存在secondary_data全局变量中的堆栈取出，保存在该CPU的SP中，并跳转到secondary_start_kernel继续执行。思考一下其中的意义：

我们都知道，CPU启动后，需要先配置好堆栈，才能进行后续的函数调用，这里使用的是该CPU idle thread的堆栈。就这么简单吗？当然不是，看一下kernel中“current”指针（获取当前task结构的宏定义）的实现方法：

1: #define current get_current()

2: #define get_current() (current_thread_info()->task)

3:

4: static inline struct thread_info \*current_thread_info(void)

5: {

6:         register unsigned long sp asm ("sp");

7:         return (struct thread_info \*)(sp & ~(THREAD_SIZE - 1));

8: }

9:

> 有没有豁然开朗的感觉？通过CPU的SP指针，是可以获得CPU的当前task的（这和linux kernel进程管理的实现有关，我们不深究）。也就是说，当CPU SP被赋值为idle thread的堆栈的那一瞬间，当前的上下文已经是idle thread了！！

至于后面的secondary_start_kernel，就比较简单了，使能GIC、Timers，设置CPU为online状态，使能本地IRQ中断。等等。最后，调用cpu_startup_entry，进入cpuidle的loop中，已经和“[Linux \_cpu_idle framework(1)\_概述和软件架构](http://www.wowotech.net/pm_subsystem/cpuidle_overview.html)”中描述接上了，自此，CPU UP完成。

##### 4.4 cpu_down流程

cpu_down是cpu_up的反过程，用于将某个CPU从系统中移出。从表面上看，它和cpu_up的过程应该类似，但实际上它的处理过程却异常繁琐、复杂，同时牵涉到非常多的进程调度的知识，鉴于篇幅，本文就不再继续分析了。如果有机会，后面再专门用一篇文章分析这个过程。

另外，前面提到的smpboot有关的内容，也和cpu_down的过程有关，也就不再介绍了。

#### 5. 小结

由本文的分析可知，cpu control有关的过程，其本身的逻辑比较简单，复杂之处在于：与此相关的系统服务（任务、中断、timer等等）的迁移。如果要理解这个过程，就必须有深厚的进程调度、中断管理等背景知识作支撑。不着急，来日方长，有机会我们继续分析。

_原创文章，转发请注明出处。蜗窝科技_，[www.wowotech.net](http://www.wowotech.net/pm_subsystem/cpu_hotplug.html)。

标签: [Linux](http://www.wowotech.net/tag/Linux) [cpu](http://www.wowotech.net/tag/cpu) [hotplug](http://www.wowotech.net/tag/hotplug) [cpu_up](http://www.wowotech.net/tag/cpu_up) [cpu_down](http://www.wowotech.net/tag/cpu_down) [smpboot](http://www.wowotech.net/tag/smpboot)

[![](http://www.wowotech.net/content/uploadfile/201605/ef3e1463542768.png)](http://www.wowotech.net/support_us.html)

« [Linux vm运行参数之（一）：overcommit相关的参数](http://www.wowotech.net/memory_management/overcommit.html) | [Linux内核同步机制之（六）：Seqlock](http://www.wowotech.net/kernel_synchronization/seqlock.html)»

**评论：**

**rikeyone**\
2017-04-10 16:23

hi 窝窝：\
请问一下，本文介绍说cpu hotplug内核只是提供接口，那么应用要怎么决定是否要enable/disable cpu呢？\
目前linux/android的完整解决方案是怎样的呢？\
麻烦大神解答一下

[回复](http://www.wowotech.net/pm_subsystem/cpu_hotplug.html#comment-5445)

**zd**\
2017-01-03 09:51

Hi, Wowo.看了你写的cpu hotplug和cpufreq，想请教一个问题，cpuhotplug和cpufreq实现CPU调节过程中，哪个机制先调节？

[回复](http://www.wowotech.net/pm_subsystem/cpu_hotplug.html#comment-5075)

**[wowo](http://www.wowotech.net/)**\
2017-01-03 13:19

@zd：这应该是应用程序决定的，kernel只提供剑，至于杀谁，是人决定的。

[回复](http://www.wowotech.net/pm_subsystem/cpu_hotplug.html#comment-5078)

**[wowo](http://www.wowotech.net/)**\
2016-04-05 19:55

@cym,没有研究过ESA

[回复](http://www.wowotech.net/pm_subsystem/cpu_hotplug.html#comment-3788)

**cym**\
2016-03-30 19:41

hi wowo，为什么有些时候开了hotplug会比关闭还耗电呢？按照常理来说，两个核各为20%的loading应该比单核40%耗电吧？

[回复](http://www.wowotech.net/pm_subsystem/cpu_hotplug.html#comment-3753)

**[wowo](http://www.wowotech.net/)**\
2016-03-30 22:13

@cym："开了hotplug"具体指什么意思呢？动态开关核？还是在两个核都是20%的时候把其中的一个关闭？\
有没有使用动态调频功能？20%和40%的时候，CPU的频率和电压是一样的吗？\
这里面变数比较多，还真不好下结论。

[回复](http://www.wowotech.net/pm_subsystem/cpu_hotplug.html#comment-3756)

**cym**\
2016-03-30 22:59

@wowo：1.开了动态开关核，开了dvfs，其他条件都是一样的，对比开与没开动态开关核的功耗，发现没开的功耗反而低。\
2.这是由此产生的疑问：有两种情况（都是关了动态开关核），一种是单核2G（固定频率）运行游戏，处于40%的loading；另一种是双核1G（固定频率），运行相同的游戏，两个cpu有可能处于20%（加起来应该是40%）。困惑的是，这两种情况下，哪种耗电更多呢？

[回复](http://www.wowotech.net/pm_subsystem/cpu_hotplug.html#comment-3760)

**[wowo](http://www.wowotech.net/)**\
2016-03-31 08:58

@cym：个人观点，条件一样的情况下，应该是两个核更耗电。\
我们可以从能量的角度去解释：\
假设当前的事情需要消耗的能量是E（比如说游戏），刨除开销，无论用多少核，消耗的能力都是E（就是我们说的功耗）。也就是说，在理想情况下，40% loading的1核，和20% loading的2核，耗电应该相等。\
但这仅仅是理想情况，另外60%（或者80%）的idle时间，其实就是开销，这种情况下，2核肯定比1核的开销大。因此从省电的角度看，把一个core的loading提到100%之后，再开另一个core，是最划算的。

[回复](http://www.wowotech.net/pm_subsystem/cpu_hotplug.html#comment-3761)

**cym**\
2016-03-31 15:45

@wowo：谢谢wowo，茅塞顿开。

[回复](http://www.wowotech.net/pm_subsystem/cpu_hotplug.html#comment-3765)

**cym**\
2016-04-01 14:34

@cym：hi wowo,那为什么我们现在的策略不是照你说的那样去实现呢？是不是对性能会有影响？

[回复](http://www.wowotech.net/pm_subsystem/cpu_hotplug.html#comment-3766)

**cym**\
2016-04-05 10:25

@wowo：hi wowo,那为什么我们现在的策略不是照你说的那样去实现呢？是不是对性能会有影响？

[回复](http://www.wowotech.net/pm_subsystem/cpu_hotplug.html#comment-3780)

**[wowo](http://www.wowotech.net/)**\
2016-04-05 11:20

@cym：理想和现实之间，总会有差距吧，具体原因，只能具体情况具体对待了。何况，系统设计的时候，功耗只是考虑之一，稳定性和性能，也不得不考虑。

[回复](http://www.wowotech.net/pm_subsystem/cpu_hotplug.html#comment-3783)

**cym**\
2016-04-05 15:17

@wowo：wowo ，你有研究过eas吗

**[archer](http://www.wowotech.net/)**\
2016-01-28 11:21

写的太好了，谢谢！请教一个问题，possible cpus是boot时就确定了的，有没有什么机制可以动态插入一个cpu吗？这样的话是不是possible cpus也可以变化？

[回复](http://www.wowotech.net/pm_subsystem/cpu_hotplug.html#comment-3453)

**[wowo](http://www.wowotech.net/)**\
2016-01-28 11:36

@archer：可以参考“4.2 CPU hotplug的过程分析”，hotplug会改变online状态，但possible需要事先配置好。

[回复](http://www.wowotech.net/pm_subsystem/cpu_hotplug.html#comment-3454)

**[tigger](http://www.wowotech.net/)**\
2015-09-23 13:51

Hi WOWO\
为什么通过如下的操作可以得到cluster id，\
unsigned int cluster = (mpidr >> 8) & 0xf;

我在datasheet没找到，cluster id 存放到 Affinitty 1 中，你知道这个说明在哪里吗？\
Aff1, bits \[15:8\]\
Affinity level 1. Third highest level affinity field.

[回复](http://www.wowotech.net/pm_subsystem/cpu_hotplug.html#comment-2677)

**[wowo](http://www.wowotech.net/)**\
2015-09-23 16:35

@tigger：你看的datasheet是指“DDI0487A_d_armv8_arm.pdf”吗？体系结构不会规定具体的位放什么东西的，是否存放cluster ID，是由CPU决定的，例如A57，您可以参考这个“DDI0488G_cortex_a57_mpcore_trm.pdf”。

[回复](http://www.wowotech.net/pm_subsystem/cpu_hotplug.html#comment-2678)

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

  - [Windows系统结合MinGW搭建软件开发环境](http://www.wowotech.net/soft/6.html)
  - [Linux PM QoS framework(1)\_概述和软件架构](http://www.wowotech.net/pm_subsystem/pm_qos_overview.html)
  - [PELT算法浅析](http://www.wowotech.net/process_management/pelt.html)
  - [Linux电源管理(11)\_Runtime PM之功能描述](http://www.wowotech.net/pm_subsystem/rpm_overview.html)
  - [一例hardened_usercopy的bug解析](http://www.wowotech.net/linux_kenrel/480.html)

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
