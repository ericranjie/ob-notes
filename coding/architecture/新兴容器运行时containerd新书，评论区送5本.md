

开发内功修炼

 _2024年06月05日 08:58_ _北京_

> 《containerd 原理剖析与实战》新书内购中，**限时 69.9 元购买**。文末免费赠书5本。  

## 

大家好，我是飞哥！

  

现在无论是微服务也好，还是大模型也好，其底层基本都是基于Kubernetes 作为编排调度工具的。Kubernetes 底层又是基于各种容器运行时来进行工作，如果能理解容器运行时是如何工作的一定对手头的工作会帮助很大。

  

Kubernetes支持的容器CRI很多。不过官方从2020年之后开始弃用 dockershim，转而推荐使用从 Docker 项目中分离出来的 containerd 容器运行时。

  

但目前在业界介绍 containerd 的资料还很少。我的字节同事赵吉壮为此专门写了一本，填补了这个空白。

  

为了回馈咱们「开发内功修炼」读者，感谢大家长期以来对公众号的关注和支持，所以我和他要了5本送给大家。由于上次发现有机器刷赞的行为。所以这次修改下送书规则，如下：  

1）限定2024-06-01号之前就已经关注了飞哥公众号的读者参加

2）在评论区进行评论

3）2024-06-06日中午12点后我会在评论区挑选出5条优质评论，每条评论赠送一本

  

下面我们开始展开对这书书的介绍。  

  
大模型与云原生

近年来，大语言模型的热度可谓是愈发高涨，尤其是今年年初 Sora 的出现，更是让全球再次看到了AIGC 的巨大威力。

但其实大模型所依赖的基础设施仍然需要云原生和 Kubernetes 的支持。GTC 2024大会上，NVIDIA创始人兼首席执行官黄仁勋，不仅推出了目前全球最强劲的 GPU 芯片 B200。还专门为大规模部署 AI 模型推出了 "英伟达推理微服务 (NVIDIA NIM)"

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/ofDLjhIrfGBiarlTxcJfrUTArMN0QMrFIV9DKV4hRMXp9IWjQjnQ2sniak1ZWWicKhXT30l0XJJo1PATkg4ukIPibQ/640?wx_fmt=jpeg&from=appmsg&wxfrom=13&tp=wxpic)

英伟达 的 AI 推理服务 NIM（NVIDIA Inference Manager） 

NVIDIA NIM 是一个强大的工具，可以简化和优化基于NVIDIA GPU的推理任务的管理和部署。**NIM 的运行依赖 kubernetes 和 containerd 容器运行时**，NIM 微服务通过打包算法、系统和运行时优化，并添加行业标准的 API，能够大幅度简化 AI 模型部署过程**。**

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

基于kubernetes 和 containerd 管理 NVIDIA GPU

  

NIM 通过 kubernetes 来编排 AI 模型部署任务，同时，通过 containerd 集成自家的 nvidia-container-runtime 管理 GPU 设备，实现 GPU 算力的资源池化。

在大模型的背景下，容器运行时的重要性更加明显。相比传统的虚拟化技术，容器启动速度更快，同时共享内核的轻量型虚拟化，大幅减少了资源的开销。这对于大模型的训练和推理来说非常重要，因为它们通常需要快速部署和高效的资源利用。

## 为什么写这本书

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

Kubernetes 作为容器编排领域的事实标准毫无疑问，同时大模型时代的到来也证明了云原生依然是无可撼动的云计算基础设施。随着 2020 年 **Kubernetes 在 v1.20 版本宣布开始弃用 dockershim**，越来越多的 企业在构建 Kubernetes 集群是选择 containerd 作为底层运行时，这使得 containerd 在整个云原生技术栈中的地位日益提升。

CRI 支持的容器运行时有很多，其中 containerd 作为从 Docker 项目中分离出来的项目，由于经历了 Docker 多年生产环境的磨练，相比其他 CRI 运行时也更加健壮、成熟。正因如此，它也是 kubernetes 官方推荐使用的运行时。

Docker 作为老牌的容器运行时，市面上关于它的书籍和资料很多，Kubernetes 的书籍也很多，而 containerd 作为一个新兴的容器运行时，**截止本书出版前，却依然没有一个系统介绍 containerd 的书籍。**

作为一名云原生以及容器技术的忠实粉丝，笔者很早就接触到了 containerd 项目，见证了 containerd 项目的发展，并为之取得的成就感到骄傲。也对 containerd 项目充满了信心。因此希望通过这本书让更多的人了解 containerd，体验 containerd 带来的价值。

## 本书内容

本书从云原生与容器运行时讲起，内容涵盖云原生以及容器的发展史、容器技术的 Linux 原理、containerd 的架构、原理、功能、部署、配置、插件扩展开发等，并详细介绍 containerd生产实践中的配置以及落地实践，使读者对 containerd 的概念、原理、实践有比较清晰的了解。

下面是本书目录：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

## 

该书目前优惠价 69.9，购买入口如下。

  

  
  
大咖推荐

本书的出版也得到了 CNCF、浙江大学计算机系 SEL 实验室、火山引擎边缘云、边缘计算社区、kata containerd 架构委员会等专家的倾力推荐。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

最后，附上本书的购买链接，本书刚刚上架，原价 109，**前期限时优惠内购 69.9 元**，感兴趣的朋友也可以直接下单购买。  

  

最后，想参与评论区送书活动的同学发挥你们的才华，准备开始在评论区留下你的墨宝吧！

阅读 3424

​

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/BBjAFF4hcwqciaicnxpic7g4KKVZKSeQTzic5FoaXsQeLG979y9iaTNVPKTSO9BzSM7zXoJzibbBtEyb6Df5OTj0Yw3Q/300?wx_fmt=png&wxfrom=18)

开发内功修炼

26558

125

**留言 125**

暂无留言