# 

原创 朱欣欣 高可用架构

_2021年09月15日 10:12_

> 作者朱欣欣，开源爱好者，Apache APISIX committer。
>
> 个人 Github 主页：https://github.com/starsz

身份认证在日常生活当中是非常常见的一项功能，大家平时基本都会接触到。比如用支付宝消费时的人脸识别确认、公司上班下班时的指纹/面部打卡以及网站上进行账号密码登录操作等，其实都是身份认证的场景体现。

![图片](https://mmbiz.qpic.cn/mmbiz_png/8XkvNnTiapOPcwtMUBNGFYZKArGKFvX50jBzAP7opa9MKia9KDfNr7TG1fibb34icbicTR5Hx3yxvU8M67sShJQNrwg/640?wx_fmt=png&wxfrom=13&tp=wxpic)

如上图，Jack 通过账号密码请求服务端应用，服务端应用中需要有一个专门用做身份认证的模块来处理这部分的逻辑。请求处理完毕子后，如果使用 JWT Token 认证方式，服务器会反馈一个 Token 去标识这个用户为 Jack。如果登录过程中账号密码输入错误，就会导致身份认证失败。这个场景大家肯定都非常熟悉，这里就不做展开举例了。

**网络身份认证的意义在哪里？**

**安全性**

身份认证确保了后端服务的安全性，避免了未经授权的访问，从而确保数据的安全性。比如防止某些黑客攻击，以及一些恶意调用，这些都可以通过身份认证进行阻拦。

**实用性**

从实用性的角度考虑，它可以更方便地记录操作者或调用方。比如上班打卡其实就是通过记录「谁进行了这项操作」来统计员工上班信息。其次它可以记录访问频率及访问频次。例如记录某用户在最近几分钟中发起请求的频率和频次，可以判断这个用户活跃程度，也可以判断是否为恶意攻击等。

**功能性**

在功能层面，它通过识别身份可以对不同的身份进行不同权限的操作处理。比如在公司里，你的身份权限无法使用某些页面或服务。再细致一点，对不同身份或者不同的 API 接口调用方做一些相关技术上的限制策略，如限流限速等，以此来达到根据不同的用户（调用方）采取不同的功能限制。

**使用 Apache APISIX 进行集中式身份认证优点**

**从传统到新模式**

如下图所示，左侧的图为我们展示了一种比较常见的传统身份认证方法。每一个应用服务模块去开发一个单独的身份认证模块，用来支持身份认证的一套流程处理。但当服务量多了之后，就会发现这些模块的开发工作量都是非常巨大且重复的。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

这种时候，我们可以通过把这部分的开发逻辑放置到 Apache APISIX 的网关来实现统一，减少开发量。

图右所示，用户或应用方直接去请求 Apache APISIX，然后 Apache APISIX 通过识别并认证通过后，会将鉴别的身份信息传递到上游应用服务。之后上游应用服务就可以从请求头中读到这部分信息，然后进行后续相关工作的处理。讲完了大概流程，我们来详细罗列下。

**优点一：配置收敛，统一管理**

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

如上图是一张 APISIX-Dashboard 的界面截图，可以看到路由、Consumer 等数据信息。这里的 Consumer 可以理解为一个应用方，比如可以为应用专门去创建一个 Consumer 并配置相关的认证插件，例如 JWT、Basic-Auth 等插件。当有新的服务出现时，我们需要再创建一个 Consumer，然后将这部分配置信息给配置到第二个应用服务上。

通过集中式身份认证，可以将客户配置进行收敛并统一管理，达到降低运维成本的效果。

**优点二：插件丰富，减少开发**

Apache APISIX 作为一个 API 网关，目前已开启与各种插件功能的适配合作，插件库也比较丰富。目前已经可与大量身份认证相关的插件进行搭配处理，如下图所示。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

基础认证插件比如 Key-Auth、Basic-Auth，他们是通过账号密码的方式进行认证。复杂一些的认证插件如 Hmac-Auth、JWT-Auth。如 Hmac-Auth 通过对请求信息做一些加密，生成一个签名。当 API 调用方将这个签名携带到 Apache APISIX 的网关 Apache APISIX 会以相同的算法计算签名，只有当签名方和应用调用方认证相同时才予以通过。

Authz-casbin 插件是目前 Apche APISIX 与 Casbin 社区正在进行合作开发的一个项目，它的应用原理是按照 Casbin 的规则，去处理一些基于角色的权限管控 (RBAC)，进行 ACL 相关操作。

其他则是一些通用认证协议和联合第三方组件进行合作的认证协议，例如 OpenID-Connect 身份认证机制，以及 LDAP 认证等。

**优点三：配置灵活，功能强大**

功能强大该如何理解，就是 Apache APISIX 中可以针对每一个 Consumer （即调用方应用）去做不同级别的插件配置。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

如上图，我们创建了两个消费者 Consumer A，Consumer B。我们将 Consumer A 应用到应用1，则后续应用1的访问将会开启 Consumer A 的这部分插件，例如 IP 黑白名单，限制并发数量等。将 Consumer B 应用到应用2 ，由于开启了 http-log 插件，则应用2的访问日志将会通过 HTTP 的方式发送到日志系统进行收集。

**Apache APISIX 中身份认证的玩法**

关于 Apache APISIX 身份认证的玩法，这里为大家提供三个阶段的玩法推荐，大家仅作参考，也可以在这些基础上，进行更深度的使用和应用。

**初级：普通玩法**

普通玩法推荐大家基于 Key-Auth、Basic-Auth、JWT- Auth 和 Hmac-Auth 进行，这些插件的使用需要与 Consumer 进行关联使用。

**步骤一：创建路由，配置身份认证插件**

创建路由，配置上游为 httpbin.org，同时开启 basic-auth 插件。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**步骤二：创建消费者，配置身份信息**

创建消费者 foo。在消费者中，我们需要配置用户的认证信息，例如 username 为 foo，password 为 bar。

**步骤三：应用服务携带认证信息进行访问**

应用携带 username:foo , password: bar 访问 Apache APISIX。Apache APISIX 通过认证，并将请求 Authorization 请求头携带至上游 httpbin.org 。由于 httpbin.org 中 get 接口会将请求信息返回，因此我们可以在其中观察到请求头 Authorization。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**中级：进阶玩法**

进阶模式下，是使用 Apache APISIX 与 OpenID-Connect 插件进行对接第三方认证服务。OpenID-Connect 是一种认证机制，可以采用该认证机制对接用户的用户管理系统或者用户管理服务，例如国内的 Authing 和腾讯云，国外的 Okta 和 Auth0 等。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

具体操作如下，这里使用 Okta 云服务举例：

**步骤一：创建 OpenID-Connect 应用**

在 Okta 控制台页面创建一个支持 OpenID-Connect 的应用。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**步骤二：创建路由，配置 OpenID-Connect 插件**

创建路由，配置访问的上游地址为 httpbin.org，同时开启 openid-connect 插件，将 Okta 应用的配置填写到 openid-connect 插件中。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**步骤三：用户访问时，会跳转至登录页面。登录完成后，上游应用获取用户信息**

此时，当用户访问 Apache APISIX 时，会先重定向到 Okta 登录页面。此时需要在该页面填写 Okta 中已经存在的用户的账号密码。登陆完成之后，Apache APISIX 会将本次认证的 Access-Token，ID-Token 携带至上游，同时将本次认证的用户信息以 base64 的方式编码至 Userinfo 请求头，传递至上游。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**高级：DIY 玩法**

这里提供的 DIY 玩法是利用了 Serverless 插件，这款插件功能丰富，玩法也非常多。大家如果有自己的使用玩法，也欢迎在社区内进行交流。

具体操作流程就是将自己的代码片段通过 Serverless 插件上传到 Apache APISIX，这个过程中 Serverless 插件支持动态配置和热更新。

**方式一：自定义判断逻辑**

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

通过请求头、参数等相关信息，来进行一些判断。通过这种做法，我们可以去实现自己的一些规则，比如说结合企业内部的一些认证，或者去写一些自己的业务逻辑。

**方式二：发起认证鉴权请求**

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

通过 HTTP 库发起一个 HTTP 请求，我们可以利用第三方服务做一些认证、鉴权相关事情，然后解析返回结果。通过这种方式，我们可以做的事情就可以扩展很多。比如对接 Redis 或数据库，只要是通过TCP 或 UDP 这种形式的，都可以通过 Serverless 插件进行尝试。

**配置上传**

完成自定义代码片之后，我们创建路由，并将代码片配置到 Serverless 插件。后面再通过访问 Apache APISIX 获取相应的信息反馈，测试插件是否生效。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**参考阅读：**

- [爱奇艺本地实时Cache方案](http://mp.weixin.qq.com/s?__biz=MzAwMDU1MTE1OQ==&mid=2653557084&idx=1&sn=b41806694b7f1d19c7bd075e5b3bdccd&chksm=813980c4b64e09d2e233282c2eff54f9779a17d9193352f7d9603e47f8a3f47b33131478d4a7&scene=21#wechat_redirect)

- [周末思考：浅谈如何成为技术一号位？](http://mp.weixin.qq.com/s?__biz=MzAwMDU1MTE1OQ==&mid=2653557029&idx=1&sn=40bc23df3d547948a00e11c8a59d04e2&chksm=813980bdb64e09ab336d38b04595f0218ae172703dc6359ff34b573d6817a012c41ea7833aba&scene=21#wechat_redirect)

- [go语言最全优化技巧总结，值得收藏！](http://mp.weixin.qq.com/s?__biz=MzAwMDU1MTE1OQ==&mid=2653556997&idx=1&sn=31b1e815ca1b3c6f5ae7d80e718e36e2&chksm=8139809db64e098b3d9d7d45c4b28d7d1ffb57fe1a0e735d95450e22e3fd411730e37b4c9799&scene=21#wechat_redirect)

- [如何支持亿级用户分流实验？AB实验平台在爱奇艺的实践](http://mp.weixin.qq.com/s?__biz=MzAwMDU1MTE1OQ==&mid=2653556999&idx=1&sn=767af0e29acedecffdc60028f8eeca98&chksm=8139809fb64e098995903714e3cc08454d77905eb64fc2f89011d3e8d5a6c50a09663f58ab0e&scene=21#wechat_redirect)

- [从源码角度分析 Mybatis 工作原理](http://mp.weixin.qq.com/s?__biz=MzAwMDU1MTE1OQ==&mid=2653556964&idx=1&sn=8cfc964017c0a5f3d56a24016bb9af50&chksm=8139817cb64e086aa7c9f3cf0e3ad3d51cc337848a3c80cfd79800df873a2dbe0eec0c149c96&scene=21#wechat_redirect)

技术原创及架构实践文章，欢迎通过公众号菜单「联系我们」进行投稿。

**高可用架构**

**改变互联网的构建方式**

![](http://mmbiz.qpic.cn/mmbiz_png/8XkvNnTiapONl06YmHad4csRU93kcbJ76JIWzEAmOSVooibFHHkzfWzzkc7dpU4H06Wp9F6Z687vIghdawxvl47A/300?wx_fmt=png&wxfrom=19)

**高可用架构**

高可用架构公众号。

439篇原创内容

公众号

阅读 3312

​

写留言

**留言 2**

- 千罹

  2021年9月15日

  赞

  666 API6

- 杨\*\*

  2021年9月15日

  赞

  跟cas有啥区别

已无更多数据

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/8XkvNnTiapONl06YmHad4csRU93kcbJ76JIWzEAmOSVooibFHHkzfWzzkc7dpU4H06Wp9F6Z687vIghdawxvl47A/300?wx_fmt=png&wxfrom=18)

高可用架构

12126

2

写留言

**留言 2**

- 千罹

  2021年9月15日

  赞

  666 API6

- 杨\*\*

  2021年9月15日

  赞

  跟cas有啥区别

已无更多数据
