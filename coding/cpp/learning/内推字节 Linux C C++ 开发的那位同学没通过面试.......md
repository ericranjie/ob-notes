
原创 张小方 CppGuide

 _2021年03月24日 12:00_

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/ic8RqseyjxMPRffmX1RfSpu2FgLzUqZ4k37EraaviaT1IianXjxNpbRlLqaUu376n5iaOqZicEXklbPnYIt1RqQBlmg/640?wx_fmt=jpeg&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

##   

## 最近同事内推了一位 Linux C/C++ 后端开发的同学到我们公司面试，我是一面的面试官，很遗憾这位工作了两年的同学面试表现不是很好。  

我问了如下一些问题：

> “Redis 持久化机制，redis 销毁方式机制，MQ 实现原理，C++ 虚函数，hash 冲突的解决，memcached 一致性哈希，socket 函数、select/poll/epoll模型，同步互斥，异步非阻塞，回调的概念，innodb索引原理，单向图最短路径，动态规划算法。”

为了帮助更多的同学拿到满意的 offer，我整理了一下发出来，那么 Linux C/C++ 岗位一般会问哪些知识点呢？

## C++ 面试一般考察的内容

### 1. 语言基础

**C++ 虚函数**这是面试初、中级 C++ 职位一个概率 95% 以上的面试题。一般有以下几种问法：

1.在有继承关系的父子类中，构建和析构一个子类对象时，父子构造函数和析构函数的执行顺序分别是怎样的？

2.在有继承关系的类体系中，父类的构造函数和析构函数一定要申明为 virtual 吗？如果不申明为 virtual 会怎样？

3.什么是 C++ 多态？C++ 多态的实现原理是什么？

4.什么是虚函数？虚函数的实现原理是什么？

5.什么是虚表？虚表的内存结构布局如何？虚表的第一项（或第二项）是什么？

6.菱形继承（类 D 同时继承 B 和 C，B 和 C又继承自A）体系下，虚表在各个类中的布局如何？如果类B和类C同时有一个成员变了m，m如何在D对象的内存地址上分布的？是否会相互覆盖？

另外，时至今日，你一定要熟悉 C++11/14/17 常用的语言特性和类库，这里简单地列一下：

- 统一的类成员初始化语法与 std::initializer_list
    
- 注解标签（attributes）
    
- final/override/=default/=delete 语法
    
- auto 关键字
    
- Range-based 循环语法
    
- 结构化绑定
    
- stl 容器新增的实用方法
    
- std::thread
    
- 线程局部存储 thread_local
    
- 线程同步原语 std::mutex、std::condition_variable 等
    
- 原子操作类
    
- 智能指针类
    
- std::bind/std::function
    

C++11/14 网上的资料已经很多了，C++17 的资料不多，重头戏还是 C++11 引入的各种实用特性，这就给读者推荐一本我读过的：

- **《深入理解 C++11：C++11 新特性解析与应用》**
    
- **《深入应用 C++11：代码优化与工程级应用》**
    
- **《C++17 完全指南》**
    
- **《Cpp 17 in Detail》**
    

这里网络上也有人分享出来，下载链接：

> 链接: https://pan.baidu.com/s/1Af_6-bkugcTFljpIoeIk7Q 密码: mltg

建议购买正版哦。

C++11/14/17 的语法虽然很实用，但是需要一定的练习才能掌握，推荐几个学习 C++11/14/17 的开源项目：

### 1. filezilla

filezilla 是一款开源的 FTP 软件，其源码下载地址如下：

> https://svn.filezilla-project.org/svn/FileZilla3/trunk

需要使用 svn 工具来下载，安装好 svn 工具后，在 svn 界面中 checkout 上述地址或者使用如下命令下载：

> svn co https://svn.filezilla-project.org/svn/FileZilla3/trunk filezilla

如果使用 svn 图形化工具，直接使用以下 svn 地址将源码 checkout 到指定目录即可：

> https://svn.filezilla-project.org/svn/FileZilla3/trunk

如果你不知道怎么下载，可以直接在“**高性能服务器开发**”公众号回复“**获取Filezilla源码**”进行下载。

### 2. uWebSocket 网络库

