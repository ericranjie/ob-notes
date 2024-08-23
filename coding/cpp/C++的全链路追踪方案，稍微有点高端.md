# 

原创 程序喵大人 程序喵大人

 _2022年01月05日 08:24_

**背景：**本人主要在做C++ SDK的开发，需要给到业务端去集成，在集成的过程中可能会出现某些功能性bug，即没有得到想要的结果。那怎么调试？

  

**分析：**这种问题其实调试起来稍微有点困难，它不像crash，当发生crash时还能拿到堆栈信息去分析，然而功能性bug没有crash，也就没法捕捉对应到当时的堆栈信息。因为不是在本地，也没法用编译器debug。那思路就剩log了，一种方式是考虑在SDK内部的关键路径下打印详细的log，当出现问题时拿到log去分析。然而总有漏的时候，谁能保证log一定打的很全面，很有可能问题就出现在没有log的函数中。

  

**解决：**基于上面的背景和问题分析，考虑是否能做一个全链路追踪的方案，把打印出整个SDK的调用路径，从哪个函数进入，从哪个函数退出等。

  

**想法1：**可以考虑在SDK的每个接口都加一个context结构体参数，记录下来函数的调用路径，这可能是比较通用有效的方案，但是SDK接口已经固定了，更改接口要面临的困难很大，业务端基本不会同意，所以这种方案不适合我们现有情况，当然一个从0开始建设的中间件和SDK可以考虑考虑。

  

**想法2：**有没有一种不用改接口，还能追踪到函数调用路径的方案？

  

继续沿着这个思路继续调研，我找到了gcc和clang编译器的一个编译参数：-finstrument-functions，编译时添加此参数会在函数的入口和出口处触发一个固定的回调函数，即：

```
__cyg_profile_func_enter(void *callee, void *caller);
```

参数就是callee和caller的地址，那怎么将地址解析成对应函数名？可以使用dladdr函数：

```
int dladdr(const void *addr, Dl_info *info);
```

看下下面的代码：

```
// tracing.cc
```

这是测试文件：

```
// test_trace.cc
```

如果在func()中调用了一些其他的函数呢？

```
#include <iostream>
```

再重新编译后输出会是这样：

```
enter [no dli_sname nd std] (./a.out)
```

上面我只贴出了部分信息，这显然不是我们想要的，我们只想要显示自定义的函数调用路径，其他的都想要过滤掉，怎么办？

  

这里可以将自定义的函数都加一个统一的前缀，在打印时只打印含有前缀的符号，这种个人认为是比较通用的方案。

  

下面是我过滤掉std和gnu子串的代码：

```
if (!strcasestr(name, "std") && !strcasestr(name, "gnu")) {
```

重新编译后就会输出我想要的结果：

```
g++ test_trace.cc tracing.cc -std=c++14 -finstrument-functions -rdynamic -ldl;./a.out
```

还有一种方式是在编译时使用下面的参数：

```
-finstrument-functions-exclude-file-list
```

它可以排除不想要做trace的文件，但是这个参数只在gcc中可用，在clang中却不支持 ，所以上面的字符串过滤方式更通用一些。

  

上面只能拿到函数的名字，不能定位到具体的文件和行号，如果想要获得更多信息，需要结合bfd系列参数(bfd_find_nearest_line)和libunwind一起使用，大家可以继续研究。。。

  

**tips1：**这是一篇抛砖引玉的文章，本人不是后端开发，据我所知后端C++中有很多成熟的trace方案，大家有更好的方案可以留言，分享一波。

  

**tips2：**上面的方案可以达到链路追踪的目的，但本人最后没有应用到项目中，因为本人在做的项目对性能要求较高，使用此种方案会使整个SDK性能下降严重，无法满足需求正常运行。于是暂时放弃了链路追踪的这个想法。

  

本文的知识点还是值得了解一下的，大家或许会用得到。在研究的过程中我也发现了一个基于此种方案的开源项目（call-stack-logger），感兴趣的也可以去了解了解。

  

  

往期推荐

  

  

