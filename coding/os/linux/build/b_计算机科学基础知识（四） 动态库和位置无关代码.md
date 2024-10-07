作者：[linuxer](http://www.wowotech.net/author/3 "linuxer") 发布于：2015-3-6 9:39 分类：[基础学科](http://www.wowotech.net/sort/basic_subject)
# 一、前言

本文主要描述了动态库以及和动态库有紧密联系的位置无关代码的相关资讯。首先介绍了动态库和位置无关代码的源由，了解这些背景知识有助于理解和学习动态库。随后，我们通过加-fPIC和不加这个编译选项分别编译出两个relocatable object file，看看编译器是如何生成位置无关代码的。最后，我们自己动手编写一个简单的动态库，并解析了一些symbol Visibility、动态符号表等一些相关基本概念。

本文中的描述是基于ARM MCU，GNU/linux平台而言的，本文是个人对动态库的理解，如果有错误，请及时指出。
# 二、背景介绍

位置无关代码实际上是和动态库概念紧密联系在一起的，本章首先描述为何会提出动态库的概念，然后解释动态库为何需要编译成PIC的代码。
## 1、为何会提出动态库的概念？

引入静态库后，解决了一些问题，但是仍然存在下面的弊端：

（1）任何对静态库的升级都需要rebuild（或者叫做relink）的过程
（2）通用的函数（例如标准IO函数scanf和printf）存在于各个静态链接的程序中，导致编译后的静态可执行程序的size比较大，在各个可执行程序中，这些通用的函数代码是重复的，占用了磁盘和内存资源

正因为如此，动态库和动态链接的概念被提出来来解决这些问题。动态库也是一种ELF格式的对象文件，在运行的时候，它可以被加载到任何的地址执行。
## 2、动态库为何需要编译成PIC的代码？

无论是动态库还是静态库，其本质都是代码共享。对于静态库，其代码以及数据都是在各个静态链接的可执行文件中有一份copy，所有符号的地址已经确定，因此在loading的时候，OS会比较轻松。不过这种代码共享无法在run time的时候共享代码，从而导致了资源的浪费。当然，它的好处就是简单、速度快（无需dynamic linker来重定位符号）。对于静态编译，static linker将多个编译单元（.o文件和库文件）整合成一个模块，因此，进入run time，实际上只有一个执行模块。对于动态链接，在run time的时候，除了可执行文件这个模块，该可执行文件所依赖的各个动态库也是一个个的运行模块，这时候，可执行文件调用动态库的符号实际上是就是需要引用其他运行模块的符号了。对于可执行文件而言，loader将其加载到哪个地址并不关键，反正每个进程都有自己独一无二的地址空间，可执行文件可以mapping到各自virtual memory space的相同地址也无妨，不过对于动态库模块而言，就有些麻烦了。如果我们不将动态库编译成PIC的也就是意味着loader一定要把动态库加载到某个特定的地址（该地址编译的时候就确定了）上它才可以正确的执行。假设我们有A B C D四个动态库，假设程序P1依赖A B两个动态库，P2依赖C D两个动态库，那么A B和C D的动态库的加载地址有重叠也没有关系，P1和P2可以同时运行。但是如果有一个新的程序P3依赖A B C D四个动态库，那么前面为动态库分配的加载地址就不能正常工作了。当然，重新为这四个动态库分配load address（让地址不重叠）也是ok的，但是这样一来，P1虽然没有使用C D这两个动态库，但是P1的地址空间还是要保留C D动态库的那段地址，对于地址这样宝贵资源，这么浪费简直是暴殄天物。更重要的是：这样的机制实际上对进程虚拟地址的管理就变得非常复杂了，假设A B C D是分配了一段连续的地址，如果C动态库更新了，size变大了，原本分配的地址空间不够了，怎么办？我们必须再寻找一个新的地址段来加载C动态库。如果系统只有四个动态库起始还是OK的，如果动态库非常非常多……怎么办？更糟的是：不同的系统使用不同的动态库，管理起来更令人头痛

最好的方法就是将动态库编译成PIC（Position Independent Code），也就是说动态库可以被加载到任何地址并正确运行。
# 三、动手实践：观察PIC的.o文件的反汇编结果
## 1、源代码foo.c
```cpp
include <stdio.h>  
int xxx = 0x1234;  
int yyy;  
int foo(void)  
{  
yyy = 0x5678;  
printf("xxx=%x yyy=%x\n", xxx, yyy);  
return xxx;  
}
```
## 2、观察foo.o文件中的符号定位信息

使用arm-linux-gcc –c foo.c将source code编译成relocatable file。我们来看看正文段中的relocation信息：
```cpp
00000030  00000e1c R_ARM_CALL        00000000   printf  
00000044  00000f02 R_ARM_ABS32       00000004   yyy  
0000004c  00000c02 R_ARM_ABS32       00000000   xxx
```
R_ARM_ABS32是一种ARM平台上的absolute 32-bit relocation，在32 bit的ARM平台上，这种重定位的方式是没有任何约束的，可以将地址重定位到4G地址空间的任何位置。具体实现方式需要参考反编译的汇编代码，我们来看看汇编代码是如何访问yyy这个数据的：
```cpp
……  
8:   e59f2034        ldr     r2, pc, #52   ; 44 <.text+0x44>  
c:   e59f3034        ldr     r3, pc, #52   ; 48 <.text+0x48>  
10:   e5823000        str     r3, r2  
……  
44:   00000000        .word   0x00000000  
48:   00005678        .word   0x00005678
```
具体做法非常的简单，在这段代码的后面（也是.text section的一部分）给出一个32-bit的跳板memory（上面黑色加粗的那一行），位于<.text+0x44>，这个memory用于保存yyy符号的运行地址。由于同在一个正文段，因此它们之间的offset是确定的，使用“ldr     r2, [pc, #52] ”这样的PC-relative的访问指令可以访问到yyy变量的地址，通过“str     r3, [r2]”可以将yyy变量的内容保存到r3中。

下面我们我们再看看函数符号的访问。R_ARM_CALL这种类型的重定位信息主要用于函数调用的（对应的ARM指令就是BL和BLX），实现也很简单，如下：
```cpp
……  
30:   ebfffffe        bl      0
……
```
BL指令是一个PC-relative指令，会将控制权交给相对于当前PC值的一个地址上去（同时设定lr寄存器），bl这条指令的0～23个bit（用imm24表示））用来表示相对与PC的偏移地址，最终跳转到的地址是PC＋（imm24在低位添加00b，然后做符号扩展），也就是正负32M的区域（注意：BL不能任意跳转4G范围的地址空间）。之所以添加两个0是因为offset地址总是4字节对齐的。

对于静态链接，很简单，虽然那些重定位信息在正文段，但是没有关系，在程序loading之前，static linker可以修改正文段的内容。
## 3、编译PIC的.o文件并观察

编译成位置无关代码也就意味着这段代码多半是动态库的一部分，需要动态加载到一个编译时候未知的地址上。也就是说上文中使用的方法已经不行了，编译时候符号的地址还是不确定的，因此static linker无法将地址填入到.text section中。在loading的时候，虽然知道了符号runtime address，但是正文段是read only的，也无法修改。怎么办呢？我们来一起看看程序如何实现。

使用arm-linux-gcc -fPIC–c foo.c将source code编译成relocatable file。我们来看看正文段中的relocation信息：
```cpp
Relocation section '.rel.text' at offset 0x4e0 contains 5 entries:  
Offset     Info    Type            Sym.Value  Sym. Name  
00000048  00000f1b R_ARM_PLT32       00000000   printf  
00000064  00001019 R_ARM_BASE_PREL   00000000   GLOBAL_OFFSET_TABLE  
00000068  0000111a R_ARM_GOT_BREL    00000004   yyy  
00000070  00000d1a R_ARM_GOT_BREL    00000000   xxx
```
我们首先看看_GLOBAL_OFFSET_TABLE_这个符号，看起来和传说中的GOT（Global Offset Table）有关。那么什么是GOT呢？它有什么作用呢？我们先回到c代码，思考一下对xxx符号的访问。这时候，我们能确定xxx的runtime address吗？当然不能，离loading还远着呢，这时候我们能确定访问xxx的代码（.text section中）和xxx符号（.data section）之间offset吗？也不能，因为还有多个.o文件最后被link成一个动态库。怎么办？我们必须借助一个桥梁来让数据访问变得Position Independent，这个桥梁就是GOT（Global Offset Table）。当然GOT必须是可读可写的，因为后续在run time的时候还要修改其内容。_GLOBAL_OFFSET_TABLE_就是定义了GOT在memory中的位置。因此64那个位置的重定位信息和GOT相关，R_ARM_BASE_PREL这个relocation type则说明这个重定位信息说明该位置保存了GOT offset。由于目前还是.o文件，还没有确定最后GOT信息，因此需要这个relocation的信息，一旦完成动态库的编译，这个relocation entry就不需要了。

R_ARM_GOT_BREL这个type说明这个重定位信息是一个描述GOT entry和GOT起始位置的offset。例如：yyy这个符号还需要relocation，那么它的relocation位于正文段offset是0x68的位置，其内容保存了yyy符号在GOT entry中的地址和GOT起始位置的偏移。OK，有了这些铺垫，可以看看程序对yyy这个数据是如何访问的：
```cpp
……  
c:   e59f4050        ldr     r4, pc, #80   ; 64 <.text+0x64>  
10:   e08f4004        add     r4, pc, r4 －－－－－－－－－－－－－－－获得GOT的起始位置的地址  
14:   e59f304c        ldr     r3, pc, #76   ; 68 <.text+0x68> －－－－－获得yyy符号在GOT中的offset  
18:   e7942003        ldr     r2, r4, r3 －－－－－－－－－－－－－－获得yyy符号的runtime address  
1c:   e59f3048        ldr     r3, pc, #72   ; 6c <.text+0x6c>  
20:   e5823000        str     r3, r2 －－－－－－－－－－－－－－－设定yyy符号的内容  
……  
64:   0000004c        .word   0x0000004c－－－－－GOT offset  
68:   00000000        .word   0x00000000－－－－－yyy的地址在GOT中的偏移  
6c:   00005678        .word   0x00005678
```
由此可见，PIC的代码对全局数据的访问都是通过GOT来完成的，从而做到了位置无关。
# 四、动手实践：观察动态库的反汇编结果

## 1、如何生成动态库？

我们准备动手做一个动态库了，先看source code，一如既往的简单（注意：我们不建议导出动态库中的数据符号，这里主要是为了描述动态库的概念而这么做的）：
```cpp
int xxx = 0x1234;

int yyy;  
int foo(void)  
{  
yyy = 0x5678;  
return xxx;  
}
```
通过下面的命令可以编译出一个libfoo的动态库：
```cpp
arm-linux-gcc -shared -fPIC -o libfoo.so foo.c
```
-shared告知gcc生成share object文件，而-fPIC则告诉gcc请生成位置无关代码。
## 2、观察符号表的变化

我们在[relocatable object](http://www.wowotech.net/basic_subject/compile-link-load.html)中已经对符号表进行了描述：对静态编译的程序而言，.o文件中的符号表一是要对外宣称自己定义了哪些符号，另外一个是向外宣布自己引用了哪些符号，需要其他模块来支持。有了这些信息，static linker才能整合各个relocatable object file中的资源，互通有无，最后融合成一个静态的可执行程序。因此，实际上，对于静态的可执行程序，在加载执行的时候，其符号表已经没有任何意义了（不过可以方便debug），对于CPU而言，其执行就是要知道地址就OK了（静态编译程序所有的符号都已经定位了），符号什么的它不关心，因此，实际上符号表可以删除。如果你愿意，你可以通过strip命令来进行实验，看看tripped和not stripped的elf文件有什么不同。

然而，计算机科学的发展是不断前进的，当有了动态库之后，符号表会怎样呢？我们自己可以动手生成一个动态链接的可执行程序或者动态库并观察其中的符号表信息（恰好上一节已经生成一个libfoo.so，就它吧）。通过readelf工具，我们可以看到，动态链接的程序中有两个符号表，一个是大家之前就熟悉的.symtab section（我们称之符号表），另外一个就是.dynsym section（动态符号表）。这两个符号表都有自己对应的string table，分别是.strtab和.dynstr section。

.symtab section我们前面的文章都有描述，为何又增加了一个.dynsym section呢？我们先假设我们编译出来的动态库只有一个符号表，那么当使用strip命令删除符号表以及对应的字符串表之后会怎样？当其他程序调用该动态库提供的接口API函数的时候，dynamic linker还能找到对应的API函数符号吗？当然不行，符号表都删除了还想怎样。静态链接的程序之所以可以strip掉符号表以及对应的字符串表那是因为程序中所有符号都已经尘埃落定（所有符号已经重定位），因此strip后也毫无压力，但是动态链接的情况下，程序中的没有定位的符号以及动态库中宣称的符号都需要有一个特别的符号表（是正常符号表的子集）来保存动态链接符号的信息，这个表就是动态连接符号表（.dynsym section）。

OK，最后总结一下：符号表（.symtab section）是指导static linker工作的，运行的时候可以不需要。动态符号表（.dynsym section）是给dynamic linker用的，程序（或者动态库）运行的时候，dynamic linker用动态符号表的信息来定位符号。
## 3、Binding Property和Symbol Visibility

我们在讲述[relocatable object file](http://www.wowotech.net/basic_subject/compile-link-load.html)的时候已经给出了binding属性（binding property）的解释。一个符号可能有global、local和weak三种binding property。这个binding property主要是被static linker用来进行.o之间的符号解析（symbol resolution）的。Bind属性之外还有一个属性我们一直没有描述（通过readelf观察符号表的时候，该属性对应列的名字是Vis的那个），我们称之Symbol Visibility或者符号的可见性。之所以前面的文章中没有描述主要是因为Symbol visibility是和动态库以及动态链接相关的。

当引入动态连接和动态库的概念之后，代码和数据的共享会变得复杂一些。和binding property不一样，Symbol Visibility是针对运行模块（动态链接的可执行程序或者动态库）之间的相互引用。例如我们有A.o B.o C.o三个编译模块，static linker将这三个.o文件link成一个libABC.so文件。A.o模块要调用B.o中的一个函数bb，那么bb函数就一定需要是一个GLOBAL类型的，但是bb函数并不是动态库libABC.so的接口API（或者称之export symbol），也就是说，为了更好的封装性，我们希望bb这个函数对外不可见，dynamic linker看不到这个符号，bb不参与动态符号解析。如果动态库导出所有的符号，那么，在动态链接的时候，符号冲突的可能性就非常的大，特别是对于那些大型项目，可能该项目涉及的每个动态库都是由不同team负责的。除了模块的封装性之外，Symbol Visibility也是和程序的性能有关。如果导出太多的符号，除了占用更多的内存，还意味着增加loading time和dynamic linking time。

看，不控制Symbol Visibility的危害还是很大D，这时候阅读本文的你估计一定会问：那么控制Symbol Visibility哪家强呢？我推荐使用大杀器static关键字，简单，实用，人人会。给function或者全局变量加上static关键字，别说是对dynamic linker（运行模块之间的引用）进行了限制，就是static linker（.o 文件之间的引用）也是拿他毫无办法。当然，缺点也很明显：不能在动态库的多个.o之间共享。在这种场景下，我们需要求助其他方法了，对于gcc，我们可以用下面的方法：

> 符号类型 符号名字 __attribute__ ((visibility ("xxx")))；

其中xxx指明了该符号的Symbol Visibility属性，Symbol Visibility属性可以设定为：

（1）DEFAULT（虽然命名是default，但是有些public的味道）。该属性的符号被导出，该符号可以被其他运行模块访问
（2）PROTECTED。同DEFAULT，不过该符号不能被overridden。也就是说，如果一个动态库中的符号是PROTECTED，那么动态库中的代码访问该符号是享有优先权的，即便其他的运行模块定义了同名的符号。
（3）HIDDEN。HIDDEN的符号不会被导出，不参与动态链接。
（4）INTERNAL。其他运行模块不能访问该类型的符号。

回到上一节描述的这个source code，其中有三个符号：xxx、yyy和foo，都是被导出的，可以被其他的模块调用。如果你有兴趣，可以自己试着控制符号的visibility，看看效果如何。

## 4、动态库文件的加载

libfoo这个shared object elf文件的加载是根据Program header进行的。在ELF file header中可以看到该动态库共计4个program header，如下：
```cpp
Program Headers:  
Type           Offset   VirtAddr   PhysAddr   FileSiz MemSiz  Flg Align  
LOAD           0x000000 0x00000000 0x00000000 0x005c0 0x005c0 R E 0x8000  
LOAD           0x0005c0 0x000085c0 0x000085c0 0x00118 0x00120 RW  0x8000  
DYNAMIC        0x0005cc 0x000085cc 0x000085cc 0x000e0 0x000e0 RW  0x4  
GNU_STACK      0x000000 0x00000000 0x00000000 0x00000 0x00000 RW  0x4
```
带有LOAD标记的那些program header entry会被mapping到进程地址空间上去。第一项是code segment，由于动态库的代码是PIC的，因此其VirtAddr和PhysAddr都是0，表示可以运行在任意地址上。第二项是data segment，在实际中，动态库的code和data segment都是连续加载的，因此，如果code segment的run time地址是0的话，那么data segment的地址应该是0x5c0，不过由于code segment是0x8000对齐的，因此data segment的地址被设定为0x85c0。当然，如果实际该动态库被加载到了进程的X虚拟地址上的话，data segment的runtime地址应该是X + 0x85c0。对于动态库而言，其code segment可以被多个进程共享，也就是说，虽然code segment被加载到不同的进程的不同的虚拟地址空间，但是其物理地址是一样的，只不过各个进程设定自己的page table就OK了。对于data segment，各个进程都有自己的副本，不可能共享的。

没有LOAD标记，这说明第三项和第四项（DYNAMIC这个entry下一节描述）都是和进程加载无关的（不占用进程虚拟地址空间）。GNU_STACK是用来告诉操作系统，当加载ELF文件的时候，如何控制stack的属性。这是和系统安全相关（通过stack来攻击系统），我们在[relocatable object file](http://www.wowotech.net/basic_subject/compile-link-load.html)的时候已经描述，这里略过（[https://wiki.gentoo.org/wiki/Hardened/GNU_stack_quickstart](https://wiki.gentoo.org/wiki/Hardened/GNU_stack_quickstart "https://wiki.gentoo.org/wiki/Hardened/GNU_stack_quickstart")中有更详细的信息）。

## 5、如何找到动态链接的信息

和静态链接的可执行序程序相比，DYNAMIC那个program header entry是动态库文件特有的。既然是动态库，当然要参与动态链接的过程，因此动态库的ELF文件需要提供一些dynamic linking信息给OS以及dynamic linker，DYNAMIC那个program header entry就是起这个作用的。dynamic segment只包含了一个section，名字是.dynamic。需要注意的是.dynamic section也是data segment的一部分被加载到了进程的地址空间中。下面我们仔细看看libfoo.so的Dynamic section的内容：
```cpp
Dynamic section at offset 0x5cc contains 24 entries:  
Tag        Type                         Name/Value  
0x00000001 (NEEDED)                     Shared library: libc.so.6  
0x0000000c (INIT)                       0x460  
0x0000000d (FINI)                       0x5ac  
0x00000019 (INIT_ARRAY)                 0x85c0  
0x0000001b (INIT_ARRAYSZ)               4 (bytes)  
0x0000001a (FINI_ARRAY)                 0x85c4  
0x0000001c (FINI_ARRAYSZ)               4 (bytes)  
……
```
我们先不着急看具体的各个项次的含义，我们先看看section table中对.dynamic的描述：
```cpp
Nr Name              Type            Addr     Off    Size   ES Flg Lk Inf Al  
……  
16 .dynamic          DYNAMIC         000085cc 0005cc 0000e0 08  WA  3   0  4
```
由此可知，.dynamic section是有Entry size的，也就是说，这个section中的内容是按照8个byte形成一个个的entry，下面的这个Elf32_Dyn（对于64bit的CPU，对应是Elf64_Dyn）数据结构可以解析这8个bytes：
```cpp
typedef struct {  
Elf32_Sword    d_tag;            /* Dynamic entry type /  
union    {  
Elf32_Word d_val;            / Integer value /  
Elf32_Addr d_ptr;            / Address value /  
} d_un;  
} Elf32_Dyn;
```
d_tag定义dynamic entry的类型，而根据tag的不同，附加数据d_un可能是一个整数类型d_val，其含义和具体的tag相关，或者附加数据是一个虚拟地址d_ptr。了解了这些信息后，我们可以来解析.dynamic section的具体内容了。

dynamic tag是NEEDED这个entry标识libfoo这个动态库依赖的object文件。ldd工具可以打印出给定程序或者动态库的share library的依赖关系，本质上ldd就是应用了NEEDED这个tag信息。对于libfoo.so这个动态库，它会依赖libc.so.6这个动态库，也就是c库了。不过，你可能会奇怪，我们c代码没有引用任何的c库函数啊，怎么会依赖c库呢？其实这和静态链接的hello world程序类似，我们在讲[静态链接](http://www.wowotech.net/basic_subject/static-link.html)的时候已经描述了，你可以在build libfoo.so的时候加上-v的选项，这时候你可以从不断滚动的屏幕信息中找到答案：你的c代码不是一个人在战斗。你可以可以从.text中看到一些端倪，例如.text中有一个call_gmon_start的函数，这个函数本来就不是我们的c代码定义的符号，我们的c代码只定义了foo函数以及xxx、yyy这两个变量符号。本来以为在.text中只有foo的定义，call_gmon_start是从那里冒出来的呢？实际上这个符号定义在crti.o中（在最后生成libfoo.so的动态库的时候，有若干个crt*.o参与其中）。libfoo.so定义了call_gmon_start这个函数，那么什么时候调用呢？这又回到了linux下动态库的结构这个问题上：虽然动态库定义了一些符号（函数或者全局变量），但是，我们希望在调用这些函数或者访问这些变量之前，先执行一些初始化的代码（这发生在动态库加载的时候，dlopen的时候，由dynamic linker负责）。这些初始化代码被放到一些特殊的section（例如.init），libfoo.so的.init section的反汇编结果如下：
```cpp
00000460 <init>:  
460:    e52de004     str    lr, sp, #-4!  
464:    e24dd004     sub    sp, sp, #4    ; 0x4  
468:    eb000009     bl    494 －－－－－以上来自crti.o

这里可以存放动态库自己定义的初始化函数，当然我们这么简单的动态库当然没有。  
46c:    e28dd004     add    sp, sp, #4    ; 0x4－－－－－－以下来自crtn.o  
470:    e8bd8000     ldmia    sp!, {pc}
```
INIT（对应.init section）到FINI_ARRAYSZ这些entry都是和该动态库的初始化和退出函数相关的。当dynamic linker open这个动态库的时候（dlopen）会执行初始化函数，当dynamic linker close这个动态库的时候（dlclose）会执行退出函数。还有很多dynamic tag，这里主要关注结构，暂且略过，一言以蔽之，dynamic linker可以通过.dynamic section找到所有它需要的动态链接信息。
## 6、动态库中访问全局变量

我们来看看foo中如何访问yyy这个符号的。yyy的重定位信息如下（.rel.dyn section中）：
```cpp
000086bc  00000815 R_ARM_GLOB_DAT    000086dc   yyy
```
符号表中可以查到GOT的位置：
```cpp
56: 000086ac     0 OBJECT  LOCAL  HIDDEN  ABS _GLOBAL_OFFSET_TABLE_
```
当然0x86ac是一个offset，并不是run time address，毕竟只有loading后才知道其具体的地址信息。如果该动态库被loading到address_libfoo，那么GOT实际应该位于address_libfoo+0x86ac。而yyy符号的地址在address_libfoo+0x86bc，dynamic linker会在适当的时间把真实的yyy符号的地址写入到这个位置的。由此可见，在offset是0x000086bc（GOT中的某个entry）的位置上保存了yyy符号的重定位信息。
```cpp
……  
568:    e59f202c     ldr    r2, pc, #44    ; 59c <.text+0x108> －－－获取GOT到当前指令的偏移  
56c:    e08f2002     add    r2, pc, r2 －－－－－－－－－－－－－－获取GOT的绝对地址  
570:    e59f3028     ldr    r3, pc, #40    ; 5a0 <.text+0x10c> －－－获取yyy在GOT中的偏移  
574:    e7921003     ldr    r1, r2, r3 －－－－－－－－－－－－－－从GOT entry找到yyy的绝对地址  
578:    e59f3024     ldr    r3, pc, #36    ; 5a4 <.text+0x110> －－－r3被赋值0x5678  
57c:    e5813000     str    r3, r1 －－－－－－－－－－－－－－－给yyy赋值  
……  
59c:    00008138     .word    0x00008138 －－－－－－－－－－－指令到GOT的偏移  
5a0:    00000010     .word    0x00000010 －－－－－－－－－－－yyy符号在GOT中的offset  
5a4:    00005678     .word    0x00005678  
5a8:    00000018     .word    0x00000018
```
虽然不知道GOT的绝对地址，但是在静态链接的时候，代码段的代码和GOT的偏移是已经确定的（loading的时候是按照program header中的信息进行loading，code segment和data segment是连续的），因此，在指令中可以通过59c这个桥梁获取GOT的首地址，加上entry偏移就可以获取指定符号的GOT入口地址，从该GOT入口地址中可以取出runtime的符号的绝对地址。

 _原创文章，转发请注明出处。蜗窝科技_
_[http://www.wowotech.net/basic_subject/pic.html](http://www.wowotech.net/basic_subject/pic.html)  

标签: [动态库](http://www.wowotech.net/tag/%E5%8A%A8%E6%80%81%E5%BA%93) [PIC](http://www.wowotech.net/tag/PIC) [位置无关代码](http://www.wowotech.net/tag/%E4%BD%8D%E7%BD%AE%E6%97%A0%E5%85%B3%E4%BB%A3%E7%A0%81)

---

« [Linux时间子系统之（二）：软件架构](http://www.wowotech.net/timer_subsystem/time-subsyste-architecture.html) | [Linux电源管理(14)_从设备驱动的角度看电源管理](http://www.wowotech.net/pm_subsystem/device_driver_pm.html)»

**评论：**

**hooong**  
2020-12-23 17:38

献丑了-_-  
Specifies the address to branch to. The branch target address is calculated by:  
1. Sign-extending the 24-bit signed (two's complement) immediate to 30 bits.  
2. Shifting the result left two bits to form a 32-bit value.  
3. Adding this to the contents of the PC, which contains the address of the branch  
instruction plus 8 bytes.  
PC = PC + (SignExtend_30(signed_immed_24) << 2)  
fffff3拓展为30bit,这个是负数,所以高位全置1,再左移2位,==>FFFFFFCC,即-52.  
offset = 100c0 + (-52) + 8 = 10094

[回复](http://www.wowotech.net/basic_subject/pic.html#comment-8166)

**hong**  
2020-12-22 19:45

修正一下  
readelf -s a.out:  
14: 00010094    36 FUNC    GLOBAL DEFAULT    1 get_data

[回复](http://www.wowotech.net/basic_subject/pic.html#comment-8165)

**hong**  
2020-12-22 19:35

窝哥,请教一下.  
$ readelf -r main.o  
00000008  00000b1c R_ARM_CALL        00000000   get_data  
$ readelf -s a.out  
14: 000100bc    36 FUNC    GLOBAL DEFAULT    1 get_data  
$ arm-himix200-linux-objdump -d main.o  
8:    ebfffffe     bl    0 <get_data>  
$objdump -d a.out:  
100c0:    ebfffff3     bl    10094 <get_data>  
  
S= 000100bc,P = 000100c0, 这里A要怎么计算呢?(A=feffff00?) (S+A-P)怎么算都得不到fffff3

[回复](http://www.wowotech.net/basic_subject/pic.html#comment-8164)

**cong**  
2018-07-13 16:28

大神可以列一下参考的书或者资料吗？应该不止csapp那本吧，怎么可以扣的这么细！

[回复](http://www.wowotech.net/basic_subject/pic.html#comment-6844)

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
    
    - [一例centos7.6内核hardlock的解析](http://www.wowotech.net/linux_kenrel/479.html)
    - [页面回收的基本概念](http://www.wowotech.net/memory_management/page_reclaim_basic.html)
    - [Linux kernel内核配置解析(5)_Boot options(基于ARM64架构)](http://www.wowotech.net/linux_kenrel/kernel_config_boot_option.html)
    - [蓝牙协议分析(4)_IPv6 Over BLE介绍](http://www.wowotech.net/bluetooth/ipv6_over_ble_intro.html)
    - [Linux TTY framework(3)_从应用的角度看TTY设备](http://www.wowotech.net/tty_framework/application_view.html)
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