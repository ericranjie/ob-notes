# 

Original 陈涛 酷玩BPF

_2024年09月29日 08:20_ _四川_

在libbpf/bpftool的github issue列表上经常可以看到maintainer推荐retsnoop工具去定位eBPF工具执行失败的原因，例如：

![Image](https://mmbiz.qpic.cn/sz_mmbiz_png/RWnbgykjTNIqybQF1kFqGzrSlicPCAa36CdsADBQ9b6jZx344d1LeyNMGph7GfibpBEwKjwk9Cicichc8w4ibleQBWg/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

retsnoop本身也是出自Andrii之手，libbpf 70%的代码贡献都来自该大佬，如此光环下足见该工具多么硬核。

使用 retsnoop 的一个最典型的场景是执行eBPF工具或者其他应用app时遇到来自内核的报错，但不清楚是哪部分内核代码出错了，当然我们也可以用bpftrace等工具挨个函数打点去追踪内核代码，但这样做效率显然很低。而retsnoop的做法是我们可以通过通配符定义的方式，对相关函数做全量trace，只要遇到函数返回错误，就会把这部分代码的栈信息显示出来，从而方便问题的快速定位。

# 快速入门

这部分内容直接取至retsnoop项目的使用文档，工具有很多控制参数，这里只选几个常用的介绍，更详细的介绍可以直接参考项目文档。

retsnoop 支持三种不同且互补的模式：

### stack trace mode

默认的**堆栈跟踪模式**简洁地指向满足用户条件（例如，从系统调用返回的错误）的最深函数调用堆栈。它显示了一系列函数调用、堆栈每一层的相应源代码位置，并发出延迟和返回结果：

`$ sudo ./retsnoop -e '*sys_bpf' -a ':kernel/bpf/*.c'   Receiving data...   20:19:36.372607 -> 20:19:36.372682 TID/PID 8346/8346 (simfail/simfail):                          entry_SYSCALL_64_after_hwframe+0x63  (arch/x86/entry/entry_64.S:120:0)                       do_syscall_64+0x35                   (arch/x86/entry/common.c:80:7)                       . do_syscall_x64                     (arch/x86/entry/common.c:50:12)       73us [-ENOMEM]  __x64_sys_bpf+0x1a                   (kernel/bpf/syscall.c:5067:1)       70us [-ENOMEM]  __sys_bpf+0x38b                      (kernel/bpf/syscall.c:4947:9)                       . map_create                         (kernel/bpf/syscall.c:1106:8)                       . find_and_alloc_map                 (kernel/bpf/syscall.c:132:5)   !   50us [-ENOMEM]  array_map_alloc   !*   2us [NULL]     bpf_map_alloc_percpu   ^C   Detaching... DONE in 251 ms.   `

### func trace mode

**函数调用跟踪模式**（ `-T` ）还提供了给定函数集的控制流的详细跟踪，允许更全面地理解内核行为：

`FUNCTION CALL TRACE                               RESULT                 DURATION   -----------------------------------------------   --------------------  ---------   → bpf_prog_load       → bpf_prog_alloc           ↔ bpf_prog_alloc_no_stats                 [0xffffc9000031e000]    5.539us       ← bpf_prog_alloc                              [0xffffc9000031e000]   10.265us       [...]       → bpf_prog_kallsyms_add           ↔ bpf_ksym_add                            [void]                  2.046us       ← bpf_prog_kallsyms_add                       [void]                  6.104us   ← bpf_prog_load                                   [5]                   374.697us   `

### lbr mode

**lbr 模式**（最后分支记录）允许用户“回顾”并更深入地了解各个函数的内部结构，跟踪“不可见”的内联函数，并将问题一直精确到各个 C 语句。当跟踪内核的不熟悉部分而不知道要查找什么时，此模式特别有用。它支持迭代发现过程，而无需太多了解在哪里查找以及哪些功能是相关的。**但此模式依赖intel cpu硬件特性，同时也只有在5.16+内核上才能使用。**

`$ sudo ./retsnoop -e '*sys_bpf' -a 'array_map_alloc_check' --lbr=any   Receiving data...   20:29:17.844718 -> 20:29:17.844749 TID/PID 2385333/2385333 (simfail/simfail):   ...   [#22] ftrace_trampoline+0x14c                                    ->  array_map_alloc_check+0x5   (kernel/bpf/arraymap.c:53:20)   [#21] array_map_alloc_check+0x13  (kernel/bpf/arraymap.c:54:18)  ->  array_map_alloc_check+0x75  (kernel/bpf/arraymap.c:54:18)         . bpf_map_attr_numa_node    (include/linux/bpf.h:1735:19)      . bpf_map_attr_numa_node    (include/linux/bpf.h:1735:19)   [#20] array_map_alloc_check+0x7a  (kernel/bpf/arraymap.c:54:18)  ->  array_map_alloc_check+0x18  (kernel/bpf/arraymap.c:57:5)         . bpf_map_attr_numa_node    (include/linux/bpf.h:1735:19)   [#19] array_map_alloc_check+0x1d  (kernel/bpf/arraymap.c:57:5)   ->  array_map_alloc_check+0x6f  (kernel/bpf/arraymap.c:62:10)   [#18] array_map_alloc_check+0x74  (kernel/bpf/arraymap.c:79:1)   ->  __kretprobe_trampoline+0x0   ...   `

另外也支持进程号过滤：

`-p, --pid=PID              Only trace given PID. Can be specified multiple                                times     -P, --no-pid=PID           Skip tracing given PID. Can be specified multiple                                times`

如果只关注内核函数执行时长，可以通过-L参数：

`-L, --longer=MS            Only emit stacks that took at least a given amount                                of milliseconds`

# 源码解析

resnoop用了大量的高版本eBPF特性，仅hook内核函数的方式就支持fentry、kprobe、kprobe_multi三种。除此之外，也用到了ringbuffer、全局变量、lbr特性等，如果想完整使用retsnoop的功能，建议使用5.16+内核。下面简单介绍下各\*.bpf.c文件的作用。

### 相关文件

- calib_feat.bpf.c

探测当前系统是否满足eBPF的高版本特性，例如是否支持ringbuffer、kprobe_mult、lbr采栈i等，择机使用当前系统上所能支持的eBPF特性。

- mass_attacher.c

所有hook函数的入口，handle_func_entry/handle_func_exit 作为通过入口hook所有函数

- retsnoop.bpf.c

整个retsnoop核心功能实现在此文件中，其实现的大致思路如下图所示。

### 整体实现思路

1. 用户传入通配符匹配参数，通过通配符匹配如 “_sys_bpf_”，将所有跟sys_bpf相关的目标函数都会做hook处理；

1. 每个目标函数开始执行和执行返回都会做hook处理，如下图步骤1，2，在函数开始执行时记录时间、seq_id(唯一标识)等信息，在函数返回时根据其返回结果判断函数是否正常(是否为0、是否非NULL等)，如果不正常就记录其栈信息，如下图步骤3

1. 最后当记录的栈深度为0时，既如下图func1返回时，上报记录的追踪信息，如下图步骤4

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

### 使用案例

下面介绍几个之前借助retsnoop工具解决的问题。

1.percpu map value值32K大小限制

工具开发中，percpu map是我们经常用到的map类型，但其value 值有32K的限制，一旦超过该大小，当我们跑工具时会有右图的报错，单看“Cannot allocate memory”提示内存不足，但查整机还有很多free内存。直到使用retsnoop才看到是申请percpu 内存时返回了NULL，根因就是内核中`PCPU_MIN_UNIT_SIZE` (32K)的限制。

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

2.aarch64 原子指令未支持，造成load失败

在5.15的aarch64内核上执行eBPF工具，报了“failed to load：-524”错误，以往当verifier失败时会有相关的内核日志，而此问题却只有一个错误码，难以定位错误原因。因此使用retsnoop抓取bpf load相关的函数，可以看到bpf_prog_load时内核报了”ENOTSUPPORT”错误，凭借此函数的错误提示定位到是代码中使用了sync_fetch_and_and原子子令，而aarch64下5.15的内核对该eBPF指令还未支持。

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

这么好用的工具只能在高版本使用确实心有不甘，所以感兴趣的朋友也可以尝试将其移植到低版本(仅支持kprobe和perf buffer的内核(4.19等))中，如果有相关实现也欢迎在给我们投稿。

请关注本公众号，如果你有eBPF及linux相关问题，请联系微信号 wenamao，邀请加入 酷玩BPF学习交流群。

# 参考

https://github.com/anakryiko/retsnoop

Reads 527

​

[](javacript:;)

![](http://mmbiz.qpic.cn/sz_mmbiz_png/RWnbgykjTNLYUsm55aeV0aP5Qet4yIGmBEsbwoZdgdUdYTGJJrYknQLvVdzjs9wIdkHrNoKfEGcvCyy7HFjTvA/300?wx_fmt=png&wxfrom=18)

酷玩BPF

7603

Comment

Comment

**Comment**

暂无留言
