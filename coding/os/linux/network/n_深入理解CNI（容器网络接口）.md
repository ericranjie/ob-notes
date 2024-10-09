# 

Linux云计算网络

_2021年09月12日 10:30_

\*\*

01、 CNI简

\*\*

容器网络的配置是一个复杂的过程，为了应对各式各样的需求，容器网络的解决方案也多种多样，例如有flannel，calico，kube-ovn，weave等。同时，容器平台/运行时也是多样的，例如有Kubernetes，Openshift，rkt等。如果每种容器平台都要跟每种网络解决方案一一对接适配，这将是一项巨大且重复的工程。当然，聪明的程序员们肯定不会允许这样的事情发生。想要解决这个问题，我们需要一个抽象的接口层，将容器网络配置方案与容器平台方案解耦。

CNI（Container Network Interface）就是这样的一个接口层，它定义了一套接口标准，提供了规范文档以及一些标准实现。采用CNI规范来设置容器网络的容器平台不需要关注网络的设置的细节，只需要按CNI规范来调用CNI接口即可实现网络的设置。

CNI最初是由CoreOS为rkt容器引擎创建的，随着不断发展，已经成为事实标准。目前绝大部分的容器平台都采用CNI标准（rkt，Kubernetes ，OpenShift等）。本篇内容基于CNI最新的发布版本v0.4.0。

值得注意的是，Docker并没有采用CNI标准，而是在CNI创建之初同步开发了CNM（Container Networking Model）标准。但由于技术和非技术原因，CNM模型并没有得到广泛的应用。

**2、CNI是怎么工作的**

CNI的接口并不是指HTTP，gRPC接口，CNI接口是指对可执行程序的调用（exec)。这些可执行程序称之为CNI插件，以K8S为例，K8S节点默认的CNI插件路径为 /opt/cni/bin ，在K8S节点上查看该目录，可以看到可供使用的CNI插件：

```c
$ ls /opt/cni/bin/
```

CNI的工作过程大致如下图所示：

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

CNI通过JSON格式的配置文件来描述网络配置，当需要设置容器网络时，由容器运行时负责执行CNI插件，并通过CNI插件的标准输入（stdin）来传递配置文件信息，通过标准输出（stdout）接收插件的执行结果。图中的 libcni 是CNI提供的一个go package，封装了一些符合CNI规范的标准操作，便于容器运行时和网络插件对接CNI标准。

举一个直观的例子，假如我们要调用bridge插件将容器接入到主机网桥，则调用的命令看起来长这样：

```c
# CNI_COMMAND=ADD 顾名思义表示创建。
```

插件入参

容器运行时通过设置环境变量以及从标准输入传入的配置文件来向插件传递参数。

**环境变量**

- CNI_COMMAND ：定义期望的操作，可以是ADD，DEL，CHECK或VERSION。

- CNI_CONTAINERID ： 容器ID，由容器运行时管理的容器唯一标识符。

- CNI_NETNS：容器网络命名空间的路径。（形如 /run/netns/\[nsname\] )。

- CNI_IFNAME ：需要被创建的网络接口名称，例如eth0。

- CNI_ARGS ：运行时调用时传入的额外参数，格式为分号分隔的key-value对，例如 FOO=BAR;ABC=123

- CNI_PATH : CNI插件可执行文件的路径，例如/opt/cni/bin。

**配置文件**

文件示例：

```
{
```

**公共定义部分**

配置文件分为公共部分和插件定义部分。公共部分在CNI项目中使用结构体NetworkConfig定义：

```
type NetworkConfig struct {
```

- cniVersion 表示希望插件遵循的CNI标准的版本。

- name 表示网络名称。这个名称并非指网络接口名称，是便于CNI管理的一个表示。应当在当前主机(或其他管理域)上全局唯一。

- type 表示插件的名称，也就是插件对应的可执行文件的名称。

- bridge 该参数属于bridge插件的参数，指定主机网桥的名称。

- ipam 表示IP地址分配插件的配置，ipam.type 则表示ipam的插件类型。

更详细的信息，可以参考官方文档。

**插件定义部分**

上文提到，配置文件最终是传递给具体的CNI插件的，因此插件定义部分才是配置文件的“完全体”。公共部分定义只是为了方便各插件将其嵌入到自身的配置文件定义结构体中，举bridge插件为例：

```
type NetConf struct {
```

各插件的配置文件文档可参考官方文档。

**插件操作类型**

CNI插件的操作类型只有四种：ADD ， DEL ， CHECK 和 VERSION。插件调用者通过环境变量 CNI_COMMAND 来指定需要执行的操作。

**ADD**

ADD 操作负责将容器添加到网络，或对现有的网络设置做更改。具体地说，ADD 操作要么：

- 为容器所在的网络命名空间创建一个网络接口，或者

- 修改容器所在网络命名空间中的指定网络接口

例如通过 ADD 将容器网络接口接入到主机的网桥中。

其中网络接口名称由 CNI_IFNAME 指定，网络命名空间由 CNI_NETNS 指定。

