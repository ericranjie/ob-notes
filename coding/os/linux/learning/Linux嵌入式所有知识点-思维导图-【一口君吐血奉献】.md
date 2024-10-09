# 

Original 土豆居士 一口Linux

_2021年09月22日 11:52_

# 推荐关注👇下方公众号学习更多Linux、驱动知识！

![](http://mmbiz.qpic.cn/mmbiz_png/icRxcMBeJfc8535w2vKlsLPf5hwdMjpYrzuVCHx0rcQmvv8rYqTFtIyic5qErtciaibqaIOWgeKkDsOMeae4HciaUaw/300?wx_fmt=png&wxfrom=19)

**一口Linux**

《从零开始学ARM》作者，一起学习嵌入式，Linux，网络，驱动，arm知识。

300篇原创内容

公众号

## 一、前言

很多粉丝问我，我的Linux和嵌入式当初是如何学习的？

其实彭老师在最初学习的过程中，走了相当多的弯路：

**有些可以不学的花了太多的时间去啃 有些作为基础必须优先学习的，却忽略了， 结果工作中用到这些知识时傻眼了 有些需要后面进阶阶段学习的，结果提前看了，看的晕头转向，浪费时间**

作为初学者，抓不住重点，走弯路，

**哪些要了解就可以了，哪些必须熟练掌握，**

根本搞不清楚，

相信每个过来人都深有体会。

所以一口君特地了整理了嵌入式驱动工程师学习路线：

《[嵌入式驱动工程师学习路线](https://mp.weixin.qq.com/s?__biz=MzUxMjEyNDgyNw==&mid=2247496985&idx=1&sn=c3d5e8406ff328be92d3ef4814108cd0&chksm=f96b87edce1c0efb6f60a6a0088c714087e4a908db1938c44251cdd5175462160e26d50baf24&scene=21&token=1106299436&lang=zh_CN#wechat_redirect)》

那么我当初到底是如何学习Linux、网络、ARM、Linux驱动等各个领域的技术的呢？

上几张图，大家自己看下吧。

### 1. 我看过的部分书籍

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)这只是其中很少一部分，有一些都送给我学生了，

其中关于驱动的书，基本都看了四五遍。

### 2. 整理过的文章

下面是一口君的有道云笔记，这么多年学习嵌入式，总结的各种技术文章目录均存于此：

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)下面是学习驱动总结的所有知识点对应的目录（红框内均是），每一个目录下都是几十篇文章。!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)我收藏在有道云笔记的文章，都是我精心筛选过的，并且有许多文章是重新整理过的

比如I2C这个知识点，我会从网上搜集各种关于这个知识点的文章，

因为作者使用的平台不一样，开发任务的重点不一样，

有的搞硬件的会从画电路图角度讲解， 有的用的是单片机，那么他讲解的角度就是基于裸机驱动角度， 有的用的是Cortex系列，在linux上跑的，那么就会讲Linux架构下驱动的编写， 也有作者会分析Linux内核I2C子系统的实现等等

这些文章的有的讲的深，有的讲的浅，各种优缺点， 那么我就会把这些文章有闪光点的地方全部吸取， 然后汇总到我的笔记中

所以特别怕网易哪天把有道云笔记给下线了，

那我要亏大了。

### 3. 开发板合影

下面是一口君这么多年所玩过的部分开发板合影：!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

这些板子包括：CC2530、arduino、pcduino、树莓派1、树莓派2、树莓派3、ARM A-8 A-9、正点原子阿尔法、GPRS套件、智能小车套件，还有各种传感器、显示屏、外设；

这些板子、配件还送给我的学生一部分，前后累计起来也有不少钱了。

钱财事小，毕竟是要学知识，知识是无价的！

**这几年，一口君看了无数的博文、书籍、视频， 也看了无数的用户手册、 编写了无数的代码， 解了无数的bug，**

而这些都是在业余时间完成的。

## 二、所以到底要如何学习Linux、嵌入式？

关于这个问题，一口君无法给出一个标准答案，

因为每个人的专业、基础、年龄、兴趣、毅力都不一样，

