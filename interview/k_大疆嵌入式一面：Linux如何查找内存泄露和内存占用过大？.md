
Original 往事敬秋风 深度Linux _2024年08月23日 09:10_ _湖南_

作为程序员，最常见的就是排查内存泄漏，不过我们一般的内存泄漏是针对特定的程序去排查，相对来说比较容易,但是如果是维护人员，不知道哪个程序有内存泄漏,甚至是应用程序的内存泄漏，还是内核的内存泄漏都不明确，所以一定要有一定的查内存泄漏的章法。

# (1) 内存泄漏是什么？

内存泄漏是指程序运行过程中分配的内存没有被正确释放，导致这部分内存无法再次使用，从而造成内存资源的浪费。内存泄漏可能会导致系统性能下降、程序崩溃或者消耗过多的系统资源；内存泄漏通常发生在动态分配的堆内存上，当程序通过调用malloc、new等函数来申请内存空间时，在使用完毕后应该使用free、delete等函数来释放这些已经不再需要的空间。如果忘记了释放这些空间，就会造成内存泄漏。

常见的引起内存泄漏的原因包括：指针或引用未被正确清理、循环引用、缓冲区溢出等。解决内存泄漏问题需要仔细检查代码，并确保所有分配的内存都得到了适时释放，对于大型项目和长时间运行的程序，及时发现和解决潜在的内存泄漏问题非常重要。可以利用工具进行静态代码分析或者动态检测来帮助定位和修复内存泄漏问题。同时，良好的编码习惯和使用智能指针等技术也有助于预防和减少内存泄漏的发生。

(2)内存占用过大为什么？

内存占用过大的原因可能有很多，以下是一些常见的情况：

- 内存泄漏：当程序在运行时动态分配了内存但未正确释放时，会导致内存泄漏。这意味着那部分内存将无法再被其他代码使用，最终导致内存占用增加。

- 频繁的动态内存分配和释放：如果程序中频繁进行大量的动态内存分配和释放操作，可能会导致内存碎片化问题。这样系统将难以有效地管理可用的物理内存空间。

- 数据结构和算法选择不当：某些数据结构或算法可能对特定场景具有较高的空间复杂度，从而导致内存占用过大。在设计和选择数据结构和算法时应综合考虑时间效率和空间效率。

- 缓存未及时清理：如果程序中使用了缓存机制，并且没有及时清理或管理缓存大小，就会导致缓存占用过多的内存空间。

- 高并发环境下资源竞争：在高并发环境下，多个线程同时访问共享资源（包括对内存的申请和释放）可能引发资源竞争问题。若没有适当的同步机制或锁策略，可能导致内存占用过大。

- 第三方库或框架问题：使用的第三方库或框架可能存在内存管理不当、内存泄漏等问题，从而导致整体程序的内存占用过大。

# (3) 内存泄露和内存占用过大区别？

内存泄漏指的是在程序运行过程中，动态分配的内存空间没有被正确释放，导致这些内存无法再被其他代码使用。每次发生内存泄漏时，系统可用的物理内存空间就会减少一部分，最终导致整体的内存占用量增加。

而内存占用过大则是指程序在运行时所消耗的物理内存超出了合理范围或预期值。除了因为内存泄漏导致的额外占用外，其他原因如频繁的动态内存分配和释放、数据结构和算法选择不当、缓存管理问题等都可能导致程序的内存占用过大。

可以说，内存在被正确管理和使用时，即使有一定程度的动态分配和释放操作，也不会造成明显的长期累积效应，即不会出现持续性的内存占用过大情况。而如果存在未及时释放或回收的资源（即发生了内存泄漏），随着时间推移会逐渐积累并导致整体的内存占用越来越高。

因此，在排查和解决内存占用过大问题时，需要注意是否存在内存泄漏，并且还需综合考虑其他可能导致内存占用过大的因素。

# 一、概述

内存泄漏（Memory Leak）是指程序中已动态分配的堆内存由于某种原因程序未释放或无法释放，造成系统内存的浪费，导致程序运行速度减慢甚至系统崩溃等严重后果。

## 特点

- 隐蔽性 因为内存泄漏的产生原因是内存块未被释放，属于遗漏型缺陷而不是过错型缺陷
- 积累性 内存泄漏通常不会直接产生可观察的错误症状，而是逐渐积累，降低系统整体性能，极端的情况下可能使系统崩溃。最直观的问题就是为什么我们的程序开始运行好好的，过段时间就异常退出。

