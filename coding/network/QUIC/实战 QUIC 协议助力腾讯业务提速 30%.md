
CPP开发者

 _2022年01月08日 12:05_

以下文章来源于鹅厂架构师 ，作者sTGW-QUIC


![](http://wx.qlogo.cn/mmhead/Q3auHgzwzM6kTFJr72TDAXYicYmJibCVwYyxwcv4V10gMdLVu3ia68l2w/0)

**鹅厂架构师**.

本公众号旨在分享存储、计算、接入等技术沉淀，和各位一起探索业界领先产品技术

](https://mp.weixin.qq.com/s?__biz=MzAxNDI5NzEzNg==&mid=2651169735&idx=1&sn=ee99bf7dc3081bbfb31dfaeacb95c48b&chksm=80647298b713fb8e808568bec2b50872b19e76fe2c0e02b02b869c92dfd773a827ad321d41b2&mpshare=1&scene=24&srcid=0108BiBghxyRKpQE9pl73RFC&sharer_sharetime=1641617994074&sharer_shareid=5fb9813bfe9ffc983435bfc8d8c5e9ca&key=daf9bdc5abc4e8d009f0800a0fcc6063d418c93851647ba2d58ce0c2d214e9df6f745acc34de6e755989dfedc1b59a9df9ef126dbb1d3229efa6dbd9da1cbe7b0000e958a6daab4fea8d8e72280db33770f90a62a78bb1471161389c4974aea75d5adeba316ca80b2b5d71cc9ce770795f0026bea4a8152cb01d9e997e326f80&ascene=0&uin=MTEwNTU1MjgwMw%3D%3D&devicetype=Windows+11+x64&version=63090b19&lang=zh_CN&countrycode=CN&exportkey=n_ChQIAhIQlAlw0oUm1MbfNfkm%2FtXYGBLmAQIE97dBBAEAAAAAAGMVGMFYtb0AAAAOpnltbLcz9gKNyK89dVj0PRNGcSRDXLMvecUNswnfl7sLI0R98wVzAnorG%2BBP5MXaa5mmAbosHeIgZ0tPkW32LT0x7vShsn7VYM4akwf9CfmSX0ij5xwBOzP1qhPNecTQOcwV36uuCMSgBKXH3J7Xf1hdIsknd%2BLQ5P2UIK2AivU5BisxnXWIJJH1xBNUvQ9XfAMDgDgInBBe0kw%2BBAfPUYOTd1E9uqUzFg69mU%2FDyJML3zribe9SKyNGuKejtAQu%2BsW1efe2sU3Fb%2F2zp%2FDw&acctmode=0&pass_ticket=dQwiFzMVnyRuCQxM%2BydVhPo8gA9l5MTwpyQgrmUpfyqiXkJGveT6CQWouDU7W7Ch&wx_header=1&fasttmpl_type=0&fasttmpl_fullversion=7350504-zh_CN-zip&fasttmpl_flag=1#)

之前分享过一篇QUIC在蚂蚁金服落地的文章：

  

[实战|QUIC协议在蚂蚁集团落地](http://mp.weixin.qq.com/s?__biz=MzAxNDI5NzEzNg==&mid=2651166425&idx=2&sn=a3488690f0cd78b5f97bf9473846793b&chksm=80644786b713ce902a2f0a928bf0c8d2d73f49c84fbabb4ac7b47def2f954406304e339022de&scene=21#wechat_redirect)  

  

现在分析一篇，QUIC在腾讯落地的文章，希望大家了解和学习新技术是如何在大厂落地，其中会遇到什么困难，技术难点攻破，以及工程实践。

  

导语

  

腾讯核心业务用户登录耗时降低30%，下载场景500ms内请求成功率从HTTPS的60%提升到90%，腾讯的移动端APP在弱网、跨网场景下取得媲美正常网络的用户体验。

  

这是腾讯网关sTGW团队打造的TQUIC网络协议栈在实时通信、音视频、在线游戏、在线广告等多个腾讯业务落地取得的成果。

  

TQUIC基于下一代互联网传输协议HTTP3/QUIC深度优化，日均请求量级突破千亿次，在腾讯云CLB、CDN开放云客户使用。

  

本文重点分享了sTGW团队在协议栈基础能力、私有协议、明文传输等功能研发经验，并且针对弱网场景，分享腾讯如何基于0-RTT握手、连接迁移、实时传输等能力帮助业务用户体验提升。

  

**一、QUIC/HTTP3协议介绍**

  

QUIC全称quick udp internet connection，“快速UDP互联网连接”，（和英文quick谐音，简称“快”）是由Google提出的基于UDP进行可靠传输的协议。

  

QUIC在应用层实现了丢包恢复、拥塞控制、滑窗机制等保证数据传输的可靠性，同时对传输的数据具备前向安全的加密能力。HTTP3则是IETF(互联网工程任务组)基于QUIC协议基础进行设计的新一代HTTP协议。

  

QUIC/HTTP3分层模型及与HTTP2对比：

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

**二、QUIC核心优势是什么？**

  

1

●

0-RTT建立连接

  

QUIC基于的UDP协议本身无需握手，并且它早于TLS 1.3协议，就实现了自己的0-RTT加密握手。下图分别代表了1-RTT握手（首次建连），成功的0-RTT握手，以及失败回退的握手。

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

2

●

无队头阻塞的多路复用

  

相比于HTTP/2的多路复用，QUIC不会受到队头阻塞的影响，各个流更独立，多路复用的效果也更好。

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

3

●

连接迁移

  

跟TCP用四元组标识一个唯一连接不同，QUIC使用一个64位的ConnectionID来标识连接，基于这个特点，QUIC的使用连接迁移机制，在四元组发生变化时（比如客户端从WIFI切换到蜂窝网络），尝试“保留”先前的连接，从而维持数据传输不中断。

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

4

●

全应用态协议栈

  

QUIC核心逻辑都在用户态，能灵活的修改连接参数、替换拥塞算法、更改传输行为。而TCP核心实现在内核态，改造需要修改内核并且进行系统重启，成本极高。

  

**三、QUIC网络协议栈的选型**

  

虽然QUIC各个特性看上去很美好，但需要客户端/服务端的网络协议栈都支持QUIC协议。截止目前，除iOS 15 在指定接口NSURLSession 及限制条件前提下，支持了HTTP3，其他系统及主流网络库均不支持QUIC。

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

如何让业务快速将QUIC协议用起来，用这些先进特性加速网络性能？

  

QUIC协议栈的实现成本非常高，主要体现在两方面：

  

1. 实现复杂度很高，如上面介绍，QUIC/HTTP3横跨传输层、安全、应用层，相当于要把TCP+TLS+HTTP重新实现一次。
    
2. QUIC一直保持着高速发展，分为gQUIC（Google QUIC）、iQUIC(IETF-QUIC)两大类，衍生的QUIC子版本有几十个。
    

  

为了快速把QUIC协议落地，给业务提升网络性能，我们选择了开源的Chromium cronet网络协议栈作为基础。

  

Chomium，作为占领全球浏览器绝对地位的Chrome的开源代码,有Google强大的研发团队支撑，其网络协议栈是一个相对独立的组件，被称为Cronet。

  

- 协议栈完整性：完善的QUIC协议栈，还包括HTTP2/WEBSOCKET/FTP/SOCKS协议。
    
- QUIC版本支持：支持gQUIC和iQUIC，并且还在不断保持更新。
    
- 跨平台性：非常好，基于chrome的跨平台能力，对于各类操作系统、终端都有适配。
    

  

**四、QUIC协议栈的工程实践**

  

Cronet能直接用起来吗？结合我们的实践与业务同学的反馈，直接使用的问题和接入困难度是比较大的。

  

问题一：代码体积过大，逻辑层级多，不利于集成和安装包体积控制（移动端）

  

Cronet核心及关联的第三方库代码有大概85w行，涉及2800多个类。但其实Cronet里大部分代码都与QUIC没有关系，由于其作为浏览器的网络协议栈，集成了大量浏览器行为逻辑，而这些能力对于网络协议栈是不需要的。

  

其次，QUIC协议只是Cronet里众多通信协议之一，除QUIC外的其他协议，通用的平台或软件（例如Nginx）本身就已经有实现，没有必要重复建设,这些协议的存在除了增加协议栈内部逻辑复杂度，还增大了整个库的体积，例如在安卓平台上，cronet动态库的体积接近3MB，这对于一些体积敏感的应用是一个巨大的挑战。

  

针对体积问题，我们进行了代码精简和lib体积缩减的探究。

  

第一步，分析归纳

  

通过对cronet代码的分析和理解，冗余的代码被我们分成了三种：

  

1. 无用的内部逻辑，例如HTTP模块里包含了很多浏览器才会用到的代码和功能。
    
2. 无需用到的的协议，例如FTP、Websocket等
    
3. 与quic无关的功能模块，例如tcp连接池等
    

  

第二步，代码裁剪

  

针对分析归纳中的三种问题，我们做了针对性的裁剪。

  

首先是精简了关键类，例如协议管控的类中，核心流程步骤被从21步压缩到了5步，函数数量从146个减少到24个，将浏览器相关的冗余逻辑去除。

  

接着对用不到的协议类型、模块组件做了剔除。裁剪后的效果如下：

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

**五、打磨协议栈的易用性**

  

虽然在工程方向，解决了体积大小和编译集成的问题。实际接入使用时，Cronet的易用性依然不够好。

  

通过挖掘业务需求，寻找共性，我们整理出了如下痛点：

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

在通道直连方面，我们将底层udp socket粒度的接口进行封装后直接对外可见，用户可通过socket粒度的接口直接发起IP直连的QUIC请求，同时也保留了DNS建连能力，在保持原生能力的同时，拓展了用户的使用场景。

  

在网络参数配置和性能数据打点能力上，我们深入协议栈细节，逐个分析了多个核心模块，将关键的参数和性能数据抽象出来。并且在控制面上将配置参数、性能打点整合对外呈现。

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

**六、进阶之路：私有协议和明文传输**

  

除了基础功能的打磨，TQUIC协议栈也提供了进阶功能进一步满足业务丰富的使用需求。在Cronet中，要想使用QUIC协议，应用层传输的报文必须是HTTP，也就是所谓的HTTP3协议。

  

但HTTP报文对于游戏、音视频等业务是个巨大的阻碍，它们当前都是通过TCP或者UDP传输自定义的协议的，如果为了接入QUIC而把应用层数据从私有协议强行改为HTTP3，无疑是本末倒置。

  

另外，由于是自定义协议，这些报文一般不需要QUIC进行加密，但加密是QUIC协议的标配，这会消耗额外的性能。

  

为此，在仔细研究了Quic核心代码后，我们研发对私有协议、明文传输的支持，来满足业务传输自定义协议的需求。

  

首先是在QuicStream中，允许stream直接收发数据报文，HTTP流程只是其中一个选择。

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

为了实现明文传输，如果直接去改加解密流程，对代码的入侵较大，如果考虑不周容易引入未知风险。

  

为了尽量较少代码入侵以及维护原生实现的安全运行，我们将QuicFramer中的加解密套件选择处进行了hook，引入了FakeEncrypt/FakeDecrypt替换真实的加解密套件，以极小的入侵代价低成本的实现了明文传输。

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

在做完明文传输方案后，我们意识到由于这是一个非常底层的修改，对于客户端和服务端来需要高度一致的，要么双端都选择加密，要么双端都选择明文。

  

如果双端不统一，则握手就会失败。为了使兼容性更好，减少运维成本和失败风险，我们在握手协商过程中，加入了明文传输的协商。

  

如下图流程，当前的握手过程，使用了AEAD这个tag标识了待协商的加密算法。

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

改进后，AEAD可以携带明文的加密算法，客户端如果也认可，则在下一次CHLO中选择该算法，则之后两边都进行明文传输。

  

**七、弱网优化之连接迁移**

  

随着移动互联网的发展，移动端APP的能力越来越强大，我们可以轻松通过手机进行会议通话、视频直播、在线游戏等，这类应用实时性要求很高。

  

而移动端的特点是网络会经常发生变化，例如通过手机进行在线会议的同时，从办公区走到电梯，随着网络信号衰退到切换，APP需要重新建立连接，这会触发用户重新登录，体验很差。

  

前面背景介绍的QUIC连接迁移可以做到跨网场景自动迁移，业务无任何感知。那是不是使用QUIC后就可以直接用起来呢？

  

我们以Google cronet为例，在iOS下使用Cronet进行了一次切网尝试，通过抓包发现网络切换后，quic其实进行了重新握手，并没有如预期内的进行连接迁移。

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

通过代码分析原因后，我们发现Google其实并没有将连接迁移在各个系统版本上进行支持，仅在安卓平台“打折扣”的支持了，他们是在java层注册系统通知才能使用连接迁移，对于native的网络库以及其他操作系统是很不友好的。

  

既然原生实现并不好用，经过我们的改造，使用跨平台通用的内核层调用，达到了预期的效果，并且不依赖任何特定的操作系统组件。

  

在逐步的实践过程中，我们还发现，如果仅仅是切网时候发起连接迁移并不完美。有时候客户端处在弱网环境，wifi信号并未彻底断开，但传输数据实际上已经有损。

  

为了解决这种场景下的问题，我们建立了一套弱网评估模型，通过主动探测和弱网评估进行启发式分析，在数据通道有损的情况下，尽可能早的主动进行连接迁移，减少对上层业务的影响。

  

通过研发跨平台通用的连接迁移，以及启发式主动迁移能力，最终在客户端看到的连接迁移效果如下，当切网发生时，连接并未断开，无需重连即可续传数据。

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

业务在使用QUIC连接迁移后，对网络切换无感知，数据不中断。

  

或许有同学发现了，这里仅仅讲了连接迁移在客户端上的实现和效果，毕竟连接迁移后，客户端源IP发生了变化，云端接入层难道不需要做适配吗？

  

答案是需要的，sTGW作为公司云CLB和自研的主要七层网关接入，在QUIC连接迁移方面做了完整的解决方案，目前也已经将能力开放出来，在QUIC集群上全量上线。

  

**八、弱网优化之完全0-RTT握手与金融级前向安全**

  

如下是GQUIC的握手流程，原生实现里，应用发生冷启动时，首先会进行1RTT握手，拿到服务端的证书和ServerConfig（简称SCFG），随后SCFG会被作为随机数用于生成非前向安全的密钥。

  

因此在下一次发送数据包时，便可用非前向安全的密钥进行加密。直到收到了服务端发来的Server Hello后，通过类似DHE算法生成前向安全秘钥，自此开始发送前向安全的数据包。

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

在原生实现中，SCFG会被临时存储在进程内存堆区，下次发起新连接或者热启动时，直接就能进入0-RTT流程，但冷启动则永远是1-RTT。

  

基于原生的实现情况，我们针对握手流程做了两个方向的优化。

  

1. 实现100% 0-RTT成功率，用于在握手时延极其敏感的业务上，例如广告请求、API调用、短连接下载等。
    
2. 强制前向安全，针对安全敏感型业务，例如金融业务，0-RTT中发送的非前向安全数据包风险远高于前向安全数据包，因此对于金融型业务，可以强制1-RTT握手生产前向安全密钥后，再发送数据包。
    

改进后的两种握手流程如下图

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

**九、弱网优化之实时传输**

  

实时传输是QUIC的一个拓展功能，目前在IETF草稿阶段。实时传输适用于对数据可靠性要求不高，但非常注重数据实时性的业务。例如音视频传输、互动游戏等。实时传输在QUIC中的定位，以及与可靠传输的区别如下：

  

- 相同点
    

1. 在QUIC连接建立、创建QUIC数据包、数据加解密这些基础功能，不可靠数据与可靠数据都是共用的。
    
2. 不可靠传输也有拥塞控制、ACK机制，与可靠传输一致。
    

- 不同点：
    

1. 不可靠数据不受滑动窗口限制，滑窗窗口满只限制可靠数据传输。
    
2. 发生丢包重传时，只重传可靠数据帧，不可靠数据帧不进行重传。
    

  

不可靠数据没有quic stream概念，只是frame粒度

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

这其中，一个关键点在于数据是否重传，IETF草稿的定义对这块比较开放，可以完全不重传，也可以选择性重传。

  

为此，TQUIC在实现实时传输时，做了灵活的改造，对于实时传输的数据，提供多种重传策略供使用者选择，可以完全不重传，也可以选择性重传某个重要的数据（比如关键帧），我们也在尝试做动态重传控制，依托我们的弱网判断模型，动态调整重传策略。

  

**十、总结**

  

经过腾讯sTGW团队持续投入。TQUIC在腾讯多个重要业务落地，覆盖业务包括实时通信、音视频、在线游戏、在线广告等。在登录成功率、登录耗时、握手时延、下载速度等性能指标上均取得了明显收益。

  

腾讯云客户通过CLB使用HTTP3后，延迟降低了超过20个点，也欢迎大家接入使用。

  

附部分业务效果总结如下：

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

  

**相关链接**

---

**https://datatracker.ietf.org/doc/html/rfc9000**  

**https://datatracker.ietf.org/doc/draft-ietf-quic-datagram/?include_text=1**

**https://docs.google.com/document/d/1g5nIXAIkN_Y-7XJW5K45IblHd_L2f5LTaDUDwvZ5L6g/edit**

**https://dl.acm.org/doi/abs/10.1145/3098822.3098842**

  

- EOF -

推荐阅读  点击标题可跳转

1、[手撕汇编。。。](http://mp.weixin.qq.com/s?__biz=MzAxNDI5NzEzNg==&mid=2651169485&idx=1&sn=06fc2bb3d2c96891d75c7a4c30c41051&chksm=80647392b713fa84742b5ccd29b879786c18416f91391c9028bc5a3dee66d93df02b81cd33e5&scene=21#wechat_redirect)

2、[实战 | QUIC 协议在蚂蚁集团落地](http://mp.weixin.qq.com/s?__biz=MzAxNDI5NzEzNg==&mid=2651166425&idx=2&sn=a3488690f0cd78b5f97bf9473846793b&chksm=80644786b713ce902a2f0a928bf0c8d2d73f49c84fbabb4ac7b47def2f954406304e339022de&scene=21#wechat_redirect)

3、[QUIC 是如何解决TCP 性能瓶颈的？](http://mp.weixin.qq.com/s?__biz=MzAxNDI5NzEzNg==&mid=2651165245&idx=1&sn=c6ca338a102dd880561dddea4d7ca73c&chksm=80644362b713ca7401f524d7866db92cdbf77b5ffaf66a11bf10181fcf0a67599c36bbb307fc&scene=21#wechat_redirect)

  

**关注『CPP开发者』**  

看精选C++技术文章 . 加C++开发者专属圈子

![](http://mmbiz.qpic.cn/mmbiz_png/pldYwMfYJpia3uWic6GbPCC1LgjBWzkBVqYrMfbfT6o9uMDnlLELGNgYDP496LvDfiaAiaOt0cZBlBWw4icAs6OHg8Q/300?wx_fmt=png&wxfrom=19)

**CPP开发者**

我们在 Github 维护着 9000+ star 的C语言/C++开发资源。日常分享 C语言 和 C++ 开发相关技术文章，每篇文章都经过精心筛选，一篇文章讲透一个知识点，让读者读有所获～

24篇原创内容

公众号

点赞和在看就是最大的支持❤️

阅读 2287

​