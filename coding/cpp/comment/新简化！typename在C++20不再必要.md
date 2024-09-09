

Original 里缪 CppMore

 _2022年01月09日 20:50_

![Image](https://mmbiz.qpic.cn/mmbiz_jpg/9XBBCfGaPEkGT3rUlvw111IJvHI8laOlSNKMkxCxDtpreEdD3LF4bolQDwRZrticPwOiad1GbhAdJx9Ev70gZDSg/640?wx_fmt=jpeg&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

  

根据提案P0634-Down with typename，C++20之后typename在有些地方不再必要。

原文主要内容如下：

> If X::Y — where T is a template parameter — is to denote a type, it must be preceded by the keyword typename; otherwise, it is assumed to denote a name producing an expression. There are currently two notable exceptions to this rule: base-specifiers and mem-initializer-ids. For example:
> 
>   
> template<class T>  
> struct D: T::B { // No typename required here.  
> };
> 
>   
> Clearly, no typename is needed for this base-specifier because nothing but a type is possible in that context. 

简言之，以前若需要使用模板中的嵌套类型，必须在前面加上typename以表示这是一个类型。

而编译器供应商很快意识到在许多语境下其实他们知道指定的是否是类型，于是提议将typename变为可选项，只在编译器无法确定的语境下typename才是必选项。

这项提议内容最终加入了C++20，因此，C++20编写模板代码将会进一步简化。

举个例子，C++20可以这样写代码：

 1#include <iostream>  
 2  
 3struct S {  
 4    using X = int;  
 5    using R = char;  
 6};  
 7  
 8  
 9// 类模板  
10template<class T,  
11         auto F = typename T::X{}>    // 需要typename  
12struct Foo {         
13    using X2 = T::X;                  // 不需要typename  
14    typedef T::X X3;                  // 不需要typename                   
15    T::X val;                         // 不需要typename  
16  
17    T::R f(T::X x) {                  // 不需要typename  
18        typename T::X i;              // 需要typename  
19        return static_cast<T::R>(x);  // 不需要typename  
20    }                                   
21  
22    auto g() -> T::R {                // 不需要typename  
23    }                                   
24  
25    template<class U = T::X>          // 不需要typename  
26    void k(U);                          
27};                                      
28  
29  
30// 函数模板                             
31template<class T>                       
32T::R                                  // 不需要typename  
33f(typename T::X x)                    // 需要typename  
34{                                       
35    using X2 = T::X;                  // 不需要typename  
36    typedef typename T::X X3;         // 需要typename  
37    typename T::X val;                // 需要typename  
38  
39    typename T::R f();                // 需要typename  
40    auto g() -> T::R;                 // 不需要typename  
41    void k(typename T::X);            // 需要typename  
42  
43    auto lam = [](T::X) {};           // 不需要typename  
44  
45    return static_cast<T::R>(x);      // 不需要typename  
46}  
47  
48int main()  
49{  
50    Foo<S> foo;  
51    f<S>(65);  
52  
53    return 0;  
54}  

示例基本列举了大多数使用typename的情境，其中许多情境下typename已不再需要。

这里解释一点，为何在类模板与函数模板中，有些情境相同，但是typename使用规则却并不相同呢？

回答这个问题，再次引用一下[Understanding variadic templates](http://mp.weixin.qq.com/s?__biz=MzUxOTQ4NjIzNw==&mid=2247484890&idx=1&sn=11894b324373b22733fd3f49cfb3176e&chksm=f9f9aba8ce8e22be8898b0ba3ab7bf4d3b3b6b8acbad9db77babe5a95f0c491ad14ac07b192d&scene=21#wechat_redirect)，在这篇文章中我抛出了一个观点：函数模板处理的是数值，类模板处理的是类型。  

在「类作用域」和「函数作用域」下编译器对于嵌套类型的解析形式是不一样的，前者解析为类型，后者解析为非类型。因此，在函数模板里许多地方依旧需要使用typename，而在类模板里则不必要。  

总之就是，仍旧需要使用typename的地方都可能存在歧义，所以需要借助typename来标识其为类型。  

此外，这里还有一个有趣的点。

Modern C++中建议使用using代替typedef来定义类型别名，以前二者除了写法似乎并不区别。

但是现在，使用using定义嵌套类型的别名时不用指定typename，而typedef仍然需要。因此，using代替typedef绝对是一种比较好的做法。

  

![](https://mmbiz.qlogo.cn/mmbiz_jpg/mUBQ21ETsDA7DH60GIcuqUvvWgCWsibCibLv0Zos5hpvrZTqZ4lPh0CttCd9DxkpwOcOHM8UqPviaC0P7THN3KF4g/0?wx_fmt=jpeg)

里缪

![赞赏二维码](https://mp.weixin.qq.com/s?__biz=MzUxOTQ4NjIzNw==&mid=2247487128&idx=1&sn=3014247aead6101e4cd6770390a495de&chksm=f9f9a0eace8e29fc5a53706cd273ee64be5034dd0ce205ae0a18ddb93871505c735fd3963cee&mpshare=1&scene=24&srcid=0109ITGHuSoW1g9x0CkFtEdK&sharer_sharetime=1641740657904&sharer_shareid=5fb9813bfe9ffc983435bfc8d8c5e9ca&key=daf9bdc5abc4e8d06773cb69d1d1b33c0228a4f0e165c75b7c07a913696829dd8ddd0d24ef1ca4f8ea28bcecdaecf8caacc888105cdeddb05d611fc9c28bcfab228e95fc8acb8c3789c699698f8bd7841ef6514a64e33de655c17939daf0c3ff11aabb8a43dd51dc6753c620baae0898942b7cf70c3a9e0036a02f94e37400d7&ascene=14&uin=MTEwNTU1MjgwMw%3D%3D&devicetype=iMac+MacBookAir10%2C1+OSX+OSX+14.6.1+build(23G93)&version=13080710&nettype=WIFI&lang=en&session_us=gh_ee7428b045cc&countrycode=CN&fontScale=100&exportkey=n_ChQIAhIQ%2Fk8YnoPQFQYbXSE60acJ3RKUAgIE97dBBAEAAAAAANeQIpCAnzYAAAAOpnltbLcz9gKNyK89dVj0OpfQxLprw6klVkGI%2F9SSEfg7gzNJkFvOgederimEfJ0uj9vE84CZQUPunAePfRO%2F%2FDLXYn9lOAb2QQb772VlA5zPCVJMEBI9SQyj8i6ZRk7ZcESHZmYTI699BE2jfS0CWz6UBMjJvIejqRrQ2tS7vKnTZUFG%2BbhRm4OctdYbITPu%2FFbiku8QWUEShKd8CQRS3QSs98BM7AXgC67JW3BSnilF9FjEmk5T0SULs%2FI63vC97AkrTr72CGurLSqYeUYi2XsLQS827QtrUHx5%2F3g8VLRvIPVF4jJWX6LyhdrFniahsZZepEva6C6Lmexfaw%3D%3D&acctmode=0&pass_ticket=Tbpr3Gmj%2BaAQBppuhIr568eheleV6sfli%2BsAEEbemQYOygbTXl2oNdKMLGIIoR5V&wx_header=0)Like the Author

C++2013

C++20 · 目录

上一篇泛型Lambda，如此强大！下一篇Using C++20 Formatting Library

Reads 605

​

People who liked this content also liked

CMake入门教程

我关注的号

程序喵大人

不喜欢

不看的原因

OK

- 内容低质
- 不看此公众号内容

![](https://mmbiz.qpic.cn/sz_mmbiz_jpg/Av3j9GXyMmGWksd8DfWNZ1JzyN0v7ibEBpevb5SmwBiaoUcGBZZWqSAylJcj0GTgOJb510pTfIGOyNwwYPStVzBg/0?wx_fmt=jpeg)

Linux高性能编程_时间轮

物联网心球

不喜欢

不看的原因

OK

- 内容低质
- 不看此公众号内容

![](https://mmbiz.qpic.cn/mmbiz_jpg/jvHp1D7f9iczO2KfSbiaKiccbc7I9nAqibcwFszuf8cbm1XN5rckMGxKviclWscPjTgfickNiaicGE36ma0lxKQMjk839g/0?wx_fmt=jpeg&tp=wxpic)

Shell中的四种变量

硅特嵌入式

不喜欢

不看的原因

OK

- 内容低质
- 不看此公众号内容

![](https://mmbiz.qpic.cn/sz_mmbiz_jpg/raSIvamLFr63E1rHdOcEgyRib9zI3jPBUPMaW4xNDEfstDjF8iaIhDwRvw9Su1C0MFWPU880RX8nEhXQ7Y1jwxeQ/0?wx_fmt=jpeg)

Comment

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/9XBBCfGaPEly5AtcdkIKicpv55w2VlBiaQ7TxtJneeDmgO3mxz4onND7rW2UjfSib9KC1FtBB4U6TupnBMfenoCgQ/300?wx_fmt=png&wxfrom=18)

CppMore

813

Comment

Comment

**Comment**

暂无留言