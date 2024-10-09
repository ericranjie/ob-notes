点击关注 👉 顶级架构师
_2022年03月31日 17:33_
来源：juejin.cn/post/7062146999616798727
上一篇：[领域驱动设计（DDD）的几种典型架构介绍（图文详解）](http://mp.weixin.qq.com/s?__biz=MzIzNjM3MDEyMg==&mid=2247533791&idx=1&sn=35fe44b1f81b7cfc53adeaf5016c7d44&chksm=e8dae57adfad6c6c9249a71c4b7f06a04a6e201f1a2a356ca2ac9bd4ef362d6859a3977984dd&scene=21#wechat_redirect)

****大家好，我是顶级架构师。****

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/gHvX5TiczgWnhXnkWaR8rJ6TiaGRrFxU9IJ6pfvwphmfUuhtUfHuUGfYqCmRF2jibB5amEIkF56BEII7Rtns4A5aQ/640?wx_fmt=png&wxfrom=13)

Nginx 是一个高性能的 HTTP 和反向代理服务器，特点是占用内存少，并发能力强，事实上 Nginx 的并发能力确实在同类型的网页服务器中表现较好。\
Nginx 专为性能优化而开发，性能是其最重要的要求，十分注重效率，有报告 Nginx 能支持高达 50000 个并发连接数。

**01** **Nginx 知识网结构图**
Nginx 的知识网结构图如下：