内存泄漏并非指内存在物理上的消失，而是应用程序分配某段内存后，由于使用错误，导致在释放该段内存之前就失去了对该段内存的控制，从而造成了内存未释放而浪费掉。

## 产生的原因

我们在进行程序开发的过程使用动态存储变量时，不可避免地面对内存管理的问题。程序中动态分配的存储空间，在程序执行完毕后需要进行释放。没有释放动态分配的存储空间而造成内存泄漏，是使用动态存储变量的主要问题。

一般情况下，作为开发人员会经常使用系统提供的内存管理基本函数，如malloc、realloc、calloc、free等，完成动态存储变量存储空间的分配和释放。但是，当开发程序中使用动态存储变量较多和频繁使用函数调用时，就会经常发生内存管理错误。

## 二、虚拟内存泄露

一般来说，我们观察系统的内存占用喜欢用top命令，然后输入m，对系统中整体的内存占用情况做个排序，然后在重点观察，内存占用排在前几位的进程，再逐步的分析。

```cpp
[root@VM-0-2-centos ~]# top -p 5576top - 18:21:46 up 198 days, 20:07,  2 users,  load average: 0.10, 0.04, 0.05Tasks:   1 total,   0 running,   1 sleeping,   0 stopped,   0 zombie%Cpu(s):  0.7 us,  0.3 sy,  0.0 ni, 99.0 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 stKiB Mem :  1882008 total,    78532 free,   116516 used,  1686960 buff/cacheKiB Swap:        0 total,        0 free,        0 used.  1606660 avail Mem  PID USER      PR  NI    VIRT    RES    SHR S %CPU %MEM     TIME+ COMMAND                                                                                                                  5576 root      20   0  184064  11248   1124 S  0.0  0.6  10:34.98 nginx   
```

虽然top 也可以观察到单独的进程的内存变化，不过一般不太好比较内存变化的规律，推荐使用pidstat工具，此工具需要先安装，通过命令：

```
yum install sysstat
```

pidstat 基本说明如下：

- u：默认的参数，显示各个进程的cpu使用统计
- r：显示各个进程的内存使用统计
- d：显示各个进程的IO使用情况
- p：指定进程号
- w：显示每个进程的上下文切换情况
- t：显示选择任务的线程的统计信息外的额外信息
- T { TASK | CHILD | ALL }

假如我们观察到如下的内存占用情况：`pidstat -r -p pid 5`

```c
[root@VM-0-2-centos ~]# pidstat -r -p 5981 5Linux 3.10.0-1127.19.1.el7.x86_64 (VM-0-2-centos)   07/24/2021  _x86_64_    (1 CPU)06:25:55 PM   UID       PID  minflt/s  majflt/s     VSZ    RSS   %MEM  Command06:26:00 PM     0      5981      0.20      0.00    4416    352   0.02  a.out06:26:05 PM     0      5981      0.00      0.00    4416    352   0.02  a.out06:26:10 PM     0      5981      0.20      0.00    4456    352   0.02  a.out06:26:15 PM     0      5981      0.00      0.00    4456    352   0.02  a.out06:26:20 PM     0      5981      0.00      0.00    4456    352   0.02  a.out06:26:25 PM     0      5981      0.20      0.00    4496    352   0.02  a.out06:26:30 PM     0      5981      0.00      0.00    4496    352   0.02  a.out06:26:35 PM     0      5981      0.20      0.00    4536    352   0.02  a.out06:26:40 PM     0      5981      0.00      0.00    4536    352   0.02  a.out06:26:45 PM     0      5981      0.20      0.00    4576    352   0.02  a.out06:26:50 PM     0      5981      0.00      0.00    4576    352   0.02  a.out06:26:55 PM     0      5981      0.20      0.00    4616    352   0.02  a.out 
```

我们注意下，VSZ即虚拟内存的占用每10s增加40，单位为k，即10s增加40k的虚拟内存，下面来具体分析下这个程序的内存泄露情况。

## 三、分析泄露原因

我们来分析这个进程的内存分布情况，来分析这泄露的内存有什么特点：

