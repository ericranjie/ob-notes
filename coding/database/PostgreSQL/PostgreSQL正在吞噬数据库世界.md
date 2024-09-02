
原创 冯若航 非法加冯

 _2024年03月04日 08:18_ _北京_

PostgreSQL 并不是一个简单的关系型数据库，而是一个数据管理的抽象框架，具有吞噬整个数据库世界的力量。而这也是正在发生的事情 —— “一切皆用 Postgres” 已经不再是少数精英团队的前沿探索，而是成为了一种进入主流视野的最佳实践。

---

OLAP 领域迎来踢馆者

在 2016 年的一次数据库沙龙里，我提出了一个观点： 现在 PostgreSQL 生态的一个主要遗憾是，缺少一个**足够好**的列式存储分析插件来做 OLAP 分析。尽管PostgreSQL 本身提供了很强大的分析功能集，应付常规的分析任务绰绰有余。但在较大数据量下全量分析的**性能**，相比专用的实时数仓仍然有些不够看。

以分析领域的权威评测 Clickbench 为例，我们在其中标注出了 PostgreSQL 与生态扩展插件以及兼容衍生数据库在其中的性能表现。原生未经过调优的 PostgreSQL 表现较为拉垮（**x1050**），但经过调优后可以达到（**x47**）；此外还有三个与分析有关系的扩展：列存 **Hydra**（**x42**），时序扩展 **TimescaleDB**（**x103**），以及分布式扩展 **Citus**（**x262**）。

