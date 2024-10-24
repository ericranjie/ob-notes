
原创 酷玩BPF 酷玩BPF _2024年04月28日 08:30_ _浙江_

云场景下，虚拟化成了主流技术，virtio作为guest和host之间数据传输的重要通道，许多IO卡住（IO Hang）、网络时延高的问题有时就出在guest到host的前后端报文发送与获取上。对于网络，延迟高的点非常多，比如软中断、硬中断、调度延迟、用户态收包延迟等，而前后端交界处的延迟点，平常关注的并不多。

在之前的一篇文章，我们介绍了[virtio-net 队列可观测](http://mp.weixin.qq.com/s?__biz=MzkyMjM4MTcwOQ==&mid=2247484791&idx=1&sn=d3c4b4042e84ff3addcbc78e5edd593c&chksm=c1f47995f683f0830070c231cbbd84c90698736c5aac55ca32321191dd9db1a9e5fbc91cb4a6&scene=21#wechat_redirect)，目的是帮助大家如何去分析报文是卡在前端还是后端，姑且叫网络卡住（Net Hang）吧。本文主要关注vitio-net的时延问题，希望只在guest上就能快速定位时延问题出在前端还是后端。

# 前后端数据交互

virtio-net 前后端交互主要是通过vring进行管理的，报文收发包有很多文章会介绍到，我们以下图简要回顾一下收包的过程，从virtio-net 后端到前端是如何配合的。

![[Pasted image 20241024225407.png]]

1.guest会先预备有效的数据区域给host，当host 网卡中收到包时往vring中写数据；

2.host数据写完后通过中断告知guset；

3.guest收到中断后，往vring中读数据，读数据的过程中会关闭中断，host此时不会发中断，但是host可以继续写数据，直到vring中没有数据，guest结束本次收包流程；

下图所示为硬中断和软中断及调度执行的过程：

![[Pasted image 20241024225417.png]]

上图1、2两个阶段，存在时延的影响点有：

1.host收到了数据包，但是发送中断通知较慢，造成了时延，属于阶段1的情况；

2.guest收到了中断通知，产生软中断到真正执行收包完成较慢，属于阶段2的情况。

# 时延分析思路

我们来看内核代码处理流程(以kernel 4.9版本为例)：

vring_interrupt是virtio中断处理handler，通过回调skb_recv_done，在硬中断结束后，就会进入到软中断处理。

![[Pasted image 20241024225428.png]]

guest在收包的开始阶段会关闭host的中断，在完成收包后再次开启，分别在virtqueue_disable_cb virtqueue_enable_cb_prepare设置，因此我们可以考虑probe这两个点来标记guest收包完成。

由于这两个是公共函数，可以换成skb_recv_done 和 napi_complete_done。

![[Pasted image 20241024225441.png]]

实际probe的效果：

![[Pasted image 20241024225458.png]]

这里有个问题就是如果host一直写，guest一直读，数据量超过64的话，那么一次poll received 的值就会等于budget(该值为64)，guset会继续poll，不会认为收包结束。

所以如果同一时间收到的包越多，这两个probe的时间也越长，因此可以加上收包数的影响。这个收包数量在receive_buf中有统计。

![[Pasted image 20241024225513.png]]

内核对一直poll的情况也做了2个jiffies的限制。

![[Pasted image 20241024225523.png]]

通过上面的方法阶段2可以大概评估，下面介绍下阶段1的思路。

在guest完成收包后会再次判断last_used_idx和used_idx是否一致，guest每收一个包last_used_idx++，host每写一个包used_idx++，如果不一致那么说明host又往vring中写数据，此时又会关中断，发起napi_schedule开始收包。

![[Pasted image 20241024225537.png]]

![[Pasted image 20241024225549.png]]

因此我们可以将last_used_idx 和used_idx 是否一致作为host写数据的标志，这里有个前提是当前不在收包的过程中。

采取的方法：

创建一个timer定时轮询两个值是否一致，将两者不一致且不在收包的过程中（可以看关中断的标志位或者napi的状态）作为起始时间，将两者一致作为结束时间。

中间还可以结合是否有开/关中断及收包个数判断在timer采样周期内是否结束并开始了新的一次收发包，确保采样不轮空。

假设这样的场景：host有写数据但是中断一致迟迟未通知，那么跟踪下来可以看到有时延，且中间没有收包数。

以上只能确定一轮收包的情况，对于单个包的时延情况较难判断，原因在于收包的结束标志是以vring中有无数据。而且每次从vring中读一个数据包就往上层发送。

# 输出结果

![[Pasted image 20240918133848.png]]

分析上面介绍的阶段1 的情况：

timer定时5ms轮询一次，以cpu2为例，last_used_idx和used_idx不相同说明host有数据包过来，last_used_idx此时没有改变，且avail_flags_shadow置为0，说明host的中断没有立马上来，否则的话在中断函数中会将avail_flags_shadow置为1。

# 核心代码

时延检测的关键在于通过定时器去分析几个idx的变化情况，以及时延的变化情况，通过脚本提取出存在时延高的情况，并进一步确定前后端的问题。

```c
static enum hrtimer_restart trace_hrtimer_handler(struct hrtimer *hrtimer){ u64 now; u64 delta; u64 last_time; now = local_clock(); if (more_used(virtnet_trace_vq) &&   is_virtqueue_disable_cb(virtnet_trace_vq)) {  last_time = __this_cpu_read(virtnet_trace_cpu_trace->last_time);  delta = now - last_time;  if (is_used_idx_move(virtnet_trace_vq) &&    (delta > THRES)) {   record_trace_data(delta);#ifdef VIRTNET_DEBUG   printk("cpu:%d, vring last_used_idx:%u, used_idx:%u,avail_flags_shadow:%u, delta:%lluus\n",     smp_processor_id(), virtnet_trace_vq->last_used_idx,     virtnet_trace_vq->vring.used->idx,     virtnet_trace_vq->avail_flags_shadow,     delta / 1000U);#endif  } } else {  virtnet_trace_used_idx = virtnet_trace_vq->vring.used->idx;  __this_cpu_write(virtnet_trace_cpu_trace->last_time, now); } hrtimer_forward_now(hrtimer, ns_to_ktime(virtnet_trace_period * 1000U)); return HRTIMER_RESTART;}
```

# 总结

本文通过内核模块的方式，来分析在virtio网卡中，前端和后端之间是否存在较大的延时，对我们定位报文整体延时有很大的帮助。这里没有及时用eBPF来实现是因为性能等问题，如果大家有兴趣，可以修改为eBPF的实现方式，贡献Coolbpf（https://gitee.com/anolis/coolbpf）项目中来。

阅读 984

​

喜欢此内容的人还喜欢

libevent，一个束之高阁的c++库

莫七七编程

不喜欢

不看的原因

确定

- 内容低质
- 不看此公众号内容

GDB调试指南入门篇：揭开程序运行世界的神秘面纱

我关注的号

linux源码阅读

不喜欢

不看的原因

确定

- 内容低质
- 不看此公众号内容

libvpx，一个扶摇直上的c++库

梦梦爱编程

不喜欢

不看的原因

确定

- 内容低质
- 不看此公众号内容

![](https://mmbiz.qpic.cn/sz_mmbiz_jpg/iaqktmiciciaJhOyIv0AVDBV5uMgicsGb6gdslWPbwBx7oiafBRaSiczgHWGcOGbkmhY7NQ1WItxpLU0Q940ktV00uLDw/0?wx_fmt=jpeg)

写留言

[](javacript:;)

![](http://mmbiz.qpic.cn/sz_mmbiz_png/RWnbgykjTNLYUsm55aeV0aP5Qet4yIGmBEsbwoZdgdUdYTGJJrYknQLvVdzjs9wIdkHrNoKfEGcvCyy7HFjTvA/300?wx_fmt=png&wxfrom=18)

酷玩BPF

91139

写留言

写留言

**留言**

暂无留言
