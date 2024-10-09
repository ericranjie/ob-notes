悟空聊架构

_2021年12月07日 18:20_

以下文章来源于一枝花算不算浪漫 ，作者小飞 & 花哥

\[

![](http://wx.qlogo.cn/mmhead/Q3auHgzwzM6GibzlDRQbzDScog5MYxiblMdd56V8gOlF2F3iaDUuHib6CA/0)

**一枝花算不算浪漫**.

放弃不难，但坚持一定很酷。记录一个普通Javaer成长之路：http://www.cnblogs.com/wang-meng

\](https://mp.weixin.qq.com/s?\_\_biz=MzAwMjI0ODk0NA==&mid=2451961617&idx=2&sn=575990a340da00461f75cd4df2765b64&chksm=8d1c0e8eba6b87988cc361a0557185990df7816d787a7c26e948dc6035f46a01c6e3da3f4f02&mpshare=1&scene=24&srcid=1207PRgy5OZmO52zfMYQmXjP&sharer_sharetime=1638890272341&sharer_shareid=5fb9813bfe9ffc983435bfc8d8c5e9ca&key=daf9bdc5abc4e8d028b5423f80ba0ba5812f1ee22f9bc40559f1b1c70686f408fe7ce654abc85653a7017e0ff276647757c966c685b0449196557767def7413923104d6486b7cc20cf4a6677963a0262b3e25282fb1b1cb034e9269fbdce017b3197e185344a1f2434bcda6dd729eaaed56dda67d261ee29c5c12234dcdea573&ascene=0&uin=MTEwNTU1MjgwMw%3D%3D&devicetype=Windows+11+x64&version=63090b19&lang=zh_CN&countrycode=CN&exportkey=n_ChQIAhIQfcsM0MQqeZFyPKcb9wMUcRLmAQIE97dBBAEAAAAAAE2TOHdJOrgAAAAOpnltbLcz9gKNyK89dVj0qn3H7p%2BpQGwyxHCYKSECZDAWTouo9vuzRVbD1KMvClbKeCAj3EsCv3EtxEG9NaHo6UIk3fagH5KK7gg%2FOFDv%2BfXsX3P8c09OgvOKc2pQWDr0LGoe0j54%2Bs3pDJjrfEkQxyP%2FE5WmxtKYf59YFvppOMEL0YaIIioJ5hxPTQlJoZD06FgB7VpY1LOj52FJr7K%2BRUespSU1dGq8TwSiEKuJpJlwafzSIbaSeDCbTNnB0b5pJzYaQyN0AuD6pyOM5ovJ&acctmode=0&pass_ticket=UgBJWls70RGz94nqyrM0lrkpxgAWKbJDfjbOI9yQ3ZlKB9KVDojBFK4Xzl%2FiuER5&wx_header=1&fasttmpl_type=0&fasttmpl_fullversion=7351805-zh_CN-zip&fasttmpl_flag=1#)

大家好，我是悟空呀~本篇给大家带来的是 Netty 核心知识。建议收藏起来慢慢看~

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/SfAHMuUxqJ2uDuAHD45Yryn9ehtufwIxuJVI7k4Y3YwxDnu8ptK0teEopqyfUc7NibgRoEcuZAw6E4Tln2RH0ag/640?wx_fmt=jpeg&wxfrom=13&tp=wxpic)

NIO思维导图.png

![](http://mmbiz.qpic.cn/mmbiz_png/SfAHMuUxqJ2JicQpKzblLHz64qoibVa3ATNA4rH8mIYXAF3OErAzxFKHzf5qiaiblb4rAMuAXXMJHEcKcvaHv4ia9rA/300?wx_fmt=png&wxfrom=19)

**悟空聊架构**

用故事讲解分布式、架构。 《 JVM 性能调优实战》专栏作者， 《Spring Cloud 实战 PassJava》开源作者， 自主开发了 PMP 刷题小程序。

205篇原创内容

公众号

## 一、网络编程基础回顾

### 1. Socket

`Socket`本身有“插座”的意思，不是Java中特有的概念，而是一个语言无关的标准，任何可以实现网络编程的编程语言都有`Socket`。在`Linux`环境下，用于表示进程间网络通信的特殊文件类型，其本质为内核借助缓冲区形成的伪文件。既然是文件，那么理所当然的，我们可以使用文件描述符引用套接字。

与管道类似的，`Linux`系统将其封装成文件的目的是为了统一接口，使得读写套接字和读写文件的操作一致。区别是管道主要应用于本地进程间通信，而套接字多应用于网络进程间数据的传递。

可以这么理解：`Socket`就是网络上的两个应用程序通过一个双向通信连接实现数据交换的编程接口API。

`Socket`通信的基本流程具体步骤如下所示:

（1）服务端通过`Listen`开启监听，等待客户端接入。

（2）客户端的套接字通过`Connect`连接服务器端的套接字，服务端通过`Accept`接收客户端连接。在`connect-accept`过程中，操作系统将会进行三次握手。

（3）客户端和服务端通过`write`和`read`发送和接收数据，操作系统将会完成`TCP`数据的确认、重发等步骤。

（4）通过`close`关闭连接，操作系统会进行四次挥手。

针对Java编程语言，`java.net`包是网络编程的基础类库。其中`ServerSocket`和`Socket`是网络编程的基础类型。

`SeverSocket`是服务端应用类型。`Socket`是建立连接的类型。当连接建立成功后，服务器和客户端都会有一个`Socket`对象示例，可以通过这个`Socket`对象示例，完成会话的所有操作。对于一个完整的网络连接来说，`Socket`是平等的，没有服务器客户端分级情况。

### 2. IO模型介绍

对于一次IO操作，数据会先拷贝到内核空间中，然后再从内核空间拷贝到用户空间中，所以一次`read`操作，会经历两个阶段：

（1）等待数据准备

（2）数据从内核空间拷贝到用户空间

基于以上两个阶段就产生了五种不同的IO模式。

1. 阻塞IO：从进程发起IO操作，一直等待上述两个阶段完成，此时两阶段一起阻塞。

1. 非阻塞IO：进程一直询问IO准备好了没有，准备好了再发起读取操作，这时才把数据从内核空间拷贝到用户空间。第一阶段不阻塞但要轮询，第二阶段阻塞。

1. 多路复用IO：多个连接使用同一个select去询问IO准备好了没有，如果有准备好了的，就返回有数据准备好了，然后对应的连接再发起读取操作，把数据从内核空间拷贝到用户空间。两阶段分开阻塞。

1. 信号驱动IO：进程发起读取操作会立即返回，当数据准备好了会以通知的形式告诉进程，进程再发起读取操作，把数据从内核空间拷贝到用户空间。第一阶段不阻塞，第二阶段阻塞。

1. 异步IO：进程发起读取操作会立即返回，等到数据准备好且已经拷贝到用户空间了再通知进程拿数据。两个阶段都不阻塞。

这五种IO模式不难发现存在这两对关系：同步和异步、阻塞和非阻塞。那么稍微解释一下：

#### 同步和异步

- **同步：** 同步就是发起一个调用后，被调用者未处理完请求之前，调用不返回。

- **异步：** 异步就是发起一个调用后，立刻得到被调用者的回应表示已接收到请求，但是被调用者并没有返回结果，此时我们可以处理其他的请求，被调用者通常依靠事件，回调等机制来通知调用者其返回结果。

同步和异步的区别最大在于异步的话调用者不需要等待处理结果，被调用者会通过回调等机制来通知调用者其返回结果。

#### 阻塞和非阻塞

- **阻塞：** 阻塞就是发起一个请求，调用者一直等待请求结果返回，也就是当前线程会被挂起，无法从事其他任务，只有当条件就绪才能继续。

- **非阻塞：** 非阻塞就是发起一个请求，调用者不用一直等着结果返回，可以先去干其他事情。

阻塞和非阻塞是针对进程在访问数据的时候，根据IO操作的就绪状态来采取的不同方式，说白了是一种读取或者写入操作方法的实现方式，阻塞方式下读取或者写入函数将一直等待，而非阻塞方式下，读取或者写入方法会立即返回一个状态值。

如果组合后的同步阻塞(`blocking-IO`)简称`BIO`、同步非阻塞(`non-blocking-IO`)简称`NIO`和异步非阻塞(`asynchronous-non-blocking-IO`)简称`AIO`又代表什么意思呢？

- **BIO** (同步阻塞I/O模式): 数据的读取写入必须阻塞在一个线程内等待其完成。这里使用那个经典的烧开水例子，这里假设一个烧开水的场景，有一排水壶在烧开水，BIO的工作模式就是， 叫一个线程停留在一个水壶那，直到这个水壶烧开，才去处理下一个水壶。但是实际上线程在等待水壶烧开的时间段什么都没有做。

- **NIO**(同步非阻塞): 同时支持阻塞与非阻塞模式，但这里我们以其同步非阻塞I/O模式来说明，那么什么叫做同步非阻塞？如果还拿烧开水来说，NIO的做法是叫一个线程不断的轮询每个水壶的状态，看看是否有水壶的状态发生了改变，从而进行下一步的操作。

- **AIO**(异步非阻塞I/O模型): 异步非阻塞与同步非阻塞的区别在哪里？异步非阻塞无需一个线程去轮询所有IO操作的状态改变，在相应的状态改变后，系统会通知对应的线程来处理。对应到烧开水中就是，为每个水壶上面装了一个开关，水烧开之后，水壶会自动通知我水烧开了。

`java` 中的 `BIO`、`NIO`和`AIO`理解为是 `Java 语言`在操作系统层面对这三种 `IO` 模型的封装。程序员在使用这些 封装API 的时候，不需要关心操作系统层面的知识，也不需要根据不同操作系统编写不同的代码，只需要使用`Java`的API就可以了。由此，为了使读者对这三种模型有个比较具体和递推式的了解，并且和本文主题`NIO`有个清晰的对比，下面继续延伸。

#### Java BIO

`BIO`编程方式通常是是Java的上古产品，自JDK 1.0-JDK1.4就有的东西。编程实现过程为：首先在服务端启动一个`ServerSocket`来监听网络请求，客户端启动`Socket`发起网络请求，默认情况下`SeverSocket`会建立一个线程来处理此请求，如果服务端没有线程可用，客户端则会阻塞等待或遭到拒绝。服务器实现模式为一个连接一个线程，即客户端有连接请求时服务器端就需要启动一个线程进行处理。大致结构如下：

![图片](https://mmbiz.qpic.cn/mmbiz_png/TCh3WuejsNRsll4w4D6WuibNEViaU3cb7OuvtjlpN9ictEYaPGdiak9dLQiavh8CtFzMOoLHZ2rp6eicuiaRmeK7kZicYA/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

如果要让 `BIO` 通信模型能够同时处理多个客户端请求，就必须使用多线程（主要原因是 `socket.accept()`、`socket.read()`、 `socket.write()` 涉及的三个主要函数都是同步阻塞的），也就是说它在接收到客户端连接请求之后为每个客户端创建一个新的线程进行链路处理，处理完成之后，通过输出流返回应答给客户端，线程销毁。这就是典型的 一请求一应答通信模型 。我们可以设想一下如果这个连接不做任何事情的话就会造成不必要的线程开销，不过可以通过**线程池机制**改善，线程池还可以让线程的创建和回收成本相对较低。使用线程池机制改善后的 `BIO` 模型图如下:

![图片](https://mmbiz.qpic.cn/mmbiz_png/TCh3WuejsNRsll4w4D6WuibNEViaU3cb7OO5LndAzUVIkMicy2dzA2nZXTghicrRNjwNUoXNdBjAah6uuUFB2jicNHw/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

`BIO`方式适用于连接数目比较小且固定的架构，这种方式对服务器资源要求比较高，并发局限于应用中，是JDK1.4以前的唯一选择，但程序直观简单易懂。`Java BIO`编程示例网上很多，这里就不进行coding举例了，毕竟后面`NIO`才是重点。

#### Java NIO

`NIO`（New IO或者No-Blocking IO），从JDK1.4 开始引入的`非阻塞IO`，是一种`非阻塞`+ `同步`的通信模式。这里的`No Blocking IO`用于区分上面的`BIO`。

`NIO`本身想解决 `BIO`的并发问题，通过`Reactor模式`的事件驱动机制来达到`Non Blocking`的。当 `socket` 有流可读或可写入 `socket` 时，操作系统会相应的通知应用程序进行处理，应用再将流读取到缓冲区或写入操作系统。

也就是说，这个时候，已经不是一个连接就 要对应一个处理线程了，而是有效的请求，对应一个线程，当连接没有数据时，是没有工作线程来处理的。

当一个连接创建后，不需要对应一个线程，这个连接会被注册到 `多路复用器`上面，所以所有的连接只需要一个线程就可以搞定，当这个线程中的`多路复用器` 进行轮询的时候，发现连接上有请求的话，才开启一个线程进行处理，也就是一个请求一个线程模式。

`NIO`提供了与传统BIO模型中的`Socket`和`ServerSocket`相对应的`SocketChannel`和`ServerSocketChannel`两种不同的套接字通道实现，如下图结构所示。这里涉及的`Reactor`设计模式、多路复用`Selector`、`Buffer`等暂时不用管，后面会讲到。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

NIO 方式适用于连接数目多且连接比较短(轻操作)的架构，比如聊天服务器，并发局 限于应用中，编程复杂，JDK1.4 开始支持。同时，`NIO`和普通IO的区别主要可以从存储数据的载体、是否阻塞等来区分：

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

NIO和普通IO区别.png

#### Java AIO

与 `NIO` 不同，当进行读写操作时，只须直接调用 API 的 `read` 或 `write` 方法即可。这两种方法均为异步的，对于读操作而言，当有流可读取时，操作系统会将可读的流传入 `read` 方 法的缓冲区，并通知应用程序；对于写操作而言，当操作系统将 `write` 方法传递的流写入完毕时，操作系统主动通知应用程序。即可以理解为，`read/write` 方法都是异步的，完成后会主动调用回调函数。在 `JDK7` 中，提供了异步文件通道和异步套接字通道的实现，这部分内容被称作 `NIO`.

`AIO` 方式使用于连接数目多且连接比较长(重操作)的架构，比如相册服务器，充分调用 `OS` 参与并发操作，编程比较复杂，`JDK7` 开始支持。

目前来说 `AIO` 的应用还不是很广泛，`Netty` 之前也尝试使用过 `AIO`，不过又放弃了。

## 二、NIO核心组件介绍

### 1. Channel

在`NIO`中，基本所有的IO操作都是从`Channel`开始的，`Channel`通过`Buffer(缓冲区)`进行读写操作。

`read()`表示读取通道中数据到缓冲区，`write()`表示把缓冲区数据写入到通道。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

Channel和Buffer互相操作.png

`Channel`有好多实现类，这里有三个最常用：

- `SocketChannel`：一个客户端发起TCP连接的Channel

- `ServerSocketChannel`：一个服务端监听新连接的TCP Channel，对于每一个新的Client连接，都会建立一个对应的SocketChannel

- `FileChannel`：从文件中读写数据

其中`SocketChannel`和`ServerSocketChannel`是网络编程中最常用的，一会在最后的示例代码中会有讲解到具体用法。

### 2. Buffer

#### 概念

`Buffer`也被成为内存缓冲区，本质上就是内存中的一块，我们可以将数据写入这块内存，之后从这块内存中读取数据。也可以将这块内存封装成`NIO Buffer`对象，并提供一组常用的方法，方便我们对该块内存进行读写操作。

`Buffer`在`java.nio`中被定义为抽象类：

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

Buffer结构体系.png

我们可以将`Buffer`理解为一个数组的封装，我们最常用的`ByteBuffer`对应的数据结构就是`byte[]`

#### 属性

`Buffer`中有4个非常重要的属性：**capacity、limit、position、mark**

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

Buffer中基本属性.png

- `capacity`属性：容量，Buffer能够容纳的数据元素的最大值，在Buffer初始化创建的时候被赋值，而且不能被修改。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

capacity.png

上图中，初始化Buffer的容量为8（图中从0~7，共8个元素），所以**capacity = 8**

- `limit`属性：代表Buffer可读可写的上限。

- 写模式下：`limit` 代表能写入数据的上限位置，这个时候`limit = capacity`读模式下：在`Buffer`完成所有数据写入后，通过调用`flip()`方法，切换到读模式，此时`limit`等于`Buffer`中实际已经写入的数据大小。因为`Buffer`可能没有被写满，所以**limit\<=capacity**

- `position`属性：代表读取或者写入`Buffer`的位置。默认为0。

- 写模式下：每往`Buffer`中写入一个值，`position`就会自动加1，代表下一次写入的位置。

- 读模式下：每往`Buffer`中读取一个值，`position`就自动加1，代表下一次读取的位置。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

NIO属性概念.png

从上图就能很清晰看出，读写模式下**capacity、limit、position**的关系了。

- `mark`属性：代表标记，通过mark()方法，记录当前position值，将position值赋值给mark，在后续的写入或读取过程中，可以通过reset()方法恢复当前position为mark记录的值。

这几个重要属性讲完，我们可以再来回顾下：

> 0 \<= mark \<= position \<= limit \<= capacity

现在应该很清晰这几个属性的关系了~

#### Buffer常见操作

##### 创建Buffer

- `allocate(int capacity)`

`ByteBuffer buffer = ByteBuffer.allocate(1024);   int count = channel.read(buffer);   `

例子中创建的`ByteBuffer`是基于堆内存的一个对象。

- `wrap(array)`

`wrap`方法可以将数组包装成一个`Buffer`对象:

`ByteBuffer buffer = ByteBuffer.wrap("hello world".getBytes());   channel.write(buffer);   `

- `allocateDirect(int capacity)`

通过`allocateDirect`方法也可以快速实例化一个`Buffer`对象，和`allocate`很相似，这里区别的是`allocateDirect`创建的是基于**堆外内存**的对象。

堆外内存不在JVM堆上，不受GC的管理。堆外内存进行一些底层系统的IO操作时，效率会更高。

##### Buffer写操作

`Buffer`写入可以通过`put()`和`channel.read(buffer)`两种方式写入。

通常我们NIO的读操作的时候，都是从`Channel`中读取数据写入`Buffer`，这个对应的是`Buffer`的**写操作**。

##### Buffer读操作

`Buffer`读取可以通过`get()`和`channel.write(buffer)`两种方式读入。

还是同上，我们对`Buffer`的读入操作，反过来说就是对`Channel`的**写操作**。读取`Buffer`中的数据然后写入`Channel`中。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

Channel和Buffer互相操作.png

##### 其他常见方法

- `rewind()`：重置position位置为0，可以重新读取和写入buffer，一般该方法适用于读操作，可以理解为对buffer的重复读。

`public final Buffer rewind() {       position = 0;       mark = -1;       return this;   }   `

- `flip()`：很常用的一个方法，一般在写模式切换到读模式的时候会经常用到。也会将position设置为0，然后设置limit等于原来写入的position。

`public final Buffer flip() {       limit = position;       position = 0;       mark = -1;       return this;   }   `

- `clear()`：重置buffer中的数据，该方法主要是针对于写模式，因为limit设置为了capacity，读模式下会出问题。

`public final Buffer clear() {       position = 0;       limit = capacity;       mark = -1;       return this;   }   `

- `mark()&reset()`: `mark()`方法是保存当前`position`到变量`mark`z中，然后通过`reset()`方法恢复当前`position`为`mark`，实现代码很简单，如下：

`public final Buffer mark() {       mark = position;       return this;   }      public final Buffer reset() {       int m = mark;       if (m < 0)           throw new InvalidMarkException();       position = m;       return this;   }   `

常用的读写方法可以用一张图总结一下：

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

Buffer读写操作.png

### 3. Selector

#### 概念

`Selector`是NIO中最为重要的组件之一，我们常常说的`多路复用器`就是指的`Selector`组件。`Selector`组件用于轮询一个或多个`NIO Channel`的状态是否处于可读、可写。通过轮询的机制就可以管理多个Channel，也就是说可以管理多个网络连接。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

Selector原理图.png

#### 轮询机制

1. 首先，需要将Channel注册到Selector上，这样Selector才知道需要管理哪些Channel

1. 接着Selector会不断轮询其上注册的Channel，如果某个Channel发生了读或写的时间，这个Channel就会被Selector轮询出来，然后通过SelectionKey可以获取就绪的Channel集合，进行后续的IO操作。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

轮询机制.png

#### 属性操作

1. 创建Selector

通过`open()`方法，我们可以创建一个`Selector`对象。

`Selector selector = Selector.open();   `

2. 注册Channel到Selector中

我们需要将`Channel`注册到`Selector`中，才能够被`Selector`管理。

`channel.configureBlocking(false);   SelectionKey key = channel.register(selector, SelectionKey.OP_READ);   `

某个`Channel`要注册到`Selector`中，那么该Channel必须是**非阻塞**，所有上面代码中有个`configureBlocking()`的配置操作。

在`register(Selector selector, int interestSet)`方法的第二个参数，标识一个`interest`集合，意思是Selector对哪些事件感兴趣，可以监听四种不同类型的事件：

`public static final int OP_READ = 1 << 0;   public static final int OP_WRITE = 1 << ;   public static final int OP_CONNECT = 1 << 3;   public static final int OP_ACCEPT = 1 << 4;   `

- `Connect事件` ：连接完成事件( TCP 连接 )，仅适用于客户端，对应 SelectionKey.OP_CONNECT。

- `Accept事件` ：接受新连接事件，仅适用于服务端，对应 SelectionKey.OP_ACCEPT 。

- `Read事件` ：读事件，适用于两端，对应 SelectionKey.OP_READ ，表示 Buffer 可读。

- `Write事件` ：写时间，适用于两端，对应 SelectionKey.OP_WRITE ，表示 Buffer 可写。

`Channel`触发了一个事件，表明该时间已经准备就绪：

- 一个Client Channel成功连接到另一个服务器，成为“连接就绪”

- 一个Server Socket准备好接收新进入的接，称为“接收就绪”

- 一个有数据可读的Channel，称为“读就绪”

- 一个等待写数据的Channel，称为”写就绪“

当然，`Selector`是可以同时对多个事件感兴趣的，我们使用或运算即可组合多个事件：

`int interestSet = SelectionKey.OP_READ | SelectionKey.OP_WRITE;   `

#### Selector其他一些操作

##### 选择Channel

`public abstract int select() throws IOException;   public abstract int select(long timeout) throws IOException;   public abstract int selectNow() throws IOException;   `

当Selector执行`select()`方法就会产生阻塞，等到注册在其上的Channel准备就绪就会立即返回，返回准备就绪的数量。

`select(long timeout)`则是在`select()`的基础上增加了超时机制。`selectNow()`立即返回，不产生阻塞。

**有一点非常需要注意：** `select` 方法返回的 `int` 值，表示有多少 `Channel` 已经就绪。

自上次调用`select` 方法后有多少 `Channel` 变成就绪状态。如果调用 `select` 方法，因为有一个 `Channel` 变成就绪状态则返回了 1 ；

若再次调用 `select` 方法，如果另一个 `Channel` 就绪了，它会再次返回1。

##### 获取可操作的Channel

`Set selectedKeys = selector.selectedKeys();   `

当有新增就绪的`Channel`，调用`select()`方法，就会将key添加到Set集合中。

## 三、代码示例

前面铺垫了这么多，主要是想让大家能够看懂`NIO`代码示例，也方便后续大家来自己手写`NIO` 网络编程的程序。创建NIO服务端的主要步骤如下：

> `1. 打开ServerSocketChannel，监听客户端连接   2. 绑定监听端口，设置连接为非阻塞模式   3. 创建Reactor线程，创建多路复用器并启动线程   4. 将ServerSocketChannel注册到Reactor线程中的Selector上，监听ACCEPT事件   5. Selector轮询准备就绪的key   6. Selector监听到新的客户端接入，处理新的接入请求，完成TCP三次握手，建立物理链路   7. 设置客户端链路为非阻塞模式   8. 将新接入的客户端连接注册到Reactor线程的Selector上，监听读操作，读取客户端发送的网络消息   9. 异步读取客户端消息到缓冲区   10.对Buffer编解码，处理半包消息，将解码成功的消息封装成Task   11.将应答消息编码为Buffer，调用SocketChannel的write将消息异步发送给客户端   `

`NIOServer.java` ：

`public class NIOServer {             private static Selector selector;          public static void main(String[] args) {           init();           listen();       }          private static void init() {           ServerSocketChannel serverSocketChannel = null;              try {               selector = Selector.open();                  serverSocketChannel = ServerSocketChannel.open();               serverSocketChannel.configureBlocking(false);               serverSocketChannel.socket().bind(new InetSocketAddress(9000));               serverSocketChannel.register(selector, SelectionKey.OP_ACCEPT);               System.out.println("NioServer 启动完成");           } catch (IOException e) {               e.printStackTrace();           }       }          private static void listen() {           while (true) {               try {                   selector.select();                   Iterator<SelectionKey> keysIterator = selector.selectedKeys().iterator();                   while (keysIterator.hasNext()) {                       SelectionKey key = keysIterator.next();                       keysIterator.remove();                       handleRequest(key);                   }               } catch (Throwable t) {                   t.printStackTrace();               }           }       }          private static void handleRequest(SelectionKey key) throws IOException {           SocketChannel channel = null;           try {               if (key.isAcceptable()) {                   ServerSocketChannel serverSocketChannel = (ServerSocketChannel) key.channel();                   channel = serverSocketChannel.accept();                   channel.configureBlocking(false);                   System.out.println("接受新的 Channel");                   channel.register(selector, SelectionKey.OP_READ);               }                  if (key.isReadable()) {                   channel = (SocketChannel) key.channel();                   ByteBuffer buffer = ByteBuffer.allocate(1024);                   int count = channel.read(buffer);                   if (count > 0) {                       System.out.println("服务端接收请求：" + new String(buffer.array(), 0, count));                       channel.register(selector, SelectionKey.OP_WRITE);                   }               }                  if (key.isWritable()) {                   ByteBuffer buffer = ByteBuffer.allocate(1024);                   buffer.put("收到".getBytes());                   buffer.flip();                      channel = (SocketChannel) key.channel();                   channel.write(buffer);                   channel.register(selector, SelectionKey.OP_READ);               }           } catch (Throwable t) {               t.printStackTrace();               if (channel != null) {                   channel.close();               }           }       }   }   `

`NIOClient.java`：

`public class NIOClient {          public static void main(String[] args) {           new Worker().start();       }          static class Worker extends Thread {           @Override           public void run() {               SocketChannel channel = null;               Selector selector = null;               try {                   channel = SocketChannel.open();                   channel.configureBlocking(false);                      selector = Selector.open();                   channel.register(selector, SelectionKey.OP_CONNECT);                   channel.connect(new InetSocketAddress(9000));                   while (true) {                       selector.select();                       Iterator<SelectionKey> keysIterator = selector.selectedKeys().iterator();                       while (keysIterator.hasNext()) {                           SelectionKey key = keysIterator.next();                           keysIterator.remove();                              if (key.isConnectable()) {                               System.out.println();                               channel = (SocketChannel) key.channel();                                  if (channel.isConnectionPending()) {                                   channel.finishConnect();                                      ByteBuffer buffer = ByteBuffer.allocate(1024);                                   buffer.put("你好".getBytes());                                   buffer.flip();                                   channel.write(buffer);                               }                                  channel.register(selector, SelectionKey.OP_READ);                           }                              if (key.isReadable()) {                               channel = (SocketChannel) key.channel();                               ByteBuffer buffer = ByteBuffer.allocate(1024);                               int len = channel.read(buffer);                                  if (len > 0) {                                   System.out.println("[" + Thread.currentThread().getName()                                           + "]收到响应：" + new String(buffer.array(), 0, len));                                   Thread.sleep(5000);                                   channel.register(selector, SelectionKey.OP_WRITE);                               }                           }                              if(key.isWritable()) {                               ByteBuffer buffer = ByteBuffer.allocate(1024);                               buffer.put("你好".getBytes());                               buffer.flip();                                  channel = (SocketChannel) key.channel();                               channel.write(buffer);                               channel.register(selector, SelectionKey.OP_READ);                           }                       }                   }               } catch (Exception e) {                   e.printStackTrace();               } finally{                   if(channel != null){                       try {                           channel.close();                       } catch (IOException e) {                           e.printStackTrace();                       }                   }                      if(selector != null){                       try {                           selector.close();                       } catch (IOException e) {                           e.printStackTrace();                       }                   }               }           }       }   }   `

打印结果：

`// Server端   NioServer 启动完成   接受新的 Channel   服务端接收请求：你好   服务端接收请求：你好   服务端接收请求：你好      // Client端   [Thread-0]收到响应：收到   [Thread-0]收到响应：收到   [Thread-0]收到响应：收到   `

## 四、总结

回顾一下使用 `NIO` 开发服务端程序的步骤：

1. 创建 `ServerSocketChannel` 和业务处理线程池。

1. 绑定监听端口，并配置为非阻塞模式。

1. 创建 `Selector`，将之前创建的 `ServerSocketChannel` 注册到 `Selector` 上，监听 `SelectionKey.OP_ACCEPT`。

1. 循环执行 ``` Selector.select()`` 方法，轮询就绪的 ```Channel\`。

1. 轮询就绪的 `Channel` 时，如果是处于 `OP_ACCEPT` 状态，说明是新的客户端接入，调用 `ServerSocketChannel.accept` 接收新的客户端。

1. 设置新接入的 `SocketChannel` 为非阻塞模式，并注册到 `Selector` 上，监听 `OP_READ`。

1. 如果轮询的 `Channel` 状态是 `OP_READ`，说明有新的就绪数据包需要读取，则构造 `ByteBuffer` 对象，读取数据。

那从这些步骤中基本知道开发者需要熟悉的知识点有：

1. `jdk-nio`提供的几个关键类：`Selector` , `SocketChannel` , `ServerSocketChannel` , `FileChannel` ,`ByteBuffer` ,`SelectionKey`

1. 需要知道网络知识：tcp粘包拆包 、网络闪断、包体溢出及重复发送等

1. 需要知道`linux`底层实现，如何正确的关闭`channel`，如何退出注销`selector` ，如何避免`selector`太过于频繁

1. 需要知道如何让`client`端获得`server`端的返回值,然后才返回给前端，需要如何等待或在怎样作熔断机制

1. 需要知道对象序列化，及序列化算法

1. 省略等等，因为我已经有点不舒服了，作为程序员的我习惯了舒舒服服简单的API，不用太知道底层细节，就能写出比较健壮和没有Bug的代码...

**NIO 原生 API 的弊端 :**

**① NIO 组件复杂 :** 使用原生 `NIO` 开发服务器端与客户端 , 需要涉及到 服务器套接字通道 ( `ServerSocketChannel` ) , 套接字通道 ( `SocketChannel` ) , 选择器 ( `Selector` ) , 缓冲区 ( `ByteBuffer` ) 等组件 , 这些组件的原理 和API 都要熟悉 , 才能进行 `NIO` 的开发与调试 , 之后还需要针对应用进行调试优化

**② NIO 开发基础 :** `NIO` 门槛略高 , 需要开发者掌握多线程、网络编程等才能开发并且优化 `NIO` 网络通信的应用程序

**③ 原生 API 开发网络通信模块的基本的传输处理 :** 网络传输不光是实现服务器端和客户端的数据传输功能 , 还要处理各种异常情况 , 如 连接断开重连机制 , 网络堵塞处理 , 异常处理 , 粘包处理 , 拆包处理 , 缓存机制 等方面的问题 , 这是所有成熟的网络应用程序都要具有的功能 , 否则只能说是入门级的 Demo

**④ NIO BUG :** `NIO` 本身存在一些 BUG , 如 `Epoll` , 导致 选择器 ( `Selector` ) 空轮询 , 在 JDK 1.7 中还没有解决

`Netty` 在 `NIO` 的基础上 , 封装了 Java 原生的 `NIO API` , 解决了上述哪些问题呢 ？

相比 Java NIO，使用 `Netty` 开发程序，都简化了哪些步骤呢？...等等这系列问题也都是我们要问的问题。不过因为这篇只是介绍`NIO`相关知识，没有介绍`Netty API`的使用，所以介绍`Netty API`使用简单开发门槛低等优点有点站不住脚。那么就留到后面跟大家一起开启`Netty`学习之旅，探讨人人说好的`Netty`到底是不是江湖传言的那么好。

![](http://mmbiz.qpic.cn/mmbiz_png/GfOyjHGwCoN5Ky9IZgsdyZgM3lbEC5Ixe3ZgxBu5MdzvjWFyC7uqrsffcyhtrD8r8t0qicxbGmNxKYr4l4mANKQ/300?wx_fmt=png&wxfrom=19)

**面试突击**

大厂面试突击，专注分享面试题，如计算机基础、计算机网络、Java后端、前端Vue。

16篇原创内容

公众号

阅读 1009

​

写留言

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/SfAHMuUxqJ2JicQpKzblLHz64qoibVa3ATNA4rH8mIYXAF3OErAzxFKHzf5qiaiblb4rAMuAXXMJHEcKcvaHv4ia9rA/300?wx_fmt=png&wxfrom=18)

悟空聊架构

312

写留言

写留言

**留言**

暂无留言
