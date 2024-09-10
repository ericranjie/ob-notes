# 

原创 何浩乾 冯力 SmartX技术团队 _2022年03月29日 17:57_

  

Coroutine（协程）一直都是比较热门的基础概念，现代化程序语言从语言或者标准库层面做到了对协程支持，例如 Golang，以及 C++ 20。本文介绍如何使用 GDB 来分析自实现的协程。在 C/C++ 中，可以使用最基本的几个 C API 来实现协程，本文只简单介绍协程的实现，关于协程的更多实现细节和内容，网络上有很多文章，我们不去详细分析，我们的重点是介绍如何使用 GDB，像调试 POSIX thread 一样，调试协程。

# 协程背景

协程不是系统级线程，内核不会有一个 task_struct 相关联，协程的创建以及切换调度不需要内核调度器的参与，全部都是用户自定义。而协程可以分为有栈协程和无栈协程，“有栈”和“无栈”的含义不是指协程在运行时是否需要栈，而是指协程是否可以在其任意嵌套函数（子函数、匿名函数）中被挂起，显然有栈协程是可以的，而无栈协程则不可以。本文讨论内容的对象是有栈协程。

协程创建以及调度相关的 C API 有：

- setjmp
    
- longjmp
    
- makecontext
    
- swapcontext
    
- getcontext
    
- setcontext
    

GDB 原生并不支持对协程的调试，要使用 GDB 来追踪到协程的上下文，我们需要知道任意协程的上下文信息，包括寄存器信息，而这其实就是协程如何保存、恢复和切换上下文的原理。我们知道函数是运行在调用栈上，如果将一个函数作为协程，很自然地联想到保存协程上下文就是保存当前调用环境（通常是堆栈指针、程序计数器、寄存器的值和 signal mask）；恢复上下文是将保存的调用环境重新写入；而切换上下文是保存当前正在运行的协程函数的上下文，恢复另一个要运行的协程函数的上下文，所以我们先来分析协程的切换细节。

## 协程切换

前文提到协程的切换本质上是保存当前函数运行栈，装载新的函数栈，然后开始就可以继续执行新的协程。可以使用 setjmp/longjmp 来实现，也可以使用 swapcontext 来实现，先来看一下基于 setjmp/longjmp 的协程切换，基于 ucontext 的更简单一些，本文不再分析。

### setjmp/longjmp

setjmp 和 longjmp 实现保存上下文和非局部跳转，"非局部跳转" 指不像 goto 让控制流在当前函数桢内跳转，而是在栈上跳过若干个函数帧，返回到当前函数调用路径上的某一个函数中。这两个函数是 C 的标准库函数，在 glibc 中找到这个函数的原型：

```
int setjmp(jmp_buf env);void longjmp(jmp_buf env, int val);
```

setjmp 函数将有关调用环境的各种信息（通常是堆栈指针、指令指针、关键寄存器的值和信号掩码）保存在缓冲区 env 中，以供 longjmp 以后使用。在这种情况下，setjmp 返回 0。longjmp 函数使用保存在 env 中的信息将控制权转移回上一次调用 setjmp 的位置，并将堆栈恢复（“倒回”）到上一次调用 setjmp 时的状态。longjmp 成功跳转到 setjmp 就像 setjmp 第二次返回（第一次是保存上下文信息时返回了 0）一样，并且第二次将返回 longjmp 的第二个参数 val，如果程序员错误地将值 0 传递给 val，则第二次将改为返回 1，这样便可以根据 setjmp 的返回值来继续执行当前函数，协程切换可以这样实现：

```
if (setjmp(prev->jmpbuf) == 0) {    longjmp(jmpbuf, 1);}
```

setjmp 是保存寄存器而 longjmp 是恢复寄存器，所以我们分析 setjmp 就可以。

setjmp 在 glibc 的实现如下：

