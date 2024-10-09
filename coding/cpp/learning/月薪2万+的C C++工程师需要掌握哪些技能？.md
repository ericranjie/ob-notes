Linux开发架构之路

_2024年03月26日 20:29_ _湖南_

如果你想要拿到月薪2万+，那么你在C/C++领域需要具备什么样的专业能力呢？

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/8pECVbqIO0y1tyicyGd1jVicLvW21FwZxQnZl4gkdwV81b16KnwAqibUDicBnJCLsBmah6e5xqsSeO1IuULPdPt3OQ/640?wx_fmt=png&from=appmsg&wxfrom=13)

（一）C语言

作为一名C程序员，熟练掌握C语言是最基本的一项技能。关于如何学好C语言，以及C语言话题的讨论，网上有很多经典的文章。很多人工作一段时间以后都自认为自己的C语言水平已经很高了。可实际在工作中，接触的东西也多了，开源项目多了以后，才发现自己的C语言能力太一般了。宏函数千变万化的写法，指针百花缭乱的用法…等等。写代码时，应常常问自己：这个行为是C语言规范定义的吗？如果是，是C89还是C99？我现在用的编译器支持吗？如果不是C语言规范定义的，那么在程序运行的这个平台，行为是确定的吗？所以建议大家平时可以多想想这些问题，查查资料，相信一定会对C语言有更深的理解。

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/8pECVbqIO0y1tyicyGd1jVicLvW21FwZxQjdtR0FdUyv1JE2xGbkuE3NibFsa316ug2ia8bXdmuwmGTcCAXTf8709Q/640?wx_fmt=png&from=appmsg&wxfrom=13)

（二）UNIX/Linux系统编程

在UNIX/Linux系统上开发程序，掌握系统编程API是必不可少的技能。而这方面的经典书籍都是一些大部头的英文著作，让人望而生畏。首先可以先找一本口碑不错的中文书先看一下，了解一下都有哪些种类的API。这样以后用到时，再去精读经典英文著作里的相关章节，或是查man手册。此外，如果有时间，可以学习一下经典的开源项目，了解这些开源项目是如何使用这些API的。举个例子，Redis是很多人推荐的一个很不错的学习C语言的开源项目。在阅读代码时，会看到保存数据到文件时会用到fsync函数，再去深入地了解一下这个函数的作用。这样比单纯地去看那些著作效果要好很多。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

（三）网络编程及相关知识

关于网络方面，以下三点是必会的技能：

a）网络协议。在日常的工作中，大家接触和使用最多的无疑是TCP/IP协议族。此外，由于工作领域不同，也可能用到其它的协议。比方说，做电信相关的程序开发，平时可能接触SCTP协议会更多一些。对于这些协议，掌握最基本的知识是必须的，其它的边边角角知识可以等到用时再查。举例来说，TCP协议的“三次握手”，“四次挥手”，“TIME-WAIT状态”这些基本的知识点要弄明白，其它的边角知识学完老不用忘得也快，还是用时google一下效率更高。

b）Socket编程。Socket编程的经典书籍一点不比讲系统编程的书薄，所以可以选一本相对薄点，口碑不错的精读一下，这样基本就掌握的百分之五、六十了。另外有时间还是看一下经典的开源代码。这里还拿Redis举例，Redis里关于处理网络连接和通信的代码量不大，而且基本涵盖了所有常见的UNIX平台，看完以后一定受益匪浅。

c）协议分析工具。TCPdump、snoop（Solaris平台工具）、wireshark等这些工具不仅能帮助我们抓取数据包，还能分析数据包，这对debug网络程序有非常大的帮助。所以，我们至少要掌握这些工具最常用的功能。此外，对于开放源代码的工具，我们更是可以从代码中学到很多知识。举例来说，做短信相关的项目，可以从wireshark的分析短信协议的代码里学到很多东西，这可以帮助开发者对短信协议有了更清晰的理解。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

（四）脚本编程能力

一提到脚本编程，大家首先想到的可能就是Bash shell脚本编程了。不错，在Unix/Linux上，Bash shell也许就是用的最广泛的脚本编程语言。应用开发工程师主要用Bash shell做两个方面的工作：a）用于编写监控服务脚本；b）写一些简单的单元测试脚本，比如循环发一些命令，等等。但是Bash shell的功能远远要比这些强大。一些高手用Bash shell编程语言写出了很好玩的游戏，也有人做出了很cool的网络应用。所以建议大家有兴趣可以多了解一下Unix/Linux的这层“壳”。当然，你也可以选择学习Python、Perl、Ruby等。不过相比这些语言，Bash shell至少你不用自己去安装，而且它能做的事同样很强大。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

