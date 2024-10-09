# 

原创 qcrao 码农桃花源

_2021年09月08日 09:00_

某天写代码时发现自己对 IDE 的依赖非常深，如果没了 Goland 就不会写代码了，心里为之一惊。

Goland 的自动补全功能已经是必需品了，只要打出相关的几个字符，不管是变量名还是函数调用，都能帮你直接补全。我们只需要往相应的位置填东西就行了。

进而又想到，当补全功能缺失或者暂时失灵的情况下，该如何快速地查出某个函数的具体用法呢？

假设我们想要对字符串做 split，却忘了具体用法，下面是几种常见的查文档方法。

# Google

![图片](https://mmbiz.qpic.cn/mmbiz_png/ASQrEXvmx60Ej12RxibHRhU7AVflnGY3DHuUhkqibUnGhnXYWibsq9ndr1giboWvN0rLujibEqAZCTXSFciaYzibTeChw/640?wx_fmt=png&wxfrom=13&tp=wxpic)

google

在设置了语言是 english 的情况下，还是挺精准的。直接定位到 Go 官方文档。

# Dash

![图片](https://mmbiz.qpic.cn/mmbiz_png/ASQrEXvmx60Ej12RxibHRhU7AVflnGY3DlLmuRVNicdQI5pueAUl1ADTXtMqGZb1ibNdvukOv4YiagpJ6qlQEial5Dw/640?wx_fmt=png&wxfrom=13&tp=wxpic)

Dash

同样很准确，搜索词不需要很精准。

# devdocs.io\[1\]

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

devdocs

这个也不错，而且支持很多种语言。

# pkg.go.dev

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

pkg.go.dev

优点是官方文档，最权威，逼格最高。缺点是要准确地记住包名+函数名。

# go doc

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

cmd

优点是直接 iTerm2 里就可以查看，缺点是需要准确地记住包名+函数名。

有些大佬用 vim 写代码，在 shell 环境里直接能查文档，还是很有用的。不过对我等用 Goland 的菜鸡用处不大。😂

______________________________________________________________________

上面这几种方法我用得最多的还是 Google，可能这并不是最快的方式，但是它总是能帮你找到所有有用的信息。没有 Google，我可能也不会写代码了。

最近看到一篇文章\[2\]，就讲了如何利用 Go 标准库做出一个好用的查文档工具。

原理是利用 Go 提供的包解析工具，把所有的导出类型列出来。然后在我们搜索的时候用模糊匹配的方式找到符合的类型，再用这个精确的类型调用 `go doc`。

流程如下：

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

gdoc 原理

在 Linux 下结合 dmenu，使用非常顺滑：

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

gdoc-cmd

偷个懒，直接用原文的动图。😀

当然，不嫌弃浏览器的情况下，还提供了一个可视化的界面，同样有模糊匹配的功能且可以一键直达 `pkg.go.dev` 对应的页面。比 google 可能快一点。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

gdoc-web

选中其中一个，会直接跳转过来：

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

跳转到 pkg.go.dev

# 后记

不过，即使知道了这些方法，可能最后还是会退化到用 Google 直接搜，因为啥都不需要记，所有的东西都可以用 Google 搜索出来。

这也是最方便的方法，什么额外的事情都不用做。因为方便，成本低，自然就想把所有的事情都挪到它上面来做，即使有很多专业的查文档工具的情况下，还是会这么做。

一件事，如果容易，那就会经常做。反之，如果成本比较高，结果不是做这件事花的时间更多，而是我们选择不去做它。

不知道你平时查文档时用的什么方法，欢迎留言一起讨论。

### 参考资料

\[1\]

devdocs.io: _https://devdocs.io/_

\[2\]

文章: _https://eli.thegreenplace.net/2018/command-line-autocomplete-for-go-documentation/_

Go 语言33

终身学习14

Go 语言 · 目录

上一篇\[\]int 能转换为 \[\]interface 吗？下一篇介绍一个欧神写的剪贴板多端同步神器

阅读原文

阅读 1984

​

写留言

**留言 5**

- Levy

  2021年9月8日

  赞2

  不错，看完了，我选择用goland

- Siege lion

  2021年9月8日

  赞1

  补充一下，windows下的Zeal与Dash等同

  码农桃花源

  作者2021年9月8日

  赞

  ![[强]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

- Hollson

  2021年9月9日

  赞

  再好的第三方都不用

- Asong

  2021年9月8日

  赞

  赞![[耶]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

已无更多数据

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/ASQrEXvmx62PpTb2V6eBBF4lNBnFP1zsL9CJcqsECwriaKh9NoHqqiaIJyhgudorYzkgtDvDvRX6bLp7CPMKufibg/300?wx_fmt=png&wxfrom=18)

码农桃花源

11分享2

5

写留言

**留言 5**

- Levy

  2021年9月8日

  赞2

  不错，看完了，我选择用goland

- Siege lion

  2021年9月8日

  赞1

  补充一下，windows下的Zeal与Dash等同

  码农桃花源

  作者2021年9月8日

  赞

  ![[强]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

- Hollson

  2021年9月9日

  赞

  再好的第三方都不用

- Asong

  2021年9月8日

  赞

  赞![[耶]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

已无更多数据