```
ENTRY (__sigsetjmp) /* Save registers.  */ movq %rbx, (JB_RBX*8)(%rdi)#ifdef PTR_MANGLE                                         ⇐======================== 已经定义了此宏# ifdef __ILP32__ /* Save the high bits of %rbp first, since PTR_MANGLE will    only handle the low bits but we cannot presume %rbp is    being used as a pointer and truncate it.  Here we write all    of %rbp, but the low bits will be overwritten below.  */ movq %rbp, (JB_RBP*8)(%rdi)# endif mov %RBP_LP, %RAX_LP                                ⇐======================== 进入这里 PTR_MANGLE (%RAX_LP) mov %RAX_LP, (JB_RBP*8)(%rdi)#else movq %rbp, (JB_RBP*8)(%rdi)#endif movq %r12, (JB_R12*8)(%rdi) movq %r13, (JB_R13*8)(%rdi) movq %r14, (JB_R14*8)(%rdi) movq %r15, (JB_R15*8)(%rdi) lea 8(%rsp), %RDX_LP /* Save SP as it will be after we return.  */#ifdef PTR_MANGLE PTR_MANGLE (%RDX_LP)#endif movq %rdx, (JB_RSP*8)(%rdi) mov (%rsp), %RAX_LP /* Save PC we are returning to now.  */ LIBC_PROBE (setjmp, 3, LP_SIZE@%RDI_LP, -4@%esi, LP_SIZE@%RAX_LP)#ifdef PTR_MANGLE PTR_MANGLE (%RAX_LP)#endif movq %rax, (JB_PC*8)(%rdi)
```

我们重点关注 %rbp, %rbx, %rsp, %rip, %r12, %r13, %r14, %r15, %rip 这 8 个寄存器的保存。

对于 %rbx、%r12、%r13、%r14、%r15,  glibc 没有对他们做多余的操作，就是将他们保存在了 %rdi + offset 的位置。

```
movq %rbx, (JB_RBX*8)(%rdi)movq %r12, (JB_R12*8)(%rdi)movq %r13, (JB_R13*8)(%rdi)movq %r14, (JB_R14*8)(%rdi)movq %r15, (JB_R15*8)(%rdi)
```

但是对 %rbp、%rsp、%rip 这 3 个寄存器，glibc 做了一些额外的操作，这里重点就是 `PTR_MANGLE` ，这是 glibc 的一个安全增强操作 pointer guard。

```
mov %RBP_LP, %RAX_LPPTR_MANGLE (%RAX_LP)mov %RAX_LP, (JB_RBP*8)(%rdi)
```

上面的操作即为将 %rbp 寄存器值加密后存放到 (JB_RBP*8)(%rdi)。下面我们来探究 glibc 中的 pointer guard。

## Pointer Guard

这个功能是 glibc 为了安全，增加攻击者在 glibc 中操纵指针（尤其是函数指针）的难度的做法。此功能也被称为 "pointer mangling" 或 "pointer guard"。

实现方法就是上文提到的 `PTR_MANGLE` 这个宏，可以理解成 “加密”。与之对应的“解密” 的宏是 `PTR_DEMANGLE` ，longjmp 用到了这个宏。

如果应用程序想要加密在 *stored_ptr 中存储的函数指针，可以这样做：`*stored_ptr = PTR_MANGLE(ptr)；`对应的，解密就是 `ptr = PTR_DEMANGLE(*stored_ptr);`

来具体看看这两个宏在做什么：

```
#  define PTR_MANGLE(reg) xor %fs:POINTER_GUARD, reg;        \    rol $2*LP_SIZE+1, reg#  define PTR_DEMANGLE(reg) ror $2*LP_SIZE+1, reg;         \    xor %fs:POINTER_GUARD, reg/* Long and pointer size in bytes.  */#define LP_SIZE 8POINTER_GUARD  offsetof (tcbhead_t, pointer_guard)
```

`PTR_MANGLE` 就是对 reg 寄存器和 `%fs:POINTER_GUARD` 进行异或，然后按位左旋转 (bitwise rotate) reg 寄存器  `2 * LP_SIZE + 1` 位（在64位的机器上， LP_SIZE 为 8）；`PTR_MANGLE` 进行了相反的操作。

