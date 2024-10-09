# 

原创 字节跳动STE团队 字节跳动SYS Tech

_2023年09月15日 17:25_ _北京_

![](http://mmbiz.qpic.cn/mmbiz_png/Z4txjviceemNaH5Gc7V2ltc9AB79kQxpM7pOjtTiagt1TlQtCTrJ6zyPma8xbw3sGpeecTwveKP4GLqQa2nJ49EA/300?wx_fmt=png&wxfrom=19)

**字节跳动SYS Tech**

聚焦系统技术领域，分享前沿技术动态、技术创新与实践、行业技术热点分析。

42篇原创内容

公众号

近日，由中国科学院计算技术研究所、字节跳动、中国科学院计算机网络信息中心和 Intel 共同撰写的关于 Open vSwitch 转发性能优化的论文被系统领域顶级国际会议 EuroSys 2024 正式接受。EuroSys 由 ACM SIGOPS 组织于 2005 年发起并承办，是目前欧洲系统领域最好的国际学术会议之一，也是目前国际系统领域的重要会议，本次EuroSys分为春季与秋季两个投稿周期，春季周期共有244篇论文投稿，39篇被接收，接收率为16.0%。

![图片](https://mmbiz.qpic.cn/mmbiz_svg/hqDXUD6csU8f9Z5wkbLZ7GBRFq8v9CzicWzGibLtwhqwicbAhQ6EPIucJwp9CMM0oIriaEz8U00ul5EZFc9axAqeibbzmvVTcGtC6/640?wx_fmt=svg&wxfrom=13)

**论文信息**

Heng Pan, Peng He, Zhenyu Li, Pan Zhang, Junjie Wan, Yuhao Zhou, XiongChun Duan, Yu Zhang, Gaogang Xie. **Hoda: a High-performance Open vSwitch Dataplane with Multiple Specialized Data Paths**. Proceedings of the 19th European Conference on Computer Systems, Athens, Greece, April, 2024.

云网络通常采用 Open vSwitch (OvS) 构建底层灵活可编程数据面，但在复杂云场景下（如隧道、带状态防火墙等）其转发性能会大幅下降。通过细粒度的性能诊断与分析，发现性能下降的根本原因主要是在于不同场景的转发任务的差异性与当前 OvS 数据面一刀切的通用设计之间存在巨大鸿沟。为此，设计了一种面向 OvS 的数据面定制化框架——Hoda，Hoda 相比于最新的 OvS **性能提升高达 1.7 倍**，端到端典型的 Nginx 服务**请求处理时间可降低 20%**。

**热门招聘**

字节跳动STE团队诚邀您的加入！团队长期招聘，**北京、上海、深圳、杭州、US、UK** 均设岗位，以下为近期的招聘职位信息，有意向者可直接扫描海报二维码投递简历，期待与你早日相遇，在字节共赴星辰大海！若有问题可咨询小助手微信：sys_tech，岗位多多，快来砸简历吧！

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/Z4txjviceemOHrhacvXFictibXibRT9aUzzTiavs7Jq1TpdoYr4MLaov3USTavWT9UwEZcJoRsu7a8Fs5v171TMBuVw/640?wx_fmt=jpeg&wxfrom=13&wx_lazy=1&wx_co=1&tp=wxpic)

**往期精彩**

# [深入分析C++全局变量初始化](http://mp.weixin.qq.com/s?__biz=Mzg3Mjg2NjU4NA==&mid=2247484664&idx=1&sn=db677b4cfdc597acafd9b7023188975b&chksm=cee9f28ff99e7b997077d1e83ec47313e38eb488a57fb29957d64eb93cc935143d21b2e8eb48&scene=21#wechat_redirect)

# [BMC部件管理：AF_MCTP应用落地实践](http://mp.weixin.qq.com/s?__biz=Mzg3Mjg2NjU4NA==&mid=2247484591&idx=1&sn=996effa848ced022aa2245051cef9fac&chksm=cee9f2d8f99e7bce10cc6a9f38333ae751e5a64a1d02a4a97adee7baeb09b68457f4706e0fd1&scene=21#wechat_redirect)

[ByteFUSE分布式文件系统的演进与落地](http://mp.weixin.qq.com/s?__biz=Mzg3Mjg2NjU4NA==&mid=2247484577&idx=1&sn=acf23f871cb1805b8fbdecfd2ddb7ab9&chksm=cee9f2d6f99e7bc08a0bd4051ee28946343c1e747351b05eb852a65e54e1bbc7501374ce60be&scene=21#wechat_redirect)

# [深入分析HWASAN检测内存错误原理](http://mp.weixin.qq.com/s?__biz=Mzg3Mjg2NjU4NA==&mid=2247484531&idx=1&sn=57acfda58d6cc0dffb9fa5e5c6f488bb&chksm=cee9f204f99e7b12dbd8a84aea2c1299fb28297aea4ecc76af0dfcecbf4294e07d1c8c756627&scene=21#wechat_redirect)

# [深度解析C/C++之Strict Aliasing Rule](http://mp.weixin.qq.com/s?__biz=Mzg3Mjg2NjU4NA==&mid=2247484517&idx=1&sn=049fb8f6743e4caf6f9138688cd841a1&chksm=cee9f212f99e7b043523bfe80d8cb6ea1a86a5c6d2a2a14a0aa5944fbb1e116190d594ad8156&scene=21#wechat_redirect)

[深入浅出 Sanitizer Interceptor 机制](http://mp.weixin.qq.com/s?__biz=Mzg3Mjg2NjU4NA==&mid=2247483868&idx=1&sn=a85112e88cd187e27f418ff044247956&chksm=cee9f7abf99e7ebdac55e36077b4f915bccbe33a8a6ee9920136a1dc9598e2ae62bb95d3a25f&scene=21#wechat_redirect)

[Jemalloc内存分配与优化实践](http://mp.weixin.qq.com/s?__biz=Mzg3Mjg2NjU4NA==&mid=2247484507&idx=1&sn=0befa2bec46c3fe64807f5b79b2940d5&chksm=cee9f22cf99e7b3a6235b06cb0bd87c3f8e9b10c3ba989b253a97a5dc92b261cfde0b39db230&scene=21#wechat_redirect)

[DPDK内存碎片优化，性能最高提升30+倍!](http://mp.weixin.qq.com/s?__biz=Mzg3Mjg2NjU4NA==&mid=2247484462&idx=1&sn=406c59905ea57718018c602cba66f40e&chksm=cee9f259f99e7b4f7597eb6754367214a28f120c062c72c827f6e8c4225f9e54e4c65d4395d8&scene=21#wechat_redirect)

[万字长文，GPU Render Engine 详细介绍](http://mp.weixin.qq.com/s?__biz=Mzg3Mjg2NjU4NA==&mid=2247484413&idx=1&sn=d8f5f4f195b931d775b7d5934f0e3cb4&chksm=cee9f58af99e7c9c07080f22165120fcf3d321a14b943077c23649f88b27da29529db44de933&scene=21#wechat_redirect)

[DPDK Graph Pipeline框架简介与实现原理](http://mp.weixin.qq.com/s?__biz=Mzg3Mjg2NjU4NA==&mid=2247483974&idx=1&sn=25a198745ee4bec9755fe1f64f500dd8&chksm=cee9f431f99e7d27527c7a74c0fd2969c00cf904cd5a25d09dd9e2ba1adb1e2f1a3ced267940&scene=21#wechat_redirect)

[字节跳动算力监控系统的落地与实践](http://mp.weixin.qq.com/s?__biz=Mzg3Mjg2NjU4NA==&mid=2247484264&idx=1&sn=c27c08c225fcceb65a5ffc071dba3f95&chksm=cee9f51ff99e7c09a0c103bc4b362fcdcba7d8e6de74ab251afe1beb5b7af0412e619dd53f13&scene=21#wechat_redirect)

[JVM 内存使用大起底](http://mp.weixin.qq.com/s?__biz=Mzg3Mjg2NjU4NA==&mid=2247484211&idx=1&sn=c65b23b2ab50f6d56d81cfe434c8ea53&chksm=cee9f544f99e7c52d7767db2a74fda23ed7a6c06f8c7a82c1219a7e1474839bf5301f78ff38e&scene=21#wechat_redirect)

[字节跳动在PGO反馈优化技术上的探索与实践](http://mp.weixin.qq.com/s?__biz=Mzg3Mjg2NjU4NA==&mid=2247483904&idx=1&sn=335efeed20a6b42d5ee6e6543ae12018&chksm=cee9f477f99e7d6141de97d85c125979daaae14d98e73225792fa8e41cf0d6269e97d5c7a397&scene=21#wechat_redirect)

[内核优化之 PSI 篇:快速发现性能瓶颈，提高资源利用率](http://mp.weixin.qq.com/s?__biz=Mzg3Mjg2NjU4NA==&mid=2247483887&idx=1&sn=32ec7c9200a72f13ef2c1b8756ea2b8f&chksm=cee9f798f99e7e8e6e44a4f7464ee18471cd4e12def97fdf207f58d37dd317e3fafa6614e04f&scene=21#wechat_redirect)

[探索Mac mini M1的QEMU/KVM虚拟化实现](http://mp.weixin.qq.com/s?__biz=Mzg3Mjg2NjU4NA==&mid=2247483881&idx=1&sn=282483df28129da2d193239743397260&chksm=cee9f79ef99e7e884ca202597076df9915145a6c36670ae3bab2419a0da13deed072ee59e058&scene=21#wechat_redirect)

[字节跳动提出KVM内核热升级方案，效率提升5.25倍！](http://mp.weixin.qq.com/s?__biz=Mzg3Mjg2NjU4NA==&mid=2247483751&idx=1&sn=6f818349a0d6579902d9a7ca1a512cca&chksm=cee9f710f99e7e06957baac9ce25433ed8a463a417479f243136cef254073f10e018d941be4b&scene=21#wechat_redirect)

[安全容器文件服务 Virtio-fs 的高可用实践](http://mp.weixin.qq.com/s?__biz=Mzg3Mjg2NjU4NA==&mid=2247483815&idx=1&sn=92edbb33cbbbf22c80ccdf6537a2ca37&chksm=cee9f7d0f99e7ec671106416c7d5cee58497af8f427a89f5e9a2ce051277944116db069d10e3&scene=21#wechat_redirect)

[字节跳动开源高性能C++ JSON库sonic-cpp](http://mp.weixin.qq.com/s?__biz=Mzg3Mjg2NjU4NA==&mid=2247483853&idx=1&sn=b88e45bf75cd57482cf7860e669416a8&chksm=cee9f7baf99e7eacb5de6f9118a590db4bb659420bc2f4cb6c6964225409f559715ea8145186&scene=21#wechat_redirect)

[eBPF 技术实践：加速容器网络转发，耗时降低60%+](http://mp.weixin.qq.com/s?__biz=Mzg3Mjg2NjU4NA==&mid=2247483701&idx=1&sn=c99b7df5b8457f6d4f87fa18fea82eba&chksm=cee9f742f99e7e54f317d4f9eb355daf8cdceb4d3894e7601f5fc124e8b3b8567ee5f2efc46a&scene=21#wechat_redirect)

[十年磨一剑，veLinux深度解读](http://mp.weixin.qq.com/s?__biz=Mzg3Mjg2NjU4NA==&mid=2247483680&idx=1&sn=d767d17e8b28baacb25f759c02e7bd0c&chksm=cee9f757f99e7e4136580232d4c56d619feacdd939e84ed9780d010b73417c5739b223a5c4dd&scene=21#wechat_redirect)

**关于STE团队**

**字节跳动STE团队**（System Technologies&Engineering，系统技术与工程），一直致力于操作系统内核与虚拟化、系统基础软件与基础库的构建和性能优化、超大规模数据中心的系统稳定性和可靠性建设、新硬件与软件的协同设计等基础技术领域的研发与工程化落地，具备全面的基础软件工程能力，为字节上层业务保驾护航。同时，团队积极关注社区技术动向，拥抱开源和标准，欢迎更多同学加入我们，一起交流学习。扫描下方二维码了解职位详情，欢迎大家投递简历至huangxuechun.hr@bytedance.com 、wangan.hr@bytedance.com。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

扫码查看职位详情

技术创新成果9

技术创新成果 · 目录

上一篇DPDK内存碎片优化，性能最高提升30+倍!

阅读 3298

​

喜欢此内容的人还喜欢

发消息

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/Z4txjviceemNaH5Gc7V2ltc9AB79kQxpM7pOjtTiagt1TlQtCTrJ6zyPma8xbw3sGpeecTwveKP4GLqQa2nJ49EA/300?wx_fmt=png&wxfrom=18)

字节跳动SYS Tech

15213

发消息
