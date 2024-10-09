# 

Original 章亦春 QCon全球软件开发大会

_2016年08月26日 20:16_

Nginx 是一款高性能的开源网络服务器和负载均衡器，而 Lua 是一种小巧而高效的动态语言，当 Lua 的实现整合进了 Nginx 服务器，催生出了 ngx_lua 这个 Nginx 扩展模块，显著提高了 Nginx 配置上的灵活性，大大扩展了 Nginx 的应用范围，甚至有生产用户基于 Nginx 本身构建比较完整的 Web 应用。ngx_lua 模块已然成为 Nginx 世界最流行的模块之一。

在 QCon 北京 2015 的演讲中，CloudFlare Inc. 系统工程师章亦春，向我们介绍了：

1. 当初为什么选择 Lua，以及具体是如何把 Lua 实现整合进 Nginx 的；

1. 如何利用 Lua 协程，避免非阻塞 IO 编程中编写大量嵌套回调函数的痛苦；

1. LuaJIT 的实现架构，以及如何充分利用它的性能优势；

1. 根据这几年的实践总结出：那些在高性能 Web 服务器和 Web 应用中常见的性能瓶颈和应对方法；

1. Nginx + Lua 在 CloudFlare、Adobe、CNN、淘宝网、网易、去哪儿网等国内外公司的基本应用模式。

最后，展示了一系列基于 SystemTap 的高级在线调试技术和工具。以及如何在不停在线服务的情况下，按需检查整个软件栈内部的行为细节，实时在线追踪 Lua 解释器/编译器和 Nginx 服务器中的性能瓶颈及其他问题。并讨论了那些针对 Lua 和 Nginx 的基于 GDB 的高级调试工具的设计与实现。

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

作者介绍

章亦春，CloudFlare Inc. 系统工程师。以写程序为生，喜欢摆弄各种 UNIX 风格的工具，以及不同的编程语言，例如 C/C++、Lua、Perl、Python、Haskell 等。

目前供职于美国的 CloudFlare 公司，使用自己的技术构建 CDN 需要的软件平台。工作中以使用有趣的工具和技术追踪 bug 和性能瓶颈为乐，所以近年来特别着迷于 dtrace 风格的动态追踪技术，同时不得不始终学习整个软件栈，包括操作系统内核在内的各种技术细节。执着于高性能的 Web 服务，从而不得不写很多的 C 代码乃至机器代码。

多年来一直游走在开源世界中，只选择拥抱和支持开源的国内外雇主。创建并一直维护着 OpenResty 这个开源项目，每天花很多时间处理和回复开源社区的用户邮件。经常向 Nginx 贡献补丁，和 Nginx 官方团队保持着良好的沟通和协作关系。特别喜欢参加开源界的技术聚会并作分享。

幻灯片

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

在公众号后台回复“ngx”，即可下载完整幻灯片。

______________________________________________________________________

****▽****

**延展阅读**（点击标题）：

