
CPP开发者 _2022年04月20日 11:50_

Jason Donenfeld 是 WireGuard 的主要开发者，同时他也是 Linux 内核随机数相关代码的维护者，近日在他的领导下，Linux 内核的随机数生成器代码有了巨大幅度的改进。

# **getrandom()背景**

2014年的时候，getrandom()被加入kernel，当时一个理由是因为LibreSSL project在Linux上抱怨在文件描述符被黑客恶意用光的情况下没有办法生成随机数。此前的获取方法是从/dev/urandom来读取，不过如果攻击者让所有文件都被打开，那么就没法打开/dev/urandom了，也就没法拿到随机苏。这样大家就创建了getrandom()系统调用，能够在不需要文件描述符（甚至像chroom或者container里这种没有/dev/urandom文件节点）的情况下就能拿到随机数了。

getrandom()是故意设计成这样的，就是希望在/dev/urandom随机数池初始化好之前阻塞住不返回。在发明getrandom()调用之前，user space没有任何方法能确保系统采集了足够的entropy熵值并生成了随机数池。也就是说/dev/urandom的行为是kernel ABI定义好的，不可以更改，而新加的系统调用则可以不遵循这个限制。

要想完成CRNG的初始化，需要有512 bit的entropy，或者说需要有4096次中断。Ts'o觉得这个值选得有些保守。getrandom()的文档里清楚的描述了这个阻塞行为，不过user space开发者仍然有人会错用这个API。

Ts'o认为今后这里肯定会有问题，必须得找个方法，要么能让user space开发者知道这里不能保证启动阶段之后就能使用getrandom，要么让大家能绝对信任Intel和RDRAND指令。否则的话，今后kernel里面哪里又有一个优化导致中断数变少的话，又会导致某人的系统或者application卡死了。

**Linus Torvalds**指出，RDRAND指令又不是在每个系统上都有的，所以没法依靠RDRAND指令来解决所有问题。他也觉得今后ssd会使用更多的polling而不是中断的情况下，情况会变得更糟，大家要针对这种“更少中断”的环境早作打算。

# **优化**

在之前的 Linux 5.17 中，Jason Donenfeld 就在随机代码用 BLAKE2s 代替了 SHA1，由于 BLAKE2s 自带的特性，前者通常比后者更快更安全。经过测试，通过这个简单的转换就能获得 131% 左右的速度提升。

虽然在 Linux 5.17 中有了速度上的大幅提升，但 Jason Donenfeld 对此并没满足。因此在 Linux 5.18 中他对随机代码作出了更多的改进。

![[Pasted image 20240920124501.png]]

通过查看 Linux 的 random.git 仓库的日志能够看出（上图），开发者 Jason Donenfeld 在最近两天时间里进行了大量的代码提交。这些提交内容都将在 3 月下旬 Linux 5.18 的合并窗口启动时引入内核。

![[Pasted image 20240920124507.png]]

在邮件中特别强调到，通过使用正在开发的最新代码，用于获取随机字节的 getrandom() 调用能够获得更好的性能。在配备英特尔 Xeon E5-2697 v2 @ 2.70GHz CPU 和 112G 内存的设备上进行 stress-ng getrandom() 基准测试后，更是获得了 8450% 的性能提升。

此次更改基本上会将之前的全局结构（实际上是 per-numa 节点结构）更改为 per-cpu 结构，这意味着快速路径上的许多锁都会消失。因此，当在具备多核的 CPU 上同时尝试 getrandom() 时，毫无疑问性能会出现提升。只不过没想到在测试中能带来 8450% 的提升。

除此之外，当从 per-numa 更改为 per-cpu 后，也将不再需要被推迟到工作队列上线后才能进行。也正如我之前所说，此次改进将会为高核心数的电脑和服务器带来巨大收益。

来源：OSC开源社区

链接：https://www.oschina.net/news/183698/linux-getrandom

- EOF -

推荐阅读  点击标题可跳转

1、[内核虚拟机原理：网络可编程](http://mp.weixin.qq.com/s?__biz=MzAxNDI5NzEzNg==&mid=2651170783&idx=1&sn=0b6db991015ff8b3315aa86bf8c42c38&chksm=80647680b713ff96d94c570741cbe118b37038a352d8a002bfa12d9468fdef2ee69af577724e&scene=21#wechat_redirect)

2、[Linux 内核源码如何学习？](http://mp.weixin.qq.com/s?__biz=MzAxNDI5NzEzNg==&mid=2651170211&idx=1&sn=7ba738bb7fc747a91a7ddb5c611188cf&chksm=806474fcb713fdea4c77748ab87d77b965a4a9d3bef422b0505f1b0c3de1471f40643a4757e6&scene=21#wechat_redirect)

3、[深入理解 Linux 内核之内存寻址](http://mp.weixin.qq.com/s?__biz=MzAxNDI5NzEzNg==&mid=2651170178&idx=1&sn=b6bfee64d154b061ebd9be0e0b27f4f0&chksm=806474ddb713fdcb07c76b9126814749b9dc00bc2c0e10bf6034df438b65b0b84c95774be443&scene=21#wechat_redirect)

**关注『CPP开发者』**

---

**CPP开发者**

我们在 Github 维护着 9000+ star 的C语言/C++开发资源。日常分享 C语言 和 C++ 开发相关技术文章，每篇文章都经过精心筛选，一篇文章讲透一个知识点，让读者读有所获～

24篇原创内容

公众号

点赞和在看就是最大的支持❤️

函数3

linux内核2

函数 · 目录

上一篇C++ 虚函数表剖析下一篇C 函数指针别再停留在语法，得上升到软件设计

Reads 4446

​

People who liked this content also liked

free() 函数只传入一个内存地址，为什么能知道要释放多大的内存？

我关注的号

CPP开发者

不喜欢

不看的原因

OK

- 内容低质
- 不看此公众号内容

![](https://mmbiz.qpic.cn/mmbiz_jpg/pldYwMfYJpiawQcapMaR3POhqqIhveVHcSwEVINtRVFv14Wagg9JqtiborQYQ5ZZu2rLKEvX3Ov4diaQB2ibdGia7icQ/0?wx_fmt=jpeg&tp=wxpic)

关于近期暗网上国内信息泄露的情报汇总

4个朋友读过

CatalyzeSec

不喜欢

不看的原因

OK

- 内容低质
- 不看此公众号内容

![](https://mmbiz.qpic.cn/sz_mmbiz_jpg/EqMwaEZz0ykOdkvA53BqLbcbBDJ9Xvocf0SQnjHQiczhbqbnNm4LM5uuxLichCBUszIpyv2MYTzdg6aaCnW8nNGA/0?wx_fmt=jpeg)

如何让你的C程序打印的log多一点色彩？（超级实用）

我常看的号

一口Linux

不喜欢

不看的原因

OK

- 内容低质
- 不看此公众号内容

![](https://mmbiz.qpic.cn/sz_mmbiz_jpg/icRxcMBeJfc9TF3bYLZmEP9FQicGGtvsaK9DurNhuJGQwwj7Fiaj0pamRrIIro07vgCLqlPv3KVVcnZico9xKxfDNA/0?wx_fmt=jpeg)

Comment

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/pldYwMfYJpia3uWic6GbPCC1LgjBWzkBVqYrMfbfT6o9uMDnlLELGNgYDP496LvDfiaAiaOt0cZBlBWw4icAs6OHg8Q/300?wx_fmt=png&wxfrom=18)

CPP开发者

12ShareWow

Comment

Comment

**Comment**

暂无留言
