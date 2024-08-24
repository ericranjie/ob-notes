# 

高可用架构

 _2022年02月23日 09:50_

以下文章来源于Redis Gossip ，作者ShooterIT

[

![](http://wx.qlogo.cn/mmhead/Q3auHgzwzM621ia73TJymq4wRabjBqnVJicIdQWeHeEl6HZuwanPNuiaA/0)

**Redis Gossip**.

Send and receive redis messages

](https://mp.weixin.qq.com/s?__biz=MzAwMDU1MTE1OQ==&mid=2653558912&idx=1&sn=2f7fb59bc8ea933d1d27f3e3c2300c5f&chksm=81398918b64e000e77b419b9cfae00bd4a24f3e2e7fd79ad4e579131103c11af08e544cb8fc7&mpshare=1&scene=24&srcid=0223ToEq17v9P2nk9tLWvMov&sharer_sharetime=1645582987613&sharer_shareid=5fb9813bfe9ffc983435bfc8d8c5e9ca&key=daf9bdc5abc4e8d0813a763f47f88fbdc4502c8e27adb33cbe63562e8a1c6a14a80a32320343a0d422f52b829d1b0117a46a2a90e2b4ada5bd035cee94a4984f5b73add64f052e55e85c10d4bdae0b6d40b626eb1c86ab37ea5ddb35bc6fcded30959a1bec4473a80e796d44c5af0785ebcb77b1c261d222aa717ca267919782&ascene=0&uin=MTEwNTU1MjgwMw%3D%3D&devicetype=Windows+11+x64&version=63090b19&lang=zh_CN&countrycode=CN&exportkey=n_ChQIAhIQm96dMFudIUPVPvPb5RNH0BLmAQIE97dBBAEAAAAAAAEBJIZOf9cAAAAOpnltbLcz9gKNyK89dVj0wUoxNKAQ1g46nLn3ZlXOFp41rnu9yiQBNwLjS0cj%2BprxbLp5cb7EjBuySHV3t3KhvFwsEelU4gYb6ScCHR2t%2FKEDJN0kFtXxgV5ZyBZwU062HK7zJgZhDg4uNl0t1z0rz%2BBckl8vS8CDEMNlko0PfiiakizpxKP20aB3qh2CXQO5OFLF5IXGFwN26fscpUrhqJX69ST8adzey5Gh9R8NqW%2BvuqUSm%2FnOTkGSSeyx1iSToWr6gWVzl%2Fe21ZCVYZCD&acctmode=0&pass_ticket=tA5pa1w93JJ0%2BBW2mjsqjdMjPf1WSUYOmHGDxNhs2HG6UKbKHK3uWlQSK4dehe0s&wx_header=1&fasttmpl_type=0&fasttmpl_fullversion=7351805-zh_CN-zip&fasttmpl_flag=1#)

本文将主要分析 Redis 主从复制中的内存消耗过多和堵塞问题，以及 Redis 7.0 (尚未发布) 的共享复制缓冲区方案是如何解决这些问题的。

## 1 Redis 主从复制原理简介

尽管本文的目的不是讲解 Redis 主从复制的原理，但在开始进入主题之前，我们先简单回顾一下 Redis 主从复制的基本原理。Redis 的主从复制主要分为两种情况：

- **全量同步**
    

当主库收到从库的同步请求时，如果从库的复制历史与主库不一致，或者未能在复制积压区中找到从库请求的同步点时，则会与从库进行全量同步。主库通过 fork 子进程产生内存快照，然后将数据序列化为 RDB 格式同步到从库，使从库的数据与主库某一时刻的数据一致。

- **命令传播**
    

当从库与主库完成全量同步后，进入命令传播阶段，主库将变更数据的命令发送到从库，从库将执行相应命令，使从库与主库数据持续保持一致。

## 2 Redis 复制缓存区相关问题分析

### 2.1 多从库时主库内存占用过多

![图片](https://mmbiz.qpic.cn/mmbiz_png/cJMXkxTQicjueTI1ZSGdZ9WAOibVhBQqic00BURlialQowpkQlOXLICXls0a9GrIcuDo63V6QPnSh0iaibfPfhZAmOicA/640?wx_fmt=png&wxfrom=13&tp=wxpic)

图1 Redis 主从复制

如上图所示，对于 Redis 主库，当用户的写请求到达时，主库会将变更命令分别写入所有从库的缓存区（OutputBuffer)，以及复制积压区 (ReplicationBacklog)。需要指出的是，全量同步时依然会执行该逻辑，所以在全量同步阶段经常会触发 client-output-buffer-limit，主库断开与从库的连接，导致主从同步失败，甚至出现循环持续失败的情况。

该实现一个明显的问题是内存占用过多，所有从库的连接在主库上是独立的，也就是说每个从库 OutputBuffer 占用的内存空间也是独立的，那么主从复制消耗的内存就是所有从库缓冲区内存大小之和。如果我们设定从库的 client-output-buffer-limit 为 1GB，如果有三个从库，则在主库上可能会消耗 3GB 的内存用于主从复制。另外，真实环境中从库的数量不是确定的，这也导致 Redis 实例的内存消耗不可控。

### 2.2 OutputBuffer 拷贝和释放的堵塞问题

Redis 为了提升多从库全量复制的效率和减少 fork 产生 RDB 的次数，会尽可能的让多个从库共用一个 RDB，如下代码所示，当已经有一个从库触发 RDB BGSAVE 时，后续需要全量同步的从库会共享这次 BGSAVE 的 RDB，为了从库复制数据的完整性，会将之前从库的 OutputBuffer 拷贝到请求全量同步从库的 OutputBuffer 中。

```
/* To attach this slave, we check that it has at least all the * capabilities of the slave that triggered the current BGSAVE. */if (ln && ((c->slave_capa & slave->slave_capa) == slave->slave_capa)) {    /* Perfect, the server is already registering differences for     * another slave. Set the right state, and copy the buffer.     * We don't copy buffer if clients don't want. */    if (!(c->flags & CLIENT_REPL_RDBONLY)) copyClientOutputBuffer(c,slave);    replicationSetupSlaveForFullResync(c,slave->psync_initial_offset);    serverLog(LL_NOTICE,"Waiting for end of BGSAVE for SYNC");} else if ( ... )
```

> 注：为避免无效的复制数据积累和拷贝，我实现了 rdb-only 复制[1] 以更好地支持远程全量备份。

`copyClientOutputBuffer` 看似只是一个简单的 buffer 拷贝，但可能存在堵塞问题，因为 OutputBuffer 链表上的数据可达数百 MB 甚至数 GB 之多，对其拷贝可能使用百毫秒甚至秒级的时间，而且该堵塞问题没法通过日志或者 latency 观察到，但影响却很大。

同样地，当 OutputBuffer 大小触发 limit 限制时，Redis 就是关闭该从库链接，而在释放 OutputBuffer 时，也需要释放数百 MB 甚至数 GB 的数据，其耗时对 Redis 而言也很长。

### 2.3 ReplicationBacklog 的限制

我们知道复制积压缓冲区 ReplicationBacklog 是 Redis 实现部分重同步的基础，如果从库可以进行增量同步，则主库会从 ReplicationBacklog 中拷贝从库缺失的数据到其 OutputBuffer。拷贝的数据最多是 ReplicationBacklog 的大小，为了避免拷贝数据过多的问题，我们通常不会让该值过大，一般百兆左右。但在大容量实例中，为了避免由于主从网络中断导致的全量同步，我们又希望该值大一些，这就存在矛盾了。

此外，我们在重新设置 ReplicationBacklog 大小时，会导致 ReplicationBacklog 中的内容全部清空，所以如果在变更该配置期间发生主从断链重连，则很有可能导致全量同步。

> 注: Redis 存在重复拷贝 ReplicationBacklog 的问题， 避免 ReplicationBacklog 拷贝的 PR[2] 已修复。

## 3 共享复制缓存区的设计与实现

### 3.1 方案简述

每个从库在主库上单独拥有自己的 OutputBuffer，但其存储的内容却是一样的，一个最直观的想法就是主库在命令传播时，将这些命令放在一个全局的复制数据缓冲区中，多个从库共享这份数据，不同的从库对引用复制数据缓冲区中不同的内容，这就是『共享复制缓存区』方案的核心思想。实际上，复制积压缓冲区（ReplicationBacklog）中的内容与从库 OutputBuffer 中的数据也是一样的，所以该方案中，ReplicationBacklog 和从库一样共享一份复制缓冲区的数据，也避免了 ReplicationBacklog 的内存开销。

```
/* Similar with 'clientReplyBlock', it is used for shared buffers between * all replica clients and ReplicationBacklog. */typedef struct replBufBlock {    int refcount;           /* Number of replicas or repl backlog using. */    long long id;           /* The unique incremental number. */    long long repl_offset;  /* Start replication offset of the block. */    size_t size, used;    char buf[];} replBufBlock;
```

借鉴了 Redis 客户端输出缓冲区的实现方案，『共享复制缓存区』方案中复制缓冲区 (ReplicationBuffer) 的表示也采用链表的表示方法，将 ReplicationBuffer 数据切割为多个 16KB 的数据块 (replBufBlock)，然后使用链表来维护起来。为了维护不同从库的对 ReplicationBuffer 的使用信息，在 replBufBlock 中增加了如下字段：

- `refcount`：block 的引用计数
    
- `id`：block 的唯一标识，单调递增的数值
    
- `repl_offset`：block 开始的复制偏移
    

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)  

图2 ReplicationBacklog 和 从库对 ReplicationBuffer 的引用描述

  

如上图所示，ReplicationBuffer 是由多个 replBufBlock 组成的链表，当 ReplicatonBacklog 或从库对某个 block 使用时，便对**正在使用**的 replBufBlock 增加引用计数，可以看到，ReplicatonBacklog 正在使用的 replBufBlock refcount 是 1，从库 A 和 B 正在使用的 replBufBlock refcount 是 2。当从库使用完当前的 replBufBlock（已经将数据发送给从库）时，就会对其 refcount 减 1 而且移动到下一个 replBufBlock，并对其 refcount 加 1。

**2.1 提到的多从库消耗内存过多的问题通过共享复制缓存区方案得到了解决，对于 2.2 和 2.3 提出的问题是否解决了呢？**

```
typedef struct client {    ...    listNode *ref_repl_buf_node; /* Referenced node of replication buffer blocks,                                  * see the definition of replBufBlock. */    size_t ref_block_pos;        /* Access position of referenced buffer block,                                  * i.e. the next offset to send. */    ...} client;
```

首先来看 2.2 中的问题， **OutputBuffer 拷贝**问题，当前从库的 OutputBuffer 的描述只有对共享 ReplicationBuffer 的引用信息，如上代码所示，所以之前的数据深拷贝变成了更新引用信息，即对正在使用的 replBufBlock refcount 加 1，这仅仅是一条简单的赋值操作，非常轻量。**OutputBuffer 释放**问题呢？在当前的方案中释放从库 OutputBuffer 就变成了对其正在使用的 replBufBlock refcount 减 1，也是一条赋值操作，不会有任何阻塞。

```
/* Replication backlog is not separate memory, it just is one consumer of * the global replication buffer. This structure records the reference of * replication buffers. Since the replication buffer block list may be very long, * it would cost much time to search replication offset on partial resync, so * we use one rax tree to index some blocks every REPL_BACKLOG_INDEX_PER_BLOCKS * to make searching offset from replication buffer blocks list faster. */typedef struct replBacklog {    listNode *ref_repl_buf_node; /* Referenced node of replication buffer blocks,                                  * see the definition of replBufBlock. */    size_t unindexed_count;      /* The count from last creating index block. */    rax *blocks_index;           /* The index of reocrded blocks of replication                                  * buffer for quickly searching replication                                  * offset on partial resynchronization. */    long long histlen;           /* Backlog actual data length */    long long offset;            /* Replication "master offset" of first                                  * byte in the ReplicationBacklog buffer.*/} replBacklog;
```

对于 2.3 提出的问题也很容易解决了，因为 ReplicatonBacklog 也只是记录了对 ReplicationBuffer 的引用信息，如上代码所示，对 ReplicatonBacklog 的拷贝也仅仅成了找到正确的 replBufBlock，然后对其 refcount 加 1。无需担心 ReplicatonBacklog 过大导致的拷贝堵塞问题。而且对 ReplicatonBacklog 大小的变更也仅仅是配置的变更，不会清掉数据。

### 3.2 关键问题

- **ReplicationBuffer 的裁剪和释放**
    

ReplicationBuffer 不可能无限增长，Redis 会有相应的逻辑对其进行裁剪，简单来说，Redis 会从头访问 replBufBlock 链表，如果发现 replBufBlock refcount 为 0，则会释放它，直到迭代到第一个 replBufBlock refcount 不为 0 才停止，对应实现函数实现为 `incrementalTrimReplicationBacklog`。所以想要释放 ReplicationBuffer，只需要减少相应 ReplBufBlock 的 refcount，有以下主要会减少 refcount 的情况：

- 当从库使用完当前的 replBufBlock 会对其 refcount 减 1
    
- 当从库断开链接时会对正在引用的 replBufBlock refcount 减 1，无论是因为超过 client-output-buffer-limit 导致的断开还是网络原因导致的断开
    
- 当 ReplicationBacklog 引用的 replBufBlock 数据量超过设置的该值大小时，会对正在引用的 replBufBlock refcount 减 1，以尝试释放内存
    

> > 注：一般而言，设置的从库 client-output-buffer-limit 会远大于 ReplicationBacklog 大小，所以如果有一个慢从库节点，则其引用的 ReplicationBuffer 将大于 ReplicationBacklog，但是为了尽可能的进行部分重同步（PSYNC)，这种情况下 ReplicationBacklog 不会尝试释放 replBufBlock，从而隐式地将 ReplicationBacklog 变大，以便尽可能支持部分重同步。

