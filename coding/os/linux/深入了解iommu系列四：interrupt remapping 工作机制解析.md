# 

Linux阅码场

 _2022年03月17日 08:00_

以下文章来源于技不辱你 ，作者技不辱你

[

![](http://wx.qlogo.cn/mmhead/Q3auHgzwzM6X3nOSmeyKczpEcnpyiazdQYGVoQaALq7sOIp186qibo5Q/0)

**技不辱你**.

一位虚拟化and软硬一体化从业者的技术随笔和工作感悟。

](https://mp.weixin.qq.com/s?__biz=Mzg2OTc0ODAzMw==&mid=2247502890&idx=1&sn=334155d2f2454c3dfff07d40997eef76&source=41&key=daf9bdc5abc4e8d0c749c904211e41464adb984e6e34b57a3db3703075b07d44650eb7637be7a746d7d2a50471b77fe643f7a9d672d74df651c75a12869816200be94d90a01997a75cbe5977b1323dbca3cafefbd84a9e509b60f4683605b89ac614f3df340fd0416cf876fb7813cd4a2d44d8f87dc143426bb17ae44f952648&ascene=0&uin=MTEwNTU1MjgwMw%3D%3D&devicetype=Windows+11+x64&version=63090b19&lang=zh_CN&countrycode=CN&exportkey=n_ChQIAhIQ3P1YGhX1Yvtbx42NKdmfZBLmAQIE97dBBAEAAAAAAIrxOcUCW0cAAAAOpnltbLcz9gKNyK89dVj0hhCOl5PvDwe%2Fr6w%2FtoB62OLZgmBFT8R7SXYZ%2F2xYBMkjSnKO%2FmmAT1ICdgaiemR4u%2BJ1DGjTPrRRFFq4hgTodgPyQXYayOvAF0I2ed5xmKg7iWyKSUuTHyiB44y8IV1RJY9YDp1T5dZq0ZMqfvauUrFD8KMyacFfqdKkE1XRS06NNCUR%2F7RehOFAHxtAYlGo5Ds0FuOkg6ZiJb4vzUtoP9qSIJjoHIoaUj6%2BhGckUyGy7SJtr2MOeYlC6aj3o1XE&acctmode=0&pass_ticket=f%2FP0CGsAgFcc2n%2Bpdt%2BcVhlZhUULQAFh2llzkWAJ0ABuSTwCDI2Pnm89fRW5sxkI&wx_header=1#)

### 序言

系列三主要分析了interrupt remaping底层硬件的架构设计和工作原理，同时也简要地描述了一下interrupt remapping在操作系统层面的初始化流程。那么这一篇我们来仔细分析一下它到底是如何工作的。

### 工作机制

### 非虚拟化场景

在非虚拟化场景下当设备去enable msix中断的时候，其调用路径是这样的

`pci_enable_msix_range->__pci_enable_msix_range->__pci_enable_msix   ->msix_capability_init->pci_msi_setup_msi_irqs   `

这里对pci_msi_setup_msi_irqs进行一下简单的介绍

`static int pci_msi_setup_msi_irqs(struct pci_dev *dev, int nvec, int type)   {           struct irq_domain *domain;              domain = dev_get_msi_domain(&dev->dev);           if (domain && irq_domain_is_hierarchy(domain))                   return msi_domain_alloc_irqs(domain, &dev->dev, nvec);              return arch_setup_msi_irqs(dev, nvec, type);   }   `

通常都会default走arch_setup_msi_irqs，除了在deviceTree里面对设备的irq_domain进行了显示的绑定则会直接走msi_domain_alloc_irqs。接着往下看

`pci_msi_setup_msi_irqs->arch_setup_msi_irqs->native_setup_msi_irqs   `

`int native_setup_msi_irqs(struct pci_dev *dev, int nvec, int type)   {           struct irq_domain *domain;           struct irq_alloc_info info;              init_irq_alloc_info(&info, NULL);           info.type = X86_IRQ_ALLOC_TYPE_MSI;           info.msi_dev = dev;     //最终调到intel_get_irq_domain,返回ir_msi_domain           domain = irq_remapping_get_irq_domain(&info);           if (domain == NULL)                   domain = msi_default_domain;           if (domain == NULL)                   return -ENOSYS;              return msi_domain_alloc_irqs(domain, &dev->dev, nvec);   }   `

我们先看一下`msi_domain_alloc_irqs` 的调用链，然后再一一分析相关函数

 `msi_domain_alloc_irqs     ->msi_domain_prepare_irqs     ->__irq_domain_alloc_irqs     ->msi_check_reservation_mode     ....   //省略部分主要是有些分支判断这里先不list出来，后面涉及到的时候会具体介绍`

- msi_domain_prepare_irqs 调用`pci_msi_prepare` 函数做一些初始化的动作设置arg的type，msi_dev, cpumask等
    
- 接着遍历这个设备msi_list当中的所有的msi_desc，在调用`__irq_domain_alloc_irqs` 为其分配irq之前先调用`pci_msi_domain_set_desc->pci_msi_domain_calc_hwirq` 为每个msi_desc分配一个全局唯一的初始ID即hwirq(关于它的作用我们后面再说)，接着我们来看一下__irq_domain_alloc_irqs的核心逻辑。我们先看一下这个核心函数的基本调用链，然后我们再对相关的函数一一分析
    
      `__irq_domain_alloc_irqs      ->irq_domain_alloc_descs      ->irq_domain_alloc_irq_data      ->irq_domain_alloc_irqs_hierarchy      ->irq_domain_insert_irq`
    
            `for (i = 0; i < nr_irqs; i++) {               irq_data = irq_get_irq_data(virq + i);               irq_data->domain = domain;                  for (parent = domain->parent; parent; parent = parent->parent) {                       irq_data = irq_domain_insert_irq_data(parent, irq_data);                       if (!irq_data) {                               irq_domain_free_irq_data(virq, i + 1);                               return -ENOMEM;                       }               }       }`
    

- 首先调用 `irq_domain_alloc_descs` 分配irq_desc，其核心逻辑就是从 allocated_irqs这个bitmap里面分配未被使用的irq。
    
- 接着调用irq_domain_alloc_irq_data，为每个irq的irq_data->domain以及其为所属domain的父亲辈domain，爷爷辈domain 的irq_data作相关数据初始化，逻辑如下：
    

`static struct irq_data *irq_domain_insert_irq_data(struct irq_domain *domain,                                                      struct irq_data *child)   {           struct irq_data *irq_data;              irq_data = kzalloc_node(sizeof(*irq_data), GFP_KERNEL,                                   irq_data_get_node(child));           if (irq_data) {                   child->parent_data = irq_data;                   irq_data->irq = child->irq;                   irq_data->common = child->common;                   irq_data->domain = domain;           }              return irq_data;   }   `

- 然后调用`irq_domain_alloc_irqs_hierarchy`函数，具体逻辑如下：
    

`int irq_domain_alloc_irqs_hierarchy(struct irq_domain *domain,                                       unsigned int irq_base,                                       unsigned int nr_irqs, void *arg)   {           return domain->ops->alloc(domain, irq_base, nr_irqs, arg);   }   `

注意这里的domain为ir_msi_domain，其irq_domain_ops如下

`static const struct irq_domain_ops msi_domain_ops = {           .alloc          = msi_domain_alloc,           .free           = msi_domain_free,           .activate       = msi_domain_activate,           .deactivate     = msi_domain_deactivate,   };   `

所以其调用的alloc的callback为msi_domain_alloc函数，我们再来看看其具体实现：

- irq_find_mapping(domain, hwirq)，首先通过上面提到的hwirq来判断是否已经在该domain已经进行映射。
    
- irq_domain_alloc_irqs_parent，先将它parent domain的irq也分配出来。由于ir_msi_domain的父domain为ir_domain其irq_domain_os为`inel_ir_domain_ops`其alloc为`intel_irq_remapping_alloc` 这个函数为中断remapping功能的核心所在，因此我们来仔细看一下这个函数：
    

- 首先还是继续调用irq_domain_alloc_irqs_parent，而ir_domain的父domain为x86_vector_domain。而`x86_vector_domain`的`alloc` callback函数的核心逻辑就是给相应的irq分配相关cpu lapic 上的中断vector，如果你在alloc_irq的时候指定了irq affinity的cpu_mask则直接从这些cpu的lapic当中分配；如果你没有指定则在系统所有的cpu当中找到合适的cpu lapic然后只是先reserve vector等到irq active的时候才真正去分配。这里面有三个概念一个是irq，一个是hwirq，还有一个就是vector，大家可能有点晕他们之间到底是什么关系？系统通过irq来找到irq_desc和irq_data，hwirq和irq在一个中断domain里面是一一映射的关系，然后vector是lapic最终能识别的东西也是irte表里面的最终要写入的中断number。
    
- 接着调用alloc_irte函数去分配irte，其核心逻辑就是从上文所说的bitmap当中找到nvec个可用的interrupt remapping entry，同时返回开始位置即初始index
    
- 接着为每个irq准备irte表即设置subhandle, index，还有就是上文提到irte中的其他field，最后就是设置msi or msi-x信息格式。
    

- 回到`msi_domain_alloc`函数，接着通过一个loop对irq所对应的irq_data相关信息做初始化
    

        `for (i = 0; i < nr_irqs; i++) {             //msi_init为msi_domain_ops_init                   ret = ops->msi_init(domain, info, virq + i, hwirq + i, arg);                   if (ret < 0) {                           if (ops->msi_free) {                                   for (i--; i > 0; i--)                                           ops->msi_free(domain, info, virq + i);                           }                           irq_domain_free_irqs_top(domain, virq, nr_irqs);                           return ret;                   }           }`

这里的`msi_domain_ops_init` 主要调用的函数如下

`msi_domain_ops_init    -> irq_domain_set_hwirq_and_chip //设置irq_data当中的hwirq，chip, chip_data    ->__irq_set_handler     //如果msi_domain_info初始化了handler则进行设置    ->irq_set_handler_data //如果msi_domain_info初始化了handler_data则进行设置          handler and hanadler_data的具体情况如下    ================================       static struct msi_domain_info pci_msi_ir_domain_info = {           .flags          = MSI_FLAG_USE_DEF_DOM_OPS | MSI_FLAG_USE_DEF_CHIP_OPS |                             MSI_FLAG_MULTI_PCI_MSI | MSI_FLAG_PCI_MSIX,           .ops            = &pci_msi_domain_ops,           .chip           = &pci_msi_ir_controller,           .handler        = handle_edge_irq,           .handler_name   = "edge",   };` 

- 分析完`msi_domain_alloc` ，至此`irq_domain_alloc_irqs_hierarchy`的逻辑基本看完了，之后接着往下看。
    

        `for (i = 0; i < nr_irqs; i++)                   irq_domain_insert_irq(virq + i);`

`irq_domain_insert_irq`的核心逻辑就是完成virq跟hwirq在不同层级的irq domain中的mapping。

上面把`__irq_domain_alloc_irqs` 的逻辑分析完了，回到`msi_domain_alloc_irqs` 继续往下走

`can_reserve = msi_check_reservation_mode(domain, info, dev);      ========      static bool msi_check_reservation_mode(struct irq_domain *domain,                                          struct msi_domain_info *info,                                          struct device *dev)   {           struct msi_desc *desc;     //bus_token通常情况下都是DOMAIN_BUS_PCI_MSI           if (domain->bus_token != DOMAIN_BUS_PCI_MSI)                   return false;     //如果配置了CONFI_GENERIC_IRQ_RESERVATION_MODE的话这个flag是一定会被置上的           if (!(info->flags & MSI_FLAG_MUST_REACTIVATE))                   return false;     //CONFIG_PCI_MSI 默认enable，然后pci_mis_ignore_mask从代码上看没有被置1           if (IS_ENABLED(CONFIG_PCI_MSI) && pci_msi_ignore_mask)                   return false;              /*            * Checking the first MSI descriptor is sufficient. MSIX supports            * masking and MSI does so when the maskbit is set.            */           desc = first_msi_entry(dev);     //通常情况下新设备一般都是MSIX           return desc->msi_attrib.is_msix || desc->msi_attrib.maskbit;   }   `

接着往下看

        `for_each_msi_entry(desc, dev) {                   virq = desc->irq;                   if (desc->nvec_used == 1)                           dev_dbg(dev, "irq %d for MSI\n", virq);                   else                           dev_dbg(dev, "irq [%d-%d] for MSI\n",                                   virq, virq + desc->nvec_used - 1);                   /*                    * This flag is set by the PCI layer as we need to activate                    * the MSI entries before the PCI layer enables MSI in the                    * card. Otherwise the card latches a random msi message.                    */        //在创建ir_msi_domain的时候这个flag会被置上                   if (!(info->flags & MSI_FLAG_ACTIVATE_EARLY))                           continue;                      irq_data = irq_domain_get_irq_data(domain, desc->irq);                   if (!can_reserve)                           irqd_clr_can_reserve(irq_data);                   ret = irq_domain_activate_irq(irq_data, can_reserve);                   if (ret)                           goto cleanup;           }`

上面的核心逻辑在`irq_domain_activate_irq(irq_data, can_reserve)` ，它最终会调到函数`__irq_domain_activate_irq` 它的具体逻辑如下：

`static int __irq_domain_activate_irq(struct irq_data *irqd, bool reserve)   {           int ret = 0;              if (irqd && irqd->domain) {                   struct irq_domain *domain = irqd->domain;                      if (irqd->parent_data)                           ret = __irq_domain_activate_irq(irqd->parent_data,                                                           reserve);                   if (!ret && domain->ops->activate) {                           ret = domain->ops->activate(domain, irqd, reserve);                           /* Rollback in case of error */                           if (ret && irqd->parent_data)                                   __irq_domain_deactivate_irq(irqd->parent_data);                   }           }           return ret;   }   `

从实现可以看出它使用了递归逻辑优先调用最上层的domain的activate函数，从上篇文章的梳理可以知道系统当中整个domain的有关系如下：

`x86_vector_domain ---- 父亲         |      |      ir_domain   -------- 儿子         |      |      ir_msi_domain -------孙子      `

我们先看一下x86_vector_domain的activate实现

`MANAGED_IRQ_SHUTDOWN_VECTOR      static int x86_vector_activate(struct irq_domain *dom, struct irq_data *irqd,                                  bool reserve)   {           struct apic_chip_data *apicd = apic_chip_data(irqd);           unsigned long flags;           int ret = 0;              trace_vector_activate(irqd->irq, apicd->is_managed,                                 apicd->can_reserve, reserve);              /* Nothing to do for fixed assigned vectors */           if (!apicd->can_reserve && !apicd->is_managed)                   return 0;              raw_spin_lock_irqsave(&vector_lock, flags);           if (reserve || irqd_is_managed_and_shutdown(irqd))                   vector_assign_managed_shutdown(irqd);           ........   }      `

从上面的实现来看当reserve为true的时候，通过vector_assign_managed_shutdown为每个irq设置vector为`MANAGED_IRQ_SHUTDOWN_VECTOR` 即并没有分配实际的vector。接下来我们看一下ir_domain的activate函数

`static int intel_irq_remapping_activate(struct irq_domain *domain,                                           struct irq_data *irq_data, bool reserve)   {           intel_ir_reconfigure_irte(irq_data, true);           return 0;   }   `

核心实现主要是由`intel_ir_reconfigure_irte`来实现，其主要把在alloc irq阶段创建的irte sync到ir_table里面。

最后看一下ir_msi_domain的activate函数，具体如下

`static int msi_domain_activate(struct irq_domain *domain,                                  struct irq_data *irq_data, bool early)   {           struct msi_msg msg[2] = { [1] = { }, };              BUG_ON(irq_chip_compose_msi_msg(irq_data, msg));           msi_check_level(irq_data->domain, msg);           irq_chip_write_msi_msg(irq_data, msg);           return 0;   }      `

核心就是通过irq_chip_wirte_msi_msg将msi or msix msg写入到pcie设备的msi cap 的vector table里面。

                `writel(msg->address_lo, base + PCI_MSIX_ENTRY_LOWER_ADDR);                   writel(msg->address_hi, base + PCI_MSIX_ENTRY_UPPER_ADDR);                   writel(msg->data, base + PCI_MSIX_ENTRY_DATA);`

至此已经完成了irq的整个分配流程，上面还有一个遗留的问题那就是每个irq所对应的vector并没有真正去分配，那具体什么时候去分配的呢？答案就是在request_irq的时候去分配的，同时会再次调用每个domain的activate函数。

#### 虚拟化场景

我们先来看一下在设备直通场景该设备的中断是如何初始化的，具体的实现是从vfio_realize当中的`vfio_msix_early_setup`函数开始的。**这个函数里面首先会去读初始化msix的table_bar, pba_bar, entries等信息，然后接着调用`vfio_pci_fixup_msix_region`对msix table 所在的bar mmap region进行修剪。这里背后的逻辑是说直通设备的mmio访问(即对bar的访问)除了第一次之外，后面都是按照内存通过EPT处理的即不再退出。但是misx中断处理这一块有些特殊，比如在guest里面我们可能修改网卡中断的绑核信息那这个时候vmm这一侧就必须要感知道因为要同步去修改irte的内容，所以这个函数的主要作用就是把msi_table相关的区域从region里面去掉，然后通过纯模拟的方式去实现guest对msix table的相关访问**。交代完这个背景之后我们接着往下看

`vfio_add_capabilities    ->vfio_msix_setup->vfio_msix_setup->msix_init   `

`msix_init` 函数做的事情主要有在设备的pci配置空间添加msix cap，完善msix_table, msix pba在qemu侧模拟配置空间的位置信息，将所有的vector mask以及设置msix_table_mmio_ops和msix_pba_mmio_ops，同时把所有的中断vector mask掉。上面把qemu这一侧直通设备msix初始化的流程整理完了，后面的流程就是vm启动之后在相关驱动当中通过`__pci_enable_msix->msix_capability_init`去申请中断vector了，具体的逻辑可以参考上面非虚拟化场景的相关逻辑(区别在里msix msg的格式为compatibility format)。这里我们重点关注一下申请中断的过程中与后端是如何交互并且成功使能的。

`static int msix_capability_init(struct pci_dev *dev, struct msix_entry *entries,                                   int nvec, const struct irq_affinity *affd)   {           int ret;           u16 control;           void __iomem *base;              ...... //略去              ret = pci_msi_setup_msi_irqs(dev, nvec, PCI_CAP_ID_MSIX);           if (ret)                   goto out_avail;              /* Check if all MSI entries honor device restrictions */           ret = msi_verify_entries(dev);           if (ret)                   goto out_free;              /*            * Some devices require MSI-X to be enabled before we can touch the            * MSI-X registers.  We need to mask all the vectors to prevent            * interrupts coming in before they're fully set up.            */           pci_msix_clear_and_set_ctrl(dev, 0,                                   PCI_MSIX_FLAGS_MASKALL | PCI_MSIX_FLAGS_ENABLE);              msix_program_entries(dev, entries);          ...... //略去          pci_msix_clear_and_set_ctrl(dev, PCI_MSIX_FLAGS_MASKALL, 0);          ......  //略去      }      `

上面是msix_capability_init的核心逻辑，其中申请中断的核心函数pci_msi_setup_msi_irqs里面跟后端需要进行交互的关键部分就是把msix msg写到后端即写msix table。那我们可以看一下qemu侧的相关实现

`static void msix_table_mmio_write(void *opaque, hwaddr addr,                                     uint64_t val, unsigned size)   {       PCIDevice *dev = opaque;       int vector = addr / PCI_MSIX_ENTRY_SIZE;       bool was_masked;          was_masked = msix_is_masked(dev, vector);       pci_set_long(dev->msix_table + addr, val);       msix_handle_mask_update(dev, vector, was_masked);   }   `

`static void msix_handle_mask_update(PCIDevice *dev, int vector, bool was_masked)   {       bool is_masked = msix_is_masked(dev, vector);    //由于此时vector还处于masked状态，所以直接返回。       if (is_masked == was_masked) {           return;       }          msix_fire_vector_notifier(dev, vector, is_masked);          if (!is_masked && msix_is_pending(dev, vector)) {           msix_clr_pending(dev, vector);           msix_notify(dev, vector);       }   }   `

注意由于这个时刻vector还是mask状态，所以只会更新后端msix_table的信息。后下面的逻辑当中要与后端进行交互的逻辑如下：

        `/*            * Some devices require MSI-X to be enabled before we can touch the            * MSI-X registers.  We need to mask all the vectors to prevent            * interrupts coming in before they're fully set up.            */           pci_msix_clear_and_set_ctrl(dev, 0,                                   PCI_MSIX_FLAGS_MASKALL | PCI_MSIX_FLAGS_ENABLE);              msix_program_entries(dev, entries);          ...... //略去          pci_msix_clear_and_set_ctrl(dev, PCI_MSIX_FLAGS_MASKALL, 0);`  

上面的逻辑当中`pci_msix_clear_and_set_ctrl` 这个函数主要是写设备的config_space相对应的后端处理函数为`vfio_pci_write_config`下面截取一下其核心的实现进行相关的分析：

`......   else if (pdev->cap_present & QEMU_PCI_CAP_MSIX &&           ranges_overlap(addr, len, pdev->msix_cap, MSIX_CAP_LENGTH)) {           int is_enabled, was_enabled = msix_enabled(pdev);              pci_default_write_config(pdev, addr, val, len);              is_enabled = msix_enabled(pdev);              if (!was_enabled && is_enabled) {               vfio_msix_enable(vdev);           } else if (was_enabled && !is_enabled) {               vfio_msix_disable(vdev);           }        ......      `

因为设置了`PCI_MSIX_FLAGS_ENABLE` 所以在qemu这一端会走到vfio_msix_enable的逻辑，这个函数主要做的事情就是在vfio这一层enable vector 0中的中断到qemu侧另外就是为这个设备的`msix_vector_use_notifier` 和`msix_vector_release_notifier`。接着再回到guest里面的逻辑，`msix_program_entries` 会将所有的中断vector mask掉，然后再clear掉msix ctr flag当中的`PCI_MSIX_FLAGS_MASKALL` 位。讲到这里可能很多人就会问vector到底是在什么时候unmask的呢？答案就是在guest里面调用request_irq的时候：

`request_irq->request_threaded_irq->__setup_irq->irq_startup->__irq_startup->unmask_irq->pci_msi_unmask_irq   `

上面的逻辑我就不细讲了，大家如果有兴趣可以自己去翻阅一下相关的代码。我们着重来看一下qemu这一侧的相关逻辑。由于要unmask vector所以要写msix_table mmio，后端对应的具体ops函数如下

`msix_table_mmio_write ->msix_fire_vector_notifier   `

`static void msix_fire_vector_notifier(PCIDevice *dev,                                         unsigned int vector, bool is_masked)   {       MSIMessage msg;       int ret;          if (!dev->msix_vector_use_notifier) {           return;       }       if (is_masked) {           dev->msix_vector_release_notifier(dev, vector);       } else {     //由于是unmask，所以具体的执行逻辑如下           msg = msix_get_message(dev, vector);           ret = dev->msix_vector_use_notifier(dev, vector, msg);           assert(ret >= 0);       }   }   `

`msix_vector_use_notifier` 调用的callback为`vfio_msix_vector_use`其最终调用的函数为`vfio_msix_vector_do_use` 。下面我们着重看一下这个函数的具体逻辑是什么

- 首先为该vector的interrupt EventNotifier初始化eventfd，然后为这个fd设置handler函数为`vfio_msi_interrupt`
    
- 接着调用`vfio_add_kvm_msi_virq` ，这个函数里面首先为vector的kvm_interrupt 这个EventNotifier初始化eventfd；接着调用`kvm_irqchip_add_msi_route`在这个函数里面首先在used_gsi_bitmap里找到一个可用的virq，接着为这个vector创建 `kvm_irq_routing_entry`其核心逻辑就是在kvm_state的irq_routes增加一个新的entry，同时把used_gsi_bitmap给更新掉；接着调用`kvm_arch_add_msi_route_post` ，这个函数里面会新建一个`MSIRouteEntry`，然后将其加到全局的`msi_route_list`里面。最后通过`kvm_irqchip_commit_routes`通过`KVM_SET_GSI_ROUTING` 这个ioctl来设置irq routing(注意这里qemu会把kvm_state->irq_routes作为一个整体来提交)，具体的逻辑我们需要到kvm里面看一下相关的逻辑。
    
    `setup_routing_entry->kvm_set_routing_entry   `
    
    其核心逻辑就是将qemu侧的routing entry 添加到`kvm->irq_routing` 这个irq路由表里面，然后不同类型的路由其对应的表项也是不同的，比如在msi or msix的情况下表项是这样的
    
      `e->set = kvm_set_msi; //具体的中断delivery 函数     e->msi.address_lo = ue->u.msi.address_lo;     e->msi.address_hi = ue->u.msi.address_hi;     e->msi.data = ue->u.msi.data;`
    

然后再回到`vfio_add_kvm_msi_virq` 函数接着调用`kvm_irqchip_add_irqfd_notifier_gsi` 在kvm侧添加irqfd

`struct kvm_irqfd irqfd = {           .fd = fd,  // vector->kvm_interrupt 的eventfd           .gsi = virq,           .flags = assign ? 0 : KVM_IRQFD_FLAG_DEASSIGN, //assign 为true       };         kvm_vm_ioctl(s, KVM_IRQFD, &irqfd);      `

如上面所示通过KVM_IRQFD再次call到kvm里面具体调用的函数`kvm_irqfd`，最终调用`kvm_irqfd_assign` 函数。这个函数的具体逻辑在kvm里面创建一个相应的`kvm_kernel_irqfd`，然后为这个irqfd初始化 irq inject，irq shutdown函数以及相应的eventfd。然后把irqfd加到`kvm->irqfds.items`里面。如果kvm enable 了irq_bypass则需要为这个irqfd初始化consumer相关的数据结构。再回到`vfio_msix_vector_do_use` 这个函数接着往下看：

`........          if (vdev->nr_vectors < nr + 1) {           vfio_disable_irqindex(&vdev->vbasedev, VFIO_PCI_MSIX_IRQ_INDEX);           vdev->nr_vectors = nr + 1;           ret = vfio_enable_vectors(vdev, true);           if (ret) {               error_report("vfio: failed to enable vectors, %d", ret);           }         .......   `

核心逻辑如上面所示，从上面的实现所看首先调用`vfio_disable_irqindex`，我们先看一下这个函数的具体实现

`void vfio_disable_irqindex(VFIODevice *vbasedev, int index)   {       struct vfio_irq_set irq_set = {           .argsz = sizeof(irq_set),           .flags = VFIO_IRQ_SET_DATA_NONE | VFIO_IRQ_SET_ACTION_TRIGGER,           .index = index,           .start = 0,           .count = 0,       };          ioctl(vbasedev->fd, VFIO_DEVICE_SET_IRQS, &irq_set);   }   `

在vfio侧其调用逻辑如下

`vfio_pci_set_msi_trigger->vfio_msi_disable   `

具体的函数实现我这里就不贴出来了，其主要做的事情有

1. 释放irq bypass 相关数据结构，释放eventfd等
    
2. `pci_free_irq_vectors`  disable掉msix，释放为这个设备申请的irq
    
3. 将num_ctx清0 然后设置irq_type为`VFIO_PCI_NUM_IRQS`
    

调完disable函数之后，qemu里面紧接着调用`vfio_enable_vectors`，我们把这个函数的重点部分解析一下，首先创建一个irq_set

 `struct vfio_irq_set *irq_set;    ........    irq_set = g_malloc0(argsz);       irq_set->argsz = argsz;       irq_set->flags = VFIO_IRQ_SET_DATA_EVENTFD | VFIO_IRQ_SET_ACTION_TRIGGER;       irq_set->index = msix ? VFIO_PCI_MSIX_IRQ_INDEX : VFIO_PCI_MSI_IRQ_INDEX;       irq_set->start = 0;       irq_set->count = vdev->nr_vectors;       fds = (int32_t *)&irq_set->data;`

然后接着将目前为止将要和已经enable的vector的fd (每个vector的kvm_interrupt eventfd) 全部添加到fds，接着调用ioctl `VFIO_DEVICE_SET_IRQS` call到vfio，我们接着看vfio当中这一块的逻辑

`vfio_pci_set_msi_trigger->vfio_msi_enable         ->vfio_msi_set_block   `

vfio_msi_enable这个函数最核心的逻辑就是为这个直通设备申请物理的irq(之前申请的irq在把该设备从原生driver remove的时候已经free掉了)，具体调用逻辑如下

`pci_alloc_irq_vectors->pci_alloc_irq_vectors_affinity->__pci_enable_msix_range      `

是不是看到了跟非虚拟化场景下的一样的函数`__pci_enable_msix_range` , 具体实现大家可以参照上一节当中的相关实现逻辑；接着往下看vfio_msi_set_bock

`.........              for (i = 0, j = start; i < count && !ret; i++, j++) {                   int fd = fds ? fds[i] : -1;                   ret = vfio_msi_set_vector_signal(vdev, j, fd, msix);           }      .........   `

关键函数为`vfio_msi_set_vector_signal` ，其核心实现就是通过request_irq 为该直通设备注册中断处理函数vfio_msihandler同时register irq bypass producer。而vfio_msihandler就是通过之前注册的irqfd 将相应的中断delivery到vm里面。

### 总结和思考

interrupt remapping的功能最初是为了解决虚拟化中断delivery的问题而提出的，但是从我们的整篇文章的分析来看完全可以不需要这个remapping的功能。为什么这样说呢？因为通过分析我们可以看到直通设备中断注入过程是硬件中断先打到host，然后wakeup 相应的msi_handler；接着msi_handler通过kvm里面注册的irqfd 触发kvm_set_msi将相应的vector注到vm里面。但是intel什么还要搞这个呢？这个就不得不说直通设备的另外一种中断注入方式post interrupt，后面我们再来详细介绍一下。

注：封面图来自于 https://kernelgo.org/vtd_interrupt_remapping_code_analysis.html

阅读 2206

​

写留言

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/YHBSoNHqDiaH5V6ycE2TlM9McXhAbaV9aUBaTAibVhLibXH1rL4JTMb2eoQfdCicTm4wycOsopO82jG3szvYF3mTJw/300?wx_fmt=png&wxfrom=18)

Linux阅码场

435

写留言

写留言

**留言**

暂无留言