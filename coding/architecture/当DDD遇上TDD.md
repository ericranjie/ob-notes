
原创 张晓龙 CPP与系统软件联盟

 _2022年01月28日 08:30_

  

  

  

**点击蓝字**

**关注我们**

  

  

**特|邀|作|者**

![图片](https://mmbiz.qpic.cn/mmbiz_png/YQ55UxynrojmRE1zZZ6ict9MzNaMibMFUcnwU2Kr6gnDndlGLWPQIicNWOcZVdHnw4LTRa1dGaPTxyfqeLkIpTDpA/640?wx_fmt=png&wxfrom=13&tp=wxpic)

**张晓龙**

**中兴通讯云平台资深软件架构师**

公司十佳教练，Go 语言知名打桩框架 gomonkey 作者，具有十多年软件架构和开发经验。近年来专注于 PaaS 和 5G 等大型平台软件的设计与开发，尤其对于 DDD 和微服务架构具有深刻的理解，对于大型软件的重构具有丰富的实战经验。曾作为演讲嘉宾出席全球C++及系统软件技术大会、架构师峰会、质量竞争力大会和领域驱动设计峰会等，广受好评。

  

“Design is there to enable you to keep changing the software easily in the long term” —— Kent Beck

![图片](https://mmbiz.qpic.cn/mmbiz/YQ55UxynrojmRE1zZZ6ict9MzNaMibMFUcgsRBEQOgYetxGviahnHDkvibupsfv12iaYzMzZ4p0FlB8yaKtVAQT7pjg/640?wx_fmt=other&wxfrom=13)

  

1.

  

**引言**

  

**极限编程**

极限编程（ExtremeProgramming，XP）是由Kent Beck在1996年提出的，是一种软件工程方法学，在敏捷软件开发中可能是最富有成效的几种方法学之一。1999年Kent Beck出版了书籍《解析极限编程：拥抱变化》，2004年他又出版了该书籍的第二版。

  

极限编程之所以叫“极限”，它背后的理念就是把好的实践推向极限：

- 如果集成是好的，那么我们就尽早集成，做到极限就是每一次修改都集成，这就是持续集成；
    
- 如果测试是好的，那么我们就尽早测试，做到极限就是先写测试再根据测试调整代码，这就是测试驱动开发；
    
- 如果代码评审是好的，那么我们就多做评审，做到极限就是随时随地的代码评审，这就是结对编程；
    
- 如果客户交流是好的，那么我们就多和客户交流，做到极限就是客户与开发团队时时刻刻在一起工作，这就是现场客户。
    

  

XP实践，如下图所示：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E "微信图片_20220127183036.jpg")

  

### **TBD发展史**

测试驱动开发（Test-Driven Development，TDD）位于XP实践的最内层，属于个人实践环。在个人实践环有四个实践，除过TDD，还有重构、结对编程和简单设计，其中TDD、重构和结对编程属于拉动实践，而简单设计是目标，属于核心实践。

  

1999年Martin Fowler出版了书籍《重构：改善既有代码的设计》，使重构技术从编程高手们的小圈子走出，成为众多普通程序员日常开发工作中不可或缺的一部分，就像空气和水一样，2018年他又出版了该书籍的第二版。

  

2002年Kent Beck出版了书籍《测试驱动开发》，书中不仅以案例的形式呈现了测试驱动开发的原则和方法，而且详尽地阐述了测试驱动开发的模式和最佳实践，为TDD技能向普通程序员的传播奠定了基础。

  

2003年Eric Evans出版了书籍《领域驱动设计：软件核心复杂性应对之道》，书中给出了领域驱动设计的系统化方法，并将人们普遍接受的一些最佳实践综合到一起，融入了作者的见解和经验，展现了一些可扩展的设计最佳实践、已验证过的技术以及便于应对复杂领域的软件项目开发的基本原则。同年，Dan North借鉴了DDD中的统一语言，并针对TDD实践的困惑提出了行为驱动开发（Behavior-Driven Development，BDD）的概念，随后又开发出了JBehave框架。

  

### **TBD侧重点不同**

  

软件开发的首要目标是满足用户的所有需求，包括功能性需求和非功能性需求。软件开发的最大挑战是需求变化，我们需要及时识别变化，并有效的管理变化。对于好的软件来说，我们需要打造一个兼顾易理解性和低成本响应变化的领域模型，并且不断演进和精炼。

  

作为开发人员，我们要以正确的方式来打造正确的软件，以满足用户需求，同时从容应对需求在未来的变化，这里有三个重要因素：

- 快速反馈：反馈速度越快，应对变化的能力越强，修复故障的成本越低；
    
- 改善设计：好的设计没有重复，易于理解且没有冗余，可以帮助我们低成本的应对需求变化；
    
- 贴近业务：领域模型可以约束代码的实现，同时代码可以精准的表达领域模型，让软件设计复杂度更加贴近业务的本质复杂度，从而软件可以更加高效的应对需求变化。
    

TDD，BDD和DDD的侧重点不同，如下图所示：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)  

