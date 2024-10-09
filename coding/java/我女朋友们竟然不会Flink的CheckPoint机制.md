# 

敖丙

_2021年11月12日 11:40_

以下文章来源于Java3y ，作者Java3y

\[

![](http://wx.qlogo.cn/mmhead/Q3auHgzwzM4DI3YtjSCADvHEKYQRjkRnoGrIAibbkT26EGFbS3kxBqQ/0)

**Java3y**.

austin开源项目作者

\](https://mp.weixin.qq.com/s?\_\_biz=MzAwNDA2OTM1Ng==&mid=2453155177&idx=2&sn=2a588f278ba5848d5cbb404ef99ae5a3&chksm=8cfd0feabb8a86fc8ec66611632fc119eb4062be86a30f3b83dd7782ccab803dd2e06a731dd3&mpshare=1&scene=24&srcid=1112sBo1IyghJJhXRMRQQNrX&sharer_sharetime=1636691427323&sharer_shareid=5fb9813bfe9ffc983435bfc8d8c5e9ca&key=daf9bdc5abc4e8d0d43729266eae4f2d1329efb5117aff9382c0f850ae4a6b57294849f0a116677e725b38ccf71378fbe9717b57535d98dfcd3ea60656bb11691974bb3b158897342653204fd4c3325bdb036309b4a62c5e19c67eb3f84cce38692ed8afd81cee3d75b38fbf94879540d0c6c3ebb75b0d0d5ea1770d52bb9fdb&ascene=0&uin=MTEwNTU1MjgwMw%3D%3D&devicetype=Windows+11+x64&version=63090b19&lang=zh_CN&countrycode=CN&exportkey=n_ChQIAhIQU2UDb0s6Pqqs40MYywnP4hLmAQIE97dBBAEAAAAAAMjUJzmlMfgAAAAOpnltbLcz9gKNyK89dVj0IR6R1qKpBc5EzsIuF7rMx01aJGoxQ4SAnBoKPWjWLOsHTIdeQLqfMNFPc7N9iu4py1juhM9%2B9jTcUUUzMWo6izNiQCujw9Ngf56Kazmzdyt4ikIeCQe4YoTTpY62ilESODx521ot%2BgwyRpCP97i3L%2Ff7df%2BpB0eTDT9S3caqDBHkleUhw2j3WtdZ1CFVZTi5Nvf1TnyMHZjF1WA5mP6cwqkKc0csCdbvdvii01djFIWpLeIrSLVeSd7m3t0yqoLa&acctmode=0&pass_ticket=UoLOX4S6skszhMvYRaZgtwiYHqvoFgzmYrNHlIp12EAVRru456%2BqiOJpinIrDsO4&wx_header=1&fasttmpl_type=0&fasttmpl_fullversion=7353068-zh_CN-zip&fasttmpl_flag=3#)

这里已经是`Flink`的第三篇原创啦。第一篇《[**Flink入门教程**](http://mp.weixin.qq.com/s?__biz=MzAwNDA2OTM1Ng==&mid=2453155137&idx=1&sn=d5bfea5535e636f9d9dd6cd67177fd50&chksm=8cfd0fc2bb8a86d4c4753cf81d283dd18681fc12e5f5fa1e374d37507038008a0c4ad56d82df&scene=21#wechat_redirect)》讲解了`Flink`的基础和相关概念，第二篇《[**背压原理**](http://mp.weixin.qq.com/s?__biz=MzAwNDA2OTM1Ng==&mid=2453155158&idx=1&sn=67ea0584a5f6d2cd2da91045541d508b&chksm=8cfd0fd5bb8a86c384c952fab4ab0d28cebdb76cc62d2e0c75df82d061d71b4e0ff643cd5003&scene=21#wechat_redirect)》讲解了什么是背压，在`Flink`背压大概的流程是怎么样的。

这篇来讲`Flink`另一个比较重要的知识，就是它的容错机制`checkpoint`原理。

所谓的`CheckPoint`其实就是`Flink`会在指定的时间段上保存状态的信息，如果`Flink`挂了可以将**上一次**状态信息再捞出来，重放还没保存的数据来执行计算，最终可以实现`exactly once`。

状态**只持久化一次**到**最终**的存储介质中（本地数据库/HDFS)，在Flink下就叫做`exactly once`（计算的数据可能会重复（无法避免），但**状态在存储介质**上只会存储一次）。

> 前排提醒，本文基于Flink 1.7
>
> 《**浅入浅出**学习Flink的checkpoint知识》

## 开胃菜（复习）

作为用户，我们写好`Flink`的程序，上管理平台提交，`Flink`就跑起来了(只要程序代码没有问题），细节对用户都是屏蔽的。

![图片](https://mmbiz.qpic.cn/sz_mmbiz_jpg/2BGWl1qPxib3hiahYr0d7EEfWUI7WOYphI7rjmY0GO9hcniaqvEJf05GHLiaQ4T86ZyK7XxguYYmMDs9tO6HbShAhg/640?wx_fmt=jpeg&wxfrom=13)

实际上大致的流程是这样的：

1. `Flink`会根据我们所写代码，会生成一个`StreamGraph`的图出来，来代表我们所写程序的拓扑结构。

1. 然后在提交的之前会将`StreamGraph`这个图**优化**一把(可以合并的任务进行合并），变成`JobGraph`

1. 将`JobGraph`提交给`JobManager`

1. `JobManager`收到之后`JobGraph`之后会根据`JobGraph`生成`ExecutionGraph`（`ExecutionGraph` 是 `JobGraph` 的并行化版本）

1. `TaskManager`接收到任务之后会将`ExecutionGraph`生成为真正的`物理执行图`

![图片](https://mmbiz.qpic.cn/sz_mmbiz_jpg/2BGWl1qPxib3hiahYr0d7EEfWUI7WOYphIse46CBCBhjd5HJzxiciaE5dj8mqmdoI3GKLZHOAEbrraCcIiaM074d4rQ/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

可以看到`物理执行图`真正运行在`TaskManager`上`Transform`和`Sink`之间都会有`ResultPartition`和`InputGate`这俩个组件，`ResultPartition`用来发送数据，而`InputGate`用来接收数据。

![图片](https://mmbiz.qpic.cn/sz_mmbiz_jpg/2BGWl1qPxib3hiahYr0d7EEfWUI7WOYphI8url99FLpptX4VMqydU8eGv2fWmoQ55BsXqGbl2LLNZjvjHDbkfXDg/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

屏蔽掉这些`Graph`，可以发现`Flink`的架构是：`Client`->`JobManager`->`TaskManager`

![图片](https://mmbiz.qpic.cn/sz_mmbiz_jpg/2BGWl1qPxib3hiahYr0d7EEfWUI7WOYphIrkmDDPK9LRDJorK7Hn1Ux1AlGbs0HRVjMB5LEsnsrWWnO805RfsmNQ/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

从名字就可以看出，`JobManager`是干「管理」，而`TaskManager`是真正干活的。回到我们今天的主题，`checkpoint`就是由`JobManager`发出。

![图片](https://mmbiz.qpic.cn/sz_mmbiz_jpg/2BGWl1qPxib3hiahYr0d7EEfWUI7WOYphIun19SrEkswCdwtuJYnPHbbyXCVBWLUeIOwoXIWGPibYJFTfslsW4YfA/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

`Flink`本身就是**有状态**的，`Flink`可以让你选择**执行过程中的数据**保存在哪里，目前有三个地方，在`Flink`的角度称作`State Backends`：

- _MemoryStateBackend_（内存）

- _FsStateBackend_（文件系统，一般是HSFS）

- _RocksDBStateBackend_（RocksDB数据库）

同样地，`checkpoint`信息就是保存在`State Backends`上

![图片](https://mmbiz.qpic.cn/sz_mmbiz_jpg/2BGWl1qPxib3hiahYr0d7EEfWUI7WOYphIg7o1dYV0XwfQh81Hiaf5dI1zb3vpkhIkPyrbrIb5tmrYSq7YibiahlkrA/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

先来简单描述一下`checkpoint`的实现流程：

`checkpoint`的实现大致就是插入`barrier`，每个`operator`收到`barrier`就上报给`JobManager`，等到所有的`operator`都上报了`barrier`，那`JobManager` 就去完成一次`checkpointi`

![图片](https://mmbiz.qpic.cn/sz_mmbiz_jpg/2BGWl1qPxib3hiahYr0d7EEfWUI7WOYphIDQHNkZYurlVYGf2yj4NZKkRDiajzMLkKIDswRCvJgRl8Be0yicE2cAvQ/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

因为`checkpoint`机制是`Flink`实现**容错机制**的关键，我们在实际使用中，往往都要配置`checkpoint`相关的配置，例如有以下的配置：

`final StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();   env.enableCheckpointing(5000);   env.getCheckpointConfig().setCheckpointingMode(CheckpointingMode.EXACTLY_ONCE);   env.getCheckpointConfig().setMinPauseBetweenCheckpoints(500);   env.getCheckpointConfig().setCheckpointTimeout(60000);   env.getCheckpointConfig().setMaxConcurrentCheckpoints(1);   env.getCheckpointConfig().enableExternalizedCheckpoints(CheckpointConfig.ExternalizedCheckpointCleanup.RETAIN_ON_CANCELLATION);   `

简单铺垫过后，我们就来撸源码了咯？

## Checkpoint（原理）

### JobManager发送checkpoint

从上面的图我们可以发现 `checkpoint`是由`JobManager`发出的，并且`JobManager`收到的是`JobGraph`，会将`JobGraph`转换成`ExecutionGraph`。

这块在`JobMaster`的构造器就能体现出来：

`public JobMaster(...) throws Exception {     // 创建ExecutionGraph     this.executionGraph = createAndRestoreExecutionGraph(jobManagerJobMetricGroup);    }   `

我们点击进去`createAndRestoreExecutionGraph`看下：

![图片](https://mmbiz.qpic.cn/sz_mmbiz_jpg/2BGWl1qPxib3hiahYr0d7EEfWUI7WOYphIsADrO9TAicrZxLm630OPSDhOeccGibWoaYPSJJDqcA2B3BOh6LnBol7A/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

看`CheckpointCoordinator`这个名字，就觉得他很重要，有木有？它从`ExecutionGraph`来，我们就进去`createExecutionGraph`里边看看呗。

点了两层`buildGraph()`方法，可以看到在方法的末尾处有`checkpoint`相关的信息：

`executionGraph.enableCheckpointing(       chkConfig.getCheckpointInterval(),       chkConfig.getCheckpointTimeout(),       chkConfig.getMinPauseBetweenCheckpoints(),       chkConfig.getMaxConcurrentCheckpoints(),       chkConfig.getCheckpointRetentionPolicy(),       triggerVertices,       ackVertices,       confirmVertices,       hooks,       checkpointIdCounter,       completedCheckpoints,       rootBackend,       checkpointStatsTracker);   `

前面的几个参数就是我们在配置`checkpoint`参数的时候指定的，而`triggerVertices/confirmVertices/ackVertices`我们溯源看了一下，在源码中注释也写得清清楚楚的。

`// collect the vertices that receive "trigger checkpoint" messages.   // currently, these are all the sources    List<JobVertexID> triggerVertices = new ArrayList<>();      // collect the vertices that need to acknowledge the checkpoint   // currently, these are all vertices   List<JobVertexID> ackVertices = new ArrayList<>(jobVertices.size());      // collect the vertices that receive "commit checkpoint" messages   // currently, these are all vertices   List<JobVertexID> commitVertices = new ArrayList<>(jobVertices.size());   `

下面还是进去`enableCheckpointing()`看看大致做了些什么吧：

`// 将上面的入参分别封装成ExecutionVertex数组   ExecutionVertex[] tasksToTrigger = collectExecutionVertices(verticesToTrigger);   ExecutionVertex[] tasksToWaitFor = collectExecutionVertices(verticesToWaitFor);   ExecutionVertex[] tasksToCommitTo = collectExecutionVertices(verticesToCommitTo);      // 创建触发器   checkpointStatsTracker = checkNotNull(statsTracker, "CheckpointStatsTracker");      // 创建checkpoint协调器   checkpointCoordinator = new CheckpointCoordinator(     jobInformation.getJobId(),     interval,     checkpointTimeout,     minPauseBetweenCheckpoints,     maxConcurrentCheckpoints,     retentionPolicy,     tasksToTrigger,     tasksToWaitFor,     tasksToCommitTo,     checkpointIDCounter,     checkpointStore,     checkpointStateBackend,     ioExecutor,     SharedStateRegistry.DEFAULT_FACTORY);      // 设置触发器   checkpointCoordinator.setCheckpointStatsTracker(checkpointStatsTracker);         // 状态变更监听器   // job status changes (running -> on, all other states -> off)   if (interval != Long.MAX_VALUE) {     registerJobStatusListener(checkpointCoordinator.createActivatorDeactivator());   }   `

值得一提的是，点进去`CheckpointCoordinator()`构造方法可以发现有状态后端`StateBackend`的身影（因为`checkpoint`就是保存在所配置的状态后端）

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

如果`Job`的状态变更了，`CheckpointCoordinatorDeActivator`是能监听到的。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

当我们的`Job`启动的时候，又简单看看`startCheckpointScheduler()`里边究竟做了些什么操作：

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

它会启动一个定时任务，我们具体看看定时任务具体做了些什么`ScheduledTrigger`，然后看到比较重要的方法：`triggerCheckpoint()`

这块代码的逻辑有点多，我们简单来总结一下

1. 前置检查（是否可以触发`checkpoint`，距离上一次checkpoint的间隔时间是否符合...)

1. 检查是否所有的需要做`checkpoint`的Task都处于`running`状态

1. 生成`checkpointId`，然后生成`PendingCheckpoint`对象来代表待处理的检查点

1. 注册一个定时任务，如果`checkpoint`超时后取消`checkpoint`

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

注：检查`task`的任务状态时，只会把`source`的`task`封装给进`Execution[]`数组

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

`JobManager`侧只会发给`source`的`task`发送`checkpoint`

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

### JobManager发送总结

贴的图有点多，最后再来简单总结一波，顺便画个流程图，你就会发现还是比较清晰的。

1. `JobManager` 收到`client`提交的`JobGraph`

1. `JobManger` 需要通过`JobGraph`生成`ExecutionGraph`

1. 在生成`ExcutionGraph`的过程中实际上就会触发`checkpoint`的逻辑

1. 定时任务会前置检查（其实就是你实际上配置的各种参数是否符合）

1. 判断`checkpoint`相关的`task`是否都是`running`状态，将`source`的任务封装到`Execution`数组中

1. 创建`checkpointID`/`checkpointStorageLocation`(checkpoint保存的地方)/`PendingCheckpoint`（待处理的checkpoint）

1. 创建定时任务（如果当`checkpoint`超时，会将相关状态清除，重新触发）

1. 真正触发`checkPoint`给`TaskManager`(只会发给`source`的`task`)

1. 找出所有`source`和需要`ack`的Task

1. **创建`checkpointCoordinator` 协调器**

1. 创建`CheckpointCoordinatorDeActivator`监听器，监听`Job`状态的变更

1. 当`Job`启动时，会触发`ScheduledTrigger` 定时任务

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

### TaskManager（source Task接收）

前面提到了，`JobManager` 在生成`ExcutionGraph`时，会给所有的`source` 任务发送`checkpoint`，那么`source`收到`barrier`又是怎么处理的呢？会到`TaskExecutor`这里进行处理。

`TaskExecutor`有个`triggerCheckpoint()`方法对接收到的`checkpoint`进行处理：

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

进入`triggerCheckpointBarrier()`看看：

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

再想点进去`triggerCheckpoint()`看实现时，我们会发现走到`performCheckpoint()`这个方法上：

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

从实现的注释我们可以很方便看出方法大概做了什么：

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

这块我们先在这里放着，知道`Source`的任务接收到`Checkpoint`会广播到下游，然后会做快照处理就好。

下面看看非`Source` 的任务接收到`checkpoint`是怎么处理的。

### TaskManager（非source Task接收）

在上一篇《[背压原理](https://mp.weixin.qq.com/s?__biz=MzI4Njg5MDA5NA==&mid=2247495581&idx=1&sn=f83ad7fe5d8c1d73d5d9153d3e89138d&chksm=ebd4ae9cdca3278a53b855ade2ed7fc510cddcb68d1e61893ece3d0094e81740de0c0278184d&token=44975441&lang=zh_CN&scene=21#wechat_redirect)》又或是这篇的基础铺垫上，其实我们可以看到在`Flink`接收数据用的是`InputGate`，所以我们还是回到`org.apache.flink.streaming.runtime.io.StreamInputProcessor#processInput`这个方法上

随后定位到处理数据的逻辑：

`final BufferOrEvent bufferOrEvent = barrierHandler.getNextNonBlocked();   `

想点击进去，发现有两个实现类：

- `BarrierBuffer`

- `BarrierTracker`

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

这两个实现类其实就是对应着`AT_LEAST_ONCE` 和`EXACTLY_ONCE`这两种模式。

`/**    * The BarrierTracker keeps track of what checkpoint barriers have been received from    * which input channels. Once it has observed all checkpoint barriers for a checkpoint ID,    * it notifies its listener of a completed checkpoint.    *    * <p>Unlike the {@link BarrierBuffer}, the BarrierTracker does not block the input    * channels that have sent barriers, so it cannot be used to gain "exactly-once" processing    * guarantees. It can, however, be used to gain "at least once" processing guarantees.    *    * <p>NOTE: This implementation strictly assumes that newer checkpoints have higher checkpoint IDs.    */      /**    * The barrier buffer is {@link CheckpointBarrierHandler} that blocks inputs with barriers until    * all inputs have received the barrier for a given checkpoint.    *    * <p>To avoid back-pressuring the input streams (which may cause distributed deadlocks), the    * BarrierBuffer continues receiving buffers from the blocked channels and stores them internally until    * the blocks are released.    */   `

简单翻译下就是：

- `BarrierTracker`是`at least once`模式，只要`inputChannel`接收到`barrier`，就直接通知完成处理`checkpoint`

- `BarrierBuffer`是`exactly-once`模式，当所有的`inputChannel`接收到`barrier`才通知完成处理`checkpoint`，如果有的`inputChannel`还没接收到`barrier`，那已接收到`barrier`的`inputChannel`会读数据到缓存中，直到所有的`inputChannel`都接收到`barrier`，这有可能会造成反压。

说白了，就是`BarrierBuffer`会有**对齐**`barrier`的处理。

这里又提到`exactly-once`和`at least once`了。在文章开头也说过`Flink`是可以实现`exactly-once`的，含义就是：状态**只持久化一次**到**最终**的存储介质中（本地数据库/HDFS)。

在这里我还是画个图和举个例子配合`BarrierBuffer`/`BarrierTracker`来解释一下。

现在我有一个`Topic`，假定这个`Topic`有两个分区`partition`（又或者你可以理解我设置消费的并行度是2）。现在要拉取`Kafka`这两个分区的数据，由算子`Map`进行消费转换，期间在转化的时候可能会存储些信息到`State`(`Flink`给我们提供的存储，你就当做是会存到`HDFS`上就好了)，最终输出到`Sink`。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

从上面的知识点我们应该可以知道， 在`Flink`做`checkpoint`的时候`JobManager`往每个`Source`任务（简单对应图上的两个`paritiion`） 发送`checkpointId`，然后做快照存储。

显然，`source`任务存储最主要的内容就是消费分区的`offset`嘛。比如现在`source 1`的`offerset`是`100`，而`source2`的`offset`是`105`。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

目前看来`source2`的数据会比`source1`的数据先到达`Map`

假定我们用的是`BarrierBuffer` `exactly-once`模式，那么`source2`的`barrier`到达`Map`算子的后，`source2`之后的数据只能停下来，放到`buffer`上，不做处理。等`source1` 的`barrier`来了以后，再真正处理`source2`放在`buffer`的数据。

这就是所谓的`barrier`对齐

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

假定我们用的是`BarrierTracker` `at least once`模式，那么`source2`的`barrier`到达`Map`算子的后，`source2`之后的数据不会停下来等待`source1`，后面的数据会继续处理。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

现在问题就来了，那**对不对齐的区别**是什么呢？

依照上面图的的运行状态（无论是`BarrierTracker` `at least once`模式还是`BarrierBuffer` `exactly-once`模式），现在我们的`checkpoint`都没做，因为`source1`的`barrier`还没到`sink`端呢。现在`Flink`挂了，那显然会重新拉取`source 1`的`offerset`是**小于**`100`，而`source2`的`offset`是**小于**`105`的数据，`State`的最终信息也不会保存。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

`checkpoint`从没做过的时候，对数据不会产生任何的影响（所以这里在`Flink`的内部是没啥区别的）

而假设我们现在是`BarrierTracker` `at least once`模式，没有任何问题，程序继续执行。现在`source1`的`barrier`也走到了`slink`，最后完成了一次`checkpoint`。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

由于`source2`的`barrier`比`source1`的`barrier`要快，那么`source1`所处理的`State`的数据实际是包括`offset>105`的数据的，自然`Flink`保存的时候也会把这部分保存进去。

程序继续运行，刚好保存完`checkpoint`后，此时系统出了问题，挂了。因为`checkpoint`已经做完了，所以`Flink`会从`source 1`的`offerset`是`100`，而`source2`的`offset`是`105`重新消费。

但是，由于我们是`BarrierTracker` `at least once`模式，所以`State`里边的保存状态实际上有过`source2`的`offset` 大于`105` 的记录了。那`source2`重新从`offset`是`105`开始消费，那就是会重复消费！

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

理解了上面所讲的话，那再看`BarrierBuffer` `exactly-once`模式应该就不难理解了（**各位大哥大嫂你也要经过这个`operator`处理保存吗？我们一起吧？有问题，我们一起重来，没问题我们一起保存**）

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

无论是`BarrierTracker`还是`BarrierBuffer`也好，在处理`checkpoint`的时候都需要调用`notifyCheckpoint()` 方法，而`notifyCheckpoint()`方法最终调用的是`triggerCheckpointOnBarrier`

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

`triggerCheckpointOnBarrier()`最终还是会调用`performCheckpoint()`方法，所以无论是`source`接收到`checkpoint`还是`operator`接收到`checkpoint`，最终还是会调用`performCheckpoint()`方法。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

大家有兴趣可以进去`checkpointState()`方法里边详细看看，里边会对`State`状态信息进行写入，完成后上报给`TaskManager`

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

### TaskManager总结

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

- `TaskExecutor`接收到`JobManager`下发的`checkpoint`，由`triggerCheckpoint`方法进行处理

- `triggerCheckpoint`方法最终会调用`org.apache.flink.streaming.runtime.tasks.StreamTask#triggerCheckpoint`，而最主要的就是`performCheckpoint`方法

- `performCheckpoint`方法会对`checkpoint`做前置处理，`barrier`广播到下游，处理`State`状态**做快照**，最后回到成功消息给`JobManager`

- 普通算子由`org.apache.flink.streaming.runtime.io.StreamInputProcessor#processInput`这个方法读取数据，具体处理逻辑在`getNextNonBlocked`方法上。

- 该方法有两个实例，分别是`BarrierBuffer`和`BarrierTracker`，这两个实例对应着`checkpoint`不同的模式（至少一次和精确一次）。精确一次需要对`barrier`对齐，有可能导致反压的情况

- 最后处理完，会调用`notifyCheckpoint`方法，实际上还是会调`performCheckpoint`方法

所以说，最终处理`checkpoint`的逻辑是一致的，只是会`source`会直接通过`TaskExecutor`处理，而普通算子会根据不同的配置在接收到后有不同的实例处理：`BarrierTracker`/`BarrierBuffer`。

### JobManager接收回应

前面提到了，无论是`source`还是普通算子，都会调用`performCheckpoint`方法进行处理。

`performCheckpoint`方法里边处理完`State`快照的逻辑，会调用`reportCompletedSnapshotStates`告诉`JobManager`快照已经处理完了。

`reportCompletedSnapshotStates`方法里边又会调用`acknowledgeCheckpoint`方法通过`RPC`去通知`JobManager`

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

兜兜转转，最后还是会回到`checkpointCoordinator`上，调用`receiveAcknowledgeMessage`进行处理

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

进入到`receiveAcknowledgeMessage`上，主要就是下面图的逻辑：处理完返回不同的状态，根据不同的状态进行处理

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

主要我们看的其实就是`acknowledgeTask`方法里边做了些什么。

在 `PendingCheckpoint`维护了两个Map：

`//  已经接收到 Ack 的算子的状态句柄   private final Map<OperatorID, OperatorState> operatorStates;      // 需要 Ack 但还没有接收到的 Task   private final Map<ExecutionAttemptID, ExecutionVertex> notYetAcknowledgedTasks;   `

然后我们进去`acknowledgeTask`简单了解一下可以发现就是在处理`operatorStates`和`notYetAcknowledgedTasks`

`synchronized (lock) {      if (discarded) {       return TaskAcknowledgeResult.DISCARDED;      }             // 接收到Task了，从notYetAcknowledgedTasks移除      final ExecutionVertex vertex = notYetAcknowledgedTasks.remove(executionAttemptId);         if (vertex == null) {       if (acknowledgedTasks.contains(executionAttemptId)) {        return TaskAcknowledgeResult.DUPLICATE;       } else {        return TaskAcknowledgeResult.UNKNOWN;       }      } else {       acknowledgedTasks.add(executionAttemptId);      }             // ...      if (operatorSubtaskStates != null) {       for (OperatorID operatorID : operatorIDs) {           // ...        OperatorState operatorState = operatorStates.get(operatorID);        // 新来的operatorID，添加到operatorStates        if (operatorState == null) {         operatorState = new OperatorState(          operatorID,          vertex.getTotalNumberOfParallelSubtasks(),          vertex.getMaxParallelism());         operatorStates.put(operatorID, operatorState);        }             //....       }      }   `

等到所有的`Task`都到齐以后，就会调用`isFullyAcknowledged`进行处理。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

最后调用`completedCheckpoint = pendingCheckpoint.finalizeCheckpoint();`来实现最终的存储，所有完毕以后会通知所有的`Task` 现在`checkpoint`已经完成了。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

## 最后

总的来说，这篇文章带着大家走马观花撸了下`Checkpoint`，很多细节我也没去深入，但我认为这篇文章可以让你大概了解到`Checkpoint`的实现过程。

最后再来看看官网的图，看完应该大概就能看得懂啦：

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

相信我，或许你现在还没用到`Flink`，但等你真正去用`Flink`的时候，`checkpoint`是肯定得搞搞的（：现在可能有的同学还没看懂，没关系，先点个赞👍，收藏起来，后面就用得上了。

参考资料：

- https://ci.apache.org/projects/flink/flink-docs-release-1.12/concepts/stateful-stream-processing.html

- https://blog.csdn.net/weixin_40809627/category_9631155.html

- https://www.jianshu.com/p/4d31d6cddc99

- https://www.jianshu.com/p/d2fb32ba2c9b

三歪老婆会了吗？

阅读 3134

​

写留言

**留言 4**

- #FFF

  2021年11月12日

  赞

  们???

- 晚

  2021年11月12日

  赞

  这个女朋友（们）用的非常好

- Sonder

  2021年11月12日

  赞

  3y是我的![[嘿哈]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![[嘿哈]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![[嘿哈]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

- FreeSoul

  2021年11月12日

  赞

  只要丙你会就行呀…… 要不然3y找不到请教问题的机会去靠近你![[旺柴]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

已无更多数据

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/uChmeeX1Fpzoaxam21AeaqcNGgib2gF333O0cQP4ZWX2M1A0QKaOzMuDsomwbYpfkrQOvXbb3yOx2XeRdYGBLSw/300?wx_fmt=png&wxfrom=18)

敖丙

关注

633

4

写留言

**留言 4**

- #FFF

  2021年11月12日

  赞

  们???

- 晚

  2021年11月12日

  赞

  这个女朋友（们）用的非常好

- Sonder

  2021年11月12日

  赞

  3y是我的![[嘿哈]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![[嘿哈]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![[嘿哈]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

- FreeSoul

  2021年11月12日

  赞

  只要丙你会就行呀…… 要不然3y找不到请教问题的机会去靠近你![[旺柴]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

已无更多数据
