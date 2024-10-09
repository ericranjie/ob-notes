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

作者：[wowo](http://www.wowotech.net/author/2 "runangaozhong@163.com") 发布于：2016-9-2 21:49 分类：[u-boot分析](http://www.wowotech.net/sort/u-boot)

#### 1. 前言

Linux kernel在ARM架构中引入device tree（全称是flattened device tree，后续将会以FDT代称）的时候\[1\]，其实怀揣了一个Unify Kernel的梦想----同一个Image，可以支持多个不同的平台。随着新的ARM64架构将FDT列为必选项，并将和体系结构有关的代码剥离之后，这个梦想已经接近实现：

> 在编译linux kernel的时候，不必特意的指定具体的架构和SOC，只需要告诉kernel本次编译需要支持哪些板级的platform即可，最终将会生成一个Kernel image，以及多个和具体的板子（哪个架构、哪个SOC、哪个版型）有关的FDT image（dtb文件）。
>
> bootloader在启动的时候，根据硬件环境，加载不同的dtb文件，即可使linux kernel运行在不同的硬件平台上，从而达到unify kernel的目标。

本文将基于嵌入式产品中普遍使用的u-boot，以其新的uImage格式（FIT image，Flattened uImage Tree）为例，介绍达到此目标的步骤，以及背后的思考和意义。

#### 2. Legacy uImage

从u-boot的角度看，它要boot一个二进制文件（例如kernel Image），需要了解该文件的一些信息，例如：

> 该文件的类型，如kernel image、dtb文件、ramdisk image等等？
>
> 该文件需要放在memory的哪个位置（加载地址）？
>
> 该文件需要从memory哪个位置开始执行（执行地址）？
>
> 该文件是否有压缩？
>
> 该文件是否有一些完整性校验的信息（如CRC）？
>
> 等等

结合“[X-010-UBOOT-使用booti命令启动kernel(Bubblegum-96平台)](http://www.wowotech.net/x_project/bubblegum_uboot_booti.html)”中有关booti的例子，上面信息被隐含在我们的命令行中了，例如：

> 通过DFU工具，将指定的image文件下载到指定的memory地址，间接的指定了二进制文件加载地址；
>
> booti命令本身，说明加载的文件类型是ARM64平台的Image文件；
>
> 通过booti的参数，可以指定Kernel Image、ramdisk、DTB文件的执行位置；
>
> 等等。

不过，这种做法缺点很明显（总结来说，就是太啰嗦了）：

> 需要往memory中搬不同的二进制文件（Kernel、DTB、ramdisk等）；
>
> boot指令（booti等）有比较复杂的参数；
>
> 无法灵活地处理二进制文件的校验、解压缩等操作；
>
> 等等。

为了解决上述缺点，u-boot自定义了一种Image格式----uImage。最初的时候，uImage的格式比较简单，就是为二进制文件加上一个header（具体可参考“[include/image.h](https://github.com/wowotechX/u-boot/blob/x_integration/include/image.h)”中的定义），标示该文件的特性。然后在boot该类型的Image时，从header中读取所需的信息，按照指示，进行相应的动作即可。这种原始的Image格式，称作Legacy uImage，其特征可总结为：

> 1）使用mkimage工具（位于u-boot source code的tools/mkimage中）生成。
>
> 2）支持OS Kernel Images、RAMDisk Images等多种类型的Image。
>
> 3）支持gzip、bzip2等压缩算法。
>
> 4）支持CRC32 checksums。
>
> 5）等等。

最后，之所以称作Legacy，说明又有新花样了，这种旧的方式，我们就不再过多关注了，拥抱新事物去吧。

#### 3. FIT uImage

##### 3.1 简介

device tree在ARM架构中普及之后，u-boot也马上跟进、大力支持，毕竟，美好的Unify kernel的理想，需要bootloader的成全。为了支持基于device tree的unify kernel，u-boot需要一种新的Image格式，这种格式需要具备如下能力：

> 1）Image中需要包含多个dtb文件。
>
> 2）可以方便的选择使用哪个dtb文件boot kernel。

综合上面的需求，u-boot推出了全新的image格式----FIT uImage，其中FIT是flattened image tree的简称。是不是觉得FIT和FDT（flattened device tree）有点像？没错，它利用了Device Tree Source files（DTS）的语法，生成的image文件也和dtb文件类似（称作itb），下面我们会详细描述。

##### 3.2 思路

为了简单，我们可以直接把FIT uImage类比为device tree的dtb文件，其生成和使用过程为\[2\]：

> image source file        mkimage + dtc                           transfer to target\
> +                -----------------------------> image file -----------------------------------> bootm\
> image data file(s)

其中image source file(.its)和device tree source file(.dts)类似，负责描述要生成的image file的信息（上面第2章描述的信息）。mkimage和dtc工具，可以将.its文件以及对应的image data file，打包成一个image file。我们将这个文件下载到么memory中，使用bootm命令就可以执行了。

##### 3.3 image source file的语法

image source file的语法和device tree source file完全一样（可参考\[3\]\[4\]\[5\]中的例子），只不过自定义了一些特有的节点，包括images、configurations等。说明如下：

1）images节点

指定所要包含的二进制文件，可以指定多种类型的多个文件，例如multi.its\[5\]中的包含了3个kernel image、2个ramdisk image、2个fdt image。每个文件都是images下的一个子node，例如：

> kernel@2 {\
> description = "2.6.23-denx";\
> data = /incbin/("./2.6.23-denx.bin.gz");\
> type = "kernel";\
> arch = "ppc";\
> os = "linux";\
> compression = "gzip";\
> load = \<00000000>;\
> entry = \<00000000>;\
> hash@1 {\
> algo = "sha1";\
> };\
> };

可以包括如下的关键字：

> description，描述，可以随便写；
>
> data，二进制文件的路径，格式为----/incbin/("path/to/data/file.bin")；
>
> type，二进制文件的类型，"kernel", "ramdisk", "flat_dt"等，具体可参考中\[6\]的介绍；
>
> arch，平台类型，“arm”, “i386”等，具体可参考中\[6\]的介绍；
>
> os，操作系统类型，linux、vxworks等，具体可参考中\[6\]的介绍；
>
> compression，二进制文件的压缩格式，u-boot会按照执行的格式解压；
>
> load，二进制文件的加载位置，u-boot会把它copy对应的地址上；
>
> entry，二进制文件入口地址，一般kernel Image需要提供，u-boot会跳转到该地址上执行；
>
> hash，使用的数据校验算法。

2）configurations

可以将不同类型的二进制文件，根据不同的场景，组合起来，形成一个个的配置项，u-boot在boot的时候，以配置项为单位加载、执行，这样就可以根据不同的场景，方便的选择不同的配置，实现unify kernel目标。还以multi.its\[5\]为例，

> configurations {\
> default = "config@1";
>
> config@1 {\
> description = "tqm5200 vanilla-2.6.23 configuration";\
> kernel = "kernel@1";\
> ramdisk = "ramdisk@1";\
> fdt = "fdt@1";\
> };
>
> config@2 {\
> description = "tqm5200s denx-2.6.23 configuration";\
> kernel = "kernel@2";\
> ramdisk = "ramdisk@1";\
> fdt = "fdt@2";\
> };
>
> [config@3](mailto:config@3) {\
> description = "tqm5200s denx-2.4.25 configuration";\
> kernel = "kernel@3";\
> ramdisk = "ramdisk@2";\
> };\
> };

它包含了3种配置，每种配置使用了不同的kernel、ramdisk和fdt，默认配置项由“default”指定，当然也可以在运行时指定。

##### 3.4 Image的编译和使用

FIT uImage的编译过程很简单，根据实际情况，编写image source file之后（假设名称为kernel_fdt.its），在命令行使用mkimage工具编译即可：

> $ mkimage -f kernel_fdt.its kernel_fdt.itb

其中-f指定需要编译的source文件，并在后面指定需要生成的image文件（一般以.itb为后缀，例如kernel_fdt.itb）。

Image文件生成后，也可以使用mkimage命令查看它的信息：

> $ mkimage -l kernel.itb

最后，我们可以使用dfu工具将生成的.idb文件，下载的memory的某个地址（没有特殊要求，例如0x100000），然后使用bootm命令即可启动，步骤包括：

1）使用iminfo命令，查看memory中存在的images和configurations。

2）使用bootm命令，执行默认配置，或者指定配置。

使用默认配置启动的话，可以直接使用bootm：

> bootm 0x100000

选择其它配置的话，可以指定配置名:

> bootm 0x100000#config@2

以上可参考“[doc/uImage.FIT/howto.txt](https://github.com/wowotechX/u-boot/blob/x_integration/doc/uImage.FIT/howto.txt)\[2\]”，具体细节我们会在后续的文章中结合实例说明。

#### 4. 总结

本文简单的介绍了u-boot为了实现Unify kernel所做的努力，但有一个问题，大家可以思考一下：bootloader的unify怎么保证呢？

> SOC厂家提供（固化、提供二进制文件等）？
>
> 格式统一，UEFI？

后面有时间的话，可以追着这个疑问研究研究。

#### 5. 参考文档

\[1\] [Device Tree（一）：背景介绍](http://www.wowotech.net/device_model/why-dt.html)

\[2\] [doc/uImage.FIT/howto.txt](https://github.com/wowotechX/u-boot/blob/x_integration/doc/uImage.FIT/howto.txt)

\[3\] [doc/uImage.FIT/kernel.its](https://github.com/wowotechX/u-boot/blob/x_integration/doc/uImage.FIT/kernel.its)

\[4\] [doc/uImage.FIT/kernel_fdt.its](https://github.com/wowotechX/u-boot/blob/x_integration/doc/uImage.FIT/kernel_fdt.its)

\[5\] [doc/uImage.FIT/multi.its](https://github.com/wowotechX/u-boot/blob/x_integration/doc/uImage.FIT/multi.its)

\[6\] [doc/uImage.FIT/source_file_format.txt](https://github.com/wowotechX/u-boot/blob/x_integration/doc/uImage.FIT/source_file_format.txt)

原创文章，转发请注明出处。蜗窝科技，[www.wowotech.net](http://www.wowotech.net/u-boot/fit_image_overview.html)。

标签: [u-boot](http://www.wowotech.net/tag/u-boot) [fit](http://www.wowotech.net/tag/fit) [uImage](http://www.wowotech.net/tag/uImage) [its](http://www.wowotech.net/tag/its) [itb](http://www.wowotech.net/tag/itb)

[![](http://www.wowotech.net/content/uploadfile/201605/ef3e1463542768.png)](http://www.wowotech.net/support_us.html)

« [Fix-Mapped Addresses](http://www.wowotech.net/memory_management/fixmap.html) | [Linux内存模型](http://www.wowotech.net/memory_management/memory_model.html)»

**评论：**

**jake**\
2020-03-19 17:09

wowo大神太厉害了，不经意发现了你的宝藏博客。最近一直在拜读，每篇都很有思想，全是干货。学到了很多，多谢，多谢。

[回复](http://www.wowotech.net/u-boot/fit_image_overview.html#comment-7922)

**albert**\
2020-03-12 17:31

压缩过的uimage，这里uncompressed是什么意思？\
20200306_13:59:16## Booting kernel from Legacy Image at 00007fc0 ...\
20200306_13:59:16   Image Name:   Linux-3.10.52\
20200306_13:59:16   Image Type:   ARM Linux Kernel Image (uncompressed)\
20200306_13:59:16   Data Size:    2600896 Bytes = 2.5 MiB

[回复](http://www.wowotech.net/u-boot/fit_image_overview.html#comment-7919)

**收信的加西亚**\
2019-08-03 12:53

厉害

[回复](http://www.wowotech.net/u-boot/fit_image_overview.html#comment-7569)

**Blackrose**\
2017-07-28 11:38

你好，我在网上看到embeddedlinux.org.cn引用这篇文章内容，但未属版权名，参考链接如下。\
http://www.embeddedlinux.org.cn/emb-linux/system-development/201707/14-6970.html

[回复](http://www.wowotech.net/u-boot/fit_image_overview.html#comment-5854)

**[wowo](http://www.wowotech.net/)**\
2017-07-28 11:53

@Blackrose：非常感谢，希望能多一些像您这样尊重原创的朋友。\
我已经发邮件提醒他们了。\
感谢~~~!!

[回复](http://www.wowotech.net/u-boot/fit_image_overview.html#comment-5855)

**左边**\
2016-09-08 15:24

有没有关于openwrt方面的知识？

[回复](http://www.wowotech.net/u-boot/fit_image_overview.html#comment-4545)

**[wowo](http://www.wowotech.net/)**\
2016-09-08 18:20

@左边：openwrt更倾向与OS层面吧？我们平时很少涉及

[回复](http://www.wowotech.net/u-boot/fit_image_overview.html#comment-4549)

**ooonebook**\
2016-09-07 21:39

另外，还有一个问题，假设我加一个arm的kernel配置到fit_uImage.its中，make uImage的时候，会因为原来的./out/linux/arch/arm64/boot/Image并不存在导致编译错误。\
这种情况下，wowo有没有什么建议来做兼容比较合适，重新弄个fit_uImage.its文件吗？\
刚学习FIT，问题比较多~谢谢wowo~~

[回复](http://www.wowotech.net/u-boot/fit_image_overview.html#comment-4533)

**[wowo](http://www.wowotech.net/)**\
2016-09-07 21:54

@ooonebook：你要建立一个自己的image节点，然后把你的image的路径放进去，例如：\
kernel@5 {\
description = "ooonebook  kernel";\
data = /incbin/("./your path");\
type = "kernel";\
...\
};

[回复](http://www.wowotech.net/u-boot/fit_image_overview.html#comment-4536)

**ooonebook**\
2016-09-07 22:18

@wowo：类似这样加的，\
kernel@arm64 {\
description = "Unify(TODO) ARM64 Linux kernel";\
data = /incbin/("./out/linux/arch/arm64/boot/Image");\
type = "kernel";\
arch = "arm64";\
os = "linux";\
compression = "none";\
load = \<0x00080000>;\
entry = \<0x00080000>;\
};  \
kernel@arm {\
description = "Unify(TODO) ARM64 Linux kernel";\
data = /incbin/("./out/linux/arch/arm/boot/Image");\
type = "kernel";\
arch = "arm";\
os = "linux";\
compression = "none";\
load = \<0x30080000>;\
entry = \<0x30080000>;\
};  \
但是会报错\
/home/xys0610/code/project-x/build/out/u-boot/tools/mkimage -f /home/xys0610/code/project-x/build/fit_uImage.its /home/xys0610/code/project-x/build/out/xprj_uImage.itb\
FATAL ERROR: Couldn't open "./out/linux/arch/arm64/boot/Image": No such file or directory\
似乎这样子兼容不了

[回复](http://www.wowotech.net/u-boot/fit_image_overview.html#comment-4538)

**[wowo](http://www.wowotech.net/)**\
2016-09-07 22:19

@ooonebook：你要确认你所在的平台，使用的是什么格式的Image文件，你的是zImage？还是什么gz？arm下面肯定没有Image这个文件吧？

[回复](http://www.wowotech.net/u-boot/fit_image_overview.html#comment-4539)

**ooonebook**\
2016-09-07 22:46

@wowo：有的，而且测试过了，把原来的kernel@arm64节点和fdt节点屏蔽掉就OK。\
/home/xys0610/code/project-x/build/out/u-boot/tools/mkimage -f /home/xys0610/code/project-x/build/fit_uImage.its /home/xys0610/code/project-x/build/out/xprj_uImage.itb\
Can't set 'timestamp' property for '' node (FDT_ERR_NOSPACE)\
FIT description: U-Boot uImage source file for X project\
Created:         Wed Sep  7 07:44:08 2016\
Image 0 (kernel@arm)\
Description:  Unify(TODO) ARM64 Linux kernel\
Created:      Wed Sep  7 07:44:08 2016\
Type:         Kernel Image\
Compression:  uncompressed\
Data Size:    5295008 Bytes = 5170.91 kB = 5.05 MB\
Architecture: ARM\
OS:           Linux\
Load Address: 0x30080000\
Entry Point:  0x30080000\
Default Configuration: 'conf@bubblegum96'\
Configuration 0 (conf@bubblegum96)\
Description:  Boot Linux kernel with FDT blob\
Kernel:       kernel@arm64\
FDT:          fdt@bubblegum96\
Configuration 1 (conf@tiny210)\
Description:  Boot Linux kernel with FDT blob\
Kernel:       kernel@arm

您可以把Image改个名字然后再加个对应名字的节点测试一下看看。

[回复](http://www.wowotech.net/u-boot/fit_image_overview.html#comment-4540)

**ooonebook**\
2016-09-07 22:47

@ooonebook：不屏蔽掉就会报"./out/linux/arch/arm64/boot/Image": No such file or directory 的错

[回复](http://www.wowotech.net/u-boot/fit_image_overview.html#comment-4541)

**[wowo](http://www.wowotech.net/)**\
2016-09-08 09:28

@ooonebook：啊~我明白你的意思了。\
.its里面所有的BIN文件，都需要同时存在，才能编译通过。dtc解释器没办法过滤那些不存在的image。\
因此我们要在一个itb中支持多个平台的话，确实很麻烦。\
这里面还涉及一个问题：空间的浪费。例如Image相同，支持load、entry不同的话，size也要double。\
我觉得FIT uImage还有很多改进空间。\
当然，现在没有这种unify的需求~~~~~~

**ooonebook**\
2016-09-07 21:24

hi,wowo,有几个疑问请教一下：\
1、make -C $(KERNEL_DIR) CROSS_COMPILE=$(CROSS_COMPILE) KBUILD_OUTPUT=$(KERNEL_OUT_DIR) ARCH=$(BOARD_ARCH) $(KERNEL_DEFCONFIG)\
ifeq ($(BOARD_ARCH), arm64)\
KERNEL_DEFCONFIG=xprj_defconfig\
endif\
不是很明白这样改的意义是什么，毕竟不同平台可能会有区别，比如s5pv210就要配置上CONFIG_ARCH_S5PV210这个选项。\
2、从代码上看，bootm_decomp_image，uboot会把kernel部分再搬移到load地址上？\
所以说kernel.itb必须避开kernel可能的load区域，对吗？\
3、compression，二进制文件的压缩格式，u-boot会按照执行的格式解压；\
这个意思是，原来zImage上有一段解压代码，现在放在uboot解压？\
还是说mkimage的时候，会进行压缩，然后uboot中会对其进行解压？所以生成kernel镜像的时候其实不需要进行压缩？

[回复](http://www.wowotech.net/u-boot/fit_image_overview.html#comment-4532)

**[wowo](http://www.wowotech.net/)**\
2016-09-07 21:52

@ooonebook：问题1：因为ARM64架构，完全依赖device tree，也就是说，在不考虑kernel size的情况下，可以把所有的driver编译到kernel中，然后利用dtb文件区分，从这个角度看，一个config文件应该能够满足需求。如果不能，则要思考自己的软件设计是否有问题。当然arm架构，就另当别论了。\
问题2：是的，itb文件不能放在load地址上，不然会报错。稍后我会发一篇实践的文章，其中涉及到这里了。\
问题3：mkimage不会压缩，而是你的Image文件可以自行压缩，u-boot会在跳转前帮你解压。

[回复](http://www.wowotech.net/u-boot/fit_image_overview.html#comment-4535)

**ooonebook**\
2016-09-07 22:16

@wowo：谢谢wowo解答~

[回复](http://www.wowotech.net/u-boot/fit_image_overview.html#comment-4537)

**发表评论：**

昵称

邮件地址 (选填)

个人主页 (选填)

![](http://www.wowotech.net/include/lib/checkcode.php)

- ### 站内搜索

  蜗窝站内  互联网

- ### 功能

  [留言板\
  ](http://www.wowotech.net/message_board.html)[评论列表\
  ](http://www.wowotech.net/?plugin=commentlist)[支持者列表\
  ](http://www.wowotech.net/support_list)

- ### 最新评论

  - Shiina\
    [一个电路（circuit）中，由于是回路，所以用电势差的概念...](http://www.wowotech.net/basic_subject/voltage.html#8926)
  - Shiina\
    [其中比较关键的点是相对位置概念和点电荷的静电势能计算。](http://www.wowotech.net/basic_subject/voltage.html#8925)
  - leelockhey\
    [你这是哪个内核版本](http://www.wowotech.net/pm_subsystem/generic_pm_architecture.html#8924)
  - ja\
    [@dream：我看完這段也有相同的想法，引用 @dream ...](http://www.wowotech.net/kernel_synchronization/spinlock.html#8922)
  - 元神高手\
    [围观首席power managerment专家](http://www.wowotech.net/pm_subsystem/device_driver_pm.html#8921)
  - 十七\
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

  - [Linux的时钟](http://www.wowotech.net/timer_subsystem/clock-id-in-linux.html)
  - [Linux kernel debug技巧----开启DEBUG选项](http://www.wowotech.net/linux_application/kernel_debug_enable.html)
  - [eMMC 原理 1 ：Flash Memory 简介](http://www.wowotech.net/basic_tech/flash_memory_intro.html)
  - [Unix的历史](http://www.wowotech.net/tech_discuss/Unix%E7%9A%84%E5%8E%86%E5%8F%B2.html)
  - [Linux时间子系统之（六）：POSIX timer](http://www.wowotech.net/timer_subsystem/posix-timer.html)

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