![图片](https://mmbiz.qpic.cn/mmbiz/K0TMNq37VN28Cq2l76Q0pUpBuvBWcmI4J9ibicG7IqjUjAmXZ37WNrvN0BXjtF6bmroO9Foxm4eQz9mPJ580pSdg/640?wx_fmt=jpeg&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

**02\*\*\*\*反向代理**

\*\*正向代理：\*\*局域网中的电脑用户想要直接访问网络是不可行的，只能通过代理服务器来访问，这种代理服务就被称为正向代理。

![图片](https://mmbiz.qpic.cn/mmbiz/K0TMNq37VN28Cq2l76Q0pUpBuvBWcmI4nHjmfqAJ909p4IxXsyEVw3ibTbDtWVOR3XI4zg3jibsjuJ0BKGnAmlAg/640?wx_fmt=jpeg&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

\*\*反向代理：\*\*客户端无法感知代理，因为客户端访问网络不需要配置，只要把请求发送到反向代理服务器，由反向代理服务器去选择目标服务器获取数据，然后再返回到客户端。

此时反向代理服务器和目标服务器对外就是一个服务器，暴露的是代理服务器地址，隐藏了真实服务器 IP 地址。

![图片](https://mmbiz.qpic.cn/mmbiz/K0TMNq37VN28Cq2l76Q0pUpBuvBWcmI4RAt9Hicic96l0QwHOmgC3WicsN67kDZgmkvs7pr5ENicdy8atic1FK90mKw/640?wx_fmt=jpeg&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

**03**

**负载均衡**

客户端发送多个请求到服务器，服务器处理请求，有一些可能要与数据库进行交互，服务器处理完毕之后，再将结果返回给客户端。

普通请求和响应过程如下图：

![图片](https://mmbiz.qpic.cn/mmbiz/K0TMNq37VN28Cq2l76Q0pUpBuvBWcmI4DtTUsXwwBbM3Q9WFQjzPjIOCB09KN7dXiamticEzn2RsjnJyYlGSqDAg/640?wx_fmt=jpeg&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

但是随着信息数量增长，访问量和数据量飞速增长，普通架构无法满足现在的需求。
我们首先想到的是升级服务器配置，可以由于摩尔定律的日益失效，单纯从硬件提升性能已经逐渐不可取了，怎么解决这种需求呢？
我们可以增加服务器的数量，构建集群，将请求分发到各个服务器上，将原来请求集中到单个服务器的情况改为请求分发到多个服务器，也就是我们说的负载均衡。

图解负载均衡：

![图片](https://mmbiz.qpic.cn/mmbiz/K0TMNq37VN28Cq2l76Q0pUpBuvBWcmI4EwZkaLS3ZibyHvaSiaXMA6tStFdIqicWYGO9UDnAHAKOjiaIkvhAkwGAAw/640?wx_fmt=jpeg&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

假设有 15 个请求发送到代理服务器，那么由代理服务器根据服务器数量，平均分配，每个服务器处理 5 个请求，这个过程就叫做负载均衡。

**04\*\*\*\*动静分离**

为了加快网站的解析速度，可以把动态页面和静态页面交给不同的服务器来解析，加快解析的速度，降低由单个服务器的压力。扩展：[接私活儿](http://mp.weixin.qq.com/s?__biz=MzkxMDI2NzUxNA==&mid=2247487905&idx=1&sn=807099c4af99516ea78a743faec8b994&chksm=c12f54b4f658dda2d0c3aba752d4f5d89769ed9e04a863a0b28887c0628155013c4b9382af5c&scene=21#wechat_redirect)

动静分离之前的状态：
!\[\[Pasted image 20241006112144.png\]\]
!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

动静分离之后：
!\[\[Pasted image 20241006112149.png\]\]
!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**05\*\*\*\*Nginx安装**

Nginx 如何在 Linux 安装
参考链接：
`https://blog.csdn.net/yujing1314/article/details/97267369`

Nginx 常用命令

查看版本：

`./nginx -v      `

启动：

`./nginx      `

关闭（有两种方式，推荐使用 ./nginx -s quit）：

`./nginx -s stop ./nginx -s quit`

重新加载 Nginx 配置：

`./nginx -s reload      `

Nginx 的配置文件

配置文件分三部分组成：

##### **①全局块**

从配置文件开始到 events 块之间，主要是设置一些影响 Nginx 服务器整体运行的配置指令。

并发处理服务的配置，值越大，可以支持的并发处理量越多，但是会受到硬件、软件等设备的制约。
!\[\[Pasted image 20241006112207.png\]\]
!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

##### **②events 块**

影响 Nginx 服务器与用户的网络连接，常用的设置包括是否开启对多 workprocess 下的网络连接进行序列化，是否允许同时接收多个网络连接等等。

支持的最大连接数：
!\[\[Pasted image 20241006112210.png\]\]
!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

##### **③HTTP 块**

诸如反向代理和负载均衡都在此配置。

`location[ = | ~ | ~* | ^~] url{}      `

location 指令说明，该语法用来匹配 url，语法如上：

- \*\*=：\*\*用于不含正则表达式的 url 前，要求字符串与 url 严格匹配，匹配成功就停止向下搜索并处理请求。

- \*\*~：\*\*用于表示 url 包含正则表达式，并且区分大小写。

- \*\*~\*：\*\*用于表示 url 包含正则表达式，并且不区分大小写。

- \*\*^~：\*\*用于不含正则表达式的 url 前，要求 Nginx 服务器找到表示 url 和字符串匹配度最高的 location 后，立即使用此 location 处理请求，而不再匹配。

- 如果有 url 包含正则表达式，不需要有 ~ 开头标识。

**06\*\*\*\*反向代理实战**

### **①配置反向代理**

目的：在浏览器地址栏输入地址 www.123.com 跳转 Linux 系统 Tomcat 主页面。

**②具体实现**

先配置 Tomcat，因为比较简单，此处不再赘叙，并在 Windows 访问：
!\[\[Pasted image 20241006112224.png\]\]
!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

具体流程如下图：
!\[\[Pasted image 20241006112227.png\]\]
!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

修改之前：
!\[\[Pasted image 20241006112232.png\]\]
!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

配置如下：
!\[\[Pasted image 20241006112237.png\]\]
!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

再次访问：
!\[\[Pasted image 20241006112240.png\]\]
!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**③反向代理 2**
目标：

- 访问 http://192.168.25.132:9001/edu/ 直接跳转到 192.168.25.132:8080
- 访问 http://192.168.25.132:9001/vod/ 直接跳转到 192.168.25.132:8081

\*\*准备：\*\*配置两个 Tomcat，端口分别为 8080 和 8081，都可以访问，端口修改配置文件即可。另外，搜索公众号顶级算法后台回复“私活神器”，获取一份惊喜礼包。
!\[\[Pasted image 20241006113054.png\]\]
!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)
!\[\[Pasted image 20241006113057.png\]\]
!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

新建文件内容分别添加 8080！！！和 8081！！！
!\[\[Pasted image 20241006113117.png\]\]
!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)
!\[\[Pasted image 20241006113121.png\]\]
!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

响应如下图：
!\[\[Pasted image 20241006113128.png\]\]
!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)
!\[\[Pasted image 20241006113144.png\]\]
!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

具体配置如下：
!\[\[Pasted image 20241006113151.png\]\]
!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

重新加载 Nginx：

`./nginx -s reload      `

访问：
!\[\[Pasted image 20241006113157.png\]\]
!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)
!\[\[Pasted image 20241006113202.png\]\]
!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

实现了同一个端口代理，通过 edu 和 vod 路径的切换显示不同的页面。

**反向代理小结**

\*\*第一个例子：\*\*浏览器访问 www.123.com，由 host 文件解析出服务器 ip 地址\
192.168.25.132 www.123.com。

然后默认访问 80 端口，而通过 Nginx 监听 80 端口代理到本地的 8080 端口上，从而实现了访问 www.123.com，最终转发到 tomcat 8080 上去。

第二个例子：

- 访问 http://192.168.25.132:9001/edu/ 直接跳转到 192.168.25.132:8080
- 访问 http://192.168.25.132:9001/vod/ 直接跳转到 192.168.25.132:8081

实际上就是通过 Nginx 监听 9001 端口，然后通过正则表达式选择转发到 8080 还是 8081 的 Tomcat 上去。

**07**

**负载均衡实战**

①修改 nginx.conf，如下图：
!\[\[Pasted image 20241006113231.png\]\]
!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)
!\[\[Pasted image 20241006113236.png\]\]
!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

②重启 Nginx：

`./nginx -s reload      `

③在 8081 的 Tomcat 的 webapps 文件夹下新建 edu 文件夹和 a.html 文件，填写内容为 8081！！！！

④在地址栏回车，就会分发到不同的 Tomcat 服务器上：
!\[\[Pasted image 20241006113240.png\]\]
!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)
!\[\[Pasted image 20241006113243.png\]\]
!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

