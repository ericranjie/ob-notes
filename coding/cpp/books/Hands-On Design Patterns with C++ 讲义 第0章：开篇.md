# 

原创 果冻虾仁 编程往事

_2022年02月27日 23:23_

**`点击底部阅读原文，观看视频版`**

# 开场白

Hello大家好，我是果冻。前几天有个自称B站运营的人（咱也不知道真假）通过知乎找到我，邀请我去报名B站星计划的活动，去做一名up主。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/hQZ4NEZ2sic72tUfvjgLjrywcM59QNFibcEDxictvDlRIbQ7UB2J22kfugmiahrYvtm4F5KxoJwKCwOZ5gHBztoGww/640?wx_fmt=jpeg&wxfrom=13&tp=wxpic)

这个活动会送一些流量奖励吧。不过流量不流量的其实无所谓，我之前就考虑过做视频，主要还是想通过录视频锻炼一下自己讲话和组织语言的能力吧。毕竟这在职场上也是很重要的能力，工作时间越长，这些技术以外的软能力就变得日益重要起来。

于是我就重新注册了一个B站账号，作为UP主加入知识区了。但是老ID：**`果冻虾仁`** 不知道被谁注册了，提示重名。就连 **`编程往事`** 也被注册过了……所以我给ID后面加了一个特殊符号：**`|`** 。现在是 **`果冻虾仁 |`** 。不过应该不影响站内搜索结果，搜索果冻虾仁，还是能搜索到我。

