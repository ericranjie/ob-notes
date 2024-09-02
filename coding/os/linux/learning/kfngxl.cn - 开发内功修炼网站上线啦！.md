

原创 张彦飞allen 开发内功修炼

 _2024年03月27日 09:10_ _北京_

大家好，我是飞哥！  

和大家分享一个事情，最近我花了两个周末，搞了个开发内功修炼技术网站。站点地址是 https://kfngxl.cn 。请大家收藏，后续我写的新文章也会陆续同步到网站上。

我估计不少同学也对建站的过程感兴趣，所以今天我专门写一篇文章来给大家分享下我是如何把这个小站搭起来的。

### 建站的动机

最早诞生想要搭建一个网站的想法是在两年前。

首先是因为各家写技术文章的平台一般都会有各种限制，比如知乎不让随便嵌二维码，微信公众号不让有外站的超链。而如果能建一个自己的技术网站就没这个限制了，想咋玩咋玩。所以就想建个自己的站。

另外还有个原因是想搞个图床。因为我写的文章中包含了很多的图片。在我把技术文章同步到各个平台的时候，最影响效率的就是图片问题。每次上传一个平台都需要把图片重新手工上传，非常的麻烦。如果有个自己外网能访问的图床这些事情都会方便很多。

后来在2021年9月的时候，我买了个kfngxl的域名。但是由于事情太多，就一直把建网站的事情给抛开了没搞。域名也一直处于闲置状态。

就在前几天在腾讯云活动上花 198 搞到两年的轻量云服务器后（活动地址在文章尾部），我觉得还是该把这件事给启动起来。所以搞了两个周末终于把网站给搭起来了。

### 建站技术选型

现在建站的技术太多了，可谓是让人眼花缭乱。

首先一提到技术博客，最值得推荐的都是使用 Github Pages。Github Pages 可以直接将网站的静态页面托管到 Github 的仓库中，连服务器都不需要买，非常的方便。

再搭配上静态博客生成器，如 jekyll、 hexo 和 hugo 等。可以方便地将 markdown 渲染成指定风格的页面，并和图片一起发布。我之前在搜狗的时候，也在团队里让大家用过一段时间。

不过静态站的虽然样式很多，也可选。但终究是一个静态页，最终都是固定的。我个人不太喜欢静态页面受束缚的感觉。而且由于也没有自己的服务器，体会不到自己驾驭服务器和代码的快感。

再加上我工作的中从事过多年的 web 开发，对 LNMP 技术栈很熟悉。所以我还是更想搞一个动态网站。这样每一行代码都是自己可以控制的。

在动态网站里，最流行的是 WordPress。但是我更希望找一个轻量一点的项目。因为虽然我一来是喜欢简洁的风格，不喜欢太繁杂的项目。二来我是想后面进行一些二次开发的。但我并没有太多的时间消化和理解这些开源项目的代码。

后来我选中的是 Typecho，一个非常轻量的博客网站项目。这个项目的源代码压缩后只有 610 KB。想看懂这些代码并花不了多长时间，非常对我的胃口！

买好了服务器，技术选型也做好了之后，下一步就是实际动手了。

### 部署 typecho

搭建个人技术网站最实用的就是轻量云服务器。因为它不仅仅是有 2C2G 的算力，更重要的是提供了一个外网 IP，还有 4M 的外网带宽，和 300GB 每月的流量。

服务器买下来后，首先是安装镜像。虽然腾讯云上有 Typecho 的镜像。但是我更推荐的是使用宝塔镜像。因为使用宝塔后台管理服务器太方便了。在服务器上安装 LNMP 软件，修改配置、重启服务、给服务器上传文件，甚至是直接在服务器上云编辑源代码都非常容易。

使用宝塔后台执行了如下几步操作。

1）**安装 Nginx、Mysql 和 PHP**。

记得版本要选的稍微高一点，别太低了。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

2）**配置 host**。

一般在自己的 Linux 上，需要手工修改 Nginx 的配置文件来添加 host。但是宝塔后台直接在界面上可以操作。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

这个时候，其实一个基于 Nginx 的静态网站就做好了。index.html 等文件都是外网可访问的了。

3）**下载并安装 typecho**

到官网上下载 typecho 源码。然后通过宝塔后台上传到服务器中 host 的根目录下。然后解压。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

4）**动态网站初始化**typecho是基于 Mysql 来工作的。所以需要事先准备一个 Mysql 实例，这个同样在宝塔界面上就可以创建。创建完成后，把用户名密码记下来。然后通过 ip 访问 typecho 的管理后台。按照说明填入数据库用户名密码。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

之后网站就创建成功，并可以通过外网IP访问了。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

### 域名备案

直接通过 IP 访问的方式太山寨了。所以还需要一个域名。我掏出了两年前在阿里云上购买的 kfngxl.cn，给它配置了一条 DNS 的 A 记录。这样访问域名就可以自动解析到我的服务器上。（由于域名是在阿里云买的，所以域名是在阿里云后台完成）

