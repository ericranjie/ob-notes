程序喵大人
 _2021年12月08日 08:24_
The following article is from 编程往事 Author 果冻虾仁
C++码农，brpc committer，搜广推在线工程。专注互联网后端技术分享、行业观察以及个人成长！也欢迎关注我的知乎：果冻虾仁

> 首先声明一点，虚表并非是C++语言的官方标准的一部分，只是各家编译器厂商在实现多态时的解决方案。另外即使同为虚表不同的编译器对于虚表的设计可能也是不同的，本文主要基于`Itanium C++ ABI`（适用于gcc和clang）。

## 从C的POD类型到C++的类

首先回顾一下C语言纯POD的结构体（struct）。如果用C语言实现一个类似面向对象的类，应该怎么做呢？

### 写法一
```cpp
#include <stdio.h>   typedef struct Actress {       int height; // 身高       
	int weight; // 体重      
	int age;    // 年龄（注意，这不是数据库，不必一定存储生日）          
	void (*desc)(struct Actress*);   
} Actress;      // obj中各个字段的值不一定被初始化过，   // 通常还会在类内定义一个类似构造函数的函数指针，这里简化   void profile(Actress* obj) {       printf("height:%d weight:%d age:%d\n", obj->height, obj->weight, obj->age);   }      int main() {       Actress a;       a.height = 168;       a.weight = 50;       a.age = 20;       a.desc = profile;          a.desc(&a);       return 0;   }   
```
想达到面向对象中数据和操作封装到一起的效果，只能给struct里面添加函数指针，然后给函数指针赋值。然而在C语言的项目中你很少会看到这种写法，主要原因就是函数指针是有空间成本的，这样写的话每个实例化的对象中都会有一个指针大小（比如8字节）的空间占用，如果实例化N个对象，每个对象有M个成员函数，那么就要占用`N*M*8`的内存。

所以通常C语言不会用在struct内定义成员函数指针的方式，而是直接：

### 写法二
```cpp
#include <stdio.h> 
typedef struct Actress {
int height; // 身高
int weight; // 体重
int age;    // 年龄（注意，这不是数据库，不必一定存储生日）      } Actress;      void desc(Actress* obj) {       printf("height:%d weight:%d age:%d\n", obj->height, obj->weight, obj->age);   }      int main() {       Actress a;       a.height = 168;       a.weight = 50;       a.age = 20;          desc(&a);       return 0;   }   
```
Redis中AE相关的代码实现，便是如此。

再看一个C++普通的类：
```cpp
#include <stdio.h>   
class Actress {   public:
	int height; // 身高
    int weight; // 体重
    int age;    // 年龄（注意，这不是数据库，不必一定存储生日）
    void desc() {           printf("height:%d weight:%d age:%d\n", height, weight, age);       }   };      int main() {       Actress a;       a.height = 168;       a.weight = 50;       a.age = 20;          a.desc();       return 0;   }   
```
你觉得你这个class实际相当于C语言两种写法中的哪一个？

看着像写法一？其实相当于写法二。C++编译器实际会帮你生成一个类似上例中C语言写法二的形式。这也算是C++ `zero overhead`(`零开销`)原则的一个体现。

> **You shouldn't pay for what you don't use.**

当然实际并不完全一致，因为C++支持重载的关系，会存在命名崩坏。但主要思想相同，虽不中，亦不远矣。

看到这，你会明白：C++中类和操作的封装只是对于程序员而言的。而编译器编译之后其实还是面向过程的代码。编译器帮你给成员函数增加一个额外的类指针参数，运行期间传入对象实际的指针。类的数据（成员变量）和操作（成员函数）其实还是分离的。

每个函数都有地址（指针），不管是全局函数还是成员函数在编译之后几乎类似。

在类不含有虚函数的情况下，编译器在编译期间就会把函数的地址确定下来，运行期间直接去调用这个地址的函数即可。这种函数调用方式也就是所谓的`静态绑定`（`static binding`）。

## 何谓多态？

