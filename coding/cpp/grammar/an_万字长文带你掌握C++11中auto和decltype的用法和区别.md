_2024年04月26日 08:30_ _上海_

**C语言与CPP编程**

专注分享C语言与C++编程知识，内容包括：C/C++入门、进阶、深入、校招、社招以及程序员成长那些事，关注我，编程路上不迷路！

# **一、C++ auto类型推导完全攻略**

在 C++11 之前的版本（C++98 和 C++ 03）中，定义变量或者声明变量之前都必须指明它的类型，比如 int、char 等；但是在一些比较灵活的语言中，比如 C#、JavaScript、PHP、Python 等，程序员在定义变量时可以不指明具体的类型，而是让编译器（或者解释器）自己去推导，这就让代码的编写更加方便。

C++11 为了顺应这种趋势也开始支持自动类型推导了！C++11 使用 auto 关键字来支持自动类型推导。

**1、auto 类型推导的语法和规则**

在之前的 C++ 版本中，auto 关键字用来指明变量的存储类型，它和 static 关键字是相对的。auto 表示变量是自动存储的，这也是编译器的默认规则，所以写不写都一样，一般我们也不写，这使得 auto 关键字的存在变得非常鸡肋。

C++11 赋予 auto 关键字新的含义，使用它来做自动类型推导。也就是说，使用了 auto 关键字以后，编译器会在编译期间自动推导出变量的类型，这样我们就不用手动指明变量的数据类型了。

auto 关键字基本的使用语法如下：

```cpp
auto name = value;
```

name 是变量的名字，value 是变量的初始值。

注意：auto 仅仅是一个占位符，在编译器期间它会被真正的类型所替代。或者说，C++ 中的变量必须是有明确类型的，只是这个类型是由编译器自己推导出来的。

auto 类型推导的简单例子：

```cpp
auto n = 10;
auto f = 12.8;
auto p = &n;
auto url = “http://c.biancheng.net/cplus/”;
```

下面我们来解释一下：

第 1 行中，10 是一个整数，默认是 int 类型，所以推导出变量 n 的类型是 int。

第 2 行中，12.8 是一个小数，默认是 double 类型，所以推导出变量 f 的类型是 double。

第 3 行中，&n 的结果是一个 int\* 类型的指针，所以推导出变量 p 的类型是 int\*。

第 4 行中，由双引号""包围起来的字符串是 const char\* 类型，所以推导出变量 url 的类型是 const char\*，也即一个常量指针。

我们也可以连续定义多个变量：

int n = 20;

auto \*p = &n, m = 99;

先看前面的第一个子表达式，&n 的类型是 int\*，编译器会根据 auto \*p 推导出 auto 为 int。后面的 m 变量自然也为 int 类型，所以把 99 赋值给它也是正确的。

这里我们要注意，推导的时候不能有二义性。在本例中，编译器根据第一个子表达式已经推导出 auto 为 int 类型，那么后面的 m 也只能是 int 类型，如果写作m=12.5就是错误的，因为 12.5 是double 类型，这和 int 是冲突的。

还有一个值得注意的地方是：使用 auto 类型推导的变量必须马上初始化，这个很容易理解，因为 auto 在 C++11 中只是“占位符”，并非如 int 一样的真正的类型声明。

**2、auto 的高级用法**

auto 除了可以独立使用，还可以和某些具体类型混合使用，这样 auto 表示的就是“半个”类型，而不是完整的类型。请看下面的代码：

```cpp
int  x = 0; auto *p1 = &x;   //p1 为 int *，auto 推导为 int
auto  p2 = &x;   //p2 为 int*，auto 推导为 int* 
auto &r1  = x;   //r1 为 int&，auto 推导为 int 
auto r2 = r1;    //r2 为  int，auto 推导为 int
```

下面我们来解释一下：

第 2 行代码中，p1 为 int\* 类型，也即 auto * 为 int \*，所以 auto 被推导成了 int 类型。

第 3 行代码中，auto 被推导为 int\* 类型，前边的例子也已经演示过了。

第 4 行代码中，r1 为 int & 类型，auto 被推导为 int 类型。

第 5 行代码是需要重点说明的，r1 本来是 int& 类型，但是 auto 却被推导为 int 类型，这表明当=右边的表达式是一个引用类型时，auto 会把引用抛弃，直接推导出它的原始类型。

接下来，我们再来看一下 auto 和 const 的结合：

```
int  x = 0;
```

下面我们来解释一下：

第 2 行代码中，n 为 const int，auto 被推导为 int。

