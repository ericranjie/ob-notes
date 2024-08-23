
原创 孟杰 腾讯云开发者

 _2022年02月08日 17:57_

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/VY8SELNGe977Xa5zfy5iaV3agpS11Cqm4psjPOibic6BZSicnBFh6uWzCFp3uqN5R114Fq85DmuCzdL3eESlQ37bFA/640?wx_fmt=jpeg&wxfrom=13&tp=wxpic)

  

导语 | 云加社区祝大家新年快乐！新春假期结束的第一篇干货，为大家带来的是从C++转向Rust主题的内容。在日常的开发过程中，长期使用C++，在使用Rust的过程中可能会碰到一些问题。本文是From C++ To Rust的第二篇，在这一篇里，主要介绍错误处理和生命周期两个主题。

  

此前，我介绍了其中思维方式的转变（mind shift）：_《_[_**详细解答！从C++转向Rust需要注意哪些问题？**_](http://mp.weixin.qq.com/s?__biz=MzI2NDU4OTExOQ==&mid=2247526794&idx=1&sn=8c42b2e179ebd4bb00c330ecce54adee&chksm=eaa873dadddffacce6358d1539424b8cc948031eba4cf4de3f98fab08017adbc1e177211b723&scene=21#wechat_redirect)_》_

  

**一、错误处理**

  

### **（一）C++**

  

任何生产级别的软件开发中，错误处理都需要被妥善考虑。C++通常会有两种错误处理的风格：

  

- 从C继承下来的**返回值**风格。所有函数都返回整型，用错误码来表示各种错误情况。
    

  

- C++的异常，在出错的位置抛出**异常**，然后在错误处理的位置捕捉异常。
    

  

这两种方案各有优劣，这里简单地说明一下。

  

返回值风格的优点是清晰，错误发生的位置和处理方法都写得很直白；缺点即是拖沓，错误代码与业务代码交错在一起，使得主要逻辑不突出。同时占用了返回值位置，影响逻辑的表达。另外，没有强制错误检查，可能会遗漏错误检查而导致代码缺陷。如下：

  

```
if (OK != foo()) {
```

  

**异常**恰恰相反，错误有独立的处理流，通常不与业务逻辑相交，使得业务逻辑看起 来很清晰；但是由于异常的隐性，使得任何位置都可能抛出异常，函数的退出点也变得隐晦，带来**异常安全**问题，增加了代码编写的心智负担。如下：

  

```
void foo() {
```

  

如果上面代码中的bar抛出异常，程序的执行流程将从bar函数跳出进入**异常处理流程**，因此后面delete语句不能得到执行，导致thing泄漏。

  

解决此问题的方法是使用**智能指针**，它们使用了RAII机制确保了函数在各种情况下均能妥善地释放动态分配的对象。

  

  

### **（二）Rust**

  

- #### **Result<T，E>**
    

  

Rust没有提供异常机制，与使用Option来解决可选的情况类似，它使用了Result来提供此功能。Result的定义如下：

  

```
pub enum Result<T, E> {
```

  

可以看到，Result的定义几乎与Option一样。只是在异常的情况返回时多带一个错误类型。举一个具体的例子：

  

```
#[derive(Debug)]
```

  

上面let id=fetch_id()?；一句，使用了?操作符，实际相当于执行如下语句：

  

```
let id = match fetch_id() {
```

  

相当于，如果被调函数(fetch_id)正常返回则unwrap其值；反之，则将被调函数的错误向上返回。

  

相对于C/C++，Rust在此处，实际上在尝试做到某种**平衡**：

  

- 没有**异常**，没有引入新的执行模型。函数的执行流程可以采用简单的**返回值**方式分析，便于理解。
    

  

- ?操作符的引入，使用语法糖一方面减少错误处理代码，代码更清爽；另一方面也**显式地**注明了所有**返回点**。
    

  

- Result中携带的返回值T必须unwrap之后才能使用，这在类型系统上保证了错误必须被处理，**不能沉默地忽略**。
    

  

- 错误处理是**强类型**的。通过Result中的E类型参数向上返回错误时，必须要求E类型不变。这里产生了一些Rust错误处理的独特要求，后面再展开。
    

  

- 由于Safe Rust不能直接操作裸指针，所以不论函数从什么位置返回，均确保通过**RAII**机制释放了指针。
    
      
    

- #### **panic!**
    

####   

#### 在Rust中，错误被划成了两类：**可恢复的(recoverable)** 和**不可恢复的(unrecoverable)**。所谓**可恢复**通常指的是可以上报给用户，修复之后，然后**重试**一下的错误，比如：文件未找到，配置错误等。而**不可恢复**一般是由于代码Bug导致的，程序已经进入未定义状态，继续执行可能产生**未定义行为**，比如：数组越界访问。

  

对于可恢复的错误，使用Result<T，E>返回错误，交由调用方决定该如何处理。而对于不可恢复的错误，使用panic!宏直接中止程序的执行。

  

  

#### **（三）Rust错误处理惯例**

  

如之前所说，Rust的错误处理是**强类型**的。因此，不能像C++的异常一样，错误可以穿透多层调用栈；相反，错误必须被逐层处理、翻译，不能一抛到底。这个工作其实是较为繁琐的。举个例子：

  

```
#[derive(Debug)]
```

  

这段代码不能编译通过，因为std::fs::read_to_string和String::parse的 返回值虽然都叫Result，但却不是相同的类型(因为E被定义为库局部的错误了)。因此，这里都不能直接使用?操作符。而是需要进行错误类型的翻译，转成我们自己定义的MyError：

  

```
pub fn fetch_id() -> Result<u64> {
```

  

显然，写法1过入繁冗，实在称不上优雅。而写法2直接使用标准库函数map_err来完成错误类型的映射，会干净很多。但是如果映射的代码比较复杂，或者同样的处理会多次重复，就会希望将错误映射集的代码中起来。因此，社区中也提供了库来简化这部分处理，如：thiserror，anyhow。这两个库分别对应了**库级别**与**应用级别**的错误处理。

  

所谓**库级别**指的是编写为可被其它库或者应用复用的代码。因此，并不清楚错误最终会被如何处理，所以最终会在库级别统一Error的类型，并最终将底层转译到该错误类型。如上例中的MyError。上例在使用thiserror改写之后：

  

```
#[derive(thiserror::Error, Debug)]
```

  

可以看使用thiserror后，代码清爽了很多。thiserror会为MyError自动实现底层Error的From trait。所以在fetch_id中就可以直接用?操作符将底层 Error映射到MyError。

  

而**应用级别**的Error不需要进一步上传给调用者，只需要有一个Result类型 可以集中处理所有的底层Error即可。因此，此时不需要自定义MyError， 使用anyhow改写之后如下：

  

```
use anyhow::{Context, Result};
```

  

anyhow为泛型(generic)的Result<T，E>实现了Context trait。而Context提供了context函数，将原来的Result<T，E>转成了Result<T，anyhow::Error>，最终在应用级别将错误类型统一到anyhow::Error上。

  

限于篇幅，这里不再对这两个库做更深入说明，更细致的说明可以参考以下详细文档： 

  

- Rust:Structuring and handling errors in 2020（https://nick.groenen.me/posts/rust-error-handling/）
    
- thiserror（https://docs.rs/thiserror）
    
- anyhow（https://docs.rs/anyhow）
    

  

  

**二、生命周期**

  

终于到这个主题了！显然**生命周期**是Rust最独特的特性，没有之一。虽然在各种语言都会定义对象的生命周期，但将其在语言中静态表达出来的只有Rust。因此，虽然早有接触，但是在Rust碰到还是会觉得陌生，甚至晦涩。在这里笔者尝试记录下自己学习这个概念的关键点，想到什么说什么，不会是一个系统的教程，只是记录C++的熟悉者容易忽略的一些点。

  

### **（一）谈谈生命周期**

  

简单地说，生命周期就是一个对象的存续时间。对于支持引用的语言来说， 引用目标在**使用时必须存在**是程序正确运行的基础；同时因为计算机内**有限的资源**，所以在对象使用完毕后，必须尽早释放。生命周期可以**手动管理**，但是因为程序的复杂性，手动管理是一件成本很高并且易错的工作。所以也就诞生了各种**自动管理**生命周期的算法，当前典型的算法有两类，**引用计数(RC)**和**垃圾收集(GC)**。

  

而Rust实际是探索了第三种自动管理的方案：**编译期的静态检查-BorrowChecker**，它通过分析变量的定义域(Scope)与移动(Move)规则，来保证通过引用使用目标对象的安全性。(注：在NLL BorrowChecker引入后，定义域不再是严格的代码块)。

  

初次接触Rust，最奇怪的就是生命周期的记法了：'a。很陌生，很费解。为什么需要它？解决什么问题？说一下我的理解：

  

- 'a 是一种**标记**，BorrowChecker通过**比较生命周期**来保证引用的安全性；
    

  

- 一般地，**所有引用都含有生命周期标记**。只是因为避免语言过于繁冗，Rust允许开发在一些情况下省略该标记(Lifetime Elision)；
    

  

- 因为BorrowChecker工作在**编译期**，所以生命周期标记**合并在****泛型系统****中**，具体实现为**泛型参数**中的一项。
    
      
    
      
    

### **（二）生命周期省略-Lifetime Elision**

  

Rust为了代码的清爽，允许开发在很多情况下都可以不使用生命周期标记。

  
这使得生命周期标记的出现场景比较微妙，比如：

  

```
fn print(s: &str);
```

  

实际上应该理解为：

  

```
fn print<'a>(s: &'a str);
```

  

该函数接受一个字符串的引用，显然，这个引用目标的生命周期一定可以覆盖 print的执行期，print并不需要对引用的生命周期做更特别的静态检查。因此，Rust允许省略这个生命周期标记。

  

具体的省略规则可以参考文档：Lifetime Elision

（https://doc.rust-lang.org/nomicon/lifetime-elision.html）

  

说一下我的理解：

  

首先定义了**输入引用**和**输出引用**，文档中为了严谨，描述得比较长。简单地说，**除了函数返回的引用外，其它都是输入引用**。然后依据以下规则省略引用：

  

- 规则1：每个**输入引用**都给予**独****立**的生命周期；
    

  

- 规则2：如果**只有一个**输入引用，那么该引用的生命周期给到**全部省略的输出引用**；
    

  

- 规则3：如果是**多个**输入引用，并且其中有&self或者&mut self，那么self的生命周期给到**全部省略的输出引用**；
    

  

- 其它情况都是错误的，必须**显式地**进行生命周期标注。
    

  

虽然Rust允许开发省略标注，但是需要注意的是：**Rust根据上面规则自动恢复的标注，有可能并不是你想要表达的目的**。如文档中的例子：

  

```
fn substr(s: &str, until: usize) -> &str;
```

  

应用规则2，取消省略之后：

  

```
fn substr<'a>(s: &'a str, until: usize) -> &'a str;
```

  

在取子串的情形中，返回的子串生命周期与输入参数一致，因此，默认恢复的标注是合理的。但是如果是下面函数：

  

```
fn country_abbr(c: &str) -> &str {
```

  

应用规则2，取消省略后的签名是：

  

```
fn country_abbr<'a>(c: &'a str) -> &'a str;
```

  

可以知道，返回的“CN”，“US”，“Unknown”的生命周期是'static，由于'static的长度比所有其它生命周期都长，因此，将其以&'a str的类型返回不会有编译错误。但是这个结果会缩小country_abbr的使用范围，这可能并不是我们想要的结果。如下代码会无法编译通过：

  

```
fn check_lifetime(abbr: &'static str) {}
```

  

所以，在了解了生命周期的省略规则后，country_abbr的签名应该写作：

  

```
fn country_abbr(c: &str) -> &'static str;
```

  

另一个需要注意的地方是：**对于接受多个引用参数的函数，每个引用的生命周期都是独立的**。如下：

  

```
fn foo(bar: &str, baz: &str);
```

  

应该应用规则1和规则2展开为：

  

```
fn foo<'a, 'b>(bar: &'a str, baz: &'b str);
```

  

而不是:

  

```
fn foo<'a>(bar: &'a str, baz: &'a str);
```

  

因为后期实际上要求了两个参数的生命周期必须是**一样的**， 因此施加了比前者更强的约束。

  

  

### **（三）子类化(Subtyping)与变型(Variance)**

  

写下这个标题时，我心里也是没有什么底的：因为相对来说这是一些抽象及陌生的概念，使用简单且易于理解的语言将其说明白，对我来说是也很大的挑战。下面的说明都是使用自己理解的语言来表达的，**不追求特别严谨精确**，但希望易于理解。

  

- #### **子类化Subtyping**
    

  

为了加快思考，人脑会将一些常用的推导变成直觉，不自觉地忽略底层的逻辑细节，子类化(Subtyping)就是其中一个例子。因为在C++中，**子类关系**通常在**继承关系**中体现，所以从C++转过来的话，很容易下意识地认为子类就是继承。而事实上，**子类关系是比继承关系更一般的(generic)关系**。或者换句话说，继承关系是子类关系的一种实现。

  

所以，在Rust中不能简单地将子类化理解为继承，需要重新整理一下。笔者从几个点来理解：

  

- 子类关系符合里氏替换原则。即是说，如果S是T的子类，那么类型为T的**形参**可以填入类型为S的**实参**。说人话：在需要使用某个类型的场合，也可以使用**该类型的子类来代替**。白话：**子类比超类更有用**。
    

  

- 在逻辑学中，**内涵**指概念所拥有的**属性**；而**外延**指的具备概念属性的**事物**。对应到类型系统，**内涵**指是某个类型的**属性或方法**；而**外延**指的是该类型的**所有****实例**。所以，**子类**比超类有**更多内涵**，**更少外延**；而**超类反之**。
    

  

说了这么多，终于可以回到生命周期主题了。笔者在学习生命周期的过程中， 碰到第一个**反直觉**的结论是：**'static是****所有其它生命周期的子类**，可以写作'static<: 'a ('a是任意任命周期)。你看：明明'static是最长最大最多的生命周期，为什么是子类？是小的那端？理解起来很不自然。

  

