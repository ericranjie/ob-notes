# 

原创 骨哥说事 骨哥说事

 _2021年11月26日 14:43_

|   |
|---|
|# ****声明：****文章中涉及的程序(方法)可能带有攻击性，仅供安全研究与教学之用，读者将其信息做其他用途，由用户承担全部法律及连带责任，文章作者不承担任何法律及连带责任。|

# 

**本篇文章共4918字，预计阅读时间15分钟。**

# **背景介绍：**

  

今天的文章来自国外一位女黑客Vickie Li 分享的API安全经验分享，这位女黑客主要从事攻防研究，在推特上有着2万+的粉丝。她的推特是：https://twitter.com/vickieli7

  

  

**API安全的TOP 10漏洞：**

首先来看看OWASP对API安全的TOP 10漏洞定义。

  

- Broken Object-Level Authorization
    
- Broken User Authentication
    
- Excessive Data Exposure
    
- Lack of Resources & Rate Limiting
    
- Broken Function-Level Authorization
    
- Mass Assignment
    
- Security Misconfiguration
    
- Injection
    
- Improper Assets Management
    
- and Insufficient Logging & Monitoring
    

  

那么接下来就将从API TOP的第一位开始，Broked Object-Level Authorization（破坏对象级别授权），了解这些漏洞是如何发生的，我们应该如何识别以及预防这些漏洞的发生。

  

---

  

API #1: Broken Object-Level Authorization（破坏对象级别授权）  

  

API 通常公开用于访问资源的对象标识符，当在这些端点上没有正确实施访问控制时，攻击者可以查看或操作他们不应该有权访问的资源，该漏洞影响所有类型的API接口架构，包括SOAP、REST和GraphQL。

  

首先来看一个例子，假设API允许用户根据其用户ID检索有关其支付方式的详细信息：

  

https://api.example.com/v1.1/users/payment/show?user_id=12

  

如果应用程序不需要在 API 调用中提供额外的身份证明，而只是将请求的数据返回给任何人，那么应用程序就会向攻击者公开敏感信息，攻击者可以猜测、泄露或暴力破解受害者的 ID，甚至通过 API 接口窃取他们的支付信息。

  

通常，应用程序允许用户根据对象 ID 而不是他们的用户 ID 来访问数据对象，比如Example.com 有一处 API 接口，允许用户使用数字ID检索其私人消息的内容：

  

https://api.example.com/v1.1/messages/show?id=12345

  

如果服务器没有在此接口上进行相应的访问控制，攻击者就可以强行使用这些数字ID检索其他用户的私人消息！

  

**此类漏洞的影响：**  

  

该漏洞的影响取决于公开的数据对象，如果用户的PII或凭证等关键对象被公开暴露，该漏洞可能会导致数据泄露或应用程序受损。

  

此漏洞甚至可能被攻击者编写脚本从而自动查询所有用户ID并抓取数据，如果此漏洞发生在在线购物网站，攻击者可能会窃取数百万个银行帐户、信用卡号和住址。在银行网站上，则可能会导致用户的信用信息和纳税信息泄露。

  

如果在密码重置、密码修改和帐户恢复等关键功能上存在此类漏洞，攻击者则可以利用这些漏洞接管用户或管理员帐户。

  

**如何防止此类漏洞？**

防止此类漏洞比较理想的方法是使用访问令牌或其他加密形式对API用户的ID进行保护，然后，对所有需要某些权限的敏感API接口实施访问控制。另外请记住，每个API接口和请求方法都应该被审计和适当保护。例如，我们经常会看到一些API接口的实现，**只需更改为不同的请求方法即可轻松绕过。**

  

如果以上无法做到，请尽量使用随机且不可预测的数值作为数据对象引用，而不是简单的数字ID，然而，简单地使用随机ID并不能被认为是全面的保护，因为ID可能会被泄露或被盗，假设下面这个API不受访问控制的限制：

  

