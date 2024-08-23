
Original asson OPPO内核工匠

 _2023年04月14日 17:00_ _广东_

 **术语**

1. Node：插件，节点，处理数据的最小实体单元；
    
2. Parent Node和Child Node：前向Node和后向Node
    
3. Pipeline：管道，用于管理插件，可以由至少一个插件、至少一个Sub Pipeline组成
    
4. Sub Pipeline：子管道，用于简化Pipeline的设计，sub Pipeline可嵌套，内部可以包含至少一个插件
    
5. Port：端口，依附于node，分为input 和 ouput Port
    
6. 领域设计：为了解决某个领域内常见的、复杂性问题而提出的一种通用软件设计解决方案
    

  

**一、 背景**

广义多媒体子领域中的多个框架，比如分布式媒体框架、多模态创作引擎中的编辑框架、video子系统的stagefright框架、camera子系统的camx-chi框架（qcom）都离不开Pipeline领域设计，此外audio子系统hal层3A在内的算法责任链、display子系统的DRM框架底层实现也有pipeline的影子。究竟何为pipeline领域设计，以及该领域设计的要点是什么，正是本文要阐明的问题。

  

音视频媒体业务多样和复杂，可以简单划分为传统的多媒体业务（采编播），以及这三种基础业务的组合：

  

1. 比如Wifi Display可以由采集和播放的组合而来；
    
2. 比如远场RTC，上行可以由录制的pipeline组成，下行可以由播放的pipeline组成；
    
3. 比如说高清视效，可以基于播放的pipeline加上超分等filter构成；
    

此外，Camera子系统的业务场景更加复杂，因为pipeline设计上一方面要充分发挥底层ISP硬件的pipeline能力，另一方面要满足拍照等后处理竞争力场景（夜景、夜枭、HDR、多摄融合、人像、专业等多种模式），同时也要考虑到框架的灵活扩展，导致上层pipeline设计上更加复杂。

  

正因如此，我们化繁为简，先从本质入手，首先思考下符合Pipeline领域设计的多媒体框架的要求是什么。在我看来，至少要满足以下2点关键要求：

1. 插件 **快速** 集成 - 满足各种各样的新的cv、音频算法的集成
    
2. Pipeline的可**配置** - 满足多样的业务场景，每个业务场景对应一个Pipeline
    

  

下文先对广义多媒体领域中的媒体框架做简单介绍，接着以基础的视频播放为例来说明pipeline思想精髓，最后我们会结合video领域的媒体框架说明pipeline领域设计的演进方向。

  

**二、竞品分析**

本章节的竞品分析，从下面主要几个维度来展开：

1. 信息流
    
2. 控制流
    
3. 数据流
    
4. 线程模型
    
5. 框架（可配置、插件、pipeline框架等）
    

从这5个维度分析一方面可以快速抓住媒体框架的本质或者关键点，另一方面可以看出该框架的优缺点。

  

**2.1 FFMPEG**

  

FFMPEG这里就不展开介绍，值得一提的是，按照上述媒体框架的指标，FFMPEG不能算是一个多媒体框架。实际中使用FFMPEG较多的场景主要是FFMPEG的插件库。此外FFMPEG中有部分插件是GPL license的，需要同过disable_gpl宏来关闭gpl license的插件。

  

不过，FFMPEG中也有pipeline filtergraph，用于音视频的算法处理，有两种graph的创建方式：简单滤波（基于API Call的方式）以及复杂滤波（基于命令行方式构建graph），具体的可以参考[FFMPEG] 定制滤波器。

  

下图中黑色菱形为input pad，灰色菱形为output pad；

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

  

  

图2.1.1 FFMPEG的filter graph

**2.2 Gstreamer**

  

**2.2.1 概述**

Gstreamer是一个非常强大和通用的多媒体框架，其基于pipeline领域设计。Gstreamer框架的许多优点都来自于它的模块化：Gstreamer可以无缝地合并新的插件模块。但是由于模块化和强大的功能往往以更大的复杂度为代价，事实上开发一个新的插件或者搭建好一个新的pipeline工作量也非常大。此外Gst依赖GObject/Glibc等基础库，升级和维护也麻烦，Gst主要实现为C语言。

  

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

图2.2.1 Gst 框架图

  

蓝色背景为预置功能，黄色背景为三方拓展功能，其中：

1. Media Applications：应用层，包括Gstreamer自带的一些工具（Gst-launch，Gst-inspect等），以及基于Gstreamer封装的库（Gst-player，Gst-rtsp-server，Gst-editing-services等)根据不同场景实现的应用。
    
2. Core Framework：核心框架层，主要提供：
    

- 上层应用所需接口
    
- Plugin的框架
    
- Pipline的框架
    
- 数据在各个Element间的传输及处理机制
    
- 多个媒体流（Streaming）间的同步（比如音视频同步）
    
- 其他各种所需的工具库
    

3.Plugins：最下层为各种插件，实现具体的数据处理及音视频输出，由Core Framework层负责插件的加载及管理。

  

**2.2.2 关键术语介绍**

  

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

图2.2.2 Gst ogg pipeline

  

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

图2.2.3 Gst 关键术语示意图

  

结合图2.2.2和2.2.3，Gst pipeline涉及到几个关键的术语，解释如下：

1. 组件(Elements)：插件，组成pipeline的基本单元，每个element可以看成业务场景的子过程。一个element有至少一个source pads以及sink pads。组件按照type来划分，包括：source、filter、sink。
    
2. 管道（pipelines）：pipeline跟业务场景一一对应，一个具体的业务场景需要至少一个pipeline处理。一个pipeline由至少一个组件或者bin组成。
    
3. bin：箱柜，理解为子pipeline，bin中由至少一个组件构成。
    