但是帅不过 3 秒。仅仅过了十分钟，腾讯云就提示由于域名未备案。域名的方式就被掐断了。不过好在备案过程不复杂，自己只是初步填了一个信息，后面很快就有腾讯云的客服电话联系。告知哪里信息填的不合适，该如何修改。不需要自己太操心。

所以接下来就是长达一周多的域名备案等待。大约6-7个工作日后。备案就成功了。

### 网站“装修”

既然ICP备案都下来了，网站还保持毛坯房状态是不太好的。所以我下决心花点时间好好把网站装修装修。

原始的 typecho 外观太朴素了。所以需要一个更好一点的外观样式。于是我把 typecho 官网翻了一遍，最终选定的是 DUX 这个主题。

主题的源码下载后上传到网站的 usr/themes 目录下。接着需要进行一些配置。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

网页的顶部导航栏、是否启用缩略图样式、是否配置幻灯片、侧边栏都显示哪些内容、作者简介、代码是否高亮展示、底部备案号、百度统计 js 代码等都是在上面的界面中配置的。

### 自研图片处理插件

由于我的技术博客中有大量的图片，所以我的网站需要合理处理图片。

关于图片的第一个需求是要能自动添加图片水印。原因是网上总有那么一小撮人随便抄袭和转发别人的劳动成果，并且连个出处都不标。我下载了个 waterMark 插件。

第二个需求是图片要进行压缩。我的不少图片有300多KB的，直接传到博客上的话，会浪费不少流量。因此我希望将来编辑文章上传图片的时候，图片能够自动进行压缩。因此又下载了个插件。

但是很不幸，上面两个插件配合的不好。本来压缩插件能压缩掉差不多70%的体积。但是经过图片水印插件一处理，体积又增加了好多。原因是插件之间运行的顺序不好控制。

后来我一咬牙，自己参考这两个插件的基础之上，开发了个新插件。新插件对上传的图片先添加水印，水印固定添加在图片的中心位置，并且给图片设置默认的灰色底色。然后再使用 pngquant 对 png 进行压缩。搞到晚上 3 点多终于开发成功了。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

300 KB 的图片成功搞到了 100 KB 左右。

### 申请 HTTPS 证书

现在的网站，HTTPS 是标配了。默认的 HTTP 页面在用户的浏览器里会被提示不安全。所以申请个证书也是很有必要的。我在腾讯云上申请了免费的证书。通过宝塔配置到网站上

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

剩下还有最后一个事是公安备案。根据腾讯云的提示是要求处理。根据网上的经验，我把网站的评论功能关了。不给祖国和自己添麻烦。我是按说明提交了公安备案，至于结果等两个个星期再看。

### 内容同步

关注过我的读者也都知道我发表了有上百篇的硬核技术文章。这些文章之前都是在公众号和知乎平台上发布的。现在想把他们在自己的站上再发表一遍的话工作量可不小。

所以我把我老婆也卷入到了这个小站的建设中来了。她来帮我把一篇一篇的历史文章同步到网站中。目前已经把网络篇同步完了。CPU、内存、磁盘方面的文章预计接下来一周左右同步完。

到网站上看文章的时候要记得感谢你们的飞嫂！  

### 建站总结

建站的大概步骤是

- 购买轻量云服务器
    
- 安装LNMP、安装 typecho、
    
- ICP 备案
    
- 配置装修动态网站
    
- 申请配置 HTTPS 证书
    
- 公安备案
    

至于成本的话，我觉得主要是精力的投入成本。钱的投入买对服务器的话很便宜。我的支出是 99 一年的腾讯云轻量服务器买了两年花了 198 元，域名花了 150 多。其它的就没有了。

腾讯云 99 服务器是在2024采购节上买的，我相中的是它让优惠价再多续费一年的政策，活动地址在下方的小程序里。如果只买一年的话更便宜，只要50多块。