```c
[root@VM-0-2-centos ~]# pmap -x 59815981:   ./a.outAddress           Kbytes     RSS   Dirty Mode  Mapping0000000000400000       4       4       0 r-x-- a.out0000000000600000       4       4       4 r---- a.out0000000000601000       4       4       4 rw--- a.out00007faab436e000    2720     272     272 rw---   [ anon ]00007faab4616000    1804     260       0 r-x-- libc-2.17.so00007faab47d9000    2048       0       0 ----- libc-2.17.so00007faab49d9000      16      16      16 r---- libc-2.17.so00007faab49dd000       8       8       8 rw--- libc-2.17.so00007faab49df000      20      12      12 rw---   [ anon ]00007faab49e4000     136     108       0 r-x-- ld-2.17.so00007faab4a06000    2012     212     212 rw---   [ anon ]00007faab4c03000       8       8       8 rw---   [ anon ]00007faab4c05000       4       4       4 r---- ld-2.17.so00007faab4c06000       4       4       4 rw--- ld-2.17.so00007faab4c07000       4       4       4 rw---   [ anon ]00007ffe0f3f5000     132      16      16 rw---   [ stack ]00007ffe0f47c000       8       4       0 r-x--   [ anon ]ffffffffff600000       4       0       0 r-x--   [ anon ]---------------- ------- ------- ------- total kB            8940     940     564
```

其中Address为开始的地址，Kbytes是虚拟内存的大小，RSS为真实内存的大小，Dirty为未同步到磁盘上的脏页，Mode为内存的权限，rw为可写可读，rx为可读和可执行。通过几次观察，我们发现：

```c
00007faab436e000    2720     272     272 rw---   [ anon ]
```

为泄露部分，此为匿名内存区，也就是没有映射文件，为malloc或mmap分配的内存。同样是每次增加40K。

此时还是只能大概知道内存泄露的位置，我们还先找到具体的代码位置，这个该怎么分析？代码的申请，无非是通过malloc和brk这些库函数进行内存调用，我们可以用strace跟踪下。

```c
[root@VM-0-2-centos ~]# strace -f -t -p 5981 -o trace.stracestrace: Process 5981 attachedstrace: Process 8519 attachedstrace: Process 8533 attachedstrace: Process 8547 attachedstrace: Process 8557 attachedstrace: Process 8575 attached^Cstrace: Process 5981 detached
```

我们通过-t选项来显示时间，-f来跟踪子进程。直接用cat命令查看跟踪的文件内容，会发现内容相当多，只要是系统调用都打印了出来，可以通过每次增加40k这个有用的信息搜索下：

```c
[root@VM-0-2-centos ~]# grep 40960  trace.strace 5981  19:01:44 mmap(NULL, 40960, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS|MAP_STACK, -1, 0) = 0x7faab403a0005981  19:01:55 mmap(NULL, 40960, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS|MAP_STACK, -1, 0) = 0x7faab40300005981  19:02:06 mmap(NULL, 40960, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS|MAP_STACK, -1, 0) = 0x7faab40260005981  19:02:17 mmap(NULL, 40960, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS|MAP_STACK, -1, 0) = 0x7faab401c0005981  19:02:28 mmap(NULL, 40960, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS|MAP_STACK, -1, 0) = 0x7faab4012000
```

至此我们找到了具体的泄露代码位置。看下这个测试代码：

```c
#include <stdio.h>
#include <unistd.h>
#include <sys/mman.h>
#include <sys/types.h>
#include <sys/wait.h>
#define _SCHED_H 
#define __USE_GNU 
#include <bits/sched.h> 
#define STACK_SIZE 40960 
int func(void *arg){    printf("thread enter.\n");    sleep(1);    printf("thread exit.\n");     return 0;}int main(){    int thread_pid;    int status;    int w;     while (1) {        void *addr = mmap(NULL, STACK_SIZE, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS|MAP_STACK, -1, 0);        if (addr == NULL) {            perror("mmap");            goto error;        }        printf("creat new thread...\n");        thread_pid = clone(&func, addr + STACK_SIZE, CLONE_SIGHAND|CLONE_FS|CLONE_VM|CLONE_FILES, NULL);        printf("Done! Thread pid: %d\n", thread_pid);        if (thread_pid != -1) {            do {                w = waitpid(-1, NULL, __WCLONE | __WALL);                if (w == -1) {                    perror("waitpid");                    goto error;                }            } while (!WIFEXITED(status) && !WIFSIGNALED(status));        }        sleep(10);   } error:    return 0;}
```

这个测试程序利用mmap申请一块匿名私有的内存，clone为系统函数，pthread_create 和fork底层都是调用它，用来创建进程/线程，将func的地址指针存放在子进程堆栈的某个位置处，该位置就是该封装函数本身返回地址存放的位置，最后一个参数为func的执行参数。clone可以更灵活控制共享，比如可以控制是否共享内存空间，是否共享打开文件，是否共享相同的信号处理函数等。

