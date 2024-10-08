作者：[linuxer](http://www.wowotech.net/author/3 "linuxer") 发布于：2014-10-10 17:56 分类：[内核同步机制](http://www.wowotech.net/sort/kernel_synchronization)

一、源由

我们的程序逻辑经常遇到这样的操作序列：

1、读一个位于memory中的变量的值到寄存器中

2、修改该变量的值（也就是修改寄存器中的值）

3、将寄存器中的数值写回memory中的变量值

如果这个操作序列是串行化的操作（在一个thread中串行执行），那么一切OK，然而，世界总是不能如你所愿。在多CPU体系结构中，运行在两个CPU上的两个内核控制路径同时并行执行上面操作序列，有可能发生下面的场景：

|   |   |
|---|---|
|CPU1上的操作|CPU2上的操作|
|读操作||
||读操作|
|修改|修改|
|写操作||
||写操作|

多个CPUs和memory chip是通过总线互联的，在任意时刻，只能有一个总线master设备（例如CPU、DMA controller）访问该Slave设备（在这个场景中，slave设备是RAM chip）。因此，来自两个CPU上的读memory操作被串行化执行，分别获得了同样的旧值。完成修改后，两个CPU都想进行写操作，把修改的值写回到memory。但是，硬件arbiter的限制使得CPU的写回必须是串行化的，因此CPU1首先获得了访问权，进行写回动作，随后，CPU2完成写回动作。在这种情况下，CPU1的对memory的修改被CPU2的操作覆盖了，因此执行结果是错误的。

不仅是多CPU，在单CPU上也会由于有多个内核控制路径的交错而导致上面描述的错误。一个具体的例子如下：

|   |   |
|---|---|
|系统调用的控制路径|中断handler控制路径|
|读操作||
||读操作|
||修改|
||写操作|
|修改||
|写操作||

系统调用的控制路径上，完成读操作后，硬件触发中断，开始执行中断handler。这种场景下，中断handler控制路径的写回的操作被系统调用控制路径上的写回覆盖了，结果也是错误的。

二、对策

对于那些有多个内核控制路径进行read-modify-write的变量，内核提供了一个特殊的类型atomic_t，具体定义如下：

> typedef struct {  
>     int counter;  
> } atomic_t;

从上面的定义来看，atomic_t实际上就是一个int类型的counter，不过定义这样特殊的类型atomic_t是有其思考的：内核定义了若干atomic_xxx的接口API函数，这些函数只会接收atomic_t类型的参数。这样可以确保atomic_xxx的接口函数只会操作atomic_t类型的数据。同样的，如果你定义了atomic_t类型的变量（你期望用atomic_xxx的接口API函数操作它），这些变量也不会被那些普通的、非原子变量操作的API函数接受。

具体的接口API函数整理如下：

|   |   |
|---|---|
|接口函数|描述|
|static inline void atomic_add(int i, atomic_t *v)|给一个原子变量v增加i|
|static inline int atomic_add_return(int i, atomic_t *v)|同上，只不过将变量v的最新值返回|
|static inline void atomic_sub(int i, atomic_t *v)|给一个原子变量v减去i|
|static inline int atomic_sub_return(int i, atomic_t *v)|同上，只不过将变量v的最新值返回|
|static inline int atomic_cmpxchg(atomic_t *ptr, int old, int new)|比较old和原子变量ptr中的值，如果相等，那么就把new值赋给原子变量。  <br>返回旧的原子变量ptr中的值|
|atomic_read|获取原子变量的值|
|atomic_set|设定原子变量的值|
|atomic_inc(v)|原子变量的值加一|
|atomic_inc_return(v)|同上，只不过将变量v的最新值返回|
|atomic_dec(v)|原子变量的值减去一|
|atomic_dec_return(v)|同上，只不过将变量v的最新值返回|
|atomic_sub_and_test(i, v)|给一个原子变量v减去i，并判断变量v的最新值是否等于0|
|atomic_add_negative(i,v)|给一个原子变量v增加i，并判断变量v的最新值是否是负数|
|static inline int atomic_add_unless(atomic_t *v, int a, int u)|只要原子变量v不等于u，那么就执行原子变量v加a的操作。  <br>如果v不等于u，返回非0值，否则返回0值|

三、ARM中的实现

我们以atomic_add为例，描述linux kernel中原子操作的具体代码实现细节：

> #if __LINUX_ARM_ARCH__ >= 6 －－－－－－－－－－－－－－－－－－－－－－（1）  
> static inline void atomic_add(int i, atomic_t *v)  
> {  
>     unsigned long tmp;  
>     int result;
> 
>     prefetchw(&v->counter); －－－－－－－－－－－－－－－－－－－－－－－－－（2）  
>     __asm__ __volatile__("@ atomic_add\n" －－－－－－－－－－－－－－－－－－（3）  
> "1:    ldrex    %0, [%3]\n" －－－－－－－－－－－－－－－－－－－－－－－－－－（4）  
> "    add    %0, %0, %4\n" －－－－－－－－－－－－－－－－－－－－－－－－－－（5）  
> "    strex    %1, %0, [%3]\n" －－－－－－－－－－－－－－－－－－－－－－－－－（6）  
> "    teq    %1, #0\n" －－－－－－－－－－－－－－－－－－－－－－－－－－－－－（7）  
> "    bne    1b"  
>     : "=&r" (result), "=&r" (tmp), "+Qo" (v->counter) －－－对应％0，％1，％2  
>     : "r" (&v->counter), "Ir" (i) －－－－－－－－－－－－－对应％3，％4  
>     : "cc");  
> }
> 
> #else
> 
> #ifdef CONFIG_SMP  
> #error SMP not supported on pre-ARMv6 CPUs  
> #endif
> 
> static inline int atomic_add_return(int i, atomic_t *v)  
> {  
>     unsigned long flags;  
>     int val;
> 
>     raw_local_irq_save(flags);  
>     val = v->counter;  
>     v->counter = val += i;  
>     raw_local_irq_restore(flags);
> 
>     return val;  
> }  
> #define atomic_add(i, v)    (void) atomic_add_return(i, v)
> 
> #endif

（1）ARMv6之前的CPU并不支持SMP，之后的ARM架构都是支持SMP的（例如我们熟悉的ARMv7-A）。因此，对于ARM处理，其原子操作分成了两个阵营，一个是支持SMP的ARMv6之后的CPU，另外一个就是ARMv6之前的，只有单核架构的CPU。对于UP，原子操作就是通过关闭CPU中断来完成的。

（2）这里的代码和preloading cache相关。在strex指令之前将要操作的memory内容加载到cache中可以显著提高性能。

（3）为了完整性，我还是重复一下汇编嵌入c代码的语法：嵌入式汇编的语法格式是：asm(code : output operand list : input operand list : clobber list)。output operand list 和 input operand list是c代码和嵌入式汇编代码的接口，clobber list描述了汇编代码对寄存器的修改情况。为何要有clober list？我们的c代码是gcc来处理的，当遇到嵌入汇编代码的时候，gcc会将这些嵌入式汇编的文本送给gas进行后续处理。这样，gcc需要了解嵌入汇编代码对寄存器的修改情况，否则有可能会造成大麻烦。例如：gcc对c代码进行处理，将某些变量值保存在寄存器中，如果嵌入汇编修改了该寄存器的值，又没有通知gcc的话，那么，gcc会以为寄存器中仍然保存了之前的变量值，因此不会重新加载该变量到寄存器，而是直接使用这个被嵌入式汇编修改的寄存器，这时候，我们唯一能做的就是静静的等待程序的崩溃。还好，在output operand list 和 input operand list中涉及的寄存器都不需要体现在clobber list中（gcc分配了这些寄存器，当然知道嵌入汇编代码会修改其内容），因此，大部分的嵌入式汇编的clobber list都是空的，或者只有一个cc，通知gcc，嵌入式汇编代码更新了condition code register。

大家对着上面的code就可以分开各段内容了。@符号标识该行是注释。

这里的__volatile__主要是用来防止编译器优化的。也就是说，在编译该c代码的时候，如果使用优化选项（-O）进行编译，对于那些没有声明__volatile__的嵌入式汇编，编译器有可能会对嵌入c代码的汇编进行优化，编译的结果可能不是原来你撰写的汇编代码，但是如果你的嵌入式汇编使用__asm__ __volatile__(嵌入式汇编)的语法格式，那么也就是告诉编译器，不要随便动我的嵌入汇编代码哦。

  

（4）我们先看ldrex和strex这两条汇编指令的使用方法。ldr和str这两条指令大家都是非常的熟悉了，后缀的ex表示Exclusive，是ARMv7提供的为了实现同步的汇编指令。

  
LDREX  <Rt>, [<Rn>]  
  
<Rn>是base register，保存memory的address，LDREX指令从base register中获取memory address，并且将memory的内容加载到<Rt>(destination register)中。这些操作和ldr的操作是一样的，那么如何体现exclusive呢？其实，在执行这条指令的时候，还放出两条“狗”来负责观察特定地址的访问（就是保存在[<Rn>]中的地址了），这两条狗一条叫做local monitor，一条叫做global monitor。  
  
STREX <Rd>, <Rt>, [<Rn>]  
  
和LDREX指令类似，<Rn>是base register，保存memory的address，STREX指令从base register中获取memory address，并且将<Rt> (source register)中的内容加载到该memory中。这里的<Rd>保存了memeory 更新成功或者失败的结果，0表示memory更新成功，1表示失败。STREX指令是否能成功执行是和local monitor和global monitor的状态相关的。对于Non-shareable memory（该memory不是多个CPU之间共享的，只会被一个CPU访问），只需要放出该CPU的local monitor这条狗就OK了，下面的表格可以描述这种情况

  

|   |   |   |
|---|---|---|
|thread 1|thread 2|local monitor的状态|
|||Open Access state|
|LDREX||Exclusive Access state|
||LDREX|Exclusive Access state|
||Modify|Exclusive Access state|
||STREX|Open Access state|
|Modify||Open Access state|
|STREX||在Open Access state的状态下，执行STREX指令会导致该指令执行失败|
|||保持Open Access state，直到下一个LDREX指令|

开始的时候，local monitor处于Open Access state的状态，thread 1执行LDREX 命令后，local monitor的状态迁移到Exclusive Access state（标记本地CPU对xxx地址进行了LDREX的操作），这时候，中断发生了，在中断handler中，又一次执行了LDREX ，这时候，local monitor的状态保持不变，直到STREX指令成功执行，local monitor的状态迁移到Open Access state的状态（清除xxx地址上的LDREX的标记）。返回thread 1的时候，在Open Access state的状态下，执行STREX指令会导致该指令执行失败（没有LDREX的标记，何来STREX），说明有其他的内核控制路径插入了。

对于shareable memory，需要系统中所有的local monitor和global monitor共同工作，完成exclusive access，概念类似，这里就不再赘述了。

大概的原理已经描述完毕，下面回到具体实现面。

> "1:    ldrex    %0, [%3]\n"

其中％3就是input operand list中的"r" (&v->counter)，r是限制符（constraint），用来告诉编译器gcc，你看着办吧，你帮我选择一个通用寄存器保存该操作数吧。％0对应output openrand list中的"=&r" (result)，=表示该操作数是write only的，&表示该操作数是一个earlyclobber operand，具体是什么意思呢？编译器在处理嵌入式汇编的时候，倾向使用尽可能少的寄存器，如果output operand没有&修饰的话，汇编指令中的input和output操作数会使用同样一个寄存器。因此，&确保了％3和％0使用不同的寄存器。

（5）完成步骤（4）后，％0这个output操作数已经被赋值为atomic_t变量的old value，毫无疑问，这里的操作是要给old value加上i。这里％4对应"Ir" (i)，这里“I”这个限制符对应ARM平台，表示这是一个有特定限制的立即数，该数必须是0～255之间的一个整数通过rotation的操作得到的一个32bit的立即数。这是和ARM的data-processing instructions如何解析立即数有关的。每个指令32个bit，其中12个bit被用来表示立即数，其中8个bit是真正的数据，4个bit用来表示如何rotation。更详细的内容请参考ARM ARM文档。

（6）这一步将修改后的new value保存在atomic_t变量中。是否能够正确的操作的状态标记保存在％1操作数中，也就是"=&r" (tmp)。

（7）检查memory update的操作是否正确完成，如果OK，皆大欢喜，如果发生了问题（有其他的内核路径插入），那么需要跳转到lable 1那里，从新进行一次read-modify-write的操作。

_原创文章，转发请注明出处。蜗窝科技_

_[http://www.wowotech.net/linux_kenrel/atomic.html](http://www.wowotech.net/linux_kenrel/atomic.html)_

标签: [原子操作](http://www.wowotech.net/tag/%E5%8E%9F%E5%AD%90%E6%93%8D%E4%BD%9C) [atomic](http://www.wowotech.net/tag/atomic) [内核同步](http://www.wowotech.net/tag/%E5%86%85%E6%A0%B8%E5%90%8C%E6%AD%A5)

[![](http://www.wowotech.net/content/uploadfile/201605/ef3e1463542768.png)](http://www.wowotech.net/support_us.html)

« [Linux内核同步机制之（二）：Per-CPU变量](http://www.wowotech.net/kernel_synchronization/per-cpu.html) | [Linux电源管理(11)_Runtime PM之功能描述](http://www.wowotech.net/pm_subsystem/rpm_overview.html)»

**评论：**

**testtest**  
2023-02-14 13:38

ldrex和strex的方案，如果三个线程竞争是不是有问题？如下场景  
  
ldrex 1 => r1        thread1  
modify r1++            thread1  
ldrex 1 => r2        thread2  
modify r2++            thread2  
strex r2            thread2 success  
ldrex 2 => r3        thread3  
modify r3++            thread3  
strex r1            thread1 success  
strex r3            thread3 failed  
ldrex 2 => r3        thread3  
modify r3++            thread3  
strex r3            thread3 success

[回复](http://www.wowotech.net/kernel_synchronization/atomic.html#comment-8740)

**wangdl**  
2023-08-26 20:39

@testtest：分析的对啊，完全没问题

[回复](http://www.wowotech.net/kernel_synchronization/atomic.html#comment-8809)

**testtest**  
2024-01-25 10:25

@wangdl：就是线程1的ldrex因为线程2先strex而失效，但在线程1strex之前，其他线程如果又ldrex一次，那么线程1下来的strex是不是还是有效的？

[回复](http://www.wowotech.net/kernel_synchronization/atomic.html#comment-8864)

**Liu**  
2022-06-08 11:13

这里有一个疑惑，如果关闭中断后由外部中断触发怎么办？是不是说如果系统要求很高的时候这种保证原子的方法不太实用呢

[回复](http://www.wowotech.net/kernel_synchronization/atomic.html#comment-8627)

**mingtian**  
2022-08-10 18:05

@Liu：后面的原子操作并没有关闭中断，只是通过局部检视和全局检视两种实现了原子操作

[回复](http://www.wowotech.net/kernel_synchronization/atomic.html#comment-8667)

**wangxingxing**  
2020-07-31 16:59

Hi您好，想请教一下对整型变量如下的这些位操作是在哪里实现的呀：  
extern void _set_bit(int nr, volatile unsigned long * p);  
extern void _clear_bit(int nr, volatile unsigned long * p);  
extern void _change_bit(int nr, volatile unsigned long * p);  
extern int _test_and_set_bit(int nr, volatile unsigned long * p);  
extern int _test_and_clear_bit(int nr, volatile unsigned long * p);  
extern int _test_and_change_bit(int nr, volatile unsigned long * p);

[回复](http://www.wowotech.net/kernel_synchronization/atomic.html#comment-8078)

**nothing**  
2019-08-26 12:55

a                          b  
ldrex-->设置独占标记  
                           ldrex  
                           modify-->加1操作  
                           strex  
                           ldrex  
modify-->加1操作  
strex  
                           modify-->加1操作  
                           strex  
a、b线程在同一个CPU核上，如果按照上面执行顺序发展的话，a线程的modify是否会成功？

[回复](http://www.wowotech.net/kernel_synchronization/atomic.html#comment-7613)

**Alex.Rowe**  
2019-12-10 16:08

@nothing：@nothing，提到的这个场景，a线程的modify-strex操作完成后，个人也是觉得可能会产生数据的不一致问题。谁能帮忙解决下困惑啊？  
  
a                          b  
ldrex-->设置独占标记  
                           ldrex  
                           modify-->加1操作  
                           strex-->清除mem_addr对应的独占标记  
                           ldrex-->设置mem_addr对应的独占标记  
modify-->加1操作，修改的旧值  
strex-->清除mem_addr对应的独占标记  
                           modify-->加1操作  
                           strex-->没有mem_addr对应的独占标记，所以执行失败

[回复](http://www.wowotech.net/kernel_synchronization/atomic.html#comment-7781)

**Alex.Rowe**  
2019-12-11 13:50

@Alex.Rowe：可能是Context switch时，软件上有clrex或dummy strex操作，所以a的strex会失败？

[回复](http://www.wowotech.net/kernel_synchronization/atomic.html#comment-7786)

**[Saturn](http://no/)**  
2020-03-12 17:45

@Alex.Rowe：http://infocenter.arm.com/help/topic/com.arm.doc.dht0008a/DHT0008A_arm_synchronization_primitives.pdf  
针对这个问题，这份文档(page:1-7)里面有描述。上下文切换的时候local monitor会重置。  
  
Resetting monitors  
When an operating system performs a context switch, it must reset the local monitor to open state, to prevent false positives occurrig。

[回复](http://www.wowotech.net/kernel_synchronization/atomic.html#comment-7920)

**九五二七**  
2020-07-04 16:20

@Saturn：@Saturn，您好，请教一下，按照您的回复，a线程中的strex会失败，前面的modify并没有写进memory，但由于失败跳到1b，所以下次的ldrex如果没有context switch的话是可以modify成功的是吗，而且是在b线程modify之后？也就是说atomic_add不管怎样都会更新成功，从这个层面来看，atomic_add操作就是原子的，尽管在汇编语句中可能有switch？

[回复](http://www.wowotech.net/kernel_synchronization/atomic.html#comment-8044)

**cp**  
2021-05-07 20:34

@Saturn：如果a b是两个硬件线程呢,a线程会成功写入?

[回复](http://www.wowotech.net/kernel_synchronization/atomic.html#comment-8226)

**yaoike**  
2019-05-04 16:28

请教下，原文：  硬件arbiter的限制使得CPU的写回必须是串行化的。  
  
那是不是说，受硬件arbiter的限制，不考虑其它方面，纯粹从多核CPU（比如说：8核CPU) 在写数据到 memory 这一点上，相比于单核CPU，并无优势。  
  
即单核，多核CPU 同时写同样一堆数据到 memory ，耗时一样。

[回复](http://www.wowotech.net/kernel_synchronization/atomic.html#comment-7391)

**[bsp](http://www.wowotech.net/)**  
2020-11-09 10:46

@yaoike：忽略cache以及指令的pipeline时，耗时基本一样。

[回复](http://www.wowotech.net/kernel_synchronization/atomic.html#comment-8132)

**弄堂小子**  
2018-10-19 10:10

请教下，  
原文中：对于UP，原子操作就是通过关闭CPU中断来完成的。  
  是否可以理解关闭中断有包括禁止抢占的作用，不然没法保证原子？  
  评论区有提到V6之前用的是SWP和SWPB指令，现在用关中断方式处理了，这两个是如何比较取舍的？

[回复](http://www.wowotech.net/kernel_synchronization/atomic.html#comment-6996)

**xiaoai**  
2019-05-14 16:36

@弄堂小子：第一个问题我的理解：因为进程的调度从原始来说是依赖于中断的(往往是定时器中断)，如果关闭中断基本上就相当于是终止了进程的调度，这样多线程访问临界资源的问题就规避了。还有一种就是其他中断可能访问这种临界资源，这时候关闭了中断也就避免了并行的修改。这样就实现了原子操作。

[回复](http://www.wowotech.net/kernel_synchronization/atomic.html#comment-7417)

**九五二七**  
2020-07-04 16:24

@弄堂小子：您的两个问题我也比较疑惑，请问有找到答案吗？

[回复](http://www.wowotech.net/kernel_synchronization/atomic.html#comment-8045)

**double_qiang**  
2023-03-14 11:58

@九五二七：我的理解抢占时机可以简化成两种，一种是依赖中断发生调度时，另外一个就是当前进程主动调用schedule，显然后面这种的抢占时机肯定是不存在原子操作的

[回复](http://www.wowotech.net/kernel_synchronization/atomic.html#comment-8753)

**sgy1993**  
2018-02-10 11:42

对以下代码有点疑问?希望您能解疑.....  
static inline void atomic_add(int i, atomic_t *v)  
{  
    unsigned long tmp;  
    int result;  
  
    prefetchw(&v->counter); －－－－－－－－－－－－－－－－－－－－－－－－－（2）  
    __asm__ __volatile__("@ atomic_add\n" －－－－－－－－－－－－－－－－－－（3）  
"1:    ldrex    %0, [%3]\n" －－－－－－－－－－－－－－－－－－－－－－－－－－（4）  
"    add    %0, %0, %4\n" －－－－－－－－－－－－－－－－－－－－－－－－－－（5）  
"    strex    %1, %0, [%3]\n" －－－－－－－－－－－－－－－－－－－－－－－－－（6）  
"    teq    %1, #0\n" －－－－－－－－－－－－－－－－－－－－－－－－－－－－－（7）  
"    bne    1b"  
    : "=&r" (result), "=&r" (tmp), "+Qo" (v->counter) －－－对应％0，％1，％2  
    : "r" (&v->counter), "Ir" (i) －－－－－－－－－－－－－对应％3，％4  
    : "cc");  
}  
假如只有一个cpu, a程序想对v->counter加1操作, 执行(4)之后, 发生中断,开始执行b程序, b也要访问这个v->counter(对其进行加1操作),之前a执行了ldrex, 所以b的 strex对couter的操作是能够写进去的...此时又转回去执行a..会一直循环到执行成功,...这样从结果上看不就 加 2了吗??  
  
a                          b  
ldrex-->设置独占标记  
                           ldrex  
                           modify-->加1操作  
                           strex  
modify-->加1操作  
strex

[回复](http://www.wowotech.net/kernel_synchronization/atomic.html#comment-6552)

**hello-world**  
2018-02-11 09:35

@sgy1993：执行过程如下：  
a                          b  
ldrex-->设置独占标记  
                           ldrex  
                           modify-->加1操作  
                           strex  
modify-->加1操作  
strex（error，retry）  
ldrex-->设置独占标记  
modify-->加1操作  
strex（OK）  
因此，结果是对的。

[回复](http://www.wowotech.net/kernel_synchronization/atomic.html#comment-6554)

**sgy1993**  
2018-02-11 13:16

@hello-world：感谢回复:  
    a只是想加1.. 但是由于b程序能成功加1, 最终结果变量加了2...这个正常吗?

[回复](http://www.wowotech.net/kernel_synchronization/atomic.html#comment-6556)

**九五二七**  
2020-07-04 16:18

@sgy1993：@sgy1993，您好，b线程也要加1，我觉得没有问题。

[回复](http://www.wowotech.net/kernel_synchronization/atomic.html#comment-8043)

**benson**  
2018-01-01 23:45

armv8上多核结构上，spinlock上必须每个都核都把mmu初始化后，才能用吗？

[回复](http://www.wowotech.net/kernel_synchronization/atomic.html#comment-6444)

**michael**  
2017-03-27 11:33

有些看不懂。  
对于原子操作，Linux内核是通过哪种原理或者机制，保证了同步？  
博主能否用一些常规语言来描述一下。谢谢了

[回复](http://www.wowotech.net/kernel_synchronization/atomic.html#comment-5381)

**michael**  
2017-03-27 11:34

@michael：另外，汇编语言也可以被中断打断的吧？

[回复](http://www.wowotech.net/kernel_synchronization/atomic.html#comment-5382)

**[linuxer](http://www.wowotech.net/)**  
2017-03-28 11:47

@michael：原子操作是和CPU ARCH相关的内容，在linux内核中提供了统一的接口，底层有各自的实现。

[回复](http://www.wowotech.net/kernel_synchronization/atomic.html#comment-5387)

**michael**  
2017-03-28 15:07

@linuxer：原子操作过程中，interrupt controller是不会响应中断的吧，包括CPU中断和外设中断?

[回复](http://www.wowotech.net/kernel_synchronization/atomic.html#comment-5388)

**[linuxer](http://www.wowotech.net/)**  
2017-03-29 09:53

@michael：除非你disable了interrupt controller的功能，否则它应该是会响应中断请求的，并通知到CPU，当然cpu是否响应中断是看CPU是否disable了local的中断响应，如果没有，那么cpu当然会响应异步中断事件。  
当然，如果原子操作只是一条指令（例如SWP指令），那么中断响应要么在该指令之前发生，要么在该指令之后发生。  
如果是exclusive指令来完成原子操作，那么中断是可以插入的。

[回复](http://www.wowotech.net/kernel_synchronization/atomic.html#comment-5391)

**michael**  
2017-03-30 13:06

@linuxer：感谢您的及时回复。  
  
您的文档中这样描述： 对于UP，原子操作就是通过关闭CPU中断来完成的  
  
对于UP（单核），由于原子操作是不允许打断的，所以需要关闭中断来实现吗？

[回复](http://www.wowotech.net/kernel_synchronization/atomic.html#comment-5392)

**[linuxer](http://www.wowotech.net/)**  
2017-03-31 19:13

@michael：我文章中的原话是：  
＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝  
（1）ARMv6之前的CPU并不支持SMP，之后的ARM架构都是支持SMP的（例如我们熟悉的ARMv7-A）。因此，对于ARM处理，其原子操作分成了两个阵营，一个是支持SMP的ARMv6之后的CPU，另外一个就是ARMv6之前的，只有单核架构的CPU。对于UP，原子操作就是通过关闭CPU中断来完成的。  
＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝  
这里的“对于up”是指的那些只支持UP的的ARMv6之前的ARM处理器，这些ARM处理器没有原子指令支持，read-modify-write类型的操作只能是采用关中断来实现。

[回复](http://www.wowotech.net/kernel_synchronization/atomic.html#comment-5405)

**michael**  
2017-04-04 15:18

@linuxer：感谢您的回复。  
  
常说的，原子操作是最小的执行单元，执行过程中不会休眠，也不会中断。  
  
这是不是针对ARMv6之前的ARM处理器的？  
因为根据您的描述，从ARMv7开始，原子操作似乎可以被中断打断的。

**ron**  
2017-02-27 11:29

@linuxer: 我有个问题请教一下，arm中atomic_add/atomic_sub 这些操作是通过ldex/stex来实现的，但是atomic_set 只是通过#define atomic_set(v, i) （(（v）->counter) = (i)）来实现，那么atomic_set 不存在竞争的情况吗？

[回复](http://www.wowotech.net/kernel_synchronization/atomic.html#comment-5257)

**[linuxer](http://www.wowotech.net/)**  
2017-02-27 19:17

@ron：atomic_add/atomic_sub 这些操作三个步骤（read-modify-write）：  
1. read memory to register  
2. add register  
3. update the new value to the memory  
  
对于atomic_set，只有上面的步骤3而已，一个str操作就OK了，因此不需要特别处理。

[回复](http://www.wowotech.net/kernel_synchronization/atomic.html#comment-5267)

**ron**  
2017-02-28 14:58

@linuxer：@linuxer 多谢。got it.

[回复](http://www.wowotech.net/kernel_synchronization/atomic.html#comment-5271)

**sgy1993**  
2018-02-10 11:29

@linuxer：atomic_set(v, i)----会翻译成两条汇编指令啊!  
例如---  
atomic_t number = ATOMIC_INIT(1);  
atomic_set(&number, 4);  
上面这条指令的反汇编是  
  10:    e3a02004     mov    r2, #4  
  14:    e5832000     str    r2, [r3]  
这个怎么保证原子操作呢?加入 r2 = 4之后, 别的程序修改了r2.不就有问题????

[回复](http://www.wowotech.net/kernel_synchronization/atomic.html#comment-6551)

**hello-world**  
2018-02-11 09:31

@sgy1993：能保证：  
14:    e5832000     str    r2, [r3]  
这条指令是原子的就OK了。

[回复](http://www.wowotech.net/kernel_synchronization/atomic.html#comment-6553)

**sgy1993**  
2018-02-11 13:12

@hello-world：我觉的应该是mov r2,4执行完之后,是可能被中断打断的,但是中断处理的时候,会保存寄存器的值在堆栈上,中断处理返回来之后,r2的值会从堆栈上弹出来.这样counter的值仍然是4...(如果把4改成一个全局变量也是一样,理论上都是从内存某处取数据放到寄存器(ldr指令), 然后在str 寄存器的值-->内存)..即便ldr执行完,有中断改变了全局变量的值,但由于值已经被ldr存在寄存器里了..所以结果最终还是正确的...个人觉得原子操作应该是即便中断发生,而不影响最终结果的操作

[回复](http://www.wowotech.net/kernel_synchronization/atomic.html#comment-6555)

1 [2](http://www.wowotech.net/kernel_synchronization/atomic.html/comment-page-2#comments)

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
    
    - [Linux DMA Engine framework(2)_功能介绍及解接口分析](http://www.wowotech.net/linux_kenrel/dma_engine_api.html)
    - [tty驱动分析](http://www.wowotech.net/tty_framework/435.html)
    - [对heziq网友问题的回答](http://www.wowotech.net/pull-up-resistor.html)
    - [ARM CPU性能实验](http://www.wowotech.net/arm-performance-test.html)
    - [一个较复杂dcache问题](http://www.wowotech.net/linux_kenrel/484.html)
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