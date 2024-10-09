作者：[heaven](http://www.wowotech.net/author/532) 发布于：2022-2-11 1:34 分类：[ARMv8A Arch](http://www.wowotech.net/sort/armv8a_arch)
[Zhang Binghua](https://tinylab.org/arm-wfe/#author-footer "点击联系作者") 创作于 2020/05/19
有发表于泰晓科技，原文见：
[https://tinylab.org/arm-wfe/](https://tinylab.org/arm-wfe/)

## 1 背景简介

今天我想分享一个跟多核锁原理相关的东西，由于我搞 arm 居多，所以目前只研究了 arm 架构下的 WFE 指令，分享出来，如果有表述不精准或者错误的地方还请大家指出，非常感谢。研究这个原因也是只是想搞清楚所以然和来龙去脉，以后写代码可以更游刃有余。

```cpp
1. static inline void arch_spin_lock(arch_spinlock_t *lock)
2. {
3. 	unsigned long tmp;
4. 	u32 newval;
5. 	arch_spinlock_t lockval;
6. 	prefetchw(&lock->slock);
7. 	__asm__ __volatile__(
8. "1: ldrex %0, [%3]\n"
9. " add %1, %0, %4\n"
10. " strex %2, %1, [%3]\n"
11. " teq %2, #0\n"
12. " bne 1b"
13. 	: "=&r" (lockval), "=&r" (newval), "=&r" (tmp)
14. 	: "r" (&lock->slock), "I" (1 << TICKET_SHIFT)
15. 	: "cc");
16. 	while (lockval.tickets.next != lockval.tickets.owner) {
17. 		wfe();
18. 		lockval.tickets.owner = READ_ONCE(lock->tickets.owner);
19. 	}
20. 	smp_mb();
21. }
```

我印象之前的 kernel 是没有这个 wfe 这个函数的，当 cpu0 获取到锁后，如果 cpu1 再想获取锁，此时会被 lock 住，然后进入死等的状态，那么 wfe 这个指令的作用是会让 cpu 进入 low power standby，这样可以降低功耗，本来发生竞态时其他的 cpu 都要等待这个锁释放才能运行，有了这个指令，相当于是“因祸得福”了，还可以降低功耗，当然这是有条件的，后面追溯并研究了一下 wfe 这个指令的作用。

## 3 spinlock 与 WFE、SEV、WFI

首先 `spin_lock` 函数，搞内核的大家都知道，那么我把 linux-stable 的代码黏贴出来如下：

```cpp
1. static __always_inline void spin_lock(spinlock_t *lock)
2. {
3. 	raw_spin_lock(&lock->rlock);
4. }
5. #define raw_spin_lock(lock)	_raw_spin_lock(lock)
6. #ifndef CONFIG_INLINE_SPIN_LOCK
7. void __lockfunc _raw_spin_lock(raw_spinlock_t *lock)
8. {
9. 	__raw_spin_lock(lock);
10. }
11. EXPORT_SYMBOL(_raw_spin_lock);
12. #endif
13. static inline void __raw_spin_lock(raw_spinlock_t *lock)
14. {
15. 	preempt_disable();
16. 	spin_acquire(&lock->dep_map, 0, 0, _RET_IP_);
17. 	LOCK_CONTENDED(lock, do_raw_spin_trylock, do_raw_spin_lock);
18. }
19. /*
20. * We are now relying on the NMI watchdog to detect lockup instead of doing
21. * the detection here with an unfair lock which can cause problem of its own.
22. */
23. void do_raw_spin_lock(raw_spinlock_t *lock)
24. {
25. 	debug_spin_lock_before(lock);
26. 	arch_spin_lock(&lock->raw_lock);
27. 	mmiowb_spin_lock();
28. 	debug_spin_lock_after(lock);
29. }
30. /*
31. * ARMv6 ticket-based spin-locking.
32. *
33. * A memory barrier is required after we get a lock, and before we
34. * release it, because V6 CPUs are assumed to have weakly ordered
35. * memory.
36. */
37. static inline void arch_spin_lock(arch_spinlock_t *lock)
38. {
39. 	unsigned long tmp;
40. 	u32 newval;
41. 	arch_spinlock_t lockval;
42. 	prefetchw(&lock->slock);
43. 	__asm__ __volatile__(
44. "1: ldrex %0, [%3]\n"
45. " add %1, %0, %4\n"
46. " strex %2, %1, [%3]\n"
47. " teq %2, #0\n"
48. " bne 1b"
49. 	: "=&r" (lockval), "=&r" (newval), "=&r" (tmp)
50. 	: "r" (&lock->slock), "I" (1 << TICKET_SHIFT)
51. 	: "cc");
52. 	while (lockval.tickets.next != lockval.tickets.owner) {
53. 		wfe();
54. 		lockval.tickets.owner = READ_ONCE(lock->tickets.owner);
55. 	}
56. 	smp_mb();
57. }
```

对于 arm32：

```cpp
1. #if __LINUX_ARM_ARCH__ >= 7 || \
2. 	(__LINUX_ARM_ARCH__ == 6 && defined(CONFIG_CPU_32v6K))
3. #define sev()	__asm__ __volatile__ ("sev" : : : "memory")
4. #define wfe()	__asm__ __volatile__ ("wfe" : : : "memory")
5. #define wfi()	__asm__ __volatile__ ("wfi" : : : "memory")
6. #else
7. #define wfe()	do { } while (0)
8. #endif
```

对于 arm64：

```cpp
1. #define sev()		asm volatile("sev" : : : "memory")
2. #define wfe()		asm volatile("wfe" : : : "memory")
3. #define wfi()		asm volatile("wfi" : : : "memory")
4. static inline void arch_spin_unlock(arch_spinlock_t *lock)
5. {
6. 	smp_mb();
7. 	lock->tickets.owner++;
8. 	dsb_sev();
9. }
10. #define SEV		__ALT_SMP_ASM(WASM(sev), WASM(nop))
11. static inline void dsb_sev(void)
12. {
13. 	dsb(ishst);
14. 	__asm__(SEV);
15. }
16. #ifdef CONFIG_SMP
17. #define __ALT_SMP_ASM(smp, up)						\
18. 	"9998: " smp "\n"						\
19. 	" .pushsection \".alt.smp.init\", \"a\"\n"		\
20. 	" .long 9998b\n"					\
21. 	" " up "\n"						\
22. 	" .popsection\n"
23. #else
24. #define __ALT_SMP_ASM(smp, up)	up
25. #endif
```

以上我们可以看出，在 lock 的时候使用 WFE，在 unlock 的时候使用 SEV，这个必须要成对使用，原因我下面会说。

对于内核版本 Linux 3.0.56：

```cpp
1. static inline void arch_spin_lock(arch_spinlock_t *lock)
2. {
3. 	unsigned long tmp;
4. 	__asm__ __volatile__(
5. "1: ldrex %0, [%1]\n"
6. " teq %0, #0\n"
7. 	WFE("ne")
8. " strexeq %0, %2, [%1]\n"
9. " teqeq %0, #0\n"
10. " bne 1b"
11. 	: "=&r" (tmp)
12. 	: "r" (&lock->lock), "r" (1)
13. 	: "cc");
14. 	smp_mb();
15. }
```

对于 Linux 2.6.18：

```cpp
1. define _raw_spin_lock(lock)     __raw_spin_lock(&(lock)->raw_lock)
2. static inline void __raw_spin_lock(raw_spinlock_t *lock)
3. {
4.     unsigned long tmp;
5.     __asm__ __volatile__(
6. "1: ldrex %0, [%1]\n"
7. " teq %0, #0\n"
8. " strexeq %0, %2, [%1]\n"
9. " teqeq %0, #0\n"
10. " bne 1b"
11.     : "=&r" (tmp)
12.     : "r" (&lock->lock), "r" (1)
13.     : "cc");
14.     smp_mb();
15. }
```

以上大家可以看出，最早期的 kernel 版本是没有 wfe 这条指令的，后面的版本才有。

## 4 WFE、SEV 与 WFI 的作用与工作原理

那这条指令的作用是什么呢？我们可以上 arm 官网去查看这条指令的描述：\[ARM Software development tools

- SEV

\](http://infocenter.arm.com/help/index.jsp?lang=en)

> 1. SEV causes an event to be signaled to all cores within a multiprocessor system. If SEV is implemented, WFE must also be implemented.

SEV 指令可以产生事件信号，发送给全部的 cpu，让他们唤醒。如果 SEV 实现了，那么 WFE 也必须被实现。这里的事件信号其实会表现为 Event register，这是个一 bit 的 register，如果有事件，那么此 bit 为真。

- WFE

> If the Event Register is not set, WFE suspends execution until one of the following events\
> occurs:\
> • an IRQ interrupt, unless masked by the CPSR I-bit\
> • an FIQ interrupt, unless masked by the CPSR F-bit\
> • an Imprecise Data abort, unless masked by the CPSR A-bit\
> • a Debug Entry request, if Debug is enabled\
> • an Event signaled by another processor using the SEV instruction.\
> If the Event Register is set, WFE clears it and returns immediately.

> If WFE is implemented, SEV must also be implemented.

对于 WFE，如果 Event Register 没有设置，WFE 会让 cpu 进入 low-power state，直到下面列举的五个 events 产生，比如说中断等等都会唤醒当前因为 WFE 而 suspend 的cpu。

如果 Event Register 被设置了，那么 WFE 会直接返回，不让 cpu 进入low-power state，目的是因为既然有事件产生了，说明当前 cpu 需要干活，不能 suspend，所以才这样设计。

这里我很好奇 Event Register 到底是怎么理解的。因此需要阅读手册《ARM Architecture Reference Manual.pdf》，下面我会做说明。

- WFI

> WFI suspends execution until one of the following events occurs:\
> • an IRQ interrupt, regardless of the CPSR I-bit\
> • an Imprecise Data abort, unless masked by the CPSR A-bit
>
> • a Debug Entry request, regardless of whether Debug is enabled.
>
> • an FIQ interrupt, regardless of the CPSR F-bit

而对于 WFI 这种，不会判断 Event register，暴力的直接让 cpu 进入 low-power state，直到有上述四个 events 产生才会唤醒 cpu。

注意，这里 WFE 比 WFI 多了一个唤醒特性：

> an Event signaled by another processor using the SEV instruction.

也就是说 SEV 是不会唤醒 WFI 指令休眠的 cpu 的。这点需要特别注意。

接下来我谈下这个 Event Register 是怎么回事了。看这个文档《ARM Architecture Reference Manual.pdf》

- The Event Register

> The Event Register is a single bit register for each processor. When set, an event register indicates that an event has occurred, since the register was last cleared, that might require some action by the processor. Therefore, the processor must not suspend operation on issuing a WFE instruction.\
> The reset value of the Event Register is UNKNOWN.\
> The Event Register is set by:\
> • an SEV instruction\
> • an event sent by some IMPLEMENTATION DEFINED mechanism\
> • a debug event that causes entry into Debug state\
> • an exception return.\
> As shown in this list, the Event Register might be set by IMPLEMENTATION DEFINED mechanisms.
>
> Software cannot read or write the value of the Event Register directly.
>
> The Event Register is cleared only by a Wait For Event instruction.

以上就是 Event Register 的表述，上述已经说的很明白了，Event Register 只有一个 bit，可以被 set 的情况总有四种大类型。当任意一个条件满足的时候，Event Register 都可以被 set，那么当 WFE 进入的时候会进行 Event Register 的判断，如果为真，就直接返回。

再来看看 WFE 的介绍：

> Wait For Event is a hint instruction that permits the processor to enter a low-power state until one of a number of events occurs, including events signaled by executing the SEV instruction on any processor in the multiprocessor system. For more information, see Wait For Event and Send Event on page B1-1197.
>
> In an implementation that includes the Virtualization Extensions, if HCR.TWE is set to 1, execution of a WFE instruction in a Non-secure mode other than Hyp mode generates a Hyp Trap exception if, ignoring the value of the HCR.TWE bit, conditions permit the processor to suspend execution. For more information see Trapping use of the WFI and WFE instructions on page B1-1249.

接下来上 WFE 这条指令的伪代码流程：

```cpp
1. Assembler syntax
2. WFE{<c>}{<q>}
3. where:
4. <c>, <q> See Standard assembler syntax fields on page A8-285.
5. Operation
6. if ConditionPassed() then
7.     EncodingSpecificOperations();
8.     if EventRegistered() then
9.         ClearEventRegister();
10.     else
11.         if HaveVirtExt() && !IsSecure() && !CurrentModeIsHyp() && HCR.TWE == '1' then
12.             HSRString = Zeros(25);
13.             HSRString<0> = '1';
14.             WriteHSR('000001', HSRString);
15.             TakeHypTrapException();
16.         else
17.             WaitForEvent();
18. Exceptions
19. Hyp Trap.
```

看了上面这段 WFE 的伪代码，一目了然，首先判断 `ConditionPassed()`，这些函数大家可以在 arm 手册中查看其详细含义，如果 `EventResigerted()` 函数为真，也就是这个 1 bit 的寄存器为真，那么就清除此 bit，然后退出返回，不会让 cpu 进入 low power state；

如果不是异常处理，`TakeHypTrapException()`，那么就 `WaitForEvent()`，等待唤醒事件到来，到来了，就唤醒当前 cpu。

为什么有事件来了就直接返回呢，因为 WFE 的设计认为，如果此时有 event 事件，那么说明当前 cpu 要干活，那就没必要进入 low power state 模式。

如果没有事件产生，那么就可以进入 low power state 模式，因为 cpu 反正也是在等待锁，此时也干不了别的事情，还不如休眠还可以降低功耗。

当然，irq，fiq 等很多中断都可以让 WFE 休眠的 cpu 唤醒，那么这样做还有什么意义呢？比如说时钟中断是一直产生的，那么 cpu 很快就醒了啊，都不用等到发 SEV，那么既然是用 `spin_lock`，也可以在中断上半部使用，也可以在进程上下文，既然是自旋锁，就意味着保护的这段代码是要足够精简，不希望被其他东西打断，那么如果你保护的这部分代码非常长，这时候整个系统响应很可能会变慢，因为如果这时候有人也要使用这个锁的话，那么是否保护的这段代码设计上是有问题的。因此用 `spin_lock` 保护的函数尽可能要短，如果长的话可能需要换其他锁，或者考虑下是否真的要这么长的保护措施。

`TakeHypTrapException()`，是进入异常处理

```cpp
1. Pseudocode description of taking the Hyp Trap exception
2. The TakeHypTrapException() pseudocode procedure describes how the processor takes the exception:
3. // TakeHypTrapException()
4. // ======================
5. TakeHypTrapException()
6.     // HypTrapException is caused by executing an instruction that is trapped to Hyp mode as a
7.     // result of a trap set by a bit in the HCR, HCPTR, HSTR or HDCR. By definition, it can only
8.     // be generated in a Non-secure mode other than Hyp mode.
9.     // Note that, when a Supervisor Call exception is taken to Hyp mode because HCR.TGE==1, this
10.     // is not a trap of the SVC instruction. See the TakeSVCException() pseudocode for this case.
11.     preferred_exceptn_return = if CPSR.T == '1' then PC-4 else PC-8;
12.     new_spsr_value = CPSR;
13.     EnterHypMode(new_spsr_value, preferred_exceptn_return, 20);
14. Additional pseudocode functions for exception handling on page B1-1221 defines the EnterHypMode() pseudocode
15. procedure.
```

ClearEventRegister()

Clear the Event Register of the current processor 清除Event Register的bit

EventRegistered()

Determine whether the Event Register of the current processor is set Event Register bit为真，即被设置过

WaitForEvent()

Wait until WFE instruction completes

等待 Events 事件，有任何一个 Event 事件来临，都会唤醒当前被 WFE suspend 下去的 cpu， 如果是 SEV，会唤醒全部被 WFE suspend 下去的cpu。

如下是关于 WFE 的 wake up events 事件描述和列举：

- WFE wake-up events

> The following events are WFE wake-up events:\
> • the execution of an SEV instruction on any processor in the multiprocessor system\
> • a physical asynchronous abort, unless masked by the CPSR.A bit\
> — when HCR.IMO is set to 1, a virtual IRQ interrupt, unless masked by the CPSR.I bit\
> — when HCR.FMO is set to 1, a virtual FIQ interrupt, unless masked by the CPSR.F bit\
> • an asynchronous debug event, if invasive debug is enabled and the debug event is permitted\
> • an event sent by some IMPLEMENTATION DEFINED mechanism.\
> • a physical IRQ interrupt, unless masked by the CPSR.I bit\
> • a physical FIQ interrupt, unless masked by the CPSR.F bit\
> — when HCR.AMO is set to 1, a virtual asynchronous abort, unless masked by the CPSR.A bit\
> • in Non-secure state in any mode other than Hyp mode:\
> • an event sent by the timer event stream, see Event streams on page B8-1934\
> In addition to the possible masking of WFE wake-up events shown in this list, when invasive debug is enabled and DBGDSCR\[15:14\] is not set to 0b00, DBGDSCR.INTdis can mask interrupts, including masking them acting as WFE wake-up events. For more information, see DBGDSCR, Debug Status and Control Register on page C11-2206.
>
> NoteFor more information about CPSR masking see Asynchronous exception masking on page B1-1181. If the configuration of the masking controls provided by the Security Extensions, or Virtualization Extensions, mean that a CPSR mask bit cannot mask the corresponding exception, then the physical exception is a WFE wake-up event, regardless of the value of the CPSR mask bit.
>
> As shown in the list of wake-up events, an implementation can include IMPLEMENTATION DEFINED hardware mechanisms to generate wake-up events.

接下来我们看下 WFI 的伪代码：

```cpp
1. Assembler syntax
2. WFI{<c>}{<q>}
3. where:
4. <c>, <q> See Standard assembler syntax fields on page A8-285.
5. Operation
6. if ConditionPassed() then
7.     EncodingSpecificOperations();
8.     if HaveVirtExt() && !IsSecure() && !CurrentModeIsHyp() && HCR.TWI == '1' then
9.         HSRString = Zeros(25);
10.         HSRString<0> = '1';
11.         WriteHSR('000001', HSRString);
12.         TakeHypTrapException();
13.     else
14.         WaitForInterrupt();
15. Exceptions
16. Hyp Trap.
```

相关解释：

> WFI\
> Wait For Interrupt is a hint instruction that permits the processor to enter a low-power state until one of a number of asynchronous events occurs. For more information, see Wait For Interrupt on page B1-1200.

> In an implementation that includes the Virtualization Extensions, if HCR.TWI is set to 1, execution of a WFE instruction in a Non-secure mode other than Hyp mode generates a Hyp Trap exception if, ignoring the value of the HCR.TWI bit, conditions permit the processor to suspend execution. For more information see Trapping use of the WFI and WFE instructions on page B1-1249.

以上我们可以看出，WFI 并没有去判断 event register，因此看伪代码可以很直观的看出 WFI 与 WFE 的区别。

以上是我个人的理解，如果有表述不精准或者不准确的地方还请大家指出，欢迎交流。

## 5 参考资料

- [ARM 架构下 spinlock 原理 (代码解读)](https://blog.csdn.net/adaptiver/article/details/72389453)
- [ARM WFI 和 WFE 指令](http://www.wowotech.net/armv8a_arch/wfe_wfi.html)
- [ARM Software development tools](http://infocenter.arm.com/help/index.jsp?lang=en)
- [Linux 内核自旋锁 `spinlock_t` 机制](https://www.jianshu.com/p/f0d6e7103d9b)

标签: [ARM](http://www.wowotech.net/tag/ARM) [wfe](http://www.wowotech.net/tag/wfe) [wfi](http://www.wowotech.net/tag/wfi)

______________________________________________________________________

« [load_balance函数代码详解](http://www.wowotech.net/process_management/load_balance_function.html) | [CFS任务放置代码详解](http://www.wowotech.net/process_management/task_placement_detail.html)»

**评论：**

**[bsp](http://www.wowotech.net/)**\
2022-02-14 18:07

一个问题：\
1、cpu idle在arm的实现，cpu会 disable local irq后进入wfi，此时能否被irq唤醒？ 唤醒后 是往下走还是进入irq handler？

看了下wfi的实现：\
首先cpu disable local irq后进入wfi，肯定能被irq唤醒。唤醒后，是往下面的指令走，常用来做cpu idle。\
如果cpu enable了local irq后进入wfi，被irq唤醒后是 进入irq handler，处理完后才会回到原来的地方。

[回复](http://www.wowotech.net/armv8a_arch/499.html#comment-8555)

**[bsp](http://www.wowotech.net/)**\
2022-02-14 19:40

@bsp：首先cpu disable local irq后进入wfi，肯定能被irq唤醒。唤醒后，是往下面的指令走，常用来做cpu idle。local_irq_enable后 再进入irq handler。

[回复](http://www.wowotech.net/armv8a_arch/499.html#comment-8556)

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

  - ja\
    [@dream：我看完這段也有相同的想法，引用 @dream ...](http://www.wowotech.net/kernel_synchronization/spinlock.html#8922)
  - 元神高手\
    [围观首席power managerment专家](http://www.wowotech.net/pm_subsystem/device_driver_pm.html#8921)
  - 十七\
    [内核空间的映射在系统启动时就已经设定好，并且在所有进程的页表...](http://www.wowotech.net/process_management/context-switch-arch.html#8920)
  - lw\
    [sparse模型和disconti模型没看出来有什么本质区别...](http://www.wowotech.net/memory_management/memory_model.html#8919)
  - 肥饶\
    [一个没设置好就出错](http://www.wowotech.net/linux_kenrel/516.html#8918)
  - orange\
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

  - [Linux内核的整体架构](http://www.wowotech.net/linux_kenrel/11.html)
  - [使用pxe方式安装系统](http://www.wowotech.net/linux_application/288.html)
  - [CFS任务放置代码详解](http://www.wowotech.net/process_management/task_placement_detail.html)
  - [进程切换分析（3）：同步处理](http://www.wowotech.net/process_management/scheudle-sync.html)
  - [DRAM 原理 3 ：DRAM Device](http://www.wowotech.net/basic_tech/321.html)

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
