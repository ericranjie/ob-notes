原创 董旭 技术简说
 _2021年12月31日 08:51_


**前面文章分析了sockfs文件系统、socket底层的原理：**  
1、[浅析Linux sockfs文件系统](http://mp.weixin.qq.com/s?__biz=Mzg5MTU1ODgyMA==&mid=2247484140&idx=1&sn=2d9bd9abf3cf2c5c5e37fe0875d8bc69&chksm=cfcacc27f8bd453123a37ee9c877384340069da85bff6ceb624447d7d57bfab628b7d5be794c&scene=21#wechat_redirect)
2、[Linux内核 | socket底层的来龙去脉](http://mp.weixin.qq.com/s?__biz=Mzg5MTU1ODgyMA==&mid=2247484240&idx=1&sn=38b7bc12fb6720b389b16bfb158bf329&chksm=cfcacd9bf8bd448d7325d8d94d80b7a019d0389cab7c8cfeddd441ca8c28da0f0882a6311cfb&scene=21#wechat_redirect)

**在两章的基础上分析Netlink机制，并编写使用Netlink机制的案例**

**一、Netlink简介**

Linux中内核与用户空间数据交换有很多种方式，如系统调用、procfs、debugfs等，这些通信方式都是同步通信方式，由用户态主动发起向内核态的通信，内核无法主动发起通信。而**Netlink是一种异步全双工的通信方式，支持由内核态主动发起通信**，内核为Netlink通信提供了一组特殊的API接口，用户态则基于socket API，内核发送的数据会保存在接收进程socket 的接收缓存中，由接收进程处理。Netlink也是网络应用程序与内核通信的最常用接口，如路由damon、防火墙(NETLINK_FIREWALL)等。Netlink机制作为一种**内核与用户空间通信**的机制，同时也**是一套IP服务协议，代表着一种特殊的socket通信方式，对于Linux内核与用户空间进行双向数据传输是非常好的方式。**

Netlink通信机制的简易流程如下图所示，**本文****将从用户态通过系统调用创建netlink socket、内核态调用netlink_kernel_create创建内核Netlink套接字进行分析，并在文章最后编写使用Netlink的案例。**  

![图片](https://mmbiz.qpic.cn/mmbiz_png/EjWxxIM2EQJ2wBWrVMdUI34qGUYYibzJhZJftDCpZqhibqh2icBo8TSR05OBo4TnjXVCqHsB1BXSQOaeSrgVLNr9g/640?wx_fmt=png&wxfrom=13&tp=wxpic)

**二、用户态创建Netlink套接字**

**1、用户态创建Netlink socket 的相关分析**

在[Linux内核 | socket底层的来龙去脉](http://mp.weixin.qq.com/s?__biz=Mzg5MTU1ODgyMA==&mid=2247484240&idx=1&sn=38b7bc12fb6720b389b16bfb158bf329&chksm=cfcacd9bf8bd448d7325d8d94d80b7a019d0389cab7c8cfeddd441ca8c28da0f0882a6311cfb&scene=21#wechat_redirect)文章中分析了普通socket的创建过程，关于socket的创建过程的函数大概如下所示，不同的协议族最后调用不同的协议族的socket创建函数，**如AF_INET协议族调用inet_create函数(如下图红色部分**)。  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**由于用户态也是采用socket API方式，所以用户态创建socket的过程与普通的协议族是一样的**(**Netlink的协议族是AF_NETLINK，对应的创建函数是netlink_create,如下代码所示**)，如下图所示：

`static const struct net_proto_family netlink_family_ops = {    .family = PF_NETLINK,    .create = netlink_create,  //netlink socket创建函数    .owner = THIS_MODULE, /* for consistency 8) */   };`

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

在AF_INET协议族中再根据不同的socket类型进行赋值,如下inet_create函数中实现所示

`static int inet_create(struct net *net, struct socket *sock, int protocol,            int kern)   {    struct sock *sk;    struct inet_protosw *answer;    struct inet_sock *inet;    struct proto *answer_prot;    ......       sock->ops = answer->ops;  //根据不同的协议赋值不同的操作集     ......   }`

其中struct inet_protosw answer有一个全局数组，根据不同的协议进行不同的封装，TCP协议下struct inet_protosw answer结构体中函数操作集就是封装的inet_stream_ops，如下所示：

`const struct proto_ops inet_stream_ops = {    .family     = PF_INET,    .owner     = THIS_MODULE,    .release    = inet_release,    .bind     = inet_bind,    .connect    = inet_stream_connect,    .socketpair    = sock_no_socketpair,    .accept     = inet_accept,    .getname    = inet_getname,    .poll     = tcp_poll,    .ioctl     = inet_ioctl,    .listen     = inet_listen,    .shutdown    = inet_shutdown,    .setsockopt    = sock_common_setsockopt,    .getsockopt    = sock_common_getsockopt,    .sendmsg    = inet_sendmsg,    .recvmsg    = inet_recvmsg,    .mmap     = sock_no_mmap,    .sendpage    = inet_sendpage,    .splice_read    = tcp_splice_read,    .read_sock    = tcp_read_sock,    .sendmsg_locked    = tcp_sendmsg_locked,    .sendpage_locked   = tcp_sendpage_locked,    .peek_len    = tcp_peek_len,   #ifdef CONFIG_COMPAT    .compat_setsockopt = compat_sock_common_setsockopt,    .compat_getsockopt = compat_sock_common_getsockopt,    .compat_ioctl    = inet_compat_ioctl,   #endif   };`

netlink机制(**AF_NETLINK协议族**)中，相关的函数集在__netlinkl_cteate函数中进行赋值，如下所示：

`static int __netlink_create(struct net *net, struct socket *sock,          struct mutex *cb_mutex, int protocol,          int kern)   {    struct sock *sk;    struct netlink_sock *nlk;       sock->ops = &netlink_ops;//设置AF_NETLINK协议族的函数集     ......   }`

netlink_ops如下所示：

`static const struct proto_ops netlink_ops = {    .family = PF_NETLINK,    .owner = THIS_MODULE,    .release = netlink_release,    .bind =  netlink_bind,    .connect = netlink_connect,    .socketpair = sock_no_socketpair,    .accept = sock_no_accept,    .getname = netlink_getname,    .poll =  datagram_poll,    .ioctl = netlink_ioctl,    .listen = sock_no_listen,    .shutdown = sock_no_shutdown,    .setsockopt = netlink_setsockopt,    .getsockopt = netlink_getsockopt,    .sendmsg = netlink_sendmsg,    .recvmsg = netlink_recvmsg,    .mmap =  sock_no_mmap,    .sendpage = sock_no_sendpage,   };`

在AF_INET协议族中，TCP、UDP核心的结构体:struct tcp_sock、struct udp_sock，它们代表TCP、UDP协议中socket底层的使用，在Netlink机制(AF_NETLINK协议族)中，**netlink_sock作为底层socket的使用，如下所示****：**

`struct netlink_sock {    /* 结构体sock必须处于最前端，这样可以通过sock指针找到netlink_sock*/    struct sock  sk;     u32   portid;  //表示本套接字自己绑定的id号，对于内核来说它就是0    u32   dst_portid; //表示目的id号    u32   dst_group;    u32   flags;    u32   subscriptions;    u32   ngroups; //表示协议支持多播组数量    unsigned long  *groups; //保存组位掩码    unsigned long  state;    size_t   max_recvmsg_len;    wait_queue_head_t wait;    bool   bound;    bool   cb_running;    int   dump_done_errno;    struct netlink_callback cb;    struct mutex  *cb_mutex;//互斥锁    struct mutex  cb_def_mutex;//默认的互斥锁    void   (*netlink_rcv)(struct sk_buff *skb);//保存接收用户态数据后的处理函数    int   (*netlink_bind)(struct net *net, int group);    void   (*netlink_unbind)(struct net *net, int group);    struct module  *module;       struct rhash_head node;    struct rcu_head  rcu;    struct work_struct work;   };`

**由于struct sock 结构体位于netlink_sock结构体成员的最前端，所以在分配完struct sock后可以直接找到netlink_sock首地址(struct sock结构体地址与netlink_sock结构体首地址一样****)**，之后对netlink_sock进行初始化，如下在netlinkl_cteate中的实现源码，其中**nlk_sk函数获取到netlink_sock的首地址,也就是struct sock的首地址。**

`static int netlink_create(struct net *net, struct socket *sock, int protocol,        int kern)   {     struct netlink_sock *nlk;    ......    nlk = nlk_sk(sock->sk);    nlk->module = module;    nlk->netlink_bind = bind;    nlk->netlink_unbind = unbind;    ......   }`

**2、关于Netlink socket通信地址**

以上将用户态创建Netlink socket的过程进行了浅析，在创建完socket后，用户态便可进行一系列的bind、send等操作。由于netlink机制是Linux内核与用户空间进行双向数据传输的机制，那么与普通的socket通信有一些不同，如下图**在用户态****通信地址上的不同：**

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

用户态通信地址结构：struct sockaddr_nl如下所示

`struct sockaddr_nl {    __kernel_sa_family_t nl_family; //nl_family始终为AF_NETLINK    unsigned short nl_pad;  //nl_pad始终为0    __u32  nl_pid;  //nl_pid为netlink套接字的单播地址，在发送消息时用于表示目的套接字的地址           __u32  nl_groups; //nl_groups表示组播组   };   `

nl_pid为netlink套接字的单播地址，在发送消息时用于表示目的套接字的地址，在用户空间绑定时可以指定为当前进程的PID号。nl_groups表示组播组。在发送消息时用于表示目的多播组，在绑定地址时用于表示加入的多播组。这里nl_groups为一个32位无符号数，其中的每一位表示一个 多播组，一个netlink套接字可以加入多个多播组用以接收多个多播组的多播消息（最多支持32个）。

**三、内核Netlink套接字创建分析**

**N****etlink预定义的协议类型:**如下所示，这些协议已经为不同的系统应用所使用，每种不同的应用都有特有的传输数据的格式，**因此如果用户不使用这些协议，需要加入自己定义的协议号。**

`#define NETLINK_ROUTE  0 /* Routing/device hook    */   #define NETLINK_UNUSED  1 /* Unused number    */   #define NETLINK_USERSOCK 2 /* Reserved for user mode socket protocols  */   #define NETLINK_FIREWALL 3 /* Unused number, formerly ip_queue  */   #define NETLINK_SOCK_DIAG 4 /* socket monitoring    */   #define NETLINK_NFLOG  5 /* netfilter/iptables ULOG */   #define NETLINK_XFRM  6 /* ipsec */   #define NETLINK_SELINUX  7 /* SELinux event notifications */   #define NETLINK_ISCSI  8 /* Open-iSCSI */   #define NETLINK_AUDIT  9 /* auditing */   #define NETLINK_FIB_LOOKUP 10    #define NETLINK_CONNECTOR 11   #define NETLINK_NETFILTER 12 /* netfilter subsystem */   #define NETLINK_IP6_FW  13   #define NETLINK_DNRTMSG  14 /* DECnet routing messages */   #define NETLINK_KOBJECT_UEVENT 15 /* Kernel messages to userspace */   #define NETLINK_GENERIC  16   /* leave room for NETLINK_DM (DM Events) */   #define NETLINK_SCSITRANSPORT 18 /* SCSI Transports */   #define NETLINK_ECRYPTFS 19   #define NETLINK_RDMA  20   #define NETLINK_CRYPTO  21 /* Crypto layer */   #define NETLINK_SMC  22 /* SMC monitoring */      #define NETLINK_INET_DIAG NETLINK_SOCK_DIAG      #define MAX_LINKS 32` 

**内核netlink配置结构：**

struct netlink_kernel_cfg，如下所示该结构体包含了内核netlink的可选参数，其中groups用于指定最大的多播组；flags成员可以为NL_CFG_F_NONROOT_RECV或NL_CFG_F_NONROOT_SEND，这两个符号前者用来限定非超级用户是否可以绑定到多播组，后者用来限定非超级用户是否可以发送组播；**input指针用于指定回调函数，该回调函数用于接收和处理来自用户空间的消息（若无需接收来自用户空间的消息可不指定）**，最后的三个函数指针实现sock的绑定和解绑定等操作，会添加到nl_table对应的项中去。

`/* optional Netlink kernel configuration parameters */   struct netlink_kernel_cfg {    unsigned int groups;    unsigned int flags;    void  (*input)(struct sk_buff *skb);    struct mutex *cb_mutex;    int  (*bind)(struct net *net, int group);    void  (*unbind)(struct net *net, int group);    bool  (*compare)(struct net *net, struct sock *sk);   };`  

**netlink属性头：**struct nlattr,如下所示，netlink的消息头后面跟着的是消息的有效载荷部分，它采用的是格式为“类型——长度——值”，简写TLV。其中类型和长度使用属性头nlattr来表示。

`struct nlattr {    __u16           nla_len;    __u16           nla_type;   };` 

**netlink套接字结构体:**netlink_sock结构体，其在用户态系统调用创建的socket和内核socket是一样的，如下所示(在二、中介绍)：

`struct netlink_sock {    /* 结构体sock必须处于最前端，这样可以通过sock指针找到netlink_sock*/    struct sock  sk;     u32   portid;  //表示本套接字自己绑定的id号，对于内核来说它就是0    u32   dst_portid; //表示目的id号    u32   dst_group;    u32   flags;    u32   subscriptions;    u32   ngroups; //表示协议支持多播组数量    unsigned long  *groups; //保存组位掩码    unsigned long  state;    size_t   max_recvmsg_len;    wait_queue_head_t wait;    bool   bound;    bool   cb_running;    int   dump_done_errno;    struct netlink_callback cb;    struct mutex  *cb_mutex;//互斥锁    struct mutex  cb_def_mutex;//默认的互斥锁    void   (*netlink_rcv)(struct sk_buff *skb);//保存接收用户态数据后的处理函数    int   (*netlink_bind)(struct net *net, int group);    void   (*netlink_unbind)(struct net *net, int group);    struct module  *module;       struct rhash_head node;    struct rcu_head  rcu;    struct work_struct work;   };   `

内核Netlink套接字创建主流程如下所示，**最终也会和用户态通过系统调用创建socket的调用函数一样：__netlink_create。**

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

`static inline struct sock *   netlink_kernel_create(struct net *net, int unit, struct netlink_kernel_cfg *cfg)   {    return __netlink_kernel_create(net, unit, THIS_MODULE, cfg);   }   `

netlink_kernel_create是对__netlink_kernel_create函数的封装：

```
struct sock *__netlink_kernel_create(struct net *net, int unit, struct module *module,   struct netlink_kernel_cfg *cfg){ struct socket *sock; struct sock *sk; struct netlink_sock *nlk;//用于netlink中socket的使用 struct listeners *listeners = NULL; struct mutex *cb_mutex = cfg ? cfg->cb_mutex : NULL; ...... if (sock_create_lite(PF_NETLINK, SOCK_DGRAM, unit, &sock))//分配socket结构体，并初始化，将socket的type类型指定为SOCK_DGRAM  return NULL; if (__netlink_create(net, sock, cb_mutex, unit, 1) < 0)//分配struct sock、struct netlink_sock等，并初始化  goto out_sock_release_nosk; sk = sock->sk;  ...... sk->sk_data_ready = netlink_data_ready; if (cfg && cfg->input)//设置接收数据包的回调函数函数  nlk_sk(sk)->netlink_rcv = cfg->input;  ......}
```

其中sock_create_lite函数分配socket结构体并初始化：

`int sock_create_lite(int family, int type, int protocol, struct socket **res)   {    int err;    struct socket *sock = NULL;       err = security_socket_create(family, type, protocol, 1);    if (err)     goto out;       sock = sock_alloc();  //分配socket结构体    if (!sock) {     err = -ENOMEM;     goto out;    }       sock->type = type; //设置socket类型    err = security_socket_post_create(sock, family, type, protocol, 1);    ......   }   `

__netlink_create函数如下，创建struct sock，根据首地址获取netlink_sock，并进行初始化

```
static int __netlink_create(struct net *net, struct socket *sock,       struct mutex *cb_mutex, int protocol,       int kern){ struct sock *sk; struct netlink_sock *nlk; sock->ops = &netlink_ops;//设置函数操作集 sk = sk_alloc(net, PF_NETLINK, GFP_KERNEL, &netlink_proto, kern); if (!sk)  return -ENOMEM; sock_init_data(sock, sk); nlk = nlk_sk(sk); //从struct sk获取netlink_sock首地址 if (cb_mutex) {  lockdep_set_class_and_name(nlk->cb_mutex,        nlk_cb_mutex_key  nlk->cb_mutex = cb_mutex; } else {  nlk->cb_mutex = &nlk->cb_def_mutex;  mutex_init(nlk->cb_mutex);s + protocol,        nlk_cb_mutex_key_strings[protocol]); } init_waitqueue_head(&nlk->wait); sk->sk_destruct = netlink_sock_destruct; sk->sk_protocol = protocol; return 0;}
```

netlink_kernel_create函数最后将设置数据包接收函数，如下所示：

 `if (cfg && cfg->input)     nlk_sk(sk)->netlink_rcv = cfg->input;`

**内核中netlink相关宏定义：**

`#define NLMSG_ALIGNTO   4U/* 宏 NLMSG_ALIGN(len) 用于得到不小于len且字节对齐的最小数值 */   #define NLMSG_ALIGN(len) ( ((len)+NLMSG_ALIGNTO-1) & ~(NLMSG_ALIGNTO-1) )   /* Netlink 头部长度 */   #define NLMSG_HDRLEN     ((int) NLMSG_ALIGN(sizeof(struct nlmsghdr)))   /* 计算消息数据 len 的真实消息长度（消息体 + 消息头）*/   #define NLMSG_LENGTH(len) ((len) + NLMSG_HDRLEN)   /* 宏 NLMSG_SPACE(len) 返回不小于 NLMSG_LENGTH(len) 且字节对齐的最小数值 */   #define NLMSG_SPACE(len) NLMSG_ALIGN(NLMSG_LENGTH(len))   /* 宏 NLMSG_DATA(nlh) 用于取得消息的数据部分的首地址，设置和读取消息数据部分时需要使用该宏 */   #define NLMSG_DATA(nlh)  ((void*)(((char*)nlh) + NLMSG_LENGTH(0)))   /* 宏 NLMSG_NEXT(nlh,len) 用于得到下一个消息的首地址, 同时 len 变为剩余消息的长度 */   #define NLMSG_NEXT(nlh,len)  ((len) -= NLMSG_ALIGN((nlh)->nlmsg_len), \                  (struct nlmsghdr*)(((char*)(nlh)) + NLMSG_ALIGN((nlh)->nlmsg_len)))   /* 判断消息是否 >len */   #define NLMSG_OK(nlh,len) ((len) >= (int)sizeof(struct nlmsghdr) && \               (nlh)->nlmsg_len >= sizeof(struct nlmsghdr) && \               (nlh)->nlmsg_len <= (len))   /* NLMSG_PAYLOAD(nlh,len) 用于返回 payload 的长度*/   #define NLMSG_PAYLOAD(nlh,len) ((nlh)->nlmsg_len - NLMSG_SPACE((len)))`

**四、使用Netlink机制**  

**内核态 netlink.c：**定义NETLINK_TEST协议，并实现input回调函数接口：net_rcv_msg，收到用户态数据后向用户态发送数据

`#include <linux/init.h>   #include <linux/module.h>   #include <linux/types.h>   #include <net/sock.h>   #include <linux/netlink.h>         #define NETLINK_TEST     30   #define MSG_LEN            100   #define USER_PORT        66         struct sock *nlsk;      extern struct net init_net;         int send_usrmsg(char *pbuf, uint16_t len)   {       struct sk_buff *nl_skb;       struct nlmsghdr *nlh;   //消息头部          int ret;       //创建sk_buff       nl_skb = nlmsg_new(len, GFP_ATOMIC);       if(!nl_skb)       {           printk("netlink alloc failure\n");           return -1;       }              /* 设置netlink消息头部 */       nlh = nlmsg_put(nl_skb, 0, 0, NETLINK_TEST, len, 0);       if(nlh == NULL)       {           printk("nlmsg_put failaure \n");           nlmsg_free(nl_skb);           return -1;       }           /* 拷贝数据发送 */       memcpy(nlmsg_data(nlh), pbuf, len);       ret = netlink_unicast(nlsk, nl_skb, USER_PORT, MSG_DONTWAIT);           return ret;      }         static void netlink_rcv_msg(struct sk_buff *skb)   {          struct nlmsghdr *nlh;       char *umsg = NULL;       char *kmsg = "hello users";       //从skb中获取data字段，并转换成nlh进行读取       nlh = nlmsg_hdr(skb);       //读取nlh后面的数据部分       umsg = NLMSG_DATA(nlh);       if(umsg){           printk("kernel recv from user: %s\n", umsg);           printk("port id :%d\n",NETLINK_CB(skb).portid);           send_usrmsg(kmsg, strlen(kmsg));    //给用户态发消息       }          }   struct netlink_kernel_cfg cfg = {       .input = netlink_rcv_msg,   };            static int __init test_netlink_init(void)   {       nlsk  = (struct sock *)netlink_kernel_create(&init_net,NETLINK_TEST,&cfg);       return 0;      }       static void __exit test_netlink_exit(void)   {       printk("exit......\n");   }   MODULE_LICENSE("GPL");   module_init(test_netlink_init);   module_exit(test_netlink_exit);   `

****Makefile:****

`obj-m:=netlink.o                                               CURRENT_PATH:=$(shell pwd)       LINUX_KERNEL:=$(shell uname -r)     LINUX_KERNEL_PATH:=/usr/src/linux-headers-$(LINUX_KERNEL)                                            all:    make -C $(LINUX_KERNEL_PATH) M=$(CURRENT_PATH) modules  #编译模块   clean:    make -C $(LINUX_KERNEL_PATH) M=$(CURRENT_PATH) clean   #清理`

**用户态 user.c：给内核发消息，并接收内核发来的数据**

`#include <stdio.h>   #include <stdlib.h>   #include <sys/socket.h>   #include <string.h>   #include <linux/netlink.h>   #include <stdint.h>   #include <unistd.h>      #define NETLINK_TEST 30   #define MSG_LEN 100   #define MAX_PLOAD 200   typedef struct _user_msg_info   {       struct nlmsghdr hdr;       char msg[MSG_LEN];   } user_msg_info;      int main(int argc,char **argv)   {       int sockfd;       /*       struct sockaddr_nl {        __kernel_sa_family_t    nl_family;  // AF_NETLINK （跟AF_INET对应）        unsigned short  nl_pad;     // zero         __u32       nl_pid;     // port ID  （通信端口号）        __u32       nl_groups;  //multicast groups mask    };       */       struct sockaddr_nl saddr, daddr;      /* struct nlmsghd 是netlink消息头   struct nlmsghdr {          __u32       nlmsg_len;  // Length of message including header        __u16       nlmsg_type; // Message content        __u16       nlmsg_flags;    // Additional flags        __u32       nlmsg_seq;  // Sequence number        __u32       nlmsg_pid;  // Sending process port ID    };   */        struct nlmsghdr *nlh;       user_msg_info u_info;       char *msg = "hell kernel, I am user process!";       socklen_t len;          //创建socket       sockfd = socket(AF_NETLINK,SOCK_RAW, NETLINK_TEST);          //初始化地址       memset(&saddr,0,sizeof(saddr));              //地址赋值       saddr.nl_family = AF_NETLINK;       saddr.nl_pad = 0;       saddr.nl_pid = 66;       saddr.nl_groups = 0;          //地址与sockt绑定-bind       bind(sockfd,(struct sockaddr *)&saddr,sizeof(saddr));              //初始化目的地址       memset(&daddr,0,sizeof(daddr));       memset(&daddr, 0, sizeof(daddr));       daddr.nl_family = AF_NETLINK;       daddr.nl_pid = 0; // to kernel        daddr.nl_groups = 0;                 //初始化消息头       nlh = (struct nlmsghdr *)malloc(NLMSG_SPACE(MAX_PLOAD));       memset(nlh, 0, sizeof(struct nlmsghdr));       nlh->nlmsg_len = NLMSG_SPACE(MAX_PLOAD);       nlh->nlmsg_flags = 0;       nlh->nlmsg_type = 0;       nlh->nlmsg_seq = 0;       nlh->nlmsg_pid = saddr.nl_pid; //self port          //设置消息内容       memcpy(NLMSG_DATA(nlh),msg,strlen(msg));          //发送消息       sendto(sockfd,nlh,nlh->nlmsg_len,0,(struct sockaddr *)&daddr,sizeof(struct sockaddr_nl));       printf("send kernel :%s",msg);              memset(&u_info, 0, sizeof(u_info));       len = sizeof(struct sockaddr_nl);       //接收消息       recvfrom(sockfd,&u_info,sizeof(user_msg_info),0,(struct sockaddr *)&daddr,&len);       printf("\n");       printf("from kernel:%s\n",u_info.msg);       close(sockfd);       return 0;   }`

**编译、插入内核：创建内核Netlink socket:**

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**运行用户态程序：向内核发送消息，并接收来自内核的消息：**  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

END

![](http://mmbiz.qpic.cn/mmbiz_png/EjWxxIM2EQI0YHzCpIYYwO0iaqh08EGCibYEjLqIqYm2CXPzmicQTkxqF453q1d9RcSicSLGjjCNyBsjDXdx8oDhcA/300?wx_fmt=png&wxfrom=19)

**技术简说**

主要分享Linux内核、内核网络知识，欢迎大家关注！

35篇原创内容

公众号

点个关注，一起学技术！  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

您的点赞和关注是我最大的动力

内核网络剖析22

内核网络剖析 · 目录

上一篇Linux内核 | socket底层的来龙去脉下一篇Linux TCP数据接收 | 快路径与慢路径

阅读 867

​