后来一句话解了我的疑惑：**生命周期的长度体现的是内涵**。这句话想想还有点哲学意味。

  

因此，<: 描述的是**外延**的大小，所以，任何大于'a的生命周期都是'a的实例，而'static的实例只有一个，就是'static本身。显然，'a的外延大于'static，所以'static是子类。从**有用性**的角度理解，'static可以在任何需要生命周期的地方使用，是最有用的，所以根据前面说到的，子类比超类更有用，'static显然是子类。

  

- #### **变型Variance**
    

  

在介绍变型之前，需要先引入另外一个概念**类型构造子****(Type Constructor)**。首先这个概念要与C++中的**构造函数(Constructor)**区别开来：构造函数是用于创建类型的**新实例**；而类型构造子则是用于创建**新类型**：

  

- 可以是和类型或者积类型的构造。在Rust中可以认为是enum或者struct的定义式；
    

  

- 可以是**泛型类型的实例化**。如：Vec<u8>。
    

  

在考虑**变型**时，主要是第二种情形，即：泛型类型的实例化。我们可以将**泛型类型**理解为**类型的函数**，因为其接收类型参数，返回新的类型。这样，我们就可以引出**变型**的三种情况了：

  

假设有类型构造子：F<T>, 并且有两个具体的类型：Super和Sub满足Sub<:Super，这两个具体类型通过F<T>可以分别构建新类型F<Sub>和F<Super>

  

