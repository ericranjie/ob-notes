
爱听音乐的章鱼哥 Linux内核之旅 _2022年02月17日 00:02_

现代各种处理器采用了许多不同的技术，但从物理结构上，CPU的层次结构却可以被统一的表示如下：

最底层为超线程层，此技术为Intel公司研发，目前在ARM等其他架构处理器中并未采用。超线程技术充分利用空闲CPU资源，在单一时间内可以让一个物理核心同时处理两个线程的工作。因此，在开启超线程的4物理核心的机器上，往往可以看到存在有8个逻辑CPU。

再往上的层次就是物理核心层。单独的一个物理核心的标志往往是其独占的私有缓存（Cache）。通常在两级缓存的情况下，每个物理核心独享L1 Cache，L2 Cache则为几个物理核心共享；在三级缓存的情况下，L1与L2为私有，L3为共享。当然前面说的是通常情况，不能武断地认为二级缓存与三级缓存的情况都是统一的。

继续往上层走就是处理器层级，多个物理核心可以共享最后一级缓存（LLC），这多个物理核心就被称为是一个Cluster或者Socket。芯片厂商会把多个物理核心（Core）封装到一个片（Chip）上，主板上会留有插槽，一个插槽可以插一个Chip。

最上层就到了NUMA的概念了，为了减少对内存总线的访问竞争，可以将CPU分属于不同的Node节点，通常情况下，CPU只访问Node内的memory。

在不考虑NUMA的情况下，CPU所属层次从高到低可以表示为`Cluster ---> Core ---> Threads`。

了解CPU的层次结构有什么作用呢？

以服务器为例，负载均衡是调度器常要考虑的问题。负载均衡在进行任务迁移的时候，把任务移动到哪个CPU上是重中之重。如果开启了超线程，那么移动到超线程层次的兄弟CPU上，被迁移后，原Cache上进程的数据仍然可用，可以说是最优解；如果移动到物理核心层的兄弟CPU上，则LLC上的原先该进程的数据仍然可以被访问到。可以说，迁移到离原CPU层次更近的兄弟CPU上带来的影响更小。

以手机等小型设备为例，功耗是核心问题。尤其是ARM多采用big core和little core的组合结构，即一个Cluster的物理核心是功耗和性能都比较高的CPU，另一个Cluster的物理核心是功耗和性能都比较低的CPU。这时候任务迁移，除了要考虑层次的兄弟临近关系，还要考虑移动到哪个CPU上可以保证功耗和性能的平衡问题。

那么，CPU的层次结构在Linux kernel中是如何表示的呢？

我们以ARM64下CPU的初始化开始入手。ARM64平台下CPU的初始化需要读取DTS（Device Tree Source，设备树）文件。

随意挑选arm64目录下一个处理器的dtsi文件的部分为例：

```cpp
 cpus {  
 #address-cells = <1>;  
 #size-cells = <0>;  
   
 cpu-map {  
 cluster0 {  
 core0 {  
 cpu = <&cpu0>;  
 };  
                ......  
            };  
            ......  
    };  
   
    cpu0: cpu@20000 {  
      device_type = "cpu";  
 compatible = "arm,cortex-a57";  
 reg = <0x20000>;  
 enable-method = "psci";  
 next-level-cache = <&cluster0_l2>;  
 };  
    ......  
   
     cluster0_l2: l2-cache0 {  
 compatible = "cache";  
 };  
    ......  
 };
```

以上是DTS文件中CPU定义的一个统一的格式。

```cpp
start_kernel --> setup_arch --> smp_init_cpus
```

`smp_init_cpus`在系统启动的时候读取DTS文件中CPU信息，获得CPU数量，并调用`smp_cpu_setup`函数进行possible位图的设置。

```cpp
 static int __init smp_cpu_setup(int cpu)  
 {  
 const struct cpu_operations *ops;  
   
 if (init_cpu_ops(cpu))  
 return -ENODEV;  
   
 ops = get_cpu_ops(cpu);  
 if (ops->cpu_init(cpu))  
 return -ENODEV;  
   
 set_cpu_possible(cpu, true);  
   
 return 0;  
 }
```