虚函数的出现其实就是为了实现面向对象三个特性之一的`多态`（`polymorphism`）。
```cpp
#include <stdio.h>
#include <string>
using std::string;   class Actress {   public:       Actress(int h, int w, int a):height(h),weight(w),age(a){};          virtual void desc() {           printf("height:%d weight:%d age:%d\n", height, weight, age);       }          int height; // 身高       int weight; // 体重       int age;    // 年龄（注意，这不是数据库，不必一定存储生日）   };      class Sensei: public Actress {   public:       Sensei(int h, int w, int a, string c):Actress(h, w, a),cup(c){};       virtual void desc() {           printf("height:%d weight:%d age:%d cup:%s\n", height, weight, age, cup.c_str());       }       string cup;      };      int main() {       Sensei s(168, 50, 20, "36D");          s.desc();       return 0;   }   
```
上例子，最终输出显而易见：

> height:168 weight:50 age:20 cup:36D

再看：
```cpp
    Sensei s(168, 50, 20, "36D");          Actress* a = &s;       a->desc();          Actress& a2 = s;       a2.desc();
```
这种情况下，用父类指针指向子类的地址，最终调用desc函数还是调用子类的。输出：
```cpp
height:168 weight:50 age:20 cup:36D   height:168 weight:50 age:20 cup:36D   
```
这个现象称之为`动态绑定`（`dynamic binding`）或者`延迟绑定`（`lazy binding`）。

但倘若你 把父类Actress中desc()函数前面的`vitural`去掉，这个代码最终将调用父类的函数desc()，而非子类的desc()！输出：
```cpp
height:168 weight:50 age:20   height:168 weight:50 age:20   
```
这是为什么呢？指针实际指向的还是子类对象的内存空间，可是为什么不能调用到子类的desc()？这个就是我在第一部分说过的：**类的数据（成员变量）和操作（成员函数）其实是分离的。**

仅从对象的内存布局来看，只能看到成员变量，看不到成员函数。因为调用哪个函数是编译期间就确定了的，编译期间只能识别父类的desc()。

好了，现在我们对于C++如何应用多态有了一定的了解，那么多态又是如何实现的呢？

## 终于我们谈到虚表

C++具体多态的实现一般是编译器厂商自由发挥的。但无独有偶，使用虚表指针来实现多态几乎是最常见做法（基本上已经是最好的多态实现方法）。废话不多说，继续看代码，有微调：
```cpp
#include <stdio.h>
class Actress {   public:       Actress(int h, int w, int a):height(h),weight(w),age(a){};          virtual void desc() {           printf("height:%d weight:%d age:%d\n", height, weight, age);       }          virtual void name() {           printf("I'm a actress");       }          int height; // 身高       int weight; // 体重       int age;    // 年龄（注意，这不是数据库，不必一定存储生日）   };      class Sensei: public Actress {   public:       Sensei(int h, int w, int a, const char* c):Actress(h, w, a){           snprintf(cup, sizeof(cup), "%s", c);       };       virtual void desc() {           printf("height:%d weight:%d age:%d cup:%s\n", height, weight, age, cup);       }       virtual void name() {           printf("I'm a sensei");       }       char cup[4];      };      int main() {       Sensei s(168, 50, 20, "36D");       s.desc();          Actress* a = &s;       a->desc();          Actress& a2 = s;       a2.desc();       return 0;   }   
```
父类有两个虚函数，子类重载了这两个虚函数。

clang有个命令可以输出对象的内存布局（不同编译器内存布局未必相同，但基本类似）：
```cpp
clang -cc1 -fdump-record-layouts -stdlib=libc++ actress.cpp   
```
可以得到：
```cpp
*** Dumping AST Record Layout            0 | class Actress            0 |   (Actress vtable pointer)            8 |   int height           12 |   int weight           16 |   int age              | [sizeof=24, dsize=20, align=8,              |  nvsize=20, nvalign=8]      *** Dumping AST Record Layout            0 | class Sensei            0 |   class Actress (primary base)            0 |     (Actress vtable pointer)            8 |     int height           12 |     int weight           16 |     int age           20 |   char [4] cup              | [sizeof=24, dsize=24, align=8,              |  nvsize=24, nvalign=8]   
```
内存布局、大小、内存对齐都一目了然。

可以发现父类Actress的起始位置多了一个`Actress vtable pointer`。子类Sensei是在父类的基础上多了自己的成员cup。

也就是说在含有虚函数的类编译期间，编译器会自动给这种类在起始位置追加一个虚表指针，一般称之为：`vptr`。vptr指向一个虚表，称之为：`vtable` 或`vtbl`，虚表中存储了实际的函数地址。

