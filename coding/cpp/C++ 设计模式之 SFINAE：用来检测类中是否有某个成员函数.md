
CPP开发者

 _2022年01月07日 11:55_

以下文章来源于编程往事 ，作者果冻虾仁

[

![](http://wx.qlogo.cn/mmhead/Q3auHgzwzM7Hvg7rQmorRljlcVCzwYTttaruhY8OCBSft64AYB32Cg/0)

**编程往事**.

C++码农，brpc committer，搜广推在线工程。专注互联网后端技术分享、行业观察以及个人成长！也欢迎关注我的知乎：果冻虾仁

](https://mp.weixin.qq.com/s?__biz=MzAxNDI5NzEzNg==&mid=2651169684&idx=1&sn=3f60f93b5c10ae6dbc677ceae985260c&chksm=806472cbb713fbdd5c8850d3352a51dc491b82f71ab439036edbf27ee579b72f175d541ee31e&mpshare=1&scene=24&srcid=0107nRfHzeUhJxK9GOcTFLwC&sharer_sharetime=1641527818086&sharer_shareid=5fb9813bfe9ffc983435bfc8d8c5e9ca&key=daf9bdc5abc4e8d0b89c0d2ad9c53a4b7fa5295f04e88169b327166a669c47386fe758efee4ba34875700f2abd8b997f469fb1fff79a8f8cb97fc7572eaaa1c26e59bf9ba0ba42402df52861503f17684148af3160cc239238f5fe85c388ec0525d344e93fdb70a0b9e205d19a7039ec1546d7f1a1a5c773a951366c53df3b8f&ascene=0&uin=MTEwNTU1MjgwMw%3D%3D&devicetype=Windows+11+x64&version=63090b19&lang=zh_CN&countrycode=CN&exportkey=n_ChQIAhIQqtEIxkp5229A1Pu4WSCubBLmAQIE97dBBAEAAAAAAAclCsFcv10AAAAOpnltbLcz9gKNyK89dVj0sGt49kBtBqfPmP2V3gR5pMKlz%2BRV9tjgtzI1bIl%2BlSpSvA69u3bAKaZC7bvTWYiwLrPrXqNVfjxQxr4G%2BW4ZiOev54NGIB6n5ToXKRTG%2FVt%2Fg%2BYeIAKHikI8przD4l3KHUMP391KQRNjOV60dmqS0qJIWRZkPB%2BtHvd%2FahhQxSb5%2FoDsIVee%2F9UKxJN13fiBnQNkdrOd%2Bo%2BrhEy1%2FYSG%2Bd3hG3arCn5Kmr2GKdNdYm47f6hAMVToSQthWGizCWLl&acctmode=0&pass_ticket=2wTclGaZpx0f3aWctdLoJXhMGxHJqyrkYiQxbNV6krLBQRyNQYdukhmUnP72fJBr&wx_header=1&fasttmpl_type=0&fasttmpl_fullversion=7350504-zh_CN-zip&fasttmpl_flag=1#)

针对类中特定成员函数的检测其实在工作中也可能用到。C++中可以用`SFINAE`技巧达到这个目的。  

![图片](https://mmbiz.qpic.cn/mmbiz_png/hQZ4NEZ2sic6IxKk564AlUBJ5d1lSfJL6Dias1LWKjBEm6eFmgQ3Libq5FiareWH2GolkicHrfq9lSESyGORQfKx6Tw/640?wx_fmt=png&wxfrom=13&tp=wxpic)

**SFINAE**是**Substitution Failure Is Not An Error**的缩写，直译为：匹配失败不是错误。属于C++模板编程中的高级技巧，但属于模板元编程中的基本技巧。

当然我其实也并不是C++元编程方面的专家，只是搜集过一些常见的实现方式，然后做过一些测试。在这个过程中，我发现有些常见的`SFINAE`写法是有问题的，下面探讨一下。

举个例子，我们来check一下C++标准库的类中有没有`push_back()`成员函数。在C++11之前，可以这样写，经过测试是没有问题的：

`#include <iostream>   #include <map>   #include <list>   #include <set>   #include <string>   #include <vector>   struct has_push_back {          template <typename C, void (C::*)(const typename C::value_type&)>       struct Helper;          template <typename C, void (C::*)(typename C::value_type)>       struct Helper2;          template <typename C>       static bool test(...) {           return false;       }       template <typename C>       static bool test(Helper<C, &C::push_back>*) {           return true;       }       template <typename C>       static bool test(Helper2<C, &C::push_back>*) {           return true;       }   };      int main() {       std::cout << has_push_back::test<std::list<int> >(NULL) << std::endl;       std::cout << has_push_back::test<std::map<int, int> >(NULL) << std::endl;       std::cout << has_push_back::test<std::set<int> >(NULL) << std::endl;       std::cout << has_push_back::test<std::string>(NULL) << std::endl;       std::cout << has_push_back::test<std::vector<int> >(NULL) << std::endl;       return 0;   }   `

SFINAE实现方式有很多种，细节处可能不同。

两个Helper类的模板参数中。第二个参数为 push_back的函数指针类型。之所以弄了两个Helper，是因为std::string的push_back的参数为char。也就是value_type类型。

而其他STL容器。则是`const value_type&`。所以才用了两个Helper。如果是检测其他成员函数，比如size则不需要这么麻烦只要一个Helper即可。

而test函数，对于返回true的模板函数，其参数是一个指针类型。所以实际check的时候，传入一个NULL就可以匹配到。

C++11之后：

`#include <iostream>   #include <list>   #include <map>   #include <set>   #include <string>   #include <vector>      template <typename>   using void_t = void;      template <typename T, typename V = void>   struct has_push_back:std::false_type {};      template <typename T>   struct has_push_back<T, void_t<decltype(std::declval<T>().push_back(std::declval<typename T::value_type>()))>>:std::true_type {};      int main() {       std::cout << has_push_back<std::list<int>>::value << std::endl;       std::cout << has_push_back<std::map<int, int>>::value << std::endl;       std::cout << has_push_back<std::set<int>>::value << std::endl;       std::cout << has_push_back<std::string>::value << std::endl;       std::cout << has_push_back<std::vector<int>>::value << std::endl;       return 0;   }   `

C++11简洁了许多。如果需求是要检测任意成员函数，而不限定是哪个函数的话，毫无疑问，需要借助宏了。将上面的代码改变成宏的版本，push_back作为宏的一个参数，即可。

我这里为什么用`push_back()`举例呢？**因为网上能找到的各种SFINAE的实现版本中，很多对于push_back的检测都是有问题的。** 而以上列举这两种，都能准确检测出string、vector、list中的`push_back()`。

**当然C++11之前的版本，需要你能枚举出push_back的各种参数种类才行，若待检测的成员函数重载版本比较多的时候，则可能很麻烦。所以还是C++11之后的版本简洁且通用**。

下面列举一个**常见但某些情况下会存在问题**的`SFINAE`范本：

`class Base {      };   class Drive:Base {   public:       void hello() {}   };   template <typename T>   struct has_hello {       typedef char Yes[1]; // 或     typedef int8_t Yes;       typedef char No[2];  // 或     typedef int16_t No;          template <typename>       static No& has(...);          template <typename C>       static Yes& has(decltype(&C::hello));          static const bool value = sizeof(has<T>(NULL)) == sizeof(Yes);   };      int main() {       std::cout << has_hello<Base>::value << std::endl;       std::cout << has_hello<Drive>::value << std::endl;   }   `

OK，这个用来检测类中是否有hello成员函数是可以的。但是改变成push_back的版本则有问题。

`#include <iostream>   #include <list>   #include <map>   #include <set>   #include <string>   #include <vector>   // 错误的示范   template <typename T>   struct has_push_back {       typedef char Yes[1];       typedef char No[2];          template <typename>       static No& has(...);          template <typename C>       static Yes& has(decltype(&C::push_back));          static const bool value = sizeof(has<T>(NULL)) == sizeof(Yes);   };      int main() {       std::cout << has_push_back<std::list<int> >::value << std::endl;       std::cout << has_push_back<std::map<int, int> >::value << std::endl;       std::cout << has_push_back<std::set<int> >::value << std::endl;       std::cout << has_push_back<std::string>::value << std::endl;       std::cout << has_push_back<std::vector<int> >::value << std::endl;       return 0;   }   `

上面这个是一个典型的有问题的案例——常见范本改变的push_back检测，对上面这几个类，只有string能判断为true。vector、list都check有误。

该版本也有很多其他变种。所谓变种主要是在has的返回值、value的判断方面做改编。也有一定问题，具体大家自己测试吧。

  

- EOF -

推荐阅读  点击标题可跳转

1、[原来这才是 Socket！](http://mp.weixin.qq.com/s?__biz=MzAxNDI5NzEzNg==&mid=2651169462&idx=1&sn=837e8acfa18c5e68ab7bf28bcf56d3c5&chksm=806473e9b713faffbed47c5a706abc384deaf48cdd6166d895b6e9b59c07e9afe625781f8109&scene=21#wechat_redirect)

2、[模版定义一定要写在头文件中吗?](http://mp.weixin.qq.com/s?__biz=MzAxNDI5NzEzNg==&mid=2651169461&idx=1&sn=4e88151de7f08d1ac38b762f18b9caba&chksm=806473eab713fafcc6b857601c664c21bccb72d1db1dbcf1c9f7f47541ef19c1a87d32d9d6b0&scene=21#wechat_redirect)

3、[C++ Trick：什么时候需要前置声明？](http://mp.weixin.qq.com/s?__biz=MzAxNDI5NzEzNg==&mid=2651169463&idx=1&sn=dd7dbafea61f90f9d5ed14ee9141ee2e&chksm=806473e8b713fafedb08df9c5c38a5a2308e43afeaf5fc4314afc051e662f7ea290ed93dacf7&scene=21#wechat_redirect)

  

**关注『CPP开发者』**  

看精选C++技术文章 . 加C++开发者专属圈子

![](http://mmbiz.qpic.cn/mmbiz_png/pldYwMfYJpia3uWic6GbPCC1LgjBWzkBVqYrMfbfT6o9uMDnlLELGNgYDP496LvDfiaAiaOt0cZBlBWw4icAs6OHg8Q/300?wx_fmt=png&wxfrom=19)

**CPP开发者**

我们在 Github 维护着 9000+ star 的C语言/C++开发资源。日常分享 C语言 和 C++ 开发相关技术文章，每篇文章都经过精心筛选，一篇文章讲透一个知识点，让读者读有所获～

24篇原创内容

公众号

点赞和在看就是最大的支持❤️

阅读 2572

​