# 

云原生技术爱好者社区

_2024年08月22日 08:22_ _广东_

**_1_**

**概述**

用 Rust 编写的 Linux 调度程序的主要瓶颈 scx_rustland 是内核和用户空间之间的通信。

BPF_MAP_TYPE_RINGBUF 即使使用环形缓冲区（ / ）可以大大改善通信本身，但 BPF_MAP_TYPE_USER_RINGBUF 多级队列不可避免地会导致调度程序以不太节省工作量的方式运行。

这具有双重效果，使得基于 vruntime 的调度策略更有效（能够积累更多任务并优先处理那些对延迟要求更严格的任务），但任务排队也有缺点，即不能充分利用 CPU 的能力，可能会导致吞吐量方面的倒退。

为了减少这种“缓冲区膨胀”效应，我决定在 BPF 中完全重新实现 scx_rustland，摆脱用户空间开销，并将这个新的调度程序命名为 scx_bpfland。

**_2_**

**执行**

scx_bpfland 使用与分类交互式任务和常规任务相同的逻辑 scx_rustland。它根据每秒自愿上下文切换的平均次数来识别交互式任务，当此平均值超过移动阈值时，将其分类为交互式任务。此外，明确唤醒的任务会立即标记为交互式任务，并且除非其平均自愿上下文切换次数低于阈值，否则将保持这种状态。

每个 CPU 都有队列，用于直接将任务分派到空闲 CPU。如果所有 CPU 都处于繁忙状态，则将任务分派到全局优先级队列或全局常规队列（取决于任务被归类为“交互式”还是“常规”）。

然后，首先从每个 CPU 队列中执行任务，然后从优先级队列中执行任务，最后从常规队列中执行任务。这意味着常规任务可能会被交互式任务所取代，因此为了防止无限期的饥饿，scx_bpfland 有一个“饥饿时间阈值”参数（可通过命令行配置），当超过阈值时，该参数强制调度程序从常规队列中至少执行一个任务。

对于被归类为“交互式”的任务，如果在全局常规队列之前使用单独的队列，则可以以类似中断的方式立即处理“交互式”事件。

最后，scx_bpfland 根据优先级和共享队列中等待的任务数量，为任务分配一个可变的时间片：等待的任务越多，时间片越短。这确保在最大时间片期间内，所有排队的任务都有机会运行（取决于它们的优先级和“交互性”）。

**_3_**

**测试**

调度程序的逻辑非常简单，但在实践中被证明非常有效。当然，它并不是在所有可能的情况下都表现良好，但调度程序的整个目的是高度专业化，优先考虑延迟敏感的工作负载，而不是 CPU 密集型工作负载。

因此，scx_bpfland 不应将其视为默认 Linux 调度程序 (EEVDF) 的替代品，它总体上仍然是最佳选择。

然而，在某些特殊情况下，当延迟很重要时， scx_bpfland 可以提供改进的、一致的响应水平。

这次运行的 Phoronix 测试套件精选测试 主要旨在测量延迟和响应时间，显示了对所执行的交互任务进行积极优先排序的一些好处 scx_bpfland。

**_4_**

**结果**

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

与默认调度程序相比，PostgreSQL 和 Hackbench 都显示出显着的提升（PostgreSQL 的读/写延迟高达 39%），因为它们的工作负载纯粹受延迟限制。

FFMpeg 也显示出一些改进（实时流式传输提高 9%，上传配置文件提高 7%），主要是因为工作负载不仅仅是编码，还涉及消息传递（以生产者 - 消费者的方式）。

nginx 显示出 8.4% 的提升，这也是因为基准测试主要强调短暂连接（测量连接响应时间）。

与 EEVDF 相比， Apache 的性能下降了 9% scx_bpfland。

nginx 和 Apache 基准测试都依赖于 wrk，这是一个 HTTP 基准测试工具，具有多线程设计和通过 epoll 和 kqueue 可扩展的事件通知系统。

为了从调度程序的角度更好地理解正在发生的事情，我们可以查看在 30 秒的运行期间两种情况下客户端 (wrk) 的运行队列延迟（任务在调度程序队列中等待的时间）的分布：

```

```

```
[nginx - scx_bpfland]
```

```

```

```

```

首先让我们看一下样本量（通过 bpftrace 在每次任务切换时收集）。我们可以看到，Apache 的任务切换次数比 nginx 高得多（大约 10 倍）。

考虑到客户端是相同的（wrk），差异显然是在服务器上，实际上 Apache 以阻塞方式处理响应，而 nginx 以非阻塞方式处理响应。

top 的输出也证实了这一点，我们可以看到 nginx 使用的 CPU 比客户端 (wrk) 多得多，而在 Apache 的情况下则相反。

因此，在 Apache 场景中，客户端和服务器均被归类为“交互式” scx_bpfland，因为它们具有阻塞特性；而在 nginx 场景中，客户端 (wrk) 被归类为交互式，而服务器 (nginx) 被归类为 CPU 密集型，因此客户端的优先级更高。这有助于提高生产者/消费者管道的整体性能，在使用 scx_bpfland 时在基准测试中获得更好的分数。

