
Linux内核之旅

 _2024年04月26日 17:51_ _陕西_

以下文章来源于Linux工坊 ，作者知书码迹

[

![](http://wx.qlogo.cn/mmhead/Q3auHgzwzM5a9XjdxW2ojPn8dI88M4XMZC2MUI1wlW8kLUfxItetDw/0)

**Linux工坊**.

融实用性和趣味性于一体，专注于Linux内核、JAVA、数据结构算法面试题、物联网络等领域热点分享。以新视角分享码农圈内技术干货。期待与你共同成长。个人主页：https://szp2016.github.io/

](https://mp.weixin.qq.com/s?__biz=MzI3NzA5MzUxNA==&mid=2664617576&idx=1&sn=2ec1ed28ee18f1fbe3b8768fa5980df7&chksm=f04dfb8dc73a729b9ab65bd83a0d3eae98630ed6d68b7b5a4afaf4241f0f093326aab6c421ec&mpshare=1&scene=24&srcid=0426cZVqW51nm5juWdDOThbG&sharer_shareinfo=6f940134b39bba51d1be86e2feeca506&sharer_shareinfo_first=6f940134b39bba51d1be86e2feeca506&key=daf9bdc5abc4e8d0883447fceb9c13bf1c9f1ae083469e55531b922f5f674102266e68fbd73775569d980b5d6c154fd1695c40ecdc77879cec4034037a25313064bc38bbc9ff0837d4e7448c440ebcae1fc7faca653d2a73333be4ccc2e78a39ddb563ae97037bf21580d1c0e416e7ec156b28a30c77318d7c74ca9ca2715829&ascene=0&uin=MTEwNTU1MjgwMw%3D%3D&devicetype=Windows+11+x64&version=63090b19&lang=zh_CN&countrycode=CN&exportkey=n_ChQIAhIQ%2B889tEc8Pz0uP4NtN1Om%2BhLmAQIE97dBBAEAAAAAAK1fLSjaZjwAAAAOpnltbLcz9gKNyK89dVj0A%2BZrOfNSa4E8oFuu9FgfPlpLjF7z3CySGJfxRUgrJB6ZhkqV%2FM%2BAwn33%2FWPw7pd6TzeuCQFJjkcUiI6bNjhtYMs6WlPAQnJLtAznJj5AayOfDii75KLh%2F32RJehfh2ClhQYOCzTwre%2FPqMKPGXck%2BKfQyz0DFw9RR5uQl00XUWE1kQqnbweoO9AYpvJG%2BH0V8mKDvgmY%2BgLKSrDl2g%2FGOlukdYLky1zhOpLIqRW5csNjT4R1ijYI%2Bc78%2F76tgCve&acctmode=0&pass_ticket=Mg5CW0oHy6v3l9uGSFMOryrfdVO6TZZyZzPyn%2B5g7r5MLjv61RcI1HdyIROPoU8x&wx_header=1&fasttmpl_type=0&fasttmpl_fullversion=7350504-zh_CN-zip&fasttmpl_flag=1#)

eBPF 技术十分强大，也非常令人振奋，它允许开发者向 Linux 内核的关键部分添加自定义代码，并通过编写简单的 C 或 Go 程序与之交互。

  

利用 eBPF 程序，你可以检查它们挂载的进程内存中的数据。但是，为了实现这一点，eBPF 程序需要了解它们处理的数据结构。为了做到这一点，你只需添加一行简单的代码：#include "vmlinux.h"。

  

☾前世今生☽

vmlinux.h 是一个由当前运行的 Linux 内核生成的头文件，其中包含了内核源代码中使用的所有类型定义。

  

这个头文件的存在使得开发者能够更加方便地在编写 eBPF 程序时使用内核数据结构，从而实现更精细化的操作和控制。

在编译构建 Linux 内核时，其中一个关键输出文件是名为 vmlinux 的可执行文件，它通常被称为内核镜像。

  

vmlinux 是一个 ELF 格式的文件，包含了编译后的可引导内核，其中包括了符号表、代码段、数据段等信息，为内核的运行提供了基础。

  

在 Linux 的领地中，有一把名为 bpftool 的工具，它有一个特定的功能：读取 vmlinux 镜像文件，生成 vmlinux.h 头文件。这个文件包含了内核中的每个类型定义，文件体积通常非常庞大。  
  
bpftool 的命令如下：  

bpftool btf dump file /sys/kernel/btf/vmlinux format c > vmlinux.h

  

  

![](https://mmbiz.qpic.cn/mmbiz_jpg/6TLLOkcZRbib6Y84LWgw6bn1ibgZ96kkM6uezbkSap5OcIT2FpgMMFRwP9cB0LYnMQicsB1QJTHiabGYpMzItHJiaibw/640?wx_fmt=jpeg&wxfrom=13&tp=wxpic)

  

通过这个命令，bpftool 仿佛化身为奇妙的艺术家，从镜像文件中提取数据，以 C 语言形式呈现在 vmlinux.h 文件中。如此一来，在编写 eBPF 程序时，您能轻松地使用内核的类型定义，更好地探索和运用内核的数据结构和功能。  

  

当你导入这个头文件时，你的bpf程序就能读取原始内存，并知道哪些字节对应于你想要使用的结构体的哪些字段！  
  
比如，Linux用一个叫做task_struct的类型来表示进程的概念。如果你想要在bpf程序中检查task_struct中的值，你就需要知道它的定义。

  

☾多内核兼容☽

由于 vmlinux.h 文件是从你安装的内核生成的，如果你尝试在运行不同内核版本的另一台计算机上运行它而没有重新编译，你的 bpf 程序可能会出现问题。这是因为，随着版本的变化，Linux 源代码中内部数据结构的定义也会发生变化。

  

然而，使用 libbpf 可以实现所谓的 “CO:RE” 或 “Compile once, run everywhere”（一次编译，到处运行）。在 libbpf 中定义了一些宏（例如 BPF_CORE_READ），它们会分析你尝试访问的在vmlinux.h 中定义的类型中的字段。如果您想要访问的字段在正在运行的内核使用的结构定义中被移动，这些宏助手会帮助你找到它。因此，无论你是否使用从自己内核生成的 vmlinux.h 文件编译了 bpf 程序，然后在另一个内核上运行，都没有关系。

  

☾总结☽

vmlinux.h 是从当前运行的 Linux 内核生成的头文件，包含了内核源代码中使用的所有类型定义。对于 eBPF程序而言，它至关重要，因为它使开发者能够直接与内核数据结构和函数交互，而无需依赖外部内核头文件。将 vmlinux.h 包含在 eBPF 程序中，开发者就能够直接访问和操作内核数据，实现更加高效和强大的程序执行。  
  
在本质上，vmlinux.h 充当了 eBPF 程序与 Linux 内核之间的桥梁，为开发者提供了对关键内核内部的访问，同时确保了不同内核版本之间的兼容性。这降低了由于内核头文件更改而导致程序破坏的风险，并简化了 eBPF 应用程序的开发过程。

  

参考文献：https://www.aquasec.com/blog/vmlinux-h-ebpf-programs/

阅读 1644

​

写留言

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/SeWfibBcBT0EibtIWVNvshnuWMN1AoJw3poFIsbpaIVyZibCCqwBUR21rcDfrQgoqYzaYNdS14IIXzvmzvibdDa5Rw/300?wx_fmt=png&wxfrom=18)

Linux内核之旅

13837

写留言

写留言

**留言**

暂无留言