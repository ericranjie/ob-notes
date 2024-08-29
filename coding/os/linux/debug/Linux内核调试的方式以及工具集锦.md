
CHENG Jian Linux内核之旅

 _2023年01月05日 09:43_ _河南_

原文作者：CHENG Jian

原文链接：https://kernel.blog.csdn.net/article/details/68948080

  

![Image](https://mmbiz.qpic.cn/mmbiz_gif/SeWfibBcBT0GLHfB4W3GOuXibbfGOmolYkC40Mhadv2YN86hWbfwN3MMn0M7ObOrCwwNMdHa0gOPMT1hOqTkYf9g/640?wx_fmt=gif&tp=wxpic&wxfrom=5&wx_lazy=1)

  

![Image](https://mmbiz.qpic.cn/mmbiz_png/SeWfibBcBT0GLHfB4W3GOuXibbfGOmolYkuu2hNBKQelcVAD3wkvOMg9zGibTlkxy0vSNSiaXnVajtyhnzCBffu6Gw/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

本作品采用知识共享署名-非商业性使用-相同方式共享 4.0 国际许可协议进行许可, 转载请注明出处, 谢谢合作

因本人技术水平和知识面有限, 内容如有纰漏或者需要修正的地方, 欢迎大家指正, 也欢迎大家提供一些其他好的调试工具以供收录, 鄙人在此谢谢啦

  

# 1 内核调试以及工具总结

---

内核总是那么捉摸不透, 内核也会犯错, 但是调试却不能像用户空间程序那样, 为此内核开发者为我们提供了一系列的工具和系统来支持内核的调试.

内核的调试, 其本质是内核空间与用户空间的数据交换, 内核开发者们提供了多样的形式来完成这一功能.

![Image](https://mmbiz.qpic.cn/mmbiz_jpg/SeWfibBcBT0GLHfB4W3GOuXibbfGOmolYk2JxRlkuKljhCIakocsAAxwR5Ek4nOdCibPdzhRhBKdicyR1SsSPb8lZQ/640?wx_fmt=jpeg&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

# 2 用户空间与内核空间数据交换的文件系统

---

内核中有三个常用的伪文件系统: procfs, debugfs和sysfs.

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

它们都用于Linux内核和用户空间的数据交换, 但是适用的场景有所差异：

- `procfs` 历史最早, 最初就是用来跟内核交互的唯一方式, 用来获取处理器、内存、设备驱动、进程等各种信息.
    
- `sysfs` 跟 `kobject` 框架紧密联系, 而 `kobject` 是为设备驱动模型而存在的, 所以 `sysfs` 是为设备驱动服务的.
    
- `debugfs` 从名字来看就是为 `debug` 而生, 所以更加灵活.
    
- `relayfs` 是一个快速的转发 `(relay)` 数据的文件系统, 它以其功能而得名. 它为那些需要从内核空间转发大量数据到用户空间的工具和应用提供了快速有效的转发机制.
    

**相关资料链接：**

在 Linux 下用户空间与内核空间数据交换的方式, 第 2 部分: procfs、seq_file、debugfs和relayfs：http://www.ibm.com/developerworks/cn/linux/l-kerns-usrs2/

Linux 文件系统：procfs, sysfs, debugfs 用法简介：http://www.tinylab.org/show-the-usage-of-procfs-sysfs-debugfs/

## 2.1 procfs文件系统

---

- `ProcFs` 介绍
    

`procfs` 是比较老的一种用户态与内核态的数据交换方式, 内核的很多数据都是通过这种方式出口给用户的, 内核的很多参数也是通过这种方式来让用户方便设置的. 除了 `sysctl` 出口到 `/proc` 下的参数, `procfs` 提供的大部分内核参数是只读的. 实际上, 很多应用严重地依赖于procfs, 因此它几乎是必不可少的组件. 前面部分的几个例子实际上已经使用它来出口内核数据, 但是并没有讲解如何使用, 本节将讲解如何使用`procfs`.

- 参考资料：
    
    用户空间与内核空间数据交换的方式(2)——procfs：http://www.cnblogs.com/hoys/archive/2011/04/10/2011141.html
    

## 2.2 sysfs文件系统

---

内核子系统或设备驱动可以直接编译到内核, 也可以编译成模块, 编译到内核, 使用前一节介绍的方法通过内核启动参数来向它们传递参数, 如果编译成模块, 则可以通过命令行在插入模块时传递参数, 或者在运行时, 通过 `sysfs` 来设置或读取模块数据.

`Sysfs` 是一个基于内存的文件系统, 实际上它基于`ramfs`, `sysfs` 提供了一种把内核数据结构, 它们的属性以及属性与数据结构的联系开放给用户态的方式, 它与 `kobject` 子系统紧密地结合在一起, 因此内核开发者不需要直接使用它, 而是内核的各个子系统使用它. 用户要想使用 `sysfs` 读取和设置内核参数, 仅需装载 `sysfs` 就可以通过文件操作应用来读取和设置内核通过 `sysfs` 开放给用户的各个参数：

```
mkdir -p /sysfs
```

注意, 不要把 `sysfs` 和 `sysctl` 混淆, `sysctl` 是内核的一些控制参数, 其目的是方便用户对内核的行为进行控制, 而 `sysfs` 仅仅是把内核的 `kobject` 对象的层次关系与属性开放给用户查看, 因此 `sysfs`的绝大部分是只读的, 模块作为一个 `kobject` 也被出口到 `sysfs`, 模块参数则是作为模块属性出口的, 内核实现者为模块的使用提供了更灵活的方式, 允许用户设置模块参数在 `sysfs` 的可见性并允许用户在编写模块时设置这些参数在 `sysfs` 下的访问权限, 然后用户就可以通过 `sysfs` 来查看和设置模块参数, 从而使得用户能在模块运行时控制模块行为.

**相关资料链接：**

用户空间与内核空间数据交换的方式(6)——模块参数与sysfs：http://www.cnblogs.com/hoys/archive/2011/04/10/2011470.html

## 2.3 debugfs文件系统

---

内核开发者经常需要向用户空间应用输出一些调试信息, 在稳定的系统中可能根本不需要这些调试信息, 但是在开发过程中, 为了搞清楚内核的行为, 调试信息非常必要, printk可能是用的最多的, 但它并不是最好的, 调试信息只是在开发中用于调试, 而 `printk` 将一直输出, 因此开发完毕后需要清除不必要的 `printk` 语句, 另外如果开发者希望用户空间应用能够改变内核行为时, `printk` 就无法实现.

因此, 需要一种新的机制, 那只有在需要的时候使用, 它在需要时通过在一个虚拟文件系统中创建一个或多个文件来向用户空间应用提供调试信息.

有几种方式可以实现上述要求：

- 使用 `procfs`, 在 `/proc` 创建文件输出调试信息, 但是 `procfs` 对于大于一个内存页(对于 `x86` 是 `4K`)的输出比较麻烦, 而且速度慢, 有时回出现一些意想不到的问题.
    
- 使用 `sysfs`( `2.6` 内核引入的新的虚拟文件系统), 在很多情况下, 调试信息可以存放在那里, 但是sysfs主要用于系统管理，它希望每一个文件对应内核的一个变量，如果使用它输出复杂的数据结构或调试信息是非常困难的.
    
- 使用 `libfs` 创建一个新的文件系统, 该方法极其灵活, 开发者可以为新文件系统设置一些规则, 使用 `libfs` 使得创建新文件系统更加简单, 但是仍然超出了一个开发者的想象.
    

为了使得开发者更加容易使用这样的机制, `Greg Kroah-Hartman` 开发了 `debugfs`(在 `2.6.11` 中第一次引入), 它是一个虚拟文件系统, 专门用于输出调试信息, 该文件系统非常小, 很容易使用, 可以在配置内核时选择是否构件到内核中, 在不选择它的情况下, 使用它提供的API的内核部分不需要做任何改动.

**相关资料链接：**

用户空间与内核空间数据交换的方式(1)——debugfs：http://www.cnblogs.com/hoys/archive/2011/04/10/2011124.html  

Linux内核里的DebugFS：http://www.cnblogs.com/wwang/archive/2011/01/17/1937609.html

Linux驱动调试的Debugfs的使用简介：http://soft.chinabyte.com/os/110/12377610.shtml

Linux Debugfs文件系统介绍及使用：http://blog.sina.com.cn/s/blog_40d2f1c80100p7u2.html

Linux内核里的DebugFS：http://www.cnblogs.com/wwang/archive/2011/01/17/1937609.html

Debugging the Linux Kernel with debugfs：http://opensourceforu.com/2010/10/debugging-linux-kernel-with-debugfs/

debugfs-seq_file：http://lxr.free-electrons.com/source/drivers/base/power/wakeup.c

Linux Debugfs文件系统介绍及使用：http://blog.sina.com.cn/s/blog_40d2f1c80100p7u2.html

Linux 文件系统：procfs, sysfs, debugfs 用法简介：http://www.tinylab.org/show-the-usage-of-procfs-sysfs-debugfs/

用户空间与内核空间数据交换的方式(1)——debugfs：http://www.cnblogs.com/hoys/archive/2011/04/10/2011124.html

Linux 运用debugfs调试方法：http://www.xuebuyuan.com/1023006.html

## 2.4 relayfs文件系统

---

`relayfs` 是一个快速的转发(`relay`)数据的文件系统, 它以其功能而得名. 它为那些需要从内核空间转发大量数据到用户空间的工具和应用提供了快速有效的转发机制.

`Channel` 是 `relayfs` 文件系统定义的一个主要概念, 每一个 `channel` 由一组内核缓存组成, 每一个 `CPU` 有一个对应于该 `channel` 的内核缓存, 每一个内核缓存用一个在 `relayfs` 文件系统中的文件文件表示, 内核使用 `relayfs` 提供的写函数把需要转发给用户空间的数据快速地写入当前 `CPU` 上的 `channel` 内核缓存, 用户空间应用通过标准的文件 `I/` O函数在对应的 `channel` 文件中可以快速地取得这些被转发出的数据 `mmap` 来. 写入到 `channel` 中的数据的格式完全取决于内核中创建`channel` 的模块或子系统.

`relayfs` 的用户空间`API` :

`relayfs` 实现了四个标准的文件 `I/O` 函数, `open、mmap、poll和close`

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

注意 : 用户态应用在使用上述 API 时必须保证已经挂载了 relayfs 文件系统, 但内核在创建和使用 channel时不需要relayfs 已经挂载. 下面命令将把 relayfs 文件系统挂载到 /mnt/relay.

**相关资料链接：**

用户空间与内核空间数据交换的方式(4)——relayfs：http://www.cnblogs.com/hoys/archive/2011/04/10/2011270.html

Relay：一种内核到用户空间的高效数据传输技术：https://www.ibm.com/developerworks/cn/linux/l-cn-relay/

## 2.5 seq_file

---

一般地, 内核通过在 `procfs` 文件系统下建立文件来向用户空间提供输出信息, 用户空间可以通过任何文本阅读应用查看该文件信息, 但是 `procfs` 有一个缺陷, 如果输出内容大于1个内存页, 需要多次读,因此处理起来很难, 另外, 如果输出太大, 速度比较慢, 有时会出现一些意想不到的情况, `Alexander Viro` 实现了一套新的功能, 使得内核输出大文件信息更容易, 该功能出现在 `2.4.15`(包括 `2.4.15`)以后的所有 `2.4` 内核以及 `2.6` 内核中, 尤其是在 `2.6` 内核中，已经大量地使用了该功能

**相关资料链接：**

用户空间与内核空间数据交换的方式(3)——seq_file：http://www.cnblogs.com/hoys/archive/2011/04/10/2011261.html

内核proc文件系统与seq接口（4）—seq_file接口编程浅析：http://blog.chinaunix.net/uid-20543672-id-3235254.html

Linux内核中的seq操作：http://www.cnblogs.com/qq78292959/archive/2012/06/13/2547335.html

seq_file源码分析：http://www.cppblog.com/csjiaxin/articles/136681.html

用序列文件(seq_file)接口导出常用数据结构：http://blog.chinaunix.net/uid-317451-id-92670.html

seq_file机制：http://blog.csdn.net/a8039974/article/details/24052619

  

# 3 printk

---

在内核调试技术之中, 最简单的就是 `printk` 的使用了, 它的用法和C语言应用程序中的 `printf` 使用类似, 在应用程序中依靠的是 `stdio.h` 中的库, 而在 `linux` 内核中没有这个库, 所以在 `linux` 内核中,实现了自己的一套库函数, `printk` 就是标准的输出函数

**相关资料链接：**  

linux内核调试技术之printk：http://www.cnblogs.com/veryStrong/p/6218383.html

调整内核printk的打印级别：http://blog.csdn.net/tonywgx/article/details/17504001

linux设备驱动学习笔记–内核调试方法之printk：http://blog.csdn.net/itsenlin/article/details/43205983

  

# 4 ftrace && trace-cmd

---

## 4.1 trace && ftrace

---

`Linux`当前版本中, 功能最强大的调试、跟踪手段. 其最基本的功能是提供了动态和静态探测点, 用于探测内核中指定位置上的相关信息.

静态探测点, 是在内核代码中调用 `ftrace` 提供的相应接口实现, 称之为静态是因为, 是在内核代码中写死的, 静态编译到内核代码中的, 在内核编译后, 就不能再动态修改. 在开启 `ftrace` 相关的内核配置选项后, 内核中已经在一些关键的地方设置了静态探测点, 需要使用时, 即可查看到相应的信息.

动态探测点, 基本原理为 : 利用 `mcount` 机制, 在内核编译时, 在每个函数入口保留数个字节, 然后在使用 `ftrace`时, 将保留的字节替换为需要的指令, 比如跳转到需要的执行探测操作的代码。

`ftrace` 的作用是帮助开发人员了解 `Linux` 内核的运行时行为, 以便进行故障调试或性能分析.

最早 `ftrace` 是一个 `function tracer`, 仅能够记录内核的函数调用流程. 如今 `ftrace` 已经成为一个

`framework`, 采用 `plugin` 的方式支持开发人员添加更多种类的 `trace` 功能.

`Ftrace` 由 `RedHat` 的 `Steve Rostedt` 负责维护. 到 `2.6.30` 为止, 已经支持的 `tracer` 包括 :

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

这里还没有列出所有的 `tracer`, `ftrace` 是目前非常活跃的开发领域, 新的 `tracer` 将不断被加入内核。

**相关资料链接：**

ftrace和它的前端工具trace-cmd(深入了解Linux系统的利器)：http://blog.yufeng.info/archives/1012

ftrace 简介：https://www.ibm.com/developerworks/cn/linux/l-cn-ftrace/

内核性能调试–ftrace：http://blog.chinaunix.net/uid-20589411-id-3501525.html

使用 ftrace 调试 Linux 内核，第 1 部分：https://www.ibm.com/developerworks/cn/linux/l-cn-ftrace1

ftrace的使用：http://blog.csdn.net/cybertan/article/details/8258394

[转]Linux内核跟踪之trace框架分析：http://blog.chinaunix.net/uid-24063584-id-2642103.html

Linux trace使用入门：http://blog.csdn.net/jscese/article/details/46415531

## 4.2 ftrace前端工具trace-cmd

---

- trace-cmd 介绍
    

`trace-cmd` 和 开源的 `kernelshark` 均是内核`Ftrace` 的前段工具, 用于分分析核性能.

他们相当于是一个 `/sys/kernel/debug/tracing` 中文件系统接口的封装, 为用户提供了更加直接和方便的操作.

- 使用
    

```
#  收集信息
```

trace-cmd: A front-end for Ftrace：https://lwn.net/Articles/410200/

其本质就是对`/sys/kernel/debug/tracing/events` 下各个模块进行操作, 收集数据并解析

  

# 5 Kprobe && systemtap

---

## 5.1 内核kprobe机制

---

`kprobe` 是 `linux` 内核的一个重要特性, 是一个轻量级的内核调试工具, 同时它又是其他一些更高级的内核调试工具(比如 `perf` 和 `systemtap`)的 “基础设施”, 4.0版本的内核中, 强大的 `eBPF` 特性也寄生于 `kprobe` 之上, 所以 `kprobe` 在内核中的地位就可见一斑了.

`Kprobes` 提供了一个强行进入任何内核例程并从中断处理器无干扰地收集信息的接口. 使用 `Kprobes` 可以收集处理器寄存器和全局数据结构等调试信息。开发者甚至可以使用 `Kprobes` 来修改 寄存器值和全局数据结构的值.

如何高效地调试内核?

`printk` 是一种方法, 但是 `printk` 终归是毫无选择地全量输出, 某些场景下不实用, 于是你可以试一下`tracepoint`, 我使能 `tracepoint` 机制的时候才输出. 对于傻傻地放置 `printk` 来输出信息的方式,`tracepoint` 是个进步, 但是 `tracepoint` 只是内核在某些特定行为(比如进程切换)上部署的一些静态锚点, 这些锚点并不一定是你需要的, 所以你仍然需要自己部署`tracepoint`, 重新编译内核. 那么`kprobe` 的出现就很有必要了, 它可以在运行的内核中动态插入探测点, 执行你预定义的操作.

它的基本工作机制是 : 用户指定一个探测点, 并把一个用户定义的处理函数关联到该探测点, 当内核执行到该探测点时, 相应的关联函数被执行，然后继续执行正常的代码路径.

`kprobe` 实现了三种类型的探测点 : `kprobes`, `jprobes`和 `kretprobes`(也叫返回探测点). `kprobes` 是可以被插入到内核的任何指令位置的探测点, `jprobes` 则只能被插入到一个内核函数的入口, 而 `kretprobes` 则是在指定的内核函数返回时才被执行.

**相关资料链接：**

kprobe工作原理：http://blog.itpub.net/15480802/viewspace-1162094/

随想录(强大的kprobe)：http://blog.csdn.net/feixiaoxing/article/details/40351811

kprobe原理解析（一）：http://www.cnblogs.com/honpey/p/4575928.html

## 5.2 前端工具systemtap

---

`SystemTap` 是监控和跟踪运行中的 `Linux` 内核的操作的动态方法. 这句话的关键词是动态, 因为 `SystemTap` 没有使用工具构建一个特殊的内核, 而是允许您在运行时动态地安装该工具. 它通过一个`Kprobes` 的应用编程接口 (`API`) 来实现该目的.

`SystemTap` 与一种名为 `DTrace` 的老技术相似，该技术源于 `Sun Solaris` 操作系统. 在 `DTrace` 中, 开发人员可以用 `D` 编程语言(`C` 语言的子集, 但修改为支持跟踪行为)编写脚本. `DTrace` 脚本包含许多探针和相关联的操作, 这些操作在探针 “触发” 时发生. 例如, 探针可以表示简单的系统调用，也可以表示更加复杂的交互，比如执行特定的代码行

`DTrace` 是 `Solaris` 最引人注目的部分, 所以在其他操作系统中开发它并不奇怪. `DTrace` 是在 `Common Development and Distribution License (CDDL)` 之下发行的, 并且被移植到 `FreeBSD` 操作系统中.

另一个非常有用的内核跟踪工具是 `ProbeVue`, 它是 `IBM` 为 `IBM® AIX®` 操作系统 `6.1` 开发的. 您可以使用 `ProbeVue` 探查系统的行为和性能, 以及提供特定进程的详细信息. 这个工具使用一个标准的内核以动态的方式进行跟踪.

考虑到 `DTrace` 和 `ProbeVue` 在各自的操作系统中的巨大作用, 为 `Linux` 操作系统策划一个实现该功能的开源项目是势不可挡的. `SystemTap` 从 `2005` 年开始开发, 它提供与 `DTrace` 和 `ProbeVue` 类似的功能. 许多社区还进一步完善了它, 包括 `Red Hat`、`Intel`、`Hitachi` 和 `IBM` 等.

这些解决方案在功能上都是类似的, 在触发探针时使用探针和相关联的操作脚本.

**相关资料链接：**

SystemTap 学习笔记 - 安装篇：https://segmentfault.com/a/1190000000671438

Linux 自检和 SystemTap 用于动态内核分析的接口和语言：https://www.ibm.com/developerworks/cn/linux/l-systemtap/

Brendan’s blog Using SystemTap：http://dtrace.org/blogs/brendan/2011/10/15/using-systemtap/

内核调试神器SystemTap — 简介与使用（一）：http://blog.csdn.net/zhangskd/article/details/25708441

内核探测工具systemtap简介：http://www.cnblogs.com/hazir/p/systemtap_introduction.html

SystemTap Beginner：http://blog.csdn.net/kafeiflynn/article/details/6429976

使用systemtap调试linux内核：http://blog.csdn.net/heli007/article/details/7187748

Ubuntu Kernel Debuginfo：http://ddebs.ubuntu.com/pool/main/l/linux

Linux 下的一个全新的性能测量和调式诊断工具 Systemtap, 第 3 部分: Systemtap：https://www.ibm.com/developerworks/cn/linux/l-cn-systemtap3/

  

# 6 kgdb && kgtp

---

## 6.1 kgdb

---

- KDB 和 KGDB 合并, 并进入内核
    

`KGDB` 是大名鼎鼎的内核调试工具, 他是由 `KDB` 和 `KGDB` 项目合并而来.

`kdb` 是一个Linux系统的内核调试器, 它是由SGI公司开发的遵循GPL许可证的开放源码调试工具. `kdb` 嵌入在`Linux` 内核中. 为内核&&驱动程序员提供调试手段. 它适合于调试内核空间的程序代码. 譬如进行设备驱动程序调试. 内核模块的调试等.

`kgdb` 和 `kdb` 现在已经合并了. 对于一个正在运行的`kgdb` 而言, 可以使用 `gdbmonitor` 命令来使用 `kdb` 命令. 比如

```
(gdb)gdb monitor ps -A
```

就可以运行 `kdb` 的 `ps` 命令了.

分析一下 `kdb` 补丁和合入主线的 `kdb` 有啥不同

`kdb`跟 `kgdb` 合并之后, 也可以使用 `kgdb` 的`IO` 驱动(比如键盘), 但是同时也 `kdb`也丧失了一些功能.合并之后的`kdb`不在支持汇编级的源码调试. 因此它现在也是平台独立的.

1. kdump和kexec已经被移除。
    
2. 从/proc/meninfo中获取的信息比以前少了。
    
3. bt命令现在使用的是内核的backtracer，而不是kdb原来使用的反汇编。
    
4. 合并之后的kdb不在具有原来的反汇编（id命令）
    

总结一下 : `kdb` 和 `kgdb` 合并之后，系统中对这两种调试方式几乎没有了明显的界限，比如通过串口进行远程访问的时候，可以使用 `kgdb` 命令, 也可以使用 `kdb` 命令（使用gdb monitor实现）

## 6.2 KGTP

---

`KGTP` 是一个 实时 轻量级 `Linux` 调试器 和 跟踪器. 使用 `KGTP`

使用 `KGTP` 不需要在 `Linux` 内核上打 `PATCH` 或者重新编译, 只要编译KGTP模块并 `insmod` 就可以.

其让 `Linux` 内核提供一个远程 `GDB` 调试接口, 于是在本地或者远程的主机上的GDB可以在不需要停止内核的情况下用 `GDB tracepoint` 和其他一些功能 调试 和 跟踪 `Linux`.

即使板子上没有 `GDB` 而且其没有可用的远程接口, `KGTP` 也可以用离线调试的功能调试内核（见http://code.google.com/p/kgtp/wiki/HOWTOCN#/sys/kernel/debug/gtpframe和离线调试）。

KGTP支持 X86-32 ， X86-64 ， MIPS 和 ARM 。  
KGTP在Linux内核 2.6.18到upstream 上都被测试过。  
而且还可以用在 Android 上(见 http://code.google.com/p/kgtp/wiki/HowToUseKGTPinAndroid)

**相关资料链接：**

github-KGTP：https://github.com/teawater/kgtp

KGTP内核调试使用：http://blog.csdn.net/djinglan/article/details/15335653

KGTP中增加对GDB命令“set trace-buffer-size”的支持 - Week 5：http://blog.csdn.net/calmdownba/article/details/38659317

  

# 7 perf

---

`Perf` 是用来进行软件性能分析的工具。  
通过它, 应用程序可以利用 `PMU`, `tracepoint` 和内核中的特殊计数器来进行性能统计. 它不但可以分析指定应用程序的性能问题 (`per thread`). 也可以用来分析内核的性能问题, 当然也可以同时分析应用代码和内核，从而全面理解应用程序中的性能瓶颈.

最初的时候, 它叫做 `Performance counter`, 在 `2.6.31` 中第一次亮相. 此后他成为内核开发最为活跃的一个领域. 在 `2.6.32` 中它正式改名为 `Performance Event`, 因为 `perf` 已不再仅仅作为 `PMU` 的抽象, 而是能够处理所有的性能相关的事件.

使用 `perf`, 您可以分析程序运行期间发生的硬件事件，比如 `instructions retired` , `processor clock cycles` 等; 您也可以分析软件事件, 比如 `Page Fault` 和进程切换。

这使得 `Perf` 拥有了众多的性能分析能力, 举例来说，使用 `Perf` 可以计算每个时钟周期内的指令数, 称为 `IPC`, `IPC` 偏低表明代码没有很好地利用 `CPU`.

`Perf` 还可以对程序进行函数级别的采样, 从而了解程序的性能瓶颈究竟在哪里等等. `Perf` 还可以替代 `strace`, 可以添加动态内核 `probe` 点. 还可以做 `benchmark` 衡量调度器的好坏.

人们或许会称它为进行性能分析的”瑞士军刀”, 但我不喜欢这个比喻, 我觉得 `perf` 应该是一把世间少有的倚天剑.

金庸笔下的很多人都有对宝刀的癖好, 即便本领低微不配拥有, 但是喜欢, 便无可奈何. 我恐怕正如这些人一样, 因此进了酒馆客栈, 见到相熟或者不相熟的人, 就要兴冲冲地要讲讲那倚天剑的故事.

**相关资料链接：**

Perf – Linux下的系统性能调优工具，第 1 部分：https://www.ibm.com/developerworks/cn/linux/l-cn-perf1/index.html

perf Examples：http://www.brendangregg.com/perf.html

改进版的perf, Performance analysis tools based on Linux perf_events (aka perf) and ftrace：https://github.com/brendangregg/perf-tools

Perf使用教程：http://blog.chinaunix.net/uid-10540984-id-3854969.html

linux下的内核测试工具——perf使用简介：http://blog.csdn.net/trochiluses/article/details/10261339

perf 移植：http://www.cnblogs.com/helloworldtoyou/p/5585152.html

  

# 8 其他Tracer工具

---

## 8.1 LTTng

---

`LTTng` 是一个 `Linux` 平台开源的跟踪工具, 是一套软件组件, 可允许跟踪 `Linux` 内核和用户程序, 并控制跟踪会话(开始/停止跟踪、启动/停止事件 等等). 这些组件被绑定如下三个包 :

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**相关资料链接：**

Linux 平台开源的跟踪工具：LTTng：http://www.open-open.com/lib/view/open1413946397247.html

用 lttng 跟踪内核：http://blog.csdn.net/xsckernel/article/details/17794551

LTTng and LTTng project：http://blog.csdn.net/ganggexiongqi/article/details/6664331

## 8.2 eBPF

---

extended Berkeley Packet Filter（eBPF）是一个可以在事件上运行程序的高效内核虚拟机（JIT）。它可能最终会提供 ftrace 和 perf_events 的内核编程，并强化其他的 tracer。这是 Alexei Starovoitov 目前正在开发的，还没有完全集成，但是从4.1开始已经对一些优秀的工具有足够的内核支持了，如块设备I/O的延迟热图。可参考其主要作者 Alexei Starovoitov 的BPF slides和eBPF samples。

## 8.3 Ktap

---

ktap 在过去是一款前景很好的 tracer，它使用内核中的 lua 虚拟机处理，在没有调试信息的情况下在嵌入式设备上运行的很好。它分为几个步骤，并在有一段时间似乎超过了 Linux 上所有的追踪器。然后 eBPF 开始进行内核集成，而 ktap 的集成在它可以使用 eBPF 替代它自己的虚拟机后才开始。因为 eBPF 仍将持续集成几个月，ktap 开发者要继续等上一段时间。我希??今年晚些时候它能重新开发。

## 8.4 dtrace4linux

---

dtrace4linux 主要是 Paul Fox 一个人在业余时间完成的，它是 Sun DTrace 的 Linux 版本。它引入瞩目，还有一些 provider 可以运行，但是从某种程度上来说还不完整，更多的是一种实验性的工具（不安全）。我认为，顾忌到许可问题，人们会小心翼翼的为 dtrace4linux 贡献代码：由于当年 Sun 开源DTrace 使用的是 CDDL 协议，而 dtrace4linux 也不大可能最终进入 Linux kernel。Paul 的方法很可能会使其成为一个 add-on。我很乐意看到 Linux 平台上的 DTrace 和这个项目的完成，我认为当我加入 Netflix 后将会花些时间来协助完成这个项目。然而，我还是要继续使用内置的 tracers，如 ftrace 和 perf_events。

## 8.5 OL DTrace

---

Oracle Linux DTrace为了将 DTrace 引入 Linux，特别是 Oracle Linux，做出了很大的努力。这些年来发布的多个版本表明了它的稳定进展。开发者们以一种对这个项目的前景看好的态度谈论着改进 DTrace 测试套件。很多有用的 provider 已经完成了，如：syscall, profile, sdt, proc, sched 以及USDT。我很期待 fbt（function boundary tracing, 用于内核动态跟踪）的完成，它是 Linux 内核上非常棒的 provider。OL DTrace 最终的成功将取决于人们对运行 Oracle Linux（为技术支持付费）有多大兴趣，另一方面取决于它是否完全开源：它的内核元件是开源的，而我没有看到它的用户级别代码。

## 8.6 sysdig

---

sysdig是一个使用类tcpdump语法来操作系统事件的新tracer，它使用lua提交进程。它很优秀，它见证了系统跟踪领域的变革。它的局限性在于它只在当前进行系统调用，在提交进行时将所有事件转储为用户级别。你可以使用系统调用做很多事情，然而我还是很希望它能支持跟踪点、kprobe和uprobe。我还期待它能支持eBPF做内核摘要。目前，sysdig开发者正在增加容器支持。留意这些内容。

Read more

Reads 4669

​

Comment

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/SeWfibBcBT0EibtIWVNvshnuWMN1AoJw3poFIsbpaIVyZibCCqwBUR21rcDfrQgoqYzaYNdS14IIXzvmzvibdDa5Rw/300?wx_fmt=png&wxfrom=18)

Linux内核之旅

2849

Comment

Comment

**Comment**

暂无留言