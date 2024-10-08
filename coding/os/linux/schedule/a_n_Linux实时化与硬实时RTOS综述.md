原创 伟林 Linux阅码场
 _2022年02月17日 08:00_
伟林，中年码农，从事过电信、手机、安全、芯片等行业，目前依旧从事Linux方向开发工作，个人爱好Linux相关知识分享，个人微博CSDN pwl999，欢迎大家关注！ 
#  文章目录

**1. 背景介绍**
**1.1 OS 实时难题**
**1.2 Linux 实时补丁**
**1.3 Xenomai + Linux 双内核**
**1.4 HW-RTOS**
**1.5 More**
**2. 优化点1：API**
**2.1 原理介绍**
2.1.1 Software API Execution time
2.1.2 Software Influence of Queue Handling
2.1.3 HW-RTOS API Execution time
2.1.4 HW-RTOS Influence of Queue Handling
**2.2 具体实现**
**2.3 性能测试**
**3. 优化点2：Tick offloading**
**3.1 原理介绍**
3.1.1 Software Tick
3.1.2 HW-RTOS Tick
**4. 优化点3：Hardware ISR (HW ISR)**
**4.1 原理介绍**
4.1.1 Software ISR
4.1.2 Software ISR handed over to Task
4.1.3 HW-RTOS ISR handed over to Task
4.1.4 Non-OS managed Interrupt & Direct Interrupt Service
**4.2 具体实现**
**4.3 性能测试**
**4.4 相关应用**
4.4.1 Multiple interrupts using HW ISR
4.4.2 Cyclic activation task
4.4.3 Using cyclic activation task for network synchronization
**5. 优化点4：Task Management**
**5.1 原理介绍**
5.1.1 HW-RTOS Scheduler
5.1.1 HW-RTOS Context Switch
**6. 典型案例**
**6.1 Network and RTOS**
**7. 下一代 HW-RTOS**
## 1.背景介绍

在工业控制领域 实时(Real Time) 是一个核心要求。

  

- 实时系统的定义：实时系统是指计算的正确性不仅依赖于逻辑的正确性而且依赖于产生结果的时间，如果系统的时间限制不能得到满足，系统将会产生故障。在工业领域这种故障可能造成灾难性的结果。
    
- 实时操作系统：该操作系统有能力提供一个指定范围内的服务响应时间。
    
      
    

一个 OS kernel 给程序员提供了一系列的服务，例如： multitasking、 interrupt management、 inter-task communication and signaling、 resource management、 time management、 memory partition management 等等。OS 的主要工作就是资源封装和调度，它提供对CPU、Memory、Storage、Network等资源的封装和调度。

  

对于 实时(Real Time) 这个课题来说，我们重点关注其中的 CPU 资源调度。

  

**1.1 OS 实时难题**

- 1、CPU 调度模型
    

  

在CPU资源调度方面，OS 主要提供一个多任务(multitasking)的运行环境，以方便应用的开发。在开发某个应用时首先把工作拆解成多个任务(Task/Thread)，每个任务都可以简化成一个简单的无限循环：

  

```
void MyTask (void) 
```

  

对一个任务来说它好像独占CPU，但实际上是多个任务共享一个 CPU 的，由 OS Kernel 根据任务的状态和优先级来动态调度运行的：

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

可以看到OS对CPU调度提供了封装，站在OS之上可以很方便的开发、移植应用，应用只需要关注自己的 Task 循环，其他复杂操作对OS都是透明。但是另一方面，OS 的这些封装不可避免带来了开销，通常来说一个 CPU 上了 OS 以后会比裸机系统更慢，在频率较低的 MCU 系统上这些开销的占比会更为明显。

  

- 2、Event Driven
    
      
    

另一个重要的概念：任何一个系统都是事件驱动(Event Driven)的，准确来说是中断驱动的。

  

如上面代码所示，任务(Task)都是等待event，然后处理事务。任何一个任务得以运行，都是因为它收到了一个 Event，这个 Event 可能是一个中断、也可能是超时到期、还有可能是其他任务发出的IPC信号，继续追查发出IPC信号的任务最后的源头 Event 肯定是一个外部设备硬件中断 或者 是内部的 Timer中断。中断引起了 Event 传递，形成了逐个运行多个任务的链条(Chain)。一个系统内部会存在很多条这种链条。

  

- 3、Real Time
    

  

