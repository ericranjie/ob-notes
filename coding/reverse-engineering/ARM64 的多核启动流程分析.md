# 

人人极客社区

 _2022年03月09日 08:28_

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

工作中遇到的多核 ARM CPU 越来越多，总结分享一些多核启动的知识，希望能帮助更多小伙伴。  

在 ARM64 架构下如果想要启动多核，有 spin-table 和 psci 两种方式，下面针对这两种启动流程进行分析。

## 代码版本

- boot-wrapper-aarch64 version : 28932c41e14d730b8b9a7310071384178611fb32
    
- linux v5.14
    

## 多核 CPU 的启动方式

嵌入式系统的启动的基本流程是先运行 `bootloader` ，然后由 `bootloader` 引导启动 kernel，这里无论启动的是 rt-thread 或者是 linux 原理都是一样的。

上电后所有的 `CPU` 都会从 `bootrom` 里面开始执行代码，为了防止并发造成的一些问题，需要将除了 `primary cpu` 以外的 `cpu` 拦截下来，这样才能保证启动的顺序是可控的。

## spin-table 启动方法

在启动的过程中，`bootloader` 中有一道栅栏，它拦住了除了 `cpu0` 外的其他 `cpu`。`cpu0` 直接往下运行，进行设备初始化以及运行 `Kernel`。其他 `cpu0` 则在栅栏外进入睡眠状态。

`cpu0` 在初始化 `smp` 的时候，会在 `cpu-release-addr` 里面填入一个地址并唤醒其他 `cpu`。这时睡眠的 `cpu` 接受到信号，醒来的时候会先检查 `cpu-release-addr` 这个地址里面的数据是不是有效。如果该地址是有效的（非 0 ），意味着自己需要真正开始启动了，接下来他会跳转到。

下面我们看看 `arm64` 里面的实现，在 `arch/arm64/boot/dts/xxx.dts` 中有如下描述：

```
1cpu@0 {2    device_type = "cpu";3    compatible = "arm,armv8";4    reg = <0x0 0x0="">;5    enable-method = "spin-table"; /* 选择使用 spin-table 方式启动  */6    cpu-release-addr = <0x0 0x8000fff8="">;7};
```

  

在 `arch/arm64/kernel/smp_spin_table.c` 中处理了向其他 cpu 发送信号的方法：

1、先是获取 release_addr 的虚拟地址

2、向该地址写入从 cpu 的入口地址

3、通过 sev() 指令唤醒其他 cpu

  

```
 1static int smp_spin_table_cpu_prepare(unsigned int cpu) 2{ 3    __le64 __iomem *release_addr; 4    phys_addr_t pa_holding_pen = __pa_symbol(function_nocfi(secondary_holding_pen)); 5 6    if (!cpu_release_addr[cpu]) 7        return -ENODEV; 8 9    /*10     * The cpu-release-addr may or may not be inside the linear mapping.11     * As ioremap_cache will either give us a new mapping or reuse the12     * existing linear mapping, we can use it to cover both cases. In13     * either case the memory will be MT_NORMAL.14     */15    release_addr = ioremap_cache(cpu_release_addr[cpu],16                     sizeof(*release_addr));17    if (!release_addr)18        return -ENOMEM;1920    /*21     * We write the release address as LE regardless of the native22     * endianness of the kernel. Therefore, any boot-loaders that23     * read this address need to convert this address to the24     * boot-loader's endianness before jumping. This is mandated by25     * the boot protocol.26     */27    writeq_relaxed(pa_holding_pen, release_addr);28    dcache_clean_inval_poc((__force unsigned long)release_addr,29                (__force unsigned long)release_addr +30                    sizeof(*release_addr));3132    /*33     * Send an event to wake up the secondary CPU.34     */35    sev();3637    iounmap(release_addr);3839    return 0;40}
```

  

`Bootloader` 部分以 `boot-wrapper-aarch64` 中的代码做示例，非主 CPU 会轮询检查 mbox（其地址等同cpu-release-addr）中的值，当其值为 0 的时候继续睡眠，否则就跳转到内核执行，代码如下所示：