ReplicationBuffer 的释放也需要注意，因为如果一个从库引用的 replBufBlock 过多，它断开时释放的 replBufBlock 可能很多，这会造成堵塞问题，所以我们会限制一次释放的个数，未及时释放的内存在 ServerCron 中渐进式释放。

- **ReplicatonBacklog 和从库只对正在使用的 replBufBlock 增减 refcount 的原因**
    

在对 replBufBlock 增减 refcount 中，一个直观的思路是，对所有要使用的 replBufBlock 都增减引用计数，同样释放 replBufBlock 时，只需要检查 refcount 是否为 0。但这种思路的最大问题是从库在断开链接时需要对其所有增加过引用计数的 replBufBlock 都减 1，这又会出现了 2.2 提出的**释放 OutputBuffer** 造成堵塞的问题。从库只对正在使用的 replBufBlock 增减 refcount，直觉上这套机制看似有些脆弱，那么如何保证不会错误释放 replBufBlock 呢，尤其是那些不是正在使用但之后会使用，即 refcount == 0 的 replBufBlock？

首先 replBufBlock 通过链表维护，天然具有有序性，也保证了 ReplicationBuffer 的正确性，其次，与我们上述提到的释放策略有关，我们只释放链表**第一个** refcount 为 0 的 replBufBlock，若不为 0 则**终止**对 replBufBlock 链表的遍历从而避免对中间 refcount 为 0 的 replBufBlock 的释放。

