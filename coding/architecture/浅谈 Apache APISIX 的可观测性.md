

原创 Apache APISIX 高可用架构

 _2021年11月09日 09:40_

可观测性是从系统外部去观察系统内部程序的的运行时状态和资源使用情况。衡量可观测性的主要手段包括：Metrics、Logging 和 Tracing，下图是 Metrics、Logging 和 Tracing 之间的关系。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

举个例子，Tracing 和 Logging 重合的部分代表的是 Tracing 在 request 级别产生的日志，并通过 Tracing ID 将 Tracing 和 Logging 关联起来。对这份日志进行一定的聚合运算之后，能够得到一些 Metrics。Tracing 自身也会产生一些 Metrics，例如调用量之间的关系。

  

## 

**Apache APISIX 的可观测性能力**

Apache APISIX 拥有完善的可观测性能力：支持 Tracing 和 Metrics、拥有丰富的 Logging 插件生态、支持查询节点状态。

  

### 

**Tracing**  

Apache APISIX 支持多种 Tracing 插件，包括：Zipkin、OpenTracing 和 SkyWalking。需要注意是：Tracing 插件默认处于关闭状态，使用前需要手动开启 Tracing 插件；Tracing 插件需要与路由或全局规则绑定，如果没有采样率的要求，建议与全局规则绑定，这样可以避免遗漏。

  

### 

**Metrics**

在 Apache APISIX 中， Metrics 的相关信息通过 Prometheus Exporter上报，兼容 Prometheus 的数据格式。在 Apache APISIX 中使用 Prometheus Plugin 有两件事情需要注意。

  

**第一，请尽量提高路由、服务和上游这三者名称的可读性。**

Prometheus Plugin 中有一个名为 `prefer_name`的参数，将这个参数的值设置为`true`时，即：`prefer_name: true`，如果路由、服务和上游这三者的名称可读性比较强，这会带来一些好处：后续通过 Grafana 监控大屏展示参数的时候，不仅能够清楚地展示出所有的数据，还能够清晰地知晓这些数据的来源。如果`prefer_name`参数的值为`false`，则只会展示资源的 ID 作为数据来源，例如 路由 ID、上游 ID，进而造成监控大屏的可读性较低的问题。

**第二，Prometheus Plugin 必须与路由或者全局规则绑定，然后才可以查看到指定资源的 Metrics。**

完成上述设置以后，Metrics 的数据会存储在 Prometheus 里面。由于 Prometheus 的存储性能很好，但展示性能欠佳，所以我们需要借助 Grafana Dashboard 展示数据。我们可以看到 Nginx 实例的 Metrics、网络带宽的 Metrics、路由和上游的 Metrics 等，详情如下图所示：

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

### 

**Logging**

Apache APISIX 支持多种日志插件，可以与其他外部的平台直接分享日志数据。Error Log 插件支持 HTTP 与 TCP 协议，并且兼容 SkyWalking 的日志格式。也可以通过 FluentBit 等日志收集组件，将日志同步到日志平台进行处理。

Access Log 插件目前还不支持在日志格式里面进行嵌套。因为 Access Log 插件是路由级别的，所以需要跟路由进行绑定，才可以收集到路由的访问日志。但是日志的格式是全局的，而全局只能有一份日志格式。

  

### 

**支持查询节点状态**

Apache APISIX 的支持查询节点状态，启用之后，可以通过 `/apisix/status`收集到节点的信息，包括节点数、等待链接数、处理连接数等。

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

### 

**美中不足**

上文讲到，Apache APISIX 的可观测性能力非常完善，能够收集 Metrics、Logging 和 Tracing 等信息。虽然借助 Apache APISIX 的内置插件配合 Grafana Dashboard，能够解决监控数据收集和指标可视化问题，但是各种数据分散在各个平台。期望有一个可观测性分析平台能集成 Metrics、Logging、Tracing 信息，能够将所有数据联动起来。

  

## 

**使用 Apache SkyWalking 增强 Apache APISIX 的观测能力**

Apache SkyWalking 是一个针对分布式系统的应用性能监控（APM）和可观测性分析平台。它提供了多维度应用性能分析手段，从分布式拓扑图到应用性能指标、Trace、日志的关联分析与告警。

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

### 

**一站式数据处理**

Apache SkyWalking 支持对接 Metrics、Logging、Tracing 等多种监控数据，兼容 Prometheus 数据模型，还可以通过 Log Analysis Language 进行二次聚合，产生新的 Metrics。

### 

  

**更详细的数据展示**

Apache SkyWalking 的 Dashboard 分为上下两个区域。上部是功能选择区域，下部是面板内容。Dashboard 提供全局、服务、示例、Endpoint 等多个实体维度的 Metrics 相关信息，支持以不同的视图展示可观测性。以全局视图为例，展示的 Metrics 包括：服务负载、慢服务数量、不健康的服务数量等，如下图所示。

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

另外值得一说的是 SkyWalking Dashboard 的 Trace 视图。SkyWalking 提供了 3 种展现形式：列表、树状图和表格。Trace 视图是分布式追踪的典型视图，这些视图允许用户从不同角度查看追踪数据，特别是 Span 间的耗时关系。

