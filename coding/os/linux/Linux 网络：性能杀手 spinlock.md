
Linux内核之旅

 _2023年05月04日 19:59_ _陕西_

The following article is from eBPF Talk Author Leon Hwang


**eBPF Talk**.

专注于 eBPF 技术，以及 Linux 网络上的 eBPF 技术应用

](https://mp.weixin.qq.com/s?__biz=MzI3NzA5MzUxNA==&mid=2664615140&idx=1&sn=ed122d50dfbf213b5dbb677f064bf035&chksm=f04de101c73a68171c12770830b95816ddd689219a98030cceec6918ac2308b5ba182371ba1f&mpshare=1&scene=24&srcid=0505zL9zd2ko4BPkvPXLz1Vp&sharer_sharetime=1683265864901&sharer_shareid=8397e53ca255d0bca170c6327d62b9af&key=daf9bdc5abc4e8d0e466774336f48c0d983d82c380500aca9e7683f0aa2be816a29815f2d2a637c6566d5a22ae73eb98553110af14a4b6a589fb728daf81c9dde0632bb2c27fe025c55077c4c5fd18466cf2bb6062d5aa6d0f0d71592263a364ed88c474b753ae346a901bdb205ebce13b6894fbcafc473015805a2378fe12a6&ascene=14&uin=MTEwNTU1MjgwMw%3D%3D&devicetype=iMac+MacBookAir10%2C1+OSX+OSX+14.6.1+build(23G93)&version=13080710&nettype=WIFI&lang=en&session_us=gh_3e85a6de261f&countrycode=CN&fontScale=100&exportkey=n_ChQIAhIQG5Rednm2Xy2R0vlqR1qtfxKUAgIE97dBBAEAAAAAAH3qJLmB90oAAAAOpnltbLcz9gKNyK89dVj0UuBMr%2BIR6O3KWmttps5MYuVFVNLeXFOaiO6VJpQgbvgudPSGz0y4pnDwtBIFqlKNolUZndYW2XGRQARPY%2FI%2FL98%2F7mkN3codd48sz%2FlSCEXRJM2Lj6gXR1G15XvKS8%2FEaczI4TM0XgmCI2R3x83NOiPknxi1dGBIKCYdS4c%2B%2Bn3mbRYm0MJsdYR4CGRdc%2Fb5zbOe6ObdKCkWs%2F%2FQVSSQDfZcF%2F3fedV6ownkkH7AD6hlWGsmL3c%2F9cTtf2hukjKWQ29wHXEBsjEGf9oO8xEudqXUgmkufW013sWLokw1tSOgbV2iFxaAd07RdidSXw%3D%3D&acctmode=0&pass_ticket=MV13lecejYFTWw7FQJeNLBx7z%2F8It0550ZqxphjfFJk5XM8nLF9sjGU7O%2Bv4Dwj5&wx_header=0#)

![](http://mmbiz.qpic.cn/mmbiz_png/6QxwXGlhS9FPHblxYiaxMiaYWhCKBUs37o2XqGcg9ickZoh10hor44VRx8KT08XIaCYKCzUNBdxb2JWPCy6TFTzjw/300?wx_fmt=png&wxfrom=19)

**eBPF Talk**

专注于 eBPF 技术，以及 Linux 网络上的 eBPF 技术应用

139篇原创内容

公众号

在内核网络协议栈里，有不少地方使用到了 `spinlock`（此 `spinlock` 非 `struct bpf_spin_lock`）。如果使用不当，CPU 就会消耗在 `spinlock` 上。

最近正好死磕一个 Linux 网络性能问题，最终发现是 `spinlock` 导致的。

## 性能问题排查

因为是 Linux 网络性能问题，所以性能瓶颈在内核里，通过 `top` 等常规手段行不通了。

需要通过以下手段一顿操作：

1. `mpstat -p ALL 1`
    
2. `perf top --cpu=3`
    
3. `timeout 5 bpftrace -e 'kprobe:native_queued_spin_lock_slowpath{@[stack(12)] = count()}'`
    
4. 参考 CPU Flame Graphs[1] 制作 CPU 火焰图
    

最终发现，性能瓶颈在于 `dev_queue_xmit()` 设备层发包时的 `native_queued_spin_lock_slowpath()` `spinlock` 上。

## 深挖问题原因

技术背景：网卡有 N 个 txqueue，每个 txqueue 对应一个 CPU，每个 CPU 都会调用 `__dev_xmit_skb()` 函数通过 txqueue 发包。

先看一下在这情况下，内核协议栈的设备层是如何发包的。

`dev_queue_xmit()                    // ${KERNEL}/net/core/dev.c   |-->__dev_queue_xmit() {       |   txq = netdev_core_pick_tx(dev, skb, sb_dev);       |   q = rcu_dereference_bh(txq->qdisc);       |   rc = __dev_xmit_skb(skb, q, dev, txq);       }       |-->__dev_xmit_skb() {           |   spinlock_t *root_lock = qdisc_lock(q);           |   spin_lock(root_lock);   // 1. lock           |   __qdisc_run(q);           }           |-->__qdisc_run()           // ${KERNEL}/net/sched/sch_generic.c               |-->qdisc_restart()                   |-->sch_direct_xmit() {                           spin_unlock(root_lock);                           dev_hard_start_xmit()                           spin_lock(root_lock);   // 2. lock                       }   `

根据 bpftrace 的结果，分析出有 2 个地方调用 `spin_lock()` 函数上锁了。

## 解惑

疑问：这个 `spinlock` 从属于哪个 `qdisc` 呢？

从上面代码片段可知，这个 `spinlock` 来自 txqueue 上的 `qdisc`。

继续问：txqueue 上的 `qdisc` 是从哪里来的呢？

似乎没有简单的答案，因为涉及到了复杂的 tc 子系统。

直接分析一下 tc 里给设备 EGRESS 创建 `qdisc` 的过程吧。

`tc_modify_qdisc()                  // ${KERNEL}/net/sched/sch_api.c   |-->qdisc_graft() {           for (i = 0; i < num_q; i++) {               dev_queue = netdev_get_tx_queue(dev, i);               old = dev_graft_qdisc(dev_queue, new);           }       }   `

这是每个 txqueue 都嫁接了同一个 `qdisc`。

## 解法

似乎没有比较直接简单的解法，只能从根本上解决问题。

奇葩需求，奇葩解法：分而治之，避免所有 txqueue 共享一个 `qdisc`。

因奇葩需求，所以需要将网络包从网卡接收到内核协议栈，在设备层特殊处理一下后，再从网卡发出去。

而网卡有 N 个 rxqueue，解法就是为每个 rxqueue 都创建一对 veth 设备、每对 veth 设备都有 N 个 txqueue，最后从 veth 对端设备将网络包转发到网卡发送出去。

如此，网卡上就没有 tc EGRESS 规则了，这里就不会有 `spinlock` 问题了。

## 奇葩小需求：在 tc-bpf 里如何根据 rxqueue index 将网络包转发到指定 veth 设备呢？

根据 [eBPF Talk: tc-bpf 转发网络包](https://mp.weixin.qq.com/s?__biz=MjM5MTQxNTk5MA==&mid=2247484914&idx=1&sn=a61bd40e3497326d4e7dcd22abab4b18&scene=21#wechat_redirect)，tc-bpf 不支持 `bpf_redirect_map()` helper 函数，所以有没有办法做到类似 `bpf_redirect_map()` 的功能呢？

答案是肯定的，而且有多种方法。

1. 使用一个 bpf map 保存 veth 设备的 ifindex，使用 rxqueue index 当作 key 查询得到 veth 设备的 ifindex。
    
2. 把该 bpf map 改为一个 ifindex 数组的全局变量，使用 rxqueue index 当作索引查询得到 veth 设备的 ifindex。
    
3. 把该全局变量改为一个 ifindex 数组的常量，使用 rxqueue index 当作索引查询得到 veth 设备的 ifindex。
    

得到 ifindex 后使用 `bpf_redirect(ifindex, 0)` 函数将网络包转发到指定 veth 设备。

## 小结

1. `spinlock` 会导致 CPU 消耗，特别是在多 CPU 上。
    
2. 用分治法，避免多个 CPU 共享一个 `spinlock`。
    

### 参考资料

[1]

CPU Flame Graphs: _https://www.brendangregg.com/FlameGraphs/cpuflamegraphs.html_

Reads 4982

​

People who liked this content also liked

虚拟内存、用户空间 、内核空间、用户态、内核态

我常看的号

Linux开发架构之路

不喜欢

不看的原因

OK

- 内容低质
- 不看此公众号内容

![](https://mmbiz.qpic.cn/sz_mmbiz_jpg/8pECVbqIO0wsszqy9Yd9HRlAqk50XMhaAicWG3zVOFKOuv66bsb5UuCPtIJ57VvKvMrjIbuxPKbIYjUdr4CgMkA/0?wx_fmt=jpeg)

CPU性能分析与优化（三）

我关注的号

CPP每周推送

不喜欢

不看的原因

OK

- 内容低质
- 不看此公众号内容

![](https://mmbiz.qpic.cn/mmbiz_jpg/UZD17icqbEY6VvXLSYicicib6M5NdgOibkQPiaugVRiay56JSRBJzKv36jBQF8ibl7W8YYyTwATDyMJs9meFWAjntguq6w/0?wx_fmt=jpeg&tp=wxpic)

你最应该读的Rust最佳书籍

我关注的号

coding到灯火阑珊

不喜欢

不看的原因

OK

- 内容低质
- 不看此公众号内容

![](https://mmbiz.qpic.cn/mmbiz_jpg/aZSfDXbVYlPh39AIUJyDVicYOMdOfviasXzVgOwaqibaYStZ37RGBiaUOuVc98wWlwibKrRAWmQp8HqR7Ahw9WuuDxQ/0?wx_fmt=jpeg&tp=wxpic)

Comment

**留言 3**

- 超人腿短
    
    浙江2023年5月29日
    
    Like1
    
    你的需求不需要全局流控的话，用sch_mq就可以解决啊现成的～
    
- 风二中
    
    北京2023年5月5日
    
    Like
    
    看不懂
    
- zhang
    
    北京2023年5月5日
    
    Like
    
    这个是什么场景下出现的呢？
    

已无更多数据

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/SeWfibBcBT0EibtIWVNvshnuWMN1AoJw3poFIsbpaIVyZibCCqwBUR21rcDfrQgoqYzaYNdS14IIXzvmzvibdDa5Rw/300?wx_fmt=png&wxfrom=18)

Linux内核之旅

1855

3

Comment

**留言 3**

- 超人腿短
    
    浙江2023年5月29日
    
    Like1
    
    你的需求不需要全局流控的话，用sch_mq就可以解决啊现成的～
    
- 风二中
    
    北京2023年5月5日
    
    Like
    
    看不懂
    
- zhang
    
    北京2023年5月5日
    
    Like
    
    这个是什么场景下出现的呢？
    

已无更多数据