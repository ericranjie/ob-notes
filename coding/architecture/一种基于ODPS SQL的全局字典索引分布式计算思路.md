原创 云义 阿里云开发者

_2024年07月03日 08:31_ _浙江_

阿里妹导读

本文提供一种能充分利用分布式计算资源来计算全局字典索引的方法，以解决在大数据量下使用上诉方式导致所有数据被分发到单个reducer进行单机排序带来的性能瓶颈。

在一些业务场景中，我们需要将字符串映射为一个整形数字，并确保全局唯一，比如在BitMap字典索引计算场景。常用的方法是将数据集根据需要映射的字符串做全局order by排序，将排序号作为字符串的整形映射。该方式能达到目标，但如果一次要计算的数据量太大（十亿级别或更大），会跑耗时较久甚至跑不出来。﻿

本文提供一种能充分利用分布式计算资源来计算全局字典索引的方法，以解决在大数据量下使用上诉方式导致所有数据被分发到单个reducer进行单机排序带来的性能瓶颈。

order by方式

sort_data数据集中id已经唯一，数据量级9.3亿，直接对数据集全局开窗排序写法如下：

```
INSERT OVERWRITE TABLE test_data_result
```

DAG图：

﻿![图片](https://mmbiz.qpic.cn/mmbiz_jpg/Z6bicxIx5naLyomiaArtXicGveNftVqOPzAG3z2UUXnx2ZficFzEXtYYMc0h8ibH1nfHRia2puyZPJibZERg1Z4v2a9cQ/640?wx_fmt=other&from=appmsg&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)﻿

耗时30分钟，reducer只有一个，这里设置set odps.sql.reducer.instances=256是不起作用的，在这种方式下reducer只能有一个才能保证是全局唯一的。

分布式优化思路

大数据量下计算最忌讳的就是演变为单机计算，好一点的情况是跑很久，数据量再大可能直接跑不出来。如何利用计算资源水平扩展的优势来解决？思路是先做分桶进行分布式局部排序、再基于桶大小得到全局索引。类似于操作系统指令寻址的基址寻址，我们需要对每个id做绝对地址映射，利用一个基址加上相对地址得到绝对地址。

代码参考如下：

```
--第一步
```

1.第一步先对id做hash分桶，然后计算每个桶的大小，桶数量可以根据需要计算的id数量来评估，这里分100000个桶，然后计算出每个id在桶内的相对位置bucket_rel_index，同时计算出桶大小bucket_size；

2.第二步根据桶大小计算每个桶的基址；

3.第三步将桶基址+id在桶内的相对地址得到全局唯一的绝对地址id_index；

﻿![图片](https://mmbiz.qpic.cn/mmbiz_jpg/Z6bicxIx5naLyomiaArtXicGveNftVqOPzAMW5qepbeqEEX8RiaqZkA6vXlMvIA7f8VZeicSozYXtc8oTAIcbtbHiciaQ/640?wx_fmt=other&from=appmsg&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)﻿

优化后避免了单机计算，耗时2分钟。﻿

日常优化记录，也许并非最优，如有其他方式欢迎探讨。

**随需而动：自动弹性，稳定交付**

本方案使用应用型负载均衡（ALB）和弹性伸缩（ESS）智能分配网络流量、动态调整服务器资源，提高应用的高可用性和吞吐量，弹性控制资源利用率、缩减资源成本。快**点击阅读原文**查看详情吧～

阅读原文

阅读 4796

​

收藏此内容的人还喜欢

一不小心掉入了 Java Interface 的陷阱

我常看的号

阿里云开发者

不喜欢

不看的原因

确定

- 内容低质
- 不看此公众号内容

![](https://mmbiz.qpic.cn/mmbiz_jpg/Z6bicxIx5naLL5Pm94K2TQ50LMMmR3T4adbUbCRtZcpe2ZTSSNqIl8BZAtLsogOicgJ6AKYtr9pXgXP4dYcL3eVg/0?wx_fmt=jpeg&tp=wxpic)

深入探讨Java的分层编译

我常看的号

阿里云开发者

不喜欢

不看的原因

确定

- 内容低质
- 不看此公众号内容

![](https://mmbiz.qpic.cn/mmbiz_jpg/Z6bicxIx5naLcrdexmrT5Dc5PftUvDIW8Fpyu8vDAxUkBpuxV31Y50NZWtIJibBApNpVJOauHriamy56T6jblEeww/0?wx_fmt=jpeg&tp=wxpic)

MySQL 8.0：filesort 性能退化的问题分析

我常看的号

阿里云开发者

不喜欢

不看的原因

确定

- 内容低质
- 不看此公众号内容

![](https://mmbiz.qpic.cn/mmbiz_jpg/Z6bicxIx5naKUlu2HvmqF0xcwteNJvQ8xCR7Xla7ibjkBlpSqfpN3fXoEGFwxfCJgMqtn9q771p0olDuqWQJt1rw/0?wx_fmt=jpeg&tp=wxpic)

写留言

**留言 3**

- FriesW

  山东7月3日

  赞1

  老弟写得非常好![[强]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![[强]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

- 简墨成竹

  北京7月3日

  赞

  写的非常好

- 庞兴华

  北京7月3日

  赞

  思路很好，值得拥有！核心的思路是识别了该业务rownumber只是需要一个唯一id，并不是真的需要全局排序，有序不重复，就可以利用分布式排序优化![[强]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

已无更多数据

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/Z6bicxIx5naI1jwOfnA1w4PL2LhwNia76vBRfzqaQVVVlqiaLjmWYQXHsn1FqBHhuGVcxEHjxE9tibBFBjcB352fhQ/300?wx_fmt=png&wxfrom=18)

阿里云开发者

351768

3

写留言

**留言 3**

- FriesW

  山东7月3日

  赞1

  老弟写得非常好![[强]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![[强]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

- 简墨成竹

  北京7月3日

  赞

  写的非常好

- 庞兴华

  北京7月3日

  赞

  思路很好，值得拥有！核心的思路是识别了该业务rownumber只是需要一个唯一id，并不是真的需要全局排序，有序不重复，就可以利用分布式排序优化![[强]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

已无更多数据
