Linux阅码场
 _2022年01月24日 08:00_
以下文章来源于程序猿的日常干货 ，作者谢昊成

](https://mp.weixin.qq.com/s?__biz=Mzg2OTc0ODAzMw==&mid=2247502841&idx=1&sn=228e70c33cb2a70cde4b8423abcb281a&source=41&key=daf9bdc5abc4e8d06bc47932f851321204aa03f620b90368927765a85c91f4c9170a7d15962ebe095b1b8b582d6291a91a04bb1f29dee0c883afe05924b7437c9b2599ef1d01adcfead13aac3fad127572fb9892ee8ca98c12ea0f1fab16a5e54c5a28a39bda917c1f95e67afa6b2f262e68941d362154cbfc0b4a58ade49c59&ascene=0&uin=MTEwNTU1MjgwMw%3D%3D&devicetype=Windows+11+x64&version=63090b19&lang=zh_CN&countrycode=CN&exportkey=n_ChQIAhIQR2ZFObWEhZsPInDdlHJovBLmAQIE97dBBAEAAAAAAM0uJwCRtagAAAAOpnltbLcz9gKNyK89dVj0JLIjpL2QwjlePpXI8%2B31BY%2BWgjIT%2Fu4LAAlPHdJlsYclch2yc5FtLO8p%2FgpV56LNXgpPctK2X00oX4DH%2Bfrt5nHYd1Z5613kK8MCYbQlRYcdl94Ty1V5ZxAIXIYMLmspQKiupyktvB9oLoi1bZ3w2Q3bdqfaB7ExlBsjH2KMUhM0XfJkgpmVB0swrtFyNUseQacr1cfV%2FqivJUkXVNY1Kfq6rHiFN%2BgobW%2BM5%2BfBBGhagJcy7rkNG14dKMXxxdTo&acctmode=0&pass_ticket=uf3La%2BlulmFRmsXLvPtwg%2BBYJXZ2h4GaW3DoY4WY2AQGRaByc5Ub%2F4ZRt%2BKuprHR&wx_header=1#)

**“** 给小猿点个关注吧！**”**  
## 查看系统meminfo

查看系统内存情况：

```c
free -h
```

查看meminfo，：

```c
/proc/meminfo ：
```

对于内存泄露的问题，需要关注的主要有：

```c
MemTotal:       32571632 kBMemFree:         3910664 kBMemAvailable:    7495000 kBBuffers:          124784 kBCached:          6162332 kBSwapCached:            0 kB...Slab:             591388 kBSReclaimable:     518688 kBSUnreclaim:        72700 kB...VmallocTotal:   34359738367 kBVmallocUsed:      620516 kBVmallocChunk:   34358945788 kB...HugePages_Total:      16HugePages_Free:        2HugePages_Rsvd:        0HugePages_Surp:        0Hugepagesize:    1048576 kB
```

vmalloc的指标含义为：

```c
VMallocTotal — The total amount of memory, in kilobytes, of total allocated virtual address space. VMallocUsed — The total amount of memory, in kilobytes, of used virtual address space.VMallocChunk — The largest contiguous block of memory, in kilobytes, of available virtual address space.
```

需要注意的是，在较新内核版本上这里的vmalloc指标已经被弃用，所以可能会看到输出为0，但是这种情况并不代表这系统中没有使用vmalloc内存。
## 查找泄露内存类型

通过查看meminfo是可以知道系统的内存消耗情况，但是知道了不同类型的内存消耗，那么如何评估消耗多大才算是不正常呢？这个便是定位问题的关键。通常的做法是通过与一台刚启动没多久的机器（正常机）作对比，从而发现异常的指标。

比如，我遇到的一个案例，对比正常机和故障机器内存：

```c
free：差距有3708Mcache+buff:消耗量差不多VmallocUsed：差距有2678MSUnreclaim：差距有887M
```

通过上面的信息对比发现，后两种类型的内存差距相加等于3565M。因此认为内存泄露是泄露在slab+vmalloc上。
## 查看slabinfo

```c
cat /proc/slabinfo
```

通过查看slabinfo能够看出系统中所有的slab使用情况，在我的机器上截取的部分输出如下所示：

```c
slabinfo - version: 2.1# name            <active_objs> <num_objs> <objsize> <objperslab> <pagesperslab> : tunables <limit> <batchcount> <sharedfactor> : slabdata <active_slabs> <num_slabs> <sharedavail>isofs_inode_cache     92     92    704   23    4 : tunables    0    0    0 : slabdata      4      4      0ext4_groupinfo_4k    420    420    144   28    1 : tunables    0    0    0 : slabdata     15     15      0ext4_inode_cache   76561  79569   1176   27    8 : tunables    0    0    0 : slabdata   2947   2947      0ext4_allocation_context    128    128    128   32    1 : tunables    0    0    0 : slabdata      4      4      0ext4_io_end         1344   1344     64   64    1 : tunables    0    0    0 : slabdata     21     21      0ext4_extent_status  91678  91902     40  102    1 : tunables    0    0    0 : slabdata    901    901      0jbd2_journal_handle    365    365     56   73    1 : tunables    0    0    0 : slabdata      5      5      0jbd2_journal_head   1632   1666    120   34    1 : tunables    0    0    0 : slabdata     49     49      0jbd2_revoke_table_s    256    256     16  256    1 : tunables    0    0    0 : slabdata      1      1      0jbd2_revoke_record_s    512    512     32  128    1 : tunables    0    0    0 : slabdata      4      4      0scsi_sense_cache      64     64    128   32    1 : tunables    0    0    0 : slabdata      2      2      0MPTCPv6                0      0   2112   15    8 : tunables    0    0    0 : slabdata      0      0      0ip6-frags              0      0    224   18    1 : tunables    0    0    0 : slabdata      0      0      0PINGv6                 0      0   1344   24    8 : tunables    0    0    0 : slabdata      0      0      0RAWv6                 48     48   1344   24    8 : tunables    0    0    0 : slabdata      2      2      0UDPv6                396    396   1472   22    8 : tunables    0    0    0 : slabdata     18     18      0tw_sock_TCPv6          0      0    272   30    2 : tunables    0    0    0 : slabdata      0      0      0request_sock_TCPv6      0      0    336   24    2 : tunables    0    0    0 : slabdata      0      0      0TCPv6                144    144   2624   12    8 : tunables    0    0    0 : slabdata     12     12      0
```

每一列所代表的含义为：

•name：slab object对象的名称•active_objs：活动对象（slab object）个数，即被申请走的对象个数（正在被使用）。•num_objs：总对象（slab object）个数，slab本身具备缓存作用，所以num_objs可能会比active_objs要大，这是一种合理的现象。•objsize：每个对象（slab object）大小，以字节为单位。•objperslab：slab中存放的是对象（slab object），这个指标表示一个slab中包含多少个对象（slab object）。•pagesperslab : tunables：一个slab占用几个page内存页。可以算一下，一个slab的大小为320*12=3840，小于4096（我这里，内存页为4K），所以一个slab只需占用1个page内存页。另外，cache对象本身需要存储一个额外的管理信息，即有overhead，所以一个xfs_buf slab的大小不止3840。•active_slabs：活动slab个数。•num_slabs：总slab个数。

slabtop也可以查看系统中的slab对象，并且可以按照不同的指标进行排序输出，比如可以按照slab活动对象个数排序查看slab，快速查找申请量最多的slab：

```c
slabtop -s -a
```
## 查看vmallocinfo：

内核中申请内存，除了使用slab，还可能使用vmallocinfo申请，对于vmalloc的查看，需要借助于另外一个接口：

```c
cat /proc/vmallocinfo
```

它的输出示例：

```c
virtual address range of the area, size in bytes, caller information of the creator, and optional information0xffffc90006404000-0xffffc90006406000    8192 ebt_register_table+0xa2/0x380 [ebtables] pages=1 vmalloc N0=10xffffc90006406000-0xffffc90006408000    8192 ebt_register_table+0xb7/0x380 [ebtables] pages=1 vmalloc N0=10xffffc90006408000-0xffffc9000640a000    8192 ebt_register_table+0xb7/0x380 [ebtables] pages=1 vmalloc N0=10xffffc9000640c000-0xffffc9000640e000    8192 ebt_register_table+0xa2/0x380 [ebtables] pages=1 vmalloc N0=10xffffc9000640e000-0xffffc90006410000    8192 ebt_register_table+0xb7/0x380 [ebtables] pages=1 vmalloc N0=1
```

每一列的含义分别是：

•占用的内核虚拟地址范围•占用的内存大小size，以字节为单位•申请vmalloc的函数•其他option信息

对于option选项信息，参考下表：![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E "null")
## 查看page alloc size

对于内核使用alloc_page接口直接申请的内存，并没有指标可以进行查看，这一类型的内存申请一般都是由内核在使用，一般开发者比较少用，但同样可以通过计算得出：

```
page alloc size = MemTotal - HugePages - MemFree - Slab - Vmalloc - anon page - cache - buffer
```

## 查看内存申请调用栈：

•ftrace

比如当发送内存泄露时，追查kmalloc调用的地方，可以使用调用栈跟踪触发器：

```
echo 'stacktrace' >       /sys/kernel/debug/tracing/events/kmem/kmalloc/trigger
```

如果要进一步细化，追查特定大小内存申请时的调用栈：

```
echo 'stacktrace:5 if (bytes_req >= 256 && bytes_req <= 512)' >  /sys/kernel/debug/tracing/events/kmem/kmalloc/trigger
```

stacktrace:5 这里是指只跟踪前5次事件，也就最多打印5次相关的调用栈。这个触发器的含义是只打印kmalloc申请大小为256到512大小之间的事件。如果要跟踪slab对象的申请，比如我们知道了某个slab可能存在泄漏，那么查看代码找到这个slab对象的size，可以这样：

```c
echo 'stacktrace:5 if (bytes_req == 256)'  > /sys/kernel/debug/tracing/events/kmem/kmem_cache_alloc/trigger
```

这样将只打印对应size大小的内核代码调用栈，那么其中必然会包含我们预期要关注的slab对象，那么就知道该对象会在哪里被调用了。

•perf

使用perf也可以实现一样的功能，比如：

```c
perf record -e 'kmem:kmem_cache_alloc' -a -g --filter 'bytes_req == 256'
```

## 查看特定slab对象的内存分配

上面提到使用ftrace或者perf可以跟踪内存分配的调用栈，它们底层都是依赖于kmem tracepoint来实现的，获取的信息只能是以size大小来过滤，而无法指定某一个slab来作为trace对象，那么现在就来介绍另外一种能够trace特定slab对象的方法。这种方法借助的是slub debug特性，因此主要针对的是slub分配器，至于其他类型的分配器并不具有通用性。对于slub分配器，它在sysfs中提供了一些debug的文件节点：

```c
/sys/kernel/slab/<slab-name>/<debug-option>
```

其中有一个trace选项，可以用来跟踪slab对象的申请和分配，比如我们想要跟踪mm_struct这个对象：

```c
echo 1 > /sys/kernel/slab/mm_struct/trace
```

这个选项打开之后会在dmesg系统日志中打印内存的申请和释放过程，以及对应的调用堆栈，比如：

```c
[3904231.500556] TRACE mm_struct free 0xffff88085be9d140 inuse=18 fp=0xffff88085be99900[3904231.500570] Object ffff88085be9d1b0: 21 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  !...............[3904231.500733] Call Trace:[3904231.500736]  [<ffffffff816b4d9a>] dump_stack+0x19/0x1b[3904231.500740]  [<ffffffff816b2018>] free_debug_processing+0x1ca/0x259[3904231.500743]  [<ffffffff81088997>] ? __mmdrop+0x77/0xb0[3904231.500746]  [<ffffffff811eaeb0>] __slab_free+0x250/0x2f0[3904231.500749]  [<ffffffff811eb132>] kmem_cache_free+0x1e2/0x200[3904231.500752]  [<ffffffff81088997>] __mmdrop+0x77/0xb0[3904231.500754]  [<ffffffff81088ec8>] mmput+0xc8/0xf0[3904231.500758]  [<ffffffff81214f74>] flush_old_exec+0x434/0x820[3904231.500760]  [<ffffffff8126c44c>] load_elf_binary+0x33c/0xe00[3904231.500770]  [<ffffffffc0097064>] ? load_misc_binary+0x64/0x460 [binfmt_misc][3904231.500772]  [<ffffffff812e27c3>] ? ima_get_action+0x23/0x30[3904231.500776]  [<ffffffff812e1e2e>] ? process_measurement+0x8e/0x250[3904231.500779]  [<ffffffff812e22e9>] ? ima_bprm_check+0x49/0x50[3904231.500781]  [<ffffffff8126c110>] ? load_elf_library+0x220/0x220[3904231.500784]  [<ffffffff8121468f>] search_binary_handler+0xef/0x310[3904231.500786]  [<ffffffff81215bd6>] do_execve_common.isra.25+0x5b6/0x6c0[3904231.500789]  [<ffffffff81215f79>] SyS_execve+0x29/0x30[3904231.500792]  [<ffffffff816c6719>] stub_execve+0x69/0xa0
```

---

https://www.kernel.org/doc/html/latest/filesystems/proc.html

  

阅读 5562

​

写留言

**留言 3**

- Cloud Huang
    
    2022年1月24日
    
    赞3
    
    虽然看不懂但是还是要看
    
- 糖铺
    
    2022年1月25日
    
    赞
    
    真香
    
- halowell
    
    2022年1月25日
    
    赞
    
    准备拿到我司的程序上试试。有个地方有内存泄露，知道怎么解决但想不明白原因。无锁队列中存的结构体，结构体中有string,从队列中取数据用的vector<结构体指针> 。解决办法是把string换成char*,动态创建，用完释放。
    

已无更多数据

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/YHBSoNHqDiaH5V6ycE2TlM9McXhAbaV9aUBaTAibVhLibXH1rL4JTMb2eoQfdCicTm4wycOsopO82jG3szvYF3mTJw/300?wx_fmt=png&wxfrom=18)

Linux阅码场

34219

3

写留言

**留言 3**

- Cloud Huang
    
    2022年1月24日
    
    赞3
    
    虽然看不懂但是还是要看
    
- 糖铺
    
    2022年1月25日
    
    赞
    
    真香
    
- halowell
    
    2022年1月25日
    
    赞
    
    准备拿到我司的程序上试试。有个地方有内存泄露，知道怎么解决但想不明白原因。无锁队列中存的结构体，结构体中有string,从队列中取数据用的vector<结构体指针> 。解决办法是把string换成char*,动态创建，用完释放。
    

已无更多数据