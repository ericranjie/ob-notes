耗子当年好像是偶然的一个机会才被允许进入阿里的机密级会议，然后才明白了双十一的原理...

腾讯程序员 腾讯技术工程

 _2022年06月23日 11:35_ _广东_

![图片](https://mmbiz.qpic.cn/mmbiz_gif/j3gficicyOvasIjZpiaTNIPReJVWEJf7UGpmokI3LL4NbQDb8fO48fYROmYPXUhXFN8IdDqPcI1gA6OfSLsQHxB4w/640?wx_fmt=gif&wxfrom=13&tp=wxpic)

  

Apache 软件基金会（即 Apache Software Foundation，简称为 ASF）于近日正式宣布，Apache InLong（应龙） 从孵化器成功毕业，成为基金会顶级项目。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvavZUO55FfNL5S6Z9Gr39mNOXOKpKnIvgmn4H0BMXfHgsmo7cXBAel199HpKkz3bePaUNkmicp0J7kg/640?wx_fmt=jpeg&wxfrom=13&tp=wxpic)

[The Apache Software Foundation Announces Apache® InLong™ as a Top-Level Project](https://www.globenewswire.com/en/news-release/2022/06/22/2467186/17401/en/The-Apache-Software-Foundation-Announces-Apache-InLong-as-a-Top-Level-Project.html)

Apache InLong 的毕业，标志着业界首个一站式大数据集成 Apache 顶级项目诞生，也标志着第一个由腾讯捐献的 Apache 项目孵化成功，中国本土原生的顶级项目再增一员，恭喜 InLong 社区。Apache 软件基金会是专门为支持开源软件项目而办的一个非盈利性组织，是世界上最大的开源基金会，管理着 2.27 亿多行代码，并 100% 免费向公众提供价值超过 220 亿美元的软件。

Apache InLong 在孵化期间，连续发布 12 个版本，关闭超 2300 个 Issue，来自国内外的社区开发者，一起完成了 Manager 元数据管理重构、基于 Flink SQL 的 Sort ETL 方案、基于标签的跨地域多集群等特性。目前 Apache InLong 正广泛应用于广告、支付、社交、游戏、人工智能等行业，为多领域客户提供高效化、便捷化的数据集成服务。

"InLong 社区专注于为海量数据打造统一的、一站式的数据集成框架，帮助企业简化数据的接入、ETL 和分发过程”，Apache InLong PMC Chair 张超表示，“InLong 的毕业，标志着一个开放、多元、成熟的开源社区成功建立。感谢所有帮助过项目的所有导师、开源社区、贡献者等， 在未来的征程中，项目将继续践行 Apache Way，通过社区开发者的共同努力，助力企业数字化转型”。

### 关于 Apache InLong

Apache InLong（应龙）是一站式的海量数据集成框架，提供自动、安全、可靠和高性能的数据传输能力，方便业务构建基于流式的数据分析、建模和应用。Apache InLong 以腾讯大数据的 TDBank 系统为基础，依托近百万亿级别的数据接入和处理能力，整合了数据采集、汇聚、存储、分拣数据处理全流程，拥有简单易用、灵活扩展、稳定可靠等特性。该项目最初于 2019 年 11 月由腾讯大数据团队捐献到 Apache 孵化器，2022 年 6 月正式毕业成为 Apache 顶级项目。

目前， Apache InLong 已经发布第12个版本，在孵化期间不断更新和完善功能，为社区和企业提供更多元化的开放解决方案。与此同时，InLong 正广泛应用于广告、支付、社交、游戏、人工智能等各个行业领域，为多领域客户提供高效化便捷化服务。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvavZUO55FfNL5S6Z9Gr39mNOol1HUK4rumWxOmDClOMkmUtWLUWbNhossn9SMew9Rib68ZsMM29SCoQ/640?wx_fmt=jpeg&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

作为一个面向大数据集成的开源框架，Apache InLong 拥有架构上的优势，项目在发展的过程中逐渐形成了以下特点：

- 简单易用，基于 SaaS 模式对外服务，用户只需要按主题发布和订阅数据即可完成数据的上报，传输和分发工作。
    
- 稳定可靠，系统源于实际的线上系统，服务近百万亿级的高性能及上千亿级的高可靠数据数据流量，系统稳定可靠。
    
- 功能完善，支持各种类型的数据接入方式，多种不同类型的MQ集成，以及基于配置规则的实时数据ETL和数据分拣落地，并支持以可插拔方式扩展系统能力。
    
- 服务集成，支持统一的系统监控、告警，以及细粒度的数据指标呈现，对于管道的运行情况，以数据主题为核心的数据运营情况，汇总在统一的数据指标平台，并支持通过业务设置的告警信息进行异常告警提醒。
    
- 灵活扩展，全链条上的各个模块基于协议以可插拔方式组成服务，业务可根据自身需要进行组件替换和功能扩展。
    

### Apache InLong 技术亮点

#### 低成本、高性能的 InLong TubeMQ

选用一款消息队列服务，需要考虑成本、性能、稳定性、可靠性、可维护性等方面。在万亿级别的海量数据场景，一般的消息队列服务需要通过大量的机器资源去堆积整体的吞吐能力，会出现机器成本高、超大集群不易维护等问题。InLong TubeMQ 是 Apache InLong 全链路数据集成解决方案自带的一款消息队列服务，相比较业界主流的消息队列服务，拥有低成本、高性能、高稳定性的特点。InLong TubeMQ 是基于有损服务的前提下，采用尽可能保证数据不丢、服务不受阻的思路进行设计，力求方案简单维护简便。在 TubeMQ 的设计里，分区故障并不影响 Topic 的整体对外服务，只要 Topic 有一个分区存活，整体的对外服务就不会受阻。同时，TubeMQ的数据时延 P99 可以做到毫秒级，这样保证了业务可以尽可能快的消费完数据，做到尽可能不丢。另外，TubeMQ独有的数据存储方案设计性能要比 Kafka 的TPS至少高出50%以上（有些机型上还是翻倍的效果），同时借助存储方案的不同，单机容纳的Topic数和分区数更多，进而可以使得集群规模更大，减少维护成本。下图给出了 InLong TubeMQ 和 Kafka、Pulsar 的全方位对比：

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvavZUO55FfNL5S6Z9Gr39mNO0cUWvvvaYwfwsjVrXFwQeKqzodug3ibqHsicmZ1JPAm5tgtsknwm3TWQ/640?wx_fmt=jpeg&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

当然，在整个 Apache InLong 的架构中，由于对消息队列的支持完成了插件化，InLong TubeMQ 并不是完全和系统耦合，而是作为一种可选服务提供给社区用户。用户可根据开发和使用经验，选择其它消息队列服务，比如 Apache Pulsar 和 Apache Kafka。

#### 基于 Flink SQL 的 InLong Sort ETL

随着 Apache InLong 的用户和开发者逐渐增多，更丰富的使用场景和低成本运营诉求越来越强烈，其中，InLong 全链路增加 Transform（T）的需求反馈最多。为了支持该能力，InLong 实现了基于 Apache Flink SQL 的 InLong Sort ETL 方案。首先，基于 Apache Flink SQL 主要有以下方面的考量：

- Flink SQL 拥有强大的表达能力带来的高可扩展性、灵活性，基本上 Flink SQL 能支持社区大多数需求场景。当 Flink SQL 内置的函数不满足需求时，我们还可通过各种UDF来扩展。
    
- Flink SQL 相比 Flink 底层 API 实现开发成本更低，只有第一次需要实现 Flink SQL 的转换逻辑，后续可专注于 Flink SQL 能力本身的构建，比如扩展 Connector、自定义函数UDF等。
    
- 一般来说，Flink SQL 将更健壮、运行也将更稳定。原因在于 Flink SQL 屏蔽了 Flink 底层大量的细节，有强大的社区支持，并且经过大量用户的实践。
    
- 对用户来说，Flink SQL 也更加通俗易懂，特别是对使用过 SQL 用户来说，使用方式简单、熟悉，这有助于用户快速落地。
    
- 对于存量实时任务的迁移，如果其原本就是 SQL 类型的任务，尤其是 Flink SQL 任务，其迁移成本极低，部分情况下甚至都不用做任何改动。
    

基于 Apache Flink SQL 的 InLong Sort ETL 方案，目前已支持 13 种常见的 Data Node，用户也可以基于该方案快速扩展新的 Extract Node 和 Load Node。另外，除了和 InLong Manager/Dashboard 搭配使用，提供全链路数据集成服务（称之为标准架构），InLong Sort 也支持独立运行（称之为轻量化架构），只需要准备 Flink 环境和 InLong Sort，就可以快速完成小规模数据集的 ETL 处理。InLong Sort 整体的技术方案可以见下图：

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvavZUO55FfNL5S6Z9Gr39mNOf9LuevBySvH4icoHqHVnlV7lJltlEIgGJ8gYVzXEe6SDiaB2FrfUorSQ/640?wx_fmt=jpeg&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

#### InLong DataProxy 为海量数据提供路由能力

相比较其他数据集成解决方案，InLong 架构最明显的区别，在数据采集和消息队列中间，多了一个叫做 DataProxy 的服务。InLong DataProxy 主要有连接收敛、路由、数据压缩和协议转换等作用。DataProxy 充当了 InLong 采集端到消息队列的桥梁，当 DataProxy 从 Manager 模块拉取数据流元数据后，数据流和消息队列 Topic 名称对应关系也就确定了。当 DataProxy 收到消息时，会首先发送到 Memory Channel 中进行压缩，并使用本地的 Producer 往后端 Cache 层（即消息队列）发送数据。当消息队列异常出现发送失败时，DataProxy 会将消息缓存到 Disk Channel，也就是本地磁盘中。InLong DataProxy 整体架构基于 Apache Flume，扩展了 Source 层和 Sink 层，并对容灾转发做了优化处理，提升了系统的稳定性。下图为 DataProxy 数据处理流程：

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvavZUO55FfNL5S6Z9Gr39mNOBztyXTh4Ka8EGAgdGvsXVM1WPlWicFA7wVqnzsqZmicuhmvVC0h5dT7A/640?wx_fmt=jpeg&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

#### InLong Audit 独立于数据流的全链路审对账服务

InLong Audit 是独立于 InLong 的配套服务，主要用于对 InLong 全链路的 Agent、DataProxy、Sort 模块的入条数、入流量、出条数、出流量等进行实时审计对账，查看数据流是否有丢失或者重复，方便快速定位数据和服务异常。目前 InLong Audit 对账的粒度有分钟、小时、天三种粒度。InLong Audit 的整体架构图，可以参考下方:

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvavZUO55FfNL5S6Z9Gr39mNOyv0lLy9FD559YXN8QAicCwtFmMRtBLcJJaTcN5AbAib1a5LiazEBWpC0w/640?wx_fmt=jpeg&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

在整个 InLong Audit 审计流中，审计 SDK 嵌套在需要审计的子系统中，在数据流级别进行数据埋点，并将审计结果发送到审计接入层。审计接入层将审计数据写到 MQ ( Kafka 或者 Pulsar) 的专门的审计 Topic 中，审计分发服务（AuditDds）消费 MQ 的审计数据，并将审计数据写到 MySQL/Elasticsearch/ClickHouse。在接口层，提供多个时间粒度将 MySQL/Elasticsearch/ClickHouse 的数据进行封装，向前端页面展示、审计对账等提供对账查询服务。

  

### 关于 Apache 软件基金会

Apache 软件基金会成立于 1999 年，是世界上最大的开源基金会，管理着 2.27 亿多行代码，并 100% 免费向公众提供价值超过 220 亿美元的软件。ASF 的全志愿者社区从负责监督 Apache HTTP 服务器的 21 名原始创始人发展到 820 多个个人成员和 200 个项目管理委员会，他们通过 ASF 的精英管理流程（称为“The Apache Way”）。Apache 软件是几乎所有最终用户计算设备不可或缺的一部分，从笔记本电脑到平板电脑，再到跨企业的移动设备和任务关键型应用程序。Apache 项目为大多数 Internet 提供动力，管理 EB 级数据，执行 teraflops 运算，并在几乎每个行业中存储数十亿个对象。商业友好和许可的 Apache License v2 是一个开源行业标准，帮助启动数十亿美元的公司，并使全球无数用户受益。ASF 是一家美国 501(c)(3) 非营利慈善组织，由个人捐款和企业赞助商资助，包括 Aetna、阿里云计算、亚马逊网络服务、Anonymous、百度、彭博、Capital One、Cloudera、康卡斯特、Confluent、滴滴出行、Facebook、谷歌、华为、IBM、Indeed、LINE Corporation、微软、Namebase、Pineapple Fund、红帽、Replicated、Salesforce、Talend、Target、腾讯、联合投资、VMware、Workday和雅虎。如需更多信息，请访问 http://apache.org/ 和 https://twitter.com/TheASF。

  

### 毕业寄语

“我们很高兴看到 InLong 践行 Apache Way，并以顶级项目的身份从 Apache 孵化器毕业”，腾讯副总裁蒋杰表示：“腾讯致力于构建开源生态系统，让全球开发者能够平等便捷地利用开源代码。Apache InLong 广泛应用于腾讯内部海量业务以及外部的金融、政府等关键业务。未来，腾讯将持续为社区贡献我们的力量，携手全球开发者共同构建繁荣的大数据生态系统。”

“在 Apache 孵化期间，InLong 社区给我留下了非常深刻的印象，社区开发者和用户都在以健康的方式发展。”Apache InLong 导师 JB Onofre 说， “ Apache InLong 是一个很好的例子，说明了如何在 Apache 软件基金会建立一个健康、活跃的社区和成功的项目。”

“恭喜 Apache InLong 作为 ASF TLP 毕业！” NextArch 基金会 TOC 的 Mark Shan 说， “InLong 降低大数据使用门槛，期待 InLong 为社区和企业提供更多元化的开放解决方案”。

“Apache InLong 集成了 Apache Pulsar 的能力，借助 Pulsar 的队列缓冲功能提高了数据处理吞吐量，”StreamNative 联合创始人、Apache Pulsar PMC 成员翟佳说，“我很高兴看到 Apache InLong 的毕业，不同于以往的开源大数据项目，InLong 整合了多个大数据项目的能力，拥有丰富的使用场景。恭喜InLong社区”。

“作为一站式数据集成服务，Apache InLong 完整的周边生态大大降低了用户的使用门槛”，Sangfor 大数据技术专家、Apache InLong PMC 吴中波表示：“InLong 提供的高性能、低成本的消息队列服务正成为企业降本增效的选择。祝愿 InLong 社区未来发展更好更快”。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

Apache InLong 项目官方网站：[**https://inlong.apache.org**](https://inlong.apache.org/)

Apache InLong GitHub 地址：**https://github.com/apache/inlong**

腾讯程序员

，赞1736

阅读 1.1万

​

喜欢此内容的人还喜欢

C++内存问题排查攻略

我常看的号

腾讯技术工程

不喜欢

不看的原因

确定

- 内容低质
- 不看此公众号内容

![](https://mmbiz.qpic.cn/sz_mmbiz_jpg/j3gficicyOvavvGZnomhIYDruIzlO8wuNGfY7ADhgBBdvNl9IS7VP905BncdhblRXCVWvH3IstKRxQiaicDLlAEzDQ/0?wx_fmt=jpeg)

把 Spring Boot 3.3项目从 31.5M 瘦身到 0.82M，部署超级快

路条编程

不喜欢

不看的原因

确定

- 内容低质
- 不看此公众号内容

![](https://mmbiz.qpic.cn/sz_mmbiz_jpg/iciaYydulQQ937aG7vNh4hZibWQpdc8sg28ic7pBpNFIuE1In5fPS8y6TyrYtxOpVh9iaNgv5J6SnLjPKosuoAwice7A/0?wx_fmt=jpeg)

使用存储过程挺香的！为何阿里要禁止？

IT 邦德

不喜欢

不看的原因

确定

- 内容低质
- 不看此公众号内容

![](https://mmbiz.qpic.cn/sz_mmbiz_jpg/m8icwCXib8q5bjShU0sypw5hTS5ocD8PdddZvalzt161x1ibI7BMCT2jiaF425OjxKXic1SGvDOJXJqkpklrcicyogng/0?wx_fmt=jpeg)

写留言

[](javacript:;)

![](http://mmbiz.qpic.cn/sz_mmbiz_png/j3gficicyOvauPPfL7J2AVERiaoMJy9NBIwbJE2ZRJX7FZ2Dx7IibtTwdlqYSqTZTCsXkDS2jvNF8wWJKcibxXtOHng/300?wx_fmt=png&wxfrom=18)

腾讯技术工程

62320

写留言

写留言

**留言**

暂无留言