Linux云计算网络
_2022年01月17日 08:13_
原文链接：www.ttlsa.com/linux-command/ss-replace-netstat/

ss命令用于显示socket状态。他可以显示 PACKET sockets，TCP sockets，UDP sockets，DCCP sockets，RAW sockets，Unix domain sockets等等统计。它比其他工具展示等多tcp和state信息。

它是一个非常实用、快速、有效的跟踪IP连接和sockets的新工具。SS命令可以提供如下信息：

- 所有的TCP sockets
- 所有的UDP sockets
- 所有ssh/ftp/ttp/https持久连接
- 所有连接到Xserver的本地进程
- 使用 state（例如：connected，synchronized，SYN-RECV，SYN-SENT，TIME-WAIT）、地址、端口过滤
- 所有的 state FIN-WAIT-1 tcpsocket 连接以及更多

很多流行的 Linux 发行版都支持 ss 以及很多监控工具使用ss命令。熟悉这个工具有助于您更好的发现与解决系统性能问题。本人强烈建议使用ss命令替代 netstat 部分命令，例如netsat -ant/lnt等。

展示他之前来做个对比，统计服务器并发连接数

```c
netstat  
# time netstat -ant | grep EST | wc -l  
3100  
  
real 0m12.960s  
user 0m0.334s  
sys 0m12.561s  
# time ss -o state established | wc -l  
3204  
  
real 0m0.030s  
user 0m0.005s  
sys 0m0.026s
```

结果很明显ss统计并发连接数效率完胜netstat，在ss能搞定的情况下，你还会在选择netstat吗，还在犹豫吗，看以下例子，或者跳转到帮助页面。

常用ss命令：

```c
ss -l 显示本地打开的所有端口  
ss -pl 显示每个进程具体打开的socket  
ss -t -a 显示所有tcp socket  
ss -u -a 显示所有的UDP Socekt  
ss -o state established '( dport = :smtp or sport = :smtp )' 显示所有已建立的SMTP连接  
ss -o state established '( dport = :http or sport = :http )' 显示所有已建立的HTTP连接  
ss -x src /tmp/.X11-unix/* 找出所有连接X服务器的进程  
ss -s 列出当前socket详细信息:
```

显示sockets简要信息，列出当前已经连接，关闭，等待的tcp连接

```c
# ss -s  
Total: 3519 (kernel 3691)  
TCP: 26557 (estab 3163, closed 23182, orphaned 194, synrecv 0, timewait 23182/0), ports 1452  
  
Transport Total IP IPv6  
* 3691 - -  
RAW 2 2 0  
UDP 10 7 3  
TCP 3375 3368 7  
INET 3387 3377 10  
FRAG 0 0 0
```

列出当前监听端口

```c
# ss -lRecv-Q Send-Q Local Address:Port Peer Address:Port  
0 10 :::5989 :::*  
0 5 *:rsync *:*  
0 128 :::sunrpc :::*  
0 128 *:sunrpc *:*  
0 511 *:http *:*  
0 128 :::ssh :::*  
0 128 *:ssh *:*  
0 128 :::35766 :::*  
0 128 127.0.0.1:ipp *:*  
0 128 ::1:ipp :::*  
0 100 ::1:smtp :::*  
0 100 127.0.0.1:smtp *:*  
0 511 *:https *:*  
0 100 :::1311 :::*  
0 5 *:5666 *:*  
0 128 *:3044 *:*
```

ss列出每个进程名及其监听的端口

```c
# ss -pl
```

ss列所有的tcp sockets

```c
# ss -t -a
```

ss列出所有udp sockets

```c
# ss -u -a
```

ss列出所有http连接中的连接

```c
# ss -o state established '( dport = :http or sport = :http )'
```

- 以上包含对外提供的80，以及访问外部的80

- 用以上命令完美的替代netstat获取http并发连接数，监控中常用到

ss列出本地哪个进程连接到x server

