
CPP开发者

 _2022年06月21日 11:50_ _浙江_

↓推荐关注↓

![](http://mmbiz.qpic.cn/mmbiz_png/DSU8cv1j3ibStMRcibJLd4TkNlt53KNZj0A2IicORH4REC4ics87icsx703M5giby2wuofz3dicMsHVcXDMXTM6t6VBQw/300?wx_fmt=png&wxfrom=19)

**开源前哨**

点击获取10万+ star的开发资源库。 日常分享热门、有趣和实用的开源项目～

167篇原创内容

公众号

谷歌 Chrome 安全团队称其一直在致力于改善 Chrome 浏览器的内存安全；近期，该团队正在研究使用 heap scanning 技术来提高 C++ 的内存安全。

虽然从内存安全方面出发，Rust 当下可能更受大众喜爱。但 Chrome 安全团队认为，尽管人们对比 C++ 具有更强内存安全保证的其他语言有兴趣，但在可预见的未来，像 Chromium 这样的大型代码库将使用 C++。鉴于此，Chrome 工程师已经找到了使 C++ 更安全的方法，以减少缓冲区溢出和 use-after free (UAF) 等与内存相关的安全漏洞。数据表明，这些漏洞占所有软件安全漏洞的 70%。

`auto* foo =newFoo();      delete foo;      // The memory location pointed to by foo is not representing      // a Foo object anymore, as the object has been deleted (freed).      foo->Process();`

  

如上示例，当应用程序使用的内存被返回到底层系统，但指针指向一个过期的对象时，就会出现一个被称为悬空指针（dangling pointers）的情况，通过它进行的任何访问都会导致 UAF 访问。在最好的情况下，此类错误会导致 well-defined 的崩溃；在最坏的情况下，它们会造成可以被恶意行为者利用的破坏。 

在较大的代码库中，UAF 通常很难被发现，因为对象的所有权是在不同组件之间转移的。这个问题非常普遍，以至于到目前为止，工业界和学术界都在频繁地针对其提出缓解策略。而 Chrome 中 C++ 的使用也没有什么不同，大多数高严重性安全漏洞都是 UAF 问题。近期发布的 Chrome 102 中，就修复了一个关键的 UAF 问题，且八个高危漏洞中有六个是 UAF。

为了解决这一问题，Chrome 方面已经使用了各种技术手段；包括 C++ 智能指针（如 MiraclePtr）、编译器中的静态分析、动态工具（如 C++ sanitizers ）、代码模糊器，以及一个名为  Oilpan 的 C++ 垃圾回收器。Oilpan、MiraclePtr 和基于智能指针的解决方案需要大量采用应用程序代码。

此外，谷歌还探索了另一种方法：内存隔离（memory quarantine）。基本思路是将 explicitly freed memory 放入隔离区，并且仅在达到特定安全条件时才使其可用。Chrome 安全团队在博文中总结了在 Chrome 中实验隔离和 heap scanning 的历程。

工作原理在于，用隔离和 heap scanning 保证 temporal safety 的主要思想是避免重用内存，直到证明没有更多的（悬空的）指针指向它。为了避免改变 C++ 用户代码或其语义，提供 new 和 delete 的内存分配器被拦截。

![图片](https://mmbiz.qpic.cn/mmbiz_png/dkwuWwLoRK9XzYt7xKVxyhGIkd0B7XPOsNibyfODJPY8T06V5RiaiaTWN8TKRwFI2KlOe9GCmZR5PicJXwcicOsicyiaA/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

在调用删除时，内存实际上被放入隔离区，无法再用于应用程序的后续新调用。“在某些时候触发了 heap scan，它扫描整个堆，就像垃圾回收器一样，以查找对隔离内存块的引用。那些没有从常规应用内存中获得引用的块被转移回分配器，在那里它们可以被重新用于后续的分配。”

根据介绍，谷歌的 heap scanning 由一套被命名为 StarScan（简称为 *Scan）的算法组成。他们将 *Scan 应用于渲染器进程的非托管部分，使用 Speedometer2 评估性能影响，并尝试了不同版本的 *Scan。

![图片](https://mmbiz.qpic.cn/mmbiz_png/dkwuWwLoRK9XzYt7xKVxyhGIkd0B7XPO3ouwP9mIFNRht3KnTuwDSAmm93JxmcAU6yOEZ18a450w33q7pALyDQ/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

测试结果表明，*Scan 的一个基础版本造成了 8% 的内存回归。“所有这些开销从何而来？不出所料，heap scanning 极其受 memory bound 影响，因为扫描线程必须遍历和检查整个用户内存的引用”。在进行了多方面优化之后，Speedometer2 回归从 8% 降低到了 2%。此外，有关内存消耗的测量结果则表明，渲染进程中的扫描使内存消耗减少约 12%。

MTE（内存标签扩展，Memory Tagging Extension）是 ARM v8.5A 架构上的一个新扩展，有助于检测软件内存使用中的错误；这些错误可以是 spatial errors（如 out-of-bounds accesses），也可以是 temporal errors（use-after-free）。

谷歌方面获得了一些支持 MTE 的 actual hardware，并在渲染器过程中重新进行了实验。结果表明，虽然 MTE 和 memory zeroing 会带来一些成本，但 Speedometer2 中的内存回归约为 2%。实验还表明，在 MTE 之上添加 *Scan 没有可衡量的成本。 

![图片](https://mmbiz.qpic.cn/mmbiz_png/dkwuWwLoRK9XzYt7xKVxyhGIkd0B7XPOI6phnm8W32KJtetwliaavDVqKIG8wNd378uqdO6OYW2iaaDAV6AjRZoQ/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

Chrome 安全团队最后总结称，C++ 可以编写出高性能应用程序，但需要付出安全性方面的代价。Hardware memory tagging 可以修复 C++ 的一些安全缺陷，同时保持高性能。

“我们期待在未来看到更广泛地采用 Hardware memory tagging，并建议在 Hardware memory tagging 之上使用 *Scan 来修复 C++ 的 temporary memory safety。使用的 MTE 硬件和 *Scan 的实现都是 prototypes，我们预计仍有性能优化的空间。”

相关链接：https://security.googleblog.com/2022/05/retrofitting-temporal-memory-safety-on-c.html

> 来源：OSC开源社区

- EOF -

  

![图片](https://mmbiz.qpic.cn/mmbiz_svg/SQd7RF5caa2sRkiaG4Lib8FHMVW1Ne13lrN37SiaB2ibEDF4OD31Vxh71vWXuOC2VaWME2CltDJsGdA5LnsdhdJianUR3GkoXe1Nx/640?wx_fmt=svg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

**加主页君微信，不仅C/C++技能+1**

![图片](https://mmbiz.qpic.cn/mmbiz_svg/SQd7RF5caa2sRkiaG4Lib8FHMVW1Ne13lr4b5vuiaNBnGZKzQI3kAgC4XOZVFnBxvvrXI2GOpiaH06UjrJSc4fqoPBZDKzPVRicCN/640?wx_fmt=svg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)![图片](https://mmbiz.qpic.cn/mmbiz_png/UzDNI6O6hCFBc2O6VZiaHtzQn9pYBAmTD9EaEHCDBLkxE8Pln85fKLpIy3sRib8FX0Lzoagbs8TYxC5aAgTubZyw/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

主页君日常还会在个人微信分享**C/C++开发学习资源**和**技术文章精选**，不定期分享一些**有意思的活动**、**岗位内推**以及**如何用技术做业余项目**

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/pldYwMfYJphJz37DqqcKXT6Zq6iaBZTQPRqVrSvhoULY5mS2CyAvMTnmEia6dxcs69c2TibQ8OOrVkhF5klQZ3I1g/640?wx_fmt=jpeg&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

加个微信，打开一扇窗

  

推荐阅读  点击标题可跳转

1、[探索 OS 的内存管理原理](http://mp.weixin.qq.com/s?__biz=MzAxNDI5NzEzNg==&mid=2651171440&idx=1&sn=f932790192e9897fa48613d66aac1856&chksm=80647b2fb713f239e7a511ed314b4f2cb388b2a5dc55b994421f839fd36e70072c5035385645&scene=21#wechat_redirect)

2、[内存泄漏-原因、避免以及定位](http://mp.weixin.qq.com/s?__biz=MzAxNDI5NzEzNg==&mid=2651171157&idx=1&sn=50b86af788b12d18fe6ae298064ae30d&chksm=8064780ab713f11cfcf9a44ef05073ea65aa79a876230ab0b275730f72013462ebbc5688333b&scene=21#wechat_redirect)

3、[30 张图带你领略 glibc 内存管理精髓](http://mp.weixin.qq.com/s?__biz=MzAxNDI5NzEzNg==&mid=2651170981&idx=1&sn=61be27deb019126bb32fd6d8da71d44a&chksm=806479fab713f0ec2c47394610dc0e91e7fd51d01b473566f10d9adfa3dda9a8a48bfa5d15ae&scene=21#wechat_redirect)

  

**关注『CPP开发者』**  

看精选C++技术文章 . 加C++开发者专属圈子

![](http://mmbiz.qpic.cn/mmbiz_png/pldYwMfYJpia3uWic6GbPCC1LgjBWzkBVqYrMfbfT6o9uMDnlLELGNgYDP496LvDfiaAiaOt0cZBlBWw4icAs6OHg8Q/300?wx_fmt=png&wxfrom=19)

**CPP开发者**

我们在 Github 维护着 9000+ star 的C语言/C++开发资源。日常分享 C语言 和 C++ 开发相关技术文章，每篇文章都经过精心筛选，一篇文章讲透一个知识点，让读者读有所获～

24篇原创内容

公众号

点赞和在看就是最大的支持❤️  

内存5

浏览器1

c++4

内存 · 目录

上一篇80% 的 Linux 都不懂的内存问题下一篇80% 的 Linux 都不懂的内存问题

阅读 3919

​

喜欢此内容的人还喜欢

写留言

**留言 4**

- MstnFan
    
    北京2022年6月21日
    
    赞6
    
    Chrome相当垃圾，它经常在后台运行一个report程序导致我CPU狂转。![[尴尬]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
- Mr.Renᯤ⁶ᴳ
    
    四川2022年6月21日
    
    赞1
    
    Chrome的源码不敢恭维，之前去扣代码功能点，在资源释放方面，还在错误分支里边处理，RAII机制都没用起来，我笑了
    
- Why so serious!
    
    浙江2022年6月22日
    
    赞
    
    看到说“RAII没用起来”的留言，我笑了![[微笑]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)…同时好奇你怎么会有权限看到Chrome代码（断定你不是google的人），Chromium她没有你想扣的代码吗？
    
- jarryyang
    
    上海2022年6月21日
    
    赞
    
    chrome是我红芯科技搞的呢![[愉快]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    

已无更多数据

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/pldYwMfYJpia3uWic6GbPCC1LgjBWzkBVqYrMfbfT6o9uMDnlLELGNgYDP496LvDfiaAiaOt0cZBlBWw4icAs6OHg8Q/300?wx_fmt=png&wxfrom=18)

CPP开发者

7分享2

4

写留言

**留言 4**

- MstnFan
    
    北京2022年6月21日
    
    赞6
    
    Chrome相当垃圾，它经常在后台运行一个report程序导致我CPU狂转。![[尴尬]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
- Mr.Renᯤ⁶ᴳ
    
    四川2022年6月21日
    
    赞1
    
    Chrome的源码不敢恭维，之前去扣代码功能点，在资源释放方面，还在错误分支里边处理，RAII机制都没用起来，我笑了
    
- Why so serious!
    
    浙江2022年6月22日
    
    赞
    
    看到说“RAII没用起来”的留言，我笑了![[微笑]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)…同时好奇你怎么会有权限看到Chrome代码（断定你不是google的人），Chromium她没有你想扣的代码吗？
    
- jarryyang
    
    上海2022年6月21日
    
    赞
    
    chrome是我红芯科技搞的呢![[愉快]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    

已无更多数据