- **ReplicatonBacklog 使用 ReplicationBuffer 的问题和解决方案**
    

当从库尝试与主库进行增量重同步时，会发送自己的 repl_offset, 若从库可满足增量重同步条件，则主库会从 ReplicationBacklog 中拷贝从库缺失的数据到从库 OutputBuffer，虽然现在的拷贝变成了仅仅是对特定 replBufBlock 引用计数的改变，在每个 replBufBlock 中记录了该其第一个字节对应的 repl_offset，但如何高效地从数万个 replBufBlock 的链表中找到特定的那个, 仍然一件棘手的事情。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

图3 查找包含特定 repl_offset 的 replBufBlock

直接从头到位遍历链表查找对应的 replBufBlock 必然会耗费较多时间而堵塞服务。起初，我额外使用了一个链表用于索引固定区间间隔的 replBufBlock，每 1000 个 replBufBlock 记录一个索引信息，当查找 repl_offset 时，会先从索引链表中查起，然后再查找 replBufBlock 链表，类似于跳表的查找实现，如上图所示。但这个方案在极端场景下可能会查找超过千次，有 10 毫秒以上的延迟，与 Redis 的维护者 Oran Agra 讨论后，我们对此仍不满足，遂放弃。

最终使用 rax 树实现了对 replBufBlock 固定区间间隔的索引，每 64 个记录一个索引点。一方面，rax 索引占用的内存较少；另一方面，查询效率也是非常高，理论上查找比较次数不会超过 100，耗时在 1 毫秒以内。

