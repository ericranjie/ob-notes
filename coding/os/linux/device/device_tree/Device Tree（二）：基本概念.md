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

作者：[linuxer](http://www.wowotech.net/author/3 "linuxer") 发布于：2014-5-30 16:47 分类：[统一设备模型](http://www.wowotech.net/sort/device_model)

一、前言

一些背景知识（例如：为何要引入Device Tree，这个机制是用来解决什么问题的）请参考[引入Device Tree的原因](http://www.wowotech.net/linux_kenrel/why-dt.html)，本文主要是介绍Device Tree的基础概念。

简单的说，如果要使用Device Tree，首先用户要了解自己的硬件配置和系统运行参数，并把这些信息组织成Device Tree source file。通过DTC（Device Tree Compiler），可以将这些适合人类阅读的Device Tree source file变成适合机器处理的Device Tree binary file（有一个更好听的名字，DTB，device tree blob）。在系统启动的时候，boot program（例如：firmware、bootloader）可以将保存在flash中的DTB copy到内存（当然也可以通过其他方式，例如可以通过bootloader的交互式命令加载DTB，或者firmware可以探测到device的信息，组织成DTB保存在内存中），并把DTB的起始地址传递给client program（例如OS kernel，bootloader或者其他特殊功能的程序）。对于计算机系统（computer system），一般是firmware->bootloader->OS，对于嵌入式系统，一般是bootloader->OS。

本文主要描述下面两个主题：

1、Device Tree source file语法介绍

2、Device Tree binaryfile格式介绍

二、Device Tree的结构

在描述Device Tree的结构之前，我们先问一个基础问题：是否Device Tree要描述系统中的所有硬件信息？答案是否定的。基本上，那些可以动态探测到的设备是不需要描述的，例如USB device。不过对于SOC上的usb host controller，它是无法动态识别的，需要在device tree中描述。同样的道理，在computer system中，PCI device可以被动态探测到，不需要在device tree中描述，但是PCI bridge如果不能被探测，那么就需要描述之。

为了了解Device Tree的结构，我们首先给出一个Device Tree的示例：

> / o device-tree  
>       |- name = "device-tree"  
>       |- model = "MyBoardName"  
>       |- compatible = "MyBoardFamilyName"  
>       |- #address-cells = <2>  
>       |- #size-cells = <2>  
>       |- linux,phandle = <0>  
>       |  
>       o cpus  
>       | | - name = "cpus"  
>       | | - linux,phandle = <1>  
>       | | - #address-cells = <1>  
>       | | - #size-cells = <0>  
>       | |  
>       | o PowerPC,970@0  
>       |   |- name = "PowerPC,970"  
>       |   |- device_type = "cpu"  
>       |   |- reg = <0>  
>       |   |- clock-frequency = <0x5f5e1000>  
>       |   |- 64-bit  
>       |   |- linux,phandle = <2>  
>       |  
>       o memory@0  
>       | |- name = "memory"  
>       | |- device_type = "memory"  
>       | |- reg = <0x00000000 0x00000000 0x00000000 0x20000000>  
>       | |- linux,phandle = <3>  
>       |  
>       o chosen  
>         |- name = "chosen"  
>         |- bootargs = "root=/dev/sda2"  
>         |- linux,phandle = <4>

从上图中可以看出，device tree的基本单元是node。这些node被组织成树状结构，除了root node，每个node都只有一个parent。一个device tree文件中只能有一个root node。每个node中包含了若干的property/value来描述该node的一些特性。每个node用节点名字（node name）标识，节点名字的格式是[node-name@unit-address](mailto:node-name@unit-address)。如果该node没有reg属性（后面会描述这个property），那么该节点名字中必须不能包括@和unit-address。unit-address的具体格式是和设备挂在那个bus上相关。例如对于cpu，其unit-address就是从0开始编址，以此加一。而具体的设备，例如以太网控制器，其unit-address就是寄存器地址。root node的node name是确定的，必须是“/”。

在一个树状结构的device tree中，如何引用一个node呢？要想唯一指定一个node必须使用full path，例如/node-name-1/node-name-2/node-name-N。在上面的例子中，cpu node我们可以通过/cpus/PowerPC,970@0访问。

属性（property）值标识了设备的特性，它的值（value）是多种多样的：

1、可能是空，也就是没有值的定义。例如上图中的64-bit ，这个属性没有赋值。

2、可能是一个u32、u64的数值（值得一提的是cell这个术语，在Device Tree表示32bit的信息单位）。例如#address-cells = <1> 。当然，可能是一个数组。例如<0x00000000 0x00000000 0x00000000 0x20000000>

4、可能是一个字符串。例如device_type = "memory" ，当然也可能是一个string list。例如"PowerPC,970"

三、Device Tree source file语法介绍

了解了基本的device tree的结构后，我们总要把这些结构体现在device tree source code上来。在linux kernel中，扩展名是dts的文件就是描述硬件信息的device tree source file，在dts文件中，一个node被定义成：

> [label:] node-name[@unit-address] {  
>    [properties definitions]  
>    [child nodes]  
> }

“[]”表示option，因此可以定义一个只有node name的空节点。label方便在dts文件中引用，具体后面会描述。child node的格式和node是完全一样的，因此，一个dts文件中就是若干嵌套组成的node，property以及child note、child note property描述。

考虑到空泛的谈比较枯燥，我们用实例来讲解Device Tree Source file 的数据格式。假设蜗窝科技制作了一个S3C2416的开发板，我们把该development board命名为snail，那么需要撰写一个s3c2416-snail.dts的文件。如果把所有的开发板的硬件信息（SOC以及外设）都描述在一个文件中是不合理的，因此有可能其他公司也使用S3C2416搭建自己的开发板并命令pig、cow什么的，如果大家都用自己的dts文件描述硬件，那么其中大部分是重复的，因此我们把和S3C2416相关的硬件描述保存成一个单独的dts文件可以供使用S3C2416的target board来引用并将文件的扩展名变成dtsi（i表示include）。同理，三星公司的S3C24xx系列是一个SOC family，这些SOCs（2410、2416、2450等）也有相同的内容，因此同样的道理，我们可以将公共部分抽取出来，变成s3c24xx.dtsi，方便大家include。同样的道理，各家ARM vendor也会共用一些硬件定义信息，这个文件就是skeleton.dtsi。我们自下而上（类似C＋＋中的从基类到顶层的派生类）逐个进行分析。

1、skeleton.dtsi。位于linux-3.14\arch\arm\boot\dts目录下，具体该文件的内容如下：

> / {  
>     #address-cells = <1>;  
>     #size-cells = <1>;  
>     chosen { };  
>     aliases { };  
>     memory { device_type = "memory"; reg = <0 0>; };  
> };

device tree顾名思义是一个树状的结构，既然是树，必然有根。“/”是根节点的node name。“{”和“}”之间的内容是该节点的具体的定义，其内容包括各种属性的定义以及child node的定义。chosen、aliases和memory都是sub node，sub node的结构和root node是完全一样的，因此，sub node也有自己的属性和它自己的sub node，最终形成了一个树状的device tree。属性的定义采用property ＝ value的形式。例如#address-cells和#size-cells就是property，而<1>就是value。value有三种情况：

1）属性值是text string或者string list，用双引号表示。例如device_type = "memory"

2）属性值是32bit unsigned integers，用尖括号表示。例如#size-cells = <1>

3）属性值是binary data，用方括号表示。例如`binary-property = [0x01 0x23 0x45 0x67]`

如果一个device node中包含了有寻址需求（要定义reg property）的sub node（后文也许会用child node，和sub node是一样的意思），那么就必须要定义这两个属性。“#”是number的意思，#address-cells这个属性是用来描述sub node中的reg属性的地址域特性的，也就是说需要用多少个u32的cell来描述该地址域。同理可以推断#size-cells的含义，下面对reg的描述中会给出更详细的信息。

chosen node主要用来描述由系统firmware指定的runtime parameter。如果存在chosen这个node，其parent node必须是名字是“/”的根节点。原来通过tag list传递的一些linux kernel的运行时参数可以通过Device Tree传递。例如command line可以通过bootargs这个property这个属性传递；initrd的开始地址也可以通过linux,initrd-start这个property这个属性传递。在本例中，chosen节点是空的，在实际中，建议增加一个bootargs的属性，例如：

> "root=/dev/nfs nfsroot=1.1.1.1:/nfsboot ip=1.1.1.2:1.1.1.1:1.1.1.1:255.255.255.0::usbd0:off console=ttyS0,115200 mem=64M@0x30000000"

通过该command line可以控制内核从usbnet启动，当然，具体项目要相应修改command line以便适应不同的需求。我们知道，device tree用于HW platform识别，runtime parameter传递以及硬件设备描述。chosen节点并没有描述任何硬件设备节点的信息，它只是传递了runtime parameter。

aliases 节点定义了一些别名。为何要定义这个node呢？因为Device tree是树状结构，当要引用一个node的时候要指明相对于root node的full path，例如/node-name-1/node-name-2/node-name-N。如果多次引用，每次都要写这么复杂的字符串多少是有些麻烦，因此可以在aliases 节点定义一些设备节点full path的缩写。skeleton.dtsi中没有定义aliases，下面的section中会进一步用具体的例子描述之。

memory device node是所有设备树文件的必备节点，它定义了系统物理内存的layout。device_type属性定义了该node的设备类型，例如cpu、serial等。对于memory node，其device_type必须等于memory。reg属性定义了访问该device node的地址信息，该属性的值被解析成任意长度的（address，size）数组，具体用多长的数据来表示address和size是在其parent node中定义（#address-cells和#size-cells）。对于device node，reg描述了memory-mapped IO register的offset和length。对于memory node，定义了该memory的起始地址和长度。

本例中的物理内存的布局并没有通过memory node传递，其实我们可以使用command line传递，我们command line中的参数[“mem=64M@0x30000000](mailto:%E2%80%9Cmem=64M@0x30000000)”已经给出了具体的信息。我们用另外一个例子来加深对本节描述的各个属性以及memory node的理解。假设我们的系统是64bit的，physical memory分成两段，定义如下：

RAM: starting address 0x0, length 0x80000000 (2GB)  
RAM: starting address 0x100000000, length 0x100000000 (4GB)

对于这样的系统，我们可以将root node中的#address-cells和#size-cells这两个属性值设定为2，可以用下面两种方法来描述物理内存：

> 方法1：
> 
> memory@0 {  
>     device_type = "memory";  
>     reg = <0x000000000 0x00000000 0x00000000 0x80000000  
>               0x000000001 0x00000000 0x00000001 0x00000000>;  
> };
> 
> 方法2：
> 
> memory@0 {  
>     device_type = "memory";  
>     reg = <0x000000000 0x00000000 0x00000000 0x80000000>;  
> };
> 
> memory@100000000 {  
>     device_type = "memory";  
>     reg = <0x000000001 0x00000000 0x00000001 0x00000000>;  
> };

2、s3c24xx.dtsi。位于linux-3.14\arch\arm\boot\dts目录下，具体该文件的内容如下（有些内容省略了，领会精神即可，不需要描述每一个硬件定义的细节）：

> #include "skeleton.dtsi"
> 
> / {  
>     compatible = "samsung,s3c24xx"; －－－－－－－－－－－－－－－－－－－（A）  
>     interrupt-parent = <&intc>; －－－－－－－－－－－－－－－－－－－－－－（B）
> 
>     aliases {  
>         pinctrl0 = &pinctrl_0; －－－－－－－－－－－－－－－－－－－－－－－－（C）  
>     };
> 
>     intc:interrupt-controller@4a000000 { －－－－－－－－－－－－－－－－－－（D）  
>         compatible = "samsung,s3c2410-irq";  
>         reg = <0x4a000000 0x100>;  
>         interrupt-controller;  
>         #interrupt-cells = <4>;  
>     };
> 
>     [serial@50000000](mailto:serial@50000000) { －－－－－－－－－－－－－－－－－－－－－－（E）   
>         compatible = "samsung,s3c2410-uart";  
>         reg = <0x50000000 0x4000>;  
>         interrupts = <1 0 4 28>, <1 1 4 28>;  
>         status = "disabled";  
>     };
> 
>     pinctrl_0: pinctrl@56000000 {－－－－－－－－－－－－－－－－－－（F）  
>         reg = <0x56000000 0x1000>;
> 
>         wakeup-interrupt-controller {  
>             compatible = "samsung,s3c2410-wakeup-eint";  
>             interrupts = <0 0 0 3>,  
>                      <0 0 1 3>,  
>                      <0 0 2 3>,  
>                      <0 0 3 3>,  
>                      <0 0 4 4>,  
>                      <0 0 5 4>;  
>         };  
>     };
> 
> ……  
> };

这个文件描述了三星公司的S3C24xx系列SOC family共同的硬件block信息。首先提出的问题就是：为何定义了两个根节点？按理说Device Tree只能有一个根节点，所有其他的节点都是派生于根节点的。我的猜测是这样的：Device Tree Compiler会对DTS的node进行合并，最终生成的DTB只有一个root node。OK，我们下面开始逐一分析：

（A）在描述compatible属性之前要先描述model属性。model属性指明了该设备属于哪个设备生产商的哪一个model。一般而言，我们会给model赋值“manufacturer,model”。例如model = "samsung,s3c24xx"。samsung是生产商，s3c24xx是model类型，指明了具体的是哪一个系列的SOC。OK，现在我们回到compatible属性，该属性的值是string list，定义了一系列的modle（每个string是一个model）。这些字符串列表被操作系统用来选择用哪一个driver来驱动该设备。假设定义该属性：compatible = “aaaaaa”, “bbbbb"。那么操作操作系统可能首先使用aaaaaa来匹配适合的driver，如果没有匹配到，那么使用字符串bbbbb来继续寻找适合的driver，对于本例，compatible = "samsung,s3c24xx"，这里只定义了一个modle而不是一个list。对于root node，compatible属性是用来匹配machine type的（在device tree代码分析文章中会给出更细致的描述）。对于普通的HW block的节点，例如interrupt-controller，compatible属性是用来匹配适合的driver的。

（B）具体各个HW block的interrupt source是如何物理的连接到interruptcontroller的呢？在dts文件中是用interrupt-parent这个属性来标识的。且慢，这里定义interrupt-parent属性的是root node，难道root node会产生中断到interrupt controller吗？当然不会，只不过如果一个能够产生中断的device node没有定义interrupt-parent的话，其interrupt-parent属性就是跟随parent node。因此，与其在所有的下游设备中定义interrupt-parent，不如统一在root node中定义了。

intc是一个lable，标识了一个device node（在本例中是标识了interrupt-controller@4a000000 这个device node）。实际上，interrupt-parent属性值应该是是一个u32的整数值（这个整数值在Device Tree的范围内唯一识别了一个device node，也就是phandle），不过，在dts文件中中，可以使用类似c语言的Labels and References机制。定义一个lable，唯一标识一个node或者property，后续可以使用&来引用这个lable。DTC会将lable转换成u32的整数值放入到DTB中，用户层面就不再关心具体转换的整数值了。

关于interrupt，我们值得进一步描述。在Device Tree中，有一个概念叫做interrupt tree，也就是说interrupt也是一个树状结构。我们以下图为例（该图来自Power_ePAPR_APPROVED_v1.1）：

[![it](http://www.wowotech.net/content/uploadfile/201405/672b228062cd3b04e412010e4d06b4cd20140530084657.gif "it")](http://www.wowotech.net/content/uploadfile/201405/efb4c78b62a8caf9ba7dfc3fffc165d620140530084653.gif)

系统中有一个interrrupt tree的根节点，device1、device2以及PCI host bridge的interrupt line都是连接到root interrupt controller的。PCI host bridge设备中有一些下游的设备，也会产生中断，但是他们的中断都是连接到PCI host bridge上的interrupt controller（术语叫做interrupt nexus），然后报告到root interrupt controller的。每个能产生中断的设备都可以产生一个或者多个interrupt，每个interrupt source（另外一个术语叫做interrupt specifier，描述了interrupt source的信息）都是限定在其所属的interrupt domain中。

在了解了上述的概念后，我们可以回头再看看interrupt-parent这个属性。其实这个属性是建立interrupt tree的关键属性。它指明了设备树中的各个device node如何路由interrupt event。另外，需要提醒的是interrupt controller也是可以级联的，上图中没有表示出来。那么在这种情况下如何定义interrupt tree的root呢？那个没有定义interrupt-parent的interrupt controller就是root。

（C）pinctrl0是一个缩写，他是/pinctrl@56000000的别名。这里同样也是使用了Labels and References机制。

（D）intc（node name是interrupt-controller@4a000000 ，我这里直接使用lable）是描述interrupt controller的device node。根据S3C24xx的datasheet，我们知道interrupt controller的寄存器地址从0x4a000000开始，长度为0x100（实际2451的interrupt的寄存器地址空间没有那么长，0x4a000074是最后一个寄存器），也就是reg属性定义的内容。interrupt-controller属性为空，只是用来标识该node是一个interrupt controller而不是interrupt nexus（interrupt nexus需要在不同的interrupt domains之间进行翻译，需要定义interrupt-map的属性，本文不涉及这部分的内容）。#interrupt-cells 和#address-cells概念是类似的，也就是说，用多少个u32来标识一个interrupt source。我们可以看到，在具体HW block的interrupt定义中都是用了4个u32来表示，例如串口的中断是这样定义的：

> interrupts = <1 0 4 28>, <1 1 4 28>;  

（E） 从reg属性可以serial controller寄存器地址从0x50000000 开始，长度为0x4000。对于一个能产生中断的设备，必须定义interrupts这个属性。也可以定义interrupt-parent这个属性，如果不定义，则继承其parent node的interrupt-parent属性。 对于interrupt属性值，各个interrupt controller定义是不一样的，有的用3个u32表示，有的用4个。具体上面的各个数字的解释权归相关的interrupt controller所有。对于中断属性的具体值的描述我们会在device tree的第三份文档－代码分析中描述。

（F）这个node是描述GPIO控制的。这个节点定义了一个wakeup-interrupt-controller 的子节点，用来描述有唤醒功能的中断源。

3、s3c2416.dtsi。位于linux-3.14\arch\arm\boot\dts目录下，具体该文件的内容如下（有些内容省略了，领会精神即可，不需要描述每一个硬件定义的细节）：

> #include "s3c24xx.dtsi"  
> #include "s3c2416-pinctrl.dtsi"
> 
> / {  
>     model = "Samsung S3C2416 SoC";   
>     compatible = "samsung,s3c2416"; －－－－－－－－－－－－－－－A
> 
>     cpus { －－－－－－－－－－－－－－－－－－－－－－－－－－－－B  
>         #address-cells = <1>;  
>         #size-cells = <0>;
> 
>         cpu {  
>             compatible = "arm,arm926ejs";  
>         };  
>     };
> 
>     interrupt-controller@4a000000 { －－－－－－－－－－－－－－－－－C  
>         compatible = "samsung,s3c2416-irq";  
>     };
> 
> ……
> 
> };

（A）在s3c24xx.dtsi文件中已经定义了compatible这个属性，在s3c2416.dtsi中重复定义了这个属性，一个node不可能有相同名字的属性，具体如何处理就交给DTC了。经过反编译，可以看出，DTC是丢弃掉了前一个定义。因此，到目前为止，compatible ＝ samsung,s3c2416。在s3c24xx.dtsi文件中定义了compatible的属性值被覆盖了。

（B）对于根节点，必须有一个cpus的child node来描述系统中的CPU信息。对于CPU的编址我们用一个u32整数就可以描述了，因此，对于cpus node，#address-cells 是1，而#size-cells是0。其实CPU的node可以定义很多属性，例如TLB，cache、频率信息什么的，不过对于ARM，这里只是定义了compatible属性就OK了，arm926ejs包括了所有的processor相关的信息。

（C）s3c24xx.dtsi文件和s3c2416.dtsi中都有[interrupt-controller@4a000000](mailto:interrupt-controller@4a000000)这个node，DTC会对这两个node进行合并，最终编译的结果如下：

> interrupt-controller@4a000000 {  
>         compatible = "samsung,s3c2416-irq";  
>         reg = <0x4a000000 0x100>;  
>         interrupt-controller;  
>         #interrupt-cells = <0x4>;  
>         linux,phandle = <0x1>;  
>         phandle = <0x1>;  
>     };

4、s3c2416-pinctrl.dtsi

  这个文件定义了pinctrl@56000000 这个节点的若干child node，主要用来描述GPIO的bank信息。

5、s3c2416-snail.dts

  这个文件应该定义一些SOC之外的peripherals的定义。

四、Device Tree binary格式

1、DTB整体结构

经过Device Tree Compiler编译，Device Tree source file变成了Device Tree Blob（又称作flattened device tree）的格式。Device Tree Blob的数据组织如下图所示：

[![dt](http://www.wowotech.net/content/uploadfile/201405/e8c156f0cf3751cd223cca8cbf96132920140530084707.gif "dt")](http://www.wowotech.net/content/uploadfile/201405/85adc8b356e424f83c425948b3f1628220140530084702.gif)

2、DTB header。

对于DTB header，其各个成员解释如下：

|   |   |
|---|---|
|header field name|description|
|magic|用来识别DTB的。通过这个magic，kernel可以确定bootloader传递的参数block是一个DTB还是tag list。|
|totalsize|DTB的total size|
|off_dt_struct|device tree structure block的offset|
|off_dt_strings|device tree strings block的offset|
|off_mem_rsvmap|offset to memory reserve map。有些系统，我们也许会保留一些memory有特殊用途（例如DTB或者initrd image），或者在有些DSP+ARM的SOC platform上，有写memory被保留用于ARM和DSP进行信息交互。这些保留内存不会进入内存管理系统。|
|version|该DTB的版本。|
|last_comp_version|兼容版本信息|
|boot_cpuid_phys|我们在哪一个CPU（用ID标识）上booting|
|dt_strings_size|device tree strings block的size。和off_dt_strings一起确定了strings block在内存中的位置|
|dt_struct_size|device tree structure block的size。和和off_dt_struct一起确定了device tree structure block在内存中的位置|

3、 memory reserve map的格式描述

这个区域包括了若干的reserve memory描述符。每个reserve memory描述符是由address和size组成。其中address和size都是用U64来描述。

  

4、device tree structure block的格式描述

device tree structure block区域是由若干的分片组成，每个分片开始位置都是保存了token，以此来描述该分片的属性和内容。共计有5种token：

（1）FDT_BEGIN_NODE (0x00000001)。该token描述了一个node的开始位置，紧挨着该token的就是node name（包括unit address）

（2）FDT_END_NODE (0x00000002)。该token描述了一个node的结束位置。

（3）FDT_PROP (0x00000003)。该token描述了一个property的开始位置，该token之后是两个u32的数据，分别是length和name offset。length表示该property value data的size。name offset表示该属性字符串在device tree strings block的偏移值。length和name offset之后就是长度为length具体的属性值数据。

（4）FDT_NOP (0x00000004)。

（5）FDT_END (0x00000009)。该token标识了一个DTB的结束位置。

一个可能的DTB的结构如下：

（1）若干个FDT_NOP（可选）

（2）FDT_BEGIN_NODE

              node name

              paddings

（3）若干属性定义。

（4）若干子节点定义。（被FDT_BEGIN_NODE和FDT_END_NODE包围）

（5）若干个FDT_NOP（可选）

（6）FDT_END_NODE

（7）FDT_END

  

5、device tree strings bloc的格式描述

device tree strings bloc定义了各个node中使用的属性的字符串表。由于很多属性会出现在多个node中，因此，所有的属性字符串组成了一个string block。这样可以压缩DTB的size。

  

_原创文章，转发请注明出处。蜗窝科技_，[www.wowotech.net。](http://www.wowotech.net/linux_kenrel/dt_basic_concept.html)

标签: [Device](http://www.wowotech.net/tag/Device) [tree](http://www.wowotech.net/tag/tree)

[![](http://www.wowotech.net/content/uploadfile/201605/ef3e1463542768.png)](http://www.wowotech.net/support_us.html)

« [Device Tree（三）：代码分析](http://www.wowotech.net/device_model/dt-code-analysis.html) | [Linux电源管理(4)_Power Management Interface](http://www.wowotech.net/pm_subsystem/pm_interface.html)»

**评论：**

**billxiang**  
2021-11-16 11:25

感觉和qemu的配置很像，哈哈哈

[回复](http://www.wowotech.net/device_model/dt_basic_concept.html#comment-8375)

**BeDook**  
2019-05-28 16:37

作为菜鸟，刚接触设备树，看到博主们和其他前辈的讨论和详细的解释，收获颇多，尤其是博主的讲解很有条理，对Linux内核方面可以说是庖丁解牛，功力深厚，这得多少年的积淀啊，向博主们学习，立马转成蜗窝粉！

[回复](http://www.wowotech.net/device_model/dt_basic_concept.html#comment-7443)

**[linux2607](http://blog.csdn.net/armfpga123?ref=toolbar)**  
2018-01-04 20:43

郭大侠，请问device tree中，16位的整数怎么表示？用什么符号括起来？

[回复](http://www.wowotech.net/device_model/dt_basic_concept.html#comment-6454)

**风露**  
2017-12-24 01:58

蜗窝大师，再请教一个问题：  
dt结构中，哪些node的名字是不能改变的（soc 、chosen不可以？），哪些是可以改变的？

[回复](http://www.wowotech.net/device_model/dt_basic_concept.html#comment-6415)

**hello-world**  
2017-12-25 12:09

@风露：这位大哥，你的问题太细了，而且又很泛，别人没有办法回答啊。

[回复](http://www.wowotech.net/device_model/dt_basic_concept.html#comment-6420)

**tmmdh**  
2019-03-11 19:38

@风露：我想你是想问设备树中的保留关键字吧？分析一下dtc源码就知道了

[回复](http://www.wowotech.net/device_model/dt_basic_concept.html#comment-7262)

**悟空不念经**  
2017-11-20 11:58

写的真好！

[回复](http://www.wowotech.net/device_model/dt_basic_concept.html#comment-6221)

**seajiajia**  
2017-10-17 16:22

你好，请问下嵌入式arm用dts这种机制，那arm server呢？似乎没有找到对应的dts文件？  
换句而言，一个server有多少内存资源，需要在device tree里边注明吗？不能动态改变？

[回复](http://www.wowotech.net/device_model/dt_basic_concept.html#comment-6118)

**[wowo](http://www.wowotech.net/)**  
2017-10-18 13:17

@seajiajia：没有看过server的平台，还真不知道。  
至于内存，linux mm中有hotplug机制，不知道server会不会用。

[回复](http://www.wowotech.net/device_model/dt_basic_concept.html#comment-6121)

**melo**  
2017-08-01 11:42

@linuxer @wowo 请问  突然想知道内核是如何知道设备树文件是怎么使用的呢  也就是说 设备树的使能？

[回复](http://www.wowotech.net/device_model/dt_basic_concept.html#comment-5864)

**hello**  
2017-08-01 17:39

@melo：都不知道你想要问什么问题。

[回复](http://www.wowotech.net/device_model/dt_basic_concept.html#comment-5865)

**kobj**  
2017-06-10 16:54

是不是父节点中定义的变量，子节点中没有定义 都会从父节点中继承下来呢？

[回复](http://www.wowotech.net/device_model/dt_basic_concept.html#comment-5652)

**sunshine.huang**  
2017-04-23 15:38

hi linuxer  
请教个问题  
  
关于这段：  
unit-address的具体格式是和设备挂在那个bus上相关。例如对于cpu，其unit-address就是从0开始编址，以此加一。而具体的设备，例如以太网控制器，其unit-address就是寄存器地址  
  
有什么途径可以获取unit-address全部格式的说明呢？比如你列举了这几个是从哪里获取得来的，除了你列举的这些，其余的具体还有哪些，分别都是什么格式的？  
  
谢谢！

[回复](http://www.wowotech.net/device_model/dt_basic_concept.html#comment-5480)

**hello world**  
2017-04-24 15:30

@sunshine.huang：Documentation/devicetree目录下可以找到所有相关的信息。

[回复](http://www.wowotech.net/device_model/dt_basic_concept.html#comment-5483)

**麦兜要努力**  
2017-02-22 14:58

@wowo请教一下版主，  
1. 本例中的物理内存的布局并没有通过memory node传递，其实我们可以使用command line传递，我们command line中的参数“mem=64M@0x30000000”已经给出了具体的信息。  
如果dtsi中有定义以下的memory node,然后同时dtsi里面又要bootcmd,那到底是通过memory node去传递还是command line去传递呢？如何区分呢？      
memory@0{  
        device_type = "memory";  
        reg = <0x00000000 0x02000000>;  
    };  
chosen {  
       bootargs = "console=ttyS0,38400 mem=32M@0x00000000 init=/init earlyprintk";  
  
}  
2.memory block定义有的节点有两种，一种是node name是memory@形态的，另外一种是node中定义了device_type属性并且其值是memory.参考dtsi里面的node,发现memory@形态中也有定义device_type,如  
  
        memory@0 {  
        device_type = "memory";  
        reg = <0x20000000 0x1f000000>, <0xc0000000 0x3f000000>;  
    };  
or  
        memory@70000000 {  
        device_type = "memory";  
        reg = <0x70000000 0x20000000>;  
    };  
感觉就有点混淆了?如果commandline里面没有传递mem=32M@0x00000000，发现memory@{}和memory{},在Virtual kernel memory layout的lowmem会有差别..  
  
希望能得到版主的解答！感激！

[回复](http://www.wowotech.net/device_model/dt_basic_concept.html#comment-5234)

**[linuxer](http://www.wowotech.net/)**  
2017-02-22 15:41

@麦兜要努力：第一个问题在本站的内存布局（http://www.wowotech.net/memory_management/memory-layout.html）文档中已经有描述，可以参考，简单来说：  
1、dts中的memory node给出内存布局信息。  
2、在过去，没有device tree的时代，mem这个命令行参数传递了memory bank的信息，内核根据这个信息来创建系统内存的初始布局。在ARM64中，由于强制使用device tree，因此mem这个启动参数失去了本来的意义，现在它只是定义了memory的上限（最大的系统内存地址），可以限制DTS传递过来的内存参数。  
  
这是arm64的处理，至于arm平台，你可以自行看看代码是如何实现的。

[回复](http://www.wowotech.net/device_model/dt_basic_concept.html#comment-5235)

1 [2](http://www.wowotech.net/device_model/dt_basic_concept.html/comment-page-2#comments) [3](http://www.wowotech.net/device_model/dt_basic_concept.html/comment-page-3#comments) [4](http://www.wowotech.net/device_model/dt_basic_concept.html/comment-page-4#comments) [5](http://www.wowotech.net/device_model/dt_basic_concept.html/comment-page-5#comments) [6](http://www.wowotech.net/device_model/dt_basic_concept.html/comment-page-6#comments)

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
    
    - [Linux时间子系统之（十二）：periodic tick](http://www.wowotech.net/timer_subsystem/periodic-tick.html)
    - [Linux设备模型(7)_Class](http://www.wowotech.net/device_model/class.html)
    - [ARM64的启动过程之（二）：创建启动阶段的页表](http://www.wowotech.net/armv8a_arch/create_page_tables.html)
    - [Why Memory Barriers中文翻译（下）](http://www.wowotech.net/kernel_synchronization/why-memory-barrier-2.html)
    - [Linux内核同步机制之（二）：Per-CPU变量](http://www.wowotech.net/kernel_synchronization/per-cpu.html)
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