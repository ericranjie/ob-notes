选自modernescpp

**作者：JP Tech等**

**机器之心编译**

**参与：Panda、杜伟**

> C++20（C++ 编程语言标准 2020 版）将是 C++ 语言一次非常重大的更新，将为这门语言引入大量新特性。C++ 开发者 Rainer Grimm 通过一系列博客文章介绍 C++20 的新特性。目前这个系列文章已经更新了两篇，本篇是第二篇，主要介绍了 C++20 的核心语言（包括一些新的运算符和指示符）。

**C++20 的核心语言**

之前的一篇博客概览式地介绍了 C++20 的概念、范围、协程和模块，下面开始介绍它的核心语言。

[https://mmbiz.qpic.cn/mmbiz_png/KmXPKA19gWibicMtMPFIyPdUxjIbk7w0vuX8NLTDEy4mX3TXQx7xyw7feALMunkJPVEUia1mj2avY57x19ObpofJg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1](https://mmbiz.qpic.cn/mmbiz_png/KmXPKA19gWibicMtMPFIyPdUxjIbk7w0vuX8NLTDEy4mX3TXQx7xyw7feALMunkJPVEUia1mj2avY57x19ObpofJg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

**三路比较运算符 <=>**

三路比较运算符 <=> 通常被称为太空船运算符。飞船运算符能确定两个值 A 和 B 谁大谁小或相等。

编译器可以自动生成三路比较运算符。你只需要用 default 礼貌地要求它即可。在这种情况下，你会得到所有六种比较运算符：==、!=、<、 <=、>、>=。

```
#include <compare>
struct MyInt {
  int value;
  MyInt(int value): value{value} { }
  auto operator<=>(const MyInt&) const = default;
};
```

默认的 <=> 执行的是字典顺序比较（lexicographical comparison），使用从基类开始从左到右并以声明顺序（declaration order）使用非静态元素的顺序。

微软的博客上有一些相当复杂精细的示例：[https://devblogs.microsoft.com/cppblog/simplify-your-code-with-rocket-science-c20s-spaceship-operator/](https://devblogs.microsoft.com/cppblog/simplify-your-code-with-rocket-science-c20s-spaceship-operator/)

```cpp
struct Basics {
  int i;
  char c;
  float f;
  double d;
  auto operator<=>(const Basics&) const = default;
};

struct Arrays {
  int ai[1];
  char ac[2];
  float af[3];
  double ad[2][2];
  auto operator<=>(const Arrays&) const = default;
};

struct Bases : Basics, Arrays {
  auto operator<=>(const Bases&) const = default;
};

int main() {
  constexpr Bases a = { { 0, 'c', 1.f, 1. },
                        { { 1 }, { 'a', 'b' }, { 1.f, 2.f, 3.f }, { { 1., 2. }, { 3., 4. } } } };
  constexpr Bases b = { { 0, 'c', 1.f, 1. },
                        { { 1 }, { 'a', 'b' }, { 1.f, 2.f, 3.f }, { { 1., 2. }, { 3., 4. } } } };
  static_assert(a == b);
  static_assert(!(a != b));
  static_assert(!(a < b));
  static_assert(a <= b);
  static_assert(!(a > b));
  static_assert(a >= b);
}
```

我认为，这个代码段中最复杂的部分不是太空船运算符，而是使用聚合初始化（aggregate initialisation）来实现 Base 的初始化。聚合初始化本质上意味着如果所有元素是公开的，那么你可以直接初始化类类型（class、struct 或 union）的元素。在这个案例中，你可以使用示例中那样的 braced-initialisation-list。好吧，这确实经过了简化，详见：[https://en.cppreference.com/w/cpp/language/aggregate_initialization](https://en.cppreference.com/w/cpp/language/aggregate_initialization)

**将字符串文字作为模板参数**

在 C++20 之前，你不能将字符串用作非类型的模板参数。使用 C++20 时，你则可以这么做。我们可以在标准定义的 basic_fixed_string 中使用它们，其有一个 constexpr 构造函数。这个 constexpr 构造函数能在编译时实例化这个固定字符串。

```cpp
template<std::basic_fixed_string T>
class Foo {
    static constexpr char const* Name = T;
public:
    void hello() const;
};

int main() {
    Foo<"Hello!"> foo;
    foo.hello();
}
```

**constexpr 虚拟函数**

由于动态类型是未知的，所以无法在常量表达式（constant expression）中调用虚拟函数。这个限制将在 C++20 中被解除。

**指定初始化器**

我首先谈谈聚合初始化。下面是一个简单示例：

```
// aggregateInitialisation.cpp

#include <iostream>

struct Point2D{
    int x;
    int y;
};

class Point3D{
public:
    int x;
    int y;
    int z;
};

int main(){

    std::cout << std::endl;

    Point2D point2D {1, 2};
    Point3D point3D {1, 2, 3};

    std::cout << "point2D: " << point2D.x << " " << point2D.y << std::endl;
    std::cout << "point3D: " << point3D.x << " " << point3D.y << " " << point3D.z << std::endl;

    std::cout << std::endl;

}
```

我认为无需对这个程序进行解释。看看这个程序的输出：

[https://mmbiz.qpic.cn/mmbiz_png/KmXPKA19gWibicMtMPFIyPdUxjIbk7w0vucCePTOKjicNNHCgwAnnvO0VUvIuZQhSXHfkYDrbg3WgmfwLBSicdcnpQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1](https://mmbiz.qpic.cn/mmbiz_png/KmXPKA19gWibicMtMPFIyPdUxjIbk7w0vucCePTOKjicNNHCgwAnnvO0VUvIuZQhSXHfkYDrbg3WgmfwLBSicdcnpQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

显式总比隐式好。我们看看这是什么意思。程序 aggregateInitialisation.cpp 中的初始化非常容易出错，因为你可能写反这个构造函数的参数，而且你永远没法察觉。来自 C99 的指定初始化器就能在这里大显身手了。

```
// designatedInitializer.cpp

#include <iostream>

struct Point2D{
    int x;
    int y;
};

class Point3D{
public:
    int x;
    int y;
    int z;
};

int main(){

    std::cout << std::endl;

    Point2D point2D {.x = 1, .y = 2};
    // Point2D point2d {.y = 2, .x = 1};         // (1) error
    Point3D point3D {.x = 1, .y = 2, .z = 2};
    // Point3D point3D {.x = 1, .z = 2}          // (2)  {1, 0, 2}

    std::cout << "point2D: " << point2D.x << " " << point2D.y << std::endl;
    std::cout << "point3D: " << point3D.x << " " << point3D.y << " " << point3D.z << std::endl;

    std::cout << std::endl;

}
```

实例 Point2d 和 Point3D 的参数从名称就能看出来。这个程序的输出就等同于程序 aggregateInitialisation.cpp 的输出。带注释（1）和（2）的行很有意思。行（1）会报错，因为指定器的顺序与它们的声明顺序不匹配。在（3）行中，y 的指定器缺失了。在这个案例中，y 会被初始化为 0，比如使用 braces-initialisation-list {1, 0, 3}.

**对 lambda 的各种改进**

C++20 在 lambda 方面的改进也很多。

如果你想要了解改动的细节，请参阅 Bartek 的博客：[https://www.bfilipek.com/2019/02/lambdas-story-part1.html，里面介绍了](https://www.bfilipek.com/2019/02/lambdas-story-part1.html%EF%BC%8C%E9%87%8C%E9%9D%A2%E4%BB%8B%E7%BB%8D%E4%BA%86) C++17 和 C++20 中的 lambda 改进。总之，我们会迎来两个有意思的变化。

- 允许 [=, this] 作为 lambda capture，并通过 [=] 弃用这个的隐式 capture

```
struct Lambda {
    auto foo() {
        return [=] { std::cout << s << std::endl; };
    }

    std::string s;
};

struct LambdaCpp20 {
    auto foo() {
        return [=, this] { std::cout << s << std::endl; };
    }

    std::string s;
};
```

在 C++20 中，通过在结构体 lambda 中复制而实现隐式 [=] capture 会出现弃用警告。如果你通过复制 [=, this] 来显式地获取它，就不会收到 C++20 的弃用警告。

- 模板 lambda

你可能和我一样，最先想到的是：我们为什么需要模板 lambda？当你用 C++14 的 [](auto x){ return x; } 写一个通用 lambda 时，编译器会自动使用一个模板化的调用运算符来生成一个类：

```
template <typename T>
T operator(T x) const {
    return x;
}
```

有时候，你想要定义一个只对某个特定类型（如 std::vector）有效的 lambda。现在，模板 lambda 能帮我们做到这一点。你可以不使用类型参数，而是使用概念：

```
auto foo = []<typename T>(std::vector<T> const& vec) {
        // do vector specific stuff
    };
```

**新属性：[[likely]] 和 [[unlikely]]**

C++20 有 [[likely]] 和 [[unlikely]] 两个新属性。这两个新属性都允许为优化器提供提示：执行的路径是更可能或是更不可能。

```
for(size_t i=0; i < v.size(); ++i){
  if (unlikely(v[i] < 0)) sum -= sqrt(-v[i]);
  else sum += sqrt(v[i]);
}
```

**指示符 consteval 和 constinit**

新的指示符 consteval 会创建一个即时函数。对于一个即时函数，每一次函数调用都必然产生一个编译时常量表达式。即时函数是隐式的 constexpr 函数。

```
consteval int sqr(int n) {
  return n*n;
}
constexpr int r = sqr(100);  // OK

int x = 100;
int r2 = sqr(x);             // Error
```

因为 x 不是常量表达式，所以最后的赋值会出错。因此，编译时不会执行 sqr(x)。

constinit 会确保有静态存储持续的变量在编译时被初始化。静态存储持续（static storage duration）的意思是对象会在程序开始时分配，在程序结束时又会重新分配。对于命名空间范围内声明的对象（全局对象），声明为 static 或 extern 的对象有静态存储持续。

**std::source_location**

C++11 有两个宏 **LINE** 和 **FILE** 来获取代码行和文件的信息。而在 C++20 中，类 source_location 能提供有关源代码的文件名、行号、列号和函数名信息。下面这个来自 cppreference.com的示例展示了第一种用途：

```
#include <iostream>
#include <string_view>
#include <source_location>

void log(std::string_view message,
         const std::source_location& location = std::source_location::current())
{
    std::cout << "info:"
              << location.file_name() << ":"
              << location.line() << " "
              << message << '\\n';
}

int main()
{
    log("Hello world!");  // info:main.cpp:15 Hello world!
}
```

*原文链接：[https://www.modernescpp.com/index.php/c-20-the-core-language*](https://www.modernescpp.com/index.php/c-20-the-core-language*)

**本文为机器之心编译，转载请联系本公众号获得授权。**

✄------------------------------------------------

**加入机器之心（全职记者 / 实习生）：hr@jiqizhixin.com**

**投稿或寻求报道：content@jiqizhixin.com**

**广告 & 商务合作：bd@jiqizhixin.com**