4. Pads：衬垫，按照类型划分有source和sink pad，依附于Elements
    
5. Bus：总线，用于消息传递，从组件到上层应用。上层需要预先设置需要监听的消息。
    
6. Buffers：source到sink的媒体数据传输
    
7. Events：用于应用到组件或者组件间的消息传递
    
8. Messages：用于组件到应用的消息传递
    

  

**2.2.3 优缺点**

1. 优点
    

- 插件式的、基于pipeline的媒体框架
    
- 有丰富的插件库
    
- 生态强大，跨os、跨平台
    
- 可扩展，开发者聚焦于插件开发
    
- 可维护，有一定的调试工具
    

     2.缺点

- 基于c的插件开发复杂
    
- 不支持pipeline的可配置，需要通过API Call手动连接
    
- 过于灵活的机制，不一定适用多媒体具体场景（动态pipeline）
    
- 依赖glibc、gobject，版本升级和维护有一定困难
    

  

**2.3 OMX**

1. 非多媒体框架，在AL、IL、DL层分别定义了一整套层与层之间的标准接口。在早期的功能机时代一般IL层一般用于编解码。
    
2. IL层常用于视频的编解码，最新Android版本已经切换成了codec2.0。
    
3. 接口多为异步式，API定义比较晦涩。
    
4. OMX原生接口不能够满足嵌入式媒体框架的需求，Android做了一定的拓展，比如说为了满足解码输出buffer零拷贝，加入了usegraphicbuffer的支持等。
    

**2.4 DirectShow/MediaFoundation**

**2.4.1 Directshow**

  

2.4.1.1 概述

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

图2.4.1 DirectShow软硬件框架

  

Directshow是由模块化功能模块组合而成，每个功能模块都采取 COM 组件方式，称为Filter。 Directshow  提供了一系列的标准的模块可用于应用开发，开发者也可以自定义Filter 来扩展Directshow 的应用。下面以播放一个 AVI 的视频文件为例：

  

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

图2.4.2 window AVI播放graph

  

2.4.1.2 关键术语介绍

  

1. Filter ：从统一的CBaseFilter抽象类中继承，一般分为下面几种类型：
    

- source filter： 数据来源可以是文件、网络、照相机等。
    
- transform filter： 变换过滤器的工作是获取输入流，处理数据，并生成输出流。
    
- renderer filter：显示或者播放，或者保存本地文件等。
    
- splitter filter： 分割过滤器把输入流分割成多个输出。例如， AVI 分割过滤器把一个 AVI 格式的字节流分割成视频流和音频流。
    
- mux filter： 混合过滤器把多个输入组合成一个单独的数据流。例如，AVI 混合过滤器把视频流和音频流合成一个 AVI 格式的字节流。
    

     2.Filter Graph Manager和filter graph：

- Filter Graph Manager提供API来构建filter graph，filter graph类似于Gst的pipeline（管道）。
    
- 支持动态pipeline，如图2.4.3。
    

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

图2.4.3 动态pipeline演示

  

- 支持通过API Call的方式搭建pipeline，也支持配置文件方式（可视化编辑）搭建pipeline。
    

3.Pin：类似于Gst的pad，从类型上来看有input和output。从上一个filter的out pin输出到下一个filter的in pin。子类Pin从CBasePin抽象类中继承。

4.控制流：

- 分层，上层不能直接控制插件，通过pipeline来控制。
    
- 处理结果以event的方式通知到上层。
    

5.信息流：

- Graph 事件 ：Graph 管理器采用事件机制将 graph 中发生的事件通知给应用程序 。
    

6.数据流：

- IMemAllocator ：内存分配器抽象类，引用技术的方式来管理内存
    
- IMediaSample ：数据流抽象类，分配的内存保存在内存池中（推测）
    
- 数据流推拉机制：push和pull。推模式使用的是IMemInputPin 接口，拉模式使用 IAsyncReader 接口，推模式比拉模式要更常用 。                                              
    

7.工具支持：graphEdit，支持可视化编辑。

  

2.4.1.3 优缺点

  

1. 优点
    
    优点类似于Gst框架。
    
    生态完善，插件库丰富。
    
2. 缺点
    
    不支持drm。
    
    filter基于com机制，对开发者有一定的要求。
    

  

**2.4.2 MediaFoundation**

  

2.4.2.1 概述

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

图2.4.4 mediafoundation graph示意图

  

Media Foundation提供两种不同的编程模型，类似于Android stagefright。前者应用只需要基于MediaSession的API来控制媒体业务运行，数据流经过MediaSession，有MediaSession传给下一个插件；后一种方式应用可以获取到数据流，并自我搭建pipeline完成数据流的传递。

  

提供2种不同的搭建pipeline的方式来满足不同业务场景的需求（类似于Android的Stagefright和ExoPlayer）。在pipeline框架设计上，思想跟directshow类似。

  

2.4.2.2 关键术语介绍

  

- 工具支持：MFTrace和TopoEdit
    
- 其他参考官网，设计思想继承于Directshow。
    

  

2.4.2.3 优缺点

  

1. 优点
    

支持drm。

对hdr视频支持的更好。

2.缺点

相比directshow完全推到重来，directshow用户难以迁移过来，目前也是只是在drm场景下替代了directshow。

**2.5 AVFoundation**

  

2.5.1 概述

  

在apple官网上没有找到技术架构方面的描述，下面这张图来自于csdn外网，当前最新的AVFoundation统一apple的多个设备，watch、ios、pc等。

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

图2.5.1 AVFoundation分层架构

  

1. AVKit和UIkit分别对应视频播放和录制，提供更高level的API的封装，接口易用。跟后面提到的AVAsset等接口不一样的是，基于AVAsset等模块允许开发者自己搭建pipeline，适用于更加复杂的业务场景。
    

  