对实时(Real Time)系统来说，不仅仅要求OS能提供多任务环境，更要求任务能在极短的时间之内响应外部的中断事件。这个要求也充满了挑战，即要享受 OS 带来的好处，又要把 OS 带来的开销将到最小。

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

OS 实时模型的时序如上图所示，实时优化就是围绕这张图来展开的。其中的时间点和可能的优化方式如下：

  

|   |   |   |
|---|---|---|
|时间段|延时原因|可能的优化方式|
|Interrupt Latency|由于系统中其他人关中断造成的|减少 OS API 的执行时间<br><br>减少 ISR 的处理时间|
|ISR|Handle+signal event+schedule|-|
|ISR(handle)|-|Handle Over Task 中断<br><br>线程化|
|ISR(singal event)|signal api 动作可能随负载等多而增加|-|
|ISR (schedule)|schedule api 动作可能随负载增多而增加|增加抢占(preemption)<br><br>调度算法防止优先级翻转|
|ContextSwitch|-|目前没有太好的优化方法|

  

  

**1.2 Linux 实时补丁**

大名鼎鼎的 Linux RT-Preempt Patch 试图从以下方面改善 Linux 的实时性：

|   |   |   |
|---|---|---|
|Index|Item|Note|
|1|中断线程化|减少ISR处理时的关中断时间|
|2|高精度时钟|增加系统的定时精度，<br><br>并且在不需要的时候可以关闭tick进入NoHZ状态|
|3|实时调度算法|时间片调度，从O(1)换成CFS的O(log n)<br><br>实时调度，SCHED_FIFO算法|
|4|临界区可抢占|减少自旋锁、大内核锁、RCU锁的保护区域<br><br>尽量替换成Mutex锁|
|5|优先级继承|-|

  

  

实时补丁大大改善了 Linux 的实时性能，但还是软实时的水平。

  

**1.3** **Xenomai + Linux** **双内核**

优先处理实时任务，linux也被视为其中一个线程，本身也有调度器，但须等到没有实时任务时（空闲状态），才会执行linux thread。

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

Xenomai + Linux 双内核可以达到 RTOS 的实时水平，也有把这种称为硬实时。

  

但是 RTOS 的实时还是存在不确定性，因为OS API等临界区的关中断时间还是存在不确定性，和系统的负载相关联。这也是 HW-RTOS 的优化点。

  

**1.4 HW-RTOS**

HW-RTOS (hardware real-time operating system，硬件实时操作系统)是一种基于硬件实现的实时操作系统，是瑞萨电子的专有技术。HW-RTOS支持大约30个api，都是通过硬件实现的。与传统的软件RTOS相比，硬件实现提供了极高水平的实时性能。

  

可以把HW-RTOS看作是CPU的硬件浮点单元(FPU)，加速内核常用操作的专用硬件。然而，HW-RTOS并不执行基于软件的内核中通常可用的所有操作。这类似于浮点处理器，它只提供基本的浮点运算，如加、减、乘、除、从浮点数到整型和从整型到浮点数的运算，省略了更复杂的运算，如超越运算、对数运算和其他运算。在使用HW-RTOS作为骨干时，更复杂的内核操作通常可以在软件中实现。

  

HW-RTOS 主要利用了以下手段：

  

- HW-RTOS 通过并行处理的方式，把 OS 调度类的负载从 CPU 上 Offload。
    
- 当上述的调度负载由软件来完成时，受当前资源多少的影响执行时间会变得不确定(例如：往一个排序链表中插入一个成员，消耗时间是不确定的)。但是 HW-RTOS 内部的硬件处理也是并行的，它不受当前资源多少的影响，所以它处理负载的时间可以做到接近恒定。
    

  

HW-RTOS实现了以下目标：

  

```
Fast API execution with little fluctuation
```

  

  

从支持的设备来说，瑞萨的 RZ 系列 和 R-IN32M3 系列支持了 HW-RTOS 功能：

