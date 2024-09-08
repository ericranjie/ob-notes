# 

Original 我不想种地 码砖杂役

 _2021年10月05日 10:35_

大学时候学的c，参加工作后用了12年的cpp，再之后又用了2年c，对c和cpp还算熟悉。

  

我对cpp和c的认识，经历过一个变化的过程。

  

最开始觉得cpp很强大很不错，碰到用c的团队总劝说他们尝试cpp，苦口婆心，循循善诱，时至今日，虽然我依然觉得大多数应用开发，cpp是更好的选择，但已经没有了当年那份执念和执着。

我也教过人学编程，之前总选cpp，现在的话，通常会选择c，我不再认为cpp适合初学编程者。

  

在工作的第一个十年，我是cpp的忠实拥趸，在网上看到喷cpp的文章，我更倾向认为他是不熟悉cpp，人有用自己熟悉的东西的习惯，人内心的成见像大山，多年以后，再碰到批评cpp的言论，我大概会有几分共情。

  

这是我认知的变化过程，但变化的原因是我用c的经历。

  

首先，cpp是一门学着学着就忘了正事的语言。它太复杂了，包括面向过程编程、面向对象编程、泛型编程、模板元编程等多种编程范式，甚至还有一点函数式编程的影子，它不像Java那种纯面向对象，猜想可能是因为cpp最初是作为带类的c而设计的缘故。

  

通常，我们把cpp视为c的超集，cpp兼容c，可能这话不一定足够严谨，但大体上如此，我碰到的唯一c能编译过而cpp不能的场景，就是void*到struct xx*的自动类型转换。

  

第二，cpp有很多为了开发库而添加的特征，一般应用开发其实用不着，这些特征的引入增加了理解成本，提高了学习门槛，但充满好奇心的程序员总是抑制不住学习研究使用它的冲动，这股原始冲动像未成年发情的动物。

  

第三，日益膨胀的标准库是另一个负担，98/03标准之前，cpp甚至没有STL，它只是一个带类的c，你可以定义成员函数，可以继承派生，仅此而已。

  

惠普的Alexander博士和limeng女士编写了第一版STL，a博士花了很多时间说服本贾尼，让STL进cpp标准。STL虽然充满学院派色彩，但同时它的实现也非常高效，非常专业，STL对cpp是有益的补充，自此以后，cpp终于有了自己的标准模板库，可以以可移植的方式放心大胆的使用通用容器和算法。

  

但11 14 17 20标准不停的给标准库加东西，且cpp似乎秉承能用标准库解决的问题就不增加新语法的原则，新加的东西，仔细研究会觉得有益且必要，但大即原罪，它让cpp的学习曲线变的更陡。

  

cpp的另一个问题就是很多规则都很隐式，你写下一行代码，它会偷偷给你调用一些函数，最简单的就是构造和析构，你定义一个对象，构造函数将被隐式调用，出作用域，析构函数将被自动执行，当然这些规则cpp第一课都会学，但有一些规则没有这么明显，它总是静悄悄的执行一些你意想不到的逻辑，比如操作符重载，比如operate type()类型转换，比如单参implicit构造，这些语法特征本意是为了编码方便，但其实引入混乱，相比而言，我宁愿多写几行代码，更喜欢显式的调用。

  

相比而言，c的语法简单得多，变量，函数，数组，指针，结构体，仅此而已，所有的逻辑被包装成一个个功能函数，然后把数据作为参数灌进去，吐出结果（就像把米倒进电饭煲煮出米饭一样），或者改变状态，所有的动作都是一个个函数调用触发和完成，唯一的模糊就是函数指针。

  

另外，c的标准库也很简单，也就一百多个接口，常用的strcpy，memcpy，strlen加起来不够二三十个，却可以干任何事情，用c开发的团队项目，大家只需要在一个很小的公共知识集下工作，它很容易达成共识，没有那么多捣蛋鬼。

  

