# 

芋道源码

 _2024年09月07日 16:19_ _上海_

The following article is from 勇哥Java实战 Author 勇哥java实战分享

[

![](http://wx.qlogo.cn/mmhead/Q3auHgzwzM6yW3qKT0UHU1I6pwFEOGiaH3FEwicSx0ODGSRsX6W3XiadQ/0)

**勇哥Java实战**.

Java 基础、高并发解决方案三剑客（缓存、消息队列、分库分表）、实战项目讲解。

](https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247600568&idx=2&sn=e8ff097b6a5ea8eb27e3f3f9ea6a67ee&chksm=fb2a1f0bef6473ced7ecba631f0b751d80acca10ebd2777afd7de2a1a6bbd43cd084acf2ae3d&mpshare=1&scene=24&srcid=0907SE1FXU3ZFxfjSZWcZ8lg&sharer_shareinfo=21be164d78a29b5acee382d9819e420c&sharer_shareinfo_first=21be164d78a29b5acee382d9819e420c&key=daf9bdc5abc4e8d06709521741167f94e86ef70cfab7e1830c1bd64406a947622377a75774406b277685373fdad8ce045fa939bf4c1233a5650750a86717b1033916099900b8fe5c47c52868297e9304c1fe63970a1edf9f01aff63a109b3d95beb6c539f46c4d3d08b9fbc0718d8d951bfe69f2d0b502f51608314a20dbc4d3&ascene=14&uin=MTEwNTU1MjgwMw%3D%3D&devicetype=iMac+MacBookAir10%2C1+OSX+OSX+14.6.1+build(23G93)&version=13080710&nettype=WIFI&lang=en&session_us=gh_f4580c01069c&countrycode=CN&fontScale=100&exportkey=n_ChQIAhIQ3m9DEAIrUX0WDP0kcaIRQhKNAgIE97dBBAEAAAAAAPp3M2cYVKIAAAAOpnltbLcz9gKNyK89dVj0CrHRuyOozlmnPsPXmJfx0MN9TTesxBC9Aj78oFxCzSKlN3eSLAGONYWDLH7PwY%2BKmWAPGWO8NmiEp0xu9%2Fr2G1zN3AWROkR4N4s8ugG6ylRljG4lRAVP2E5dyne%2Fsfjd%2FyMgYKtBizTrTGJb9qA1LNqMQjaAv8cUrKOAp7Vqm%2Bv5j7zL821clcdf0rRov6vJ2z2phPa%2FS8MsTtJeJHMZvmyg8o40PH%2BsYZzvK9JEdQmnWy4CDZ2OmwFNSAJK56Yo1DBzWHTyuac%2B9u%2BJV4BkRtn1zoBSBRaq23BZ8ai33zIBEpKgLE0p&acctmode=0&pass_ticket=dTXvPKkVljgdSvFNLaD0mHcbLOiVzBf4fJU9hkCGbTJGT%2BKSouYnFE2YwiQs54ZZ&wx_header=0#)

> 👉 **这是一个或许对你有用****的社群**
> 
> 🐱 一对一交流/面试小册/简历优化/求职解惑，欢迎加入「[**芋道快速开发平台**](http://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247576728&idx=1&sn=1298645b025eb51d9078e8c3de7b3c17&chksm=fa4bd329cd3c5a3fd63e455cf39507d3611a7b040be1381fd5ecfc0a7ce3867575fda4b7313d&scene=21#wechat_redirect)」知识星球。下面是星球提供的部分资料： 
> 
> - [《项目实战（视频）》](http://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247576728&idx=1&sn=1298645b025eb51d9078e8c3de7b3c17&chksm=fa4bd329cd3c5a3fd63e455cf39507d3611a7b040be1381fd5ecfc0a7ce3867575fda4b7313d&scene=21#wechat_redirect)：从书中学，往事上**“练”**
>     
> - [《互联网高频面试题》](http://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247576728&idx=1&sn=1298645b025eb51d9078e8c3de7b3c17&chksm=fa4bd329cd3c5a3fd63e455cf39507d3611a7b040be1381fd5ecfc0a7ce3867575fda4b7313d&scene=21#wechat_redirect)：面朝简历学习，春暖花开
>     
> - [《架构 x 系统设计》](http://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247576728&idx=1&sn=1298645b025eb51d9078e8c3de7b3c17&chksm=fa4bd329cd3c5a3fd63e455cf39507d3611a7b040be1381fd5ecfc0a7ce3867575fda4b7313d&scene=21#wechat_redirect)：摧枯拉朽，掌控面试高频场景题
>     
> - [《精进 Java 学习指南》](http://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247576728&idx=1&sn=1298645b025eb51d9078e8c3de7b3c17&chksm=fa4bd329cd3c5a3fd63e455cf39507d3611a7b040be1381fd5ecfc0a7ce3867575fda4b7313d&scene=21#wechat_redirect)：系统学习，互联网主流技术栈
>     
> - [《必读 Java 源码专栏》](http://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247576728&idx=1&sn=1298645b025eb51d9078e8c3de7b3c17&chksm=fa4bd329cd3c5a3fd63e455cf39507d3611a7b040be1381fd5ecfc0a7ce3867575fda4b7313d&scene=21#wechat_redirect)：知其然，知其所以然
>     

![Image](https://mmbiz.qpic.cn/mmbiz_gif/JdLkEI9sZfdWPYr0VKaXztEGHacpRyle7tbZkryrsxIpnAfjRt03ibrcloEZqlRPaVKcb0nD2PrYjtovwOAaFlA/640?wx_fmt=gif&tp=wxpic&wxfrom=5&wx_lazy=1)

> 👉**这是一个或许对你有用的开源项目**
> 
> 国产 Star 破 10w+ 的开源项目，前端包括管理后台 + 微信小程序，后端支持单体和微服务架构。
> 
> 功能涵盖 RBAC 权限、SaaS 多租户、数据权限、商城、支付、工作流、大屏报表、微信公众号、CRM 等等功能：
> 
> - Boot 仓库：https://gitee.com/zhijiantianya/ruoyi-vue-pro
>     
> - Cloud 仓库：https://gitee.com/zhijiantianya/yudao-cloud
>     
> - 视频教程：https://doc.iocoder.cn
>     
> 
> 【国内首批】支持 JDK 21 + SpringBoot 3.2.2、JDK 8 + Spring Boot 2.7.18 双版本 

[来源：勇哥java实战分享](https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247487551&idx=1&sn=18f64ba49f3f0f9d8be9d1fdef8857d9&scene=21#wechat_redirect)

- [1 什么是 Mybatis 动态SQL](https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247487551&idx=1&sn=18f64ba49f3f0f9d8be9d1fdef8857d9&chksm=fa496f8ecd3ee698f4954c00efb80fe955ec9198fff3ef4011e331aa37f55a6a17bc8c0335a8&scene=21&token=899450012&lang=zh_CN#wechat_redirect)
    
- [2 人生第一次线上OOM事故](https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247487551&idx=1&sn=18f64ba49f3f0f9d8be9d1fdef8857d9&chksm=fa496f8ecd3ee698f4954c00efb80fe955ec9198fff3ef4011e331aa37f55a6a17bc8c0335a8&scene=21&token=899450012&lang=zh_CN#wechat_redirect)
    
- [3 前后端参数校验](https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247487551&idx=1&sn=18f64ba49f3f0f9d8be9d1fdef8857d9&chksm=fa496f8ecd3ee698f4954c00efb80fe955ec9198fff3ef4011e331aa37f55a6a17bc8c0335a8&scene=21&token=899450012&lang=zh_CN#wechat_redirect)
    
- [4 复用和专用要做平衡](https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247487551&idx=1&sn=18f64ba49f3f0f9d8be9d1fdef8857d9&chksm=fa496f8ecd3ee698f4954c00efb80fe955ec9198fff3ef4011e331aa37f55a6a17bc8c0335a8&scene=21&token=899450012&lang=zh_CN#wechat_redirect)
    
- [5 防御性编程意识](https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247487551&idx=1&sn=18f64ba49f3f0f9d8be9d1fdef8857d9&chksm=fa496f8ecd3ee698f4954c00efb80fe955ec9198fff3ef4011e331aa37f55a6a17bc8c0335a8&scene=21&token=899450012&lang=zh_CN#wechat_redirect)
    
- [6 写到最后](https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247487551&idx=1&sn=18f64ba49f3f0f9d8be9d1fdef8857d9&chksm=fa496f8ecd3ee698f4954c00efb80fe955ec9198fff3ef4011e331aa37f55a6a17bc8c0335a8&scene=21&token=899450012&lang=zh_CN#wechat_redirect)
    

---

![Image](https://mmbiz.qpic.cn/mmbiz_png/JdLkEI9sZffFicLcEP7jAxLZvr81vCSMQmye80CaYeNLUD8tLBIrKbibE8SIecicN1j78tlhA0oSiau1TMgMETrOSw/640?wx_fmt=png&from=appmsg&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

这篇文章，我们聊聊 Mybatis 动态 SQL  ，以及我对于编程技巧的几点思考 ，希望对大家有所启发。

![Image](https://mmbiz.qpic.cn/mmbiz_png/JdLkEI9sZffFicLcEP7jAxLZvr81vCSMQKHBibStwct0veIYw2er62DZBqProEc2NyMNMy1A8Q3ATO0lOSBf8ibSw/640?wx_fmt=png&from=appmsg&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

## [1 什么是 Mybatis 动态SQL](https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247576728&idx=1&sn=1298645b025eb51d9078e8c3de7b3c17&scene=21#wechat_redirect)

如果你使用过 JDBC 或其它类似的框架，你应该能理解根据不同条件拼接 SQL 语句有多痛苦，例如拼接时要确保不能忘记添加必要的空格，还要注意去掉列表最后一个列名的逗号。

Mybatis  借助功能强大 OGNL 表达式，可以根据参数条件，动态生成执行 SQL 。

使用动态 SQL 最常见情景是根据条件包含 where 子句的一部分。

![Image](https://mmbiz.qpic.cn/mmbiz_png/JdLkEI9sZffFicLcEP7jAxLZvr81vCSMQGCGqY9VNE4OglZj1T5Yhj08G88mPrXKIsBQUJqdiaJQABXZVW8HC65w/640?wx_fmt=png&from=appmsg&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

这条语句提供了可选的查询博客文章列表 ,如果不传入 “title”，那么所有处于 “ACTIVE” 状态的 博客都会返回。

如果传入了 “title” 参数，那么就会对 “title” 一列进行模糊查找并返回对应的 BLOG 结果。

如果希望通过 “title” 和 “author” 两个参数都可以搜索，只需要加入另一个条件即可，见下图：

![Image](https://mmbiz.qpic.cn/mmbiz_png/JdLkEI9sZffFicLcEP7jAxLZvr81vCSMQpq8czjIB9sITuK8jynedia6EHcOnmHZmVKiaY1DBo4tcaDVLHX4JoKhw/640?wx_fmt=png&from=appmsg&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

我们也可以使用 where 标签，该标签只会在子元素返回任何内容的情况下才插入 “WHERE” 子句。而且，若子句的开头为 “AND” 或 “OR”，where 标签也会将它们去除。

![Image](https://mmbiz.qpic.cn/mmbiz_png/JdLkEI9sZffFicLcEP7jAxLZvr81vCSMQSGqQZUujn0JKknnSpEiajEC6ENEjsUMN5zl0rphQ23RKxFyvBFH3JFw/640?wx_fmt=png&from=appmsg&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

**Mybatis 还支持 choose (when, otherwise)、trim (where, set)、foreach 等其他的动态标签，这里就不一一赘述了。**

> 基于 Spring Boot + MyBatis Plus + Vue & Element 实现的后台管理系统 + 用户小程序，支持 RBAC 动态权限、多租户、数据权限、工作流、三方登录、支付、短信、商城等功能
> 
> - 项目地址：https://github.com/YunaiV/ruoyi-vue-pro
>     
> - 视频教程：https://doc.iocoder.cn/video/
>     

## [2 人生第一次线上OOM事故](https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247576728&idx=1&sn=1298645b025eb51d9078e8c3de7b3c17&scene=21#wechat_redirect)

我曾服务一家电商公司的用户中心，用户中心提供用户注册，查询，修改等基础功能 。

那个时候 Dubbo 等 RPC 框架并没有开源，所有的服务都以 HTTP 接口形式提供，数据传输格式是 XML 。

因为写接口非常费劲，所以为了**接口复用** ，我写了一个通用接口 getUserByConditions ，该接口支持通过 「用户名」、「昵称」、「手机号」、「用户编号」这三个查询用户基本信息。

使用的是 ibatis （mybatis 的前身）, SQLMap 见下图 。

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

当构建动态 SQL 查询时，条件通常会追加到 `WHERE` 子句后，而以 `WHERE 1 = 1` 开头，可以轻松地使用 `AND` 追加其他条件。

用户中心在上线后，竟然每隔三四个小时就发生了内存溢出问题 ，经过通过和 DBA 沟通，发现高频次出现**全表查用户表** ，执行 SQL 变成 ：

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

查看日志后，发现前端传递的参数出现了**空字符串** ，笔者在代码中并**没有做参数校验** ，所以才出现全表查询 ，当时用户表的数据是 1000多万 ，页面调用几次，用户中心服务就 OOM 了。

最终解决问题的方式很简单，后端在接收参数时，做了参数校验。

事故虽然解决了，但对我的影响一直延续至今。我仍然记得当时站在运维同学旁边，不断的看他调整 JVM 参数 ，重启服务的画面，自己那个时候真的是羞愧难当，心中发誓：**以后绝对不可以发生类似的事故** 。

对于动态 SQL ，我的编程思维也经历了如下三个阶段 ：

- 前后端参数校验
    
- 复用和专用要做平衡
    
- 防御性编程意识
    

> 基于 Spring Cloud Alibaba + Gateway + Nacos + RocketMQ + Vue & Element 实现的后台管理系统 + 用户小程序，支持 RBAC 动态权限、多租户、数据权限、工作流、三方登录、支付、短信、商城等功能
> 
> - 项目地址：https://github.com/YunaiV/yudao-cloud
>     
> - 视频教程：https://doc.iocoder.cn/video/
>     

## [3 前后端参数校验](https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247576728&idx=1&sn=1298645b025eb51d9078e8c3de7b3c17&scene=21#wechat_redirect)

为了提升开发效率，我们人为的将系统分为前端、后端，分别由两拨不同的人员开发 ，经常出现系统问题时，两拨人都非常不服气，相互指责。

要想系统健壮，前后端应该同时做接口参数校验 （**后端必须做参数校验** ），当大家都遵循这个规约时，出现系统问题的风险大大减少。

**1、前端校验**

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

前端校验，主要是为了提高用户体验，例如用户输入一个邮箱地址，要校验这个邮箱地址是否合法，没有必要发送到服务端进行校验，直接在前端用 js 进行校验即可。

但是大家需要明白的是，前端校验无法代替后端校验，前端校验可以有效的提高用户体验，但是无法确保数据完整性，因为在 B/S 架构中，用户可以方便的拿到请求地址，然后直接发送请求，传递非法参数。

**2、后端校验**

**后端必须做参数校验，后端必须做参数校验，后端必须做参数校验** ，重要的事情表达三次。

数据在网络传输过程中有可能被篡改了，或者数据不满足业务需求，假如不做后端参数校验，则有可能导致系统异常，或者业务出现重大事故。

比如在 SpringBoot 项目中，我们可以使用 **hibernate-validator** 进行参数校验 。

POST、PUT 请求一般会使用 requestBody 传递参数，这种情况下，后端使用 DTO 对象进行接收。

只要给 DTO 对象加上 @Validated 注解就能实现自动参数校验。比如，有一个保存 User 的接口，要求 userName 长度是 2-10，account 和 password 字段长度

是 6-20。

在 DTO 字段上声明约束注解：

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

在方法参数上声明校验注解:

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

虽然，我们可以使用接口校验，可以保证动态 SQL 的参数正确，但是假如我们仅仅只是复用 SQLMap （Dao 方法）时，也有可能因为调用方传递参数错误，导致非预期的问题。

当然，我们也可以**使用 Mybatis 拦截器** 从根本上来解决，但是我想这样会加大系统的复杂度。于是，我思考了了另外一点：**复用和专用要做平衡** 。

## [4 复用和专用要做平衡](https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247576728&idx=1&sn=1298645b025eb51d9078e8c3de7b3c17&scene=21#wechat_redirect)

我当时写的那个接口 getUserByConditions ，是支持四种不同参数的查询，同样也是为了省时间，快点出活。

后来，随着我工作经验的日益丰富，我的编程习惯也慢慢发生了改变，对于业务需求明确的场景，我更多的倾向于将通用接口拆分成专用接口。

比如 getUserByConditions 可以拆分成如下四个接口 ，

- 按照用户 ID 查询用户信息
    
- 按照用户昵称查询用户信息
    
- 按照手机号查询用户信息
    
- 按照用户名查询用户信息
    

比如按照用户 ID 查询用户信息 ， SQLMAP 就简化为：

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

通过这样的拆分，我们的接口设计更加细粒度，也更容易维护 , 同时也可以规避 where 1 =1 产生的不确定性（虽然我做了后端校验，依然存在不确定性）。

有的同学会有疑问：假如拆分得太细，会不会增加我编写接口和 SQLMap 的工作量 ？

笔者的思路是：定制自己的代码生成器，将生成的 SQLMap  、Mapper 保证更细的颗粒度。

## [5 防御性编程意识](https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247576728&idx=1&sn=1298645b025eb51d9078e8c3de7b3c17&scene=21#wechat_redirect)

笔者刚入行的时候，只是机械性的完成任务，并没有思考代码后面的资源占用，以及有没有可能产生恶劣的影响。

随着见识更多的系统，学习开源项目，笔者慢慢培养了一种习惯：

- 这段代码会占用多少系统资源
    
- 如何规避风险 ，做好预防性编程。
    

其实，这和玩游戏差不多 ，在玩游戏的时，我们经常说一个词，那就是**意识。**

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

上图，后裔跟墨子在压对面马可蔡文姬，看到小地图中路铠跟小乔的视野，方向是往下路来的，这时候我们就得到了一个信息。

知道对面的人要来抓，或者是协防，这种情况我们只有两个人，其他的队友都不在，只能选择避战，强打只会损失两名“大将”。

通过小地图的信息，并且想出应对方法，就是叫做“猜测意识”。

编程也是一样的，我们思考代码可能产生的系统资源占用，以及可能存在的风险，并做好防御性编程，就是**编程的意识** 。

## [6 写到最后](https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247576728&idx=1&sn=1298645b025eb51d9078e8c3de7b3c17&scene=21#wechat_redirect)

人生第一次线上 OOM 事故，因我在使用 Mybatis 动态 SQL 时，没有做后端校验而出现，造成了比较坏的影响。

在后面的职业生涯里面，为了规避生产环境的事故，我试着打磨自己的编程思维，比如**做好后端校验** 、**平衡好复用和专用接口** 、**培养防御性编程的意识** 。

絮絮叨叨这么多，大家可能觉得我小题大做了，但事实是，类似我这样的事故层出不穷，上周我又目睹了一起。

IT 世界有了那么多框架，好像我们依然写不好代码，我有点沮丧。

---

欢迎加入我的知识星球，全面提升技术能力。

👉 加入方式，**“**长按**”或“**扫描**”下方二维码噢**：

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

星球的**内容包括**：项目实战、面试招聘、源码解析、学习路线。

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**文章有帮助的话，在看，转发吧。**

**谢谢支持哟 (*^__^*）**

Read more

Reads 1620

​

Comment

**留言 1**

- 丫丫的猫
    
    上海2小时前
    
    Like2
    
    Github 最火开源项目！ ① Spring Boot 多模块：https://github.com/YunaiV/ruoyi-vue-pro ② Spring Cloud 微服务：https://github.com/YunaiV/yudao-cloud
    
    PinnedAuthor liked
    

已无更多数据

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/JdLkEI9sZfcTJicC5sSiaD2Y4XgGFI1cFr8JBcIWEoS8ygoNIUlSaD8DS6CsgiaB0RpejXp5Biavia2vicHdIIqaLbTg/300?wx_fmt=png&wxfrom=18)

芋道源码

6322

1

Comment

**留言 1**

- 丫丫的猫
    
    上海2小时前
    
    Like2
    
    Github 最火开源项目！ ① Spring Boot 多模块：https://github.com/YunaiV/ruoyi-vue-pro ② Spring Cloud 微服务：https://github.com/YunaiV/yudao-cloud
    
    PinnedAuthor liked
    

已无更多数据