uWebSocket 是一款开源的 WebSocket 库，最新版使用了大量 C++17 的语法，美中不足的是这个库代码存在不少 bug，我在项目中使用了它，但修改了其大量的 bug，有兴趣的朋友也可以下载下来看一下：

下载地址：

> https://github.com/uNetworking/uWebSockets

### 3. TeamTalk 的 PC 端

TeamTalk 是蘑菇街开源的一款用于企业内部的即时通信工具，其下载地址是：

> https://github.com/balloonwj/TeamTalk/tree/master/win-client

### 4. 最后是我的开源 Flamingo IM

> https://github.com/balloonwj/flamingo balloonwj/flamingo

## 2. 算法与数据结构基础

说到算法和数据结构，对于社招人士和对于应届生一般是不一样的，对于大的互联网公司和一般的小的企业也是不一样的。下面根据我当面试官面试别人和找工作被别人面试经验来谈一谈。

先说考察的内容，除了一些特殊的岗位，常见的算法和数据结构面试问题有如下：

1.排序（常考的排序按频率考排序为：快速排序 > 冒泡排序 > 归并排序 > 桶排序）

一般对于对算法基础有要求的公司，如果你是应届生或者工作经验在一至三年内，以上算法如果写不出来，给面试官的影响会非常不好，甚至直接被 pass 掉。对于工作三年以上的社会人士，如果写不出来，但是能分析出其算法复杂度、最好和最坏的情况下的复杂度，说出算法大致原理，在多数面试官面前也可以过的。注意，如果你是学生，写不出来或者写的不对，基本上面试过不了。

1.二分查找

二分查找的算法尽量要求写出来。当然，大多数面试官并不会直接问你二分查找，而是结合具体的场景，例如如何求一个数的平方根，这个时候你要能想到是二分查找。我在2017年年底，面试agora时，面试官问了一个问题：如何从所有很多的ip地址中快速找个某个ip地址。

2.链表

无论是应届生还是工作年限不长的社会人士，琏表常见的操作一定要熟练写出来，如链表的查找、定位、反转、连接等等。还有一些经典的问题也经常被问到，如两个链表如何判断有环（我在2017年面试饿了么二面、上海黄金交易所一面被问过）。链表的问题一般不难，但是链表的问题存在非常多的“坑”，如很多人不注意边界检查、空链表、返回一个链表的函数应该返回链表的头指针等等。

3.队列与栈

对于应届生来说一般这一类问的比较少，但是对于社会人士尤其是中高级岗位开发，会结合相关的问题问的比较多，例如让面试者利用队列写一个多线程下的生产者和消费者程序，全面考察的多线程的资源同步与竞态问题（下文介绍多线程面试题时详细地介绍）。栈一般对于基础要求高的面试，会结合函数调用实现来问。即函数如何实现的，包括函数的调用的几种常见调用方式、参数的入栈顺序、内存栈在地址从高向低扩展、栈帧指针和栈顶指针的位置、函数内局部变量在栈中的内存分布、函数调用结束后，调用者和被调用者谁和如何清理栈等等。某年面试京东一基础部门，面试官让写从0加到100这样一个求和算法，然后写其汇编代码。

4.哈希表

哈希表是考察最多的数据结构之一。常见的问题有哈希冲突的检测、让面试者写一个哈希插入函数等等。基本上一场面试下来不考察红黑树基本上就会问哈希表，而且问题可浅可深。我印象比较深刻的是，当年面试百度广告推荐部门时，二面问的一些关于哈希表的问题。当时面试官时先问的链表，接着问的哈希冲突的解决方案，后来让写一个哈希插入算法，这里需要注意的是，你的算法中插入的元素一定要是通用元素，所以对于 C++ 或者 Java 语言，一定要使用模板这一类参数作为哈希插入算法的对象。然后，就是哈希表中多个元素冲突时，某个位置的元素使用链表往后穿成一串的方案。最终考察 linux 下 malloc（下面的ptmalloc） 函数在频繁调用造成的内存碎片问题，以及开源方案解决方案 tcmalloc 和 jemalloc。总体下来，面试官是一步步引导你深入。（有兴趣的读者可以自行搜索，网上有很多相关资料）

5.树

