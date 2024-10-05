
作者：[smcdef](http://www.wowotech.net/author/531) 发布于：2019-1-23 22:59 分类：[内核同步机制](http://www.wowotech.net/sort/kernel_synchronization)

编译器（compiler）的工作就是优化我们的代码以提高性能。这包括在不改变程序行为的情况下重新排列指令。因为compiler不知道什么样的代码需要线程安全（thread-safe），所以compiler假设我们的代码都是单线程执行（single-threaded），并且进行指令重排优化并保证是单线程安全的。因此，当你不需要compiler重新排序指令的时候，你需要显式告诉compiler，我不需要重排。否则，它可不会听你的。本篇文章中，我们一起探究compiler关于指令重排的优化规则。

> 注：测试使用aarch64-linux-gnu-gcc版本：7.3.0

## 编译器指令重排(Compiler Instruction Reordering)

compiler的主要工作就是将对人们可读的源码转化成机器语言，机器语言就是对CPU可读的代码。因此，compiler可以在背后做些不为人知的事情。我们考虑下面的C语言代码：
```cpp
1. int a, b;

3. void foo(void)
4. {
5. 	a = b + 1;
6. 	b = 0;
7. } 
```
  
使用aarch64-linux-gnu-gcc在不优化代码的情况下编译上述代码，使用objdump工具查看foo()反汇编结果：

```c
<foo>:	...	ldr	w0, [x0]	// load b to w0	add	w1, w0, #0x1	...	
str	w1, [x0]	// a = b + 1	
..	
str	wzr, [x0]	// b = 0
```

我们应该知道Linux默认编译优化选项是-O2，因此我们采用-O2优化选项编译上述代码，并反汇编得到如下汇编结果：
```cpp
1. <foo>:
2. 	...
3. 	ldr	w2, [x0]	// load b to w2
4. 	str	wzr, [x0]	// b = 0
5. 	add	w0, w2, #0x1
6. 	str	w0, [x1]	// a = b + 1
7. 	... 
```
  
比较优化和不优化的结果，我们可以发现。在不优化的情况下，a 和 b 的写入内存顺序符合代码顺序（program order）。但是-O2优化后，a 和 b 的写入顺序和program order是相反的。-O2优化后的代码转换成C语言可以看作如下形式：

```c
int a, b; void foo(void){	register int reg = b; 	b = 0;	a = reg + 1;}
```

这就是compiler reordering（编译器重排）。为什么可以这么做呢？对于单线程来说，a 和 b 的写入顺序，compiler认为没有任何问题。并且最终的结果也是正确的（a == 1 && b == 0）。

这种compiler reordering在大部分情况下是没有问题的。但是在某些情况下可能会引入问题。例如我们使用一个全局变量`flag`标记共享数据`data`是否就绪。由于compiler reordering，可能会引入问题。考虑下面的代码（无锁编程）：
```cpp
1. int flag, data;

3. void write_data(int value)
4. {
5.     data = value;
6.     flag = 1;
7. } 
```

如果compiler产生的汇编代码是flag比data先写入内存。那么，即使是单核系统上，我们也会有问题。在flag置1之后，data写45之前，系统发生抢占。另一个进程发现flag已经置1，认为data的数据已经准别就绪。但是实际上读取data的值并不是45。为什么compiler还会这么操作呢？因为，compiler是不知道data和flag之间有严格的依赖关系。这种逻辑关系是我们人为强加的。我们如何避免这种优化呢？

## 显式编译器屏障(Explicit Compiler Barriers)

为了解决上述变量之间存在依赖关系导致compiler错误优化。compiler为我们提供了编译器屏障（compiler barriers），可用来告诉compiler不要reorder。我们继续使用上面的foo()函数作为演示实验，在代码之间插入compiler barriers。

```c
#define barrier() __asm__ __volatile__("": : :"memory") 
int a, b; void foo(void){	a = b + 1;	barrier();	b = 0;}
```

barrier()就是compiler提供的屏障，作用是告诉compiler内存中的值已经改变，之前对内存的缓存（缓存到寄存器）都需要抛弃，barrier()之后的内存操作需要重新从内存load，而不能使用之前寄存器缓存的值。并且可以防止compiler优化barrier()前后的内存访问顺序。barrier()就像是代码中的一道不可逾越的屏障，barrier前的 load/store 操作不能跑到barrier后面；同样，barrier后面的 load/store 操作不能在barrier之前。依然使用-O2优化选项编译上述代码，反汇编得到如下结果：

```c
<foo>:	...	ldr	w2, [x0]	// load b to w2	add	w2, w2, #0x1	
str	w2, [x1]	// a = a + 1	
str	wzr, [x0]	// b = 0	
...
```

我们可以看到插入compiler barriers之后，a 和 b 的写入顺序和program order一致。因此，当我们的代码中需要严格的内存顺序，就需要考虑compiler barriers。

## 隐式编译器屏障(Implied Compiler Barriers)

除了显示的插入compiler barriers之外，还有别的方法阻止compiler reordering。例如CPU barriers 指令，同样会阻止compiler reordering。后续我们再考虑CPU barriers。

除此以外，当某个函数内部包含compiler barriers时，该函数也会充当compiler barriers的作用。即使这个函数被inline，也是这样。例如上面插入barrier()的foo()函数，当其他函数调用foo()时，foo()就相当于compiler barriers。考虑下面的代码：
```cpp
1. int a, b, c;

3. void fun(void)
4. {
5. 	c = 2;
6. 	barrier();
7. }

9. void foo(void)
10. {
11. 	a = b + 1;
12. 	fun();		/* fun() call act as compiler barriers */
13. 	b = 0;
14. } 
```
  
fun()函数包含barrier()，因此foo()函数中fun()调用也表现出compiler barriers的作用。同样可以保证 a 和 b 的写入顺序。如果fun()函数不包含barrier()，结果又会怎么样呢？实际上，大多数的函数调用都表现出compiler barriers的作用。但是，这不包含inline的函数。因此，fun()如果被inline进foo()，那么fun()就不会具有compiler barriers的作用。如果被调用的函数是一个外部函数，其副作用会比compiler barriers还要强。因为compiler不知道函数的副作用是什么。它必须忘记它对内存所作的任何假设，即使这些假设对该函数可能是可见的。我么看一下下面的代码片段，printf()一定是一个外部的函数。
```cpp
1. int a, b;

3. void foo(void)
4. {
5. 	a = 5;
6. 	printf("smcdef");
7. 	b = a;
8. } 
```
  
同样使用-O2优化选项编译代码，objdump反汇编得到如下结果。

```c
<foo>:	...	mov	w2, #0x5			// #5	
str	w2, [x19]			// a = 5	bl	640 <__printf_chk@plt>		// printf()	ldr	w1, [x19]			// reload a to w1	...	
str	w1, [x0]			// b = a
```

compiler不能假设printf()不会使用或者修改 a 变量。因此在调用printf()之前会将 a 写5，以保证printf()可能会用到新值。在printf()调用之后，重新从内存中load a 的值，然后赋值给变量 b。重新load a 的原因是compiler也不知道printf()会不会修改 a 的值。

因此，我们可以看到即使存在compiler reordering，但是还是有很多限制。当我们需要考虑compiler barriers时，一定要显示的插入barrier()，而不是依靠函数调用附加的隐式compiler barriers。因为，谁也无法保证调用的函数不会被compiler优化成inline方式。
## barrier()除了防止编译乱序，还没能做什么

barriers()作用除了防止compiler reordering之外，还有什么妙用吗？我们考虑下面的代码片段。

```c
int run = 1; void foo(void){	while (run)		;}
```

run是个全局变量，foo()在一个进程中执行，一直循环。我们期望的结果时foo()一直等到其他进程修改run的值为0才推出循环。实际compiler编译的代码和我们会达到我们预期的结果吗？我们看一下汇编代码。

```c

0000000000000748 <foo>: 748:	90000080 	adrp	x0, 10000 74c:	f947e800 	ldr	x0, [x0, #4048] 750:	b9400000 	ldr	w0, [x0]				// load run to w0 754:	d503201f 	nop 758:	35000000 	cbnz	w0, 758 <foo+0x10>	// if (w0) while (1); 75c:	d65f03c0 	ret
```

汇编代码可以转换成如下的C语言形式。

```c
int run = 1; void foo(void){	register int reg = run; 	if (reg)		while (1)			;}
```

compiler首先将run加载到一个寄存器reg中，然后判断reg是否满足循环条件，如果满足就一直循环。但是循环过程中，寄存器reg的值并没有变化。因此，即使其他进程修改run的值为0，也不能使foo()退出循环。很明显，这不是我们想要的结果。我们继续看一下加入barrier()后的结果。

```c
0000000000000748 <foo>: 748:	90000080 	adrp	x0, 10000 74c:	f947e800 	ldr	x0, [x0, #4048] 750:	b9400001 	ldr	w1, [x0]				// load run to w0 754:	34000061 	cbz	w1, 760 <foo+0x18> 758:	b9400001 	ldr	w1, [x0]				// load run to w0 75c:	35ffffe1 	cbnz	w1, 758 <foo+0x10>	// if (w0) goto 758 760:	d65f03c0 	ret
```

我们可以看到加入barrier()后的结果真是我们想要的。每一次循环都会从内存中重新load run的值。因此，当有其他进程修改run的值为0的时候，foo()可以正常退出循环。为什么加入barrier()后的汇编代码就是正确的呢？因为barrier()作用是告诉compiler内存中的值已经变化，后面的操作都需要重新从内存load，而不能使用寄存器缓存的值。因此，这里的run变量会从内存重新load，然后判断循环条件。这样，其他进程修改run变量，foo()就可以看得见了。

在Linux kernel中，提供了[cpu_relax()](https://elixir.bootlin.com/linux/v4.15.7/source/arch/arm64/include/asm/processor.h#L178)函数，该函数在ARM64平台定义如下：
```cpp
1. static inline void cpu_relax(void)
2. {
3. 	asm volatile("yield" ::: "memory");
4. } 
```

我们可以看出，cpu_relax()是在barrier()的基础上又插入一条汇编指令yield。在kernel中，我们经常会看到一些类似上面举例的while循环，循环条件是个全局变量。为了避免上述所说问题，我们就会在循环中插入cpu_relax()调用。
  
```cpp
1. int run = 1;

3. void foo(void)
4. {
5. 	while (run)
6. 		cpu_relax();
7. }
```
  
当然也可以使用Linux 提供的READ_ONCE()。例如，下面的修改也同样可以达到我们预期的效果。

```c
int run = 1; void foo(void){	while (READ_ONCE(run))	/* similar to while (*(volatile int *)&run) */		;}
```

当然你也可以修改run的定义为`volatile int run，`就会得到如下代码。同样可以达到预期目的。

```cpp
1. volatile int run = 1;

3. void foo(void)
4. {
5. 	while (run)
6. 		;
7. }
```
  
关于volatile更多使用建议可以参考[这里](https://www.kernel.org/doc/html/v4.10/process/volatile-considered-harmful.html)。

标签: [barrier](http://www.wowotech.net/tag/barrier)

---

« [copy_{to,from}_user()的思考](http://www.wowotech.net/memory_management/454.html) | [CFS调度器（6）-总结](http://www.wowotech.net/process_management/452.html)»

**评论：**

**steven**  
2019-07-17 16:03

<foo>:  
    ...  
    ldr    w0, [x0]    // load b to w0  
    add    w1, w0, #0x1  
    ...  
    str    w1, [x0]    // a = b + 1   这个地方有问题，str  w1, [x0]不应该是x0，否则变成b=b+1  
    ...  
    str    wzr, [x0]    // b = 0

[回复](http://www.wowotech.net/kernel_synchronization/453.html#comment-7535)

**[smcdef](http://www.wowotech.net/)**  
2019-07-17 23:08

@steven：这个是其实是汇编直接copy过来的，中间省略部分可能就是改变了x0的值。但是的确这段代码会让人觉得歧义。

[回复](http://www.wowotech.net/kernel_synchronization/453.html#comment-7538)

**石石**  
2019-07-09 16:21

请问compiler reorder 跟CPU execution reorder 该如何理解?  
  
当使用了barrier(), 让compiler正确的编译出正确的顺序(program order),  
却没有对cpu out-of-order execution做出确保(execution order)?  
  
对一个C programmer而言, 这样的问题该理解到哪个层次, 才能确保程式正确性?

[回复](http://www.wowotech.net/kernel_synchronization/453.html#comment-7522)

**石石**  
2019-07-09 16:38

@石石：如果cpu有OOOE, 不能单纯用 __asm__ __volatile__("": : :"memory")  
应该使用__asm__ __volatile__("dmb": : :"memory"),  
machine-specific implementation 应该使用 __sync_synchronize().

[回复](http://www.wowotech.net/kernel_synchronization/453.html#comment-7523)

**[smcdef](http://www.wowotech.net/)**  
2019-07-09 23:06

@石石：是的，这篇文章其实算是乱序的上篇，仅仅是编译器乱序。然而CPU也是存在乱序的，所以CPU乱序部分的文章还没有发表。后面会补上。

[回复](http://www.wowotech.net/kernel_synchronization/453.html#comment-7525)

**steven**  
2019-03-05 20:25

add    w0, w2, #0x13 有笔误，应该是#0x1吧

[回复](http://www.wowotech.net/kernel_synchronization/453.html#comment-7222)

**[smcdef](http://www.wowotech.net/)**  
2019-03-07 22:27

@steven：是的，感谢。

[回复](http://www.wowotech.net/kernel_synchronization/453.html#comment-7246)

**威点零**  
2019-02-19 11:55

问个问题，为什么我们正常写程序的时候，不需要考虑加内存屏障？  
所以是不是要加上限定关系？比如有前后因果关系的指令不会重排，用户层面全局变量使用互斥锁能防止重排？

[回复](http://www.wowotech.net/kernel_synchronization/453.html#comment-7190)

**[smcdef](http://www.wowotech.net/)**  
2019-02-24 20:54

@威点零：考虑屏障，一般都是无锁编程需要。如果使用同步原语（自旋锁，信号量，互斥锁等）保护共享资源，我们是不需要考虑内存屏障的。因为，同步原语的实现已经帮助我们考虑好了。

[回复](http://www.wowotech.net/kernel_synchronization/453.html#comment-7202)

**wulala**  
2021-06-03 15:25

@smcdef：其实我更关心层主说的"是不是要加上限定关系？比如有前后因果关系的指令不会重排",并且什么样的"限定关系"能达到这样的效果

[回复](http://www.wowotech.net/kernel_synchronization/453.html#comment-8241)

**xtzt**  
2019-01-24 14:22

写的很好！！！

[回复](http://www.wowotech.net/kernel_synchronization/453.html#comment-7152)

**yayaya**  
2019-01-24 10:44

编译器（compiler）的工作就是优化我们的代码以提高性能??

[回复](http://www.wowotech.net/kernel_synchronization/453.html#comment-7148)

**[smcdef](http://www.wowotech.net/)**  
2019-01-24 10:54

@yayaya：很明显其中之一的功能嘛！

[回复](http://www.wowotech.net/kernel_synchronization/453.html#comment-7149)

**yayaya**  
2019-01-24 11:29

@smcdef：我处女座^_^

[回复](http://www.wowotech.net/kernel_synchronization/453.html#comment-7150)

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
    
    - [快讯：蓝牙5.0发布（新特性速览）](http://www.wowotech.net/bluetooth/bluetooth_5_0_overview.html)
    - [玩转BLE(2)_使用bluepy扫描BLE的广播数据](http://www.wowotech.net/bluetooth/bluepy_scan.html)
    - [内存一致性模型](http://www.wowotech.net/memory_management/456.html)
    - [linux内核同步](http://www.wowotech.net/98.html)
    - [我的bash shell学习笔记](http://www.wowotech.net/linux_application/134.html)
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