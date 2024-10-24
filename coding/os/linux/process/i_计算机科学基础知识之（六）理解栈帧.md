
作者：[linuxer](http://www.wowotech.net/author/3 "linuxer") 发布于：2015-3-12 13:00 分类：[基础学科](http://www.wowotech.net/sort/basic_subject)

# 一、前言

本文以一个简单的例子来描述ARM linux下的stack frame。

本文也是对tigger网友问题的回复。

# 二、源代码

> #include \<stdio.h>
>
> static int static_interface_leaf( int x, int y )\
> {\
> int tmp0 = 0x12;\
> int tmp1 = 0x34;\
> int tmp2 = 0x56;
>
> tmp0 = x;\
> tmp1 = y;
>
> return (tmp0+tmp1+tmp2);\
> }
>
> int public_interface_leaf( int x, int y )\
> {\
> int tmp0 = 0x12;\
> int tmp1 = 0x34;\
> int tmp2 = 0x56;
>
> tmp0 = x;\
> tmp1 = y;
>
> return (tmp0+tmp1+tmp2);\
> }
>
> void public_interface( int x )\
> {\
> int tmp0 = 0x12;\
> int tmp1 = 0x34;
>
> tmp0 = x;\
> public_interface_leaf( tmp0, tmp1 );\
> static_interface_leaf( tmp0, tmp1 );\
> }
>
> int main(int argc, char \*\*argv)\
> {\
> int tmp0 = 0x12;
>
> public_interface( tmp0 );
>
> return 0;\
> }

# 三、逐级stack frame分析

## 1、准备知识

根据AAPCS的描述，stack是full-descending并且需要满足两种约束：一种是通用约束，适用所有的场景，另外一种是针对public interface的约束。通用约束有3条：

（1）SP只能访问stack base和stack limit之间的memory，即Stack-limit \< SP \<= stack-base

（2）SP必须对齐在4个字节上，即SP mod 4 = 0

（3）函数只能访问自己能回溯的那些栈帧。例如f1调用f2，而f2函数又调用了f3，那么f3是可以访问自己的stack以及f2和f1的stack，也就是说，函数可以访问\[SP, stack-base – 1\]之间的内容

对public interface的约束多了一条，就是SP必须对齐在8个字节上，即SP mod 8 = 0

关于ARM的ABI，还有一份文档，IHI0046B_ABI_Advisory_1，这份文件中讲到，在调用所有的AAPCS兼容的函数的时候都要求SP是对齐在8个字节上。

## 2、起始点的用户栈的情况

在[静态链接](http://www.wowotech.net/basic_subject/static-link.html)文档中，我们说过，函数的入口函数不是main函数而是_start函数，调用序列是_start()->\_\_libc_start_main()->main()。main函数之前对于所有的程序都是一样的，因此不需要每一个程序员都重复进行那些动作，因此留给程序员一个main函数的入口，开始自己相关逻辑的处理。内核在start函数（我在这里以及后面的文档中省略了下划线）之前的stack frame并不是空的，内核会创建一些资料在stack上，具体如下：

![[Pasted image 20241024221544.png]]

具体怎么在用户栈上建立上面的数据结构，有兴趣的同学可以参考内核的create_elf_tables函数。此外，需要提醒的是这些数据内容虽然在栈上，但是不是stack frame的一部分，有点类似内核空间到用户空间参数传递的味道。为何这么说呢？因为在start函数中有一条汇编指令：mov    fp, #0，该指令清除frame pointer，在debugger做栈的回溯的时候，当fp等于0的时候也就意味着到了最外层函数。

## 3、start函数的start frame

> 0000829c \<\_start>:\
> 829c:    e59fc024     ldr    ip, \[pc, #36\]    ; 82c8 \<.text+0x2c>\
> 82a0:    e3a0b000     mov    fp, #0    ; 0x0－－－－－－－－最外层函数，清除frame pointer\
> 82a4:    e49d1004     ldr    r1, \[sp\], #4－－－－－－－－－－r1 = argc, sp=sp+4，sp指向了argv\[\]\
> 82a8:    e1a0200d     mov    r2, sp－－－－－－－－－－r2保存了stack end，也就是argv\[\]那个位置\
> 82ac:    e52d2004     str    r2, \[sp, #-4\]!－－－－－－－－将stack end压入栈\
> 82b0:    e52d0004     str    r0, \[sp, #-4\]!－－－－－－－－将rtld_fini压入栈\
> 82b4:    e59f0010     ldr    r0, \[pc, #16\]    ; 82cc \<.text+0x30>\
> 82b8:    e59f3010     ldr    r3, \[pc, #16\]    ; 82d0 \<.text+0x34>\
> 82bc:    e52dc004     str    ip, \[sp, #-4\]!－－－－－－－－将fini压入栈\
> 82c0:    ebffffef     bl    8284 \<.text-0x18>－－－－－－－call \_\_libc_start_main\
> 82c4:    ebffffeb     bl    8278 \<.text-0x24>\
> 82c8:    0000848c     .word    0x0000848c\
> 82cc:    00008454     .word    0x00008454\
> 82d0:    00008490     .word    0x00008490

在调用\_\_libc_start_main函数之前，stack frame的情况如下：

[![start_sf](http://www.wowotech.net/content/uploadfile/201503/5a87d992a4fbfda5ba719a4107b5c4c520150312045255.gif "start_sf")](http://www.wowotech.net/content/uploadfile/201503/09b27cb760a7c2827560acb1c0197db420150312045151.gif)

大家可以对照上面的汇编和图片，我这里只是描述基本知识点：

1、stack的确是full-descending的，SP指向了start函数的顶部，下一个函数必须先减SP，才能保存其栈上的数据。

2、内核到用户空间当然是public interface，因此在进入start函数的时候SP当前是8字节对齐。而start函数的栈有3个变量共计12个字节，在调用\_\_libc_start_main函数这个public interface的时候当然也要8字节对齐，按理说这里start函数有一个小小的4字节的空洞，但实际上，代码是抹去了用户栈的argc这个参数，因此start的栈的细节如下：

[![ks](http://www.wowotech.net/content/uploadfile/201503/3ef3080ba54612b34981a6ce481daabb20150312045500.gif "ks")](http://www.wowotech.net/content/uploadfile/201503/0714246a903301c7aa0c673c47775c5020150312045358.gif)

虽然抹去了用户栈的argc这个参数，不过没有关系，反正它已经保存在了r1寄存器中了。

4、\_\_libc_start_main函数的stack frame

\_\_libc_start_main是libc定义的符号，我们动态链接的时候，这些代码没有进入我们测试的ELF文件。这里略过吧，毕竟查阅c库代码也是非常烦人的事情。

5、main函数的stack frame

> 00008454
>
> :\
> 8454:    e92d4800     stmdb    sp!, {fp, lr}－－－将上一个函数的 fp和lr寄存器压入stack, sp=sp-8\
> 8458:    e28db004     add    fp, sp, #4    ; －－－上一个函数的sp＋4就是本函数stack frame的开始\
> 845c:    e24dd010     sub    sp, sp, #16    ; 0x10\
> 8460:    e1a03000     mov    r3, r0\
> 8464:    e50b1014     str    r1, \[fp, #-20\]－－－－－－保存argv\
> 8468:    e54b300d     str    r3, \[fp, #-16\]－－－－－－保存argc\
> 846c:    e3a03012     mov    r3, #18    ; 0x12－－－tmp0 = 0x12，\[fp, #-8\]就是源代码的tmp0\
> 8470:    e50b3008     str    r3, \[fp, #-8\]\
> 8474:    e51b0008     ldr    r0, \[fp, #-8\]－－－－－传递tmp0参数\
> 8478:    ebffffe3     bl    840c\
> 847c:    e3a03000     mov    r3, #0    ; 0x0\
> 8480:    e1a00003     mov    r0, r3\
> 8484:    e24bd004     sub    sp, fp, #4    ; 0x4\
> 8488:    e8bd8800     ldmia    sp!, {fp, pc}

在调用public_interface之前，main函数的stack frame如下：

[![main_sf](http://www.wowotech.net/content/uploadfile/201503/de963c4d87aaeabf711d43e932b2b9d420150312045711.gif "main_sf")](http://www.wowotech.net/content/uploadfile/201503/fb507fd5b0e1df6c902bdb333a3912f320150312045605.gif)

对照代码和图片，我们有下面的解释：

（1）第一条指令就是stmdb，这里db就是decrease before的意思，再次确认stack的确是full-descending的

（2）虽然只有一个临时变量tmp0，但是编译器还是传递了argc和argv这两个参数，具体为何我也没有考虑清楚，因此在分配main的stack frame的时候使用了sub    sp, sp, #16，分配4个int型数据，当然是为了对齐8字节。

（3）在一个函数的执行过程中，sp和fp之间就是该函数的stack frame。sp指向stack frame的顶部（低地址），fp指向顶部。

（4）由于main函数的fp加4就是\_\_libc_start_main的sp，因此在main函数的stack上不需要保存其sp，只要保存fp就OK了。

6、public_interface的stack frame

> 0000840c :\
> 840c:    e92d4800     stmdb    sp!, {fp, lr}\
> 8410:    e28db004     add    fp, sp, #4    ; 0x4\
> 8414:    e24dd010     sub    sp, sp, #16    ; 0x10\
> 8418:    e50b0010     str    r0, \[fp, #-16\]－－－－－－－－－中间变量，保存传入的x参数\
> 841c:    e3a03012     mov    r3, #18    ; 0x12\
> 8420:    e50b300c     str    r3, \[fp, #-12\]－－－－－－－－－tmp0 = 0x12\
> 8424:    e3a03034     mov    r3, #52    ; 0x34\
> 8428:    e50b3008     str    r3, \[fp, #-8\]－－－－－－－－－－tmp1 = 0x34\
> 842c:    e51b3010     ldr    r3, \[fp, #-16\]\
> 8430:    e50b300c     str    r3, \[fp, #-12\]－－－－－－－－－tmp0 = x\
> 8434:    e51b000c     ldr    r0, \[fp, #-12\]\
> 8438:    e51b1008     ldr    r1, \[fp, #-8\]\
> 843c:    ebffffda     bl    83ac\
> 8440:    e51b000c     ldr    r0, \[fp, #-12\]\
> 8444:    e51b1008     ldr    r1, \[fp, #-8\]\
> 8448:    ebffffbf     bl    834c\
> 844c:    e24bd004     sub    sp, fp, #4    ; 0x4\
> 8450:    e8bd8800     ldmia    sp!, {fp, pc}

栈帧情况如下：

[![pli_sf](http://www.wowotech.net/content/uploadfile/201503/a669d34714ff667d77f6c8f41562938720150312045918.gif "pli_sf")](http://www.wowotech.net/content/uploadfile/201503/88c2b984e5a95c6a2f258fcb91668b5020150312045816.gif)

这里比较简单，大家自行分析就OK了。

7、调用static函数

根据AAPCS的描述，只有public接口才需要SP 8字节对齐。不过测试程序表明所有的都是8字节对齐的，我的编译器关于ABI的缺省设定是-mabi=aapcs-linux，猜想可能是所有的函数都被编译成AAPCS-comforming fuction。具体大家可以自己写代码练习一下。

参考文献

1、AAPCS。Procedure Call Standard for the ARM Architecture

2、IHI0046B_ABI_Advisory_1。ABI for the ARM Architecture Advisory Note – SP must be 8-byte aligned on entry to AAPCS-conforming functions

_原创文章，转发请注明出处。蜗窝科技_

\_[http://www.wowotech.net/basic_subject/stack-frame.html](http://www.wowotech.net/basic_subject/stack-frame.html)\
\_

标签: [stack](http://www.wowotech.net/tag/stack) [frame](http://www.wowotech.net/tag/frame) [栈帧](http://www.wowotech.net/tag/%E6%A0%88%E5%B8%A7)

[![](http://www.wowotech.net/content/uploadfile/201605/ef3e1463542768.png)](http://www.wowotech.net/support_us.html)

« [Linux power supply class(1)\_软件架构及API汇整](http://www.wowotech.net/pm_subsystem/psy_class_overview.html) | [计算机科学基础知识（五）: 动态链接](http://www.wowotech.net/basic_subject/dynamic-link.html)»

**评论：**

**misakmm20001**\
2017-02-12 18:08

倒数第2张图  argc argv位置是不是有问题  应该在\_\_libc_start_main栈帧上吧？  应该在lr的上边吧？\
\_\_libc_start_main调用main  不应该是先把main的参数压栈  然后继续压lr和fp吗 所以main的参数应该在lr的上面吧？

[回复](http://www.wowotech.net/basic_subject/stack-frame.html#comment-5213)

**[linuxer](http://www.wowotech.net/)**\
2017-02-13 09:05

@misakmm20001：对于ARM32而言，如果参数的数目小于4个，那么其实根本不需要把参数压栈的动作，寄存器r0～r3就完成了参数传递。因此，在\_\_libc_start_main调用main的时候，argc argv并不是在\_\_libc_start_main的栈上，而是通过寄存器传递。不过，在main函数中，会把r0和r1（即argc 和argv）保存在自己的栈上（类似与临时变量），因此，图片上将argc 和argv画在main函数的stack frame上。

[回复](http://www.wowotech.net/basic_subject/stack-frame.html#comment-5215)

**misaka**\
2017-02-13 15:26

@linuxer：谢谢 明白了 估计这个还是有平台相关性的。 应该也存在我说的这种情况。

[回复](http://www.wowotech.net/basic_subject/stack-frame.html#comment-5217)

**[tigger](http://www.wowotech.net/)**\
2015-03-12 20:12

pop current stack\
你是说依靠压栈的时候，把fp也会压了，然后pop 刚进入这个函数时候的sp里面的fp就得到上一个的fp，对不？

[回复](http://www.wowotech.net/basic_subject/stack-frame.html#comment-1333)

**[linuxer](http://www.wowotech.net/)**\
2015-03-13 00:14

@tigger：假设下面的场景A调用B函数：\
A()\
{\
...\
B()\
...\
}\
在B函数的一开始就会把fp_A压入B函数的stack，并且fp_B = sp_A - 4。\
对于B函数而言，sp_B和fp_B之间的内容就是B函数的stack frame，在B函数中想回溯上一级函数（也就是A函数）其实就是求sp_A和fp_A，fp_A在B的stack frame上，访问\[fp_B, #-4\]就可获取fp_A，而sp_A = fp_B + 4

如果有更深入层次的函数调用，重复上面的过程，就完成了栈的回溯。当fp等于0的时候，意外着来到最外层的函数

[回复](http://www.wowotech.net/basic_subject/stack-frame.html#comment-1334)

**[tigger](http://www.wowotech.net/)**\
2015-03-13 10:00

@linuxer：ok got it\
thx

[回复](http://www.wowotech.net/basic_subject/stack-frame.html#comment-1335)

**[tigger](http://www.wowotech.net/)**\
2015-03-12 15:34

老大果然厉害啊，瞬间清晰了不少\
1：8454:    e92d4800     stmdb    sp!, {fp, lr}－－－将上一个函数的 fp和lr寄存器压入stack, sp=sp+8\
这里应该笔误了吧，sp=sp-8\
2：8468:    e54b300d     strb    r3, \[fp, #-13\]－－－－－－保存argc\
这里应该是#-16？\
3：不知道有没有手动unwind过一个堆栈，比如我现在知道一个地址是当前的函数的sp，\
但是我想从这个sp的地址，不断的查看call stack，怎么样能找到这一个函数跟另一函数的交接处呢？

[回复](http://www.wowotech.net/basic_subject/stack-frame.html#comment-1331)

**[linuxer](http://www.wowotech.net/)**\
2015-03-12 18:32

@tigger：1：8454:    e92d4800     stmdb    sp!, {fp, lr}－－－将上一个函数的 fp和lr寄存器压入stack, sp=sp+8\
这里应该笔误了吧，sp=sp-8\
－－－－－－－－－－－－－－－－－－－－－－－－－－\
的确是写错了，应该是向低地址方向移动sp指针

2：8468:    e54b300d     strb    r3, \[fp, #-13\]－－－－－－保存argc\
这里应该是#-16？\
－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－\
^\_^，我的程序有一个小错误，我鬼使神差的把int argc写成char argc了，如果是int argc，当然应该是#-16

3：不知道有没有手动unwind过一个堆栈，比如我现在知道一个地址是当前的函数的sp，\
但是我想从这个sp的地址，不断的查看call stack，怎么样能找到这一个函数跟另一函数的交接处呢？\
－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－\
如果知道当前函数的sp和fp，栈的回溯很简单，通过fp+4可以知道外层函数的顶部（sp），而底部（fp），可以pop current stack获得。

[回复](http://www.wowotech.net/basic_subject/stack-frame.html#comment-1332)

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

  - [Device Tree（一）：背景介绍](http://www.wowotech.net/device_model/why-dt.html)
  - [MMC/SD/SDIO介绍](http://www.wowotech.net/basic_tech/mmc_sd_sdio_intro.html)
  - [启动regulator framework分析任务](http://www.wowotech.net/74.html)
  - [内存初始化（上）](http://www.wowotech.net/memory_management/mm-init-1.html)
  - [编译乱序(Compiler Reordering)](http://www.wowotech.net/kernel_synchronization/453.html)

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
