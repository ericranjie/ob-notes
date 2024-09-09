
原创 Wang Shaodong vivo互联网技术

 _2021年11月24日 21:01_

> 作者：vivo互联网服务器团队-Wang Shaodong

  

一、概述

  

众所周知，Redis是一个高性能的数据存储框架，在高并发的系统设计中，Redis也是一个比较关键的组件，是我们提升系统性能的一大利器。深入去理解Redis高性能的原理显得越发重要，当然Redis的高性能设计是一个系统性的工程，涉及到很多内容，本文重点关注Redis的IO模型，以及基于IO模型的线程模型。

  

我们从IO的起源开始，讲述了阻塞IO、非阻塞IO、多路复用IO。基于多路复用IO，我们也梳理了几种不同的Reactor模型，并分析了几种Reactor模型的优缺点。基于Reactor模型我们开始了Redis的IO模型和线程模型的分析，并总结出Redis线程模型的优点、缺点，以及后续的Redis多线程模型方案。本文的重点是对Redis线程模型设计思想的梳理，捋顺了设计思想，就是一通百通的事了。

  

> 注：本文的代码都是伪代码，主要是为了示意，不可用于生产环境。

  

二、网络IO模型发展史

  

我们常说的网络IO模型，主要包含阻塞IO、非阻塞IO、多路复用IO、信号驱动IO、异步IO，本文重点关注跟Redis相关的内容，所以我们重点分析阻塞IO、非阻塞IO、多路复用IO，帮助大家后续更好的理解Redis网络模型。

  

我们先看下面这张图；

  

