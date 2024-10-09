# 

原创 程序喵大人 程序喵大人

_2022年02月15日 08:40_

今天，想聊聊workflow这个开源项目。

关于workflow，我之前特意写过一篇文章【[推荐学习这个C++开源项目](http://mp.weixin.qq.com/s?__biz=MzkyODE5NjU2Mw==&mid=2247487891&idx=1&sn=46c64de685c49e3c33790cc8e37e308f&chksm=c21d232ff56aaa39452b82b8367eeb2afe815d69b3200465b238aff39f3427b964a6ca69940b&scene=21#wechat_redirect)】。

今天还是想再啰嗦啰嗦，因为自己这一年也在带团队从0到1做项目，需要负责整个项目的架构设计、接口设计、模块划分等工作。

做了一段时间后再回过头复盘一下，深知架构设计、接口设计的重要性，也感受到了架构设计的困难程度，编码和设计相比，真的容易的多了。

然后自己又回头来研究了一下workflow，想着学习下其他项目的设计理念，随着自己研究的越来越深入，越来越感觉它的高端，对外暴露特别简单的接口却能完成非常复杂的功能。

上篇文章是基础篇，主要向大家普及一下workflow的特点和作用，感兴趣的朋友可以移步到那里哈。

本篇文章是进阶篇，主要就是想介绍下workflow的任务模型，其他的框架一般只能处理普通的网络通信，而workflow却特别适用于通信与计算关系很复杂的应用。其实我最感兴趣的是它的内存管理机制，下面也会详细介绍。

![图片](https://mmbiz.qpic.cn/mmbiz_png/ULKUSjXeQbxecm8xLaBwx8No4ULvOhia5lDT50uBLQU87wqzJX7EE7o3hWpXrweY8ocIujHpI3BYcaic8jaAhKCQ/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

![图片](https://mmbiz.qpic.cn/mmbiz_png/ibcmMMRWnQqgnnluehp1gvTmXu91avx6l8eAeIBSteQIq08WRE1KCqVJmw2jVQIct7QW7kwIicRJCWnahQbnodSQ/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

优秀的系统设计

![图片](https://mmbiz.qpic.cn/mmbiz_png/ibkh2ApF4n7oeicic4eMUb4QVp7yic4StzhlZ09EjaIJVuCHFQW7sg1QAeNBU8XIFyY3piaBicCO4xN1dgoXFeRW0VHQ/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/HRE5ZE2T1usI7mNYIEDqJCqIaXZecP68qgGQ9Vg7ibW1bOaGP05zOsI6iaII4ueX2uemvJlm9Q0B9T6ibdYN3ibexQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

在作者的设计理念中：程序 = 协议 + 算法 + 任务流。

\*\*协议：\*\*就是指通用的网络协议，比如http、redis等，当然还可以自定义网络协议，这里只需要提供序列化和反序列化函数就可以达到想要的效果。

\*\*算法：\*\*workflow提供了一些通用的算法，比如sort、merge、reduce等，当然还可以自定义算法，用过C++ STL的朋友应该都知道如何自定义算法吧，在workflow中，任何复杂的计算都应该包装成算法。

\*\*任务流：\*\*我认为这应该就是整个系统设计的核心，通过任务流来抽象封装实际的业务逻辑，就是把开发好的协议和算法组成一个任务流程图，然后调度执行这个图。

![图片](https://mmbiz.qpic.cn/mmbiz_png/ULKUSjXeQbxecm8xLaBwx8No4ULvOhia5lDT50uBLQU87wqzJX7EE7o3hWpXrweY8ocIujHpI3BYcaic8jaAhKCQ/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

![图片](https://mmbiz.qpic.cn/mmbiz_png/ibcmMMRWnQqgnnluehp1gvTmXu91avx6l8eAeIBSteQIq08WRE1KCqVJmw2jVQIct7QW7kwIicRJCWnahQbnodSQ/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

任务流

![图片](https://mmbiz.qpic.cn/mmbiz_png/ibkh2ApF4n7oeicic4eMUb4QVp7yic4StzhlZ09EjaIJVuCHFQW7sg1QAeNBU8XIFyY3piaBicCO4xN1dgoXFeRW0VHQ/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/HRE5ZE2T1usI7mNYIEDqJCqIaXZecP68qgGQ9Vg7ibW1bOaGP05zOsI6iaII4ueX2uemvJlm9Q0B9T6ibdYN3ibexQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

这里多聊聊任务流，在workflow中，一切业务逻辑皆是任务，多个任务会组成任务流，任务流可组成图，这个图可能是串联图，可能是并联图，也有可能是串并联图，类似于这种：

![图片](https://mmbiz.qpic.cn/mmbiz_png/JeibBY5FJRBGc1KoSp59UoeSokw1YoSjpO99WJlu3CIMb46Ijk1ngujD2PSaoaMoN7KjLW8MaVl4nJXf6drFk5A/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

也可能是这种复杂的DAG图：

![图片](https://mmbiz.qpic.cn/mmbiz_png/JeibBY5FJRBGc1KoSp59UoeSokw1YoSjpalHPbJcvhrX4cWxTJOsVicY4WayY6YFhqcUbcxoycVU3aIEKnoAW5vg/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

图的层次结构可以由用户自定义，框架也是支持动态地创建任务流。

引用作者的一句话：

![图片](https://mmbiz.qpic.cn/mmbiz_png/NrhDAEo5XibRNaJjoxKYNX9l2t40HvX9IZPib1wwN68aLCTPN0NrNdQ6ILTN0Me9d8z7K8hWowA4Em783Edl5vHw/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

![图片](https://mmbiz.qpic.cn/mmbiz_png/I4HHac4BPfXGJlbMBCs8sEk48Wj6E3NJib5Z8lx6SBM2INnF39aIB2E9mzibvyl5lhZhvwqzsk1G2GzCzHalKQ6g/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

如果把业务逻辑想象成用设计好的电子元件搭建电路，那么每个电子元件内部可能又是一个复杂电路。

workflow对任务做了很好的抽象和封装。整个系统包含6种基础任务：通讯、文件IO、CPU、定时器、计数器。

workflow提供了任务工厂，所有的任务都由任务工厂产生，并且会自动回收。

大多数情况下，通过任务工厂创建的任务都是一个复合任务，但用户并不感知。例如一个http请求，可能包含很多次异步过程（DNS，重定向），内部有很多复杂的任务，但对用户来讲，这就是一次简单的通信任务。

哪有什么岁月静好，只不过是有人替你负重前行。workflow的一大特点就是接口暴露的特别简洁，非常方便用户的接入，外部接入如此简单，肯定是将很多组合的逻辑都放在了内部，但其实workflow项目内部代码结构层次非常简洁清晰，感兴趣的朋友可以自己研究研究哈。

![图片](https://mmbiz.qpic.cn/mmbiz_png/ULKUSjXeQbxecm8xLaBwx8No4ULvOhia5lDT50uBLQU87wqzJX7EE7o3hWpXrweY8ocIujHpI3BYcaic8jaAhKCQ/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

![图片](https://mmbiz.qpic.cn/mmbiz_png/ibcmMMRWnQqgnnluehp1gvTmXu91avx6l8eAeIBSteQIq08WRE1KCqVJmw2jVQIct7QW7kwIicRJCWnahQbnodSQ/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

内存管理机制

![图片](https://mmbiz.qpic.cn/mmbiz_png/ibkh2ApF4n7oeicic4eMUb4QVp7yic4StzhlZ09EjaIJVuCHFQW7sg1QAeNBU8XIFyY3piaBicCO4xN1dgoXFeRW0VHQ/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/HRE5ZE2T1usI7mNYIEDqJCqIaXZecP68qgGQ9Vg7ibW1bOaGP05zOsI6iaII4ueX2uemvJlm9Q0B9T6ibdYN3ibexQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

还有就是项目的内存管理机制，workflow整个项目都遵循着谁申请谁释放的原则，内部申请的内存不需要外部显式释放，内部会自动回收内存。

而且整个项目都没有使用shared_ptr，那多个对象共同使用同一块裸内存，workflow是怎么处理的呢？

内部为这种需要共享的对象各自维护一个引用计数，手动incref和decref，至于为什么要这样做，可以看看workflow美女架构师的回答【https://www.zhihu.com/question/33084543/answer/2209929271】。

我总结了一下：

- shared_ptr是非侵入式指针，一层包一层，而且为了保持shared_ptr覆盖对象整个生命周期，每次传递时都必须带着智能指针模板，使用具有传染性，而且也比较麻烦。

- shared_ptr引用计数的内存区域和数据区域不一致，不连续，缓存失效可能导致性能问题，尽管有make_shared，但还是容易用错。

- 手动为对象做incref和decref，使用起来更灵活，可以明确在引用计数为固定数字时做一些自定义操作，而且方便调试。因为手动管理对象的引用计数，就会要求开发者明晰对象的生命周期，明确什么时候该使用对象，什么时候该释放对象。

- 如果使用shared_ptr可能会激起开发者的惰性，反正也不需要管理内存啦，就无脑使用shared_ptr呗，最后出现问题时调试起来也比较困难。

那再深入源码中研究研究，看看workflow是如何做到把对象指针给到外部后，内部还自动回收的。

拿WFClientTask举例说明一下，workflow中所有的Task都是通过Factory创建：

```
static WFHttpTask *create_http_task(const std::string& url,
```

注意，create参数里有一个callback，workflow一定会执行这个callback，然后内部回收掉该WFClientTask占用的内存，任何任务的生命周期都是从创建到callback函数结束。

它是怎么做到的？继续看下WFClientTask的继承层次结构：

```
template <class REQ, class RESP>
```

WFClientTask继承于WFNetworkTask，WFNetworkTask又继承于SubTask。

SubTask内部有subtask_done()方法，看下它的实现：

```
void SubTask::subtask_done() {
```

subtask_done()方法中会调用它的done()方法，然而这几个方法都是virtual方法，看看WFClientTask是怎么重写它们的：

```
template <class REQ, class RESP>
```

子类重写了done()方法，可以看到在它的实现里，触发了callback，然后调用了delete this，释放掉了当前占用的这块内存。

那谁调用的done()，可以看下上面的代码，subtask_done()会触发done()，那谁触发的subtask_done()：

```
void CommRequest::dispatch() {
```

可以看到，dispatch()里触发了subtask_done()，那谁触发的dispatch()：

```
template<class REQ, class RESP>
```

这里可以看到，Task的start()方法触发start_series_work()，进而触发dispatch()方法。

总结一下：

**● 步骤一**

通过工厂方法创建WFClientTask，同时设置callback；

**● 步骤二**

外部调用start()方法，start()中调用Workflow::start_series_work()方法；

**● 步骤三**

start_series_work()中调用SubTask的dispatch()方法，这个dispatch()方法由SubTask的子类CommRequest（WFClientTask的父类）实现；

**● 步骤四**

dispatch()方法在异步操作结束后会触发subtask_done()方法；

**● 步骤五**

subtask_done()方法内会触发done()方法；

**● 步骤六**

done()方法内会触发callback，然后delete this；

**● 步骤七**

内存释放完成。

其实这块可以猜到，想要销毁自己的内存，基本上也就delete this这个方法。

然而我认为这中间调用的思想和过程才是我们需要重点关注和学习的，远不止我上面描述的这么简单，感兴趣的朋友可以自己去研究研究哈。

关于workflow还有很多优点，这里就不一一列举啦，它的源码也值得我们CPP开发者学习和进阶，具体可以看我之前的文章。

发现workflow团队对这个项目相当重视，还特意建了个QQ交流群（群号码是618773193），对此项目有任何问题都可以在这个群里探讨，也方便了我们学习，真的不错。项目地址如下：https://github.com/sogou/workflow，也可以点击阅读原文直达。

在访问GitHub遇到困难时，可使用他们的Gitee官方仓库：https://gitee.com/sogou/workflow。

项目也特别实用，据说搜狗内外很多团队都在使用workflow。感觉这个项目值得学习的话就给人家个star，不要白嫖哈，对项目团队来说也是一种认可和鼓励。

阅读原文

阅读 3455

​

喜欢此内容的人还喜欢

写留言

**留言 21**

- Chanchan

  2022年2月15日

  赞5

  写了点workflow的源码分析 https://zhuanlan.zhihu.com/p/441615277 https://github.com/chanchann/workflow_annotation

  置顶

  程序喵大人

  作者2022年2月15日

  赞

  强啊，给你置顶

  Chanchan

  2022年2月15日

  赞

  哈哈，感谢喵哥，最近也在学习，慢慢更新源码分析

- 吴德宝(^0^)

  2022年2月15日

  赞3

  喵大人YYDS

  程序喵大人

  作者2022年2月15日

  赞

  别

- Forward

  2022年2月15日

  赞2

  喵哥，可以推荐推荐一些C++初学者的开源项目吗？

  程序喵大人

  作者2022年2月15日

  赞2

  会的，等一段时间，还在弄文案

- 好运来

  2022年2月28日

  赞

  喵哥，可以出一期文章，来介绍你一下你平时是如何学习、总结项目源码的吗？

  程序喵大人

  作者2022年2月28日

  赞1

  敬请期待

- Jstein

  2022年2月15日

  赞

  讲下taskflow呗

  程序喵大人

  作者2022年2月15日

  赞1

  敬请期待

- 廿七画生

  2022年2月15日

  赞1

  这个很不错👌，算法pipeline可以借用一下

- 向海

  2022年2月15日

  赞1

  还有其他不错的开源项目吗，想听你介绍介绍![[让我看看]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

  程序喵大人

  作者2022年2月15日

  赞

  敬请期待，确实有

- 雨乐

  2022年2月15日

  赞1

  虽然看不懂，但仍然感觉很牛逼的样子

  程序喵大人

  作者2022年2月15日

  赞

  挺有意思的，我研究了好长时间，还有几个正在研究！

- Xman

  2022年2月15日

  赞

  太棒了，我学习一下这种新的对象管理方式，正好需要。

- 黄星艳

  2022年2月15日

  赞

  版主，你的开源项目进行咋样了呀？

  程序喵大人

  作者2022年2月15日

  赞

  做着呢

- 任国庆Pic

  2022年2月15日

  赞

  shared_ptr是非侵入式指针，一层包一层，而且为了保持shared_ptr覆盖对象整个生命周期，每次传递时都必须带着智能指针模板，使用具有传染性，而且也比较麻烦。 这个怎么理解？？？

  程序喵大人

  作者2022年2月15日

  赞

  一两句话说不清楚，可以先去搜一下非侵入式智能指针和侵入式智能指针

已无更多数据

[](javacript:;)

![](http://mmbiz.qpic.cn/sz_mmbiz_png/Av3j9GXyMmGmg0HNK7fR6Peq3uf9pUoN5Qt0s0ia8IH97pvIiaADsEvy9nm3YnR5bmA13O8ic0JJOicQO0IboKGdZg/300?wx_fmt=png&wxfrom=18)

程序喵大人

25分享12

21

写留言

**留言 21**

- Chanchan

  2022年2月15日

  赞5

  写了点workflow的源码分析 https://zhuanlan.zhihu.com/p/441615277 https://github.com/chanchann/workflow_annotation

  置顶

  程序喵大人

  作者2022年2月15日

  赞

  强啊，给你置顶

  Chanchan

  2022年2月15日

  赞

  哈哈，感谢喵哥，最近也在学习，慢慢更新源码分析

- 吴德宝(^0^)

  2022年2月15日

  赞3

  喵大人YYDS

  程序喵大人

  作者2022年2月15日

  赞

  别

- Forward

  2022年2月15日

  赞2

  喵哥，可以推荐推荐一些C++初学者的开源项目吗？

  程序喵大人

  作者2022年2月15日

  赞2

  会的，等一段时间，还在弄文案

- 好运来

  2022年2月28日

  赞

  喵哥，可以出一期文章，来介绍你一下你平时是如何学习、总结项目源码的吗？

  程序喵大人

  作者2022年2月28日

  赞1

  敬请期待

- Jstein

  2022年2月15日

  赞

  讲下taskflow呗

  程序喵大人

  作者2022年2月15日

  赞1

  敬请期待

- 廿七画生

  2022年2月15日

  赞1

  这个很不错👌，算法pipeline可以借用一下

- 向海

  2022年2月15日

  赞1

  还有其他不错的开源项目吗，想听你介绍介绍![[让我看看]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

  程序喵大人

  作者2022年2月15日

  赞

  敬请期待，确实有

- 雨乐

  2022年2月15日

  赞1

  虽然看不懂，但仍然感觉很牛逼的样子

  程序喵大人

  作者2022年2月15日

  赞

  挺有意思的，我研究了好长时间，还有几个正在研究！

- Xman

  2022年2月15日

  赞

  太棒了，我学习一下这种新的对象管理方式，正好需要。

- 黄星艳

  2022年2月15日

  赞

  版主，你的开源项目进行咋样了呀？

  程序喵大人

  作者2022年2月15日

  赞

  做着呢

- 任国庆Pic

  2022年2月15日

  赞

  shared_ptr是非侵入式指针，一层包一层，而且为了保持shared_ptr覆盖对象整个生命周期，每次传递时都必须带着智能指针模板，使用具有传染性，而且也比较麻烦。 这个怎么理解？？？

  程序喵大人

  作者2022年2月15日

  赞

  一两句话说不清楚，可以先去搜一下非侵入式智能指针和侵入式智能指针

已无更多数据
