![](https://pic4.zhimg.com/80/v2-0bbd06e7c88bf3608e208414745d838d_1440w.jpg)

Intel DPDK全称Intel Data Plane Development Kit，是intel提供的数据平面[开发工具](https://zhida.zhihu.com/search?q=%E5%BC%80%E5%8F%91%E5%B7%A5%E5%85%B7&zhida_source=entity&is_preview=1)集，为Intel architecture（IA）处理器架构下[用户空间](https://zhida.zhihu.com/search?q=%E7%94%A8%E6%88%B7%E7%A9%BA%E9%97%B4&zhida_source=entity&is_preview=1)高效的数据包处理提供库函数和驱动的支持，它不同于Linux系统以通用性设计为目的，而是专注于网络应用中数据包的高性能处理。DPDK应用程序是运行在用户空间上利用自身提供的数据平面库来收发数据包，绕过了Linux[内核协议栈](https://zhida.zhihu.com/search?q=%E5%86%85%E6%A0%B8%E5%8D%8F%E8%AE%AE%E6%A0%88&zhida_source=entity&is_preview=1)对数据包处理过程。Linux内核将DPDK应用程序看作是一个普通的用户态进程，包括它的编译、连接和加载方式和[普通程序](https://zhida.zhihu.com/search?q=%E6%99%AE%E9%80%9A%E7%A8%8B%E5%BA%8F&zhida_source=entity&is_preview=1)没有什么两样。DPDK程序启动后只能有一个主线程，然后创建一些子线程并绑定到指定CPU核心上运行。

### **背景**

传统Linux网络驱动的问题

![](https://pic3.zhimg.com/80/v2-127a091929e937bc284d2af77879e330_1440w.webp)

### **对比**

![](https://pica.zhimg.com/80/v2-8b6801819920f805924bae81c2b5109a_1440w.webp)

DPDK 有[三大法宝](https://zhida.zhihu.com/search?q=%E4%B8%89%E5%A4%A7%E6%B3%95%E5%AE%9D&zhida_source=entity&is_preview=1)

- ByPass Kernel , UIO/VFIO
- [微架构](https://zhida.zhihu.com/search?q=%E5%BE%AE%E6%9E%B6%E6%9E%84&zhida_source=entity&is_preview=1)优化. Cache/DDIO/SIMD
- 内存管理. HugePage/mbuf/mempoo

![](https://pica.zhimg.com/80/v2-0c2eee8cc90445015005e48a0514389a_1440w.webp)

左边是传统**内核数据通路**：数据从 网卡 -> 驱动 -> 协议栈 -> Socket接口 -> 业务

右边是DPDK的方式，**基于UIO（Userspace I/O）旁路数据**：数据从网卡 -> DPDK轮询模式-> DPDK基础库 -> 业务

详细参考：

![](https://pic2.zhimg.com/80/v2-7a3cf9bdf0a6a610da32c6da79460cb5_1440w.webp)

[深入理解DPDK程序设计|Linux网络2.0](https://link.zhihu.com/?target=http%3A//mp.weixin.qq.com/s%3F__biz%3DMzkyMTIzMTkzNA%3D%3D%26mid%3D2247523649%26idx%3D1%26sn%3D5bcdd0efff2d2322df4af877ea61bfcd%26chksm%3Dc1846a10f6f3e3061cb336a623a28ec04cedb002f5bf5293175412b0973119d4625fac868870%26scene%3D21%23wechat_redirect)

### **设计原理（文末有高清PDF获取方式）**

![](https://pic3.zhimg.com/80/v2-1bd1edb90625c377d244e68fcc6a7f96_1440w.webp)

### **DPDK组成**

![](https://pic1.zhimg.com/80/v2-e0e53eac38c029348ac9b01802fa810e_1440w.webp)

### **详细内容**

### **DPDK报文转发**

### 

\*\*

![](https://pic4.zhimg.com/80/v2-3de0430ad694cd6e9b06186d61eaba1d_1440w.webp)

\*\*

### **内存管理**

\*\*

![](https://pica.zhimg.com/80/v2-83ea198547d08000e8ea690dcdea54de_1440w.webp)

\*\*

### **网卡性能优化**

\*\*

![](https://pica.zhimg.com/80/v2-5bef2e8b179e1fdf350becdb7544123c_1440w.webp)

\*\*

### **网卡多队列**

\*\*

![](https://picx.zhimg.com/80/v2-924a7ca6c5104d36bd78e57258426555_1440w.webp)

\*\*

### **硬件加速与功能卸载**

\*\*

![](https://pic4.zhimg.com/80/v2-a77ee3a64e20fedb4f77ee591a4dcad9_1440w.webp)

\*\*

### **DPDK内核驱动**

\*\*

![](https://pic2.zhimg.com/80/v2-0e3a68495766dadf6f51b48d5315c089_1440w.webp)

\*\*

### 

### **网络虚拟化**

\*\*

![](https://pic1.zhimg.com/80/v2-b1dfc9c7f0e8dcdd8eff7052c50ff97e_1440w.webp)

\*\*

### 

### **OVS DPDK**

![](https://pica.zhimg.com/80/v2-f1c1efcb38bb50726fdd89ec65b07228_1440w.webp)

### **网络存储优化SPDK**

![](https://pic3.zhimg.com/80/v2-1274921723f2c6474e71257baeed3f10_1440w.webp)

## **编程指南**

![](https://pic4.zhimg.com/80/v2-262ed913b1042e90964c73750c1d4c31_1440w.webp)

![](https://pic4.zhimg.com/80/v2-1d459c0950200446fbdb5d0da1dfab21_1440w.webp)

高清完整版PDF，请在公众号里面回复"dpdk" 获取

《DPDK架构高清版.pdf》

《DPDK编程指南.pdf》

- END -

**看完一键三连在看，转发，点赞**

**是对文章最大的赞赏，极客重生感谢你**

推荐阅读

[Linux Kernel TCP/IP Stack|Linux网络硬核系列](https://link.zhihu.com/?target=http%3A//mp.weixin.qq.com/s%3F__biz%3DMzkyMTIzMTkzNA%3D%3D%26mid%3D2247510568%26idx%3D1%26sn%3D79f335aaab5c0a36c0a66c5bfb1619ae%26chksm%3Dc1845d79f6f3d46f81b6fd24335eb8994c9daf21b6846d80af2cad73d9f638c5dda48b02892c%26scene%3D21%23wechat_redirect)

[深入理解DPDK程序设计|Linux网络2.0](https://link.zhihu.com/?target=http%3A//mp.weixin.qq.com/s%3F__biz%3DMzkyMTIzMTkzNA%3D%3D%26mid%3D2247523649%26idx%3D1%26sn%3D5bcdd0efff2d2322df4af877ea61bfcd%26chksm%3Dc1846a10f6f3e3061cb336a623a28ec04cedb002f5bf5293175412b0973119d4625fac868870%26scene%3D21%23wechat_redirect)

[TCP/IP协议栈到底是内核态好还是用户态好？](https://link.zhihu.com/?target=http%3A//mp.weixin.qq.com/s%3F__biz%3DMzkyMTIzMTkzNA%3D%3D%26mid%3D2247534895%26idx%3D2%26sn%3Dd043f0959dc1c85b7bb019d4a6191e67%26chksm%3Dc184be7ef6f337687f2952dbed55651d7b9ded35638890b690348857e31ae2e192645da31fe7%26scene%3D21%23wechat_redirect)

**最后**

推荐一个后端+底层开发[大本营](https://zhida.zhihu.com/search?q=%E5%A4%A7%E6%9C%AC%E8%90%A5&zhida_source=entity&is_preview=1)，帮你精通后端技术全栈，分享互联网大厂offer秘籍，校招，社招，内推，简历优化，模拟面试，跳槽，普升，职场进阶经验，都是过来人，帮你少走弯路，这里有关于职场的一切信息，分析IT职场的本质，帮助大家收获高薪和高级职位。

![](https://pic2.zhimg.com/80/v2-993e20927c0d8a68f4671a71bda5043d_1440w.webp)

二期线上直播分享内容：

● IT行业发展现状

● 编程语言的（go，C++,java，python等）选择，学习路线，大厂现状

● 如何训练扎实基本功

● 如何学习和攻破网络：关于网络的一切知识，[网络协议](https://zhida.zhihu.com/search?q=%E7%BD%91%E7%BB%9C%E5%8D%8F%E8%AE%AE&zhida_source=entity&is_preview=1)(TCP和UDP核心知识），[网络编程](https://zhida.zhihu.com/search?q=%E7%BD%91%E7%BB%9C%E7%BC%96%E7%A8%8B&zhida_source=entity&is_preview=1)（阻塞/非阻塞，同步/异常，reuse address/port，epoll/selecty底层原理，Reactor模型，C10M高并发高性能编程），网络架构，协议栈，网络排障等。

星球入口： [https://wx.zsxq.com/mweb/views](https://link.zhihu.com/?target=https%3A//wx.zsxq.com/mweb/views/joingroup/join_group.html%3Fgroup_id%3D51122582242854%26secret%3Df7knab6frb2zflji4in0pdwgbug8pf8n%26inviter_id%3D28514824185811%26share_from%3DShareToWechat%26keyword%3DbQrFamA)

[详细参考：极客分享](https://link.zhihu.com/?target=http%3A//mp.weixin.qq.com/s%3F__biz%3DMzkyMTIzMTkzNA%3D%3D%26mid%3D2247556775%26idx%3D2%26sn%3D8498ea3f9efea0925556ed6df7ec619f%26chksm%3Dc184e9f6f6f360e06bfc631b7d8af8ad8a396fca06395ac674b52549b96f9cc55c2702d6697d%26scene%3D21%23wechat_redirect)

欢迎大家加入极客星球，后端大本营，成为职场高薪玩家！
