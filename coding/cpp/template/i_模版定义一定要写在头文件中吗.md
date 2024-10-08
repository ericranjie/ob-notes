原创 程序喵大人 程序喵大人
_2021年12月20日 08:24_

大家在使用C++代码时或多或少都会使用到模板，使用模板时应该都是把定义放在了头文件中，因为放在源文件中定义会编译失败。

那问题来了，模板中的函数定义一定要写在头文件中吗？

先说结论：不一定要放在头文件中，定义也可以放在源文件中，但操作起来还是有点麻烦的。

继续往下看看

先看一段正常的模板代码：

```cpp
// template.h  
#include <iostream>  
  
template <typename T>  
struct TemplateTest {  
    T value;  
    void func();  
};  
  
template <typename T>  
void TemplateTest<T>::func() {  
    std::cout << typeid(T).name() << std::endl;  
}  
  
// test_template.cc  
#include "template.h"  
  
int main() {  
    TemplateTest<int> test;  
    test.func();  
    return 0;  
}
```

这段代码没啥毛病，因为实现放在了头文件中，也会正常输出。

如果我把函数定义放在源文件中呢，会怎么样？

```cpp
// template.h  
#include <iostream>  
  
template <typename T>  
struct TemplateTest {  
    T value;  
    void func();  
};  
  
// template.cc  
template <typename T>  
void TemplateTest<T>::func() {  
    std::cout << typeid(T).name() << std::endl;  
}  
  
// test_template.cc  
#include "template.h"  
  
int main() {  
    TemplateTest<int> test;  
    test.func();  
    return 0;  
}
```

嗯，不出意外，编译报错了，报了个没有某个函数实现的error：

```cpp
/tmp/ccPghOjU.o: In function `main':  
test_template.cc:(.text+0x1f): undefined reference to `TemplateTest<int>::func()'  
collect2: error: ld returned 1 exit status
```

为什么没有此函数定义？

先补充个基础知识，模板的本质。本质其实就是类型泛化，可以用一个T代替多种类型，对于上面的模板，假如有此种使用：

```cpp
TemplateTest<int> test;
```

那模板最终可能变成这样：

```cpp
struct TemplateTest_int {  
    int value;  
    void func() {  
        std::cout << typeid(int).name() << std::endl;  
    }  
};
```

如果有这两种使用：

```cpp
TemplateTest<int> test;  
TemplateTest<float> test;
```

那模板最终可能会变成这样：

```cpp
struct TemplateTest_int {  
    int value;  
    void func() {  
        std::cout << typeid(int).name() << std::endl;  
    }  
};  
  
struct TemplateTest_float {  
    float value;  
    void func() {  
        std::cout << typeid(float).name() << std::endl;  
    }  
};
```

模板最终会展开成什么样，取决于用户是怎么使用它的。

那回到上面的问题，为什么把定义放在源文件中，编译就失败了呢，因为每个源文件的编译都是独立的，尽管在test_template.cc进行了TemplateTesttest的使用，但是在template.cc中却感知不到，所以也就没法定义相关的实现。

思路来了，只需要让template.cc中感知到T有int类型的情况，那编译应该就没问题。这里有个语法，叫模板实例化，像这样：

```cpp
template struct TemplateTest<int>;
```

把这行代码放在template.cc中：

```cpp
// template.cc  
#include "template.h"  
  
template <typename T>  
void TemplateTest<T>::func() {  
    std::cout << typeid(T).name() << std::endl;  
}  
  
template struct TemplateTest<int>;
```

整个代码的编译就没得问题了，通过这种方式即可以实现模板函数声明与实现的分离。

这也印证了上面的结论。

这里我再抛出**几个问题**，大家可以讨论讨论：

- 模板的使用是否会导致代码段体积增大？怎么解决？
- 模板的函数定义放在了头文件中，好多个源文件都include此头文件，是否会导致函数的multi definition类型的链接报错？实际使用中貌似没有报错，为什么？大家有想过吗？

______________________________________________________________________

往期推荐

为什么建议少用if语句！

