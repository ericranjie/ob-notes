原创 Gao Xiang vivo互联网技术

_2022年01月19日 20:59_

> 作者：vivo互联网服务器团队-Gao Xiang

一、什么是事件驱动架构

当下，随着微服务的兴起，容器化技术的发展，以及云原生、serverless 概念的普及，事件驱动再次引起业界的广泛关注。

所谓事件驱动的架构，也就是使用事件来实现跨多个服务的业务逻辑。事件驱动架构是一种设计应用的软件架构和模型，可以最大程度减少耦合度，很好地扩展与适配不同类型的服务组件。在这一架构里，当有重要事件发生时，比如更新业务数据，某个服务会发布事件，其它服务则订阅这些事件；当某一服务接收到事件就可以执行自己的业务流程，更新业务数据，同时发布新的事件触发下一步。

事件的发布与订阅，需要依赖于一个可靠的消息代理。见下图：

![图片](https://mmbiz.qpic.cn/mmbiz_png/4g5IMGibSxt6nHDL3ZnSe8h6pLA5mZW8gh4pfAkNwur0DjyVlPxia1qrITic9hYpXNlVwjVmhbSQunmPsQxXnVzMw/640?wx_fmt=png&wxfrom=13&tp=wxpic)

当然，事实上有不少软件项目都使用了消息队列，但是这里需要明确的是，对消息队列的使用并不意味着你的项目就一定是事件驱动架构，很多项目只是由于技术方面的驱动，小范围地采用了某些消息队列的产品而已。偌大一个系统，如果你的消息队列只是用作邮件发送的通知，那么这样系统自然谈不上采用了事件驱动架构。

在采用事件驱动架构时，我们需要考虑业务的建模、事件的设计、上下文的边界以及更多技术方面的因素，这个系统工程应该如何从头到尾的落地，是需要经过思考和推敲的。总而言之，“事件驱动架构”的设计并不是一件易事。本文在后面有个例子供参考。

另外，如果盲目使用事件驱动设计架构，就有可能要承担中断业务逻辑的风险，因为这些业务逻辑具有概念上的高度内聚，却采用了解耦机制将它们联系在一起。换句话说，就是将原本需要组织在一起的代码强行分离，并且这样难于定位处理流程，还有数据一致性保证等问题。为了防止我们的代码变成一堆复杂的逻辑，我们应当在某些明确场景下使用事件驱动架构。以经验来讲，以下三 种场景可以使用事件驱动开发：

- 组件的解耦

- 执行异步任务

- 跟踪状态的变化

二、什么时候使用事件驱动架构

2.1 组件的解耦

当服务（或组件） A 需要执行服务 B 中的业务逻辑，相比于直接调用，我们可以向事件代理（事件分发器）中发送一个事件。服务 B 通过监听分发器中的特殊事件类型，然后当这类事件被接收到时去执行它。

这意味着服务 A 和服务 B 都依赖于事件代理和事件，而无需关注彼此实现：即完成它们的解耦。见下图：

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

基于这种松耦合，服务可以用不同的语言实现。解耦后的服务能够轻松地在网络上相互独立地扩展，通过动态添加或删除事件生产者和消费者来修改他们的系统，而不需要更改任何服务中的任何逻辑。

2.2 执行异步任务

有时我们会有一系列需要执行的业务逻辑，但是由于它们需要耗费相当长的执行时间，所以我们不想看到用户耗费时间去等待这些逻辑处理完成。在这种情况下，最好将它们作为异步任务来运行，并立即向用户返回一条信息，通知其稍后继续处理相关操作。

比如，内容字段的检查等入库流程可以采用“同步”执行处理，但是执行内容理解则采用”异步“任务去处理。在这种情况下，我们所要做的是触发一个事件，将事件加入到任务队列中，直到一个服务能够获取并执行这个任务。此时，相关的业务逻辑是否处在同一个上下文中环境中并不重要，不管怎么说，业务逻辑都是被执行了。

2.3 跟踪状态的变化

在传统的数据存储方式中，我们通过实体模型存数据。当这些实体模型中的数据发生变化时，我们只需更新数据库中的行记录来表示新的值。这里有个问题，就是业务上我们无法准确存储数据的变更和修改时间。但是在事件驱动架构中，可以通过事件溯源将包含修改的内容存入到事件里。下面会详细讨论“事件溯源“。

三、为什么使用事件驱动架构

当大家谈论事件驱动架构时，比如大家说自己恰好在最近的项目中采用了事件驱动架构，实际上，他们可能在谈论下面这四种模式中的一种或者几种：

- 事件通知

- 事件承载状态转移

- 事件溯源

- CQRS

> 注：概念来源2017年GOTO Conference上Martin Fowler分享的The many meanings of Event-Driven architecture。

3.1 事件通知

假设我们现在想要设计一个简易的内容平台，包含三部分：

- 内容引入系统

- 作者微服务

- 关注中心

当内容创作者通过内容引入系统上传视频之后，会触发如下的一个调用流程见下图：

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

1. 内容引入系统收到创作者上传的视频，执行入库流程；

1. 内容引入系统调用作者微服务的API，增加“视频-创作者”的从属关系；

1. 作者服务调用关注中心的API，让关注中心给关注了这个创作者的其他用户发送作者视频更新的通知。

上面这个调用流程，不可避免地创建了下面的依赖关系：

- 内容引入系统依赖于作者微服务的API，虽然内容引入系统其实不太关心作者微服务的业务。

- 作者微服务依赖于关注中心的API，虽然作者微服务也不关心关注中心的业务和处理流程。

这种依赖关系很有可能并不是我们所期望的。内容引入系统是一个比较通用的业务，不同类型的内容引入系统很可能会有相似功能，如字段类型检查、入内容库、启动高敏审核等。作者服务则是一个非常专业的系统，如不同源、不同类型的内容关于作者的业务逻辑是不同的。让一个通用的系统依赖于一个专业的系统，不管从设计角度，还是后续系统维护角度，都是不一个好的方案。作者微服务可能会经常根据业务需求做变更，但内容引入系统相对稳定，而上面这种依赖关系让我们难以在“不对内容引入系统做调整的情况”下随意更改作者微服务。

从架构层面，我们希望让作者微服务依赖于内容引入系统，让一个专业的系统依赖于一个稳定的、通用的系统，增加系统的稳定性。这个时候我们可以借助于“事件通知”。见下图：

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**优点**

- 架构更健壮。如果加入队列的事件能够在源组件中执行，但在其它组件中由于 bug 导致其无法执行（由于将其加入到队列任务中，它们可以在 bug 修复后再执行）。

- 业务处理减少延迟。当用户无需等待所有的逻辑都执行完成时，可以将这类工作加入到事件队列。

- 便于系统扩展，能够让组件的研发团队独立开发，加快项目进度、降低功能难度、减少问题发生并且更有组织性。

- 将信息封装在“事件”里，便于系统内传播。

**缺点**

- 如果没有合理使用，可能使我们的代码变成“面条式”代码。

- 数据一致性问题。由于流程依赖于最终的一致性，因此通常不支持ACID事务，因此重复或乱序事件的处理会使服务代码更加复杂，并且难以测试和调试所有情况。

“事件通知”的缺点和优点相对应，正是因为它提供了很好的解耦能力，我们会比较难通过阅读代码去得到整个系统和流程的全貌。因为这些逻辑之间的关系不再是之前的依赖关系。这将会是一个挑战。

3.2 事件承载状态转移

我们在使用事件通知时，事件里面往往不会包含下游系统处理这个事件需要的所有信息。比如当内容发生下架变更时，内容平台会生成一个“内容下架“的事件，但当下游系统处理这个事件时，往往还需要知道，该内容上个状态是什么，是谁触发下架等信息，才能完成后续处理。所以不可避免地，下游系统在处理这个事件时，往往还需要通过平台服务来获取这些额外信息。

为了解决这个问题，我们引入一个种新的模式，叫做“事件承载状态转移”。简单来说，就是让事件的消费方自己保留一份在业务处理过程中需要用到的上游系统的数据。比如让下游系统保留一份在处理内容状态变更事件时所需要用到的内容变更前的状态，避免回头去平台查询。

**优点**

- 架构更健壮。减少事件消费方对生产方的额外依赖（获取事件处理所需数据）；

- 业务处理减少延迟。增加事件消费方系统的响应速度，因为不再需要调用平台API以获取事件处理所需数据；

- 无需担心被查询组件的负载（尤其是远程组件）。

**缺点**

- 尽管现在数据存储已经不再是问题根源，依然会保存多个只读的数据副本，一致性进一步被破坏；

- 增加数据处理的复杂度，即使处理逻辑符合规范，它也需要额外处理和维护外部数据的本地副本业务逻辑。

3.3 事件溯源

有些时候我们不但关心系统当前的状态，我们还关心如何变成当前这个状态的，但是数据库仅仅简单地保存实体的当前状态。事件溯源可以帮助我们解决这个问题。

事件溯源是一个特别的思路，它并不持久化实体对象，而是只把初始状态和每次变更的事件记录下来，并在内存中根据事件还原实体对象的最新状态，mysql主从备份用到的binary log以及redis的aof持久化机制，都可以认为是“事件溯源”的实现。

事件溯源在做完数据库更新之后，它将事件的发送操作转换为往数据库或者日志系统中写入一条事件记录，其它节点通过查询数据库或者文件系统，来得到这些事件，并通过回放来确保数据的最终一致性。

**优点**

- 可以呈现一个完整的变动历史；

- 提供更方便的debug手段；

- 可以回溯到任何一个历史状态；

- 方便修改当前事件；

**缺点**

- 要实现一个可靠和高性能的事件仓库（保存的事件记录）并不是一件容易的事情，应用代码需要根据事件库的 API 进行重写。

3.4 CQRS

CQRS全称是Command Query Responsibility Segregation。简单来说，就是针对系统的读写操作，使用不同的数据模型、API接口、安全机制等，来达到对读写操作的完全隔离，满足不同的业务需求。见下图：

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

根据存储在事件库中的事件集合，可以计算得到每个业务实体的状态，这些状态以物化视图的方式存储在一个数据库中。当有新的事件产生时，也同样会自动更新视图。这样，视图查询服务就可以像查询普通的数据库数据一样实现各种查询场景。具体的设计可参考下图所示：

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

四、事件驱动架构在内容平台中的实践

在当今社会，内容“横行”的时代，内容平台企业需要有极强的灵活性和应变能力。特别是在中国这样一个内容行业（如视频）飞速发展的市场里，企业要求平台能够快速地对内容业务需求做出应对，否则就会丧失先发优势。这有点类似于现代战争条件下，各国都要求部队具备快速反应能力，这种能力主要体现在平台能够通过快速开发或者重用 / 整合现有资源来达到快速响应业务需求。

随着内容行业业务越来越庞大复杂，所涉及的存储类型、处理器、账号体系、效率工具、数据和结算系统等非常多，这就要求平台有很强的整合能力以及对异构环境的适配能力。

最后，由于内容行业的发展日新月异，特定类型的内容业务（如小视频）都会在其初中期发展后迎来一个快速膨胀期，业务量和业务类型会急剧增加，这也要求平台有很好的可扩展性。相关平台架构见下图：

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

4.1 创建事件

事件其实是DDD（领域驱动设计）中的一个概念，表示的是在一个领域中所发生的一次对业务有价值的事情，落到技术层面就是任何影响业务流程或者状态的改变。事件具有自己的属性，比如发生的时间、发生了什么、事件之间的关系、状态以及变化，事件也可以生成新的事件，根据不同的事件生成新的业务事件。在创建事件时，首先需要记录事件的一些通用信息，比如唯一标识ID和创建时间等，为此创建事件基类ContentEvent：

```
public abstract class AbstractContentEvent {
```

在一般场景下，事件一般随着聚合根（也是DDD的一个概念，这里泛指视频id）状态的更新而产生，另外，在事件的消费方，有时我们希望监听发生在某个聚合根下的所有事件，为此建议为每一个聚合根对象创建相应的事件基类，其中包含聚合根videoId，比如对于视频（Video）类，创建VideoEvent：

```
public class VideoEvent extends AbstractContentEvent {
```

然后对于实际的视频事件，统一继承自VideoEvent，比如对于视频引入的VideoInputEvent事件；

```
public class VideoInputEvent extends VideoEvent {
```

视频域事件的继承链见下图；

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

在创建事件时，需要注意两点：

1. 事件本身应该是不变的；

1. 事件应该携带与事件发生时相关的上下文数据信息，但是并不是整个聚合根的状态数据。例如，在视频引入时可以携带视频的基本信息article，而对于视频状态更新的VideoStatusChangeEvent事件，则应该同时包含更新前后的状态status：

```
public class VideoStatusChangeEvent extends VideoEvent {
```

4.2 发布事件

发布事件有多种方式，比如可以在应用程序中发布。通常的业务处理过程都会更新数据库然后发布事件，这里一个比较常见的场景是：需要保证数据库更新和事件发布之间的原子性，也即要么二者都成功，要么都失败；当然也有不需要保证原子性的场景。如果需要保证原子性，以“内容引入”的业务流程为例，见下图：

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

- 接收内容；

- 写入内容表；

- 写入事件表，且和内容表的更新在同一个本地数据库事务中；

- 事务完成后，触发事件的发送；

- 读取事件表；

- 将事件发送到消息队列；

- 发送成功后，将记录标注为“已发送”；

4.3 消费事件

在消费事件时，除了完成基本的消息处理逻辑外，我们需要重点关注以下三点：

- 消费方的幂等性；

- 消费方有可能进一步产生事件；

- 消费方的数据一致性；

对于“幂等性”，事件的发送机制保证的是“至少一次投递”，这是有消息中间件保证，技术选型时需要注意。为了能够正确地处理重复消息，要求消费方是幂等的，即多次消费事件与单次消费该事件的效果相同。保证“消费幂等性”的方法有很多，这里介绍一种。在消费方创建一个事件表，用于记录已经消费过的事件，在处理事件时，首先检查该事件是否已经被消费过，如果是则不做任何消费处理。

对于第二点，依然沿用前文讲到的“事件表”的方式。事实上，无论是处理服务请求，还是作为消息的消费方，对于聚合根（videoId）来讲都是无感知的，事件由聚合根产生进而由事件库持久化，这些过程都与具体的业务操作源头无关。

对于“数据一致性”，本质上是由第二点引出，事件驱动架构在业务对象之间通过异步的消息来同步状态，有些消息也可以同时发布给多个服务，在“消息引起了一个服务的同步”后可能会引起另外的消息，事件会扩散开。严格意义上的事件驱动是没有同步调用的，如何保证一致性，就要比非事件驱动架构要复杂，通常采用“cache aside”模式和“分布式锁”来保证一致性。

综上，在消费事件的过程中，应用程序需要更新业务表、事件记录表，此时整个事件的发布和消费过程见下图；

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

五、总结

主流场景下，传统面向服务（或以数据驱动）的平台存在系统性不足，需要增强以下能力：

- 在传统数据集成基础上需要进一步提升业务集成能力。

- 需要提高集成平台的业务敏捷性和反应能力。

- 需要进一步实现业务系统间的解耦和高可靠性。

- 需要进一步提升管控平台的实时响应能力。

”事件驱动架构“天然地满足了这些能力要求。事件驱动架构”天生“的优点，比如，封装、高内聚和低耦合，还可以提升代码的可维护性、性能和业务增长的需求，通过事件溯源模式，还能提高系统数据的可靠性。

不过，事件驱动同样存在弊端，因为无论是概念上的复杂度还是技术上的复杂度都增加了，当它被滥用时将导致灾难性的后果。所以，在技术栈的选用方面，给出以下寄语：

> 1）不要“盲目的追新” 技术人员的喜好往往是什么技术流行就追什么技术。现在的技术发展快，前后端不断涌现各种框架，我们恨不得把这些框架都用在自己的项目里才行，按实际出发，按需所用，适当的预留技术预研的空间。

> 2）不要“按技术站队，以结果反推“ 很多人把手段当成了目的，成为了框架的信徒。用了Java开发，你的设计就一定是面向对象吗？用了Spring boot就是微服务了吗？一定要技术和实际场景结合，架构师也要深入了解掌握技术，但是更多的是了解技术的优劣和使用场景，而不是简单的生搬硬套。

END

猜你喜欢

- [vivo 敏感词匹配系统的设计与实践](http://mp.weixin.qq.com/s?__biz=MzI4NjY4MTU5Nw==&mid=2247492482&idx=1&sn=784519dee3a27a12f65869767efb02f6&chksm=ebdb9310dcac1a0684abe75c2bac085d959f5dafc2bd444f3f4a8e82db238b7c23f966481493&scene=21#wechat_redirect)

- [前端质量提升利器-马可代码覆盖率平台](http://mp.weixin.qq.com/s?__biz=MzI4NjY4MTU5Nw==&mid=2247492808&idx=2&sn=11c632a4e7e56414cb05dd1235984ffc&chksm=ebdb945adcac1d4ccffcda5e4deaddba54ff4c63fa2908a2a83224b0ea1411811d6708a66882&scene=21#wechat_redirect)

- [vivo 全球商城：商品系统架构设计与实践](http://mp.weixin.qq.com/s?__biz=MzI4NjY4MTU5Nw==&mid=2247492304&idx=1&sn=4f2a666cd60ce0297db5c20a763cb4b9&chksm=ebdb9242dcac1b54e234786ee3b2d60a873f304ee5c562410e95fbc4ef21f969c0113c7189e0&scene=21#wechat_redirect)

![](http://mmbiz.qpic.cn/mmbiz_png/4g5IMGibSxt45QXJZicZ9gaNU2mRSlvqhQd94MJ7oQh4QFj1ibPV66xnUiaKoicSatwaGXepL5sBDSDLEckicX1ttibHg/300?wx_fmt=png&wxfrom=19)

**vivo互联网技术**

分享 vivo 互联网技术干货与沙龙活动，推荐最新行业动态与热门会议。

431篇原创内容

公众号

云原生2

云原生 · 目录

下一篇vivo 云原生容器探索和落地实践

阅读 3196

​
