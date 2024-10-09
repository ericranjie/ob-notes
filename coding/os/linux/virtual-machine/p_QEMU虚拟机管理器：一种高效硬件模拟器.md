原创 往事敬秋风 深度Linux

_2024年03月28日 22:04_ _湖南_

QEMU（Quick EMUlator）是一种开源的虚拟机监视器和模拟器，它可以模拟多个硬件平台，包括x86、ARM、PowerPC等。QEMU广泛应用于虚拟化、嵌入式系统开发和仿真等领域。

![](http://mmbiz.qpic.cn/mmbiz_png/dkX7hzLPUR0Ao40RncDiakbKx1Dy4uJicoqwn5GZ5r7zSMmpwHdJt32o95wdQmPZrBW038j8oRSSQllpnOUDlmUg/300?wx_fmt=png&wxfrom=19)

**深度Linux**

拥有15年项目开发经验及丰富教学经验，曾就职国内知名企业项目经理，部门负责人等职务。研究领域：Windows&Linux平台C/C++后端开发、Linux系统内核等技术。

181篇原创内容

公众号

作为虚拟机监视器，QEMU允许在一个物理主机上同时运行多个虚拟机，并提供对这些虚拟机的管理和控制能力。它支持各种操作系统，包括Linux、Windows和其他许多操作系统。作为模拟器，QEMU可以将不同架构的二进制代码在一个主机上进行执行，从而实现跨平台的软件开发与测试。这使得开发人员可以在自己的工作环境中运行并调试不同体系结构下的程序。

QEMU还提供了丰富的功能和扩展性，如硬件加速、网络配置、磁盘镜像和快照等。它被广泛应用于云计算、容器技术、嵌入式系统仿真和移动设备开发等领域，并受到众多开发者和组织的支持与贡献。

## 一、QEMU简介

QEMU是一款开源的模拟器及虚拟机监管器(Virtual Machine Monitor, VMM)。QEMU主要提供两种功能给用户使用。一是作为用户态模拟器，利用动态代码翻译机制来执行不同于主机架构的代码。二是作为虚拟机监管器，模拟全系统，利用其他VMM(Xen, KVM, etc)来使用硬件提供的虚拟化支持，创建接近于主机性能的虚拟机。

用户可以通过不同Linux发行版所带有的软件包管理器来安装QEMU。如在Debian系列的发行版上可以使用下面的命令来安装：

```c
sudo apt-get install qemu
```

除此之外，也可以选择从源码安装。

获取QEMU源码

可以从QEMU下载官网上下载QEMU源码的tar包，以命令行下载2.0版本的QEMU为例：

```c
$wget http://wiki.qemu-project.org/download/qemu-2.0.0.tar.bz2$tar xjvf qemu-2.0.0.tar.bz2
```

编译及安装

获取源码后，可以根据需求来配置和编译QEMU

```c
$cd qemu-2.0.0 //如果使用的是git下载的源码，执行cd qemu$./configure --enable-kvm --enable-debug --enable-vnc --enable-werror  --target-list="x86_64-softmmu"或者用户模式（使能TCI）$./configure --target-list=arm-linux-user --enable-tcg-interpreter$make -j8$sudo make install
```

configure脚本用于生成Makefile，其选项可以用`./configure --help`查看。这里使用到的选项含义如下：

```c
--enable-kvm：编译KVM模块，使QEMU可以利用KVM来访问硬件提供的虚拟化服务。--enable-vnc：启用VNC。--enalbe-werror：编译时，将所有的警告当作错误处理。--target-list：选择目标机器的架构。默认是将所有的架构都编译，但为了更快的完成编译，指定需要的架构即可。
```

安装好之后，会生成如下应用程序：
!\[\[Pasted image 20240918110732.png\]\]
!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

- ivshmem-client/server：这是一个 guest 和 host 共享内存的应用程序，遵循 C/S 的架构。

- qemu-ga：这是一个不利用网络实现 guest 和 host 之间交互的应用程序（使用 virtio-serial），运行在 guest 中。

- qemu-io：这是一个执行 Qemu I/O 操作的命令行工具。

- qemu-system-x86_64：Qemu 的核心应用程序，虚拟机就由它创建的。

- qemu-img：创建虚拟机镜像文件的工具，下面有例子说明。

- qemu-nbd：磁盘挂载工具。

## 二、基本原理

QEMU作为系统模拟器时，会模拟出一台能够独立运行操作系统的虚拟机。如下图所示，每个虚拟机对应主机(Host)中的一个QEMU进程，而虚拟机的vCPU对应QEMU进程的一个线程。

系统虚拟化最主要是虚拟出CPU、内存及I/O设备。虚拟出的CPU称之为vCPU，QEMU为了提升效率，借用KVM、XEN等虚拟化技术，直接利用硬件对虚拟化的支持，在主机上安全地运行虚拟机代码(需要硬件支持)。虚拟机vCPU调用KVM的接口来执行任务的流程如下(代码源自QEMU开发者Stefan的技术博客)：

```c
open("/dev/kvm")ioctl(KVM_CREATE_VM)ioctl(KVM_CREATE_VCPU)for (;;) {     ioctl(KVM_RUN)     switch (exit_reason) {     case KVM_EXIT_IO:  /* ... */     case KVM_EXIT_HLT: /* ... */     }}
```

QEMU发起ioctrl来调用KVM接口，KVM则利用硬件扩展直接将虚拟机代码运行于主机之上，一旦vCPU需要操作设备寄存器，vCPU将会停止并退回到QEMU，QEMU去模拟出操作结果。

虚拟机内存会被映射到QEMU的进程地址空间，在启动时分配。在虚拟机看来，QEMU所分配的主机上的虚拟地址空间为虚拟机的物理地址空间。

QEMU在主机用户态模拟虚拟机的硬件设备，vCPU对硬件的操作结果会在用户态进行模拟，如虚拟机需要将数据写入硬盘，实际结果是将数据写入到了主机中的一个镜像文件中。

## 三、创建及使用虚拟机

使用qemu-img创建虚拟机镜像，虚拟机镜像用来模拟虚拟机的硬盘，在启动虚拟机之前需要创建镜像文件。

```c
qemu-img create -f qcow2 test-vm-1.qcow2 10G
```

-f 选项用于指定镜像的格式，qcow2 格式是 Qemu 最常用的镜像格式，采用来写时复制技术来优化性能。test-vm-1.qcow2 是镜像文件的名字，10G是镜像文件大小。镜像文件创建完成后，可使用 qemu-system-x86 来启动x86 架构的虚拟机：

使用 qemu-system-x86 来启动 x86 架构的虚拟机

```c
qemu-system-x86_64 test-vm-1.qcow2
```

因为 test-vm-1.qcow2 中并未给虚拟机安装操作系统，所以会提示 “No bootable device”，无可启动设备。

启动 VM 安装操作系统镜像

```
qemu-system-x86_64 -m 2048 -enable-kvm test-vm-1.qcow2 -cdrom ./Centos-Desktop-x86_64-20-1.iso
```

-m 指定虚拟机内存大小，默认单位是 MB， -enable-kvm 使用 KVM 进行加速，-cdrom 添加 fedora 的安装镜像。可在弹出的窗口中操作虚拟机，安装操作系统，安装完成后重起虚拟机便会从硬盘 ( test-vm-1.qcow2 ) 启动。之后再启动虚拟机只需要执行：

```
qemu-system-x86_64 -m 2048 -enable-kvm test-vm-1.qcow2
```

qemu-img 支持非常多种的文件格式，可以通过 qemu-img -h 查看，其中 raw 和 qcow2 是比较常用的两种，raw 是 qemu-img 命令默认的，qcow2 是 qemu 目前推荐的镜像格式，是功能最多的格式。

## 四、Qemu源码结构

Qemu 软件虚拟化实现的思路是采用二进制指令翻译技术，主要是提取 guest 代码，然后将其翻译成 TCG 中间代码，最后再将中间代码翻译成 host 指定架构的代码，如 x86 体系就翻译成其支持的代码形式，ARM 架构同理。
!\[\[Pasted image 20240918110751.png\]\]
!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

所以，从宏观上看，源码结构主要包含以下几个部分：

- /vl.c：最主要的模拟循环，虚拟机环境初始化，和 CPU 的执行。

- /target-arch/translate.c：将 guest 代码翻译成不同架构的 TCG 操作码。

- /tcg/tcg.c：主要的 TCG 代码。

- /tcg/arch/tcg-target.c：将 TCG 代码转化生成主机代码。

- /cpu-exec.c：主要寻找下一个二进制翻译代码块，如果没有找到就请求得到下一个代码块，并且操作生成的代码块。

其中，涉及的主要几个函数如下：
!\[\[Pasted image 20240918110756.png\]\]
!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

知道了这个总体的代码结构，再去具体了解每一个模块可能会相对容易一点。

(1)开始执行

主要比较重要的c文件有：/vl.c,/cpus.c, /exec-all.c, /exec.c, /cpu-exec.c。

QEMU的main函数定义在/vl.c中，它也是执行的起点，这个函数的功能主要是建立一个虚拟的硬件环境。它通过参数的解析，将初始化内存，需要的模拟的设备初始化，CPU参数，初始化KVM等等。接着程序就跳转到其他的执行分支文件如：/cpus.c, /exec-all.c, /exec.c, /cpu-exec.c。

(2)硬件模拟

所有的硬件设备都在/hw/ 目录下面，所有的设备都有独自的文件，包括总线，串口，网卡，鼠标等等。它们通过设备模块串在一起，在vl.c中的machine \_init中初始化。这里就不讲每种设备是怎么实现的了。

(3)目标机器

现在QEMU模拟的CPU架构有：Alpha, ARM, Cris, i386, M68K, PPC, Sparc, Mips, MicroBlaze, S390X and SH4。

我们在QEMU中使用./configure 可以配置运行的架构，这个脚本会自动读取本机真实机器的CPU架构，并且编译的时候就编译对应架构的代码。对于不同的QEMU做的事情都不同，所以不同架 构下的代码在不同的目录下面。/target-arch/目录就对应了相应架构的代码，如/target-i386/就对应了x86系列的代码部分。虽然 不同架构做法不同，但是都是为了实现将对应客户机CPU架构的TBs转化成TCG的中间代码。这个就是TCG的前半部分。

(4)主机

这个部分就是使用TCG代码生成主机的代码，这部分代码在/tcg/里面，在这个目录里面也对应了不同的架构，分别在不同的子目录里面，如i386就在/tcg/i386中。整个生成主机代码的过程也可以教TCG的后半部分。

(5)文件总结

```c
/vl.c:                      最主要的模拟循环，虚拟机机器环境初始化，和CPU的执行。/target-arch/translate.c    将客户机代码转化成不同架构的TCG操作码。/tcg/tcg.c                  主要的TCG代码。/tcg/arch/tcg-target.c      将TCG代码转化生成主机代码/cpu-exec.c  其中的cpu-exec()函数主要寻找下一个TB（翻译代码块），如果没找到就请求得到下一个TB，并且操作生成的代码块。
```

### 4.1TCG动态翻译

QEMU在 0.9.1版本之前使用DynGen翻译c代码.当我们需要的时候TCG会动态的转变代码，这个想法的目的是用更多的时间去执行我们生成的代码。当新的代 码从TB中生成以后， 将会被保存到一个cache中，因为很多相同的TB会被反复的进行操作，所以这样类似于内存的cache，能够提高使用效率。而 cache的刷新使用LRU算法。
!\[\[Pasted image 20240918111036.png\]\]
!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

编译器在执行器会从源代码中产生目标代码，像GCC这种编译器，它为了产生像函数调用目标代码会产生一些特殊的汇编目标代码，他们能够让编译器需要知道在调用函数。需要什么，以及函数调用以后需要返回什么，这些特殊的汇编代码产生过程就叫做函数的Prologue和Epilogue，这里就叫前端和后段吧。我在其他文章中也分析过汇编调用函数的过程，至于汇编里面函数调用过程中寄存器是如何变化的，在本文中就不再描述了。

函数的后端会恢复前端的状态，主要做下面2点：

- 1. 恢复堆栈的指针，包括栈顶和基地址。

- 2. 修改cs和ip，程序回到之前的前端记录点。

TCG就如编译器一样可以产生目标代码，代码会保存在缓冲区中，当进入前端和后端的时候就会将TCG生成的缓冲代码插入到目标代码中。

接下来我们就来看下如何翻译代码的：

客户机代码
!\[\[Pasted image 20240918111043.png\]\]
!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

TCG中间代码
!\[\[Pasted image 20240918111048.png\]\]
!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

主机代码
!\[\[Pasted image 20240918111053.png\]\]
!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

### 4.2TB链

在QEMU中，从代码cache到静态代码再回到代码cache，这个过程比较耗时，所以在QEMU中涉及了一个TB链将所有TB连在一起，可以让一个TB执行完以后直接跳到下一个TB，而不用每次都返回到静态代码部分。具体过程如下图：
!\[\[Pasted image 20240918111058.png\]\]
!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

### 4.3TCG代码分析

接下来来看看QEMU代码中中到底怎么来执行这个TCG的，看看它是如何生成主机代码的。

main_loop(...){/vl.c} :

函数main_loop 初始化qemu_main_loop_start()然后进入无限循环cpu_exec_all() ， 这个是QEMU的一个主要循环，在里面会不断的判断一些条件，如虚拟机的关机断电之类的。

qemu_main_loop_start(...){/cpus.c} :

函数设置系统变量 qemu_system_ready = 1并且重启所有的线程并且等待一个条件变量。

cpu_exec_all(...){/cpus.c} :

它是cpu循环，QEMU能够启动256个cpu核，但是这些核将会分时运行，然后执行qemu_cpu_exec() 。

struct CPUState{/target-xyz/cpu.h} :

它是CPU状态结构体，关于cpu的各种状态，不同架构下面还有不同。

cpu_exec(...){/cpu-exec.c}:

这个函数是主要的执行循环，这里第一次翻译之前说道德TB，TB被初始化为(TranslationBlock \*tb) ，然后不停的执行异常处理。其中嵌套了两个无限循环 find tb_find_fast() 和tcg_qemu_tb_exec().\
cantb_find_fast()为客户机初始化查询下一个TB，并且生成主机代码。\
tcg_qemu_tb_exec()执行生成的主机代码

struct TranslationBlock {/exec-all.h}:

结构体_TranslationBlock_包含下面的成员：PC, CS_BASE, Flags （表明TB）, tc_ptr (指向这个TB翻译代码的指针), tb_next_offset\[2\], tb_jmp_offset\[2\] (接下去的Tb), \*jmp_next\[2\], \*jmp_first (之前的TB).

tb_find_fast(...){/cpu-exec.c} :

函数通过调用获得程序指针计数器，然后传到一个哈希函数从 tb_jmp_cache\[\] （一个哈希表）得到TB的所以，所以使用tb_jmp_cache可以找到下一个TB。如果没有找到下一个TB，则使用tb_find_slow。

tb_find_slow(...){/cpu-exec.c}:

这个是在快速查找失败以后试图去访问物理内存，寻找TB。

tb_gen_code(...){/exec.c}:

开始分配一个新的TB，TB的PC是刚刚从CPUstate里面通过using get_page_addr_code()找到的\
phys_pc = get_page_addr_code(env, pc);\
tb = tb_alloc(pc);\
ph当调用cpu_gen_code() 以后，接着会调用tb_link_page()，它将增加一个新的TB，并且指向它的物理页表。

cpu_gen_code(...){translate-all.c}:

函数初始化真正的代码生成，在这个函数里面有下面的函数调用：

```c
gen_intermediate_code(){/target-arch/translate.c}->gen_intermediate_code_internal(){/target-arch/translate.c }->disas_insn(){/target-arch/translate.c}
```

disas_insn(){/target-arch/translate.c}

函数disas_insn() 真正的实现将客户机代码翻译成TCG代码，它通过一长串的switch case，将不同的指令做不同的翻译，最后调用tcg_gen_code。

tcg_gen_code(...){/tcg/tcg.c}:

这个函数将TCG的代码转化成主机代码，这个就不细细说明了，和前面类似。

#define tcg_qemu_tb_exec(...){/tcg/tcg.g}:\
通过上面的步骤，当TB生成以后就通过这个函数进行执行

```c
next_tb = tcg_qemu_tb_exec(tc_ptr) :extern uint8_t code_gen_prologue[];#define tcg_qemu_tb_exec(tb_ptr) ((long REGPARM(*)(void *)) code_gen_prologue)(tb_ptr)
```

通过上面的步骤我们就解析了QEMU是如何将客户机代码翻译成主机代码的，了解了TCG的工作原理。接下来看看QEMU与KVM是怎么联系的。

### 4.4IOCTL使用流程

在QEMU-KVM中，用户空间的QEMU是通过IOCTL与内核空间的KVM模块进行通讯的。

(1)创建KVM

在/vl.c中通过kvm_init()将会创建各种KVM的结构体变量，并且通过IOCTL与已经初始化好的KVM模块进行通讯，创建虚拟机。然后创建VCPU，等等。

(2)KVM_RUN

这个IOCTL是使用最频繁的，整个KVM运行就不停在执行这个IOCTL，当KVM需要QEMU处理一些指令和IO等等的时候就会退出通过这个IOCTL退回到QEMU进行处理，不然就会一直在KVM中执行。

它的初始化过程：

vl.c中调用machine->init初始化硬件设备接着调用pc_init_pci，然后再调用pc_init1。

接着通过下面的调用初始化KVM的主循环，以及CPU循环。在CPU循环的过程中不断的执行KVM_RUN与KVM进行交互。

```
pc_init1->pc_cpus_init->pc_new_cpu->cpu_x86_init->qemu_init_vcpu->kvm_init_vcpu->ap_main_loop->kvm_main_loop_cpu->kvm_cpu_exec->kvm_run
```

(3)KVM_IRQ_LINE

这个IOCTL和KVM_RUN是不同步的，它也是个频率非常高的调用，它就是一般中断设备的中断注入入口。当设备有中断就通过这个IOCTL最终 调用KVM里面的kvm_set_irq将中断注入到虚拟的中断控制器。在kvm中会进一步判断属于什么中断类型，然后在合适的时机写入vmcs。当然在 KVM_RUN中会不断的同步虚拟中断控制器，来获取需要注入的中断，这些中断包括QEMU和KVM本身的，并在重新进入客户机之前注入中断。

## 五、QEMU中内存管理

相关配置参数

QEMU的命令行中有参数：

```c
-m [size=]megs[,slots=n,maxmem=size] 
```

用于指定客户机初始运行时的内存大小以及客户机最大内存大小，以及内存芯片槽的数量（DIMM）。

之所以QEMU可以指定最大内存、槽等参数，是因为QEMU可以模拟DIMM的热插拔，客户机操作系统可以和在真实的系统上一样，检测新内存被插入或者拔出。也就是说，内存热插拔的粒度是DIMM槽（或者说DIMM集合），而不是最小的byte。

与内存相关的数据结构：
!\[\[Pasted image 20240918111112.png\]\]
!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

PCDIMMDevice和HostMemoryBackend对象都是在QEMU中用户可见的客户机内存。它们能通过QEMU命令行或者QMP监控器接口来管理。

PCDIMMDevice数据结构是使用QEMU中的面向对象编程模型QOM定义的，对应的对象和类的数据结构如下。通过在QEMU进程中创建一个新的PCDIMMDevice对象，就可以实现内存的热插拔。

值得注意的是，客户机启动时的初始化内存，可能不会被模拟成PCDIMMDevice设备，也就是说，这部分初始化内存不能进行热插拔。PCDIMMDevice的定义在include/hw/mem/pc-dimm.h中。

```c
typedef struct PCDIMMDevice {    /* private */    DeviceState parent_obj;    /* public */    uint64_t addr;    uint32_t node; //numa node    int32_t slot;  //slot编号    HostMemoryBackend *hostmem;} PCDIMMDevice;typedef struct PCDIMMDeviceClass {    /* private */    DeviceClass parent_class;    /* public */    MemoryRegion *(*get_memory_region)(PCDIMMDevice *dimm);} PCDIMMDeviceClass;
```

每个PCDIMMDevice对象都与 HostMemoryBackend对象相关联。HostMemoryBackend也是使用QEMU中的面向对象编程模型QOM定义的。HostMemoryBackend定义在include/sysemu/hostmem.h中。HostMemoryBackend对象包含了客户机内存对应的真正的主机内存，这些内存可以是匿名映射的内存，也可以是文件映射内存。文件映射的客户机内存允许Linux在物理主机上透明大页机制的使用（hugetlbfs），并且能够共享内存，从而使其他进程可以访问客户机内存。

```c
 struct HostMemoryBackend {    /* private */    Object parent;    /* protected */    uint64_t size;    bool merge, dump;    bool prealloc, force_prealloc;    DECLARE_BITMAP(host_nodes, MAX_NODES + 1);    HostMemPolicy policy;    MemoryRegion mr;};struct HostMemoryBackendClass {    ObjectClass parent_class;    void (*alloc)(HostMemoryBackend *backend, Error **errp);};
```

HostMemoryBackend对象中的内存被实际映射到通过qemu_ram_alloc()函数（代码定义在exec.c中）RAMBlock数据结构中。每个RAMBlock都有一个指针指向被映射内存的起始位置，同时包含一个ram_addr_t的位移变量。ram_addr_t位于全局的命名空间中，因此RAMBlock能够通过offset来查找。

RAMBlock定义在include/exec/ram_addr.h中。RAMBlock受RCU机制保护，所谓RCU，即Read-COPY-Update。

```c
typedef uint64_t ram_addr_t;struct RAMBlock {    struct rcu_head rcu; //该数据结构受rcu机制保护    struct MemoryRegion *mr;     uint8_t *host;          //RAMBlock的内存起始位置    ram_addr_t offset;  //在所有的RAMBlock中offset    ram_addr_t used_length;  //已使用长度    ram_addr_t max_length;  //最大分配内存    void (*resized)(const char*, uint64_t length, void *host);    uint32_t flags;    /* Protected by iothread lock.  */    char idstr[256];      //RAMBlock的ID    /* RCU-enabled, writes protected by the ramlist lock */    QLIST_ENTRY(RAMBlock) next;    int fd;  //映射文件的描述符};
```

所有的RAMBlock保存在全局的RAMBlock的链表中，名为RAMList，它有专门的数据结构定义。RAMList数据结构定义在include/exec/ram_addr.h中，而全局的ram_list变量则定义在exec.c中。因此这个链表保存了客户机的内存对应的所有的物理机的实际内存信息。

跟踪脏页

当客户机CPU或者DMA将数据保存到客户机内存时，需要通知下列一些用户：

- 1. 热迁移特性依赖于跟踪脏页，因此他们能够在被改变之后重新传输。
- 2. 图形卡模拟依赖于跟踪脏的视频内存，用于重画某些界面。

所有的CPU架构都有内存地址空间、有些CPU架构又有一个IO地址空间。它们在QEMU中被表示为AddressSpace数据结构，它定义在include/exec/memory.h中。而每个地址空间都包含一个MemoryRegion的树状结构，所谓树状结构，指的是每个MemoryRegion的内部可以含有MemoryRegion，这样的包含所形成的树状结构。

MemoryRegion是联系客户机内存和包含这一部分内存的RAMBlock。每个MemoryRegion都包含一个在RAMBlock中ram_addr_t类型的offset，每个RAMBlock也有一个MemoryRegion的指针。

MemoryRegion不仅可以表示RAM，也可以表示I/O映射内存，在访问时可以调用read/write回调函数。这也是硬件从客户机CPU注册的访问被分派到相应的模拟设备的方法。

```c
struct AddressSpace {    /* All fields are private. */    struct rcu_head rcu;    char *name;    MemoryRegion *root;    /* Accessed via RCU.  */    struct FlatView *current_map; //AddressSpace的一张平面视图，它是AddressSpace所有正在使用的MemoryRegion的集合，这是从CPU的视角来看到的。    int ioeventfd_nb;    struct MemoryRegionIoeventfd *ioeventfds;    struct AddressSpaceDispatch *dispatch;    struct AddressSpaceDispatch *next_dispatch;    MemoryListener dispatch_listener;    QTAILQ_ENTRY(AddressSpace) address_spaces_link;};struct MemoryRegion {    Object parent_obj;    /* All fields are private - violators will be prosecuted */    const MemoryRegionOps *ops;  //与MemoryRegion相关的操作    const MemoryRegionIOMMUOps *iommu_ops;    void *opaque;    MemoryRegion *container;      Int128 size;    hwaddr addr;         //在AddressSpace中的地址    void (*destructor)(MemoryRegion *mr);    ram_addr_t ram_addr;  //MemoryRegion的起始地址    uint64_t align;   //big-endian or little-endian    bool subpage;    bool terminates;    bool romd_mode;    bool ram;    bool skip_dump;    bool readonly; /* For RAM regions */    bool enabled;     //如果为true，表示已经通知kvm使用这段内存    bool rom_device;  //是否是只读内存    bool warning_printed; /* For reservations */    bool flush_coalesced_mmio;    bool global_locking;    uint8_t vga_logging_count;    MemoryRegion *alias;    hwaddr alias_offset;    int32_t priority;    bool may_overlap;    QTAILQ_HEAD(subregions, MemoryRegion) subregions;    QTAILQ_ENTRY(MemoryRegion) subregions_link;    QTAILQ_HEAD(coalesced_ranges, CoalescedMemoryRange) coalesced;    const char *name;  //MemoryRegion的名字,调试时使用    uint8_t dirty_log_mask;  //表示哪种dirty map被使用，共有三种    //IOevent文件描述符的管理    unsigned ioeventfd_nb;    MemoryRegionIoeventfd *ioeventfds;    NotifierList iommu_notify;};
```

2023年往期回顾

[C/C++发展方向（强烈推荐！！）](http://mp.weixin.qq.com/s?__biz=Mzg4NDQ0OTI4Ng==&mid=2247487749&idx=1&sn=e57e6f3df526b7ad78313d9428e55b6b&chksm=cfb9586cf8ced17a8c7830e380a45ce080c2b8258e145f5898503a779840a5fcfec3e8f8fa9a&scene=21#wechat_redirect)

[Linux内核源码分析（强烈推荐收藏！](http://mp.weixin.qq.com/s?__biz=Mzg4NDQ0OTI4Ng==&mid=2247487832&idx=1&sn=bf0468e26f353306c743c4d7523ebb07&chksm=cfb95831f8ced127ca94eb61e6508732576bb150b2fb2047664f8256b3da284d0e53e2f792dc&scene=21#wechat_redirect)[）](http://mp.weixin.qq.com/s?__biz=Mzg4NDQ0OTI4Ng==&mid=2247487832&idx=1&sn=bf0468e26f353306c743c4d7523ebb07&chksm=cfb95831f8ced127ca94eb61e6508732576bb150b2fb2047664f8256b3da284d0e53e2f792dc&scene=21#wechat_redirect)

[从菜鸟到大师，用Qt编写出惊艳世界应用](http://mp.weixin.qq.com/s?__biz=Mzg4NDQ0OTI4Ng==&mid=2247488117&idx=1&sn=a83d661165a3840fbb23d0e62b5f303a&chksm=cfb95b1cf8ced20ae63206fe25891d9a37ffe76fd695ef55b5506c83aad387d55c4032cb7e4f&scene=21#wechat_redirect)

[存储全栈开发：构建高效的数据存储系统](http://mp.weixin.qq.com/s?__biz=Mzg4NDQ0OTI4Ng==&mid=2247487696&idx=1&sn=b5ebe830ddb6798ac5bf6db4a8d5d075&chksm=cfb959b9f8ced0af76710c070a6db2677fb359af735e79c6378e82ead570aa1ce5350a146793&scene=21#wechat_redirect)

[突破性能瓶颈：释放DPDK带来无尽潜力](http://mp.weixin.qq.com/s?__biz=Mzg4NDQ0OTI4Ng==&mid=2247488150&idx=1&sn=2cc5ace4391d4eed060fa463fc73dd92&chksm=cfb95bfff8ced2e9589eecd66e7c6e906f28626bef6c1bce3fa19ae2d1404baa25ba3ebfedaa&scene=21#wechat_redirect)

[嵌入式音视频技术：从原理到项目应用](http://mp.weixin.qq.com/s?__biz=Mzg4NDQ0OTI4Ng==&mid=2247488135&idx=1&sn=dcc56a3af5401c338944b9b94f24c699&chksm=cfb95beef8ced2f8231c647033188be3ded88cbacf7d75feab3613f4d7f8fd8e4c8106ac88cf&scene=21#wechat_redirect)

[](http://mp.weixin.qq.com/s?__biz=MzIwNzA1OTA2OQ==&mid=2657215412&idx=1&sn=d3c64ac21056b84814db9903b656853a&chksm=8c8d62a6bbfaebb09df1f727e201c401a1663596a719fa9cb4c7859158e62ebedb5bd5f13b8f&scene=21#wechat_redirect)[C++游戏开发，基于魔兽开源后端框架](http://mp.weixin.qq.com/s?__biz=Mzg4NDQ0OTI4Ng==&mid=2247486920&idx=1&sn=82ca8a72473850e431c01e7c39d8c8c0&chksm=cfb944a1f8cecdb7051863f58d334ff3dea8afae1ecd8bc693287b2f483d2ce8f94eed565253&scene=21#wechat_redirect)

C/C++开发92

linux内核108

C/C++开发 · 目录

上一篇深入理解VFIO：一种高效的虚拟化设备访问技术下一篇最新趋势与前沿技术：探索Qt5 C++开发领域的未来方向

阅读 2726

​