面试高频的树是红黑树，也有一部分是B树（B+树）。红黑树一般的问的深浅不一，大多数面试官只要能说出红黑树的概念、左旋右旋的方式、分析出查找和插入的平均算法复杂度和最好最坏时的算法复杂度，并不要写面试者写出具体代码实现。一般 C++ 面试问 stl 的map，java 面试问 TreeMap 基本上就等于开始问你红黑树了，要有心里准备。笔者曾经面试爱奇艺被问过红黑树。B树一般不会直接问，问的最多的形式是通过问 MySQL 索引实现原理（数据库知识点将在下文中讨论）。笔者面试腾讯看点部门二面被问到过。

6.图

图的问题就我个人面试从来没遇到过，不过据我某位哥哥所说，他在进三星电子之前有一道面试题就是深度优先和广度优先问题。

7.其他的一些算法

如 A*寻路、霍夫曼编码也偶尔会在某一个领域的公司的面试中被问到。

## 3. 编码基本功

还有一类面试题不好分类，笔者姑且将其当作是考察编码基本功，这类问题既可以考察算法也可以考察你写代码基本素养，这些素养不仅包括编码风格、计算机英语水平、调试能力等，还包括你对细节的掌握和易错点理解，如有意识地对边界条件的检查和非法值的过滤。请读者看以下的代码执行结果是什么？

`for(char i = 0; i < 256; ++i)    {       printf("%d\n", i);   }   `

下面再列举几个常见的编码题：

（1）实现一个 memmov 函数 这个题目考查点在于 memmov 函数与 memcpy 函数的区别，这两者对于源地址与目标地址内存有重叠的这一情况的处理方式是不一样的。

（2）实现 strcpy 或 strncpy 函数，这类函数写出来没啥难度，但是**一定要注意边界条件检查**，还有就是**其返回值**一定要是目标内存地址，以支持所谓的链式拷贝。**这两类细节没考虑到基本上算不通过。**

即：

`strcpy(dest3, strcpy(dest2, strcpy(dest1, src1)));   `

（3）实现atoi函数 这个函数的签名如下：

`int atoi(const char* p);   `

容易疏忽的地方有如下几点：

- 小数点问题，如数字 0.123 和 .123 都是合法的；
    
- 正负号问题，如 +123 和 -123；
    
- 考虑如何识别第一个非法字符问题，如 123Z89，则应转换成应该 123。
    

## 4. 多线程开发基础

现如今的多核 CPU 早已经是司空见惯，而多线程编程早已经是“飞入寻常百姓家”。对于大多数桌面应用（与 Web 开发相对），尤其是像后台开发这样的岗位，且**面试者是社会人员**（有一定的工作经验），如果面试者不熟悉多线程编程，那么一般会被直接 pass 掉。

这里说的 **“熟悉多线程编程”** 到底熟悉到什么程度呢？一般包括：知道何种场合下需要新建新的线程、线程如何创建和等待、线程与进程的关系、线程局部存储（TLS 或者叫 thread local）、多线程访问资源产生竞态的原因和解决方案等等、熟练使用所在操作系统平台提供的线程同步的各种原语。

对于 C++ 开发者，你需要：

- 对于 Windows 开发者，你需要熟练使用 Interlock系列函数、CriticalSection、Event、Mutex、Semphore等API 函数和两个重要的函数 WaitForSingleObject、WaitForMultipleObjects。
    
- 对于 Linux 开发者，你需要熟练使用 mutex、semphore、condition_variable、read-write-lock 等操作系统API。
    
- 可以使用 C++ 实现一个简单的线程池，当然支持优先级、动态创建线程功能就更好了。
    

## 5. 数据库

数据库知识一般在大的互联网企业对应届生不做硬性要求，对于小的互联网企业或社会人士一般有一定的要求。其要求一般包括：

（1）熟悉基本 SQL 操作 包括增删改查（insert、delete、update、select语句），排序 order，条件查询（where 子语句），限制查询结果数量（LIMIT语句）等

（2）稍微高级一点的 SQL 操作（如 Group by，in，join，left join，多表联合查询，别名的使用，select 子语句等）

（3）索引的概念、索引的原理、索引的创建技巧

（4）数据库本身的操作，建库建表，数据的导入导出

（5）数据库用户权限控制（权限机制）

