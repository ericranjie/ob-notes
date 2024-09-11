# 

原创 往事敬秋风 深度Linux

 _2024年08月25日 09:10_

我还以为是硬件里边电平的边沿触发和水平触发，水平触发是指在信号达到或超过某个阈值时触发，只要信号保持在该阈值之上（或之下），触发状态就会一直持续。换句话说，水平触发检测的是信号的绝对水平而不关心其变化速率。常见的例子是开关门磁感应器，当门磁感应器所接收到的磁场强度超过预设阈值时，可以认为门被打开了。

![](http://mmbiz.qpic.cn/mmbiz_png/dkX7hzLPUR0Ao40RncDiakbKx1Dy4uJicoqwn5GZ5r7zSMmpwHdJt32o95wdQmPZrBW038j8oRSSQllpnOUDlmUg/300?wx_fmt=png&wxfrom=19)

**深度Linux**

拥有15年项目开发经验及丰富教学经验，曾就职国内知名企业项目经理，部门负责人等职务。研究领域：Windows&Linux平台C/C++后端开发、Linux系统内核等技术。

184篇原创内容

公众号

边缘触发是指在信号上升沿或下降沿出现时触发。当信号从低电平跃迁到高电平（上升沿）或从高电平跃迁到低电平（下降沿）时才会触发。边缘触发更加关注信号变化的瞬间，并且只有在出现特定变化时才会进行响应。例如，在数字系统中，计数器可能会在输入信号上升沿或下降沿处进行计数操作，后来才发现理解错了，原来软件编程里也有类似的概念。

水平触发（LT）和边缘触发（ET）是在事件驱动编程中常见的两种触发模式，特别是在使用epoll系统调用进行I/O多路复用时。

水平触发（LT）：

- 当一个文件描述符上有可读或可写事件发生时，LT模式会持续通知应用程序直到处理完所有就绪事件。
    
- 应用程序需要反复检查文件描述符是否可读/可写，并且对于非阻塞IO操作，可能会返回EAGAIN错误。
    
- LT模式适合于基于阻塞I/O或者非阻塞I/O的应用。
    

边缘触发（ET）：

- 当一个文件描述符从无就绪变为就绪状态时，ET模式会通过一次通知告知应用程序。
    
- 应用程序需要立即处理该事件，并读取/写入尽可能多的数据，直到再次返回EAGAIN错误或者数据全部处理完成。
    
- ET模式适合于高性能的异步非阻塞I/O应用，能够更好地利用系统资源。
    

在边缘触发（ET）模式下，epoll_wait系统调用仅在新事件到达时才返回就绪状态，即只有当文件描述符从无就绪状态切换为就绪状态时才会触发通知。这要求应用程序必须立即对事件进行处理，并尽可能地读取/写入更多数据。如果应用程序没有及时处理完所有就绪事件或者未读取全部数据，则可能导致事件丢失。

相比之下，水平触发（LT）模式中，每次调用epoll_wait都会收到已经就绪的事件通知，即使应用程序未处理完该事件或者未读取全部数据。这意味着需要反复检查文件描述符是否可读/可写，并且对于非阻塞IO操作，可能会返回EAGAIN错误。

优缺点方面，在边缘触发模式下，由于只有在状态变化时才通知应用程序，可以减少不必要的上下文切换和事件通知频率，适合高性能异步非阻塞I/O场景。而水平触发模式则更适合于轮询方式的处理，在某些特定场景下可能产生较大的开销和资源浪费。

选择使用哪种模式需要根据具体的应用需求和性能要求来决定，并考虑用户态和内核态之间的切换次数以及资源利用效率。 了解边缘触发：需要等到epoll_wait从内核中获取到新的就绪事件才会触发

- 水平触发：只要满足就绪事件的条件，epoll_wait都会收到事件(未处理或未处理完的事件)
    
- 优缺点：考虑不同应用场景下，用户态和内核态的切换次数的细节
    

## 一、epoll的数据结构

epoll工作环境？

- epoll工作在应用程序和内核协议栈之间。
    
- epoll是在内核协议栈和vfs都有的情况下才有的。
    
![[Pasted image 20240911163105.png]]
![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

epoll 的核心数据结构是：1个红黑树和1个双向链表。还有3个核心API。
![[Pasted image 20240911163111.png]]
![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

可以看到，链表和红黑树使用的是同一个结点。实际上是红黑树管理所有的IO，当内部IO就绪的时候就会调用epoll的回调函数，将相应的IO添加到就绪链表上。数据结构有epitm和eventpoll，分别代表红黑树和单个结点，在单个结点上分别使用rbn和rblink使得结点同时指向两个数据结构。

### 1.1红黑树

- 因为链表在查询，删除的时候毫无疑问时间复杂度是O(n)；
    
- 数组查询很快，但是删除和新增时间复杂度是O(n)；
    
- 二叉搜索树虽然查询效率是lgn，但是如果不是平衡的，那么就会退化为线性查找，复杂度直接来到O(n)；
    
- B+树是平衡多路查找树，主要是通过降低树的高度来存储上亿级别的数据，但是它的应用场景是内存放不下的时候能够用最少的IO访问次数从磁盘获取数据。比如数据库聚簇索引，成百上千万的数据内存无法满足查找就需要到内存查找，而因为B+树层高很低，只需要几次磁盘IO就能获取数据到内存，所以在这种磁盘到内存访问上B+树更适合。
    

因为我们处理上万级的fd，它们本身的存储空间并不会很大，所以倾向于在内存中去实现管理，而红黑树是一种非常优秀的平衡树，它完全是在内存中操作，而且查找，删除和新增时间复杂度都是lgn，效率非常高，因此选择用红黑树实现epoll是最佳的选择。

当然不选择用AVL树是因为红黑树是不符合AVL树的平衡条件的，红黑树用非严格的平衡来换取增删节点时候旋转次数的降低，任何不平衡都会在三次旋转之内解决；而AVL树是严格平衡树，在增加或者删除节点的时候，根据不同情况，旋转的次数比红黑树要多。所以红黑树的插入效率更高。

### 1.2就绪socket列表-双向链表

就绪列表存储的是就绪的socket，所以它应能够快速的插入数据。

程序可能随时调用epoll_ctl添加监视socket，也可能随时删除。当删除时，若该socket已经存放在就绪列表中，它也应该被移除。（事实上，每个epoll_item既是红黑树节点，也是链表节点，删除红黑树节点，自然删除了链表节点）所以就绪列表应是一种能够快速插入和删除的数据结构。双向链表就是这样一种数据结构，epoll使用双向链表来实现就绪队列（rdllist）。

红黑树和就绪队列的关系

红黑树的结点和就绪队列的结点的同一个节点，所谓的加入就绪队列，就是将结点的前后指针联系到一起。所以就绪了不是将红黑树结点delete掉然后加入队列。他们是同一个结点，不需要delete。

```c
struct epitem {RB_ ENTRY(epitem) rbn;LIST_ ENTRY(epitem) rdlink;int rdy; //exist in List
			   int sockfd;struct epoll_ event event ;};struct eventpoll {ep_ _rb_ tree rbr;int rbcnt ;LIST_ HEAD( ,epitem) rdlist;int rdnum;int waiting;pthread_ mutex_ t mtx; //rbtree updatepthread_ spinlock_ t 1ock; //rdList updatepthread_ cond_ _t cond; //bLock for eventpthread_ mutex_ t cdmtx; //mutex for cond};|
```

### 1.3三个API

```
int epoll_create(int size)
```

功能：内核会产生一个epoll 实例数据结构并返回一个文件描述符epfd，这个特殊的描述符就是epoll实例的句柄，后面的两个接口都以它为中心。同时也会创建红黑树和就绪列表，红黑树来管理注册fd，就绪列表来收集所有就绪fd。size参数表示所要监视文件描述符的最大值，不过在后来的Linux版本中已经被弃用（同时，size不要传0，会报invalid argument错误）。

```
int epoll_ctl(int epfd， int op， int fd， struct epoll_event *event)
```

功能：将被监听的socket文件描述符添加到红黑树或从红黑树中删除或者对监听事件进行修改；同时向内核中断处理程序注册一个回调函数，内核在检测到某文件描述符可读/可写时会调用回调函数，该回调函数将文件描述符放在就绪链表中。

```
int epoll_wait(int epfd， struct epoll_event *events， int maxevents， int timeout);
```

功能：阻塞等待注册的事件发生，返回事件的数目，并将触发的事件写入events数组中。

events: 用来记录被触发的events，其大小应该和maxevents一致

maxevents: 返回的events的最大个数处于ready状态的那些文件描述符会被复制进ready list中，epoll_wait用于向用户进程返回ready list(就绪列表)。

events和maxevents两个参数描述一个由用户分配的struct epoll event数组，调用返回时，内核将就绪列表(双向链表)复制到这个数组中，并将实际复制的个数作为返回值。

注意，如果就绪列表比maxevents长，则只能复制前maxevents个成员；反之，则能够完全复制就绪列表。

另外，struct epoll event结构中的events域在这里的解释是：在被监测的文件描述符上实际发生的事件。

调用epoll_create时，在内核cache里建了个红黑树用于存储以后epoll_ctl传来的socket外，还会再建立一个list链表，用于存储准备就绪的事件，内部使用回调机制，红黑树中的节点通过回调函数添加到双向链表。

当epoll_wait调用时，仅仅观察这个双向链表里有没有数据即可。有数据就返回，没有数据就sleep，等到timeout时间到后即使链表没数据也返回。所以，epoll_wait非常高效。而且，通常情况下即使我们要监控百万计的句柄，大多一次也只返回很少量的准备就绪句柄而已，所以，epoll_wait仅需要从内核态copy少量的句柄到用户态而已。

epoll和poll/select区别？

- （1）使用接口：select/poll需要把fds总集拷贝到内核协议栈中，epoll不需要。
    
- （2）实现原理：select/poll在内核内循环 遍历是否有就绪io，epoll是单个加入红黑树。
    

解释：poll/select每次都要把fds总集拷贝到内核协议栈内，内核采取轮询/遍历，返回就绪的fds集合。（大白话：poll/select的fds是存放在用户态协议栈，调用时拷贝到内核协议栈中并轮询，轮询完成后再拷贝到用户态协议栈）。而epoll是通过epoll_ctl每次有新的io就加入到红黑树里，有触发的时候用epoll_wait带出即可，不需要拷贝总集。

## 二、epoll的实现原理

为什么需要epoll？

epoll是Linux操作系统提供的一种事件驱动的I/O模型，用于高效地处理大量并发连接的网络编程。它相比于传统的select和poll方法，具有更高的性能和扩展性。使用epoll可以实现以下几个优势：

1. 高效处理大量并发连接：epoll采用了事件驱动的方式，只有当有可读或可写事件发生时才会通知应用程序，避免了遍历所有文件描述符的开销。
    
2. 内核与用户空间数据拷贝少：使用epoll时，内核将就绪的文件描述符直接填充到用户空间的事件数组中，减少了内核与用户空间之间数据拷贝次数。
    
3. 支持边缘触发（Edge Triggered）模式：边缘触发模式下，仅在状态变化时才通知应用程序。这意味着每次通知只包含最新状态的文件描述符信息，可以有效避免低效循环检查。
    
4. 支持水平触发（Level Triggered）模式：水平触发模式下，在就绪期间不断地进行通知，直到应用程序处理完该文件描述符。
    

select与poll的缺陷？

`select` 和 `poll` 都是Unix系统中用来监视一组文件描述符的变化的系统调用。它们可以监视文件描述符的三种变化：可读性、可写性和异常条件。`select` 和 `poll` 的主要缺陷如下：

- 文件描述符数量限制：select 和 poll 都有一个限制，就是它们只能监视少于1024个文件描述符的变化。这对于现代的网络编程来说是不够的，因为一个进程往往需要监视成千上万的连接。
    
- 效率问题：虽然 select 和 poll 可以监视多个文件描述符，但是它们在每次调用的时候都需要传递所有要监视的文件描述符集合，这会导致效率的降低。
    
- 信息不足：select 和 poll 返回的只是哪些文件描述符已经准备好了，但是它们并不告诉你具体是哪一个。这就需要对所有要监视的文件描述符进行遍历，直到找到准备好的文件描述符为止。
    
- 信号中断：select 和 poll 调用可以被信号中断，这可能会导致调用失败。
    
- 为了解决这些问题，现代操作系统中引入了新的系统调用 epoll 来替代 select 和 poll。epoll 没有文件描述符的限制，它可以监视大量的文件描述符，并且可以实现即开即用，无需传递所有文件描述符集合。此外，epoll 可以直接告诉你哪些文件描述符已经准备好，这大大提高了处理效率。
    

### 2.1epoll操作

epoll 在 linux 内核中申请了一个简易的文件系统，把原先的一个 select 或者 poll 调用分为了三个部分：调用 epoll_create 建立一个 epoll 对象（在 epoll 文件系统中给这个句柄分配资源）、调用 epoll_ctl 向 epoll 对象中添加连接的套接字、调用 epoll_wait 收集发生事件的连接。这样只需要在进程启动的时候建立一个 epoll 对象，并在需要的时候向它添加或者删除连接就可以了，因此，在实际收集的时候，epoll_wait 的效率会非常高，因为调用的时候只是传递了发生 IO 事件的连接。

epoll 实现

我们以 linux 内核 2.6 为例，说明一下 epoll 是如何高效的处理事件的，当某一个进程调用 epoll_create 方法的时候，Linux 内核会创建一个 eventpoll 结构体，这个结构体中有两个重要的成员。

```
第一个是 rb_root rbr，这是红黑树的根节点，存储着所有添加到 epoll 中的事件，也就是这个 epoll 监控的事件。第二个是 list_head rdllist 这是一个双向链表，保存着将要通过 epoll_wait 返回给用户的、满足条件的事件。
```

每一个 epoll 对象都有一个独立的 eventpoll 结构体，这个结构体会在内核空间中创造独立的内存，用于存储使用 epoll_ctl 方法向 epoll 对象中添加进来的事件。这些事件都会挂到 rbr 红黑树中，这样就能够高效的识别重复添加的节点。

所有添加到 epoll 中的事件都会与设备（如网卡等）驱动程序建立回调关系，也就是说，相应的事件发生时会调用这里的方法。这个回调方法在内核中叫做 ep_poll_callback，它把这样的事件放到 rdllist 双向链表中。在 epoll 中，对于每一个事件都会建立一个 epitem 结构体。

当调用 epoll_wait 检查是否有发生事件的连接时，只需要检查 eventpoll 对象中的 rdllist 双向链表中是否有 epitem 元素，如果 rdllist 链表不为空，则把这里的事件复制到用户态内存中的同时，将事件数量返回给用户。通过这种方法，epoll_wait 的效率非常高。epoll-ctl 在向 epoll 对象中添加、修改、删除事件时，从 rbr 红黑树中查找事件也非常快。这样，epoll 就能够轻易的处理百万级的并发连接。

epoll工作模式

epoll 有两种工作模式，LT（水平触发）模式与 ET（边缘触发）模式。默认情况下，epoll 采用 LT 模式工作。两个的区别是：

- Level_triggered(水平触发)：当被监控的文件描述符上有可读写事件发生时，epoll_wait()会通知处理程序去读写。如果这次没有把数据一次性全部读写完(如读写缓冲区太小)，那么下次调用 epoll_wait() 时，它还会通知你在上没读写完的文件描述符上继续读写，当然如果你一直不去读写，它会一直通知你。如果系统中有大量你不需要读写的就绪文件描述符，而它们每次都会返回，这样会大大降低处理程序检索自己关心的就绪文件描述符的效率。
    
- Edge_triggered(边缘触发)：当被监控的文件描述符上有可读写事件发生时，epoll_wait() 会通知处理程序去读写。如果这次没有把数据全部读写完(如读写缓冲区太小)，那么下次调用 epoll_wait()时，它不会通知你，也就是它只会通知你一次，直到该文件描述符上出现第二次可读写事件才会通知你。这种模式比水平触发效率高，系统不会充斥大量你不关心的就绪文件描述符。
    

当然，在 LT 模式下开发基于 epoll 的应用要简单一些，不太容易出错，而在 ET 模式下事件发生时，如果没有彻底地将缓冲区的数据处理完，则会导致缓冲区的用户请求得不到响应。注意，默认情况下 Nginx 采用 ET 模式使用 epoll 的。

### 2.2 I/O 多路复用

(1)阻塞OR非阻塞

我们知道，对于 linux 来说，I/O 设备为特殊的文件，读写和文件是差不多的，但是 I/O 设备因为读写与内存读写相比，速度差距非常大。与 cpu 读写速度更是没法比，所以相比于对内存的读写，I/O 操作总是拖后腿的那个。网络 I/O 更是如此，我们很多时候不知道网络 I/O 什么时候到来，就好比我们点了一份外卖，不知道外卖小哥们什么时候送过来，**这个时候有两个处理办法：**

`第一个是我们可以先去睡觉，外卖小哥送到楼下了自然会给我们打电话，`

`这个时候我们在醒来取外卖就可以了。`

`第二个是我们可以每隔一段时间就给外卖小哥打个电话，`

`这样就能实时掌握外卖的动态信息了。`

第一种方式对应的就是阻塞的 I/O 处理方式，进程在进行 I/O 操作的时候，进入睡眠，如果有 I/O 时间到达，就唤醒这个进程。第二种方式对应的是非阻塞轮询的方式，进程在进行 I/O 操作后，每隔一段时间向内核询问是否有 I/O 事件到达，如果有就立刻处理。

阻塞的原理

工作队列

阻塞是进程调度的关键一环，指的是进程在等待某事件（如接收到网络数据）发生之前的等待状态，recv、select和epoll都是阻塞方法，以简单网络编程为例

下图中的计算机中运行着A、B、C三个进程，其中进程A执行着上述基础网络程序，一开始，这3个进程都被操作系统的工作队列所引用，处于运行状态，会分时执行
![[Pasted image 20240911163150.png]]
![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

当进程A执行到创建socket的语句时，操作系统会创建一个由文件系统管理的socket对象（如下图）。这个socket对象包含了发送缓冲区、接收缓冲区、等待队列等成员。等待队列是个非常重要的结构，它指向所有需要等待该socket事件的进程。
![[Pasted image 20240911163154.png]]
![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

当程序执行到recv时，操作系统会将进程A从工作队列移动到该socket的等待队列中（如下图）。由于工作队列只剩下了进程B和C，依据进程调度，cpu会轮流执行这两个进程的程序，不会执行进程A的程序。所以进程A被阻塞，不会往下执行代码，也不会占用cpu资源。
![[Pasted image 20240911163159.png]]
![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

ps：操作系统添加等待队列只是添加了对这个“等待中”进程的引用，以便在接收到数据时获取进程对象、将其唤醒，而非直接将进程管理纳入自己之下。上图为了方便说明，直接将进程挂到等待队列之下。

唤醒进程

当socket接收到数据后，操作系统将该socket等待队列上的进程重新放回到工作队列，该进程变成运行状态，继续执行代码。也由于socket的接收缓冲区已经有了数据，recv可以返回接收到的数据。

(2)线程池OR轮询

在现实中，我们当然选择第一种方式，但是在计算机中，情况就要复杂一些。我们知道，在 linux 中，不管是线程还是进程都会占用一定的资源，也就是说，系统总的线程和进程数是一定的。如果有许多的线程或者进程被挂起，无疑是白白消耗了系统的资源。而且，线程或者进程的切换也是需要一定的成本的，需要上下文切换，如果频繁的进行上下文切换，系统会损失很大的性能。一个网络服务器经常需要连接成千上万个客户端，而它能创建的线程可能之后几百个，线程耗光就不能对外提供服务了。这些都是我们在选择 I/O 机制的时候需要考虑的。这种阻塞的 I/O 模式下，一个线程只能处理一个流的 I/O 事件，这是问题的根源。

这个时候我们首先想到的是采用线程池的方式限制同时访问的线程数，这样就能够解决线程不足的问题了。但是这又会有第二个问题了，多余的任务会通过队列的方式存储在内存只能够，这样很容易在客户端过多的情况下出现内存不足的情况。

还有一种方式是采用轮询的方式，我们只要不停的把所有流从头到尾问一遍，又从头开始。这样就可以处理多个流了。

(3)代理

采用轮询的方式虽然能够处理多个 I/O 事件，但是也有一个明显的缺点，那就是会导致 CPU 空转。试想一下，如果所有的流中都没有数据，那么 CPU 时间就被白白的浪费了。

为了避免CPU空转，可以引进了一个代理。这个代理比较厉害，可以同时观察许多流的I/O事件，在空闲的时候，会把当前线程阻塞掉，当有一个或多个流有I/O事件时，就从阻塞态中醒来，于是我们的程序就会轮询一遍所有的流，这就是 select 与 poll 所做的事情，可见，采用 I/O 复用极大的提高了系统的效率。

### 2.3内核接收网络数据全过程

如下图所示，进程在recv阻塞期间，计算机收到了对端传送的数据（步骤①）。数据经由网卡传送到内存（步骤②），然后网卡通过中断信号通知CPU有数据到达，CPU执行中断程序（步骤③）。此处的中断程序主要有两项功能，先将网络数据写入到对应socket的接收缓冲区里面（步骤④），再唤醒进程A（步骤⑤），重新将进程A放入工作队列中。
![[Pasted image 20240911163207.png]]
![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

唤醒线程的过程如下图所示：
![[Pasted image 20240911163443.png]]
![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

## 三、协议栈如何与epoll通信？

协议栈和epoll模块之间的通信是异步的，没有耦合，不需要等待。

**通知时机：**

- （1）协议栈三次握手完成，往accept全连接队列里加入这个节点时，通知epoll有事件来了epollin；
    
- （2）客户端发了1个数据到协议栈，协议栈此时要返回ack给客户端的这里的时机，会通知epoll有事件可读 epollin。
    

## 四、epoll线程安全如何加锁？

1、对红黑树枷锁：一种是锁整棵树，另一种是锁子树。一般使用互斥锁。

2、对就绪队列枷锁：用自旋锁，队列操作比较简单，等到一些时间比让出线程更高效点。

### 4.1等待队列实现原理

(1)功能介绍

进程有多种状态，当进程做好准备后，它就处于就绪状态（TASK_RUNNING），放入运行队列，等待内核调度器来调度。当然，同一时刻可能有多个进程进入就绪状态，但是却可能只有1个CPU是空闲的，所以最后能不能在CPU上运行，还要取决于优先级等多种因素。当进程进行外部设备的IO等待操作时，由于外部设备的操作速度一般是非常慢的，所以进程会从就绪状态变为等待状态（休眠），进入等待队列，把CPU让给其它进程。直到IO操作完成，内核“唤醒”等待的进程，于是进程再度从等待状态变为就绪状态。

在用户态，进程进行IO操作时，可以有多种处理方式，如阻塞式IO，非阻塞式IO，多路复用(select/poll/epoll)，AIO（aio_read/aio_write）等等。这些操作在内核态都要用到等待队列。

(2)相关的结构体

typedef struct __wait_queue wait_queue_t;

```c
struct __wait_queue {
unsigned int flags;
#define WQ_FLAG_EXCLUSIVE 0x01
struct task_struct * task; // 等待队列节点对应的进程
					wait_queue_func_t func;   // 等待队列的回调函数 ，在进程被唤醒    
					struct list_head task_list;
					};
					这个是等待队列的节点，在很多等待队列里，这个func函数指针默认为空函数。但是，在select/poll/epoll函数中，这个func函数指针不为空，并且扮演着重要的角色。
					struct __wait_queue_head{
					    spinlock_t lock;
					        struct list_head task_list;
					        };
```

typedef struct __wait_queue_head wait_queue_head_t;这个是等待队列的头部。其中task_list里有指向下一个节点的指针。为了保证对等待队列的操作是原子的，还需要一个自旋锁lock。

这里需要提一下内核队列中被广泛使用的结构体struct list_head。

```c
struct list_head{    struct list_head *next, *prev;};
```

(3)实现原理

可以看到，等待队列的核心是一个list_head组成的双向链表。

其中，第一个节点是队列的头，类型为wait_queue_head_t，里面包含了一个list_head类型的成员task_list。

接下去的每个节点类型为 wait_queue_t，里面也有一个list_head类型的成员task_list，并且有个指针指向等待的进程。通过这种方式，内核组织了一个等待队列。

那么，这个等待队列怎样与一个事件关联呢？

在内核中，进程在文件操作等事件上的等待，一定会有一个对应的等待队列的结构体与之对应。例如，等待管道的文件操作（在内核看来，管道也是一种文件）的进程都放在管道对应inode.i_pipe->wait这个等待队列中。这样，如果管道文件操作完成，就可以很方便地通过inode.i_pipe->wait唤醒等待的进程。

在大部分情况下（如系统调用read），当前进程等待IO操作的完成，只要在内核堆栈中分配一个wait_queue_t的结构体，然后初始化，把task指向当前进程的task_struct，然后调用add_wait_queue（）放入等待队列即可。

但是，在select/poll中，由于系统调用要监视多个文件描述符的操作，因此要把当前进程放入多个文件的等待队列，并且要分配多个wait_queue_t结构体。这时候，在堆栈上分配是不合适的。因为内核堆栈很小。所以要通过动态分配的方式来分配wait_queue_t结构体。除了在一些结构体里直接定义等待队列的头部，内核的信号量机制也大量使用了等待队列。信号量是为了进行进程同步而引入的。与自旋锁不同的是，当一个进程无法获得信号量时，它会把自己放到这个信号量的等待队列中，转变为等待状态。当其它进程释放信号量时，会唤醒等待的进程。

epoll 关键结构体：

```
struct ep_pqueue{    poll_table pt;    struct epitem *epi;};
```

这个结构体类似于select/poll中的struct poll_wqueues。由于epoll需要在内核态保存大量信息，所以光光一个回调函数指针已经不能满足要求，所以在这里引入了一个新的结构体struct epitem。

```
struct epitem{     struct rb_node rbn;    红黑树，用来保存eventpoll     struct list_head rdllink;    双向链表，用来保存已经完成的eventpoll     struct epoll_filefd ffd;    这个结构体对应的被监听的文件描述符信息     int nwait;    poll操作中事件的个数     struct list_head pwqlist;    双向链表，保存着被监视文件的等待队列，功能类似于select/poll中的poll_table     struct eventpoll *ep;    指向eventpoll，多个epitem对应一个eventpoll     struct epoll_event event;    记录发生的事件和对应的fd     atomic_t usecnt;    引用计数     struct list_head fllink;    双向链表，用来链接被监视的文件描述符对应的struct file。因为file里有f_ep_link，    用来保存所有监视这个文件的epoll节点     struct list_head txlink;    双向链表，用来保存传输队列     unsigned int revents;    文件描述符的状态，在收集和传输时用来锁住空的事件集合};
```

该结构体用来保存与epoll节点关联的多个文件描述符，保存的方式是使用红黑树实现的hash表。至于为什么要保存，下文有详细解释。它与被监听的文件描述符一一对应。

```
struct eventpoll{     spinlock_t lock;    读写锁     struct mutex mtx;    读写信号量     wait_queue_head_t wq;     wait_queue_head_t poll_wait;     struct list_head rdllist;    已经完成的操作事件的队列。     struct rb_root rbr;    保存epoll监视的文件描述符    struct epitem *ovflist;    struct user_struct *user;};
```

这个结构体保存了epoll文件描述符的扩展信息，它被保存在file结构体的private_data中。它与epoll文件节点一一对应。通常一个epoll文件节点对应多个被监视的文件描述符。所以一个eventpoll结构体会对应多个epitem结构体。

那么，epoll中的等待事件放在哪里呢？见下面

```
struct eppoll_entry{     struct list_head llink;    void *base;    wait_queue_t wait;    wait_queue_head_t *whead;};与select/poll的struct poll_table_entry相比，epoll的表示等待队列节点的结构体只是稍有不同，与struct poll_table_entry比较一下。struct poll_table_entry{    struct file * filp;    wait_queue_t wait;    wait_queue_head_t * wait_address;};
```

由于epitem对应一个被监视的文件，所以通过base可以方便地得到被监视的文件信息。又因为一个文件可能有多个事件发生，所以用llink链接这些事件。

相关内核代码:fs/eventpoll.c

判断一个tcp套接字上是否有激活事件:net/ipv4/tcp.c:tcp_poll函数，每个epollfd在内核中有一个对应的eventpoll结构对象。

其中关键的成员是一个readylist(eventpoll:rdllist)和一棵红黑树(eventpoll:rbr)，eventpoll的红黑树中，红黑树的作用是使用者调用EPOLL_MOD的时候可以快速找到fd对应的epitem。

epoll_ctl的功能是实现一系列操作，如把文件与eventpollfs文件系统的inode节点关联起来。这里要介绍一下eventpoll结构体，它保存在file->f_private中，记录了eventpollfs文件系统的inode节点的重要信息，其中成员rbr保存了该epoll文件节点监视的所有文件描述符。组织的方式是一棵红黑树，这种结构体在查找节点时非常高效。首先它调用ep_find()从eventpoll中的红黑树获得epitem结构体。然后根据op参数的不同而选择不同的操作。如果op为EPOLL_CTL_ADD，那么正常情况下epitem是不可能在eventpoll的红黑树中找到的，所以调用ep_insert创建一个epitem结构体并插入到对应的红黑树中。

ep_insert()首先分配一个epitem对象，对它初始化后，把它放入对应的红黑树。此外，这个函数还要作一个操作，就是把当前进程放入对应文件操作的等待队列。这一步是由下面的代码完成的。

```
init_poll_funcptr(&epq.pt, ep_ptable_queue_proc);......revents = tfile->f_op->poll(tfile, &epq.pt);
```

函数先调用init_poll_funcptr注册了一个回调函数ep_ptable_queue_proc，ep_ptable_queue_proc函数会在调用f_op->poll时被执行。

```
static void ep_ptable_queue_proc(struct file *file, wait_queue_head_t *whead,           poll_table *pt){struct epitem *epi = ep_item_from_epqueue(pt);struct eppoll_entry *pwq;if (epi->nwait >= 0 && (pwq = kmem_cache_alloc(pwq_cache, GFP_KERNEL))) { init_waitqueue_func_entry(&pwq->wait, ep_poll_callback); pwq->whead = whead;pwq->base = epi;add_wait_queue(whead, &pwq->wait);list_add_tail(&pwq->llink, &epi->pwqlist); epi->nwait++;} else { epi->nwait = -1; }}
```

该函数分配一个epoll等待队列结点eppoll_entry：一方面把它挂到文件操作的等待队列中，另一方面把它挂到epitem的队列中。此外，它还注册了一个等待队列的回调函数ep_poll_callback。当文件操作完成，唤醒当前进程之前，会调用ep_poll_callback()，把eventpoll放到epitem的完成队列中（注释：通过查看代码，此处应该是把epitem放到eventpoll的完成队列，只有这样才能在epoll_wait()中只要看eventpoll的完成队列即可得到所有的完成文件描述符），并唤醒等待进程。

如果在执行f_op->poll以后，发现被监视的文件操作已经完成了，那么把它放在完成队列中了，并立即把等待操作的那些进程唤醒。

```
if (!(epi = kmem_cache_alloc(epi_cache, GFP_KERNEL)))return -ENOMEM;ep_rbtree_insert(ep, epi);
```

调用epoll_wait的时候,将readylist中的epitem出列,将触发的事件拷贝到用户空间.之后判断epitem是否需要重新添加回readylist。

epitem重新添加到readylist必须满足下列条件：

1) epitem上有用户关注的事件触发. 2) epitem被设置为水平触发模式(如果一个epitem被设置为边界触发则这个epitem不会被重新添加到readylist中，在什么时候重新添加到readylist请继续往下看)。 注意，如果epitem被设置为EPOLLONESHOT模式，则当这个epitem上的事件拷贝到用户空间之后,会将这个epitem上的关注事件清空(只是关注事件被清空,并没有从epoll中删除，要删除必须对那个描述符调EPOLL_DEL)，也就是说即使这个epitem上有触发事件，但是因为没有用户关注的事件所以不会被重新添加到readylist中。

epitem被添加到readylist中的各种情况(当一个epitem被添加到readylist如果有线程阻塞在epoll_wait中,那个线程会被唤醒)：

1)对一个fd调用EPOLL_ADD，如果这个fd上有用户关注的激活事件，则这个fd会被添加到readylist. 2)对一个fd调用EPOLL_MOD改变关注的事件，如果新增加了一个关注事件且对应的fd上有相应的事件激活，则这个fd会被添加到readylist. 3)当一个fd上有事件触发时(例如一个socket上有外来的数据)会调用ep_poll_callback(见eventpoll::ep_ptable_queue_proc),

如果触发的事件是用户关注的事件，则这个fd会被添加到readylist中，了解了epoll的执行过程之后,可以回答一个在使用边界触发时常见的疑问.在一个fd被设置为边界触发的情况下,调用read/write,如何正确的判断那个fd已经没有数据可读/不再可写.epoll文档中的建议是直到触发EAGAIN错误.而实际上只要你请求字节数小于read/write的返回值就可以确定那个fd上已经没有数据可读/不再可写，最后用一个epollfd监听另一个epollfd也是合法的,epoll通过调用eventpoll::ep_eventpoll_poll来判断一个epollfd上是否有触发的事件(只能是读事件)。

以下是个人读代码总结：

```
SYSCALL_DEFINE4(epoll_wait, int, epfd, struct epoll_event __user *, events,int, maxevents, int, timeout) SYSCALL_DEFINE4(epoll_ctl, int, epfd, int, op, int, fd,struct epoll_event __user *, event)epoll_ctl的机制大致如下：mutex_lock(&ep->mtx);epi = ep_find(ep, tfile, fd); //这里就是去ep->rbr 红黑树查找error = -EINVAL;switch (op) {case EPOLL_CTL_ADD:if (!epi) {             epds.events |= POLLERR | POLLHUP;            error = ep_insert(ep, &epds, tfile, fd);        } else            error = -EEXIST;        break;    case EPOLL_CTL_DEL:        if (epi)             error = ep_remove(ep, epi);        else            error = -ENOENT;        break;     case EPOLL_CTL_MOD:        if (epi) {             epds.events |= POLLERR | POLLHUP;            error = ep_modify(ep, epi, &epds);         } else             error = -ENOENT;         break;    }   mutex_unlock(&ep->mtx);
```

### 4.2源码分析

(1)sys_epoll_wait()函数：

```
/*  * Implement the event wait interface for the eventpoll file. It is the kernel  * part of the user space epoll_wait(2).  */  SYSCALL_DEFINE4(epoll_wait, int, epfd, struct epoll_event __user *, events,          int, maxevents, int, timeout)  {      int error;      struct file *file;      struct eventpoll *ep;        /* The maximum number of event must be greater than zero */      /*      * 检查maxevents参数。      */      if (maxevents <= 0 || maxevents > EP_MAX_EVENTS)          return -EINVAL;        /* Verify that the area passed by the user is writeable */      /*      * 检查用户空间传入的events指向的内存是否可写。参见__range_not_ok()。      */      if (!access_ok(VERIFY_WRITE, events, maxevents * sizeof(struct epoll_event))) {          error = -EFAULT;          goto error_return;      }        /* Get the "struct file *" for the eventpoll file */      /*      * 获取epfd对应的eventpoll文件的file实例，file结构是在epoll_create中创建      */      error = -EBADF;      file = fget(epfd);      if (!file)          goto error_return;        /*      * We have to check that the file structure underneath the fd      * the user passed to us _is_ an eventpoll file.      */      /*      * 通过检查epfd对应的文件操作是不是eventpoll_fops      * 来判断epfd是否是一个eventpoll文件。如果不是      * 则返回EINVAL错误。      */      error = -EINVAL;      if (!is_file_epoll(file))          goto error_fput;        /*      * At this point it is safe to assume that the "private_data" contains      * our own data structure.      */      ep = file->private_data;        /* Time to fish for events ... */      error = ep_poll(ep, events, maxevents, timeout);    error_fput:      fput(file);  error_return:        return error;  }
```

sys_epoll_wait（）是epoll_wait()对应的系统调用，主要用来获取文件状态已经就绪的事件，该函数检查参数、获取eventpoll文件后调用ep_poll（）来完成主要的工作。在分析ep_poll（）函数之前，先介绍一下使用epoll_wait（）时可能犯的错误（接下来介绍的就是我犯过的错误）：

返回EBADF错误

除非你故意指定一个不存在的文件描述符，否则几乎百分百肯定，你的程序有BUG了！从源码中可以看到调用fget（）函数返回NULL时，会返回此错误。fget（）源码如下：

```
struct file *fget(unsigned int fd)  {      struct file *file;      struct files_struct *files = current->files;        rcu_read_lock();      file = fcheck_files(files, fd);      if (file) {          if (!atomic_long_inc_not_zero(&file->f_count)) {              /* File object ref couldn't be taken */              rcu_read_unlock();              return NULL;          }      }      rcu_read_unlock();        return file;  }
```

主要看这句(struct files_struct *files = current->files;)，这条语句是获取描述当前进程已经打开的文件的files_struct结构，然后从这个结构中查找传入的fd对应的file实例，如果没有找到，说明当前进程中打开的文件不包括这个fd，所以几乎百分百肯定是程序设计的问题。我的程序出错，就是因为在父进程中创建了文件描述符，但是将子进程变为守护进程了，也就没有继承父进程中打开的文件。

死循环（一般不会犯，但是我是第一次用，犯了）

epoll_wait（）中有一个设置超时时间的参数，所以我在循环中没有使用睡眠队列的操作，想依赖epoll的睡眠操作，所以在返回值小于等于0时，直接进行下一次循环，没有充分考虑epoll_wait（）的返回值小于0时的不同情况，所以代码写成了下面的样子：

```
for(;;) {      ......      events = epoll_wait(fcluster_epfd, fcluster_wait_events,               fcluster_wait_size, 3000);          if (unlikely(events <= 0)) {              continue;          }      .......  }
```

当epoll_wait（）返回EBADF或EFAULT时，就会陷入死循环，因此此时还没有进入睡眠的操作。

(2)ep_poll（）函数

```
static int ep_poll(struct eventpoll *ep, struct epoll_event __user *events,             int maxevents, long timeout)  {      int res, eavail;      unsigned long flags;      long jtimeout;      wait_queue_t wait;        /*      * Calculate the timeout by checking for the "infinite" value (-1)      * and the overflow condition. The passed timeout is in milliseconds,      * that why (t * HZ) / 1000.      */      /*      * timeout是以毫秒为单位，这里是要转换为jiffies时间。      * 这里加上999(即1000-1)，是为了向上取整。      */      jtimeout = (timeout < 0 || timeout >= EP_MAX_MSTIMEO) ?          MAX_SCHEDULE_TIMEOUT : (timeout * HZ + 999) / 1000;    retry:      spin_lock_irqsave(&ep->lock, flags);        res = 0;      if (list_empty(&ep->rdllist)) {          /*          * We don't have any available event to return to the caller.          * We need to sleep here, and we will be wake up by          * ep_poll_callback() when events will become available.          */          init_waitqueue_entry(&wait, current);          wait.flags |= WQ_FLAG_EXCLUSIVE;          /*          * 将当前进程加入到eventpoll的等待队列中，          * 等待文件状态就绪或直到超时，或被          * 信号中断。          */          __add_wait_queue(&ep->wq, &wait);            for (;;) {              /*              * We don't want to sleep if the ep_poll_callback() sends us              * a wakeup in between. That's why we set the task state              * to TASK_INTERRUPTIBLE before doing the checks.              */              set_current_state(TASK_INTERRUPTIBLE);              /*              * 如果就绪队列不为空，也就是说已经有文件的状态              * 就绪或者超时，则退出循环。              */              if (!list_empty(&ep->rdllist) || !jtimeout)                  break;              /*              * 如果当前进程接收到信号，则退出              * 循环，返回EINTR错误              */              if (signal_pending(current)) {                  res = -EINTR;                  break;              }                spin_unlock_irqrestore(&ep->lock, flags);              /*              * 主动让出处理器，等待ep_poll_callback()将当前进程              * 唤醒或者超时,返回值是剩余的时间。从这里开始              * 当前进程会进入睡眠状态，直到某些文件的状态              * 就绪或者超时。当文件状态就绪时，eventpoll的回调              * 函数ep_poll_callback()会唤醒在ep->wq指向的等待队列中的进程。              */              jtimeout = schedule_timeout(jtimeout);              spin_lock_irqsave(&ep->lock, flags);          }          __remove_wait_queue(&ep->wq, &wait);            set_current_state(TASK_RUNNING);      }      /* Is it worth to try to dig for events ? */      /*      * ep->ovflist链表存储的向用户传递事件时暂存就绪的文件。      * 所以不管是就绪队列ep->rdllist不为空，或者ep->ovflist不等于      * EP_UNACTIVE_PTR，都有可能现在已经有文件的状态就绪。      * ep->ovflist不等于EP_UNACTIVE_PTR有两种情况，一种是NULL，此时      * 可能正在向用户传递事件，不一定就有文件状态就绪，      * 一种情况时不为NULL，此时可以肯定有文件状态就绪，      * 参见ep_send_events()。      */      eavail = !list_empty(&ep->rdllist) || ep->ovflist != EP_UNACTIVE_PTR;        spin_unlock_irqrestore(&ep->lock, flags);        /*      * Try to transfer events to user space. In case we get 0 events and      * there's still timeout left over, we go trying again in search of      * more luck.      */      /*      * 如果没有被信号中断，并且有事件就绪，      * 但是没有获取到事件(有可能被其他进程获取到了)，      * 并且没有超时，则跳转到retry标签处，重新等待      * 文件状态就绪。      */      if (!res && eavail &&          !(res = ep_send_events(ep, events, maxevents)) && jtimeout)          goto retry;        /*      * 返回获取到的事件的个数或者错误码      */      return res;  }
```

ep_poll（）的主要过程是：首先将超时时间（以毫秒为单位）转换为jiffies时间，然后检查是否有事件发生，如果没有事件发生，则将当前进程加入到eventpoll中的等待队列中，直到事件发生或者超时。如果有事件发生，则调用ep_send_events（）将发生的事件传入用户空间的内存。ep_send_events（）函数将用户传入的内存简单封装到ep_send_events_data结构中，然后调用ep_scan_ready_list（）将就绪队列中的事件传入用户空间的内存。

(3)ep_scan_ready_list（）函数

```
/**  * ep_scan_ready_list - Scans the ready list in a way that makes possible for  *                      the scan code, to call f_op->poll(). Also allows for  *                      O(NumReady) performance.  *  * @ep: Pointer to the epoll private data structure.  * @sproc: Pointer to the scan callback.  * @priv: Private opaque data passed to the @sproc callback.  *  * Returns: The same integer error code returned by the @sproc callback.  */  static int ep_scan_ready_list(struct eventpoll *ep,                    int (*sproc)(struct eventpoll *,                         struct list_head *, void *),                    void *priv)  {      int error, pwake = 0;      unsigned long flags;      struct epitem *epi, *nepi;      LIST_HEAD(txlist);        /*      * We need to lock this because we could be hit by      * eventpoll_release_file() and epoll_ctl().      */      /*      * 获取互斥锁，该互斥锁在移除eventpoll文件(eventpoll_release_file() )、      * 操作文件描述符(epoll_ctl())和向用户传递事件(epoll_wait())之间进行互斥      */      mutex_lock(&ep->mtx);        /*      * Steal the ready list, and re-init the original one to the      * empty list. Also, set ep->ovflist to NULL so that events      * happening while looping w/out locks, are not lost. We cannot      * have the poll callback to queue directly on ep->rdllist,      * because we want the "sproc" callback to be able to do it      * in a lockless way.      */      spin_lock_irqsave(&ep->lock, flags);      /*      * 将就绪队列中就绪的文件链表暂存在临时      * 表头txlist中，并且初始化就绪队列。      */      list_splice_init(&ep->rdllist, &txlist);      /*      * 将ovflist置为NULL，表示此时正在向用户空间传递      * 事件。如果此时有文件状态就绪，不会放在      * 就绪队列中，而是放在ovflist链表中。      */      ep->ovflist = NULL;      spin_unlock_irqrestore(&ep->lock, flags);        /*      * Now call the callback function.      */      /*      * 调用ep_send_events_proc()将就绪队列中的事件      * 存入用户传入的内存中。      */      error = (*sproc)(ep, &txlist, priv);        spin_lock_irqsave(&ep->lock, flags);      /*      * During the time we spent inside the "sproc" callback, some      * other events might have been queued by the poll callback.      * We re-insert them inside the main ready-list here.      */      /*      * 在调用sproc指向的函数将就绪队列中的事件      * 传递到用户传入的内存的过程中，可能有文件      * 状态就绪，这些事件会暂存在ovflist链表中，      * 所以这里要将ovflist中的事件移到就绪队列中。      */      for (nepi = ep->ovflist; (epi = nepi) != NULL;           nepi = epi->next, epi->next = EP_UNACTIVE_PTR) {          /*          * We need to check if the item is already in the list.          * During the "sproc" callback execution time, items are          * queued into ->ovflist but the "txlist" might already          * contain them, and the list_splice() below takes care of them.          */          if (!ep_is_linked(&epi->rdllink))              list_add_tail(&epi->rdllink, &ep->rdllist);      }      /*      * We need to set back ep->ovflist to EP_UNACTIVE_PTR, so that after      * releasing the lock, events will be queued in the normal way inside      * ep->rdllist.      */      /*      * 重新初始化ovflist，表示传递事件已经完成，      * 之后再有文件状态就绪，这些事件会直接      * 放在就绪队列中。      */      ep->ovflist = EP_UNACTIVE_PTR;        /*      * Quickly re-inject items left on "txlist".      */      /*      * 如果sproc指向的函数ep_send_events_proc()中处理出错或者某些文件的      * 触发方式设置为水平触发(Level Trigger)，txlist中可能还有事件，需要      * 将这些就绪的事件重新添加回eventpoll文件的就绪队列中。      */      list_splice(&txlist, &ep->rdllist);        if (!list_empty(&ep->rdllist)) {          /*          * Wake up (if active) both the eventpoll wait list and          * the ->poll() wait list (delayed after we release the lock).          */          if (waitqueue_active(&ep->wq))              wake_up_locked(&ep->wq);          if (waitqueue_active(&ep->poll_wait))              pwake++;      }      spin_unlock_irqrestore(&ep->lock, flags);        mutex_unlock(&ep->mtx);        /* We have to call this outside the lock */      if (pwake)          ep_poll_safewake(&ep->poll_wait);        return error;  }
```

ep_scan_ready_list（）函数的参数sproc指向的函数是ep_send_events_proc（），参见ep_send_events（）函数。

(4)ep_send_events_proc（）函数

```c
/*  * @head:已经就绪的文件列表  * @priv:用来存储已经就绪的文件  */
static int ep_send_events_proc(struct eventpoll *ep, struct list_head *head,
							   void *priv)  {
							         struct ep_send_events_data *esed = priv;
							               int eventcnt;
							                     unsigned int revents;      struct epitem *epi;
							                           struct epoll_event __user *uevent;        /*      * We can loop without lock because we are passed a task private list.      * Items cannot vanish during the loop because ep_scan_ready_list() is      * holding "mtx" during this call.      */
							                                 for (eventcnt = 0, uevent = esed->events;
							                                            !list_empty(head) && eventcnt < esed->maxevents;) {          epi = list_first_entry(head, struct epitem, rdllink);
							                                                        list_del_init(&epi->rdllink);            /*          * 调用文件的poll函数有两个作用，一是在文件的唤醒          * 队列上注册回调函数，二是返回文件当前的事件状          * 态，如果第二个参数为NULL，则只是查看文件当前          * 状态。          */
							                                                                  revents = epi->ffd.file->f_op->poll(epi->ffd.file, NULL) &              epi->event.events;            /*          * If the event mask intersect the caller-requested one,          * deliver the event to userspace. Again, ep_scan_ready_list()          * is holding "mtx", so no operations coming from userspace          * can change the item.          */          if (revents) {              /*              * 向用户内存传值失败时，将当前epitem实例重新放回              * 到链表中，从这里也可以看出，在处理失败后，head指向的              * 链表(对应ep_scan_ready_list()中的临时变量txlist)中              * 有可能会没有完全处理完，因此在ep_scan_ready_list()中              * 需要下面的语句              *    list_splice(&txlist, &ep->rdllist);              * 来将未处理的事件重新放回到eventpoll文件的就绪队列中。              */              if (__put_user(revents, &uevent->events) ||                  __put_user(epi->event.data, &uevent->data)) {                  list_add(&epi->rdllink, head);                  /*                  * 如果此时已经获取了部分事件，则返回已经获取的事件个数，                  * 否则返回EFAULT错误。                  */                  return eventcnt ? eventcnt : -EFAULT;              }              eventcnt++;              uevent++;              if (epi->event.events & EPOLLONESHOT)                  epi->event.events &= EP_PRIVATE_BITS;              /*              * 如果是触发方式不是边缘触发(Edge Trigger)，而是水平              * 触发(Level Trigger)，需要将当前的epitem实例添加回              * 链表中，下次读取事件时会再次上报。              */              else if (!(epi->event.events & EPOLLET)) {                  /*                  * If this file has been added with Level                  * Trigger mode, we need to insert back inside                  * the ready list, so that the next call to                  * epoll_wait() will check again the events                  * availability. At this point, noone can insert                  * into ep->rdllist besides us. The epoll_ctl()                  * callers are locked out by                  * ep_scan_ready_list() holding "mtx" and the                  * poll callback will queue them in ep->ovflist.                  */                  list_add_tail(&epi->rdllink, &ep->rdllist);              }          }      }        return eventcnt;  }
```

### 4.3如何加锁

3个api做什么事情

```c
epoll_create() ===》创建红黑树的根节点epoll_ctl() ===》add,del,mod 增加、删除、修改结点epoll_wait() ===》把就绪队列的结点copy到用户态放到events里面，跟recv函数很像
```

分析加锁

- 如果有3个线程同时操作epoll，有哪些地方需要加锁？我们用户层面一共就只有3个api可以使用
    
- 如果同时调用 epoll_create() ，那就是创建三颗红黑树，没有涉及到资源竞争，没有关系。
    
- 如果同时调用 epoll_ctl() ，对同一颗红黑树进行，增删改，这就涉及到资源竞争需要加锁了，此时我们对整棵树进行加锁。
    
- 如果同时调用epoll_wait() ，其操作的是就绪队列，所以需要对就绪队列进行加锁。
    

我们要扣住epoll的工作环境，在应用程序调用 epoll_ctl() ，协议栈会不会有回调操作红黑树结点？调用epoll_wait() copy出来的时候，协议栈会不会操作操作红黑树结点加入就绪队列？综上所述：

```c
epoll_ctl() 对红黑树加锁epoll_wait()对就绪队列加锁回调函数()   对红黑树加锁,对就绪队列加锁
```

那么红黑树加什么锁，就绪队列加什么锁呢？

对于红黑树这种节点比较多的时候，采用互斥锁来加锁。就绪队列就跟生产者消费者一样，结点是从协议栈回调函数来生产的，消费是epoll_wait()来消费。那么对于队列而言，用自旋锁（对于队列而言，插入删除比较简单，cpu自旋等待比让出的成本更低，所以用自旋锁）。

## 五、ET与LT的实现

- ET边沿触发，只触发一次
    
- LT水平触发，如果没有读完就一直触发
    

代码如何实现ET和LT的效果呢？水平触发和边沿触发不是故意设计出来的，这是自然而然，水到渠成的功能。水平触发和边沿触发代码只需要改一点点就能实现。从协议栈检测到接收数据，就调用一次回调，这就是ET，接收到数据，调用一次回调。而LT水平触发，检测到recvbuf里面有数据就调用回调。所以ET和LT就是在使用回调的次数上面的差异。

那么具体如何实现呢？协议栈流程里面触发回调，是天然的符合ET只触发一次的。那么如果是LT，在recv之后，如果缓冲区还有数据那么加入到就绪队列。那么如果是LT，在send之后，如果缓冲区还有空间那么加入到就绪队列。那么这样就能实现LT了。

## 六、epoll内核源码详解

网上很多博客说epoll使用了共享内存,这个是完全错误的 ,可以阅读源码,会发现完全没有使用共享内存的任何api，而是 使用了copy_from_user跟__put_user进行内核跟用户虚拟空间数据交互。

```c
/* *  fs/eventpoll.c (Efficient event retrieval implementation) *  Copyright (C) 2001,...,2009	 Davide Libenzi * *  This program is free software; you can redistribute it and/or modify *  it under the terms of the GNU General Public License as published by *  the Free Software Foundation; either version 2 of the License, or *  (at your option) any later version. * *  Davide Libenzi <davidel@xmailserver.org> * */
/* * 在深入了解epoll的实现之前, 先来了解内核的3个方面. * 1. 等待队列 waitqueue * 我们简单解释一下等待队列: * 队列头(wait_queue_head_t)往往是资源生产者, * 队列成员(wait_queue_t)往往是资源消费者, * 当头的资源ready后, 会逐个执行每个成员指定的回调函数, * 来通知它们资源已经ready了, 等待队列大致就这个意思. * 2. 内核的poll机制 * 被Poll的fd, 必须在实现上支持内核的Poll技术, * 比如fd是某个字符设备,或者是个socket, 它必须实现 * file_operations中的poll操作, 给自己分配有一个等待队列头. * 主动poll fd的某个进程必须分配一个等待队列成员, 添加到 * fd的对待队列里面去, 并指定资源ready时的回调函数. * 用socket做例子, 它必须有实现一个poll操作, 这个Poll是 * 发起轮询的代码必须主动调用的, 该函数中必须调用poll_wait(), * poll_wait会将发起者作为等待队列成员加入到socket的等待队列中去. * 这样socket发生状态变化时可以通过队列头逐个通知所有关心它的进程. * 这一点必须很清楚的理解, 否则会想不明白epoll是如何 * 得知fd的状态发生变化的. * 3. epollfd本身也是个fd, 所以它本身也可以被epoll, * 可以猜测一下它是不是可以无限嵌套epoll下去...  * * epoll基本上就是使用了上面的1,2点来完成. * 可见epoll本身并没有给内核引入什么特别复杂或者高深的技术, * 只不过是已有功能的重新组合, 达到了超过select的效果. *//*  * 相关的其它内核知识: * 1. fd我们知道是文件描述符, 在内核态, 与之对应的是struct file结构, * 可以看作是内核态的文件描述符. * 2. spinlock, 自旋锁, 必须要非常小心使用的锁, * 尤其是调用spin_lock_irqsave()的时候, 中断关闭, 不会发生进程调度, * 被保护的资源其它CPU也无法访问. 这个锁是很强力的, 所以只能锁一些 * 非常轻量级的操作. * 3. 引用计数在内核中是非常重要的概念, * 内核代码里面经常有些release, free释放资源的函数几乎不加任何锁, * 这是因为这些函数往往是在对象的引用计数变成0时被调用, * 既然没有进程在使用在这些对象, 自然也不需要加锁. * struct file 是持有引用计数的. *//* --- epoll相关的数据结构 --- *//* * This structure is stored inside the "private_data" member of the file * structure and rapresent the main data sructure for the eventpoll * interface. *//* 每创建一个epollfd, 内核就会分配一个eventpoll与之对应, 可以说是 * 内核态的epollfd. */
struct eventpoll {	/* Protect the this structure access */	
spinlock_t lock;	/*	 * This mutex is used to ensure that files are not removed	 * while epoll is using them. This is held during the event	 * collection loop, the file cleanup path, the epoll file exit	 * code and the ctl operations.	 */	/* 添加, 修改或者删除监听fd的时候, 以及epoll_wait返回, 向用户空间	 * 传递数据时都会持有这个互斥锁, 所以在用户空间可以放心的在多个线程	 * 中同时执行epoll相关的操作, 内核级已经做了保护. */	
struct mutex mtx;	/* Wait queue used by sys_epoll_wait() */	/* 调用epoll_wait()时, 我们就是"睡"在了这个等待队列上... */	
wait_queue_head_t wq;	/* Wait queue used by file->poll() */	/* 这个用于epollfd本事被poll的时候... */	
wait_queue_head_t poll_wait;	/* List of ready file descriptors */	/* 所有已经ready的epitem都在这个链表里面 */	
struct list_head rdllist;	/* RB tree root used to store monitored fd structs */	/* 所有要监听的epitem都在这里 */	
struct rb_root rbr;	/*		这是一个单链表链接着所有的struct epitem当event转移到用户空间时	 */	 * This is a single linked list that chains all the "struct epitem" that	 * happened while transfering ready events to userspace w/out	 * holding ->lock.	 */	
struct epitem *ovflist;	/* The user that created the eventpoll descriptor */	/* 这里保存了一些用户变量, 比如fd监听数量的最大值等等 */	struct user_struct *user;};/* * Each file descriptor added to the eventpoll interface will * have an entry of this type linked to the "rbr" RB tree. *//* epitem 表示一个被监听的fd */struct epitem {	/* RB tree node used to link this structure to the eventpoll RB tree */	/* rb_node, 当使用epoll_ctl()将一批fds加入到某个epollfd时, 内核会分配	 * 一批的epitem与fds们对应, 而且它们以rb_tree的形式组织起来, tree的root	 * 保存在epollfd, 也就是struct eventpoll中. 	 * 在这里使用rb_tree的原因我认为是提高查找,插入以及删除的速度.	 * rb_tree对以上3个操作都具有O(lgN)的时间复杂度 */	struct rb_node rbn;	/* List header used to link this structure to the eventpoll ready list */	/* 链表节点, 所有已经ready的epitem都会被链到eventpoll的rdllist中 */	struct list_head rdllink;	/*	 * Works together "struct eventpoll"->ovflist in keeping the	 * single linked chain of items.	 */	/* 这个在代码中再解释... */	struct epitem *next;	/* The file descriptor information this item refers to */	/* epitem对应的fd和struct file */	struct epoll_filefd ffd;	/* Number of active wait queue attached to poll operations */	int nwait;	/* List containing poll wait queues */	struct list_head pwqlist;	/* The "container" of this item */	/* 当前epitem属于哪个eventpoll */	struct eventpoll *ep;	/* List header used to link this item to the "struct file" items list */	struct list_head fllink;	/* The structure that describe the interested events and the source fd */	/* 当前的epitem关系哪些events, 这个数据是调用epoll_ctl时从用户态传递过来 */	struct epoll_event event;};struct epoll_filefd {	struct file *file;	int fd;};/* poll所用到的钩子Wait structure used by the poll hooks */struct eppoll_entry {	/* List header used to link this structure to the "struct epitem" */	struct list_head llink;	/* The "base" pointer is set to the container "struct epitem" */	struct epitem *base;	/*	 * Wait queue item that will be linked to the target file wait	 * queue head.	 */	wait_queue_t wait;	/* The wait queue head that linked the "wait" wait queue item */	wait_queue_head_t *whead;};/* Wrapper struct used by poll queueing */struct ep_pqueue {	poll_table pt;	struct epitem *epi;};/* Used by the ep_send_events() function as callback private data */struct ep_send_events_data {	int maxevents;	struct epoll_event __user *events;};/* --- 代码注释 --- *//* 你没看错, 这就是epoll_create()的真身, 基本啥也不干直接调用epoll_create1了, * 另外你也可以发现, size这个参数其实是没有任何用处的... */SYSCALL_DEFINE1(epoll_create, int, size){        if (size <= 0)                return -EINVAL;        return sys_epoll_create1(0);}/* 这才是真正的epoll_create啊~~ */SYSCALL_DEFINE1(epoll_create1, int, flags){	int error;	struct eventpoll *ep = NULL;//主描述符	/* Check the EPOLL_* constant for consistency.  */	/* 这句没啥用处... */	BUILD_BUG_ON(EPOLL_CLOEXEC != O_CLOEXEC);	/* 对于epoll来讲, 目前唯一有效的FLAG就是CLOEXEC */	if (flags & ~EPOLL_CLOEXEC)		return -EINVAL;	/*	 * Create the internal data structure ("struct eventpoll").	 */	/* 分配一个struct eventpoll, 分配和初始化细节我们随后深聊~ */	error = ep_alloc(&ep);	if (error < 0)		return error;	/*	 * Creates all the items needed to setup an eventpoll file. That is,	 * a file structure and a free file descriptor.	 */	/* 这里是创建一个匿名fd, 说起来就话长了...长话短说:	 * epollfd本身并不存在一个真正的文件与之对应, 所以内核需要创建一个	 * "虚拟"的文件, 并为之分配真正的struct file结构, 而且有真正的fd.	 * 这里2个参数比较关键:	 * eventpoll_fops, fops就是file operations, 就是当你对这个文件(这里是虚拟的)进行操作(比如读)时,	 * fops里面的函数指针指向真正的操作实现, 类似C++里面虚函数和子类的概念.	 * epoll只实现了poll和release(就是close)操作, 其它文件系统操作都有VFS全权处理了.	 * ep, ep就是struct epollevent, 它会作为一个私有数据保存在struct file的private指针里面.	 * 其实说白了, 就是为了能通过fd找到struct file, 通过struct file能找到eventpoll结构.	 * 如果懂一点Linux下字符设备驱动开发, 这里应该是很好理解的,	 * 推荐阅读 <Linux device driver 3rd>	 */	error = anon_inode_getfd("[eventpoll]", &eventpoll_fops, ep,				 O_RDWR | (flags & O_CLOEXEC));	if (error < 0)		ep_free(ep);	return error;}/* * 创建好epollfd后, 接下来我们要往里面添加fd咯* 来看epoll_ctl* epfd 就是epollfd* op ADD,MOD,DEL* fd 需要监听的描述符* event 我们关心的events*/SYSCALL_DEFINE4(epoll_ctl, int, epfd, int, op, int, fd,		struct epoll_event __user *, event){	int error;	struct file *file, *tfile;	struct eventpoll *ep;	struct epitem *epi;	struct epoll_event epds;	error = -EFAULT;	/* 	 * 错误处理以及从用户空间将epoll_event结构copy到内核空间.	 */	if (ep_op_has_event(op) &&	    copy_from_user(&epds, event, sizeof(struct epoll_event)))		goto error_return;	/* Get the "struct file *" for the eventpoll file */	/* 取得struct file结构, epfd既然是真正的fd, 那么内核空间	 * 就会有与之对于的一个struct file结构	 * 这个结构在epoll_create1()中, 由函数anon_inode_getfd()分配 */	error = -EBADF;	file = fget(epfd);	if (!file)		goto error_return;	/* Get the "struct file *" for the target file */	/* 我们需要监听的fd, 它当然也有个struct file结构, 上下2个不要搞混了哦 */	tfile = fget(fd);	if (!tfile)		goto error_fput;	/* The target file descriptor must support poll */	error = -EPERM;	/* 如果监听的文件不支持poll, 那就没辙了.	 * 你知道什么情况下, 文件会不支持poll吗?	 */	if (!tfile->f_op || !tfile->f_op->poll)		goto error_tgt_fput;	/*	 * We have to check that the file structure underneath the file descriptor	 * the user passed to us _is_ an eventpoll file. And also we do not permit	 * adding an epoll file descriptor inside itself.	 */	error = -EINVAL;	/* epoll不能自己监听自己... */	if (file == tfile || !is_file_epoll(file))		goto error_tgt_fput;	/*	 * At this point it is safe to assume that the "private_data" contains	 * our own data structure.	 */	/* 取到我们的eventpoll结构, 来自与epoll_create1()中的分配 */	ep = file->private_data;	/* 接下来的操作有可能修改数据结构内容, 锁之~ */	mutex_lock(&ep->mtx);	/*	 * Try to lookup the file inside our RB tree, Since we grabbed "mtx"	 * above, we can be sure to be able to use the item looked up by	 * ep_find() till we release the mutex.	 */	/* 对于每一个监听的fd, 内核都有分配一个epitem结构,	 * 而且我们也知道, epoll是不允许重复添加fd的,	 * 所以我们首先查找该fd是不是已经存在了.	 * ep_find()其实就是RBTREE查找, 跟C++STL的map差不多一回事, O(lgn)的时间复杂度.	 */	epi = ep_find(ep, tfile, fd);	error = -EINVAL;	switch (op) {		/* 首先我们关心添加 */	case EPOLL_CTL_ADD:		if (!epi) {			/* 之前的find没有找到有效的epitem, 证明是第一次插入, 接受!			 * 这里我们可以知道, POLLERR和POLLHUP事件内核总是会关心的			 * */			epds.events |= POLLERR | POLLHUP;			/* rbtree插入, 详情见ep_insert()的分析			 * 其实我觉得这里有insert的话, 之前的find应该			 * 是可以省掉的... */			error = ep_insert(ep, &epds, tfile, fd);		} else			/* 找到了!? 重复添加! */			error = -EEXIST;		break;		/* 删除和修改操作都比较简单 */	case EPOLL_CTL_DEL:		if (epi)			error = ep_remove(ep, epi);		else			error = -ENOENT;		break;	case EPOLL_CTL_MOD:		if (epi) {			epds.events |= POLLERR | POLLHUP;			error = ep_modify(ep, epi, &epds);		} else			error = -ENOENT;		break;	}	mutex_unlock(&ep->mtx);error_tgt_fput:	fput(tfile);error_fput:	fput(file);error_return:	return error;}/* 分配一个eventpoll结构 */static int ep_alloc(struct eventpoll **pep){	int error;	struct user_struct *user;	struct eventpoll *ep;	/* 获取当前用户的一些信息, 比如是不是root啦, 最大监听fd数目啦 */	user = get_current_user();	error = -ENOMEM;	ep = kzalloc(sizeof(*ep), GFP_KERNEL);	if (unlikely(!ep))		goto free_uid;	/* 这些都是初始化啦 */	spin_lock_init(&ep->lock);	mutex_init(&ep->mtx);	init_waitqueue_head(&ep->wq);//初始化自己睡在的等待队列	init_waitqueue_head(&ep->poll_wait);//初始化	INIT_LIST_HEAD(&ep->rdllist);//初始化就绪链表	ep->rbr = RB_ROOT;	ep->ovflist = EP_UNACTIVE_PTR;	ep->user = user;	*pep = ep;	return 0;free_uid:	free_uid(user);	return error;}/* * Must be called with "mtx" held. *//*  * ep_insert()在epoll_ctl()中被调用, 完成往epollfd里面添加一个监听fd的工作 * tfile是fd在内核态的struct file结构 */static int ep_insert(struct eventpoll *ep, struct epoll_event *event,		     struct file *tfile, int fd){	int error, revents, pwake = 0;	unsigned long flags;	struct epitem *epi;	struct ep_pqueue epq;	/* 查看是否达到当前用户的最大监听数 */	if (unlikely(atomic_read(&ep->user->epoll_watches) >=		     max_user_watches))		return -ENOSPC;	/* 从著名的slab中分配一个epitem */	if (!(epi = kmem_***_alloc(epi_***, GFP_KERNEL)))		return -ENOMEM;	/* Item initialization follow here ... */	/* 这些都是相关成员的初始化... */	INIT_LIST_HEAD(&epi->rdllink);	INIT_LIST_HEAD(&epi->fllink);	INIT_LIST_HEAD(&epi->pwqlist);	epi->ep = ep;	/* 这里保存了我们需要监听的文件fd和它的file结构 */	ep_set_ffd(&epi->ffd, tfile, fd);	epi->event = *event;	epi->nwait = 0;	/* 这个指针的初值不是NULL哦... */	epi->next = EP_UNACTIVE_PTR;	/* Initialize the poll table using the queue callback */	/* 好, 我们终于要进入到poll的正题了 */	epq.epi = epi;	/* 初始化一个poll_table	 * 其实就是指定调用poll_wait(注意不是epoll_wait!!!)时的回调函数,和我们关心哪些events,	 * ep_ptable_queue_proc()就是我们的回调啦, 初值是所有event都关心 */	init_poll_funcptr(&epq.pt, ep_ptable_queue_proc);	/*	 * Attach the item to the poll hooks and get current event bits.	 * We can safely use the file* here because its usage count has	 * been increased by the caller of this function. Note that after	 * this operation completes, the poll callback can start hitting	 * the new item.	 */	/* 这一部很关键, 也比较难懂, 完全是内核的poll机制导致的...	 * 首先, f_op->poll()一般来说只是个wrapper, 它会调用真正的poll实现,	 * 拿UDP的socket来举例, 这里就是这样的调用流程: f_op->poll(), sock_poll(),	 * udp_poll(), datagram_poll(), sock_poll_wait(), 最后调用到我们上面指定的	 * ep_ptable_queue_proc()这个回调函数...(好深的调用路径...).	 * 完成这一步, 我们的epitem就跟这个socket关联起来了, 当它有状态变化时,	 * 会通过ep_poll_callback()来通知.	 * 最后, 这个函数还会查询当前的fd是不是已经有啥event已经ready了, 有的话	 * 会将event返回. */	revents = tfile->f_op->poll(tfile, &epq.pt);	/*	 * We have to check if something went wrong during the poll wait queue	 * install process. Namely an allocation for a wait queue failed due	 * high memory pressure.	 */	error = -ENOMEM;	if (epi->nwait < 0)		goto error_unregister;	/* Add the current item to the list of active epoll hook for this file */	/* 这个就是每个文件会将所有监听自己的epitem链起来 */	spin_lock(&tfile->f_lock);	list_add_tail(&epi->fllink, &tfile->f_ep_links);	spin_unlock(&tfile->f_lock);	/*	 * Add the current item to the RB tree. All RB tree operations are	 * protected by "mtx", and ep_insert() is called with "mtx" held.	 */	/* 都搞定后, 将epitem插入到对应的eventpoll中去 */	ep_rbtree_insert(ep, epi);	/* We have to drop the new item inside our item list to keep track of it */	spin_lock_irqsave(&ep->lock, flags);	/* If the file is already "ready" we drop it inside the ready list */	/* 到达这里后, 如果我们监听的fd已经有事件发生, 那就要处理一下 */	if ((revents & event->events) && !ep_is_linked(&epi->rdllink)) {		/* 将当前的epitem加入到ready list中去 */		list_add_tail(&epi->rdllink, &ep->rdllist);		/* Notify waiting tasks that events are available */		/* 谁在epoll_wait, 就唤醒它... */		if (waitqueue_active(&ep->wq))			wake_up_locked(&ep->wq);		/* 谁在epoll当前的epollfd, 也唤醒它... */		if (waitqueue_active(&ep->poll_wait))			pwake++;	}	spin_unlock_irqrestore(&ep->lock, flags);	atomic_inc(&ep->user->epoll_watches);	/* We have to call this outside the lock */	if (pwake)		ep_poll_safewake(&ep->poll_wait);	return 0;error_unregister:	ep_unregister_pollwait(ep, epi);	/*	 * We need to do this because an event could have been arrived on some	 * allocated wait queue. Note that we don't care about the ep->ovflist	 * list, since that is used/cleaned only inside a section bound by "mtx".	 * And ep_insert() is called with "mtx" held.	 */	spin_lock_irqsave(&ep->lock, flags);	if (ep_is_linked(&epi->rdllink))		list_del_init(&epi->rdllink);	spin_unlock_irqrestore(&ep->lock, flags);	kmem_***_free(epi_***, epi);	return error;}/* * This is the callback that is used to add our wait queue to the * target file wakeup lists. *//*  * 该函数在调用f_op->poll()时会被调用. * 也就是epoll主动poll某个fd时, 用来将epitem与指定的fd关联起来的. * 关联的办法就是使用等待队列(waitqueue) */static void ep_ptable_queue_proc(struct file *file, wait_queue_head_t *whead,				 poll_table *pt){	struct epitem *epi = ep_item_from_epqueue(pt);	struct eppoll_entry *pwq;	if (epi->nwait >= 0 && (pwq = kmem_***_alloc(pwq_***, GFP_KERNEL))) {		/* 初始化等待队列, 指定ep_poll_callback为唤醒时的回调函数,		 * 当我们监听的fd发生状态改变时, 也就是队列头被唤醒时,		 * 指定的回调函数将会被调用. */		init_waitqueue_func_entry(&pwq->wait, ep_poll_callback);		pwq->whead = whead;		pwq->base = epi;		/* 将刚分配的等待队列成员加入到头中, 头是由fd持有的 */		add_wait_queue(whead, &pwq->wait);		list_add_tail(&pwq->llink, &epi->pwqlist);		/* nwait记录了当前epitem加入到了多少个等待队列中,		 * 我认为这个值最大也只会是1... */		epi->nwait++;	} else {		/* We have to signal that an error occurred */		epi->nwait = -1;	}}/* * This is the callback that is passed to the wait queue wakeup * machanism. It is called by the stored file descriptors when they * have events to report. *//*  * 这个是关键性的回调函数, 当我们监听的fd发生状态改变时, 它会被调用. * 参数key被当作一个unsigned long整数使用, 携带的是events. */static int ep_poll_callback(wait_queue_t *wait, unsigned mode, int sync, void *key){	int pwake = 0;	unsigned long flags;	struct epitem *epi = ep_item_from_wait(wait);//从等待队列获取epitem.需要知道哪个进程挂载到这个设备	struct eventpoll *ep = epi->ep;//获取	spin_lock_irqsave(&ep->lock, flags);	/*	 * If the event mask does not contain any poll(2) event, we consider the	 * descriptor to be disabled. This condition is likely the effect of the	 * EPOLLONESHOT bit that disables the descriptor when an event is received,	 * until the next EPOLL_CTL_MOD will be issued.	 */	if (!(epi->event.events & ~EP_PRIVATE_BITS))		goto out_unlock;	/*	 * Check the events coming with the callback. At this stage, not	 * every device reports the events in the "key" parameter of the	 * callback. We need to be able to handle both cases here, hence the	 * test for "key" != NULL before the event match test.	 */	/* 没有我们关心的event... */	if (key && !((unsigned long) key & epi->event.events))		goto out_unlock;	/*	 * If we are trasfering events to userspace, we can hold no locks	 * (because we're accessing user memory, and because of linux f_op->poll()	 * semantics). All the events that happens during that period of time are	 * chained in ep->ovflist and requeued later on.	 */	/* 	 * 这里看起来可能有点费解, 其实干的事情比较简单:	 * 如果该callback被调用的同时, epoll_wait()已经返回了,	 * 也就是说, 此刻应用程序有可能已经在循环获取events,	 * 这种情况下, 内核将此刻发生event的epitem用一个单独的链表	 * 链起来, 不发给应用程序, 也不丢弃, 而是在下一次epoll_wait	 * 时返回给用户.	 */	if (unlikely(ep->ovflist != EP_UNACTIVE_PTR)) {		if (epi->next == EP_UNACTIVE_PTR) {			epi->next = ep->ovflist;			ep->ovflist = epi;		}		goto out_unlock;	}	/* If this file is already in the ready list we exit soon */	/* 将当前的epitem放入ready list */	if (!ep_is_linked(&epi->rdllink))		list_add_tail(&epi->rdllink, &ep->rdllist);	/*	 * Wake up ( if active ) both the eventpoll wait list and the ->poll()	 * wait list.	 */	/* 唤醒epoll_wait... */	if (waitqueue_active(&ep->wq))		wake_up_locked(&ep->wq);	/* 如果epollfd也在被poll, 那就唤醒队列里面的所有成员. */	if (waitqueue_active(&ep->poll_wait))		pwake++;out_unlock:	spin_unlock_irqrestore(&ep->lock, flags);	/* We have to call this outside the lock */	if (pwake)		ep_poll_safewake(&ep->poll_wait);	return 1;}/* * Implement the event wait interface for the eventpoll file. It is the kernel * part of the user space epoll_wait(2). */SYSCALL_DEFINE4(epoll_wait, int, epfd, struct epoll_event __user *, events,		int, maxevents, int, timeout){	int error;	struct file *file;	struct eventpoll *ep;	/* The maximum number of event must be greater than zero */	if (maxevents <= 0 || maxevents > EP_MAX_EVENTS)		return -EINVAL;	/* Verify that the area passed by the user is writeable */	/* 这个地方有必要说明一下:	 * 内核对应用程序采取的策略是"绝对不信任",	 * 所以内核跟应用程序之间的数据交互大都是copy, 不允许(也时候也是不能...)指针引用.	 * epoll_wait()需要内核返回数据给用户空间, 内存由用户程序提供,	 * 所以内核会用一些手段来验证这一段内存空间是不是有效的.	 */	if (!access_ok(VERIFY_WRITE, events, maxevents * sizeof(struct epoll_event))) {		error = -EFAULT;		goto error_return;	}	/* Get the "struct file *" for the eventpoll file */	error = -EBADF;	/* 获取epollfd的struct file, epollfd也是文件嘛 */	file = fget(epfd);	if (!file)		goto error_return;	/*	 * We have to check that the file structure underneath the fd	 * the user passed to us _is_ an eventpoll file.	 */	error = -EINVAL;	/* 检查一下它是不是一个真正的epollfd... */	if (!is_file_epoll(file))		goto error_fput;	/*	 * At this point it is safe to assume that the "private_data" contains	 * our own data structure.	 */	/* 获取eventpoll结构 */	ep = file->private_data;	/* Time to fish for events ... */	/* OK, 睡觉, 等待事件到来~~ */	error = ep_poll(ep, events, maxevents, timeout);error_fput:	fput(file);error_return:	return error;}/* 这个函数真正将执行epoll_wait的进程带入睡眠状态... */static int ep_poll(struct eventpoll *ep, struct epoll_event __user *events,		   int maxevents, long timeout){	int res, eavail;	unsigned long flags;	long jtimeout;	wait_queue_t wait;//等待队列	/*	 * Calculate the timeout by checking for the "infinite" value (-1)	 * and the overflow condition. The passed timeout is in milliseconds,	 * that why (t * HZ) / 1000.	 */	/* 计算睡觉时间, 毫秒要转换为HZ */	jtimeout = (timeout < 0 || timeout >= EP_MAX_MSTIMEO) ?		MAX_SCHEDULE_TIMEOUT : (timeout * HZ + 999) / 1000;retry:	spin_lock_irqsave(&ep->lock, flags);	res = 0;	/* 如果ready list不为空, 就不睡了, 直接干活... */	if (list_empty(&ep->rdllist)) {		/*		 * We don't have any available event to return to the caller.		 * We need to sleep here, and we will be wake up by		 * ep_poll_callback() when events will become available.		 */		/* OK, 初始化一个等待队列, 准备直接把自己挂起,		 * 注意current是一个宏, 代表当前进程 */		init_waitqueue_entry(&wait, current);//初始化等待队列,wait表示当前进程		__add_wait_queue_exclusive(&ep->wq, &wait);//挂载到ep结构的等待队列		for (;;) {			/*			 * We don't want to sleep if the ep_poll_callback() sends us			 * a wakeup in between. That's why we set the task state			 * to TASK_INTERRUPTIBLE before doing the checks.			 */			/* 将当前进程设置位睡眠, 但是可以被信号唤醒的状态,			 * 注意这个设置是"将来时", 我们此刻还没睡! */			set_current_state(TASK_INTERRUPTIBLE);			/* 如果这个时候, ready list里面有成员了,			 * 或者睡眠时间已经过了, 就直接不睡了... */			if (!list_empty(&ep->rdllist) || !jtimeout)				break;			/* 如果有信号产生, 也起床... */			if (signal_pending(current)) {				res = -EINTR;				break;			}			/* 啥事都没有,解锁, 睡觉... */			spin_unlock_irqrestore(&ep->lock, flags);			/* jtimeout这个时间后, 会被唤醒,			 * ep_poll_callback()如果此时被调用,			 * 那么我们就会直接被唤醒, 不用等时间了... 			 * 再次强调一下ep_poll_callback()的调用时机是由被监听的fd			 * 的具体实现, 比如socket或者某个设备驱动来决定的,			 * 因为等待队列头是他们持有的, epoll和当前进程			 * 只是单纯的等待...			 **/			jtimeout = schedule_timeout(jtimeout);//睡觉			spin_lock_irqsave(&ep->lock, flags);		}		__remove_wait_queue(&ep->wq, &wait);		/* OK 我们醒来了... */		set_current_state(TASK_RUNNING);	}	/* Is it worth to try to dig for events ? */	eavail = !list_empty(&ep->rdllist) || ep->ovflist != EP_UNACTIVE_PTR;	spin_unlock_irqrestore(&ep->lock, flags);	/*	 * Try to transfer events to user space. In case we get 0 events and	 * there's still timeout left over, we go trying again in search of	 * more luck.	 */	/* 如果一切正常, 有event发生, 就开始准备数据copy给用户空间了... */	if (!res && eavail &&	    !(res = ep_send_events(ep, events, maxevents)) && jtimeout)		goto retry;	return res;}/* 这个简单, 我们直奔下一个... */static int ep_send_events(struct eventpoll *ep,			  struct epoll_event __user *events, int maxevents){	struct ep_send_events_data esed;	esed.maxevents = maxevents;	esed.events = events;	return ep_scan_ready_list(ep, ep_send_events_proc, &esed);}/** * ep_scan_ready_list - Scans the ready list in a way that makes possible for *                      the scan code, to call f_op->poll(). Also allows for *                      O(NumReady) performance. * * @ep: Pointer to the epoll private data structure. * @sproc: Pointer to the scan callback. * @priv: Private opaque data passed to the @sproc callback. * * Returns: The same integer error code returned by the @sproc callback. */static int ep_scan_ready_list(struct eventpoll *ep,			      int (*sproc)(struct eventpoll *,					   struct list_head *, void *),			      void *priv){	int error, pwake = 0;	unsigned long flags;	struct epitem *epi, *nepi;	LIST_HEAD(txlist);	/*	 * We need to lock this because we could be hit by	 * eventpoll_release_file() and epoll_ctl().	 */	mutex_lock(&ep->mtx);	/*	 * Steal the ready list, and re-init the original one to the	 * empty list. Also, set ep->ovflist to NULL so that events	 * happening while looping w/out locks, are not lost. We cannot	 * have the poll callback to queue directly on ep->rdllist,	 * because we want the "sproc" callback to be able to do it	 * in a lockless way.	 */	spin_lock_irqsave(&ep->lock, flags);	/* 这一步要注意, 首先, 所有监听到events的epitem都链到rdllist上了,	 * 但是这一步之后, 所有的epitem都转移到了txlist上, 而rdllist被清空了,	 * 要注意哦, rdllist已经被清空了! */	list_splice_init(&ep->rdllist, &txlist);	/* ovflist, 在ep_poll_callback()里面我解释过, 此时此刻我们不希望	 * 有新的event加入到ready list中了, 保存后下次再处理... */	ep->ovflist = NULL;	spin_unlock_irqrestore(&ep->lock, flags);	/*	 * Now call the callback function.	 */	/* 在这个回调函数里面处理每个epitem	 * sproc 就是 ep_send_events_proc, 下面会注释到. */	error = (*sproc)(ep, &txlist, priv);	spin_lock_irqsave(&ep->lock, flags);	/*	 * During the time we spent inside the "sproc" callback, some	 * other events might have been queued by the poll callback.	 * We re-insert them inside the main ready-list here.	 */	/* 现在我们来处理ovflist, 这些epitem都是我们在传递数据给用户空间时	 * 监听到了事件. */	for (nepi = ep->ovflist; (epi = nepi) != NULL;	     nepi = epi->next, epi->next = EP_UNACTIVE_PTR) {		/*		 * We need to check if the item is already in the list.		 * During the "sproc" callback execution time, items are		 * queued into ->ovflist but the "txlist" might already		 * contain them, and the list_splice() below takes care of them.		 */		/* 将这些直接放入readylist */		if (!ep_is_linked(&epi->rdllink))			list_add_tail(&epi->rdllink, &ep->rdllist);	}	/*	 * We need to set back ep->ovflist to EP_UNACTIVE_PTR, so that after	 * releasing the lock, events will be queued in the normal way inside	 * ep->rdllist.	 */	ep->ovflist = EP_UNACTIVE_PTR;	/*	 * Quickly re-inject items left on "txlist".	 */	/* 上一次没有处理完的epitem, 重新插入到ready list */	list_splice(&txlist, &ep->rdllist);	/* ready list不为空, 直接唤醒... */	if (!list_empty(&ep->rdllist)) {		/*		 * Wake up (if active) both the eventpoll wait list and		 * the ->poll() wait list (delayed after we release the lock).		 */		if (waitqueue_active(&ep->wq))			wake_up_locked(&ep->wq);		if (waitqueue_active(&ep->poll_wait))			pwake++;	}	spin_unlock_irqrestore(&ep->lock, flags);	mutex_unlock(&ep->mtx);	/* We have to call this outside the lock */	if (pwake)		ep_poll_safewake(&ep->poll_wait);	return error;}/* 该函数作为callbakc在ep_scan_ready_list()中被调用 * head是一个链表, 包含了已经ready的epitem, * 这个不是eventpoll里面的ready list, 而是上面函数中的txlist. */static int ep_send_events_proc(struct eventpoll *ep, struct list_head *head,			       void *priv){	struct ep_send_events_data *esed = priv;	int eventcnt;	unsigned int revents;	struct epitem *epi;	struct epoll_event __user *uevent;	/*	 * We can loop without lock because we are passed a task private list.	 * Items cannot vanish during the loop because ep_scan_ready_list() is	 * holding "mtx" during this call.	 */	/* 扫描整个链表... */	for (eventcnt = 0, uevent = esed->events;	     !list_empty(head) && eventcnt < esed->maxevents;) {		/* 取出第一个成员 */		epi = list_first_entry(head, struct epitem, rdllink);		/* 然后从链表里面移除 */		list_del_init(&epi->rdllink);		/* 读取events, 		 * 注意events我们ep_poll_callback()里面已经取过一次了, 为啥还要再取?		 * 1. 我们当然希望能拿到此刻的最新数据, events是会变的~		 * 2. 不是所有的poll实现, 都通过等待队列传递了events, 有可能某些驱动压根没传		 * 必须主动去读取. */		revents = epi->ffd.file->f_op->poll(epi->ffd.file, NULL) &			epi->event.events;		if (revents) {			/* 将当前的事件和用户传入的数据都copy给用户空间,			 * 就是epoll_wait()后应用程序能读到的那一堆数据. */			if (__put_user(revents, &uevent->events) ||			    __put_user(epi->event.data, &uevent->data)) {				list_add(&epi->rdllink, head);				return eventcnt ? eventcnt : -EFAULT;			}			eventcnt++;			uevent++;			if (epi->event.events & EPOLLONESHOT)				epi->event.events &= EP_PRIVATE_BITS;			else if (!(epi->event.events & EPOLLET)) {				/* 嘿嘿, EPOLLET和非ET的区别就在这一步之差呀~				 * 如果是ET, epitem是不会再进入到readly list,				 * 除非fd再次发生了状态改变, ep_poll_callback被调用.				 * 如果是非ET, 不管你还有没有有效的事件或者数据,				 * 都会被重新插入到ready list, 再下一次epoll_wait				 * 时, 会立即返回, 并通知给用户空间. 当然如果这个				 * 被监听的fds确实没事件也没数据了, epoll_wait会返回一个0,				 * 空转一次.				 */				list_add_tail(&epi->rdllink, &ep->rdllist);			}		}	}	return eventcnt;}/* ep_free在epollfd被close时调用, * 释放一些资源而已, 比较简单 */static void ep_free(struct eventpoll *ep){	struct rb_node *rbp;	struct epitem *epi;	/* We need to release all tasks waiting for these file */	if (waitqueue_active(&ep->poll_wait))		ep_poll_safewake(&ep->poll_wait);	/*	 * We need to lock this because we could be hit by	 * eventpoll_release_file() while we're freeing the "struct eventpoll".	 * We do not need to hold "ep->mtx" here because the epoll file	 * is on the way to be removed and no one has references to it	 * anymore. The only hit might come from eventpoll_release_file() but	 * holding "epmutex" is sufficent here.	 */	mutex_lock(&epmutex);	/*	 * Walks through the whole tree by unregistering poll callbacks.	 */	for (rbp = rb_first(&ep->rbr); rbp; rbp = rb_next(rbp)) {		epi = rb_entry(rbp, struct epitem, rbn);		ep_unregister_pollwait(ep, epi);	}	/*	 * Walks through the whole tree by freeing each "struct epitem". At this	 * point we are sure no poll callbacks will be lingering around, and also by	 * holding "epmutex" we can be sure that no file cleanup code will hit	 * us during this operation. So we can avoid the lock on "ep->lock".	 */	/* 之所以在关闭epollfd之前不需要调用epoll_ctl移除已经添加的fd,	 * 是因为这里已经做了... */	while ((rbp = rb_first(&ep->rbr)) != NULL) {		epi = rb_entry(rbp, struct epitem, rbn);		ep_remove(ep, epi);	}	mutex_unlock(&epmutex);	mutex_destroy(&ep->mtx);	free_uid(ep->user);	kfree(ep);}/* File callbacks that implement the eventpoll file behaviour */static const struct file_operations eventpoll_fops = {	.release	= ep_eventpoll_release,	.poll		= ep_eventpoll_poll};/* Fast test to see if the file is an evenpoll file */static inline int is_file_epoll(struct file *f){	return f->f_op == &eventpoll_fops;}/* OK, eventpoll我认为比较重要的函数都注释完了... */
```

epoll_create

从slab缓存中创建一个eventpoll对象,并且创建一个匿名的fd跟fd对应的file对象, 而eventpoll对象保存在struct file结构的private指针中,并且返回, 该fd对应的file operations只是实现了poll跟release操作。

创建eventpoll对象的初始化操作，获取当前用户信息,是不是root,最大监听fd数目等并且保存到eventpoll对象中 初始化等待队列,初始化就绪链表,初始化红黑树的头结点。

epoll_ctl操作

将epoll_event结构拷贝到内核空间中，并且判断加入的fd是否支持poll结构(epoll,poll,selectI/O多路复用必须支持poll操作)，并且从epfd->file->privatedata获取event_poll对象,根据op区分是添加删除还是修改, 首先在eventpoll结构中的红黑树查找是否已经存在了相对应的fd,没找到就支持插入操作,否则报重复的错误，相对应的修改,删除比较简单就不啰嗦了。

插入操作时,会创建一个与fd对应的epitem结构,并且初始化相关成员,比如保存监听的fd跟file结构之类的，重要的是指定了调用poll_wait时的回调函数用于数据就绪时唤醒进程,(其内部,初始化设备的等待队列,将该进程注册到等待队列)完成这一步, 我们的epitem就跟这个socket关联起来了, 当它有状态变化时, 会通过ep_poll_callback()来通知，最后调用加入的fd的file operation->poll函数(最后会调用poll_wait操作)用于完成注册操作，最后将epitem结构添加到红黑树中。

epoll_wait操作

计算睡眠时间(如果有),判断eventpoll对象的链表是否为空,不为空那就干活不睡明.并且初始化一个等待队列,把自己挂上去,设置自己的进程状态，为可睡眠状态.判断是否有信号到来(有的话直接被中断醒来,),如果啥事都没有那就调用schedule_timeout进行睡眠，如果超时或者被唤醒,首先从自己初始化的等待队列删除,然后开始拷贝资源给用户空间了，拷贝资源则是先把就绪事件链表转移到中间链表,然后挨个遍历拷贝到用户空间, 并且挨个判断其是否为水平触发,是的话再次插入到就绪链表。

2023年往期回顾

[C/C++发展方向（强烈推荐！！）](http://mp.weixin.qq.com/s?__biz=Mzg4NDQ0OTI4Ng==&mid=2247487749&idx=1&sn=e57e6f3df526b7ad78313d9428e55b6b&chksm=cfb9586cf8ced17a8c7830e380a45ce080c2b8258e145f5898503a779840a5fcfec3e8f8fa9a&scene=21#wechat_redirect)  

[Linux内核源码分析（强烈推荐收藏！](http://mp.weixin.qq.com/s?__biz=Mzg4NDQ0OTI4Ng==&mid=2247487832&idx=1&sn=bf0468e26f353306c743c4d7523ebb07&chksm=cfb95831f8ced127ca94eb61e6508732576bb150b2fb2047664f8256b3da284d0e53e2f792dc&scene=21#wechat_redirect)[）](http://mp.weixin.qq.com/s?__biz=Mzg4NDQ0OTI4Ng==&mid=2247487832&idx=1&sn=bf0468e26f353306c743c4d7523ebb07&chksm=cfb95831f8ced127ca94eb61e6508732576bb150b2fb2047664f8256b3da284d0e53e2f792dc&scene=21#wechat_redirect)  

[从菜鸟到大师，用Qt编写出惊艳世界应用](http://mp.weixin.qq.com/s?__biz=Mzg4NDQ0OTI4Ng==&mid=2247488117&idx=1&sn=a83d661165a3840fbb23d0e62b5f303a&chksm=cfb95b1cf8ced20ae63206fe25891d9a37ffe76fd695ef55b5506c83aad387d55c4032cb7e4f&scene=21#wechat_redirect)  

[存储全栈开发：构建高效的数据存储系统](http://mp.weixin.qq.com/s?__biz=Mzg4NDQ0OTI4Ng==&mid=2247487696&idx=1&sn=b5ebe830ddb6798ac5bf6db4a8d5d075&chksm=cfb959b9f8ced0af76710c070a6db2677fb359af735e79c6378e82ead570aa1ce5350a146793&scene=21#wechat_redirect)

[突破性能瓶颈：释放DPDK带来无尽潜力](http://mp.weixin.qq.com/s?__biz=Mzg4NDQ0OTI4Ng==&mid=2247488150&idx=1&sn=2cc5ace4391d4eed060fa463fc73dd92&chksm=cfb95bfff8ced2e9589eecd66e7c6e906f28626bef6c1bce3fa19ae2d1404baa25ba3ebfedaa&scene=21#wechat_redirect)

[嵌入式音视频技术：从原理到项目应用](http://mp.weixin.qq.com/s?__biz=Mzg4NDQ0OTI4Ng==&mid=2247488135&idx=1&sn=dcc56a3af5401c338944b9b94f24c699&chksm=cfb95beef8ced2f8231c647033188be3ded88cbacf7d75feab3613f4d7f8fd8e4c8106ac88cf&scene=21#wechat_redirect)

[](http://mp.weixin.qq.com/s?__biz=MzIwNzA1OTA2OQ==&mid=2657215412&idx=1&sn=d3c64ac21056b84814db9903b656853a&chksm=8c8d62a6bbfaebb09df1f727e201c401a1663596a719fa9cb4c7859158e62ebedb5bd5f13b8f&scene=21#wechat_redirect)[C++游戏开发，基于魔兽开源后端框架](http://mp.weixin.qq.com/s?__biz=Mzg4NDQ0OTI4Ng==&mid=2247486920&idx=1&sn=82ca8a72473850e431c01e7c39d8c8c0&chksm=cfb944a1f8cecdb7051863f58d334ff3dea8afae1ecd8bc693287b2f483d2ce8f94eed565253&scene=21#wechat_redirect)

C/C++开发95

面试八股文4

大疆2

C/C++开发 · 目录

上一篇C/C++一站式就业校招指导，细分方向和项目准备

阅读 1800

​