- **协变-covariance**: 如果新类型和类型参数的**关系一致**，即满足 F<Sub> <: F<Super>，则称之为F<T>对T协变。
    

  

- **逆变-contravariance**: 如果新类型和类型参数的**关系相反**，即满足 F<Super> <: F<Sub>，则称之为F<T>对T逆变。
    

  

- **不变-invariance**: 如果新类型和类型参数的**关系无关**，即不满足任何约束，则称之为F<T>对T不变。
    

  

终于介绍完了两个抽象概念，可以回来谈Rust了。

  

Rust当前**没有**定义类型间的子类关系。trait虽然可以继承，但并不是符合定义的子类关系（无法将&dyn Derive直接传给&dyn Base）。因此，在当前版本的Rust中，子类关系只在生命周期中存在。

  

在Rust的文档中，有一个表描述了各种类型的变型关系，这里针对不太容易理解的两种情况进一步说明：

  

- ##### **&'a mut T为什么对T是不变(invariant)?**
    

  

根据《锈灵书》在介绍变型的相关章节中提供的例子：

  

```
fn evil_feeder(pet: &mut Animal) {
```

  

&mut T对T的不变性(invariant) 是为了阻止通过修改超类的引用&mut Animal将Dog的实例复制到Cat的内存上(*pet=spike)。但是这个例子还是有一些不清晰的地方：

  