2.AVFoundation：该层主要的模块如下：

- AVAsset ：是否可以用于回放、编辑、导出；获取该媒体的内容持续时间、创建日期、首选播放音量等技术参数。
    
- AVMetadataItem ：提供元数据支持，允许开发者读、写媒体的描述信息，比如唱片和艺术家信息。
    
- AVPlayer 和 AVPlayerItem：视频播放，此外还有AVAudioPlayer用于音频播放。
    
- AVCaptureSession ：视频录制，此外还有AVAudioRecorder用于音频录制。
    
- AVAssetReader 和 AVAssetWriter ：实现对媒体资源更底层数据（字节级）的操作。
    

      3.Core audio：提供了数字音频服务为iOS与OS X, 它提供了一系列框架去处理音频

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

图2.5.2 core audio分层架构

  

4.Core media：处理音视频（媒体流）的pipeline

Core animation：5.Corehttps://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/CoreAnimation_guide/Introduction/Introduction.htmlAnimation是iOS和 OS X 平台上负责图形渲染与动画的基础框架，如下，其中metal和graphics分别是3d和2d渲染的API；

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

图2.5.3 Core Animation分层架构

  

注：Core Audio、Media以及Animation 框架支持用户调用不同level的API来满足不同的业务场景。

  

2.5.2 关键术语介绍

  

内部实现无法得知，从技术上推断也是应该采用了pipeline领域设计。

  

2.5.3 优缺点

  

1. 优点
    
    生态强大，支持不同层次的API来满足不同的业务场景需求，兼顾性能.
    
2. 缺点：NA
    

  

**2.6 MediaPipe**

  

2.6.1 概述

  

MediaPipe是google的一个开源项目（2019年推出），支持跨平台的常用ML方案，支持很多常用的AI功能。支持多种OS（Android、IOS）以及支持多个开发语言（Python、CPP以及JS），下面是几个主要AI功能。

1. 人脸检测。
    
2. FaceMesh: 从图像/视频中重建出人脸的3D Mesh，可以用于AR渲染。
    
3. 人像分割: 从图像/视频中把人分割出来，可用于视频会议，像Zoom/钉钉都有这样的功能。
    
4. 手势跟踪：可以标出21个关键点的3D坐标。
    
5. 人体姿态估计: 可以给出33个关键点的3D坐标。
    
6. 头发上色：可以把头发检测出来，并图上颜色。
    

以facedetect为例，MediaPipe提供了可视化的pipeline编辑工具（类似directshow / mediafoundation），右边是facedetect的pipeline配置，左边是对应的可视化pipeline topology图（如图2.6.1），两边都支持同时编辑，实时刷新。下面的pipeline是基于gpu的人脸检测，在检测之前根据人脸检测算法要求进行预处理（imageTransformation），然后做边缘填充（ImageTransformation_2），通过gpu tflite推理得到具体的人脸框，然后叠加到原始图像上（AnnotationOverlay）。可以看出facedetect pipeline不仅支持人脸检测，最终的结果也方便地叠加到了image上。

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

图2.6.1 mediaPipeline facedetect graph示意图  

  

2.6.2 关键术语介绍

1. Calculator：
    

- 算子：对应Gst中的Element。算子中API比较简单（只有4个基础API：GetContract/open/process/close），都为同步接口。算子在定义的时候注册到全局的list中，当业务启动的时由框架引擎先校验配置文件，校验通过后根据配置文件生成对应的graph topology，并通过算子的name来匹配要加载的算子。MediaPipe已经包含了多个由Google实现的计算单元（预定义），也向用户提供定制新计算单元的基类来支持用户自定义的算子。
    
- Stream：按照类型划分有input和output，一个算子中有至少一个input stream和至少一个output stream。
    

2.graph：

- Graph表示pipeline，一个graph中由多算子或者subgraph组成。
    
- GraphConfig：graph配置，描述图的拓扑和功能的配置信息。图2.6.1右侧对应facedetect的graph配置。graph配置是否有效，通过ValidatedGraphConfig模块来校验。
    
- Subgraph：子图，方便用户更大粒度的复用，类似于Gst中的bin。
    

3.线程模型：采用线程池方案，每个算子的process函数（同步接口）作为一个task，加入到taskqueue中，调度器从taskqueue按照fifo顺序（或者优先级（待确认），支持不同的executor）从线程池中找出一条空闲线程执行。

  

4.控制流：

分层架构，算子由graph直接驱动，业务只能调用graph接口。

  

5.信息流：

通过protobuf来定义，并在graph和caculator间、以及caculator之间传递。

  

6.数据流

封装形式：Packet，支持任何类型的数据。

支持自动引用技术，每个input stream和output stream有共享的buffer pool。

  

**2.6.3 优缺点**

1. 优点
    

- 插件式的、基于pipeline的媒体框架。
    
- 有常用的ML插件库。
    
- 有一定的工具（可视化编辑）和调试手段。
    

2.缺点

- 缺少媒体框架音频和视频的一些基础插件（音频编码，视频hw编解码等）。
    
- API不满足采编播的业务场景需求，API太多，缺少一些必要的控制手段（比如说播放的暂停和恢复，stop等控制手段）。
    
- 生态不完善，开发者有限，NV有自己的deepstream框架。
    

  

**2.7 Cow**

  

2.7.1 概述

  

cow是基于Alios系统开发的基于pipeline设计的、插件式的媒体框架，其下内置丰富的插件，其上支持多种不同的多媒体业务。pipeline框架主体部分基于c++开发，上层基于js语言。

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

图2.7.1 cow媒体框架示意图

  

