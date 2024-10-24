
深入浅出BPF _2024年04月09日 19:22_ _上海_

以下文章来源于云原生指北 ，作者张晓辉Addo

去年学习 eBPF，分享过 几篇 eBPF 方面的学习笔记\[1\]，都是面向 eBPF 的应用。为了准备下一篇文章，这次决定从 Linux 源码入手，深入了解 eBPF 的工作原理。因此这篇又是一篇学习笔记，假如你对 eBPF 的工作原理也感兴趣，不如跟随我的脚步一起。文章中若有任何问题，请不吝赐教。

这里不会再对 eBPF 进行过多的介绍，可以参考我的另一篇 使用 eBPF 技术实现更快的网络数据包传输\[2\]，结合 追踪 Kubernetes 中的数据包\[3\] 可以了解 eBPF 的基本内容以及其在网络加速方面的应用。

接下来我们还是使用 eBPF sockops\[4\] 中的程序 bpf_sockops\[5\] 为例， 配合 Linux v6.8\[6\] 源码探索 eBPF 的工作原理。

**注：由于公众号排版问题阅读可能不友好，可以点击阅读原文跳转到博客阅读。**

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/tMghG0NOfxcBs8icbCrnMQQnxTNDBvTiavDumap7U6PezNsDgmjlJ8cHibibTbKaoZIG60CVXyMMoFptCjia2O0P8Ig/640?wx_fmt=png&from=appmsg&wxfrom=13)

## BPF 程序操作

在 load.sh\[7\] 脚本中，完成了程序的加载和挂载操作，下面的命令使用 bpftool\[8\] 分别完成 BPF 程序的加载和挂载。

`#load   sudo bpftool prog load bpf_sockops.o "/sys/fs/bpf/bpf_sockop"   #attach   sudo bpftool cgroup attach "/sys/fs/cgroup/unified/" sock_ops pinned "/sys/fs/bpf/bpf_sockop"   `

这里 bpftool 是对内核函数 `bpf()` 封装的命令行工具，用于管理和操作 BPF 程序与 Map。

### 加载

`sudo bpftool prog load bpf_sockops.o "/sys/fs/bpf/bpf_sockop"   `

命令 `bpftool prog load` 将 `bpf_sockops.o` 加载到路径 `/sys/fs/bpf/bpf_sockop` 中。

bpftool 对 BPF 程序的加载是由调用 `bpf()`\[9\] 指定命令 `BPF_PROG_LOAD` 并传入 加载选项`bpf_prog_load_opts`\[10\] 来完成的：

`syscall(__NR_bpf, BPF_PROG_LOAD, &attr, sizeof(attr))   `

- syscall bpf()\[11\] bpf 系统函数

- bpf_prog_load\[13\] 为程序分配内存、初始化、检查证书、运行 verifier、创建文件描述符（fd）等

- \_\_sys_bpf\[12\] 执行 bpf 命令 BPF_PROG_LOAD

加载成功后的程序，然后就可以进行挂载了。

### 挂载

`sudo bpftool cgroup attach "/sys/fs/cgroup/unified/" sock_ops pinned "/sys/fs/bpf/bpf_sockop"   `

命令 `bpftool cgroup attach` 将加载（pin 到文件系统中）的程序 `/sys/fs/bpf/bpf_sockop` 挂载到 cgroup `/sys/fs/cgroup/unified/`，挂载的类型为 `sock_ops`。这个 `sock_ops` 是 bpftool 所使用的库 `libbpf` 定义，也被是 ELF 部件名，对应着 BPF 程序类型\[14\] `BPF_PROG_TYPE_SOCK_OPS`，挂载类型\[15\] 为 `BPF_CGROUP_SOCK_OPS`。

> 在 eBPF 编程中，ELF（Executable and Linkable Format）文件用于存储编译后的 eBPF 程序和相关数据。ELF 文件由多个部分（sections）组成，每个部分包含不同类型的信息，比如程序代码、符号表、调试信息等。