这里先了解一下内核中存在的四种CPU的状态表示，分别是`possible`、`online`、`present`和`active`。

```cpp
 typedef struct cpumask { DECLARE_BITMAP(bits, NR_CPUS); } cpumask_t;  
   
 #define DECLARE_BITMAP(name,bits) \  
 unsigned long name[BITS_TO_LONGS(bits)]  
   
 extern struct cpumask __cpu_possible_mask;  
 extern struct cpumask __cpu_online_mask;  
 extern struct cpumask __cpu_present_mask;  
 extern struct cpumask __cpu_active_mask;  
 #define cpu_possible_mask ((const struct cpumask *)&__cpu_possible_mask)  
 #define cpu_online_mask   ((const struct cpumask *)&__cpu_online_mask)  
 #define cpu_present_mask ((const struct cpumask *)&__cpu_present_mask)  
 #define cpu_active_mask   ((const struct cpumask *)&__cpu_active_mask)
```

可以从`smp_cpu_setup`函数看出，如果一个CPU编号对应的`struct cpu_operations`被建立成功，并且其对应的`init`函数没有执行失败，则将该CPU标记为possible。

```cpp
 struct cpu_operations {  
 const char*name;  
 int(*cpu_init)(unsigned int);  
 int(*cpu_prepare)(unsigned int);  
 int(*cpu_boot)(unsigned int);  
 void(*cpu_postboot)(void);  
 #ifdef CONFIG_HOTPLUG_CPU  
 bool(*cpu_can_disable)(unsigned int cpu);  
 int(*cpu_disable)(unsigned int cpu);  
 void(*cpu_die)(unsigned int cpu);  
 int(*cpu_kill)(unsigned int cpu);  
 #endif  
 #ifdef CONFIG_CPU_IDLE  
 int(*cpu_init_idle)(unsigned int);  
 int(*cpu_suspend)(unsigned long);  
 #endif  
 };
```

`cpu_init`：读取提出一个逻辑CPU的激活方法的必要数据；
`cpu_prepare`：测试是否可以启动给定CPU；
`cpu_boot`：启动CPU；
`cpu_postboot`：启动CPU后必要的清理工作。

除了上述的几个成员函数，可以看到其他成员函数是分别和CPU热插拔和CPUIDLE相关的电源操作，kernel抽象出了`struct cpu_operations`这个结构体，将启动、热插拔、空闲态接口统一起来，交给各个体系结构自己完成。

`start_kernel --> arch_call_rest_init --> rest_init --> kernel_init --> kernel_init_freezable --> smp_prepare_cpus`

接下来一直到`smp_prepare_cpus`函数，才到设置present位图的时候。

以下是`smp_prepare_cpus`的部分代码：

```cpp
 void __init smp_prepare_cpus(unsigned int max_cpus)  
 {  
 const struct cpu_operations *ops;  
 int err;  
 unsigned int cpu;  
 unsigned int this_cpu;  
   
    ......  
   
 for_each_possible_cpu(cpu) {  
   
 per_cpu(cpu_number, cpu) = cpu;  
   
 if (cpu == smp_processor_id())  
 continue;  
   
 ops = get_cpu_ops(cpu);  
 if (!ops)  
 continue;  
   
 err = ops->cpu_prepare(cpu);  
 if (err)  
 continue;  
   
 set_cpu_present(cpu, true);  
 numa_store_cpu_info(cpu);  
 }  
 }
```

前面看到，possible位图的设置依赖于`cpu_operations`结构体的初始化以及是否读取到正确的CPU信息；那这里present位图的设置则依赖于`cpu_prepare`的执行结果，也就是检测对应CPU是否可以启动。

`kernel_init_freezable --> smp_init --> bringup_nonboot_cpus`。

```cpp
 void bringup_nonboot_cpus(unsigned int setup_max_cpus)  
 {  
 unsigned int cpu;  
   
 for_each_present_cpu(cpu) {  
 if (num_online_cpus() >= setup_max_cpus)  
 break;  
 if (!cpu_online(cpu))  
 cpu_up(cpu, CPUHP_ONLINE);  
 }  
 }
```

