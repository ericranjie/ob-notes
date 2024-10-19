
Linux云计算网络 _2021年09月09日 08:13_

The following article is from 赐我白日梦 Author 赐我白日梦

# 一、iptables是什么？你为啥要学？

Linux的网络控制模块在内核中，叫做`netfilter`。而`iptables`是位于用户空间的一个命令行工具，它作用在`OIS7层网络模型`中的第四层，用来和内核的`netfilter`交互，配置`netfilter`进而实现对网络的控制、流量的转发 。

> 毫不夸张的说，整个linux系统的网络安全就是基于netfilter构建起来的。

如果你想搞懂docker或者是k8s的网络调度模型，或者是去你自己的机器上查看一下他们自动生成的转发规则，那么肯定要需要对iptables有一定的认知，不然学了半天docker或者是k8s真的是只会停留在使用的这个层面上。

有人可能会说，哎？现在k8s不是已经不把docker看作是亲儿子了吗？然后流量的调度转发规则也更倾向于用LVS了，巴拉巴拉一大堆。嗯，有道理......   那，你敢不学iptables吗？Hhh.....

总之，当你感觉自己以后碰到的技术栈需要你提前了解一些这方面的知识点，可以花几分钟粗略的看一下。

这对你肯定是大有裨益的！

全文较长、建议收藏

# 二、iptables、防火墙之间有啥关系？

Iptables is an extremely flexible firewall utility built for Linux operating systems.

Whether you’re a novice Linux geek or a system administrator, there’s probably some way that iptables can be a great use to you. Read on as we show you how to configure the most versatile Linux firewall.

简单的说就是：iptables 是一个简单、灵活、实用的命令行工具，可以用来配置、控制 linux 防火墙。

# 三、iptables安装

安装、启动、查看、开启启动

```c
 ~]# yum install -y iptables-services ~]# yum start|restart|reload|stop|status iptables
```

# 四、iptables的五表五链及流量走向

iptables中总共有4张表还有5条链，我们可以在链上加不同的规则。

五张表：filter表、nat表、mangle表、raw表、security表

五条链：prerouting、input、output、forward、postrouting

你可以通过`iptables -t ${表名} -nL`查看表上的链
![[Pasted image 20240918114016.png]]


整理一下就得到了如下脑图：
!\[\[Pasted image 20240918114045.png\]\]
!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

先不着急使用iptables命令，大家可以先参考下面这张图，看看如下几种流量的走向。

- 来自本机流量经过了iptables的哪些节点，最终又可以流到哪里去？

- 来自互联网其他主机的流量，经过了本机iptables的哪些节点，最终又可以流到哪里去？

大家打起12分的精神，接下来会出现3张图....

**第一张**：是摘自iptables的wiki百科中的图，如下：
!\[\[Pasted image 20240918114036.png\]\]
!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

我知道很多人都看不懂，没关系，大家只要关注图中的蓝色部分：流量大走向如下：

`raw prerouting` -> `conntrack` -> `mangle prerouting` -> `nat prerouting` - >`decision 路由选择` -> 可能是input，也可能是output。

记住：在wiki百科中的流量走向图中，mangle.prerouting和nat.prerouting之间没有任何判断逻辑就好了。路由选择判断发生在nat.prerouting之后。

______________________________________________________________________

\*\*第二张：\*\*摘自github上写的一篇文章：理解 kube-proxy 中 iptables 规则

这张图是一张更为精确的流量走向图，并且他区分好了：`incoming packet`、`locally gennerated packge` 这种来源不同的流量大走向，原图如下：
!\[\[Pasted image 20240918114057.png\]\]
!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

但是我感觉他这个图稍微有点问题：你可以看下图左上角部分，mange-prerouting和 nat-prerouting之间多了一个 localhost source的判断。但是在iptables维基百科中给出的图中，它俩之间并没有这个判断。

如果你仔细看下，这个localhost source的判断和后面的for this host其实是挺重复的。而这个for this host判断正好对应着第一张图中的 bridge decsion判断。

所以，我总是感觉图应该修改成下面这样，它不一定对，但是起码能自圆其说。对整体理解的影响也不大。

而且图改成这个样子，肯定是方便会我们理解这个过程，而且错也错不了哪里去。
!\[\[Pasted image 20240918114109.png\]\]
!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

稍微解析一下上图：

1、`红`、`蓝`、`绿`、`紫`分别代表上一小节中提到的`iptables`的四张表。如果你开启着SELinux，还会多出一个`security`表

2、上图左上角写的：`incoming packet`，表示这是从互联网其他设备中来的流量。它大概的走向是：先经过各个表的`prerouting`阶段，再经由`routing decision`（可以理解成查路由表，做路由选择）决定这些流量是应该交由本机处理，还是该通过其他网口`forword`转发走。

