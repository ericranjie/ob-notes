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

作者：[schspa](http://www.wowotech.net/author/542) 发布于：2021-11-15 19:42 分类：[ARMv8A Arch](http://www.wowotech.net/sort/armv8a_arch)

## Table of Contents

- [在Linux内核中看到下面这句话:](http://www.wowotech.net/admin/#orgd3d6602)
- [先看看ARM64平台原子操作实现原理](http://www.wowotech.net/admin/#org4a2bcdd)
  - [linux](http://www.wowotech.net/admin/#orgf810d5a)
  - [ARM64平台原子操作指令](http://www.wowotech.net/admin/#org0e5e96b)
- [ARMV8a中的设置](http://www.wowotech.net/admin/#org921b556)
  - [ARMv8a手册](http://www.wowotech.net/admin/#org9a6d2c0)
  - [Crotex A55手册](http://www.wowotech.net/admin/#org72fbc95)
  - [CPUECTLR.ATOM](http://www.wowotech.net/admin/#org7767aac)
- [总线的设置](http://www.wowotech.net/admin/#org8bf72ef)
- [对于内核注释中提到的omap平台，硬件上使用了哪种互连网络，最终导致了系统无法在non-cached内存中使用原子操作？](http://www.wowotech.net/admin/#org550476f)
- [DONE 对于上述平台，如果软件上去使用原子操作指令访问non-cached内存，会有什么后果？](http://www.wowotech.net/admin/#org68d8a06)

## 在Linux内核中看到下面这句话:

> At least on ARM, pgprot_noncached causes the\
> memory to be mapped strongly ordered, and atomic operations on strongly ordered\
> memory are implementation defined, and won't work on many ARMs such as omaps.

所以, 为什么对用户non-cached的内存,部分平台不支持原子操作?

## 先看看ARM64平台原子操作实现原理

先看看Linux内核中的实现。

### linux

- atomic_read & atomic_set

  #define atomic_read(v)			READ_ONCE((v)->counter)
  #define atomic_set(v, i)		WRITE_ONCE(((v)->counter), (i))

  对于读和写，在ARM平台上使用正常使用的读写操作即可。

- atomic_add & atomic_dec\
  对于加减的原子操作，由于需要执行读，改，写三步，需要使用特殊的指令才可以实现。

  static inline void
  atomic_add(int i, atomic_t \*v)
  {
  kasan_check_write(v, sizeof(\*v));
  arch_atomic_add(i, v);
  }
  #define atomic_add atomic_add

  ATOMIC_OP(atomic_add)

  #define ATOMIC_OP(op)									\
  static inline void arch\_##op(int i, atomic_t \*v)	\
  {													\
  \_\_lse_ll_sc_body(op, i, v);					\
  }

  #define \_\_lse_ll_sc_body(op, ...)				\
  ({											\
  system_uses_lse_atomics() ?			\
  \__lse_##op(__VA_ARGS__) :		\
  \__ll_sc_##op(__VA_ARGS__);		\
  })

- Linux atomic指令的两种实现

  - LSE

    使用ARMv8.1中新增加的原子操作指令

    #define ATOMIC_OP(op, asm_op)									\
    static inline void \__lse_atomic_##op(int i, atomic_t \*v)	\
    {															\
    asm volatile(											\
    "	" #asm_op "	%w\[i\], %\[v\]\\n"				\
    : \[i\] "+r" (i), \[v\] "+Q" (v->counter)		\
    : "r" (v));								\
    }

    ```
      ATOMIC_OP(andnot, stclr)
      ATOMIC_OP(or, stset)
      ATOMIC_OP(xor, steor)
      ATOMIC_OP(add, stadd)

      static inline void __lse_atomic64_sub(s64 i, atomic64_t *v)
    ```

    {
    asm volatile(
    "	neg	%\[i\], %\[i\]\\n"
    "	stadd	%\[i\], %\[v\]"
    : \[i\] "+&r" (i), \[v\] "+Q" (v->counter)
    : "r" (v));
    }

    从上可以看到，系统使用了单条指令stadd就完成了原子加操作，这些指令是ARMv8.1 添加的指令，并不是所有的AARCH64都支持这种指令。

  - LL_SC\]\] Load-link/store-condiitional

    #define ATOMIC_OP(op, asm_op, constraint)								\
    static inline void													\
    __ll_sc_atomic_##op(int i, atomic_t \*v)								\
    {																	\
    unsigned long tmp;												\
    int result;														\
    \
    asm volatile("// atomic_" #op "\\n"								\
    \_\_LL_SC_FALLBACK(									\
    "	prfm	pstl1strm, %2\\n"		\
    "1:	ldxr	%w0, %2\\n"			\
    "	" #asm_op "	%w0, %w0, %w3\\n"	\
    "	stxr	%w1, %w0, %2\\n"			\
    "	cbnz	%w1, 1b\\n")				\
    : "=&r" (result), "=&r" (tmp), "+Q" (v->counter)	\
    : \_\_stringify(constraint) "r" (i));				\
    }

    ```
      #define ATOMIC_OPS(...)							\
    ```

    ATOMIC_OP(__VA_ARGS__)						\
    ATOMIC_OP_RETURN(        , dmb ish,  , l, "memory", __VA_ARGS__)\
    ATOMIC_OP_RETURN(\_relaxed,        ,  ,  ,         , __VA_ARGS__)\
    ATOMIC_OP_RETURN(\_acquire,        , a,  , "memory", __VA_ARGS__)\
    ATOMIC_OP_RETURN(\_release,        ,  , l, "memory", __VA_ARGS__)\
    ATOMIC_FETCH_OP (        , dmb ish,  , l, "memory", __VA_ARGS__)\
    ATOMIC_FETCH_OP (\_relaxed,        ,  ,  ,         , __VA_ARGS__)\
    ATOMIC_FETCH_OP (\_acquire,        , a,  , "memory", __VA_ARGS__)\
    ATOMIC_FETCH_OP (\_release,        ,  , l, "memory", __VA_ARGS__)

    ATOMIC_OPS(add, add, I)
    ATOMIC_OPS(sub, sub, J)

    从这里的实现可一看到，系统同过ldxr和stxr指令对配和算数运算指令一同完成原子操作。

### ARM64平台原子操作指令

A64: ldx, ldax,stx,stlx\
A32/T32: ldrex, strex, ldaex, stlex

- 从上面Linux的实现中就可以得知，在ARMv8中有对于原子操作有两种不同得实现，一种是LLSC形式的原子操作，另一种是LSE

## ARMV8a中的设置

### ARMv8a手册

[ARMV8a中对于原子操作的描述](http://www.wowotech.net/admin/assets/screenshot_20191004_112725.png)\
以上地方仅仅描述了原子操作指令使用时需要注意的地方，并无法找到我们的答案，下面去看看CPU手册吧。

### Crotex A55手册

[Crotex-A55-Atomic-Operation](http://www.wowotech.net/admin/assets/screenshot_20191004_120904.png)

- 从上面可以看到,在ARMv8中, 对于cacheable memory, 原子操作都是没有问题的, 因为系统可以通过cache来完成原子操作.

- 对于devices或者non-cacheable内存, 原子操作依赖于互联网络的支持. 在arm上就是各种AMBA总线,如果互联网络不支持的话,就会引发同步或者异步的异常.

  从以上信息可知，对于部分non-cacheable内存，在ARM平台上，不支持原子操作的原因是因为硬件的互连网络不支持原子操作。

- TODO 为什么store atomics会引发异步的异常? 而不是同步异常?

### CPUECTLR.ATOM

![CPUECTLR-ATOM.png](https://schspa.tk/2019/09/22/assets/CPUECTLR-ATOM.png)

通过CPU的这个寄存器可以控制atomic访问的时候具体是使用near/far，默认的配置中，根据不同的情况，硬件一般会自动根据cache hit的情况自动切换，这之中并不需要软件的参与。

## 总线的设置

关于AMBA总线:\
参考 `代码改变世界ctw` 的文章，可以对AMBA总线有个大概的了解\
[https://blog.csdn.net/weixin_42135087/article/details/111557929](https://blog.csdn.net/weixin_42135087/article/details/111557929)\
在总线上，为了exclude access，硬件上有一套具体的协议来支持，并且有相应信号。\
[AMBA AXI: Atomic transaction support](https://developer.arm.com/documentation/102202/0200/Atomic-accesses)\
![2021-11-03_20-51.png](https://schspa.tk/2019/09/22/assets/2021-11-03_20-51.png)\
从上面arm官方的示意图中，AMBA中 exclusive access monitor 会存储传输的id和地址，由此来监控原子传输。

## 对于内核注释中提到的omap平台，硬件上使用了哪种互连网络，最终导致了系统无法在non-cached内存中使用原子操作？

由于没有具体的OMAP平台资料，由上述的信息可以得知，这个是由于SOC内部的总线，或者最后端内存的硬件实现而造成的。不光是OMAP，很多ARM平台的SOC都有相同的问题. 但是一般cache都是打开的状态，所以软件一般不需要关心这个问题。

## DONE 对于上述平台，如果软件上去使用原子操作指令访问non-cached内存，会有什么后果？

出现问题之后cpu会进入同步异常\
Data abort with DFSC:\
0b110101 implementation defined fault (Unsupported Exclusive or Atomic access).

esr_el3        = 0x0000000096000035

标签: [原子操作](http://www.wowotech.net/tag/%E5%8E%9F%E5%AD%90%E6%93%8D%E4%BD%9C) [atomic](http://www.wowotech.net/tag/atomic) [arm64](http://www.wowotech.net/tag/arm64) [aarch64](http://www.wowotech.net/tag/aarch64)

[![](http://www.wowotech.net/content/uploadfile/201605/ef3e1463542768.png)](http://www.wowotech.net/support_us.html)

« [CFS任务的负载均衡（load balance）](http://www.wowotech.net/process_management/load_balance_detail.html) | [irq wakeup in linux](http://www.wowotech.net/pm_subsystem/491.html)»

**评论：**

**[wahaha02](http://https//www.cnblogs.com/wahaha02/)**\
2021-11-18 19:56

最近也刚好遇到这个问题。后来翻阅AMR标准手册，其中只定义了LDREX and STREX operations shall only be performed on memory supporting the Normal memory attribute。后来也咨询了CPU厂家，对于non-cached attribute内存上的原子操作，不能使用独占访存指令，按他们建议，改用了hwspinlock实现。

[回复](http://www.wowotech.net/armv8a_arch/492.html#comment-8376)

**raceant**\
2022-04-28 22:00

@wahaha02：hwspinlock 是怎样实现的呢？

[回复](http://www.wowotech.net/armv8a_arch/492.html#comment-8606)

**[schspa](http://www.wowotech.net/)**\
2022-09-06 19:19

@raceant：hwspinlock是外部IP提供的（不同厂家的IP会有不同），每次操作都需要通过MMIO来方位外部寄存器(Device nGnRnE)，速度会慢的多. 这种的每次访问都不会经过cache.

[回复](http://www.wowotech.net/armv8a_arch/492.html#comment-8671)

**[schspa](http://www.wowotech.net/)**\
2022-09-06 19:19

@raceant：hwspinlock是外部IP提供的（不同厂家的IP会有不同），每次操作都需要通过MMIO来方位外部寄存器(Device nGnRnE)，速度会慢的多. 这种的每次访问都不会经过cache.

[回复](http://www.wowotech.net/armv8a_arch/492.html#comment-8672)

**[schspa](http://www.wowotech.net/)**\
2022-09-06 19:19

@raceant：hwspinlock是外部IP提供的（不同厂家的IP会有不同），每次操作都需要通过MMIO来方位外部寄存器(Device nGnRnE)，速度会慢的多. 这种的每次访问都不会经过cache.

[回复](http://www.wowotech.net/armv8a_arch/492.html#comment-8673)

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

  - [基于Hikey的"Boot from USB"调试](http://www.wowotech.net/x_project/hikey_usb_boot.html)
  - [linux kernel的中断子系统之（四）：High level irq event handler](http://www.wowotech.net/irq_subsystem/High_level_irq_event_handler.html)
  - [eMMC 原理 3 ：分区管理](http://www.wowotech.net/basic_tech/emmc_partitions.html)
  - [关于numa loadbance的死锁分析](http://www.wowotech.net/linux_kenrel/482.html)
  - [X-000-PRE-开发环境搭建](http://www.wowotech.net/x_project/develop_env.html)

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