（6）MySQL的两种数据库引擎的区别

（7）SQL 优化技巧

以上属于对开发的基本的数据库知识要求，你可以找一本相关入门级的数据库图书学习即可。

高级开发除了以上要求还要熟悉高可用 **MySQL、主从同步、读写分离、分表分库** 等技术，这些技术的细节一定要清楚，它们是你成为技术专家或者高级架构的必备知识。

**我们在实际面试时，在讨论高可用服务服务方案时，很多面试者也会和我们讨论到这些技术，但是不少面试者只知道这些技术的大致思想，细节往往说不清楚，细节不会就意味着你的高可用方案无法落地，企业需要可以落地的方案。**

这些技术我首推 **《高性能 MySQL》** 这本书，这本书高级开发者一定要通读的，另外还有 2 本非常好的图书也推荐一下：一本是 **《MySQL 排错指南》，** 读完这本书以后，你会对整个“数据库世界”充满了清晰的认识；另外一本是 **《数据库索引设计与优化》，** 这本书读起来非常舒服，尤其是对于喜欢算法和数据结构的同学来说。

网上也有同学整理分享出来，下载链接（喜欢记得买正版哦）：

> 链接: https://pan.baidu.com/s/1mCTqrOXkpsRbEpPK-lmrVg  密码: foqu

## 6. 网络编程

网络编程这一块，对于应届生或者初级岗位一般只会问一些基础网络通信原理（如三次握手和四次挥手）的 socket  基础 API 的使用，客户端与服务器端网络通信的流程（回答 【客户端创建 socket -> 连接 server ->收发数据；服务器端创建 socket -> 绑定 IP 和端口号 -> 启动侦听 ->接受客户端连接 ->与客户端通信收发数据】即可）、TCP 与 UDP 的区别等等。

对于工作经验三年以内的社会人士或者一些中级面试者一般会问一些稍微重难点问题，如 select 函数的用法，非阻塞 connect 函数的写法，epoll 的水平和边缘模式、阻塞socket与非阻塞 socket 的区别、send/recv 函数的返回值情形、REUSE_ADDR 选项等等。Windows 平台可能还会问 WSAEventSelect 和 WSAAsyncSelect 函数的用法、完成端口（IOCP 模型）。

对于三年以上尤其是“号称”自己设计过服务器、看过开源网络通信库代码的面试者，面试官一般会深入问一些问题，这类问题要么是实际项目中常见的难题或者网络通信细节，根据我的经验，一般有这样一些问题：

1.nagle 算法；

2.keepalive 选项；

3.Linger 选项；

4.对于某一端出现大量 CLOSE_WAIT 或者 TIME_WAIT 如何解决；

5.通讯协议如何设计或如何解决数据包的粘包与分片问题；

6.心跳机制如何设计；（可能不会直接问问题本身，如问如何检查死链）

7.断线重连机制如何设计；

8.对 IO Multiplexing 技术的理解；

9.收发数据包正确的方式，收发缓冲区如何设计；

10.优雅关闭；

11.定时器如何设计；

12.epoll 的实现原理。

举两个例子，让读者感受一下：

> B 站面试曾被问的一个问题：如果 A 机器与 B 机器网络 connect 成功后从未互发过数据，此时其中一机器突然断电，则另外一台机器与断电的机器之间的网络连接处于哪种状态？

## 7. 内存数据库&缓存技术

时下以 NoSql key-value 为思想的内存数据库大行其道，广泛地用于各种后台项目开发。所以 **熟悉一种或几种内存数据库程序已经是面试后台开发的基本要求，** 而这当中以 Redis 为最典型代表，这里以 Redis 为例。

- 第一层面一般是对 Redis 的基础用法的考察 如考察 Redis 支持的基础数据类型、Redis的数据持久化、事务等。
    
- 第二层面不仅考察 Redis 的基础用法，还会深入到 Redis 源码层面上，如 Redis 的网络通信模型、Redis 各种数据结构的实现等等。
    
- Redis 高可用技术、cluster、哨兵策略等。
    

笔者以为，无论是从找工作应付面试还是从提高技术的角度，Redis 是一个非常值得学习的开源软件，希望广大读者有意识地去了解、学习它。

