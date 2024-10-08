不才陈某 码猿技术专栏

_2024年02月17日 14:08_ _浙江_

大家好，我是不才陈某~

相信很多人知道石中剑这个典故，在此典故中，天命注定的亚瑟很容易的就拔出了这把石中剑，但是由于资历不被其他人认可，所以他颇费了一番周折才成为了真正意义上的英格兰全境之王，亚瑟王。

说道这把剑，剑身上铭刻着这样一句话：ONLY THE KING CAN TAKE THE SWORD FROM THE STONE。

虽然典故中的 the king 是指英明之主亚瑟王，但是在本章中，这个 king 就是读者自己。

我们今天不仅要从百万并发基石上拔出这把 epoll 之剑，也就是 Netty，而且要利用这把剑大杀四方，一如当年的亚瑟王凭借此剑统一了英格兰全境一样。

说到石中剑 Netty，我们知道他极其强悍的性能以及纯异步模型，释放出了极强的生产力，内置的各种编解码编排，心跳包检测，粘包拆包处理等，高效且易于使用，以至于很多耳熟能详的组件都在使用，比如 Hadoop，Dubbo 等。

但是他是如何做到这些的呢？本章将会以庖丁解牛的方式，一步一步的来拔出此剑。

## Netty 的异步模型

说起 Netty 的异步模型，我相信大多数人，只要是写过服务端的话，都是耳熟能详的，bossGroup 和 workerGroup 被 ServerBootstrap 所驱动，用起来简直是如虎添翼。

再加上各种配置化的 handler 加持，组装起来也是行云流水，俯拾即是。但是，任何一个好的架构，都不是一蹴而就实现的，那她经历了怎样的心路历程呢？

①经典的多线程模型

