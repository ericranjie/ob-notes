# 

水滴与银弹

 _2021年11月25日 08:50_

编者荐语：

最近很多人问我 IO 多路复用的实现原理，本来想自己写一篇，但看到闪客的这篇文章，我觉得讲得非常清晰，推荐给大家。

以下文章来源于低并发编程 ，作者闪客

[

![](http://wx.qlogo.cn/mmhead/Szib8ySqErWKicsDp6kKGYLVPk4THU3kDliaExDbic56tFFzfeA8oGtTEt4QwKUz8YW6SnQHxpxB4KE/0)

**低并发编程**.

学技术也可以变得有趣

](https://mp.weixin.qq.com/s?__biz=MzIyOTYxNDI5OA==&mid=2247489816&idx=1&sn=c35e80395dc227007a87e57ae9890ff0&chksm=e8beaeaddfc927bb3e4a985b4a380b3a8f27a99b1332bc64c1932c93c5c19721d5a0a904af90&mpshare=1&scene=24&srcid=1125pi8HTDfUfBSBYw49QCdt&sharer_sharetime=1637808059458&sharer_shareid=5fb9813bfe9ffc983435bfc8d8c5e9ca&key=daf9bdc5abc4e8d0176ec00c4b4a1ca17681afe1a1e5a1d2cbb7d6fc2cf938e86559d897124cb98b9bd2233b587c242ff2b04c23ab9deb83e358212242b19e6e32e4715f97e7fc6a15a55e894884f81e0bd3367176638e55dc34dd081c2b759879ab70504637683b22f95301e48aeedde9eed10f241b734fc611793b41af970c&ascene=0&uin=MTEwNTU1MjgwMw%3D%3D&devicetype=Windows+11+x64&version=63090b19&lang=zh_CN&countrycode=CN&exportkey=n_ChQIAhIQ90fjoqajpdxfo5uiu247ZxLmAQIE97dBBAEAAAAAAC0FFJKgNjAAAAAOpnltbLcz9gKNyK89dVj02zAZPyL7kXNlAwB3p%2BpoiYwd9enG6JMxBAmNxV6zdMix5hrnVk%2FBBux5gySflrc8U1E3yjhTk1792EeT%2FiT0n9fh0qeqtdU5in43VPyyGpaoUhjQAz4o9GSyJtrGgKJUslCaXvvoN1P8m%2B3IdnJF%2BRhGd6AE2MQ%2BCB7Y5Kl5yQpBdqnfn5PeUslq7iPzhmMZ2V1elueVMXbeDLkq6AbpZu1gr%2BqWAq2H9o%2BPBYJF1VppiKgtaKTtdFeI4Vkr870F&acctmode=0&pass_ticket=zqBYyy9mbD0ATjPB8ssZHwrviBZjzRIkGB4AFf6QIJlYk1VCZNAFAzI4GS%2FcPbL3&wx_header=1&fasttmpl_type=0&fasttmpl_fullversion=7351805-zh_CN-zip&fasttmpl_flag=1#)

为了讲多路复用，当然还是要跟风，采用鞭尸的思路，先讲讲传统的网络 IO 的弊端，用拉踩的方式捧起多路复用 IO 的优势。  

为了方便理解，以下所有代码都是伪代码，知道其表达的意思即可。

**Let's go**

  

# 

**阻塞 IO**

  

服务端为了处理客户端的连接和请求的数据，写了如下代码。

```
listenfd = socket();   // 打开一个网络通信端口bind(listenfd);        // 绑定listen(listenfd);      // 监听while(1) {  connfd = accept(listenfd);  // 阻塞建立连接  int n = read(connfd, buf);  // 阻塞读数据  doSomeThing(buf);  // 利用读到的数据做些什么  close(connfd);     // 关闭连接，循环等待下一个连接}
```

这段代码会执行得磕磕绊绊，就像这样。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

可以看到，服务端的线程阻塞在了两个地方，一个是 accept 函数，一个是 read 函数。

如果再把 read 函数的细节展开，我们会发现其阻塞在了两个阶段。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

这就是传统的阻塞 IO。  

整体流程如下图。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

所以，如果这个连接的客户端一直不发数据，那么服务端线程将会一直阻塞在 read 函数上不返回，也无法接受其他客户端连接。

这肯定是不行的。

  

# 

**非阻塞 IO**

为了解决上面的问题，其关键在于改造这个 read 函数。

有一种聪明的办法是，每次都创建一个新的进程或线程，去调用 read 函数，并做业务处理。

```
while(1) {  connfd = accept(listenfd);  // 阻塞建立连接  pthread_create（doWork);  // 创建一个新的线程}void doWork() {  int n = read(connfd, buf);  // 阻塞读数据  doSomeThing(buf);  // 利用读到的数据做些什么  close(connfd);     // 关闭连接，循环等待下一个连接}
```

这样，当给一个客户端建立好连接后，就可以立刻等待新的客户端连接，而不用阻塞在原客户端的 read 请求上。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

不过，这不叫非阻塞 IO，只不过用了多线程的手段使得主线程没有卡在 read 函数上不往下走罢了。操作系统为我们提供的 read 函数仍然是阻塞的。

所以真正的非阻塞 IO，不能是通过我们用户层的小把戏，**而是要恳请操作系统为我们提供一个非阻塞的 read 函数**。

这个 read 函数的效果是，如果没有数据到达时（到达网卡并拷贝到了内核缓冲区），立刻返回一个错误值（-1），而不是阻塞地等待。

操作系统提供了这样的功能，只需要在调用 read 前，将文件描述符设置为非阻塞即可。

```
fcntl(connfd, F_SETFL, O_NONBLOCK);int n = read(connfd, buffer) != SUCCESS);
```

这样，就需要用户线程循环调用 read，直到返回值不为 -1，再开始处理业务。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

这里我们注意到一个细节。

非阻塞的 read，指的是在数据到达前，即数据还未到达网卡，或者到达网卡但还没有拷贝到内核缓冲区之前，这个阶段是非阻塞的。

当数据已到达内核缓冲区，此时调用 read 函数仍然是阻塞的，需要等待数据从内核缓冲区拷贝到用户缓冲区，才能返回。

整体流程如下图

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

# 

**IO 多路复用**

为每个客户端创建一个线程，服务器端的线程资源很容易被耗光。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

当然还有个聪明的办法，我们可以每 accept 一个客户端连接后，将这个文件描述符（connfd）放到一个数组里。

```
fdlist.add(connfd);
```

然后弄一个新的线程去不断遍历这个数组，调用每一个元素的非阻塞 read 方法。

```
while(1) {  for(fd <-- fdlist) {    if(read(fd) != -1) {      doSomeThing();    }  }}
```

这样，我们就成功用一个线程处理了多个客户端连接。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

你是不是觉得这有些多路复用的意思？

但这和我们用多线程去将阻塞 IO 改造成看起来是非阻塞 IO 一样，这种遍历方式也只是我们用户自己想出的小把戏，每次遍历遇到 read 返回 -1 时仍然是一次浪费资源的系统调用。

在 while 循环里做系统调用，就好比你做分布式项目时在 while 里做 rpc 请求一样，是不划算的。

所以，还是得恳请操作系统老大，提供给我们一个有这样效果的函数，我们将一批文件描述符通过一次系统调用传给内核，由内核层去遍历，才能真正解决这个问题。

## 

**select**

select 是操作系统提供的系统调用函数，通过它，我们可以把一个文件描述符的数组发给操作系统， 让操作系统去遍历，确定哪个文件描述符可以读写， 然后告诉我们去处理：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

select系统调用的函数定义如下。

```
int select(    int nfds,    fd_set *readfds,    fd_set *writefds,    fd_set *exceptfds,    struct timeval *timeout);// nfds:监控的文件描述符集里最大文件描述符加1// readfds：监控有读数据到达文件描述符集合，传入传出参数// writefds：监控写数据到达文件描述符集合，传入传出参数// exceptfds：监控异常发生达文件描述符集合, 传入传出参数// timeout：定时阻塞监控时间，3种情况//  1.NULL，永远等下去//  2.设置timeval，等待固定时间//  3.设置timeval里时间均为0，检查描述字后立即返回，轮询
```

服务端代码，这样来写。

首先一个线程不断接受客户端连接，并把 socket 文件描述符放到一个 list 里。

```
while(1) {  connfd = accept(listenfd);  fcntl(connfd, F_SETFL, O_NONBLOCK);  fdlist.add(connfd);}
```

然后，另一个线程不再自己遍历，而是调用 select，将这批文件描述符 list 交给操作系统去遍历。

```
while(1) {  // 把一堆文件描述符 list 传给 select 函数  // 有已就绪的文件描述符就返回，nready 表示有多少个就绪的  nready = select(list);  ...}
```

不过，当 select 函数返回后，用户依然需要遍历刚刚提交给操作系统的 list。

只不过，操作系统会将准备就绪的文件描述符做上标识，用户层将不会再有无意义的系统调用开销。

```
while(1) {  nready = select(list);  // 用户层依然要遍历，只不过少了很多无效的系统调用  for(fd <-- fdlist) {    if(fd != -1) {      // 只读已就绪的文件描述符      read(fd, buf);      // 总共只有 nready 个已就绪描述符，不用过多遍历      if(--nready == 0) break;    }  }}
```

正如刚刚的动图中所描述的，其直观效果如下。（同一个动图消耗了你两次流量，气不气？）

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

可以看出几个细节：

1. select 调用需要传入 fd 数组，需要拷贝一份到内核，高并发场景下这样的拷贝消耗的资源是惊人的。（可优化为不复制）

2. select 在内核层仍然是通过遍历的方式检查文件描述符的就绪状态，是个同步过程，只不过无系统调用切换上下文的开销。（内核层可优化为异步事件通知）

3. select 仅仅返回可读文件描述符的个数，具体哪个可读还是要用户自己遍历。（可优化为只返回给用户就绪的文件描述符，无需用户做无效的遍历）

整个 select 的流程图如下。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

可以看到，这种方式，既做到了一个线程处理多个客户端连接（文件描述符），又减少了系统调用的开销（多个文件描述符只有一次 select 的系统调用 + n 次就绪状态的文件描述符的 read 系统调用）。

  

## 

**poll**

poll 也是操作系统提供的系统调用函数。

```
int poll(struct pollfd *fds, nfds_tnfds, int timeout);struct pollfd {  intfd; /*文件描述符*/  shortevents; /*监控的事件*/  shortrevents; /*监控事件中满足条件返回的事件*/};
```

它和 select 的主要区别就是，去掉了 select 只能监听 1024 个文件描述符的限制。  

## 

**epoll**

epoll 是最终的大 boss，它解决了 select 和 poll 的一些问题。

还记得上面说的 select 的三个细节么？

1. select 调用需要传入 fd 数组，需要拷贝一份到内核，高并发场景下这样的拷贝消耗的资源是惊人的。（可优化为不复制）

2. select 在内核层仍然是通过遍历的方式检查文件描述符的就绪状态，是个同步过程，只不过无系统调用切换上下文的开销。（内核层可优化为异步事件通知）

3. select 仅仅返回可读文件描述符的个数，具体哪个可读还是要用户自己遍历。（可优化为只返回给用户就绪的文件描述符，无需用户做无效的遍历）

所以 epoll 主要就是针对这三点进行了改进。

1. 内核中保存一份文件描述符集合，无需用户每次都重新传入，只需告诉内核修改的部分即可。

2. 内核不再通过轮询的方式找到就绪的文件描述符，而是通过异步 IO 事件唤醒。

3. 内核仅会将有 IO 事件的文件描述符返回给用户，用户也无需遍历整个文件描述符集合。

具体，操作系统提供了这三个函数。

第一步，创建一个 epoll 句柄

```
int epoll_create(int size);
```

第二步，向内核添加、修改或删除要监控的文件描述符。

```
int epoll_ctl(  int epfd, int op, int fd, struct epoll_event *event);
```

第三步，类似发起了 select() 调用

```
int epoll_wait(  int epfd, struct epoll_event *events, int max events, int timeout);
```

使用起来，其内部原理就像如下一般丝滑。  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

如果你想继续深入了解 epoll 的底层原理，推荐阅读飞哥的《[图解 | 深入揭秘 epoll 是如何实现 IO 多路复用的！](http://mp.weixin.qq.com/s?__biz=MjM5Njg5NDgwNA==&mid=2247484905&idx=1&sn=a74ed5d7551c4fb80a8abe057405ea5e&chksm=a6e304d291948dc4fd7fe32498daaae715adb5f84ec761c31faf7a6310f4b595f95186647f12&scene=21#wechat_redirect)》，从 linux 源码级别，一行一行非常硬核地解读 epoll 的实现原理，且配有大量方便理解的图片，非常适合源码控的小伙伴阅读。

  

> > **后记**
> > 
> >   
> > 
> >   
> > 
> >   

   

大白话总结一下。

一切的开始，都起源于这个 read 函数是操作系统提供的，而且是阻塞的，我们叫它 **阻塞 IO**。

为了破这个局，程序员在用户态通过多线程来防止主线程卡死。

后来操作系统发现这个需求比较大，于是在操作系统层面提供了非阻塞的 read 函数，这样程序员就可以在一个线程内完成多个文件描述符的读取，这就是 **非阻塞 IO**。

但多个文件描述符的读取就需要遍历，当高并发场景越来越多时，用户态遍历的文件描述符也越来越多，相当于在 while 循环里进行了越来越多的系统调用。

后来操作系统又发现这个场景需求量较大，于是又在操作系统层面提供了这样的遍历文件描述符的机制，这就是 **IO 多路复用**。

多路复用有三个函数，最开始是 select，然后又发明了 poll 解决了 select 文件描述符的限制，然后又发明了 epoll 解决 select 的三个不足。

---

所以，IO 模型的演进，其实就是时代的变化，倒逼着操作系统将更多的功能加到自己的内核而已。

如果你建立了这样的思维，很容易发现网上的一些错误。

比如好多文章说，多路复用之所以效率高，是因为用一个线程就可以监控多个文件描述符。

这显然是知其然而不知其所以然，多路复用产生的效果，完全可以由用户态去遍历文件描述符并调用其非阻塞的 read 函数实现。而多路复用快的原因在于，操作系统提供了这样的系统调用，使得原来的 while 循环里多次系统调用，变成了一次系统调用 + 内核层遍历这些文件描述符。

就好比我们平时写业务代码，把原来 while 循环里调 http 接口进行批量，改成了让对方提供一个批量添加的 http 接口，然后我们一次 rpc 请求就完成了批量添加。

阅读 6750

​

写留言

**留言 19**

- Cooper
    
    2021年11月25日
    
    赞4
    
    批量添加这个比喻真形象
    
- 130
    
    2021年11月25日
    
    赞1
    
    每次都认真看每一篇，都很不错的文章
    
- 稳稳的幸福
    
    2021年11月25日
    
    赞1
    
    老大 我的学习脑图呢![[苦涩]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
    水滴与银弹
    
    作者2021年11月25日
    
    赞
    
    最近有点忙，后面会安排
    
- 仝亚亚
    
    2021年11月25日
    
    赞1
    
    先赞后看![[呲牙]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
- 彬
    
    2021年11月28日
    
    赞
    
    这个文章写得真好，之前看过3-4篇介绍多路复用的文章，都没这篇透彻，特别动图处理的非常形象，谢谢分享~![[抱拳]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
- tk
    
    2021年11月25日
    
    赞
    
    高质量文章
    
- lw
    
    2021年11月25日
    
    赞
    
    代码写的好，图也画的溜tql
    
- 章鱼小白
    
    2021年11月25日
    
    赞
    
    我就是文中所说的对多路复用的理解仅限于多线程+非阻塞read的人了
    
- Aurora
    
    2021年11月25日
    
    赞
    
    动画好评
    
- Milittle
    
    2021年11月25日
    
    赞
    
    这个写的清晰明了 赞一个
    
- 橘子
    
    2021年11月25日
    
    赞
    
    醉了，今天正在想找一篇。这就来了。
    
- 洁仔
    
    2021年11月25日
    
    赞
    
    某闪窃喜![[旺柴]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
    水滴与银弹
    
    作者2021年11月25日
    
    赞
    
    ![[呲牙]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
- 黄朝殿
    
    2021年11月25日
    
    赞
    
    动画太好了，求教如何制作？
    
    水滴与银弹
    
    作者2021年11月25日
    
    赞
    
    flash动画
    
- 严建国
    
    2021年11月25日
    
    赞
    
    哈哈哈没点进来我还以为是kaito自己写的 应该会比这篇详细
    
    水滴与银弹
    
    作者2021年11月25日
    
    赞
    
    这篇也很棒
    
    严建国
    
    2021年11月25日
    
    赞
    
    破玩意系列都挺赞，这篇之前看过就是异步io唤醒那些没展开讲
    

已无更多数据

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/gB9Yvac5K3OBY9BQfiaUT3Pk36rVKvoY6CY3FjR65gibwRBvD74I9Tx1iaqQNZKAvReApSlD527pV3rQQ8spibHHcw/300?wx_fmt=png&wxfrom=18)

水滴与银弹

891923

19

写留言

**留言 19**

- Cooper
    
    2021年11月25日
    
    赞4
    
    批量添加这个比喻真形象
    
- 130
    
    2021年11月25日
    
    赞1
    
    每次都认真看每一篇，都很不错的文章
    
- 稳稳的幸福
    
    2021年11月25日
    
    赞1
    
    老大 我的学习脑图呢![[苦涩]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
    水滴与银弹
    
    作者2021年11月25日
    
    赞
    
    最近有点忙，后面会安排
    
- 仝亚亚
    
    2021年11月25日
    
    赞1
    
    先赞后看![[呲牙]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
- 彬
    
    2021年11月28日
    
    赞
    
    这个文章写得真好，之前看过3-4篇介绍多路复用的文章，都没这篇透彻，特别动图处理的非常形象，谢谢分享~![[抱拳]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
- tk
    
    2021年11月25日
    
    赞
    
    高质量文章
    
- lw
    
    2021年11月25日
    
    赞
    
    代码写的好，图也画的溜tql
    
- 章鱼小白
    
    2021年11月25日
    
    赞
    
    我就是文中所说的对多路复用的理解仅限于多线程+非阻塞read的人了
    
- Aurora
    
    2021年11月25日
    
    赞
    
    动画好评
    
- Milittle
    
    2021年11月25日
    
    赞
    
    这个写的清晰明了 赞一个
    
- 橘子
    
    2021年11月25日
    
    赞
    
    醉了，今天正在想找一篇。这就来了。
    
- 洁仔
    
    2021年11月25日
    
    赞
    
    某闪窃喜![[旺柴]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
    水滴与银弹
    
    作者2021年11月25日
    
    赞
    
    ![[呲牙]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
- 黄朝殿
    
    2021年11月25日
    
    赞
    
    动画太好了，求教如何制作？
    
    水滴与银弹
    
    作者2021年11月25日
    
    赞
    
    flash动画
    
- 严建国
    
    2021年11月25日
    
    赞
    
    哈哈哈没点进来我还以为是kaito自己写的 应该会比这篇详细
    
    水滴与银弹
    
    作者2021年11月25日
    
    赞
    
    这篇也很棒
    
    严建国
    
    2021年11月25日
    
    赞
    
    破玩意系列都挺赞，这篇之前看过就是异步io唤醒那些没展开讲
    

已无更多数据