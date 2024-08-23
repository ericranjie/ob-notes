# 

CallMeEngineer C语言与CPP编程

 _2021年11月18日 08:46_

![Image](https://mmbiz.qpic.cn/mmbiz_jpg/0m4YX595Fom7dUgDLM0OJCxo425I9OKv3jfAgDFAcZH1cfSbV7rKJYKF8qgPJ7phxn55jLQhyRnicECicm1mOzIQ/640?wx_fmt=jpeg&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

# 一、关于对象  

- C 语言是**程序性**的，语言本身并没有支持数据和函数之间的**关联性**
    
- C++ 中可能采取**抽象数据类型**，或者是**多层次的类**结构完成
    
- C++ 的封装并没有增加多少成本，每一个成员函数虽然在class中声明，但是却不出现在每个对象中
    

- 每一个**非内联**的成员函数只会诞生**一个**函数实例
    
- 每个内联函数会在其**每一个**使用者身上产生一个函数实例
    

- C++ 在布局以及存储时间上主要的额外负担是由**virtual**引起的
    

- 虚函数机制用以支持一个有效率的“**执行期绑定**”
    
- 虚基类用来实现“**多次出现在继承关系中的基类，有一个单一而被共享的实例**”
    

- 还有一些多重继承下的额外负担，发生在**一个派生类和其第二或后继之基类的转换**之间
    

## 1.1 C++对象模式

- C++对象模型有以下几点
    

- 每个类中存放一个指针称为**vptr**，指向**虚函数表**
    
- 表中每个都指向一个虚函数
    
- **非静态数据成员**放在类对象**内**
    
- **静态数据成员**放在类对象**外**
    
- **静态和非静态成员函数**也放在类对象**外**
    
- 虚函数则不同
    

C++对象模型

## 1.2 关键词所带来的差异

- `int ( *pq ) ( ); //声明`
    
- 当语言无法区分那是一个声明还是一个表达式时，我们需要一个超越语言范围的规则，而该规则会将上述式子判断为一个“声明“
    
- struct和class可以相互替换，他们只是默认的**权限不一样**
    
- 如果一个程序员需要拥有C声明的那种struct布局，可以抽出来**单独**成为struct声明，并且和C++部分**组合**起来
    

## 1.3 对象的差异

- C++支持三种程序范式：**程序模型、抽象数据类型模型、面向对象模型**
    

- 面向对象模型在继承体系中 ，有时候编译期间无法确定指针或引用所指类型
    

- C++支持的多态类型：
    

1. 经由一组隐式的转化操作：如派生类指针转化为指向父类的指针
    
2. 经由虚函数机制
    
3. 经由**dynamic_cast 和 typeid运算符**
    

- **一个class所占的大小包括：**
    

- 其非静态成员所占的大小
    
- 由于内存对齐填补上的大小
    
- 加上支持虚函数而产生的大小
    

- 指针的类型，只能代表其让编译器如何解释其所指向的地址内容，和它本身类型无关，所以**转换其实是一种编译器指令，不改变所指向的地址，只影响怎么解释它给出的地址**
    
- 当一个**基类对象**被初始化为一个**子类对象**时，派生类就会被切割用来塞入较小的基类内存中，派生类不会留下任何东西，多态也不会再呈现。
    

# 二、构造函数语意学

## 2.1 默认构造函数的构造操作

- 以下四种情况下，会合成有用的构造函数：
    

- 类声明（或继承）一个虚函数
    
- 类派生自一个继承串链，其中有一个或更多的虚基类
    
- **带有默认构造函数的成员函数对象，不过这个合成操作只有在构造函数真正需要被调用时才发生，但只是调用其成员的默认构造函数，其他则不会初始化**
    
- **如果一个派生类的父类带有默认构造函数，那么子类如果没有定义构造函数，则会合成默认构造函数，如果有的话但是没有调用父类的，则编译器会插入一些代码调用父类的默认构造函数**
    
- **带有一个虚函数的类**
    
- **带有一个虚基类的类**
    

- C++新手常见的两个**误解**：
    

- **任何class如果没有定义默认构造函数，就会被合成出来一个**
    
- **编译器合成出来的默认构造函数会显式设定类中的每一个数据成员的额 默认值**
    

## 2.2 拷贝构造函数的构造操作

- 有三种情况会调用拷贝构造函数：
    

- **对一个对象做显式的初始化操作**
    
- **当对象被当作参数交给某个函数**
    
- **当函数传回一个类对象时**
    

- 如果类没有声明一个拷贝函数，就会有**隐式的声明和隐式的定义**出现，同**默认构造函数**一样在使用时才合成出来
    
- 什么情况下一个类不展现“浅拷贝语意”：
    

- **编译器会合成一个拷贝构造函数，安插一些代码用来设定虚基类指针和偏移的初值，对每个成员执行必要的深拷贝初始化操作，以及执行其他的内存相关工作**
    
- **编译器会显式的设定新类的虚函数表，而不是直接拷贝过来指向同一个**
    
- 这两个编译器都会合成拷贝构造函数并且**安插进那个成员和基类的拷贝构造函数**
    
- 当类内含有一个成员类而后者的类声明中有一个**拷贝构造函数**（例如内含有string成员变量）
    
- 当类继承自一个基类而**基类**中存在**拷贝构造函数**
    
- 当类声明了一个或多个**虚函数**
    
- 当类派生自一个继承串链，其中有一个或多个**虚基类**
    

## 2.3 程序转化语意学

- **在将一个类作为另一个类的初值情况下，语言允许编译器有大量的自由发挥的空间，用来提升效率，但是缺点是不能安全的规划拷贝构造函数的副作用，必须视其执行而定**
    
- 拷贝构造的应用，编译器会多多少的进行部分转换，尤其是**当一个函数以值传递的方式传回一个对象，而该对象有一个合成的构造函数**，此外编译器也会对拷贝构造的调用进行**调优**，以额外的第一参数取代NRV（Named Return Value）
    

## 2.4 成员们的初始化队伍

- 四种情况下你需要使用成员初始化列表
    

- **当初始化一个引用成员变量**
    
- **当初始化一个const 成员变量**
    
- **当调用一个基类的构造函数，而它拥有一组参数**
    
- **当调用一个类成员变量的构造函数，而它拥有一组参数**
    

`class Word{    String _name;    int _cnt;   public:    Word(){     _name = 0;     _cnt = 0;    }   /*使用成员列表初始化可以解决            Word() : _name(0)，_cnt(0){       }   */    }   `

> 上式不会报错，但是会有效率问题，因为**这样会先产生一个临时的string对象，然后将它初始化，之后以一个赋值运算符将临时对象指定给_name，再摧毁临时的对象**

- 成员初始化列表中的初始化顺序是按照**类中的成员变量声明的顺序，与成员初始化列表的排列顺序无关**
    

# 3、Data语意学

`class X{};   class Y : public virtual X {};   class Z : public virtual X {};   class A : public Y,public Z {};      sizeof(X) //1   sizeof(Y) //4   sizeof(Z) //4   sizeof(A) //8   `

- X为1是因为编译器的处理，在其中插入了1个char，为了**让其对象能在内存中有自己独立的地址**
    
- Y，Z**是因为虚基类表的指针**
    
- **A 中含有Y和Z所以是8**
    
- 每一个类对象大小的影响因素：
    

- **非静态成员变量的大小**
    
- **virtual特性**
    
- **内存对齐**
    

## 3.1 数据成员的绑定

- **如果类的内部有typedef,请把它放在类的起始处，因为防止先看到的是全局的和这个typedef相同的冲突，编译器会选择全局的，因为先看到全局的**
    

## 3.2 数据成员的布局

- **非静态成员变量的在内存中的顺序和其声明顺序是一致的**
    
- 但是**不一定是连续的**，因为中间可能有内存对齐的填补物
    
- virtual**机制的指针所放的位置和编译器有关**
    

## 3.3 成员变量的存取

- **静态变量**都被放在一个全局区，与类的大小无关，**正如对其取地址得到的是与类无关的数据类型**，如果两个类有相同的静态成员变量，编译器会暗自为其名称**编码**，使两个名称都不同
    
- **非静态成员变量**则是直接放在对象内，经由对象的**地址和在类中的偏移地址**取得，但是在继承体系下，情况就会不一样，因为编译器无法确定此时的指针指的具体是父类对象还是子类对象
    

## 3.4 继承下的数据成员

- **在下面给定的两个类中依次讨论不同情况：**
    

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

原本的数据模型

- ### 在单一继承没有虚函数的情况下布局图
    

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

单一继承且无虚函数  

- **这种情况下常见错误**：
    

- 可能会重复设计一些操作相同的**函数**，我们可以把某些函数写成**inline**,这样就可以在子类中**调用**父类的某些函数来**实现简化**
    
- **把数据放在同一个类中和继承起来的内存布局可能不同，因为每个类需要内存对齐**
    

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

- 叠在一起的内存布局
    

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

分层继承的布局

> **可见内存大了100%**

- **容易出现的不易发现的问题：**
    

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

继承下易犯错误  

- ### 当加上多态之后，对空间上增加的额外负担包括：
    

- **析构函数的调用顺序是反向的，从子类到父类**
    
- **导入一个虚函数表，表中的个数是声明的虚函数的个数加上一个或两个slots(用来支持运行类型识别)**
    
- **在每个对象中加入vptr，提供执行期的链接，使每一个类能找到相应的虚函数表**
    
- **加强构造函数，使它能够为vptr设定初值，让它指向对应的虚函数表，这可能意味着在派生类和每一个基类的构造函数中，重新设定vptr的值**
    
- **加强析构函数，使它能够消抹“指向类的相关虚函数表”的vptr,vptr很可能以及在子类析构函数中被设定为子类的虚表地址。**
    
- **以下是三种情况**不同的继承下会有不同的布局
    
- Vptr放在尾端
    
- ![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)
    
- vptr放在前端
    
- ![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)
    

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

含虚函数的数据分布

- ### 多重继承
    
- **单一继承特点：**派生类和父类对象都是从相同的地址开始，区别只是派生类比较大能容纳自己的非静态成员变量
    
- 多重继承下会比较复杂
    

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

多重继承关系

- **一个派生对象，把它的地址指定给最左边的基类，和单一继承一样，因为起始地址是一样的，但是后面的需要更改，因为需要加上前面基类的大小，才能得到后面基类的地址**
    

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

多重继承数据分布

- ### 虚继承
    
- STL标准库中使用的虚继承：
    
- 虚继承关系图
    
- ![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)
    
- **虚继承关系：**
    
      
    
    ![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)
    
    虚继承例子  
    
- **虚继数据在内存中的分布**
    
- 虚继承数据模型
    
- ![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)
    
- 虚继承数据模型2
    
- ![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)
    

## 3.5 对象成员的效率

- 程序员如果关心程序效率，应该实际**测试**，不要光凭推论、常识判断或假设。
    
- 优化操作并不一定总是能够有效运行，我不止一次以优化方式来 编译一个已通过编译的正常程序，却以失败收场
    

## 3.6 指向数据成员的指针

- vptr通常放在**起始处或尾端**，与编译器有关，C++标准允许放在类中的任何位置
    
- 取某个类成员变量的地址，通常取到得的是在**类的首地址的偏移位置**
    

- **例如** `& Point3d::z;` **将得到在类的偏移位置，最低限度是类的成员大小总和，而这个偏移量通常都被加上了1**
    

- **如果用一个真正绑定类对象（也就是使用 . 操作符访问成员变量）去取地址，得到的将会是内存中真正的地址**
    
- 在多重继承下，若要蒋第二个积累的指针和一个与派生类绑定的成员结合起来，那么将会因为需要加入偏移量而变得相当复杂
    

# 四、Function 语意学

- **C++支持三种类型的成员函数：static 、non-static 、virtual**
    

- **不能直接存取non-static数据**
    
- **不能被声明为const**
    
- static函数**限制**：
    

## 4.1 成员函数的各种调用方式

- **非静态成员函数：C++会保证至少和一般的普通的函数有相同的效率，经过三个步骤的转换**
    

- **改写函数，安插一个额外的参数到该函数中，用来提供一个存取管道------即this指针**
    
- **对每一个非静态成员的存取操作改成使用this指针来调用**
    
- **将成员函数改写成一个外部函数，并且名称改为独一无二的**
    

- **虚函数成员函数：也会经过类似的转化**
    

- **vptr是编译器产生的指针，指向虚函数表，其名称也会被改为独一无二**
    
- **1 是该函数在虚函数表中的索引**
    
- **ptr 则是this指针**
    
- 例如：`ptr->normalize()`会被转化为
    
- `( * ptr->vptr[1] )( ptr )`
    

- 静态成员函数：它**没有this指针**，因此会有以下**限制**：
    

- **它不能直接存取类中的非成员变量**
    
- **它不能够被声明为const、volatile 和 virtual**
    
- **它不需要经过类的对象才能被调用----虽然很多事情况是这样调用的**
    

## 4.2 详解虚成员函数

- 虚函数一般实现模型：**每一个类都有一个虚函数表，内含该类中有作用的虚函数地址，然后每个对象有一个虚函数指针，指向虚表位置**
    
- **多态含义：以一个基类的指针（或引用），寻址出一个子类对象**
    
- **什么是积极多态？**
    
- 当被指出的对象**真正使用**时，多态就变成积极的了
    
- ### 单一继承虚函数布局图
    

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

单一继承虚函数图

- **一个类派生自Point会发生什么事？三种可能的情况**
    

- **它可以继承基类所声明的虚函数的函数实例**，**该函数实例的地址会被拷贝进子类的虚表的相对应的slot之中**
    
- **它可以使用自己的虚函数实例----它自己的函数实例地址必须放在对应的slot之中**
    
- **它可以加入一个新的虚函数，这时候虚函数表的尺寸会增加一个slot，而新加入的函数实例地址也会被放进该slot之中**
    

- ### 多重继承下的虚函数
    
- 多继承虚函数图
    
- ![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)
    
- ### 虚拟继承下的虚函数
    
- 虚继承下的虚函数图
    
- ![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)
    
- ## 4.4指向成员函数的指针
    
- **对于普通的成员函数，编译器会将其转化为一个函数指针，然后使用成员函数的地址去初始化**
    

- 例如  `double (Point :: *pmf ) ( );`
    
- **转化为**  `double ( Point :: coord )( ) = &Point :: x;`
    
- **这样调用**  `(coord) (& origin)或 (coord)(ptr)`
    

- **对一个虚函数取地址，在vc编译器下，要么得到vacll thunk地址（虚函数时候），要么得到的是函数地址（普通函数）**
    

## 4.5 内联函数

- **inline只是向编译器提出一个请求，是否真的优化取决于编译器自己的判定**
    
- 对于形式参数，会采用：
    

- 常量表达式替换
    
- 常量替换
    
- 引入临时变量来避免多次求值操作
    

- 对于局部变量，会采用：
    

- 使用临时变量
    

# 五、构造、析构、拷贝语意学

- **应注意的一些问题：**
    

- **构造函数不要写为纯虚函数，因为当抽象类中有数据的时候，将无法初始化**
    
- **把所有函数设计成虚函数，再由编译器去除虚函数是错误的，不应该成为虚函数的函数不要设计成虚函数**
    
- **当你无法抉择一个函数是否需要为const时，尤其是抽象类，最好不设置成const**
    

## 5.1 "无继承" 情况下的对象构造

- **对于没有初始化的全局变量，C语言中会将其放在一个未初始化全局区，而C++会将所有全局对象初始化**
    
- 对于不需要构造函数、析构函数、赋值操作的类或结构体，C++会将其打上**POD**标签，赋值时按照c那样的位搬运
    
- **对于含有虚函数的类，编译器会在构造函数的开始放入一些初始化虚表和虚指针的操作**
    
- **面对函数以值方式返回，编译器会将其优化为加入一个参数的引用方式，避免多次构造函数**
    

## 5.2 继承体系下的对象构造

- 构造函数会含有大量的隐藏嘛，因为编译器会进行**扩充**：
    

- **如果类被列于成员初始化列表中，那么如果有任何显式指定的参数，都应该传递过去。若没有列于list中，而类中有一个默认构造，亦应该调用**
    
- 此外，类中的每一个虚基类的偏移位置必须**在执行期可被存取**
    
- **如果类对象是最底层的类，其构造函数可能被调用，某些用以支持这一行为的机制必须被放进来**
    
- **如果基类被列于成员初始化列表中，那么任何显式指定的参数应该传递过去**
    
- **如果基类没有被列于基类初始化列表中，而它有默认的构造函数，那么就调用**
    
- **如果基类是多重继承下的第二个或后继的基类，那么this指针必须有所调整**
    
- 记录在成员**初始化列表中的成员数据**初始化会被放进构造函数的本体，并以成员在类中**声明**的顺序为顺序
    
- 如果有一个成员并**没有**出现在成员初始化列表中，但是它由一个**默认构造函数**，那么也会被**调用**
    
- **在那之前，如果类对象由虚表指针，它必须被设定初值，指向适当的虚表**
    
- **在那之前**，**所有**上一层的**基类构造函数**必须被调用，以**基类声明的顺序**（不是成员初始化列表出现的顺序）
    
- **在那之前**，所有**虚基类构造函数**必须被调用，**从左到右，从最深到最浅**
    

- 不要忘记在赋值函数中，**检查自我赋值的情况**
    

## 5.3 对象复制语意学

- 当一个类复制给另一个类时，能采用的有三种方式：
    

- 如果有需要，会自动生成一个**浅拷贝**，至于什么时候需要**深拷贝**（见第二章讲）
    
- **什么都不做，会实施默认行为**
    
- **提供一个拷贝复制运算符**
    
- **显式地拒绝把一个类拷贝给另一个**
    

- **虚基类会使其复制操作调用一次以上，因此我们应该避免在徐杰类中声明数据成员**
    

## 5.5 析构语意学

- 什么时候会**合成析构函数**？
    

- **在类内含的成员函数有析构函数**
    
- **基类含有析构函数**
    

- 析构的**正确顺序**：
    

- **析构函数的本体首先被执行，vptr会在程序员的代码执行前被重设。**
    
- **如果类拥有成员类对象，而后者拥有析构函数，那么他们会以其声明顺序的相反顺序被调用**
    
- **如果类内涵一个vptr,现在被重新设定，指向适当的积累的虚表**
    
- **如果有任何直接的非虚基类拥有析构函数，它们会以其声明顺序的相反顺序被调用**
    
- **如果有任何虚基类拥有析构函数，而且目前讨论的这个类是最尾端的类，那么它们会以其原来的构造顺序的相反顺序被调用**
    

# 六、执行期语意学

- **C++难以从程序源码看出表达式的复杂过程，因为你并不知道编译器会在其中添加多少代码**
    
- 编译器对**不同**的对象会做**不同**的操作：
    

- 如果对象没有定义构造函数和析构函数，那么编译器只需要分配需要存储10个连续空间即可
    
- 如果有构造函数，且如果有名字，则会分为是否含有虚基类调用不同的函数来构造，如果没有名字，例如没有knots，则会使用new来分配在堆中
    
- 当声明结束时会有类似构造的析构过程
    
- 我们无法在程序中取出一个构造函数的地址
    
- 会增加临时的对象用来判断其是否被构造，用来保证在**第一次进入含有该静态对象的起始处**调用一次构造函数，并且**在离开文件的时候利用临时对象判断是否已经被构造来决定是否析构**
    
- 如果异常处理被支持，那么那些对象将不能被放置到try区段之内
    
- 增加了程序的复杂度
    
- 对于**全局对象**：编译器会在添加__main函数和_exit函数(和C库中的不同)，并且在这两个函数中对所有全局对象进行静态初始化和析构
    
    全局对象编译器操作
    
- ![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)
    
- 使用被静态初始化的对象，有一些**缺点**：
    
- 因此，不建议用那些需要静态初始化的全局对象
    
- 对于**局部静态对象**：
    
- 对于**对象数组**:（例如`Point knots[10]`）
    

## 6.2 new 和 delete 运算符

- 对于**普通类型变量**：
    
- 例如 `int *pi = new int(5)`
    

- 调用函数库中的new运算符`if(int *pi = _new (sizeof(int) ))`
    
- 再配置初值 `*pi = 5`
    

- 对于delete来说 `delete pi;`
    

- 则先进行保护 `if( pi != 0)`
    
- 再调用delete   `_delete(pi)`
    

- 对于**成员对象**：
    

- 对于`Point3d* origin = new Point3d`
    
- 实际调用operator new，其代码如下
    
- ```
    extern void* operator new (size_t size){    if(size == 0)        size = 1;    void * last_alloc;    while(!(last_alloc = malloc(size))){        if(_new_handler)            ( *_new_handler)();        else            return 0;    }    return last_alloc;}
    ```
    
- 语言要求每一次对new的调用都必须传回一个独一无二的指针，为了解决这个问题，传回一个指向默认为1Byte的内存区块，允许程序员自己定义_new_handler函数，并且循环调用
    
- 至于delete也相同
    
    `extern void operator delete (void *ptr){    if(ptr)     free( (char*)ptr)   }   `
    
- 对于对象数组，会在分配的内存上方放上cookies，来存储数组个数，方便delete调用来析构
    
- **程序员最好避免以一个基类指向一个子类所组成的数组---如果子类对象比其基类大的话**
    
- 解决方式：
    
- `for(int ix = 0; ix < elem_count; ++ix){    Point3d *p = &((Point3d*)ptr)[ix];    delete p;   }   `
    
- **程序员必须迭代走过整个数组，把delete运算符实施与每一个元素身上。以此方式，调用操作将是virtual。因此，Point3d和Point的析构函数都会实施与每一个对象上**
    

- Placement Operator new的语意
    

- **有一个预先定义好的重载的new运算符，称为placement operator new。它需要第二个参数，类型为void***
    
- 形如 `Point2w *ptw = new (arena) Point2w`,**其中arena指向内存中的一个区块，用以放置新产生出来的Point2w 对象**
    
- `void* operator new(size_t , void* p){    return p;   }   `
    
- **如果我们在已有对象的基础上调用placement new的话，原来的析构函数并不会被调用，而是直接删除原来的指针，但是不能使用delete 原来的指针**
    
- 正确的方法应该是 :
    
    `//错误：   delete p2w;   p2w = new(arwna) Point2w;   //正确：   p2w->Point2w;   p2w = new(arena) Point2w;   `
    

- ## 6.3 临时性对象
    
- 临时对象在类的表达式并赋值，函数以值方式传参等都会**产生临时对象**-----而临时对象会构造和析构，所以会拖慢程序的效率，我们应该尽量避免
    

# 七、站在对象模型的顶端

- 三个著名的C++语言扩充性质：**模板、异常（EH）、RTTI（runtime type identification）**
    
- ## 7.1 Template
    
- **模板实例化时间线**：
    

- **当编译器看到模板类的声明时，什么都不会做，不会进行实例化**
    
- **模板类中明确类型的参数，通过模板类的某个实例化版本才能存取操作**
    
- **即使是静态类型的变量，也需要与具体的实例版本关联，不同版本有不同的一份**
    
- **如果声明一个模板类的指针，那么不会进行实例化，但是如果是引用，那么会进行实例化**
    
- **对于模板类中的成员函数，只有在函数使用的时候才会进行实例化**
    

- 模板名称决议的方法-----**即如果非成员函数在类中调用，那么会调用名称相同的哪个版本：**
    

- **会根据该函数是否与模板有关来判断，如果是已知类型，那么会在定义的范围内直接查找，如果以来模板的具体类型，那么会在实例化的范围查找**
    

- ## 7.2 异常处理
    
- C++**异常处理**由**三个**主要的语汇组件：
    

- **一个throw子语。它在程序某处发出一个exception，exception可以说内建类型也可以是自定义类型**
    
- **一个或多个catch子句。每一个catch子句都是一个exceotion hander，它用来表示说，这个子句准备处理某种类型exception，并且在封闭的大括号区段中提供实际的处理程序**
    
- **一个try区段。它被围绕一系列的叙述句，这些叙述句可能会引发catch子句起作用**
    

- **当一个异常被抛出去，控制权会从函数调用中被释放出来，并寻找一个吻合的catch子句。如果都没有吻合者，那么默认的处理例程 terminate()会被调用，当控制权被放弃后，堆栈中的每一个函数调用也就被推离。这个程序被称为 unwingding the stack 。在每一个函数被推离堆栈之前，函数的局部类的析构会被调用**
    
- **因此一个解决办法就是将类封装在一个类中，这样变成局部类，如果抛出异常也会被自动析构**
    
- 当一个异常抛出，**编译系统必须**：
    

- **摧毁所有活跃局部类**
    
- **从堆栈中将目前的函数(unwind)释放掉**
    
- **进行到程序堆栈的下一个函数中去，然后重复上述步骤2—5**
    

1. **检查发生throw操作的函数**
    
2. **决定throw操作是否发生在try区段**
    
3. **若是，编译系统必须把异常类型拿来和每一个catch子句进行比较**
    
4. **如果比较后吻合，流程控制应该交到catch子句中**
    
5. **如果throw的发生并不在try区段中，或没有一个catch子句吻合，那么系统必须：**
    

- ## 7.3 执行期类型识别
    
- 当两个类有继承关系的时候，我们有转换需求时，可以进行向下转型，但是很多时候是**不安全的**
    
- **C++的RTTI (执行期类型识别)提供了一个安全的向下转型设备，但是只对多态（继承和动态绑定）的类型有效，其用来支持RTTI的策略就是，在C++的虚函数表的第一个slot处，放一个指针，指向保存该类的一些信息----即type_info类（在编译器文件中可以找到其定义）**
    
- **dynamic_cast运算符可以在执行期决定真正的类型**
    

- **如果引用真正转换到适当的子类，则可以继续**
    
- **如果不能转换的化，会抛出 bad_cast 异常**
    
- **通常使用 try 和 catch 来进行判断是否成功**
    
- **如果转型成功，则会返回一个转换后的指针**
    
- **如果是不安全的，会传回0**
    
- **放在程序中通过 if 来判断是否成功，采取不同措施**
    
- 对于**指针**来说：
    
- 对于**引用**来说：
    

- **Typeid运算符**：
    

- **可以传入一个引用,typeid运算符会传回一个const reference,类型为type_info。其内部已经重载了 == 运算符，可以直接判断两个是否相等，回传一个bool值**
    
- 例如  `if( typeid( rt ) == typeid( fct ) )`
    
- **RTTI虽然只适用于多态类，但是事实上type_info object也适用于内建类，以及非多态的使用者自定类型，只不过内建类型 得到的type_info类是静态取得，而不是执行期取得**
    

![](https://res.wx.qq.com/op_res/NN_GToMiIjsXzgPzF9-74ZzwR3cA9-fv3o9eWo8f5gQWqx71CmGlY8kFxuIxZaG0TB1bFeMCmh1DGN_pWMRg0A)

**帐号已迁移**

公众号

Reads 1932

​

Comment

**留言 2**

- 简之
    
    2021年11月18日
    
    Like1
    
    不错，看完书可以做回顾提纲
    
- Kosmo
    
    2021年11月18日
    
    Like1
    
    最近想看还没看，可以和书一起啃了，非常感谢！
    

已无更多数据

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/ibLeYZx8Co5JKf72TOeLcba56VknmOtKrMWnS3gyv2Z3RPZ6S28sAtAKSyozOHMDzI8LEkz8ic8eH2v4ZysDq6sQ/300?wx_fmt=png&wxfrom=18)

C语言与CPP编程

11113

2

Comment

**留言 2**

- 简之
    
    2021年11月18日
    
    Like1
    
    不错，看完书可以做回顾提纲
    
- Kosmo
    
    2021年11月18日
    
    Like1
    
    最近想看还没看，可以和书一起啃了，非常感谢！
    

已无更多数据