原创 程序喵大人 程序喵大人
_2021年12月07日 08:24_

最近研究了一波RTTI，整理了一下知识点，在这里分享一下，下面是目录：

RTTI 是 Run Time Type Information 的缩写，从字面上来理解就是运行时期的类型信息，它的主要作用就是动态判断运行时期的类型。

一般在dynamic_cast和typeid中用到，例如父类B的指针转换子类A的指针，dynamic_cast会判断B究竟是不是A的父类，如果不是，会返回nullptr，相对于强转会更加安全。依据什么判断的呢？就是RTTI。

先看下面这段代码：

```
#include <iostream>
```

结果如下：

```
clang++ test_rtti.cc -std=c++11;./a.out
```

上面的代码是正常的一段使用多态的代码，同时也包含了子类指针转基类指针，基类指针转子类指针，从输出结果中可以看到，使用dynamic_cast进行不合理的基类子类指针转换时，会返回nullptr，而强转则不会返回nullptr，运行时肯定就会出现奇奇怪怪的错误，比较难排查。

如果在编译时加上-fno-rtti会怎么样？结果是这样：

```
clang++ test_rtti.cc -std=c++11 -fno-rtti
```

可以看到，加上了-fno-rtti编译时，使用typeid或dynamic_cast会报错，即添加-fno-rtti编译会禁止我们使用dynamic_cast和typeid。那为什么要禁止使用他们呢？

1. RTTI的空间成本非常高：每个带有vtable（至少一个虚拟方法）的类都将获得RTTI信息，其中包括类的名称及其基类的信息。此信息用于实现typeid运算符以及dynamic_cast。（大小问题大家可以自己编写代码验证一下）

1. 速度慢，运行时多判断了一层，性能肯定更慢一些。

tips：我这里又将typeid和dynamic_cast去掉重新编译，结果表明添加了-fno-rtti，还是可以正常使用多态，所以大家不用担心rtti的禁用会影响多态的使用。

都知道RTTI信息是存在于虚函数表中，而添加-fno-rtti后代表禁止了RTTI，那虚函数表中还会有rtti信息吗？

我这里使用clang的命令查看一下虚函数表：

```
clang -Xclang -fdump-vtable-layouts -stdlib=libc++ -fno-rtti -c test_rtti.cc
```

通过结果可以看到，即使添加了-fno-rtti，虚函数表中还是会存在RTTI指针，但是我查看很多文档都说rtti会导致可执行文件的体积增大一些（毕竟-fno-rtti最大的目的就是为了减小代码和可执行文件的大小），所以我估计指针指向的块里面可能什么信息都没有，具体就不得而知了。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

再稍微介绍下软件开发的基本原则，以下内容摘自《软件开发的201个原则》：

关于软件质量，业界普遍认为有**3**个决定性要素：人、过程和工具。如何基于这些要素提升代码的质量和开发效率，是软件工程研究者和实践者一直在努力的方向。

不同的公司有不同的文化背景，虽然开发不同的软件项目有不同的实践过程，但所要遵守的基本原则都是一样的。大量实践证明——

**了解软件开发基本原则的工程师，比那些不了解基本原则的，编写代码的质量和开发效率明显胜出一筹。**

**原则 1 质量第一\*\*\*\*QUALITY IS #1**

无论如何定义质量，客户都不会容忍低质量的产品。质量必须被量化，并建立可落地实施的机制，以促进和激励质量目标的达成。即使质量没达到要求，也要按时交付产品，这似乎是政治正确的行为，但这是短视的。从中长期来看，这样做是自杀。质量必须被放在首位，没有可商量的余地。Edward Yourdon 建议，当你被要求加快测试、忽视剩余的少量 bug、在设计或需求达成一致前就开始编码时，要直接说“不”。

**原则 7 尽早把产品交给客户\*\*\*\*GIVE PRODUCTS TO CUSTOMERS EARLY**

在需求阶段，无论你多么努力地试图去了解客户的需求，都不如给他们一个产品，让他们使用它，这是确定他们真实需求的最有效方法。如果遵循传统的瀑布式开发模型，那么在 99% 的开发资源已经耗尽之后，才会第一次向客户交付产品。如此一来，大部分的客户需求反馈将发生在资源耗尽之后。和以上方法相反，可在开发过程的早期构建一个快速而粗糙的原型。将这个原型交付给客户，收集反馈，然后编写需求规格说明并进行正规的开发。使用这种方法，当客户体验到产品的第一个版本时，只消耗了 5%～20% 的开发资源。如果原型包含合适的功能，就可以更好地理解和把握最有风险的客户需求，最终产品也就更有可能让客户满意。这有助于确保将剩余的资源用于开发正确的系统。

**原则 17 只要可能，购买而非开发**IF POSSIBLE, BUY INSTEAD OF BUILD