libbpf 类型 `sock_ops` => BPF 程序类型 `BPF_PROG_TYPE_SOCK_OPS` => 挂载类型 `BPF_CGROUP_SOCK_OPS`，对应到程序 `bpf_sockops.c` 中部件名（`__section`）为 `sockops` 的代码块。

关于 `sock_ops` 挂载点:

> `sock_ops` 通常指的是在 Linux 内核中处理套接字操作的一系列函数和操作。
>
> `sock_ops` 具体可以包括一系列的操作，如创建套接字、绑定套接字到特定地址和端口、监听来自其他套接字的连接请求、接受连接请求、发送和接收数据、以及关闭套接字等。这些操作通常通过一组预定义的 API 来提供，例如 POSIX 套接字 API，它定义了一系列函数，如 `socket()`、`bind()`、`listen()`、`accept()`、`send()`、`recv()` 和 `close()` 等，供应用程序调用。

这次 bpftool 是通过 `bpf()` 执行执行 `BPF_PROG_ATTACH` 并传入 挂载选项 `bpf_prog_attach_opts`\[16\] 来完成的。

`syscall(__NR_bpf, BPF_PROG_ATTACH, &attr, sizeof(attr))   `

- syscall bpf()\[17\] bpf 系统函数

- cgroup_bpf_prog_attach\[19\]

- \_\_cgroup_bpf_attach\[21\]

- bpf_prog_put\[22\] 检查 cgroup 上是否存在相同**挂载类型**的程序，如果存在，则进行替换。

- static_branch_inc\[23\] 如果不存在，则将 `cgroup_bpf_enabled_key` 计数器中，该**挂载类型**的计数 +1。

- cgroup_bpf_prog_attach\[20\]

- bpf_prog_attach\[18\]

> `cgroup_bpf_enabled_key` 特定类型 cgroup BPF 程序的计数器。
>
> !!! 在运行时，会用到该计数器。

到此，我们已经成功将程序挂载到 cgroup 的 sock_ops 上。

## 套接字操作 sock_ops

套接字的操作很多，这里以连接建立过程中服务端 `accept` 操作为例。
!\[\[Pasted image 20240920144340.png\]\]
!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

依然是从系统调用 `accept` 开始。

- accept\[24\]

- do_accept\[26\] 此处 `ops->accept()` 中的 ops 对应着 proto_ops inet_stream_ops\[27\] **有状态的 socket（如 TCP）** 的相关操作

- inet_accept\[29\]  `sk1->sk_prot->accept()` 这里的 `sk_prot` 提供了 TCP 协议 `proto tcp_prot`\[30\] 的具体操作

- tcp_v4_rcv\[34\] 此时第一次握手刚开始，sock（套接字在内核协议栈这层的体现） 的状态还是 `TCP_LISTEN`

- tcp_v4_do_rcv\[35\] 在连接成功建立前，每次握手都会对状态进行处理。

- tcp_init_transfer\[37\] sock 的状态被设置为 `BPF_SOCK_OPS_PASSIVE_ESTABLISHED_CB`，开始进行数据传输。

- BPF_CGROUP_RUN_PROG_SOCK_OPS\[39\] 执行挂载类型为 `BPF_CGROUP_SOCK_OPS` 的 BPF 程序。

- bpf_skops_established\[38\]

- tcp_rcv_state_process\[36\] 我们直接看最后一次握手，也就是**收到客户端的 ACK**，完成与客户端连接的建立。

- tcp_prot.accept\[31\]

- inet_csk_accept 开始处理三次握手，调用 TCP 协议的实现来处理。inet_init\[32\] 注册了 `IPPROTO_TCP` 也就是 TCP 协议的实现，也就是 net_protocol tcp_protocol\[33\]，其 `handler` 为 `tcp_v4_rcv`。

- inet_stream_ops.accept\[28\]

- \_\_sys_accept4_file\[25\]