`cpu_up --> _cpu_up --> cpuhp_up_callbacks --> cpuhp_invoke_callback --> bringup_cpu --> __cpu_up --> boot_secondary`

`cpu_up`，这里可以这么简单的理解，最开始系统中只有一个CPU，当代码执行到这里的时候，开始启动其他的CPU，其他CPU的启动过程，首先是需要复制主CPU的0号线程（即空闲线程），然后去执行热插拔相关的回调函数，一直执行到`boot_secondary`，在这里最终会执行到`struct cpu_operations`中的回调函数`cpu_boot`。

```cpp
 static int boot_secondary(unsigned int cpu, struct task_struct *idle)  
 {  
 const struct cpu_operations *ops = get_cpu_ops(cpu);  
   
 if (ops->cpu_boot)  
 return ops->cpu_boot(cpu);  
   
 return -EOPNOTSUPP;  
 }
```

而通常在`struct cpu_operations`中的成员`cpu_boot`的执行中，会执行到`secondary_entry --> secondary_startup --> __secondary_switched --> secondary_start_kernel`。

而在`secondary_start_kernel`中会调用`set_cpu_online`函数设置online位图。

active这个位图的修改函数定义在CPU热插拔的回调函数数组中，也就是`struct cpuhp_step cpuhp_hp_states[]`中，该位图服务于调度器，在开启热插拔的情况下，调度器需要实时监控CPU热插拔的每个信息，因此需要该位图实时将某个CPU移除或添加。

说完了四种位图，突然发现还没有说到CPU的层次结构在代码上的表示，CPU的层次结构使用一个结构体`struct cpu_topology`来表示。

```cpp
 struct cpu_topology {  
 int thread_id;  
 int core_id;  
 int package_id;  
 int llc_id;  
 cpumask_t thread_sibling;  
 cpumask_t core_sibling;  
 cpumask_t llc_sibling;  
 };  
 
 #ifdef CONFIG_GENERIC_ARCH_TOPOLOGY  
 extern struct cpu_topology cpu_topology[NR_CPUS];
```

其实，代码上CPU的层次结构的赋值代码与CPU启动的时间基本一致，主CPU的赋值是在`smp_prepare_cpus`函数中；其余CPU是在`secondary_start_kernel`函数中。

直接来看`cpu_topology`结构体的赋值函数`store_cpu_topology`。在调用该函数之前，已经在`init_cpu_topology`函数中进行过`cpu_topology`数组的初始化了，其中`thread_id`、`core_id`、`package_id`和`llc_id`皆初始化为-1。

```cpp
 void store_cpu_topology(unsigned int cpuid)  
 {  
 struct cpu_topology *cpuid_topo = &cpu_topology[cpuid];  
 u64 mpidr;  
   
 if (cpuid_topo->package_id != -1)  
 goto topology_populated;  
   
 mpidr = read_cpuid_mpidr();  
   
 /* Uniprocessor systems can rely on default topology values */  
 if (mpidr & MPIDR_UP_BITMASK)  
 return;  
   
 cpuid_topo->thread_id  = -1;  
 cpuid_topo->core_id    = cpuid;  
 cpuid_topo->package_id = cpu_to_node(cpuid);  
   
 pr_debug("CPU%u: cluster %d core %d thread %d mpidr %#016llx\n",  
  cpuid, cpuid_topo->package_id, cpuid_topo->core_id,  
  cpuid_topo->thread_id, mpidr);  
   
 topology_populated:  
 update_siblings_masks(cpuid);  
 }
```

以arm64为例，`thread_id`应该是超线程情况下才有意义，这里不考虑。`core_id`等同于物理核心id，也就是逻辑核心id，二者一致，`package_id`指向了其所属的NUMA的Node号。`update_sibling_masks`则用于更新用于表示各个层级的兄弟关系。

说了这么多，一直在说CPU的物理层次结构在代码上的表示，却还没有说到调度中CPU物理结构的参与。

这里关注`kernel_init_freezable`函数有这样一段代码：

