
CppGuide

 _2022年07月11日 14:44_

  

![](https://res.wx.qq.com/op_res/NN_GToMiIjsXzgPzF9-74ZzwR3cA9-fv3o9eWo8f5gQWqx71CmGlY8kFxuIxZaG0TB1bFeMCmh1DGN_pWMRg0A)

**帐号已迁移**

公众号

![Image](https://mmbiz.qpic.cn/mmbiz_jpg/KyXfCrME6UIwibrSgYXWO0eFqHzSOIqC3x3bTErdtPEhUTdbYo5af9KjUwbkdevibJpEBsBhiae5npUEtKsRt5j8g/640?wx_fmt=jpeg&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

  

  

https://www.zhihu.com/question/290767285/answer/1200063036

  

作者：龙泉寺扫地僧（首席浏览器架构吹牛师，全球最小chromium内核--miniblink作者），侵删

我们来看开源的chromium，这货确实相当相当的复杂。源码拉下来就有十多G。  

  

我们不禁好奇，chromium到底有哪些玩意，为啥平时感觉只是显示个网页、几句HTML而已，怎么会需要这么多代码？

  

第一眼从目录结构上，chromium包含这些东西：

  

**base**，通用代码，基础组件，包含字符串、文件、线程、消息队列等工具类集合。

  

**cc**，Chromium compositor 的缩写，负责渲染合成。

  

**chrome**，Chromium 浏览器外壳实现。

  

**content**，多进程沙盒浏览器的核心代码，管理进程架构和线程架构。

  

**gpu**，OpenGL 封装代码，包含 CommandBuffer 和 OpenGL 兼容性支持等。

  

**net**，网络栈实现。

  

**ipc**，进程间消息通信实现。

  

**media**，多媒体封装代码，包含了媒体内容捕获和播放的组件集合。

  

**mojo**，类似于 Android 的 AIDL，提供了跨语言（C++ / Java / JavaScript）跨平台的进程间对象（Object）通信机制；。

  

**skia**，图形库，这里存放的是 Chromium 对 skia 的 配置和扩展代码，另有 third_party/skia 目录存放原生的 skia 代码。

  

**third_party**，网页排版引擎。第三方库

  

**ui**，UI 框架。

  

**v8**，V8 JavaScript 引擎库。

  

看起来还好吧？但实际上，这里面每一个展开来讲，都是一本厚厚的工具书的容量。

  

比如net，看起来只是个网络库，然而里面包含主机解析，cookies，网络改变探测，SSL，资源缓存，ftp，HTTP， OCSP实现，代理 (SOCKS和HTTP) 配置，解析，脚本获取（包括各种不同系统下实现），QUIC，socket池，SPDY，WebSockets……里面每一项展开来讲，就是一本书。

  

v8层，看起来功能很单一，只是实现一下js嘛，但里面包括字节码解析器，JIT 编译器，多代GC，inspector (调试支持)，内存和 CPU 的 profiler（性能统计），WebAssembly 支持，两种 post-mortem diagnostics 的支持，启动快照，代码缓存、代码热点分析……里面每一项展开来讲，又是一本书，还是难坑的编译原理和优化方向。

  

Skia，看起来只是个图形库嘛，用点画出各种图。然而里面包括十几种矢量的绘制，文字绘制、GPU加速、矢量的指令录制以及回放（还要能支持线程安全）、各种图像格式的编解码、pdf的生成（这个是个隐藏的很深的功能，但很有趣。Skia支持把矢量图绘制成pdf）、GPU渲染优化（即以上部分功能需要用gpu来渲染）……里面每项展开来讲，又是一本书。

  

另外值得一提的是，skia是谷歌收购的。不知道谷歌是觉得自己没实力做，还是太费功夫。总之谷歌选择了直接买别人的代码来完成这些功能。

  

ui，看起来只是一套UI 框架嘛。然而chromium需要一套全平台适配的ui库，还要能支持gpu加速。不过可惜的是里面没实现richedit。ui库的设计，深入来做，其实可以说又是个浏览器了。

  

等一下，以上这些，看起来只是浏览器的外层。我们最关心的网页排版呢？这个难道不是浏览器的核心嘛！

  

是的，神奇的是，chromium把排版引擎blink放到了third_party下，而且架构上真的当成了一个第三方库一样对待。

  

据谷歌的员工说，这是历史原因……好吧姑且信了。然而这个第三方库，成了当之无愧的最复杂，功能最重要的第三方库。

  

blink的工作包括：

  

- 实现web平台的规范（例如，HTML标准），包括DOM，CSS和Web IDL
    
- 配合V8运行JavaScript
    
- 从底层网络堆栈请求资源
    
- 构建DOM树
    
- 计算样式和布局
    
- 请求chrome compositor（上文提到的cc层）并绘制图形。
    

  

说起来简单。看一下现在的HTML、CSS规范，各种细节加起来……有快上万页。

  

除了chromium layout组、firefox的开发人员等，我想没几个人会去仔细阅读并一个个的实现这些规范吧。光是看目录和文字描述，就头大了，更别说要完整的实现出来。

  

往往一个简单的display:gird\flex背后就是庞大复杂的计算，而且还要充分考虑性能上如何优化，滚动时如何更快的展示…

  

另外排版还需要支持世界各国的奇奇怪怪的文字。例如从右往左写、规则复杂无比阿拉伯文。相比之下，汉字这种方块字的排版简直就是弟弟。还有各种奇怪的unicode字符。

  

![Image](https://mmbiz.qpic.cn/mmbiz_png/KyXfCrME6UIwibrSgYXWO0eFqHzSOIqC354H6ic29QAWtgJACDDtD4kBglgKBQO5vyasvn3ZnndQtgeibsYdJWsBw/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

  

怎么能处理好这些字符和语言，并配合几千页的html、css排版规则正确显示出来……这是个极度烧脑的事情。

  

我们再从排版这个大泥坑里跳出来看看外面别的东西。这时候你会发现……外面的泥坑好像更大。随便说几个，比如：

  

**多进程框架**。嗯，你需要更多的进程来渲染更多的网页，这样才能崩溃了也不影响其他网页。

  

注意，chromium把渲染排版放在渲染进程，但绘制到窗口又是主进程。这里面少不了各种跨进程通信、同步。对于代码的编写以及调试，是个很考验编程功底的事情。

  

**webrtc**。网络视频相关。又是一个被收购的库。关于webrtc，你需要知道它能实现多人实时语音、降噪、网络传输视频、摄像头的捕获，音频算法实现（比如 fft），视频算法实现（比如 h264 协议格式）， Socket、线程、锁等基础库（是的，webrtc也造了套自己的轮子）。又是个庞大的组件。

  

**密码管理、下载管理、扩展管理**。

  

一套调度整个**多进程框架以及blink的核心层**。在chromium被称之为content层，负责处理一切繁琐的细节。例如各种系统、平台的鼠标键盘消息派发，历史栈（前进后退），页面缓存。

  

**沙箱机制**。负责隔离以及降低子进程的权限。沙箱的实现上，在不同系统做了诸多hook操作。

  

**chrome相关的外壳及应用**。例如我们常见的标题栏、url栏，webui如设置页、历史记录页。对，其实chrome单词的原意就是这个。

  

**Clound_Print**，谷歌云打印相关，提供谷歌浏览器页面预览打印清单。

  

**Courgetter**，谷歌提供的二进制文件对比核心算法，用于比较不同版本的二进制差异。谷歌为了方便升级，搞了套升级策略和算法。

  

**神奇的syzygy优化**。是的，谷歌也嫌chrome太大了、加载太慢了。于是他们开发了一套工具链，优化重排布PE二进制文件的算法来达到优化程序。Chrome浏览器应用了Syzygy优化之后，程序冷启动的页面调度（paging traffic）优化了80%，加载的Image的Working Set优化了40%。简单的说，谷歌为了优化启动性能，从编译器上对exe、dll开始做手脚了。

  

**Media**，Chrome的多媒体模块，支持音频播放和录音等功能。这里用到了ffmpeg。但在ffmpeg外，为了和blink配合，又是包裹了厚厚的一层，用来处理好渲染管线。另外MSE API也花了不少功夫。

  

**swiftshader**。很有趣的一个模块，用纯软件的代码，完整实现了opengl的接口。可以在没有硬件加速的机器上跑起opengl。也是个庞大的库，而且也是被收购的。看起来谷歌对图形学方面的很多工程似乎不擅长？还是不想觉得应该交给更专业的团队去做。

  

**gn、gyp、ninja**。chromium为了更方便的管理编译，自己撸了三套轮子。类似makefile、cmake，然后底层调用ninja再到vs或者clang负责具体编译其他的点还有很多很多，以后想到了再补充。

  

总之，以上随意一个点，要正确的实现，都是一个团队的工作量，都可以写成一本书。然而chromium把他们全部实现了，而且还在不停的加入新的功能。

  

看到这里，大家应该明白为啥强如微软，也放弃维护他们自己的浏览器内核了。因为需要投入的人力财力实在是太恐怖了。chromium团队，光是开发人员，都已经上千了。

  

假如每个人员年薪是100w 人民币，持续投入十年，这个支出就是几十亿，这还不算周边的测试、产品、UI。

  

最关键的是，就算微软愿意投入十亿，能保证做到chromium相同的功能吗？就算能做到相同的功能，还不是另外一套chromium，能做出其他优势吗？

  

于是最后微软也放弃了，干脆直接从开源的chromium上改起，把微软需要的功能融入chromium。

  

所以，chromium的霸权就是这么来的。看似开源免费，实则把所有开发者和对手，紧紧的捆绑在自己周围。

  

好了，现在吐槽开始了。

  

chromium称霸浏览器界以来，看起来开源，谁都可以拿去改。然而比起它的前辈，我觉得从“道德”上，chromium要“差”很多。

  

最让我受不了的一点是，chromium在无尽的往里面塞功能的时候，很少想过是否别人可以轻易的移除它们。

  

chromium代码号称模块化、高内聚低耦合，然而如果你想砍掉一些不需要的东西，对不起，没有宏控制，手动删代码吧。有几个人能有精力一点点的去掉里面这些繁琐的功能呢？

  

这就导致一个问题，需要chromium某一部分功能的人，必须被强塞进一堆谷歌认为你需要的东西。对比之下，为啥我说比起chromium的前辈要差很多呢，其实我指的正是webkit。webkit最让人欣赏的一点就是它在专注实现内核的同时，大部分功能都是可“拆卸”的。有宏可以关闭。甚至连svg这种排版上的小功能都有宏可以关闭。

  

而chromium，如果我需要排版、音视频，但不需要多进程呢？如果我需要音视频但不需要webrtc呢？对不起，谷歌没这考虑。要带就全都带上吧，不带你自己砍代码吧。

  

等你辛苦几个星期砍完代码，谷歌告诉你，我都更新了2、3个版本啦，你要不要更新下代码？哦？新版本chromium架构又大改了？哭了……

  

这也就造成现在基于chromium的一堆开发框架，如electron、cef、nwjs，全都动不动100多M的大小。因为从chromium的框架设计上，就很难把那些极其庞大复杂的细节功能排除掉。

  

这些功能对于谷歌作为一个浏览器来说，当然是必要的。然而回到本问题，“浏览器内核”真的需要这么复杂吗？

  

浏览器需要这么复杂，这是真的；然而作为一个浏览器内核提供给一些sdk给别人用，也需要这么复杂吗？我们用electron写一套进供销管理系统的客户端，你会需要带上几十M的webrtc、webgl、多媒体播放、天城文支持吗？

  

最后打破这个疑问的，是我很多年后进入了QQ浏览器的移动端组（其实就是x5内核，微信上被大家吐槽最多的那个）。当年的x5内核其实是基于webkit改造的。从chromium回到webkit，我突然有种豁然开朗的感觉。

  

回到问题开头，从浏览器内核的角度，其实没那么复杂，只要做好网络、排版、渲染，就足以应付大部分使用场景了。

  

这让我想起浏览器早期年代，群雄争霸的时代，那时候浏览器内核很小。从几百K到几M的浏览器都有。我记得早年的移动设备上跑的浏览器，css支持的都不好，不过特别小巧，有的才几百K而已。

  

然而后来组里架构调整，x5内核为了跟上时代，从webkit切回了chromium（也是因为被骂了太多了，当时x5号称移动端IE6，做过微信相关开发的人应该深有体会）。

  

对chromium深恶痛绝的我，当时有了一个大胆的想法。把排版引擎blink（也就是webkit在chromium里的继承者）重新从chromium里剥离出来，再补上一些周边的设施、组件，再次成为一个完整独立的浏览器内核。

  

当然我还是有自知之明的。一个浏览器内核而已，不可能实现chromium同样的功能。但能把排版、渲染、网络、视频实现，就差不多了。

  

其实我也不是想做个浏览器，而是想专注内核这块，做一个提供给第三方app嵌入的内核。作为一个内核，其实不需要上面讲的所有那一大堆功能。比如，网络层，大部分人不需要什么网络改变探测，ftp，OCSP实现，代理配置、解析、脚本获取，QUIC，socket池，SPDY什么的。大部分人仅仅需要一个http的实现，可以拉取到服务器资源。

  

我用300k的curl代替了十余M的chromium net库，并工作良好。少了的功能可以用插件形式补上嘛。一些和排版渲染无关的功能，我都打算做成插件。例如音视频、webgl、webrtc、多进程等。

  

目前这个项目已经撸了5年了，开源在github上，提供C接口方便其他语言调用。整个编译出来就是一个1-20m左右的dll（视编译选项而定），甚至还包含了一个electron的精简实现：

  

https://github.com/weolar/miniblink49。

![](https://res.wx.qq.com/op_res/NN_GToMiIjsXzgPzF9-74ZzwR3cA9-fv3o9eWo8f5gQWqx71CmGlY8kFxuIxZaG0TB1bFeMCmh1DGN_pWMRg0A)

**帐号已迁移**

公众号

  

**推荐阅读：**

- [我们说 TCP 是流式协议究竟意味着什么？](http://mp.weixin.qq.com/s?__biz=MzU2MTkwMTE4Nw==&mid=2247495748&idx=1&sn=82387a846f5ad456fa5167a3a7090659&chksm=fc730ba8cb0482be2291be4e91bded252b9471a9251e068b92c10c042b6dca72ae0b2e617b39&scene=21#wechat_redirect)
    
- [一个 WebSocket 服务器是如何开发出来的？](http://mp.weixin.qq.com/s?__biz=MzU2MTkwMTE4Nw==&mid=2247496466&idx=2&sn=160f286eadaf09371a6a1548ceb2aee8&chksm=fc7308fecb0481e8c52f28aec7dc1c089506cc2913f2a663320572114938ca69d174ecbafc21&scene=21#wechat_redirect)
    
- [从零实现一个 http 服务器](http://mp.weixin.qq.com/s?__biz=MzU2MTkwMTE4Nw==&mid=2247496370&idx=1&sn=0d390265121bb14bb5d6cb86319415bb&chksm=fc73095ecb0480480409b61ffdebe80d49a4bd9d86817cb93bcd6096ed67f5790dfe34be9905&scene=21#wechat_redirect)
    
- [使用 epoll 时需要将 socket 设为非阻塞吗？](http://mp.weixin.qq.com/s?__biz=MzU2MTkwMTE4Nw==&mid=2247498075&idx=2&sn=c6e10bb86520405093758570d2050d07&chksm=fc7302b7cb048ba1fe0a6686097534d72bc1b4dc19446b7d9d289d572ab931c47f4a7795953b&scene=21#wechat_redirect)
    
- [Linux 的 epoll 使用 LT + 非阻塞 IO 和 ET + 非阻塞 IO 有效率上的区别吗？](http://mp.weixin.qq.com/s?__biz=MzU2MTkwMTE4Nw==&mid=2247495528&idx=2&sn=ed38afc395bd25b390c3150d7682704e&chksm=fc731484cb049d92fa0f90f2c02ea74aaa036da448c19d6b532d96d5a8d0d4ff4dfe24d4b02f&scene=21#wechat_redirect)
    

## 如果想加入 **高质量技术交流群** 进行交流，

## 可以先加我微信 **easy_coder**，备注"加微信群"，我拉你入群，

## 备注不对不加哦

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

  

如有收获，点个在看，诚挚感谢![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

Reads 2269

​

Comment

**留言 7**

- 傲慢才是生存的障碍
    
    广东2022年7月11日
    
    Like2
    
    虽然看不懂，但是能感觉很厉害。
    
- 纵横
    
    四川2022年7月11日
    
    Like1
    
    都是一群有想法的人，有想法且去做了，且做成了，都是厉害的人。
    
- 勇者
    
    河北2022年7月11日
    
    Like
    
    张老师，哪天讲讲云原生方面的技术呢？或者互联网后面可能发展的技术方向方面的文章，谢谢！
    
- 广东2022年7月11日
    
    Like
    
    中学生来膜拜大佬 mini_blink比原版chrome不知好用多少
    
- 俊俊
    
    江苏2022年7月11日
    
    Like
    
    扫地僧确实牛逼，他的项目我赞助了1000![[呲牙]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
    CppGuide
    
    Author2022年7月12日
    
    Like
    
    ![[强]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
- 李斌
    
    上海2022年7月11日
    
    Like
    
    不明觉厉，底层原理都是设计巧妙，能学到的东西确实多
    

已无更多数据

[](javacript:;)

![](http://mmbiz.qpic.cn/sz_mmbiz_png/GSweNIrkicYvM1mIwPctlYONEDKJwUfRZ57uAkVR59MpX1cVnmmnyPZ5O9OCuys78Sy6fOEncwfgWpgCo9Tibeag/300?wx_fmt=png&wxfrom=18)

CppGuide

17Share11

7

Comment

**留言 7**

- 傲慢才是生存的障碍
    
    广东2022年7月11日
    
    Like2
    
    虽然看不懂，但是能感觉很厉害。
    
- 纵横
    
    四川2022年7月11日
    
    Like1
    
    都是一群有想法的人，有想法且去做了，且做成了，都是厉害的人。
    
- 勇者
    
    河北2022年7月11日
    
    Like
    
    张老师，哪天讲讲云原生方面的技术呢？或者互联网后面可能发展的技术方向方面的文章，谢谢！
    
- 广东2022年7月11日
    
    Like
    
    中学生来膜拜大佬 mini_blink比原版chrome不知好用多少
    
- 俊俊
    
    江苏2022年7月11日
    
    Like
    
    扫地僧确实牛逼，他的项目我赞助了1000![[呲牙]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
    CppGuide
    
    Author2022年7月12日
    
    Like
    
    ![[强]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
- 李斌
    
    上海2022年7月11日
    
    Like
    
    不明觉厉，底层原理都是设计巧妙，能学到的东西确实多
    

已无更多数据