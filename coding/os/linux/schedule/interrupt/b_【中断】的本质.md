
Linux云计算网络 _2021年10月25日 08:13_

以下文章来源于IOT物联网小镇 ，作者道哥

在软件开发中，中断是一个绕不开的重要话题，但是，不知道您是否遇到过这样的困惑：

很多书籍、文章在介绍中断相关的知识点时，说的都挺有道理。

这篇文章对中断的讲解很正确，那篇文章在描述中断的时候也挺对的，但是，这两篇文章中，怎么有些内容是矛盾的啊？！

单独看任何一篇文章感觉都有道理，看的越多，反而越迷糊？

好比在森林里迷路了，如果只有一个指南针，肯定能走出来。

但是，如果你有 `2` 个指南针，所指的方向却是相反的，这个时候应该相信谁呢？！

我们仔细梳理了一下就会发现：每一篇文章都是在一定的语境、一定的上下文环境中来讲解的，不同文章的矛盾之处，恰恰是它们所描述的那个上下文大环境不同。

> 上下文环境，就是描述当前正在执行的程序相关的静态信息，比如：有哪些代码段，栈空间在哪里，进程描述信息在什么位置，当前执行到哪一条指令等等。

如果我们没有一个全局的视角，在同一个上下文环境中来对比不同的文章，就会让自己的理解和认识越来越蒙圈。

因此，对于这种概念比较庞杂，无法用某种确定的逻辑来贯穿的知识点，在脑袋中一定要有一幅全局的地图。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

只有对这个全局的地图掌握了，在具体学习每一个局部的知识点时，才能知道自己所处的位置在哪里，才不至于走偏。

这篇文章，我们继续去繁从简，从 `8086` 这个最简单的处理器入手，来聊一下关于中断的一些知识。

有了这个储备，理清了基本的脉络之后，以后再去学习 `Linux` 系统中的中断相关内容时，才会有原来如此的感觉！

# 中断向量与中断描述符

中断向量这个词很时髦，也很神秘！

按道理，不应该在第一部分就端上中断向量这盘硬菜，应该从中断源开始聊起。

但是，毕竟我们已经学习过那么多关于中断的知识了，脑袋中肯定是对中断已经有了一些的基本认知。

所以，在这里我们还是首先来明确一下中断向量和中断描述符这个问题。

在前面的文章中已经聊过关于实模式和保护模式的问题，在 【Linux 从头学】这个系列中，我们一直以来描述的都是实模式下的事情。

> 本文是实模式下的最后一篇文章，下一篇文章将会进入保护模式。

那么，中断向量就是工作在实模式下的，处理器通过中断号和中断向量，来定位到相应的中断处理程序。

而中断描述符呢，就是工作在保护模式下，处理器通过中断号和中断描述符，来定位到相应的中断处理程序。

也就是说：中断向量和中断描述符，它俩的根本作用是一样的。

只是它们存在于不同的大环境中，而且从描述上也能感觉到，保护模式下的中断描述符会更复杂一些，功能也更强大一些。

它俩就像一对兄弟一样，从外表上看是差不多，功能也是类似。但是透入到内部去看，就会发现有很多的不同之处。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

因此，这篇文章我们讲解的就是在实模式下的中断，这一点请大家先明白。

## 中断的分类

在 `x86` 系统中，中断的分类如下：
!\[\[Pasted image 20240925102246.png\]\]
!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

#### 内部中断

所谓的内部中断，是在 `CPU` 内部产生并进行处理的。比如：

> 1. CPU 遇到一条除以 0 的指令时，将产生 0 号中断，并调用相应的中断处理程序;
>
> 1. CPU 遇到一条不存在的非法指令时，将产生 6 号中断，并调用相应的中断处理程序;

对于内部中断，有时候也称之为异常。

软中断也属于内部中断，是非常有用的，它是由 `int` 指令触发的。比如 `int3` 这条指令，`gdb` 就是利用它来实现对应用程序的调试。

