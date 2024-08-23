
张小方 CppGuide

 _2021年10月27日 14:33_

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/ic8RqseyjxMNIHpGL9MPSukSuVSf30wyLBNCKqyUEvMTKic4WWXZLfx9jJQM3kYQmUho1kzkBF9ibWk0sbplib1pHg/640?wx_fmt=jpeg&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

  

写了十多年 C++，做过 10+ 大型 C/C++ 商业项目，分享一下给我帮助很大的开源 C++ 项目吧。  

## 

一、推荐一些不错的 C++ 开源项目

如何阅读开源项目可以看《[2020 年好好读一读开源代码吧](http://mp.weixin.qq.com/s?__biz=MzU2MTkwMTE4Nw==&mid=2247488282&idx=1&sn=cea60e8d4c92c91f8e9bf9351a864df8&chksm=fc70e8f6cb0761e0b04a986916fcaefbded16184ed8b14af4b9fc576ec22553e5220c98166eb&scene=21#wechat_redirect)》。

  

如何调试开源 C/C++ 项目的代码可以看《[如何 C/C++ 开源代码](https://mp.weixin.qq.com/s?__biz=MzI0NTE4NTE4Nw==&mid=2648792470&idx=1&sn=d65a9f9d927923e3c19c3645c4328ab1&scene=21#wechat_redirect)》。

  

### 

1. libevent

libevent 是广泛应用的 C++ 网络库，也是现在很多 C/C++ 网络库的雏形，想了解 C/C++ 网络库最初的形态、设计与演化思想一定要好好学习一下这个库，可以这么说，想成为 C/C++ 高手，libevent 是必学源码之一。

  

源码地址：

https://github.com/libevent/libevent

  

我整理了一些关于 libevent 的学习资料：

> 链接: https://pan.baidu.com/s/1WdfnS6SDkxKseHDLb6tpaQ
> 
> 提取码: ikk4

# ![图片](https://mmbiz.qpic.cn/mmbiz_jpg/ic8RqseyjxMNIHpGL9MPSukSuVSf30wyLNjcQgF4KxYkoTK6oPxQAXxcjBSHK85pLyYyic7VtKlZXLUZGbB0ahLQ/640?wx_fmt=jpeg&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/ic8RqseyjxMNIHpGL9MPSukSuVSf30wyLaEmkjNia0hCiaYvw5kAtjAiaxtjqhiba8IgBz75FqtDsrfwDVfrIVGibGyA/640?wx_fmt=jpeg&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

### 

2. Filezilla

Filezilla 是我用了十多年的一款开源 FTP 软件，2016 年的时候，我无意中发现 Filezilla 竟然用 C++ 11 重写了。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

Filezilla界面

  

FileZilla 的源码是一个德国开发者写的，其代码质量也不错，而且使用的是 C++11 写的。可以一边调试一边学习，学完后，我的 C++11 功能得到了大大增强。

  

不怕你笑话，我在上学的时候，曾看过 Filezilla 0.x 版本的代码，那个时候 UI 界面用的还是 MFC。

  

贴一下 Filezilla 的部分代码，红框标出来的部分为 C++11 的语法特性：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

代码质量总体很不错。我修改了下让其可以在 Visual Studio 中调试，这样你可以一边调试一边学习。

  

一套源码如果能够容易编译、调试，同时其业务是容易理解的（通俗地说，就是这套代码的功能是什么的），那么才利于新手学习。

  

我已经将环境和依赖都配置好了，代码获取方式：

![](https://res.wx.qq.com/op_res/NN_GToMiIjsXzgPzF9-74ZzwR3cA9-fv3o9eWo8f5gQWqx71CmGlY8kFxuIxZaG0TB1bFeMCmh1DGN_pWMRg0A)

**帐号已迁移**

公众号

关注后回复“获取Filezilla源码”

### 

3. uWebSocket

uWebSocket 是一款开源的 WebSocket 库，最新版使用了大量 C++17 的语法，代码量非常少。

  

下载地址：

> https://github.com/uNetworking/uWebSockets

我们改造了这个项目，用于我们的交易系统的行情推送服务器。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

### 

4. TeamTalk

TeamTalk 是蘑菇街开源的一款用于企业内部的即时通信工具，学好它可以作为面试项目对付大多数初中级 C/C++ 面试，**强烈推荐给应届生和工作不久的 C/C++ 开发的同学**，其代码下载地址是：

> https://github.com/balloonwj/TeamTalk

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

### 

5. poco 库

https://pocoproject.org/

  

poco 库是一个代码质量非常高，且文档比较丰富的 C++ 库，实现了常用的一些功能，可以根据自己的喜好挨个看。POCO 有大量 C/C++ 实用技巧，每段代码都很精华，而且可以单个单个 cpp 文件地看，比看 《Effective C++》系列爽多了。

  

### 

6. FlamingIM

https://github.com/balloonwj/flamingo

  

我为 Flamingo 专门录制了 3 部高清技术讲解视频以方便读者学习，视频中介绍了 Flamingo 的编译和部署方法、整体架构、各个模块的技术实现细节以及如何学习Flamingo的方法，需要视频教程下载方法： 

![](https://res.wx.qq.com/op_res/NN_GToMiIjsXzgPzF9-74ZzwR3cA9-fv3o9eWo8f5gQWqx71CmGlY8kFxuIxZaG0TB1bFeMCmh1DGN_pWMRg0A)

**帐号已迁移**

公众号

关注后回复“flamingo”获取flamingo教学视频

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

### 

7. tinySTL

https://github.com/balloonwj/TinySTL

  

### 

8. 金山卫士

主界面

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

垃圾清理

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

ARP防火墙

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

隐私保护器

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

源码获取方法：

![](http://mmbiz.qpic.cn/mmbiz_png/OIdl32LQlqpmKgWE0wRXCiafyf8F0GZbX5ldCfUdmYLpngEDp1QfSD6RsuukHIq0llJ8yiayfCqEFibVX9n6OQwTQ/300?wx_fmt=png&wxfrom=19)

**程序员小方**

技术，生活，编码，加班，读书学习，这里是程序员小方的 IT 生活。

21篇原创内容

公众号

关注后回复“五套源码”获取金山卫士源码

  

### 

9. 电驴源码

  

作为电驴源码的受益者的过来人，如果你认真把这套电驴源码看完，你可以学习到大量 C++ 编程技巧，学会如何成为一个客户端架构师。强烈推荐给做 Windows C/C++ 开发的同学。  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

**源码获取方式：**

![](http://mmbiz.qpic.cn/mmbiz_png/OIdl32LQlqpmKgWE0wRXCiafyf8F0GZbX5ldCfUdmYLpngEDp1QfSD6RsuukHIq0llJ8yiayfCqEFibVX9n6OQwTQ/300?wx_fmt=png&wxfrom=19)

**程序员小方**

技术，生活，编码，加班，读书学习，这里是程序员小方的 IT 生活。

21篇原创内容

公众号

关注后回复“五套源码”获取电驴源码

  

  

## 

二、应该如何学习 C++

C/C++ 这门语言与其他高级语言不同，它是离操作系统较近的语言。所以学好 C/C++ 体系的技术栈必须结合操作系统的运行机制来学习，通俗地说，就是你必须掌握操作系统层面的几大基础知识，他们是汇编、编译链接与运行时体系、狭义的操作系统原理、多线程、网络编程，只有这样学习，你才能学的懂、学的通透、学以致用。咱们学习 C++ 不是为了理论研究，而是付诸实践，投入生产是吧。

用一张图来概括一下 C++ 技术栈吧：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

## 

三、学好 C++ 语言包括 C++11/14/17 常用语法

C++ 面试关于语法部分一般会问以下一些问题，当然这些问题也是 C++ 开发必备：

- 在有继承关系的父子类中，构建和析构一个子类对象时，父子构造函数和析构函数的执行顺序分别是怎样的？
    
- 在有继承关系的类体系中，父类的构造函数和析构函数一定要申明为 virtual 吗？如果不申明为 virtual 会怎样？
    
- 什么是 C++ 多态？C++ 多态的实现原理是什么？
    
- 什么是虚函数？虚函数的实现原理是什么？
    
- 什么是虚表？虚表的内存结构布局如何？虚表的第一项（或第二项）是什么？
    
- 菱形继承（类 D 同时继承 B 和 C，B 和 C 又继承自 A）体系下，虚表在各个类中的布局如何？如果类B和类C同时有一个成员变了m，m如何在D对象的内存地址上分布的？是否会相互覆盖？
    

  

部分同学对以上问题总是搞不清楚，但是又不知道如何学习，于是从网上找各种文章来学习，造成这块的知识非常零碎，无法构成体系，其实这与其在网上花费大量时间，不如系统地看一下侯捷老师翻译的《深度探索 C++ 对象模型》一书。

  

这本书专注于 C++ 面向对象程序设计的底层机制，包括结构式语意、临时性对象的生成、封装、继承，以及虚拟——虚拟函数和虚拟继承。这本书让你知道：一旦你能够了解底层实现模型，你的程序代码将获得多么大的效率。对于 C++ 底层机制感兴趣的读者，这是一本让你大呼过瘾的绝妙好书。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

另外，时至今日，各大企业虽然项目中未必用到 C++11/14/17 常用的语言特性和类库，但是面试还是对这些有一定的要求的，常问的有：

- 统一的类成员初始化语法与 std::initializer_list<T>
    
- 注解标签（attributes）
    
- final/override/=default/=delete 语法
    
- auto 关键字
    
- Range-based 循环语法
    
- 结构化绑定
    
- stl 容器新增的实用方法
    
- std::thread
    
- 线程局部存储 thread_local
    
- 线程同步原语 std::mutex、std::condition_variable 等
    
- 原子操作类
    
- 智能指针类
    
- std::bind/std::function
    

C++11/14 网上的资料已经很多了，C++17 的资料不多，重头戏还是 C++11 引入的各种实用特性，这就给读者推荐一些我读过的不错的书籍：

- 《深入理解 C++11：C++11 新特性解析与应用》
    
- 《深入应用 C++11：代码优化与工程级应用》
    
- 《C++17 完全指南》
    
- 《Cpp 17 in Detail》
    

> 链接: https://pan.baidu.com/s/1MfvlrHqsDMtga8gW7JNgzA 提取码: thup

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

## 

四、进一步提高 C++

当你学习好了 C++ 语言本身，你可以学习一下 C++ 的一些常见惯用法和高性能编码实践，这里我推荐一本经典书籍叫《提高C++性能的编程技术》，这本书详细讨论了临时对象、内存管理、继承、虚函数、内联、引用计数以及 stl 等一切有可能提升 C++ 效率的细节内容。最终，该书将c++性能提升的各种终极利器，完美地呈现在读者的面前！最关键的是，这本书非常薄，但是书中介绍的每个专题都是实战中常用的技术，强烈推荐。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

## 

五、C++ 工程实践

在掌握了 C++ 常用语法和语言背后的实现机制和常用惯用法后，我强烈推荐另外两本书，一本是 《C++ API 设计》 和 《大规模 C++ 程序设计》，前者从细粒度地教你在实际开发中如何设计 C++ API 接口，后者告诉你大型 C++ 程序小到单个 .h/cpp 文件如何编写，大到大型 C++ 项目如何组织的最佳实践。这两本书都是工程实践的图书，书中的技术可以实实在在地用于一线开发。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

## 

六、与 C/C++ 相关的必知必会知识

第一个基础知识是汇编，我们学习汇编不是一定要用汇编来写代码，就像我们学习 C/C++ 也不一定单纯为了面试和找工作。

  

对于 C/C++ 的同学来说，汇编是建议一定要掌握的，只有这样，你才能在书写 C++ 代码的时候，清楚地知道你的每一行C++代码背后对应着什么样的机器指令，if/for/while 等基本程序结构如何实现的，函数的返回值如何返回的，为什么整型变量的数学运算不是原子的，最终你知道如何书写代码才能做到效率最高。掌握了汇编，你可以明白，在 C++ 中，一个栈对象从构造到析构，其整个生命周期里，开发者的代码、编译器和操作系统分别做了什么。掌握了汇编，你可以理解函数调用是如何实现的，你可以理解函数的几种调用方法，为什么printf这样的函数其调用方式不能是 __stdcall，而必须是 __cdecl。掌握了汇编，你就能明白为什么一个类对象增加一个方法不会增加其实际占的内存空间。推荐的书籍是王爽老师的《汇编（第三版）》 和韩宏老师的《老码识途 从机器码到框架的系统观逆向修炼之路》 。

  

第二个基础知识是编译、链接与运行时体系知识。作为一个开发者，要清楚地知道我们写的 C/C++ 程序是如何通过预处理、编译与链接等步骤最终变成可执行的二进制文件，操作系统如何识别一个文件为可执行文件，一个可执行文件包含什么内容，执行时如何加载到进程的地址空间，程序的每一个变量和数据位于进程地址空间的什么位置，如何引用到。一个进程的地址空间有些什么内容，各段地址分布着什么内容，为什么读写空指针或者野指针会有内存问题。一个进程如何装在各个 so 或 dll 文件的，这些文件被加载到进程地址空间的什么位置，如何被执行，数据如何被交换。

  

第三个基础知识是狭义的操作系统原理。这里加上“狭义”二字是因为从广义上来讲，以上所说的内容都是操作系统原理的范畴。狭义的操作系统原理这里包括操作系统如何管理进程与线程，虚拟内存与物理内存之间的对应关系，何为内存映射文件，进程之间如何通信等等。

  

这两者推荐的书单：《程序员的自我修养》和 《Windows 核心编程》，尤其是《程序员的自我修养》，搞 C++ 开发不看此书，读尽 C++ 语言书也枉然！

  

第四个基础知识是多线程知识。严格来说，这点已经包括在第三点之中了，我之所以将其单独列出来，是因为多线程编程是我们做应用服务最常用的技术之一。最近面试过几个学历非常好的同学，对于一个进程中如果某个线程因为内存问题而退出，是否会导致整个进程退出的问题答不好，实在不应该。多线程知识其实不难学，立足于理解与实践而不是应付面试，可以学的很好。无论是 Windows 还是 Linux 操作系统，操作系统提供的线程同步对象就那么几种，Windows 常用的有临界区（关键端）、Event、互斥体、信号量等，Linux 有互斥体、信号量、读写锁、条件变量，这些知识点学过则会，不学则不会。这些线程同步原语花上几天就能搞得清楚，大多数同学不是学不会，而是不愿意学，但是偏偏喜欢在简历上写上自己熟悉多线程编程。面试的时候，被问到条件变量的虚假唤醒机制都说不清楚，非要说自己用过条件变量。这是一些同学犯的很低级的错误，如果真用过条件变量，如果不知道虚假唤醒机制，那一定写的代码是不对的。市场上目前没有任何一本图书对以上知识形成体系的介绍，当然，我的本书填补了这一空缺，你将从本书中获得从进程与线程的关系，再到常用的线程同步原语的区别与使用场景，再到线程池以及基于生产者消费者模型的消息队列，以及对协程思想介绍的相关知识。

  

掌握了常见的多线程同步原语之后，接下来可以找一些带多线程的项目去学习一下，不管是否带 UI 的都行。我推荐的一种方式是，使用 gdb 或者 Visual Studio 调试器将你需要学习的多线程程序中断下来，在多线程面板，看看这个进程一共有多少个正在运行的线程，分析每个线程的作用，然后研究下这些线程在何时何地创建的，为什么需要创建新的线程。尝试爱过几个人，面对爱情你会诚实很多；尝试研究几个多线程项目，面对多线程你会熟练许多。

> 推荐的书单 《[C++ 服务器开发精髓](https://mp.weixin.qq.com/s?__biz=MzU2MTkwMTE4Nw==&mid=2247495537&idx=1&sn=c8c68a2a7a0f6ef8881d34ed86a1d5fb&scene=21#wechat_redirect)》

第五个是网络编程，直白地说就是 Socket 编程。操作系统层面提供的 API 会在相当长的时间内保持接口不变，一旦学成，终生受用。理解和掌握常用的基础 socket API 不仅可以最大化地去定制各种网络通信框架，更不用说使用市面上流行的网络通信库了，最重要的是，它会是你排查各种网络疑难杂症坚实的技术保障。操作系统层面提供的网络模型就那么几种，无论像 Java/Go/Python 等语言如何封装，作为技术的源头，我们有什么理由不去掌握它呢？推荐的书单 《TCP/IP 网络编程》和《Linux 高性能服务器编程》 。

  

以上是基于 C++ 技术栈来说，并没有包括算法与数据结构、数据库等方面的基本功。总而言之，学习 C++ 不能只盯着 C++ 语法本身，还要熟悉 C++ 技术栈背后的一系列操作系统原理。

## 

七、C/C++ 光说不练，也学不好，要多调试哦  

光纯粹通过看书或者看代码，可能学习效果不一定好，你一定要装一个 C/C++ 语言开发工具实际去写一写 C/C++ 代码，Windows 上我推荐学习 Visual Studio 编译、调试和运行 C/C++ 代码，以下是一些步骤：

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

Visual Studio 2019 启动画面  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

创建一个 C/C++ 项目

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

选择控制台类型的程序  

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

设置工程名称

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

编译项目

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

项目编译成功

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

如果要使用 C 编译，将 .cpp 扩展名改成 .c

  

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

按 F9 添加断点  

  

  

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

启动调试  

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

学会在断点窗口查看和操作断点  

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

学会在监视窗口查看当前变量值

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

学会在函数堆栈窗口查看和切换堆栈

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

学会在内存窗口查看变量的内存值  

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

调试状态下应该学会使用的窗口

  

  

你不想只用 C 语言写一些黑洞洞的命令行程序，可以看一下《Windows 程序设计（第五版）》这本书，这本书从零教你如何编写 Windows 界面程序：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

这本书是一本经典的 Windows 编程圣经，曾经伴随着近 50 万 Windows 程序员步入编程殿堂，成长为 IT 时代的技术精英，可以这么说，中国常用的 QQ、360 安全卫士的开发者都是靠这本书启蒙的。

  

如果你在 Linux 下使用 C 语言编写服务程序，要学会 gcc、Makefile、CMake 等工具，推荐一下 《GNU Make 手册》：

> 链接: https://pan.baidu.com/s/11NZs2IPi8sMZu4n8LRgttA
> 
> 提取码: hcj6

和 《CMake 最佳实践》这两本书：

> 链接: https://pan.baidu.com/s/1flVSq6bwu0n-Y7vXheVpkw
> 
> 提取码: 44ta

编译好程序之后，你可以使用 GDB 进行调试，GDB 掌握常用命令即可，这个很容易学，实际上手操作一下，几分钟就学会了，GDB 常用命令：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

GDB 常用命令

  

学习 GDB 推荐《Debugging with GDB》一书：

> 链接: https://pan.baidu.com/s/1e3toHeLoQrnEr1KS7Rhn7g
> 
> 提取码: h0d3

  

## 如果在学习和研究上述开源项目的过程中遇到问题，可以加入加入 **高性能服务器开发微信交流群** 进行求助，先加微信 **easy_coder**，备注"加微信群"，拉你入群，备注不对不加哦。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

学好 C/C++ 并非一朝一夕之功，需要不断坚持加努力，一旦学成，奇妙无穷，成为技术卷王！！

  

八、福利时间

  

我知道我的很多读者都是 C/C++ 开发者，为了感谢大家的支持，我自费购买 了 6 本讲解现代 C++ 特性的 C++ 新书《**现代 C++ 语言核心特性解析**》送给大家，这是今年刚出版不久的新书。

  

抽奖参与方式：

![](http://mmbiz.qpic.cn/mmbiz_png/OIdl32LQlqpmKgWE0wRXCiafyf8F0GZbX5ldCfUdmYLpngEDp1QfSD6RsuukHIq0llJ8yiayfCqEFibVX9n6OQwTQ/300?wx_fmt=png&wxfrom=19)

**程序员小方**

技术，生活，编码，加班，读书学习，这里是程序员小方的 IT 生活。

21篇原创内容

公众号

关注后回复『 小方来本C++ 』参与抽奖  

  

关注『**程序员小方**』公众号，回复『 小方来本C++ 』即可参与抽奖。  

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

  

  

原创不易，如果觉得有用，请给我点个赞呗～

阅读 8455

​

写留言

**留言 42**

- 小罗罗
    
    2021年10月27日
    
    赞2
    
    收藏
    
- 禾田里的人
    
    2021年10月27日
    
    赞1
    
    一定要学起来
    
- 李玉森
    
    2021年10月27日
    
    赞1
    
    干活满满
    
- KC
    
    2021年10月27日
    
    赞1
    
    奥利给![[强]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)
    
- A-代码牛仔-CCFOJ.COM
    
    2021年10月27日
    
    赞1
    
    小方来本C++
    
    CppGuide
    
    作者2021年10月27日
    
    赞
    
    在 公众号：程序员小方 中回复这个关键字哦。
    
- 光子错
    
    2021年10月27日
    
    赞1
    
    头发太密了
    
- 风岚
    
    2021年10月27日
    
    赞1
    
    楼主好人
    
- 我爱熊出没
    
    四川2022年7月15日
    
    赞
    
    万分感谢，准研二立个flag
    
    CppGuide
    
    作者2022年7月16日
    
    赞
    
    加油。
    
- 二十三
    
    2022年2月16日
    
    赞
    
    经常在食堂看到张哥，有几次吃饭排队就站我前面，没敢打招呼，怕认错人![[捂脸]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)
    
- 大鹅
    
    2021年11月10日
    
    赞
    
    硬核资料
    
    CppGuide
    
    作者2021年11月10日
    
    赞
    
    点赞在看转发走起来![[好的]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)
    
- 天涯
    
    2021年11月4日
    
    赞
    
    大佬，加你微信兑奖，同意下啊![[呲牙]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)
    
    CppGuide
    
    作者2021年11月4日
    
    赞
    
    OK
    
- Guardian
    
    2021年10月29日
    
    赞
    
    小方来本C++ 小方来本C++ 小方来本C++
    
    CppGuide
    
    作者2021年10月30日
    
    赞
    
    ![[流汗]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)
    
- 东方神韵
    
    2021年10月29日
    
    赞
    
    谢谢分享
    
- 甘草
    
    2021年10月28日
    
    赞
    
    群主的照片感觉很面熟，难道在哪里见过
    
- 于佑飞
    
    2021年10月28日
    
    赞
    
    大佬的头发还是很茂密
    
- 大地
    
    2021年10月28日
    
    赞
    
    棒呆了
    
- alice
    
    2021年10月28日
    
    赞
    
    感谢楼主！
    
- 一去、二三里
    
    2021年10月27日
    
    赞
    
    厉害![[强]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)
    
- Hide on bug
    
    2021年10月27日
    
    赞
    
    看到了gdb的书，太赞了
    
- linkinghack
    
    2021年10月27日
    
    赞
    
    已购买C++服务器开发精髓![[抱拳]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)
    
- 陈琦
    
    2021年10月27日
    
    赞
    
    写的很棒
    
- 坤超
    
    2021年10月27日
    
    赞
    
    书本内容很实用，群主也真帅![[呲牙]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)
    
- 全宏
    
    2021年10月27日
    
    赞
    
    非常棒
    
- 叶孤洋
    
    2021年10月27日
    
    赞
    
    封面帅哥是本尊吗![[呲牙]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)
    
    CppGuide
    
    作者2021年10月27日
    
    赞
    
    你猜![😄](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)
    
- 伊牙牙嘿哈
    
    2021年10月27日
    
    赞
    
    大佬爆照了
    
- Yohua
    
    2021年10月27日
    
    赞
    
    前辈十八般武艺，样样齐全，令人佩服呀！![[强]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)
    
- 不瞒
    
    2021年10月27日
    
    赞
    
    方哥，你用的什么电脑 什么配置哇
    
    CppGuide
    
    作者2021年10月27日
    
    赞
    
    中配，可以带动photoshop就可以。
    
- CHH
    
    2021年10月27日
    
    赞
    
    ![[色]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)看得口水都流了
    
- 旅客
    
    2021年10月27日
    
    赞
    
    号主的文章干货满满，谢谢号主。能请教您一下如何保养头发吗![[捂脸]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)
    
    CppGuide
    
    作者2021年10月27日
    
    赞
    
    这。。。。。。
    
- 若水
    
    2021年10月27日
    
    赞
    
    ![[色]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)![[色]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)![[色]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)![[色]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)![[色]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)![[色]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)
    
- 金色熊族
    
    2021年10月27日
    
    赞
    
    太有用了，感谢！
    
- Ivy
    
    2021年10月27日
    
    赞
    
    真是大佬！摩拜
    
- 成熟的男人孤独的狼
    
    2021年10月27日
    
    赞
    
    我太爱了
    
- Jemmy
    
    2021年10月27日
    
    赞
    
    大佬爆照了 帅 ![[Respect]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)
    

已无更多数据

[](javacript:;)

![](http://mmbiz.qpic.cn/sz_mmbiz_png/GSweNIrkicYvM1mIwPctlYONEDKJwUfRZ57uAkVR59MpX1cVnmmnyPZ5O9OCuys78Sy6fOEncwfgWpgCo9Tibeag/300?wx_fmt=png&wxfrom=18)

CppGuide

151248

42

写留言

**留言 42**

- 小罗罗
    
    2021年10月27日
    
    赞2
    
    收藏
    
- 禾田里的人
    
    2021年10月27日
    
    赞1
    
    一定要学起来
    
- 李玉森
    
    2021年10月27日
    
    赞1
    
    干活满满
    
- KC
    
    2021年10月27日
    
    赞1
    
    奥利给![[强]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)
    
- A-代码牛仔-CCFOJ.COM
    
    2021年10月27日
    
    赞1
    
    小方来本C++
    
    CppGuide
    
    作者2021年10月27日
    
    赞
    
    在 公众号：程序员小方 中回复这个关键字哦。
    
- 光子错
    
    2021年10月27日
    
    赞1
    
    头发太密了
    
- 风岚
    
    2021年10月27日
    
    赞1
    
    楼主好人
    
- 我爱熊出没
    
    四川2022年7月15日
    
    赞
    
    万分感谢，准研二立个flag
    
    CppGuide
    
    作者2022年7月16日
    
    赞
    
    加油。
    
- 二十三
    
    2022年2月16日
    
    赞
    
    经常在食堂看到张哥，有几次吃饭排队就站我前面，没敢打招呼，怕认错人![[捂脸]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)
    
- 大鹅
    
    2021年11月10日
    
    赞
    
    硬核资料
    
    CppGuide
    
    作者2021年11月10日
    
    赞
    
    点赞在看转发走起来![[好的]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)
    
- 天涯
    
    2021年11月4日
    
    赞
    
    大佬，加你微信兑奖，同意下啊![[呲牙]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)
    
    CppGuide
    
    作者2021年11月4日
    
    赞
    
    OK
    
- Guardian
    
    2021年10月29日
    
    赞
    
    小方来本C++ 小方来本C++ 小方来本C++
    
    CppGuide
    
    作者2021年10月30日
    
    赞
    
    ![[流汗]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)
    
- 东方神韵
    
    2021年10月29日
    
    赞
    
    谢谢分享
    
- 甘草
    
    2021年10月28日
    
    赞
    
    群主的照片感觉很面熟，难道在哪里见过
    
- 于佑飞
    
    2021年10月28日
    
    赞
    
    大佬的头发还是很茂密
    
- 大地
    
    2021年10月28日
    
    赞
    
    棒呆了
    
- alice
    
    2021年10月28日
    
    赞
    
    感谢楼主！
    
- 一去、二三里
    
    2021年10月27日
    
    赞
    
    厉害![[强]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)
    
- Hide on bug
    
    2021年10月27日
    
    赞
    
    看到了gdb的书，太赞了
    
- linkinghack
    
    2021年10月27日
    
    赞
    
    已购买C++服务器开发精髓![[抱拳]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)
    
- 陈琦
    
    2021年10月27日
    
    赞
    
    写的很棒
    
- 坤超
    
    2021年10月27日
    
    赞
    
    书本内容很实用，群主也真帅![[呲牙]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)
    
- 全宏
    
    2021年10月27日
    
    赞
    
    非常棒
    
- 叶孤洋
    
    2021年10月27日
    
    赞
    
    封面帅哥是本尊吗![[呲牙]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)
    
    CppGuide
    
    作者2021年10月27日
    
    赞
    
    你猜![😄](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)
    
- 伊牙牙嘿哈
    
    2021年10月27日
    
    赞
    
    大佬爆照了
    
- Yohua
    
    2021年10月27日
    
    赞
    
    前辈十八般武艺，样样齐全，令人佩服呀！![[强]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)
    
- 不瞒
    
    2021年10月27日
    
    赞
    
    方哥，你用的什么电脑 什么配置哇
    
    CppGuide
    
    作者2021年10月27日
    
    赞
    
    中配，可以带动photoshop就可以。
    
- CHH
    
    2021年10月27日
    
    赞
    
    ![[色]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)看得口水都流了
    
- 旅客
    
    2021年10月27日
    
    赞
    
    号主的文章干货满满，谢谢号主。能请教您一下如何保养头发吗![[捂脸]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)
    
    CppGuide
    
    作者2021年10月27日
    
    赞
    
    这。。。。。。
    
- 若水
    
    2021年10月27日
    
    赞
    
    ![[色]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)![[色]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)![[色]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)![[色]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)![[色]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)![[色]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)
    
- 金色熊族
    
    2021年10月27日
    
    赞
    
    太有用了，感谢！
    
- Ivy
    
    2021年10月27日
    
    赞
    
    真是大佬！摩拜
    
- 成熟的男人孤独的狼
    
    2021年10月27日
    
    赞
    
    我太爱了
    
- Jemmy
    
    2021年10月27日
    
    赞
    
    大佬爆照了 帅 ![[Respect]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)
    

已无更多数据