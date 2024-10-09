# 

焉知汽车

_2022年03月19日 22:09_

以下文章来源于十一号组织 ，作者18号线不到安研路

\[

![](http://wx.qlogo.cn/mmhead/Q3auHgzwzM7WQOyvZH6ESBHnjIWrLwSFz91QRH5bvUcsdAYPEWWm1g/0)

**十一号组织**.

记录生而为人的证据，聚焦智能网联与人情冷暖。

\](https://mp.weixin.qq.com/s?\_\_biz=MzU5ODQ5ODQ1Mw==&mid=2247573792&idx=2&sn=c90ee609e6377a7641c1c916231aa6dc&chksm=fe40acdac93725cc7a6c96291c492af55c6bca18f2e2646bd8fff1006c831117266a428cbe0f&mpshare=1&scene=24&srcid=0320saP8qCi2Kh4T08PDZ16c&sharer_sharetime=1647757664397&sharer_shareid=5fb9813bfe9ffc983435bfc8d8c5e9ca&key=daf9bdc5abc4e8d01f165ef694c29c63ddd49b72785f2cd97b140fb060442a4276c70983fcd8ea0956af42ac584721c1120bc8132afd814b06501e03cf5a68644a46f61bbd6a87e1f3574f7961323110e09b31549d38ebb885821863781b83731ce2013d25c6b94692b05ebb23c9ccf68592cfd6603627f1a81a0c5956cab9e6&ascene=0&uin=MTEwNTU1MjgwMw%3D%3D&devicetype=Windows+11+x64&version=63090b19&lang=zh_CN&countrycode=CN&exportkey=n_ChQIAhIQkdVkWOiM5Rr3%2BU1N2j8fBBLmAQIE97dBBAEAAAAAAAbnN%2Fn6C9oAAAAOpnltbLcz9gKNyK89dVj0Cqc%2FugNvZPmbOsqYenty%2Bep2vsmIZE%2BNMcNjcqrl5bththmcfCn7%2Bgkhjmwc9Xof%2FkL3l9qk68CEGcvDIu%2FssZ5bQi8JfJ1KjS5Ee9b0Ql8bflqa1RkC6xlPB6RajJpD51r36koclp6Q4rsP9Eud6VVyXIAZWLcqUbEPoN0%2B7%2F%2FPxFSdt5xTb0C1VnIWGTpKF4Z2aMCzOVTSxBE66%2FPayuTywKGCx869nVSog8cI9pFpBdSbHY5DEdOHBn909KFL&acctmode=0&pass_ticket=TjZyFFVwa4lZdT%2Fxu%2BGTEcj2UMcDGySST%2Faw920w2bi7CNllnsAAlXG975gsFkun&wx_header=1&fasttmpl_type=0&fasttmpl_fullversion=7351805-zh_CN-zip&fasttmpl_flag=1#)

![图片](https://mmbiz.qpic.cn/mmbiz_gif/dgmES0HFW0tMiaQE3PThel6EVf4MAGZCjWRjoVagOpyuibYkjlkIZgDePbcDwiakPwvdRYUkYcHTQBAjVl05dN17Q/640?wx_fmt=gif&wxfrom=13&tp=wxpic)

**作者 |** 18号线不到安研路

**出品 |** 十一号组织

**知圈** **|** 进“域控制器群”请加微13636581676,备注域

对于整车OTA类型，主要分为两类，FOTA（Firmware-over-the-air）和SOTA（Software-over-the-air），两者均为主机厂重点关注及逐步落地的领域，可适应不同场景的OTA需求。

**FOTA和SOTA概述**

FOTA通过给车辆控制器下载安装完整的固件镜像，来实现系统功能完整的升级更新。例如升级车辆的智驾系统，让驾驶员享受越来越多的辅助驾驶功能；升级车辆的座舱系统，提高驾驶员疲劳检测的准确率；升级车辆的制动系统，提升车辆的制动性能。特斯拉曾在Model 3在上市后，发现其刹车逻辑存在优化空间，通过FOTA对制动系统进行升级之后，制动刹车距离由46m缩短为40m，大幅提升了行车时的安全性。

FOTA涉及控制器核心功能（控制策略）的一个完整的系统性更新，对整车性能影响较大，升级过程对时序、稳定性、安全性要求极高，同时升级前置条件包括挡位、电量、车速等要求，升级过程一般不支持点火用车，蔚来车主曾在首都长安街给全国车主免费上过生动的一课。

SOTA通过给车辆控制器安装“增量包”，来实现控制器功能的一个“增量”更新，一般应用于娱乐系统和智驾系统。例如更换多媒体系统操作界面，优化仪表盘显示风格，更新娱乐主机里的地图程序时，用到的都是SOTA升级方式。

SOTA涉及控制器应用层一个小范围的功能局部更新，对整车性能影响较小，升级前置条件要求较低。SOTA的增量更新策略，可以大幅减小升级包文件大小、从而节约网络流量和存储空间。可以想象随着SOTA范围的扩大及技术的成熟，以后在车辆行驶的过程中，吃着火锅唱着歌，车辆就可以自动完成功能的更新迭代。

FOTA和SOTA功能定义目前没有明确清晰的界限。遇到男同事咨询时，我总是不负责任的举出手机升级的例子。从iOS14更新到iOS15是FOTA，从微信7.9升级到9.0是SOTA。

![图片](https://mmbiz.qpic.cn/mmbiz_png/AAPqbJEf3f5K9P0XGsBwT8f5UqHERxrp6QQztNicPudJ6ibo6jVNEPJuorItxqmyicYyyeiahlDDia8y1K752jNaxqg/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

本文主要介绍当前对于整车功能的SOTA技术链路概述。本文介绍以下三种技术路径，一种是针对车机、仪表APP类应用的基于Android平台的SOTA架构；其二针对基于Autosar AP的控制器，通过UCM服务实现新功能的上线；最后一种为当前的研究热点——云边协同策略。

**Android应用的实现**

Android的SOTA技术已经相当成熟，点击应用商店中应用旁的更新按钮即可实现，读者在自己的手机上已操作不下百遍，当前不少车机、仪表的系统基于Android实现，针对Android平台的APP应用、主题、皮肤，实现路径类似于手机的应用商城，云端建立版本仓库，用户在车机软件商店点击安装后，车端从TSP下载安装包(apk)，由车机或仪表执行安装或卸载。

![图片](https://mmbiz.qpic.cn/mmbiz_png/AAPqbJEf3f7eSFhrMDcgGyUG5rdKXG9eG9B6b1emHMekuQHnFx46ictzQZ7K3aZIIN521pceNPQNwTk9cPd9LZg/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

**基于AP AutoSAR 的SOA实现**

Autosar CP架构下，所有应用都是静态配置的，一旦软件编译完成就不可更改，其调用的周期也是确定。而在Autosar AP架构下，一切都是OS中的进程，应用是动态运行的，何时调用、进程生存周期、资源占用及进程结束等都由系统动态管理，好比你手机上的App何时打开、运行后其会调用的资源及何时关闭都是动态进行的。应用们通过ARA(Autosar Run-time for Adaptive)进行通讯，可支持或扩展对SOME/IP、TSN、DDS等SOA通讯技术的应用。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

在AP平台的服务中，UCM负责安全地更新，安装，删除和保留软件记录。类似Linux中的dpkg或YUM等软件包管理系统，可以更新或修改Adaptive Platform上的软件。

UCM实现SOTA功能的业务流程如下图所示，其中包含如下几个重要的模块。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

UCM Master：为UCM提供服务接口的客户端，它从云端或诊断工具接收、验证、解析软件包，并将软件包传输至UCM或诊断应用程序进行后续激活、回滚等处理。

UCM：为与UCM Master处于同一网络中的UCM服务实例。

OTA Client：建立云端和 UCM Master 之间的通信，以传输增量包的信息。

车辆状态管理器：从多个车端ECU收集状态，并计算相应的安全状态，根据车辆包中的安全策略，在发生变化时通知UCM Master。如果不满足安全策略，UCM Master可采取相关措施，如通知用户当前不满足前置条件，同时推迟、暂停或取消更新。

整个SOTA流程包含以下几个过程：

**一、升级包的打包和组装**

UCM的安装单位是软件包。该程序包包含一个或多个可执行文件。除了应用程序和配置数据外，每个软件包包含Manifest，Manifest是arxml类型的文件，元数据包括软件包名称、版本、依赖关系等。

**二、升级包传输**

使用UCM服务接口的客户端可以位于同一个AUTOSAR AP平台上，也可以是远程客户端。一个或多个软件包传输到UCM的内部缓冲区。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

传输完成后，会对软件包的内容进行身份验证。

**三、升级包安装**

安装和卸载操作通过UCM的ProcessSwPackage接口执行。值得一提的是，UCM支持A/B分区升级。对于A/B分区升级方案，让旧版本先在A区运行，升级非活跃的B分区。处理完成重启后，运行B分区的新版本。A/B分区的升级策略可以实现车辆行驶过程中的应用部署，即便升级失败也不会影响整车的其他功能。

**四、升级包激活**

对于一些功能，UCM在升级过程中必须同时更新多个软件，因此UCM需根据软件包的依赖关系后激活。如升级是通过 A/B 分区的，分区之间发生交换，新的非活动分区成为新活动分区的副本。

**五、升级包回滚**

遇到异常应用需要回滚至稳定版本，UCM决定回滚的版本选择，而回滚操作由Autosar AP的Persistency服务处理。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**基于云边协同的实现**

介绍之前，先絮叨几句虚拟化、虚拟机和容器。每位程序员的开发环境配置可能会有所不同，这会导致你的代码直接拿到同事的电脑上无法正常运行。企业的开发环境、测试环境和生产环境在配置和所依赖的文件上也会有所不同。如何确保在不同硬件、不同环境都能正常运行你的BUG，是当前软件行业的热点。云原生是该理念的最佳实践，通过微服务，容器化，持续交付等新技术，实现了开发运维一体化，让服务生于云，长于云。

虚拟机（VM，Virtual Machine）是虚拟化技术的落地先锋，通过软件模拟的具有完整硬件系统功能的、运行在一个完全隔离环境中的完整计算机系统，可以显著提高设备的工作效率，减轻了企业对硬件资源的依赖。

当前汽车行业落地的是Hypervisor 的虚拟化方案，其可以在多核异构的单芯片上运行多个不同类型的操作系统，各系统间共享硬件资源，既是彼此独立又可交互信息。Hypervisor 即满足了日益复杂场景下的不同业务需求，又提高了硬件资源的使用效率，大幅降低了成本，虚拟化所具备的不同操作系统间的隔离能力，大大提升了系统的可靠性和安全性。

座舱中的联屏车机往往会采用该方案，实现主驾和副驾不同系统的需求。但是每个VM都需要安装操作系统才能执行应用程序，倘若用户只需在VM中运行或迁移简单的应用程序时，采用VM不仅操作繁琐而且浪费大量资源，因此企业迫切需要轻量级的虚拟化技术。

容器技术起源于Linux，是一种轻量化的内核虚拟化技术，通过隔离进程和资源，共享同一个操作系统内核，可以保证在开发、测试以及生产等不同环境和硬件配置下都具有移植性、一致性。虚拟机技术与容器技术的典型功能架构框图如下。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

容器技术和虚拟机技术关键特性的对比如下。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

容器技术随着Docker的出现才在“格子衬衫”群体中流行开来。Docker将集装箱思想运用到软件打包技术上，为代码提供了一个基于容器的标准化运输系统，其产品logo就是这一思想的最形象诠释。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

Docker 是一个开源的应用容器引擎，开发者利用Docker可以将任何应用及其依赖打包成一个轻量级、可移植、自包含的镜像，然后发布到任何流行的 Linux、Windows等操作系统的机器上。Docker 技术包含三个核心基础概念：

（1）镜像（Image），一个层叠的只读文件系统，除了提供容器运行时所需的程序、库、资源、配置等文件外，还包含了一些为运行时准备的配置参数。镜像不包含任何动态数据，其被创建之后内部不会被改变。镜像用来创建 Docker容器，同一个镜像文件，可以生成多个同时运行的容器；

（2）容器（Container），容器是镜像创建的运行实例，容器可以被创建、启动、停止、删除、暂停等。每个容器都是相互隔离；

（3）仓库（Registry），镜像文件存放的地方。用户创建完镜像后，可以将其上传到到仓库，需要在另一台主机上使用该镜像时，只需要从仓库上下载即可。

Docker使用客户端/服务器架构模式，运行逻辑如下图所示。Docker守护进程作为服务端的请求，负责构建、拉取、启动Docker容器。Docker守护进程一般在Docker主机后台运行，用户使用Docker客户端直接跟Docker守护进程进行信息交互。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

Docker客户端，向 Docker守护进程发起Docker构建、拉取、启动的请求，Docker守护进程完成操作并返回结果。Docker客户端既可以在访问本地Docker守护进程，也可以访问远程Docker守护进程。蓝色虚线展示了Docker构建并存放于本地Docker的构建工作流。紫色虚线展示了从镜像仓库拉取镜像至Docker主机或将Docker主机镜像推送至镜像仓库的拉取工作流。红色虚线展示了镜像安装及容器启动的启动工作流。

Docker主机，一个物理或者虚拟的机器用于执行 Docker守护进程和容器。

Docker守护进程，接收并处理Docker客户端发送的请求，监测Docker API的请求和管理Docker对象，比如镜像、容器、网络和数据卷。

Docker目前在互联网的线上行业应用非常广泛，实现了“Write Once，Run Everywhere”的目标，常见的应用场景如下。

（1）对于开发而言，服务和应用更容易跨平台移植。避免了由于平台的变化而生成的Bug，同样也避免了和测试、运维之间扯皮甩锅。不用再抱怨“怎么在正式环境上总是BUG，测试环境可是没这个问题的”。老板再也不用担心环境迁移造成问题的可能性，更好给下属们分锅了。

（2）对于运维而言，可以通过容器编排应用（如K8S），按业务需求自动管理应用的实例，适合动态扩容和缩容。典型的场景，就是电商的双十一活动会导致订单服务的负载剧增，无容器技术时，运维需要“自愿”通宵加班，手动新增订单服务的实例，并实时监控各实例的状态。目前，运维可能左手咖啡，右手王者，眼睛盯一下就可以了。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

对于上文提到的容器编排，你需要管理运行应用程序的容器，并确保不会停机。例如，如果一个容器发生故障，则需要启动另一个容器。如果系统处理此行为，会不会更容易？这时，Kubernetes和KubeEdge就出场了。

Kubernetes提供了一个可弹性运行分布式系统的框架，满足扩展要求、故障转移、部署模式等。Kubernetes能够自主的管理容器来保证云平台中的容器按照用户的期望状态运行。在Kubernetes中，所有的容器均在Pod中运行，一个Pod可以承载一个或者多个相关的容器，同一个Pod中的容器会部署在同一个物理机器上并且能够共享资源。Pod也可以包含0个或者多个磁盘卷组，这些卷组将会以目录的形式提供给一个容器，或者被所有Pod中的容器共享。然而，Kubernetes的应用场景仅限于线上的云服务，并不支持设备端。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

KubeEdge基于Kubernetes构建，可将本机容器化应用编排和管理扩展到边缘端设备。为网络和应用程序提供核心基础架构支持，并在云端和边缘端部署应用，同步元数据，主要优势有如下四点：

（1）边缘计算。通过在Edge上运行的业务逻辑，可以在生成数据的本地保护和处理大量数据。这减少了网络带宽需求以及边缘和云之间的消耗。这样可以提高响应速度，降低成本并保护客户的数据隐私；

（2）简化开发。开发人员可以编写基于常规HTTP或MQTT的应用程序，对其进行容器化，然后在Edge或Cloud中的任何位置运行它们中的更合适的一个；

（3）Kubernetes原生支持。借助KubeEdge，用户可以在Edge节点上编排应用，管理设备并监视应用和设备的状态，就像云中的传统Kubernetes 集群一样；

（4）大量的应用。可以轻松地将现有的复杂机器学习，图像识别，事件处理和其他高级应用程序部署和部署到Edge。

**说了这么多，和SOTA又有什么关系呢？**

近日，在CNCF举办的云原生会议上，报道了某OEM打造的车云协同案例，通过KubeEdge这一轻量化的框架，将汽车作为节点接入云端，由云端动态管理节点上的服务。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

该架构引入了容器化技术，将功能与底层软件完全解耦，容器中承载的功能包括智能座舱、远程协作、机器学习、自动驾驶等高存储高算力的业务。实现后能为将来的第三方开发平台打下基础，可构建丰富的车辆功能生态，软件可售业务也可基于此实现。更重要的是，车辆可改变仅通过FOTA实现功能迭代变革的现状，通过该框架，或许能够实现不停车升级，节约用户的时间成本。

传统的分布式电子架构，在软件定义汽车的浪潮中，遇到了很多发展瓶颈。在新功能的迭代问题上，系统复杂度大，缺乏灵活性和扩展性；软硬件紧耦合导致很难实现横跨多个ECU/传感器的复杂功能，对FOTA也提出了更高的要求。因此，计算能力中央化，建立通用的计算平台是各家OEM的研究方向，云边协同作为其中功能迭代的另一条通道，解决了FOTA升级时间长、车辆状态要求苛刻的痛点，一旦落地势必将引起了各方的极大关注，可能成为SOTA的首选技术路径。

**小结**

当前，车机第三方APP的SOTA生态和运营已趋于成熟，但对于整车控制器相关功能的SOTA而言，技术落地尚在进行中。不过，OEM在完成SOTA的基础能力建设之后，内部或第三方应用的开发者，只需遵从相应的开发规范，通过调用引擎的接口即可实现SOTA。相信在落地之后，对于软件定义汽车又是一个极大的技术进展。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

阅读 1778

​

写留言

**留言 3**

- ᴍ.ᴀʀʀᴄ

  2022年3月19日

  赞

  目前的车载应用商店里的应用版本更新是SOTA吗？通过运营平台进行推送升级是否属于SOTA范围？

- 星辰大海

  2022年3月19日

  赞

  仓库单词打错了， 是不是这个repositary

  焉知汽车

  作者2022年3月19日

  赞

  ……

已无更多数据

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/dgmES0HFW0t9jic3zA309PJF849hJuDBOFAdEUBibqQ4N0kPG00mibHAjpkNia6Wf7ZN0HW64Espl5socDqsvyx2cw/300?wx_fmt=png&wxfrom=18)

焉知汽车

524

3

写留言

**留言 3**

- ᴍ.ᴀʀʀᴄ

  2022年3月19日

  赞

  目前的车载应用商店里的应用版本更新是SOTA吗？通过运营平台进行推送升级是否属于SOTA范围？

- 星辰大海

  2022年3月19日

  赞

  仓库单词打错了， 是不是这个repositary

  焉知汽车

  作者2022年3月19日

  赞

  ……

已无更多数据
