

CPP开发者

 _2022年03月20日 11:50_

The following article is from 云加社区 Author 沈芳

[

![](http://wx.qlogo.cn/mmhead/Q3auHgzwzM4VrfJGRic9cMlydQkzsTsDFptqtib3k4CxI3TOVia4Nmicpw/0)

**云加社区**.

腾讯云官方社区公众号，汇聚技术开发者群体，分享技术干货，打造技术影响力交流社区。

](https://mp.weixin.qq.com/s?__biz=MzAxNDI5NzEzNg==&mid=2651170617&idx=1&sn=cd2ce5cdf7c540e4f400000077590baa&chksm=80647666b713ff7082e304116139723f739d6ed382d39a8291226b335d4208d3b75bdb2c0f31&mpshare=1&scene=24&srcid=0320qIOGI7xdqcP0zRZUFYiJ&sharer_sharetime=1647757682337&sharer_shareid=5fb9813bfe9ffc983435bfc8d8c5e9ca&key=daf9bdc5abc4e8d0d199f1bdef64fa58e77bdd3a4190b6b66bdeb63ac45e8fadad88984e6e6ee66294a25f7e39cbf456c3a8189f116f18ff3cb031213826d22aff9c866ebe003b1cbebd2770e59bfe63dd85ce772e0e5f4694633f4392896d3797667902f9de7bddba6e51a38f662b7afcfa32ee9bcd92439a9329b69bc858a9&ascene=14&uin=MTEwNTU1MjgwMw%3D%3D&devicetype=iMac+MacBookAir10%2C1+OSX+OSX+14.6.1+build(23G93)&version=13080710&nettype=WIFI&lang=en&session_us=gh_e829fd9228a9&countrycode=CN&fontScale=100&exportkey=n_ChQIAhIQLZxIqfqCYhrHGDfDyIQokxKBAgIE97dBBAEAAAAAAAc3Oi5bDv4AAAAOpnltbLcz9gKNyK89dVj0XSM%2BsB9Qv%2FBQ1hCMbFbj%2B06EXtMBYsK98FdHYOf3qCB0J3yzmYrKZXbPcgAa%2FjlBFF49PEQ%2FNnD9aHsjGvej2d6oHi42Ajegn9JBrpVjkZH7tmfq2GjiUu9D8WjRbPGDU%2F4fw5pypwxldPvMZOQW8ec7d1k5vI%2FWNX7mT5yS%2FtGSkqsWWlNQ2M5nxub0SNeuk0nZroFK%2Fz%2FHQ8qOOMF44LGCW9boDsGmRhs6oT8i9rVhn4mj6C3fV6Sjq%2FCCOIpmlXB4vd8yd%2BTg9FlD7F7z0BczB4Zfq8rmfZP9&acctmode=0&pass_ticket=EyjEwA%2FoC%2Bf0YVOgi%2FwCSNEUKV4qxD%2BbE230qN1rnW1FecSwiF3HKsZR2aGcm8xX&wx_header=0#)

导语 | 本文将深入Function这部分进行介绍，主要内容是如何利用模板完成对C++函数的类型擦除，以及如何在运行时调用类型擦除后的函数。有的时候我们需要平衡类型擦除与性能的冲突，所以本文也会以lua function wrapper这种功能为例，简单介绍这部分。  

  

在上篇《[_**C++反射：全面解读property的实现机制！**_](http://mp.weixin.qq.com/s?__biz=MzAxNDI5NzEzNg==&mid=2651170588&idx=1&sn=a8c3ac76c6b88e69fe7d45f0c1c0327f&chksm=80647643b713ff5538a6d6bfdb26d5d23616fc50e1a39f17f3131063f6360df9dd98e208ec30&scene=21#wechat_redirect)》中我们对反射中的Property实现做了相关的介绍，本篇将深入Function这部分进行介绍。

  

**一、 Function示例代码**

  

```
//-------------------------------------
```

  

### **（一）注册的代码**

  

上述代码中，我们通过__register_type<T>()创建的ClassBuilder提供的。function(name，func)函数来完成注册。

  

```
__register_type<Vector3>("Vector3").function("DotProduct", &Vector3::DotProduct);
```

  

上例中我们就将Vector3::DotProduct()函数注册到MetaClass中了。

  

  

### **（二）使用的代码**

  

运行时我们获取到的也是类型擦除后的Function对象，如上例中的 dotProductFunc，所以运行时我们需要通过runtime命名空间下提供的辅助设施runtime::call()来完成对应函数的调用，c++的动态版函数类型擦除后的入口参数是统一的Args，出口参数是Value，runtime::call()提供了任意输入参数到Args的转换，如下所示, 我们即可完成对obj对象上的DotProduct函数的调用:

  

```
auto dotRet = runtime::Call(*dotProductFunc, obj, otherVec);
```

  

  

### **（三）整体文章的展开思路**

  

本篇文章的展开思路与Property那篇基本保持一致:

  

- 一些基本知识。
    

  

- 运行时函数的表达-Function类。
    

  

- 反射函数的注册。
    

  

- Lua版本反射函数的实现。
    

  

- 反射函数的运行时分析。
    

  

  

**二、 基本知识**

  

Function Traits和Type Traits在c++11推出后都逐渐变得成熟，一个适配C++14/17的函数&类型萃取库对于像反射这种库也是至关重要的，但Function Traits和Type Traits本质还是依赖SIFINAE做各种类型特化和推导，属于细节非常多但真正的技巧比较少的部分，本文就直接略过对Function Traits和Type Traits细节的分析推导，假定Function Traits和Type Traits已经是成熟稳定的代码部分，我们基于这部分稳定代码做上层的设计编码。

  

另外本文主要分析**函数部分的处理过程**，所以主要关注Function Traits的提供的特性，而不对每种函数的特化实现进行展开。

  

反射库所使用的TFunctionTratis包含的主要信息如下图所示:

  

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

### **（一）TFunctionTraits<>::kind**

  

FunctionKind枚举，主要有以下值:

  

```
/**
```

  

  

###  **（二）TFunctionTraits<>::ExposedType**

  

返回值类型。

  

  

### **（三）TFunctionTraits<>::Details::FunctionCallTypes**

  

std::tuple<>类型，函数所有参数的tuple<>类型，注意类的成员函数首个参数是类对象本身。

  

  

**三、 运行时函数的表达——Function类**

  

为了实现类中函数的动态调用过程，我们需要对类的成员函数进行类型擦除，形成统一的MetaFunction后，方便运行时获取和调用，以获得运行时的动态调用能力。在framework反射库的实现中，**Function是一个虚基类**，定义如下(节选):

  

```
class Function : public Type {
```

  

接口包括获取函数名，父类名，也包括像获取调用参数个数，类型，返回值类型这些常规方法，不一一列举了。需要注意的是并没有Invoke的方法，这个主要是因为不同用途(如纯C++的调用，和for lua的Invoke，类型擦除后的调用方式会略有差异)。C++的调用(依托Args和Value来完成调用参数和返回值类型的统一):

  

```
 virtual Value Execute(const Args& args) const = 0;
```

  

lua的调用(依托Lua虚拟机的调用机制来完成函数类型的统一):

  

```
virtual int CallStraight(lua_State* L) const = 0;
```

  

  

**四、反射函数的注册**

  

函数的注册过程本质上是类的成员函数，经由类型擦除后，变为统一的类型(上一节中Function对象)存入MetaClass中组织起来，方便运行时动态使用的过程。大致流程如下(略过declare<>获取ClassBuilder的这步)

  

### **（一）从ClassBuilder创建一个function说起**

  

```
template <typename T>
```

  

  

**（二）由newFunction()到FunctionImpl()，真正实现函数类型擦除的地方**  

  

```
// Used by ClassBuilder to create new function instance.
```

  

(注意此处对FuncTraits的使用，另外框架相关单元测试里也给出了大量的Ponder Type Traits的测试代码。)

  

  

### **（三）FunctionImpl()的具体实现**

  

```
FunctionImpl(IdRef name, F function, P... policies) : Function(name)
```

  

注意ponder实现函数多用途的方式，用了一个枚举的模板和相关的特化实现，打开Lua支持后，会执行两次processUses<>，分别对应processUses<uses::Uses::eRuntimeModule>()和processUses< uses::Uses::eLuaModule >，一个用来实现标准的C++反射支持，另外一个则是用于Lua的导出支持。

  

这个地方的实现比较复杂，Ponder借助了一些辅助的设施来完成同一函数不同用途的注册方式的分离，我们先来看一下这些辅助设施的定义，再结合processUses<>()简单说明实现机制:

  

```
/**
```

  

此处定义了两个tuple，根据相关的定义也能大概猜到，大致是通过定义的enum值去匹配相关tuple中不同位置type的一种做法，能够比较好的实现基于enum->tuple index->types的一种dispatcher，compiler阶段就能完成的匹配，还是比较巧妙的，后续会结合具体的代码说明这部分的详细使用。

  

  

### **（四）processUse<>的具体实现**

  

processUses<>的代码实现如下:

  

```
uses::Uses::PerFunctionUserData m_userData;
```

  

主要是对上文中的Uses结构体中的两个tuple类型的使用(Uses::PerFunctionData，Uses::Users)，以枚举值 eRuntimeModule，eLuaModule作为processUses的非类型模板参数，两次调用该模板函数，我们即可得到两个不同类型的FunctionCaller存储至m_userData，这部分只包含了对tuple的访问(std::tuple_element<>，std::get<>())，通过Uses结构体的特殊构造和tuple的辅助函数，可以借助不同的enum值来完成不同用途和不同类型的FunctionCaller的生成和存储。大部分是编译期行为，很值得借鉴的一种方式。

  

下面我们来具体看一下Ponder完成函数类型擦除的过程，也就是上述Process::template perFunction<>()的具体实现 (注意此处template关键字的作用是告诉编译器perFunction本身也是模板函数，不加在GCC等编译器上可能会报错)。

  

  

### **（五）C++版本反射函数的实现(RuntimeUse::perFunction())**

  

我们先来看一下RuntimeUse::perFunction()的实现:

  

```
struct RuntimeUse
```

  

perFunction的作用主要是完成对不同函数(参数与返回值可能都不一样)的类型擦除，形成统一类型的FunctionCaller。下面我们具体来看一下FunctionCallerImpl<>的具体实现。

  

Ponder C++反射实现函数类型擦除的方式比较特殊，不是通过得到一个统一类型的函数对象来实现的类型擦除，而是通过类继承和虚函数的方式来实现的类型擦除，代码如下:

  

```
//-----------------------------------------------------------------------------
```

  

如上所示，特化的FunctionCallerImpl<>会实现基类的Value excute(const Args& args)方法，基类的excute方法的参数和返回值是固定的，这样我们针对不同的函数会最终得到一个有统一excute()函数的FunctionCaller对象，间接完成了函数的类型擦除。(另外一种方式是通过模板推导存储一个固定参数表和返回值的lambda，也可以完成函数的类型擦除)

  

我们上述仅介绍了ponder内部最终存储函数的方式和基本的使用形式( 统一的excute()接口)，具体的函数到最终存储形式的过程被忽略了，这里基于前文提到的成熟的Function Traits功能展开一下中间的处理部分。

  

- #### **FunctionWrapper<>模板类**
    

  

通过FunctionWrapper<>模板类完成std::function<>函数对象的生成以及统一参数和返回值的call<>()方法的支持。注意FunctionCallerImpl中对FunctionWrapper类的使用:

  

```
typedef typename FTraits::Details::FunctionCallTypes CallTypes;
```

  

注意此处使用Function Traits直接为FunctionWrapper提供参数列表和返回值(FunctionTraits<>::Details::FunctionCallTypes和FunctionTraits<>::ExposedType)。

  

FunctionWrapper的代码以及使用到的CallHelper的实现代码如下:

  

```
template <typename R, typename FTraits, typename FPolicies>
```

  

此处重点关注std::make_index_sequence<>和std::index_sequence<>的使用，借助index_sequence相关的函数，我们可以很方便的对varidic template进行处理，此处通过index_sequence的使用，我们可以很好的完成args中包含的arg到函数需要的正确类型参数的转换:

  

```
ConvertArgs<A>::convert(args, Is)...
```

  

ConvertArgs<>和ChooseCallReturner<>一个是将从args中取到的Value置换为具体类型的参数，一个是将具体类型的返回值置换为Value，通过这种方式，最终实现了函数的调用参数和返回值的统一，通过这段代码，我们也能看到在C++14/17后，相关的函数类型擦除的代码对比原来的实现会简化非常多，已经很容易理解了。

  

另外，对于没有返回值的函数，也有专门特化的CallHelper，代码如下:

  

```
// Specialization of CallHelper for functions returning void
```

  

对比有返回值的版本，差异主要是直接返回Value::nothing，所以我们也可以简单的通过call的返回值是否为Value::nothing来判断反射函数是否有返回值，这也是Rpc库使用的方式。

  

上面我们有提到ConvertArgs<>和ChooseCallReturner<>，通过这两者我们很好的实现了调用函数的参数统一以及返回值统一，这里我们也对其实现做一下具体的拆解，当然, 主要的类型转换的实现其实更多的是依赖Value和UserObject本身的实现，此处我们不对这两者做具体的展开，与Function Traits一样，我们把这两者当成即有成熟功能，来方便理清函数类型擦除相关的核心代码。

  

- #### **ConvertArgs<>模板类**
    

  

CovertArgs<>整体实现代码如下:

  

```
//-----------------------------------------------------------------------------
```

  

首先是template<int TFrom，typename TTo>struct ConvertArg的实现，前面的TForm是ValueKind值，后面的TTo是目标类型，对于非User类型的Value，模板推导出的是最前面的实现，最后直接执行Value::to<>()模板函数来完成Value到目标类型的转换，注意此处对于Covert错误的处理是直接抛异常。后续的两个特化实现分别针对reference和const reference，主要依赖UserObject的ref<>()和cref<>()模板函数，最后就是CallHelper<>模板类使用到的的template<typename A>struct ConvertArgs实现，其实就是对template<int TFrom，typename TTo>struct ConvertArg的简单包装。

  

- #### **ChooseCallReturner<>模板类**
    

  

ChooseCallReturner<>的具体实现代码如下:

  

```
//-----------------------------------------------------------------------------
```

  

此处注意注意Return Policy的实现，通过policy::ReturnCopy和policy::ReturnInternalRef我们可以控制Value的创建方式，默认是Copy方式创建Value，其余的主要是Value本身支持从不同类型T构造的特性来完成的。

  

Value对不同类型T的支持特性可以自行查阅Value的实现，目前版本的Value的内部通过ponder自己实现的variants来完成对不同类型T的存取，但其实第一版的ponder重度依赖boost，所以第一版的实现也是直接使用的boost::variants，后续V2版本解除了对boost的依赖，但variants的实现也大量参考了boost的实现，所以对这部分细节感兴趣的可以直接查阅boost::variants相关的文档和源码，更容易理解其中的细节。

  

  

**五、 Lua版本反射函数的实现**

 **——LuaUse::perFunction()**

  

LuaUse::perFunction()的目的与C++反射函数的目的一致，也是完成对普通函数的类型擦除，形成统一的函数对象类型，只是生成的统一FunctionCaller对象不同。

  

```
struct LuaUse
```

  

具体的实现与上一节的很多地方都一样，我们主要关注针对Lua的那部分特性。

  

```
// Base for runtime function caller
```

  

首先看到的差异点是FunctionCaller对象上的m_luaFunc成员:

  

```
int (*m_luaFunc)(lua_State*);
```

  

以及pushFunction()成员函数:

  

```
 void pushFunction(lua_State* L)
```

  

先忽略类型擦除的过程，我们先来看Lua版的FunctionCaller，对比C++的FunctionCaller，差异之处为所有函数会被处理为标准Lua C函数的类型(lua_CFunction类型，int为返回值，lua_State*作为入口参数)，另外通过额外多出来的pushFunction()函数可以将m_luaFunc作为c closure入栈，当然FunctionCaller本身的this指针被当成light userdata作为这个c closure的up value被传入lua虚拟机中。

  

我们接下来看看FunctionCallerImpl，对比C++版的实现，区别最大的是call函数，此处的call函数也是个lua_CFunction类型的函数，同时我们也很容易观察到生成的静态call函数被当成构造函数的参数，最终赋值给了FunctionCaller内的m_luaFunc，我们知道Lua与C++的交互主要是通过lua_State来完成的，要在Lua中调用C++函数，我们需要间接的通过lua_State来传入参数和输出返回值，所以对应的FunctionWrapper对比C++版本也是特殊实现的，并且都带入了lua_State作为额外的参数。类同上文第二部分第四小节, 我们也深入分析FunctionWrapper的实现以及从Lua虚拟机上传入参数以及传出返回值的过程。

  

**(一）FunctionWrapper<>模板类**

  

```
template <typename R, typename FTraits, typename FPolicies>
```

  

与C++版本一致的部分我们不再展开讲解，首先我们注意到与C++版本一样，FunctionCallerImpl中存储的std::function函数对象类型与C++版本实现一致，同样，CallHelper也有无返回值的版本，主要差别是CovertArgs<>()和ChooseCallReturner<>()的实现，都变成了带lua_State参数的版本，原因也是显而意见的，需要通过lua_State来交换需要的数据，Lua版与C++版本的实现主要的差异也在这里，我们接下来具体看看这两个模板函数的实现。

  

  

**（二）CovertArgs<>模板类**

  

```
//-----------------------------------------------------------------------------
```

  

很容易发现Lua版的ConvertArgs仅是对LuaValueReader<>的简单包装和使用，而阅读LuaValueReader的实现发现是对各种数据类型的特化实现，包含了各种lua c api的访问，比较特殊的是对lua table，c++侧的UserObject等的处理，熟悉lua c api的话这些代码都比较容易读懂，此处不再展开了，仅给出string_view的实现供参考:

  

```
template <>
```

  

  

### **（三）ChooseCallReturner<>模板类**

  

```
// Handle returning copies
```

  

ChooseCallReturner<>因为Policy的存在，实现版本较多，此处仅贴出其中一个实现供参考。与CovertArgs一样，ChooseCallRetruner<>也是对LuaValueWriter<>模板类的包装和使用，我们同样给出其中一个LuaValueWriter的实现供参考:

  

```
template <typename T>
```

  

  

### **（四）小结**

  

其实对于c++ ->lua的Wrapper，我们当然可以复用第4节的设施，直接针对:

  

```
virtual Value Execute(const Args& args) const = 0;
```

  

包装lua c function，也是极简单的，但考虑到性能，ponder的做法是复用了相关的Traits实现，重新包装了第5节的Function实现，这样可以得到更高性能的跨语言调用设施。所以很多时候，我们应该是在整个系统不同层面去衡量性价比，像上述代码实现不那么繁复，又能够得到更好的性能的实现，我们肯定会更多考虑。

  

通过上述C++版和Lua版的函数反射实现，我们其实可以发现在Ponder已有的设施下，实现不同目的反射函数变得相当的简单，基于C++版本反射函数的实现思路，可以非常方便的实现其他目的版本的反射函数(如Lua版)，这也是Ponder本身实现的完备和强大之处。

  

另外对于lua bridge来说，光一个function的实现肯定是不够的，后续将会相对完整的介绍怎么基于已有的c++反射特性来实现一个项目级的lua bridge。

  

  

**六、 反射函数的运行时分析**

###   

### **（一）c++::function的执行分析**

  

与Property篇类同，我们也给出一个运行时的分析，方便大家更好的了解整个Function机制的运转方式。运行时的测试选用的依然是最前面示例的代码:

  

```
math::Vector3 otherVec(1.0, 2.0, 3.0);
```

  

简洁起见，仅给出最顶层call stack的展开:

  

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

相关的最顶层代码:

  

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

最终执行的模板实例格式化后如下所示:

  

```
framework::reflection::runtime::detail::TCallHelper<
```

  

通过层层嵌套的模板特化，我们最后完成了运行时函数的动态调用。

  

  

### **（二）lua::function的执行分析**

  

lua::function的执行与c++::function的执行过程非常类同，这里不重复展开，有兴趣的同学可以自行尝试。

  

  

**七、 总结**

  

至此整体反射的实现的理论介绍已经靠一段路，本系列文章后续会继续介绍剩下更侧重应用的几篇:

  

- C++反射深入浅出 - lura的前世今生
    
- C++反射深入浅出 - 反射信息的自动生成
    
- C++反射深入浅出 - 反射的其他应用
    
- C++反射深入浅出 - c++20 concept改造
    

  

**参考资料：**

1.[github ponder库] 

https://github.com/billyquith/ponder

  

- EOF -

推荐阅读  点击标题可跳转

1、[C++ std::function技术浅谈](http://mp.weixin.qq.com/s?__biz=MzAxNDI5NzEzNg==&mid=2651161055&idx=1&sn=293e88aa38405b70b93d1bccec1a3608&chksm=80645080b713d99611b4b5c151466beea653330113d7ccb38d0650091a84616bf3784190201a&scene=21#wechat_redirect)

2、[关于std::function，几个行之有效的扩展小技巧](http://mp.weixin.qq.com/s?__biz=MzAxNDI5NzEzNg==&mid=2651162493&idx=2&sn=2bf1037d416a6435dc2622f6f724ae53&chksm=80645622b713df34c05af73fae02b18e3cebe912c0614faa27386dcbf3b994fe00413ce28c00&scene=21#wechat_redirect)

3、[如何优雅地实现 C++ 编译期静态反射](http://mp.weixin.qq.com/s?__biz=MzAxNDI5NzEzNg==&mid=2651160215&idx=1&sn=da590aeb1d3c4abc4b9f162e07fbd5d6&chksm=8064afc8b71326de8a15a2966c291c23b8f23a2147c48cb25aa814d531112f1113f3fd3b3aa2&scene=21#wechat_redirect)

  

**关注『CPP开发者』**  

看精选C/C++技术文章 

![](http://mmbiz.qpic.cn/mmbiz_png/pldYwMfYJpia3uWic6GbPCC1LgjBWzkBVqYrMfbfT6o9uMDnlLELGNgYDP496LvDfiaAiaOt0cZBlBWw4icAs6OHg8Q/300?wx_fmt=png&wxfrom=19)

**CPP开发者**

我们在 Github 维护着 9000+ star 的C语言/C++开发资源。日常分享 C语言 和 C++ 开发相关技术文章，每篇文章都经过精心筛选，一篇文章讲透一个知识点，让读者读有所获～

24篇原创内容

公众号

点赞和在看就是最大的支持❤️  

Reads 3275

​

Comment

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/pldYwMfYJpia3uWic6GbPCC1LgjBWzkBVqYrMfbfT6o9uMDnlLELGNgYDP496LvDfiaAiaOt0cZBlBWw4icAs6OHg8Q/300?wx_fmt=png&wxfrom=18)

CPP开发者

3Share4

Comment

Comment

**Comment**

暂无留言