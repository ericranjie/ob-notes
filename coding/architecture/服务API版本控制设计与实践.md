# 

架构师社区

_2021年11月26日 14:08_

以下文章来源于vivo互联网技术 ，作者Song Jie

\[

![](http://wx.qlogo.cn/mmhead/Q3auHgzwzM5gDg6UckibibLvqjUtNVo6qqOeiaYc07q5N7pm5GRwvRQxg/0)

**vivo互联网技术**.

分享 vivo 互联网技术干货与沙龙活动，推荐最新行业动态与热门会议。

\](https://mp.weixin.qq.com/s?\_\_biz=MzU0OTE4MzYzMw==&mid=2247526848&idx=2&sn=8ca432e57e39c30fbf13ff377778f2b1&chksm=fbb1e03eccc6692831f495e46f2e1973015acd326709d56c28a0a8869d16e524a74b3898d6fe&mpshare=1&scene=24&srcid=1126nTFpPwIqp8kaRgCMD50u&sharer_sharetime=1637916969662&sharer_shareid=5fb9813bfe9ffc983435bfc8d8c5e9ca&key=daf9bdc5abc4e8d06e53633bbeba1aadae6c2ea8ef94b7bce859c9ff6d436850d53fefcaa7d900d08db631c64f2dbca9948685e618b6e08c72dbb28fc2ac3855737fbb9351662f67bdad6caa7ffc086fbe1aedb897e599b48d378875ae8b540ffd7d6990617750d5aaaee11c31fee6f75bb840c26f03946d4205757495c04f91&ascene=0&uin=MTEwNTU1MjgwMw%3D%3D&devicetype=Windows+11+x64&version=63090b19&lang=zh_CN&countrycode=CN&exportkey=n_ChQIAhIQhIIbcrB7OW9%2BhSMmBXI4nRLmAQIE97dBBAEAAAAAAIKeOLS%2FhKwAAAAOpnltbLcz9gKNyK89dVj0tY9iSSjMy6hRGLceQHT9T59UMwjO0So05ixzdIn%2B9I1RcVT5ne%2F2NwN8DnEsI0GAai1kQtCTgNVrPd8lTSUuhZYNMyGKZZdwgSqcua8e8VyEVC2Ez9dOJ9K5ko93qleMpFeZDPpDvWsmcW5Zqb7pQPmnSc8WdqFV%2Bpicmly2U8r6sfc2ZWhcJMKZq3aqQLU4nJroPrj0WFCLyLjn3bmEQimil1rhbFCvpt4ZeU319xDlSFKRhvtwIfvD%2FTZvNy90&acctmode=0&pass_ticket=m4cF6rEM5zxFrth8znDjvKDDNUSzfplmo5V3TKoCeDTooKZxFT4FZvQuJMDcMsbQ&wx_header=1&fasttmpl_type=0&fasttmpl_fullversion=7351805-zh_CN-zip&fasttmpl_flag=1#)

> 作者：vivo互联网服务器团队-Song jie

一、前言

笔者曾负责vivo应用商店服务器开发，有幸见证应用商店从百万日活到几千万日活的发展历程。应用商店客户端经历了大大小小上百个版本迭代后，服务端也在架构上完成了单体到服务集群、微服务升级。

下面主要聊一聊在业务快速发展过程中，产品不断迭代，服务端在兼容不同版本客户端的API遇到的问题的一些经验和心得。一方面让团队内童鞋对已有的一些设计思想有一个更彻底的理解，另一方面也是希望能引起一些遇到类似场景同行的共鸣，提供解决思路。

二、通用解决方案

应用商店客户端迭代非常频繁，发布新的APP版本的时候，势必导致出现多版本，这样服务端就会导致多个不同的客户端请求。强制用户升级APP，可能会导致用户流失，因此采用多版本共存就是必须的。以下是业界讨论过的的一些SOA服务API版本控制方法参考\[1\]。在实际开发中原则上离不开以下三个方案。

> **方案一：The Knot 无版本**——即平台的API永远只有一个版本，所有的用户都必须使用最新的API，任何API的修改都会影响到平台所有的用户。（如下图1）
>
> **方案二：Point-to-Point**——点对点，即平台的API版本自带版本号，用户根据自己的需求选择使用对应的API，需要使用新的API特性，用户必须自己升级。（如下图2）
>
> **方案三：Compatible Versioning**——兼容性版本控制，和The Knot一样，平台只有一个版本，但是最新版本需要兼容以前版本的API行为。（如下图3）

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

（引用自：The Costs of Versioning an API）

简单分析，The Knot只维护最新版本，对服务端而言维护有一定简化了，但是要求服务使用者及时适配最新的版本，这种做法不太适用用户产品，目前内部服务比较适用。Point-Point针对不同客户的版本提供独立的服务，当随着版本的增加开发和运维的维护成本会增加，这种在后面我们面对“协议升级”的时候有使用。

方案三应该是最常用的情况，服务端向后兼容。后面案例也主要采用这种思想，具体的做法也是有很多种，会结合具体的业务场景使用不同策略，这个会是接下来讨论的重点。

三、具体业务场景面临的挑战和探索

3.1 The Knot 无版本和Point-to-Point模式的应用场景

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

上图是我们应用商店迭代变化的一个缩影，业务发展到一定阶段面临以下挑战：

> 1）业务发展前期，作为服务提供方，服务端不仅要支撑多个版本应用商店客户端，同时服务于软件侧的PC助手；
>
> 2）产品形态变化多样，服务端接口变更和维护面临多版本客户端兼容的挑战；
>
> 3）架构逻辑上，服务端采用早期传统架构，开发和维护成本比较高；服务端与客户端进行交互的协议优化升级；以及服务拆分势在必行。

所以服务端协议、框架升级以及公共服务拆分是首要解决的方向。改造经历了两个过程：

- **阶段一**新版本新的接口一律采用新的JSON协议；已有功能接口进行兼容处理，根据客户端版本进行区分，返回不同协议的格式内容。

- **阶段二**随着业务迭代，新的版本商店依赖的所有接口都完成了协议升级后，为了提升服务的稳定性，旧的协议性能无法明显提升，一方面升级后端架构和框架，提升开发效率和可维护性。同时拆分和独立新的工程，实现历史工程只提供给历史版本使用。我们针对大流量高并发、以及基础服务场景比如首页、详情、下载进行独立服务独立拆分。同时也提取一些公共的内部RPC服务，比如获取应用详情、过滤服务等。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

经过改造，服务端架构如上图所示。

1）至此Old-Service后续只用进行相应的维护工作即可，对应Point-to-Point版本。

2）内部的RPC服务由于只提供内部服务，服务端和客户端可以随时同步升级，只要维护最新的版本就可以，采用The Knot模式。这里需要注意的是服务的升级需要注意保持向下兼容，在扩展字段或者修改字段的时候需要特别小心，不然可能在服务升级的时候会引起客户端调用的异常。

3.2 Compatible Versioning：兼容性版本控制

兼容性版本控制应该是最常见的版本控制方式，特别是在C/S架构当中，具体的兼容性版本不同的策略总结有API版本、客户端版本号、功能参数标志等。

**场景一：API版本号控制**

随着互联网发展的，用户体验要求也是越来越高，产品形式也会随之每年有不一样的变化。除了避免审美疲劳外，也是在不断探索如何提升屏效、点击率和转化。就拿应用商店首页列表举例。

应用列表在形态上经历过单一的**应用双排 -> 单排  -> 单排+穿插**的布局。内容上也经历了不同商业化模式、人工排期到算法等演进。

每个版本接口内部逻辑变化是十分大的，有明显差异。如果只是简单在service层根据版本进行判断处理，会导致处理逻辑会变得异常复杂，并且还可能导致对低版本产生影响。同时商店首页是十分重要的业务场景，结合风险考虑，类似这样对场景，在接口URL上新增版本字段，不同对版本使用不同的值，在控制层根据不同的版本进行不同的处理逻辑会更加合理，简单有效。具体策略也有比如在URL上新增接口版本字段/{version}/index、请求头携带版本参数等。

**场景二：客户端版本号控制**

类似首页列表，商店的穿插Banner也经历了多个版本的迭代。如下图所示。这些穿插样式都是在不同版本下出现的，在样式布局，支持跳转能力等方面各个版本的支持程度不一样，接口返回时需要进行相应的处理适配、过滤等处理。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

这类场景如果采用场景一的方案升级新的接口也能够解决，但是会存在大量重复代码，而且新增接口对于客户端接口改造、特别是一些接口路径会影响到大数据埋点统计，也是有比较高的沟通和维护成本在里面。

为了提升代码复用性。使用客户端版本号控制是首选考虑策略。但是需要注意，如果只是简单的在代码层面根据客户端版本号进行判断，会存在以下问题需要考虑：

> 1）代码层面会存在各种判断，造成的代码可读性差，有没有更加优雅的方法；
>
> 2）存在一个客观情况。那就是客户端的版本号是存在不确定性的。由于客户端采用火车发布模式 参考\[2\]，多版本并行开发，导致版本号存在变动、版本跳跃不连续的情况时有发生，也给服务端开发带来了不少困扰。

如何思考解决这些问题呢？其实对于不同的产品形态涉及的一些资源或者产品模块本身出现在不同的迭代周期，可以认为他们具备了版本或者时间的属性。站在程序员视角，把某个资源支持对应的客户端版本作为这个资源对象的一个成员属性。每种资源具有这种属性后，也有相应的逻辑行为来对应成员方法---根据属性进行过滤。这样的设计赋予资源了属性和行为后，资源具备了统一的、灵活的过滤能力，而不再是简单的硬编码根据版本进行if-else判断。

有了方案后，实施起来就比较容易了。开发分配资源ID，并且设置对应支持客户端版本范围。过滤逻辑统一到资源对象。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

代码层面可以将过滤逻辑统一封装到一个工具类（示例代码），在各个业务接口返回进行过滤。更加优雅的方案是建立统一的资源上层类，封装资源过滤方法，所有资源位的资源对象实现该上层类，统一在获取资源逻辑完成过滤能力。

**场景三：新增功能标识参数**

应用商店业务主要提供用户发现和下载新应用、更新手机已安装的应用。商店有增量更新可以减小更新包体积，因此也叫省流量更新，有效提升用户体验。前期我们使用开源的增量算法，但是发现该算法在部分机器合成拆分包会耗时很长，甚至引起crash。

于是项目组寻求更加高效拆分算法。类似在这些已有接口的进行功能增强的场景，除了提供新的API或者内部简单通过客户端版本判断进行扩展外，有没有更好的方案呢？因为除了这些方案已知的弊端外，需要从长远考虑，比如前面提到的算法，后续还会不会存在升级的可能，下载接口会不会有更多能力的增强。

结合上面思考，在原来接口基础上新增**标志参数**字段，表示该请求发出的客户端支持的能力。为了后续扩展，字段类型为整数值，不只是简单的boolean，服务端通过**位运算完成判断逻辑**。客户端也摆脱某个功能与版本的强一致性，不用去记录某个版本具有某种能力。

四、关于接口设计的更多思考

最后补充一些踩过的坑和反思。服务端在提供接口时，不能仅仅关注接口的实现，更多的时候需要关注接口的使用方，他们使用的场景、调用时机等等。否则开发在对接口问题排查、维护花费的时间会比实际开发的耗时要多上好几倍。

> 1）**场景化**：具体到什么是场景化呢，拿商店客户端的帮助用户检测手机安装的应用版本是否最新的服务举例，检测时机是存在不同的场景的，比如用户启动、用户切换wlan环境、定时检测等。当需要进行精细化分析，哪些请求是有效的，哪些会引起集中请求时，这个时候如果请求上没有场景区分，那么分析将无从下手。所以在与客户端沟通接口设计时，请带上场景这个因素。接口设计上可参考如/app/{scene}/upgrade，定义好各个场景名称，在路径上带上具体的场景，这样对线上不同来源请求量级、问题分析都会有很大好处。

> 2）**鉴权和服务隔离**：除了场景需要考虑外，接口调用在分配时做好记录和鉴权以及服务隔离。比如商店的部分接口服务不仅提供给客户端，同时也会提供给手机系统应用调用。目前vivo上亿的存量用户体量，这里要十分小心，系统应用的调用量控制不当，并发可比商店本身要大的多。首先前期与服务调用方评估沟通、做好设计，避免出问题。即使在出问题时，也要有机制能够快速发现问题、能够分析出问题的来源，降低问题带来的损失。

至此上面解决问题的思路，都与具体业务以及背景有一定关系。随着技术不断迭代和发展，在移动端APP页面动态性，目前业界也有了更多高效的技术方案，比如谷歌的Flutter、Weex等。这些技术能够实现灵活扩展，多端统一，性能也能够接近native。不仅减少了客户端发版频次，也减少了服务端兼容性处理成本。目前我们vivo也有团队在使用和实践。

技术不断更迭，没有最好的方案，只有最适合的方案。开发过程中不仅满足当前实现，更多的是考虑到后续扩展性和可维护性。**开发不能一味追求高端技术，技术最终服务于业务，坚持长期主义，效率至上**。

五、参考资料

1、The Costs of Versioning an API

2、敏捷开发，火车发布模式

![](https://res.wx.qq.com/t/fed_upload/b39ef69e-c4d6-4169-8612-5f00a84860e7/wx-avatar-default.svg)

公众号

阅读 1699

​