### 3.3 其他相关问题

- **mem_total_replication_buffers**
    

在 Redis 的 INFO 命令输出中增加了 `mem_total_replication_buffers` 字段，用以展示全局复制缓存区 ReplicationBuffer 的大小。因共享复制缓冲区，只有所有从库引用的复制数据超过 ReplicationBacklog 时，我们才认为超过的内存是从库消耗的内存，即 `mem_clients_slaves` 字段的数值，否则 `mem_clients_slaves` 为 0。

- **Key 驱逐**
    

在 Redis 进行 key 驱逐时, 从库的 OutputBuffer 内存是不算入 Redis 的内存消耗，因共享复制缓冲区机制，我们认为只有超出 ReplicationBacklog 大小的部分是从库额外的内存消耗，也就是说 ReplicationBacklog 消耗的内存仍然会导致 key 的驱逐。

- **IO 多线程**
    

由于所有从库和 ReplicationBacklog 使用全局复制缓冲区，如果启用 IO 多线程，为了保证数据访问线程安全，我们必须让主线程处理发送数据到所有从库的工作。但在此之前，其他 IO 线程可以处理从库的发送 OutputBuffer 工作，无差别对待普通客户端和从库客户端。

## 4 总结

本文主要分析了 Redis 主从复制缓存区存在的几个问题：内存占用过多，OutputBuffer 拷贝和释放造成的不易察觉的堵塞问题，以及 ReplicaitonBacklog 的不足，然后提出了共享复制缓存区的方案，并分析了该方案的几个关键问题。希望该方案能够减少大家在运维 Redis 过程中遇到的主从复制问题，降低内存消耗，避免线上稳定性问题，Cheers!

