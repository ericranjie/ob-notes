原创 向鹿 阿里云开发者

_2024年04月25日 08:31_ _浙江_

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/Z6bicxIx5naIZaPcibbv0y2BsSzFeERVt2TNbBXfyTfMVCFs1k2ylyPzBLXKdkfgyWPtYTJyTicO9qpnn8JgibSakQ/640?wx_fmt=jpeg&from=appmsg&wxfrom=13&tp=wxpic)

阿里妹导读

一条SQL语句的执行究竟经历了哪些过程？作者作为一个刚入职的大数据研发新人对SQL任务执行整个流程进行了整理，本文就作者学习内容和体会供大家参考。

作为一个刚刚入职的大数据萌新研发，我对SQL任务执行整个流程充满好奇，一条SQL语句的执行究竟经历了哪些过程？在查阅了相关文档之后，我整理得到了这篇文档，在此分享我的学习内容和体会供大家参考。

一、整体流程

一个SQL任务从建立到得到运行结果，中间涉及到了多个系统间的交互，这里我们先给出一张整体的流程图，接下来就让我们从一个简单的小需求—统计活动中每个奖品的发放数量，开始我们的探索之旅吧。

﻿﻿![图片](https://mmbiz.qpic.cn/mmbiz_png/Z6bicxIx5naIZaPcibbv0y2BsSzFeERVt2vuYxMS7LqnicficgwhUycmzBxFjRBDOmuia3KHaYCSe81fuVt2zH4PlQw/640?wx_fmt=png&from=appmsg&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

二、任务开发上线

**新建&编辑任务**

首先，我们需要在IDE中创建并编写一个SQL任务，在DataPhin研发页面新建离线周期计算任务，编写sql代码。

统计每个活动奖品数量的SQL：

```
SELECT  prize_id
```

**配置调度信息**

在编写完SQL任务之后，我们需要配置任务的调度信息。这些调度信息被视为任务的元数据，并通过数据库进行维护。只有在配置了正确的调度信息之后，任务才能被正确地调度和执行。常见的SQL任务配置如下表所示：

|   |   |
|---|---|
|基本信息|任务节点id、任务名称、节点类型、运维负责人|
|调度参数|Dataphin任务调度时使用的参数，根据任务调度的业务日期、定时时间及参数的取值格式自动替换为具体的值，实现在任务调度时间内参数的动态替换。<br><br>业务日期：调度日期前一天<br><br>调度/定时时间：任务实例计划运行的时间|
|调度属性|实例生成方式：T+1次日生成、发布后即时生成|
|调度类型：正常调度、暂停调度、空跑调度|
|生效日期：自动调度运行的日期范围|
|重跑属性：多次重跑是否会影响结果|
|调度周期：按照规律间隔多久执行一次该任务，可分为月调度、周调度、日调度、小时调度和分钟调度|
|cron表达式：对应调度周期cron表达式|
|调度依赖|节点间的上下游依赖关系，上游任务节点运行完成且运行成功，下游任务节点才会开始运行。|
|节点上下文参数|输入参数（接收上游节点的输出参数值作为当前节点的输入）、输出参数|
|执行信息|执行引擎、调度资源组|

**提交发布**

在完成SQL任务的编写和调度信息的配置后，我们可以提交任务并生成发布包。在进行发布之前，我们可以在开发环境中进行冒烟测试。冒烟测试会生成一个包含代码和调度信息的实例。选择昨日之前生成的实例，其调度时间已经满足，任务会立即运行。选择昨日生成的实例需要等待调度时间到达才会运行。任务提交并且冒烟测试通过后，我们可以进入发布中心中，选中待发布的任务对象，进行发布。

三、转实例/实例生产

现在任务已经发布到线上了，后面会发生什么呢？

时间来到晚上22:00，此时Phoenix调度系统开始忙碌起来了，它根据大家发布的任务节点定义，提前编译生成了一批可执行的任务实例，并且根据任务之间的血缘依赖和时间依赖，将这一批任务实例组成DAG去调度执行。转实例时，还会通过系统内置的解析函数解析任务配置的cron时间表达式，为对应任务实例设置具体的运行时间。

﻿﻿!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

四、定时调度/实例启动

经过上一步转实例的过程，对应的任务实例DAG也就绪了。那么接下来怎么在正确的时间，将任务实例唤起执行呢？

早期的实例调度系统是使用Quartz框架实现的，通过注册Quartz事件，并配置触发器的cron时间表达式来指定事件的触发时间。Quartz框架会根据时间表达式，在预定的时间点触发相应的事件，从而启动对应的任务实例。由于Quartz框架在实例数量达到百万量级的时候会出现性能瓶颈，新的调度引擎采用的数据库维护状态+异步查询的自研方案。具体来说，就是通过在数据库维护所有任务实例的执行时间，而后调度系统的后台线程**异步定时查询**满足时间的任务实例，触发启动执行。

五、调度资源分配

# 唤起任务实例之后，我们需要给实例分配资源，提交给ODPS去执行，这部分逻辑是由执行引擎Alisa来完成的。

Alisa拿到任务后，基于任务资源组资源是否空闲、调度任务执行集群资源是否空闲将任务下发到某台gateway上提交。gateway是Alisa集群中负责提交ODPS任务和管理任务状态的进程，由于任务要提交到计算引擎去执行，在提交任务到任务运行的过程中会一直保持一个会话连接，需要占用一个槽位（slot），槽位其实是指提交任务的能力。为了保障重要业务优先使用资源，以及满足多用户、多租户资源使用需求，设计了gateway执行集群和资源组模式来控制任务使用的槽位。一个集群可以包含多台gateway机器，集群的槽位可以分配给多个资源组使用。

﻿﻿!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

通过将项目空间设置到某个资源组，这个项目空间提交的任务就能够使用这个资源组的槽位，每个资源组通过设置最大槽位(max_slot)来控制并发，可通过tesla实时查看项目空间的槽位使用情况。

﻿﻿!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

Alisa执行引擎负责管理slot使用情况，并且实时的和每台gateway机器保持一个心跳连接，负责给任务分配槽位。若调度槽位打满，那么作业处于拥塞状态，等待分配槽位。当任务分配到槽位后，就会在Alisa指定的gateway机器上开始提交任务，即在gateway上启动odpscmd进程，通过odpscmd提交作业。

六、ODPS运行作业

任务通过gateway进程使用odpscmd的方式提交作业之后，正式进入的ODPS系统的执行过程。

ODPS系统可以分为控制层和计算层两个部分，提交的作业会先进入odps的控制层，控制层包括请求处理器Worker、调度器Scheduler和作业执行管理器Executor。作业提交生成instance后，Scheduler会负责instance的调度。Scheduler首先维护一个instance列表，然后从列表中取出instance，把Instance分解成多个Task，把可运行的Task放到优先级队列TaskPool中，进入控制集群排队。Scheduler的一个后台线程会对优先级队列TaskPool定时排序，另一个后台线程会定时查询计算集群的资源情况。Executor主动轮询Scheduler，Scheduler判断如果控制集群还有资源，就把排序第一的SQLTask发送给Executor。

﻿﻿!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

Executor在拿到任务之后，会对SQL调用SQL Parse Planner词法语法分析器，经过词法分析、语法分析后得到抽象语法树（AST），然后经过逻辑分析后得到优化后的逻辑执行计划，再经过物理分析后得到优化后的物理执行计划。

﻿﻿!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

物理执行计划中按照数据处理过程中是否需要shuffle划分，得到物理算子DAG图，其中一个节点对应一个Fuxi Task，根据DAG图和instance的元信息（每个Fuxi Task的实例个数、资源规格等），生成Fuxi Job的配置文件，提交给Fuxi Master。

七、Fuxi计算层

伏羲是大数据计算平台的分布式调度执行框架，支撑着ODPS以及PAI等多种计算引擎每天上千万分布式作业的执行，用于处理每天EB级别的大量数据。伏羲集群由Master和Agent组成，Agent分布在每个计算节点上，负责单个计算节点的资源管理，并将节点信息汇报给Master，Master收集每个节点的资源使用情况，统一管理和协调计算资源的分配。

﻿﻿!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

ODPS的任务经过处理变成Fuxi Task的描述文件被提交给Fuxi Master处理，Fuxi Master会选择一个空闲节点启动该任务的Application Master进程。Application Master启动之后会根据任务的配置信息向Fuxi Master申请计算资源，后续该任务的资源申请都由Application Master去完成。

Fuxi Master收到资源请求以后，若集群quota被打满，资源紧张，则会在计算集群上排队，等待分配资源。若资源有空闲，会根据Application所属的quota组、Job的优先级、是否打开抢占等条件，找到适合的资源（可能只能满足资源请求的一部分）分配给Application Master，通知Application Master进程，并将分配情况告知资源所在节点的Fuxi Agent。如果请求未被完全满足时，Fuxi Master会在有其他可分配空闲资源时继续分配资源给该Application Master。

﻿﻿!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

Application Master收到资源分配的消息后，会将具体的作业计划发送给相应的Fuxi Agent，作业计划包含要启动具体的进程所需要的信息，可执行文件名，启动参数，资源(比如cpu/mem)限制等。

Fuxi Agent在接到Application Master的作业计划后，会启动worker作业进程。worker启动以后，会和Application Master通信，从分布式存储中读取数据并开始执行计算逻辑。随着计算任务的执行，会不断有worker被拉起和释放，当worker完成所有计算任务之后，会将结果写入到对应的文件夹下，向Application Master汇报任务已成功完成，由Application Master向Fuxi Master通信完成资源释放。

至此，整个ODPS任务执行完成，各个组件依次更新任务状态。这个SQL任务的实例被置为成功，完成了其光荣的使命，等待着调度引擎的下一次唤醒。

阅读 1.5万

​

写留言

**留言 2**

- 姜明松

  上海4月25日

  赞26

  简单却体系化 以小见大的文章

- 孙志计

  上海4月26日

  赞2

  厉害![[强]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![[强]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![[强]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

已无更多数据

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/Z6bicxIx5naI1jwOfnA1w4PL2LhwNia76vBRfzqaQVVVlqiaLjmWYQXHsn1FqBHhuGVcxEHjxE9tibBFBjcB352fhQ/300?wx_fmt=png&wxfrom=18)

阿里云开发者

11859419

2

写留言

**留言 2**

- 姜明松

  上海4月25日

  赞26

  简单却体系化 以小见大的文章

- 孙志计

  上海4月26日

  赞2

  厉害![[强]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![[强]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![[强]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

已无更多数据