**DEL**

DEL 操作负责从网络中删除容器，或取消对应的修改，可以理解为是 ADD 的逆操作。具体地说，DEL 操作要么：

- 为容器所在的网络命名空间删除一个网络接口，或者

- 撤销 ADD 操作的修改

例如通过 DEL 将容器网络接口从主机网桥中删除。

其中网络接口名称由 CNI_IFNAME 指定，网络命名空间由 CNI_NETNS 指定。

**CHECK**

CHECK 操作是v0.4.0加入的类型，用于检查网络设置是否符合预期。容器运行时可以通过CHECK来检查网络设置是否出现错误，当CHECK返回错误时（返回了一个非0状态码），容器运行时可以选择Kill掉容器，通过重新启动来重新获得一个正确的网络配置。

**VERSION**

VERSION 操作用于查看插件支持的版本信息。

```
$ CNI_COMMAND=VERSION /opt/cni/bin/bridge
```

链式调用

单个CNI插件的职责是单一的，比如bridge插件负责网桥的相关配置， firewall插件负责防火墙相关配置， portmap 插件负责端口映射相关配置。因此，当网络设置比较复杂时，通常需要调用多个插件来完成。CNI支持插件的链式调用，可以将多个插件组合起来，按顺序调用。例如先调用 bridge 插件设置容器IP，将容器网卡与主机网桥连通，再调用portmap插件做容器端口映射。容器运行时可以通过在配置文件设置plugins数组达到链式调用的目的：

```
{
```

细心的读者会发现，plugins这个字段并没有出现在上文描述的配置文件结构体中。的确，CNI使用了另一个结构体——NetworkConfigList来保存链式调用的配置：

```
type NetworkConfigList struct {
```

但CNI插件是不认识这个配置类型的。实际上，在调用CNI插件时，需要将NetworkConfigList转换成对应插件的配置文件格式，再通过标准输入（stdin）传递给CNI插件。例如在上面的示例中，实际上会先使用下面的配置文件调用 bridge 插件：

```
{
```

再使用下面的配置文件调用tuning插件：

```
{
```

需要注意的是，当插件进行链式调用的时候，不仅需要对NetworkConfigList做格式转换，而且需要将前一次插件的返回结果添加到配置文件中（通过prevResult字段），不得不说是一项繁琐而重复的工作。不过幸好libcni 已经为我们封装好了，容器运行时不需要关心如何转换配置文件，如何填入上一次插件的返回结果，只需要调用 libcni 的相关方法即可。

\*\*

3、示例

\*\*

接下来将演示如何使用CNI插件来为Docker容器设置网络。

下载CNI插件

为方便起见，我们直接下载可执行文件：

```
wget https://github.com/containernetworking/plugins/releases/download/v0.9.1/cni-plugins-linux-amd64-v0.9.1.tgz
```

如果你是在K8S节点上实验，通常节点上已经有CNI插件了，不需要再下载，但要注意将后续的 CNI_PATH 修改成/opt/cni/bin。

示例1——调用单个插件

在示例1中，我们会直接调用CNI插件，为容器设置eth0接口，为其分配IP地址，并接入主机网桥mynet0。

跟docker默认使用的使用网络模式一样，只不过我们将docker0换成了mynet0。

**启动容器**

虽然Docker不使用CNI规范，但可以通过指定 --net=none 的方式让Docker不设置容器网络。以nginx镜像为例：

```
contid=$(docker run -d --net=none --name nginx nginx) # 容器ID
```

启动容器的同时，我们需要记录一下容器ID，命名空间路径，方便后续传递给CNI插件。容器启动后，可以看到除了lo网卡，容器没有其他的网络设置：

```
nsenter -t $pid -n ip a
```

nsenter是namespace enter的简写，顾名思义，这是一个在某命名空间下执行命令的工具。-t表示进程ID, -n表示进入对应进程的网络命名空间。

**添加容器网络接口并连接主机网桥**

接下来我们使用bridge插件为容器创建网络接口，并连接到主机网桥。创建bridge.json配置文件，内容如下：

```
{
```

调用bridge插件ADD操作：

```
CNI_COMMAND=ADD CNI_CONTAINERID=$contid CNI_NETNS=$netnspath CNI_IFNAME=eth0 CNI_PATH=~/cni/bin ~/cni/bin/bridge < bridge.json
```

调用成功的话，会输出类似的返回值：

```
{
```

再次查看容器网络设置：

```
nsenter -t $pid -n ip a
```

可以看到容器中已经新增了eth0网络接口，并在ipam插件设定的子网下为其分配了IP地址。host-local类型的 ipam插件会将已分配的IP信息保存到文件，避免IP冲突，默认的保存路径为/var/lib/cni/network/$NETWORK_NAME：

**从主机访问验证**

由于mynet0是我们添加的网桥，还未设置路由，因此验证前我们需要先为容器所在的网段添加路由：

