# 

原创 bin的技术小屋 bin的技术小屋

_2022年02月23日 09:37_

![](http://mmbiz.qpic.cn/mmbiz_png/sOIZXFW0vUZ9qpdibKBrYASLXXicypdMcnlrAGcicnHQyWYNZvic3C5OpgEicMDGsAcibZTKiaNECcNXDKJiaIBr2XGTow/300?wx_fmt=png&wxfrom=19)

**bin的技术小屋**

专注源码解析系列原创技术文章，分享自己的技术感悟。谈笑有鸿儒，往来无白丁。无丝竹之乱耳，无案牍之劳形。斯是陋室，惟吾德馨。

37篇原创内容

公众号

> 本系列Netty源码解析文章基于 **4.1.56.Final**版本

![图片](https://mmbiz.qpic.cn/mmbiz_png/sOIZXFW0vUbsJJoRlck3EIX1zd6sgNrGvtTrBQnwUILUgN2L3QHuS0VPeYo81EZ4ARbZIk8XE1EhbQ5eeKRIWQ/640?wx_fmt=png&wxfrom=13&tp=wxpic)

本文概览.png

## 前文回顾

在前边的系列文章中，我们从内核如何收发网络数据开始以一个C10K的问题作为主线详细从内核角度阐述了网络IO模型的演变，最终在此基础上引出了Netty的网络IO模型如下图所示：

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

netty中的reactor.png

> 详细内容可回看[?《从内核角度看IO模型的演变》](https://mp.weixin.qq.com/s?__biz=Mzg2MzU3Mjc3Ng==&mid=2247483737&idx=1&sn=7ef3afbb54289c6e839eed724bb8a9d6&chksm=ce77c71ef9004e08e3d164561e3a2708fc210c05408fa41f7fe338d8e85f39c1ad57519b614e&scene=21#wechat_redirect)

后续我们又围绕着Netty的主从Reactor网络IO线程模型，在[?《Reactor模型在Netty中的实现》](https://mp.weixin.qq.com/s?__biz=Mzg2MzU3Mjc3Ng==&mid=2247483907&idx=1&sn=084c470a8fe6234c2c9461b5f713ff30&chksm=ce77c444f9004d52e7c6244bee83479070effb0bc59236df071f4d62e91e25f01715fca53696&scene=21#wechat_redirect)一文中详细阐述了Netty的主从Reactor模型的创建，以及介绍了Reactor模型的关键组件。搭建了Netty的核心骨架如下图所示：

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

主从Reactor线程组.png

在核心骨架搭建完毕之后，我们随后又在[?《详细图解Reactor启动全流程》](https://mp.weixin.qq.com/s?__biz=Mzg2MzU3Mjc3Ng==&mid=2247484005&idx=1&sn=52f51269902a58f40d33208421109bc3&chksm=ce77c422f9004d340e5b385ef6ba24dfba1f802076ace80ad6390e934173a10401e64e13eaeb&scene=21#wechat_redirect)一文中阐述了Reactor启动的全流程，一个非常重要的核心组件NioServerSocketChannel开始在这里初次亮相，承担着一个网络框架最重要的任务--高效接收网络连接。我们介绍了NioServerSocketChannel的创建，初始化，向Main Reactor注册并监听OP_ACCEPT事件的整个流程。在此基础上，Netty得以整装待发，枕戈待旦开始迎接海量的客户端连接。

![图片](https://mmbiz.qpic.cn/mmbiz_png/sOIZXFW0vUbsJJoRlck3EIX1zd6sgNrGsn4fFaBGsA3Qib2QmxiaehIiaic7HZgR7ibVQKMLmY0ul012J2ic3I44Pr5A/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

Reactor启动后的结构.png

随后紧接着我们在[?《Netty如何高效接收网络连接》](https://mp.weixin.qq.com/s?__biz=Mzg2MzU3Mjc3Ng==&mid=2247484184&idx=1&sn=726877ce28cf6e5d2ac3225fae687f19&chksm=ce77c55ff9004c493b592288819dc4d4664b5949ee97fed977b6558bc517dad0e1f73fab0f46&scene=21#wechat_redirect)一文中详细介绍了Netty高效接收客户端网络连接的全流程，在这里Netty的核心重要组件NioServerSocketChannel开始正是登场，在NioServerSocketChannel中我们创建了客户端连接NioSocketChannel，并详细介绍了NioSocketChannel的初始化过程，随后通过在NioServerSocketChannel的pipeline中触发ChannelRead事件，并最终在ServerBootstrapAcceptor中将客户端连接NioSocketChannel注册到Sub Reactor中开始监听客户端连接上的OP_READ事件，准备接收客户端发送的网络数据也就是本文的主题内容。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

image.png

自此Netty的核心组件全部就绪并启动完毕，开始起飞~~~

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

主从Reactor组完整结构.png

之前文章中的主角是Netty中主Reactor组中的Main Reactor以及注册在Main Reactor上边的NioServerSocketChannel，那么从本文开始，我们文章中的主角就切换为Sub Reactor以及注册在SubReactor上的NioSocketChannel了。

下面就让我们正式进入今天的主题，看一下Netty是如何处理OP_READ事件以及如何高效接收网络数据的。

## 1. Sub Reactor处理OP_READ事件流程总览

![图片](https://mmbiz.qpic.cn/mmbiz_png/sOIZXFW0vUbsJJoRlck3EIX1zd6sgNrGJCjQJqYs13qZfBceLVWKYiaQBw4Y6jp79WthNSdbb2cNlibR7dNnKBag/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

OP_READ事件处理.png

客户端发起系统IO调用向服务端发送数据之后，当网络数据到达服务端的网卡并经过内核协议栈的处理，最终数据到达Socket的接收缓冲区之后，Sub Reactor轮询到NioSocketChannel上的`OP_READ事件`就绪，随后Sub Reactor线程就会从JDK Selector上的阻塞轮询API`selector.select(timeoutMillis)`调用中返回。转而去处理NioSocketChannel上的`OP_READ事件`。

> 注意这里的Reactor为负责处理客户端连接的Sub Reactor。连接的类型为NioSocketChannel，处理的事件为OP_READ事件。

在之前的文章中笔者已经多次强调过了，Reactor在处理Channel上的IO事件入口函数为`NioEventLoop#processSelectedKey`。

`public final class NioEventLoop extends SingleThreadEventLoop {          private void processSelectedKey(SelectionKey k, AbstractNioChannel ch) {           final AbstractNioChannel.NioUnsafe unsafe = ch.unsafe();           ..............省略.................              try {               int readyOps = k.readyOps();                  if ((readyOps & SelectionKey.OP_CONNECT) != 0) {                  ..............处理OP_CONNECT事件.................               }                     if ((readyOps & SelectionKey.OP_WRITE) != 0) {                 ..............处理OP_WRITE事件.................               }                     if ((readyOps & (SelectionKey.OP_READ | SelectionKey.OP_ACCEPT)) != 0 || readyOps == 0) {                   //本文重点处理OP_ACCEPT事件                   unsafe.read();               }           } catch (CancelledKeyException ignored) {               unsafe.close(unsafe.voidPromise());           }       }      }   `

这里需要重点强调的是，当前的执行线程现在已经变成了Sub Reactor，而Sub Reactor上注册的正是netty客户端NioSocketChannel负责处理连接上的读写事件。

所以这里入口函数的参数`AbstractNioChannel ch`则是IO就绪的客户端连接`NioSocketChannel`。

开头通过`ch.unsafe()`获取到的NioUnsafe操作类正是NioSocketChannel中对底层JDK NIO SocketChannel的Unsafe底层操作类。实现类型为`NioByteUnsafe`定义在下图继承结构中的`AbstractNioByteChannel`父类中。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

image.png

下面我们到`NioByteUnsafe#read`方法中来看下Netty对`OP_READ事件`的具体处理过程：

## 2. Netty接收网络数据流程总览

我们直接按照老规矩，先从整体上把整个OP_READ事件的逻辑处理框架提取出来，让大家先总体俯视下流程全貌，然后在针对每个核心点位进行各个击破。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

Netty接收网络数据流程.png

> 流程中相关置灰的步骤为Netty处理连接关闭时的逻辑，和本文主旨无关，我们这里暂时忽略，等后续笔者介绍连接关闭时，会单独开一篇文章详细为大家介绍。

从上面这张Netty接收网络数据总体流程图可以看出NioSocketChannel在接收网络数据的整个流程和我们在上篇文章[?《Netty如何高效接收网络连接》](https://mp.weixin.qq.com/s?__biz=Mzg2MzU3Mjc3Ng==&mid=2247484184&idx=1&sn=726877ce28cf6e5d2ac3225fae687f19&chksm=ce77c55ff9004c493b592288819dc4d4664b5949ee97fed977b6558bc517dad0e1f73fab0f46&scene=21#wechat_redirect)中介绍的NioServerSocketChannel在接收客户端连接时的流程在总体框架上是一样的。

NioSocketChannel在接收网络数据的过程处理中，也是通过在一个`do{....}while(...)`循环read loop中不断的循环读取连接NioSocketChannel上的数据。

同样在NioSocketChannel读取连接数据的read loop中也是受最大读取次数的限制。默认配置最多只能读取16次，超过16次无论此时NioSocketChannel中是否还有数据可读都不能在进行读取了。

这里read loop循环最大读取次数可在启动配置类ServerBootstrap中通过`ChannelOption.MAX_MESSAGES_PER_READ`选项设置，默认为16。

`ServerBootstrap b = new ServerBootstrap();   b.group(bossGroup, workerGroup)     .channel(NioServerSocketChannel.class)     .option(ChannelOption.MAX_MESSAGES_PER_READ, 自定义次数)   `

\*\*Netty这里为什么非得限制read loop的最大读取次数呢？\*\*为什么不在read loop中一次性把数据读取完呢？

这时候就是考验我们大局观的时候了，在前边的文章介绍中我们提到Netty的IO模型为主从Reactor线程组模型，在Sub Reactor Group中包含了多个Sub Reactor专门用于监听处理客户端连接上的IO事件。

为了能够高效有序的处理全量客户端连接上的读写事件，Netty将服务端承载的全量客户端连接分摊到多个Sub Reactor中处理，同时也能保证`Channel上IO处理的线程安全性`。

其中一个Channel只能分配给一个固定的Reactor。一个Reactor负责处理多个Channel上的IO就绪事件，Reactor与Channel之间的对应关系如下图所示：

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

image.png

而一个Sub Reactor上注册了多个NioSocketChannel，Netty不可能在一个NioSocketChannel上无限制的处理下去，要将读取数据的机会均匀分摊给其他NioSocketChannel，所以需要限定每个NioSocketChannel上的最大读取次数。

此外，Sub Reactor除了需要监听处理所有注册在它上边的NioSocketChannel中的IO就绪事件之外，还需要腾出事件来处理有用户线程提交过来的异步任务。从这一点看，Netty也不会一直停留在NioSocketChannel的IO处理上。所以限制read loop的最大读取次数是非常必要的。

> 关于Reactor的整体运转架构，对细节部分感兴趣的同学可以回看下笔者的[?《一文聊透Netty核心引擎Reactor的运转架构》](https://mp.weixin.qq.com/s?__biz=Mzg2MzU3Mjc3Ng==&mid=2247484087&idx=1&sn=0c065780e0f05c23c8e6465ede86cba0&chksm=ce77c4f0f9004de63be369a664105708bc5975b52993f4a6df223caed34cc1ef6185a16acd75&scene=21#wechat_redirect)这篇文章。

所以基于这个原因，我们需要在read loop循环中，每当通过`doReadBytes`方法从NioSocketChannel中读取到数据时（方法返回值会大于0，并记录在allocHandle.lastBytesRead中），都需要通过`allocHandle.incMessagesRead(1)`方法统计已经读取的次数。当达到16次时不管NioSocketChannel是否还有数据可读，都需要在read loop末尾退出循环。转去执行Sub Reactor上的异步任务。以及其他NioSocketChannel上的IO就绪事件。平均分配，雨露均沾！！

`public abstract class MaxMessageHandle implements ExtendedHandle {              //read loop总共读取了多少次           private int totalMessages;             @Override           public final void incMessagesRead(int amt) {               totalMessages += amt;           }      }   `

本次read loop读取到的数据大小会记录在`allocHandle.lastBytesRead`中

`public abstract class MaxMessageHandle implements ExtendedHandle {               //本次read loop读取到的字节数           private int lastBytesRead;           //整个read loop循环总共读取的字节数           private int totalBytesRead;              @Override           public void lastBytesRead(int bytes) {               lastBytesRead = bytes;               if (bytes > 0) {                   totalBytesRead += bytes;               }           }   }   `

- `lastBytesRead < 0`：表示客户端主动发起了连接关闭流程，Netty开始连接关闭处理流程。这个和本文的主旨无关，我们先不用管。后面笔者会专门用一篇文章来详解关闭流程。

- `lastBytesRead = 0`：表示当前NioSocketChannel上的数据已经全部读取完毕，没有数据可读了。本次OP_READ事件圆满处理完毕，可以开开心心的退出read loop。

- 当`lastBytesRead > 0`：表示在本次read loop中从NioSocketChannel中读取到了数据，会在NioSocketChannel的pipeline中触发ChannelRead事件。进而在pipeline中负责IO处理的ChannelHandelr中响应，处理网络请求。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

fireChannelRread.png

`public class EchoServerHandler extends ChannelInboundHandlerAdapter {          @Override       public void channelRead(ChannelHandlerContext ctx, Object msg) {             .......处理网络请求，比如解码,反序列化等操作.......       }   }   `

最后会在read loop循环的末尾调用`allocHandle.continueReading()`判断是否结束本次read loop循环。这里的结束循环条件的判断会比我们在介绍NioServerSocketChannel接收连接时的判断条件复杂很多，笔者会将这个判断条件的详细解析放在文章后面细节部分为大家解读，这里大家只需要把握总体核心流程，不需要关注太多细节。

总体上在NioSocketChannel中读取网络数据的read loop循环结束条件需要满足以下几点：

- 当前NioSocketChannel中的数据已经全部读取完毕，则退出循环。

- 本轮read loop如果没有读到任何数据，则退出循环。

- read loop的读取次数达到16次，退出循环。

当满足这里的read loop退出条件之后，Sub Reactor线程就会退出循环，随后会调用`allocHandle.readComplete()`方法根据本轮read loop总共读取到的字节数`totalBytesRead`来决定是否对用于接收下一轮OP_READ事件数据的ByteBuffer进行扩容或者缩容。

最后在NioSocketChannel的pipeline中触发`ChannelReadComplete事件`，通知ChannelHandler本次OP_READ事件已经处理完毕。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

fireChannelReadComplete.png

`    public class EchoServerHandler extends ChannelInboundHandlerAdapter {          @Override       public void channelRead(ChannelHandlerContext ctx, Object msg) {          .......处理网络请求，比如解码,反序列化等操作.......       }          @Override       public void channelReadComplete(ChannelHandlerContext ctx) {           ......本次OP_READ事件处理完毕.......           ......决定是否向客户端响应处理结果......       }   }    `

### 2.1 ChannelRead与ChannelReadComplete事件的区别

> 有些小伙伴可能对Netty中的一些传播事件触发的时机，或者事件之间的区别理解的不是很清楚，概念容易混淆。在后面的文章中笔者也会从源码的角度出发给大家说清楚Netty中定义的所有异步事件，以及这些事件之间的区别和联系和触发时机，传播机制。

这里我们主要探讨本文主题中涉及到的两个事件：ChannelRead事件与ChannelReadComplete事件。

从上述介绍的Netty接收网络数据流程总览中我们可以看出`ChannelRead事件`和`ChannelReadComplete事件`是不一样的，但是对于刚接触Netty的小伙伴来说从命名上乍一看感觉又差不多。

下面我们来看这两个事件之间的差别：

Netty服务端对于一次OP_READ事件的处理，会在一个`do{}while()`循环read loop中分多次从客户端NioSocketChannel中读取网络数据。每次读取我们分配的ByteBuffer容量大小，初始容量为2048。

- `ChanneRead事件`：一次循环读取一次数据，就触发一次`ChannelRead事件`。本次最多读取在read loop循环开始分配的DirectByteBuffer容量大小。这个容量会动态调整，文章后续笔者会详细介绍。

- `ChannelReadComplete事件`：当读取不到数据或者不满足`continueReading`的任意一个条件就会退出read loop，这时就会触发`ChannelReadComplete事件`。表示本次`OP_READ事件`处理完毕。

> **这里需要特别注意下**触发`ChannelReadComplete事件`并不代表NioSocketChannel中的数据已经读取完了，只能说明本次`OP_READ事件`处理完毕。因为有可能是客户端发送的数据太多，Netty读了`16次`还没读完，那就只能等到下次`OP_READ事件`到来的时候在进行读取了。

______________________________________________________________________

以上内容就是Netty在接收客户端发送网络数据的全部核心逻辑。目前为止我们还未涉及到这部分的主干核心源码，笔者想的是先给大家把核心逻辑讲解清楚之后，这样理解起来核心主干源码会更加清晰透彻。

经过前边对网络数据接收的核心逻辑介绍，笔者在把这张流程图放出来，大家可以结合这张图在来回想下主干核心逻辑。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

Netty接收网络数据流程.png

下面笔者会结合这张流程图，给大家把这部分的核心主干源码框架展现出来，大家可以将我们介绍过的核心逻辑与主干源码做个一一对应，还是那句老话，我们要从主干框架层面把握整体处理流程，不需要读懂每一行代码，文章后续笔者会将这个过程中涉及到的核心点位给大家拆开来各个击破！！

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

image.png

## 3. 源码核心框架总览

`@Override           public final void read() {               final ChannelConfig config = config();                  ...............处理半关闭相关代码省略...............               //获取NioSocketChannel的pipeline               final ChannelPipeline pipeline = pipeline();               //PooledByteBufAllocator 具体用于实际分配ByteBuf的分配器               final ByteBufAllocator allocator = config.getAllocator();               //自适应ByteBuf分配器 AdaptiveRecvByteBufAllocator ,用于动态调节ByteBuf容量               //需要与具体的ByteBuf分配器配合使用 比如这里的PooledByteBufAllocator               final RecvByteBufAllocator.Handle allocHandle = recvBufAllocHandle();               //allocHandler用于统计每次读取数据的大小，方便下次分配合适大小的ByteBuf               //重置清除上次的统计指标               allocHandle.reset(config);                  ByteBuf byteBuf = null;               boolean close = false;               try {                   do {                       //利用PooledByteBufAllocator分配合适大小的byteBuf 初始大小为2048                       byteBuf = allocHandle.allocate(allocator);                       //记录本次读取了多少字节数                       allocHandle.lastBytesRead(doReadBytes(byteBuf));                       //如果本次没有读取到任何字节，则退出循环 进行下一轮事件轮询                       if (allocHandle.lastBytesRead() <= 0) {                           // nothing was read. release the buffer.                           byteBuf.release();                           byteBuf = null;                           close = allocHandle.lastBytesRead() < 0;                           if (close) {                               ......表示客户端发起连接关闭.....                           }                           break;                       }                          //read loop读取数据次数+1                       allocHandle.incMessagesRead(1);                       //客户端NioSocketChannel的pipeline中触发ChannelRead事件                       pipeline.fireChannelRead(byteBuf);                       //解除本次读取数据分配的ByteBuffer引用，方便下一轮read loop分配                       byteBuf = null;                   } while (allocHandle.continueReading());//判断是否应该继续read loop                      //根据本次read loop总共读取的字节数，决定下次是否扩容或者缩容                   allocHandle.readComplete();                   //在NioSocketChannel的pipeline中触发ChannelReadComplete事件，表示一次read事件处理完毕                   //但这并不表示 客户端发送来的数据已经全部读完，因为如果数据太多的话，这里只会读取16次，剩下的会等到下次read事件到来后在处理                   pipeline.fireChannelReadComplete();                      .........省略连接关闭流程处理.........               } catch (Throwable t) {                   ...............省略...............               } finally {                  ...............省略...............               }           }       }`

> 这里再次强调下当前执行线程为Sub Reactor线程，处理连接数据读取逻辑是在NioSocketChannel中。

首先通过`config()`获取客户端NioSocketChannel的Channel配置类NioSocketChannelConfig。

通过`pipeline()`获取NioSocketChannel的pipeline。我们在[?《详细图解Netty Reactor启动全流程》](https://mp.weixin.qq.com/s?__biz=Mzg2MzU3Mjc3Ng==&mid=2247484005&idx=1&sn=52f51269902a58f40d33208421109bc3&chksm=ce77c422f9004d340e5b385ef6ba24dfba1f802076ace80ad6390e934173a10401e64e13eaeb&token=1943348780&lang=zh_CN&scene=21#wechat_redirect)一文中提到的Netty服务端模板所举的示例中，NioSocketChannelde pipeline中只有一个EchoChannelHandler。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

客户端channel pipeline结构.png

### 3.1 分配DirectByteBuffer接收网络数据

Sub Reactor在接收NioSocketChannel上的IO数据时，都会分配一个ByteBuffer用来存放接收到的IO数据。

这里大家可能觉得比较奇怪，为什么在NioSocketChannel接收数据这里会有两个ByteBuffer分配器呢？一个是ByteBufAllocator，另一个是RecvByteBufAllocator。

`final ByteBufAllocator allocator = config.getAllocator();       final RecvByteBufAllocator.Handle allocHandle = recvBufAllocHandle();`

**这两个ByteBuffer又有什么区别和联系呢？**

在上篇文章[?《抓到Netty一个Bug，顺带来透彻地聊一下Netty是如何高效接收网络连接》](https://mp.weixin.qq.com/s?__biz=Mzg2MzU3Mjc3Ng==&mid=2247484184&idx=1&sn=726877ce28cf6e5d2ac3225fae687f19&chksm=ce77c55ff9004c493b592288819dc4d4664b5949ee97fed977b6558bc517dad0e1f73fab0f46&scene=21&cur_album_id=2217816582418956300#wechat_redirect)中，笔者为了阐述上篇文章中提到的Netty在接收网络连接时的Bug时，简单和大家介绍了下这个RecvByteBufAllocator。

在上篇文章提到的NioServerSocketChannelConfig中，这里的RecvByteBufAllocator类型为ServerChannelRecvByteBufAllocator。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

image.png

> 还记得这个ServerChannelRecvByteBufAllocator类型在**4.1.69.final**版本引入是为了解决笔者在上篇文章中提到的那个Bug吗？在**4.1.69.final**版本之前，NioServerSocketChannelConfig中的RecvByteBufAllocator类型为AdaptiveRecvByteBufAllocator。

而在本文中NioSocketChannelConfig中的RecvByteBufAllocator类型为AdaptiveRecvByteBufAllocator。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

image.png

所以这里`recvBufAllocHandle()`获得到的RecvByteBufAllocator为AdaptiveRecvByteBufAllocator。顾名思义，这个类型的RecvByteBufAllocator可以根据NioSocketChannel上每次到来的IO数据大小来自适应动态调整ByteBuffer的容量。

对于客户端NioSocketChannel来说，它里边包含的IO数据时客户端发送来的网络数据，长度是不定的，所以才会需要这样一个可以根据每次IO数据的大小来自适应动态调整容量的ByteBuffer来接收。

如果我们把用于接收数据用的ByteBuffer看做一个桶的话，那么小数据用大桶装或者大数据用小桶装肯定是不合适的，所以我们需要根据接收数据的大小来动态调整桶的容量。而AdaptiveRecvByteBufAllocator的作用正是用来根据每次接收数据的容量大小来动态调整ByteBuffer的容量的。

现在RecvByteBufAllocator笔者为大家解释清楚了，接下来我们继续看ByteBufAllocator。

> 大家这里需要注意的是AdaptiveRecvByteBufAllocator并不会真正的去分配ByteBuffer，它只是负责动态调整分配ByteBuffer的大小。

而真正具体执行内存分配动作的是这里的ByteBufAllocator类型为PooledByteBufAllocator。它会根据AdaptiveRecvByteBufAllocator动态调整出来的大小去真正的申请内存分配ByteBuffer。

> PooledByteBufAllocator为Netty中的内存池，用来管理堆外内存DirectByteBuffer。

AdaptiveRecvByteBufAllocator中的allocHandle在上篇文章中我们也介绍过了，它的实际类型为MaxMessageHandle。

`public class AdaptiveRecvByteBufAllocator extends DefaultMaxMessagesRecvByteBufAllocator {          @Override       public Handle newHandle() {           return new HandleImpl(minIndex, maxIndex, initial);       }              private final class HandleImpl extends MaxMessageHandle {                     .................省略................       }   }   `

在MaxMessageHandle中包含了用于动态调整ByteBuffer容量的统计指标。

`public abstract class MaxMessageHandle implements ExtendedHandle {           private ChannelConfig config;              //用于控制每次read loop里最大可以循环读取的次数，默认为16次           //可在启动配置类ServerBootstrap中通过ChannelOption.MAX_MESSAGES_PER_READ选项设置。           private int maxMessagePerRead;              //用于统计read loop中总共接收的连接个数，NioSocketChannel中表示读取数据的次数           //每次read loop循环后会调用allocHandle.incMessagesRead增加记录接收到的连接个数           private int totalMessages;              //用于统计在read loop中总共接收到客户端连接上的数据大小           private int totalBytesRead;              //表示本次read loop 尝试读取多少字节，byteBuffer剩余可写的字节数           private int attemptedBytesRead;              //本次read loop读取到的字节数           private int lastBytesRead;                      //预计下一次分配buffer的容量，初始：2048           private int nextReceiveBufferSize;           ...........省略.............   }`

在每轮read loop开始之前，都会调用`allocHandle.reset(config)`重置清空上一轮read loop的统计指标。

`@Override           public void reset(ChannelConfig config) {               this.config = config;               //默认每次最多读取16次               maxMessagePerRead = maxMessagesPerRead();               totalMessages = totalBytesRead = 0;           }`

在每次开始从NioSocketChannel中读取数据之前，需要利用`PooledByteBufAllocator`在内存池中为ByteBuffer分配内存，默认初始化大小为`2048`，这个容量由`guess()方法`决定。

`byteBuf = allocHandle.allocate(allocator);`

`@Override           public ByteBuf allocate(ByteBufAllocator alloc) {               return alloc.ioBuffer(guess());           }              @Override           public int guess() {               //预计下一次分配buffer的容量，一开始为2048               return nextReceiveBufferSize;           }`

在每次通过`doReadBytes`从NioSocketChannel中读取到数据后，都会调用`allocHandle.lastBytesRead(doReadBytes(byteBuf))`记录本次读取了多少字节数据，并统计本轮read loop目前总共读取了多少字节。

`@Override           public void lastBytesRead(int bytes) {               lastBytesRead = bytes;               if (bytes > 0) {                   totalBytesRead += bytes;               }           }`

每次循环从NioSocketChannel中读取数据之后，都会调用`allocHandle.incMessagesRead(1)`。统计当前已经读取了多少次。如果超过了最大读取限制此时16次，就需要退出read loop。去处理其他NioSocketChannel上的IO事件。

`@Override           public final void incMessagesRead(int amt) {               totalMessages += amt;           }`

在每次read loop循环的末尾都需要通过调用`allocHandle.continueReading()`来判断是否继续read loop循环读取NioSocketChannel中的数据。

`@Override           public boolean continueReading() {               return continueReading(defaultMaybeMoreSupplier);           }              private final UncheckedBooleanSupplier defaultMaybeMoreSupplier = new UncheckedBooleanSupplier() {               @Override               public boolean get() {                   //判断本次读取byteBuffer是否满载而归                   return attemptedBytesRead == lastBytesRead;               }           };              @Override           public boolean continueReading(UncheckedBooleanSupplier maybeMoreDataSupplier) {               return config.isAutoRead() &&                      (!respectMaybeMoreData || maybeMoreDataSupplier.get()) &&                      totalMessages < maxMessagePerRead &&                      totalBytesRead > 0;           }`

- `attemptedBytesRead :`表示当前ByteBuffer预计尝试要写入的字节数。

- `lastBytesRead :`表示本次read loop真实读取到了多少个字节。

`defaultMaybeMoreSupplier`用于判断经过本次read loop读取数据后，ByteBuffer是否满载而归。如果是满载而归的话（attemptedBytesRead == lastBytesRead），表明可能NioSocketChannel里还有数据。如果不是满载而归，表明NioSocketChannel里没有数据了已经。

是否继续进行read loop需要**同时**满足以下几个条件：

- `totalMessages < maxMessagePerRead` 当前读取次数是否已经超过`16次`，如果超过，就退出`do(...)while()`循环。进行下一轮`OP_READ事件`的轮询。因为每个Sub Reactor管理了多个NioSocketChannel，不能在一个NioSocketChannel上占用太多时间，要将机会均匀地分配给Sub Reactor所管理的所有NioSocketChannel。

- `totalBytesRead > 0` 本次`OP_READ事件`处理是否读取到了数据，如果已经没有数据可读了，那么就直接退出read loop。

- `!respectMaybeMoreData || maybeMoreDataSupplier.get()` 这个条件比较复杂，它其实就是通过`respectMaybeMoreData`字段来控制NioSocketChannel中可能还有数据可读的情况下该如何处理。

- `maybeMoreDataSupplier.get()`：true表示本次读取从NioSocketChannel中读取数据，ByteBuffer满载而归。说明可能NioSocketChannel中还有数据没读完。fasle表示ByteBuffer还没有装满，说明NioSocketChannel中已经没有数据可读了。

- `respectMaybeMoreData = true`表示要对可能还有更多数据进行处理的这种情况要`respect`认真对待,如果本次循环读取到的数据已经装满`ByteBuffer`，表示后面可能还有数据，那么就要进行读取。如果`ByteBuffer`还没装满表示已经没有数据可读了那么就退出循环。!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

- `respectMaybeMoreData = false`表示对可能还有更多数据的这种情况不认真对待 `not respect`。不管本次循环读取数据`ByteBuffer`是否满载而归，都要继续进行读取，直到读取不到数据在退出循环，属于无脑读取。

同时满足以上三个条件，那么read loop继续进行。继续从NioSocketChannel中读取数据，直到读取不到或者不满足三个条件中的任意一个为止。

### 3.2 从NioSocketChannel中读取数据

`public class NioSocketChannel extends AbstractNioByteChannel implements io.netty.channel.socket.SocketChannel {          @Override       protected int doReadBytes(ByteBuf byteBuf) throws Exception {           final RecvByteBufAllocator.Handle allocHandle = unsafe().recvBufAllocHandle();           allocHandle.attemptedBytesRead(byteBuf.writableBytes());               return byteBuf.writeBytes(javaChannel(), allocHandle.attemptedBytesRead());       }   }   `

这里会直接调用底层JDK NIO的`SocketChannel#read`方法将数据读取到DirectByteBuffer中。读取数据大小为本次分配的DirectByteBuffer容量，初始为2048。

## 4. ByteBuffer动态自适应扩缩容机制

由于我们一开始并不知道客户端会发送多大的网络数据，所以这里先利用`PooledByteBufAllocator`分配一个初始容量为`2048`的DirectByteBuffer用于接收数据。

`byteBuf = allocHandle.allocate(allocator);`

这就好比我们需要拿着一个桶去排队装水，但是第一次去装的时候，我们并不知道管理员会给我们分配多少水，桶拿大了也不合适拿小了也不合适，于是我们就先预估一个差不多容量大小的桶，如果分配的多了，我们下次就拿更大一点的桶，如果分配少了，下次我们就拿一个小点的桶。

在这种场景下，我们需要ByteBuffer可以自动根据每次网络数据的大小来动态自适应调整自己的容量。

而ByteBuffer动态自适应扩缩容机制依赖于AdaptiveRecvByteBufAllocator类的实现。让我们先回到AdaptiveRecvByteBufAllocator类的创建起点开始说起~~

### 4.1 AdaptiveRecvByteBufAllocator的创建

在前文[?《Netty是如何高效接收网络连接》](https://mp.weixin.qq.com/s?__biz=Mzg2MzU3Mjc3Ng==&mid=2247484184&idx=1&sn=726877ce28cf6e5d2ac3225fae687f19&chksm=ce77c55ff9004c493b592288819dc4d4664b5949ee97fed977b6558bc517dad0e1f73fab0f46&scene=21&cur_album_id=2217816582418956300#wechat_redirect)中我们提到，当Main Reactor监听到OP_ACCPET事件活跃后，会在NioServerSocketChannel中accept完成三次握手的客户端连接。并创建NioSocketChannel，伴随着NioSocketChannel的创建其对应的配置类NioSocketChannelConfig类也会随之创建。

`public NioSocketChannel(Channel parent, SocketChannel socket) {           super(parent, socket);           config = new NioSocketChannelConfig(this, socket.socket());       }`

最终会在NioSocketChannelConfig的父类`DefaultChannelConfig`的构造器中创建`AdaptiveRecvByteBufAllocator`。并保存在`RecvByteBufAllocator rcvBufAllocator`字段中。

`public class DefaultChannelConfig implements ChannelConfig {          //用于Channel接收数据用的buffer分配器  AdaptiveRecvByteBufAllocator       private volatile RecvByteBufAllocator rcvBufAllocator;          public DefaultChannelConfig(Channel channel) {               this(channel, new AdaptiveRecvByteBufAllocator());       }      }   `

在`new AdaptiveRecvByteBufAllocator()`创建AdaptiveRecvByteBufAllocator类实例的时候会先触发AdaptiveRecvByteBufAllocator类的初始化。

我们先来看下AdaptiveRecvByteBufAllocator类的初始化都做了些什么事情：

### 4.2 AdaptiveRecvByteBufAllocator类的初始化

`public class AdaptiveRecvByteBufAllocator extends DefaultMaxMessagesRecvByteBufAllocator {          //扩容步长       private static final int INDEX_INCREMENT = 4;       //缩容步长       private static final int INDEX_DECREMENT = 1;          //RecvBuf分配容量表（扩缩容索引表）按照表中记录的容量大小进行扩缩容       private static final int[] SIZE_TABLE;         static {           //初始化RecvBuf容量分配表           List<Integer> sizeTable = new ArrayList<Integer>();           //当分配容量小于512时，扩容单位为16递增           for (int i = 16; i < 512; i += 16) {               sizeTable.add(i);           }              //当分配容量大于512时，扩容单位为一倍           for (int i = 512; i > 0; i <<= 1) {               sizeTable.add(i);           }              //初始化RecbBuf扩缩容索引表           SIZE_TABLE = new int[sizeTable.size()];           for (int i = 0; i < SIZE_TABLE.length; i ++) {               SIZE_TABLE[i] = sizeTable.get(i);           }       }   }   `

AdaptiveRecvByteBufAllocator 主要的作用就是为接收数据的`ByteBuffer`进行扩容缩容，那么每次怎么扩容？扩容多少？怎么缩容？缩容多少呢？？

这四个问题将是本小节笔者要为大家解答的内容~~~

Netty中定义了一个`int型`的数组`SIZE_TABLE`来存储每个扩容单位对应的容量大小。建立起扩缩容的容量索引表。每次扩容多少，缩容多少全部记录在这个容量索引表中。

在AdaptiveRecvByteBufAllocatorl类初始化的时候会在`static{}`静态代码块中对扩缩容索引表`SIZE_TABLE`进行初始化。

从源码中我们可以看出`SIZE_TABLE`的初始化分为两个部分：

- 当索引容量小于`512`时，`SIZE_TABLE`中定义的容量索引是从`16开始`按`16`递增。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

image.png

- 当索引容量大于`512`时，`SIZE_TABLE`中定义的容量索引是按前一个索引容量的2倍递增。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

image.png

### 4.3 扩缩容逻辑

现在扩缩容索引表`SIZE_TABLE`已经初始化完毕了，那么当我们需要对`ByteBuffer`进行扩容或者缩容的时候如何根据`SIZE_TABLE`决定扩容多少或者缩容多少呢？？

这就用到了在AdaptiveRecvByteBufAllocator类中定义的扩容步长`INDEX_INCREMENT = 4`，缩容步长`INDEX_DECREMENT = 1`了。

我们就以上面两副扩缩容容量索引表`SIZE_TABLE`中的容量索引展示截图为例，来介绍下扩缩容逻辑，假设我们当前`ByteBuffer`的容量索引为`33`，对应的容量为`2048`。

#### 4.3.1 扩容

当对容量为`2048`的ByteBuffer进行扩容时，根据当前的容量索引`index = 33` **加上** 扩容步长`INDEX_INCREMENT = 4`计算出扩容后的容量索引为`37`，那么扩缩容索引表`SIZE_TABLE`下标`37`对应的容量就是本次ByteBuffer扩容后的容量`SIZE_TABLE[37] = 32768`

#### 4.3.1  缩容

同理对容量为`2048`的ByteBuffer进行缩容时，我们就需要用当前容量索引`index = 33` **减去** 缩容步长`INDEX_DECREMENT = 1`计算出缩容后的容量索引`32`，那么扩缩容索引表`SIZE_TABLE`下标`32`对应的容量就是本次ByteBuffer缩容后的容量`SIZE_TABLE[32] = 1024`

### 4.4 扩缩容时机

`public abstract class AbstractNioByteChannel extends AbstractNioChannel {           @Override           public final void read() {               .........省略......               try {                   do {                         .........省略......                   } while (allocHandle.continueReading());                      //根据本次read loop总共读取的字节数，决定下次是否扩容或者缩容                   allocHandle.readComplete();                      .........省略.........                  } catch (Throwable t) {                   ...............省略...............               } finally {                  ...............省略...............               }           }   }   `

在每轮read loop结束之后，我们都会调用`allocHandle.readComplete()`来根据在allocHandle中统计的在本轮read loop中读取字节总大小，来决定在下一轮read loop中是否对DirectByteBuffer进行扩容或者缩容。

`public abstract class MaxMessageHandle implements ExtendedHandle {             @Override          public void readComplete() {                   //是否对recvbuf进行扩容缩容                   record(totalBytesRead());          }             private void record(int actualReadBytes) {               if (actualReadBytes <= SIZE_TABLE[max(0, index - INDEX_DECREMENT)]) {                   if (decreaseNow) {                       index = max(index - INDEX_DECREMENT, minIndex);                       nextReceiveBufferSize = SIZE_TABLE[index];                       decreaseNow = false;                   } else {                       decreaseNow = true;                   }               } else if (actualReadBytes >= nextReceiveBufferSize) {                   index = min(index + INDEX_INCREMENT, maxIndex);                   nextReceiveBufferSize = SIZE_TABLE[index];                   decreaseNow = false;               }           }           }   `

我们以当前ByteBuffer容量为`2048`，容量索引`index = 33`为例，对`allocHandle`的扩容缩容规则进行说明。

> 扩容步长`INDEX_INCREMENT = 4`，缩容步长`INDEX_DECREMENT = 1`。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

image.png

#### 4.4.1 缩容

- 如果本次`OP_READ事件`实际读取到的总字节数`actualReadBytes`在SIZE_TABLE\[index - INDEX_DECREMENT\]与SIZE_TABLE\[index\]之间的话，也就是如果本轮read loop结束之后总共读取的字节数在`[1024,2048]`之间。说明此时分配的`ByteBuffer`容量正好，不需要进行缩容也不需要进行扩容。比如本次`actualReadBytes = 2000`，正好处在`1024`与`2048`之间。说明`2048`的容量正好。

- 如果`actualReadBytes` 小于等于 SIZE_TABLE\[index - INDEX_DECREMENT\]，也就是如果本轮read loop结束之后总共读取的字节数小于等于`1024`。表示本次读取到的字节数比当前ByteBuffer容量的下一级容量还要小，说明当前ByteBuffer的容量分配的有些大了，设置缩容标识`decreaseNow = true`。当下次`OP_READ事件`继续满足缩容条件的时候，开始真正的进行缩容。缩容后的容量为SIZE_TABLE\[index - INDEX_DECREMENT\]，但不能小于SIZE_TABLE\[minIndex\]。

> 注意需要满足两次缩容条件才会进行缩容，且缩容步长为1，缩容比较谨慎

#### 4.4.2 扩容

如果本次`OP_READ事件`处理总共读取的字节数`actualReadBytes` 大于等于 当前ByteBuffer容量(nextReceiveBufferSize)时，说明ByteBuffer分配的容量有点小了，需要进行扩容。扩容后的容量为SIZE_TABLE\[index + INDEX_INCREMENT\]，但不能超过SIZE_TABLE\[maxIndex\]。

> 满足一次扩容条件就进行扩容，并且扩容步长为4， 扩容比较奔放

### 4.5 AdaptiveRecvByteBufAllocator类的实例化

AdaptiveRecvByteBufAllocator类的实例化主要是确定ByteBuffer的初始容量，以及最小容量和最大容量在扩缩容索引表`SIZE_TABLE`中的下标：`minIndex`和`maxIndex`。

AdaptiveRecvByteBufAllocator定义了三个关于ByteBuffer容量的字段：

- `DEFAULT_MINIMUM` ：表示ByteBuffer最小的容量，默认为`64`，也就是无论ByteBuffer在怎么缩容，容量也不会低于`64`。

- `DEFAULT_INITIAL`：表示ByteBuffer的初始化容量。默认为`2048`。

- `DEFAULT_MAXIMUM` ：表示ByteBuffer的最大容量，默认为`65536`，也就是无论ByteBuffer在怎么扩容，容量也不会超过`65536`。

`public class AdaptiveRecvByteBufAllocator extends DefaultMaxMessagesRecvByteBufAllocator {          static final int DEFAULT_MINIMUM = 64;       static final int DEFAULT_INITIAL = 2048;       static final int DEFAULT_MAXIMUM = 65536;          public AdaptiveRecvByteBufAllocator() {           this(DEFAULT_MINIMUM, DEFAULT_INITIAL, DEFAULT_MAXIMUM);       }          public AdaptiveRecvByteBufAllocator(int minimum, int initial, int maximum) {                      .................省略异常检查逻辑.............              //计算minIndex maxIndex           //在SIZE_TABLE中二分查找最小 >= minimum的容量索引 ：3           int minIndex = getSizeTableIndex(minimum);           if (SIZE_TABLE[minIndex] < minimum) {               this.minIndex = minIndex + 1;           } else {               this.minIndex = minIndex;           }              //在SIZE_TABLE中二分查找最大 <= maximum的容量索引 ：38           int maxIndex = getSizeTableIndex(maximum);           if (SIZE_TABLE[maxIndex] > maximum) {               this.maxIndex = maxIndex - 1;           } else {               this.maxIndex = maxIndex;           }              this.initial = initial;       }   }   `

接下来的事情就是确定最小容量DEFAULT_MINIMUM 在SIZE_TABLE中的下标`minIndex`，以及最大容量DEFAULT_MAXIMUM 在SIZE_TABLE中的下标`maxIndex`。

从AdaptiveRecvByteBufAllocator类初始化的过程中，我们可以看出SIZE_TABLE中存储的数据特征是一个有序的集合。

我们可以通过**二分查找**在SIZE_TABLE中找出`第一个`容量大于等于DEFAULT_MINIMUM的容量索引`minIndex`。

同理通过**二分查找**在SIZE_TABLE中找出`最后一个`容量小于等于DEFAULT_MAXIMUM的容量索引`maxIndex`。

根据上一小节关于`SIZE_TABLE`中容量数据分布的截图，我们可以看出`minIndex = 3`，`maxIndex = 38`

#### 4.5.1 二分查找容量索引下标

`private static int getSizeTableIndex(final int size) {           for (int low = 0, high = SIZE_TABLE.length - 1;;) {               if (high < low) {                   return low;               }               if (high == low) {                   return high;               }                  int mid = low + high >>> 1;//无符号右移，高位始终补0               int a = SIZE_TABLE[mid];               int b = SIZE_TABLE[mid + 1];               if (size > b) {                   low = mid + 1;               } else if (size < a) {                   high = mid - 1;               } else if (size == a) {                   return mid;               } else {                   return mid + 1;               }           }       }`

> 经常刷LeetCode的小伙伴肯定一眼就看出这个是**二分查找的模板**了。

它的目的就是根据给定容量，在扩缩容索引表`SIZE_TABLE`中，通过**二分查找**找到`最贴近`给定size的容量的索引下标（第一个大于等于 size的容量）

### 4.6 RecvByteBufAllocator.Handle

前边我们提到最终动态调整ByteBuffer容量的是由AdaptiveRecvByteBufAllocator中的`Handler`负责的，我们来看下这个`allocHandle`的创建过程。

`protected abstract class AbstractUnsafe implements Unsafe {              private RecvByteBufAllocator.Handle recvHandle;              @Override           public RecvByteBufAllocator.Handle recvBufAllocHandle() {               if (recvHandle == null) {                   recvHandle = config().getRecvByteBufAllocator().newHandle();               }               return recvHandle;           }      }   `

从allocHandle的获取过程我们看到最allocHandle的创建是由`AdaptiveRecvByteBufAllocator#newHandle`方法执行的。

`public class AdaptiveRecvByteBufAllocator extends DefaultMaxMessagesRecvByteBufAllocator {          @Override       public Handle newHandle() {           return new HandleImpl(minIndex, maxIndex, initial);       }          private final class HandleImpl extends MaxMessageHandle {           //最小容量在扩缩容索引表中的index           private final int minIndex;           //最大容量在扩缩容索引表中的index           private final int maxIndex;           //当前容量在扩缩容索引表中的index 初始33 对应容量2048           private int index;           //预计下一次分配buffer的容量，初始：2048           private int nextReceiveBufferSize;           //是否缩容           private boolean decreaseNow;              HandleImpl(int minIndex, int maxIndex, int initial) {               this.minIndex = minIndex;               this.maxIndex = maxIndex;                  //在扩缩容索引表中二分查找到最小大于等于initial 的容量               index = getSizeTableIndex(initial);               //2048               nextReceiveBufferSize = SIZE_TABLE[index];           }              .......................省略...................       }      }   `

这里我们看到Netty中用于动态调整ByteBuffer容量的`allocHandle`的实际类型为`MaxMessageHandle`。

下面我们来介绍下`HandleImpl`中的核心字段，它们都和ByteBuffer的容量有关：

- `minIndex` ：最小容量在扩缩容索引表`SIZE_TABE`中的index。默认是`3`。

- `maxIndex` ：最大容量在扩缩容索引表`SIZE_TABE`中的index。默认是`38`。

- `index` ：当前容量在扩缩容索引表`SIZE_TABE`中的index。初始是`33`。

- `nextReceiveBufferSize` ：预计下一次分配buffer的容量，初始为`2048`。在每次申请内存分配ByteBuffer的时候，采用`nextReceiveBufferSize`的值指定容量。

- `decreaseNow ：` 是否需要进行缩容。

## 5. 使用堆外内存为ByteBuffer分配内存

AdaptiveRecvByteBufAllocator类只是负责动态调整ByteBuffer的容量，而具体为ByteBuffer申请内存空间的是由`PooledByteBufAllocator`负责。

### 5.1 类名前缀Pooled的来历

在我们使用Java进行日常开发过程中，在为对象分配内存空间的时候我们都会选择在JVM堆中为对象分配内存，这样做对我们Java开发者特别的友好，我们只管使用就好而不必过多关心这块申请的内存如何回收，因为JVM堆完全受Java虚拟机控制管理，Java虚拟机会帮助我们回收不再使用的内存。

但是JVM在进行垃圾回收时候的`stop the world`会对我们应用程序的性能造成一定的影响。

除此之外我们在[?《聊聊Netty那些事儿之从内核角度看IO模型》](https://mp.weixin.qq.com/s?__biz=Mzg2MzU3Mjc3Ng==&mid=2247483737&idx=1&sn=7ef3afbb54289c6e839eed724bb8a9d6&chksm=ce77c71ef9004e08e3d164561e3a2708fc210c05408fa41f7fe338d8e85f39c1ad57519b614e&scene=21#wechat_redirect)一文中介绍IO模型的时候提到，当数据达到网卡时，网卡会通过DMA的方式将数据拷贝到内核空间中，这是`第一次拷贝`。当用户线程在用户空间发起系统IO调用时，CPU会将内核空间的数据再次拷贝到用户空间。这是`第二次拷贝`。

于此不同的是当我们在JVM中发起IO调用时，比如我们使用JVM堆内存读取`Socket接收缓冲区`中的数据时，**会多一次内存拷贝**，CPU在`第二次拷贝`中将数据从内核空间拷贝到用户空间时，此时的用户空间站在JVM角度是`堆外内存`，所以还需要将堆外内存中的数据拷贝到`堆内内存`中。这就是`第三次内存拷贝`。

同理当我们在JVM中发起IO调用向`Socket发送缓冲区`写入数据时，JVM会将IO数据先`拷贝`到`堆外内存`，然后才能发起系统IO调用。

**那为什么操作系统不直接使用JVM的`堆内内存`进行`IO操作`呢？**

因为JVM的内存布局和操作系统分配的内存是不一样的，操作系统不可能按照JVM规范来读写数据，所以就需要`第三次拷贝`中间做个转换将堆外内存中的数据拷贝到JVM堆中。

______________________________________________________________________

所以基于上述内容，在使用JVM堆内内存时会产生以下两点性能影响：

1. JVM在垃圾回收堆内内存时，会发生`stop the world`导致应用程序卡顿。

1. 在进行IO操作的时候，会多产生一次由堆外内存到堆内内存的拷贝。

**基于以上两点使用`JVM堆内内存`对性能造成的影响**，于是对性能有卓越追求的Netty采用`堆外内存`也就是`DirectBuffer`来为ByteBuffer分配内存空间。

采用堆外内存为ByteBuffer分配内存的好处就是：

- 堆外内存直接受操作系统的管理，不会受JVM的管理，所以JVM垃圾回收对应用程序的性能影响就没有了。

- 网络数据到达之后直接在`堆外内存`上接收，进程读取网络数据时直接在堆外内存中读取，所以就避免了`第三次内存拷贝`。

所以Netty在进行 I/O 操作时都是使用的堆外内存，可以避免数据从 JVM 堆内存到堆外内存的拷贝。但是由于堆外内存不受JVM的管理，所以就需要额外关注对内存的使用和释放，稍有不慎就会造成内存泄露，于是Netty就引入了**内存池**对`堆外内存`进行统一管理。

**PooledByteBufAllocator类的这个前缀`Pooled`就是`内存池`的意思，这个类会使用Netty的内存池为ByteBuffer分配`堆外内存`。**

### 5.2 PooledByteBufAllocator的创建

#### 创建时机

在服务端NioServerSocketChannel的配置类NioServerSocketChannelConfig以及客户端NioSocketChannel的配置类NioSocketChannelConfig**实例化的时候会触发**PooledByteBufAllocator的创建。

`public class DefaultChannelConfig implements ChannelConfig {       //PooledByteBufAllocator       private volatile ByteBufAllocator allocator = ByteBufAllocator.DEFAULT;          ..........省略......   }   `

创建出来的PooledByteBufAllocator实例保存在`DefaultChannelConfig类`中的`ByteBufAllocator allocator`字段中。

#### 创建过程

`public interface ByteBufAllocator {          ByteBufAllocator DEFAULT = ByteBufUtil.DEFAULT_ALLOCATOR;              ..................省略............   }   `

`public final class ByteBufUtil {          static final ByteBufAllocator DEFAULT_ALLOCATOR;          static {           String allocType = SystemPropertyUtil.get(                   "io.netty.allocator.type", PlatformDependent.isAndroid() ? "unpooled" : "pooled");           allocType = allocType.toLowerCase(Locale.US).trim();              ByteBufAllocator alloc;           if ("unpooled".equals(allocType)) {               alloc = UnpooledByteBufAllocator.DEFAULT;               logger.debug("-Dio.netty.allocator.type: {}", allocType);           } else if ("pooled".equals(allocType)) {               alloc = PooledByteBufAllocator.DEFAULT;               logger.debug("-Dio.netty.allocator.type: {}", allocType);           } else {               alloc = PooledByteBufAllocator.DEFAULT;               logger.debug("-Dio.netty.allocator.type: pooled (unknown: {})", allocType);           }              DEFAULT_ALLOCATOR = alloc;                      ...................省略..................       }   }   `

从ByteBufUtil类的初始化过程我们可以看出，在为ByteBuffer分配内存的时候是否使用内存池在Netty中是可以配置的。

- 通过系统变量`-D io.netty.allocator.type` 可以配置是否使用内存池为ByteBuffer分配内存。默认情况下是需要使用内存池的。但是在安卓系统中默认是不使用内存池的。

- 通过`PooledByteBufAllocator.DEFAULT`获取**内存池ByteBuffer分配器**。

`public static final PooledByteBufAllocator DEFAULT =               new PooledByteBufAllocator(PlatformDependent.directBufferPreferred());`

> 由于本文的主线是介绍Sub Reactor处理`OP_READ事件`的完整过程，所以这里只介绍主线相关的内容，这里只是简单介绍下在接收数据的时候为什么会用`PooledByteBufAllocator`来为`ByteBuffer`分配内存。而内存池的架构设计比较复杂，所以笔者后面会单独写一篇关于Netty内存管理的文章。

______________________________________________________________________

## 总结

本文介绍了Sub Reactor线程在处理OP_READ事件的整个过程。并深入剖析了AdaptiveRecvByteBufAllocator类动态调整ByteBuffer容量的原理。

同时也介绍了Netty为什么会使用堆外内存来为ByteBuffer分配内存，并由此引出了Netty的内存池分配器PooledByteBufAllocator 。

在介绍AdaptiveRecvByteBufAllocator类和PooledByteBufAllocator一起组合实现动态地为ByteBuffer分配容量的时候，笔者不禁想起了多年前看过的《Effective Java》中第16条 `复合优先于继承`。

Netty在这里也遵循了这条军规，首先两个类设计的都是单一的功能。

- AdaptiveRecvByteBufAllocator类只负责动态的调整ByteBuffer容量，并不管具体的内存分配。

- PooledByteBufAllocator类负责具体的内存分配，用内存池的方式。

这样设计的就比较灵活，具体内存分配的工作交给具体的`ByteBufAllocator`,可以使用内存池的分配方式`PooledByteBufAllocator`，也可以不使用内存池的分配方式`UnpooledByteBufAllocator`。具体的内存可以采用JVM堆内内存（HeapBuffer），也可以使用堆外内存（DirectBuffer）。

而`AdaptiveRecvByteBufAllocator`只需要关注调整它们的容量工作就可以了，而并不需要关注它们具体的内存分配方式。

最后通过`io.netty.channel.RecvByteBufAllocator.Handle#allocate`方法灵活组合不同的内存分配方式。这也是`装饰模式`的一种应用。

`byteBuf = allocHandle.allocate(allocator);   `

好了，今天的内容就到这里，我们下篇文章见~~~~

![](http://mmbiz.qpic.cn/mmbiz_png/sOIZXFW0vUZ9qpdibKBrYASLXXicypdMcnlrAGcicnHQyWYNZvic3C5OpgEicMDGsAcibZTKiaNECcNXDKJiaIBr2XGTow/300?wx_fmt=png&wxfrom=19)

**bin的技术小屋**

专注源码解析系列原创技术文章，分享自己的技术感悟。谈笑有鸿儒，往来无白丁。无丝竹之乱耳，无案牍之劳形。斯是陋室，惟吾德馨。

37篇原创内容

公众号

![](https://mmbiz.qlogo.cn/mmbiz_jpg/ZgMuHNwbpX4TOrXq2bEVVOPfGjaVfrOv7P8iaZC3GicBPGsLjSzYOthibcnonl9YShwvMsgrPL5JLvs6nfqCRW6EA/0?wx_fmt=jpeg)

bin的技术小屋

让本该造火箭的我们，不再拧螺丝

![赞赏二维码](https://mp.weixin.qq.com/s?__biz=Mzg2MzU3Mjc3Ng==&mid=2247484244&idx=1&sn=831060fc38caa201d69f87305de7f86a&chksm=ce77c513f9004c05b48f849ff99997d6d7252453135ae856a029137b88aa70b8e046013d596e&mpshare=1&scene=24&srcid=0223fnbVJQeWDObJ1nkKeOST&sharer_sharetime=1645582886730&sharer_shareid=5fb9813bfe9ffc983435bfc8d8c5e9ca&key=daf9bdc5abc4e8d088ae246169e24e25cad490577f8cd549e8dc1e3d2efcd2cecb6340f938469a35df791d7ade3cb5f85121557eb506d54bd71f7fda8238952d15c51cc4397965238aa94d7ec2a2c943a347eebe0c7509bbee54f35d1c65ea39face2a09bc94db48172c0dac1b98a8b70a6af9cbaef75b1791b51649de4ec2f0&ascene=0&uin=MTEwNTU1MjgwMw%3D%3D&devicetype=Windows+11+x64&version=63090b19&lang=zh_CN&countrycode=CN&exportkey=n_ChQIAhIQELJK7wsbDSibsbbMs2%2FFHhLVAQIE97dBBAEAAAAAAF8fOMKL6RAAAAAOpnltbLcz9gKNyK89dVj0cdilluaohPIxhgfoP9atKiM1NVWVaPOtugzosFLYkg1XXecGBPIDkcm2SmwngXe%2B31L%2F2j0MbzrUFrATC0c4JUm4pEqt8ubaGJaGekk8xdHCA9PE467CgovpvPyo2n3EMSvMiB3Jyu5Mj37rhhjO4XYobx%2BK2D8c%2FvN3Vu%2FgR%2B5WJxwnsLx0BmHEHnaNUZLQOvxLhHKhaqv9HgiDtraXPuc%2Bikh51YcEdAYtLrgLWA%3D%3D&acctmode=0&pass_ticket=OE4jBHnpyk%2Briyw2cZA5wdnp6bQYEscfMUEg1olVVY8BHALt2Abm2ao92h%2BlIoQm&wx_header=1&fasttmpl_type=0&fasttmpl_fullversion=7351805-zh_CN-zip&fasttmpl_flag=1)喜欢作者

2人喜欢

![](http://wx.qlogo.cn/mmopen/xnzE5DgUtTNSr3icicib7ecNvpw3X2S1UWnVqq4fC3cvSgHyG8LcLjf3m7VTckEd7ttQqhib0lG1Zk5bicR2R3tHe729FdaU2VMqiaKXtO2VjVYPobia9icgequVcQ2dKiaG06xeY/64)![](http://wx.qlogo.cn/mmopen/Oy6yxt2FicVvZNshITxXhkPgEfuz8Ahb5br8ibvZaQriaDEkldvbj6oSht2JD3jyH3khVMqbatuvgN32pDVAM4sM1qVJvMVj2ue/64)

聊聊netty那些事儿15

聊聊netty那些事儿 · 目录

上一篇抓到Netty一个Bug，顺带来透彻地聊一下Netty是如何高效接收网络连接的下一篇重磅硬核 | 一文聊透对象在JVM中的内存布局，以及内存对齐和压缩指针的原理及应用

阅读 1772

​