我们可以看到，mmap申请内存后，需要通过munmap来释放，这里面没有释放，所以导致了虚拟内存泄露，这里面申请的内存只实际使用了4个字节，即复制了func的指针，其他的内存均没有使用，其实仔细观察会发现还有部分的物理内存泄露，每次4个字节，可以通过pmap -x 查到。

在 waitpid后面添加：munmap(addr,STACK_SIZE); 即可以实现内存的释放。

如何排查内存泄漏

> 我们平时开发过程中不可避免的会遇到内存泄漏问题，这是常见的问题。既然发生了内存泄漏，我们就要排查内存泄漏的问题。想必大家也经常会用到以下排查内存问题的工具，如下：

- memwatch
- mtrace
- dmalloc
- ccmalloc
- valgrind
- debug_new

## 四、valgrind 分析程序内存泄露

这个是比较常见的方法，一般通过下面命令来查看内存泄露：

```c
valgrind --tool=memcheck --leak-check=full ./b[root@VM-0-2-centos test]# valgrind --tool=memcheck --leak-check=full ./b==14374== Memcheck, a memory error detector==14374== Copyright (C) 2002-2017, and GNU GPL'd, by Julian Seward et al.==14374== Using Valgrind-3.15.0 and LibVEX; rerun with -h for copyright info==14374== Command: ./b==14374== ==14374== Warning: set address range perms: large range [0x5205040, 0x24605040) (undefined)address:0x5205040==14374== Warning: set address range perms: large range [0x5205040, 0x24605040) (defined)524288000==14374== ==14374== HEAP SUMMARY:==14374==     in use at exit: 524,288,000 bytes in 1 blocks==14374==   total heap usage: 1 allocs, 0 frees, 524,288,000 bytes allocated==14374== ==14374== 524,288,000 bytes in 1 blocks are possibly lost in loss record 1 of 1==14374==    at 0x4C29F73: malloc (vg_replace_malloc.c:309)==14374==    by 0x400675: main (test.c:17)==14374== ==14374== LEAK SUMMARY:==14374==    definitely lost: 0 bytes in 0 blocks==14374==    indirectly lost: 0 bytes in 0 blocks==14374==      possibly lost: 524,288,000 bytes in 1 blocks==14374==    still reachable: 0 bytes in 0 blocks==14374==         suppressed: 0 bytes in 0 blocks
```

明确说明在test.c的17行有可能是内存泄露：==14374== by 0x400675: main (test.c:17)

## 五、AddressSanitizer(ASan)工具

AddressSanitizer（ASan）是一种内存错误检测工具，属于Clang/LLVM编译器工具套件的一部分。它用于检测和调试C/C++程序中的内存问题，如缓冲区溢出、使用已释放或未初始化的内存等。

ASan在编译时通过插入额外的代码来动态地检测程序运行过程中的内存访问错误。它会跟踪每个分配的内存块，并在程序执行期间监视对这些内存块的访问情况。如果发现任何非法操作，比如访问已释放或越界的内存，ASan将立即报告该错误，并提供有关问题位置和堆栈跟踪信息。

使用ASan可以帮助开发人员及早发现和修复潜在的内存错误，提高程序的稳定性和安全性。但需要注意，由于ASan会引入额外代码和运行时检查，可能会增加程序运行时的开销和内存占用量。它非常快，只拖慢程序两倍左右（比起Valgrind快多了）。它包括一个编译器instrumentation模块和一个提供malloc()/free()替代项的运行时库。从gcc 4.8开始，AddressSanitizer成为gcc的一部分。当然，要获得更好的体验，最好使用4.9及以上版本，因为gcc 4.8的AddressSanitizer还不完善，最大的缺点是没有符号信息。

使用方法：

- 用-fsanitize=address选项编译和链接你的程序。

- 用-fno-omit-frame-pointer编译，以得到更容易理解stack trace。

- 可选择-O1或者更高的优化级别编译

```c
gcc -fsanitize=address -o main -g main.c 
```

案例分析：

```c
#include <iostream>
int main() {    int* arr = new int[5];    // 内存访问错误 - 越界访问数组    for (int i = 0; i <= 5; ++i) {        arr[i] = i;    }    delete[] arr;    return 0;}
```

在使用ASan进行编译和运行时，你可以按照以下步骤：

（1）确保你已经安装了Clang/LLVM编译器。

（2）将上述代码保存到一个名为`main.cpp`的文件中。

（3）执行以下命令来编译程序，并启用ASan工具：

```c
clang++ -fsanitize=address -g main.cpp -o test
```

