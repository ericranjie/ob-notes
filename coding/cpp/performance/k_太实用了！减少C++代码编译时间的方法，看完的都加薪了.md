c++ 的代码包含头文件和实现文件两部分， 头文件一般是提供给别人（也叫客户）使用的， 但是一旦头文件发生改变，不管多小的变化，所有引用他的文件就必须重新编译，编译就要花时间，假如你做的工程比较大（比如二次封装chrome这类的开发）,重新编译一次的时间就会浪费上班的大部分时间，这样干了一天挺累的， 但是你的老板说你没有产出，结果你被fired， 是不是很怨啊， 如果你早点看到这段文章，你就会比你的同事开发效率高那么一些，那样被fired就不会是你了，你说这篇文章是不是价值千金！开个玩笑 ：）

言归正传，怎样介绍编译时间呢， 我知道的就3个办法：

1.删除不必要的#include,替代办法 使用前向声明 (forward declared )

2.删除不必要的一大堆私有成员变量，转而使用 "impl" 方法

3.删除不必要的类之间的继承

为了讲清楚这3点，还是举个实例比较好,这个实例我会一步一步的改进（因为我也是一点一点摸索出来了，如果哪里说错了， 你就放心的喷吧，我会和你在争论到底的，呵呵）

现在先假设你找到一个新工作，接手以前某个程序员写的类，如下

```cpp
// old.h: 这就是你接收的类
//
#include <iostream>
#include <ostream>
#include <list>

// 5 个 分别是file , db, cx, deduce or error ， 水平有限没有模板类
// 只用 file and cx 有虚函数.
#include "file.h" // class file
#include "db.h" // class db
#include "cx.h" // class cx
#include "deduce.h" // class deduce
#include "error.h" // class error

class old : public file, private db {
public:
old( const cx& );
db get_db( int, char* );
cx get_cx( int, cx );
cx& fun1( db );
error fun2( error );
virtual std::ostream& print( std::ostream& ) const;
private:
std::list<cx> cx_list_;
deduce deduce_d_;
};
inline std::ostream& operator<<( std::ostream& os,const old& old_val )
{ return old_val.print(os); }
```

这个类看完了， 如果你已经看出了问题出在哪里， 接下来的不用看了， 你是高手， 这些基本知识对你来说太小儿科，要是像面试时被问住了愣了一下，请接着看吧

**▍先看怎么使用第一条： 删除不必要的#include**

这个类引用 5个头文件， 那意味着那5个头文件所引用的头文件也都被引用了进来， 实际上， 不需要引用5 个，只要引用2个就完全可以了

**1.删除不必要的#include,替代办法 使用前向声明 (forward declared )**

**1.1删除头文件 iostream，** 我刚开始学习c++ 时照着《c++ primer》 抄，只要看见关于输入，输出就把 iostream 头文件加上， 几年过去了， 现在我知道不是这样的， 这里只是定义输出函数， 只要引用ostream 就够了

**1.2.ostream头文件也不要，** 替换为 iosfwd , 为什么， 原因就是， 参数和返回类型只要前向声明就可以编译通过， 在iosfwd 文件里 678行（我的环境是vs2013，不同的编译环境具体位置可能会不相同，但是都有这句声明） 有这么一句

```cpp
typedef basic_ostream<char, char_traits<char> > ostream;
inline std::ostream& operator<<( std::ostream& os,const old& old_val )
{ return old_val.print(os); }
```

除此之外，要是你说这个函数要操作ostream 对象， 那还是需要#include <ostream> , 你只说对了一半， 的确， 这个函数要操作ostream 对象， 但是请看他的函数实现，

里面没有定义一个类似 std::ostream os, 这样的语句，话说回来，但凡出现这样的定义语句， 就必须#include 相应的头文件了 ，因为这是请求编译器分配空间，而如果只前向声明 class XXX; 编译器怎么知道分配多大的空间给这个对象！

看到这里， old.h头文件可以更新如下了：

```cpp
// old.h: 这就是你接收的类
//
#include <iosfwd> //新替换的头文件
#include <list>

// 5 个 分别是file , db, cx, deduce or error ， 水平有限没有模板类
// 只用 file and cx 有虚函数.
#include "file.h" // class file ， 作为基类不能删除，删除了编译器就不知道实例化old 对象时分配多大的空间了
#include "db.h" // class db， 作为基类不能删除，同上
#include "cx.h" // class cx
#include "deduce.h" // class deduce
// error 只被用做参数和返回值类型， 用前向声明替换#include "error.h"
class error；

class old : public file, private db {
public:
old( const cx& );
db get_db( int, char* );
cx get_cx( int, cx );
cx& fun1( db );
error fun2( error );
virtual std::ostream& print( std::ostream& ) const;
private:
std::list<cx> cx_list_; // cx 是模版类型，既不是函数参数类型也不是函数返回值类型，所以cx.h 头文件不能删除
deduce deduce_d_; // deduce 是类型定义，也不删除他的头文件
};
inline std::ostream& operator<<( std::ostream& os,const old& old_val )
{ return old_val.print(os); }
```

