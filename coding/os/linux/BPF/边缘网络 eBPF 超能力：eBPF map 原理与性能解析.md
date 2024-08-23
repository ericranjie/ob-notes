# 

Linux云计算网络

 _2022年01月11日 08:13_

以下文章来源于火山引擎边缘计算 ，作者SimonCZW

[

![](http://wx.qlogo.cn/mmhead/Q3auHgzwzM5VaCZiaFRLzPdK5RUwvzrxFU4sJibfq8QwyEQTmV2K4vRw/0)

**火山引擎边缘计算**.

火山引擎边缘云官方账号，连接与计算无处不在。

](https://mp.weixin.qq.com/s?__biz=MzI1OTY2MzMxOQ==&mid=2247498537&idx=1&sn=0f69384dd802b80b2bdc68b7affbdae2&chksm=ea77cf91dd00468776106abb7d18060db6904d01f456f83aee4fc288bb9e05b02712bac3ddd0&mpshare=1&scene=24&srcid=0111IfmgxB0mEG2SSRLvEKDY&sharer_sharetime=1641886527610&sharer_shareid=5fb9813bfe9ffc983435bfc8d8c5e9ca&key=daf9bdc5abc4e8d0d11c193eca0e3188149315428b0783966792229f08b6f226a96b0dd828c65d1f4f55ea03ac5a9f7704871b196d18f4a27488637fb88a9449ae6a0526fc60c354dd0c07ff863622ec4aa7a2c9211e683b1bf33b53116dab07d815ef7e49555d3bdbb021c775675042ed45fb09f2fcb444a728bc673459f5d6&ascene=0&uin=MTEwNTU1MjgwMw%3D%3D&devicetype=Windows+11+x64&version=63090b19&lang=zh_CN&countrycode=CN&exportkey=n_ChQIAhIQafUlm5h1ORg%2BZaqPdM6X8RLmAQIE97dBBAEAAAAAAICMF8bz5o4AAAAOpnltbLcz9gKNyK89dVj0RCQSN8GTS8bGuGjD0S8IXpXXTpIuDHWGDkMI3oftgTFWdvParZHkhjADxcj3NXhiUD0g5ZnL%2F%2BBXGfJKA7Nu6YgpKeIbqjabQ30kr5K7cDT3N6SMQ3qMnQQR5FLxN3Fw5b4Dp7o9y%2BuKBAKZ6pfe1%2FTWkO9w733u1y%2Fjt11fen%2BsFebWeqnK%2FET7TCdr1M9IO%2F74aw7K7x5wxU%2Fh5v%2BeGCoDlaiAYAuf9155b0V7al8lhkiEHZ%2F7XSRiQDutA0aq&acctmode=0&pass_ticket=b%2FZRlh%2B3oWAOivmuCeyX9nQNkpUYukp7yR%2FtKIevEE8n8wj5GN1FUZha%2BQwnwkWc&wx_header=1&fasttmpl_type=0&fasttmpl_fullversion=7351805-zh_CN-zip&fasttmpl_flag=1#)

导读

众所周知，大型 eBPF 程序构建过程中 eBPF map 必不可少。火山引擎边缘计算在数据面也大量使用了 eBPF 及其 map 机制。如何用好 map 是 eBPF 网络编程中关键的一环，不同 map 的性能差异也较大。本文组织 eBPF map 相关的底层实现，为大家详细解析 eBPF map 的原理及性能。

  

1. 什么是 eBPF map
    
2. eBPF map 原理
    
3. eBPF map 性能
    
4. 总结
    

  

**01**

# **背景**

  

众所周知，大型 eBPF 程序构建过程中 eBPF map 必不可少。map 是打通数据面和控制面的关键机制。在实际使用过程中，我们可以通过 map 存储弹性公网 IP 配置数据、在数据面匹配时通过 map 来查询弹性公网 IP，然后执行限速、NAT 等逻辑，以及通过 map 来存储链接等。

  

火山引擎边缘计算在数据面也大量使用了 eBPF 及其 map 机制，并基于 eBPF 实现了 VPC 网络、负载均衡、弹性公网 IP、外网防火墙等一系列高性能、高可用的云原生网络解决方案。

  

![图片](https://mmbiz.qpic.cn/mmbiz_png/RzqQKpfl3D83zLL6cyJLk5iaYz6ibNGQ7ibJbQTgKVbfiab8yFrTDSoGoUNKz9Wdx0PNiciaxGnjLwqCGlk2icOBB4ZHA/640?wx_fmt=png&wxfrom=13&tp=wxpic)

火山引擎边缘计算云平台架构图

  

eBPF map 有多种不同类型，支持不同的数据结构，最常见的例如 Array、Percpu Array、Hash、Percpu Hash、lru Hash、Percpu lru Hash、lpm 等等。那么选取哪个类型的 map，如何用好 map 就是 eBPF 网络编程中关键的一环，不同 map 的性能也是相差很大的。本文组织 eBPF map 相关的底层实现，为大家详细解析 eBPF map 的原理及性能。

  

  

02

# **什么是 eBPF map**

  

eBPF map 是一个通用的数据结构存储不同类型的数据，提供了用户态和内核态数据交互、数据存储、多程序共享数据等功能。官方描述[1]：

  

> eBPF maps are a generic data structure for storage of different data types. Data types are generally treated as binary blobs, so a user just specifies the size of the key and the size of the value at map-creation time. In other words, a key/value for a given map can have an arbitrary structure.

> A user process can create multiple maps (with key/value-pairs being opaque bytes of data) and access them via file descriptors. Different eBPF programs can access the same maps in parallel. It's up to the user process and eBPF program to decide what they store inside maps.

  

## **eBPF 数据面中怎么使用 map**

  

在 eBPF 数据面中，我们使用 eBPF map 只需要按照规范定义 map 的结构，然后使用 bpf_map_lookup_elem、bpf_map_update_elem、bpf_map_delete_elem 等 helper function 就可以对 map 进行查询、更新、删除等操作。

  

下面以开源项目 cilium[2] 展示了一个 map 的使用例子：

  

1、map 的定义：定义全局的变量 ENDPOINTS_MAP，定义了 map 相关属性，比如类型 hash、key value 的大小、map 的大小等等。

  

```
struct bpf_elf_map __section_maps ENDPOINTS_MAP = {
```

  

2、查询 map：

  

```
static __always_inline __maybe_unused struct endpoint_info *
```

  

可以看到：map_lookup_elem 帮助函数只需要传入 &ENDPOINTS_MAP 和 key 即可。

  

那么问题来了：

  

- 在内核态中 ENDPOINTS_MAP 的内存是怎么分配的？
    

- 内核态不同的 eBPF 程序怎么复用同一个 ENDPOINTS_MAP，每个程序怎么拿到 ENDPOINTS_MAP 的内存地址？
    

- 用户态程序又是怎么使用 map，怎么关联上 ENDPOINTS_MAP 并对其进行操作？
    

  

  

03

# **eBPF map 原理**

  

## **eBPF 加载器与 map**

  

eBPF 编程绕不开的是：将编写好的 eBPF 程序加载到内核，然后在内核态执行 eBPF 程序。因此需要有一个加载器将 eBPF 程序以及程序使用的 eBPF map 加载到内核中（或者复用已存在的 map）。

  

## **eBPF 加载器介绍**

  

eBPF 程序加载的本质是 BPF 系统调用，Linux 内核通过 BPF 系统调用提供 eBPF 相关的一切操作，比如：程序加载、map 创建删除等。常见的 loader 都是对这个系统调用的封装，部分 loader 提供更加原生接近系统调用的操作，部分 loader 则是进行了更多封装使得编程更便捷。下面介绍一下常见的 loader。

  

- **iproute2**
    

  

iproute2[3] 提供 ip 命令、tc 命令将 eBPF 程序加载到内核并挂载到网口，提供更多封装能力。开发者只需要实现 eBPF 代码，使用 ip/tc 命令指定挂载的 eBPF 程序和挂载的网口即可。

  

```
# xdp程序类型
```

  

iproute2 提供了更便利的使用方式，比如：上面看到定义的 ENDPOINTS_MAP 中，定义了 pinning 属性为 PIN_GLOBAL_NS。iproute2 就会将这个 map pin 到 eBPF 文件系统中，如果 eBPF 文件系统已存在一个 pinned 的 map 则直接复用，实现多个程序共享一个 map 的效果。

  

典型案例：cilium 项目使用 iproute2 加载 BPF 程序。

  

- **libbpf 库**
    

  

内核实现的 libbpf 库[4]，封装了 BPF 系统调用，使得加载 BPF 程序更便捷。libbpf 不像 iproute2，它能够使 BPF 相关操作更为便捷，没有做过多封装。如果要将程序加载到内核，则需要自己实现一个用户态程序，调用 libbpf 的 API 去加载到内核。如果要复用 pinned 在 BPF 文件系统的 MAP，也需要用户态程序调用 libbpf 的 API，在加载程序时进行相关处理。

  

典型案例：Facebook 开源的 katran 项目使用 libbpf 加载 eBPF 程序。

  

- **cilium/ebpf 库**
    

  

cilium/ebpf 库[5] 是一个 GO 语言版本的 libbpf 库，它封装了 BPF 系统调用，与内核提供的 libbpf 类似。使用 cilium/ebpf 库实现用户态程序加载 eBPF 到内核，在很多方面都类似 libbpf，区别在于这个库是 GO 语言的，更加方便使用 GO 语言构建一套 eBPF 程序的控制面方案。

  

- **bcc**
    

  

bcc[6] 实现了将用户态编译、加载、绑定的功能都集成了起来，方便用户使用，对用户的接口更友好。支持 Python 接口以及很多基于 eBPF 实现的分析工具。

  

## **BPF 系统调用**

  

Linux 内核通过 BPF 系统调用并提供 BPF 相关的能力。对于 eBPF 编程中的 map，当然也有 BPF 系统调用提供的能力。BPF 系统调用定义：

  

```
SYSCALL_DEFINE3(bpf, int, cmd, union bpf_attr __user *, uattr, unsigned int, size)
```

  

BPF 系统调用通过第一个参数 cmd 来区分相关的 BPF 操作，map 常见的 cmd 有：创建 MAP、查询 MAP 中元素、更新 MAP 中元素、删除 MAP 中元素等等。cmd 如下：

  

```
enum bpf_cmd {
```

  

## **eBPF map 的创建**

  

细心的你可能已经发现 BPF 系统调用有一个 BPF_MAP_CREATE 的cmd，这就能回答我们上面的第一个问题：在内核中，ENDPOINTS_MAP 的内存是怎么分配的？

map 是需要调用 BPF 系统调用来创建的。内核态的系统调用执行过程有几步：

  

- 根据 MAP 类型，调用具体 MAP 类型的处理，并根据指定的 key size、value size、map size 分配内核态的内存；
    

- 为 MAP 分配唯一的 id；
    

- 为 MAP 创建一个匿名 fd，并返回；
    

  

**在我们的 eBPF 代码中，仅需要定义 map 全局变量，即可在代码中直接使用了，没有相关调用 bpf syscall 创建 map 的逻辑。**那么其内部机制是怎样的？是 map 创建的过程然后由 loader 加载器完成的，编译器和加载器根据同一个约定完成这项工作。

  

上面 cilium 的例子中，ENDPOINTS_MAP 的全局变量定义时有一个关键字 __section_maps，这个关键字是一个宏，最终展示是 __attribute__((section("maps")))。这个编译器属性告诉编译器将 ENDPOINTS_MAP 变量放在编译生成的 .o 文件(elf)中，名为 maps 的 section。

  

在使用 iproute2 加载程序时，打开 .o 文件时，会读取 maps 命名的 section，并将其中存储的一个个 map 读取出来，然后调用 BPF 系统调用在内核创建 eBPF map。

  

如果是 libbpf 或 cilium/ebpf 库，需要调用 API 将 .o 文件打开，也支持相应的解析 elf 文件工作。并根据 API 调用规范，后续调用相关 API 创建 MAP。所以，这些 lib 库需要我们自己实现用户态的程序加载 BPF 代码到内核，而 iproute2 完全封装了这些流程。当然使用 lib 库能够更加灵活去做一些工作。

  

比如，在 libbpf 的 API bpf_object__open 打开 .o 文件的过程，就会解析文件中 section 名字为 maps 的段，将每个 map 解析出来。调用链：

  

```
bpf_object__open
```

  

另外有一个点值得注意，libbpf 和 iproute2 对 map 的结构定义是不一样的。libbpf 是：

  

```
struct bpf_map_def {
```

  

而 iproute2 是：

  

```
struct bpf_elf_map {
```

  

可以看到 iproute2 的定义是增加了 id 和 pinning 两个字段，用于提供更加便捷的功能。比如：pinning 用于指定这个 map 是否需要 pin 到 BPF 文件系统，用于复用 map。

  

其实内核提供的 BPF 系统调用，只需要5个关键属性作为参数（type/key_size/value_size/max_entries/map_flags），这个结构叫什么是什么并不重要，如果你自己实现 loader 来解析 .o 文件，也可以自定义 map 结构，只需要在 loader 最终调用 BPF 系统调用时有这几个参数即可。

  

**总结来说，loader 会调用 BPF 的系统调用到内核创建 eBPF map，然后内核返回创建 map 的 fd。**

  

## **eBPF 程序与 map 关联**

  

我们程序代码中直接使用 map 时，直接使用 map 全局变量，怎么与 loader 通过系统调用创建的 map 关联？

  

事实上，程序访问 map，关键的实现如下：

  

1. 在 loader 加载 BPF 程序到内核之前，loader 都会先将所有定义在“maps” section 中的 map 创建在内核中（也可能是复用内核已有的 map），内核会返回 map 的 fd。
    
2. loader 将内核返回的 map 的 fd，替换到使用的 map eBPF 指令的常量字段中，相当于直接修改编译后的 BPF 指令。
    
3. loader 在指令替换 map fd 后，才会调用 cmd 为 BPF_PROG_LOAD 的 bpf syscall，将程序加载到内核。
    
4. 内核在加载 eBPF 程序系统调用过程中，会根据 eBPF 指令中常量字段存储的 MAP fd，找到内核态中的 map，然后将 map 内存地址替换到对应的 BPF 指令。
    

5. 最终，BPF 程序在内核执行阶段能根据指令存储的内存地址访问到 map。
    

  

上面一句话听起来有点抽象，我们将实现剖析一下。

  

首先，在上文已经介绍了 loader 会解析“maps”命名的 section，libbpf 会将 map 数据保存在一个 bpf_object 结构并返回。然后，用户态的程序还需要调用 libbpf 的 API bpf_object__load 将 bpf_object 结构真正加载到内核（注：iproute2 不需要你调用这么多 API，只需要根据其编码规范定义相关结构，然后它会帮你完成这一切）。

  

bpf_object__load API 关于 MAP 的处理调用链：

  

- 先根据解析好的 map，调用 BPF 系统调用到内核创建 map；
    

- 将 map 的 fd 替换到 BPF 指令中；
    

- 采用 BPF 程序（指令）调用 BPF 系统调用将程序加载到内核；
    

  

```
bpf_object__load
```

  

loader 是怎么将 map 的 fd 替换到指令中的呢？

  

在 BPF 程序编译后，生成的 .o 文件是可重定位的对象文件(Relocatable file)，在一般的程序编译过程中，还需要一步链接，链接器将各个目标文件组装在一起，解决符号依赖，库依赖关系，最终生成可执行文件。对于BPF编程来说，只需要生成 .o 文件，然后将文件加载到内核。对于 Loader，在将文件加载到内核前，会把 .o 文件中的 BPF 指令读取出来，解决 map 的“重定位”，然后再将指令加载到内核。Loader 算是完成了 map 的“链接”工作。

  

下面以一个程序实际例子来说明 map 重定位过程，有兴趣可以查看各个 loader 的实现，最终的结果是一致的。对于 BPF 编程，定义一个程序的格式会在函数前定义 SEC()。以 Facebook katran 代码为例子：

  

```
SEC("xdp-root")
```

  

而 SEC 宏：实际上是告诉编译器将下面函数代码放到 .o 文件中以 NAME 命名的 section。

  

```
#define SEC(NAME) __attribute__((section(NAME), used))
```

  

所以，通过 readelf -S 查看编译生成的 .o 文件，可以看到一个以 xdp-root 命名的 section，这个 section 存放的就是 xdp_root 的 BPF 程序编译后的 BPF 指令。同时，我们可以看到有一个 .rel 前缀加上 xdp-root 命名的 section，这个 section 存放的就是 xdp-root 代码中需要重定位的符号 symbol。这个 .relxdp-root section 的头部信息可以看到 type 是 REL，Info 是 13（对应 xdp-root 的 section id）：

  

```
Section Headers:
```

  

通过命令 readelf -x 14 可以将可重定位对象文件 (.o) 的 .relxdp-root section 读取出来：

  

```
Hex dump of section '.relxdp-root':
```

  

根据 Section Headers 信息可知 .relxdp-root 这个 section 每个 entry 的大小是 0x10（16字节），对应上述 Hex dump 出来的每一行的大小。所以 .relxdp-root section 共有4个 entry。

  

对应上述 dump 处理的数据，每一个重定位的 entry 的组成：

  

- 低 64 位是 offset，用于关联当前 entry 对应代码 section 中具体的指令。代码 section 指令位置到其 section 起始地址，偏移的长度 offset 等于这个 offset；
    

- 高 64 位是 Info，其中的低 32 位对应重定位 symbol 的 type，高 32 位对应重定位 symbol 的index（在 symbol 表的 id）。
    

  

由于字节序问题，上述 dump 对应 entry 结构反过来看：

  

以第一个 entry 为例子：symbol index 是 44030000（字节序转换后：0x0344），通过 readelf -s 命令查看符号表 836(0x344) 对应的符号是 root_array，符合我们上 xdp-root 的代码。我们看到重定向 entry 有4条，都是对应 root_array 符号，代码共4次循环调用 root_array 符号（BPF 编译会将循环展开）：

  

```
# readelf -s
```

  

通过 readelf 直接将 xdp-root section 打印出来（都是 BPF 指令）：

  

```
Hex dump of section 'xdp-root':
```

  

根据第一个重定位 entry 的 offset 是 08000000 00000000（字节序转换后是 0x8），找到对应指令（编译后 .o 文件中的 BPF 指令都是 8 字节为单位）：18020000 00000000。

  

在编译后的汇编指令结构是：

  

```
struct bpf_insn {
```

  

18020000 00000000 指令的 opcode 是 0x18。opCode 意义是：load memory、immediate value、size 是 double(64 bits)，用于访问 map 的指令。

  

在 loader 层面对于根据重定向 entry 找到的指令，且这个重定向 entry 关联的 symbol 是 map 情况下，会将 map 的 fd 写到这条指令的常量字段(imm)，并将指令的 src 寄存器设置为 BPF_PSEUDO_MAP_FD。最终，loader 将程序加载到内核时，对于访问 map 的 BPF 指令的常量字段就存储了 map 的 fd。

  

## **内核系统调用的处理**

  

加载 eBPF 程序的系统调用处理过程中，会针对替换到指令的 map fd 进行处理，将其改为 MAP 的内存地址。

  

系统调用 eBPF 程序加载调用过程中，会调用 resolve_pseudo_ldimm64 函数：遍历全部指令，找到访问 MAP 的指令（opcode 是 0x18、src_reg 是1），将 MAP 的 fd 替换为 MAP 的内存地址（写到指令的常量字段），这样 BPF 指令执行过程中就能直接访问 MAP。核心代码：

  

```
static int resolve_pseudo_ldimm64(struct bpf_verifier_env *env){
```

  

## **eBPF map 原理小结**

  

结合上述的分析，**可以说 eBPF map 是编译器、加载器、Linux 内核协同实现的**。

  

- **编译器规范了我们的编码方式**，比如需要定义 map 存放在命名为 maps 的 section。
    

- **加载器约束了 map 定义的结构**，并且会解析 .o 文件在内核创建 map，同时实现“重定位”功能将 map 的 fd 写到相关指令的常量，再将程序指令加载到内核。
    

- **内核根据 fd 找到 map 的内存地址，再替换到指令中**，最终 bpf 指令执行时能够根据内存地址直接访问 map。比如，我们使用 bpf_map_lookup_elem helper 来查询 map，在内核中就能直接拿到 map 的内存地址进行访问。
    

  

另外，多个 eBPF 程序共享一个 MAP 的原理也就不难理解了：

  

- 内核提供了一个 BPF 文件系统，我们可以将 map pin 到文件系统中，然后多程序就能在文件系统找到 MAP 的 fd（不同程序返回不同的 fd，对应的 MAP 是同一个 id）。
    

- 在 loader 层面将共享的 map 的 fd 写到程序指令的常量，内核系统调用处理时将对应 map 的地址替换给指令，最终指令执行就能访问到 map。
    

- 不同的 BPF 程序访问同一个内存地址，就实现了 map 的共享，最终可以构建更为复杂的 eBPF 程序。
    

#   

  

04

# **eBPF map 性能**

  

eBPF map 位于在内核的内存中，eBPF 程序在执行时是直接访问内存的，一般来说我们通过 helper function 来访问 map，比如查询 map 是通过 bpf_map_lookup_elem。不同类型的 map 查询、更新、删除的实现都是不同的，性能也是差别很大，针对不同场景使用不同类型的 map，以及了解不同 map 性能差异对于 eBPF 编程来说相当重要。

  

这里给出经过测试验证的结论及建议：

  

- 一般来说 eBPF 程序作为数据面更多是查询，**常用的 map 的查询性能：array > percpu array > hash > percpu hash > lru hash > lpm**。**map 查询对 eBPF 性能有不少的影响**，比如：lpm 类型 map 的查询在我们测试发现最大影响 20% 整体性能、lru hash 类型 map 查询影响 10%。
    

- 特别注意，**array 的查询性能比 percpu array 更好，hash 的查询性能也比 percpu hash 更好**，这是由于 array 和 hash 的 lookup helper 层面在内核有更多的优化。**对于数据面读多写少情况下，使用 array 比 percpu array 更优（hash、percpu hash 同理），而对于需要数据面写数据的情况使用 percpu 更优，比如统计计数。**
    

- 尽可能在一个 map 中查询到更多的数据，减少 map 查询次数。
    

- 尽可能使用 array map，在控制面实现更复杂的逻辑，如分配一个 index，将一些 hash map 查询转换为 array map 查询。
    

- eBPF map 也可以指定 numa 创建，另外不同类型的 map 也会有一些额外的 flags 可以用来调整特性，比如：lru hash map 有 no_common_lru 选项来优化写入性能。
    

#   

  

**05**

# **总结**

  

通过上述原理的剖析以及实际的性能测试调优，我们对于如何写好 eBPF 程序有了更深刻的理解。在此基础之上，我们通过 eBPF 实现了一套高性能高可用的云原生网络解决方案，欢迎大家使用火山引擎边缘计算相关产品。也欢迎优秀的你加入我们，探索无限可能。

  

> **Tips：**岗位详情及简历投递请微信联系火山引擎边缘计算小助手（微信：19117352830）！

  

**参考资料：**

  

[1] https://man7.org/linux/man-pages/man2/bpf.2.html

[2] https://github.com/cilium/cilium

[3] https://github.com/shemminger/iproute2

[4] https://github.com/torvalds/linux/blob/master/tools/lib/bpf/libbpf.c

[5] https://github.com/cilium/ebpf

[6]https://github.com/iovisor/bcc

  

---

后台回复“加群”，带你进入高手如云交流群

  

**推荐阅读：**

[深入理解 Cache 工作原理](http://mp.weixin.qq.com/s?__biz=MzI1OTY2MzMxOQ==&mid=2247497507&idx=1&sn=91327167900834132efcbc6129d26cd0&chksm=ea77c39bdd004a8de63a542442b5113d5cde931a2d76cff5c92b7e9f68cd561d3ed6ef7a1aa5&scene=21#wechat_redirect)

[Cilium 容器网络的落地实践](http://mp.weixin.qq.com/s?__biz=MzI1OTY2MzMxOQ==&mid=2247497237&idx=1&sn=d84b91d9e416bb8d18eee409b6993743&chksm=ea77c2addd004bbb0eda5815bbf216cff6a5054f74a25122c6e51fafd2512100e78848aad65e&scene=21#wechat_redirect)

[【中断】的本质](http://mp.weixin.qq.com/s?__biz=MzI1OTY2MzMxOQ==&mid=2247496751&idx=1&sn=dbdb208d4a9489981364fa36e916efc9&chksm=ea77c097dd004981e7358d25342f5c16e48936a2275202866334d872090692763110870136ad&scene=21#wechat_redirect)  

[图解 | Linux内存回收之LRU算法](http://mp.weixin.qq.com/s?__biz=MzI1OTY2MzMxOQ==&mid=2247496417&idx=1&sn=4267d317bb0aa5d871911f255a8bf4ad&chksm=ea77c659dd004f4f54a673830560f31851dfc819a2a62f248c7e391973bd14ab653eaf2a63b8&scene=21#wechat_redirect)  

[Linux 应用内存调试神器- ASan](http://mp.weixin.qq.com/s?__biz=MzI1OTY2MzMxOQ==&mid=2247496414&idx=1&sn=897d3d39e208652dcb969b5aca221ca1&chksm=ea77c666dd004f70ebee7b9b9d6e6ebd351aa60e3084149bfefa59bca570320ebcc7cadc6358&scene=21#wechat_redirect)

[深入理解 Cilium 的 eBPF 收发包路径](http://mp.weixin.qq.com/s?__biz=MzI1OTY2MzMxOQ==&mid=2247496345&idx=1&sn=22815aeadccc1c4a3f48a89e5426b3f3&chksm=ea77c621dd004f37ff3a9e93a64e145f55e621c02a917ba0901e8688757cc8030b4afce2ef63&scene=21#wechat_redirect)

[Page Cache和Buffer Cache关系](http://mp.weixin.qq.com/s?__biz=MzI1OTY2MzMxOQ==&mid=2247495951&idx=1&sn=8bc76e05a63b8c9c9f05c3ebe3f99b7a&chksm=ea77c5b7dd004ca18c71a163588ccacd33231a58157957abc17f1eca17e5dcb35147b273bc52&scene=21#wechat_redirect)

[深入理解DPDK程序设计|Linux网络2.0](http://mp.weixin.qq.com/s?__biz=MzI1OTY2MzMxOQ==&mid=2247495791&idx=1&sn=5d9f3bdc29e8ae72043ee63bc16ed280&chksm=ea77c4d7dd004dc1eb0cee7cba6020d33282ead83a5c7f76a82cb483e5243cd082051e355d8a&scene=21#wechat_redirect)  

[一文读懂基于Kubernetes打造的边缘计算](http://mp.weixin.qq.com/s?__biz=MzI1OTY2MzMxOQ==&mid=2247495291&idx=1&sn=0aebc6ee54af03829e15ac659db923ae&chksm=ea77dac3dd0053d5cd4216e0dc91285ff37607c792d180b946bc09783d1a2032b0dffbcb03f0&scene=21#wechat_redirect)

[网络方案 Cilium 入门教程](http://mp.weixin.qq.com/s?__biz=MzI1OTY2MzMxOQ==&mid=2247495170&idx=1&sn=54d6c659853f296fd6e6e20d44b06d9b&chksm=ea77dabadd0053ac7f72c4e742942f1f59d29000e22f9e31d7146bcf1d7d318b68a0ae0ef91e&scene=21#wechat_redirect)

[Docker  容器技术使用指南](http://mp.weixin.qq.com/s?__biz=MzI1OTY2MzMxOQ==&mid=2247494756&idx=1&sn=f7384fc8979e696d587596911dc1f06b&chksm=ea77d8dcdd0051ca7dacde28306c535508b8d97f2b21ee9a8a84e2a114325e4274e32eccc924&scene=21#wechat_redirect)

[云原生/云计算发展白皮书（附下载）](http://mp.weixin.qq.com/s?__biz=MzI1OTY2MzMxOQ==&mid=2247494647&idx=1&sn=136f21a903b0771c1548802f4737e5f8&chksm=ea77df4fdd00565996a468dac0afa936589a4cef07b71146b7d9ae563d11882859cc4c24c347&scene=21#wechat_redirect)

[使用 GDB+Qemu 调试 Linux 内核](http://mp.weixin.qq.com/s?__biz=MzI1OTY2MzMxOQ==&mid=2247493336&idx=1&sn=268fae00f4f88fe27b24796644186e9e&chksm=ea77d260dd005b76c10f75dafc38428b8357150f3fb63bc49a080fb39130d6590ddea61a98b5&scene=21#wechat_redirect)

[防火墙双机热备](http://mp.weixin.qq.com/s?__biz=MzI1OTY2MzMxOQ==&mid=2247493173&idx=1&sn=53975601d927d4a89fe90d741121605b&chksm=ea77d28ddd005b9bdd83dac0f86beab658da494c4078af37d56262933c866fcb0b752afcc4b9&scene=21#wechat_redirect)

[常见的几种网络故障案例分析与解决](http://mp.weixin.qq.com/s?__biz=MzI1OTY2MzMxOQ==&mid=2247493157&idx=1&sn=de0c263f74cb3617629e84062b6e9f45&chksm=ea77d29ddd005b8b6d2264399360cfbbec8739d8f60d3fe6980bc9f79c88cc4656072729ec19&scene=21#wechat_redirect)

[Kubernetes容器之间的通信浅谈](http://mp.weixin.qq.com/s?__biz=MzI1OTY2MzMxOQ==&mid=2247493145&idx=1&sn=c69bd59a40281c2d7e669a639e1a50cd&chksm=ea77d2a1dd005bb78bf499ea58d3b6b138647fc995c71dcfc5acaee00cd80209f43db878fdcd&scene=21#wechat_redirect)

[kube-proxy 如何与 iptables 配合使用](http://mp.weixin.qq.com/s?__biz=MzI1OTY2MzMxOQ==&mid=2247492982&idx=1&sn=2b842536b8cdff23e44e86117e3d940f&chksm=ea77d1cedd0058d82f31248808a4830cbe01077c018a952e3a9034c96badf9140387b6f011d6&scene=21#wechat_redirect)

[完美排查入侵](http://mp.weixin.qq.com/s?__biz=MzI1OTY2MzMxOQ==&mid=2247492931&idx=1&sn=523a985a5200430a7d4c71333efeb1d4&chksm=ea77d1fbdd0058ed83726455c2f16c9a9284530da4ea612a45d1ca1af96cb4e421290171030a&scene=21#wechat_redirect)

[QUIC也不是万能的](http://mp.weixin.qq.com/s?__biz=MzI1OTY2MzMxOQ==&mid=2247491959&idx=1&sn=61058136134e7da6a1ad1b9067eebb95&chksm=ea77d5cfdd005cd9261e3dc0f9689291895f0c9764aa1aa740608c0916405a5b94302f659025&scene=21#wechat_redirect)

[为什么要选择智能网卡？](http://mp.weixin.qq.com/s?__biz=MzI1OTY2MzMxOQ==&mid=2247491828&idx=1&sn=d81f41f6e09fac78ddddc287feabe502&chksm=ea77d44cdd005d5a48dc97e13f644ea24d6e9533625ce8f204d4986b6ba07f328c5574820703&scene=21#wechat_redirect)

[网络排错大讲解~](http://mp.weixin.qq.com/s?__biz=MzI1OTY2MzMxOQ==&mid=2247488574&idx=1&sn=68df1982e9f23ce42457d00ce529b012&chksm=ea742086dd03a9902adb16d2b7648149209fed6c9811c6dd05be5101b42d462cb48e269b6e9d&scene=21#wechat_redirect)

[OVS 和 OVS-DPDK 对比](http://mp.weixin.qq.com/s?__biz=MzI1OTY2MzMxOQ==&mid=2247488294&idx=2&sn=303d9baefa768f887ba36213df793025&chksm=ea74279edd03ae88bc91300a1666066881dd763879b714941120fb9e41e787c557c2930ff5db&scene=21#wechat_redirect)

[微软出品的最新K8S学习指南3.0](http://mp.weixin.qq.com/s?__biz=MzI1OTY2MzMxOQ==&mid=2247488271&idx=1&sn=a52af9aba2ea8bbc5fd4440f85ece458&chksm=ea7427b7dd03aea1dbf106326168333c651b403dd44abf4e652d5a1912377d64becabac5051a&scene=21#wechat_redirect)下载

▼

  

_**_****喜欢，就给我一个****“在看”****_**_

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**10T 技术资源大放送！包括但不限于：云计算、虚拟化、微服务、大数据、网络、**Linux、**Docker、Kubernetes、Python、Go、C/C++、Shell、PPT 等。在公众号内回复「****1024****」**，即可免费获！！****

阅读 1838

​

写留言

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/1TDxR6xkRSFD7ibPKdP9JOqxFxp199B6LLCw80xbTAicxIQPgpsq5ib8kvx7eKfq6erLb1CAT4ZKa3RSnl9IWfELw/300?wx_fmt=png&wxfrom=18)

Linux云计算网络

9分享6

写留言

写留言

**留言**

暂无留言