|   |   |
|---|---|
|Product|Description|
|RZ/N1D|Cortex®-A7(Dual) + Cortex®-M3 4p GbE Switch +1 MAC EtherCAT Slave Controller Sercos|
|RZ/N1S|Cortex®-A7 + Cortex®-M3 4p GbE Switch +1 MAC EtherCAT Slave Controller Sercos|
|RZ/N1L|Cortex®-M3 4p GbE Switch +1 MAC EtherCAT Slave Controller Sercos|
|RZ/T1|Cortex®-R4 Processor with FPU + Cortex®-M3 2p Ether Switch + 1 MAC EtherCAT Slave Controller* (*Option)|
|R-IN32M3-EC|Cortex®-M3 2p Ether Switch On chip PHY EtherCAT Slave Controller|
|R-IN32M3-CL|Cortex®-M3 2p GbE Switch CC-Link IE Field Controller|
|R-IN32M3-CL2|Cortex®-M4 Processor with FPU 2p GbE Switch On chip GbE PHY CC-Link IE Field Controller|

  

从支持的 OS 来说，目前 uITRON 和 μC/OS‑III 推出了支持 HW-RTOS 的版本，但是目前找不到具体的实例代码。μC/OS‑III HW-RTOS 号称做到和原版 μC/OS‑III 的接口和体验一致：  

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

接下来的章节，我们来详细的分析 HW-RTOS 实现的相关优化点。

  

**1.5 More**

需要特别注意的是，就算实现了 HW-RTOS 但是还是有影响实时的硬件因素存在：Cache Miss、TLB Miss、分支预测失败、长跳转与短跳转、DVFS 等等。

  

航空航天等行业会使用专业的WCET(Worst-caseExecutionTime最坏执行时间)工具，对系统的实时性做一个全面分析。

  

## 2.优化点1：API

  

**2.1 原理介绍**

传统软件 OS API 会给系统来带开销，影响系统的性能和实时性，主要体现在两方面：

(1)、OS API 开销对实时性能的影响比大多数软件工程师认为的要大得多，并且发现很难保证最坏情况下的值。核心点就是 API 的函数复杂度是 O(n) 而不是 O(1)。

(2)、OS API 在执行期间为了保证一致性大部分情况下是关中断的，如果最坏情况下它的执行时间是难以预估的，那么它带来的最坏关中断时间也是不可预估的。

  

**2.1.1 Software API Execution time**

图3-1显示了日本最常用的RTOS——ITRON的API执行时间。包含“/D”的API是通过分派执行的API进程。图3-1的测量是在静态条件下进行的。

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

但是由于RTOS的执行时间取决于当前的内部状态，因此它们是动态波动的：

  

(1)、以Set Flag API执行时间为例。如图3-2所示。图中显示了所有Set Flag API的执行时间。但是，等待同一事件的任务越多，执行时间就越长。换句话说，执行时间会根据RTOS的内部状态而波动：

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

(2)、Wait队列的基本原理。如图3-3所示，RTOS的Wait队列运行在基于优先级的FCFS (First Come, First Served)算法上。具体来说，每个对象为每个任务优先级级别都有一个队列(在本例中，优先级0被指定为最高优先级，优先级n被指定为最低优先级)。例如，要实现信号量，每个信号量标识符必须有n个队列。每个希望获取信号量的任务必须在队列中等待。当一个任务释放它的信号量时，优先级最高的队列头的任务被选中，该信号量被该任务获取：

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

(3)、Set Flag API 的关键流程如下。对于标志控制，系统必须在设置后比较等待标志模式和事件表中的标志模式，以确定释放等待的条件是否满足。换句话说，RTOS必须从图3-3中队列的头部开始依次检查每个任务，并比较标志模式。结果，Set Flag系统调用的处理时间与等待相同标志的任务数量成比例显著增加，如图3-2所示。

  

因此，API执行时发生的RTOS开销会动态变化，以响应RTOS执行时的内部状态。通常，大多数设计人员希望系统调用执行在几百个时钟周期内得到结果。然而，现在很多人都没有意识到，某些条件的重叠可能会花费更多的时间。因此，当上面讨论的这些条件堆积起来时，开销会突然增加，这意味着某些过程不能在指定的时间限制内完成，从而导致实时系统的整体故障。

  

**2.1.2 Software Influence of Queue Handling**

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

如图3-3中Wait队列的逻辑结构所示，RTOS的每个对象都需要优先级队列。通常，软件中实现的队列使用列表结构。在RTOS中，TCB通过一个列表体系结构相互连接，以实现一个等待队列(TCB: Task Control Block)，图3-3中的队列实现如图3-4所示。当一个任务从Wait队列中取出时，优先级最高的队列头的任务将首先取出。因此，软件必须检查任务是否被添加到从最高优先级开始的队列中，以找到任务正在等待的最高优先级队列。作为结果，所需的处理时间会因队列条件的不同而有很大差异。

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

