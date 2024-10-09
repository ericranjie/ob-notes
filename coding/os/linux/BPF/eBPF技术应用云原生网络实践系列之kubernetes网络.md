Linux云计算网络

_2022年03月19日 19:39_

![](https://mmbiz.qpic.cn/mmbiz_png/aibHzUdEOibUllc3N6BDMLDevQhcx8OtAkMm5mmOF2GvIZhibcMMZIzWpExwV0blhJtrBeqDjb3T4fPhhjUHNpnZw/640?wx_fmt=png&wxfrom=13&tp=wxpic)

本文转自“云巅论剑”公众号

kubernetes网络性能问题

kubernetes网络可分为Pod网络、服务网络、网络策略三个部分。Pod网络依据kubernetes网络模型连接集群中的Pod、Node到一个扁平的3层网络中，也是服务网络、网络策略实现的底座。服务网络在Pod网络的路径上添加负载均衡的能力支持Pod到service的访问。网络策略则通过状态防火墙功提供pod间的访问控制。

![图片](https://mmbiz.qpic.cn/mmbiz_png/bDZYPQUwvDnnH9r7E1MmiascQuyRiamDH37Amla6O8U1q8sfka8V5Gj8OWYLoaaX9mmXuly1DtJhpM4Hp7Wj83nQ/640?wx_fmt=png&wxfrom=13&tp=wxpic "image.png")

当前服务网络和网络策略多基于kernel协议栈的L3包过滤技术Netfilter框架实现，包括conntrack, iptables, nat, ipvs等。Netfilter作为2001年加入kernel的的包过滤技术，提供了灵活的框架和丰富的功能，也是Linux中应用最为广泛的包过滤技术。不过其在性能、扩展能力等方面也一直存在一些问题，例如iptables规则由于是线性结构存在更新慢、匹配慢问题，conntrack在高并发场景中性能下降快、连接表被打满的问题等等。Netfilter的这些问题也导致了服务网络、网络策略难以满足大规模的kubernetes集群和一些高性能场景的需求。

为解决服务网络、网络策略基于netfilter实现的性能问题，也有一些通过云上网络产品来实现的方法。例如由L4负载均衡实现服务网络、安全组实现网络策略。云上网络产品提供了丰富的网络功能和确定的SLA的保障，但因当前云网产品主要聚焦在IaaS层，其创建速度、配额等限制存在难以满足云原生场景的弹性、快速等需求。

Pod网络的实现依据是否经过kernel L3协议栈可分为两大类：经过L3协议栈的路由、Bridge等，bypass L3协议栈的IPVlan、 MacVlan,、直通等。经过L3协议栈的方案由于datapath中保留了完整的L3的处理，可兼容当前的服务网络、网络策略的实现，但因L3的逻辑较重这类方案性能较低。bypass L3协议栈的方案，由于datapath逻辑简单性能扩展性都较好，但存在较难实现服务网络、网络策略的问题。

综上，kubernetes网络在性能上的问题总结为如下几类：

|   |   |
|---|---|
||问题|
|服务网络、网络策略|- 基于netfilter框架的方案，性能、扩展性较低<br>    <br>- 云产品方案，无法满足云原生场景的要求|
|Pod网络|- 经过L3协议栈的方案性能较低<br>    <br>- bypass L3协议栈的方案难以实现服务网络、网络策略|

考虑kubenetes网络当前的现状，我们需要重新实现一种不依赖于Netfilter的轻量的服务网络、网络策略实现。一方面可以应用于IPVlan、MacVlan、直通等高性能Pod网络中、补齐其对服务网络、网络策略的支持，另一方面新的实现需要有更好的性能、扩展能力。

# 基于eBPF的方案

## eBPF: 可编程的转发面

eBPF在kernel实现了一个基于寄存器的虚拟机，并定于了自己的64bit指令集。通过系统调用，我们可以加载一段编写好的eBPF程序到kernel，并在相应的事件发生触发运行。同时kernel也提供的相关验证机制，确保加载的eBPF程序其可以被安全的执行，避免对kernel产生破坏，导致painc, 安全隐患等风险。

kernel当前在许多模块中增加了eBPF的支持，并定义了多种的eBPF程序类型。每种类型确定了其可以加载到kernel中的位置、可以调用的kernel helper函数、运行eBPF程序时传递的对象、以及对kernel对象的读写权限等。目前网络是eBPF应用较多也是发展较快的的子系统之一。从协议栈底层的网卡到上层的socket，均存在多种类型的eBPF的支持。

同时，针对当前netfilter的一些性能，社区也在基于eBPF实现一个新的名为bpfilter的包过滤技术https://lwn.net/Articles/755919/。

从网络开发者的视角，eBPF相当于在kernel中提供了一种可编程的数据面，同时由kernel去保证程序的安全性。我们希望基于此实现一种高性能的服务网络和网络策略，同时应用于IPVlan, MacVlan, 直通等bypass L3的Pod网络。从而在kubernetes中提供一种综合性能较好的网络解决方案。

## tc-ebpf：一种新的服务网络&网络策略实现

tc-ebpf位于Linux协议栈的L2层，可以hook网络设备的入、出双向流量到eBPF实现的datapath，基于tc-ebpf实现网络网络&网络策略优势有：

- hook在tc层，不依赖kernel的L3协议栈能力，可较容易的支持ipvlan、macvlan、直通等多种高性能网络模式;

- 可以读写skb_buff，kernel提供的helper函数功能丰富，较容易实现复杂的datapath。

- 规则的匹配、更新基于Hash，比线性操作的iptables规则性能高、扩展性好

- datapath可编程。可在不重启、升级kernel的前提下，对datpath进行更新、优化

通过tc-ebpf构建服务网络&网络策略一般需要实现如下几个模块：

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E "image.png")

Conntrack是服务网络、网络策略的基础模块。egress, ingress流量均会先经过conntrack的处理，并生成容器流量的连接状态。

LoadBalance+ReverseNat为服务网络的模块。LoadBalance为egress流量选择可用的endpoint，并将负载均衡信息同步到连接追踪信息中；ReverseNat为ingress流量查找LoadBalance信息并执行发向nat，修改源IP为cluster ip。

Policy模块基于Conntrack的连接状态，对于新连接应用用户定义的Policy规则。

我们也调研了当前相关的社区一些实现，Cilium是基于eBPF技术构建kubernetes网络能力其中的佼佼者，使用eBPF实现了Pod网络、服务网络、网络策略等能力，但也发现cilium目前支持的网络模式没有很好的应用于公有云的场景。

- vxlan封包模式：在封包解包过程中造成额外的性能损失，并且封包之后的UDP包不能利用tso等网卡offload性能优化

- linux路由模式：在一些虚拟化环境中不支持2层的路由协议，比如在阿里云上没办法通过路由协议打通不同机器的Pod IP段？

- cni chain: cilium当前支持的的cni chain的方式，可以对接veth的网络插件，但性能上的优化并不明显。

## Terway With eBPF: 我们的方案

Terway能够将容器网络和虚拟机网络在同一个网络平面，在不同主机之间容器网络通信时不会有封包等损失，不依赖于分布式路由也能让集群规模不受限于路由条目限制。

我们和Terway合作，通过CNI chain的方式连接Terway和Cilium，进一步优化了Kubernetes的网络性能。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E "image.png")

