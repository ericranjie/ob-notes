原创 程序喵大人 程序喵大人
_2022年04月26日 10:26_

# 前言

之前发过一篇面试题的文章，近期我花了大量时间整理出这篇，现在奉上。更文不易，感谢支持。

之前发过一篇面试题的文章，但是没有贴答案，近期我花了大量时间做了一下整理。

注意：

本文仅整理了那些面试过程中能突出你能力的问题和答案，那些常见的被问烂了的问题，我这里没有做整理，读者可以自己在网络上搜索，有很多文章。

本文目的是在你回答某个面试题时，提供给你回答本题以及延申的思路，篇幅有限，不会针对某一个知识点讲解的特别细。

不过，针对每个知识点，我都贴出了相关细节的链接，可以打开具体链接去针对性学习。

对const和volatile的理解

const表示修饰的变量不可修改，如果修改，那编译器会报错。如果对一个const修饰的变量取地址或引用之后再修改，那就会产生未定义行为，编译器不保证会得到想要的结果。

volatile有一点一定要注意，它和多线程一点关系都没有，它的作用就是内存可见性，防止编译器对volatile修饰的变量做一些相关优化，举个例子：

不对变量加volatile，编译器会对变量做一些优化：

![图片](https://mmbiz.qpic.cn/mmbiz_png/JeibBY5FJRBGraDdbA07L3HKZMKzlAfv2sFicGxZylaCnXtGCaQTtFgg14u5kwf88C49ZorH85Bq7sDVYTEKiaO0A/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

而加了volatile修饰，生成的汇编是这样：

![图片](https://mmbiz.qpic.cn/mmbiz_png/JeibBY5FJRBGraDdbA07L3HKZMKzlAfv2U7v1zHU2PfU9zjf492QZ82r1EK7QAzjW0gic5ChZ9T4jehzCXqiaXFNg/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

C++的volatile一般只会用在与硬件通信，平时我们编程几乎用不到。

具体可以看：https://en.cppreference.com/w/cpp/language/cv

Linux生成可执行程序的过程

四个过程：

1. 预处理，主要做了以下工作：

- 展开所有#define宏定义，进行文本替换

- 删除程序中所有的注释

- 处理所有的条件编译，#if、#ifdef、#elif等

- 处理所有的#include指令，把这些头文件的内容都复制到引用的源文件中

- 添加行号和文件名标识，方便编译器产生警告及调试信息

- 保留所有的#pragma编译器指令，因为编译器会使用他们

3. 编译，把预处理后的文件进行一系列操作生成相应的汇编文件

- 词法分析：又称词法扫描，通过扫描器，利用有限状态机的算法将源码中的字符串分割成一系列记号，如加减乘除数字括号等。

- 语法分析：使用语法分析器对词法分析产生的记号运用上下文无关语法的手段进行语法分析，产生语法分析树。这期间如果表达式不合法（括号不匹配等），就会报错。

- 语义分析：语法分析检查表达式是否合法，语义分析检查表达式是否有意义，如浮点型整数赋值给指针，编译器就会报错。

- 中间语言生成：做一些语法树优化，如6+2=8。

- 目标代码生成及优化：将中间代码生成目标汇编代码。

5. 汇编， 使用汇编器将汇编代码转成机器可以执行的指令，其实就是将汇编指令和机器指令按照对照表一一翻译。

1. 链接，上面处理的基本单元都是源文件，多个源文件经过上面的处理后变成多个目标文件，链接就是将多个目标文件打包组装成一个整体的过程，这个整体就是可执行程序。

想更深入了解的朋友，建议看看《程序员的自我修养》，也可以看看我的总结篇：

[gcc a.c 究竟经历了什么？](http://mp.weixin.qq.com/s?__biz=MzkyODE5NjU2Mw==&mid=2247484769&idx=1&sn=6db2570301a2500d8edc88a2263e6570&chksm=c21d37ddf56abecb7100571ab45863cdf5e54142c561f698478984f8093ee413db44e9da9b9a&scene=21#wechat_redirect)

Linux程序是如何运行起来的

这块涉及到很多知识点：

- 什么是进程，进程是如何建立的？

- 什么是虚拟内存？为什么要有虚拟内存？

- 程序如何运行起来的？

1. 通过fork系统调用创建一个新的进程

1. 通过execve系统调用执行指定的ELF文件，附带环境变量和参数

- 检查ELF可执行文件的有效性，比如魔数（通过魔数可以确定文件格式）、Segment的数量等

- 寻找动态链接的段，设置动态链接器路径

- 根据ELF可执行文件的程序头表描述，对ELF文件进行映射，比如代码、数据、只读数据

- 初始化ELF进程环境

- 将系统调用的返回地址修改为ELF可执行文件的入口地址

1.

具体可以看：[Linux可执行文件如何装载进虚拟内存](http://mp.weixin.qq.com/s?__biz=MzkyODE5NjU2Mw==&mid=2247484764&idx=1&sn=d46f2056dc18279a5968fd446f441b7b&chksm=c21d37e0f56abef6d52d145187f2be962ee94023aadf472810435f0b9f9c592b054da5882bbe&scene=21#wechat_redirect)

内存缺页的过程

这块涉及很多前置知识点

- 虚拟内存管理

- 虚拟内存与物理内存的映射

- 页表和页面的概念

内存缺页就是要访问的页不在主存中，需要操作系统将页调入主存后再进行访问，此时会暂时停止指令的执行，产生一个页不存在的异常。

对应的异常处理程序就会从选择一页调入到内存，调入内存后之前的异常指令就可以继续执行。

缺页中断的处理过程如下：

1. 如果内存中有空闲的物理页面，则分配一物理页帧r，然后转第4步，否则转第2步；

1. 选择某种页面置换算法，选择一个将被替换的物理页帧r，它所对应的逻辑页为q，如果该页在内存期间被修改过，则需把它写回到外存；

1. 将q所对应的页表项进行修改，把驻留位置0；

1. 将需要访问的页p装入到物理页面r中；

1. 修改p所对应的页表项的内容，把驻留位置1，把物理页帧号置为x；

1. 重新运行被中断的指令。

具体可以看：[操作系统内存管理，你能回答这8个问题吗？](http://mp.weixin.qq.com/s?__biz=MzkyODE5NjU2Mw==&mid=2247484706&idx=1&sn=f99afd2c8c38d97d0e4d333f9c821143&chksm=c21d379ef56abe881bc15e963ea0a8417fb2e80ca022d86024be1d9dbe6e6009396520031e0d&scene=21#wechat_redirect)

函数调用的过程，调用时栈空间是如何变化的？

函数调用都是以栈帧为单位，通常用栈来传递函数参数、保存返回地址、保存寄存器（即函数调用的上下文）及存储本地局部变量等。

一个单独的栈帧结构如图：

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

为单个函数调用分配的那部分栈称为栈帧，栈帧的边界由两个指针界定：寄存器%ebp为帧指针，指向当前栈帧的起始处，通常较为固定；

寄存器%esp为栈指针，指向当前栈帧的栈顶位置，当程序执行时，栈指针可以移动，因此大多数数据的访问都是相对于帧指针的。

调用函数时，栈帧指针会进行相应的变化，为函数调用开辟空间，当函数返回值，栈帧指针会恢复原状。

一次函数调用的栈帧图如下：

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

具体可以看：[图解Linux是如何进行函数调用的？](http://mp.weixin.qq.com/s?__biz=MzkyODE5NjU2Mw==&mid=2247484759&idx=1&sn=cf1b674074da02041df517d9f66f5c64&chksm=c21d37ebf56abefd8eb96446323d0880e00ac1c1e399122460ff0e73eec37610d9c10371a651&scene=21#wechat_redirect)

auto的推导规则

这块有个前置知识点，即cv，cv可不是指ctrl+c/v，而是指const和volatile。

有个这个cv概念后，auto的推导规则就比较简单了：

- 在不声明为引用或指针时，auto会忽略等号右边的引用类型和cv限定。

- 在声明为引用或指针时，auto会保留等号右边的引用和cv限定。

这里补充一个decltype的推导规则：decltype不会像auto一样忽略引用和cv限定，它会保留这些东西。

对于decltype(exp)：

- exp是表达式，decltype(exp)和exp类型相同。

- exp是函数调用，decltype(exp)和函数返回值类型相同。

- 其它情况，如果exp是左值，decltype(exp)是exp类型的左值引用。

左值和右值的概念

面试时可以这样回答：能取地址的就是左值，不能取地址的就是右值。

具体可以看下这个链接：https://stackoverflow.com/questions/3601602/what-are-rvalues-lvalues-xvalues-glvalues-and-prvalues

这里还需要了解一个知识点，移动语义。

官方介绍是：移动语义主要是提供将昂贵的对象从内存中的一个地址移动到另外一个地址的能力，同时窃取源资源以便以最小的代价构建目标。

它的主要用途是使能移动构造函数和移动赋值运算符，使得我们在使用临时对象时，减少不必要的拷贝操作。

使用多线程需要注意些什么？

- 使用thread要注意调用join或者detach

- 不要迷信多线程，使用多线程之前做好技术评估，为什么要用多线程

- 注意内存同步问题，考虑使用锁或者原子变量

- 使用锁时考虑使用RAII锁，可以进一步说明unique_lock和lock_guard的区别，进一步优化的话可以提出读写锁的概念

- 再提出死锁的概念，引入死锁的四个必要条件，以及怎么解决死锁（一般就是控制加锁的顺序），可以再引出C++17的scoped_lock

- 注意使用条件变量的坑，这块可以提出虚假唤醒和信号丢失两个概念。

具体可以看这篇文章[使用条件变量的坑你知道吗](http://mp.weixin.qq.com/s?__biz=MzkyODE5NjU2Mw==&mid=2247484746&idx=1&sn=177d6f687413eee2865a108db7d22b49&chksm=c21d37f6f56abee070e3ab0b043756cbbf19b53403f8d98bee5d273a0aee85a2311319c69706&scene=21#wechat_redirect)

对atomic的理解

我们固有的理解肯定是atomic是原子操作，比锁快，这里有一个问题：atomic真是原子的吗？

> cpp原子库有些实现是调用操作系统提供的接口，所以这个实现可能跟操作系统有关。同时，又因为操作系统的实现是依赖于硬件的，所以具体的锁的实现要取决于硬件的支持情况。
>
> 题主有一个误区，因为原子操作一定需要关调度，关中断，这个理解是错的。锁的本质是同步数据，理论上说，只要保证特定的数据不被修改即可。在硬件层面上看，就是特定的物理内存，特定的cache line，不被修改，所以，并不是所有原子操作，都一定要关调度，关中断。

> 大多数主流的CPU，都会提供硬件指令，原子修改某个内存，对于Intel的CPU，手册里有详细的描述。
>
> 如果操作系统使用的是硬件指令实现原子操作，那么用这些指令就可以了，这种实现不需要锁，不需要关中断或者调度。
>
> 对于ARM/PPC/RISCV也有类似的指令。
>
> 但是，也有例外情况。

> 比如ARM32/PPC32/RISCV32上不支持对8B数据的原子操作，只有Intel在32位环境中提供了CMPXCHG8B的指令（甚至于，Intel还提供了CMPXCHG16B的指令），这种情况就比较麻烦了。
>
> atomic库里是提供了各种长度的数据的，如果硬件本身不支持，那么就需要通过软件实现。atomic库里有std::atomic_is_lock_free来告诉应用程序，这个操作是不是lock free的，如果不是，那么底层就是用锁来实现的。

> 如果不是lock free的，那么其实它的实现跟用户自己用锁来实现是差不多的，甚至性能还不如用户自己实现。通过软件实现的原子操作是需要关中断，关调度，使用mem fence等动作保证数据不被改变，如果是用户自己的代码，确认中断不会更改关键数据的话，那么可以不用关中断。
>
> 某些开源库并没有考虑到不同硬件上的原子操作差异，所以在某些平台上，使用了原子操作的开源库实际上是会有风险的（比如openmp的kmp库在PPC/RISCV上）。
>
> 所以，原子操作可能是虚假的（软件模拟），也可能是真实的（硬件指令），在X86平台上，基本上都是真实的，在非X86平台，有可能是虚假的。

上述这一段回答引用自知乎，具体可以看源链接：https://www.zhihu.com/question/469476598/answer/1979378196。

在回答atomic相关理解时还可以延申到atomic的6种memory_order上，详见https://en.cppreference.com/w/cpp/atomic/memory_order

CAS ABA问题

CAS即compare and swap：通常会记录下某块内存中的旧值，通过对旧值进行一系列的操作后得到新值，然后通过CAS操作将新值与旧值进行交换。如果这块内存的值在这期间内没被修改过，则旧值会与内存中的数据相同，这时CAS操作将会成功执行 使内存中的数据变为新值。如果内存中的值在这期间内被修改过，则一般\[2\]来说旧值会与内存中的数据不同，这时CAS操作将会失败，新值将不会被写入内存。看代码：

```
int cas(long *addr, long old, long new){    
```

ABA是无所结构常见的一种问题：

1. 进程P1读取了一个数值A

1. P1被挂起(时间片耗尽、中断等)，进程P2开始执行

1. P2修改数值A为数值B，然后又修改回A

1. P1被唤醒，比较后发现数值A没有变化，程序继续执行。

对于P1来说，数值A未发生过改变，但实际上A已经被变化过了，继续使用可能会出现问题。在CAS操作中，由于比较的多是指针，这个问题将会变得更加严重。

具体可以直接看维基百科：https://zh.wikipedia.org/wiki/%E6%AF%94%E8%BE%83%E5%B9%B6%E4%BA%A4%E6%8D%A2

无锁编程

即不用锁编程，一般场景就是实现无锁队列，这里我贴一个比较好的无锁队列的实现：https://github.com/taskflow/taskflow/blob/master/taskflow/core/tsq.hpp

如何设计结构体？

这块考察的就是内存对齐的知识点，即如何能把结构体设计的大小更小一些，即巧妙的运用内存对齐的原理，减少结构体中hole的出现。

具体看这：[内存对齐之格式修订版](http://mp.weixin.qq.com/s?__biz=MzkyODE5NjU2Mw==&mid=2247484780&idx=1&sn=e09722a29ce08b81bb4d7782685478f2&chksm=c21d37d0f56abec63473a3dfb0fd234083d772a91d4aa03cec1abe07307e14482a878eb87d09&scene=21#wechat_redirect)

介绍一下返回值优化

返回值优化就是函数在返回值时避免了临时对象的创建，减少了拷贝操作。C++11后默认时开启的返回值优化，可以在编译时使用-fno-elide-constructors指令来关闭返回值优化。

这里可以多提一点，就是什么情况下不会触发返回值优化：

- 有一个典型的场景就是函数内有if-else分支，在分支内return是不会触发返回值优化的。

- 另一个典型场景就是返回的是非局部变量，这都不会触发返回值优化

具体看这：https://shaharmike.com/cpp/rvo/

用过哪些设计模式？

列举几个常用的设计模式：

- 单例模式

- 策略模式

- 适配器模式

- 原型模式

- 代理模式

- 责任链模式

- 观察者模式

具体看这：[一文让你搞懂设计模式](http://mp.weixin.qq.com/s?__biz=MzkyODE5NjU2Mw==&mid=2247484791&idx=1&sn=82ec021524713d413722f8a70e995917&chksm=c21d37cbf56abedd05ac2f45f94cc011133b742977f2952e28a96260fe192807f0347565d5fb&scene=21#wechat_redirect)

我自己也买了一本关于设计模式的优质电子书（https://refactoringguru.cn/design-patterns），来这里的应该都是付费的朋友，想看这本书可以加我微信好友cpp-father免费领取。

智能指针用法

智能指针应该是C++面试过程中必问，这块有几个重要知识点需要了解：

- shared_ptr与unique_ptr的区别，它们各自的原理

- shared_ptr可以多个指针共同指向并管理一个对象，当所有指针生命周期都结束后，对象自动销毁，原理在于拷贝析构时候引用计数对应的加和减

- unique_ptr只能一个指针指向一个对象，不能拷贝，只能move，原理在于unique_ptr的拷贝构造函数使用了delete修饰。

- 为什么要引入weak_ptr

- 为了解决shared_ptr循环引用的问题

- 为什么要引入make_shared

- 更加安全

- 对象内存和引用计数在一块连续内存上，可以更好的命中Cache

- 为什么要引入enable_shared_from_this

- 为了让类可以返回指向自身的智能指针

- 因为不能直接使用this包装成shared_ptr返回，那样对象就会被delete两次，crash

- 手写shared_ptr

- 这块可以看[智能指针](http://mp.weixin.qq.com/s?__biz=MzkyODE5NjU2Mw==&mid=2247492538&idx=1&sn=195554e6c6b40fac0d5b3bdc33cc4b44&chksm=c21ed106f56958102a61e6a3a660bb0a07ad676042327e16577d56922ec4e3e095232cc607ee&scene=21#wechat_redirect)（这里面的代码有一点点bug，具体去评论区里看一下）

对协程有过了解吗

（其实我对协程的理解也不是很透彻，后续我会抽出一段时间来研究协程，然后做一次分享）

协程分为有栈协程和无栈协程：

- 有栈协程：goroutine和libco都是有栈协程，主要就是保存恢复各种当前上下文状态，可以通过ucontext相关调用来实现

- 无栈协程：基于状态机，不需要切换栈帧，C++20支持的就是无栈协程，需要编译器提供相关的语义支持，比如yield、async等，性能较高，但使用起来很麻烦。

贴一个大佬写的简易协程库：https://github.com/chenyahui/AnnotatedCode/blob/master/coroutine/coroutine.c

再贴几个协程相关的链接：

- https://juejin.cn/post/6961414532715511839

- https://zhuanlan.zhihu.com/p/347445164

- http://warmcat.org/chai/blog/?p=5558

锁和原子的选择

普通变量优先使用原子操作，较大的临界区逻辑代码使用锁，其实如果是较大的类对象，尽管我们使用atomic修饰的，它底层还是使用的锁。

可以用is_lock_free()来判断：

```
#include <iostream>
```

C++如何实现反射

- https://zhuanlan.zhihu.com/p/158147380

- https://zhuanlan.zhihu.com/p/337200770

- https://www.zhihu.com/question/62012225

反射这块挺有意思，但一两句话说不明白，后续我会单独写篇文章介绍，反射最简单的实现就是可以使用一个map\<std::string, function>，然后根据key找到相应的function去执行。

OS中断和异常的区别

在OS中，中断可以理解为是外部事件触发的事件，而异常是内部执行指令引起的事件。

中断（外中断）：IO中断，硬件故障等

异常（内中断）：系统调用，page fault等

C++怎么做到的运行时多态

这里其实考的就是虚函数表的概念，具体看这篇文章：[面试系列之C++的对象布局【建议收藏】](http://mp.weixin.qq.com/s?__biz=MzkyODE5NjU2Mw==&mid=2247484758&idx=1&sn=4e614430f666f63ab135c13a716d07c1&chksm=c21d37eaf56abefc8d2a1dc3e09a8146d242475cb0900ee5a94ab6a94a991168a887f7351821&scene=21#wechat_redirect)

死锁的检测与解决

死锁的四个必要条件：

1. 互斥条件：一个资源每次只能被一个进程/线程使用

1. 请求与保持条件：一个进程因请求资源而阻塞时，对已获得的资源保持不放

1. 不可剥夺条件：进程已获得的资源，在未使用完之前，不能被强行剥夺

1. 循环等待条件：若干进程之间形成一种头尾相接的循环等待关系

我们平时开发中基本没有用到解决死锁的方法，都是在开发过程中要避免出现死锁，一般都是要在加锁过程中注意加锁的顺序。每个进程确保请求锁的顺序相同。

这块也可以使用std::scoped_lock（C++17），它会自动控制加锁的顺序。

具体看这个：[多线程中如何使用gdb精确定位死锁问题](http://mp.weixin.qq.com/s?__biz=MzkyODE5NjU2Mw==&mid=2247484777&idx=1&sn=330eefbb1f18d9e251af57052a544f74&chksm=c21d37d5f56abec30d9029d0f4e90440be23a876eae99736ce636cf9f896670e274220705283&scene=21#wechat_redirect)

malloc是怎么实现的，为什么free只需要传内存地址，不需要传内存大小？

因为malloc时候传了内存大小，内部会自动在malloc返回地址之前的16个字节内存储了内存大小，在free时找到这16字节，就能确定相应的内存大小。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

具体可以看这篇文章：[10张图22段代码，万字长文带你搞懂虚拟内存模型和malloc内部原理](http://mp.weixin.qq.com/s?__biz=MzkyODE5NjU2Mw==&mid=2247484726&idx=1&sn=18d9dc7f8b76a2a9a0b29b39ff6dabea&chksm=c21d378af56abe9c56f3d4da55b4d2d90995bae0e1a4a13c3e5cf7f33473d2642615b445aeb1&scene=21#wechat_redirect)

C++中锁有多少种

参考cppreference：

- std::mutex

- std::timed_mutex

- std::recursive_mutex

- std::recursive_timed_mutex

平时开发使用std::mutex就足够了，我也是只使用过std::mutex。

轻量级锁（fast userspace mutex）是怎么实现的？

futex可以简单理解为是系统mutex的一个优化产物，也可以理解为是用户态的mutex，内部有个变量标识竞争状态，在无竞争的情况下不会触发系统调用，不会进入内核态，只有在有竞争的情况下才会触发系统调用，进入到内核态，达到真正的mutex效果，这样的目的就是减少系统调用的次数，优化性能。

具体可以看这个帖子：https://groups.google.com/g/pongba/c/ifN9j1608f0

还有这个：https://linux.die.net/man/7/futex

自己实现一个shared_ptr

直接看这个：[手撸一个智能指针](http://mp.weixin.qq.com/s?__biz=MzkyODE5NjU2Mw==&mid=2247492538&idx=1&sn=195554e6c6b40fac0d5b3bdc33cc4b44&chksm=c21ed106f56958102a61e6a3a660bb0a07ad676042327e16577d56922ec4e3e095232cc607ee&scene=21#wechat_redirect)

string的实现？为什么设计成两种模式？为什么16字节以下用栈内存，以上用堆内存

简单贴一下源码：

```
enum
```

注意这个union，这是string的一个优化点，作用就是节省内存。

当长度\<16，就会使用局部的栈内存，当>=16时，才会重新申请一块堆内存用于存储里面的数据。

内存池的设计方案

没有最好的内存池设计方案，只有最适合的方案，不同的项目针对内存的需求是不同的，至于内存池的方案则需要因地制宜。

这里仅介绍内存池的思想：在真正使用内存之前，预先分配一定数量的内存出来，当有内存需求的时候，直接从这个池子里面取内存，而无需陷入内核通过系统调用申请内存，归还则类似，直接把它还回池子里，方便下次再用。

具体可以看下这位大佬的文章，他的这个内存池就是专为自己项目而设计的：[【性能优化】高效内存池的设计与实现](http://mp.weixin.qq.com/s?__biz=MzkyODE5NjU2Mw==&mid=2247491272&idx=2&sn=a301597e71c3945627fda5429da2c1b1&chksm=c21d2c74f56aa5627a855a20ab64afdd32c792239a0c895e3505e5bee5d61dcaebc8f0313e42&scene=21#wechat_redirect)

模板使用的优缺点

优点：

- 比较灵活，可重用

- 通过编译期自动生成代码，大大减少了开发时间

缺点：

- 调试起来比较困难

- 一般情况下模板中函数的实现都会放在头文件中，会暴露出实现细节

- 大量使用模板，会导致编译时间较长。

好的，就整理到这，有任何问题欢迎留言讨论。

![](https://res.wx.qq.com/op_res/NN_GToMiIjsXzgPzF9-74ZzwR3cA9-fv3o9eWo8f5gQWqx71CmGlY8kFxuIxZaG0TB1bFeMCmh1DGN_pWMRg0A)

**帐号已迁移**

公众号

![](https://mmbiz.qlogo.cn/mmbiz_jpg/cVRiaDiak22mwiavM3vMdDewC7JicatYWEHr0HouU30MEjU4zHYYeQf1TwDNtAYTRm44p9DciaGgrxJqiaL4unX81jnw/0?wx_fmt=jpeg)

程序喵大人

![赞赏二维码](https://mp.weixin.qq.com/s?__biz=MzkyODU5MTYxMA==&mid=2247493420&idx=1&sn=db44458f9ea7c2b8a364ffc4bb0e9091&source=41&key=daf9bdc5abc4e8d069b4ba489e3af78f2e5433a6dffdf4f63164db21d3b96153edc3fa2bf122398900116670dd5f0461e78a3f57f7a9bbf2431feace156f401e9449f70ccb5d105844a950109f48385c390d08883cb2cc87c3023a9d0c9e0f81726afa57c7afbb4b64af1875bdf2b54886cb075754facb86293e8feea4bebb53&ascene=0&uin=MTEwNTU1MjgwMw%3D%3D&devicetype=Windows+11+x64&version=63090b19&lang=zh_CN&countrycode=CN&exportkey=n_ChQIAhIQ82vUMbPP3%2FyyP8QoM7ZXJRLmAQIE97dBBAEAAAAAAMMrJmPoIeoAAAAOpnltbLcz9gKNyK89dVj0qGpr%2FZc8R3fZjRDGk6lehHOaoFDNe3UIKGBUDEuAdKao3Eai5I%2BgQEeip3paLKz3P8Wqv0lQ5u8TKMQBUFIo3xoAGGQ9xVG1dissS%2BU8mIzk3TaY8%2BF3N0%2F47MABQUsnI8G0OG4YR2z7Zc6nFo1evfh%2BQhBDy5yVThHFmIMWWtbeCMUJPiiyEipzuzhR%2BJwPqITetqNr%2Fja6UgXyWOO4UeyQ4zwSch7VStQLTxvVj8Zyiqnjmg8ihbpHwx9G8013&acctmode=0&pass_ticket=9lx62E2mRBV8a7TA%2FwxjVf8zLt8e0upbueBOlwymeuiLaCIe96ivJK3NwMER7%2BKN&wx_header=1)钟意作者

201 人付费

![](http://wx.qlogo.cn/mmopen/I1YzhXxW8YAlDt6xrX0KKk4nqnaTfKoJWGCuopVvweLtdWQGxEuPKlQibyPOicxbF4ibHwsFibNic7X6YCaibsLYl76RvPSItJqhHLicKp2DEHVkthDukuQmdaCuEgC2YHpoibTV/64)

![](http://wx.qlogo.cn/mmopen/ajNVdqHZLLBa4f8GHlzedr7cYicsicZIoPqgO2X1ibBQKmugC1celBckiccorbt3drNK8xXqXWJakSbp1xNiawjLQ0A/64)

![](http://wx.qlogo.cn/mmopen/lW5Nk2bsL2n3aCSQs9NEkBJx7da5iaHmic4v4ydcclotRcYq4UwD4u03QUypNL9qMz6sz1AoumJiaEWSmyvzLbHvE9f5tdM2ze1/64)

![](http://wx.qlogo.cn/mmopen/I1YzhXxW8YD7Vw5vIbZ8J74h5vqtm71MyGksDeAT12qq8qjyDUzygL7iaEq1zwYiaAN98sthqo3ibegGY5JSL1KD4YfotEVul2nY668xcQuDmWtc1mLZpQwdwVguxZCOFgl/64)

![](http://wx.qlogo.cn/mmopen/ajNVdqHZLLCw7ZqHEGibFofMiaAa2v1d1IFPDQB2nnbaM457swbE3VpQlbgNibkffjADDywkBKDibrhEChXAvDvlUSy8qQ8Q1iaatacseUiaMlxbgQV2mCDPwaTv3schRSYCQc/64)

![](http://wx.qlogo.cn/mmopen/REdvicaEDISRrdGtS47mNmyCCiaCnwTuDiaE1FxYnn2qLiatoKoEjjNBnNztZiatfyRibicJSWibbraCkyQpQkQlnruegOQ5icrandDsP/64)

![](http://wx.qlogo.cn/mmopen/PiajxSqBRaELntWczrUiaPquRNkrr3UG8gMGiaKahvaVH2JwLx28pic6obfGKo0Mz2722RNbf0ibUPhkXm9IH2hwlszHF6Q3pCxK6sxNS0Sib2vzr4NQegZo8Z87xvfPib7Sqbg/64)

![](http://wx.qlogo.cn/mmopen/ASFxIBJQ90KWN7huMjsEaZfuXqDWFw7oA2ohVbeibG36ntu318EMY9A311wf6iad8bPoegQQoW6Sq9jAFllvzicdXJFJLxZOzV0/64)

![](http://wx.qlogo.cn/mmopen/ajNVdqHZLLAsTq7sic23ibXrFTq5T7pTUODHibxX5cmTA0F5AKtXYxuaOmhUUm3picia3ENO8ibML0n5jcAX2MGZOynghxSmoNe9mUCySnfkMSEC8/64)

![](http://wx.qlogo.cn/mmopen/icWN8ZNEXrkNTRiaGbnnrtmTBDbjBQQwHgHjCpIDVXdLydliaYzMsrzfrXQXWXia4zsn20ZFTwDqVhBsNvr7lGMorQNfuoKOCNkz/64)

![](http://wx.qlogo.cn/mmopen/lW5Nk2bsL2lz6SoTAnMAOVZP934nKPzYwGzFyMBKrSp4sZbPAEcmIdHDfUicMHk0PKWP2eSbyBF3jxOaUUZO13cx3m11VtSiatuRSr2GdAglJk0PPUgJicPD7Zicqcx1zwfs/64)

![](http://wx.qlogo.cn/mmopen/ajNVdqHZLLBKLy64ibq3sL75NbbPXEkzBD1F6oRHrQssgOWPaI2kVkFpsj2Yw64sOR4ic85uJ9LQcw6acJ97WuyA/64)

![](http://wx.qlogo.cn/mmopen/I1YzhXxW8YDSJDubGAmWwVvrXBFc30tEAUH9CAdcb2KjpiaxgmafEicZibw7Hjc4n8JibPOGjyMG0ohh7rqAicURwVFnKV3oeicQOWyXWAsB1Z86PXIE4XmjQQ0LibCBA4iaNbLB/64)

![](http://wx.qlogo.cn/mmopen/REdvicaEDISRzRliaYU7dOgZoD3n53Bek9TagC3oZ3OyrNiaJ9fcu7A2Qzqkp0yYL3KqNhUQIwA5TIichhTj3Q680iamOjMhJ2dz1ANu9ic72buo0WXNURRmpMY70upmNdwpPU/64)

![](http://wx.qlogo.cn/mmopen/I1YzhXxW8YC0HEsPW5ibyu3AnQ9qZFw3SYEGeic3JpABG2KLicnE7x3xUaibiaPa8GDhznDg3S9vpeFZ1bO82GEzeWpcSlkwy8ibpJ/64)

![](http://wx.qlogo.cn/mmopen/icWN8ZNEXrkNTRiaGbnnrtmeQWiaAibv2HsphaGgvHeF3FKLicDXVOPwian5X4tSMJfCbf9FHsy4FlkYarPODAL1YhtMv8VHmuic8Is/64)

![](http://wx.qlogo.cn/mmopen/REdvicaEDISRrdGtS47mNm3ib6Tkw4j3J9FdvhQ7Xh8osFPtY70N4HHib9oicGcbnSrZ0geBG9tEMoIhx5L6w69fvbictpiazOVia6G/64)

![](http://wx.qlogo.cn/mmopen/lW5Nk2bsL2nEibICF2BXFPyJklYEljSfZ7JuuOYgbvJ7Hu2nibD0sicDPnVKtHdMbeGG4Ilu85YKTfZCD5lM6qJvx1mtRyc7mmrf1DubxN0c9luXG0bMZeRlyjQV8lMjL6Q/64)

![](http://wx.qlogo.cn/mmopen/REdvicaEDISQVQQ0f672OPLkayKYmBchtouFDAqFzzhicGeHaGibNFVO4vfAsCKm5eFVssoMBQwra9z3OricePf03q2hlxSA5B5T/64)

![](http://wx.qlogo.cn/mmopen/PiajxSqBRaEJeZzHNX7eMZqzAJJiaJia3cxGuicdYrn7AcmhnkuebJmvxGWwc5Ae7H14VnAIYKV5KDfLKr2egs7vIQ/64)

![](http://wx.qlogo.cn/mmopen/icWN8ZNEXrkMviczia0HjhIviaX6Tg8H3kL7rE0zAP2hueLZGM9jsJauheyd6hQrLJ7KWdamzBVJasn6ezepribbPIl5AMxEJvA9VcWHDiciccibknibKy98jE49y7pOyibWsyGxw8/64)

![](http://wx.qlogo.cn/mmopen/lW5Nk2bsL2nEUMN39zIxaRBpH6jtCibuLoOpoUjJaKuYmxj2chJ2k45yWg3ib6XfT34HHS9pTlygIvib0qF7gsOltV0fvnbn76M/64)

![](http://wx.qlogo.cn/mmopen/wFnpEhoSNd0txxTzRuLEeWibOLWD71JzC3proInBf8OichNQ06AsEPdTCiamnicJqKyiacYHSx4d5UUrn7bCZ9z8eMOoKlx7eSe1ToBcViacSoVkic12FjCSxkDwpPwsoQGdHKS/64)

![](http://wx.qlogo.cn/mmopen/icWN8ZNEXrkMUtTeq5hEKNmj1xiah3TFlTdqwYQZJdf1viaiaxvqd3mxVdvwhL6f45WjKbicweHxKKgY2bCabeeTb5Q7bib99ldMXf/64)

![](http://wx.qlogo.cn/mmopen/icWN8ZNEXrkOHuHcz0BpgsFT3Ab0mG68Qljss7nZK24ic1xVqgyUBGneXn8srL4ytKaMzqQcTOTjtQHFv8rUbGuw/64)

![](http://wx.qlogo.cn/mmopen/I1YzhXxW8YB83bXTG9vTTu04vGQdsz4IGtc2ZAcsFFU4vjdqfWHtIEsZZb4dcX72fRcOVyD0M2mxOGGvUuEtrBhKt8bGox6TGlqlKVorGbVVSDzd7kO711TuSXlALsBY/64)

![](http://wx.qlogo.cn/mmopen/Q3auHgzwzM4HrMxSfqgyXb267iabM0d9aoUGWASawoxWr7ta3DVOMzN2cJBzSibflIQX3U1oOyicdics7KF9oasEWgjPCxewT8bicTqJyaCfZTy4/64)

![](http://wx.qlogo.cn/mmopen/lW5Nk2bsL2kf0IMr8AyGp4Xl1TicP9P6Imq3hIQdIJHbicBa7EPmneHERW0RbgCDGyU6oavPYqh0G1ydmIUROmOEZPu7PDYFyS/64)

![](http://wx.qlogo.cn/mmopen/lW5Nk2bsL2miaLEQoZUOfQ5PaicvMT16KShmc6EPib250pS1xL7ZfeYTnXdibtm3l2UXgaK7F10GgrOUPic76rynwGjKHgyDtnwbfDvhuEMx8Bvuzc47ckbDzSdyAL54ZjlA5/64)

![](http://wx.qlogo.cn/mmopen/Q3auHgzwzM5MgicDMbNFID3sAZX7dDxJfibEjrEjoltounicIbJnvc50IllKuEYazV5gEIzdI6mF3x5Bw4HxxUFCwbtG0Ir1FHuw4BN8ribfYnfhdkTcepxdVC1qzWl4UytUpaL3DhHu08E/64)

![](http://wx.qlogo.cn/mmopen/lW5Nk2bsL2nhUUgNZYhZnVIO7xicz5sBlxCoGCtwZ3bJHPYwX1mYV98U2KiakXer9dzy9h7hBEU7NVLJhBFJlLcChxfJVrV0qU/64)

![](http://wx.qlogo.cn/mmopen/icWN8ZNEXrkNTRiaGbnnrtmXTXDZnG5XMHRWiaGmkQBoufzMdP9Pm62LKHSCibdvAcMCia58qeqO441lDj4tMnPic8huDy3ibj2omfP/64)

![](http://wx.qlogo.cn/mmopen/icWN8ZNEXrkMUtTeq5hEKNo7uHROSIM7HuBZX89OgNlrvM8Wl3sIYViarlKtwYBQXV57kMj5nicapoIeYBURiaPtlKnUKN1EvgNh/64)

![](http://wx.qlogo.cn/mmopen/lW5Nk2bsL2kmyZuAxRo23xuNDWJWyCQsxlXQUpV3zuSMEsXvx7OYu52cDxrG9p4nFA1yqicofHKL8iace9gicGAYBp3w9e2pe0pKB6Zj9QXrXdia1iccw3ics8tCkw1xibp7gTQ/64)

![](http://wx.qlogo.cn/mmopen/I1YzhXxW8YAiaaR5OjJ8ZiclLV4CYVC9tWFsLBUVmBNeE6LfphQSorBzyYwfqjVaJeA3Dd8mKZjXWNyZ49licupRzgZlrw2nYKK/64)

![](http://wx.qlogo.cn/mmopen/lW5Nk2bsL2k7hib6NgLxD8D0BY8z2vsgddic3qaenPYRR62GWxEmmaibHibmKaSEDdZ90bDnLa098T6dePJVkeWenehSI3EvNk6g/64)

![](http://wx.qlogo.cn/mmopen/REdvicaEDISSNqqqGrGqswPP0eWZm7fITWRnIW0v5JjibqBdhBrLP4txFTjN5VwuZ90ndUBFROGLIB31uicxP2ApJ1VIH52aAX1/64)

![](http://wx.qlogo.cn/mmopen/icWN8ZNEXrkOicIWJpMAzxSwocj7B988HKnlfztD07et0Kfnss5YauE2RE56yicFH5VdPwuIk575wiaIPrDenIktXGIDe6DRQmmX/64)

![](http://wx.qlogo.cn/mmopen/REdvicaEDISTJaJozQS0eatpDhXCw3uicQrxPud4iaGmibAHzuUl6iaiaxyR3vs3cWLzsRGM37ib7cJ0Lw5icqP8PiaHJgorunr61p41y/64)

![](http://wx.qlogo.cn/mmopen/Q3auHgzwzM5cCT671tjj5YGk9EXkdD7zNDJApicpib3QFlBBicmgiauwWH6xvma74ictj5S5nD5AykHicmDSe9a7gk5Ce8x2npllqXdSfepyXfTLoibJoLPJDDM6VPMLfAgCmDY/64)

![](http://wx.qlogo.cn/mmopen/icWN8ZNEXrkOIgrMwXe1LOialRUDDh4Pf0qXk3MqNr2afXVxamYHbibUAP66MiaUBJvGVsk8dqQD6EFfy4WvTiaRKlVGceQBP6lJF/64)

![](http://wx.qlogo.cn/mmopen/icWN8ZNEXrkNUEQLMFWwmJRVyjdqIfOmZ70q06ibm7FTJicyibEWeHKdJPujxiaXChxEDgU2ArwqzXcGRz4Dv1ymFeMNV5L8ibkZes/64)

![](http://wx.qlogo.cn/mmopen/sTW3ujMbibiaicJWWBYkwfZ6ibPwr9CeyGh48VJH7iaxO1T2JaWagaM5iaiaeCcRLSsDzzdbK7IB7HVw3BOiaxUJDN8OQO7QsnZaQTxHsXWZnm8F8azfnavktDCiawupQqTWSxSLD/64)

![](http://wx.qlogo.cn/mmopen/PiajxSqBRaEJRUWJomkEauK1Mg5PYoAyG9ddibkqvRamJAGZxhGickI7iaS49jOaoyEA98Uz7VOiadRMmTh74lqToYmRk0TEttuOUN2Rha0XC21hOxGacVzNzbxngZQvOuJY7/64)

![](http://wx.qlogo.cn/mmopen/bwq8IeLDpYYsugy6nI1XwxCZlQOeicXgc8Gn76JcOoAFMvZSnThlTEbuMmOnq4QQoS3HkDyqk2ofWib9AvJ4pxt5Y2W1mObHuH/64)

![](http://wx.qlogo.cn/mmopen/lW5Nk2bsL2kyNCZW8FWicXvExrRLibibZGPHUAbCiciaOK8zwut4rWQYTL7MLxBFY3mjVaSxH01mM5Oab27YKqR3OxHprMYnRRDxE/64)

![](http://wx.qlogo.cn/mmopen/ajNVdqHZLLCpVxB9hXoy7PecGqo2MJXSgbJic5WZ454okMqrlkgnqAjRgtYDwUvdnfb4WmjvHsULAGjqRkByJ3Q/64)

![](http://wx.qlogo.cn/mmopen/Q3auHgzwzM5nwOAzA3Ou7ucXY7bQOw83fosjFgnicV6PmdqLM2Bk7MIT18QWlruQoeEZE3FWibOzF9AyOZ5UYkDYtXfFlTtIdttj3Tfv6U8ibo/64)

​

写留言

**留言 6**

- DevYk

  2022年4月26日

  赞1

  话说有没 ppt 之类的文档勒

  程序喵大人

  作者2022年4月26日

  赞1

  那没有，后续会录制讲课视频

- hj

  2022年4月26日

  赞1

  支持下

  程序喵大人

  作者2022年4月26日

  赞

  谢谢

- 雨乐

  2022年4月26日

  赞1

  感谢喵大人 后面能否出一篇模板相关的文章，类似于为了模板而模板

  程序喵大人

  作者2022年4月26日

  赞

  首先请问下，为了模板而模板是？

已无更多数据

[](javacript:;)

![](http://mmbiz.qpic.cn/sz_mmbiz_png/Av3j9GXyMmGmg0HNK7fR6Peq3uf9pUoN5Qt0s0ia8IH97pvIiaADsEvy9nm3YnR5bmA13O8ic0JJOicQO0IboKGdZg/300?wx_fmt=png&wxfrom=18)

程序喵大人

2416

6

写留言

**留言 6**

- DevYk

  2022年4月26日

  赞1

  话说有没 ppt 之类的文档勒

  程序喵大人

  作者2022年4月26日

  赞1

  那没有，后续会录制讲课视频

- hj

  2022年4月26日

  赞1

  支持下

  程序喵大人

  作者2022年4月26日

  赞

  谢谢

- 雨乐

  2022年4月26日

  赞1

  感谢喵大人 后面能否出一篇模板相关的文章，类似于为了模板而模板

  程序喵大人

  作者2022年4月26日

  赞

  首先请问下，为了模板而模板是？

已无更多数据
