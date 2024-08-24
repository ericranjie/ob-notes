# 

openGauss

 _2021年10月20日 18:01_

现代社会信息数据爆炸式增长，工业界业务需求纷繁复杂。数据存储的数据量，建表数量也都不断增长。openGauss通用的普通表，每个数据表对应一个逻辑逻辑上的大文件（最大32T），该逻辑文件又按照固定的大小划分多个实际文件存在对应的数据库目录下面。所以，每张数据表随着数据量的增多，底层的数据存储所需文件数量会逐渐增多。同时，openGauss对外提供hashbucket表、大分区表等特性，每张数据表会被拆分为若干个子表，底层所需文件数量更是成倍增长。由此，这种存储管理模式存在以下问题：

1. 对文件系统依赖大，无法进行细粒度的控制提升可维护性；

2. 大数据量下文件句柄过多，目前只能依赖虚拟句柄来解决，影响系统性能；

3. 小文件数量过多会导致全量build、全量备份等场景下的随机IO问题，影响性能；

为了解决以上问题，openGauss引入段页式存储管理机制，类似于操作系统的段页式内存管理，但是在实现机制上区别很大。

**一、 段页式实现原理**

在段页式存储管理下，表空间和数据文件以段(Segment)、区(Extent)以及页（Page/Block）为逻辑组织方式进行存储的分配和管理。如下图所示。具体来说，一个database(在一个tablespace中)有且仅有一个段空间（segment space），实际物理存储可以是一个文件，也可以拆分成多个文件。该database中所有table都从该段空间中分配数据。所以表的个数和实际物理文件个数无关。每个table有一个逻辑上的segment，该table所有的数据都存在该segment上。每个segment会挂载多个extent，每个extent是一块连续的物理页。Extent的大小可以根据业务需求灵活调整，避免存储空间的浪费。

