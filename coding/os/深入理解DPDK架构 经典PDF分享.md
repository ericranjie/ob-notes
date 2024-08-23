# 
IEEEE

 _2021年10月28日 08:55_

以下文章来源于极客重生 ，作者极客重生

[

![](http://wx.qlogo.cn/mmhead/Q3auHgzwzM5dshlIEKpEhkUKbZoLeGqWdTZ4ia7z4wcOTuLo2U3gSSg/0)

**极客重生**.

技术学习分享，一起进步

](https://mp.weixin.qq.com/s?__biz=MzU4Mzc0NTcwNw==&mid=2247494832&idx=1&sn=54f2b802eb242cfca99469c4ac6fa299&chksm=fda6c574cad14c62773b6cd293d60e6d7e1f5aa6fe001e6fdbabbb71f856ddbb8c81a8e471b7&mpshare=1&scene=24&srcid=10286RDOQdGTtGDGQBsP7UTY&sharer_sharetime=1635386614734&sharer_shareid=5fb9813bfe9ffc983435bfc8d8c5e9ca&key=daf9bdc5abc4e8d080d5d40569d9bd39012dc4d2d49c9c4d0cd588508a1aa62a624de60d6339f4b92338e565853a58fbc78dc895f67a5d192ef95731de1b4125e5da41d8be311390fe5a899f31aa6bed1b61a25278e79c7717f0bbc76e77ea097b6de8eea640aac946deb1cda4999e2145404a2210d3242a53a1746808dddf7d&ascene=0&uin=MTEwNTU1MjgwMw%3D%3D&devicetype=Windows+11+x64&version=63090b19&lang=zh_CN&countrycode=CN&exportkey=n_ChQIAhIQQVb7SzpNjWomkK8P8OKCExLmAQIE97dBBAEAAAAAAAaaDcMCDn4AAAAOpnltbLcz9gKNyK89dVj002OFU8dLoMgH6kzUW0f0U%2B9vg%2Fyt7u%2BEPbgUvG1EYTnAY%2FEFA3LXpQxSH9WZKcF433%2BjiGVmOySoPFj0y81pUohtuXpKetTB7xC2VVlfiAcxB3h0RTeoJ9Awfz5iqhVuI5KqPQA9W0taXXdXKEbST9cQqMG5e9XS%2BvVhemxUJZvpTOP5jtUW0s9vo1OwTZvZSx22aG4DwNKSZ6XzbsMbxubNUlQhjvcdJUY1wzX3QsVArKTtdZ%2FlzPoTxVYkmLWj&acctmode=0&pass_ticket=E1G%2Fxnws45hEvFam2od%2FqL322KtTgJiqpLdfNOYAXa0Wgas9KgQEkqTSDqiIURO%2F&wx_header=1&fasttmpl_type=0&fasttmpl_fullversion=7351805-zh_CN-zip&fasttmpl_flag=1#)

![图片](https://mmbiz.qpic.cn/mmbiz_png/cYSwmJQric6nN9w8C9uDYQP1ucBmkXgbsewIibzfG3E7g8kAN7ibR3WbzbO8SBibVDhaLEO6FyuWxnWNNQpq0nAQKA/640?wx_fmt=png&wxfrom=13&tp=wxpic)

  

Intel DPDK全称Intel Data Plane Development Kit，是intel提供的数据平面开发工具集，为Intel architecture（IA）处理器架构下用户空间高效的数据包处理提供库函数和驱动的支持，它不同于Linux系统以通用性设计为目的，而是专注于网络应用中数据包的高性能处理。DPDK应用程序是运行在用户空间上利用自身提供的数据平面库来收发数据包，绕过了Linux内核协议栈对数据包处理过程。Linux内核将DPDK应用程序看作是一个普通的用户态进程，包括它的编译、连接和加载方式和普通程序没有什么两样。DPDK程序启动后只能有一个主线程，然后创建一些子线程并绑定到指定CPU核心上运行。

  

##### 背景

传统Linux网络驱动的问题

![图片](https://mmbiz.qpic.cn/mmbiz_png/cYSwmJQric6nN9w8C9uDYQP1ucBmkXgbs0iaM6RgS3Bw6PjyuRq6lDkuzPKlicxUGUeSwmb4LGXFQOoRmw1HFlRwg/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

##### 对比

![图片](https://mmbiz.qpic.cn/mmbiz_png/cYSwmJQric6nN9w8C9uDYQP1ucBmkXgbsPBsiaMJXCibfV5eic675FhNYU28PkmdEqbH8NibL0k4qo4ITxvXbkoSLNA/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

  

DPDK 有三大法宝

- ByPass Kernel , UIO/VFIO
    
- 微架构优化. Cache/DDIO/SIMD
    
- 内存管理. HugePage/mbuf/mempoo
    

  

  

![图片](https://mmbiz.qpic.cn/mmbiz_png/cYSwmJQric6nN9w8C9uDYQP1ucBmkXgbsXhnXGicVVU7ib5eHfp5V5bYVY4CNtd3BkYgJYickyIy1uKPNXr4DqpY0Q/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

  

左边是传统**内核数据通路**：数据从 网卡 -> 驱动 -> 协议栈 -> Socket接口 -> 业务

右边是DPDK的方式，**基于UIO（Userspace I/O）旁路数据**：数据从网卡 -> DPDK轮询模式-> DPDK基础库 -> 业务

  

详细参考：  

[

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

深入理解DPDK程序设计|Linux网络2.0







](http://mp.weixin.qq.com/s?__biz=MzkyMTIzMTkzNA==&mid=2247523649&idx=1&sn=5bcdd0efff2d2322df4af877ea61bfcd&chksm=c1846a10f6f3e3061cb336a623a28ec04cedb002f5bf5293175412b0973119d4625fac868870&scene=21#wechat_redirect)

  

##### 设计原理（文末有高清PDF获取方式）

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

  

##### DPDK组成

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

##### 详细内容

###### DPDK报文转发

######   
![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

###### 内存管理

###### ![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

###### 网卡性能优化

###### ![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

###### 网卡多队列

###### ![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

###### 硬件加速与功能卸载

###### ![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

###### DPDK内核驱动

  

###### ![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

###### 网络虚拟化

###### ![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

##### OVS DPDK

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

##### 网络存储优化SPDK

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

**编程指南**

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

高清完整版PDF，请在公众号里面回复"dpdk" 获取

《DPDK架构高清版.pdf》

《DPDK编程指南.pdf》

  

- END -

![](http://mmbiz.qpic.cn/mmbiz_png/nrgibEGYIoVJUicpguZaiajUVLb0EMHmIMknczOwIJQPx7QoStEXebjx0qmzdibHfxfXEg52o5bMMhtC56r3Q5rGJA/300?wx_fmt=png&wxfrom=19)

**IEEEE**

分享社会热点，矛盾焦点，技术新点，八卦笑点，除了这四点，还有六点 ......

222篇原创内容

公众号

  

阅读 477

​