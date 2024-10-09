原文：[https://alexgaynor.net/2019/apr/21/modern-c++-wont-save-us/作者：Alex](https://alexgaynor.net/2019/apr/21/modern-c++-wont-save-us/%E4%BD%9C%E8%80%85%EF%BC%9AAlex) Gaynor，翻译：CSDN

经常有程序员为C++辩护说：“只要你不使用任何从C继承过来的功能，C++就是安全的”！但事实并非如此。

根据本文作者在大型C++项目上（遵从现代的惯用做法）的经验来看，C++提供的类型完全不能阻止漏洞的泛滥。本文中就会给出一些完全根据现代C++的惯用做法编写的代码，你会发现这些代码仍然会引发漏洞。

**以下为译文：**

我经常批评内存不安全的语言，主要是C和C++，以及它们引发的大量安全漏洞。根据大量使用C和C++的软件项目的审查结果，我得出了一个结论：软件行业应该使用内存安全的语言（例如Rust和Swift）。

人们常常在回复我时说，这个问题并不是C和C++本身的问题，而是使用这两种语言的开发者的错。

具体来说，我经常听到人们为C++辩护说：“只要你不使用任何从C继承过来的功能，C++就是安全的”（我理解这句话指的是原始指针、数组作为指针使用、手动malloc/free以及其他类似功能。但我认为有一点值得注意，由于C的特性明确地融入了C++，那么在实践中，大部分C++代码都需要处理类似的情况。），或者类似的话，比如只要遵从现代C++的类型和惯用做法，就不会引发内存方面的漏洞。

我很感谢C++的智能指针类型，因为这种类型的确非常有用。不幸的是，根据我在大型C++项目上（遵从现代的惯用做法）的经验来看，光靠这些类型完全不能阻止漏洞的泛滥。我会在本文中给出一些完全根据现代C++的惯用做法编写的代码，你会发现这些代码仍然会引发漏洞。

**掩盖“释放后使用”的引用**

我想说的第一个例子最初是Kostya Serebryany提出的（[https://github.com/isocpp/CppCoreGuidelines/issues/1038），这个例子可以说明C++的std::string_view能够很容易地掩盖“释放后使用”的漏洞：](https://github.com/isocpp/CppCoreGuidelines/issues/1038%EF%BC%89%EF%BC%8C%E8%BF%99%E4%B8%AA%E4%BE%8B%E5%AD%90%E5%8F%AF%E4%BB%A5%E8%AF%B4%E6%98%8EC++%E7%9A%84std::string_view%E8%83%BD%E5%A4%9F%E5%BE%88%E5%AE%B9%E6%98%93%E5%9C%B0%E6%8E%A9%E7%9B%96%E2%80%9C%E9%87%8A%E6%94%BE%E5%90%8E%E4%BD%BF%E7%94%A8%E2%80%9D%E7%9A%84%E6%BC%8F%E6%B4%9E%EF%BC%9A)

```cpp
#include <iostream>
#include <string>
#include <string_view>

int main() {
std::string s = "Hellooooooooooooooo ";
std::string_view sv = s + "World\\n";
std::cout << sv;
}
```

在这段代码中，s + "World\\n"分配了一个新的std::string，然后将其转换成std::string_view。此时临时的std::string被释放，但sv依然指向它原来拥有的内存。任何对sv的访问都会造成“释放后使用”的漏洞。

天啊！C++的编译器无法检测到sv拥有某个引用，而该引用的寿命比被引用的对象还要长的情况。同样的问题也会影响std::span，它也是个非常现代的C++类型。

另一个有意思的例子是使用C++的lambda功能来掩盖引用：

```c
#include <memory>
#include <iostream>
#include <functional>

std::function<int(void)> f(std::shared_ptr<int> x) {
return [&]() { return *x; };
}

int main() {
std::function<int(void)> y(nullptr);
    {
std::shared_ptr<int> x(std::make_shared<int>(4));
        y = f(x);
    }
std::cout << y() << std::endl;
}
```

上述代码中，f中的\[&\]表明lambda用引用的方式来捕获值。然后在main中，x超出了作用域，从而销毁了指向数据的最后一个引用，导致数据被释放。此时y就成了悬空指针。即使我们谨慎地使用智能指针也无法避免这个问题。没错，人们的确会编写代码来处理std::shared_ptr<T>&，作用之一就是设法避免引用计数无谓的增加或减少。

**std::optional<T>解引用**

std::optional表示一个可能存在也可能不存在的值，通常用来替换哨兵值（如-1或nullptr）。它提供的一些方法，如value()，能够提取出它包含的T，并在optional为空的时候抛出异常。但是，它也定义了operator\*和operator->。

这两个方法能访问底层的T，但它们并不会检查optional是否包含值。

例如，下面的代码就会返回未初始化的值：

```cpp
#include <optional>

int f() {
std::optional<int> x(std::nullopt);
return *x;
}
```

如果用std::optional来代替nullptr，就会产生更加严重的问题！对nullptr进行解引用会产生段错误（这并不是安全漏洞，只要不是在旧的内核上）。而对nullopt进行解引用会产生未初始化的值作为指针，这会导致严重的安全问题。尽管T\*也可能拥有未经初始化的值，但是这种情况非常罕见，远远不如对正确地初始化成nullptr的指针进行解引用的操作。

而且，这个问题并不需要使用原始的指针。即使使用智能指针也能得到未初始化的野指针：

```cpp
#include <optional>
#include <memory>

std::unique_ptr<int> f() {
std::optional<std::unique_ptr<int>> x(std::nullopt);
return std::move(*x);
}
```

**std::span<T>索引**

std::span<T>能让我们方便地传递指向一片连续内存的引用以及长度值。这样针对多种不同类型进行编程就很容易：std::span\<uint8_t>可以指向std::vector\<uint8_t>、std::array\<uint8_t, N>拥有的内存，甚至可以指向原始指针拥有的内存。不检查边界就会导致安全漏洞，而许多情况下，span能帮你确保长度是正确的。

与其他STL数据结构一样，span的operator\[\]方法并不会进行任何边界检查。这是可以理解的，因为operator\[\]是最常用的方法，也是访问数据结构的默认方法。而至少从理论上，std::vector和std::array可以安全地使用，因为它们提供了at()方法，该方法会进行边界检查（在实践中我从来没见人用过这个方法，不过可以想象一个项目，通过静态分析工具来禁止调用std::vector<T>::operator\[\]）。span不提供at()方法，也不提供任何进行边界检查的方法。

有趣的是，Firefox和Chromium移植的std::span都会在operator\[\]中进行边界检查，所以这两个项目也无法安全地移植到std::span上。

**结论**

现代C++的惯用做法带来了许多改变，能够改善安全性：智能指针能更好地表示预想的生命周期，std::span能保证永远有正确的长度，std::variant为union提供了安全的抽象。但是，现代C++也引入了一些新的漏洞祸根：lambda捕获导致的释放后使用，未初始化的optional，以及没有边界检查的span。

以我编写比较现代的C++的经验，以及审查Rust代码（包括使用了大量unsafe的Rust代码）的经验来看，现代C++的安全性完全比不上那些保证内存安全的语言，如Rust、Swift（或者Python和JavaScript，尽管我很少见到能够合理地用Python或C++编写的程序）。

不可否认，将现有的C和C++代码移植到其他语言依然是个难题。但无论如何，问题应该是我们应该怎样做，而不是我们是否应该做。事实证明，即使最现代的C++惯用做法，也不可能保证C++的正确性。