`POINTER_GUARD`  也是一个宏，计算了 pointer_guard 在结构体 tcbhead_t 中的偏移量，拓展 `PTR_MANGLE` 这个宏为两条汇编指令：

```
xor %fs:offsetof(tcbhead_t, pointer_guard), reg;rol $2*LP_SIZE+1, reg
```

这两个宏利用 pointer_guard 分别对指针进行了加密和解密操作，加密由异或以及 bitwise rotate，而加密使用的 key 来自 `%fs:offsetof(tcbhead_t, pointer_guard)`。由此可以得出， %fs 寄存器保存了 tcbhead_t 这个结构体的基地址。

查看 glibc 源码找到 tcbhead_t 的定义，根据代码，在 X86-64 下 pointer_guard 在 tcbhead_t 中的偏移就是 0x30：

```
typedef struct{  void *tcb;  /* Pointer to the TCB.  Not necessarily the thread descriptor used by libpthread.  */  dtv_t *dtv;  void *self;  /* Pointer to the thread descriptor.  */  int multiple_threads;  int gscope_flag;  uintptr_t sysinfo;  uintptr_t stack_guard;  uintptr_t pointer_guard; ...} tcbhead_t;
```

FS 段寄存器在如今的内存平坦化下已经不像它的名字所表达的意思一样，如今操作系统可以自由的使用他们。在 Linux 下，FS 寄存器用于存放线程控制块（TCB）的地址，一般由线程库管理。在 x86 架构下，由于支持分段，访问内存的指令可以使用基于段寄存器的寻址模式：Segment-register:Byte-address， `PTR_MANGLE` 这个宏中的`%fs:offsetof(tcbhead_t, pointer_guard)`就是使用这种寻址模式。通过 Segment base address + Byte-address 计算出了所要访问虚拟地址，这允许实现 TLS 在不同线程访问同一个变量得到不同的值。

通过上面的分析，最终可以得出解密的操作如下：

```
def glibc_ptr_demangle(val, pointer_guard):    return gdb.parse_and_eval('(((uint64_t)%s >> 0x11) | ((uint64_t)%s << (64 - 0x11))) ^ (uint64_t)%s'                              % (val, val, pointer_guard))
```

## TLS 与 FS 寄存器

Thread local storage(TLS) 是线程独占的空间，可以使用 __thread 关键字告诉编译器某一个变量应当被放入线程局部存储的空间中。TLS 只能被当前线程访问和修改，使得其不能像一般的变量一样被简单的存储到 ELF 文件的某个段，而且 TLS 变量也与一般的变量不同。来看一个例子，这个例子定义了 3 个全局的 TLS 变量，我们分别在 main thread 和 test  thread 打印他们的值和地址。

```
#include <pthread.h>#include <stdio.h>__thread int a = 123; // 0x7b__thread unsigned long long b = 234; // 0xea__thread int c;void test(void *func) {    printf("in thread test: a = %d, b = %llu, c = %p\n", a, b, &c);    printf("in thread test: &a = %p, b = %p\n", &a, &b);}int main() {    a = 345; // 0x159    b = 456; // 0x1c8    c = 567; // 0x237    printf("in main thread: a = %d, b = %llu, c = %p\n", a, b, &c);    printf("in main thread: &a = %p, b = %p\n", &a, &b);    pthread_t pid;    pthread_create(&pid, NULL, test, NULL);    pthread_join(pid, NULL);    return 0;}
```

程序运行的结果如下：

```
 in main thread: a = 345, b = 456, c = 0x7f5e4bf04738 in main thread: &a = 0x7f5e4bf04728, b = 0x7f5e4bf04730 in thread test: a = 123, b = 234, c = 0x7f5e4b7096f8 in thread test: &a = 0x7f5e4b7096e8, b = 0x7f5e4b7096f0
```

