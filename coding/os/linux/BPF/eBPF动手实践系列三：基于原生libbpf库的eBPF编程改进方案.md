
闻茂泉 深入浅出BPF

 _2024年03月18日 12:09_ _上海_

**01** **欲穷千里目，更上一层楼**

在上一篇文章《[eBPF动手实践系列二：构建基于纯C语言的eBPF项目](https://mp.weixin.qq.com/s?__biz=MzUwOTkwNzQxMg==&mid=2247485351&idx=1&sn=5025af06acf375c6aeb0f03700f7ce6c&scene=21#wechat_redirect)》中，我们初步实现了脱离内核源码进行纯C语言eBPF项目的构建。libbpf库在早期和内核源码结合的比较紧密，如今的libbpf库更加成熟，已经完全脱离内核源码独立发展。

为了更加具体的理解linux内核版本演进和libbpf版本演进的关系，本文在“附录A”中总结了各个内核版本源码示例中所依赖的libbpf库的对应版本信息。

大部分版本的内核获取libbpf版本的方法如下，从libbpf库目录的libbpf.map文件中提取最大的版本号信息。这里的"source"为内核源码所在目录。

```
$ cat ./source/tools/lib/bpf/libbpf.map | grep -oE '^LIBBPF_([0-9.]+)' | sort -rV | head -n1 | cut -d'_' -f2
```

较早版本的内核在./tools/lib/bpf/Makefile文件中直接定义了libbpf的版本信息。

```
$ cat ./source/tools/lib/bpf/Makefile
```

**02** **eBPF编程方案简介**

为了简化 eBPF程序的开发流程，降低开发者在使用 libbpf 库时的入门难度，libbpf-bootstrap 框架应运而生。基于libbpf-bootstrap框架的编程方案是目前网络上看到的最主流编程方案。此外，网络上也偶见比较古老的仅依赖一个bpf_load.c文件的C语言编程方案，这个方案并不需要依赖libbpf库的支持。

主流的C语言实现的eBPF编程方案，大体上就是以下三种，笔者总共将其归纳为3代。

![](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

除了经典的C语言编程方案，一些编程框架还选择使用Python语言，Go语言，或者Rust语言作为用户态加载的实现语言。

**尽管libbpf-bootstrap骨架C语言方案、基于libbpfgo库的go语言方案等已经被大家广泛使用和接受。但笔者认为基于原生libbpf库的eBPF编程方案仍然具备很多独特的优势。以下是原生libbpf库eBPF编程方案的一些独特优势：**

1. **更深的控制和灵活性**：直接使用原生libbpf 库的方案意味着可以与更底层交互，实现更多的控制，定制加载和管理 eBPF 程序和 maps 过程，满足更复杂的需求。
    
2. **更好的学习和理解**：libbpf-bootstrap封装抽象屏蔽了很多细节，直接使用原生libbpf可以对 eBPF 子系统有更深入的理解，有利于开发者对 eBPF 内部工作原理的理解。
    
3. **更细粒度的依赖管理**：直接使用原生libbpf库能够指定依赖的 libbpf 库版本和功能，进而更精细化地管理项目依赖关系。
    
4. **更好的低版本内核适应性**：基于原生libbpf库的方案，在低版本操作系统发行版和低版本内核上可以有更好的兼容性。
    

本文将由浅入深介绍第 2 代原生libbpf库的eBPF编程方案，并提出一种改进思路。

**03** **准备eBPF开发的基础环境**

主流的linux发行版大多是基于rpm包或deb包的包管理系统。不同的包管理系统，初始化eBPF开发环境时所依赖的包，也略有差别。本文将分别进行介绍。

**3.1  rpm包基础环境初始化**

在RPM包发行版环境，需要安装一些编译过程的基础包、编译工具包、库依赖包和头文件依赖包等。我们推荐使用如下一些发行版及其兼容环境：Anolis 8.8、Kylin V10、CentOS  8.5、和 Fedora 39 等。

详细安装步骤如下：

```
$  yum install git make                               # 基础包
```

**3.2  deb包基础环境初始化**

在 DEB 包发行版环境，需要安装一些编译过程的基础包、编译工具包、库依赖包和头文件依赖包等。推荐使用Ubuntu 22.04 或Debian 12 等发行版及其兼容环境。

详细安装步骤如下：

```
$  apt-get update                                     # 更新apt源信息
```

**04** **构建基于原生libbpf库的eBPF项目**

本文的目的是向大家分享一个以第2代 ebpf 编程方案为基础的改进ebpf编译构建方案。本节先用一些篇幅内容，对第2代方案本身的构建编译过程做一些介绍。

libbpf库具有一定的向下兼容能力，可以选择使用截至目前最新的归档版本libbpf-1.3.0来搭建编程环境。以 libbpf-1.3.0版本libbpf库为基础，下文会提供若干实例代码，来剖析ebpf构建原理。完成了基础环境的初始化，就可以开始搭建我们的eBPF项目。所有的代码示例都可以通过如下git项目获取。为了后面访问方便，这里用一个shell变量NATIVE_LIBBPF用来存储工作目录。

```
$ cd ~
```

**4.1  初步构建基于原生libbpf库的eBPF项目**

首先来看一个基于原生libbpf库的第2代eBPF构建实例。ebpf初学者，可以考虑选择跟踪 execve 系统调用产生的事件。

```
$ cd $NATIVE_LIBBPF                                    # 返回工作目录
```

执行trace_execve命令，对编译结果进行验证，完美验证通过。

**4.2  eBPF项目的目录结构解析**

介绍下trace_execve_libbpf130的目录结构。

|   |   |
|---|---|
|**trace_execve_libbpf130目录**|**说明**|
|./|项目用户态代码和主Makefile|
|./progs|项目内核态bpf程序代码|
|./include|项目的业务代码相关的头文件|
|./helpers|非来自于libbpf库的一些helpler文件|
|./tools/lib/bpf/|来自于libbpf-1.3.0/src/|
|./tools/include/|来自于libbpf-1.3.0/include/|
|./tools/build/|项目构建时一些feature探测代码|
|./tools/scripts/|项目Makefile所依赖的一些功能函数|

再介绍下本项目trace_execve_libbpf130和libbpf-1.3.0库的对应关系。下载libbpf-1.3.0库解压后，使用diff命令进行目录对比。

1. 目录native_libbpf_guide/trace_execve_libbpf130/tools/lib/bpf/内容，除Makefile内容外都来自目录~/libbpf-1.3.0/src/。
    
2. 目录native_libbpf_guide/trace_execve_libbpf130/tools/include/来自目录~/libbpf-1.3.0/include/，所有内容都完全一致。
    
3. 除以上两部分来自libbpf-1.3.0库以外的文件，其余都由本项目原创贡献。
    

```
$ cd ~
```

在这个项目中添加ebpf的代码，可以遵循这样的目录结构。用户态加载文件放到根目录下，内核态bpf文件放到progs目录下，用户态和内核态公共的头文件放到include目录下。

```
$ cd $NATIVE_LIBBPF                                    # 返回工作目录
```

**4.3  eBPF项目的Makefile解析**

```
$ cd $NATIVE_LIBBPF                                    # 返回工作目录
```

trace_execve_libbpf130项目有4个Makefile，分别如下：

1. ./Makefile是主文件，用于生成用户态eBPF程序trace_execve。
    
2. ./progs/Makefile 用于生成内核态BPF程序trace_execve.bpf.o。
    
3. ./tools/lib/bpf/Makefile 用于生成libbpf.a静态库。
    
4. ./tools/build/feature/Makefile 用于一些feature的探测。
    

在项目空间的根目录运行make命令进行项目构建时，会首先执行Makefile文件。在Makefile文件中会通过make的-C选项间接触发progs/Makefile和tools/lib/bpf/Makefile的执行。

感兴趣的同学可以通过上一章节中提到的make --debug=v,m SHELL="bash -x" 命令逐步debug这些makefile的执行过程。

下文重点分析下编译过程的一些编译参数，让我们加深对eBPF构建过程的理解。

**4.4   C语言编译器的目录搜索选项**

在开始分析eBPF程序的编译参数之前，先要简单说一下C语言编译器（gcc/clang）的目录搜索选项。C语言的头文件都需要按照目录搜索选项的指引，才能正确找到它所在位置。

除了日常我们熟知的-I选项，clang/gcc的目录搜索选项还有很多，它们优先级的顺序依次如下：

1. 头文件引用方式include "myheader.h"，则在当前文件所在目录查找myheader.h头文件。
    
2. 头文件引用方式include "myheader.h"，如果有-iquote mydir选项，则在mydir目录查找myheader.h头文件。
    
3. 头文件引用方式include <myheader.h>，如果有-I mydir选项，则在mydir目录查找myheader.h头文件。
    
4. 头文件引用方式include <myheader.h>，如果有-isystem mydir选项，则在mydir目录查找myheader.h头文件。
    
5. 头文件引用方式include <myheader.h>，继续在标准系统目录（Standard system directories）查找myheader.h头文件。标准系统目录是指/usr/lib64/clang/15.0.7/include 、/usr/local/include 和/usr/include。
    
6. 头文件引用方式include <myheader.h>，如果有-idirafter mydir选项，则在mydir目录查找myheader.h头文件。
    

**4.5  内核态bpf程序编译参数解析**

内核态bpf程序trace_execve.bpf.o文件，是由 bpf 文件trace_execve.bpf.c使用clang命令编译产生。trace_execve.bpf.c文件的头文件依赖如下。

```
$ cat progs/trace_execve.bpf.c
```

从前面项目构建过程中，可以提取出完整的内核态bpf程序的编译命令。

```
$ clang -iquote ./../include/ -iquote ./../helpers -I./../tools/lib/ -I./../tools/include/uapi -idirafter /usr/lib64/clang/15.0.7/include \
```

下面对一些关键环节做一些解析：

1. 头文件vmlinux.h由bpftool工具在编译时动态生成，vmlinux.h包含了绝大多数bpf程序的内核态和用户态(uapi)依赖。通过编译选项-I./../tools/lib/可以搜索到vmlinux.h头文件。
    
2. 通过-I./../tools/lib/编译选项，可以在./tools/lib/目录下的bpf子目录中查找到bpf_helpers.h和bpf_tracing.h头文件，这些头文件都是对vmlinux.h头文件内核态依赖的补充。
    
3. 通过-iquote ./../include/编译选项，可以在./include/目录中查找到trace_execve.h和common.h头文件。
    
4. 在上面这些头文件依赖的预处理过程中，会依赖一些宏变量来决定预处理的展开逻辑。上面编译命令中的宏就是起这些作用，-DENABLE_ATOMICS_TESTS -D__KERNEL__ -D__BPF_TRACING__ -D__TARGET_ARCH_x86。比如在bpf_tracing.h头文件中，就有#if defined(__TARGET_ARCH_x86)的宏判断语句，来决定预处理展开逻辑走x86分支。
    
5. 编译选项-target bpf，指示Clang将代码生成为针对eBPF目标的目标代码。编译选项-mcpu=v3，指示Clang生成针对v3版本的eBPF处理器的目标代码。编译选项-mlittle-endian：指示Clang生成适用于小端序处理器的目标代码。
    
6. 通过-I./../tools/include/uapi编译选项，可以在./tools/include/uapi/目录下的linux子目录中查找到bpf.h头文件。同时kernel-headers包引入的/usr/include/linux/目录下也有bpf.h，./tools/include/uapi下的bpf.h优先级会覆盖它。此外，目录./tools/include/uapi/linux下的头文件和vmlinux.h头文件存在一定的重叠，通常情况下同时加载会出现编译冲突。如果在一些简单的 ebpf 使用场景，可以使用<linux/bpf.h>替代<vmlinux.h>。
    

**4.6  用户态加载程序编译参数解析**

用户态eBPF程序trace_execve文件，是由源文件trace_execve.c文件使用gcc命令编译。trace_execve.c文件的头文件依赖如下。

```
$ cat trace_execve.c
```

从前面项目构建过程中，也可以提取出完整的用户态程序的编译命令。

```
gcc -iquote ./helpers/ -iquote ./include/ -I./tools/lib/ -I./tools/include/ -g -c -o trace_execve.o trace_execve.c
```

1. 通过-I./tools/include/编译选项，可以在./tools/include/目录下的linux子目录中查找到<linux/ring_buffer.h>头文件。
    
2. 通过-I./tools/lib/编译选项，可以在./tools/lib/目录下的bpf子目录中查找到<bpf/libbpf.h>头文件。在一些古老的代码示例中，有<libbpf.h>这样使用头文件的用法，目前最新的ebpf项目实例，都会将libbpf库的libbpf.h以及同目录的头文件都放到bpf子目录下，因此推荐统一使用<bpf/libbpf.h>的用法。
    
3. 通过-iquote ./include/编译选项，可以在./include/目录中查找到trace_execve.h和common.h头文件。
    
4. 其他头文件都可以在由kerne-headers包提供的标准系统目录（Standard system directories）的/usr/include/目录及子目录中查找到。所以，<linux/perf_event.h>最终会在/usr/include/linux/perf_event.h位置被查找到。可以看出同样是<linux/xxx.h>形式的头文件，<linux/perf_event.h>和<linux/ring_buffer.h>却在两个完全不同的搜索路径查找到。
    

**4.7  libbpf.a静态库编译解析**

关于libbpf.a静态库的编译过程，上一篇文章已经有所介绍。这里仅再次强调下，在本项目中，我们完全实现了libbpf库的自主可控，可控源代码，可控编译构建过程。这至少给我们带来如下两方面好处：

1. 对于一些ebpf的资深人士，可以自主修改libbpf库中不尽如人意的地方，实现满足自己业务需求的优化。
    
2. 对于一些ebpf的初学者，完全可以在libbpf库中任意感兴趣的地方，通过插入printf或其他断点方式，深入学习libbpf库的原理。
    

**05**

**改进基于原生libbpf库的eBPF项目构建**

**5.1  传统方案美中不足**

在上文中，我们初步实现了基于libbpf库的第 2 代 eBPF项目的构建。但截止到目前，此方案还有一个明显的缺陷。让我们继续上一篇的案例来分析，在搭建完开发环境后执行如下步骤。

```
$ cd $NATIVE_LIBBPF                                    # 返回工作目录
```

从实验结果可以看出，当我们把bpf目标文件trace_execve.bpf.o改名为trace_execve.bpf.o.bak后，trace_execve程序执行会报错，提示读取trace_execve.bpf.o文件不存在。而当我们再次将备份后的bpf目标文件trace_execve.bpf.o.bak改回原名trace_execve.bpf.o后，重新执行trace_execve程序又一切正常了。这说明，当前方案构建后，需要将trace_execve程序和bpf目标文件trace_execve.bpf.o这一组文件一起进行分发，才能正常执行。这给我们在工程的实现上带来了很大的挑战。

为了解决上面提到的问题，第 3 代 ebpf 编程方案 libbpf-bootstrap框架发明了skeleton骨架，即使用*.skel.h头文件的方式，将bpf目标文件trace_execve.bpf.o的内容编译进trace_execve程序。这样后续只需分发trace_execve二进制程序文件即可。

如果不依赖libbpf-bootstrap编程框架，继续仅依赖 libbpf 库是否可以做到这一点呢？答案是可以的，本文独辟蹊径，给大家分享一个使用hexdump命令轻松实现*.skel.h头文件的方式。

**5.2  使用hexdump生成skel.h头文件**

简单归纳一下使用libbpf-bootstrap框架编程过程中的构建步骤。

|   |   |   |
|---|---|---|
|**步骤**|**libbpf-bootstrap框架构建**|**可改进机会点**|
|1|bpftool btf dump file vmlinux format c > vmlinux.h||
|2|clang -O2 -target bpf -c trace_execve.bpf.c -o trace_execve.bpf.o||
|3|bpftool gen skeleton trace_execve.bpf.o > trace_execve.skel.h|此步骤用hexdump替换bpftool|
|4|gcc -o trace_execve trace_execve.c -lbpf  -lelf|此步骤更改加载函数为libbpf标准函数|

分析libbpf-bootstrap编程框架的实现原理，可以了解到。在第3步会依靠bpftool工具将trace_execve.bpf.o这个目标文件转换成十六进制格式的文本，并将这个文本内容作为trace_execve.skel.h头文件中的一个变量的值，最后还需要让trace_execve.c用户态加载文件包含这个trace_execve.skel.h头文件。这其中将bpf目标文件转换成十六进制文本并生成skel.h头文件的过程最为关键。

libbpf-bootstrap编程框架非常成熟，但方案使用中必须遵循他的一些规则，比如头文件trace_execve.skel.h的命令必须包含程序的关键词trace_execve，再比如加载函数trace_execve_bpf__load()也必须包含程序的关键词trace_execve。如何能不依赖这个规范，实现一个更加轻量级的编程方案呢？这让我们想到了hexdump命令，可以用它替换bpftool工具，并且生成符合自己期望的头文件。

```
$ hexdump -v -e '"\\\x" 1/1 "%02x"' trace_execve.bpf.o > trace_execve.hex
```

**5.3  深入构建基于原生libbpf库的eBPF项目**

下面我们就尝试依靠hexdump命令实现一个单一可执行文件的解决方案。开始体验我们基于第 2 代编程方案改进的eBPF项目，进入项目代码。

```
$ cd $NATIVE_LIBBPF                                    # 返回工作目录
```

分别执行trace_execve和probe_execve两个命令，对编译结果进行验证，均完美验证通过。这里我们在trace_execve实例基础上又增加了一个probe_execve实例，说明hexdump_skel_libbpf130项目是支持多实例编译的。

下面我们来验证下本文开头的情况，看看没有了bpf目标文件时的情形。

```
$ cd $NATIVE_LIBBPF                                    # 返回工作目录
```

从运行结果看，虽然删除了两个bpf目标文件trace_execve.bpf.o和probe_execve.bpf.o，仅仅依靠trace_execve和probe_execve两个文件即可成功执行。可以再尝试将trace_execve 可执行文件拷贝到其他目录，结果依然可行。

**5.4  改进的eBPF项目Makefile解析**

hexdump_skel_libbpf130项目也是同样的4个Makefile，其中将bpf目标文件编译到用户态加载进程中的环节主要在项目的主Makefile中实现。还是老办法获取make构建的详细过程。

```
$ cd $NATIVE_LIBBPF                                    # 返回工作目录
```

对于构建日志的分析可以参考前面文章，我们把关键环节提取出来。

```
$ cat make.log | grep -n "Considering target file"
```

从关键构建步骤中，我们可以了解到：

1. probe_execve和trace_execve两个target都是all目标的下级目标，并且probe_execve和trace_execve是串行的。这个里隐含的一个意思是，当trace_execve开始构建的时候，probe_execve已经完全构建完毕，probe_execve这个最终可执行文件已经生成完毕。此时，probe_execve构建过程中所依赖的所有中间文件都不再需要了。所以，probe_execve和trace_execve构建过程中依赖的中间文件是可以重名的。
    
2. tools/lib/bpf/libbpf.a和helpers/uprobe_helper.o已经提前编译好了，就不再做过多的说明了。最终的用户态可执行加载程序的主要依赖链条如下。
    

```
trace_execve
```

再看一下主Makefile的源码，为了实现以上的目标依赖，我们连用了5个静态模式规则（Static Pattern Rules）。

```
$(HELPER_OBJECTS): %.o:%.c
```

其中任何一个静态模式规则的目标集合，都是通过项目根目录下*.c文件的集合，进行局部字符串替换获得。

```
SOURCES := $(wildcard *.c)
```

**5.5  从file到memory实现读取elf的转变**

本方案的主要逻辑是在主Makefile中实现，但也需要c代码中做一些调整。bpf文件trace_execve.bpf.c并不需要任何修改，只需要在用户态加载程序trace_execve.c做一些调整。

传统的读取bpf目标文件方式，相关代码如下：

```
char filename[256] = "progs/trace_execve.bpf.o";
```

改进后的读取memory方式，相关代码如下：

```
#include "skeleton.skel.h"
```

很明显libbpf库提供了bpf_object__open_file（bpf_object__open）和bpf_object__open_mem两个函数用于读取elf格式的bpf目标文件trace_execve.bpf.o。区别是bpf_object__open_file是在trace_execve运行时，再去读取trace_execve.bpf.o文件内容，而bpf_object__open_mem是在编译时，已经把elf内容编译进trace_execve程序。至于bpf_object__open函数在libbpf库的libbpf.c文件中是对bpf_object__open_file函数的封装。

这两个libbpf库函数，最终都是调用elf标准库函数实现了相关功能，具体代码实现是在libbpf库的libbpf.c文件中的bpf_object__elf_init函数中，代码如下：

```
static int bpf_object__elf_init(struct bpf_object *obj){
```

可以看出，bpf_object__open_mem函数的最终实现是elf的elf_memory函数，bpf_object__open_file函数的最终实现是elf的elf_begin函数。

**5.6  原生libbpf库与libbpf-bootstrap的若干区别**

相比较第3代的 libbpf-bootstrap框架方案和第2代的传统libbpf库方案，使用hexdump命令的原生libbpf库第 2 代改进方案方案在实现方法上，有一些独特的优势。

这里将这三种方案的主要区别归纳总结如下：

![](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

这里补充下，trace_execve_bpf__open()函数的实现，也是间接通过libbpf库的bpf_object__open_skeleton()函数，最终也调用了bpf_object__open_mem()函数。

**5.7  使用attach_tracepoint替代attach**

在ebpf用户态程序的加载过程中，有一个attach的步骤。细心的读者应该已经发现了，在trace_execve_libbpf130项目中，我们使用的是bpf_program__attach()函数实现的静态探针点的attach。而在hexdump_skel_libbpf130项目中，我们使用的却是bpf_program__attach_tracepoint()函数实现的静态探针点的attach。区别是bpf_program__attach_tracepoint函数的参数中会指定静态探针点的具体信息，而bpf_program__attach不用指定静态探针点的信息。进一步阅读bpf_program__attach函数的源代码可以了解到，它是依靠内核态的bpf目标文件中SEC的节名称信息来获取和确定静态探针点的信息的。总结这两种方法如下：

|   |   |   |
|---|---|---|
||**trace_execve.c中相关代码**|**trace_execve.bpf.c中相关代码**|
|attach方案A|bpf_program__attach(bpf_prog)|SEC("tracepoint/syscalls/sys_enter_execve")|
|attach方案B|bpf_program__attach_tracepoint(bpf_prog, "syscalls", "sys_enter_execve")|SEC("tracepoint")|

很明显，在trace_execve.c和trace_execve.bpf.c的代码中，只要有一处设置静态探针点即可。如果两处都设置，而且两处设置的静态探针点信息冲突的情况下，会以用户态的bpf_program__attach_tracepoint函数设置的信息为准。

libbpf库中的bpf_link__destroy()函数是负责对attach函数生成的link进行销毁的函数。attach和destroy的过程实际上就是对内核静态探针点开启和关闭的过程。

在这里特别推荐使用方案B中的bpf_program__attach_tracepoint替代方案A中的bpf_program__attach方法，这样方便我们在用户态代码中灵活的开关ebpf的采集。除了专门用于静态探针点的bpf_program__attach_tracepoint()函数，还有适用于其他类型的专用的attach函数，例如bpf_program__attach_kprobe()、bpf_program__attach_kprobe()、bpf_program__attach_uprobe()和bpf_program__attach_usdt()等。

**5.8  使用by_name替代by_title**

在稍早一些libbpf库中提供2个函数用于获取bpf progam 类型数据，分别是bpf_object__find_program_by_name()函数和bpf_object__find_program_by_title()函数。以trace_execve_libbpf130项目的 bpf代码为例。

```
SEC("tracepoint/syscalls/sys_enter_execve")
```

其中tracepoint/syscalls/sys_enter_execve这个字符串就称为title，trace_execve_enter这个函数名就称为name。结合上文的结论，后续推荐bpf内核态代码中都使用SEC("tracepoint")的语法格式，那么使用by_title函数将不再能做出区分。因此这里特别推荐大家今后使用by_name的函数替代by_titile的函数。而且，在最新版的libbpf库中，也彻底移除了bpf_object__find_program_by_title()函数。

**06**

**基于原生libbpf库改进方案构建USDT和Uprobe项目**

基于hexdump命令的改进型原生libbpf库编程方案不但在内核态跟踪诊断上表现完美，在用户态应用进程的跟踪诊断上依然可以表现得非常出色。本节内容将在上文的基础上，继续分析如何使用原生libbpf库开发和构建USDT和Uprobe项目。

**6.1  用户态模拟程序**

用户态应用程序的ebpf，还需要准备一个模拟程序。尤其是针对USDT类型，还需要在模拟程序中进行静态打点。本小节将提供一个如何打USDT跟踪点的实例。

```
$ cd $NATIVE_LIBBPF                                    # 返回工作目录
```

执行完以上步骤，就启动了用户态模拟程序umark，后续即可通过USDT和Uprobe方式，追踪umark进程的运行情况。

下面初步对umark模拟程序的代码做一些介绍。

```
$ ls 
```

其中func_uprobe1和func_uprobe2是两个C语言函数用于下文的uprobe跟踪实例的追踪。DTRACE_PROBE1和DTRACE_PROBE2是两个宏函数，用于在umark.c程序中打USDT的静态跟踪点。最多支持传入12个跟踪点参数，即DTRACE_PROBE1、DTRACE_PROBE2，一直到DTRACE_PROBE12。probe1和probe2是这个静态跟踪点的name，groupa和groupb是跟踪点name的分组名，可以省略。

DTRACE_PROBE1宏函数定义在std.h头文件内，需要提前安装头文件所在包。

在rpm包环境，sdt.h头文件属于systemtap-sdt-devel这个rpm包。

```
$ find /usr/include/ -name sdt.h
```

在deb包环境，sdt.h头文件属于systemtap-sdt-dev这个deb包。

```
$ find /usr/include/ -name sdt.h
```

令人欣慰的是，这个sdt.h头文件并无太多额外依赖，简单修改后，可以独立维护。于是，我们可以将其拷贝到本项目根目录。并将<sys/sdt.h>的头文件引用方式改为"sdt.h"。

**6.2  构建基于libbpf库的USDT和Uprobe项目**

下面我们就进一步介绍下使用第 2 代改进编程方案的ebpf跟踪用户态进程的解决方案。开始体验我们的eBPF项目trace_user_libbpf130，进入项目代码。

```
$ cd $NATIVE_LIBBPF                                    # 返回工作目录
```

分别执行uprobe_test和usdt_test两个命令，对编译结果进行验证，均完美验证通过。

trace_user_libbpf130项目的构建和编译过程与前面项目hexdump_skel_libbpf130无太多差异，不再做过多赘述。下文将着重对本项目中USDT和Uprobe的相关C语言源码进行解析。

**6.3  USDT代码解析**

trace_user_libbpf130项目中的USDT部分，开启了2个usdt静态探针点的跟踪，这2个静态探针点分别是probe1和probe2。

第一个静态探针点实例，选择在attach时，通过bpf_program__attach_usdt函数的参数指定静态探针点的相关信息。包括跟踪的进程信息"/usr/bin/umark"，usdt组名信息"groupa"，usdt名称信息"probe1"等，代码如下：

```
bpf_program__attach_usdt(bpf_prog1, -1, "/usr/bin/umark", "groupa", "probe1", NULL);
```

第二个静态探针点实例，选择在bpf目标文件中，通过SEC宏的方式指定静态探针点的相关信息。包括跟踪的进程信息"/usr/bin/umark"，usdt组名信息"groupb"，usdt名称信息"probe2"等，代码如下：

```
SEC("usdt//usr/bin/umark:groupb:probe2")
```

**6.4  BPF_USDT宏函数解析**

目前主流的USDT类型的ebpf代码实例，在bpf目标文件中都使用BPF_USDT宏函数来定义ebpf的处理函数，例如本项目实例中。

```
int BPF_USDT(usdt_probe1, int x)
```

在这里，宏函数BPF_USDT的第1个参数"usdt_probe1"才是真正的函数名，也就是前文所述by_name的name信息。宏函数的第2个参数"int x"才是usdt_probe1函数的第一个参数，依次类推。

各种USDT类型的ebpf代码实例中，很少见到对这个宏函数BPF_USDT原理的分析。此处，我们借助第二个USDT静态探针点在bpf目标文件中的使用来解析它。代码实例的关键部分如下：

```
int usdt_probe2(struct pt_regs *ctx);
```

这4行代码，前两行是函数声明，后两行是函数定义。usdt_probe2函数内部调用了____usdt_probe2函数。一些代码解读：

1. always_inline，意味着无论优化设置如何，编译器都应该始终将这个函数内联到任何调用它的地方。
    
2. typeof(usdt_probe2(0)) 用于确定 usdt_probe2 的返回类型，从而确保 ____usdt_probe2 与 usdt_probe2 有相同的返回类型。
    
3. ({ long _x; bpf_usdt_arg(ctx, 0, &_x); (void *)_x; }) 这个复合语句用于获取USDT探针的参数值。
    
4. 使用 bpf_usdt_arg 辅助函数来获取探针的第一个参数，并将其存储到局部变量 _x 中。再将 _x 强制转换为 void * 类型并传递给 ____usdt_probe2 函数。同样的操作也对第二个参数 y 进行。
    

特别强调一下bpf_usdt_arg辅助函数来自于usdt.bpf.h头文件，但本项目有2个usdt.bpf.h头文件，其中一个在libbpf库中，另外一个在./helpers/目录下，helpers 目录下的是经过本项目改造过的。此示例中生效的是./helpers/目录下的。

```
$ cd $NATIVE_LIBBPF                                    # 返回工作目录
```

**6.5  Uprobe代码解析**

trace_user_libbpf130项目中的Uprobe部分，开启了2个uprobe类型探针点的跟踪，这2个uprobe探针点分别是probe1和probe2。

第一个uprobe探针点实例，选择在attach时，通过bpf_program__attach_uprobe函数的参数指定uprobe探针点的相关信息。包括uprobe的类型（0表示函数进入时，1表示函数返回时），跟踪的进程信息"/usr/bin/umark"，被跟踪的函数在进程中的偏移量 func_off1等。需要提前通过get_elf_func_offset()函数计算出这个偏移量，此函数定义在了helpers/uprobe_helper.c文件内。相关代码如下：

```
func_off1 = get_elf_func_offset("/usr/bin/umark", "func_uprobe1");
```

第二个uprobe探针点实例，选择在bpf目标文件中，通过SEC宏的方式指定uprobe探针点的相关信息。包括跟踪的进程信息"/usr/bin/umark"，被跟踪的应用进程中的函数"func_uprobe2"等。此种情况，libbpf库会自动计算这个偏移量。代码如下：

```
SEC("uprobe//usr/bin/umark:func_uprobe2")
```

**6.6  BPF_KPROBE宏函数解析**

目前主流的Uprobe类型的ebpf代码实例，在bpf目标文件中都使用BPF_KPROBE宏函数来定义ebpf的处理函数，例如本项目实例中。

```
int BPF_KPROBE(user_probe1, unsigned long long int x)
```

在这里，宏函数BPF_KPROBE的第1个参数"user_probe1"才是真正的函数名，也就是前文所述by_name的name信息。宏函数的第2个参数"unsigned long long int x"才是user_probe1函数的第一个参数，依次类推。

各种Uprobe类型的ebpf代码实例中，也同样很少见到对这个宏函数BPF_KPROBE原理的分析。此处，我们借助第二个Uprobe探针点在bpf目标文件中的使用来解析它。关键的代码实例如下：

```
long user_probe2(struct pt_regs *ctx);
```

这4行代码，前两行是函数声明，后两行是函数定义。user_probe2函数内部调用了____user_probe2函数。一些代码解读：

1. inline typeof(user_probe2(0)) ____user_probe2(struct pt_regs *ctx, unsigned long long int x, unsigned long long int y);  这是内联函数____user_probe2的声明。
    
2. typeof(user_probe2(0))用于确定____user_probe2函数的返回类型，保证与user_probe2函数的返回类型一致。
    
3. typeof(user_probe2(0)) user_probe2(struct pt_regs *ctx) { return ____user_probe2(ctx, (unsigned long long int)PT_REGS_PARM1(ctx), (unsigned long long int)PT_REGS_PARM2(ctx)); }  这是user_probe2函数的定义。它使用PT_REGS_PARM1(ctx)和PT_REGS_PARM2(ctx)宏来获取用户空间探针传递给eBPF程序的前两个参数。
    

如果对于以上的代码解读如果还有不明白的地方，可以尝试问问GPT。

**07**

**技术交流**

本文为eBPF动手实践系列的第三篇，我们实现了基于libbpf库的纯C语言eBPF项目的构建。

下一篇我们会进一步深入到 ebpf 程序内部的代码逻辑追本溯源，探寻ebpf程序的核心逻辑。

欢迎有想法或者有问题的同学，加群交流eBPF技术以及工程实践。

- SREWorks数智运维工程群（**钉钉群号：35853026**）
    
- 跟踪诊断技术 SIG 开发者&用户群（**钉钉群号：33304007**）
    

## **附录**

### **附录A**

各个内核版本源码示例所依赖的libbpf库的对应版本信息。

|   |   |   |
|---|---|---|
|kernel版本|libbpf版本|备注|
|kernel-6.5|LIBBPF_1.3.0||
|kernel-6.4|LIBBPF_1.2.0||
|kernel-6.3|LIBBPF_1.2.0||
|kernel-6.2|LIBBPF_1.1.0||
|kernel-6.1|LIBBPF_1.0.0||
|kernel-6.0|LIBBPF_1.0.0||
|kernel-5.19|LIBBPF_1.0.0|开始有usdt.bpf.h文件|
|kernel-5.18|LIBBPF_0.8.0||
|kernel-5.17|LIBBPF_0.7.0||
|kernel-5.16|LIBBPF_0.6.0||
|kernel-5.15|LIBBPF_0.5.0||
|kernel-5.14|LIBBPF_0.5.0||
|kernel-5.13|LIBBPF_0.4.0||
|kernel-5.12|LIBBPF_0.3.0||
|kernel-5.11|LIBBPF_0.3.0||
|kernel-5.10|LIBBPF_0.2.0||
|kernel-5.9|LIBBPF_0.1.0||
|kernel-5.8|LIBBPF_0.0.9||
|kernel-5.7|LIBBPF_0.0.8||
|kernel-5.6|LIBBPF_0.0.7||
|kernel-5.5|LIBBPF_0.0.6||
|kernel-5.4|LIBBPF_0.0.5||
|kernel-5.3|LIBBPF_0.0.4||
|kernel-5.2|LIBBPF_0.0.3||
|kernel-5.1|LIBBPF_0.0.2||
|kernel-5.0|LIBBPF_0.0.1||
|kernel-4.19|LIBBPF_0.0.1||
|kernel-4.18|LIBBPF_0.0.1||
|kernel-4.9|LIBBPF_0.0.1||

/ END /

**更多推荐**

[![](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)](https://mp.weixin.qq.com/s?__biz=Mzg4MzgxNDk2OA==&mid=2247488689&idx=1&sn=5caacb9017d20a0e19d2e4fbe8bd38e7&chksm=cf40f3e6f8377af09fcd89e4c954129abad0fce08b6f9821e13b68db65fdda1899258b57b6bb&scene=21&cur_album_id=2629420659357483011#wechat_redirect)

[![](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)](https://mp.weixin.qq.com/s?__biz=Mzg4MzgxNDk2OA==&mid=2247488312&idx=1&sn=73a92f6f5eb5d6ca266b66033c3bfdd0&chksm=cf40f46ff8377d791342ff49eb1262059996b4e2cd98ba7f99212e3af866305d9f3a9677107a&token=1384723228&lang=zh_CN&scene=21#wechat_redirect)

[![](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)](https://mp.weixin.qq.com/s?__biz=Mzg4MzgxNDk2OA==&mid=2247492170&idx=1&sn=71efb188901359ba2c928750d6d88be7&chksm=cf43051df8348c0bc1b3faa5610ef7f386afa118aebb9741e4aa896a12df1404c295b11857d2&scene=21&cur_album_id=2629420659357483011#wechat_redirect)

阿里云大数据计算&机器学习推出免费试用活动，其中包含Maxcompute、Hologres、实时计算Flink版、机器学习PAI等多款热门产品，点击「**阅读原文**」了解详细试用规则，一键参与试用。  

  

阅读原文

阅读 866

​