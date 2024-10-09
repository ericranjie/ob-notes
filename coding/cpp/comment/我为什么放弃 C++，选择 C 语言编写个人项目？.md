IT服务圈儿

_2022年09月08日 17:30_ _江苏_

![](http://mmbiz.qpic.cn/mmbiz_png/DY5wiavf38z6zz5UTGncwycEWAHcFljyMHXqusdAx8GX12IaE7WU9UR7HfTYwMVeUtVuygDWpBlUBXgJBEXMaVA/300?wx_fmt=png&wxfrom=19)

**IT服务圈儿**

关注互联网前沿资讯，提供实用的学习资源。我们是有温度、有态度的IT自媒体平台。

17篇原创内容

公众号

【CSDN 编者按】一直以来，C 和 C++ 都是非常优秀的编程语言。不过，两种语言虽名称有些相似，但应用场景存在巨大的不同。对于 C 语言而言，其主要被用于操作系统、容器、物联网、数据库等领域的开发，而 C++ 则是开发桌面软件、图形处理、游戏、网站的最佳工具。在本文中，作者原以为 C++ 在开发基础设施时会更胜一筹，然而经过与 C 语言的尝试对比，发现事实并非如此。

原文链接：https://250bpm.com/blog:4/?continueFlag=c778b3c9525d12a012a35269d830ebcc

声明：本文为 CSDN 翻译，转载需注明来源。

作者 | Martin Sústrik

译者 | 弯月           责编 | 屠敏

出品 | CSDN（ID：CSDNnews）

**以下为翻译正文：**

首先声明，在整个职业生涯中，我一直在使用C++，而且在做大多数项目时，C++仍然是我的首选语言。

因此，在开始构建个人项目ZeroMQ（可伸缩的分布式或并发应用程序设计的高性能异步消息库）时，我也选用了C++，主要原因如下：

1. C++包含一些数据结构和算法库。如果使用C语言，我将不得不依赖第三方库，或者自己动手编写基本算法。

1. C++会强制我在编程风格上保持一些基本的统一性。例如，this参数不允许使用几种不同的机制将指针传递给正在处理的对象，而这个问题在C项目中很常见。同样，不可以明确将成员变量标记为私有，此外还有C++的一些其他特征。

1. 使用C语言实现虚函数非常复杂，会导致理解和管理代码的难度加剧。不过，严格来说，这个问题其实是上一个问题的一个子集，但我觉得有必要单独指出。

1. 最后，每个人都喜欢在代码的末尾自动调用析构函数。

然而，事到如今，我不得不承认C++是一个糟糕的选择。下面，我来解释一下原因。

首先，我的个人项目ZeroMQ是一个持续运行的基础设施，永远不应该出故障，永远不应该表现出未定义的行为。因此，错误处理至关重要，必须做到明确且严格。

然而，**C++的异常处理并不能满足我的需求**。如果程序不会出错，那么选择C++没有任何问题，只需将main函数包装在try/catch中，集中在一个地方处理所有错误。

如果你的目标是保证不会出现未定义的行为，那么C++的异常处理就会变成一场噩梦。由于C++解耦了异常的发生与处理，因此错误处理非常容易，但也造成了你几乎不可能保证程序永远不会运行未定义的行为。

在C语言中，错误的产生和处理是紧密结合的，在同一块源代码中。因此，在出错时很容易理解发生了什么：

```
int rc = fx ();
```

而在C++中，你只能抛出错误，却不清楚究竟发生了什么：

```
int rc = fx ();
```

问题在于，你并不清楚在哪里处理异常。处理错误的代码在同一个函数中会更加方便理解，尽管不太方便阅读：

```
try {
```

然而，我们来考虑同一个函数抛出两个不同的错误，结果会怎么样：

```
class exception1 {};
```

以下是等效的C代码：

```
...
```

相较之下，C语言更加方便阅读，而且编译器也会生成更高效的代码。

然而，C++的问题还不仅限于此。考虑某个函数会引发异常，但不会处理异常的情况。在这种情况下，错误的处理可以放到任何地方，具体取决于从哪里调用该函数。

针对不同的情况，采用不同的方式处理异常？这种方法听起来似乎很有道理，但很快就会变成一场噩梦。

在修复某个Bug时，你会发现许多其他地方也有相同的Bug，因为它们都复制了同一段错误处理代码。每当添加一个函数调用，就有可能增加一个新异常，如果调用函数的代码没有妥善处理该异常，就意味着增加了一个新Bug。

如果你还想坚持“没有未定义的行为”原则，就不得不引入新异常，以便区分不同的故障模式。但是，添加新异常就意味着，它会上升到不同的地方。你必须在所有地方添加相应的异常处理，否则就会出现未定义的行为。

看到这里，你可能想说：这就是异常的正确用法啊？

然而问题在于，\*\*异常只是一个工具，目的是用更系统的方式管理呈现指数增长的错误处理代码，但它并不能解决根本的问题。\*\*甚至可以说，异常有可能导致情况恶化，因为你不仅需要编写新的异常类型，还需要针对新类型编写异常处理代码。

考虑到上述问题，我决定使用C++，但不使用异常。如今我的这个项目就是这样实现的。

不幸的是，问题并没有就此止步……

考虑一下，如果对象的初始化失败，会发生什么？构造函数没有返回值，因此只能通过抛出异常来报告失败。但是，我决定不使用异常。所以，我们必须像下面这样处理：

```
class foo
```

在创建实例时，会调用构造函数（这个函数不会失败），然后调用init函数（这个函数可能会失败）。

与C语言相比，C++代码更复杂：

```
struct foo
```

然而，C++代码真正的问题在于，如果开发人员在构造函数中编写一些代码，会发生什么？

在这种情况下，会出现一个特殊的新对象状态。由于对象已构造，但尚未调用init函数，因此是“半初始化”状态。我们应该修改对象（特别是析构函数）来处理这个新状态。这意味着，给每个方法添加新条件。

有人可能想说，这还不是因为你人为地添加了不使用异常的限制？！如果构造函数中抛出异常，C++运行时会正确地清理对象，不会出现“半初始化”状态。

话虽如此，然而问题在于，如果使用异常，如上所述，就必须处理所有与异常相关的复杂性。对于一个需要在遇到故障时表现出优秀的健壮性的基础设施组件来说，这不是一个合理的选择。

此外，即使初始化没有问题，对象的销毁也绝对会遇到问题。你不能在析构函数中抛出异常。这可不是我强加的人为限制，而是因为如果在进程中调用析构函数，或者恢复栈时恰好抛出异常，就会导致整个进程崩溃。

因此，如果销毁可能失败，你就需要两个单独的函数来处理它：

```
class foo
```

这就遇到了与初始化相同的问题：一个“半终止”状态，我们必须以某种方式处理，向各个成员函数添加新条件。

```
class foo
```

与之相比，C语言的代码如下。其中只有两种状态。未初始化对象/内存，我们无需担心上述问题，而且结构可以包含任意数据。而且只要对象进入已初始化的状态，就可以正常工作。因此，对象中不需要状态机：

```
struct foo
```

考虑一下，如果在上述代码中添加继承，会发生什么。C++允许将基类初始化为派生类构造函数的一部分。如果抛出异常，就会破坏已成功初始化的对象：

```
class foo : public bar
```

然而，一旦引入单独的init函数，状态的数量就会开始增长。除了未初始化、半初始化、初始化和半终止状态之外，你还会遇到这些状态的组合。你可以想象一个基类已完全初始化、但派生类半初始化的对象。

对于这样的对象，几乎不可能确保其行为不出问题。对象的半初始化和半终止部分有很多不同的组合，并且鉴于它们只在非常罕见的情况下才会引发故障，因此大多数相关代码可能未经测试就进入了生产。

综上所述，我认为，如果你的需求是不允许出现未定义的行为，则不适合面向对象的编程。这个问题不仅限于C++，任何具有构造函数和析构函数的面向对象语言都不适合。

因此，更适合面向对象语言的项目是：对开发速度有要求、但对“不存在未定义的行为”没有太高要求。

这个问题没有灵丹妙药。系统编程选择C语言更为合适。

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

1、[有了这个库，以后再也不用写正则表达式了！](http://mp.weixin.qq.com/s?__biz=MzUyMTY4MTk2OA==&mid=2247604326&idx=3&sn=13b3de83ae3b9c8ab254f3967495388b&chksm=f9d47539cea3fc2f6635e9f26905c1f951cf147b95cbd286676357c0955d31e67d7b97f11ea2&scene=21#wechat_redirect)

2、[React：搞了半天，我才是低代码的最佳形态](http://mp.weixin.qq.com/s?__biz=MzUyMTY4MTk2OA==&mid=2247604261&idx=3&sn=8e33d4d92aa003aa7bd0085e30b9e1ae&chksm=f9d4757acea3fc6c91b3abeb22020a1c31476b04299c3e27d6f518b111b531ece8f0802f78e0&scene=21#wechat_redirect)

3[、](http://mp.weixin.qq.com/s?__biz=MzUyMTY4MTk2OA==&mid=2247604208&idx=3&sn=b7de06300131bc23d2bae128f0d886ae&chksm=f9d47aafcea3f3b9cb1b5a2ddbd97f2113b18e73b33eeb3df5034b659538f80822b07c8964b1&scene=21#wechat_redirect)[看了我常用的IDEA插件，同事也开始悄悄安装了...](http://mp.weixin.qq.com/s?__biz=MzUyMTY4MTk2OA==&mid=2247604208&idx=3&sn=b7de06300131bc23d2bae128f0d886ae&chksm=f9d47aafcea3f3b9cb1b5a2ddbd97f2113b18e73b33eeb3df5034b659538f80822b07c8964b1&scene=21#wechat_redirect)

4、[Docker 是怎么实现的？前端怎么用 Docker 做部署？](http://mp.weixin.qq.com/s?__biz=MzUyMTY4MTk2OA==&mid=2247604107&idx=3&sn=fea29ae57e46827962325592e1b58b9c&chksm=f9d47ad4cea3f3c2ceae239bd9c2807c0d5eb4eefbf2193eaaa97f6b562aa74ace77b843f626&scene=21#wechat_redirect)

5、[${} 和 #{} 有什么区别？](http://mp.weixin.qq.com/s?__biz=MzUyMTY4MTk2OA==&mid=2247604106&idx=3&sn=8be537923b7cd1c8d9c9226df0f9bc26&chksm=f9d47ad5cea3f3c3848e88fe221a4fbd76e22c6f7b29353297410d29e74f2a9a1dab3ac5fe2b&scene=21#wechat_redirect)

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

点分享

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

点点赞

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

点在看

Reads 1985

​

Comment

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/DY5wiavf38z6zz5UTGncwycEWAHcFljyMHXqusdAx8GX12IaE7WU9UR7HfTYwMVeUtVuygDWpBlUBXgJBEXMaVA/300?wx_fmt=png&wxfrom=18)

IT服务圈儿

LikeShareWow

Comment

Comment

**Comment**

暂无留言