![](https://mmbiz.qpic.cn/mmbiz_png/19cc2hfD2rAuvUlicOda8HEpUtWaMSVpWRLdg4eQicJYbm0xNVJmwovQWoUuS1V01kGMuqjYuKKHCSsicOP9B5B9A/640?wx_fmt=png&wxfrom=13&tp=wxpic)

此模型中，服务端起来后，客户端连接到服务端，服务端会为每个客户端开启一个线程来进行后续的读写操作。

客户端少的时候，整体性能和功能还是可以的，但是如果客户端非常多的时候，线程的创建将会导致内存的急剧飙升从而导致服务端的性能下降，严重者会导致新客户端连接不上来，更有甚者，服务器直接宕机。

此模型虽然简单，但是由于其简单粗暴，所以难堪大用，建议在写服务端的时候，要彻底的避免此种写法。

②经典的 Reactor 模型

由于多线程模型难堪大用，所以更好的模型一直在研究之中，Reactor 模型，作为天选之子，也被引入了进来，由于其强大的基于事件处理的特性，使得其成为异步模型的不二之选。

Reactor 模型由于是基于事件处理的，所以一旦有事件被触发，将会派发到对应的 event handler 中进行处理。

所以在此模型中，有两个最重要的参与者，列举如下：

- Reactor：主要用来将 IO 事件派发到相对应的 handler 中，可以将其想象为打电话时候的分发总机，你先打电话到总机号码，然后通过总机，你可以分拨到各个分机号码。

- Handlers：主要用来处理 IO 事件相关的具体业务，可以将其想象为拨通分机号码后，实际上为你处理事件的员工。

!\[\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

上图为 Reactor 模型的描述图，具体来说一下：

Initiation Dispatcher 其实扮演的就是 Reactor 的角色，主要进行 Event Demultiplexer，即事件派发。

而其内部一般都有一个 Acceptor，用于通过对系统资源的操纵来获取资源句柄，然后交由 Reactor，通过 handle_events 方法派发至具体的 EventHandler 的。

Synchronous Event Demultiplexer 其实就是 Acceptor 的角色，此角色内部通过调用系统的方法来进行资源操作。

比如说，假如客户端连接上来，那么将会获得当前连接，假如需要删除文件，那么将会获得当前待操作的文件句柄等等。

这些句柄实际上是要返回给 Reactor 的，然后经由 Reactor 派发下放给具体的 EventHandler。

Event Handler 这里，其实就是具体的事件操作了。其内部针对不同的业务逻辑，拥有不同的操作方法。

比如说，鉴权 EventHandler 会检测传入的连接，验证其是否在白名单，心跳包 EventHanler 会检测管道是否空闲。

业务 EventHandler 会进行具体的业务处理，编解码 EventHandler 会对当前连接传输的内容进行编码解码操作等等。

由于 Netty 是 Reactor 模型的具体实现，所以在编码的时候，我们可以非常清楚明白的理解 Reactor 的具体使用方式，这里暂时不讲，后面会提到。

由于 Doug Lea 写过一篇关于 NIO 的文章，整体总结的极好，所以这里我们就结合他的文章来详细分析一下 Reactor 模型的演化过程。

!\[\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

上图模型为单线程 Reator 模型，Reactor 模型会利用给定的 selectionKeys 进行派发操作，派发到给定的 handler。

之后当有客户端连接上来的时候，acceptor 会进行 accept 接收操作，之后将接收到的连接和之前派发的 handler 进行组合并启动。

!\[\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

上图模型为池化 Reactor 模型，此模型将读操作和写操作解耦了出来，当有数据过来的时候，将 handler 的系列操作扔到线程池中来进行，极大的提到了整体的吞吐量和处理速度。

!\[\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

上图模型为多 Reactor 模型，此模型中，将原本单个 Reactor 一分为二，分别为 mainReactor 和 subReactor。

其中 mainReactor 主要进行客户端连接方面的处理，客户端 accept 后发送给 subReactor 进行后续处理处理。

这种模型的好处就是整体职责更加明确，同时对于多 CPU 的机器，系统资源的利用更加高一些。

!\[\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

从 Netty 写的 server 端，就可以看出，boss worker group 对应的正是主副 Reactor。

之后 ServerBootstrap 进行 Reactor 的创建操作，里面的 group，channel，option 等进行初始化操作。

而设置的 childHandler 则是具体的业务操作，其底层的事件分发器则通过调用 Linux 系统级接口 epoll 来实现连接并将其传给 Reactor。

## 石中剑 Netty 强悍的原理（JNI）

Netty 之剑之所以锋利，不仅仅因为其纯异步的编排模型，避免了各种阻塞式的操作，同时其内部各种设计精良的组件，终成一统。

且不说让人眼前一亮的缓冲池设计，读写标随心而动，摒弃了繁冗复杂的边界检测，用起来着实舒服之极。

原生的流控和高低水位设计，让流速控制真的是随心所欲，铸就了一道相当坚固的护城河。

齐全的粘包拆包处理方式，让每一笔数据都能够清晰明了；而高效的空闲检测机制，则让心跳包和断线重连等设计方案变得如此俯拾即是。

上层的设计如此优秀，其性能又怎能甘居下风。由于底层通讯方式完全是 C 语言编写，然后利用 JNI 机制进行处理，所以整体的性能可以说是达到了原生 C 语言性能的强悍程度。

说道 JNI，这里我觉得有必要详细说一下，他是我们利用 Java 直接调用 C 语言原生代码的关键。

JNI，全称为Java Native Interface，翻译过来就是 Java 本地接口，他是 Java 调用 C 语言的一套规范。具体来看看怎么做的吧。

步骤一，先来写一个简单的 Java 调用函数：

`public class DataSynchronizer {       /**        * 加载本地底层C实现库        */       static {           System.loadLibrary("synchronizer");       }       /**        * 底层数据同步方法        */       private native String syncData(String status);       /**        * 程序启动，调用底层数据同步方法        *        * @param args        */       public static void main(String... args) {           String rst = new DataSynchronizer().syncData("ProcessStep2");           System.out.println("The execute result from C is : " + rst);       }   }   `

可以看出，是一个非常简单的 Java 类，此类中，syncData 方法前面带了 native 修饰，代表此方法最终将会调用底层 C 语言实现。main 方法是启动类，将 C 语言执行的结果接收并打印出来。

然后，打开我们的 Linux 环境，这里由于我用的是 linux mint，依次执行如下命令来设置环境：

`执行apt install default-jdk 安装java环境，安装完毕。      通过update-alternatives --list java 获取java安装路径，这里为：/usr/lib/jvm/java-11-openjdk-amd64         设置java环境变量 export JAVA_HOME=/usr/lib/jvm/java-11-openjdk-amd64      环境设置完毕之后，就可以开始进行下一步了。   `

步骤二，编译，首先，进入到代码 DataSynchronizer.c 所在的目录，然后运行如下命令来编译 Java 源码：

`javac -h . DataSynchronizer.java   `

编译完毕之后，可以看到当前目录出现了如下几个文件：

!\[\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

其中 DataSynchronizer.h 是生成的头文件，这个文件尽量不要修改，整体内容如下：

`/* DO NOT EDIT THIS FILE - it is machine generated */   #include <jni.h>   /* Header for class DataSynchronizer */   #ifndef _Included_DataSynchronizer   #define _Included_DataSynchronizer   #ifdef __cplusplus   extern "C" {   #endif   /*    * Class:     DataSynchronizer    * Method:    syncData    * Signature: (Ljava/lang/String;)Ljava/lang/String;    */   JNIEXPORT jstring JNICALL Java_DataSynchronizer_syncData     (JNIEnv *, jobject, jstring);   #ifdef __cplusplus   }   #endif   #endif   `

其中 JNIEXPORT jstring JNICALL Java_DataSynchronizer_syncData 方法，就是给我们生成的本地 C 语言方法，我们这里只需要创建一个 C 语言文件，名称为 DataSynchronizer.c。

将此头文件加载进来，实现此方法即可：

`#include <jni.h>   #include <stdio.h>   #include "DataSynchronizer.h"      JNIEXPORT jstring JNICALL Java_DataSynchronizer_syncData(JNIEnv *env, jobject obj, jstring str) {      // Step 1: Convert the JNI String (jstring) into C-String (char*)      const char *inCStr = (*env)->GetStringUTFChars(env, str, NULL);      if (NULL == inCStr) {           return NULL;       }         // Step 2: Perform its intended operations      printf("In C, the received string is: %s\n", inCStr);      (*env)->ReleaseStringUTFChars(env, str, inCStr);  // release resources         // Prompt user for a C-string      char outCStr[128];      printf("Enter a String: ");      scanf("%s", outCStr);         // Step 3: Convert the C-string (char*) into JNI String (jstring) and return      return (*env)->NewStringUTF(env, outCStr);   }   `

其中需要注意的是，JNIEnv\* 变量，实际上指的是当前的 JNI 环境。而 jobject 变量则类似 Java 中的 this 关键字。

jstring 则是 C 语言层面上的字符串，相当于 Java 中的 String。整体对应如下：

!\[\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

最后，我们来编译一下：

`gcc -fPIC -I"$JAVA_HOME/include" -I"$JAVA_HOME/include/linux" -shared -o libsynchronizer.so DataSynchronizer.c   `

编译完毕后，可以看到当前目录下又多了一个 libsynchronizer.so 文件（这个文件类似 Windows 上编译后生成的 .dll 类库文件）：

!\[\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

此时我们可以运行了，运行如下命令进行运行：

`java -Djava.library.path=. DataSynchronizer   `

得到结果如下：

`java -Djava.library.path=. DataSynchronizer   In C, the received string is: ProcessStep2   Enter a String: sdfsdf   The execute result from C is : sdfsdf   `

从这里看到，我们正确的通过 java jni 技术，调用了 C 语言底层的逻辑，然后获取到结果，打印了出来。

在 Netty 中，也是利用了 jni 的技术，然后通过调用底层的 C 语言逻辑实现，来实现高效的网络通讯的。

感兴趣的同学可以扒拉下 Netty 源码，在 transport-native-epoll 模块中，就可以见到具体的实现方法了。

!\[\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

## IO 多路复用模型

石中剑，之所以能荡平英格兰全境，自然有其最强悍的地方。

相应的，Netty，则也是不遑多让，之所以能够被各大知名的组件所采用，自然也有其最强悍的地方，而本章节的 IO 多路复用模型，则是其强悍的理由之一。

在说 IO 多路复用模型之前，我们先来大致了解下 Linux 文件系统。

在 Linux 系统中，不论是你的鼠标，键盘，还是打印机，甚至于连接到本机的 socket client 端，都是以文件描述符的形式存在于系统中，诸如此类，等等等等。

所以可以这么说，一切皆文件。来看一下系统定义的文件描述符说明：

!\[\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

从上面的列表可以看到，文件描述符 0，1，2 都已经被系统占用了，当系统启动的时候，这三个描述符就存在了。

其中 0 代表标准输入，1 代表标准输出，2 代表错误输出。当我们创建新的文件描述符的时候，就会在 2 的基础上进行递增。

可以这么说，文件描述符是为了管理被打开的文件而创建的系统索引，他代表了文件的身份 ID。对标 Windows 的话，你可以认为和句柄类似，这样就更容易理解一些。

由于网上对 Linux 文件这块的原理描述的文章已经非常多了，所以这里我不再做过多的赘述，感兴趣的同学可以从 Wikipedia 翻阅一下。

由于这块内容比较复杂，不属于本文普及的内容，建议读者另行自研。

## select 模型

此模型是 IO 多路复用的最早期使用的模型之一，距今已经几十年了，但是现在依旧有不少应用还在采用此种方式，可见其长生不老。

首先来看下其具体的定义（来源于 man 二类文档）：

`int select(int nfds, fd_set *readfds, fd_set *writefds, fd_set *errorfds, struct timeval *timeout);   `

这里解释下其具体参数：

- 参数一：nfds，也即 maxfd，最大的文件描述符递增一。这里之所以传最大描述符，为的就是在遍历 fd_set 的时候，限定遍历范围。

- 参数二：readfds，可读文件描述符集合。

- 参数三：writefds，可写文件描述符集合。

- 参数四：errorfds，异常文件描述符集合。

- 参数五：timeout，超时时间。在这段时间内没有检测到描述符被触发，则返回。

下面的宏处理，可以对 fd_set 集合（准确的说是 bitmap，一个描述符有变更，则会在描述符对应的索引处置 1）进行操作：

- FD_CLR(inr fd,fd_set\* set) ：用来清除描述词组 set 中相关 fd 的位，即 bitmap 结构中索引值为 fd 的值置为 0。

- FD_ISSET(int fd,fd_set \*set)：用来测试描述词组 set 中相关 fd 的位是否为真，即 bitmap 结构中某一位是否为 1。

- FD_SET（int fd,fd_set\*set）：用来设置描述词组 set 中相关 fd 的位，即将 bitmap 结构中某一位设置为 1，索引值为 fd。

- FD_ZERO（fd_set \*set）：用来清除描述词组 set 的全部位，即将 bitmap 结构全部清零。

首先来看一段服务端采用了 select 模型的示例代码：

`//创建server端套接字，获取文件描述符       int listenfd = socket(PF_INET,SOCK_STREAM,0);       if(listenfd < 0) return -1;          //绑定服务器       bind(listenfd,(struct sockaddr*)&address,sizeof(address));       //监听服务器       listen(listenfd,5);           struct sockaddr_in client;       socklen_t addr_len = sizeof(client);          //接收客户端连接       int connfd = accept(listenfd,(struct sockaddr*)&client,&addr_len);          //读缓冲区       char buff[1024];           //读文件操作符       fd_set read_fds;            while(1)       {           memset(buff,0,sizeof(buff));              //注意：每次调用select之前都要重新设置文件描述符connfd，因为文件描述符表会在内核中被修改           FD_ZERO(&read_fds);           FD_SET(connfd,&read_fds);              //注意：select会将用户态中的文件描述符表放到内核中进行修改，内核修改完毕后再返回给用户态，开销较大           ret = select(connfd+1,&read_fds,NULL,NULL,NULL);           if(ret < 0)           {               printf("Fail to select!\n");               return -1;           }              //检测文件描述符表中相关请求是否可读           if(FD_ISSET(connfd, &read_fds))           {               ret = recv(connfd,buff,sizeof(buff)-1,0);               printf("receive %d bytes from client: %s \n",ret,buff);           }       }   `

上面的代码我加了比较详细的注释了，大家应该很容易看明白，说白了大概流程其实如下：

- 首先，创建 socket 套接字，创建完毕后，会获取到此套接字的文件描述符。

- 然后，bind 到指定的地址进行监听 listen。这样，服务端就在特定的端口启动起来并进行监听了。

- 之后，利用开启 accept 方法来监听客户端的连接请求。一旦有客户端连接，则将获取到当前客户端连接的 connection 文件描述符。

双方建立连接之后，就可以进行数据互传了。需要注意的是，在循环开始的时候，务必每次都要重新设置当前 connection 的文件描述符，是因为文件描描述符表在内核中被修改过，如果不重置，将会导致异常的情况。

重新设置文件描述符后，就可以利用 select 函数从文件描述符表中，来轮询哪些文件描述符就绪了。

此时系统会将用户态的文件描述符表发送到内核态进行调整，即将准备就绪的文件描述符进行置位，然后再发送给用户态的应用中来。

用户通过 FD_ISSET 方法来轮询文件描述符，如果数据可读，则读取数据即可。

举个例子，假设此时连接上来了 3 个客户端，connection 的文件描述符分别为 4，8，12。

那么其 read_fds 文件描述符表（bitmap 结构）的大致结构为 00010001000100000....0。

由于 read_fds 文件描述符的长度为 1024 位，所以最多允许 1024 个连接。

!\[\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

而在 select 的时候，涉及到用户态和内核态的转换，所以整体转换方式如下：

!\[\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

所以，综合起来，select 整体还是比较高效和稳定的，但是呈现出来的问题也不少。

这些问题进一步限制了其性能发挥：

- 文件描述符表为 bitmap 结构，且有长度为 1024 的限制。

- fdset 无法做到重用，每次循环必须重新创建。

- 频繁的用户态和内核态拷贝，性能开销较大。

- 需要对文件描述符表进行遍历，O(n) 的轮询时间复杂度。

## poll 模型

考虑到 select 模型的几个限制，后来进行了改进，这也就是 poll 模型，既然是 select 模型的改进版，那么肯定有其亮眼的地方，一起来看看吧。

当然，这次我们依旧是先翻阅 linux man 二类文档，因为这是官方的文档，对其有着最为精准的定义。

`int poll(struct pollfd *fds, nfds_t nfds, int timeout);   `

其实，从运行机制上说来，poll 所做的功能和 select 是基本上一样的，都是等待并检测一组文件描述符就绪，然后在进行后续的 IO 处理工作。

只不过不同的是，select 中，采用的是 bitmap 结构，长度限定在 1024 位的文件描述符表，而 poll 模型则采用的是 pollfd 结构的数组 fds。

也正是由于 poll 模型采用了数组结构，则不会有 1024 长度限制，使其能够承受更高的并发。

pollfd 结构内容如下：

`struct pollfd {       int   fd;         /* 文件描述符 */       short events;     /* 关心的事件 */       short revents;    /* 实际返回的事件 */   };   `

从上面的结构可以看出，fd 很明显就是指文件描述符，也就是当客户端连接上来后，fd 会将生成的文件描述符保存到这里。

而 events 则是指用户想关注的事件；revents 则是指实际返回的事件，是由系统内核填充并返回，如果当前的 fd 文件描述符有状态变化，则 revents 的值就会有相应的变化。

events 事件列表如下：

!\[\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

revents 事件列表如下：

!\[\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

从列表中可以看出，revents 是包含 events 的。接下来结合示例来看一下：

`//创建server端套接字，获取文件描述符       int listenfd = socket(PF_INET,SOCK_STREAM,0);       if(listenfd < 0) return -1;          //绑定服务器       bind(listenfd,(struct sockaddr*)&address,sizeof(address));       //监听服务器       listen(listenfd,5);           struct pollfd pollfds[1];       socklen_t addr_len = sizeof(client);          //接收客户端连接       int connfd = accept(listenfd,(struct sockaddr*)&client,&addr_len);          //放入fd数组       pollfds[0].fd = connfd;       pollfds[0].events = POLLIN;          //读缓冲区       char buff[1024];           //读文件操作符       fd_set read_fds;            while(1)       {           memset(buff,0,sizeof(buff));              /**            ** SELECT模型专用            ** 注意：每次调用select之前都要重新设置文件描述符connfd，因为文件描述符表会在内核中被修改            ** FD_ZERO(&read_fds);            ** FD_SET(connfd,&read_fds);           ** 注意：select会将用户态中的文件描述符表放到内核中进行修改，内核修改完毕后再返回给用户态，开销较大           ** ret = select(connfd+1,&read_fds,NULL,NULL,NULL);           **/              ret = poll(pollfds, 1, 1000);           if(ret < 0)           {               printf("Fail to poll!\n");               return -1;           }              /**            ** SELECT模型专用            ** 检测文件描述符表中相关请求是否可读            ** if(FD_ISSET(connfd, &read_fds))            ** {            **   ret = recv(connfd,buff,sizeof(buff)-1,0);            **   printf("receive %d bytes from client: %s \n",ret,buff);            ** }            **/           //检测文件描述符数组中相关请求           if(pollfds[0].revents & POLLIN){               pollfds[0].revents = 0;               ret = recv(connfd,buff,sizeof(buff)-1,0);               printf("receive %d bytes from client: %s \n",ret,buff);           }       }`

由于源码中，我做了比较详细的注释，同时将和 select 模型不一样的地方都列了出来，这里就不再详细解释了。

总体说来，poll 模型比 select 模型要好用一些，去掉了一些限制，但是仍然避免不了如下的问题：

- 用户态和内核态仍需要频繁切换，因为 revents 的赋值是在内核态进行的，然后再推送到用户态，和 select 类似，整体开销较大。

- 仍需要遍历数组，时间复杂度为 O（N）。

## epoll 模型

如果说 select 模型和 poll 模型是早期的产物，在性能上有诸多不尽人意之处，那么自 Linux 2.6 之后新增的 epoll 模型，则彻底解决了性能问题，一举使得单机承受百万并发的课题变得极为容易。

现在可以这么说，只需要一些简单的设置更改，然后配合上 epoll 的性能，实现单机百万并发轻而易举。

同时，由于 epoll 整体的优化，使得之前的几个比较耗费性能的问题不再成为羁绊，所以也成为了 Linux 平台上进行网络通讯的首选模型。

讲解之前，还是 linux man 文档镇楼：linux man epoll 4 类文档 linux man epoll 7 类文档，俩文档结合着读，会对 epoll 有个大概的了解。

和之前提到的 select 和 poll 不同的是，此二者皆属于系统调用函数，但是 epoll 则不然，他是存在于内核中的数据结构。

可以通过 epoll_create，epoll_ctl 及 epoll_wait 三个函数结合来对此数据结构进行操控。

说到 epoll_create 函数，其作用是在内核中创建一个 epoll 数据结构实例，然后将返回此实例在系统中的文件描述符。

此 epoll 数据结构的组成其实是一个链表结构，我们称之为 interest list，里面会注册连接上来的 client 的文件描述符。

其简化工作机制如下：

!\[\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

说道 epoll_ctl 函数，其作用则是对 epoll 实例进行增删改查操作。有些类似我们常用的 CRUD 操作。

这个函数操作的对象其实就是 epoll 数据结构，当有新的 client 连接上来的时候，他会将此 client 注册到 epoll 中的 interest list 中，此操作通过附加 EPOLL_CTL_ADD 标记来实现。

当已有的 client 掉线或者主动下线的时候，他会将下线的 client从epoll 的 interest list 中移除，此操作通过附加 EPOLL_CTL_DEL 标记来实现。

当有 client 的文件描述符有变更的时候，他会将 events 中的对应的文件描述符进行更新，此操作通过附加 EPOLL_CTL_MOD 来实现。

当 interest list 中有 client 已经准备好了，可以进行 IO 操作的时候，他会将这些 clients 拿出来，然后放到一个新的 ready list 里面。

其简化工作机制如下：

!\[\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

说道 epoll_wait 函数，其作用就是扫描 ready list，处理准备就绪的 client IO，其返回结果即为准备好进行 IO 的 client 的个数。通过遍历这些准备好的 client，就可以轻松进行 IO 处理了。

上面这三个函数是 epoll 操作的基本函数，但是，想要彻底理解 epoll，则需要先了解这三块内容，即：inode，链表，红黑树。

在 Linux 内核中，针对当前打开的文件，有一个 open file table，里面记录的是所有打开的文件描述符信息；同时也有一个 inode table，里面则记录的是底层的文件描述符信息。

这里假如文件描述符 B fork 了文件描述符 A，虽然在 open file table 中，我们看新增了一个文件描述符 B，但是实际上，在 inode table 中，A 和 B 的底层是一模一样的。

这里，将 inode table 中的内容理解为 Windows 中的文件属性，会更加贴切和易懂。

这样存储的好处就是，无论上层文件描述符怎么变化，由于 epoll 监控的数据永远是 inode table 的底层数据，那么我就可以一直能够监控到文件的各种变化信息，这也是 epoll 高效的基础。

简化流程如下：

!\[\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

数据存储这块解决了，那么针对连接上来的客户端 socket，该用什么数据结构保存进来呢？

这里用到了红黑树，由于客户端 socket 会有频繁的新增和删除操作，而红黑树这块时间复杂度仅仅为 O(logN)，还是挺高效的。

有人会问为啥不用哈希表呢？当大量的连接频繁的进行接入或者断开的时候，扩容或者其他行为将会产生不少的 rehash 操作，而且还要考虑哈希冲突的情况。

虽然查询速度的确可以达到 o(1)，但是 rehash 或者哈希冲突是不可控的，所以基于这些考量，我认为红黑树占优一些。

客户端 socket 怎么管理这块解决了，接下来，当有 socket 有数据需要进行读写事件处理的时候，系统会将已经就绪的 socket 添加到双向链表中，然后通过 epoll_wait 方法检测的时候。

其实检查的就是这个双向链表，由于链表中都是就绪的数据，所以避免了针对整个客户端 socket 列表进行遍历的情况，使得整体效率大大提升。

整体的操作流程为：

- 首先，利用 epoll_create 在内核中创建一个 epoll 对象。其实这个 epoll 对象，就是一个可以存储客户端连接的数据结构。

- 然后，客户端 socket 连接上来，会通过 epoll_ctl 操作将结果添加到 epoll 对象的红黑树数据结构中。

- 然后，一旦有 socket 有事件发生，则会通过回调函数将其添加到 ready list 双向链表中。

- 最后，epoll_wait 会遍历链表来处理已经准备好的 socket，然后通过预先设置的水平触发或者边缘触发来进行数据的感知操作。

从上面的细节可以看出，由于 epoll 内部监控的是底层的文件描述符信息，可以将变更的描述符直接加入到 ready list，无需用户将所有的描述符再进行传入。

同时由于 epoll_wait 扫描的是已经就绪的文件描述符，避免了很多无效的遍历查询，使得 epoll 的整体性能大大提升，可以说现在只要谈论 Linux 平台的 IO 多路复用，epoll 已经成为了不二之选。

## 水平触发和边缘触发

上面说到了 epoll，主要讲解了 client 端怎么连进来，但是并未详细的讲解 epoll_wait 怎么被唤醒的，这里我将来详细的讲解一下。

水平触发，意即 Level Trigger，边缘触发，意即 Edge Trigger，如果单从字面意思上理解，则不太容易，但是如果将硬件设计中的水平沿，上升沿，下降沿的概念引进来，则理解起来就容易多了。

比如我们可以这样认为：

!\[\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

如果将上图中的方块看做是 buffer 的话，那么理解起来则就更加容易了，比如针对水平触发，buffer 只要是一直有数据，则一直通知；而边缘触发，则 buffer 容量发生变化的时候，才会通知。

虽然可以这样简单的理解，但是实际上，其细节处理部分，比图示中展现的更加精细，这里来详细的说一下。

①边缘触发

针对读操作，也就是当前 fd 处于 EPOLLIN 模式下，即可读。此时意味着有新的数据到来，接收缓冲区可读，以下 buffer 都指接收缓冲区：

buffer 由空变为非空，意即有数据进来的时候，此过程会触发通知：

!\[\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

buffer 原本有些数据，这时候又有新数据进来的时候，数据变多，此过程会触发通知：

!\[\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

buffer 中有数据，此时用户对操作的 fd 注册 EPOLL_CTL_MOD 事件的时候，会触发通知：

!\[\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

针对写操作，也就是当前 fd 处于 EPOLLOUT 模式下，即可写。此时意味着缓冲区可以写了，以下 buffer 都指发送缓冲区：

buffer 满了，这时候发送出去一些数据，数据变少，此过程会触发通知：

!\[\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

buffer 原本有些数据，这时候又发送出去一些数据，数据变少，此过程会触发通知：

!\[\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

这里就是 ET 这种模式触发的几种情形，可以看出，基本上都是围绕着接收缓冲区或者发送缓冲区的状态变化来进行的。

晦涩难懂？不存在的，举个栗子：

在服务端，我们开启边缘触发模式，然后将 buffer size 设为 10 个字节，来看看具体的表现形式。

服务端开启，客户端连接，发送单字符 A 到服务端，输出结果如下：

`-->ET Mode: it was triggered once      get 1 bytes of content: A      -->wait to read!   `

可以看到，由于 buffer 从空到非空，边缘触发通知产生，之后在 epoll_wait 处阻塞，继续等待后续事件。

这里我们变一下，输入 ABCDEFGHIJKLMNOPQ，可以看到，客户端发送的字符长度超过了服务端 buffer size，那么输出结果将是怎么样的呢？

`-->ET Mode: it was triggered once      get 9 bytes of content: ABCDEFGHI      get 8 bytes of content: JKLMNOPQ      -->wait to read!   `

可以看到，这次发送，由于发送的长度大于 buffer size，所以内容被折成两段进行接收，由于用了边缘触发方式，buffer 的情况是从空到非空，所以只会产生一次通知。

②水平触发

水平触发则简单多了，他包含了边缘触发的所有场景，简而言之如下：

当接收缓冲区不为空的时候，有数据可读，则读事件会一直触发：

!\[\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

当发送缓冲区未满的时候，可以继续写入数据，则写事件一直会触发：

!\[\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

同样的，为了使表达更清晰，我们也来举个栗子，按照上述入输入方式来进行。

服务端开启，客户端连接并发送单字符 A，可以看到服务端输出情况如下：

`-->LT Mode: it was triggered once!      get 1 bytes of content: A   `

这个输出结果，毋庸置疑，由于 buffer 中有数据，所以水平模式触发，输出了结果。

服务端开启，客户端连接并发送 ABCDEFGHIJKLMNOPQ，可以看到服务端输出情况如下：

`-->LT Mode: it was triggered once!      get 9 bytes of content: ABCDEFGHI      -->LT Mode: it was triggered once!      get 8 bytes of content: JKLMNOPQ   `

从结果中，可以看出，由于 buffer 中数据读取完毕后，还有未读完的数据，所以水平模式会一直触发，这也是为啥这里水平模式被触发了两次的原因。

有了这两个栗子的比对，不知道聪明的你，get 到二者的区别了吗？

在实际开发过程中，实际上 LT 更易用一些，毕竟系统帮助我们做了大部分校验通知工作，之前提到的 SELECT 和 POLL，默认采用的也都是这个。

但是需要注意的是，当有成千上万个客户端连接上来开始进行数据发送，由于 LT 的特性，内核会频繁的处理通知操作，导致其相对于 ET 来说，比较的耗费系统资源，所以，随着客户端的增多，其性能也就越差。

而边缘触发，由于监控的是 FD 的状态变化，所以整体的系统通知并没有那么频繁，高并发下整体的性能表现也要好很多。

但是由于此模式下，用户需要积极的处理好每一笔数据，带来的维护代价也是相当大的，稍微不注意就有可能出错。所以使用起来须要非常小心才行。

至于二者如何抉择，诸位就仁者见仁智者见智吧。

行文到这里，关于 epoll 的讲解基本上完毕了，大家从中是不是学到了很多干货呢？

由于从 Netty 研究到 linux epoll 底层，其难度非常大，可以用曲高和寡来形容，所以在这块探索的文章是比较少的，很多东西需要自己照着 man 文档和源码一点一点的琢磨（linux 源码详见 eventpoll.c 等）。

这里我来纠正一下搜索引擎上，说 epoll 高性能是因为利用 mmap 技术实现了用户态和内核态的内存共享，所以性能好。

我前期被这个观点误导了好久，后来下来了 Linux 源码，翻了一下，并没有在 epoll 中翻到 mmap 的技术点，所以这个观点是错误的。

这些错误观点的文章，国内不少，国外也不少，希望大家能审慎抉择，避免被错误带偏。

所以，epoll 高性能的根本就是，其高效的文件描述符处理方式加上颇具特性边的缘触发处理模式，以极少的内核态和用户态的切换，实现了真正意义上的高并发。

## 手写 epoll 服务端

实践是最好的老师，我们现在已经知道了 epoll 之剑怎么嵌入到石头中的，现在就让我们不妨尝试着拔一下看看。

手写 epoll 服务器，具体细节如下（非 C 语言 coder，代码有参考）：

`#include <stdio.h>   #include <sys/types.h>   #include <sys/socket.h>   #include <netinet/in.h>   #include <arpa/inet.h>   #include <unistd.h>   #include <string.h>   #include <fcntl.h>   #include <stdlib.h>   #include <sys/epoll.h>   #include <pthread.h>   #include <errno.h>   #include <stdbool.h>   #define MAX_EVENT_NUMBER 1024   //事件总数量   #define BUFFER_SIZE 10          //缓冲区大小，这里为10个字节   #define ENABLE_ET 0             //ET模式   /* 文件描述符设为非阻塞状态    * 注意：这个设置很重要，否则体现不出高性能    */   int SetNonblocking(int fd)   {       int old_option = fcntl(fd, F_GETFL);       int new_option = old_option | O_NONBLOCK;       fcntl(fd, F_SETFL, new_option);       return old_option;   }   /* 将文件描述符fd放入到内核中的epoll数据结构中并将fd设置为EPOLLIN可读，同时根据ET开关来决定使用水平触发还是边缘触发模式     * 注意：默认为水平触发，或上EPOLLET则为边缘触发   */   void AddFd(int epoll_fd, int fd, bool enable_et)   {       struct epoll_event event;  //为当前fd设置事件       event.data.fd = fd;        //指向当前fd       event.events = EPOLLIN;    //使得fd可读       if(enable_et)       {           event.events |= EPOLLET; //设置为边缘触发       }       epoll_ctl(epoll_fd, EPOLL_CTL_ADD, fd, &event);  //将fd添加到内核中的epoll实例中       SetNonblocking(fd);  //设为非阻塞模式                         }   /*  LT水平触发     *  注意：水平触发简单易用，性能不高，适合低并发场合    *        一旦缓冲区有数据，则会重复不停的进行通知，直至缓冲区数据读写完毕    */   void lt_process(struct epoll_event* events, int number, int epoll_fd, int listen_fd)   {       char buf[BUFFER_SIZE];       int i;       for(i = 0; i < number; i++) //已经就绪的事件，这些时间可读或者可写       {           int sockfd = events[i].data.fd; //获取描述符           if(sockfd == listen_fd)  //如果监听类型的描述符，则代表有新的client接入，则将其添加到内核中的epoll结构中           {               struct sockaddr_in client_address;               socklen_t client_addrlength = sizeof(client_address);               int connfd = accept(listen_fd, (struct sockaddr*)&client_address, &client_addrlength); //创建连接并返回文件描述符（实际进行的三次握手过程）               AddFd(epoll_fd, connfd, false);  //添加到epoll结构中并初始化为LT模式           }           else if(events[i].events & EPOLLIN) //如果客户端有数据过来           {               printf("-->LT Mode: it was triggered once!\n");               memset(buf, 0, BUFFER_SIZE);                int ret = recv(sockfd, buf, BUFFER_SIZE - 1, 0);               if(ret <= 0)  //读取数据完毕后，关闭当前描述符               {                   close(sockfd);                   continue;               }               printf("get %d bytes of content: %s\n", ret, buf);           }           else           {               printf("something unexpected happened!\n");           }       }   }   /*  ET Work mode features: efficient but potentially dangerous */   /*  ET边缘触发    *  注意：边缘触发由于内核不会频繁通知，所以高效，适合高并发场合，但是处理不当将会导致严重事故             其通知机制和触发方式参见之前讲解，由于不会重复触发，所以需要处理好缓冲区中的数据，避免脏读脏写或者数据丢失等    */   void et_process(struct epoll_event* events, int number, int epoll_fd, int listen_fd)   {       char buf[BUFFER_SIZE];       int i;       for(i = 0; i < number; i++)       {           int sockfd = events[i].data.fd;           if(sockfd == listen_fd) //如果有新客户端请求过来，将其添加到内核中的epoll结构中并默认置为ET模式           {               struct sockaddr_in client_address;               socklen_t client_addrlength = sizeof(client_address);               int connfd = accept(listen_fd, (struct sockaddr*)&client_address, &client_addrlength);               AddFd(epoll_fd, connfd, true);            }           else if(events[i].events & EPOLLIN) //如果客户端有数据过来           {               printf("-->ET Mode: it was triggered once\n");               while(1) //循环等待               {                   memset(buf, 0, BUFFER_SIZE);                   int ret = recv(sockfd, buf, BUFFER_SIZE - 1, 0);                   if(ret < 0)                   {                       if(errno == EAGAIN || errno == EWOULDBLOCK) //通过EAGAIN检测，确认数据读取完毕                       {                           printf("-->wait to read!\n");                           break;                       }                       close(sockfd);                       break;                   }                   else if(ret == 0) //数据读取完毕，关闭描述符                   {                       close(sockfd);                   }                   else //数据未读取完毕，继续读取                   {                       printf("get %d bytes of content: %s\n", ret, buf);                   }               }           }           else           {               printf("something unexpected happened!\n");           }       }   }   int main(int argc, char* argv[])   {       const char* ip = "10.0.76.135";       int port = 9999;          //套接字设置这块，参见https://www.gta.ufrj.br/ensino/eel878/sockets/sockaddr_inman.html       int ret = -1;       struct sockaddr_in address;        bzero(&address, sizeof(address));       address.sin_family = AF_INET;       inet_pton(AF_INET, ip, &address.sin_addr);       address.sin_port = htons(port);       int listen_fd = socket(PF_INET, SOCK_STREAM, 0);    //创建套接字并返回描述符       if(listen_fd < 0)       {           printf("fail to create socket!\n");           return -1;       }       ret = bind(listen_fd, (struct sockaddr*)&address, sizeof(address)); //绑定本机       if(ret == -1)       {           printf("fail to bind socket!\n");           return -1;       }       ret = listen(listen_fd, 5); //在端口上监听       if(ret == -1)       {           printf("fail to listen socket!\n");           return -1;       }       struct epoll_event events[MAX_EVENT_NUMBER];       int epoll_fd = epoll_create(5);  //在内核中创建epoll实例，flag为5只是为了分配空间用，实际可以不用带       if(epoll_fd == -1)       {           printf("fail to create epoll!\n");           return -1;       }       AddFd(epoll_fd, listen_fd, true); //添加文件描述符到epoll对象中       while(1)       {           int ret = epoll_wait(epoll_fd, events, MAX_EVENT_NUMBER, -1); //拿出就绪的文件描述符并进行处理           if(ret < 0)           {               printf("epoll failure!\n");               break;           }           if(ENABLE_ET) //ET处理方式           {               et_process(events, ret, epoll_fd, listen_fd);           }           else  //LT处理方式           {               lt_process(events, ret, epoll_fd, listen_fd);           }       }       close(listen_fd); //退出监听       return 0;   }   `

详细的注释我都已经写上去了，这就是整个 epoll server 端全部源码了，仅仅只有 200 行左右，是不是很惊讶。

接下来让我们来测试下性能，看看能够达到我们所说的单机百万并发吗？其实悄悄的给你说，Netty 底层的 C 语言实现，和这个是差不多的。

## 单机百万并发实战

在实际测试过程中，由于要实现高并发，那么肯定得使用 ET 模式了。

但是由于这块内容更多的是 Linux 配置的调整，且前人已经有了具体的文章了，所以这里就不做过多的解释了。

这里我们主要是利用 VMware 虚拟机一主三从，参数调优，来实现百万并发。

此块内容由于比较复杂，先暂时放一放，后续将会搭建环境并对此手写 server 进行压测。

_参考资料：_

- _https://www3.ntu.edu.sg/home/ehchua/programming/java/JavaNativeInterface.html_

- _https://copyconstruct.medium.com/the-method-to-epolls-madness-d9d2d6378642_

## 推荐阅读（求关注，别白嫖！）

1. [微服务最重要的10个设计模式](https://mp.weixin.qq.com/s?__biz=MzU3MDAzNDg1MA==&mid=2247517905&idx=1&sn=3d33a3fa39e9fa8276f83c6783f5a6d6&chksm=fcf7591ccb80d00a37f65928c2c2c636b3e7aa3d295cc1df00926633029221384ba7396dc92d&token=123612753&lang=zh_CN&scene=21#wechat_redirect)

1. [如何用 ELK 搭建 TB 级的日志监控系统？](https://mp.weixin.qq.com/s?__biz=MzU3MDAzNDg1MA==&mid=2247517680&idx=1&sn=57ce7829078aad1dd748244a47039612&chksm=fcf75a3dcb80d32ba66b8fd562c45df1d3fb395c0875030aecbfd5471ff2583cadcac6952295&token=154658036&lang=zh_CN&scene=21#wechat_redirect)

1. [实战干货！Spring Cloud Gateway 整合 OAuth2.0 实现分布式统一认证授权！](https://mp.weixin.qq.com/s?__biz=MzU3MDAzNDg1MA==&mid=2247503249&idx=1&sn=b33ae3ff70a08b17ee0779d6ccb30b53&chksm=fcf7125ccb809b4aa4985da09e620e06c606754e6a72681c93dcc88bdc9aa7ba0cb64f52dbc3&scene=21&cur_album_id=2042874937312346114#wechat_redirect)

1. [从实现原理来讲，Nacos 为什么这么强？](https://mp.weixin.qq.com/s?__biz=MzU3MDAzNDg1MA==&mid=2247514933&idx=1&sn=374da0ea32321baf6938ff2e611d8fce&chksm=fcf764f8cb80edee2a0c493f58570b1502fb093ccd38fd498de1f6c1213e24e0355d8bcd713f&scene=21&cur_album_id=2042874937312346114#wechat_redirect)

1. [阿里限流神器Sentinel夺命连环 17 问？](https://mp.weixin.qq.com/s?__biz=MzU3MDAzNDg1MA==&mid=2247498039&idx=1&sn=3a3caee655ff015b46249bd51aa4dc79&chksm=fcf726facb80afecea4d48faf94a9940b80ba21b325510cf4be6f7c7bce2f3c73266857f65d1&scene=21&cur_album_id=2042874937312346114#wechat_redirect)

1. [openFeign夺命连环9问，这谁受得了？](https://mp.weixin.qq.com/s?__biz=MzU3MDAzNDg1MA==&mid=2247496653&idx=1&sn=7185077b3bdc1d094aef645d677ec472&chksm=fcf72c00cb80a516a8d1bc3b89400e202f2cbc1fd465e6c51e84a9a3543ec1c8bcfe8edeaec2&scene=21&cur_album_id=2042874937312346114#wechat_redirect)

1. [Spring Cloud Gateway夺命连环10问？](https://mp.weixin.qq.com/s?__biz=MzU3MDAzNDg1MA==&mid=2247499894&idx=1&sn=f1606e4c00fd15292269afe052f5bca2&chksm=fcf71fbbcb8096ad349e6da50b0b9141964c2084d0a38eba977fe8baa3fbe8af3b20c7591110&scene=21&cur_album_id=2042874937312346114#wechat_redirect)

![](http://mmbiz.qpic.cn/mmbiz_png/19cc2hfD2rA07Je2pY1o0ic2KcPRn44icO8GVcKRdwiaYvrE6bNeTbWPicyV7c7jWmSyzsiaWASjjckzBcsJMJw06pA/300?wx_fmt=png&wxfrom=19)

**码猿技术专栏**

前蚂蚁P8，纯粹的技术人，以专栏的形式分享java全栈技术，求职面试、进阶网站：java-family.cn

134篇原创内容

公众号

个人观点，仅供参考

阅读 2006

​
