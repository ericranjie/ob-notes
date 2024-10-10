
2024-03-28 [CPU篇](https://kfngxl.cn/index.php/category/cpu/) 阅读(213) 评论(0)

相信各位同学都听说过一个建议，就是系统调用比函数调用开销大很多，要尽量减少系统调用的次数，以提高你的代码的性能。那么问题来了，我们是否可以给出量化的指标。一次系统调用到底要多大的开销，需要消耗掉多少CPU时间？

首先说说系统调用是什么，当你的代码需要做IO操作（open、read、write）、或者是进行内存操作（mmpa、sbrk）、甚至是说要获取一个系统时间（gettimeofday），就需要通过系统调用来和内核进行交互。无论你的用户程序是用什么语言实现的，是php、c、java还是go，只要你是建立在Linux内核之上的，你就绕不开系统调用。

![[Pasted image 20241010225439.png]]

大家可以通过strace命令来查看到你的程序正在执行哪些系统调用。比如我查看了一个正在生产环境上运行的nginx当前所执行的系统调用，如下:

```c
# strace -p 28927
Process 28927 attached  
epoll_wait(6, {{EPOLLIN, {u32=96829456, u64=140312383422480}}}, 512, -1) = 1  
accept4(8, {sa_family=AF_INET, sin_port=htons(55465), sin_addr=inet_addr("10.143.52.149")}, [16], SOCK_NONBLOCK) = 13  
epoll_ctl(6, EPOLL_CTL_ADD, 13, {EPOLLIN|EPOLLRDHUP|EPOLLET, {u32=96841984, u64=140312383435008}}) = 0  
epoll_wait(6, {{EPOLLIN, {u32=96841984, u64=140312383435008}}}, 512, 60000) = 1  
 
```

简单介绍了下系统调用，那么相信各位同学都听说过一个建议，就是系统调用的开销很大，要尽量减少系统调用的次数，以提高你的代码的性能。那么问题来了，我们是否可以给出量化的指标。一次系统调用到底要多大的开销，需要消耗掉多少CPU时间？好了，废话不多说，我们直接进行一些测试，用数据来说话。

# 实验1

首先我对线上正在服务的nginx进行strace统计，可以看出系统调用的耗时大约分布在1-15us左右。因此我们可以大致得出结论，系统调用的耗时大约是1us级别的，当然由于不同系统调用执行的操作不一样，执行当时的环境不一样，因此不同的时刻，不同的调用之间会存在耗时上的上下波动。

```c
# strace -cp 8527  
strace: Process 8527 attached  
% time     seconds  usecs/call     calls    errors syscall  
------ ----------- ----------- --------- --------- ----------------  
 44.44    0.000727          12        63           epoll_wait  
 27.63    0.000452          13        34           sendto 
 10.39    0.000170           7        25        21 accept4  
  5.68    0.000093           8        12           write  
  5.20    0.000085           2        38           recvfrom  
  4.10    0.000067          17         4           writev  
  2.26    0.000037           9         4           close  
  0.31    0.000005           1         4           epoll_ctl 
```

# 实验2

我们再手工写段代码，对read系统调用进行测试，代码参见[test02](https://kfngxl.cn/index.php/archives/608/tests/test02/main.c)

> 注意，只能用read库函数来进行测试，不要使用fread。因此fread是库函数在用户态保留了缓存的，而read是你每调用一次，内核就老老实实帮你执行一次read系统调用。

首先创建一个固定大小为1M的文件

```c
dd if=/dev/zero of=in.txt bs=1M count=1
```

然后再编译代码进行测试

```c
#cd tests/test02/  
#gcc main.c -o main  
#time ./main  
real    0m0.258s   
user    0m0.030s  
sys     0m0.227s  
```

由于上述实验是循环了100万次，所以平均每次系统调用耗时大约是200ns多一些。

# 系统调用到底在干什么？

## 先看看系统调用花费的CPU指令数

x86-64 CPU有一个特权级别的概念。内核运行在最高级别，称为Ring0，用户程序运行在Ring3。正常情况下，用户进程都是运行在Ring3级别的，但是磁盘、网卡等外设只能在内核Ring0级别下来来访问。因此当我们用户态程序需要访问磁盘等外设的时候，要通过系统调用进行这种特权级别的切换

对于普通的函数调用来说，一般只需要进行几次寄存器操作，如果有参数或返回函数的话，再进行几次用户栈操作而已。而且用户栈早已经被CPU cache接住，也并不需要真正进行内存IO。

但是对于系统调用来说，这个过程就要麻烦一些了。系统调用时需要从用户态切换到内核态。由于内核态的栈用的是内核栈，因此还需要进行栈的切换。SS、ESP、EFLAGS、CS和EIP寄存器全部都需要进行切换。

而且栈切换后还可能有一个隐性的问题，那就是CPU调度的指令和数据一定程度上破坏了局部性原理，导致一二三级数据缓存、TLB页表缓存的命中率一定程度上有所下降。

除了上述堆栈和寄存器等环境的切换外，系统调用由于特权级别比较高，也还需要进行一系列的权限校验、有效性等检查相关操作。所以系统调用的开销相对函数调用来说要大的多。我们在[test02](https://kfngxl.cn/index.php/archives/608/tests/test02/main.c)的基础上计算一下每个系统调用需要执行的CPU指令数。

```c
# perf stat ./main

 Performance counter stats for './main':

        251.508810 task-clock                #    0.997 CPUs utilized
                 1 context-switches          #    0.000 M/sec
                 1 CPU-migrations            #    0.000 M/sec
                97 page-faults               #    0.000 M/sec
       600,644,444 cycles                    #    2.388 GHz                     [83.38%]
       122,000,095 stalled-cycles-frontend   #   20.31% frontend cycles idle    [83.33%]
        45,707,976 stalled-cycles-backend    #    7.61% backend  cycles idle    [66.66%]
     1,008,492,870 instructions              #    1.68  insns per cycle
                                             #    0.12  stalled cycles per insn [83.33%]
       177,244,889 branches                  #  704.726 M/sec                   [83.32%]
             7,583 branch-misses             #    0.00% of all branches         [83.33%]
```

对实验代码进行稍许改动，把for循环中的read调用注释掉，再重新编译运行

```c
# gcc main.c -o main  
# perf stat ./main  

 Performance counter stats for './main':  

          3.196978 task-clock                #    0.893 CPUs utilized
                 0 context-switches          #    0.000 M/sec
                 0 CPU-migrations            #    0.000 M/sec
                98 page-faults               #    0.031 M/sec
         7,616,703 cycles                    #    2.382 GHz                       [68.92%]
         5,397,528 stalled-cycles-frontend   #   70.86% frontend cycles idle      [68.85%]  
         1,574,438 stalled-cycles-backend    #   20.67% backend  cycles idle  
         3,359,090 instructions              #    0.44  insns per cycle  
                                             #    1.61  stalled cycles per insn  
         1,066,900 branches                  #  333.721 M/sec
               799 branch-misses             #    0.07% of all branches           [80.14%]  

       0.003578966 seconds time elapsed  
```

平均每次系统调用CPU需要执行的指令数（1,008,492,870 - 3,359,090）/1000000 = 1005。

## 再深挖系统调用的实现

如果非要扒到内核的实现上，我建议大家参考一下《深入理解LINUX内核-第十章系统调用》。最初系统调用是通过汇编指令int（中断）来实现的，当用户态进程发出int $0x80指令时，CPU切换到内核态并开始执行system_call函数。 只不过后来大家觉得系统调用实在是太慢了，因为int指令要执行一致性和安全性检查。后来Intel又提供了“快速系统调用”的sysenter指令，我们验证一下。

```c
# perf stat -e syscalls:sys_enter_read ./main  

 Performance counter stats for './main':  

            1,000,001 syscalls:sys_enter_read  

       0.006269041 seconds time elapsed  
```

上述实验证明，系统调用确实是通过sys_enter指令来进行的。

# 相关命令

- strace

  - strace -p $PID: 实时统计进程陷入的系统调用
  - strace -cp $PID: 对进程执行一段时间内的汇总，然后以排行榜的形式给出来，非常实用

- perf

  - perf list： 列出所有能够perf采样点
  - perf stat： 统计CPU指令数、上下文切换等缺省时间
  - perf stat -e 事件： 指定采样时间进行统计
  - perf top： 统计整个系统内消耗最多的函数或指令
  - perf top -e： 同上，但是可以指定采样点

# 结论

- 系统调用虽然使用了“快速系统调用”指令，但耗时仍大约在200ns+，多的可能到十几us
- 每个系统调用内核要进行许多工作，大约需要执行1000条左右的CPU指令

> 系统调用确实开销蛮大的，函数调用时ns级别的，系统调用直接上升到了百ns，甚至是十几us，所以确实应该尽量减少系统调用。但是即使是10us，仍然是1ms的百分之一，所以还没到了谈系统调用色变的程度，能理性认识到它的开销既可。
>
> 为什么系统调用之间的耗时相差这么多？因为系统调用花在内核态用户态的切换上的时间是差不多的，但区别在于不同的系统调用当进入到内核态之后要处理的工作不同，呆在内核态里的时候相差较大。

写在最后，由于我的这些知识在公众号里文章比较分散，很多人似乎没有理解到我对知识组织的体系结构。而且图文也不像视频那样理解起来更直接。所以我在知识星球上规划了视频系列课程，包括**硬件原理、内存管理、进程管理、文件系统、网络管理、Golang语言、容器原理、性能观测、性能优化九大部分大约 120 节内容**，每周更新。加入方式参见[我要开始搞知识星球啦](https://mp.weixin.qq.com/s/_8ux274sY-As__Xwoqmewg)、[如何才能高效地学习技术,我投“融汇贯通”一票](https://mp.weixin.qq.com/s/z82z9jqnt08gBLYGxLHY2g)

Github：[https://github.com/yanfeizhang/coder-kung-fu](https://github.com/yanfeizhang/coder-kung-fu)\

---