**2.7.2 关键术语介绍**

  

1. Node：插件。
    
2. pipeline：由Node构成，不支持subpipeline。
    
3. 信息流：
    

- pipeline和Node之间以及Node之间的信息交互都是通过MediaMeta（key-value）。
    

4.数据流：

- 支持pull和push方式，支持MediaBuffer的统一封装形式。
    
- 支持buffer引用技术管理，生产者负责制，但缺少内存池管理。
    
- 如果前后插件的推拉动作不匹配，需要加入额外的推拉插件来串联，保证数据流通。
    

5.控制流：

- Node只接受pipeline的控制，上层接口只能控制pipeline（分层）。
    
- Pipeline和Node接口多为异步接口，reset为同步。
    

6.线程模型：

- 每个Node内部有消息处理主线程和work线程（按需）。
    
- 当pipeline比较复杂的时候，会消耗很多的线程资源。
    

  

**2.7.3 优缺点**

1. 优点
    

- 插件式的、基于pipeline的媒体框架。
    
- 丰富的基础插件库。
    
- 架构灵活，易于维护和拓展。
    

2.缺点

- pipeline缺少配置化和可视化手段，需要通过API Call的方式来搭建pipeline。
    
- 当pipeline过于复杂的时候，导致pipeline内部的线程太多。
    
- 调试工具有限。
    
- 缺少内存池等统一管控内存的手段。
    

  

**2.8 AVPipeline**

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

图2.8.1 avpipeline多媒体底座示意图

  

如图2.8.1，AVPipeline可以看成cow媒体框架的升级版本，跨os/设备，不过当前只支持Android，基于NDK开发，有java和c++两套接口来满足不同层次的开发者，在cow的基础上新增了如下功能：

- 支持pipeline可配置（新增pipeline解析引擎）和API Call的方式构建pipeline。
    
- 简化数据流的推拉方式，只保留push的方式。
    
- 简化插件和pipeline的线程管理逻辑。
    
- 支持更多的AI插件（超分，场景检测，声音事件检测）。
    
- 基于可配置的pipeline框架，支持单个插件再封装后开放给app来调用。
    

  

更多的详情在第三章节，以及后续的演进在第四章节中展开。

**2.9 鸿蒙**

OpenHarmony技术架构在分布式媒体框架技术文档中有介绍，这里不展开了。这里说一说OpenHarmony中跟多媒体相关部分。

  

截止2019年9.30，鸿蒙系统中的多媒体框架大概分为2个阶段：

1. 阶段1：类似于Android stagefright，对文件mux和demuxer、音视频decoder和encoder、音视频sink 各 定义了一套interface，框架部分按照具体的业务场景来调用这些interface，pipeline固化，不易拓展。interface的具体实现一部分由鸿蒙实现，另一部分开放给外部开发者实现。
    
2. 阶段2：借助Gst的原始开发者，对Gst做了裁剪，只保留了Gst-core以及Gst必要的逻辑（相应的Gst框架的大部分实现上的优点也得以保留），并满足iot等设备的要求。后续在Gst裁剪的基础上开发媒体业务。
    

缺点：同Gst媒体框架的缺点。

**2.10 Stagefright**

Android的媒体播放框架采用过的技术方案分别有：OpenCore->Awesomeplayer->Nuplayer。从Android5.0版本（大概）开始，前2个技术方案被废弃。

  

媒体录制框架采用过的技术方案分别有：OpenCore以及stagefrightRecorder。从Android4.0版本（大概）开始，前一个技术方案被废弃。

  

当前的播放和录制框架提供了一整套控制接口给app来使用，如果需要更加细粒度的控制方式，需要使用Android提供的exoplayer，参考2.12章节。

  

stagefright框架大家都比较熟悉，不展开分析，说下其缺点：

1. pipeline固化，不易拓展。
    
2. 插件间的接口不一致（soource、filter、sink），pipeline管理麻烦。
    
3. video codec和sink的接口耦合。
    

结论：没有遵从pipeline领域设计。

  

**2.11 其他多媒体框架**

ijkplayer：ijkplayer是由b站开源的播放器项目，底层基于FFMPEG等三方库, 支持Android和iOS操作系统，有一定生态和开发者。

  

ExoPlayer：ExoPlayer是一款适用于Android的应用程序级媒体播放器。它为Android的MediaPlayer API提供了一个替代方案，可以在本地和互联网上播放音频和视频。ExoPlayer支持Android的MediaPlayer API目前不支持的功能，包括DASH和SmoothStreaming自适应回放。与MediaPlayer API不同，ExoPlayer易于定制和扩展，并且可以通过Play Store应用程序更新进行更新。

  

MLT framework：基于插件化的编排机制，可按需进行灵活的功能定制和扩展，是一款成熟稳定的开源多媒体编辑框架。

  

**三、Pipeline在video领域的已有实践**

本章以2.8章节的AVPipeline为例，说明Pipeline领域设计的运用。

  

**3.1 关键点**

Pipeline 作为一个领域设计，其思想为：保证数据 只在 管道中进行高效地流通，这里有2个关键点：

1. 只在管道中流通。
    
2. 高效地流通：高效是指要保证高性能（尽可能减少数据拷贝）。
    

**3.2 设计过程**

  

面对一个新的业务需求，一般先分解成几个子过程，对于每个子过程尽可能的内敛（正交设计），然后再把这几个子过程拼接串联起来，最后再把这些子过程组装成完整的链路。如下：

1. 过程分解
    

- 子过程分解的粒度要尽可能小，保证在差异化的业务场景中被方便地替换，同时在相似的业务场景中能保证复用性。
    
- 但是也不能过于小，否则pipeline的链路过长，增加pipeline链路的复杂性。
    

