原创 扫地僧666 IEEEE

_2021年11月27日 23:07_

**![图片](https://mmbiz.qpic.cn/mmbiz_jpg/W9DqKgFsc6ibRRBRwpAc6LV4p31Ik5mX2rAqcRyUdzic9mymwSPDIWzS7DJMKDzD76zYPiaP8nGkGzgy2Gf9iaZ3tA/640?wx_fmt=jpeg&wxfrom=13&tp=wxpic)**

在 BPF 出现之前，如果你想去做包过滤，你必须拷贝所有的包到用户空间，然后才能去过滤它们

-- Julia Evans

**eBPF 简介**

**![图片](https://mmbiz.qpic.cn/mmbiz_png/OdIoEOgFgUH7w4sb2zibRuPlewVLAf4EGGzHnMBHRAqm3tbYOM7pADvpxFg88dOP3sV5pZcYSoibI4iayaCibLvC6g/640?wx_fmt=png&wxfrom=13&tp=wxpic)**

扩展的Berkeley Packet Filter（eBPF）是一种内核虚拟机，可用于系统跟踪。eBPF使程序员能够编写在更安全和受限制的环境中在内核空间中执行的代码。

![图片](https://mmbiz.qpic.cn/mmbiz_png/akGXyic486nVedX8cKPlIaLbAvPhmo4MVpS1dsIKxOlMzHnIYYYFiarbZoCkq6XWZR8YibjfLkT7ic4y0c3ZxlLV9Q/640?wx_fmt=png&wxfrom=13&tp=wxpic)

2014 年初，Alexei Starovoitov 实现了 eBPF（extended Berkeley Packet Filter）。经过重新设计，eBPF 演进为一个通用执行引擎，可基于此开发性能分析工具、软件定义网络等诸多场景。eBPF 最早出现在 3.18 内核中，此后原来的 BPF 就被称为经典 BPF，缩写 cBPF（classic BPF），cBPF 现在已经基本废弃。现在，Linux 内核只运行 eBPF，内核会将加载的 cBPF 字节码透明地转换成 eBPF 再执行。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

### ebpf 主要有哪些功能

1. Tracing: kprobe/uprobe/tracepoint

1. Security: seccomp

1. Net: bpfilter, tc, sockmap, XDP

可以从这张图上看看不同版本的内核和其支持的功能。 !\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

ebpf架构

eBPF除了提供指令集，还提供了如map之类的数据结构来做键值对存储，辅助函数(helper function)来和内核进行交互，尾调用的方式来链式请求其他的BPF程序，安全加固原语以及一个伪文件系统的功能来追踪BPF程序中的对象(例如map, programs)

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

eBPF核心功能

eBPF的使用场景已经不局限于网络监控分析，可以基于eBPF开发性能分析、系统追踪、软件定义网络与安全等多种类型的应用。而能够保证这些场景实现的关键在于eBPF主要提供了三种核心功能：BPF钩子、BPF Map数据结构以及BPF辅助函数。具体功能介绍见表1。

_表1 eBPF三种核心功能介绍_

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

eBPF工作过程

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

eBPF 与内核模块对比

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**xdp**

XDP（eXpress Data Path）是基于 eBPF 实现的高性能、可编程的数据平面技术。基本的软件架构如下图所示：!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

XDP 位于网卡驱动层，当数据包经过 DMA 存放到 ring buffer 之后，分配 skb 之前，即可被 XDP 处理。数据包经过 XDP 之后，会有 4 种去向：

- XDP_DROP：丢包

- XDP_PASS：上送协议栈

- XDP_TX：从当前网卡发送出去

- XDP_REDIRECT：从其他网卡发送出去

  由于 XDP 位于整个 Linux 内核网络软件栈的底部，能够非常早地识别并丢弃攻击报文，具有很高的性能。这为我们改善 iptables/nftables 协议栈丢包的性能瓶颈，提供了非常棒的解决方案。

BPF 技术与 XDP（eXpress Data Path） 和 TC（Traffic Control） 组合可以实现功能更加强大的网络功能，更可为 SDN 软件定义网络提供基础支撑。XDP 只作用与网络包的 Ingress 层面，BPF 钩子位于网络驱动中尽可能早的位置，无需进行原始包的复制就可以实现最佳的数据包处理性能，挂载的 BPF 程序是运行过滤的理想选择，可用于丢弃恶意或非预期的流量、进行 DDOS 攻击保护等场景；而 TC Ingress 比 XDP 技术处于更高层次的位置，BPF 程序在 L3 层之前运行，可以访问到与数据包相关的大部分元数据，是本地节点处理的理想的地方，可以用于流量监控或者 L3/L4 的端点策略控制，同时配合 TC egress 则可实现对于容器环境下更高维度和级别的网络结构。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

应用案例

# 高性能ACL：ACL 控制平面负责创建 eBPF 的 Program、Map，注入 XDP 处理流程中。其中 eBPF 的 Program 存放报文匹配、丢包等处理逻辑，eBPF 的 Map 存放 ACL 规则。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

编程语言

对于早期的cBPF程序的编写，一般都是直接使用BPF指令集来编写程序。像tcpdump这类工具，提供的使用方法可以类似于高级语言的人性化的表达式的使用，但是其实还是一样的，只不过是让libcap进行解析了。BPF程序的编写难度是极高的。

后来，由于BPF的扩展，急需要一种高级语言来编写BPF程序，就出现了c语言的编程。通过c语言进行编写，然后，通过clang/llvm将c语言编译为BPF字节码，然后在注入到内核中。但是对于注入的方式，还是需要通过自己手动的方式才能够注入。

后来，就出现了BPF Compiler Collection（BCC），BCC 是一个 python 库，但是其中有很大一部分的实现是基于 C 和 C++的，python 只不过实现了对 BCC 应用层接口的封装而已。使用bcc的最大好处是，用户只需要关注BPF程序的设计，对于剩余的工作都不用管，包括编译、解析 ELF、加载 BPF 代码块以及创建 map 等等基本可以由 BCC 一力承担，无需多劳开发者费心。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

总结

ebpf 最大的用途目前应该还是在上面提到的几个方面：内核调试，安全，网络数据转发。比如目前 facebook 发布的 l4LB 和 DDoS 防护，Google 的用于追踪分析的 BPFd。

![](http://mmbiz.qpic.cn/mmbiz_png/nrgibEGYIoVJUicpguZaiajUVLb0EMHmIMknczOwIJQPx7QoStEXebjx0qmzdibHfxfXEg52o5bMMhtC56r3Q5rGJA/300?wx_fmt=png&wxfrom=19)

**IEEEE**

分享社会热点，矛盾焦点，技术新点，八卦笑点，除了这四点，还有六点 ......

222篇原创内容

公众号

![](https://mmbiz.qlogo.cn/mmbiz_jpg/QmYLMv8VLF2owib18kkHS7RmWHfBjQfXyoogHhp8zVt1AxibzOtrT87K9YIyNFPuiawdGLSqOr9abRXVjqLDydzvg/0?wx_fmt=jpeg)

扫地僧666

![赞赏二维码](https://mp.weixin.qq.com/s?__biz=MzU4Mzc0NTcwNw==&mid=2247495160&idx=1&sn=02cf930712f36905802cd48cbfc2aef6&chksm=fda6c43ccad14d2ac689a942c6e6b7a492f712f4a98feecbad86bdce7cfe5a637178e3263d50&mpshare=1&scene=24&srcid=1127tkoeNr3YR5GcsPQ2BwMU&sharer_sharetime=1638028145691&sharer_shareid=5fb9813bfe9ffc983435bfc8d8c5e9ca&key=daf9bdc5abc4e8d0f11cbe11704439b1e995ac7cb2bfda76f53e0d2a7978f0d21eb5ce7d9586f7ea1a026480cefe340d2ed9b7adacc36b17dc63b01d31f3cf4d2594ac4133271fc58d6aead08b47bb143e0b581f641998f1f1ccd05e9f21c31f461ef3f204b9caf9a2d3029a1c5c20ffe7e960935ad82d119ce473e952be9a5b&ascene=0&uin=MTEwNTU1MjgwMw%3D%3D&devicetype=Windows+11+x64&version=63090b19&lang=zh_CN&countrycode=CN&exportkey=n_ChQIAhIQTD5%2FfEIeR5yjxwbM0S7i9BLmAQIE97dBBAEAAAAAAEtVFhdOs88AAAAOpnltbLcz9gKNyK89dVj0zkN4bY6GimDfT9Pj2wcRWKudVpMiTax%2BUqzA6d6LPEjWEaqbM%2BsORYRvtYHSXWeg%2BnXi9OTFvwm%2BA3KpHxrM3C5XjLsMlpBsNZiaHArXT5Gk8m%2FVrY3fZdaeqMScbm7NnJsGVa4o3JtduOmzMGV51dZc1A8cyVNiJdyIWMSRM92YbXf%2ByXGn%2FM5GbGMQlR58NsL2ebhUUF8NBYE73i7efJAPxkztJvwwFcgMkCojrvHA6DA3LsJpiZ9Vtd0Ih2Qk&acctmode=0&pass_ticket=fuyL8dWWMQzmNgaIgYtO6yr9Q06CqCV14Q4UmoG0QQDh%2FVJ51LE1CbAkY1Thpn0D&wx_header=1&fasttmpl_type=0&fasttmpl_fullversion=7350504-zh_CN-zip&fasttmpl_flag=1)喜欢作者

5 人付费

![](http://wx.qlogo.cn/mmopen/N4HWkmwbSVSztJC7lCJGUXvJ0AvRUz2aAN22hrYljKe63X5LibXLzEIBDCYtzdO1PDk9A3QksPXIAOVwnhzPl2BsPJMps5ODTCCs5omUjYADAmEf6ANNmIqXOgxkCYJlJ/64)

![](http://wx.qlogo.cn/mmopen/ESPO4QOSuQPTh9DnYOKUSBQdWncJvGXP9xE1Vvo2D4qaVdaXz7tibgia7bzKabchrNSaicRQbsKR2RU09SfiaxniawkYWNgE9vJPj/64)

![](http://wx.qlogo.cn/mmopen/HFpkCYfkhEiaUTDDmFeMxmiaiaSU9V3w50iaQiakfKibwpRHqxDg2G9CaOASx2CxEjzBGsiaT7lRnD3YybiaGAnLWJHDchFkyrFqoxiat/64)

![](http://wx.qlogo.cn/mmopen/GY5IDas89Ljo7Kh9cL8VGGx3yBtia9zHicIdzFz27R1QpbqRewwXCKxu5D3jwiaJ6KdOibmm1cQaTdhyj5gp69qfo9pDLOsqSFkt/64)

![](http://wx.qlogo.cn/mmopen/ajNVdqHZLLCndHia8cXrv8uZS072GI8icDiapzF0NrQkGDC0cDiblbNkxJrYHKUorZ24Lpm27E9uFdBPBwXv3CsFYkHv5CHuEQhDTjuxwjbxCvvP3myDCNUYYWPCNzgXp8Kr/64)

​