> 另外一些像分布式、RPC、MQ 等技术和 C++ 本身关系不是很紧密，这里就不罗列了。

## 8. 项目经验

除了社会招聘和一些小型的企业，一般的大型互联网公司对应届生不会做过多的项目经验要求，而是希望他们算法与数据结构等基础扎实、动手实践能力强即可。对于一般的小公司，对于应届生会要求其至少熟练使用一门编程语言以及相应的开发工具，号称熟悉 Linux C++ 开发的面试者，不熟悉 GDB 调试基本上不是真正的熟悉 Linux C++ 开发；号称熟悉汇编或者反汇编，不熟悉 IDA 或者 OllyDbg，基本上也是名不符实的；号称熟悉 VC++ 开发，连 F8、F9、F10、F11、F12 等快捷键不熟悉也是难以经得住面试官的提问的。**受疫情影响，很多面试都改成了线上面试，当你写算法题时如果你对开发工具不熟悉，面试官基本一下子就能看出来。** 这点请大家注意。

这里给一些学历不算好，学校不是非常有名，尤其是二本以下的广大想进入 IT 行业的同学一个建议，在大学期间除了要学好计算机专业基础知识以外，一定要熟练使用一门编程语言以及相应的开发工具。

关于项目经验，许多面试者认为一定要是自己参与的项目，其实也可以来源于你学习和阅读他人源码或开源软件的源码，如果你能理解并掌握这些开源软件中的思想和技术，在面试的时候能够与面试官侃侃而谈，面试官也会非常满意的。

很多同学可能纠结大学或者研究生期间要不要跟着导师做一些项目。当然，如果这些项目是课程要求，那么你必须得参加；如果这些项目是可以选择性的，尤其是一些仅仅拿着第三方的库进行所谓的包装和加工，那么建议可以少参加一些。