2.

  

**TDD**

  

### **TDD是什么**

  

TDD以测试作为开发过程的中心，要求在编写任何产品代码之前，首先编写用于定义产品代码行为的测试，而编写的产品代码又要以使测试通过为目标。TDD要求测试可以完全自动化地运行，在对代码进行重构前后必须运行测试。这是一种革命性的开发方法，能够造就简单、清晰和高质量的代码。

  

虽然TDD中T（测试）是第一个字母，但是：

- TDD是一项开发活动，而不是测试活动；
    
- 测试是手段，设计是目标。
    
      
    

### **TDD的过程**

  

TDD的过程 = TDD的三个步骤 + TDD的三条规则

  
首先，我们看一下TDD的三个步骤，如下图所示：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

1. 添加一个测试，测试失败（变成红色）；
    
2. 快速使测试通过（变成绿色）；
    
3. 优化设计（变成蓝色）。
    

  

其次，我们了解一下TDD的三条规则：

1. 不允许写任何产品代码，除非是为了让失败的测试用例能通过；
    
2. 不允许写更多的产品代码，只要刚刚让失败的测试用例通过即可；
    
3. 不允许写更多的测试代码，只要刚刚让测试失败即可，编译失败也算失败。
    

  

在TDD中，我们按照**红-绿-重构**的节奏进行开发，测试与重构是相辅相成的：没有测试，重构只能是提心吊胆的实施；没有重构，测试会变得越来越不好写。

  

众所周知，TDD落地的难度很大，一个原因是观念上的差异，另一个原因也是更重要的是写测试就像老虎吃天一样无处下手，因此有一些实践TDD的同学会将TDD解释为**To Difficult to Do**。写测试难，一般是因为需求太大了，所以我们要拆分需求，就是将一个大需求拆分成一个个具体的任务，针对一个具体的小任务再写测试就变得比较容易了。从测试出发考虑问题的这种思考方式，会彻底颠覆我们原有的工作习惯，甚至是为了测试而调整设计，结果是我们得到了一个更好的设计，因此很多懂TDD的同学会把TDD解释为**Test Driven Design**。

  

现在我们可以真正理解测试驱动开发中“驱动”的含义了：**首先“驱动”着我们把需求拆分成一个个的任务，然后“驱动”着我们给出一个可测试的设计，最后在具体的写代码阶段，又会“驱动”着我们不断优化写出来的代码**。

  

### **关键点**

经过这几年的产品开发，笔者已将TDD作为最常用的XP实践之一，感受比较深的关键点有：

  

1. **任务拆分**  
    任务拆分是TDD最难的基本技能之一，要求具备较高的软件设计能力，并非一日之功，需要长时间积累，因此有些同学调侃：TDD不难，但TDD基础很难。
    
2. **分离关注点**  
    一个测试用例关注一个问题，不要写大而全的用例，同时用例是黑盒的，用例之间彼此独立，每个用例要保证自己的前置和后置完备。
    
3. **小步快跑**  
    添加一个测试用例，快速使测试通过，小步、安全、灵活、流畅的持续重构。当测试失败时，就那么几行修改，通过走查代码就可以快速定位问题，可以真正做到debug free。  
    当在5分钟内解决不了测试失败的问题时，立即回滚，然后重新出发。  
    测试及时反馈，一直进行“红色->绿色->蓝色”的正向循环，人的奖励神经不断被刺激，长期处于兴奋中，经常忘记时间。
    