SkyWalking Dashboard 也支持拓扑图。拓扑图是根据探针上行数据分析出的整体拓扑结构。拓扑图支持点击展现和下钻单个服务的性能统计、Tracing、告警，也可以点击拓扑图中的关系线，展示服务之间、服务示例间的性能 Metrics。

  

### 

**支持容器化部署**

Kubernetes 是一个开源的云原生容器化集群管理平台，目标是让部署容器化的应用简单且高效。Apache SkyWalking 后台可以部署在 Kubernetes 之中，而且得益于 Kubernetes 的高效率管理，可以保证 UI 组件的高可用性。

如果在集群上部署了 Apache APISIX，Apache SkyWalking 支持以 sidecar 或服务发现的形式部署 SkyWalking Satellite，监控集群中的 Apache APISIX。

## 

  

**未来计划**

Apache APISIX 在未来仍会继续加强可观测性相关的功能支持，例如：

1. 解决 SkyWalking Nginx-Lua 插件的 peer 缺失问题 
    
2. 支持在日志中打印 trace id 
    
3. 接入访问日志 
    
4. 支持网关元数据 
    

  

## 

**结语**

本文介绍了 Apache APISIX 的可观测性能力以及如何通过 Apache SkyWalking 提升Apache APISIX 的可观测性能力。未来两个社区还会继续深度合作，进一步增强 Apache APISIX 的可观测性能力。希望大家能够多多地参与到 Apache APISIX 和 Apache SkyWalking 项目中来。如果你对这两个开源项目很感兴趣，却不熟悉代码，写文章、做视频、对外分享、积极参与社区和邮件列表讨论都是很不错方式。

  

## 

**关于 Apache APISIX**

Apache APISIX 是一个动态、实时、高性能的开源 API 网关，提供负载均衡、动态上游、灰度发布、服务熔断、身份认证、可观测性等丰富的流量管理功能。Apache APISIX 可以帮忙企业快速、安全的处理 API 和微服务流量，包括网关、Kubernetes Ingress 和服务网格等。

  

**Apache APISIX 落地用户（仅部分）**

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

- Apache APISIX GitHub：https://github.com/apache/apisix
    
- Apache APISIX 官网：https://apisix.apache.org/
    
- Apache APISIX 文档：https://apisix.apache.org/zh/docs/apisix/getting-started
    

  

**参考阅读：**

  

- [百度爱番番数据分析体系的架构与实践](http://mp.weixin.qq.com/s?__biz=MzAwMDU1MTE1OQ==&mid=2653557555&idx=1&sn=adb40ef8a88d3dae1cb7d674dbf9a569&chksm=813982abb64e0bbdebab9a1a01172d752a1125e4aa8f56c43e0cd0f2f113db049fd535c148c0&scene=21#wechat_redirect)
    
- [云原生环境下对“多活”架构的思考](http://mp.weixin.qq.com/s?__biz=MzAwMDU1MTE1OQ==&mid=2653557529&idx=1&sn=14da4880327117208802868fea5e687c&chksm=81398281b64e0b971e390a98ee3db92f4cea408eb96c10b7d60ef22d059911d097c75403667a&scene=21#wechat_redirect)
    
- [从C++转向Rust需要注意哪些问题？](http://mp.weixin.qq.com/s?__biz=MzAwMDU1MTE1OQ==&mid=2653557429&idx=1&sn=d75c76d9b4c3740a71a8c6eb6d3a5565&chksm=8139832db64e0a3b2de34b45702110ae8d5279be71a7b67a44524ff2de210272f03cc82f0694&scene=21#wechat_redirect)
    
- [高并发场景下JVM调优实践之路](http://mp.weixin.qq.com/s?__biz=MzAwMDU1MTE1OQ==&mid=2653557460&idx=1&sn=202bc4f21fe199b1708782dd01b1eb8b&chksm=8139834cb64e0a5a0642d53259eefcc6ebfd6531f2d1ce4a71103bf35a44e8a8fdefaeaaf46c&scene=21#wechat_redirect)  
    
- [Apache APISIX 扩展指南](http://mp.weixin.qq.com/s?__biz=MzAwMDU1MTE1OQ==&mid=2653557416&idx=1&sn=2c9557cd5839c2a07bdf2b861238ad07&chksm=81398330b64e0a262f3930c4cff66314ba528ad4b20e6b14226e309827bfaef3e3c23533e96d&scene=21#wechat_redirect)  
    

  

技术原创及架构实践文章，欢迎通过公众号菜单「联系我们」进行投稿。  

  

**高可用架构**

**改变互联网的构建方式**

![](http://mmbiz.qpic.cn/mmbiz_png/8XkvNnTiapONl06YmHad4csRU93kcbJ76JIWzEAmOSVooibFHHkzfWzzkc7dpU4H06Wp9F6Z687vIghdawxvl47A/300?wx_fmt=png&wxfrom=19)

**高可用架构**

高可用架构公众号。

439篇原创内容

公众号

  

阅读 3630

​

写留言

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/8XkvNnTiapONl06YmHad4csRU93kcbJ76JIWzEAmOSVooibFHHkzfWzzkc7dpU4H06Wp9F6Z687vIghdawxvl47A/300?wx_fmt=png&wxfrom=18)

高可用架构

9110

写留言

写留言

**留言**

暂无留言