Terway作为CNI chain的第一个CNI，在阿里云VPC之上创建Pod网络连接Pod到VPC，提供高性能的容器网络通信。而Cilium作为第二个CNI，实现高性能的服务网络和网络策略。

### IPVlan Based CNI Chain

综合单Node上Pod密度、性能等因素，Pod网络采用VPC的eni多ip特性，基于IPVlan L2的方式创建。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E "image.png")

同时在Cilium中实现一种IPVlan模式的CNI chain mode，在terway创建创建的Pod网络基础上接入tc-eBPF的datapath，实现服务网络和网络策略。!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E "image.png")

# 性能评估

网络性能评估主要与Terway当前三种网络模式对比：

- 策略路由模式

- IPVlan

- ENI

本方案实现的Terway With eBPF(下列图中简称eBPF)模式性能测试结论：

- Pod网络性能

- ebpf模式接近ipvlan模式，相比下降2%，相比策略路由提升11%，

- 扩展性能

- 当Pod的数量从3增加156时，IPVlan和eBPF下降约13%,  策略路由下降约23%

- 服务网络性能

- 相比kube-proxy(iptables模式)短连接提升约32%，长连接提升19%

- 相比kube-proxy(ipvs模式)提升短连接约60%，长连接提升约50%

- 扩展性能