```c
# ss -x src /tmp/.X11-unix/*
```

ss列出处在FIN-WAIT-1状态的http、https连接

```c
# ss -o state fin-wait-1 '( sport = :http or sport = :https )'
```

ss常用的state状态：

```c
established  
syn-sent  
syn-recv  
fin-wait-1  
fin-wait-2  
time-wait  
closed  
close-wait  
last-ack  
listen  
closing  
all : All of the above states  
connected : All the states except for listen and closed  
synchronized : All the connected states except for syn-sent  
bucket : Show states, which are maintained as minisockets, i.e. time-wait and syn-recv.  
big : Opposite to bucket state.
```

ss使用IP地址筛选

```c
ss src ADDRESS_PATTERN  
src：表示来源  
ADDRESS_PATTERN：表示地址规则  
如下：  
ss src 120.33.31.1 # 列出来之20.33.31.1的连接  
  
＃ 列出来至120.33.31.1,80端口的连接  
ss src 120.33.31.1:http  
ss src 120.33.31.1:8
```

ss使用端口筛选

```c
ss dport OP PORT  
OP:是运算符  
PORT：表示端口  
dport：表示过滤目标端口、相反的有sport
```

OP运算符如下：

```c
<= or le : 小于等于 >= or ge : 大于等于  
== or eq : 等于  
!= or ne : 不等于端口  
< or lt : 小于这个端口 > or gt : 大于端口
```

OP实例

```c
ss sport = :http 也可以是 ss sport = :80  
ss dport = :http  
ss dport \> :1024  
ss sport \> :1024  
ss sport \< :32000  
ss sport eq :22  
ss dport != :22  
ss state connected sport = :http  
ss \( sport = :http or sport = :https \)  
ss -o state fin-wait-1 \( sport = :http or sport = :https \) dst 192.168.
```

为什么ss比netstat快：

netstat是遍历/proc下面每个PID目录，ss直接读/proc/net下面的统计信息。所以ss执行的时候消耗资源以及消耗的时间都比netstat少很多

ss命令帮助

```c
# ss -h  
Usage: ss [ OPTIONS ]  
       ss [ OPTIONS ] [ FILTER ]  
   -h, --help           this message  
   -V, --version        output version information  
   -n, --numeric        don't resolve service names  
   -r, --resolve       resolve host names  
   -a, --all            display all sockets  
   -l, --listening      display listening sockets  
   -o, --options       show timer information  
   -e, --extended      show detailed socket information  
   -m, --memory        show socket memory usage  
   -p, --processes      show process using socket  
   -i, --info           show internal TCP information  
   -s, --summary        show socket usage summary  
   -4, --ipv4          display only IP version 4 sockets  
   -6, --ipv6          display only IP version 6 sockets  
   -0, --packet display PACKET sockets  
   -t, --tcp            display only TCP sockets  
   -u, --udp            display only UDP sockets  
   -d, --dccp           display only DCCP sockets  
   -w, --raw            display only RAW sockets  
   -x, --unix           display only Unix domain sockets  
   -f, --family=FAMILY display sockets of type FAMILY  
   -A, --query=QUERY, --socket=QUERY  
       QUERY := {all|inet|tcp|udp|raw|unix|packet|netlink}[,QUERY]  
   -D, --diag=FILE      Dump raw information about TCP sockets to FILE  
   -F, --filter=FILE   read filter information from FILE  
       FILTER := [ state TCP-STATE ] [ EXPRESSION ]
```

______________________________________________________________________

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

_\*\*_****喜欢，就给我一个****“在看”\*\*\*\*_\*\*_

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**10T 技术资源大放送！包括但不限于：云计算、虚拟化、微服务、大数据、网络、**Linux、**Docker、Kubernetes、Python、Go、C/C++、Shell、PPT 等。在公众号内回复「****1024****」**，即可免费获！！****

阅读 1536

​
