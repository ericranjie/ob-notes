Original 彭伟林 人人极客社区
 _2022年09月23日 08:18_ _江苏_
**人人极客社区**
工程师们自己的Linux底层技术社区，分享体系架构、内核、网络、安全和驱动。

> 作者简介：
> 
> 伟林，中年码农，从事过电信、手机、安全、芯片等行业，目前依旧从事Linux方向开发工作，个人爱好Linux相关知识分享。

## 原理概述

为什么要研究链接和加载？写一个小的main函数用户态程序，或者是一个小的内核态驱动ko，都非常简单。但是这一切都是在gcc和linux内核的封装之上，你只是实现了别人提供的一个接口，至于程序怎样启动、怎样运行、怎样实现这些机制你都一无所知。接着你会对程序出现的一些异常情况束手无策，对内核代码中的一些用法不能理解，对makefile中的一些实现不知所云。所以这就是我们要研究链接和加载的目的：明白程序的映像文件是怎么组织的，程序启动是怎么实现的，相关的机制是怎么联系在一起的。“你应当了解真相，真相会使你自由”。

**链接和加载**(linker and loader)： linker即链接器，它负责将多个.c编译生成的.o文件，链接成一个可执行文件或者是库文件；loader即加载器，它原本的功能很单一只是将可执行文件的段拷贝到编译确定的内存地址即可，但是有了动态链接库以后，部分的外部库引用符号在加载的时候才会得到解析，所以加载也要处理链接器的相同操作重定位。

这方面的资料乍一看起来非常晦涩难懂，其实根本的功能非常简单：链接和加载的最核心的内容就是重定位。链接器负责将多个.o文件链接重定位成一个大文件，而加载器再将这个大文件重定位到一个进程空间当中去。

在linux环境下，链接和加载的机制最终有一个载体来承担，这个载体就是elf文件。所以从研究elf文件格式入手，是理解链接和加载原理的好方法。

本文档描述的链接和加载主要针对用户程序而言，在操作系统的链接和加载和这里有些不同，因为如果你编译一个内核，在加载内核的时候又有谁来做动态加载呢？关于内核实现的不同以后再在专门文档中描述。

## 重定位原理

前面已经说过链接和加载的核心内容就是重定位，所以开篇先用通俗易懂的语言来阐明重定位的原理。

### 符号表(Symbol Table)：

符号表就是一张字符符号和地址的对应表，例如使用“nm file“、”readelf -s file “等命令可以读出一个elf文件的符号表。符号表的作用就是一个助记符，用一个字符串来标示某些抽象的地址，它能标示的地址有代码地址和数据地址，代码地址包括函数名、跳转标号，数据地址包括全局变量。