（4）运行生成的可执行文件：

```c
./test
```

当程序运行时，如果存在任何内存错误，ASan将会报告并显示相应的错误信息，包括错误位置、堆栈跟踪等。

## 六、其他内存泄露分析

其实上面的内存泄露是我们知道了具体的泄露的进程，然后再做详细分析。那么如果不知道哪里内存泄露了，有什么办法，可以通过分析meminfo文件，来观察泄露的类型。

```c
[root@VM-0-2-centos test]# cat /proc/meminfoMemTotal:        1882008 kBMemFree:          752948 kBMemAvailable:    1610108 kBBuffers:          564900 kBCached:           399584 kBSwapCached:            0 kBActive:           808140 kBInactive:         220812 kBActive(anon):      64548 kBInactive(anon):      488 kBActive(file):     743592 kBInactive(file):   220324 kBUnevictable:           0 kBMlocked:               0 kBSwapTotal:             0 kBSwapFree:              0 kB....
```

‌meminfo文件‌是Linux系统中用于显示内存使用情况的详细信息文件，它位于`/proc`目录下，提供了关于系统内存使用的全面信息。通过查看和分析meminfo文件的内容，可以了解系统的内存使用状况，包括总内存、空闲内存、缓存、交换分区等信息，这对于排查内存相关的问题非常有帮助。

meminfo文件包含的主要信息及其含义如下：

- ‌MemTotal‌：系统总内存大小。

- ‌MemFree‌：系统空闲内存大小。

- ‌MemAvailable‌：可用内存大小，包括空闲内存和缓存。

- ‌Buffers‌：用于缓存数据的内存大小。

- ‌Cached‌：用于缓存文件系统的内存大小。

- ‌SwapCached‌：用于缓存交换分区的内存大小。

- ‌Active‌ 和 ‌Inactive‌：分别表示活动和非活动内存大小，即正在使用或最近使用的内存和最近没有使用的内存。

- ‌SwapTotal‌ 和 ‌SwapFree‌：交换分区总大小和空闲大小。

- ‌Dirty‌ 和 ‌Writeback‌：等待写回到磁盘的内存大小和正在写回到磁盘的内存大小。

- ‌AnonPages‌、‌Mapped‌、‌Shmem‌ 等：分别表示用于匿名映射、已映射到文件的内存、共享内存大小等。

- ‌Slab‌、‌SReclaimable‌、‌SUnreclaim‌ 等：内核数据结构缓存的内存大小以及可回收和不可回收的Slab内存大小。

- ‌KernelStack‌、‌PageTables‌ 等：内核栈的内存大小和页面表的内存大小。

- ‌CommitLimit‌ 和 ‌Committed_AS‌：可用内存可支持的最大内存大小和已分配的内存大小，包括内存和交换分区。

- ‌VmallocTotal‌、‌VmallocUsed‌ 等：虚拟内存总大小和已使用的虚拟内存大小。

排查内存问题时，可以通过以下步骤进行：

- 首先，使用`cat /proc/meminfo`命令查看meminfo文件的内容，了解系统的整体内存使用情况。
- 分析MemTotal和MemFree的值，了解系统的总内存和可用空闲内存。
- 注意MemAvailable的值，它表示应用程序可用的内存，与MemFree的区别在于MemAvailable考虑了Buffers和Cached的大小，这些通常在系统需要时可以被回收。
- 检查SwapUsage（虽然meminfo文件中没有直接显示SwapUsage，但可以通过SwapTotal和SwapFree计算得出），如果Swap空间被大量使用，可能意味着物理内存不足。
- 注意Active、Inactive、Dirty和Writeback等值，这些指标可以帮助你了解系统当前的内存使用模式和可能的性能瓶颈。
- 如果发现某些特定类型的内存使用异常高（如AnonPages、Shmem等），可能需要进一步调查这些类型的内存使用情况，以确定是否存在内存泄漏或其他问题。
- 使用其他工具如`free`、`vmstat`、`top`或`htop`等命令提供的信息与meminfo文件的内容进行对比，以获得更全面的系统内存使用情况视图。

通过综合分析meminfo文件的内容和其他相关工具的输出，可以有效地排查和解决Linux系统中的内存相关问题‌

C/C++开发95

面试八股文4

内存38

C/C++开发 · 目录

上一篇腾讯一面：malloc是如何分配内存的，free怎么知道该释放多少内存？下一篇C/C++一站式就业校招指导，细分方向和项目准备

Reads 14.0k

​
