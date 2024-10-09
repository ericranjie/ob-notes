Linux内核之旅

_2024年05月01日 13:12_ _陕西_

以下文章来源于云巅论剑 ，作者卢烈

**云巅论剑**.

发掘Linux相关内核技术，探索操作系统未来发展方向。

\](https://mp.weixin.qq.com/s?\_\_biz=MzI3NzA5MzUxNA==&mid=2664617582&idx=1&sn=2842836429faab0c0fdea9bac2959363&chksm=f04dfb8bc73a729d40155c5863cb116ab9f6dac72dd61e4e522cc81f94f2d05674c7a4e0d0c5&mpshare=1&scene=24&srcid=0502qZCYBeKNtBWuONZVIdnG&sharer_shareinfo=fe468619fb57e1c1b337d1d2108d3bd3&sharer_shareinfo_first=fe468619fb57e1c1b337d1d2108d3bd3&key=daf9bdc5abc4e8d0787aefccf840cbd8ef2cf6e893e92008f36d3936c048ca1d40e61cefed19017ab238e3c37ee4e2601f99bec0879d59f5c0dde178b301506ada33aae36cc7f91d652f17bb45a9bdbf3b00dcc7c2e0b3ac3da4ae1cff922f7bc23d1427f0927d18fd9a544b72dd5564e39a5ad2212ab69fe674119f736afcd0&ascene=0&uin=MTEwNTU1MjgwMw%3D%3D&devicetype=Windows+11+x64&version=63090621&lang=zh_CN&countrycode=CN&exportkey=n_ChQIAhIQ8gAjaN5UEHabuDXAL5ReSBLmAQIE97dBBAEAAAAAAAkeLyq%2BBZ8AAAAOpnltbLcz9gKNyK89dVj0XBuGXFr%2FKp1U4p6tNEMlut%2BYoa2Hx6L2ECtJBGPIGW4MTrt8WQwUL87V2e4frtX3Ic9R6c9H2ntbwcsSOx%2BPOiXh%2FH%2BswcsGlyfym6OVB5RrXVOtZzuchODFS3jdaqGClqvrtePpt8wDFVmjd62WXbHCCewja1Dg000O%2FHWrHTSzXJC9pmLRDmpqA21c1ydwnKnj4pcdeTSYb2TexjPWaWQ6yTS6utHGU8EGcG8QeSskfcuEAbEykWlSJmyNJje3&acctmode=0&pass_ticket=yibkohbe0bO3Mc2IZ2laS0BHAG4rUIlnlZs9jToI3W5If2ONMJBlj2Th9YWv7ZBv&wx_header=1&fasttmpl_type=0&fasttmpl_fullversion=7350504-zh_CN-zip&fasttmpl_flag=1#)

> tcp 四次挥手是超经典的网络知识，但是网络中的异常状况千奇百怪，说不定会“偷袭”到标准流程的盲区。最近笔者遇到了一个罕见的挥手乱序问题，经过对内核代码的分析和试验，最后终于找到了原因，角度可谓刁钻。
>
> 本文从技术视角，将排查过程记录下来，既是对整个过程的小小总结，将事情彻底完结掉，也是对 tcp 实现的一些细节的学习记录。

⚠️本文内容包括但不限于：tcp 四次挥手（同时关闭），tcp 包的 seq/ack 号规则，tcp 状态机，内核 tcp 代码，tcp 发送窗口等知识。

_**01**_

**问题是什么？**

> 内核版本 5.10.112。

一句话：四次挥手中，由于 fin 包和 ack 包乱序，导致等了一次 timeout 才关闭连接。

过程细节：

- 同时关闭的场景，server 和 client **几乎同时**向对方发送 fin 包。

- client 先收到了 server 的 fin 包，并回传 ack 包。

- 然而 server 处发生乱序，**先收到了 client 的 ack 包，后收到了 fin 包**。

- 结果表现为 server **未能正确处理 client 的 fin 包**，未能返回正确的 ack 包。

- client 没收到（针对 fin 的）ack 包，因此**等待超时后重传 fin 包**，之后才回归正常关闭连接的流程。

### **问题抓包具体分析**

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/bDZYPQUwvDn3sDSbOTOJZjiaBVs46tH0nibPaYOIrVajDb5ia6tqicgJicML57J63XH4PEIpnMZmviaeKSKxReU7ZwRQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

图中上半部分是 client，下半部分是 server。

重点关注 ID 为**14913、14914、20622、20623** 这四个包，后面为了方便分析，对 seq 和 ack 号取后四位：

- 20622（seq=4416，ack=753），client 发送的 fin 包：client 主动关闭连接，向 server 发送 fin 包。

- 14913（seq=753，ack=4416），server 发送的 fin 包：server 主动关闭连接，向 client 发送 fin 包。

- 20623（seq=4417，ack=754），client 响应的 ack 包：client 收到 server 的 fin，响应一个 ack 包。

- 14914（seq=754，ack=4416），server 发送的 ack 包。

问题发生在 server 处（红框位置），发送 14913 后：

- 先收到 **20623**（seq=4417），但此时期望收到的 seq 为 4416，所以被标记为\[previous segment not captured\]。

- 然后收到 **20622**，回传了一个 ack 包，ID 为 14914，问题就出现在这里：这个数据包的 ack=4416，这意味着 server 还在等待 seq=4416 的数据包，换言之，fin-20622 没有被 server 真正接收到。

- client 发现 20622 没有被正确接收，因此在等到 timeout 后，重新发送了 fin 包（id=20624），此后连接正常关闭。

（这里再次强调一下ack-20623 和 fin-20622，后面会经常提到这两个包）

首先，这个现象在直觉上是很不合理的，tcp 应当有恰当的机制保证乱序恢复。这里 20622 和 20623 都已经到达了 server，虽然发生了乱序，也不应当影响 server 把两者都接收，这是主要的疑问点所在。

经过初步分析，我们推测最可能的原因是 20622 被 server 的内核忽略了（原因目前未知）。既然是内核的行为，就先尝试在本地环境复现这个问题。然而喜闻乐见地没有成功。

### **新问题：尝试复现未成功**

为了模拟上述乱序的场景，我们使用两台 ecs，在 client 上伪造 tcp 包，与 server 处的正常 socket 通信。

server 处的抓包结果如下：

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/bDZYPQUwvDn3sDSbOTOJZjiaBVs46tH0nd0GlykPCq3xqMIgxmcTd5SG5LooDWoUHkAUtP2LoKZjKThUMZWGqvg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

注意看 No.为 5、6、7、8 的包：

- 5：server 向 client 发送 fin（这里不知为何有一次重传，但是不影响后面的效果，没有深究）。

- 6：client 先传回了 seq=1002 的 ack 包。

- 7：client 后传回了 seq=1001 的 fin 包。

- 8：server 传回了ack=1002 的 ack 包，ack=1002 意味着 client 的 fin 包被正常接收了！（如果在问题场景下，此时回传的 ack 包，ack 应当为 1001）

之后，为了保持内核版本一致，把相同的程序转移到本地虚拟机上运行，得到同样的结果。换言之，复现失败了。

### **附：模拟程序代码**

工具：python + scapy

> 这里用 scapy 伪造 client，发送乱序的 ack 和 fin，是为了观察 server 回传的 ack包。
>
> 因为 client 并未真的走了 tcp 协议，所以无论复现成功与否，都不能观察到超时重传。

（1）server 处正常 socket 监听：

`import socket      server_ip = "0.0.0.0"   server_port = 12346      server_socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)   server_socket.bind((server_ip, server_port))   server_socket.listen(5)      connection, client_address = server_socket.accept()      connection.close() #发送fin   server_socket.close()   `

（2）client 模拟乱序：

`from scapy.all import *   import time   import sys      target_ip = "略"   target_port = 12346   src_port = 1234      #伪造数据包，建立tcp连接   ip = IP(dst=target_ip)   syn = TCP(sport=src_port, dport=target_port, flags="S", seq=1000)   syn_ack = sr1(ip / syn)   if syn_ack and TCP in syn_ack and syn_ack[TCP].flags == "SA":       print("Received SYN-ACK")       ack = TCP(sport=src_port, dport=target_port,                  flags="A", seq=syn_ack.ack, ack=syn_ack.seq+1)       send(ip / ack)       print("Sent ACK")   else:       print("Failed to establish TCP connection")        def handle_packet(packet):       if TCP in packet and packet[TCP].flags & 0x01:           print("Received FIN packet") #若收到server的fin，先传ack，再传fin           ack = TCP(sport=src_port, dport=target_port,                      flags="A", seq=packet.ack+1, ack=packet.seq+1)           send(ip / ack)                    time.sleep(0.1)           fin = TCP(sport=src_port, dport=target_port,                      flags="FA", seq=packet.ack, ack=packet.seq)           send(ip / fin)           sys.exit(0)   sniff(prn=handle_packet)   `

## **问题出现的位置？**

server 处出现乱序，结果连接未能正常关闭，而是等到 client 超时重传 fin 包，才关闭连接。

## **问题带来什么影响？**

server 处连接关闭的时间变长了（额外增加 200ms），对时延敏感的场景影响明显。

## **本文要解决什么问题？**

- 该现象是否是内核的合法行为？（先剧透一下，是合法行为）

- 为什么本地复现失败了？

_**02**_

**问题排查**

经过了大约 6 个周末的间断式\[看代码-试验\]循环，终于找到了问题所在！

下面将简要描述问题排查的过程，也包括我们的一些失败尝试。

## **初步分析**

回到上面的问题，现在不仅不清楚问题原因，本地复现还完美符合理想情况。

简单来说：

- 本地复现-乱序不影响挥手；

- 问题场景-乱序导致超时重传。

可以确定，问题很大概率出现在 server 对 ack-20623 和 fin-20622 的处理上。

（下面会以 ack-20623 和 fin-20622 代指乱序的 ack 和 fin 包）

关键在于：server 发送 fin 后（进入 FIN_WAIT_1 状态），对后面收到的乱序 ack-20623 和 fin-20622 是如何处理的。这里涉及到 tcp 的状态转移，所以，首要问题是确定其中的状态转移过程。之后才能根据状态转移锁定对应的代码片段，做具体分析。

## **确定状态转移**

由于问题发生在挥手过程中，很自然想到通过观察状态转移来判断数据包的接收/处理情况。

我们结合复现过程，利用 ss 和 eBPF ，监控 tcp 的状态变化。确定了 server 在收到ack-20623后，由FIN_WAIT_1进入了FIN_WAIT_2状态，这意味着 ack-20623 被正确处理了。那么问题大概率出现在 fin-20622 的处理上，这也证实了我们最初的猜测。

> 这里还有一个奇怪的点：按照正确的挥手流程，server 在FIN_WAIT_2收到fin后应当进入TIMEWAIT 状态。我们在 ss 中观察到了这个状态转移，但是使用ebpf监控时，并没有捕捉到这个状态转移。
>
> 当时我们并未关注这个问题，后来才知晓原因：eBPF 实现中，只记录tcp_set_state()引发的状态转移。而此处虽然进入了 TIMEWAIT 状态，却并未经过tcp_set_state()，因此 eBPF 中无法看到。
>
> 关于这里如何进入TIMEWAIT，请看末尾的“番外”一节。

**附：eBPF 监控结果**

（FIN_WAIT1 转移到 FIN_WAIT2 时，snd_una 有更新，确定 ack-20623 被正确处理了）

`<idle>-0    [000] d.s. 42261.233642: PASSIVE_ESTABLISHED: start monitor tcp state change   <idle>-0    [000] d.s. 42261.233651: port:12346,snd_nxt:154527568,snd_una:154527568   <idle>-0    [000] d.s. 42261.233652: rcv_nxt:1001,recved:0,acked:0      <...>-9451 [007] d... 42261.233808: changing from ESTABLISHED to FIN_WAIT1   <...>-9451 [007] d... 42261.233815: port:12346,snd_nxt:154527568,snd_una:154527568   <...>-9451 [007] d... 42261.233816: rcv_nxt:1001,recved:0,acked:0      <idle>-0    [000] dNs. 42261.464578: changing from FIN_WAIT1 to FIN_WAIT2   <idle>-0    [000] dNs. 42261.464588: port:12346,snd_nxt:154527569,snd_una:154527569   <idle>-0    [000] dNs. 42261.464589: rcv_nxt:1001,recved:0,acked:1   `

## **内核源码分析**

事已至此，不得不看一看内核源码了。结合上面的分析，问题大概率发生在  tcp_rcv_state_process() 函数中，抽取出其中关于 TCP_FIN_WAIT2 的片段，然而很遗憾，在这个片段中没有发现疑点：

（tcp_rcv_state_process是接收数据包时处理状态转移的函数，位于net/ipv4/tcp_input.c）

`case TCP_FIN_WAIT1:   case TCP_FIN_WAIT2:   /* RFC 793 says to queue data in these states,   * RFC 1122 says we MUST send a reset.   * BSD 4.4 also does reset.   */   if (sk->sk_shutdown & RCV_SHUTDOWN) {   if (TCP_SKB_CB(skb)->end_seq != TCP_SKB_CB(skb)->seq &&   after(TCP_SKB_CB(skb)->end_seq - th->fin, tp->rcv_nxt)) { //经过分析，不符合该条件   NET_INC_STATS(sock_net(sk), LINUX_MIB_TCPABORTONDATA);   tcp_reset(sk);   return 1;   }   }   fallthrough;   case TCP_ESTABLISHED:   tcp_data_queue(sk, skb); //如果进入了这个函数，乱序会被纠正，fin的处理也在该函数中   queued = 1;   break;   `

如果运行到了这里，基本可以确定 fin 会被正常处理，所以我们将这个位置作为我们检查的终点。也就是说，乱序的 fin-20622 应当是没有成功到达此处的。我们从这个位置开始，向前查找，找到了一个非常可疑的位置，同样是在 tcp_rcv_state_process 中。

`//检查ack值是否合法   acceptable = tcp_ack(sk, skb, FLAG_SLOWPATH |   FLAG_UPDATE_TS_RECENT |   FLAG_NO_CHALLENGE_ACK) > 0;      if (!acceptable) { //如果不合法   if (sk->sk_state == TCP_SYN_RECV) //挥手过程中不会进入这个分支   return 1; /* send one RST */   tcp_send_challenge_ack(sk, skb); //回传一个ack然后丢弃   goto discard;   }   `

假如这里对 fin-20622 的 ack 检查没有通过，那么也会发送一个 ack（即包 14914, 这段代码中为 challenge ack），然后丢弃掉（没有进入处理 fin 的流程）。这和问题场景是非常符合的。继续分析 tcp_ack()函数，也找到了可能会判定非法的点：

`/*这一段是判断收到的ack值与本地发送窗口的关系，   这里snd_una意为send un-acknowledge，即发送了，但未被ack的位置   */   if (before(ack, prior_snd_una)) { //如果收到的ack值，已经被前面的包ack了   /* RFC 5961 5.2 [Blind Data Injection Attack].[Mitigation] */   ···   goto old_ack;   }   ···   old_ack:   /* If data was SACKed, tag it and see if we should send more data.   * If data was DSACKed, see if we can undo a cwnd reduction.   */   ···      return 0;   `

总结一下：fin-20622 有一种可能的处理路径，符合问题场景的表现。从 server 的视角：

- 首先收到ack-20623，更新了 snd_una 的值为该包的 ack 值，即 754。

- 然后收到 fin-20622，在检查 ack 值的阶段，由于该包的 ack=753，小于此时的 snd_nxt，因此被判定为 old_ack，非法。之后 acceptable 返回值为 0。

- 由于 ack 值被判定为非法，内核传回一个 challenge ack 包， 然后直接丢掉fin-20622。

- 因此，最终 fin-20622 被 tcp_rcv_state_process 丢弃，没有进入 fin 包处理的流程。

这样，相当于 server 并没有收到 fin 信号，与问题场景吻合。

找到了这一条可疑路径，接下来就要想办法验证了。

由于精确到了具体的代码片段，并且实际代码相当复杂，仅通过代码分析很难确定真实的运行路径。

于是我们放出大招，直接修改内核，验证上述位点的 tcp 状态信息，主要是状态转移和发送窗口。

## **修改内核配合测试**

具体过程不再赘述，我们有了新的发现：

（提示：使用的依然是“表现正常”的复现脚本）

1. 收到 ack-20623 时，snd_una 确实被更新了，这符合上面的假设，为 fin 包丢弃提供了条件。

1. 乱序的 fin 包根本没有进入 tcp_rcv_state_process()函数，而是被外层的tcp_v4_rcv()函数按照 TIMEWAIT 流程直接处理，最终关闭连接。

显然，第二点很可能是导致复现失败的关键。

- 更加证明了我们先前的假设，如果fin能进入 tcp_rcv_state_process()函数，应该就能复现出问题。但可能因为线上场景与复现场景存在某些配置差异，导致代码路径分歧。

- 另外这个发现也颠覆了我们的认知，按照 tcp 的挥手流程，在收到 fin-20622前，server 发送 fin 后收到了 ack，那么应当处于 FIN_WAIT_2 状态，工具监控结果也是如此，为何这里是 TIMEWAIT 呢。

带着这些问题，我们回到代码中，继续分析。在 ack 检查和 fin处理之间，找到一处最可疑的位置：

`case TCP_FIN_WAIT1: {   int tmo;   ···      if (tp->snd_una != tp->write_seq) //一种异常情况，还有数据待发送   break; //可疑      tcp_set_state(sk, TCP_FIN_WAIT2); //转移至FIN_WAIT2，并且关闭发送方向   sk->sk_shutdown |= SEND_SHUTDOWN;      sk_dst_confirm(sk);      if (!sock_flag(sk, SOCK_DEAD)) { //延迟关闭   /* Wake up lingering close() */   sk->sk_state_change(sk);   break; //可疑   }   ···   //可能会进入timewait相关的逻辑   tmo = tcp_fin_time(sk); //计算fin超时   if (tmo > sock_net(sk)->ipv4.sysctl_tcp_tw_timeout) {   //如果超时时间很大，则启动keepalive timer探活   inet_csk_reset_keepalive_timer(sk,   tmo - sock_net(sk)->ipv4.sysctl_tcp_tw_timeout);   } else if (th->fin || sock_owned_by_user(sk)) {   /* Bad case. We could lose such FIN otherwise.   * It is not a big problem, but it looks confusing   * and not so rare event. We still can lose it now,   * if it spins in bh_lock_sock(), but it is really   * marginal case.   */   inet_csk_reset_keepalive_timer(sk, tmo);   } else { //否则直接进入timewait；经过测试，复现失败时ack包进入了这个分支   tcp_time_wait(sk, TCP_FIN_WAIT2, tmo);   goto discard;   }   break;   }   `

这个片段对应 ack-20623 的处理过程，确实发现了和 TIMEWAIT 的关联，所以我们怀疑到前面的两个 break 上。如果提前触发了 break，是不是就不会导致 TIMEWAIT，进而能够复现成功？

话不多说直接动手，通过修改代码，发现两个 break 任意触发一个，都能够复现出问题场景，导致连接无法正常关闭！

对比两个 break 的条件，**SOCK_DEAD** 成为最大嫌疑者。

## **关于 SOCK_DEAD**

从字面意思推测，这个 flag 应当和 tcp 的关闭过程有关，在内核代码中查找，发现两处相关的函数：

`/*   * Shutdown the sending side of a connection. Much like close except   * that we don't receive shut down or sock_set_flag(sk, SOCK_DEAD).   */      void tcp_shutdown(struct sock *sk, int how)   {   /* We need to grab some memory, and put together a FIN,   * and then put it into the queue to be sent.   * Tim MacKenzie(tym@dibbler.cs.monash.edu.au) 4 Dec '92.   */   if (!(how & SEND_SHUTDOWN))   return;      /* If we've already sent a FIN, or it's a closed state, skip this. */   if ((1 << sk->sk_state) &   (TCPF_ESTABLISHED | TCPF_SYN_SENT |   TCPF_SYN_RECV | TCPF_CLOSE_WAIT)) {   /* Clear out any half completed packets. FIN if needed. */   if (tcp_close_state(sk))   tcp_send_fin(sk);   }   }   EXPORT_SYMBOL(tcp_shutdown);   `

从注释可以看出，这个函数具有close的一部分功能，但是不会sock_set_flag(sk, SOCK_DEAD)。那么再看一看tcp_close()：

`void tcp_close(struct sock *sk, long timeout)   {   struct sk_buff *skb;   int data_was_unread = 0;   int state;      ···   if (unlikely(tcp_sk(sk)->repair)) {   sk->sk_prot->disconnect(sk, 0);   } else if (data_was_unread) {   /* Unread data was tossed, zap the connection. */   NET_INC_STATS(sock_net(sk), LINUX_MIB_TCPABORTONCLOSE);   tcp_set_state(sk, TCP_CLOSE);   tcp_send_active_reset(sk, sk->sk_allocation);   } else if (sock_flag(sk, SOCK_LINGER) && !sk->sk_lingertime) {   /* Check zero linger _after_ checking for unread data. */   sk->sk_prot->disconnect(sk, 0);   NET_INC_STATS(sock_net(sk), LINUX_MIB_TCPABORTONDATA);   } else if (tcp_close_state(sk)) {   /* We FIN if the application ate all the data before   * zapping the connection.   */   tcp_send_fin(sk); //发送fin包   }      sk_stream_wait_close(sk, timeout);      adjudge_to_death:   state = sk->sk_state;   sock_hold(sk);   sock_orphan(sk); //这里会设置SOCK_DEAD flag   ···   }   EXPORT_SYMBOL(tcp_close);   `

这里，tcp_shutdown和tcp_close都是tcp协议的标准接口，可以用于关闭连接：

`struct proto tcp_prot = {   .name = "TCP",   .owner = THIS_MODULE,   .close = tcp_close, //close在这   .pre_connect = tcp_v4_pre_connect,   .connect = tcp_v4_connect,   .disconnect = tcp_disconnect,   .accept = inet_csk_accept,   .ioctl = tcp_ioctl,   .init = tcp_v4_init_sock,   .destroy = tcp_v4_destroy_sock,   .shutdown = tcp_shutdown, //shutdown在这   .setsockopt = tcp_setsockopt,   .getsockopt = tcp_getsockopt,   .keepalive = tcp_set_keepalive,   .recvmsg = tcp_recvmsg,   .sendmsg = tcp_sendmsg,   ···   };   EXPORT_SYMBOL(tcp_prot);      `

综上，shutdown 和 close 的一个重要差异在于 shutdown 不会设置 SOCK_DEAD。

我们将复现脚本的 close()换成 shutdown()再测试，终于成功复现了 fin 被丢弃的结果！（并且通过打印日志，确定丢弃原因就是之前提到的 old_ack，终于验证了我们的假设。）

下面只需要回归线上场景，确认是否真的调用了 shutdown()关闭连接。经过线上同学的确认，此处 server 确实是用了 shutdown()关闭连接（通过 nginx 的 lingering_close）。

至此，终于真相大白！

_**03**_

**总结**

最后，回答最初的两个问题作为总结：

- 该现象是否是内核的合法行为？

- 是合法行为，是内核检查 ack 的逻辑导致的。

- 内核会根据收到的 ack 值，更新发送窗口参数 snd_una，并由 snd_una 判断 ack 包是否需要处理。

- 由于 fin-20622 的 ack 值小于 ack-20623，且 ack-20623 先到达，更新了snd_una。后到达的fin在ack检查过程中，对比snd_una时被认为是已经ack过的包，不需要再处理，结果被直接丢弃，并回传一个challenge_ack。导致了问题场景。

- 为什么本地复现失败了？

- 关闭 tcp 连接时，使用了 close()接口，而线上环境使用的是 shutdown()。

- shutdown 不会设置SOCK_DEAD，而 close 则相反，导致复现时的代码路径与问题场景出现分歧。

_**04**_

**番\*\*\*\*外：close()下的tcp状态转移**

其实还遗留了一个问题：

为什么用 close()关闭连接时，没有观察到 fin 包的状态转移 FIN_WAIT_2 -> TIMEWAIT（没有进入 tcp_rcv_state_process）？

这要从 FIN_WAIT_1 收到 ack 后讲起，上面的代码分析中提到，如果没有触发两个可疑的 break，处理 ack 时将会进入：

`case TCP_FIN_WAIT1: {   int tmo;   ···   else {   tcp_time_wait(sk, TCP_FIN_WAIT2, tmo);   goto discard;   }   break;   }      `

tcp_time_wait()主要逻辑如下：

`/*   * Move a socket to time-wait or dead fin-wait-2 state.   */   void tcp_time_wait(struct sock *sk, int state, int timeo)   {   const struct inet_connection_sock *icsk = inet_csk(sk);   const struct tcp_sock *tp = tcp_sk(sk);   struct inet_timewait_sock *tw;   struct inet_timewait_death_row *tcp_death_row = &sock_net(sk)->ipv4.tcp_death_row;      //创建tw，其中将tcp状态置为TCP_TIME_WAIT   tw = inet_twsk_alloc(sk, tcp_death_row, state);      if (tw) { //创建成功，则会进行初始化   struct tcp_timewait_sock *tcptw = tcp_twsk((struct sock *)tw);   const int rto = (icsk->icsk_rto << 2) - (icsk->icsk_rto >> 1); //计算超时时间   struct inet_sock *inet = inet_sk(sk);      tw->tw_transparent = inet->transparent;   tw->tw_mark = sk->sk_mark;   tw->tw_priority = sk->sk_priority;   tw->tw_rcv_wscale = tp->rx_opt.rcv_wscale;   tcptw->tw_rcv_nxt = tp->rcv_nxt;   tcptw->tw_snd_nxt = tp->snd_nxt;   tcptw->tw_rcv_wnd = tcp_receive_window(tp);   tcptw->tw_ts_recent = tp->rx_opt.ts_recent;   tcptw->tw_ts_recent_stamp = tp->rx_opt.ts_recent_stamp;   tcptw->tw_ts_offset = tp->tsoffset;   tcptw->tw_last_oow_ack_time = 0;   tcptw->tw_tx_delay = tp->tcp_tx_delay;      /* Get the TIME_WAIT timeout firing. */   //确定超时时间   if (timeo < rto)   timeo = rto;      if (state == TCP_TIME_WAIT)   timeo = sock_net(sk)->ipv4.sysctl_tcp_tw_timeout;      /* tw_timer is pinned, so we need to make sure BH are disabled   * in following section, otherwise timer handler could run before   * we complete the initialization.   */   //更新维护timewait sock的结构   local_bh_disable();   inet_twsk_schedule(tw, timeo);   /* Linkage updates.   * Note that access to tw after this point is illegal.   */   inet_twsk_hashdance(tw, sk, &tcp_hashinfo); //加入全局哈希表（tcp_hashinfo）   local_bh_enable();   } else {   /* Sorry, if we're out of memory, just CLOSE this   * socket up. We've got bigger problems than   * non-graceful socket closings.   */   NET_INC_STATS(sock_net(sk), LINUX_MIB_TCPTIMEWAITOVERFLOW);   }      tcp_update_metrics(sk); //更新tcp统计指标，不影响本次行为   tcp_done(sk); //销毁掉sk   }   EXPORT_SYMBOL(tcp_time_wait);   `

可见，在这个过程中，原本的 sk 被销毁了，并且创建了对应的 inet_timewait_sock，进入计时。换言之，close的server 收到 ack 时，虽然会进 入FIN_WAIT_2，但是之后立即切换到了 TIMEWAIT 状态，且没有经过标准的tcp_set_state()函数，致使 eBPF 没有监控到。

之后再收到 fin 包时，则根本不会进入 tcp_rcv_state_process()，而是由外层tcp_v4_rcv()进行 timewait 流程处理。具体来讲，tcp_v4_rcv()将根据收到的 skb查询对应的内核 sk，这里会查到上面创建的 timewait_sock，其状态为TIMEWAIT，所以直接进入 timewait 的处理，核心代码如下：

`int tcp_v4_rcv(struct sk_buff *skb)   {   struct net *net = dev_net(skb->dev);   struct sk_buff *skb_to_free;   int sdif = inet_sdif(skb);   int dif = inet_iif(skb);   const struct iphdr *iph;   const struct tcphdr *th;   bool refcounted;   struct sock *sk;   int ret;   ···   th = (const struct tcphdr *)skb->data;   ···   lookup:   sk = __inet_lookup_skb(&tcp_hashinfo, skb, __tcp_hdrlen(th), th->source,   th->dest, sdif, &refcounted); //从全局哈希表tcp_hashinfo中查询sk   ···   process:   if (sk->sk_state == TCP_TIME_WAIT)   goto do_time_wait;   ···   do_time_wait: //正常的timewait处理流程   ···   goto discard_it;   }   `

综上，server 调用close()关闭连接，收到ack后会转入 FIN_WAIT_2，然后立刻转移为 TIMEWAIT，不需要等待 client 的 fin 包。

一种简单的定性理解：调用 close()的 socket 意味着完全关闭接收和发送，这样进入FIN_WAIT_2 等待对方的 fin 意义不大（等待对方fin的一个主要目的是确定对方发送完毕），所以在确认己方发送的 fin 被对面收到之后（收到了 client 针对 fin 的ack），就可以进入 TIMEWAIT 状态了。

### —— 完 ——

阅读 1835

​

写留言

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/SeWfibBcBT0EibtIWVNvshnuWMN1AoJw3poFIsbpaIVyZibCCqwBUR21rcDfrQgoqYzaYNdS14IIXzvmzvibdDa5Rw/300?wx_fmt=png&wxfrom=18)

Linux内核之旅

2212110

写留言

写留言

**留言**

暂无留言
