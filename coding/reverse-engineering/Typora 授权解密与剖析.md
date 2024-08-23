# 

sunfishi 看雪学苑

 _2021年12月30日 17:59_

![](https://mmbiz.qpic.cn/sz_mmbiz_jpg/1UG7KPNHN8GNhS2q2lYlw5D5XXZyx3XTiasWyH0LL1z6taceq2cosW30IXexUrab0RbMRC3dem1os2NNoGbS3XA/640?wx_fmt=jpeg&wxfrom=13)  

本文为看雪论坛优秀文章

看雪论坛作者ID：sunfishi

  

11月23日，Typora 正式发布 1.0 版本，进入了收费时代。

1.0 版本是一次性付费而非订阅的，只要支付人民币 89 元，可以在 3 台设备里使用。

  

##   

1

  

Typora之于我

  

如你所见，这一篇文章就是使用Typora所写。自搭建个人博客起，Typora就成为了我主要的写作平台。  

用惯了Markdown，WordPress的古腾堡编辑器没法满足我的需求，于是开始寻找替代品，最终的结果便是typora。

当然，多数人使用的原因不外乎以下

- 轻盈、干净
    
- 所见即所得
    
- 图床
    
- 主题、生态
    
- (beta)免费
    
- ……
    

  

如今，typora进入收费阶段，不乏使用者被迫迁移至其他写作工具上。下面，我们来一探究竟。

  

##   

2

  

敬告

  

请勿使用盗版，支持正版授权，文中内容仅作学习和讨论，请不要从事任何非法行为。由此产生的任何问题都将读者/用户（您）承担。

##   

##   

3

  

寻踪觅源

  

通过火绒剑监测行为日志，程序加载的一些模块。

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/1UG7KPNHN8GNhS2q2lYlw5D5XXZyx3XTqrTy6qylCu8WZCWxp0f6gDveZTw6plEU7g21WF5wZPv9LFoU46sJKw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

在Windows下，typora会记录日志至{UsersRoot}\AppData\Roaming\Typora\typora.log，能看到可疑的注册表操作记录。

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/1UG7KPNHN8GNhS2q2lYlw5D5XXZyx3XTLsb7rREnIM2ibghaePRqQxtAyh4kmbBaJ6EDzib5vciaGqMOg7LkfHe2A/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)正版激活的注册项内容：

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/1UG7KPNHN8GNhS2q2lYlw5D5XXZyx3XTNVepOFd7M7r7rI5c1QrypRym3wuLrLSOW2ost82AcUjTEH6piansrrg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

尝试修改SLicense：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

重新运行软件后，从错误日志中发现调用栈暴露。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

##   

4

  

渐入佳境

  

这里关注到了app.asar，通过搜索引擎，尝试解包。

```
npm install -g asar
```

  

发现文件被加密：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

JavaScript不管是字节码还是明文脚本都会在运行时加载，结合模块列表寻找加载点。

关注到解包得到的main.node：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

IDA寻找字符串特征：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

通过交叉引用定位，看到一些导入函数。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

由字符串联想到对加密文件进行的base64解码，导入表查找到 napi Node-API | Node.js API 文档 (nodejs.cn)。

简单分析伪代码后，其实就是运行。

```
Buffer.from(e,"base64")
```

  

##   

5

  

刻舟求剑

  

尝试Findcrypt寻找算法，找到AES的Sbox和InvBox，通过交叉引用定位到可疑函数点 main.node+E440。

IDA动态调试，模块加载断点：

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

跑起来，直至加载main.node：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

分析模块后，定位base+offset下断，运行，看到：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

正好与我们的文件对应偏移16。

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

继续调试能看到 分组加密的形式：  

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

同时能够找到前16字节：  

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

正是作为iv进行异或：

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

##   

  

6

  

柳暗花明

  

分析调用函数，最终能够确定其函数功能：

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

通过偏移EF19，能够确定AES轮数为13轮，对应为AES 256，偏移B510处的函数，能够得到AESKey。

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

##   

  

7

  

落叶归根

  

解密得到明文脚本，授权主逻辑在Lisence.js中。

授权逻辑如下图：

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

本地验证->获取用户特征->网络验证授权->返回密文->RSA公钥解密->设备指纹对比

破解的思路，不多做阐述。

修改完成后，只需要按相同格式加密并打包为app.asar即可实现补丁Patch

##   

  

8

  

typoraCracker

  

typoraCracker是一个Typora解包解密程序，也是一个打包加密程序。你可以轻松的打造独属于你的补丁，但请注意法律上的可行性。

  

##   

9

  

测试

  

总有一种人，喜欢享受“正版”激活的感觉。而我就是……

我采用 Patch+KeyGen，补丁去除网络授权，KeyGen用于本地验证，测试成功。

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

typora针对electron下的源码加固仍是一片空白。

简单思考后，传统代码混淆的方式对关键逻辑的保护依然有较大的提升空间，不失为一个恰当的加固方向。

期待typora越做越好。——来自一个正版使用者

  

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**看雪ID：sunfishi**

https://bbs.pediy.com/user-home-883754.htm

*本文由看雪论坛 sunfishi 原创，转载请注明来自看雪社区

  

  

