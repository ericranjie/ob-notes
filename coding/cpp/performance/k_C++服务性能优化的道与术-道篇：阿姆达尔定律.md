程序喵大人

_2022年01月28日 08:24_

以下文章来源于编程往事 ，作者果冻虾仁

\[

![](http://wx.qlogo.cn/mmhead/Q3auHgzwzM7Hvg7rQmorRljlcVCzwYTttaruhY8OCBSft64AYB32Cg/0)

**编程往事**.

C++码农，brpc committer，搜广推在线工程。专注互联网后端技术分享、行业观察以及个人成长！也欢迎关注我的知乎：果冻虾仁

\](https://mp.weixin.qq.com/s?\_\_biz=MzkyODU5MTYxMA==&mid=2247493374&idx=2&sn=08933830ce701f96d1723592685a9470&source=41&key=daf9bdc5abc4e8d0079a30fe3f46078ac6d508ff71e1ef0b8370b94ae43203c0f97a0e5fb1469279e8bb72b25a30d2a0eae7af009c805f9f0c5b89e75bccb29fa0fb11e8040bdcd926017df169b18fbc0fb09dcb64cc771fb9280fcc8c7ea55340458a00b5f6d172ad531709eb209c7cb99a07c8c57bbd579f43e4f92fd374a9&ascene=0&uin=MTEwNTU1MjgwMw%3D%3D&devicetype=Windows+11+x64&version=63090b19&lang=zh_CN&countrycode=CN&exportkey=n_ChQIAhIQZRGz2prnaBJaQibmCOHsoBLmAQIE97dBBAEAAAAAAIBkKogPWS0AAAAOpnltbLcz9gKNyK89dVj0AUbBhIJllpHtIbkKi%2F2MY3W%2FGOpSBfXyXstXV378HiNXgQM13mzboTrshNPnqxRGPbXH93bZaJvOs%2BFiAkf%2B4WB%2BcArZ2Y5BTiV3l2vNXszvHtZCNS6%2F3TKcxn2Rv38KHtewByRNWfjtDylx7bBCGKF3r8PfjMv1KIldxGeYpL5TU5OMrRj5Morlg8%2BOzFTv5sB4PAXNf9pVmcklfwCsuJgr2JvVWBesb32iMTdWZeG6ZQo2Ow4%2FO0WcN9QaMoHv&acctmode=0&pass_ticket=ps%2BZazmgxwIi4ZDTUmyjc4I%2FbIpPoase8X9cSDE8%2FB5aasravIglX%2FgC8ZVtGCW8&wx_header=1#)

## 开场白

在之前的文章 [《2004：当CPU温和地走入那个良夜》](https://mp.weixin.qq.com/s?__biz=MjM5NDIyMjI3OQ==&mid=2649005254&idx=1&sn=8e98097d5f9d5a4b982cf462bde02c69&scene=21#wechat_redirect) 中我讲到了2000年后摩尔定律的终结，CPU时钟频率定格，多核成为CPU发展的新方向，并行计算成为趋势。

在谈到并行计算的时候我们不得不提的就是阿姆达尔定律。

阿姆达尔定律即 **Amdahl's Law**。是由美国计算机科学家 **Gene Amdahl** （1922/11/16 – 2015/11/10）在 1967 年提出，旨在用公式描述在并行计算中多核处理器理论上能够提高多少倍速度。没错，学术界总是领先工业界几十年。上世纪六十年代，多核并不是刚需，而Amdahl老爷子提出的这一定律却为几十年后的程序员们指引了方向。

## 公式定义

在程序未使用多核时，有如下定义：

（）

x表示的就是程序的执行时间，其实和相同。a表示可以并行计算的代码耗时占比。

如果这个程序在N核的CPU上执行，则新的执行时间为：

（）

由于性能和耗时一般是成反比的，即耗时越低，表明性能越好。所以可以用如下公式表示性能：

表示性能提速（新性能是旧性能的多少倍）的效果：

一般将这个比值称为加速比，加速就是speed up，简写做S：故有如下公式：

n 为并行节点处理个数，可以理解为 CPU 的核心数。

## 举例探讨

别小看这个数学公式，他几乎可以让你避免做很多性能优化方面的无用功。

建设你的线上服务跑在一个32核的机器上，服务代码中有30%的代码可以进行并行化，那么进行并行化改造之后的性能是之前的多少倍呢？

约等于1.41倍，从耗时结果上来看，建设原先总耗时是140毫秒，那么进行完并行化优化之后耗时将变成100毫秒左右。

而如果你的服务中只有5%的代码可以进行并行化改造，那么优化之后的性能收益是：

性能变化几乎不大。即使你给原先100毫秒的服务，降低了5毫秒，变成100毫秒，但工作和产出不成正比，因为并行显然会增加额外的系统复杂度和维护成本。

讲到这，你会感觉，这不就是二八原则吗？没错，阿姆达尔定律所阐明的道理和二八原则如初一折，但是他用更加数学化的语言，用准确的公式定义出来了。他便足以让我们在正式开展工作之前，便得以评估自己是否在做无用功，从而让我们把精力聚焦到更有价值的部分。而传统的二八原则只是模糊的定义了大概这么一类现象，但是不管是二还是八都是模糊的数字。类似的表述还有“长尾效应”。

## 使用延伸

前面说到阿姆达尔定律定义出来的加速比公式，其实也可以推广到非并行计算领域。也就是说即使我并不是在做服务的并行化改造，我依然能从这个公式中受益。这是为什么呢？

先不考虑外部IO的耗时，当然IO一般是大头，但不在本文讨论范围再举个例子。当谈到服务本身的性能优化的时候，我们一下子可能会想到很多套路。比如C++语法优化，减少拷贝，减少频繁创建大对象。又比如系统级优化，减少系统调用等等。这些都是好的。

但是如果一个优化点，其占比不高，那么其优化带来的收益也是有限的。**再来一个例子**，比如：假设一个程序耗时100ms，其中多次运行某个逻辑总花费了80ms。现在你能做一些优化对其性能提升30%，那么对于程序整体的性能提升是多少呢？同样阿姆达尔定律可以告诉你：

总体性能提升了22%。

而如果这个逻辑总花费是10ms，你加班加点从大小周到996，对这个逻辑的性能提升了1倍！那么对程序整体的性能提升是多少呢？

虽然也提升了5%的性能，但是投入的时间显然更多。

所以这就引出了阿姆达尔定律中的一个经典教义：

> 如果被优化代码在程序整体运行时间中占比不大，那么即使对它的优化非常成功也是不值得的！

您别说我还真有切身说法。我们都知道系统调用的性能是很差的，很久以前，我集中解决了一下系统调用的问题，将一些可以不经过系统调用的逻辑进行替换。比如把`time()`函数换成`gettimeofday()`，当然严格意义上来说`gettimeofday()`也算系统调用，毕竟它也是在man手册第二页中的。但是Linux引入的`VDSO`机制，将其进行了优化。这里不展开讨论了。

这轮优化后，本来信心满满等着和领导汇报工作成果，却发现耗时几乎无变化。心想：**经验主义害死人啊**。

其实并不是“经验”不对，也不是“理论”有误，只是我当时并不知道这凌驾于其他任何优化法则的：**阿姆达尔定律！**

系统调用虽然有性能问题，但是在我整个服务中的影响占比是不高的，这里当然也不能单纯的从代码量来看，也要看一次系统调用大概花费的时间。不过我说这个例子，倒也不是说我们就要对这种明知有性能损害，但占比不高的问题听之任之。不不不，我也是有代码洁癖的，只是说我们通过理论分析可以将这种优化的优先级降低，或者裹挟一些其他方面的优化来一起做一个版本。

所以我还是奉劝大家先做一下大概的评估，这样不至于你的辛勤工作在别人眼里看起来没有卵用。或者也可以在向领导讲述某次“失败”的优化的时候找点理论支撑。

![](https://res.wx.qq.com/op_res/NN_GToMiIjsXzgPzF9-74ZzwR3cA9-fv3o9eWo8f5gQWqx71CmGlY8kFxuIxZaG0TB1bFeMCmh1DGN_pWMRg0A)

**帐号已迁移**

公众号

阅读 1430

​

喜欢此内容的人还喜欢

写留言

**留言 2**

- 攒钱买月亮

  2022年1月28日

  赞3

  那么问题来了，amdahl's law中的a如何计算，作者大佬用了"比如是30%"这样的说法，回避了这个重要的问题，能否请您具体探讨一下如何确定一段待优化的逻辑的占比呢？

  程序喵大人

  作者2022年1月28日

  赞1

  这个问题，我也只是停留在理论层面![[囧]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)

已无更多数据

[](javacript:;)

![](http://mmbiz.qpic.cn/sz_mmbiz_png/Av3j9GXyMmGmg0HNK7fR6Peq3uf9pUoN5Qt0s0ia8IH97pvIiaADsEvy9nm3YnR5bmA13O8ic0JJOicQO0IboKGdZg/300?wx_fmt=png&wxfrom=18)

程序喵大人

6分享4

2

写留言

**留言 2**

- 攒钱买月亮

  2022年1月28日

  赞3

  那么问题来了，amdahl's law中的a如何计算，作者大佬用了"比如是30%"这样的说法，回避了这个重要的问题，能否请您具体探讨一下如何确定一段待优化的逻辑的占比呢？

  程序喵大人

  作者2022年1月28日

  赞1

  这个问题，我也只是停留在理论层面![[囧]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)

已无更多数据
