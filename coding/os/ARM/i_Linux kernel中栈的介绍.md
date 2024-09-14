
原创 代码改变世界ctw 人人极客社区

 _2021年12月05日 21:14_

## linux kernel中的中断irq的栈stack

### arm64体系的irq的栈

先看下irq_handler的中断函数处理的过程：

`/*    * Interrupt handling.    */    .macro irq_handler    ldr_l x1, handle_arch_irq   -----将handle地址保存在x1    mov x0, sp    irq_stack_entry   ------ 切换栈，也就是将svc栈切换程irq栈. 在此之前，SP还是EL1_SP，在此函数中，将EL1_SP保存，再将IRQ栈的地址写入到SP寄存器    blr x1     ——————执行中断处理函数    irq_stack_exit   ------ 恢复EL1_SP(svc栈)    .endm   `

 `.macro irq_stack_entry    mov x19, sp   // preserve the original sp    //将svc mode下的栈地址(也就是EL1_SP)保存到x19       /*     * Compare sp with the base of the task stack.     * If the top ~(THREAD_SIZE - 1) bits match, we are on a task stack,     * and should switch to the irq stack.     */   #ifdef CONFIG_THREAD_INFO_IN_TASK    ldr x25, [tsk, TSK_STACK]    eor x25, x25, x19    and x25, x25, #~(THREAD_SIZE - 1)    cbnz x25, 9998f   #else    and x25, x19, #~(THREAD_SIZE - 1)    cmp x25, tsk    b.ne 9998f   #endif       adr_this_cpu x25, irq_stack, x26    mov x26, #IRQ_STACK_START_SP     //IRQ_STACK_START_SP这是irq mode的栈地址    add x26, x25, x26       /* switch to the irq stack */    mov sp, x26     //将irq栈地址，写入到sp       /*     * Add a dummy stack frame, this non-standard format is fixed up     * by unwind_frame()     */    stp     x29, x19, [sp, #-16]!    mov x29, sp      9998:    .endm`

`/*    * x19 should be preserved between irq_stack_entry and    * irq_stack_exit.    */   .macro irq_stack_exit   mov sp, x19     //x19保存着svc mode下的栈，也就是EL1_SP   .endm   `

那么irq的栈在哪设置的，多大呢？

在irq.h中定义了，irq栈的地址和size。

`#define IRQ_STACK_SIZE   THREAD_SIZE   #define IRQ_STACK_START_SP  THREAD_START_SP   `

thread_info.h中定义了大小。

`define THREAD_SIZE  16384     //也就是irq栈的大小大概15k   #define THREAD_START_SP  (THREAD_SIZE - 16)    //也就是irq栈的首地址是从"0地址+15k"这个地方开始的   `

## linux kernel中的栈stack

### 概念介绍：内核栈、内核空间进程栈、用户空间进程栈

**内核栈**：

在linux最开始启动的时候，系统只有一个进程（更准确的说是kernel thread），就是PID等于0的那个进程，叫做swapper进程（或者叫做idle进程）。

**用户空间进程栈、内核空间进程栈**：

对于一个应用程序而言，可以运行在用户空间，也可以通过系统调用进入内核空间。在用户空间，使用的是用户栈，也就是我们软件工程师编写用户空间程序的时候，保存局部变量的stack。陷入内核后，当然不能用用户栈了，这时候就需要使用到内核栈。所谓内核栈其实就是处于SVC mode时候使用的栈。

## 内核栈的实现

内核栈是静态定义的，如下：

`（kernel-4.14/arch/arm/include/asm/thread_info.h）     #define init_thread_info (init_thread_union.thread_info)     #define init_stack  (init_thread_union.stack)   `

`(kernel-4.14/include/linux/sched.h)   union thread_union {   #ifndef CONFIG_THREAD_INFO_IN_TASK    struct thread_info thread_info;   #endif    unsigned long stack[THREAD_SIZE/sizeof(long)];   };   `

