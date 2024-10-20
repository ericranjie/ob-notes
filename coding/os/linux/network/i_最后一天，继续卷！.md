
原创 小林coding 小林coding _2021年12月31日 17:23_

大家好，我是小林。

2021年最后一天，我也要卷下。

早上有个读者问了我图解网络 PDF 里的问题：

![[Pasted image 20241019185619.png]]

就是他不明白「**为什么 TCP 三次握手期间，为什么客户端和服务端的初始化序列号要求不一样的呢？**」

我的图解网络 PDF 在解释这个原因的时候，就写的几句话，可能会让人看的很懵逼。

后来，我跟他交流半个小时，终于把他讲明白了。

我是一步一步把他讲明白的，我觉得应该有不少人会有类似的问题，所以今天在肝一篇！

# 正文

> 为什么 TCP 三次握手期间，为什么客户端和服务端的初始化序列号要求不一样的呢？

主要原因是为了防止历史报文被下一个相同四元组的连接接收。

> TCP 四次挥手中的 TIME_WAIT 状态不是会持续 2 MSL 时长，历史报文不是早就在网络中消失了吗？

是的，如果能正常四次挥手，由于 TIME_WAIT 状态会持续  2 MSL 时长，历史报文会在下一个连接之前就会自然消失。

但是来了，我们并不能保证每次连接都能通过四次挥手来正常关闭连接。

假设每次建立连接，客户端和服务端的初始化序列号都是从 0 开始：

![[Pasted image 20241019185644.png]]

过程如下：

- 客户端和服务端建立一个 TCP 连接，在客户端发送数据包被网络阻塞了，而此时服务端的进程重启了，于是就会发送 RST 报文来断开连接。

- 紧接着，客户端又与服务端建立了与上一个连接相同四元组的连接；

- 在新连接建立完成后，上一个连接中被网络阻塞的数据包正好抵达了服务端，刚好该数据包的序列号正好是在服务端的接收窗口内，所以该数据包会被服务端正常接收，就会造成数据错乱。

可以看到，如果每次建立连接，客户端和服务端的初始化序列号都是一样的话，很容易出现历史报文被下一个相同四元组的连接接收的问题。

> 客户端和服务端的初始化序列号不一样不是也会发生这样的事情吗？

是的，即使客户端和服务端的初始化序列号不一样，也会存在收到历史报文的可能。

但是我们要清楚一点，历史报文能否被对方接收，还要看该历史报文的序列号是否正好在对方接收窗口内，如果不在就会丢弃，如果在才会接收。

如果每次建立连接客户端和服务端的初始化序列号都「不一样」，就有大概率因为历史报文的序列号「不在」对方接收窗口，从而很大程度上避免了历史报文，比如下图：

![[Pasted image 20241019185706.png]]

相反，如果每次建立连接客户端和服务端的初始化序列号都「一样」，就有大概率遇到历史报文的序列号刚「好在」对方的接收窗口内，从而导致历史报文被新连接成功接收。

所以，每次初始化序列号不一样能够很大程度上避免历史报文被下一个相同四元组的连接接收，注意是很大程度上，并不是完全避免了。

> 那客户端和服务端的初始化序列号都是随机的，那还是有可能随机成一样的呀？

RFC793 提到初始化序列号 ISN 随机生成算法：ISN = M + F(localhost, localport, remotehost, remoteport)。

- M是一个计时器，这个计时器每隔4毫秒加1。

- F 是一个 Hash 算法，根据源IP、目的IP、源端口、目的端口生成一个随机数值，要保证 hash 算法不能被外部轻易推算得出。

可以看到，随机数是会基于时钟计时器递增的，基本不可能会随机成一样的初始化序列号。

> 懂了，客户端和服务端初始化序列号都是随机生成的话，就能避免连接接收历史报文了。

是的，但是也不是完全避免了。

为了能更好的理解这个原因，我们先来了解序列号（SEQ）和初始序列号（ISN）。

- **序列号**，是 TCP 一个头部字段，标识了 TCP 发送端到 TCP 接收端的数据流的一个字节，因为 TCP 是面向字节流的可靠协议，为了保证消息的顺序性和可靠性，TCP 为每个传输方向上的每个字节都赋予了一个编号，以便于传输成功后确认、丢失后重传以及在接收端保证不会乱序。**序列号是一个 32 位的无符号数，因此在到达 4G 之后再循环回到 0**。

