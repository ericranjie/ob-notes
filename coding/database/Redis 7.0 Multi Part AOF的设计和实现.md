# 

原创 驱动 阿里云开发者

 _2022年02月11日 08:05_

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/Z6bicxIx5naLfmV3vZ4nITVUm01NdB5UlsF2w36rZ4aWEasQkAafx0LDxt0ToyrCdXgucjIjku3ppbzNJkicZUJQ/640?wx_fmt=jpeg&wxfrom=13&tp=wxpic)

  

redis5

redis · 目录

上一篇一文详解Redis中BigKey、HotKey的发现与处理下一篇一文搞懂redis

阅读原文

阅读 1.3万

​

写留言

**留言 2**

- 踏雪无痕
    
    2022年2月14日
    
    赞
    
    close旧文件的时候是提交一个异步任务到bio里，因此是异步close不会卡主进程
    
- Xianggg
    
    2022年2月11日
    
    赞
    
    主进程在维护 INCR 文件轮转切换（close+open）的时候会卡一下吗，我们遇到 MySQL 在切换 binlog 的时候如果是刚好有提交阶段有可能会产生轻微的抖动，请问 MP-AOF 有遇到过吗
    

已无更多数据

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/Z6bicxIx5naI1jwOfnA1w4PL2LhwNia76vBRfzqaQVVVlqiaLjmWYQXHsn1FqBHhuGVcxEHjxE9tibBFBjcB352fhQ/300?wx_fmt=png&wxfrom=18)

阿里云开发者

891026

2

写留言

**留言 2**

- 踏雪无痕
    
    2022年2月14日
    
    赞
    
    close旧文件的时候是提交一个异步任务到bio里，因此是异步close不会卡主进程
    
- Xianggg
    
    2022年2月11日
    
    赞
    
    主进程在维护 INCR 文件轮转切换（close+open）的时候会卡一下吗，我们遇到 MySQL 在切换 binlog 的时候如果是刚好有提交阶段有可能会产生轻微的抖动，请问 MP-AOF 有遇到过吗
    

已无更多数据