> 另外，大厂面试一般还有出一些场景题综合考察面试者的水平，这类场景题一般是多种技术的结合，可以参考我这里写的：《[来看一看两道大厂面试场景题](http://mp.weixin.qq.com/s?__biz=MzU2MTkwMTE4Nw==&mid=2247494228&idx=1&sn=f23f52d9a32f293111be55f1fab9e49a&chksm=fc7311b8cb0498ae4ed78213429d5129ba0e3a0fb1faffed9fba396eeed839430cb1a97e6786&scene=21#wechat_redirect)》。

另外我在我的公众号 **『高性能服务器开发』** 也为想做后台开发的同学整理了如下资料（公众号回复“**文章下载**”即可打包带走这些资料），欢迎关注：

- [C++高级进阶](https://mp.weixin.qq.com/mp/appmsgalbum?action=getalbum&album_id=1406722424518967297&__biz=MzU2MTkwMTE4Nw==#wechat_redirect)
    
      
    
- [后端开发面试题](https://mp.weixin.qq.com/mp/appmsgalbum?action=getalbum&album_id=1406847298814050304&__biz=MzU2MTkwMTE4Nw==#wechat_redirect)
    
      
    
- [多线程重难点解析](https://mp.weixin.qq.com/mp/appmsgalbum?action=getalbum&album_id=1406836024357126144&__biz=MzU2MTkwMTE4Nw==#wechat_redirect)
    
      
    
- [网络编程重难点解析](https://mp.weixin.qq.com/mp/appmsgalbum?action=getalbum&album_id=1406695350672523267&__biz=MzU2MTkwMTE4Nw==#wechat_redirect)
    
      
    
- [网络通信协议深度解析](https://mp.weixin.qq.com/mp/appmsgalbum?action=getalbum&album_id=1406825429595553793&__biz=MzU2MTkwMTE4Nw==#wechat_redirect)
    
      
    
- [服务器开发进阶](https://mp.weixin.qq.com/mp/appmsgalbum?action=getalbum&album_id=1406709287187087365&__biz=MzU2MTkwMTE4Nw==#wechat_redirect)
    
      
    
- [面试、offer 与薪资那些事儿](https://mp.weixin.qq.com/mp/appmsgalbum?action=getalbum&album_id=1406623755513856001&__biz=MzU2MTkwMTE4Nw==#wechat_redirect)
    
      
    
- [职业规划](https://mp.weixin.qq.com/mp/appmsgalbum?action=getalbum&album_id=1406488296171208708&__biz=MzU2MTkwMTE4Nw==#wechat_redirect)
    
      
    
- [张小方的故事](https://mp.weixin.qq.com/mp/appmsgalbum?action=getalbum&album_id=1406508875423121410&__biz=MzU2MTkwMTE4Nw==#wechat_redirect)
    

如果对后端开发感兴趣，想加入 **Linux** **服务器开发微信交流群** 进行交流，可以先加我微信 **easy_coder**，备注"加微信群"，我拉你入群，备注不对不加哦。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

### 原创不易，点在看是最大的支持

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

阅读 7209

​

写留言

**留言 41**

- 龙
    
    广东2023年4月13日
    
    赞1
    
    电子书能再分享一下吗![[社会社会]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)
    
    置顶
    
    CppGuide
    
    作者2023年4月13日
    
    赞2
    
    加微信easy_coder领取。
    
- 范蠡
    
    2021年3月24日
    
    赞3
    
    想知道文中的面试题的答案的小伙伴可以加微信群交流，加我微信easy_coder拉你入群。
    
    置顶
    
- Mr.Renᯤ⁶ᴳ
    
    2021年3月24日
    
    赞14
    
    没能力千万不要找别人内推，对谁都不好，自己有能力才找朋友帮忙内推，对朋友是一份信任，对内推公司是一种态度，对自己也是一份责任
    
    CppGuide
    
    作者2021年3月24日
    
    赞
    
    说的好。
    
- 编程指北
    
    2021年3月24日
    
    赞5
    
    👍🏻强
    
    CppGuide
    
    作者2021年3月24日
    
    赞
    
    小北来了呀![[愉快]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)
    
- 轩辕之风
    
    2021年3月24日
    
    赞1
    
    完了，只能回答90%的问题![/::-|](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)
    
    CppGuide
    
    作者2021年3月24日
    
    赞2
    
    我每次看我的leader面试，前前后后面了几十人了。结果至今录取了2人。。。。。。
    
- A🍀 飞鸟
    
    2021年3月24日
    
    赞2
    
    会这么多，只会累死自己
    
- 张先生 Enfp 竞选者
    
    2021年12月21日
    
    赞
    
    让面试者利用队列写一个多线程下的生产者和消费者程序，全面考察的多线程的资源同步与竞态问题（下文介绍多线程面试题时详细地介绍） 大佬有写这个吗？没找到
    
    CppGuide
    
    作者2021年12月21日
    
    赞1
    
    有的，你看条件变量那篇文章。
    
    张先生 Enfp 竞选者
    
    2021年12月23日
    
    赞
    
    看到了 谢谢大佬
    
- 安神豆腐脑
    
    2021年3月25日
    
    赞1
    
    基础知识很重要啊
    
- 阿秀
    
    2021年3月24日
    
    赞1
    
    范哥良心面试官，问的问题都很常见哈哈
    
    CppGuide
    
    作者2021年3月24日
    
    赞
    
    楼上的说太难了，你说都很常见，你是高手![[社会社会]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)
    
    阿秀
    
    2021年3月24日
    
    赞1
    
    ![[旺柴]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)C++基础那些，挺常见的 也就那些有区分度了![[捂脸]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)
    
- 柳阿九。
    
    2021年3月24日
    
    赞
    
    多来点干货
    
    CppGuide
    
    作者2021年3月24日
    
    赞1
    
    啥都不说了吧，求在看和转发。
    
- 迪士尼在逃王子
    
    2021年3月24日
    
    赞1
    
    ![[Lol]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)跟网易面试问题差不多
    
- 峰哥。
    
    2021年3月24日
    
    赞1
    
    字节面试不容易啊
    
    CppGuide
    
    作者2021年3月24日
    
    赞1
    
    我问的就是一些基础。。。。。。大佬这。。
    
- 罗登
    
    2021年3月24日
    
    赞1
    
    c++太难了吧
    
    CppGuide
    
    作者2021年3月24日
    
    赞
    
    任何技术学好都不容易。
    
- 可爱一只小小鱼
    
    2021年3月25日
    
    赞
    
    做嵌入式开发的后面想转后台，大佬有啥建议没![[Blush]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)
    
    CppGuide
    
    作者2021年3月25日
    
    赞
    
    补一下后台需要的技术栈，然后去面试看看，之前有不少星球球友都是嵌入式转的后台。
    
- 彼岸鱼
    
    2021年3月25日
    
    赞
    
    太真实了 这些问题这次春招问了好多
    
- 涛
    
    2021年3月24日
    
    赞
    
    好家伙！我收藏了！光看封面就知道是大佬，
    
- 雷小帅
    
    2021年3月24日
    
    赞
    
    怒赞![[强]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)
    
    CppGuide
    
    作者2021年3月24日
    
    赞
    
    看你的VCR，你竟然这么年轻～
    
- 孤客
    
    2021年3月24日
    
    赞
    
    之前找实习的时候就是问的这些![[Facepalm]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)![[Facepalm]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)
    
    CppGuide
    
    作者2021年3月24日
    
    赞
    
    然后呢，实习找到了吗？
    
    孤客
    
    2021年3月24日
    
    赞
    
    找到了，已经转正了，过几个月就去报道了![[Lol]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)
    
    CppGuide
    
    作者2021年3月24日
    
    赞
    
    ![[强]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)
    
- ManateeLazyCat
    
    2021年3月24日
    
    赞
    
    基础不牢 地动山摇
    
- 张自远
    
    2021年3月24日
    
    赞
    
    ![/:strong](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)
    
- Barney
    
    2021年3月24日
    
    赞
    
    说的好
    
- 吴少侠，也在江湖
    
    2021年3月24日
    
    赞
    
    目前正在积极转行中。看了群主的面试大部分考察的内容，C++基础部分，数据结构与算法(除了红黑树不是很了解外)，以及基本的数据库基本要求这些都还OK。接下来就要准备开始学习多线程编程网络编程，再做几个项目，就准备去找工作了，加油，给我冲！！！
    
    CppGuide
    
    作者2021年3月24日
    
    赞
    
    这些内容都不难，高级面试不会问这么基础的。加油！
    
- Tudou
    
    2021年3月24日
    
    赞
    
    问题有解答吗
    
    CppGuide
    
    作者2021年3月24日
    
    赞
    
    加群交流吧。
    

已无更多数据

[](javacript:;)

![](http://mmbiz.qpic.cn/sz_mmbiz_png/GSweNIrkicYvM1mIwPctlYONEDKJwUfRZ57uAkVR59MpX1cVnmmnyPZ5O9OCuys78Sy6fOEncwfgWpgCo9Tibeag/300?wx_fmt=png&wxfrom=18)

CppGuide

511024

41

写留言

**留言 41**

- 龙
    
    广东2023年4月13日
    
    赞1
    
    电子书能再分享一下吗![[社会社会]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)
    
    置顶
    
    CppGuide
    
    作者2023年4月13日
    
    赞2
    
    加微信easy_coder领取。
    
- 范蠡
    
    2021年3月24日
    
    赞3
    
    想知道文中的面试题的答案的小伙伴可以加微信群交流，加我微信easy_coder拉你入群。
    
    置顶
    
- Mr.Renᯤ⁶ᴳ
    
    2021年3月24日
    
    赞14
    
    没能力千万不要找别人内推，对谁都不好，自己有能力才找朋友帮忙内推，对朋友是一份信任，对内推公司是一种态度，对自己也是一份责任
    
    CppGuide
    
    作者2021年3月24日
    
    赞
    
    说的好。
    
- 编程指北
    
    2021年3月24日
    
    赞5
    
    👍🏻强
    
    CppGuide
    
    作者2021年3月24日
    
    赞
    
    小北来了呀![[愉快]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)
    
- 轩辕之风
    
    2021年3月24日
    
    赞1
    
    完了，只能回答90%的问题![/::-|](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)
    
    CppGuide
    
    作者2021年3月24日
    
    赞2
    
    我每次看我的leader面试，前前后后面了几十人了。结果至今录取了2人。。。。。。
    
- A🍀 飞鸟
    
    2021年3月24日
    
    赞2
    
    会这么多，只会累死自己
    
- 张先生 Enfp 竞选者
    
    2021年12月21日
    
    赞
    
    让面试者利用队列写一个多线程下的生产者和消费者程序，全面考察的多线程的资源同步与竞态问题（下文介绍多线程面试题时详细地介绍） 大佬有写这个吗？没找到
    
    CppGuide
    
    作者2021年12月21日
    
    赞1
    
    有的，你看条件变量那篇文章。
    
    张先生 Enfp 竞选者
    
    2021年12月23日
    
    赞
    
    看到了 谢谢大佬
    
- 安神豆腐脑
    
    2021年3月25日
    
    赞1
    
    基础知识很重要啊
    
- 阿秀
    
    2021年3月24日
    
    赞1
    
    范哥良心面试官，问的问题都很常见哈哈
    
    CppGuide
    
    作者2021年3月24日
    
    赞
    
    楼上的说太难了，你说都很常见，你是高手![[社会社会]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)
    
    阿秀
    
    2021年3月24日
    
    赞1
    
    ![[旺柴]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)C++基础那些，挺常见的 也就那些有区分度了![[捂脸]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)
    
- 柳阿九。
    
    2021年3月24日
    
    赞
    
    多来点干货
    
    CppGuide
    
    作者2021年3月24日
    
    赞1
    
    啥都不说了吧，求在看和转发。
    
- 迪士尼在逃王子
    
    2021年3月24日
    
    赞1
    
    ![[Lol]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)跟网易面试问题差不多
    
- 峰哥。
    
    2021年3月24日
    
    赞1
    
    字节面试不容易啊
    
    CppGuide
    
    作者2021年3月24日
    
    赞1
    
    我问的就是一些基础。。。。。。大佬这。。
    
- 罗登
    
    2021年3月24日
    
    赞1
    
    c++太难了吧
    
    CppGuide
    
    作者2021年3月24日
    
    赞
    
    任何技术学好都不容易。
    
- 可爱一只小小鱼
    
    2021年3月25日
    
    赞
    
    做嵌入式开发的后面想转后台，大佬有啥建议没![[Blush]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)
    
    CppGuide
    
    作者2021年3月25日
    
    赞
    
    补一下后台需要的技术栈，然后去面试看看，之前有不少星球球友都是嵌入式转的后台。
    
- 彼岸鱼
    
    2021年3月25日
    
    赞
    
    太真实了 这些问题这次春招问了好多
    
- 涛
    
    2021年3月24日
    
    赞
    
    好家伙！我收藏了！光看封面就知道是大佬，
    
- 雷小帅
    
    2021年3月24日
    
    赞
    
    怒赞![[强]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)
    
    CppGuide
    
    作者2021年3月24日
    
    赞
    
    看你的VCR，你竟然这么年轻～
    
- 孤客
    
    2021年3月24日
    
    赞
    
    之前找实习的时候就是问的这些![[Facepalm]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)![[Facepalm]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)
    
    CppGuide
    
    作者2021年3月24日
    
    赞
    
    然后呢，实习找到了吗？
    
    孤客
    
    2021年3月24日
    
    赞
    
    找到了，已经转正了，过几个月就去报道了![[Lol]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)
    
    CppGuide
    
    作者2021年3月24日
    
    赞
    
    ![[强]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)
    
- ManateeLazyCat
    
    2021年3月24日
    
    赞
    
    基础不牢 地动山摇
    
- 张自远
    
    2021年3月24日
    
    赞
    
    ![/:strong](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)
    
- Barney
    
    2021年3月24日
    
    赞
    
    说的好
    
- 吴少侠，也在江湖
    
    2021年3月24日
    
    赞
    
    目前正在积极转行中。看了群主的面试大部分考察的内容，C++基础部分，数据结构与算法(除了红黑树不是很了解外)，以及基本的数据库基本要求这些都还OK。接下来就要准备开始学习多线程编程网络编程，再做几个项目，就准备去找工作了，加油，给我冲！！！
    
    CppGuide
    
    作者2021年3月24日
    
    赞
    
    这些内容都不难，高级面试不会问这么基础的。加油！
    
- Tudou
    
    2021年3月24日
    
    赞
    
    问题有解答吗
    
    CppGuide
    
    作者2021年3月24日
    
    赞
    
    加群交流吧。
    

已无更多数据