4. **用例要对产品代码非入侵**  
    just do no harm! 不要为了测试通过在产品代码里加各种预编译宏，不要为了测试通过给产品代码增加很多测试分支。我们要通过抽象和防腐层来解决测试问题，同时可以使用Fake、Stub和Mock技术。
    
5. **测试代码和产品代码一样重要**  
    产品代码的正确性有测试代码保证，那么测试代码的正确性谁来保证呢？当然是程序员自己。我们要把测试代码写得非常简单，让错误无处藏身。但实际情况是，很多程序员都不重视测试代码，写的测试代码可读性差，而且很长，非常难维护。我们要重视测试代码，让它保持简单、清晰、深合己意且富有表达力。我们要有一个好鼻子，当嗅到测试代码有坏味道时，要第一时间进行重构。
    
    关于测试代码的重构，笔者给大家推荐一本书，书名是《xUnit测试模式：测试码重构》。
    

  

3.

  

**BDD**

  

### **BDD的起源**

  

Dan North在实施TDD实践时，时常会有一些困惑萦绕脑海，这也是很多程序在实践TDD时的困惑：

- 陷入代码细节，而忘记系统是为了解决用户需求
    
- 较多的关注How，而非What
    
- 单元测试写不完，不知道何时才算结束
    
- 大量的单元测试难以阅读及维护
    
- 传统单元测试与实现耦合太深，往往一个函数或方法就对应一个或多个测试
    
- 过度追求单元测试带来的覆盖率指标，可能会延误开发进度
    

  

软件架构中一个重要的概念就是分层，就是在一些模型的基础之上，再构建一些新的模型，测试也不例外。2003年Dan North受到DDD的影响 ，将TDD的层次拉高，提出了BDD。软件变化的源动力在业务需求上，因此需要将TDD的层次提升到业务上，而校验业务的正确与否的就是业务行为，那么就将TDD中的测试改成行为，于是BDD就诞生了。

  

### **BDD是什么**

  

了解了BDD的起源后，需要探讨一下BDD是什么：

- 从改良TDD开始，到注重用自然语言测试系统行为，再到对整个开发流程的优化，尤其是前期与领域专家和测试人员的互动；
    
- BDD是一种敏捷式的开发流程，鼓励开发人员、测试人员和领域专家互相合作，在对话中搜集系统的行为与具体使用案例，并使用DDD统一语言来描述；
    
- 通过不断迭代与合作的过程，同时关注测试系统的What，而非How，我们可以更早的取得共识并专注在业务目标上。  
    

  

下面是一个BDD示例：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

从这个例子我们不难看出，BDD的测试用例有很强的可读性。即便我们不熟悉技术，单凭这段文字，我们也能看出这个用例想要表达的业务含义，所以说BDD测试用例更贴近业务。

  

虽然这个表述已经很贴近业务了，但它并不是自然语言描述，而是有一种特定的格式，其实这是一门领域特定语言（Domain Specific Language，简称 DSL），称之为Gherkin。

  

这种DSL的核心点就是“Given…When…Then”描述格式（简称GWT描述格式），对应一条用户故事的验收准则：

- Given是一个假设前提
    
- When表示具体的操作
    
- Then则对应着这个用例要验证的结果
    

  

Gherkin语言这层只提供了业务描述，需要有一层胶水将用例和实现关联起来，而这层胶水在Cucumber框架术语中称之为步骤定义（Step Definition）。

  

BDD用例是站在业务的角度来描述的，并且Given、When和Then都是独立的，可以自由组合，这就意味着一旦基础框架搭建好了，用户就可以使用这些基础语句来编写新的测试用例，甚至可以不需要技术人员参与。

  

### **BDD的延伸**

  

从上面BDD的示例可以看出，**BDD的用例和普通测试的用例只是在表述方式上有所差异，从结构上看，二者几乎是完全等价的。所以，只要你想，完全可以采用 BDD 的方式进行从单元测试到系统测试所有类型的测试**。

  

TDD有广义和狭义之分。广义TDD既包括单元测试驱动开发（Unit Test-Driven Development，UTDD），又包括验收测试驱动开发（Acceptance Test-Driven Development，ATDD）。我们通常说的TDD是狭义的，对应UTDD。

  