https://api.example.com/v1.1/messages/show?id=d0c240ea139206019f692d

  

尽管ID很难猜测，但攻击者可能会窃取或泄露这些ID，另外，默认情况下，存储在URL中的ID也可用于浏览器扩展和浏览历史。

  

API #2: Broken User Authentication（破坏用户验证）  

  

API 很难进行身份验证，通常，在 API 调用期间提示输入用户凭据或使用多重身份验证是不现实的。

  

因此，API 系统中的身份验证通常使用嵌入到各个 API 调用中的访问令牌（access token）来实现对用户的身份验证，如果身份验证没有正确配置，攻击者可以利用这些错误配置伪装成其他用户。

**无身份验证的API**  

  

首先，API可能完全缺少身份验证机制，有时，API开发人员假设API接口只能由授权的应用程序访问，不会被其他任何人发现，它们允许任何人使用该API进行查询，在这种情况下，任何人只要能弄清楚其查询结构，就可以通过API自由请求数据或执行操作。

  

**身份验证配置错误**

值得庆幸的是，缺少身份验证的API正变的越来越不常见，在大多数情况下，破坏用户身份验证是由错误的访问令牌（access token）设计或实现引起的。

  

一个常见的错误是没有正确生成访问令牌，首先，如果token短、简单或可预测，攻击者就可能暴力破解它们，当使用弱加密或散列算法生成熵不足的token或从用户信息导出令牌时，可能会发生这种情况。来看看下面这个API token会有什么问题？

  

access_token=dmlja2llbGk=

  

是的，令牌只是用户名“vickieli”的base64编码。

  

就算不使用简单的token字符串的API也可能是不安全的，例如，使用JSON Web Token(JWT)也可能会出现签名不正确或丢失签名的问题，**如果对管理员或其他具有特殊权限的用户使用不安全令牌进行身份验证，则更为危险。**

  

**Token时效久**

即使Token生成正确，不正确的Token失效也可能会导致安全问题，在许多API实现中，长时效Token是一个巨大的安全问题。

  

API令牌应定期过期，并且应在注销、密码修改、帐户恢复和帐户删除等敏感操作之后过期，**如果访问令牌没有失效，攻击者可以在窃取令牌后无限期地保持对系统的访问。**

  

**Token泄漏**

有时，开发人员会不安全地传输访问令牌（access token），例如在URL或通过未加密的传输流量，如果令牌是通过URL传输的，则任何有权使用浏览器扩展或浏览器历史记录的人窃取访问令牌。

  

https://api.example.com/v1.1/users/payment/show?user_id=12&access_token=360f91d065e56a15a0d9a0b4e170967b  

  

当然，对于未加密通信传输的API令牌，攻击者也可以利用中间人(MITM)攻击对受害者的通信传输进行拦截。

  

**如何防止身份验证破坏？**

  

首先，需要确保对所有敏感数据和功能实施访问控制，其次，确保API的访问令牌长度足够“长”、且为随机和不可预测，如果使用从用户信息生成的令牌，请尽量使用强算法和密钥来确保用户无法伪造自己的令牌，最后，如果API使用基于签名的身份验证(如JWT)，请在令牌上实现强签名并正确验证它。

  

定期在注销、密码重置、帐户恢复和帐户删除等操作之后使令牌无效，应将访问令牌（access token）视为机密，永远不要在URL或未加密的流量中传输它们。

  

API #3: Excessive Data Exposure（过度的数据暴露）  

  

什么是过度数据暴露？就是当应用程序通过API接口获得响应时，服务器向用户透露了除必要信息外的其它更多信息。

  

让我们来看一个简单的 API 用例，Web 应用程序使用 API 服务检索信息，然后使用这些信息填充网页以展示给用户。

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

对于许多API服务，API客户端应用程序不能选择在API调用中返回哪些数据字段，假设应用程序从API检索用户信息以显示用户配置文件，那么检索用户信息的API调用如下所示：

  

https://api.example.com/v1.1/users/show?user_id=12

  

