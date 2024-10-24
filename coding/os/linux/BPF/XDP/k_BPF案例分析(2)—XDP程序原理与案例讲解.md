
Original 董旭 技术简说 _2021年11月19日 15:08_

BPF案例分析(2)—XDP程序

**相关阅读清单**

.1、[BPF 编程环境搭建](http://mp.weixin.qq.com/s?__biz=Mzg5MTU1ODgyMA==&mid=2247483953&idx=1&sn=f204e441e066302b9999cdcff690981d&chksm=cfcaccfaf8bd45ec2912ab7fa388a0286659fb6c315bf8244ba1bda5b981fe7f7ab5fbf6dc1d&scene=21#wechat_redirect)

.2、[BPF原理深度分析与案例分析(1)](http://mp.weixin.qq.com/s?__biz=Mzg5MTU1ODgyMA==&mid=2247484004&idx=1&sn=5b681f09f60f96aa8992d894bb3b7b4d&chksm=cfcaccaff8bd45b9d6694d931bd8824a5d025c3dd7f6246275b90dbda0a9ce5f3de3cbdafedb&scene=21#wechat_redirect)

在[BPF深度分析与案例分析(1)](http://mp.weixin.qq.com/s?__biz=Mzg5MTU1ODgyMA==&mid=2247484004&idx=1&sn=5b681f09f60f96aa8992d894bb3b7b4d&chksm=cfcaccaff8bd45b9d6694d931bd8824a5d025c3dd7f6246275b90dbda0a9ce5f3de3cbdafedb&scene=21#wechat_redirect)中讲到BPF虚拟机会根据不同的BPF程序类型决定在何种事件触发BPF程序、何时触发BPF程序，同时BPF程序类型决定了BPF程序的上下文参数，例如在BPF深度分析与案例分析(1)中分析的套接字过滤程序(BPF_PROG_TYPE_SOCKER_FILTR类型的BPF程序)，该程序类型的BPF程序的上下文参数是struct \_\_sk_buff,该结构体是内核结构体sk_buff中的一些关键字段，内核在执行BPF程序时会将对这些关键字段的访问转换成“真正”sk_buff结构体的偏移量。套接字过滤程序会附加到原始套接字上，用于对该套接字的观测，但是不允许修改数据包内容或更改其目的地。该程序类型用于数据包的旁路嗅探，tcpdump就是基于这个原理。

本文将从上篇介绍套接字过滤程序(BPF_PROG_TYPE_SOCKER_FILTR类型的BPF程序)的方式讲解XDP程序。并在文章最后编写两个XDP程序进行实验和分析。

XDP简介

XDP(eXpress Data Path)的程序类型是BPF_PROG_TYPE_XDP，该程序类型的BPF程序设计目标是在网络数据路径中引入可编程性，在Linux内核分配内存（skb）之前就已经完成处理,在网络包达到内核之前XDP就已经触发并执行。与套接字过滤程序(BPF_PROG_TYPE_SOCKER_FILTR类型的BPF程序)在处理路径上的不同：套接字过滤程序在内核协议栈处理收发包流程时进行旁路监听与观测，而XDP程序是在数据包到达内核协议栈之前(网卡驱动程序收到数据包时)触发XDP类型的BPF程序并进行数据包的处理。在行为上的不同：套接字过滤程序只能进行观测、过滤等旁路嗅探数据包，而XDP程序可以对数据包进行修改、重定向、丢弃等。

场景：DDos防御，四层负载均衡等场景

优势：XDP执行时skb都还没创建，开销非常低，因此效率非常高，通过BPF hook对内核进行运行时编程，但基于内核而不是绕过(bypass)内核。

三种工作模式   ：

Native XDP(XDP_FLAGS_DRV_MODE):这是一种默认的模式，XDP BPF程序运行在网络驱动的早期接收路径(RX队列)上，但是要保证当前的驱动程序是否支持这种模式。Offloaded XDP(XDP_FLAGS_HW_MODE)：ffloadedXDP模式中，XDP BPF程序直接在NIC中处理报文，而不会使用主机的CPU。因此，处理报文的成本非常低，性能要远远高于native XDP。Generic XDP(XDP_FLAGS_SKB_MODE)：对没有实现native或offloaded模式的XDP，内核提供了一种处理XDP的通用方案，但性能远低于前两种模式。

XDP程序的上下文参数(传入的参数)

上篇文章讲到的套接字过滤程序(BPF_PROG_TYPE_SOCKER_FILTR类型的BPF程序)的上下文参数：

`//该结构体是内核中struct sk_buff的关键字段，在内核在执行BPF程序时会将对这些关键字段的访问转换成“真正”sk_buff结构体的偏移量   struct __sk_buff {    __u32 len;    __u32 pkt_type;    __u32 mark;    __u32 queue_mapping;    __u32 protocol;    __u32 vlan_present;    __u32 vlan_tci;    __u32 vlan_proto;    __u32 priority;    __u32 ingress_ifindex;    __u32 ifindex;    __u32 tc_index;    __u32 cb[5];    __u32 hash;    __u32 tc_classid;    __u32 data;    __u32 data_end;    __u32 napi_id;       /* Accessed by BPF_PROG_TYPE_sk_skb types from here to ... */    __u32 family;    __u32 remote_ip4; /* Stored in network byte order */    __u32 local_ip4; /* Stored in network byte order */    __u32 remote_ip6[4]; /* Stored in network byte order */    __u32 local_ip6[4]; /* Stored in network byte order */    __u32 remote_port; /* Stored in network byte order */    __u32 local_port; /* stored in host byte order */    /* ... here. */       __u32 data_meta;   };   `

XDP程序(BPF_PROG_TYPE_SOCKER_FILTR类型的BPF程序)的上下文参数如下：

`struct xdp_md {    __u32 data;//数据包的开始    __u32 data_end;//数据包的结束    __u32 data_meta;//供XDP程序与其他交换数据包元数据时使用   };   `

XDP程序的返回值

在程序中可以定义以上的返回值，各返回值的产生的动作如下：

`// include/uapi/linux/bpf.h      enum xdp_action {       XDP_ABORTED = 0,       XDP_DROP,          XDP_PASS,          XDP_TX,       XDP_REDIRECT,   };   `

XDP_DROP：丢弃数据包

XDP_RTX:     转发数据包(可能发生在修改数据包之前或之后)

XDP_REDIRECT:  重定向

XDP_PASS :等效于不做任何处理

XDP_ABORTED:eBPF程序错误

如何attach XDP程序

在讲解套接字过滤程序时，分析了套接字过滤程序是通过SO_ATTACH_BPF setsockopt()进行attach，如下面的程序片段(可参考文章)：

`....   if (load_bpf_file(filename)) {     printf("%s", bpf_log_buf);     return 1;    }     //主要是完成：sock = socket(PF_PACKET, SOCK_RAW | SOCK_NONBLOCK | SOCK_CLOEXEC, htons(ETH_P_ALL));    sock = open_raw_sock("lo");     /*因为 sockex1_kern.o 中 bpf 程序的类型为 BPF_PROG_TYPE_SOCKET_FILTER,所以这里需要用用 SO_ATTACH_BPF 来指明程序的 sk_filter 要挂载到哪一个套接字上，其中prof_fd为注入到内核的BPF程序的描述符*/    assert(setsockopt(sock, SOL_SOCKET, SO_ATTACH_BPF, prog_fd,        sizeof(prog_fd[0])) == 0);   ...   `

那么XDP程序是如何attach的?

通过 netlink socket 消息 attach：

- 首先创建一个 netlink 类型的 socket：socket(AF_NETLINK, SOCK_RAW, NETLINK_ROUTE)

- 然后发送一个 NLA_F_NESTED | 43 类型的 netlink 消息，表示这是 XDP message。消息中包含 BPF fd, the interface index (ifindex) 等信息。

attach的具体实现：

`//其中ifindex是当前系统的网卡索引号，prog_fd[0]是插入到内核的eBPF程序(XDP程序的描述符)   set_link_xdp_fd(ifindex, prog_fd[0], xdp_flags)   `

该函数的具体实现在samples/bpf/load.c中

大致就是上面说的创建一个 netlink 类型的 socket：socket(AF_NETLINK, SOCK_RAW, NETLINK_ROUTE)，然后发送一个 NLA_F_NESTED | 43 类型的 netlink 消息。

如何向内核加载XDP程序‍

在上一篇文章中介绍了加载eBPF程序的过程，主要是利用load_bpf_file->do_load_bpf_file函数，并最终调用 sys_bpf(BPF_PROG_LOAD, &attr, sizeof(attr))系统调用进行加载，详细可以阅读[上一篇文章](http://mp.weixin.qq.com/s?__biz=Mzg5MTU1ODgyMA==&mid=2247484004&idx=1&sn=5b681f09f60f96aa8992d894bb3b7b4d&chksm=cfcaccaff8bd45b9d6694d931bd8824a5d025c3dd7f6246275b90dbda0a9ce5f3de3cbdafedb&scene=21#wechat_redirect)。

XDP程序除了可以上述系统调用的方式加载外，还可以通过iproute2中提供的ip命令，该命令具有充当XDP前端的能力，可以将XDP程序加载到HOOK点。下文会采用两种方式分别进行加载。

XDP Demo1

##### 本demo使用ip命令进行加载到HOOK点，没有使用到用户态展示XDP处理详情。

#### xdp_demo1.c

`#include <linux/bpf.h>   #include <linux/ip.h>   #include <linux/tcp.h>   #include <linux/in.h>   #include <linux/if_ether.h>   #define SEC(NAME) __attribute__((section(NAME), used))      SEC("xdp")   int xdp_drop_the_world(struct xdp_md *ctx) {       //从xdp程序的上下文参数获取数据包的起始地址和终止地址    void *data = (void *)(long)ctx->data;    void *data_end = (void *)(long)ctx->data_end;    int ipsize = 0;    __u32 idx;       //以太网头部    struct ethhdr  *eth = data;       //ip头部    struct iphdr *ip;    struct tcphdr *tcp;       //以太网头部偏移量    ipsize = sizeof(*eth);    ip = data + ipsize;    ipsize += sizeof(struct iphdr);       //异常数据包，丢弃    if(data + ipsize > data_end){     return XDP_DROP;    }       //从ip头部获取上层协议    idx = ip->protocol;    //如果是icmp协议，则drop掉       if(idx == IPPROTO_ICMP){        return XDP_DROP;       }       return XDP_PASS;                }      char _license[] SEC("license") = "GPL";   `

> 上述程序的功能：屏蔽掉系统的icmp协议数据包，这将导致主机的ping功能失效

#### 编译与加载

使用clang编译器进行编译

`dx@ubuntu:~$ sudo clang -O2 -target bpf -c xdp_demo1.c -o xdp_demo1.o   `

使用readelf -S xdp_demo1.o可以看到程序中定义的section，如xdp，license,关于BPF的section也可查看[上一篇文章](http://mp.weixin.qq.com/s?__biz=Mzg5MTU1ODgyMA==&mid=2247484004&idx=1&sn=5b681f09f60f96aa8992d894bb3b7b4d&chksm=cfcaccaff8bd45b9d6694d931bd8824a5d025c3dd7f6246275b90dbda0a9ce5f3de3cbdafedb&scene=21#wechat_redirect)。

`dx@ubuntu:~$ readelf -S xdp_demo1.o   There are 6 section headers, starting at offset 0x140:      Section Headers:     [Nr] Name              Type             Address           Offset          Size              EntSize          Flags  Link  Info  Align     [ 0]                   NULL             0000000000000000  00000000          0000000000000000  0000000000000000           0     0     0     [ 1] .strtab           STRTAB           0000000000000000  00000100          000000000000003e  0000000000000000           0     0     1     [ 2] .text             PROGBITS         0000000000000000  00000040          0000000000000000  0000000000000000  AX       0     0     4     [ 3] xdp               PROGBITS         0000000000000000  00000040          0000000000000058  0000000000000000  AX       0     0     8     [ 4] license           PROGBITS         0000000000000000  00000098          0000000000000004  0000000000000000  WA       0     0     1     [ 5] .symtab           SYMTAB           0000000000000000  000000a0          0000000000000060  0000000000000018           1     2     8   Key to Flags:     W (write), A (alloc), X (execute), M (merge), S (strings), I (info),     L (link order), O (extra OS processing required), G (group), T (TLS),     C (compressed), x (unknown), o (OS specific), E (exclude),     p (processor specific)   `

使用readelf -h xdp_demo1.o可以看到，Machine字段：Linux BPF

`dx@ubuntu:~$ readelf -h xdp_demo1.o   ELF Header:     Magic:   7f 45 4c 46 02 01 01 00 00 00 00 00 00 00 00 00      Class:                             ELF64     Data:                              2's complement, little endian     Version:                           1 (current)     OS/ABI:                            UNIX - System V     ABI Version:                       0     Type:                              REL (Relocatable file)     Machine:                           Linux BPF     Version:                           0x1     Entry point address:               0x0     Start of program headers:          0 (bytes into file)     Start of section headers:          320 (bytes into file)     Flags:                             0x0     Size of this header:               64 (bytes)     Size of program headers:           0 (bytes)     Number of program headers:         0     Size of section headers:           64 (bytes)     Number of section headers:         6     Section header string table index: 1   `

下面将使用ip命令将编译好的xdp程序进行加载

`#ip link set dev [device name] xdp obj [编译后的xdp程序名] sec [section name] verbose   dx@ubuntu:~$ sudo ip link set dev ens33 xdp obj xdp_demo1.o sec xdp verbose      Prog section 'xdp' loaded (5)!    - Type:         6    - Instructions: 11 (0 over limit)    - License:      GPL      Verifier analysis:      0: (b7) r0 = 1   1: (61) r2 = *(u32 *)(r1 +4)   2: (61) r1 = *(u32 *)(r1 +0)   3: (bf) r3 = r1   4: (07) r3 += 34   5: (2d) if r3 > r2 goto pc+4    R0=inv1 R1=pkt(id=0,off=0,r=34,imm=0) R2=pkt_end(id=0,off=0,imm=0) R3=pkt(id=0,off=34,r=34,imm=0) R10=fp0   6: (71) r1 = *(u8 *)(r1 +23)   7: (b7) r0 = 1   8: (15) if r1 == 0x1 goto pc+1    R0=inv1 R1=inv(id=0,umax_value=255,var_off=(0x0; 0xff)) R2=pkt_end(id=0,off=0,imm=0) R3=pkt(id=0,off=34,r=34,imm=0) R10=fp0   9: (b7) r0 = 2   10: (95) exit      from 8 to 10: R0=inv1 R1=inv1 R2=pkt_end(id=0,off=0,imm=0) R3=pkt(id=0,off=34,r=34,imm=0) R10=fp0   10: (95) exit      from 5 to 10: safe   processed 13 insns, stack depth 0   `

#### 验证

1、首先使用 ip address命令查看所挂载的网卡：

`dx@ubuntu:~$ ip a   1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000       link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00       inet 127.0.0.1/8 scope host lo          valid_lft forever preferred_lft forever       inet6 ::1/128 scope host           valid_lft forever preferred_lft forever   2: ens33: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 xdpgeneric/id:27 qdisc fq_codel state UP group default qlen 1000       link/ether 00:0c:29:b5:b1:93 brd ff:ff:ff:ff:ff:ff       inet 192.168.18.187/24 brd 192.168.18.255 scope global dynamic noprefixroute ens33          valid_lft 1630sec preferred_lft 1630sec       inet6 fe80::8347:a6e5:3218:8048/64 scope link noprefixroute           valid_lft forever preferred_lft forever   `

可以看到在ens33网卡接口的MTU字段后面，显示了 xdpgeneric/id:27，它显示了两个有用的信息。

- 已使用的驱动程序为xdpgeneric

- XDP程序的ID为32

2、查看XDP程序的效果

ping 8.8.8.8 共10次，结果如下，丢包为100%

`dx@ubuntu:~$ ping 8.8.8.8 -c20   PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.      --- 8.8.8.8 ping statistics ---   20 packets transmitted, 0 received, 100% packet loss, time 19448ms   `

#### 卸载xdp程序与验证

`#卸载命令  ip link set dev [dev name] xdp off   dx@ubuntu:~$ sudo ip link set dev ens33 xdp off   #验证如下，丢包率为0   dx@ubuntu:~$ ping 8.8.8.8 -c2   PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.   64 bytes from 8.8.8.8: icmp_seq=1 ttl=128 time=39.6 ms   64 bytes from 8.8.8.8: icmp_seq=2 ttl=128 time=42.5 ms      --- 8.8.8.8 ping statistics ---   2 packets transmitted, 2 received, 0% packet loss, time 1001ms   rtt min/avg/max/mdev = 39.625/41.085/42.545/1.460 ms   `

### XDP Demo2

##### 本demo使用bpf的系统调用进行加载到HOOK点，并使用MAP映射，用户态读取Map并展示XDP处理详情。本程序在BPF 编程环境下进行编译与运行(参考：[BPF编程 环境搭建](http://mp.weixin.qq.com/s?__biz=Mzg5MTU1ODgyMA==&mid=2247483953&idx=1&sn=f204e441e066302b9999cdcff690981d&chksm=cfcaccfaf8bd45ec2912ab7fa388a0286659fb6c315bf8244ba1bda5b981fe7f7ab5fbf6dc1d&scene=21#wechat_redirect))。

#### xdp_demo2_kern.o

`#include <linux/bpf.h>   #include <linux/ip.h>   #include <linux/tcp.h>   #include <linux/in.h>   #include <linux/if_ether.h>   #include "bpf_helpers.h"   #include "bpf_endian.h"   #define SEC(NAME) __attribute__((section(NAME), used))      //定义一个 map,用于统计协议收包统计   struct bpf_map_def SEC("maps") rxcnt = {    .type = BPF_MAP_TYPE_PERCPU_ARRAY,    .key_size = sizeof(u32),    .value_size = sizeof(long),    .max_entries = 256,   };      SEC("xdp")   int xdp_drop_the_world(struct xdp_md *ctx) {    void *data = (void *)(long)ctx->data;    void *data_end = (void *)(long)ctx->data_end;    int ipsize = 0;    __u32 idx;    u16 port;    long *value;    struct ethhdr  *eth = data;    struct iphdr *ip;    struct tcphdr *tcp;    ipsize = sizeof(*eth);    ip = data + ipsize;    ipsize += sizeof(struct iphdr);    if(data + ipsize > data_end){     return XDP_DROP;    }    idx = ip->protocol;       //判断协议字段，若为icmp则drop，若为TCP则屏蔽掉22端口    switch(idx){           case IPPROTO_ICMP:                   value = bpf_map_lookup_elem(&rxcnt,&idx);     if(value)     (*value) += 1;  //icmp协议丢包记录++     return XDP_DROP;    case IPPROTO_TCP:     tcp = data + ipsize;     if(tcp + 1 > data_end)      return XDP_DROP;     port = bpf_ntohs(tcp->dest);     if(port == 22){      value = bpf_map_lookup_elem(&rxcnt,&idx);      if(value)        (*value) += 1;  //tcp协议22端口丢包记录++      return XDP_DROP;     }       }           return XDP_PASS;    }   char _license[] SEC("license") = "GPL";   `

> 上述程序的功能：屏蔽掉系统的icmp协议数据包与TCP协议的22端口，这将导致一些远程连接服务、ping工具失效

由于我们要采用bpf系统调用的方式加载xdp程序，并且想要读取MAP中的信息，所以编写用户态进行对编译好的xdp进行加载与展示xdp处理数据的详情。xdp_demo2_user.c

`#include <linux/bpf.h>   #include <linux/if_link.h>   #include <assert.h>   #include <errno.h>   #include <signal.h>   #include <stdio.h>   #include <stdlib.h>   #include <string.h>   #include <unistd.h>   #include <libgen.h>   #include <sys/resource.h>      #include "bpf_load.h"   #include "bpf_util.h"   #include "libbpf.h"   #define IPPROTO_ICMP 1   #define IPPROTO_TCP 6         static int ifindex;   static __u32 xdp_flags;      //便于用户态使用ctrl+c终止xdp程序，并将xdp程序从HOOK点卸载   static void int_exit(int sig)   {           //从HOOK点卸载xdp程序    set_link_xdp_fd(ifindex, -1, xdp_flags);    exit(0);   }      //读取xdp针对策略的丢包个数   static void poll_stats(int interval)   {    //获取cpu逻辑核心数    unsigned int nr_cpus = bpf_num_possible_cpus();    const unsigned int nr_keys = 256;    __u64 values[nr_cpus];    __u32 key;    int i;          while (1) {     sleep(interval);            //循环每个映射的索引     for (key = 0; key < nr_keys; key++) {      __u64 sum = 0;               //查询索引对应的值：value      assert(bpf_map_lookup_elem(map_fd[0], &key, values) == 0);      for (i = 0; i < nr_cpus; i++)       //计算每个逻辑cpu上处理的协议统计       sum += values[i];      if (sum){       if(key==6)       printf("TCP: %10llu pkt\n", sum);       else if( key == 1)       printf("ICMP :%10llu pkt\n", sum);         }           }    }   }   //用法提示   static void usage(const char *prog)   {    fprintf(stderr,     "usage: %s [OPTS] IFINDEX\n\n"     "OPTS:\n"     "    -S    use skb-mode\n"     "    -N    enforce native mode\n",     prog);   }      int main(int argc, char **argv)   {    struct rlimit r = {RLIM_INFINITY, RLIM_INFINITY};    const char *optstr = "SN";    char filename[256];    int opt;       while ((opt = getopt(argc, argv, optstr)) != -1) {     switch (opt) {     case 'S':      xdp_flags |= XDP_FLAGS_SKB_MODE;      break;     case 'N':      xdp_flags |= XDP_FLAGS_DRV_MODE;      break;     default:      usage(basename(argv[0]));      return 1;     }    }       if (optind == argc) {     usage(basename(argv[0]));     return 1;    }       if (setrlimit(RLIMIT_MEMLOCK, &r)) {     perror("setrlimit(RLIMIT_MEMLOCK)");     return 1;    }       //获取运行指定的网卡参数(网卡索引，也就是XDP要HOOK的网卡)    ifindex = strtoul(argv[optind], NULL, 0);       snprintf(filename, sizeof(filename), "%s_kern.o", argv[0]);       //调用load_bpf_file函数，继而调用bpf系统调用将编辑的xdp程序进行加载    if (load_bpf_file(filename)) {     printf("%s", bpf_log_buf);     return 1;    }       if (!prog_fd[0]) {     printf("load_bpf_file: %s\n", strerror(errno));     return 1;    }          signal(SIGINT, int_exit);    signal(SIGTERM, int_exit);        //使用set)link_xdp_fd函数将XDP程序attach    if (set_link_xdp_fd(ifindex, prog_fd[0], xdp_flags) < 0) {     printf("link set xdp fd failed\n");     return 1;    }    printf("yes\n");       poll_stats(2);    return 0;   }   `

#### 编译

在BPF 编程环境中，我们可以很便利地使用现成的Makefile进行编程，在Makefile中添加如下：

`....   hostprogs-y += xdp_demo2   ....   xdp_demo2-objs := bpf_load.o $(LIBBPF) xdp_demo2_user.o   ...   always += xdp_demo2_kern.o   ...   HOSTLOADLIBES_xdp_demo2 += -lelf   ...   `

进行make编程通过即可:

`dx@ubuntu:/usr/src/linux-4.15.0/samples/bpf$ sudo vim Makefile    [sudo] password for dx:    dx@ubuntu:/usr/src/linux-4.15.0/samples/bpf$ cd ../..   dx@ubuntu:/usr/src/linux-4.15.0$ sudo make M=samples/bpf/   `

#### 加载、运行

从xdp_demo2_user.c中就可以看出，xdp_demo2_kern.o是在执行用户态程序实时加载的：

`snprintf(filename, sizeof(filename), "%s_kern.o", argv[0]);       //调用load_bpf_file函数，继而调用bpf系统调用将编辑的xdp程序进行加载    if (load_bpf_file(filename)) {     printf("%s", bpf_log_buf);     return 1;    }`

**运行：（因为该程序将TCP22端口的数据DROP掉了，不建议在远程连接服务器的环境下进行运行）**

`#2为指定的网卡索引，在这指我主机的ens33网卡接口   dx@ubuntu:/usr/src/linux-4.15.0/samples/bpf$ sudo ./xdp_demo2 2   `

#### 验证：

1、测试TCP 22端口：使用xshell等远程连接工具来连接主机 使用tcpdump抓取tcp 22端口的数据包。

`#1、先开启xdp程序   dx@ubuntu:/usr/src/linux-4.15.0/samples/bpf$ sudo ./xdp_demo2 2   #2、使用tcpdump抓包，其中192.168.18.187是本机ip地址   dx@ubuntu:~$ sudo tcpdump tcp port 22 and src host 192.168.18.187   #3、使用xshell等远程连接工具对主机进行连接   `

可以看到tcpdump没有抓包任何相关的数据包

在xdp程序的运行结果中查看：远端连接的数据包都给DROP并统计了DROP的数据包的个数。

`dx@ubuntu:/usr/src/linux-4.15.0/samples/bpf$ sudo ./xdp02 2   yes   TCP:          1 pkt   TCP:          2 pkt   TCP:          3 pkt   TCP:          3 pkt   TCP:          4 pkt   TCP:          4 pkt   TCP:          4 pkt   TCP:          4 pkt   TCP:          4 pkt   TCP:          4 pkt   TCP:          6 pkt   TCP:          7 pkt   TCP:          7 pkt   TCP:          8 pkt   `

2、测试icmp协议

`#运行xdp程序   dx@ubuntu:/usr/src/linux-4.15.0/samples/bpf$ sudo ./xdp02 2   #使用ping工具，ping 8.8.8.8 5次   dx@ubuntu:~$ ping 8.8.8.8 -c5   PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.      --- 8.8.8.8 ping statistics ---   5 packets transmitted, 0 received, 100% packet loss, time 4078ms   `

可以看到ping的数据包全部DROP，在xdp中统计到DROP了5个

`dx@ubuntu:/usr/src/linux-4.15.0/samples/bpf$ sudo ./xdp02 2   yes   ICMP :         1 pkt   ICMP :         2 pkt   ICMP :         4 pkt   ICMP :         5 pkt   ICMP :         5 pkt   ICMP :         5 pkt   ICMP :         5 pkt   `

更易上手的XDP编程方式

BCC是 python 封装的 eBPF 外围工具集，可以大大提高BPF 程序开发的效率,而且安装以及搭建环境简单。

BCC仓库地址：https://github.com/iovisor/bcc，仓库中有相关环境搭建，项目安装，编程案例。

其中在：https://github.com/iovisor/bcc/tree/master/examples/networking/xdp有一些使用BCC实现XDP程序的案例。

END

![](http://mmbiz.qpic.cn/mmbiz_png/EjWxxIM2EQI0YHzCpIYYwO0iaqh08EGCibYEjLqIqYm2CXPzmicQTkxqF453q1d9RcSicSLGjjCNyBsjDXdx8oDhcA/300?wx_fmt=png&wxfrom=19)

**技术简说**

主要分享Linux内核、内核网络知识，欢迎大家关注！

35篇原创内容

公众号

点个关注，一起学技术！

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

您的点赞和关注是我最大的动力

BPF3

BPF · 目录

上一篇BPF 深度分析与案例讲解(1)

Reads 311

​
