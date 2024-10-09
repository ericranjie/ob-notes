# 

Linux内核之旅

_2021年09月28日 20:20_

The following article is from CodeTrip Author JiaoTuan

\[

![](http://wx.qlogo.cn/mmhead/Q3auHgzwzM4nExwIdGKOJulzBEKoUGflRjZAbvZGxl1Er4os44kBug/0)

**CodeTrip**.

今天也要加油鸭！

\](https://mp.weixin.qq.com/s?\_\_biz=MzI3NzA5MzUxNA==&mid=2664610351&idx=1&sn=1b04206388762a0a272909ac5ce37051&chksm=f04d97cac73a1edc0453677a614c4bb5edc0305d43b49325dde920c41b11455d5aa7207ab113&mpshare=1&scene=24&srcid=0928pRq4CX0wt7HcoxbhuWyi&sharer_sharetime=1632833087916&sharer_shareid=5fb9813bfe9ffc983435bfc8d8c5e9ca&key=daf9bdc5abc4e8d077cc348bb36e331e2c589d4748488b876f62b86a9a93d233fef92d70d87aa46bccf44187760b925dfa786ae1c695d289db3578d0e2f0a7e4c5b9beb007712a067a143393fab4cabc06989c0fda32be4c3d679cf08c94cf9be6ec62c9b0ffa8c838ef177e4c742ae36087fb26c7fc41e3de0dd883dd61e646&ascene=14&uin=MTEwNTU1MjgwMw%3D%3D&devicetype=iMac+MacBookAir10%2C1+OSX+OSX+14.6.1+build(23G93)&version=13080710&nettype=WIFI&lang=en&session_us=gh_3e85a6de261f&countrycode=CN&fontScale=100&exportkey=n_ChQIAhIQXf%2FHvbQUSZz5KRWLCRV5LxKUAgIE97dBBAEAAAAAAArPDA0EDp0AAAAOpnltbLcz9gKNyK89dVj0jaICPS2E1BzWzVTiK0vCL%2FP0wVWto6WUPWFD1PRIXLHMpFJlZgziqu4Y99igfX3gsrzPAa3wnKL3EEOlaZtjpbnJ1%2FfGl%2F%2Fq%2BDqCeMH4XWyUWIbbRc2izMGAz34DXYc%2BZP%2FhfFB7VDeIgsy5eRQDZwchn5DyCVjryS0G6kgZqEmYmDyu5YQKojkut7i0uZhViiKGdZ2aFGwqR%2FaF3Bpz2sfX0TRINKHmv8XYPsAGTjchEDP%2BhKGY4LLWq1ICKMabTD4NLKZGVbB20GTv3D8z%2Fg0HEZcUUUW8O5nrJMWfD76xqOpeBr%2FB7x3u2%2BWqfw%3D%3D&acctmode=0&pass_ticket=8R5Et6binWMNrB7qIBTB%2BGZuLLwOJDCMV2RpF8h2pcA0CfiujX6I21f8fUrBrCKl&wx_header=0#)

# 一个简单的XDP程序

## 1. 环境

`内核版本：`Linux-4.15

`gcc版本:` 7.5.0

`llvm:`  >= version 3.7.1

`clang:` >= version 3.4.0

`虚拟机A IP：`10.211.55.17

`虚拟机B IP：`10.211.55.12

**依赖：**

`sudo apt install libncurses5-dev flex bison libelf-dev binutils-dev libssl-dev   `

## 2. 编译

Linux内核实现了cBPF/eBPF虚拟机，通过llvm和clang编译自己写的c代码。

sample/bpf目录下有内核bpf示例代码

进入内核后：

1. 配置内核

生成.config文件

`make menuconfig   `

2. 关联内核头文件

`make headers_install   `

3. 编译内核bpf例子

`make M=sample/bpf   `

示例代码编译成功。

基于该环境编写自己的`xdp/eBPF`程序。

## 3. 代码

**test_xdp1_kern.c:**

`#define KBUILD_MODNAME "foo"   #include <linux/bpf.h>   #include <linux/if_ether.h>   #include <linux/in.h>   #include <linux/ip.h>   #include <uapi/linux/tcp.h>   #include <uapi/linux/udp.h>   #include "bpf_helpers.h"   #include "bpf_endian.h"      struct packet {           unsigned int src;           unsigned int dst;           unsigned short l3proto;           unsigned short l4proto;           unsigned short sport;           unsigned short dport;   };      struct bpf_map_def SEC("maps") my_xdp1 = {           .type = BPF_MAP_TYPE_HASH,           .key_size = sizeof(__u32),           .value_size = sizeof(struct packet),           .max_entries = 1,   };      SEC("xdp1")   int prog1(struct xdp_md *ctx)   {           void *data_end = (void *)(long)ctx->data_end;           void *data = (void *)(long)ctx->data;           struct ethhdr *eth = data;           struct packet p = {};           int key = 0;           u32 lookup_addr;           u16 *value;           u8 flags;           if (data + sizeof(struct ethhdr) > data_end) {                   return XDP_DROP;           }              p.l3proto = bpf_htons(eth->h_proto);              if (bpf_htons(eth->h_proto) == ETH_P_IP) {                   struct iphdr *iph;                   iph = data + sizeof(struct ethhdr);                   if (iph + 1 > data_end)                           return XDP_DROP;                      p.src = iph->saddr;                   p.dst = iph->daddr;                   p.l4proto = iph->protocol;                   p.sport = -1;                   p.dport = -2;                   if (iph->saddr ==  204985098)                   {                           if (iph->protocol == IPPROTO_TCP) {                                   struct tcphdr *tcph;                                   tcph = data + sizeof(struct ethhdr) + sizeof(struct iphdr);                                   if (tcph + 1 > data_end)                                           return XDP_DROP;                                      p.sport = tcph->source;                                   p.dport = tcph->dest;                                   bpf_map_update_elem(&my_xdp1,&key , &p, BPF_ANY);                                   tcph->dest = bpf_htons(8080);                                      return XDP_PASS;                           } else if (iph->protocol == IPPROTO_UDP) {                                   struct udphdr *udph;                                   udph = data + sizeof(struct ethhdr) + sizeof(struct iphdr);                                   if (udph + 1 > data_end)                                           return XDP_DROP;                                      p.sport = udph->source;                                   p.dport = udph->dest;                                   bpf_map_update_elem(&my_xdp1,&key , &p, BPF_ANY);                                   udph->dest = bpf_htons(8080);                                      return XDP_PASS;                           }                   }                   bpf_map_update_elem(&my_xdp1,&key , &p, BPF_ANY);           }           return XDP_DROP;   }   char _license[] SEC("license") = "GPL";   `

**test_xdp1_user.c:**

`#include <errno.h>   #include <stdio.h>   #include <linux/if_link.h>   #include <string.h>   #include <unistd.h>      #include <sys/socket.h>   #include <netinet/in.h>   #include <arpa/inet.h>      #include <arpa/inet.h>   #include "libbpf.h"   #include "bpf_load.h"   typedef unsigned short u16;   typedef unsigned int u32;   typedef unsigned long u64;   static const char *mapfile = "/sys/fs/bpf/xdp/globals/my_xdp1";      struct packet {           unsigned int src;           unsigned int dst;           unsigned short l3proto;           unsigned short l4proto;           unsigned short sport;           unsigned short dport;   };      int main(int argc, char **argv)   {              int fd;           u64 key = 0;           struct packet p = {};           int result;           struct in_addr saddr;              // fd = bpf_obj_get(mapfile);              char filename[256];           //int recv_fd = map_fd[0];              snprintf(filename, sizeof(filename), "%s_kern.o",argv[0]);           if (load_bpf_file(filename)) {                   printf("%s", bpf_log_buf);                   return 1;           }              if (set_link_xdp_fd(2, prog_fd[0], XDP_FLAGS_SKB_MODE) < 0)           {                   printf("link set xdp fd failed\n");                   return 1;           }           printf("src  sport  dport\n");           while(1){                      //sleep(1);                   result = bpf_map_lookup_elem(map_data[0].fd,&key,&p);                   if (result == 0 )                   {                           if(p.src!=0&&p.dst!=0 &&p.sport!=p.dport)                           {       saddr.s_addr = p.src;                                   printf("%s   %d  %d\n",inet_ntoa(saddr),ntohs(p.sport),ntohs(p.dport));                              }                   }           }           return 0;   }   `

**Makefile:**

makefile有四处需要添加

`......   hostprogs-y += test_xdp1   ......   test_xdp1-objs := bpf_load.o $(LIBBPF) test_xdp1_user.o   ......   always += test_xdp1_kern.o   ......   HOSTLOADLIBES_test_xdp1 += -lelf   `

回到源码目录下执行如下命令：

`make M=sample/bpf   `

编译成功。

# kern.c分析

`test_xdp1_kern.c`将除IP为10.211.5.12外的数据报全部过滤，且将该IP发送过来数据报端口进行修改，修改为8080

## 1. 创建BPF Map

使用ELF约定创建BPF映射

`struct packet {           unsigned int src;           unsigned int dst;           unsigned short l3proto;           unsigned short l4proto;           unsigned short sport;           unsigned short dport;   };      struct bpf_map_def SEC("maps") my_xdp1 = {           .type = BPF_MAP_TYPE_HASH,           .key_size = sizeof(__u32),           .value_size = sizeof(struct packet),           .max_entries = 1,   };   `

## 2. 更新BPF Map

使用内核提供帮助函数更新BPF Map。

`bpf_map_update_elem(&my_xdp1,&key , &p, BPF_ANY);`

内核程序可以直接访问BPF映射，参数一直接使用Map指针。

`参数二：`要更新键指针

`参数三：`存入的值，此处值类型为我们定义的结构体

`参数四：`更新映射的方式

|值|含义|
|---|---|
|BPF_ANY|元素已存在，内核将更新元素，不存在则创建元素|
|BPF_NOEXIST|仅在元素不存在时内核创建元素|
|BPF_EXIST|仅在元素存在时内核创建元素|

# user.c分析

## 1. 加载kern.o

使用load_bpf_file函数加载test_xdp1_kern.o将xdp内核程序加载进内核，本质是使用BPF_PROG_LOAD命令进行系统调用

## 2. 挂载网卡

使用set_link_xdp_fd函数将BPF程序加载到目标网卡上

`set_link_xdp_fd(2, prog_fd[0], XDP_FLAGS_SKB_MODE)   `

`参数一：` 目标网卡序号ip a查看

`参数二：`prog_fd\[0\]为BPF程序加载到内存后生成的文件描述符fd

加载内核空间BPF程序时，一旦fd生成后，就添加到这个数组中

> samples/bpf/bpf_load.c

`int prog_fd[MAX_PROGS];   `

`参数三：`该参数为加载XDP程序的类型

> include/uapi/linux/if_link.h

`/* XDP section */      #define XDP_FLAGS_UPDATE_IF_NOEXIST (1U << 0)   #define XDP_FLAGS_SKB_MODE  (1U << 1) // xdpgeneric通用XDP模式   #define XDP_FLAGS_DRV_MODE  (1U << 2)// 默认的工作模式   #define XDP_FLAGS_HW_MODE  (1U << 3)// offloaded XDP   #define XDP_FLAGS_MODES   (XDP_FLAGS_SKB_MODE | \         XDP_FLAGS_DRV_MODE | \         XDP_FLAGS_HW_MODE)   #define XDP_FLAGS_MASK   (XDP_FLAGS_UPDATE_IF_NOEXIST | \         XDP_FLAGS_MODES)   `

## 3. 读取数据

使用bpf_map_lookup_elem函数。

`bpf_map_lookup_elem(map_data[0].fd,&key,&p);   `

与内核函数区别是参数一使用文件描述符引用映射

`map_data[0].fd   `

内核使用map_data全局变量保存BPF映射信息，该变量为bpf_map_data结构数组，按照程序中指定映射的顺序排序。

> samples/bpf/bpf_load.h

`struct bpf_map_data {    int fd;    char *name;    size_t elf_offset;    struct bpf_map_def def;   };   `

该结构存储map描述符和名字

# 运行测试

## 1. 未挂载xdp程序

使用wireshark抓包，可以看到正常通信

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

xdp_before

## 2. 启动xdp程序

可以看到除了IP为10.211.55.12其余IP都被过滤，且通过访问8090端口发现，其端口更改为8080

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

xdp_after

## 3. 遇到的问题

wireshark显示，此处更改端口为8080数据包出现问题，不能正常访问8080端口资源

Reads 3091

​

Comment

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/SeWfibBcBT0EibtIWVNvshnuWMN1AoJw3poFIsbpaIVyZibCCqwBUR21rcDfrQgoqYzaYNdS14IIXzvmzvibdDa5Rw/300?wx_fmt=png&wxfrom=18)

Linux内核之旅

715

Comment

Comment

**Comment**

暂无留言