另一个问题出现了。在RTOS中存在大量的队列。例如，如果有64个信号量标识符、64个事件标识符和16个优先级级别，那么总数量将是64×2×16 = 2048个队列。因此，即使对象的数量相对较低，也需要大量的资源。图3-5显示了一个可能的改进。每个队列的尾部连接到下一个优先级下降的队列(即下一个较低的队列)头部的TCB。这个结构允许很容易地从Wait队列中删除任务，只需选择指针所指示的任务。它还大大减少了指针的数量。然而，如果仔细考虑，这种改进思想会导致在将任务添加到队列时进行大量处理。例如，当将优先级为7的任务添加到队列中时，搜索从队列头部开始，然后当搜索到达优先级为7的任务时，它将新任务添加到最后一个7级任务的末尾。

  

不管怎样，很明显，队列处理需要大量的处理时间，而时间将根据队列的条件而变化。此外，队列处理是一个非常重要的过程，因此在每个过程期间都禁用中断。

  

下面我们将探讨队列处理需要多少时间，并评估这对实时性能有多大影响。测试模型如图3-6 所示：

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

测试结果如下图所示：

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

伴随队列处理的API执行时间会根据RTOS中队列的状态急剧波动。此外，由此导致的中断禁用周期几乎与api执行时间一样长，因此中断禁用周期也可以根据RTOS队列内部状态动态波动。

因此，当大量任务被添加到队列中时，队列处理可能会产生意想不到的开销和意想不到的长中断禁用周期，从而可能导致实时系统中出现意想不到的错误。很有可能，应用程序和软件设计人员通常不会考虑RTOS中连接到队列的任务数量，但提前了解这些任务可能导致的问题真的很重要。

  

**2.1.3 HW-RTOS API Execution time**

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

HW-RTOS中调用api的方法如图4-1所示。如图所示，应用程序调用的API通过库软件作为硬件信号传递给HW-RTOS，返回值也通过库软件接收。库软件还根据HW-RTOS的指令执行调度过程。在图4-2中，我们可以看到HW-RTOS和传统软件RTOS(现在称为SW RTOS)执行时间测量的比较。

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

HW-RTOS中的API执行时间非常快，大多数系统调用可以在10个周期内执行。然而，库软件的开销确实会增加执行时间，如图4-2所示。在HW-RTOS中，相同的API的执行时间不会因为内部状态而改变，就像在SW RTOS中显示的设置标志一样。另一个巨大的优点是系统调用的最坏情况值可以在数据表中指定。

  

**2.1.4 HW-RTOS Influence of Queue Handling**

  

HW-RTOS使用瑞萨特有的“虚拟队列”技术实现硬件队列。Virtual Queue可以将任务添加到队列中，从队列中删除任务，也可以从队列中间删除任务，每个操作周期为2个周期。因此，无论队列的状态是什么，处理都可以在指定的时间内完成。

  

和软件方式在同样测试模型下的对比：

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

**2.2 具体实现**

  

HW-RTOS 通过硬件大概实现了 30 个 API 函数，具体包括以下几个方面：

```
Semaphore
```

  

如下图所示，HW-RTOS作为系统总线上的一个外围模块来实现。

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

HW-RTOS为API调用实现了3个专门的寄存器:

```
API register, 
```

  

瑞萨已经为处理这些寄存器准备了一个操作系统库。用户可以使用操作系统库轻松调用API调用，就像传统的软件RTOS一样。当调用set_flg API调用时，OS库将参数写入参数寄存器，并将API的类型写入API寄存器。当HW-RTOS接收到这些信息时，它执行API并将返回值写入结果寄存器。OS库将返回值报告给调用方应用程序。执行API时可能需要进行任务切换。在这种情况下，HW-RTOS表示需要进行任务切换，并将下一个应该执行的任务的ID写入结果寄存器，以将该信息传递到OS库。OS库根据此信息执行上下文切换。

  

**2.3 性能测试**

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

上图显示了API执行时间。传统软件RTOS的执行时间用深紫色表示，HW-RTOS用浅紫色表示。HW-RTOS不仅比软件RTOS的执行时间短，而且波动也不大。

  

## 3.优化点2：Tick offloading

  

**3.1 原理介绍**

  

**3.1.1 Software Tick**

