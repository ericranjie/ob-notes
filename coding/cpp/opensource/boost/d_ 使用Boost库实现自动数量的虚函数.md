# 

原创 陶东明 CPP与系统软件联盟

_2022年02月10日 11:40_

**点击蓝字**

**关注我们**

特邀作者：陶东明，某通讯公司的敏捷技术教练，从事基础编程能力提升工作，CSDN的C++版资深网友。

1.

**关键词**

- C++17

- boost::mpl

- boost1.75+::pfr

- 《Modern C++ Design》

2.

**背景**

### 我们遇到一个场景，简化抽象后，就是：

- ### 需要实现一个foo函数模板，能处理类型Data的每一种数据成员类型
- ### 在漫长的调用链里，分别有人提供这3个参数，并最终调用到foo
- ### 知道是否要swap的地方只能传递出一个确定类型的值，无法做到一路上都是模板函数的转调用。原因放到后面“可行而又不可行的解”部分来略微解释。如果调用链全程都可以实现为模板函数，那本文章就不用存在了，直接模板函数一路平推就好啦。

```
struct Data
```

3.

**问题**

这个if令我感到很不爽。因为：

- 函数的圈复杂度增加了

- 基于覆盖率指标的测试，工作量也大增了

- 最严重的：如果交换参数的逻辑改变了，需要多人/多处的代码修改

4.

**通常方案**

```
struct Swapper
```

- 圈复杂度降低了

- 2个函数都可以单独测试了

- 修改也集中到一个人身上了

绝大部分场合，这个解已经非常完美了。

5.

**画蛇添足的需求**

但是，Swapper代码里仍然存在了if判断，这必定意味着，对于“需不需要swap”一共做过2次判断（创建这个swapper的地方无论如何都是要有一次信息判断的），有重复判断之嫌。

所以，我们的画蛇添足式需求是：能不能一共只要一次if判断，也就是从Swapper里去掉那次判断。

6.

**失败的尝试**

```
using Swapper = 。。。;
```

很可惜，这个尝试失败了，因为实现不了这个Swapper。

- std::function可以保存接口相同的lambda对象，但是不包括这种auto的lambda。

- 那就自己用类型擦除技术实现一个这样的万能function？恐怕也是做不到的。

  std::function就是用类型擦除实现的，要可以的话，标准库肯定也就直接提供了这样的万能function了。

7.

**可行而又不可行的解**

为什么我们没能实现为全程都是模板函数呢，因为我们有一个环节是事件注册机制的，

```
map<Event, Handler> eventHandlerMap;
```

然后，在后续调用链过程中再获得了first和second，再若干转调用，最终到了foo函数。

Handler这东西，它要么是个普通函数指针，要么是个基类指针（或者加上包装类）+派生类overload虚函数，要么就是std::function+lambda了。

它们的共同点都是：无法提供模板函数作为调用接口。

那么，在基类里提供一组重载的接口函数呢？这确实是可以的。

```
struct HandlerBase
```

但是这个解我无法接受，因为

- dontSwap和needSwap虽然比Data类的数据成员稳定多了，但是仍然是可能变化的，没有自适应方案的话，那就整天改代码吧。而自适应方案正是本文在讨论的东西。

- 如果还有第二个参数也如此需要重载解决，那我们就可能遇到组合爆炸。

8.

**掘地三尺式的解**

看起来，唯一可行的方案就是自动重载一组虚函数了。条件看起来还算不过于苛刻：

- 函数体里的内容是完全相同的。

- 需要重载多少个，只取决于类型Data有多少种数据成员，肯定没有其它不能预先确定的类型，也不是无穷个。

- 只是Data是变化的，所以，我们的代码不可以是写死的，要能自动根据Data的变化而变化。

这么一分析，大问题就变成了3个小问题了：

1. 从Data里获取所有数据成员的类型列表TYPE_LIST

1. TYPE_LIST进行类型去重，获得TYPE_SET

1. 对TYPE_SET里的每一个元素，都构造出一个虚函数

看过《Modern C++ Design》的都应该会2、3，boost::mpl库也很早就提供了2、3的解决方案。而boost 1.75刚刚提供了问题1的（最佳）解决方案：**反射库PFR**。

使用PFR库也有若干限制，比如需要C++17编译器，只能用在最简单的聚合类上。但是它是完全非侵入式的，所需代码也是最少的。如果用不了它，那还可以考虑boost::hana，或其它各种反射库。本文的重点是拿到元数据后的处理过程，就暂不对PFR以及反射进行展开讨论啦。

下面给出代码：

```
#include <boost/mpl/size.hpp>
```

这里给喜欢自己钻研的小伙伴们留一个提问：上面的代码，思路没问题，但是实际使用会有问题，在SwapperBase 里少了一个不起眼的声明性语句，你能修复问题点么？想不出来的，可以到本文结束处看答案。

完整的示例代码：

https://www.compiler-explorer.com/z/vWW5aG51G

9.

**峰回路转解**

上面这个实现，实在太复杂了。如果我们肯放下“面向对象编程的原则：不要向下类型映射，更加不要从void \*进行强制类型转换”，这里还有一个非常简单的解：

```
typedef tuple<const void *, const void *> (*Swapper)(const void *, const void *);
```

因为“从void \*进行强制类型转换”被严格控制在确定的范围内，其实并非真的存在类型安全方面的风险。C++作为一个多风格编程语言，我们要在“对原则的遵守”和“实现的便利性”上进行平衡，这个解完全可以成立的。

10

**附注**

上面的提问，答案是少了一句using NextBase::Call;

**关于我们**

CPP与系统软件联盟是Boolan旗下连接30万+中高端C++与系统软件研发骨干、技术经理、总监、CTO和国内外技术专家的高端技术交流平台。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

2022年3月11-12日，「全球C++及系统软件技术大会」将于上海隆重召开，“C++之父”Bjarne Stroustrup将发布主题演讲。来自海内外近40位专家将围绕包括：现代C++、系统级软件、架构与设计演化、高性能与低时延、质量与效能、工程与工具链、嵌入式开发、分布式与网络应用 共8大主题，深度探讨最佳工程实践和前沿方法。大会官网：www.cpp-summit.org

**点击此处“阅读原文”获取大会演讲全套PPT**

C++9

系统软件6

C++ · 目录

上一篇低延迟场景下的性能优化实践下一篇C++高性能大规模服务器开发实践

阅读原文

阅读 1322

​