[支撑饿了么业务十倍提升的技术运维体系](http://mp.weixin.qq.com/s?__biz=MzIwMjE5MDU4OA==&mid=2653120121&idx=1&sn=86050a6b1f0a42c9d2cafe62a3442570&scene=21#wechat_redirect)

[SRE是什么鬼？——来自Google DevOps经验的落地实践](http://mp.weixin.qq.com/s?__biz=MzIwMjE5MDU4OA==&mid=2653120111&idx=1&sn=5deda28383d15cd3b4c3d665ec2f8a59&scene=21#wechat_redirect)

[滴滴运维架构的演化史](http://mp.weixin.qq.com/s?__biz=MzIwMjE5MDU4OA==&mid=2653119558&idx=1&sn=f9979a91eb0625d855871ac91c7c1833&scene=21#wechat_redirect)

## [看豆瓣如何加固「监控」这条防线](http://mp.weixin.qq.com/s?__biz=MzIwMjE5MDU4OA==&mid=2653119804&idx=1&sn=0164a8479ba03ef60e6a81c87ce7ddbe&scene=21#wechat_redirect)

[从「运维系统」进化到「企业管控系统」总共分几步？](http://mp.weixin.qq.com/s?__biz=MzIwMjE5MDU4OA==&mid=2653120012&idx=1&sn=c3dbb8a188b48d12b5739a3a5c2355c3&scene=21#wechat_redirect)

______________________________________________________________________

QCon 上海 2016 将于 10 月 20~22 日在上海宝华万豪酒店举行，届时将有一大波技术专家带来精彩演讲。

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

黄浩 Twitter 高级工程师

黄浩目前主要负责 Observability 团队的监控报警系统。之前曾就职于百度和小米。

《Twitter 的监控系统是如何处理十亿量级 metrics 的——Twitter Observability 栈架构实践》

> Twitter 的 Observability 栈包含了核心的 Timeseries Database，实时的监控报表系统，报警和自动故障恢复系统，以及分布式的日志分析和 tracing 系统。在 Twitter，它是整个公司最关键的内部架构之一，是保证各个服务可用性的关键。目前整个监控报警系统每分钟处理 25 亿次的 metrics 写入，170 万的复杂查询和 25 000 的报警规则。日志分析系统和 tracing 系统是工程师们平时追查问题的主要平台。在本演讲中，黄浩将向大家分享整个架构的设计，以及演进中的思考和经验。

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

马小鹏 阿里妈妈全景业务监控平台技术负责人

马小鹏于 2013 年加入阿里巴巴，一直从事大规模系统日志分析及应用的研发。主导了直通车广告主报表平台、直通车实时报表存储选型、阿里妈妈全景监控平台的设计和研发。在数据应用建设方面保持持续的探索和思考，对大数据典型应用场景、统计算法模型的应用、时间序列的分析监测有丰富的经验。

《全景业务监控平台（Goldeneye）》

> 全景业务监控平台（Goldeneye）是阿里妈妈在业务监控方向上的一次大数据应用创新, 相比传统的同环比报警检测方式精确度更高。
>
> 本次演讲向大家介绍一种基于数据统计分析的业务监控检测方法，通过收集监测数据的样本，并使用智能检测算法模型，让程序自动对监控项指标的基准值、阈值做预测，在检测判断异常报警时使用规则组合和均值漂移算法，能精确地判断需要报警的异常点和变点。因为传统的同环比对比比较单调，在工作日和节假日对差异下存在大量的误报、漏报，在监测指标波动时不能有效地过滤掉不值得关注的疑似异常，大量的误报会淹没真正的异常报警。
>
> 我们从预测样本的选取、监控项报警检测灵敏度区分、异常持续状态次数、均值漂移过程等方面做了智能检测程序，可以避免人工维护的惰性和不可持续性带来的隐患。
>
> 在故障辅助定位方面，我们通过建立全链路 tracing、上下游数据关联依赖、数据粒度逐层细分、诊断树模型等方式，缩小排查定位问题的范围，直接通过数据分析提供可参考的定位信息，在实际应用中可以降低故障带来的损失。

LinkedIn Kafka 组高级软件工程师秦江杰，苏宁云商 IT 总部执行总裁助理乔新亮，声网 CEO 赵斌，阿里巴巴无线技术专家隐风，华为软件云平台资深架构师苗彩霞，LinkedIn 业务分析经理赵晟等技术专家都将在 QCon 上海 2016 做分享，\*\*更多信息，\*\***可点击“阅读原文”，访问大会网站。现在报名，可享 9 折优惠。**

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

Read more

Reads 2028

​

Comment

**留言 2**

- tivan

  2016年8月26日

  Like1

  业界良心之作。

- Thetj

  2017年8月13日

  Like

  春哥 感动

已无更多数据

[](javacript:;)

![](http://mmbiz.qpic.cn/sz_mmbiz_png/LVQGLiaeqSouMLRxYuObge0E6ibnDRRhNfQI7ZRf2voDn5spSlyBAN8csZRyLpLNfeU35Nm97Sz8FCJdyDqrJDMQ/300?wx_fmt=png&wxfrom=18)

QCon全球软件开发大会

Follow

5ShareWow

2

Comment

**留言 2**

- tivan

  2016年8月26日

  Like1

  业界良心之作。

- Thetj

  2017年8月13日

  Like

  春哥 感动

已无更多数据
