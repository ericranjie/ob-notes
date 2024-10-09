# 

Original 里缪 CppMore

_2022年02月18日 19:20_

晚上好。

本篇开始介绍C++反射，这是第一篇。

## C++中的产生式元编程

元编程在C++已有几十年的历史，一般来说，我们是指编译期的编程。

产生式元编程，则侧重于强调「产生代码的代码」这种编程方式。

C++提供了许多生成代码的特性，如图所示。

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

宏虽然只是C中的一个简单替换的工具，却也具有强大的产生代码的能力。过去的文章多次使用它来简化重复的工作，想必大家并不陌生。

模板作为C++元编程的工具，可以完成泛型编程「抽象化」的工作，借其可实现变量、函数、类的泛化，从而自动产生成百上千行代码。同样，大家对此自然更加熟悉。

Fold Expressions是C++17提供的展开参数包的简便方式，这同样是一种产生代码的特性。见[C++17: Simplify Code with Fold Expressions](http://mp.weixin.qq.com/s?__biz=MzUxOTQ4NjIzNw==&mid=2247484950&idx=1&sn=3d833a2cc642d1826f6699b86b722b12&chksm=f9f9a864ce8e217285659c1ad7fa5a02a98cd2b030971ad8bab859637b7b5f17f45514cf1acd&scene=21#wechat_redirect)。

Expansion Statements是P1306提出的一种新语句，可以减少遍历时的重复，主要是方便反射遍历用的。这也是一种产生代码的特性，举个遍历tuple的例子，

```cpp
auto tup = std::make_tuple(0, 'a', 3.14);  
template for (auto& elem : tup)  
    std::cout << elem << std::endl;  
```

语法起初是for...，后来改为了template for，这代码相当于如下代码：

```cpp
auto tup = std::make_tuple(0, 'a', 3.14);  
{  
    auto elem  = std::get<0>(tup);  
    std::cout << elem << std::endl;  
}  
{  
    auto elem  = std::get<1>(tup);  
    std::cout << elem << std::endl;  
}  
{  
    auto elem  = std::get<2>(tup);  
    std::cout << elem << std::endl;  
}
```

该提案由于时间原因没能进入C++20，之后三年没动静，最近CWG进行了review，但是作者一直没有更新论文，也没能进入C++23。似乎放弃了？（https://github.com/cplusplus/papers/issues/156）

虽然已经拥有这么多生成代码的特性，但是我们依旧无法轻易完成像序列化、ORM、远程调用、schema generation等等需求。因为C++缺少获取类型元信息的机制，所以无法获取像类型的参数列表、成员类型、成员名称等等这些信息。

缺少的就是反射的机制，这是一种产生代码更加强大的能力。

C++早已成立了专门的小组SG7来负责反射的研究工作，近些年也算是取得了一些发展，最快也许C++26可以加入。

## 反射的相关概念

本节简单地说下反射相关的概念，以建立共识。

首先给大家介绍两个词：_**reflection**_和_**reification**_。

反射一般包含两个部分，获取和构建，如图所示：

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

获取指的是从类型得到「类型元信息」，元信息要比类型高一层，所以是「自下而上」的结构。也就是说，这是一种从具体到抽象的结构，这个步骤就称为\_**reflection**\_。

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

而构建指的是从「类型元信息」再次得到类型，这是「自上而下」的结构，因此它是从抽象到具体，这个完全相反的步骤就称为\_**reification**\_。

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

接着，再向大家介绍一个词：_**Introspection**_。

简而言之，这指的是询问某个类型是否具有什么东西的特性。比如，查询一个类型是否派生自另一个类型，是否具有get_data()成员函数，是否拥有data属性，是否能转换成另一个类型，诸如此类。

这个特性就是反射能力的一部分，其实C++通过type traits和Concepts已经提供许多相关能力了。

C++缺少的是类型遍历的能力，有了这种能力，就能够遍历出类的模板参数、成员类型和函数列表这些信息，再通过_Introspection_便可操纵某些具体的变量或函数，自动产生其他版本的类。

到此，本节就先介绍这几个比较大的概念。

## C++中的反射

反射分为动态反射和静态反射，动态反射就是运行期的反射，静态反射就是编译期的反射。

C++的反射是静态反射，第一个Reflection TS基于N4766。当时还是基于类型来表示反射信息，举个例子：

```cpp
template <typename T>  
std::string get_type_name() {  
    namespace reflect = std::experimental::reflect;  
    // T_t is an Alias reflecting T:  
    using T_t = reflexpr(T);  
    // aliased_T_t is a Type reflecting the type for which T is a synonym:  
    using aliased_T_t = reflect::get_aliased_t<T_t>;  
    return reflect::get_name_v<aliased_T_t>;  
}  
std::cout << get_type_name<std::string>(); // outputs basic_string
```

看不懂也没关系，因为早就废弃这种方式了。

这种方式是为了简化和模板元的结合，然而出于多方面考虑，SG7转而支持value-based reflection，也就是现在的反射。为此，C++20提供了许多扩展特性来支持反射的设计，例如consteval function，std::is_constant_evaluated()，constexpr dynamic allocation。

新式的反射语法，举个例子：

```cpp
#include <meta>  
template<Enum T>  
std::string to_string(T value) {  
    template for (constexpr auto e : std::meta::members_of(^T)) {  
        if([:e:] == value) {  
            return std::string(std::meta::name_of(e));  
        }  
    }  
    return "<unnamed>";  
}
```

这段代码是要以string形式输出枚举类型的值。

获取类型元信息的操作符是"^ operator"，读作\*\*_lifting operator_\*\*，意思就是向上获取类型的元信息。这对应上一节介绍的\_**reflection**\_这个词。（\_reflection_操作的原本语法是reflexpr()，太重更换了）

通过std::meta::members_of()便可通过类型元信息获取到枚举类型的所有成员，以std::span返回。

如何遍历返回的结果呢？就用到了前面介绍的_expansion statements\_，也就是代码中的template for。

那么如何再把类型\_**reification**_出来呢？语法就是\[: _reflection_ :\]，这个称为_splice construct_。C++把这个过程称为\_**splicing**，\_它和_reification_指的是一个东西。

通过比较枚举值，相同则使用std::meta::name_of()返回枚举值的名称，以std::string_view返回。

上面的例子很简洁的说明了C++反射的用法，贯穿了上节介绍的概念，可以让大家先对反射有个大体的理解。

## 百花齐放

C++反射进入标准的速度如此慢，导致在过去这么多年，已经产生了许多用户自己实现反射的办法。

这大体可以分为三个阶段，如图所示。

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

首先来说T0阶段。

因为没有类型元信息机制，这些实作手法往往需要自己「保存」类型信息，然后再根据保存的内容「获取」相关的反射信息。

这些实现手法在编译期完成这些「保存」工作，就属于静态反射，反之则是动态反射。

不论如何，这些实现使用起来都有不少束缚，比如需要用户手动注册类型信息、不支持有些_Introspection\_、操作繁琐等等。

其次来说T1阶段，这些实现不需要用户手动注册类型信息。

第一种方式是采用TMP的一些技巧，保存类型的一些信息，使用起来比较方便，但局限不小，比如无法获取成员的名字。

第二种方式比较高大上，它通过提供特殊的编译器来提供类型元信息，也就是自己实现一套内置的反射系统。这种方式的反射特性比较完整，使用起来很方便，功能很强大，但是这些扩展换个编译器就无法编译，相当于是没有达成标准的野路子。

最后是T2阶段，也就是C++标准的反射。

前面已经介绍过，故不再赘述。需要注意的是，这里Reflection TS强调的是目前编译期支持的标准反射，而Valued-based reflection强调的是最新的反射。

## 总结

如题所言，本篇是讲反射「通识」。

因此不会太涉及细节，目的是先从大的方向上让大家对C++反射有个整体的了解。

后面的章节，则会再来针对每个模块谈论细节。

原本只打算写一篇的，包含一些细节，但是内容比想象的要多，所以拆解成为更有层次的一个小系列，不会太多，四五篇足矣。

里缪

![赞赏二维码](<https://mp.weixin.qq.com/s?__biz=MzUxOTQ4NjIzNw==&mid=2247487376&idx=1&sn=9a5f15e01f7670ccac3d273778e15132&chksm=f9f9a1e2ce8e28f4394905e1c959f8c31d424ee48d0c386a2f0cbf1dbb4eb4e127b418f87335&mpshare=1&scene=24&srcid=0219vkGpwBeBbUESZBUxwpkN&sharer_sharetime=1645267585098&sharer_shareid=5fb9813bfe9ffc983435bfc8d8c5e9ca&key=daf9bdc5abc4e8d0d96bd02b92aaf57568c04bae08d9540fe7b5b2a53634aaca5ca6531d92c104467e92e5cc85e491d40bfc0b7b3e76e7a68b90d14b5956cba7ff5265a63c114f70105b1fced746ba7d5687edd65b36d4593ad8753971495d2d6ed0ca21a2c3ab0fa969673c119856ba63b885095f4fb84ac3e55c950750e918&ascene=14&uin=MTEwNTU1MjgwMw%3D%3D&devicetype=iMac+MacBookAir10%2C1+OSX+OSX+14.6.1+build(23G93)&version=13080710&nettype=WIFI&lang=en&session_us=gh_ee7428b045cc&countrycode=CN&fontScale=100&exportkey=n_ChQIAhIQibnMw%2BFq%2Fzwi8VvvIbN5fRKUAgIE97dBBAEAAAAAANkXNKFaGP8AAAAOpnltbLcz9gKNyK89dVj0rU4TEN3pUwzr%2BfM0CDqFSR5ZMMdIOnwaj07%2FaMSyopr4A5v2lexerTrRzCXAAgFRXk68lauqsdqAnNyUmxg7361sLYDN5ivh%2Fhtqrg0WavTLh1WFhAWO2nulC%2FZqdjITxTTfBXnKReQ%2FuETlk9B%2FycYKbuzEt%2FdVBYouYS0Au5AL6m0%2B2OhmP2zREH62ekKEcwR%2F0O8k2m%2FqqGV2FIKNoHsfXfrkaVmU9SPinrSYd9p0mrHQ9102yAZ0lHNLDDT7T6MCXAa6QhrMp8fTXVG2lvB9yoiZ3515bEIdfzvVnx8FoZBzXcrt6oZgiTx32w%3D%3D&acctmode=0&pass_ticket=5h9op511WgSFQkH8Qc4kv4bQfHCrP4pUp3tuWd%2FZ8iNJR25yjKng3vg2zEZcLMYu&wx_header=0>)Like the Author

C++ Reflection5

Expansion Statements3

C++ Reflection · 目录

下一篇C++反射 第二章 探索

Reads 1357

​

Comment

**留言 10**

- ⃢👁-👁⃢🍃

  2022年2月18日

  Like

  能推荐几个库吗，现在在用rttr.谢谢！

  CppMore

  Author2022年2月18日

  Like3

  别急，过两天就再更新了，会包含各个阶段的库

- Running couplets

  2022年3月12日

  Like1

  可以理解为基于io的code吗？

  CppMore

  Author2022年3月12日

  Like

  不对

- 余老师

  2022年2月26日

  Like

  可以重点介绍一下虚幻引擎和qt的反射机制，虽然虚幻借住了c#

  CppMore

  Author2022年2月26日

  Like

  可以考虑一下

- 吴小吴

  2022年2月21日

  Like

  坐等更新，辛苦了![[玫瑰]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)

  CppMore

  Author2022年2月22日

  Like

  有点忙，在码字了

- 奇点创客

  2022年2月18日

  Like

  期待更新

  CppMore

  Author2022年2月18日

  Like

  安排上

已无更多数据

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/9XBBCfGaPEly5AtcdkIKicpv55w2VlBiaQ7TxtJneeDmgO3mxz4onND7rW2UjfSib9KC1FtBB4U6TupnBMfenoCgQ/300?wx_fmt=png&wxfrom=18)

CppMore

1794

10

Comment

**留言 10**

- ⃢👁-👁⃢🍃

  2022年2月18日

  Like

  能推荐几个库吗，现在在用rttr.谢谢！

  CppMore

  Author2022年2月18日

  Like3

  别急，过两天就再更新了，会包含各个阶段的库

- Running couplets

  2022年3月12日

  Like1

  可以理解为基于io的code吗？

  CppMore

  Author2022年3月12日

  Like

  不对

- 余老师

  2022年2月26日

  Like

  可以重点介绍一下虚幻引擎和qt的反射机制，虽然虚幻借住了c#

  CppMore

  Author2022年2月26日

  Like

  可以考虑一下

- 吴小吴

  2022年2月21日

  Like

  坐等更新，辛苦了![[玫瑰]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)

  CppMore

  Author2022年2月22日

  Like

  有点忙，在码字了

- 奇点创客

  2022年2月18日

  Like

  期待更新

  CppMore

  Author2022年2月18日

  Like

  安排上

已无更多数据