`BPF_SOCK_OPS_PASSIVE_ESTABLISHED_CB` 是 socket.accept() 接受连接请求并完成连接建立的操作符，也是众多 `sock_ops` 操作符\[40\] 中的一个。这些操作符，可以被看作是 事件 Event\[41\]，程序的触发则是由事件驱动的。例如：

- 如果客户端发起连接请求并完成三次握手后的操作符是 `BPF_SOCK_OPS_ACTIVE_ESTABLISHED_CB`；

- 套接字进入监听状态时的操作符是 `BPF_SOCK_OPS_TCP_LISTEN_CB`；

- 数据被确认 `BPF_SOCK_OPS_DATA_ACK_CB`

- TCP 状态改变 `BPF_SOCK_OPS_STATE_CB`

最后就是 BPF 程序的执行了，不多做赘述，有兴趣的看这里的 分析\[42\]。

参考资料

\[1\]

几篇 eBPF 方面的学习笔记: _https://atbug.com/search?s=ebpf_

\[2\]

使用 eBPF 技术实现更快的网络数据包传输: _https://atbug.com/accelerate-network-packets-transmission/_

\[3\]

追踪 Kubernetes 中的数据包: _https://atbug.com/tracing-network-packets-in-kubernetes/_

\[4\]

eBPF sockops: _https://github.com/addozhang/ebpf-sockops/_

\[5\]

bpf_sockops: _https://github.com/addozhang/ebpf-sockops/blob/master/bpf_sockops.c#L28_

\[6\]

Linux v6.8: _https://github.com/torvalds/linux/tree/v6.8_

\[7\]

load.sh: _https://github.com/addozhang/ebpf-sockops/blob/master/load.sh_

\[8\]

bpftool: _https://github.com/libbpf/bpftool_

\[9\]

`bpf()`: _https://github.com/torvalds/linux/blob/715d82ba636cb3629a6e18a33bb9dbe53f9936ee/kernel/bpf/syscall.c#L5559_

\[10\]

加载选项`bpf_prog_load_opts`: _https://github.com/torvalds/linux/blob/05c31b4ab20527c4d1695130aaecc54ef59a0e54/tools/lib/bpf/bpf.h#L64_

\[11\]

syscall bpf(): _https://github.com/torvalds/linux/blob/715d82ba636cb3629a6e18a33bb9dbe53f9936ee/kernel/bpf/syscall.c#L5559_

\[12\]

\_\_sys_bpf: _https://github.com/torvalds/linux/blob/715d82ba636cb3629a6e18a33bb9dbe53f9936ee/kernel/bpf/syscall.c#L5456_

\[13\]

bpf_prog_load: _https://github.com/torvalds/linux/blob/715d82ba636cb3629a6e18a33bb9dbe53f9936ee/kernel/bpf/syscall.c#L2595_

\[14\]

BPF 程序类型: _https://github.com/torvalds/linux/blob/d17aff807f845cf93926c28705216639c7279110/tools/include/uapi/linux/bpf.h#L964_

\[15\]

挂载类型: _https://github.com/torvalds/linux/blob/d17aff807f845cf93926c28705216639c7279110/tools/include/uapi/linux/bpf.h#L1000_

\[16\]

挂载选项 `bpf_prog_attach_opts`: _https://github.com/torvalds/linux/blob/05c31b4ab20527c4d1695130aaecc54ef59a0e54/tools/lib/bpf/bpf.h#L321C8-L321C28_

\[17\]

syscall bpf(): _https://github.com/torvalds/linux/blob/715d82ba636cb3629a6e18a33bb9dbe53f9936ee/kernel/bpf/syscall.c#L5466_

\[18\]

bpf_prog_attach: _https://github.com/torvalds/linux/blob/715d82ba636cb3629a6e18a33bb9dbe53f9936ee/kernel/bpf/syscall.c#L3936_

\[19\]

cgroup_bpf_prog_attach: _https://github.com/torvalds/linux/blob/v6.8/kernel/bpf/cgroup.c#L1130_

\[20\]

cgroup_bpf_prog_attach: _https://github.com/torvalds/linux/blob/v6.8/kernel/bpf/cgroup.c#L733_