[![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458409596&idx=3&sn=658012a9debc76c73efed03342d41358&chksm=b18f6af686f8e3e09180522b243def309e265532cfadfdbf5becd7d9ec90f4505136959ebdfe&scene=21#wechat_redirect)  

  

**#** **往期推荐**

1.[内核漏洞学习-HEVD-StackOverflowGS](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458412464&idx=1&sn=5a039a6ed17cd45f21ce5a62193ecadb&chksm=b18f553a86f8dc2c5250cec782b359472e738f92b3a1444d56a2e1be19c4397574c0ae545dd3&scene=21#wechat_redirect)  

2.[人人都可以拯救正版硬件受害者(Jlink提示Clone)](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458412435&idx=1&sn=2291b3449965e4c7dadae15038d6ccf8&chksm=b18f551986f8dc0f62c7e4af9fd45ef493b10cc356f246e2416f641084f39ff4701922276658&scene=21#wechat_redirect)

3.[frida内存检索svc指令查找sendto和recvfrom进行hook抓包](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458412044&idx=1&sn=ef112858ca0f0ca0e103ceac6372253c&chksm=b18f548686f8dd907b435772d501200db5cf95d2bf62665a9edc09b3f20c0c62067aa108b3ee&scene=21#wechat_redirect)

4.[从0开始实现一个简易的主动调用框架](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458412042&idx=1&sn=f15dd34551a91eb3d7732fbdb1153cd0&chksm=b18f548086f8dd96535c2b41fce61602aa03f6409be2ef684a0e9fe608926cdab85e109764db&scene=21#wechat_redirect)

5.[Kernel从0开始](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458411220&idx=1&sn=bcb5872be8a7d92dee039fc565075057&chksm=b18f505e86f8d948bdad54fa350494b56fe67f2dbb8b0a3c8261b5b620f23609fc987c7fd8cb&scene=21#wechat_redirect)

6.[通过PsSetLoadImageNotifyRoutine学习模块监控与反模块监控](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458410644&idx=1&sn=e36fcc9b2b7a1efffb84bc3673b248b6&chksm=b18f6e1e86f8e708e321843fbdc68404b28db00aa48fed2c974a402a7708db095f94820c3da1&scene=21#wechat_redirect)

  

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

  

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**球分享**

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**球点赞**

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**球在看**

  

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

点击“阅读原文”，了解更多！

阅读原文

阅读 6867

​

写留言

**留言 15**

- 卡拉米
    
    2021年12月30日
    
    赞6
    
    这么硬核吗
    
- 201724
    
    2021年12月30日
    
    赞5
    
    89块钱还是值得入正的
    
- Desec
    
    2021年12月30日
    
    赞4
    
    注册表改个数字就可以无限试用
    
- 天蓝
    
    2021年12月30日
    
    赞2
    
    89块钱而已，何必。人家免费了那么多年，算是良心了。
    
- 大道不孤
    
    2021年12月30日
    
    赞2
    
    大佬![/::|](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
- 卡尔
    
    2021年12月30日
    
    赞2
    
    大佬牛皮![[抱拳]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
- 凯
    
    2021年12月31日
    
    赞
    
    支持正版
    
- 一抔
    
    2021年12月31日
    
    赞
    
    大佬，真硬核，我也试试去
    
- 海角虹楼
    
    2021年12月31日
    
    赞
    
    厉害！
    
- 清风
    
    2021年12月30日
    
    赞
    
    eagle早就破解了
    
- FK
    
    2021年12月30日
    
    赞
    
    89块的正版跟免费版的区别是啥？
    
- 阿强
    
    2021年12月30日
    
    赞
    
    原来安装的typora版本还能正常使用吗？
    
- Linbay
    
    2021年12月30日
    
    赞
    
    electron的软件eagle1.91后就无解了吧？一碰asar就无法运行软件，卸载重装后依然无法运行。这个软件的加密方式最牛，至今无人可破。
    
- 天马
    
    2021年12月30日
    
    赞
    
    老哥动作好快
    
- 信息牛
    
    2021年12月30日
    
    赞
    
    已入正，感谢分享
    

已无更多数据

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/1UG7KPNHN8EGLfh77kFmnicd9WOic2ibvhCibFdB4bL4srJCgo2wnvdoXLxpIvAkfCmmcptXZB0qKWMoIP8iaibYN2FA/300?wx_fmt=png&wxfrom=18)

看雪学苑

59分享18

15

写留言

**留言 15**

- 卡拉米
    
    2021年12月30日
    
    赞6
    
    这么硬核吗
    
- 201724
    
    2021年12月30日
    
    赞5
    
    89块钱还是值得入正的
    
- Desec
    
    2021年12月30日
    
    赞4
    
    注册表改个数字就可以无限试用
    
- 天蓝
    
    2021年12月30日
    
    赞2
    
    89块钱而已，何必。人家免费了那么多年，算是良心了。
    
- 大道不孤
    
    2021年12月30日
    
    赞2
    
    大佬![/::|](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
- 卡尔
    
    2021年12月30日
    
    赞2
    
    大佬牛皮![[抱拳]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
- 凯
    
    2021年12月31日
    
    赞
    
    支持正版
    
- 一抔
    
    2021年12月31日
    
    赞
    
    大佬，真硬核，我也试试去
    
- 海角虹楼
    
    2021年12月31日
    
    赞
    
    厉害！
    
- 清风
    
    2021年12月30日
    
    赞
    
    eagle早就破解了
    
- FK
    
    2021年12月30日
    
    赞
    
    89块的正版跟免费版的区别是啥？
    
- 阿强
    
    2021年12月30日
    
    赞
    
    原来安装的typora版本还能正常使用吗？
    
- Linbay
    
    2021年12月30日
    
    赞
    
    electron的软件eagle1.91后就无解了吧？一碰asar就无法运行软件，卸载重装后依然无法运行。这个软件的加密方式最牛，至今无人可破。
    
- 天马
    
    2021年12月30日
    
    赞
    
    老哥动作好快
    
- 信息牛
    
    2021年12月30日
    
    赞
    
    已入正，感谢分享
    

已无更多数据