```cpp
smp_init();  
sched_init_smp();
```

前面已经了解到`smp_init`函数执行完成时，各个次CPU已经启动完成，此时进入`sched_init_smp`函数。

```cpp
sched_init_smp --> sched_init_domains --> build_sched_domains

 static int  
 build_sched_domains(const struct cpumask *cpu_map, struct sched_domain_attr *attr)  
 {  
 enum s_alloc alloc_state = sa_none;  
 struct sched_domain *sd;  
 struct s_data d;  
 struct rq *rq = NULL;  
 int i, ret = -ENOMEM;  
 struct sched_domain_topology_level *tl_asym;  
 bool has_asym = false;  
   
 if (WARN_ON(cpumask_empty(cpu_map)))  
 goto error;  
   
     // 核心函数1，此时的cpu_map是active的位图  
 alloc_state = __visit_domain_allocation_hell(&d, cpu_map);  
 if (alloc_state != sa_rootdomain)  
 goto error;  
   
 tl_asym = asym_cpu_capacity_level(cpu_map);  
   
 /* Set up domains for CPUs specified by the cpu_map: */  
 for_each_cpu(i, cpu_map) {  
 struct sched_domain_topology_level *tl;  
 int dflags = 0;  
   
 sd = NULL;  
 for_each_sd_topology(tl) {  
 if (tl == tl_asym) {  
 dflags |= SD_ASYM_CPUCAPACITY;  
 has_asym = true;  
 }  
   
 if (WARN_ON(!topology_span_sane(tl, cpu_map, i)))  
 goto error;  
   
 // 核心函数2  
 sd = build_sched_domain(tl, cpu_map, attr, sd, dflags, i);  
   
 // 最底层的时候，进行赋值操作  
 if (tl == sched_domain_topology)  
 *per_cpu_ptr(d.sd, i) = sd;  
 if (tl->flags & SDTL_OVERLAP)  
 sd->flags |= SD_OVERLAP;  
 if (cpumask_equal(cpu_map, sched_domain_span(sd)))  
 break;  
 }  
 }  
   
 /* Build the groups for the domains */  
 for_each_cpu(i, cpu_map) {  
 for (sd = *per_cpu_ptr(d.sd, i); sd; sd = sd->parent) {  
 sd->span_weight = cpumask_weight(sched_domain_span(sd));  
 if (sd->flags & SD_OVERLAP) {  
 if (build_overlap_sched_groups(sd, i))  
 goto error;  
 } else {  
 // 核心函数3  
 if (build_sched_groups(sd, i))  
 goto error;  
 }  
 }  
 }  
   
 /* Calculate CPU capacity for physical packages and nodes */  
 for (i = nr_cpumask_bits-1; i >= 0; i--) {  
 if (!cpumask_test_cpu(i, cpu_map))  
 continue;  
   
 for (sd = *per_cpu_ptr(d.sd, i); sd; sd = sd->parent) {  
 claim_allocations(i, sd);  
 init_sched_groups_capacity(i, sd);  
 }  
 }  
   
 /* Attach the domains */  
 rcu_read_lock();  
 for_each_cpu(i, cpu_map) {  
 rq = cpu_rq(i);  
 sd = *per_cpu_ptr(d.sd, i);  
   
 /* Use READ_ONCE()/WRITE_ONCE() to avoid load/store tearing: */  
 if (rq->cpu_capacity_orig > READ_ONCE(d.rd->max_cpu_capacity))  
 WRITE_ONCE(d.rd->max_cpu_capacity, rq->cpu_capacity_orig);  
   
 cpu_attach_domain(sd, d.rd, i);  
 }  
 rcu_read_unlock();  
   
 if (has_asym)  
 static_branch_inc_cpuslocked(&sched_asym_cpucapacity);  
   
 if (rq && sched_debug_enabled) {  
 pr_info("root domain span: %*pbl (max cpu_capacity = %lu)\n",  
 cpumask_pr_args(cpu_map), rq->rd->max_cpu_capacity);  
 }  
   
 ret = 0;  
 error:  
 __free_domain_allocs(&d, alloc_state, cpu_map);  
   
 return ret;  
 }
```