3、再看上图中的左上部分，`incoming packet`在做`routing decision`之前会先经过`nat preroutings`阶段，我们可以在这个阶段做dnat （目标地址改写），简单来说就是：比如这个数据包原来的`dst ip`是百度的，按理说经过`routing decision`之后会进入forward转发阶段，但是在这里你可以把目标地址改写成自己，让数据流入input通路，在本机截获这个数据包。

4、上图右上角写的：`locally generated packet`，表示这是本机自己生成的流量。它会一路经过各个表的output链，然后流到output interface（网卡）上。你注意下，流量在被打包成outgoing packet之前，会有个localhost dest的判断，如果它判断流量不是发往本机的话，流量会经过nat表的postrouting阶段。一般会在这里做`DNAT`源地址改写。

> 1、至于什么是DNAT、SNAT后文都会讲
>
> 2、大家看上图以及解析的时候，还是应该有一个存疑态度的哈。
>
> 3、我的理解不一定就对，只不过是在现阶段勉强还能自圆其说。

小结：思考这样一个问题

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

所以经过对上图的简单分析，如果咱想自定义对流量进行控制，**那该怎么办？**

这并不复杂。但是在这想该怎么办之前，我们得先搞清楚，通常情况下我们会对流量做那些控制？无非如下：

1. 丢弃来自xxx的流量

1. 丢弃去往xxx的流量

1. 只接收来自xxx的流量

1. 在刚流量流入时，将目标地址改写成其他地址

1. 在流量即将流出前，将源地址改写成其他地址

1. 将发往A的数据包，转发给B

等等等等，如果你足够敏感，你就能发现，上面这六条干预策略，`filter`、`nat`这两张表已经完全能满足我们的需求了，我们只需要在这两张表的不同链上加自己的规则就行，如下：

1. 丢弃来自xxx的流量（`filter表INPUT链`）

1. 丢弃去往xxx的流量（`filter表OUTPUT链`）

1. 只接收来自xxx的流量（`filter表INPUT链`）

1. 在刚流量流入时，将目标地址改写成其他地址（`nat表prerouting链`）

1. 在流量即将流出前，将源地址改写成其他地址（`nat表postrouting链`）

1. 将发往A的数据包，转发给B（`filter表forward链`）

______________________________________________________________________

数据包在iptables中的走向还可以简化成下面这张图
!\[\[Pasted image 20240918114129.png\]\]
!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

参考：https://zjj2wry.github.io/network/iptables/

参考：《Kubernetes网络权威指南》

### 五、iptables commands

```c
iptables -t ${表名}  ${Commands} ${链名}  ${链中的规则号} ${匹配条件} ${目标动作}
```

**表名**：4张表，`filter`、`nat`、`mangle`、`raw`

**Commands**：尾部追加`-A`、检查`-C`、删除`-D`、头部插入`-I`、替换`-R`、查看全部`-L`、清空`-F`、新建chain`-N`、默认规则`-P`(默认为ACCEPT)

**链名**：5条链，`PREROUTING`、`INPUT`、`FORWOARD`、`OUTPUT`、`POSTROUTING`

**匹配条件**：`-p`协议、`-4` 、`-6`、`-s 源地址`、`-d 目标地址`、`-i 网络接口名`

**目标动作**：拒绝访问`-j REJECT`、允许通过`-j ACCEPT`、丢弃`-j DROP`、记录日志 `-j LOG`、源地址转换`-j snat`、目标地址转换`-j dnat`、还有`RETURN`、`QUEUE`

可以通过像如下查看使用帮助文档。主要可以分如下三个部分

```c
 ~]# iptables --help Usage:    # 使用案例 Commands: # 命令选项 Options:  # 其他可选项
```

参考：https://wiki.centos.org/HowTos/Network/IPTables

> 这里面有一些针对linux操作系统iptables简单的教学

### 六、filter表

#### 6.1、usage尝鲜 filter表及规则

```c
# 清空防火墙 ~]
# iptables -F 
# 查看 (policy ACCEPT) 表示默认规则是接收
~]# iptables -t filter -LChain INPUT (policy ACCEPT)target     prot opt source               destinationChain FORWARD (policy ACCEPT)target     prot opt source               destinationChain OUTPUT (policy ACCEPT)target     prot opt source               destination# 添加规则，丢弃|接受｜拒绝｜所有进来的数据包# 注意执行之后shell会断开链接，只能重启机器~]# iptables -t filter -A INPUT -j DROP|ACCEPT|REJECT|LOG# 再查看（如果还想操作某个Chain中的具体某条规则，可以加--line-numbers参数）~]# iptables -t filter -L --line-numbersChain INPUT (policy ACCEPT)num  target     prot opt source               destination1    ACCEPT     all  --  0.0.0.0/0            0.0.0.0/0Chain FORWARD (policy ACCEPT)num  target     prot opt source               destinationChain OUTPUT (policy ACCEPT)num  target     prot opt source               destination# 修改默认规则[root@ecs-kc1-large-2-linux-20201228182212 ~]# iptables -t filter -P INPUT DROP[root@ecs-kc1-large-2-linux-20201228182212 ~]# iptables -t filter -LChain INPUT (policy DROP)target     prot opt source               destinationACCEPT     all  --  anywhere             anywhereREJECT     all  --  anywhere             anywhere             reject-with icmp-port-unreachableChain FORWARD (policy ACCEPT)target     prot opt source               destinationChain OUTPUT (policy ACCEPT)target     prot opt source               destination
```