第 3 行代码中，n 为 const int 类型，但是 auto 却被推导为 int 类型，这说明当=右边的表达式带有 const 属性时， auto 不会使用 const 属性，而是直接推导出 non-const 类型。

第 4 行代码中，auto 被推导为 int 类型，这个很容易理解，不再赘述。

第 5 行代码中，r1 是 const int & 类型，auto 也被推导为 const int 类型，这说明当 const 和引用结合时，auto 的推导将保留表达式的 const 类型。

最后我们来简单总结一下 auto 与 const 结合的用法：

当类型不为引用时，auto 的推导结果将不保留表达式的 const 属性；

当类型为引用时，auto 的推导结果将保留表达式的 const 属性。

**3、auto 的限制**

前面介绍推导规则的时候我们说过，使用 auto 的时候必须对变量进行初始化，这是 auto 的限制之一。那么，除此以外，auto 还有哪些其它的限制呢？

- **auto 不能在函数的参数中使用。**

这个应该很容易理解，我们在定义函数的时候只是对参数进行了声明，指明了参数的类型，但并没有给它赋值，只有在实际调用函数的时候才会给参数赋值；而 auto 要求必须对变量进行初始化，所以这是矛盾的。

- **auto 不能作用于类的非静态成员变量（也就是没有 static 关键字修饰的成员变量）中。**

- **auto 关键字不能定义数组，比如下面的例子就是错误的：**

char url\[\] = “http://c.biancheng.net/”;

auto str\[\] = url; //arr 为数组，所以不能使用 auto

- **auto 不能作用于模板参数，请看下面的例子：**

```
template <typename T>
```

**4、auto 的应用**

说了那么多 auto 的推导规则和一些注意事项，那么 auto 在实际开发中到底有什么应用呢？下面我们列举两个典型的应用场景。

- **使用 auto 定义迭代器**

auto 的一个典型应用场景是用来定义 stl 的迭代器。

我们在使用 stl 容器的时候，需要使用迭代器来遍历容器里面的元素；不同容器的迭代器有不同的类型，在定义迭代器时必须指明。而迭代器的类型有时候比较复杂，书写起来很麻烦，请看下面的例子：

```
#include <vector>
```

可以看出来，定义迭代器 i 的时候，类型书写比较冗长，容易出错。然而有了 auto 类型推导，我们大可不必这样，只写一个 auto 即可。

修改上面的代码，使之变得更加简洁：

```
#include <vector>
```

auto 可以根据表达式 v.begin() 的类型（begin() 函数的返回值类型）来推导出变量 i 的类型。

- **auto 用于泛型编程**

auto 的另一个应用就是当我们不知道变量是什么类型，或者不希望指明具体类型的时候，比如泛型编程中。我们接着看例子：

```
#include <iostream>
```

运行结果：

100

http://c.biancheng.net/cplus/

本例中的模板函数 func() 会调用所有类的静态函数 get()，并对它的返回值做统一处理，但是 get() 的返回值类型并不一样，而且不能自动转换。这种要求在以前的 C++ 版本中实现起来非常的麻烦，需要额外增加一个模板参数，并在调用时手动给该模板参数赋值，用以指明变量 val 的类型。

但是有了 auto 类型自动推导，编译器就根据 get() 的返回值自己推导出 val 变量的类型，就不用再增加一个模板参数了。

下面的代码演示了不使用 auto 的解决办法：

```
#include <iostream>
```

**二、C++ decltype类型推导完全攻略**

decltype 是 C++11 新增的一个关键字，它和 auto 的功能一样，都用来在编译时期进行自动类型推导。不了解 auto 用法的读者请转到《C++ auto》。

decltype 是“declare type”的缩写，译为“声明类型”。

既然已经有了 auto 关键字，为什么还需要 decltype 关键字呢？因为 auto 并不适用于所有的自动类型推导场景，在某些特殊情况下 auto 用起来非常不方便，甚至压根无法使用，所以 decltype 关键字也被引入到 C++11 中。

auto 和 decltype 关键字都可以自动推导出变量的类型，但它们的用法是有区别的：

auto varname = value;

decltype(exp) varname = value;

其中，varname 表示变量名，value 表示赋给变量的值，exp 表示一个表达式。

auto 根据=右边的初始值 value 推导出变量的类型，而 decltype 根据 exp 表达式推导出变量的类型，跟=右边的 value 没有关系。

另外，auto 要求变量必须初始化，而 decltype 不要求。这很容易理解，auto 是根据变量的初始值来推导出变量类型的，如果不初始化，变量的类型也就无法推导了。decltype 可以写成下面的形式：

