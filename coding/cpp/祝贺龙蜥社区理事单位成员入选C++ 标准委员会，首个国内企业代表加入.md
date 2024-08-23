
云巅论剑 云巅论剑

 _2021年11月17日 17:30_

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

近日，龙蜥社区的贡献者之一、来自理事单位阿里云的许传奇，代表其团队基础软件程序语言与编译器（以下简称“程序语言与编译器团队”）加入了 C++ 标准委员会。**这是首个国内企业代表进入 C++ 标准委员会**。

C++20 是 C++ 的一个重大更新，例如 Coroutine、Module、Concept 以及  Range 等。其中 Coroutine 可以让程序员以同步方式编写高并发的异步代码，会带来性能与开发效率的双重提升。**程序语言与编译器团队实现了一个高性能的轻量级协程库，允许 C++ 开发者以同步方式写异步代码**。也正因为这个特点，同步代码可以很方便地切换到协程代码，同时完成异步化，这往往能获得一个数量级的性能提升。而协程也可以使代码更简洁易懂、方便维护。

但 Coroutine 在正式进入 C++20 时，其支持并不完善。一方面是编译器支持层面有许多问题，如优化不完善、bug 比较多等；另一方面是在标准制定层面，Coroutine 只制定了基础语法，并没有完成协程库的制定。由于 C++20 协程的语法对 C++ 开发者而言难以理解，不容易直接使用，因此一个包装好的协程库是必须的。**如****果没有一个稳定的编译器支持，那使用协程必然是没有希望的；同时如果没有一个好用易懂的协程库，那大规模地使用协程也必然没有希望**。

据许传奇透露，新语言标准在大规模 C++ 项目中的规模化落地并不容易，因为是最新标准，在落地过程中遇到的许多问题在公开的互联网中并不存在，更不用提解决方案了，所以大部分时间都花在理解与解决这些问题上。在积累新标准在大规模  C++ 项目中的实践经验的同时，一方面对当前标准的设计有了更深的理解，另一方面也看到了可以改进标准的机会。经过持续努力，**程序语言与编译器团队完成了协程在大型 C++ 项目中的规模化应用**。在这个过程中，他们不断地尝试将经验、 问题与解决方案反馈到 Clang/LLVM 与 C++ 社区，也得到了社区的高度认可。

龙蜥社区一直秉持着开放、中立的原则，一方面欢迎更多的企业和企业优秀成员加入社区，一方面社区企业和成员们也在积极为国际社区做贡献。龙蜥社区的理事单位会一直持续地将基础软件领域的工作贡献到 Linux Kernel、OpenJDK、Clang/LLVM、GCC 等社区，另外其他的工作也会逐步开源。

进入 ISO C++ 标准委员会，这代表着龙蜥社区理事单位之前在 C++ 语言方面的工作走在正确的道路上，也代表着其正式踏进了语言演化生态的上游，进入了设计阶段。

许传奇表示，希望通过参与程序语言标准的制订，进入到程序语言演化周期的上游，以把握住程序语言技术演进的主航道，打造出领先的程序语言基础设施。未来，这一成果也将支持和反馈到龙蜥社区中。

再次恭喜龙蜥社区成员许传奇同学入选，也欢迎更多优秀的人加入龙蜥社区。

**团队招人啦：**

「阿里云程序语言与编译器团队」负责编译、语言(Java 等)运行时的研发与产品化。团队针对阿里各业务的需求、新技术的发展、新硬件的引入，在编译器、语言运行时等基础领域进行研究创新。目前已经形成 Alibaba Dragonwell、Eclipse Jifa、Alibaba LLVM 等多个产品。加入这个优秀的团队，你可以不断挑战基础软件领域的新技术，你可以向社区提 Patch，你的工作可以应用到阿里巴巴的数十万台服务器上。

**招聘官****网地址：**

1.https://talent.alibaba.com/off-campus-position/797028?spm=a1z9iw.13825095.0.0.33d03ae7X3l1tt

2.https://talent.alibaba.com/off-campus-position/726460?spm=a1z9iw.13825095.0.0.33d03ae7X3l1tt

招聘详情可点击【**阅读原文**】查看，等你来。

**往期精彩推荐**