#### 6.2、案例：filter的流量过滤

比如我想将发送百度的数据包丢弃，就可以在OUTPUT Chain上添加如下的规则

```c
~]# iptables -t filter -A OUTPUT -p tcp -d 220.181.38.251 -j DROP
```

测试一下，结果夯住
!\[\[Pasted image 20240918114244.png\]\]
!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

查看规则，并显示序列号`iptables -L --line-number`
!\[\[Pasted image 20240918114252.png\]\]
!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

删除过滤规则，语法:`iptables -t 表名 -D 链名 规则序号`

```c
~]# iptables -t filter -D OUTPUT 1
```

在指定表的指定链中的指定的位置插入规则

```c
# -I 默认是在头部插入，也可以指定插入的位置
[root@ecs-kc1-large-2-linux-20201228182212 ~]# iptables -t filter -I INPUT 2 -j REJECT[root@ecs-kc1-large-2-linux-20201228182212 ~]# iptables -L --line-numbersChain INPUT (policy ACCEPT)num  target     prot opt source               destination1    ACCEPT     all  --  anywhere             anywhere2    REJECT     all  --  anywhere             anywhere             reject-with icmp-port-unreachableChain FORWARD (policy ACCEPT)num  target     prot opt source               destinationChain OUTPUT (policy ACCEPT)num  target     prot opt source               destination
```

### 七、iptables的匹配规则

**什么是匹配规则？**

这个东西并不难理解，他其实就是一种描述。比如说：我想丢弃来自A的流量，体现在iptables语法上就是：`iptables -t filter -A INPUT -s ${A的地址} -j DROP` ，这段话中的`来自A`其实就是匹配规则。

**常见的规则如下：**

源地址：`-s 192.168.1.0/24`

目标地址：`-d 192.168.1.11`

协议：`-p tcp|udp|icmp`

从哪个网卡进来：`-i eth0|lo`

从哪个网卡出去：`-o eth0|lo`

目标端口（必须制定协议）：`-p tcp|udp --dport 8080`

源端口（必须制定协议）：`-p tcp|udp --sport 8080`

### 八、补充两个小实验，加深流量对走向的理解

#### 8.1、实验一：加深对匹配规则的了解

通过这个实验搞清楚：当数据包命中了该链上的某一个规则后，还会继续往下匹配该链上的其他规则吗？
!\[\[Pasted image 20240918114314.png\]\]
!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

我们在`10.4.4.8`的fitler表、INPUT链上添加如下规则

```c
iptables -t filter -A INPUT -s 10.4.4.11 -j DROP
iptables -t filter -A INPUT -s 10.4.4.12 -j ACCEPT
iptables -t filter -A INPUT -s 10.4.4.11 -j ACCEPT
```

问：ip为`10.4.4.12`的机器能ping通`10.4.4.8`吗？

答：肯定能

问：ip为`10.4.4.11`的机器能ping通`10.4.4.8`吗？

答：不能

流量的走向如下图，它会挨个经过各表的input链，其中当然也包含我们加了DROP规则的filter表的INPUT链。

而你仔细看上面的规则，第一条是DROP、第三条是ACCEPT。再结合最终实验的结果是ping不通，\*\*所以不难猜出流量确实会依次经过链上的规则，但是当有命中的规则后，会去执行该规则`-j`参数指定的动作。不再往下匹配本链上的其他规则。
!\[\[Pasted image 20240918114336.png\]\]
!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

#### 8.2、实验二：特殊的`-j LOG`

实验二思路如下图：
!\[\[Pasted image 20240918114348.png\]\]
!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

我们在`10.4.4.8`上添加如下规则

```c
# 清空现有的规则iptables -t filter -F
# 修改默认规则，默认DROP所有流量
iptables -t filter -P INPUT DROP
# 再添加如下两条规则
iptables -t filter -A INPUT -s 10.4.4.11 -j LOGiptables -t filter -A INPUT -s 10.4.4.11  -j ACCEPT
```

问：从`10.4.4.12`ping`10.4.4.8`能通吗？

答：不能，因为`10.4.4.8`的filter表的INPUT链的默认规则是`DROP`，意思是如果没有命令filter表INPUT链中的其他规则，那么就会DROP这个数据包。所以很显然，`10.4.4.12`的数据包会被drop

问：从`10.4.4.11`ping`10.4.4.8`能通吗？