可以看出，main thread 和 test thread 打印的全局变量的值和地址完全不同，也就是说对于每一个全局变量，每一个线程都有一个自己的空间。我们在 GDB 深入查看一下，在 main 函数中的 printf 语句打断点，查看对应的汇编：

```
   │0x400679 <main>         push   %rbp   │0x40067a <main+1>       mov    %rsp,%rbp   │0x40067d <main+4>       sub    $0x10,%rsp   │0x400681 <main+8>       movl   $0x159,%fs:0xffffffffffffffe8   │0x40068d <main+20>      movq   $0x1c8,%fs:0xfffffffffffffff0   │0x40069a <main+33>      movl   $0x237,%fs:0xfffffffffffffff8B+>│0x4006a6 <main+45>      mov    %fs:0xfffffffffffffff0,%rdx   │0x4006af <main+54>      mov    %fs:0xffffffffffffffe8,%eax   │0x4006b7 <main+62>      mov    %fs:0x0,%rcx
```

在汇编代码中发现在访问全局变量时又出现了 %fs 寄存器，查看这个寄存器的值 。由于权限问题，在用户态不能直接访问 %fs 寄存器，但内核为我们提供了系统调用 arch_prctl 可以获得 FS Segment base address。arch_prctl 的函数原型是：

```
int arch_prctl(int code, unsigned long addr);int arch_prctl(int code, unsigned long *addr);
```

arch_prctl 设置特定于体系结构的进程或线程状态。code 选择一个子函数并将参数 addr 传递给它；对于“set”操作，addr 被解释为 unsigned long，对于“get”操作，addr 被解释为 unsigned long*。arch_prctl 只适用于 X86/X86-64 平台。

code 有如下几种：

- ARCH_SET_CPUID
    
- ARCH_GET_CPUID
    
- ARCH_SET_FS
    
- ARCH_GET_FS
    
- ARCH_SET_GS
    
- ARCH_GET_GS
    

下面的 code 为 0x1003，对应 ARCH_GET_FS，用来获取 FS 寄存器的值，存放到 %rsp-0x8 的地址上。

```
(gdb) call arch_prctl(0x1003, $rsp - 0x8)$2 = 0(gdb) x/gx $rsp - 0x80x7fffffffe078: 0x00007ffff7fe9740
```

进入 test thread，进行和在 main thread 类似的操作，并得到了类似的结果：

```
// asmB+>│0x400613 <test+12>              mov    %fs:0xfffffffffffffff0,%rdx   │0x40061c <test+21>              mov    %fs:0xffffffffffffffe8,%eax(gdb) call arch_prctl(0x1003, $rsp - 0x8)$3 = 0(gdb) x /gx $rsp - 0x80x7ffff77efef8: 0x00007ffff77f0700
```

main thread 的 FS 寄存器值为 0x00007ffff7fe9740，test thread 的 FS 寄存器值为 0x00007ffff77f0700。上述分析证明了每一个线程都会有 TLS 变量的副本，并且可以通过 %fs 寄存器来间接寻址，每个线程都有自己的 FS 段基址。不过虽然每个线程有自己的独立的线程控制块，但是他们的 pointer guard 值是相同的，这样就可以方便的跨线程加解密 %rbp, %rsp, %rip 寄存器。

# 如何回溯协程栈

回到 setjmp 函数中，综合之前的分析，glibc 在保存 %rbp、%rsp、%rip 的值时使用了 tcbhead_t.pointer_guard 作为 key 对原值进行加密，那么在恢复协程上下文时就要对这三个寄存器进行解密，即实现 `PTR_DEMANGLE` 这个宏。

setjmp 和 longjmp 这两个函数的核心是 jmp_buf 类型的 env ，所以我们来看看jmp_buf。