[

函数返回值的行业潜规则



](http://mp.weixin.qq.com/s?__biz=MzkyODE5NjU2Mw==&mid=2247491440&idx=1&sn=0baf7e7e28d0c4c0f6ca6b14fe0df2a7&chksm=c21d2dccf56aa4daa84dcc87f541efae82f575abfe474815d3764ace918f9638267ddac8801a&scene=21#wechat_redirect)

[

面试常问的16个C语言问题，你能答上来几个？



](http://mp.weixin.qq.com/s?__biz=MzkyODE5NjU2Mw==&mid=2247491401&idx=1&sn=5809b2e5e58a4530c4a16dc51f9fb32e&chksm=c21d2df5f56aa4e3e32faf409318e101e53c5ec490fb1d1db3474f8de8b6c110d7149b8f5218&scene=21#wechat_redirect)

[

模版定义一定要写在头文件中吗?



](http://mp.weixin.qq.com/s?__biz=MzkyODE5NjU2Mw==&mid=2247491391&idx=1&sn=149e8289e0545d5f7513661aef792f9c&chksm=c21d2d83f56aa49509c87ea188089e86d2d779de1bdd31a5d3b72eccf4c6d0ca21c9b1a54d86&scene=21#wechat_redirect)

[

为什么建议少用if语句！



](http://mp.weixin.qq.com/s?__biz=MzkyODE5NjU2Mw==&mid=2247491362&idx=1&sn=0cafab8395443227af0879160dcf90a0&chksm=c21d2d9ef56aa4881bc639ec5025ec5ff2bc12db424f9b7a2266505cb66e72311f48de86fa99&scene=21#wechat_redirect)

[

四万字长文，这是我见过最好的模板元编程文章！



](http://mp.weixin.qq.com/s?__biz=MzkyODE5NjU2Mw==&mid=2247491328&idx=1&sn=a9e4147a1ffe5de9c8a7f0c8d5377405&chksm=c21d2dbcf56aa4aa5d5e60e3a4ecece847cbcb777a97f437010b1300c02ab36f9ecc01dca65d&scene=21#wechat_redirect)

[

如何正确的理解指针和结构体指针、指针函数、函数指针这些东西？



](http://mp.weixin.qq.com/s?__biz=MzkyODE5NjU2Mw==&mid=2247491314&idx=1&sn=d62d17d2ba885e310fd6d9d29eee5159&chksm=c21d2c4ef56aa558652b15c1dea02aff9d84f322988633ed2add536952a64a30ae622323c749&scene=21#wechat_redirect)

[

C++为什么要弄出虚表这个东西？



](http://mp.weixin.qq.com/s?__biz=MzkyODE5NjU2Mw==&mid=2247491276&idx=1&sn=7301628743ca8baaaa56e9958808e72f&chksm=c21d2c70f56aa5668bf00c481548527c4cb4c786197d5b663908aec211dae0c8750eb00d2d34&scene=21#wechat_redirect)

  

![](https://res.wx.qq.com/op_res/NN_GToMiIjsXzgPzF9-74ZzwR3cA9-fv3o9eWo8f5gQWqx71CmGlY8kFxuIxZaG0TB1bFeMCmh1DGN_pWMRg0A)

**帐号已迁移**

公众号

  

阅读原文

阅读 3792

​

写留言

**留言 30**

- 程序喵
    
    2022年1月5日
    
    赞2
    
    小伙伴们可加微信cxmdrcpp进技术交流群，与更多同道中人一起成长。
    
    置顶
    
- 小ç涂鸦
    
    2022年1月5日
    
    赞
    
    喵哥可不可以写一些关于muduo网络库的，这个挺适合我这种初学者的![[坏笑]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)
    
    程序喵大人
    
    作者2022年1月5日
    
    赞1
    
    muduo作者不是有一本书吗？虽然一些内容有争议，但写的很好，看完会有很多收获的
    
    小ç涂鸦
    
    2022年1月5日
    
    赞
    
    那个书好难看![[流泪]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)
    
    程序喵大人
    
    作者2022年1月5日
    
    赞3
    
    书还是不错的，值得看，他还有博客和pdf，都很好，我就看他的资料长大的
    
- jojogh得木
    
    2022年1月5日
    
    赞2
    
    高级
    
    程序喵大人
    
    作者2022年1月5日
    
    赞2
    
    大佬肯定有更好的方案，分享一波![[可怜]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)
    
    jojogh得木
    
    2022年1月5日
    
    赞
    
    我也没有哈哈 虽然我也是做sdk的，但是我的应用没那么高级![[坏笑]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)
    
    程序喵大人
    
    作者2022年1月5日
    
    赞
    
    谦虚了
    
- 秦时明月
    
    2022年2月4日
    
    赞1
    
    喵兄当心你在这里提go不合适， 参数名和值呢
    
    程序喵大人
    
    作者2022年2月4日
    
    赞
    
    ？
    
- Just
    
    2022年1月24日
    
    赞
    
    windows上可以直接调试器加载符号调试 linux不太熟
    
    程序喵大人
    
    作者2022年1月24日
    
    赞1
    
    说的貌似不是一个问题
    
- Moment
    
    2022年1月5日
    
    赞1
    
    学习了，学习了。
    
- 秦时明月
    
    2022年2月4日
    
    赞
    
    链路跟踪的参数和值呢
    
    程序喵大人
    
    作者2022年2月4日
    
    赞
    
    这块你有啥好办法不
    
    秦时明月
    
    2022年2月5日
    
    赞
    
    喵兄新年好,这个一两句解答不了,喵兄等吸收完 你所有的文章,咱们再来解决这个问题,哈哈
    
- 桀。
    
    2022年1月20日
    
    赞
    
    哈哈，我也做sdk的，没有这么高级
    
    程序喵大人
    
    作者2022年1月24日
    
    赞
    
    可以尝试尝试，挺有意思的
    
- 义勇
    
    2022年1月6日
    
    赞
    
    AspectC++？
    
    程序喵大人
    
    作者2022年1月24日
    
    赞
    
    这稍微麻烦一些
    
- 画中的你低着头挨打
    
    2022年1月5日
    
    赞
    
    打出来只有地址的trace文件，整个脚本，发版就上传符号表，tracer文件自动去服务器解析不好吗?
    
    程序喵大人
    
    作者2022年1月5日
    
    赞
    
    有道理
    
    程序喵大人
    
    作者2022年1月5日
    
    赞
    
    但是怎么记录符号路径
    
- 阿秀
    
    2022年1月5日
    
    赞
    
    收藏。。。
    
    程序喵大人
    
    作者2022年1月5日
    
    赞
    
    go有成熟的方案![[偷笑]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)
    
- zxy
    
    2022年1月5日
    
    赞
    
    高端
    
- 闪闪的光
    
    2022年1月5日
    
    赞
    
    学习了。
    
- 雨乐
    
    2022年1月5日
    
    赞
    
    学习了。不过正如文中所说，如果日志太多，会影响性能。那么是否可以考虑有个远程开关，既可以打开或者关闭日志跟踪，又可以调整日志级别，最好也可以选择部分用户。这样的话当sdk出现crash的时候，就能远程操。必要的时候可以通过上报，把日志上报给服务器进行分析
    
    程序喵大人
    
    作者2022年1月5日
    
    赞
    
    是一种方案，但不适合我这个项目，这个方案不是因为日志太多影响性能，而是dladdr函数耗时较高，在移动端开启后功能都不能正常运行，更不用说调试了，但是我测试在pc端性能还好
    

已无更多数据

[](javacript:;)

![](http://mmbiz.qpic.cn/sz_mmbiz_png/Av3j9GXyMmGmg0HNK7fR6Peq3uf9pUoN5Qt0s0ia8IH97pvIiaADsEvy9nm3YnR5bmA13O8ic0JJOicQO0IboKGdZg/300?wx_fmt=png&wxfrom=18)

程序喵大人

29分享7

30

写留言

**留言 30**

- 程序喵
    
    2022年1月5日
    
    赞2
    
    小伙伴们可加微信cxmdrcpp进技术交流群，与更多同道中人一起成长。
    
    置顶
    
- 小ç涂鸦
    
    2022年1月5日
    
    赞
    
    喵哥可不可以写一些关于muduo网络库的，这个挺适合我这种初学者的![[坏笑]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)
    
    程序喵大人
    
    作者2022年1月5日
    
    赞1
    
    muduo作者不是有一本书吗？虽然一些内容有争议，但写的很好，看完会有很多收获的
    
    小ç涂鸦
    
    2022年1月5日
    
    赞
    
    那个书好难看![[流泪]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)
    
    程序喵大人
    
    作者2022年1月5日
    
    赞3
    
    书还是不错的，值得看，他还有博客和pdf，都很好，我就看他的资料长大的
    
- jojogh得木
    
    2022年1月5日
    
    赞2
    
    高级
    
    程序喵大人
    
    作者2022年1月5日
    
    赞2
    
    大佬肯定有更好的方案，分享一波![[可怜]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)
    
    jojogh得木
    
    2022年1月5日
    
    赞
    
    我也没有哈哈 虽然我也是做sdk的，但是我的应用没那么高级![[坏笑]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)
    
    程序喵大人
    
    作者2022年1月5日
    
    赞
    
    谦虚了
    
- 秦时明月
    
    2022年2月4日
    
    赞1
    
    喵兄当心你在这里提go不合适， 参数名和值呢
    
    程序喵大人
    
    作者2022年2月4日
    
    赞
    
    ？
    
- Just
    
    2022年1月24日
    
    赞
    
    windows上可以直接调试器加载符号调试 linux不太熟
    
    程序喵大人
    
    作者2022年1月24日
    
    赞1
    
    说的貌似不是一个问题
    
- Moment
    
    2022年1月5日
    
    赞1
    
    学习了，学习了。
    
- 秦时明月
    
    2022年2月4日
    
    赞
    
    链路跟踪的参数和值呢
    
    程序喵大人
    
    作者2022年2月4日
    
    赞
    
    这块你有啥好办法不
    
    秦时明月
    
    2022年2月5日
    
    赞
    
    喵兄新年好,这个一两句解答不了,喵兄等吸收完 你所有的文章,咱们再来解决这个问题,哈哈
    
- 桀。
    
    2022年1月20日
    
    赞
    
    哈哈，我也做sdk的，没有这么高级
    
    程序喵大人
    
    作者2022年1月24日
    
    赞
    
    可以尝试尝试，挺有意思的
    
- 义勇
    
    2022年1月6日
    
    赞
    
    AspectC++？
    
    程序喵大人
    
    作者2022年1月24日
    
    赞
    
    这稍微麻烦一些
    
- 画中的你低着头挨打
    
    2022年1月5日
    
    赞
    
    打出来只有地址的trace文件，整个脚本，发版就上传符号表，tracer文件自动去服务器解析不好吗?
    
    程序喵大人
    
    作者2022年1月5日
    
    赞
    
    有道理
    
    程序喵大人
    
    作者2022年1月5日
    
    赞
    
    但是怎么记录符号路径
    
- 阿秀
    
    2022年1月5日
    
    赞
    
    收藏。。。
    
    程序喵大人
    
    作者2022年1月5日
    
    赞
    
    go有成熟的方案![[偷笑]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)
    
- zxy
    
    2022年1月5日
    
    赞
    
    高端
    
- 闪闪的光
    
    2022年1月5日
    
    赞
    
    学习了。
    
- 雨乐
    
    2022年1月5日
    
    赞
    
    学习了。不过正如文中所说，如果日志太多，会影响性能。那么是否可以考虑有个远程开关，既可以打开或者关闭日志跟踪，又可以调整日志级别，最好也可以选择部分用户。这样的话当sdk出现crash的时候，就能远程操。必要的时候可以通过上报，把日志上报给服务器进行分析
    
    程序喵大人
    
    作者2022年1月5日
    
    赞
    
    是一种方案，但不适合我这个项目，这个方案不是因为日志太多影响性能，而是dladdr函数耗时较高，在移动端开启后功能都不能正常运行，更不用说调试了，但是我测试在pc端性能还好
    

已无更多数据