- **初始序列号**，在 TCP 建立连接的时候，客户端和服务端都会各自生成一个初始序列号，它是基于时钟生成的一个随机数，来保证每个连接都拥有不同的初始序列号。**初始化序列号可被视为一个 32 位的计数器，该计数器的数值每 4 微秒加 1，循环一次需要 4.55 小时**。

给大家抓了一个包，下图中的 Seq 就是序列号，其中红色框住的分别是客户端和服务端各自生成的初始序列号。
![[Pasted image 20240919095426.png]]

通过前面我们知道，**序列号和初始化序列号并不是无限递增的，会发生回绕为初始值的情况，这意味着无法根据序列号来判断新老数据**。

不要以为序列号的上限值是 4GB，就以为很大，很难发生回绕。在一个速度足够快的网络中传输大量数据时，序列号的回绕时间就会变短。如果序列号回绕的时间极短，我们就会再次面临之前延迟的报文抵达后序列号依然有效的问题。

为了解决这个问题，就需要有 TCP 时间戳。tcp_timestamps 参数是默认开启的，开启了 tcp_timestamps 参数，TCP 头部就会使用时间戳选项，它有两个好处，**一个是便于精确计算 RTT ，另一个是能防止序列号回绕（PAWS）**。

试看下面的示例，假设 TCP 的发送窗口是 1 GB，并且使用了时间戳选项，发送方会为每个 TCP 报文分配时间戳数值，我们假设每个报文时间加 1，然后使用这个连接传输一个 6GB 大小的数据流。
![[Pasted image 20240919095434.png]]

32 位的序列号在时刻 D 和 E 之间回绕。假设在时刻B有一个报文丢失并被重传，又假设这个报文段在网络上绕了远路并在时刻 F 重新出现。如果 TCP 无法识别这个绕回的报文，那么数据完整性就会遭到破坏。

使用时间戳选项能够有效的防止上述问题，如果丢失的报文会在时刻 F 重新出现，由于它的时间戳为 2，小于最近的有效时间戳（5 或 6），因此防回绕序列号算法（PAWS）会将其丢弃。

防回绕序列号算法要求连接双方维护最近一次收到的数据包的时间戳（Recent TSval），每收到一个新数据包都会读取数据包中的时间戳值跟 Recent TSval 值做比较，**如果发现收到的数据包中时间戳不是递增的，则表示该数据包是过期的，就会直接丢弃这个数据包**。

> 懂了，客户端和服务端的初始化序列号都是随机生成，能很大程度上避免历史报文被下一个相同四元组的连接接收，然后又引入时间戳的机制，从而完全避免了历史报文被接收的问题。

嗯嗯，没错。

______________________________________________________________________

好了，就卷到这了。

自从图解网络 PDF 发布之后，小林又根据读者面试时遇到的网络问题写了很多篇网络相关的文章，这部分内容是可以考虑加入到图解网络 PDF 里的。

而且之前的网络 PDF 有很多读者帮我勘误了很多错别字，所以明年得找个时间重新整理出一个新版本的图解网络 PDF。

回想 2021 年出了《图解网络》和《图解系统》PDF：

