# 

Original 字节跳动STE团队 字节跳动SYS Tech

 _2023年09月08日 17:30_ _河南_

![](http://mmbiz.qpic.cn/mmbiz_png/Z4txjviceemNaH5Gc7V2ltc9AB79kQxpM7pOjtTiagt1TlQtCTrJ6zyPma8xbw3sGpeecTwveKP4GLqQa2nJ49EA/300?wx_fmt=png&wxfrom=19)

**字节跳动SYS Tech**

聚焦系统技术领域，分享前沿技术动态、技术创新与实践、行业技术热点分析。

42篇原创内容

公众号

  

**背景**

在C++开发中， 我们经常会面临一些有关全局对象的一些问题， 比如两个动态库之间全局对象初始化相互引用的问题， 即用外部的全局对象来初始化当前全局对象， 由于引用的外部全局对象未预先初始化，从而触发了coredump。那么如何在开发中避免由于全局对象初始化顺序而引发的各种程序崩溃呢？我们需要了解清楚全局变量是何时完成初始化， 初始化的顺序是什么， 以及如何控制全局对象的初始化顺序。本篇文章将重点介绍全局对象的初始化相关的内容，并进行深入分析，帮助大家更好地理解其原理。

  

![Image](https://mmbiz.qpic.cn/mmbiz_svg/hqDXUD6csU8f9Z5wkbLZ7NdTOqRuPmCFM1boKo7picnBCiaSDv1gG2PN7GiaOfwMpVvEaaDsy64sBvta5XSmHYlXfsIibSS0MNb0/640?wx_fmt=svg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

**测试环境**

- CPU 架构：x86_64  
    
- 操作系统及基础库版本：Debian GLIBC 2.28-10
    
- Compiled by GNU CC version 8.3.0.
    

  

**程序启动入口是 main 吗**

给一段简单的用例

```
struct C {
```

编译成可执行文件， 使用 gdb 在第 2 行打一个断点，运行：

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

可以看到， 在调用栈中并没有看到 main 的身影， 这是因为在 main 之前， 先进入到了全局对象 c 的构造函数中， 来完成初始化相关的工作。并且栈底指向了 _start， 这可能就是我们要找的入口函数。在确定 _start 是不是我们要找的入口函数之前， 需要先了解一下 Entry point 的概念：

  

**Entry point：  
**https://docs.oracle.com/cd/E19120-01/open.solaris/819-0690/chapter6-43405/index.html

> e_entry: The virtual address to which the system first transfers control, thus starting the process.

在 gdb 中继续执行：

```
info files
```

来观察可执行程序的各个 section 在内存中的分布：

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

这里的 Entry point， 就是程序运行起来的入口地址， 我们可以进一步调试它：

```
b *0x555555555040
```

可以看到， 该入口地址指向了 _start。  

  

**main 之前**

下图给出了 main 之前与初始化相关的关键流程：

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

给一段简单的示例 test.cpp：

```
#include
```

当我们将这段代码编译成可以执行文件时， 站在开发者的角度上来说， 只有 test.cpp 一个文件参与了这个过程。但实际上并非如此， 追加 -v 选项后可以看到详细的编译连接过程， 实际上还有一些 .o 文件参与了链接的过程， 这些 .o 文件除了标准库相关的文件， 还包含了程序启动相关的文件， 我们接下来要做的就是找到这些与程序启动相关的文件， 这有助于我们理解 main 之前发生的一些事情。

  

**-nostdlib:  
**https://gcc.gnu.org/onlinedocs/gcc/Link-Options.html

> Do not use the standard system startup files or libraries when linking.

在找到程序启动相关的文件之前， 我们准备通过 -nostdlib 来忽略这些系统相关的文件和库， 当然除了 C library， 因为我们用了 printf， 然后来尝试编译我们的程序：

```
g++ -g -nostdlib -lc test.cpp
```

可以看到， 找不到 _start 符号， 在运行时发生了 coredump。

  

_start 是程序启动的入口， 这是一个很好的起点。后续会详细介绍 _start 到 main 之间发生了什么， 同时找到程序启动相关的文件， 来让这段简单的程序编译通过并成功运行起来。

  

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

 **_start**

_start介绍 ：  
https://elixir.bootlin.com/glibc/glibc-2.28/source/sysdeps/x86_64/start.S#L58

> This is the canonical entry point, usually the first thing in the text segment.
> 
> Extract the arguments as encoded on the stack and set up the arguments for `main': argc, argv.  envp will be determined later in __libc_start_main.

  

简化一下 _start 的指令代码：

```
_start:
```

更多指令细节可以参考：http://6.s081.scripts.mit.edu/sp18/x86-64-architecture-guide.html

  

上述指令代码的解释：

• xorl %ebp, %ebp：寄存器清零。

• mov %RDX_LP, %R9_LP：rdx 存储了共享库终止函数的地址， 将该地址存入第 6 参数寄存器 r9。

• popq %rsi：将 argc 存入第 2 参数寄存器 rsi。

• mov %RSP_LP, %RDX_LP：将 argv 地址存入第 3 参数寄存器 rdx。

• pushq %rsp：压入堆栈地址。

• mov $__libc_csu_fini, %R8_LP：将 __libc_csu_fini 存入第 5 参数寄存器 r8。

• mov $__libc_csu_init, %RCX_LP：将 __libc_csu_init 存入第 4 参数寄存器 rcx。

• mov $main, %RDI_LP：将 main 存入第 1 参数寄存器 rdi。

• call *__libc_start_main@GOTPCREL(%rip)：调用 __libc_start_main

• hlt：保证退出。

也就是说， _start 所做的事情就是调用 __libc_start_main。

  

在程序启动相关的文件中， _start 是定义在 crt1.o 中的， 我们尝试链接它:

```
g++ -g -nostdlib /usr/lib/x86_64-linux-gnu/crt1.o -lc test.cpp
```

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

可以看到， 找不到 _init 而发生错误。

  

那 _init 是在哪里调用的， 看到上述告警信息， 在 __libc_csu_init 中引用了 _init， 而在前面介绍的 _start 内容中， __libc_csu_init 是作为参数传入 __libc_start_main 里的， 我们顺着这个调用链来看看里面发生了什么事。

  

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**__libc_start_main**

__libc_start_main介绍：  
https://elixir.bootlin.com/glibc/glibc-2.28/source/csu/libc-start.c#L129

  

简化一下 __libc_start_main 的代码：

```
STATIC int
```

在这段代码里主要关注这几件事：

• __environ = ev：存储环境变量信息。

• __cxa_atexit(...)：注册全局析构。

• (*init)(...)：调用全局构造。

• result = main(...)：调用 main。

• exit(result)：调用exit。

可以看到， 在调用 main 之前， 会调用全局构造 init， 而这里的 init 就是 __libc_csu_init。

  

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**__libc_csu_init**

__libc_csu_init介绍：  
https://elixir.bootlin.com/glibc/glibc-2.28/source/csu/elf-init.c#L67

  

简化一下 __libc_csu_init 的代码：

```
void
```

在这个函数中主要关注这 2 件事：

1. 调用 _init()， 它被定义在 .init 段。

2. 循环调用 __init_array_start 数组里面的内容， 它被定义在 .init_array 段。

  

 **.init**

_init 对应 ELF sections 中的 .init section。详情查看：  
https://elixir.bootlin.com/glibc/glibc-2.28/source/sysdeps/x86_64/crti.S#L63

```
_init:
```

这里的 PREINIT_FUNCTION 实际上是 __gmon_start__， 与 gprof 有关， 只有开启 -pg 选项才会生效， 否则为0x0:

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

在程序启动相关的文件中，_init 是定义在 crti.o 中的， 我们尝试链接它:

```
g++ -g -nostdlib /usr/lib/x86_64-linux-gnu/crt1.o /usr/lib/x86_64-linux-gnu/crti.o -lc test.cpp
```

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

可以看到， 测试文件可以正常编译链接， 但是在运行时发生了 coredump。

我们需要看下可执行文件的反汇编：

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

test   %rax,%rax， 即如果 rax 为 0 则 je     1012 <_init+0x12> 跳转到 1012， 这是一个一块非法内存地址。结合 _init 的定义和可执行程序的反汇编代码， 可以看出在结尾是没有 ret 返回指令的， 这部分实际上是被定义在了 crtn.S 中：

https://elixir.bootlin.com/glibc/glibc-2.28/source/sysdeps/x86_64/crtn.S#L39

  

简化下代码：

```
_init:
```

使用汇编代码在某个section中插入代码：.section .init,"ax",@progbits  

  

在程序启动相关的文件中，_init 结尾部分是定义在 crtn.o 中的， 我们尝试链接它:

```
g++ -g -nostdlib /usr/lib/x86_64-linux-gnu/crt1.o /usr/lib/x86_64-linux-gnu/crti.o /usr/lib/x86_64-linux-gnu/crtn.o -lc test.cpp
```

如上图结果显示：运行成功！也就是说， 要成功编译运行这段简单的程序， 至少需要 crt1.o、crti.o 和 crtn.o 这三个启动相关的文件。  

  

**.init_array**

> DT_INIT_ARRAY: This element holds the address of the array of pointers to initialization functions.

__init_array_start 对应 ELF sections 中的 .init_array section。  

  

在 __libc_csu_init 中我们可以看到， _init 并没有做有关全局对象初始化相关的工作， 实际在程序走到 main 之前， 是通过 __init_array 来实现程序本身的一些初始化的工作的：

```
  const size_t size = __init_array_end - __init_array_start;
```

__init_array_start 是一个函数指针数组， 里面存储了全局初始化相关的函数:

```
#include
```

```
g++ -g -nostdlib /usr/lib/x86_64-linux-gnu/crt1.o /usr/lib/x86_64-linux-gnu/crti.o /usr/lib/x86_64-linux-gnu/crtn.o -lc test.cpp
```

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

可以看到 .init_array 里面存储了2个地址：0x10e1 和 0x1137， 反汇编查看下这个地址：  

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

可以看到 0x10e1 就对应了我们自定义编写的 attribute constructor，0x1137 对应了全局对象 c 的初始化相关的函数。

  

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**.preinit_array**

> DT_PREINIT_ARRAY: This element holds the address of the array of pointers to pre-initialization functions. The DT_PREINIT_ARRAY table is processed only in an executable file; it is ignored if contained in a shared object.

.preinit_array 是在进程建立映射并重定位之后， 但在所有初始化函数之前执行的初始化函数列表。

  

也就是说， .preinit_array 中的初始化函数是更早被执行的：

https://elixir.bootlin.com/glibc/glibc-2.28/source/elf/dl-init.c#L78

  

可以通过源码发现， .preinit_array 的预初始化函数是在 _dl_init 中调用的， 而 _dl_init 是在 _start 之前被调用。

  

我们可以在 .preinit_array 中注册一个初始化函数：

```
#include
```

```
g++ -g -nostdlib /usr/lib/x86_64-linux-gnu/crt1.o /usr/lib/x86_64-linux-gnu/crti.o /usr/lib/x86_64-linux-gnu/crtn.o -lc test.cpp
```

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

可以看到 preinit 函数是最早被执行的， 我们可以观察下这个函数的存放位置：  

```
objdump -s -j .preinit_array a.out
```

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

可以看到多了一个 .preinit_array section， 里面存放了一个地址为 0x10f4， 反汇编查看下地址信息：

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

可以看到 0x10f4 对应了我们自定义编写的 preinit 函数。

  

**全局对象构造**

  

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**构造过程**

给一段简单的示例代码：  

```
#include
```

```
g++ -g -nostdlib /usr/lib/x86_64-linux-gnu/crt1.o /usr/lib/x86_64-linux-gnu/crti.o /usr/lib/x86_64-linux-gnu/crtn.o -lc test.cpp
```

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

观察调用栈， 发现在调用 C::C() 之前还调用了另外两个函数， 分别是 _GLOBAL__sub_I_c 和 __static_initialization_and_destruction_0。  

  

在之前介绍 .init_array 的时候， 我们其实就发现 .init_array 里存的函数地址并不是 C::C()， 而是 _GLOBAL__sub_I_c， 这是编译器以 _GLOBAL__sub_I_ 为前缀给每个全局对象生成的symbol， 专门用于全局对象初始化的事宜， 我们看下它的内容：

```
f 1
```

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

_GLOBAL__sub_I_c 中调用了 __static_initialization_and_destruction_0($0x1, $0xffff)， 我们再进一步观察下 __static_initialization_and_destruction_0 的指令信息：

```
f 0
```

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

__static_initialization_and_destruction_0 的定义位于 https://github.com/gcc-mirror/gcc/blob/releases/gcc-8/gcc/cp/decl2.c#L3576， 有些信息解释了该函数参数列表的含义：

> __initialize_p：If __INITIALIZE_P is nonzero, it performs initializations. Otherwise, it performs destructions.
> 
> __priority：It only performs those initializations or destructions with the indicated __PRIORITY.

  

在 __static_initialization_and_destruction_0 中来调用 c 的构造函数 C::C()， 完成初始化。

  

**为什么不在 .init_array 里直接放构造函数的地址：**

构造函数是与对象相关联的， 在调用构造函数之前需要为当前对象开辟内存， 需要将内存地址传入构造函数。在 .init_array 中只放构造函数的地址很明显是无法确定构造的是哪个对象的， 因此编译器在外部进行了封装。

  

在上述的示例中， 全局对象初始化的过程用链：__libc_csu_init -> _GLOBAL__sub_I_c -> __static_initialization_and_destruction_0 -> C::C()。

  

  

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**init_priority**

  

init_priority介绍：  
https://gcc.gnu.org/onlinedocs/gcc/C_002b_002b-Attributes.html#index-init_005fpriority-variable-attribute

_In Standard C++, objects defined at namespace scope are guaranteed to be initialized in an order in strict accordance with that of their definitions in a given translation unit. No guarantee is made for initializations across translation units. However, GNU C++ allows users to control the order of initialization of objects defined at namespace scope with the init_priority attribute by specifying a relative priority, a constant integral expression currently bounded between 101 and 65535 inclusive. Lower numbers indicate a higher priority._

在 C++ 标准中， 编译单元内的对象是按照定义的顺序进行初始化的，通过 init_priority 可以来控制构造函数被调用的相对顺序。

看一段简单的示例：https://godbolt.org/z/6ozEGWKPq

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

通过 init_priority 来使后定义的对象先初始化：

https://godbolt.org/z/s5G1G3eEd

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

查看下该示例的 section 与 反汇编代码：

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

init_priority 决定了 初始化函数在 .init_array 里面的相对顺序， 200 的排在 201 的前面， 看下 __static_initialization_and_destruction_0 的汇编：

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

__priority 在 __static_initialization_and_destruction_0 仅表示一个数字， 这个数字代表了要调用的初始化函数， 它并不会实际控制初始化函数的调用顺序， 实际控制调用顺序的是 init_priority。

  

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**思考**

  

**Q:** Does __attribute__((constructor)) execute before C++ static initialization?

**A：**https://stackoverflow.com/questions/8433484/c-static-initialization-vs-attribute-constructor

  

**全局对象初始化顺序**

全局对象初始化顺序介绍：  
https://downloads.ti.com/docs/esd/SPRUI04/global-object-constructors-stdz0560845.html

> Constructors for global objects from a single module are invoked in the order declared in the source code, but the relative order of objects from different object files is unspecified.

在同一编译单元内， 可以通过 init_priority 来控制全局对象的相对初始化顺序；跨编译单元之间全局对象初始化的顺序是未定义的。  

  

**1. 编译单元内**

在同一编译单元内， 我们可以使用 init_priority 很好控制全局初始化函数的执行顺序。代码：https://godbolt.org/z/7ox99qv1n

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**2. 跨编译单元**

看一段示例：

  

test.cpp:

```
#include "third.h"
```

thrid.h:  

```
#include
```

third.cpp:

```
#include "third.h"
```

  

**3. 静态链接**

将 third.cpp 编译成静态库并链接到主程序执行：

```
g++ -g -c third.cpp
```

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

可以看到程序发生了core， 使用 gdb 进行调试：

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

发现在在走到 G_c 构造函数的时候， 依赖的 G_tc 对象还没初始化， 在 G_tc.dosomething(); 的时候由于访问到非法内存而引发程序崩溃。

  

我们知道全局对象的构造函数是放在 .init_array 里面的， 静态链接的时候， 已经将两个编译单元内全局对象的构造函数相对地址按照某种顺序放在了 .init_array 里面， 查看可执行程序的 section 信息：

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

那么他们的执行顺序为：先 0x1188， 后 0x1209， 查看可执行程序的反汇编代码， 查看这两个地址对应的内容：

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

也就是说， 先构造 G_c， 再构造 G_tc。

  

**思考**：

https://downloads.ti.com/docs/esd/SPRUI04/global-constructors-stdz0545810.html#STDZ0545810

> The linker combines the .init_array section form each input file to form a single table in the .init_array section.

静态链接， 在链接阶段会把所有编译单元的.init_array按照某种顺序进行合并， 那么这个顺序是随机的吗？  

https://codebrowser.dev/llvm/lld/ELF/OutputSections.cpp.html#703

> Sorts input sections by section name suffixes, so that .foo.N comes before .foo.M if N < M.

  

答案是否定的， 链接器会根据 .init_array 中存放的初始化函数的后缀来进行排序， 注释中的 N 和 M 是 init_priority 指定的数字。

  

**4. 动态链接**

将 third.cpp 编译成动态库并链接到主程序执行：

Shell  

```
g++ -g -shared -fPIC third.cpp -o libthird.so
```

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

程序正确运行， 在 G_c 初始化之前， G_tc 已经完成了初始化。  

  

那么在动态链接的场景里， 这段程序一定是 100% 运行的吗？

  

答案是一定的， 这是因为对于动态链接的程序来说， loader 会预先加载动态库， 完成动态库中全局对象的初始化， 然后再执行主程序中全局对象的初始化：https://www.gnu.org/software/hurd/glibc/startup.html

  

可以看到在进入 _start 之前， 会先调用 _dl_init， 完成动态库的初始化工作：

https://elixir.bootlin.com/glibc/glibc-2.28/source/elf/dl-init.c#L78

  

当然， 这只是动态库相对于主程序而言。在复杂的项目中， 很可能会出现两个动态库的全局对象之间有引用的情况， 在这种情况下， 我们依然是无法保证全局对象的初始化的顺序， 这依赖于 linker 和 loader 来决定依赖的多个动态库谁的初始化工作先进行。

  

**5. 如何避免**

可以看出， 控制跨编译单元的全局对象的初始化顺序是一个棘手的问题，在 C++ 中这是一个未定义的行为， 这受链接器和加载器的影响。最安全的方案就是尽量避免使用全局变量或跨编译单元之间全局对象的相互引用。

https://en.cppreference.com/w/cpp/language/siof

> The Construct on First Use Idiom can be used to avoid the static initialization order fiasco and ensure that all objects are initialized in the correct order.

在初始化全局对象时， 使用 Construct on First Use 来初始化， 即在首次使用时进行初始化， 可以认为是延迟初始化的一种设计方式：

test.cpp:‍

```
#include "third.h"
```

thrid.h:

```
#include
```

third.cpp:

```
#include "third.h"
```

  

**std::cout 如何工作**

https://en.cppreference.com/w/cpp/io/cout

> These objects are guaranteed to be initialized during or before the first time an object of type std::ios_base::Init is constructed and are available for use in the constructors and destructors of static objects with ordered initialization (as long as  is included before the object is defined).

  

std::cout 是一个全局对象， 并且在 C++11 中， 它可以保证在其他全局对象构造之前被正确初始化， 它是通过 Schwarz Counter 这项设计来实现的， 并且纳入了 C++标准。

  

在 iostream 头文件中， 定义了 static ios_base::Init __ioinit; 私有的全局对象， 当包含 iostream 头文件时， 根据定义的顺序， 会预先执行 __ioinit 的初始化， 在 Init::Init() 里面会完成 cout 的初始化工作：

• 在此之前会定义一块全局的char类型的缓冲区， 该缓冲区的大小要能装得下流对象的大小， 并定义全局流对象指向这块缓冲区。

• 在 Init::Init() 中使用 placement new 在缓冲区中构造流对象。

  

以 Schwarz Counter 来改写之前的示例：

test.cpp:

```
C++
```

thrid.h:  

```
#include
```

third.cpp:

```
#include "third.h"
```

  

注：https://stackoverflow.com/questions/9251763/c-static-initialization-via-schwartz-counter

在工程代码中并不建议使用 Schwarz Counter 来设计你的代码：

• 确保在当前编译单元中引入了初始化相关的头文件。

• 保证全局变量在其他各个编译单元的全局变量前完成初始化而带来的启动延迟。

  

**参考链接**

- _https://www.gnu.org/software/hurd/glibc/startup.html_
    
- _https://refspecs.linuxbase.org/elf/gabi4+/ch5.dynamic.html_
    
- _https://refspecs.linuxfoundation.org/elf/gabi4+/ch4.sheader.html#special_sections_
    
- _https://stackoverflow.com/questions/42912038/what-is-the-difference-between-cxa-atexit-and-atexit_
    
- _https://gcc.gnu.org/onlinedocs/gcc-11.2.0/gcc/C_002b_002b-Attributes.html#C_002b_002b-Attributes_
    

  

**热门招聘**

字节跳动STE团队诚邀您的加入！团队长期招聘，**北京、上海、深圳、杭州、US、UK** 均设岗位，以下为近期的招聘职位信息，有意向者可直接扫描海报二维码投递简历，期待与你早日相遇，在字节共赴星辰大海！若有问题可咨询小助手微信：sys_tech，岗位多多，快来砸简历吧！

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**往期精彩**

# [BMC部件管理：AF_MCTP应用落地实践](http://mp.weixin.qq.com/s?__biz=Mzg3Mjg2NjU4NA==&mid=2247484591&idx=1&sn=996effa848ced022aa2245051cef9fac&chksm=cee9f2d8f99e7bce10cc6a9f38333ae751e5a64a1d02a4a97adee7baeb09b68457f4706e0fd1&scene=21#wechat_redirect)

[ByteFUSE分布式文件系统的演进与落地](http://mp.weixin.qq.com/s?__biz=Mzg3Mjg2NjU4NA==&mid=2247484577&idx=1&sn=acf23f871cb1805b8fbdecfd2ddb7ab9&chksm=cee9f2d6f99e7bc08a0bd4051ee28946343c1e747351b05eb852a65e54e1bbc7501374ce60be&scene=21#wechat_redirect)

# [深入分析HWASAN检测内存错误原理](http://mp.weixin.qq.com/s?__biz=Mzg3Mjg2NjU4NA==&mid=2247484531&idx=1&sn=57acfda58d6cc0dffb9fa5e5c6f488bb&chksm=cee9f204f99e7b12dbd8a84aea2c1299fb28297aea4ecc76af0dfcecbf4294e07d1c8c756627&scene=21#wechat_redirect)

# [深度解析C/C++之Strict Aliasing Rule](http://mp.weixin.qq.com/s?__biz=Mzg3Mjg2NjU4NA==&mid=2247484517&idx=1&sn=049fb8f6743e4caf6f9138688cd841a1&chksm=cee9f212f99e7b043523bfe80d8cb6ea1a86a5c6d2a2a14a0aa5944fbb1e116190d594ad8156&scene=21#wechat_redirect)

[深入浅出 Sanitizer Interceptor 机制](http://mp.weixin.qq.com/s?__biz=Mzg3Mjg2NjU4NA==&mid=2247483868&idx=1&sn=a85112e88cd187e27f418ff044247956&chksm=cee9f7abf99e7ebdac55e36077b4f915bccbe33a8a6ee9920136a1dc9598e2ae62bb95d3a25f&scene=21#wechat_redirect)

[Jemalloc内存分配与优化实践](http://mp.weixin.qq.com/s?__biz=Mzg3Mjg2NjU4NA==&mid=2247484507&idx=1&sn=0befa2bec46c3fe64807f5b79b2940d5&chksm=cee9f22cf99e7b3a6235b06cb0bd87c3f8e9b10c3ba989b253a97a5dc92b261cfde0b39db230&scene=21#wechat_redirect)

[DPDK内存碎片优化，性能最高提升30+倍!](http://mp.weixin.qq.com/s?__biz=Mzg3Mjg2NjU4NA==&mid=2247484462&idx=1&sn=406c59905ea57718018c602cba66f40e&chksm=cee9f259f99e7b4f7597eb6754367214a28f120c062c72c827f6e8c4225f9e54e4c65d4395d8&scene=21#wechat_redirect)

[万字长文，GPU Render Engine 详细介绍](http://mp.weixin.qq.com/s?__biz=Mzg3Mjg2NjU4NA==&mid=2247484413&idx=1&sn=d8f5f4f195b931d775b7d5934f0e3cb4&chksm=cee9f58af99e7c9c07080f22165120fcf3d321a14b943077c23649f88b27da29529db44de933&scene=21#wechat_redirect)

[DPDK Graph Pipeline框架简介与实现原理](http://mp.weixin.qq.com/s?__biz=Mzg3Mjg2NjU4NA==&mid=2247483974&idx=1&sn=25a198745ee4bec9755fe1f64f500dd8&chksm=cee9f431f99e7d27527c7a74c0fd2969c00cf904cd5a25d09dd9e2ba1adb1e2f1a3ced267940&scene=21#wechat_redirect)

[字节跳动算力监控系统的落地与实践](http://mp.weixin.qq.com/s?__biz=Mzg3Mjg2NjU4NA==&mid=2247484264&idx=1&sn=c27c08c225fcceb65a5ffc071dba3f95&chksm=cee9f51ff99e7c09a0c103bc4b362fcdcba7d8e6de74ab251afe1beb5b7af0412e619dd53f13&scene=21#wechat_redirect)

[JVM 内存使用大起底](http://mp.weixin.qq.com/s?__biz=Mzg3Mjg2NjU4NA==&mid=2247484211&idx=1&sn=c65b23b2ab50f6d56d81cfe434c8ea53&chksm=cee9f544f99e7c52d7767db2a74fda23ed7a6c06f8c7a82c1219a7e1474839bf5301f78ff38e&scene=21#wechat_redirect)

[字节跳动在PGO反馈优化技术上的探索与实践](http://mp.weixin.qq.com/s?__biz=Mzg3Mjg2NjU4NA==&mid=2247483904&idx=1&sn=335efeed20a6b42d5ee6e6543ae12018&chksm=cee9f477f99e7d6141de97d85c125979daaae14d98e73225792fa8e41cf0d6269e97d5c7a397&scene=21#wechat_redirect)

[内核优化之 PSI 篇:快速发现性能瓶颈，提高资源利用率](http://mp.weixin.qq.com/s?__biz=Mzg3Mjg2NjU4NA==&mid=2247483887&idx=1&sn=32ec7c9200a72f13ef2c1b8756ea2b8f&chksm=cee9f798f99e7e8e6e44a4f7464ee18471cd4e12def97fdf207f58d37dd317e3fafa6614e04f&scene=21#wechat_redirect)

[探索Mac mini M1的QEMU/KVM虚拟化实现](http://mp.weixin.qq.com/s?__biz=Mzg3Mjg2NjU4NA==&mid=2247483881&idx=1&sn=282483df28129da2d193239743397260&chksm=cee9f79ef99e7e884ca202597076df9915145a6c36670ae3bab2419a0da13deed072ee59e058&scene=21#wechat_redirect)

[字节跳动提出KVM内核热升级方案，效率提升5.25倍！](http://mp.weixin.qq.com/s?__biz=Mzg3Mjg2NjU4NA==&mid=2247483751&idx=1&sn=6f818349a0d6579902d9a7ca1a512cca&chksm=cee9f710f99e7e06957baac9ce25433ed8a463a417479f243136cef254073f10e018d941be4b&scene=21#wechat_redirect)

[安全容器文件服务 Virtio-fs 的高可用实践](http://mp.weixin.qq.com/s?__biz=Mzg3Mjg2NjU4NA==&mid=2247483815&idx=1&sn=92edbb33cbbbf22c80ccdf6537a2ca37&chksm=cee9f7d0f99e7ec671106416c7d5cee58497af8f427a89f5e9a2ce051277944116db069d10e3&scene=21#wechat_redirect)

[字节跳动开源高性能C++ JSON库sonic-cpp](http://mp.weixin.qq.com/s?__biz=Mzg3Mjg2NjU4NA==&mid=2247483853&idx=1&sn=b88e45bf75cd57482cf7860e669416a8&chksm=cee9f7baf99e7eacb5de6f9118a590db4bb659420bc2f4cb6c6964225409f559715ea8145186&scene=21#wechat_redirect)

[eBPF 技术实践：加速容器网络转发，耗时降低60%+](http://mp.weixin.qq.com/s?__biz=Mzg3Mjg2NjU4NA==&mid=2247483701&idx=1&sn=c99b7df5b8457f6d4f87fa18fea82eba&chksm=cee9f742f99e7e54f317d4f9eb355daf8cdceb4d3894e7601f5fc124e8b3b8567ee5f2efc46a&scene=21#wechat_redirect)

[十年磨一剑，veLinux深度解读](http://mp.weixin.qq.com/s?__biz=Mzg3Mjg2NjU4NA==&mid=2247483680&idx=1&sn=d767d17e8b28baacb25f759c02e7bd0c&chksm=cee9f757f99e7e4136580232d4c56d619feacdd939e84ed9780d010b73417c5739b223a5c4dd&scene=21#wechat_redirect)

  

**关于STE团队**

**字节跳动STE团队**（System Technologies&Engineering，系统技术与工程），一直致力于操作系统内核与虚拟化、系统基础软件与基础库的构建和性能优化、超大规模数据中心的系统稳定性和可靠性建设、新硬件与软件的协同设计等基础技术领域的研发与工程化落地，具备全面的基础软件工程能力，为字节上层业务保驾护航。同时，团队积极关注社区技术动向，拥抱开源和标准，欢迎更多同学加入我们，一起交流学习。扫描下方二维码了解职位详情，欢迎大家投递简历至huangxuechun.hr@bytedance.com 、wangan.hr@bytedance.com。

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

扫码查看职位详情

c++5

c++ · 目录

上一篇深度解析C/C++之Strict Aliasing Rule下一篇Jemalloc内存分配与优化实践

Reads 1792

​

Send Message

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/Z4txjviceemNaH5Gc7V2ltc9AB79kQxpM7pOjtTiagt1TlQtCTrJ6zyPma8xbw3sGpeecTwveKP4GLqQa2nJ49EA/300?wx_fmt=png&wxfrom=18)

字节跳动SYS Tech

22144

Send Message