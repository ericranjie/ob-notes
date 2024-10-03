
作者：[安庆](http://www.wowotech.net/author/539 "oppo混合云内核&虚拟化负责人，架构并孵化了oppo的云游戏，云手机等产品。") 发布于：2021-7-6 10:38 分类：[Linux内核分析](http://www.wowotech.net/sort/linux_kenrel)

背景：这个是在centos 7.6.1810的环境上复现的，智能网卡是目前很多云服务器上的网卡标配，在oppo主要用于vpc等场景，智能网卡的代码随着功能的增强导致复杂度一直在上升，驱动的bug一直是内核bug中的大头，在遇到类似问题时，内核开发者由于对驱动代码不熟悉，排查会比较费劲,本身涉及的背景知识有：dma_pool,dma_page,net_device,mlx5_core_dev设备，设备卸载，uaf问题等,另外，这个bug目测在最新的linux基线也没有解决,本文单独拿出来列举是因为uaf问题相对比较独特。
下面列一下我们是怎么排查并解决这个问题的。

# 一、故障现象
oppo云内核团队接到连通性告警报障，发现机器复位：

```c
UPTIME: 00:04:16-------------运行的时间很短
LOAD AVERAGE: 0.25, 0.23, 0.11
TASKS: 2027
RELEASE: 3.10.0-1062.18.1.el7.x86_64
MEMORY: 127.6 GB
PANIC: "BUG: unable to handle kernel NULL pointer dereference at           (null)"
PID: 23283
COMMAND: "spider-agent"
TASK: ffff9d1fbb090000  [THREAD_INFO: ffff9d1f9a0d8000]
CPU: 0
STATE: TASK_RUNNING (PANIC)

crash> bt
PID: 23283  TASK: ffff9d1fbb090000  CPU: 0   COMMAND: "spider-agent"
 #0 [ffff9d1f9a0db650] machine_kexec at ffffffffb6665b34
 #1 [ffff9d1f9a0db6b0] __crash_kexec at ffffffffb6722592
 #2 [ffff9d1f9a0db780] crash_kexec at ffffffffb6722680
 #3 [ffff9d1f9a0db798] oops_end at ffffffffb6d85798
 #4 [ffff9d1f9a0db7c0] no_context at ffffffffb6675bb4
 #5 [ffff9d1f9a0db810] __bad_area_nosemaphore at ffffffffb6675e82
 #6 [ffff9d1f9a0db860] bad_area_nosemaphore at ffffffffb6675fa4
 #7 [ffff9d1f9a0db870] __do_page_fault at ffffffffb6d88750
 #8 [ffff9d1f9a0db8e0] do_page_fault at ffffffffb6d88975
 #9 [ffff9d1f9a0db910] page_fault at ffffffffb6d84778
    [exception RIP: dma_pool_alloc+427]//caq:异常地址
    RIP: ffffffffb680efab  RSP: ffff9d1f9a0db9c8  RFLAGS: 00010046
    RAX: 0000000000000246  RBX: ffff9d0fa45f4c80  RCX: 0000000000001000
    RDX: 0000000000000000  RSI: 0000000000000246  RDI: ffff9d0fa45f4c10
    RBP: ffff9d1f9a0dba20   R8: 000000000001f080   R9: ffff9d00ffc07c00
    R10: ffffffffc03e10c4  R11: ffffffffb67dd6fd  R12: 00000000000080d0
    R13: ffff9d0fa45f4c10  R14: ffff9d0fa45f4c00  R15: 0000000000000000
    ORIG_RAX: ffffffffffffffff  CS: 0010  SS: 0018
#10 [ffff9d1f9a0dba28] mlx5_alloc_cmd_msg at ffffffffc03e10e3 [mlx5_core]//涉及的模块
#11 [ffff9d1f9a0dba78] cmd_exec at ffffffffc03e3c92 [mlx5_core]
#12 [ffff9d1f9a0dbb18] mlx5_cmd_exec at ffffffffc03e442b [mlx5_core]
#13 [ffff9d1f9a0dbb48] mlx5_core_access_reg at ffffffffc03ee354 [mlx5_core]
#14 [ffff9d1f9a0dbba0] mlx5_query_port_ptys at ffffffffc03ee411 [mlx5_core]
#15 [ffff9d1f9a0dbc10] mlx5e_get_link_ksettings at ffffffffc0413035 [mlx5_core]
#16 [ffff9d1f9a0dbce8] __ethtool_get_link_ksettings at ffffffffb6c56d06
#17 [ffff9d1f9a0dbd48] speed_show at ffffffffb6c705b8
#18 [ffff9d1f9a0dbdd8] dev_attr_show at ffffffffb6ab1643
#19 [ffff9d1f9a0dbdf8] sysfs_kf_seq_show at ffffffffb68d709f
#20 [ffff9d1f9a0dbe18] kernfs_seq_show at ffffffffb68d57d6
#21 [ffff9d1f9a0dbe28] seq_read at ffffffffb6872a30
#22 [ffff9d1f9a0dbe98] kernfs_fop_read at ffffffffb68d6125
#23 [ffff9d1f9a0dbed8] vfs_read at ffffffffb684a8ff
#24 [ffff9d1f9a0dbf08] sys_read at ffffffffb684b7bf
#25 [ffff9d1f9a0dbf50] system_call_fastpath at ffffffffb6d8dede
    RIP: 00000000004a5030  RSP: 000000c001099378  RFLAGS: 00000212
    RAX: 0000000000000000  RBX: 000000c000040000  RCX: ffffffffffffffff
    RDX: 000000000000000a  RSI: 000000c00109976e  RDI: 000000000000000d---read的文件fd编号
    RBP: 000000c001099640   R8: 0000000000000000   R9: 0000000000000000
    R10: 0000000000000000  R11: 0000000000000206  R12: 000000000000000c
    R13: 0000000000000032  R14: 0000000000f710c4  R15: 0000000000000000
    ORIG_RAX: 0000000000000000  CS: 0033  SS: 002b                
```

从堆栈看，是某进程读取文件触发了一个内核态的空指针引用。
# 二、故障现象分析

从堆栈信息看：

1、当时进程打开fd编号为13的文件，这个从rdi的值可以看出。
2、speed_show 和 __ethtool_get_link_ksettings 表示在读取网卡的速率值
下面看下打开的文件是哪个，

```c
crash> files 23283
PID: 23283  TASK: ffff9d1fbb090000  CPU: 0   COMMAND: "spider-agent"
ROOT: /rootfs    CWD: /rootfs/home/service/app/spider
 FD       FILE            DENTRY           INODE       TYPE PATH
....
  9 ffff9d0f5709b200 ffff9d1facc80a80 ffff9d1069a194d0 REG  /rootfs/sys/devices/pci0000:3a/0000:3a:00.0/0000:3b:00.0/net/p1p1/speed---这个还在
 10 ffff9d0f4a45a400 ffff9d0f9982e240 ffff9d0fb7b873a0 REG  /rootfs/sys/devices/pci0000:5d/0000:5d:00.0/0000:5e:00.0/net/p3p1/speed---注意对应关系  0000:5e:00.0 对应p3p1
 11 ffff9d0f57098f00 ffff9d1facc80240 ffff9d1069a1b530 REG  /rootfs/sys/devices/pci0000:3a/0000:3a:00.0/0000:3b:00.1/net/p1p2/speed---这个还在
 13 ffff9d0f4a458a00 ffff9d0f9982e0c0 ffff9d0fb7b875f0 REG  /rootfs/sys/devices/pci0000:5d/0000:5d:00.0/0000:5e:00.1/net/p3p2/speed---注意对应关系 0000:5e:00.1 对应p3p2
....
```

注意上面 pci编号与 网卡名称的对应关系，后面会用到。
打开文件读取speed本身应该是一个很常见的流程，
下面从 exception RIP: dma_pool_alloc+427 进一步分析为什么触发了NULL pointer dereference
展开具体的堆栈如下：

```c
#9 [ffff9d1f9a0db910] page_fault at ffffffffb6d84778
    [exception RIP: dma_pool_alloc+427]
    RIP: ffffffffb680efab  RSP: ffff9d1f9a0db9c8  RFLAGS: 00010046
    RAX: 0000000000000246  RBX: ffff9d0fa45f4c80  RCX: 0000000000001000
    RDX: 0000000000000000  RSI: 0000000000000246  RDI: ffff9d0fa45f4c10
    RBP: ffff9d1f9a0dba20   R8: 000000000001f080   R9: ffff9d00ffc07c00
    R10: ffffffffc03e10c4  R11: ffffffffb67dd6fd  R12: 00000000000080d0
    R13: ffff9d0fa45f4c10  R14: ffff9d0fa45f4c00  R15: 0000000000000000
    ORIG_RAX: ffffffffffffffff  CS: 0010  SS: 0018
    ffff9d1f9a0db918: 0000000000000000 ffff9d0fa45f4c00 
    ffff9d1f9a0db928: ffff9d0fa45f4c10 00000000000080d0 
    ffff9d1f9a0db938: ffff9d1f9a0dba20 ffff9d0fa45f4c80 
    ffff9d1f9a0db948: ffffffffb67dd6fd ffffffffc03e10c4 
    ffff9d1f9a0db958: ffff9d00ffc07c00 000000000001f080 
    ffff9d1f9a0db968: 0000000000000246 0000000000001000 
    ffff9d1f9a0db978: 0000000000000000 0000000000000246 
    ffff9d1f9a0db988: ffff9d0fa45f4c10 ffffffffffffffff 
    ffff9d1f9a0db998: ffffffffb680efab 0000000000000010 
    ffff9d1f9a0db9a8: 0000000000010046 ffff9d1f9a0db9c8 
    ffff9d1f9a0db9b8: 0000000000000018 ffffffffb680ee45 
    ffff9d1f9a0db9c8: ffff9d0faf9fec40 0000000000000000 
    ffff9d1f9a0db9d8: ffff9d0faf9fec48 ffffffffb682669c 
    ffff9d1f9a0db9e8: ffff9d00ffc07c00 00000000618746c1 
    ffff9d1f9a0db9f8: 0000000000000000 0000000000000000 
    ffff9d1f9a0dba08: ffff9d0faf9fec40 0000000000000000 
    ffff9d1f9a0dba18: ffff9d0fa3c800c0 ffff9d1f9a0dba70 
    ffff9d1f9a0dba28: ffffffffc03e10e3 

#10 [ffff9d1f9a0dba28] mlx5_alloc_cmd_msg at ffffffffc03e10e3 [mlx5_core]
    ffff9d1f9a0dba30: ffff9d0f4eebee00 0000000000000001 
    ffff9d1f9a0dba40: 000000d0000080d0 0000000000000050 
    ffff9d1f9a0dba50: ffff9d0fa3c800c0 0000000000000005 --r12是rdi ,ffff9d0fa3c800c0
    ffff9d1f9a0dba60: ffff9d0fa3c803e0 ffff9d1f9d87ccc0 
    ffff9d1f9a0dba70: ffff9d1f9a0dbb10 ffffffffc03e3c92 
#11 [ffff9d1f9a0dba78] cmd_exec at ffffffffc03e3c92 [mlx5_core]
```

从堆栈中取出对应的 mlx5_core_dev 为 ffff9d0fa3c800c0

```c
crash> mlx5_core_dev.cmd ffff9d0fa3c800c0 -xo
struct mlx5_core_dev {
  [ffff9d0fa3c80138] struct mlx5_cmd cmd;
}

crash> mlx5_cmd.pool ffff9d0fa3c80138
  pool = 0xffff9d0fa45f4c00------这个就是dma_pool，写驱动代码的同学会经常遇到

```

出问题的代码行号为：

```c
crash> dis -l dma_pool_alloc+427 -B 5
/usr/src/debug/kernel-3.10.0-1062.18.1.el7/linux-3.10.0-1062.18.1.el7.x86_64/mm/dmapool.c: 334
0xffffffffb680efab <dma_pool_alloc+427>:        mov    (%r15),%ecx
而对应的r15，从上面的堆栈看，确实是null。
    305 void *dma_pool_alloc(struct dma_pool *pool, gfp_t mem_flags,
    306                      dma_addr_t *handle)
    307 {
...
    315         spin_lock_irqsave(&pool->lock, flags);
    316         list_for_each_entry(page, &pool->page_list, page_list) {
    317                 if (page->offset < pool->allocation)---//caq:当前满足条件
    318                         goto ready;//caq:跳转到ready
    319         }
    320 
    321         /* pool_alloc_page() might sleep, so temporarily drop &pool->lock */
    322         spin_unlock_irqrestore(&pool->lock, flags);
    323 
    324         page = pool_alloc_page(pool, mem_flags & (~__GFP_ZERO));
    325         if (!page)
    326                 return NULL;
    327 
    328         spin_lock_irqsave(&pool->lock, flags);
    329 
    330         list_add(&page->page_list, &pool->page_list);
    331  ready:
    332         page->in_use++;//caq:表示正在引用
    333         offset = page->offset;//从上次用完的地方开始使用
    334         page->offset = *(int *)(page->vaddr + offset);//caq:出问题的行号
...
    }
```

从上面的代码看，page->vaddr为NULL,offset也为0，才会引用NULL，page有两个来源，
第一种是从pool中的page_list中取，
第二种是从pool_alloc_page临时申请，当然申请之后会挂入到pool中的page_list,
下面查看一下这个page_list.
```c
crash> dma_pool ffff9d0fa45f4c00 -x
struct dma_pool {
  page_list = {
    next = 0xffff9d0fa45f4c80, 
    prev = 0xffff9d0fa45f4c00
  }, 

  lock = {
    {
      rlock = {
        raw_lock = {
          val = {
            counter = 0x1
          }
        }
      }
    }
  }, 

  size = 0x400, 
  dev = 0xffff9d1fbddec098, 
  allocation = 0x1000, 
  boundary = 0x1000, 
  name = "mlx5_cmd\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000", 
  pools = {
    next = 0xdead000000000100, 
    prev = 0xdead000000000200
  }
}

crash> list dma_pool.page_list -H 0xffff9d0fa45f4c00 -s dma_page.offset,vaddr
ffff9d0fa45f4c80
  offset = 0
  vaddr = 0x0
ffff9d0fa45f4d00
  offset = 0
  vaddr = 0x0
```

从 dma_pool_alloc 函数的代码逻辑看，pool->page_list确实不为空，而且满足
if (page->offset < pool->allocation) 的条件，所以第一个page应该是 ffff9d0fa45f4c80
也就是从第一种情况取出的：

```c
crash> dma_page ffff9d0fa45f4c80
struct dma_page {
  page_list = {
    next = 0xffff9d0fa45f4d00, 
    prev = 0xffff9d0fa45f4c80
  }, 
  vaddr = 0x0, //caq:这个异常，引用这个将导致crash
  dma = 0, 
  in_use = 1, //caq:这个标记为在使用，符合page->in_use++;
  offset = 0
}
```

问题分析到这里，因为dma_pool中的page，申请之后，vaddr都会初始化，
一般在pool_alloc_page 中进行初始化，怎么可能会NULL呢？
然后查看一下这个地址：

```c
crash> kmem ffff9d0fa45f4c80-------这个是dma_pool中的page
CACHE            NAME                 OBJSIZE  ALLOCATED     TOTAL  SLABS  SSIZE
ffff9d00ffc07900 kmalloc-128//caq:注意这个长度  128       8963     14976    234     8k
  SLAB              MEMORY            NODE  TOTAL  ALLOCATED  FREE
  ffffe299c0917d00  ffff9d0fa45f4000     0     64         29    35
  FREE / [ALLOCATED]
   ffff9d0fa45f4c80  

      PAGE         PHYSICAL      MAPPING       INDEX CNT FLAGS
ffffe299c0917d00 10245f4000                0 ffff9d0fa45f4c00  1 2fffff00004080 slab,head
```

由于以前用过类似的dma函数，印象中dma_page没有这么大，再看看第二个dma_page如下：

```c
crash> kmem ffff9d0fa45f4d00
CACHE            NAME                 OBJSIZE  ALLOCATED     TOTAL  SLABS  SSIZE
ffff9d00ffc07900 kmalloc-128              128       8963     14976    234     8k
  SLAB              MEMORY            NODE  TOTAL  ALLOCATED  FREE
  ffffe299c0917d00  ffff9d0fa45f4000     0     64         29    35
  FREE / [ALLOCATED]
   ffff9d0fa45f4d00  

      PAGE         PHYSICAL      MAPPING       INDEX CNT FLAGS
ffffe299c0917d00 10245f4000                0 ffff9d0fa45f4c00  1 2fffff00004080 slab,head

crash> dma_page ffff9d0fa45f4d00
struct dma_page {
  page_list = {
    next = 0xffff9d0fa45f5000, 
    prev = 0xffff9d0fa45f4d00
  }, 
  vaddr = 0x0, -----------caq：也是null
  dma = 0, 
  in_use = 0, 
  offset = 0
}

crash> list dma_pool.page_list -H 0xffff9d0fa45f4c00 -s dma_page.offset,vaddr
ffff9d0fa45f4c80
  offset = 0
  vaddr = 0x0
ffff9d0fa45f4d00
  offset = 0
  vaddr = 0x0
ffff9d0fa45f5000
  offset = 0
  vaddr = 0x0
.........
```

看来不仅是第一个dma_page有问题，所有在pool中的dma_page单元都一样，

那直接查看一下dma_page的正常大小：

```c
crash> p sizeof(struct dma_page)
$3 = 40
```

按道理长度才40字节，就算申请slab的话，也应该扩展为64字节才对，怎么可能像上面那个dma_page

一样是128字节呢？为了解开这个疑惑，找一个正常的其他节点对比一下：

```c
crash> net
   NET_DEVICE     NAME   IP ADDRESS(ES)
ffff8f9e800be000  lo     127.0.0.1
ffff8f9e62640000  p1p1   
ffff8f9e626c0000  p1p2   
ffff8f9e627c0000  p3p1   -----//caq:以这个为例
ffff8f9e62100000  p3p2   
```

然后根据代码：通过net_device查看mlx5e_priv：

```c
static int mlx5e_get_link_ksettings(struct net_device *netdev,
                    struct ethtool_link_ksettings *link_ksettings) {
...
    struct mlx5e_priv *priv    = netdev_priv(netdev);
...
}

static inline void *netdev_priv(const struct net_device *dev)
{
  return (char *)dev + ALIGN(sizeof(struct net_device), NETDEV_ALIGN);
}

crash> px sizeof(struct net_device)
$2 = 0x8c0

crash> mlx5e_priv.mdev ffff8f9e627c08c0---根据偏移计算
  mdev = 0xffff8f9e67c400c0

crash> mlx5_core_dev.cmd 0xffff8f9e67c400c0 -xo
struct mlx5_core_dev {
  [ffff8f9e67c40138] struct mlx5_cmd cmd;
}

crash> mlx5_cmd.pool ffff8f9e67c40138
  pool = 0xffff8f9e7bf48f80

crash> dma_pool 0xffff8f9e7bf48f80
struct dma_pool {
  page_list = {
    next = 0xffff8f9e79c60880, //caq:其中的一个dma_page
    prev = 0xffff8fae6e4db800
  }, 

.......
  size = 1024, 
  dev = 0xffff8f9e800b3098, 
  allocation = 4096, 
  boundary = 4096, 

  name = "mlx5_cmd\000\217\364{\236\217\377\377\300\217\364{\236\217\377\377\200\234>\250\217\217\377\377", 

  pools = {

    next = 0xffff8f9e800b3290, 

    prev = 0xffff8f9e800b3290

  }

}

crash> dma_page 0xffff8f9e79c60880     //caq:查看这个dma_page

struct dma_page {

  page_list = {

    next = 0xffff8f9e79c60840, -------其中的一个dma_page

    prev = 0xffff8f9e7bf48f80

  }, 

  vaddr = 0xffff8f9e6fc9b000, //caq:正常vaddr不可能会NULL的

  dma = 69521223680, 

  in_use = 0, 

  offset = 0

}

  

crash> kmem 0xffff8f9e79c60880

CACHE            NAME             OBJSIZE  ALLOCATED     TOTAL  SLABS  SSIZE

ffff8f8fbfc07b00 kmalloc-64--正常长度    64     667921    745024  11641  4k

  SLAB              MEMORY            NODE  TOTAL  ALLOCATED  FREE

  ffffde5140e71800  ffff8f9e79c60000     0     64         64     0

  FREE / [ALLOCATED]

  [ffff8f9e79c60880]

  

      PAGE         PHYSICAL      MAPPING       INDEX CNT FLAGS

ffffde5140e71800 1039c60000                0        0  1 2fffff00000080 slab

```

以上操作要求对net_device和mlx5相关驱动代码比较熟悉。

相比于异常的dma_page,正常的dma_page是一个64字节的slab，所以很明显，

要么这个是一个踩内存问题，要么是一个uaf(used after free )问题。

一般问题查到这，怎么快速判断是哪一种类型呢？因为这两种问题，涉及到内存紊乱，

一般都比较难查，这时候需要跳出来，

我们先看一下其他运行进程的情况，找到了一个进程如下：

  

```c

crash> bt 48263

PID: 48263  TASK: ffff9d0f4ee0a0e0  CPU: 56  COMMAND: "reboot"

 #0 [ffff9d0f95d7f958] __schedule at ffffffffb6d80d4a

 #1 [ffff9d0f95d7f9e8] schedule at ffffffffb6d811f9

 #2 [ffff9d0f95d7f9f8] schedule_timeout at ffffffffb6d7ec48

 #3 [ffff9d0f95d7faa8] wait_for_completion_timeout at ffffffffb6d81ae5

 #4 [ffff9d0f95d7fb08] cmd_exec at ffffffffc03e41c9 [mlx5_core]

 #5 [ffff9d0f95d7fba8] mlx5_cmd_exec at ffffffffc03e442b [mlx5_core]

 #6 [ffff9d0f95d7fbd8] mlx5_core_destroy_mkey at ffffffffc03f085d [mlx5_core]

 #7 [ffff9d0f95d7fc40] mlx5_mr_cache_cleanup at ffffffffc0c60aab [mlx5_ib]

 #8 [ffff9d0f95d7fca8] mlx5_ib_stage_pre_ib_reg_umr_cleanup at ffffffffc0c45d32 [mlx5_ib]

 #9 [ffff9d0f95d7fcc0] __mlx5_ib_remove at ffffffffc0c4f450 [mlx5_ib]

#10 [ffff9d0f95d7fce8] mlx5_ib_remove at ffffffffc0c4f4aa [mlx5_ib]

#11 [ffff9d0f95d7fd00] mlx5_detach_device at ffffffffc03fe231 [mlx5_core]

#12 [ffff9d0f95d7fd30] mlx5_unload_one at ffffffffc03dee90 [mlx5_core]

#13 [ffff9d0f95d7fd60] shutdown at ffffffffc03def80 [mlx5_core]

#14 [ffff9d0f95d7fd80] pci_device_shutdown at ffffffffb69d1cda

#15 [ffff9d0f95d7fda8] device_shutdown at ffffffffb6ab3beb

#16 [ffff9d0f95d7fdd8] kernel_restart_prepare at ffffffffb66b7916

#17 [ffff9d0f95d7fde8] kernel_restart at ffffffffb66b7932

#18 [ffff9d0f95d7fe00] SYSC_reboot at ffffffffb66b7ba9

#19 [ffff9d0f95d7ff40] sys_reboot at ffffffffb66b7c4e

#20 [ffff9d0f95d7ff50] system_call_fastpath at ffffffffb6d8dede

    RIP: 00007fc9be7a5226  RSP: 00007ffd9a19e448  RFLAGS: 00010246

    RAX: 00000000000000a9  RBX: 0000000000000004  RCX: 0000000000000000

    RDX: 0000000001234567  RSI: 0000000028121969  RDI: fffffffffee1dead

    RBP: 0000000000000002   R8: 00005575d529558c   R9: 0000000000000000

    R10: 00007fc9bea767b8  R11: 0000000000000206  R12: 0000000000000000

    R13: 00007ffd9a19e690  R14: 0000000000000000  R15: 0000000000000000

    ORIG_RAX: 00000000000000a9  CS: 0033  SS: 002b

  

```

  

为什么会关注这个进程，因为这么多年以来，因为卸载模块引发的uaf问题排查不低于20次了，

有时候是reboot，有时候是unload，有时候是在work中释放资源，

所以直觉上，觉得和这个卸载有很大关系。

下面分析一下，reboot流程里面操作到哪了。

  

```c

2141 void device_shutdown(void)

   2142 {

   2143         struct device *dev, *parent;

   2144 

   2145         spin_lock(&devices_kset->list_lock);

   2146         /*

   2147          * Walk the devices list backward, shutting down each in turn.

   2148          * Beware that device unplug events may also start pulling

   2149          * devices offline, even as the system is shutting down.

   2150          */

   2151         while (!list_empty(&devices_kset->list)) {

   2152                 dev = list_entry(devices_kset->list.prev, struct device,

   2153                                 kobj.entry);

........

   2178                 if (dev->device_rh && dev->device_rh->class_shutdown_pre) {

   2179                         if (initcall_debug)

   2180                                 dev_info(dev, "shutdown_pre\n");

   2181                         dev->device_rh->class_shutdown_pre(dev);

   2182                 }

   2183                 if (dev->bus && dev->bus->shutdown) {

   2184                         if (initcall_debug)

   2185                                 dev_info(dev, "shutdown\n");

   2186                         dev->bus->shutdown(dev);

   2187                 } else if (dev->driver && dev->driver->shutdown) {

   2188                         if (initcall_debug)

   2189                                 dev_info(dev, "shutdown\n");

   2190                         dev->driver->shutdown(dev);

   2191                 }

   }

```

从上面代码看出以下两点：

  

1、每个device 的 kobj.entry 成员串接在 devices_kset->list 中。

  

2、每个设备的shutdown流程从 device_shutdown 看是串行的。

  

从reboot 的堆栈看，卸载一个 mlx设备的流程包含如下：

  

pci_device_shutdown-->shutdown-->mlx5_unload_one-->mlx5_detach_device

                                                -->mlx5_cmd_cleanup-->dma_pool_destroy

  

mlx5_detach_device的流程分支为：

  

```c

void dma_pool_destroy(struct dma_pool *pool)

{

.......

        while (!list_empty(&pool->page_list)) {//caq:将pool中的dma_page一一删除

                struct dma_page *page;

                page = list_entry(pool->page_list.next,

                                  struct dma_page, page_list);

                if (is_page_busy(page)) {

.......

                        list_del(&page->page_list);

                        kfree(page);

                } else

                        pool_free_page(pool, page);//每个dma_page去释放

        }

  

        kfree(pool);//caq：释放pool

.......        

}

  

static void pool_free_page(struct dma_pool *pool, struct dma_page *page)

{

        dma_addr_t dma = page->dma;

  

#ifdef  DMAPOOL_DEBUG

        memset(page->vaddr, POOL_POISON_FREED, pool->allocation);

#endif

        dma_free_coherent(pool->dev, pool->allocation, page->vaddr, dma);

        list_del(&page->page_list);//caq:释放后会将page_list成员毒化

        kfree(page);

}

  

```

从reboot的堆栈中，查看对应的 信息

  

```c

 #4 [ffff9d0f95d7fb08] cmd_exec at ffffffffc03e41c9 [mlx5_core]

    ffff9d0f95d7fb10: ffffffffb735b580 ffff9d0f904caf18 

    ffff9d0f95d7fb20: ffff9d00ff801da8 ffff9d0f23121200 

    ffff9d0f95d7fb30: ffff9d0f23121740 ffff9d0fa7480138 

    ffff9d0f95d7fb40: 0000000000000000 0000001002020000 

    ffff9d0f95d7fb50: 0000000000000000 ffff9d0f95d7fbe8 

    ffff9d0f95d7fb60: ffff9d0f00000000 0000000000000000 

    ffff9d0f95d7fb70: 00000000756415e3 ffff9d0fa74800c0 ----mlx5_core_dev设备，对应的是 p3p1，

    ffff9d0f95d7fb80: ffff9d0f95d7fbf8 ffff9d0f95d7fbe8 

    ffff9d0f95d7fb90: 0000000000000246 ffff9d0f8f3a20b8 

    ffff9d0f95d7fba0: ffff9d0f95d7fbd0 ffffffffc03e442b 

 #5 [ffff9d0f95d7fba8] mlx5_cmd_exec at ffffffffc03e442b [mlx5_core]

    ffff9d0f95d7fbb0: 0000000000000000 ffff9d0fa74800c0 

    ffff9d0f95d7fbc0: ffff9d0f8f3a20b8 ffff9d0fa74bea00 

    ffff9d0f95d7fbd0: ffff9d0f95d7fc38 ffffffffc03f085d 

 #6 [ffff9d0f95d7fbd8] mlx5_core_destroy_mkey at ffffffffc03f085d [mlx5_core]

 ```

要注意，reboot正在释放的 mlx5_core_dev 是 ffff9d0fa74800c0，这个设备对应的net_device是：

p3p1,而 23283 进程正在访问的 mlx5_core_dev 是 ffff9d0fa3c800c0 ，对应的是 p3p2。

```c

crash> net

   NET_DEVICE     NAME   IP ADDRESS(ES)

ffff9d0fc003e000  lo     127.0.0.1

ffff9d1fad200000  p1p1   

ffff9d0fa0700000  p1p2   

ffff9d0fa00c0000  p3p1  对应的 mlx5_core_dev 是 ffff9d0fa74800c0

ffff9d0fa0200000  p3p2  对应的 mlx5_core_dev 是 ffff9d0fa3c800c0

```

我们看下目前还残留在 devices_kset 中的device：

  

```c

crash> p devices_kset

devices_kset = $4 = (struct kset *) 0xffff9d1fbf4e70c0

crash> p devices_kset.list

$5 = {

  next = 0xffffffffb72f2a38, 

  prev = 0xffff9d0fbe0ea130

}

  

crash> list -H -o 0x18 0xffffffffb72f2a38 -s device.kobj.name >device.list

  

我们发现p3p1 与 p3p2均不在 device.list中，

  

[root@it202-seg-k8s-prod001-node-10-27-96-220 127.0.0.1-2020-12-07-10:58:06]# grep 0000:5e:00.0 device.list //caq:未找到 这个是 p3p1，当前reboot流程正在卸载。

[root@it202-seg-k8s-prod001-node-10-27-96-220 127.0.0.1-2020-12-07-10:58:06]# grep 0000:5e:00.1 device.list //caq:未找到，这个是 p3p2,已经卸载完

[root@it202-seg-k8s-prod001-node-10-27-96-220 127.0.0.1-2020-12-07-10:58:06]# grep 0000:3b:00.0  device.list //caq:这个mlx5设备还没unload

  kobj.name = 0xffff9d1fbe82aa70 "0000:3b:00.0",

[root@it202-seg-k8s-prod001-node-10-27-96-220 127.0.0.1-2020-12-07-10:58:06]# grep 0000:3b:00.1 device.list //caq:这个mlx5设备还没unload

  kobj.name = 0xffff9d1fbe82aae0 "0000:3b:00.1",

  

```

由于 p3p2 与 p3p1均不在 device.list中，而根据 pci_device_shutdown的串行卸载流程，

当前正在卸载的是 p3p1，所以很确定的是 23283 进程访问的是卸载后的cmd_pool,

根据前面描述的卸载流程 ：

pci_device_shutdown-->shutdown-->mlx5_unload_one-->mlx5_cmd_cleanup-->dma_pool_destroy

此时的pool已经被释放了，pool中的dma_page均无效的。

  

然后尝试google对应的bug，查看到一个跟当前现象极为相似，

redhat遇到了类似的问题：https://access.redhat.com/solutions/5132931

  

但是，红帽在这个链接中认为解决了uaf的问题，合入的补丁却是：

```c

commit 4cca96a8d9da0ed8217cfdf2aec0c3c8b88e8911

Author: Parav Pandit <parav@mellanox.com>

Date:   Thu Dec 12 13:30:21 2019 +0200

  

diff --git a/drivers/infiniband/hw/mlx5/main.c b/drivers/infiniband/hw/mlx5/main.c

index 997cbfe..05b557d 100644

--- a/drivers/infiniband/hw/mlx5/main.c

+++ b/drivers/infiniband/hw/mlx5/main.c

@@ -6725,6 +6725,8 @@ void __mlx5_ib_remove(struct mlx5_ib_dev *dev,

                      const struct mlx5_ib_profile *profile,

                      int stage)

 {

+       dev->ib_active = false;

+

        /* Number of stages to cleanup */

        while (stage) {

                stage--;

  

```

敲黑板，三遍：

这个合入是不能解决对应的bug的，比如如下的并发：

我们用一个简单的图来表示一下并发处理：

  

```c

    CPU1                                                            CPU2

                                                                   dev_attr_show

    pci_device_shutdown                                            speed_show

      shutdown                          

        mlx5_unload_one

          mlx5_detach_device

            mlx5_detach_interface

              mlx5e_detach

               mlx5e_detach_netdev

                 mlx5e_nic_disable

                   rtnl_lock

                     mlx5e_close_locked 

                     clear_bit(MLX5E_STATE_OPENED, &priv->state);---只清理了这个bit

                   rtnl_unlock                   

                                                  rtnl_trylock---持锁成功后

                                                  netif_running 只是判断net_device.state的最低位

                                                    __ethtool_get_link_ksettings

                                                    mlx5e_get_link_ksettings

                                                      mlx5_query_port_ptys()

                                                      mlx5_core_access_reg()

                                                      mlx5_cmd_exec

                                                      cmd_exec

                                                      mlx5_alloc_cmd_msg

          mlx5_cmd_cleanup---清理dma_pool                                                       

                                                      dma_pool_alloc---访问cmd.pool,触发crash

  

```

所以如果要真正解决这个问题，还需要 netif_device_detach 中清理 __LINK_STATE_START的bit位，

或者在 speed_show 中判断一下 __LINK_STATE_PRESENT 位？如果考虑影响范围，不想动公共流程，则应该

在 mlx5e_get_link_ksettings 中 判断一下 __LINK_STATE_PRESENT。

这个就留给喜欢跟社区打交道的同学去完善吧。

  

```c

static void mlx5e_nic_disable(struct mlx5e_priv *priv)

{

.......

  rtnl_lock();

  if (netif_running(priv->netdev))

    mlx5e_close(priv->netdev);

  netif_device_detach(priv->netdev);

  //caq:增加一下清理 __LINK_STATE_PRESENT位 

  rtnl_unlock();

.......

```

  

#### 三、故障复现

1、竞态问题，可以制造类似上图cpu1 与cpu2 的竞争场景。

  

#### 四、故障规避或解决

  

可能的解决方案是：

  

1、不要按照红帽https://access.redhat.com/solutions/5132931那样升级。

  

2、单独打补丁。

  

#### 五、作者简介

  

陈安庆，目前在oppo混合云负责linux内核及容器，虚拟机等虚拟化方面的工作，

  

联系方式：微信与手机同号：18752035557。

[![](http://www.wowotech.net/content/uploadfile/201605/ef3e1463542768.png)](http://www.wowotech.net/support_us.html)

« [CFS任务的负载均衡（概述）](http://www.wowotech.net/process_management/load_balance.html) | [一个较复杂dcache问题](http://www.wowotech.net/linux_kenrel/484.html)»

**评论：**

**eexplorer**  
2022-08-02 14:54

crash> dma_pool ffff9d0fa45f4c00 -x  
struct dma_pool {  
...  
  pools = {  
    next = 0xdead000000000100,  
    prev = 0xdead000000000200  
  }  
}  
  
从dma_pool.pools这个指针的值，可以判定这个dma_pool已经被destroy了，所以后续去访问这个dma_pool的fields都是不正确的。  
  
比如，page_list.prev的值是指向自己，但是page_list.next不是，所以这个list不是空的。page_list.prev应该是指向最后一个entry的，但是现在确实指向自己，所以这个时候去解析dma_pool里的fields，会得到一些奇怪的结论。  
  
crash> dma_pool ffff9d0fa45f4c00 -x  
struct dma_pool {  
  page_list = {  
    next = 0xffff9d0fa45f4c80,  
    prev = 0xffff9d0fa45f4c00

[回复](http://www.wowotech.net/linux_kenrel/485.html#comment-8661)

**[linuxer](http://www.wowotech.net/)**  
2021-09-01 15:17

等我从绿厂退休就会补齐的，哈哈。

[回复](http://www.wowotech.net/linux_kenrel/485.html#comment-8285)

**大哥您好**  
2021-09-03 17:36

@linuxer：同样期待啊。。。

[回复](http://www.wowotech.net/linux_kenrel/485.html#comment-8289)

**JovenGeek**  
2021-08-27 12:00

linux的时间子系统的七到十一章不打算写了吗。。。。。。

[回复](http://www.wowotech.net/linux_kenrel/485.html#comment-8283)

**发表评论：**

 昵称

 邮件地址 (选填)

 个人主页 (选填)

![](http://www.wowotech.net/include/lib/checkcode.php) 

- ### 站内搜索
    
       
     蜗窝站内  互联网
    
- ### 功能
    
    [留言板  
    ](http://www.wowotech.net/message_board.html)[评论列表  
    ](http://www.wowotech.net/?plugin=commentlist)[支持者列表  
    ](http://www.wowotech.net/support_list)
- ### 最新评论
    
    - ja  
        [@dream：我看完這段也有相同的想法，引用 @dream ...](http://www.wowotech.net/kernel_synchronization/spinlock.html#8922)
    - 元神高手  
        [围观首席power managerment专家](http://www.wowotech.net/pm_subsystem/device_driver_pm.html#8921)
    - 十七  
        [内核空间的映射在系统启动时就已经设定好，并且在所有进程的页表...](http://www.wowotech.net/process_management/context-switch-arch.html#8920)
    - lw  
        [sparse模型和disconti模型没看出来有什么本质区别...](http://www.wowotech.net/memory_management/memory_model.html#8919)
    - 肥饶  
        [一个没设置好就出错](http://www.wowotech.net/linux_kenrel/516.html#8918)
    - orange  
        [点赞点赞，对linuxer的文章总结到位](http://www.wowotech.net/device_model/dt-code-file-struct-parse.html#8917)
- ### 文章分类
    
    - [Linux内核分析(25)](http://www.wowotech.net/sort/linux_kenrel) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=4)
        - [统一设备模型(15)](http://www.wowotech.net/sort/device_model) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=12)
        - [电源管理子系统(43)](http://www.wowotech.net/sort/pm_subsystem) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=13)
        - [中断子系统(15)](http://www.wowotech.net/sort/irq_subsystem) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=14)
        - [进程管理(31)](http://www.wowotech.net/sort/process_management) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=15)
        - [内核同步机制(26)](http://www.wowotech.net/sort/kernel_synchronization) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=16)
        - [GPIO子系统(5)](http://www.wowotech.net/sort/gpio_subsystem) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=17)
        - [时间子系统(14)](http://www.wowotech.net/sort/timer_subsystem) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=18)
        - [通信类协议(7)](http://www.wowotech.net/sort/comm) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=20)
        - [内存管理(31)](http://www.wowotech.net/sort/memory_management) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=21)
        - [图形子系统(2)](http://www.wowotech.net/sort/graphic_subsystem) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=23)
        - [文件系统(5)](http://www.wowotech.net/sort/filesystem) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=26)
        - [TTY子系统(6)](http://www.wowotech.net/sort/tty_framework) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=27)
    - [u-boot分析(3)](http://www.wowotech.net/sort/u-boot) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=25)
    - [Linux应用技巧(13)](http://www.wowotech.net/sort/linux_application) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=3)
    - [软件开发(6)](http://www.wowotech.net/sort/soft) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=1)
    - [基础技术(13)](http://www.wowotech.net/sort/basic_tech) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=6)
        - [蓝牙(16)](http://www.wowotech.net/sort/bluetooth) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=10)
        - [ARMv8A Arch(15)](http://www.wowotech.net/sort/armv8a_arch) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=19)
        - [显示(3)](http://www.wowotech.net/sort/display) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=22)
        - [USB(1)](http://www.wowotech.net/sort/usb) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=28)
    - [基础学科(10)](http://www.wowotech.net/sort/basic_subject) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=7)
    - [技术漫谈(12)](http://www.wowotech.net/sort/tech_discuss) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=8)
    - [项目专区(0)](http://www.wowotech.net/sort/project) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=9)
        - [X Project(28)](http://www.wowotech.net/sort/x_project) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=24)
- ### 随机文章
    
    - [内存初始化代码分析（一）：identity mapping和kernel image mapping](http://www.wowotech.net/memory_management/__create_page_tables_code_analysis.html)
    - [Linux PM QoS framework(3)_per-device PM QoS](http://www.wowotech.net/pm_subsystem/per_device_pm_qos.html)
    - [Linux DMA Engine framework(1)_概述](http://www.wowotech.net/linux_kenrel/dma_engine_overview.html)
    - [CFS调度器（6）-总结](http://www.wowotech.net/process_management/452.html)
    - [linux内核中的GPIO系统之（5）：gpio subsysem和pinctrl subsystem之间的耦合](http://www.wowotech.net/gpio_subsystem/pinctrl-and-gpio.html)
- ### 文章存档
    
    - [2024年2月(1)](http://www.wowotech.net/record/202402)
    - [2023年5月(1)](http://www.wowotech.net/record/202305)
    - [2022年10月(1)](http://www.wowotech.net/record/202210)
    - [2022年8月(1)](http://www.wowotech.net/record/202208)
    - [2022年6月(1)](http://www.wowotech.net/record/202206)
    - [2022年5月(1)](http://www.wowotech.net/record/202205)
    - [2022年4月(2)](http://www.wowotech.net/record/202204)
    - [2022年2月(2)](http://www.wowotech.net/record/202202)
    - [2021年12月(1)](http://www.wowotech.net/record/202112)
    - [2021年11月(5)](http://www.wowotech.net/record/202111)
    - [2021年7月(1)](http://www.wowotech.net/record/202107)
    - [2021年6月(1)](http://www.wowotech.net/record/202106)
    - [2021年5月(3)](http://www.wowotech.net/record/202105)
    - [2020年3月(3)](http://www.wowotech.net/record/202003)
    - [2020年2月(2)](http://www.wowotech.net/record/202002)
    - [2020年1月(3)](http://www.wowotech.net/record/202001)
    - [2019年12月(3)](http://www.wowotech.net/record/201912)
    - [2019年5月(4)](http://www.wowotech.net/record/201905)
    - [2019年3月(1)](http://www.wowotech.net/record/201903)
    - [2019年1月(3)](http://www.wowotech.net/record/201901)
    - [2018年12月(2)](http://www.wowotech.net/record/201812)
    - [2018年11月(1)](http://www.wowotech.net/record/201811)
    - [2018年10月(2)](http://www.wowotech.net/record/201810)
    - [2018年8月(1)](http://www.wowotech.net/record/201808)
    - [2018年6月(1)](http://www.wowotech.net/record/201806)
    - [2018年5月(1)](http://www.wowotech.net/record/201805)
    - [2018年4月(7)](http://www.wowotech.net/record/201804)
    - [2018年2月(4)](http://www.wowotech.net/record/201802)
    - [2018年1月(5)](http://www.wowotech.net/record/201801)
    - [2017年12月(2)](http://www.wowotech.net/record/201712)
    - [2017年11月(2)](http://www.wowotech.net/record/201711)
    - [2017年10月(1)](http://www.wowotech.net/record/201710)
    - [2017年9月(5)](http://www.wowotech.net/record/201709)
    - [2017年8月(4)](http://www.wowotech.net/record/201708)
    - [2017年7月(4)](http://www.wowotech.net/record/201707)
    - [2017年6月(3)](http://www.wowotech.net/record/201706)
    - [2017年5月(3)](http://www.wowotech.net/record/201705)
    - [2017年4月(1)](http://www.wowotech.net/record/201704)
    - [2017年3月(8)](http://www.wowotech.net/record/201703)
    - [2017年2月(6)](http://www.wowotech.net/record/201702)
    - [2017年1月(5)](http://www.wowotech.net/record/201701)
    - [2016年12月(6)](http://www.wowotech.net/record/201612)
    - [2016年11月(11)](http://www.wowotech.net/record/201611)
    - [2016年10月(9)](http://www.wowotech.net/record/201610)
    - [2016年9月(6)](http://www.wowotech.net/record/201609)
    - [2016年8月(9)](http://www.wowotech.net/record/201608)
    - [2016年7月(5)](http://www.wowotech.net/record/201607)
    - [2016年6月(8)](http://www.wowotech.net/record/201606)
    - [2016年5月(8)](http://www.wowotech.net/record/201605)
    - [2016年4月(7)](http://www.wowotech.net/record/201604)
    - [2016年3月(5)](http://www.wowotech.net/record/201603)
    - [2016年2月(5)](http://www.wowotech.net/record/201602)
    - [2016年1月(6)](http://www.wowotech.net/record/201601)
    - [2015年12月(6)](http://www.wowotech.net/record/201512)
    - [2015年11月(9)](http://www.wowotech.net/record/201511)
    - [2015年10月(9)](http://www.wowotech.net/record/201510)
    - [2015年9月(4)](http://www.wowotech.net/record/201509)
    - [2015年8月(3)](http://www.wowotech.net/record/201508)
    - [2015年7月(7)](http://www.wowotech.net/record/201507)
    - [2015年6月(3)](http://www.wowotech.net/record/201506)
    - [2015年5月(6)](http://www.wowotech.net/record/201505)
    - [2015年4月(9)](http://www.wowotech.net/record/201504)
    - [2015年3月(9)](http://www.wowotech.net/record/201503)
    - [2015年2月(6)](http://www.wowotech.net/record/201502)
    - [2015年1月(6)](http://www.wowotech.net/record/201501)
    - [2014年12月(17)](http://www.wowotech.net/record/201412)
    - [2014年11月(8)](http://www.wowotech.net/record/201411)
    - [2014年10月(9)](http://www.wowotech.net/record/201410)
    - [2014年9月(7)](http://www.wowotech.net/record/201409)
    - [2014年8月(12)](http://www.wowotech.net/record/201408)
    - [2014年7月(6)](http://www.wowotech.net/record/201407)
    - [2014年6月(6)](http://www.wowotech.net/record/201406)
    - [2014年5月(9)](http://www.wowotech.net/record/201405)
    - [2014年4月(9)](http://www.wowotech.net/record/201404)
    - [2014年3月(7)](http://www.wowotech.net/record/201403)
    - [2014年2月(3)](http://www.wowotech.net/record/201402)
    - [2014年1月(4)](http://www.wowotech.net/record/201401)

[![订阅Rss](http://www.wowotech.net/content/templates/default/images/rss.gif)](http://www.wowotech.net/rss.php "RSS订阅")

Copyright @ 2013-2015 [蜗窝科技](http://www.wowotech.net/ "wowotech") All rights reserved. Powered by [emlog](http://www.emlog.net/ "采用emlog系统")