decltype(exp) varname;

- **exp 注意事项**

原则上讲，exp 就是一个普通的表达式，它可以是任意复杂的形式，但是我们必须要保证 exp 的结果是有类型的，不能是 void；例如，当 exp 调用一个返回值类型为 void 的函数时，exp 的结果也是 void 类型，此时就会导致编译错误。

C++ decltype 用法举例：

```
int a = 0;
```

可以看到，decltype 能够根据变量、字面量、带有运算符的表达式推导出变量的类型。读者请留意第 4 行，y 没有被初始化。

**1、decltype 推导规则**

上面的例子让我们初步感受了一下 decltype 的用法，但你不要认为 decltype 就这么简单，它的玩法实际上可以非常复杂。当程序员使用 decltype(exp) 获取类型时，编译器将根据以下三条规则得出结果：

- 如果 exp 是一个不被括号( )包围的表达式，或者是一个类成员访问表达式，或者是一个单独的变量，那么 decltype(exp) 的类型就和 exp 一致，这是最普遍最常见的情况。

- 如果 exp 是函数调用，那么 decltype(exp) 的类型就和函数返回值的类型一致。

- 如果 exp 是一个左值，或者被括号( )包围，那么 decltype(exp) 的类型就是 exp 的引用；假设 exp 的类型为 T，那么 decltype(exp) 的类型就是 T&。

为了更好地理解 decltype 的推导规则，下面来看几个实际的例子。

【实例1】exp 是一个普通表达式：

```
#include <string>
```

这段代码很简单，按照推导规则 1，对于一般的表达式，decltype 的推导结果就和这个表达式的类型一致。

【实例2】exp 为函数调用：

```
//函数声明
```

需要注意的是，exp 中调用函数时需要带上括号和参数，但这仅仅是形式，并不会真的去执行函数代码。

【实例3】exp 是左值，或者被( )包围：

```
using namespace std;
```

这里我们需要重点说一下左值和右值：左值是指那些在表达式执行结束后依然存在的数据，也就是持久性的数据；右值是指那些在表达式执行结束后不再存在的数据，也就是临时性的数据。有一种很简单的方法来区分左值和右值，对表达式取地址，如果编译器不报错就为左值，否则为右值。

**2、decltype 的实际应用**

auto 的语法格式比 decltype 简单，所以在一般的类型推导中，使用 auto 比使用 decltype 更加方便，你可以转到《C++ auto》查看很多类似的例子，本节仅演示只能使用 decltype 的情形。

我们知道，auto 只能用于类的静态成员，不能用于类的非静态成员（普通成员），如果我们想推导非静态成员的类型，这个时候就必须使用 decltype 了。下面是一个模板的定义：

```
#include <vector>
```

单独看 Base 类中 m_it 成员的定义，很难看出会有什么错误，但在使用 Base 类的时候，如果传入一个 const 类型的容器，编译器马上就会弹出一大堆错误信息。原因就在于，T::iterator并不能包括所有的迭代器类型，当 T 是一个 const 容器时，应当使用 const_iterator。

要想解决这个问题，在之前的 C++98/03 版本下只能想办法把 const 类型的容器用模板特化单独处理，增加了不少工作量，看起来也非常晦涩。但是有了 C++11 的 decltype 关键字，就可以直接这样写：

```
template <typename T>
```

看起来是不是很清爽？

注意，有些低版本的编译器不支持T().begin()这种写法，以上代码我在 VS2019 下测试通过，在 VS2015 下测试失败。

**三、汇总auto和decltype的区别**

通过《C++ auto》和《C++ decltype》两节的学习，相信大家已经掌握了 auto 和 decltype 的语法规则以及使用场景，这节我们将 auto 和 decltype 放在一起，综合对比一下它们的区别，并告诉大家该如何选择。

**1、语法格式的区别**

auto 和 decltype 都是 C++11 新增的关键字，都用于自动类型推导，但是它们的语法格式是有区别的，如下所示：

auto varname = value; //auto的语法格式

decltype(exp) varname \[= value\]; //decltype的语法格式

其中，varname 表示变量名，value 表示赋给变量的值，exp 表示一个表达式，方括号\[ \]表示可有可无。

auto 和 decltype 都会自动推导出变量 varname 的类型：

auto 根据=右边的初始值 value 推导出变量的类型；

decltype 根据 exp 表达式推导出变量的类型，跟=右边的 value 没有关系。