（五）操作系统及CPU体系结构

也许有一天，你会碰到这样的情景：你的程序在Solaris上会发生core dump，在Linux上却运行的好好的。经过一番艰苦的debug，最后得到的原因是两种操作系统对线程的调度策略不一样，这会使一个对全局变量没有加锁就访问的bug在Linux上很难出现。所以你需要尽可能地去了解你使用的操作系统，这样无论对写程序还是debug都会有很大的帮助。比如，你需要了解进程的内存布局，这样你就知道栈和堆到底在内存的哪段空间，为什么内存写越界有时会core dump，有时没事。

除了操作系统，了解CPU的体系结构也是一门必修课。比方说，SPARC CPU要求字节对齐，而X86 CPU则没有这个要求。又比如SPARC CPU是大端模式，而X86 CPU是小端模式，这就要求你对像位域这样的定义要格外小心。你还要了解你使用的CPU的汇编语言，至少能大概看懂。因为有些时候，当你从C代码中找不出bug的原因时，就需要你“透过现象看本质”，从汇编代码层面看看到底发生了什么。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

（六）编译器和调试器

“工欲善其事，必先利其器”。编译器负责把源代码生成可执行文件，而调试器则是在程序出现bug时，用来“降妖除魔”的不二神器。以大家最熟悉的gcc和gdb为例子。

gcc有很多编译选项，除了要熟悉像-O，-g这些最基本的选项，建议大家可以多了解一些其它不常见的选项。因为这些选项很可能帮助我们找到程序的一些bug。举个例子，在比较新的gcc版本里，增加了-fstack-protector这个选项，而它可以帮助我们检查到缓冲区溢出这种bug。此外，你还可能碰到这种情况，一个bug总是发生在程序优化后的版本，而不会出现在没经过优化的版本。所以，多了解你的编译器，你就可以更好地了解你的程序是如何生成的。

一个程序员不可能不碰到bug，而这个时候，调试器就是最好的工具。可以说，在遇到bug时调试技巧和手段是否丰富是衡量一个程序员的能力和水平的重要参考。除此以外，gdb另一个重要用途就是分析程序的core dump文件。程序的core dump文件好比一桩刑事案件的“犯罪现场”，而gdb则是刑侦官员用来在现场提取线索的工具。对gdb越熟悉，就越能从core dump文件提取有价值的信息，也就越有助于我们定位到程序bug的“root cause”。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

（七）DTrace/SystemTap

DTrace是由Sun的几位才华横溢的工程师开发的，最开始只支持Solaris操作系统，现在FreeBSD和Mac OS X也都支持了。Linux上类似的工具有SystemTap，也有人把DTrace移植到Linux上，不过效果似乎并不好。简单地说，DTrace可以几乎不会在对整个系统有任何性能影响下，让你了解你的程序所发生的一切。这对分析程序的热点（“Hot spot”），了解程序的执行流程，定位程序bug都有很大的帮助。有些时候，DTrace可能是你唯一的工具。举例来说，有个程序只发生在生产环境，而在实验室环境无法复现（当然，理论上任何bug都可以复现，只是我们没有找到复现条件。）。你不可能在你怀疑的代码打上断点，然后用gdb去调试。这时你只能借助于DTrace，通过它去了解程序到底是如何运行的，当时的变量值是什么。此外，DTrace还可以帮你了解操作系统的kernel，这会满足很多geek的好奇心。

如何学习这些技术呢？这里给大家推荐一个非常不错的课程（零声教育），以下的完整的学习路线，也可以作为大家查漏补缺的工具。

涵盖手写代码实现：sdpk文件系统，dpdk用户态协议栈,异步网络库zvnet，协程，io\_ uring，Nginx，bpf，线程池，内存池，连接池，原子操作, ringbuffer，定时器，死锁检测，分布式锁，日志，probuf，kafka，grpc，udp可靠传输

上线项目：KV存储项目，图床项目，即时通讯项目等

还不熟悉的朋友，这里可以先领取一份linux c/c++开发必备技术栈资料（入坑不亏）

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**网页版：**

https://www.0voice.com/uiwebsite/html/courses/v14.5.html

1.精进基石专栏

1.1 数据结构与算法

