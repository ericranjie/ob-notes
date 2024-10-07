原创 极客重生 极客重生
 _2022年03月21日 12:39_

自 3.15 以来，您可能一直在关注内核社区中扩展的 Berkeley 包过滤器 (eBPF) 的开发，或者您可能仍然将 Berkeley 包过滤器与 Van Jacobson 在 1992 年所做的工作联系在一起。您可能已经使用 BPF 和 tcpdump 多年，或者您可能已经开始在您的数据平面中使用！本文从网络性能的角度描述其关键发展，以及为什么现在这对网络运营商、系统管理员和企业解决方案提供商变得重要，就像它自成立以来就一直适用于运营商的大型数据中心和云计算网络。
#### **BPF 或 eBPF——有什么区别？**
##### **虚拟机**

从根本上说，eBPF 仍然是 BPF：它是一个小型虚拟机，运行从用户空间注入并附加到内核中特定挂钩的程序。它可以对网络数据包进行分类和操作。多年来，它一直在 Linux 上用于过滤数据包并避免对用户空间进行昂贵的复制，例如使用 tcpdump。然而，虚拟机的范围在过去几年中已经发生了翻天覆地的变化。

![](https://mmbiz.qpic.cn/mmbiz_png/cYSwmJQric6m8q6ldrtKZpiaDtHg9QicdcxCfTup7sswSPrsrlLfdphTqz3vx5Ly5SiagLQw9gFKkG3tjJC3Qx74BA/640?wx_fmt=png&wxfrom=13&tp=wxpic)

Classic BPF (cBPF)，旧版本，由一个 32 位宽的累加器、一个也可以在指令中使用的 32 位宽的“X”寄存器和 16 个用作暂存存储器的 32 位寄存器组成. 这显然导致了一些关键限制。顾名思义，经典的伯克利包过滤器主要限于（无状态）包过滤。任何更复杂的事情都将在其他子系统中完成。

eBPF通过使用扩展的寄存器和指令集、添加映射（没有任何大小限制的键/值存储）、512 字节堆栈、更复杂的查找、帮助程序，显着扩展了 BPF 的用例集可从程序内部调用的函数，以及链接多个程序的可能性。现在可以进行状态处理，与用户空间程序的动态交互也是如此。由于这种改进的灵活性，使用 eBPF 处理的数据包的分类级别和可能的交互范围已大大扩展。

但新功能不能以牺牲安全为代价。为确保正确行使 VM 增加的责任，内核中实现的验证程序已被修订和合并。这个验证器检查代码中的任何循环（这可能导致可能的无限循环，从而挂起内核）和任何不安全的内存访问。它拒绝任何不符合安全标准的程序。每次用户尝试注入程序时，都会在实时系统上执行此步骤，然后将 BPF 字节码 JITed 到所选平台的本机汇编指令中。
![[Pasted image 20240920201607.png]]
图 2. 主机上 eBPF 程序的编译流程。部分支持的 CPU 架构未显示。

为了允许在 eBPF 的限制内难以实现或优化的任何关键功能，有许多帮助程序旨在协助执行过程，例如地图查找或随机数的生成。
##### ****钩子 - 数据包在哪里分类？****

由于其灵活性和实用性，eBPF 的钩子数量正在激增。但是，我们将重点关注数据路径低端的那些。这里的关键区别在于 eBPF 在驱动程序空间中添加了一个额外的钩子。此挂钩称为 eXpress DataPath，或 XDP。这允许用户在将 skb（套接字缓冲区）元数据结构添加到数据包之前丢弃、反射或重定向数据包。性能可以提高约 4-5 倍。
![[Pasted image 20240920201613.png]]
图 3. 高性能网络相关 eBPF 钩子与简单用例的性能比较
### **将 eBPF 卸载到 NFP**
![[Pasted image 20240920201619.png]]
图 4. 包含 NFP JIT 的编译流程（未显示一些支持的 CPU 架构）

这可能的关键原因是因为 BPF 机器映射到我们在 NFP 上的流处理内核的能力非常好，这意味着运行在 15-25W 之间的基于 NFP 的 Agilio CX SmartNIC 可以从主机卸载大量处理. 在下面的负载平衡示例中，NFP 处理的数据包数量与来自主机的近 12 个 x86 内核的总和相同，由于 PCIe 带宽限制，主机在物理上无法处理数量（使用的内核：Intel Xeon CPU E5-2630 v4 @ 2.20GHz）。
![[Pasted image 20240920201627.png]]
图 5. NFP 和 x86 CPU (E5-2630 v4) 上示例负载均衡器的性能对比

性能是使用 BPF 硬件卸载是编程 SmartNIC 的正确技术的主要原因之一。但这不是唯一的：让我们回顾一下其他一些激励措施。

1. 灵活性：BPF 在主机上提供的主要优势之一是能够即时重新加载程序。这可以在运行的数据中心中动态替换程序。否则可能是树外内核代码或插入到其他一些不太灵活的子系统中的代码现在可以轻松加载或卸载。由于不需要重新启动系统的错误，这为数据中心提供了显着优势：相反，只需重新加载调整后的程序即可。

这个模型现在也可以扩展到卸载。用户能够在流量运行时在 NFP 上动态加载、卸载、重新加载程序。这种在运行时动态重写固件提供了一个强大的工具，可以被动地使用 NFP 的灵活性和性能。

2. 延迟：通过卸载 eBPF，延迟显着减少，因为数据包不必跨越 PCIe 边界。这可以改善负载平衡或 NAT 用例的网络性能。请注意，通过避免 PCIe 边界，在 DDoS 预防案例中还有一个显着的好处，因为数据包不再跨越边界，否则可能会在构造良好的 DDoS 攻击下形成瓶颈。
![[Pasted image 20240920201635.png]]
图 6. 驱动程序中卸载 XDP 与 XDP 的延迟，注意使用卸载时延迟的一致性

3. 用于对 SmartNIC 数据路径进行编程的接口：通过能够使用 eBPF 对 SmartNIC 进行编程，这意味着非常容易实现诸如速率限制、数据包过滤、不良行为缓解或其他传统 NIC 必须在芯片中实现的特性等特性. 这可以针对最终用户的特定用例进行定制。
##### **好的 - 这一切都很棒 - 但我如何实际使用它？**

首先要做的是将内核更新到 4.16 或更高版本。我会推荐 4.17（撰写本文时的开发版本）或更高版本，以利用尽可能多的功能。请参阅用户指南以获取最新版本的功能，以及如何使用它们的示例。

此处不赘述，还可以注意到，与 eBPF 工作流相关的工具正在开发中，并且已经在遗留 cBPF 版本方面得到了很大的改进。用户现在通常会用 C 编写程序，并使用 clang-LLVM 提供的后端将它们编译成 eBPF 字节码。也可以使用其他语言，包括 Go、Rust 或 Lua。对 eBPF 架构的支持被添加到传统工具中：llvm-objdump 可以以人类可读的格式转储 eBPF 字节码，llvm-mc 可以用作 eBPF 汇编器，strace 可以跟踪对 bpf() 系统调用的调用。其他工具的一些工作仍在进行中：binutils 反汇编器应该很快支持 NFP 微码，而 valgrind 即将获得对系统调用的支持。还创建了新工具：特别是 bpftool。
##### **bpfilter**

对于企业系统管理员或 IT 架构师来说，目前仍然悬而未决的一个关键问题是，这如何适用于拥有经过多年完善和维护且基于 iptables 的设置的最终用户。更改此设置的风险在于，很明显，某些事情可能难以接受，必须修改编排层，应该构建新的 API，等等。要解决这个问题，请进入提议的 bpfilter 项目。正如昆汀今年早些时候所写：

_“这项技术是针对 Linux 中 iptables 防火墙的基于 eBPF 的新后端的提议。在撰写本文时，它还处于非常早期的阶段：它是由 David Miller（网络系统的维护者）、Alexei Starovoitov 和 Daniel 于 2018 年 2 月中旬左右在 Linux netdev 邮件列表上作为 RFC（征求意见稿）提交的Borkmann（内核中 BPF 部分的维护者）。所以，请记住，后面的所有细节都可能改变，或者根本无法到达内核！_   

_从技术上讲，用于配置防火墙的 iptables 二进制文件将保持不变，而内核中的 xtables 部分可以被一组新的命令透明地替换，这些命令需要 BPF 子系统将防火墙规则转换为 eBPF 程序。然后可以将该程序附加到与网络相关的内核挂钩之一，例如在流量控制接口 (TC) 或驱动程序级别 (XDP) 上。规则转换将发生在一种新型内核模块中，该模块介于传统模块和普通 ELF 用户空间二进制文件之间。运行在具有完全权限但不能直接访问内核的特殊线程中，因此提供的攻击面较小，这种特殊类型的模块将能够直接与 BPF 子系统通信（主要通过系统调用）。同时，使用标准用户空间工具来开发、调试甚至模糊它仍然非常容易！除了这个新的模块对象之外，bpfilter 方法的好处还有很多。借助 eBPF 验证程序，预计会提高安全性。重用 BPF 子系统可能会使该组件的维护比传统的 xtables 更容易，并且可能会在以后提供与同样依赖 BPF 的内核其他组件的集成。当然，利用即时 (JIT) 编译，或可能的程序硬件卸载，将大大提高性能！” 重用 BPF 子系统可能会使该组件的维护比传统的 xtables 更容易，并且可能会在以后提供与同样依赖 BPF 的内核其他组件的集成。当然，利用即时 (JIT) 编译，或可能的程序硬件卸载，将大大提高性能！”_   

正在开发 bpfilter 作为该问题的解决方案，它允许最终用户无缝迁移到这种新的高性能范例。只需看下面：比较内核和一系列简单的 iptables 规则的卸载到 NFP 与 iptables (netfilter) 后端、较新的 nftables、主机上的 bpfilter 并卸载到 SmartNIC ，下图清楚地显示了性能区别所在.  

![[Pasted image 20240920201654.png]]

图 7. bpfilter 与旧 iptables 实现的性能比较
### **概括**

因此，我们有它。就目前而言，内核社区内正在产生的是一个巨大的网络转变。eBPF 是一个强大的工具，它为内核带来了可编程性。它可以处理**拥塞控制（TCP-BPF）、跟踪（kprobes、tracepoints）和高性能网络（XDP、cls_bpf）**。由于它在社区中的成功，可能会出现其他用例。除此之外，这种过渡一直延伸到最终用户，他们很快就能无缝地离开旧的 iptables 后端，转而使用更新的、更高效的基于 XDP 的后端——使用与今天相同的工具。特别是，这将允许直接的硬件卸载，并在用户迁移到 10G 及以上网络时提供必要的灵活性。

- END -

---

  

**看完一键三连********在看********，**转发****，********点赞****

**是对文章最大的赞赏，极客重生感谢你****![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)**

  

推荐阅读

  

[

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

定个目标｜建立自己的技术知识体系







](https://mp.weixin.qq.com/s?__biz=MzkyMTIzMTkzNA==&mid=2247570933&idx=1&sn=04bf666a7a402ed65b83020dc28dd543&chksm=c18522a4f6f2abb27ca1d09eaf646cc7eca2feea9c7f39618d3048b87634695955a8831992a9&scene=21#wechat_redirect)

  

[

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

大厂后台开发基本功修炼路线和经典资料







](https://mp.weixin.qq.com/s?__biz=MzkyMTIzMTkzNA==&mid=2247566986&idx=1&sn=40c3846a5a328ba18b73a16d329b756f&chksm=c18531dbf6f2b8cd4a93bd9aa5cad4020daf8da7a54156f754f5c2b403c58b02b40a79ed79f4&scene=21#wechat_redirect)

[

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

个人学习方法分享







](https://mp.weixin.qq.com/s?__biz=MzkyMTIzMTkzNA==&mid=2247513515&idx=2&sn=e2a42c2e110f85d451138b2d7c65491e&chksm=c18442faf6f3cbecfd885fa01b76cfaf52dd314c533b099959856a05604bb5f6afd29c5c7e26&scene=21#wechat_redirect)

[

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

eBPF在大厂的应用







](https://mp.weixin.qq.com/s?__biz=MzkyMTIzMTkzNA==&mid=2247575154&idx=1&sn=d918621452b74db082ff80727dc474c1&chksm=c1855123f6f2d83535a6b95fa57d8f3236121ea2f27a4c3138f124fc97a31097d9e5ca717440&scene=21#wechat_redirect)

  

[

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

万字长文|深入理解XDP全景指南







](https://mp.weixin.qq.com/s?__biz=MzkyMTIzMTkzNA==&mid=2247570640&idx=1&sn=54c01e177fff74b6ece261ddbf671d91&chksm=c1852381f6f2aa973f1d1cd3addfa4d7ba98fff69146f6ea8a16cccdc176eb2868dcd15134a7&scene=21#wechat_redirect)

  

[

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

Linux网络新技术基石 |eBPF and XDP







](https://mp.weixin.qq.com/s?__biz=MzkyMTIzMTkzNA==&mid=2247511570&idx=1&sn=18d5f1045b7ed6e27e1c8fb36302c3f9&chksm=c1845943f6f3d055c2533d580acb2d4258daf2a84ffba30e2cc6376b3ffdb1687773ed4c19a3&scene=21#wechat_redirect)

  

你好，这里是极客重生，我是阿荣，大家都叫我荣哥，从华为->外企->到互联网大厂，目前是大厂资深工程师，多次获得五星员工，多年职场经验，技术扎实，专业后端开发和后台架构设计，热爱底层技术，丰富的实战经验，分享技术的本质原理，希望帮助更多人蜕变重生，拿BAT大厂offer，培养高级工程师能力，成为技术专家，实现高薪梦想，期待你的关注！[**点击蓝字查看我的成长之路**。](http://mp.weixin.qq.com/s?__biz=MzkyMTIzMTkzNA==&mid=2247564006&idx=1&sn=8c88b0d7222ce0e310a012e90bff961f&chksm=c1850db7f6f284a133be27ad27ecd965c3c942583f69bd1a26c34ebf179b4cd6ceb3d7e787b4&scene=21#wechat_redirect)

  

校招/社招/简历/面试技巧/大厂技术栈分析/后端开发进阶/优秀开源项目/直播分享/技术视野/实战高手等, [极客星球](http://mp.weixin.qq.com/s?__biz=MzkyMTIzMTkzNA==&mid=2247568828&idx=1&sn=ec346110ca42539f5d797f94eee57e9a&chksm=c1853aedf6f2b3fbf80600f4f71059ce2eddcb7c3e900953014764a3d16d2fd77f8190303898&scene=21#wechat_redirect)希望成为最有技术价值星球，尽最大努力为星球的同学提供面试，跳槽，技术成长帮助！详情查看->[极客星球](http://mp.weixin.qq.com/s?__biz=MzkyMTIzMTkzNA==&mid=2247568828&idx=1&sn=ec346110ca42539f5d797f94eee57e9a&chksm=c1853aedf6f2b3fbf80600f4f71059ce2eddcb7c3e900953014764a3d16d2fd77f8190303898&scene=21#wechat_redirect)  

  

![](https://res.wx.qq.com/t/fed_upload/b39ef69e-c4d6-4169-8612-5f00a84860e7/wx-avatar-default.svg)

公众号

                                                                求点赞，在看，分享三连![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

深入理解网络52

深入理解Linux系统51

深入理解网络 · 目录

上一篇eBPF在大厂的应用下一篇同步/异步，阻塞/非阻塞概念深度解析

阅读原文

阅读 2716

​