![图片](https://mmbiz.qpic.cn/mmbiz_png/4g5IMGibSxt5Z10x8K04Dm5HVpHeMHDSf9kLvDXtMQx0FjQzZeUO0exqjnWibj3r6bibwOqcQ2aJnjUIke0Ubz0yA/640?wx_fmt=png&wxfrom=13&tp=wxpic)

  

2.1 阻塞IO

  

我们经常说的阻塞IO其实分为两种，一种是单线程阻塞，一种是多线程阻塞。这里面其实有两个概念，阻塞和线程。

  

- 阻塞：指调用结果返回之前，当前线程会被挂起，调用线程只有在得到结果之后才会返回；
    
- 线程：系统调用的线程个数。
    

  

像建立连接、读、写都涉及到系统调用，本身是一个阻塞的操作。

  

**2.1.1 单线程阻塞**

  

服务端单线程来处理，当客户端请求来临时，服务端用主线程来处理连接、读取、写入等操作。

  

以下用代码模拟了单线程的阻塞模式；

```
import java.net.Socket;
```

  

我们准备用两个客户端同时发起连接请求、来模拟单线程阻塞模式的现象。同时发起连接，通过服务端日志，我们发现此时服务端只接受了其中一个连接，主线程被阻塞在上一个连接的read方法上。

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

我们尝试关闭第一个连接，看第二个连接的情况，我们希望看到的现象是，主线程返回，新的客户端连接被接受。

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

从日志中发现，在第一个连接被关闭后，第二个连接的请求被处理了，也就是说第二个连接请求在排队，直到主线程被唤醒，才能接收下一个请求，符合我们的预期。

  

**此时不仅要问，为什么呢？**

  

主要原因在于accept、read、write三个函数都是阻塞的，主线程在系统调用的时候，线程是被阻塞的，其他客户端的连接无法被响应。

  

通过以上流程，我们很容易发现这个过程的缺陷，服务器每次只能处理一个连接请求，CPU没有得到充分利用，性能比较低。如何充分利用CPU的多核特性呢？自然而然的想到了——**多线程逻辑。**

  

**2.1.2 多线程阻塞**

  

对工程师而言，代码解释一切，直接上代码。

  

**BIO多线程**

```
package net.io.bio;
```

  

同样，我们并行发起两个请求；

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

两个请求，都被接受，服务端新增两个线程来处理客户端的连接和后续请求。

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

我们用多线程解决了，服务器同时只能处理一个请求的问题，但同时又带来了一个问题，如果客户端连接比较多时，服务端会创建大量的线程来处理请求，但线程本身是比较耗资源的，创建、上下文切换都比较耗资源，又如何去解决呢？

  

2.2 非阻塞

  

如果我们把所有的Socket（文件句柄，后续用Socket来代替fd的概念，尽量减少概念，减轻阅读负担）都放到队列里，只用一个线程来轮训所有的Socket的状态，如果准备好了就把它拿出来，是不是就减少了服务端的线程数呢？

  

一起看下代码，单纯非阻塞模式，我们基本上不用，为了演示逻辑，我们模拟了相关代码如下；

```
package net.io.bio;
```

  

系统初始化，等待连接；

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

发起两个客户端连接，线程开始轮询两个连接中是否有数据。

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

两个连接分别输入数据后，轮询线程发现有数据准备好了，开始相关的逻辑处理（单线程、多线程都可）。

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

再用一张流程图辅助解释下（系统实际采用文件句柄，此时用Socket来代替，方便大家理解）。

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

服务端专门有一个线程来负责轮询所有的Socket，来确认操作系统是否完成了相关事件，如果有则返回处理，如果无继续轮询，大家一起来思考下？此时又带来了什么问题呢。

  

CPU的空转、系统调用（每次轮询到涉及到一次系统调用，通过内核命令来确认数据是否准备好），造成资源的浪费，那有没有一种机制，来解决这个问题呢？

  

2.3 IO多路复用

  

服务端没有专门的线程来做轮询操作（应用程序端非内核），而是由事件来触发，当有相关读、写、连接事件到来时，主动唤起服务端线程来进行相关逻辑处理。模拟了相关代码如下；

  

**IO多路复用**

```
import java.net.InetSocketAddress;
```

同时创建两个连接；

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

两个连接无阻塞的被创建；

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

无阻塞的接收读写；

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

再用一张流程图辅助解释下（系统实际采用文件句柄，此时用Socket来代替，方便大家理解）。

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

当然操作系统的多路复用有好几种实现方式，我们经常使用的select()，epoll模式这里不做过多的解释，有兴趣的可以查看相关文档，IO的发展后面还有异步、事件等模式，我们在这里不过多的赘述，我们更多的是为了解释Redis线程模式的发展。

  

三、NIO线程模型解释

  

我们一起来聊了阻塞、非阻塞、IO多路复用模式，那Redis采用的是哪种呢？

  

Redis采用的是IO多路复用模式，所以我们重点来了解下多路复用这种模式，如何在更好的落地到我们系统中，不可避免的我们要聊下Reactor模式。

  

首先我们做下相关的名词解释；

  

- **Reactor**：类似NIO编程中的Selector，负责I/O事件的派发；
    
- **Acceptor**：NIO中接收到事件后，处理连接的那个分支逻辑；
    
- **Handler**：消息读写处理等操作类。
    

  

3.1 单Reactor单线程模型

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

**处理流程**

  

- Reactor监听连接事件、Socket事件，当有连接事件过来时交给Acceptor处理，当有Socket事件过来时交个对应的Handler处理。
    

  

**优点**

  

- 模型比较简单，所有的处理过程都在一个连接里；
    
- 实现上比较容易，模块功能也比较解耦，Reactor负责多路复用和事件分发处理，Acceptor负责连接事件处理，Handler负责Scoket读写事件处理。
    

  

**缺点**

  

- 只有一个线程，连接处理和业务处理共用一个线程，无法充分利用CPU多核的优势。
    
- 在流量不是特别大、业务处理比较快的时候系统可以有很好的表现，当流量比较大、读写事件比较耗时情况下，容易导致系统出现性能瓶颈。
    

  

怎么去解决上述问题呢？既然业务处理逻辑可能会影响系统瓶颈，那我们是不是可以把业务处理逻辑单拎出来，交给线程池来处理，一方面减小对主线程的影响，另一方面利用CPU多核的优势。这一点希望大家要理解透彻，方便我们后续理解Redis由单线程模型到多线程模型的设计的思路。

  

3.2 单Reactor多线程模型

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

这种模型相对单Reactor单线程模型，只是将业务逻辑的处理逻辑交给了一个线程池来处理。

  

**处理流程**

  

- Reactor监听连接事件、Socket事件，当有连接事件过来时交给Acceptor处理，当有Socket事件过来时交个对应的Handler处理。
    
- Handler完成读事件后，包装成一个任务对象，交给线程池来处理，把业务处理逻辑交给其他线程来处理。
    

  

**优点**

  

- 让主线程专注于通用事件的处理（连接、读、写），从设计上进一步解耦；
    
- 利用CPU多核的优势。
    

  

**缺点**

  

- 貌似这种模型已经很完美了，我们再思考下，如果客户端很多、流量特别大的时候，通用事件的处理（读、写）也可能会成为主线程的瓶颈，因为每次读、写操作都涉及系统调用。
    

  

有没有什么好的办法来解决上述问题呢？通过以上的分析，大家有没有发现一个现象，当某一个点成为系统瓶颈点时，想办法把他拿出来，交个其他线程来处理，那这种场景是否适用呢？

  

3.3 多Reactor多线程模型

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

这种模型相对单Reactor多线程模型，只是将Scoket的读写处理从mainReactor中拎出来，交给subReactor线程来处理。

  

**处理流程**

  

- mainReactor主线程负责连接事件的监听和处理，当Acceptor处理完连接过程后，主线程将连接分配给subReactor；
    
- subReactor负责mainReactor分配过来的Socket的监听和处理，当有Socket事件过来时交个对应的Handler处理；
    
- Handler完成读事件后，包装成一个任务对象，交给线程池来处理，把业务处理逻辑交给其他线程来处理。
    

  

**优点**

  

- 让主线程专注于连接事件的处理，子线程专注于读写事件吹，从设计上进一步解耦；
    
- 利用CPU多核的优势。
    

  

**缺点**

  

- 实现上会比较复杂，在极度追求单机性能的场景中可以考虑使用。
    

  

四、Redis的线程模型

  

4.1 概述

  

以上我们聊了，IO网路模型的发展历史，也聊了IO多路复用的reactor模式。那Redis采用的是哪种reactor模式呢？在回答这个问题前，我们先梳理几个概念性的问题。

  

Redis服务器中有两类事件，文件事件和时间事件。

  

- **文件事件**：在这里可以把文件理解为Socket相关的事件，比如连接、读、写等；
    
- **时间时间**：可以理解为定时任务事件，比如一些定期的RDB持久化操作。
    

  

本文重点聊下Socket相关的事件。

  

4.2 模型图

  

首先我们来看下Redis服务的线程模型图；

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

IO多路复用负责各事件的监听（连接、读、写等），当有事件发生时，将对应事件放入队列中，由事件分发器根据事件类型来进行分发；

  

如果是连接事件，则分发至连接应答处理器；GET、SET等redis命令分发至命令请求处理器。

  

命令处理完后产生命令回复事件，再由事件队列，到事件分发器，到命令回复处理器，回复客户端响应。

  

4.3 一次客户端和服务端的交互流程

  

**4.3.1 连接流程**

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

**连接过程**

  

- Redis服务端主线程监听固定端口，并将连接事件绑定连接应答处理器。
    
- 客户端发起连接后，连接事件被触发，IO多路复用程序将连接事件包装好后丢人事件队列，然后由事件分发处理器分发给连接应答处理器。
    
- 连接应答处理器创建client对象以及Socket对象，我们这里关注Socket对象，并产生ae_readable事件，和命令处理器关联，标识后续该Socket对可读事件感兴趣，也就是开始接收客户端的命令操作。
    
- 当前过程都是由一个主线程负责处理。
    

  

**4.3.2 命令执行流程**

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

**SET命令执行过程**

  

- 客户端发起SET命令，IO多路复用程序监听到该事件后（读事件），将数据包装成事件丢到事件队列中（事件在上个流程中绑定了命令请求处理器）；
    
- 事件分发处理器根据事件类型，将事件分发给对应的命令请求处理器；
    
- 命令请求处理器，读取Socket中的数据，执行命令，然后产生ae_writable事件，并绑定命令回复处理器；
    
- IO多路复用程序监听到写事件后，将数据包装成事件丢到事件队列中，事件分发处理器根据事件类型分发至命令回复处理器；
    
- 命令回复处理器，将数据写入Socket中返回给客户端。
    

  

4.4 模型优缺点

  

以上流程分析我们可以看出Redis采用的是单线程Reactor模型，我们也分析了这种模式的优缺点，那Redis为什么还要采用这种模式呢？

  

**Redis本身的特性**

  

- 命令执行基于内存操作，业务处理逻辑比较快，所以命令处理这一块单线程来做也能维持一个很高的性能。
    

  

**优点**

  

- Reactor单线程模型的优点，参考上文。
    

  

**缺点**

  

- Reactor单线程模型的缺点也同样在Redis中来体现，唯一不同的地方就在于业务逻辑处理（命令执行）这块不是系统瓶颈点。
    
- 随着流量的上涨，IO操作的的耗时会越来越明显（read操作，内核中读数据到应用程序。write操作，应用程序中的数据到内核），当达到一定阀值时系统的瓶颈就体现出来了。
    

  

**Redis又是如何去解的呢？**

  

哈哈~将耗时的点从主线程拎出来呗？那Redis的新版本是这么做的吗？我们一起来看下。

  

4.5 Redis多线程模式

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

Redis的多线程模型跟”多Reactor多线程模型“、“单Reactor多线程模型有点区别”，但同时用了两种Reactor模型的思想，具体如下；

  

- Redis的多线程模型是将IO操作多线程化，本身逻辑处理过程（命令执行过程）依旧是单线程，借助了单Reactor思想，实现上又有所区分。
    
- 将IO操作多线程化，又跟单Reactor衍生出多Reactor的思想一致，都是将IO操作从主线程中拎出来。
    

  

**命令执行大致流程**

  

- 客户端发送请求命令，触发读就绪事件，服务端主线程将Socket（为了简化理解成本，统一用Socket来代表连接）放入一个队列，主线程不负责读；
    
- IO 线程通过Socket读取客户端的请求命令，主线程忙轮询，等待所有 I/O 线程完成读取任务，IO线程只负责读不负责执行命令；
    
- 主线程一次性执行所有命令，执行过程和单线程一样，然后需要返回的连接放入另外一个队列中，有IO线程来负责写出（主线程也会写）；
    
- 主线程忙轮询，等待所有 I/O 线程完成写出任务。
    

  

五、总结

  

了解一个组件，更多的是要去了解他的设计思路，要去思考为什么要这么设计，做这种技术选型的背景是啥，对后续做系统架构设计有什么参考意义等等。一通百通，希望对大家有参考意义。

  

END

猜你喜欢

- [初探 Redis 客户端 Lettuce：真香！](http://mp.weixin.qq.com/s?__biz=MzI4NjY4MTU5Nw==&mid=2247491453&idx=1&sn=c74628fed0ae2ee5d79ec428c36f13f5&chksm=ebd86fefdcafe6f96a4fbec28af80cfcf8a8375dbc8bc6a548243fdd1461798e54fc79bced90&scene=21#wechat_redirect)
    
- [从源码角度分析 Mybatis 工作原理](http://mp.weixin.qq.com/s?__biz=MzI4NjY4MTU5Nw==&mid=2247491885&idx=2&sn=53bd38a0734076018ddfa640f20bbed4&chksm=ebdb91bfdcac18a92e0eee2cd7aa4b8b721ad13ae4b381e27e39bb6bd8d4e106f9e49b20cdb6&scene=21#wechat_redirect)
    
- [带你走进MySQL全新高可用解决方案-MGR](http://mp.weixin.qq.com/s?__biz=MzI4NjY4MTU5Nw==&mid=2247491752&idx=2&sn=0fc24f990210b55021c069fbda7f6b69&chksm=ebdb903adcac192c80d286b49d976f9ca52092f43edb52205d73c4e5ba5db983f5a359f29ae2&scene=21#wechat_redirect)
    

![](https://res.wx.qq.com/t/fed_upload/b39ef69e-c4d6-4169-8612-5f00a84860e7/wx-avatar-default.svg)

公众号

数据库38

数据库 · 目录

上一篇深入剖析Redis客户端Jedis的特性和原理下一篇vivo数据库与存储平台的建设和探索

阅读 2258

修改于2021年11月25日

​