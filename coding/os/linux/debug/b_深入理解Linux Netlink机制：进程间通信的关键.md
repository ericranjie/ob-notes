![](https://zhuanlan.zhihu.com/p/693908092)

[](https://www.zhihu.com/)

首发于[Linux内核](https://www.zhihu.com/column/c_1290342714786152448)

写文章

![点击打开planet-frontier的主页](https://pic1.zhimg.com/v2-abed1a8c04700ba7d72b45195223e0ff_l.jpg?source=32738c0c&needBackground=1)

![深入理解Linux Netlink机制：进程间通信的关键](https://picx.zhimg.com/70/v2-184b51e1c04c7b7b7fc54252f91937ee_1440w.awebp?source=172ae18b&biz_tag=Post)

# 深入理解Linux Netlink机制：进程间通信的关键

[![玩转Linux内核](https://picx.zhimg.com/v2-52b5b5f22089a2f57aeffb52f2590a2e_l.jpg?source=172ae18b)](https://www.zhihu.com/people/gang-hao-xin-dong-23)

[玩转Linux内核](https://www.zhihu.com/people/gang-hao-xin-dong-23)

​![](https://pic1.zhimg.com/v2-4812630bc27d642f7cafcd6cdeca3d7a.jpg?source=88ceefae)

专注于C/C++领域技术、职业发展，公众号/深度Linux

已关注

22 人赞同了该文章

​

目录

收起

一、什么是netlink

二、用户态数据结构

三、netlink内核数据结构

四、netlink内核API

五、netlink实例

Netlink是Linux内核中用于进程间通信的机制。它提供了一种可靠且高效的方式，使[用户空间](https://zhida.zhihu.com/search?content_id=242331924&content_type=Article&match_order=1&q=%E7%94%A8%E6%88%B7%E7%A9%BA%E9%97%B4&zhida_source=entity)程序可以与内核进行通信，并在运行时监控和控制系统状态，Netlink机制通过一个特殊的套接字族（AF_NETLINK）实现，允许用户空间程序发送和接收各种类型的网络相关消息。这些消息可以涉及网络配置、[路由表](https://zhida.zhihu.com/search?content_id=242331924&content_type=Article&match_order=1&q=%E8%B7%AF%E7%94%B1%E8%A1%A8&zhida_source=entity)更新、连接状态变化等。

使用Netlink机制，用户空间程序可以与内核模块交互，例如操作网络设备、配置IP地址、管理路由表、获取[网络统计](https://zhida.zhihu.com/search?content_id=242331924&content_type=Article&match_order=1&q=%E7%BD%91%E7%BB%9C%E7%BB%9F%E8%AE%A1&zhida_source=entity)信息等。同时，Netlink还支持多播功能，使得多个进程可以同时监听同一类消息，在编程层面上，通过使用netlink库（libnl）或者直接操作netlink套接字来实现与内核的通信。

## 一、什么是netlink

Netlink套接字是用以实现**用户进程**与**内核进程**通信的一种特殊的进程间通信(IPC) ,也是[网络应用程序](https://zhida.zhihu.com/search?content_id=242331924&content_type=Article&match_order=1&q=%E7%BD%91%E7%BB%9C%E5%BA%94%E7%94%A8%E7%A8%8B%E5%BA%8F&zhida_source=entity)与内核通信的最常用的接口。

Netlink 是一种特殊的 socket，它是 Linux 所特有的，类似于 BSD 中的AF_ROUTE 但又远比它的功能强大，目前在最新的 Linux 内核（2.6.14）中使用netlink 进行应用与内核通信的应用很多，包括：路由 daemon（NETLINK_ROUTE），1-wire 子系统（NETLINK_W1），[用户态](https://zhida.zhihu.com/search?content_id=242331924&content_type=Article&match_order=1&q=%E7%94%A8%E6%88%B7%E6%80%81&zhida_source=entity) socket 协议（NETLINK_USERSOCK），防火墙（NETLINK_FIREWALL），socket 监视（NETLINK_INET_DIAG），netfilter 日志（NETLINK_NFLOG），ipsec 安全策略（NETLINK_XFRM），SELinux 事件通知（NETLINK_SELINUX），iSCSI 子系统（NETLINK_ISCSI），进程审计（NETLINK_AUDIT），转发信息表查询 （NETLINK_FIB_LOOKUP），netlink connector(NETLINK_CONNECTOR),netfilter 子系统（NETLINK_NETFILTER），IPv6 防火墙（NETLINK_IP6_FW），DECnet [路由信息](https://zhida.zhihu.com/search?content_id=242331924&content_type=Article&match_order=1&q=%E8%B7%AF%E7%94%B1%E4%BF%A1%E6%81%AF&zhida_source=entity)（NETLINK_DNRTMSG），内核事件向用户态通知（NETLINK_KOBJECT_UEVENT），通用 netlink（NETLINK_GENERIC）。

Netlink 是一种在内核与用户应用间进行双向数据传输的非常好的方式，用户态应用使用标准的 socket API 就可以使用 netlink 提供的强大功能，[内核态](https://zhida.zhihu.com/search?content_id=242331924&content_type=Article&match_order=1&q=%E5%86%85%E6%A0%B8%E6%80%81&zhida_source=entity)需要使用专门的内核 API 来使用 netlink。

一般来说用户空间和内核空间的通信方式有三种：`/proc、ioctl、Netlink`。而前两种都是单向的，而Netlink可以实现双工通信。

**Netlink 相对于系统调用，[ioctl](https://zhida.zhihu.com/search?content_id=242331924&content_type=Article&match_order=2&q=ioctl&zhida_source=entity) 以及 /proc 文件系统而言具有以下优点：**

1. 为了使用 netlink，用户仅需要在 include/linux/netlink.h 中增加一个新类型的 netlink 协议定义即可， 如 #define NETLINK_MYTEST 17 然后，内核和用户态应用就可以立即通过 socket API 使用该 netlink 协议类型进行数据交换。但系统调用需要增加新的系统调用，ioctl 则需要增加设备或文件， 那需要不少代码，proc 文件系统则需要在 /proc 下添加新的文件或目录，那将使本来就混乱的 /proc 更加混乱。
1. netlink是一种[异步通信机制](https://zhida.zhihu.com/search?content_id=242331924&content_type=Article&match_order=1&q=%E5%BC%82%E6%AD%A5%E9%80%9A%E4%BF%A1%E6%9C%BA%E5%88%B6&zhida_source=entity)，在内核与用户态应用之间传递的消息保存在socket缓存队列中，发送消息只是把消息保存在接收者的socket的接 收队列，而不需要等待接收者收到消息，但系统调用与 ioctl 则是[同步通信](https://zhida.zhihu.com/search?content_id=242331924&content_type=Article&match_order=1&q=%E5%90%8C%E6%AD%A5%E9%80%9A%E4%BF%A1&zhida_source=entity)机制，如果传递的数据太长，将影响调度粒度。
1. 使用 netlink 的内核部分可以采用模块的方式实现，使用 netlink 的应用部分和内核部分没有编译时依赖，但系统调用就有依赖，而且新的系统调用的实现必须静态地连接到内核中，它无法在模块中实现，使用新系统调用的应用在编译时需要依赖内核。
1. netlink 支持多播，内核模块或应用可以把消息多播给一个netlink组，属于该neilink 组的任何内核模块或应用都能接收到该消息，[内核事件](https://zhida.zhihu.com/search?content_id=242331924&content_type=Article&match_order=2&q=%E5%86%85%E6%A0%B8%E4%BA%8B%E4%BB%B6&zhida_source=entity)向用户态的通知机制就使用了这一特性，任何对内核事件感兴趣的应用都能收到该子系统发送的内核事件，在 后面的文章中将介绍这一机制的使用。
1. 内核可以使用 netlink 首先发起会话，但系统调用和 ioctl 只能由用户应用发起调用。
1. netlink 使用标准的 socket API，因此很容易使用，但系统调用和 ioctl则需要专门的培训才能使用。

Netlink协议基于BSD socket和AF_NETLINK地址簇，使用32位的端口号寻址，每个Netlink协议通常与一个或一组内核服务/组件相关联，如NETLINK_ROUTE用于获取和设置路由与链路信息、NETLINK_KOBJECT_UEVENT用于内核向用户空间的udev进程发送通知等。

## 二、用户态数据结构

用户态应用使用标准的socket APIs， socket(), bind(), sendmsg(), recvmsg() 和 close() 就能很容易地使用 netlink socket，查询手册页可以了解这些函数的使用细节，本文只是讲解使用 netlink 的用户应该如何使用这些函数。注意，使用 netlink 的应用必须包含[头文件](https://zhida.zhihu.com/search?content_id=242331924&content_type=Article&match_order=1&q=%E5%A4%B4%E6%96%87%E4%BB%B6&zhida_source=entity) linux/netlink.h。当然 socket 需要的头文件也必不可少，sys/socket.h。Netlink通信跟常用UDP Socket通信类似，`struct sockaddr_nl`是netlink通信地址，跟普通`socket struct [sockaddr_in](https://zhida.zhihu.com/search?content_id=242331924&content_type=Article&match_order=1&q=sockaddr_in&zhida_source=entity)`类似。

**(1)struct sockaddr_nl结构：**

```c
struct sockaddr_nl {
     __kernel_sa_family_t    nl_family;  /* AF_NETLINK （跟AF_INET对应）*/
     unsigned short  nl_pad;     /* zero */
     __u32       nl_pid;     /* port ID  （通信端口号）*/
     __u32       nl_groups;  /* multicast groups mask */
};
```

**(2)struct nlmsghd 结构：**

```c
/* struct nlmsghd 是netlink消息头*/
struct nlmsghdr {   
    __u32       nlmsg_len;  /* Length of message including header */
    __u16       nlmsg_type; /* Message content */
    __u16       nlmsg_flags;    /* Additional flags */ 
    __u32       nlmsg_seq;  /* Sequence number */
    __u32       nlmsg_pid;  /* Sending process port ID */
};
```

nlmsg_type：消息状态，内核在include/uapi/linux/netlink.h中定义了以下4种通用的消息类型，它们分别是：

```c
#define NLMSG_NOOP      0x1 /* Nothing.     */
#define NLMSG_ERROR     0x2 /* Error        */
#define NLMSG_DONE      0x3 /* End of a dump    */
#define NLMSG_OVERRUN       0x4 /* Data lost        */
#define NLMSG_MIN_TYPE      0x10    /* < 0x10: reserved control messages */
```

nlmsg_flags：消息标记，它们用以表示消息的类型，如下:

```c
/* Flags values */
#define NLM_F_REQUEST       1   /* It is request message.   */
#define NLM_F_MULTI     2   /* Multipart message, terminated by NLMSG_DONE */
#define NLM_F_ACK       4   /* Reply with ack, with zero or error code */
#define NLM_F_ECHO      8   /* Echo this request        */
#define NLM_F_DUMP_INTR     16  /* Dump was inconsistent due to sequence change */

/* Modifiers to GET request */
#define NLM_F_ROOT  0x100   /* specify tree root    */
#define NLM_F_MATCH 0x200   /* return all matching  */
#define NLM_F_ATOMIC    0x400   /* atomic GET       */
#define NLM_F_DUMP  (NLM_F_ROOT|NLM_F_MATCH)

/* Modifiers to NEW request */
#define NLM_F_REPLACE   0x100   /* Override existing        */
#define NLM_F_EXCL  0x200   /* Do not touch, if it exists   */
#define NLM_F_CREATE    0x400   /* Create, if it does not exist */
#define NLM_F_APPEND    0x800   /* Add to end of list       */
```

**(3)struct msghdr [结构体](https://zhida.zhihu.com/search?content_id=242331924&content_type=Article&match_order=1&q=%E7%BB%93%E6%9E%84%E4%BD%93&zhida_source=entity)**

```c
struct iovec {                    /* Scatter/gather array items */
     void  *iov_base;              /* Starting address */
     size_t iov_len;               /* Number of bytes to transfer */
 };
  /* iov_base: iov_base指向数据包缓冲区，即参数buff，iov_len是buff的长度。msghdr中允许一次传递多个buff，以数组的形式组织在 msg_iov中，msg_iovlen就记录数组的长度 （即有多少个buff）  */
 struct msghdr {
     void         *msg_name;       /* optional address */
     socklen_t     msg_namelen;    /* size of address */
     struct iovec *msg_iov;        /* scatter/gather array */
     size_t        msg_iovlen;     /* # elements in msg_iov */
     void         *msg_control;    /* ancillary data, see below */
     size_t        msg_controllen; /* ancillary data buffer len */
     int           msg_flags;      /* flags on received message */
 };
```

为了创建一个 netlink socket，用户需要使用如下参数调用 socket()：

```c
socket(AF_NETLINK, SOCK_RAW, netlink_type)
```

第一个参数必须是 AF_NETLINK 或 PF_NETLINK，在 Linux 中，它们俩实际为一个东西，它表示要使用netlink，第二个参数必须是SOCK_RAW或SOCK_DGRAM， 第三个参数指定[netlink协议](https://zhida.zhihu.com/search?content_id=242331924&content_type=Article&match_order=1&q=netlink%E5%8D%8F%E8%AE%AE&zhida_source=entity)类型，如前面讲的用户自定义协议类型NETLINK_MYTEST， NETLINK_GENERIC是一个通用的协议类型，它是专门为用户使用的，因此，用户可以直接使用它，而不必再添加新的协议类型。内核预定义的协议类 型有：

```c
#define NETLINK_ROUTE           0       /* Routing/device hook                          */
#define NETLINK_W1              1       /* 1-wire subsystem                             */
#define NETLINK_USERSOCK        2       /* Reserved for user mode socket protocols      */
#define NETLINK_FIREWALL        3       /* Firewalling hook                             */
#define NETLINK_INET_DIAG       4       /* INET socket monitoring                       */
#define NETLINK_NFLOG           5       /* netfilter/iptables ULOG */
#define NETLINK_XFRM            6       /* ipsec */
#define NETLINK_SELINUX         7       /* SELinux event notifications */
#define NETLINK_ISCSI           8       /* Open-iSCSI */
#define NETLINK_AUDIT           9       /* auditing */
#define NETLINK_FIB_LOOKUP      10
#define NETLINK_CONNECTOR       11
#define NETLINK_NETFILTER       12      /* netfilter subsystem */
#define NETLINK_IP6_FW          13
#define NETLINK_DNRTMSG         14      /* DECnet routing messages */
#define NETLINK_KOBJECT_UEVENT  15      /* Kernel messages to userspace */
#define NETLINK_GENERIC         16
```

对于每一个netlink协议类型，可以有多达 32多播组，每一个[多播组](https://zhida.zhihu.com/search?content_id=242331924&content_type=Article&match_order=2&q=%E5%A4%9A%E6%92%AD%E7%BB%84&zhida_source=entity)用一个位表示，netlink 的多播特性使得发送消息给同一个组仅需要一次系统调用，因而对于需要多拨消息的应用而言，大大地降低了系统调用的次数。

函数 bind() 用于把一个打开的 netlink socket 与 netlink 源 socket 地址绑定在一起。netlink socket 的地址结构如下：

```c
struct sockaddr_nl
{
  sa_family_t    nl_family;
  unsigned short nl_pad;
  __u32          nl_pid;
  __u32          nl_groups;
};
```

字段 nl_family 必须设置为 AF_NETLINK 或着 PF_NETLINK，字段 nl_pad 当前没有使用，因此要总是设置为 0，字段 nl_pid 为接收或发送消息的进程的 ID，如果希望内核处理消息或多播消息，就把该字段设置为 0，否则设置为处理消息的进程 ID。字段 nl_groups 用于指定多播组，bind 函数用于把调用进程加入到该字段指定的多播组，如果设置为 0，表示调用者不加入任何多播组。

传递给 bind 函数的地址的 nl_pid 字段应当设置为本进程的进程 ID，这相当于 netlink socket 的本地地址。但是，对于一个进程的多个线程使用 netlink socket 的情况，字段 nl_pid 则可以设置为其它的值，如：

```c
pthread_self() << 16 | getpid();
```

因此字段 nl_pid 实际上未必是进程 ID,它只是用于区分不同的接收者或发送者的一个标识，用户可以根据自己需要设置该字段。函数 bind 的调用方式如下：

```c
bind(fd, (struct sockaddr*)&nladdr, sizeof(struct sockaddr_nl));
```

fd为前面的 socket 调用返回的文件描述符，参数 nladdr 为 struct sockaddr_nl 类型的地址。为了发送一个 netlink 消息给内核或其他用户态应用，需要填充目标 netlink socket 地址，此时，字段 nl_pid 和 nl_groups 分别表示接收消息者的进程 ID 与多播组。如果字段 nl_pid 设置为 0，表示消息接收者为内核或多播组，如果 nl_groups为 0，表示该消息为单播消息，否则表示多播消息。使用函数 sendmsg 发送 netlink 消息时还需要引用结构 struct msghdr、struct nlmsghdr 和 struct iovec，结构 struct msghdr 需如下设置：

```c
struct msghdr msg;
memset(&msg, 0, sizeof(msg));
msg.msg_name = (void *)&(nladdr);
msg.msg_namelen = sizeof(nladdr);
```

其中 nladdr 为消息接收者的 netlink 地址，struct nlmsghdr 为 netlink socket 自己的消息头，这用于[多路复用](https://zhida.zhihu.com/search?content_id=242331924&content_type=Article&match_order=1&q=%E5%A4%9A%E8%B7%AF%E5%A4%8D%E7%94%A8&zhida_source=entity)和多路分解 netlink 定义的所有协议类型以及其它一些控制，netlink 的内核实现将利用这个消息头来多路复用和多路分解已经其它的一些控制，因此它也被称为netlink [控制块](https://zhida.zhihu.com/search?content_id=242331924&content_type=Article&match_order=1&q=%E6%8E%A7%E5%88%B6%E5%9D%97&zhida_source=entity)。因此，应用在发送 netlink 消息时必须提供该消息头。

```c
struct nlmsghdr
{
  __u32 nlmsg_len;   /* Length of message */
  __u16 nlmsg_type;  /* Message type*/
  __u16 nlmsg_flags; /* Additional flags */
  __u32 nlmsg_seq;   /* Sequence number */
  __u32 nlmsg_pid;   /* Sending process PID */
};
```

字段 nlmsg_len 指定消息的总长度，包括紧跟该结构的数据部分长度以及该结构的大小，字段 nlmsg_type 用于应用内部定义消息的类型，它对 netlink 内核实现是透明的，因此大部分情况下设置为 0，字段 nlmsg_flags 用于设置消息标志，可用的标志包括：

```c
/* Flags values */
#define NLM_F_REQUEST           1       /* It is request message.       */
#define NLM_F_MULTI             2       /* Multipart message, terminated by NLMSG_DONE */
#define NLM_F_ACK               4       /* Reply with ack, with zero or error code */
#define NLM_F_ECHO              8       /* Echo this request            */
/* Modifiers to GET request */
#define NLM_F_ROOT      0x100   /* specify tree root    */
#define NLM_F_MATCH     0x200   /* return all matching  */
#define NLM_F_ATOMIC    0x400   /* atomic GET           */
#define NLM_F_DUMP      (NLM_F_ROOT|NLM_F_MATCH)
/* Modifiers to NEW request */
#define NLM_F_REPLACE   0x100   /* Override existing            */
#define NLM_F_EXCL      0x200   /* Do not touch, if it exists   */
#define NLM_F_CREATE    0x400   /* Create, if it does not exist */
#define NLM_F_APPEND    0x800   /* Add to end of list           */
```

1. 标志NLM_F_REQUEST用于表示消息是一个请求，所有应用首先发起的消息都应设置该标志。
1. 标志NLM_F_MULTI 用于指示该消息是一个多部分消息的一部分，后续的消息可以通过宏NLMSG_NEXT来获得。
1. 宏NLM_F_ACK表示该消息是前一个请求消息的响应，顺序号与进程ID可以把请求与响应关联起来。
1. 标志NLM_F_ECHO表示该消息是相关的一个包的回传。
1. 标志NLM_F_ROOT 被许多 netlink 协议的各种数据获取操作使用，该标志指示被请求的数据表应当整体返回用户应用，而不是一个条目一个条目地返回。有该标志的请求通常导致响应消息设置 NLM_F_MULTI标志。注意，当设置了该标志时，请求是协议特定的，因此，需要在字段 nlmsg_type 中指定协议类型。
1. 标志 NLM_F_MATCH 表示该协议特定的请求只需要一个数据子集，数据子集由指定的协议特定的过滤器来匹配。
1. 标志 NLM_F_ATOMIC 指示请求返回的数据应当原子地收集，这预防数据在获取期间被修改。
1. 标志 NLM_F_DUMP 未实现。
1. 标志 NLM_F_REPLACE 用于取代在数据表中的现有条目。
1. 标志 NLM_F_EXCL\_ 用于和 CREATE 和 APPEND 配合使用，如果条目已经存在，将失败。
1. 标志 NLM_F_CREATE 指示应当在指定的表中创建一个条目。
1. 标志 NLM_F_APPEND 指示在表末尾添加新的条目。

内核需要读取和修改这些标志，对于一般的使用，用户把它设置为 0 就可以，只是一些高级应用（如 netfilter 和路由 daemon 需要它进行一些复杂的操作），字段 nlmsg_seq 和 nlmsg_pid 用于[应用追踪](https://zhida.zhihu.com/search?content_id=242331924&content_type=Article&match_order=1&q=%E5%BA%94%E7%94%A8%E8%BF%BD%E8%B8%AA&zhida_source=entity)消息，前者表示顺序号，后者为消息来源进程 ID。下面是一个示例：

```c
#define MAX_MSGSIZE 1024
char buffer[] = "An example message";
struct nlmsghdr nlhdr;
nlhdr = (struct nlmsghdr *)malloc(NLMSG_SPACE(MAX_MSGSIZE));
strcpy(NLMSG_DATA(nlhdr),buffer);
nlhdr->nlmsg_len = NLMSG_LENGTH(strlen(buffer));
nlhdr->nlmsg_pid = getpid();  /* self pid */
nlhdr->nlmsg_flags = 0;
```

结构 struct iovec 用于把多个消息通过一次系统调用来发送，下面是该结构使用示例：

```c
struct iovec iov;
iov.iov_base = (void *)nlhdr;
iov.iov_len = nlh->nlmsg_len;
msg.msg_iov = &iov;
msg.msg_iovlen = 1;
```

在完成以上步骤后，消息就可以通过下面语句直接发送：

```c
sendmsg(fd, &msg, 0);
```

应用接收消息时需要首先分配一个足够大的缓存来保存消息头以及消息的数据部分，然后填充消息头，添完后就可以直接调用函数 recvmsg() 来接收。

```c
#define MAX_NL_MSG_LEN 1024
struct sockaddr_nl nladdr;
struct msghdr msg;
struct iovec iov;
struct nlmsghdr * nlhdr;
nlhdr = (struct nlmsghdr *)malloc(MAX_NL_MSG_LEN);
iov.iov_base = (void *)nlhdr;
iov.iov_len = MAX_NL_MSG_LEN;
msg.msg_name = (void *)&(nladdr);
msg.msg_namelen = sizeof(nladdr);
msg.msg_iov = &iov;
msg.msg_iovlen = 1;
recvmsg(fd, &msg, 0);
```

注意：fd为socket调用打开的netlink socket描述符，在消息接收后，nlhdr指向接收到的消息的消息头，nladdr保存了接收到的消息的目标地址，宏NLMSG_DATA(nlhdr)返回指向消息的数据部分的指针。

在linux/netlink.h中定义了一些方便对消息进行处理的宏，这些宏包括：

```c
#define NLMSG_ALIGNTO   4
#define NLMSG_ALIGN(len) ( ((len)+NLMSG_ALIGNTO-1) & ~(NLMSG_ALIGNTO-1) )
```

宏NLMSG_ALIGN(len)用于得到不小于len且字节对齐的最小数值。

```c
#define NLMSG_LENGTH(len) ((len)+NLMSG_ALIGN(sizeof(struct nlmsghdr)))
```

宏NLMSG_LENGTH(len)用于计算数据部分长度为len时实际的消息长度。它一般用于分配消息缓存。

```c
#define NLMSG_SPACE(len) NLMSG_ALIGN(NLMSG_LENGTH(len))
```

宏NLMSG_SPACE(len)返回不小于NLMSG_LENGTH(len)且字节对齐的最小数值，它也用于分配消息缓存。

```c
#define NLMSG_DATA(nlh)  ((void*)(((char*)nlh) + NLMSG_LENGTH(0)))
```

宏NLMSG_DATA(nlh)用于取得消息的数据部分的首地址，设置和读取消息数据部分时需要使用该宏。

```c
#define NLMSG_NEXT(nlh,len)      ((len) -= NLMSG_ALIGN((nlh)->nlmsg_len), \
                      (struct nlmsghdr*)(((char*)(nlh)) + NLMSG_ALIGN((nlh)->nlmsg_len)))
```

宏NLMSG_NEXT(nlh,len)用于得到下一个消息的首地址，同时len也减少为剩余消息的总长度，该宏一般在一个消息被分成几个部分发送或接收时使用。

```c
#define NLMSG_OK(nlh,len) ((len) >= (int)sizeof(struct nlmsghdr) && \
                           (nlh)->nlmsg_len >= sizeof(struct nlmsghdr) && \
                           (nlh)->nlmsg_len <= (len))
```

宏NLMSG_OK(nlh,len)用于判断消息是否有len这么长。

```c
#define NLMSG_PAYLOAD(nlh,len) ((nlh)->nlmsg_len - NLMSG_SPACE((len)))
```

宏NLMSG_PAYLOAD(nlh,len)用于返回payload的长度，函数close用于关闭打开的netlink socket。

## 三、netlink内核数据结构

**(1)netlink消息类型：**

```c
#define NETLINK_ROUTE       0   /* Routing/device hook              */
#define NETLINK_UNUSED      1   /* Unused number                */
#define NETLINK_USERSOCK    2   /* Reserved for user mode socket protocols  */
#define NETLINK_FIREWALL    3   /* Unused number, formerly ip_queue     */
#define NETLINK_SOCK_DIAG   4   /* socket monitoring                */
#define NETLINK_NFLOG       5   /* netfilter/iptables ULOG */
#define NETLINK_XFRM        6   /* ipsec */
#define NETLINK_SELINUX     7   /* SELinux event notifications */
#define NETLINK_ISCSI       8   /* Open-iSCSI */
#define NETLINK_AUDIT       9   /* auditing */
#define NETLINK_FIB_LOOKUP  10  
#define NETLINK_CONNECTOR   11
#define NETLINK_NETFILTER   12  /* netfilter subsystem */
#define NETLINK_IP6_FW      13
#define NETLINK_DNRTMSG     14  /* DECnet routing messages */
#define NETLINK_KOBJECT_UEVENT  15  /* Kernel messages to userspace */
#define NETLINK_GENERIC     16
/* leave room for NETLINK_DM (DM Events) */
#define NETLINK_SCSITRANSPORT   18  /* SCSI Transports */
#define NETLINK_ECRYPTFS    19
#define NETLINK_RDMA        20
#define NETLINK_CRYPTO      21  /* Crypto layer */

#define NETLINK_INET_DIAG   NETLINK_SOCK_DIAG

#define MAX_LINKS 32
```

**(2)netlink常用宏：**

```c
#define NLMSG_ALIGNTO   4U
/* 宏NLMSG_ALIGN(len)用于得到不小于len且字节对齐的最小数值 */
#define NLMSG_ALIGN(len) ( ((len)+NLMSG_ALIGNTO-1) & ~(NLMSG_ALIGNTO-1) )

/* Netlink 头部长度 */
#define NLMSG_HDRLEN     ((int) NLMSG_ALIGN(sizeof(struct nlmsghdr)))

/* 计算消息数据len的真实消息长度（消息体 +　消息头）*/
#define NLMSG_LENGTH(len) ((len) + NLMSG_HDRLEN)

/* 宏NLMSG_SPACE(len)返回不小于NLMSG_LENGTH(len)且字节对齐的最小数值 */
#define NLMSG_SPACE(len) NLMSG_ALIGN(NLMSG_LENGTH(len))

/* 宏NLMSG_DATA(nlh)用于取得消息的数据部分的首地址，设置和读取消息数据部分时需要使用该宏 */
#define NLMSG_DATA(nlh)  ((void*)(((char*)nlh) + NLMSG_LENGTH(0)))

/* 宏NLMSG_NEXT(nlh,len)用于得到下一个消息的首地址, 同时len 变为剩余消息的长度 */
#define NLMSG_NEXT(nlh,len)  ((len) -= NLMSG_ALIGN((nlh)->nlmsg_len), \
                  (struct nlmsghdr*)(((char*)(nlh)) + NLMSG_ALIGN((nlh)->nlmsg_len)))

/* 判断消息是否 >len */
#define NLMSG_OK(nlh,len) ((len) >= (int)sizeof(struct nlmsghdr) && \
               (nlh)->nlmsg_len >= sizeof(struct nlmsghdr) && \
               (nlh)->nlmsg_len <= (len))

/* NLMSG_PAYLOAD(nlh,len) 用于返回payload的长度*/
#define NLMSG_PAYLOAD(nlh,len) ((nlh)->nlmsg_len - NLMSG_SPACE((len)))
```

**(3)netlink 内核常用函数**

netlink_kernel_create内核函数用于创建内核socket与用户态通信

```c
static inline struct sock *
netlink_kernel_create(struct net *net, int unit, struct netlink_kernel_cfg *cfg)
/* net: net指向所在的网络命名空间, 一般默认传入的是&init_net(不需要定义);  定义在net_namespace.c(extern struct net init_net);
   unit：netlink协议类型
   cfg： cfg存放的是netlink内核配置参数（如下）
*/

/* optional Netlink kernel configuration parameters */
struct netlink_kernel_cfg {
    unsigned int    groups;  
    unsigned int    flags;  
    void        (*input)(struct sk_buff *skb); /* input 回调函数 */
    struct mutex    *cb_mutex; 
    void        (*bind)(int group); 
    bool        (*compare)(struct net *net, struct sock *sk);
};
```

**(4)单播netlink_unicast() 和 多播netlink_broadcast()**

```c
/* 发送单播消息 */
extern int netlink_unicast(struct sock *ssk, struct sk_buff *skb, __u32 portid, int nonblock);
/*
 ssk: netlink socket 
 skb: skb buff 指针
 portid： 通信的端口号
 nonblock：表示该函数是否为非阻塞，如果为1，该函数将在没有接收缓存可利用时立即返回，而如果为0，该函数在没有接收缓存可利用定时睡眠
*/

/* 发送多播消息 */
extern int netlink_broadcast(struct sock *ssk, struct sk_buff *skb, __u32 portid,
                 __u32 group, gfp_t allocation);
/* 
   ssk: 同上（对应netlink_kernel_create 返回值）、
   skb: 内核skb buff
   portid： 端口id
   group: 是所有目标多播组对应掩码的"OR"操作的合值。
   allocation: 指定内核内存分配方式，通常GFP_ATOMIC用于中断上下文，而GFP_KERNEL用于其他场合。这个参数的存在是因为该API可能需要分配一个或多个缓冲区来对多播消息进行clone
*/
```

## **四、netlink内核API**

netlink的内核实现在.c文件net/core/af_netlink.c中，内核模块要想使用netlink，也必须包含头文件linux /netlink.h。内核使用netlink需要专门的API，这完全不同于用户态应用对netlink的使用。如果用户需要增加新的netlink协 议类型，必须通过修改linux/netlink.h来实现，当然，目前的netlink实现已经包含了一个通用的协议类型 NETLINK_GENERIC以方便用户使用，用户可以直接使用它而不必增加新的协议类型。前面讲到，为了增加新的netlink协议类型，用户仅需增 加如下定义到linux/netlink.h就可以：

```c
#define NETLINK_MYTEST  17
```

只要增加这个定义之后，用户就可以在内核的任何地方引用该协议，在内核中，为了创建一个netlink socket用户需要调用如下函数：

```cpp
struct sock *
netlink_kernel_create(int unit, void (*input)(struct sock *sk, int len));
```

参数unit表示netlink协议类型，如NETLINK_MYTEST，参数input则为内核模块定义的netlink消息处理函数，当有消 息到达这个netlink socket时，该input[函数指针](https://zhida.zhihu.com/search?content_id=242331924&content_type=Article&match_order=1&q=%E5%87%BD%E6%95%B0%E6%8C%87%E9%92%88&zhida_source=entity)就会被引用。函数指针input的参数sk实际上就是函数netlink_kernel_create返回的 struct sock指针，sock实际是socket的一个内核表示数据结构，用户态应用创建的socket在内核中也会有一个struct sock结构来表示。下面是一个input函数的示例：

```c
void input (struct sock *sk, int len)
{
 struct sk_buff *skb;
 struct nlmsghdr *nlh = NULL;
 u8 *data = NULL;
 while ((skb = skb_dequeue(&sk->receive_queue)) 
       != NULL) {
 /* process netlink message pointed by skb->data */
 nlh = (struct nlmsghdr *)skb->data;
 data = NLMSG_DATA(nlh);
 /* process netlink message with header pointed by 
  * nlh and data pointed by data
  */
 }   
}
```

函数input()会在发送进程执行sendmsg()时被调用，这样处理消息比较及时，但是，如果消息特别长时，这样处理将增加系统调用 sendmsg()的执行时间，对于这种情况，可以定义一个内核线程专门负责消息接收，而函数input的工作只是唤醒该内核线程，这样sendmsg将 很快返回。

函数skb = skb_dequeue(&sk->receive_queue)用于取得socket sk的接收队列上的消息，返回为一个struct sk_buff的结构，skb->data指向实际的netlink消息。

函数skb_recv_datagram(nl_sk)也用于在netlink socket nl_sk上接收消息，与skb_dequeue的不同指出是，如果socket的接收队列上没有消息，它将导致调用进程睡眠在等待队列 nl_sk->sk_sleep，因此它必须在[进程上下文](https://zhida.zhihu.com/search?content_id=242331924&content_type=Article&match_order=1&q=%E8%BF%9B%E7%A8%8B%E4%B8%8A%E4%B8%8B%E6%96%87&zhida_source=entity)使用，刚才讲的内核线程就可以采用这种方式来接收消息。

下面的函数input就是这种使用的示例：

```c
void input (struct sock *sk, int len)
{
  wake_up_interruptible(sk->sk_sleep);
}
```

当内核中发送netlink消息时，也需要设置目标地址与源地址，而且内核中消息是通过struct sk_buff来管理的， linux/netlink.h中定义了一个宏：

```c
#define NETLINK_CB(skb)         (*(struct netlink_skb_parms*)&((skb)->cb))
```

来方便消息的地址设置。下面是一个消息地址设置的例子：

```c
NETLINK_CB(skb).pid = 0;
NETLINK_CB(skb).dst_pid = 0;
NETLINK_CB(skb).dst_group = 1;
```

字段pid表示消息发送者进程ID，也即源地址，对于内核，它为 0， dst_pid 表示消息接收者进程 ID，也即目标地址，如果目标为组或内核，它设置为 0，否则 dst_group 表示目标组地址，如果它目标为某一进程或内核，dst_group 应当设置为 0。

在内核中，模块调用函数 netlink_unicast 来发送单播消息：

```c
int netlink_unicast(struct sock *sk, struct sk_buff *skb, u32 pid, int nonblock);
```

参数sk为函数netlink_kernel_create()返回的socket，参数skb存放消息，它的data字段指向要发送的 netlink消息结构，而skb的控制块保存了消息的地址信息，前面的宏NETLINK_CB(skb)就用于方便设置该控制块， 参数pid为接收消息进程的pid，参数nonblock表示该函数是否为非阻塞，如果为1，该函数将在没有接收缓存可利用时立即返回，而如果为0，该函 数在没有接收缓存可利用时睡眠。

内核模块或子系统也可以使用函数netlink_broadcast来发送广播消息：

```c
void netlink_broadcast(struct sock *sk, struct sk_buff *skb, u32 pid, u32 group, int allocation);
```

前面的三个参数与netlink_unicast相同，参数group为接收消息的多播组，该参数的每一个代表一个多播组，因此如果发送给多个多播 组，就把该参数设置为多个多播组组ID的位或。参数allocation为内核内存分配类型，一般地为GFP_ATOMIC或 GFP_KERNEL，GFP_ATOMIC用于原子的上下文（即不可以睡眠），而GFP_KERNEL用于非原子上下文。

在内核中使用函数sock_release来释放函数netlink_kernel_create()创建的netlink socket：

```c
void sock_release(struct socket * sock);
```

注意函数netlink_kernel_create()返回的类型为struct sock，因此函数sock_release应该这种调用：

```c
sock_release(sk->sk_socket);
```

sk为函数netlink_kernel_create()的返回值。在源代码包中 给出了一个使用 netlink 的示例，它包括一个内核模块 netlink-exam-kern.c 和两个应用程序 netlink-exam-user-recv.c, netlink-exam-user-send.c。内核模块必须先插入到内核，然后在一个终端上运行用户态接收程序，在另一个终端上运行用户态发送程 序，发送程序读取参数指定的[文本文件](https://zhida.zhihu.com/search?content_id=242331924&content_type=Article&match_order=1&q=%E6%96%87%E6%9C%AC%E6%96%87%E4%BB%B6&zhida_source=entity)并把它作为 netlink 消息的内容发送给内核模块，内核模块接受该消息保存到内核缓存中，它也通过proc接口出口到 procfs，因此用户也能够通过 /proc/netlink_exam_buffer 看到全部的内容，同时内核也把该消息发送给用户态接收程序，用户态接收程序将把接收到的内容输出到屏幕上。

## 五、netlink实例

**(1)用户态程序 （sendto()， recvfrom()）**

```c
#include <stdio.h>
#include <stdlib.h>
#include <sys/socket.h>
#include <string.h>
#include <linux/netlink.h>
#include <stdint.h>
#include <unistd.h>
#include <errno.h>

#define NETLINK_TEST    30
#define MSG_LEN            125
#define MAX_PLOAD        125

typedef struct _user_msg_info
{
    struct nlmsghdr hdr;
    char  msg[MSG_LEN];
} user_msg_info;

int main(int argc, char **argv)
{
    int skfd;
    int ret;
    user_msg_info u_info;
    socklen_t len;
    struct nlmsghdr *nlh = NULL;
    struct sockaddr_nl saddr, daddr;
    char *umsg = "hello netlink!!";

    /* 创建NETLINK socket */
    skfd = socket(AF_NETLINK, SOCK_RAW, NETLINK_TEST);
    if(skfd == -1)
    {
        perror("create socket error\n");
        return -1;
    }

    memset(&saddr, 0, sizeof(saddr));
    saddr.nl_family = AF_NETLINK; //AF_NETLINK
    saddr.nl_pid = 100;  //端口号(port ID) 
    saddr.nl_groups = 0;
    if(bind(skfd, (struct sockaddr *)&saddr, sizeof(saddr)) != 0)
    {
        perror("bind() error\n");
        close(skfd);
        return -1;
    }

    memset(&daddr, 0, sizeof(daddr));
    daddr.nl_family = AF_NETLINK;
    daddr.nl_pid = 0; // to kernel 
    daddr.nl_groups = 0;

    nlh = (struct nlmsghdr *)malloc(NLMSG_SPACE(MAX_PLOAD));
    memset(nlh, 0, sizeof(struct nlmsghdr));
    nlh->nlmsg_len = NLMSG_SPACE(MAX_PLOAD);
    nlh->nlmsg_flags = 0;
    nlh->nlmsg_type = 0;
    nlh->nlmsg_seq = 0;
    nlh->nlmsg_pid = saddr.nl_pid; //self port

    memcpy(NLMSG_DATA(nlh), umsg, strlen(umsg));
    ret = sendto(skfd, nlh, nlh->nlmsg_len, 0, (struct sockaddr *)&daddr, sizeof(struct sockaddr_nl));
    if(!ret)
    {
        perror("sendto error\n");
        close(skfd);
        exit(-1);
    }
    printf("send kernel:%s\n", umsg);

    memset(&u_info, 0, sizeof(u_info));
    len = sizeof(struct sockaddr_nl);
    ret = recvfrom(skfd, &u_info, sizeof(user_msg_info), 0, (struct sockaddr *)&daddr, &len);
    if(!ret)
    {
        perror("recv form kernel error\n");
        close(skfd);
        exit(-1);
    }

    printf("from kernel:%s\n", u_info.msg);
    close(skfd);

    free((void *)nlh);
    return 0;
}
```

**(2)Netlink 内核模块代码**

```c
#include <linux/init.h>
#include <linux/module.h>
#include <linux/types.h>
#include <net/sock.h>
#include <linux/netlink.h>

#define NETLINK_TEST     30
#define MSG_LEN            125
#define USER_PORT        100

MODULE_LICENSE("GPL");
MODULE_AUTHOR("zhangwj");
MODULE_DESCRIPTION("netlink example");

struct sock *nlsk = NULL;
extern struct net init_net;

int send_usrmsg(char *pbuf, uint16_t len)
{
    struct sk_buff *nl_skb;
    struct nlmsghdr *nlh;

    int ret;

    /* 创建sk_buff 空间 */
    nl_skb = nlmsg_new(len, GFP_ATOMIC);
    if(!nl_skb)
    {
        printk("netlink alloc failure\n");
        return -1;
    }

    /* 设置netlink消息头部 */
    nlh = nlmsg_put(nl_skb, 0, 0, NETLINK_TEST, len, 0);
    if(nlh == NULL)
    {
        printk("nlmsg_put failaure \n");
        nlmsg_free(nl_skb);
        return -1;
    }

    /* 拷贝数据发送 */
    memcpy(nlmsg_data(nlh), pbuf, len);
    ret = netlink_unicast(nlsk, nl_skb, USER_PORT, MSG_DONTWAIT);

    return ret;
}

static void netlink_rcv_msg(struct sk_buff *skb)
{
    struct nlmsghdr *nlh = NULL;
    char *umsg = NULL;
    char *kmsg = "hello users!!!";

    if(skb->len >= nlmsg_total_size(0))
    {
        nlh = nlmsg_hdr(skb);
        umsg = NLMSG_DATA(nlh);
        if(umsg)
        {
            printk("kernel recv from user: %s\n", umsg);
            send_usrmsg(kmsg, strlen(kmsg));
        }
    }
}

struct netlink_kernel_cfg cfg = { 
        .input  = netlink_rcv_msg, /* set recv callback */
};  

int test_netlink_init(void)
{
    /* create netlink socket */
    nlsk = (struct sock *)netlink_kernel_create(&init_net, NETLINK_TEST, &cfg);
    if(nlsk == NULL)
    {   
        printk("netlink_kernel_create error !\n");
        return -1; 
    }   
    printk("test_netlink_init\n");
    
    return 0;
}

void test_netlink_exit(void)
{
    if (nlsk){
        netlink_kernel_release(nlsk); /* release ..*/
        nlsk = NULL;
    }   
    printk("test_netlink_exit!\n");
}

module_init(test_netlink_init);
module_exit(test_netlink_exit);
```

**(3)Makeflie**

```c
#
#Desgin of Netlink
#

MODULE_NAME :=netlink_test
obj-m :=$(MODULE_NAME).o

KERNELDIR ?= /lib/modules/$(shell uname -r)/build
PWD := $(shell pwd)

all:
    $(MAKE) -C $(KERNELDIR) M=$(PWD)

clean:
    $(MAKE) -C $(KERNELDIR) M=$(PWD) clean
```

**(4)运行结果**

首先将编译出来的Netlink内核模块插入到系统当中（insmod netlink_test.ko），然后运行应用程序，可以看到如下输出：

```c
# 应用程序打印
send kernel:hello netlink!!
from kernel:hello users!!!
# 内核打印 
[25024.276345] test_netlink_init
[25117.548350] kernel recv from user: hello netlink!!
```

**往期回顾：**

- [2024 技术展望 |](https://link.zhihu.com/?target=http%3A//mp.weixin.qq.com/s%3F__biz%3DMzg4NDQ0OTI4Ng%3D%3D%26mid%3D2247487749%26idx%3D1%26sn%3De57e6f3df526b7ad78313d9428e55b6b%26chksm%3Dcfb9586cf8ced17a8c7830e380a45ce080c2b8258e145f5898503a779840a5fcfec3e8f8fa9a%26scene%3D21%23wechat_redirect)[C/C++发展方向（强烈推荐！！）](https://link.zhihu.com/?target=http%3A//mp.weixin.qq.com/s%3F__biz%3DMzg4NDQ0OTI4Ng%3D%3D%26mid%3D2247487749%26idx%3D1%26sn%3De57e6f3df526b7ad78313d9428e55b6b%26chksm%3Dcfb9586cf8ced17a8c7830e380a45ce080c2b8258e145f5898503a779840a5fcfec3e8f8fa9a%26scene%3D21%23wechat_redirect)
- [2024 技术展望 |](https://link.zhihu.com/?target=http%3A//mp.weixin.qq.com/s%3F__biz%3DMzIwNzA1OTA2OQ%3D%3D%26mid%3D2657215309%26idx%3D1%26sn%3D7c25ec3689a7338ca7e58719b8f98ad0%26chksm%3D8c8d625fbbfaeb492d4a10fccf0d6cb490e9b4ff642a0cbc2da425193d796dbe4c3f8b628d14%26scene%3D21%23wechat_redirect) [Linux内核源码分析（强烈推荐收藏！](https://link.zhihu.com/?target=http%3A//mp.weixin.qq.com/s%3F__biz%3DMzg4NDQ0OTI4Ng%3D%3D%26mid%3D2247487832%26idx%3D1%26sn%3Dbf0468e26f353306c743c4d7523ebb07%26chksm%3Dcfb95831f8ced127ca94eb61e6508732576bb150b2fb2047664f8256b3da284d0e53e2f792dc%26scene%3D21%23wechat_redirect)[）](https://link.zhihu.com/?target=http%3A//mp.weixin.qq.com/s%3F__biz%3DMzg4NDQ0OTI4Ng%3D%3D%26mid%3D2247487832%26idx%3D1%26sn%3Dbf0468e26f353306c743c4d7523ebb07%26chksm%3Dcfb95831f8ced127ca94eb61e6508732576bb150b2fb2047664f8256b3da284d0e53e2f792dc%26scene%3D21%23wechat_redirect)
- [2024 技术展望 |](https://link.zhihu.com/?target=http%3A//mp.weixin.qq.com/s%3F__biz%3DMzIwNzA1OTA2OQ%3D%3D%26mid%3D2657215314%26idx%3D1%26sn%3D464575d273ede0a461fdffb22c8407f8%26chksm%3D8c8d6240bbfaeb5656784cc9ccbcd4411828ef0178274435f9ab7c770dde02188446cd34c439%26scene%3D21%23wechat_redirect) [从菜鸟到大师！用Qt编写出惊艳世界的应用](https://link.zhihu.com/?target=http%3A//mp.weixin.qq.com/s%3F__biz%3DMzg4NDQ0OTI4Ng%3D%3D%26mid%3D2247488117%26idx%3D1%26sn%3Da83d661165a3840fbb23d0e62b5f303a%26chksm%3Dcfb95b1cf8ced20ae63206fe25891d9a37ffe76fd695ef55b5506c83aad387d55c4032cb7e4f%26scene%3D21%23wechat_redirect)
- [2024 技术展望 |](https://link.zhihu.com/?target=http%3A//mp.weixin.qq.com/s%3F__biz%3DMzIwNzA1OTA2OQ%3D%3D%26mid%3D2657215371%26idx%3D1%26sn%3Dc20f0717c8c66decb63a5bcd342c786e%26chksm%3D8c8d6299bbfaeb8f4b8d3aa95b8d4ab7c9ca4596152accdfb9b2f3a492352908dcc8ddcbdffa%26scene%3D21%23wechat_redirect) [探索存储全栈开发：构建高效可靠的数据存储系统](https://link.zhihu.com/?target=http%3A//mp.weixin.qq.com/s%3F__biz%3DMzg4NDQ0OTI4Ng%3D%3D%26mid%3D2247487696%26idx%3D1%26sn%3Db5ebe830ddb6798ac5bf6db4a8d5d075%26chksm%3Dcfb959b9f8ced0af76710c070a6db2677fb359af735e79c6378e82ead570aa1ce5350a146793%26scene%3D21%23wechat_redirect)
- [2024 技术展望 |](https://link.zhihu.com/?target=http%3A//mp.weixin.qq.com/s%3F__biz%3DMzIwNzA1OTA2OQ%3D%3D%26mid%3D2657215412%26idx%3D1%26sn%3Dd3c64ac21056b84814db9903b656853a%26chksm%3D8c8d62a6bbfaebb09df1f727e201c401a1663596a719fa9cb4c7859158e62ebedb5bd5f13b8f%26scene%3D21%23wechat_redirect) [突破性能瓶颈：释放DPDK带来的无尽潜力](https://link.zhihu.com/?target=http%3A//mp.weixin.qq.com/s%3F__biz%3DMzg4NDQ0OTI4Ng%3D%3D%26mid%3D2247488150%26idx%3D1%26sn%3D2cc5ace4391d4eed060fa463fc73dd92%26chksm%3Dcfb95bfff8ced2e9589eecd66e7c6e906f28626bef6c1bce3fa19ae2d1404baa25ba3ebfedaa%23rd)
- [2024 技术展望 |](https://link.zhihu.com/?target=http%3A//mp.weixin.qq.com/s%3F__biz%3DMzIwNzA1OTA2OQ%3D%3D%26mid%3D2657215412%26idx%3D1%26sn%3Dd3c64ac21056b84814db9903b656853a%26chksm%3D8c8d62a6bbfaebb09df1f727e201c401a1663596a719fa9cb4c7859158e62ebedb5bd5f13b8f%26scene%3D21%23wechat_redirect) [嵌入式音视频技术解析：从原理到项目应用](https://link.zhihu.com/?target=http%3A//mp.weixin.qq.com/s%3F__biz%3DMzg4NDQ0OTI4Ng%3D%3D%26mid%3D2247488135%26idx%3D1%26sn%3Ddcc56a3af5401c338944b9b94f24c699%26chksm%3Dcfb95beef8ced2f8231c647033188be3ded88cbacf7d75feab3613f4d7f8fd8e4c8106ac88cf%26scene%3D21%23wechat_redirect)
- [2024 技术展望 |](https://link.zhihu.com/?target=http%3A//mp.weixin.qq.com/s%3F__biz%3DMzIwNzA1OTA2OQ%3D%3D%26mid%3D2657215412%26idx%3D1%26sn%3Dd3c64ac21056b84814db9903b656853a%26chksm%3D8c8d62a6bbfaebb09df1f727e201c401a1663596a719fa9cb4c7859158e62ebedb5bd5f13b8f%26scene%3D21%23wechat_redirect) [C++游戏后端开发，基于魔兽开源后端框架](https://link.zhihu.com/?target=http%3A//mp.weixin.qq.com/s%3F__biz%3DMzg4NDQ0OTI4Ng%3D%3D%26mid%3D2247486920%26idx%3D1%26sn%3D82ca8a72473850e431c01e7c39d8c8c0%26chksm%3Dcfb944a1f8cecdb7051863f58d334ff3dea8afae1ecd8bc693287b2f483d2ce8f94eed565253%23rd)

![动图封面](https://pic3.zhimg.com/v2-937a1ab104f7526be077ec69d7c41c42_b.jpg)

______________________________________________________________________

发布于 2024-04-22 17:34・IP 属地湖南

\[

进程间通信

\](https://www.zhihu.com/topic/20138175)

\[

Linux

\](https://www.zhihu.com/topic/19554300)

\[

C / C++

\](https://www.zhihu.com/topic/19601705)

​赞同 22​​1 条评论

​分享

​喜欢​收藏​申请转载

​

赞同 22

​

分享

![](https://pic1.zhimg.com/v2-abed1a8c04700ba7d72b45195223e0ff_l.jpg?source=32738c0c&needBackground=1)

理性发言，友善互动

1 条评论

默认

最新

[![阿源](https://pic1.zhimg.com/v2-5b6e04088f424eb9932d99e2977b0fee_l.jpg?source=06d4cd63)](https://www.zhihu.com/people/349eecc6baa51961f9e840864173f5d8)

[阿源](https://www.zhihu.com/people/349eecc6baa51961f9e840864173f5d8)

写得很清详细！

04-25 · 四川

​回复​喜欢

### 文章被以下专栏收录

\[

![Linux内核](https://picx.zhimg.com/v2-f111d7ee1c41944859e975a712c0883b_l.jpg?source=172ae18b)

\](https://www.zhihu.com/column/c_1290342714786152448)

## \[

Linux内核

\](https://www.zhihu.com/column/c_1290342714786152448)

一种开源电脑操作系统内核。C语言写成

### 推荐阅读

\[

![一文了解linux 内核与用户空间通信之netlink使用方法](https://picx.zhimg.com/v2-079893de15b959258de121a8195bd22e_250x0.jpg?source=172ae18b)

# 一文了解linux 内核与用户空间通信之netlink使用方法

极致Linux内核

\](https://zhuanlan.zhihu.com/p/552291792)\[

![由浅入深探讨Linux进程间通信（上篇）](https://pic1.zhimg.com/v2-24c6236e675e9870f1505cd35cf564cb_250x0.jpg?source=172ae18b)

# 由浅入深探讨Linux进程间通信（上篇）

物联网心球

\](https://zhuanlan.zhihu.com/p/669988948)\[

# Linux进程间通信的实现原理？

作为Linux应用程序的开发人员，对Linux的进程间通信方式肯定是了如指掌，平时的开发中应该会大量的使用到。当你迅速的在键盘上按下【CTRL+C】终止掉一个正在运行中的命令时，你有没有仔细的…

point...发表于linux...

\](https://zhuanlan.zhihu.com/p/69553562)\[

# Linux进程间通信方式有哪些？

前言进程能够单独运行并且完成一些任务，但是也经常免不了和其他进程传输数据或互相通知消息，即需要进行通信，本文将简单介绍一些进程之间相互通信的技术--进程间通信（InterProcess Commu…

守望

\](https://zhuanlan.zhihu.com/p/63916424)
