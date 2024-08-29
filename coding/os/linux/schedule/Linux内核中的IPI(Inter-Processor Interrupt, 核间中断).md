
原创 TrickBoy CodeTrap

 _2024年03月30日 00:00_ _江苏_

在Linux内核中，IPI(Inter-Processor Interrupt, 核间中断)是在多处理器系统下，一种常用的CPU间的通信机制。  

该机制允许一个CPU向其余一个或多个CPU发送中断，从而触发目标CPU上相应的处理函数。

本文将从**为什么要使用IPI**，**如何使用IPI**以及**IPI的实现原理**来分析Linux中的IPI机制。  

**为什么要使用IPI**  

昨天，有个朋友问了我这样一个问题：  

他有个任务，需要使用一个内核模块定时去获取某个CPU上的curr进程信息，应该怎么去实现代码呢？  

我的**第一个想法**是：直接去遍历进程链表，依据task_struct的on_cpu字段去进行判断，不就OK了吗？  

代码大致是这样：

```
typedef struct {
```

输出结果如下：  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

1. 为什么只有3个CPU上的curr信息？  

因为0号进程（idle进程）不在进程链表中。其余CPU都处于idle状态，因此无法通过进程链表找到它们的curr进程信息。  

2. 为什么判断依据是task_struct的on_cpu成员，而非是task->state == TASK_RUNNING呢？

因为task_struct在加入rq之后，state都是TASK_RUNNING的，无法作为进程是否在CPU上运行的证据。而on_cpu字段的更新是在context_switch（实际进程切换的前后）去设置的。PS：进程切换前去设置next（task_struct）的on_cpu为1，进程切换后去设置prev（task_struct）的on_cpu为0。

但使用这个方法明显存在一定的问题：

1. 每次都需要完整地去遍历一遍进程链表，即该任务的性能表现是与系统中的进程数量强相关的。

2. on_cpu字段的更新机制可能会导致出现在遍历时发现一个CPU上有两个task_struct的on_cpu字段皆为1的情况。

因此，**第二个想法**是去找一找内核里面现有的API可以做到这个事情吗？（指定CPU，返回CPU上当前运行的进程）

的确是有这样的函数：  

```
DECLARE_PER_CPU_SHARED_ALIGNED(struct rq, runqueues);
```

但这些函数的定义是在/kernel/sched/sched.h中的，没有办法被内核模块引用到。  

**第三个想法**是一个“骚操作”，写一个kprobe函数，默认是disable的，当需要去获得信息的时候再enable这个kprobe，获取到信息后再enable它。  

代码大致是这样：  

```
static atomic_t cpu_get_num;
```

输出结果如下：  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

但这种实现的问题是：

1. 注册完kprobe函数后，实际内核text中挂载函数的地址的指令已经被替换。虽然刚开始是enable的，但仍然会有跳转的开销。

2. 获得各个CPU上进程的信息的时机依赖于挂载点的选择，如果目标函数很久都没有触发，则无疑会增加等待的时间。

这时候突然想到，kprobe方式是被动等待其余CPU触发int3中断，这种被动的方式是带来这么多问题的根本原因。所以，与其被动挨打，不如主动出击。我完全可以使用主动触发其余CPU的中断，来让它们将自己的curr信息进行报告。

因此，IPI不失为一个好选择。  

**如何使用IPI**  

IPI机制在很多驱动以及内核逻辑的实现上被应用很多。比如在DVFS的驱动中，一个CPU上的任务要去调整其余CPU的频率，就会使用IPI机制。又或者一个高优先级的任务加入运行队列，这时候需要使用IPI机制来通知对应CPU上的任务下处理器。  

依旧按照前面的需求来完成代码：

```
static void do_get_cpu_current(void *args)
```

输出结果如下：  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

这里忽略掉当前内核模块所在的CPU，输出了其余CPU上的curr信息。向其它CPU发送IPI中断使用了smp_call_function这个函数。

这个函数的的定义如下：

```
/**
```

smp_call_function()：在除自身之外的所有CPU上运行一个函数。

func：目标函数  

info：目标函数的参数  

wait：是否等待其余CPU上的任务完成  

注意：只能用于进程上下文。  

当然除了这个API外，还有其余的API，譬如：

smp_call_function_single——在指定某个CPU上运行一个函数。

smp_call_function_any——给任意给定的CPU上运行一个函数。

除了前面提到的这些同步接口，还有异步接口（注册函数后，直接返回，可以用于非进程上下文），比如：

smp_call_function_single_async——在指定某个CPU上运行一个异步函数。

**IPI的实现原理**

知道了IPI有哪些接口，现在分析一下IPI的实现原理。

直接看一次广播给多个CPU的函数的实现：smp_call_function_many。  

```
/**
```

接着看smp_call_function_many_cond函数：

```
static void smp_call_function_many_cond(const struct cpumask *mask,
```

实际调用的核心部分是这一部分：  

```
    if (nr_cpus == 1)
```

在这之前的主要工作是：

1. 设置当前CPU的per_cpu变量cfd->cpumask为目标CPU的mask

2. 依次遍历目标CPU的mask，找到对应CPU的per_cpu变量cfd->pcpu->csd，更新csd的信息存放目标函数和参数，之后将当前csd加入到对应CPU的call_single_queue队列中

至于，wait的实现，则是在下发任务前csd_lock()函数中设置csd->node.u_flags的CSD_FLAG_LOCK标记位。  