如前面所述，类型间的子类关系在Rust并未定义，所以这里上面提到“Dog is a subtype of Animal”并不准确。

  

另外，由于trait object是一个动态尺寸类型(dynamic sized type)，所以必须Dog，Cat必须位于某种指针之后，因此，let spike: Dog=...不是合法的代码。

  

从逻辑上说，拿到某个类的指针，并不能用子类(当然也不能用超类)实例去覆盖该类的实例，因此，&mut T应该是不变的(invariant)。笔者推测是否也是Rust为了保留以后类型子类化的能力。

  

- ##### **fn(T)-> ()为什么对T是逆变(contravariant)?**
    

  

这是文档中唯一的逆变的例子，所以多说明一下。fn(T) -> ()是函数类型，用该类型描述某个作用场景(即，参数位置)时，其实是回调的场景。因此，回调函数的参数类型T，实际是对调用方的要求。这个要求越少(即，更加泛化，约束少，更偏向超类)， 回调函数反而使用场景更大(即，更有用)。前面已经说到，更有用的是子类。

  

举个例子：

  

```
struct A;
```

  

这里cb1的参数类型&'statc A是cb0的参数类型&'a A的子类，但是cb1却不能被foo接受。

  

  

### **（四）关于生命周期容易产生的误解**

  

