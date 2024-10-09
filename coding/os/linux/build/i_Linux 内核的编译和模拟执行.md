雄 小潘的CS学习笔记

_2024年03月30日 23:08_ _江苏_

小伙伴们大家好啊，此篇笔者将介绍内核的编译和模拟执行。在做好上一篇的准备后，我们将会使用交叉编译工具链来编译内核并生成最终可以在 arm 架构的 CPU 上执行的内核镜像（请注意此篇笔者执行每条命令时的目录）。

源代码需要用编译工具编译生成最终的可被执行的二进制文件，在笔者的 Ubuntu 系统上，直接使用 gcc 编译出来的内核是可以在 Intel x86 架构的 CPU 上执行。为了能编译出能在 arm 架构的 CPU 上运行的内核，就需要使用交叉编译工具链，它是用于在一种计算机体系结构上生成针对另一种不同计算机体系结构的目标代码的工具集合，在笔者的目录下则是上一篇中提到的 arm-linux-gnueabihf ，笔者并没有对其进行编译而是直接解压使用其编译工具，小伙伴可以 ls arm-linux-gnueabihf 目录下的 bin 目录（顺带说一下，笔者的目录结构就是上一篇中 _**tree**_ 命令列出来的结构）：

```c
pzx@pzx-vm:~/linux/arm-linux-gnueabihf$ ls -l bin | grep gcc
```

上面的 ls -l bin | grep gcc 指令需要将其分开来看，_**ls**_ 在 Linux 系统中是用来列出文件或者目录的指令，它可以显示出需要列出来的目录内容，上面的例子中\_**ls**\_ 就是将 arm-linux-gnueabihf/bin 目录下的内容列出，_**-l**_ 选项则是表示显示文件的权限、所有者、所属组、大小、修改日期等信息，读者可以尝试不加 _**-l**_ 选项查看输出结果是什么。接下来要说的是 _**grep**_ 命令，它在 Linux 中用来在指定文件中查找对应字符串的命令，所以指令的后半段则是要在文件中找出包含 gcc 字符串的内容。那么 _**grep**_ 要搜索的文件在哪里呢？这就需要提及两条指令中的竖线 **|** 了，它在 Linux 中是一个管道符，功能是将上一条指令的输入作为下一条指令的输出。有关管道相关内容还请读者自行查找资料。笔者不做赘述，所以上面命令的意思是列出 bin 目录下包含 gcc 字符串的内容。

为什么要介绍上面的内容呢？因为编译内核时我们将使用上述的 arm-linux-gnueabihf-gcc 编译器来编译内核的源码。由于笔者也是在网上查找教程，所以使用的是网上用的比较多的 vexpress-a9 开发板的配置来编译内核。

内核的编译十分简单，只需要切换到内核的目录下执行以下几条指令即可：

```c
make ARCH=arm CROSS_COMPILE=../arm-linux-gnueabihf/bin/arm-linux-gnueabihf- vexpress_defconfig
```

_**make**_ 是一个构建工具，用来编译代码，这也是上一篇笔者安装的软件之一，ARCH 和 CROSS_COMPILE后面介绍，vexpress_defconfig /zImage /modules /dtbs 则是我们要编译的目标，_**-j2**_ 表示同时启动两个编译任务以加快构建速度，大家也可以将 2 换成其他的数字。

ARCH=arm 表示要构建的 CPU 架构是 arm，CROSS_COMPILE=../arm-linux-gnueabihf/bin/arm-linux-gnueabihf- 则表示使用的编译工具的前缀是 CROSS_COMPILE=../arm-linux-gnueabihf/bin/arm-linux-gnueabihf-，为什么要强调是前缀呢？笔者简单的 _**grep**_ 一把 Makefile：

```c
pzx@pzx-vm:~/linux/linux-5.4.246$ cat Makefile | grep CROSS_COMPILE
```

可以看到上面的 CC = $(CROSS_COMPILE)gcc，CC 是我们编译内核使用的工具，$(CROSS_COMPILE) 在 Makefile 中表示使用此变量，因此展开就是 CC 为 ../arm-linux-gnueabihf/bin/arm-linux-gnueabihf-gcc，（读者可以阅读上面输出的以 # 开头的注释部分），这个编译工具就是前面 _**ls**_ 出来的工具。读者也可以直接在 Makefile 中改上面的这个变量以达到相同的功能。

内核编译完成后，生成的内核镜像文件是内核源码目录中 arch/arm/boot/ 目录的 zImage，设备树文件则是 arch/arm/boot/dts/ 目录的 vexpress-v2p-ca9.dtb。如果和笔者的编译方式不同，可以使用 _**find**_ 命令查找：

```c
pzx@pzx-vm:~/linux/linux-5.4.246$ find ./ -name zImage
```

有关 find 命令笔者在此篇就不过多描述，有了内核镜像文件和设备树文件后，我们就可以使用 qemu 模拟启动内核了：

```c
qemu-system-arm -M vexpress-a9 -m 256M -kernel ./arch/arm/boot/zImage -dtb ./arch/arm/boot/dts/vexpress-v2p-ca9.dtb
```

内核可以启动了，Booting Linux on physical CPU 0x0 的打印在内核源码的 arch/arm/kernel/setup.c 文件中，应该算是内核启动的第一条打印了。但是最后却以 Kernel panic 结束，这是由于未提供可挂载的根文件系统导致的，笔者后续会更新制作最简单的根文件系统以进入 Linux 的控制台，有关根文件系统，读者可以先参考下列资料：

https://refspecs.linuxfoundation.org/FHS_3.0/fhs/index.html

以上就是此篇的主要内容了，感谢大家可以阅读到此。有问题还请批评指正，谢谢。

阅读 1627

​

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/CpAhU3mTia4BBKBYs9Lx64EfX5BONeE1AwVb2wjjLZdfTxD4CZPtibNMib8iaWAAYTlPMn5g4VDILDFQ2cERgujTbA/300?wx_fmt=png&wxfrom=18)

小潘的CS学习笔记

关注

4108在看