![图片](https://mmbiz.qpic.cn/mmbiz_png/SX6wqnysYmrHvTa1MDhk4ic79yxQBVddLCffOjVf4HRlQYFRoJ1fEZrje2UfxVe5yHCln3rxVeBSic3wI9AdsWEw/640?wx_fmt=png&wxfrom=13&tp=wxpic)

图 1 段页式存储设计示意图

段页式文件可以自动扩容，不需要用户手动指定，直到磁盘空间用满或者达到tablespace设置的limit限制。段页式存储不会自动回收磁盘空间。当某些数据表被删除之后，其在段页式文件中占据的空间，会被保留，即段页式文件中会存在一些空洞，磁盘空间没有被释放。这些空洞会被后面新扩展或者创建出来的表重用。用户如果确定不需要重用这些空洞，可以手动调用系统函数，来进行磁盘空间的回收，释放磁盘空间。

内部实现上，每个segment 对应原先页式存储的一个物理文件。比如每个分区表、每个hashbucket表的一个bucket，都会有一个单独的segment。每个segment下面会挂载多个extent，每个extent在文件中是连续的，但extent之间未必连续。Segment会动态扩展，加入新的extent，但不能直接回收某个extent。可以通过对整个table做truncate或者cluster等方式，以segment为粒度回收存储空间。

目前支持四种大小的extent，分别是64K/1M/8M/64M。对于一个segment来说，每一次扩展的extent的大小是固定的。前16个extent大小为64K，第17到第143个extent大小为1MB，依次类推。具体参数如下表所示。

表 1 segment存储extent分类表

|   |   |   |   |   |   |
|---|---|---|---|---|---|
|Group|Extent size|Extent page count|Extent count range|Total page count|Total size|
|1|64K|8|[1, 16]|128|1M|
|2|1M|128|[17, 143]|16K|128M|
|3|8M|1024|[144, 255]|128K|1G|
|4|64M|8192|[256, …]|…|…|

**二、 段页式表使用指导**

用户在用SQL语句 create table建表时可以通过指定参数segment=on，使得行存表可以使用段页式的方式存储数据。如果指定hashbucket=on，则默认强制使用segment=on。目前段页式存储不支持列存表。段页式表空间是自动创建的，不需要用户有额外的命令。

**1. 指定参数segment=on，创建段页式普通表**

```
create table t1(a int, b int, PRIMARY KEY(a,b)) with(segment=on);
```

![图片](https://mmbiz.qpic.cn/mmbiz_png/SX6wqnysYmrHvTa1MDhk4ic79yxQBVddLslib8yFJCA6Vxc4YSow7ptKvMfLxroSscEROMULhfbCYDJn0qibQftYQ/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

**2. 指定参数hashbucket=on，创建段页式hashbucket表**

```
create table t1(a int, b int, PRIMARY KEY(a,b)) with(hashbucket=on);
```

![图片](https://mmbiz.qpic.cn/mmbiz_png/SX6wqnysYmrHvTa1MDhk4ic79yxQBVddL81zz0Ww2H6PIGtA34tq2bOvtpW1oiaOc1dNZETGXHKg3VKalTnJ8mibw/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

为了让用户更好使用段页式功能，openGauss提供了两个built in的系统函数，显示extent的使用情况。用户可以使用这两个视图，决定是否回收和回收哪一部分的数据。

- pg_stat_segment_space_info(Oid tablespace, Oid database); 参数是tablespace和database的Oid，输出位于该表空间下所有ExtentGroup的使用信息。
    

表 2  pg_stat_segment_space_info视图字段信息

|   |   |
|---|---|
|名称|描述|
|extent_size|该ExtentGroup的extent规格，单位是block数|
|total_blocks|物理文件总extent数目|
|meta_data_blocks|表空间管理的metadata占用的block数，只包括space header，map page等，不包括segment head。|
|used_data_blocks|存数据占用的extent数目。包括segment head。|
|utilization|使用的block数占总block数的百分比。即(used_data_blocks+meta_data_block)/total_blocks。|
|high_water_mark|高水位线，被分配出去的extent，最大的物理页号。超过高水位线的block都没有被使用，可以被直接回收。|

![图片](https://mmbiz.qpic.cn/mmbiz_png/SX6wqnysYmrHvTa1MDhk4ic79yxQBVddLBO6V8HNetjzuOurqZ8GlobKsSk4P5LhMbtpibq4iaZGXBVyYN49wsGsg/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

- pg_stat_segment_extent_usage(Oid tablespace, Oid databse, uint32 extent_type); 每次返回一个ExtentGroup中，每个被分配出去的extent的使用情况。extent_type表示ExtentGroup的类型，合理取值为[1,5]的int值。在此范围外的会报error。
    

![图片](https://mmbiz.qpic.cn/mmbiz_png/SX6wqnysYmrHvTa1MDhk4ic79yxQBVddLyw8JvQKPR4HSnqZSIHHf0uXvKJafFgeTA037RMTLbyvY9w9d9q0BLg/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

表 3 pg_stat_segment_extent_usage视图字段信息

|   |   |
|---|---|
|名称|描述|
|start_block|Extent的起始物理页号|
|extent_size|Extent的大小|
|usage_type|Extent的使用类型，比如segment head，data extent等。|
|ower_location|有指针指向该extent的对象的位置。比如data extent的owner就是它所属的segment的head位置。|
|special_data|该extent在它owner中的位置。该字段的数据跟使用类型有关。比如data extent的special data就是它在所属segment中的extent id。|

![图片](https://mmbiz.qpic.cn/mmbiz_png/SX6wqnysYmrHvTa1MDhk4ic79yxQBVddLyw8JvQKPR4HSnqZSIHHf0uXvKJafFgeTA037RMTLbyvY9w9d9q0BLg/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

  

- gs_spc_shrink(Oid tablespace, Oid database, uint32 extent_type);每次清理一个ExtentGroup。Shrink的目标大小是自动计算出来的，为active的数据量 + 128MB，然后向上取整和128MB对齐。
    

**三、 总结**

openGauss为了解决hashbucket表、大分区表数量较多时，底层文件句柄过多的问题，提供了段页式解决方案。段页式对外将表对应逻辑上的一个段(segment)，底层不同的segment存储在一个物理文件上，大大减少了底层物理文件的句柄。即使在大数据量下，也避免了普通表那种文件句柄过多的场景，提升了系统可维护性。同时，对于全量build、全量备份等场景，减少小文件数量过多引起的随机IO，可以提升系统IO性能。同时可以看到当前段页式表相关的参数都是固定的，未来openGauss可以探索利用AI技术，对段页式存储机制进行参数自动调参，从而可以为用户提供更智能，性能更优的段页式存储策略。

  

**◆ 相关推荐◆**

  

[

一文汇总全密态数据库的基本使用方法

2021-10-16

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/SX6wqnysYmqrhnes7EiaibkWjMOTa3Nkns4NC9fxslcaGibWBUxmgicgYGP62pmuZDud1zqXgKXgicMY9JvicYj9FbIg/640?wx_fmt=jpeg&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

](http://mp.weixin.qq.com/s?__biz=MzIyMDE3ODk1Nw==&mid=2247496684&idx=1&sn=23b4836a188534b3a49e3bb240ade863&chksm=97cd4c8ea0bac598f8b6423d00d8a861ffe4be2d8d5308226a0d1138406563c61e687dd565d2&scene=21#wechat_redirect)

[

全密态黑科技再升级，无感知加解密原理剖析

2021-10-13

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/SX6wqnysYmp8L81iceWtSBLZPiaozaJUGVdILxnqOUPGZMeXyWMc7VXC12I1HicS9arTH6o9YibxzxFDEXCc6Lgvkw/640?wx_fmt=jpeg&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

](http://mp.weixin.qq.com/s?__biz=MzIyMDE3ODk1Nw==&mid=2247496609&idx=1&sn=dc272194522f35aa29ae2d12bf3d468f&chksm=97cd4cc3a0bac5d521c6c25a66c782d61cb738fa02615ca76c0d79ce12e1e50640308a07dad2&scene=21#wechat_redirect)

[

Ustore在openGauss闪亮登场，重构openGauss数据存储的灵魂

2021-10-11

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/SX6wqnysYmpicvib9KwfjvtP3btLSiaggXiauiaNN4T5mNu1GyTlnV9t5E0aWQDttpw8qDTabS8hDoKQRyS0NQwLSqw/640?wx_fmt=jpeg&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

](http://mp.weixin.qq.com/s?__biz=MzIyMDE3ODk1Nw==&mid=2247496361&idx=1&sn=2f5d03e4eeb68f1196ab79712ad70181&chksm=97cd4dcba0bac4dd37de0668686c8f4e93be1bc13d49ad6f1175cd27b83a61d3631939e9629c&scene=21#wechat_redirect)

[

深度干货！openGauss日志共识框架大揭秘

2021-09-29

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/SX6wqnysYmofFoPZUV1V3NdIY8h3ymRbhVnTPN54m8otiaEonhVUnp8VT5jf6S5oCOc5XU0aF7uia1Libf3pDUCMg/640?wx_fmt=jpeg&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

](http://mp.weixin.qq.com/s?__biz=MzIyMDE3ODk1Nw==&mid=2247496231&idx=1&sn=d5a389efa29c629498ec38b49e06a7f2&chksm=97cd4d45a0bac45312214ac3532238fd5accc9dcd93bb81d16c3ae0ac107b8221f3d97c1bf03&scene=21#wechat_redirect)

[

DB4AI：使能数据库原生AI计算，助力数据湖场景业务成功

2021-09-27

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/SX6wqnysYmp9gZ1RcLuJzPAURbVAjfFL8taSvH629EWVxR619KNbFALHtPVCaxA7uw7djA6V5TDPdCme5SibCfw/640?wx_fmt=jpeg&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

](http://mp.weixin.qq.com/s?__biz=MzIyMDE3ODk1Nw==&mid=2247496212&idx=1&sn=86b3623d08c218d8043706540ef0d7e4&chksm=97cd4d76a0bac4604c0937386c0e4b2ad4953f28c5b6af80b00e02e9f5ca6cc73cdc666a68d5&scene=21#wechat_redirect)

[

解密openGauss DB4AI框架的内部机理

2021-09-26

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/SX6wqnysYmpFgd4ktLw8oOZ63y6XK28TKrCFjXo1dKAMATicqZ9adiap0Vz6PqFX0uPibQALxMlcuaSTSYRPXjuMQ/640?wx_fmt=jpeg&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

](http://mp.weixin.qq.com/s?__biz=MzIyMDE3ODk1Nw==&mid=2247496108&idx=1&sn=2e2bf19585bbe0366d58ea6fb18e1fa9&chksm=97cd4ecea0bac7d88ef086e8e2d995432226177e0fd28a71374c163bd3dfad30a28de0f06369&scene=21#wechat_redirect)

![](http://mmbiz.qpic.cn/mmbiz_png/SX6wqnysYmqI2wl74q492VQlNWzLR1kdGibOhic3KXoB1iaJYBMUNo3YF23kOxhdA0GUalaXTib8uwTibKFDUw21wwQ/300?wx_fmt=png&wxfrom=19)

**openGauss**

开源关系型数据库

70篇原创内容

公众号

技术干货109

技术干货 · 目录

上一篇全密态黑科技再升级，无感知加解密原理剖析下一篇关于openGauss账本数据库：你想知道的这里都有

阅读 414

​