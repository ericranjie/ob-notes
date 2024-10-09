# 

原创 宋宝华 Linux阅码场

_2018年05月26日 09:20_

本系列2012年的时候发表在我的blog上面，现搬到公众号

![图片](https://mmbiz.qpic.cn/mmbiz_png/Ass1lsY6byvL27ndW65dwSaR3xsSmlic4CBlACjZlRMlibzlOia1iaVALHzhJ9nSwMPEZFaXnIoibgwoZp8JYAy9b4Q/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

笔者决定，从今天开始，连载Android架构纵横谈系列。之所以叫纵横谈而不是叫别的题目，是因为整个系列是横着竖着乱弹琴，可以说是阴阳不分，黑白颠倒，望湘园里望湘园。我不谈任何一个小的点，比如启动过程、某个HAL移植、一个具体的native service或者Java service，我要谈的是横穿在其中的设计思想，因此，我谈的任何一个方面，都有可能涉及到Android从内核到应用的所有层面。

这个纵横谈既然决定乱弹，我们就不要那么严肃了，人民邮电出版社的老黄让我在《Linux设备驱动开发详解》里一直严肃着，让我装深沉，我郁闷啊，我独立的人格不为人们所理解啊。这个系列没人管，我们放开了整。《Linux设备驱动开发详解》曾经是年度10大畅销经典，这个系列怎么也是年度10大瞎扯吧？

![图片](https://mmbiz.qpic.cn/mmbiz_png/Ass1lsY6byvL27ndW65dwSaR3xsSmlic4I99RN1rA9G0I0y4WK8ZoIEhkInVzevaMIX2OA5tQia8cmH7ndKPVmog/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

今天我们要整的是Android软件架构的超强自愈能力。自愈说白了，就是不小心被人k了，不进医院，自己躺了几天，并且辅助心灵阿q精神胜利疗法， 就又活蹦乱跳了。作为一个屌丝，咱们也只能这样了。咱们像小强一样活着，不断自愈。Android 估计也是个资深屌丝啊，它可以说处处都自愈 。你想啊，一个Android，啥都要整啊，里面多少组件啊，Zygote啊，Dalvik啊，SystemServer啊，各种service和框架啊，你妹的，丈母娘要买房，老婆要买车，随时要跪搓衣板，不具备自愈能力还能混吗？

说到这里，我们要稍微严肃点，这边开始严肃认真地进行打劫活动。自愈能力，其实是电信、工控、航天等嵌入式系统的基本要求，Android 作为大型软件，挂的时候我们往往看到一个桌面退隐江湖，新的桌面马上起来，而不是直接死机了。死了个东方不败，又来个任我行啊，这就是Android的江湖规则。自愈极其重要，我们平时说的看门狗就是典型的自愈方式之一。看门狗干什么的，就是在那蹲点，情况不对比如动车要撞了，就开始“汪汪”了。没有看门狗是不行的，如果神九的软件跑飞了，不恢复回来，那我们的宇航员就变太空垃圾了。有人要问了，软件不跑飞不就行了吗？软件不跑飞，我已经失业，哪里还有机会来这里纵横乱弹琴呢？

Android里面有几个地方都有东西“蹲点”，虽然不是明确的看门狗。

第一只狗：带你去投胎

Android的第一只狗在init进程里面，init进程会通过捕获SIGCHLD得知其子进程的死亡，并据情况决定是否重启之。记住，在Linux的世界里，在子进程死亡的时候，父进程会收到一个SIGCHLD信号，而后父进程通过wait()系统调用，清理子进程僵尸。大家都知道，一个进程死亡的时候，如果它还有子进程，典型地，子进程会被托孤给init进程，这种情况非常普遍，所以任何一个Linux系统，它的init程序，少不了要做一件事情，就是反复通过wait()清理僵尸，否则Linux系统就会尸横遍野，整个一部生化危机啊有木有？Linux是个怎样残酷的世界啊，我艰于呼吸视听啊，哪里还能有什么言语？永远都是白发人送黑发人，父进程清理子进程。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/Ass1lsY6byvL27ndW65dwSaR3xsSmlic4SicYlhYPBGicL10UWnaxcuiaianrngiaaH529OQLh8DsLrhicia6aMZ3833ibQ/640?wx_fmt=jpeg&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)\
Android的init进程为SIGCHLD绑定了信号处理函数sigchld_handler()，并创建了一个socket用于接收该函数中发送的socket消息。sigchld_handler()函数只是简单的派送一个消息到该socket：

![图片](https://mmbiz.qpic.cn/mmbiz_png/Ass1lsY6byvL27ndW65dwSaR3xsSmlic4BFORF4lbj0fjT5IH4tMr3SpSuic37FDjUmvFvwbQgWNudChdRMib5KJA/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

说到这里，我们要特别点名批评一下信号啊。这位同学很不厚道啊，作为一个异步闯入的事件，经常在别的同学“工程进行中”的时候跑进来，把人吓成这样了：

![图片](https://mmbiz.qpic.cn/mmbiz_png/Ass1lsY6byvL27ndW65dwSaR3xsSmlic4ykgXR0aV2jCGibEzSA5lSye0Q5ye3Cf2I9lsg2oa65EcYkTpiaqjOSDQ/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

我在一些公司上Linux课的时候，很多人估计不记得学什么了，都还给我了。但是不晓得还记住我的一句话不，“中断是万恶之源”，一是耗油耗电，二是非法侵占。你想啊，凡是异步的都是恐怖的，都是打断正常业务逻辑的，你下X片，他给你罚款3000，这种属于乱发中断抢钱吧？信号之于用户空间多线程，正如中断之于内核，所以，你要特别注意线程与信号间的安全和死锁啊。多线程编程里面，上了信号就不是闹着好玩的。一般有线程访问临界资源时屏蔽信号、信号处理线程化等方式处理。特别要批评很多长着大脑从不想问题的同学啊，很多同学那就是一瞎写啊，应该怎么规划线程，怎么用信号那都是随心所欲啊。你看人家sigchld_handler()，干完一票就跑，那就是正确的信号处理函数伟大的游击战术啊。你千万不要在信号处理函数里搞什么会战啊。

看看init进程如何处理收到的消息呢，源代码情景分析这样的文章最恶心，你看了五年跟没有看一样，基本上不知所云，所以我们不搞那一套。中国的Linux开发者太痛苦，很多人把源代码情景分析当小说在读，一定从头读到尾，其实那个最多一新华字典，应该是参考书而非通读读物，读完了，除了精神崩溃以外，就是精神彻底崩溃，天天让你背新华字典你还活地下去不?

init在这里接收消息并进一步处理：

![图片](https://mmbiz.qpic.cn/mmbiz_png/Ass1lsY6byvL27ndW65dwSaR3xsSmlic4ib6mGuDgcFDicEyDNxEoicXGgJGiac3fjVPp2v9XfqS2eIl5NP5Vibths0w/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

好吧，进去看看wait_for_one_process()吧，请注意我不是要搞情景分析，我只抓一点点代码出来。情景分析之类的恐怖片看多了，会影响三观。各位同学在竖立正确三观的前提下，请自行研读android/system/core/init。所谓正确的三观，就是不是为了读代码而读代码，而是为了分析问题而找代码啃。情景分析为什么三观不正，是因为完全不先提出问题，分析问题，最后研读代码，而是上来一锤子拿代码砸晕你先。绝望啊！

![图片](https://mmbiz.qpic.cn/mmbiz_png/Ass1lsY6byvL27ndW65dwSaR3xsSmlic4RLx3rYTxfALiagfkhFGbFUvvO0BOIgT3TUibuuORB7NRrriczFsBz7eRA/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

![图片](https://mmbiz.qpic.cn/mmbiz_png/Ass1lsY6byvL27ndW65dwSaR3xsSmlic4Z6q5z4EVXblusgSlJI4Eo21eAhjVBBSLlhBfPca6zaqLl8Q3z3sQxQ/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

特别留意加中文注释的几个地方:

**1. 无主游魂**

挂的时候有“untracked pid %d exited”打印的进程属于init的子进程，但是并没有在 init.rc里面注册service。典型地，那些先父仙游后被托孤给init的进程！实在是太悲惨了，这样的进程，直到死都还没户口，实在是屌丝中的蚕丝。看到这种进程，读书人一声长叹啊！咱们沪上海漂估计就是这种了。谁叫你不是土生土长的上海人fork出来的呢？

土生土长的上海人，有户口的service是这样的：

![图片](https://mmbiz.qpic.cn/mmbiz_png/Ass1lsY6byvL27ndW65dwSaR3xsSmlic4mQTR2jc2pG6lrwSptrrwyIDMtHRNl5vYFWr29bOllU4rNnRbXnnQlg/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

**2. 拿一次性签证的ONESHOT service**

untracked的进程像没有户口的上海工程师，ONESHOT的进程像拿single entry签证赴美的人，搞一把没第二次机会除非再签。需要在service下面加上oneshot，这样的service挂了就挂了，不会想着投胎，正如：

![图片](https://mmbiz.qpic.cn/mmbiz_png/Ass1lsY6byvL27ndW65dwSaR3xsSmlic4w0ucbPPj5nF5cBbwe0OylsgLtSiajCYOOPV7aDfJhjZ6ice0QjlBic9Zw/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

此外，如果service被加了“disabled”标记，也不会自动启动而需要显示地启动。

**3. 可以投胎的女鬼**

其他的进程就爽了，因为被“svc->flags |= SVC_RESTARTING”盖章了，死了会重新投胎， 下次init执行到restart_processes()的时候就可以重启之，而且之前顺带还可以在投胎的时候执行点什么，如想投胎个好人家什么、定点空投到某人等，就像聊斋之小谢、聊斋之鲁公女等。咱们就不要像蒲松龄同学那么yy了，美丽女鬼投胎来报答我们的机率不高了，好像我们前世没做过什么好事？不过我们一直在期待！

![图片](https://mmbiz.qpic.cn/mmbiz_png/Ass1lsY6byvL27ndW65dwSaR3xsSmlic4OTDlplcVstmL9VuFScA8WDNcaebvrvuhiawvLhQk5lnFutEziaR2By0w/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

这个动作通过onrestart指定：

![图片](https://mmbiz.qpic.cn/mmbiz_png/Ass1lsY6byvL27ndW65dwSaR3xsSmlic4WhmdeuL5aEibUskJfX2wDVJEUpUiaOLibXuO3CbhwbbzyCOBaIXmdqeFw/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

在这里，我们要哀叹一下, 兰若寺真不是个好地方，Android就好多了，只要你有户口，又不是ONESHOT，你就可以马上重新投胎，聂小倩啊聂小倩，你真是死不逢地啊！于是你造就了一段荡气回肠的流传千古的爱恋。张国荣《倩女幽魂》，我最爱的电影之一。

**4. 压死Android的最后一根稻草**

如果一个service是critical的，而它又在短时间内反复挂，restart后又总是夭折，我们很可能不能再让它这么痛苦下去了，唯一的方法就是让它解脱，于是整个系统重启。这样的service包括：

![图片](https://mmbiz.qpic.cn/mmbiz_png/Ass1lsY6byvL27ndW65dwSaR3xsSmlic469H1v5w5icaknbkuE6Vw8syIsdBFrJJkXn7KaYqF4gQZ1NicF8e9pWNA/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

Android，第一只狗，就是这样负责一个挂掉的service的重启的。下集我们讲第二只狗：生死与共的Zygote与SystemServer。

今天累了，欲知后事如何，请听下回分解。我们赶着芒果台新版《天涯明月刀》前一天开始播放Android架构纵横谈，也算是抢一点收视率。老版《天涯明月刀》，我的童年，那些逝去的年华啊！

谨以本回，献给看《天涯明月刀》长大的一代人！如今我们已经而立或不惑，我们逝去的青春，就像一首歌。

古刹空潭犹自远，长阶醉卧知谁见。 \
天涯明月箫声断，刀映孤星，已是离人倦。

(请听下回分解)

Linuxer是专业的Linux及系统软件技术交流社区，Linux系统人才培养基地，企业和Linux人才的连接枢纽。

查看我们精华技术文章请移步:

[Linuxer精华文章汇总](http://mp.weixin.qq.com/s?__biz=MzAwMDUwNDgxOA==&mid=2652664500&idx=1&sn=5f4c1e85cf5a4c38d3a1b55ac67fc75e&chksm=810f3429b678bd3f51a2581adb1bb581ad6f62adcbdfb2fa4ed77eaef0154cd34a4da3f87460&scene=21#wechat_redirect)

求职招聘请移步:

[Linuxer: 连接企业和Linux人才的platform总线](http://mp.weixin.qq.com/s?__biz=MzAwMDUwNDgxOA==&mid=2652664416&idx=1&sn=072133a70fb12a7cf932d861ea5bce7b&chksm=810f34fdb678bdeb4f2d53ab2d0a9fd4e80e2ff0355bd6f80447a0d95a9e826099b543743cb4&scene=21#wechat_redirect)

扫描二维码关注我们

![图片](https://mmbiz.qpic.cn/mmbiz_png/Ass1lsY6bytaJia8v1tBt03P9TibIPl0FSJfG7PJ7IpicBic79v5fm1IWzKL8py26ANMH477bjpNuozvEhiaSaLG60g/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

如果觉得好，请

转发

转发

转发

阅读 2069

​

喜欢此内容的人还喜欢

riscv有没有摸着arm过河

我关注的号

Linux阅码场

不喜欢

不看的原因

确定

- 内容低质
- 不看此公众号内容

![](https://mmbiz.qpic.cn/mmbiz_jpg/YHBSoNHqDiaEGicBCMzoKdZOuBI57fFD37NlglPGHJYk8j6ddbuTwISSC7yz3bVEfVQ3CNG1Q3C5mMydeo5t9S0A/0?wx_fmt=jpeg&tp=wxpic)

【课程】8小时学透ARM体系架构

我关注的号

Linux阅码场

不喜欢

不看的原因

确定

- 内容低质
- 不看此公众号内容

![](https://mmbiz.qpic.cn/mmbiz_jpg/YHBSoNHqDiaFzlibHfu5O9icgSthZbWu1Scq9Lu1z7DREEpxlJye7k3AbZQ4PqBOStpECQycNpqB5Pc8eWoFibud5A/0?wx_fmt=jpeg&tp=wxpic)

《深入剖析Linux内核反向映射机制》在线视频课程

我关注的号

Linux阅码场

不喜欢

不看的原因

确定

- 内容低质
- 不看此公众号内容

![](https://mmbiz.qpic.cn/mmbiz_jpg/YHBSoNHqDiaEGicBCMzoKdZOuBI57fFD37C96USYreViablWZf6rpXWbq6SGiaOINPBWVEM4ibdZYiajQhmzzlYCW4gQ/0?wx_fmt=jpeg&tp=wxpic)

写留言

**留言 8**

- 喻青

  2018年5月26日

  赞

  楼主的博客地址可否粘贴下，小弟拜读

  Linux阅码场

  作者2018年5月26日

  赞6

  我们现在文章都是第一时间在linuxer公众号发表。如果你看博客，在https://blog.csdn.net/21cnbao

- 旭日清风

  2018年5月26日

  赞4

  "所谓正确的三观，就是不是为了读代码而读代码，而是为了分析问题而找代码啃。" 受教了宋老师![[抱拳]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

- 黃大椿

  2018年5月26日

  赞2

  寶華現在改行寫小說了？

- 李凡辉

  2018年5月26日

  赞2

  严肃的文章虽登得了大雅之堂，但纵横戏说的文章更容易深入人心，所以与Linux 驱动那本书相比，还是更喜欢本篇这种纵横谈。

- 第九天魔王

  2018年5月26日

  赞2

  当年刚发表在博客上就拜读过，让我对父子进程之间的关系豁然开朗，再也没有忘记。

- 浮尘

  2018年5月29日

  赞1

  以后像大神学习，不搞源码情景分析。

- KylinShui

  2018年5月26日

  赞1

  比喻很有意思

- 胡先浪

  2018年5月27日

  赞

  娓娓道来，受教![[憨笑]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

已无更多数据

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/YHBSoNHqDiaH5V6ycE2TlM9McXhAbaV9aUBaTAibVhLibXH1rL4JTMb2eoQfdCicTm4wycOsopO82jG3szvYF3mTJw/300?wx_fmt=png&wxfrom=18)

Linux阅码场

181在看

8

写留言

**留言 8**

- 喻青

  2018年5月26日

  赞

  楼主的博客地址可否粘贴下，小弟拜读

  Linux阅码场

  作者2018年5月26日

  赞6

  我们现在文章都是第一时间在linuxer公众号发表。如果你看博客，在https://blog.csdn.net/21cnbao

- 旭日清风

  2018年5月26日

  赞4

  "所谓正确的三观，就是不是为了读代码而读代码，而是为了分析问题而找代码啃。" 受教了宋老师![[抱拳]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

- 黃大椿

  2018年5月26日

  赞2

  寶華現在改行寫小說了？

- 李凡辉

  2018年5月26日

  赞2

  严肃的文章虽登得了大雅之堂，但纵横戏说的文章更容易深入人心，所以与Linux 驱动那本书相比，还是更喜欢本篇这种纵横谈。

- 第九天魔王

  2018年5月26日

  赞2

  当年刚发表在博客上就拜读过，让我对父子进程之间的关系豁然开朗，再也没有忘记。

- 浮尘

  2018年5月29日

  赞1

  以后像大神学习，不搞源码情景分析。

- KylinShui

  2018年5月26日

  赞1

  比喻很有意思

- 胡先浪

  2018年5月27日

  赞

  娓娓道来，受教![[憨笑]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

已无更多数据