2.插件抽象，业务子过程称为插件，要满足以下3个原则：

- 尽可能复用已有的插件逻辑。
    
- 尽可能通过已有的插件来组合。
    
- 在不满足前两条情况下，开发新插件。
    

3.插件组合

- 插件组合的目的是为了生成pipeline（作为管理插件的实体，跟具体的业务场景相关）。
    
- 插件组合以配置文件的方式组合。
    
- 尽量复用已有的pipeline，通过pipeline裁剪的方式来构造满足业务场景的pipeline。
    
- 对于高频出现的插件组合，封装成sub pipeline方式来复用。
    

4.封装

- 基于Pipeline的抽象接口封装，使用外观模式封装成符合具体业务场景的API。
    
- 单进程、cs模式都封装在实体内部。
    

**3.3 音频播放**

前面说的比较抽象，下面我们从一个实际的应用场景出发，以视频超分为例，一步步展开，并展示如何通过Pipeline来是实现此功能。先从最简单的音频播放场景展开，一个普通的音频播放大概可以分为三个子过程：

1. 音频文件解析：对应AVDemuxer，输入为本地或者网络侧的url（或者数据），输出音频流（mp3等音频编码格式），可以基于FFMPEG或者其他三方库来实现；
    
2. 音频数据解码：对应AudioDecoder，输入为音频流，输出PCM，可以基于FFMPEG或者其他的音频解码三方库来实现；
    
3. 音频PCM渲染：对应AudioSink，输入为PCM，直接输出到speaker或者耳机设备等，可以基于pluseaudio、cras、audioopensl、aaaudio等接口来实现；
    

在分解的过程中，需要综合考虑分解的插件粒度和性能；具体的分解结果用类图表示如下：

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

图3.3.1 audio pipeline示意图

  

如上，PipelineAudioPlayer为管理音频插件的pipeline，注意到数据只在插件中流通，不会到Pipeline层面（PipelineAudioPlayer）。此外，还需要注意到初始版本每个插件的输入和输出的数据格式不一样。

  

**3.4 抽象**

**3.4.1 Node抽象**

  

考虑到PipelineAudioPlayer对每个具体的插件进行统一的状态、事件等管理操作，按照依赖倒置原则（DIP），上层Pipeline不应该依赖具体的插件，而是应该依赖插件的抽象接口，所以新增了一个INode基类，INode中定义了插件的标准接口，这样PipelineAudioPlayer只依赖INode，内部有一个数组mNodes，维护着所有的插件；在运行状态，Pipeine的命令绑定到具体的子类插件；对性能要求较高的场景也可以采用静态绑定。

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

图3.4.1 Node抽象

**3.4.1.1 插件类型划分**

  

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

图3.4.2 Node类型划分示意图

  

我们把插件层单独抽离出来，如上。从前后连接关系发现：

1. AVDemuxer：只有后向插件，一般称为Source Node，负责源数据的获取和管理；
    
2. AudioDecoder：有前向和后向插件，一般称为Filter Node，负责数据的处理；
    
3. AudioSink：只有前向插件，一般称为Sink Node，负责数据的渲染，渲染到device设备、io设备（保存本地文件）等；
    

这三类插件由于前后连接关系和在pipeline中的位置不一样，接口设计上也有差异，差异部分的逻辑放在各自的基类中，因此插件的继承关系修改如下：

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

图3.4.3 node分层示意图

  

**3.4.1.2 即插即用**

  

由于业务场景和pipeline一一对应，这些pipeline各自管理者数量不等、功能异同的插件集。

从节省内存等资源角度出发，在业务加载的过程再加载对应的插件，并为之分配系统资源，在业务结束的时候及时卸载对应的插件，并回收初始化分配的系统资源；所以在编译时候，需要把每个插件编译生成一个动态库，在业务运行之初通过插件管理器对这些插件加载，这便是插件的即插即用。

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

图3.4.4 node工厂示意图

  

**3.4.2 Pipeline抽象**

同样，多个业务场景下对应多个pipeline，这些Pipeline需要抽象出一套接口IPipeline，便于更上层模块对其进行管理，此原因之一；原因二Pipeline模块内部也有业务逻辑和非业务逻辑同样需要分离管理（参考章节3.10.2），把非业务逻辑放在公共的模块中管理；

这样，修改后的音频播放pipeline和插件类图如下：

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

图3.4.5 pipeline抽象

  

**3.5 外观模式**

API跟具体的业务场景相关，比如录制的API跟播放的API定义肯定有所区别，如何在统一的IPipeline基础上定义符合业务场景的API（易用性）。具体到音频播放，需要封装AudioPlayer的接口给client端调用，如下：

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

图3.4.6 业务场景API封装

  

抽象出这套接口还有一个收益，隐藏了AudioPlayer和IPipeline的通信方式，AudioPlayer设计成单进程模式或者cs模式都可以（单进程模式一般用于三方媒体框架，cs模式用于系统媒体框架），方便后续的拓展，比如差异化的通信方式可以在AudioPleyer的子类中实现，对AudioPlayer的client隐藏其实现细节。

  

**3.6 Pipeline（Graph）建联**

前面我们讲了pipeline的子过程分解、Node和Pipeline抽象，还没有涉及到插件间是怎么建立连接的。插件和插件的连接过程大概可以分为三步走：

1. 前向插件直接获取后向插件的实例指针，用于向后向插件中写入数据。
    
2. 在步骤一的基础上，插件抽象出类似于Port的概念，减少插件间的接触面。
    
3. 在步骤二的基础上，插件间的建立连接的过程做成可配置，用graph解析框架进行解析。
    

这里我们先简单假设，插件间的连接通过步骤一完成；在后面的章节我们会讲到其他2个步骤；

  