\](http://mp.weixin.qq.com/s?\_\_biz=MzkyODE5NjU2Mw==&mid=2247491362&idx=1&sn=0cafab8395443227af0879160dcf90a0&chksm=c21d2d9ef56aa4881bc639ec5025ec5ff2bc12db424f9b7a2266505cb66e72311f48de86fa99&scene=21#wechat_redirect)

\[

工作8年，我决定离开上海

\](http://mp.weixin.qq.com/s?\_\_biz=MzkyODE5NjU2Mw==&mid=2247491347&idx=1&sn=0aeb90e3f30eb00f3a2e19d81809a401&chksm=c21d2daff56aa4b9c7e1f0d3c22ca52b3b4cfb7769531fc13393ccd884d351d94dcff59382b0&scene=21#wechat_redirect)

\[

四万字长文，这是我见过最好的模板元编程文章！

\](http://mp.weixin.qq.com/s?\_\_biz=MzkyODE5NjU2Mw==&mid=2247491328&idx=1&sn=a9e4147a1ffe5de9c8a7f0c8d5377405&chksm=c21d2dbcf56aa4aa5d5e60e3a4ecece847cbcb777a97f437010b1300c02ab36f9ecc01dca65d&scene=21#wechat_redirect)

\[

如何正确的理解指针和结构体指针、指针函数、函数指针这些东西？

\](http://mp.weixin.qq.com/s?\_\_biz=MzkyODE5NjU2Mw==&mid=2247491314&idx=1&sn=d62d17d2ba885e310fd6d9d29eee5159&chksm=c21d2c4ef56aa558652b15c1dea02aff9d84f322988633ed2add536952a64a30ae622323c749&scene=21#wechat_redirect)

\[

C++为什么要弄出虚表这个东西？

\](http://mp.weixin.qq.com/s?\_\_biz=MzkyODE5NjU2Mw==&mid=2247491276&idx=1&sn=7301628743ca8baaaa56e9958808e72f&chksm=c21d2c70f56aa5668bf00c481548527c4cb4c786197d5b663908aec211dae0c8750eb00d2d34&scene=21#wechat_redirect)

\[

研究了一波RTTI，再介绍软件开发的201个原则，文末再送6本书

\](http://mp.weixin.qq.com/s?\_\_biz=MzkyODE5NjU2Mw==&mid=2247491275&idx=1&sn=c2902c6a544a7d04ecec8d08b465ade8&chksm=c21d2c77f56aa561b4263d8d0ea4ff59414d71c62d7fc36cf9e51082a5e793802a8185715c63&scene=21#wechat_redirect)

\[

【性能优化】高效内存池的设计与实现

\](http://mp.weixin.qq.com/s?\_\_biz=MzkyODE5NjU2Mw==&mid=2247491272&idx=2&sn=a301597e71c3945627fda5429da2c1b1&chksm=c21d2c74f56aa5627a855a20ab64afdd32c792239a0c895e3505e5bee5d61dcaebc8f0313e42&scene=21#wechat_redirect)

\[

网传阿里裁员2万人…

\](http://mp.weixin.qq.com/s?\_\_biz=MzkyODE5NjU2Mw==&mid=2247491211&idx=1&sn=d4afcf8f4eda5be0ac37f73979e6ea87&chksm=c21d2c37f56aa521ae3e64e1e6a069ab6f71c7fc21ff32022a002e1982f9a170914cec281737&scene=21#wechat_redirect)

\[

腾讯 C++ 笔试/面试题及答案

\](http://mp.weixin.qq.com/s?\_\_biz=MzkyODE5NjU2Mw==&mid=2247491181&idx=1&sn=c69456ab8c2621b1ca6c533d4159f3cf&chksm=c21d2cd1f56aa5c7745c0d48b6a653bf11f5024be6cf8eeb3380786ae65a4ec38af1e7de844f&scene=21#wechat_redirect)

\[

介绍一个C++中非常有用的设计模式

\](http://mp.weixin.qq.com/s?\_\_biz=MzkyODE5NjU2Mw==&mid=2247491077&idx=1&sn=f78a45bd27b6225cfd989c51b3bb4485&chksm=c21d2cb9f56aa5afc2c967c887a6cc61a82a53af429332f3518521048545148d09e410444760&scene=21#wechat_redirect)

![](https://mmbiz.qlogo.cn/mmbiz_jpg/cVRiaDiak22mwiavM3vMdDewC7JicatYWEHr0HouU30MEjU4zHYYeQf1TwDNtAYTRm44p9DciaGgrxJqiaL4unX81jnw/0?wx_fmt=jpeg)

程序喵大人

![赞赏二维码](https://mp.weixin.qq.com/s?__biz=MzkyODU5MTYxMA==&mid=2247493343&idx=1&sn=b6045d2248b6e148b9bb0314ed0a726b&source=41&key=daf9bdc5abc4e8d05fb9cd6e3805b2dac2c53237db6747e589fdc6edc1077c11cb73d18ab675f5f16a43f33d03fa10b95e62001d0ccdc4e22cd97037b9d14dd3188382559232070824e71945c7e24145041086958bc1a9ca92198c8b744ea9d8758ad1abe4ba3ef7f21c8db0209401ca8560056d8d530b6458193b799053b887&ascene=0&uin=MTEwNTU1MjgwMw%3D%3D&devicetype=Windows+11+x64&version=63090b19&lang=zh_CN&countrycode=CN&exportkey=n_ChQIAhIQXbVeqKt%2F48oKlc8yMi4fzBLmAQIE97dBBAEAAAAAABWTNguml7cAAAAOpnltbLcz9gKNyK89dVj0GEVgWisS%2FzE05%2BCRRrvgym9fQxUsihAoRVM2vjQyQwhj2ISZTNZe7nTokawzU11W%2BN3mKee%2BcNS2gjIpRGB5Y%2BkyQC12rZPicFkxOQgaqxFbagy23Nt%2Be3Ms7469XHcadWQn%2F7gZU8aBW8FDebdYPc%2BEdhF03QgwWfF44juXpoBA8CYesVM8rQ9YJ17tZe4jnC01iBDug%2BrGXqvfB5%2B7XA%2B9hYVhVl67cCtYSQ6j28R6oChUCIeqbtEGHr2sdZDH&acctmode=0&pass_ticket=Vsg12gw45s1SMN96j4Ztm1WuWcUGA3%2BYCXjPM4Z9%2BTEV6aaD%2BMtx2%2BIdEe3RWZGr&wx_header=1)钟意作者

3人喜欢

![](http://wx.qlogo.cn/mmopen/icWN8ZNEXrkMUtTeq5hEKNjk3FvGcgNhicxlgKVRN1wicEIcibvUjVsmkcYbQricInxOHsE7l8BoFwrK92C3203UD2b1mPupxNQCh/64)![](http://wx.qlogo.cn/mmopen/I1YzhXxW8YAjFBJzT2b6KCHddGvWmGz7CMf9dQXOjTCy7JV62JSeqZll0Lx2z88HtAiaLJibmvXYAr4n6VthUgBQ/64)![](http://wx.qlogo.cn/mmopen/REdvicaEDISSNqqqGrGqswBmlA2JD71ZELsnDFuicRRT3YXUAAialJ1iaU8YWLB0El5Yp9XhO0YACYsNPhL8ltXtkic7V1NcUANn5/64)

阅读原文

阅读 2311

​

写留言

**留言 16**

- Xman

  2021年12月20日

  赞9

  你把我用了多年的技术秘密泄露了，我一般在头文件中还做个类型限制![[呲牙]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![[呲牙]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

- Zen

  2021年12月20日

  赞2

  模板和宏有点类似，会在每个源文件都生成一份函数定义。这样会产生很多奇怪的事，比如函数的地址不一致，函数重定义错误，产生了多份的重复代码。所以linker必须负责消除重复代码只保留一份代码，使其保持正确的语义

- 城南花已开

  2021年12月20日

  赞1

  第二![[得意]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

- 已连接到互联网

  2021年12月20日

  赞1

  第一![[旺柴]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

  程序喵大人

  作者2021年12月20日

  赞

  认证

- ᯤ⁶ᴳ

  2021年12月25日

  赞

  template struct TemplateTest<int>;放在头文件中也可以吧，作者认为放在头文件和放在定义文件中哪个更好一些呢

- 徐徐图之文

  2021年12月21日

  赞

  大佬在b站有号吗

- cactus

  2021年12月20日

  赞

  问题是我问的，实验室师姐提出的问题。喵哥对粉丝真上心![[玫瑰]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

  程序喵大人

  作者2021年12月20日

  赞

  感谢你提供素材![😄](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

- 传。

  2021年12月20日

  赞

  请问如果模板类中的模板函数 可以这样做吗

  程序喵大人

  作者2021年12月20日

  赞

  也类似

  传。

  2021年12月20日

  赞

  好 我研究下 谢谢大佬![[憨笑]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

- Chunel

  2021年12月20日

  赞

  喵大人，可以试试看.inl文件吧![[色]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

- μLeoΣ_ω_λΛδ_ξηΔ\*

  2021年12月20日

  赞

  感谢喵哥 解惑 哈哈哈

- 雨乐

  2021年12月20日

  赞

  项目中很少用到模板，确实很方便，就是对后面接手人不是很友好![😂](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

  程序喵大人

  作者2021年12月20日

  赞

  简单的用一用还是挺方便的![[偷笑]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

已无更多数据

[](javacript:;)

![](http://mmbiz.qpic.cn/sz_mmbiz_png/Av3j9GXyMmGmg0HNK7fR6Peq3uf9pUoN5Qt0s0ia8IH97pvIiaADsEvy9nm3YnR5bmA13O8ic0JJOicQO0IboKGdZg/300?wx_fmt=png&wxfrom=18)

程序喵大人

2318

16

写留言

**留言 16**

- Xman

  2021年12月20日

  赞9

  你把我用了多年的技术秘密泄露了，我一般在头文件中还做个类型限制![[呲牙]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![[呲牙]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

- Zen

  2021年12月20日

  赞2

  模板和宏有点类似，会在每个源文件都生成一份函数定义。这样会产生很多奇怪的事，比如函数的地址不一致，函数重定义错误，产生了多份的重复代码。所以linker必须负责消除重复代码只保留一份代码，使其保持正确的语义

- 城南花已开

  2021年12月20日

  赞1

  第二![[得意]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

- 已连接到互联网

  2021年12月20日

  赞1

  第一![[旺柴]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

  程序喵大人

  作者2021年12月20日

  赞

  认证

- ᯤ⁶ᴳ

  2021年12月25日

  赞

  template struct TemplateTest<int>;放在头文件中也可以吧，作者认为放在头文件和放在定义文件中哪个更好一些呢

- 徐徐图之文

  2021年12月21日

  赞

  大佬在b站有号吗

- cactus

  2021年12月20日

  赞

  问题是我问的，实验室师姐提出的问题。喵哥对粉丝真上心![[玫瑰]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

  程序喵大人

  作者2021年12月20日

  赞

  感谢你提供素材![😄](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

- 传。

  2021年12月20日

  赞

  请问如果模板类中的模板函数 可以这样做吗

  程序喵大人

  作者2021年12月20日

  赞

  也类似

  传。

  2021年12月20日

  赞

  好 我研究下 谢谢大佬![[憨笑]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

- Chunel

  2021年12月20日

  赞

  喵大人，可以试试看.inl文件吧![[色]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

- μLeoΣ_ω_λΛδ_ξηΔ\*

  2021年12月20日

  赞

  感谢喵哥 解惑 哈哈哈

- 雨乐

  2021年12月20日

  赞

  项目中很少用到模板，确实很方便，就是对后面接手人不是很友好![😂](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

  程序喵大人

  作者2021年12月20日

  赞

  简单的用一用还是挺方便的![[偷笑]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

已无更多数据