```
 1/** 2 * Wait for an address to appear in mbox, and jump to it. 3 * 4 * @mbox: location to watch 5 * @invalid: value of an invalid address, 0 or -1 depending on the boot method 6 * @is_entry: when true, pass boot parameters to the kernel, instead of 0 7 */ 8void __noreturn spin(unsigned long *mbox, unsigned long invalid, int is_entry) 9{10    unsigned long addr = invalid;1112    while (addr == invalid) {13        wfe();14        addr = *mbox;15    }1617    if (is_entry)18#ifdef KERNEL_3219        jump_kernel(addr, 0, ~0, (unsigned long)&dtb, 0);20#else21        jump_kernel(addr, (unsigned long)&dtb, 0, 0, 0);22#endif2324    jump_kernel(addr, 0, 0, 0, 0);2526    unreachable();27}2829/**30 * Primary CPU finishes platform initialisation and jumps to the kernel.31 * Secondaries are parked, waiting for their mbox to contain a valid address.32 *33 * @cpu: logical CPU number34 * @mbox: location to watch35 * @invalid: value of an invalid address, 0 or -1 depending on the boot method36 */37void __noreturn first_spin(unsigned int cpu, unsigned long *mbox,38               unsigned long invalid)39{40    if (cpu == 0) {41        init_platform();4243        *mbox = (unsigned long)&entrypoint;44        sevl();45        spin(mbox, invalid, 1);46    } else {47        *mbox = invalid;48        spin(mbox, invalid, 0);49    }5051    unreachable();52}
```

## PSCI 启动方法

另外一种 enable-method 就是 PSCI，依旧先从 kernel 开始分析。先看 `arch/arm64/boot/dts/mediatek/mt8173.dtsi` 文件，里面 `cpu` 节点选择了PSCI 的方法：

```
1cpu0: cpu@0 {2    compatible = "arm,cortex-a53";3    device_type = "cpu";4    enable-method = "psci";    /* 启动方式选择 PSCI */5    operating-points-v2 = <&cpu_opp_table>;6    reg = <0x0>;7    cpu-idle-states = <&CPU_SLEEP_0>;8};
```

  

并且有一个 PSCI 的节点：

```
1psci {2    compatible = "arm,psci-1.0", "arm,psci-0.2", "arm,psci";3    method = "smc";4    cpu_suspend   = <0x84000001>;5    cpu_off          = <0x84000002>;6    cpu_on          = <0x84000003>;7};
```

  

**在 `PSCI` 中的节点详细说明请参考文档：**kernel/Documentation/devicetree/bindings/arm/psci.txt。在此仅说一下 method 字段。该字段有两个可选值：smc 和 hvc。表示调用 PSCI 功能使用什么指令。smc、hvc、svc 这些指令都是由低运行级别向更高级别请求服务的指令。

和系统调用一样。调用了该指令，cpu 会进入异常切入更高的权限。异常处理程序根据下面传上来的参数决定给予什么服务，smc 陷入 EL3，hvc 陷入 EL2，svc 陷入EL1。在 ARMv8 里面，EL3 总是是 secure 状态，EL2 是虚拟机管态，EL1 是普通的系统态。

接下来可以看看 `arch/arm64/kernel/psci.c` 里面的代码，psci_ops.cpu_on 最终调用 smc call：

```
 1static int cpu_psci_cpu_boot(unsigned int cpu) 2{ 3    phys_addr_t pa_secondary_entry = __pa_symbol(function_nocfi(secondary_entry)); 4    int err = psci_ops.cpu_on(cpu_logical_map(cpu), pa_secondary_entry); 5    if (err) 6        pr_err("failed to boot CPU%d (%d)\n", cpu, err); 7 8    return err; 9}1011static unsigned long __invoke_psci_fn_smc(unsigned long function_id,12            unsigned long arg0, unsigned long arg1,13            unsigned long arg2)14{15    struct arm_smccc_res res;1617    arm_smccc_smc(function_id, arg0, arg1, arg2, 0, 0, 0, 0, &res);18    return res.a0;19}
```

  

Bootloader 以 `boot-wrapper-aarch64` 作分析，看 psci.c 里的 psci_call 实现函数，通过 fid 与 PSCI_CPU_OFF 和 PSCI_CPU_ON 相比，找出需要执行的动作：

```
 1long psci_call(unsigned long fid, unsigned long arg1, unsigned long arg2) 2{ 3    switch (fid) { 4    case PSCI_CPU_OFF: 5        return psci_cpu_off(); 6 7    case PSCI_CPU_ON_64: 8        return psci_cpu_on(arg1, arg2); 910    default:11        return PSCI_RET_NOT_SUPPORTED;12    }13}
```

  

当然 `boot-wrapper-aarch64` 里也需要同样的定义：

```
1#define PSCI_CPU_OFF        0x840000022#define PSCI_CPU_ON_32      0x840000033#define PSCI_CPU_ON_64      0xc4000003
```

  

`boot-wrapper-aarch64` 按照和 `kernel` 约定的好参数列表，为目标 `cpu` 设置好跳转地址，然后返回到 `kernel`  执行，下面给出关键代码说明：