传统OS需要处理一些定时任务。包括定时器、sleep()、xxx_timeout()、时间片调度等。为了这个目的，测量时间的软件处理被周期性地激活。这就是所谓的Tick进程。如图所示，为了激活tick进程，需要周期性中断。

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

如上图图所示，虽然tick是一个不可或缺的功能，但它有以下三个缺点：

(1)、应用程序会周期性的停止，CPU的使用效率会降低。

(2)、由于节拍正在执行一个非常关键的进程，因此在进程执行期间禁用所有中断。因此，这对中断响应时间有负面影响。

(3)、由于滴答过程需要通过软件来实现，所以滴答间隔不可能非常短。换句话说，高度精确的时间管理是不可能的。

  

并且 Tick 的处理时长不是固定的。Tick进程检查处于等待状态的任务是否超时，因此当大量任务同时超时时，处理需求会增加。

  

**3.1.2 HW-RTOS Tick**

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

HW-RTOS完全在硬件上实现tick过程。这个功能被称为 tick offloading。tick过程在HW-RTOS内部执行。因此，不需要周期中断滴答，也不需要CPU执行滴答过程。如图所示，CPU在任何时候都可以自由运行应用软件。只有在通过超时执行上下文切换时才会停止。此外，由于滴答过程在非常高的速度执行，滴答间隔可以缩短。基于上述原因，与传统软件相比，tick offloading可以提供以下优点。

```
No drop in CPU efficiency caused by tick process
```

  

## 4.优化点3：Hardware ISR(HW ISR)

  

**4.1 原理介绍**

**4.1.1 Software ISR**

传统OS中当一个中断发生时，一个中断服务程序(ISR)被激活。通常，在ISR执行时，中断是禁用的。在下图的上半部分，ISR1和ISR2根据中断类型交替激活。如果ISR1的处理时间延长，另一个中断将被错过或延迟，如图下方所示。对于实时系统来说，错过或延迟中断是不受欢迎的。

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

**4.1.2 Software ISR handed over to Task**

一般采用以下方法来避免此类问题。如下图所示，ISR1的处理移交给任务1,ISR2的处理移交给任务2。由于任务没有禁用中断，其他中断不会受到影响。移交处理的方法如下。任务1等待一个标志。当第一个中断(中断1)发生时，ISR1执行API来释放任务1的等待状态。这种方法最大限度地减少中断处理对其他中断的影响。例如，Linux 下的中断线程化。

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

让我们更详细地看看这种方式下发生中断时ISR是如何执行API的。下图显示了这一点。让我们假设在Task_A运行时发生了一个中断：

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

```
1.The RTOS switches CPU registers and activates the ISR.
```

  

上述过程相当复杂，通常需要500 - 1500个循环。

  

**4.1.3 HW-RTOS ISR handed over to Task**

  

上述过程如果使用HW-RTOS，图中显示的所有RTOS处理(除了dispatch处理)都是在硬件上实现的，因此处理速度非常快。HW-RTOS进一步加速了这一过程。也就是说，它加速了ISR进程。ISR简单地调用与中断源对应的API。通过将这部分实现到硬件中，可以加速它。这种实现称为硬件ISR (HW ISR)。HW ISR的时序图如下图所示。

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

```
1.An interrupt occurs and HW-RTOS commences operation. HW-RTOS activates the HW ISR.
```

  

注意，在HW- rtos和HW ISR正在处理时，CPU是空闲的，可以继续处理Task_A。CPU只在任务切换期间停止。

  

下图显示了在API执行结束时Task_B处于就绪状态的一个例子，但是当Task_B被交给ISR处理时，Task_B的优先级比Task_A的优先级低或相等。在本例中，由于Task_A具有更高的优先级，因此不需要切换任务，因此Task_A继续处理。即使中断发生，它也不会引起CPU开销：

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

通过使用HW ISR，您可以获得以下好处:

  

```
1.Greatly reduce CPU overhead during interrupts
```

  

对于希望快速执行的进程，可以简单地提高HW ISR激活的任务的优先级。当然，HW-RTOS也同时支持传统的ISR。

  

**4.1.4 Non-OS managed Interrupt & Direct Interrupt Service**

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

有操作系统管理的中断，也有操作系统外部管理的中断。RTOS、ISR和任务之间的优先级关系如图3所示。从图中可以看到，任务的优先级最低，然后是ISR，然后是RTOS。

  