[1.fastFFI 官宣开源，一款高效的Java跨语言通信框架](http://mp.weixin.qq.com/s?__biz=MzUxNjE3MTcwMg==&mid=2247486608&idx=1&sn=6d22cea70b09396b6ce8c30387f3a5ad&chksm=f9aa3e49ceddb75feebece326222876ddf7a4919cd189720dcd8349868eeb32f54c0e9b51162&scene=21#wechat_redirect)  

[2.双十一咋省钱？KeenTune助你业务资源省省省](http://mp.weixin.qq.com/s?__biz=MzUxNjE3MTcwMg==&mid=2247486559&idx=1&sn=ac1ef70ab5293ca319ba648685cf29f2&chksm=f9aa3e86ceddb790baac51500f8527a5c8c87a235bc0a5a09f7f9dd57f19fb9546cb5f4bf0ef&scene=21#wechat_redirect)  

[3.LifseaOS 悄然来袭，一款为云原生而生的 OS](http://mp.weixin.qq.com/s?__biz=MzUxNjE3MTcwMg==&mid=2247486533&idx=1&sn=1d36226e13846a21569c3b6a7ee8123e&chksm=f9aa3e9cceddb78aa227a01d5d824dd844c92152ff1f8e4f7d1ffacb3d21e52a1653f0ccf0f6&scene=21#wechat_redirect)  

[4.6位大咖坐在一起聊了聊：龙蜥社区做了什么、以及能带来什么？](http://mp.weixin.qq.com/s?__biz=MzUxNjE3MTcwMg==&mid=2247486513&idx=1&sn=a150f437b51235dd68ae7c2132f4378c&chksm=f9aa3ee8ceddb7fe09bc0bb7031ef028721cba2baa39311e9f0abeb97e0d1a4916ebc5f885fa&scene=21#wechat_redirect)  

[5.龙蜥操作系统将捐赠开放原子开源基金会](http://mp.weixin.qq.com/s?__biz=MzUxNjE3MTcwMg==&mid=2247486509&idx=1&sn=3ed1fc206d989ccacf310bf7f00beaf6&chksm=f9aa3ef4ceddb7e24cba0dc3d67b1d54950771166aaddea99e0a7aada9dd104f14ec98358507&scene=21#wechat_redirect)  

[![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)](http://mp.weixin.qq.com/s?__biz=MzUxNjE3MTcwMg==&mid=2247486608&idx=1&sn=6d22cea70b09396b6ce8c30387f3a5ad&chksm=f9aa3e49ceddb75feebece326222876ddf7a4919cd189720dcd8349868eeb32f54c0e9b51162&scene=21#wechat_redirect)

阅读原文

阅读 1032

​

喜欢此内容的人还喜欢

天气雷达拼图系统V3.0产品数据解析

MeteoInfo

不喜欢

不看的原因

确定

- 内容低质
- 不看此公众号内容

![](https://mmbiz.qpic.cn/mmbiz_jpg/xpWiaNAD02dUU8e4cL2qU3XFZkY6ZDcg8fs4NhnibEL8gECYL7EYSiaRWWViaBh0PU7ib6oaic7lsPX8LxSghUmefZvg/0?wx_fmt=jpeg&tp=wxpic)

乳臭未干的我却给领导干部上课，而且已经是第二个单位了

IT狂人日志

不喜欢

不看的原因

确定

- 内容低质
- 不看此公众号内容

![](https://mmbiz.qpic.cn/sz_mmbiz_jpg/BR8aBnW55lkoYIzspUGngs4Tl1HD5UxtPRdYia11XfKV1yzrvlfKucDL6Tq5sWVbGIz5ARu3LGyk6w3NZummgAg/0?wx_fmt=jpeg)

新员工一口气写完这些C语言例子，领导直接给他转正了！

我关注的号

一口Linux

不喜欢

不看的原因

确定

- 内容低质
- 不看此公众号内容

![](https://mmbiz.qpic.cn/sz_mmbiz_jpg/icRxcMBeJfc8gONP9Zs4ELWOcP4aNxZibhUGbKgT2ZF3u6iaN8YoGAnfiaZgeHpLZzsGQ7IXE0TdicX8NMDxgay4Opg/0?wx_fmt=jpeg)

写留言

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/bDZYPQUwvDnKe3dtY0vLjWoesXQTtm9b7FibLgECY6rl5Hz4C7NYVQf26icepTa1yUOdbxTNWFxYNm9g5Fv0iaMRA/300?wx_fmt=png&wxfrom=18)

云巅论剑

13分享2

写留言

写留言

**留言**

暂无留言