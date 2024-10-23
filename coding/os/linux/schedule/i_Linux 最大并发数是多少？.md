
CPP开发者 _2021年11月30日 11:55_

以下文章来源于后端研究所 ，作者程序员大白啊

# 1. 开场白

在开始今天的文章之前，先抛一个面试题出来：

> 你接触过的单机最大并发数是多少？\
> 你认为当前正常配置的服务器物理机最大并发数可以到多少？\
> 说说你的理解和分析。

思考几分钟，如果你可以**有理有据地说出答案**，那确实就不用再往下看了，关上手机去陪陪家人是个不错的选择。

思考几分钟，如果你**没有头绪或者对答案不确定**，那么你先不用着急关闭页面去玩耍，你应该继续往下看，因为这个问题很不错。

对于后端开发人员来说，并发数往往和技术难度是呈正相关的，实际上也确实如此：**体量决定架构**。

服务端根据不同业务场景会有不同的侧重点，单纯追求高并发其实并不是根本目的，**高可用&稳定性更重要**。

所以最终我们的目的是：**保证高可用高稳定的基础上追求高并发，降本增效**。

高可用&高并发是我们直观感受到的，本质上这是个**复杂的系统工程**，每个环节都会影响结果，每一块都值得研究和深入。

![[Pasted image 20241023223743.png]]

# 2. C10K问题和C10M问题

在2000年初的时候，全球互联网的规模并不大，但是当时就已经提出了C10K问题，所谓**C10K就是单机1w并发问题**，虽然现在不觉得是个难题了，但是这在当初是很有远见和挑战的问题。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

C10K问题最早由**Dan Kegel**发布于其个人站点，原文链接如下:

> **http://www.kegel.com/c10k.html**

相关资料显示Dan Kegel目前工作于**Google**，从1978年起开始接触计算机编程，是Winetricks和Crosstool的作者，大佬年轻时的照片：

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

Dan Kegel这篇文章阅读难度并不大，大白建议从事服务端开发或者对高性能网络开发有兴趣的读者尝试读一读。

在APUE第三版都没有提到epoll，所以**我们解决C10K问题的时间并不长**，其中IO复用epoll/kqueue/iocp等技术对于C10k问题的解决起到了非常重要的作用。

开源大神们基于epoll/kqueue等开发了诸如libevent/libuv等网络库，从而大幅提高了高并发网络的开发效率，对于C/C++程序员来说并不陌生。
!\[\[Pasted image 20240923220928.png\]\]

这里简单提一下针对下一个10年的展望和挑战：**C10M问题**。

站在浪尖的那一批人早就开始思考**让单机达到1000w并发**，现在听起来感觉不可思议，但是要达到这个目标，除了硬件上的提升，更重要的是**对系统软件和协议栈的改造**。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

Errata Security的CEO Robert Graham在Shmoocon 2013大会上的演讲，大佬重要的观点是：

> **不要让OS内核执行所有繁重的任务：将数据包处理、内存管理、处理器调度等任务从内核转移到应用程序高效地完成，让诸如Linux这样的OS只处理控制层，数据层完全交给应用程序来处理。**

确实也是如此，**难道你不觉得Linux内核做了太多不该自己做的事情了吗**？

近几年出现的DPDK、PFRING、NETMAP等技术也是类似的思想，现在流行的协处理器+CPU的架构也是这样的：
!\[\[Pasted image 20240923220943.png\]\]
!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

## 3. 服务器最大并发数分析

前面提到的C10K和C10M问题都是围绕着提升服务器并发能力展开的，但是难免要问：**服务器最大的并发上限是多少**？
!\[\[Pasted image 20240923220949.png\]\]

### 3.1 五元组

做过通信的盆友们一定听过**五元组**这个概念，一个五元组可以唯一标记一个网络连接，所以要理解和分析最大并发数，就必须理解五元组：
!\[\[Pasted image 20240923221049.png\]\]
!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

这样的话，就可以基本认为：**理论最大并发数 = 服务端唯一五元组数**。

### 3.2 端口&IP组合数

那么对于服务器来说，服务端唯一五元组数最大是多少呢？

**有人说是65535**，显然不是，但是之所以会有这类答案是因为当前Linux的端口号是2字节大小的short类型，总计2^16个端口，除去一些系统占用的端口，可用端口确实只剩下64000多了。

对于服务端本身来说，DestPort数量确实有限，假定有多张网卡，每个网卡绑定多个IP，**服务端的Port端口数和IP数的组合类型也是有限的**。

对于客户端来说，本身的端口和IP也是一样有限的，虽然这是个**组合问题**，但是数量还是有限的：
!\[\[Pasted image 20240923221055.png\]\]
!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

### 3.3 并发数理论极限

看了前面的端口&IP的组合数计算，好像并发数并不会特别大。

**错了，是真的会很大。**

分析一下，前面的计算都是针对单个服务器或者客户端的，但是**实际上每个服务器会应对全网的所有客户端**，那么从服务端看，源IP和源Port的数量是非常大的。