到目前为止， 删除了一些代码， 是不是心情很爽，据说看一个程序员的水平有多高， 不是看他写了多少代码，而是看他少写了多少代码。

如果你对C++ 编程有更深一步的兴趣， 接下来的文字你还是会看的，再进一步删除代码， 但是这次要另辟蹊径了

**▍2.删除不必要的一大堆私有成员变量，转而使用 "impl" 方法**

**2.1.使用 "impl" 实现方式写代码，减少客户端代码的编译依赖**

impl 方法简单点说就是把 类的私有成员变量全部放进一个impl 类， 然后把这个类的私有成员变量只保留一个impl\* 指针，代码如下

```cpp
// file old.h
class old {
//公有和保护成员
// public and protected members
private:
//私有成员， 只要任意一个的头文件发生变化或成员个数增加，减少，所有引用old.h的客户端必须重新编译
// private members; whenever these change,
// all client code must be recompiled
};
```

改写成这样：

```cpp
// file old.h
class old {
//公有和保护成员
// public and protected members
private:
class oldImpl* pimpl_;
// 替换原来的所有私有成员变量为这个impl指针，指针只需要前向声明就可以编译通过，这种写法将前向声明和定义指针放在了一起， 完全可以。
//当然，也可以分开写
// a pointer to a forward-declared class
};

// file old.cpp
struct oldImpl {
//真正的成员变量隐藏在这里， 随意变化， 客户端的代码都不需要重新编译
// private members; fully hidden, can be
// changed at will without recompiling clients
};
```

不知道你看明白了没有， 看不明白请随便写个类试验下，我就是这么做的，当然凡事也都有优缺点，下面简单对比下：

[Untitled](https://www.notion.so/d2dd5631b0784137938b7a9abdac0dbc?pvs=21)

改为impl实现后是这样的：

```cpp
// 只用 file and cx 有虚函数.
#include "file.h"
#include "db.h"
class cx;
class error；

class old : public file, private db {
public:
old( const cx& );
db get_db( int, char* );
cx get_cx( int, cx );
cx& fun1( db );
error fun2( error );
virtual std::ostream& print( std::ostream& ) const;
private:
class oldimpl* pimpl; //此处前向声明和定义
};
inline std::ostream& operator<<( std::ostream& os,const old& old_val )
{ return old_val.print(os); }

//implementation file old.cpp
class oldimpl{
std::list<cx> cx_list_;
deduce dudece_d_;
};
```

**▍3.删除不必要的类之间的继承**

面向对象提供了继承这种机制，但是继承不要滥用， old class 的继承就属于滥用之一， class old 继承file 和 db 类， 继承file是公有继承，继承db 是私有继承，继承file 可以理解， 因为file 中有虚函数， old 要重新定义它， 但是根据我们的假设， 只有file 和 cx 有虚函数，私有继承db 怎么解释？! 那么唯一可能的理由就是：

通过私有继承—让某个类不能当作基类去派生其他类，类似Java里final关键字的功能，但是从实例看，显然没有这个用意， 所以这个私有继承完全不必要， 应该改用包含的方式去使用db类提供的功能, 这样就可以把"db.h"头文件删除， 把db 的实例也可以放进impl类中，最终得到的类是这样的：

```cpp
// 只用 file and cx 有虚函数.
#include "file.h"
class cx;
class error;
class db;
class old : public file {
public:
old( const cx& );
db get_db( int, char* );
cx get_cx( int, cx );
cx& fun1( db );
error fun2( error );
virtual std::ostream& print( std::ostream& ) const;
private:
class oldimpl* pimpl; //此处前向声明和定义
};
inline std::ostream& operator<<( std::ostream& os,const old& old_val )
{ return old_val.print(os); }

//implementation file old.cpp
class oldimpl{
std::list<cx> cx_list_;
deduce dudece_d_;
};
```

**小结一下：**

这篇文章只是简单的介绍了减少编译时间的几个办法：

1.删除不必要的#include,替代办法 使用前向声明 (forward declared )

2.删除不必要的一大堆私有成员变量，转而使用 "impl" 方法

3.删除不必要的类之间的继承

这三条希望对您有所帮助，也欢迎您发表评论， 大家一起讨论进步。

来源：CNblogs，作者：还在想啊