我们在实践TDD的时候，可以单元测试分为两个层级：

- 一种是故事级别的UT，也可以叫端到端UT，或者叫FT（Functional Test，功能测试），这时用BDD的用例风格来表达用例故事的验收准则，实现快速反馈和贴近业务，对应ATDD；
    
- 一种是任务级别的UT，测试的是一个比较稳定的业务素材或技术素材，或者是基本素材的一个组合，这时仍可用GWT描述格式来表达任务的验收准则，实现快速反馈和改善设计，对应UTDD。
    

  

ATDD的步骤，如下图所示：

  
![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)  

  

可以看出，ATDD的步骤可以简记为4个D：

- Discuss讨论：敏捷团队和客户或者客户代表详细讨论用户故事，包含用户故事应有和不应有的预期行为；
    
- Distill提取：开发团队分析讨论中的条目，并提取成验收这些行为的测试，整个团队应充分认识“完成”对用户故事的意义，这正是验收标准所在；
    
- Develop开发：团队开发测试代码和产品代码以实现产品特性；
    
- Demo示范：产品特性实现后，团队向客户或客户代表演示以获得反馈。
    
      
    

ATDD与TDD（UTDD）协作，如下图所示：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)  

这里需要强调三点：

- 每个用户故事的验收准则有多条（故事级别to do list），每条验收准则对应一个ATDD用例；
    
- ATDD用例一般粒度比较大，很难做到小步快跑，这时我们需要针对该ATDD用例的功能实现再做任务拆分，拆分后的任务要有自己的验收准则，每个任务的验收准则可以有多条（任务级别to do list），每条验收准则对应一个UTDD用例；
    
- ATDD用例写完后，测试变红，直接无法进行UTDD，可以暂时将该用例注掉，待UTDD驱动完该ATDD用例的所有产品代码后再打开该ATDD用例进行验证，验证通过后再做ATDD层次的重构。
    

  

如果任务拆分的不合理，则会导致UTDD用例的维护成本较高，所以任务拆分的前置基础是需要具备高质量的软件设计技能。我们要充分权衡成本收益比，从而找到小步快跑的最佳节奏。

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

4.

  

**DDD**

  

### **DDD的起源**

一直以来，我们按照传统的方式开发软件，如下图所示：

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

分析模型和设计模型的分离，会导致分析师头脑中的业务模型和设计师头脑中的业务模型不一致，通常要映射一下。伴随着重构和bug fix的进行，设计模型不断演进，和分析模型的差异会越来越大。有些时候，分析师站在分析模型的角度认为某个需求较容易实现，而设计师站在设计模型的角度认为该需求较难实现，那么双方都很难理解对方的模型。长此以往，在分析模型和设计模型之间就会存在致命的隔阂，从任何活动中获得的知识都无法流畅的传递给另一方。  

  

Eric Evans在2004年出版了领域驱动设计（Domain-Driven Design，DDD）的开山之作《领域驱动设计：软件核心复杂性应对之道》，**抛弃将分析模型与设计模型分离的做法，寻找单个模型来满足两方面的要求，这就是领域模型**。许多系统的真正复杂之处不在于技术，而在于领域本身，在于业务用户及其执行的业务活动。如果在设计时没有获得对领域的深刻理解，没有通过模型将复杂的领域逻辑以模型概念和模型元素的形式清晰地表达出来，那么无论我们使用多么先进、多么流行的基础设施，都难以保证项目的真正成功。

  

### **DDD的根基**

  

DDD作为一种软件开发方法，它可以帮助我们设计高质量的软件模型。在正确实现的情况下，我们通过DDD完成的设计恰恰就是软件的工作方式。

  

学习DDD，就要掌握DDD的根基：统一语言（Ubiquitous Language，UL）和模型驱动设计（Model-Driven Design，MDD），而领域驱动设计的过程，就是建立起统一语言和识别模型的过程。

  

#### **统一语言**

  

业务方和技术方一起创建一套适用于领域建模的统一语言，统一语言必须在团队范围内达成一致，最常用的实践方法是**事件风暴**。所有成员都使用统一语言进行交流，每个人都能听懂别人在说什么，统一语言也是对软件模型的直接反映。业务方和技术方在一起工作，这样开发出来的软件能够准确的表达业务规则。领域模型基于统一语言，是关于某个特定业务领域的软件模型，如下图所示：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  
  

