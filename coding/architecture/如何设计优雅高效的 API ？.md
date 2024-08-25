# 

原创 Ivan Čukić CPP与系统软件联盟

 _2022年03月24日 20:00_

  

  

  

**点击蓝字**

**关注我们**

  

  

**特|邀|嘉|宾**

![图片](https://mmbiz.qpic.cn/mmbiz_png/YQ55UxynrohtrKIOdm2rBavDgdnl5pFmh5hwFvxpGBuEoBqlrGnsHH9HIsGYVgjLqz0hrw84SBiaWUN4uTd8Mrw/640?wx_fmt=png&wxfrom=13&tp=wxpic "WechatIMG18.png")

**Ivan Čukić**

**KDAB高级工程师、《C++函数式编程》作者**

Ivan Čukić 是《C++函数式编程》一书的作者，KDAB高级工程师，同时是最大的C++自由软件项目KDAB的核心开发者，他也是自由软件的热爱者。他还在贝尔格莱德的数学系教授C++与函数式编程。他也是在贝尔格莱德大学获得他的博士学位。他经常受世界各地的技术会议邀请发表演讲。

  

**01**

**遥远的故事**

  

  

  

我们先来讲一个故事。这不是一个始于 C++ 大陆的故事。

  

通常当我们思考以命令式编程语言像Java、Python等进行软件开发时，我们的程序中有一个全局状态，通常被称为“世界”。如果你需要改世界中的什么东西，就可以直接改，这没什么问题。我们有状态，就可以改状态；但通常在纯函数式编程世界中，你不能改一个已经被设置过的值，所以如果你想在纯函数世界中做一个改变，就得创造一个应用了该改变的新世界, 你无法改变旧的那个世界。

  

如果你想要创造几个不同的平行世界，就像在漫画中、哲学中那样，你做的每一个动作都会创建一个平行世界，你可以有不同的线程或者程序的不同部分来创建不同的平行世界，一切都能正常运行。

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

所以你在你的程序中执行的每一个动作都会创建一个应用了那个改变的先前世界的拷贝，这也意味着我们可以有一打平行的、略有差异的世界存在于我们的软件系统中。而现在通常我们不会关心所有的平行世界，我们只关心单一的一个世界。  

  

那么我们关心单一世界这个观点适用于命令式编程语言，也适用于函数式编程语言。函数式编程世界的名人之一 Philip Wadler 写过一篇名为《线性类型可以改变世界》的论文, 其观点是：在函数式编程语言中人们被迫创建世界，是因为他们无法改变世界。

  

他想到了如何让纯语言允许变化，虽然这听起来有一点矛盾，他在论文中提出的新观点是引入“线性类型“。当你有一个线性类型的值时，如果该值只能被使用一次，如果你想在世界中做一个改变，就会消耗原来的值，并假装你创造了一个新的值和新的世界，而实际上你在幕后改变了旧的世界。

  

所以从语言的角度来看，这仍然是一个纯函数。你并没有正式改变什么，但实际上在幕后你已经改变了世界。

  

**02**

**克隆人的进攻**

  

  

  

现在的问题是，为什么我想在 C++ 大会上谈论“克隆人的进攻”这个话题。我的意思是，在 C++ 中，很显然我们是可以改变我们的状态的，只要不写 const。但 C++ 有些别的问题可以用线性类型来解决。就我所知，现在 C++ 最受开发者欢迎的部分是 ｝。

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

每当你在C++会议上问起某人 C++ 中最强大的字符，人们会回答是 ｝, 因为当我们到达 ｝时，我们触发了“资源获取即初始化”机制，于是我们释放了在代码块中被占用的所有资源。那如果我们做恰当的内存管理，套接字管理和文件管理等，单个代码块中所有值的析构函数都将被调用，所有资源都将被释放。但有一种资源一旦使用，就永远不会被释放，那就是时间。

  

就是说, 随我们程序运行流逝的时间，我们永远不能拿回来了，再多的花括号都无法帮我们避免浪费时间。

  

而在 C++ 中，如果我们想做纯函数式编程，需要不断地创造拷贝，至少在 C++ 11 之前的旧 C++ 是这样，就算我们不再需要原始的对象，也都得每次创建一个拷贝。

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

不过幸运的是到了 C++ 11，我们有了一对真正重要的字符：&&。它使我们能告诉编译器旧的值不再需要了，所以不用创建一个拷贝，我们基本上只需照着 Philip Wadler 对 Haskell 及其他函数式编程语言的提议去做。我们来偷取旧的值，假装正创造一个新的值，但实际并没有。我们只是重用旧数据，并将它移动到新的对象中。

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

现在当人们谈论移动语义，通常是关于资源所有权，比如 unique_ptr 和其他类似的东西，或是关于优化，比如拷贝 vs 移动，在多数复杂类中复制通常比移动昂贵。有一个方面不怎么被提及，那就是移动语义也可以用来从 API 内部派文档的用场。一旦你看到移动语义出现在了函数签名中，就无需再写一段注释来说明什么。

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

所以如果你看到这样的一个函数，API 会告诉我们不管传什么进去这个函数，都不能再使用原来的值了。

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

如果你想说, 一个对象将不再被使用了，并且我们正调用这个对象的成员函数，那个成员函数是 &&，或者说是右值引用限定的，这意味着无论 this 指针指向什么内容，它们之后都不可用了。它不一定非得是不可用的，但我们别再指望能够使用由 this 指针所指向的旧的值了。

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

而显然，如果你看到这样一个函数返回右值引用，它告诉我们对象已经存在，而我们无需创建拷贝，我们可以从那个对象偷过来。

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

如果把这两个例子结合，我们能做的，比如说我们有一个函数，既接受一个右值引用，又返回一个右值引用，在多数情况下这意味着，我们会给出一个不会再用的对象，这个函数会对那个对象做一些事情，然后返回给我们一个该函数再也不会使用的值，那这就标明一种 API 设计它传达了，不会进行复制以及相关的事情。如果我们有的是寻常的函数像：type foo(type)，那我们就没法讲函数的相关行为了。我们将不能判断它是否在复制上花了时间。

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

所以问题是如何强制移动语义，如何表达一个不接受其他、只接受右值引用的函数模板。如果你只是写 template T 以及 T&&，这显然不会正好是个右值引用，这个现在正式被叫做“转发引用”，曾被称为“通用引用”——Scott Meyers 创造的名词。这个函数几乎可以接受任何东西，现在问题是我们如何能限制它只接受临时值或右值引用？

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

这是一张来自布拉格会议的照片，C++ 20标准在会上被确认并被再次投票，于是我们有了 concepts。

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

我说的 concept 你可以认为是你有个在编译期被执行的谓词，像 constexper bool 是用了一个更好的语法。

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

你写 concept，concept xxx = xxx，那只是检查某个类型是一个整数的普通的 concept，你不该这样用，因为 concept 是设计来表达一些更好的东西，不只是用来检查某个东西是否是你需要的类型。

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

那如果你想限制函数模板，我写 template < typename T > requires，我们可以为 T 类型指定一个谓词，T 需要满足谓词的条件才能让这个函数工作，现在问题是我们用哪个谓词来表达一个右值引用？

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

我们需要参考“引用折叠”的规则。如果你有一个右值引用 && 跟一个左值引用 &，那会得出一个左值引用。这里只有一种情形我们会得到一个右值引用，就是最后一个，如果在任何一点我们有一个左值引用，那么它就不会是一个 && 了，那么它就不会是一个右值引用了。

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

那如果你想限制一个函数模板，我们只需要写 template < typename T >，并且 require T 不是一个左值引用，这样就像我们对函数模板做了对普通函数一样的事，创建了一个只接受一个右值引用的函数模板，所以是临时对象或被移动的对象。

  

**03**

**API**

  

  

  

让我们回到API。标准库中最有用、最有名的函数之一是 std::getline。它有一个不幸的 API，迫使大量初学者去使用代替品做同样的事。代替品会接受一个 char* buffer，而不是接受一个字符串和一个输入流。std::getline 的问题是它接受两个参数。老实说这俩参数都是同时入参和出参。第一个参数，显然我们要从输入中读取，我们要改变输入，但第二个参数不那么明显。我们给 getline 传一个字符串，之所以要那个字符串是让 getline 有些预分配好的缓冲区去写入从输入中得到的字符串，所以，它比每次调用 getline 再分配一个新字符串有效得多。但如果API看起来是这样，如果我们有一个 getline 接收一个std::string 的右值引用，并返回一个 std::string 的右值引用，这将表达这个函数接收这个字符串，因为它需要消耗它使用旧字符串的五脏六腑，它将表达它，它会重用那个缓冲区来传值回来给我们，所以我们可以这样 getline cin 并移动旧字符串，于是我们跟编译器传达我们不再需要 s 的旧内容，无论 getline 返回什么，你都会把它赋给 s，所以 s 的旧值被放弃了。当函数返回时，它会给我们一个新的值。

  

根据标准，通常情况下，对“移动自”（移动源）的类型的唯一要求是它能够被析构，但所有同样类型，除了以上保证，也包含了它能被赋值。所以当你有一个对象在“被移动自”的状态时，有两种操作你总是应当实现，即使在标准中没有强制这点，你也总是应该为“移动自“的对象实现析构函数和赋值函数，因为这样的代码挺有用的。一个函数的结果被赋给了什么东西，你移动了旧值给这个函数，或移动别的东西给这个函数。

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

我们看一个例子。一种挺常见的情况是你串接一些字符串，如果你有一个输入流序列，并且你打算传入它进入一个基于范围的 for loop 中迭代操作，你可以只是调用 result.append() .append() .append()。我猜当闻名天下的 Sean Parent 看到像这样的 for loop 时会说这不必用 for loop，它有一个数据集，而且它串接数据集中的所有元素，所以应该用 accumulate。

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

现在我把语法稍微缩短了一点，使它更便于幻灯片播放，那它不是标准的 accumulate，因为它接收一个范围和一个序列的数据集，而不是一堆迭代器。这段代码写起来要比 for loop 好得多，因为 for loop 只会累加改变状态，只要写 accumulate 初始值是一个空字符串，并且 accumulate 一个输入流序列中的所有值，你就会得到串接的字符串。但这里的大问题是如果你这么做了，它会彻底破坏我们程序的性能。

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

那有一些原因，第一，在C++20之前，accmulate 看起来是这样的。我们会在 accumulate 里有一个 while loop，每一次迭代它会串接之前累积操作的 init 的值，跟现在序列中的当前元素。

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

依次如图所示，对于每一次迭代我们将用最近一次操作的结果创建一个临时字符串，下一次将会使用之前生成的字符串再去串接下一个临时字符串，在每一次迭代中总是串接一个又一个临时字符串，我们会创建一个新的临时字符串，并将所有数据从旧字符串复制到新字符串。.append() 比这个快得多，通常都不需要重新分配任何东西，因为通常字符串的容量大到足以容纳子结点完整的累积值。

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

所以 C++ 20 做的一件事是加入了 std::move( init )，我们不调用加号运算符，不创建一个拷贝，这么做是表示之前 init 的值我们不再感兴趣了，你调用 +操作符时可以随意处置。在 std::string 和许多其他类型中，+ 操作符是右值引用限定的，所以当你写std::move(init) + ... ，它实际会调用 .append()。

  

所以在 C++20 中，如果你已经升级了你的编译器和标准库，在 while loop 的每一次迭代中就将不再有完整的拷贝，只会有 .append() ，然后移动赋值操作，因为他们仍然有 init = ...，所以这应该会比之前的大大高效。在 C++ 中，拷贝是性能杀手，其它事情都好办，但创建大型数据的拷贝从来都不是一个好主意。那么，我们来看看能用它做什么，如何通过避免复制和做更多事情来提高性能。

  

**04**

**性能**

  

  

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

这是我们要检验的两个基本函数的老方法的速度，这里的API是可变化的，因此不像一个调用 .append() 的函数那么好。第二个函数使用通常的 + 运算符，大多数人会觉得第二比第一更易读，至少多数非 C++ 开发者。

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

现在，如果我们检查由这些函数产生的汇编输出，做个比较。可以看到在左手边有一个 .append()，它没有做任何复杂的事情，只是调用了一个 .append()，它通过调用构造函数构造返回值并返回。在右边，即使你不分析所有看到的汇编指令，也应该关注到 _Unwind_Resume，所以如果没有别的，除了拷贝，第二个实现是处理异常，这在左手边是看不到的。所以右边 + 操作符一定会比左边慢，除非你在整个项目中关闭异常。

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

那么我们试着比较一下，如果我们有 .append()，运用我们学到的，使用 std::move，如果我们能用一个 .append()，这意味着我们不再需要 s 的旧值，所以我们可以写 std::move(s) ， 而不是 s + ...

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

由这些代码生成的汇编实质上是一样的，右边没有复杂的操作，这两者之间的唯一区别是在左边将调用复制构造函数，而在右边将调用结果的 move 构造函数。所以 std::move(s) + ...甚至比 return s.append() 还好，因为它不需要拷贝旧值，它只是把旧的字符串移动到结果（指针操作），这挺不错。

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

有一段很不错的演讲来自我遇到过的最聪明的人之一——John Lakos 关于分配器的演讲。他说的某些点我不大认同，他说我们没法在 C++ 中有好的API设计，我们不能有函数式风格的 API，我们不能在 C++ 中有纯粹的 API，因为这总是会导致性能下降，原因是 C++ 里即使是构造一个值也可能会慢。他称如果你想要速度极快，那你应当这样写API：你就构造一个值，然后不断地去修改它，即使是构造函数也可能会慢。显然那不是他谈话的重点，他提出的分配器的提案挺好，但他提的观点有点让我一直放不下，我想找到即使需要性能也能写好的应用程序接口的办法。

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

因此我们需要分析下 C++ 如何从函数返回值。编译器可以进行有几个层次的优化，第一个是 RVO（返回值优化）。在有些情况下，如果你有一个名为 s 的局部变量，你可以从函数中返回 s，它不会真的在函数内部创建 s，但 s 将被实例化在调用者中，无论谁调用了你的函数，它都避免了为了在调用者中初始化值而创建一份返回值的拷贝。但是返回值优化不总是有效的，至少不是在所有可能的情况下它都是有效的，而且它是必须在最基本的情况下工作。

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

当 RVO 不能执行时，另一种方法是将返回值移动给调用者，这是标准中被引入一个 bug，如果你有一个类型为 T 的局部变量，但你返回 optional<T>，如果你只说返回 T，你会得到一份拷贝。但幸运的是人们注意到在这种情况下，性能上和情理上可以就推断作者要 return std::move(t)。你知道实现了 CWG1579 特性的编译器，即使你不写 return std::move(t)，它也会自动认为你想写的正是那样。所以不是一个接受 t 的构造函数并且创建一个 t 的拷贝，而是会调用接收一个右值引用的optinal的构造函数。

  

最后一种可能是创建一个结果的拷贝来初始化调用者的值（拷贝构造），这是我们无论如何都要避免的。

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

就像我在这个案例中说的，如果你用某个局部变量来构造 U，它将被移动构造。那问题是我们能否做出比 RVO 更好的东西， 因为 John Lakos 说过 RVO 对他而言也还不够好。

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

我们通常这样写返回值的方式，如果你想用函数式写法，我们接受值，也返回值。我们先和用 C++ 惯用写法来比较一下两者产生的汇编代码。

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

我们得到两份完全不同的汇编代码，需要注意的是我们有异常处理和拷贝构造，有很多右边没有的东西，这里的代码比右边的大几倍，很明显这是低效的。

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

我们不用 C++ 惯用手法，换成参数值和返回值，以右值引用，参数将会是 string&&，结果类型也是 string&&。

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

当我们比较这些汇编，它们看起来90%是一样的，只有一点细微的差别在图中蓝色部分，它调用了一个 append 函数，然后从函数返回。在左手边非纯函数的 API 会跳转到 append 函数，所以右边增加的实质只是一个从函数的返回，开销也没那么大，不管怎么样，这个返回都会在 append 中发生。

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

让我们看一个更真实的例子， 而不仅仅是 hell() 和 gree()。如果你有 accmulate() ，我们想要创建一些测试，所以我们希望能够使用accumulate来连接字符串，我们看到老版本的 accumulate 调用操作符，你传给它的任意函数作为操作符作用于 init 和 *first，我们会检查这 accmulate 产生的汇编，当我们传给它的是一个接受字符串值和返回字符串值的 lambda 的时候，我们将把它与 C++ 20 版本的 accumulate 做比较。它只有一个小改动，改成了 std::move(init)。

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

如果你看这两份汇编，会发现用了 std::move 的汇编将避免左边的一些操作，避免拷贝构造函数，它会用 move 构造函数替换它们，基本上就是这样，所以可以预期它会更有效率，但我们并不真的期望它会多么更有效率。

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

如果它切换到基于右值的 API，写一个 lambda 给 C++ 20版本的 accumulate，它不接受值，而接受一个字符串的右值引用，返回一个字符串的右值引用，然后我们将看到由它生成的汇编。这里有一件事需要注意，如果你打算使用这里的 &&，并在 lambda 的函数体中返回一个右值引用，我们不能使用 std::move(acc)+ ...因为不幸的是 .s.append 返回字符串的值，所以标准库并没有遵循我的返回右值引用的思路，它返回一个字符串值，如果它试图返回右值引用，那将是一个悬空引用，因为我们的函数返回一个 &&，标准函数返回一个值，但如果标准库函数是右值引用限定的，而 + 返回一个字符串的右值引用，那我们可以有和上一张幻灯片一样的代码，我们就可以有 std::move(acc) + x。

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

如果你比较生成的汇编，可以再次看到这是非常不同的，我们无法真正看懂左右两边的汇编而指出发生了什么，所以每当你不想或不能看出汇编的行为时，但你要做的是创建基准测试，所以不需要看汇编，让我用漂亮多彩的柱状图来表明刚才看到的不同方式速度的差别。

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

作为基线，我们将有 C++ 17版本的 accumulate，加上标准库的函数对象 std::plus，我们想看看在 C++ 20 中如何改进这些。所以如果你把 accumulate 换成使用移动语义的 C++ 20 版本，仍然采用 std::plus，总体你会有 0% 的优化。发生这种情况的原因是即使你有 std::move，当我们传参给 std::plus，std::plus接收的是值，这里我们不能期望有任何性能提升，因为 std::plus 无论如何都会拷贝字符串，但如果你把 std::plus 换成之前幻灯片中的 lambda，就是调用 std::move(init) + val 的一般 lambda，通过避免拷贝，避免 std::plus() 等等，写成一个标准 lambda 带来的性能提升将是非常非常明显的。

  

第二是由于我们已经用了 std::move(init)，如果我们创造一个 lambda，不以值接收累计的值，只接受一个值的右值引用 auto&& 或转发引用，那与前一个相比，性能提升完全是小巫见大巫。所以在每次累加的迭代中，移动操作会少一次，这在任何意义上都对我们没有帮助。但如果我们替换返回值会发生什么呢？我们也做了右值引用，而不是值字符串，就会出现图中第五根柱状图。这条线几乎看不见，这不是幻灯片上的小故障。

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

我们把它放大，然后把前两根大的做所有拷贝的柱状图拿掉，只显示最后三条，那么采用标准的基于值的 lambda 的 accumulate 是第一根柱状图，第二根用的是返回一个值但接收一个右值引用的 lambda，最后一根用的 lambda 接收和返回右值引用，参数和返回值结果，把它和之前那副图中正常的红色柱状图进行比较来看，这是一个巨大的性能提升。这和你在每次循环迭代中手动调用  .append 得到的性能是一样的。所以你可以有看起来是纯函数式的 API，你不关心是否有些东西改变了你的状态或者其他东西，你可以写我样优雅的 API 而不损失性能。

  

问题是当我们有一个返回类型时，不是一个值类型，而是一个引用类型，那这一切能有多安全。就像我说的，如果你试图返回 s + ，就会是一个悬空引用，所以我们需要额外注意写这样的 API 时不要在函数内部犯错误。

  

**05**

**安全性**

  

  

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

因此我们通常可以在三种地方使用引用作为参数和返回值，而且我们可以储存作为引用的结果参数，通过 && 传递参数一直相当安全，所以无需多虑，你总是可以创建一个接受 && 的函数。我们来看最后一个存储结果为 && 的例子。我得说，如果你几乎总是觉得不安全，或许应该一直把结果存储为值。中间第二个，如果你不注意的话，通过用返回会有一些问题。

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

所以我们可以参考一下标准，在包含了临时对象的创建点的完整表达式被求值的最后一步中所有临时对象被销毁，如果创建了多个临时对象，摧毁它们的顺序与创造它们的顺序相反，这意味着如果我们有函数式风格的 API，我们有一个值，并且调用了一打函数作用于该值，所有 s 调用函数 f() 作用于它，并且调用函数 g() 作用于 f() 的结果等，只有当到达分号时临时对象才会被摧毁。那么当所有的函数 f g h等都已经完成了，即使那些函数也依赖于以右值引用被传入的临时值，那个临时值会存活，直到所有函数退出。

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

所以如果你用这种签名写一个函数，一个函数接收一个右值引用返回一个右值引用，这个函数使用起来是完全安全的。如果你从函数返回的是你修改的 t，或者如果你返回一个全局对象的引用，或者对于某个函数 f 属于外部的对象的引用，所以在函数的各个部分，你返回这两个中的一个都会是完全安全的，只要不是返回临时局部变量。

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

如果你有一个函数只返回右值引用，只有当你返回的东西是全局的，或者对 f() 是外部的东西，或者是比 f() 生存更久的，才是安全的。

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

如果你用 converted_type&& 作为结果，但是你有一个类型是通过右值引用获得的，如果结果需要一个转换过的 t 的值，这就永远都不会安全。

  

那么我们在返回优化中看到的例子，当标准中发生 bug 时，你不应该返回一个右值引用。但这在正常可变化的丑陋（非函数式）的代码中还算好的，你还是需要创建一个转换类型的新实例，所以你仍然需要调用 converted_type 的构造函数，那如果你在这里使用值返回就不是问题。

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

所以有一些准则是: 考虑以右值引用返回，但是要小心不要做任何非法的事情，并且总是按值存储结果。

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

有一种情况很明显，如果你有一个 foo 函数返回右值引用，并且你会尝试使用基于范围的 for loop 在结果的一些字段中，所以调用 foo().value，这将是未定义行为，因为 foo() 返回的结果是临时的，临时变量的生存期不会被延长到 for loop 的全部迭代，它几乎会被立即销毁，你会在一个不存在的对象中访问一个叫做 .value() 的东西。

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

为了避免这种情况，每当你有一个函数返回一个右值引用，而你没有立即将它传递给另一个函数，就把它赋值给一个变量，在 C++ 20 我们可以这样来代替 foo().value()，就写 auto f = foo(); 然后对 f.value() 迭代，这将创建临时对象，比如储存它，并且其生命周期会扩展到和 f 存在的周期一样长。

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

你也可以使用一些额外的东西让你的生活更轻松点，你应该经常使用 clang-tidy，对我们重要的 clang-tidy 规则是 user-after-move，所以如果试图在某变量被移动后使用它，我们总是会收到警告；如果一些变量声明了而没有用，我们也应该打开警告。而对于错误处理，应该避免使用异常和类似机制。如果你想有好的函数式 API，你应该使用 optional<T> 或 excpected<T, E>。

  

  

**关于我们**

  

  

CPP与系统软件联盟是Boolan旗下连接30万+中高端C++与系统软件研发骨干、技术经理、总监、CTO和国内外技术专家的高端技术交流平台。

  

2022年4月23-24日，Boolan 线下精品公开课即将在上海万豪虹桥大酒店开讲！课程官网：www.boolan.com/workshop

  

- 世界顶级 C++ 开发技术权威 Scott Meyers 大师课 _Effective__系列_ 独家版权课程之**《C++嵌入式编程最佳实践》**
    
      
    
    **![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)**
    

  

- 著名系统内核专家、《软件调试》作者张银奎：**《Linux 系统内核开发与调试》**
    
      
    
    **![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)**
    
      
    

  

  

**点击此处“阅读原文”获取详细课程大纲**

阅读原文

阅读 1140

修改于2022年03月31日

​