负载均衡方式如下：

- 轮询（默认）。

- weight，代表权，权越高优先级越高。

- fair，按后端服务器的响应时间来分配请求，相应时间短的优先分配。

- ip_hash，每个请求按照访问 ip 的 hash 结果分配，这样每一个访客固定的访问一个后端服务器，可以解决 Session 的问题。

!\[\[Pasted image 20241006113255.png\]\]
!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

______________________________________________________________________

!\[\[Pasted image 20241006113257.png\]\]
!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

______________________________________________________________________

!\[\[Pasted image 20241006113302.png\]\]

## !\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**08**

## **动静分离实战**

## 什么是动静分离？把动态请求和静态请求分开，不是讲动态页面和静态页面物理分离，可以理解为 Nginx 处理静态页面，Tomcat 处理动态页面。

动静分离大致分为两种：

- 纯粹将静态文件独立成单独域名放在独立的服务器上，也是目前主流方案。[关注Java后端栈](http://mp.weixin.qq.com/s?__biz=MzkwOTI2ODY1Mg==&mid=2247484529&idx=1&sn=8d377f48747ab0bb00050cd4fed9e032&chksm=c13c07b2f64b8ea4f8f666988da96db4e3c0818318c819ddf82829a6c1ae1e63c4eb0e3bb7e9&scene=21#wechat_redirect)

- 将动态跟静态文件混合在一起发布，通过 Nginx 分开。

#### 动静分离图析：

!\[\[Pasted image 20241006113311.png\]\]
!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

实战准备，准备静态文件：
!\[\[Pasted image 20241006113316.png\]\]
!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)
!\[\[Pasted image 20241006113319.png\]\]
!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

配置 Nginx，如下图：
!\[\[Pasted image 20241006113323.png\]\]
!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

Nginx 高可用

如果 Nginx 出现问题：
!\[\[Pasted image 20241006113327.png\]\]
!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

解决办法：
!\[\[Pasted image 20241006113331.png\]\]!\[\[Pasted image 20241006113344.png\]\]
!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

前期准备：

- **两台 Nginx 服务器**

- **安装 Keepalived**

- **虚拟 ip**

## 安装 Keepalived：

`[root@192 usr]# yum install keepalived -y[root@192 usr]# rpm -q -a keepalivedkeepalived-1.3.5-16.el7.x86_64      `

修改配置文件：

`[root@192 keepalived]# cd /etc/keepalived[root@192 keepalived]# vi keepalived.conf      `

分别将如下配置文件复制粘贴，覆盖掉 keepalived.conf，虚拟 ip 为 192.168.25.50。

对应主机 ip 需要修改的是：

- smtp_server 192.168.25.147（主）smtp_server 192.168.25.147（备）

- state MASTER（主） state BACKUP（备）