要降低不断上涨的软件开发成本和风险，最有效的方法就是，购买现成的软件，而不是自己从头开发。确实，现成的软件也许只能解决 75% 的问题。但考虑一下从头开发的选择吧：支付至少 10 倍于购买软件的费用，且要冒着超出预算 100% 且延期的风险（如果最后能够完成！），并且最终发现，它只能满足 75% 的预期。对一个客户来说，新的软件开发项目似乎最初总是令人兴奋的。开发团队也是“乐观的”，对“最终”解决方案充满了希望。但几乎很少有软件开发项目能够顺利运行。不断增加的成本通常会导致需求被缩减，最终研发出的软件可以满足的需求也许跟现成的软件差不多。作为一个开发者，应该复用尽可能多的软件。复用是“购买而非开发”原则在较小范围内的体现。

**原则 22 技术优先于工具**TECHNIQUE BEFORE TOOLS

一个没规矩的木匠使用了强大的工具，会变成一个危险的没规矩的木匠。一个没规矩的软件工程师使用了工具，会变成一个危险的没规矩的软件工程师。在使用工具前，你应该先要“有规矩”（即理解并遵循适当的软件开发方法）。当然，你也要了解如何使用工具，但这和“有规矩”相比是第二位的。我强烈建议，在投资于工具以对某项技术“自动化”之前，先手工验证这项技术，并说服自己和管理层：这项技术是可行的。在大多数情况下，如果一项技术在手工操作时不灵，那么在自动操作时也不灵。

**原则 37 要承担责任**TAKE RESPONSIBILITY

在所有工程学科中，如果一个设计失败，工程师会受到责备。因此，当一座大桥倒塌时，我们会问“工程师哪里做错了？”当一个软件失败了，工程师很少受到责备。如果他们被责备了，他们会回答，“肯定是编译器出错了”，或“我只是按照指定方法的 15 个步骤做的”，或“我的经理让我这么干的”，或“计划剩余的时间不够”。事实是，在任何工程学科中，用最好的方法也可能产出糟糕的设计，用最过时的方法也可能做出精致的设计。不要有任何借口。如果你是一个系统的开发者，把它做好是你的责任。要承担这个责任。要么做好，要么就压根不做。

## 福利

送六本书：《软件开发的201个原则》，豆瓣 9.1 分

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

活动规则：

**走心奖**：本篇文章留言，我会选出三位走心留言，另外经常留言+分享的同学我会重点考虑！

**幸运奖**：加我好友，12 月 9 号 20:00 我会发一条朋友圈，点赞方式抽奖，送出三本。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

土豪朋友们，也可以直接购买，购买方式在下方：

阅读原文

阅读 2752

​

写留言

**留言 35**

