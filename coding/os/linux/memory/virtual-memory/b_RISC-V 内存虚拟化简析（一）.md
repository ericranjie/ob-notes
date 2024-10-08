Original 晓泰 泰晓科技
_2023年05月16日 15:09_ _广东_

> Corrector: TinyCorrect v0.1-rc3 - \[comments codeinline epw\]\
> Author:   XiakaiPan 13212017962@163.com\
> Date:     2022/08/12\
> Revisor:  walimis, Falcon\
> Project:  RISC-V Linux 内核剖析\
> Proposal: RISC-V 虚拟化技术调研与分析\
> Sponsor:  PLCT Lab, ISCAS

本周继续连载 RISC-V 虚拟化系列文章，记得收藏分享+关注，写文章领补贴：gitee.com/tinylab/riscv-linux

该活动统一采用泰晓社区自研 Linux Lab 开源实验环境，也可选用免装即插即跑 Linux Lab Disk (https://tinylab.org/linux-lab-disk)，某宝检索“泰晓 Linux”可找到。**Linux Lab v1.1 Inside —— 内核开发从未像今天这般简单！**

# RISC-V 内存虚拟化简析（一）

## 前言

本文首先简要介绍了 RISC-V 特权指令中的基本内存管理指令，包括 `SFENCE.VMA` (in S-Extension), `HFENCE.VVMA` 和 `HFENCE.GVMA` (in H-Extension) 等，随后分析了它们在指令集模拟器 Spike 中的实现，最后对与内存管理密切相关的两组 CSR（控制与状态寄存器）`xstatus` 和 `xatp` 的结构和功能进行了分析。

## RISC-V 特权指令集总览

RISC-V 特权指令集如下表所示，包括 Trap 返回指令（`sret`, `mret`）、中断管理指令、S-Mode 内存管理指令和 H-Mode 指令（包含内存管理指令和加载、保存指令），其中 S-Mode 和 H-Mode 的内存管理指令功能类似。
![Image](https://mmbiz.qpic.cn/mmbiz_png/XXJQJDtx0eZCqqGC5Fr3ecA7jGCZiag9CuRxmicjibRWia0MxQRaN3E88B5fATF1bVebzQpGTfeu71gjLgP5UbNmrA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

priv-insts

### M-Level 指令

`sret` 和 `mret` 指令涉及从 Trap 跳转处理中返回，其在虚拟化中的作用，可以参考 此文 的 **Trap 与虚拟化** 一节。

`WFI`(`Waiting For Interrupt`) 指令有时被实现为无操作 `NOP` 指令。或者使 CPU 进入低功耗休眠状态，等待中断唤醒。

### S-Level 指令

`SFENCE.VMA` 是 S 模式中的内存管理指令，而 `SINVAL.VMA`, `SFENCE.W.INVAL`, `SFENCE.INVAL.IR` 是其扩展指令，用于实现更细粒度的内存管理。

### H 扩展指令

H 扩展的指令包括内存管理和数据加载存储指令两部分，其中，内存管理指令可以视为 S 模式的内存管理指令的变体，它们的功能相似；而 H 扩展的 `Load`, `Store` 指令与基础指令集里对应指令有所不同（指令生效的特权模式不同），而机器所在的特权模式（M，HS，VS，U）可通过特定 CSR 的特定位确定，故而其具体实现往往涉及特定 CSR 的读写和判断。

## 内存管理指令

### S 模式内存管理指令

#### SFENCE.VMA 解读

![Image](https://mmbiz.qpic.cn/mmbiz_png/XXJQJDtx0eZCqqGC5Fr3ecA7jGCZiag9CmsiaDly52eUHts4QyMibARF9muiaGYqOg1gPKghUBWPxwfZsc0wYnhtEg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)
sfence.vma

该指令有两个作用：序列化对特定 PTE (Page Table Entry) 的读写；无效化对应于特定地址空间（Address Space, AS）的特定虚拟地址（Virtual Address, VA）的 PTE。

该指令有两个源操作数：`rs1` 和 `rs2`。`rs1` 用于指定虚拟地址（Virtual ADDRess），`rs2` 用于指定地址空间（Address Space IDentifier）。

- 若 ，该指令将会对所有页表的页表项进行操作，若 ，该指令将对由  指定的虚拟地址对应的 PTE 进行操作。
- 若 ，该指令将会对所有地址空间进行操作，若 ，该指令将对由  指定的 AS 进行操作。

表示 0。

具体功能如下表所示：

|操作（Order/Invalidate）|||||
|---|---|---|---|---|
|Order Read/Write to **page table entries**|corresponding to virtual address|any level of the page tables|for address space identified by|for all spaces|
|Invalidate **address-translation cache entries**|that contain leaf page table entries corresponding to virtual address|invalidate all entries|that match address space identified by|for all address space|

#### Spike 模拟器实现

##### SFENCE.VMA

`// riscv/insns/sfence_vma.h: line 1   require_extension('S');     // 需要支持 S 扩展   require_impl(IMPL_MMU);     // 需要 MMU 实现的支持   if (STATE.v) {              // 如果处于虚拟化状态（V=1），则机器的特权级是 VS（S）或 VU（U），意味着此处执行的是虚拟机中的指令       // 若当前处于 VU-Mode 或 hstatus.VTVM=1，则意味着需要对 VM 中的指令进行 trap 处理，即调用 require_novirt()       if (STATE.prv == PRV_U || get_field(STATE.hstatus->read(), HSTATUS_VTVM))           require_novirt();       } else {       /* 若机器并未支持虚拟化或处于虚拟化后的 HS-Mode 或 M-Mode，则根据 mstatus.TVM 判断处理该条指令所需的特权级：        * mstatus.TVM=1，trap 到 M-Mode 进行处理        * mstatus.TVM=0，trap 到 S-Mode 进行处理        */       require_privilege(get_field(STATE.mstatus->read(), MSTATUS_TVM) ? PRV_M : PRV_S);   }   // 此处采取了最简化版的 sfence.vma 功能实现：冲刷掉所有的 TLB 内容   MMU.flush_tlb();   `

##### require_novirt()

`require_novirt()` 定义如下，该函数完成对 VM 中的指令的 trap 处理。

`// riscv/decode.h: line 247   #define require_novirt() if (unlikely(STATE.v)) throw trap_virtual_instruction(insn.bits())   `

Trap 相关的定义如下所示，上述 `require_novirt()` 函数实际上以异常的形式抛出一个 `trap_virtual_instruction` 类的实例。

`// riscv/trap.h: line 118   DECLARE_INST_TRAP(CAUSE_VIRTUAL_INSTRUCTION, virtual_instruction)      // riscv/trap.h: line 77   #define DECLARE_INST_TRAP(n, x) class trap_##x : public insn_trap_t { \    public: \     trap_##x(reg_t tval) : insn_trap_t(n, /*gva*/false, tval) {} \     const char* name() { return "trap_"#x; } \   };      // riscv/trap.h: line 41   class insn_trap_t : public trap_t   {    public:     insn_trap_t(reg_t which, bool gva, reg_t tval)       : trap_t(which), gva(gva), tval(tval) {}     bool has_gva() override { return gva; }     bool has_tval() override { return true; }     reg_t get_tval() override { return tval; }    private:     bool gva;     reg_t tval;   };      // riscv/trap.h: line 16   class trap_t   {    public:     trap_t(reg_t which) : which(which) {}     virtual bool has_gva() { return false; }     virtual bool has_tval() { return false; }     virtual reg_t get_tval() { return 0; }     virtual bool has_tval2() { return false; }     virtual reg_t get_tval2() { return 0; }     virtual bool has_tinst() { return false; }     virtual reg_t get_tinst() { return 0; }     reg_t cause() { return which; }        virtual const char* name()  {       const char* fmt = uint8_t(which) == which ? "trap #%u" : "interrupt #%u";       sprintf(_name, fmt, uint8_t(which));       return _name;     }       private:     char _name[16];     reg_t which;   };   `

##### flush_tlb()

`flush_tlb()` 函数的实现如下，该函数将所有的 TLB（Translation Look-aside Buffer）项置为 -1，同时将指令缓存（icache）全部置为 -1。

`// riscv/mmu.cc: line 32   void mmu_t::flush_tlb()   {     memset(tlb_insn_tag, -1, sizeof(tlb_insn_tag));     memset(tlb_load_tag, -1, sizeof(tlb_load_tag));     memset(tlb_store_tag, -1, sizeof(tlb_store_tag));        flush_icache();   }      // riscv/mmu.h: line 431   reg_t tlb_insn_tag[TLB_ENTRIES];   reg_t tlb_load_tag[TLB_ENTRIES];   reg_t tlb_store_tag[TLB_ENTRIES];      // riscv/mmu.h: line 426   static const reg_t TLB_ENTRIES = 256;      // riscv/mmu.h: line 423   icache_entry_t icache[ICACHE_ENTRIES];      // riscv/mmu.cc: line 26   void mmu_t::flush_icache()   {     for (size_t i = 0; i < ICACHE_ENTRIES; i++)       icache[i].tag = -1;   }   `

### H 扩展内存管理指令

#### HFENCE.VVMA & HFENCE.GVMA

![Image](https://mmbiz.qpic.cn/mmbiz_png/XXJQJDtx0eZCqqGC5Fr3ecA7jGCZiag9CE5q4Lkux5Foh4gZQyAFiaxCjOBBXVoWon3wGFibdVnoADwYMcbksxmDg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)
!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

hfence

`HFENCE.xVMA` 指令格式与 `SFENCE.VMA` 完全相同，不同之处在于指令有效时所在的特权级不同，所处理的特权级也各有不同。下方表格比较了这三条指令的特权级、CSR、功能、实现以及对应 Trap 的不同。

| 指令                                                   | `SFENCE.VMA`                                                                                                                                                           | `HFENCE.VVMA`                                                                                                                                                                                                      | `HFENCE.GVMA`                                                                                                                                                                                                                                                                               |
| ---------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 指令生效所需的特权级                                           | M                                                                                                                                                                      | M/HS                                                                                                                                                                                                               | HS (`mstatus.TVM=0`) / M                                                                                                                                                                                                                                                                    |
| 作用于某特权级                                              | S                                                                                                                                                                      | VS                                                                                                                                                                                                                 | HS                                                                                                                                                                                                                                                                                          |
| 对应的 **atp** (Address Translation and Protection) CSR | the current `satp` (either the HS-level `satp` when V=0 or `vsatp` when V=1)                                                                                           | `vsatp`                                                                                                                                                                                                            | `hgatp`                                                                                                                                                                                                                                                                                     |
| 指令作用：控制读写顺序，无效化 TLB 特定项                              | to guarantee writing finishes before reading; to invalidate TLB                                                                                                        | much the same as temporarily entering VS-mode and executing `SFENCE.VMA`; applies only to a single virtual machine, identified by the setting of `hgatp.VMID`                                                      | to guarantee stores of current hart finishes before reading of the guest hart                                                                                                                                                                                                               |
| 简化版实现：Over-Fence                                     | ignore `rs1` and `rs2`, always perform a global fence (with no exception for invalid `rs1`)                                                                            | ignore `rs1` and `rs2` as well as `hgatp.VMID`, always perform a **global fence for the VS-level memory management of all virtual machines**, or even a **global fence for all memory-management data structures** | ignore `rs1` and `VMID` in `rs2`, always perform a **global fence for the guest-physical memory management of all virtual machines**, or even a **global fence for all memory-management data structures**                                                                                  |
| Trap                                                 | For implementations that make `satp.MODE` read-only zero (always Bare), attempts to execute an `SFENCE.VMA` instruction might raise an _illegal instruction exception_ | Neither `mstatus.TVM` nor `hstatus.VTVM` causes `HFENCE.VVMA` to trap                                                                                                                                              | Attempts to execute `HFENCE.VVMA` or `HFENCE.GVMA` when V=1 cause a _virtual instruction trap_, while attempts to do the same in U-mode cause an _illegal instruction trap_; Attempting to execute `HFENCE.GVMA` in HS-mode when `mstatus.TVM=1` also causes an _illegal instruction trap_. |

#### Spike 模拟器实现

##### hfence.gvma

在虚拟机里的虚拟地址访问需要经历 VS 和 G 两个阶段的转换，这两个阶段分别由 CSR `vsatp` 和 `hgatp` 控制，完成从原始虚拟地址（original virtual address, VA）到客户机物理地址（guest physical address, GPA）再到监视器物理地址（hypervisor physical address, HPA）的转换，分别记作 **VS-Stage** 和 **G-Stage**。

`// riscv/insns/hfence_gvma.h: line 1   require_extension('H');     // 需 H 扩展支持   require_novirt();           // 需引入 trap 机制处理该指令   /* 根据 mstatus.TVM 判断在什么特权级下处理上述指令 trap：    * mstatus.TVM=1，在 M-Mode 处理该 trap    * mstatus.TVM=0，在 S-Mode 处理该 trap    */   require_privilege(get_field(STATE.mstatus->read(), MSTATUS_TVM) ? PRV_M : PRV_S);   MMU.flush_tlb();            // FENCE 指令功能的最简化实现：over-fence，flush 全部的 TLB 项      `

##### hfence.vvma

`hfence.vvma` 指令用于在 S-Mode 执行特定 VM 中的内存管理。其功能可以视作暂时进入 VS-Mode 执行 `SFENCE.VMA`，不同之处在于 `hfence.vvma` 指令从 `vsatp` 处获得要进行操作的地址空间，而 `SFENCE.VMA` 指令则从当前 `satp` (the HS-level `satp` when V=0 or `vsatp` when V=1) 处获得要进行操作的地址空间或虚拟机。

`// riscv/insns/hfence_vvma.h: line 1   require_extension('H');   require_novirt();   require_privilege(PRV_S);   MMU.flush_tlb();      `

### 内存管理指令模拟器实现小结

在 Spike 对本文所提到的三个内存管理指令的实现可以总结如下：

||`sfence.vma`|`hfence.gvma`|`hfence.vvma`|
|---|---|---|---|
|whether require_novirt() to trap|V=1, VU-Mode or `hstatus.VTVM` =1|necessary|necessary|
|require_privilege()|`mstatus.TVM` ==1 ? M: S|`mstatus.TVM` ==1 ? M: S|S|
|MMU.flush_tlb()|necessary|necessary|necessary|

## 两类内存管理相关的 CSR

### XSTATUS CSR

`xstatus` (x-level status) 用于表示处于 X 特权级的状态控制状态寄存器。

|M-Mode|HS-Mode|VS-Mode|
|---|---|---|
|`mstatus` to keep track of and controls the **hart’s current operating state**|`sstatus` to keeps track of the **processor’s current operating state**|`vsstatus` is **VS-mode’s version of `sstatus`**, when V=1, `vsstatus` substitutes for `sstatus`|
||`hstatus` provides facilities analogous to the `mstatus` register for tracking and controlling the **exception behavior of a VS-mode guest**||
![Image](https://mmbiz.qpic.cn/mmbiz_png/XXJQJDtx0eZCqqGC5Fr3ecA7jGCZiag9CVdrd0k2402jhibAKXW8oYA7N2hahvGk6lErU1picEcHiaHibs4BHFFvQBA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)
!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

此图由 Mermaid 生成

#### mstatus

参考特权指令级手册 3.1.6.5，如下表所示，`mstatus` CSR 与虚拟化相关的部分包含三位，分别对应 S-Mode 下的内存管理操作如 `SFENCE.VMA`、中断等候和 Trap 返回。

|Bit|Name|与此位相关的指令|Detail|
|---|---|---|---|
|TVM|Trap Virtual Memory|supervisor virtual-memory management operations（内核态虚拟内存管理操作）|TVM=1, write to `satp` or execution of `SFENCE.VMA` or `SINVAL.VMA` in **S-mode** will cause _illegal instruction exception_; TVM=0, permitted in S-mode; 0 if S-mode not supported|
|TW|Timeout Wait|`WFI` (Wait For Interrupt)|TW=0, execute `WFI` in lower-privileged mode directly; TW=1, execute as before within time limit, if out of time limit, cause _illegal instruction exception_; 0 if only M-mode support|
|TSR|Trap `SRET`|`sret` (Supervisor RETurn)|TSR=1, execute `sret` in S-mode causes _illegal instruction exception_; TSR=0, permitted; 0 if no S-mode support|
![Image](https://mmbiz.qpic.cn/mmbiz_png/XXJQJDtx0eZCqqGC5Fr3ecA7jGCZiag9CJlDrWd33rSB9iaafU6tZGqwROr7KEAqrKjllNIW4DtxoreRH0nWxKQg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)
!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

mstatus

#### hstatus

`hstatus` 是 H 扩展引入的一个处理 HS 和 VS 特权级 Trap 的 CSR，与 `mstatus` 一同完成 H 扩展支持下的 Trap 处理。
![Image](https://mmbiz.qpic.cn/mmbiz_png/XXJQJDtx0eZCqqGC5Fr3ecA7jGCZiag9CrMz8ME6duuMP9mkRib4LxY6u3obUcmicAZkHTGNIOmkH1trjd6Lic03Bg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)
!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

hstatus

#### mstatus, hstatus 对比（XLEN=64）

`hstatus` 是在 `mstatus` 基础上补足对于虚拟机的管理的 CSR，二者的功能区有相似之处，如 `hstatus.VTVM` 就是用于管理 VM 中的虚拟内存 trap 的标志位，类似于 `mstatus.TVM` 是用于管理 S-Mode 下 Supervisor 的虚拟内存的标志位，在一个支持 H 扩展的机器中，`mstatus.TVM` 将被用于 Hypervisor 的虚拟地址管理。二者也各自具有一些独有的标志位，如 `mstatus` 的扩展状态位（Extension Context Status）和 `hstatus` 中用于管理 VM 中数据读写指令可用性的 `HU` 位。

|Function|mstatus|hstatus|Difference or Effects|
|---|---|---|---|
|Global Interrupt-Enable Stack|MIE, SIE|**VGEIN** (Virtual Guest External Interrupt Number)|selects a guest external interrupt source for VS-level external interrupts|
||MPIE, SPIE|||
|Privilege Control|MPP, SPP|**SPVP** (Supervisor Previous Virtual Privilege)|written by the implementation whenever **a trap is taken into HS-mode when V=1**|
|Base ISA Control|SXL|VSXL||
||UXL|||
|Memory Privilege|MPRV (Modify PRiVilege)|||
||MXR (Make eXecutable Readable)|||
||SUM (permit Supervisor User Memory access)|||
|Endianness Control|MBE, SBE, and UBE|VSBE||
|Virtualization Support|TVM (Trap Virtual Memory)|VTVM|V instructions affect execution only in **VS-mode**, and cause _virtual instruction exceptions_ instead of _illegal instruction exceptions_|
||TW (Timeout Wait)|VTW||
||TSR (Trap SRet)|VTSR||
|Extension Context Status|FS\[1:0\]|||
||VS\[1:0\]|||
||XS\[1:0\]|||
|Virtualization Trap Control|**MPV** (Machine Previous Virtualization mode)|**SPV** (Supervisor Previous Virtualization mode)|written by the implementation whenever **a trap is taken into M/HS-mode**|
||**GVA** (Guest Virtual Address)|**GVA** (Guest Virtual Address)|written by the implementation whenever **a trap is taken into M/HS-mode**|
|VM Load/Store Control||**HU** (Hypervisor in U-mode)|controls whether the virtual-machine load/store instructions, `HLV`, `HLVX`, and `HSV`, can **be used also in U-mode**|

#### sstatus 和 vsstatus

`sstatus` 可以视为 `mstatus` 的一个子集，保存的是作用于 S 特权级的信息。在简化版的 `sstatus` 的实现中，对于 `sstatus` 对应区域的读写等同于对 `mstatus` 对应区域的读写（这一点在 QEMU 和 Spike 对 `sret` 指令的实现中有所体现，详情参见 此文 中的 **返回指令与虚拟化** 一节）。
![Image](https://mmbiz.qpic.cn/mmbiz_png/XXJQJDtx0eZCqqGC5Fr3ecA7jGCZiag9C37t7XzGyllJSMicRMDam4kCdrAhRvQ2YYZQc7ZpEfCUoZTaqOuUbAbA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)
!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

sstatus
![Image](https://mmbiz.qpic.cn/mmbiz_png/XXJQJDtx0eZCqqGC5Fr3ecA7jGCZiag9CKvbqNGiaTcTJibK5EMRlXt38ZvsFyG2n8cya5yU8xxTBTnHg2ulgIQAw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)
!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

vsstatus

如上所示，CSR `sstatus` 和 `vsstatus` 的分区和功能一致，只是服务于不同特权模式下的运行。现将如上 CSR 中与特权级转换和地址转换相关的功能位归纳如下。

|Function|Field|
|---|---|
|Base ISA Control|UXL|
|Privilege Control|SPP|
|Memory Privilege|MXR|
|Endianness Control|UBE|
|Interrupt-Enable Stack|SIE|
||SPIE|
|Extension Context Status|VS\[1:0\]|
||FS\[1:0\]|
||XS\[1:0\]|

#### 小结

从 CSR 的分区来看，`mstatus`, `hstatus`, `sstatus/vsstatus` 的关系如下：
![Image](https://mmbiz.qpic.cn/mmbiz_png/XXJQJDtx0eZCqqGC5Fr3ecA7jGCZiag9CJ2IDaA3abtibricVI45fIu1krJwqhOicw61TJrpacexaatsL1eNT24Jibg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)
!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

此图由 Mermaid 生成

`sstatus` 的内容是 `mstatus` 的一部分，所以在实现时对 `sstatus` 的修改等同于对 `mstatus` 的修改；`hstatus` 与 `mstatus` 只有一个相同的 `GVA` (Guest Virtual Address) 区。

从功能上来看，`hstatus` 提供了对 VM 进行管理的相关支持。它与 `mstatus` 相互配合共同完成对 Hypervisor 和 VM 的管理。`vsstatus` 则是 `sstatus` 的拷贝，用于 VM 内部。虚拟机切换相当于线程的上下文切换，这个过程中 `sstatus` 充当低特权级的 `mstatus` 子集拷贝，用于另一个虚拟机 `vsstatus` 的初始化。

### XATP CSR

`xatp` (x-level Address Translation and Protection) 表示处于 X 特权级的地址转换与保护控制状态寄存器。

#### satp

_S-Mode_ Address Translation and Protection (_satp_) Registers: 用于控制 S-Mode 下的地址转换和保护。

| Field Name | Width (32) | Range (32) | Width (64) | Range (64) | Function                                                                                                      |
| ---------- | ---------- | ---------- | ---------- | ---------- | ------------------------------------------------------------------------------------------------------------- |
| MODE       | 1          | 31         | 4          | 63:60      | selects the address-translation scheme                                                                        |
| ASID       | 9          | 30:22      | 16         | 59:44      | **address space identifier**, which facilitates address-translation fences on a per-address-space basis       |
| PPN        | 22         | 21:0       | 44         | 43:0       | hold the **page table number** of the root page table, i.e., its supervisor physical address divided by 4 KiB |
|            |            |            |            |            |                                                                                                               |

!\[\[Pasted image 20240913184925.png\]\]!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

- MODE

  MODE 位用来标记当前系统所支持的内存模式，32 位和 64 位有所不同。Bare 为裸机模式，不存在 VA 和 PA 的转换以及内存保护机制。用 1 和 8-10 表示 64 位系统下基于页表的采用不同虚拟地址位宽的内存系统。!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)\
  ![Image](https://mmbiz.qpic.cn/mmbiz_png/XXJQJDtx0eZCqqGC5Fr3ecA7jGCZiag9CU6T40icCAMYj5AZZ9rnk90eSUcLVY108dL5xP4nibnyeibaf3DJ0ZpSbg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

- _Active_ Status

  `satp` CSR 的值仅在特定特权级（**S-Mode, U-Mode**）有效，处于这两个特权级时，`satp` 的值会被用于虚拟地址转换。

#### hgatp

![Image](https://mmbiz.qpic.cn/mmbiz_png/XXJQJDtx0eZCqqGC5Fr3ecA7jGCZiag9CxGFw8RfQTekX0FuWcMLoTxO7jRgt51cfO6ykXp7b0hYtyh6UzcHpiag/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)
!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

hgatp

- PPN：保存 guest-physical 根页表编号

- VMID：相较于 `satp` 的 `ASID` 留出左边两位，剩余的位用于保存 Virtual Machine IDentifier

- MODE：与 `satp` 相同，指示系统采用的内存模式

![Image](https://mmbiz.qpic.cn/mmbiz_png/XXJQJDtx0eZCqqGC5Fr3ecA7jGCZiag9CiczQ575mbITghbKlqIUkibs6WHZibTrwtrp3uJHULKicsBnPI7CUy2eyFw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)
!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

hgatp-MODE

仅当处于 U 特权模式且 `hstatus.HU=0` 时，`vsatp` 才被视作无效状态。

#### vsatp

![Image](https://mmbiz.qpic.cn/mmbiz_png/XXJQJDtx0eZCqqGC5Fr3ecA7jGCZiag9CK58A1micDDqiaQl4rmeTexO9han3Z0hYHbM8C7hjcCxRTelMaKRlzKlw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)
!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

vsatp

V=1 时，`vsatp` 将取代 `satp`，所有对 `satp` 的操作（读写）将在 `vsatp` 上完成。`vsatp` 将用于 VS-Stage 的地址转换。

仅当处于 U 特权模式且 `hstatus.HU=0` 时，`vsatp` 才被视作无效状态。

#### 小结

`xatp` CSR 包含三个部分，MODE 部分用于标识采用哪种地址模式，第二部分用于指示隶属于不同部分的地址（进程的地址空间（Address Space）或虚拟机的标志符（VMID）），第三部分用于保存多级页表的根物理页的编号，进行地址转换时结合虚拟地址的 Offset 获取实际要访问的物理地址。

|xatp|被用于\*特权级的地址转换|无效化的条件|
|---|---|---|
|`satp`|S-Mode|M-Mode|
|`hgatp`|H-Mode, G-Stage|U-Mode and `hstatus.HU=0`|
|`vsatp`|VS-Mode, VS-Stage|U-Mode and `hstatus.HU=0`|

## 总结

RISC-V 通过引入 S 扩展实现了能够运行操作系统的 Machine/Supervisor/User-Mode 三级特权模式，同时基于操作系统对内存的虚拟化，S 扩展中添加了与之对应的内存管理指令 `SFENCE.VMA` 以实现对于地址空间和虚拟地址的访问控制。在引入 H 扩展之后，原有的三级特权模式变成了 Machine/Hypervisor-Extended Supervisor/Virtual Supervisor/Virtual User-Mode 四层特权模式，同时为了兼容原有的三级模式，Hypervisor-Extended Supervisor 和 Virtual Supervisor 共同归属于 S-Mode，而 Virtual User-Mode 则属于 U-Mode。新的特权级的引入需要引入新的内存管理指令以实现更完整的内存管理，`HFENCE.VVMA` 和 `HFENCE.GVMA` 两条指令由此引入。

上述三条指令分别与 `xstatus`, `xatp` 等对应特权级的 CSR 配合，管理特定特权级的内存。

本文尝试对上述指令和 CSR 及其关系进行了梳理，分析了以上指令在 Spike 模拟器中的实现。后续将会进一步解读为了实现更细粒度内存管理而引入的的扩展指令集 `Svinval`，并结合 S-Mode 和 H-Mode 下的地址转换机制及其模拟器实现，分析上述 CSR 与指令之间的互动机制。

## 参考资料

- RISC-V 特权指令集手册

______________________________________________________________________

**首发地址**：https://tinylab.org/riscv-kvm-mem-virt-1\
**技术服务**：https://tinylab.org/ruma.tech

左下角 **阅读原文** 可访问外链。都看到这里了，就随手在看+分享一下吧 ;-)

RISC-V127

linux内核164

RISC-V · 目录

上一篇RISC-V KVM 虚拟化：用户态程序下一篇RISC-V Linux 内核技术直播预告 第 52 期

Read more

Reads 482

​

Comment

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/XXJQJDtx0ea3GDXLMgprFptA1sKUYEYt3NNtA8bBuREmchgSXogPkvzqHvAANlQ8a6Tx0yeuzh1K5ibuqtWyYTA/300?wx_fmt=png&wxfrom=18)

泰晓科技

211

Comment

Comment

**Comment**

暂无留言
