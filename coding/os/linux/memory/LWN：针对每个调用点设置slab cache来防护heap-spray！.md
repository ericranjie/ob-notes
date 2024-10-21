
Linux News搬运工

_2024年09月02日 13:29_ _上海_

关注了就能看到更多这么棒的文章哦～

## Per-call-site slab caches for heap-spraying protection

> By Jonathan Corbet\
> August 20, 2024\
> Gemini-1.5-flash translation\
> https://lwn.net/Articles/986174/

攻击者经常使用的一种入侵系统的手段是 **堆喷射 (heap spraying)** 。简单来说，攻击者会用可能多的精心构造的数据填充堆（heap），希望目标系统能够以错误的方式使用这些数据。如果能够阻止堆喷射，攻击者将失去一个重要的工具。内核现在已经拥有了一些堆喷射防御机制，包括即将在 6.11 版本中合并的 **专用桶式分配器 (dedicated bucket allocator)**，但它的作者 Kees Cook 认为还可以做更多的事情。

堆喷射攻击可以通过尽可能多地分配对象并用攻击者选择的数据填充每个对象来执行。如果内核能够被欺骗来使用这些数据，例如当作是要调用的函数的地址，那么攻击者就可以获得控制权。堆喷射本身并不是漏洞，但它可以简化实际漏洞的利用，例如 **释放后使用 (use-after-free)** 错误或可以覆盖指针。内核的 `kmalloc()` 函数（以及几个变体）是从堆中分配内存的。由于 `kmalloc()` 在整个内核中被广泛使用，任何可以用于堆喷射的调用站点都可能被用于利用内核中遥远、无关部分的漏洞。这使得 `kmalloc()` 堆成为对攻击者非常有吸引力的攻击目标。

`kmalloc()` 从一组固定大小对象的 “桶 (bucket)” 中分配内存；其中大多数（但不是全部）大小是 2 的幂。例如，一个 48 字节的分配请求将导致分配一个 64 字节的对象。 `kmalloc()` 底层结构从某种意义上来说是一个堆数组，每个堆用于分配特定大小范围的内存。这种分离可以使堆喷射攻击更容易，因为不需要覆盖整个堆来针对特定大小的对象。

专用桶式分配器为被认为存在特别高风险出现堆喷射攻击的分配站点创建了一组单独的桶。例如，任何可以从用户空间发起并用用户提供的数据填充的分配都将是专用桶集合的合适使用场景。然后，即使攻击者设法彻底喷洒那个堆，也不会影响其他任何分配；攻击者精心选择的数据不能被用来攻击内核的任何其他部分。

获得最全面堆喷射防护的方法是为每个 `kmalloc()` 调用站点创建一组专用桶。但这将很昂贵；每组桶都会占用相当多的内存。这种底层函数如果变得太低效的话，会让内核开发人员倾向于在权衡之后放弃；因此为每个调用站点创建一组桶是不可能发生的。

Cook 的 **这组新的补丁集 (This new patch series)** 是基于一个事后看来显而易见的观察结果：大多数 `kmalloc()` 调用站点请求的是固定大小的对象，这些对象永远不会改变。通常，这个大小（例如，特定结构的大小）在编译时是已知的。在这种情况下，为调用站点提供一个专用于所需大小的单一 **slab (slab)** 将提供与之等效的堆喷射攻击防护级别。没有必要为所有其他大小提供桶；它们永远不会被使用。

这个想法唯一的问题是，内核中有数千个 `kmalloc()` 调用站点。逐一检查每一个将是一项繁琐且可能容易出错的任务，会导致大量代码变更。但是，编译器知道传递给任何给定 `kmalloc()` 调用的大小参数是编译时常量与否；所需要的只是一个将该信息传达给调用本身的方法。如果该信息伴随着识别调用站点的某个内容，slab 分配器就可以为有意义的调用站点设置专用 slab。

因此，问题归结为如何以一种有效的方式将该信息传达给 =kmalloc()=。Cook 的方法是对 **代码标记框架 (code-tagging framework)** 的一个有趣改编，该框架在 6.10 版本中被合并。代码标记是内存分配概要分析子系统的一部分，该子系统旨在帮助查找与分配相关的错误；它将分配与请求它们的调用站点绑定在一起，因此比如说开发人员可以找到内存泄漏的来源。

