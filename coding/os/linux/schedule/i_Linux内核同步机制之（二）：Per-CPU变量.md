
作者：[linuxer](http://www.wowotech.net/author/3 "linuxer") 发布于：2014-10-16 11:17 分类：[内核同步机制](http://www.wowotech.net/sort/kernel_synchronization)

# 一、源由：为何引入Per-CPU变量？

## 1、lock bus带来的性能问题

在ARM平台上，ARMv6之前，SWP和SWPB指令被用来支持对shared memory的访问：

```cpp
SWP <Rt>, <Rt2>, \[<Rn>\]
```

Rn中保存了SWP指令要操作的内存地址，通过该指令可以将Rn指定的内存数据加载到Rt寄存器，同时将Rt2寄存器中的数值保存到Rn指定的内存中去。

我们在[原子操作](http://www.wowotech.net/linux_kenrel/atomic.html)那篇文档中描述的read-modify-write的问题本质上是一个保持对内存read和write访问的原子性的问题。也就是说对内存的读和写的访问不能被打断。对该问题的解决可以通过硬件、软件或者软硬件结合的方法来进行。早期的ARM CPU给出的方案就是依赖硬件：SWP这个汇编指令执行了一次读内存操作、一次写内存操作，但是从程序员的角度看，SWP这条指令就是原子的，读写之间不会被任何的异步事件打断。具体底层的硬件是如何做的呢？这时候，硬件会提供一个lock signal，在进行memory操作的时候设定lock信号，告诉总线这是一个不可被中断的内存访问，直到完成了SWP需要进行的两次内存访问之后再clear lock信号。

lock memory bus对多核系统的性能造成严重的影响（系统中其他的processor对那条被lock的memory bus的访问就被hold住了），如何解决这个问题？最好的锁机制就是不使用锁，因此解决这个问题可以使用釜底抽薪的方法，那就是不在系统中的多个processor之间共享数据，给每一个CPU分配一个不就OK了吗。

当然，随着技术的发展，在ARMv6之后的ARM CPU已经不推荐使用SWP这样的指令，而是提供了LDREX和STREX这样的指令。这种方法是使用软硬件结合的方法来解决原子操作问题，看起来代码比较复杂，但是系统的性能可以得到提升。其实，从硬件角度看，LDREX和STREX这样的指令也是采用了lock-free的做法。OK，由于不再lock bus，看起来Per-CPU变量存在的基础被打破了。不过考虑cache的操作，实际上它还是有意义的。

## 2、cache的影响

在[The Memory Hierarchy](http://www.wowotech.net/basic_subject/memory-hierarchy.html)文档中，我们已经了解了关于memory一些基础的知识，一些基础的内容，这里就不再重复了。我们假设一个多核系统中的cache如下：

![[Pasted image 20241023223553.png]]

每个CPU都有自己的L1 cache（包括data cache和instruction cache），所有的CPU共用一个L2 cache。L1、L2以及main memory的访问速度之间的差异都是非常大，最高的性能的情况下当然是L1 cache hit，这样就不需要访问下一阶memory来加载cache line。

我们首先看在多个CPU之间共享内存的情况。这种情况下，任何一个CPU如果修改了共享内存就会导致所有其他CPU的L1 cache上对应的cache line变成invalid（硬件完成）。虽然对性能造成影响，但是系统必须这么做，因为需要维持cache的同步。将一个共享memory变成Per-CPU memory本质上是一个耗费更多memory来解决performance的方法。当一个在多个CPU之间共享的变量变成每个CPU都有属于自己的一个私有的变量的时候，我们就不必考虑来自多个CPU上的并发，仅仅考虑本CPU上的并发就OK了。当然，还有一点要注意，那就是在访问Per-CPU变量的时候，不能调度，当然更准确的说法是该task不能调度到其他CPU上去。目前的内核的做法是在访问Per-CPU变量的时候disable preemptive，虽然没有能够完全避免使用锁的机制（disable preemptive也是一种锁的机制），但毫无疑问，这是一种代价比较小的锁。

# 二、接口

## 1、静态声明和定义Per-CPU变量的API如下表所示：

|   |   |
|---|---|
|声明和定义Per-CPU变量的API|描述|
|DECLARE_PER_CPU(type, name)  <br>DEFINE_PER_CPU(type, name)|普通的、没有特殊要求的per cpu变量定义接口函数。没有对齐的要求|
|DECLARE_PER_CPU_FIRST(type, name)  <br>DEFINE_PER_CPU_FIRST(type, name)|通过该API定义的per cpu变量位于整个per cpu相关section的最前面。|
|DECLARE_PER_CPU_SHARED_ALIGNED(type, name)  <br>DEFINE_PER_CPU_SHARED_ALIGNED(type, name)|通过该API定义的per cpu变量在SMP的情况下会对齐到L1 cache line ，对于UP，不需要对齐到cachine line|
|DECLARE_PER_CPU_ALIGNED(type, name)  <br>DEFINE_PER_CPU_ALIGNED(type, name)|无论SMP或者UP，都是需要对齐到L1 cache line|
|DECLARE_PER_CPU_PAGE_ALIGNED(type, name)  <br>DEFINE_PER_CPU_PAGE_ALIGNED(type, name)|为定义page aligned per cpu变量而设定的API接口|
|DECLARE_PER_CPU_READ_MOSTLY(type, name)  <br>DEFINE_PER_CPU_READ_MOSTLY(type, name)|通过该API定义的per cpu变量是read mostly的|

看到这样“丰富多彩”的Per-CPU变量的API，你是不是已经醉了。这些定义使用在不同的场合，主要的factor包括：

－该变量在section中的位置

－该变量的对齐方式

－该变量对SMP和UP的处理不同

－访问per cpu的形态

例如：如果你准备定义的per cpu变量是要求按照page对齐的，那么在定义该per cpu变量的时候需要使用DECLARE_PER_CPU_PAGE_ALIGNED。如果只要求在SMP的情况下对齐到cache line，那么使用DECLARE_PER_CPU_SHARED_ALIGNED来定义该per cpu变量。

## 2、访问静态声明和定义Per-CPU变量的API

静态定义的per cpu变量不能象普通变量那样进行访问，需要使用特定的接口函数，具体如下：

> get_cpu_var(var)
>
> put_cpu_var(var)

上面这两个接口函数已经内嵌了锁的机制（preempt disable），用户可以直接调用该接口进行本CPU上该变量副本的访问。如果用户确认当前的执行环境已经是preempt disable（例如持有spinlock），那么可以使用lock-free版本的Per-CPU变量的API:\_\_get_cpu_var。

## 3、动态分配Per-CPU变量的API如下表所示：

|   |   |
|---|---|
|动态分配和释放Per-CPU变量的API|描述|
|alloc_percpu(type)|分配类型是type的per cpu变量，返回per cpu变量的地址（注意：不是各个CPU上的副本）|
|void free_percpu(void \_\_percpu \*ptr)|释放ptr指向的per cpu变量空间|

## 4、访问动态分配Per-CPU变量的API如下表所示：

|   |   |
|---|---|
|访问Per-CPU变量的API|描述|
|get_cpu_ptr|这个接口是和访问静态Per-CPU变量的get_cpu_var接口是类似的，当然，这个接口是for 动态分配Per-CPU变量|
|put_cpu_ptr|同上|
|per_cpu_ptr(ptr, cpu)|根据per cpu变量的地址和cpu number，返回指定CPU number上该per cpu变量的地址|

# 三、实现

## 1、静态Per-CPU变量定义

我们以DEFINE_PER_CPU的实现为例子，描述linux kernel中如何实现静态Per-CPU变量定义。具体代码如下：

> #define DEFINE_PER_CPU(type, name)                    \\
> DEFINE_PER_CPU_SECTION(type, name, "")

type就是变量的类型，name是per cpu变量符号。DEFINE_PER_CPU_SECTION宏可以把一个per cpu变量放到指定的section中，具体代码如下：

> #define DEFINE_PER_CPU_SECTION(type, name, sec)                \\
> \_\_PCPU_ATTRS(sec) PER_CPU_DEF_ATTRIBUTES            \\－－－－－安排section\
> __typeof__(type) name－－－－－－－－－－－－－－－－－－－－－－定义变量

在这里具体arch specific的percpu代码中（arch/arm/include/asm/percpu.h）可以定义PER_CPU_DEF_ATTRIBUTES，以便控制该per cpu变量的属性，当然，如果arch specific的percpu代码不定义，那么在general arch-independent的代码中（include/asm-generic/percpu.h）会定义为空。这里可以顺便提一下Per-CPU变量的软件层次：

（1）arch-independent interface。在include/linux/percpu.h文件中，定义了内核其他模块要使用per cpu机制使用的接口API以及相关数据结构的定义。内核其他模块需要使用per cpu变量接口的时候需要include该头文件

（2）arch-general interface。在include/asm-generic/percpu.h文件中。如果所有的arch相关的定义都是一样的，那么就把它抽取出来，放到asm-generic目录下。毫无疑问，这个文件定义的接口和数据结构是硬件相关的，只不过软件抽象各个arch-specific的内容，形成一个arch general layer。一般来说，我们不需要直接include该头文件，include/linux/percpu.h会include该头文件。

（3）arch-specific。这是和硬件相关的接口，在arch/arm/include/asm/percpu.h，定义了ARM平台中，具体和per cpu相关的接口代码。

我们回到正题，看看\_\_PCPU_ATTRS的定义：

```cpp
define \_\_PCPU_ATTRS(sec)                        \\
\_\_percpu attribute((section(PER_CPU_BASE_SECTION sec)))    \\
PER_CPU_ATTRIBUTES
```

PER_CPU_BASE_SECTION 定义了基础的section name symbol，定义如下：

```cpp
ifndef PER_CPU_BASE_SECTION\
ifdef CONFIG_SMP\
define PER_CPU_BASE_SECTION ".data..percpu"\
else\
define PER_CPU_BASE_SECTION ".data"\
endif\
endif
```

虽然有各种各样的静态Per-CPU变量定义方法，但是都是类似的，只不过是放在不同的section中，属性不同而已，这里就不看其他的实现了，直接给出section的安排：

（1）普通per cpu变量的section安排

|   |   |   |
|---|---|---|
||SMP|UP|
|Build-in kernel|".data..percpu" section|".data" section|
|defined in module|".data..percpu" section|".data" section|

（2）first per cpu变量的section安排

|   |   |   |
|---|---|---|
||SMP|UP|
|Build-in kernel|".data..percpu..first" section|".data" section|
|defined in module|".data..percpu..first" section|".data" section|

（3）SMP shared aligned per cpu变量的section安排

|   |   |   |
|---|---|---|
||SMP|UP|
|Build-in kernel|".data..percpu..shared_aligned" section|".data" section|
|defined in module|".data..percpu" section|".data" section|

（4）aligned per cpu变量的section安排

|   |   |   |
|---|---|---|
||SMP|UP|
|Build-in kernel|".data..percpu..shared_aligned" section|".data..shared_aligned" section|
|defined in module|".data..percpu" section|".data..shared_aligned" section|

（5）page aligned per cpu变量的section安排

|   |   |   |
|---|---|---|
||SMP|UP|
|Build-in kernel|".data..percpu..page_aligned" section|".data..page_aligned" section|
|defined in module|".data..percpu..page_aligned" section|".data..page_aligned" section|

（6）read mostly per cpu变量的section安排

|   |   |   |
|---|---|---|
||SMP|UP|
|Build-in kernel|".data..percpu..readmostly" section|".data..readmostly" section|
|defined in module|".data..percpu..readmostly" section|".data..readmostly" section|

了解了静态定义Per-CPU变量的实现，但是为何要引入这么多的section呢？对于kernel中的普通变量，经过了编译和链接后，会被放置到.data或者.bss段，系统在初始化的时候会准备好一切（例如clear bss），由于per cpu变量的特殊性，内核将这些变量放置到了其他的section，位于kernel address space中\_\_per_cpu_start和\_\_per_cpu_end之间，我们称之Per-CPU变量的原始变量（我也想不出什么好词了）。

只有Per-CPU变量的原始变量还是不够的，必须为每一个CPU建立一个副本，怎么建？直接静态定义一个NR_CPUS的数组？NR_CPUS定义了系统支持的最大的processor的个数，并不是实际中系统processor的数目，这样的定义非常浪费内存。此外，静态定义的数据在内存中连续，对于UMA系统而言是OK的，对于NUMA系统，每个CPU上的Per-CPU变量的副本应该位于它访问最快的那段memory上，也就是说Per-CPU变量的各个CPU副本可能是散布在整个内存地址空间的，而这些空间之间是有空洞的。本质上，副本per cpu内存的分配归属于内存管理子系统，因此，分配per cpu变量副本的内存本文不会详述，大致的思路如下：

[![percpu](http://www.wowotech.net/content/uploadfile/201410/8d9ace9500fce839a185e4567a9c3de420141016031726.gif "percpu")](http://www.wowotech.net/content/uploadfile/201410/1fe906ab8a54768c668f4ea25eed3ab620141016031724.gif)

内存管理子系统会根据当前的内存配置为每一个CPU分配一大块memory，对于UMA，这个memory也是位于main memory，对于NUMA，有可能是分配最靠近该CPU的memory（也就是说该cpu访问这段内存最快），但无论如何，这些都是内存管理子系统需要考虑的。无论静态还是动态per cpu变量的分配，其机制都是一样的，只不过，对于静态per cpu变量，需要在系统初始化的时候，对应per cpu section，预先动态分配一个同样size的per cpu chunk。在vmlinux.lds.h文件中，定义了percpu section的排列情况：

```cpp
define PERCPU_INPUT(cacheline)                        \\
VMLINUX_SYMBOL(\_\_per_cpu_start) = .;                \\
\*(.data..percpu..first)                        \\
. = ALIGN(PAGE_SIZE);                        \\
\*(.data..percpu..page_aligned)                    \\
. = ALIGN(cacheline);                        \\
\*(.data..percpu..readmostly)                    \\
. = ALIGN(cacheline);                        \\
\*(.data..percpu)                        \\
\*(.data..percpu..shared_aligned)                \\
VMLINUX_SYMBOL(\_\_per_cpu_end) = .;
```

对于build in内核的那些per cpu变量，必然位于\_\_per_cpu_start和\_\_per_cpu_end之间的per cpu section。在系统初始化的时候（setup_per_cpu_areas），分配per cpu memory chunk，并将per cpu section copy到每一个chunk中。

2、访问静态定义的per cpu变量

代码如下：

> #define get_cpu_var(var) (\*({                \\
> preempt_disable();                \\
> &\_\_get_cpu_var(var); }))

再看到get_cpu_var和\_\_get_cpu_var这两个符号，相信广大人民群众已经相当的熟悉，一个持有锁的版本，一个lock-free的版本。为防止当前task由于抢占而调度到其他的CPU上，在访问per cpu memory的时候都需要使用preempt_disable这样的锁的机制。我们来看\_\_get_cpu_var：

> #define \_\_get_cpu_var(var) (\*this_cpu_ptr(&(var)))
>
> #define this_cpu_ptr(ptr) \_\_this_cpu_ptr(ptr)

对于ARM平台，我们没有定义\_\_this_cpu_ptr，因此采用asm-general版本的：

> #define \_\_this_cpu_ptr(ptr) SHIFT_PERCPU_PTR(ptr, \_\_my_cpu_offset)

SHIFT_PERCPU_PTR这个宏定义从字面上就可以看出它是可以从原始的per cpu变量的地址，通过简单的变换（SHIFT）转成实际的per cpu变量副本的地址。实际上，per cpu内存管理模块可以保证原始的per cpu变量的地址和各个CPU上的per cpu变量副本的地址有简单的线性关系（就是一个固定的offset）。\_\_my_cpu_offset这个宏定义就是和offset相关的，如果arch specific没有定义，那么可以采用asm general版本的，如下：

> #define \_\_my_cpu_offset per_cpu_offset(raw_smp_processor_id())

raw_smp_processor_id可以获取本CPU的ID，如果没有arch specific没有定义\_\_per_cpu_offset这个宏，那么offset保存在\_\_per_cpu_offset的数组中（下面只是数组声明，具体定义在mm/percpu.c文件中），如下：

> #ifndef \_\_per_cpu_offset\
> extern unsigned long \_\_per_cpu_offset\[NR_CPUS\];
>
> #define per_cpu_offset(x) (\_\_per_cpu_offset\[x\])\
> #endif

对于ARMV6K和ARMv7版本，offset保存在TPIDRPRW寄存器中，这样是为了提升系统性能。

## 3、动态分配per cpu变量

这部分内容留给内存管理子系统吧。

_原创文章，转发请注明出处。蜗窝科技_

[http://www.wowotech.net/linux_kenrel/per-cpu.html](http://www.wowotech.net/linux_kenrel/per-cpu.html)

标签: [内核同步](http://www.wowotech.net/tag/%E5%86%85%E6%A0%B8%E5%90%8C%E6%AD%A5) [Per-CPU变量](http://www.wowotech.net/tag/Per-CPU%E5%8F%98%E9%87%8F)

---

« [Linux common clock framework(1)\_概述](http://www.wowotech.net/pm_subsystem/clk_overview.html) | [Linux内核同步机制之（一）：原子操作](http://www.wowotech.net/kernel_synchronization/atomic.html)»

**评论：**

**[bird](http://q2621473170/)**\
2019-08-26 16:02

是很多人在考虑同步的事情, 可能是因为例子里,提到了lock bus 这些.  其实本质上,\
Per-CPU是基于空间换时间的方法, 让每个CPU都有自己的私有数据段(放在L1中),并将一些变量私有化到\
每个CPU的私有数据段中. 单个CPU在访问自己的私有数据段时, 不需要考虑其他CPU之间的竞争问题,也不存在同步的问题.  注意只有在该变量在各个CPU上逻辑独立时才可使用.

[回复](http://www.wowotech.net/kernel_synchronization/per-cpu.html#comment-7615)

**Magicmanoooo**\
2021-10-20 09:58

@bird：大佬，可以这么理解吗：在没有per-cpu之前，多个cpu之间即使对于同一变量没有逻辑依赖，但依旧只声明了一份，全局使用，这势必会造成竞争。现在有了per-cpu，则实现了多cpu之间的对同一变量的解耦，我直接就在每个cpu上都创建一份该变量，大家各自使用，互不干扰？

[回复](http://www.wowotech.net/kernel_synchronization/per-cpu.html#comment-8338)

**立志学linux**\
2023-05-26 16:19

@Magicmanoooo：既然这样，那为什么不干脆用好几个不同的变量呢？还是说percpu变量就是等于定义了好几个不同的变量？

[回复](http://www.wowotech.net/kernel_synchronization/per-cpu.html#comment-8785)

**duckwu**\
2024-05-26 21:35

@立志学linux：我觉得percpu变量就是等于定义了好几个不同的变量但是比定义多个变量更好。假设一个进程在运行过程中需要某个变量，该变量在各个CPU上逻辑独立，但是程序员在编码时是无法知道该进程运行在哪个cpu上，也就无法为其指定对应的变量。想反，让cpu自己去选择对应的变量即per-cpu变量就可以解决这个问题。

[回复](http://www.wowotech.net/kernel_synchronization/per-cpu.html#comment-8902)

**xiaoai**\
2019-05-27 17:51

Hi linux:\
当然，还有一点要注意，那就是在访问Per-CPU变量的时候，不能调度，当然更准确的说法是该task不能调度到其他CPU上去。

我不太理解这里，即使调度到其他CPU上面去也没关系啊，反正访问操作的也是其他CPU上的值。我理解是处理好本CPU上的同步就好了，关闭调度是为了避免该CPU上其他可能存在的并发调度。

[回复](http://www.wowotech.net/kernel_synchronization/per-cpu.html#comment-7442)

**[bird](http://q2621473170/)**\
2019-08-26 15:59

@xiaoai：因为per_cpu的原则是各个CPU只访问属于自己的数据, 故在实际访问时要关闭抢占. 即当A CPU在访问自己的私有变量时,该task不能被调度到其他CPU去, 不然就造成了B在访问A CPU的私有变量.

[回复](http://www.wowotech.net/kernel_synchronization/per-cpu.html#comment-7614)

**speedan**\
2017-03-08 11:53

linuxer：你好，请教个问题：\
关于per-cpu变量的接口，只说get_cpu_var()这类接口会处理抢占的问题，但没说明中断互斥的问题。\
比如有个per-cpu变量也会在中断处理程序中用到，那在普通进程上下文中使用这个per-cpu变量时，是不是在调用get_cpu_var()之前也应该先通过spin_lock()或关本地中断的方式来进行互斥，防止和中断处理程序的竞争问题?

[回复](http://www.wowotech.net/kernel_synchronization/per-cpu.html#comment-5292)

**speedan**\
2017-03-08 13:43

@speedan：另外，下边的用法中，get_cpu()和put_cpu()应该是在for循环之类还是之外，望请指点，谢谢！\
int i, curcpu;\
curcpu = get_cpu(); // 禁止抢占\
for_each_possible_cpu(i) {\
sec_info = &per_cpu(my_birthday, i).sec_info; // per_cpu()不会禁止抢占\
sec_info->sec++;\
}\
put_cpu(); // 重新开启抢占

[回复](http://www.wowotech.net/kernel_synchronization/per-cpu.html#comment-5293)

**[linuxer](http://www.wowotech.net/)**\
2017-03-08 23:01

@speedan：你这是什么代码？你能不能描述一下临界区中的数据访问情况是怎样的？

[回复](http://www.wowotech.net/kernel_synchronization/per-cpu.html#comment-5296)

**speedan**\
2017-03-09 12:22

@linuxer：不好意思，我之前描述的问题不准确，修改如下：\
我的理解，使用per-cpu变量的前提是每个CPU只能访问此变量对应当前CPU的部分，这样就不存在多个CPU并发访问的问题。\
但对于需要遍历per-cpu变量的场景，意味着在当前CPU需要访问其他所有CPU对应的部分，这样使用per-cpu变量的前提就不再成立，此时似乎陷入了自我矛盾之中，应该怎么处理？\
我看内核的代码在这种情况下好像没有做过多的保护，所以挺奇怪的。

// 下边的代码是我设想的一个遍历的例子\
int i, curcpu, sum;\
static DEFINE_PER_CPU(int, arr) = {0};\
for_each_possible_cpu(i) { // 遍历per-cpu变量时怎么处理多CPU并发访问的问题？\
sum += per_cpu(arr, i);\
per_cpu(arr, i)++;\
}

[回复](http://www.wowotech.net/kernel_synchronization/per-cpu.html#comment-5298)

**[linuxer](http://www.wowotech.net/)**\
2017-03-09 17:43

@speedan：其实这个问题仍然是一个基本的临界区保护的问题，per cpu变量并不会让临界区的保护更复杂，只不过是把数据访问分散到各个cpu上而已。

我们来举一个实际的例子好了：有一个per cpu变量A，该变量表示某个事件的count，该变量不会在中断上下文中访问，只会在进程上下文访问。访问的场景有两个：\
1、各个cpu上的写进程会对A累加操作，记录各个cpu上事件发生的次数\
2、读进程遍历全部cpu，对各个cpu上的A累加，得到一个globa count值

在这样的数据访问情况下，读进程需要锁保护吗？如果没有锁保护，那么有可能写进程会插入，但是这又有什么关系？累加后的那个globa count值有那么高的精度要求吗？

因此，结论是： 遍历per-cpu变量时需不需要保护，用什么锁机制保护是和per cpu变量的临界区数据访问情况相关的，不同的情况需要不同的分析。

[回复](http://www.wowotech.net/kernel_synchronization/per-cpu.html#comment-5302)

**speedan**\
2017-03-09 22:46

@linuxer：你说的这种原子变量的情况我也设想过，只要精度不要求，确实没关系。

我只是看到内核中也有把per cpu变量用到了结构体中，这样就有可能出现结构体部分被修改的情况。\
不过如你所说，对于这种情况，如果意识到会出现竞争问题就不应该使用per cpu变量了吧……

[回复](http://www.wowotech.net/kernel_synchronization/per-cpu.html#comment-5304)

**smallfish**\
2022-07-26 14:26

@linuxer：大佬，你说利用 globa count来统计各个cpu 本地的累加值，这个好理解，但是我的操作不是++或者--， 我要对这个count 赋值，比如说count=0，  那这个是不是还得进行同步。

[回复](http://www.wowotech.net/kernel_synchronization/per-cpu.html#comment-8655)

**[linuxer](http://www.wowotech.net/)**\
2017-03-08 22:46

@speedan：per-cpu变量只是能够保护来自不同cpu的并发访问，并不能保护同一个cpu上，进程上下文和中断上下文中的并发，这时候，往往需要其他的同步原语。

[回复](http://www.wowotech.net/kernel_synchronization/per-cpu.html#comment-5295)

**speedan**\
2017-03-09 12:25

@linuxer：你说得对，我理解得不够本质，赞！！！

[回复](http://www.wowotech.net/kernel_synchronization/per-cpu.html#comment-5299)

**风华正茂**\
2017-01-19 14:48

请问一下，既然每个cpu都有自己独立的变量，那么他们是怎么做到同步的呢？

[回复](http://www.wowotech.net/kernel_synchronization/per-cpu.html#comment-5166)

**[wowo](http://www.wowotech.net/)**\
2017-01-20 08:37

@风华正茂：不需要同步（可以扒扒后面的评论，有讨论这个事情）。

[回复](http://www.wowotech.net/kernel_synchronization/per-cpu.html#comment-5170)

**风华正茂**\
2017-01-20 10:34

@wowo：在QQ群里面讨论过了，最后感觉per_cpu的机制不像是为同步准备的，更像是一种新的思路

[回复](http://www.wowotech.net/kernel_synchronization/per-cpu.html#comment-5173)

**风华正茂**\
2017-01-20 10:43

@wowo：有没有关于 per_cpu的 demo？

[回复](http://www.wowotech.net/kernel_synchronization/per-cpu.html#comment-5174)

**hello**\
2017-01-20 10:31

@风华正茂：不需要同步，各个CPU都是看到自己的私有的数据，例如对于percpu变量X，CPUa只能看到自己的那个Xa，其他的CPU不会访问Xa，因此不需要同步。

[回复](http://www.wowotech.net/kernel_synchronization/per-cpu.html#comment-5172)

**lamaboy**\
2016-08-20 11:44

你好

我看了Per-CPU变量 还是没有理出Per-CPU变量 的使用场景，，\
请指教下了   谢谢了

[回复](http://www.wowotech.net/kernel_synchronization/per-cpu.html#comment-4428)

**[linuxer](http://www.wowotech.net/)**\
2016-08-22 18:56

@lamaboy：千言万语汇成一句话：性能，per-cpu变量就是为了更好的性能而已。或者更通俗的将：就是用空间换时间的一种优化而已。

[回复](http://www.wowotech.net/kernel_synchronization/per-cpu.html#comment-4436)

**[puppypyb](http://www.wowotech.net/)**\
2015-04-29 10:07

Quote:"当然，还有一点要注意，那就是在访问Per-CPU变量的时候，不能调度，当然更准确的说法是该task不能调度到其他CPU上去。目前的内核的做法是在访问Per-CPU变量的时候disable preemptive"

谈一点感想，有不对的地方请指出：\
引入per-cpu变量的原因就是文中所说的，每个cpu都在自己的变量副本上工作，这样每次读写就可以避免锁开销，上下文切换和cache等一系列问题。假设当前task运行在其中一个core cpu0上且获得了var在该cpu上的本地副本，然后该task在操作该副本的过程中因被抢占而转移到另外一个core cpu1上运行，那么接下来cpu1将会继续操作该变量副本（属于cpu0），违反了“每个cpu都工作在自己副本上”这样一个前提，该机制就乱套了。

上面是我所认为的disable preemptive的初衷，就是在操作per-cpu副本的过程中防止task调度到其他cpu上去。但同样的get_cpu_var(var)的时候也disable了本CPU的调度。我想了下，在这样的场景下disable本cpu的调度好像没有必要，我理解为disable preemptive的副作用。

[回复](http://www.wowotech.net/kernel_synchronization/per-cpu.html#comment-1629)

**scnutiger**\
2016-03-23 19:34

@puppypyb：我看到一个代码似乎per-CPU在访问的时候没有禁止抢占。\
内核里面对vfsmount这个结构体做计数器增加的时候\
struct vfsmount \*mntget(struct vfsmount \*mnt)\
直接就对per-cpu的mnt_count值做this_cpu_add(mnt->mnt_pcp->mnt_count, n);操作了\
最后统计vfsmount结构体的count值的时候\
在读写锁的保护下，把per-cpu的count值都加到一起了，也没有后面comment里面的阈值的感觉\
for_each_possible_cpu(cpu) {\
count += per_cpu_ptr(mnt->mnt_pcp, cpu)->mnt_count;\
}

[回复](http://www.wowotech.net/kernel_synchronization/per-cpu.html#comment-3717)

**[郭健](http://www.wowotech.net/)**\
2016-03-24 10:10

@scnutiger：this_cpu_add在4.1.10内核中的实现如下：

#define this_cpu_add(pcp, val)        \__pcpu_size_call(this_cpu_add_, pcp, val)

如果你愿意一层层的看下去，会发现其实this_cpu_add也会内嵌禁止抢占的代码的。

[回复](http://www.wowotech.net/kernel_synchronization/per-cpu.html#comment-3722)

**[郭健](http://www.wowotech.net/)**\
2016-03-24 10:05

@puppypyb：你说的很对，只不过最后一段话我不是非常理解。我是这么看的，用户在使用percpu变量的时候调用场景是这样的：\
－－－－－－－－－－－－－－－－\
get_cpu_var

访问per-cpu变量

put_cpu_var\
－－－－－－－－－－－－－－－－\
用户并不需要显式的调用禁止抢占，per-cpu API已经封装好了，当然，一个好的模块实现必然是这样的。

[回复](http://www.wowotech.net/kernel_synchronization/per-cpu.html#comment-3721)

**[xiaoxiao](http://www.wowotech.net/)**\
2015-04-22 09:37

Hi\
我一直不明白per-CPU的意义。它是为了解决CPU之间同步问题引入的，但到底是怎么解决的呢？\
每个CPU都有自己的副本，也没有讲副本和原始变量之间的同步。如果不需要同步，那每个CPU的副本不相当于四个变量了？如果需要同步，那不跟设计per-cpu之前一样的问题？

是否是这样：\
每个CPU都可以对自己的副本进行操作，比如+1，-3之类的操作，但是不会实时同步到原始变量中，但是有个阈值，比如30，当你修改超过30后，则必须同步修改到原始变量中。但是这样的话，原始变量中的值是个近似值，不是准确值。（什么情况下可以使用计数器的近似值而不需要准确值呢？）

[回复](http://www.wowotech.net/kernel_synchronization/per-cpu.html#comment-1543)

**[linuxer](http://www.wowotech.net/)**\
2015-04-22 12:46

@xiaoxiao：毫无疑问per-CPU变量之间不需要同步，否则per cpu不就没有意义了吗？每个CPU上的执行的代码就是访问属于本cpu的那个变量。原始变量仅仅是作为一个initial data区域，当per cpu变量的副本分配之后，将原始变量的initial data copy到各自per cpu的变量副本中。

[回复](http://www.wowotech.net/kernel_synchronization/per-cpu.html#comment-1549)

**[xiaoxiao](http://www.wowotech.net/)**\
2015-04-22 17:44

@linuxer：Hi linuxer\
多谢回复。确实per-cpu之间是不需要同步的，但是原始变量和副本之间还是有同步的需要的（有spinlock）。刚找到在professional linux kernel architecture书中关于per-cpu的说明还算详细（5.2.9章节），但是还是不甚明了。副本中确实存的是修改，不是修改后的值。在需要近似值的地方，直接取原始变量percpu_counter_read()就可以了，但是需要准确值的话，需要调用percpu_counter_sum()来获取。只是什么情况下会只需要近似值就OK了？\
真觉得这些设计真的是巧妙啊～

[回复](http://www.wowotech.net/kernel_synchronization/per-cpu.html#comment-1556)

**[jinxin](http://www.linux2web.net/)**\
2015-12-23 10:32

@xiaoxiao：你说的近似值是个什么概念噢

[回复](http://www.wowotech.net/kernel_synchronization/per-cpu.html#comment-3273)

**[forion](http://www.wowotech.net/)**\
2014-10-22 10:06

hi linuxer\
请教一个问题，如果我一个semaphore，初始化两次会有什么后果？\
比如\
static struct semaphore seam;

sema_init(&seam, 0);

sema_init(&seam, 0);

这样然后比如我

down(&seam);\
这样会有什么后果呢？？

[回复](http://www.wowotech.net/kernel_synchronization/per-cpu.html#comment-545)

**[forion](http://www.wowotech.net/)**\
2014-10-22 10:31

@forion：如果我用kthread_run();创建两个名字一模一样的线程呢？换句话说，kthread_run();跑两次，里面的参数都一样。这样又是一种怎样的情况呢？

[回复](http://www.wowotech.net/kernel_synchronization/per-cpu.html#comment-546)

**[linuxer](http://www.wowotech.net/)**\
2014-10-22 12:12

@forion：在3.14内核中已经没有kthread_run()这个接口了。我估计这是旧内核中创建内核线程的接口函数。如果调用kthread_run两次，那么就创建两个一样的线程了，当然，其pid是不一样的。这是我直观的感觉，当然要沿着你那个版本的内核代码看一看，是否有一些限制

[回复](http://www.wowotech.net/kernel_synchronization/per-cpu.html#comment-548)

**[linuxer](http://www.wowotech.net/)**\
2014-10-22 12:03

@forion：如果你保证你的执行序列是两次初始化，然后调用down，在没有配置CONFIG_LOCKDEP的时候应该是没有问题的。\
static inline void sema_init(struct semaphore \*sem, int val)\
{\
static struct lock_class_key \_\_key;\
\*sem = (struct semaphore) \_\_SEMAPHORE_INITIALIZER(\*sem, val);\
lockdep_init_map(&sem->lock.dep_map, "semaphore->lock", &\_\_key, 0);\
}\
sema_init如果连续执行两次，应该是第二次覆盖第一次初始化的结果。lockdep_init_map这个函数在没有配置CONFIG_LOCKDEP的时候没有实际的代码逻辑。

[回复](http://www.wowotech.net/kernel_synchronization/per-cpu.html#comment-547)

**[forion](http://www.wowotech.net/)**\
2014-10-22 13:27

@linuxer：那如果定义了CONFIG_LOCKDEP的时候呢？我也去review一下code。\
我现在的code里面是定义了CONFIG_LOCKDEP。我现在是4个core的cpu\
我现在是遇到一个这样的问题，一个工程师在他的代码的probe函数里面写了一个sema的init，以及kthread_run,但是这个probe会走两次，所以sema 以及线程会初始化两次。然后up的操作是在中断处理里面。很多driver都是这样的一个逻辑。\
然后发现经常死锁在线程里面down（）；sema的时候，具体就是down（）操作里面的raw_spin_lock_irqsave，但是具体为什么会卡在这里，我还没有从代码上面找到原因。因为我没有看到其他的core拿到这把锁。表面上看，肯定是这把锁，被别人拿了，所以想请教下你，整理一下逻辑。

[回复](http://www.wowotech.net/kernel_synchronization/per-cpu.html#comment-549)

**[linuxer](http://www.wowotech.net/)**\
2014-10-22 14:36

@forion：probe函数会走两次是设计如此吗？我听你的描述觉得这个driver怪怪的。\
一般driver的逻辑是使用等待队列的，初始化函数中init 等待队列，创建线程。\
在线程中进行处理，当需要等待硬件事件的时候，wait在等待队列上，在中断handler中唤醒等待队列的task。

如果probe两次是对的，是否是有两个device需要初始化？也就是说，每次把device加入bus的时候，都会调用该driver的probe函数？如果是这样，那么需要为每一个device建立一个semaphore而不是共用，一般而言，会定义一个xxx_device的数据结构，这个数据结构中有一个semaphore成员，用来保护对该device实例的访问

我感觉不是LOCKDEP的问题，还是semaphore使用的问题

[回复](http://www.wowotech.net/kernel_synchronization/per-cpu.html#comment-550)

**[forion](http://www.wowotech.net/)**\
2014-10-22 15:44

@linuxer：首先我自己写的driver确实不多，惭愧惭愧，我都是用到的时候去看已有的差不多相似的driver，去copy。\
你说的等待队列，应该是一个通用的逻辑。现在这个drvier的逻辑是直接kthread_run起一个线程，然后线程刚开始的地方，down一个sema然后睡下去，在中断handler中up sema然后唤醒这个线程

现在这个driver里面确实是有两个device，所以probe两次，所以init了两个sema，以及kthread_run起了两个线程。

关于等待队列，这个你能举例一个common的driver，我去学习一下吗？

[回复](http://www.wowotech.net/kernel_synchronization/per-cpu.html#comment-551)

**[linuxer](http://www.wowotech.net/)**\
2014-10-22 16:21

@forion：你可以去看看Linux Device Driver的第六章，blocking I/O那个小节。那里描述的非常详细了。

[回复](http://www.wowotech.net/kernel_synchronization/per-cpu.html#comment-552)

**[linuxer](http://www.wowotech.net/)**\
2014-10-16 15:49

SHIFT_PERCPU_PTR宏定义的含义很简单，不过具体实现还是有些模糊之处，\_\_verify_pcpu_ptr和RELOC_HIDE有谁知道其工作原理吗？

[回复](http://www.wowotech.net/kernel_synchronization/per-cpu.html#comment-535)

**[forion](http://www.wowotech.net/)**\
2014-10-16 14:38

这样就需要访问下一阶memory来加载cache line。\
这里缺了个不

[回复](http://www.wowotech.net/kernel_synchronization/per-cpu.html#comment-533)

**[linuxer](http://www.wowotech.net/)**\
2014-10-16 15:40

@forion：已经修改了，多谢多谢！

[回复](http://www.wowotech.net/kernel_synchronization/per-cpu.html#comment-534)

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

  - [Linux内核同步机制之（九）：Queued spinlock](http://www.wowotech.net/kernel_synchronization/queued_spinlock.html)
  - [CPU 多核指令 —— WFE 原理](http://www.wowotech.net/armv8a_arch/499.html)
  - [fixmap addresses原理](http://www.wowotech.net/memory_management/440.html)
  - [Linux 2.5.43版本的RCU实现（废弃）](http://www.wowotech.net/kernel_synchronization/Linux-2-5-43-RCU.html)
  - [Linux cpuidle framework(3)\_ARM64 generic CPU idle driver](http://www.wowotech.net/pm_subsystem/cpuidle_arm64.html)

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