- kuberenetes服务从1增加到5000时，eBPF和kube-proxy(ipvs模式)性能都表现出较好的扩展能力持平，5000个服务性能略微下降3%；kube-proxy(iptables模式)短连接下降约60%， 长连接下降约6%左右

- 网络策略性能

- pod和policy为1时，eBPF模式比策略路由在短连接提升约32%，长连接提升约16%

- pod和policy为增加到120时，短连接eBPF相比策略路由性能提升19%；长连接提升24%

## Pod网络

tcp性能测试结果:

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E "image.png")

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E "image.png")

udp性能测试结果:

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E "image.png")!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E "image.png")

## 

## 服务网络

service网络测试不同网络模型下的service network的转发性能和扩展能力。测试对比了pod网络，service为1，1000， 5000四种类型。测试工具采用ab、nginx，nginx配置为8核心。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E "image.png")!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E "image.png")

## 网络策略

Terway的网络模式中仅有策略路由。本测试对比了两种模式在被测pod为1配置1个policy以及被测pod为120配置120个policy下的性能数据。测试工具为ab-nginx, nginx配置为8核心。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E "image.png")!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E "image.png")

# 总结 & 后续

分析当前Kubernetes中网络(Pod网络、服务网络及网络策略)中的性能、扩展能力等问题。我们调研了kenrel eBPF技术及社区开源的eBPF容器方案Cilium，最后在阿里云上结合Terway，基于ENI多IP的网络特性实现了一种高性能容器网络方案。同时我们在cilium实现的IPVlan L2的CNI chain方案也已开源到Cilium社区。

在当前方案中Host Namespace的服务网络和网络策略的实现仍然是由kube-proxy实现。目前主要考虑Host Namespace中的服务网络流量在集群流量中占比较小，且流量类型较为复杂(包括cluter ip、node port、extern ip等)，由eBPF实现的优先级不高收益较小。

tc-ebpf由于在Linux的L3之前，暂无还无法支持分片报文的重组。这会导致服务网络&网络策略的实现对分片报文失效，对于此，我们设计了redirect分片到L3协议栈重组重入eBPF的方案。不过考虑到kubernets集群大都部署在同类网络中，分片报文出现的概率相对较低，未在当前版本中实现。另外，cilium当前也在eBPF中增加了服务网络&网络策略对分片报文的支持 (https://github.com/cilium/cilium/pull/10264)，目前该实现还存在无法分片报文乱序到底的问题。

# 展望

eBPF作为新技术为kerenl的网络转发面开发提供了巨大的可能性。我们目前的方案相当于在L2层实现了服务网络、网络策略，虽然在性能上相比基于netfilter的方案存在一定的提升，但实现的conntrack模块在网络上依然是一个不可忽视的开销。另一方面我们也在思考，在kubernetes的pod或node中的datapath上是否真的需要conntrack？是否可以基于socket的状态信息实现无负载开销的conntrack。

基于此我们也在探索在socket层实现服务网络、网络策略的可能性。我们希望可以借助socket层的eBPF，构建一种开销极小conntrack，进而实现一种性能、扩展性更优的服务网络、网络策略方案。

______________________________________________________________________

后台回复“加群”，带你进入高手如云交流群

**推荐阅读：**

[容器网络|深入理解Cilium](http://mp.weixin.qq.com/s?__biz=MzI1OTY2MzMxOQ==&mid=2247498724&idx=1&sn=d33d04212563f4596ff08c0c6f81533d&chksm=ea77cf5cdd00464a4e4a34adfc8916f701fa6db789ec0d7e7aaea898711dec43ecd4c7d2adbe&scene=21#wechat_redirect)

**[Linux下的TCP测试工具](http://mp.weixin.qq.com/s?__biz=MzI1OTY2MzMxOQ==&mid=2247498690&idx=1&sn=c5a176e6c8e9f1c7258abbfb06b53cb7&chksm=ea77cf7add00466c51d525833a9b22101a2da60a640740cb1e243eb88830e9eceaaa59cf5ea7&scene=21#wechat_redirect)**

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

阅读 1007

​

写留言

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/1TDxR6xkRSFD7ibPKdP9JOqxFxp199B6LLCw80xbTAicxIQPgpsq5ib8kvx7eKfq6erLb1CAT4ZKa3RSnl9IWfELw/300?wx_fmt=png&wxfrom=18)

Linux云计算网络

72在看

写留言

写留言

**留言**

暂无留言