首先，分析第1个核心函数`__visit_domain_allocation_hell`。

```cpp
 static enum s_alloc  
 __visit_domain_allocation_hell(struct s_data *d, const struct cpumask *cpu_map) {  
 memset(d, 0, sizeof(*d));  
   
 if (__sdt_alloc(cpu_map))  
 return sa_sd_storage;  
 d->sd = alloc_percpu(struct sched_domain *);  
 if (!d->sd)  
 return sa_sd_storage;  
 d->rd = alloc_rootdomain();  
 if (!d->rd)  
 return sa_sd;  
   
 return sa_rootdomain;  
 }
```

首先进入`__sdt_alloc`函数。

```cpp
 static int __sdt_alloc(const struct cpumask *cpu_map)  
 {  
 struct sched_domain_topology_level *tl;  
 int j;  
   
 for_each_sd_topology(tl) {  
 struct sd_data *sdd = &tl->data;  
   
 sdd->sd = alloc_percpu(struct sched_domain *);  
 if (!sdd->sd)  
 return -ENOMEM;  
   
 sdd->sds = alloc_percpu(struct sched_domain_shared *);  
 if (!sdd->sds)  
 return -ENOMEM;  
   
 sdd->sg = alloc_percpu(struct sched_group *);  
 if (!sdd->sg)  
 return -ENOMEM;  
   
 sdd->sgc = alloc_percpu(struct sched_group_capacity *);  
 if (!sdd->sgc)  
 return -ENOMEM;  
   
 for_each_cpu(j, cpu_map) {  
 struct sched_domain *sd;  
 struct sched_domain_shared *sds;  
 struct sched_group *sg;  
 struct sched_group_capacity *sgc;  
   
 sd = kzalloc_node(sizeof(struct sched_domain) + cpumask_size(),  
 GFP_KERNEL, cpu_to_node(j));  
 if (!sd)  
 return -ENOMEM;  
   
 *per_cpu_ptr(sdd->sd, j) = sd;  
   
 sds = kzalloc_node(sizeof(struct sched_domain_shared),  
 GFP_KERNEL, cpu_to_node(j));  
 if (!sds)  
 return -ENOMEM;  
   
 *per_cpu_ptr(sdd->sds, j) = sds;  
   
 sg = kzalloc_node(sizeof(struct sched_group) + cpumask_size(),  
 GFP_KERNEL, cpu_to_node(j));  
 if (!sg)  
 return -ENOMEM;  
   
 sg->next = sg;  
   
 *per_cpu_ptr(sdd->sg, j) = sg;  
   
 sgc = kzalloc_node(sizeof(struct sched_group_capacity) + cpumask_size(),  
 GFP_KERNEL, cpu_to_node(j));  
 if (!sgc)  
 return -ENOMEM;  
   
 #ifdef CONFIG_SCHED_DEBUG  
 sgc->id = j;  
 #endif  
   
 *per_cpu_ptr(sdd->sgc, j) = sgc;  
 }  
 }  
   
 return 0;  
 }
```

在这个函数中我们接触到第一个比较关键的数据结构`struct sched_domain_topology_level`。

```cpp
 struct sched_domain_topology_level {  
 sched_domain_mask_f mask;  
 sched_domain_flags_f sd_flags;  
 int    flags;  
 int    numa_level;  
 struct sd_data      data;  
 #ifdef CONFIG_SCHED_DEBUG  
 char                *name;  
 #endif  
 };  
   
 static struct sched_domain_topology_level default_topology[] = {  
 #ifdef CONFIG_SCHED_SMT  
 { cpu_smt_mask, cpu_smt_flags, SD_INIT_NAME(SMT) },  
 #endif  
 #ifdef CONFIG_SCHED_MC  
 { cpu_coregroup_mask, cpu_core_flags, SD_INIT_NAME(MC) },  
 #endif  
 { cpu_cpu_mask, SD_INIT_NAME(DIE) },  
 { NULL, },  
 };  
   
 static struct sched_domain_topology_level *sched_domain_topology =  
 default_topology;  
   
 #define for_each_sd_topology(tl)\  
 for (tl = sched_domain_topology; tl->mask; tl++)  
   
 void set_sched_topology(struct sched_domain_topology_level *tl)  
 {  
 if (WARN_ON_ONCE(sched_smp_initialized))  
 return;  
   
 sched_domain_topology = tl;  
 }
```