另外，auto 要求变量必须初始化，也就是在定义变量的同时必须给它赋值；而 decltype 不要求，初始化与否都不影响变量的类型。这很容易理解，因为 auto 是根据变量的初始值来推导出变量类型的，如果不初始化，变量的类型也就无法推导了。

auto 将变量的类型和初始值绑定在一起，而 decltype 将变量的类型和初始值分开；虽然 auto 的书写更加简洁，但 decltype 的使用更加灵活。

请看下面的例子：

```
auto n1 = 10;
```

这些用法在前面的两节中已经进行了分析，此处就不再赘述了。

**2、对 cv 限定符的处理**

「cv 限定符」是 const 和 volatile 关键字的统称：

- const 关键字用来表示数据是只读的，也就是不能被修改；

- volatile 和 const 是相反的，它用来表示数据是可变的、易变的，目的是不让 CPU 将数据缓存到寄存器，而是从原始的内存中读取。

在推导变量类型时，auto 和 decltype 对 cv 限制符的处理是不一样的。decltype 会保留 cv 限定符，而 auto 有可能会去掉 cv 限定符。

以下是 auto 关键字对 cv 限定符的推导规则：

- 如果表达式的类型不是指针或者引用，auto 会把 cv 限定符直接抛弃，推导成 non-const 或者 non-volatile 类型。

- 如果表达式的类型是指针或者引用，auto 将保留 cv 限定符。

下面的例子演示了对 const 限定符的推导：

//非指针非引用类型

```
const int n1 = 0;
```

在 C++ 中无法将一个变量的完整类型输出，我们通过对变量赋值来判断它是否被 const 修饰；如果被 const 修饰那么赋值失败，如果不被 const 修饰那么赋值成功。虽然这种方案不太直观，但也是能达到目的的。

n2 赋值成功，说明不带 const，也就是 const 被 auto 抛弃了，这验证了 auto 的第一条推导规则。p2 赋值失败，说明是带 const 的，也就是 const 没有被 auto 抛弃，这验证了 auto 的第二条推导规则。

n3 和 p3 都赋值失败，说明 decltype 不会去掉表达式的 const 属性。

**3、对引用的处理**

当表达式的类型为引用时，auto 和 decltype 的推导规则也不一样；decltype 会保留引用类型，而 auto 会抛弃引用类型，直接推导出它的原始类型。请看下面的例子：

```
#include <iostream>
```

运行结果：

10, 10, 20

99, 99, 99

从运行结果可以发现，给 r2 赋值并没有改变 n 的值，这说明 r2 没有指向 n，而是自立门户，单独拥有了一块内存，这就证明 r 不再是引用类型，它的引用类型被 auto 抛弃了。

给 r3 赋值，n 的值也跟着改变了，这说明 r3 仍然指向 n，它的引用类型被 decltype 保留了。

**4、总结**

auto 虽然在书写格式上比 decltype 简单，但是它的推导规则复杂，有时候会改变表达式的原始类型；而 decltype 比较纯粹，它一般会坚持保留原始表达式的任何类型，让推导的结果更加原汁原味。

从代码是否健壮的角度考虑，我推荐使用 decltype，它没有那么多是非；但是 decltype 总是显得比较麻烦，尤其是当表达式比较复杂时，例如：

vector nums;

decltype(nums.begin()) it = nums.begin();

而如果使用 auto 就会清爽很多：

vector nums;

auto it = nums.begin();

在实际开发中人们仍然喜欢使用 auto 关键字（我也这么干），因为它用起来简单直观，更符合人们的审美。如果你的表达式类型不复杂，我还是推荐使用 auto 关键字，优雅的代码总是叫人赏心悦目，沉浸其中。

**四、C++返回值类型后置（跟踪返回值类型）**

在泛型编程中，可能需要通过参数的运算来得到返回值的类型。考虑下面这个场景：

```
template <typename R, typename T, typename U>
```

我们并不关心 a+b 的类型是什么，因此，只需要通过 decltype(a+b) 直接得到返回值类型即可。但是像上面这样使用十分不方便，因为外部其实并不知道参数之间应该如何运算，只有 add 函数才知道返回值应当如何推导。

那么，在 add 函数的定义上能不能直接通过 decltype 拿到返回值呢？

```
template <typename T, typename U>
```

当然，直接像上面这样写是编译不过的。因为 t、u 在参数列表中，而 C++ 的返回值是前置语法，在返回值定义的时候参数变量还不存在。

可行的写法如下：

```
template <typename T, typename U>
```

考虑到 T、U 可能是没有无参构造函数的类，正确的写法应该是这样：

