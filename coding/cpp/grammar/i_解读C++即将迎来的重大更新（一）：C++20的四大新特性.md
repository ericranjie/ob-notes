选自modernescpp
**作者：JP Tech等**
**机器之心编译**
**参与：Panda、杜伟**

> C++20（C++ 编程语言标准 2020 版）将是 C++ 语言一次非常重大的更新，将为这门语言引入大量新特性。近日，C++ 开发者 Rainer Grimm 正通过一系列博客文章介绍 C++20 的新特性。目前这个系列文章已经更新了两篇，本篇是第一篇，主要介绍了 C++20 的 Big Four（四大新特性：概念、范围、协程和模块）以及核心语言（包括一些新的运算符和指示符）。

!\[\[Pasted image 20241006174431.png\]\]

C++20 有很多更新，**上图展示了 C++20 更新的概况**。下面作者首先介绍 了 C++20 的编译器支持情况，然后介绍 The Big Four（四大新特性）以及核心语言方面的新特性。

**C++20 的编译器支持**
适应新特性的最简单方法是试用它们。那么接下来我们就面临着这个问题：哪些编译器支持 C++20 的哪些特性？一般来说，[cppreference.com/compiler_support_能提供在核心语言和库方面的答案。](http://cppreference.com/compiler_support_%E8%83%BD%E6%8F%90%E4%BE%9B%E5%9C%A8%E6%A0%B8%E5%BF%83%E8%AF%AD%E8%A8%80%E5%92%8C%E5%BA%93%E6%96%B9%E9%9D%A2%E7%9A%84%E7%AD%94%E6%A1%88%E3%80%82)

简单来说，全新的 GCC、Clang 和 EDG 编译器能提供对核心语言的最佳支持。此外，MSVC 和 Apple Clang 编译器也支持许多 C++20 特性。
!\[\[Pasted image 20241006174449.png\]\]

_C++20 核心语言特征。_
库方面的情况类似。GCC 在库方面的支持最好，接下来是 Clang 和 MSVC 编译器。
!\[\[Pasted image 20241006174502.png\]\]
_C++20 库特征。_

上面的截图仅展示了对应表格的前面一部分，可以看出这些编译器的表现并不是非常令人满意。即使你使用的是全新的编译器，这些编译器仍然不支持很多新特性。
通常来说，你能找到尝试这些新特性的方法。下面是两个示例：

- 概念：
  GCC 支持概念的前一个版本；
- std::jthread：
  GitHub 上有一个实现草案，来自 Nicolai Josuttis：
  [https://github.com/josuttis/jthread](https://github.com/josuttis/jthread)

简单来说，问题没有那么严重。只需要一些调整修改，很多新特性就能进行尝试。如有必要，我会提到如何进行这样的修改。

**四大新特性**
**概念（concept）**

使用模板进行通用编程的关键思想是定义能通过各种类型（type）使用的函数和类。但是，在实例化模板时经常会出现用错类型的问题，其结果通常是几页难懂的报错信息。

现在概念来了，这个问题可以休矣。概念让你能为模板编写要求，而编译器则可以检查这个要求。概念革新了我们思考和编写通用代码的方式。原因如下：

- 模板的要求是接口的一部分；
- 类模板中的函数重载或特殊化可以基于概念进行；
- 因为编译器能够比较模板参数的要求与实际的模板参数，所以能得到更好的报错信息。

但是，这还不是全部。

- 你可以使用预定义的概念，也可以定义你自己的概念；
- auto 和概念的用法统一到了一起。你可以不使用 auto，而是使用概念；
- 如果一个函数声明使用了一个概念，那么它会自动变成一个函数模板。由此，编写函数模板就变得与编写函数一样简单。

下面的代码片段展示了一个简单概念 Integral 的定义和使用方式：

```cpp
template<typename T>
concept bool Integral(){
    return std::is_integral<T>::value;
}

Integral auto gcd(Integral auto a,
                  Integral auto b){
    if( b == 0 ) return a;
    else return gcd(b, a % b);
}
```

Integral 这个概念需要

```cpp
std::is_integral<T>::value
```

中的类型参数 T。

```cpp
std::is_integral<T>::value 
```

这个函数来自 type-traits 库，它能在 T 为整数检查编译时间。如果

```cpp
std::is_integral<T>::value
```

的值为 true，则没有问题。如果不为 true，则你会
收到一个编译时间报错。如果你很好奇（你也应该好奇），我的这篇文章介绍了 type-
traits 库：\[https://www.modernescpp.com/index.php/tag/type-traits。\]

(https://www.modernescpp.com/index.php/tag/type-traits%E3%80%82)

gcd 算法是基于欧几里德算法确定最大公约数（greatest common divisor）。我使用了这个缩写函数模板句法来定义 gcd。gcd 要求其参数和返回类型支持概念 Integral。gcd 是一类对参数和返回值都有要求的函数模板。当我删除这个句法糖（syntactic sugar）时，也许你能看到 gcd 的真正本质。

下面这段代码在语义上与 gcd 算法等效：

```cpp
template<typename T>
requires Integral<T>()
T gcd(T a, T b){
    if( b == 0 ) return a;
    else return gcd(b, a % b);
}
```

如果你还没看到 gcd 的真正本质，过几周我还会专门发布一篇介绍概念的文章。

**范围库（Ranges Library）**

范围库是概念的首个客户。它支持的算法满足以下条件：

- 可以直接在容器上操作；无需迭代器指定一个范围；
- 可以宽松地评估；
- 可以组合。

简单来说：范围库支持函数模式（functional patterns）。

代码可能比语言描述更清楚。下面的函数用竖线符号展示了函数组成：

```cpp
#include <vector>
#include <ranges>
#include <iostream>

int main(){
  std::vector<int> ints{0, 1, 2, 3, 4, 5};
  auto even = [](int i){ return 0 == i % 2; };
  auto square = [](int i) { return i * i; };

  for (int i : ints | std::view::filter(even) |
                      std::view::transform(square)) {
    std::cout << i << ' ';             // 0 4 16
  }
}
```

even 是一个 lambda 函数，其在 i 为偶数时返回；lambda 函数 square 则会将 i 映射为它的平方。其余的必须从左到右读取的第 i 个函数组成：for (int i : ints | std::view::filter(even) | std::view::transform(square)). 将过滤器 even 应用于 ints 的每个元素，然后将其余的每个元素映射为它们的平方。如果你熟悉函数编程，那么这读起来就像一篇散文诗。

**协程（Coroutines）**

协程是广义的函数，能在保持状态的同时暂停或继续。协程通常用来编写事件驱动型应用。事件驱动型应用可以是模拟、游戏、服务器、用户接口或算法。协程也通常被用于协作式多任务（cooperative multitasking）。

我们这里不介绍 C++20 的具体协程，而会介绍编写协程的框架。编写协程的框架由 20 多个函数构成，其中一部分需要你去实现，另一部分则可能需要重写。因此，你可以根据需求调整协程。

下面展示了一个特定协程的用法。下面的程序使用了一个能产生无限数据流的生成器：

```cpp
Generator<int> getNext(int start = 0, int step = 1){
    auto value = start;
    for (int i = 0;; ++i){
        co_yield value;            // 1
        value += step;
    }
}

int main() {
    std::cout << std::endl;

    std::cout << "getNext():";
    auto gen = getNext();
    for (int i = 0; i <= 10; ++i) {
        gen.next();               // 2
        std::cout << " " << gen.getValue();
    }

    std::cout << "\\n\\n";

    std::cout << "getNext(100, -10):";
    auto gen2 = getNext(100, -10);
    for (int i = 0; i <= 20; ++i) {
        gen2.next();             // 3
        std::cout << " " << gen2.getValue();
    }

    std::cout << std::endl;
}
```

必须补充几句。这段代码只是一个代码段。函数 getNext 是一个协程，因为它使用了关键字 co_yield。getNext 有一个无限的循环，其会在 co_yield 之后返回 value。调用 next()（注释的 第 2、3 行）会继续这个协程，接下来的 getValue 调用会获取这个值。在 getNext 调用之后，这个协程再一次暂停。其暂停会一直持续到下一次调用 next()。我的这个示例中有一个很大的未知，即 getNext 函数的返回值

```cpp
Generator<int>
```

。这部分内容很复杂，后面我在写协程的文章中更详细地介绍。
使用 Wandbox 在线编译器，我可以向你展示这个程序的输出：
!\[\[Pasted image 20241006175104.png\]\]

**模块（Module）**
模块部分简单介绍一下就好。模块承诺能够实现：

- 更快的编译时间；
- 宏的隔离；
- 表达代码的逻辑结构；
- 不必再使用头文件（header file）；
- 摆脱丑陋的宏方法。

\*原文链接：[https://www.modernescpp.com/index.php/thebigfour\*](https://www.modernescpp.com/index.php/thebigfour*)

**本文为机器之心编译，转载请联系本公众号获得授权。**

✄------------------------------------------------