1.1.1 随处可见的红黑树

- 红黑树的应用场景进程调度cfs，内存管理

- 红黑树的数学证明与推导

- 手撕红黑树的左旋与右旋

- 红黑树添加的实现与添加三种情况的证明

- 红黑树删除的实现与删除四种情况的证明

- 红黑树的线程安全的做法

- 分析红黑树工程实用的特点

1.1.2 磁盘存储链式的B树与B+树

- 磁盘结构分析与数据存储原理

- 多叉树的运用以及B树的定义证明

- B树插入的两种分裂

- B树删除的前后借位与节点合并

- 手撕B树的插入，删除，遍历，查找

- B+树的定义与实现

- B+树叶子节点的前后指针

- B+树的应用场景与实用特点

- B+树的线程安全做法

1.1.3 海量数据去重的Hash与BloomFilter，bitmap

- hash的原理与hash函数的实现

- hash的应用场景

- 分布式hash的实现原理

- 海量数据去重布隆过滤器

- 布隆过滤的数学推导与证明

1.2 设计模式

1.2.1 创建型设计模式

- 单例模式

- 策略模式

- 观察者模式

- 工厂方法模式与抽象工厂模式

- 原型模式

1.2.2 结构型设计模式

- 适配器模式

- 代理模式

- 责任链模式

- 状态模式

- 桥接模式

- 组合模式

1.3 c++新特性

1.3.1 stI容器，智能指针，正则表达式

- unordered_map

- stl容器

- hash的用法与原理

- shared_ptr，unique_ptr

- basic_regex， sub_match

- 函数对象模板function,bind

1.3.2 新特性的线程，协程，原子操作，lamda表达式

- atomic的用法与原理

- thread_local 与condition_variable

- 异常处理exception_ptr

- 错误处理error_category

- coroutine的用法与原理

1.4 Linux工程管管理

1.4.1 Makefile/cmake/configure

- Makefile的规则与make的工作原理，

- 单文件编译与多文件编译

- Makefile的参数传递

- 多目录文件夹递归编译与嵌套执行make

- Makefile的通配符，伪目标，文件搜索

- Makefile的操作函数与特殊语法

- configure生成makefile的原则

- cmake的写法

1.4.2 分布式版本控制git

- git的工作流程

- 创建操作与基本操作

- 分支管理，查看提交历史

- git服务器搭建

1.4.3 Linux系统运行时参数命令

- 进程间通信设施状态 ipcs

- Linux系统运行时长 uptime

- CPU平均负载和磁盘活动 iostat

- 监控，收集和汇报系统活动 Sar

- 监控多处理器使用情况 mpstat

- 监控进程的内存使用情况 pmap

- 系统管理员调优和基准测量工具 nmon

- 密切关注L inux系统glances

- 查看系统调用 strace

- ftp服务器基本信息 ftptop

- 电量消耗和电源管理 powertop

- 监控mysql的线程和性能 mytop

- 系统运行参数分析 htop/ top/atop

- Linux网络统计监控工具 netstat

- 显示和修改网络接口控制器 ethtool

- 网络数据包分析利刃 tcpdump

- 远程登陆服务的标准协议 telnet

- 获取实时网络统计信息 iptraf

- 显示主机上网络接口带宽使用情况 iftop

2.高性能网络设计专栏

2.1 网络编程异步网络库 zvnet

2.1.1 网络io与io多路复用select/poll/epoll

- socket与文件描述符的关联

- 多路复用select/poll

- 代码实现LT/ET的区别

2.1.2 事件驱动reactor的原理与实现

- reactor针对业务实现的优点

- epoll封装send_cb/recv_cb/accept_cb

- reactor 多核实现

- 跨平台(select/epoll/kqueue)的封装reactor

- redis，memcached，nginx网络组件

2.1.3 http服务器的实现

- reactor sendbuffer 与recvbuffer 封装http协议

- http协议格式

- 有限状态机fsm解析http

- 其他协议websocket,tcp文件传输

2.2 网络原理

2.2.1 服务器百万并发实现(实操)

- 同步处理与异步处理的数据差异

- 网络io线程池异步处理

- ulimit的fd的百万级别支持

- sysctl. conf的rmem与wmem的调优

- conntrack的原理分析

2.2.3 Posix API与网络协议栈

- connect，listen，accept与三次握手

- listen参数backlog

- syn泛洪的解决方案

- close与四次挥手

- 11个状态迁移

- 大量close\_ wait与time\_ wait的原因与解决方案

