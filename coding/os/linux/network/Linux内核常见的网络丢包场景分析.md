# 

Linux开发架构之路

 _2024年09月05日 21:16_ _湖南_

## 

摘要

一个数据包在网络中传输的过程中，是没法保证一定能被目的机接收到的。其中有各种各样的丢包原因，今天来学习一下数据包经过 linux 内核时常见的丢包场景。

## 

1 收发包处理流程

有必要再回顾下 linux 内核的收发包处理流程，才能更好的认清楚数据包所经过的环节。

![Image](https://mmbiz.qpic.cn/sz_mmbiz_png/8pECVbqIO0xW2eEPkZDvlJyUMBcaQADG5mpq3icKSpjNmKDWNZ1NXpibOaFPZNVY5LEEhia8tVMwibumEcQzIfHT6w/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

  

当网卡接收到报文时，将数据包 DMA 拷贝到 RingBuf 并触发一个硬中断。继而 cpu 开始执行对应的（硬）中断处理例程。在硬中断处理例程中，将数据包 list 放在了每 cpu 变量 poll_list 中，紧接着触发了一个收包软中断。对应 cpu 的软中断线程 ksoftirqd 处理网络包接收软中断（ net_rx_action() ），将数据包从 RingBuf 中取出，协议栈层层处理，经网络层到传输层，数据包被放到 socket 的接收队列中。随后被应用层收取数据。

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

同样的，在发送数据时，skb 也是经过协议栈层层处理。在网络层对 skb 克隆，填充路由项，依次流经 netfilter 框架的 local_out 、post_routing 点。在邻居子系统填充了 mac 地址，在网络设备子系统中将 skb 放到了发送队列 RingBuf 中。在网卡发送完成后，又会通过硬中断触发软中断，在软中断处理函数中进行 RingBuf 的清理。

接下来进入丢包正题。

## 

2 硬件网卡相关

### 

2.1 ring buffer满

一般物理网卡丢包时，可以通过 ethtool -S 看到相应的统计信息：

```
[root@centos ~]# ethtool -s eht0
```

如果发现 rx_no_buffer_count 一直在增长，基本上是因为接收的 RingBuf 满导致的丢包。当然问题的核心原因还是接收流量太大& cpu 处理慢，导致数据包积压造成了丢包。具体的，可能有以下几种场景：

a 硬中断分发不均

通过 cat /proc/interrupts 来查看网卡的硬中断是否均衡，如果报文处理都集中在一个核上，那么很大可能会导致 cpu 处理不过来导致丢包。这种情况可以考虑关闭 irqbalance，使用 set_irq_affinity.sh 来进行手动绑核，使硬中断的分发更均匀。

```
[root@centos ~]# cat /proc/interrupts
```

b 会话分发不均

有时候发现硬中断已经是均衡的了，但网络流量触发的中断量级更大，并且集中在某几个核上面，这也是可能导致丢包的原因之一。对于支持多接收队列的网卡，通过 RSS 功能，根据数据包的某些字段（如源ip、目的ip、源端口、目的端口）将数据包哈希到不同的接收队列中，进而分发给不同的 cpu 进行处理。

查看网卡是否支持多队列：

```
[root@centos ~]# ethtool -l eth0
```

  

```
# 查看 udp 会话分发哈希关键字
```

通过 ethtool --config-ntuple 可以修改分发使用的哈希关键字。

c rps

如果网卡不支持多队列，或者支持的多队列远远小于CPU核数（当然这两种情况都比较少见了）。linux内核提供了 rps 机制来进行软中断的负载均衡。比如可以通过 echo ff > /sys/class/net/eth0/queues/rx-0/rps_cpus 来指定某个接收队列负载均衡到哪几个核处理。当然由于通过软件进行模拟，相比硬中断均衡，性能会下降很多，一般没有特殊原因，不建议开启。

d 突发流量

如果是间接性的突发流量导致丢包，可以通过 ethtool -g 来查看当前的 RingBuf 配置，并通过ethtool -G 命令加大 RingBuf 的 size，减少突发流量导致的丢包。

```
[root@centos ~]# ethtool -g eth0
```

需要C/C++ Linux服务器架构师学习资料加qun812855908获取（资料包括C/C++，Linux，golang技术，Nginx，ZeroMQ，MySQL，Redis，fastdfs，MongoDB，ZK，流媒体，CDN，P2P，K8S，Docker，TCP/IP，协程，DPDK，ffmpeg等），免费分享

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

### 

2.2 利用 ntuple 保证关键业务

有些情况下，需要保证控制报文高优先级进行处理，默认情况下，网卡无差别对待所有报文，通过 rss 技术来选择队列，无法区分控制报文。可以开启网卡的 ntuple 功能来实现，比如将 tcp 端口号为 23 的报文，定向到 queue 9，与此同时其他报文仍使用 rss 技术，并且设置为使用 8 个队列：

```
[root@centos ~]# ethtool -K eth0 ntuple on
```

当对相应的业务进行绑核处理后，tcp 的 23 端口即可以跟其他业务不冲突，保证其高优先级进行处理。当然这个方案依赖于底层网卡的实现，并不是所有网卡都支持的。

## 

3 arp丢包

### 

3.1 neighbor table overflow

arp 比较常见的一个问题是 neighbor 表满了，内核会大量打印如下的消息：

```
[22181.923055] neighbour: arp_cache: neighbor table overflow!
```

cat /proc/net/stat/arp_cache也能看到大量的 table_fulls 统计：

```
[root@centos ~]# cat /proc/net/stat/arp_cache 
```

主要原因跟arp的参数如下几个参数相关，将其改大到合适的值即可解决该问题：

```
# 保存在于ARP高速缓存中的最少层数，如果少于这个数，垃圾收集器将不会运行
```

### 

3.2 unresolved drops

当发送报文时，如果还没解析到 arp，就会发送 arp 请求，并缓存相应的报文。当然这个缓存是有限制的，默认为 SK_WMEM_MAX（即与 net.core.wmem_default 相同）。当大量发送报文并且 arp 没解析到时，就可能超过 queue_len 导致丢包，cat /proc/net/stat/arp_cache可以看到unresolved_discards 的统计：

```
[root@centos ~]# cat /proc/net/stat/arp_cache
```

可以通过调整 /proc/sys/net/ipv4/neigh/eth0/unres_qlen_bytes 参数来缓解该问题。

## 

4 conntrack丢包：nf_conntrack: table full

当连接数比较多的时候，可能遇到这个连接跟踪表满的错误。

```
[ 2306.529864] nf_conntrack: nf_conntrack: table full, dropping packet
```

conntrack 有个最大表项数（/proc/sys/net/netfilter/nf_conntrack_max）限制来控制。当 conntrack 数目达到 nf_conntrack_max 后，就会尝试将一些未 assured 状态的 conntrack 提前老化掉，尝试8次，未找到合适的可以提前老化的表项，就直接进行丢包处理。

出于性能的考虑，调大 nf_conntrack_max 的同时，一般也需要调整 nf_conntrack_buckets，建议 max/buckets 小于8 比较合适，这样即使被攻击了，也能快速的进行 early_drop。比如调整 max 到300万，buckets 最好也做相应的调整：

```
[root@centos ~]# echo 3000000 > /proc/sys/net/netfilter/nf_conntrack_max
```

## 

5 udp接收buffer满

当一个快的 udp sender，会导致一个较慢的 udp receiver socket recv buffer 满，导致丢包。尤其是存在大量突发的报文时。通过netstat查看 receive buffer errors 的统计就可以获知是否存在这种情况：

```
[root@centos ~]# netstat -su
```

一般通过修改 /proc/sys/net/core/rmem_max，并且设置 SO_RCVBUF 选项来增加 socket buffer 可以缓解该问题。

## 

6 丢包定位

### 

6.1 dropwatch 查看丢包

除了 tcpdump 抓包进行分析外，dropwatch 也是网络协议栈丢包检查利器。实现原理也比较清晰，在 net/core/drop_monitor.c 文件中，当 kfree_skb 被调用时，内核就会记录下来，并通过 netlink 来通知用户态的 dropwatch 来记录。使用起来也非常简单，可以看到所有调用 kfree_skb() 的地方都会被输出：

```
[root@centos ~]# dropwatch -l kas
```

### 

6.2 利用 iptables LOG 跟踪报文流程

有时候，可能会有多个路由出口，为了知道特定的报文是如何路由出去的，可以借助 iptables LOG 来进行跟踪（需要结合limit模块来做限速，避免输出过多）。如下可以看到 ping 包是通过eth0 接口出去的：

```
[root@centos ~]# iptables -I OUTPUT -m limit --limit 1/s -p icmp -j LOG
```

### 

6.3 利用 iptables 规则跟踪丢包

还记得 netfilter 框架的的 5 个hook点吗？

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

从图中可以看到，接收到报文，路由前，路由后，发送报文，都可以在配置 iptables 规则来进行跟踪。先梳理清楚数据包经过的 hook 点，接着配置 iptables 规则，使得协议命中但不会产生具体的动作，比如对于 ipsec 协议，通过以下两条规则统计的报文数观察在 OUTPUT 之后到 POSTROUTING 之间的丢包个数：

```
[root@centos ~]# iptables -t mangle -I OUTPUT -p esp
```

## 

7 总结

除了上面介绍的一些场景外，还有很多其他的场景，比如 ip 层分片重组导致的丢包、tc配置的丢包等等，这里不再一一举例来说明了。总的来说，多总结，多思考，善于使用 linux 内核提供给我们的工具，才能更好的排查问题。

**往期精彩回顾**

- **[linux内核开发是程序员的新风口？那linux内核该如何学习？](http://mp.weixin.qq.com/s?__biz=Mzg2MzQzMjc0Mw==&mid=2247497591&idx=1&sn=99d838e981dfc3ac12357e2f61cdae0b&chksm=ce7a0e3ff90d87299cb4aff5b3e94cc530684899d861dd707d73b3b75c190a08a336493efddb&scene=21#wechat_redirect)  
    **
    
- [DPDK入门指南，DPDK学习路线总结](http://mp.weixin.qq.com/s?__biz=Mzg2MzQzMjc0Mw==&mid=2247497642&idx=1&sn=8566996ea02f6e52f3ba40bf0bf0f1e9&chksm=ce7a0ee2f90d87f4e0e930c8acc9fe33247f51a0f00f31f187aafe747e1704e42b7b796f9ed4&scene=21#wechat_redirect)  
    
- [存储开发怎么样？如何学习C++存储开发技术？](http://mp.weixin.qq.com/s?__biz=Mzg2MzQzMjc0Mw==&mid=2247497068&idx=1&sn=71902e44963ce1f2f88680904c2761b4&chksm=ce7a0c24f90d8532ebc0f3feb38036782face63ccca579d83a621f90ba3c51d20eeeb852682a&scene=21#wechat_redirect)  
    
- [你离腾讯T8还有多远？最全最详细的C/C++后端开发技术栈](http://mp.weixin.qq.com/s?__biz=Mzg2MzQzMjc0Mw==&mid=2247496622&idx=1&sn=766d7b899916f94d40acfa4f37cb96a6&chksm=ce7a0ae6f90d83f04946be5843ee515cd41a92c71c93260535e7a805c527cc164c06c0270c02&scene=21#wechat_redirect)  
    
- [**音视频/流媒体开发工程师工作内容主要是在做什么 ?**](http://mp.weixin.qq.com/s?__biz=Mzg2MzQzMjc0Mw==&mid=2247497231&idx=1&sn=eceb84d323d030231a891acdb6a898c2&chksm=ce7a0f47f90d865132083d54275386e294bde708efcd1defdf9b6a8d793d208cdb6e980d4c53&scene=21#wechat_redirect)
    

Reads 3242

​

Comment

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/8pECVbqIO0wM26DrqMOOCH0fLxcaCMw0RTe7vAiaUYITgvtM3gtFu2SJUUTX3CN62BiaQfC3BjWVlDwwKqiaCpNIg/300?wx_fmt=png&wxfrom=18)

Linux开发架构之路

4354240

Comment

Comment

**Comment**

暂无留言