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

作者：[smcdef](http://www.wowotech.net/author/531) 发布于：2018-2-11 22:32 分类：[内存管理](http://www.wowotech.net/sort/memory_management)

1. 前言

KASAN是一个动态检测内存错误的工具。KASAN可以检测全局变量、栈、堆分配的内存发生越界访问等问题。功能比SLUB DEBUG齐全并且支持实时检测。越界访问的严重性和危害性通过我之前的文章（SLUB DEBUG技术）应该有所了解。正是由于SLUB DEBUG缺陷，因此我们需要一种更加强大的检测工具。难道你不想吗？KASAN就是其中一种。KASAN的使用真的很简单。但是我是一个追求刨根问底的人。仅仅止步于使用的层面，我是不愿意的，只有更清楚的了解实现原理才能更加熟练的使用工具。不止是KASAN，其他方面我也是这么认为。但是，说实话，写这篇文章是有点底气不足的。因为从我查阅的资料来说，国内没有一篇文章说KASAN的工作原理，国外也是没有什么文章关注KASAN的原理。大家好像都在说How to use。由于本人水平有限，就根据现有的资料以及自己阅读代码揣摩其中的意思。本文章作为抛准引玉，如果有不合理的地方还请指正。  
注：文章代码分析基于linux-4.15.0-rc3。

图片显示有点走形，请点开查看。

  
2. 简介  
KernelAddressSANitizer（KASAN）是一个动态检测内存错误的工具。它为找到use-after-free和out-of-bounds问题提供了一个快速和全面的解决方案。KASAN使用编译时检测每个内存访问，因此您需要GCC 4.9.2或更高版本。检测堆栈或全局变量的越界访问需要GCC 5.0或更高版本。目前KASAN仅支持x86_64和arm64架构（linux 4.4版本合入）。你使用ARM64架构，那么就需要保证linux版本在4.4以上。当然了，如果你使用的linux也有可能打过KASAN的补丁。例如，使用高通平台做手机的厂商使用linux 3.18同样支持KASAN。

  
3. 如何使用  
使用KASAN工具是比较简单的，只需要添加kernel以下配置项。  
CONFIG_SLUB_DEBUG=y  
CONFIG_KASAN=y  
为什么这里必须打开SLUB_DEBUG呢？是因为有段时间KASAN是依赖SLUBU_DEBUG的，什么意思呢？就是在Kconfig中使用了depends on，明白了吧。不过最新的代码已经不需要依赖了，可以看下提交。但是我建议你打开该选项，因为log可以输出更多有用的信息。重新编译kernel即可，编译之后你会发现boot.img（Android环境）大小大了一倍左右。所以说，影响效率不是没有道理的。不过我们可以作为产品发布前的最后检查，也可以排查越界访问等问题。我们可以查看内核日志内容是否包含KASAN检查出的bugs信息。

  
4. KASAN是如何实现检测的？  
KASAN的原理是利用额外的内存标记可用内存的状态。这部分额外的内存被称作shadow memory（影子区）。KASAN将1/8的内存用作shadow memory。使用特殊的magic num填充shadow memory，在每一次load/store（load/store检查指令由编译器插入）内存的时候检测对应的shadow memory确定操作是否valid。连续8 bytes内存（8 bytes align）使用1 byte shadow memory标记。如果8 bytes内存都可以访问，则shadow memory的值为0；如果连续N(1 =< N <= 7) bytes可以访问，则shadow memory的值为N；如果8 bytes内存访问都是invalid，则shadow memory的值为负数。  
 [![1.png](http://www.wowotech.net/content/uploadfile/201802/4a471518360139.png "点击查看原图")](http://www.wowotech.net/content/uploadfile/201802/4a471518360139.png)  
在代码运行时，每一次memory access都会检测对应的shawdow memory的值是否valid。这就需要编译器为我们做些工作。编译的时候，在每一次memory access前编译器会帮我们插入__asan_load##size()或者__asan_store##size()函数调用（size是访问内存字节的数量）。这也是要求更新版本gcc的原因，只有更新的版本才支持自动插入。  
mov x0, #0x5678  
movk x0, #0x1234, lsl #16  
movk x0, #0x8000, lsl #32  
movk x0, #0xffff, lsl #48  
mov w1, #0x5  
bl __asan_store1  
strb w1, [x0]  
上面一段汇编指令是往0xffff800012345678地址写5。在KASAN打开的情况下，编译器会帮我们自动插入bl __asan_store1指令，__asan_store1函数就是检测一个地址对应的shadow memory的值是否允许写1 byte。蓝色汇编指令就是真正的内存访问。因此KASAN可以在out-of-bounds的时候及时检测。__asan_load##size()和__asan_store##size()的代码在mm/kasan/kasan.c文件实现。

  
4.1. 如何根据shadow memory的值判断内存访问操作是否valid？

  
shadow memory检测原理的实现主要就是__asan_load##size()和__asan_store##size()函数的实现。那么KASAN是如何根据访问的address以及对应的shadow memory的状态值来判断访问是否合法呢？首先看一种最简单的情况。访问8 bytes内存。

long *addr = (long *)0xffff800012345678;  
*addr = 0;

以上代码是访问8 bytes情况，检测原理如下：

long *addr = (long *)0xffff800012345678;  
char *shadow = (char *)(((unsigned long)addr >> 3) + KASAN_SHADOW_OFFSE);  
if (*shadow)  
    report_bug();  
*addr = 0;

红色区域类似是编译器插入的指令。既然是访问8 bytes，必须要保证对应的shadow mempry的值必须是0，否则肯定是有问题。那么如果访问的是1,2 or 4 bytes该如何检查呢？也很简单，我们只需要修改一下if判断条件即可。修改如下：  
if (*shadow && *shadow < ((unsigned long)addr & 7) + N); //N = 1,2,4  
如果*shadow的值为0代表8 bytes均可以访问，自然就不需要report bug。addr & 7是计算访问地址相对于8字节对齐地址的偏移。还是使用下图来说明关系吧。假设内存是从地址8~15一共8 bytes。对应的shadow memory值为5，现在访问11地址。那么这里的N只要大于2就是invalid。

[![2.png](http://www.wowotech.net/content/uploadfile/201802/fb5c1519483501.png "点击查看原图")](http://www.wowotech.net/content/uploadfile/201802/fb5c1519483501.png)  
   
4.2. shadow memory内存如何分配？

  
在ARM64中，假设VA_BITS配置成48。那么kernel space空间大小是256TB，因此shadow memory的内存需要32TB。我们需要在虚拟地址空间为KASAN shadow memory分配地址空间。所以我们有必要了解一下ARM64 memory layout。  
基于linux-4.15.0-rc3的代码分析，我绘制了如下memory layout（VA_BITS = 48）。kernel space起始虚拟地址是0xffff_0000_0000_0000，kernel space被分成几个部分分别是KASAN、MODULE、VMALLOC、FIXMAP、PCI_IO、VMEMMAP以及linear mapping。其中KASAN的大小是32TB，正好是kernel space大小的1/8。不知道你注意到没有，KERNEL的位置相对以前是不是有所不一样。你的印象中，KERNEL是不是位于linear mapping区域，这里怎么变成了VMALLOC区域？这里是Ard Biesheuvel提交的修改。主要是为了迎接ARM64世界的KASLR（which allows the kernel image to be located anywhere in the vmalloc area）的到来。

[![3.png](http://www.wowotech.net/content/uploadfile/201802/10fb1518360141.png "点击查看原图")](http://www.wowotech.net/content/uploadfile/201802/10fb1518360141.png)  
   
4.3. 如何建立shadow memory的映射关系？

  
当打开KASAN的时候，KASAN区域位于kernel space首地址处，从0xffff_0000_0000_0000地址开始，大小是32TB。shadow memory和kernel address转换关系是：shadow_addr = (kaddr >> 3)  + KASAN_SHADOW_OFFSE。为了将[0xffff_0000_0000_0000, 0xffff_ffff_ffff_ffff]和[0xffff_0000_0000_0000, 0xffff_1fff_ffff_ffff]对应起来，因此计算KASAN_SHADOW_OFFSE的值为0xdfff_2000_0000_0000。我们将KASAN区域放大，如下图所示。  
 [![4.png](http://www.wowotech.net/content/uploadfile/201802/09dd1518360142.png "点击查看原图")](http://www.wowotech.net/content/uploadfile/201802/09dd1518360142.png)  
KASAN区域仅仅是分配的虚拟地址，在访问的时候必须建立和物理地址的映射才可以访问。上图就是KASAN建立的映射布局。左边是系统启动初期建立的映射。在kasan_early_init()函数中，将所有的KASAN区域映射到kasan_zero_page物理页面。因此系统启动初期，KASAN并不能工作。右侧是在kasan_init()函数中建立的映射关系，kasan_init()函数执行结束就预示着KASAN的正常工作。我们将不需要address sanitizer功能的区域同样还是映射到kasan_zero_page物理页面，并且是readonly。我们主要是检测kernel和物理内存是否存在UAF或者OOB问题。所以建立KERNEL和linear mapping（仅仅是所有的物理地址建立的映射区域）区域对应的shadow memory建立真实的映射关系。MOUDLE区域对应的shadow memory的映射关系也是需要创建的，但是映射关系建立是动态的，他在module加载的时候才会去创建映射关系。

  
4.4. 伙伴系统分配的内存的shadow memory值如何填充？

  
既然shadow memory已经建立映射，接下来的事情就是探究各种内存分配器向shadow memory填充什么数据了。首先看一下伙伴系统allocate page(s)函数填充shadow memory情况。  
 [![5.png](http://www.wowotech.net/content/uploadfile/201802/82661518360143.png "点击查看原图")](http://www.wowotech.net/content/uploadfile/201802/82661518360143.png)  
假设我们从buddy system分配4 pages。系统首先从order=2的链表中摘下一块内存，然后根据shadow memory address和memory address之间的对应的关系找对应的shadow memory。这里shadow memory的大小将会是2KB，系统会全部填充0代表内存可以访问。我们对分配的内存的任意地址内存进行访问的时候，首先都会找到对应的shadow memory，然后根据shadow memory value判断访问内存操作是否valid。  
如果释放pages，情况又是如何呢？  
 [![6.png](http://www.wowotech.net/content/uploadfile/201802/f19c1518360145.png "点击查看原图")](http://www.wowotech.net/content/uploadfile/201802/f19c1518360145.png)  
同样的，当释放pages的时候，会填充shadow memory的值为0xFF。如果释放之后，依然访问内存的话，此时KASAN根据shadow memory的值是0xFF就可以断，这是一个use-after-free问题。

  
4.5. SLUB分配对象的内存的shadow memory值如何填充？

  
当我们打开KASAN的时候，SLUB Allocator管理的object layout将会放生一定的变化。如下图所示。  
 [![7.png](http://www.wowotech.net/content/uploadfile/201802/9eb91518360146.png "点击查看原图")](http://www.wowotech.net/content/uploadfile/201802/9eb91518360146.png)  
在打开SLUB_DEBUG的时候，object就增加很多内存，KASAN打开之后，在此基础上又加了一截。为什么这里必须打开SLUB_DEBUG呢？是因为有段时间KASAN是依赖SLUBU_DEBUG的，什么意思呢？就是在Kconfig中使用了depends on，明白了吧。不过最新的代码已经不需要依赖了，可以看下提交。  
当我们第一次创建slab缓存池的时候，系统会调用kasan_poison_slab()函数初始化shadow memory为下图的模样。整个slab对应的shadow memory都填充0xFC。  
 [![8.png](http://www.wowotech.net/content/uploadfile/201802/602e1518360148.png "点击查看原图")](http://www.wowotech.net/content/uploadfile/201802/602e1518360148.png)  
上述步骤虽然填充了0xFC，但是接下来初始化object的时候，会改变一些shadow memory的值。我们先看一下kmalloc(20)的情况。我们知道kmalloc()就是基于SLUB Allocator实现的，所以会从kmalloc-32的kmem_cache中分配一个32 bytes object。  
 [![9.png](http://www.wowotech.net/content/uploadfile/201802/7afb1518360149.png "点击查看原图")](http://www.wowotech.net/content/uploadfile/201802/7afb1518360149.png)  
首先调用kmalloc(20)函数会匹配到kmalloc-32的kmem_cache，因此实际分配的object大小是32 bytes。KASAN同样会标记剩下的12 bytes的shadow memory为不可访问状态。根据object的地址，计算shadow memory的地址，并开始填充数值。由于kmalloc()返回的object的size是32 bytes，由于kmalloc(20)只申请了20 bytes，剩下的12 bytes不能使用。KASAN必须标记shadow memory这种情况。object对应的4 bytes shadow memory分别填充00 00 04 FC。00代表8个连续的字节可以访问。04代表前4个字节可以访问。作为越界访问的检测的方法。总共加在一起是正好是20 bytes可访问。0xFC是Redzone标记。如果访问了Redzone区域KASAN就会检测out-of-bounds的发生。  
当申请使用之后，现在调用kfree()释放之后的shadow memory情况是怎样的呢？看下图。  
 [![10.png](http://www.wowotech.net/content/uploadfile/201802/586e1518360151.png "点击查看原图")](http://www.wowotech.net/content/uploadfile/201802/586e1518360151.png)  
根据object首地址找到对应的shadow memory，32 bytes object对应4 bytes的shadow memory，现在填充0xFB标记内存是释放的状态。此时如果继续访问object，那么根据shadow memory的状态值既可以确定是use-after-free问题。

  
4.6. 全局变量的shadow memory值如何填充？

  
前面的分析都是基于内存分配器的，Redzone都会随着内存分配器一起分配。那么global variables如何检测呢？global variable的Redzone在哪里呢？这就需要编译器下手了。编译器会帮我们填充Redzone区域。例如我们定义一个全局变量a，编译器会帮我们填充成下面的样子。  
char a[4];  
转换

1. struct {
2.     char original[4];
3.     char redzone[60];
4. } a; //32 bytes aligned

  
如果这里你问我为什么填充60 bytes。其实我也不知道。这个转换例子也是从KASAN作者的PPT中拿过来的。估计要涉及编译器相关的知识，我无能为力了，但是下面做实验来猜吧。当然了，PPT的内容也需要验证才具有说服力。尽信书则不如无书。我特地写三个全局变量来验证。发现System.map分配地址之间的差值正好是0x40。因此这里的确是填充60 bytes。  
另外从我的测试发现，如果上述的数组a的大小是33的时候，填充的redzone就是63 bytes。所以我推测，填充的原理是这样的。全局变量实际占用内存总数S（以byte为单位）按照每块32 bytes平均分成N块。假设最后一块内存距离目标32 bytes还差y bytes（if S%32 == 0，y = 0），那么redzone填充的大小就是(y + 32) bytes。画图示意如下（S%32 != 0）。因此总结的规律是：redzone = 63 – (S - 1) % 32。  
 [![11.png](http://www.wowotech.net/content/uploadfile/201802/59b21519483509.png "点击查看原图")](http://www.wowotech.net/content/uploadfile/201802/59b21519483509.png)  
全局变量redzone区域对应的shadow memory是在什么填充的呢？又是如何调用的呢？这部分是由编译器帮我们完成的。编译器会为每一个全局变量创建一个函数，函数名称是：_GLOBAL__sub_I_65535_1_##global_variable_name。这个函数中通过调用__asan_register_globals()函数完成shadow memory标记。并且将自动生成的这个函数的首地址放在.init_array段。在kernel启动阶段，通过以下代调用关系最终调用所有全局变量的构造函数。kernel_init_freeable()->do_basic_setup() ->do_ctors()。do_ctors()代码实现如下：  

1. static void __init do_ctors(void)
2. {
3.     ctor_fn_t *fn = (ctor_fn_t *) __ctors_start;
4.     for (; fn < (ctor_fn_t *) __ctors_end; fn++)
5.         (*fn)();
6. }

  

这里的代码意思对于轻车熟路的你再熟悉不过了吧。因为内核中这么搞的太多了。便利__ctors_start和__ctors_end之间的所有数据，作为函数地址进行调用，即完成了所有的global variables的shadow memory初始化。我们可以从链接脚本中知道__ctors_start和__ctors_end的意思。  
#define KERNEL_CTORS()  . = ALIGN(8);              \  
            VMLINUX_SYMBOL(__ctors_start) = .; \  
            KEEP(*(.ctors))            \  
            KEEP(*(SORT(.init_array.*)))       \  
            KEEP(*(.init_array))           \  
            VMLINUX_SYMBOL(__ctors_end) = .;  
上面说了这么多，不知道你是否产生了疑心？怎么都是猜啊！猜的能准确吗？是的，我也这么觉得。是骡子是马，拉出来溜溜呗！现在用事实说话。首先我创建一个c文件drivers/input/smc.c。在smc.c文件中创建3个全局变量如下：  
 [![12.png](http://www.wowotech.net/content/uploadfile/201802/9eb61518360152.png "点击查看原图")](http://www.wowotech.net/content/uploadfile/201802/9eb61518360152.png)  
然后就随便使用吧！编译kernel，我们先看看System.map文件中，3个全局变量分配的地址。  
ffff200009f540e0 B smc_num1  
ffff200009f54120 B smc_num2  
ffff200009f54160 B smc_num3  
还记得上面说会有一个形如_GLOBAL__sub_I_65535_1_##global_variable_name的函数吗？在System.map文件文件中，我看到了_GLOBAL__sub_I_65535_1_smc_num1符号。但是没有smc_num2和smc_num3的构造函数。你是不是很奇怪，不是每一个全局变量都会创建一个类似的构造函数吗？马上为你揭晓。我们先执行aarch64-linux-gnu-objdump –s –x –d vmlinux > vmlinux.txt命令得到反编译文件。现在好多重要的信息在vmlinux.txt。现在主要就是查看vmlinux.txt文件。先看一下_GLOBAL__sub_I_65535_1_smc_num1函数的实现。  
ffff200009381df0 <_GLOBAL__sub_I_65535_1_smc_num1>:  
ffff200009381df0:   a9bf7bfd    stp x29, x30, [sp,#-16]!  
ffff200009381df4:   b0001800    adrp    x0, ffff200009682000  
ffff200009381df8:   91308000    add x0, x0, #0xc20  
ffff200009381dfc:   d2800061    mov x1, #0x3                    // #3  
ffff200009381e00:   910003fd    mov x29, sp  
ffff200009381e04:   9100c000    add x0, x0, #0x30  
ffff200009381e08:   97c09fb8    bl  ffff2000083a9ce8 <__asan_register_globals>  
ffff200009381e0c:   a8c17bfd    ldp x29, x30, [sp],#16  
ffff200009381e10:   d65f03c0    ret  
汇编和C语言传递参数在ARM64平台使用的是x0~x7。通过上面的汇编计算一下，x0=0xffff200009682c50，x1=3。然后调用__asan_register_globals()函数，x0和x1就是传递的参数。我们看一下__asan_register_globals()函数实现。  

1. void __asan_register_globals(struct kasan_global *globals, size_t size)
2. {
3.     int i;
4.     for (i = 0; i < size; i++)
5.         register_global(&globals[i]);
6. }

  

size是3就是要初始化全局变量的个数，所以这里只需要一个构造函数即可。一次性将3个全局变量全部搞定。这里再说一点猜测吧！我猜测是以文件为单位编译器创建一个构造函数即可，将本文件全局变量一次性全部打包初始化。第一个参数globals是0xffff200009682c50，继续从vmlinux.txt中查看该地址处的数据。struct kasan_global是编译器帮我们自动创建的结构体，每一个全局变量对应一个struct kasan_global结构体。struct kasan_global结构体存放的位置是.data段，因此我们可以从.data段查找当前地址对应的数据。数据如下：  
ffff200009682c50 6041f509 0020ffff 07000000 00000000  
ffff200009682c60 40000000 00000000 d0d62b09 0020ffff  
ffff200009682c70 b8d62b09 0020ffff 00000000 00000000  
ffff200009682c80 202c6809 0020ffff 2041f509 0020ffff  
ffff200009682c90 1f000000 00000000 40000000 00000000  
ffff200009682ca0 e0d62b09 0020ffff b8d62b09 0020ffff  
ffff200009682cb0 00000000 00000000 302c6809 0020ffff  
ffff200009682cc0 e040f509 0020ffff 04000000 00000000  
ffff200009682cd0 40000000 00000000 f0d62b09 0020ffff  
ffff200009682ce0 b8d62b09 0020ffff 00000000 00000000  
首先ffff200009682c50对应的第一个数据6041f509 0020ffff，这是个啥？其实是一个地址数据，你是不是又疑问了，ARM64的kernel space地址不是ffff开头吗？这个怎么60开头？其实这个地址数据是反过来的，你应该从右向左看。这个地址其实是ffff200009f54160。这不正是smc_num3的地址嘛！解析这段数据之前需要了解一下struct kasan_global结构体。  

1. /* The layout of struct dictated by compiler */
2. struct kasan_global {
3.     const void *beg;        /* Address of the beginning of the global variable. */
4.     size_t size;            /* Size of the global variable. */
5.     size_t size_with_redzone;   /* Size of the variable + size of the red zone. 32 bytes aligned */
6.     const void *name;
7.     const void *module_name;    /* Name of the module where the global variable is declared. */
8.     unsigned long has_dynamic_init; /* This needed for C++ */
9. #if KASAN_ABI_VERSION >= 4
10.     struct kasan_source_location *location;
11. #endif
12. };

  
第一个成员beg就是全局变量的首地址。跟上面的分析一致。第二个成员size从上面数据看出是7，正好对应我们定义的smc_num3[7]，正好7 bytes。size_with_redzone的值是0x40，正好是64。根据上面猜测redzone=63-(7-1)%32=57。加上size正好是64，说明之前猜测的redzone计算方法没错。name成员对应的地址是ffff2000092bd6d0。看下ffff2000092bd6d0存储的是什么。  
ffff2000092bd6d0 736d635f 6e756d33 00000000 00000000  smc_num3........  
所以name就是全局变量的名称转换成字符串。同样的方式得到module_name的地址是ffff2000092bd6b8。继续看看这段地址存储的数据。  
ffff2000092bd6b0 65000000 00000000 64726976 6572732f  e.......drivers/  
ffff2000092bd6c0 696e7075 742f736d 632e6300 00000000  input/smc.c.....  
一目了然，module_name是文件的路径。has_dynamic_init的值就是0，这是C++需要的。我用的GCC版本是5.0左右，所以这里的KASAN_ABI_VERSION=4。这里location成员的地址是ffff200009682c20，继续追踪该地址的数据。  
ffff200009682c20 b8d62b09 0020ffff 0e000000 0f000000  
解析这段数据之前要先了解struct kasan_source_location结构体。  

1. /* The layout of struct dictated by compiler */
2. struct kasan_source_location {
3.     const char *filename;
4.     int line_no;
5.     int column_no;
6. };

  
第一个成员filename地址是ffff2000092bd6b8和module_name一样的数据。剩下两个数据分别是14和15，分别代表全局变量定义地方的行号和列号。现在回到上面我定义变量的截图，仔细数数列号是不是15，行号截图中也有哦！特地截出来给你看的。剩下的struct kasan_global数据就是smc_num1和smc_num2的数据。分析就不说了。前面说_GLOBAL__sub_I_65535_1_smc_num1函数会被自动调用，该地址数据填充在__ctors_start和__ctors_end之间。现在也证明一下观点。先从System.map得到符号的地址数据。  
ffff2000093ac5d8 T __ctors_start  
ffff2000093ae860 T __ctors_end  
然后搜索一下_GLOBAL__sub_I_65535_1_smc_num1的地址ffff200009381df0被存储在什么位置，记得搜索的关键字是f01d3809 0020ffff。  
ffff2000093ae0c0 f01d3809 0020ffff 181e3809 0020ffff  
可以看出ffff2000093ae0c0地址处存储着_GLOBAL__sub_I_65535_1_smc_num1函数地址。这个地址不是正好位于__ctors_start和__ctors_end之间嘛！  
现在就剩下__asan_register_globals()函数到底是是怎么初始化shadow memory的呢？以char a[4]为例，如下图所示。  
 [![13.png](http://www.wowotech.net/content/uploadfile/201802/c00b1519483390.png "点击查看原图")](http://www.wowotech.net/content/uploadfile/201802/c00b1519483390.png)  
a[4]只有4 bytes可以访问，所以对应的shadow memory的第一个byte值是4，后面的redzone就填充0xFA作为越界检测。a[4]只有4 bytes可以访问，所以对应的shadow memory的第一个byte值是4，后面的redzone就填充0xFA作为越界检测。因为这里是全局变量，因此分配的内存区域位于kernel区域。

  
4.7. 栈分配变量的readzone是如何分配的？

  
从栈中分配的变量同样和全局变量一样需要填充一些内存作为redzone区域。下面继续举个例子说明编译器怎么填充。首先来一段正常的代码，没有编译器的插手。

1. void foo()
2. {
3.     char a[328];
4. }

  
再来看看编译器插了哪些东西进去。

void foo()  
{    char rz1[32];    char a[328];  
    char rz2[56];  
    int *shadow = （&rz1 >> 3）+ KASAN_SHADOW_OFFSE;  
    shadow[0] = 0xffffffff;  
    shadow[11] = 0xffffff00;  
    shadow[12] = 0xffffffff;  
------------------------使用完毕----------------------------------------  
    shadow[0] = shadow[11] = shadow[12] = 0;  
}

红色部分是编译器填充内存，rz2是56，可以根据上一节全局变量的公式套用计算得到。但是这里在变量前面竟然还有32 bytes的rz1。这个是和全局变量的不同，我猜测这里是为了检测栈变量左边界越界问题。蓝色部分代码也是编译器填充，初始化shadow memory。栈的填充就没有探究那么深入了，有兴趣的读者可以自己探究。

  
5. Error log信息包含哪些信息？

  
从kernel的Documentation文档找份典型的KASAN bug输出的log信息如下。  
==================================================================  
BUG: AddressSanitizer: out of bounds access in kmalloc_oob_right+0x65/0x75 [test_kasan] at addr ffff8800693bc5d3  
Write of size 1 by task modprobe/1689  
=============================================================================  
BUG kmalloc-128 (Not tainted): kasan error  
-----------------------------------------------------------------------------

Disabling lock debugging due to kernel taint  
INFO: Allocated in kmalloc_oob_right+0x3d/0x75 [test_kasan] age=0 cpu=0 pid=1689  
 __slab_alloc+0x4b4/0x4f0  
 kmem_cache_alloc_trace+0x10b/0x190  
 kmalloc_oob_right+0x3d/0x75 [test_kasan]  
 init_module+0x9/0x47 [test_kasan]  
 do_one_initcall+0x99/0x200  
 load_module+0x2cb3/0x3b20  
 SyS_finit_module+0x76/0x80  
 system_call_fastpath+0x12/0x17  
INFO: Slab 0xffffea0001a4ef00 objects=17 used=7 fp=0xffff8800693bd728 flags=0x100000000004080  
INFO: Object 0xffff8800693bc558 @offset=1368 fp=0xffff8800693bc720

Bytes b4 ffff8800693bc548: 00 00 00 00 00 00 00 00 5a 5a 5a 5a 5a 5a 5a 5a  ........ZZZZZZZZ  
Object ffff8800693bc558: 6b 6b 6b 6b 6b 6b 6b 6b 6b 6b 6b 6b 6b 6b 6b 6b  kkkkkkkkkkkkkkkk  
Object ffff8800693bc568: 6b 6b 6b 6b 6b 6b 6b 6b 6b 6b 6b 6b 6b 6b 6b 6b  kkkkkkkkkkkkkkkk  
Object ffff8800693bc578: 6b 6b 6b 6b 6b 6b 6b 6b 6b 6b 6b 6b 6b 6b 6b 6b  kkkkkkkkkkkkkkkk  
Object ffff8800693bc588: 6b 6b 6b 6b 6b 6b 6b 6b 6b 6b 6b 6b 6b 6b 6b 6b  kkkkkkkkkkkkkkkk  
Object ffff8800693bc598: 6b 6b 6b 6b 6b 6b 6b 6b 6b 6b 6b 6b 6b 6b 6b 6b  kkkkkkkkkkkkkkkk  
Object ffff8800693bc5a8: 6b 6b 6b 6b 6b 6b 6b 6b 6b 6b 6b 6b 6b 6b 6b 6b  kkkkkkkkkkkkkkkk  
Object ffff8800693bc5b8: 6b 6b 6b 6b 6b 6b 6b 6b 6b 6b 6b 6b 6b 6b 6b 6b  kkkkkkkkkkkkkkkk  
Object ffff8800693bc5c8: 6b 6b 6b 6b 6b 6b 6b 6b 6b 6b 6b 6b 6b 6b 6b a5  kkkkkkkkkkkkkkk.  
Redzone ffff8800693bc5d8: cc cc cc cc cc cc cc cc                          ........  
Padding ffff8800693bc718: 5a 5a 5a 5a 5a 5a 5a 5a                          ZZZZZZZZ  
CPU: 0 PID: 1689 Comm: modprobe Tainted: G    B          3.18.0-rc1-mm1+ #98  
Hardware name: QEMU Standard PC (i440FX + PIIX, 1996), BIOS rel-1.7.5-0-ge51488c-20140602_164612-nilsson.home.kraxel.org 04/01/2014  
 ffff8800693bc000 0000000000000000 ffff8800693bc558 ffff88006923bb78  
 ffffffff81cc68ae 00000000000000f3 ffff88006d407600 ffff88006923bba8  
 ffffffff811fd848 ffff88006d407600 ffffea0001a4ef00 ffff8800693bc558  
Call Trace:  
 [<ffffffff81cc68ae>] dump_stack+0x46/0x58  
 [<ffffffff811fd848>] print_trailer+0xf8/0x160  
 [<ffffffffa00026a7>] ? kmem_cache_oob+0xc3/0xc3 [test_kasan]  
 [<ffffffff811ff0f5>] object_err+0x35/0x40  
 [<ffffffffa0002065>] ? kmalloc_oob_right+0x65/0x75 [test_kasan]  
 [<ffffffff8120b9fa>] kasan_report_error+0x38a/0x3f0  
 [<ffffffff8120a79f>] ? kasan_poison_shadow+0x2f/0x40  
 [<ffffffff8120b344>] ? kasan_unpoison_shadow+0x14/0x40  
 [<ffffffff8120a79f>] ? kasan_poison_shadow+0x2f/0x40  
 [<ffffffffa00026a7>] ? kmem_cache_oob+0xc3/0xc3 [test_kasan]  
 [<ffffffff8120a995>] __asan_store1+0x75/0xb0  
 [<ffffffffa0002601>] ? kmem_cache_oob+0x1d/0xc3 [test_kasan]  
 [<ffffffffa0002065>] ? kmalloc_oob_right+0x65/0x75 [test_kasan]  
 [<ffffffffa0002065>] kmalloc_oob_right+0x65/0x75 [test_kasan]  
 [<ffffffffa00026b0>] init_module+0x9/0x47 [test_kasan]  
 [<ffffffff810002d9>] do_one_initcall+0x99/0x200  
 [<ffffffff811e4e5c>] ? __vunmap+0xec/0x160  
 [<ffffffff81114f63>] load_module+0x2cb3/0x3b20  
 [<ffffffff8110fd70>] ? m_show+0x240/0x240  
 [<ffffffff81115f06>] SyS_finit_module+0x76/0x80  
 [<ffffffff81cd3129>] system_call_fastpath+0x12/0x17  
Memory state around the buggy address:  
 ffff8800693bc300: fc fc fc fc fc fc fc fc fc fc fc fc fc fc fc fc  
 ffff8800693bc380: fc fc 00 00 00 00 00 00 00 00 00 00 00 00 00 fc  
 ffff8800693bc400: fc fc fc fc fc fc fc fc fc fc fc fc fc fc fc fc  
 ffff8800693bc480: fc fc fc fc fc fc fc fc fc fc fc fc fc fc fc fc  
 ffff8800693bc500: fc fc fc fc fc fc fc fc fc fc fc 00 00 00 00 00  
>ffff8800693bc580: 00 00 00 00 00 00 00 00 00 00 03 fc fc fc fc fc  
                                                                           ^  
 ffff8800693bc600: fc fc fc fc fc fc fc fc fc fc fc fc fc fc fc fc  
 ffff8800693bc680: fc fc fc fc fc fc fc fc fc fc fc fc fc fc fc fc  
 ffff8800693bc700: fc fc fc fc fb fb fb fb fb fb fb fb fb fb fb fb  
 ffff8800693bc780: fb fb fb fb fb fb fb fb fb fb fb fb fb fb fb fb  
 ffff8800693bc800: fb fb fb fb fb fb fb fb fb fb fb fb fb fb fb fb  
==================================================================  
输出的信息很丰富，包含了bug发生的类型、SLUB输出的object内存信息、Call Trace以及shadow memory的状态值。其中红色信息都是比较重要的信息。我没有写demo历程，而是找了一份log信息，不是我想偷懒，而是锻炼自己。怎么锻炼呢？我想问的是，从这份log中你可以推测代码应该是怎么样的？我可以得到一下信息：  
1) 程序是通过kmalloc接口申请内存的；  
2) 申请的内存大小是123 bytes，即p = kamlloc(123);  
3) 代码中类似往p[123]中写1 bytes导致越界访问的bug；  
4) 在3)步骤发生前没有任何的对该内存的写操作；  
如果你也能得到以上4点猜测，我觉的我写的这几篇文章你是真的看明白了。首先输出信息是有SLUB的信息，所以应该是通过kmalloc()接口；在打印的shadow memory的值中，我们看到连续的15个0和一个3，所以申请的内存size就是15x8+3=123；由于是往ffff8800693bc5d3地址写1个字节，并且object首地址是ffff8800693bc558，所以推测是往p[123]写1 byte出问题；由于log中将object中所有的128 bytes数据全部打印出来，一共是127个0x6b和一个0xa5（SLUB DEBUG文章介绍的内容）。所以我推测在3)步骤发生前没有任何的对该内存的写操作。 

  

6. 补充

我看了linux-4.18的代码，KASAN的log输出已经发生了部分变化。例如：上面举例的SLUB的object的内容就不会打印了。我们用一下的程序展示这些变化（实际上就是上面举例用的程序）。

1. static noinline void __init kmalloc_oob_right(void)
2. {
3. 	char *ptr;
4. 	size_t size = 123;

6. 	ptr = kmalloc(size, GFP_KERNEL);
7. 	if (!ptr) {
8. 		pr_err("Allocation failed\n");
9. 		return;
10. 	}

12. 	ptr[size] = 'x';
13. 	kfree(ptr);
14. }

针对以上代码，KASAN检测到bug后的输出log如下：

1. ==================================================================
2. BUG: KASAN: slab-out-of-bounds in kmalloc_oob_right+0x6c/0x8c
3. Write of size 1 at addr ffffffc0cb114d7b by task swapper/0/1

5. CPU: 4 PID: 1 Comm: swapper/0 Tainted: G S      W       4.9.82-perf+ #310
6. Hardware name: Qualcomm Technologies, Inc. SDM632 PMI632
7. Call trace:
8. [<ffffff90cf88d9f8>] dump_backtrace+0x0/0x320
9. [<ffffff90cf88dd2c>] show_stack+0x14/0x20
10. [<ffffff90cfdd1148>] dump_stack+0xa8/0xd0
11. [<ffffff90cfabf298>] print_address_description+0x60/0x250
12. [<ffffff90cfabf6a0>] kasan_report.part.2+0x218/0x2f0
13. [<ffffff90cfabfac0>] kasan_report+0x20/0x28
14. [<ffffff90cfabdc64>] __asan_store1+0x4c/0x58
15. [<ffffff90d1a4f760>] kmalloc_oob_right+0x6c/0x8c
16. [<ffffff90d1a50448>] kmalloc_tests_init+0xc/0x68
17. [<ffffff90cf8845dc>] do_one_initcall+0xa4/0x1f0
18. [<ffffff90d1a011ac>] kernel_init_freeable+0x244/0x300
19. [<ffffff90d0d6da70>] kernel_init+0x10/0x110
20. [<ffffff90cf8842a0>] ret_from_fork+0x10/0x30

22. Allocated by task 1:
23.  kasan_kmalloc+0xd8/0x188
24.  kmem_cache_alloc_trace+0x130/0x248
25.  kmalloc_oob_right+0x4c/0x8c
26.  kmalloc_tests_init+0xc/0x68
27.  do_one_initcall+0xa4/0x1f0
28.  kernel_init_freeable+0x244/0x300
29.  kernel_init+0x10/0x110
30.  ret_from_fork+0x10/0x30

32. Freed by task 1:
33.  kasan_slab_free+0x88/0x178
34.  kfree+0x84/0x298
35.  kobject_uevent_env+0x144/0x620
36.  kobject_uevent+0x10/0x18
37.  device_add+0x5f8/0x860
38.  amba_device_try_add+0x22c/0x2f8
39.  amba_device_add+0x20/0x128
40.  of_platform_bus_create+0x390/0x478
41.  of_platform_bus_create+0x21c/0x478
42.  of_platform_populate+0x4c/0xb8
43.  of_platform_default_populate_init+0x78/0x8c
44.  do_one_initcall+0xa4/0x1f0
45.  kernel_init_freeable+0x244/0x300
46.  kernel_init+0x10/0x110
47.  ret_from_fork+0x10/0x30

49. The buggy address belongs to the object at ffffffc0cb114d00
50.  which belongs to the cache kmalloc-128 of size 128
51. The buggy address is located 123 bytes inside of
52.  128-byte region [ffffffc0cb114d00, ffffffc0cb114d80)
53. The buggy address belongs to the page:
54. page:ffffffbf032c4500 count:1 mapcount:0 mapping: (null) index:0xffffffc0cb115200 compound_mapcount: 0
55. flags: 0x4080(slab|head)
56. page dumped because: kasan: bad access detected

58. Memory state around the buggy address:
59.  ffffffc0cb114c00: fc fc fc fc fc fc fc fc fc fc fc fc fc fc fc fc
60.  ffffffc0cb114c80: fc fc fc fc fc fc fc fc fc fc fc fc fc fc fc fc
61. >ffffffc0cb114d00: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 03
62.                                                                 ^
63.  ffffffc0cb114d80: fc fc fc fc fc fc fc fc fc fc fc fc fc fc fc fc
64.  ffffffc0cb114e00: fc fc fc fc fc fc fc fc fc fc fc fc fc fc fc fc
65. ==================================================================

我们从上面的log可以分析如下数据：

- line2：发生越界访问位置。  
    
- line3：越界写1个字节，写的地址是0xffffffc0cb114d7b。当前进程是comm是swapper/0，pid是1。  
    
- line7：Call trace，方便定位出问题的函数调用关系。  
    
- line22：该object分配的调用栈，并指出分配内存的进程pid是1。  
    
- line32：释放该object的调用栈（上次释放），并指出释放内存的进程pid是1。  
    
- line49：指出slub相关的信息，从“kmalloc-28”的kmem_cache分配的object。object起始地址是0xffffffc0cb114d00。  
    
- line51：访问出问题的地址位于object起始地址偏移123 bytes的位置。object的地址范围是[0xffffffc0cb114d00, 0xffffffc0cb114d80)。object实际大小是128 bytes。  
    
- line61：出问题地址对应的shadow memory的值，可以确定申请内存的实际大小是123 bytes。

  

  

参考文献：  
1．How to use KASAN to debug memory corruption in OpenStack environment.pdf

2．KernelAddressSanitizer (KASan) a fast memory error detector for the Linux kernel.pdf

  

  

_原创文章，转发请注明出处。蜗窝科技，www.wowotech.net。_  

标签: [KASAN原理](http://www.wowotech.net/tag/KASAN%E5%8E%9F%E7%90%86)

[![](http://www.wowotech.net/content/uploadfile/201605/ef3e1463542768.png)](http://www.wowotech.net/support_us.html)

« [Deadline调度器之（二）：细节和使用方法](http://www.wowotech.net/process_management/dl-scheduler-2.html) | [deadline调度器之（一）：原理](http://www.wowotech.net/process_management/deadline-scheduler-1.html)»

**评论：**

**pete**  
2020-07-10 14:00

"KASAN不能检测VMALLOC分配的内存越界等问题"，那么vmalloc的问题有什么办法看吗？

[回复](http://www.wowotech.net/memory_management/424.html#comment-8051)

**elliot**  
2024-05-02 13:57

@pete：KASAN目前已经支持vmalloc的检测了

[回复](http://www.wowotech.net/memory_management/424.html#comment-8894)

**keith**  
2019-06-26 15:42

struct {  
    char original[4];  
    char redzone[60];  
} a; //32 bytes aligned  
  
请教一下大神，请问要通过怎样的方法，才能看到编译器的填充结构体呢？谢谢，望解答。

[回复](http://www.wowotech.net/memory_management/424.html#comment-7489)

**elliot**  
2024-05-02 13:58

@keith：有一个工具叫做pahole，可以解决你的问题

[回复](http://www.wowotech.net/memory_management/424.html#comment-8895)

**[smcdef](http://www.wowotech.net/)**  
2018-12-22 16:51

在这里补充一下：为什么KASAN不支持vmalloc分配的内存呢？  
  
答案在文章的“4.3. 如何建立shadow memory的映射关系？”节已经说了。当然说的不够明确。在这一节中可以看到整个KASAN shadow memory的映射关系。VMALLOC区域对应的shadow memory映射到kasan_zero_page物理页面。kasan_zero_page是一个0页。整个页的内容都是0。回想一下，0在shadow memory中意味着什么。就是代表所有的VMALLOC区域的内存访问都是合法的。所以间接的也表明，KASAN不能检测VMALLOC分配的内存越界等问题。

[回复](http://www.wowotech.net/memory_management/424.html#comment-7103)

**kongsuozt**  
2022-12-01 09:52

@smcdef：vmalloc也可以做，不过得自己去实现shadow memory的映射、状态更新以及分配释放的hook插入

[回复](http://www.wowotech.net/memory_management/424.html#comment-8711)

**xx**  
2018-10-29 11:25

static __always_inline bool memory_is_poisoned_1(unsigned long addr)  
{  
        s8 shadow_value = *(s8 *)kasan_mem_to_shadow((void *)addr);  
  
        if (unlikely(shadow_value)) {  
                s8 last_accessible_byte = addr & KASAN_SHADOW_MASK;  
                return unlikely(last_accessible_byte >= shadow_value);  
        }  
  
        return false;  
}  
  
从这个代码看，你的分析貌似不对哦。这个并不是 1 个 bit 对应一个 byte

[回复](http://www.wowotech.net/memory_management/424.html#comment-7004)

**[smcdef](http://www.wowotech.net/)**  
2018-11-21 11:30

@xx：我的描述如下：  
连续8 bytes内存（8 bytes align）使用1 byte shadow memory标记。如果8 bytes内存都可以访问，则shadow memory的值为0；如果连续N(1 =< N <= 7) bytes可以访问，则shadow memory的值为N；如果8 bytes内存访问都是invalid，则shadow memory的值为负数。  
  
关注的是shadow memory的值，并没有说1 个 bit 对应一个 byte。  
  
请问你指出的不对的地方是这个吗？如果我理解有问题，还请麻烦详细指出“分析貌似不对”的地方，以免影响后面的读者。谢谢

[回复](http://www.wowotech.net/memory_management/424.html#comment-7045)

**nzg**  
2018-10-22 17:03

您好，请问一下：  
我的环境kenrel 版本是4.4.83, 虚拟地址39bits， 4kb page size.  
KASAN 区间应该是在kernel 起始的64GB。  
kernel image (只看一下代码段)  
_stext =ffffff8008081000  
_etext = ffffff8008ac7000 ， kernel 起始地址 VA_START=0xffffff8000000000  
  
这样 Kasan区域是不是就把kernel 覆盖了阿？ 会有问题吗

[回复](http://www.wowotech.net/memory_management/424.html#comment-6998)

**nzg**  
2018-10-22 17:07

@nzg：找到答案了，code中 kimage也会顺着偏移^^

[回复](http://www.wowotech.net/memory_management/424.html#comment-6999)

**berton**  
2018-09-04 10:12

有个疑问，为什么shadow memroy要用1byte来标记对应的原本内存的8字节，3bit就可以表示8字节前多少个字节被使用，考虑到方便对齐等问题，4bit就可以实现对应的操作。就是1/2byte的shadow memory对应原本内存的8byte。不是更节省内存吗？  
但是，在KASAN这么设计的原因是啥？

[回复](http://www.wowotech.net/memory_management/424.html#comment-6926)

**berton**  
2018-09-04 10:15

@berton：4bit不光是对齐问题，还有一个都不可以使用的问题，但是同样是4bit就可以了  
弱弱的问一下这个问题

[回复](http://www.wowotech.net/memory_management/424.html#comment-6927)

**smc**  
2018-09-06 09:54

@berton：我猜测是是为了方便计算，如果采用1byte，计算一个地址对应的shadow memory地址可以直接用公式shadow_addr = (kaddr >> 3)  + KASAN_SHADOW_OFFSE得到。然后就可以直接填充对应的数值。如果使用4bit表示的话，计算地址和填充数据变得复杂。

[回复](http://www.wowotech.net/memory_management/424.html#comment-6930)

**berton**  
2018-09-13 20:22

@smc：采用4bit操作的话，其实也不会带来太多的计算上的麻烦，使用位操作，对1byte的前4bit和后4bit位操作，也不会太麻烦，位操作也不会带来很高的性能损耗。  
预留其他的标志位？4bit的话同样可以预留7种情况，还可以节省一半的虚拟地址，对于32位系统下尤为珍贵，单纯为了计算方便损耗1/16的虚地址，感觉有点浪费。。。。。

[回复](http://www.wowotech.net/memory_management/424.html#comment-6948)

**fjling**  
2018-07-03 14:50

这篇文章很赞。  
  
请教一个slub相关的问题。  
比如，alloc/free track这块区域有数据写入时，这里可能是slub代码本身的操作，也可能是kernel driver误操作。  
编译器不可能对这块区域做特殊处理吧。那么后者会报错吗？能说说这里的细节吗？  
  
谢谢！

[回复](http://www.wowotech.net/memory_management/424.html#comment-6834)

**fjling**  
2018-07-03 15:06

@fjling：刚才看了下代码，似乎自己搞明白了。自问自答一下，如果我理解有错，请指正。  
  
在kernel/mm等一些内核目录，属于可信赖的代码模块。  
在这些目录下的Makefile里，会加入KASAN_SANITIZE :=n，告诉编译器不需要做内存访问的检查。

[回复](http://www.wowotech.net/memory_management/424.html#comment-6835)

**[smcdef](http://www.wowotech.net/)**  
2018-12-16 00:11

@fjling：除了一些信任模块，可以在这些目录下的Makefile里，会加入KASAN_SANITIZE :=n以外。  
针对你说的问题：alloc/free track这块区域数据的写入，会在写入前metadata_access_enable();访问结束后调用metadata_access_disable();  
metadata_access_enable()其实就是关闭kasan的检测bug report机制（即使检测不能访问，也不报错）。  
  
可以参考代码：https://elixir.bootlin.com/linux/v4.15.7/source/mm/slub.c#L503

[回复](http://www.wowotech.net/memory_management/424.html#comment-7086)

**hzm**  
2018-06-15 11:27

最后一个例子 kernel的地址是 ffff8800693bc5d3  
公式：（shadow_addr = (kaddr >> 3)  + KASAN_SHADOW_OFFSE）  
怎么计算shadow memory出来的地址跟例子不符合？

[回复](http://www.wowotech.net/memory_management/424.html#comment-6804)

**[smcdef](http://www.wowotech.net/)**  
2018-06-15 21:14

@hzm：举的例子中，我并没有说KASAN_SHADOW_OFFSE等于多少啊！你怎么知道不符合呢？

[回复](http://www.wowotech.net/memory_management/424.html#comment-6808)

**hzm**  
2018-06-19 10:31

@smcdef：哦，那例子中的ffff8800693bc580这个地址，应该要先知道KASAN_SHADOW_OFFSE这个才能推出来的吧？我还以为文章说的和例子说的offset是一样的

[回复](http://www.wowotech.net/memory_management/424.html#comment-6810)

**smcdef_tmp**  
2018-06-19 10:36

@hzm：其实是不一样的。举例中其实是随便找的一份log，这个KASAN_SHADOW_OFFSE是不知道的。其实我们也没必要知道。当然了根据这份log可以计算出KASAN_SHADOW_OFFSE，但是好像也没什么用

[回复](http://www.wowotech.net/memory_management/424.html#comment-6811)

**hzm**  
2018-06-19 16:43

@smcdef_tmp：这样的话怎么知道哪个是出问题地址 相对应的 shadow memory呀？ 看不明白，如果不知道shadown memory的话，下面的结论推不出来  
  
2) 申请的内存大小是123 bytes，即p = kamlloc(123);  
  
劳烦LZ解答一下，谢谢

[回复](http://www.wowotech.net/memory_management/424.html#comment-6812)

**[smcdef](http://www.wowotech.net/)**  
2018-06-20 09:32

@hzm：仔细看下下面的log，还有红色数字，其实log中已经将shadow memory的内容打印出来了。不需要自己计算和推倒。出问题的地址也是log打印出来的，最后一个事例就是一份完整的log，这所有的信息都是已知的。

[回复](http://www.wowotech.net/memory_management/424.html#comment-6813)

**phybio**  
2018-06-13 10:29

请问KASAN对vmalloc分配的内存，出现的use-after-free和out-of-bounds，能否检测出来？  
还是只是对kmalloc分配的slub可以检测？

[回复](http://www.wowotech.net/memory_management/424.html#comment-6799)

**[smcdef](http://www.wowotech.net/)**  
2018-06-13 23:06

@phybio：不能检测，因为KASAN初始化的时候没有为vmalloc区域分配shadow memory，当然检测功能还有全局变量等，更详细的检测可以参考kasan提供的测试代码，见lib/test_kasan.c

[回复](http://www.wowotech.net/memory_management/424.html#comment-6801)

**zecard**  
2018-10-19 08:22

@smcdef：伙伴算法没有做redzone  怎么防止OOB？ 比如两块相邻的内存块都被申请过 影子内存都是0 .  即使一块越界到另一边也没办法检测！

[回复](http://www.wowotech.net/memory_management/424.html#comment-6995)

**ecjtusbs**  
2023-04-19 11:12

@zecard：“两块相邻的内存块都被申请过 影子内存都是0” 这种情况的越界检测是否有效，不知道答主有没有明确的结论，我的感觉应该无法区分。

[回复](http://www.wowotech.net/memory_management/424.html#comment-8774)

**卷心菜**  
2024-04-25 14:52

@zecard：这种情况kasan应该检测不出来

[回复](http://www.wowotech.net/memory_management/424.html#comment-8890)

**rafal**  
2018-04-02 14:29

最后一个例子有画龙点睛的作用

[回复](http://www.wowotech.net/memory_management/424.html#comment-6637)

1 [2](http://www.wowotech.net/memory_management/424.html/comment-page-2#comments)

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
    
    - ja  
        [@dream：我看完這段也有相同的想法，引用 @dream ...](http://www.wowotech.net/kernel_synchronization/spinlock.html#8922)
    - 元神高手  
        [围观首席power managerment专家](http://www.wowotech.net/pm_subsystem/device_driver_pm.html#8921)
    - 十七  
        [内核空间的映射在系统启动时就已经设定好，并且在所有进程的页表...](http://www.wowotech.net/process_management/context-switch-arch.html#8920)
    - lw  
        [sparse模型和disconti模型没看出来有什么本质区别...](http://www.wowotech.net/memory_management/memory_model.html#8919)
    - 肥饶  
        [一个没设置好就出错](http://www.wowotech.net/linux_kenrel/516.html#8918)
    - orange  
        [点赞点赞，对linuxer的文章总结到位](http://www.wowotech.net/device_model/dt-code-file-struct-parse.html#8917)
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
    
    - [玩转BLE(2)_使用bluepy扫描BLE的广播数据](http://www.wowotech.net/bluetooth/bluepy_scan.html)
    - [蜗窝微信群问题整理](http://www.wowotech.net/tech_discuss/question_set_1.html)
    - [CFS任务的负载均衡（任务放置）](http://www.wowotech.net/process_management/task_placement.html)
    - [新技能get: 订阅Linux内核邮件列表](http://www.wowotech.net/linux_application/lkml.html)
    - [Linux电源管理(7)_Wakeup events framework](http://www.wowotech.net/pm_subsystem/wakeup_events_framework.html)
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