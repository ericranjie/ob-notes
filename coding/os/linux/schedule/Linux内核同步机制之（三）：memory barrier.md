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

作者：[linuxer](http://www.wowotech.net/author/3 "linuxer") 发布于：2014-11-14 19:20 分类：[内核同步机制](http://www.wowotech.net/sort/kernel_synchronization)

一、前言

我记得以前上学的时候大家经常说的一个词汇叫做所见即所得，有些编程工具是所见即所得的，给程序员带来极大的方便。对于一个c程序员，我们的编写的代码能所见即所得吗？我们看到的c程序的逻辑是否就是最后CPU运行的结果呢？很遗憾，不是，我们的“所见”和最后的执行结果隔着：

1、编译器

2、CPU取指执行

编译器将符合人类思考的逻辑（c代码）翻译成了符合CPU运算规则的汇编指令，编译器了解底层CPU的思维模式，因此，它可以在将c翻译成汇编的时候进行优化（例如内存访问指令的重新排序），让产出的汇编指令在CPU上运行的时候更快。然而，这种优化产出的结果未必符合程序员原始的逻辑，因此，作为程序员，作为c程序员，必须有能力了解编译器的行为，并在通过内嵌在c代码中的memory barrier来指导编译器的优化行为（这种memory barrier又叫做优化屏障，Optimization barrier），让编译器产出即高效，又逻辑正确的代码。

CPU的核心思想就是取指执行，对于in-order的单核CPU，并且没有cache（这种CPU在现实世界中还存在吗？），汇编指令的取指和执行是严格按照顺序进行的，也就是说，汇编指令就是所见即所得的，汇编指令的逻辑被严格的被CPU执行。然而，随着计算机系统越来越复杂（多核、cache、superscalar、out-of-order），使用汇编指令这样贴近处理器的语言也无法保证其被CPU执行的结果的一致性，从而需要程序员（看，人还是最不可以替代的）告知CPU如何保证逻辑正确。

综上所述，memory barrier是一种保证内存访问顺序的一种方法，让系统中的HW block（各个cpu、DMA controler、device等）对内存有一致性的视角。

二、不使用memory barrier会导致问题的场景

1、编译器的优化

我们先看下面的一个例子：

> preempt_disable（）
> 
> 临界区
> 
> preempt_enable

有些共享资源可以通过禁止任务抢占来进行保护，因此临界区代码被preempt_disable和preempt_enable给保护起来。其实，我们知道所谓的preempt enable和disable其实就是对当前进程的struct thread_info中的preempt_count进行加一和减一的操作。具体的代码如下：

> #define preempt_disable() \  
> do { \  
>     preempt_count_inc(); \  
>     barrier(); \  
> } while (0)

linux kernel中的定义和我们的想像一样，除了barrier这个优化屏障。barrier就象是c代码中的一个栅栏，将代码逻辑分成两段，barrier之前的代码和barrier之后的代码在经过编译器编译后顺序不能乱掉。也就是说，barrier之后的c代码对应的汇编，不能跑到barrier之前去，反之亦然。之所以这么做是因为在我们这个场景中，如果编译为了榨取CPU的performace而对汇编指令进行重排，那么临界区的代码就有可能位于preempt_count_inc之外，从而起不到保护作用。

现在，我们知道了增加barrier的作用，问题来了，barrier是否够呢？对于multi-core的系统，只有当该task被调度到该CPU上执行的时候，该CPU才会访问该task的preempt count，因此对于preempt enable和disable而言，不存在多个CPU同时访问的场景。但是，即便这样，如果CPU是乱序执行（out-of-order excution）的呢？其实，我们也不用担心，正如前面叙述的，preempt count这个memory实际上是不存在多个cpu同时访问的情况，因此，它实际上会本cpu的进程上下文和中断上下文访问。能终止当前thread执行preempt_disable的只有中断。为了方便描述，我们给代码编址，如下：

|   |   |   |
|---|---|---|
|地址|该地址的汇编指令|CPU的执行顺序|
|a|preempt_disable（）|临界区指令1|
|a+4|临界区指令1|preempt_disable（）|
|a+8|临界区指令2|临界区指令2|
|a+12|preempt_enable|preempt_enable|

当发生中断的时候，硬件会获取当前PC值，并精确的得到了发生指令的地址。有两种情况：

（1）在地址a发生中断。对于out-of-order的CPU，临界区指令1已经执行完毕，preempt_disable正在pipeline中等待执行。由于是在a地址发生中断，也就是preempt_disable地址上发生中断，对于硬件而言，它会保证a地址之前（包括a地址）的指令都被执行完毕，并且a地址之后的指令都没有执行。因此，在这种情况下，临界区指令1的执行结果被抛弃掉，因此，实际临界区指令不会先于preempt_disable执行

（2）在地址a＋4发生中断。这时候，虽然发生中断的那一刻的地址上的指令（临界区指令1）已经执行完毕了，但是硬件会保证地址a＋4之前的所有的指令都执行完毕，因此，实际上CPU会执行完preempt_disable，然后跳转的中断异常向量执行。

上面描述的是优化屏障在内存中的变量的应用，下面我们看看硬件寄存器的场景。一般而言，串口的驱动都会包括控制台部分的代码，例如：

> static struct console xx_serial_console = {  
> ……  
>     .write        = xx_serial_console_write,  
> ……  
> };

如果系统enable了串口控制台，那么当你的驱动调用printk的时候，实际上最终是通过console的write函数输出到了串口控制台。而这个console write的函数可能会包含下面的代码：

> do {  
>     获取TX FIFO状态寄存器  
>     barrier();  
> } while (TX FIFO没有ready);  
> 写TX FIFO寄存器;

对于某些CPU archtecture而言（至少ARM是这样的），外设硬件的IO地址也被映射到了一段内存地址空间，对编译器而言，它并不知道这些地址空间是属于外设的。因此，对于上面的代码，如果没有barrier的话，获取TX FIFO状态寄存器的指令可能和写TX FIFO寄存器指令进行重新排序，在这种情况下，程序逻辑就不对了，因为我们必须要保证TX FIFO ready的情况下才能写TX FIFO寄存器。

对于multi core的情况，上面的代码逻辑也是OK的，因为在调用console write函数的时候，要获取一个console semaphore，确保了只有一个thread进入，因此，console write的代码不会在多个CPU上并发。和preempt count的例子一样，我们可以问同样的问题，如果CPU是乱序执行（out-of-order excution）的呢？barrier只是保证compiler输出的汇编指令的顺序是OK的，不能确保CPU执行时候的乱序。 对这个问题的回答来自ARM architecture的内存访问模型：对于program order是A1-->A2的情况（A1和A2都是对Device或是Strongly-ordered的memory进行访问的指令），ARM保证A1也是先于A2执行的。因此，在这样的场景下，使用barrier足够了。 对于X86也是类似的，虽然它没有对IO space采样memory mapping的方式，但是，X86的所有操作IO端口的指令都是被顺执行的，不需要考虑memory access order。  
 

2、cpu architecture和cache的组织

注：本章节的内容来自对Paul E. McKenney的Why memory barriers文档理解，更细致的内容可以参考该文档。这个章节有些晦涩，需要一些耐心。作为一个c程序员，你可能会抱怨，为何设计CPU的硬件工程师不能屏蔽掉memory barrier的内容，让c程序员关注在自己需要关注的程序逻辑上呢？本章可以展开叙述，或许能解决一些疑问。

（1）基本概念

在[The Memory Hierarchy](http://www.wowotech.net/basic_subject/memory-hierarchy.html)文档中，我们已经了解了关于cache一些基础的知识，一些基础的内容，这里就不再重复了。我们假设一个多核系统中的cache如下：

 [![cache arch](http://www.wowotech.net/content/uploadfile/201411/6ae345c874b4a99f06046f32377c7af320141114112003.gif "cache arch")](http://www.wowotech.net/content/uploadfile/201411/e35f2f4793d734a566d1d230d1b83b4620141114112002.gif)

我们先了解一下各个cpu cache line状态的迁移过程：

（a）我们假设在有一个memory中的变量为多个CPU共享，那么刚开始的时候，所有的CPU的本地cache中都没有该变量的副本，所有的cacheline都是invalid状态。

（b）因此当cpu 0 读取该变量的时候发生cache miss（更具体的说叫做cold miss或者warmup miss）。当该值从memory中加载到chache 0中的cache line之后，该cache line的状态被设定为shared，而其他的cache都是Invalid。

（c）当cpu 1 读取该变量的时候，chache 1中的对应的cache line也变成shared状态。其实shared状态就是表示共享变量在一个或者多个cpu的cache中有副本存在。既然是被多个cache所共享，那么其中一个CPU就不能武断修改自己的cache而不通知其他CPU的cache，否则会有一致性问题。

（d）总是read多没劲，我们让CPU n对共享变量来一个load and store的操作。这时候，CPU n发送一个read invalidate命令，加载了Cache n的cache line，并将状态设定为exclusive，同时将所有其他CPU的cache对应的该共享变量的cacheline设定为invalid状态。正因为如此，CPU n实际上是独占了变量对应的cacheline（其他CPU的cacheline都是invalid了，系统中就这么一个副本），就算是写该变量，也不需要通知其他的CPU。CPU随后的写操作将cacheline设定为modified状态，表示cache中的数据已经dirty，和memory中的不一致了。modified状态和exclusive状态都是独占该cacheline，但是modified状态下，cacheline的数据是dirty的，而exclusive状态下，cacheline中的数据和memory中的数据是一致的。当该cacheline被替换出cache的时候，modified状态的cacheline需要write back到memory中，而exclusive状态不需要。

（e）在cacheline没有被替换出CPU n的cache之前，CPU 0再次读该共享变量，这时候会怎么样呢？当然是cache miss了（因为之前由于CPU n写的动作而导致其他cpu的cache line变成了invalid，这种cache miss叫做communiction miss）。此外，由于CPU n的cache line是modified状态，它必须响应这个读得操作（memory中是dirty的）。因此，CPU 0的cacheline变成share状态（在此之前，CPU n的cache line应该会发生write back动作，从而导致其cacheline也是shared状态）。当然，也可能是CPU n的cache line不发生write back动作而是变成invalid状态，CPU 0的cacheline变成modified状态，这和具体的硬件设计相关。

（2）Store buffer

我们考虑另外一个场景：在上一节中step e中的操作变成CPU 0对共享变量进行写的操作。这时候，写的性能变得非常的差，因为CPU 0必须要等到CPU n上的cacheline 数据传递到其cacheline之后，才能进行写的操作（CPU n上的cacheline 变成invalid状态，CPU 0则切换成exclusive状态，为后续的写动作做准备）。而从一个CPU的cacheline传递数据到另外一个CPU的cacheline是非常消耗时间的，而这时候，CPU 0的写的动作只是hold住，直到cacheline的数据完成传递。而实际上，这样的等待是没有意义的，因此，这时候cacheline的数据仍然会被覆盖掉。为了解决这个问题，多核系统中的cache修改如下：

[![cache arch1](http://www.wowotech.net/content/uploadfile/201411/4f1fe5220dacaa8ac8f18f4efd43b5b020141114112007.gif "cache arch1")](http://www.wowotech.net/content/uploadfile/201411/a872a1863fec02585bb786a5c382d3eb20141114112005.gif)

这样，问题解决了，写操作不必等到cacheline被加载，而是直接写到store buffer中然后欢快的去干其他的活。在CPU n的cacheline把数据传递到其cache 0的cacheline之后，硬件将store buffer中的内容写入cacheline。

虽然性能问题解决了，但是逻辑错误也随之引入，我们可以看下面的例子：

我们假设a和b是共享变量，初始值都是0，可以被cpu0和cpu1访问。cpu 0的cache中保存了b的值（exclusive状态），没有a的值，而cpu 1的cache中保存了a的值，没有b的值，cpu 0执行的汇编代码是（用的是ARM汇编，没有办法，其他的都不是那么熟悉）：

> ldr     r2, [pc, #28]   -------------------------- 取变量a的地址  
> ldr     r4, [pc, #20]   -------------------------- 取变量b的地址  
> mov     r3, #1  
> str     r3, [r2]           --------------------------a=1  
> str     r3, [r4]           --------------------------b=1

CPU 1执行的代码是：

>              ldr     r2, [pc, #28]   -------------------------- 取变量a的地址
> 
>              ldr     r3, [pc, #20]  -------------------------- 取变量b的地址  
> start:     ldr     r3, [r3]          -------------------------- 取变量b的值  
>             cmp     r3, #0          ------------------------ b的值是否等于0？  
>             beq     start            ------------------------ 等于0的话跳转到start
> 
>             ldr     r2, [r2]          -------------------------- 取变量a的值

当cpu 1执行到--取变量a的值--这条指令的时候，b已经是被cpu0修改为1了，这也就是说a＝1这个代码已经执行了，因此，从汇编代码的逻辑来看，这时候a值应该是确定的1。然而并非如此，cpu 0和cpu 1执行的指令和动作描述如下：

|   |   |   |   |
|---|---|---|---|
|cpu 0执行的指令|cpu 0动作描述|cpu 1执行的指令|cpu 1动作描述|
|str     r3, [r2]  <br>（a=1）|1、发生cache miss  <br>2、将1保存在store buffer中  <br>3、发送read invalidate命令，试图从cpu 1的cacheline中获取数据，并invalidate其cache line  <br>  <br>注：这里无需等待response，立刻执行下一条指令|ldr     r3, [r3]   <br>（获取b的值）|1、发生cache miss  <br>2、发送read命令，试图加载b对应的cacheline  <br>  <br>注：这里cpu必须等待read response，下面的指令依赖于这个读取的结果|
|str     r3, [r4]   <br>（b=1）|1、cache hit  <br>2、cacheline中的值被修改为1，状态变成modified|||
||响应cpu 1的read命令，发送read response（b＝1）给CPU 0。write back，将状态设定为shared|||
|||cmp     r3, #0|1、cpu 1收到来自cpu 0的read response，加载b对应的cacheline，状态为shared  <br>2、b等于1，因此不必跳转到start执行|
|||ldr     r2, [r2]  <br>（获取a的值）|1、cache hit  <br>2、获取了a的旧值，也就是0|
||||响应CPU 0的read invalid命令，将a对应的cacheline设为invalid状态，发送read response和invalidate ack。但是已经酿成大错了。|
||收到来自cpu 1的响应，将store buffer中的1写入cache line。|||

  对于硬件，CPU不清楚具体的代码逻辑，它不可能直接帮助软件工程师，只是提供一些memory barrier的指令，让软件工程师告诉CPU他想要的内存访问逻辑顺序。这时候，cpu 0的代码修改如下：

> ldr     r2, [pc, #28]   -------------------------- 取变量a的地址  
> ldr     r4, [pc, #20]   -------------------------- 取变量b的地址  
> mov     r3, #1  
> str     r3, [r2]           --------------------------a=1
> 
> 确保清空store buffer的memory barrier instruction  
> str     r3, [r4]           --------------------------b=1

这种情况下，cpu 0和cpu 1执行的指令和动作描述如下：

|   |   |   |   |
|---|---|---|---|
|cpu 0执行的指令|cpu 0动作描述|cpu 1执行的指令|cpu 1动作描述|
|str     r3, [r2]  <br>（a=1）|1、发生cache miss  <br>2、将1保存在store buffer中  <br>3、发送read invalidate命令，试图从cpu 1的cacheline中获取数据，并invalidate其cache line  <br>  <br>注：这里无需等待response，立刻执行下一条指令|ldr     r3, [r3]   <br>（获取b的值）|1、发生cache miss  <br>2、发送read命令，试图加载b对应的cacheline  <br>  <br>注：这里cpu必须等待read response，下面的指令依赖于这个读取的结果|
|memory barrier instruction|CPU收到memory barrier指令，知道软件要控制访问顺序，因此不会执行下一条str指令，要等到收到read response和invalidate ack后，将store buffer中所有数据写到cacheline之后才会执行后续的store指令|||
|||cmp     r3, #0  <br>beq     start|1、cpu 1收到来自cpu 0的read response，加载b对应的cacheline，状态为shared  <br>2、b等于0，跳转到start执行|
||||响应CPU 0的read invalid命令，将a对应的cacheline设为invalid状态，发送read response和invalidate ack。|
||收到来自cpu 1的响应，将store buffer中的1写入cache line。|||
|str     r3, [r4]   <br>（b=1）|1、cache hit，但是cacheline状态是shared，需要发送invalidate到cpu 1  <br>2、将1保存在store buffer中  <br>  <br>注：这里无需等待invalidate ack，立刻执行下一条指令|||
|…|…|…|…|

由于增加了memory barrier，保证了a、b这两个变量的访问顺序，从而保证了程序逻辑。

（3）Invalidate Queue

我们先回忆一下为何出现了stroe buffer：为了加快cache miss状态下写的性能，硬件提供了store buffer，以便让CPU先写入，从而不必等待invalidate ack（这些交互是为了保证各个cpu的cache的一致性）。然而，store buffer的size比较小，不需要特别多的store命令（假设每次都是cache miss）就可以将store buffer填满，这时候，没有空间写了，因此CPU也只能是等待invalidate ack了，这个状态和memory barrier指令的效果是一样的。

怎么解决这个问题？CPU设计的硬件工程师对性能的追求是不会停歇的。我们首先看看invalidate ack为何如此之慢呢？这主要是因为cpu在收到invalidate命令后，要对cacheline执行invalidate命令，确保该cacheline的确是invalid状态后，才会发送ack。如果cache正忙于其他工作，当然不能立刻执行invalidate命令，也就无法会ack。

怎么破？CPU设计的硬件工程师提供了下面的方法：

[![cache arch2](http://www.wowotech.net/content/uploadfile/201411/b4c569d306427421b5b657fdcfce3cf120141114112010.gif "cache arch2")](http://www.wowotech.net/content/uploadfile/201411/46e1bbd0ba094941caf23050e1db2d2d20141114112008.gif)

Invalidate Queue这个HW block从名字就可以看出来是保存invalidate请求的队列。其他CPU发送到本CPU的invalidate命令会保存于此，这时候，并不需要等到实际对cacheline的invalidate操作完成，CPU就可以回invalidate ack了。

同store buffer一样，虽然性能问题解决了，但是对memory的访问顺序导致的逻辑错误也随之引入，我们可以看下面的例子（和store buffer中的例子类似）：

我们假设a和b是共享变量，初始值都是0，可以被cpu0和cpu1访问。cpu 0的cache中保存了b的值（exclusive状态），而CPU 1和CPU 0的cache中都保存了a的值，状态是shared。cpu 0执行的汇编代码是：

> ldr     r2, [pc, #28]   -------------------------- 取变量a的地址  
> ldr     r4, [pc, #20]   -------------------------- 取变量b的地址  
> mov     r3, #1  
> str     r3, [r2]           --------------------------a=1
> 
> 确保清空store buffer的memory barrier instruction  
> str     r3, [r4]           --------------------------b=1

CPU 1执行的代码是：

>              ldr     r2, [pc, #28]   -------------------------- 取变量a的地址
> 
>              ldr     r3, [pc, #20]  -------------------------- 取变量b的地址  
> start:     ldr     r3, [r3]          -------------------------- 取变量b的值  
>             cmp     r3, #0          ------------------------ b的值是否等于0？  
>             beq     start            ------------------------ 等于0的话跳转到start
> 
>             ldr     r2, [r2]          -------------------------- 取变量a的值

这种情况下，cpu 0和cpu 1执行的指令和动作描述如下：

|   |   |   |   |
|---|---|---|---|
|cpu 0执行的指令|cpu 0动作描述|cpu 1执行的指令|cpu 1动作描述|
|str     r3, [r2]  <br>（a=1）|1、a值在CPU 0的cache中状态是shared，是read only的，因此，需要通知其他的CPU  <br>2、将1保存在store buffer中  <br>3、发送invalidate命令，试图invalidate CPU 1中a对应的cache line  <br>  <br>注：这里无需等待response，立刻执行下一条指令|ldr     r3, [r3]   <br>（获取b的值）|1、发生cache miss  <br>2、发送read命令，试图加载b对应的cacheline  <br>  <br>注：这里cpu必须等待read response，下面的指令依赖于这个读取的结果|
||||收到来自CPU 0的invalidate命令，放入invalidate queue，立刻回ack。|
|memory barrier instruction|CPU收到memory barrier指令，知道软件要控制访问顺序，因此不会执行下一条str指令，要等到收到invalidate ack后，将store buffer中所有数据写到cacheline之后才会执行后续的store指令|||
||收到invalidate ack后，将store buffer中的1写入cache line。OK，可以继续执行下一条指令了|||
|str     r3, [r4]   <br>（b=1）|1、cache hit  <br>2、cacheline中的值被修改为1，状态变成modified|||
||收到CPU 1发送来的read命令，将b值（等于1）放入read response中，回送给CPU 1，write back并将状态修改为shared。|||
||||收到response（b＝1），并加载cacheline，状态是shared|
|||cmp     r3, #0|b等于1，不会执行beq指令，而是执行下一条指令|
|||ldr     r2, [r2]  <br>（获取a的值）|1、cache hit （还没有执行invalidate动作，命令还在invalidate queue中呢）  <br>2、获取了a的旧值，也就是0|
||||对a对应的cacheline执行invalidate 命令，但是，已经晚了|

可怕的memory misorder问题又来了，都是由于引入了invalidate queue引起，看来我们还需要一个memory barrier的指令，我们将程序修改如下：

>              ldr     r2, [pc, #28]   -------------------------- 取变量a的地址
> 
>              ldr     r3, [pc, #20]  -------------------------- 取变量b的地址  
> start:     ldr     r3, [r3]          -------------------------- 取变量b的值  
>             cmp     r3, #0          ------------------------ b的值是否等于0？  
>             beq     start            ------------------------ 等于0的话跳转到start
> 
> 确保清空invalidate queue的memory barrier instruction
> 
>             ldr     r2, [r2]          -------------------------- 取变量a的值

这种情况下，cpu 0和cpu 1执行的指令和动作描述如下：

|   |   |   |   |
|---|---|---|---|
|cpu 0执行的指令|cpu 0动作描述|cpu 1执行的指令|cpu 1动作描述|
|str     r3, [r2]  <br>（a=1）|1、a值在CPU 0的cache中状态是shared，是read only的，因此，需要通知其他的CPU  <br>2、将1保存在store buffer中  <br>3、发送invalidate命令，试图invalidate CPU 1中a对应的cache line  <br>  <br>注：这里无需等待response，立刻执行下一条指令|ldr     r3, [r3]   <br>（获取b的值）|1、发生cache miss  <br>2、发送read命令，试图加载b对应的cacheline  <br>  <br>注：这里cpu必须等待read response，下面的指令依赖于这个读取的结果|
||||收到来自CPU 0的invalidate命令，放入invalidate queue，立刻回ack。|
|memory barrier instruction|CPU收到memory barrier指令，知道软件要控制访问顺序，因此不会执行下一条str指令，要等到收到invalidate ack后，将store buffer中所有数据写到cacheline之后才会执行后续的store指令|||
||收到invalidate ack后，将store buffer中的1写入cache line。OK，可以继续执行下一条指令了|||
|str     r3, [r4]   <br>（b=1）|1、cache hit  <br>2、cacheline中的值被修改为1，状态变成modified|||
||收到CPU 1发送来的read命令，将b值（等于1）放入read response中，回送给CPU 1，write back并将状态修改为shared。|||
||||收到response（b＝1），并加载cacheline，状态是shared|
|||cmp     r3, #0|b等于1，不会执行beq指令，而是执行下一条指令|
|||memory barrier instruction|CPU收到memory barrier指令，知道软件要控制访问顺序，因此不会执行下一条ldr指令，要等到执行完invalidate queue中的所有的invalidate命令之后才会执行下一个ldr指令|
|||ldr     r2, [r2]  <br>（获取a的值）|1、cache miss  <br>2、发送read命令，从CPU 0那里加载新的a值|

  由于增加了memory barrier，保证了a、b这两个变量的访问顺序，从而保证了程序逻辑。

三、linux kernel的API

linux kernel的memory barrier相关的API列表如下：

|   |   |
|---|---|
|接口名称|作用|
|barrier()|优化屏障，阻止编译器为了进行性能优化而进行的memory access reorder|
|mb()|内存屏障（包括读和写），用于SMP和UP|
|rmb()|读内存屏障，用于SMP和UP|
|wmb()|写内存屏障，用于SMP和UP|
|smp_mb()|用于SMP场合的内存屏障，对于UP不存在memory order的问题（对汇编指令），因此，在UP上就是一个优化屏障，确保汇编和c代码的memory order是一致的|
|smp_rmb()|用于SMP场合的读内存屏障|
|smp_wmb()|用于SMP场合的写内存屏障|

barrier()这个接口和编译器有关，对于gcc而言，其代码如下：

> #define barrier() __asm__ __volatile__("": : :"memory")

这里的__volatile__主要是用来防止编译器优化的。而这里的优化是针对代码块而言的，使用嵌入式汇编的代码分成三块：

1、嵌入式汇编之前的c代码块

2、嵌入式汇编代码块

3、嵌入式汇编之后的c代码块

这里__volatile__就是告诉编译器：不要因为性能优化而将这些代码重排，我需要清清爽爽的保持这三块代码块的顺序（代码块内部是否重排不是这里的__volatile__管辖范围了）。

barrier中的嵌入式汇编中的clobber list没有描述汇编代码对寄存器的修改情况，只是有一个memory的标记。我们知道，clober list是gcc和gas的接口，用于gas通知gcc它对寄存器和memory的修改情况。因此，这里的memory就是告知gcc，在汇编代码中，我修改了memory中的内容，嵌入式汇编之前的c代码块和嵌入式汇编之后的c代码块看到的memory是不一样的，对memory的访问不能依赖于嵌入式汇编之前的c代码块中寄存器的内容，需要重新加载。

优化屏障是和编译器相关的，而内存屏障是和CPU architecture相关的，当然，我们选择ARM为例来描述内存屏障。

四、内存屏障在ARM中的实现

TODO

  

_原创文章，转发请注明出处。蜗窝科技_

[http://www.wowotech.net/kernel_synchronization/memory-barrier.html](http://www.wowotech.net/kernel_synchronization/memory-barrier.html)

标签: [Memory](http://www.wowotech.net/tag/Memory) [内存屏障](http://www.wowotech.net/tag/%E5%86%85%E5%AD%98%E5%B1%8F%E9%9A%9C) [barrier](http://www.wowotech.net/tag/barrier)

[![](http://www.wowotech.net/content/uploadfile/201605/ef3e1463542768.png)](http://www.wowotech.net/support_us.html)

« [程序员的“纪律性”（《程序员杂志》署名文章）](http://www.wowotech.net/tech_discuss/109.html) | [Linux PM domain framework(1)_概述和使用流程](http://www.wowotech.net/pm_subsystem/pm_domain_overview.html)»

**评论：**

**wmzjzwlzs**  
2022-12-11 10:09

smp_mb实际是dmb ish，在inner shareable内的多core管用.如果是多个cluster之间的core应该是dmb osh吧？linux支持多cluster为啥smp_mb不是dmb osh？

[回复](http://www.wowotech.net/kernel_synchronization/memory-barrier.html#comment-8718)

**muzimuke**  
2022-12-04 15:11

"它实际上会本cpu的进程上下文和中断上下文访问"  
文中的这句话是不是写错了呀？

[回复](http://www.wowotech.net/kernel_synchronization/memory-barrier.html#comment-8713)

**1912**  
2021-08-26 16:59

>当该值从memory中加载到chache 0中的cache line之后，该cache line的状态被设定为shared  
明明是exclusive的

[回复](http://www.wowotech.net/kernel_synchronization/memory-barrier.html#comment-8280)

**tuhh**  
2019-04-20 11:27

能不能将参考文献也一起列一下呢

[回复](http://www.wowotech.net/kernel_synchronization/memory-barrier.html#comment-7364)

**blazer**  
2022-09-27 15:35

@tuhh：我也觉得参考文献列出来好一些，毕竟cache的这些性质俺们也不一定懂呢

[回复](http://www.wowotech.net/kernel_synchronization/memory-barrier.html#comment-8675)

**hayden**  
2018-04-20 14:41

有点搞不懂，CPU的乱序执行，为什么要和中断扯上关系，那如果在执行这段代码时，CPU是乱序执行的，没有  
中断来，这样代码不就是按照下表最右边的CPU执行顺序一致了吗，那这个时候CPU执行的代码顺序和汇编指令  
的顺序不一致了？  
  
地址    该地址的汇编指令            CPU的执行顺序  
a    preempt_disable（）    临界区指令1  
a+4    临界区指令1            preempt_disable（）  
a+8    临界区指令2            临界区指令2  
a+12    preempt_enable            preempt_enable

[回复](http://www.wowotech.net/kernel_synchronization/memory-barrier.html#comment-6686)

**Xavior**  
2020-03-25 16:32

@hayden：这里的中断只是为了说明，CPU乱序执行不会对使用barrier（）造成影响

[回复](http://www.wowotech.net/kernel_synchronization/memory-barrier.html#comment-7928)

**pansila**  
2018-01-15 22:40

do {  
    获取TX FIFO状态寄存器  
    barrier();  
} while (TX FIFO没有ready);  
写TX FIFO寄存器;  
  
这里不明白，难道barrier不是应该加到循环外就可以了么  
  
do {  
    获取TX FIFO状态寄存器  
} while (TX FIFO没有ready);  
barrier();  
写TX FIFO寄存器;

[回复](http://www.wowotech.net/kernel_synchronization/memory-barrier.html#comment-6474)

**pansila**  
2018-01-15 22:18

第一个问题，如果已经提交了，应该就属于中断发生在了下一条指令，需要把后续指令也提交的情况；第二个问题也是我想问的，乱序两条临界指令，而后禁止抢占还能救回来么？

[回复](http://www.wowotech.net/kernel_synchronization/memory-barrier.html#comment-6472)

**[linuxer](http://www.wowotech.net/)**  
2018-01-17 14:53

@pansila：这份文档是很久以前写的，是在不了解CPU微架构的情况下自己的胡思乱想。实际上即便是乱序执行的CPU，其实也是顺序提交的，确定了中断命中的指令之后，CPU要做的就是把那些提前执行但为提交的指令结果进行回滚。

[回复](http://www.wowotech.net/kernel_synchronization/memory-barrier.html#comment-6481)

**adance**  
2017-03-22 23:44

您好，内存一致性协议比如MESI跟内存屏障有啥区别呢？内存屏障解决的问题不应该是一致性协议的内容吗

[回复](http://www.wowotech.net/kernel_synchronization/memory-barrier.html#comment-5362)

**[linuxer](http://www.wowotech.net/)**  
2017-03-23 09:25

@adance：memory barrier是解决memory order问题的。例如：你写了一段c程序，从program order来看，对内存A的访问先于对内存B的访问，但是在实际cpu的执行过程中，B却先于A而执行。这种乱序一方面来自编译器，可以通过Optimization barrier进行限制，另外一方面来自cpu，这可以通过memory barrier指令来解决。  
  
一致性应该是cache coherence，和memory hierarcy相关，发生在多个master访问共享的memory资源的时候。

[回复](http://www.wowotech.net/kernel_synchronization/memory-barrier.html#comment-5365)

**adance**  
2017-03-23 11:07

@linuxer：感谢您的回复！  
我们先抛开编译器优化带来的乱序，就讨论你上文第二种cpu乱序：  
其实你上面你讲的过程中就是基于一致性协议MESI的，  
我是这么理解的：以上你讲的CPU内存访问乱序其实是由于store buffer和 Invalidate Queue等结构造成的，假如没有它们，这里以你上文的store buffer例子来假设，假设没有store buffer，cpu0读取变量a的值时只能等待cpu1返回a的值并对invalidate的回应，这样的话是不是就不会有乱序了呢？  
Invalidate Queue也可以以此推断，但是cpu加这两个就是为了提升性能，不至于空等待，加快处理速度，所以我觉得，这种cpu内存访问乱序的问题是为了考虑性能而对一致性协议MESI只支持弱一致性，要是强一致性，理想状态下就不会有这种问题，所以这种乱序还是属于一致性范围，cpu也提供了软件层面的memory barrier指令，来满足应用某些场合强一致性的要求。  
请指教！

[回复](http://www.wowotech.net/kernel_synchronization/memory-barrier.html#comment-5367)

**[linuxer](http://www.wowotech.net/)**  
2017-03-23 16:33

@adance：其实我对这些概念也不是理解的特别透彻，但是我的理解就是memory barrier和cache coherence protocol是两码事。

[回复](http://www.wowotech.net/kernel_synchronization/memory-barrier.html#comment-5370)

**adance**  
2017-04-06 22:10

@linuxer：嗯...有心得可以交流...还有一个问题请教：你怎么理解虚拟地址，逻辑地址，线性地址的?有些书说逻辑地址就是虚拟地址，有些书说线性地址是虚拟地址...烦请指教

[回复](http://www.wowotech.net/kernel_synchronization/memory-barrier.html#comment-5430)

**[linuxer](http://www.wowotech.net/)**  
2017-04-07 16:52

@adance：logical address和linear address是X86平台特定的术语，不适合其他平台。  
对于X86处理器，地址映射分成两部，logical address通过segmentation unit的映射之后变成liear address，然后通过paging unit，linear address映射成物理地址。一般的处理器都是虚拟地址通过MMU unit映射成物理地址，这是比较通用的概念，当然，也有稍微复杂一点的，例如在支持虚拟化的ARMv8中，VA到PA的mapping可以有IPA这样的一个中间level。  
  
对于虚拟地址的定义，我的理解就是在汇编指令中操作的那些地址，从这个角度看，x86平台的logical address更接近我对虚拟地址的理解。

[回复](http://www.wowotech.net/kernel_synchronization/memory-barrier.html#comment-5432)

**randy**  
2015-12-02 11:45

说的围绕多个cpu对cache一致性的维护，是否可以说明下单核下mb的实现是怎么样的？

[回复](http://www.wowotech.net/kernel_synchronization/memory-barrier.html#comment-3187)

**[linuxer](http://www.wowotech.net/)**  
2015-12-03 08:41

@randy：这份文档是我准备废弃的文章之一，写的太凌乱了，很容易让人误解，你可以稍微等等，后续我们重写这一篇的。

[回复](http://www.wowotech.net/kernel_synchronization/memory-barrier.html#comment-3190)

**zachary**  
2016-07-26 17:55

@linuxer：重写的在哪里？

[回复](http://www.wowotech.net/kernel_synchronization/memory-barrier.html#comment-4321)

**[linuxer](http://www.wowotech.net/)**  
2016-07-27 16:53

@zachary：抱歉，写这份文档的时候不知道天高地厚，现在了解了一些反而不敢下笔了，hehe～～因此我们没有重写，只是翻译了perfbook中的几个章节，如果有兴趣可以看看。  
等到真正理解了memory barrier，我们再写一篇好了。

[回复](http://www.wowotech.net/kernel_synchronization/memory-barrier.html#comment-4326)

**[tigger](http://www.wowotech.net/)**  
2014-11-17 18:33

因此，在这种情况下，临界区指令1的执行结果被抛弃掉，因此，实际临界区指令不会先于preempt_disable执行  
这个是arm 硬件实现的吗？

[回复](http://www.wowotech.net/kernel_synchronization/memory-barrier.html#comment-742)

**[linuxer](http://www.wowotech.net/)**  
2014-11-17 19:39

@tigger：对于那些支持superscalar或者out-of-order excution的CPU，CPU core必须从一段代码中精准的找到一条指令作为发生中断的现场，该条指令（包括这条指令）之前的指令必须执行完毕，而之后的指令的运行结果必须抛弃掉（如果memory type是normal类型），这样的操作当然是ARM硬件完成的。  
  
问题：如果memory type是device type或者strong order type的，会如何处理？呵呵～～～我文章中也给出答案了

[回复](http://www.wowotech.net/kernel_synchronization/memory-barrier.html#comment-746)

**Hisenberg**  
2017-11-01 18:33

@linuxer：linuxer，对于“临界区指令1的执行结果被抛弃掉”，假如这条指令是对某个变量加4，或者像内存中写入一个值，那怎么抛弃结果呢？恢复该变量或内存去到初始值吗？如果不是的话，从中断回来以后，会重新执行a地址之后的指令，那是不是会造成临界区指令1被执行了2次，即变量被加了两次或者内存地址被写了2次？这是没有barrier可能造成的后果吗？

[回复](http://www.wowotech.net/kernel_synchronization/memory-barrier.html#comment-6160)

**[linuxer](http://www.wowotech.net/)**  
2017-11-02 12:00

@Hisenberg：这里我们需要对“指令执行”分的更细致一些：对于一个汇编指令，例如给某个内存写入一个值，它的执行过程分成两个步骤，一个是执行阶段，另外一个是执行结果的提交阶段，完成了指令的提交阶段，内存的数值才会被修改。  
  
因此，所谓“临界区指令1的执行结果被抛弃掉”是指在第一个执行阶段的结果被抛弃掉，或者说把放入到commit buffer中的执行结果抛弃掉，而这个执行结果实际上并没有真正的被提交。

[回复](http://www.wowotech.net/kernel_synchronization/memory-barrier.html#comment-6162)

**Hisenberg**  
2017-11-02 14:24

@linuxer：再请问一下哦，那硬件会保证在中断发生的时候，“临界区指令1”的执行结果不会提交吗？或者说硬件在什么情况下会认为可以提交指令的执行结果了呢？因为如果只是时间问题的话，会不会出现这种乱序情况“临界区指令1，临界区指令2，preempt dis, preempt en”，这个时候，中断还是在a地址发生，但是前面已经执行了2条指令，并且第一条的结果已经提交了呢？

[回复](http://www.wowotech.net/kernel_synchronization/memory-barrier.html#comment-6164)

**pansila**  
2018-01-15 22:33

@Hisenberg：应该没法区分吧，可以提交的乱序指令CPU就会放行，如果发生中断，显然不能抛弃已提交指令，只能算作在提交指令上发生了中断，然后按照此规则执行该执行的，抛弃该抛弃的。

[回复](http://www.wowotech.net/kernel_synchronization/memory-barrier.html#comment-6473)

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
    
    - [Fix-Mapped Addresses](http://www.wowotech.net/memory_management/fixmap.html)
    - [Debian8 内核升级实验](http://www.wowotech.net/linux_application/debian8-upgrade-kernel.html)
    - [ARM64的启动过程之（六）：异常向量表的设定](http://www.wowotech.net/armv8a_arch/238.html)
    - [X-024-OHTHERS-在windows平台下使用libusb](http://www.wowotech.net/x_project/libusb_on_windows.html)
    - [进程切换分析（2）：TLB处理](http://www.wowotech.net/process_management/context-switch-tlb.html)
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