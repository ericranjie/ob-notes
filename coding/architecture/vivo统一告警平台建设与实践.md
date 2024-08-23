# 

高可用架构

 _2021年11月19日 09:10_

以下文章来源于vivo互联网技术 ，作者Chen Ningning

[

![](http://wx.qlogo.cn/mmhead/Q3auHgzwzM5gDg6UckibibLvqjUtNVo6qqOeiaYc07q5N7pm5GRwvRQxg/0)

**vivo互联网技术**.

分享 vivo 互联网技术干货与沙龙活动，推荐最新行业动态与热门会议。

](https://mp.weixin.qq.com/s?__biz=MzAwMDU1MTE1OQ==&mid=2653557750&idx=1&sn=d27b9b702a3bbdfa58d686ea30fbe14f&chksm=8139826eb64e0b7816434d4f04a6e5756cb53a64edb030bc31284e42d7d6f04d17ec37d3486f&mpshare=1&scene=24&srcid=1119XomI1jKtzcsRqUvfE93S&sharer_sharetime=1637297349994&sharer_shareid=5fb9813bfe9ffc983435bfc8d8c5e9ca&key=daf9bdc5abc4e8d014f6011458c5f41913d80bd6cc2eb3f29fae05551f65f4ab3def2077693105d8857e507d3ad7a48e28591780b89350c6ba45660978df78788c323ad9c7e3b1edeffb25cfe01822b4964bac201c4aeab9a66dda9bda1004bea08e7a3bfe0ab7ee4e0cd2368d56b784d51fcd427f7104fc283cce2ff3b8adb4&ascene=0&uin=MTEwNTU1MjgwMw%3D%3D&devicetype=Windows+11+x64&version=63090b19&lang=zh_CN&countrycode=CN&exportkey=n_ChQIAhIQaa5S6H0LQj%2B7zCmceL7aHhLmAQIE97dBBAEAAAAAAFaKOAWG7w8AAAAOpnltbLcz9gKNyK89dVj0FGF6nz26xAaZbhlCMVPMNnq2Z3uB20iMud2BQEUv92o%2FzxqOPE6%2BlINIUANKmSmKefK9FkKm6GDLua6ivXRuFB7f6qF1jHwNzLy3QyJgflso9pOJ6UhFE5QeDluu1pdL54B%2Flnbp%2BsdSRj70v%2BVeYahOkim6lHyA0IEt6K%2FbYtPj5PycfLeMbWlUgW0g8nvgssEFoMEXFOxHMImxlD94VL5sDE94%2Fj66Sr0Heo0%2FXt07t6iZJr3jxjCAASl7AChb&acctmode=0&pass_ticket=4oEmee5amu%2F9AiCwitPgObkH8ZC4daf2X0bxg%2Fv8phiMp%2FnTTdlgYdyezes14Exr&wx_header=1&fasttmpl_type=0&fasttmpl_fullversion=7351805-zh_CN-zip&fasttmpl_flag=1#)

> 作者：vivo互联网服务器团队-Chen Ningning

一、背景

  

一套监控系统检测和告警是密不可分的，检测用来发现异常，告警用来将问题信息发送给相应的人。vivo监控系统1.0时代各个监控系统分别维护一套计算、存储、检测、告警收敛逻辑，这种架构下对底层数据融合非常不利，也就无法实现监控系统更广泛场景的应用，所以需要进行整体规划，重新对整个监控系统架构进行调整，在这样的背景下统一监控的目标被确立。

  

以前监控被划分为基础监控、通用监控、调用链、日志监控、拨测监控等几大系统，统一监控的目标是将各个监控指标数据进行统一计算、统一存储、统一检测、统一告警、统一展示。这里不作赘述，后面会出一期vivo监控系统演进的文章进一步说明。

  

上面我们说了监控统一的大背景。以前各个监控系统会各自进行告警收敛、消息组装等工作，为了减少冗余，需要将收敛等工作由一个服务统一做处理，与此同时告警中心平台也到了更新迭代的阶段，这样就需要建设一个对内部各业务统一提供告警收敛、消息组装、告警发送的告警平台，有了这个构想，我们准备将各系统告警收敛能力与告警发送能力下沉，将统一告警服务做成一个与各监控服务解偶的通用服务。

二、现状分析

  

对于1.0时代的监控系统来说，如图1所示各个监控系统要先进行告警收敛，然后分别和老的告警中心进行对接，才能将告警消息发送出来。每一套系统都要单独进行维护一套规则，有很多重复功能建设，而实际这些功能具有高度通用性，完全可以建立合理模型对异常检测服务生成的异常进行统一处理，从而生成问题，然后进行统一的消息组装，最后发送告警消息。

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

 (图1 老监控系统告警流程图）

  

在监控系统中一个异常从被检测出来到最终发出告警有几个重要概念：

  

**异常**

在一个检测窗口(窗口大小可以自定义)，一个或几个指标值达到检测规则定义的异常阈值，就产生一个异常。如图2所示，检测规则定义当指标值在一个检测窗口为6的检测周期内，有3个数据点超过阈值就认为有异常，我们简称这个检测规则为6-3，如图所示第一个检测窗口内(蓝色虚线筐内)只有6和7两个点的指标值超过阈值(95)，不满足6-3的条件，所以第一个检测窗口没有异常。在第二个检测窗口内(绿色虚线框内)有6、7、8三个点的指标值超过阈值(95)，所以第二个窗口就是一个异常。

  

**问题**

一个连续的周期内产生的所有同类异常的集合，我们称之为问题。如图2所示，第二个检测窗口就是一个异常，同时这个异常也会对应有一个问题A，如果第三个检测窗口也是一个异常，那么这个异常对应的问题也是A，所以问题和异常是一对多的关系。

  

**告警**

当一个问题通过告警系统将消息以短信、电话、邮件等方式告知给用户时，我们称之为一条告警。

  

**恢复**

当问题对应的异常不满足检测规则定义的异常条件时，就认为所有异常都恢复了，同时问题也认为恢复了，恢复也会发送相应的恢复通知。

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

 (图2 时序数据异常检测原理图）

  

三、衡量指标

  

一个系统我们如何衡量它的好坏，如何提升它，如何管理它？管理学大师彼得·德鲁克曾说“你如果无法度量它，就无法管理它(If you can’t measure it, you can’t manage it)”。从这里可以看出，如果想全面管理提升一个系统，就需要先对它的各项性能指标有一个衡量，知道它的薄弱点在哪里，找到病症所在才能对症下药。

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

 （图3 运维指标时间节点关系图）

  

图3是监控系统运营指标和对应时间节点关系图，主要体现了MTTD、MTTA、MTTF、MTTR、MTBF等指标与时间节点的对应关系，这些指标对于提升系统性能，帮助运维团队及早发现问题有很高的参考价值。业界有很多云告警平台也很注重这些指标，下面我们着重介绍一下MTTA、MTTR这两个和告警平台关系紧密的指标：

  

**MTTA**(Mean time to acknowledge,平均应答时间)：

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

（图4 MTTA计算公式）

  

- t[i] -- 监控系统运行期间第i个服务出现问题后运维团队或者研发人员响应问题的时间；
    
- r[i] -- 监控系统运行期间第i个服务出现问题的总次数。
    

  

平均应答时间是运维团队或者研发团队响应所有问题所花费的平均时间。MTTA度量标准用于衡量运维团队或研发团队的响应能力和警报系统的效率。通过跟踪和最小化MTTA，项目管理团队可以优化流程，提高问题解决效率，保障服务可用性，提升用户满意度[1]。

  

**MTTR**(Mean Time To Repair,平均维修时间)：

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

（图5 MTTR计算公式[2]）

  

- t[ri] -- 监控系统运行期间第i个服务出现r次告警后服务恢复正常状态的总时间
    
- r[i] -- 监控系统运行期间第i个服务出现告警的总次数
    

  

平均修复时间（MTTR）是修复系统并将其恢复到正常功能所需的平均时间。运维或研发人员开始处理异常，MTTR便开始计算，并且一直进行到被中断的服务完全恢复（包括所需的任何测试时间）为止。在IT服务管理行业中，MTTR中的R并不总是表示维修。它也可以表示恢复，响应或解决。尽管这些指标都对应MTTR，但是它们都有各自的含义，因此，要弄清楚要使用哪个MTTR，有助于我们更好的分析理解问题。让我们简要地看一下它们各自的含义：

> **1）平均恢复时间**(Mean time to recovery)是从系统告警中恢复所需的平均时间。这涵盖了从服务异常导致告警到恢复正常的整个过程。MTTR是衡量整个恢复过程速度的指标。

> **2）平均响应时间**(Mean time to respond)表示从出现第一个告警开始到系统从故障中恢复到正常状态所需的平均时间，不包括告警系统中的任何延迟。该MTTR通常用于网络安全中，以衡量团队缓解系统攻击的效率。

> **3）平均解决时间**(Mean time to resolve)表示完全解决系统故障所花费的平均时间，包括检测故障、诊断问题以及确保故障不再发生来解决问题所需的时间。此 MTTR 指标主要用于衡量不可预见事件的解决过程，而不是服务请求。

  

提升 MTTA 的核心是找对人、找到人[3]，只有在最短的时间内找对能处理问题的人才能有效提升MTTR。通常在生产实践过程中我们会遇到“告警泛滥”的问题，大量的告警出现时需要运维人员或者开发同学去解决，对于应激敏感的同学来说很容易出现“狼来了”的现象，只要收到告警就会很紧张，同时当大量的告警信息频发骚扰我们运维人员，会引发告警疲劳，体现为不重要的事件太多，最根本的问题较少，频繁处理普通事件，重要的信息淹没在汪洋大海中。[4]

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

（图6 告警泛滥问题图[5]）

  

四、功能设计

  

通过上面两个重要指标的分析，我们总结出要从**告警数量**、**告警收敛**、**告警升级**等方面着手，减少告警发送的数量，提升告警准确性，最终提升解决问题的效率，降低问题恢复时长。下面我们从系统和功能层面说明如何降低告警量，把真正有价值的告警信息发送到用户手中。本文也将重点围绕告警消息收敛进行讲解。

  

从图1中可以看出各个监控系统中都有很多重复的功能模块，所以针对这些功能模块我们可以将其抽离出来，如图7所示将告警收敛、告警屏蔽、告警升级等能力统一建设在统一告警服务中。这种架构下统一告警服务与检测相关服务完全解耦，在能力上具有一定的通用性。例如其它有告警或消息收敛需求的业务团队想接入统一告警，统一告警要能满足消息收敛发送的需求，同时也要满足消息直接发送的需求。统一告警会提供灵活可配置的消息发送方式，提供简单、多样的功能满足各类需求。

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

（图7 统一告警系统结构图）

  

4.1 告警收敛

  

对于告警平台每天会产生数以万计的告警，这些告警对于运维或开发人员都需要去分析、甄别优先级、并处理故障。数以万计的告警如果不加收敛每条异常都发送告警，势必会增大运维人员的工作压力，当然也不是所有的告警都需要并且有必要发送给运维人员进行处理。所以我们需要对告警通过多种手段进行收敛，下面我们从四个方面介绍一下如何进行告警收敛。

  

**首次告警等待**

当一个异常产生之后我们不会立即去做告警，而是通过等待一段时间才会去做告警发送，一般这个时间可以通过系统自定义，这个值如果太大就会影响告警延迟，太小不能提升告警合并效果。例如首次告警等待时间为5s，当一个服务下节点1出现A指标异常，5s内节点2也出现了A指标异常，那么发送告警时节点1和节点2会被合并到一起发送告警通知。

  

**告警间隔**

问题在没有恢复前，系统会根据告警间隔的配置每隔一段时间发送一条告警信息，告警间隔用来控制告警发送的频率。

  

**异常收敛维度**

异常收敛维度用来将同个维度下的异常合并在一起。例如同个节点路径A下，通过同一个检测规则产生的异常，会在告警发送的时候根据配置的异常收敛维度合并在一起。

  

**消息合并维度**

当多个异常收敛成一个问题，在发送告警的时候会涉及到消息合并，消息合并维度就是用来指定哪些维度可以合并。可能这样理解有些晦涩，我们可以通过图8看一下从异常到消息的转换过程。

  

假如一个异常有两个维度名字和性别，当这两个异常经过统一告警，我们会根据配置的收敛策略进行合并，从图中我们可以看到性别被定义为异常收敛维度，通常异常收敛维度的选择一定是两个或两个以上具有相同的属性的异常，这样在消息合并后只取相同属性的同一个值，对应到示例图，我们会将${sex}占位符替换成男。而名字是被定义为告警合并维度，就表示所有异常中名字是都要展示在消息文本中，所以在消息合并的时候我们会将${name}占位符对应的信息一一拼接在消息文本中。

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

（图8 消息文本替换示意图 ）

  

4.2 告警认领

  

当出现告警后如果有人认领了该告警，那么后续相同告警只会发送给告警认领人。告警认领主要是为了解决告警有人跟进后，减少将告警发给其他人员，也能从一定程度上解决告警被重复处理的问题。被认领的告警可以取消认领。

  

4.3 告警屏蔽

  

对于同一个问题，可以设置告警屏蔽，后续如果有该问题对应的告警产生，那么将不会被发送出去。告警屏蔽能减少故障在定位解决过程中，或者服务在发版变更过程中造成的告警，能有效减少无效告警对运维人员造成的困扰，屏蔽可以设置为周期性的，也可以设置为屏蔽某一时段，当然也可以取消屏蔽。

  

4.4 告警回调

  

当告警规则配置了回调，那么当产生告警，就会调用回调接口，使服务或业务恢复正常。告警回调的目的是当某个服务有告警产生，希望系统能够通过一些自动化的配置，使服务恢复到正常状态，缩短故障恢复的时间，也能够紧急情况下第一时间快速恢复服务。

  

4.5 误告标注

  

对于一个问题，用户可以通过误告标注备注该异常是否为误告警。误告标注的主要目的是通过标注让系统开发人员知道异常检测过程中，哪写点还需要提升优化，提高告警的准确性，为用户提供真实有效的告警提供保障。

  

4.6 告警升级

  

当告警发生一定时间仍没有恢复，那么系统就会根据配置自动进行告警升级处理，然后将告警升级信息通过配置发送给对应的人员。告警升级一定程度上是为了缩短MTTA，当告警长时间未恢复，可以认为故障没有及时得到响应，这时就需要更高级别的人员介入处理。

  

如图9所示，每天告警系统会发送大量的告警，当然这些告警会分别发送给不同服务的告警接收人。告警并不是越多越好，而是应该第一时间准确反映出服务的异常情况，所以如何提升有效告警，提高告警准确性，减少告警量至关重要。通过以上系统设计和功能设计能够有效减少重复告警发送。

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

（图9 主机监控告警次数图）  

  

五、架构设计

  

上面我们从系统和功能层名讲解了如何针对老架构下存在的各种问题进行解决，那么对于这个构想我们应该用一套什么架构来实现。

  

下面我们看下如何设计这套架构。统一告警作为整个监控最后一环，既要满足告警发送能力也要满足业务服务发送通知的需求，所以统一告警的各种能力要具有通用性。统一告警服务要做到与其它服务低耦合，尤其是与已有监控系统做到解偶，这样才能真正把通用能力释放出来。服务要能做到根据业务场景的不同适配不同的业务逻辑，比如有的业务需要做告警收敛，有的业务不需要，那么服务要提供灵活的接入方式以适用业务需要。

  

如图10所示，统一告警核心逻辑由收敛服务实现，收敛服务可以消费kafka中的异常，也可以通过RestFul接口接收推送过来的异常，异常会先经过异常处理生成一个问题，然后将问题和异常存入MySql库，经过告警收敛模块问题会被推送到Redis延时队列中，延时队列会用来控制消息出队时间，消息从队列取出之后会进行文本组装等操作，最后会通过配置发送出去。

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

（图10 统一告警架构图）

  

配置管理服务用来管理应用、事件、告警等配置信息，元数据同步服务用来从其它服务同步告警收敛所需的元数据。

  

六、核心实现

  

统一告警的核心是告警收敛，收敛的目的就是减少发送重复的告警消息，避免因为大量的告警对于告警接收人造成告警麻痹。

  

前文已经说到使用延时队列做告警收敛，延时队列在电商和支付项目中使用较多，比如商品下单后10分钟未支付就要自动取消订单。在告警系统中使用延时队列主要目的是，在一定的时间内尽可能多的将同一个问题对应的异常合并在一起，减少告警发送的数量。举个例，如果一个服务A有三个节点，当发生异常时，一般情况下每个节点的异常都会生成一条告警发送出去，但是经过告警收敛时候我们可以将三个节点的告警合并，由一条告警做通知。

  

延时队列有很多种方式实现，这里我们选择了Redis实现延时队列，选用 Redis 延时队列主要的原因就是其支持高性能的 score 排序，同时 Redis 的持久化特性，保证了消息的消费和存贮问题。

  

如图11所示，一个问题通过一系列校验去重之后放入redis延时队列，队列中到期时间最小的问题会被排到最前面，同时有一个监听任务不断查看队列中是否有过期的任务，如果有过期的任务会被取出，取出的消息会经过消息组装等操作最终形成一条消息文本，然后根据配置通过不同的通道发送出去。

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

（图11 延时任务执行原理图[6]）  

  

七、未来展望

  

基于统一告警服务定位来看，告警服务要能简单、高效、准确的告诉运维或者开发人员，哪里有故障需要去处理。所以对于后续服务的建设，应该考虑如何进一步减少人为的配置，增强告警智能化收敛的能力，同时还要增强根因定位的能力，以上通过AI的加持能够很好的解决此类问题。目前各大厂商都在向着AIOps探索前进，并且已经有一些产品投入使用，但是AIOps何时大规模落地，就目前来看还需要一段时间。相较于AI的使用，当前最紧迫的就是让统一告警串联起上下游服务，从而打通数据，为数据流转铺平道路，增强服务的自动化程度，并且支持从更高维度实现告警发送，为故障的发现和解决提供更准确的信息。

  

八、参考资料

  

[1]What are MTTR, MTBF, MTTF, and MTTA? A guide to Incident Management metrics

[2]平均修复时间[Z].

[3]运维不容错过的4个关键指标！

[4]PIGOSS TOC 智慧服务中心让告警管理更智能

[5]大规模智能告警收敛与告警根因技术实践[EB/OL].

[6]你知道Redis可以实现延迟队列吗?

  

**参考阅读：**

  

- [美图Android编译速度优化实践指南](http://mp.weixin.qq.com/s?__biz=MzAwMDU1MTE1OQ==&mid=2653557685&idx=1&sn=f0be7227e57943ee3b4bffd5388396f5&chksm=8139822db64e0b3b19b1bfee97d9ca5c756e2bf9c416fa93c228727eb4987229af72955436af&scene=21#wechat_redirect)
    
- [基于etcd实现大规模服务治理应用实战](http://mp.weixin.qq.com/s?__biz=MzAwMDU1MTE1OQ==&mid=2653557639&idx=1&sn=7655a93ad702670b5ff685e6aa7b7468&chksm=8139821fb64e0b09d47272952230a73325af8a9a5dbb14f94cd8c4d2c2ba6e2e5cc411555885&scene=21#wechat_redirect)
    
- [深入剖析Redis客户端Jedis的特性和原理](http://mp.weixin.qq.com/s?__biz=MzAwMDU1MTE1OQ==&mid=2653557624&idx=1&sn=11274fdc8454ab6ebc4ba1d562148a6c&chksm=813982e0b64e0bf671d7b3a32219e5f2a905372018d1add48f78ac0ff63b9e033a7c58b1782c&scene=21#wechat_redirect)
    
- [终于！12年后Golang支持泛型了！（内含10个实例）](http://mp.weixin.qq.com/s?__biz=MzAwMDU1MTE1OQ==&mid=2653557609&idx=1&sn=9ee127addb3d59542d13d92328bef739&chksm=813982f1b64e0be77d1226bbb8ae3c610246cb03f45dbd18376a324659fa88eeb4ff5daf4e2f&scene=21#wechat_redirect)
    
- [浅谈 Apache APISIX 的可观测性](http://mp.weixin.qq.com/s?__biz=MzAwMDU1MTE1OQ==&mid=2653557572&idx=1&sn=2927f2304ef6f8975992d33624e13b4c&chksm=813982dcb64e0bcac3f596d3c6ffb906f0094a17168cb6694e1d6fa486d47eee03b2af509214&scene=21#wechat_redirect)
    

  

技术原创及架构实践文章，欢迎通过公众号菜单「联系我们」进行投稿。  

  

**高可用架构**

**改变互联网的构建方式**

  

![](https://res.wx.qq.com/t/fed_upload/b39ef69e-c4d6-4169-8612-5f00a84860e7/wx-avatar-default.svg)

公众号

阅读 5496

​