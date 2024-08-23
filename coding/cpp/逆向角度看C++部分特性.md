# 

唱过阡陌 看雪学苑

 _2022年04月02日 17:59_

![Image](https://mmbiz.qpic.cn/sz_mmbiz_jpg/1UG7KPNHN8Edqm0iaLicibArzJ2Guib4IFQUtZUXCs0NpicCqQmibXIibib7rEWMkqgnm6a1K3OWrMa5fO1aRbO2tQurkg/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)  

本文为看雪论坛优秀文章  
看雪论坛作者ID：唱过阡陌

  

  

#### **单/多继承**

  

**单继承**

  

测试源码：

```
#define MAIN __attribute__((constructor))
```

  

LOG日志：  
![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

内存情况：  
![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

可以看到实际上单继承就是把 baseClass 的成员变量完全copy了一份放在了我们childClass的前面。

  

**多继承**

```
// 新增一个BaseNewClass，让ChildClass:BaseClass继承这两个Class
```

```
// LOG()日志
```

  

其实也都是成员变量按顺序往后排就完事。

####   

#### **虚函数**

```
// 测试源码
```

  

```
// 日志
```

  

由上我们可以看到这两个Class的地址的开始位置都多了一个指针，指针后面的才是我们真实的结构体值，这第一个指针就是 vptr(虚函数指针)，指向了虚函数表，然后再去读一下这个指针。

```
//读取vptr指向的位置
```

  

此时打开IDA验证一下这前两个地址就是真实的函数地址。

  
// IDA查看地址：  
![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

同理我们去看看另一个childClass类也会得到类似的结果：

```
//
```

  

第一二三个：明显就是对应的虚函数具体的函数地址。第四五个：应该是和 **C++中的RTTI机制** 相关。

// IDA查看地址：  
![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

简单归纳一下：

  

① 继承这种操作其实就是得到了一个父类数据结构的副本，他们的vptr和属于子类部分的数据结构都是独有的。

  

② 继承后父类的虚函数表也会被子类完全继承。若无覆盖时，子类的虚函数表会完全拷贝一份父类的虚函数表项，并将自己子类的虚函数表项拼接在上表后面。

  

③ 如果子类覆盖了父类的某一个虚函数，虚函数表项值改变顺序不变。

  

这里简单的提及了一下，更详细的关于虚函数的介绍可以查看 这篇文章（_https://blog.csdn.net/smartgps2008/article/details/90745271_）。

至于里面提到的关于 安全性 的反思：

  

① Base1 b1 = new Derive(); 将子类的指针转为一个父类指针，只是在c++语法上限制了其对部分操作的可能性。 "子类中的未覆盖父类的成员函数" ，对它的理解应该是：它本是是什么还是什么，语法上的限制完全可以使用指针操作来实现一定程度和语法的背道而驰。他提出的第二点 *"访问non-public的虚函数" 其实和上述这一点也差不多的意思。

  

② 补充一点：其实对于继承中的成员变量也有同样类似的效果，父类不管把成员的访问权限设置为什么，其实子类都有一个完整的拷贝，同样可以通过指针操作绕过c++语法的禁止，去访问并修改父类非公开成员变量。

####   

#### **拷贝构造**

####   

#### 源码以及汇编情况：

```
NOINLINE
```

  

// 全局视图：  
![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

列举出以下的几种情况：函数参数值传递（值传递和引用传递）

  

参见 test1 test2 可见：

值传递对于基础数据类型会直接mov出一个副本，值传递对象(class/struct)的话会调用对象的拷贝构造函数得到一个新的副本，所以对于类对象太大的情况建议使用指针传递或者使用引用传递（指针传递和引用传递在汇编层面其实是一样的都是传递了一个指针[见上图]）。

  

参见 test3 可见：

test3进行了值传递，在进入函数前先对ChildClass调用了一次拷贝构造函数，将栈上拷贝出来的该类传递进了 test3。

  

函数返回值

  

参见 test4 可见：

test4 和 test3 同样在调用前都先调用了一次拷贝构造函数，但是test4的第一个参数是在栈上提前申请好预留给test4返回的空间，第二个参数为拷贝好的指向副本的类指针，进入test4后也会发现在内部在调用了一次拷贝构造函数，也就是说值传递加上返回值这种写法相比直接引用传递会多调用两次拷贝构造函数。

  

从一个类创建另一个类

  

参见 test1(v4) 上面的两句：其实也是调用的拷贝构造函数，v4指向的拷贝好的类在栈上的首地址，第一个代表读取vptr，第二个代表读取vtable的第一个函数（child2->showLOG();就是ChildClass的第一个虚函数），然后再把自己（v4）当成this传递给这个虚函数调用。

  

拷贝构造拷贝父类

  

详见下图：// 由编译器为我们生成的拷贝构造函数

  

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

```
ChildClass(const ChildClass &child){
```

  

// 由我们自己编写的拷贝构造函数：

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

// 虚函数表：  
![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

由此可见调用子类的拷贝构造函数会先调用父类的构造函数，然后在调用当前类的拷贝构造，这里的off_85600就是 vptr ，从虚函数表中也可以看见，子类覆盖了父类的虚函数就会指向子类的虚函数。

  

若没有覆盖，表项中依旧是指向父类的函数地址，而且顺序是按照父类的虚函数表顺序排列，子类中父类没有的虚函数会按顺序继续排在后面，不同类的虚函数表其实都是在编译期就已经确定了的，不同类的虚函数表处于临近的内存区域。

  

#### **类的 构造/析构 函数调用时机**

####   

#### 详见下图（ChildClass中新增了一个析构函数），// 新增析构函数：

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

####   

#### **类继承权限**

####   

#### 类的继承权限并不会影响子类继承父类子类所拥有的父类的成员变量个数，换句话说，不管父类的成员变量是什么权限，之类都完全拥有一份父类的成员变量的拷贝（这里就不展示）。

####   

#### **类型的强转**

####   

#### 主要是针对 dynamic_cast 向下转型的情况。

```
BaseClass* baseTmp = dynamic_cast<BaseClass*>(child1);
```

  

// 向下转型  
![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

由上两图可见对于类 向上转型 dynamic_cast 和 static_cast 本质是一样的，没有做任何处理。

  

dynamic_cast 向下转型的时候是借助了 RTTI 机制，就是我们前面图中看到的vptr->vtable 除了虚函数以后的指针标识该类的类型用于动态类型转换，同样也是typeid这个操作符的信息来源，具体可以参考 这篇文章（_http://c.biancheng.net/view/2343.html_）。

  

其实虚表什么的都是在编译期间就已经完全确定了，之前还误解以为动态类型转换中的向上转型可以让该子对象调用已经被子对象覆盖的父对象的方法，想多了想多了... 但是如果真想实现这样的"向上转型"也不是不行，借助指针去操作虚函数表即可 ↓  

  

// 实现所谓的"向上转型"  

  

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

// 效果图  

  

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

最后两条日志可见，我们对同一对象调用 showLOG() 一个是父函数，一个是子函数，对应代码 815 和 819 行。

  

#### **简介 lambda**

####   

#### 细节介绍参考 

#### 这个（_https://en.cppreference.com/w/cpp/language/lambda_） 和 

#### 这篇文章（_https://zhuanlan.zhihu.com/p/384314474_）

  

引用传递和值传递  
![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

####   

#### **关于lambda表达式分为三部分解析**

#####   

##### 1、表达式位于类中 （为了看到最原始的实现，不要开编译器优化）

```
#define xASM(x) __asm __volatile__ (x)
```

  

构造以及调用testFunction  
![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

testFunction汇编也可以明显看到被IDA识别为了lambda表达式。  
![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

从这里我们可以看到虽然源码后面两个lambda表达式虽然没有捕获参数，但是依旧有一个栈地址的传递(可以理解为一个空 this)。

传递栈上地址逐个相差一个指针长度。  
![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

```
0x4*6 = 24(0x18)  sp+0x4 sp+0x8 sp+0xc
```

  

上述源代码中没有表现出来，即便是空 lambda 实现，编译器依旧会传递一个栈上地址过去，这里也不做展示了。

  

然后后面两个 lambda 实现主要是为了实践，即使不捕获任何的参数，依旧可以拿到类实例，以及去读取类成员变量。

  

在类里面的 lambda 可以理解为对 () 的重载（被IDA也是识别为重载 operator()）。

  

读取类成员变量日志  
![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

#####   

##### 2、表达式位于类外无捕获参数

```
NOINLINE
```

  

表达式位于类外无捕获参数  
![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

中间函数用来返回lambda函数真实的地址  
![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

中间跳板函数  
![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

lambda函数的实现，和普通函数没有啥差别。  
![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

类外 lambda 函数，*lambda 都会生产这样的一个跳转逻辑。  

类内 lambda 函数不管有没有捕获参数都是直接理解为匿名类重载 () 运算符。

  

testB 日志  
![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

#####   

##### 3、表达式位于类外有捕获参数

```
NOINLINE
```

  

IDA反汇编  
![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

testC 日志  
![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

从 sub_319EC(v7, 3, 4) → testNoCatch(3, 4)  

可以看出 [] 捕获的参数其实都在第一个参数，lambda的传参在 第二个参数往后，结合把lambda理解为一个重载 () 运算符的类也是自洽的。

  

从 sub_31A4C(v6, 2) → testCatch(2);  
再去对应看v6的参数，也就可以更加理解，lambda 表达式引用传递和值传递的区别，源码中的 c为一个类 （理解为→ 构造 : sub_3198C(v10, &v9); | 析构 : sub_31BB4(v10);），栈传参的时候源码中的引用传递放在最前面，其次按顺序传递参数。

  

从 sub_3198C(v10, &v9); 和 sub_31BB4(v10)  
sub_3198C(v10, &v9) → auto c = make_unique<int>(12);  
sub_31BB4(v10) → 作用域结束，对unique指针的析构。

  

从 上图 29 30 行可见：对带捕获参数的 lambda 表达式取地址得到的只是 匿名类(分配在栈上)的首地址，其实从栈的角度看也是待传参数数组的首地址。

  

没有带捕获参数的 lambda 表达式 基本上可以等价于一个普通函数，函数地址通过 来获得（编译器针对表达式特殊处理的）；带捕获参数的 lambda 表达式 不能使用 ，如果使用 & 只能获得该匿名类首地址，而且 匿名lambda类的构造函数可以理解为inline构造。

  

lambda 表达式可以使用 [=] / [&] 捕获外部 值传递 / 引用传递，编译器只会把使用到的变量按照对应传递方式传递给匿名lambda类，没用到的变量不会被拷贝。

#####   

由此上结论我们可以将 dobby hook 稍微封装一下。

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

registerHook的重载第三个参数（Callback）本来是想用模板的但是好像不太行。

得到一个类似于java函数回调一样的写法。  
这里srcCall可以用一个变长参数简写一下代码。

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

#####   

##### 函数返回对象

```
class testClass {
```

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

“SUB SP, SP, #24” 4*6 总计开辟了 6 个位置，最下面的那个位置(fp-0x4) 用于存放栈检查的fp或者说成sp。  

  

正常情况下 class() 构造出来的类就在当前函数栈中，但是这里有一个特例：对于在 函数getTestClass 中 创建在栈上的class(tmp)，实际上他真实存在的位置是在 函数testRetClass 的栈上 位置 {fp-0x8,fp-0x14},最顶上的那个位置（sp）是空的（在这里分配栈最小差值0x8），结合上述描述再去看地址 0x4F5FC 就是logd中的第三个参数 “tt.d”。  
ps：如果这里的 class 只有三个成员变量，这里的 “SUB SP, SP, #0x18” 将会变成 “SUB SP, SP, #0x10”,刚好用满栈的四个位置。

  

对于地址 0x4F5E8 这里的这个函数调用就是对 类testClass 的初始化，传递了第一个地址（函数testRetClass栈地址）进去 对 int a，b，c，d 的初始化就放在 “LOGD("testClass");” 之前。

  

##### **模板类的实现**

  

模板类的实现  
![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

实现  
![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

- 它的上一层是一个 plt 跳转。
    
- 使用模板对应生成了多个实现方法。
    

  

该文章作为日常学习理解的记录，理解可能又不准确的地方，欢迎大佬们指出！

  

  

  

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

**看雪ID：唱过阡陌**

https://bbs.pediy.com/user-home-868525.htm

*本文由看雪论坛 唱过阡陌 原创，转载请注明来自看雪社区

  

[![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458432554&idx=1&sn=c2fae9d4ea7ea778a2420b1eaf9c3049&chksm=b18f84a086f80db6cce2e1d3374a2b2302c05e3155b02da0702b9698d873ceef0647ea3ac5ae&scene=21#wechat_redirect)

  

**#** **往期推荐**

1.[CVE-2014-4113提权漏洞学习笔记](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458436393&idx=1&sn=ba0b123c2f6bbb8a064eb98c8f072883&chksm=b18ff3a386f87ab545263604908d5ea80e31f71151d4cbe88e32032b5e84d1703cf672077a2d&scene=21#wechat_redirect)  

2.[Go语言模糊测试工具：Go-Fuzz](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458436361&idx=1&sn=958edb34ca25f9d92fb82eeb805b4882&chksm=b18ff38386f87a95a93eb8421c6c38eafba637102f260a222bd78ffeb0b637f2d88a4f3e6db7&scene=21#wechat_redirect)

3.[关于黑客泄漏nvidia Windows显卡驱动代码分析](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458435353&idx=1&sn=d083ac0895a4da3cb79bf48a095a6ac2&chksm=b18f8f9386f80685afabc66b558c5699da68d263250d019e5fa285807136c7efe3852c32120d&scene=21#wechat_redirect)

4.[一个BLE智能手环的分析](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458435277&idx=1&sn=d88c653fff7d382ef439d0efa57e2fba&chksm=b18f8e4786f80751ae4423ece82db91f147d73b125c8c1aed7079dc0d00f84aba8e69fe46d24&scene=21#wechat_redirect)

5.[VT虚拟化技术笔记](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458435257&idx=1&sn=4eb2253d86aca58137430161f49ae884&chksm=b18f8e3386f80725a1d67e05c8f217ae53c097c3ca88428735a80d287ebb5686e6750b5a121e&scene=21#wechat_redirect)

6.[通过DWARF Expression将代码隐藏在栈展开过程中](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458435194&idx=1&sn=993e8680a0ac340939597cac74ffc2ab&chksm=b18f8ef086f807e6c616a2e7e0d6b2b2777b26dbef6858c602c84be38c491c4b21f66f4a7580&scene=21#wechat_redirect)

  

  

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

  

  

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**球分享**

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**球点赞**

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**球在看**

  

  

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

点击“阅读原文”，了解更多！

“雪花”创作激励计划242

“雪花”创作激励计划 · 目录

上一篇CVE-2014-4113提权漏洞学习笔记下一篇Android netlink&svc 获取 Mac方法深入分析

Read more

Reads 6501

​

Comment

**留言 1**

- unituniverse
    
    2022年4月2日
    
    Like2
    
    1.类继承、虚函数的编译实现方案有好几种。2.λ表达式存在的意义有暗示编译器作者应做就地调用优化(虽然实现也像inline那样看作者脸色)。3.原子、线程本地存储、协程等实现方案，此文无
    

已无更多数据

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/1UG7KPNHN8EGLfh77kFmnicd9WOic2ibvhCibFdB4bL4srJCgo2wnvdoXLxpIvAkfCmmcptXZB0qKWMoIP8iaibYN2FA/300?wx_fmt=png&wxfrom=18)

看雪学苑

3319

1

Comment

**留言 1**

- unituniverse
    
    2022年4月2日
    
    Like2
    
    1.类继承、虚函数的编译实现方案有好几种。2.λ表达式存在的意义有暗示编译器作者应做就地调用优化(虽然实现也像inline那样看作者脸色)。3.原子、线程本地存储、协程等实现方案，此文无
    

已无更多数据