5.

  

**MDD**

  

有了统一语言后，就可以进行MDD了，而MDD一般分为两个阶段：战略设计（Strategic Design）和战术设计（Tactical Design）。

  

##### **战略设计**

战略设计属于高层设计，一般包括子域划分、限界上下文识别和上下文映射。

  

从广义上来讲，领域（Domain）是一个组织所做的事情以及其中所包含的一切，表示整个业务系统，对应着要解决的问题。软件开发要解决问题，而解决问题要分而治之。所谓分而治之，就是要把问题分解了，对应到领域驱动设计中，就是要把一个大领域分解成若干的小领域，而这个分解出来的小领域就是子域（Subdomain）。子域的划分其实就是对统一语言的分类，而对于一个真实项目而言，划分出来的子域可能会有很多，但并非每个子域都一样重要。所以，我们还要把划分出来的子域再做一下区分，分成核心域（Core Domain）、支撑域（Supporting Subdomain）和通用域（Generic Subdomain）。

  

有了划分出来的子域后，如何组织这些子域从而落地到解决方案中？这就引出了领域驱动设计中的一个重要的概念，限界上下文（Bounded Context，BC）。限界上下文，顾名思义，它形成了一个边界，一个限定了通用语言自由使用的边界，一旦出界，含义便无法保证。BC是一个显式的边界，领域模型便存在于这个边界之内。领域模型是关于某个特定业务领域的软件模型。通常，领域模型通过对象模型来实现，这些对象同时包含了数据和行为，并且表达了准确的业务含义。由于“领域模型”包含了“领域”这个词，我们可能会认为应该为整个业务系统创建一个单一的、内聚的和全功能式的模型。然而，这并不是我们使用DDD的目标。正好相反，领域模型存在于BC内。

  

有了BC后，BC间要集成，这时上下文映射（Context Map）就要发挥作用了。

  

常见的上下文映射模式如下：

- 合作关系（Partnership）；  
    
- 共享内核（Shared Kernel）；
    
- 客户方-供应方开发（Customer-Supplier Development）；
    
- 遵奉者（Conformist）；
    
- 防腐层（Anticorruption Layer）；
    
- 开放主机服务（Open Host Service）；
    
- 发布语言（Published Language）；
    
- 分离方式（Separate Ways）；
    
- 发布/订阅事件（Publish/Subscribe Events）；
    
- 大泥球（Big Ball of Mud）。
    

  

##### **战术设计**

  

战术设计属于低层设计，就是如何具体地组织不同的业务模型。

在BC内为了凸显领域模型，进而分离业务复杂度和技术复杂度，DDD提出了分层架构，主要包括用户界面层，应用层，领域层和基础设施层。对于DDD分层架构感兴趣的同学，可以看看作者之前写的一篇文章《DDD分层架构的三种模式》。

  

战术设计包含了很多设计元素，比如聚合，实体，值对象，领域事件，领域服务，应用服务，工厂，仓储等。

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

我们要借助这些设计元素，先建立正确的领域模型，再基于面向模型的实现模式来落地代码，准确的表达领域模型。

### **DDD的过程**

**![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)**

  

**DDD的主要过程：**

- 业务方梳理需求；
    
- 业务方和技术方一起定义统一语言，同时尝试用统一语言来描述需求；
    
- 业务方和技术方一起使用MDD建模，先战略建模（确定子域、BC和Context Map），再战术建模（确定分层架构和领域模型），同时消化统一语言，更深刻的理解业务；
    
- 技术方的代码实现要受到领域模型的约束，不能随心所欲跟着感觉走，同时基于面向模型的实现模式来准确的表达领域模型，让软件的实现复杂度逼近业务的本质复杂度。
    

  

可以看出，**代码实现“交付”了领域的业务需求，并且代码实现是领域的业务需求“驱动”出来的**。

  

统一语言及其背后的领域模型将业务方和技术方从各自的领地拉到了一个中间地带，也就是说统一语言与领域模型既不完全属于业务方，也不完全属于技术方，而是双方共有的部分。于是技术方与业务方就都有了话语权，至少有了可以沟通和协同的空间。

  