一口君唯一能确定的是：

**在学习嵌入式、Linux的路上，没有任何的捷径可走，**

不论是要入门、进阶、转行还是兴趣爱好，

必须制定一个长期的学习计划、并利用好自己所有的业余时间，刻苦学习，

**坚持是成功的唯一方法！**

## 三、嵌入式学习知识点思维导图

为了帮助更多的朋友学习，现特地将嵌入式学习知识点，整理成思维导图。

**或者方式见文章底部**

大家可以根据自己所处的阶段，

有针对性的来补充自己的知识，尽量少走弯路，

如果还是搞不清楚，

也欢迎大家加我好友，

**一起交流技术， 一起卷到天昏地暗，海枯石烂！**

**山无棱、天地合，我也不与君绝！**

部分截图如下：!\[ \](https://img-blog.csdnimg.cn/d31c101d823b4279b1f8340c0e4db4ea.png

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

完整的pdf文档请后台回复：**嵌入式思维导图**

- END -

______________________________________________________________________

**关注，回复【****1024****】海量Linux资料赠送**

**精彩文章合集**

![](http://mmbiz.qpic.cn/mmbiz_png/icRxcMBeJfc8535w2vKlsLPf5hwdMjpYrzuVCHx0rcQmvv8rYqTFtIyic5qErtciaibqaIOWgeKkDsOMeae4HciaUaw/300?wx_fmt=png&wxfrom=19)

**一口Linux**

《从零开始学ARM》作者，一起学习嵌入式，Linux，网络，驱动，arm知识。

300篇原创内容

公众号

[linux](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=MzUxMjEyNDgyNw==&action=getalbum&album_id=1507350615537025026#wechat_redirect)入门

[C语言](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=MzUxMjEyNDgyNw==&action=getalbum&album_id=1531367002169212930#wechat_redirect)

[Linux驱动](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=MzUxMjEyNDgyNw==&action=getalbum&album_id=1502410824114569216#wechat_redirect)

[ARM](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=MzUxMjEyNDgyNw==&action=getalbum&album_id=1614665559315382276#wechat_redirect)

[计算机网络](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=MzUxMjEyNDgyNw==&action=getalbum&album_id=1598710257097179137#wechat_redirect)

[粉丝问答](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=MzUxMjEyNDgyNw==&action=getalbum&album_id=1629876820810465283#wechat_redirect)

[所有原创](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=MzUxMjEyNDgyNw==&action=getalbum&album_id=1479949091139813387#wechat_redirect)

点击“**阅读原文**”查看更多分享，欢迎**点分享、收藏、点赞、在看**

![](https://mmbiz.qlogo.cn/mmbiz_jpg/h5qLUehhRhd5fEkYx3icmhEn7HzMF25aia45k0kTVelC8aG5ZNsXn5IXCjR3hkgg4fqM2zFRXDp3GqrbDhKoeEoQ/0?wx_fmt=jpeg)

土豆居士

植发

![赞赏二维码](<https://mp.weixin.qq.com/s?__biz=MzUxMjEyNDgyNw==&mid=2247497822&idx=1&sn=1e2aed9294f95ae43b1ad057c2262980&chksm=f96b8aaace1c03bc2c9b0c3a94c023062f15e9ccdea20cd76fd38967b8f2eaad4dfd28e1ca3d&mpshare=1&scene=24&srcid=0922qCU9UEkTVcDlmCFtOtgC&sharer_sharetime=1632307038128&sharer_shareid=5fb9813bfe9ffc983435bfc8d8c5e9ca&key=daf9bdc5abc4e8d0e5c9f79de23d65516a3989f5482742f4a85e11974b75e36db5d21616e1c8669ed4ce16f7a616adfcd3a95fbe2c4ef2092172b15ff8ae84da32519e8a3dbc14321c68944958a0caa0a3731a250bb318ea5b13f17208b33c67cac2413a60f5ac7e712fa07238614a8d6415e4729650a8a417518463f4e3fb8b&ascene=14&uin=MTEwNTU1MjgwMw%3D%3D&devicetype=iMac+MacBookAir10%2C1+OSX+OSX+14.6.1+build(23G93)&version=13080710&nettype=WIFI&lang=en&session_us=gh_fc2c47bdbd29&countrycode=CN&fontScale=100&exportkey=n_ChQIAhIQ4oqStpm21fWyEGXSQZHCBRKUAgIE97dBBAEAAAAAAM9yExmCxJAAAAAOpnltbLcz9gKNyK89dVj0%2BM%2BFGJ%2F3tGoJLiv78VHJMcL192kHf%2FOt4waDRQsG673o1ApxnfWL4rWtxLOwl%2FpNt0TDqtvMYdSBZYokcc6qGNCYfB0LynToazGSHBQCUR7F3eG5piR0zzU%2FUQ4FalDz9OGH9iGlCxg1u2mN%2FZCC71jWBGAUiDccIeu49tzVNy5DbvnpIv4TTm70uCwDBy8wAkSVW%2BdBPx8S4I3qMAQTWZpmiluqCclVF7%2BcOpDuuOpOuV0EICTZoJMcFjiXykdG8HL1GGu0zyfRc4I67K9oqBuhUfrcEGb1CaWdB5OLUxGNIp%2Fs7GRijM%2BFu37WDA%3D%3D&acctmode=0&pass_ticket=F%2BEJKHm0jS4GT5NK7hmK1N5UzkBSKFtoZPRpriWwNalUbd20Wx3TFzlYfY65ETuI&wx_header=0>)Like the Author

13 like(s)

![](http://wx.qlogo.cn/mmopen/QzPtZqIzA0mMGaIaSUW2ZfubJgkkNIfTa3aslTGUlaDMWelK9r1pesCAycTsFpBK0Vjw544Dztpz627WHeiaiapbPTZJhbRcYcr582PD7xhacAUTxwZCUxDCeZPuJWoHlD/64)![](http://wx.qlogo.cn/mmopen/ajNVdqHZLLAuohh7ibCRxJRzZg6AlTYDq8yEV5RLtMeibxr7XV7ANa1YmGhq8uXoOBCFMjHWickNbicX9jcMor0juSR95XzPNTmoZxt3hZJ0a9L0ibmzuFon3ZBevrIFwKGja/64)![](http://wx.qlogo.cn/mmopen/mvkWabd1wdNnFDeBOpqgBt18JTrbXFIXTeh6Gb0g52v1O1Nyib4MsXZgx1ndc4QczJPkGS7MwfHN9zhzkPbib02orYlTvBRVMy/64)![](http://wx.qlogo.cn/mmopen/6LtnejF3Tb49GCPWJjuficQVh9kAePmJU8oDe6hy7Txh2ib6icfhTeeKQDQBqzriaWicK0ojyPFErjBlzMYJrYdjCdr6c7IsME3nr/64)![](http://wx.qlogo.cn/mmopen/6LtnejF3Tb6HKzSwPeqcyvsLaOxsNdQL3dOQBz2JMEJUHXmTlNqWVhKySdy5K7icXarINWsOribnU5iaZ9PLFFoR79RmQsHibYE9/64)![](http://wx.qlogo.cn/mmopen/ZwL47XUG5IibanyUPBwHlOgiaer8YLNnQlic5QvWp6d2CMLZhL6qQOIxtA9xM8H3X1RMnS3IQoAHhmzzPAfzx9qmh1QibTBhMpXQ/64)![](http://wx.qlogo.cn/mmopen/ZwL47XUG5IicterfBEHk2DZXyzFXr59iaXFxUucbmF5S4yf9NYL7t3chPxhiamtzr75pbYYHovm3qVNsoCG8RQxKNVr3rnbmibedPXotnZa7Uh9ic3ib4jQ0iaWVpTGeHKwy70u/64)![](http://wx.qlogo.cn/mmopen/mvkWabd1wdOpzYOc4Fg3jqbTIh8pWic2L7TQlr6tOicfPhcOrW3A7nIOGJqBllj29aXJ84596iarmztibwHg5riczefgcB9FpPQlV/64)![](http://wx.qlogo.cn/mmopen/mvkWabd1wdPGA8yziaNm8BmEK4yM0XovqN44CnlHU8YhOwYXO2vORpncq79zT8U9qfgYLY5ZgYHuvqkkbr0Stfcia8lhiaaaQN2/64)![](http://wx.qlogo.cn/mmopen/mvkWabd1wdMngFuwxIwgGv9vd6USjuDLwceQDU02AYw3bVQyGWKvvaia9nTdVnov6l2cIvcG4Wc6WRrI0H2cPFcPfqSxz9mwT/64)![](http://wx.qlogo.cn/mmopen/QzPtZqIzA0naYte5NEarydInbRMexEFA3CeCoxJN5iaia2NlGDKsNUFJiatrB54ibhxpvt9n08e3IDEOPkZYdRyForMxMVG4oABA/64)![](http://wx.qlogo.cn/mmopen/Q3auHgzwzM7nMuew5GERK3ibcOfTlMqDXeosWclsQ7aqUeC63pal6cvAF2XOcvibopNfZPicA9aDUw3KA1ytl9XFg/64)![](http://wx.qlogo.cn/mmopen/QzPtZqIzA0naYte5NEarybVSFGTFY7CqkdkklkU0VXEtEv1TujcSaee4hHMc1qagxTOmwktZ206DebekDsOWKONCaL14ibMm1/64)

粉丝问答29

所有原创240

科普娱乐88

Linux驱动71

粉丝问答 · 目录

上一篇Linux驱动|cdev_init、cdev_alloc区别

Read more

Reads 18.1k

​

Comment

**留言 83**

- Voyager

  2021年9月22日

  Like39

  向彭老师学习！微信➕了不少大佬，而彭老师是我最佩服的之一，业余时间努力学习，还在群里引导解答大家的疑问，简直是人生楷模！

  Pinned

  一口Linux

  Author2021年9月22日

  Like23

  不给你置顶就实在说不过去了。

- 一口Linux-彭

  2021年9月22日

  Like31

  原创不易，感谢各位的支持！解压密码yikoupeng

  Pinned

- xiaoY

  2021年9月22日

  Like15

  有道云笔记可以分享么![[皱眉]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)

  一口Linux

  Author2021年9月22日

  Like23

  那是我最宝贵的精华![[难过]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)![[难过]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)

- 在路上

  2021年9月22日

  Like14

  彭哥，不要你的云笔记。能写个文章谈谈如何做笔记，积累经验吗？感觉做了很多，解决了很多，但回头没多少记得了。

  一口Linux

  Author2021年9月22日

  Like12

  我想想，回头整理下，对于新手应该怎么整理技术笔记

- LDQ_2024

  2021年9月22日

  Like9

  期待有道云笔记![[奸笑]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)

  一口Linux

  Author2021年9月22日

  Like5

  这是要把我最宝贵的东西夺走啊![[流泪]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)

- 良

  2021年9月22日

  Like6

  太多了，杂且难，同等比较，前期薪资还不如互联网的

  一口Linux

  Author2021年9月22日

  Like8

  前期可能还不如我们小区收废品的赚的多，唉![😔](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)

- 孤沙OS

  2021年9月22日

  Like6

  把自己逼到除了往前走没有退路的路上，就学好了，对自己要狠![[得意]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)

  一口Linux

  Author2021年9月22日

  Like

  必须狠！

- 杨源鑫

  2021年9月23日

  Like3

  老彭，我只对你的笔记感兴趣![[呲牙]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)

  一口Linux

  Author2021年9月23日

  Like1

  一大票人加我好友跟我要笔记![[偷笑]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)![[偷笑]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)![[偷笑]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)

  杨源鑫

  2021年9月23日

  Like

  看来咱哥俩好的份子上![[坏笑]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)

  一口Linux

  Author2021年9月23日

  Like2

  其实笔记有水分![[偷笑]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)![[偷笑]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)![[偷笑]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)

- idiot

  2021年9月22日

  Like

  搞驱动的真是痛并快乐着😁

  一口Linux

  Author2021年9月22日

  Like3

  是的，有时候调试几天，最后发现只是改一个寄存器

- Morgan

  2021年9月22日

  Like3

  博主，能介绍一下嵌入式linux应用开发的路线吗

- 蓝白

  2021年9月22日

  Like2

  准备自学Linux驱动驱动开发，向君学习，加油！

  一口Linux

  Author2021年9月22日

  Like2

  加油！B站有驱动入门视频可以学习下。

- Peter

  2021年9月22日

  Like2

  牛逼 我的彭

  一口Linux

  Author2021年9月22日

  Like2

  差peter太远了。。。。继续努力

- 好嘞

  安徽2023年5月17日

  Like

  有道云笔记可以分享一下嘛

  一口Linux

  Author2023年5月17日

  Like1

  太乱，不分享了，不过我也把里面文章都写出来了，看我原创汇总吧

- 满眼星辰

  陕西2022年5月6日

  Like1

  有道云笔记可以分享嘛，太强了![[呲牙]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)![[呲牙]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)

  一口Linux

  Author2022年5月6日

  Like

  那个就不了，老铁![[偷笑]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)![[偷笑]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)

  春夏秋冬

  陕西2022年5月6日

  Like

  ![[抱拳]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)![[抱拳]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)

- 野书

  2021年12月17日

  Like1

  文章大纲很专业！按照思维导图查漏补缺绝对受益匪浅![[呲牙]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)

  一口Linux

  Author2021年12月17日

  Like1

  你不用学了![[偷笑]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)![[偷笑]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)![[偷笑]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)

  野书

  2021年12月17日

  Like1

  还是得学呀，大厂喜欢问;平时不总结，面试就吃大亏了![[呲牙]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)![[呲牙]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)

- zengkefu

  2021年9月23日

  Like

  牛人

  一口Linux

  Author2021年9月23日

  Like1

  我努力成为牛人！![[奋斗]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)![[奋斗]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)

- 风二中

  2021年9月22日

  Like1

  佩服，这需要很多精力和时间

  一口Linux

  Author2021年9月22日

  Like1

  啥也不说了。

- Brain

  2021年9月22日

  Like

  彭老师，整理的贴心了

  一口Linux

  Author2021年9月22日

  Like1

  暖男我！！![[偷笑]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)

- Eason

  2021年9月22日

  Like

  学了屠龙，日常点灯

  一口Linux

  Author2021年9月22日

  Like1

  恭喜你，入门了！

- 果果小师弟

  2021年9月22日

  Like1

  强

- Johnny

  2021年9月22日

  Like1

  前同事，校友![[强]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)

  一口Linux

  Author2021年9月22日

  Like

  好兄弟！

- free

  上海7/23

  Like

  1024

- 刘爱文

  江苏7/22

  Like

  嵌入式思维导图

  一口Linux

  Author7/22

  Like

  后台回复，老铁

- AB-24601

  上海7/21

  Like

  学好英语很重要 各位想成为Linux大牛的一定要学好英语

- AB-24601

  上海7/21

  Like

  看的书质量太差了

- 薛国斌

  北京5/10

  Like

  嵌入式思维导图

  一口Linux

  Author5/10

  Like

  后台回复，老铁

- 生命在于运动

  广东2023年6月25日

  Like

  真的用心

- clc.

  宁夏2023年4月17日

  Like

  老师有没有考虑将笔记传到博客园,这个平台算是国内比较纯粹的技术文章分享平台,不像百度搜的那些引流文章充斥各种垃圾广告。个人只是觉得好的东西更应该被发现也能带来相应的回报

  一口Linux

  Author2023年4月17日

  Like

  全网都有啊，都是一口Linux

- 金凤

  广东2023年3月13日

  Like

  可以拉进微信群吗？![[玫瑰]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)

  一口Linux

  Author2023年3月13日

  Like

  加我好友yikoupeng

- 徐先生

  江苏2022年8月31日

  Like

  学习笔记以后有出售的打算，我第一个买

  一口Linux

  Author2022年8月31日

  Like

  这也能卖吗？？

- 🍀

  湖南2022年7月30日

  Like

  真的佩服老师

- heyuan

  北京2022年6月5日

  Like

  太强了，业余时间就可以做到这些![[捂脸]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)![[捂脸]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)

- Nice

  广东2022年4月30日

  Like

  彭老师，yyds

- Nick

  2022年2月6日

  Like

  求分享下做笔记的方法，我自己也有用印象笔记做笔记 但是时间久了笔记就忘记了，感觉复用率不高

  一口Linux

  Author2022年2月6日

  Like

  没啥特别的，就是做好分类，相同知识点的文章，尽量自己整个一遍，几篇合成一篇，按照自己的理解，忘了可以随时回头去查阅。很多知识点我也是看了就忘，用的多的可能好一些。

- August

  2021年10月2日

  Like

  有QQ群吗

  一口Linux

  Author2021年10月2日

  Like

  只有微信群

- xiangjin。

  2021年9月30日

  Like

  大佬的笔记分享的嘛。

  一口Linux

  Author2021年9月30日

  Like

  给我留点底裤吧。。。

- 鸳鸯台

  2021年9月24日

  Like

  嵌入式思维导图

  一口Linux

  Author2021年9月24日

  Like

  后台，老铁

  一口Linux

  Author2021年9月24日

  Like

  百度云： 链接：https://pan.baidu.com/s/1ttxxx7I9WJlPchiVrMLqVw 提取码：5i85

- Fighter

  2021年9月23日

  Like

  逻辑被你说复杂了，本来可以说清楚一些东西的！

- 赵琛

  2021年9月22日

  Like

  ![[强]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)![[强]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)![[强]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)![[强]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)![[强]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)![[强]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)![[强]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)![[强]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)![[强]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)

- 吾思

  2021年9月22日

  Like

  rar文件被报检测到病毒

  一口Linux

  Author2021年9月22日

  Like

  不是吧，就一个pdf

- 情报小哥

  2021年9月22日

  Like

  贴心的彭哥～

  一口Linux

  Author2021年9月22日

  Like

  对老弟也必须贴心。

- Lucas

  2021年9月22日

  Like

  yyds

- 仲一

  2021年9月22日

  Like

  彭老师，太强了![[强]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)

  一口Linux

  Author2021年9月22日

  Like

  感谢老弟！不强的

- 科岩

  2021年9月22日

  Like

  一口君yyds，学习了！

- D&F

  2021年9月22日

  Like

  牛逼，太给力了，很清晰

- MiXiGe

  2021年9月22日

  Like

  学习笔记能分享下嘛

- 墨

  2021年9月22日

  Like

  牛蛙

- 唧唧复唧唧

  2021年9月22日

  Like

  彭老师yyds![[色]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)

已无更多数据

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/icRxcMBeJfc8535w2vKlsLPf5hwdMjpYrzuVCHx0rcQmvv8rYqTFtIyic5qErtciaibqaIOWgeKkDsOMeae4HciaUaw/300?wx_fmt=png&wxfrom=18)

一口Linux

16813548

83

Comment

**留言 83**

- Voyager

  2021年9月22日

  Like39

  向彭老师学习！微信➕了不少大佬，而彭老师是我最佩服的之一，业余时间努力学习，还在群里引导解答大家的疑问，简直是人生楷模！

  Pinned

  一口Linux

  Author2021年9月22日

  Like23

  不给你置顶就实在说不过去了。

- 一口Linux-彭

  2021年9月22日

  Like31

  原创不易，感谢各位的支持！解压密码yikoupeng

  Pinned

- xiaoY

  2021年9月22日

  Like15

  有道云笔记可以分享么![[皱眉]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)

  一口Linux

  Author2021年9月22日

  Like23

  那是我最宝贵的精华![[难过]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)![[难过]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)

- 在路上

  2021年9月22日

  Like14

  彭哥，不要你的云笔记。能写个文章谈谈如何做笔记，积累经验吗？感觉做了很多，解决了很多，但回头没多少记得了。

  一口Linux

  Author2021年9月22日

  Like12

  我想想，回头整理下，对于新手应该怎么整理技术笔记

- LDQ_2024

  2021年9月22日

  Like9

  期待有道云笔记![[奸笑]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)

  一口Linux

  Author2021年9月22日

  Like5

  这是要把我最宝贵的东西夺走啊![[流泪]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)

- 良

  2021年9月22日

  Like6

  太多了，杂且难，同等比较，前期薪资还不如互联网的

  一口Linux

  Author2021年9月22日

  Like8

  前期可能还不如我们小区收废品的赚的多，唉![😔](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)

- 孤沙OS

  2021年9月22日

  Like6

  把自己逼到除了往前走没有退路的路上，就学好了，对自己要狠![[得意]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)

  一口Linux

  Author2021年9月22日

  Like

  必须狠！

- 杨源鑫

  2021年9月23日

  Like3

  老彭，我只对你的笔记感兴趣![[呲牙]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)

  一口Linux

  Author2021年9月23日

  Like1

  一大票人加我好友跟我要笔记![[偷笑]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)![[偷笑]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)![[偷笑]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)

  杨源鑫

  2021年9月23日

  Like

  看来咱哥俩好的份子上![[坏笑]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)

  一口Linux

  Author2021年9月23日

  Like2

  其实笔记有水分![[偷笑]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)![[偷笑]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)![[偷笑]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)

- idiot

  2021年9月22日

  Like

  搞驱动的真是痛并快乐着😁

  一口Linux

  Author2021年9月22日

  Like3

  是的，有时候调试几天，最后发现只是改一个寄存器

- Morgan

  2021年9月22日

  Like3

  博主，能介绍一下嵌入式linux应用开发的路线吗

- 蓝白

  2021年9月22日

  Like2

  准备自学Linux驱动驱动开发，向君学习，加油！

  一口Linux

  Author2021年9月22日

  Like2

  加油！B站有驱动入门视频可以学习下。

- Peter

  2021年9月22日

  Like2

  牛逼 我的彭

  一口Linux

  Author2021年9月22日

  Like2

  差peter太远了。。。。继续努力

- 好嘞

  安徽2023年5月17日

  Like

  有道云笔记可以分享一下嘛

  一口Linux

  Author2023年5月17日

  Like1

  太乱，不分享了，不过我也把里面文章都写出来了，看我原创汇总吧

- 满眼星辰

  陕西2022年5月6日

  Like1

  有道云笔记可以分享嘛，太强了![[呲牙]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)![[呲牙]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)

  一口Linux

  Author2022年5月6日

  Like

  那个就不了，老铁![[偷笑]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)![[偷笑]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)

  春夏秋冬

  陕西2022年5月6日

  Like

  ![[抱拳]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)![[抱拳]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)

- 野书

  2021年12月17日

  Like1

  文章大纲很专业！按照思维导图查漏补缺绝对受益匪浅![[呲牙]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)

  一口Linux

  Author2021年12月17日

  Like1

  你不用学了![[偷笑]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)![[偷笑]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)![[偷笑]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)

  野书

  2021年12月17日

  Like1

  还是得学呀，大厂喜欢问;平时不总结，面试就吃大亏了![[呲牙]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)![[呲牙]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)

- zengkefu

  2021年9月23日

  Like

  牛人

  一口Linux

  Author2021年9月23日

  Like1

  我努力成为牛人！![[奋斗]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)![[奋斗]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)

- 风二中

  2021年9月22日

  Like1

  佩服，这需要很多精力和时间

  一口Linux

  Author2021年9月22日

  Like1

  啥也不说了。

- Brain

  2021年9月22日

  Like

  彭老师，整理的贴心了

  一口Linux

  Author2021年9月22日

  Like1

  暖男我！！![[偷笑]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)

- Eason

  2021年9月22日

  Like

  学了屠龙，日常点灯

  一口Linux

  Author2021年9月22日

  Like1

  恭喜你，入门了！

- 果果小师弟

  2021年9月22日

  Like1

  强

- Johnny

  2021年9月22日

  Like1

  前同事，校友![[强]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)

  一口Linux

  Author2021年9月22日

  Like

  好兄弟！

- free

  上海7/23

  Like

  1024

- 刘爱文

  江苏7/22

  Like

  嵌入式思维导图

  一口Linux

  Author7/22

  Like

  后台回复，老铁

- AB-24601

  上海7/21

  Like

  学好英语很重要 各位想成为Linux大牛的一定要学好英语

- AB-24601

  上海7/21

  Like

  看的书质量太差了

- 薛国斌

  北京5/10

  Like

  嵌入式思维导图

  一口Linux

  Author5/10

  Like

  后台回复，老铁

- 生命在于运动

  广东2023年6月25日

  Like

  真的用心

- clc.

  宁夏2023年4月17日

  Like

  老师有没有考虑将笔记传到博客园,这个平台算是国内比较纯粹的技术文章分享平台,不像百度搜的那些引流文章充斥各种垃圾广告。个人只是觉得好的东西更应该被发现也能带来相应的回报

  一口Linux

  Author2023年4月17日

  Like

  全网都有啊，都是一口Linux

- 金凤

  广东2023年3月13日

  Like

  可以拉进微信群吗？![[玫瑰]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)

  一口Linux

  Author2023年3月13日

  Like

  加我好友yikoupeng

- 徐先生

  江苏2022年8月31日

  Like

  学习笔记以后有出售的打算，我第一个买

  一口Linux

  Author2022年8月31日

  Like

  这也能卖吗？？

- 🍀

  湖南2022年7月30日

  Like

  真的佩服老师

- heyuan

  北京2022年6月5日

  Like

  太强了，业余时间就可以做到这些![[捂脸]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)![[捂脸]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)

- Nice

  广东2022年4月30日

  Like

  彭老师，yyds

- Nick

  2022年2月6日

  Like

  求分享下做笔记的方法，我自己也有用印象笔记做笔记 但是时间久了笔记就忘记了，感觉复用率不高

  一口Linux

  Author2022年2月6日

  Like

  没啥特别的，就是做好分类，相同知识点的文章，尽量自己整个一遍，几篇合成一篇，按照自己的理解，忘了可以随时回头去查阅。很多知识点我也是看了就忘，用的多的可能好一些。

- August

  2021年10月2日

  Like

  有QQ群吗

  一口Linux

  Author2021年10月2日

  Like

  只有微信群

- xiangjin。

  2021年9月30日

  Like

  大佬的笔记分享的嘛。

  一口Linux

  Author2021年9月30日

  Like

  给我留点底裤吧。。。

- 鸳鸯台

  2021年9月24日

  Like

  嵌入式思维导图

  一口Linux

  Author2021年9月24日

  Like

  后台，老铁

  一口Linux

  Author2021年9月24日

  Like

  百度云： 链接：https://pan.baidu.com/s/1ttxxx7I9WJlPchiVrMLqVw 提取码：5i85

- Fighter

  2021年9月23日

  Like

  逻辑被你说复杂了，本来可以说清楚一些东西的！

- 赵琛

  2021年9月22日

  Like

  ![[强]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)![[强]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)![[强]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)![[强]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)![[强]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)![[强]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)![[强]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)![[强]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)![[强]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)

- 吾思

  2021年9月22日

  Like

  rar文件被报检测到病毒

  一口Linux

  Author2021年9月22日

  Like

  不是吧，就一个pdf

- 情报小哥

  2021年9月22日

  Like

  贴心的彭哥～

  一口Linux

  Author2021年9月22日

  Like

  对老弟也必须贴心。

- Lucas

  2021年9月22日

  Like

  yyds

- 仲一

  2021年9月22日

  Like

  彭老师，太强了![[强]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)

  一口Linux

  Author2021年9月22日

  Like

  感谢老弟！不强的

- 科岩

  2021年9月22日

  Like

  一口君yyds，学习了！

- D&F

  2021年9月22日

  Like

  牛逼，太给力了，很清晰

- MiXiGe

  2021年9月22日

  Like

  学习笔记能分享下嘛

- 墨

  2021年9月22日

  Like

  牛蛙

- 唧唧复唧唧

  2021年9月22日

  Like

  彭老师yyds![[色]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)

已无更多数据