从客户端运行队列延迟的分布也证实了这一点：使用 nginx 时，任务在调度程序队列中花费的时间较少 scx_bpfland，而在 Apache 场景中则相反。

显然，运行队列延迟只是一个指标，并不总是准确代表实际性能：任务可能会在调度程序的队列中花费更多时间，但仍能实现整体性能的提高，例如，如果生产者/消费者管道得到更好的优化。然而，在这种情况下，它似乎准确地解释了两个调度程序之间的性能差距。

那么，我们如何才能改进，scx_bpfland 以便在 Apache 场景中也能更好地工作呢？通过不要过于激进地缩减服务员函数中任务的可变时间片，可以缓解性能下降。

通过调整这个逻辑，我们也可以改善这个特定的场景，可能会退化其他场景，但这就是好处 sched_ext：允许快速实施和开展实验并实施专门的调度策略。

Apache 的回归表明它 scx_bpfland 总体上不是一个更好的调度程序，但它强调了一个事实，即可以利用它 sched_ext 来快速实现和测试专门的调度程序。

**_5_**

**结论**

总之， 使用 Rust 在用户空间中设计新的调度程序原型，然后在 BPF 中重新实现它们可以成为设计新的专用调度程序的有效工作流程。

此类技术提供的快速编辑/编译/测试周期 sched_ext 对于快速迭代这些原型非常有价值，使开发人员能够在更短的时间内获得有意义的结果。

scx_bpfland 是这种方法的一个实际示例，展示了如何将灵活的 Rust 用户空间环境中的初始开发有效地过渡到 BPF 以增强性能。

这个实验不仅强调了 sched_ext 和 eBPF 在实现高效和适应性调度程序开发方面的强大组合，而且还表明这 sched_ext 可能是实现 Linux 中可插入模块化调度的基本一步。

## 推荐

[运维技能大合集](<https://mp.weixin.qq.com/s?__biz=MzI3NTEwOTA4OQ==&mid=2649185045&idx=1&sn=e8a669ff8ea1b91ab79ae86cb3761f30&chksm=f27b15f0f7f305bd0908a0b8c54fc31da02d650225531a3e3a8aa592f67a88cc5bafae0918c4&mpshare=1&scene=24&srcid=08248NBiNFGsPB4PF9BbdTyv&sharer_shareinfo=425611c28badade47c8976230008c633&sharer_shareinfo_first=e2dd5afd9e9472b3a8fdfc523328544a&key=daf9bdc5abc4e8d03fb3f182ff654b48f32fa98f23f9934040f7729799e5d83edf2368a6047833988b37e62d0c3a0f9a8725c6d8c71ec0595dfa14ab51080dd67800bf5235796e86b63f5765c23062d0134a45419c4f41e4b80dd61b731b0f5b8fe349fd02a386c76693daf23c8c1d7ebfc8369068ba600a6cd939aea085df67&ascene=14&uin=MTEwNTU1MjgwMw%3D%3D&devicetype=iMac+MacBookAir10%2C1+OSX+OSX+14.6.1+build(23G93)&version=13080710&nettype=WIFI&lang=en&countrycode=CN&fontScale=100&exportkey=n_ChQIAhIQzyCOzuUf%2FTGBpQhy0kH0wBKUAgIE97dBBAEAAAAAAOd8EmT7%2BckAAAAOpnltbLcz9gKNyK89dVj0rG02lgpT52O012TvgoEJ1U%2FHzYIa%2FtGJ78qG9wzajL5ZV8xYiA2ySDQreFfuZzxgF25PECcQQ4L%2BpoYd4kbounb%2BADRV7X%2F3lXSzb4PTSlhX2dg6Ic9AM4Deuc582U5wJk09VEeIsI9uGRHB6ynGvTK8mr75VHejD7leeW4G9S3Z2HI884YSwqqn9MH5nixsx8bo79%2BM0e5mrzbw%2FMY2dwHCy8pDXVuZcpMSkVURi6fFjE%2BjVyEhUAynN3xR6Kdbn6W7tqFSt6OQSBuIJhzHYbuxwv1GNJCLRdOl6xUKp2VYS4OR6sRb%2FXrcyF059w%3D%3D&acctmode=0&pass_ticket=8%2BY%2BfEgUhrEBRq0jbdpBF2%2B4ZIn5hi3ct0vE%2Ba85w1gIUNVNiJbX5IRua%2F7B4Duj&wx_header=0>)

______________________________________________________________________

随手关注或者”在看“，诚挚感谢！

个人思考86

Kubernetes159

云原生技术62

个人思考 · 目录

上一篇背压(Back Pressure)原理与流量控制

Read more

Reads 235

​

Comment

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/hvZjCFh6diaQEgYWsPZBjiaUX30SgROIHsvKCFUpS894xnwNlNsAybG1IKsa7fMU7yPYYheKMZa1Ou6xSWX0YiaXg/300?wx_fmt=png&wxfrom=18)

云原生技术爱好者社区

5171

Comment

Comment

**Comment**

暂无留言
