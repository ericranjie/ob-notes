作者：[linuxer](http://www.wowotech.net/author/3 "linuxer") 发布于：2016-11-18 18:25 分类：[内存管理](http://www.wowotech.net/sort/memory_management)
# 一、前言

同样的，本文是[内存初始化](http://www.wowotech.net/memory_management/mm-init-1.html)文章的一份补充文档，希望能够通过这样的一份文档，细致的展示在初始化阶段，Linux 4.4.6内核如何从device tree中提取信息，完成内存布局的任务。具体的cpu体系结构选择的是ARM64。
# 二、memory type region的构建

memory type是一个memblock模块（内核初始化阶段的内存管理模块）的术语，memblock将内存块分成两种类型：一种是memory type，另外一种是reserved type，分别用数组来管理系统中的两种类型的memory region。本小节描述的是系统如何在初始化阶段构建memory type的数组。
## 1、扫描device tree

在完成fdt内存区域的地址映射之后（fixmap_remap_fdt），内核会对fdt进行扫描，以便完成memory type数组的构建。具体代码位于setup_machine_fdt--->early_init_dt_scan--->early_init_dt_scan_nodes中：
```cpp
void init early_init_dt_scan_nodes(void) {  
of_scan_flat_dt(early_init_dt_scan_chosen, boot_command_line); // －－－－－－（1）  
of_scan_flat_dt(early_init_dt_scan_root, NULL);   
of_scan_flat_dt(early_init_dt_scan_memory, NULL); // －－－－－－－－－－－－－（2）  
}
```
（1）of_scan_flat_dt函数是用来scan整个device tree，针对每一个node调用callback函数，因此，这里实际上是针对设备树中的每一个节点调用early_init_dt_scan_chosen函数。之所以这么做是因为device tree blob刚刚完成地址映射，还没有展开，我们只能使用这种比较笨的办法。这句代码主要是寻址chosen node，并解析，将相关数据放入到boot_command_line。

（2）概念同上，不过是针对memory node进行scan。
## 2、传统的命令行参数解析
```cpp
int init early_init_dt_scan_chosen(unsigned long node, const char uname, int depth, void *data) {  
int l;  
const char *p;

if (depth != 1 || !data ||  
(strcmp(uname, "chosen") != 0 && strcmp(uname, "chosen@0") != 0))  
return 0; －－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－（1）

early_init_dt_check_for_initrd(node); －－－－－－－－－－－－－－－－－－－－（2）

/ Retrieve command line /  
p = of_get_flat_dt_prop(node, "bootargs", &l);  
if (p != NULL && l > 0)  
strlcpy(data, p, min((int)l, COMMAND_LINE_SIZE)); －－－－－－－－－－－－（3）


ifdef CONFIG_CMDLINE  
ifndef CONFIG_CMDLINE_FORCE  
if (!((char *)data)0)  
endif  
strlcpy(data, CONFIG_CMDLINE, COMMAND_LINE_SIZE);  
endif / CONFIG_CMDLINE / －－－－－－－－－－－－－－－－－－－－－－－（4）


return 1;  
}
```
（1）上面我们说过，early_init_dt_scan_chosen会为device tree中的每一个node而调用一次，因此，为了效率，不是chosen node的节点我们必须赶紧闪人。由于chosen node是root node的子节点，因此其depth必须是1。这里depth不是1的节点，节点名字不是"chosen"或者[chosen@0](mailto:chosen@0)和我们毫无关系，立刻返回。

（2）解析chosen node中的initrd的信息

（3）解析chosen node中的bootargs（命令行参数）并将其copy到boot_command_line。

（4）一般而言，内核有可能会定义一个default command line string（CONFIG_CMDLINE），如果bootloader没有通过device tree传递命令行参数过来，那么可以考虑使用default参数。如果系统定义了CONFIG_CMDLINE_FORCE，那么系统强制使用缺省命令行参数，bootloader传递过来的是无效的。
## 3、memory node解析
```cpp
int init early_init_dt_scan_memory(unsigned long node, const char uname, int depth, void *data)  
{  
const char *type = of_get_flat_dt_prop(node, "device_type", NULL);  
const __be32 *reg, *endp;  
int l;   
if (type == NULL) {  
if (!IS_ENABLED(CONFIG_PPC32) || depth != 1 || strcmp(uname, "memory@0") != 0)  
return 0;  
} else if (strcmp(type, "memory") != 0)  
return 0; －－－－－－－－－－－－－－－－－－－－－－－－－－－－－（1）

reg = of_get_flat_dt_prop(node, "linux,usable-memory", &l);  
if (reg == NULL)  
reg = of_get_flat_dt_prop(node, "reg", &l);－－－－－－－－－－－－－－－（2）  
if (reg == NULL)  
return 0;

endp = reg + (l / sizeof(__be32)); －－－－－－－－－－－－－－－－－－－－（3）

while ((endp - reg) >= (dt_root_addr_cells + dt_root_size_cells)) {  
u64 base, size;

base = dt_mem_next_cell(dt_root_addr_cells, ®);  
size = dt_mem_next_cell(dt_root_size_cells, ®); －－－－－－－－－－（4）

early_init_dt_add_memory_arch(base, size);－－－－－－－－－－－－－－（5）  
}

return 0;  
}
```
（1）如果该memory node是root node的子节点的话，那么它一定是有device_type属性并且其值是字符串”memory”。不是的话就可以返回了。不过node没有定义device_type属性怎么办？大部分的平台都可以直接返回了，除了PPC32，对于这个平台，如果memory node是更深层次的节点的话，那么它是没有device_type属性的，这时候可以根据node name来判断。当然，目标都是一致的，不是自己关注的node就赶紧闪人。

（2）该memory node的物理地址信息保存在"linux,usable-memory"或者"reg"属性中（reg是我们常用的）

（3）l / sizeof(__be32)是reg属性值的cell数目，reg指向第一个cell，endp指向最后一个cell。

（4）memory node的reg属性值其实就是一个数组，数组中的每一个entry都是base address和size的二元组。解析reg属性需要两个参数，dt_root_addr_cells和dt_root_size_cells，这两个参数分别定义了root节点的子节点（比如说memory node）reg属性中base address和size的cell数目，如果等于1，基地址（或者size）用一个32-bit的cell表示。对于ARMv8，一般dt_root_addr_cells和dt_root_size_cells等于2，表示基地址（或者size）用两个32-bit的cell表示。

注：dt_root_addr_cells和dt_root_size_cells这两个参数的解析在early_init_dt_scan_root中完成。

（5）针对该memory mode中的每一个memory region，调用early_init_dt_add_memory_arch向系统注册memory type的内存区域（实际上是通过memblock_add完成的）。
## 4、解析memory相关的early option

setup_arch--->parse_early_param函数中会对early options解析解析，这会导致下面代码的执行：
```cpp
static int init early_mem(char p)  
{

memory_limit = memparse(p, &p) & PAGE_MASK;

return 0;  
}  
early_param("mem", early_mem);
```
在过去，没有device tree的时代，mem这个命令行参数传递了memory bank的信息，内核根据这个信息来创建系统内存的初始布局。在ARM64中，由于强制使用device tree，因此mem这个启动参数失去了本来的意义，现在它只是定义了memory的上限（最大的系统内存地址），可以限制DTS传递过来的内存参数。
# 三、reserved type region的构建

保留内存的定义主要在fixmap_remap_fdt和arm64_memblock_init函数中进行，我们会按照代码顺序逐一进行各种各样reserved type的memory region的构建。

1、保留fdt占用的内存，代码如下：
```cpp
void init fixmap_remap_fdt(phys_addr_t dt_phys)  
{……

memblock_reserve(dt_phys, size);

……}
```
fixmap_remap_fdt主要是为fdt建立地址映射，在该函数的最后，顺便就调用memblock_reserve保留了该段内存。

2、保留内核和initrd占用的内容，代码如下：
```cpp
void init arm64_memblock_init(void)  
{  
memblock_enforce_memory_limit(memory_limit); －－－－－－－－－－－－－－－－（1）  
memblock_reserve(__pa(text), _end - _text);－－－－－－－－－－－－－－－－－－（2）  
ifdef CONFIG_BLK_DEV_INITRD  
if (initrd_start)  
memblock_reserve(__virt_to_phys(initrd_start), initrd_end - initrd_start);－－－－－－（3）  
endif  
……}
```
（1）我们前面解析了DTS的memory节点，已经向系统加入了不少的memory type的region，当然reserved memory block也会有一些，例如DTB对应的memory就是reserved。memory_limit可以对这些DTS的设定给出上限，memblock_enforce_memory_limit函数会根据这个上限，修改各个memory region的base和size，此外还将大于memory_limit的memory block（包括memory type和reserved type）从列表中删掉。

（2）reserve内核代码、数据区等（_text到_end那一段，具体的内容可以参考内核链接脚本）

（3）保留initital ramdisk image区域（从initrd_start到initrd_end区域）

3、通过early_init_fdt_scan_reserved_mem函数来分析dts中的节点，从而进行保留内存的动作，代码如下：

> void __init early_init_fdt_scan_reserved_mem(void)  
> {  
>     int n;  
>     u64 base, size;
> 
>     if (!initial_boot_params)－－－－－－－－－－－－－－－－－－－－－－－－（1）  
>         return;
> 
>     /* Process header /memreserve/ fields */  
>     for (n = 0; ; n++) {  
>         fdt_get_mem_rsv(initial_boot_params, n, &base, &size);－－－－－－－－（2）  
>         if (!size)  
>             break;  
>         early_init_dt_reserve_memory_arch(base, size, 0);－－－－－－－－－－－（3）  
>     }
> 
>     of_scan_flat_dt(__fdt_scan_reserved_mem, NULL);－－－－－－－－－－－－（4）  
>     fdt_init_reserved_mem();  
> }

（1）initial_boot_params实际上就是fdt对应的虚拟地址。在early_init_dt_verify中设定的。如果系统中都没有有效的fdt，那么没有什么可以scan的，return，走人。

（2）分析fdt中的 /memreserve/ fields ，进行内存的保留。在fdt的header中定义了一组memory reserve参数，其具体的位置是fdt base address + off_mem_rsvmap。off_mem_rsvmap是fdt header中的一个成员，如下：

> struct fdt_header {  
> ……  
>     fdt32_t off_mem_rsvmap;－－－－－－/memreserve/ fields offset  
> ……};

fdt header中的memreserve可以定义多个，每个都是（address，size）二元组，最后以0，0结束。

（3）保留每一个/memreserve/ fields定义的memory region，底层是通过memblock_reserve接口函数实现的。

（4）对fdt中的每一个节点调用__fdt_scan_reserved_mem函数，进行reserved-memory节点的扫描，之后调用fdt_init_reserved_mem函数进行内存预留的动作，具体参考下一小节描述。

4、解析reserved-memory节点的内存，代码如下：

> static int __init __fdt_scan_reserved_mem(unsigned long node, const char *uname,  
>                       int depth, void *data)  
> {  
>     static int found;  
>     const char *status;  
>     int err;
> 
>     if (!found && depth == 1 && strcmp(uname, "reserved-memory") == 0) { －－－－－－－（1）  
>         if (__reserved_mem_check_root(node) != 0) {  
>             pr_err("Reserved memory: unsupported node format, ignoring\n");   
>             return 1;  
>         }  
>         found = 1; －－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－（2）  
>         return 0;  
>     } else if (!found) {   
>         return 0; －－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－（3）  
>     } else if (found && depth < 2) { －－－－－－－－－－－－－－－－－－－－－－－－－（4）  
>         return 1;  
>     }
> 
>     status = of_get_flat_dt_prop(node, "status", NULL); －－－－－－－－－－－－－－－－（5）  
>     if (status && strcmp(status, "okay") != 0 && strcmp(status, "ok") != 0)  
>         return 0;
> 
>     err = __reserved_mem_reserve_reg(node, uname); －－－－－－－－－－－－－－－－（6）  
>     if (err == -ENOENT && of_get_flat_dt_prop(node, "size", NULL))  
>         fdt_reserved_mem_save_node(node, uname, 0, 0); －－－－－－－－－－－－－－－（7）
> 
>     /* scan next node */  
>     return 0;  
> }

（1）found 变量记录了是否搜索到一个reserved-memory节点，如果没有，我们的首要目标是找到一个reserved-memory节点。reserved-memory节点的特点包括：是root node的子节点（depth == 1），node name是"reserved-memory"，这可以过滤掉一大票无关节点，从而加快搜索速度。

（2）reserved-memory节点应该包括#address-cells、#size-cells和range属性，并且#address-cells和#size-cells的属性值应该等于根节点对应的属性值，如果检查通过（__reserved_mem_check_root），那么说明找到了一个正确的reserved-memory节点，可以去往下一个节点了。当然，下一个节点往往是reserved-memory节点的subnode，也就是真正的定义各段保留内存的节点。更详细的关于reserved-memory的设备树定义可以参考Documentation\devicetree\bindings\reserved-memory\reserved-memory.txt文件。

（3）没有找到reserved-memory节点之前，of_scan_flat_dt会不断的遍历下一个节点，而在__fdt_scan_reserved_mem函数中返回0表示让搜索继续，如果返回1，表示搜索停止。

（4）如果找到了一个reserved-memory节点，并且完成了对其所有subnode的scan，那么是退出整个reserved memory的scan过程了。

（5）如果定义了status属性，那么要求其值必须要是ok或者okay，当然，你也可以不定义该属性（这是一般的做法）。

（6）定义reserved memory有两种方法，一种是静态定义，也就是定义了reg属性，这时候，可以通过调用__reserved_mem_reserve_reg函数解析reg的（address，size）的二元数组，逐一对每一个定义的memory region进行预留。实际的预留内存动作可以调用memblock_reserve或者memblock_remove，具体调用哪一个是和该节点是否定义no-map属性相关，如果定义了no-map属性，那么说明这段内存操作系统根本不需要进行地址映射，也就是说这块内存是不归操作系统内存管理模块来管理的，而是归于具体的驱动使用（在device tree中，设备节点可以定义memory-region节点来引用在memory node中定义的保留内存，具体可以参考reserved-memory.txt文件）。

（7）另外一种定义reserved memory的方法是动态定义，也就是说定义了该内存区域的size（也可以定义alignment或者alloc-range进一步约定动态分配的reserved memory属性，不过这些属性都是option的），但是不指定具体的基地址，让操作系统自己来分配这段memory。

5、预留reserved-memory节点的内存

device tree中的reserved-memory节点及其子节点静态或者动态定义了若干的reserved memory region，静态定义的memory region起始地址和size都是确定的，因此可以立刻调用memblock的模块进行内存区域的预留，但是对于动态定义的memory region，__fdt_scan_reserved_mem只是将信息保存在了reserved_mem全局变量中，并没有进行实际的内存预留动作，具体的操作在fdt_init_reserved_mem函数中，代码如下：

> void __init fdt_init_reserved_mem(void)  
> {  
>     int i;
> 
>     __rmem_check_for_overlap(); －－－－－－－－－－－－－－－－－－－－－－－－－（1）
> 
>     for (i = 0; i < reserved_mem_count; i++) {－－遍历每一个reserved memory region  
>         struct reserved_mem *rmem = &reserved_mem[i];  
>         unsigned long node = rmem->fdt_node;  
>         int len;  
>         const __be32 *prop;  
>         int err = 0;
> 
>         prop = of_get_flat_dt_prop(node, "phandle", &len);－－－－－－－－－－－－－－－（2）  
>         if (!prop)  
>             prop = of_get_flat_dt_prop(node, "linux,phandle", &len);  
>         if (prop)  
>             rmem->phandle = of_read_number(prop, len/4);
> 
>         if (rmem->size == 0)－－－－－－－－－－－－－－－－－－－－－－－－－－－－（3）  
>             err = __reserved_mem_alloc_size(node, rmem->name,  
>                          &rmem->base, &rmem->size);  
>         if (err == 0)  
>             __reserved_mem_init_node(rmem);－－－－－－－－－－－－－－－－－－－－（4）  
>     }  
> }

（1）检查静态定义的 reserved memory region之间是否有重叠区域，如果有重叠，这里并不会对reserved memory region的base和size进行调整，只是打印出错信息而已。

（2）每一个需要被其他node引用的node都需要定义"phandle", 或者"linux,phandle"。虽然在实际的device tree source中看不到这个属性，实际上dtc会完美的处理这一切的。

（3）size等于0的memory region表示这是一个动态分配region，base address尚未定义，因此我们需要通过__reserved_mem_alloc_size函数对节点进行分析（size、alignment等属性），然后调用memblock的alloc接口函数进行memory block的分配，最终的结果是确定base address和size，并将这段memory region从memory type的数组中移到reserved type的数组中。当然，如果定义了no-map属性，那么这段memory会从系统中之间删除（memory type和reserved type数组中都没有这段memory的定义）。

（4）保留内存有两种使用场景，一种是被特定的驱动使用，这时候在特定驱动的初始化函数（probe函数）中自然会进行处理。还有一种场景就是被所有驱动或者内核模块使用，例如CMA，per-device Coherent DMA的分配等，这时候，我们需要借用device tree的匹配机制进行这段保留内存的初始化动作。有兴趣的话可以看看RESERVEDMEM_OF_DECLARE的定义，这里就不再描述了。

6、通过命令行参数保留CMA内存

arm64_memblock_init--->dma_contiguous_reserve函数中会根据命令行参数进行CMA内存的保留，本文暂不描述之，留给CMA文档吧。

四、总结

物理内存布局是归于memblock模块进行管理的，该模块定义了struct memblock memblock这样的一个全局变量保存了memory type和reserved type的memory region list。而通过这两个memory region的数组，我们就知道了操作系统需要管理的所有系统内存的布局情况。

_原创文章，转发请注明出处。蜗窝科技_

标签: [Memory](http://www.wowotech.net/tag/Memory) [内存布局](http://www.wowotech.net/tag/%E5%86%85%E5%AD%98%E5%B8%83%E5%B1%80) [layout](http://www.wowotech.net/tag/layout)

[![](http://www.wowotech.net/content/uploadfile/201605/ef3e1463542768.png)](http://www.wowotech.net/support_us.html)

« [X-018-KERNEL-串口驱动开发之serial console](http://www.wowotech.net/x_project/serial_driver_porting_3.html) | [X-017-KERNEL-串口驱动开发之uart driver框架](http://www.wowotech.net/x_project/serial_driver_porting_2.html)»

**评论：**

**wmzjzwlzs**  
2017-12-25 21:57

发现一个64位高通平台的一个现象，假设一个内核函数在system.map中的地址是A，在kernel中把这个函数地址打印出来是B，A和B理论上应该相等的吧，可是实际是每次重启B都会变化，但是A和B的低20位是相同的，为什么A和B不同

[回复](http://www.wowotech.net/memory_management/memory-layout.html#comment-6421)

**aaa**  
2017-12-26 21:59

@wmzjzwlzs：4.4内核吧，kaslr

[回复](http://www.wowotech.net/memory_management/memory-layout.html#comment-6423)

**wmzjzwlzs**  
2017-12-26 22:14

@aaa：是的，赞赞赞。  
你们好厉害，问了好多大神都不知道

[回复](http://www.wowotech.net/memory_management/memory-layout.html#comment-6424)

**zm**  
2017-05-12 15:12

int __init early_init_dt_scan_memory(unsigned long node, const char *uname, int depth, void *data)  
{  
    while ((endp - reg) >= (dt_root_addr_cells + dt_root_size_cells)) {  
        u64 base, size;  
        base = dt_mem_next_cell(dt_root_addr_cells, ®);  
        size = dt_mem_next_cell(dt_root_size_cells, ®); －－－－－－－－－－（4）  
        early_init_dt_add_memory_arch(base, size);－－－－－－－－－－－－－－（5）  
    }  
    return 0;  
}  
  
怎么感觉dt_root_addr_cells，应该是dt_root_addr_cells = sizeof(_u32) * dt_root_addr_cells，不然  
base不等于reg[0], size 不等于reg[1]，能帮忙解释一下吗，谢谢？

[回复](http://www.wowotech.net/memory_management/memory-layout.html#comment-5543)

**Nick**  
2017-02-22 18:35

early_init_dt_scan_memory应该只会去扫dtsi里面的node吧，奇怪的是为啥dtsi里面有关memory node的都去掉了，bootcommandline里面mem@也去掉了，居然还能打印出扫到memory node的log...  
  
early_init_dt_scan_memory  
memory scan node memory, reg size 8, data: 0 7001 2000000 1000000,  
- 0 ,  1700000

[回复](http://www.wowotech.net/memory_management/memory-layout.html#comment-5236)

**云**  
2017-02-23 12:59

@Nick：可能的原因如下：  
1. “bootcommandline里面mem@也去掉了”， 你这里只修改了kernel的config中的cmdline， 对吧？  
2. 如果你使用u-boot来引导kernel, 那么在booti的过程中，u-boot会将环境变量bootargs中的mem=xxxM解析出来，并且写入dtb中。 故kernel初始化时还是能scan到这个memory node.  
  
因此， 要彻底去掉memory node,你还需要额外将bootargs中的memory信息去掉； 不过那样你的kernel估计都没法boot起来：）

[回复](http://www.wowotech.net/memory_management/memory-layout.html#comment-5239)

**Nick**  
2017-02-27 13:50

@云：@云，谢谢云的回复，我是将dtsi里面chosen的bootargs中的memory信息去掉的...然后发现还是有打印出扫到memory node的log... dtsi里面也没有任何memory node的信息了...

[回复](http://www.wowotech.net/memory_management/memory-layout.html#comment-5258)

**云**  
2017-02-27 14:34

@Nick：我指的是uboot的环境变量里面的bootargs. 你在uboot中pri看看bootargs中有木有mem=xxx。

[回复](http://www.wowotech.net/memory_management/memory-layout.html#comment-5260)

**Nick**  
2017-02-27 16:13

@云：addmisc=setenv bootargs ${bootargs}  
在uboot里面只搜到这个，没有找到mem=xxx,只有在dtsi里面有。

[回复](http://www.wowotech.net/memory_management/memory-layout.html#comment-5262)

**Nick**  
2017-02-27 16:15

@Nick：另外请教一下，这个early_init_dt_scan_memory 里面的log出来，  
  
memory scan node memory, reg size 8, data: 0 7001 2000000 1000000,  
reg size怎么只有8，不是应该是32吗？

[回复](http://www.wowotech.net/memory_management/memory-layout.html#comment-5263)

**[bsp](http://www.wowotech.net/)**  
2017-11-27 11:05

@Nick：我做过实时dump dtb的工具，可以将 内存里最后形成的dtb 变成dts。  
从最终的dts看到 是有memory节点或者启动参数中有‘mem=***’的。  
原理如下：  
1, check the reserved memory in /sys/kernel/debug/memblock/reserved, find which range is dtb by: devmem $dtb_addr |grep EDFE0DD0, EDFE0DD0 is the dtb's magic number;  
2, dump the dtb memory into a tmp file by mapping /dev/mem;  
3, use dtc to convert the dtb to dts.

[回复](http://www.wowotech.net/memory_management/memory-layout.html#comment-6241)

**[wowo](http://www.wowotech.net/)**  
2017-11-27 18:51

@bsp：多谢分享，非常帅的工具！！！

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
    
    - [Linux内核同步机制之（八）：mutex](http://www.wowotech.net/kernel_synchronization/504.html)
    - [计算机科学基础知识（三）:静态库和静态链接](http://www.wowotech.net/basic_subject/static-link.html)
    - [内存初始化（上）](http://www.wowotech.net/memory_management/mm-init-1.html)
    - [ARM64的启动过程之（一）：内核第一个脚印](http://www.wowotech.net/armv8a_arch/arm64_initialize_1.html)
    - [Linux DMA Engine framework(1)_概述](http://www.wowotech.net/linux_kenrel/dma_engine_overview.html)
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