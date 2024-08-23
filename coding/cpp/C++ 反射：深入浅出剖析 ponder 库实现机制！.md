# 

CPP开发者

 _2022年03月10日 11:50_

The following article is from 云加社区 Author 沈芳

[

![](http://wx.qlogo.cn/mmhead/Q3auHgzwzM4VrfJGRic9cMlydQkzsTsDFptqtib3k4CxI3TOVia4Nmicpw/0)

**云加社区**.

腾讯云官方社区公众号，汇聚技术开发者群体，分享技术干货，打造技术影响力交流社区。

](https://mp.weixin.qq.com/s?__biz=MzAxNDI5NzEzNg==&mid=2651170564&idx=1&sn=063490ec4745d0ca82e13e7cc2d15d4d&chksm=8064765bb713ff4da88b460eaa6050a93c86e9535a13744e12127382b26ca3cf2bf212b83608&mpshare=1&scene=24&srcid=0310PZzN5eHc3RvVRbPfGu5y&sharer_sharetime=1646892779248&sharer_shareid=5fb9813bfe9ffc983435bfc8d8c5e9ca&key=daf9bdc5abc4e8d00c634e1e9180f45ef393cd965bf62577b2dd04cc90b409d95b419df093f7e5e78d4da735c87e8ef842d0aa3d9638216170cd5f5514e1362a2011bfe7b8b444a6da06d8d6b15a406989f221c8afe0d63050a29aa6edd024f3cc14cf8f2f74ef7d191ad56f2617a3ce2ede5ecc240207838e2a0b1ef3578160&ascene=14&uin=MTEwNTU1MjgwMw%3D%3D&devicetype=iMac+MacBookAir10%2C1+OSX+OSX+14.6.1+build(23G93)&version=13080710&nettype=WIFI&lang=en&session_us=gh_e829fd9228a9&countrycode=CN&fontScale=100&exportkey=n_ChQIAhIQ9P9No1lk9GySBD%2BQsyQgiRL%2BAQIE97dBBAEAAAAAAHMeFOk9C8cAAAAOpnltbLcz9gKNyK89dVj0dKzgziuWCLi0JtpEkhr0lcy%2BhJ6oBon9feK%2B9LosK6grxsLZX6z3y61I619QAVm5z5x1aUzANuJx0jY0OoWTR1Y6lKtadTRPqk4JeDslSeIX6TAQfgUY%2BQrNN%2BZflJ8zixK24k8O8ERMaynkVre%2ByzhVMCMIzb7puF4jAIN5VyvyAWw%2BEBmQ1i%2FF0ykvtVSngRc%2Bzradk7jhNS%2BLGxaDHpmvVQ%2FDCLqe8OYuVFZY2JMiyEmtIKRdJqD6uKHvGd0nikeJJXCRzKPNgPAJfAjkGVLsBm6gZSUT&acctmode=0&pass_ticket=jjh0%2F10OuP3SrpS%2FlhkFW8jItaLL0gckhZNu%2BfWp3%2Ffzt%2BAGVk4IRWT2Zzsfv%2BWt&wx_header=0#)

导语 | 给静态语言添加动态特性，似乎是C++社区一件大家乐见其成的事情，轮子也非常多，我们不一一列举前辈们造的各种流派的轮子了，主要还是结合我们框架用到的C++反射实现，结合C++的新特性，来系统的拆解目前框架中的反射实现。另外代码最早脱胎于ponder，整体处理流程基本与原版一致，所以相关的源码可以直接参考[ponder的原始代码](https://github.com/billyquith/ponder)   

  

**一、简单的示例**

  

```
//-------------------------------------
```

  

上面的代码**演示了框架中C++属性和方法的注册****，以及怎么动态的设置获取对象的属性和调用相关的方法了，最后也演示了如何调用对象的overload方法**。我们先不对具体的代码做太细致的拆解，先有个大概印象，注意以下几点：

  

- 对于反射对象的构建，我们使用的代码：
    

  

```
 runtime::createWithArgs(*metaClass, Args{ 1.0, 2.0, 3.0 });
```

  

- 对于属性的获取，我们使用的代码：
    

  

```
const reflection::Property* fieldX = nullptr;
```

  

- 对于函数的调用，我们使用的代码：
    

  

```
const reflection::Function* overFunc = nullptr;
```

  

其中有几个“神奇”的类，Args，Value，UserObject，你注意到这点，那么恭喜你关注到了动态反射实现最核心的点了，我们要做到运行时反射，先得完全类型的“统一”。

  

  

**二、实现概述**

  

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

上图是relection的工程视图，我们重点关注图中标有蓝色数字的几部分，也大概对他们的作用先做简单的描述:

  

- meta: 反射信息的包装呈现的部分。  
    

  

- traits: 用于做compiler time的类型萃取，通过相关的代码，我们可以方便的对类的成员和函数做编译期分析，如获取函数的返回值类型和参数类型。  
    

  

- type_erasure: 为了让反射的使用有统一的接口，我们必须提供基本的类型擦除容器，从而可以运行时动态的调用某个对象的接口或者获取对象的属性。  
    

  

- builder: meta信息不是自动就有的，我们需要一种手段，从原始类去构建相关的meta信息(非侵入式)。  
    

  

- runtime:提供一些相比meta，更友好的接口，如上面看到的 runtime::createWithArgs()方法。
    

  

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

几者的大致关系如上图所示:

  

- traits，type_erasure: 提供最核心的功能支持。  
    

  

- builder: 完成Compiler time对原始类的分析和Meta信息的输出，实际上是compiler time->runtime的桥梁，也相当于是整个反射的脚手架，我们通过builder的实现来完成对反射meta信息的构成。  
    

  

- meta，runtime: 一起完成了运行时信息的表达和提供相关的使用接口。
    

  

下面我们来具体看一下每一部分的实现。

  

  

**三、Meta实现—类C#的表达**

  

聊到运行时系统，我们首先想到的是弱鸡的rtti，功能弱也就算了，很多地方还被禁用，能够运行时获取的信息和相关的设施都十分简陋。

  

另外一点要提的是，C++使用的是compiler time类型完备的设计方式，运行时类型都会被塌陷掉，圈子里的黑话叫“擅长裸奔”。

  

所以我们想要给C++增加动态反射的能力，第一点需要做的就是想办法为它补充足够的运行时信息，也就是类似C#的各种meta data，然后我们就可以愉快的利用动态特性来开发一些更具通用性的序列化反序列化，跨语言支持的功能了。

  

那么运行时创建操作对象，需要哪些信息? 我们直接给出答案：

  

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

整个反射meta部分的设计其实就是为了满足前面说到的，补齐运行时需要的类型相关的各种meta信息。

  

了解了meta信息包含的部分，我们来结合代码看一下具体的实现。

  

### **（一）reflection::Class**

  

```
class Class : public Type
```

  

具体实现跟前面的图基本是一一对应的，这里不展开细述了。

  

  

### **（二）reflection::Function**

  

```
class Function : public Type
```

  

注意此处的Function是一个虚的实现，后面我们会看到真正的Function是由builder来构建出来的。

  

另外一点是meta function没有像C#那样直接给出Invoke方法，这个是因为目前的实现针对不同使用场合，类型擦除的函数是不同的，比如对于lua，类型擦除的函数原型是lua_CFunction。对于C++，则是：

  

```
std::function<Value(Args)>;
```

  

不同场合不同统一类型的好处是不需要Wrapper，没有额外的性能开销，但同时也会导致外围的使用变麻烦，这里可能需要根据项目实际情况做一定的调整。

  

  

### **（三）reflection::Property**

  

```
class Property : public Type
```

  

这个类跟reflection::Function一样，也是一个虚基类，builder会负责具体的Property的实现，真正特化实现的主要是getValue()，setValue()这两个虚接口，外部则直接使用get()，set()接口来完成对对应属性的获取和设置操作。

  

  

### **（四）static property**

  

static property的实现比较简单，目前其实是利用Function来间接实现的，只提供read only类型的static变量。这个地方不详细展开了。

  

  

**四、traits实现—基础工具**

  

这部分实现也是C++新特性引入后“减法”效应比较明显的部分, 也是type erasure的基础。核心部分主要是两块，TypeTraits和FunctionTraits，我们主要侧重于了解两者的功能和使用，忽略实现部分，随着c++20的到来，concept的引入，这部分又可以得到进一步的简化。目前主要是通过C++的SIFINEA特性来完成相关的推导实现，更多是细节的处理相关的代码，了解相关的具体实现价值感觉有限，就不做太细致的展开了。

  

### **（一）TypeTraits**

  

TypeTraits主要用于移除类型的Pointer和Reference等修饰，获取原始类型。另外我们也可以通过其提供的get()，getPointer()函数来快速获取预期类型的数据。

  

```
template <typename T>
```

  

- #### **DataType<>**
    

  

TypeTraits中通过DataType<>间接获取了类型T的DataType(移除*，& 修饰的类型)，DataType的实现如下：

  

```
template <typename T, typename E = void>
```

  

- #### **示例**
    

  

```
using rstudio::reflection::detail::TypeTraits;
```

  

  

### **（二）FunctionTraits**

  

FunctionTraits主要用来完成编译期对函数类型的推导，获取它的返回值类型，参数类型等信息。常见于各种跨语言中间层的实现，如Lua的各种bridge，也是函数类型擦除的起点，我先得知道这个函数本身的信息，才能进一步做type erasure。我们先来看一个例子：

  

```
/*
```

  

这个是对function和function pointer的FucntionTraits<>特化，其他函数类型的特化提供的能力与这个的特化基本一致，主要包含以下信息：

  

- kind: 函数的类型，其实就是FunctionTraits的特化实现各类，详见下文。  
    

  

- Details: 函数的具体信息，如返回值类型，参数表tuple<>等，都存储在其中。  
    

  

- BoundType: 函数类型。  
    

  

- ExposedType: 返回值类型。  
    

  

- ExposedTraits: ExposedType的TypeTraits。  
    

  

- DataType: ExposedType的数据类型。  
    

  

- DispatchType: 配合std::function<>使用，作为std::function的模板参数，这样就可以构造一个与原始Function类型匹配的std::function<>对象了。
    

  

下文对几个重点属性做一下具体的说明：

  

- #### **FunctionTraits<>::kind**
    

  

FunctionKind枚举，主要有以下值：

  

```
/**
```

  

- #### **FunctionTraits<>::Detatils**
    

  

Details本身也是一个模板类型，包含两种实现，FunctionDetails<>和MethodDetails<>，两者主要成员基本一致，MethodDetails<>针对的是类的成员函数，所以会额外多一个ClassType，两者差异很小，我们就只列出FunctionDetails<>的代码了：

  

```
template <typename T>
```

  

实现比较简洁，主要的成员：

  

- ParamTypes: 包含函数所有参数的tuple<>。
    

  

- ReturnType: 函数的返回值类型。
    

  

- FuncType: 函数指针类型。
    

  

- DispatchType: 配合std::function<>一起使用，作为std::function的模板参数，这样就可以构造一个与原始Function类型匹配的函数对象了。
    

  

- FunctionCallTypes: 同ParamTypes，注意对于MethodDetails<>，这两者是有区别的，ParamTypes不包含类本身，MethodDetails首个参数是类本身。
    

  

#### 使用示例：

  

```
using rstudio::reflection::detail::FunctionTraits;
```

  

FunctionTraits的单元测试代码，各种类型的函数的Traits。

  

  

### **（三）ArrayTraits**

  

ponder原始的array traits主要通过ArrayMapper来完成，ArrayMapper用在了ArrayPropertyImpl中，同其他Traits，我们先以std::vector<>的特化实现来看一下ArrayMapper:

  

```
template <typename T>
```

  

ArrayMapper通过各种版本的特化实现提供对各类型Array进行操作的统一接口，如:

  

- size()  
    

  

- get()  
    

  

- set()  
    

  

- insert()  
    

  

- resize()等
    

  

另外对于一个类型，我们也可以简单的通过ArrayMapper<T>::isArray来判断它是否是一个数组。

  

  

### **（四）其它Traits**

  

- #### **IsSmartPointer<>**
    

  

```
template <typename T, typename U>
```

  

判断一个类型是否为smart pointer(可以考虑直接使用concept实现)。

  

- #### **get_pointer()**
    

  

```
template<class T>
```

  

为smart pointer和raw pointer提供获取raw pointer的统一接口。

  

  

**五、type_erasure实现——统一外观**

  

## 我们来回顾一下Meta部分介绍的Property的两个接口：

  

```
virtual Value getValue(const UserObject& object) const = 0;
```

  

很容易看到其中的UserObject和Value, 这基本是整个反射type_erasure实现最核心的两个对象了。其中UserObject用于统一表达通过MetaClass创建的对象，Value则类似variants，用于统一表达反射支持的所有类型的值。通过这两个类型的引入，我们很好的完成了getValue() 和 setValue()对所有类型的统一表达。下面我们来具体介绍一下这两者的实现。

  

### **（一）UserObject**

  

区别于reflection::Class用于表达Meta信息，UserObject其实就是实际数据部分，两者结合在一起，我们就能够获取完整的运行时的类型表达了，可以根据ID或者name动态的构造一个对象，并对它进行属性的获取设置或者接口的调用。

  

```
class UserObject
```

  

估计跟大部分人想的满屏的复杂的实现不一样，UserObject的代码其实比较简单，原因前面说过了，UserObject主要完成对象数据的持有和管理，数据的持有ponder原有实现做得比较简单，直接使用std::shared_ptr<>，通过几种从AbstractObjectHolder继承下来的不同用途的Holder，来完成对数据的持有和生命周期的管理。

  

UserObject的接口大概有几类:

  

- #### **静态构造方法**
    

  

```
template <typename T>
```

  

静态构造一个UserObject，几者的主要区别在于对象生命周期管理的差异：

  

- makeRef() : 不创建对象，间接持有对象，所以可能会有一个对象被UserObject持有的时候，被外界错误的释放导致异常的问题。
    

  

- makeCopy(): 区别于makeRef，Holder内部会直接创建并持有一个相关对象的拷贝，生命周期安全的实现。
    

  

- makeOwned(): 类同makeCopy()，差别的地方在于会转移外界对象的控制权到UserObject，也是生命周期安全的实现。
    
      
    
      
    

- #### **泛型构造函数**
    

  

```
template <typename T>
```

  

对类型T的构造实现，注意内部调用的是前面介绍的makeCopy()静态方法，所以通过这种方式构造的UserObject是生命周期安全的。

  

- #### **到原始C++类型的转换**
    

  

```
template <typename T>
```

  

注意转换失败会直接抛出C++异常。

  

- #### **Property相关的便利接口**
    

  

```
Value get(IdRef property) const;
```

  

对Property进行存取操作的快捷接口。

  

- #### **其他**
    

  

```
void* pointer() const;
```

  

比较常用的两个特殊接口，其中：

  

- pointer(): 用于获取UserObject内部存储对象的raw pointer。
    

  

- **getClass(): 用于获取UserObject对应的MetaClass。
    

  

- nothing: UserObject的空值，判断一个UserObject是否为空可以直接与该静态变量比较。
    

  

  

### **（二）Value**

  

Value的实现也很简单，核心代码如下：

  

```
using Variant = std::variant<
```

  

其实就是通过std::variant<>，定义了一个包含：

  

- NoType
    

  

- bool
    

  

- int64_t
    

  

- double
    

  

- string
    

  

- EnumObject
    

  

- UserObject
    

  

- ArrayObject
    

  

- BuildInValueRef  
      
    

以上这些类型的一个和类型，然后利用std::visit()访问内部的std::variant来完成各种操作的一个实现，实现思路比较简单，这样对于反射支持的类型，我们都可以统一用Value来进行表达了。

  

常用的接口如下:

  

```
ValueKind kind() const;
```

  

  

**六、builder实现——合到一起**

  

builder的目的主要就是利用compiler time特性，通过类型推导的方式完成从原始类型到Meta信息的生成。

  

## **（一）function的实现**

  

function的实现请参考[[reflection function implement]]。

  

  

## **（二）property的实现**

  

property的实现请参考[[refection property implement]]。

  

  

### **（三）总结**

  

由于C++本身compiler time类型其实是相对完备的，给我们提供了一个通过compiler time特性来完成Meta信息生成的目的，Property的实现与function类同，利用Traits和一些其他设施，最后整个meta class的信息就被填充完成了，然后我们就能够在运行时对已经注册的类型进行使用了。

  

  

**七、runtime实现——便利的运行时接口**

  

### **（一）对象创建和删除的辅助方法**

  

```
template <typename... A>
```

  

通过这些方法我们可以完成对反射对象的快速创建和销毁操作。

  

  

### **（二）函数调用**

  

```
template <typename... A>
```

  

介绍Function对象的时候，我们没有发现上面的invoke方法，ponder原始的实现，是间接通过辅助的call()，callStatic()方法来完成的对应meta function的执行，这个地方应该是可以考虑直接在Function上提供对应实现，而不是目前的模式。

  

  

**八、Future——思考题**

  

反射之后呢? 更进一步:

  

```
// Define the interface of something that can be drawn
```

  

需要实现类rust traits这种编译期和运行期一致的多态，我们还需要做哪些事情？我们已经具备的特性？还需要的特性？

  

  

**九、小结**

  

其实系统的了解后会发现，随着C++本身的迭代，像反射这种轮子，开发难度变得越来越简单，对比C++98年代的luabind，cpp-framework中的反射实现代码已经很精简了，而且我们也能发现功能更强大，适应的场合更多了，代码复杂度也大幅下降了，从这个角度看，可以看到C++新特性的迭代，其实是在做减法的，可以让你有更简洁易懂的方式，去表达原来需要更复杂的实现去做的事情。从14/17这个节点去学习和了解Meta Programming，是一个不错的时间点。

  

**参考资料：**

1.[github ponder库] 

https://github.com/billyquith/ponder

  

- EOF -

推荐阅读  点击标题可跳转

1、[手撸一个智能指针](http://mp.weixin.qq.com/s?__biz=MzAxNDI5NzEzNg==&mid=2651170397&idx=1&sn=a82511b52f016766d4d73f86f67b053a&chksm=80647702b713fe14fc7bc435537ec386a147189e581d1bec3e60e3d97a7d13123343292b1644&scene=21#wechat_redirect)

2、[新简化！typename 在 C++20 不再必要](http://mp.weixin.qq.com/s?__biz=MzAxNDI5NzEzNg==&mid=2651170379&idx=1&sn=b8705b47fc9c3cc6d34551854c733cfc&chksm=80647714b713fe0214e441d4989c3b78eb3fd8f9f1018ade851c3b4f27eb88b8c033018c2b4b&scene=21#wechat_redirect)

3、[理解可变参数模板](http://mp.weixin.qq.com/s?__biz=MzAxNDI5NzEzNg==&mid=2651170377&idx=1&sn=c8824e4a39ec44314b7c203569fd8a8b&chksm=80647716b713fe00c6b95b71cb4da10ebf3f28786df00a51d6655b177567febb95da7a603d91&scene=21#wechat_redirect)

  

**关注『CPP开发者』**  

看精选C/C++技术文章 

![](http://mmbiz.qpic.cn/mmbiz_png/pldYwMfYJpia3uWic6GbPCC1LgjBWzkBVqYrMfbfT6o9uMDnlLELGNgYDP496LvDfiaAiaOt0cZBlBWw4icAs6OHg8Q/300?wx_fmt=png&wxfrom=19)

**CPP开发者**

我们在 Github 维护着 9000+ star 的C语言/C++开发资源。日常分享 C语言 和 C++ 开发相关技术文章，每篇文章都经过精心筛选，一篇文章讲透一个知识点，让读者读有所获～

24篇原创内容

公众号

点赞和在看就是最大的支持❤️  

Reads 2763

​

Comment

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/pldYwMfYJpia3uWic6GbPCC1LgjBWzkBVqYrMfbfT6o9uMDnlLELGNgYDP496LvDfiaAiaOt0cZBlBWw4icAs6OHg8Q/300?wx_fmt=png&wxfrom=18)

CPP开发者

614

Comment

Comment

**Comment**

暂无留言