答：可以。

大概的逻辑我用下面的伪代码通俗表达出来

```c
// 遍历filter.input链上的所有规则for _,pattern := range filter.INPUT.patterns{    if pattern.动作 == `-j LOG`  {    // 将日志记录进 /var/log/message    continue  }    if pattern {    // 执行该规则定义的动作    break;  }  }
```

#### 8.3、实验总结

iptables中每条链下面的规则处理顺序是从上到下逐条遍历的，除非碰到了`DROP`、`REJECT`、`RETURN`。

还有就是如果定义的行为是JUMP，那就会相应的jump到指定链上的指定规则上，如下：
!\[\[Pasted image 20240918114413.png\]\]
!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

### 九、`iptables`中的模块

- 多端口

Case: 可以在命令中像下面这样指定多个连续的端口

```
iptables -t ${表名} ${commands} ${chain} ${规则号} --dport 20:30 -j ${动作}其中的20:30表示20和30之间的所有端口
```

想指定多个不连续的端口可以使用iptables的`multiport`

```c
# 查看帮助文档~]# iptables -m multiport --help
...
multiport match options:[!] --source-ports port[,port:port,port...] --sports ...    match source port(s)[!] --destination-ports port[,port:port,port...] --dports ...    match destination port(s)[!] --ports port[,port:port,port]    match both source and destination port(s)#命令例子iptables -t ${表名} ${commands} ${chain} ${规则号}       ${-p 协议} -m multiport --dports 20,30 -j ${动作}#相当于如下两行命令iptables -t ${表名} ${commands} ${chain}  ${规则号} -p ${协议} --dprot 20 -j ${动作}iptables -t ${表名} ${commands} ${chain}  ${规则号} -p ${协议} --dprot 30 -j ${动作}
```

- ip范围

查看帮助文档

```c
~]# iptables -m iprange --helpiprange match options:[!] --src-range ip[-ip]    Match source IP in the specified range[!] --dst-range ip[-ip]    Match destination IP in the specified range
```

案例：

```c
# 拒绝指定范围内的ip访问自己80端口iptables -t filter -A INPUT      -m -iprange --src-range 10.10.10.2-10.10.10.5     -p tcp --dport 80         -j DROP
```

- 连接状态

查看帮助文档：

```c
~]# iptables -m state --helpstate match options: [!] --state [INVALID|ESTABLISHED|NEW|RELATED|UNTRACKED][,...]
```

INVALID：无效连接

NEW：首次访问和服务端建立的连接，之后和服务端的交互时连接的state不再是NEW

ESTABLISHED：连接完成，和服务端建立连接之后的所有交互，包括访问服务端、服务器的响应。

RELATED：相关联的连接，如ftp使用20、21建立连接，再使用其他端口交互数据时的这个连接就是和第一次建立连接时的相关的连接。state也就是`RELATED`。

> 想让防火墙识别出连接的状态为RELATED，需要让iptable加载插件`vi /etc/sysconfig/iptables-config`，然后修改：`IPTABLES_MODULES="nf_conntrack_ftp"`，再重启防火墙即可。

案例：

```
# 放行所有状态为ESTABLISHD的数据包iptables -f filter -A OUTPT     -m state --state ESTABLISHED     -j ACCEPT
```

### 十、nat表

查看NAT表，可以找到它有4条链，如下

```
 ~]# iptables -t nat -LChain PREROUTING (policy ACCEPT)target     prot opt source               destinationChain INPUT (policy ACCEPT)target     prot opt source               destinationChain OUTPUT (policy ACCEPT)target     prot opt source               destinationChain POSTROUTING (policy ACCEPT)target     prot opt source               destination
```

通常我们会通过给nat表中的`PREROUTING`、`POSTROUTING`这两条链添加规则来实现`SNAT`、`DNAT`如下:

`-j SNAT` ，源地址转换说的时在数据包发送出去前，我们将数据包的src ip修改成期望的值。所以这个规则需要添加在`POSTROUTING`中。

`-j DNAT`，目标地址转换？比如发送者发过来的数据包的`dst ip = A`，那目标地址转换就是允许我们修改`dst ip=B`，也就是将数据包发送给B处理，这个过程对发送者来说是感知不到的，他会坚定的认为自己的数据包会发往A，而且B也感知不到这个过程，它甚至会认为这个数据包从一开始就是发给他的。

`-j MASQUERADE`，地址伪装

#### 10.1、案例：使用`nat表`完成`SNAT`

案例：将`src ip = 10.10.10.10`转换成`src ip = 22.22.22.22`然后将数据包发给期望的目的ip所在的机器

