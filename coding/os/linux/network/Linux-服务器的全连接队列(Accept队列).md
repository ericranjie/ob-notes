#

Linux内核之旅

 _2022年02月21日 13:05_

以下文章来源于技术简说 ，作者董旭

[

![](http://wx.qlogo.cn/mmhead/Q3auHgzwzM5b9FrINsvZl3ribRAQicibVs74cJ9zMmiagjpZREkxAeAFuQ/0)

**技术简说**.

主要分享Linux内核、内核网络知识，欢迎大家关注！

](https://mp.weixin.qq.com/s?__biz=MzI3NzA5MzUxNA==&mid=2664611059&idx=1&sn=e527a3436473659bd65ac46d4b9e036c&chksm=f04d9116c73a1800a9d700881d3fdfb41cdd34eedbdf67bd11dc5ab50f3a720352c5f68c8184&mpshare=1&scene=24&srcid=0221mYMeliCW8SMELQdUlYrL&sharer_sharetime=1645423684055&sharer_shareid=5fb9813bfe9ffc983435bfc8d8c5e9ca&key=daf9bdc5abc4e8d0802ee59967972fc6281a4d6678186aa43b2910c3030347f72f7dbb9dae9c3b2141a3895a44f0a14d15488beb05d77b7036f346f51617e93b327ffab4cbabb06a0d274f82ad1a0d032144b10e6da37949d02d3e204da5d184d44dfaaeffdae00aa8ce805d27904460c1f6056b13f19298095a5079216b553f&ascene=0&uin=MTEwNTU1MjgwMw%3D%3D&devicetype=Windows+11+x64&version=63090b19&lang=zh_CN&countrycode=CN&exportkey=n_ChQIAhIQz3YgkwNXOyljxTYCSaXlchLZAQIE97dBBAEAAAAAAMk9N9SSIMUAAAAOpnltbLcz9gKNyK89dVj0cgh6HwNFma4ENGkOZ00HtMVQAl3mxOrAnVvn1HELVKeF22DSPQZCaMqzeJpzXy9v%2FvM%2F2teoJFvEfU6GN40bxCPRcxhKyIg6K5cEl2xQfDLX4ggJzw1NrPB8BnNZOsZkDsUGty8Kk5ujmrh4hQ7lDQyeqnmJZasX3Bh%2BugsaFUxUrIBzvKIN0QdRZuXDW9tEec%2Ft5USHzsLXXEdspVltMDhQqdGjD5DR2%2BbUnFnOFyhd5a8%3D&acctmode=0&pass_ticket=Z0aywmByOphl43bFCJE5VEMDBWVY%2BUmljStU7Ish11X%2Bjq4sgFdJXy%2BbgkJCAVIZ&wx_header=1&fasttmpl_type=0&fasttmpl_fullversion=7351805-zh_CN-zip&fasttmpl_flag=1#)

[上篇文章(Linux内核角度分析服务器Listen细节)](http://mp.weixin.qq.com/s?__biz=Mzg5MTU1ODgyMA==&mid=2247484692&idx=1&sn=f4b8887d505c64e160d8085597786659&chksm=cfcacbdff8bd42c95e0db7a60c46a8c688596d7d71dd6046a7325c2a7565320c75d89b543b8c&scene=21#wechat_redirect)分析了服务器Listen的底层细节，其中也分析了Listen系统调用的backlog参数，其决定了服务器Listen过程中全连接队列(Accept队列)的最大长度。本文将更进一步分析全连接队列(Accept队列)以及backlog参数是如何影响中全连接队列(Accept队列)的，并通过小实验直观了解backlog参数对全连接队列(Accept队列)的影响。

**全连接队列是什么？**

全连接队列存储3次握手成功并已建立的连接，将其称为全连接队列，也可称为接收队列(Accept队列)，本文中的描述将称为Accept队列或全连接队列。如下红框中所示，**全连已成功建立三次握手，当前的TCP状态为ESTABLISHED，但是服务端还未Accept的队列。**

  

![图片](https://mmbiz.qpic.cn/mmbiz_png/EjWxxIM2EQKGC4tNpCicE3Dtyic0D2TFK3gPIo9ejze1VZhX3bFiaYFgxFxd380Eo62d0DNP2R9U3L0ryiaa6CGnPw/640?wx_fmt=png&wxfrom=13&tp=wxpic)

**那么这全连接队列**(Accept队列)**在Linux内核中用什么数据结构进行表示？**

**连接请求块-****存储队列**

**在介绍Accept队列前先看一下连接请求块：存储相关连接请求的队列的结构体**

连接请求块的存储队列是对SYN同步队列(半连接)队列(服务端收到客户端SYN请求并回复SYN+ACK的队列)、接收(全连接)队列的描述。在Linux内核中使用`request_sock_queue` 进行表示，如下结构体所示:

```
struct request_sock_queue { spinlock_t  rskq_lock; u8   rskq_defer_accept; u32   synflood_warned; atomic_t  qlen;//半连接队列长度计数 atomic_t  young; struct request_sock *rskq_accept_head;//接收队列队列头部 struct request_sock *rskq_accept_tail;// struct fastopen_queue fastopenq;  /* Check max_qlen != 0 to determine          * if TFO is enabled.          */};
```

该结构体描述了两种队列的相关信息，第一个是半连接队列的长度，使用`atomic_t qlen`来表示，第二个是Accept队列链表，使用struct request_sock *rskq_accept_head;来表示 Accept队列链表的头部，`struct request_sock *rskq_accept_tail;`表示Accept队列链表的尾部。  

**Accept接收(全连接)队列**

**由连接请求块-存储队列的结构体可以看到全连接队列-Accept队列由struct request_sock结构体进行表示**，如下所示，服务器端收到SYN请求之后，内核会建立连接请求块（req）

```
struct request_sock { struct sock_common  __req_common;#define rsk_refcnt   __req_common.skc_refcnt#define rsk_hash   __req_common.skc_hash#define rsk_listener   __req_common.skc_listener#define rsk_window_clamp  __req_common.skc_window_clamp#define rsk_rcv_wnd   __req_common.skc_rcv_wnd struct request_sock  *dl_next; u16    mss; u8    num_retrans; /* number of retransmits */ u8    cookie_ts:1; /* syncookie: encode tcpopts in timestamp */ u8    num_timeout:7; /* number of timeouts */ u32    ts_recent; struct timer_list  rsk_timer; const struct request_sock_ops *rsk_ops; struct sock   *sk; u32    *saved_syn; u32    secid; u32    peer_secid;};
```

结构体成员变量 `request_sock *dl_next` 指向队列中下一个Accept队列节点，Accept队列与存储队列直接的关系如下图所示：  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

在[上一篇文章](http://mp.weixin.qq.com/s?__biz=Mzg5MTU1ODgyMA==&mid=2247484692&idx=1&sn=f4b8887d505c64e160d8085597786659&chksm=cfcacbdff8bd42c95e0db7a60c46a8c688596d7d71dd6046a7325c2a7565320c75d89b543b8c&scene=21#wechat_redirect)中分析服务器listen函数调用时，发现到`listen()`将调用`inet_csk_listen_start()`，后者将调用`reqsk_queue_alloc()`创建`struct request_sock queue icsk_accept_queue`，即创建存储队列的结构体。然后进行一些队列长度相关参数的设定。

在分析长度相关参数的设置代码之前，回顾一下用户传入的backlog参数在内核中最终如何取值的,如下代码所示，内核变量backlog的最终取值为 **backlog = Min(用户传入的backlog值，somaxconn)**，其中somaxconn的值是Linux系统的默认值：128，该值可以通过 /proc/sys/net/core/somaxconn进行设置  。经常会有一个**问题：****Listen时backlog参数越大，Accept队列会越大吗？**

从上面的分析也可以看出来答案，内核中backlog变量的最终取值是Listen系统调用传入的backlog与系统默认值两者之间的最小值， 所以在Listen时backlog的需求超过系统默认值128时，需要修改系统默认值以满足更大的需求。

下面分析长度相关参数的设置，如下代码节选自inet_csk_listen_start函数，`sk_max_ack_backlog`是对Accept队列最大长度进行限制,`sk_ack_backlog`是对当前Accept队列的长度进行计数，**最开始初始化为0，也就是计数从0开始。**

```
 reqsk_queue_alloc(&icsk->icsk_accept_queue); sk->sk_max_ack_backlog = backlog; sk->sk_ack_backlog = 0; inet_csk_delack_init(sk);
```

**下面举例分析一下**Accept队列并分析sk_ack_backlog如何对Accept队列进行计数、sk_max_ack_backlog = backlog如何对Accept接收队列长度的限制：  

**1、服务器收到客户端三次握手最后一个ACK时：**  

收到客户端最后一个ACK后·，服务器调用tcp_v4_rcv->tcp_v4_syn_rcv_sock,然后通过tcp_check_req函数进行检查，如果一切检查正常的话，使用回调syn_recv_sock处理去创建子套接口（child），之后由函数inet_csk_complete_hashdance中设置req->sk = child，然后将req放入全连接队列icsk_accept_queue里面。

如下是tcp_check_req函数：

```
struct sock *tcp_check_req(struct sock *sk, struct sk_buff *skb,      struct request_sock *req,      bool fastopen){ ...... child = inet_csk(sk)->icsk_af_ops->syn_recv_sock(sk, skb, req, NULL,        req, &own_req); ...... return inet_csk_complete_hashdance(sk, child, req, own_req); ......}
```

syn_recv_sock对应的回调函数首先是**对Accept队列进行判断：当前的Accept队列是否满，未满的情况下才会去创建子套接口**

```
struct sock *tcp_v4_syn_recv_sock(const struct sock *sk, struct sk_buff *skb,      struct request_sock *req,      struct dst_entry *dst,      struct request_sock *req_unhash,      bool *own_req){ ...... if (sk_acceptq_is_full(sk))  goto exit_overflow;......}
```

判断队列是否满：sk_acceptq_is_full(sk)，从此也看到了sk_max_ack_backlog对Accept接收队列的限制。  

```
static inline bool sk_acceptq_is_full(const struct sock *sk){ return sk->sk_ack_backlog > sk->sk_max_ack_backlog;}
```

inet_csk_complete_hashdance将创建好的子套接口添加到Accept队列如下：  

```
struct sock *inet_csk_complete_hashdance(struct sock *sk, struct sock *child,      struct request_sock *req, bool own_req){ if (own_req) {  inet_csk_reqsk_queue_drop(sk, req);  reqsk_queue_removed(&inet_csk(sk)->icsk_accept_queue, req);  if (inet_csk_reqsk_queue_add(sk, req, child))   return child; } /* Too bad, another child took ownership of the request, undo. */ bh_unlock_sock(child); sock_put(child); return NULL;}
```

inet_csk_complete_hashdance函数的参数:**own_req仅当tcp_check_req函数中成功创建child子套接口才会为真。inet_csk_reqsk_queue_add函数会将设置req->sk = child，然后将req放入Accept队列icsk_accept_queue里面如下所示：**

```
struct sock *inet_csk_reqsk_queue_add(struct sock *sk,          struct request_sock *req,          struct sock *child){ struct request_sock_queue *queue = &inet_csk(sk)->icsk_accept_queue; spin_lock(&queue->rskq_lock); if (unlikely(sk->sk_state != TCP_LISTEN)) {  inet_child_forget(sk, req, child);  child = NULL; } else {  req->sk = child;//req与子套接口关联  req->dl_next = NULL;    /*如果全连接队列头部为空，也就是队列为空时  if (queue->rskq_accept_head == NULL)   queue->rskq_accept_head = req;//设置当前的req队列为头部  else    /*如果全连接队列不为空时*/   queue->rskq_accept_tail->dl_next = req;//尾插到队列最后  queue->rskq_accept_tail = req;//设置当前的req为队列尾部  sk_acceptq_added(sk); } spin_unlock(&queue->rskq_lock); return child;}
```

**2、****服务端接收到客户端最后一个ACK并加入Accept队列后**  

      **服务器Accept获取Accept队列的请求套接口，并****删除该请求套接口时**

在1、中也提到：收到客户端最后一个ACK后·，服务器调用tcp_v4_rcv->tcp_v4_syn_rcv_sock,然后通过tcp_check_req函数进行检查，如果一切检查正常的话，使用回调syn_recv_sock处理去创建子套接口，之后由函数inet_csk_complete_hashdance将子套接口添加到ACCEPT队列中。那么当添加Accept接收队列后，就要进行队列的计数，inet_csk_reqsk_queue_add函数调用sk_accepttq_added(sk)：

```
static inline void sk_acceptq_added(struct sock *sk){ sk->sk_ack_backlog++;}
```

**子套接口接入到Accept队列后调用该函数进行：**`sk->sk_ack_backlog++`**,****从而对队列进行计数管理。**  

并且当服务执行accept()后，accept()将返回已建立的连接,此时需要删除该请求套接口，删除过程如下：

```
static inline struct request_sock *reqsk_queue_remove(struct request_sock_queue *queue,            struct sock *parent){ struct request_sock *req; spin_lock_bh(&queue->rskq_lock); req = queue->rskq_accept_head; if (req) {  sk_acceptq_removed(parent);  queue->rskq_accept_head = req->dl_next;  if (queue->rskq_accept_head == NULL)   queue->rskq_accept_tail = NULL; } spin_unlock_bh(&queue->rskq_lock); return req;}
```

函数reqsk_queue_remove为简单的链表移除单个元素的操作，rskq_accept_head为链表的头，注意**ACCEPT队列总是从头部开始移除队列中的子套接口元素，即用户层的accept操作总是取走队列中的第一个子套接口,**如下图所示,绿色的线即头部重新指向被移除的next元素。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

reqsk_queue_remove函数完成以上移除队列元素的操作之前执行： `sk_acceptq_remove(parent)`的操作，如下所示：  

```
static inline void sk_acceptq_removed(struct sock *sk){ sk->sk_ack_backlog--;}
```

`sk->sk_ack_backlog--`**操作对队列元素的个数进行更新**  

综上分析，sk_ack_backlog是对Accept接收队列的计数，sk_max_ack_backlog限制了Accept接收队列的最大长度，sk_max_ack_backlog也正是Listen系统调用传入的参数backlog。

**小实验**

**1、首先创建一个服务端：**

服务端开启Listen，并设置Listen函数的backlog参数为5，即全连接队列**最大长度只能到6**(由上面的内核分析也可知，sk_ack_backlog对Accept队列的计数是从0开始的，长度限制变量sk_max_ack_backlog就是Min(用户传入的backlog,系统默认值128)，也就是说sk_max_ack_backlog为5，也就是说最大长度为6)，**重要的一点是服务端不进行Accept处理：**

```
#include <stdio.h>#include <stdlib.h>#include <string.h>#include <errno.h>#include <sys/socket.h>#include <netinet/in.h>#include <unistd.h>int main(int argc, char *argv[]){ int sock_fd, conn_fd; struct sockaddr_in server_addr; int port_number = 5200; // 创建socket描述符 if ((sock_fd = socket(AF_INET, SOCK_STREAM, 0)) == -1) {  fprintf(stderr,"Socket error:%s\n\a", strerror(errno));  exit(1); } // 填充sockaddr_in结构 bzero(&server_addr, sizeof(struct sockaddr_in)); server_addr.sin_family = AF_INET; server_addr.sin_addr.s_addr = htonl(INADDR_ANY); server_addr.sin_port = htons(port_number); // 绑定sock_fd描述符 if (bind(sock_fd, (struct sockaddr *)(&server_addr), sizeof(struct sockaddr)) == -1) {  fprintf(stderr,"Bind error:%s\n\a", strerror(errno));  exit(1); } // 监听sock_fd描述符 if(listen(sock_fd, 5) == -1) {  fprintf(stderr,"Listen error:%s\n\a", strerror(errno));  exit(1); }        //阻塞到这，不去Accept与接收数据，这将导致全连接队列    while (1){            }     if ((conn_fd = accept(sock_fd, (struct sockaddr *)NULL, NULL)) == -1) {  printf("accept socket error: %s\n\a", strerror(errno)); } close(conn_fd);close(sock_fd); exit(0);}
```

**2、运行服务端程序：**  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**3、编写客户端程序:**要求向服务端发起多次连接(大于6次)，使用Go语言编写的客户端程序如下，并发10个去连接服务端

```
package mainimport ("fmt""net""time")func connect() { _, err := net.Dial("tcp4", "127.0.0.1:5200") if err != nil {  fmt.Println(err) } fmt.Println("三次握手成功！\n")}func main(){ for i := 0; i < 10; i++ {  go connect() } time.Sleep(time.Minute * 10)}
```

**4、运行客户端，可以看到连接了6次**

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

通过ss命令：ss -nlt  

-l： 显示正在监听(Listening)的socket

-n ：不解析服务器名称

-t ：只显示 tcp socket

Recv-Q：当前全连接队列的大小，也就是当前已完成三次握手并等待服务端 accept() 的 TCP 连接；

Send-Q：当前全连接最大队列长度(从0开始计数)，上面服务器的最大全连接长度为6(0~5)；

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

可以看到服务端 127.0.0.1:5200的Send-Q为5(0~5),即最大全连接长度为6，Recv-Q是当前的Accept队列的长度为6。10个并行连接只有6个成功完成3次握手，剩下4个都未完成三次握手。说明TCP 全连接队列过小，就容易溢出，当发生 TCP 全连接队溢出的时候，后续的请求就会被丢弃。

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

END

  

![](http://mmbiz.qpic.cn/mmbiz_png/EjWxxIM2EQI0YHzCpIYYwO0iaqh08EGCibYEjLqIqYm2CXPzmicQTkxqF453q1d9RcSicSLGjjCNyBsjDXdx8oDhcA/300?wx_fmt=png&wxfrom=19)

**技术简说**

主要分享Linux内核、内核网络知识，欢迎大家关注！

35篇原创内容

公众号

阅读 1989

​

写留言

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/SeWfibBcBT0EibtIWVNvshnuWMN1AoJw3poFIsbpaIVyZibCCqwBUR21rcDfrQgoqYzaYNdS14IIXzvmzvibdDa5Rw/300?wx_fmt=png&wxfrom=18)

Linux内核之旅

1119

写留言

写留言

**留言**

暂无留言