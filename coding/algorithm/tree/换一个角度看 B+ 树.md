# 

原创 小林coding 小林coding

_2021年12月14日 17:16_

大家好，我是小林。

大家背八股文的时候，都知道 MySQL 里 InnoDB 存储引擎是采用 B+ 树来组织数据的。

这点没错，但是大家知道 B+ 树里的节点里存放的是什么呢？查询数据的过程又是怎样的？

这次，我们**从数据页的角度看 B+ 树**，看看每个节点长啥样。

![图片](https://mmbiz.qpic.cn/mmbiz_png/J0g14CUwaZdgV8QoMTlAsFLK9yrPWQiapa9WjTibSAwqDlFk48PtzKiaY3UEcsWxIv8lcBfBqhcXH9qqB0sGZ5NFA/640?wx_fmt=png&wxfrom=13&tp=wxpic)

### InnoDB 是如何存储数据的？

MySQL 支持多种存储引擎，不同的存储引擎，存储数据的方式也是不同的，我们最常使用的是 InnoDB 存储引擎，所以就跟大家图解下InnoDB 是如何存储数据的。

记录是按照行来存储的，但是数据库的读取并不以「行」为单位，否则一次读取（也就是一次 I/O 操作）只能处理一行数据，效率会非常低。

因此，**InnoDB 的数据是按「数据页」为单位来读写的**，也就是说，当需要读一条记录的时候，并不是将这个记录本身从磁盘读出来，而是以页为单位，将其整体读入内存。

数据库的 I/O 操作的最小单位是页，**InnoDB 数据页的默认大小是 16KB**，意味着数据库每次读写都是以 16KB 为单位的，一次最少从磁盘中读取 16K 的内容到内存中，一次最少把内存中的 16K 内容刷新到磁盘中。

数据页包括七个部分，结构如下图：
!\[\[Pasted image 20240910131102.png\]\]
!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

这 7 个部分的作用如下图：
!\[\[Pasted image 20240910131108.png\]\]
!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

在 File Header 中有两个指针，分别指向上一个数据页和下一个数据页，连接起来的页相当于一个双向的链表，如下图所示：
!\[\[Pasted image 20240910131114.png\]\]
!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

采用链表的结构是让数据页之间不需要是物理上的连续的，而是逻辑上的连续。

数据页的主要作用是存储记录，也就是数据库的数据，所以重点说一下数据页中的 User Records 是怎么组织数据的。

**数据页中的记录按照「主键」顺序组成单向链表**，单向链表的特点就是插入、删除非常方便，但是检索效率不高，最差的情况下需要遍历链表上的所有节点才能完成检索。

因此，数据页中有一个**页目录**，起到记录的索引作用，就像我们书那样，针对书中内容的每个章节设立了一个目录，想看某个章节的时候，可以查看目录，快速找到对应的章节的页数，而数据页中的页目录就是为了能快速找到记录。

那 InnoDB 是如何给记录创建页目录的呢？页目录与记录的关系如下图：
!\[\[Pasted image 20240910131119.png\]\]
!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

页目录创建的过程如下：

1. 将所有的记录划分成几个组，这些记录包括最小记录和最大记录，但不包括标记为“已删除”的记录；

1. 每个记录组的最后一条记录就是组内最大的那条记录，并且最后一条记录的头信息中会存储该组一共有多少条记录，作为 n_owned 字段（上图中粉红色字段）

1. 页目录用来存储每组最后一条记录的地址偏移量，这些地址偏移量会按照先后顺序存储起来，每组的地址偏移量也被称之为槽（slot），每个槽相当于指针指向了不同组的最后一个记录。

从图可以看到，**页目录就是由多个槽组成的，槽相当于分组记录的索引**。然后，因为记录是按照「主键值」从小到大排序的，所以**我们通过槽查找记录时，可以使用二分法快速定位要查询的记录在哪个槽（哪个记录分组），定位到槽后，再遍历槽内的所有记录，找到对应的记录**，无需从最小记录开始遍历整个页中的记录链表。

以上面那张图举个例子，5 个槽的编号分别为 0，1，2，3，4，我想查找主键为 11 的用户记录：

- 先二分得出槽中间位是 (0+4)/2=2 ，2号槽里最大的记录为 8。因为 11 > 8，所以需要从 2 号槽后继续搜索记录；

- 再使用二分搜索出 2 号和 4 槽的中间位是 (2+4)/2= 3，3 号槽里最大的记录为 12。因为 11 \< 12，所以主键为 11 的记录在 3 号槽里；

- 再从 3 号槽指向的主键值为 9 记录开始向下搜索 2 次，定位到主键为 11 的记录，取出该条记录的信息即为我们想要查找的内容。

看到第三步的时候，可能有的同学会疑问，如果某个槽内的记录很多，然后因为记录都是单向链表串起来的，那这样在槽内查找某个记录的时间复杂度不就是 O(n) 了吗？

这点不用担心，InnoDB 对每个分组中的记录条数都是有规定的，槽内的记录就只有几条：

- 第一个分组中的记录只能有 1 条记录；

- 最后一个分组中的记录条数范围只能在 1-8 条之间；

- 剩下的分组中记录条数范围只能在 4-8 条之间。

### B+ 树是如何进行查询的？

上面我们都是在说一个数据页中的记录检索，因为一个数据页中的记录是有限的，且主键值是有序的，所以通过对所有记录进行分组，然后将组号（槽号）存储到页目录，使其起到索引作用，通过二分查找的方法快速检索到记录在哪个分组，来降低检索的时间复杂度。

但是，当我们需要存储大量的记录时，就需要多个数据页，这时我们就需要考虑如何建立合适的索引，才能方便定位记录所在的页。

为了解决这个问题，**InnoDB 采用了 B+ 树作为索引**。磁盘的 I/O 操作次数对索引的使用效率至关重要，因此在构造索引的时候，我们更倾向于采用“矮胖”的 B+ 树数据结构，这样所需要进行的磁盘 I/O 次数更少，而且 B+ 树 更适合进行关键字的范围查询。

更详细的为什么采用 B+ 树作为索引的原因可以看我之前写的这篇：「[索引为什么能提高查询性能？](https://mp.weixin.qq.com/s?__biz=MzUxODAzNDg4NQ==&mid=2247486903&idx=1&sn=1c6ca6817baa255265fc25dd69551179&scene=21#wechat_redirect)」

InnoDB 里的 B+ 树中的**每个节点都是一个数据页**，结构示意图如下：
!\[\[Pasted image 20240910131129.png\]\]
!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

通过上图，我们看出  B+ 树的特点：

- 只有叶子节点（最底层的节点）才存放了数据，非叶子节点（其他上层节）仅用来存放目录项作为索引。

- 非叶子节点分为不同层次，通过分层来降低每一层的搜索量；

- 所有节点按照索引键大小排序，构成一个双向链表，便于范围查询；

我们再看看 B+ 树如何实现快速查找主键为 6 的记录，以上图为例子：

- 从根节点开始，通过二分法快速定位到符合页内范围包含查询值的页，因为查询的主键值为 6，在\[1, 7)范围之间，所以到页 30 中查找更详细的目录项；

- 在非叶子节点（页30）中，继续通过二分法快速定位到符合页内范围包含查询值的页，主键值大于 5，所以就到叶子节点（页16）查找记录；

- 接着，在叶子节点（页16）中，通过槽查找记录时，使用二分法快速定位要查询的记录在哪个槽（哪个记录分组），定位到槽后，再遍历槽内的所有记录，找到主键为 6 的记录。

可以看到，在定位记录所在哪一个页时，也是通过二分法快速定位到包含该记录的页。定位到该页后，又会在该页内进行二分法快速定位记录所在的分组（槽号），最后在分组内进行遍历查找。

### 聚集索引和二级索引

另外，索引又可以分成聚集索引和非聚集索引（二级索引），它们区别就在于叶子节点存放的是什么数据：

- 聚集索引的叶子节点存放的是实际数据，所有完整的用户记录都存放在聚集索引的叶子节点；

- 二级索引的叶子节点存放的是主键值，而不是实际数据。

因为表的数据都是存放在聚集索引的叶子节点里，所以 InnoDB 存储引擎一定会为表创建一个聚集索引，且由于数据在物理上只会保存一份，所以聚簇索引只能有一个。

InnoDB 在创建聚簇索引时，会根据不同的场景选择不同的列作为索引：

- 如果有主键，默认会使用主键作为聚簇索引的索引键；

- 如果没有主键，就选择第一个不包含 NULL 值的唯一列作为聚簇索引的索引键；

- 在上面两个都没有的情况下，InnoDB 将自动生成一个隐式自增 id 列作为聚簇索引的索引键；

一张表只能有一个聚簇索引，那为了实现非主键字段的快速搜索，就引出了二级索引（非聚簇索引/辅助索引），它也是利用了 B+ 树的数据结构，但是二级索引的叶子节点存放的是主键值，不是实际数据。

二级索引的 B+ 树如下图，数据部分为主键值：
!\[\[Pasted image 20240910131137.png\]\]
!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

因此，**如果某个查询语句使用了二级索引，但是查询的数据不是主键值，这时在二级索引找到主键值后，需要去聚簇索引中获得数据行，这个过程就叫作「回表」，也就是说要查两个 B+ 树才能查到数据。不过，当查询的数据是主键值时，因为只在二级索引就能查询到，不用再去聚簇索引查，这个过程就叫作「索引覆盖」，也就是只需要查一个 B+ 树就能找到数据。**

### 总结

InnoDB 的数据是按「数据页」为单位来读写的，默认数据页大小为 16 KB。每个数据页之间通过双向链表的形式组织起来，物理上不连续，但是逻辑上连续。

数据页内包含用户记录，每个记录之间用单项链表的方式组织起来，为了加快在数据页内高效查询记录，设计了一个页目录，页目录存储各个槽（分组），且主键值是有序的，于是可以通过二分查找法的方式进行检索从而提高效率。

为了高效查询记录所在的数据页，InnoDB 采用 b+ 树作为索引，每个节点都是一个数据页。

如果叶子节点存储的是实际数据的就是聚簇索引，一个表只能有一个聚簇索引；如果叶子节点存储的不是实际数据，而是主键值则就是二级索引，一个表中可以有多个二级索引。

在使用二级索引进行查找数据时，如果查询的数据能在二级索引找到，那么就是「索引覆盖」操作，如果查询的数据不在二级索引里，就需要先在二级索引找到主键值，需要去聚簇索引中获得数据行，这个过程就叫作「回表」。

关于索引的内容还有很多，比如索引失效、索引优化等等，这些内容我下次在讲啦！

**图解系列文章：**

[图解文章汇总](http://mp.weixin.qq.com/s?__biz=MzUxODAzNDg4NQ==&mid=2247498715&idx=1&sn=327ef60e378670ac4fd2f1a70a50f0b2&chksm=f98dbf71cefa3667e1720da56a9e32f9c0fa837eae9c84eeeb1dfd6b6ce851f72e51086285ed&scene=21#wechat_redirect)

[计算机基础学习路线](http://mp.weixin.qq.com/s?__biz=MzUxODAzNDg4NQ==&mid=2247498783&idx=1&sn=1475ec7f0ce70b132a83a1c125aff62c&chksm=f98db8b5cefa31a352922736bb11b2a6353d64a6d07ecc49567f6c0c55c4ffdf079d404674d4&scene=21#wechat_redirect)

[小林的图解系统，大曝光！](http://mp.weixin.qq.com/s?__biz=MzUxODAzNDg4NQ==&mid=2247492900&idx=1&sn=2c1d06a667b1e17e6d8caabff2bbb85b&chksm=f98da18ecefa28986109f13d28c1a06f304cb4d897eb2e931e79dc82d0b054ef6a9f7e1e4e91&scene=21#wechat_redirect)

[不鸽了，小林的「图解网络 3.0 」发布！](http://mp.weixin.qq.com/s?__biz=MzUxODAzNDg4NQ==&mid=2247491944&idx=1&sn=b90deba780ae3840668e21127e467b83&chksm=f98da5c2cefa2cd456045e9b2ed92837ed10e4a2c650f463b29ef5d7f8f4d01014d92225acad&scene=21#wechat_redirect)

[为了拿捏 Redis 数据结构，我画了 40 张图（完整版）](http://mp.weixin.qq.com/s?__biz=MzUxODAzNDg4NQ==&mid=2247501112&idx=1&sn=e42b6c61c6747e2c2f3b890ab4e4b844&chksm=f98d8192cefa0884606c5284499d76eeb3966ac2d3de9fbc4a405448313dcf79eb41b7c9501e&scene=21#wechat_redirect)

![](http://mmbiz.qpic.cn/mmbiz_png/J0g14CUwaZfTwwjfpJhXgIrYMgtVcLhQQBVb02clZfKicbxaibSTNJqXe9Zu8ydiavZKJWJAIhKcnD9hBuKU92JZQ/300?wx_fmt=png&wxfrom=19)

**小林coding**

专注图解计算机基础，让天下没有难懂的八股文！刷题网站：xiaolincoding.com

471篇原创内容

公众号

图解MySQL34

图解MySQL · 目录

上一篇聊聊 MySQL 的优化思路下一篇女朋友问我：为什么 MySQL 喜欢 B+ 树？我笑着画了 20 张图

阅读 1.1万

​

写留言

**留言 53**

- Foolish

  2021年12月14日

  赞11

  今晚复习的数据结构B+树，刚好小林推了一篇文章，哈哈，小林写文章是真的用心了。

  小林coding

  作者2021年12月14日

  赞7

  可以，有缘了

- 竹子

  2021年12月14日

  赞6

  如果一行数据大小超过了16k会是怎么样的过程呢？

  小林coding

  作者2021年12月14日

  赞3

  插入失败。。。数据页的大小也是可以设置的，通过设置innodb_page_size这个变量

- Z.

  2021年12月14日

  赞5

  小林快出版图解书吧，太棒了

- 🏔E\`

  2021年12月14日

  赞5

  高产似...![[哇]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![[强]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

- castle🏀

  2021年12月14日

  赞1

  “再从 3 号槽指向的主键值为 9 记录开始向下搜索 2 次，定位到主键为 11 的记录，取出该条记录的信息即为我们想要查找的内容” 小林哥，3号槽指向的不应该是第3组的最后一条记录吗，应该没法定位到主键值为9的记录吧![[尴尬]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)我理解这里应该是通过2号槽所指的记录（第3组第一条记录的前一条记录）来定位的，不知道对不对

  小林coding

  作者2021年12月14日

  赞3

  嗯嗯，你说的是正常的，这里我写的不对。槽是记录是分组内主键值最大的记录。所以需要通过2号槽找到3号槽的第一个记录，然后在遍历找到对应的记录。

- 魔山

  2021年12月14日

  赞1

  varchar是怎么存的？不是说不占固定大小吗。要是存进去特别大的值怎么办，要叶分裂吗？

  小林coding

  作者2021年12月14日

  赞3

  嗯嗯字段的值太大，同一个数据页里存放的记录就越少，容易产生页分裂。

- 晴晴

  2021年12月14日

  赞

  请教，select语句，会判断记录或undo log的trx_id字段来获取可见版本的数据，那仅访问二级索引就能完成的select语句，是如何获取可见版本的数据的呢？貌似二级索引没有trx_id字段吧？

  小林coding

  作者2021年12月14日

  赞1

  是的，二级索引没有trx_id字段，二级索引判断可见性的过程为两步：

  小林coding

  作者2021年12月15日

  赞2

  1、二级索引页中的页头部分有一个名为page_max_trx_id，记录的是当前最新的事务id。当select语句访问某个二级索引记录时，如果readview的min_trx_id大于page_max_trx_id，说明该页面中的所有记录都对该readview可见。否则执行第二步

  小林coding

  作者2021年12月15日

  赞2

  2. 需要进行回表操作再判断可见性。找到对应的聚集索引记录后再按照原本聚集索引判断可见性的方法。

- 快乐星球

  2021年12月14日

  赞2

  槽像是跳表

- 西瓜大西瓜🍉

  2021年12月14日

  赞1

  通过我仔细的观察，封面是摆拍的，塑料封皮都没撕![[旺柴]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

  小林coding

  作者2021年12月14日

  赞2

  电子版看过觉得不错，就买新书来收藏了![[旺柴]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

- JuiceApple

  2021年12月15日

  赞1

  底部相连的b+树和槽感觉都和跳表类似

- shq

  2021年12月15日

  赞

  MySQL不是b\*？

  小林coding

  作者2021年12月15日

  赞1

  b\*是啥？

- 雪碧拔拔凉

  2021年12月15日

  赞

  这些知识都是从哪来的，看书或是看mysql源码。求指教？

  小林coding

  作者2021年12月15日

  赞1

  《mysql技术内幕：innodb存储》-《从根儿上理解mysql》

- B W

  2021年12月15日

  赞

  innodb_page_size 为什么不能设置的大一点，大于16 那这样就能保存更多的数据了，，一直不明白

  小林coding

  作者2021年12月16日

  赞1

  不管是内存中的数据，还是磁盘中的数据，操作系统都是按页（一页大小通常是 4KB来读取的，一次会读一页的数据。如果要读取的数据量超过一页的大小，就会触发多次 IO 操作。所以，innodb的数据页越大，触发的io操作次数就越多。

- mingjie

  2021年12月14日

  赞

  大佬能不能讲一下MySQL的学习路线

  小林coding

  作者2021年12月14日

  赞1

  《sql必知必会》-《mysql技术内幕：innodb存储》-《从根儿上理解mysql》-《高性能mysql》

- Ks

  2021年12月14日

  赞1

  催更redis pdf![[嘿哈]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![[嘿哈]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

  小林coding

  作者2021年12月14日

  赞

  哈哈mysql也有很多读者催，两边都会同时更新

- 专撕狗🐶

  2021年12月14日

  赞1

  刚好今天正在看mysql的b加树，你就发了文章![[呲牙]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

- 阿秀

  2021年12月14日

  赞1

  学到新东西了![[奸笑]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

  小林coding

  作者2021年12月14日

  赞1

  ![[奸笑]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

- 快乐星球

  2021年12月14日

  赞1

  页里面的记录在磁盘上是主键顺序物理存储的吗，也就是主键索引维护成有多大吗

  小林coding

  作者2021年12月14日

  赞1

  嗯嗯，页里的记录并不一定是物理上顺序的，每个记录采用单向链表组织起来，逻辑上是顺序的。插入和删除记录的时候，会维护顺序。

  快乐星球

  2021年12月14日

  赞1

  可以理解成，主键索引是连续的，记录关于主键不一定是连续的

  快乐星球

  2021年12月14日

  赞1

  如果主键是一个很长的字符串，维护成本可能很高吗

  小林coding

  作者2021年12月14日

  赞1

  嗯嗯，一般主键值最好是自增int类型的字段，因为innodb是由索引组织表，数据是安装具体索引的顺序来进行存储的，不建议使用char或者uuid作为主键值。

  快乐星球

  2021年12月14日

  赞1

  看了看 确实物理上不一定连续 一次磁盘io读入内存 再操作内存就快了 没必要做成卡夫卡那个样子

- IMG

  2021年12月14日

  赞1

  小林的文章都非常用心![[强]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

- 赵永伟

  北京2023年3月8日

  赞

  大佬，这个数据页，是专指的叶子节点吗？非叶子节点有无叶目录呢？若是有，那么它的查找方式是否跟叶子节点一样？我看有的文章里管非叶子节点称作索引页，里面没有叶目录，只是单纯的使用二分来查找检索。 比较糊涂，烦请大佬有空指正![[啤酒]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

  小林coding

  作者2023年3月9日

  赞

  数据页不是专指叶子节点，数据页可以保存索引或者记录，查找方式也是一样，页目录方式的二分查找。

- 习惯过了头

  上海2022年10月12日

  赞

  小林大佬，有两个问题想咨询下你，1）数据页的结构，里面有个页号，这个页号我看了\<mysql技术内幕>那本书关于数据页的结构里面也没找到在哪定义的 2)mysql在执行全表扫描，比如查询非索引列，加载数据页的顺序是按照数据页的物理存储顺序加载的么？

  小林coding

  作者2022年10月12日

  赞

  我本文的数据页的结构是参考《myslq是怎么运行的》，建议看这本书，这本书对mysql很多结构都解释比较清楚。

- Mrhandsomeli

  广东2022年7月13日

  赞

  聚簇索引的叶子节点里面的data是包含了mvvc里面所有版本的数据吗？

  小林coding

  作者2022年7月13日

  赞

  不是，版本数据在undo日志里

- 胖火花

  2022年3月31日

  赞

  小林你好，想问下，聚簇索引里，是按照主键值从小到达排序的。那在二级索引中，数据页中的每个记录是按照索引的值从小到大排序的吗。

  小林coding

  作者2022年3月31日

  赞

  嗯嗯是的

- 无为

  2022年1月25日

  赞

  up您好，在有多个数据页时，“从根节点开始，通过二分法快速定位到符合页内范围包含查询值的页” 这个二分法是对由主键值构成单链表做二分吗，还是对页目录做二分？

  小林coding

  作者2022年1月25日

  赞

  页目录二分

- 生活没有如果

  2022年1月13日

  赞

  大佬，小白问两个问题： 1、叶子节点存储的到底是记录本身，还是记录在磁盘的地址呀？网上也没说的太细，但你第二篇好像提到存地址 2、计算机读取数据一般都是一个页4K来读取的，你说Innodb的页16K要做4次IO操作，但是网上为啥说一次IO呢？到底谁是正确的呀![😂](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif) 感觉你和网上说的有出入呢，我越来越懵啦。 希望能回答下呢，谢谢

  小林coding

  作者2022年1月13日

  赞

  1.innodb的叶子节点存的是记录本身，mysam存储引擎存的是记录的地址。 2.如果页在机械硬盘连续存储，那可以以一次的顺序io读完16k，如果是随机存储的，就需要4次随机io。

- dxlong

  2021年12月22日

  赞

  这个公众号认为一个节点大小为\[块\]的大小，节点中存放数据的方式为有序数组 ![[裂开]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif) https://mp.weixin.qq.com/s/oaYH-JqNQKB8n5VrTxwBgg

  小林coding

  作者2021年12月22日

  赞

  他讲的不是innodb的b+树

- 王帅

  2021年12月19日

  赞

  小林越来越棒👍🏻

已无更多数据

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/J0g14CUwaZfTwwjfpJhXgIrYMgtVcLhQQBVb02clZfKicbxaibSTNJqXe9Zu8ydiavZKJWJAIhKcnD9hBuKU92JZQ/300?wx_fmt=png&wxfrom=18)

小林coding

关注

711635

53

写留言

**留言 53**

- Foolish

  2021年12月14日

  赞11

  今晚复习的数据结构B+树，刚好小林推了一篇文章，哈哈，小林写文章是真的用心了。

  小林coding

  作者2021年12月14日

  赞7

  可以，有缘了

- 竹子

  2021年12月14日

  赞6

  如果一行数据大小超过了16k会是怎么样的过程呢？

  小林coding

  作者2021年12月14日

  赞3

  插入失败。。。数据页的大小也是可以设置的，通过设置innodb_page_size这个变量

- Z.

  2021年12月14日

  赞5

  小林快出版图解书吧，太棒了

- 🏔E\`

  2021年12月14日

  赞5

  高产似...![[哇]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![[强]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

- castle🏀

  2021年12月14日

  赞1

  “再从 3 号槽指向的主键值为 9 记录开始向下搜索 2 次，定位到主键为 11 的记录，取出该条记录的信息即为我们想要查找的内容” 小林哥，3号槽指向的不应该是第3组的最后一条记录吗，应该没法定位到主键值为9的记录吧![[尴尬]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)我理解这里应该是通过2号槽所指的记录（第3组第一条记录的前一条记录）来定位的，不知道对不对

  小林coding

  作者2021年12月14日

  赞3

  嗯嗯，你说的是正常的，这里我写的不对。槽是记录是分组内主键值最大的记录。所以需要通过2号槽找到3号槽的第一个记录，然后在遍历找到对应的记录。

- 魔山

  2021年12月14日

  赞1

  varchar是怎么存的？不是说不占固定大小吗。要是存进去特别大的值怎么办，要叶分裂吗？

  小林coding

  作者2021年12月14日

  赞3

  嗯嗯字段的值太大，同一个数据页里存放的记录就越少，容易产生页分裂。

- 晴晴

  2021年12月14日

  赞

  请教，select语句，会判断记录或undo log的trx_id字段来获取可见版本的数据，那仅访问二级索引就能完成的select语句，是如何获取可见版本的数据的呢？貌似二级索引没有trx_id字段吧？

  小林coding

  作者2021年12月14日

  赞1

  是的，二级索引没有trx_id字段，二级索引判断可见性的过程为两步：

  小林coding

  作者2021年12月15日

  赞2

  1、二级索引页中的页头部分有一个名为page_max_trx_id，记录的是当前最新的事务id。当select语句访问某个二级索引记录时，如果readview的min_trx_id大于page_max_trx_id，说明该页面中的所有记录都对该readview可见。否则执行第二步

  小林coding

  作者2021年12月15日

  赞2

  2. 需要进行回表操作再判断可见性。找到对应的聚集索引记录后再按照原本聚集索引判断可见性的方法。

- 快乐星球

  2021年12月14日

  赞2

  槽像是跳表

- 西瓜大西瓜🍉

  2021年12月14日

  赞1

  通过我仔细的观察，封面是摆拍的，塑料封皮都没撕![[旺柴]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

  小林coding

  作者2021年12月14日

  赞2

  电子版看过觉得不错，就买新书来收藏了![[旺柴]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

- JuiceApple

  2021年12月15日

  赞1

  底部相连的b+树和槽感觉都和跳表类似

- shq

  2021年12月15日

  赞

  MySQL不是b\*？

  小林coding

  作者2021年12月15日

  赞1

  b\*是啥？

- 雪碧拔拔凉

  2021年12月15日

  赞

  这些知识都是从哪来的，看书或是看mysql源码。求指教？

  小林coding

  作者2021年12月15日

  赞1

  《mysql技术内幕：innodb存储》-《从根儿上理解mysql》

- B W

  2021年12月15日

  赞

  innodb_page_size 为什么不能设置的大一点，大于16 那这样就能保存更多的数据了，，一直不明白

  小林coding

  作者2021年12月16日

  赞1

  不管是内存中的数据，还是磁盘中的数据，操作系统都是按页（一页大小通常是 4KB来读取的，一次会读一页的数据。如果要读取的数据量超过一页的大小，就会触发多次 IO 操作。所以，innodb的数据页越大，触发的io操作次数就越多。

- mingjie

  2021年12月14日

  赞

  大佬能不能讲一下MySQL的学习路线

  小林coding

  作者2021年12月14日

  赞1

  《sql必知必会》-《mysql技术内幕：innodb存储》-《从根儿上理解mysql》-《高性能mysql》

- Ks

  2021年12月14日

  赞1

  催更redis pdf![[嘿哈]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![[嘿哈]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

  小林coding

  作者2021年12月14日

  赞

  哈哈mysql也有很多读者催，两边都会同时更新

- 专撕狗🐶

  2021年12月14日

  赞1

  刚好今天正在看mysql的b加树，你就发了文章![[呲牙]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

- 阿秀

  2021年12月14日

  赞1

  学到新东西了![[奸笑]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

  小林coding

  作者2021年12月14日

  赞1

  ![[奸笑]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

- 快乐星球

  2021年12月14日

  赞1

  页里面的记录在磁盘上是主键顺序物理存储的吗，也就是主键索引维护成有多大吗

  小林coding

  作者2021年12月14日

  赞1

  嗯嗯，页里的记录并不一定是物理上顺序的，每个记录采用单向链表组织起来，逻辑上是顺序的。插入和删除记录的时候，会维护顺序。

  快乐星球

  2021年12月14日

  赞1

  可以理解成，主键索引是连续的，记录关于主键不一定是连续的

  快乐星球

  2021年12月14日

  赞1

  如果主键是一个很长的字符串，维护成本可能很高吗

  小林coding

  作者2021年12月14日

  赞1

  嗯嗯，一般主键值最好是自增int类型的字段，因为innodb是由索引组织表，数据是安装具体索引的顺序来进行存储的，不建议使用char或者uuid作为主键值。

  快乐星球

  2021年12月14日

  赞1

  看了看 确实物理上不一定连续 一次磁盘io读入内存 再操作内存就快了 没必要做成卡夫卡那个样子

- IMG

  2021年12月14日

  赞1

  小林的文章都非常用心![[强]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

- 赵永伟

  北京2023年3月8日

  赞

  大佬，这个数据页，是专指的叶子节点吗？非叶子节点有无叶目录呢？若是有，那么它的查找方式是否跟叶子节点一样？我看有的文章里管非叶子节点称作索引页，里面没有叶目录，只是单纯的使用二分来查找检索。 比较糊涂，烦请大佬有空指正![[啤酒]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

  小林coding

  作者2023年3月9日

  赞

  数据页不是专指叶子节点，数据页可以保存索引或者记录，查找方式也是一样，页目录方式的二分查找。

- 习惯过了头

  上海2022年10月12日

  赞

  小林大佬，有两个问题想咨询下你，1）数据页的结构，里面有个页号，这个页号我看了\<mysql技术内幕>那本书关于数据页的结构里面也没找到在哪定义的 2)mysql在执行全表扫描，比如查询非索引列，加载数据页的顺序是按照数据页的物理存储顺序加载的么？

  小林coding

  作者2022年10月12日

  赞

  我本文的数据页的结构是参考《myslq是怎么运行的》，建议看这本书，这本书对mysql很多结构都解释比较清楚。

- Mrhandsomeli

  广东2022年7月13日

  赞

  聚簇索引的叶子节点里面的data是包含了mvvc里面所有版本的数据吗？

  小林coding

  作者2022年7月13日

  赞

  不是，版本数据在undo日志里

- 胖火花

  2022年3月31日

  赞

  小林你好，想问下，聚簇索引里，是按照主键值从小到达排序的。那在二级索引中，数据页中的每个记录是按照索引的值从小到大排序的吗。

  小林coding

  作者2022年3月31日

  赞

  嗯嗯是的

- 无为

  2022年1月25日

  赞

  up您好，在有多个数据页时，“从根节点开始，通过二分法快速定位到符合页内范围包含查询值的页” 这个二分法是对由主键值构成单链表做二分吗，还是对页目录做二分？

  小林coding

  作者2022年1月25日

  赞

  页目录二分

- 生活没有如果

  2022年1月13日

  赞

  大佬，小白问两个问题： 1、叶子节点存储的到底是记录本身，还是记录在磁盘的地址呀？网上也没说的太细，但你第二篇好像提到存地址 2、计算机读取数据一般都是一个页4K来读取的，你说Innodb的页16K要做4次IO操作，但是网上为啥说一次IO呢？到底谁是正确的呀![😂](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif) 感觉你和网上说的有出入呢，我越来越懵啦。 希望能回答下呢，谢谢

  小林coding

  作者2022年1月13日

  赞

  1.innodb的叶子节点存的是记录本身，mysam存储引擎存的是记录的地址。 2.如果页在机械硬盘连续存储，那可以以一次的顺序io读完16k，如果是随机存储的，就需要4次随机io。

- dxlong

  2021年12月22日

  赞

  这个公众号认为一个节点大小为\[块\]的大小，节点中存放数据的方式为有序数组 ![[裂开]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif) https://mp.weixin.qq.com/s/oaYH-JqNQKB8n5VrTxwBgg

  小林coding

  作者2021年12月22日

  赞

  他讲的不是innodb的b+树

- 王帅

  2021年12月19日

  赞

  小林越来越棒👍🏻

已无更多数据
