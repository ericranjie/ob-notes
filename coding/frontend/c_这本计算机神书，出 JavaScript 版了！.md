玩转VS Code

_2024年03月17日 09:15_ _上海_

《计算机程序的构造和解释》（**S**tructure and **I**nterpretation of **C**omputer **P**rograms，简记为**SICP**）是MIT的基础课教材，出版后引起计算机教育界的广泛关注，对推动全世界大学计算机科学技术教育的发展和成熟产生了很大影响。这本书的第1版于1984年出版，第2版于1996年出版，至今已被全世界100多所大学采用为教材，其中包括斯坦福大学、普林斯顿大学、牛津大学等。

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/CwicwFUdzg1a2BQkOxuUYSFPh1Ysd4KqgNrEcicrLUNdtJOXwHBHrQ5oZpM0DyFzOnB9VR0ibUA5wC4AibHNjw9LvA/640?wx_fmt=png&from=appmsg&wxfrom=13)

书号：9787111630548

出版时间：2019-07-01

机械工业出版社把SICP（第2版）引进中国，于2004年出版，至今已近20年了。令人感兴趣的是，SICP至今仍然受到国内关心计算机科学技术的人们，特别是计算机专业的优秀学生和青年计算机工作者的关注。

与许多计算机科学领域的入门教材不同，SICP的最主要关注点并不在基础语言中各种编程结构的形式和意义，也没有深入讨论巧妙或深刻的算法。与众不同地，一方面，**SICP注目于帮助读者理解基于计算的观点看世界、看问题的重要性，掌握相关的基本概念和观点，建立基于计算思考问题的习惯**，也就是今天人们常说的计算思维。另一方面，**SICP也深入讨论了通过计算的方式处理和解决问题时必须掌握的主要技术与方法，最重要的就是分解问题和组织计算，以及建立和使用抽象的各种技术与方法。**

SICP的章节目录清晰地反映了作者的基本想法：

- 第1、2两章分别讨论函数（或过程）抽象和数据抽象的作用，它们的建立和使用；

- 第3章讨论抽象数据对象本身的状态和变化，相关的模块化的问题及其在计算实践中的重要性；

- 第4章讨论元语言抽象，也就是设计和实现面向应用的新语言的问题；

- 第5章可以看作前面讨论的应用，而应用的对象问题就是JavaScript语言在寄存器机器上的实现。这里的寄存器机器是现代计算机的抽象模型，这里的讨论也说明了抽象的高级语言如何落地。

读者现在拿在手里的这本书是SICP的一个改编本\*\*（SCIP JS）**。与SICP的不同之处，就在于这个**改编本用更多计算机工作者熟悉的****JavaScript****语言作为讨论的工具\*\*，而没有用原SICP里使用的Scheme语言。因此，这里程序实例的形式更接近各种常规的编程语言，可能更容易被更多读者接受。本书的内容是原SICP的翻版，作者编写本书的基本目标是尽可能完整准确地反映原书的宗旨和精神，同时又使这些能被更多的人理解和重视。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

书号：9787111734635

出版时间：2024-02-01

由于本书的根源和作者的意图，本书的基本内容和结构都来自SICP，许多一般性的讨论直接来自原书，但也有许多地方针对JavaScript做了一些调整和修改。**本书比较好地反映了SICP的思想，是一本非常好的学习计算机科学技术的读物**，值得每一个关心计算机领域，并有心在这个领域中深入学习和努力工作的人士阅读学习。

正如作者所言，这本书并不想作为JavaScript的入门教科书。书中对JavaScript语言的介绍远非完整，读者不应该希冀通过阅读本书学习JavaScript编程。但另一方面，由于本书的宗旨和内容，对它的学习一定会**有助于读者学习JavaScript**（一般而言，学习任何常见的编程语言，如Java、Python或C）。如果读者学过JavaScript（或其他编程语言），阅读这本书能帮助你更好地理解程序设计和一般的软件开发，从而有可能在这些领域中做得更出色、更高效、更得心应手。如果本书是你学习计算机科学技术的第一本书（或者学的第一门课），这段学习经历能为你今后的学习建立一个坚实的基础，帮助你更顺利地度过这段专业学习。无论如何，认真地阅读这本书，都是一件非常值得做的事情。

对于本书的学习，必须和相应的实际编程、用计算机解决问题的实践相结合。只读不做，当然不可能真正领悟计算机科学技术的真谛。另一方面，只是抄录、运行和试验书中给出代码，也不能得到其中的真传。作为这本书的真正有心的读者，你必须亲自一次次地经历使用计算机（通过编程）解决问题的实践过程。本书的作者已经为读者提供了学习所需的许多材料和资源，希望读者好好利用。

▲快速抢购

本文作者：裘宗燕，北京大学数学学院信息科学系教授\
本文摘编自《计算机程序的构造和解释：JavaScript版》译者序

送书抽奖活动：

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

阅读 3938

​

**留言 4**

作者已设置关注后才可以留言

- Rom Yim

  广东3月17日

  赞1

  我想起我那吃灰的算法导论![[捂脸]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

- Tan

  河南3月17日

  赞

  看来ken Thompson 老爷子说的很正确，计算机行业发展太慢了，不适合年轻人，直接让他孩子学医学了![[偷笑]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![[偷笑]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![[偷笑]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

- 飓先生

  江西3月17日

  赞

  神书抽奖

  玩转VS Code

  作者3月17日

  赞

  不是这里哈

已无更多数据

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/Z4WC9OGHQlGs17KiajQbcp0Kf17ZVq3lsCUGUPjOaaydTvzBicsKxoDLN8176Yy1MyYKwTXU40QNiagbGl1LuXCMA/300?wx_fmt=png&wxfrom=18)

玩转VS Code

关注

15795

4

**留言 4**

作者已设置关注后才可以留言

- Rom Yim

  广东3月17日

  赞1

  我想起我那吃灰的算法导论![[捂脸]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

- Tan

  河南3月17日

  赞

  看来ken Thompson 老爷子说的很正确，计算机行业发展太慢了，不适合年轻人，直接让他孩子学医学了![[偷笑]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![[偷笑]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![[偷笑]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

- 飓先生

  江西3月17日

  赞

  神书抽奖

  玩转VS Code

  作者3月17日

  赞

  不是这里哈

已无更多数据