代码标记并非真正作为内核强化技术，但它确实提供了这里需要的调用站点信息。Cook 的系列首先通过添加一个指示器来增强为每个调用站点存储的标记信息，该指示器表明分配大小是否为常量，如果是，则该大小是什么。当 `kmalloc()` 调用被执行时，该信息将对 slab 分配器可用。

如果给定的分配请求处于 `GFP_ATOMIC` 级别，它将以通常的方式进行处理，以避免在该路径上添加任何额外的分配动作。否则，分配器将检查该调用站点是否使用固定大小；如果是，则将为此站点创建一个专用 slab 并用于满足分配请求（以及所有后续请求）。如果大小 _不_ 为固定值，则将改为创建一个完整的桶集。无论哪种情况，该决定都将存储在代码标记中以加快未来的调用。值得注意的是，直到第一个调用执行之前，才为任何给定的调用站点完成此设置，这意味着它不会为在任何给定内核中永远不会执行的许多 `kmalloc()` 调用站点执行。

如果这个系列被合并，内核将拥有三级针对堆喷射攻击的防御机制。 **随机 slab 选项 (randomized slab option)** 在 6.6 版本中被合并，它创建了 16 组 slab 桶，然后将每个调用站点随机分配到一组。它的内存开销相对较低，但保护效果是概率性的——它减少了攻击者能够喷洒目标堆的可能性，但没有消除它。 **专用桶选项 (dedicated-buckets option)** 提供更强的保护，但它受到需要显式识别有风险的调用站点并手动隔离它们的限制。相反，这个新的选项提供了强大的堆喷射防护，但它不可避免地会增加 slab 分配器的内存开销。

这种开销的多少将取决于正在运行的工作负载。对于一个未指定的 Linux 内核发行版，Cook 报告说， `/proc/slabinfo` 中报告的 slab 数量增长了五倍左右。如果这个系列被合并到 **主线 (mainline)** 中，将由发行版决定是否启用此选项。然而，当内核要在存在堆喷射攻击高风险的系统上运行时，这可能是一个容易做出的决定。

全文完\
LWN 文章遵循 CC BY-SA 4.0 许可协议。

欢迎分享、转载及基于现有协议再创作～

长按下面二维码关注，关注 LWN 深度文章以及开源社区的各种新近言论～

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

素材来源官方媒体/网络新闻

Read more

Reads 311

修改于2024年09月04日

​

People who favorited this content also liked

LWN：调查文件系统的错误来源！

我关注的号

Linux News搬运工

不喜欢

不看的原因

OK

- 内容低质
- 不看此公众号内容

![](https://mmbiz.qpic.cn/sz_mmbiz_jpg/NhncyvThqYkL4cePRn7nEbOqXHFMh80tb7oPK38rYuPFdicNbkSicpqw14X0Y9nzjuw578v4ibQibFa1b6agzvvFeg/0?wx_fmt=jpeg)

UGit：腾讯自研的 Git 客户端，真的好用！

FED实验室

不喜欢

不看的原因

OK

- 内容低质
- 不看此公众号内容

![](https://mmbiz.qpic.cn/mmbiz_jpg/9RM1R8UKDIaf4ficOIkNkerbjJt0gknYG5PZAs6BUQiaFBicmQK6YchunIj8b4PsE0BrI8BqzJsWBmIg53Zoic3QuA/0?wx_fmt=jpeg&tp=wxpic)

循序渐进讲解linux系统发送网络包处理流程

我常看的号

Linux开发架构之路

不喜欢

不看的原因

OK

- 内容低质
- 不看此公众号内容

![](https://mmbiz.qpic.cn/sz_mmbiz_jpg/8pECVbqIO0yeYDGXUt7gOz4yAaaPoxZNGMIyQhqLneyibPP2QRp51M9DwYMlTHAzeicF3ZSDg6VklzoqIWe0pnZA/0?wx_fmt=jpeg)

Comment

**留言 1**

- 张涛Robin

  5天前

  Like1

  use after free，应该译为：（内存）释放后再次被使用

已无更多数据

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/NhncyvThqYlCdgIwkM1pqCsEOWbfOLTGyVMR6IR4OAn5GpNpjdjibAdTiaTckshz2ia2JaoNgw3cibIuerUy89KE8Q/300?wx_fmt=png&wxfrom=18)

Linux News搬运工

2151

1

Comment

**留言 1**

- 张涛Robin

  5天前

  Like1

  use after free，应该译为：（内存）释放后再次被使用

已无更多数据