```
iptables -t nat -A POSTROUTING -s 10.10.10.10 -j SNAT --to 22.22.22.1# 如果公网地址不停的变化，可以像下面这样设置：# step1:使用MASQUERADE做地址伪装，意思是我们不用手动指定将src ip转成哪个ip# step2:让本机自己去查有没有去往dst ip的通路，如果有的话，就用这个通路的ip作为src ip# step3:另一边数据包的接收者来说，src ip就是step2中自动找到的通路网口的ip地址iptables -t nat -A POSTROUTING -s 10.10.10.10 -j MASQUERADE
```

总结：源地址转换一般是数据包从私网流向公网时做的转换，将私网地址转换成公网地址，会将这个转换记录记录到地址转换表中，目的是为了数据包从公网回来时，能正确的回到私网中发送请求的机器中。

#### 10.2、案例：通过`nat表`完成`DNAT`

> DNAT全称：`Destnation Network Address Tranlater`目标地址改写。

当我 `telnet 192.168.0.2 55`时，我期望`tables`规则将我的目标地址改写成`20.181.38.148:80`

先尝试`telnet 192.168.0.2 55`，发现没有任何route去该host

```
~]# telnet 192.168.0.2 55Trying 192.168.0.2...telnet: connect to address 192.168.0.2: No route to host
```

好，再使用iptable添加DNAT规则，命令如下：

```
~] iptables -t nat -A OUTPUT -p tcp -d 192.168.0.2 --dport 55 -j DNAT --to-destination 220.181.38.148:80  # 参数解析 # -A 表示append，追加iptable规则 # OUTPUT 是内核的回调钩子，流量被转发出去前会经过这个阶段，DNAT的规则就加在这个阶段。 # -p protocol 协议 # -d destnation 目标地址 # --dport 目标口 # -j jump 跳转（跳转到目标规则） # --to-destination 目标地址  # 命令含义： 当OUTPUT Chain上的数据包使用的协议是tcp、目标地址是192.168.0.2、目标端口是55时， 我跳转到DNAT模式，将目标地址改成：220.181.38.148:80
```

再尝试`telnet 192.168.0.2 55`，会发现链路通了！

```
~]# telnet 192.168.0.2 55Trying 192.168.0.2...Connected to 192.168.0.2.Escape character is '^]'.Connection closed by foreign host.
```

再查看iptable规则，如下：
!\[\[Pasted image 20240918114529.png\]\]
!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

> 可以看到有很多的`Chain`比如上面的OUTPUT就是一条链，它是操作系统的一个hook，当请求被发送出去时会经过这个hook。
>
> 还有就是通过iptable的DNAT能力可以做tizi！你仔细琢磨一下，原理就在上面的图中哦～

补充典型应用场景：公司有自己建设的机房，机房中机器的ip都在：`10.10.10.0/24`网段，机房对外提供服务就需要通过网络设备暴露公网一个ip，比如是：`22.22.22.1:80`

当网络设备收到`dst ip = 22.22.22.1:80`的数据包时，就会做`DNAT`转换，将`dst ip`转换成内网中的某台服务器的ip，比如就是:`dtp ip = 10.10.10.10`，然后将数据包转发给这台机器让它处理数据包。

### 十一、相关配置文件

iptables的配置文件在`/etc/sysconfig`目录如下：

```
-rw-------  1 root root  635 10月  2 2020 ip6tables-rw-------  1 root root 2134 10月  2 2020 ip6tables-config-rw-------  1 root root  550 10月  2 2020 iptables-rw-------  1 root root 2116 10月  2 2020 iptables-config
```

`iptables`启动时会加载这个配置文件中定义好的各表、链中的规则
!\[\[Pasted image 20240918114538.png\]\]
!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

一般我们会先使用`iptables`修改各种规则，如果想让iptables启动、重启时，继续使用你刚才修改的配置，我们就会考虑使用`iptables-save`命令将规则导出为配置文件。

```
# 导出某张表的iptables规则[root@bairimeng ~]# iptables-save -t filter > 1.bak# 导出全力量iptables规则[root@bairimeng ~]# iptables-save > 2.bak
```

再将文件中的内容拷贝进`/etc/sysconfig/iptables`中即可

### 十二、串联 `iptables`、`路由表`

通过下面的逻辑，串联`iptables`和`路由表`这两个知识点

我们可以试着结合OIS7层网络模型来看这件事
!\[\[Pasted image 20240918114546.png\]\]
!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

我们着重看下主机B接受收主机A发送给它的数据包都经历了哪些步骤。

数据包在物理层时还是比特流，再往上传输到数据链路层被转换成数据帧，我们只需要按照以太网协议解析数据帧就能解析出数据包中的MAC地址。因此，我们有了如下的判断逻辑：

```c
if 接受包.mac == 本机mac{ // 可以考虑进一步处理
}else{ // 丢弃}
```

经过通过了mac地址的校验，数据包被传输到网络层，对网络层来说，不同服务器之间的数据交互是和IP直接挂钩的，所以按照网络层的协议解析数据包我们能得到接受包的ip信息。再结合iptables四表五链的概念，所以我们可以进一步完善上面的逻辑如下：