由非os管理中断激活的处理程序称为非os管理中断处理程序。正如您在图3中看到的，非os管理的中断处理程序的优先级甚至高于RTOS进程。因此，即使RTOS处于进程的中间，该进程也可以被中断并立即激活处理程序。由于软件在此激活中绝对没有角色，因此直到处理程序被激活之前的延迟是由CPU定义的中断延迟。这种非os管理的中断处理程序通常用于不能允许大量RTOS开销的应用程序，如高速伺服电机控制。

  

ISR和非os管理的中断处理程序之间的另一个区别是，在处理程序过程中是否可以调用API。由于ISR是在RTOS管理下运行的，因此可以调用API。然而，由于非OS管理的中断处理程序可以在OS执行关键进程时被激活，自然不可能使用任何其他RTOS函数。因此，如图2所示，不可能使用API将处理程序流程的一部分移交给任务。不能在中断处理程序期间调用API是一个巨大的缺点。通常，如果您追踪任何软件进程的源头，就会发现一个中断。换句话说，当一个中断发生时，一个特定的进程被激活，然后这个进程再激活另一个进程，所以系统就像一条链。因此，当您不能从处理程序调用API时，特别是同步或通信API，这是一个致命的问题。

  

怎么样能让非os管理中断能唤醒对应 Task ？这样既能快速响应，又能拥有在 Task 中调用系统 API 的能力。这就是 Direct Interrupt Service 机制的目的。

  

可以同时把发送给中断 CPU 中断控制器 和 HW-RTOS:

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

当一个中断信号产生时，非os管理的中断处理程序激活，同时HW-RTOS内部调用相应的API。该API与非os管理的中断处理程序并行执行，在本例中，Task B从等待状态被释放。任务B在非os管理的中断处理程序结束的同时被激活：

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

**4.2 具体实现**

  

HW-RTOS最多支持32个hw-isr的架构，如图7所示：

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

HW-ISR都需要在HW-RTOS中设置三个寄存器：

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

**4.3 性能测试**

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

上图显示了中断延迟。从中断的发生到ISR的激活，再到下一个任务的激活，时间是测量的。在软件RTOS中，中断延迟高，抖动大;而在HW-RTOS中，中断延迟低，抖动小。当使用HW ISR时，您可以看到性能上的巨大改进。

  

**4.4 相关应用**

  

**4.4.1 Multiple interrupts using HW ISR**

  

使用HW ISR，很容易实现多个中断。通常，多个中断是通过给每个中断信号分配一个优先级来实现的。在HW ISR中，根据激活的任务的优先级实现多个中断。下图中，值越低优先级越高。在下面的例子中，任务A具有最高的优先级。三个中断线被设置为使用HW ISR来执行API，分别释放信号量6、信号量3和信号量4。当多个中断发生时，根据等待的任务的优先级进行处理。

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

**4.4.2 Cyclic activation task**

  

HW-RTOS没有循环处理函数。然而，使用硬件ISR可以实现等效功能。此外，使用HW ISR的循环激活任务的启动时间比在传统软件RTOS上运行的循环处理程序短。这个过程很简单:定义HW ISR的输入为带有嵌入式R-IN引擎的设备的内置定时器的输出。在下面的例子中，任务A是一个循环激活任务。由于在100 MHz的操作下，最坏情况下的启动时间为3.5微秒，因此可以实现具有极高实时性能的循环处理。

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

通过构建一个包含应用程序示例1中所示的多个中断处理的系统，您可以运行多个循环激活任务。下图显示了两个循环激活任务。任务B是一个周期激活任务，周期是任务a的5倍。实际上，可以定义三个或更多个循环激活任务。每个任务的周期不必互相同步。也可以从外部引脚触发循环激活。

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

**4.4.3 Using cyclic activation task for network synchronization**

  

配备R-IN引擎的瑞萨设备在硬件上具有IEEE1588支持功能。通过安装IEEE1588协议软件，可以在通过网络连接的站点之间进行时间同步。通过向计时器输入同步信号并使用应用程序示例2中所示的方法，您可以在通过网络连接的HW-RTOSs上同步激活循环任务。循环任务的启动波动非常小，在最坏的情况下，在IEEE 1588的波动上增加了3.5微秒的延迟。时间同步也可以使用EtherCAT来实现。

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

## 5.优化点4：Task Management

  