**3.7 控制流**

  

到这里为止，一个简单的、基于Pipeline的音频播放器引擎的雏形就算完成了。当client通过AudioPlayer的接口来完成播放的时候，start、stop等命令是通过如下链路AudioPlayer->PipelineAudioPlay->插件 传递的，可以看出这是一个简单的分层设计。需要注意的时候，AudioPlayer不应该跳过PipelineAudioPlay直接控制某个具体的插件，这是Pipeline思想在控制流设计上的一个关键点。

  

当前版本Pipeline和Node主要接口皆为异步，reset接口为同步（必须），异步接口保证在Pipeline和Node层面Api响应能及时返回。

  

**3.8 信息流**

信息流主要用于Pipeline和插件之间（传递用户设置的参数等），以及插件间的信息传递（传递媒体格式等）；

  

信息设计上需要解决以下3个方面的问题：

1. 信息的封装
    
2. 信息的解析
    
3. 信息的传递
    
      
    

**3.8.1 信息封装**

从扩展性来讲，可以设计一个MediaMeta（需要根据信息流的大小来权衡设计），内部封装key-value键值对的方式来管理所有的信息；当然也可以采用string-parcel的方式（camera hal就是使用这种方式）。

  

**3.8.2 信息解析**

当信息达到目的模块时候，目的模块按需提取该模块所需要的信息；虽然解析的时候需要一次遍历，但也是一个经济实用的方法。

  

**3.8.3 信息传递**

主要通过下面2种方式：

1. Pipeline和插件间的信息传递通过API Call的方式（比如SetParameters等）。
    
2. 通过数据流（类似于MediaBuffer，该buffer结构中不仅包含了插件间需要传递的数据，也包含了该数据的格式，比如说音频的采样率、通道数等、视频的分辨率等）。
    

第二种通知方式可以保证信息能第一时间有效通知到下一个插件。

  

**3.9 数据流**

数据流的设计上需要解决以下4点：

- 数据流的传输通道
    
- 数据的传输方式
    
- 不同的数据格式封装方式
    
- 内存碎片的管理
    

  

**3.9.1 接口隔离**

为了方便数据流的传递，前向插件和后向插件之间需要设计一个数据通道；由于每个插件的数据通道可能不一样，一般有下面4种方式：

- 单入单出（比如AudioDecoder）
    
- 单入多出（比如AVDemuxer）
    
- 多入单出（比如Muxer）
    
- 多入多出
    

前面章节提到插件被Pipeline管理，从性能角度出发，插件设计为即插即用；如果前向插件直接使用后向插件的实例指针进行数据传递的话，无疑增大了插件间的接触面，设计上来讲需要抽象出一个类似于Port的模块，如下图，只用于插件间的数据流的传递，具体的设计我们后面会讲到。

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

注：插件中并不是所有的input 和 ouput Port在某个具体的业务场景中都会用到。

  

**3.9.2 推拉方式**

数据的传递方式一般有2种：push和poll；

1. push方式：往往常用于实时流，这样上一个插件产生的数据可及时地传递给下一个插件。当然非实时流也可以用；
    
2. poll方式：一般用于非实时流，后向插件主动从前向插件中拉数据；
    

框架设计中两种方式都可以加以采用，但考虑到push方式能够满足所有的多媒体场景，这里我们只保留push方式，poll方式不考虑。这样设计的好处是，插件间的连接只需要考虑数据format是否满足，而不需要推拉方式是否满足（如果前向插件是poll，后向插件是push，会导致数据无法传递），简化了Pipeline设计。后面我们会讲到Pipeline的可配置，也是基于简化了的数据传递方式实现的。

  

**3.9.3 封装格式**

  

多媒体数据包含音频流（PCM，bytebuffer），视频流（yuv、rgb、h264、h265等），需要设计统一的封装格式（比如MediaBuffer），Mediabuffer内部封装了实际的数据，以及数据的具体类型；当数据传递到具体的插件时，在插件间连接关系正确的前提下，后向插件都能够解析该类型的buffer；如果插件不能处理该数据类型，说明Pipeline运行错误，或者配置的Pipeline有问题（没有采用类似Gst的参数协商机制）。

  

MediaBuffer设计上主要考虑以下几点：

- 管理不同类型的buffer。
    
- 减少buffer的拷贝。
    
- 主动释放，不需要用户参与内存的管理，统一的内存管理接口。
    
- 生产者负责制（生产者生产buffer后，等消费者使用完后，引用计数降为0，自动触发释放函数）。
    
- 频繁小内存分配和释放（音频流比较常见）。
    

  

**3.9.4 buffer queue**

前面提到插件间采用push方式传递数据，如果插件的处理数据速度不一致，就会导致下游插件内部堆积了大量的buffer，从而导致内存耗尽，因此需要一个buffer queue模块来管理MediaBuffer；该buffer queue模块需要考虑以下几点：

- 支持可缓存的最大buffer数目设置。
    
- 支持空闲buffer数目的查询。
    
- 缓存buffer，并在内部缓存的buffer数目大于最大数目时通知调用者（流控策略1）。
    
- 获取buffer，并在内部缓存的buffer数目小于最小数目时通知调用者（流控策略2）。
    

设计出的内存池模块大概如下：

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

 图3.9.2 buffer queue API

  

**3.9.5 防抖Port**

在接口隔离章节提到插件间的Port模块，用来传递数据；同时考虑到在数据传递的过程中防抖，Port内部应该封装前面提到的SimplePool模块，PoolWriter的类图如下：

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

图3.9.3 防抖Port

  

值得一提的是，PoolWriter和SimplePool的关系是私有继承，私有继承从语义上是用...实现，这种实现方式在pipeline的实践项目中会经常使用，详情参考前面的链接。

  