```c
// 在2层网络（数据链路层）做如下判断
if 接受包.mac == 本机mac{    // 流经各表的prerouting链    // 在这个阶段你可以做很多事情，比如在nat表的prerouting阶段加dnat规则   if 接受包.ip == 本机ip { // routing decision        // 1.1、按理说会进入各表的input链    // 1.2、但是如果当前设备是路由器，它只有OSI的前3层，所以它只能查地址转换表，将数据包转发走    // 2、local processing     } else {        // 进入 forwoard 阶段    if 接受包.dstIp & route.Genmask == route.Destination {      // 1、通过该网卡将数据包转发出去      // 2、转发走前进入各表的postrouting阶段    }else{      // 1、经由Gateway将数据包转发出去      // 2、转发走前进入各表的postrouting阶段1    }     }}else{ // 丢弃}
```

> 上面这个总结完全是我自圆其说的理解，不一定真的对，但是也错不了哪里去，希望大家带着存疑的态度看。

### 十三、参考资料

1、https://zjj2wry.github.io/network/iptables/

2、维基百科：https://en.wikipedia.org/wiki/Iptables

3、man page https://linux.die.net/man/8/iptables

4、centos iptables wiki：https://wiki.centos.org/HowTos/Network/IPTables

5、ibm：https://www.ibm.com/docs/en/linux-on-systems?topic=tests-firewall-iptables-rules

6、参考：《Kubernetes网络权威指南》

### 欢迎关注啦！

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

______________________________________________________________________

后台回复“加群”，带你进入高手如云交流群

**推荐阅读：**