但c也有不足，首先c的抽象能力不足，很多人对抽象没有足够的理解，这是因为抽象很抽象，我三言两语也说不清楚。抽象不足导致的结果就是实现同样的功能c写的代码量会多一些，c写的东西模块化能力差一些，读起来没那么快，而cpp可能看到类名就知道它要干嘛，c为了避免冲突会给相关函数起一个公共前缀，导致函数名不如cpp的成员函数简洁有力。

  

此外，因为c没有标准容器之类，所以每个项目都要自己写链表，哈希，红黑树，映射表等容器和算法，这些基础东西都不是标准化的，相当于地方方言，你从衡阳跳到潮汕，得重新学一门方言，这也是很大的一个不足。

  

所以，如果没有编程经验的人学习编程，我觉得还是c比较好，它的语法足够简单，这样容易让你聚焦真正重要的东西，比如编程思维，比如设计理念，时间要优先花费在真正重要的事情上，而不是在浩渺的语法海洋里放逐。

  

既然c和cpp都有不足，是不是新发明一种语言能解决问题？我不觉得是这样，我认为受控的cpp是一个好的思路，即c with classes，这似乎是cpp最初的模样，可不！

![](https://res.wx.qq.com/mmbizappmsg/en_US/htmledition/js/images/icon/common/icon_avatar_default702c7e.svg)

我不想种地

 别打call，打钱 

![赞赏二维码](https://mp.weixin.qq.com/s?__biz=MzU4NDY5ODU3NQ==&mid=2247485140&idx=1&sn=be50d910667f79c89be50867bc5720dc&chksm=fd949974cae310625ddb2de7ebbe5c13498432a9f7906d46dd2709a275931969ed184757eaf9&mpshare=1&scene=24&srcid=0218GPKuDCsaUMAgsDzqZKuD&sharer_sharetime=1645151280790&sharer_shareid=5fb9813bfe9ffc983435bfc8d8c5e9ca&key=daf9bdc5abc4e8d0c0368b02e0a1c06cf870558b7b62eec9baf33f709421104c6506fbf6aa990a894e5d71a1938eb105257deafcb31889a485ecdf5838ee245ab0366137bc9a2266637c9365a2dd1baff6c552a1fef2129173b56d49f614e3bcc75b181146771d387984a667d4af7094cd8e007a7c2dd3fd5134d83939521562&ascene=14&uin=MTEwNTU1MjgwMw%3D%3D&devicetype=iMac+MacBookAir10%2C1+OSX+OSX+14.6.1+build(23G93)&version=13080710&nettype=WIFI&lang=en&countrycode=CN&fontScale=100&exportkey=n_ChQIAhIQW0nbhHEJKZuHzahua1SDsBKFAgIE97dBBAEAAAAAAFBkIlNRz8MAAAAOpnltbLcz9gKNyK89dVj0ra7D80Z%2FnAwzQtKgkpkviW7lnDJ0HeAgfwIVuJby1BIxXlozAHqX7uLljOMHAjoOek6%2B7PZMd8Ug7rgJ66WO9e8G7IKVJO4G8SksL0HxFVVZKyAmrnVA7MYEPEDgvZ7B8i%2Bln7ELstuQ9tREYaEs%2BVfFlCnADn5mVxNmDBn%2FNXaDweCiL2G5TNk7sDPiyxF9%2BSf1ObjvaFO9ijk8zIXUQ1kwTv1tM4Ls%2BO5bwvYIBWaSlpK15iTSvIcjQKLek1M3mDQsRsrm1y0NeaQHUjjCpygywP4vZQd3mn2c4gex7Q%3D%3D&acctmode=0&pass_ticket=e2wUvou%2BfO2rubneTGUOKGLFBfWsobJpnHCHn0BSa11IniflsKgL43owB1VqV%2FKc&wx_header=0)Kudos to the Author

1 like(s)

![](http://wx.qlogo.cn/mmopen/3chEujbBMxEhgMr2nxCbCrT8NiaKSsGGRvMDZ0LNgC6NZCJC5NI85Zh04ocFyc8Zic0qTdYfpKYrIqQiacEtYrHcVA4MyHWUIX2/64)

Reads 445

​