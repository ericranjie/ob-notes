# 

原创 奇伢 奇伢云存储

_2021年12月06日 07:46_

![](https://res.wx.qq.com/op_res/NN_GToMiIjsXzgPzF9-74ZzwR3cA9-fv3o9eWo8f5gQWqx71CmGlY8kFxuIxZaG0TB1bFeMCmh1DGN_pWMRg0A)

**帐号已迁移**

1篇原创内容

公众号

坚持思考，就会很酷

![图片](https://mmbiz.qpic.cn/mmbiz_png/4UtmXsuLoNcrF7fqACzibViaJLDuIfiaQlR98lnyHMaXkV44zcR5sUdZL1picm1gtjic8sgS9zmRibWIg4wnjPrjZagQ/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

![图片](https://mmbiz.qpic.cn/mmbiz_png/gBJOYg4SEEUvEqC5oO30octlCXHubelsesdolc6YJHWRf1C8UTPs3scrsXWFoCHYsntUfbpKFrDicJ5tFbLPxBw/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

关于 Quorum 的两个维度

![图片](https://mmbiz.qpic.cn/mmbiz_png/gBJOYg4SEEUvEqC5oO30octlCXHubelsl8Q8jryjW8fJDBj8r4A48RcHDMj7ibJfRRhRWSJUUWNTqiaibDr9dmstQ/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

前几回说了那么多框架，设计思想的文章。今天分享一个很小的点，etcd 的 quorum 是怎么实现的？

Quorum 机制本质就是一个关于**多数派**的事情，这个**多数派**应用的有两个方面：

1. 选举过程：获得**多数节点投票的节点**才能获胜，成为 Leader ；

1. 运行过程：**被多数节点 commit 的日志**位置，这个才是被集群可靠记录的位置。被集群 commit 的日志才能被应用 apply ；

**那么这里有两个小思考问题：**

1. 既然是选举过程，那怎么**选举结果唱票的**？

1. 既然是运行过程，那集群的这些**节点怎么确认集群的 commit 位置**？

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

有选举自然有唱票

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

唱票是在选举流程中的一个步骤。还记得以前选班干部的时候，在黑板上写“正”字，谁得票多谁就获胜当选。

etcd 里面也有选举，也就是 Leader 的选举。Leader 获胜的依据是的票满足大多数，也就是满足 quorum 机制。

今天我们就来看看 etcd 的唱票是怎么做的？

很简单的思路，我们给每个参与选举的朋友计数，得票超过半数的，那么就胜出。

比如说 A，B，C，D，E 五个人竞选，那么得到 3 票的就可以胜出。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

来看看 etcd 的唱票

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

选举属于 quorum 机制，代码位于 etcd/raft/quorum/ 下。quorum 的核心实现在 MajorityConfig 的结构体，其实就是个 map 的封装：

`type MajorityConfig map[uint64]struct{}   `

这个 map 的 key 是节点的 id，这里面包含了集群的节点，map 的 value 不重要，所用用的是 struct{} 类型。

思考个小问题：那既然 value 不 care ，那为什么不用 slice 结构？

**其实就是为了查找的需求，map 的查找是常数级别，value 又用的 struct{} ，不占空间，一举两得。**

唱票和集群 commit 的实现就是它的方法，我们先看下唱票的实现：

`// etcd/raft/quorum/majority.go   func (c MajorityConfig) VoteResult(votes map[uint64]bool) VoteResult {       // 搞个长度为 2 的数组       ny := [2]int{}       // 遍历集群节点       for id := range c {           v, ok := votes[id]           if !ok {               // 暂时没投票的               missing++               continue           }           if v {               // 投票赞同的               ny[1]++           } else {               // 投票拒绝的               ny[0]++           }       }       q := len(c)/2 + 1       if ny[1] >= q {           // 选举成功：得票数超过半数，，比如 votes => [yes, yes, yes]           return VoteWon       }       if ny[1]+missing >= q {           // 未知情况：不确定成功，也不确定失败           return VotePending       }       // 选举失败       return VoteLost   }   `

唱票的实现很简单，就如下几个步骤：

1. 遍历集群节点；

1. 统计谁赞同了、谁拒绝了、谁还没投票；

1. 唱票的结果有三种：成功，失败，待定；

1. 赞同投票的超过半数（ len(c)/2+1 ），则胜利；

这实现可太简单了，就是一个遍历投票结果，写“正”字，“正”字超过半数则胜出。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

集群的节点怎么确认集群的 commit 位置？

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

集群内被多数节点 commit 的位置才是集群的 commit 点。也就是说这个也需要满足 quorum 。这个就有意思了。

**关键步骤：排序，然后取中间的位置。**

取的这个中间的位置就是满足 quorum 的 commit 。

`// etcd/raft/quorum/majority.go   func (c MajorityConfig) CommittedIndex(l AckedIndexer) Index {       // 遍历集群节点：取出每个节点的 commit       for id := range c {           if idx, ok := l.AckedIndex(id); ok {               srt[i] = uint64(idx)               i--           }       }       // 排个序       insertionSort(srt)          // 取中间，这个位置就是大多数 commit 的位置，属集群共识       pos := n - (n/2 + 1)       return Index(srt[pos])   }   `

这个实现就很有意思了，捞出每个节点当前的 commit 位置，组成一个数组，然后给这个数组排个序，取中间的位置。这个位置就是集群的 commit 位置，也就是 apply 的位置。

先把集群每个节点的 commit 位置取出来，是这样的：

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

后来排个序是这样的，**黑色的节点 commit 位置则是集群的 commit 位置**：

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

总结

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

1. Quorum 机制是分布式系统中很重要的理论部分，这是一个关于**多数派**的机制。etcd 关于多数派有两个方面：Leader 选举和 raft 日志运行；

1. etcd 的唱票实现非常简单，就是一个计数“正”字的实现，用一个 map 记录集群的节点，投票计数超过多数则胜出；

1. etcd 确认集群 commit 位置则是先把**每个节点的 commit 位置放在数组**，然后**排个序**，然后**取中间位置，这个位置就是集群的 commit 位置**；

1. 多数节点 commit 过的日志才是集群 commit 的位置，**集群 commit 的日志才能 apply** ，这个要记住喽；

1. 集群 commit 位置将**由 Leader 通过心跳或者日志复制的消息**告诉其他节点；

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

后记

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**点赞、在看** 是对奇伢最大的支持。

~完～

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

往期推荐

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

往期推荐

\[

云原生 etcd 系列｜深入剖析数据多版本 MVCC 机制

\](http://mp.weixin.qq.com/s?\_\_biz=Mzg3NTU3OTgxOA==&mid=2247496167&idx=1&sn=3396667786aa6b5e412696ee5ca58eac&chksm=cf3de122f84a683465023c8eeb35a772ef919512c0a71246694d52089c3ff3393055d3287297&scene=21#wechat_redirect)

\[

云原生 etcd 系列｜用“租约”给 key 加一个期限！

\](http://mp.weixin.qq.com/s?\_\_biz=Mzg3NTU3OTgxOA==&mid=2247495743&idx=1&sn=119a392f8eb9f0988622f1734f4fec12&chksm=cf3de0faf84a69ecaa5b67b494d04e54530a4b1598b750c6ecbedc7c8d1942880d6c27c9166d&scene=21#wechat_redirect)

\[

云原生 etcd 系列｜存储引擎 boltdb 的设计奥秘？

\](http://mp.weixin.qq.com/s?\_\_biz=Mzg3NTU3OTgxOA==&mid=2247495574&idx=1&sn=b8386da1af752048c3bd5a712c37a178&chksm=cf3dff53f84a76453adb995cab68d201ec75267560e6c8a74601294efd0bf378d9c5201bbb6d&scene=21#wechat_redirect)

\[

云原生 etcd 系列｜快照技术是什么？

\](http://mp.weixin.qq.com/s?\_\_biz=Mzg3NTU3OTgxOA==&mid=2247495207&idx=1&sn=a56573ae0d2c6e5850926a96547f6033&chksm=cf3dfee2f84a77f4a5265802da9a160c034065737268a86dd5a6621ee61df964da94baaf901b&scene=21#wechat_redirect)

坚持思考，方向比努力更重要。**关注我：奇伢云存储。欢迎加我好友，技术交流。**

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**欢迎加我好友，技术交流。**

![](https://res.wx.qq.com/op_res/NN_GToMiIjsXzgPzF9-74ZzwR3cA9-fv3o9eWo8f5gQWqx71CmGlY8kFxuIxZaG0TB1bFeMCmh1DGN_pWMRg0A)

**帐号已迁移**

1篇原创内容

公众号

![](https://mmbiz.qlogo.cn/mmbiz_jpg/iaeiczgvdQ9mN1mmOG1e1BzDmThWc2ibcxCAPr4rJFibqdQzyAKWqdvwaW2rdecibibYD2Cm8F8L7tySbot1DqiaWo0Bw/0?wx_fmt=jpeg)

奇伢

你的支持，是我创作的动力。

![赞赏二维码](https://mp.weixin.qq.com/s?__biz=MzkyOTU5MTc3NQ==&mid=2247499252&idx=1&sn=b5b8eed8553041a45ac4e98ff78ff83b&source=41&key=daf9bdc5abc4e8d038f30b380dca2f26323a3948bfccde859a2476ecd88ce5ee47e09e95cf2a19be2005df94be4176a08802537f742360eeae80ffdb8d6b25b7d73fdc6c7bbc02a96b3ec48afa4450f9691b99a7ca652ef1225fc57e6fa6688ecbff21f51c0ea7133b701711dcec11b37fd0b6bd5ad982fc8870024a63e4c1c5&ascene=0&uin=MTEwNTU1MjgwMw%3D%3D&devicetype=Windows+11+x64&version=63090b19&lang=zh_CN&countrycode=CN&exportkey=n_ChQIAhIQrKZlblsP9WxokKCePm7CCBLmAQIE97dBBAEAAAAAANC%2BKi8isioAAAAOpnltbLcz9gKNyK89dVj01ZI%2BO9YEZ7BBHYdUeOPiEcbRjGZX%2BNVE5cYQbW%2F4G0bBPiMW%2Fv0eDDEZsaudYcMkh1QJXT2wi%2FiN7d0otccO%2Byfd62qfacYZ5rlsnZ8oVDdUaKrEOZ77o9qZOfNCbjaAjZ1sdbGLKtPUR9%2FKIIiQZICdlb79WI8jCnI%2FA%2Bq1w8nJGx8SNr9YAPuLUJRJ0S3JEp0mtd6GPHWQ1pYZDleOR%2B0ti47xow0Ex31UJ5JkuPnQWyrGPvASuP%2FMe2lfYpR4&acctmode=0&pass_ticket=soR9eUpu8Z9aC5w6EgoLR0QGE1VpFfKUAK%2FPEsktOjDTogga2PJMO8GCoGr8N5%2Fp&wx_header=1)钟意作者

1人喜欢

![](http://wx.qlogo.cn/mmopen/tqRiaNianNl1mfvwLAib4gcZnY3PR5oAPQVbbypicWJIJevcYMFzkrTnzk4WZjCo8aoiaZooiaiaiafF0g8MtKjcbAm2XwTvqYJxqc6ZjVgG4wdEAdtNgZ3Cse6PtTMaCRuITTt2/64)

分布式存储16

云原生9

分布式存储 · 目录

上一篇云原生 etcd 系列｜深入剖析数据多版本 MVCC 机制下一篇存储架构｜Haystack 太强了！存 2600 亿图片

阅读 1100

​

写留言

**留言 3**

- 奇伢

  2021年12月6日

  赞3

  Github 源码注释仓库：https://github.com/liqingqiya/readcode-etcd-v3.4.10 。 加我微信： liqingqiya2019，拉你进群，讨论 etcd 更多知识。 也可搜索知识星球: 奇伢云存储，获取更多存储知识。

  置顶

- -太白

  2021年12月6日

  赞

  什么时候apply？谁发起？

  奇伢云存储

  作者2021年12月6日

  赞

  apply 是每个节点自己发起apply。 commit 位置则是 leader 同步给集群其他结点的。

已无更多数据

[](javacript:;)

![](http://mmbiz.qpic.cn/sz_mmbiz_png/Pe6fMET7W1sn1HSod7sy8vX2Hqicfos9njfWcj5GP7Ub5K35kOVgzwZia69byvdHu9B3UjtmZIRIssa0K4wby5eA/300?wx_fmt=png&wxfrom=18)

奇伢云存储

15分享9

3

写留言

**留言 3**

- 奇伢

  2021年12月6日

  赞3

  Github 源码注释仓库：https://github.com/liqingqiya/readcode-etcd-v3.4.10 。 加我微信： liqingqiya2019，拉你进群，讨论 etcd 更多知识。 也可搜索知识星球: 奇伢云存储，获取更多存储知识。

  置顶

- -太白

  2021年12月6日

  赞

  什么时候apply？谁发起？

  奇伢云存储

  作者2021年12月6日

  赞

  apply 是每个节点自己发起apply。 commit 位置则是 leader 同步给集群其他结点的。

已无更多数据