一旦业务方接受了统一语言，实际上就是放弃了对业务 100% 的控制权，也就意味着统一语言在业务上能够赋予技术方更大的控制权。统一语言在给技术方带来额外的权利同时，也隐含着额外的义务，即借助统一语言，在提炼知识的循环中，要接受业务方的监督与反馈，那么技术方也就丧失了对代码 100% 的控制权。

  

DDD使得技术方像业务方一样也受到重视，可以和业务方结对，名正言顺而不是低三下四的去学业务知识，从这个层面讲，DDD发展了敏捷；DDD将面向对象的分析模型和设计模型合一成领域模型，而不再割裂，从这个层面讲，DDD发展了面向对象。

  

DDD是一种模型驱动的设计方法，但我们首先强调的并不是模型的好坏，而是模型与软件实现间的关联。因为模型无论起点多么低，只要能够持续改进，总有一天就会有好的结果。

  

关联模型与代码实现，最终的目的是为了达到这样一种状态：修改模型就是修改代码；修改代码就是修改模型。如果不关联模型与代码实现，就会慢慢将模型本身分裂为更接近业务的分析模型和更接近实现的设计模型。这个时候，分析模型就会逐渐退化成纯粹的沟通需求的工具，而一旦脱离了实现的约束，分析模型就会变得天马行空，不着边际，成为纯ppt模型。

  

理解了DDD的本质后，当我们遇到复杂的领域问题，选择DDD就是一种自然而然的选择，而不是为了DDD而DDD。业务方和技术方高效协同，无障碍沟通，共同打造一个兼顾易理解性和低成本响应变化的领域模型，同时不断演进和精炼。

  

6.

  

**融合思路**

  

我们梳理一下TDD、BDD和DDD的关系：

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

我们知道，一个业务需求可以拆分成多个业务场景，而一个业务场景就是UML中的一个Use Case，对应DDD中的一个App Service。所谓的任务拆分，就是针对App Service的功能实现进行拆分，而拆分后的子任务与DDD中的设计元素一一对应，如下图所示。

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

当DDD遇上TDD，我们看看全景是什么样的：  

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

将TDD、BDD和DDD嵌入到软件的研发流程中：

- 在需求分析阶段：使用DDD消化领域知识，建立统一语言；
    
- 在架构设计阶段：使用DDD战略设计，分解问题域的复杂度和解决方案域的复杂度；使用DDD战术设计分离业务复杂度和技术复杂度，建立领域模型；使用BDD方法得到用户故事的验收准则，确定ATDD用例（故事级别to do list）；
    
- 在代码开发阶段：在分层架构中使用TDD方法来驱动DDD代码落地，包括ATDD流程和UTDD流程。
    

7.

  

**小结**

  

TDD、BDD和DDD不是互相取代的关系，而是互补增强的。关于反馈速度、改善设计和贴近业务三个重要因素，TDD、BDD和DDD侧重点各不相同，我们要掌握它们的目的、核心价值和指导原则，并不断丰富工具箱。

  

BDD是TDD的改良和延伸，使得ATDD有章可循。广义TDD包括UTDD和ATDD，狭义TDD仅包括UTDD。当团队实践TDD时，可以将TDD与DDD有效结合起来，以便提高生产力。对于软件设计能力不强的团队，建议先从ATDD做起，等时机成熟，再尝试ATDD+UTDD。

  

当DDD遇上TDD，可以碰撞出很多火花，期待上演很多精彩的故事！

  

  

**关于我们**

  

  

CPP与系统软件联盟是Boolan旗下连接30万+中高端C++与系统软件研发骨干、技术经理、总监、CTO和国内外技术专家的高端技术交流平台。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

2022年3月11-12日，「全球C++及系统软件技术大会」将于上海隆重召开，“C++之父”Bjarne Stroustrup将发布主题演讲。来自海内外近40位专家将围绕包括：现代C++、系统级软件、架构与设计演化、高性能与低时延、质量与效能、工程与工具链、嵌入式开发、分布式与网络应用 共8大主题，深度探讨最佳工程实践和前沿方法。大会官网：www.cpp-summit.org

  

  

  

**点击此处“阅读原文”获取大会演讲全套PPT**

领域驱动设计1

测试驱动开发1

架构设计1

阅读原文

阅读 1600

​