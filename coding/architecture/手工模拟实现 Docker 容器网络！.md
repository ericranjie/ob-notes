# 

原创 张彦飞allen 开发内功修炼

_2021年11月16日 08:28_

大家好，我是飞哥！

如今服务器虚拟化技术已经发展到了深水区。现在业界已经有很多公司都迁移到容器上了。我们的开发写出来的代码大概率是要运行在容器上的。因此深刻理解容器网络的工作原理非常的重要。只有这样将来遇到问题的时候才知道该如何下手处理。

网络虚拟化，其实用一句话来概括就是**用软件来模拟实现真实的物理网络连接**。比如 Docker 就是用纯软件的方式在宿主机上模拟出来的独立网络环境。我们今天来徒手打造一个虚拟网络，实现在这个网络里访问外网资源，同时监听端口提供对外服务的功能。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

看完这一篇后，相信你对 Docker 虚拟网络能有进一步的理解。好了，我们开始！

## 一、基础知识回顾

### 1.1 veth、bridge 与 namespace

Linux 下的 **veth** 是一对儿虚拟网卡设备，和我们常见的 lo 很类似。在这儿设备里，从一端发送数据后，内核会寻找该设备的另一半，所以在另外一端就能收到。不过 veth 只能解决一对一通信的问题。详情参见[轻松理解 Docker 网络虚拟化基础之 veth 设备！](https://mp.weixin.qq.com/s?__biz=MjM5Njg5NDgwNA==&mid=2247486424&idx=1&sn=d66fe4ebf1cd9e5079606f71a0169697&scene=21#wechat_redirect)

如果有很多对儿 veth 需要互相通信的话，就需要引入 **bridge** 这个虚拟交换机。各个 veth 对儿可以把一头连接在 bridge 的接口上，bridge 可以和交换机一样在端口之间转发数据，使得各个端口上的 veth 都可以互相通信。参见

**Namespace** 解决的是隔离性的问题。每个虚拟网卡设备、进程、socket、路由表等等网络栈相关的对象默认都是归属在 init_net 这个缺省的 namespace 中的。不过我们希望不同的虚拟化环境之间是隔离的，用 Docker 来举例，那就是不能让 A 容器用到 B 容器的设备、路由表、socket 等资源，甚至连看一眼都不可以。只有这样才能保证不同的容器之间复用资源的同时，还不会影响其它容器的正常运行。参见

通过 veth、namespace 和 bridge 我们在一台 Linux 上就能虚拟多个网络环境出来。而且它们之间、和宿主机之间都可以互相通信。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

关于这三个技术的详情，可以参考下面这三篇文章：

- [轻松理解 Docker 网络虚拟化基础之 veth 设备！](https://mp.weixin.qq.com/s?__biz=MjM5Njg5NDgwNA==&mid=2247486424&idx=1&sn=d66fe4ebf1cd9e5079606f71a0169697&scene=21#wechat_redirect)

- [聊聊 Linux 上软件实现的“交换机” - Bridge！](https://mp.weixin.qq.com/s?__biz=MjM5Njg5NDgwNA==&mid=2247486949&idx=1&sn=51bf822b1ee4f6a667d6253965d49201&scene=21#wechat_redirect)

- [彻底弄懂 Linux 网络命名空间](https://mp.weixin.qq.com/s?__biz=MjM5Njg5NDgwNA==&mid=2247487019&idx=1&sn=a0cbac49913071cd13e64ad5abd16d5c&scene=21#wechat_redirect)

但是这三篇文章过后，我们还剩下一个问题没有解决，那就是虚拟出来的网络环境和外部网络的通信。还拿 Docker 容器来举例，你启动的容器里的服务肯定是需要访问外部的数据库的。还有就是可能需要暴露比如 80 端口对外提供服务。例如在 Docker 中我们通过下面的命令将容器的 80 端口上的 web 服务要能被外网访问的到。

我们今天的文章主要就是解决这两个问题的，一是从虚拟网络中访问外网，二是在虚拟网络中提供服务供外网使用。解决它们需要用到**路由**和 **nat** 技术。

### 1.2 路由选择

Linux 是在发送数据包的时候，会涉及到路由过程。这个发送数据包既包括本机发送数据包，也包括途径当前机器的数据包的转发。

先来看本机发送数据包。其中本机发送在[25 张图，一万字，拆解 Linux 网络包发送过程](https://mp.weixin.qq.com/s?__biz=MjM5Njg5NDgwNA==&mid=2247485146&idx=1&sn=e5bfc79ba915df1f6a8b32b87ef0ef78&scene=21#wechat_redirect)这一篇我们讨论过。

\*\*所谓路由其实很简单，就是该选择哪张网卡（虚拟网卡设备也算）将数据写进去。\*\*到底该选择哪张网卡呢，规则都是在路由表中指定的。Linux 中可以有多张路由表，最重要和常用的是 local 和 main。

local 路由表中统一记录本地，确切的说是本网络命名空间中的网卡设备 IP 的路由规则。

`#ip route list table local   local 10.143.x.y dev eth0 proto kernel scope host src 10.143.x.y   local 127.0.0.1 dev lo proto kernel scope host src 127.0.0.1   `

其它的路由规则，一般都是在 main 路由表中记录着的。可以用 `ip route list table local` 查看，也可以用更简短的 `route -n`

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

再看途径当前机器的数据包的转发。除了本机发送以外，转发也会涉及路由过程。如果 Linux 收到数据包以后发现目的地址并不是本地的地址的话，就可以选择把这个数据包从自己的某个网卡设备上转发出去。这个时候和本机发送一样，也需要读取路由表。根据路由表的配置来选择从哪个设备将包转走。

不过值得注意的是，Linux 上转发功能默认是关闭的。也就是发现目的地址不是本机 IP 地址默认是将包直接丢弃。需要做一些简单的配置，然后 Linux 才可以干像路由器一样的活儿，实现数据包的转发。

### 1.3 iptables 与 NAT

Linux 内核网络栈在运行上基本上是一个纯内核态的东西，但为了迎合各种各样用户层不同的需求，内核开放了一些口子出来供用户层来干预。其中 iptables 就是一个非常常用的干预内核行为的工具，它在内核里埋下了五个钩子入口，这就是俗称的五链。

Linux 在接收数据的时候，在 IP 层进入 ip_rcv 中处理。再执行路由判断，发现是本机的话就进入 ip_local_deliver 进行本机接收，最后送往 TCP 协议层。在这个过程中，埋了两个 HOOK，第一个是 PRE_ROUTING。这段代码会执行到 iptables 中 pre_routing 里的各种表。发现是本地接收后接着又会执行到 LOCAL_IN，这会执行到 iptables 中配置的 input 规则。

在发送数据的时候，查找路由表找到出口设备后，依次通过 \_\_ip_local_out、 ip_output 等函数将包送到设备层。在这两个函数中分别过了 OUTPUT 和 POSTROUTING 开的各种规则。

如果是转发过程，Linux 收到数据包发现不是本机的包可以通过查找自己的路由表找到合适的设备把它转发出去。那就先是在 ip_rcv 中将包送到 ip_forward 函数中处理，最后在 ip_output 函数中将包转发出去。在这个过程中分别过了 PREROUTING、FORWARD 和 POSTROUTING 三个规则。

综上所述，iptables 里的五个链在内核网络模块中的位置就可以归纳成如下这幅图。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

数据接收过程走的是 1 和 2，发送过程走的是 4 、5，转发过程是 1、3、5。有了这张图，我们能更清楚地理解 iptable 和内核的关系。

在 iptables 中，根据实现的功能的不同，又分成了四张表。分别是 raw、mangle、nat 和 filter。其中 nat 表实现我们常说的 NAT（Network AddressTranslation） 功能。其中 nat 又分成 SNAT（Source NAT）和 DNAT（Destination NAT）两种。

SNAT 解决的是内网地址访问外部网络的问题。它是通过在 POSTROUTING 里修改来源 IP 来实现的。

DNAT 解决的是内网的服务要能够被外部访问到的问题。它在通过 PREROUTING 修改目标 IP 实现的。

## 二、 实现虚拟网络外网通信

基于以上的基础知识，我们用纯手工的方式搭建一个可以和 Docker 类似的虚拟网络。而且要实现和外网通信的功能。

### 1. 实验环境准备

我们先来创建一个虚拟的网络环境出来，其命名空间为 net1。宿主机的 IP 是 10.162 的网段，可以访问外部机器。虚拟网络为其分配 192.168.0 的网段，这个网段是私有的，外部机器无法识别。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

这个虚拟网络的搭建过程如下。先创建一个 netns 出来，命名为 net1。

`# ip netns add net1   `

创建一个 veth 对儿（veth1 - veth1_p），把其中的一头 veth1 放在 net1 中，给它配置上 IP，并把它启动起来。

`# ip link add veth1 type veth peer name veth1_p   # ip link set veth1 netns net1   # ip netns exec net1 ip addr add 192.168.0.2/24 dev veth1  # IP   # ip netns exec net1 ip link set veth1 up   `

创建一个 bridge，给它也设置上 ip。接下来把 veth 的另外一端 veth1_p 插到 bridge 上面。最后把网桥和 veth1_p 都启动起来。

`# brctl addbr br0   # ip addr add 192.168.0.1/24 dev br0   # ip link set dev veth1_p master br0   # ip link set veth1_p up   # ip link set br0 up   `

这样我们就在 Linux 上创建出了一个虚拟的网络。创建过程和 [聊聊 Linux 上软件实现的“交换机” - Bridge！](https://mp.weixin.qq.com/s?__biz=MjM5Njg5NDgwNA==&mid=2247486949&idx=1&sn=51bf822b1ee4f6a667d6253965d49201&scene=21#wechat_redirect) 中一样，只不过今天为了省事，只创建了一个网络出来，上一篇中创建出来了两个。

### 2. 请求外网资源

现在假设我们上面的 net1 这个网络环境中想访问外网。这里的外网是指的虚拟网络宿主机外部的网络。

我们假设它要访问的另外一台机器 IP 是 10.153.*\_.*\_ ，这个 10.153._*.*_ 后面两段由于是我的内部网络，所以隐藏起来了。你在实验的过程中，用自己的 IP 代替即可。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

我们直接来访问一下试试

`# ip netns exec net1 ping 10.153.*.*   connect: Network is unreachable   `

提示网络不通，这是怎么回事？用这段报错关键字在内核源码里搜索一下：

`//file: arch/parisc/include/uapi/asm/errno.h   #define ENETUNREACH 229 /* Network is unreachable */      //file: net/ipv4/ping.c   static int ping_sendmsg(struct kiocb *iocb, struct sock *sk, struct msghdr *msg,      size_t len)   {    ...    rt = ip_route_output_flow(net, &fl4, sk);    if (IS_ERR(rt)) {     err = PTR_ERR(rt);     rt = NULL;     if (err == -ENETUNREACH)      IP_INC_STATS_BH(net, IPSTATS_MIB_OUTNOROUTES);     goto out;    }    ...   out:     return err;    }   `

在 ip_route_output_flow 这里的返回值判断如果是 ENETUNREACH 就退出了。这个宏定义注释上来看报错的信息就是 “Network is unreachable”。

这个 ip_route_output_flow 主要是执行路由选路。所以我们推断可能是路由出问题了，看一下这个命名空间的路由表。

`# ip netns exec net1 route -n   Kernel IP routing table   Destination     Gateway         Genmask         Flags Metric Ref    Use Iface   192.168.0.0     0.0.0.0         255.255.255.0   U     0      0        0 veth1   `

怪不得，原来 net1 这个 namespace 下默认只有 192.168.0.\* 这个网段的路由规则。我们 ping 的 IP 是 10.153.*\_.*\_ ，根据这个路由表里找不到出口。自然就发送失败了。

我们来给 net 添加上默认路由规则，只要匹配不到其它规则就默认送到 veth1 上，同时指定下一条是它所连接的 bridge（192.168.0.1）。

`# ip netns exec net1 route add default gw 192.168.0.1 veth1`

再 ping 一下试试。

`# ip netns exec net1 ping 10.153.*.* -c 2   PING 10.153.*.* (10.153.*.*) 56(84) bytes of data.      --- 10.153.*.* ping statistics ---   2 packets transmitted, 0 received, 100% packet loss, time 999ms   `

额好吧，仍然不通。上面路由帮我们把数据包从 veth 正确送到了 bridge 这个网桥上。接下来网桥还需要 bridge 转发到 eth0 网卡上。所以我们得打开下面这两个转发相关的配置

`# sysctl net.ipv4.conf.all.forwarding=1   # iptables -P FORWARD ACCEPT   `

不过这个时候，还存在一个问题。那就是外部的机器并不认识 192.168.0.\* 这个网段的 ip。它们之间都是通过 10.153.*\_.*\_ 来进行通信的。设想下我们工作中的电脑上没有外网 IP 的时候是如何正常上网的呢？外部的网络只认识外网 IP。没错，那就是我们上面说的 NAT 技术。

我们这次的需求是实现内部虚拟网络访问外网，所以需要使用的是 SNAT。它将 namespace 请求中的 IP（192.168.0.2）换成外部网络认识的 10.153.*.*，进而达到正常访问外部网络的效果。

`# iptables -t nat -A POSTROUTING -s 192.168.0.0/24 ! -o br0 -j MASQUERADE   `

来再 ping 一下试试，欧耶，通了！

`# ip netns exec net1 ping 10.153.*.*   PING 10.153.*.* (10.153.*.*) 56(84) bytes of data.   64 bytes from 10.153.*.*: icmp_seq=1 ttl=57 time=1.70 ms   64 bytes from 10.153.*.*: icmp_seq=2 ttl=57 time=1.68 ms   `

这时候我们可以开启 tcpdump 抓包查看一下，在 bridge 上抓到的包我们能看到还是原始的源 IP 和 目的 IP。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

再到 eth0 上查看的话，源 IP 已经被替换成可和外网通信的 eth0 上的 IP 了。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

至此，容器就可以通过宿主机的网卡来访问外部网络上的资源了。我们来总结一下这个发送过程

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

## 3. 开放容器端口

我们再考虑另外一个需求，那就是把在这个命名空间内的服务提供给外部网络来使用。

和上面的问题一样，我们的虚拟网络环境中 192.168.0.2 这个 IP 外界是不认识它的。只有这个宿主机知道它是谁。所以我们同样还需要 NAT 功能。

这次我们是要实现外部网络访问内部地址，所以需要的是 DNAT 配置。DNAT 和 SNAT 配置中有一个不一样的地方就是需要明确指定容器中的端口在宿主机上是对应哪个。比如在 docker 的使用中，是通过 -p 来指定端口的对应关系。

`# docker run -p 8000:80 ...   `

我们通过如下这个命令来配置 DNAT 规则

`# iptables -t nat -A PREROUTING  ! -i br0 -p tcp -m tcp --dport 8088 -j DNAT --to-destination 192.168.0.2:80   `

这里表示的是宿主机在路由之前判断一下如果流量不是来自 br0，并且是访问 tcp 的 8088 的话，那就转发到 192.168.0.2:80 。

在 net1 环境中启动一个 Server

`# ip netns exec net1 nc -lp 80   `

外部选一个ip，比如 10.143.*.*， telnet 连一下 10.162.*.* 8088 试试，通了！

`# telnet 10.162.*.* 8088   Trying 10.162.*.*...   Connected to 10.162.*.*.   Escape character is '^]'.   `

开启抓包， `# tcpdump -i eth0 host 10.143.*.*`。可见在请求的时候，目的是宿主机的 IP 的端口。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

但数据包到宿主机协议栈以后命中了我们配置的 DNAT 规则，宿主机把它转发到了 br0 上。在 bridge 上由于没有那么多的网络流量包，所以不用过滤直接抓包就行，`# tcpdump -i br0`。

在 br0 上抓到的目的 IP 和端口是已经替换过的了。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

bridge 当然知道 192.168.0.2 是 veth 1。于是，在 veth1 上监听 80 的服务就能收到来自外界的请求了！我们来总结一下这个接收过程

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

## 三、总结

现在业界已经有很多公司都迁移到容器上了。我们的开发写出来的代码大概率是要运行在容器上的。因此深刻理解容器网络的工作原理非常的重要。这有这样将来遇到问题的时候才知道该如何下手处理。

本文开头我们先是简单介绍了 veth、bridge、namespace、路由、iptables 等基础知识。Veth 实现连接，bridge 实现转发，namespace 实现隔离，路由表控制发送时的设备选择，iptables 实现 nat 等功能。

接着基于以上基础知识，我们采用纯手工的方式搭建了一个虚拟网络环境。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

这个虚拟网络可以访问外网资源，也可以提供端口服务供外网来调用。这就是 Docker 容器网络工作的基本原理。

整个实验我打包写成一个 Makefile，放到了这里：https://github.com/yanfeizhang/coder-kung-fu/tree/main/tests/network/test07

最后，我们再扩展一下。今天我们讨论的问题是 Docker 网络通信的问题。Docker 容器通过端口映射的方式提供对外服务。外部机器访问容器服务的时候，仍然需要通过容器的宿主机 IP 来访问。

在 Kubernets 中，对跨主网络通信有更高的要求，要**不同宿主机之间的容器可以直接互联互通**。所以 Kubernets 的网络模型也更为复杂。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

![](https://mmbiz.qlogo.cn/mmbiz_jpg/iaMl8iazctEu93J6Io4KDZQmx03HErmVeXnSnUAQ0MD6Nia3d97E8vmljv5DibLTWSeLES1JHicWVA2mynPVlbt1FAA/0?wx_fmt=jpeg)

张彦飞allen

赞赏不分多少，头像出现就好！

![赞赏二维码](https://mp.weixin.qq.com/s?__biz=MjM5Njg5NDgwNA==&mid=2247487294&idx=1&sn=c03e1dfbb98075390d562cabaeac62c8&chksm=a6e30e0591948713ff1e03335d8d6292c5745a6c7044deeee1b054190310e46e995cef003026&mpshare=1&scene=24&srcid=1116LUPnA1mjyF39Me8uGbP4&sharer_sharetime=1637042342312&sharer_shareid=5fb9813bfe9ffc983435bfc8d8c5e9ca&key=daf9bdc5abc4e8d014d41f63c52cffd0fb424f489e9ce93fd6fc7ea4c3363681f8e4ec230c99dfd5346f45c41c0c26ce23f420936a0c45803671d93d8aa38efdaf800952f677356fa7d7ef0077ebe09ec2f2f64b10c26f928af2bd265f3729aacee20b12365b00dae4503404b67c5620dbdd269b6064c477389bc3dd92eeabc2&ascene=0&uin=MTEwNTU1MjgwMw%3D%3D&devicetype=Windows+11+x64&version=63090b19&lang=zh_CN&countrycode=CN&exportkey=n_ChQIAhIQQrsr1l%2F1%2B6m1%2B1EjFxUxThLmAQIE97dBBAEAAAAAAAQgOaerwt8AAAAOpnltbLcz9gKNyK89dVj0EXxii0x7rCYDPk5v9j12MRscu9uoMGq3ppHJoEOwgNVdLXqa36lD97wUV6pday85asx%2FPAo3p%2Bgs3Mxq93Ct%2FjZj4UwQTKgEX5%2BLMa1lRFDE6KyXtiSx6YEApyP0sHsGSNDZ1rliFUG2yWEmKsgyMj1Kp%2BVAQwmBLOzVxO7jMzMk1dCM2Oui4%2Fir7oLMpxRb3Zf1VS9rb%2B3gryAVItrWzdjXP9easqax00VYk%2Bp0%2FjFpDkyUy%2F88SIspWhbdBQD7&acctmode=0&pass_ticket=7zFlKK1V6YGtl89banzDdX1RHk4h1wOtw60R850vh4csPwiklbk75GULbZ7U839k&wx_header=1&fasttmpl_type=0&fasttmpl_fullversion=7351805-zh_CN-zip&fasttmpl_flag=1)喜欢作者

4人喜欢

![](http://wx.qlogo.cn/mmopen/Q3auHgzwzM6uZ8AHaMjXkpbOguAJKSLBNpEMnpzOW6fzJsf77ymYCqqibaYxWEMGAf9sKPXkmIoibzoDpJVPgpXHIgyiatwhXJYTg64vbA3Nzo/64)![](http://wx.qlogo.cn/mmopen/PiajxSqBRaEIRIJlJh45sMhe6pkLediaRBSLegcVX48SGFzZLWZlPzkWyPbMEKhPPqlEcXqfJXPoQRZibxVPQTVUw/64)![](http://wx.qlogo.cn/mmopen/Sb650G6FHc1EYg3PxSSCAqh3IwsdGS5hNRMyHAnAGapX4So4btdV2yqbtmq5mIJXhkmwGVgvbeiaeyDCD98e2KYmxtVLy5caT/64)![](http://wx.qlogo.cn/mmopen/ukUXYlZ023mgshW4eJlrohxs6S14PVSwn3zro8B9CKDNbM4IX0C8XApiamWVrtZQXt7JOo8MwqKsEfh974ib4cxDLm6cicgNEo3/64)

docker1

linux3

网络2

开发内功修炼之网络篇42

服务器2

阅读 5587

修改于2021年11月23日

​

写留言

**留言 33**

- ![[咖啡]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)卡布奇诺

  2021年11月16日

  赞4

  非常强

  开发内功修炼

  作者2021年11月16日

  赞1

  🤝

- 周萝卜

  2021年11月16日

  赞3

  太过分了，怎么可以这么强

  开发内功修炼

  作者2021年11月16日

  赞1

  ![😂](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

- 晨光

  2021年11月16日

  赞2

  牛牛牛

  开发内功修炼

  作者2021年11月16日

  赞1

  感谢

- cheerful_Dustin

  2021年11月16日

  赞2

  很强

- Dr.Huo

  2021年11月16日

  赞2

  沙发。。。

  开发内功修炼

  作者2021年11月16日

  赞2

  稳了

- 波

  2021年12月5日

  赞

  容器访问外网需要转发到eth0，并且SNAT，为什么ping的回应到达eth0时，我们不需要手动定义DNAT来转发到br0然后到达容器呢？查了其他网说主机会自动记录映射，不太明白什么映射，icmp头信息什么的？

  开发内功修炼

  作者2021年12月6日

  赞1

  这个特性叫链接跟踪，你可以理解成snat这次的替换会记录下来，包括端口啥的。等回包到达，就能知道哪个包是需要反替换的。具体源码我还没查，不过大致应该是这么个过程。

- Xin.Guo

  2021年11月19日

  赞

  飞哥写的很好，不过对于ping报文从br0到eth1有跳跃，我分析了一下，由于东西多，评论区写不下，所以写了一篇文章https://mp.weixin.qq.com/s/uwYO9zJG6NhUMUZ5nkd5Kw，烦请指正。

  开发内功修炼

  作者2021年11月22日

  赞1

  赞！

- coincidence

  2021年11月18日

  赞1

  蹲飞哥分析k8s的calico、flannel、canal![[Onlooker]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)。

- 爱挖辣辣的少年

  2021年11月17日

  赞

  在发送数据的时候，查找路由表找到出口设备后，依次通过 \_\_ip_local_out、 ip_output 等函数将包送到设备层。在这两个函数中分别过了 OUTPUT 和 PREROUTING 开的各种规则。 ——————————- 最后一句，是不是应该是postrouting，而不是prerouting

  开发内功修炼

  作者2021年11月17日

  赞1

  感谢指出，等我有空看看

  开发内功修炼

  作者2021年11月23日

  赞

  已经更正

- orz

  上海2023年11月29日

  赞

  非常强，深入浅出![[强]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![[强]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

  开发内功修炼

  作者2023年11月29日

  赞

  🤝

- gk

  广东2023年3月20日

  赞

  它将 namespace 请求中的 IP（192.168.0.2）换成外部网络认识的 10.153.*.*，进而达到正常访问外部网络的效果。 ———————————————————— 是不是打错了？10.153.*.*改为10.162.*.*？

  开发内功修炼

  作者2023年3月20日

  赞

  感谢指出，确实应该是这样

- 星星不说话

  北京2022年8月3日

  赞

  飞哥，有一点不明白，veth 是一对，宿主机一个，容器内一个，容器内的veth是在哪个网络命名空间呐，在宿主机没有找到

  开发内功修炼

  作者2022年8月3日

  赞

  容器是一个独立的命名空间，所以得在容器的命名空间里看

  星星不说话

  北京2022年8月3日

  赞

  飞哥，我还是不明白，如果说 nsenter -t {PID} -n 就已经进入到容器的网络命名空间了，那在宿主机执行ip netns show 命令为何结果为空，不是应该有个类似文章中net1的命名空间吗？

  开发内功修炼

  作者2022年8月4日

  赞

  新版默认都把命名空间信息文件删了，你搜下"查看 Docker 容器的名字空间"，就能找到查看的办法了

  星星不说话

  2022年8月4日

  赞

  好的，感谢飞哥![🙏](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

- LG

  2021年11月17日

  赞

  太好玩了！

  LG

  2021年11月17日

  赞

  建议多来，寓教于乐

- 二马

  2021年11月16日

  赞

  无比佩服

  开发内功修炼

  作者2021年11月17日

  赞

  🤝

- 钟

  2021年11月16日

  赞

  强大的魔法攻击加成

- 淫角大王

  2021年11月16日

  赞

  各位如果在做第二小节实验时发现即使添加SNAT规则自定义的netns还是访问不了外网可能是因为FORWARD链默认是DROP的原因，两种解决办法：1.更改forward链默认处理规则为ACCEPT；2.在forward链中添加规则允许br0和物理网卡接口之间互相转发。

  开发内功修炼

  作者2021年11月16日

  赞

  👍🏻

已无更多数据

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/BBjAFF4hcwqciaicnxpic7g4KKVZKSeQTzic5FoaXsQeLG979y9iaTNVPKTSO9BzSM7zXoJzibbBtEyb6Df5OTj0Yw3Q/300?wx_fmt=png&wxfrom=18)

开发内功修炼

391318

33

写留言

**留言 33**

- ![[咖啡]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)卡布奇诺

  2021年11月16日

  赞4

  非常强

  开发内功修炼

  作者2021年11月16日

  赞1

  🤝

- 周萝卜

  2021年11月16日

  赞3

  太过分了，怎么可以这么强

  开发内功修炼

  作者2021年11月16日

  赞1

  ![😂](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

- 晨光

  2021年11月16日

  赞2

  牛牛牛

  开发内功修炼

  作者2021年11月16日

  赞1

  感谢

- cheerful_Dustin

  2021年11月16日

  赞2

  很强

- Dr.Huo

  2021年11月16日

  赞2

  沙发。。。

  开发内功修炼

  作者2021年11月16日

  赞2

  稳了

- 波

  2021年12月5日

  赞

  容器访问外网需要转发到eth0，并且SNAT，为什么ping的回应到达eth0时，我们不需要手动定义DNAT来转发到br0然后到达容器呢？查了其他网说主机会自动记录映射，不太明白什么映射，icmp头信息什么的？

  开发内功修炼

  作者2021年12月6日

  赞1

  这个特性叫链接跟踪，你可以理解成snat这次的替换会记录下来，包括端口啥的。等回包到达，就能知道哪个包是需要反替换的。具体源码我还没查，不过大致应该是这么个过程。

- Xin.Guo

  2021年11月19日

  赞

  飞哥写的很好，不过对于ping报文从br0到eth1有跳跃，我分析了一下，由于东西多，评论区写不下，所以写了一篇文章https://mp.weixin.qq.com/s/uwYO9zJG6NhUMUZ5nkd5Kw，烦请指正。

  开发内功修炼

  作者2021年11月22日

  赞1

  赞！

- coincidence

  2021年11月18日

  赞1

  蹲飞哥分析k8s的calico、flannel、canal![[Onlooker]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)。

- 爱挖辣辣的少年

  2021年11月17日

  赞

  在发送数据的时候，查找路由表找到出口设备后，依次通过 \_\_ip_local_out、 ip_output 等函数将包送到设备层。在这两个函数中分别过了 OUTPUT 和 PREROUTING 开的各种规则。 ——————————- 最后一句，是不是应该是postrouting，而不是prerouting

  开发内功修炼

  作者2021年11月17日

  赞1

  感谢指出，等我有空看看

  开发内功修炼

  作者2021年11月23日

  赞

  已经更正

- orz

  上海2023年11月29日

  赞

  非常强，深入浅出![[强]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![[强]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

  开发内功修炼

  作者2023年11月29日

  赞

  🤝

- gk

  广东2023年3月20日

  赞

  它将 namespace 请求中的 IP（192.168.0.2）换成外部网络认识的 10.153.*.*，进而达到正常访问外部网络的效果。 ———————————————————— 是不是打错了？10.153.*.*改为10.162.*.*？

  开发内功修炼

  作者2023年3月20日

  赞

  感谢指出，确实应该是这样

- 星星不说话

  北京2022年8月3日

  赞

  飞哥，有一点不明白，veth 是一对，宿主机一个，容器内一个，容器内的veth是在哪个网络命名空间呐，在宿主机没有找到

  开发内功修炼

  作者2022年8月3日

  赞

  容器是一个独立的命名空间，所以得在容器的命名空间里看

  星星不说话

  北京2022年8月3日

  赞

  飞哥，我还是不明白，如果说 nsenter -t {PID} -n 就已经进入到容器的网络命名空间了，那在宿主机执行ip netns show 命令为何结果为空，不是应该有个类似文章中net1的命名空间吗？

  开发内功修炼

  作者2022年8月4日

  赞

  新版默认都把命名空间信息文件删了，你搜下"查看 Docker 容器的名字空间"，就能找到查看的办法了

  星星不说话

  2022年8月4日

  赞

  好的，感谢飞哥![🙏](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

- LG

  2021年11月17日

  赞

  太好玩了！

  LG

  2021年11月17日

  赞

  建议多来，寓教于乐

- 二马

  2021年11月16日

  赞

  无比佩服

  开发内功修炼

  作者2021年11月17日

  赞

  🤝

- 钟

  2021年11月16日

  赞

  强大的魔法攻击加成

- 淫角大王

  2021年11月16日

  赞

  各位如果在做第二小节实验时发现即使添加SNAT规则自定义的netns还是访问不了外网可能是因为FORWARD链默认是DROP的原因，两种解决办法：1.更改forward链默认处理规则为ACCEPT；2.在forward链中添加规则允许br0和物理网卡接口之间互相转发。

  开发内功修炼

  作者2021年11月16日

  赞

  👍🏻

已无更多数据
