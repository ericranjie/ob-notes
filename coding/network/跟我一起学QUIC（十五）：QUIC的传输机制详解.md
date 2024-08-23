
原创 网络小菜鸟 网络小菜鸟

 _2024年03月11日 08:00_ _广东_

我们知道，QUIC是一个基于UDP的可靠性传输协议，既然UDP是不可靠的传输，那么就应该在QUIC上层采取机制来保障数据的可靠传输。在可靠传输机制上，QUIC汲取了TCP的思想，实现了ACK机制、丢包检测机制、拥塞控制算法等，但在实现上又与TCP稍有不同，体现了更为优秀的传输机制。本文将一起来探讨QUIC的传输机制，是如何保障QUIC的可靠性的。  

PS：本文是QUIC的最核心的东西，涉及理论知识的讲解，公式算法的介绍，还有代码实现的介绍。代码实现是以Nginx的实现来讲解的，Nginx的实现完全按照协议规范来实现的。文章内容不容易理解，有一定难度，并且很长，请耐心~  

  

**1、QUIC传输机制的基础框架**  

在一条连接的生命周期中，会存在三个空间，分别是初始空间、握手空间、应用数据空间；在每个空间中的数据包号是不会重复的，在同一个空间中数据包号是单调递增的。这一点与TCP差别很大。这个设计让QUIC的丢包检测比TCP的降低了大量的复杂度。

