
Original Leon Hwang eBPF Talk

 _2024年08月26日 08:10_

![](http://mmbiz.qpic.cn/mmbiz_png/6QxwXGlhS9FPHblxYiaxMiaYWhCKBUs37o2XqGcg9ickZoh10hor44VRx8KT08XIaCYKCzUNBdxb2JWPCy6TFTzjw/300?wx_fmt=png&wxfrom=19)

**eBPF Talk**

专注于 eBPF 技术，以及 Linux 网络上的 eBPF 技术应用

140篇原创内容

公众号

最近在使用 ringbuf 的 `bpf_ringbuf_reserve()` 时踩了一个坑，记录一下。

## ringbuf 简介

ringbuf 是 BPF 中能够取代 **PERF_EVENT_ARRAY** 的特殊 map 类型，提供了类似的 helpers:

- `bpf_ringbuf_output()`: 将数据写入 ringbuf。
    
- `bpf_ringbuf_reserve()`: 为数据预留空间。
    
- `bpf_ringbuf_submit()`: 提交预留的数据。
    
- `bpf_ringbuf_discard()`: 丢弃预留的数据。
    
- bpf: Implement BPF ring buffer and verifier support for it[1] since 5.8 kernel.
    

根据该 commit，推荐的用法是 `bpf_ringbuf_reserve()` 加 `bpf_ringbuf_submit()`/`bpf_ringbuf_discard()`，而不是 `bpf_ringbuf_output()`；因为 `bpf_ringbuf_output()` 需要拷贝数据。

## `bpf_ringbuf_reserve()` 有锁吗？

答案：**有**。

直接翻看 `bpf_ringbuf_reserve()` 的源码：

`// https://github.com/torvalds/linux/blob/5be63fc19fcaa4c236b307420483578a56986a37/kernel/bpf/ringbuf.c#L408         static void *__bpf_ringbuf_reserve(struct bpf_ringbuf *rb, u64 size)   {       // ...          cons_pos = smp_load_acquire(&rb->consumer_pos);          if (in_nmi()) {           if (!spin_trylock_irqsave(&rb->spinlock, flags))               return NULL;       } else {           spin_lock_irqsave(&rb->spinlock, flags);       }          pend_pos = rb->pending_pos;       prod_pos = rb->producer_pos;       new_prod_pos = prod_pos + len;          // ...          /* pairs with consumer's smp_load_acquire() */       smp_store_release(&rb->producer_pos, new_prod_pos);          spin_unlock_irqrestore(&rb->spinlock, flags);          return (void *)hdr + BPF_RINGBUF_HDR_SZ;   }      BPF_CALL_3(bpf_ringbuf_reserve, struct bpf_map *, map, u64, size, u64, flags)   {       struct bpf_ringbuf_map *rb_map;          if (unlikely(flags))           return 0;          rb_map = container_of(map, struct bpf_ringbuf_map, map);       return (unsigned long)__bpf_ringbuf_reserve(rb_map->rb, size);   }   `

在写 bpf 代码时，没留意该锁对性能的影响，导致性能变得很差。示例：

`static __always_inline void   record_event(struct xdp_md *xdp)   {       struct event_t *event;          event = bpf_ringbuf_reserve(&ringbuf, sizeof(*event));       if (!event)           return;          event->pkt_len = xdp->data_end - xdp->data;       if (event->pkt_len <= MTU)           return;          __fill_event(event, xdp);       bpf_ringbuf_submit(event, &ringbuf);   }   `

对于这种情况，需要将 `bpf_ringbuf_reserve()` 调整到 `if` 语句之后，避免不必要的锁操作：

`static __always_inline void   record_event(struct xdp_md *xdp)   {       struct event_t *event;       int pkt_len;          pkt_len = xdp->data_end - xdp->data;       if (pkt_len <= MTU)           return;          event = bpf_ringbuf_reserve(&ringbuf, sizeof(*event));       if (!event)           return;          event->pkt_len = pkt_len;       __fill_event(event, xdp);       bpf_ringbuf_submit(event, &ringbuf);   }   `

有没有更高效率的使用办法呢？

## `bpf_ringbuf_query()` **BPF_RB_AVAIL_DATA**

查看 `bpf_ringbuf_query()` 的 helper 文档：

`/*    * bpf_ringbuf_query    *    *  Query various characteristics of provided ring buffer. What    *  exactly is queries is determined by *flags*:    *    *  * **BPF_RB_AVAIL_DATA**: Amount of data not yet consumed.    *  * **BPF_RB_RING_SIZE**: The size of ring buffer.    *  * **BPF_RB_CONS_POS**: Consumer position (can wrap around).    *  * **BPF_RB_PROD_POS**: Producer(s) position (can wrap around).    *    *  Data returned is just a momentary snapshot of actual values    *  and could be inaccurate, so this facility should be used to    *  power heuristics and for reporting, not to make 100% correct    *  calculation.    *    * Returns    *  Requested value, or 0, if *flags* are not recognized.    */   static __u64 (*bpf_ringbuf_query)(void *ringbuf, __u64 flags) = (void *) 134;   `

可以通过 `bpf_ringbuf_query()` 获取 ringbuf 未消费的数据量，从而推算出可用来塞数据的空间大小。

`static __always_inline void   record_event(struct xdp_md *xdp)   {       struct event_t *event;       __u64 avail_data;       int pkt_len;          pkt_len = xdp->data_end - xdp->data;       if (pkt_len <= MTU)           return;          avail_data = bpf_ringbuf_query(&ringbuf, BPF_RB_AVAIL_DATA);       if (RINGBUF_SIZE - avail_data < sizeof(*event))           return;          event = bpf_ringbuf_reserve(&ringbuf, sizeof(*event));       if (!event)           return;          event->pkt_len = pkt_len;       __fill_event(event, xdp);       bpf_ringbuf_submit(event, &ringbuf);   }   `

不过，查询 **BPF_RB_AVAIL_DATA** 得付出一点代价：

`// https://github.com/torvalds/linux/blob/5be63fc19fcaa4c236b307420483578a56986a37/kernel/bpf/ringbuf.c#L299      static unsigned long ringbuf_avail_data_sz(struct bpf_ringbuf *rb)   {       unsigned long cons_pos, prod_pos;          cons_pos = smp_load_acquire(&rb->consumer_pos);       prod_pos = smp_load_acquire(&rb->producer_pos);       return prod_pos - cons_pos;   }      BPF_CALL_2(bpf_ringbuf_query, struct bpf_map *, map, u64, flags)   {       struct bpf_ringbuf *rb;          rb = container_of(map, struct bpf_ringbuf_map, map)->rb;          switch (flags) {       case BPF_RB_AVAIL_DATA:           return ringbuf_avail_data_sz(rb);       case BPF_RB_RING_SIZE:           return ringbuf_total_data_sz(rb);       case BPF_RB_CONS_POS:           return smp_load_acquire(&rb->consumer_pos);       case BPF_RB_PROD_POS:           return smp_load_acquire(&rb->producer_pos);       default:           return 0;       }   }   `

其中的 `smp_load_acquire()` 涉及到内存屏障，会有一定的开销。

参考：LINUX KERNEL MEMORY BARRIERS[2].

## ringbuf 的大小要求

在使用 ringbuf 时，`max_entries` 必须是 2 的幂次方、而且还要求是 **PAGE_SIZE** 的倍数。

`// https://github.com/torvalds/linux/blob/5be63fc19fcaa4c236b307420483578a56986a37/kernel/bpf/ringbuf.c#L189      static struct bpf_map *ringbuf_map_alloc(union bpf_attr *attr)   {       struct bpf_ringbuf_map *rb_map;          if (attr->map_flags & ~RINGBUF_CREATE_FLAG_MASK)           return ERR_PTR(-EINVAL);          if (attr->key_size || attr->value_size ||           !is_power_of_2(attr->max_entries) ||           !PAGE_ALIGNED(attr->max_entries))           return ERR_PTR(-EINVAL);          rb_map = bpf_map_area_alloc(sizeof(*rb_map), NUMA_NO_NODE);       if (!rb_map)           return ERR_PTR(-ENOMEM);          bpf_map_init_from_attr(&rb_map->map, attr);          rb_map->rb = bpf_ringbuf_alloc(attr->max_entries, rb_map->map.numa_node);       if (!rb_map->rb) {           bpf_map_area_free(rb_map);           return ERR_PTR(-ENOMEM);       }          return &rb_map->map;   }   `

## 总结

记住以下 3 点经验：

0. 使用 `bpf_ringbuf_reserve()` 时，要注意锁的开销。
    
1. 可以通过 `bpf_ringbuf_query()` 查询 ringbuf 的未消费数据量，从而推算出可以用来塞数据的空间大小。
    
2. ringbuf 的 `max_entries` 必须是 2 的幂次方、而且还要求是 **PAGE_SIZE** 的倍数。
    

参考资料

[1]

bpf: Implement BPF ring buffer and verifier support for it: _https://github.com/torvalds/linux/commit/457f44363a8894135c85b7a9afd2bd8196db24ab_

[2]

LINUX KERNEL MEMORY BARRIERS: _https://www.kernel.org/doc/Documentation/memory-barriers.txt_

![](https://mmbiz.qlogo.cn/mmbiz_jpg/dXcFqVriadOlGTfj0axJoeJwD6zYrpbz8ItWnYW2SCiapEIrjialS5zkG2G63YWls9QoQtRFWdmpwHIWnpl8TtB6Q/0?wx_fmt=jpeg)

Leon Hwang

 创作不易，赞赏杯咖啡☕️ 

![赞赏二维码](https://mp.weixin.qq.com/s?__biz=MjM5MTQxNTk5MA==&mid=2247485697&idx=1&sn=0a8f2bd5f299dfdf156e7e39df0bcceb&chksm=a7201ec40628e73d06c29c01689ed834233e3414d86d5b6e76825b94fc7f97f8c931b1aaf60c&mpshare=1&scene=24&srcid=0826ZzOJBAhoi6rKdXK54EIl&sharer_shareinfo=982a220c9df004f85055701d7f59ed90&sharer_shareinfo_first=982a220c9df004f85055701d7f59ed90&key=daf9bdc5abc4e8d0c082738b971249491db8c8309872cc330d6caf6d51c96ea7132529846c5cd53b21fca3444fd23ee341876a5a06487a142c2e81afea4e15d76bc28be2cfbd0e0699f46a24a06b55c6f9d7d9739b910257c587fa80309b62613f3e0ac2b3a20e6d2a4e52f93bc51a1ed3fd68aed07021920b90d3f3aeae28d4&ascene=14&uin=MTEwNTU1MjgwMw%3D%3D&devicetype=iMac+MacBookAir10%2C1+OSX+OSX+14.6.1+build(23G93)&version=13080710&nettype=WIFI&lang=en&session_us=gh_1ca20b19d611&countrycode=CN&fontScale=100&exportkey=n_ChQIAhIQUnWY5naN1BNT%2B9%2FWsJ8r9RKUAgIE97dBBAEAAAAAACyWIlKsvMIAAAAOpnltbLcz9gKNyK89dVj0dfHVJoiMIU%2BJXwHLlXPdwAHqnjLxtIQcLR3MLAiCSOiwz15kfwe7%2BvqbxX4vyNpdN0DlM2DW5e5Q6j7DEYUIaWajVjLdwW4NI33t1FrZO9TtwTtrnJZPFVdK8p39wPMdUXjLRfEIw12r8Fm6RomMyYIuk%2FlK1LTqUPg%2FKlSZMY8zklq8X4GG9hW0R41jOdzFSgls8P3WPCZBp9qFolF72MidkTI%2FixFPvtDRo9EPLxzroLFFxw%2Bb2tGZakMnd%2FHoVrBdMvpk%2BuOmik5rU5iqzupNzUzcCdsHS2QWJywUx7l4scMUNhmtwdWDIibyjA%3D%3D&acctmode=0&pass_ticket=0sNJA9EKpMpl%2BO2AlAnNF2wqvDEeFje7LPXaZAHxLISCef8bSr8e2roc8v2Q7xGU&wx_header=0)Like the Author

eBPF Talk129

ringbuf1

eBPF Talk · 目录

上一篇eBPF Talk: XDP 解析所有 TCP options

Read more

Reads 370

​

Comment

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/6QxwXGlhS9FPHblxYiaxMiaYWhCKBUs37o2XqGcg9ickZoh10hor44VRx8KT08XIaCYKCzUNBdxb2JWPCy6TFTzjw/300?wx_fmt=png&wxfrom=18)

eBPF Talk

8201

Comment

Comment

**Comment**

暂无留言