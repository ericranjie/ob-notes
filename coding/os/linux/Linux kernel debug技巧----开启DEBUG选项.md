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

作者：[wowo](http://www.wowotech.net/author/2 "runangaozhong@163.com") 发布于：2016-11-1 19:39 分类：[Linux应用技巧](http://www.wowotech.net/sort/linux_application)

kernel的source code中有很多使用pr_debug/dev_dbg输出的日志信息（例如device tree解析的代码，drivers/of/fdt.c）。默认情况下，kernel不会将这些日志输出到控制台上，除非：

> 1）开启了DEBUG宏，并且
> 
> 2）kernel printk的默认日志级别大于7

看似简单，不过我相信每个人都问过这样的问题（不管是问自己还是问别人，特别是在调试kernel启动过程的时候，例如device tree的匹配、device probe等）：怎么开启DEBUG选项？

之所以有这篇短文，是因为我也问过（不止一次），于是就记录如下：

1）开启了DEBUG宏

其实开启DEBUG宏的方法很简单，在需要pr_debug/dev_dbg输出的模块开头，直接#define DEBUG即可，kernel中有一个例子：

> /* init/main.c */
> 
> #define DEBUG           /* Enable initcall_debug */

不过这种方法有个缺点：我们必须准确的知道需要debug那个C文件，如果想大网撒鱼（例如，想debug为什么新修改的DTS文件没有起作用，而又对kernel fdt的代码不是很熟悉），就麻烦了。这里我给一个大杀器：在编译kernel的时候，通过KCFLAGS直接传递，这样可以全局生效，如下（以本站的“[X Project](http://www.wowotech.net/sort/x_project)”为例）：

> diff --git a/Makefile b/Makefile  
> index 0835b1c..59625f4 100644  
> --- a/Makefile  
> +++ b/Makefile  
> @@ -83,7 +83,7 @@ kernel-config:  
> kernel:  
>         mkdir -p $(KERNEL_OUT_DIR)  
>         make -C $(KERNEL_DIR) CROSS_COMPILE=$(CROSS_COMPILE) KBUILD_OUTPUT=$(KERNEL_OUT_DIR) ARCH=$(BOARD_ARCH) $(KERNEL_D  
> -       make -C $(KERNEL_DIR) CROSS_COMPILE=$(CROSS_COMPILE) KBUILD_OUTPUT=$(KERNEL_OUT_DIR) ARCH=$(BOARD_ARCH) $(KERNEL_T  
> +       make -C $(KERNEL_DIR) CROSS_COMPILE=$(CROSS_COMPILE) KBUILD_OUTPUT=$(KERNEL_OUT_DIR) KCFLAGS=-DDEBUG ARCH=$(BOARD_  
>  

2）设置kernel printk的默认日志级别为8

修改printk的默认日志级别的方法有多种，例如直接修改printk.c(新kernel为printk.h)中的CONSOLE_LOGLEVEL_DEFAULT宏定义。不过修改kernel原生代码的方式稍显粗暴，我们还有优雅一些的手段，例如通过命令行参数的loglevel变量传递，如下：

> diff --git a/arch/arm64/configs/xprj_defconfig b/arch/arm64/configs/xprj_defconfig  
> index 5d0d591..9335d3f 100644  
> --- a/arch/arm64/configs/xprj_defconfig  
> +++ b/arch/arm64/configs/xprj_defconfig  
> @@ -320,7 +320,7 @@ CONFIG_FORCE_MAX_ZONEORDER=11  
> #  
> # Boot options  
> #  
> -CONFIG_CMDLINE="earlycon=owl_serial"  
> +CONFIG_CMDLINE="earlycon=owl_serial loglevel=8"  
> CONFIG_CMDLINE_FORCE=y  
> # CONFIG_EFI is not set

3）修改完之后，编译并启动kernel看看效果吧（是不是很爽？）

> Starting kernel ...
> 
> flushing dcache successfully.  
> [    0.000000] Booting Linux on physical CPU 0x0  
> [    0.000000] Linux version 4.6.0-rc5+ (pengo@ubuntu) (gcc version 4.8.3 20131202 (prerelease) (crosstool-NG linaro-1.13.1-4.8-2013.12 - Linaro GCC 2013.11) ) #17 SMP Tue Nov 1 03:52:32 PDT 2016  
> [    0.000000] Boot CPU: AArch64 Processor [410fd032]  
> [    0.000000] earlycon: owl_serial0 at I/O port 0x0 (options '')  
> [    0.000000] bootconsole [owl_serial0] enabled  
> [    0.000000] On node 0 totalpages: 524288  
> [    0.000000]   DMA zone: 8192 pages used for memmap  
> [    0.000000]   DMA zone: 0 pages reserved  
> [    0.000000]   DMA zone: 524288 pages, LIFO batch:31  
> [    0.000000]  -> unflatten_device_tree()  
> [    0.000000] Unflattening device tree:  
> [    0.000000] magic: d00dfeed  
> [    0.000000] size: 00001000  
> [    0.000000] version: 00000011  
> [    0.000000]   size is cb0, allocating...  
> [    0.000000]   unflattening ffffffc07ffed1c8...  
> [    0.000000] fixed up name for  ->  
> [    0.000000] fixed up name for memory -> memory  
> [    0.000000] fixed up name for chosen -> chosen  
> [    0.000000] fixed up name for interrupt-controller@e00f1000 -> interrupt-controller  
> [    0.000000] fixed up name for timer -> timer  
> [    0.000000]  <- unflatten_device_tree()  
> [    0.000000] Failed to find device node for boot cpu  
> [    0.000000] missing boot CPU MPIDR, not enabling secondaries  
> [    0.000000] mask of set bits 0x0  
> [    0.000000] MPIDR hash: aff0[0] aff1[8] aff2[16] aff3[32] mask[0x0] bits[0]  
> [    0.000000] percpu: Embedded 14 pages/cpu @ffffffc07ffdd000 s28032 r0 d29312 u57344  
> [    0.000000] pcpu-alloc: s28032 r0 d29312 u57344 alloc=14*4096  
> [    0.000000] pcpu-alloc: [0] 0  
> [    0.000000] Detected VIPT I-cache on CPU0  
> [    0.000000] Built 1 zonelists in Zone order, mobility grouping on.  Total pages: 516096  
> [    0.000000] Kernel command line: earlycon=owl_serial loglevel=8  
> [    0.000000] doing Booting kernel, parsing ARGS: 'earlycon=owl_serial loglevel=8'  
> [    0.000000] doing Booting kernel: earlycon='owl_serial'  
> [    0.000000] doing Booting kernel: loglevel='8'  
> [    0.000000] PID hash table entries: 4096 (order: 3, 32768 bytes)  
> [    0.000000] Dentry cache hash table entries: 262144 (order: 9, 2097152 bytes)  
> [    0.000000] Inode-cache hash table entries: 131072 (order: 8, 1048576 bytes)  
> [    0.000000] software IO TLB [mem 0x79cb3000-0x7dcb3000] (64MB) mapped at [ffffffc079cb3000-ffffffc07dcb2fff]  
> [    0.000000] Memory: 1993604K/2097152K available (1032K kernel code, 78K rwdata, 128K rodata, 120K init, 201K bss, 103548K reserved, 0K cma-reserved)  
> [    0.000000] Virtual kernel memory layout:  
> [    0.000000]     modules : 0xffffff8000000000 - 0xffffff8008000000   (   128 MB)  
> [    0.000000]     vmalloc : 0xffffff8008000000 - 0xffffffbdbfff0000   (   246 GB)  
> [    0.000000]       .text : 0xffffff8008080000 - 0xffffff8008182000   (  1032 KB)  
> [    0.000000]     .rodata : 0xffffff8008182000 - 0xffffff80081a3000   (   132 KB)  
> [    0.000000]       .init : 0xffffff80081a3000 - 0xffffff80081c1000   (   120 KB)  
> [    0.000000]       .data : 0xffffff80081c1000 - 0xffffff80081d4800   (    78 KB)  
> [    0.000000]     fixed   : 0xffffffbffe7fd000 - 0xffffffbffec00000   (  4108 KB)  
> [    0.000000]     PCI I/O : 0xffffffbffee00000 - 0xffffffbfffe00000   (    16 MB)  
> [    0.000000]     memory  : 0xffffffc000000000 - 0xffffffc080000000   (  2048 MB)  
> [    0.000000] SLUB: HWalign=64, Order=0-3, MinObjects=0, CPUs=1, Nodes=1  
> [    0.000000] Hierarchical RCU implementation.  
> [    0.000000]  Build-time adjustment of leaf fanout to 64.  
> [    0.000000]  RCU restricting CPUs from NR_CPUS=64 to nr_cpu_ids=1.  
> [    0.000000] RCU: Adjusting geometry for rcu_fanout_leaf=64, nr_cpu_ids=1  
> [    0.000000] NR_IRQS:64 nr_irqs:64 0  
> [    0.000000] of_irq_init: init /interrupt-controller@e00f1000 (ffffffc07ffed800), parent           (null)  
> [    0.000000] OF: ** translation for device /interrupt-controller@e00f1000 **  
> [    0.000000] OF: bus is default (na=2, ns=2) on /  
> [    0.000000] OF: translating address: 00000000 e00f1000  
> [    0.000000] OF: reached root node  
> [    0.000000] OF: ** translation for device /interrupt-controller@e00f1000 **  
> [    0.000000] OF: bus is default (na=2, ns=2) on /  
> [    0.000000] OF: translating address: 00000000 e00f2000  
> [    0.000000] OF: reached root node  
> [    0.000000] OF: ** translation for device /interrupt-controller@e00f1000 **  
> [    0.000000] OF: bus is default (na=2, ns=2) on /  
> [    0.000000] OF: translating address: 00000000 e00f2000  
> [    0.000000] OF: reached root node  
> [    0.000000] irq: Added domain (null)  
> [    0.000000] of_irq_parse_one: dev=/timer, index=0  
> [    0.000000]  intspec=1 intlen=12  
> [    0.000000]  intsize=3 intlen=12  
> [    0.000000] of_irq_parse_raw:  /interrupt-controller@e00f1000:00000001,0000000d,00000f08  
> [    0.000000] of_irq_parse_raw: ipar=/interrupt-controller@e00f1000, size=3  
> [    0.000000]  -> addrsize=0  
> [    0.000000]  -> got it !  
> [    0.000000] of_irq_parse_one: dev=/timer, index=1  
> [    0.000000]  intspec=1 intlen=12  
> [    0.000000]  intsize=3 intlen=12  
> [    0.000000] of_irq_parse_raw:  /interrupt-controller@e00f1000:00000001,0000000e,00000f08  
> [    0.000000] of_irq_parse_raw: ipar=/interrupt-controller@e00f1000, size=3  
> [    0.000000]  -> addrsize=0  
> [    0.000000]  -> got it !  
> [    0.000000] of_irq_parse_one: dev=/timer, index=2  
> [    0.000000]  intspec=1 intlen=12  
> [    0.000000]  intsize=3 intlen=12  
> [    0.000000] of_irq_parse_raw:  /interrupt-controller@e00f1000:00000001,0000000b,00000f08  
> [    0.000000] of_irq_parse_raw: ipar=/interrupt-controller@e00f1000, size=3  
> [    0.000000]  -> addrsize=0  
> [    0.000000]  -> got it !  
> [    0.000000] of_irq_parse_one: dev=/timer, index=3  
> [    0.000000]  intspec=1 intlen=12  
> [    0.000000]  intsize=3 intlen=12  
> [    0.000000] of_irq_parse_raw:  /interrupt-controller@e00f1000:00000001,0000000a,00000f08  
> [    0.000000] of_irq_parse_raw: ipar=/interrupt-controller@e00f1000, size=3  
> [    0.000000]  -> addrsize=0  
> [    0.000000]  -> got it !  
> [    0.000000] Architected cp15 timer(s) running at 24.00MHz (phys).  
> [    0.000000] clocksource: arch_sys_counter: mask: 0xffffffffffffff max_cycles: 0x588fe9dc0, max_idle_ns: 440795202592 ns  
> [    0.000031] sched_clock: 56 bits at 24MHz, resolution 41ns, wraps every 4398046511097ns  
> [    0.007999] Registered 0xffffff800816bf78 as sched_clock source  
> [    0.013906] Calibrating delay loop (skipped), value calculated using timer frequency.. 48.00 BogoMIPS (lpj=96000)  
> [    0.024093] pid_max: default: 4096 minimum: 301  
> [    0.028687] Mount-cache hash table entries: 4096 (order: 3, 32768 bytes)  
> [    0.035281] Mountpoint-cache hash table entries: 4096 (order: 3, 32768 bytes)  
> [    0.042468] kobject: 'fs' (ffffffc079802c00): kobject_add_internal: parent: '', set: ''  
> [    0.051687] No CPU information found in DT  
> [    0.055499] CPU0: cluster 0 core 0 thread -1 mpidr 0x00000080000000  
> [    0.061718] ASID allocator initialised with 65536 entries  
> [    0.067687] Brought up 1 CPUs  
> [    0.070031] SMP: Total of 1 processors activated.  
> [    0.074718] CPU: All CPU(s) started at EL2  
> [    0.079281] kobject: 'devices' (ffffffc079823a98): kobject_add_internal: parent: '', set: ''  
> [    0.088249] kobject: 'devices' (ffffffc079823a98): kobject_uevent_env  
> [    0.094656] kobject: 'devices' (ffffffc079823a98): kobject_uevent_env: attempted to send uevent without kset!  
> [    0.104531] kobject: 'dev' (ffffffc079823b00): kobject_add_internal: parent: '', set: ''  
> [    0.113624] kobject: 'block' (ffffffc079823b80): kobject_add_internal: parent: 'dev', set: ''  
> [    0.122656] kobject: 'char' (ffffffc079823c00): kobject_add_internal: parent: 'dev', set: ''  
> [    0.131562] kobject: 'bus' (ffffffc079823c98): kobject_add_internal: parent: '', set: ''  
> [    0.140687] kobject: 'bus' (ffffffc079823c98): kobject_uevent_env  
> [    0.146749] kobject: 'bus' (ffffffc079823c98): kobject_uevent_env: attempted to send uevent without kset!  
> [    0.156281] kobject: 'system' (ffffffc079823d18): kobject_add_internal: parent: 'devices', set: ''  
> [    0.165718] kobject: 'system' (ffffffc079823d18): kobject_uevent_env  
> [    0.172062] kobject: 'system' (ffffffc079823d18): kobject_uevent_env: attempted to send uevent without kset!  
> [    0.181843] kobject: 'class' (ffffffc079823d98): kobject_add_internal: parent: '', set: ''  
> [    0.191124] kobject: 'class' (ffffffc079823d98): kobject_uevent_env  
> [    0.197343] kobject: 'class' (ffffffc079823d98): kobject_uevent_env: attempted to send uevent without kset!  
> [    0.207062] kobject: 'firmware' (ffffffc079823e00): kobject_add_internal: parent: '', set: ''  
> [    0.216593] device: 'platform': device_add  
> [    0.220656] kobject: 'platform' (ffffff80081d2680): kobject_add_internal: parent: 'devices', set: 'devices'  
> [    0.230374] kobject: 'platform' (ffffff80081d2680): kobject_uevent_env  
> [    0.236874] kobject: 'platform' (ffffff80081d2680): kobject_uevent_env: filter function caused the event to drop!  
> [    0.247093] kobject: 'platform' (ffffffc079822818): kobject_add_internal: parent: 'bus', set: 'bus'  
> [    0.256124] kobject: 'platform' (ffffffc079822818): kobject_uevent_env  
> [    0.262624] kobject: 'platform' (ffffffc079822818): fill_kobj_path: path = '/bus/platform'  
> [    0.270843] kobject: 'devices' (ffffffc079823e98): kobject_add_internal: parent: 'platform', set: ''  
> [    0.280468] kobject: 'devices' (ffffffc079823e98): kobject_uevent_env  
> [    0.286874] kobject: 'devices' (ffffffc079823e98): kobject_uevent_env: filter function caused the event to drop!  
> [    0.297031] kobject: 'drivers' (ffffffc079823f18): kobject_add_internal: parent: 'platform', set: ''  
> [    0.306656] kobject: 'drivers' (ffffffc079823f18): kobject_uevent_env  
> [    0.313062] kobject: 'drivers' (ffffffc079823f18): kobject_uevent_env: filter function caused the event to drop!  
> [    0.323187] bus: 'platform': registered  
> [    0.326999] kobject: 'cpu' (ffffffc079822a18): kobject_add_internal: parent: 'bus', set: 'bus'  
> [    0.335593] kobject: 'cpu' (ffffffc079822a18): kobject_uevent_env  
> [    0.341656] kobject: 'cpu' (ffffffc079822a18): fill_kobj_path: path = '/bus/cpu'  
> [    0.349031] kobject: 'devices' (ffffffc079823f98): kobject_add_internal: parent: 'cpu', set: ''  
> [    0.358218] kobject: 'devices' (ffffffc079823f98): kobject_uevent_env  
> [    0.364624] kobject: 'devices' (ffffffc079823f98): kobject_uevent_env: filter function caused the event to drop!  
> [    0.374749] kobject: 'drivers' (ffffffc079820d18): kobject_add_internal: parent: 'cpu', set: ''  
> [    0.383937] kobject: 'drivers' (ffffffc079820d18): kobject_uevent_env  
> [    0.390374] kobject: 'drivers' (ffffffc079820d18): kobject_uevent_env: filter function caused the event to drop!  
> [    0.400499] bus: 'cpu': registered  
> [    0.403874] device: 'cpu': device_add  
> [    0.407531] kobject: 'cpu' (ffffffc079822c10): kobject_add_internal: parent: 'system', set: 'devices'  
> [    0.416718] kobject: 'cpu' (ffffffc079822c10): kobject_uevent_env  
> [    0.422781] kobject: 'cpu' (ffffffc079822c10): kobject_uevent_env: filter function caused the event to drop!  
> [    0.432562] kobject: 'container' (ffffffc079822e18): kobject_add_internal: parent: 'bus', set: 'bus'  
> [    0.441656] kobject: 'container' (ffffffc079822e18): kobject_uevent_env  
> [    0.448249] kobject: 'container' (ffffffc079822e18): fill_kobj_path: path = '/bus/container'  
> [    0.456656] kobject: 'devices' (ffffffc079820d98): kobject_add_internal: parent: 'container', set: ''  
> [    0.466374] kobject: 'devices' (ffffffc079820d98): kobject_uevent_env  
> [    0.472781] kobject: 'devices' (ffffffc079820d98): kobject_uevent_env: filter function caused the event to drop!  
> [    0.482937] kobject: 'drivers' (ffffffc079820e18): kobject_add_internal: parent: 'container', set: ''  
> [    0.492624] kobject: 'drivers' (ffffffc079820e18): kobject_uevent_env  
> [    0.499031] kobject: 'drivers' (ffffffc079820e18): kobject_uevent_env: filter function caused the event to drop!  
> [    0.509187] bus: 'container': registered  
> [    0.513093] device: 'container': device_add  
> [    0.517249] kobject: 'container' (ffffffc079824010): kobject_add_internal: parent: 'system', set: 'devices'  
> [    0.526937] kobject: 'container' (ffffffc079824010): kobject_uevent_env  
> [    0.533531] kobject: 'container' (ffffffc079824010): kobject_uevent_env: filter function caused the event to drop!  
> [    0.543843] kobject: 'devicetree' (ffffffc079820f98): kobject_add_internal: parent: 'firmware', set: ''  
> [    0.553718] kobject: 'devicetree' (ffffffc079820f98): kobject_uevent_env  
> [    0.560406] kobject: 'devicetree' (ffffffc079820f98): kobject_uevent_env: attempted to send uevent without kset!  
> [    0.570531] doing early, parsing ARGS: 'earlycon=owl_serial loglevel=8'  
> [    0.577124] doing early: earlycon='owl_serial'  
> [    0.581562] doing early: loglevel='8'  
> [    0.585187] doing core, parsing ARGS: 'earlycon=owl_serial loglevel=8'  
> [    0.591687] doing core: earlycon='owl_serial'  
> [    0.596031] doing core: loglevel='8'  
> [    0.599593] kobject: 'kernel' (ffffffc079825000): kobject_add_internal: parent: '', set: ''  
> [    0.608937] clocksource: jiffies: mask: 0xffffffff max_cycles: 0xffffffff, max_idle_ns: 7645041785100000 ns  
> [    0.618656] doing postcore, parsing ARGS: 'earlycon=owl_serial loglevel=8'  
> [    0.625499] doing postcore: earlycon='owl_serial'  
> [    0.630156] doing postcore: loglevel='8'  
> [    0.634062] device class 'bdi': registering  
> [    0.638218] kobject: 'bdi' (ffffffc079824218): kobject_add_internal: parent: 'class', set: 'class'  
> [    0.647156] kobject: 'bdi' (ffffffc079824218): kobject_uevent_env  
> [    0.653218] kobject: 'bdi' (ffffffc079824218): fill_kobj_path: path = '/class/bdi'  
> [    0.660781] kobject: 'mm' (ffffffc079825100): kobject_add_internal: parent: 'kernel', set: ''  
> [    0.669781] kobject: 'amba' (ffffffc079824418): kobject_add_internal: parent: 'bus', set: 'bus'  
> [    0.678437] kobject: 'amba' (ffffffc079824418): kobject_uevent_env  
> [    0.684593] kobject: 'amba' (ffffffc079824418): fill_kobj_path: path = '/bus/amba'  
> [    0.692124] kobject: 'devices' (ffffffc079825198): kobject_add_internal: parent: 'amba', set: ''  
> [    0.701406] kobject: 'devices' (ffffffc079825198): kobject_uevent_env  
> [    0.707812] kobject: 'devices' (ffffffc079825198): kobject_uevent_env: filter function caused the event to drop!  
> [    0.717968] kobject: 'drivers' (ffffffc079825218): kobject_add_internal: parent: 'amba', set: ''  
> [    0.727249] kobject: 'drivers' (ffffffc079825218): kobject_uevent_env  
> [    0.733656] kobject: 'drivers' (ffffffc079825218): kobject_uevent_env: filter function caused the event to drop!  
> [    0.743781] bus: 'amba': registered  
> [    0.747249] device class 'tty': registering  
> [    0.751406] kobject: 'tty' (ffffffc079824618): kobject_add_internal: parent: 'class', set: 'class'  
> [    0.760343] kobject: 'tty' (ffffffc079824618): kobject_uevent_env  
> [    0.766406] kobject: 'tty' (ffffffc079824618): fill_kobj_path: path = '/class/tty'  
> [    0.773937] doing arch, parsing ARGS: 'earlycon=owl_serial loglevel=8'  
> [    0.780437] doing arch: earlycon='owl_serial'  
> [    0.784781] doing arch: loglevel='8'  
> [    0.788343] vdso: 2 pages (1 code @ ffffff8008186000, 1 data @ ffffff80081c8000)  
> [    0.795874] DMA: preallocated 256 KiB pool for atomic allocations  
> [    0.801781] of_platform_bus_create() - skipping /memory, no compatible prop  
> [    0.808687] of_platform_bus_create() - skipping /chosen, no compatible prop  
> [    0.815656] OF: ** translation for device /interrupt-controller@e00f1000 **  
> [    0.822562] OF: bus is default (na=2, ns=2) on /  
> [    0.827156] OF: translating address: 00000000 e00f1000  
> [    0.832281] OF: reached root node  
> [    0.835562] OF: ** translation for device /interrupt-controller@e00f1000 **  
> [    0.842499] OF: bus is default (na=2, ns=2) on /  
> [    0.847093] OF: translating address: 00000000 e00f2000  
> [    0.852218] OF: reached root node  
> [    0.855499] of_irq_parse_one: dev=/interrupt-controller@e00f1000, index=0  
> [    0.862249] OF: ** translation for device /interrupt-controller@e00f1000 **  
> [    0.869187] OF: bus is default (na=2, ns=2) on /  
> [    0.873781] OF: translating address: 00000000 e00f1000  
> [    0.878906] OF: reached root node  
> [    0.882187] OF: ** translation for device /interrupt-controller@e00f1000 **  
> [    0.889124] OF: bus is default (na=2, ns=2) on /  
> [    0.893718] OF: translating address: 00000000 e00f2000  
> [    0.898843] OF: reached root node  
> [    0.902124] OF: ** translation for device /interrupt-controller@e00f1000 **  
> [    0.909062] OF: bus is default (na=2, ns=2) on /  
> [    0.913656] OF: translating address: 00000000 e00f1000  
> [    0.918781] OF: reached root node  
> [    0.922062] of_dma_get_range: no dma-ranges found for node(/interrupt-controller@e00f1000)  
> [    0.930312] platform e00f1000.interrupt-controller: device is not dma coherent  
> [    0.937499] platform e00f1000.interrupt-controller: device is not behind an iommu  
> [    0.944937] device: 'e00f1000.interrupt-controller': device_add  
> [    0.950843] kobject: 'e00f1000.interrupt-controller' (ffffffc079824820): kobject_add_internal: parent: 'platform', set: 'devices'  
> [    0.962437] bus: 'platform': add device e00f1000.interrupt-controller  
> [    0.968874] kobject: 'e00f1000.interrupt-controller' (ffffffc079824820): kobject_uevent_env  
> [    0.977187] kobject: 'e00f1000.interrupt-controller' (ffffffc079824820): fill_kobj_path: path = '/devices/platform/e00f1000.interrupt-controller'  
> [    0.990218] of_irq_parse_one: dev=/timer, index=0  
> [    0.994874]  intspec=1 intlen=12  
> [    0.998062]  intsize=3 intlen=12  
> [    1.001281] of_irq_parse_raw:  /interrupt-controller@e00f1000:00000001,0000000d,00000f08  
> [    1.009343] of_irq_parse_raw: ipar=/interrupt-controller@e00f1000, size=3  
> [    1.016093]  -> addrsize=0  
> [    1.018781]  -> got it !  
> [    1.021281] of_irq_parse_one: dev=/timer, index=1  
> [    1.025968]  intspec=1 intlen=12  
> [    1.029187]  intsize=3 intlen=12  
> [    1.032374] of_irq_parse_raw:  /interrupt-controller@e00f1000:00000001,0000000e,00000f08  
> [    1.040437] of_irq_parse_raw: ipar=/interrupt-controller@e00f1000, size=3  
> [    1.047218]  -> addrsize=0  
> [    1.049906]  -> got it !  
> [    1.052406] of_irq_parse_one: dev=/timer, index=2  
> [    1.057093]  intspec=1 intlen=12  
> [    1.060281]  intsize=3 intlen=12  
> [    1.063499] of_irq_parse_raw:  /interrupt-controller@e00f1000:00000001,0000000b,00000f08  
> [    1.071562] of_irq_parse_raw: ipar=/interrupt-controller@e00f1000, size=3  
> [    1.078312]  -> addrsize=0  
> [    1.080999]  -> got it !  
> [    1.083531] of_irq_parse_one: dev=/timer, index=3  
> [    1.088187]  intspec=1 intlen=12  
> [    1.091406]  intsize=3 intlen=12  
> [    1.094624] of_irq_parse_raw:  /interrupt-controller@e00f1000:00000001,0000000a,00000f08  
> [    1.102687] of_irq_parse_raw: ipar=/interrupt-controller@e00f1000, size=3  
> [    1.109437]  -> addrsize=0  
> [    1.112124]  -> got it !  
> [    1.114624] of_irq_parse_one: dev=/timer, index=4  
> [    1.119312]  intspec=1 intlen=12  
> [    1.122531]  intsize=3 intlen=12  
> [    1.125718] of_irq_parse_one: dev=/timer, index=0  
> [    1.130406]  intspec=1 intlen=12  
> [    1.133624]  intsize=3 intlen=12  
> [    1.136812] of_irq_parse_raw:  /interrupt-controller@e00f1000:00000001,0000000d,00000f08  
> [    1.144874] of_irq_parse_raw: ipar=/interrupt-controller@e00f1000, size=3  
> [    1.151624]  -> addrsize=0  
> [    1.154312]  -> got it !  
> [    1.156843] of_irq_parse_one: dev=/timer, index=1  
> [    1.161531]  intspec=1 intlen=12  
> [    1.164718]  intsize=3 intlen=12  
> [    1.167937] of_irq_parse_raw:  /interrupt-controller@e00f1000:00000001,0000000e,00000f08  
> [    1.175999] of_irq_parse_raw: ipar=/interrupt-controller@e00f1000, size=3  
> [    1.182749]  -> addrsize=0  
> [    1.185437]  -> got it !  
> [    1.187968] of_irq_parse_one: dev=/timer, index=2  
> [    1.192624]  intspec=1 intlen=12  
> [    1.195843]  intsize=3 intlen=12  
> [    1.199031] of_irq_parse_raw:  /interrupt-controller@e00f1000:00000001,0000000b,00000f08  
> [    1.207093] of_irq_parse_raw: ipar=/interrupt-controller@e00f1000, size=3  
> [    1.213874]  -> addrsize=0  
> [    1.216562]  -> got it !  
> [    1.219062] of_irq_parse_one: dev=/timer, index=3  
> [    1.223749]  intspec=1 intlen=12  
> [    1.226968]  intsize=3 intlen=12  
> [    1.230156] of_irq_parse_raw:  /interrupt-controller@e00f1000:00000001,0000000a,00000f08  
> [    1.238218] of_irq_parse_raw: ipar=/interrupt-controller@e00f1000, size=3  
> [    1.244968]  -> addrsize=0  
> [    1.247656]  -> got it !  
> [    1.250187] of_dma_get_range: no dma-ranges found for node(/timer)  
> [    1.256343] platform timer: device is not dma coherent  
> [    1.261437] platform timer: device is not behind an iommu  
> [    1.266812] device: 'timer': device_add  
> [    1.270624] kobject: 'timer' (ffffffc079824a20): kobject_add_internal: parent: 'platform', set: 'devices'  
> [    1.280156] bus: 'platform': add device timer  
> [    1.284499] kobject: 'timer' (ffffffc079824a20): kobject_uevent_env  
> [    1.290749] kobject: 'timer' (ffffffc079824a20): fill_kobj_path: path = '/devices/platform/timer'  
> [    1.299593] doing subsys, parsing ARGS: 'earlycon=owl_serial loglevel=8'  
> [    1.306249] doing subsys: earlycon='owl_serial'  
> [    1.310749] doing subsys: loglevel='8'  
> [    1.314499] device: 'cpu0': device_add  
> [    1.318218] kobject: 'cpu0' (ffffffc07ffe1468): kobject_add_internal: parent: 'cpu', set: 'devices'  
> [    1.327218] bus: 'cpu': add device cpu0  
> [    1.331031] kobject: 'cpu0' (ffffffc07ffe1468): kobject_uevent_env  
> [    1.337187] kobject: 'cpu0' (ffffffc07ffe1468): fill_kobj_path: path = '/devices/system/cpu/cpu0'  
> [    1.346156] device class 'misc': registering  
> [    1.350281] kobject: 'misc' (ffffffc079824e18): kobject_add_internal: parent: 'class', set: 'class'  
> [    1.359281] kobject: 'misc' (ffffffc079824e18): kobject_uevent_env  
> [    1.365437] kobject: 'misc' (ffffffc079824e18): fill_kobj_path: path = '/class/misc'  
> [    1.373156] device class 'power_supply': registering  
> [    1.378093] kobject: 'power_supply' (ffffffc079803018): kobject_add_internal: parent: 'class', set: 'class'  
> [    1.387812] kobject: 'power_supply' (ffffffc079803018): kobject_uevent_env  
> [    1.394656] kobject: 'power_supply' (ffffffc079803018): fill_kobj_path: path = '/class/power_supply'  
> [    1.403749] doing fs, parsing ARGS: 'earlycon=owl_serial loglevel=8'  
> [    1.410093] doing fs: earlycon='owl_serial'  
> [    1.414249] doing fs: loglevel='8'  
> [    1.417656] clocksource: Switched to clocksource arch_sys_counter  
> [    1.423718] device class 'mem': registering  
> [    1.427843] kobject: 'mem' (ffffffc079827018): kobject_add_internal: parent: 'class', set: 'class'  
> [    1.436781] kobject: 'mem' (ffffffc079827018): kobject_uevent_env  
> [    1.442843] kobject: 'mem' (ffffffc079827018): fill_kobj_path: path = '/class/mem'  
> [    1.450374] device: 'null': device_add  
> [    1.454124] kobject: 'virtual' (ffffffc079826080): kobject_add_internal: parent: 'devices', set: ''  
> [    1.463656] kobject: 'mem' (ffffffc079826100): kobject_add_internal: parent: 'virtual', set: '(null)'  
> [    1.472843] kobject: 'null' (ffffffc079827210): kobject_add_internal: parent: 'mem', set: 'devices'  
> [    1.481843] kobject: 'null' (ffffffc079827210): kobject_uevent_env  
> [    1.487999] kobject: 'null' (ffffffc079827210): fill_kobj_path: path = '/devices/virtual/mem/null'  
> [    1.496937] device: 'zero': device_add  
> [    1.500656] kobject: 'zero' (ffffffc079827410): kobject_add_internal: parent: 'mem', set: 'devices'  
> [    1.509656] kobject: 'zero' (ffffffc079827410): kobject_uevent_env  
> [    1.515812] kobject: 'zero' (ffffffc079827410): fill_kobj_path: path = '/devices/virtual/mem/zero'  
> [    1.524749] device: 'full': device_add  
> [    1.528468] kobject: 'full' (ffffffc079827610): kobject_add_internal: parent: 'mem', set: 'devices'  
> [    1.537468] kobject: 'full' (ffffffc079827610): kobject_uevent_env  
> [    1.543624] kobject: 'full' (ffffffc079827610): fill_kobj_path: path = '/devices/virtual/mem/full'  
> [    1.552562] device: 'random': device_add  
> [    1.556468] kobject: 'random' (ffffffc079827810): kobject_add_internal: parent: 'mem', set: 'devices'  
> [    1.565656] kobject: 'random' (ffffffc079827810): kobject_uevent_env  
> [    1.571968] kobject: 'random' (ffffffc079827810): fill_kobj_path: path = '/devices/virtual/mem/random'  
> [    1.581249] device: 'urandom': device_add  
> [    1.585249] kobject: 'urandom' (ffffffc079827a10): kobject_add_internal: parent: 'mem', set: 'devices'  
> [    1.594499] kobject: 'urandom' (ffffffc079827a10): kobject_uevent_env  
> [    1.600937] kobject: 'urandom' (ffffffc079827a10): fill_kobj_path: path = '/devices/virtual/mem/urandom'  
> [    1.610374] device: 'kmsg': device_add  
> [    1.614093] kobject: 'kmsg' (ffffffc079827c10): kobject_add_internal: parent: 'mem', set: 'devices'  
> [    1.623124] kobject: 'kmsg' (ffffffc079827c10): kobject_uevent_env  
> [    1.629249] kobject: 'kmsg' (ffffffc079827c10): fill_kobj_path: path = '/devices/virtual/mem/kmsg'  
> [    1.638187] device: 'tty': device_add  
> [    1.641843] kobject: 'tty' (ffffffc079826280): kobject_add_internal: parent: 'virtual', set: '(null)'  
> [    1.651031] kobject: 'tty' (ffffffc079827e10): kobject_add_internal: parent: 'tty', set: 'devices'  
> [    1.659937] kobject: 'tty' (ffffffc079827e10): kobject_uevent_env  
> [    1.665999] kobject: 'tty' (ffffffc079827e10): fill_kobj_path: path = '/devices/virtual/tty/tty'  
> [    1.674781] device: 'console': device_add  
> [    1.678749] kobject: 'console' (ffffffc079829010): kobject_add_internal: parent: 'tty', set: 'devices'  
> [    1.688031] kobject: 'console' (ffffffc079829010): kobject_uevent_env  
> [    1.694437] kobject: 'console' (ffffffc079829010): fill_kobj_path: path = '/devices/virtual/tty/console'  
> [    1.703937] doing device, parsing ARGS: 'earlycon=owl_serial loglevel=8'  
> [    1.710562] doing device: earlycon='owl_serial'  
> [    1.715062] doing device: loglevel='8'  
> [    1.718812] bus: 'platform': add driver alarmtimer  
> [    1.723562] kobject: 'alarmtimer' (ffffffc079828600): kobject_add_internal: parent: 'drivers', set: 'drivers'  
> [    1.733437] kobject: 'alarmtimer' (ffffffc079828600): kobject_uevent_env  
> [    1.740124] kobject: 'alarmtimer' (ffffffc079828600): fill_kobj_path: path = '/bus/platform/drivers/alarmtimer'  
> [    1.750156] Registering platform device 'alarmtimer'. Parent at platform  
> [    1.756843] device: 'alarmtimer': device_add  
> [    1.761093] kobject: 'alarmtimer' (ffffffc079829220): kobject_add_internal: parent: 'platform', set: 'devices'  
> [    1.771062] bus: 'platform': add device alarmtimer  
> [    1.775812] kobject: 'alarmtimer' (ffffffc079829220): kobject_uevent_env  
> [    1.782499] kobject: 'alarmtimer' (ffffffc079829220): fill_kobj_path: path = '/devices/platform/alarmtimer'  
> [    1.792218] bus: 'platform': driver_probe_device: matched device alarmtimer with driver alarmtimer  
> [    1.801124] bus: 'platform': really_probe: probing driver alarmtimer with device alarmtimer  
> [    1.809437] devices_kset: Moving alarmtimer to end of list  
> [    1.814906] driver: 'alarmtimer': driver_bound: bound to device 'alarmtimer'  
> [    1.821937] bus: 'platform': really_probe: bound device alarmtimer to driver alarmtimer  
> [    1.829999] workingset: timestamp_bits=60 max_order=19 bucket_order=0  
> [    1.836312] Failed to find cpu0 device node  
> [    1.840468] Unable to detect cache hierarchy from DT for CPU 0  
> [    1.846281] bus: 'platform': add driver gpio-clk  
> [    1.850874] kobject: 'gpio-clk' (ffffffc079828900): kobject_add_internal: parent: 'drivers', set: 'drivers'  
> [    1.860593] kobject: 'gpio-clk' (ffffffc079828900): kobject_uevent_env  
> [    1.867093] kobject: 'gpio-clk' (ffffffc079828900): fill_kobj_path: path = '/bus/platform/drivers/gpio-clk'  
> [    1.876781] doing late, parsing ARGS: 'earlycon=owl_serial loglevel=8'  
> [    1.883281] doing late: earlycon='owl_serial'  
> [    1.887624] doing late: loglevel='8'  
> [    1.891187] device: 'cpu_dma_latency': device_add  
> [    1.895843] kobject: 'misc' (ffffffc079826780): kobject_add_internal: parent: 'virtual', set: '(null)'  
> [    1.905124] kobject: 'cpu_dma_latency' (ffffffc079829610): kobject_add_internal: parent: 'misc', set: 'devices'  
> [    1.915187] kobject: 'cpu_dma_latency' (ffffffc079829610): kobject_uevent_env  
> [    1.922281] kobject: 'cpu_dma_latency' (ffffffc079829610): fill_kobj_path: path = '/devices/virtual/misc/cpu_dma_latency'  
> [    1.933218] device: 'network_latency': device_add  
> [    1.937874] kobject: 'network_latency' (ffffffc079829810): kobject_add_internal: parent: 'misc', set: 'devices'  
> [    1.947937] kobject: 'network_latency' (ffffffc079829810): kobject_uevent_env  
> [    1.955031] kobject: 'network_latency' (ffffffc079829810): fill_kobj_path: path = '/devices/virtual/misc/network_latency'  
> [    1.965968] device: 'network_throughput': device_add  
> [    1.970906] kobject: 'network_throughput' (ffffffc079829a10): kobject_add_internal: parent: 'misc', set: 'devices'  
> [    1.981218] kobject: 'network_throughput' (ffffffc079829a10): kobject_uevent_env  
> [    1.988593] kobject: 'network_throughput' (ffffffc079829a10): fill_kobj_path: path = '/devices/virtual/misc/network_throughput'  
> [    2.000031] device: 'memory_bandwidth': device_add  
> [    2.004781] kobject: 'memory_bandwidth' (ffffffc079829c10): kobject_add_internal: parent: 'misc', set: 'devices'  
> [    2.014937] kobject: 'memory_bandwidth' (ffffffc079829c10): kobject_uevent_env  
> [    2.022124] kobject: 'memory_bandwidth' (ffffffc079829c10): fill_kobj_path: path = '/devices/virtual/misc/memory_bandwidth'  
> [    2.033437] Warning: unable to open an initial console.  
> [    2.038531] Freeing unused kernel memory: 120K (ffffff80081a3000 - ffffff80081c1000)  
> [    2.046124] This architecture does not have kernel memory protection.  
> [    2.052593] Kernel panic - not syncing: No working init found.  Try passing init= option to kernel. See Linux Documentation/init.txt for guidance.  
> [    2.065624] Kernel Offset: disabled  
> [    2.069093] Memory Limit: none  
> [    2.072124] ---[ end Kernel panic - not syncing: No working init found.  Try passing init= option to kernel. See Linux Documentation/init.txt for guidance.

  
_原创文章，转发请注明出处。蜗窝科技，[www.wowotech.net](http://www.wowotech.net/linux_application/kernel_debug_enable.html)。_

标签: [debug](http://www.wowotech.net/tag/debug) [Linux](http://www.wowotech.net/tag/Linux) [Kernel](http://www.wowotech.net/tag/Kernel) [printk](http://www.wowotech.net/tag/printk) [pr_debug](http://www.wowotech.net/tag/pr_debug) [dev_dbg](http://www.wowotech.net/tag/dev_dbg)

[![](http://www.wowotech.net/content/uploadfile/201605/ef3e1463542768.png)](http://www.wowotech.net/support_us.html)

« [X-015-KERNEL-ARM generic timer driver的移植](http://www.wowotech.net/x_project/generic_timer_porting.html) | [X-014-KERNEL-ARM GIC driver的移植](http://www.wowotech.net/x_project/gic_driver_porting.html)»

**评论：**

**Jere**  
2019-07-17 21:42

@wowo，您好，  
  
我在在kernel/Makefile中，加入KCFLAGS += -DDEBUG，然后make编译出zImage，烧到板子里，启动会停在starting kernel就没任何打印了，不加KCFLAGS += -DDEBUG的zImage可以正常启动。您有没有遇到这种情况？  
谢谢

[回复](http://www.wowotech.net/linux_application/kernel_debug_enable.html#comment-7537)

**大鱼**  
2019-08-07 11:28

@Jere：打印太多了？

[回复](http://www.wowotech.net/linux_application/kernel_debug_enable.html#comment-7573)

**L**  
2019-05-10 19:46

我在linux的顶层Makefile中  
KBUILD_CFLAGS 后添加 -DDEBUG  
由于linux里面的c文件有很多定义了DEBUG，编译时会报重复定义DEBUG的错误，怎么比较好的解决呢，谢谢

[回复](http://www.wowotech.net/linux_application/kernel_debug_enable.html#comment-7407)

**gongcm**  
2019-05-31 13:39

@L：直接在顶层makefile中 加上 KCFLAGS += -DDEBUG

[回复](http://www.wowotech.net/linux_application/kernel_debug_enable.html#comment-7449)

**uxer**  
2019-07-10 10:17

@gongcm：在kernel/Makefile中，加入KCFLAGS += -DDEBUG，还是会报错，重复定义DEBUG

[回复](http://www.wowotech.net/linux_application/kernel_debug_enable.html#comment-7526)

**EP**  
2019-09-03 17:52

@uxer：KBUILD_CFLAGS   += -w  
但一般打印太多，开机卡死，最好不要在顶层的Makefile中添加

[回复](http://www.wowotech.net/linux_application/kernel_debug_enable.html#comment-7642)

**肤了个浅**  
2017-11-08 10:38

去掉DEBUG这个宏之后，dev_dbg这个函数还会编译进去吗？

[回复](http://www.wowotech.net/linux_application/kernel_debug_enable.html#comment-6183)

**[wowo](http://www.wowotech.net/)**  
2017-11-09 18:19

@肤了个浅：不会了，你可以去看代码:-)

[回复](http://www.wowotech.net/linux_application/kernel_debug_enable.html#comment-6187)

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
    
    - Shiina  
        [一个电路（circuit）中，由于是回路，所以用电势差的概念...](http://www.wowotech.net/basic_subject/voltage.html#8926)
    - Shiina  
        [其中比较关键的点是相对位置概念和点电荷的静电势能计算。](http://www.wowotech.net/basic_subject/voltage.html#8925)
    - leelockhey  
        [你这是哪个内核版本](http://www.wowotech.net/pm_subsystem/generic_pm_architecture.html#8924)
    - ja  
        [@dream：我看完這段也有相同的想法，引用 @dream ...](http://www.wowotech.net/kernel_synchronization/spinlock.html#8922)
    - 元神高手  
        [围观首席power managerment专家](http://www.wowotech.net/pm_subsystem/device_driver_pm.html#8921)
    - 十七  
        [内核空间的映射在系统启动时就已经设定好，并且在所有进程的页表...](http://www.wowotech.net/process_management/context-switch-arch.html#8920)
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
    
    - [显示技术介绍(3)_CRT技术](http://www.wowotech.net/display/crt_intro.html)
    - [显示技术介绍(1)_概述](http://www.wowotech.net/display/display_tech_overview.html)
    - [Linux serial framework(1)_概述](http://www.wowotech.net/comm/serial_overview.html)
    - [教育漫谈(1)_《乌合之众》摘录](http://www.wowotech.net/tech_discuss/jymt1.html)
    - [ACCESS_ONCE宏定义的解释](http://www.wowotech.net/process_management/access-once.html)
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