再看下虚表存储了什么东西。你在网上搜一下资料，肯定会说虚表里存储了虚函数的地址，但是其实不止这些！clang同样有命令:
```cpp
clang -Xclang -fdump-vtable-layouts -stdlib=libc++ -c actress.cpp   
```
g++也有打印虚表的操作（请在Linux上使用g++），会自动写到一个文件里：
```cpp
g++ -fdump-class-hierarchy actress.cpp   
```
看下clang的结果：
```cpp
Vtable for 'Actress' (4 entries).      0 | offset_to_top (0)      1 | Actress RTTI          -- (Actress, 0) vtable address --      2 | void Actress::desc()      3 | void Actress::name()      VTable indices for 'Actress' (2 entries).      0 | void Actress::desc()      1 | void Actress::name()      Vtable for 'Sensei' (4 entries).      0 | offset_to_top (0)      1 | Sensei RTTI          -- (Actress, 0) vtable address --          -- (Sensei, 0) vtable address --      2 | void Sensei::desc()      3 | void Sensei::name()      VTable indices for 'Sensei' (2 entries).      0 | void Sensei::desc()      1 | void Sensei::name()  
```
g++的结果（其实也比较清晰，甚至更清晰）：
```cpp
Vtable for Actress   Actress::_ZTV7Actress: 4u entries   0     (int (*)(...))0   8     (int (*)(...))(& _ZTI7Actress)   16    (int (*)(...))Actress::desc   24    (int (*)(...))Actress::name      Class Actress      size=24 align=8      base size=20 base align=8   Actress (0x0x7f9b1fa8c960) 0       vptr=((& Actress::_ZTV7Actress) + 16u)      Vtable for Sensei   Sensei::_ZTV6Sensei: 4u entries   0     (int (*)(...))0   8     (int (*)(...))(& _ZTI6Sensei)   16    (int (*)(...))Sensei::desc   24    (int (*)(...))Sensei::name      Class Sensei      size=24 align=8      base size=24 base align=8   Sensei (0x0x7f9b1fa81138) 0       vptr=((& Sensei::_ZTV6Sensei) + 16u)     Actress (0x0x7f9b1fa8c9c0) 0         primary-for Sensei (0x0x7f9b1fa81138)   
```
可以看出二者其实基本一致，只是个别名称叫法不同。
![[Pasted image 20241006162004.png]]
![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

所有虚函数的的调用取的是哪个函数（地址）是在运行期间通过查虚表确定的。

更新：vptr指向的并不是虚表的表头，而是直接指向的虚函数的位置。使用gdb或其他工具可以发现：
```cpp
(gdb) p s   $2 = {<Actress> = {_vptr.Actress = 0x400a70 <vtable for Sensei+16>, height = 168, weight = 50, age = 20}, cup = "36D"}   
```
vptr指向的是Sensei的`vtable + 16`个字节的位置，也就是虚表的地址。

虚表本身是连续的内存。动态绑定的实现也就相当于（假设p为含有虚函数的对象指针）：
```cpp
(*(p->vptr)[n])(p)   
```
但其实上面的图片也只是简化版，不是完整的的虚表。通过gdb查看，你其实可以发现子类和父类的虚表是连在一起的。上面gdb打印出了虚表指针指向：0x400a70。我们倒退16个字节（0x400a60）输出一下：
![[Pasted image 20241006161956.png]]
可以发现子类和父类的虚表其实是连续的。并且下面是它们的typeinfo信息也是连续的。

虚表的第一个条目`vtable for Sensei`值为0。

虚表的第二个条目`vtable for Sensei+8`指向的其实是0x400ab0，也就是下面的`typeinfo for Sensei`。

再改一下代码。我们让子类Sensei只重载一个父类函数desc()。
```cpp
class Sensei: public Actress {   public:       Sensei(int h, int w, int a, const char* c):Actress(h, w, a){           snprintf(cup, sizeof(cup), "%s", c);       };       virtual void desc() {           printf("height:%d weight:%d age:%d cup:%s\n", height, weight, age, cup);       }       char cup[4];      };   
```
其他地方不变，重新用clang或g++刚才的命令执行一遍。clang的输出：
```cpp
Vtable for 'Actress' (4 entries).      0 | offset_to_top (0)      1 | Actress RTTI          -- (Actress, 0) vtable address --      2 | void Actress::desc()      3 | void Actress::name()      VTable indices for 'Actress' (2 entries).      0 | void Actress::desc()      1 | void Actress::name()      Vtable for 'Sensei' (4 entries).      0 | offset_to_top (0)      1 | Sensei RTTI          -- (Actress, 0) vtable address --          -- (Sensei, 0) vtable address --      2 | void Sensei::desc()      3 | void Actress::name()      VTable indices for 'Sensei' (1 entries).      0 | void Sensei::desc()
```
可以看到子类的name由于没有重载，所以使用的还是父类的。一图胜千言：
![[Pasted image 20241006161944.png]]
写了这么多，相信大家应该已经能理解虚表存在的意义及其实现原理。但同时我也埋下了新的坑没有填：

> 虚表中的前两个条目是做什么用的？

它俩其实是为多重继承服务的。

- 第一个条目存储的offset，是一种被称为`thunk`的技术（或者说技巧）。
- 第二个条目存储着为`RTTI`服务的`type_info`信息。
    

关于这部分的介绍，请关注后续文章！

---

往期推荐


研究了一下Android JNI，有几个知识点不太懂。



](http://mp.weixin.qq.com/s?__biz=MzkyODE5NjU2Mw==&mid=2247491051&idx=1&sn=2df322c364fa1fcb95f92357cc2f60c3&chksm=c21d2f57f56aa641ff218d3b1ae7b1382deb415ee5c13ef4af11f78fb4e5a6d609fe58af2168&scene=21#wechat_redirect)

[

介绍一个C++中非常有用的设计模式



](http://mp.weixin.qq.com/s?__biz=MzkyODE5NjU2Mw==&mid=2247491077&idx=1&sn=f78a45bd27b6225cfd989c51b3bb4485&chksm=c21d2cb9f56aa5afc2c967c887a6cc61a82a53af429332f3518521048545148d09e410444760&scene=21#wechat_redirect)

[

60 张图详解 98 个常见网络概念



](http://mp.weixin.qq.com/s?__biz=MzkyODE5NjU2Mw==&mid=2247491049&idx=1&sn=00ec9d30c764b28a8faf0a8ccbc4d405&chksm=c21d2f55f56aa643285763d8e442f4b5bd8fbd5e9596c3fdec4473229ecbd12ecc1685f02e96&scene=21#wechat_redirect)

[

没办法，基因决定的！



](http://mp.weixin.qq.com/s?__biz=MzkyODE5NjU2Mw==&mid=2247490984&idx=2&sn=fad819733c717ffd708a317adef3729b&chksm=c21d2f14f56aa6024a3847657e618ae571eb1bc053a673526213c7b48ded8b19e151a84c5958&scene=21#wechat_redirect)

[

哪家互联网公司一周工作时间最长？？太卷了！！！



](http://mp.weixin.qq.com/s?__biz=MzkyODE5NjU2Mw==&mid=2247490984&idx=1&sn=44856b5a549d725cabb0ee30c1cd1cb6&chksm=c21d2f14f56aa60238560a938ef82563aa075b3f0769bb7d64c2d016b377bf0d02ee73ff2c4e&scene=21#wechat_redirect)

[

C++的lambda是函数还是对象？



](http://mp.weixin.qq.com/s?__biz=MzkyODE5NjU2Mw==&mid=2247490981&idx=1&sn=b6f8de03c0917e3e89b9c51dfe82c58e&chksm=c21d2f19f56aa60f5856de4174d80040728686556ee6f240a1529c90f44caf41ee9c47a192ef&scene=21#wechat_redirect)

[

深入理解glibc malloc：内存分配器实现原理



](http://mp.weixin.qq.com/s?__biz=MzkyODE5NjU2Mw==&mid=2247490954&idx=1&sn=8901f03671c2a1b9e972e23e038e8875&chksm=c21d2f36f56aa620bc11c08d262e096fa34c1300d00a19e0cca2890bc14e1ebdaa0048e84aa4&scene=21#wechat_redirect)

[

到底什么是挂载？



](http://mp.weixin.qq.com/s?__biz=MzkyODE5NjU2Mw==&mid=2247490942&idx=2&sn=4c78a0e394e18c51afc51a8d92c60b73&chksm=c21d2fc2f56aa6d4da893f80953f4e59f2e61b1dcd5bc769143ee57026fc47ddd5b98606429c&scene=21#wechat_redirect)

[

C/C++为什么要专门设计个do…while？



](http://mp.weixin.qq.com/s?__biz=MzkyODE5NjU2Mw==&mid=2247490941&idx=1&sn=76c56baa78f1308f377bce82e93006f6&chksm=c21d2fc1f56aa6d72b7f6b86e1cb35d4093a8fcd9a683c776ce41f994f1ccb98045a627877f1&scene=21#wechat_redirect)

[

为什么空类大小是1



](http://mp.weixin.qq.com/s?__biz=MzkyODE5NjU2Mw==&mid=2247490920&idx=1&sn=4d91fa8ba8e88e86d7a48619c8eff94a&chksm=c21d2fd4f56aa6c2be947f417958e8a2b1ceff45123ee510680043a72336f62a8cb7a5034914&scene=21#wechat_redirect)

[

推荐一个学习技术的好网站



](http://mp.weixin.qq.com/s?__biz=MzkyODE5NjU2Mw==&mid=2247490919&idx=1&sn=62f9b373b97558357544801dc697aba1&chksm=c21d2fdbf56aa6cd60c353278b779c6503c1e4ad57a762cfa6aaa5a42744b96a9c200cb69b1c&scene=21#wechat_redirect)

[

在部队当程序员有多爽？



](http://mp.weixin.qq.com/s?__biz=MzkyODE5NjU2Mw==&mid=2247490917&idx=2&sn=41af84bf20a35d78383f7d46f902cc61&chksm=c21d2fd9f56aa6cfc3c5ba32e4f7f7a4bdb54ccf592d87d6653ed6336758b6be87ab4ec40e88&scene=21#wechat_redirect)

[

Linux最大并发数是多少？



](http://mp.weixin.qq.com/s?__biz=MzkyODE5NjU2Mw==&mid=2247490871&idx=1&sn=0d925d2c381f1061a449cc1a502421ab&chksm=c21d2f8bf56aa69da3fb49bf8e29d688d09edb6ab9144e1b00087d641343a7880b4661c2e768&scene=21#wechat_redirect)

[

C++ protected继承和private继承是不是没用的废物？



](http://mp.weixin.qq.com/s?__biz=MzkyODE5NjU2Mw==&mid=2247490783&idx=1&sn=c8917a404a17dd23e4a5adf259d13f21&chksm=c21d2e63f56aa7756e95b2a399bedcf14f4dd161febe8f3f5c9ddbb37b95805c6eb13933e15e&scene=21#wechat_redirect)

[

累够呛！整理了一份C++学习路线图！



](http://mp.weixin.qq.com/s?__biz=MzkyODE5NjU2Mw==&mid=2247490825&idx=1&sn=54200f51b9c4acc358ed4ad58c353fa0&chksm=c21d2fb5f56aa6a3c37ea3f4ef5a440806250d3fae21602573bf0cb5e9ea9ae3bbaf655e5bca&scene=21#wechat_redirect)

[

图解|工作6年多，我还是没有搞懂什么是协程的道与术



](http://mp.weixin.qq.com/s?__biz=MzkyODE5NjU2Mw==&mid=2247490744&idx=2&sn=da331c0c9255b99fbfb91495d2645aa8&chksm=c21d2e04f56aa71288711806b7ac7a20c5b8d2831a4210c83152999509da0c87375c4e195b9b&scene=21#wechat_redirect)

[

研究了一波Android Native C++内存泄漏的调试



](http://mp.weixin.qq.com/s?__biz=MzkyODE5NjU2Mw==&mid=2247490673&idx=1&sn=3eceb3d1d648d1148ceb2511663de9b2&chksm=c21d2ecdf56aa7db021af1c45e1a1044c7ee4961ba67e36e9e5fe302a8f10b3403453b75d415&scene=21#wechat_redirect)

[

参加了 40 多场面试。



](http://mp.weixin.qq.com/s?__biz=MzkyODE5NjU2Mw==&mid=2247490589&idx=1&sn=03c6d4aa03f74c857a050b3a8c500368&chksm=c21d2ea1f56aa7b7a47a6a5cd468d07519f6e6b900080901f65b38a7272a99dca8e18f9b4240&scene=21#wechat_redirect)

[

如何调试内存泄漏？方法论来了



](http://mp.weixin.qq.com/s?__biz=MzkyODE5NjU2Mw==&mid=2247490557&idx=1&sn=dd1e30f78801740f97e27969f32941c1&chksm=c21d2941f56aa057c1ca032b76e2348e82cdd3c039401ad27263b8b25aad822d5152aef46ce0&scene=21#wechat_redirect)

[

清华大学：2021 元宇宙研究报告！



](http://mp.weixin.qq.com/s?__biz=MzkyODE5NjU2Mw==&mid=2247490457&idx=1&sn=b6716858871f7576015d16d5f067c68a&chksm=c21d2925f56aa0335aa8305a1d4ae41f306a181c835bc300fee83d01b1490e7513876c42b12f&scene=21#wechat_redirect)

  

Read more

Reads 2808

​

Comment

**留言 12**

- Majesticᯤ⁶ᴳ
    
    2021年12月8日
    
    Like5
    
    Cup:36D![[闭嘴]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)，这程序写得好。。。
    
- 范经理
    
    2021年12月8日
    
    Like2
    
    父类有两个虚函数，子类重载了这两个虚函数。 ![[疑问]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)这个地方应该是重写的概念吧？
    
    程序喵大人
    
    Author2021年12月8日
    
    Like1
    
    是的，看的挺细
    
- 阿秀
    
    2021年12月8日
    
    Like
    
    我也来学学
    
    程序喵大人
    
    Author2021年12月8日
    
    Like1
    
    你别学了![[旺柴]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)
    
- cactus
    
    2021年12月9日
    
    Like
    
    using std：string 让人看到了开发的谨慎
    
    程序喵大人
    
    Author2021年12月9日
    
    Like
    
    ？
    
- Yedidya
    
    2021年12月8日
    
    Like
    
    函数指针表
    
- 心如止水
    
    2021年12月8日
    
    Like
    
    琴女的身材
    
- 最上川
    
    2021年12月8日
    
    Like
    
    喵哥写得好
    
    程序喵大人
    
    Author2021年12月8日
    
    Like
    
    这是转的，不是我写的
    
- Alex
    
    2021年12月8日
    
    Like
    
    喵哥发文，必属精品！
    

已无更多数据

[](javacript:;)

![](http://mmbiz.qpic.cn/sz_mmbiz_png/Av3j9GXyMmGmg0HNK7fR6Peq3uf9pUoN5Qt0s0ia8IH97pvIiaADsEvy9nm3YnR5bmA13O8ic0JJOicQO0IboKGdZg/300?wx_fmt=png&wxfrom=18)

程序喵大人

35Share13

12

Comment

**留言 12**

- Majesticᯤ⁶ᴳ
    
    2021年12月8日
    
    Like5
    
    Cup:36D![[闭嘴]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)，这程序写得好。。。
    
- 范经理
    
    2021年12月8日
    
    Like2
    
    父类有两个虚函数，子类重载了这两个虚函数。 ![[疑问]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)这个地方应该是重写的概念吧？
    
    程序喵大人
    
    Author2021年12月8日
    
    Like1
    
    是的，看的挺细
    
- 阿秀
    
    2021年12月8日
    
    Like
    
    我也来学学
    
    程序喵大人
    
    Author2021年12月8日
    
    Like1
    
    你别学了![[旺柴]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)
    
- cactus
    
    2021年12月9日
    
    Like
    
    using std：string 让人看到了开发的谨慎
    
    程序喵大人
    
    Author2021年12月9日
    
    Like
    
    ？
    
- Yedidya
    
    2021年12月8日
    
    Like
    
    函数指针表
    
- 心如止水
    
    2021年12月8日
    
    Like
    
    琴女的身材
    
- 最上川
    
    2021年12月8日
    
    Like
    
    喵哥写得好
    
    程序喵大人
    
    Author2021年12月8日
    
    Like
    
    这是转的，不是我写的
    
- Alex
    
    2021年12月8日
    
    Like
    
    喵哥发文，必属精品！
    

已无更多数据