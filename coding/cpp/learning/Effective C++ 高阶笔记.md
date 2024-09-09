
程序员贺同学

 _2021年11月30日 18:00_

以下文章来源于畅游码海 ，作者CallMeEngineer

[

![](http://wx.qlogo.cn/mmhead/Q3auHgzwzM64xwHaVbpExZZAQkfwibGD4ExzQibjEianicwUMOPOo3qc6Q/0)

**畅游码海**.

记录学习过程中的知识，从零开始构建学习框架，大家共同学习、共同进步！

](https://mp.weixin.qq.com/s?__biz=MzIzNzg5MDg0OA==&mid=2247489803&idx=2&sn=87012df52ebcd7067a542c8676b3c10c&chksm=e8c0e544dfb76c52f7b38ec26076768922515bfc0915b71a3d18b51c0a16ba7640918867a0ed&mpshare=1&scene=24&srcid=1201XZI4O2RNXTkQM4xpmHnW&sharer_sharetime=1638291363696&sharer_shareid=5fb9813bfe9ffc983435bfc8d8c5e9ca&key=daf9bdc5abc4e8d0f0dc7289d73481b1eeaa18706aa9f329fd4717f7c7cc3e9043bf1187aeacd0942b5bf72b62826706f9da0d00f5a3b7498f98fdffd43c4a2f15bda1b40073160aa6ef86e2930c14d22eae2f9b53af5afaaf638fad8df854aa7a6f99d03839a043c7eacfcd85f0df042014c57995d1b0b22606bd350398bfc0&ascene=0&uin=MTEwNTU1MjgwMw%3D%3D&devicetype=Windows+11+x64&version=63090b19&lang=zh_CN&countrycode=CN&exportkey=n_ChQIAhIQyWeiY1QBv32WlC1ipAHjsBLmAQIE97dBBAEAAAAAANd3LCBdpKoAAAAOpnltbLcz9gKNyK89dVj0CClYMrfo2IAW12woIcqgRLOnRNuXuKdwJCtbqSij6Up21%2FhNHqr9Q%2F%2Bv9Q6A0JHApWJcpzx0cD7BNeGqVP9KgWbTAp5gm4ONKWFQCIoYgI1QTVx2ZHs7dF0kChpKYIHqpcQvXW0x%2FTBS7r1b0DnpARA07FS%2BIWGJ4dJPqjSIq7yU3PplMv%2FY8WmT7paSuHFEeTenzCWADgQHEmm%2Bw6QQwvioVD9R7RqnwmNkEfuAlOGKP3swl6Av32QlGRQAKlPN&acctmode=0&pass_ticket=DtCoxrbLL2IYlHdRVfcXJyDACEf4T0ystHecsfuVSTzd50i%2F%2FUotZituUa%2BV%2B255&wx_header=1&fasttmpl_type=0&fasttmpl_fullversion=7350504-zh_CN-zip&fasttmpl_flag=1#)

![图片](https://mmbiz.qpic.cn/mmbiz_png/QMtSGqOCILmSSG7r2JlmvT8RHDxcjIjNfOUpiaUD3Xqrqibrmcnx4sfFxbtIp4NxSibyib2Zjp0QbpoibkVu0OaTrgA/640?wx_fmt=png&wxfrom=13&tp=wxpic)

公众号：畅游码海   这里有更多高质量原创文章和大量免费学习资料！

# Part1一、让自己习惯C++

### 条款01：视C++为**一个语言联邦**

C++并不是一个带有一组守则的一体语言：他是从四个次语言( C、Object-Oriented C++、Template、STL )  组成的联邦政府，每个次语言都有自己的规约。记住这四个次于语言你就会发现C++容易了解得多。

### 条款02:尽量以const,enum,inline替换 #define

 #define ASPECT_RATIO 1.653

以上句为例，是通过预处理器处理而不是编译器处理，有可能ASPECT_RATIO 没进入记号表内，于是如果出现了编译错误，那么编译器会提示错误信息是 1.653  而不是 ASPECT_RATIO ，你会感到非常困惑。

解决方法是用常量替换宏

const double AspectRatio = 1.653

这样编译器就可以看到ASPECT_RATIO ，而且使用常量会使代码量较小，因为预处理器只会盲目的替换而出现多份 1.653

string对象通常比char* 更好一点

对于class的专属常量，为了限制作用域在class内,并且防止产生多个实体，最好使用static

    1. 如果你的编译器支持在类内对const static 整数类型声明时获初值，则使用  
    2. 如果不支持，则在类内定义，在对应的实现文件中赋值  

如果你需要在编译器就使用一个class常量值，则应最好改用枚举类型enum，且枚举不能用来取地址，不会为它分配额外的存储空间对于形似函数的宏，最好改用inline的模板函数

### 条款 03:尽可能使用const

const出现在星号左边目标是指物是常量,出现在星号右边表示指针本身是常量,如果出现在两边，则指针和物都是常量void f1(const Widget* pw)和void f2(Widget const* pw)两种写法意义相同,都表示被指物是常量 对于STL迭代器来说，如果你希望迭代器所指的动科不可改动，你需要的是const_iterator 令函数返回一个常量值，往往可以降低因客户错误而造成的意外(例如把一个值赋值给一个返回值) 将const实施与成员函数的目的是为了明确该成员函数可作用于const对象:

```
    1. 他们使class接口比较容易理解    2. 他们使得可以操作const对象
```

const成员函数和no-const成员函数可重载，即可以同时出现，在传入不同的参数时候会调用不同的版本，但是有时我们需要这样，但是又不想代码重复，我们可以在no-const成员调用const成员函数来处理这个代码重复问题 例如：const_cast<char &>( static_cast<const TextBlock&>(*this)[position]);,经过这样里面 先安全转型使得调用的是const版本，外面再去const转型

### 条款 04:确定对象被使用前已先被初始化

对于内置类型要进行手工初始化构造函数最好使用成员初值列表，不要在构造函数中使用赋值操作来初始化，而且初值列表列出的成员变量次序应该和在class中声明的次序一样，因为声明次序就是C++保证的初始化次序 对于static对象，在跨编译单元之间的初始化次序是不能确定的，因为C++只保证在本文件内使用之前一定被初始化了

举例(使用如下方式可以解决这个问题即以loacl static对象替换non-local static对象)：

`class FileSystem{...};   FileSystem& tfs(){       static FileSystem fs;       return fs;   }   `

# Part2二、构造/析构/赋值运算

### 条款05:了解C++默默编写并调用了哪些函数

如果你不定义，编译器会自动帮你实习默认的构造函数，析构函数，拷贝赋值运算符和拷贝构造函数，但是如下几种情况不会替你生成默认的拷贝赋值运算符

```
    1. 类中含有**引用**的成员变量    2. 类中含有**const**的成员变量    3. 类的**基类**中的拷贝赋值运算符是**私有**成员函数
```

### 条款06：若不想使用编译器自动生成的函数，就应该明确拒绝

当我们不希望编译器帮我们生成相应的成员函数的时候，我们可以将其声明为private并且不予以实现

### 条款07：为多态基类声明virtual析构函数

以下情况应该为类声明一个virtual析构函数:

```
    1. 用来作为带有多态性质的基类的类    2. 一个类中带有任何virtual函数
```

如果类的设计目的不是作为基类使用，那么就不应该为它声明virtual析构函数

### 条款08：别让异常逃离析构函数

析构函数不要吐出异常，如果实在要抛出异常，那么最好使用std::abort();，放在catch中，把这个行为压下去 如果某个动作可能会抛出异常，那么最好把它放在普通函数中，而不是放在析构函数里面，让客户来执行这个函数并去处理

### 条款09：绝不再构造和析构函数中调用virtual函数

在构造和析构的时候，不要试图调用或在调用的函数中调用virtual函数，因为会调用父类版本导致出现一些未定义的错误

> 解决办法之一：

`class Transaction{       publci:        explicit Transaction(const std::string& logInfo);        void logTransaction(const std::string& logIngo) const;//把它变成这样的non-virtual函数        ...   };   Transaction::Transaction(const std::string& logInfo){       ...       logTransaction(logInfo);//这样调用   }   class BuyTransaction: public Transaction{        BuyTransaction( parameters ):Transaction(createLogString( parameters )){...}//将log信息传给基类的构造函数       private:        static std::string createLogString( parameters );//注意此函数为static函数   }      `

### 条款10：令operator= 返回一个reference to *this

为了实现连锁赋值如内置类型x= y = z =15由于=采用右结合律，所以等价于x = (y = (z = 15)),因此，为了使我们自定义类也实现，所以*重载=,+=,-=,*=使其返回refercence to this

### 条款11：在operator= 中处理“自我赋值”

在赋值的时候会出现对自我进行赋值的情况，这种情况下我们很容易写出不安全的代码

`Widget::operator=(const Widget& rhs){    delete pb; //把自己释放了    pb = new Bitmap(*rhs.pb);//这就不安全了    return *this;   }   `

因此有三种推荐的做法

1. ##### 先验证是不是相同的，是不是自我赋值
    

`Widget::operator=(const Widget& rhs){   if(this == &rhs) return *this;//验证是不是相同    delete pb;     pb = new Bitmap(*rhs.pb);    return *this;   }   `

2. ##### 在复制pb所指的东西之前别删除pb
    

`Widget::operator=(const Widget& rhs){    Bitmap* pOrig = pb;    pb = new Bitmap(*rhs.pb);//让pb指向*pb的一个副本       delete pOrig; //删除原先的pb    return *this;   }   `

3. ##### 使用交换数据的函数
    

`class Widget{   ...   void swap(Widget& rhs);//交换*this和rhs的数据   ...   };   Widget::operator=(const Widget& rhs){    Widget temp(rhs);//创建一个rhs副本    swap(temp);//交换*this和上面的副本    return *this;   }   `

### 条款12：复制对象时勿忘其每一个成分

为了确保复制的时候复制对象内的所有成员变量，我们应该在字类的构造和赋值函数中调用父类的构造和赋值函数来完成各自的任务 不要尝试在复制构造函数和赋值函数中相互调用，如果想消除重复代码，请建立一个新的成员函数，并且最好将其设为私有且命名为init

# Part3三、资源管理

### 条款13：以对象管理资源

为了防止资源泄露，我们应该在构造函数中获取资源，在析构函数中释放资源，这样可以有效的避免资源泄露 使用智能指针是一个好的办法，在C++11中auto_ptr已经被弃用，有三个常用的是unique_ptr,share_ptr和weak_ptr

### 条款14：在资源管理类中心copying行为

我们在管理RAII(构造函数中获得，析构函数中释放)观念的类时，应该对不同的情况，根据不同的目的进行处理

```
    1. 当我们处理不能同步拥有的资源的时候，可以才用**禁止复制**，如把copying操作声明为private    2. 当我们希望共同拥有资源的时候，可以采用**引用计数法**，例如使用shared_ptr    3. 当我们需要拷贝的时候，可以采用**深拷贝**    4. 或者某些时候我们可以采用**转移**底部资源拥有权的方式
```

### 条款15：在资源管理类中提供对原始资源的访问

有的api函数往往需要访问类的原始资源，所以每一个RAII类应该提供一个返回其管理的原始资源的方法 返回原始资源可以使用显示转换也可以使用隐式转换，但是往往显示转换更加安全一点，但是隐式转换更加方便

`class Font{    ...    FontHandle get() const {return f;} //显示转换    ...    operator FontHandle() const {return f;} //隐式转换函数    ....    private:     FontHandle f; //管理的原始资源   }   `

### 条款16：成对使用new和delete时要采用相同形式

不要对数组形式做typedef,因为这样会导致delete的时候调用的是delete ptr而不是delete [] ptr,对内置类型会出现未定义或有害的，对类的类型会导致无法调用剩余的析构函数，导致类中管理的资源无法释放，从而造成内存泄漏 在new 表达式中使用[ ] ,则在相应的delete 表达式中也使用 [ ]

### 条款17：以独立语句将newed对象置入智能指针

   诸如这样的语句processWidget (std::tr1::shared_ptr(new Widget),priority())

```
    1. 在先执行new Widget`语句和调用std::tr1::shared_ptr构造函数之间    2. 不能确定priority函数的执行顺序，可能在最前面，也可能在他们的中间
```

# Part4四、设计与声明

### 条款18：让接口容易被正确使用，不易被误用

我们接口应该替客户着想，考虑周全，避免它们犯错误。例如在向函数传递日期的时候，把日期参数做成类的形式，并且用static成员函数来返回固定的月份，避免用户参数写错 接口应该和内置接口保持一致，避免让客户感觉不舒服，这方面STL做的很好 tr1::shared_ptr支持定制型删除器，使用它可以防范跨DLL构建和删除的问题，可以用它来自动解除互斥锁

### 条款19：设计class犹如设计type

谨慎的设计一个类，应该遵守以下规范

```
    1. 合理的构建class的构造函数、析构函数和内存分配函数以及释放函数    2. 不能把初始化和赋值搞混了    3. 如果你的类需要被用来以值传递，复制构造函数应该设计一个通过值传递的版本    4. 你应该给你的成员变量加约束条件，保证他们是合法值，所以你的成员函数必须担负起错误检查工作    5. 如果你是派生类，那么你应该遵守基类的一些规范，如析构函数是否为virtural    6. 你是否允许你的class有转换函数，，是否允许隐式转换。如果你只允许explicit构造函数存在，就得写出专门负责执行转换的函数    7. 想清楚你的类应该有哪些函数和成员    8. 哪些应该设计为私有    9. 哪个应该是你的friend,以及将他们嵌套与另一个是否合理    10. 对效率，异常安全性以及资源运用提供了哪些保证    11. 如果你定义的不是一个新type,而是定义整个type家族，那么你应该定义一个类模板    12. 如果只是定义新的字类以便为已有的类添加机制，说不定单纯定义一个或多个non-member函数或模板更好
```

### 条款20：宁以pass-by-reference-to-const替换pass-by-value

尽量以pass-by-reference-to-const替换pass-by-value，因为前者通常比较高效，比如在含有类的传递时，避免了多次构造函数和多次析构函数的调用，大大的提高了效率 但是对于某些，比如内置类型，迭代器，函数调用等最好以值传递的形式

### 条款21：必须返回对象时，别妄想返回其reference

绝对不能返回指针或者一个引用指向一个临时变量，因为它存在栈中，一旦函数调用结束返回那么你得到的将是一个坏指针，也不能使用static变量来解决，你可以通过返回值 来解决

### 条款22：将成员变量声明为private

```
    1. 为了保证一致性    2. 可以细微的划分访问和控制以及约束    3. 内部更改后不影响使用
```

protected并不比public更具封装性

### 条款23：宁以non-member、non-friend、替换member函数

我们可以用non-member、non-friend函数来替换某些成员函数，可以增加类的封装性，包裹弹性和扩充性

### 条款24：若所有参数皆需要类型转换，请为此采用non-member函数

如果你需要为某个函数的所有参数（包括被this指针所指的那个隐喻参数）进行类型转换，那么这个函数必须是个non-member

### 条款25：考虑写出一个不抛出异常的swap函数

在你没有定义swap函数的情况下，编译器会为你调用通用的swap函数，但是有的时候那并不是高效的，因为默认情况它在置换如指针的时候把整个内存都置换 我们采取一种解决办法 1. 在类中提供一个 public swap成员函数，并且这个函数不能抛出异常2. 在类的命名空间中提供一个non-member swap函数，并令它调用类中的swap函数 3. 如果你正在编写一个类而不是模板类，为你的class特化std::swap函数，并令它调用你的swap函数 4. 请在类中声明 using std::swap,让其暴露，使得编译器自行选择更合适的版本

# Part5五、实现

### 条款26：尽可能延后变量定义式的出现时间

定义一个变量，那么你就得承受这个变量的构造和析构的成本时间，所以在定义一个变量的时候我们应该尽可能的延后定义时间，在使用前定义，这样避免我们定义了却没有使用它，造成浪费

### 条款27：尽量少做转型动作

旧式转型是C风格的转型，C++中提供四种新式转型：

1. const_cast 通常被用来将对象的常量性转除。它也是唯一有此能力的转型操作符
    
2. ##### dynamic_cast 主要用来执行“安全向下转型” ，也就是用来决定对某对象是否归属继承体系中的某个类型。它是唯一无法由旧式语法执行的动作，也是唯一可能耗费重大运行成本的转型动作
    
3. ##### reinterpret_cast 意图执行低级转型，实际动作（及结果）可能取决于编译器，这也就表示它不可移植。例如将一个pointer to int转型为一个int。这一类转型在低级代码以外很少见。
    
4. ##### static_cast 用来强迫隐式转换，例如将non-const对象转换为const对象，或将int转为double等等，它也可以用来执行上述多种转换的反向转换，例如将void* 指针转为 type 指针，将pointer-to-base 转为 pointer-ro-derived 。但它无法将 const 转为 non-const ——这个只有const_cast才能办到
    

  

  

  

```
旧式转型使用的时机是，当要调用一个explicit构造函数对一个对象传递给一个函数时，其他尽量用新式转型
```

        请记住以下：

```
    1. 如果可以的话，避免dynamic_cast转型，如果实在需要，则可以试着用别的无转型方案代替    2. 如果转型是必要的，那么应该把他隐藏于某个函数背后，客户随后可以调用该函数，而不是需要将转型放进自己的代码里    3. 宁可要新型转型，也不要使用旧式转型
```

### 条款28：避免返回handles指向对象内部成分

避免返回handle（包括引用，指针和迭代器）指向对象内部。这样可以增加封装性，也能把出现**空悬指针**的可能性降低

### 条款29：为“异常安全”而努力是值得的

异常安全函数提供以下三个保证之一：基本承诺：如果异常被抛出，程序内的任何事物仍然保持在有效状态下。没有任何对象或数据会因此而败坏，所有对象都处于一种内部前后一致的状态。然而程序的现实状态恐怕不可预料强烈保证：如果异常被抛出，程序状态不改变。调用这样的函数需要有这样的认知：如果函数成功，就是完全成功，如果函数失败，程序会恢复到“调用之前”的状态不抛掷保证：承诺绝不抛出异常，因为它们总是能够完成他们原先承诺的功能。作用于内置类型身上所有操作都提供nothrow保证，这是异常安全码中一个必不可少的关键基础材料

这三种保证是递增的关系，但是如果我们实在做不到，那么可以提供第一个基本承诺，我们在写的时候应该想如何让它具备异常安全性

```
    1. 首先以对象管理资源可以阻止资源泄漏    2. 在你能实现的情况下，尽量满足以上的最高等级
```

### 条款30：透彻了解inlining 的里里外外

inline 声明的两种方式：

```
    1. 隐喻的inline申请，即把定义写在class内部    2. 明确声明，即在定义式前加上关键字inline
```

将大多数inlining限制在小型、被频繁调用的函数身上。这可使日后调试和二进制升级更容易，也可使得潜在的代码膨胀问题最小化。不要只因为function templates出现在头文件，就将他们声明为inline

### 条款31：将文件间的编译依存关系降至最低

支持“编译依存性最小化”的思想是：相依于声明式，不要相依于定义式

```
    1. 头文件和实现相分离，头文件完全且仅有声明式    2. 使用创建接口类
```

# Part6六、继承与面向对象设计

### 条款32:确定你的public继承塑模出is-a关系

public继承意味着is-a的关系，即子类是父类的一种特殊化，适合基类的一定适合子类，每个派生类对象含有着父类对象的特点

### 条款33：避免遮掩继承而来的名称

在父类中的名称会被字类的名称覆盖，尤其是在public继承下，没有人希望这样的发生 为了避免被遮掩，可以使用using声明式或转交函数，交给子类

### 条款34：区分接口继承和接口实现

声明纯虚函数的目的就是为了让派生类只继承函数接口 声明虚函数的目的是让派生类继承该函数的接口和缺省实现 声明普通函数的目的就是让派生类强制接受自己的代码，不希望重新定义

### 条款35：考虑virtual函数以外的其他选择

### 条款36：绝不重新定义继承而来的non-virtual函数

任何情况下都不应该重新定义一个继承而来的non-virtual函数

### 条款37：绝不重新定义继承而来的缺省参数值

绝对不要重新定义一个继承而来的缺省参数值，因为缺省参数值都是静态绑定的，而virtual函数——你唯一应该覆写的东西是动态绑定

### 条款38：通过复合塑模has-a或“根据某物实现出”

区分public继承和复合 在应用领域，复合意味着一个中含有另一个，即has-a关系；在实现领域意味着根据某物实现出

### 条款39：明智而审慎地使用private继承

当需要复合时，尽可能的使用复合,必要时才使用private: 当protected成员或virtual函数牵扯进来的时候 当空间方面的利害关系，需要尺寸最小化

### 条款40：明智而审慎地使用多重继承

多重继承时候，如果其父类又继承同一个父类，所以解决的方式就是使用virtual继承，即其父类同时以virtual继承那个父类，但是相应的也会付出一些代价，例如时间更慢，需要重新定义父类的初始化等，因此设计时最好不要让这个父类有任何数据成员当单一继承和多重继承都可以，那么最好选择单一继承，多重继承也有正当的用途，可以实现同时public继承和private继承的组合

# Part7七、模板与泛型编程

### 条款41：了解隐式接口和编译期多态

显式接口：由函数的签名式（也就是函数名称、参数类型、返回类型）构成 隐式接口：不基于函数签名式，而是由有效表达式组成 面向对象和泛型编程都支持接口和多态，只不过一个以显式为主，一个以隐式为主 两种多态一个在运行期一个在编译期

### 条款42：了解typename的双重意义

声明模板参数的时候，class和typename是可以互换的，没什么不一样 但是标识嵌套从属类型名称的时候必须用typename 不得在基类列（继承的时候）或成员初值列（初始化列表）内以它作为基类修饰符

`templete<typename T>   class Derived:public Base<T>::Nested{ //基类列表中不可以加“typename”   public:       explicit Derived(int x): Base<T>::Nested(x){//mem.init.list中不允许“typename”           typename Base<T>::Nested temp; //这个是嵌套从属类型名称           ... //作为一个基类修饰符需要加上typename       }   }`   

### 条款43：学习处理模板化基类内的名称

模板化基类指的是当派生类的基类是一个模板

```
    1. 在基类函数调用之前加上  this->    2. 使用 using 声明式  ,告诉编译器，请它假设这个函数存在    3. 指出这个函数在基类中，使用基类::函数的形式写出来（不推荐这个，因为如果是virtual函数，则 会影响动态绑定）   但是当有模板全特化的时候，确实使用的没有这个函数，那么依然会报错
```

### 条款44：将与参数无关的代码抽离出来

模板生成多个类和多个函数，所以任何模板代码都不该和某个造成膨胀的模板参数产生相依关系 因非类型模板参数造成的代码膨胀，往往可以消除，做法是以函数参数或类成员变量替换模板参数 因类型模板参数造成的代码膨胀，往往可以降低，做法是让带有完全相同的二进制表述 的具体类型共享实现码

### 条款45：运用成员函数模板接受所有兼容类型

使用成员函数模板可以生成接收所有兼容类型的函数 如果你声明成员函数模板用来泛化拷贝构造函数和赋值操作，那么你还需要声明正常的拷贝构造函数和赋值操作

### 条款46：需要类型转换时请为模板定义非成员函数

当我们编写一个模板类，它提供的和这个模板祥光的函数支持所有参数的隐式类型转换，请将哪些函数定义为模板类的内部的friend函数

# Part8八、定制new和delete

### 条款49：了解new—handler的行为

当new分配失败的时候，它会先调用一个客户指定的错误处理函数（set_new_handler），一个所谓的new—handler 它是一个typedef定义出一个指针指向函数，该函数没有参数也不返回任何东西 set_new_handler的参数是个指针指向operator new 无法分配足够内存时该被调用的函数。其返回值也是个指针，指向set_new_handler 被调用前正在执行（马上就要被替换）的那个new—handler函数 一个良好设计的new—handler函数必须做以下事情:

```
    1. 让更多内存可被使用。此策略的一个做法是，程序一开始就分配一大块内存，而后当其第一次被调用，将它释还给程序使用    2. 安装另一个new—handler。可以设置让其调用另一个new—handler来替换自己，用来做不同的事情，其做法是调用set_new_handler    3. 卸载new—handler,也就是将null指针传给set_new_handler，这样new在分配不成功时抛出异常    4. 抛出bad_alloc的异常。    5. 不返回，调用abort或exit    6. C++并部支持类的专属new—handler，但其实也不需要。你可以令每个类提供自己的set_new_handler和operator new即可       set_new_handler允许客户指定一个函数，在内存分配无法获得满足时调用。       Nothrow new是一个颇为局限的工具，因为它只适用于内存分配：后继的构造函数调用还是可能抛出异常
```

### 条款50：了解new和delete的合理替换时机

替换operator new或operator delete的三个常见理由：用来检测运用上的错误 为了收集使用上的统计数据 为了增加分配和归还的速度 为了降低缺省内存管理器带来的空间额外开销，也就是实现内存池，可以节省空间为了弥补缺省分配器中的非最佳齐位 为了将相关对象成簇集中 为了获得非传统行为

了解何时可在“全局性的”或“class专属的”基础上合理替换缺省的new和delete

### 条款51：编写new和delete时需固守常规

operator new应该内含有一个无穷的循环，并在其中尝试分配内存，如果它无法满足内存需求，就该调用new-handler。它也应该有能力处理0 bytes申请，即将其按照1 byte分配。Class 专属版本应该处理“比正确大小更大的（错误）申请”，因为当有字类继承的时候，会出现传入的大小和父类大小不同，所以要进行判断形如if(size != sizeof(父类))operator delete应该在收到NULL指针的时候什么也不做，必要时交给全局的operator new来处理。

### 条款52：写了placement new也要写placement delete

当你写一个placement operator new ,请确定也写了对应的placement operator delete版本。如果没有这样做，可能回发生隐微而时断时续的内存泄露当你声明placement new 和placement delete,请确定不要无意识（非故意）地遮掩正常的全局版本，你如果想提供自定义形式，请内含所有正常形式的new和delete或利用继承机制及using声明式

# Part9九、杂项讨论

### 条款53：不要轻忽编译器的警告

不同的编译器有不同的警告标准，要严肃对待编译器发出的警告信息。努力在你的编译器的最高警告级别下争取“无任何警告”的荣誉 不要过度依赖编译器的报警能力，因为不同的编译器对待事情的态度并不相同。一旦移植到另一个编译器上，你原本依赖的警告信息有可能消失

### 条款54：让自己熟悉包括TR1在内的标准程序库

C++标准程序库的主要机能由STL、iostream、locales组成。并包含C99标准程序库。TR1添加了智能指针（例如 tr1::shared_ptr）、一般化函数指针（tr1::function）、hash-based容器、正则表达式以及另外10个组件的支持 TR1自身知识一份规范。为了获得TR1提供的好处，你需要一份实物。一个好的实物来源是Boost。

### 条款55：让自己熟悉Boost

Boost是一个社群，也是一个网站。致力于免费、源码开放、同僚复审的C++程序库开发。Boost在C++标准化过程中扮演具有影响力的角色 Boost提供许多TR1组件实现品，以及其他许多程序库

---

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**点击蓝字 · 关注我们**

  

  

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

  

**扫码关注我们**

  

  

  

更多高质量原创文章等你来看！

  

  

  

  

  

  

  

**END**

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

点个“在看”不失联

  

阅读 943

​

写留言

**留言 1**

- Mints
    
    2021年11月30日
    
    赞
    
    干货满满
    

已无更多数据

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/j5D4MI5U9vVJ8199WDILY25Z6WSrtRHxBy3pYSqlEalDuJjAEsXmuMic1nJG3CeQkia0ENcyTX1HibWEMwia2b9HsQ/300?wx_fmt=png&wxfrom=18)

程序员贺同学

1217

1

写留言

**留言 1**

- Mints
    
    2021年11月30日
    
    赞
    
    干货满满
    

已无更多数据