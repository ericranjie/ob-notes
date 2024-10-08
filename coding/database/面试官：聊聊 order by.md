ImportNew

_2024年04月26日 11:30_ _上海_

以下文章来源于飞天小牛肉 ，作者小牛肉

\[

![](http://wx.qlogo.cn/mmhead/Q3auHgzwzM49hOljJBBMfsYWyaz97icat4LTSBOIWibFE1xGB5AjTNibA/0)

**飞天小牛肉**.

InfoQ & 阿里云签约作者，分享原创技术干货和成长经验

\](https://mp.weixin.qq.com/s?\_\_biz=MjM5NzMyMjAwMA==&mid=2651534245&idx=1&sn=7e94b34f01e0823a95b0a8bdc1a9cf0a&chksm=bd245fda8a53d6cccc2f91f629c075fb5bb81db510fcecb42545c8c53067f2dc201860d8f750&mpshare=1&scene=24&srcid=0426suaq7sC3yofVLS0hs1j3&sharer_shareinfo=abf41ec45f31f9a0791daf0a03fe2c5d&sharer_shareinfo_first=abf41ec45f31f9a0791daf0a03fe2c5d&key=daf9bdc5abc4e8d07d984517b77c3b3b039b0acf122f7ff92ce056dde40453cee230ff18f292ba8c298b4d0caf01aa55f645ef554d8443fba07ac1d5f711d2cfe00ff428c985719655a14be687a16d4194ff62aa66b14121b94fadcb6fe084ba48052136df396396bae02f54caecbc7bccca10626656123451aeb67b4298be38&ascene=0&uin=MTEwNTU1MjgwMw%3D%3D&devicetype=Windows+11+x64&version=63090b19&lang=zh_CN&countrycode=CN&exportkey=n_ChQIAhIQXBAKCyEKWSz6AxeR2wvimhLmAQIE97dBBAEAAAAAAA7tA8nb%2FlIAAAAOpnltbLcz9gKNyK89dVj0h1y6QhFJ8z35D838cXKzDGhoZQNBYkk5pHlcchBLxdXV9p1DBir1S31rEooVDkivbpEQ7iODC48818lqSuaZAD4CdRM3MfS1NS1dGNaVcXW1plBndrN%2B6BjpNkSgjl7YZogoeptM9ysFUq3EYK%2Bn%2FmTnb1ryVEV%2FXAtegUPgAtN6xtW9mc56D2Knt9UekutpU%2BoNF0LqAbemehac2zKYBR265ASTM5ZGqaiWOaA6Zp7lxgVrv%2F08isJVfyEjmxGe&acctmode=0&pass_ticket=Pp7TOew1v9C9bw3lx93BVqOR0qGV5%2F7HK8FcseY3PeJvyYz5OFa8w2tePa1my1mv&wx_header=1&fasttmpl_type=0&fasttmpl_fullversion=7350504-zh_CN-zip&fasttmpl_flag=1#)

假设有这么一张用户表 user：

- id int(11)：主键

- username varchar(16)：用户名

- age int(11)：年龄

- city varchar(16)：城市

假设有这么一个需求：查询出城市是 “南京” 的所有用户名，并且按照用户名进行排序，返回前 1000 个人的姓名、年龄。

众所周知，排序使用的关键字是 order by，不难写出这样的 SQL 语句：

```
select city, username, age from user where city = '南京' order by username limit 1000;
```

这篇文章，我们就来解释下，涉及 order by 的语句具体是怎么执行的，以及有什么参数会影响执行的行为。

老规矩，背诵版在文末。

## 

## **全字段排序**

为避免全表扫描，我们在查询条件的 city 字段上面建立索引。然后用 explain 命令来看看这个语句的执行情况：

![图片](https://mmbiz.qpic.cn/mmbiz_png/PocakShgoGHX4ib41XA0rIQ38dRYkx5mecamz8wv3v6vicfmjIZsUgz7ibXWvpichmy6o8mHaFPKWB5T7gYke4ICZA/640?wx_fmt=png&wxfrom=13&tp=wxpic)

偷个懒，因为我其实一条数据也没插入（狗头保命），所以大伙儿在上图中看见的 explain 分析出来的这条 SQL 的影响行数 rows 是 1。

Extra 这个字段中的 Using filesort 表示的就是需要排序，MySQL 会给每个线程分配一块内存用于排序，称为 sort_buffer。

通常情况下，这个语句执行流程如下所示 ：

1. 初始化 sort_buffer，放入 city、username、age 这三个字段；

1. 从索引 city 找到第一个满足 city='南京' 条件的主键 id；

1. 到主键 id 的索引树上查找到对应的整行数据（回表查询），然后取出 city、username、age 三个字段的值，存入 sort_buffer 中；

1. 从索引 city 取下一个记录的主键 id；

1. 重复步骤 3、4 直到 city 的值不满足查询条件为止；

1. 对 sort_buffer 中的数据按照字段 username 做快速排序。

按照字段 username 做快速排序这个动作，可能在内存中完成，也可能需要使用外部排序，这取决于排序所需的内存和 sort_buffer 的大小，由参数 sort_buffer_size 决定。

如果要排序的数据量小于 sort_buffer_size，排序就在内存中完成。但如果排序数据量太大，内存放不下，则就需要利用磁盘临时文件来辅助排序。

解释下这里使用磁盘临时文件来进行辅助排序的含义，外部排序常用的排序算法是多路归并排序算法，具体步骤如下：

- 到主键 id 索引树上查找到对应的整行数据后，取 city、username、age 三个字段的值，存入 sort_buffer 中，能存多少是多少，当 sort_buffer 快要满时，就对 sort_buffer 中的数据进行排序，排完后，把数据临时放到磁盘的一个小文件中，然后清空 sort_buffer（这样的话，一个很大的数据，就会被分成若干个临时磁盘文件）

- 继续回到主键 id 索引树取数据，重复上一步，直到取出所有满足条件的数据

- 最后，归并已经有序的若干个临时磁盘文件，形成一个完整的有序大文件

7. 按照排序结果取前 1000 行返回给客户端

可以看出，整个排序过程，我们要查询的 city、username、age 全都参与了，所以，暂且把这个排序过程，称为全字段排序

整条语句的执行流程的示意图如下所示：

![图片](https://mmbiz.qpic.cn/mmbiz_png/PocakShgoGHX4ib41XA0rIQ38dRYkx5mebdsAyZ8OsWZp0PIicerqe8Iar1aR4Jib9Jxdia52rULBXperzRgicb8efg/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

针对上面利用磁盘临时文件进行辅助排序的过程，不知道大家会不会有个很自然的想法：sort_buffer 内存放不下，需要用到临时磁盘文件，磁盘文件越多，排序效率显然就会越低下。那为什么还要把排序不相关的字段 city、username 放到 sort_buffer 中呢？只存放排序相关的 age 字段，这样划分的磁盘文件不就相对变少了嘛~

这就是 rowid 排序 👇

## **rowid 排序**

rowid 排序，听名字大概就能理解，就是，只把需要用于排序的字段和对应的主键 id，放到 sort_buffer 中。

那怎么确定走的是全字段排序还是 rowid 排序呢？

实际上有个参数控制的。这个参数就是 max_length_for_sort_data，是 MySQL 中专门控制用于排序的行数据的长度的一个参数。它的意思是，如果单行的长度超过这个值，MySQL 就认为单行太大（那么数据量肯定就越大，sort_buffer 可能不够用），不能再像之前那样把所有 select 的字段都存进 sort_buffer 了，要换一个算法，只存排序相关的字段

```
show variables like 'max_length_for_sort_data';
```

![图片](https://mmbiz.qpic.cn/mmbiz_png/PocakShgoGHX4ib41XA0rIQ38dRYkx5me6EiaYHwYDbUwgUAzgJv5BM5j7AiaBoP93Mzbg6mfreben0Sc5YW41Miag/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

可以看到，max_length_for_sort_data 的默认值是 1024。

可以通过下面这行命令进行修改

```
SET max_length_for_sort_data = 16;
```

表中我们定义的这三个字段 city、username、age 的总长度是 36，我把 max_length_for_sort_data 设置为 16，显然，单行的长度已经超过这个值了，排序算法应该由全字段排序转成了 rowid 排序。

整个执行流程就变成如下所示的样子：

1. 初始化 sort_buffer，放入两个字段，即 username 和主键 id；

1. 从 city 索引中找到第一个满足 city='南京' 条件的主键 id；

1. 到主键 id 的索引树上查找到对应的整行数据（回表查询），取出 username 和 id 这两个字段，存入 sort_buffer 中；

1. 从 city 索引中取下一个记录的主键 id；重复步骤 3、4 直到不满足 city='南京' 的条件为止；

1. 对 sort_buffer 中的数据按照字段 username 进行排序；

1. 遍历排序结果，取前 1000 行，并按照 id 的值回到主键 id 的索引树中取出 city、username 和 age 三个字段返回给客户端。

可以看到，新的 rowid 算法放入 sort_buffer 的字段，只有要排序的列（即 username 字段）和主键 id。但有利有弊，存放在 sort_buffer 中的数据因为少了 city 和 age 字段的值，所以不能直接返回给客户端了，需要再进行一次回表查询。

这个执行流程的示意图如下：

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

从上面我们可以看出来，事实上，如果内存足够大的话，MySQL 优先选择的仍然是全字段排序，把需要的字段都放到 sort_buffer 中，这样排序后就会直接从内存里面返回查询结果了，不用再回表查询，减少磁盘访问。

回表的话应该首先去缓冲池 Buffer Pool 中找到对应版本的数据，若找不到，则需要进行磁盘读（索引文件是磁盘文件），理论上不会触发磁盘读，因为取 id 的时候已经从磁盘读取了一次放到了缓冲池 Buffer Pool 中了，但不排除，第一次取完数据放到 sort buffer 后缓存中的数据页被淘汰了，可能会触发磁盘读。

## 

## **order by 优化**

很显然，如果不排序就能得到正确的结果，那对系统的消耗会小很多，语句的执行时间也会变得更短。

那么，是不是所有的 order by 都需要排序操作呢？

并不是！

从上面分析的执行过程我们可以看到，MySQL 之所以需要 sort_buffer，并且在 sort_buffer 上做排序操作，其原因是原来的数据都是无序的。

回顾下我们的需求：查询出 city 是 “南京” 的所有 username，并且按照 username 进行排序，返回前 1000 个人的姓名、年龄。

那，如果能够保证从 city 这个索引上取出来的数据行，已经天然就是按照 username 进行递增排序的话，不就不用再排序了吗

所以，我们可以在这张表上创建一个 city 和 username 的联合索引：

```
alter table user add index idx_city_username(city, username);
```

在这个联合索引上，我们依然可以用树搜索的方式定位到第一个满足 city='南京' 的记录，并且额外确保了，接下来按顺序取 “下一条记录” 的遍历过程中，只要 city 的值是南京，username 的值就一定是有序的（不清楚的小伙伴可以回看下联合索引相关的知识）。

这样整个查询过程的流程就变成了：

1. 从联合索引 (city, username) 上找到第一个满足 city='南京' 条件的主键 id；

1. 到主键 id 的索引树上查找到对应的整行数据（回表查询），取出 username、city 和 age 这三个字段的值，作为结果集的一部分直接返回；

1. 从联合索引 (city, username) 上取下一个记录主键 id；

1. 重复步骤 2、3，直到查到第 1000 条记录，或者是不满足 city='南京' 条件时循环结束。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

可以看到，这个查询过程不需要 sort_buffer，也不需要排序，整个流程被大大缩短了。

再用 explain 分析下这条语句：

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

从图中可以看到，Extra 字段中没有 Using filesort 了，也就是不需要排序了。

而且由于 (city,username) 这个联合索引本身有序，所以这个查询也不用把 4000 行全都读一遍，只要找到满足条件的前 1000 条记录就可以退出了。也就是说，在我们这个例子里，只需要扫描 1000 次就可以了。

说到这里，不知道有没有小伙伴能够察觉点什么

回表查询！

是的，说了这么多，回表查询这个东西一直都在啊，完全可以用上 覆盖索引 来去掉回表过程啊~

不就是要回表取出 username、city 和 age 这三个字段的值吗，咱就直接创建一个 city、name 和 age 的联合索引，对应的 SQL 语句就是：

```
alter table user add index idx_city_username_age(city, username, age);
```

这样，整个流程就被进一步简化：

1. 从联合索引 (city, username, age) 树上找到第一个满足 city='南京' 条件的记录，把这条记录作为结果集的一部分直接返回；

1. 从联合索引 (city, username, age) 树上取下一个记录，同样将这条记录作为结果集的一部分直接返回

1. 重复执行步骤 2，直到查到第 1000 条记录，或者是不满足 city='南京' 条件时循环结束。

如下图所示：

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

当然了，使用覆盖索引性能上会快很多，但是索引的维护也是需要代价的，这里需要自己做一个权衡取舍~

最后放上这道题的背诵版：

面试官：SQL 优化了解过吗？

😎 小牛肉：我来说一下 order by 语句的优化。

1. order by 的基本原理其实就是 MySQL 会给每个线程分配一块内存也就是 sort_buffer 用于排序，sort_buffer 中存储的是 select 涉及到的所有的字段，可以称为全字段排序吧。排序这个动作，可能在内存中完成，也可能需要使用外部排序，这取决于排序所需的内存和 sort_buffer 的大小，由参数 sort_buffer_size 决定。如果要排序的数据量小于 sort_buffer_size，排序就在内存中完成。但如果排序数据量太大，内存放不下，就需要利用磁盘临时文件来辅助排序。

1. 这里其实可以优化下，只存放排序相关的字段，而不是 select 涉及的所有字段，这样 sort_buffer 中存放的东西就多一点，就尽可能避免使用磁盘进行外部排序，或者说使得划分的磁盘文件相对变少，减少磁盘访问。这种排序称为 rowid 排序。如果表中单行的长度超过 max_length_for_sort_data 定义的值，那 MySQL 就认为单行太大（那么数据量肯定就越大，sort_buffer 可能不够用），由全字段排序改为 rowid 排序。

以上是我们说的关于 order by 的两个参数优化，还可以根据索引进行一些优化。

1. 以 select a, b, c from table where a = xxxx order by b 为例，我们为查询条件 a 和排序条件 b 建立联合索引，联合索引就是 a 是从小到大绝对有序的，如果 a 相同，再按 b 从小到大排序，这样就不需要排序了，直接避免了排序这个操作。

1. 还可以进一步优化，由于联合索引 (a, b) 中没有 c 的值，所以从联合索引树上获取符合条件的对应主键 id 后，还需要回表查询取出 a b c 的值，这个回表查询的过程可以通过建立 (a,b,c) 覆盖索引来避免。

- EOF -

推荐阅读  点击标题可跳转

1、[你有没有在 MySQL 的 order by 上栽过跟头](http://mp.weixin.qq.com/s?__biz=MjM5NzMyMjAwMA==&mid=2651508091&idx=2&sn=8c0397a03f1ac9f5a6cd402d996f6f21&chksm=bd25a1048a522812cf4c03ce486ceef3fbe2996b14dc59d6bcf850568c0d6e901a88a78633b0&scene=21#wechat_redirect)

2、[写了个数据查询为空的 Bug，你会怎么办？](http://mp.weixin.qq.com/s?__biz=MjM5NzMyMjAwMA==&mid=2651533754&idx=1&sn=905e8fe2eb641c2e736738449b0497bd&chksm=bd245dc58a53d4d3f13a350c17c95f595214c3ef0d796773a875e5549abcbb4e0678040ee73e&scene=21#wechat_redirect)

3、[MyBatis Plus 解决大数据量查询慢问题](http://mp.weixin.qq.com/s?__biz=MjM5NzMyMjAwMA==&mid=2651525965&idx=1&sn=aa318e66988a26046bfc0d38cf74cc7f&chksm=bd247f328a53f624c13680bd07229508224b03f86dc1c98eef5d82e0bf940afd45d33ab0ce61&scene=21#wechat_redirect)

看完本文有收获？请转发分享给更多人

**关注「ImportNew」，提升Java技能**

![](http://mmbiz.qpic.cn/mmbiz_png/eZzl4LXykQypIvIeptJia56GgprA9jwo8FYmbt4dS8FxGEZwPppBlznUEXptPq5HnLhg3LmyBmV8eX5gqibJA1rg/300?wx_fmt=png&wxfrom=19)

**ImportNew**

ImportNew 专注Java 技术分享，包括Java基础技术、进阶技能、架构设计和Java技术领域动态等

221篇原创内容

公众号

点赞和在看就是最大的支持❤️

阅读 2564

​

写留言

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/eZzl4LXykQypIvIeptJia56GgprA9jwo8FYmbt4dS8FxGEZwPppBlznUEXptPq5HnLhg3LmyBmV8eX5gqibJA1rg/300?wx_fmt=png&wxfrom=18)

ImportNew

111082

写留言

写留言

**留言**

暂无留言
