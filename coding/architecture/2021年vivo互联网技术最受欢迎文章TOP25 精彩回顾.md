# 

原创 vivo互联网技术 vivo互联网技术

 _2022年01月05日 21:00_

  

**｜摘要****｜**

■ 岁月流转，时光飞逝，转眼2021年已经画上句号。过去一年，vivo 互联网技术共推送了107篇文章，涉及服务器、前端、数据库等技术。

今天小编就带大家回顾一下2021年我们最受欢迎的25篇文章（根据阅读量和点赞筛选）。

  

****1****

  

[![图片](https://mmbiz.qpic.cn/mmbiz_png/4g5IMGibSxt4xBJnoD0hicPX4uCtKzUErwkrVgbLExePciaDnoibAOQ82cVdzZYBBFz8nU5Mzo1vBdkV710VAqdWjQ/640?wx_fmt=png&wxfrom=13&tp=wxpic)](http://mp.weixin.qq.com/s?__biz=MzI4NjY4MTU5Nw==&mid=2247490364&idx=1&sn=8f5f6de38d0d28a262176757be16b6c6&chksm=ebd86baedcafe2b8d6d690fd803350b344a7b0a4ad39ac6dd6730f85166e567b035abfc83394&scene=21#wechat_redirect)

  

[《MongoDB在评论中台的实践》](http://mp.weixin.qq.com/s?__biz=MzI4NjY4MTU5Nw==&mid=2247490364&idx=1&sn=8f5f6de38d0d28a262176757be16b6c6&chksm=ebd86baedcafe2b8d6d690fd803350b344a7b0a4ad39ac6dd6730f85166e567b035abfc83394&scene=21#wechat_redirect)

  

随着公司业务发展和用户规模的增多，很多项目都在打造自己的评论功能，而评论的业务形态基本类似。当时各项目都是各自设计实现，存在较多重复的工作量；并且不同业务之间数据存在孤岛，很难产生联系。因此我们决定打造一款公司级的评论业务中台，为各业务方提供评论业务的快速接入能力。本文主要讲述 vivo 评论中台在数据库设计上的技术探索和实践。

  

  

******2******  

  

[![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)](http://mp.weixin.qq.com/s?__biz=MzI4NjY4MTU5Nw==&mid=2247491344&idx=2&sn=0ac23704bc0f1e66d065240e2538f89d&chksm=ebd86f82dcafe69499cf6dab34431a9102ed78cca2dc45c4e7172248ee605b8d0f0b1a1fadd0&scene=21#wechat_redirect)

  

[《详解Apache Dubbo的SPI实现机制》](http://mp.weixin.qq.com/s?__biz=MzI4NjY4MTU5Nw==&mid=2247491344&idx=2&sn=0ac23704bc0f1e66d065240e2538f89d&chksm=ebd86f82dcafe69499cf6dab34431a9102ed78cca2dc45c4e7172248ee605b8d0f0b1a1fadd0&scene=21#wechat_redirect)

  

本文主要分析Dubbo中对 SPI机制实现方式及相关原理，以核心类ExtensionLoader的源码解读来将实现细节进行分析，并对各使用场景使用扩展类的流程细节进行展示和总结。

  

  

********3********  

  

[![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)](http://mp.weixin.qq.com/s?__biz=MzI4NjY4MTU5Nw==&mid=2247491685&idx=2&sn=e703482f8523816b9a8413425cce9fbe&chksm=ebdb90f7dcac19e110b8f9a00d9d34f91e314f070b28755ff3a542adaab1fe59bd0d84611153&scene=21#wechat_redirect)

  

[《富文本及编辑器的跨平台方案》](http://mp.weixin.qq.com/s?__biz=MzI4NjY4MTU5Nw==&mid=2247491685&idx=2&sn=e703482f8523816b9a8413425cce9fbe&chksm=ebdb90f7dcac19e110b8f9a00d9d34f91e314f070b28755ff3a542adaab1fe59bd0d84611153&scene=21#wechat_redirect)

  

本文将围绕富文本跨平台和编辑器跨平台两个部分介绍跨平台的价值，以及如何实现跨平台。通过一些方案介绍和踩坑分享，希望能给有富文本编辑器跨平台相关需求的小伙伴带来一些帮助。

  

  

********4********

  

[![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)](http://mp.weixin.qq.com/s?__biz=MzI4NjY4MTU5Nw==&mid=2247492039&idx=1&sn=75b0bc10539e5e1dc5d1d23fb78472dc&chksm=ebdb9155dcac1843b29f1cbc18e1bed8d0708a08691997027936d67beefc4712cea576b52d9a&scene=21#wechat_redirect)

  

[《设计模式如何提升 vivo 营销自动化业务扩展性 | 引擎篇01》](http://mp.weixin.qq.com/s?__biz=MzI4NjY4MTU5Nw==&mid=2247492039&idx=1&sn=75b0bc10539e5e1dc5d1d23fb78472dc&chksm=ebdb9155dcac1843b29f1cbc18e1bed8d0708a08691997027936d67beefc4712cea576b52d9a&scene=21#wechat_redirect)

  

营销业务本身极具复杂多变性，特别是伴随着数字化营销蓬勃发展的趋势，在市场的不同时期、公司发展的不同阶段、面向不同的用户群体以及持续效果波动迭代，都会产生不同的营销策略决策。本文详细解析设计模式和相关应用如何帮助营销自动化业务提升系统扩展性，以及实践过程中的思考和总结。

  

  

********5********

  

[![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)](http://mp.weixin.qq.com/s?__biz=MzI4NjY4MTU5Nw==&mid=2247491752&idx=2&sn=0fc24f990210b55021c069fbda7f6b69&chksm=ebdb903adcac192c80d286b49d976f9ca52092f43edb52205d73c4e5ba5db983f5a359f29ae2&scene=21#wechat_redirect)

  

[《带你走进MySQL全新高可用解决方案-MGR》](http://mp.weixin.qq.com/s?__biz=MzI4NjY4MTU5Nw==&mid=2247491752&idx=2&sn=0fc24f990210b55021c069fbda7f6b69&chksm=ebdb903adcac192c80d286b49d976f9ca52092f43edb52205d73c4e5ba5db983f5a359f29ae2&scene=21#wechat_redirect)

  

MGR(全称 MySQL Group Replication 

【MySQL 组复制】)是Oracle MySQL于2016年12月发布MySQL 5.7.17推出的一个全新高可用和高扩展的解决方案。本文主要介绍MySQL Group Replication(组复制)技术的基本原理和技术演进史以及安装体验新特性。

  

  

********6********

  

[![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)](http://mp.weixin.qq.com/s?__biz=MzI4NjY4MTU5Nw==&mid=2247492039&idx=2&sn=eec30ff895a29e608fdafe78c626115d&chksm=ebdb9155dcac1843580ec28b8c31e334eb8aeb0a10573b5f3d4e7e8aaaec49893772399790a9&scene=21#wechat_redirect)

  

[《深入剖析 Spring WebFlux》](http://mp.weixin.qq.com/s?__biz=MzI4NjY4MTU5Nw==&mid=2247492039&idx=2&sn=eec30ff895a29e608fdafe78c626115d&chksm=ebdb9155dcac1843580ec28b8c31e334eb8aeb0a10573b5f3d4e7e8aaaec49893772399790a9&scene=21#wechat_redirect)

  

WebFlux 是一个异步非阻塞式的web框架 。内部以Reactor库为基础，可以在不扩充硬件资源的前提下提高系统吞吐量。适用于IO密集型的服务。

  

  

********7********

  

[![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)](http://mp.weixin.qq.com/s?__biz=MzI4NjY4MTU5Nw==&mid=2247490917&idx=3&sn=9f38d958ea75dd3593062f6c8b52d3fb&chksm=ebd86df7dcafe4e1cdfd3fbc9521f9fc5c06031f5605503aebdf66823fc6b01a14408db1c669&scene=21#wechat_redirect)

  

[《如何把 Caffeine Cache 用得如丝般顺滑》](http://mp.weixin.qq.com/s?__biz=MzI4NjY4MTU5Nw==&mid=2247490917&idx=3&sn=9f38d958ea75dd3593062f6c8b52d3fb&chksm=ebd86df7dcafe4e1cdfd3fbc9521f9fc5c06031f5605503aebdf66823fc6b01a14408db1c669&scene=21#wechat_redirect)

  

推荐系统等面向用户的后端服务通常需要多级缓存方案，通过本地-分布式缓存-数据库方式进行缓存加速。而 Caffeine Cache 正是基于 JAVA8 的高性能本地缓存组件，因其更有效的淘汰算法和良好的易用性，已被 Spring Boot 2.0 后续版本集成并替代了 Google Guava Cache。本文从最常用的 get 方法入口，结合源代码，细数作者使用 Caffeine Cache 过程中遇到的各种坑和思考，作为闭坑指南分享给各位看官。

  

********8********

  

[![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)](http://mp.weixin.qq.com/s?__biz=MzI4NjY4MTU5Nw==&mid=2247491453&idx=1&sn=c74628fed0ae2ee5d79ec428c36f13f5&chksm=ebd86fefdcafe6f96a4fbec28af80cfcf8a8375dbc8bc6a548243fdd1461798e54fc79bced90&scene=21#wechat_redirect)

  

[《初探 Redis 客户端 Lettuce：真香》](http://mp.weixin.qq.com/s?__biz=MzI4NjY4MTU5Nw==&mid=2247491453&idx=1&sn=c74628fed0ae2ee5d79ec428c36f13f5&chksm=ebd86fefdcafe6f96a4fbec28af80cfcf8a8375dbc8bc6a548243fdd1461798e54fc79bced90&scene=21#wechat_redirect)

  

Jedis 是业界常用的 Redis JAVA 客户端。而 Lettuce 是一款基于 Netty 实现的异步客户端，不但逐步覆盖了 Jedis 的各种功能，还在可靠性、易用性、可扩展性、可维护性等方面都有长足发展。笔者因项目需要，在一些场景下切换成 Lettuce 进行开发，并且希望能类似 Jedis pipeline 一样提升批量查询的性能。在对 Lettuce 进行了学习和踩坑后，笔者将过程中遇到的各种关键点总结成文章分享给大家。

  

  

********9********

  

[![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)](http://mp.weixin.qq.com/s?__biz=MzI4NjY4MTU5Nw==&mid=2247492229&idx=1&sn=5a8336c9e9b73bf5c53b6ed986084ee8&chksm=ebdb9217dcac1b019679f3b9ebf0e847661c36814069a0a05aaa9002bc13409705d2119dea6f&scene=21#wechat_redirect)

  

[《高并发场景下JVM调优实践之路》](http://mp.weixin.qq.com/s?__biz=MzI4NjY4MTU5Nw==&mid=2247492229&idx=1&sn=5a8336c9e9b73bf5c53b6ed986084ee8&chksm=ebdb9217dcac1b019679f3b9ebf0e847661c36814069a0a05aaa9002bc13409705d2119dea6f&scene=21#wechat_redirect)

  

JVM调优是系统性能优化的重要手段之一，也是最为后端工程师津津乐道的技术沉淀之一，网络上已经有非常丰富的资料，不过大都偏向于理论，且没有一步一步落地的细节。本文着重于实践，一步一步介绍线上某核心服务的JVM调优落地过程，希望能给读者提供JVM调优的思路和可参考、可落地的方案。

  

  

********10********

  

[![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)](http://mp.weixin.qq.com/s?__biz=MzI4NjY4MTU5Nw==&mid=2247490845&idx=1&sn=0886dfa5cc6c80af006778882b758f31&chksm=ebd86d8fdcafe4994c8917873b0fbe5043b60d8571be5660393271366d32088b66fb1aae4e43&scene=21#wechat_redirect)

  

《[深入剖析共识性算法 Raft](http://mp.weixin.qq.com/s?__biz=MzI4NjY4MTU5Nw==&mid=2247490845&idx=1&sn=0886dfa5cc6c80af006778882b758f31&chksm=ebd86d8fdcafe4994c8917873b0fbe5043b60d8571be5660393271366d32088b66fb1aae4e43&scene=21#wechat_redirect)》

  

分布式一致性 (distributed consensus) 是分布式系统中最基本的问题，用来保证一个分布式系统的可靠性以及容错能力。Raft 出现之前，Paxos 一直是分布式一致性算法的标准。Paxos 难以理解，更难以实现。Raft 的设计目标是简化 Paxos，使得算法既容易理解，也容易实现。

**TOP 11-25**

  

  

[**深入理解Netty-从偶现宕机看Netty流量控制**](http://mp.weixin.qq.com/s?__biz=MzI4NjY4MTU5Nw==&mid=2247491988&idx=2&sn=fe0545c4d62172728701db48b8927ae7&chksm=ebdb9106dcac1810fba96fd9f6dcedae293c5718113fc779056fa3a895e8a6215c13798a6760&scene=21#wechat_redirect) 

Netty是一款高性能网络IO框架，本文结合线上长连接真实案例，深入讲述Netty的流量控制原理以及问题解决的思路和步骤。

  

[**Hystrix实战经验分享**](http://mp.weixin.qq.com/s?__biz=MzI4NjY4MTU5Nw==&mid=2247490535&idx=3&sn=e232f4de66a832d7d05e1181e3a9ce3a&chksm=ebd86b75dcafe263b13e502bf9042ebafbdce5b1a65f5413d608f59e3051001511a55c208aac&scene=21#wechat_redirect) 

Hystrix是Netlifx开源的一款容错框架，防雪崩利器，具备服务降级，服务熔断，依赖隔离，监控(Hystrix Dashboard)等功能。本文从作者个人开发实战角度，总结分享Hystrix的使用经验，希望大家能有所收获。

  

[**Redis线程模型的前世今**](http://mp.weixin.qq.com/s?__biz=MzI4NjY4MTU5Nw==&mid=2247492422&idx=2&sn=5a02b92ede62a94486c96cfd98a914c9&chksm=ebdb93d4dcac1ac2a48a48f0bc42a57c7bef7d707a139d99de9471c9660122e813168dede58f&scene=21#wechat_redirect)**生** 

Redis线程模型为什么要这么设计，有什么优点和缺点，有哪些思想是可以借鉴的...本文从网络IO的历史、Reactor模型的历史、到Redis线程模型的设计由浅入深，慢慢道来。

  

[**源码解读Dubbo分层设计思想**](http://mp.weixin.qq.com/s?__biz=MzI4NjY4MTU5Nw==&mid=2247491913&idx=2&sn=4cc15a3f59bbf16d04a45fed47b75f3d&chksm=ebdb91dbdcac18cd2eaa59a758595d4b091774eaa42503e46fcc7b72a9dd19a30b49d3ff88e6&scene=21#wechat_redirect) 

Dubbo是一款非常优秀的分布式服务框架，国内使用非常的广泛，2018年正式成为apache顶级项目。阅读本文你将了解到Dubbo的整体分层设计，每一层的意义，以及Dubbo的初始化流程和RPC调用过程，在这个过程涉及到的领域模型Protocol、Invoker、Exporter、Invocation、Result、URL等。本文的特点在于结合源码详细的介绍每一层的实际意义。

  

[**Kafka万亿级消息实战**](http://mp.weixin.qq.com/s?__biz=MzI4NjY4MTU5Nw==&mid=2247491095&idx=2&sn=6dba8cc3b4b20d4ada1b798714daf23d&chksm=ebd86e85dcafe79360fda7e9b06a4d43d4fdcee11622602628421c8e07bb571ba5ce0fb1f7e1&scene=21#wechat_redirect) 

本文主要总结当Kafka集群流量达到 万亿级记录/天或者十万亿级记录/天  甚至更高后，我们需要具备哪些能力才能保障集群高可用、高可靠、高性能、高吞吐、安全的运行。这里总结内容主要针对Kafka2.1.1版本。

  

[**你分库分表的姿势对么？—详谈水平分库分表**](http://mp.weixin.qq.com/s?__biz=MzI4NjY4MTU5Nw==&mid=2247492174&idx=1&sn=88b2bab955ecae54085bfc8e6b738df2&chksm=ebdb92dcdcac1bcaf67f8590c4ef0d29c8cf72b4147736c02d13223f51614236b9888f87fb92&scene=21#wechat_redirect) 

随着后端数据库的存储量级和用户的访问流量越来越大，我们免不了需要对OLTP数据库进行分库分表操作，那么选取一个的水平分库分表方案就显得非常重要。本文详细介绍在水平分库分表中常见的一些误区，以及一些常用的手法，以帮助识别可能存在的问题、少走弯路。

  

[**十亿级流量下，我与Redis时延小突刺的战斗史**](http://mp.weixin.qq.com/s?__biz=MzI4NjY4MTU5Nw==&mid=2247491344&idx=1&sn=5d573319d989b908454e26711c10e817&chksm=ebd86f82dcafe694446825a09eb4c419369ee14285c7a0a5a72b80236d0f12d6dce07f51d348&scene=21#wechat_redirect) 

本文记录了一次线上服务，慢接口报警的解决流程，包括遇到线上问题的应急方案、分析问题的思路，主要集中在Redis响应比较慢时的分析和解决方案，通过对问题解决过程的总结，希望可以给大家一些参考。

  

[**vivo 全球商城：优惠券系统架构设计与实践**](http://mp.weixin.qq.com/s?__biz=MzI4NjY4MTU5Nw==&mid=2247491631&idx=1&sn=b53c69df13a80c6bf37531e0c3788188&chksm=ebdb90bddcac19ab3dae9eb635089644d6032dfc4394c3f9bcfedf837060eaeb8c5a4d70c532&scene=21#wechat_redirect) 

优惠券是电商常见的营销手段，具有灵活的特点，既可以作为促销活动的载体，也是重要的引流入口。本文主要介绍vivo商城优惠券业务发展历程、架构设计思路及应对各种业务场景的实践。

  

[**Redis大集群扩容性能优化实践**](http://mp.weixin.qq.com/s?__biz=MzI4NjY4MTU5Nw==&mid=2247492111&idx=1&sn=3bc3d6c6fc350b2157015a63a62f11c6&chksm=ebdb929ddcac1b8b4c93fff98446a4a9582c2764fd86422f7a143d990a2a50d2c90885eb6f1e&scene=21#wechat_redirect) 

在现网环境，一些使用Redis集群的业务随着业务量的上涨，往往需要进行节点扩容操作。本文介绍了一次大规模的Redis集群进行扩容操作遇到的性能问题，排查以及优化过程。

  

[**Dubbo 编解码那些事**](http://mp.weixin.qq.com/s?__biz=MzI4NjY4MTU5Nw==&mid=2247490759&idx=2&sn=2430df0edc1da8a4df4266b325e16bfa&chksm=ebd86c55dcafe543aa81ff65c3c3ce4dfbf5734639a386d7f8328cd902b8387efb849b054383&scene=21#wechat_redirect) 

Dubbo作为Java语言的RPC框架，优势之一在于屏蔽了调用细节，能够像调用本地方法一样调用远程服务。本文基于实际问题，梳理dubbo编解码链路，以及Hessian2框架的序列化逻辑。有助于提高对于Dubbo框架的学习、使用和问题排查。

  

[**Java 多线程上下文传递在复杂场景下的实践**](http://mp.weixin.qq.com/s?__biz=MzI4NjY4MTU5Nw==&mid=2247490199&idx=1&sn=a1a14f077890dd07a1a3b58b3df8c844&chksm=ebd86a05dcafe3134010253f63db63b1c3b37b9510dc1fedd7745c41cc518c06e72df1307e1d&scene=21#wechat_redirect) 

海外商城从印度做起，慢慢的会有一些其他国家的诉求，这个时候需要我们针对当前的商城做一个改造，可以支撑多个国家的商城，这里会涉及多个问题，多语言，多国家，多时区，本地化等等。本文描述了vivo海外商城在发展过程中为了适应多个国家的商城系统开发 ，如何把识别出来的国家信息在系统中传递下去，并且解决多线程，定时任务等多种场景下的问题。

  

[**深度解析 Lucene 如何实现轻量级全文索引**](http://mp.weixin.qq.com/s?__biz=MzI4NjY4MTU5Nw==&mid=2247491503&idx=2&sn=4644967adac66698109eaaea460d6903&chksm=ebd86f3ddcafe62ba9efa259af463f48fe1f39b6b7a46670df6ed98857319828a085625d2605&scene=21#wechat_redirect) 

Lucene是一个开放源码的全文检索引擎工具包，提供了完整的查询引擎和索引引擎，部分语种文本分析引擎。本文介绍了Lucene的相关使用心得，内容涵盖索引的生成、管理及搜索功能等内容和本人在轻量级的数据搜索中，深度解析Lucene如何实现全文索引。

  

[**深入剖析 RocketMQ 源码 - 消息存储模块**](http://mp.weixin.qq.com/s?__biz=MzI4NjY4MTU5Nw==&mid=2247492304&idx=2&sn=5656f373033a364f42e1ddac3294bb27&chksm=ebdb9242dcac1b54239cc3a9254ea7a7d442bda8be82f549f9e3d9f25e33003d9a72cddcf055&scene=21#wechat_redirect)

消息队列是一种服务间异步通信方式，广泛应用于微服务架构设计中的解耦、异步、削峰等场景。消息在被处理和删除之前一直存储在队列上。RocketMQ 是 2012 年阿里巴巴开源的第三代分布式消息中间件，本文主要从源码角度讲述 RocketMQ 存储模块如何设计。

  

[**亿级用户中心的设计与实践**](http://mp.weixin.qq.com/s?__biz=MzI4NjY4MTU5Nw==&mid=2247490396&idx=1&sn=1776a9a09da4592c0c48c29f88ca7484&chksm=ebd86bcedcafe2d8224229653082f43bd5d0f0a18a0aa721f02591019d163b2da0b53a3824ac&scene=21#wechat_redirect)

用户中心是互联网最为基础的核心系统，随着业务和用户的增长，势必会带来不断的挑战。如何在亿级的情况下保证系统的高可用，高性能以及高安全，本文能够给你一套实践方案。

  

[**vivo统一告警平台建设与实践**](http://mp.weixin.qq.com/s?__biz=MzI4NjY4MTU5Nw==&mid=2247492382&idx=1&sn=c88129acaeb2aed6584f35f51d3c46bb&chksm=ebdb938cdcac1a9a90d8ea251dfbce3deecb62288b4c1be49cf59372e83ed5ccad3977295624&scene=21#wechat_redirect)

在监控服务统一的大背景下，告警收敛能力下沉就成为必然，平台建设过程中，如何解决以前服务的痛点，充分释放统一告警服务的通用能力，需要经过深入的分析才能更好的抽离通用能力。文章以告警收敛为主线，逐层深入，介绍了统一告警平台的建设和实践。

  

2021年，我们依旧保持开放自由的心态，输出更多的原创技术内容，加强文章在技术深度、实践经验上的沉淀，希望能与大家一起交流、切磋。

  

END

![](http://mmbiz.qpic.cn/mmbiz_png/4g5IMGibSxt45QXJZicZ9gaNU2mRSlvqhQd94MJ7oQh4QFj1ibPV66xnUiaKoicSatwaGXepL5sBDSDLEckicX1ttibHg/300?wx_fmt=png&wxfrom=19)

**vivo互联网技术**

分享 vivo 互联网技术干货与沙龙活动，推荐最新行业动态与热门会议。

431篇原创内容

公众号

阅读 3381

​

写留言

**留言 12**

- 辉辉
    
    2022年1月5日
    
    赞3
    
    求更新营销自动化![[玫瑰]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
    vivo互联网技术
    
    作者2022年1月5日
    
    赞2
    
    给作者团队提需求了，快马加鞭创作中，记得持续关注推送哦![[玫瑰]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
- chw。
    
    2022年1月5日
    
    赞2
    
    公众号文章做的真好![[强]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![[强]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![[强]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
- 米兰PLA
    
    2022年1月6日
    
    赞
    
    文章质量很高，get很多，给你们1000个赞👍
    
    vivo互联网技术
    
    作者2022年1月6日
    
    赞1
    
    笔芯![[转圈]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
- AlexPeng
    
    2022年1月5日
    
    赞1
    
    这些文章质量都非常高
    
- Qiang
    
    2022年1月5日
    
    赞1
    
    很赞呀，真不错
    
- 王咖啡
    
    2022年1月27日
    
    赞
    
    想看广告投放平台相关
    
- 孺牛
    
    2022年1月26日
    
    赞
    
    预定春节假期阅读文章![[666]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
- 袁大山
    
    2022年1月5日
    
    赞
    
    学到了不少，谢谢分享。
    
- Yezhiwei
    
    2022年1月5日
    
    赞
    
    都很赞👍
    
- Nemo
    
    2022年1月5日
    
    赞
    
    太棒了
    

已无更多数据

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/4g5IMGibSxt45QXJZicZ9gaNU2mRSlvqhQd94MJ7oQh4QFj1ibPV66xnUiaKoicSatwaGXepL5sBDSDLEckicX1ttibHg/300?wx_fmt=png&wxfrom=18)

vivo互联网技术

37413

12

写留言

**留言 12**

- 辉辉
    
    2022年1月5日
    
    赞3
    
    求更新营销自动化![[玫瑰]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
    vivo互联网技术
    
    作者2022年1月5日
    
    赞2
    
    给作者团队提需求了，快马加鞭创作中，记得持续关注推送哦![[玫瑰]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
- chw。
    
    2022年1月5日
    
    赞2
    
    公众号文章做的真好![[强]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![[强]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![[强]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
- 米兰PLA
    
    2022年1月6日
    
    赞
    
    文章质量很高，get很多，给你们1000个赞👍
    
    vivo互联网技术
    
    作者2022年1月6日
    
    赞1
    
    笔芯![[转圈]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
- AlexPeng
    
    2022年1月5日
    
    赞1
    
    这些文章质量都非常高
    
- Qiang
    
    2022年1月5日
    
    赞1
    
    很赞呀，真不错
    
- 王咖啡
    
    2022年1月27日
    
    赞
    
    想看广告投放平台相关
    
- 孺牛
    
    2022年1月26日
    
    赞
    
    预定春节假期阅读文章![[666]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
- 袁大山
    
    2022年1月5日
    
    赞
    
    学到了不少，谢谢分享。
    
- Yezhiwei
    
    2022年1月5日
    
    赞
    
    都很赞👍
    
- Nemo
    
    2022年1月5日
    
    赞
    
    太棒了
    

已无更多数据