`global_defs {   notification_email {     acassen@firewall.loc`     `failover@firewall.loc     sysadmin@firewall.loc   }   notification_email_from Alexandre.Cassen@firewall.loc   smtp_server 192.168.25.147   smtp_connect_timeout 30   router_id LVS_DEVEL # 访问的主机地址}vrrp_script chk_nginx {  script "/usr/local/src/nginx_check.sh"  # 检测文件的地址  interval 2   # 检测脚本执行的间隔  weight 2   # 权重}vrrp_instance VI_1 {    state BACKUP    # 主机MASTER、备机BACKUP        interface ens33   # 网卡    virtual_router_id 51 # 同一组需一致    priority 90  # 访问优先级，主机值较大，备机较小    advert_int 1    authentication {        auth_type PASS        auth_pass 1111    }    virtual_ipaddress {        192.168.25.50  # 虚拟ip    }}`

启动代码如下：

`[root@192 sbin]# systemctl start keepalived.service`
!\[\[Pasted image 20241006113346.png\]\]
!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

访问虚拟 ip 成功：
!\[\[Pasted image 20241006113351.png\]\]
!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

关闭主机 147 的 Nginx 和 Keepalived，发现仍然可以访问。

原理解析
!\[\[Pasted image 20241006113355.png\]\]
!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

如下图，就是启动了一个 master，一个 worker，master 是管理员，worker是具体工作的进程。
!\[\[Pasted image 20241006113400.png\]\]
!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

worker 如何工作？如下图：
!\[\[Pasted image 20241006113404.png\]\]
!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

小结

worker 数应该和 CPU 数相等；一个 master 多个 worker 可以使用热部署，同时 worker 是独立的，一个挂了不会影响其他的。

最后给读者整理了一份**BAT**大厂面试真题，需要的可扫码回复“**面试题**”即可获取。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

****!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)****

公众号后台回复 **架构** 或者 **架构整洁** 有惊喜礼包！

**顶级架构师交流群**

**「顶级架构师」建立了读者架构师交流群，大家可以添加小编微信进行加群。欢迎有想法、乐于分享的朋友们一起交流学习。**

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

扫描添加好友邀你进架构师群，加我时注明\*\*【\*\***姓名+公司+职位】**

**版权申明：内容来源网络，版权归原作者所有。如有侵权烦请告知，我们会立即删除并表示歉意。谢谢。**

猜你还想看

