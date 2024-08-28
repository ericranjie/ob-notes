# [蜗窝科技](http://www.wowotech.net/)

### 慢下来，享受技术。

[![](http://www.wowotech.net/content/uploadfile/201401/top-1389777175.jpg)](http://www.wowotech.net/)

- [博客](http://www.wowotech.net/)
- [项目](http://www.wowotech.net/sort/project)
- [关于蜗窝](http://www.wowotech.net/about.html)
- [联系我们](http://www.wowotech.net/contact_us.html)
- [支持与合作](http://www.wowotech.net/support_us.html)
- [登录](http://www.wowotech.net/admin)

﻿

## 

作者：[安庆](http://www.wowotech.net/author/539 "oppo混合云内核&虚拟化负责人，架构并孵化了oppo的云游戏，云手机等产品。") 发布于：2020-3-30 20:54 分类：[Linux内核分析](http://www.wowotech.net/sort/linux_kenrel)

一、前言

本文以一个在centos7.6内核发生的BUGON,描述一下常见BUGON导致panic的解bug流程。目前这个bug在centos7.6最新版本未合入，需要升级到centos7.7的3.10.0-1062.13.1.el7以上版本才能解决。文中涉及到iscsi模块，xfs模块，内核协议栈收发包，slab(slub)等内存结构信息。本文对内核初学者分析和理解常见的内核问题有一定的帮助。

二、故障现象

机器出现复位，收集到了crash文件，从日志中显示为BUGON。

[85774.558604] usercopy: kernel memory exposure attempt detected from ffff9cba0bf75400 (kmalloc-512) (1024 bytes)

[85774.559261] ------------[ cut here ]------------

[85774.559839] kernel BUG at mm/usercopy.c:72!

[85774.560367] invalid opcode: 0000 [#1] SMP

[85774.560879] Modules linked in: cmac arc4 md4 nls_utf8 cifs ccm dns_resolver xfs iscsi_tcp libiscsi_tcp libiscsi iptable_raw iptable_mangle sch_sfq sch_htb scsi_transport_iscsi veth ipt_MASQUERADE nf_nat_masquerade_ipv4 xt_comment xt_mark iptable_nat nf_conntrack_ipv4 nf_defrag_ipv4 nf_nat_ipv4 xt_addrtype iptable_filter xt_conntrack nf_nat nf_conntrack br_netfilter bridge stp llc dm_thin_pool dm_persistent_data dm_bio_prison dm_bufio libcrc32c loop bonding fuse sunrpc dm_mirror dm_region_hash dm_log dm_mod dell_smbios dell_wmi_descriptor iTCO_wdt iTCO_vendor_support dcdbas skx_edac intel_powerclamp coretemp intel_rapl iosf_mbi kvm_intel kvm irqbypass crc32_pclmul ghash_clmulni_intel aesni_intel lrw gf128mul glue_helper ablk_helper cryptd ipmi_ssif sg pcspkr mei_me lpc_ich i2c_i801 mei wmi ipmi_si

[85774.564343]  ipmi_devintf ipmi_msghandler acpi_pad acpi_power_meter ip_tables ext4 mbcache jbd2 sd_mod crc_t10dif crct10dif_generic crct10dif_pclmul crct10dif_common crc32c_intel mgag200 drm_kms_helper syscopyarea sysfillrect sysimgblt fb_sys_fops ttm megaraid_sas drm ixgbe ahci igb drm_panel_orientation_quirks libahci mdio libata ptp pps_core dca i2c_algo_bit nfit libnvdimm

[85774.566809] CPU: 9 PID: 28054 Comm: tgtd Kdump: loaded **Not tainted** 3.10.0-957.27.2.el7.x86_64 #1

[85774.567446] Hardware name: Dell Inc. PowerEdge R740/0YNX56, BIOS 2.4.8 11/26/2019

[85774.568094] task: ffff9cb12e1e0000 ti: ffff9cb124224000 task.ti: ffff9cb124224000

[85774.568754] RIP: 0010:[<ffffffff9803f557>]  [<ffffffff9803f557>] __check_object_size+0x87/0x250

[85774.569419] RSP: 0018:ffff9cb124227b98  EFLAGS: 00010246

[85774.570072] RAX: 0000000000000062 RBX: ffff9cba0bf75400 RCX: 0000000000000000

[85774.570723] RDX: 0000000000000000 RSI: ffff9cc13bf13898 RDI: ffff9cc13bf13898

[85774.571372] RBP: ffff9cb124227bb8 R08: 0000000000000000 R09: ffff9cb1313e6f00

[85774.572017] R10: 000000000003bc95 R11: 0000000000000001 R12: 0000000000000400

[85774.572669] R13: 0000000000000001 R14: ffff9cba0bf75800 R15: 0000000000000400

[85774.573325] FS:  00007f41a122a740(0000) GS:ffff9cc13bf00000(0000) knlGS:0000000000000000

[85774.573994] CS:  0010 DS: 0000 ES: 0000 CR0: 0000000080050033

[85774.574655] CR2: 0000000003236fe0 CR3: 0000001023138000 CR4: 00000000007607e0

[85774.575314] DR0: 0000000000000000 DR1: 0000000000000000 DR2: 0000000000000000

[85774.575964] DR3: 0000000000000000 DR6: 00000000fffe0ff0 DR7: 0000000000000400

[85774.576609] PKRU: 55555554

[85774.577242] Call Trace:

[85774.577880]  [<ffffffff9818dd9d>] memcpy_toiovec+0x4d/0xb0

[85774.578531]  [<ffffffff9842c858>] skb_copy_datagram_iovec+0x128/0x280

[85774.579190]  [<ffffffff9849372a>] tcp_recvmsg+0x22a/0xb30

[85774.579838]  [<ffffffff984c2340>] inet_recvmsg+0x80/0xb0

[85774.580474]  [<ffffffff9841a6ec>] sock_aio_read.part.9+0x14c/0x170

[85774.581097]  [<ffffffff9841a731>] sock_aio_read+0x21/0x30

[85774.581714]  [<ffffffff98041b33>] do_sync_read+0x93/0xe0

[85774.582328]  [<ffffffff98042615>] vfs_read+0x145/0x170

[85774.582934]  [<ffffffff9804342f>] SyS_read+0x7f/0xf0

[85774.583543]  [<ffffffff98576ddb>] system_call_fastpath+0x22/0x27

[85774.584158] Code: 45 d1 48 c7 c6 05 c3 87 98 48 c7 c1 f6 57 88 98 48 0f 45 f1 49 89 c0 4d 89 e1 48 89 d9 48 c7 c7 00 27 88 98 31 c0 e8 30 e4 51 00 <0f> 0b 0f 1f 80 00 00 00 00 48 c7 c0 00 00 e0 97 4c 39 f0 73 0d

[85774.585495] RIP  [<ffffffff9803f557>] __check_object_size+0x87/0x250

[85774.586135]  RSP <ffff9cb124227b98>

三、分析过程

1、可能存在问题的原因

我们知道，在使能了CONFIG_HARDENED_USERCOPY的情况下，如果往用户态拷贝或者用户态往内核拷贝的话，会对访问的数据区进行检查，这里不展开，感兴趣的同学可以参照一下《[https://lwn.net/Articles/695991/](https://lwn.net/Articles/695991/)》来理解这种配置引入的背景。

首先，我们来看一下对应BUGON的地方：

     61 static void report_usercopy(const void *ptr, unsigned long len,

     62                             bool to_user, const char *type)

     63 {

     64         pr_emerg("kernel memory %s attempt detected %s %p (%s) (%lu bytes)\n",

     65                 to_user ? "exposure" : "overwrite",

     66                 to_user ? "from" : "to", ptr, type ? : "unknown", len);

     67         /*

     68          * For greater effect, it would be nice to do do_group_exit(),

     69          * but BUG() actually hooks all the lock-breaking and per-arch

     70          * Oops code, so that is used here instead.

     71          */

     **72         BUG();----这行出现**

     73 }

从上面代码可以看出，我们报错日志的红色字体的第一行，就是pr_emerg打印出来的。在本案例中，有几个关键字需要特别关注：

[85774.558604] usercopy: kernel memory **exposure** attempt detected **from ffff9cba0bf75400** (**kmalloc-512)** (**1024 bytes**)

意思是，我们试图从ffff9cba0bf75400 拷贝到用户态，那么出现bugon，要么是地址问题，要么是长度问题，我们下面就要判断出，到底是地址问题还是长度问题。

回到我们的堆栈，堆栈表示tgtd 进程在收tcp的网络包。

[85774.577880]  [<ffffffff9818dd9d>] memcpy_toiovec+0x4d/0xb0

[85774.578531]  [<ffffffff9842c858>] skb_copy_datagram_iovec+0x128/0x280

[85774.579190]  [<ffffffff9849372a>] tcp_recvmsg+0x22a/0xb30

[85774.579838]  [<ffffffff984c2340>] inet_recvmsg+0x80/0xb0

[85774.580474]  [<ffffffff9841a6ec>] sock_aio_read.part.9+0x14c/0x170

[85774.581097]  [<ffffffff9841a731>] sock_aio_read+0x21/0x30

[85774.581714]  [<ffffffff98041b33>] do_sync_read+0x93/0xe0

[85774.582328]  [<ffffffff98042615>] vfs_read+0x145/0x170

[85774.582934]  [<ffffffff9804342f>] SyS_read+0x7f/0xf0

[85774.583543]  [<ffffffff98576ddb>] system_call_fastpath+0x22/0x27

我们知道，收包的时候，我们从sock对象的sk_receive_queue中把存放数据的skb取出来，然后拷贝其中的数据到用户态。skb_copy_datagram_iovec的skb参数是压栈的，所以在堆栈中取出的skb如下：

**crash> sk_buff.len ffff9cb0e3b388f8**

  **len = 1024**

crash> sk_buff.data_len  ffff9cb0e3b388f8

  data_len = 1024

所以：int start = skb_headlen(skb); start为0

crash> sk_buff.head  ffff9cb0e3b388f8

  head = 0xffff9cbf9c679400 ""

crash> sk_buff.end  ffff9cb0e3b388f8 -x

  end = 0x2c0

所以，sk_buff这个结构在线性区之后，还有skb_shared_info 结构：

crash> px  0xffff9cbf9c679400 + 0x2c0

$5 = 0xffff9cbf9c6796c0

crash> skb_shared_info 0xffff9cbf9c6796c0

struct skb_shared_info {

  nr_frags = 1 '\001', ----只有一个frag,这个值确定了从skb_frag_t的拷贝次数

查看对应的page：

crash> skb_shared_info.frags 0xffff9cbf9c6796c0

  frags = {{

      page = {

        p = **0xfffff597a42fdd40**----本堆栈中的page

      },

      page_offset = 1024,

      size = 1024

}, {

。。。。。。。数组其他部分省略。。。。。。

好的，下面我们结合bug报错的地址来看一下对应page的使用：

crash> kmem ffff9cba0bf75400 ---报错的地址

CACHE            NAME                 OBJSIZE  ALLOCATED     TOTAL  SLABS  SSIZE

ffff9ca27fc07600 kmalloc-512              512     242454    464512   7258    32k

  SLAB              MEMORY            NODE  TOTAL  ALLOCATED  FREE

  fffff597a42fdc00  ffff9cba0bf70000     1     64         59     5

  FREE / [ALLOCATED]

  [ffff9cba0bf75400]

      PAGE         PHYSICAL      MAPPING       INDEX CNT FLAGS

**fffff597a42fdd40** 190bf75000                0        0  **0** 6fffff00008000 tail

可以看到，关联的page指针确实和我们skb中保存数据的page指针能对应上的。

一开始，当我看到page的引用计数为0还愣住了一下，以为是个踩内存的问题，然后我看到tail标志，想起来这个页是一个复合页，所以这个非head的page没有slab的标记，同时引用计数为0就是合理的了，因为它的slab标志和引用计数都是放在head页面里的，到这一步，说明地址不是非法访问，**只是长度问题**。

0xffff9cba0bf75400 对应的是一个kmalloc-512 的slab，在未开启slab（slub）的debug的情况下，对应的size应该是512字节，而（1024 bytes），就是我们本次需要访问的长度，对于hardened usercopy的检查来说，是越界了，也就是在 copy_to_user-->check_heap_object-->__check_heap_object中返回了kmem_cache的name，那么确定是因为访问长度问题。

接下来，我们需要解决的疑问是，为什么会访问一个512字节的object的时候，长度却是1024字节。

我们知道，如果从普通网卡收包，在skb分配的时候，除了线性区部分的内存，frag部分的内存应该都是按页分配的，哪怕修改过标准内核，这个page都应该是一个整页，而不应该是一个512字节的slab，所以要么这个数据被踩了，要么就是这个page不是从普通网卡收包而来，而是类似lo口的这种内部循环，不过，在收包到tcp的阶段，skb对应的dev指针已经被置为NULL了，我们无法通过这个指针来辨别对应的入向dev，所以我尝试从socket本身入手，看看这个是一个什么样的socket，

crash> sk_buff.sk ffff9cb0e3b388f8

  sk = 0xffff9cc1242a8000---------skb对应的sock指针

使用net -s查看，可以看到详细的端口信息如下：

66 ffff9cc11d941e00 ffff9cc1242a8000 INET:STREAM  127.0.0.1-3260 127.0.0.1-41654

可以看到，由于我们的socket是侦听在127的环回地址，端口为3260，所以这个page，还真是发送方传来的Page，而不是我们自己收包的时候申请的page。我们先查看一下对应512字节部分的数据是什么数据：

crash> rd ffff9cba0bf75400 8

ffff9cba0bf75400:  0100000046474158 0000e80301000000   XAGF............

ffff9cba0bf75410:  0200000001000000 0100000000000000   ................

ffff9cba0bf75420:  0000000001000000 0300000000000000   ................

ffff9cba0bf75430:  fc62a00304000000 00000000f0ceba02   ......b.........

看起来像是一个xfs的agf的block，由于对xfs也不是很熟悉，看代码只是怀疑有点像，

#define XFS_AGF_MAGIC   0x58414746  /* 'XAGF' */

#define XFS_AGI_MAGIC   0x58414749  /* 'XAGI' */

#define XFS_AGFL_MAGIC  0x5841464c  /* 'XAFL' */

然后开始有目的地在社区搜索相关xfs关键字，以及 usercopy 关键字相关的patch，一开始搜到下面这个：

https://bugs.centos.org/view.php?id=15859#bugnotes

出错的行号都一样，但我们目前内核已经合入ceph这个补丁，所以应该不是这个问题。再次陷入迷茫。。。

然后在搜索的过程中，另外一名同学报了一个类似问题上来，也是同样的堆栈，同样的进程，于是我开始尝试了解tgtd对应的组件，原来它就是iscsi的target端的组件，那么xfs用它来传送AGF的元数据就能够说得通了，换句话说，ceph和iscsi的地位在这种场景下是类似的。这个时候开始搜索iscsi相关的补丁，找到了对应的补丁如下：

commit 08b11eaccfcf86a3bac6755625d933ac15ccc27a

Author: Vasily Averin <vvs@virtuozzo.com>

Date:   Thu Feb 21 18:23:17 2019 +0300

    scsi: libiscsi: fall back to sendmsg for slab pages

    In "XFS over network block device" scenario XFS can create IO requests with

    slab-based XFS metadata. During processing such requests tcp_sendpage() can

    merge skb fragments with neighbour slab objects.

    If receiving side is located on the same host tcp_recvmsg() can trigger

    BUG_ON in hardening check and crash the host with following message:

    usercopy: kernel memory exposure attempt detected

                    from XXXXXXXX (kmalloc-512) (1024 bytes)

    This patch redirect such requests from sednpage to sendmsg path.  The

    problem is similar to one described in recent commit 7e241f647dc7

    ("libceph: fall back to sendmsg for slab pages")

    Signed-off-by: Vasily Averin <vvs@virtuozzo.com>

    Acked-by: Chris Leech <cleech@redhat.com>

    Signed-off-by: Martin K. Petersen <martin.petersen@oracle.com>

原来，这个1024的长度，是tcp_sendpage将用户前后两次调用的时候，做了一个检查，如果发送的是同一个page，并且偏移能够衔接上，也就是skb_can_coalesce 函数的实现逻辑，则会将对应的发送合并，然后长度进行叠加，也就是说我们的1024长度，是两个xfs的512字节的slab叠加而成，所以既不是地址有问题，也不是长度有问题，而是叠加之后的page在开启hardened_usercopy=on的机器上报错而已。

**但是，敲黑板，同学们请注意，它的改动是如下实现的：**

-       if (page_count(sg_page(sg)) >= 1 && !recv)

+       if (!recv && page_count(sg_page(sg)) >= 1 && !PageSlab(sg_page(sg)))

这个改动**表面上看**是无法解决组合页的问题，只能解决单页的情况。因为compound的page需要取它的head来判断slab标志，就像本案例中遇到的情况那样，真正的改动应该是下面那样：

-       if (page_count(sg_page(sg)) >= 1 && !recv)

+       if (!recv && page_count(sg_page(sg)) >= 1    && !PageSlab(compound_head(sg_page(sg))))

于是，我尝试给负责这块的社区大神来提交一个如上的补丁，不过Ilya Dryomov 很快回复如下：

> >AFAICT compound pages should already be taken into account, because PageSlab is defined as:

> >

> >  __PAGEFLAG(Slab, slab, PF_NO_TAIL)

> >

> >  #define __PAGEFLAG(uname, lname, policy)                       \

> >      TESTPAGEFLAG(uname, lname, policy)                         \

> >      __SETPAGEFLAG(uname, lname, policy)                        \

> >      __CLEARPAGEFLAG(uname, lname, policy)

> >

> >  #define TESTPAGEFLAG(uname, lname, policy)                     \

> >  static __always_inline int Page##uname(struct page *page)      \

> >      { return test_bit(PG_##lname, &policy(page, 0)->flags); }

> >

> > and PF_NO_TAIL policy is defined as:

>

> >  #define PF_NO_TAIL(page, enforce) ({                        \

> >      VM_BUG_ON_PGFLAGS(enforce && PageTail(page), page);     \

> >      PF_POISONED_CHECK(compound_head(page)); })

>

> > So compound_head() is called behind the scenes.

也就是说，虽然3.10的内核，需要这么改动，这个对应patch是对的，但是最新的内核，由于PageSlab的含义已经包含了compound页的考虑了，所以不需要再用compound去判断了。

四、结论和收获

从补丁来说，这个补丁是从libceph那边关联到xfs也有类似的问题，不过分析的过程还是很有趣的，如果使用iscsi同时底层文件系统又是xfs的，需要注意反向合入了，或者如果嫌麻烦的话，升级到centos7.7内核也可以。这个故障动作最小的情况下怎么解决呢，由于iscsi的lib是一个linux模块形式，可以将对应的代码合入，替换线上的模块即可。由于 hardened_usercopy 开启是会影响服务器性能的，所以在grub中设置 hardened_usercopy=off 也是一种解决办法，但是这时候需要复位机器生效了，并且对于越界拷贝的情况就无法提前暴露了，2016年linux内核引入hardened_usercopy功能之后，帮助很多模块定位了很多问题。详细研究了一下最开始遇到这个问题的时候，社区提交补丁的交流历史记录，发现一开始有人想在发包合并的地方进行判断，也就是如下改动：

> @@ -3089,7 +3089,7 @@ static inline bool skb_can_coalesce(struct sk_buff *skb, int i,

>   if (i) {

>   const struct skb_frag_struct *frag = &skb_shinfo(skb)->frags[i - 1];

>  

> - return page == skb_frag_page(frag) &&

> + return page == skb_frag_page(frag) && !PageSlab(page) &&

>          off == frag->page_offset + skb_frag_size(frag);

>   }

但是David 大神很敏锐地判断出，这种改法有问题，他的原文回复是：

No, this would be wrong.

There is no way a page fragment can be backed by slab object,

since a page fragment can be shared (the page refcount needs to be manipulated, without slab/slub

being aware of this)

**Please fix the callers.****---------最关键的一句**

是的啊，这样改动应该是最合理的，将问题掐死在源头。具体可以参考[https://www.spinics.net/lists/netdev/msg552971.html](https://www.spinics.net/lists/netdev/msg552971.html)

不过呢，也有仁兄不死心，尝试在 do_tcp_sendpages 中对于can_coalesce做一些警告：

diff --git a/net/ipv4/tcp.c b/net/ipv4/tcp.c

index 2079145a3b7c..cf9572f4fc0f 100644

--- a/net/ipv4/tcp.c

+++ b/net/ipv4/tcp.c

@@ -996,6 +996,7 @@ ssize_t do_tcp_sendpages(struct sock *sk, struct page *page, int offset,

  goto wait_for_memory;

  if (can_coalesce) {

+ WARN_ON_ONCE(PageSlab(page));

  skb_frag_size_add(&skb_shinfo(skb)->frags[i - 1], copy);

  } else {

  get_page(page);

不过这个不死心的改动最终并没能合入到基线中去，具体可参考：

[https://lore.kernel.org/netdev/a8655149-80b9-c75d-6528-0b851ea85de8@virtuozzo.com/](https://lore.kernel.org/netdev/a8655149-80b9-c75d-6528-0b851ea85de8@virtuozzo.com/)

[![](http://www.wowotech.net/content/uploadfile/201605/ef3e1463542768.png)](http://www.wowotech.net/support_us.html)

« [一例ext4使用的jbd2模块死锁分析](http://www.wowotech.net/linux_kenrel/481.html) | [一例centos7.6内核hardlock的解析](http://www.wowotech.net/linux_kenrel/479.html)»

**评论：**

**nswcfd@cu**  
2021-05-10 14:28

又重新学习了一遍，过了一年差不多都忘干净啦 :(  
  
您的文章介绍的很清晰，前因后果讲的很清楚，再次点赞

[回复](http://www.wowotech.net/linux_kenrel/480.html#comment-8236)

**[爱芭虎](http://www.aibahu.com/)**  
2020-10-28 21:28

分析的很详细，很用心的文章了。

[回复](http://www.wowotech.net/linux_kenrel/480.html#comment-8131)

**[安庆](http://www.wowotech.net/)**  
2021-05-08 14:31

@爱芭虎：才看到，谢谢

[回复](http://www.wowotech.net/linux_kenrel/480.html#comment-8229)

**[太仓招聘网](http://https//www.tczpw.com)**  
2020-05-30 10:54

很详细！花了很多时间看完

[回复](http://www.wowotech.net/linux_kenrel/480.html#comment-8004)

**[安庆](http://www.wowotech.net/)**  
2021-05-08 14:32

@太仓招聘网：嗯，篇幅比较长。

[回复](http://www.wowotech.net/linux_kenrel/480.html#comment-8230)

**[安庆](http://www.wowotech.net/)**  
2021-05-08 14:32

@太仓招聘网：嗯，篇幅比较长。

[回复](http://www.wowotech.net/linux_kenrel/480.html#comment-8231)

**[安庆](http://www.wowotech.net/)**  
2021-05-08 14:32

@太仓招聘网：嗯，篇幅比较长。

[回复](http://www.wowotech.net/linux_kenrel/480.html#comment-8232)

**nswcfd@cu**  
2020-04-02 10:30

非常好的分析！

[回复](http://www.wowotech.net/linux_kenrel/480.html#comment-7935)

**[安庆](http://www.wowotech.net/)**  
2021-05-08 14:32

@nswcfd@cu：嗯，篇幅比较长。

[回复](http://www.wowotech.net/linux_kenrel/480.html#comment-8233)

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
    
    - [Linux CPU core的电源管理(1)_概述](http://www.wowotech.net/pm_subsystem/cpu_core_pm_overview.html)
    - [CFS调度器（5）-带宽控制](http://www.wowotech.net/process_management/451.html)
    - [RCU（2）- 使用方法](http://www.wowotech.net/kernel_synchronization/462.html)
    - [ARM64的启动过程之（一）：内核第一个脚印](http://www.wowotech.net/armv8a_arch/arm64_initialize_1.html)
    - [Linux TTY framework(4)_TTY driver](http://www.wowotech.net/tty_framework/tty_driver.html)
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