- tcp keepalive与应用层心跳包

- 拥塞控制与滑动窗口

2.2.4 UDP的可靠传输协议QUIC

- udp的优缺点

- udp高并发的设计方案

- qq早期为什么选择udp作为通信协议

- udp可靠传输原理

- quic协议的设计原理

- quic的开源方案quiche

- kcp的设计方案与算法原理

2.3 自研框架:协程框架NtyCo的实现(已开源)

2.3.1 协程设计原理与汇编实现

- 协程存在的3个原因

- 同步与异步性能，服务端异步处理，客户端异步请求

- 协程原语switch, resume, yield

- 协程切换的三种实现方式，setjmp/longjmp，ucontext，汇编实现

- 汇编实现寄存器讲解

- 协程初始启动eip寄存器设置

- 协程栈空间定义，独立栈与共享栈的做法

- 协程结构体定义

2.3.2 协程调度器实现与性能测试

- 调度器的定义分析

- 超时集合，就绪队列，io等待集合的实现

- 协程调度的执行流程

- 协程接口实现，异步流程实现

- hook钩子的实现

- 协程实现mysql请求

- 协程多核方案分析

- 协程性能测试

2.4 自研框架：基于dpdk的用户态协议栈的实现(已开源)

2.4.1 用户态协议栈设计实现

- 用户态协议栈的存在场景与实现原理

- netmap开源框架

- eth协议，ip协议， udp协议实现

- arp协议实现

- icmp协议实现

2.4.2 应用层posix api的具体实现

- socket/bind/listen的实现

- accept实现

- recv/send的实现

- 滑动窗口/慢启动讲解

- 重传定时器，坚持定时器，time_wait定时器，keepalive定时器

2.4.3 手把手设计实现epoll

- epoll数据结构封装与线程安全实现

- 协议栈fd就绪回调实现

- epoll接口实现

- LT/ET的实现

2.5 高性能异步io机制 io_uring

2.5.1 与epoll媲美的io_uring

- io_uring系统调用io_uring_setup，io_uring_register, io_ur ing_enter

- I iburng的io_uring的关系

- io_uring与epoll性能对比

- io_uring的共享内存机制

2.5.2 io\_ uring的使用场景

- io_ur ing的accept，connect，recv，send实现机制

- io_uring网络读写

- io_uring磁盘读写

- proactor的实现

3.基础组件设计专栏

3.1 池式组件

3.1.1 手写线程池与性能分析(项目)

- 线程池的异步处理使用场景

- 线程池的组成任务队列执行队列

- 任务回调与条件等待

- 线程池的动态防缩

- 扩展：nginx线程池实现对比分析

3.1.2 内存池的实现与场景分析(项目)

- 内存池的应用场景与性能分析

- 内存小块分配与管理

- 内存大块分配与管理

- 手写内存池，结构体封装与API实现

- 避免内存泄漏的两种万能方法

- 定位内存泄漏的3种工具

- 扩展：nginx内存池实现

3.2 高性能组件

3.2.1 原子操作CAS与锁实现(项目)

- 互斥锁的使用场景与原理

- 自旋锁的性能分析

- 原子操作的汇编实现

3.2.2 无锁消息队列实现(项目)

- 有锁无锁队列性能

- 内存屏障Barrier

- 数组无锁队列设计实现

- 链表无锁队列设计实现

3.2.3 网络缓冲区设计

- RingBuffer设计

- 定长消息包

- ChainBuffer 设计

- 双缓冲区设计

3.2.4 定时器方案红黑树，时间轮，最小堆(项目)

- 定时器的使用场景

- 定时器的红黑树存储

- 时间轮的实现

- 最小堆的实现

- 分布式定时器的实现

3.2.5 手写死锁检测组件(项目)

- 死锁的现象以及原理

- pthread_mutex_lock/pthread_mutex_unlock dlsym的 实现

- 有向图的构建

- 有向图dfs判断环的存在

- 三个原语操作lock_before，lock_after， unlock_after

- 死锁检测线程的实现

3.2.6 手写内存泄漏检测组件(项目)

- 内存泄漏现象

- 第三方内存泄漏与代码内存泄漏

- malloc与free的dlsym实现

- 内存检测策略

- 应用场景测试

3.2.7手把手实现分布式锁(项目)

- 多线程资源竞争互斥锁，自旋锁

- 加锁的异常情况

- 非公平锁的实现

- 公平锁的实现