符号表的组织如下图所示：
![[Pasted image 20241005110917.png]]
![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

从以上描述中可以看出，符号表的作用就是将符号名称和地址进行绑定。而绑定的根本目的就是方便对符号的引用，在符号值发生改变的时候，不需要去手工改动源代码中对符号引用的地方，而这种改动是由链接程序在重新生成执行文件时自动完成的。

### 重定位表(Relocation)：

有了符号表，就需要有人对符号表进行引用，在程序的执行过程中对全局变量的引用、跳转、调用函数，这些都涉及到相应的符号引用。符号和其引用是一对多的关系，一个符号可能被代码中多处引用。因为符号值改变的时候，也需要对所有引用符号的地方的代码进行修改，所以需要还有一张表来记录符号表的引用关系，这就是重定位表：
![[Pasted image 20241005110953.png]]
![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

从上图可见，重定位表项用来记录链接和加载的过程中需要重新定位的位置，在各个段位置发生改变而引起符号地址改变时，根据重定位表来修改符号引用的值。

### GOT表(Global Offset Table)：

前面的符号表和重定位表已经满足编译和链接过程中的重定位需求。同样加载的过程中还需要重定位操作，需要将外部链接库中的函数和变量和本程序中的引用链接起来，但是由于加载过程中代码已经处于运行状态，使用链接过程中同样的重定位手段有些不合适。链接的重定位是通过重定位表直接修改代码来完成的，但是代码在运行过程中再去修改代码会带来很多问题和风险。

所以加载过程中的重定位，使用了一种改良的重定位手段：即通过两张间接访问表来屏蔽掉重定位带来的对代码的修改，访问外部数据使用GOT，访问外部程序使用PLT。这样可链接出位置无关代码PIC（Position Independent Code），需要重定位时只需要修改GOT和PLT的值，而不需要去改动可执行代码。
![[Pasted image 20241005111004.png]]
![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

GOT表用来做数据重定位的原理如上图所示。

### PLT表(Procedure Linkage Table)：

从上一节可知，加载过程中的重定位为了避免对代码的修改，引入了GOT来屏蔽对数据的访问，同理对外部代码的访问也是可以用GOT来访问的。但是为了实现动态链接的特性，即使用的时候才链接，不使用时可以不用链接，对外部代码的访问引入了一个新的表项PLT。
![[Pasted image 20241005111015.png]]
![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

## elf文件

### 相关背景

Elf文件格式，是现有linux环境下最流行的可执行文件格式，在elf文件存储的信息之上，实现了相应的链接和加载特性。 Linux环境下可执行文件格式的发展历史是：a.out -> coff -> xcoff -> elf。 Windows环境下可执行文件格式的发展历史是：dos com/exe -> pe-coff。

### elf文件格式

Linux环境下，三种类型的执行文件都可以使用elf格式来表示：可重定位文件（即编译生成但是未连接的文件）、动态库文件、可执行文件。

Elf文件提供了两种文件解析的视角，链接视角和动态加载视角。链接视角使用section的概念来解析文件，主要关注链接过程的使用；动态加载视角使用segment的概念来解析文件，主要关注加载和动态链接的实现。
![[Pasted image 20241005111029.png]]
![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

整个文件的组织框图如上所示，ELF头描述了section header table和program header table的起始位置、表项大小和个数。根据section header table来寻址相应的section，根据program header table来寻址相应的segment，可以看到一般是一个segment包含多个section。

Elf文件的原理已经在上一章中阐述，elf的具体文件格式详细描述可以参考参考资料中的“Executable and Linking Format (ELF) Specification “。这里不再详细描述，只是记一些Specification上没有的概要和重点理解。

1. 加载视角的“PT_LOAD “类型segment：
    

- 表明可加载到内存中的段，一般程序都包含两个此种类型的段.data、.text
    

2. 加载视角的“PT_INTERP“类型segment：
    

- 指定动态加载程序，即我们用 “ldd“命令看到的动态加载器
    
![[Pasted image 20241005111039.png]]
![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

3. 加载视角的“PT_DYNAMIC “类型segment：
    

- 相当于动态加载的一个入口段，指定了动态加载和链接需要的各种数据段的地址和类型。DT_NEEDED、DT_SONAME、DT_RPATH表项承载的是编译时指定的一些依赖库和搜索路径等等。
    
![[Pasted image 20241005111048.png]]
![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

### 相关工具

Linux下可以操作elf文件的有以下工具：
```cpp
a.readelf   “readelf –a file“读出elf文件的所有信息。      b.nm   “nm file“读出elf文件的符号表信息。      c.objdump   “objdump –d file“反汇编出elf文件中包含可执行代码的section，elf命令中功能最强大的一个。      d.objcopy   转换elf文件为bin或者其他格式的文件，编译内核的时候会使用到。      e.strip   去掉elf文件中符号表和调试信息，对elf文件进行减肥。      f.addr2line   将绝对地址，转换成调试信息中的源文件行号。   
```
---

ELF4

ELF · 目录

下一篇含大量图文解析及例程 | Linux下的ELF文件、链接、加载与库（下）

Reads 3186

​

Comment

**留言 3**

- Peter
    
    江苏2022年9月23日
    
    Like1
    
    大家可以加我微信进文章作者群：rrjike，一起学习，共同进步，还可以进群认识更多朋友![[玫瑰]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![[玫瑰]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![[玫瑰]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
- 今朝
    
    上海2022年9月23日
    
    Like
    
    倒数第二张手绘图是从 linker and loader 截取的 ![[Grin]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
    人人极客社区
    
    Author2022年9月23日
    
    Like
    
    明明是倒数第三张![[右哼哼]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    

已无更多数据

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/9sNwsXcN68pL55XIyzTrCHZTbIUdTibQcuzuCaYeGTXNMyn6ACmicUrpoDC0xZSap46XJ59sKysPg9Rg379f32cA/300?wx_fmt=png&wxfrom=18)

人人极客社区

171111

3

Comment

**留言 3**

- Peter
    
    江苏2022年9月23日
    
    Like1
    
    大家可以加我微信进文章作者群：rrjike，一起学习，共同进步，还可以进群认识更多朋友![[玫瑰]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![[玫瑰]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![[玫瑰]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
- 今朝
    
    上海2022年9月23日
    
    Like
    
    倒数第二张手绘图是从 linker and loader 截取的 ![[Grin]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
    人人极客社区
    
    Author2022年9月23日
    
    Like
    
    明明是倒数第三张![[右哼哼]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    

已无更多数据