```
// from <setjmp.h>/* Calling environment, plus possibly a saved signal mask.  */struct __jmp_buf_tag  {    /* NOTE: The machine-dependent definitions of `__sigsetjmp'       assume that a `jmp_buf' begins with a `__jmp_buf' and that       `__mask_was_saved' follows it.  Do not move these members       or add others before it.  */    __jmp_buf __jmpbuf;  /* Calling environment.  */    int __mask_was_saved; /* Saved the signal mask?  */    __sigset_t __saved_mask; /* Saved signal mask.  */  };typedef struct __jmp_buf_tag jmp_buf[1];
```

每个成员注释中解释的比较清楚，我们重点关注 __jmpbuf。

```
# if __WORDSIZE == 64typedef long int __jmp_buf[8];# elif defined  __x86_64__typedef long long int __jmp_buf[8];# elsetypedef int __jmp_buf[6];# endif
```

__jmpbuf 是一个数组，保存了 8 个 64 位的寄存器：%rbp, %rbx, %rsp, %r12, %r13, %r14, %r15, %rip。

关于 X86-64 的寄存器使用约定，我们来回忆一下调用者保存寄存器和被调用者保存寄存器：

调用者保存寄存器：Caller-saved registers(AKA volatile registers)。因为易失性，调用者有责任将这些寄存器压入堆栈或将它们复制到其他地方，如果它想在过程调用后恢复该值。

被调用者保存寄存器：Callee-saved registers( AKA non-volatile registers)。当调用者进行过程调用时，这些寄存器在被调用者返回后将保持相同的值，这使得被调用者有责任在返回给调用者之前保存和恢复它们或者不碰它们。

X86-64 规定 %rbx, %rsp, %rbp, %r12 ~ %r15 这些寄存器是被调用者保存寄存器，他们需要由被调用的函数保存。

__jmpbuf 保存的 8 个寄存器包括了被调用者寄存器和 %rip，要正确回溯协程栈，恢复他们就可以。

此外因为 glibc 的 pointer guard，我们在处理 %rbp、%rsp、%rip 这三个寄存器时需要将他们解密还原他们的初始值，解密的方法在上文已经给出。

最终，我们添加了如下 GDB 扩展命令来支持对协程的调试：

```
  - co-bt <coroutine-pointer>      : Show given coroutine's backtrace by a coroutine pointer  - co-bt-full <coroutine-pointer> : Show given coroutine's full backtrace by a coroutine pointer  - co-enter <coroutine-pointer>   : Enter the given coroutine's context  - co-exit                        : If enter coroutine A previously, then exit A's context  - co-show                        : Show current thread's coroutine's backtrace  - co-list                        : List all coroutines in like 'info thread'  - co-threads-list                : Classify all coroutines according tid  - co-thread-list <PID>           : List given thread's coroutines' by thread-id  - co-thread-show <PID>           : Show given thread's coroutines' backtrace by thread-id
```

# 结语

协程已经被很多项目使用，从 C++ 20 开始标准库也开始支持协程，但对于协程的 GDB 支持讨论的很少。本文简单介绍了协程的实现，并以协程切换为切入点，深入 glibc 源码分析了在恢复协程上下文时需要特殊技巧才能还原关键寄存器，最终实现了在 GDB 中获取协程栈，以及栈上的变量，大大方便了协程的调试。

# 参考资料

https://www.kernel.org/doc/html/latest/x86/x86_64/fsgs.html

https://dere.press/2020/10/18/glibc-tls/#x86-64-abi-yao-qiu-de-tls-jie-gou

https://sourceware.org/glibc/wiki/PointerEncryption

https://github.com/qemu/qemu/blob/master/scripts/qemugdb/coroutine.py

# 作者介绍

Hehaoqian、Fengli ，SmartX 研发工程师。SmartX 拥有国内最顶尖的分布式存储和基础架构软件研发团队。

c/c++1

协程1

​

![](https://mp.weixin.qq.com/rr?timestamp=1725982229&src=11&ver=1&signature=s4-NRI8-bQyLoUHUbQ7UYGiy7CuvIbluYHu3*0jv8q4Or8rMVS5jBnOaoCoXv2JRmv89ZurbFLhoj*p7kI3P7o9MJ*u5*nyyyTGh20*Jhl8=)

微信扫一扫  
关注该公众号