3.3 开源组件

3.3.1 异步日志方案spdlog (项目)

- 日志库性能瓶颈分析

- 异步日志库设计与实现

- 批量写入与双缓存冲机制

- 奔溃后的日志找回

3.3.2 应用层协议设计ProtoBuf (项目)

- IM，云平台，nginx， http， redis协议设计

- 如何保证消息完整性

- 手撕protobuf IM通 信协议

- protobuf序列化与反序列化

- protobuf编码原理  4.中间件开发专栏

4.1 Redis

4.1.1 Redis相关命令详解及其原理

- string，set , zset，list， hash

- 分布式锁的实现

- lua脚本解决ACID原子性

- Redis事务的ACID性质分析

4.1.2 Redis协议与异步方式

- Redis协议解析

- 特殊协议操作订阅发布

- 手撕异步redis协议

4.1.3 存储原理与数据模型

- string的三种编码方式 int, raw, embstr

- 双向链表的list实现

- 字典的实现，hash函数

- 解决 键冲突与 rehash

- 跳表的实现 与数据论证

- 整数集合实现

- 压缩列表原理证明

4.1.4 主从同步与对象模型

- 对象的类型与编码

- 字符串对象

- 列表对象

- 哈希对象

- 集合对象

- 有序集合

- 类型检测与命令多态

- 内存回收

- 对象共享

- 对象空转时长

- redis的3种集群方式 主从复制，sentinel，cluster

- 4种持久化方案

4.2 MySQL

4.2.1 SQL语句， 索引，视图，存储过程， 触发器

- MySQL体系结构，SQL执行流程

- SQL CURD与高级查询

- 视图，触发器，存储过程

- MySQL权限管理

4.2.2 MySQL索引原理以及SQL优化

- 索引，约束以及之间的区别

- B+树，聚集索引和辅助索引

- 最左匹配原则以及覆盖索引

- 索引失效以及索引优化原则

- EXPLAIN执行计划以及优化选择过程分析

4.2.3 MySQL事务原理分析

- 事务的ACID特性

- MySQL并发问题脏读，不可重复读，幻读

- 事务隔离级别

- 锁的类型，锁算法实现以及锁操作对象

- S锁 X锁 IS锁 IX锁

- 记录锁，间隙锁， next-key lock

- 插入意向锁，自增锁

- MVCC原理剖析

4.2.4 MySQL缓存策略

- 读写分离，连接池的场景以及其局限a

- 缓存策略问题分析

- 缓存策略强一致性解决方案

- 缓存策略最终一致性解决方案

- 2种mysql缓存同步方案从数据库与触发器+udf

- 缓存同步开源方案go-mysql-transfer

- 缓存同步开源方案canal原理分析

- 3种缓存故障，缓存击穿，缓存穿透，缓存雪崩

4.3 Kafka

4.3.1 Kafka使用场景与设计原理

- 发布订阅模式

- 点对点消息传递

- Kafka Brokers原理

- Topics 和 Partition

4.3.2Kafka存储机制

- Partition存储分布

- Partition文件存储机制

- Segment文件存储结构

- offset查找message

- 高效文件存储设计

4.4 微服务之间通信基石gRPC

4.4.1 gRPC的内部组件关联

- ClientSide与ServerSide， Channel，Serivce，Stub的概念

- 异步gRPC的实现

- 回调方式的异步调用

- Server 与Client 对RPC的实现

4.4.2 基于http2的gRPC通信协议

- 基于http协议构造

- ABNF语法

- 请求协议 Request-Headers

- gRPC上下文传递

4.5 Nginx

4.5.1 Nginx反 向代理与系统参数配置conf原理

- Nginx静态文件的配置

- Nginx动态接口代理配置

- Nginx 对Mqtt协议转发

- Nginx对Rtmp推拉流

- Openresty对Redis缓 存数据代理

- shmem的 三种实现方式

- 原子操作

- nginx channel

- 信号

- 信号量

4.5.2 Nginx过滤器模块实现

- Nginx Filter模块运行原理

- 过滤链表的顺序

- 模块开发数据结构ngx_str_t，ngx_list_t，ngx_buf_t， ngx_chain_t

- error日志的用法

- ngx_comond_t的讲解

- ngx_http_module_t的执行流程

- 文件锁，互斥锁

- slab共享内存

- 如何解决"惊群"问题

- 如何实现负载均衡

4.5.3 Nginx Handler模 块实现