详细讲解可见：[跟我一起学QUIC（十二）：数据包](http://mp.weixin.qq.com/s?__biz=Mzg5NzYzNDU2Ng==&mid=2247484427&idx=1&sn=81af363a911e6440b1e7615915b86806&chksm=c06f9c62f71815741e41d10bcfc8b9a678fb2e7e85344b2562773a377c850bb9b72d27480ba2&scene=21#wechat_redirect)  

任一QUIC数据包的发送方除非被对端确认了，不然就会被认为丢包了。当然有些QUIC帧是不需要被ACK的，这些帧往往跟其他需要ACK的帧放在同一个数据包中被发送；如果发送的一个数据包中 只包含不需要被ACK的帧，那么这个数据包不会被单独ACK，会与其他数据包一起被ACK。

**2、QUIC与TCP间的相关差异**  

**2.1 单独的数据包号空间**  

QUIC为每个密级使用单独的数据包号空间，单独的数据包号空间确保 某一密级发送的数据包的丢失 不会引起另一个密级发送的数据包的重传。  

**2.2 单调递增的数据包号**  

TCP要求有序性，即是接收到的数据包的顺序必须要与发送的顺序一致。所以当某个包丢失了之后，TCP要求重传包的包序号必须和丢失包的序号是一样的。正是这种特性导致RTT计算时出现二义性，从而RTT的测量不准确，或偏大或偏小。影响传输效率。

QUIC通过数据包号代表发送顺序，但接收顺序不由数据包号决定，接收顺序靠流帧中的流偏移决定的。当某个包丢失时，重传包的包序号会递增，只是流偏移相同。这样基于数据包号进行计算RTT的时候，不会出现二义性，这一设计极大地简化了QUIC的丢包检测机制。  

**2.3 更准确的丢包计时**  

在连续出现丢包的场景下，QUIC与TCP对拥塞窗口的计算是有区别的。  

这里涉及到一个“丢失时期”的概念，表示数据包从丢失到重传恢复这一段时间。在这个时期内，传输协议采取相应的机制进行丢包检测、恢复丢失的数据包和调整拥塞控制策略。

TCP：丢失时期是指发现数据包丢失（通过重传计时器超时或连续重复的ACKs）后的那段时间，TCP会尝试重新传输丢失的数据段，并调整拥塞窗口。TCP的丢失时期持续到序列号空间中的间隙被填满，即所有按序列排列的已发送数据段都得到确认为止。

QUIC：丢失时期是指从丢失数据包开始，直到该时期后发送的任何数据包得到确认的期间。在这个过程中，QUIC采取了响应更快的丢包检测机制，同时调整拥塞窗口。

所以，在丢失时期出现多个数据包连续丢失的情况下，TCP只会调整一次拥塞窗口（减小窗口），而QUIC会出现多次，理论上有多少个包丢失了就会出现多少次调整窗口机会。这在网络极差的情况下，QUIC更能有效地调节网络拥塞状况。

**2.4 No Reneging**

在 TCP 协议中，选择性确认（Selective Acknowledgments，SACKs）是一种特性，它允许接收方指定在乱序的情况下正确收到的数据段，这样发送方可以只重传确实丢失的数据段，而不是重传所有后续的数据段。这种技术可以显著提高在存在丢包和乱序的网络环境中的数据传送效率。

然而，在 QUIC 协议中，虽然 ACK Frames 含有与 TCP SACKs 类似的信息，允许接收方告诉发送方它已经成功接收哪些数据包，但 QUIC 不允许已确认的数据包被撤销（reneged）。也就是说，一旦一个数据包被指示为接收到了，就不能再将其作为未接收来处理。这种规则极大地简化了发送方和接收方协议实现的复杂性，并且减少了发送方的内存压力，因为发送方不再需要为可能会被撤销的确认保存信息。

**2.5 更多ACK块**  

TCP支持三个SACK块，而QUIC支持更多个ACK块。在高丢包率的环境下，能够加速恢复，减少无效重传。

**2.6 ACK Delay**  

在传统的TCP协议中，并没有显式包含ACK延迟（Ack Delay）这个概念。在TCP中，发送方根据发送数据包的时间以及接收到确认（ACK）的时间来计算往返时间（RTT），进而估算超时重传时间（RTO）以及拥塞窗口等参数。

然而，在TCP中，接收方会根据一定的策略和条件对确认消息进行延迟发送，这个过程通常称为延迟确认（Delayed ACK）。延迟确认允许接收方将多个接收到的数据包的确认合并在一个确认消息中发送，从而减少网络中的确认流量。TCP中的延迟确认算法可能会对RTT的计算产生一定的影响。

相较于TCP，QUIC协议中显式地引入了ACK延迟这个概念，即测量接收方在收到数据包时和发送确认消息之间间经历的延迟。在QUIC中，接收方会将这个延迟值包含在确认信息中告诉发送方，使得发送方减去这个ACK延迟后更准确地估计网络的RTT，以便向网络中调整传输参数。这就是TCP和QUIC在处理确认延迟的主要区别。

ACK Delay，或者称为“延迟确认”，有其优缺点，其影响向网络传输效率的具体效果取决于所处网络环境的特性。

优点方面，延迟确认可以聚合多个确认信息（尤其是在接收到多个较小的数据包的情况下），减少了被发送到网络中的确认包的数量，降低了网络开销，节省了带宽。

然而，缺点是，延迟确认可能会增加网络中的往返时间（RTT）。因为发送方需要等待接收方的确认信息来确认数据包已成功到达，并据此来计算 RTT 让其进一步用于超时重传时间和拥塞窗口等因素的确定。如果接收方使用了延迟确认，可能会导致发送方得到一个比实际网络条件更大的 RTT 估计值，从而可能影响 TCP 的性能，例如可能会导致发送方不必要的等待以及数据包的重传。

所以，延迟确认引入了一种权衡：通过减少确认包以节省带宽，但可能以增加一些 RTT 和影响某些 TCP 机制性能为代价。具体效果取决于多种因素，包括网络状况，数据包大小，数据包发送的时间间隔和具体的 TCP 实现等。

**2.7 QUIC用PTO代替了TCP的RTO和TLP**  

RTO是在TCP中使用的一个概念，主要用于处理数据包的丢包和重传。当TCP发送一个数据段后，会启动重传计时器等待接收方的确认。如果发送方在RTO时间内没有接收到接收方的确认信息，TCP会假定这个数据段丢失，然后重传这个数据段。RTO的长度是根据网络状况动态计算的，主要考虑的因素包括网络的往返时间（RTT）和对RTT的偏差等估计。

PTO则是在QUIC协议中引入的一个新概念，用于解决在丢包检测和重传中的问题。当QUIC的连续两个数据包在一个RTT内没有得到确认时，就会触发一个PTO事件。在PTO发生时，QUIC并不立即判断数据包丢失，而是发送一个新的数据包来探测路径，看是否是因为网络拥塞或其他原因导致的无法传输。这个机制允许QUIC更早地探测到潜在的丢包，从而提早进行数据的重传，提高数据传输的效率和可靠性。

**3、QUIC的RTT的计算**  

在QUIC协议中，RTT的测量对于ping控制和丢失检测是至关重要的。发送端需要计算并记录最小RTT (min_rtt)，平滑RTT (smoothed_rtt)，RTT方差 (rttvar)

以下是每个值的具体定义和计算方法：

- 最小RTT (min_rtt)：这是自连接开始以来观测到的最小的RTT值。It is reset at the start of every network path
    
- 平滑RTT (smoothed_rtt) 以及RTT方差 (rttvar)：它们都是基于新的RTT样本值的算术平均值，计算的公式在RFC中有具体的描述。这两个值都用于确定重传超时时间RTO。
    
- RTT的平均偏差(rtta村)：这是观测到的RTT样本中的平均偏差。它体现了网络路径RTT的波动程度，可以被用来帮助预测未来的RTT可能的变化范围。
    

**3.1 RTT采样**

一个端点在接收到满足以下两个条件的ACK（确认）帧时生成一个RTT样本：

- 最大的已确认数据包编号是新确认的；
    
- 至少有一个新确认的数据包是需要确认的（ack-eliciting）。
    

RTT样本（latest_rtt）是生成的，作为发送最大已确认数据包后所经过的时间：

```
latest_rtt = ack_time - send_time_of_largest_acked
```

  

**3.2 最小RTT**

TCP中没有显式地记录min_rtt，QUIC中显式记录了min_rtt，并在拥塞控制算法中用到了它。

QUIC的min_rtt，一定是该条连接上所有RTT样本中的最小值，它的初值是第一个样本RTT的值。

min_rtt的作用有哪些？  

- 损失检测：min_rtt 可以帮助确定拥塞或丢包的情况。当新的 RTT 样本比 min_rtt 小得不合理时，损失检测将拒绝该样本。这样做可以剔除异常值，避免错误地判断网络状态。
    
- 拥塞控制：在 QUIC 的拥塞控制算法中，min_rtt 是一个重要参考值。拥塞控制算法需要根据 RTT 的变化调整发送速率。通过考虑 min_rtt，拥塞控制可以据此来区分正常的 RTT 波动和实际的拥塞现象，从而作出更加精确的调整。
    
- 计算 PTO（Probe Timeout）：QUIC 使用 PTO 值来确定在未收到 ACK 时触发重传的时机。PTO 的计算基于 min_rtt，以确保得到一个合理的超时值，避免过早或过晚地触发重传。
    
- 适应网络变化：QUIC 协议可以根据网络状况的变化（如拥塞解除）定期更新 min_rtt，以改进对网络延迟的估计。这提高了 QUIC 的自适应能力。
    

**3.3 平滑RTT和RTT方差**  

QUIC协议中的smoothed_rtt（平滑往返时间）是基于多个RTT（Round-Trip Time，往返时间）样本计算得出的一种更稳定的RTT估计，主要用于连接时延估计以及拥塞控制算法。

rttvar（RTT方差）是度量 RTT 总体波动情况的指标，表示往返时间样本的变异性。

我们来看nginx中对这平滑RTT和RTT方差是如何计算的。

初始状态计算方式如下：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

在nginx中，smoothed_rtt用变量avg_rtt表示。初始值是333ms，最小rtt的初始值是-1，rtt方差（rttvar）是116ms。  

当得到第一个RTT样本后，计算方式如下：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

在得到后续RTT样本后，计算方式如下：  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

这段代码需要解释一下：ack_delay（确认延迟）在平滑RTT的计算过程中起到了重要作用。如果发现本次采样得到的RTT过大，那么有理由怀疑是ack_delay导致的RTT膨胀，所以此次RTT采样需要将ack_delay减掉。减掉之后的采样RTT用adjusted_rtt来表达。那么如何判断RTT是否过大？

```
        if (qc->min_rtt + ack_delay < latest_rtt) {
```

然后再来计算平滑RTT和方差RTT

```
qc->avg_rtt += (adjusted_rtt >> 3) - (qc->avg_rtt >> 3);
```

  

**4、QUIC丢包检测**

QUIC的数据发送方使用ACK机制来检测数据包是否丢失。而ACK机制是依赖PTO算法来实施的。本章介绍这些算法。  

**4.1 丢包检测**  

在两种情况下，可以被认定为丢包。  

- 某个已发送的数据包，在其后发送的两个数据包都已经被ACK了，但仍未收到该数据包的ACK，那么就认为这个数据包丢了。这个叫数据包数量阈值，一般设置为3，代表当前数据包的后面两个数据包被ACK了。这种情况多发生在包乱序的网络中。数据包乱序在QUIC中比在TCP中更为常见。在nginx中宏定义如下：
    

```
/* RFC 9002, 6.1.1. Packet Threshold: kPacketThreshold */
```

- 某个已发送的数据包，超过了一定的时间仍未收到它的ACK，也会被认定为丢包了。这个数据包超时时间阈值不是一个固定值，它是由我们在第3章中提到的平滑RTT 以及 最近一次采样的RTT来决定的。  
    

```
/* RFC 9002, 6.1.2. Time Threshold: kTimeThreshold, kGranularity */
```

这里需要解释一下：上面代码是计算这个阈值的，可以翻译成公式：

thr = max(kTimeThreshold * max(平滑RTT, latest_rtt), kGranularity)

kTimeThreshold被称之为RTT倍率，在nginx当中，被设定为9/8（而在TCP中，RTT倍率是5/4），这也是RFC建议的值。kGranularity被称为计时器粒度，设置为1ms。

丢包检测代码实现如下：

```
static ngx_int_t
```

**4.2 探测包超时（PTO）**  

我们在2.7章节中已经了解到：PTO则是在QUIC协议中引入的一个新概念，用于解决在丢包检测和重传中的问题。当QUIC的连续两个数据包在一个RTT内没有得到确认时，就会触发一个PTO事件。在PTO发生时，QUIC并不立即判断数据包丢失，而是发送一个新的数据包来探测路径，看是否是因为网络拥塞或其他原因导致的无法传输。这个机制允许QUIC更早地探测到潜在的丢包，从而提早进行数据的重传，提高数据传输的效率和可靠性。

**4.2.1 PTO的计算**  

```
ngx_msec_t
```

  

NGX_QUIC_TIME_GRANULARITY依然代表的是计时器颗粒度，设为1ms

PTO的计算翻译成公式：

PTO = 平滑RTT + max(4*rtt方差, 1ms) 

如果是握手阶段，则PTO还应该再加上max_ack_delay。

数据发送方在每次发送数据之后，PTO计时器都会被重启。当PTO超时后，那么PTO将会被设置为当前的两倍，这就意味着如果发生连续的PTO超时，那么PTO将会指数增长，但PTO不会无限制的增长，最终会受到空闲超时时间的限制。代码如下：

```
void
```

  

**4.2.2 发送探测包**

当PTO超时的时候，数据发送方需要发送探测包，而且一般需要发送两个探测包，所谓探测包，就是需要数据接收方返回ACK的包，在nginx的实现中，这个探测包是使用PING帧实现的。具体代码如下：

```
        /* enforce 2 udp datagrams */
```

  

  

**5、QUIC的拥塞控制**

QUIC的拥塞控制算法，在QUIC的RFC文档中定义了一种与TCP的NewReno算法类似的一种拥塞控制算法，但原则上我们是可以按照TCP的Reno、Cubic、BBR等拥塞控制算法来实现针对QUIC协议的拥塞控制算法的。

Nginx的官方实现，是根据QUIC的RFC文档中所描述的拥塞控制算法来实现的，本章将介绍这一算法的实现原理。  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

上图为拥塞控制算法的状态机。  

- 当连接新建立时或经过了持续拥塞后，进入慢启动阶段；  
    
- 在慢启动阶段，出现了丢包后，则进入拥塞恢复阶段；  
    
- 在拥塞恢复阶段，如果有新的数据包被ACK（重传的数据包也算），则进入拥塞回避阶段，又称拥塞避免阶段；
    
- 在拥塞回避（拥塞避免）阶段，又出现了丢包，则继续进入拥塞恢复阶段。  
    

**5.1 慢启动**  

初始状态中，拥塞窗口初始值一般被设置为10（与TCP保持一致），最小拥塞窗口被设置为2；随着数据包的发送，拥塞窗口会呈指数增长，直到出现了丢包，则进入恢复阶段。  

```
    qc->congestion.window = ngx_min(10 * qc->tp.max_udp_payload_size,
```

拥塞窗口有一个阈值，如果慢启动阶段没有出现丢包，但拥塞窗口超过了慢启动阈值，也会进入拥塞恢复阶段。这个阈值的初始值是无穷大的，只有经历过丢包后，这个阈值才会被重新赋值。

```
    qc->congestion.ssthresh = (size_t) -1;
```

**5.2 拥塞恢复**  

当出现丢包时，进入恢复阶段，发送方将慢启动阈值设置为当前拥塞窗口的1/2.一旦在恢复期中发送的数据包有得到ACK，那么恢复期会立即进入拥塞避免阶段，这与TCP是有些不同的，TCP是只有当引发恢复期的那个丢了的包得到了ACK，才会进入拥塞避免阶段。如何检测丢包了？见第4章。  

```
static void
```

  

**5.3 拥塞避免**

只要拥塞窗口没有超过慢启动阈值，那么都是拥塞避免阶段。当然 如果从未出现过丢包，那么将会一直处于慢启动阶段。慢启动阶段的拥塞窗口也没有超过阈值（因为阈值是无穷大），但二者的区别是 是否发生过丢包。而且拥塞避免阶段使用的是加法递增乘法递减的策略。  

也即是说，如果是发生过丢包，后来进入了拥塞避免阶段，则拥塞窗口的递增是加法的，不再是指数增长了。但是一旦再次发生丢包，则会进入拥塞恢复阶段，那就是拥塞窗口被减一半，即是乘法递减。

```
    if (cg->window < cg->ssthresh) {
```

  

**6、持续拥塞**  

当在一段足够长的时间内的所有数据包都被发送方认定为丢包时，就可以认为网络正在经历持续拥塞。判定是否是持续拥塞的时长公式如下：

```
(smoothed_rtt + max(4*rttvar, kGranularity) + max_ack_delay) * kPersistentCongestionThreshold
```

smoothed_rtt是平滑rtt，max_ack_delay是最大确认延迟（在传输参数中通报给对方的），kPersistentCongestionThreshold是持续拥塞阈值，这个值不能设置太大，也不能设置太小，一般设置为3，如果设置过大，则会使发送方对于网络中的拥塞不敏感，会导致发送方向已经很拥塞的网络中激进的发送数据包，如果设置的过小，则会使发送方对于网络中的拥塞变得极为敏感，会降低发送方的吞吐量。

接下来举个例子，如何判断是持续拥塞的：  

|时间|行为|
|---|---|
|t=0|发送1号数据包（应用数据）|
|t=1|发送2号数据包（应用数据）|
|t=1.2|接收到对于1号数据包的确认|
|t=2|发送3号数据包（应用数据）|
|t=3|发送4号数据包（应用数据）|
|t=4|发送5号数据包（应用数据）|
|t=5|发送6号数据包（应用数据）|
|t=6|发送7号数据包（应用数据）|
|t=8|发送8号数据包（PTO 1）|
|t=12|发送9号数据包（PTO 2）|
|t=12.2|接收到对于9号数据包的确认|

假设max_ack_delay=2，kPersistentCongestionThreshold=3。

当在t=12.2处接收到对于9号数据包的确认时，2号数据包至8号数据包都会被认定为丢包。

从最早的丢包数据包起至最后的丢包数据包之间的持续时间被算作拥塞期：8 - 1 = 7。持续拥塞的时长为2 * 3 = 6。由于超过了阈值并且最早的丢包数据包与最后的丢包数据包间没有数据包得到了确认，所以认为网络经历过一次持续拥塞。

尽管本例中出现了PTO超时，但是它对于持续拥塞的判定不是必需的。

Nginx中实现代码如下：  

```
static ngx_msec_t
```

  

持续拥塞发生时，拥塞窗口应被设置为最小拥塞窗口，一般是2.

  

![](https://mmbiz.qlogo.cn/mmbiz_jpg/n37mOxFMpuzXzKQJtic6zzyk9UGv2icvASb7w31CxAaLm5vibbibaVicK0zYxIS8TFrsqMmNl2UJ4ojNibjw2VJsIsibA/0?wx_fmt=jpeg)

网络小菜鸟

![赞赏二维码](https://mp.weixin.qq.com/s?__biz=Mzg5NzYzNDU2Ng==&mid=2247484528&idx=1&sn=9a8e86e96974b023a5e342509b36e49e&chksm=c06f9c19f718150f1971a5ae2d1a5b35a42a3dba7cc9deb75b64719c052ea42659840726f835&mpshare=1&scene=24&srcid=0314l76e3hkqSKWZwTdHaBXR&sharer_shareinfo=a89ad9029699ad4e7948401377fbdf6b&sharer_shareinfo_first=a89ad9029699ad4e7948401377fbdf6b&key=daf9bdc5abc4e8d053508eb40528e75058091c7581d3d512b4fb08338aa00d7ca66a73f25515a9fdebaf76e8656153248b6493ad0196b24c66fb061012574b3475d86387dd09a691aa38e4e83e705b9af9deb1eddb1d6abc4fe1e84b2a9fbc08a58bb04705ef9af4599fa2e4aa53199ff37d0a287b77fc38d42f8ef62e230003&ascene=0&uin=MTEwNTU1MjgwMw%3D%3D&devicetype=Windows+11+x64&version=63090b19&lang=zh_CN&countrycode=CN&exportkey=n_ChQIAhIQQVtsDtjQpGtpz5tYmNHqThLmAQIE97dBBAEAAAAAADa6KTN8EGoAAAAOpnltbLcz9gKNyK89dVj0cVxzjNxlI5D5gC7uRs4RuDVwTUCUYNldxYPGtXhjFmC1QNJoejvpcj3V1YcdIlkkHb7zDtXJ0bRIxqGiJwGpC9radTV8sgBK7bJy0HqNhfY007en05pWPk3oRJygWb29lrla3ZN2h%2B15dcmcXzjLzPKhjal7NZAhQa1h9VUNJOzjobnpTnt%2FjmEzEFs8FMcVbKsl%2F735D16ElzAPNdA6%2BMU0Vu286a60ORm9mhdBZOw2zUAek33YqHbHansspTIt&acctmode=0&pass_ticket=XHwuIJU23VoOjPlRKK8p%2FkH9lUJvmXcfF5KzyAOemOQj0nZ98EAKfICAZPeNCSvH&wx_header=1&fasttmpl_type=0&fasttmpl_fullversion=7350504-zh_CN-zip&fasttmpl_flag=1)钟意作者

1人喜欢

![](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

个人观点，仅供参考

阅读 346

​