**理论上服务端可以接受的客户端IP是2^32(按照IPv4计算）,端口数是2^16，目前端口号仍然是16bit的，所有这个理论最大值是2^48**，果然很大！
!\[\[Pasted image 20240923221107.png\]\]
!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

### 3.4 实际情况

天下没有免费的午餐。

**每一条连接都是要消耗系统资源的**，所以实际中可能会设置最大并发数来保证服务器的安全和稳定，所以**这个理论最大并发数是不可能达到的**。

**实际中并发数和业务是直接相关的**，像Redis这种内存型的服务端并发十几万都是没问题的，大部分来讲几十/几百/几千/几万等是存在的。

## 4. 客户端最大连接数

理解了服务器的最大并发数是2^48，那么**客户端最多可以连接多少服务器呢**？
!\[\[Pasted image 20240923221112.png\]\]
!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

对于客户端来说，当然可以借助于多网卡多IP来增加连接能力，我们仍然假定客户端只有1张网卡1个IP，由于端口数的限制到2^16，再去掉系统占用的端口，剩下可用的差不多64000。
!\[\[Pasted image 20240923221119.png\]\]
!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

也就是说，客户端虽然可以连接任意的目的IP和目的端口，但是客户端自身端口是有限的，所以**客户端的理论最大连接数是2^16**，含系统占用端口。

## 5. NAT环境下的客户端

解决前面的两个问题之后，来看另外一个问题：

> **一个公网出口NAT服务设备最多可同时支持多少内网IP并发访问外网服务？**

毕竟公网IP都是有限并且要花钱的，我们大部分机器都是在局域网中结合NAT来进行外网访问的，所以这个场景还是很熟悉的。

来看下**内网机器访问外网时的IP&端口替换和映射还原的过程**，就明白了：
!\[\[Pasted image 20240923221126.png\]\]
!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

因为这时的客户端是NAT设备，所以NAT环境下最多支持65535个并发访问外网。

## 6.小结

本文通过一道面试题切入，先描述了C10K和C10M问题，进而详细说明了客户端的最大访问数和服务端的最大并发数计算和原理，最后描述了NAT场景下的访问并发数。

虽然理论服务端并发数非常大，但是我们也没有必要觉得并发数高就厉害，服务复杂程度不一样，**切忌唯并发数来判断业务和开发者水平**。

试想echo服务和订单交易服务显然是不一样的，我们**应该做的是在服务稳定和高可用的前提下去从缓存/网络/数据库等多个角度来优化提高性能**。

最后感谢各位老铁的倾情阅读。

- EOF -

推荐阅读  点击标题可跳转

1、[这篇 CPU Cache，估计也没人看](http://mp.weixin.qq.com/s?__biz=MzAxNDI5NzEzNg==&mid=2651167790&idx=1&sn=457108d59a01c1683f667f5ab793e2db&chksm=80644d71b713c46708a05930b72259b679afdfc537b675421f48b98a58e0acf3e02625c87cf9&scene=21#wechat_redirect)

2、[深入理解 Linux 内存子系统](http://mp.weixin.qq.com/s?__biz=MzAxNDI5NzEzNg==&mid=2651167786&idx=1&sn=afb37aa02790fa95de818fd2875f7e0c&chksm=80644d75b713c46381cdc5151c3253b0eda60381be02b0b52edc24c1afd9dfb76581c03573ed&scene=21#wechat_redirect)

3、[研究了一波 Android Native C++ 内存泄漏的调试](http://mp.weixin.qq.com/s?__biz=MzAxNDI5NzEzNg==&mid=2651167785&idx=2&sn=1f9c30e3c0689456e760cf06f788120c&chksm=80644d76b713c460316cfc1463065c3f17568d67e8ae4dc60c142b84114ad8bce8ba8ecb595a&scene=21#wechat_redirect)

**关注『CPP开发者』**

看精选C++技术文章 . 加C++开发者专属圈子

![](http://mmbiz.qpic.cn/mmbiz_png/pldYwMfYJpia3uWic6GbPCC1LgjBWzkBVqYrMfbfT6o9uMDnlLELGNgYDP496LvDfiaAiaOt0cZBlBWw4icAs6OHg8Q/300?wx_fmt=png&wxfrom=19)

**CPP开发者**

我们在 Github 维护着 9000+ star 的C语言/C++开发资源。日常分享 C语言 和 C++ 开发相关技术文章，每篇文章都经过精心筛选，一篇文章讲透一个知识点，让读者读有所获～

24篇原创内容

公众号

点赞和在看就是最大的支持❤️

阅读 3785

​

写留言

**留言 2**

- 六斤好爹

  2021年11月30日

  赞9

  设计实现并大规模商用C10M的飘过，文章写的不错

- 新朋

  2021年12月2日

  赞

  楼上直接说出什么商用东西吧

已无更多数据

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/pldYwMfYJpia3uWic6GbPCC1LgjBWzkBVqYrMfbfT6o9uMDnlLELGNgYDP496LvDfiaAiaOt0cZBlBWw4icAs6OHg8Q/300?wx_fmt=png&wxfrom=18)

CPP开发者

20分享6

2

写留言

**留言 2**

- 六斤好爹

  2021年11月30日

  赞9

  设计实现并大规模商用C10M的飘过，文章写的不错

- 新朋

  2021年12月2日

  赞

  楼上直接说出什么商用东西吧

已无更多数据
