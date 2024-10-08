Original 微信号szyhb1981 Linux驿站
 _2021年11月19日 19:09_

我有一个疑问：打上实时补丁的Linux内核是软实时内核还是硬实时内核？我找人问，有人说是软实时内核，有人说是硬实时内核。我感觉很混乱，没有得到一个准确的答案。通过和内核专家保罗·E·麦肯尼（Paul E. McKenney）交流，我得到一个准确的答案。  

  

实时有两种分类方法，一种分类是分为软实时和硬实时，另一种分类是分为3种，如下。

(1) **软实时（soft real time）**。应用程序的目标是维持一个服务质量，例如视频帧的速率。错过一些期限通常意味着服务质量降低，但不会对人或事物造成任何安全攸关的后果。然而，这需要确保可接受的用户体验。

(2) **准实时（firm real time）**。和软实时相似，但在这种情况中错过期限会使输出无效。超过期限后执行的结果没有价值，必须丢弃。

(3) **硬实时（hard real time）**。最严格的类别，不能因为任何原因错过期限，因为它可能导致不想要的后果。保证硬实时约束通常需要整个系统的形式化证明（formal proofs，另一种说法是“formal verification”，即形式化验证）和经典的代码分析。这一类为典型的安全攸关场景。

  

在现实世界中，一个实时系统的规格需要包含对环境的约束、对工作负载的约束和对应用自身的约束。

对环境的约束可能指定允许的工作温度、空气质量、电磁辐射的级别和类型等等。

对工作负载的约束。负载过重可能会阻止实时系统满足期限，例如中断太多导致没有足够的处理器带宽来处理实时应用，通常的解决方法是给实时应用分配一个专用核，把中断和普通进程绑定到其它核。还可能因为排队效应使响应时间变长，所以实时系统可能会为实时进程保留一部分处理器带宽，例如有些系统有80%的空闲时间。

对应用自身的约束。实时应用只能执行那些可以合理提供有界限的延迟的操作，其它操作要么转移到应用程序的非实时部分，要么完全放弃。例如实时应用不能读或写文件，因为可能需要访问存储设备，访问存储设备的延迟时间比较长。实时应用也不能动态申请内存，因为可能触发页错误异常。

  

假设一个特定的实时系统，响应时间小于或等于100微秒的概率是100%，响应时间小于或等于90微秒的概率是99%，那么当期限是100微秒的时候它是硬实时系统，当期限是90微秒的时候它是软实时系统。

打上实时补丁的Linux内核是硬实时内核吗？这个问题没有意义。硬实时响应是整个系统的属性，而不仅仅是软件的属性。我们应该说：打上实时补丁的Linux内核，把延迟时间减小到10~100微秒，可以用来构造一个期限是10~100微秒的硬实时系统。

实时Linux内核的使用手册（HOWTO setup Linux with PREEMPT_RT properly, https://wiki.linuxfoundation.org/realtime/documentation/howto/applications/preemptrt_setup）说：“Linux in itself is not real time capable. With the additional PREEMPT_RT patch it gains real-time capabilities.”这句话不准确。应该说：打上实时补丁的Linux内核提供了一种能力，可以构造一个期限是10~100微秒的硬实时系统。

“Linux内核主线只支持软实时，Linux内核的实时内核分支支持硬实时”这种说法是错误的，两者的主要区别是实时内核分支减小了延迟时间，可以用来构造期限更小的实时系统。

  

怎么证明一个实时系统是硬实时系统？

按照实时系统的理论，要证明一个实时系统是硬实时系统，必须对整个系统做形式化证明，证明延迟时间是有界限的。Linux内核是作为通用操作系统设计的，代码太多，实现很复杂，很难对整个系统做形式化证明。

现实和理论存在差异。事实上，到2021年为止，大多数现实世界的实时系统都使用验收测试，而不是形式化证明。验收测试持续24小时或更长的时间，而且使用的负载比实际负载更重，一些测试还检查了系统承受物理损坏的能力。如果在验收测试中实时系统的响应时间小于或等于期限的概率是100%，那么我们就认可这个实时系统是硬实时系统。所以说实时Linux内核可以用来实现硬实时系统。

测试不是完美的，因为有些竞争状态（race conditions）发生的概率可能极低，所以不能保证测试测出了最坏情况的延迟时间。形式化证明可以作为一个有价值的补充，但是它也不是完美的，它可能基于错误的假设，或者自身有缺陷。

安全攸关的场景要求必须对整个系统做形式化证明。目前实时Linux内核不适合安全攸关的场景。红帽子（Red Hat）公司的丹尼尔·布里斯托（Daniel Bristot）正在研究实时Linux内核的形式化证明，在个人主页“https://bristot.me/publications/”发表了一些论文，我们可以看到他取得一些进展，提出了高效的基于自动机的运行时验证方法。

  

参考文档如下。

(1)HOWTO setup Linux with PREEMPT_RT properly, https://wiki.linuxfoundation.org/realtime/documentation/howto/applications/preemptrt_setup

(2)The real-time linux kernel: A survey on Preempt_RT, https://www.researchgate.net/publication/331290349_The_real-time_linux_kernel_A_survey_on_Preempt_RT

(3)Making Linux do Hard (Real-)Time, https://elinux.org/images/0/00/Elc2013_Roman.pdf

(4)《Is Parallel Programming Hard, And, If So, What Can You Do About It?》14.3 Parallel Real-Time Computing, https://cdn.kernel.org/pub/linux/kernel/people/paulmck/perfbook/perfbook.html

(5)《Is Parallel Programming Hard, And, If So, What Can You Do About It?》12 Formal Verification, https://cdn.kernel.org/pub/linux/kernel/people/paulmck/perfbook/perfbook.html

(6)丹尼尔·布里斯托（Daniel Bristot）的个人主页“https://bristot.me/publications/”

实时内核2

Linux内核31

实时内核 · 目录

下一篇实时Linux内核的实现

Reads 614

​