- Nginx Handler模块运行原理

- ngx_module_t/ngx_http_module_t的讲解

- ngx_http_top_body_filter/ngx_http_top_header_filter的 原理

- ngx_rbtree_t的使用方法

- ngx_rbtree自定义添加方法

- Nginx的核心数据结构ngx_cycle_t，ngx_event_moule_t

- http请求的11个处理阶段

- http包体处理

- http响应发送

- Nginx Upstream机制的设计与实现

- 模块性能测试

5.开源框架专栏

5.1 游戏服务器开发skynet (录播答疑)

5.1.1 Skynet设计原理

- 多核并发编程-多线程， 多进程，csp模型， actor模型

- actor模型实现-lua服务和c服务

- 消息队列实现

- actor消息调度

5.1.2 skynet网络层封装以及lua/c接口编程

- skynet reactor 网络模型封装

- socket/socketchannel 封装

- 手撕高性能c服务

- lua编程以及lua/c接口编程

5.1.3 skynet重要组件以及手撕游戏项目

- 基础接口 skynet.send , skynet.call , skynet.response

- 广播组件 multicastd

- 数据共享组件sharedatad datasheet

- 手撕万人同时在线游戏

5.2 分布式API网关

5.2.1 高性能web网关Openresty

- Nginx与lua模块

- Openresty访问Redis，MySQL

- Restful API接口开发

- Openresty性能分析

5.2.2 Kong动态负载均衡与服务发现

- nginx，openresty , Kong之 间的“苟且”

- 动态负载均衡的原理

- 服务发现实现的原理

- Serverless

- 监控，故障检测与恢复

- 代理层缓存与响应服务

- 系统日志

5.3 SPDK助力MySQL数据落盘，让性能腾飞( 基础设施)

5.3.1 SPDK文件系统设计与实现

- NVMe与PCle的原理

- NVMe Controller 与bdev之间的rpc

- blobstore与blob的关系

5.3.2 文件系统的posix api实现

- 4层结构设计vfs

- spdk的异步改造posix同步api

- open/write/read/close的实现

5.3.3 文件系统的性能测试与承接mysq|业务

- LD_PRELOAD更好mysql系统调用实现

- iodepth讲解

- 随机读，随机写，顺序读，顺序写

5.4 高性能计算CUDA (录播答疑)

5.4.1 gpu并行计算cuda的开发流程

- cpu+gpu的异构计算

- 计算机体系结构中的gpu

- cuda的环境搭建nvcc与srun的使用

- cuda的向量加法与矩阵乘法

- MPI与CUDA

5.4.2 音视频编解码中的并行计算

- cuda的h264编解码

- cuda的mpeg编解码

- ffmpeg的cuda支持

5.5 并行计算与异步网络引擎workflow

5.5.1 workflow的应用场景

- workflow的编程范式与设计理念

- mysql/redi s/kafka/dns的请求实现

- parallel处理与任务组装

5.5.2 workflow的组件实现

- 线程池实现

- DAG图任务

- msgqueue的实现

- 纯c的jsonparser实现

5.6 物联网通信协议mqtt的实现框架mosquitto

5.6.1 mqtt的高效使用场景

- mqtt的发布订阅模式

- 解决低带宽网络环境的数据传输

- 3种Qos等级

- 0Auth与JWT的安全认证

5.6.2 mqtt的broker

- mqtt的遗嘱机制

- 发布订阅的过滤器

- mosquitto的docker部署

- mqtt的日志实时监控

6. 云原生专栏

6.1 Docker

6.1.1. Docker风光下的内核功能(录播答疑)

- 进程 namespace

- UTS namespace

- IPC namespace

- 网络namespace

- 文件系统namesapce

- cgroup的资源控制

6. 1.2. Docker容器管理与镜像操作(录播答疑)

- Docker 镜像下载与镜像运行

- Docker 存储管理

- Docker 数据卷

- Docker 与容器安全

6.1.3. Docker网络管理(项目)

- 5种Docker网络驱动

- pipework跨主机通信

- 0vS划分vlan与隧道模式

- GRE实现跨主机Docker间通信

6.1.4. Docker云与容器编排(项目)

- Dockerfile的语法流程

- 编排神器Fig/Compose

- Flynn体系架构

- Docker改变了什么?

6.2. Kubernetes

6.2.1 k8s环境搭建(录播答疑)

- k8s集群安全设置

- k8s集群网络设置

- k8s核心服务配置

- kubectl命令工具