好了，既然加入知识区了，那就讲点课程吧。目前我的想法就是带大家学习一本我认为还不错的C++图书。这本书其实知名度并不是特别高，但是内容质量确实还是不错的。之所以知名度不高，可能是因为这本书是2019年出版的，距今只有三年，并且没有中文版。我自己看的也是英文版。![图片](https://mmbiz.qpic.cn/mmbiz_png/hQZ4NEZ2sic72tUfvjgLjrywcM59QNFibcj6rMHH9ibKd3mrM6n8DbfxlsTsou9xYwgibwiac5NFyk2DF7jeB264Dgg/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

> 其实2019年我就关注到这本书了，不过那时候免费电子版是真的找不到，只有收费的版本……所以当时就作罢了。

# 第0章 开篇

言归正传，姑且把这本书称作教材。

## 教材

|书名|Hands-On Design Patterns with C++|
|---|---|
|副标题|Solve common C++ problems with modern design patterns and build robust applications|
|作者|Fedor G. Pikus|

## 作者：Fedor G. Pikus

Fedor G. Pikus 是高性能计算（HPC）与C++领域的专家。

![图片](https://mmbiz.qpic.cn/mmbiz_png/hQZ4NEZ2sic72tUfvjgLjrywcM59QNFibchJwURWjJWUicAfM3qbiboKkmI2cVMb5ndY32qOT6PB9ILnJLGaSEjf7A/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

从1998年至今就职于 Mentor Graphics，该公司是全球三大EDA软件厂商之一，2016年被西门子以45亿美元的价格收购。

> 佩服国外程序员能坚持在一个公司二十多年。也佩服他工作这么久了还在浸淫技术，当然他是首席科学家，在国内可能早就不学技术了。

除此以外呢，Fedor Pikus也是各大技术大会上的常客，几乎每年都会来CppCon大会做分享，有时候一次大会上甚至会做多个Talk。下面列出几个他在CppCon上做过的分享。

|大会|CppCon2021|
|---|---|
|主题|Branchless Programming in C++|
|链接|https://www.youtube.com/watch?v=g-WPhYREFjk|
|**大会**|**CppCon2019**|
|主题|Back to Basics: Test-driven Development|
|链接|https://www.youtube.com/watch?v=RoYljVOj2H8|
|**大会**|**CppCon2018**|
|主题|Design for Performance|
|链接|https://www.youtube.com/watch?v=m25p3EtBua4|
|**大会**|**CppCon2017**|
|主题|C++ atomics, from basic to advanced. What do they really do?|
|链接|https://www.youtube.com/watch?v=ZQFzMfHIxng|
|**大会**|**CppCon2016**|
|主题|The speed of concurrency (is lock-free faster?)|
|链接|https://www.youtube.com/watch?v=9hJkWwHDDxs|
|**大会**|**CppCon2015**|
|主题|C++ Metaprogramming: Journey from simple to insanity and back|
|链接|https://www.youtube.com/watch?v=CZi6QqZSbFg|
|主题|Live Lock-Free or Deadlock (Practical Lock-free Programming)|
|链接|https://www.youtube.com/watch?v=lVBvHbJsg5Y|
|主题|The Unexceptional Exceptions|
|链接|https://www.youtube.com/watch?v=fOV7I-nmVXw|

# 题外话：什么是设计模式？

下面我讲个和书中内容无关的题外话。现在提到设计模式，可能更多的是Java程序员之间的讨论。其实第一本系统总结出来各种常见模式的书籍，是基于C++描述的。

![图片](https://mmbiz.qpic.cn/mmbiz_png/hQZ4NEZ2sic72tUfvjgLjrywcM59QNFibculiaBwnpGAPILxzE5bAicz1lpk6TJSPB7dat1HMibtSIiaiclCFZ4wGsMMQ/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

设计模式 —— 可复用面向对象软件的基础

由于有四位作者，因此这本书（的作者）常常被戏称为四人帮（GOF）……

但是初学者（甚至一些有经验者）经常对于设计模式一词理解不清。比如MVC是设计模式吗？proactor/reactor是设计模式吗，半同步半异步、领导者追随者是设计模式吗？

其实都是设计模式。但你可能会疑惑那这些内容怎么没出现在这本书中？

请注意该书的副标题，是“可复用面向对象软件的基础”。所以我们常说的单例模式、工厂模式、装饰器模式等等都是面向对象领域的设计模式。这个是有限定词的。

而MVC并不属于面向对象领域的设计模式，它属于Web开发架构上的设计模式，是在另外一个不同的层次上。像proactor/reactor之类的则属于网络编程相关的设计模式，也可以称作网络模型。

所以设计模式一词是一个很宽泛的词，它不止包含面向对象的内容。

# 大纲

本课程的大纲，也就是《Hands-On Design Patterns with C++》这本书的目录。它一共有18个章节。

|章节|标题|
|---|---|
|1|An Introduction to Inheritance and Polymorphism|
|2|Class and Function Templates|
|3|Memory Ownership|
|4|Swap - From Simple to Subtle|
|5|A Comprehensive Look at RAII|
|6|Understanding Type Erasure|
|7|SFINAE and Overload Resolution Management|
|8|The Curiously Recurring Template Pattern|
|9|Named Arguments and Method Chaining|
|10|Local Buffer Optimization|
|11|ScopeGuard|
|12|Friend Factory|
|13|Virtual Constructors and Factories|
|14|The Template Method Pattern and the Non-Virtual Idiom|
|15|Singleton - A Classic OOP Pattern|
|16|Policy-Based Design|
|17|Adapters and Decorators|
|18|The Visitor Pattern and Multiple Dispatch|

前面几个章节偏基础，后面开始由浅入深。讲到了SFINAE、CRTP等模板编程技术。当然对于传统的面向对象的设计模式也有讨论。比如单例模式，该书就花了大量篇幅来讨论。对于现代C++而言，很多老的议题都是值得重新拿出来讨论的，因为很多细节已经和90年代所不同了。

当然这里并不包含网络编程相关的设计模式，前面我只是说了一个题外话，本书主要还是讲解软件工程领域的设计模式（包含面向对象以及其他C++的惯用法）。

# 关注我

## 知乎：果冻虾仁

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

## B站：果冻虾仁丨

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

## 公众号：编程往事

![](http://mmbiz.qpic.cn/mmbiz_png/hQZ4NEZ2sic5WOaB0JA1LvD2OpPWttPuvBR21zFyLcgUnwGRzSSp1tpj4jOY2KnSia3xsUzeevb5tibl8ajStOZlg/300?wx_fmt=png&wxfrom=19)

**编程往事**

C++码农，brpc committer，搜广推在线工程。专注互联网后端技术分享、行业观察以及个人成长！也欢迎关注我的知乎：果冻虾仁

103篇原创内容

公众号

# 谢谢大家

**`点击阅读原文，关注我的B站。现在关注就是老粉啦。`**

C++46

C++1115

设计模式3

C++ · 目录

上一篇C++ Trick：右值引用、万能引用傻傻分不清楚下一篇Hands-On Design Patterns with C++ 下载

阅读原文

阅读 866

​

喜欢此内容的人还喜欢

零基础入门 Rust？这个 100 练习项目值得一试！

数据科学研习社

不喜欢

不看的原因

确定

- 内容低质
- 不看此公众号内容

![](https://mmbiz.qpic.cn/sz_mmbiz_jpg/tMAjXPW04C07jfKM9ojQO8ibv4xKc5og6BBSlDm2SFo7czYkEUiaHBHLUCHX0XG2pfGhaPbSIKtCzgvUX1QY7u5w/0?wx_fmt=jpeg)

要多长时间，才能写出像我们项目这样的代码？

无际单片机编程

不喜欢

不看的原因

确定

- 内容低质
- 不看此公众号内容

![](https://mmbiz.qpic.cn/sz_mmbiz_jpg/ZwPBgibS4LLgln4DLALkNHyZvurnD8VOBpnqyLTseAoFFNuBZJpG7YV7d9fbetqvEtTrJIx981D9nMrOAcD9gFA/0?wx_fmt=jpeg)

哎呀我去，Python的len竟然不是方法？

Python不灵兔

不喜欢

不看的原因

确定

- 内容低质
- 不看此公众号内容

![](https://mmbiz.qpic.cn/mmbiz_jpg/5mnoGrqV9nAQmXPNCp3numzgzDTCeiaOWg6Ptz7IgkJWkervGWpIwbPODJyj8vXAZk6QCMc1sMsSOQhY45581LQ/0?wx_fmt=jpeg&tp=wxpic)

写留言

**留言 5**

- GlobaL Flanker

  2022年2月27日

  赞1

  up主的高质量高光时刻即将来临，期待ing！

  编程往事

  作者2022年2月28日

  赞2

  ![[捂脸]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)还在摸索，录制视频课程我不专业。继续学习中

- DD

  2022年2月27日

  赞

  想要电子版![[合十]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![[合十]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![[合十]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

  编程往事

  作者2022年2月27日

  赞2

  回头我发到公众号里，敬请关注。

  DD

  2022年2月27日

  赞

  好的好的，感谢感谢

已无更多数据

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/hQZ4NEZ2sic5WOaB0JA1LvD2OpPWttPuvBR21zFyLcgUnwGRzSSp1tpj4jOY2KnSia3xsUzeevb5tibl8ajStOZlg/300?wx_fmt=png&wxfrom=18)

编程往事

1613

5

写留言

**留言 5**

- GlobaL Flanker

  2022年2月27日

  赞1

  up主的高质量高光时刻即将来临，期待ing！

  编程往事

  作者2022年2月28日

  赞2

  ![[捂脸]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)还在摸索，录制视频课程我不专业。继续学习中

- DD

  2022年2月27日

  赞

  想要电子版![[合十]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![[合十]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![[合十]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

  编程往事

  作者2022年2月27日

  赞2

  回头我发到公众号里，敬请关注。

  DD

  2022年2月27日

  赞

  好的好的，感谢感谢

已无更多数据
