
原创 果冻虾仁 编程往事

 _2022年01月31日 17:01_

C++11标准颁布距今已经十年了，在这个标准中引入了很多新的语言特性，在进一步强化C++的同时，也劝退了很多人，其中就包含右值引用。

> _T&& Doesn’t Always Mean “Rvalue Reference”_  
> by Scott Meyers

Scott Meyers曾经说过：T&&并不总是表示右值引用（rvalue reference）。

## 作为函数参数的&&

没错。&& 这两个符号，可能是初学C++或者C++11的时候，把很多人劝退的一个点。但想理清其实也不难。

首先在**非模板函数中**，&&肯定是表示右值引用，所以只能接收右值类型的参数。
```cpp
`class A {       ...   };   void foo(A&& a) {       ...   }   `
```
这个foo函数只能接受如下的右值参数。

`foo(A{});      A a;   foo(std::move(a));      A get_a() {       A a;       ...       return a;   }   foo(get_a());   `

但是这样调用则不正确，会编译失败：

    `A a1;       A& ar = a1;       foo(ar); // ERROR`

但是，在模板函数中&&则并不表示右值引用（**rvalue rference**）。比如：

`template <typename T>   void bar(T&& t) {       ...   }   `

这个T&&在C++标准中被称为**forwarding reference，** 译作**转发引用**， 俗称（非官方习惯但流行的叫法）是 **universal reference**，经常被译作**万能引用**。顾名思义，它所能接收的参数不仅仅是右值，左值也可以，故称万能。所以题主问题中的是T&&并不是右值引用。

我上文提到的模板函数bar()，可以接收如下类型的的参数（各种类型都可以！）。比如：

    `bar(A{});       A a;       bar(move(a));       bar(get_a());          A a1;       A& ar1 = a1;       bar(ar1);          const A& ar2 = a1;       bar(ar2);`

但是模板函数中的&&也不全是转发引用。比如：

`template<typename T>   void bar(vector<T>&& tv) {       ...   }   `

这里的参数tv就只能接收右值类型的参数。

又比如：

`template<typename T>   void bar(const T&& v) {       ...   }   `

加了const限制后，这里的&&也是右值引用，只能传入右值……

相信你已经看懂其中的差别。

## 接收返回值的&&

&&迷惑的另一个地方不仅在于上面介绍的，作为参数的时候。在接收函数返回值的时候，也有歧义，常常让人迷惑。

`class A {       ...   };   A&& a = test1(); // 这个&&表示的是右值引用   auto&& a = test2(); // 这个&&表示的是也是转发引用   `

在auto和&&联用的时候，它也是转发引用（万能引用），故而可以接收各种类型的函数参数。

但是如果是显式指定了类型（比如A）然后和&&联用，则只表示右值引用，只能接收右值类型的参数！

怎么样，劝退了没。

## 往期推荐