可以看到，`struct sched_domain_topology_level`使用一个数组`default_topology`给出，当然各个体系结构下也可以修改该数组。该数组一般按照CPU层次从低到高排列。

`struct sched_domain_toplogy_level`的前两个成员`mask`和`flag`分别是两个函数，用于指定该CPU层次下的兄弟cpumask和flag标志位。而`struct sd_data`是该数据结构的核心，其成员将在后续完成赋值。

这里简单看下物理核心层的`mask`成员`cpu_coregroup_mask`。

```cpp
 const struct cpumask *cpu_coregroup_mask(int cpu)  
 {  
 const cpumask_t *core_mask = cpumask_of_node(cpu_to_node(cpu));  
   
 /* Find the smaller of NUMA, core or LLC siblings */  
 if (cpumask_subset(&cpu_topology[cpu].core_sibling, core_mask)) {  
 /* not numa in package, lets use the package siblings */  
 core_mask = &cpu_topology[cpu].core_sibling;  
 }  
 if (cpu_topology[cpu].llc_id != -1) {  
 if (cpumask_subset(&cpu_topology[cpu].llc_sibling, core_mask))  
 core_mask = &cpu_topology[cpu].llc_sibling;  
 }  
   
 return core_mask;  
 }
```

可以看到，本质上是利用`cpu_topology`数组来获得`core_sibling`的值，到这里就和物理层级联系起来了。

```cpp
 struct sd_data {  
 struct sched_domain *__percpu *sd;  
 struct sched_domain_shared *__percpu *sds;  
 struct sched_group *__percpu *sg;  
 struct sched_group_capacity *__percpu *sgc;  
 };
```

分析`__sdt_alloc`，可以看到为每个层次的`struct sched_domain_topology_level`的`sd_data`分配percpu的`sched_domain`、`sched_domain_shared`、`sched_group`和`sched_group_capacity`。这样每个CPU可以通过`default_topology`数组轻易找到自己的`sched_domain`等数据。

接下来，继续分析`build_sched_domains`的第2个核心函数`build_sched_domain`，该函数在每个CPU的每个层次上都要去调用一次。

```cpp
 static struct sched_domain *build_sched_domain(struct sched_domain_topology_level *tl,  
 const struct cpumask *cpu_map, struct sched_domain_attr *attr,  
 struct sched_domain *child, int dflags, int cpu)  
 {  
 struct sched_domain *sd = sd_init(tl, cpu_map, child, dflags, cpu);  
   
 if (child) {  
 sd->level = child->level + 1;  
 sched_domain_level_max = max(sched_domain_level_max, sd->level);  
 child->parent = sd;  
   
 if (!cpumask_subset(sched_domain_span(child),  
     sched_domain_span(sd))) {  
 pr_err("BUG: arch topology borken\n");  
 #ifdef CONFIG_SCHED_DEBUG  
 pr_err("     the %s domain not a subset of the %s domain\n",  
 child->name, sd->name);  
 #endif  
 /* Fixup, ensure @sd has at least @child CPUs. */  
 cpumask_or(sched_domain_span(sd),  
    sched_domain_span(sd),  
    sched_domain_span(child));  
 }  
   
 }  
 set_domain_attribute(sd, attr);  
   
 return sd;  
 }
```

其中`sd_init`函数完成对应`sched_domain_topology_level`对应层次的`sched_domain`的一些数据的填充以及`sd_data`中mask和flag的修改。