- 你猜呀

  2021年12月7日

  赞1

  喵大哥的公众号是目前我关注干货含金量最高的了，从中学到了很多，也给项目组分享学习，希望喵大哥能一如既往地推出更多好文，共同成长进步！加油💪![[强]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

  置顶

  程序喵大人

  作者2021年12月9日

  赞3

  送你哈，姓名电话地址加我微信发我吧

- AlwaysSunny丶

  2021年12月7日

  赞

  这本书看起来是基于大学学习的软件工程方法论的引申版，大学的那本书主要介绍的是软件工程里重要的原则、流程，还有模型，代码近似等于逻辑加工，软件工程就是将逻辑加工的更完美、更通俗易懂。但是不知道原则竟然还能有201种![[旺柴]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

  置顶

  程序喵大人

  作者2021年12月8日

  赞

  姓名电话地址私信我哈，送给你

  AlwaysSunny丶

  2021年12月8日

  赞

  哇，谢谢喵哥![[旺柴]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

- 云瑶

  2021年12月7日

  赞1

  这篇文章过几天看看,关于软件的也想多学学,这几天在阅读喵大人前段时间分享的关于内存的两篇文章,正好给第一篇做了个总结. 作者先从自己工作的案例开始分析,进程被莫名kill,用户管理层，C 运行时库层，操作系统层是内存泄漏的三个层面,一顿分析只能是glibc出了问题,顺便复习了几个知识点,比如栈区是从高地址向低地址生长,是唯一不需要映射，用户却可以访问的内存区域，这也是利用堆栈溢出进行攻击的基础,数据区只存在源代码中有预定义值的全局变量和静态变量,还有mmap 映射区域至顶向下扩展这玩意好久之前就忘了.,它也是向上增长的.然后涉及到如何调用:对于heap的操作，操作系统提供了brk()函数，c运行时库提供了sbrk()函数。对于mmap映射区域的操作，操作系统提供了mmap()和munmap()函数,我们要谨记内存的延迟分配，只有在真正访问一个地址的时候才建立这个地址的物理映射,在此之前只会给你分配一个线性区在glibc中，malloc就是调用sbrk()函数将数据段的下界移动以来代表内存的分配和释放。sbrk()函数在内核的管理下，将虚拟地址空间映射到内存，供malloc()函数使用。mmap是一种内存映射文件的方法,有

  置顶

  程序喵大人

  作者2021年12月7日

  赞

  送你一本

  云瑶

  2021年12月7日

  赞

  我还没写完.后面还有,内存那几篇真心不错..

  程序喵大人

  作者2021年12月7日

  赞

  可以总结出一篇文章给我投个稿![[呲牙]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

  云瑶

  2021年12月7日

  赞

  这种看别人写的总结也可以吗

  云瑶

  2021年12月7日

  赞

  可以我就试试

  程序喵大人

  作者2021年12月7日

  赞

  可以

  程序喵大人

  作者2021年12月8日

  赞

  加我微信，给我姓名电话地址哈

- 瑜

  2021年12月9日

  赞2

  首先为喵哥点赞，为大家分享好书！大多数人重视代码量，重视编码技巧，却忽视编码的工程属性，软件工程，软件工程，软件是离不开工程的，注重软件的工程属性，才能事半功倍，减少加班时间，保住为数不多的头发![[苦涩]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

- Z.

  2021年12月7日

  赞1

  原则很重要，无论做什么都需要有一个原则，有一个规范，作为一个马上出入职场的程序猿，希望可以通过此书规范自己的软件开发能力。喵哥快内幕我![[呲牙]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

- 李想

  2021年12月7日

  赞1

  喵哥，一人一本![[色]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![[色]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![[色]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

  程序喵大人

  作者2021年12月7日

  赞

  有心无力啊![[流泪]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

- Koala

  2021年12月7日

  赞1

  我我我🙈

- 程序员小哈

  2021年12月7日

  赞

  喵哥送书，来凑凑热闹

  程序喵大人

  作者2021年12月7日

  赞

  ![[抱拳]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![[抱拳]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![[抱拳]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

- 已连接到互联网

  2021年12月7日

  赞

  支持一下，全是硬菜

- 阿秀

  2021年12月7日

  赞

  送书？我来了

  程序喵大人

  作者2021年12月7日

  赞

  送

- 惑星

  2021年12月7日

  赞

  想学音视频，喵哥有资料没

  程序喵大人

  作者2021年12月7日

  赞

  你在交流群里吗，之前我在群里分享过

- 墨尘🐳

  2021年12月7日

  赞

  喵哥推荐，必属精品，同时恭喜自己高中奖品![[呲牙]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

- 不喝咖啡

  2021年12月7日

  赞

  虽然我看不懂，但我大受震撼

- sowhat1412

  2021年12月7日

  赞

  我我我

- 夜空中最亮的星

  2021年12月7日

  赞

  rtti好耶

- 多加香菜多加葱^

  2021年12月7日

  赞

  结论呢，是推荐用强制转换还是rtti?

- e^(iα)=cosα+isinα⁵·⁰

  2021年12月7日

  赞

  感谢喵哥送书

- jun

  2021年12月7日

  赞

  我们这边还在培养卓越软件工程师，英国那边都准备上市全新一代智能机器人![[叹气]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

  程序喵大人

  作者2021年12月7日

  赞

  你是哪边？

  jun

  2021年12月7日

  赞

  指的是我们国家这边

- 电脑迷

  2021年12月7日

  赞

  真土豪应该一人送一本🙃

已无更多数据

[](javacript:;)

![](http://mmbiz.qpic.cn/sz_mmbiz_png/Av3j9GXyMmGmg0HNK7fR6Peq3uf9pUoN5Qt0s0ia8IH97pvIiaADsEvy9nm3YnR5bmA13O8ic0JJOicQO0IboKGdZg/300?wx_fmt=png&wxfrom=18)

程序喵大人

26分享14

35

写留言

**留言 35**

- 你猜呀

  2021年12月7日

  赞1

  喵大哥的公众号是目前我关注干货含金量最高的了，从中学到了很多，也给项目组分享学习，希望喵大哥能一如既往地推出更多好文，共同成长进步！加油💪![[强]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

  置顶

  程序喵大人

  作者2021年12月9日

  赞3

  送你哈，姓名电话地址加我微信发我吧

- AlwaysSunny丶

  2021年12月7日

  赞

  这本书看起来是基于大学学习的软件工程方法论的引申版，大学的那本书主要介绍的是软件工程里重要的原则、流程，还有模型，代码近似等于逻辑加工，软件工程就是将逻辑加工的更完美、更通俗易懂。但是不知道原则竟然还能有201种![[旺柴]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

  置顶

  程序喵大人

  作者2021年12月8日

  赞

  姓名电话地址私信我哈，送给你

  AlwaysSunny丶

  2021年12月8日

  赞

  哇，谢谢喵哥![[旺柴]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

- 云瑶

  2021年12月7日

  赞1

  这篇文章过几天看看,关于软件的也想多学学,这几天在阅读喵大人前段时间分享的关于内存的两篇文章,正好给第一篇做了个总结. 作者先从自己工作的案例开始分析,进程被莫名kill,用户管理层，C 运行时库层，操作系统层是内存泄漏的三个层面,一顿分析只能是glibc出了问题,顺便复习了几个知识点,比如栈区是从高地址向低地址生长,是唯一不需要映射，用户却可以访问的内存区域，这也是利用堆栈溢出进行攻击的基础,数据区只存在源代码中有预定义值的全局变量和静态变量,还有mmap 映射区域至顶向下扩展这玩意好久之前就忘了.,它也是向上增长的.然后涉及到如何调用:对于heap的操作，操作系统提供了brk()函数，c运行时库提供了sbrk()函数。对于mmap映射区域的操作，操作系统提供了mmap()和munmap()函数,我们要谨记内存的延迟分配，只有在真正访问一个地址的时候才建立这个地址的物理映射,在此之前只会给你分配一个线性区在glibc中，malloc就是调用sbrk()函数将数据段的下界移动以来代表内存的分配和释放。sbrk()函数在内核的管理下，将虚拟地址空间映射到内存，供malloc()函数使用。mmap是一种内存映射文件的方法,有

  置顶

  程序喵大人

  作者2021年12月7日

  赞

  送你一本

  云瑶

  2021年12月7日

  赞

  我还没写完.后面还有,内存那几篇真心不错..

  程序喵大人

  作者2021年12月7日

  赞

  可以总结出一篇文章给我投个稿![[呲牙]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

  云瑶

  2021年12月7日

  赞

  这种看别人写的总结也可以吗

  云瑶

  2021年12月7日

  赞

  可以我就试试

  程序喵大人

  作者2021年12月7日

  赞

  可以

  程序喵大人

  作者2021年12月8日

  赞

  加我微信，给我姓名电话地址哈

- 瑜

  2021年12月9日

  赞2

  首先为喵哥点赞，为大家分享好书！大多数人重视代码量，重视编码技巧，却忽视编码的工程属性，软件工程，软件工程，软件是离不开工程的，注重软件的工程属性，才能事半功倍，减少加班时间，保住为数不多的头发![[苦涩]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

- Z.

  2021年12月7日

  赞1

  原则很重要，无论做什么都需要有一个原则，有一个规范，作为一个马上出入职场的程序猿，希望可以通过此书规范自己的软件开发能力。喵哥快内幕我![[呲牙]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

- 李想

  2021年12月7日

  赞1

  喵哥，一人一本![[色]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![[色]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![[色]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

  程序喵大人

  作者2021年12月7日

  赞

  有心无力啊![[流泪]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

- Koala

  2021年12月7日

  赞1

  我我我🙈

- 程序员小哈

  2021年12月7日

  赞

  喵哥送书，来凑凑热闹

  程序喵大人

  作者2021年12月7日

  赞

  ![[抱拳]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![[抱拳]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![[抱拳]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

- 已连接到互联网

  2021年12月7日

  赞

  支持一下，全是硬菜

- 阿秀

  2021年12月7日

  赞

  送书？我来了

  程序喵大人

  作者2021年12月7日

  赞

  送

- 惑星

  2021年12月7日

  赞

  想学音视频，喵哥有资料没

  程序喵大人

  作者2021年12月7日

  赞

  你在交流群里吗，之前我在群里分享过

- 墨尘🐳

  2021年12月7日

  赞

  喵哥推荐，必属精品，同时恭喜自己高中奖品![[呲牙]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

- 不喝咖啡

  2021年12月7日

  赞

  虽然我看不懂，但我大受震撼

- sowhat1412

  2021年12月7日

  赞

  我我我

- 夜空中最亮的星

  2021年12月7日

  赞

  rtti好耶

- 多加香菜多加葱^

  2021年12月7日

  赞

  结论呢，是推荐用强制转换还是rtti?

- e^(iα)=cosα+isinα⁵·⁰

  2021年12月7日

  赞

  感谢喵哥送书

- jun

  2021年12月7日

  赞

  我们这边还在培养卓越软件工程师，英国那边都准备上市全新一代智能机器人![[叹气]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

  程序喵大人

  作者2021年12月7日

  赞

  你是哪边？

  jun

  2021年12月7日

  赞

  指的是我们国家这边

- 电脑迷

  2021年12月7日

  赞

  真土豪应该一人送一本🙃

已无更多数据