[![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)](https://mp.weixin.qq.com/s?__biz=MjM5NDIyMjI3OQ==&mid=2649005239&idx=1&sn=77205c024713efb3de9bda291e49cf88&scene=21#wechat_redirect)

C++ Trick：什么时候需要前置声明？

[![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)](https://mp.weixin.qq.com/s?__biz=MjM5NDIyMjI3OQ==&mid=2649002869&idx=1&sn=8a4cbdcab708fda4db74c83f7cc98625&scene=21#wechat_redirect)

C++ Trick：小心，子类隐藏父类成员函数

[![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)](https://mp.weixin.qq.com/s?__biz=MjM5NDIyMjI3OQ==&mid=2649002982&idx=1&sn=526b28505d07f61ee7b45785710101a3&scene=21#wechat_redirect)

C++ Trick：不使用friend，怎么访问private成员？

[![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)](https://mp.weixin.qq.com/s?__biz=MjM5NDIyMjI3OQ==&mid=2649002852&idx=1&sn=c4b40a66577edeaa174788ec0e56c029&scene=21#wechat_redirect)

C++ Trick：宏函数与模板类之殇

  

![](https://mmbiz.qlogo.cn/mmbiz_jpg/TQJtVgkziaZ7vvd8CiapUQMkbN7tUE9F2FEYO3VpfIuWQyuxEHwvqEOwVJrfx7xRr5XaAInN7Mk7TwgmMJwEGcSg/0?wx_fmt=jpeg)

果冻虾仁

![赞赏二维码](https://mp.weixin.qq.com/s?__biz=MjM5NDIyMjI3OQ==&mid=2649005604&idx=1&sn=32c169d3e5e714fd3ca1b218fd50b061&chksm=be9b278c89ecae9afcf94b2ae060dcf3d2412f9a0828d4492a6f90ebb82dc05b7984d30208db&mpshare=1&scene=24&srcid=0131FQZ5L9bLbb0UsLCNfEX5&sharer_sharetime=1643621269807&sharer_shareid=5fb9813bfe9ffc983435bfc8d8c5e9ca&key=daf9bdc5abc4e8d06f6644fc2810780514f4126c8af2788428db6a89527a60bfad788c80ade333b83737859daec3dd0f78948a5ad93bc0f591ef805abfe47d71138b1d6441e7bda4d00ba7a650cf7a1d72d9e52b3f8b7afd2ea52c7d59911cc6c3d941adb76df8d44e80f1c97c935e7211da356b2325995449f7a87cbaa97723&ascene=0&uin=MTEwNTU1MjgwMw%3D%3D&devicetype=Windows+11+x64&version=63090b19&lang=zh_CN&countrycode=CN&exportkey=n_ChQIAhIQwtPAZIuU8mt6QurF0p4dnBLmAQIE97dBBAEAAAAAAC0oBtLWeU0AAAAOpnltbLcz9gKNyK89dVj0jXj6jVHgTk19HqdCQ66zR6XEZqyifXimMuQdmweg598pTh%2FCFnYf5FjIuIpOhW9eTScdpFR8lyz0c2NyGLCX6FDlMmRh%2FKUqfzKXvhLmLT5pNsKRGx2KbJyfxRf9T6pEzLou0Y5MrswKkVpQPPyQcJJD6nfO9t68bC34ySm6aQPrglpsHeALcDUotHnsrKL%2BNdQr%2Fa35t3NColQe7WU%2FzOoEYvdQp%2BBTwHfSBIJ7co7BvWgbn5TPO3Yq%2FGNFOckH&acctmode=0&pass_ticket=ul9o6P8BOxSbNva3%2F7WCsX4RXZYg4FVzp6vWeOB785GcO9Xs2jvebaaXRFxoVdVk&wx_header=1&fasttmpl_type=0&fasttmpl_fullversion=7350504-zh_CN-zip&fasttmpl_flag=1)喜欢作者

C++46

C++1115

右值引用1

万能引用1

C++ · 目录

上一篇2021年我读过的计算机书籍推荐下一篇Hands-On Design Patterns with C++ 讲义 第0章：开篇

阅读原文

阅读 753

​

写留言

**留言 6**

- 勇者
    
    2022年1月31日
    
    赞3
    
    大佬为啥不给讲讲为什么普通函数形参只能传递右值引用，模板函数参数什么情况下传递右值，什么情况下传递左值
    
- 静心得意
    
    北京2022年5月13日
    
    赞
    
    大佬文章长度正好，还能把问题讲清楚，好喜欢
    
- 空
    
    2022年2月1日
    
    赞
    
    新年快乐
    
    编程往事
    
    作者2022年2月1日
    
    赞
    
    谢谢![[呲牙]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
- 金色熊族
    
    2022年1月31日
    
    赞
    
    难得大佬年二十九还在写，学习了，期待大佬更多高质量文章
    
    编程往事
    
    作者2022年1月31日
    
    赞
    
    之前在知乎写的![[捂脸]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif) 同步了一下
    

已无更多数据

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/hQZ4NEZ2sic5WOaB0JA1LvD2OpPWttPuvBR21zFyLcgUnwGRzSSp1tpj4jOY2KnSia3xsUzeevb5tibl8ajStOZlg/300?wx_fmt=png&wxfrom=18)

编程往事

744

6

写留言

**留言 6**

- 勇者
    
    2022年1月31日
    
    赞3
    
    大佬为啥不给讲讲为什么普通函数形参只能传递右值引用，模板函数参数什么情况下传递右值，什么情况下传递左值
    
- 静心得意
    
    北京2022年5月13日
    
    赞
    
    大佬文章长度正好，还能把问题讲清楚，好喜欢
    
- 空
    
    2022年2月1日
    
    赞
    
    新年快乐
    
    编程往事
    
    作者2022年2月1日
    
    赞
    
    谢谢![[呲牙]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
- 金色熊族
    
    2022年1月31日
    
    赞
    
    难得大佬年二十九还在写，学习了，期待大佬更多高质量文章
    
    编程往事
    
    作者2022年1月31日
    
    赞
    
    之前在知乎写的![[捂脸]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif) 同步了一下
    

已无更多数据