```cpp
 static struct sched_domain *  
 sd_init(struct sched_domain_topology_level *tl,  
 const struct cpumask *cpu_map,  
 struct sched_domain *child, int dflags, int cpu)  
 {  
 struct sd_data *sdd = &tl->data;  
 struct sched_domain *sd = *per_cpu_ptr(sdd->sd, cpu);  
 int sd_id, sd_weight, sd_flags = 0;  
   
 #ifdef CONFIG_NUMA  
 /*  
  * Ugly hack to pass state to sd_numa_mask()...  
  */  
 sched_domains_curr_level = tl->numa_level;  
 #endif  
   
 sd_weight = cpumask_weight(tl->mask(cpu));  
   
 if (tl->sd_flags)  
 sd_flags = (*tl->sd_flags)();  
 if (WARN_ONCE(sd_flags & ~TOPOLOGY_SD_FLAGS,  
 "wrong sd_flags in topology description\n"))  
 sd_flags &= TOPOLOGY_SD_FLAGS;  
   
 /* Apply detected topology flags */  
 sd_flags |= dflags;  
   
 *sd = (struct sched_domain){  
 .min_interval= sd_weight,  
 .max_interval= 2*sd_weight,  
 .busy_factor= 16,  
 .imbalance_pct= 117,  
   
 .cache_nice_tries= 0,  
   
 .flags= 1*SD_BALANCE_NEWIDLE  
 | 1*SD_BALANCE_EXEC  
 | 1*SD_BALANCE_FORK  
 | 0*SD_BALANCE_WAKE  
 | 1*SD_WAKE_AFFINE  
 | 0*SD_SHARE_CPUCAPACITY  
 | 0*SD_SHARE_PKG_RESOURCES  
 | 0*SD_SERIALIZE  
 | 1*SD_PREFER_SIBLING  
 | 0*SD_NUMA  
 | sd_flags  
 ,  
   
 .last_balance= jiffies,  
 .balance_interval= sd_weight,  
 .max_newidle_lb_cost= 0,  
 .next_decay_max_lb_cost= jiffies,  
 .child= child,  
 #ifdef CONFIG_SCHED_DEBUG  
 .name= tl->name,  
 #endif  
 };  
   
 cpumask_and(sched_domain_span(sd), cpu_map, tl->mask(cpu));  
 sd_id = cpumask_first(sched_domain_span(sd));  
   
 /*  
  * Convert topological properties into behaviour.  
  */  
   
 /* Don't attempt to spread across CPUs of different capacities. */  
 if ((sd->flags & SD_ASYM_CPUCAPACITY) && sd->child)  
 sd->child->flags &= ~SD_PREFER_SIBLING;  
   
 if (sd->flags & SD_SHARE_CPUCAPACITY) {  
 sd->imbalance_pct = 110;  
   
 } else if (sd->flags & SD_SHARE_PKG_RESOURCES) {  
 sd->imbalance_pct = 117;  
 sd->cache_nice_tries = 1;  
   
 #ifdef CONFIG_NUMA  
 } else if (sd->flags & SD_NUMA) {  
 sd->cache_nice_tries = 2;  
   
 sd->flags &= ~SD_PREFER_SIBLING;  
 sd->flags |= SD_SERIALIZE;  
 if (sched_domains_numa_distance[tl->numa_level] > node_reclaim_distance) {  
 sd->flags &= ~(SD_BALANCE_EXEC |  
        SD_BALANCE_FORK |  
        SD_WAKE_AFFINE);  
 }  
   
 #endif  
 } else {  
 sd->cache_nice_tries = 1;  
 }  
   
 /*  
  * For all levels sharing cache; connect a sched_domain_shared  
  * instance.  
  */  
 if (sd->flags & SD_SHARE_PKG_RESOURCES) {  
 sd->shared = *per_cpu_ptr(sdd->sds, sd_id);  
 atomic_inc(&sd->shared->ref);  
 atomic_set(&sd->shared->nr_busy_cpus, sd_weight);  
 }  
   
 sd->private = sdd;  
   
 return sd;  
 }
```

这里可以看到，`sched_domain`中的成员往往是与调度中负载均衡操作相关的一些字段，例如`min_interval`是最小的load balance间隔，`max_interval`是最大的load balance间隔等等。