服务器会对相应的用户对象进行响应：  

```
{
```

  

注意！除了有关用户的基本信息外，此API调用还返回了该用户的API Token、电话号码和住址。

  

一些开发者认为，如果他们不在网页上显示敏感信息，用户就看不到它们，正因为如此，他们在不过滤敏感信息的情况下，将整个响应信息发送给了用户浏览器，并依靠前端代码过滤私人信息，当这种情况发生时，任何访问个人资料页面的人都将能够拦截此响应，并读取用户的敏感信息！

  

**如何防止数据过度暴露？**

  

在将数据返回给用户之前没有对其获得的结果进行过滤操作时，就会发生数据过度暴露。当API发送敏感数据时，客户端应用程序应该在将数据转发给用户之前对其进行过滤，确定用户应该知道什么，理想情况下，呈现网页所需的最小数据量。

  

如果API允许，可以从API服务器请求所需的最小数据量，例如GraphQL API允许指定API请求中所需的确切字段，最后，同样要避免使用未加密的流量传输敏感信息。

  

## **如何寻找此类漏洞？**

  

这位女黑客在挖掘漏洞时，经常会挖掘此类漏洞。作为一名漏洞赏金猎人和渗透测试人员，她养成了对每个服务器响应搜索“key”、“token”和“secret”等关键字的习惯。

  

这些漏洞通常都是上述问题造成的：服务器过于宽松，从服务器返回整个 API 响应，而不是在转发给用户之前进行过滤。

  

不幸的是过度的数据暴露漏洞非常普遍，当下面的**API #4漏洞**配合使用时，可能会成为一个更大的安全问题。

  

API #4: Lack of Resources & Rate Limiting（资源不足与速率限制）

  

当API不限制客户端的请求数量或频率时，就会出现资源不足和速率限制问题。这将导致客户端每秒进行数千次甚至更多次的API调用，或者一次请求成百上千条数据记录。

  

听起来貌似没啥问题，对吧？在多数情况下，资源不足和速率限制不算什么问题，但有时，攻击者可以利用这个漏洞做更多的事。

  

**什么时候会成为问题？**

  

首先，缺少速率限制可能会影响API服务器的性能，并允许攻击者发起DoS攻击，当单个客户端或多个客户端的请求过多时，它们可能会使服务器处理请求的能力不堪重负，进而使服务变慢或对其他用户不可用。

  

另一个问题是，缺少速率限制可能会导致对身份验证和破坏对象级别授权进行暴力攻击。例如，如果对用户提交登录请求的次数没有限制，则恶意攻击者可以通过尝试使用不同的密码登录，直到成功为止，从而暴力破解用户密码；另一方面，如果应用程序遭受破坏对象级别授权，攻击者可以使用暴力手段枚举敏感数据ID。

  

最后，如果API泄露信息，那么缺少速率限制可以帮助攻击者更快地渗透出敏感数据。例如，假设某API接口用于检索用户电子邮件，并且不受任何形式的访问限制，然后返回20封电子邮件的内容：

  

https://api.example.com/v1.1/emails/view?user_id=123&entries=20

  

通过修改entries参数后的数量，可以获得5000封电子邮件内容：

  

https://api.example.com/v1.1/emails/view?user_id=123&entries=5000  

  

同样，当API调用返回用户的邮箱地址且不受速率限制时：

  

https://api.example.com/v1.1/profile/email/view?user_id=123  

  

攻击者则可以通过发送大量API请求获取用户电子邮件地址：

  

```
https://api.example.com/v1.1/profile/email/view?user_id=123
```

  

**如何防止资源不足和速率限制问题？**

  

简单的说，就是限制用户对资源的访问！但是每种功能适当速率和资源需求会有所不同，例如，身份验证端点的速率限制应该低一些，以防止暴力密码猜测攻击。首先要做的是确定这项功能的“正常使用”范围是什么，然后，阻挡那些比正常范围“高得多”的请求。

  