在Rust中，生命周期是全新的概念，因此也容易理解错误，对于常见的情形，Common Rust Lifetime Misconceptions一文介绍得非常清楚。

文档：（https://github.com/pretzelhammer/rust-blog/blob/master/posts/common-rust-lifetime-misconceptions.md#3-a-t-and-t-a-are-the-same-thing）

  

如果刚开始学习Rust特别需要注意：

  

- T既是&T也是&mut T的超类；
    

  

- &T和&mut T是不相交的集合。
    

  

对于熟悉C++重载规则的开发来说，这两点是需要注意的。在Rust中，因为T包含&T，所以，不能同时为T，&T实现一个trait. 如下：

  

```
trait Trait {}
```

  

&mut T和T也同理。因此，要为引用实现trait应该写作：

  

```
trait Trait {}
```

  

另外，即是要区分好，T: 'static和&'static T，主要规则如下：

  

- T: 'static 应该读做：类型T被生命周期'static定界；
    

  

- 如果T: 'static, 那么，T可以是生命周期为'static的借用类型，或者是**拥有所有权的类型**。
    

  

因为T: 'static包括**拥有所有权类型**，所以T：

  

- 可以在运行时动态分配；
    

  

- 不必在程序运行的整个生命周期有效；
    

  

- 可以安全地被修改；
    

  

- 可以在运行时动态释放；
    

  

- 可以具备不同的存续期。
    

  

更加一般地，T: 'a和&'a T的规则如下：

  

- T: 'a比 &'a T更具弹性且通用；
    

  

- T: 'a可以接受**拥有所有权的类型**，包含**引用的拥有所有权类型**或者**引用**；
    

  

- &'a T只接受引用；
    

  

