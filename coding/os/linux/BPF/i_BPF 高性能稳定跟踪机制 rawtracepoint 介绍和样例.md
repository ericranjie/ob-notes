

Linux内核之旅

 _2024年03月10日 15:48_ _陕西_

以下文章来源于深入浅出BPF ，作者daviddi

[

![](http://wx.qlogo.cn/mmhead/Q3auHgzwzM6qJiaaicEDXMrKNnhx5D6WCIYOhyctx1l1TLk6mT7zwsBQ/0)

**深入浅出BPF**.

专注 BPF 及相关基础技术

](https://mp.weixin.qq.com/s?__biz=MzI3NzA5MzUxNA==&mid=2664616937&idx=1&sn=4dad54909d217fc160c424361965210e&chksm=f04dfe0cc73a771a589e3a1e8023e1b2774055ba4443746375881dfe7421ccdbe45aa9d19c25&mpshare=1&scene=24&srcid=0310VCMNNgutdFGS8H8HM7Jp&sharer_shareinfo=0b6d6b1b5f3bd5ce0e0f32ac82753bcd&sharer_shareinfo_first=0b6d6b1b5f3bd5ce0e0f32ac82753bcd&key=daf9bdc5abc4e8d0b7e93933f166b6a1a4199075a5317f47810c85e498f1cb0b9461cb87e879d3f8c734be915798da89ff7c32b7ff912d14962d8f87a891f3cb0f3d2858cb366bdffe0f81c031900a98ec9016d04836f8008f451c0428c06276b547f4123c0fc9a6722a52e33b0b17f16bb25e6709debfa20a9cd6c554e15dc3&ascene=0&uin=MTEwNTU1MjgwMw%3D%3D&devicetype=Windows+11+x64&version=63090b19&lang=zh_CN&countrycode=CN&exportkey=n_ChQIAhIQARtB3j6pBLrQm5GXSLZcxRLmAQIE97dBBAEAAAAAAA9iDrhSJCIAAAAOpnltbLcz9gKNyK89dVj0blIT%2BBks0RQtuGKEshQ59lvH%2BczI%2F6%2BFa9E4TF%2Fhn8lPWRYurfb7W5AFDonkg%2FUMUe9ahNXSWLjiBuiXZm%2Bw%2FLF3hze8PPyTsIpXrxF%2BHHXJ21vsD0YW5PxP8haPnewpY7hfBYlq5GIvZjNsbCZcRFXRnX0RNnP5OmxroTriRnwQ30Cf8LVnvTbn1fgWYmzVuKovi0pcUBFzWF6e4qZPZxEP1d20%2BjXpijxvjs1hqH50vncE61gnJDsOINMqJgfe&acctmode=0&pass_ticket=7sndNVjg6wC371M7Tl20zmozv0QqsR37%2B8yyRUAFQ7xpFUjw2QtDc9%2FRXCYX3G7B&wx_header=1&fasttmpl_type=0&fasttmpl_fullversion=7350504-zh_CN-zip&fasttmpl_flag=1#)

![](http://mmbiz.qpic.cn/mmbiz_png/CrEZgiblEdJWdTnicQwNnku8Bf0M2ibrxIiannSqT6OOppxiaaQxsqiaTeLq8YAc8SRsL3AH6icXW29ypuQzsXUNfwaRg/300?wx_fmt=png&wxfrom=19)

**深入浅出BPF**

专注 BPF 及相关基础技术

38篇原创内容

公众号

本文就 Linux Trace 跟踪过程中的主要 Hook 点给出了简单介绍，然后重点介绍了 rawtracepoint 类型出现的背景，性能测试说明，并通过 libbpf 和 bpftrace 分别基于  task_rename 给出了完整实现，介绍过程结合 tracepoint 跟踪点给出对比说明。

## 1. Trace 跟踪常见的 Hook 类型

通过`eBPF`可以对多种类型的事件进行跟踪，在  trace 领域分类如下：

- 内核静态跟踪点  `tracepoint`/`rawtracepoint`/`btf-tracepoint`
    

- 参见  `/sys/kernel/tracing/available_events`
    

- 内核动态跟踪点  `k[ret]probe`， `fentry/fexit`  (基于 BTF)
    

- Kprobe  `/sys/kernel/tracing/available_filter_functions`
    

- 用户空间静态跟踪点 USDT
    

- 查看方式  `readelf -n` 或  bpftrace 工具 `bpftrace -l 'usdt:/home/dave/ebpf/linux-tracing/usdt/main:*'`
    

- 用户空间动态跟踪：`u[ret]probe`，可通过 `nm hello | grep main` 查看
    
- 性能监控计数器 PMC
    
- `perf_event`
    

本文我们重点讨论一下内核静态跟踪中的 `rawtracepoint`，最后我们基于 libbpf 开发库和 bpftrace 给出实际代码样例。

## 2. BPF 原始跟踪点 rawtracepoint

eBPF 的作者 Alexei Starovoitov 在 Linux 内核 4.17 版本中添加了一个原始跟踪点（rawtracepoint）。`rawtracepoint` 与 `tracepoint` 相比直接暴露原始参数，一定程度上避免创建稳定跟踪点参数的带来的性能开销，但由于直接对用户暴露了原始参数，因此这是属于动态跟踪的模式，属于不稳定的跟踪模式。`rawtracepoint` 相比较 `kprobe` 来讲相对稳定，因为跟踪点无论是名字还是参数变化相对低频，而相对于 `tracepoint` 的跟踪方式可提供更优的性能。`rawtracepoint` 提交实现可参见：bpf: introduce BPF_RAW_TRACEPOINT[1]。从作者提交的性能压测报告相比较 kprobe 和 tracepoint 跟踪都具有性能提升[2]，比较适用于长期监控频繁次调用的函数，比如系统调用。Tracee 安全产品监控系统调用的实现就采用 rawtracepoint 的方式[3]。

### 2.1 跟踪性能优化提升 20%

表格为作者提交时候的原始性能数据：

`tracepoint    base  kprobe+bpf tracepoint+bpf raw_tracepoint+bpf   task_rename   1.1M   769K        947K            1.0M   urandom_read  789K   697K        750K            755K   `

下图的数据是我基于内核代码中官方提供的 bench 工具运行并绘制的（运行需要提前编译内核代码），纵坐标是每秒运行的指令数：
![[Pasted image 20240920131424.png]]
![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

perf comparision of linux trace

性能压测运行方式如下：

`$ cd tools/testing/selftests/bpf   $ ./benchs/run_bench_trigger.sh   `

### 2.2 rawtracepoint 跟踪事件查看及数量统计

bpftrace 在 0.19 版本支持 rawtracepoint[4]。可以使用 bpftrace -l 查看，程序类型缩写为 rt[5]，参数类型为 arg0, arg1..。我们可使用 bpftrace -l 查看到全部的列表：

`$ sudo bpftrace -l "rawtracepoint:*"   `

在 Ubuntu 22.04 系统中 （内核版本 6.2） 系统中大概有 1480 多个：

`$ sudo bpftrace -l "rawtracepoint:*"|wc -l   1480      $ sudo bpftrace -l "tracepoint:*"|wc -l   2124   `

细心的童靴，可能以已经注意到系统中的 tarcepoint 事件却有 2124 个，这是什么原因导致的呢？

在 bpftrace 中是如何获取到 `rawtracepoint` 呢？通过分析源码，我们可以得知其实 bpftrace 中是读取了 `/sys/kernel/debug/tracing/available_events` 文件中的所有跟踪点，同时排除一部分以 `syscalls:sys_enter_` 或者 `syscalls:sys_exit_` 开头的跟踪事件。之所以要排查是因为有两个特殊情况，是因为：

- 统一用 `sys_enter` 表示 `syscalls` 分类下的 `sys_enter_xxx` 事件: `SEC("raw_tracepoint/sys_enter")`
    
- 统一用 `sys_exit` 表示 `syscalls` 分类下的 `sys_exit_xxx` 事件: `SEC("raw_tracepoint/sys_exit")`
    

即可以用 `sys_enter` 和 `sys_exit` 事件来监控所有系统调用事件。

可以通过查看 `/sys/kernel/debug/tracing/available_events` 文件的内容找到 `rawtracepoint` 可监控的事件。文件中每行内容的格式是:

`# <category>:<name>   skb:kfree_skb   `

但是，在 `rawtracepoint` 用到的是 `<name>` 的值，而不是整个 `<category>:<name>`：例如

`$ bpftrace -e 'rawtracepoint:kfree_skb  { printf("%s\n", comm)} '   Attaching 1 probe...   `

### 2.3 传递参数变化

从 BPF 程序角度来看，`rawtracepoint`  方式的参数定义和访问如下所示，后续我们将给出完整的使用样例程序。

`struct bpf_raw_tracepoint_args {          __u64 args[0];   };      int bpf_prog(struct bpf_raw_tracepoint_args *ctx)   {     // program can read args[N] where N depends on tracepoint     // and statically verified at program load+attach time   }   `

所有的参数都会通过数组指针的方式传入。这里我们基于 `__set_task_comm` 函数中定义的 `task_rename` 跟踪点通过 `tracepoint` 和 `rawtracepoint` 跟踪参数对比为例，`task_rename` 跟踪点函数在内核中的函数声明如下：

`void __set_task_comm(struct task_struct *tsk, const char *buf, bool exec);   // rawtracepoint 方式下直接压入原始参数 tsk/buf/exec   `

如果系统没有 task_rename 事件，我们可以编译如下程序手工触发验证测试：

`// gcc -o rename test_rename.c   #include <stdio.h>   #include <stdlib.h>   #include <errno.h>   #include <sys/types.h>   #include <sys/stat.h>   #include <fcntl.h>   #include <unistd.h>   #include <string.h>      #define MAX_CNT 1      static void test_task_rename(int cpu)   {    char buf[] = "test\n";    int i, fd;       fd = open("/proc/self/comm", O_WRONLY|O_TRUNC);    if (fd < 0) {     printf("couldn't open /proc\n");     exit(1);    }    for (i = 0; i < MAX_CNT; i++) {     if (write(fd, buf, sizeof(buf)) < 0) {      printf("task rename failed: %s\n", strerror(errno));      close(fd);      return;     }    }    close(fd);   }      int main()   {    test_task_rename(0);    return 0;   }   `

## 3. 使用 rawtracepoint 样例

### 3.1 libbpf 库 （基于 CO-RE）

系统中对应的 `task_rename` 跟踪点为  `tracepoint:task:task_rename` ，跟踪点格式定义如下：

`$ cat /sys/kernel/debug/tracing/events/task/task_rename/format   name: task_rename   ID: 131   format:    field:unsigned short common_type; offset:0; size:2; signed:0;    field:unsigned char common_flags; offset:2; size:1; signed:0;    field:unsigned char common_preempt_count; offset:3; size:1; signed:0;    field:int common_pid; offset:4; size:4; signed:1;     # 参数开始    field:pid_t pid; offset:8; size:4; signed:1;    field:char oldcomm[16]; offset:12; size:16; signed:0;    field:char newcomm[16]; offset:28; size:16; signed:0;    field:short oom_score_adj; offset:44; size:2; signed:1;      print fmt: "pid=%d oldcomm=%s newcomm=%s oom_score_adj=%hd", REC->pid, REC->oldcomm, REC->newcomm, REC->oom_score_adj   `

我们可以通过 libbpf 库在程序中使用结构，编写的代码如下所示：

`/* from: vmlinux.h   struct trace_entry {           short unsigned int type;           unsigned char flags;           unsigned char preempt_count;           int pid;   };      struct trace_event_raw_task_rename {           struct trace_entry ent;           pid_t pid;           char oldcomm[16];           char newcomm[16];           short int oom_score_adj;           char __data[0];   };   */      SEC("tracepoint/task/task_rename")   int prog(struct trace_event_raw_task_rename *ctx)   {     bpf_printk("task_rename -> pid %d, oldcomm %s, newcomm %s, oom %d",           ctx->pid,               ctx->oldcomm,               ctx->newcomm,               ctx->oom_score_adj );       return 0;   }   `

如果使用  `rawtracepoint` 的方式，则是将 `__set_task_comm(struct task_struct *tsk, const char *buf, bool exec)` 的参数依次压入 `bpf_raw_tracepoint_args` 结构中，args[0]  为参数 `struct task_struct *tsk`, args[1] 为 `const char *buf`，这里代表重命名的 `comm_name`，其他参数依次类推。

`bpf_raw_tracepoint_args` 参数结构如下：

`struct bpf_raw_tracepoint_args {       __u64 args[0];   };   `

在使用 raw_tracepoint 方式跟踪的代码编写如下：

`SEC("raw_tracepoint/task_rename")   int rt_prog(struct bpf_raw_tracepoint_args *ctx)   {       // void __set_task_comm(struct task_struct *tsk, const char *buf, bool exec);       struct task_struct *tsk = (struct task_struct *) ctx->args[0];       u32 pid;       u16 oom_score_adj;       char old_name[TASK_COMM_LEN] = {};       char new_name[TASK_COMM_LEN] = {};             pid = BPF_CORE_READ(tsk, pid);       // BPF_CORE_READ_INTO(&old_name, tsk, comm);       bpf_core_read(&old_name, sizeof(old_name), &tsk->comm);       bpf_core_read(&new_name, sizeof(new_name), (void *)ctx->args[1]);       oom_score_adj = BPF_CORE_READ(tsk, signal, oom_score_adj);          bpf_printk("task_rename:rt -> pid %d, oldcomm %s, newcomm %s, oom %d",                   pid,                   old_name,                   new_name,                   oom_score_adj);       return 0;   }   `

### 3.2 bpftrace

bpftrace 在 0.19 版本开始支持 rawtracepoint[6]。可以使用 bpftrace -l 查看，程序类型缩写为 rt[7]，参数类型为 arg0, arg1..

`$ bpftrace --version   bpftrace v0.20.0   `

bpftrace 通过  `tracepoint:task:task_rename` 方式跟踪：

`# bpftrace -e 'tracepoint:task:task_rename { printf("enter t:task:task_rename %s, pid %d, oldcommn %s, newcomm %s, oom 0x%x\n", comm, args->pid, args->oldcomm, args->newcomm, args->oom_score_adj); }'   $ sudo bpftrace -e 'tracepoint:task:task_rename    {    printf("enter t:task:task_rename %s,          pid %d, oldcomm %s, newcomm %s, oom 0x%x\n",           comm,          args->pid,          args->oldcomm,          args->newcomm         args->oom_score_adj); }'      Attaching 1 probe..   enter t:task:task_rename x11vnc, pid 3774, oldcommn x11vnc, newcomm x11vnc, oom 0x0   `

bpftrace 通过  `rawtracepoint:task_rename` 方式跟踪：

`# rename.bt   rawtracepoint:task_rename   {    $task = (struct task_struct *)arg0;    $pid = $task->pid;    $oom_score_adj = $task->signal->oom_score_adj;       printf("enter rt:task:task_rename %s, pid %d, oldcommn %s, newcomm %s, oom 0x%x\n",         comm,          $pid,          $task->comm,          str(arg1),          $oom_score_adj);   }   `

参考资料

[1]

bpf: introduce BPF_RAW_TRACEPOINT: _https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=c4f6699dfcb8558d138fe838f741b2c10f416cf9_

[2]

kprobe 和 tracepoint 跟踪都具有性能提升: _https://lwn.net/Articles/749972/_

[3]

rawtracepoint 的方式: _https://github.com/aquasecurity/tracee/blob/main/pkg/ebpf/c/tracee.bpf.c_

[4]

支持 rawtracepoint: _https://github.com/iovisor/bpftrace/pull/2588_

[5]

rt: _https://github.com/iovisor/bpftrace/pull/2542_

[6]

支持 rawtracepoint: _https://github.com/iovisor/bpftrace/pull/2588_

[7]

rt: _https://github.com/iovisor/bpftrace/pull/2542_

[8]

The art of writing eBPF programs: a primer: _https://sysdig.com/blog/the-art-of-writing-ebpf-programs-a-primer/_

[9]

rawtracepoint机制介绍: _https://blog.spoock.com/2023/08/17/eBPF-rawtracepoint/_

[10]

ebpf/libbpf 程序使用 raw tracepoint 的常见问题: _https://mozillazg.com/2022/05/ebpf-libbpf-raw-tracepoint-common-questions.html_

[11]

BPF程序tracepoint 和 raw_tracepoint的参数传递: _https://zhuanlan.zhihu.com/p/660840503?utm_id=0_

阅读原文

阅读 957

​

写留言

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/SeWfibBcBT0EibtIWVNvshnuWMN1AoJw3poFIsbpaIVyZibCCqwBUR21rcDfrQgoqYzaYNdS14IIXzvmzvibdDa5Rw/300?wx_fmt=png&wxfrom=18)

Linux内核之旅

5693

写留言

写留言

**留言**

暂无留言