# **API #5：Broken Function-Level Authorization（功能级授权破坏）**

功能级授权破坏是指应用程序未能将敏感功能限制给授权用户，与破坏对象级授权不同，此类漏洞特指未经授权的用户何时可以访问他们不应访问的敏感或受限功能。

  

例如，当一个用户可以修改另一个用户的帐户时，或者当普通用户可以访问站点上的管理功能时，这些安全问题都是由于缺少或错误配置的访问控制造成的，让我们来看几个例子：

  

## **示例#1：删除他人帖子**

假设一个 API 允许其用户通过发送一个 GET 请求来检索博客文章，如下所示：

  

```
GET /api/v1.1/user/12358/posts?id=32
```

  

该请求将返回用户12358 的帖子32，由于此平台上的所有帖子都是公开的，因此任何用户都可以提交该请求以访问他人的帖子，但是，由于只有用户本人可以修改这个帖子，因此只有用户 12358 可以提交 POST 请求来修改或编辑帖子。

  

如果 API 未对使用 PUT 、DELETE 的 HTTP 方法请求进行相应限制的话会怎样？在这种情况下，恶意用户可能会使用不同的 HTTP 方法修改或删除其他用户的发帖：

  

```
DELETE /api/v1.1/user/12358/posts?id=32
```

  

## **示例#2：伪装管理员**

假设该网站还允许平台管理员修改或删除任何人的帖子，因此，如果从管理员帐户发送下面这些请求都会成功：

  

```
DELETE /api/v1.1/user/12358/posts?id=32
```

  

但是该站点通过请求中的特殊标记头（header）确定是否为管理员：

  

```
Admin: 1
```

  

在这种情况下，任何恶意用户都可以简单地将此标头添加到他们的请求中并获得对这个特殊管理功能的访问权！

  

## **示例 3：门上没有锁**

  

最后，假设该站点允许管理员通过特殊的 API 请求查看站点的统计信息：

  

```
GET /api/v1.1/site/stats/hd216zla
```

  

此请求未对任何用户的进行限制，主要依赖于 URL 末尾的那一串随机字符串，以防止未经授权的用户访问它，这种做法被称为“默默无闻的安全”，通俗理解就是通过不让外人得知来增加安全性。

  

但是通过“默默无闻的安全”作为唯一的安全机制并不可靠，如果攻击者可以通过信息泄露找到隐藏的 URL，攻击者就可以访问这些敏感功能。

  

## **攻击者可以用来做什么？**

  

这主要取决于攻击者可以通过漏洞访问的功能，比如攻击者可能冒充其他用户、访问受限数据、修改他人帐户，甚至成为网站的管理员。

  

防止功能级授权被破坏的关键是根据用户的会话实施细粒度、严格的访问控制，并确保无论请求的方法、标头和 URL 参数如何，都始终如一地实施这种访问控制。

  

**未完待续......**

****====正文结束====****  