\[21\]

\_\_cgroup_bpf_attach: _https://github.com/torvalds/linux/blob/v6.8/kernel/bpf/cgroup.c#L607_

\[22\]

bpf_prog_put: _https://github.com/torvalds/linux/blob/v6.8/kernel/bpf/cgroup.c#L700_

\[23\]

static_branch_inc: _https://github.com/torvalds/linux/blob/v6.8/kernel/bpf/cgroup.c#L702_

\[24\]

accept: _https://github.com/torvalds/linux/blob/v6.8/net/socket.c#L2016_

\[25\]

\_\_sys_accept4_file: _https://github.com/torvalds/linux/blob/v6.8/net/socket.c#L1969_

\[26\]

do_accept: _https://github.com/torvalds/linux/blob/v6.8/net/socket.c#L1929_

\[27\]

proto_ops inet_stream_ops: _https://github.com/torvalds/linux/blob/eef00a82c568944f113f2de738156ac591bbd5cd/net/ipv4/af_inet.c#L1051_

\[28\]

inet_stream_ops.accept: _https://github.com/torvalds/linux/blob/eef00a82c568944f113f2de738156ac591bbd5cd/net/ipv4/af_inet.c#L1058_

\[29\]

inet_accept: _https://github.com/torvalds/linux/blob/eef00a82c568944f113f2de738156ac591bbd5cd/net/ipv4/af_inet.c#L780_

\[30\]

`proto tcp_prot`: _https://github.com/torvalds/linux/blob/da7dfaa6d6f731c30eca6ffa808b83634d43e26f/net/ipv4/tcp_ipv4.c#L3314_

\[31\]

tcp_prot.accept: _https://github.com/torvalds/linux/blob/da7dfaa6d6f731c30eca6ffa808b83634d43e26f/net/ipv4/tcp_ipv4.c#L3314_

\[32\]

inet_init: _https://github.com/torvalds/linux/blob/eef00a82c568944f113f2de738156ac591bbd5cd/net/ipv4/af_inet.c#L1997_

\[33\]

net_protocol tcp_protocol: _https://github.com/torvalds/linux/blob/eef00a82c568944f113f2de738156ac591bbd5cd/net/ipv4/af_inet.c#L1754_

\[34\]

tcp_v4_rcv: _https://github.com/torvalds/linux/blob/da7dfaa6d6f731c30eca6ffa808b83634d43e26f/net/ipv4/tcp_ipv4.c#L2319_

\[35\]

tcp_v4_do_rcv: _https://github.com/torvalds/linux/blob/da7dfaa6d6f731c30eca6ffa808b83634d43e26f/net/ipv4/tcp_ipv4.c#L1929_

\[36\]

tcp_rcv_state_process: _https://github.com/torvalds/linux/blob/v6.8/net/ipv4/tcp_input.c#L6724_

\[37\]

tcp_init_transfer: _https://github.com/torvalds/linux/blob/v6.8/net/ipv4/tcp_input.c#L6208_

\[38\]

bpf_skops_established: _https://github.com/torvalds/linux/blob/v6.8/net/ipv4/tcp_input.c#L192_

\[39\]

BPF_CGROUP_RUN_PROG_SOCK_OPS: _https://github.com/torvalds/linux/blob/v6.8/include/linux/bpf-cgroup.h#L350_

\[40\]

`sock_ops` 操作符: _sock_ops_

\[41\]

事件 Event: _https://atbug.com/accelerate-network-packets-transmission/#事件驱动_

\[42\]

分析: _https://atbug.com/accelerate-network-packets-transmission/#实现_

阅读原文

阅读 1127

​

写留言

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/CrEZgiblEdJWdTnicQwNnku8Bf0M2ibrxIiannSqT6OOppxiaaQxsqiaTeLq8YAc8SRsL3AH6icXW29ypuQzsXUNfwaRg/300?wx_fmt=png&wxfrom=18)

深入浅出BPF

6944

写留言

写留言

**留言**

暂无留言