[推荐一套开源通用后台管理系统（附源码）](http://mp.weixin.qq.com/s?__biz=MzIzNjM3MDEyMg==&mid=2247517463&idx=1&sn=39841b9fb02e185d4cb06df7f5b7cc02&chksm=e8da24b2dfadada4eea431a3df5ea184203cdc1cf4f833549f80956328901c5e005f8d388583&scene=21#wechat_redirect)

[看看人家那 IM 即时通讯系统，那叫一个优雅（附源码）](http://mp.weixin.qq.com/s?__biz=MzIzNjM3MDEyMg==&mid=2247520498&idx=1&sn=247bed1ab2e30cad54981a3eeff00810&chksm=e8da3157dfadb84118310e1067c0af1e8fc3e41b15d2dfde8ab61889b8522f603bbda08bed8c&scene=21#wechat_redirect)

[面试官：生成订单30分钟未支付，则自动取消，该怎么实现？](http://mp.weixin.qq.com/s?__biz=MzIzNjM3MDEyMg==&mid=2247520660&idx=1&sn=ea62f593bebb58a33bb77b7e65ec8a86&chksm=e8da3031dfadb927fd40c6608bc727bacc53309afcf868f10b5ba2c3153024904c92332fe5ef&scene=21#wechat_redirect)

[阿里技术专家：一文教你高效画出技术架构图](http://mp.weixin.qq.com/s?__biz=MzIzNjM3MDEyMg==&mid=2247506842&idx=1&sn=5387bf6dca96e01d5ce00400a91dde3e&chksm=e8da7a3fdfadf329bb65307a9a830e80911f1118a3e6878f0f6ed530e0940b43bf505655dfea&scene=21#wechat_redirect)

[16个 Redis 常见使用场景，面试有内容聊啦](http://mp.weixin.qq.com/s?__biz=MzIzNjM3MDEyMg==&mid=2247516679&idx=1&sn=78f34a21846352643195ae2a36b13f67&chksm=e8da23a2dfadaab4bf299ab717a2b5dcc33efb9a4bfc10bb00009317d08c06227297dd209f3c&scene=21#wechat_redirect)

[面试官问：MySQL的自增 ID 用完了，怎么办？](http://mp.weixin.qq.com/s?__biz=MzIzNjM3MDEyMg==&mid=2247517190&idx=2&sn=737c69e45fceb1e672bc81b2141d6f5d&chksm=e8da25a3dfadacb55a8c96b8fb78c8aa3c385284c5e591fd3509bc4521114795a99184dbcf17&scene=21#wechat_redirect)[](http://mp.weixin.qq.com/s?__biz=MzIzNjM3MDEyMg==&mid=2247523669&idx=2&sn=8fa8b3492f10fbee1b3308f2e53d1519&chksm=e8da3cf0dfadb5e6c44612004413499ae32efd0121df83e426d63bcb8fe6380a3a7cf5d2a0bd&scene=21#wechat_redirect)

[知名国产论坛，凉了！！！！](http://mp.weixin.qq.com/s?__biz=MzIzNjM3MDEyMg==&mid=2247523798&idx=1&sn=e6d5fe0f220f8be30d01c4b3c9d024f3&chksm=e8da3c73dfadb56578a7bb03956c9009b86daa3682ea6a8f09456abc2be939a37e01cf817bbc&scene=21#wechat_redirect)

[Elasticsearch 写入优化记录，从3000到8000/s](http://mp.weixin.qq.com/s?__biz=MzIzNjM3MDEyMg==&mid=2247533738&idx=2&sn=bac4033a49aed0c1070ffe1f95671ea8&chksm=e8dae50fdfad6c19b3e9c5ff6410ba122d94528f90561986f17872dc53fd6681dce56bd755fe&scene=21#wechat_redirect)

[分布式存储的七方面问题](http://mp.weixin.qq.com/s?__biz=MzIzNjM3MDEyMg==&mid=2247533791&idx=2&sn=72ae93197287f467dd6639bf9a8162b7&chksm=e8dae57adfad6c6cdafc37a120d587c0aa52c9ce3a5f3f7a9f7fe627186ca2bb5b3ca63dbcec&scene=21#wechat_redirect)

[订单30分钟未支付，则自动取消，该怎么实现？](http://mp.weixin.qq.com/s?__biz=MzIzNjM3MDEyMg==&mid=2247533872&idx=2&sn=603d0f1f2045a4189e239e7f9837a968&chksm=e8dae495dfad6d8315798ca7844149212c3708241520ffc5ab20a13ab76938d15ef50c7ad8f5&scene=21#wechat_redirect)

[多账号统一登陆，账号模块的系统设计](http://mp.weixin.qq.com/s?__biz=MzIzNjM3MDEyMg==&mid=2247533935&idx=2&sn=afbaf07ca3d584936a7af3b921f17a36&chksm=e8dae4cadfad6ddc7e09f0e14c74f04c06d60b68a1bd3500cf6fb06a220f04949da3e8b04a0c&scene=21#wechat_redirect)

[微服务架构统一安全认证设计与实践](http://mp.weixin.qq.com/s?__biz=MzIzNjM3MDEyMg==&mid=2247533715&idx=2&sn=23796e219029ffe26bf51a12528bdd91&chksm=e8dae536dfad6c200be43350b949f9a769eece0def34ed0896ef8d3288ccb6d381ce491d383e&scene=21#wechat_redirect)

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

阅读原文

阅读 1022

​

写留言

**留言 1**

- 逐梦、少年

  2022年3月31日

  赞1

  这文章的反向代理有一处的配置没配，localhost里面没有proxy_set_header Host $host;这行代码是无法进行反向代理的，会报404错误，可以去实践一下，网上的总结大多数没有这行代码

已无更多数据

[](javacript:;)

![](http://mmbiz.qpic.cn/sz_mmbiz_png/gHvX5TiczgWlCsOPBib3qa34WKGOy72FcvqTvt9icWjB0223JqDtJtD25EmBcaFxlJJ8P2r6KEADI3KYw7H1zuMRg/300?wx_fmt=png&wxfrom=18)

顶级架构师

10分享7

1

写留言

**留言 1**

- 逐梦、少年

  2022年3月31日

  赞1

  这文章的反向代理有一处的配置没配，localhost里面没有proxy_set_header Host $host;这行代码是无法进行反向代理的，会报404错误，可以去实践一下，网上的总结大多数没有这行代码

已无更多数据
