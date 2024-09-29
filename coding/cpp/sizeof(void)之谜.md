# 

Original 雨乐 高性能架构探索

 _2024年09月11日 12:07_ _北京_

![](http://mmbiz.qpic.cn/mmbiz_png/p3sYCQXkuHhKgtwWvzaYZodgfpphdA6WWKEMXTn6ImCCCuEzlPKicNBcpzBUyjK1XicWwqIwusqLGpwyyOc87JPQ/300?wx_fmt=png&wxfrom=19)

**高性能架构探索**

专注于分享干货，硬货，欢迎关注😄

93篇原创内容

公众号

你好，我是雨乐~

在 C 和 C++ 中，`void` 类型和 `sizeof` 运算符是每位程序员都应掌握的基本概念。然而，`sizeof(void)` 的概念可能会让人感到困惑，特别是对于这些语言的新手来说。本文将解释 `void` 是什么，为什么 `sizeof(void)` 在传统意义上没有意义，以及这些概念在实际编程中的应用。

## 什么是void？

在 C 和 C++ 中，`void`是用于表示不存在任何类型的关键字。它通常用于三种场景。

### 函数

当某个函数的返回值被声明为void，即意味着该函数不返回任何内容，那么获取函数返回内容的操作将是非法或者未定义的行为。

```
void Print() {  std::cout << "Hello world!!!" << std::endl;}
```

在这个函数中，只有一行输出操作，不返回任何类型。

### 空指针

空指针`void`( `void *`) 是一种特殊类型的指针，可以指向任何数据类型。但是，由于未指定类型，因此在解引用之前必须将其转换为适当的类型。

```
void* ptr;int num = 42;ptr = &num; // void 指针可以指向 int 类型的地址int* intPtr = (int*)ptr; // 需要将 void 指针转换为具体类型
```

### 函数参数

相信很多从事C开发的人，经常会见到形如`fun(void)`这种代码，在这种代码中，void作为函数参数的意思是该函数没有参数，或者说指定函数不接受任何参数，与`fun()`等同，不过这种写法现在已经很少见了~

```
void fun(void) {    // 函数体}
```

## sizeof操作符

在 C++ 中，`sizeof` 操作符是一个编译时运算符，用于获取数据类型或对象的内存大小。它是一个非常重要的工具，可以帮助程序员了解变量、类型或数据结构的内存占用情况。

```
int x = 10;printf("size of int: %d bytes\n", sizeof(int));printf("size of x: %d bytes\n", sizeof(x));
```

## sizeof(void)

好了，前面的一系列铺垫，就是为了引入本节主题。

对于这种问题，我一般会先在本地编码试试。

// size.c

```
#include <stdio.h>int main() {  printf("sizeof void is %d bytes\n", sizeof(void));  return 0;}
```

以及

// size.cc

```
#include <iostream>int main() {  std::cout << "sizeof void is: " << sizeof(void) << std::endl;  return 0;}
```

对于size.c，尝试使用gcc和clang进行编译：

```
clang size.c -o size // clanggcc size.c -o size // gcc
```

输出都是

```
sizeof void is 1 bytes
```

但是，对于size.cc即对于c++版本的，g++可以编译通过，输出同c版本，而clang++则编译失败，输出如下：

```
size.cc:4:38: error: invalid application of 'sizeof' to an incomplete type 'void'    4 |   std::cout << "sizeof void is: " << sizeof(void) << std::endl;      |                                      ^     ~~~~~~1 error generated.
```

说实话，看到编译和运行结果的时候，我是一脸懵的，完全出乎了我的意料~

针对上面的结果，看了某些资料，偶然翻到了标准对这块的回复(**ISO/IEC 9899:2011**)：

> The sizeof operator shall not be applied to an expression that has function type or an incomplete type, to the parenthesized name of such a type, or to an expression that designates a bit-field member.

意思是`sizeof` 操作符不得应用于以下情况的表达式：

•函数类型的表达式•不完整类型的表达式•这种类型的带括号的名称•指定位域成员的表达式

以及：

> **The void type** comprises an empty set of values; it **is an incomplete object type** that cannot be completed.

意思是void是一个不完整类型，那么问题来了，既然标准都说了void是一个不完整类型，那么gcc为什么输出为1呢？

针对这个问题，继续查资料，得到回复如下：

> In GNU C, addition and subtraction operations are supported on pointers to `void` and on pointers to functions. This is done by treating the size of a `void` or of a function as 1.
> 
> A consequence of this is that `sizeof` is also allowed on `void` and on function types, and returns 1.
> 
> The option -Wpointer-arith requests a warning if these extensions are used.

意思是，在 GNU C中，指向 `void` 指针和指向函数的指针支持加法和减法操作。这是通过将 `void` 或函数的大小视为1来实现的。

因此，`sizeof` 操作符也可以用于 `void` 和函数类型，并返回 1。选项 `-Wpointer-arith` 会在使用这些扩展时发出警告。

以上  

如果对本文有疑问可以加笔者**微信**直接交流，笔者也建了C/C++相关的技术群，有兴趣的可以联系笔者加群。  

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

![](http://mmbiz.qpic.cn/mmbiz_png/p3sYCQXkuHhKgtwWvzaYZodgfpphdA6WWKEMXTn6ImCCCuEzlPKicNBcpzBUyjK1XicWwqIwusqLGpwyyOc87JPQ/300?wx_fmt=png&wxfrom=19)

**高性能架构探索**

专注于分享干货，硬货，欢迎关注😄

93篇原创内容

公众号

Reads 1855

​

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/p3sYCQXkuHhKgtwWvzaYZodgfpphdA6WWKEMXTn6ImCCCuEzlPKicNBcpzBUyjK1XicWwqIwusqLGpwyyOc87JPQ/300?wx_fmt=png&wxfrom=18)

高性能架构探索

28659

8

Comment

**留言 8**

- y
    
    上海9/11
    
    Like3
    
    不关注的一律面试不过
    
    高性能架构探索
    
    Author9/11
    
    Like
    
    说了我想说的![[呲牙]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![[呲牙]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
- 繁缕
    
    陕西9/11
    
    Like
    
    有规定，空结构体sizeof为1
    
    高性能架构探索
    
    Author9/11
    
    Like1
    
    那个是因为可以定义空结构体数组，void可不能
    
    秒速五公里
    
    四川9/11
    
    Like
    
    回复 **高性能架构探索**：感觉像个占位符，占位符统一为1字节
    
- 何成～
    
    广东9/11
    
    Like
    
    内存长度
    
    高性能架构探索
    
    Author9/11
    
    Like1
    
    比如？
    
- 繁缕
    
    陕西9/11
    
    Like
    
    C对void规定不严格，void*还能++
    

已无更多数据