**5.1 原理介绍**

HW-RTOS 的任务管理分为两部分：

  

- 一部分是任务的调度器用来选取哪个任务进行调度。
    
- 另一部分是任务的上下文切换。
    
      
    

**5.1.1 HW-RTOS Scheduler**

  

HW-RTOS支持多达64个任务分配给16个不同的优先级级别。优先级#0是最高优先级，而#15是最低优先级。但是，优先级#15是为空闲任务保留的，在这个优先级级别上，它必须是唯一的。HW-RTOS文档还建议保留优先级#0以备将来使用。虽然64个任务听起来像是一种限制，但是具有这么多任务的应用程序实际上可能相当复杂。

  

如图4所示，HW-RTOS执行与其对应的软件(选择要运行的下一个任务)完全相同的功能，只是它使用硬连线逻辑的速度更快。

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

任务必须先被创建，然后才能被HW-RTOS管理，任务是通过在HW-RTOS的TCBs或任务控制块中写入特殊值来创建的(参见图2)。必须为每个任务分配一个优先级，以及任务堆栈的存储区域(RAM)。

  

HW-RTOS，像大多数实时内核一样，包含一个抢占式调度器。事件导致调度程序暂停任务的执行，以支持由这些事件准备运行的更高优先级的任务。事件可以通过其他任务产生，也可以通过中断设备产生。如果一个任务向高优先级的任务发送信号或消息，HW-RTOS调度器会自动向高优先级的任务发起上下文切换，然后高优先级的任务将处理该事件。类似地，如果一个中断设备向一个任务发送信号或消息，如果该信号或消息指向更高优先级的任务，则当前正在运行的任务将被挂起。在这种情况下，HW-RTOS向这个优先级更高的任务发起上下文切换。

  

**5.1.1 HW-RTOS Context Switch**

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

上下文切换分为两个部分：

  

- 第一个是为HW-RTOS保留的特殊中断处理程序，并被分配到vector #76。
    
- 第二部分是实际的上下文切换(保存和恢复CPU寄存器)，由Cortex-M3的PendSV处理程序执行。
    
      
    

**5.2 具体实现**

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

## 6.典型案例

  

**6.1 Network and RTOS**

  

当TCP/IP协议在嵌入式系统的CPU中实现时，与个人计算机的CPU不同，实现高吞吐量是非常困难的。下图的上半部分显示了使用商用TCP/IP协议栈的传输和接收的概要。只有11%的CPU处理时间花在复杂的协议处理上。其余的时间用于内存复制、重新排列报头、执行TCP校验和和RTOS处理。在这些进程中，内存复制、报头重排和TCP校验和很容易在硬件中实现。下图的中间部分显示了此实现的概要文件。然而，RTOS处理仍然有很高的开销。由于TCP/IP等协议处理具有多个任务，因此每次发送或接收数据包时都需要进行任务切换。还需要多个API调用。这就是为什么RTOS的配置文件有很高的开销。HW-RTOS解决了这个问题。使用HW-RTOS可以大大降低CPU的负载，如图下方所示。也就是说，您可以使用嵌入式系统中使用的低端cpu来实现高网络性能。此外，如果不需要那么高的网络吞吐量，可以使用较低的系统时钟率来大大降低功耗。

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

下图为R-IN32M3的框图。R-IN引擎由HW-RTOS、Cortex®-M3核心和以太网加速器组成。以太网加速器是加速上述内存复制、报头重排、TCP校验和进程的硬件。通过使用R-IN引擎，可以加速TCP/IP和其他网络协议的处理。R-IN发动机包含在R-IN32系列和RZ/N1系列的所有设备中，以及一些RZ/T1设备中。

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

下图显示了在R-IN32M3中实现的UDP/IP的吞吐量。工作时钟为100 MHz，以太网帧长度为1500字节：

  

- 顶部的栏显示了UDP校验和的吞吐量，在软件RTOS实现与HW-RTOS关闭软件执行。
    
- 中间的栏显示了硬件实现的校验和的吞吐量，
    
- 底部的栏显示了HW-RTOS打开时的吞吐量。
    

  

如你所见，带有HW-RTOS的R-IN引擎对于加速网络协议处理非常有效。

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

## 7.下一代HW-RTOS

  

下一代HW-RTOS技术仍在计划中。下一代将继承这一代的所有优势。我们还增加了对紧密耦合、循环处理程序和多核RTOS特性的支持。有了这些，我们计划为RTOS提供更高的功能和性能。

  

