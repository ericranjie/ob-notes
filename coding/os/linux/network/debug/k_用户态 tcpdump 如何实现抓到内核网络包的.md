
CPP开发者 _2021年10月13日 11:57_

The following article is from 开发内功修炼 Author 张彦飞allen

今天聊聊大家工作中经常用到的 tcpdump。

在网络包的发送和接收过程中，绝大部分的工作都是在内核态完成的。那么问题来了，我们常用的运行在用户态的程序 tcpdump 是那如何实现抓到内核态的包的呢？有的同学知道 tcpdump 是基于 libpcap 的，那么 libpcap 的工作原理又是啥样的呢。如果让你裸写一个抓包程序，你有没有思路？

按照飞哥的风格，不搞到最底层的原理咱是不会罢休的。所以我对相关的源码进行了深入分析。通过本文，你将彻底搞清楚了以下这几个问题。

- tcpdump 是如何工作的？
- netfilter 过滤的包 tcpdump 是否可以抓的到？
- 让你自己写一个抓包程序的话该如何下手？

借助这几个问题，我们来展开今天的探索之旅！

## 一、网络包接收过程

在[图解Linux网络包接收过程](http://mp.weixin.qq.com/s?__biz=MzAxODI5ODMwOA==&mid=2666548640&idx=1&sn=7e6dcbbcd569ad4f3c20e915b78b3bac&chksm=80dc890bb7ab001d4cd880c773b3e7b3b9ee4d7d9d4fac0ebbeaa6d247c70084d8cde829bcf2&scene=21#wechat_redirect)一文中我们详细介绍了网络包是如何从网卡到达用户进程中的。这个过程我们可以简单用如下这个图来表示。

![[Pasted image 20241003161545.png]]

### 找到 tcpdump 抓包点

我们在网络设备层的代码里找到了 tcpdump 的抓包入口。在 \_\_netif_receive_skb_core 这个函数里会遍历 ptype_all 上的协议。还记得上文中我们提到 tcpdump 在 ptype_all 上注册了虚拟协议。这时就能执行的到了。来看函数：

```cpp
//file: net/core/dev.c
static int __netif_receive_skb_core(struct sk_buff *skb, bool pfmemalloc)   {       ......       //遍历 ptype_all （tcpdump 在这里挂了虚拟协议）       
list_for_each_entry_rcu(ptype, &ptype_all, list) {           if (!ptype->dev || ptype->dev == skb->dev) {               if (pt_prev)                   ret = deliver_skb(skb, pt_prev, orig_dev);               pt_prev = ptype;           }       }   }   
```

在上面函数中遍历 ptype_all，并使用 deliver_skb 来调用协议中的回调函数。

```cpp
//file: net/core/dev.c
static inline int deliver_skb(...)   {    return pt_prev->func(skb, skb->dev, pt_prev, orig_dev);   }   
```

对于 tcpdump 来说，就会进入 packet_rcv 了（后面我们再说为啥是进入这个函数）。这个函数在 net/packet/af_packet.c 文件中。

```cpp
//file: net/packet/af_packet.c   
static int packet_rcv(struct sk_buff *skb, ...)   {    __skb_queue_tail(&sk->sk_receive_queue, skb);    ......   }   
```

可见 packet_rcv 把收到的 skb 放到了当前 packet socket 的接收队列里了。这样后面调用 recvfrom 的时候就可以获取到所抓到的包！！

### 再找 netfilter 过滤点

为了解释我们开篇中提到的问题，这里我们再稍微到协议层中多看一些。在 ip_rcv 中我们找到了一个 netfilter 相关的执行逻辑。

```cpp
//file: net/ipv4/ip_input.c
int ip_rcv(...)   {    ......    return NF_HOOK(NFPROTO_IPV4, NF_INET_PRE_ROUTING, skb, dev, NULL,            ip_rcv_finish);   }   
```

如果你用 NF_HOOK 作为关键词来搜索，还能搜到不少 netfilter 的过滤点。不过所有的过滤点都是位于 IP 协议层的。

在接收包的过程中，数据包是先经过网络设备层然后才到协议层的。
![[Pasted image 20241003161556.png]]

那么我们开篇中的一个问题就有了答案了。假如我们设置了 netfilter 规则，在接收包的过程中，工作在网络设备层的 tcpdump 先开始工作。还没等 netfilter 过滤，tcpdump 就抓到包了！

**所以，在接收包的过程中，netfilter 过滤并不会影响 tcpdump 的抓包！**

## 二、网络包发送过程

我们接着再来看网络包发送过程。在[25 张图，一万字，拆解 Linux 网络包发送过程](http://mp.weixin.qq.com/s?__biz=MzAxODI5ODMwOA==&mid=2666554809&idx=1&sn=31381caf815f6b0dc266b6dc432959da&chksm=80dca112b7ab2804efc8a99772fc17a1217f791a988c5087a703b10c4bf3d27fee55abfbcd85&scene=21#wechat_redirect)一文中，我们详细描述过网络包的发送过程。发送过程可以汇总成简单的一张图。
![[Pasted image 20241003161602.png]]

### 找到 netfilter 过滤点

在发送的过程中，同样是在 IP 层进入各种 netfilter 规则的过滤。

```cpp
//file: net/ipv4/ip_output.c
int ip_local_out(struct sk_buff *skb)   {    //执行 netfilter 过滤    
err = __ip_local_out(skb);   }      int __ip_local_out(struct sk_buff *skb)   {    ......    return nf_hook(NFPROTO_IPV4, NF_INET_LOCAL_OUT, skb, NULL,            skb_dst(skb)->dev, dst_output);   }   
```

在这个文件中，还能看到若干处 netfilter 过滤逻辑。

### 找到 tcpdump 抓包点

发送过程在协议层处理完毕到达网络设备层的时候，也有 tcpdump 的抓包点。

`//file: net/core/dev.c   int dev_hard_start_xmit(struct sk_buff *skb, struct net_device *dev,      struct netdev_queue *txq)   {    ...    if (!list_empty(&ptype_all))     dev_queue_xmit_nit(skb, dev);   }      static void dev_queue_xmit_nit(struct sk_buff *skb, struct net_device *dev)   {    list_for_each_entry_rcu(ptype, &ptype_all, list) {     if ((ptype->dev == dev || !ptype->dev) &&         (!skb_loop_sk(ptype, skb))) {      if (pt_prev) {       deliver_skb(skb2, pt_prev, skb->dev);       pt_prev = ptype;       continue;      }     ......     }    }    }   `

在上述代码中我们看到，在 dev_queue_xmit_nit 中遍历 ptype_all 中的协议，并依次调用 deliver_skb。这就会执行到 tcpdump 挂在上面的虚拟协议。

在网络包的发送过程中，和接收过程恰好相反，是协议层先处理、网络设备层后处理。
![[Pasted image 20241003161609.png]]

**如果 netfilter 设置了过滤规则，那么在协议层就直接过滤掉了。在下层网络设备层工作的 tcpdump 将无法再捕获到该网络包**。

## 三、TCPDUMP 启动

前面两小节我们说到了内核收发包都通过遍历 ptype_all 来执行抓包的。那么我们现在来看看用户态的 tcpdump 是如何挂载协议到内 ptype_all 上的。

我们通过 strace 命令我们抓一下 tcpdump 命令的系统调用，显示结果中有一行 socket 系统调用。Tcpdump 秘密的源头就藏在这行对 socket 函数的调用里。

`# strace tcpdump -i eth0   socket(AF_PACKET, SOCK_RAW, 768)   ......   `

socket 系统调用的第一个参数表示创建的 socket 所属的地址簇或者协议簇，取值以 AF 或者 PF 开头。在 Linux 里，支持很多种协议族，在 include/linux/socket.h 中可以找到所有的定义。这里创建的是 packet 类型的 socket。

> 协议族和地址族：每一种协议族都有其对应的地址族。比如 IPV4 的协议族定义叫 PF_INET，其地址族的定义是 AF_INET。它们是一一对应的，而且值也完全一样，所以经常混用。

```cpp
//file: include/linux/socket.h
#define AF_UNSPEC 0   #define AF_UNIX  1 /* Unix domain sockets   */   #define AF_LOCAL 1 /* POSIX name for AF_UNIX */   #define AF_INET  2 /* Internet IP Protocol  */   #define AF_INET6 10 /* IP version 6   */   #define AF_PACKET 17 /* Packet family  */   ......   
```

另外上面第三个参数 768 代表的是 ETH_P_ALL，socket.htons(ETH_P_ALL) = 768。

我们来展开看这个 packet 类型的 socket 创建的过程中都干了啥，找到 socket 创建源码。
```cpp
//file: net/socket.c   
SYSCALL_DEFINE3(socket, int, family, int, type, int, protocol)    {    ......    retval = sock_create(family, type, protocol, &sock);    }      int __sock_create(struct net *net, int family, int type, ...)   {    ......    pf = rcu_dereference(net_families[family]);    err = pf->create(net, sock, protocol, kern);   }   
```
在 \_\_sock_create 中，从 net_families 中获取了指定协议。并调用了它的 create 方法来完成创建。

net_families 是一个数组，除了我们常用的 PF_INET（ ipv4 ） 外，还支持很多种协议族。比如 PF_UNIX、PF_INET6（ipv6）、PF_PACKET等等。每一种协议族在 net_families 数组的特定位置都可以找到其 family 类型。在这个 family 类型里，成员函数 create 指向该协议族的对应创建函数。
![[Pasted image 20241003161616.png]]

根据上图，我们看到对于 packet 类型的 socket，pf->create 实际调用到的是 packet_create 函数。我们进入到这个函数中来一探究竟，这是理解 tcpdump 工作原理的关键！
```cpp
//file: packet/af_packet.c   
static int packet_create(struct net *net, struct socket *sock, int protocol,       int kern)   {    ...    po = pkt_sk(sk);    po->prot_hook.func = packet_rcv;       //注册钩子    if (proto) {     po->prot_hook.type = proto;     register_prot_hook(sk);    }   }      static void register_prot_hook(struct sock *sk)   {    struct packet_sock *po = pkt_sk(sk);    dev_add_pack(&po->prot_hook);   }   
```
在 packet_create 中设置回调函数为 packet_rcv，再通过 register_prot_hook => dev_add_pack 完成注册。注册完后，是在全局协议 ptype_all 链表中添加了一个虚拟的协议进来。
![[Pasted image 20241003161623.png]]

我们再来看下 dev_add_pack 是如何注册协议到 ptype_all 中的。回顾我们开头看到的 socket 函数调用，第三个参数 proto 传入的是 ETH_P_ALL。那 dev_add_pack 其实最后是把 hook 函数添加到了 ptype_all 里了，代码如下。

```cpp
//file: net/core/dev.c   
void dev_add_pack(struct packet_type *pt)   {    struct list_head *head = ptype_head(pt);    list_add_rcu(&pt->list, head);   }      static inline struct list_head *ptype_head(const struct packet_type *pt)   {    if (pt->type == htons(ETH_P_ALL))     return &ptype_all;    else     return &ptype_base[ntohs(pt->type) & PTYPE_HASH_MASK];   }   
```

> 我们整篇文章都以 ETH_P_ALL 为例，但其实有的时候也会有其它情况。在别的情况下可能会注册协议到 ptype_base 里了，而不是 ptype_all。同样， ptype_base 中的协议也会在发送和接收的过程中被执行到。

**总结：tcpdump 启动的时候内部逻辑其实很简单，就是在 ptype_all 中注册了一个虚拟协议而已。**

## 四、总结

现在我们再回头看开篇提到的几个问题。

**1. tcpdump是如何工作的**

用户态 tcpdump 命令是通过 socket 系统调用，在内核源码中用到的 ptype_all 中挂载了函数钩子上去。无论是在网络包接收过程中，还是在发送过程中，都会在网络设备层遍历 ptype_all 中的协议，并执行其中的回调。tcpdump 命令就是基于这个底层原理来工作的。

**2. netfilter 过滤的包 tcpdump是否可以抓的到**\
关于这个问题，得分接收和发送过程分别来看。在网络包接收的过程中，由于 tcpdump 近水楼台先得月，所以完全可以捕获到命中 netfilter 过滤规则的包。
![[Pasted image 20241003161633.png]]

但是在发送的过程中，恰恰相反。网络包先经过协议层，这时候被 netfilter 过滤掉的话，底层工作的 tcpdump 还没等看见就啥也没了。\
![[Pasted image 20241003161638.png]]


**3. 让你自己写一个抓包程序的话该如何下手**\
如果你想自己写一段类似 tcpdump 的抓包程序的话，使用 packet socket 就可以了。我用 c 写了一段抓包，并且解析源 IP 和目的 IP 的简单 demo。

源码地址：https://github.com/yanfeizhang/coder-kung-fu/blob/main/tests/network/test04/main.c

编译一下，注意运行需要 root 权限。

```c
# gcc -o main main.c
# ./main 
```

运行结果预览如下。
![[Pasted image 20241003161644.png]]

- EOF -

---

推荐阅读  点击标题可跳转

1、[一文看懂 GDB 调试上层实现](http://mp.weixin.qq.com/s?__biz=MzAxNDI5NzEzNg==&mid=2651164878&idx=1&sn=59e1fc14abb03521e86135715cc36a16&chksm=80644191b713c8872e289ae1d44d59ce3ef0f1f7ecc7eea74a467aaee0dc3324374ea4caae21&scene=21#wechat_redirect)

2、[计算机网络的 89 个核心概念](http://mp.weixin.qq.com/s?__biz=MzAxNDI5NzEzNg==&mid=2651164876&idx=1&sn=3e88950f5148145132e4a7be90c08fd5&chksm=80644193b713c885b8480ad38ed98f7185cc8ed5c08c43108460e292a09d419e1b9665181718&scene=21#wechat_redirect)

3、[上午写了一段代码，下午就被开除了，奇怪的知识又增加了](http://mp.weixin.qq.com/s?__biz=MzAxNDI5NzEzNg==&mid=2651164883&idx=1&sn=f29d8c8a08732d7dec12e998c7bb6a71&chksm=8064418cb713c89a62ab13f4479ad0e1e8542b2d3570e87e7d3981c331c1055c7bbc655fb304&scene=21#wechat_redirect)

我们在 Github 维护着 9000+ star 的C语言/C++开发资源。日常分享 C语言 和 C++ 开发相关技术文章，每篇文章都经过精心筛选，一篇文章讲透一个知识点，让读者读有所获～

24篇原创内容

公众号

点赞和在看就是最大的支持❤️

Reads 1752

​

Comment

**留言 1**

- lusa

  2021年10月13日

  Like2

  收藏了，值得深入学习抓包

已无更多数据

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/pldYwMfYJpia3uWic6GbPCC1LgjBWzkBVqYrMfbfT6o9uMDnlLELGNgYDP496LvDfiaAiaOt0cZBlBWw4icAs6OHg8Q/300?wx_fmt=png&wxfrom=18)

CPP开发者

13Share4

1

Comment

**留言 1**

- lusa

  2021年10月13日

  Like2

  收藏了，值得深入学习抓包

已无更多数据