事实上，本文主要是对 PR: Replication backlog and replicas use one global shared replication buffer[3] 动机和实现的中文描述。也感谢 sundb 和 zhaozhao.zz 对这个 PR 的 BugFix[4] 和 索引优化[5]。如有问题，欢迎大家指正！

当前，笔者在百度负责 Redis 的研发工作，是 Redis 社区的 Member，持续跟进 Redis 的优化工作，也会跟大家分享 Redis 中一些有趣的设计。同时本人也是[磁盘 KV 存储服务 Kvrocks](http://mp.weixin.qq.com/s?__biz=MzUxNTg5NzM1Nw==&mid=2247483681&idx=1&sn=e05dbf64f71a44977b38ba64a5bb9e8f&chksm=f9aee043ced969555aa489067c612c44c2ce4ac47b03587afb516fc5ed7f6f57cf921036749d&scene=21#wechat_redirect) 的维护者，该项目在快速发展阶段，欢迎大家关注。

## 参考

[1]

Implement rdb-only replication: https://github.com/redis/redis/pull/8303

[2]

Remove unnecessary replication backlog copy: https://github.com/redis/redis/pull/9157

[3]

Replication backlog and replicas use one global shared replication buffer: https://github.com/redis/redis/pull/9166

[4]

Fix not updating backlog histlen: https://github.com/redis/redis/pull/9713

[5]

Rebuild replication backlog index: https://github.com/redis/redis/pull/9720

**参考阅读：**

  

- [还在用ES查日志吗，快看看石墨文档 Clickhouse 日志架构玩法](http://mp.weixin.qq.com/s?__biz=MzAwMDU1MTE1OQ==&mid=2653558910&idx=1&sn=9bec04afef9a2a4d39f0d6a7379bf82d&chksm=813989e6b64e00f054c741beb190d62485f8ff121040ae9f6811ea709f0a5916c34c9d314a2d&scene=21#wechat_redirect)  
    
- [Java内存模型(Java Memory Model，JMM)](http://mp.weixin.qq.com/s?__biz=MzAwMDU1MTE1OQ==&mid=2653558884&idx=1&sn=74e727fc102101ca47bbafde3eac2f30&chksm=813989fcb64e00eabf46c151566996047dec073a5d9e274af919b912e639a90c0a3820a6bd9c&scene=21#wechat_redirect)
    
- [vivo数据库与存储平台的建设和探索](http://mp.weixin.qq.com/s?__biz=MzAwMDU1MTE1OQ==&mid=2653558867&idx=1&sn=b4a3502414896f1dbe0801d9ef1e34ca&chksm=813989cbb64e00ddee40db1e5ceea06ab648f1b8a9d8d1ff8328e0ace42d73575936ecddbf0e&scene=21#wechat_redirect)
    
- [爱奇艺内容中台之Serverless应用与实践](http://mp.weixin.qq.com/s?__biz=MzAwMDU1MTE1OQ==&mid=2653558835&idx=1&sn=54e86ce3b50864f9f113d939bf248779&chksm=813989abb64e00bdbd21596ac7b97f73889d55cb0fff206c021ef84a052aa63adbfa530b02a4&scene=21#wechat_redirect)
    
- [游戏案例｜Service Mesh 在欢乐游戏的应用演变和实践](http://mp.weixin.qq.com/s?__biz=MzAwMDU1MTE1OQ==&mid=2653558821&idx=1&sn=5adbd4e283270c6eb3f457f114eac64b&chksm=813989bdb64e00abc42345aa0de3120107077511c5ebfb68682406e82d8a93906d546d2def2f&scene=21#wechat_redirect)
    

  

技术原创及架构实践文章，欢迎通过公众号菜单「联系我们」进行投稿。  

  

**高可用架构**

**改变互联网的构建方式**

![](https://res.wx.qq.com/t/fed_upload/b39ef69e-c4d6-4169-8612-5f00a84860e7/wx-avatar-default.svg)

公众号

阅读 5028

​