- 1.Tightly coupled HW-RTOS
    

  

我们将从紧密耦合(tightly coupled)的HW-RTOS开始。当前一代的HW-RTOS就是我们所说的“松散耦合”(loosely coupled)。在硬件方面，松耦合版本和紧耦合版本的最大区别是在松耦合版本中，CPU、HW-RTOS和CPU寄存器的保存内存都通过系统总线连接。但是，紧耦合版本使用专用接口将这三个模块连接起来。这种体系结构允许改进系统调用性能，包括调度，以及使中断响应性能，我们可以达到传统软件RTOS的十倍以上的性能。当然，几乎没有波动。这允许构建具有更高实时性能水平的系统。

  

- 2.Cyclic handler
    

  

下一代HW-RTOS中的第二个新功能是循环处理器。HW-RTOS中循环处理程序的第一个独特特性是通过在硬件中实现循环处理程序来实现极快的启动。因此，我们将能够缩短处理程序的间隔。循环处理程序间隔将取决于处理程序进程，但应该可以使用微秒级的间隔。这将允许其用于循环控制，甚至高精度的电机控制。我们下一代HW-RTOS循环处理器的第二个独特特性是，可以将启动时间设置为绝对时间。这将允许通过网络连接的站点之间精确同步。

  

- 3.Multi-core support
    

  

下一代HW-RTOS的第三个新功能是多核支持。单核HW-RTOS的所有功能和优点延续到多核HW-RTOS。因此，作为一个实时操作系统，它提供了高水平的实时性能。此外，cpu间的处理，即cpu间的同步和cpu间的通信非常快，执行时间的波动大大减少。这些特性的好处是很容易在多核系统上实现实时应用程序系统。更具体地说，它提供:

  

```
Improved system processing performance
```

  

利用多核HW-RTOS技术，通过高速、高精度的同步，解决了传统软件RTOS中cpu间的同步和通信问题。由于使用多核HW-RTOS可以实现cpu间的同步速度小于1微秒，因此可以实现cpu间的精确定时控制。即使任务是由不同的cpu执行的，它们也是在一个公共的RTOS上定义的。因此，软件开发人员不必担心任务在哪个CPU上运行——软件可以像任务在同一个CPU上运行一样编写。因此，使用多核HW-RTOS不仅可以提供高精度的控制，而且由于系统时钟率低，还可以保持低功耗。

  

阅读 5910

​

写留言

**留言 7**

- 千山雪
    
    2022年2月17日
    
    赞1
    
    很好的文章，感谢分享
    
- 九州风
    
    2022年2月27日
    
    赞
    
    和大佬同名![[得意]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
- 云从龙
    
    2022年2月20日
    
    赞
    
    这是类似ovs offload? 还是采用类似open amp实现的双os(linux + rtos)?
    
    Linux阅码场
    
    作者2022年2月20日
    
    赞
    
    前者，由一个专用的hardware 外设来offload
    
- 李
    
    2022年2月18日
    
    赞
    
    👍
    
- 铄观.lee
    
    2022年2月18日
    
    赞
    
    厉害
    
- 小霸王
    
    2022年2月17日
    
    赞
    
    很棒！
    

已无更多数据

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/YHBSoNHqDiaH5V6ycE2TlM9McXhAbaV9aUBaTAibVhLibXH1rL4JTMb2eoQfdCicTm4wycOsopO82jG3szvYF3mTJw/300?wx_fmt=png&wxfrom=18)

Linux阅码场

331826

7

写留言

**留言 7**

- 千山雪
    
    2022年2月17日
    
    赞1
    
    很好的文章，感谢分享
    
- 九州风
    
    2022年2月27日
    
    赞
    
    和大佬同名![[得意]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
- 云从龙
    
    2022年2月20日
    
    赞
    
    这是类似ovs offload? 还是采用类似open amp实现的双os(linux + rtos)?
    
    Linux阅码场
    
    作者2022年2月20日
    
    赞
    
    前者，由一个专用的hardware 外设来offload
    
- 李
    
    2022年2月18日
    
    赞
    
    👍
    
- 铄观.lee
    
    2022年2月18日
    
    赞
    
    厉害
    
- 小霸王
    
    2022年2月17日
    
    赞
    
    很棒！
    

已无更多数据