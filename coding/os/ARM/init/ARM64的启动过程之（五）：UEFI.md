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

作者：[linuxer](http://www.wowotech.net/author/3 "linuxer") 发布于：2015-10-30 19:27 分类：[ARMv8A Arch](http://www.wowotech.net/sort/armv8a_arch)

一、前言

在准备大刀阔斧进入start_kernel之际，我又重新review了一下head.S文件，看看是否有一些遗漏的知识点，很不幸，看到了CONFIG_EFI这个配置项。当然，在一年前阅读kernel代码的时候就了解过相关的内容，但是，做为一个嵌入式工程师总是或多或少对其有些排斥，因此习惯性的忽略掉CONFIG_EFI相关的代码，逃避总不是办法，在本文中，我们一起来探讨ARM64平台上UEFI相关的内容。

二、背景介绍

1、UEFI是什么鬼？

在个人电脑刚兴趣的时代，能够进入BIOS（Basic Input/Output System）解决一些计算机的问题绝对是高手中的高手（当年我就是这么骗到老婆的）。所谓BIOS实际上就是IBM PC兼容机（多么古老的一个词汇啊）主板上的固件（firmware），这些固件可以在系统启动过程中初始化硬件，self test，加载bootloader或者OS kernel，并且能为OS提供一些基础的服务。由于各种存在的问题，后来，Intel提出来EFI（Extensible Firmware Interface）来取代BIOS interface。2005年，Intel终止了EFI规范的开发，替代它的是Unified EFI Forum负责的UEFI（Unified Extensible Firmware Interface）specification。UEFI在系统中的位置如下（图片来自wiki）：

[![uefi](http://www.wowotech.net/content/uploadfile/201510/2242f3c37e46ef4c38b152f43d9ac33320151030112749.gif "uefi")](http://www.wowotech.net/content/uploadfile/201510/e1acc41f3cee9de326092ff1aa24a84c20151030112749.gif)

随着PC和服务器的飞速发展，软件和硬件厂商都不断的研发各种新的产品来应对客户的需求，在整合成系统的时候，有大量的协调的工作需要做，并且是越来越复杂。为了加快整合，降低设计复杂度，需要一个统一的接口标准，也就是传说中的UEFI了。有了UEFI，OS（软件厂商阵营）和固件（硬件厂商阵营）就有了接口规格，这样，大家可以各自进行开发，只要符合UEFI规格就OK了。如果硬件厂商有了创新性的硬件特性，如果不需要修改UEFI接口，那么系统还是可以无缝的衔接，如果需要修改接口，那么提前修改接口规格，让参与整个系统构建的厂商可以同步前进。同样的，从软件角度看，如果创新性的软件算法需要HW的支持，那么可以通过UEFI这样的接口和硬件厂商阵营进行交互，大大加快了将整个系统交付给客户的时间。

2、UEFI关ARM什么事？

如果ARM仅仅是将目光放在移动（嵌入式）市场，那么UEFI当然不关ARM什么事情。在嵌入式ARM平台上，ROM code ＋ bootloader（例如Uboot）＋ linux kernel这样的组合可以很好的工作。但是，在推出ARMv8以及64 bit架构的的处理器之后，ARM的野心已经不满足在移动市场上称王了。不过嵌入式平台和server或者PC类的平台是有区别的：嵌入式平台往往是高度定制化的平台，各个硬件模块都是不可分割的。如果你购买了一个手机，如果你觉得LCD不满意，是不可能单独去市场购买一个LCD屏更换的。而server（PC）类产品则不然，各个模块是可以更换的。例如：可以自由的去购买一个硬盘或者显卡进行更换。

在移动平台上，firmware（ROM code）怎么做是自己的事情，只要在应用层面提供一致性的接口就OK了，反正硬件以及OS不会更换。来到服务器平台，ARM必须和她的合作伙伴（SOC，外围硬件，OS厂商等等）一起面对这样的问题：

（1）硬件平台（firmware）和OS之间的接口如何定义？

（2）如何向OS传递硬件信息？

为了让各个厂商能够协同工作，尽快将ARM服务器推向市场，选择一个标准让大家follow是一个不错的主意。我们以OS提供商为例描述选择标准的好处。如果定义了硬件平台和OS之间的标准，OS提供商可以为ARMv8 server发布一个image而不会因为任何一点硬件平台的修改就得发布一个新的OS。因此，ARMv8 server选择UEFI是很自然的事情了。

3、UEFI如何定义系统的启动过程？

相信大家对传统的嵌入式ARM平台的启动过程都是有所了解的，系统reset后，各个ARM SOC的从ROM代码开始执行（一般ARM reset之后，PC＝0，而ROM缺省地址就是0）。根据SOC厂商约定的规则，ROM code会从外部设备（串口、网络、NAND flash、USB磁盘设备或者其他磁盘设备）加载linux bootloader，bootloader会收集硬件信息，之后加载linux kernel。在UEFI规范中定义了BOOT manager，它会根据保存在NVRAM参数来决定如何load EFI Application（可能是bootloader或者其他的image file）。EFI Application的格式必须符合PE（Portable Executable ）格式。PE是一种二进制可执行文件的格式（在linux世界中，我们多半熟悉的是ELF格式），由微软开发，广泛应用在Windows平台上。

在ARMv8平台上，firmware中的boot manager可以加载支持UEFI的传统的bootloader（例如uboot），然后由uboot加载kernel，这样，kernel其实不必关心什么UEFI。当然这样有些不直观，本来OS kernel关心的那些firmeare提供的各种信息都是由bootloader进行转接，严重影响了系统整合的效率（bootloader和kernel是由不同的团队开发），因此，linux kernel image自身也可以包装成一个EFI image，由boot manager直接加载，完成启动过程。

4、PE格式介绍

下面的图片是一个PE文件格式的示意图：

[![pe-file1](http://www.wowotech.net/content/uploadfile/201510/5b233a14259075519388720198b41b3820151030112754.gif "pe-file1")](http://www.wowotech.net/content/uploadfile/201510/0b7b5a396f9e2728331cca89c688f0c220151030112750.gif)

PE文件主要由两部分组成，一部分是为了兼容MS-DOS操作系统而包装的外壳（灰色block），主要由64B的MZ header和MS-DOS stub代码区组成。在遥远的MSDOS时代，其可执行文件就需要这样的一个header，MSDOS的program loader就会根据这个header加载程序运行。在Windows时代，微软提出了PE这种格式文件，它主要是运行在windows系列的操作系统中，但是，还需要考虑MSDSO的兼容性（也就是说当MSDOS执行PE格式的文件也能够提供足够的信息让用户知道如何处理）。MS-DOS stub block是一段stub code，这段区域的主要作用是：当PE格式的image在MS-DOS下加载运行的时候，程序会执行这个区域的代码（PE的代码都是for windows的，不可能在DOS下实际执行，因此，只能执行这些stub程序），当然运行的结果仅仅是打印“This program cannot be run in DOS mode”。

另外一个区域就是实际的PE格式的文件了。主要包括PE header（绿色block）、各种Section header（蓝色block，用于描述各个section）和各个section的实际的Data。各个域的具体含义我们会结合具体的代码在下一章描述。

三、代码分析：

1、MZ header。相关代码如下所示：

> #ifdef CONFIG_EFI\
> efi_head:\
> add    x13, x18, #0x16 －－－－－－－－－－－－－－－－－－－－－－－－－－（2）\
> b    stext\
> #else\
> b    stext－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－（1）\
> .long    0 \
> #endif\
> .quad    \_kernel_offset_le－－－－－－－－－－－－－－－－－－－－－－－－－（3）\
> .quad    \_kernel_size_le \
> .quad    \_kernel_flags_le\
> .quad    0                // reserved\
> .quad    0                // reserved\
> .quad    0                // reserved\
> .byte    0x41－－－－－－－－－－－－Magic number, "ARM\\x64"\
> .byte    0x52\
> .byte    0x4d\
> .byte    0x64
>
> #ifdef CONFIG_EFI\
> .long    pe_header - efi_head－－－－－－－－－－－－－－－－－－－－－－－（4）\
> #else\
> .word    0                // reserved\
> #endif

这里定义了64字节的kernel image header，应对两种场景：一种是从普通的linux bootloader加载内核，另外一种是从UEFI firmware直接加载kernel（定义了CONFIG_EFI ），在这种场景下，这64B的内容被解释为MZ header。

（1）大部分的kernel image header都是相同的，除了第一个8-Byte和最后的4-Byte。没有定义CONFIG_EFI 是大家都比较熟悉的场景，当bootloader完成kernel image的从外设到RAM的搬移之后会执行kernel image的第一条指令。因此，这里是一条跳转到stext的指令。

（2）如果想把自己伪装成一个UEFI image，kernel需要符合PE格式，下面是一个简化版本的PE格式的示意图（仅仅包括部分格式，主要用来说明兼容MS-DOS 相关部分的内容）：

[![pe file header](http://www.wowotech.net/content/uploadfile/201510/004daaf2ef9b207203e3e27252a1bbb820151030112756.gif "pe file header")](http://www.wowotech.net/content/uploadfile/201510/46c93364c68c86d081689dbb3371624020151030112755.gif)

上图中的灰色区域就是64-Byte的MZ header（对应kernel image header的内容），当然，对于linux kernel而言，它只是伪装成PE格式而已，只要能够提供足够的信息给UEFI firmware的boot manager就OK了。PE格式的文件除了包括一个MZ header，还包括一段MS-DOS stub（上图中的黄色区域），当然，对于linux kernel image，我们没有提供这部分的内容。这里“add    x13, x18, #0x16”这条指令没有任何实际的意义，这条指令的opcode实际上就是MZ signature，用来标识这是一个DOS MZ executable的image。

（3）对于UEFI firmware而言，MS-DOS header大部分的区域都是没有什么用处的，因此正好可以用来提供信息，以便让linux的bootloader可以知道如何加载kernel（非UEFI加载的情况）。\_kernel_offset_le标识加载kernel的位置，如果等于0，表示加载到RAM的0地址的位置上。\_kernel_size_le表示需要加载的kernel image的长度，\_kernel_flags_le是表示kernel的一些属性，目前仅仅使用了bit 0，表示kernel的endianess。

（4）在UEFI firmware加载kernel的情况下，需要找到PE header以及各个section的定义了，以便boot manager完成加载kernel image的任务。在MS-DOS header中（offset是0x3c）有四个字节指向了PE header，通过它可以找到如何加载内核的各种信息。这个过程是这样的：UEFI firmware的boot manager如果发现了MZ header，那么就认为这是一个符合标准的EFI image，并在0x3c处获取PE header的位置，并继续解析其内容以便加载kernel image。

2、PE header相关代码

PE header包括三部分的内容：PE signature、COFF（Common Object File Format）file header和optional header。PE signature和COFF file header的代码如下：

> pe_header:\
> .ascii    "PE" －－－－－－－－－－－－－－－－PE header signanature\
> .short     0\
> coff_header:\
> .short    0xaa64－－－－－－－－－－表示machine type是AArch64\
> .short    2－－－－－－－－－－－－该PE文件有多少个section\
> .long    0－－－－－－－－－－－－该文件的创建时间\
> .long    0－－－－－－－－－－－－符号表信息\
> .long    1－－－－－－－－－－－－符号表中的符号的数目\
> .short    section_table - optional_header －－－－－－－－optional header的长度\
> .short    0x206－－－－－－－－－－－－－－－Characteristics，具体的含义请查看PE规格书

上节我们说过，通过MZ header可以找到PE header，所谓PE header的开始位置实际上就是一个“PE\\0\\0”的signature，随后紧接着就是COFF file header，COFF file header具体的定义如下（该表格来自PE specification）：

|   |   |   |   |
|---|---|---|---|
|**Offset**|**Size**|**Field**|**Description**|
|0|2|Machine|The number that identifies the type of target machine|
|2|2|NumberOfSections|The number of sections. This indicates the size of the section table, which immediately follows the headers.|
|4|4|TimeDateStamp|The low 32 bits of the number of seconds since 00:00 January 1, 1970 (a C run-time time_t value), that indicates when the file was created.|
|8|4|PointerToSymbolTable|The file offset of the COFF symbol table, or zero if no COFF symbol table is present. This value should be zero for an image because COFF debugging information is deprecated.|
|12|4|NumberOfSymbols|The number of entries in the symbol table. This data can be used to locate the string table, which immediately follows the symbol table. This value should be zero for an image because COFF debugging information is deprecated.|
|16|2|SizeOfOptionalHeader|The size of the optional header, which is required for executable files but not for object files. This value should be zero for an object file. For a description of the header format, see section 3.4, “Optional Header (Image Only).”|
|18|2|Characteristics|The flags that indicate the attributes of the file|

NumberOfSections定义了PE文件中的section的数目，对于linux kernel image的PE文件，包括了两个section，一个是.reloc section（这是EFI application loader需要的，我们这里只是提供了一个dummy版本的.reloc section），另外一个是.text section（整个kernel image）。

通过COFF file header中的SizeOfOptionalHeader域，UEFI firmware可以知道optional header的size。之所以是“optional”主要是因为这些header内容不一定会存在。例如：对于object文件，这些header不存在。当然，我们是UEFI image file（可执行文件），因此这些optional header是必须提供的。optional_header的最开始的域是optional header magic number，用来确定该PE文件是PE32还是PE32+格式的。根据UEFI规范，UEFI application file应该是PE32+格式的。PE32+格式的optional header格式如下：

|   |   |   |   |
|---|---|---|---|
|**Offset**|**Size**|**Header part**|**Description**|
|0|28/24|Standard fields|Fields that are defined for all implementations of COFF, including UNIX.|
|28/24|68/88|Windows-specific fields|Additional fields to support specific features of Windows (for example, subsystems).|
|96/112|Variable|Data directories|Address/size pairs for special tables that are found in the image file and are used by the operating system (for example, the import table and the export table).|

Standard fields包括了如何加载以及如何运行的信息。相关的代码如下：

> optional_header:\
> .short    0x20b                // PE32+ format\
> .byte    0x02                // MajorLinkerVersion\
> .byte    0x14                // MinorLinkerVersion\
> .long    \_end - stext            // SizeOfCode\
> .long    0                // SizeOfInitializedData\
> .long    0                // SizeOfUninitializedData\
> .long    efi_stub_entry - efi_head    // AddressOfEntryPoint\
> .long    stext_offset            // BaseOfCode

比较重要的信息包括：代码段在image file中的偏移（BaseOfCode），正文段的大小（SizeOfCode），data段的大小（SizeOfInitializedData），bss段的大小（SizeOfUninitializedData），加载到memory后入口函数（AddressOfEntryPoint，对于linux kernel而言，入口函数是efi_stub_entry）。

Windows-specific fields和Data directories主要被Windows操作系统的linker和loader使用的，这里就不详述了。

3、Section table和section Data

大家有兴趣可以自己查阅PE规格，我这里就偷懒啦，^\_^。

四、参考文献：

1、[https://lwn.net/Articles/584123/](https://lwn.net/Articles/584123/ "https://lwn.net/Articles/584123/")

2、[http://www.linaro.org/blog/when-will-uefi-and-acpi-be-ready-on-arm/](http://www.linaro.org/blog/when-will-uefi-and-acpi-be-ready-on-arm/ "http://www.linaro.org/blog/when-will-uefi-and-acpi-be-ready-on-arm/")

3、[https://lwn.net/Articles/574439/](https://lwn.net/Articles/574439/ "https://lwn.net/Articles/574439/")

4、PE规格书

5、UEFI规格书

_原创文章，转发请注明出处。蜗窝科技_

标签: [arm64](http://www.wowotech.net/tag/arm64) [UEFI](http://www.wowotech.net/tag/UEFI)

[![](http://www.wowotech.net/content/uploadfile/201605/ef3e1463542768.png)](http://www.wowotech.net/support_us.html)

« [linux kernel内存碎片防治技术](http://www.wowotech.net/memory_management/memory-fragment.html) | [Linux 3.18U盘无法正确使用](http://www.wowotech.net/226.html)»

**评论：**

**蓝漪**\
2017-03-08 16:50

窝窝文中多次提及的64字节，其计算引起了我的兴趣。\
我这边之前是这样算的：(4+4)+(6*8+4*1)+4=64，没错，和窝窝说的一样．\
但后面又查了下PE的结构，按照其数据结构，实际应该是40字节．

迷惑中，我对vmlinux进行了反汇编，相关结果如下：\
ffffffc000080000 \<\_text>:\
ffffffc000080000:   91005a4d    add x13, x18, #0x16\
ffffffc000080004:   140003ff    b   ffffffc000081000 <stext>\
ffffffc000080008:   00080000    .word   0x00080000\
ffffffc00008000c:   00000000    .word   0x00000000\
ffffffc000080010:   01b7c000    .word   0x01b7c000\
...\
ffffffc000080038:   644d5241    .word   0x644d5241\
ffffffc00008003c:   00000040    .word   0x00000040

这里可以看到，虽然定义的是quad类型，但实际vmlinux中是word类型．\
因此实际的计算公式应该是(4+4)+(6*4+4*1)+4=40．

这里有个疑问，为什么类型会变呢？

[回复](http://www.wowotech.net/armv8a_arch/UEFI.html#comment-5294)

**[linuxer](http://www.wowotech.net/)**\
2017-03-09 11:41

@蓝漪：类型当然不会变，.quad代表的就是8个字节的数据。

我帮你整理一下反汇编的结果你就明白了。

fffffc000080000 \<\_text>:\
ffffffc000080000:   91005a4d    add x13, x18, #0x16\
ffffffc000080004:   140003ff    b   ffffffc000081000 <stext>\
－－－－－－－－－－对应－－－－－－－－－－－－－－－\
efi_head:\
add    x13, x18, #0x16\
b    stext

ffffffc000080008:   00080000    .word   0x00080000\
ffffffc00008000c:   00000000    .word   0x00000000\
－－－－－－－－－－对应－－－－－－－－－－－－－－－\
.quad    \_kernel_offset_le

[回复](http://www.wowotech.net/armv8a_arch/UEFI.html#comment-5297)

**蓝漪**\
2017-03-09 22:29

@linuxer：嗯，这样理解的话也解决了我另一个疑惑（那个size怎么想也不应该是0的...）。\
那么计算的结果还是应该是64。\
但是我了解到的结构体是这样的：\
typedef struct \_IMAGE_DOS_HEADER {      // DOS .EXE header  \
WORD   e_magic;                     // Magic number  \
WORD   e_cblp;                      // Bytes on last page of file  \
WORD   e_cp;                        // Pages in file  \
WORD   e_crlc;                      // Relocations  \
WORD   e_cparhdr;                   // Size of header in paragraphs  \
WORD   e_minalloc;                  // Minimum extra paragraphs needed  \
WORD   e_maxalloc;                  // Maximum extra paragraphs needed  \
WORD   e_ss;                        // Initial (relative) SS value  \
WORD   e_sp;                        // Initial SP value  \
WORD   e_csum;                      // Checksum  \
WORD   e_ip;                        // Initial IP value  \
WORD   e_cs;                        // Initial (relative) CS value  \
WORD   e_lfarlc;                    // File address of relocation table  \
WORD   e_ovno;                      // Overlay number  \
WORD   e_res\[4\];                    // Reserved words  \
WORD   e_oemid;                     // OEM identifier (for e_oeminfo)  \
WORD   e_oeminfo;                   // OEM information; e_oemid specific  \
WORD   e_res2\[10\];                  // Reserved words  \
LONG   e_lfanew;                    // File address of new exe header  //PE头的偏移地址\
} IMAGE_DOS_HEADER, *PIMAGE_DOS_HEADER;  \
其中是18个word类型即18*2=36字节；加上最后的long4字节，只有40字节。\
既然这里是伪装成PE格式的，大小对不上，这个不应该的吧？

[回复](http://www.wowotech.net/armv8a_arch/UEFI.html#comment-5303)

**mahjong**\
2017-07-05 07:10

@蓝漪：e_lfanew指向PE header, dos header和pe header之间是长度不定的dos兼容代码

[回复](http://www.wowotech.net/armv8a_arch/UEFI.html#comment-5772)

**小豌豆**\
2016-05-09 11:08

感谢楼主

[回复](http://www.wowotech.net/armv8a_arch/UEFI.html#comment-3900)

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

  - [Linux设备模型(1)\_基本概念](http://www.wowotech.net/device_model/13.html)
  - [實作 spinlock on raspberry pi 2](http://www.wowotech.net/231.html)
  - [关于spin_lock的问题](http://www.wowotech.net/linux_kenrel/about_spin_lock.html)
  - [Linux reset framework](http://www.wowotech.net/pm_subsystem/reset_framework.html)
  - [X-013-UBOOT-使能autoboot功能](http://www.wowotech.net/x_project/uboot_autoboot.html)

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
