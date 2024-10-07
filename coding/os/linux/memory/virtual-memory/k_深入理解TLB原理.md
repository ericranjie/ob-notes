Linux云计算网络  _2022年05月21日 15:21_ _广东_
![[Pasted image 20241007231459.png]]

来源：https://zhuanlan.zhihu.com/p/108425561

今天分享一篇TLB的好文章，希望大家夯实基本功，让我们一起深入理解计算机系统。

TLB是translation lookaside buffer的简称。首先，我们知道MMU的作用是把虚拟地址转换成物理地址。

![Image](https://mmbiz.qpic.cn/mmbiz_jpg/cYSwmJQric6lL2zLKT3ORjQp4K3KJkJEm3iaIjyd0HzicdyuzW4jIgkf5moYvlhI5zlVSoSSj6dkGY9WhIULKZChA/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

**MMU工作原理**  

虚拟地址和物理地址的映射关系存储在页表中，而现在页表又是分级的。64位系统一般都是3~5级。常见的配置是4级页表，就以4级页表为例说明。分别是PGD、PUD、PMD、PTE四级页表。在硬件上会有一个叫做页表基地址寄存器，它存储PGD页表的首地址。

  

![Image](https://mmbiz.qpic.cn/mmbiz_jpg/cYSwmJQric6lL2zLKT3ORjQp4K3KJkJEmDqzR7q5iacALbMq9avNUPfTpVJVMPJtkdB2HwzWuJHbP0SqKBuHOv2A/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

**Linux分页机制**  

MMU就是根据页表基地址寄存器从PGD页表一路查到PTE，最终找到物理地址(PTE页表中存储物理地址)。这就像在地图上显示你的家在哪一样，我为了找到你家的地址，先确定你是中国，再确定你是某个省，继续往下某个市，最后找到你家是一样的原理。一级一级找下去。这个过程你也看到了，非常繁琐。如果第一次查到你家的具体位置，我如果记下来你的姓名和你家的地址。下次查找时，是不是只需要跟我说你的姓名是什么，我就直接能够告诉你地址，而不需要一级一级查找。四级页表查找过程需要四次内存访问。延时可想而知，非常影响性能。页表查找过程的示例如下图所示。以后有机会详细展开，这里了解下即可。

  

![Image](https://mmbiz.qpic.cn/mmbiz_jpg/cYSwmJQric6lL2zLKT3ORjQp4K3KJkJEmrB1e7LAMEDbpPYQDoTaEpyiatoaaTHjMictDo2qQlppHCiadMVoHmSztg/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

**page table walk**

  

## **TLB的本质是什么**

TLB其实就是一块高速缓存。数据cache缓存地址(虚拟地址或者物理地址)和数据。TLB缓存虚拟地址和其映射的物理地址。TLB根据虚拟地址查找cache，它没得选，只能根据虚拟地址查找。所以TLB是一个虚拟高速缓存。硬件存在TLB后，虚拟地址到物理地址的转换过程发生了变化。虚拟地址首先发往TLB确认是否命中cache，如果cache hit直接可以得到物理地址。否则，一级一级查找页表获取物理地址。并将虚拟地址和物理地址的映射关系缓存到TLB中。既然TLB是虚拟高速缓存（VIVT），是否存在别名和歧义问题呢？如果存在，软件和硬件是如何配合解决这些问题呢？

## **TLB的特殊**

虚拟地址映射物理地址的最小单位是4KB。所以TLB其实不需要存储虚拟地址和物理地址的低12位(因为低12位是一样的，根本没必要存储)。另外，我们如果命中cache，肯定是一次性从cache中拿出整个数据。所以虚拟地址不需要offset域。index域是否需要呢？这取决于cache的组织形式。如果是全相连高速缓存。那么就不需要index。如果使用多路组相连高速缓存，依然需要index。下图就是一个四路组相连TLB的例子。现如今64位CPU寻址范围并没有扩大到64位。64位地址空间很大，现如今还用不到那么大。因此硬件为了设计简单或者解决成本，实际虚拟地址位数只使用了一部分。这里以48位地址总线为了例说明。

![Image](https://mmbiz.qpic.cn/mmbiz_jpg/cYSwmJQric6lL2zLKT3ORjQp4K3KJkJEmjRxuuppHnJ2H8jNpHticHsaI2uvEEF1RI8wmwPtv5d0kpSlBUh3ibqcA/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

## **TLB的别名问题**

我先来思考第一个问题，别名是否存在。我们知道PIPT的数据cache不存在别名问题。物理地址是唯一的，一个物理地址一定对应一个数据。但是不同的物理地址可能存储相同的数据。也就是说，物理地址对应数据是一对一关系，反过来是多对一关系。由于TLB的特殊性，存储的是虚拟地址和物理地址的对应关系。因此，对于单个进程来说，同一时间一个虚拟地址对应一个物理地址，一个物理地址可以被多个虚拟地址映射。将PIPT数据cache类比TLB，我们可以知道TLB不存在别名问题。而VIVT Cache存在别名问题，原因是VA需要转换成PA，PA里面才存储着数据。中间多经传一手，所以引入了些问题。

## **TLB的歧义问题**

我们知道不同的进程之间看到的虚拟地址范围是一样的，所以多个进程下，不同进程的相同的虚拟地址可以映射不同的物理地址。这就会造成歧义问题。例如，进程A将地址0x2000映射物理地址0x4000。进程B将地址0x2000映射物理地址0x5000。当进程A执行的时候将0x2000对应0x4000的映射关系缓存到TLB中。当切换B进程的时候，B进程访问0x2000的数据，会由于命中TLB从物理地址0x4000取数据。这就造成了歧义。如何消除这种歧义，我们可以借鉴VIVT数据cache的处理方式，在进程切换时将整个TLB无效。切换后的进程都不会命中TLB，但是会导致性能损失。

## **如何尽可能的避免flush TLB**

首先需要说明的是，这里的flush理解成使无效的意思。我们知道进程切换的时候，为了避免歧义，我们需要主动flush整个TLB。如果我们能够区分不同的进程的TLB表项就可以避免flush TLB。

我们知道Linux如何区分不同的进程？每个进程拥有一个独一无二的进程ID。如果TLB在判断是否命中的时候，除了比较tag以外，再额外比较进程ID该多好呢！这样就可以区分不同进程的TLB表项。进程A和B虽然虚拟地址一样，但是进程ID不一样，自然就不会发生进程B命中进程A的TLB表项。所以，TLB添加一项ASID(Address Space ID)的匹配。ASID就类似进程ID一样，用来区分不同进程的TLB表项。这样在进程切换的时候就不需要flush TLB。但是仍然需要软件管理和分配ASID。

![Image](https://mmbiz.qpic.cn/mmbiz_jpg/cYSwmJQric6lL2zLKT3ORjQp4K3KJkJEmeIGQGuS6DhW7eCNtSGricqQ4WTJz6HrQ4ft9XDtu2dhr105hBBGnn0Q/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

## **如何管理ASID**

ASID和进程ID肯定是不一样的，别混淆二者。进程ID取值范围很大。但是ASID一般是8或16 bit。所以只能区分256或65536个进程。我们的例子就以8位ASID说明。所以我们不可能将进程ID和ASID一一对应，我们必须为每个进程分配一个ASID，进程ID和每个进程的ASID一般是不相等的。每创建一个新进程，就为之分配一个新的ASID。当ASID分配完后，flush所有TLB，重新分配ASID。

所以，如果想完全避免flush TLB的话，理想情况下，运行的进程数目必须小于等于256。然而事实并非如此，因此管理ASID上需要软硬结合。Linux kernel为了管理每个进程会有个task_struct结构体，我们可以把分配给当前进程的ASID存储在这里。页表基地址寄存器有空闲位也可以用来存储ASID。当进程切换时，可以将页表基地址和ASID(可以从task_struct获得)共同存储在页表基地址寄存器中。当查找TLB时，硬件可以对比tag以及ASID是否相等(对比页表基地址寄存器存储的ASID和TLB表项存储的ASID)。如果都相等，代表TLB hit。否则TLB miss。当TLB miss时，需要多级遍历页表，查找物理地址。然后缓存到TLB中，同时缓存当前的ASID。

## **多个进程共享**

我们知道内核空间和用户空间是分开的，并且内核空间是所有进程共享。既然内核空间是共享的，进程A切换进程B的时候，如果进程B访问的地址位于内核空间，完全可以使用进程A缓存的TLB。但是现在由于ASID不一样，导致TLB miss。

我们针对内核空间这种全局共享的映射关系称之为global映射。针对每个进程的映射称之为non-global映射。所以，我们在最后一级页表中引入一个bit(non-global (nG) bit)代表是不是global映射。当虚拟地址映射物理地址关系缓存到TLB时，将nG bit也存储下来。当判断是否命中TLB时，当比较tag相等时，再判断是不是global映射，如果是的话，直接判断TLB hit，无需比较ASID。当不是global映射时，最后比较ASID判断是否TLB hit。

![Image](https://mmbiz.qpic.cn/mmbiz_jpg/cYSwmJQric6lL2zLKT3ORjQp4K3KJkJEmleXacPmoXRBeKiad4H8BR1IMpkYSRtHjQqDC7ZHyVGX0jZsiafpR8ia2A/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

  

## **什么时候应该flush TLB**

我们再来最后的总结，什么时候应该flush TLB。

- 当ASID分配完的时候，需要flush全部TLB，ASID的管理可以使用bitmap管理，flush TLB后clear整个bitmap。
    
- 当我们建立页表映射的时候，就需要flush虚拟地址对应的TLB表项。
    
    第一印象可能是修改页表映射的时候才需要flush TLB，但是实际情况是只要建立映射就需要flush TLB。原因是，建立映射时你并不知道之前是否存在映射，例如，建立虚拟地址A到物理地址B的映射，我们并不知道之前是否存在虚拟地址A到物理地址C的映射情况，所以就统一在建立映射关系的时候flush TLB。
    

  

n.net/yugemengjing/article/detai

---

后台回复“加群”，带你进入高手如云交流群

  

**推荐阅读：**

**[Go高性能编程技法解读](http://mp.weixin.qq.com/s?__biz=MzI1OTY2MzMxOQ==&mid=2247499710&idx=1&sn=f3e710297b8099c2c8be43cf09ef55bb&chksm=ea77cb06dd004210f5dab97e227cf9be9253b4dfcb1cdff918e446bbe2970c165562749b9465&scene=21#wechat_redirect)**

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

[网络排错大讲解~](http://mp.weixin.qq.com/s?__biz=MzI1OTY2MzMxOQ==&mid=2247488574&idx=1&sn=68df1982e9f23ce42457d00ce529b012&chksm=ea742086dd03a9902adb16d2b7648149209fed6c9811c6dd05be5101b42d462cb48e269b6e9d&scene=21#wechat_redirect)

[OVS 和 OVS-DPDK 对比](http://mp.weixin.qq.com/s?__biz=MzI1OTY2MzMxOQ==&mid=2247488294&idx=2&sn=303d9baefa768f887ba36213df793025&chksm=ea74279edd03ae88bc91300a1666066881dd763879b714941120fb9e41e787c557c2930ff5db&scene=21#wechat_redirect)

[微软出品的最新K8S学习指南3.0](http://mp.weixin.qq.com/s?__biz=MzI1OTY2MzMxOQ==&mid=2247488271&idx=1&sn=a52af9aba2ea8bbc5fd4440f85ece458&chksm=ea7427b7dd03aea1dbf106326168333c651b403dd44abf4e652d5a1912377d64becabac5051a&scene=21#wechat_redirect)下载

▼

  

_**_****喜欢，就给我一个****“在看”****_**_

  

![Image](https://mmbiz.qpic.cn/mmbiz_png/1TDxR6xkRSEBR9TP1Wsd64sicg7J9nbB41gbaHmcM73Yy5XkC5j8Sb3EV1ZGR8NZoKlZTposjm1IIdxibmngoobg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

**10T 技术资源大放送！包括但不限于：云计算、虚拟化、微服务、大数据、网络、**Linux、**Docker、Kubernetes、Python、Go、C/C++、Shell、PPT 等。在公众号内回复「****1024****」**，即可免费获！！****

​

![](https://mp.weixin.qq.com/rr?timestamp=1725966017&src=11&ver=1&signature=HN5ByWwx7lRoWIMFo0CFeYPk4cMiyD5mDT15f6koBa8h0dsRwkOYKSs1Wnc1bf6Twlb5iYUNGFgWEA-Ar2t1tzRHcG*7dnvFPiSovdanLN8=)

Scan to Follow