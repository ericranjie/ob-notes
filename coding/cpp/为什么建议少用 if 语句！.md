# 

CPP开发者

 _2021年12月31日 11:55_

以下文章来源于非典型技术宅 ，作者无知少年

[

![](http://wx.qlogo.cn/mmhead/Q3auHgzwzM4tryPIOJZLMUiamQaw12ia8bGSicMJqq2FvXeb25OgVmpicw/0)

**非典型技术宅**.

阿宅的技术分享与日常记录，记录和分享嵌入式进阶之路。 口号是：爱技术，爱生活~

](https://mp.weixin.qq.com/s?__biz=MzAxNDI5NzEzNg==&mid=2651169449&idx=1&sn=1dc2297d6cdfa307b8a81d546c7188b0&chksm=806473f6b713fae09a0a0e37ddccdfd8d66a5d1e2bd944bd1142882303d6cef6028fdea0edd4&mpshare=1&scene=24&srcid=1231Nj083ihSvbAGGejQAvjM&sharer_sharetime=1640942945301&sharer_shareid=5fb9813bfe9ffc983435bfc8d8c5e9ca&key=daf9bdc5abc4e8d03b3f697d0a3ff65587558001f391164a3528abe0cbc8d055ffee4c32b2f4b67dad759b98735e6529bfffd8432130e0312b8cdcb05ecd79c2e76e3a3bda93d7a6c11f8dbe186e01c56b43d453b6ecb7ba31116f55d299b3ba929b850fc947b51a6c9e1b4ed5bd7faa66ea498b0df2fb96876fc7d97523ea7c&ascene=0&uin=MTEwNTU1MjgwMw%3D%3D&devicetype=Windows+11+x64&version=63090b19&lang=zh_CN&countrycode=CN&exportkey=n_ChQIAhIQ710Z0ucoKZId%2FOtREf4UeRLmAQIE97dBBAEAAAAAAK%2BLKrtgSpgAAAAOpnltbLcz9gKNyK89dVj0dfqw%2Fx0OB9ggFSfoeQeN3n%2FssjNLOxEaVZCj7Mo4OJH1vGQzrG%2Bj0FX3CRX%2FW1yo1MiLZrnJ%2BdFDkb1q%2Br3PMTD6hX654xk91wu9hwbcjV3bL%2FM1sAjP5MH5RzX9wJa1%2F3m7utlgV8uaKMGJGf5910gbkb3fQ2hPXxg%2Bsxxw5h92VO1RLar5t0YrSw4RtrIIK1o9vXF2iNI6Om5ocSzCTZN5U3Ou%2F6cHfbBzYGW80zTfAK%2FsWd26VDI0kRk6ang%2B&acctmode=0&pass_ticket=2p1xqCs4CyaYKE9Xw71lN1bXqdfwU1V7jUMzK6N9VakRadn72zPkwhVUmverxNl8&wx_header=1&fasttmpl_type=0&fasttmpl_fullversion=7350504-zh_CN-zip&fasttmpl_flag=1#)

相信大家或多或少的听说过，少用点if-else吧？但是为什么要少用呢，有人说他会影响程序运行效率，但是这并不是他最大的罪状...  

## if-else 的罪状

if-else 作为三种最基本的程序结构之一，是我们从最开始学习编程时就接触的基本语句。但是到后面的阶段就不断听人说少用if-else。

如果询问原因的话，你得到的结果大概率是if-else导致程序运行效率下降。这次来扯扯为什么我们说要少用if-else。

- 导致程序运行效率下降（大部分时候可以忽略）
    
- 破坏程序结构，导致代码难以维护（核心原因 ⭐）
    

## if 语句与运行效率

说起if语句导致程序运行效率下降，就不得不提到CPU的流水线结构，效率降低主要是由于多级流水线结构造成的。

现代的大部分CPU在执行代码的时候并不是读取一条指令然后执行一条执行的,而是使用了一种叫做流水线技术的方式，同时去执行多个操作。

### 流水线的影响

比如三级流水线就是指，CPU在执行一条指令时，同时会读取后面的指令，并对进行译码。（读取、译码、执行）![](https://mmbiz.qpic.cn/mmbiz_png/AVDJo94ibZ1pbhJgwZEzryiaEm6Qiaf1sVwFJ03qicyO2LBgzKh2kseGXBHwcRuFiat98KTmRTehXxfwCKqDxUwvPpQ/640?wx_fmt=png&wxfrom=13&tp=wxpic)

这样处理的优势很明显，使用流水线技术可以大大的提高执行效率。

但是它并不是所有时刻有效的，在程序中执行跳转代码时，CPU 会丢弃流水线现有的结果。

原因嘛，很简单！我都不执行后面的代码了，你提前读取有啥用~

所以在这个时候if语句相对于顺序执行的指令，会有几个时钟周期的差距。但这不是if语句所特有的，所有带跳转结构的语句都会这样(if、switch、for)。

### 分支预测的影响

多级流水线在遇到跳转时，会有几个时钟的周期的影响，但这并不是它被指控运行效率低的主要原因。

而是在因为它分支预测部分，它有可能有10-20个时钟周期的影响，在大量使用if的地方这种影响将被放大。

下面说说分支预测为什么会有这么大的影响。

在上面说到多级流水在遇到跳转指令时会清空当前流水线，CPU的设计者在设计引入了一种叫做分支预测的技术来进行处理这个问题。

分支预测简单说就是猜测后面的程序会执行那一段代码，并提前将它读取。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

例如一辆火车，在有很多岔道的路上前进，为了不让他每次都在岔道停下等待（清空流水线），于是想出了一个办法。

猜测火车需要前进的方向，如果猜中了火车就可以不用停下等待，而提高效率。

但是如果猜错了，则需要倒车回到岔路口重新选择。这样的错误代价就比较高了。

而大家所说的效率降低主要源于此。

## if-else 对程序结构的影响

在大部分情况下，我们是不需要考虑if语句对代码执行效率的影响，我们甚至感觉不到它的存在。

因为大部分情况下，CPU的性能是足够的（性能优化时除外）。

但是if-else对程序结构的影响却是不容忽视的，因为我们可以直观的感受到它的存在，而且对开发和维护有极大的影响。

看下面一段代码：

`if (condition1==true)       {f1();}   else if (condition2==true)       {f2();}   else if (condition3==true)       {f3();}`

这个代码非常简单：判断不同条件时执行不同的代码块。

这段代码写完测试时发现有点问题

1. condition1和condition3同时满足时应该先执行f4
    
2. condition3和condition4满足任意一个时执行f3
    
    修改代码测试通过后，于是乎代码变成了下面的模样：
    

`else if (condition1==true)   {     if (condition3==true)     { f4(); }     f1();    }   else if (condition2==true)       { f2(); }   else if (condition3==true || condition4==true)       { f3(); }   `

这只是我简单模拟的一段代码，对于稍微复杂的逻辑，if-else的数量远远大于上面的数量。  

在这样的代码中，如果各种condition都是使用flag变量进行标记时，将会是一种巨大的灾难。

我之前碰到这样的代码时，心情只能用下图表示。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

大量使用if-else，会使代码变得难以理解，同时增加后期开发和维护的成本。

这个才是少用if-else真的原因！

## 如何消除if-else

既然上面说到了if-else有这么多的问题，那应该怎样减少使用它呢？

### 1. 巧妙使用算术表达式

比如下面的代码，在num不能被5整除时，num加一

`if(num%5>0)   {     num++;   }`

可以替换成`num = num + !!(num%5);`

这种一般是在对计算结果进行简单判断时可使用，它的优化点在于消除了分支结构，提高了执行效率。（虽然说很小）。

### 使用断言（assert）

在对函数参数进行合法性检查时常用，可以减少大量对参数进行时的if-else，适用场景也比较简单。

### 查找表（函数转移表）

查找表或者函数转移表，可以对程序的整体结构进行优化或者改进。

比如下面一个计算器的代码：

`if(oper == ADD)   { Result=add(op1,op2);}   else if(oper == SUB)   { Result=add(op1,op2);}   if(oper == MUL)   { Result=mul(op1,op2);}   else if(oper == DIV)   { Result=div(op1,op2);}`

使用函数转移表可改进为

`typedef int (*oper_t)(int, int);   oper_t oper_table[]={add, sub, mul, div};   ...   result = oper_table[oper](op1,op2);`

查找表则相对更灵活，可以对不同类型的数据进行查找;

`#define arrayof(x)  (sizeof(x)/sizeof(x[0]))   typedef int (*oper_t)(int, int);   struct find_table_t {     char *oper_name;     oper_t oper_func;   }   find_table_t oper_table[]=   {{"add",add}, {"sub",sub}, {"mul",mul}, {"div",div}};      for(int i=0; i<arrayof(oper_table);i++)   {     if(strcmp(oper,oper[i].oper_name)==0)     {       result = oper[i].func(op1,op2);       return result;     }   }`

### 责任链（职责链）

责任链将一个复杂逻辑的流程进行分解，将每个判断条件的判断交给责任链节点进行处理，在处理完成后将结果传递给下一个节点。

### 状态机

状态机也是消除if-else的一种方法，在状态机中对所有条件的判断变成的状态转移。

- EOF -

推荐阅读  点击标题可跳转

1、[四万字长文，这是我见过最好的模板元编程文章！](http://mp.weixin.qq.com/s?__biz=MzAxNDI5NzEzNg==&mid=2651168705&idx=1&sn=26d2020b92e6f702d0037850ddf0be8b&chksm=80644e9eb713c78839c1fef9a9593f9d8125cb1e94b42222ab2625b836acee2a22478bcf6fd9&scene=21#wechat_redirect)

2、[C++ 为什么要弄出虚表这个东西？](http://mp.weixin.qq.com/s?__biz=MzAxNDI5NzEzNg==&mid=2651168609&idx=1&sn=edd9257dd189d7b2c549ba5903e332b7&chksm=80644e3eb713c728015176c34ac40242611327cfd3b571f6e4fd89c8e8a6bc819c98b7e19848&scene=21#wechat_redirect)

3、[硬核图解！30 张图带你搞懂！路由器，集线器，交换机，网桥，光猫有啥区别？](http://mp.weixin.qq.com/s?__biz=MzAxNDI5NzEzNg==&mid=2651168704&idx=1&sn=c0bffe995785a233fe6d238bc649ff57&chksm=80644e9fb713c78947695036c8ab8d8d6efab24371184938863ef61ec5b885e18c52894b54c6&scene=21#wechat_redirect)

  

**关注『CPP开发者』**  

看精选C++技术文章 . 加C++开发者专属圈子

![](http://mmbiz.qpic.cn/mmbiz_png/pldYwMfYJpia3uWic6GbPCC1LgjBWzkBVqYrMfbfT6o9uMDnlLELGNgYDP496LvDfiaAiaOt0cZBlBWw4icAs6OHg8Q/300?wx_fmt=png&wxfrom=19)

**CPP开发者**

我们在 Github 维护着 9000+ star 的C语言/C++开发资源。日常分享 C语言 和 C++ 开发相关技术文章，每篇文章都经过精心筛选，一篇文章讲透一个知识点，让读者读有所获～

24篇原创内容

公众号

点赞和在看就是最大的支持❤️

阅读 4575

​

写留言

**留言 13**

- 冯某乙
    
    2021年12月31日
    
    赞48
    
    这些统统都不要考虑。可读性才是首要目标
    
- 李少侠
    
    2021年12月31日
    
    赞8
    
    牺牲可读性，不值得
    
- 宇宙第三胖
    
    2021年12月31日
    
    赞6
    
    舍本求末，性能上没丝毫提升，反而极大降低可读性，变相增加维护成本。
    
- 胡劲英
    
    2021年12月31日
    
    赞3
    
    估计大部分的人找不到这个需要加速的场景
    
- 罗均-Jun
    
    2021年12月31日
    
    赞3
    
    满满的干货![[强]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![[强]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![[强]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
- 2cat
    
    2021年12月31日
    
    赞2
    
    我们最近cpp期末project要写cnn，然后老师说relu，就是max（0，x）的操作要避免if，改成（x>0）*x。但是实测不管是gcc还是msvc，优化等级多少，用if明显快200ms。 猜测浮点数乘法的开销比什么分支预测失败大多了。
    
- 白面角鸮
    
    2021年12月31日
    
    赞2
    
    if else不用，甚至if也不用
    
- Eclipse
    
    2022年1月1日
    
    赞1
    
    要记住 代码是给人看的
    
- 小汪
    
    2021年12月31日
    
    赞1
    
    抛开使用场景谈建议就是耍流氓![[裂开]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
- 李名赫
    
    2022年1月6日
    
    赞
    
    一个简单的if else能搞定的事，你非要上表，做字符串比对。你真确定新方式中没多几个你看不见的if else？
    
- 北极的企鹅
    
    2022年1月2日
    
    赞
    
    太深的if嵌套很恶心人。
    
- 影舞【暗】
    
    2022年1月1日
    
    赞
    
    职责链你尽然说比if else效率高 ![[捂脸]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
- 李洋
    
    2021年12月31日
    
    赞
    
    确实和什么cpu执行效率关系没那么大 就是发生了人祸 为了避免下次人祸的发生而已 和技术本身的关系还真不大
    

已无更多数据

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/pldYwMfYJpia3uWic6GbPCC1LgjBWzkBVqYrMfbfT6o9uMDnlLELGNgYDP496LvDfiaAiaOt0cZBlBWw4icAs6OHg8Q/300?wx_fmt=png&wxfrom=18)

CPP开发者

1517

13

写留言

**留言 13**

- 冯某乙
    
    2021年12月31日
    
    赞48
    
    这些统统都不要考虑。可读性才是首要目标
    
- 李少侠
    
    2021年12月31日
    
    赞8
    
    牺牲可读性，不值得
    
- 宇宙第三胖
    
    2021年12月31日
    
    赞6
    
    舍本求末，性能上没丝毫提升，反而极大降低可读性，变相增加维护成本。
    
- 胡劲英
    
    2021年12月31日
    
    赞3
    
    估计大部分的人找不到这个需要加速的场景
    
- 罗均-Jun
    
    2021年12月31日
    
    赞3
    
    满满的干货![[强]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![[强]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![[强]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
- 2cat
    
    2021年12月31日
    
    赞2
    
    我们最近cpp期末project要写cnn，然后老师说relu，就是max（0，x）的操作要避免if，改成（x>0）*x。但是实测不管是gcc还是msvc，优化等级多少，用if明显快200ms。 猜测浮点数乘法的开销比什么分支预测失败大多了。
    
- 白面角鸮
    
    2021年12月31日
    
    赞2
    
    if else不用，甚至if也不用
    
- Eclipse
    
    2022年1月1日
    
    赞1
    
    要记住 代码是给人看的
    
- 小汪
    
    2021年12月31日
    
    赞1
    
    抛开使用场景谈建议就是耍流氓![[裂开]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
- 李名赫
    
    2022年1月6日
    
    赞
    
    一个简单的if else能搞定的事，你非要上表，做字符串比对。你真确定新方式中没多几个你看不见的if else？
    
- 北极的企鹅
    
    2022年1月2日
    
    赞
    
    太深的if嵌套很恶心人。
    
- 影舞【暗】
    
    2022年1月1日
    
    赞
    
    职责链你尽然说比if else效率高 ![[捂脸]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
- 李洋
    
    2021年12月31日
    
    赞
    
    确实和什么cpu执行效率关系没那么大 就是发生了人祸 为了避免下次人祸的发生而已 和技术本身的关系还真不大
    

已无更多数据