很久之前写过这样的一篇文章[原来gdb的底层调试原理这么简单](https://mp.weixin.qq.com/s?__biz=MzA3MzAwODYyNQ==&mid=2247483895&idx=1&sn=ba35d1823c259a959b72a310e0a92068&scene=21#wechat_redirect)，其中就描述了 `gdb` 是如何通过插入一个 `int` 指令，来替换被调试程序的指令码，从而实现断点调试功能的。
!\[\[Pasted image 20240925102253.png\]\]
!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

#### 外部中断

`x86` `CPU` 上有 `2` 个中断引脚：`INT` 和 `INTR`，分别对应：不可屏蔽中断和可屏蔽中断。

所谓不可屏蔽，就是说：中断不可以被忽视，`CPU` 必须处理这个中断。

如果不处理，程序就没法继续执行。

而对于可屏蔽中断，`CPU` 可以忽略它不执行，因为这类中断不会对系统的执行造成致命的影响。

对于外部的可屏蔽中断，`CPU` 上只有一根 `INTR` 引脚，但是需要产生中断信号的设备那么多，如何对众多的中断信号进行区分呢？

一般都是通过可编程中断控制器(`Programmable Interrupt Controller, PIC`)，在计算机中使用最多的就是 `8259a` 芯片。

虽然现代计算机都已经是 `APIC`(高级可编程中断控制器) 了，但是由于 `8259a` 芯片是那么的经典，大部分描述外部中断的文章都会用它来举例。

每一片 `8259a` 可以提供 `8` 个中断输入引脚，两片芯片级联在一起，就可以提供 `15` 个中断信号：
!\[\[Pasted image 20240925102304.png\]\]
!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

> 1. 主片的输出引脚 INT 连接到 CPU 的 INTR 引脚上;
>
> 1. 从片的输出引脚 INT 连接到主片的引脚 2 上;

这样的话，两片 `8259a` 芯片就可以向 `CPU` 提供 `15` 个中断信号了，比如：鼠标、键盘、串口、硬盘等等外设。

> 1. 8259a 之所以称作可编程，是因为它的内部有相关的寄存器。
>
> 1. 可以通过指定的端口号，对这些寄存器进行设置，让 8 根 IRQ 中断线上的信号，在送到 CPU 时，对应不同的中断号。

另外，对于外部可屏蔽中断，有 `2` 层的屏蔽机制：

> 1. 在 8259 芯片中，有中断屏蔽寄存器，可以对 IRQ0 ~ IRQ7 输入引脚进行屏蔽;
>
> 1. 在 CPU 内部，也有一个标志寄存器，可以对某一类中断信号进行屏蔽;

## 中断号

在 `x86` 处理器中，一共支持 `256` 个中断，每一个中断都分配了一个中断号，从 `0` 到 `255`。

其中，`0 ~ 31` 号中断向量被保留，用来处理异常和非屏蔽中断(其中只有 `2` 号向量用于非屏蔽中断，其余全部是异常)。

当 `BIOS` 或者操作系统提供了异常处理程序之后，当一个异常产生时，就会通过中断向量表找到响应的异常处理程序，查找的过程马上就会介绍到。

从中断号 `32` 开始，全部分配给外部中断。

比如：

> 1. 系统定时器中断 IRQ0，分配的就是 32 号中断;
>
> 1. Linux 的系统调用，分配的就是 128 号中断;

我们来分别看一下内部中断和外部中断相关的中断号：
!\[\[Pasted image 20240925102314.png\]\]
!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

对于通过 `8259a` 可编程中断控制器接入的中断信号分配如下图所示：
!\[\[Pasted image 20240925102321.png\]\]
!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

刚才已经说过，`8259a` 是可编程的，假如我们通过配置寄存器，把 `IRQ0` 的中断号设置为 `32`, 那么主片上 `IRQ1 ~ IRQ7` 所对应的中断号依次加 `1`，从片上 `IRQ8~IRQ15` 对应的中断号也是依次递增。

所以，有时候我们可以在代码中断看到下面的宏定义：
!\[\[Pasted image 20240925102327.png\]\]
!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

## 中断向量和中断处理程序

当一个中断发生的时候，`CPU` 获取到该中断对应的中断号，下一步就是要确定调用哪一个函数来处理这个中断，这个函数就称作中断服务程序(Interrupt Service Routine，ISR)，有时候也称作中断处理程序、中断处理函数，本质都一样。

中断向，就是通过中断号去查找处理程序的重要的桥梁！
!\[\[Pasted image 20240925102336.png\]\]
!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

#### 中断向量的本质

在 `8086` 中，一个中断向量，就是一个 段地址:中断处理函数偏移量 这样的一对数据，通过这个数据，就可以定位到内存中指定位置的那个中断处理函数。

非常类似于高级编程语言中的函数指针，就是用来指向一个函数的开始地址。

`8086` 规定：`256` 个中断向量，必须从内存的 `0` 地址处开始存放。

每一个中断向量占用 `4` 个字节(`2` 个字节的段地址，`2` 个字节的偏移地址)，`256` 个中断一共占用了 `1024` 个字节的空间。

之前的文章中，已经介绍过相关的内存模型，如下图所示：
!\[\[Pasted image 20240925102347.png\]\]
!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

如果把一个中断向量看作函数指针，那么这个中断向量表就相当于是函数指针数组。

举例：

假设 `2` 号中断被触发了，`CPU` 就会到中断向量表中查找 `2` 号中断的中断向量。

因为每一个中断向量占据 `4` 个字节，那么 `2` 号中断向量的开始地址就是 `2 * 4 = 8`，第 `8` 个字节。

然后在第 `8` 个字节开始，取 `4` 个字节的内容：`0x1000:0x2000`。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

意思是：`2` 号中断的处理函数，在段地址为 0x1000，偏移量为 0x2000 的位置处。

那么 `CPU` 就按照 `8086` 的物理地址计算方式，得到中断处理函数的物理地址为 `0x12000` (段地址左移 `4` 位 + 偏移地址)，于是就跳转到该函数地址处去执行。

> 1. 由于 Linux 系统是运行在保护模式，在这个模式下，当发生中断时，是通过中断描述符来查找中断处理函数的。
>
> 1. 每一个中断描述符，描述了一个中断处理函数所在段的选择子和偏移量，本质上也是用来查找一个中断处理函数。

#### 中断处理程序的安装

既然通过中断向量，找到了中断处理程序，那么这些中断处理程序都是谁放在内存中的呢？

如果您看过一些比较底层的计算机书籍，就能看到一般都会举例：如何手动的把一个普通函数设置为一个中断处理函数。

操作步骤是：

> 1. 在代码中，写一个普通函数;
>
> 1. 把这个函数的指令码，搬运到内存中的某一个位置;
>
> 1. 把这个位置(段地址:偏移量)，作为一个中断向量，设置到中断向量表中;

此时，如果发生了该中断，你所提供的函数就作为中断处理函数被执行了。

当然了，在一个计算机系统中，`BIOS`、操作系统和各种外设，会自动为我们提供很多基本的中断处理函数的。

比如：`BIOS` 中就提供了软中断、内部中断、硬件中断等处理函数，这些函数是固化在 `BIOS` 的代码中的(映射到 `BIOS` 所在的 `ROM` 芯片上)，`BIOS` 只需要把这些处理函数的地址，写入到中断向量表中的相应位置即可。

在之前的文章中提到过，内存中的某些位置是映射到外设的 `ROM`，在这些外设的 `ROM` 中也存在一些外设自带的程序。

`BIOS` 在启动时，会扫描这些映射到外设的内存空间，通过某些关键字信息，如果发现外设有自带的程序，就会去执行。

这些外设程序一般是进行一些自身的初始化，并填写相关的中断向量表，使它们指向外设自带的中断处理程序。

对于操作系统来说就更不用说了，它会重新安排自己需要的中断处理函数，这部分内容我们以后再一起学习、讨论！

## 中断现场的保护和恢复

当一个中断发生的时候，肯定有一个正在执行的程序被打断。

当中断处理函数执行结束之后，这个被打断的程序需要从刚才被打断的地方继续执行(暂时先不要考虑从中断返回点，进行多任务切换的事情)。

而一个程序执行的上下文环境，就是处理器中的各种寄存器内容：代码段寄存器 `cs`，指令指针寄存器 `sp`，标志寄存器 `FLAGS`。

但是，在中断处理程序中，也需要使用这些寄存器。

处理器中的这些寄存器，就是每一个程序执行时上下文信息的存储容器，当然也包括终端处理程序！

因此，在进入中断处理程序之前，`CPU` 会自动的把这些寄存器 `push` 到栈中保存起来，然后再跳转到中断处理程序中去执行。

当中断处理程序执行结束后，`CPU` 会从栈中弹出这些内容，恢复到相应的寄存器中，于是被打断的程序就可以继续执行了。

## 总结：中断的本质

从功能的角度看，中断有 `2` 个作用：

> 1. 提供执行异步序列的机制；
>
> 1. 给应用程序提供进入系统层的入口；

关于第 `2` 点，以后在介绍到 `Linux` 中的 `int 0x80` 中断就非常清楚了，也就是通过中断，让应用层的程序有机会进入到系统代码中去执行。

因为应用层与操作系统层的代码，是工作在不同的安全级别。

为了系统的安全，`Linux` 操作系统提供了这样的一个机制，让低安全级别的应用程序，进入到高安全级别的操作系统代码中去执行，毕竟所有的硬件等系统资源都是由操作系统来统一管理的。

我们再从中断处理程序的安装角度来看，中断本质上就是**增加了一层间接性：通过固定位置的中断向量表，让中断处理函数的实际地址可以被动态的放在任意位置。**

为什么这么做？

假如操作系统想为某一个中断提供处理函数，那么这个处理函数的地址放在内存中的什么位置比较合适?

需要考虑 `CPU`, 内存大小和布局等多种因素，非常复杂！

而通过使用中断向量表，就在一个固定位置处存放了很多个“指针”。

当中断处理函数放在内存中某个任意位置之后，让“指针”指向这个函数的地址就可以了，从而达到解耦的目的。

这样的话，无论是发生硬件中断，还是应用层代码通过中断门来调用操作系统提供的函数，只要触发相应的中断就可以了，简化了 `CPU` 的设计。

------ End ------

关于中断的相关内容，还有很多需要学习，任重而道远！

特别是在 `Linux`　系统中，中断处理又分为上半部分、下半部分，而下半部分又可以根据不同的功能需求采取不同的机制来处理。

我仍然是持有之前的观点：磨刀不误砍柴工。

把学习周期拉长，一点一滴的积累，Haste Makes Waste！

**”了吗？**

______________________________________________________________________

后台回复“加群”，带你进入高手如云交流群

**推荐阅读：**

[我去，又又又被内存坑了！](http://mp.weixin.qq.com/s?__biz=MzI1OTY2MzMxOQ==&mid=2247496569&idx=1&sn=1825966fa828bbcd6564b901d6eae8f0&chksm=ea77c7c1dd004ed78483167a0770a242982f377e43390a6087dccb8cfe98986fcfb5fef794f6&scene=21#wechat_redirect)

[图解 | Linux内存回收之LRU算法](http://mp.weixin.qq.com/s?__biz=MzI1OTY2MzMxOQ==&mid=2247496417&idx=1&sn=4267d317bb0aa5d871911f255a8bf4ad&chksm=ea77c659dd004f4f54a673830560f31851dfc819a2a62f248c7e391973bd14ab653eaf2a63b8&scene=21#wechat_redirect)

[Linux 应用内存调试神器- ASan](http://mp.weixin.qq.com/s?__biz=MzI1OTY2MzMxOQ==&mid=2247496414&idx=1&sn=897d3d39e208652dcb969b5aca221ca1&chksm=ea77c666dd004f70ebee7b9b9d6e6ebd351aa60e3084149bfefa59bca570320ebcc7cadc6358&scene=21#wechat_redirect)

[深入理解 Cilium 的 eBPF 收发包路径](http://mp.weixin.qq.com/s?__biz=MzI1OTY2MzMxOQ==&mid=2247496345&idx=1&sn=22815aeadccc1c4a3f48a89e5426b3f3&chksm=ea77c621dd004f37ff3a9e93a64e145f55e621c02a917ba0901e8688757cc8030b4afce2ef63&scene=21#wechat_redirect)

[Page Cache和Buffer Cache关系](http://mp.weixin.qq.com/s?__biz=MzI1OTY2MzMxOQ==&mid=2247495951&idx=1&sn=8bc76e05a63b8c9c9f05c3ebe3f99b7a&chksm=ea77c5b7dd004ca18c71a163588ccacd33231a58157957abc17f1eca17e5dcb35147b273bc52&scene=21#wechat_redirect)

[深入理解DPDK程序设计|Linux网络2.0](http://mp.weixin.qq.com/s?__biz=MzI1OTY2MzMxOQ==&mid=2247495791&idx=1&sn=5d9f3bdc29e8ae72043ee63bc16ed280&chksm=ea77c4d7dd004dc1eb0cee7cba6020d33282ead83a5c7f76a82cb483e5243cd082051e355d8a&scene=21#wechat_redirect)

[一文读懂基于Kubernetes打造的边缘计算](http://mp.weixin.qq.com/s?__biz=MzI1OTY2MzMxOQ==&mid=2247495291&idx=1&sn=0aebc6ee54af03829e15ac659db923ae&chksm=ea77dac3dd0053d5cd4216e0dc91285ff37607c792d180b946bc09783d1a2032b0dffbcb03f0&scene=21#wechat_redirect)

[网络方案 Cilium 入门教程](http://mp.weixin.qq.com/s?__biz=MzI1OTY2MzMxOQ==&mid=2247495170&idx=1&sn=54d6c659853f296fd6e6e20d44b06d9b&chksm=ea77dabadd0053ac7f72c4e742942f1f59d29000e22f9e31d7146bcf1d7d318b68a0ae0ef91e&scene=21#wechat_redirect)

[Docker  容器技术使用指南](http://mp.weixin.qq.com/s?__biz=MzI1OTY2MzMxOQ==&mid=2247494756&idx=1&sn=f7384fc8979e696d587596911dc1f06b&chksm=ea77d8dcdd0051ca7dacde28306c535508b8d97f2b21ee9a8a84e2a114325e4274e32eccc924&scene=21#wechat_redirect)

[云原生/云计算发展白皮书（附下载）](http://mp.weixin.qq.com/s?__biz=MzI1OTY2MzMxOQ==&mid=2247494647&idx=1&sn=136f21a903b0771c1548802f4737e5f8&chksm=ea77df4fdd00565996a468dac0afa936589a4cef07b71146b7d9ae563d11882859cc4c24c347&scene=21#wechat_redirect)

[Linux 常用监控指标总结](http://mp.weixin.qq.com/s?__biz=MzI1OTY2MzMxOQ==&mid=2247493544&idx=1&sn=68d86833cb3934abca18c95da8b1bae6&chksm=ea77d310dd005a06ad7de14d4d9f29d88e1cadebda7ccd975b3a265e5806608ace14ba12c8b4&scene=21#wechat_redirect)

[Kubernetes 集群网络从懵圈到熟悉](http://mp.weixin.qq.com/s?__biz=MzI1OTY2MzMxOQ==&mid=2247493426&idx=1&sn=e3492cf43c4268c5948d170f4a5d2441&chksm=ea77d38add005a9cdbb5775f2bfd4a2b953e950fcb65c25e91eaea45c68bf5684e3ebc8289e0&scene=21#wechat_redirect)

[使用 GDB+Qemu 调试 Linux 内核](http://mp.weixin.qq.com/s?__biz=MzI1OTY2MzMxOQ==&mid=2247493336&idx=1&sn=268fae00f4f88fe27b24796644186e9e&chksm=ea77d260dd005b76c10f75dafc38428b8357150f3fb63bc49a080fb39130d6590ddea61a98b5&scene=21#wechat_redirect)

[防火墙双机热备](http://mp.weixin.qq.com/s?__biz=MzI1OTY2MzMxOQ==&mid=2247493173&idx=1&sn=53975601d927d4a89fe90d741121605b&chksm=ea77d28ddd005b9bdd83dac0f86beab658da494c4078af37d56262933c866fcb0b752afcc4b9&scene=21#wechat_redirect)

[常见的几种网络故障案例分析与解决](http://mp.weixin.qq.com/s?__biz=MzI1OTY2MzMxOQ==&mid=2247493157&idx=1&sn=de0c263f74cb3617629e84062b6e9f45&chksm=ea77d29ddd005b8b6d2264399360cfbbec8739d8f60d3fe6980bc9f79c88cc4656072729ec19&scene=21#wechat_redirect)

[Kubernetes容器之间的通信浅谈](http://mp.weixin.qq.com/s?__biz=MzI1OTY2MzMxOQ==&mid=2247493145&idx=1&sn=c69bd59a40281c2d7e669a639e1a50cd&chksm=ea77d2a1dd005bb78bf499ea58d3b6b138647fc995c71dcfc5acaee00cd80209f43db878fdcd&scene=21#wechat_redirect)

[kube-proxy 如何与 iptables 配合使用](http://mp.weixin.qq.com/s?__biz=MzI1OTY2MzMxOQ==&mid=2247492982&idx=1&sn=2b842536b8cdff23e44e86117e3d940f&chksm=ea77d1cedd0058d82f31248808a4830cbe01077c018a952e3a9034c96badf9140387b6f011d6&scene=21#wechat_redirect)

[完美排查入侵](http://mp.weixin.qq.com/s?__biz=MzI1OTY2MzMxOQ==&mid=2247492931&idx=1&sn=523a985a5200430a7d4c71333efeb1d4&chksm=ea77d1fbdd0058ed83726455c2f16c9a9284530da4ea612a45d1ca1af96cb4e421290171030a&scene=21#wechat_redirect)

[QUIC也不是万能的](http://mp.weixin.qq.com/s?__biz=MzI1OTY2MzMxOQ==&mid=2247491959&idx=1&sn=61058136134e7da6a1ad1b9067eebb95&chksm=ea77d5cfdd005cd9261e3dc0f9689291895f0c9764aa1aa740608c0916405a5b94302f659025&scene=21#wechat_redirect)

[为什么要选择智能网卡？](http://mp.weixin.qq.com/s?__biz=MzI1OTY2MzMxOQ==&mid=2247491828&idx=1&sn=d81f41f6e09fac78ddddc287feabe502&chksm=ea77d44cdd005d5a48dc97e13f644ea24d6e9533625ce8f204d4986b6ba07f328c5574820703&scene=21#wechat_redirect)

[网络排错大讲解~](http://mp.weixin.qq.com/s?__biz=MzI1OTY2MzMxOQ==&mid=2247488574&idx=1&sn=68df1982e9f23ce42457d00ce529b012&chksm=ea742086dd03a9902adb16d2b7648149209fed6c9811c6dd05be5101b42d462cb48e269b6e9d&scene=21#wechat_redirect)

[OVS 和 OVS-DPDK 对比](http://mp.weixin.qq.com/s?__biz=MzI1OTY2MzMxOQ==&mid=2247488294&idx=2&sn=303d9baefa768f887ba36213df793025&chksm=ea74279edd03ae88bc91300a1666066881dd763879b714941120fb9e41e787c557c2930ff5db&scene=21#wechat_redirect)

[微软出品的最新K8S学习指南3.0](http://mp.weixin.qq.com/s?__biz=MzI1OTY2MzMxOQ==&mid=2247488271&idx=1&sn=a52af9aba2ea8bbc5fd4440f85ece458&chksm=ea7427b7dd03aea1dbf106326168333c651b403dd44abf4e652d5a1912377d64becabac5051a&scene=21#wechat_redirect)下载

▼

_\*\*_****喜欢，就给我一个****“在看”\*\*\*\*_\*\*_

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**10T 技术资源大放送！包括但不限于：云计算、虚拟化、微服务、大数据、网络、**Linux、**Docker、Kubernetes、Python、Go、C/C++、Shell、PPT 等。在公众号内回复「****1024****」**，即可免费获取！！****

阅读 2308

​
