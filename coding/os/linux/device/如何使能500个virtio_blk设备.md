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

作者：[安庆](http://www.wowotech.net/author/539 "oppo混合云内核&虚拟化负责人，架构并孵化了oppo的云游戏，云手机等产品。") 发布于：2022-8-12 16:12 分类：[Linux内核分析](http://www.wowotech.net/sort/linux_kenrel)

# 一例virtio_blk设备中断占用分析

## 背景：这个是在客户的centos8.4的环境上复现的，dpu是目前很多

## 云服务器上的网卡标配了，在云豹的dpu产品测试中，dpu实现的virtio_blk

## 设备在申请中断时报错，在排查这个错误的过程中，觉得某些部分还比较有

## 趣，故记录之。本身涉及的背景知识有：irq,msi,irq_domain,

## affinity，virtio_blk，irqbalance

### 下面列一下我们是怎么排查并解决这个问题的。

  

#### 一、故障现象

内核团队接到测试组测试客户前端内核抛栈：

```

[25338.485128] virtio-pci 0000:b3:00.0: virtio_pci: leaving for legacy driver  
[25338.496174] **genirq: Flags mismatch irq 0. 00000080 (virtio418) vs. 00015a00 (timer)**  
[25338.503822] CPU: 20 PID: 5431 Comm: kworker/20:0 Kdump: loaded Tainted: G           OE    --------- -  - 4.18.0-305.30.1.jmnd2.el8.x86_64 #1  
[25338.516403] Hardware name: Inspur NF5280M5/YZMB-00882-10E, BIOS 4.1.21 08/25/2021  
[25338.523881] Workqueue: events work_for_cpu_fn  
[25338.528235] Call Trace:  
[25338.530687]  dump_stack+0x5c/0x80  
[25338.534000]  __setup_irq.cold.53+0x7c/0xd3  
[25338.538098]  request_threaded_irq+0xf5/0x160  
[25338.542371]  vp_find_vqs+0xc7/0x190  
[25338.545866]  init_vq+0x17c/0x2e0 [virtio_blk]  
[25338.550223]  ? ncpus_cmp_func+0x10/0x10  
[25338.554061]  virtblk_probe+0xe6/0x8a0 [virtio_blk]  
[25338.558846]  virtio_dev_probe+0x158/0x1f0  
[25338.562861]  really_probe+0x255/0x4a0  
[25338.566524]  ? __driver_attach_async_helper+0x90/0x90  
[25338.571567]  driver_probe_device+0x49/0xc0  
[25338.575660]  bus_for_each_drv+0x79/0xc0  
[25338.579499]  __device_attach+0xdc/0x160  
[25338.583337]  bus_probe_device+0x9d/0xb0  
[25338.587167]  device_add+0x418/0x780  
[25338.590654]  register_virtio_device+0x9e/0xe0  
[25338.595011]  virtio_pci_probe+0xb3/0x140  
[25338.598941]  local_pci_probe+0x41/0x90  
[25338.602689]  work_for_cpu_fn+0x16/0x20  
[25338.606443]  process_one_work+0x1a7/0x360  
[25338.610456]  ? create_worker+0x1a0/0x1a0  
[25338.614381]  worker_thread+0x1cf/0x390  
[25338.618132]  ? create_worker+0x1a0/0x1a0  
[25338.622051]  kthread+0x116/0x130  
[25338.625283]  ? kthread_flush_work_fn+0x10/0x10  
[25338.629731]  ret_from_fork+0x1f/0x40  
[25338.633395] virtio_blk: probe of virtio418 failed with error -16  

```

从堆栈看，是某个virtio_blk设备在probe的时候报错，错误码为-16。

#### 二、故障现象分析

  

从堆栈信息看：

  

1、virtio418是一个virtio_blk设备，在probe过程中调用 __setup_irq 返回了-16。

  

2、[25338.496174] genirq: Flags mismatch irq 0. 00000080 (virtio418) vs. 00015a00 (timer)，说明我们的virtio_blk

设备去申请了0号中断，由于0号中断被timer占用，irq子系统在比较flags时发现不符合，则打印这行。

具体代码为：

```

static int

__setup_irq(unsigned int irq, struct irq_desc *desc, struct irqaction *new)  
{

......

mismatch:  
        if (!(new->flags & IRQF_PROBE_SHARED)) {  
                pr_err("Flags mismatch irq %d. %08x (%s) vs. %08x (%s)\n",  
                       irq, new->flags, new->name, old->flags, old->name);

......

       ret = -EBUSY;

......

      return ret;

}

```

至于为什么virtio_blk会去申请0号中断，因为我们实现virtio_blk后端设备的时候，并没有支持intx，即virtio_pci类虚拟的pci_dev设备的irq值为0，本文先不管这个。

从堆栈看，virtio 申请中断是走了vp_find_vqs_intx流程，

```

crash> dis -l vp_find_vqs+0xc7  
/usr/src/debug/linux/drivers/virtio/virtio_pci_common.c: **369---行号为369**

  

    356 static int vp_find_vqs_intx(struct virtio_device *vdev, unsigned nvqs,  
    357                 struct virtqueue *vqs[], vq_callback_t *callbacks[],  
    358                 const char * const names[], const bool *ctx)  
    359 {  
 ......  
    366  
    367         err = request_irq(vp_dev->pci_dev->irq, vp_interrupt, IRQF_SHARED,  
    368                         dev_name(&vdev->dev), vp_dev);

    **369         if (err)----压栈的返回地址**

......

```

我们dpu卡实现的virtio设备，都是使能msix的，按照代码流程,应该是先尝试msix，既然能走到 vp_find_vqs_intx 流程，说明 vp_find_vqs_msix失败了，而且按照如下代码：

```

    395 int vp_find_vqs(struct virtio_device *vdev, unsigned nvqs,  
    396                 struct virtqueue *vqs[], vq_callback_t *callbacks[],  
    397                 const char * const names[], const bool *ctx,  
    398                 struct irq_affinity *desc)  
    399 {  
    400         int err;  
    401  
    402         /* Try MSI-X with one vector per queue. */  
    403         err = vp_find_vqs_msix(vdev, nvqs, vqs, callbacks, names, **true**, ctx, desc);//caq:先尝试单vqueue单中断号  
    404         if (!err)  
    405                 return 0;  
    406         /* Fallback: MSI-X with one vector for config, one shared for queues. */  
    407         err = vp_find_vqs_msix(vdev, nvqs, vqs, callbacks, names, **false**, ctx, desc);//caq:尝试多个vq共享一个中断号  
    408         if (!err)  
    409                 return 0;  
    410         /* Finally fall back to regular interrupts. */  
    411         return vp_find_vqs_intx(vdev, nvqs, vqs, callbacks, names, ctx);//caq:最后退化成intx模式  
    412 }

```

说明vp_find_vqs_msix **失败了两次**。第一次是 单vqueue单中断号的非共享方式，第二次是多个vq共用一个中断的方式。

通过打点发现，失败两次的原因是 __irq_domain_activate_irq  返回了-28

```

__irq_domain_activate_irq return=-28

0xffffffff8fb52f70 : __irq_domain_activate_irq+0x0/0x80 [kernel]  
 0xffffffff8fb54bc5 : irq_domain_activate_irq+0x25/0x40 [kernel]  
 0xffffffff8fb56bfe : msi_domain_alloc_irqs+0x15e/0x2f0 [kernel]  
 0xffffffff8fa5e5e4 : native_setup_msi_irqs+0x54/0x90 [kernel]  
 0xffffffff8feef69f : __pci_enable_msix_range+0x3df/0x5e0 [kernel]  
 0xffffffff8feef96b : pci_alloc_irq_vectors_affinity+0xbb/0x130 [kernel]  
 0xffffffff8ff7472b : vp_find_vqs_msix+0x1fb/0x510 [kernel]  
 0xffffffff8ff74aad : vp_find_vqs+0x6d/0x190 [kernel]  
  

```

查看具体的代码：

```

static int __irq_domain_activate_irq(struct irq_data *irqd, bool reserve)  
{  
int ret = 0;  
  
if (irqd && irqd->domain) {//caq:均不为NULL  
struct irq_domain *domain = irqd->domain;  
  
if (irqd->parent_data)  
ret = __irq_domain_activate_irq(irqd->parent_data,  
reserve);//caq:递归，将父domain的对应irq都active一下  
if (!ret && domain->ops->activate) {  
ret = domain->ops->activate(domain, irqd, reserve);//caq:parent 执行完activate 然后再儿子辈执行。  
/* Rollback in case of error */  
if (ret && irqd->parent_data)//caq:异常则回退  
__irq_domain_deactivate_irq(irqd->parent_data);  
}  
}  
return ret;  
}

```

由于客户 host kernel开启了 CONFIG_IRQ_DOMAIN_HIERARCHY，根据irq_domain 级别 ，该系统的irq_domain 级联如下：

![](data:image/png;base64,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)

不过上图是arm常见的，盗用arm图，本x86系统类似，irq_domain级别具体跟踪如下：

```

crash> irq_domain.name,parent 0xffff9bff87d4dec0  
  name = 0xffff9bff87c1fd60 "INTEL-IR-MSI-1-2"  
  parent = 0xffff9bff87400000  
crash> irq_domain.name,parent 0xffff9bff87400000  
  name = 0xffff9bff87c24300 "INTEL-IR-1"  
  parent = 0xffff9bff87c6c900  
crash> irq_domain.name,parent 0xffff9bff87c6c900  
  name = 0xffff9bff87c3ecd0 "VECTOR"-----------最高级的  
  parent = 0x0---所以parent为空

```

根据返回-28，根据最高级的irq_domain定位到 调用链为：

```

//caq:类比于 dma_domain_ops，在x86内是最高级的irq_domain了，因为他的domain parent为NULL  
static const struct irq_domain_ops x86_vector_domain_ops = {//caq:x86针对acpi实现的irq_domain_ops  
.alloc = x86_vector_alloc_irqs,//caq:分配中断  
.free = x86_vector_free_irqs,  
.activate = x86_vector_activate,//caq:activate实现  
.deactivate = x86_vector_deactivate,  
#ifdef CONFIG_GENERIC_IRQ_DEBUGFS  
.debug_show = x86_vector_debug_show,  
#endif  
};  
调用链：

x86_vector_activate-->activate_managed-->assign_managed_vector-->irq_matrix_alloc_managed

```

查看 代码如下：

```

int irq_matrix_alloc_managed(struct irq_matrix *m, const struct cpumask *msk,  
     unsigned int *mapped_cpu)  
{//caq:managed irq 分配  
unsigned int bit, cpu, end = m->alloc_end;  
struct cpumap *cm;  
  
if (cpumask_empty(msk))  
return -EINVAL;  
  
cpu = matrix_find_best_cpu_managed(m, msk);//caq:找最合适的cpu  
if (cpu == UINT_MAX)  
return -ENOSPC;//caq:说明没找到  
......  
}

```

由于没有开启 CONFIG_GENERIC_IRQ_DEBUGFS，所以没办法直接看到 vector_matrix 具体的值，

借助crash工具查看：

```

crash> p *vector_matrix  
$82 = {  
  matrix_bits = 256,  
  alloc_start = 32,  
  alloc_end = 236,  
  alloc_size = 204,  
  **g****lobal_available = 0**,//caq:只剩下了这么多个irq  
  global_reserved = 151,  
  systembits_inalloc = 3,  
  **total_allocated = 1922,**//caq:只分配了这么多个irq  
  online_maps = 80,  
  maps = 0x2ff20,  
  scratch_map = {18446744069952503807, 18446744073709551615, 18446744073709551615, 18446735277616529407, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0},  
  system_map = {1125904739729407, 0, 1, 18446726481523507200, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0}  
}

```

一个疑问涌上心头，为什么总共才分配了1922 个中断，全局的global_available就为0了呢？

然后继续打点，发现virtio_blk设备的vq申请中断时，走的是 apicd->is_managed 流程，而同属于virtio协议的virtio_net设备却不是。

而走managed流程，也就是申请中断时，带有了RQD_AFFINITY_MANAGED：

```

static int  
assign_irq_vector_policy(struct irq_data *irqd, struct irq_alloc_info *info)  
{  
if (irqd_affinity_is_managed(irqd))//caq:如果是 managed 的irq,也就是irq_data中有 IRQD_AFFINITY_MANAGED 标记  
return reserve_managed_vector(irqd);

```

我们回过来查看vector alloc时的调用链：

x86_vector_alloc_irqs-->assign_irq_vector_policy-->reserve_managed_vector-->irq_matrix_reserve_managed

对一个两个队列的virtio_blk申请中断时，打点发现如下：

```

m->global_available=15296 

0xffffffff87158300 : irq_matrix_reserve_managed+0x0/0x130 [kernel]---从15296减少到15256  
m->global_available=15256

call vdev=0xffff8b781ce17000,index=0,callback=0xffffffffc0448000,ctx=0,msix_vec=1----------容量减少了40  
```

由于已经缩小到是因为virtio_blk设备的中断申请流程，使用热插拔确认一下：

```

118:          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0         53          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0  IR-PCI-MSI 94371841-edge      virtio3-req.0  
 119:          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0         49          0          0          0  IR-PCI-MSI 94371842-edge      virtio3-req.1  
  
热拔前：  
crash> p *vector_matrix  
$2 = {  
  matrix_bits = 256,  
  alloc_start = 32,  
  alloc_end = 236,  
  alloc_size = 204,  
  **global_available =** **15215**,  
  global_reserved = 150,  
  systembits_inalloc = 3,  
  total_allocated = 553,  
  online_maps = 80,  
  maps = 0x2ff20,  
  scratch_map = {1179746449752063, 0, 1, 18446726481523507200, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0},  
  system_map = {1125904739729407, 0, 1, 18446726481523507200, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0}  
}  
  
热拔后：  
crash> p *vector_matrix  
$3 = {  
  matrix_bits = 256,  
  alloc_start = 32,  
  alloc_end = 236,  
  alloc_size = 204,  
  **global_available = 15296**,---增长了81个，一个config中断+两个分别为40的req中断，此时req.0和req.1是非共享模式  
  global_reserved = 150,  
  systembits_inalloc = 3,  
  total_allocated = 550,  
  online_maps = 80,  
  maps = 0x2ff20,  
  scratch_map = {481036337152, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0},  
  system_map = {1125904739729407, 0, 1, 18446726481523507200, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0}  
}

```

说明 对于mask内的cpu号，都需要一个中断容量，两个40就是因为80核的服务器，两个vq，平分中断，感兴趣的同学可以去查看irq_build_affinity_masks这个函数的实现。这样一个vector，因为开启了 IRQD_AFFINITY_MANAGED 属性，导致需要占用80个中断容量。而我们系统由于有512个virtio_blk设备，所以申请到部分设备的时候，就把总的

vector容量耗光了，但其实分配的**irq总数才不到2000**.

  

那么，virtio_blk设备什么时候开启的 IRQD_AFFINITY_MANAGED  属性的呢？查看git记录：

```

6abd6e5a44040 (Amit Shah                 2011-12-22 16:58:29 +0530  507) static int init_vq(struct virtio_blk *vblk)  
6abd6e5a44040 (Amit Shah                 2011-12-22 16:58:29 +0530  508) {  
......  
6a27b656fc021 (Ming Lei                  2014-06-26 17:41:48 +0800  515)        struct virtio_device *vdev = vblk->vdev;  
ad71473d9c437 (Christoph Hellwig         2017-02-05 18:15:25 +0100  516)        struct irq_affinity desc = { 0, };----会导致blk申请中断时，使用内核managed方式来申请，一个dev会占用cpu核数这么多的容量。

```

看起来是 ad71473d9c437  这个 commit引入了这个问题。

  

但是根据virtio_blk驱动，第一遍中断申请的时候才有affinity_managed 设置，第二遍应该并没有设置，具体 vp_find_vqs_msix 如下：

```

//caq:vq申请中断，msix 模式,per_vq_vectors 决定是个vq共享中断还是独占中断  
static int vp_find_vqs_msix(struct virtio_device *vdev, unsigned nvqs,  
struct virtqueue *vqs[], vq_callback_t *callbacks[],  
const char * const names[], bool per_vq_vectors,  
const bool *ctx,  
struct irq_affinity *desc)  
{  
struct virtio_pci_device *vp_dev = to_vp_device(vdev);  
u16 msix_vec;  
int i, err, nvectors, allocated_vectors;  
  
vp_dev->vqs = kcalloc(nvqs, sizeof(*vp_dev->vqs), GFP_KERNEL);  
if (!vp_dev->vqs)  
return -ENOMEM;  
  
if (per_vq_vectors) {//caq:单个vq 单个vector 中断号  
/* Best option: one for change interrupt, one per vq. */  
nvectors = 1;//caq:这个是config的中断,注意要和virtio_net的 ctrl vq区分  
for (i = 0; i < nvqs; ++i)  
if (callbacks[i])//caq:由于ctrl vq是不设置callbacks的，所以它没有中断  
++nvectors;  
} else {  
/* Second best: one for change, shared for all vqs. */  
nvectors = 2;  
}  
    //caq:中断总数最少为2，最多为vq个数+1,1为config的中断,另外单个vq单个vector才具备带亲核属性  
err = vp_request_msix_vectors(vdev, nvectors, per_vq_vectors,  
      **per_vq_vectors ? desc : NULL**);//caq:**nvectors 为总中断数，注意desc的配置取决于 per_vq_vectors**

//caq:virtio_pci申请msix的vector  
static int vp_request_msix_vectors(struct virtio_device *vdev, int nvectors,  
   bool per_vq_vectors, struct irq_affinity *desc)  
{  
......  
vp_dev->msix_affinity_masks  
= kcalloc(nvectors, sizeof(*vp_dev->msix_affinity_masks),  
  GFP_KERNEL);//caq:中断掩码内存的分配  
......  
if (desc) {//caq:要求带亲核属性  
flags |= PCI_IRQ_AFFINITY;//caq:带上亲核属性  
desc->pre_vectors++; /* virtio config vector */**//caq:细节，相当于指定了config中断不要设置亲核，走系统默认**  
}  
......

```

原因，因为前面很多virtio_blk设备因为一个vector占用了80个中断容量，导致整体中断数不够了，

而后面的virtio_blk设备，第一遍使用vp_find_vqs_msix带 managed_affinity属性申请中断时失败，第二遍尽管使用vq共享中断模式，由于os连一个中断都分配不出来，  也会失败，导致走入了第三个流程，也就是 vp_find_vqs_intx 流程。

在另外一个virtio_blk单vq的环境上，具体查看如下：

```

[root@localhost config_json]#  cat /proc/interrupts |grep req |tail -1  
 986:          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0        114          0          0  IR-PCI-MSI 93687809-edge      virtio180-req.0  
[root@localhost config_json]# cat /proc/irq/986/smp_affinity  
ffff,ffffffff,ffffffff  

[root@localhost config_json]#  cat /proc/interrupts |grep queues |tail -1  
1650:          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0        120          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0          0  IR-PCI-MSI 94369793-edge      virtio512-virtqueues

  

[root@localhost config_json]# cat /proc/irq/1650/smp_affinity  
ffff,ffffffff,ffffffff  

[root@localhost config_json]# ls /sys/bus/virtio/devices/virtio180/block/  
vdsh  
[root@localhost config_json]# ls /sys/bus/virtio/devices/virtio512/block/  
vdafj

```

以上可以看出，virtio180，也就是 vdsh 的block设备，走了 vp_find_vqs_msix 第一遍流程，分配了带 managed_affinity 的vector，所以他的中断名字是req结尾的，

  

而另外一个 virtio512 ，也就是 vdafj 的block设备，走了 vp_find_vqs_msix 第二遍流程，没有分配 带 managed_affinity 的vector，所以它的中断名字是 virtqueues 结尾的，

而后面的设备，只能走第三个流程，报错了，所以打点发现，除了 activate的时候会报容量不足，在alloc阶段，也会在 irq_matrix_alloc 报容量不足。

```

int irq_matrix_alloc(struct irq_matrix *m, const struct cpumask *msk,  
     bool reserved, unsigned int *mapped_cpu)  
{

......  
cpu = matrix_find_best_cpu(m, msk);  
if (cpu == UINT_MAX)//caq:在msk内，没找到合适的cpu来记账  
return -ENOSPC;  
  
cm = per_cpu_ptr(m->maps, cpu);  
bit = matrix_alloc_area(m, cm, 1, false);//caq:获取一个  
if (bit >= m->alloc_end)  
return -ENOSPC;//caq:没有资源了  
......  

打点记录如下：

Returning from:  0xffffffffb7158300 : irq_matrix_reserve_managed+0x0/0x130 [kernel]  
Returning to  :  0xffffffffb705adb7 : x86_vector_alloc_irqs+0x2d7/0x3c0 [kernel]  
 0xffffffffb75dc42c : intel_irq_remapping_alloc+0x7c/0x6d0 [kernel]  
 0xffffffffb7156438 : msi_domain_alloc+0x68/0x120 [kernel]  
 0xffffffffb715457d : __irq_domain_alloc_irqs+0x12d/0x300 [kernel]  
 0xffffffffb7156b38 : msi_domain_alloc_irqs+0x98/0x2f0 [kernel]  
 0xffffffffb705e5e4 : native_setup_msi_irqs+0x54/0x90 [kernel]  
 0xffffffffb74ef69f : __pci_enable_msix_range+0x3df/0x5e0 [kernel]  
 0xffffffffb74ef96b : pci_alloc_irq_vectors_affinity+0xbb/0x130 [kernel]  
 0xffffffffb70635e0 : kretprobe_trampoline+0x0/0x50 [kernel]  
 0x1b7574a40  
 0x1b7574a40 (inexact)  
**irq_matrix_reserve_managed return -28**  

```

#### 三、故障复现

1、只要是一开始各个核中断的aviable 容量相当，然后热插拔足够多virtio_blk设备，则**必现**。

2、如果各个核的中断的available容量相差很多，比如常见的numa节点的第一个cpu的中断占用过多，

使得走第一分支时因为某个核容量不够而reserve_managed 失败，然后

则会使得后面大量的virtio_blk走第二个分支，此时不带managed_affinity，**反而能分配成功。**

  

#### 四、故障规避或解决

  

可能的解决方案之一：

```

static int init_vq(struct virtio_blk *vblk)//caq:初始化关于vq相关的内容  
{  
int err;  
int i;  
vq_callback_t **callbacks;  
const char **names;  
struct virtqueue **vqs;  
unsigned short num_vqs;  
struct virtio_device *vdev = vblk->vdev;  
**struct irq_affinity desc = { 0, }**;//caq:去掉这行代码

```

解决方案之二：

开启irqbalance,并让服务器进入 Power-save mode 时，irqbalance 会将中断集中分配给numa节点的第一个 CPU，这样慢慢地，各个核

的available的irq 容量就相差比较大了，当然这种不太靠谱。

解决方案之三：

手工调整中断亲核，使得某些核的容量接近于0，然后再加载virtio_blk设备。

  

#### 五、作者简介

  

陈安庆，目前在dpu厂商 云豹智能 负责linux内核及虚拟化方面的工作，

  

联系方式：微信与手机同号：18752035557。

[![](http://www.wowotech.net/content/uploadfile/201605/ef3e1463542768.png)](http://www.wowotech.net/support_us.html)

« [futex基础问答](http://www.wowotech.net/kernel_synchronization/futex.html) | [Linux内核同步机制之（九）：Queued spinlock](http://www.wowotech.net/kernel_synchronization/queued_spinlock.html)»

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
    
    - [Linux cpufreq framework(2)_cpufreq driver](http://www.wowotech.net/pm_subsystem/cpufreq_driver.html)
    - [eMMC 原理 1 ：Flash Memory 简介](http://www.wowotech.net/basic_tech/flash_memory_intro.html)
    - [一个技术男眼中的Smartisan T1手机](http://www.wowotech.net/tech_discuss/112.html)
    - [支持wowo，有这样一块小天地享受技术，感觉棒棒哒！](http://www.wowotech.net/147.html)
    - [u-boot启动流程分析(1)_平台相关部分](http://www.wowotech.net/u-boot/boot_flow_1.html)
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