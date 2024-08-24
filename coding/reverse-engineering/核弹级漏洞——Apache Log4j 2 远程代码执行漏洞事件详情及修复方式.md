# 

看雪学苑

 _2021年12月10日 15:34_

![](https://mmbiz.qpic.cn/mmbiz_png/kR3MmOkZ9r7sH5gQxovib9zlQrF2WXzr7gNya6ABWyXN1lBkHQiaW9J0UrKl5HuIdAftBtD3L6fnvNgFiafygEIMQ/640?wx_fmt=png&wxfrom=13&tp=wxpic)

编辑：左右里

  

昨晚一直到现在安全圈可谓是沸腾了，所有人都在关注着一个漏洞——Apache Log4j 2远程代码执行。

  

Apache Log4j 2是一款非常优秀且流行的开源Java日志记录组件，在各大项目中有着广泛应用。利用该漏洞，攻击者可通过构造恶意请求在目标服务器上执行任意代码，从而实施窃取数据、挖矿、勒索等行为。

  

**事件详情：**

2021年12月9日，Apache官方发布了紧急安全更新以修复该远程代码执行漏洞，但更新后的Apache Log4j 2.15.0-rc1 版本被发现仍存在漏洞绕过，多家安全应急响应团队发布二次漏洞预警。

  

12月10日凌晨2点，Apache官方紧急发布log4j-2.15.0-rc2版本，以修复Apache log4j-2.15.0-rc1版本远程代码执行漏洞修复不完善导致的漏洞绕过。

  

据了解，此次漏洞是由 Log4j2 提供的 lookup 功能造成的，该功能允许开发者通过一些协议去读取相应环境中的配置。但在处理数据时，并未对输入（如${jndi）进行严格的判断，从而造成注入类代码执行。

  

**漏洞影响范围：**

Java类产品：Apache Log4j 2.x < 2.15.0-rc2

  

受影响的应用及组件（包括但不限于）如下：

Apache Solr、Apache Flink、Apache Druid、Apache Struts2、srping-boot-strater-log4j2等。

更多组件可参考如下链接：

_https://mvnrepository.com/artifact/org.apache.logging.log4j/log4j-core/usages?p=1_

  

**紧急补救措施：**

（1）设置jvm参数 -Dlog4j2.formatMsgNoLookups=true。

（2）设置log4j2.formatMsgNoLookups=True。

（3）设置系统环境变量 FORMAT_MESSAGES_PATTERN_DISABLE_LOOKUPS 为 true。

（4）采用 rasp 对lookup的调用进行阻断。

（5）采用waf对请求流量中的${jndi进行拦截。

（6）禁止不必要的业务访问外网。

  

**攻击检测：**

（1）可以通过检查日志中是否存在“jndi:ldap://”、“jndi:rmi”等字符来发现可能的攻击行为。

（2）检查日志中是否存在相关堆栈报错，堆栈里是否有JndiLookup、ldapURLContext、getObjectFactoryFromReference等与 jndi 调用相关的堆栈信息。

  

**修复建议：**

目前漏洞POC已被公开，官方已发布安全版本，若是使用Java开发语言的系统需要尽快确认是否使用Apache Log4j 2插件，并尽快升级到最新的 log4j-2.15.0-rc2 版本。

  

地址：_https://github.com/apache/logging-log4j2/releases/tag/log4j-2.15.0-rc2_

  

  

  

参考：360网络安全响应中心、 阿里云应急响应、绿盟科技CERT、 奇安信 CERT、斗象智能安全、默安科技

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)  

  

推荐文章++++  

* [2021年报告的漏洞总数揭露，创历史新高](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458408193&idx=2&sn=90491e6aa839975350b0fc5e9edc497c&chksm=b18f658b86f8ec9de619955815b243c265bbda11cbcaff7303927bb1d310d1324077105d0d87&scene=21#wechat_redirect)

* [全球范围多家知名平台受波及，亚马逊云服务中断](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458408047&idx=4&sn=daa4bbb0a5a8757a2c91bc8a87afdbb6&chksm=b18f64e586f8edf3cd7aed6c9238b613effdd19091d824df9a31b58bee7bd659e390d220d2f7&scene=21#wechat_redirect)

* [商店或关闭或只支持现金，英国北部SPAR遭遇IT中断](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458407946&idx=4&sn=81b76cd67e8dcdee4edd5d7ce724f916&chksm=b18f648086f8ed96ae1f9b3572308112518e501ce5a4bd72edbf75d9567f1fa2ca71232d5b9a&scene=21#wechat_redirect)

* [加密货币失窃超1.2亿美元，金融平台BadgerDAO遭黑客攻击](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458407897&idx=4&sn=0e24568ca388d9062480ef7b7ea47823&chksm=b18f635386f8ea45534806a7ffb2ffd015cfad93bf16a0c76e0280df43ca2a0af3980ad210db&scene=21#wechat_redirect)

* [近两万钱骡，欧洲第七次联合打击洗钱行动落幕](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458407655&idx=3&sn=871d345f875c313f793fca0816db844d&chksm=b18f626d86f8eb7b9fe654d9ba017faf6052382de17368d62c78a8a90dfa80a6413d2b4273bc&scene=21#wechat_redirect)

* [家贼难防，Ubiquiti前雇员窃取公司数据并实施勒索](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458406981&idx=3&sn=2b42c427dbef2c752ea03a6cdc6e2092&chksm=b18f60cf86f8e9d930783ef1d05e1d09b9a9499d7d74e4f05fbfac913f70e71e186a2997885e&scene=21#wechat_redirect)

* [数百万恶意短信肆虐，芬兰发布严重警报](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458406927&idx=3&sn=13589fa605f8df8727bf8c78ba28ca80&chksm=b18f608586f8e9935dfc9770070f0a1d0b0f119cd62b29eb829591054067b0a42be25b28c005&scene=21#wechat_redirect)

  

  

  
﹀  

﹀

﹀

  

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

公众号ID：ikanxue  

官方微博：看雪安全

商务合作：wsc@kanxue.com

  

  

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**球分享**

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**球点赞**

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**球在看**

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

戳“阅读原文”一起来充电吧！

阅读原文

阅读 1.9万

​

写留言

**留言 16**

- Mr. Xu
    
    2021年12月10日
    
    赞7
    
    经验证，Log4j2有效修复方案：1、方案一：升级版本到2.10.0及其以上，然后设置启动参数-Dlog4j2.formatMsgNoLookups=true2；方案二：直接升级到2.15.0，自己拉github源码编译。
    
    置顶
    
- Alec
    
    2021年12月10日
    
    赞87
    
    ![[旺柴]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)这波养活了多少安全工程师
    
- 给小乔的情书
    
    2021年12月10日
    
    赞37
    
    今天已经被这个洞搞炸了 终于见证了历史性的一刻
    
- 温馨
    
    2021年12月10日
    
    赞22
    
    这个漏洞太强了，fofa、钟馗之眼等网站都挤爆了，，，dnslog网站昨晚真的是有史以来访问量最多的一天
    
- Scruel
    
    2021年12月10日
    
    赞19
    
    2.15.0 已经 release 到 maven center 了，不需要用什么 rc 版本。另外这个版本的 release notes 里是提及到了这个漏洞是已被修复了的，且接口的 API 也是兼容的。现有应用可以引入 log4j-bom:2.15.0 统一升级版本快速解决该漏洞问题。
    
- 影光
    
    2021年12月10日
    
    赞15
    
    越多新项目转go还是有道理的，但凡在1995－2009这 15年间流行起来的语言，或多或少会有一些修不好的安全问题，因为历史包袱太重了，21世纪前10年，各种划时代的攻击方式出现，横跨web 二进制 协议等等，很多老的协议，特定时期下的语言特性当时开始处于被抛弃的边缘，但是作为一门流行语言又不得不支持这些功能，作为服务端常用语言来说，java这种过于灵活的远程类加载 和 php的rfi在当时那种网速和开发调试环境下是很好的特性(现在对安全人员来说是很好的特性/::> )，但如果在今天完全可以用微服务或者其他方式替代。现在语言开发者能做的也就是在新版本开发包内添加更严格的开关限制，慢慢把历史包袱甩掉，引导新项目用别的方式替代
    
- Chelth
    
    2021年12月10日
    
    赞14
    
    其他行业看安全：![[无语]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![[吃瓜]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![[666]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
- 暮雨斜阳
    
    2021年12月10日
    
    赞10
    
    所以说嘛，控制台输出永远的神![[偷笑]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
- 天涯浪子
    
    2021年12月10日
    
    赞10
    
    好家伙，基本java开发日志用的都是log4j，真的是绝了
    
- 渺渺兮予怀
    
    2021年12月10日
    
    赞8
    
    上一个惊动整个安全圈的还是永恒之蓝
    
- 子规
    
    2021年12月10日
    
    赞6
    
    别激动。必须是log4j 2.x 版本的，还要来rmi 和ldap url 限制，java 8 221版本以上天然免疫。springboot 2.x天然免疫。
    
- Phoenix Huang
    
    2021年12月10日
    
    赞5
    
    都在等release版
    
- 不啻于
    
    2021年12月11日
    
    赞2
    
    在失业边缘惊现直升机
    
- PC_XT
    
    2021年12月11日
    
    赞2
    
    闹挺大，Minecraft也被影响了
    
- ZYF_
    
    2021年12月10日
    
    赞2
    
    今天在服务器上复现了一下发现复现不了，服务器装的是java17
    
- 俨文母上大人
    
    2021年12月12日
    
    赞
    
    服务器瑟瑟发抖
    

已无更多数据

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/1UG7KPNHN8EGLfh77kFmnicd9WOic2ibvhCibFdB4bL4srJCgo2wnvdoXLxpIvAkfCmmcptXZB0qKWMoIP8iaibYN2FA/300?wx_fmt=png&wxfrom=18)

看雪学苑

39分享27

16

写留言

**留言 16**

- Mr. Xu
    
    2021年12月10日
    
    赞7
    
    经验证，Log4j2有效修复方案：1、方案一：升级版本到2.10.0及其以上，然后设置启动参数-Dlog4j2.formatMsgNoLookups=true2；方案二：直接升级到2.15.0，自己拉github源码编译。
    
    置顶
    
- Alec
    
    2021年12月10日
    
    赞87
    
    ![[旺柴]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)这波养活了多少安全工程师
    
- 给小乔的情书
    
    2021年12月10日
    
    赞37
    
    今天已经被这个洞搞炸了 终于见证了历史性的一刻
    
- 温馨
    
    2021年12月10日
    
    赞22
    
    这个漏洞太强了，fofa、钟馗之眼等网站都挤爆了，，，dnslog网站昨晚真的是有史以来访问量最多的一天
    
- Scruel
    
    2021年12月10日
    
    赞19
    
    2.15.0 已经 release 到 maven center 了，不需要用什么 rc 版本。另外这个版本的 release notes 里是提及到了这个漏洞是已被修复了的，且接口的 API 也是兼容的。现有应用可以引入 log4j-bom:2.15.0 统一升级版本快速解决该漏洞问题。
    
- 影光
    
    2021年12月10日
    
    赞15
    
    越多新项目转go还是有道理的，但凡在1995－2009这 15年间流行起来的语言，或多或少会有一些修不好的安全问题，因为历史包袱太重了，21世纪前10年，各种划时代的攻击方式出现，横跨web 二进制 协议等等，很多老的协议，特定时期下的语言特性当时开始处于被抛弃的边缘，但是作为一门流行语言又不得不支持这些功能，作为服务端常用语言来说，java这种过于灵活的远程类加载 和 php的rfi在当时那种网速和开发调试环境下是很好的特性(现在对安全人员来说是很好的特性/::> )，但如果在今天完全可以用微服务或者其他方式替代。现在语言开发者能做的也就是在新版本开发包内添加更严格的开关限制，慢慢把历史包袱甩掉，引导新项目用别的方式替代
    
- Chelth
    
    2021年12月10日
    
    赞14
    
    其他行业看安全：![[无语]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![[吃瓜]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![[666]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
- 暮雨斜阳
    
    2021年12月10日
    
    赞10
    
    所以说嘛，控制台输出永远的神![[偷笑]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
- 天涯浪子
    
    2021年12月10日
    
    赞10
    
    好家伙，基本java开发日志用的都是log4j，真的是绝了
    
- 渺渺兮予怀
    
    2021年12月10日
    
    赞8
    
    上一个惊动整个安全圈的还是永恒之蓝
    
- 子规
    
    2021年12月10日
    
    赞6
    
    别激动。必须是log4j 2.x 版本的，还要来rmi 和ldap url 限制，java 8 221版本以上天然免疫。springboot 2.x天然免疫。
    
- Phoenix Huang
    
    2021年12月10日
    
    赞5
    
    都在等release版
    
- 不啻于
    
    2021年12月11日
    
    赞2
    
    在失业边缘惊现直升机
    
- PC_XT
    
    2021年12月11日
    
    赞2
    
    闹挺大，Minecraft也被影响了
    
- ZYF_
    
    2021年12月10日
    
    赞2
    
    今天在服务器上复现了一下发现复现不了，服务器装的是java17
    
- 俨文母上大人
    
    2021年12月12日
    
    赞
    
    服务器瑟瑟发抖
    

已无更多数据