[![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)](https://mp.weixin.qq.com/s?__biz=MjM5Njg5NDgwNA==&mid=2247491809&idx=1&sn=169e3f7888cf8332cef5526328f12b4b&chksm=a6e0e1da919768cc4ef83128cf096b0b9c970bec681bc1aaf92bd11874abc94200d5679f2969&mpshare=1&scene=24&srcid=03293IzcqRVRhAFizhQF7e2k&sharer_shareinfo=0a942377eea06b2e637a4a74ecbf65c6&sharer_shareinfo_first=0a942377eea06b2e637a4a74ecbf65c6&key=daf9bdc5abc4e8d0163943f101b20aa28bcf4e3758e1e1e20f4220f01bc858f8ca42a9f2205848e2d747cf07ba845dd3146be5b56392ebaf5a819d614af9a12ecd6f72cb6e443353717829678e3b6c589dc88456d91fa9f3abffcd64658a7f71807545636c714a286abe9e78a0a4d79c472f792bfb2f179629b4ba5bd12eef0b&ascene=0&uin=MTEwNTU1MjgwMw%3D%3D&devicetype=Windows+11+x64&version=63090b19&lang=zh_CN&countrycode=CN&exportkey=n_ChQIAhIQyxaHkdoBSA3EhKAhRbV%2FmxLmAQIE97dBBAEAAAAAAEqnM91V96YAAAAOpnltbLcz9gKNyK89dVj0sLBITMOGeTsU4rtnTVvz630R49mu2eMYw0Fdn%2F6Vv1NojC%2BX5mREK%2F5G2svzI5JBsl31q2UjcvSx5hNqziR%2B5okvTvViuRd2x6V47kuKGc2tAOrmOG8PYcmLLf1fv2lBhsDRzmIIAolxSZn9zhFM4T0CKgpYLhCWyFknprxznAzBHmS330mzLpQsEpKaaMEpUjjOdq%2Bybfzy%2BNpc6tf6qa0ECFt3LUW9sWDPKljT6Mxo8%2BeEd4aOb8tHw8%2BpUKZ%2F&acctmode=0&pass_ticket=cTgP%2FbyvpHtXePrwegwLhX1ayZszDvcyd8RzoQ4oMa6N0Y4m4k5yf%2BASi0RRbl6Y&wx_header=1&fasttmpl_type=0&fasttmpl_fullversion=7350504-zh_CN-zip&fasttmpl_flag=1)

网站功能开发完毕以后，我发布了几篇文章，以下是网站的最终效果。

这是网站最终效果图。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

移动端上适配的效果也不错。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

总之，动态站确实会需要一些人力成本。不是像网上很多人说的那样，不需要任何技术能力就可以维护一个动态站。我在中间遇到过好几次问题，最后都是翻开源码找到问题后解决的。

如果你想简单轻松一点，用 Github Pages 搞个静态栈就可以了。如果你喜欢动态站的自由度，并且也不怕麻烦，还是很推荐搞一个真正的动态网站的。能通过技术驾驭服务器和代码，感觉会很不错。而且更重要的是功能扩展起来也方便，自己想加啥功能写几行代码就可以了，无拘无束！

网站地址：https://kfngxl.cn  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

![](https://mmbiz.qlogo.cn/mmbiz_jpg/iaMl8iazctEu93J6Io4KDZQmx03HErmVeXnSnUAQ0MD6Nia3d97E8vmljv5DibLTWSeLES1JHicWVA2mynPVlbt1FAA/0?wx_fmt=jpeg)

张彦飞allen

 赞赏不分多少，头像出现就好！ 

![赞赏二维码](https://mp.weixin.qq.com/s?__biz=MjM5Njg5NDgwNA==&mid=2247491809&idx=1&sn=169e3f7888cf8332cef5526328f12b4b&chksm=a6e0e1da919768cc4ef83128cf096b0b9c970bec681bc1aaf92bd11874abc94200d5679f2969&mpshare=1&scene=24&srcid=03293IzcqRVRhAFizhQF7e2k&sharer_shareinfo=0a942377eea06b2e637a4a74ecbf65c6&sharer_shareinfo_first=0a942377eea06b2e637a4a74ecbf65c6&key=daf9bdc5abc4e8d0163943f101b20aa28bcf4e3758e1e1e20f4220f01bc858f8ca42a9f2205848e2d747cf07ba845dd3146be5b56392ebaf5a819d614af9a12ecd6f72cb6e443353717829678e3b6c589dc88456d91fa9f3abffcd64658a7f71807545636c714a286abe9e78a0a4d79c472f792bfb2f179629b4ba5bd12eef0b&ascene=0&uin=MTEwNTU1MjgwMw%3D%3D&devicetype=Windows+11+x64&version=63090b19&lang=zh_CN&countrycode=CN&exportkey=n_ChQIAhIQyxaHkdoBSA3EhKAhRbV%2FmxLmAQIE97dBBAEAAAAAAEqnM91V96YAAAAOpnltbLcz9gKNyK89dVj0sLBITMOGeTsU4rtnTVvz630R49mu2eMYw0Fdn%2F6Vv1NojC%2BX5mREK%2F5G2svzI5JBsl31q2UjcvSx5hNqziR%2B5okvTvViuRd2x6V47kuKGc2tAOrmOG8PYcmLLf1fv2lBhsDRzmIIAolxSZn9zhFM4T0CKgpYLhCWyFknprxznAzBHmS330mzLpQsEpKaaMEpUjjOdq%2Bybfzy%2BNpc6tf6qa0ECFt3LUW9sWDPKljT6Mxo8%2BeEd4aOb8tHw8%2BpUKZ%2F&acctmode=0&pass_ticket=cTgP%2FbyvpHtXePrwegwLhX1ayZszDvcyd8RzoQ4oMa6N0Y4m4k5yf%2BASi0RRbl6Y&wx_header=1&fasttmpl_type=0&fasttmpl_fullversion=7350504-zh_CN-zip&fasttmpl_flag=1)喜欢作者

9人喜欢

![](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

非技术文18

非技术文 · 目录

上一篇上个月我去参加2023 CPP-Summit了

阅读原文

阅读 7864

​