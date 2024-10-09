Original 宋铜铃 CPP与系统软件联盟

_2022年04月29日 17:50_ _上海_

**点击蓝字**

**关注我们**

本文摘录自华为资深软件架构师宋铜铃老师在「全球C++及系统软件技术大会」的专题演讲。

**01**

**引言**

![Image](https://mmbiz.qpic.cn/mmbiz_png/YQ55UxynrogA3YY4ibWDhbnNHKnauDg1gKjVkw9prqeDaBVz6VAuREBC1Bta7hPHNEJDoJCN7K8ibkFSvY7u9Lgg/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

关于软件架构，主要分为通用软件和嵌入式软件。通用软件大家接触的更多可能是微服务，kubernetes，serverless，kafka等，现有的内容平台、中间件都很丰富，可以讲的东西也很多。而在嵌入式软件方面，主要是跟 Linux，C/C++打交道，甚至有时跟硬件，电路板，示波器，烙铁等打交道。

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**1.1 嵌入式系统开发的限制：**

- **有限的硬件资源**

- **以C语言为主要工作语言**（也可以用C++，但是C++对于人员素质要求更高，质量更难把控）

- **缺少丰富的中间件、框架**（如 Java 的 springboot）

- **不得不自己造轮子**

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**1.2 嵌入式开发语言现状：**

- C语言仍然是主流（2020年仍然排行第一）

- C程序的庞然大物让人吃惊（华为1100亿行代码）

- C程序更容易失控

- 缺少成熟的平台框架，不支持原生OOP，脆弱的内存安全性（C开发人员需要花费大量时间去定位内存、指针问题）

**1.3 C程序系统存在的问题：**

- 修改一行代码，要理解1000行代码，网状的依赖

- 腐化扩散，脆弱的接口隔离

- 对外的接口，内部接口，业务代码没有明显的边界

- 靠规章制度，而不是说技术手段来扩展接口的看护

- 添加功能困难，去掉不需要的功能也困难

- MR相互等待，和代码效率低，解决不完的 Git 冲突

- 模块无法单独调试，测试，需要把整个系统跑起来，需要一群人开展调试

- 只有把软件加载到真实的硬件上才能运行调试

**1.4 架构问题引入**

上述存在的问题，让我们不得不反思，究竟是什么导致这些问题的存在？

比如开发加班多，bug多，项目延迟多等一系列问题，大家很自然会想到这就是架构问题（架构烂、架构不好等）。架构的责任很大，决定这个项目能否成功，也决定着开发人员的幸福生活（加班、工作效率等）。

那我们作为架构人员，就需要去掌握很多有用的架构原则和理论知识。同时，我们也可以跟其他人说明我们的架构是合理的，符合 SOLID 原则。

我们来看看一些常用的架构原则：

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

所有这些原则，对于架构师来说，就是一个字“**拆**“，怎么样把一个大系统，拆成若干小系统，拆成子系统，架构师考虑问题就要考虑拆哪里，怎么拆。

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

2009年，Michael T. Fisher（迈克尔.T.费希尔）和 Martin L. Abbott（马丁.L.艾伯特）写了一本书叫《架构即未来》\_（The art of scalability）。\_原名中所谓的 art（艺术），就是你一下子学不会，需要花很长的时间去训练才能掌握。这本书里有个例子叫“拆分立方体”，告诉了我们怎么去拆分系统。它每一个轴从不同的维度去拆分，比如从功能维度、从数据维度、从实例多少的维度等等。

2014年，Martin Fowler（马丁·福勒） 与 James Lewis（詹姆斯·刘易斯）共同提出了微服务架构的概念，其中也有一个类似的拆分立方，我们称之为“微服务立方”。

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

我们为什么要做拆分？我们有很多成功的架构，也有更多失败的架构。那为什么会成功，为什么会失败，架构师如何去设计架构？每当一个新的架构设计出来的，在当时的情形看来设计得是比较完善的。类抽象得也很好，模块划分也很好，但是为什么有时候会失败呢？

所以架构师想的不仅仅是当时的业务怎么去抽象，怎么去设计的问题，更要考虑我们的架构在面临变化的时候，它能不能承受住各种不同的变化，这种带来驱动变化的这些因素的冲击。

变化的因素有哪些？比如需求的变化、性能要求的变化、硬件的变化，场景的变化、团队的变化等很多种因素。这些不同的变化可能导致我们的代码发生变更，在这个过程中我们的接口能否保持稳定。那架构师怎么去理解这些变化呢？要求我们对业务、对趋势有一个比较深刻的了解，才能发现可能存在哪些变化因素。所以架构师考虑问题一定是动态演变的。

那我们就以一个华为内部的真实案例来说明，基于C语言的微组件架构是怎么设计的。

**02**

**微组件架构设计实践案例**

**2.1 IBMC软件的架构实践**

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

IBMC是一个系统软件，它的代码规模也很大，有C/C++，也有Rust/Javascript，从驱动到用户接口到Web界面都是全栈的，自研代码近200万行，不包含里面的开源框架、Linux等，在华为内部先后支持超过100款产品，产品应用领域也很广，全球一百多个电信运营商、国内大大小小各种互联网企业都在用，在很多个产品里做到了保持一份代码总概。

该软件面临的主要架构挑战：

- 需要运行在多种嵌入式SoC上，CPU指令集可能存在差异，芯片驱动各不相同

- 需要支持数千种硬件

- 用户需要求持续活跃

- 需要支持多团队异步并行开发

为了解决上述挑战，我们的架构就在向微组件架构演进。

**2.2 微组件架构特征**

微组件架构的概念是我们总结的一个名词，我们用12特征来进行阐述：

**1. 粒度可以小于进程**

一般情况下，在我们的组件化架构软件里都是以进程为粒度，可以做到很好的代码隔离、特性隔离、故障隔离，也可以做到独立的验证、调试，但进程粒度偏大，有的时候我们可能只需要改一点点，如果这种时候增加一个组件就比较浪费。如果不增加组件，就只能在组件内增加很多定制化的代码，导致组件很不稳定。如果组件粒度小于进程，可以在不修改原有组件的情况下，新增新的组件来解决新的需求。

**2. 可独立开发**

**3. 可独立构建**

**4. 可独立调试**

**5. 可独立测试**

**6. 可裁剪**（构建阶段，组装阶段）

**7. 可替换**（构建阶段，组装阶段）

**8. 接口支持OOP**（可以多态，继承，扩展）

**9. 接口延时绑定**

在运行期才确定接口最终的行为，编码的时候是不知道接口具体怎么实现的。

**10. DAG依赖**（有向无环图）

组件、类、接口之间都是遵循DAG依赖约束，即依赖是单向的，不能成环。

**11. 接口可视**（支持静态检查、支持工具辅助开发）

可视指的是，比如你有一段代码，无论是C还是C++，其中有些函数是对外的接口，有些函数是内部的功能，如果要让一个工具去分析哪些是对外接口是很难分析出来的，这种情况下你的接口是不可见的，只有开发人员自己知道，或者是文档里有写。在CI/CD自动化集成、自动化测试的时候是不知道哪些是你的接口的。

**12. 依赖关系可视**（支持静态检查、支持工具可视化）

**2.3 微组件的实现：**

要满足上述12个特征，我们具体该怎么去实现？

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

上图是整体的架构图，由插件构成组件，两者合一为微组件。这个微组件框架很重要，要承载上述这些组件可裁剪、可装配、DAG依赖、粒度小于进程等等一些特性。无论是进程粒度的组件还是小于进程粒度的组件，它们都是在不同的代码仓里，代码仓是独立的，由不同的团队并行开发维护，最终在构建的时候，根据装配关系，生成最终的软件包。

**2.4 基于微组件架构的拆分**

基于前面所说的架构很重要的行为就是做拆分，那基于微组件架构我们怎么去做拆分，大致有如下几种分类：

- **不同功能域→拆分不同组件**

  比如功能不同的两个功能域之间没有关系的，可以独立拆分成不同组件

- **同一功能域，通用的和特定的行为→拆分为组件和插件**

  对于同样的功能，在不同的场景下有不同的需求，但是又要用到平台的功能，稍微有点差异化的时候（类似于C++的多态化），这个时候可以拆分成插件和组件。

- **行为和数据→拆分代码和类（对象）**

  行为即我们的代码，数据即为类。行为承载着我们的业务功能和流程，类在代码中还是抽象的，但在具体的产品场景下类必须实例化，就变成了对象。所以我们把实例化的对象数据放在具体产品里，即把类放在代码里。因为我们的代码具体跑在什么产品上是不确定的，这个时候，对象最终是由产品现场的环境来决定的。

- **驱动按照稳定要素拆分**

  · 芯片是稳定，芯片的组合、硬件的组合是可变的→拆分为芯片驱动和拓扑驱动。

  · 芯片驱动拆分：芯片是稳定的，但其地址、状态是可变的→拆分为芯片驱动类（代码）+芯片对象实例（数据）。

**2.5 接口设计**

我们的组件与组件之间、组件与插件之间、组件与数据之间都有接口，代码和硬件之间有驱动接口。我们架构上要保持接口的稳定性和可扩展性。那我们很容易联想到OOP思想（面向对象），所以我们用**OOP的方式设计接口**。

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

在这样的架构下，我们所有的接口都会抽象为类，实例化之后即为对象。所以”一切皆为对象，对象即接口”。

**组件间的接口，实际上是单例接口，因为我们的组件是单例部署的。**

- 组件与插件间的接口：多实例接口

- 代码中的抽象业务流程：多实例业务对象

- 驱动：硬件对象

- 依赖关系：类之间的依赖，对象之间的依赖、组件插件之间的依赖

那具体怎么来做OOP的设计？先来回顾一下 C++ 的 OOP。

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

在C++里面我们可以做继承、做重写、做实现、做扩展，还有多态。用UML语言来表达就是上图右边部分，箭头方向是代表子类指向父类。

也有人写过C的OOP，如果自己去体验一下OOP的思想去尝试一下是可以的，但是在实际产品中用C去写面向对象挺费劲的。

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**2.6 架构级OOP**

那我们究竟去实现架构级OOP？

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

- 插件是依附于组件运行的（进程是最小运行单位），但插件是独立开发的，可以由不同的团队并行开发。

- 插件1可以新增方法、属性（继承），实现了扩展组件1的业务功能。

- 插件2可以从组件的 Interface 接口进行重写、继承，改变了组件1的对外接口。

这些接口都是延迟绑定的，我们在运行的时候才能确定具体运行什么样的代码。

**2.7 DSL&GPL**

那么我们怎么表达这些接口？

我们引入了DSL(Domain Specific Language)领域特定语言。GPL(General Purpose Language)即通用编程语言，比如 Java、Python、C/C++之类的。

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

- **DSL来描述接口/类，GPL来实现具体的业务逻辑（比如用C实现）**

- **DSL是一种胶水语言，用DSL来连接组件和插件，软件和硬件，业务和数据。**

  所有的接口我们用DSL一种语言来描述，既可以让开发人员学习成本降低，理解起来也是一致的。所以我们是在系统级上实现了OOP，并不是在某个函数、模块内部实现OOP。

- **混合编程**

  有了系统级的OOP，模块内部的实现语言，就没有限制了，实现了混合语言编程。

**03**

**延迟绑定**

- 运行时确定代码和对象的关系

  即接口最终由谁实现跟我们最终装备构建时有关系，和我们选择什么组件有关。

- 对象在编码时不确定，在运行时构造

- 无编译依赖以实现独立的构建、调试、测试

- 延迟绑定可以支持组件/插件的动态替换或业务对象的动态变更

延时绑定实现示例（DSL绑定GPL）：

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

通过DSL真正达到了无编译依赖，组件在运行时可以动态替换。让架构师和程序员两种角色完全独立。

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

CI/CD 即持续集成，开发流水线，因为DSL语言对接口有显示的描述，所以很多工具如接口检查工具、面向接口的测试等等在这个时候都可以自动化完成。

**04**

**DAG依赖**

- 组件间的依赖

- 插件间的依赖

- 类之间的依赖

- 对象之间的依赖

- DAG依赖确保了系统构建时可裁剪

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

- 消息也是造成隐性依赖的重灾区

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

消息通讯：zeromq/rabbitmq，发送者阻止不了接收者的监听。消息的依赖也可能会导致不可裁剪，间接的说明依赖其消息的内容，还需要知道消息的格式定义，或者要引用头文件等。

所以我们在这里对消息队列做了一个改造，，由微服务架构来支持消息的收发，来检查消息发送者和接收者之间是否满足单向依赖关系。如果A组件不依赖于B组件，那么A就不会接收到B组件发送的广播消息。

**05**

**驱动解耦**

在嵌入式软件中，还有很重要的一部分是驱动。在Linux也有一种配置文件来描述硬件的方式：Linux DTS(Device Tree Source)设备树。DTS 是为 Linux 提供一种硬件信息的描述方式，以此代替源码中的硬件编码 (hard code)。

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

DTS的描述是在构建的时候确定的。在真实的产品场景里，构建的时候并不能确定这个硬件的连接关系，真实的产品是动态组合的，有大量的公共部件+部件的组合（很多硬件是可以更换的）。那么当器件的组合不确定时，驱动怎么写? 如果去把所有驱动的穷举出来，会导致大量代码重复，维护工作量也很大。所以驱动的行为是运行时而不是编译时确定的。

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

我们这里引入运行时驱动（runtime drive）的概念：

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

用DSL去表达驱动装配关系，还有配套的工具去编辑一个DSL代码，运行的时候框架去解析，就知道哪些驱动以什么样的方式拼装在一起，这样在运行时驱动就可以兼容这种动态组装的场景。

**06**

**DSL IDE&Tool**

工具是效率的保障，好的工具让开发人员难以出错：

- 语法检查

- 自动补齐

- 代码提示、跳转

- 依赖检查

- 代码自动生成：主要是开发了一些vscode的插件来支持

**07**

**构建系统**

- 并发构建，急速构建

- 二进制级别的装配

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**08**

**微组件架构的总结**

基于微组件架构可以获得以下收益：

- 可以以更细的粒度划分可独立开发的模块（组件及插件）

- 模块之间代码分仓隔离，以二进制文件形式进行组装

- 模块可以独立开展CI/CD，支持开发流水线

- 系统级的OOP，支持跨模块的多态性，支持接口的继承，支持重写与扩展

- 系统局部二进制级别的可裁剪性、可替换性

- 架构由DSL语言承载、业务代码支持混合语言，自由选择面向过程还是面向对象

- 接口的定义和业务逻辑代码分离

- 接口的看护可以工具化，可视化

撰稿人：和光同尘

**直播预告**

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**侯捷**

**Boolan 首席软件专家**

侯捷先生是两岸著名技术教育者，计算机图书作者、译者、书评人。著有《深入浅出MFC》、《多态与虚拟》、《STL源码剖析》、《无责任书评》三卷，译有众多脍炙人口的高阶技术书籍，包括Scott Meyers所著的“Effective C++”系列。侯捷先生还兼任教职于元智大学（台湾）、同济大学（大陆）、南京大学（大陆）。其著作、讲座影响大陆一代程序员。

5月06日晚8点，Boolan 首席软件专家侯捷老师空降直播间，带大家\*\*《与侯捷一起聊C++高维精进》\*\*：

- C++的洋葱法则怎么指导我们？

- C++都有哪些硬骨头、怎么啃？

- 高维进阶的“罗马大道”是什么？

- 精进修炼有哪些心法和手法？

立即扫码 免费预约

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

Reads 1662

修改于2022年04月30日

​