[深入理解 Cilium 的 eBPF 收发包路径](http://mp.weixin.qq.com/s?__biz=MzI1OTY2MzMxOQ==&mid=2247496345&idx=1&sn=22815aeadccc1c4a3f48a89e5426b3f3&chksm=ea77c621dd004f37ff3a9e93a64e145f55e621c02a917ba0901e8688757cc8030b4afce2ef63&scene=21#wechat_redirect)

[Page Cache和Buffer Cache关系](http://mp.weixin.qq.com/s?__biz=MzI1OTY2MzMxOQ==&mid=2247495951&idx=1&sn=8bc76e05a63b8c9c9f05c3ebe3f99b7a&chksm=ea77c5b7dd004ca18c71a163588ccacd33231a58157957abc17f1eca17e5dcb35147b273bc52&scene=21#wechat_redirect)

[深入理解DPDK程序设计|Linux网络2.0](http://mp.weixin.qq.com/s?__biz=MzI1OTY2MzMxOQ==&mid=2247495791&idx=1&sn=5d9f3bdc29e8ae72043ee63bc16ed280&chksm=ea77c4d7dd004dc1eb0cee7cba6020d33282ead83a5c7f76a82cb483e5243cd082051e355d8a&scene=21#wechat_redirect)

[一文读懂基于Kubernetes打造的边缘计算](http://mp.weixin.qq.com/s?__biz=MzI1OTY2MzMxOQ==&mid=2247495291&idx=1&sn=0aebc6ee54af03829e15ac659db923ae&chksm=ea77dac3dd0053d5cd4216e0dc91285ff37607c792d180b946bc09783d1a2032b0dffbcb03f0&scene=21#wechat_redirect)

[网络方案 Cilium 入门教程](http://mp.weixin.qq.com/s?__biz=MzI1OTY2MzMxOQ==&mid=2247495170&idx=1&sn=54d6c659853f296fd6e6e20d44b06d9b&chksm=ea77dabadd0053ac7f72c4e742942f1f59d29000e22f9e31d7146bcf1d7d318b68a0ae0ef91e&scene=21#wechat_redirect)

[聊聊非阻塞I/O编程](http://mp.weixin.qq.com/s?__biz=MzI1OTY2MzMxOQ==&mid=2247495131&idx=1&sn=70b250d4e531e9f3cae675032c586c15&chksm=ea77d963dd0050755c29483567022c9454c38f8d41b9f27f93647af45ae2192b84dcabc615ab&scene=21#wechat_redirect)

[Linux内核调度器源码分析](http://mp.weixin.qq.com/s?__biz=MzI1OTY2MzMxOQ==&mid=2247494772&idx=1&sn=89b7bbab0caf86de6f5d6a2cac8dd8fc&chksm=ea77d8ccdd0051da8534d38e23d7a1a422b974d4e9bd773187c2dea377a32b8e623f4453b0a0&scene=21#wechat_redirect)

[Docker  容器技术使用指南](http://mp.weixin.qq.com/s?__biz=MzI1OTY2MzMxOQ==&mid=2247494756&idx=1&sn=f7384fc8979e696d587596911dc1f06b&chksm=ea77d8dcdd0051ca7dacde28306c535508b8d97f2b21ee9a8a84e2a114325e4274e32eccc924&scene=21#wechat_redirect)

[Linux下的一些资源限制](http://mp.weixin.qq.com/s?__biz=MzI1OTY2MzMxOQ==&mid=2247494730&idx=1&sn=3231f464497ba1cc0d27e90064f45556&chksm=ea77d8f2dd0051e4033a8d292dd49d986e77f4b059df92cab4015a5925fec1b2cddf3a199ac0&scene=21#wechat_redirect)

[Linux 系统安全强化指南](http://mp.weixin.qq.com/s?__biz=MzI1OTY2MzMxOQ==&mid=2247494657&idx=1&sn=e168f542db246005556ce4f0c30d6a3c&chksm=ea77d8b9dd0051af485a7bdb907696ce2a0c5da251cfdc2b04f4f47a332fa29262477e5b8dcd&scene=21#wechat_redirect)

[云原生/云计算发展白皮书（附下载）](http://mp.weixin.qq.com/s?__biz=MzI1OTY2MzMxOQ==&mid=2247494647&idx=1&sn=136f21a903b0771c1548802f4737e5f8&chksm=ea77df4fdd00565996a468dac0afa936589a4cef07b71146b7d9ae563d11882859cc4c24c347&scene=21#wechat_redirect)

[Linux 常用监控指标总结](http://mp.weixin.qq.com/s?__biz=MzI1OTY2MzMxOQ==&mid=2247493544&idx=1&sn=68d86833cb3934abca18c95da8b1bae6&chksm=ea77d310dd005a06ad7de14d4d9f29d88e1cadebda7ccd975b3a265e5806608ace14ba12c8b4&scene=21#wechat_redirect)

[Kubernetes 集群网络从懵圈到熟悉](http://mp.weixin.qq.com/s?__biz=MzI1OTY2MzMxOQ==&mid=2247493426&idx=1&sn=e3492cf43c4268c5948d170f4a5d2441&chksm=ea77d38add005a9cdbb5775f2bfd4a2b953e950fcb65c25e91eaea45c68bf5684e3ebc8289e0&scene=21#wechat_redirect)

[使用 GDB+Qemu 调试 Linux 内核](http://mp.weixin.qq.com/s?__biz=MzI1OTY2MzMxOQ==&mid=2247493336&idx=1&sn=268fae00f4f88fe27b24796644186e9e&chksm=ea77d260dd005b76c10f75dafc38428b8357150f3fb63bc49a080fb39130d6590ddea61a98b5&scene=21#wechat_redirect)

[防火墙双机热备](http://mp.weixin.qq.com/s?__biz=MzI1OTY2MzMxOQ==&mid=2247493173&idx=1&sn=53975601d927d4a89fe90d741121605b&chksm=ea77d28ddd005b9bdd83dac0f86beab658da494c4078af37d56262933c866fcb0b752afcc4b9&scene=21#wechat_redirect)

[常见的几种网络故障案例分析与解决](http://mp.weixin.qq.com/s?__biz=MzI1OTY2MzMxOQ==&mid=2247493157&idx=1&sn=de0c263f74cb3617629e84062b6e9f45&chksm=ea77d29ddd005b8b6d2264399360cfbbec8739d8f60d3fe6980bc9f79c88cc4656072729ec19&scene=21#wechat_redirect)

[Kubernetes容器之间的通信浅谈](http://mp.weixin.qq.com/s?__biz=MzI1OTY2MzMxOQ==&mid=2247493145&idx=1&sn=c69bd59a40281c2d7e669a639e1a50cd&chksm=ea77d2a1dd005bb78bf499ea58d3b6b138647fc995c71dcfc5acaee00cd80209f43db878fdcd&scene=21#wechat_redirect)

[kube-proxy 如何与 iptables 配合使用](http://mp.weixin.qq.com/s?__biz=MzI1OTY2MzMxOQ==&mid=2247492982&idx=1&sn=2b842536b8cdff23e44e86117e3d940f&chksm=ea77d1cedd0058d82f31248808a4830cbe01077c018a952e3a9034c96badf9140387b6f011d6&scene=21#wechat_redirect)

[完美排查入侵](http://mp.weixin.qq.com/s?__biz=MzI1OTY2MzMxOQ==&mid=2247492931&idx=1&sn=523a985a5200430a7d4c71333efeb1d4&chksm=ea77d1fbdd0058ed83726455c2f16c9a9284530da4ea612a45d1ca1af96cb4e421290171030a&scene=21#wechat_redirect)

[QUIC也不是万能的](http://mp.weixin.qq.com/s?__biz=MzI1OTY2MzMxOQ==&mid=2247491959&idx=1&sn=61058136134e7da6a1ad1b9067eebb95&chksm=ea77d5cfdd005cd9261e3dc0f9689291895f0c9764aa1aa740608c0916405a5b94302f659025&scene=21#wechat_redirect)

[超详干货！Linux环境变量配置全攻略](http://mp.weixin.qq.com/s?__biz=MzI1OTY2MzMxOQ==&mid=2247491841&idx=1&sn=096892e3d99e85d44d196b3186c64a6b&chksm=ea77d5b9dd005caf6df2c3bc7dacfaa182a1e5c0d5e473fc847bfbb33f033eaba67e9b55e413&scene=21#wechat_redirect)

[为什么要选择智能网卡？](http://mp.weixin.qq.com/s?__biz=MzI1OTY2MzMxOQ==&mid=2247491828&idx=1&sn=d81f41f6e09fac78ddddc287feabe502&chksm=ea77d44cdd005d5a48dc97e13f644ea24d6e9533625ce8f204d4986b6ba07f328c5574820703&scene=21#wechat_redirect)

[网络排错大讲解~](http://mp.weixin.qq.com/s?__biz=MzI1OTY2MzMxOQ==&mid=2247488574&idx=1&sn=68df1982e9f23ce42457d00ce529b012&chksm=ea742086dd03a9902adb16d2b7648149209fed6c9811c6dd05be5101b42d462cb48e269b6e9d&scene=21#wechat_redirect)

[OVS 和 OVS-DPDK 对比](http://mp.weixin.qq.com/s?__biz=MzI1OTY2MzMxOQ==&mid=2247488294&idx=2&sn=303d9baefa768f887ba36213df793025&chksm=ea74279edd03ae88bc91300a1666066881dd763879b714941120fb9e41e787c557c2930ff5db&scene=21#wechat_redirect)

[微软出品的最新K8S学习指南3.0](http://mp.weixin.qq.com/s?__biz=MzI1OTY2MzMxOQ==&mid=2247488271&idx=1&sn=a52af9aba2ea8bbc5fd4440f85ece458&chksm=ea7427b7dd03aea1dbf106326168333c651b403dd44abf4e652d5a1912377d64becabac5051a&scene=21#wechat_redirect)下载

▼

_\*\*_****喜欢，就给我一个****“在看”\*\*\*\*_\*\*_

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**10T 技术资源大放送！包括但不限于：云计算、虚拟化、微服务、大数据、网络、**Linux、**Docker、Kubernetes、Python、Go、C/C++、Shell、PPT 等。在公众号内回复「****1024****」**，即可免费获取！！****

Reads 3119

​

Comment

**留言 8**

- Janrry丶龙龙

  2021年9月15日

  Like1

  第五章前面那个精简的iptables图，出去的流量是postrouteing链，作者你看看是不是写错了🤤 ，不过真的很实用，你的文章，每天上班地铁必看内容！

- cavanxf

  2021年9月11日

  Like

  再一个配置有几百条规则的系统上，如何快速定位一个tcp请求被哪个规则给拦截掉了呢？

- Z

  2021年9月10日

  Like

  我的OpenWRT路由器会丢弃目的MAC是局域网其他设备的DNS请求，查遍了iptables都没找到是为啥

- Ryan Bao

  2021年9月9日

  Like

  我见过最实用的介绍了，大赞👍🏻

- 小样儿

  2021年9月9日

  Like

  没看懂，说明文章写的还不够详细

- 果冻

  2021年9月9日

  Like

  文章开头有错误 OSI7层

- Jack Cai

  2021年9月9日

  Like

  到底几表几链？

  Linux云计算网络

  Author2021年9月9日

  Like

  文章都看完了还不知道

已无更多数据

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/1TDxR6xkRSFD7ibPKdP9JOqxFxp199B6LLCw80xbTAicxIQPgpsq5ib8kvx7eKfq6erLb1CAT4ZKa3RSnl9IWfELw/300?wx_fmt=png&wxfrom=18)

Linux云计算网络

13211

8

Comment

**留言 8**

- Janrry丶龙龙

  2021年9月15日

  Like1

  第五章前面那个精简的iptables图，出去的流量是postrouteing链，作者你看看是不是写错了🤤 ，不过真的很实用，你的文章，每天上班地铁必看内容！

- cavanxf

  2021年9月11日

  Like

  再一个配置有几百条规则的系统上，如何快速定位一个tcp请求被哪个规则给拦截掉了呢？

- Z

  2021年9月10日

  Like

  我的OpenWRT路由器会丢弃目的MAC是局域网其他设备的DNS请求，查遍了iptables都没找到是为啥

- Ryan Bao

  2021年9月9日

  Like

  我见过最实用的介绍了，大赞👍🏻

- 小样儿

  2021年9月9日

  Like

  没看懂，说明文章写的还不够详细

- 果冻

  2021年9月9日

  Like

  文章开头有错误 OSI7层

- Jack Cai

  2021年9月9日

  Like

  到底几表几链？

  Linux云计算网络

  Author2021年9月9日

  Like

  文章都看完了还不知道

已无更多数据