- yaml文件语法

6.2.2 Pod与Service的用法(录播答疑)

- Pod的管理配置

- Pod升级与回滚

- DNS服务之于k8s

- http 7层策略与TLS安全设置

6.2.3 k8s集群管理的那些事儿(项目)

- Node的管理

- namespace隔离机制

- k8s集群日志管理

- k8s集群监控

6.2.4 k8s二次开发与k8s API(项目)

- RESTful接口

- API聚合机制

- API组

- Go访问k8s API

7.性能分析专栏

7.1 性能与测试工具

7.1.1 测试框架gtest以及内存泄漏检测(录播答疑)

- googletest与googlemock文件

- 函数检测以及类测试

- test fixture测试夹具

- 类型参数化

- 事件测试

- 内存泄漏

- 设置期望，期待参数，调用次数，满足期望

7.1.2 性能工具与性能分析(录播答疑)

- MySQL性能测试工具mysqlslap

- Redis性能测试工具redis-benchmark

- http性能测试工具wrk

- Tcp性能测试工具TCPBenchmarks

- 磁盘，内存，网络性能分析

7.1.3 火焰图的生成原理与构建方式

- 火焰图工具讲解

- 火焰图使用场景与原理

- nginx 动态火焰图

- MySQL 火焰图

- Redis 火焰图

7.2 观测技术bpf与ebpf

7.2.1 内核bpf的实现原理

- 跟踪，嗅探，采样，可观测的理解

- 动态hook: kprobe/ uprobe

- 静态hook: tracepoint和USDT

- 性能监控计时器PMC模式

- cpu的观测taskset的使 用

- BPF工具bpftrace， BCC

7.2.2 bpf对内核功能的观测

- 内存观测 kmalloc与vm_area_struct

- 文件系统观测vfs的状态

- 磁盘io的观测bitesize，mdflush

- bpf对网络流量的统计

- bpf对redis-server观测

- 网络观测tcp_connect， tcp_accept, tcp_close

7.3 内核源码机制

7.3.1 进程调度机制哪些事儿

- qemu调试内存

- 进程调度cfs与其他的四个调度类

- task_struct结构体

- RCU机制与内存优化屏障

7.3.2 内核内存管理运行机制

- 虚拟内存地址布局

- SMP/ NUMA模型

- 页表与页表缓存原理

- 伙伴系统实现

- 块分配(Slab/Slub/Slob) 原理实现

- brk/kmalloc/vmalloc系统调用流程

7.3.3 文件系统组件

- 虚拟文件系统vfs

- Proc文件系统

- super_block与inode结构体

- 文件描述符与挂载流程

8. 分布式架构专栏

8.1 分布式数据库

8.1.1 不一样的kv存储RocksDB的使用场景

- 前缀搜索

- 低优先级写入

- 生存时间的支持

- Transact ions

- 快照存储

- 日志结构的数据库引擎

8.2.1 TiDB存储引擎的原理

- TiKV的Key-Value存储引擎

- 基于RBAC的权限管理

- 数据加密

8.2.2 TiDB集群方案与Replication原理

- 集群三个组件TiDB Server， PD Server, TiKV Server

- Raft协议讲解

- OLTP与0LAP

8.2分布式文件系统(录播答疑)

8.2.1 内核级支持的分布式存储Ceph

- ceph的集群部署

- monitor与0SD

- ceph5个核心组件

- ceph集群监控

- ceph性能调调优与benchmark

8.2.2 分布式ceph存储集群部署

- 同步机制

- 线性扩容

- 如何实现高可用

- 负载均衡

8.3 分布式协同

8.3.1 注册服务中心Etcd

- etcd配置服务、服务发现、集群监控、leader选举、分布式锁

- etcd体系结构详解(gRPC，WAL，Snapshot、 BoItDB、 Raft)

- etcd存储原理深入剖析(B树、 B+树)

- etcd读写机制以及事务的acid特性分析

- raft共识算法详解(leader选举+日志复制)

8.3.2 协同事件用户态文件系统fuse (项目)

- fuse的使用场景

- 文件系统读写事件

- fuse的实现原理

- /dev/fuse的作用

8.3.3快播核心技术揭秘P2P框架的实现(录播答疑)

- 网关NAT表分析

- NAT类型，完全锥型NAT，对称NAT，端口限制锥形NAT，IP限制锥型NAT

- 代码逻辑实现NAT类型检测

- 网络穿透的原理

