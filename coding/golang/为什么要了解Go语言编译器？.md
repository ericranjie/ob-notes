# 

博文视点 博文视点Broadview

_2021年09月18日 15:30_

![图片](https://mmbiz.qpic.cn/mmbiz_png/PW0wIHxgg3nr1VNxfeqxVOw2nPJHVH4xeZibzPY5F4ibOuOZLMsUMrzIibGB6KMw7EurSKv6DkrtLzuhYdBa30A9Q/640?wx_fmt=png&wxfrom=13&tp=wxpic)

编译器是一个大型且复杂的系统，一个好的编译器会很好地结合形式语言理论、算法、人工智能、系统设计、计算机体系结构及编程语言理论。Go语言的编译器遵循了主流编译器采用的经典策略及相似的处理流程和优化规则（例如经典的递归下降的语法解析、抽象语法树的构建）。另外，Go语言编译器有一些特殊的设计，例如内存的逃逸等。

编译原理值得用一本书的笔墨去讲解，通过了解Go语言编辑器，不仅可以了解大部分高级语言编译器的一般性流程与规则，也能指导我们写出更加优秀的程序。很多Go语言的语法特性都离不开编译时与运行时的共同作用。另外，如果读者希望开发go import、go fmt、go lint等扫描源代码的工具，那么同样离不开编译器的知识和Go语言提供的API。

Go语言编译器的阶段

如图1-1所示，在经典的编译原理中，一般将编译器分为编译器前端、优化器和编译器后端。这种编译器被称为三阶段编译器（three-phase compiler）。其中，编译器前端主要专注于理解源程序、扫描解析源程序并进行精准的语义表达。编译器的中间阶段（Intermediate Representation，IR）可能有多个，编译器会使用多个IR阶段、多种数据结构表示代码，并在中间阶段对代码进行多次优化。例如，识别冗余代码、识别内存逃逸等。编译器的中间阶段离不开编译器前端记录的细节。编译器后端专注于生成特定目标机器上的程序，这种程序可能是可执行文件，也可能是需要进一步处理的中间形态obj文件、汇编语言等。

![图片](https://mmbiz.qpic.cn/mmbiz_png/4YNr10rzF0wmibpyQvFDhcsZpDpNuQr9ponVq75Q0TsrZekiciaibPJkZxRMRNyIG7HfEZU1TRfWt0OLkZIvk5fx2A/640?wx_fmt=png&wxfrom=13&tp=wxpic)

图1-1  三阶段编译器

需要注意的是，编译器优化并不是一个非常明确的概念。优化的主要目的一般是降低程序资源的消耗，比较常见的是降低内存与CPU的使用率。但在很多时候，这些目标可能是相互冲突的，对一个目标的优化可能降低另一个目标的效率。同时，理论已经表明有一些代码优化存在着NP难题，这意味着随着代码的增加，优化的难度将越来越大，需要花费的时间呈指数增长。因为这些原因，编译器无法进行最佳的优化，所以通常采用一种折中的方案。

Go语言编译器一般缩写为小写的gc（go compiler），需要和大写的GC（垃圾回收）进行区分。Go语言编译器的执行流程可细化为多个阶段，包括词法解析、语法解析、抽象语法树构建、类型检查、变量捕获、函数内联、逃逸分析、闭包重写、遍历函数、SSA生成、机器码生成，如图1-2所示。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

图1-2  Go语言编译器执行流程

词法解析

和Go语言编译器有关的代码主要位于src/cmd/compile/internal目录下，在后面分析中给出的文件路径均默认位于该目录中。在词法解析阶段，Go语言编译器会扫描输入的Go源文件，并将其符号（token）化。例如“+”和“-”操作符会被转换为_IncOp，赋值符号“:= ”会被转换为_Define。这些token实质上是用iota声明的整数，定义在syntax/tokens.go中。符号化保留了Go语言中定义的符号，可以识别出错误的拼写。同时，字符串被转换为整数后，在后续的阶段中能够被更加高效地处理。图1-3为一个示例，展现了将表达式a:=b + c(12)符号化之后的情形。代码中声明的标识符、关键字、运算符和分隔符等字符串都可以转化为对应的符号。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

图1-3  Go语言编译器词法解析示例

Go语言标准库go/scanner、go/token也提供了许多接口用于扫描源代码。在下例中，我们将使用这些接口模拟对Go文本文件的扫描。

```
package main
```

在上例中，src为进行词法扫描的表达式，可以将其模拟为一个文件并调用scanner.Scanner词法，扫描后分别打印出token的位置、符号及其字符串字面量。每个标识符与运算符都被特定的token代替，例如2i被识别为复数IMAG，注释被识别为COMMENT。

```
1:1     IDENT   "cos"
```

语法解析

词法解析阶段结束后，需要根据Go语言中指定的语法对符号化后的Go文件进行解析。Go语言采用了标准的自上而下的递归下降（Top-Down Recursive-Descent）算法，以简单高效的方式完成无须回溯的语法扫描，核心算法位于syntax/nodes.go及syntax/parser.go中。图1-4为Go语言编译器对文件进行语法解析的示意图。在一个Go源文件中主要有包导入声明（import）、静态常量（const）、类型声明（type）、变量声明（var）及函数声明。

![图片](https://mmbiz.qpic.cn/mmbiz_png/4YNr10rzF0wmibpyQvFDhcsZpDpNuQr9pzjleMx02t5K8Kuzjwibo1c0Vw4eSeXEqNm1ZoqwCPNeN7Q8XMq8JHyw/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

图1-4  Go语言编译器对文件进行语法解析的示意图

源文件中的每一种声明都有对应的语法，递归下降通过识别初始的标识符，例如_const，采用对应的语法进行解析。这种方式能够较快地解析并识别可能出现的语法错误。每一种声明语法在Go语言规范中都有定义

```
//包导入声明
```

函数声明是文件中最复杂的一类语法，因为在函数体的内部可能有多种声明、赋值（例如:= ）、表达式及函数调用等。例如defer语法为defer Expression ，其后必须跟一个函数或方法。每一种声明语法或者表达式都有对应的结构体，例如a := b + f(89) 对应的结构体为赋值声明AssignStmt。Op代表当前的操作符，即“ :=”, Lhs与Rhs分别代表左右两个表达式。

```
AssignStmt struct {
```

语法解析丢弃了一些不重要的标识符，例如括号“ (”，并将语义存储到了对应的结构体中。语法声明的结构体拥有对应的层次结构，这是构建抽象语法树的基础。图1-5为a := b + c(12)语句被语法解析后转换为对应的syntax.AssignStmt结构体之后的情形。最顶层的Op操作符为token.Def（:= ）。Lhs表达式类型为标识符syntax.Name，值为标识符“a”。Rhs表达式为syntax.Operator加法运算。加法运算左边为标识符“b”，右边为函数调用表达式，类型为CallExpr。其中，函数名c的类型为syntax.Name，参数为常量类型syntax.BasicLit，代表数字12。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

图1-5  特定表达式的语法解析示例

Go语言可执行文件的生成离不开编译器所做的复杂工作，如果不理解Go语言的编译器，那么对Go语言的理解是不完整的。Go语言的编译器、运行时，本身就是用Go语言写出的既复杂又精巧的程序；探究语言设计、语法特性，本身就是学习程序设计与架构、数据结构与算法等知识的绝佳途径。学习底层原理能够帮助我们更好地了解Go语言的语法，做出合理的性能优化，设计科学的程序架构，监控程序的运行状态，排查复杂的程序异常问题，开发出检查协程泄露、语法等问题的高级工具，理解Go语言的局限性，从而在不同场景下做出合理抉择。

编译阶段包括词法解析、语法解析、抽象语法树构建、类型检查、变量捕获、函数内联、逃逸分析、闭包重写、遍历并编译函数、SSA生成、机器码生成。编译器不仅能准确地表达语义，还能对代码进行一定程度的优化。可以看到，Go语言的很多语法检查、语法特性都依赖编译时。理解编译时的基本流程、优化方案及一些调试技巧有助于写出更好的程序。

目前市面上鲜有系统介绍Go语言底层实现原理的书籍，如果你想系统了解学习更多，今天给大家推荐这本新书书，书中系统性地介绍Go语言在编译时、运行时以及语法特性等层面的底层原理和更好的使用方法。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

本书语言通俗易懂，书中有系统权威的知识解构、精美的示意图，并对照源码和参考文献字斟句酌，在一线大规模系统中提炼出设计哲学与避坑方法，对于编译时、运行时及垃圾回收的精彩讲解弥补了国内的多项缺陷，这本罕见的诚意之作必将陪伴读者实现最艰苦的能力跨越，你想要的都会到来……

**内容简介**

**本书由21章组成，这21章可以分为6部分。**

- **第1～8章**为第1部分，介绍Go语言的基础——编译时及类型系统。包括浮点数、切片、哈希表等类型以及类型转换的原理。

- **第9～11章**为第2部分，介绍程序运行重要的组成部分——函数与栈。包括栈帧布局、栈扩容、栈调试的原理，并介绍了延迟调用、异常与异常捕获的原理。

- **第12、13章**为第3部分，介绍Go语言程序设计的关键——接口。包括如何正确合理地使用接口构建程序、接口的实现原理和可能遇到的问题，并探讨了接口之上的反射原理。

- **第14～17章**为第4部分，介绍Go语言并发的核心——协程与通道。详细论述了协程的本质以及运行时调度器的调度时机与策略。介绍了通过通信来共享内存的通道本质以及通道的多路复用原理，并探讨了并发控制、数据争用问题的解决办法及锁的本质。

- **第18～20章**为第5部分，介绍Go语言运行时最复杂的模块——内存管理与垃圾回收。详细论述了Go语言中实现内存管理方法及垃圾回收的详细步骤。

- **第21章**为第6部分，介绍Go语言可视化工具——pprof与trace。详细论述了通过工具排查问题、观察系统运行状态的方法与实现原理。

**本书作者**

郑建勋

Golang contributor（Go语言垃圾回收模块代码贡献者）、Go语言精度库shopspring/decimal核心贡献者。滴滴高级研发工程师。拥有丰富的分布式、高并发、大规模微服务集群的开发设计经验。

微信公众号“gopher梦工厂”作者，知名go语言原创博主，51CTO学堂高级讲师、极客时间“每日一课”讲师。有丰富的教育经验，想读者之所想。相信这部系统且深入浅出的作品，会是读者打怪升级的最佳辅助。

**专家力荐**

这是一本Go语言的初学者和进阶学者都可以受益的书。它不仅仅介绍了Go的语言特性，还深入这些特性背后的设计考量、编译器及语言实现的细节。授人以鱼和授人以渔在本书里面一起得到了体现。更难得的是，本书并没有粘贴大段的代码，而是以图文的形式将复杂的概念解释清楚，降低了阅读和理解的难度，使得读者不会望“底层”和“深入”二词而却步。

**——叶绍志博士  Shopee技术委员会主席、顺丰速运前CTO、Google前主任工程师**

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

如果喜欢本文

欢迎 **在看**丨**留言**丨**分享至朋友圈** 三连

**热文推荐**

- [书单 | 有这10本书，不怕黑客攻击了！](http://mp.weixin.qq.com/s?__biz=MjM5NTk0NjMwOQ==&mid=2651136491&idx=1&sn=8de968bc9dae5dc4fc6764343529059e&chksm=bd014bc18a76c2d75ca271d22075df2d4a6225711af17e5d1d5e2c7602d5faaf4e25ef88dcc4&scene=21#wechat_redirect)

- [分布式系统中协调和复制技术的原理](http://mp.weixin.qq.com/s?__biz=MjM5NTk0NjMwOQ==&mid=2651136417&idx=1&sn=2828875002bad2885963d429d4d0b031&chksm=bd014b8b8a76c29de2e2d7d453bde77279dca3eeb336a0e430e8e83b003d4c3f09066d2cc426&scene=21#wechat_redirect)

- [QQ浏览器背后的推荐AI中台 | AICon](http://mp.weixin.qq.com/s?__biz=MjM5NTk0NjMwOQ==&mid=2651136374&idx=1&sn=4fdd600cabef5a6c9b025fb3238156d4&chksm=bd014b5c8a76c24a7a0af409e3496c03133042290d6ab1484f3d1944dcb13c097044c7637126&scene=21#wechat_redirect)

- [数据中台建设的9大误区，你中了几条？](http://mp.weixin.qq.com/s?__biz=MjM5NTk0NjMwOQ==&mid=2651136372&idx=1&sn=df8dff1431ccaf1ca620d8d69ea39e00&chksm=bd014b5e8a76c248c8d560ee11517ea6d43abe8dc5e4ffc40b3e74f4a59a2f618247825c86e1&scene=21#wechat_redirect)

______________________________________________________________________

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

▼点击阅读原文，查看本书详情~

阅读原文

阅读 1117

​

写留言

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/PW0wIHxgg3lICy34AaLxSSBtOGFrm6eovZRP96ic72ibb6aTQiaYEeIPd1Jl7r7wia7Bh3v8HOmOQgCQUMaTicfROgQ/300?wx_fmt=png&wxfrom=18)

博文视点Broadview

2分享在看

写留言

写留言

**留言**

暂无留言