![](http://mmbiz.qpic.cn/mmbiz_png/hZj512NN8jmgmXq99BwSVFRkYfQUE1MO941mTf1fo026kyjDI8O0cYoyTSJz5Xe6wUePb2ApCzOsnP94QuZrNQ/300?wx_fmt=png&wxfrom=19)

**骨哥说事**

一个喜爱鼓捣的技术宅

381篇原创内容

公众号

![](https://mmbiz.qlogo.cn/mmbiz_jpg/7Fww5Fj1icoTwR2cOmuT0ewV8Pp8m1mUNQicCHy5QrahqWyXvxboko1g3RVS3eOx1hpv0vt9VnibjW6LUibeYGjGicw/0?wx_fmt=jpeg)

骨哥说事

 赞赏 

![赞赏二维码](https://mp.weixin.qq.com/s?__biz=MjM5Mzc4MzUzMQ==&mid=2650254953&idx=1&sn=5629c94b3384ec07bddade1c110c5ea2&chksm=be92c5ed89e54cfb63ce0d1a18bbca9e7c81fb1afc3b8f3a633aa862ff6b25572746b60d6f44&mpshare=1&scene=24&srcid=1208zj5g4Mr918zWgHdHk11k&sharer_sharetime=1638922355536&sharer_shareid=5fb9813bfe9ffc983435bfc8d8c5e9ca&key=daf9bdc5abc4e8d0bd6e2d93d1951b9f95bf22f6889f0be624d802ab3dfc2984761a3457e6581d2585f12e704ab3ba8ca413181ad9abc96b6e34e70796b868e03020c01d9d84ca419c80f01afcca17899f5cc946c0ec453f46f3040aa34ef373d96546dc7d2a9026ea8d463a0444361f192ead9a2acdd8fce98df9eeccba7f90&ascene=0&uin=MTEwNTU1MjgwMw%3D%3D&devicetype=Windows+11+x64&version=63090b19&lang=zh_CN&countrycode=CN&exportkey=n_ChQIAhIQYOLKLcyCunIaOBinb0ZMQRLmAQIE97dBBAEAAAAAACP4Jnh8wHEAAAAOpnltbLcz9gKNyK89dVj0ILIZkjDkHr5JthOEArwc1rMj9c2EoJE2Oa7OL8YM8wd2bWJ1IkHcQFYZNZfMN2%2Bngd50U9KL2vuz%2Bc1axiGZ6nHNP9clWfDflVqTJ9%2FOFOMFhlCiiSsRIaHBE91y8abNQLcSUxJa%2BR4Uqvoin3oWI1rL7IHov91lxzDhjw1ZCMTBLE9Qo8IYaqGWK%2Bc2B1dicAlu5PkRErI%2BuKnzzacK4mlu0Q0qMsSehOq3Q8Q7q%2FkePsh5tHVUkOZIp7lFGX%2FL&acctmode=0&pass_ticket=NwAJAtxTWl%2FrBtN%2BP677qjSxdoBw6c1GCJ6n0ndVsIQMf6k1zqNBzflxM%2FRsrjz%2B&wx_header=1&fasttmpl_type=0&fasttmpl_fullversion=7351805-zh_CN-zip&fasttmpl_flag=1)稀罕作者

1人喜欢

![](http://wx.qlogo.cn/mmopen/PiajxSqBRaEKUFJT0icPnQfnHssrupyTDLqD11Fo1gibQTibJ3g7g7ichUJx9sDcrPRyl9qQaWknrahbw4WMibTHicSfuWCAVNbgwxdsEuprbu696c36fMichfQxYeYSkpk6ibwCW/64)

白帽故事286

api8

黑客5

白帽故事 · 目录

上一篇【白帽故事】利用推荐码获得全账户接管下一篇国外一位女黑客分享的API 安全指南（下篇）

阅读 1483

​

写留言

**留言 2**

- 2021年11月28日
    
    赞
    
    奇怪了。不研究安全好多年。还给我推送这个![[流泪]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)我都没关注一个搞安全的公众号。。只是有几个微信好友。腾讯现在算法太侵犯隐私了吧
    
    骨哥说事
    
    作者2021年11月28日
    
    赞
    
    可能是从好友中看过或点赞过从而推荐给你
    

已无更多数据

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/hZj512NN8jmgmXq99BwSVFRkYfQUE1MO941mTf1fo026kyjDI8O0cYoyTSJz5Xe6wUePb2ApCzOsnP94QuZrNQ/300?wx_fmt=png&wxfrom=18)

骨哥说事

1736

2

写留言

**留言 2**

- 2021年11月28日
    
    赞
    
    奇怪了。不研究安全好多年。还给我推送这个![[流泪]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)我都没关注一个搞安全的公众号。。只是有几个微信好友。腾讯现在算法太侵犯隐私了吧
    
    骨哥说事
    
    作者2021年11月28日
    
    赞
    
    可能是从好友中看过或点赞过从而推荐给你
    

已无更多数据