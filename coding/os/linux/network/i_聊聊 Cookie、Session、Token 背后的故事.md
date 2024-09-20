# 

Linux爱好者

 _2021年12月11日 11:52_

以下文章来源于后端研究所 ，作者大白斯基

[

![](http://wx.qlogo.cn/mmhead/Q3auHgzwzM7Lhjgjq6G9fRPYpJaHf91EX8kJWRaYlzWic5enxibYmiacg/0)

**后端研究所**.

专注后端开发，记录所思所得。

](https://mp.weixin.qq.com/s?__biz=MzAxODI5ODMwOA==&mid=2666559737&idx=2&sn=2e3084be35540944314dfa440ef41724&chksm=80dcbc52b7ab3544b811b2fc1cad773ce2f6dfae6e64d0242137da2d05cd3069a559d4cfcd41&mpshare=1&scene=24&srcid=1211vkRzgWUpjqWL8Ul2ga5y&sharer_sharetime=1639197183726&sharer_shareid=5fb9813bfe9ffc983435bfc8d8c5e9ca&key=daf9bdc5abc4e8d0a31ece367b59be17faae18767315c2300e930d7dea84ef192d28ea5bd55e8d6a9de017fd8365ec47bfbd02bde987e5e1612f77d5ad8458216910513aba2a825088fd70c2a0f97bc26b8022eb8d4c36f1ccde8a58261121bf6662277ee344b7fd457301cf669e70b9d078aabf912a4c6db477e9c4e8f6944b&ascene=0&uin=MTEwNTU1MjgwMw%3D%3D&devicetype=Windows+11+x64&version=63090b19&lang=zh_CN&countrycode=CN&exportkey=n_ChQIAhIQo%2B5L4ZO0S92sfGwCNZnNUxLmAQIE97dBBAEAAAAAAJbgIHN9EJoAAAAOpnltbLcz9gKNyK89dVj0TLBHF3cIDuDyleu%2FOvpmmHG3VGKbUxGsje2b1FmNXJ8qoYyHfqf%2FXEfGryF2BjVhSLNIt3rJJg3UpUlXNBOJFrCdRj%2FIBcma%2FVnLV6gRT%2BBk7O%2FVRHkm3vbYgPbB06hLOsGVXPGGBuOfDv9cPowC%2B4flI%2FMg8hyftvqYDIoGvR52X2Bw%2Fq9U42mxXcFmXdaDOuMKgvfXsjXGMH3uQTMFwDILZjHiPnwZ51Xlzt5Z4hnU2BVyct5NTNa4LhkVcU%2Bb&acctmode=0&pass_ticket=RGl%2BQdda20deyUII1NWeGEA0vjSHrLaOdXrB%2Fa3W44RK6C6WZSIo%2BToJOd88hRZr&wx_header=1&fasttmpl_type=0&fasttmpl_fullversion=7351805-zh_CN-zip&fasttmpl_flag=1#)

今天和大家聊一下关于cookie、session、token的那些事儿  

这是一个读者朋友面试微信的实习岗位时遇到的，在此和大家分享一下

![图片](https://mmbiz.qpic.cn/mmbiz_png/wAkAIFs11qaeduHjZ3hQO8VLRdhqqNIPGcic9WkcUfPLxGDON1dK2FhzZJwh6xxqh12zrIXkxVBsiciaWMOiatz1MA/640?wx_fmt=png&wxfrom=13&tp=wxpic)

话不多说，直接开车

## 1. 网站交互体验升级

作为网友的我们，每天都会使用浏览器来逛各种网站，来满足日常的工作生活需求。

![图片](https://mmbiz.qpic.cn/mmbiz_png/wAkAIFs11qaeduHjZ3hQO8VLRdhqqNIPH0394u9jdQINrWUVicZ20icfEuXye8pKPR8T6tjBXKhEZzdOTcfrBCkQ/640?wx_fmt=png&wxfrom=13&tp=wxpic)

现在的交互体验还是很丝滑的，但早期并非如此，而是一锤子买卖。

### 1.1 无状态的http协议

无状态的http协议是什么鬼？

HTTP无状态协议，是指协议对于业务处理没有记忆能力，之前做了啥完全记不住，每次请求都是完全独立互不影响的，没有任何上下文信息。

缺少状态意味着如果后续处理需要前面的信息，则它必须重传关键信息，这样可能导致每次连接传送的数据量增大。

如果大家没明白，可以想一下《夏洛特烦恼》里面的桥段：![图片](https://mmbiz.qpic.cn/mmbiz_png/wAkAIFs11qaeduHjZ3hQO8VLRdhqqNIPh5PPDicH5sPs9zaN7ibBibjYa4P67v0lEHBX9FEP1XrgcUEAIAAtrq7kw/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

大概明白了吧，假如一直用这种原生无状态的http协议，我们每换一个页面可能就得重新登录一次，那还玩个球。

所以必须要解决http协议的无状态，提升网站的交互体验，否则星辰大海是去不了的。

### 1.2 解决之道

整个事情交互的双方只有客户端和服务端，所以必然要在这两个当事者身上下手。

- **客户端来买单**  
    客户端每次请求时把自己必要的信息封装发送给服务端，服务端查收处理一下就行。
    
![[Pasted image 20240920123404.png]]
![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

- **服务端来买单**  
    客户端第一次请求之后，服务端就开始做记录，然后客户端在后续请求中只需要将最基本最少的信息发过来就行，不需要太多信息了。
    
![[Pasted image 20240920123411.png]]
![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

## 2.Cookie方案

Cookie总是保存在客户端中，按在客户端中的存储位置，可分为内存Cookie和硬盘Cookie，内存Cookie由浏览器维护，保存在内存中，浏览器关闭后就消失了，其存在时间是短暂的，硬盘Cookie保存在硬盘里，有一个过期时间，除非用户手工清理或到了过期时间，硬盘Cookie不会被删除，其存在时间是长期的。
![[Pasted image 20240920123419.png]]
![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

### 2.1 Cookie定义和作用

HTTP Cookie（也叫 Web Cookie 或浏览器 Cookie）是服务器发送到用户浏览器并保存在本地的一小块数据，它会在浏览器下次向同一服务器再发起请求时被携带并发送到服务器上。

通常Cookie用于告知服务端两个请求是否来自同一浏览器，如保持用户的登录状态。Cookie 使基于无状态的HTTP协议记录稳定的状态信息成为了可能。

Cookie 主要用于以下三个方面：

- 会话状态管理（如用户登录状态、购物车等其它需要记录的信息）
    
- 个性化设置（如用户自定义设置、主题等）
    
- 浏览器行为跟踪（如跟踪分析用户行为等）
    

### 2.2 服务端创建Cookie

当服务器收到 HTTP 请求时，服务器可以在响应头里面添加一个 Set-Cookie 选项。

浏览器收到响应后通常会保存下 Cookie，之后对该服务器每一次请求中都通过  Cookie 请求头部将 Cookie 信息发送给服务器。另外，Cookie 的过期时间、域、路径、有效期、适用站点都可以根据需要来指定。
![[Pasted image 20240920123427.png]]
![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

### 2.3 B/S的Cookie交互
![[Pasted image 20240920123434.png]]
![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

服务器使用 Set-Cookie 响应头部向用户浏览器发送 Cookie信息。

一个简单的 Cookie 可能像这样：

`Set-Cookie: <cookie名>=<cookie值>      HTTP/1.0 200 OK   Content-type: text/html   Set-Cookie: yummy_cookie=choco   Set-Cookie: tasty_cookie=strawberry   `

客户端对该服务器发起的每一次新请求，浏览器都会将之前保存的Cookie信息通过 Cookie 请求头部再发送给服务器。

`GET /sample_page.html HTTP/1.1   Host: www.example.org   Cookie: yummy_cookie=choco; tasty_cookie=strawberry   `

我来访问下淘宝网，抓个包看看这个真实的过程：
![[Pasted image 20240920123442.png]]
![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

### 2.4 存在的问题

Cookie 常用来标记用户或授权会话，被浏览器发出之后可能被劫持，被用于非法行为，可能导致授权用户的会话受到攻击，因此存在安全问题。

还有一种情况就是跨站请求伪造CSRF，简单来说 比如你在登录银行网站的同时，登录了一个钓鱼网站，在钓鱼网站进行某些操作时可能会获取银行网站相关的Cookie信息，向银行网站发起转账等非法行为。

> 跨站请求伪造（英语：Cross-site request forgery），也被称为 one-click attack 或者 session riding，通常缩写为 CSRF 或者 XSRF， 是一种挟制用户在当前已登录的Web应用程序上执行非本意的操作的攻击方法。跟跨网站脚本（XSS）相比，XSS 利用的是用户对指定网站的信任，CSRF 利用的是网站对用户网页浏览器的信任。

> 跨站请求攻击，简单地说，是攻击者通过一些技术手段欺骗用户的浏览器去访问一个自己曾经认证过的网站并运行一些操作（如发邮件，发消息，甚至财产操作如转账和购买商品）。  
> 由于浏览器曾经认证过，所以被访问的网站会认为是真正的用户操作而去运行。这利用了web中用户身份验证的一个漏洞：简单的身份验证只能保证请求发自某个用户的浏览器，却不能保证请求本身是用户自愿发出的。

不过这种情况有很多解决方法，特别对于银行这类金融性质的站点，用户的任何敏感操作都需要确认，并且敏感信息的 Cookie 只能拥有较短的生命周期。

同时Cookie有容量和数量的限制，每次都要发送很多信息带来额外的流量消耗、复杂的行为Cookie无法满足要求。
![[Pasted image 20240920123450.png]]
![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

特别注意：以上存在的问题只是Cookie被用于实现交互状态时存在的问题，但并不是说Cookie本身的问题。

试想一下：菜刀可以用来做菜，也可以被用来从事某些暴力行为，你能说菜刀应该被废除吗？

## 3. Session方案

### 3.1 Session机制的概念

如果说Cookie是客户端行为，那么Session就是服务端行为。
![[Pasted image 20240920123458.png]]
![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

Cookie机制在最初和服务端完成交互后，保持状态所需的信息都将存储在客户端，后续直接读取发送给服务端进行交互。

Session代表服务器与浏览器的一次会话过程，并且完全由服务端掌控，实现分配ID、会话信息存储、会话检索等功能。

Session机制将用户的所有活动信息、上下文信息、登录信息等都存储在服务端，只是生成一个唯一标识ID发送给客户端，后续的交互将没有重复的用户信息传输，取而代之的是唯一标识ID，暂且称之为Session-ID吧。

### 3.2 简单的交互流程

- 当客户端第一次请求session对象时候，服务器会为客户端创建一个session，并将通过特殊算法算出一个session的ID，用来标识该session对象。
    
- 当浏览器下次请求别的资源的时候，浏览器会将sessionID放置到请求头中，服务器接收到请求后解析得到sessionID，服务器找到该id的session来确定请求方的身份和一些上下文信息。
    

### 3.3 Session的实现方式

首先明确一点，Session和Cookie没有直接的关系，可以认为Cookie只是实现Session机制的一种方法途径而已，没有Cookie还可以用别的方法。

> Session和Cookie的关系就像加班和加班费的关系，看似关系很密切，实际上没啥关系。

session的实现主要两种方式：cookie与url重写，而cookie是首选方式，因为各种现代浏览器都默认开通cookie功能，但是每种浏览器也都有允许cookie失效的设置，因此对于Session机制来说还需要一个备胎。
![[Pasted image 20240920123508.png]]
![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

将会话标识号以参数形式附加在超链接的URL地址后面的技术称为URL重写。

`原始的URL：   http://taobao.com/getitem?name=baymax&action=buy   重写后的URL:   http://taobao.com/getitem?sessionid=1wui87htentg&?name=baymax&action=buy   `

### 3.4 存在的问题
![[Pasted image 20240920123516.png]]
![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

由于Session信息是存储在服务端的，因此如果用户量很大的场景，Session信息占用的空间就不容忽视。

对于大型网站必然是集群化&分布式的服务器配置，如果Session信息是存储在本地的，那么由于负载均衡的作用，原来请求机器A并且存储了Session信息，下一次请求可能到了机器B，此时机器B上并没有Session信息。

这种情况下要么在B机器重复创建造成浪费，要么引入高可用的Session集群方案，引入Session代理实现信息共享，要么实现定制化哈希到集群A，这样做其实就有些复杂了。
![[Pasted image 20240920123523.png]]
![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

## 4. Token方案

Token是令牌的意思，由服务端生成并发放给客户端，是一种具有时效性的验证身份的手段。

Token避免了Session机制带来的海量信息存储问题，也避免了Cookie机制的一些安全性问题，在现代移动互联网场景、跨域访问等场景有广泛的用途。

### 4.1 简单的交互流程
![[Pasted image 20240920123535.png]]
![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

- 客户端将用户的账号和密码提交给服务器
    
- 服务器对其进行校验，通过则生成一个token值返回给客户端，作为后续的请求交互身份令牌
    
- 客户端拿到服务端返回的token值后，可将其保存在本地，以后每次请求服务器时都携带该token，提交给服务器进行身份校验
    
- 服务器接收到请求后，解析关键信息，再根据相同的加密算法、密钥、用户参数生成sign与客户端的sign进行对比，一致则通过，否则拒绝服务
    
- 验证通过之后，服务端就可以根据该Token中的uid获取对应的用户信息，进行业务请求的响应
    

### 4.2 Token的设计思想

以JSON Web Token（JWT）为例，Token主要由3部分组成：

- Header头部信息  
    记录了使用的加密算法信息
    
- Payload 净荷信息  
    记录了用户信息和过期时间等
    
- Signature 签名信息  
    根据header中的加密算法和payload中的用户信息以及密钥key来生成，是服务端验证服务端的重要依据
    
![[Pasted image 20240920123546.png]]
![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

header和payload的信息不做加密，只做一般的base64编码，服务端收到token后剥离出header和payload获取算法、用户、过期时间等信息，然后根据自己的加密密钥来生成sign，并与客户端传来的sign进行一致性对比，来确定客户端的身份合法性。

这样就实现了用CPU加解密的时间换取存储空间，同时服务端密钥的重要性就显而易见，一旦泄露整个机制就崩塌了，这个时候就需要考虑HTTPS了。

### 4.3 Token方案的特点

- Token可以跨站共享，实现单点登录
    
- Token机制无需太多存储空间，Token包含了用户的信息，只需在客户端存储状态信息即可，对于服务端的扩展性很好
    
- Token机制的安全性依赖于服务端加密算法和密钥的安全性
    
- Token机制也不是万金油
    

## 5.总结

Cookie、Session、Token这三者是不同发展阶段的产物，并且各有优缺点，三者也没有明显的对立关系，反而常常结伴出现，这也是容易被混淆的原因。

Cookie侧重于信息的存储，主要是客户端行为，Session和Token侧重于身份验证，主要是服务端行为。

三者方案在很多场景都还有生命力，了解场景才能选择合适的方案，没有银弹。

- EOF -

推荐阅读  点击标题可跳转

1、[清华大学：2021 元宇宙研究报告！](http://mp.weixin.qq.com/s?__biz=MzAxODI5ODMwOA==&mid=2666559203&idx=2&sn=95fabc48d7c7b77ac3e0aaf1f033d3d1&chksm=80dcb248b7ab3b5ed54dbe4ab601a1a0ca540088f8b22f99e231cf1d6f824391a51c174ef622&scene=21#wechat_redirect)

2、[10 分钟看懂 Docker 和 K8S](http://mp.weixin.qq.com/s?__biz=MzAxODI5ODMwOA==&mid=2666558962&idx=1&sn=ede9eea879107df21557e1b9dcfeeb28&chksm=80dcb159b7ab384f22e68fc32d5556afd3e98c29d08757a871582d526bf5985b0c7c3fd3ec6d&scene=21#wechat_redirect)

3、[为什么腾讯/阿里不去开发被卡脖子的工业软件？](http://mp.weixin.qq.com/s?__biz=MzAxODI5ODMwOA==&mid=2666559138&idx=1&sn=fa0dca517b897bce61ea77194d4292e9&chksm=80dcb209b7ab3b1fc8e5cdb88c85f4d59f9867edbd45a1183116af87c86b8604341765eb9272&scene=21#wechat_redirect)

  

看完本文有收获？请分享给更多人  

推荐关注「Linux 爱好者」，提升Linux技能

![](http://mmbiz.qpic.cn/mmbiz_png/9aPYe0E1fb3sjicd8JxDra10FRIqT54Zke2sfhibTDdtdnVhv5Qh3wLHZmKPjiaD7piahMAzIH6Cnltd1Nco17Ihjw/300?wx_fmt=png&wxfrom=19)

**Linux爱好者**

点击获取《每天一个Linux命令》系列和精选Linux技术资源。「Linux爱好者」日常分享 Linux/Unix 相关内容，包括：工具资源、使用技巧、课程书籍等。

75篇原创内容

公众号

点赞和在看就是最大的支持❤️

阅读 2568

​

写留言

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/9aPYe0E1fb3sjicd8JxDra10FRIqT54Zke2sfhibTDdtdnVhv5Qh3wLHZmKPjiaD7piahMAzIH6Cnltd1Nco17Ihjw/300?wx_fmt=png&wxfrom=18)

Linux爱好者

2112

写留言

写留言

**留言**

暂无留言