```
 1static int psci_cpu_on(unsigned long target_mpidr, unsigned long address) 2{ 3    int ret; 4    unsigned int cpu = find_logical_id(target_mpidr); 5    unsigned int this_cpu = this_cpu_logical_id(); 6 7    if (cpu == MPIDR_INVALID) 8        return PSCI_RET_INVALID_PARAMETERS; 910    bakery_lock(branch_table_lock, this_cpu);11    ret = psci_store_address(cpu, address);   /* 写入启动地址  */12    bakery_unlock(branch_table_lock, this_cpu);1314    return ret;15}
```

## 总结

目前比较主流的多核启动方式是 PSCI，一般正式的产品都有 ATF，通过 PSCI 可以实现 CPU 的开启关闭以及挂起等操作。在实际的移植工作过程中，如果有带有 ATF 的 bootloader 那多核移植就相对容易很多，如果没有的话，也可以采用 spin_table 的方式来启动多核。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E "虚线阴影分割线")

![](https://res.wx.qq.com/t/fed_upload/b39ef69e-c4d6-4169-8612-5f00a84860e7/wx-avatar-default.svg)

公众号

5T技术资源大放送！包括但不限于：C/C++，Arm, Linux，Android，人工智能，单片机，树莓派，等等。在上面的【人人都是极客】公众号内回复「peter」，即可免费获取！！  

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E) **记得点击****分享****、****赞****和****在看****，给我充点儿电吧**

Linux93

启动2

ARM6

Linux · 目录

上一篇Kernel同步机制的底层实现下一篇一文搞懂 | Linux 时间子系统

阅读 2969

​

写留言

**留言 10**

- 道哥#IOT物联网小镇
    
    2022年3月9日
    
    赞1
    
    眼睛说:我学废了![[呲牙]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![[呲牙]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
    人人极客社区
    
    作者2022年3月9日
    
    赞1
    
    哈哈 扶我起来
    
- 程磊
    
    2022年3月9日
    
    赞1
    
    因为内核只能初始化一次，所以只能先启动一个CPU，然后其他的CPU就可以同时启动了，完成一些CPU相关的初始化工作，然后运行idle进程。
    
    人人极客社区
    
    作者2022年3月9日
    
    赞
    
    ![[强]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![[强]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)大佬
    
- 张贤斌
    
    2022年3月9日
    
    赞1
    
    Arm Trusted Firmware
    
    人人极客社区
    
    作者2022年3月9日
    
    赞
    
    ![[强]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
- NNTD
    
    2022年3月28日
    
    赞
    
    很多时候，原生的dts看不出启动方式，需要uboot在启动内核阶段往dts插入属性
    
- 风二中
    
    2022年3月9日
    
    赞
    
    没太明白，cpu是顺序启动的吗，比如先cpu0，再1，再2…n
    
- Jarvis
    
    2022年3月9日
    
    赞
    
    ATF？
    
    人人极客社区
    
    作者2022年3月9日
    
    赞
    
    嗯
    

已无更多数据

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/9sNwsXcN68pL55XIyzTrCHZTbIUdTibQcuzuCaYeGTXNMyn6ACmicUrpoDC0xZSap46XJ59sKysPg9Rg379f32cA/300?wx_fmt=png&wxfrom=18)

人人极客社区

1625

10

写留言

**留言 10**

- 道哥#IOT物联网小镇
    
    2022年3月9日
    
    赞1
    
    眼睛说:我学废了![[呲牙]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![[呲牙]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
    人人极客社区
    
    作者2022年3月9日
    
    赞1
    
    哈哈 扶我起来
    
- 程磊
    
    2022年3月9日
    
    赞1
    
    因为内核只能初始化一次，所以只能先启动一个CPU，然后其他的CPU就可以同时启动了，完成一些CPU相关的初始化工作，然后运行idle进程。
    
    人人极客社区
    
    作者2022年3月9日
    
    赞
    
    ![[强]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![[强]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)大佬
    
- 张贤斌
    
    2022年3月9日
    
    赞1
    
    Arm Trusted Firmware
    
    人人极客社区
    
    作者2022年3月9日
    
    赞
    
    ![[强]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
- NNTD
    
    2022年3月28日
    
    赞
    
    很多时候，原生的dts看不出启动方式，需要uboot在启动内核阶段往dts插入属性
    
- 风二中
    
    2022年3月9日
    
    赞
    
    没太明白，cpu是顺序启动的吗，比如先cpu0，再1，再2…n
    
- Jarvis
    
    2022年3月9日
    
    赞
    
    ATF？
    
    人人极客社区
    
    作者2022年3月9日
    
    赞
    
    嗯
    

已无更多数据