- 如果T: 'static那么T: 'a因为对于所有'a有'static >='a。
    
      
    
      
    

**三、后记**

  

这两个主题比我想像花了更多的篇幅，所以这一篇先到这里吧。后面计划继续聊聊**可修改性Mutablility**和**异步Async**。

  

  

 **作者简介**

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**孟杰**

腾讯后台开发工程师

腾讯后台开发工程师，毕业于中南大学。目前负责腾讯安全流量分析平台的后台开发工作。开发经验丰富，对程序语言，类型系统，编译等方向很感兴趣。

  

  

 **推荐阅读**

  

[技术人专属年味尽在这里！云加社区祝您虎年大吉](http://mp.weixin.qq.com/s?__biz=MzI2NDU4OTExOQ==&mid=2247533493&idx=1&sn=fead6276895838decded2b98e1893567&chksm=eaa859e5dddfd0f3f25b25ec5fc7449c6b766791dcc50841f8371f53b589d98bffa45abd856a&scene=21#wechat_redirect)  

[关于Go并发编程，你不得不知的“左膀右臂”——并发与通道！](http://mp.weixin.qq.com/s?__biz=MzI2NDU4OTExOQ==&mid=2247533017&idx=1&sn=1a23a99c282e2a2a947c1dc53b1de1a5&chksm=eaa85b89dddfd29f0bb6b5753aeff8f9d95dc2e6c0b01699fdd955ce54a431dc435d99eaefab&scene=21#wechat_redirect)  

[一文入魂：妈妈再也不用担心我不懂C++移动语义了！](http://mp.weixin.qq.com/s?__biz=MzI2NDU4OTExOQ==&mid=2247532897&idx=1&sn=02ba59278b7b1cb1602cc9b08a8fc238&chksm=eaa85b31dddfd2271f74dd3f2449eebc8d8af1e76bec640aa29fa88ad86e4149806564b7df97&scene=21#wechat_redirect)  

[图解史上最晦涩的分布式共识算法——Paxos！](http://mp.weixin.qq.com/s?__biz=MzI2NDU4OTExOQ==&mid=2247532797&idx=1&sn=a50a31a6f234d30f86fbb4ef203128e3&chksm=eaa85aaddddfd3bb6bf481f209490ecbc9883a63d31088b43e5a867502f070f68671a47e3b64&scene=21#wechat_redirect)  

  

![](http://mmbiz.qpic.cn/mmbiz_png/VY8SELNGe97ibOIthe2pvwt1H0HqX0HVJVFK9WPNQKNsibXynR5yT5S7b45uIpzN7xeZdeJIfOibPjOflZ35rKZyw/300?wx_fmt=png&wxfrom=19)

**腾讯云开发者**

腾讯云官方社区公众号，汇聚技术开发者群体，分享技术干货，打造技术影响力交流社区。

867篇原创内容

公众号

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)  

阅读 1930

​

写留言

**留言 3**

- quq🍪
    
    2022年2月8日
    
    赞5
    
    小编好像真的想要教会我![[苦涩]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![[叹气]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
    腾讯云开发者
    
    作者2022年2月8日
    
    赞
    
    📖学起来！![[加油]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)新的一年新的希望![[转圈]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
    quq🍪
    
    2022年2月8日
    
    赞1
    
    先去mind shift了！
    

已无更多数据

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/VY8SELNGe97ibOIthe2pvwt1H0HqX0HVJVFK9WPNQKNsibXynR5yT5S7b45uIpzN7xeZdeJIfOibPjOflZ35rKZyw/300?wx_fmt=png&wxfrom=18)

腾讯云开发者

20110

3

写留言

**留言 3**

- quq🍪
    
    2022年2月8日
    
    赞5
    
    小编好像真的想要教会我![[苦涩]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![[叹气]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
    腾讯云开发者
    
    作者2022年2月8日
    
    赞
    
    📖学起来！![[加油]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)新的一年新的希望![[转圈]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
    quq🍪
    
    2022年2月8日
    
    赞1
    
    先去mind shift了！
    

已无更多数据