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

作者：[linuxer](http://www.wowotech.net/author/3 "linuxer") 发布于：2015-3-10 18:15 分类：[基础学科](http://www.wowotech.net/sort/basic_subject)

一、前言

本文以类似hello world这样的简单程序为例，描述了动态连接的概念。第二章描述了整个动态链接的大概过程，随后的两章解析了程序访问动态库中的数据和调用动态库中函数的过程。

注意：阅读本文之前需要先了解[relocatable object file](http://www.wowotech.net/basic_subject/compile-link-load.html)、[静态链接](http://www.wowotech.net/basic_subject/static-link.html)以及[动态库和PIC](http://www.wowotech.net/basic_subject/pic.html)这些内容。

二、动态链接的过程概述

下面的图展示了动态链接的过程：

[![dlp](http://www.wowotech.net/content/uploadfile/201503/8c71843d5d158aad02e40e554938fc0720150310101429.gif "dlp")](http://www.wowotech.net/content/uploadfile/201503/76372e2e6e3da0cc57e4f67b72f495f320150310101327.gif)

Static Linker（对于本文的场景，它就是arm-linux-ld）接收下面的输入：

（1）命令行参数

（2）linker script

（3）各种.o文件，这些.o文件有些是你自己的源程序生成的，有些是系统自带的，例如crt\*.o

（4）各种动态库文件。同样的，可以自己生成动态库，也可以是系统自带的，例如c lib。

Static Linker最终生成了一个ELF格式的可执行程序。要把它变成一个实实在在系统中的进程还需要Dynamic Linker和Loader的协助（对于本文的场景，它应该包括linux kernel、bash以及ld-linux.so）。由于引用了动态库中的符号，static linker生成的可执行程序只完成了部分符号的relocation（各个.o文件之间的相互引用），有些符号仍然没有定位（调用动态库中的符号），这些未定位的符号需要在动态库加载后（确定运行时地址），使用dynamic linker来进行链接、定位，这个过程就是动态链接。对于静态链接，一旦完成了链接，所有的符号就已经确定了run time address，OS只要按照ELF文件中的信息进行加载就好了。对于动态链接的可执行程序，使用static linker进行链接的时候仅仅是完成了部分内容，需要动态链接的符号都还没有确定run time address，static linker仅仅是在可执行文件中嵌入了一些“线索”，在loading的时候，dynamic linker会根据这些“线索”完成剩余的链接工作。

最后再强调一次：虽然在上面的block diagram中共享库出现了两次，但是参与静态链接的时候仅仅是方便static linker进行symbol resolution，在动态链接时候，dynamic linker会真正将其mapping到进程的地址空间。

三、可执行程序访问动态库中的数据

1、source code

我们写一个小程序test.c来访问libfoo.so中的数据（libfoo.so的源代码参考[动态库和位置无关代码](http://www.wowotech.net/basic_subject/pic.html)），代码如下：

> #include \<stdio.h>\
> extern int xxx;\
> extern int yyy;\
> int main(char argc, char\*\* argv)\
> {\
> foo();\
> printf("xxx = %x, yyy = %x \\n", xxx, yyy);\
> }

2、运行模块之间的数据访问如何实现？

运行模块内的符号访问是不需要特别关注的，例如上面的这个程序，如果自己定义的一个全局变量ppp并在main函数中访问。在这种情况下，编译成可执行文件的时候，ppp的地址已经确定了，因此可以main函数的尾部定义一个ppp地址的memory（在.text section），然后使用PC-relative类型的访问获取ppp地址就OK了，static linker在最后生成可执行的ELF文件的时候，会把ppp的地址写入code segment，一切都很简单。但是，运行模块之间的符号访问（例如test.c程序访问libfoo动态库模块的xxx全局变量）的情况是怎样的呢？

我们不看结果，先自己思考一下，然后检查自己思考的是否正确。

由于xxx符号没有确定running address，考虑通过GOT来完成对xxx的访问。static linker生成动态链接的test可执行程序的时候，应该可以确定GOT的地址以及xxx符号在GOT中的offset，这时候，static linker会在data segment中创建一个GOT，并且包括一个关于xxx符号的entry（当然，这时候，GOT中的xxx符号的entry不可能写入正确的xxx地址，都还没有确定呢）。虽然编译的时候我们不知道xxx的运行地址（动态库libfoo可以被加载到任何的地址），但是dynamic linker知道啊，因此，在loading test这个程序的时候，dynamic linker可以改写GOT中xxx符号的entry，把真实的runtime地址写入就OK了。

看起来很完美，不过我们还可以进一步思考一下动态库中的共享情况。多个程序要加载libfoo动态库的时候，正文段的共享是没有问题的，因为是read only的，虽然加载到不同程序的不同的虚拟地址上去，但是通过页表可以mapping到相同的物理地址上，因此，所有进程的libfoo动态库的code segment只要copy一次就OK了。不过libfoo的data segment是RW的，因此无法在多个进程中共享，怎么办？每个进程都会将libfoo的data segment mapping到自己的地址空间，但设定为Read only，在进程修改该memory的内容的时候，产生异常，这时候分配物理内存，copy，建立页表，也就是是利用linux 的COW（copy-on-write）技术，可以实现各个进程自己特定的动态库数据区。

3、观察实际的情况

我们看看在程序中是如何访问xxx符号的：

> 000085cc
>
> :\
> ……\
> 85e8:    e59f3020     ldr    r3, \[pc, #32\]    ; 8610 \<.text+0xf4>\
> 85ec:    e5932000     ldr    r2, \[r3\]\
> ……\
> 8610:    000107ec     .word    0x000107ec\
> ……

看起来这段代码和我们想像的有些差距，看起来xxx这个符号被安排在本程序的bss section（0x000107ec这个地址属于bss section），从section table中可以看出来这一点：

> \[23\] .bss              NOBITS          **000107e8** 0007e8 **00000c** 00  WA  0   0  4

起始地址是0x107e8开始的长度为0xc的区域属于bss section。我们在上一节中所有美好的想像都崩塌了。难道我们需要对xxx这个符号进行重定位吗？好吧，我们来看看test的重定位信息，在.rel.dyn section中：

> Relocation section '.rel.dyn' at offset 0x478 contains 3 entries:\
> Offset     Info    Type            Sym.Value  Sym. Name\
> 000107dc  00001415 R_ARM_GLOB_DAT    00000000   __gmon_start__\
> 000107e8  00000114 R_ARM_COPY        000107e8   yyy\
> **000107ec  00000614 R_ARM_COPY        000107ec   xxx**

R_ARM_COPY这种类型的重定位信息只是用于ELF可执行文件，dynamic linker看到R_ARM_COPY这种类型的重定位信息就知道是在定位动态库中的一个符号，这时候，dynamic linker会copy指定size（动态连接符号表.dynsym中有该符号的size）的动态库中的memory到目标地址（对应xxx这个场景就是0x107ec，也就是bss section中的xxx符号）。copy之后，dynamic linker还会做一件事情，就是把所有访问该符号（包括动态库）的进行重定位，让这些代码使用0x107ec来访问xxx这个符号。在[动态库和位置无关代码](http://www.wowotech.net/basic_subject/pic.html)文档中，我们知道，动态库代码访问xxx变量也是通过GOT进行的，dynamic linker将xxx符号对应的GOT Entry修改成0x107ec即可。

OK，我们根据实际的观察可以得出结论：动态库中的data segemnt中的data和bss section中的数据并不会直接被进程中的代码访问，虽然它们被mapping到了进程的地址空间中去，它们的唯一的作用是作为initial data copy，也就是说，每次一个依赖该动态库的新进程loading，动态库被mapping到进程，当进程实际访问动态库中的data的时候，实际上并没有直接引用到动态库data segment mapping的那个虚拟地址上去，实际上，进程也会分配这些内存，但是这些内存的内容会在被访问之前用动态库中的initial copy来填充。

上节中我们思考的方法虽然可行，但是用COW技术导致了开销。

四、可执行程序调用动态库中的函数

1、引言

源代码还是上一章的代码，只不过我们重点关注foo函数的调用。

> ……\
> 85e4:    ebffffc3     bl    84f8 \<.text-0x24>\
> ……

我们知道，bl是PC-relative的，代码执行到这里会跳转到.text-0x24这个位置，这是一个什么样的神秘东东呢？我们看看test的program header就会明白了：

> ……\
> LOAD           0x000000 0x00008000 0x00008000 0x006c0 0x006c0 R E 0x8000 ------code segment\
> LOAD           0x0006c0 0x000106c0 0x000106c0 0x00128 0x00134 RW  0x8000------data segment\
> ……
>
> Code Segment mapping:\
> …… .rel.dyn .rel.plt .init **.plt .text** .fini .rodata .ARM.exidx .eh_frame \
> ……

code segment由若干个section组成，.text-0x24实际上会涉及.text section前面的那个section，也就是.plt。

2、什么是PLT（Procedure Linkage Table）?

首先我们先聊一聊为何会有PLT？它的目的是什么？难道有了GOT还不够，还要用PLT这样的概念持续轰炸可怜的码农？当然，对于PIC code而言，如果访问本运行模块内部的函数，那么仅仅使用GOT而不使用PLT也是OK的。由于是编译目标是位置无关，因此，传递给gcc的参数包含-fPIC这样的option，这时候，gcc在将一个个.c文件编译成.o文件的时候，对所有的全局符号（函数和变量）都使用GOT。我们可以用函数调用为例，对这些全局符号进行分类。假设一个动态库D，其由a.c b.c和c.c三个编译模块组成，那么全局的函数符号调用分成两种：

（1）该动态库D内部定义了该函数符号。例如a.c模块调用了b.c模块的bb函数

（2）该动态库D没有定义该函数符号，该符号来自其他动态库

对上面两种符号都只使用GOT，不用PLT也是OK的，方法如下：首先获取GOT首地址（紧跟代码，增加一个.word来保存该值，虽然gcc编译的时候，got首地址还不知道，但是static linker会知道并修改这个值），和该函数在GOT中的偏移（方法同上，也是紧跟代码，增加一个.word来保存该值），取出该函数地址，将控制权交给该函数。是不是觉得稍微麻烦一些，不如bl那么直接。但是使用GOT就是这样，没有办法。实际上，对于第一类别的情况，在静态链接阶段，static linker实际上知道a.c模块中调用了bb函数指令和bb函数之间的offset，因此可以直接使用bl，从而省略了上面那么复杂的过程，但是在编译成.o文件的时候，gcc哪里知道bb是一个外部动态库的符号，还是本动态库其他编译模块的符号呢？也只能是统一处理。

但是，实际的情况不都是动态库，我们的项目一般是有主程序和多个动态库模块组成，对于主程序模块，都不会编译成PIC的（一般而言），对于这种non-PIC的场景，我们可以用GOT包打天下吗？答案是：不行，让我们来看看这个场景分析。在编译主程序的各个.c文件的时候，由于没有-fPIC的参数，因此函数调用都是被编译成：

> ebfffffe        bl      0

gcc没有那么聪明，它就是按照command line传递的参数工作，没有-fPIC的参数，就一律使用bl。在链接的时候，问题来了，如果xxxx函数是一个外部的符号，static linker根本不知道其运行地址是什么，这时候怎么办？bl指令就是跳转到相对PC的一个地址去继续执行程序，这时候跳转到GOT可以吗？可以是可以，但是dynamic linker不能直接写入xxxx的地址而是要写入一段代码，这样GOT中的内容就不纯粹了，因此这些跳转的代码被移除到另外的一个section，用来协同完成动态符号定位，而保存这些代码的section就是PLT。

OK，虽然对于PIC而言，不使用PLT是OK的，但是有开销。想像一下：一个动态库中有80处的函数调用，那么每个原来可以用1条bl实现函数调用的地方，需要使用4条指令来展开，更重要的是：代码需要重复80次。因此，实际上，即使gcc知道要编译成位置无关代码，但是对于函数调用仍然被编译成bl这样的PC-relative指令，在静态链接阶段，static linker会把PLT section加上的。

3、到底函数符号是如何定位的呢？

通过第一节的描述，我们知道，在test程序中，跳转到foo实际上是跳转到foo对应的PLT entry，代码如下：

> 84f4:    ……\
> 84f8:    e28fc600     add    ip, pc, #0    ; 0x0－－－－－－－－－ip寄存器保存了当前PC值\
> 84fc:    e28cca08     add    ip, ip, #32768    ; 0x8000\
> 8500:    e5bcf2d0     ldr    pc, \[ip, #720\]!－－－－－－－－－－－获取got的地址并跳转到该处\
> 8504:    ……

上面的代码不是那么直观，我们可以直接计算一下看看：0x84f8处的指令执行之后，ip等于当前的PC值，也就是0x84f8＋8＝0x8500，加上0x8000之后等于0x10500，再加上720（0x2d0）就是0x107d0了，也就是位于.got section：

> \[21\] .got              PROGBITS        000107bc 0007bc 000024 04  WA  0   0  4

我们再来看看foo的重定位信息，位于.rel.plt，.rel.plt section的每一个entry对应一个PLT entry：

> 000107d0  00000e16 R_ARM_JUMP_SLOT   000084f8   foo

foo符号的PLT entry地址是0x84f8（参考上文），foo符号对应的got entry位于0x107d0，重定位的类型是R_ARM_JUMP_SLOT。到底foo对应的got entry上是什么内容呢？我们来看看：

> Disassembly of section .got:
>
> 000107bc \<_global_offset_table_>:\
> ...\
> 107c8:    000084cc     .word    0x000084cc\
> 107cc:    000084cc     .word    0x000084cc\
> **107d0:    000084cc     .word    0x000084cc**

因此，实际上，通过PLT和GOT的协助，程序最终跳转到0x000084cc执行。而通过section table：

> \[11\] .plt              PROGBITS        000084cc 0004cc 000050 04  AX  0   0  4

我们可以看出，跳转到0x000084cc实际上是跳转到了PLT section的第一个entry。走了一大圈，又回到了原地，不着急，我们继续看代码：

> 000084cc \<.plt>:\
> 84cc:    e52de004     str    lr, \[sp, #-4\]!－－－－－－－－－－－－－－－－将lr压入栈\
> 84d0:    e59fe004     ldr    lr, \[pc, #4\]    ; 84dc \<.plt+0x10>－－－－－－－将0x000082e0赋值给lr\
> 84d4:    e08fe00e     add    lr, pc, lr\
> 84d8:    e5bef008     ldr    pc, \[lr, #8\]!－－－－－－－－－－－－－－－－跳转到GOT\[2\]\
> 84dc:    000082e0     .word    0x000082e0\
> 84e0:    ……

看起来代码没有那么直观，我们还是照旧，直接计算。0x84d4地址的指令执行后lr = 0x000082e0 + 0x84d4 + 8 = 0x107BC，实际上就是.got的首地址，.got section的entry size是4，因此，0x84d8处的指令实际上就是跳转到GOT\[2\]处的指令执行。GOT的前三个entry是特殊的entry，GOT\[0\]是.dynamic segment的地址，dynamic linker需要这个信息进行动态符号定位（例如：找到动态符号表和重定位信息）。GOT\[1\]中保存的是识别本模块的信息。GOT\[2\]中保存了dynamic linker的入口函数地址。

实际上，在静态链接阶段，我们是无法确定dynamic linker的入口函数地址的，这时候，static linker只能是填写全0值，在内核完成将dynamic linker mapping到进程地址空间之后，才能确定该地址，从而写入GOT\[2\]。因此，第一次访问foo符号，实际进入了dynamic linker的代码执行，这时候dynamic linker当然就是解析出foo的符号地址（这时候，libfoo.so已经loading了），并且把foo的最终的运行地址写入foo对应的GOT entry（也就是0x107d0，初始化的时候被设定成.plt的首地址0x000084cc，以便去往dynamic linker）。这样，第二次访问foo的时候就可以直接去到实际的foo函数，而不必让dynamic linker对它进行relocation了。这么做，也就是实现了传说中的lazy binding。和lazy binding相反的是eager binding，也就是说在进入到程序的第一条指令执行前，所有的没有重定位的符号（主程序以及各个动态库）都先由dynamic linker扫描一遍，找到其运行地址并写入GOT，也就是说，当程序开始执行的时候，所有的符号都已经绑定了最后的running address。和eager binding不同，lazy binding那是相当懒惰，只有在程序执行到该函数的时候，才会调用dynamic linker来重定位该符号。我们知道，其实程序中很多代码都是出错处理，很可能整个进程生命期结束了都不会调用到那些代码，既然如此，为何还需要在一开始就对那些符号进行重定位呢？这不是让dynamic linker瞎费功夫嘛，这也是lazy binding存在的意义。

_原创文章，转发请注明出处。蜗窝科技_

\_[http://www.wowotech.net/basic_subject/dynamic-link.html](http://www.wowotech.net/basic_subject/dynamic-link.html)\
\_

标签: [dynamic](http://www.wowotech.net/tag/dynamic) [link](http://www.wowotech.net/tag/link) [动态链接](http://www.wowotech.net/tag/%E5%8A%A8%E6%80%81%E9%93%BE%E6%8E%A5)

[![](http://www.wowotech.net/content/uploadfile/201605/ef3e1463542768.png)](http://www.wowotech.net/support_us.html)

« [计算机科学基础知识之（六）：理解栈帧](http://www.wowotech.net/basic_subject/stack-frame.html) | [Linux时间子系统之（二）：软件架构](http://www.wowotech.net/timer_subsystem/time-subsyste-architecture.html)»

**评论：**

**leon_unique**\
2019-08-08 14:10

想请教下驱动模块在加载至系统中时，模块内部各sections在内存中是如何布局的？例如，A.ko模块内部有4个text段，分别为.text，.unlikely.text，.init.text，.exit.text，这四个代码段的长度均超过0x10000。A.ko在内存中的装载地址为0xffffff8000b58000，当由于直接或间接的原因导致内核panic时，调用栈中的某一地址为0xffffff8000b58ff0，在不结合代码及上下文的情况下如何定位该地址落在哪一个text段中？

[回复](http://www.wowotech.net/basic_subject/dynamic-link.html#comment-7578)

**leon_unique**\
2019-08-10 18:11

@leon_unique：Fixed, 参考kernel/module.c

[回复](http://www.wowotech.net/basic_subject/dynamic-link.html#comment-7584)

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

  - [zRAM内存压缩技术原理与应用](http://www.wowotech.net/memory_management/zram.html)
  - [ARM64的启动过程之（五）：UEFI](http://www.wowotech.net/armv8a_arch/UEFI.html)
  - [DRAM 原理 4 ：DRAM Timing](http://www.wowotech.net/basic_tech/330.html)
  - [页面回收的基本概念](http://www.wowotech.net/memory_management/page_reclaim_basic.html)
  - [X-005-UBOOT-device tree移植(Bubblegum-96平台)](http://www.wowotech.net/x_project/bubblegum_uboot_device_tree.html)

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