这里应该注意，`sched_domain`的父子关系，高层级的`sched_domain`是低层级的`sched_domain`的parent，低层级的`sched_domain`是高层级的child。`sched_domain`的`span`为自己该层级的兄弟CPU以及下面几个层级的兄弟CPU的并集。

创建完`sched_domain`之后，开始从CPU开始由低到高依次建立`sched_group`，这就是第3个核心函数，`build_sched_groups`。

```cpp
 static int  
 build_sched_groups(struct sched_domain *sd, int cpu)  
 {  
 struct sched_group *first = NULL, *last = NULL;  
 struct sd_data *sdd = sd->private;  
 const struct cpumask *span = sched_domain_span(sd);  
 struct cpumask *covered;  
 int i;  
   
 lockdep_assert_held(&sched_domains_mutex);  
 covered = sched_domains_tmpmask;  
   
 cpumask_clear(covered);  
   
 for_each_cpu_wrap(i, span, cpu) {  
 struct sched_group *sg;  
   
 if (cpumask_test_cpu(i, covered))  
 continue;  
   
 sg = get_group(i, sdd);  
   
 cpumask_or(covered, covered, sched_group_span(sg));  
   
 if (!first)  
 first = sg;  
 if (last)  
 last->next = sg;  
 last = sg;  
 }  
 last->next = first;  
 sd->groups = first;  
   
 return 0;  
 }
```

`build_sched_groups`函数的核心是`get_group`函数。

```cpp
 static struct sched_group *get_group(int cpu, struct sd_data *sdd)  
 {  
 struct sched_domain *sd = *per_cpu_ptr(sdd->sd, cpu);  
 struct sched_domain *child = sd->child;  
 struct sched_group *sg;  
 bool already_visited;  
   
 if (child)  
 cpu = cpumask_first(sched_domain_span(child));  
   
 sg = *per_cpu_ptr(sdd->sg, cpu);  
 sg->sgc = *per_cpu_ptr(sdd->sgc, cpu);  
   
 /* Increase refcounts for claim_allocations: */  
 already_visited = atomic_inc_return(&sg->ref) > 1;  
 /* sgc visits should follow a similar trend as sg */  
 WARN_ON(already_visited != (atomic_inc_return(&sg->sgc->ref) > 1));  
   
 /* If we have already visited that group, it's already initialized. */  
 if (already_visited)  
 return sg;  
   
 if (child) {  
 cpumask_copy(sched_group_span(sg), sched_domain_span(child));  
 cpumask_copy(group_balance_mask(sg), sched_group_span(sg));  
 } else {  
 cpumask_set_cpu(cpu, sched_group_span(sg));  
 cpumask_set_cpu(cpu, group_balance_mask(sg));  
 }  
   
 sg->sgc->capacity = SCHED_CAPACITY_SCALE * cpumask_weight(sched_group_span(sg));  
 sg->sgc->min_capacity = SCHED_CAPACITY_SCALE;  
 sg->sgc->max_capacity = SCHED_CAPACITY_SCALE;  
   
 return sg;  
 }
```

可以从`get_group`函数看出来，如果是最底层的`sched_domain`，即不存在`child`，那么`get_group`中建立的`sched_group`是该CPU独有的。而如果是非最底层的`sched_domain`，那么`get_group`中建立的`sched_group`是其`child`的`span`成员共享的。这里注意，`sched_group_capacity`和`sched_group`两位一体，表示该调度组的计算能力。

到这里，调度域和调度组的关系基本可以理清楚。在CPU层次结构上，CPU在每个层次都有一个调度域，该CPU在高层次的调度域与该CPU在低层次的调度域互成父子关系。

而调度域下皆有一个调度组，调度组是呈链表组织的。一个调度域将其调度域中所有CPU按照兄弟关系分成不同的调度组，然后以链表形式组织起来，一个调度域中的所有CPU在该CPU层次上是共享这个调度组的。

更新完成`sched_group_capacity`之后，最后一部分`cpu_attach_domain --> rq_attach_root`。

`rq_attach_root`中有这样的一行代码：`rq->rd = rd;`。至此，从`rq --> root_domain --> sched_domain`的关系已经建立完成。

______________________________________________________________________

​