```
csd->node.u_flags |= CSD_FLAG_LOCK;
```

在任务下发之后，去检查对应的CSD_FLAG_LOCK是否被清理掉。  

```
smp_cond_load_acquire(&csd->node.u_flags, !(VAL & CSD_FLAG_LOCK));
```

那现在完成了函数的注册，应该如何去触发中断呢？

以x86架构下的广播IPI为例子，其调用是：

```
arch_send_call_function_ipi_mask->
```

这里选择看apic_numachip.c中的代码：  

```
  .send_IPI      = numachip_send_IPI_one,
```

可以看到，实现了以上这个IPI的接口。

```
static void numachip_send_IPI_mask(const struct cpumask *mask, int vector)
```

这里也没什么好看的了，就是写特定地址，触发中断。  

起码，我看到的这里的代码，没有实现广播写的功能，在最底层还是for循环去写。

在前面看到，发送自定义IPI的时候，中断vector是CALL_FUNCTION_VECTOR，其实通过IPI机制发送的不止这一种中断，譬如还有之前提到的提醒某个CPU要进行调度的IPI。

```
DEFINE_IDTENTRY_SYSVEC_SIMPLE(sysvec_reschedule_ipi)
```

这里看下CALL_FUNCTION_VECTOR的处理函数，generic_smp_call_function_interrupt()。  

```
generic_smp_call_function_interrupt()->
```

__flush_smp_call_function_queue函数也没有特别，就是在之前添加任务的call_single_queue队列中取出函数指针去执行，很标准的任务列表的代码。

**最后1个问题**  

为什么有些接口只可以在进程上下文使用？而那个异步接口却可以在中断上下文使用？

参见前面的代码（在进程上下文使用的接口）其做wait的接口是使用per_cpu的一个全局变量去进行任务下发。如果上一个任务卡住，那么现在进行任务下发时也会卡住。

那如果将wait设置为false呢？因为需要依靠这个全局变量去下发，在set之前，默认做了一次wait的操作。

而异步的接口的实现是自己识别到csd（输入）被占用立即退出，因此不会有wait导致卡死的问题出现：

```
int smp_call_function_single_async(int cpu, struct __call_single_data *csd)
```

  

我对IPI的理解也并不算深入。

但对开发者来说，这种手段在特定场景下的效果可能也蛮不错的![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)。

![](https://mmbiz.qlogo.cn/mmbiz_jpg/TibnEdoonBKJXf2P0sGG5JBWKUIMIjlib4YekFs5IhJyEluzwzPibd7nl2IZjn7nbhEcEov3x4IrWYAoWv6HNziaGQ/0?wx_fmt=jpeg)

TrickBoy

![赞赏二维码](https://mp.weixin.qq.com/s?__biz=Mzg3MDYzOTQyNg==&mid=2247483697&idx=1&sn=60a1f9aca4382ab2c25bf24b2d9e0cbb&chksm=ce8bf31cf9fc7a0a84599e80eed1c7e850a5fcbf499d29c9462594b56654d64e4eae5369e896&mpshare=1&scene=24&srcid=0330o9kaFJmqUWH4ovHYRBl2&sharer_shareinfo=1e9468b27cfc62d78ac5873f4c0ac31b&sharer_shareinfo_first=1e9468b27cfc62d78ac5873f4c0ac31b&key=daf9bdc5abc4e8d0ecfc8525ad5670367b54640c22c5212335774d14ec26e8b41d7d371002e5c352160c37f0cbb074a524496f8051a59640605d38d5f8ee0e83e1567653bbb67bbc20a914020a5670cfb3f20853f574fb21e97c320185bcddbdcb7cf2a00973f1dc619cbd127f6e77983672d3a506bae74404b2eb517f997abb&ascene=0&uin=MTEwNTU1MjgwMw%3D%3D&devicetype=Windows+11+x64&version=63090b19&lang=zh_CN&countrycode=CN&exportkey=n_ChQIAhIQM84yUEIQLWhsciz%2BFQgTyRLmAQIE97dBBAEAAAAAADQYNQXp7MMAAAAOpnltbLcz9gKNyK89dVj0F1jTdSOOQs6V7jebQpwauQXQkq6WUxCjp5eVophDew8Im%2Bi%2Bh1Abn007YWU3kYP8H0Y8VAEOIq9mCGozvmsVk0JpsRQDBuxxDj9HtPkGDCeRX%2FjXLroNZY7d0PiFVF4YRItRyKtxL5NlRTW%2BUg2azO9aZgB4035tsprfY5FG5J07DeFoYXmOHHbeNIKjmq7bQSn9B1ZJXWH%2B6nvKTfutlInscQMq6usXOTNFTYW5kKTiQITa0OqGyzYgKwtalCI2&acctmode=0&pass_ticket=%2BheWQ8duYrNY1ThuaO7lLUMoYbdW1Elm9lVQlYdUhsSiPER8TfEzdYBx3OVeaBWK&wx_header=1&fasttmpl_type=0&fasttmpl_fullversion=7350504-zh_CN-zip&fasttmpl_flag=1)喜欢作者

阅读 86

修改于2024年03月30日

​

发消息

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/Dz7JZlP0rjyQkC3MaOvFTPggGQhbvqy1ibY6AtwjYXC4w9kBXBXCouwL0q7W9TlzsZ9giarG2VId58QzffAeF3JQ/300?wx_fmt=png&wxfrom=18)

CodeTrap

117在看

发消息