至此，PoolWriter承担起了管理插件间的buffer职能，包括数据的传递，流控策略等，至此，更新下Pipeline的类图：

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

图3.9.4 video子系统框架pipeline设计示意图

  

**3.10 业务和非业务逻辑分离**

**3.10.1 NodeBase**

每个插件内部都有一个消息处理主线程，插件的大部分接口设计为异步接口，这样当Pipeline遍历mNodes时，前面的插件API调用不会影响后面插件。插件的异步接口采用command设计模式，把外部的调用封装成一个个命令post到消息主线程中执行；

  

此外，每个插件的业务逻辑不一样，处理的时间长短不同；处理耗时的插件为了不影响消息处理主线程的调度可能需要新增线程来处理复杂的业务逻辑；

  

为了管理所有插件内部的消息处理主线程以及work线程（如果有），以及两个线程间的同步、状态管理等非业务逻辑，很自然地想到在插件的继承体系上新增NodeBase；这样，叶子插件只需要处理该插件业务相关的逻辑，后续新增插件可以快速集成。

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

图3.10.1 Node层次图

  

**3.10.2 PipelineBase**

同样的，PipelineBase引入的目的跟NodeBase类似，内部封装了Pipeline管理插件的公共逻辑，状态切换、事件上报等；叶子Pipeline只需要考虑业务相关的逻辑即可；这样，后续新增的Pipeline可以快速集成；

更新后的Pipeline继承体系如下：

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

图3.10.2 Pipeline层次图

**3.11 添加视频流**

前面的都是基于音频播放场景进行展开，加入视频流后需要做如下几个修改：

1. 前面AVDemuxer新增对视频文件的解析。
    
2. 新增VideoDecoder用于视频解码，解码得到yuv或者rgb数据后。
    
3. 新增VideoSink用于视频渲染渲染。
    
4. 新增video Pipeline用于支持音视频播放。
    

由于前面引入了NodeBase以及PipelineBase，新增插件以及videoPipeline的工作量相比从头构建Node和Pipeline的工作量已大大减少。

  

至此，我们构建了完整的视频播放pipeline链路。前面提到多媒体场景的复杂性，具体的某个业务对应至少一个pipeline，有没有一种可能用配置的方式来搭建pipeline，而不需要通过API call的方式来实现。在配置文件中描述插件间的graph topology，通过pipeline解析框架来解析，自动生成业务场景对应的pipeline，这就是下个章节要介绍的内容。

**3.12 Pipeline与可配置**

**3.12.1 配置文件选择**

首先配置文件的选择：有xml、json、yaml，从性能上来看使用yaml，从运行广泛性来看使用xml；

  

**3.12.2 可配置的参数**

3.12.2.1 Pipeline连接可配置

pipeline做成可配置的初衷是因为每个业务场景对应一个具体的pipeline，比如说音频播放，视频播放各对应一个pipeline，而pipeine建立连接的逻辑大同小异，差别在于配置参数的不一样，按照正交设计原则，分离差异化的部分，做成可配置的，这样对xml的配置文件做一个简单深搜就可以建立起Pipeline；

如下是纯音频播放的Pipeline：

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

图3.12.1 纯音频pipeline 拓扑配置

  

以最简单的配置文件为例，有几个关键点需要注意，否则Pipeline构建失败，如下：

- 插件的唯一标识，方便解析框架进行管理
    
- 插件的动态库名称，方便插件管理框架加载和卸载
    
- 插件的类型，方便解析框架处理三种不同类型插件的差异
    

插件的后向插件，方便解析框架建立连接关系（有向图）

  

同样的，音视频播放的Pipeline如下：

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

图3.12.2 音视频pipeline 拓扑配置

  

3.12.2.2 插件参数的可配置

  

插件相关参数也可做成可配置，参考Android的media_codecs.xml等文件

  

**3.12.3 配置文件解析**

配置好配置文件后，最后一步就是对配置文件进行解析，考虑到运行过程中对配置文件进行解析会增加启动时间，因此下面2种解析方案，我们优选方案1：

1. 在编译阶段触发py脚本进行解析，解析后的pipeline放在一个只读的全局数组中（最好放在static函数中，返回static数组），数组中保存的是不同的pipeline type的pipeline实例，每个pipeline实例中保存了该pipeline插件间的topology信息，这些topology信息从配置文件解析得到。
    
2. 考虑到由于每个配置文件，需要一套解析代码，可以通过meta-programming在编译阶段生成解析框架代码，参考下面这套静态反射框架 https://github.com/netcan/config-loader/blob/master/README_CN.md，该框架支持xml、json以及yaml，推荐使用json以及yaml；字段的解析仍然在运行阶段；
    

  

**3.12.4 动态Pipeline**

动态Pipeline是指在业务运行过程中，向Pipeline中插入或者删除一个或者某个插件；streamer支持动态Pipeline，但是对带来了插件以及框架设计上的复杂性。一般来讲，设计上是否需要支持是一个综合考虑、多方平衡的结果。

动态pipeline和pipeline剪枝有点类似，5.3章节会简单描述。

  

**3.12.5 Sub Pipeline**

  

类似于Gstreamer的bin，sub Pipeline允许嵌套，或者内部包含至少一个插件；可以看成一个粒度更大的插件；

  

在播放场景下，把AVDemuxer+AudioDecoder封装成一个sub Pipeline，音频播放的Pipeline可基于该sub Pipeline+audio sink组成，回顾下前面mediafoundation也是同样的设计思路。当一个或者几个node这种固定形态的组合方式在多个业务场景中频繁出现时，可以考虑封装成subpipeline的方式。

  