[小林的图解系统，大曝光！](http://mp.weixin.qq.com/s?__biz=MzUxODAzNDg4NQ==&mid=2247492900&idx=1&sn=2c1d06a667b1e17e6d8caabff2bbb85b&chksm=f98da18ecefa28986109f13d28c1a06f304cb4d897eb2e931e79dc82d0b054ef6a9f7e1e4e91&scene=21#wechat_redirect)

[不鸽了，小林的「图解网络 3.0 」发布！](http://mp.weixin.qq.com/s?__biz=MzUxODAzNDg4NQ==&mid=2247491944&idx=1&sn=b90deba780ae3840668e21127e467b83&chksm=f98da5c2cefa2cd456045e9b2ed92837ed10e4a2c650f463b29ef5d7f8f4d01014d92225acad&scene=21#wechat_redirect)

经常收到很多好消息，比如：

- 收到读者的打赏，不管多少钱都是对小林的最好的认可；

- 收到读者的感谢，帮助他们面试中击破了网络和系统的八股文面试，我把读者们的故事，都收录到了这里：[读者牛逼](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=MzUxODAzNDg4NQ==&action=getalbum&album_id=1869214323529613315&scene=173&from_msgid=2247493672&from_itemidx=1&count=3&nolastread=1#wechat_redirect)

本来计划今年年末出《图解MySQL》和 《图解Reids》PDF，很可惜没有完成，这个 flag 倒了。

没关系，倒了就倒了，重新纳入 2022 年的计划！重新起航！

感谢大家这一年的支持和陪伴，明年再见！

![](http://mmbiz.qpic.cn/mmbiz_png/J0g14CUwaZfTwwjfpJhXgIrYMgtVcLhQQBVb02clZfKicbxaibSTNJqXe9Zu8ydiavZKJWJAIhKcnD9hBuKU92JZQ/300?wx_fmt=png&wxfrom=19)

**小林coding**

专注图解计算机基础，让天下没有难懂的八股文！刷题网站：xiaolincoding.com

471篇原创内容

公众号

图解网络76

图解网络 · 目录

上一篇被微信面麻了，问的太细节了。。。下一篇美国能让中国从网络上消失？

阅读 7642

​

喜欢此内容的人还喜欢

写留言

**留言 36**

- 程序喵

  2021年12月31日

  赞13

  别卷了，让我们歇歇吧![[破涕为笑]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

- duduSmida

  2021年12月31日

  赞8

  要卷好明年第一天和今年最后一天！！！![[悠闲]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

  小林coding

  作者2021年12月31日

  赞6

  有始有终![[好的]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

- Flash

  2021年12月31日

  赞6

  《图解MySQL》和 《图解Reids》，快冲快冲！！！

- 树下的野草莓

  2021年12月31日

  赞5

  图解网络一路追过来的，看得很爽，放假还要再看。

  小林coding

  作者2021年12月31日

  赞1

  可以，2020年9月关注了，妥妥老粉了！

- 涛歌依旧

  2021年12月31日

  赞3

  明天开始，卷起来

- 编程指北

  2021年12月31日

  赞2

  太卷了![[奸笑]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

  小林coding

  作者2021年12月31日

  赞2

  头发已经被卷飞![[捂脸]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

- 程序员Carl

  2021年12月31日

  赞2

  卧槽，封面真帅![[好的]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

  小林coding

  作者2021年12月31日

  赞1

  ![[奸笑]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)还是卡哥帅点

- 朱晋君

  2021年12月31日

  赞2

  2022年，一起卷

- 阿秀

  2021年12月31日

  赞2

  明年再见，哦不，明天再见![[奸笑]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

  小林coding

  作者2021年12月31日

  赞1

  明年再见，哦不，6个小时再见![[旺柴]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

- Andrew

  2022年1月1日

  赞1

  卷到最后一秒

- 国庆

  2022年1月1日

  赞1

  微微一卷，以示尊敬![😂](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

- GGGG

  2021年12月31日

  赞1

  预约明早起来卷![[旺柴]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

- 雷小帅

  2021年12月31日

  赞1

  不要再卷了![[捂脸]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

- Zero

  2021年12月31日

  赞1

  冲冲冲，卷卷卷！

- 风

  2021年12月31日

  赞1

  我的小姐姐呢？不要小哥哥![[苦涩]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

- 祺噜噜

  2021年12月31日

  赞1

  休假的我看到都害怕

  小林coding

  作者2021年12月31日

  赞1

  还是你香![[旺柴]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

- 慕鹤

  2021年12月31日

  赞1

  怎么有程序猿还有头发的![[旺柴]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![[旺柴]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![[旺柴]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

  小林coding

  作者2021年12月31日

  赞1

  年轻的程序员有![[旺柴]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

- 橡树

  北京2023年11月21日

  赞

  相反，如果每次建立连接客户端和服务端的初始化序列号都「一样」，就有大概率遇到历史报文的序列号刚「好在」对方的接收窗口内 这是为啥？ 虽然都一样，但如果是随机的呢

  1条回复

- 。

  2022年3月10日

  赞

  如果时间戳也回绕了 怎么办

  小林coding

  作者2022年3月10日

  赞

  时间戳回绕的速度只与对端主机时钟频率有关。Linux以本地时钟计数（jiffies）作为时间戳的值，假设时钟计数加1需要1ms，则需要约24.8天才能回绕一半，只要报文的生存时间小于这个值的话判断新旧数据就不会出错。

  小林coding

  作者2022年3月10日

  赞

  linux对这个做了特殊处理，如果一个TCP连接连续24天不收发数据则在接收第一个包时基于时间戳的PAWS会失效，也就是可以PAWS函数会放过这个特殊的情况，认为是合法的。

  小林coding

  作者2022年3月10日

  赞

  我想到的解决办法：增加时间戳的大小，由32 bit扩大到64bit 这样虽然可以在能够预见的未来解决时间戳回绕的问题，但会导致新旧协议兼容性问题，像现在的IPv4与IPv6一样

- 不到叫啥名

  2022年2月24日

  赞

  小弟还是有一点点不懂。在本文举的例子中，因为网络拥堵而延后收到的数据包的序列号是否在服务端的接收窗口内的这件事情，似乎是与前后两次TCP连接中，客户端初始化的ISN是否相同有关，而和客户端的ISN与服务端的ISN是否相同无关。![[委屈]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)（如第二幅TCP三次握手的图中，被延迟的数据包的seq=101，这个seq是根据客户端最开始随机初始化的ISN得出的（100+1），然后第第二次建立连接时，服务端接收窗口的”501~600“中的501，也是由客户端一开始随机初始化的ISN得出的（500+1 = 501）；而101不在这个范围内，因此不会被接收。所以这不是和前后两次客户端随机初始化的ISN是否相同有关吗。![[委屈]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)）

  小林coding

  作者2022年2月24日

  赞

  客户端随机isn，目的就是为了让前后两次tcp连接中的isn不相同，从这一点就可以知道isn必须是随机生成的，所以客户端和服务端都随机生成isn。客户端的isn和服务端的isn基本不会遇到相同的，概率很低，即使遇到相同的isn，也不影响建立连接。

  不到叫啥名

  2022年2月24日

  赞

  感谢

- walker🍀

  2022年1月1日

  赞

  迄今为止，小林的图解网络是我看到最好的，没有之一，祝愿小林在新的一年里再出佳作！

- 冰河

  2022年1月1日

  赞

  继续卷，别停，催更

  小林coding

  作者2022年1月1日

  赞

  今天停更![[奸笑]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

已无更多数据

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/J0g14CUwaZfTwwjfpJhXgIrYMgtVcLhQQBVb02clZfKicbxaibSTNJqXe9Zu8ydiavZKJWJAIhKcnD9hBuKU92JZQ/300?wx_fmt=png&wxfrom=18)

小林coding

关注

45121

36

写留言

**留言 36**

- 程序喵

  2021年12月31日

  赞13

  别卷了，让我们歇歇吧![[破涕为笑]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

- duduSmida

  2021年12月31日

  赞8

  要卷好明年第一天和今年最后一天！！！![[悠闲]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

  小林coding

  作者2021年12月31日

  赞6

  有始有终![[好的]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

- Flash

  2021年12月31日

  赞6

  《图解MySQL》和 《图解Reids》，快冲快冲！！！

- 树下的野草莓

  2021年12月31日

  赞5

  图解网络一路追过来的，看得很爽，放假还要再看。

  小林coding

  作者2021年12月31日

  赞1

  可以，2020年9月关注了，妥妥老粉了！

- 涛歌依旧

  2021年12月31日

  赞3

  明天开始，卷起来

- 编程指北

  2021年12月31日

  赞2

  太卷了![[奸笑]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

  小林coding

  作者2021年12月31日

  赞2

  头发已经被卷飞![[捂脸]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

- 程序员Carl

  2021年12月31日

  赞2

  卧槽，封面真帅![[好的]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

  小林coding

  作者2021年12月31日

  赞1

  ![[奸笑]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)还是卡哥帅点

- 朱晋君

  2021年12月31日

  赞2

  2022年，一起卷

- 阿秀

  2021年12月31日

  赞2

  明年再见，哦不，明天再见![[奸笑]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

  小林coding

  作者2021年12月31日

  赞1

  明年再见，哦不，6个小时再见![[旺柴]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

- Andrew

  2022年1月1日

  赞1

  卷到最后一秒

- 国庆

  2022年1月1日

  赞1

  微微一卷，以示尊敬![😂](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

- GGGG

  2021年12月31日

  赞1

  预约明早起来卷![[旺柴]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

- 雷小帅

  2021年12月31日

  赞1

  不要再卷了![[捂脸]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

- Zero

  2021年12月31日

  赞1

  冲冲冲，卷卷卷！

- 风

  2021年12月31日

  赞1

  我的小姐姐呢？不要小哥哥![[苦涩]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

- 祺噜噜

  2021年12月31日

  赞1

  休假的我看到都害怕

  小林coding

  作者2021年12月31日

  赞1

  还是你香![[旺柴]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

- 慕鹤

  2021年12月31日

  赞1

  怎么有程序猿还有头发的![[旺柴]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![[旺柴]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![[旺柴]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

  小林coding

  作者2021年12月31日

  赞1

  年轻的程序员有![[旺柴]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

- 橡树

  北京2023年11月21日

  赞

  相反，如果每次建立连接客户端和服务端的初始化序列号都「一样」，就有大概率遇到历史报文的序列号刚「好在」对方的接收窗口内 这是为啥？ 虽然都一样，但如果是随机的呢

  1条回复

- 。

  2022年3月10日

  赞

  如果时间戳也回绕了 怎么办

  小林coding

  作者2022年3月10日

  赞

  时间戳回绕的速度只与对端主机时钟频率有关。Linux以本地时钟计数（jiffies）作为时间戳的值，假设时钟计数加1需要1ms，则需要约24.8天才能回绕一半，只要报文的生存时间小于这个值的话判断新旧数据就不会出错。

  小林coding

  作者2022年3月10日

  赞

  linux对这个做了特殊处理，如果一个TCP连接连续24天不收发数据则在接收第一个包时基于时间戳的PAWS会失效，也就是可以PAWS函数会放过这个特殊的情况，认为是合法的。

  小林coding

  作者2022年3月10日

  赞

  我想到的解决办法：增加时间戳的大小，由32 bit扩大到64bit 这样虽然可以在能够预见的未来解决时间戳回绕的问题，但会导致新旧协议兼容性问题，像现在的IPv4与IPv6一样

- 不到叫啥名

  2022年2月24日

  赞

  小弟还是有一点点不懂。在本文举的例子中，因为网络拥堵而延后收到的数据包的序列号是否在服务端的接收窗口内的这件事情，似乎是与前后两次TCP连接中，客户端初始化的ISN是否相同有关，而和客户端的ISN与服务端的ISN是否相同无关。![[委屈]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)（如第二幅TCP三次握手的图中，被延迟的数据包的seq=101，这个seq是根据客户端最开始随机初始化的ISN得出的（100+1），然后第第二次建立连接时，服务端接收窗口的”501~600“中的501，也是由客户端一开始随机初始化的ISN得出的（500+1 = 501）；而101不在这个范围内，因此不会被接收。所以这不是和前后两次客户端随机初始化的ISN是否相同有关吗。![[委屈]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)）

  小林coding

  作者2022年2月24日

  赞

  客户端随机isn，目的就是为了让前后两次tcp连接中的isn不相同，从这一点就可以知道isn必须是随机生成的，所以客户端和服务端都随机生成isn。客户端的isn和服务端的isn基本不会遇到相同的，概率很低，即使遇到相同的isn，也不影响建立连接。

  不到叫啥名

  2022年2月24日

  赞

  感谢

- walker🍀

  2022年1月1日

  赞

  迄今为止，小林的图解网络是我看到最好的，没有之一，祝愿小林在新的一年里再出佳作！

- 冰河

  2022年1月1日

  赞

  继续卷，别停，催更

  小林coding

  作者2022年1月1日

  赞

  今天停更![[奸笑]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

已无更多数据
