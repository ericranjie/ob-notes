# 

Original 逸珺 嵌入式客栈

 _2021年10月18日 17:30_

[导读] 大家好，我是逸珺。

今天来分享整理如何正确的使用断言。

![](http://mmbiz.qpic.cn/mmbiz_png/qFV4SqXFJYsicgIoE4jSibVqvtMscNibwF1j2v5T0AhwuhqkrvuZC2VhdHtohWIa6mAuAU50vEniccEFtgYjSxcR2g/300?wx_fmt=png&wxfrom=19)

**嵌入式客栈**

热爱技术，喜欢钓鱼

115篇原创内容

公众号

## 何为断言

断言一般是用于检测在某个程序位置程序**必须**满足某些条件的宏。一般用的多的可以分两种种情况：

- 前置条件：在某个程度点开始的地方
    
- 后置条件：在某段程序执行结束后，一般用于检测执行结果
    

断言发生表示程序中存在错误。因此，断言是提高程序可靠性的有效手段。也是开发阶段快速定位问题的一种很好防御式编程方法。

在C语言中，断言是一些条件判断的宏。比如C语言内置断言是用标准的 assert 宏实现的。当宏执行时，assert 的参数必须为真，否则程序中止并打印错误消息。

比如，在IAR中：

`#define assert(test) ((test) ? (void) 0 : abort())   `

也可以编程者自己定义，比如：

`#define assert(arg) { if( !(arg) ) { printf("assert in File="__FILE__" Line=%d ",__LINE__); return; } }   `

## 该怎么用

# 前置条件

比如某一个函数代码：

`#define ALLOWED_SIZE  (1024)   int func(int size, char *buffer ) {     assert( size <= ALLOWED_SIZE );     assert( format != NULL );     ...   }   `

这个函数里，使用了两次断言判断函数执行的前置条件：

- size必须要不大于ALLOWED_SIZE，func函数才真正执行其任务。因此，如果输入的size超过1024，func不会做任何处理。
    
- buffer传入的地址必须不是NULL，否则func函数不会执行。
    

具体断言判断失败了，断言宏干了什么，需要看看这个宏的实现，有可能是直接返回，有可能整个程序直接终止执行。所以看看其实现就知道了。

# 后置条件

后置条件断言一般是指判断函数的执行结果。比如：

`int func(int size, char *buffer ) {    int result;        /*中间处理部分更新这个返回值*/     ...          assert( result <= ALLOWED_SIZE );     return result;   }   `

这样写表示这个函数的返回值永远不会大于ALLOWED_SIZE。如果大于了，就证明产生错误了。

## 什么时候用

断言的最常用和最有效的用途是检查前置条件——即指定和检查函数的输入条件。两个非常常见的用途：

- 指针不是 NULL。
    
- 索引和边界范围值是在设计的合理范围之类。
    

尤其如果写一个代码包给其他的人调用的时候，这样处理会使代码提高健壮性，易用性。

当代码调用带有前置条件的断言时，必须要确保满足该函数的前置条件。但这并不意味着必须断言检查调用的每个函数的参数！

**调试的便利**：

- 如果在程序测试和调试期间违反了前置条件，也就是说断言异常了，则调用包含前置条件的函数的代码中存在bug。
    
- 如果在程序测试和调试期间违反了后置条件，则该断言前面部分代码可能有bug。
    

这样利用断言的打印，或者检测到断言指定的行为，就可以很快速的发现bug，而避免要在后期反复测试才能识别出bug。

那么什么时候用？首先，区分**程序错误**和运行时错误很重要：

- 程序错误是一个bug，永远不应该发生。
    
- 运行时错误可能在程序执行期间的任何时间发生。
    

断言不是处理运行时错误的机制。例如，由于用户在预期为正数时无意中输入了负数而导致的断言异常就是程序设计不合理。像这样的情况必须通过适当的错误检查和恢复代码（比如弹出一个提示输入合理范围）来处理，而不是通过断言来处置。

当然，实际是程序都可能会有bug，这些bug会在运行时出现。确切地说，断言要检查什么条件以及运行时错误检查代码要检查什么是设计问题。

如前所说，断言在可重用库中非常有效。比如在QT中：

`int main(int argc, char *argv[])   {       QVector <int> list;       list.append(0);       list.append(1);       qDebug() << list.at(2);              return 0;   }   `

一运行，就会有这样的结果：

`ASSERT failure in QVector<T>::at: "index out of range", file C:\Qt\Qt5.7.1\5.7\mingw53_32\include/QtCore/qvector.h, line 429   assert in File=..\src\main.cpp Line=4   `

因为list只有两个元素，list.at(2)则是去访问第3个，显然访问的元素不存在，所以就断言了。

![](http://mmbiz.qpic.cn/mmbiz_png/qFV4SqXFJYsicgIoE4jSibVqvtMscNibwF1j2v5T0AhwuhqkrvuZC2VhdHtohWIa6mAuAU50vEniccEFtgYjSxcR2g/300?wx_fmt=png&wxfrom=19)

**嵌入式客栈**

热爱技术，喜欢钓鱼

115篇原创内容

公众号

—— The End ——

  

推荐阅读  点击蓝色字体即可跳转

[☞ 实例分析如何远离漫天飞舞的全局变量](http://mp.weixin.qq.com/s?__biz=MzAwNjQ4OTE4Mw==&mid=2247485421&idx=1&sn=137b4301b7f36c31ce9edeef8e0258a3&chksm=9b0dd2ddac7a5bcb3713bd3f0b8495bb5feb86e1dad2836fa24b0f73fa9ea3f0ce653386583f&scene=21#wechat_redirect)  

☞ [步进电机调速，S曲线调速算法你会吗？](http://mp.weixin.qq.com/s?__biz=MzAwNjQ4OTE4Mw==&mid=2247488567&idx=1&sn=9b02f251e013217733c9c2878e081546&chksm=9b0dc107ac7a4811e057bad4aac0f868e721d87878e8dd367c5191bcb8488bb2a05cbe3ad230&scene=21#wechat_redirect)

☞ [图文详解Modbus-RTU协议](http://mp.weixin.qq.com/s?__biz=MzAwNjQ4OTE4Mw==&mid=2247488019&idx=1&sn=b20f87aff45f551ed547bcf11f514ba2&chksm=9b0dc723ac7a4e35924338b76f356a16d9ddc23519278e3f0912edd5140c4b3681599f0b3305&scene=21#wechat_redirect)

[☞ RS-485总线，这篇很详细](http://mp.weixin.qq.com/s?__biz=MzAwNjQ4OTE4Mw==&mid=2247487902&idx=1&sn=d8f05ee6690c63612633f811ef43b928&chksm=9b0dc4aeac7a4db8b00cf31ebe0808506d5dd166a90d7193a55a4abc9de084c99fb33e55bd92&scene=21#wechat_redirect)

欢迎**转发、留言、点赞、分享**，感谢您的支持！

C语言12

编程语言9

编程风格2

C++7

C语言 · 目录

上一篇意面虽好吃，意面式代码还是要远离下一篇手把手带你写一个中断输入设备驱动

Reads 1730

​

People who liked this content also liked

打造自己的Kubernetes王国：从小白到K8S专家的华丽转身

聊聊IT技术

不喜欢

不看的原因

OK

- 内容低质
- 不看此公众号内容

![](https://mmbiz.qpic.cn/mmbiz_jpg/wpJkhDBHClib3Q056uDQBYB5ljzrpKDSSpOyGa1UNP915dicQhnXBiasvNztymlUoqOQQ2UaGCGEpVsXgVVVVj7Lg/0?wx_fmt=jpeg&tp=wxpic)

开发了一个 Copilot 用来处理运维故障

陈少文

不喜欢

不看的原因

OK

- 内容低质
- 不看此公众号内容

![](https://mmbiz.qpic.cn/sz_mmbiz_jpg/V6icG3TMUkiaxibIgASAgtSVIblBV8vk62mpyTxgrh3W02MaTOeCq4DzDoJmC0XVwVkcwkriaNun3VkEb8O2H7VKKg/0?wx_fmt=jpeg)

GreptimeDB vs. ClickHouse vs. ElasticSearch 日志引擎性能对比报告

GreptimeDB

不喜欢

不看的原因

OK

- 内容低质
- 不看此公众号内容

![](https://mmbiz.qpic.cn/sz_mmbiz_jpg/B9yiaFdoD68ygDPPbMcMQhj8CmsNHic5sbyGjfeviasUiaKbdx4KFdzs1pTLjbypAmqEyqRUOHO00icW6BxmSXDm8Ag/0?wx_fmt=jpeg)

Comment

**留言 24**

- 逸珺
    
    2021年10月18日
    
    Like5
    
    有没有人喜欢钓鱼🎣？![[呲牙]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
- 一口Linux
    
    2021年10月18日
    
    Like
    
    学习了！！！
    
    嵌入式客栈
    
    Author2021年10月18日
    
    Like1
    
    嗯嗯，明白你是说钓鱼
    
- 骆伟健
    
    2021年10月18日
    
    Like1
    
    你说钓鱼我可就不困了啊
    
    嵌入式客栈
    
    Author2021年10月18日
    
    Like1
    
    ![[嘿哈]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)，道友![[抱拳]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
- 石同学
    
    2021年10月18日
    
    Like1
    
    调试挺好用
    
    嵌入式客栈
    
    Author2021年10月18日
    
    Like
    
    是的，快速查虫
    
- 韦启发Cris🏀
    
    2021年10月18日
    
    Like1
    
    我喜欢
    
    嵌入式客栈
    
    Author2021年10月18日
    
    Like
    
    会上瘾![😂](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
- 记得诚
    
    2021年10月18日
    
    Like1
    
    可以写一篇如何钓鱼🎣
    
    嵌入式客栈
    
    Author2021年10月18日
    
    Like
    
    哈哈，这个可以有，堂而皇之教人摸鱼![😂](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
- 静思
    
    2021年10月18日
    
    Like1
    
    断言这鱼==5斤![[666]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![[666]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![[666]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
    嵌入式客栈
    
    Author2021年10月18日
    
    Like1
    
    浮点不能这么断言![[呲牙]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
    静思
    
    2021年10月18日
    
    Like1
    
    assert(type==红尾）
    
    嵌入式客栈
    
    Author2021年10月18日
    
    Like
    
    👍👍，看来是同道中人![😄](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
- Deven
    
    2021年10月20日
    
    Like
    
    #define assert(arg) { if( !(arg) ) { printf("assert in File="__FILE__" Line=%d ",__LINE__); return; } }， 这个只能在函数无返回值时使用，具有局限性，c库的assert 调用异常直接终止程序，我们很多时候并不想程序终止， 一般函数都有返回值作为成功或者操作结果错误码， 建议用return -1,来处理结果，只在返回为int 时调用， -1代表一般错误， 其它错误可以自定义。另外自定义assert可以使用自定义的printf,有的平台printf并不能使用，这里可以自定义
    
    Deven
    
    2021年10月20日
    
    Like
    
    补充： 用标准库的assert后， 在开发完后可以用 NDEBUG 取消断言，这样可减少代码量和提高执行速度。
    
    嵌入式客栈
    
    Author2021年10月20日
    
    Like
    
    👍👍，是的，自定义的话也可以做一个编译开关，还可以根据需要做出不同的assert变体宏
    
- 玩呀熊熊
    
    2021年10月18日
    
    Like
    
    不敲代码就钓鱼的路过
    
    嵌入式客栈
    
    Author2021年10月18日
    
    Like
    
    看来我要多写写钓鱼![[机智]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
- Mango奶酪
    
    2021年10月18日
    
    Like
    
    🙌 从小喜欢到现在，不过都是从小在农村水库野钓的好玩，不懂专业钓法![[悠闲]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
    嵌入式客栈
    
    Author2021年10月18日
    
    Like
    
    我也是野钓，买了一个台钓箱子
    
- 程序员小哈
    
    2021年10月18日
    
    Like
    
    想看如何入门🎣
    
    嵌入式客栈
    
    Author2021年10月18日
    
    Like
    
    我就是入门水平
    

已无更多数据

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/qFV4SqXFJYsicgIoE4jSibVqvtMscNibwF1j2v5T0AhwuhqkrvuZC2VhdHtohWIa6mAuAU50vEniccEFtgYjSxcR2g/300?wx_fmt=png&wxfrom=18)

嵌入式客栈

10Share4

24

Comment

**留言 24**

- 逸珺
    
    2021年10月18日
    
    Like5
    
    有没有人喜欢钓鱼🎣？![[呲牙]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
- 一口Linux
    
    2021年10月18日
    
    Like
    
    学习了！！！
    
    嵌入式客栈
    
    Author2021年10月18日
    
    Like1
    
    嗯嗯，明白你是说钓鱼
    
- 骆伟健
    
    2021年10月18日
    
    Like1
    
    你说钓鱼我可就不困了啊
    
    嵌入式客栈
    
    Author2021年10月18日
    
    Like1
    
    ![[嘿哈]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)，道友![[抱拳]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
- 石同学
    
    2021年10月18日
    
    Like1
    
    调试挺好用
    
    嵌入式客栈
    
    Author2021年10月18日
    
    Like
    
    是的，快速查虫
    
- 韦启发Cris🏀
    
    2021年10月18日
    
    Like1
    
    我喜欢
    
    嵌入式客栈
    
    Author2021年10月18日
    
    Like
    
    会上瘾![😂](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
- 记得诚
    
    2021年10月18日
    
    Like1
    
    可以写一篇如何钓鱼🎣
    
    嵌入式客栈
    
    Author2021年10月18日
    
    Like
    
    哈哈，这个可以有，堂而皇之教人摸鱼![😂](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
- 静思
    
    2021年10月18日
    
    Like1
    
    断言这鱼==5斤![[666]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![[666]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![[666]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
    嵌入式客栈
    
    Author2021年10月18日
    
    Like1
    
    浮点不能这么断言![[呲牙]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
    静思
    
    2021年10月18日
    
    Like1
    
    assert(type==红尾）
    
    嵌入式客栈
    
    Author2021年10月18日
    
    Like
    
    👍👍，看来是同道中人![😄](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
- Deven
    
    2021年10月20日
    
    Like
    
    #define assert(arg) { if( !(arg) ) { printf("assert in File="__FILE__" Line=%d ",__LINE__); return; } }， 这个只能在函数无返回值时使用，具有局限性，c库的assert 调用异常直接终止程序，我们很多时候并不想程序终止， 一般函数都有返回值作为成功或者操作结果错误码， 建议用return -1,来处理结果，只在返回为int 时调用， -1代表一般错误， 其它错误可以自定义。另外自定义assert可以使用自定义的printf,有的平台printf并不能使用，这里可以自定义
    
    Deven
    
    2021年10月20日
    
    Like
    
    补充： 用标准库的assert后， 在开发完后可以用 NDEBUG 取消断言，这样可减少代码量和提高执行速度。
    
    嵌入式客栈
    
    Author2021年10月20日
    
    Like
    
    👍👍，是的，自定义的话也可以做一个编译开关，还可以根据需要做出不同的assert变体宏
    
- 玩呀熊熊
    
    2021年10月18日
    
    Like
    
    不敲代码就钓鱼的路过
    
    嵌入式客栈
    
    Author2021年10月18日
    
    Like
    
    看来我要多写写钓鱼![[机智]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
- Mango奶酪
    
    2021年10月18日
    
    Like
    
    🙌 从小喜欢到现在，不过都是从小在农村水库野钓的好玩，不懂专业钓法![[悠闲]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
    嵌入式客栈
    
    Author2021年10月18日
    
    Like
    
    我也是野钓，买了一个台钓箱子
    
- 程序员小哈
    
    2021年10月18日
    
    Like
    
    想看如何入门🎣
    
    嵌入式客栈
    
    Author2021年10月18日
    
    Like
    
    我就是入门水平
    

已无更多数据