注意sub pipeline（或通过外观模式）和node具有同样的接口（组合模式）。

  

**3.13 插件生态**

采用了配置文件的方式可以做到如下2点：

1. 内置的插件开放给三方来单独使用或者内置到用户自定义的pipeline中
    
2. 开发者提供有竞争力的或者具体的业务插件加入插件库中（如何保证插件的质量，需要充分的测试，才能通过框架的认证）
    

  

开放现有的单个插件给外部的开发者（比如说基于GPNPU/GPU的超分、视频后处理等有竞争力的算法），有两种方案：

1. 直接开放对应插件，比如说直接调用VideoSR插件接口，上层需要适配该接口（对接到上层已有的AIKit或者其他）
    
2. 对于该插件接口进行二次封装，组成pipeline（如下），复用已有的接口（减少上层改动）。如下配置的pipeline，包含了超分VideoSR插件，上层通过appsource输入待超分的视频数据，超分后的数据由pipeline送给APPSink插件，再通过回调的方式通知到APP。APP拿到超分后的数据做进一步的处理，比如说送显等。
    

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

图3.13.1 超分pipeline拓扑配置

  

当媒体框架的插件有一定的积累后，随着媒体业务的不断拓展，插件会越加丰富，在新的媒体业务下，已有插件的复用率会大大提高，并且越来越稳定（插件粒度和稳定性是一个综合权衡的结果）。以WFD为例，source端和sink端实现如下：

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

图3.13.2 插件生态和复用

  

灰色背景的插件运行在本地播放和录像的业务场景中，WFD业务场景中这些插件被替换为绿色背景的插件。

  

**3.1.4 CV或者音频算法**

在支持Pipeline可配置的基础上行，假设新增视频超分功能，只需要做如下几点：

- 新增视频超分配置文件，下面id为4的那个插件就是超分插件，在videodecoder和videosink插件中间
    

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

图3.1.4.1 超分pipeline拓扑配置

  

- 新增视频超分Pipeline：只需要处理跟超分相关的业务逻辑
    
- 新增视频超分插件：对解码后的数据做超分
    

备注：pipeline的高性能离不开pipeline中的每个插件的通力合作，比如说为了达到超分的高性能（即数据buffer尽可能的少拷贝），videodecoder插件需要通过ndkimagereader（可以看成buffercore的producer+consumer模型，producer为解码器，consumer为超分插件）拿到解码后的数据，送入给videoSR模块，超分后的数据再送去显示；为了平滑各个插件之间的fps，就需要前面提到的PoolWriter；只有Pipeline和插件各司其职，才能达到最优效果；

  

**3.15 跨OS/平台**

媒体框架支持多OS和多平台的要点：

1. 跨OS
    

- 对于pipeline而言：基于c++11（11+）实现，没有任何跟OS相关逻辑，因此能够支持Android、windows、linux、mac等多种平台。
    
- 对于插件而言：
    

在Android上，媒体插件尽量基于ndk接口开发。

在linux上，基于linux os提供接口，比如说audiosink基于pulseaudio等，video sink基于x11等。

因此，媒体框架是否支持跨os，取决于具体的底层插件的能力，通过编译多态来决定不同OS上使能的插件。

2.跨平台或者设备

对于Android系统而言，以超分为例，使用gpnpu的超分性能更好，但是兼容性差，跟具体的芯片平台强相关；使用gpu的超分，更加通用些、兼容性更好；一方面尽可能发挥硬件的能效，一方面尽可能支持更多的设备（兼容性）

  

**四、pipeline领域设计优化**

**4.1 线程模式探讨**

在插件的设计过程中，可以分为4个迭代版本：

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

图4.1.1 pipeline线程模式演进

  

**4.2 Graph DSL探讨**

这里不展开讲解，DSL基于模板元编程，关于模板元编程的概念和文章，可以参考https://github.com/MagicBowen/tlp，这是一份很好的入门材料。

  

关于graph dsl的介绍可以参考https://github.com/godsme/graph-dsl，下面展示了通过dsl语言来建立一个子graph，代码相当简洁，但是背后的设计却相当复杂。由于graph-dsl采用了meta-programming，对开发者提出了很高的要求。

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

图4.2.1 pipeline的模板元编程配置示例

  

**五、总结**

Pipeline作为多媒体框架引擎的核心思想，历史源远流长，从Gstreamer时代开始，逐渐大行其道，其设计思想在广义多媒体各个子领域都在广泛使用。

  

通过上面介绍的竞品多媒体框架，对比下来我们不难得出pipeline领域设计的关键点，但是最终pipeline领域设计还是要落实到具体的业务场景中，并结合具体的业务场景，设计出更加灵活、可靠、可拓展、可维护的媒体框架。

  

**参考文献**

https://google.github.io/mediaPipe/

https://github.com/netcan/nano-caf

https://github.com/godsme/graph-dsl

https://github.com/MagicBowen/tlp

https://trans-dsl-2.readthedocs.io/zh_CN/latest/index.html

https://github.com/godsme/graph-dsl

https://modern-cpp.readthedocs.io/zh_CN/latest/index.html

https://ricardolu.gitbook.io/Gstreamer/

mediaPipeline：https://toutiao.io/posts/c0tj6b2/preview

领域驱动设计（DDD）

领域驱动设计

  

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

长按关注内核工匠微信

Linux内核黑科技| 技术文章 | 精选教程

Reads 1829

​

Send Message

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/d4hoYJlxOjNfxSer7sH7b1yJBvlpWyx1AdCugCLScXKu60Ezh9oSCrZw36x9GTL1qvIzptqlefgS1vkQwBE7OA/300?wx_fmt=png&wxfrom=18)

OPPO内核工匠

13249

Send Message