![](https://mmbiz.qpic.cn/mmbiz_png/Wkpr3rA9wF2AjrSq2oprktCwXqVmghlSBXxvkR3eZCr7qxibg4t29h8iaA0dkz8dtMcms8Ll2zd06fL5FiaHqdztg/640?wx_fmt=png&from=appmsg&wxfrom=13&tp=wxpic)

这样的分析性能表现不能说烂，因为比起 MySQL，MariaDB 这样的纯 OLTP 数据库的辣眼表现（**x3065,x19700**）确实好很多；但第三梯队的性能表现也绝对说不上足够好，与专注于 OLAP 的第一梯队组件：Umbra，ClickHouse，Databend，SelectDB（**x3~x4**）相比，在分析性能上仍然有十几倍的性能差距。食之无味，弃之可惜。

**然而，** [**ParadeDB**](http://mp.weixin.qq.com/s?__biz=MzU5ODAyNTM5Ng==&mid=2247486913&idx=1&sn=3b7d8cf3f0e323932aba52c897f3c7a4&chksm=fe4b381ac93cb10cc6175c4c7978b5903946d369fe0084fbae5edf76ab08d84134260f28dffc&scene=21#wechat_redirect) **和** [****DuckDB****](http://mp.weixin.qq.com/s?__biz=MzU1NTg2ODQ5Nw==&mid=2247485101&idx=1&sn=902573821af9d2793c7ad6e07483d85f&chksm=fbcc8e93ccbb07858a611233118bedaebc133c0bdf44b7124f3240bcafc3fe9761b5b6053186&scene=21#wechat_redirect) **的出现改变了这一点！**

**ParadeDB** 提供的 PG 原生扩展 **pg_analytics** 实现了第二梯队（**x10**）的性能水准，与第一梯队只有 3～4 倍的性能差距。相对于其他功能上的收益，这种程度的性能差距通常是可以接受的 —— ACID，新鲜性与实时性，无需 ETL、额外学习成本、维护独立的新服务，更别提它还提供了 ElasticSearch 质量的全文检索能力。

而 **DuckDB** 则专注于 OLAP ，将分析性能这件事做到了极致（**x3.2**） —— 略过第一名 Umbra 这种学术研究型闭源数据库，DuckDB 也许是 OLAP 实战性能最快的数据库了。它并不是 PG 的扩展插件，但它是一个嵌入式文件数据库，而 [**DuckDB FDW**](http://mp.weixin.qq.com/s?__biz=MzU1NTg2ODQ5Nw==&mid=2247484878&idx=1&sn=de6efc27b12308e96878ce5b5a248ad5&chksm=fbcc8df0ccbb04e6bad84fb502e96620bb8781233b2d64b9583fdb3953e1d6ce1ab0db95685d&scene=21#wechat_redirect) 以及 [**pg_quack**](https://mp.weixin.qq.com/s?__biz=MzU1OTgxMjA4OA==&mid=2247485686&idx=1&sn=2e26bdbeb2d9393e31a1ec39124be056&scene=21#wechat_redirect) 这样的 PG 生态项目，能让 PostgreSQL 充分利用 DuckDB 带来的完整分析性能红利！

ParadeDB 与 DuckDB 的出现让 PostgreSQL 的分析性能来到了 OLAP 的第一梯队与金字塔尖，弥补了 PostgreSQL 在 OLAP 性能这最后一块关键短板。

---

## 分久必合的数据库领域

数据库诞生伊始，并没有 OLTP 与 OLAP 的分野。OLAP 数据仓库从数据库中“独立”出来，已经是上世纪九十年代时候的事了 —— 因为传统的 OLTP 数据库难以支撑起分析场景下的查询模式，数据量与性能要求。

在相当一段时间里，数据处理的最佳实践是使用 MySQL / PG 处理 OLTP 工作负载，并通过 ETL 将数据同步到专用的 OLAP 组件中去处理，比如 Greenplum, ClickHouse, Doris, Snowflake 等等。  

与许多 “专用数据库” 一样，专业的 OLAP 组件的优势往往在于**性能** —— 相比原生 PG 、MySQL 上有 1～3 个数量级的提升；而代价则是数据冗余、 大量不必要的数据搬运工作、分布式组件之间缺乏一致性、额外的专业技能带来的复杂度成本、学习成本、以及人力成本、 额外的软件许可费用、极其有限的查询语言能力、可编程性、可扩展性、有限的工具链、以及与OLTP 数据库相比更差的数据完整性和可用性 —— **但这是一个合理的利弊权衡**。

然而天下大势，**分久必合，合久必分**。[硬件遵循摩尔定律又发展了三十年](http://mp.weixin.qq.com/s?__biz=MzU5ODAyNTM5Ng==&mid=2247486489&idx=1&sn=f2be1be496de46ac5ca816ac39cfdf24&chksm=fe4b39c2c93cb0d4ff50dd6962370523a6271eab478fe9174c0c7a88fc88ea05fd3e51313ad3&scene=21#wechat_redirect)，性能翻了几个数量级，成本下降了几个数量级。在 2024 年的当下，x86 单机可以达到 512 核几个TB的内存，单卡 NVMe SSD 可达 64TB，全闪单机柜 2PB ；S3 这样对象存储更是能实现几乎没有上限的存储。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

硬件的发展解决了数据量的问题，而数据库软件的发展（PostgreSQL，ParadeDB，DuckDB）解决了查询模式的问题，而这导致分析领域 —— 所谓的“大数据” 行业基本工作假设面临挑战。

正如 DuckDB 发表的宣言《[**大数据已死**](http://mp.weixin.qq.com/s?__biz=MzU1NTg2ODQ5Nw==&mid=2247488697&idx=1&sn=53f5016903eb5560c2f3006a261385db&chksm=fbcc9c87ccbb1591aa572fe64ebbd6559ff69fe401faecc086dc9aa2d263d132f025a312030b&scene=21#wechat_redirect)》所主张的：**大数据时代已经结束了** —— 大多数人并没有那么多的数据，大多数数据也很少被查询。大数据的前沿随着软硬件发展不断后退，99% 的场景已经不再需要所谓“大数据”了。

如果 99% 的场景甚至都可以放在一台计算机上用单机/主从的 DuckDB 或 PostgreSQL 搞定，那么使用专用的分析组件还有多少意义？如果每台手机都可以自由自主收发短信，那么 BP 机还有什么存在价值？（北美医院还在用BP机，正好比也还有 1% 不到的场景也许真的需要“大数据”）

基本工作假设的变化，将重新推动数据库世界从百花齐放的“合久必分”阶段，走向“分久必合”的阶段，从大爆发到大灭绝，大浪淘沙中，新的大一统超融合数据库将会出现，重新统一 OLTP 与 OLAP。而承担重新整合数据库领域这一使命的会是谁？

---

## 吞食天地的 PostgreSQL

数据库领域有许多“细分领域”：时序数据库，地理空间数据库，文档数据库，搜索数据库，图数据库，向量数据库，消息队列，对象数据库。而 PostgreSQL 在任何一个领域都不会缺席。

一个 PostGIS 插件，成为了地理空间事实标准；一个 TimescaleDB 扩展，让一堆“通用”时序数据库尴尬的说不出话来；一个向量扩展 [**PGVector**](http://mp.weixin.qq.com/s?__biz=MzU5ODAyNTM5Ng==&mid=2247485589&idx=1&sn=931f2d794e9b8486f623f746db9f00cd&chksm=fe4b3d4ec93cb4584c9bb44b1f347189868b6c8367d8c3f8dd8703d1a906786a55c900c23761&scene=21#wechat_redirect) 插件，更是让整个**[专用向量数据库细分领域](http://mp.weixin.qq.com/s?__biz=MzU5ODAyNTM5Ng==&mid=2247486505&idx=1&sn=a585c9ff22a81a8efe6b87ce9bd66cb1&chksm=fe4b39f2c93cb0e4c5d46f54e7ba9309dc0d66b5ac73bfe6722cc39f3959e47ae78210aeea1f&scene=21#wechat_redirect)** 变成笑话。

同样的事情已经发生过很多次，而现在，我们将在拆分最早，地盘最大的一个子领域 OLAP 分析中再次见证这一点。但 PostgreSQL 要替代的可不仅仅是 OLAP 数仓，它的野望是整个数据库世界！

[![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)](http://mp.weixin.qq.com/s?__biz=MzU5ODAyNTM5Ng==&mid=2247486931&idx=1&sn=91dbe43bb6d26c760c532f4aa8d6e3cb&chksm=fe4b3808c93cb11e00194655a49bf7aa0d4d05a61a9b06ffcc57017c633de17066443ec62b6d&scene=21#wechat_redirect)

然 PostgreSQL 有何德何能，可当此大任？诚然 PostgreSQL 先进，但 Oracle 也先进；PostgreSQL 开源，但 MySQL 也开源。PostgreSQL **先进且开源**，这是它与 Oracle / MySQL 竞争的底气，但要说其独一无二的特点，那还得是它的**极致可扩展性，与繁荣的扩展生态**！

**![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)**

用户[选择 PostgreSQL 的原因](http://mp.weixin.qq.com/s?__biz=MzU5ODAyNTM5Ng==&mid=2247485216&idx=1&sn=1b59c7dda5f347145c2f39d2679a274d&chksm=fe4b32fbc93cbbed574358a3bcf127dd2e4f458638b46efaee1a885a5702a66a5d9ca18e3f90&scene=21#wechat_redirect)：开源，先进，**扩展**。

PostgreSQL 并不是一个简单的关系型数据库，而是一个数据管理的抽象框架，**具有囊括一切，吞噬整个数据库世界的力量**。而它的核心竞争力（除了开源与先进）来自**可扩展性**，即基础设施的**可复用性**与扩展插件的**可组合性**。

---

PostgreSQL 允许用户开发功能模块，复用数据库公共基础设施，以最低的成本交付功能。例如，仅有两千行代码的向量数据库扩展 pgvector 与百万行代码的 PostgreSQL 在复杂度上相比可以说微不足道，但正是这“微不足道”的扩展，实现了完整的向量数据类型与索引能力，干翻了几乎所有专用向量数据库。

为什么？因为 PGVECTOR 作者不需要操心数据库的通用额外复杂度：事务 ACID，故障恢复，备份PITR，高可用，访问控制，监控，部署，三方生态工具，客户端驱动这些需要成百上千万行代码才能解决好的问题，只需要关注自己所需问题的本质复杂度即可。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

向量数据库哪家强？

再比如，ElasticSearch 基于 Lucene 搜索库开发，而 Rust 生态有一个改进版的下一代 Tantivy 全文搜索库作为 Lucene 的替代；而 ParadeDB 只需要将其封装对接到 PostgreSQL 的接口上，即可提供比肩 ElasticSearch 的搜索服务。更重要的是，它可以站在 PostgreSQL 巨人的肩膀上，借用 PG 生态的全部合力（例如，与 PG Vector 做混合检索），不讲武德地用数据库全能王的力量，去与一个专用数据库单品来对比。

---

可扩展性带来的另一点巨大优势是扩展的**可组合性**，让不同扩展相互合作，产生出 1+1 >> 2 的协同效应。例如，TimescaleDB 可以与 PostGIS 组合使用，提供时空数据支持；再比如，提供全文检索能力的 BM25 扩展可以和提供语义模糊检索的 PGVector 扩展组合使用，提供混合检索能力。

再比如，**分布式**扩展 Citus 可以将单机主从数据库集群，原地升级改造为透明水平分片的分布式数据库集群。而这个能力是可以与其他功能正交组合的，因此，PostGIS 可以成为分布式地理数据库，PGVector 可以成为分布式向量数据库，ParadeDB 可以成为分布式全文搜索数据库，诸如此类。

---

更强大的地方在于，扩展插件是**独立演进**的，不需要繁琐的主干合并，联调协作。因此可以 Scale  —— PG 的可扩展性允许无数个团队并行探索数据库前研发展方向，而扩展全部都是的可选的，不会影响主干核心能力的稳定性。那些非常强大成熟的特性，则有机会以稳定的形态进入主干中。

通过极致可扩展性的魔法，PostgreSQL 做到了**守正出奇，实现了主干极致稳定性与功能敏捷性的统一。**扎实的基本盘配上惊人的演进速度，让它成为了数据库世界中的一个异数，改变了数据库世界的游戏规则。

---

## 改变游戏规则的玩家

**PostgreSQL 的出现，改变了数据库领域的游戏规则**：任何试图开发“新数据库内核”的团队，都需要经过这道试炼与考验 —— 相比开源免费、功能齐备的 Postgres，价值点在哪里？

至少到硬件出现革命性突破前，实用的通用数据库新内核都不太可能诞生了，因为任何单一数据库都无法与所有扩展加持下的 PG 在**整体实力**上相抗衡 —— 包括 Oracle，因为 PG 还有开源免费的必杀技。

而某个细分领域的数据库产品，如果能在单点属性（通常是性能）上相比 PostgreSQL 实现超过一个数量级的优势，那也许还有一个专用数据库的生态位存在。但通常用不了多久，便会有 PostgreSQL 生态的开源替代扩展插件滚滚而来。因为选择开发 PG 扩展，而不是一个完整数据库的团队会在追赶复刻速度上有碾压性优势！

因此，如果按照这样的逻辑展开，PostgreSQL 生态的雪球只会越滚越大，随着优势的积累，不可避免地进入一家独大的状态。在几年的时间内，实现 Linux 内核在服务器操作系统领域的状态。而各种开发者调研报告，数据库流行趋势都在印证着这一点。

[**![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)**](http://mp.weixin.qq.com/s?__biz=MzU5ODAyNTM5Ng==&mid=2247485933&idx=3&sn=ea360aa7a59a4cd23ad5f9a9f415a0a0&chksm=fe4b3c36c93cb520bda4596136e927d7cf92c597a76c04077c256588b2428202bdb7f004c08b&scene=21#wechat_redirect)

[**StackOverflow 2023 调研结果，PostgreSQL 三项全能王**](http://mp.weixin.qq.com/s?__biz=MzU5ODAyNTM5Ng==&mid=2247485933&idx=3&sn=ea360aa7a59a4cd23ad5f9a9f415a0a0&chksm=fe4b3c36c93cb520bda4596136e927d7cf92c597a76c04077c256588b2428202bdb7f004c08b&scene=21#wechat_redirect)

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

[**StackOverflow过去7年的数据库指标走势**](http://mp.weixin.qq.com/s?__biz=MzU5ODAyNTM5Ng==&mid=2247485933&idx=3&sn=ea360aa7a59a4cd23ad5f9a9f415a0a0&chksm=fe4b3c36c93cb520bda4596136e927d7cf92c597a76c04077c256588b2428202bdb7f004c08b&scene=21#wechat_redirect)

在引领潮流的 HackerNews StackOverflow 上，PostgreSQL 早已成为了最受欢迎的数据库。许多新的开源项目都默认使用 PostgreSQL 作为首要，甚至唯一的数据库 —— 例如，给各种数据库做模式管理的 Bytebase。《[云时代数据库DevOps：硅谷调研](https://mp.weixin.qq.com/s?__biz=MzkxODMzMjk1Ng==&mid=2247489525&idx=1&sn=b6e63250c22c733737857dcf86b5c3ea&scene=21#wechat_redirect)》也提出，许多新一代互联网公司都开始积极拥抱并 All in PostgreSQL。

正如《[技术极简主义：一切皆用 Postgres](http://mp.weixin.qq.com/s?__biz=MzU5ODAyNTM5Ng==&mid=2247486931&idx=1&sn=91dbe43bb6d26c760c532f4aa8d6e3cb&chksm=fe4b3808c93cb11e00194655a49bf7aa0d4d05a61a9b06ffcc57017c633de17066443ec62b6d&scene=21#wechat_redirect) 》所言：简化技术栈、减少组件、加快开发速度、降低风险并提供更多功能特性的方法之一就是 **“一切皆用 Postgres”**。Postgres 能够取代许多后端技术，包括 MySQL，Kafka、RabbitMQ、ElasticSearch，Mongo和 Redis，至少到数百万用户时都毫无问题。**一切皆用 Postgres** ，已经不再是少数精英团队的前沿探索，而是成为了一种进入主流视野的最佳实践。

  

---

## 还有什么可以做的？

我们已经不难预见到数据库领域的终局。但我们又能做什么，又应该做什么呢？

PostgreSQL 对于绝大多数场景都已经是一个足够完美的数据库内核了，在这个前提下，数据库内核[卡脖子纯属无稽之谈](http://mp.weixin.qq.com/s?__biz=MzU5ODAyNTM5Ng==&mid=2247486379&idx=1&sn=b751c51a2b73e43e61487abfdc073da3&chksm=fe4b3e70c93cb766625f9e18a92eabe31af437eb0fd7ed9d38b95750c743ce44934433c4dd66&scene=21#wechat_redirect)。这些Fork PostgreSQL 和 MySQL 并以内核魔改作为卖点的所谓“[数据库](http://mp.weixin.qq.com/s?__biz=MzU5ODAyNTM5Ng==&mid=2247486804&idx=1&sn=069c7655d0b38b0e7421e847cc7d5576&chksm=fe4b388fc93cb199756c2faad19dc2c82f6cee631a3c5ce1f505ce1e5545645652d6fa89a8e3&scene=21#wechat_redirect)”基本没啥出息。

这好比今天我们看 Linux 操作系统内核一样，尽管市面上有这么多的 Linux 操作系统发行版，但大家都选择使用同样的 Linux 内核，吃饱了撑着魔改内核属于没有困难创造困难也要上，会被业界当成山炮看待。

同理，数据库内核本身已经不再是主要矛盾，焦点将会集中到两个方向上 —— 数据库**扩展**与数据库**服务**！前者体现为数据库内部的可扩展性， 后者体现为数据库外部的可组合性。而竞争的形式，正如操作系统生态一样 —— 集中于**数据库发行版**上。对于数据库领域来说，只有那些以扩展和服务作为核心价值主张的发行版，才有最终成功的可能。

做内核的厂商不温不火，MariaDB 作为 MySQL 的亲爹 Fork 甚至都已经濒临退市，而白嫖内核自己做服务与扩展卖 RDS 的 AWS 可以赚的钵满盆翻。投资机构已经出手了许多 PG 生态的扩展插件与服务发行版：Citus，TimescaleDB，Hydra，PostgresML，ParadeDB，FerretDB，StackGres，Aiven，Neon，Supabase，Tembo，PostgresAI，以及我们正在做的 Pigsty 。

---

PostgreSQL 生态中的一个困境就是，许多扩展插件，生态工具都是独立演进，各自为战的，没有一个整合者能将他们凝聚起来形成合力。例如，提供分析的 Hydra 会打一个包一个 Docker 镜像， PostgresML 也会打自己的包和镜像，各家只发行加装了自己扩展的 Postgres 镜像。而这些朴素的镜像与包也距离 RDS 这样完整的数据库服务相距甚远。

即使是类似于 AWS RDS 这样的服务提供商与生态整合者，在诸多扩展面前也依然力有所不逮，只能提供其中的少数。更多的强力扩展出于各种原因（AGPLv3 协议，多租户租赁带来的安全挑战）而无法使用。从而难以发挥 PostgreSQL 生态扩展的协同增幅作用。  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

许多关键扩展在RDS中并不可用  

扩展是 PostgreSQL 的灵魂，无法自由使用扩展的 Postgres 就像做菜不放盐。只能和 MySQL 放在同一个 RDS 的框子里同台，龙游浅水，虎落平阳。而这正是我们想要解决的问题。

---

  

## 知行合一的实践：Pigsty

虽然接触 MySQL 和 MSSQL 要早得多，但我在 2015 年第一次上手 PostgreSQL 时，就相信它会是数据库领域的未来了。快十年过去，我也从 PG 的使用者，管理者，变为了贡献者，开发者。也不断见证着 PG 走向这一目标。

在与形形色色的用户沟通交流中，我早已发现数据库领域的木桶短板不是内核 —— 现有的 PostgreSQL 已经足够好了，而是**用好数据库内核本身的能力**，这也是 RDS 这样的服务赚的钵满盆翻的原因。

但我希望这样的能力，应该像自由软件运动所倡导的理念那样，像 PostgreSQL 内核本身一样 —— 普及到每一个用户手中，而不是必须向赛博空间上的封建云领主花大价钱租赁。

所以我打造了 **[Pigsty](http://mp.weixin.qq.com/s?__biz=MzU5ODAyNTM5Ng==&mid=2247485518&idx=1&sn=3d5f3c753facc829b2300a15df50d237&chksm=fe4b3d95c93cb4833b8e80433cff46a893f939154be60a2a24ee96598f96b32271301abfda1f&scene=21#wechat_redirect)** —— 一个开箱即用的开源 PostgreSQL 数据库发行版，旨在凝聚 PostgreSQL 生态扩展的合力，并把提供优质数据库服务的能力普及到每个用户手中。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

Pigsty 是 **P**ostgreSQL **i**n **G**reat **STY**le 的缩写，意为 **PostgreSQL 的全盛状态**。

我们提出了六点核心价值主张，对应 PostgreSQL 数据库服务中的的六个核心问题：**Postgres 的可扩展性**，**基础设施的可靠性**，**图形化的可观测性**，**服务的可用性**，**工具的可维护性**，以及**扩展模块和三方组件可组合性**。

Pigsty 六点价值主张的首字母合起来，则为 Pigsty 提供了另外一种缩写解释：

> **P**ostgres, **I**nfras, **G**raphics, **S**ervice, **T**oolbox, **Y**ours.
> 
> 属于你的图形化 Postgres 基础设施服务工具箱。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**可扩展的 PostgreSQL** 是这个发行版中最重要的价值主张。在刚刚发布的 [**Pigsty v2.6**](http://mp.weixin.qq.com/s?__biz=MzU5ODAyNTM5Ng==&mid=2247487025&idx=1&sn=c32f102718e3e9cf55cdefa7133f259f&chksm=fe4b3beac93cb2fc25c2c2c4f9ea74d4353e565ef90c5e2bbaf5881a3a031cbdff706971af9a&scene=21#wechat_redirect) 中，我们整合了上面提到的 DuckdbFDW 与 ParadeDB 扩展，这两个插件让 PostgreSQL 的分析能力得到史诗级增强，而我们确保每个用户都能轻松用得上这样的能力。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

来自 ParadeDB 创始人与 DuckdbFDW 作者的感谢致意

我们希望整合 PostgreSQL 生态里的各种力量，并将其凝聚在一起形成合力，打造一个数据库世界中的 **Ubuntu** 发行版。而我相信，内核之争早已尘埃落定，而这里才会是数据库世界的未来竞争焦点。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

开发者朋友们，你们的选择会塑造数据库世界的未来。希望我的这些工作，可以帮助你们更好的用好这世界上最先进的开源数据库内核 —— PostgreSQL。

---

## 参考阅读

[Pigsty v2.6：PostgreSQL 踢馆 OLAP](http://mp.weixin.qq.com/s?__biz=MzU5ODAyNTM5Ng==&mid=2247487025&idx=1&sn=c32f102718e3e9cf55cdefa7133f259f&chksm=fe4b3beac93cb2fc25c2c2c4f9ea74d4353e565ef90c5e2bbaf5881a3a031cbdff706971af9a&scene=21#wechat_redirect)  

[技术极简主义：一切皆用Postgres](http://mp.weixin.qq.com/s?__biz=MzU5ODAyNTM5Ng==&mid=2247486931&idx=1&sn=91dbe43bb6d26c760c532f4aa8d6e3cb&chksm=fe4b3808c93cb11e00194655a49bf7aa0d4d05a61a9b06ffcc57017c633de17066443ec62b6d&scene=21#wechat_redirect)  

[PG生态新玩家ParadeDB](http://mp.weixin.qq.com/s?__biz=MzU5ODAyNTM5Ng==&mid=2247486913&idx=1&sn=3b7d8cf3f0e323932aba52c897f3c7a4&chksm=fe4b381ac93cb10cc6175c4c7978b5903946d369fe0084fbae5edf76ab08d84134260f28dffc&scene=21#wechat_redirect)  

[DBA会被云淘汰吗？](http://mp.weixin.qq.com/s?__biz=MzU5ODAyNTM5Ng==&mid=2247486903&idx=1&sn=01c57499f41e8f51045bb8dd52586595&chksm=fe4b386cc93cb17a2d2fad903e809107162cc1e67e8ad7c8bfdd51de657c97f32f912cabe550&scene=21#wechat_redirect)  

[令人惊叹的PostgreSQL可伸缩性](http://mp.weixin.qq.com/s?__biz=MzU5ODAyNTM5Ng==&mid=2247486832&idx=1&sn=6b6b5f03b77c8a607f43f323fdf9ee7d&chksm=fe4b38abc93cb1bd84e3360b857016a9be3329c91d47c998fe73dc37d1f4b2c5571161fb0ff2&scene=21#wechat_redirect)  

[中国对PostgreSQL的贡献约等于零吗？](http://mp.weixin.qq.com/s?__biz=MzU5ODAyNTM5Ng==&mid=2247486763&idx=1&sn=d05db7200faa6b23f61ca51328439833&chksm=fe4b38f0c93cb1e6cebb4818d22555bbb94ed33f0a191b149383127c12ae3091add20a53f102&scene=21#wechat_redirect)  

[展望PostgreSQL的2024 (Jonathan Katz)](http://mp.weixin.qq.com/s?__biz=MzU5ODAyNTM5Ng==&mid=2247486752&idx=1&sn=b10354a0cee5b0ccd88df606787e1297&chksm=fe4b38fbc93cb1ed39b86882b596020ba3d2f5901bea530bf09cf2519e1ad248d1f93f648180&scene=21#wechat_redirect)  

[2023年度数据库：PostgreSQL (DB-Engine)](http://mp.weixin.qq.com/s?__biz=MzU5ODAyNTM5Ng==&mid=2247486745&idx=1&sn=b92be029db148f53239c29bea912fc78&chksm=fe4b38c2c93cb1d443ac8e6babe4d735f09404b6fac23c6045dd959f291bc28f13287571a189&scene=21#wechat_redirect)

[MySQL的正确性为何如此拉垮？](http://mp.weixin.qq.com/s?__biz=MzU5ODAyNTM5Ng==&mid=2247486710&idx=1&sn=261e4754df6c85954b50d8f68f277abe&chksm=fe4b392dc93cb03bf26554a7a232f6217b8aa78d7e35ce0566d9404dc9526d3776141e628a2b&scene=21#wechat_redirect)

[向量数据库凉了吗？](http://mp.weixin.qq.com/s?__biz=MzU5ODAyNTM5Ng==&mid=2247486505&idx=1&sn=a585c9ff22a81a8efe6b87ce9bd66cb1&chksm=fe4b39f2c93cb0e4c5d46f54e7ba9309dc0d66b5ac73bfe6722cc39f3959e47ae78210aeea1f&scene=21#wechat_redirect)

[重新拿回计算机硬件的红利](http://mp.weixin.qq.com/s?__biz=MzU5ODAyNTM5Ng==&mid=2247486489&idx=1&sn=f2be1be496de46ac5ca816ac39cfdf24&chksm=fe4b39c2c93cb0d4ff50dd6962370523a6271eab478fe9174c0c7a88fc88ea05fd3e51313ad3&scene=21#wechat_redirect)

[数据库真被卡脖子了吗？](http://mp.weixin.qq.com/s?__biz=MzU5ODAyNTM5Ng==&mid=2247486379&idx=1&sn=b751c51a2b73e43e61487abfdc073da3&chksm=fe4b3e70c93cb766625f9e18a92eabe31af437eb0fd7ed9d38b95750c743ce44934433c4dd66&scene=21#wechat_redirect)

[PG查询优化：观宏之道](https://mp.weixin.qq.com/s?__biz=MzU5ODAyNTM5Ng==&mid=2247486349&idx=1&sn=ade54570a726c0aee0d23444372bd6b9&scene=21#wechat_redirect "PG查询优化：观宏之道") 

[FerretDB：假扮成MongoDB的PostgreSQL](https://mp.weixin.qq.com/s?__biz=MzU5ODAyNTM5Ng==&mid=2247486241&idx=1&sn=f39b87095837b042e74f55f8e60bb7a9&scene=21#wechat_redirect "FerretDB：假扮成MongoDB的PostgreSQL")

[如何用 pg_filedump 抢救数据？](https://mp.weixin.qq.com/s?__biz=MzU5ODAyNTM5Ng==&mid=2247486234&idx=1&sn=d1273152e624fb31bf7be2c8f3991315&scene=21#wechat_redirect "如何用 pg_filedump 抢救数据？")

[PGSQL x Pigsty: 数据库全能王来了](https://mp.weixin.qq.com/s?__biz=MzU5ODAyNTM5Ng==&mid=2247486215&idx=1&sn=52ce37a537336a6d07448f35c7bc4cfd&scene=21#wechat_redirect "PGSQL x Pigsty: 数据库全能王来了")

[Pigsty 特性与快速上手](https://mp.weixin.qq.com/s?__biz=MzU5ODAyNTM5Ng==&mid=2247486135&idx=1&sn=7d9c4920e94efba5d0e0b6af467f596c&scene=21#wechat_redirect "Pigsty 特性与快速上手")

[PG先写脏页还是先写WAL？](https://mp.weixin.qq.com/s?__biz=MzU5ODAyNTM5Ng==&mid=2247486101&idx=1&sn=30dfc9b11f4f812e699af2711f93931a&scene=21#wechat_redirect "PG先写脏页还是先写WAL？")

[PostgreSQL：世界上最成功的数据库](https://mp.weixin.qq.com/s?__biz=MzU5ODAyNTM5Ng==&mid=2247485685&idx=1&sn=688f6d6d0f4128d7f77d710f04ff9024&scene=21#wechat_redirect "PostgreSQL：世界上最成功的数据库")

[ISD数据集：分析全球120年气候变化](https://mp.weixin.qq.com/s?__biz=MzU5ODAyNTM5Ng==&mid=2247485647&idx=1&sn=1ca65ee357516a06dca7ec13fa679f9a&scene=21#wechat_redirect "ISD数据集：分析全球120年气候变化")

[AI大模型与向量数据库 PGVECTOR](https://mp.weixin.qq.com/s?__biz=MzU5ODAyNTM5Ng==&mid=2247485589&idx=1&sn=931f2d794e9b8486f623f746db9f00cd&scene=21#wechat_redirect "AI大模型与向量数据库 PGVECTOR")

[更好的开源RDS替代：Pigsty](https://mp.weixin.qq.com/s?__biz=MzU5ODAyNTM5Ng==&mid=2247485518&idx=1&sn=3d5f3c753facc829b2300a15df50d237&scene=21#wechat_redirect "更好的开源RDS替代：Pigsty")

[PostgreSQL 到底有多强？](https://mp.weixin.qq.com/s?__biz=MzU5ODAyNTM5Ng==&mid=2247485240&idx=1&sn=9052f03ae2ef21d9e21037fd7a1fa7fe&scene=21#wechat_redirect "PostgreSQL 到底有多强？")

[为什么PostgreSQL是最成功的数据库？](https://mp.weixin.qq.com/s?__biz=MzU5ODAyNTM5Ng==&mid=2247485216&idx=1&sn=1b59c7dda5f347145c2f39d2679a274d&scene=21#wechat_redirect "为什么PostgreSQL是最成功的数据库？")

[PG与Pigsty用户需求问卷调研结果](https://mp.weixin.qq.com/s?__biz=MzU5ODAyNTM5Ng==&mid=2247484979&idx=1&sn=6b7afac9905b3d07ed7c1d43f8a2e464&scene=21#wechat_redirect "PG与Pigsty用户需求问卷调研结果")

[高可用PgSQL集群架构设计与落地](https://mp.weixin.qq.com/s?__biz=MzU5ODAyNTM5Ng==&mid=2247484546&idx=1&sn=f89c7c3b87b24ee536bfc56b8b51c2d5&scene=21#wechat_redirect "高可用PgSQL集群架构设计与落地")

[为什么说PostgreSQL前途无量？](https://mp.weixin.qq.com/s?__biz=MzU5ODAyNTM5Ng==&mid=2247484591&idx=1&sn=a6ab13d93bfa26fca969ba163b01e1d5&scene=21#wechat_redirect "为什么说PostgreSQL前途无量？")

[Postgres本地化排序规则](https://mp.weixin.qq.com/s?__biz=MzU5ODAyNTM5Ng==&mid=2247484489&idx=1&sn=11163ce0afdb14af07619ae587fadb59&scene=21#wechat_redirect "Postgres本地化排序规则")

[PG复制标识详解（Replica Identity）](https://mp.weixin.qq.com/s?__biz=MzU5ODAyNTM5Ng==&mid=2247484483&idx=1&sn=47469a6a57a497a551022b287bf1b31e&scene=21#wechat_redirect "PG复制标识详解（Replica Identity）")

[利用监控系统诊断PG慢查询](https://mp.weixin.qq.com/s?__biz=MzU5ODAyNTM5Ng==&mid=2247484478&idx=1&sn=ea44675df79b60a12273e78b358bb557&scene=21#wechat_redirect "利用监控系统诊断PG慢查询")

[数据库集群管理概念与实体命名规范](https://mp.weixin.qq.com/s?__biz=MzU5ODAyNTM5Ng==&mid=2247484195&idx=1&sn=cea57269d0ffec585547727170887441&scene=21#wechat_redirect "数据库集群管理概念与实体命名规范")

[PostgreSQL的KPI](https://mp.weixin.qq.com/s?__biz=MzU5ODAyNTM5Ng==&mid=2247484164&idx=1&sn=d69a31948d96507aca10a48587ea275c&scene=21#wechat_redirect "PostgreSQL的KPI")

[PostgreSQL监控系统Pigsty概述](https://mp.weixin.qq.com/s?__biz=MzU5ODAyNTM5Ng==&mid=2247484189&idx=1&sn=19d4381c7ec4bc4498bd56c5ee9f916b&scene=21#wechat_redirect "PostgreSQL监控系统Pigsty概述")

[故障档案：PG安装扩展导致无法连接](https://mp.weixin.qq.com/s?__biz=MzU5ODAyNTM5Ng==&mid=2247483969&idx=1&sn=c5264dc6cd36d5696138bad085a72b37&scene=21#wechat_redirect "故障档案：PG安装扩展导致无法连接")

[PostgreSQL中的表锁](https://mp.weixin.qq.com/s?__biz=MzU5ODAyNTM5Ng==&mid=2247483964&idx=1&sn=b128086019256401b135ea0aa07b0c1c&scene=21#wechat_redirect "PostgreSQL中的表锁")

[把PG放入Docker是一个好主意吗？](https://mp.weixin.qq.com/s?__biz=MzU5ODAyNTM5Ng==&mid=2247483950&idx=1&sn=9c233f5e9a690706ae96ceabb938bff9&scene=21#wechat_redirect "把PG放入Docker是一个好主意吗？")

[PostgreSQL监控系统概览](https://mp.weixin.qq.com/s?__biz=MzU5ODAyNTM5Ng==&mid=2247483915&idx=1&sn=1793258171169a5b4a75944302f1ae3a&scene=21#wechat_redirect "PostgreSQL监控系统概览")

[pg_dump导致的血案](https://mp.weixin.qq.com/s?__biz=MzU5ODAyNTM5Ng==&mid=2247483863&idx=1&sn=4b6851c0db5d2862e8698219800e28a7&scene=21#wechat_redirect "pg_dump导致的血案")

[PostgreSQL数据页面损坏修复](https://mp.weixin.qq.com/s?__biz=MzU5ODAyNTM5Ng==&mid=2247483850&idx=1&sn=b10652fc434e3f17f56bcdeaacc91974&scene=21#wechat_redirect "PostgreSQL数据页面损坏修复")

[PostgreSQL关系膨胀:原理，监控与处理](https://mp.weixin.qq.com/s?__biz=MzU5ODAyNTM5Ng==&mid=2247483768&idx=1&sn=8a5005a95e874e6a13522cab0b5c1883&scene=21#wechat_redirect "PostgreSQL关系膨胀:原理，监控与处理")

[探探PostgreSQL开发规约](https://mp.weixin.qq.com/s?__biz=MzU5ODAyNTM5Ng==&mid=2247483719&idx=1&sn=1a0a04fe974ea20026d378bd65cda57f&scene=21#wechat_redirect "探探PostgreSQL开发规约")

[并发异常那些事](https://mp.weixin.qq.com/s?__biz=MzU5ODAyNTM5Ng==&mid=2247483715&idx=1&sn=b17d3d8920a596c383745abd0dce0584&scene=21#wechat_redirect "并发异常那些事")

[PG好处都有啥？](https://mp.weixin.qq.com/s?__biz=MzU5ODAyNTM5Ng==&mid=2247483706&idx=1&sn=b842684b41ac6dde8310448ae0a81a76&scene=21#wechat_redirect "PG好处都有啥？")

[IP归属地查询的高效实现](https://mp.weixin.qq.com/s?__biz=MzU5ODAyNTM5Ng==&mid=2247483692&idx=1&sn=0cdb3609daf22fa2a5614d280da96b66&scene=21#wechat_redirect "IP归属地查询的高效实现")

[PostGIS高效解决行政区划归属查询问题](https://mp.weixin.qq.com/s?__biz=MzU5ODAyNTM5Ng==&mid=2247483688&idx=1&sn=0b08c7c47e28ceae77f89a78d38b029f&scene=21#wechat_redirect "PostGIS高效解决行政区划归属查询问题")

  

![](https://mmbiz.qlogo.cn/mmbiz_jpg/e5xS3Vwicpl8vEzwicmWCF70Y4HnvILGe2WlJnGNdXqQOyUq83icsPshyXHt8ytQhusdhppIKm6xs3TbYzVlHbJWA/0?wx_fmt=jpeg)

冯若航

![赞赏二维码](https://mp.weixin.qq.com/s?__biz=MzU5ODAyNTM5Ng==&mid=2247487055&idx=1&sn=9d7bd8b6d9b07478dba7f87d0a663535&chksm=fe4b3b94c93cb282f8d9c228425d01f29f753d4c9fbf661a1bf8ea4b0ec6a5bf6d7f2ee58f86&mpshare=1&scene=24&srcid=0321ZPXalWoAu4OMzVItiwKp&sharer_shareinfo=cd05f77a0fa49551ddcc03dbddd34c85&sharer_shareinfo_first=cd05f77a0fa49551ddcc03dbddd34c85&key=daf9bdc5abc4e8d01f7d0672ef5c0a530df722741f8b33dc05f89af66d3d163665f89cd7bde8ceea055709387c2c2bb7d237969e18fedfeb149fcad878b630bf363f67d3dcd72cb89a91a7d0e4039c8e4392a15adffcbecb8520bbbe7b8edc709cb8810c9ad2fc652b1da1e84ad84060a9e557150f65f46bef2af600b4d749ae&ascene=0&uin=MTEwNTU1MjgwMw%3D%3D&devicetype=Windows+11+x64&version=63090b19&lang=zh_CN&countrycode=CN&exportkey=n_ChQIAhIQqvK7UwyFhGevjhNg0si8gBLmAQIE97dBBAEAAAAAAFtAFDmGL7AAAAAOpnltbLcz9gKNyK89dVj0rXeLLIO8ZCZTDE%2FXxchPEjQWB2z7GNPtlKwOSZ%2BdkhpLfv3q34YMNIA8qUwGNHicKy%2FtpkkLzE143QFpdFK9EldrvPBgd0JUYFxVDn%2BZJ6PnQcso4YBS2TSm6XkjFgCiKKmVUIUh6naU6NzWYNRKACyuTqGtq1G%2B3zz1lYtx4R%2FLxFdnCvmiG%2BEJJ%2Fr%2Fu%2FMuGv5odt0MLxskriLYW4oCPuCWOFMV3hG0YkweiDkWFDGDSXg4rpX%2FPAWTTiRUXX23&acctmode=0&pass_ticket=ZQOKcjFpayxtKnfIR8COLqoCkbChMK25CeT5%2BT07%2FHFVNsHSoQYtYe64Y98UWhfi&wx_header=1&fasttmpl_type=0&fasttmpl_fullversion=7350504-zh_CN-zip&fasttmpl_flag=1)喜欢作者

4人喜欢

![](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

数据库60

PostgreSQL81

数据库 · 目录

上一篇理解时间：闰年，闰秒，时间与时区的原理下一篇数据库小圈，公众号大全

阅读原文

阅读 5.7万

​