```
template <typename T, typename U>
```

虽然成功地使用 decltype 完成了返回值的推导，但写法过于晦涩，会大大增加 decltype 在返回值类型推导上的使用难度并降低代码的可读性。

因此，在 C++11 中增加了\*\*返回类型后置（trailing-return-type，又称跟踪返回类型）\*\*语法，将 decltype 和 auto 结合起来完成返回值类型的推导。

返回类型后置语法是通过 auto 和 decltype 结合起来使用的。上面的 add 函数，使用新的语法可以写成：

```
template <typename T, typename U>
```

为了进一步说明这个语法，再看另一个例子：

```
int& foo(int& i);
```

如果说前一个例子中的 add 使用 C++98/03 的返回值写法还勉强可以完成，那么这个例子对于 C++ 而言就是不可能完成的任务了。

在这个例子中，使用 decltype 结合返回值后置语法很容易推导出了 foo(val) 可能出现的返回值类型，并将其用到了 func 上。

返回值类型后置语法，是为了解决函数返回值类型依赖于参数而导致难以确定返回值类型的问题。有了这种语法以后，对返回值类型的推导就可以用清晰的方式（直接通过参数做运算）描述出来，而不需要像 C++98/03 那样使用晦涩难懂的写法。

——

EOF

——

你好，我是飞宇，本硕均于某中流985 CS就读，先后于百度搜索、字节跳动电商以及携程等部门担任Linux C/C++后端研发工程师。

最近招聘季快到了，身边很多小伙伴都在摩拳擦掌、跃跃欲试，很多都打算看看新机会，这里推荐一个好朋友阿秀开发的互联网**大厂面试真题解析网站**，支持按照**行业、公司、岗位、科目、考察时间**等查看面试真题，有意者欢迎体验。

如果你明天就要面试了，那我建议你今晚来刷一刷这个网站，说不定就能遇到你明天的面试原题，目前已经有不少人在面试中遇到原题了，具体可以看下链接：[字节跳动后端研发岗面试考察题目Top10](https://mp.weixin.qq.com/s?__biz=Mzk0ODU4MzEzMw==&mid=2247512156&idx=1&sn=7df198a531430ff04755e8a91acea758&source=41&scene=21#wechat_redirect)、[面试中局部性原理还真有用！](https://mp.weixin.qq.com/s?__biz=Mzk0ODU4MzEzMw==&mid=2247512161&idx=1&sn=596d9f4c88e9d2492931d22dc8478151&source=41&scene=21#wechat_redirect)!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

> 网址：https://top.interviewguide.cn/

同时，我也是知乎博主@韩飞宇，日常分享C/C++、计算机学习经验、工作体会，欢迎点击[此处查看](https://mp.weixin.qq.com/s?__biz=MzkxNzQzNDM2Ng==&mid=2247494314&idx=1&sn=1e03593dc5ba6fc9ff62d261ef73af7f&chksm=c142103bf635992dded667f4fcc080afe522b60fbca0f6eeee110408086fd102568850af3c8e&scene=21#wechat_redirect)我以前的学习笔记&经验&分享的资源。

我组建了一些社群一起交流，群里有大牛也有小白，如果你有意可以一起进群交流。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

欢迎你添加我的微信，我拉你进技术交流群。此外，我也会经常在微信上分享一些计算机学习经验以及工作体验，还有一些内推机会。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

加个微信，打开另一扇窗

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

C/C++106

C/C++ · 目录

上一篇Linux 问题故障定位的技巧大全下一篇回调函数(callback)是什么？一文理解回调函数(callback)

阅读 1383

​

写留言

**留言 2**

- 超控中国

  福建4月26日

  赞1

  感觉auto的意思就是把稳定性，可靠性交给编译器？

- Bob Liu

  天津4月26日

  赞

  建议讲最新的标准，我记得14还是17decltype 有变化，直接讲20，23的目前可能落地还差点。

已无更多数据

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/ibLeYZx8Co5JKf72TOeLcba56VknmOtKrMWnS3gyv2Z3RPZ6S28sAtAKSyozOHMDzI8LEkz8ic8eH2v4ZysDq6sQ/300?wx_fmt=png&wxfrom=18)

C语言与CPP编程

8751

2

写留言

**留言 2**

- 超控中国

  福建4月26日

  赞1

  感觉auto的意思就是把稳定性，可靠性交给编译器？

- Bob Liu

  天津4月26日

  赞

  建议讲最新的标准，我记得14还是17decltype 有变化，直接讲20，23的目前可能落地还差点。

已无更多数据