## 内核进程栈、用户进程栈 的实现

Linux kernel在内核线程，或用户线程时都会分配一个page frame，具体代码如下：

`(kernel-4.14/kernel/fork.c)   static struct task_struct *dup_task_struct(struct task_struct *orig, int node)   {   ......    stack = alloc_thread_stack_node(tsk, node);    if (!stack)     goto free_tsk;   ......   }   `

底部是struct thread_info数据结构，顶部（高地址）就是该进程的栈了。

当进程切换的时候，整个硬件和软件的上下文都会进行切换，这里就包括了svc mode的sp寄存器的值被切换到调度算法选定的新的进程的内核栈上来。

## 总结

linux kernel arm64中定义的irq栈，在内存"首地址"处，大小16k. irq_hander使用irq栈。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E "虚线阴影分割线")

![](http://mmbiz.qpic.cn/mmbiz_png/9sNwsXcN68pL55XIyzTrCHZTbIUdTibQcuzuCaYeGTXNMyn6ACmicUrpoDC0xZSap46XJ59sKysPg9Rg379f32cA/300?wx_fmt=png&wxfrom=19)

**人人极客社区**

工程师们自己的Linux底层技术社区，分享体系架构、内核、网络、安全和驱动。

316篇原创内容

公众号

5T技术资源大放送！包括但不限于：C/C++，Arm, Linux，Android，人工智能，单片机，树莓派，等等。在上面的【人人都是极客】公众号内回复「peter」，即可免费获取！！  

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E) **记得点击****分享****、****赞****和****在看****，给我充点儿电吧**

Linux93

Linux中断管理9

Linux · 目录

上一篇驱动加载的本质下一篇没有IOMMU的DMA操作

阅读 3155

​

写留言

**留言 5**

- 哲思
    
    2021年12月31日
    
    赞
    
    当时钟中断后，调用schedule，这时CPU的指针和栈切换到了新的进程上，那么此时的中断栈呢，这时候中断栈还没有释放。请问这块怎么理解？？
    
    人人极客社区
    
    作者2021年12月31日
    
    赞
    
    和架构设计有关，arm64的中断栈留在原地
    
    哲思
    
    2021年12月31日
    
    赞
    
    那如果是X86的栈呢？如果栈中还保存着上次的内容，新的中断来后，栈中还有上次的残留，会有影响吧
    
- 逸
    
    2021年12月5日
    
    赞
    
    希望可以出一期内核调试的
    
    人人极客社区
    
    作者2021年12月5日
    
    赞
    
    之前有出过，可以号里搜索下哈，如果没有的话欢迎提出具体的需求![[偷笑]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    

已无更多数据

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/9sNwsXcN68pL55XIyzTrCHZTbIUdTibQcuzuCaYeGTXNMyn6ACmicUrpoDC0xZSap46XJ59sKysPg9Rg379f32cA/300?wx_fmt=png&wxfrom=18)

人人极客社区

913

5

写留言

**留言 5**

- 哲思
    
    2021年12月31日
    
    赞
    
    当时钟中断后，调用schedule，这时CPU的指针和栈切换到了新的进程上，那么此时的中断栈呢，这时候中断栈还没有释放。请问这块怎么理解？？
    
    人人极客社区
    
    作者2021年12月31日
    
    赞
    
    和架构设计有关，arm64的中断栈留在原地
    
    哲思
    
    2021年12月31日
    
    赞
    
    那如果是X86的栈呢？如果栈中还保存着上次的内容，新的中断来后，栈中还有上次的残留，会有影响吧
    
- 逸
    
    2021年12月5日
    
    赞
    
    希望可以出一期内核调试的
    
    人人极客社区
    
    作者2021年12月5日
    
    赞
    
    之前有出过，可以号里搜索下哈，如果没有的话欢迎提出具体的需求![[偷笑]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    

已无更多数据