```
ip route add 10.10.0.0/16 dev mynet0 src 10.10.0.1 # 添加路由
```

**删除容器网络接口**

删除的调用入参跟添加的入参是一样的，除了CNI_COMMAND要替换成DEL：

```
CNI_COMMAND=DEL CNI_CONTAINERID=$contid CNI_NETNS=$netnspath CNI_IFNAME=eth0 CNI_PATH=~/cni/bin ~/cni/bin/bridge < bridge.json
```

注意，上述的删除命令并未清理主机的mynet0网桥。如果你希望删除主机网桥，可以执行ip link delete mynet0 type bridge命令删除。

示例2——链式调用

在示例2中，我们将在示例1的基础上，使用portmap插件为容器添加端口映射。

**使用cnitool工具**

前面的介绍中，我们知道在链式调用过程中，调用方需要转换配置文件，并需要将上一次插件的返回结果插入到本次插件的配置文件中。这是一项繁琐的工作，而libcni已经将这些过程封装好了，在示例2中，我们将使用基于 libcni的命令行工具cnitool来简化这些操作。

示例2将复用示例1中的容器，因此在开始示例2时，请确保已删除示例1中的网络接口。

通过源码编译或go install来安装cnitool：

```
go install github.com/containernetworking/cni/cnitool@latest
```

**配置文件**

libcni会读取.conflist后缀的配置文件，我们在当前目录创建portmap.conflist：

```
{
```

从上述的配置文件定义了两个CNI插件，bridge和portmap。根据上述的配置文件，cnitool会先为容器添加网络接口并连接到主机mynet0网桥上（就跟示例1一样），然后再调用portmap插件，将容器的80端口映射到主机的8080端口，就跟docker run -p 8080:80 xxx一样。

**设置容器网络**

使用cnitool我们还需要设置两个环境变量：

NETCONFPATH： 指定配置文件（\*.conflist）的所在路径，默认路径为 /etc/cni/net.d

CNI_PATH ：指定CNI插件的存放路径。

使用cnitool add命令为容器设置网络：

```
CNI_PATH=~/cni/bin NETCONFPATH=.  cnitool add portmap $netnspath
```

设置成功后，访问宿主机8080端口即可访问到容器的nginx服务。

**删除网络配置**

使用cnitool del命令删除容器网络：

```
CNI_PATH=~/cni/bin NETCONFPATH=.  cnitool del portmap $netnspath
```

注意，上述的删除命令并未清理主机的mynet0网桥。如果你希望删除主机网桥，可以执行ip link delete mynet0 type bridge命令删除。

**4、总结**

至此，CNI的工作原理我们已基本清楚。CNI的工作原理大致可以归纳为：

通过JSON配置文件定义网络配置；

通过调用可执行程序（CNI插件）来对容器网络执行配置；

通过链式调用的方式来支持多插件的组合使用。

CNI不仅定义了接口规范，同时也提供了一些内置的标准实现，以及libcni这样的“胶水层”，大大降低了容器运行时与网络插件的接入门槛。

参考

- CNI v0.4.0规范文档

- CNI master分支规范文档

- CNI内置插件文档

- cnitool 文档

- 为什么Kubernetes不使用CNM模型

- Introduction to CNI

- CNI deep dive

作者：水立方

来源：用户投稿

原文地址：https://juejin.cn/post/6986495816949039141

______________________________________________________________________

后台回复“加群”，带你进入高手如云交流群

**推荐阅读：**

[两个99%的人都遇到过的k8s故障处理技巧](http://mp.weixin.qq.com/s?__biz=MzI1OTY2MzMxOQ==&mid=2247496371&idx=1&sn=59afa39fd43c913a97d4f67673fece80&chksm=ea77c60bdd004f1dddbc95ea7f9ad6e49ff7d1ec525c9755bc5e8a3e8e5a3a9abd84287e1ab0&scene=21#wechat_redirect)\*\*[](http://mp.weixin.qq.com/s?__biz=MzI1OTY2MzMxOQ==&mid=2247496371&idx=1&sn=59afa39fd43c913a97d4f67673fece80&chksm=ea77c60bdd004f1dddbc95ea7f9ad6e49ff7d1ec525c9755bc5e8a3e8e5a3a9abd84287e1ab0&scene=21#wechat_redirect)\
\*\*

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

Reads 1515

​

Comment

**留言 1**

- CyDlen

  2021年9月12日

  Like2

  ![[苦涩]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)“但是由于技术和非技术原因” 听君一席话，如听君一席话

已无更多数据

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/1TDxR6xkRSFD7ibPKdP9JOqxFxp199B6LLCw80xbTAicxIQPgpsq5ib8kvx7eKfq6erLb1CAT4ZKa3RSnl9IWfELw/300?wx_fmt=png&wxfrom=18)

Linux云计算网络

5ShareWow

1

Comment

**留言 1**

- CyDlen

  2021年9月12日

  Like2

  ![[苦涩]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)“但是由于技术和非技术原因” 听君一席话，如听君一席话

已无更多数据
