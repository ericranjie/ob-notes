# 

Original 探寻可能 三棵杨述

_2021年12月04日 23:20_

![Image](https://mmbiz.qpic.cn/sz_mmbiz_png/DfBaOpaDk09VcPt1zwc0pjr0dUdOEYrnvZFO6eXyZ7WibXQxYD74qO0c6TDTSC7ZKs5RUn3vqOHiaz83cvOGjTdw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

跑通上面这张图片的时候，我非常激动，因为这是我第一次在C++编写引擎这件事情上站稳了脚跟。我迫不及待的想要告诉大家这件事情，但直到今天我将PhysX也整合进来，形成了包括物理，动画，脚本系统在内的所有逻辑后，才着手写这篇文章。拥有这么一个引擎，意味着在图形学这件事上，特别是渲染和引擎这件事上，终于算是“入门”了。（虽然我玩图形已经差不多三年了）这个入门之所以打引号，后面我会详细解释，但并不妨碍我拥有了一套组件架构作为基础，这套基础上可以逐步深入包括动画，物理，渲染在内的各个组件，重构，强化，修改，而不再是在单个小分支上像无头苍蝇一样乱飞，这点意义对我个人来说是非常重要的。

我在知乎写过不少文章，其中点赞最高的一篇是《有关图形学入门的一些思考·渲染篇》，大概有四百多个赞，知乎上果然是列资源最能收获点赞，但说句实话，与其说这是一些资源，不如说是我在摸索图形的过程中，曾经尝试过的方向。今天这篇文章，我希望总结一下这几年的经历，是我自己的一段回顾，也希望给大家一种参考，这种基于实践的参考，其实比那些资源要来的更加有效，因为很多事情，只有做过了，才能明白。

## 第一个缝合怪

回想自己什么时候开始真正研究渲染这件事，其实应该是去年看OGRE代码的时候，这么一想，其实距离现在也就是两年的时间。当时在知乎上写了一系列文章，那时候找工作还被面试官抽出来问，其实我已经基本上忘记了。那时候连编译都不太会，把OGRE代码一个个导入到Xcode里面，搞到凌晨两三点还很嗨（想念在澳大的生活），编译不过就改include，从19年的下半年开始，就在做这种粘贴复制的事情。OGRE代码非常庞大，当时看的也是云里雾里，但这个过程中主要的收获大概就是知道更多C++的知识，比如分配器之类的东西。当时我看到OGRE实现了一种any类型类包装各种类型不同的数据，具体什么用我忘记了，但当时查了一下发现C++17提供了这一类型。我在开发现在的引擎时想到这件事，就拿它来包装uniform，使得各种不同类型的uniform，都可以用同一种setData的方式存储起来，并且借助渲染管线的反射来自动配置数据。

说了这么多，我的第一个缝合怪应该是来自《Metal by Tutorials》这本书我推荐过很多次，现在其实还会查看里面的实现。我做的一件事，其实就是把各章的代码整合在一起。并且其实作者本事也是有一套框架的，所以我其实没有做什么事情。为了学习这套东西，我掌握了Swift，从此开始被苹果生态困住。

苹果生态的好处是，你需要的大多数辅助工具他都可以提供，虽然每一个都不太好，但组合起来威力很大。在Xcode上，不仅可以预览模型，还可以抓帧，Swift语言不需要考虑所有权和垃圾回收之类的事情，ModelIIO帮你加载包括usd和obj在内的大多数模型，处理包括贴图和骨骼在内的大多数数据。但是当你想要链接physx的时候，就必须自己编写ObjC的桥阶层，于是，C++生态里面的大多数东西，都与我无缘了。

## 第二个缝合怪

在第一个缝合怪的引领下，我做了很多奇奇怪怪的事情，包括把一套C++写的流体模拟框架全部搬到了Swift上，并且做了GPGPU加速。我掌握了一种学习他人武功的奇怪心法，跨语言抄一遍。这种逻辑让我在后面又开始学习Rust，在Rust里面踩了非常多的坑。尽管如此，我的引擎之路依旧停步不前，折腾过一段时间的bgfx，但也止步于场景图，没有真正形成引擎的能力。

第二个缝合怪诞生于Swift复刻Oasis这个想法，其实Swift是一个参考了非常多种语言能力得到的究极缝合怪，用Swift复刻Oasis的最大障碍并不是语言，而是GL转Metal，这个过程中，我对比了WebGL和Metal两种API，也让我重新认识了Metal的API细节，包括pipelineState的缓存，管线的反射等等。搭建完成之后，我把PhysX用ObjC做了桥接连接过来，想着既然渲染和物理都有了，骨骼动画如果也可以加进来，不就是万事俱备了吗？于是想到折腾OZZ，这一折腾导致第二个缝合怪扑街。

我对骨骼动画本身没有什么概念，折腾了一圈才发现，原来有CPU蒙皮和GPU蒙皮，GPU蒙皮一般会规定joint影响的个数，例如4个，但CPU蒙皮没有限制，并且可以直接植入到现有的渲染系统里面去而不需要在Shader上做特殊的处理。一开始我把OZZ用Swift抄了一遍（因为我曾经用Rust抄过，比较顺畅，Rust都没有问题的东西其他语言更加不会有障碍），后来发现整个蒙皮逻辑都不一样，蒙皮后的数据我完全无法导入。更重要的是，在此之前我都是用ModelIO导入模型，现在要用FBX，整个引擎的文件导入已经完全乱套了，缝合了太多东西之后，缝合怪都要原地爆炸了。

就是因为这一点，让我意识到，依靠桥基层做引擎的思路已经走到尽头了，Swift也不是一种底层语言，语言表现力非常有限，甚至在生态上还不如ObjC来的自然。ObjC过去我一直觉得语法很奇特，但桥接写多了渐渐意识到，其实ObjC真的是一种C语言面向对象的扩展，相比于C++这一类看起来像是C的扩展的语言，ObjC能够兼容C所有的能力，例如bitflag之类的特性（这一特性Rust都没有支持）。并且我意识到，虽然Swift的语法和TS非常类似，但究其根本还是JIT和AOT的不同，Swift一直存在编译时间长的问题，相比于TS，改完之后立刻运行生效，Swift要等待太长的时间，甚至由于增加了IR，所以编译时间比C++要长更多。

## 第三个缝合怪

走了这么多弯路，但让我更加了解了各个语言的优劣势。并且由于有了桥接的经验，我学会了如何用 ObjC 去调用 Metal 的 API，这让我想到为何不全部回到 C++，这样永远不会有桥接的问题，并且可以一开始就规划好文件导入层的设计，让动画，物理，渲染更加有机地整合起来。

于是，第三个缝合怪诞生了，他在 Oasis 的组件架构基础上整合了，Metal 的渲染，Ozz 的动画系统，PhysX 提供的碰撞检测和物理系统，IMGUI 提供的 GUI。我统一了全局的分配器，规划好 unique_ptr 和shared_ptr分别适用那些对象，全面考虑了资源的生命周期。最终收获了开始那张图片中的代码。虽然现在还没有完全理清楚 const 的逻辑，以及还存在内存泄漏的问题（但可以在析构时检测到）引擎整体架构已经非常清晰了。

在无数次抄各种代码之后，一个又一个的缝合怪被新的组件打败，又重新设计了新的缝合怪。所以在引擎当中，对于来自OZZ，PhysX，IMGUI的各种能力我其实都还没有完全掌握，包括SIMD加速，碰撞检测算法，Atlas图集打包等等，但对我自己来说，已经是一个新的起点，可以有余力各个击破这些问题。

## 总结

在知乎上总有同学会问我一些学习路径的问题，我这篇文章或许可以给这些同学一个参考。我不是科班出生，也不未曾在游戏工作室工作过。只是将其当成是一生的爱好在做。我曾经的工作是物理仿真开发，工作语言就是C++，所以最终回归到C++让我感觉回到了自己家一样，虽然我曾经非常讨厌C++，因为这并不是一门很容易入手的语言，而且如果让初学者一开始从C++做引擎，在生命周期这件事上就已经挡住了他的去路。所以我的路径其实是从更加容易入手的语言开始，逐渐回归到C++的过程，看似弯路，但其实更容易进入到问题的本质。_**当然，曾经的我并不知道问题的本质是什么，但如果知道本质是什么，然后再去从简单的语言入手一步步实践，我想是一条不错的路径，就像是我们现在在Oasis上做的事情一样。**_

## 题图

这是最开始的那段代码渲染的结果，是一个奔跑的小狗，我在sketchFab上花了五刀买了用于测试，有时忙起来没有时间开发引擎，就会运行一段代码看一看他跑动的样子，虽然现在只有unlit材质，灯光还没做好，但依旧让我感到非常欣慰。希望每一位在这一条路上追寻的朋友，都能够保持内心最初的热爱。

![](https://mmbiz.qlogo.cn/mmbiz_jpg/jzd55crUykEt4GG7r8tOHQOxKRwoe7WLzfbhywpy5WTdEP1VcOibUDZ6Oh2Hv7Z7BlicroJQv1eBIawkaKPZ4Sqg/0?wx_fmt=jpeg)

探寻可能

![赞赏二维码](<https://mp.weixin.qq.com/s?__biz=MzI2OTQwMDIxOQ==&mid=2247483738&idx=1&sn=9692ef5f07527cafc75d1b164d69b40e&chksm=eae1a0aedd9629b8c6984bc355027f229cb318d9b4c5e0f323001d312ef387bd45597b8d1898&mpshare=1&scene=24&srcid=1208ZEY8xOUWCx44zRyWFW8f&sharer_sharetime=1638922250366&sharer_shareid=5fb9813bfe9ffc983435bfc8d8c5e9ca&key=daf9bdc5abc4e8d0f04d899e16511ee6515f772eab0904d928a3a37e7103e3ac5804d48eee140bc5b146b56e90e52efe9777cc3e296332a6ccc5c9331e530bbc5650f765883d46ac9b890b0538630e1bc03dc18c5b66d6d6242b80cf5e0d07610bbed396ef3ccd0ecb8e6ced1a467857e774ffa59f183cd42abe515985e2a6b7&ascene=14&uin=MTEwNTU1MjgwMw%3D%3D&devicetype=iMac+MacBookAir10%2C1+OSX+OSX+14.6.1+build(23G93)&version=13080710&nettype=WIFI&lang=en&countrycode=CN&fontScale=100&exportkey=n_ChQIAhIQ2UZgmhKQ%2FnbnHcV1C7cBtBKFAgIE97dBBAEAAAAAAKICIMV9RhwAAAAOpnltbLcz9gKNyK89dVj0TBGk03oOaJsf%2F1Z2CgJSmlBFkhnQQzvuh478rTii6oaSzcwi5Ml62XOPkrnD8oL5Tvn7yySvQHoV6Lkh1EvzvM1%2BrkktVrafFWCTGJvuSQk9wYBtxyFbxT%2B8KAtNq3dN4wtL4CDPO0AEFRbiTCB%2FkYBBNLkqQxKnrx9JzOvhPGbdez4FZ%2FoCrpZBnT%2B11aoQ7Fqdz7WpUzzDIMRoickaPMb0iI332501Jh%2FLcm8hd%2FU5lRinjVhN2djMklJkjksG1%2Bbru84H8XRWpukI932BY1OLazcNogHnWd49LSdECw%3D%3D&acctmode=0&pass_ticket=SMvXMMHDxD%2BCmgIh%2FIJQvIRO%2BfM7A%2Bty%2FHQ9BfTb2E6cmWhQkUpJ4K5TCaGSAorW&wx_header=0>)Like the Author

4 like(s)

![](http://wx.qlogo.cn/mmopen/PiajxSqBRaEJpGE2fdrt6DpJ3zj1zbPwT0A6OmOcQIGJHsic5wlJk1tRhp2aicDI4rymOBHydByKFSnAib8I2FKq1dlEbUM05HkF2dIen4mlo9Wy0Zibxh8KB3uta5NfobK81/64)![](http://wx.qlogo.cn/mmopen/p4WTG6rGMT1WxvmJFzSBOdUKLhVfiaicHibRN2KYsDZmruqeIlicy9EUaNgGx2HMGnNn6WhCibIlaeD6UV3ha88s5RDHpe2sxPvNk/64)![](http://wx.qlogo.cn/mmopen/PiajxSqBRaEIocLib1xibloIPQnBxKSgAdjXIGUhnZjf908HGUF6K50LS3l8xfybKiafbewZu87qqib2osXWhsY2X5vTQWCCSHTLyuAIUlueWPjPgzLSR45OtwQaVYm8ZmzxK/64)![](http://wx.qlogo.cn/mmopen/p4WTG6rGMT1WxvmJFzSBOcZFQweY0gjWicYFJaHZ9NcicsyDJqC4IqSSaASFb0z7ImqFzoeITHJuxFQmicQdbT3zW1WgSoJxl4K/64)

Read more

Reads 684

​

Comment

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/DfBaOpaDk0icGAE1G2wBcJ9eSGftUaEdttyybiaFQfhr1iaOUia8dc3nUyEBqGWorfNHamJkv019IY8ic4W44qibcfaA/300?wx_fmt=png&wxfrom=18)

三棵杨述

Follow

1314

Comment

Comment

**Comment**

暂无留言
