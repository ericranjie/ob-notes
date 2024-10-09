今天这篇文章，我想跟大家探索下Attributes这个概念。
如果你还没有听过这个概念，或是一知半解，没咋用过，那正好表明它处于一个被忽略或是低估的位置。
Meeting C++曾经对此做过一份调查，结果如下：
[https://mmbiz.qpic.cn/mmbiz_png/9XBBCfGaPEkV4xUJEK24xuSCjvpYz22jxD43SVIZGKFWmBUdibrKeZ0HINUW1U59pGy882ia4h5YUtpeoETb0EVg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1](https://mmbiz.qpic.cn/mmbiz_png/9XBBCfGaPEkV4xUJEK24xuSCjvpYz22jxD43SVIZGKFWmBUdibrKeZ0HINUW1U59pGy882ia4h5YUtpeoETb0EVg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

From Meeting C++ Community

可以看出，大概一千人填写了这份问卷，其中就有半数人表示从未用过Attributes，在被使用的Attributes当中，使用频率也相差较大。

你可能会认为，这些特性大都是针对写库或大型项目的人准备的，它们只是针对一些特定的场景进行优化，普通开发者几乎用不上。

然而，事实真的如此吗？

data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==

许多C++编译器不仅仅实现了语言的核心特性，还通过扩展提供了一些额外特性，比如gnu提供的\_\_attribute\_\_，msvc提供的\_\_declspec。

编译器可以根据这些扩展的特性进行一些优化，但由于这些特性和平台绑定，使用这些特性就会影响代码的可移植性。

因此，C++标准从C++11就开始把一些有用的扩展，慢慢添加到标准中来。

这些添加进来的扩展就叫做\*\*「C++ Attributes」\*\*，标准对语法进行了统一，使用\[\[attr\]\]或是\[\[namespace::attr\]\]来指定普通的或是带有命名空间的Attributes。

那么为什么要采用新语法，而非引入新的关键字呢？

一是可以降低Attributes加入的障碍，二是可以防止关键字泛滥。

我们的大脑在处理事务时，是需要区分「背景」跟「主体」的，若所有的Attributes都被定为关键字，那么势必引发关键字泛滥。当一切都成为了主体，就相当于一切都是背景，突出不了重点。

打个比方：

在一个RPG游戏中，包含许多剧情，这些剧情不能整体都非常平淡，也不能整体都是高潮。因为我们对于这个游戏的整体记忆，取决于它剧情高潮和结尾时的体验。剧情越是跌宕起伏、有高有低，越能够给玩家留下深刻记忆，玩家也就越会倾向于评价这个游戏好玩。

游戏里的这些重点剧情就是「主体」，过渡剧情就是「背景」。

背景是为主体服务的，去除它并不会影响整个剧情。

同样，是否使用Attributes也并不会影响程序的语义，也就是说，即使编译器忽略一个Attribute也完全没有坏处。

顺便一提，在早些时候，override和virtual本来是作为Attributes引入的，后来发现语法又丑又极易被滥用，遂改为表示语言特性的关键字，而不是注解作用的属性。

data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==

**OCW属性模型**

这里跟大家介绍一套比较有用的属性工具：「OCW属性模型」。

[https://mmbiz.qpic.cn/mmbiz_png/9XBBCfGaPEnapONN0TauFH6ojXd2hq2yhkzibUGrNq2a6nuZYNajA2vEtTotF14P1k9xoVIJsfDmiavpDs29HmNw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1](https://mmbiz.qpic.cn/mmbiz_png/9XBBCfGaPEnapONN0TauFH6ojXd2hq2yhkzibUGrNq2a6nuZYNajA2vEtTotF14P1k9xoVIJsfDmiavpDs29HmNw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

这是我自创的一个模型，它可以帮助你更好地理解、记忆跟使用Attributes。

它包含了三部分，表示Attributes的三方面意义：

- **Optimizing（优化）**：对内存、并发、控制这些方面进行优化，提高性能。
- **Constraints（约束）**：对函数、变量、类这些代码进行限制，增加安全。
- **Warning（警告）**：对有意为之的代码产生的警告进行消除，避免误报。

下面具体来介绍一下。

第一部分，优化。指的是Attributes的目的是从内存布局、对齐、并发顺序、控制冒险等等这些可以提高性能的角度对代码进行优化。

例如，\[\[no_unique_address\]\]从内存的角度优化类中数据成员的存储形式，\[\[(un)likely\]\]从动态分支预测的角度来应对控制冒险，从而提高程序性能，\[\[carries_depency\]\]从内存顺序的角度来优化并发能力。

第二部分，约束。有句名言叫，「设计是为了厉行约束」。好的设计应该尽可能在编译期就发现大部分错误，约束就是保证用户的使用方式与你的设计意图相符合，一些Attributes提供了这方面的能力。

例如，最流行的\[\[nodiscard\]\]可以在用户忽略重要的函数返回值时，进行提醒。\[\[deprecated\]\]可以标记某个组件已被弃用，并告知用户新的替代品。

第三部分，警告。C++包含许多奇技淫巧，所以有些代码看似无用，其实不然。然而编译期会对这些有意的代码进行误判，给出警告，当然也有技巧去消除这些警告，但Attributes提供了更加规范统一的做法。

例如，\[\[fallthrough\]\]可以消除有意落空的case语句，就是故意省掉case中的break所导致的错误。\[\[maybe_unused\]\]可以消除未使用的变量警告。\[\[noreturn\]\]可以解决「调用不会返回的函数时」缺少返回值的错误。

简而言之，Attributes涉及三个方面，优化、约束与警告。在你编写代码时，若程序想要更多的提高，可以停下来思考一下：

针对每块代码可不可以从OCW给程序提高一些性能，让程序更加稳定。

data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==

**Optimizing 优化**

在前面的调查结果中，可以看到涉及优化的标准Attributes，使用率实在太低。

这可能有这些原因。第一，优化往往只针对于特定的场景，所以注定使用场景不多。第二，这类Attributes牵涉的知识最是广泛，想要正确使用本就不易，普通开发者更不会轻易使用。第三，这方面教程资料尚显匮乏，许多开发者没有意识去使用这些Attributes。

下来让我们先来对这些特性有了基本的理解。

首先来看\[\[no_unique_address\]\]，它使类数据成员可以拥有相同的地址。

有什么用呢？两点作用。

第一点，也是非常重要的一点，它为我们**提供了一种创建「0字节基类子对象」的方式。**

大家都知道，在C++中，类（class, struct, union）对象至少会占有1字节的大小，即使类为空。这会导致那些没有任何数据成员的类对象大小增加，比如最著名的是使用policy-based design时产生的额外开销：

`1**struct** **my_alloc** { **void*** **allocate**(**size_t** n) { **return** **nullptr**;} };23**template**<**class** **AllocPolicy**>4**class** **Foo** {5**private**:6    **int** i;7    AllocPolicy alloc;8};`

这里，虽然my_alloc为空，但依旧占用了1字节大小，再加上**tail padding**，所以Foo的大小由4字节增加到了8字节。

面对这种情况，一种解决办法是通过继承来使用策略类，而不再将它们作为数据成员。代码如下：

`1**template**<**class** **AllocPolicy**>2**class** **Foo** : AllocPolicy {3**private**:4    **int** i;5};`

每个类对象大小至少为1,对于基类的子对象依旧适用。**所以没有任何数据成员的基类子对象并没有必要增加派生类大小，此时基类子对象的大小为0**，Foo的大小是4字节。

**注：「基类子对象」是个术语，并不是指基类的子对象，而是指子类继承基类时，子类中所包含的基类所占的那部分内存。**

然而，这种方法有什么问题呢？

许多类并没有被设计成一个基类，因此将它们作为基类也许并不合适。

\[\[no_unique_address\]\]提供了另一种解决办法，这种方式要更加优雅：

`1**template**<**class** **AllocPolicy**>2**class** **Foo** {3    **int** i;4    [[no_unique_address]] AllocPolicy alloc;5};`

这里，基类子对象的大小为0，Foo的大小依旧为4。

说完了第一点，现在来说它的第二点作用，是**告诉编译器可以重复利用padding bytes存储其它数据。**

看如下代码：

`1**struct** **my_type** { **int** i; **char** c; };23**struct** **foo** {4    my_type var;5    **char** c[3];6};`

思考一下，foo的大小是多少？

foo中包含my_type，my_type中int占4字节，char占1字节，加上tail padding的3字节，共8字节；它还包含一个3字节的char，所以现在一共占11字节，于是再加上1字节的tail padding，最终一共占12字节。

这里面tail padding一共增加了4字节的开销，其实my_type的那3字节开销，可以给3字节的char使用，这样总共就只需要占用8字节。

\[\[no_unique_address\]\]可以实现这个目标，代码如下：

`1**struct** **foo** {2    [[no_unique_address]] my_type var;3    **char** c[3];4};`

**不过就我测试，gcc和msvc似乎都还没有这种优化，msvc甚至第一点作用也不支持。**

另外，这里还需要强调一下，\[\[no_unique_address\]\]只能应用于「非静态的数据成员」，所以不要试图在静态变量或是全局变量之上使用它。。

接下来，简单说下\[\[(un)likely\]\]和\[\[carries_dependency\]\]，由于这两个我打算单独写文章，所以这里只蜻蜓点水一下。

\[\[(un)likely\]\]其实包含两个：\[\[unlikely\]\]和\[\[likely\]\]，用于在分支代码中辅助编译器实现更加准确的「分支预测」。这到底有没有用呢？对性能提升有多大用呢？等我准备好资料数据单篇中来论。

\[\[carries_dependency\]\]这个是关于并发的优化，涉及我们讲过的Memory Order，还是单篇来说。

总之，优化这部分的Attributes的确有用，而且必不可少，在合适的场景还是推荐使用。

[https://mmbiz.qpic.cn/mmbiz_png/2xicCpQfvwsVIZmNqSS9WQBacMr2Vsq9nQZ1qSFI2U2Uvibe2I3joGwsgAg0ddHgbTiaymC4xDIkl8BlP9Sjl5UNw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1](https://mmbiz.qpic.cn/mmbiz_png/2xicCpQfvwsVIZmNqSS9WQBacMr2Vsq9nQZ1qSFI2U2Uvibe2I3joGwsgAg0ddHgbTiaymC4xDIkl8BlP9Sjl5UNw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

**Constraints 约束**

涉及约束的标准Attributes，只需小做努力，程序就能获得不错的安全性，以及更加清晰地表明接口的真实意图。

因而这成了使用率最高的一类Attributes。

其中\[\[nodiscard\]\]无疑又是最常用的，它的目的在于**显式地表达所定义接口的意义**。

可以用它来标记一个函数的返回值：

`1[[nodiscard]] **int** **foo**() {2    **return** 1;       3}45**void** **g**() {6    foo();7}`

那么当你在调用时忽略foo()的返回值，就会引发警告，如图。

data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==

这样的警告有效，不过很模糊，应该使用扩展版本\[\[nodiscard("reason")\]\]来说明原因：

`1[[nodiscard("the return value indicates a state of executing result. do not ignore it.")]] 2**int** **foo**() {3    **return** 1;       4}`

现在的警告要更加友好。

data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==

那么，可以在哪些地方使用它呢？

这里列举一些使用场景：

- 错误信息，error_code
- 工厂方法（若使用智能指针则没必要），因为其中会使用malloc/new这类函数分配内存。
- 返回的信息有特定作用，用户应该处理。即返回的是non-trivial类型，用户不处理可能引发资源泄漏之类的问题。
- 不使用返回值，通常会出现错误
- std::async()，不使用返回值会导致调用变成同步的

还有一种有趣的思路，对于那些你不想让用户使用的函数，为它加上一个\[\[nodiscard\]\]返回值。这样，就增加了此类函数的使用成本，亦即用起来变得麻烦了，用户便会趋于放弃它。比如给printf加一个 :D

所以，不是所有地方都适合添加该属性，到处乱加可能会适得其反。

其次流行的是\[\[deprecated\]\]，它表明弃用某个组件，组件可以是函数、变量、类等等，也可以直接弃用整个命名空间下的所有组件。

使用起来相当简单，代码如下：

`1[[deprecated]]2**void** **foo**() {}34**int** **main**() {5    foo();6}`

因为显式指定了\[\[deprecated\]\]，所以当你试图调用foo()时，编译器会给出警告。

[https://mmbiz.qpic.cn/mmbiz_png/9XBBCfGaPEmxR2YggWdKSdHb8fDppXS92uzbHe5V7kGRLbqib7Ej7Je5VNEpEzjpG5JjRNxaUlrxNicRlbUEHl4g/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1](https://mmbiz.qpic.cn/mmbiz_png/9XBBCfGaPEmxR2YggWdKSdHb8fDppXS92uzbHe5V7kGRLbqib7Ej7Je5VNEpEzjpG5JjRNxaUlrxNicRlbUEHl4g/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

当然，只是这样，用户可能会不明就里。所以同时，你应该说明弃用的原因，以及替代品。

这使用的是扩展版的\[\[deprecated("reason")\]\]，修改上面代码如下：

`1[[deprecated("foo() may be unsafe. Consider using foo_safe() instead.")]]2**void** **foo**() {}`

现在，指定的这个原因，将会在警告时出现。

[https://mmbiz.qpic.cn/mmbiz_png/9XBBCfGaPEmxR2YggWdKSdHb8fDppXS9k4jVl0hDyoqPwcbO3C90DfqGvexmib0gSG7DFxrwiatmdVic8Qic3z7uSA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1](https://mmbiz.qpic.cn/mmbiz_png/9XBBCfGaPEmxR2YggWdKSdHb8fDppXS9k4jVl0hDyoqPwcbO3C90DfqGvexmib0gSG7DFxrwiatmdVic8Qic3z7uSA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

总结一下，\[\[nodiscard\]\]比较有用，使用场景非常多，可以一定程度杜绝用户的错误行为；\[\[deprecated\]\]可以在放弃旧的接口时，告诉用户应该使用新的接口。

data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==

**Warning 警告**

涉及警告的Attributes最为简单易用，所以使用率也还不错。

其中，\[\[maybe_unused\]\]用于消除编译器的未使用变量警告。

比如，你在DEBUG时期可能会设置许多断言来检测错误，而当你编译RELEASE版本时，就可能会产生这个警告：

`1**void** **foo**() {2    **int** dummy = 1;3    assert(dummy == 1);4}`

使用非DEBUG模式编译上面代码，结果如图。

data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==

通过使用\[\[maybe_unused\]\]，便可以消除这个警告：

`1**void** **foo**() {2    [[maybe_unused]] **int** dummy = 1;3    assert(dummy == 1);4}`

其次，来看\[\[fallthrough\]\]，它的使用场景在于switch-case语句，也非常简单。

看如下代码：

`1**void** **test**(**int** state) { 2    **switch**(state) { 3    **case** 1: 4        std::cout << "1\\n"; 5        // 没写break; 6    **case** 2: 7        std::cout << "2\\n"; 8        **break**; 9    **default**:10        **break**;11    }12}`

因为没写break，编译器会给予提醒。

[https://mmbiz.qpic.cn/mmbiz_png/9XBBCfGaPEmANlctdrM18nSrBibSE31IOeJtovJnUR5SxpAZCMCficHdezpZhcFue7lXzkLBdxIuiaBR17u9ia9R8A/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1](https://mmbiz.qpic.cn/mmbiz_png/9XBBCfGaPEmANlctdrM18nSrBibSE31IOeJtovJnUR5SxpAZCMCficHdezpZhcFue7lXzkLBdxIuiaBR17u9ia9R8A/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

但有时我们是有意落空，例如：

`1**void** **foo**(**int** connState) { 2    **switch**(connState) { 3    **default**: 4        **if**(connection_timeout()) { // 如果连接超时 5            connState = reset_connect();   // 重置连接 6            [[fall_through]];    7        } **else** { 8            **break**; 9        }10    **case** LISTEN:11        ...12    }13}`

这里有一点需要注意，**\[\[fallthrough\]\]的下一条执行语句必须得是case标签。**

最后，有一个比较特殊的Attribute，就是\[\[noreturn\]\]。

为什么说它特殊呢？因为它有两点作用，一是消除警告，二是优化。

在一开始，需要明确一个观点，它并**不是表明函数没有返回值，而是表明函数的控制流不会返回到调用方。**

我用这一段代码来进行讲解：

`1// never return 2**void** **raise**() { 3    **throw** "error"; 4} 5 6**int** **f**(**bool** b) { 7    **if**(b) { 8        **return** 10; 9    } **else** {10        raise();11    }12}1314**void** **g**() {15    f(**true**);16    std::cout << "that is impossible." << std::endl;17}`

控制流永远不会返回到调用方，往往意味着程序遇到了错误，需要终结或抛出异常。

那么为何我要在警告这块讲解\[\[noreturn\]\]，而不是在优化那里呢？

一个很重要的原因就是，**优化并不是\[\[noreturn\]\]存在的主要目的，试想一下，这种「永远不会返回」的情况有多常见？几乎很少出现，所以通常来说也没有优化的必要。它更重要的目的在于，消除警告。**

看第10行代码，因为raise()永远不会返回，所以else分支也就没有必要写return。然而编译器并不知晓，它发现存在分支没有返回，于是给出警告：

[https://mmbiz.qpic.cn/mmbiz_png/9XBBCfGaPEmANlctdrM18nSrBibSE31IOWoqpIib4cKWr5d3SPldjNxZJ5SdBiaYv9NMfyqKq6jzwiaL2Sq1xwBAFQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1](https://mmbiz.qpic.cn/mmbiz_png/9XBBCfGaPEmANlctdrM18nSrBibSE31IOWoqpIib4cKWr5d3SPldjNxZJ5SdBiaYv9NMfyqKq6jzwiaL2Sq1xwBAFQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

因此\[\[noreturn\]\]的**作用就是告诉编译器，这个函数永远不会返回，所以其后的任何代码都不会得到执行，也就自然不需要返回语句了。**

对于上述代码，若调用f(false)，那么第16行代码永远也不可能执行到，这种代码尤其应当避免。

再稍微提一下，要小心使用\[\[noreturn\]\]，如果你的函数包含了一个while循环，之后你却无意识地打破了这个循环，程序的行为可能会变得非常怪异。

总而言之，警告这类Attributes使用起来比较简单，用处当然也不大，但当你遇到了上述问题，应该想到可以使用它们来进行解决。

通过OCW模型，我们可以知道，Attributes主要是为了帮助编译器勘测代码错误，提高程序性能。

其中最有用的要数优化和约束，在平时的项目中使用这些Attributes，可以使你的代码意图更加清晰，让你更好地掌控使用者的行为。

当你有性能需求时，试着去使用Attributes，这能帮助编译器更好地优化你的代码，生成更加高效的程序。