- 网络穿透的3种情况

**了解课程详情，领取大额优惠扫描下方二维码**

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

9. 上线项目实战

9.1 dkvstore实现(上线项目)

9.1.1 kv存储的架构设计

- 存储节点定义

- tcp server/client

- hash数据存储

- list数据存储

- skiptable数据存储

- rbtree数据存储

9.1.2 网络同步与事务序列化

- 序列化与反序列化格式

- 建立事务与释放事务

- 线程安全的处理

9.1.3 内存池的使用与LRU的实现

- 大块与小块分配策略

- 内存回收机制

- 数据持久化

9.1.4 KV存储的性能测试

- 网络测试tps

- 吞吐量测试

- go, lua, java多语言支持

- hash/list/skiptable/rbtree测试

9.2 图床共享云存储(上线项目)

9.2.1 ceph架构分析和配置

- ceph架构分析

- 快速配置ceph

- 上传文件逻辑分析

- 下载文件逻辑分析

9.2.2 文件传输和接口设计

- http接口设计

- 图床数据库设计

- 图床文件上传，下载，分享功能实现

- 业务流程实现

9.2.3 容器化docker部署

- crontab定时清理数据

- docker server服务

- grpc连接池管理

9.2.4 产品上云公网发布/测试用例

- 使用云服务器的各种坑分析

- fiddler 监控http请求，postman模拟请求

- wrk测试接口吞吐量

- jmeter压力测试

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

9.3 微服务即时通讯(上线项目)

9.3.1 IM即时通讯项目框架分析和部署

- 即时通讯应用场景分析

- 即时通讯自研和使用第三方SDK优缺点

- 即时通讯数据库设计

- 接入层、 逻辑层、路由层、数据层架构.

- 即时通讯项目部署

- 即时通讯web账号注册源码分析

9.3.2 IM消息服务器/文件传输服务器

- protobuf通信协议设计

- reactor模型C++实现

- login\_ server 负载均衡手写代码实现

- 用户登录请求验证密码+混淆码MD5匹对

- 如何全量、增量拉取好友列表、用户信息

- 知乎、b站小红点点未读消息如何实现

9.3.3 IM消息服务器和路由服务器设计

- 请求登录逻辑

- 最近联系会话逻辑

- 查询用户在线主题

- 未读消息机制

- 单聊消息推拉机制

- 群聊消息推拉机制

- 路由转发机制

9.3.4 数据库代理服务器设计

- main函数主流程

- reactor+线程池+连接池处理逻辑分

- redis缓存实现消息计数(单聊和群聊)

- redis实现未读消息机制

- 如何实现群消息的推送

- 单聊消息推送、拉取优缺点

9.3.5 文件服务器和docker部署

- 在线文件传输机制分析

- 离线文件传输机制分析

- etcd微服务注册与发现

- docker制作与部署

9.3.6 产品上云公网发布/公网测试上线

- 单元测试案例

- testbench如何设计

- IM项目性能压测

- 定制私有功能

- 拓展新功能(代码)

- 云服务器部署

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

9.4 零声教学AI助手一代(上线项目)

9.4.1 AI助手架构设计与需求分析

- chatgpt的构想 与需求分析

- 基于开源项目初步构建项目

- gin框架实现代理服务

9.4.2 接口功能设计

- grpc与protobuf的使用流程

- token计数器与tokenizer的服务封装

- 敏感词识别服务

9.4.3 向量数据库与连接池设计

- redis实现上下文管理

- 问题记录保存

- web端协议解析

- OneBot协议

9.4.4 服务部署上线

- docker stack 服务部署

- wrk接口吞吐量测试

- 线上节点监控

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

9.5 魔兽世界后端TrinityCore (上线项目)

9.5.1 网络模块实现

- boost. asio 跨平台网络库

- boost. asio 核心命名空间以及异步io接口

- boost. asio 在 TrinityCore 中的封装

- 网络模块应用实践

9.5.2 地图模块实现

- 地图模块抽象: map、 area、 grid、 cell

- 地图模块驱动方式

- A0I 核心算法实现

- AABB碰撞检测实现

- A\*寻路算法实现

9.5.3战斗模块实现

- 技能设计以及实现

- AI设计

- 怪物管理

- 副本设计

9.5.4 TrinityCore 玩法实现

- 用户玩法实现-任务系统

- 数据配置以及数据库设计

- 触发机制实现

- 多人玩法实现-工会设计

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

阅读 1331

​
