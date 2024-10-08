# 

C语言与CPP编程

_2022年01月24日 10:14_

The following article is from 码砖杂役 Author 人民副首席码仔

\[

![](http://wx.qlogo.cn/mmhead/Q3auHgzwzM70JBeemeCELtvuUYBUkmn9fPjTyP5Hu4VYqrJwp9NCvQ/0)

**码砖杂役**.

希望对你有用

\](https://mp.weixin.qq.com/s?\_\_biz=MzkxNzQzNDM2Ng==&mid=2247493491&idx=1&sn=f7e4fc2a9035c6ee176c24ef6f05a9b5&source=41&key=daf9bdc5abc4e8d0e3de71a8849c5682c9e121dd3f3dc78e1ce45bc403fd2b0c30df84d2cc530e70a9f9c3eb467aac8f11e512af0b038d0930dff35a7bb8cdceb6eaeda69ba6790294cf5bb43ce10cdfe6902e6444c1583c3c2e6b283d9385596efce20d29ccb93ed914ac90f4379221d63b4aab2e730a4118f137f1c3895f88&ascene=14&uin=MTEwNTU1MjgwMw%3D%3D&devicetype=iMac+MacBookAir10%2C1+OSX+OSX+14.6.1+build(23G93)&version=13080710&nettype=WIFI&lang=en&session_us=gh_66162e1d5e79&countrycode=CN&fontScale=100&exportkey=n_ChQIAhIQ3zoqoYXvMS4tED8o6qy01xKUAgIE97dBBAEAAAAAAIr6JvQjkbwAAAAOpnltbLcz9gKNyK89dVj01hUcbWnaceYogwza2gnZH8346USyOIezKjqJZM7r68Wct7qGX%2BZsxOCeG%2B2aR0n8FKghOB0t3e2r2cYDIB%2BkTcBsJV%2BFKitoWKnmn6OsB6sPpms4Lh7LAQXhgJgnb12Jcr%2Fr3zYOMLCdb7c2tMF1Q3du3fh3Fmk1Jl1GV9%2FnCw%2FHBrawCAtYqhelX3KRQf0dPmg0%2BeBtAyBn4hjFUhRpBi9%2BcUWKKJ6Phg6tkgtr%2BmMsTyeswhXfMIxxWlt5HHJsb%2BDk3dB%2FqP1uS8ZoTXj1xTXisFF9XKQIIUCE%2BFWHHyqX5C6UD8KNubaRK02T6w%3D%3D&acctmode=0&pass_ticket=xT9Qw4K0bE1ZsE%2FT4vTmadfj%2Fke82ptflWBxbMh%2BZz8N8Wj2%2B5MGd9afALCQtHyB&wx_header=0#)

击上方“**C语言与CPP编程**”，选择“\*\*关注/\*\***置顶/星标公众号**”

# 

干货福利，第一时间送达！

**# 一、导语**

C++是一门被广泛使用的系统级编程语言，更是高性能后端标准开发语言；C++虽功能强大，灵活巧妙，但却属于易学难精的专家型语言，不仅新手难以驾驭，就是老司机也容易掉进各种陷阱。本文结合号主的工作经验和学习心得，对C++语言的一些高级特性，做了简单介绍；对一些常见的误解，做了解释澄清；对比较容易犯错的地方，做了归纳总结；希望借此能增进大家对C++语言了解，减少编程出错，提升工作效率。

**# 二、陷阱**

**## 1.我的程序里用了全局变量，为何进程退出会莫名其妙的core掉？**

Rule：C++在不同模块（源文件）里定义的全局变量，不保证构造顺序；但保证在同一模块（源文件）里定义的全局变量，按定义的先后顺序构造，按定义的相反次序析构。

我们程序在a.cpp里定义了依次全局变量X和Y;

按照规则：X先构造，Y后构造；进程停止执行的时候，Y先析构，X后析构；但如果X的析构依赖于Y，那么core的事情就有可能发生。

****结论****：如果全局变量有依赖关系，那么就把它们放在同一个源文件定义，且按正确的顺序定义，确保依赖关系正确，而不是定义在不同源文件；对于系统中的单件，单件依赖也要注意这个问题。

**## 2.坑中坑：std::sort()**

相信工作5年以上至少50%的C/C++程序员都被它坑过，我已经听到过了无数个悲伤的故事，《圣斗士星矢》，《仙剑》，还有别人家的项目《天天爱消除》，都有人掉坑，程序运行几天莫名奇妙的Crash掉，一脸懵逼。

**sort算法对比较函数有很强的约束，不能乱来**，如果要用，要自己提供比较函数或者函数对象，一定搞清楚什么叫“严格弱排序”，一定要满足以下3个特性：

1. 非自反性

1. 非对称性

1. 传递性

尽量对索引或者指针sort，而不是针对对象本身，因为如果对象比较大，交换（复制）对象比交换指针或索引更耗费。

**##  3.注意操作符短路**

考虑游戏玩家回血回蓝（魔法）刷新给客户端的逻辑。玩家每3秒回一点血，玩家每5秒回一点蓝，回蓝回血共用一个协议通知客户端，也就是说只要有回血或者回蓝就要把新的血量和魔法值通知客户端。

玩家的心跳函数heartbeat()在主逻辑线程被循环调用

``` void  GamePlayer::Heartbeat()``{ ```    `if (GenHP() || GenMP())`    `{`        `NotifyClientHPMP();`    ``` }``} ```

如果GenHP回血了，就返回true，否则false；不一定每次调用GenHP都会回血，取决于是否达到3秒间隔。

如果GenMP回蓝了，就返回true，否则false；不一定每次调用GenMP都会回血，取决于是否达到5秒间隔。

实际运行发现回血回蓝逻辑不对，Word麻，原来是操作符短路了，如果GenHP()返回true了，那GenMP()就不会被调用，就有可能失去回蓝的机会。你需要修改程序如下：

``` void GamePlayer::Heartbeat()``{ ```    `bool hp = GenHP();`    ``` bool mp = GenMP();``    ```    `if (hp || mp)`    ``` {``        NotifyClientHPMP(); ```    `}`  `}`

逻辑与（&&）跟逻辑或（||）有同样的问题， if (a && b) 如果a的表达式求值为false，b表达式也不会被计算。

有时候，我们会写出 if (ptr != nullptr && ptr->Do())这样的代码，这正是利用了操作符短路的语法特征。

**## 4.别让循环停不下来**

``` for (unsigned int i = 5; i >=0; --i)``{ ```    ``` //...``} ```

程序跑到这，WTF？根本停不下来啊？问题很简单，unsigned 永远 >=0，是不是心中一万只马奔腾？

解决这个问题很简单，但是有时候这一类的错误却没这么明显，你需要罩子放亮点。

**## 5.当心越界**

memcpy，memset有很强的限制，仅能用于POD结构，不能作用于stl容器或者带有虚函数的类。

带虚函数的类对象会有一个虚函数表的指针，memcpy将破坏该指针指向。

对非POD执行memset/memcpy，免费送你四个字：**自求多福**

**## 6.内存重叠**

内存拷贝的时候，如果src和dst有重叠，需要用memmov替代memcpy。

**## 7.理解user stack空间很有限**

不能在栈上定义过大的临时对象。一般而言，用户栈只有几兆（典型大小是4M，8M），所以栈上创建的对象不能太大。

**## 8.用sprintf格式化字符串的时候，类型和符号要严格匹配**

因为sprintf的函数实现里是按格式化串从栈上取参数，任何不一致，都有可能引起不可预知的错误；/usr/include/inttypes.h里定义了跨平台的格式化符号，比如PRId64用于格式化int64_t

**## 9.用c标准库的安全版本（带n标识）替换非安全版本**

比如用strncpy替代strcpy，用snprintf替代sprintf，用strncat代替strcat，用strncmp代替strcmp，memcpy(dst, src, n)要确保\[dst，dst+n\]和\[src, src+n\]都有有效的虚拟内存地址空间。多线程环境下，要用系统调用或者库函数的安全版本代替非安全版本（\_r版本），谨记strtok，gmtime等标准c函数都不是线程安全的。

**## STL容器的遍历删除要小心迭代器失效**

vector,list,map,set等各有不同的写法：

``` int main(int argc, char *argv[])``{ ```    `//vector遍历删除`    `std::vector<int> v(8);`    `std::generate(v.begin(), v.end(), std::rand);`    `std::cout << "after vector generate...\n";`    `std::copy(v.begin(), v.end(), std::ostream_iterator<int>(std::cout, "\n"));`    `for (auto x = v.begin(); x != v.end(); )`    `{`        `if (*x % 2)`            `x = v.erase(x);`        `else`            `++x;`    ``` }``   ``    ```    `std::cout << "after vector erase...\n";`    ``` std::copy(v.begin(), v.end(), std::ostream_iterator<int>(std::cout, "\n"));``   ``    ```    `//map遍历删除`    `std::map<int, int> m = {{1,2}, {8,4}, {5,6}, {6,7}};`    `for (auto x = m.begin(); x != m.end(); )`    `{`        `if (x->first % 2)`            `m.erase(x++);`        `else`            `++x;`    `}`    ``` return 0;``} ```

有时候遍历删除的逻辑不是这么明显，可能循环里调了另一个函数，而该函数在某种特定的情况下才会删除当前元素，这样的话，就是很长一段时间，程序都运行得好好的，而当你正跟别人谈笑风生的时候，忽然crash，这就尴尬了。

圣斗士星矢项目曾经遭遇过这个问题，基本规律是一个礼拜game server crash一次，折磨团队将近一个月。

比较low的处理方式可以把待删元素放到另一个容器WaitEraseContainer里保存下来，再走一趟单独的循环，删除待删元素。

当然，我们推荐在遍历的同时删除，因为这样效率更高，也显得行家里手。

**# 三、性能**

**## 空间置换时间**

通过空间换取时间是提高性能的惯用法，bitmap，int map\[\]这些惯用法要了然于胸。

**## 减少拷贝 & COW**

了解Copy On Write。

只要可能就应该减少拷贝，比如通过共享，比如通过引用指针的形式传递参数和返回值。

**## 延迟计算和预计算**

比如游戏服务器端玩家的战力，由属性a,b决定，也就是说属性a，b任何一个变化，都需要重算战力；但如果ModifyPropertyA(),ModifyPropertyB()之后，都重算战力却并非真正必要，因为修改属性A之后有可能马上修改B，两次重算战力，显然第一次重算的结果会很快被第二次的重算覆盖。

而且很多情况下，我们可能需要在心跳里，把最新的战力值推送给客户端，这样的话，ModifyPropertyA(),ModifyPropertyB()里，我们其实只需要把战力置脏，延迟计算，这样就能避免不必要的计算。

在GetFightValue()里判断FightValueDirtyFlag，如果脏，则重算，清脏标记；如果不脏，直接返回之前计算的结果。

预计算的思想类似。

**## 分散计算**

分散计算是把任务分散，打碎，避免一次大计算量，卡住程序。

**## 哈希**

减少字符串比较，构建hash，可能会多费一点存储空间，但收益可观，值得一试。

**## 日志节制**

日志的开销不容忽视，要分级，不要过于奔放，可以把日志作为debug手段，但要release干净。

**## 编译器为什么不给局部变量和成员变量做默认初始化**

因为效率，C++被设计为系统级的编程语言，效率是优先考虑的方向，c++秉持的一个设计哲学是“不为不必要的操作付出任何额外的代价”。所以它有别于java，不给成员变量和局部变量做默认初始化，如果需要赋初值，那就由程序员自己去保证。

****结论****：从安全的角度出发，不应使用未初始化的变量，定义变量的时候赋初值是一个好的习惯，很多错误皆因未正确初始化而起，C++11支持成员变量定义的时候直接初始化，成员变量尽量在成员初始化列表里初始化，且要按定义的顺序初始化。

**## 理解函数调用的性能开销，性能敏感函数考虑inline**

栈帧建立和销毁，参数传递，控制转移这些对性能有损。

X86_64体系结构因为通用寄存器数目增加到16个，所以64位系统下参数数目不多的函数调用，将会由寄存器传递代替压栈方式传递参数，但栈帧建立、撤销和控制转移依然会对性能有所影响。

**## 递归的优点、缺点**

优点：方便编写，容易理解。

缺点：运行速度变慢，所以需要预估好递归深度。

建议：优先考虑非递归实现版本。递归函数要有退出条件且不能递归过深，不然有爆栈危险。

**# 四、数据结构和容器**

**## 了解std::vector的方方面面和底层实现**

1. vector是动态扩容的，2的次方往上翻，为了确保数据保存在连续空间，每次扩充，会将原member悉数拷贝到新的内存块；不要保存vector内对象的指针，扩容会导致其失效 ；可以通过保存其下标index替代。

1. 运行过程中需要动态增删的vector，不宜存放大的对象本身 ，因为扩容会导致所有成员拷贝构造，消耗较大，可以通过保存对象指针替代。

1. resize()是重置大小；reserve()是预留空间，并未改变size()，可避免多次扩容；clear()并不会导致空间收缩 ，如果需要释放空间，可以跟空的vector交换，std::vector <t>.swap(v)，c++11里shrink_to_fit()也能收缩内存。

1. 理解at()和operator\[\]的区别 ：at()会做下标越界检查，operator\[\]提供数组索引级的访问，在release版本下不会检查下标，VC会在Debug版本会检查；c++标准规定:operator\[\]不提供下标安全性检查。

1. C++标准规定了std::vector的底层用数组实现，认清这一点并利用这一点。

**## 常用数据结构**

****数组****：内存连续，随机访问，性能高，局部性好，不支持动态伸缩，最常用，通常也是最正确的选择。

****链表****：动态伸缩，插入/脱离极快（特别是带前后驱指针），节点内存通常不连续（当然可以通过从固定内存池分配规避），不支持随机访问。只在需要快速增删、动态伸缩的有限场景下才被使用，比如游戏里面地图划分格子，每个格子维护一个玩家链表（格子进入玩家视野的时候需要遍历该列表），玩家会在格子之间频繁移动。

****查找****：3种：bst，hashtable，基于有序数组的bsearch。二叉搜索树（RBTree），这个从begin到end有序，最坏查找速度logN，坏处内存不连续，节点有额外空间浪费；hashtable，好的hash函数不好选，搜索最坏退化成链表，难以估计捅数量，开大了浪费内存，开小了增加冲突几率，扩容会卡一下，无序；基于有序数组的bsearch，局部性好，insert/delete慢。

**# 五、最佳实践**

**## 对于在启动时加载好，运行中不变化的查询结构，可以考虑用sorted array替代map，hash表等**

因为有序数组支持二分查找，效率跟map差不多。对于只需要在程序启动的时候构建（排序）一次的查询结构，有序数组相比map和hash可能有更好的内存命中性（局部命中性）。

运行过程中，稳定的查询结构（比如配置表，需要根据id查找配置表项，运行过程中不增删），有序数组是个不错的选择；如果不稳定，则有序数组的插入删除效率比map，hashtable差，所以选用有序数组需要注意适用场合。

**## std::map or std::unorder_map？**

想清楚他们的利弊，map是用红黑树做的，unorder_map底层是hash表做的，hash表相对于红黑树有更高的查找性能。hash表的效率取决于hash算法和冲突解决方法（一般是拉链法，hash桶），以及数据分布，如果负载因子高，就会降低命中率，为了提高命中率，就需要扩容，重新hash，而重新hash是很慢的，相当于卡一下。

而红黑树有更好的平均复杂度，所以如果数据量不是特别大，map是胜任的。

**## 积极的使用const**

理解const不仅仅是一种语法层面的保护机制，也会影响程序的编译和运行。

const常量会被编码到机器指令。

**## 理解四种转型的含义和区别**

避免用错，尽量少用向下转型（可以通过设计加以改进）

static_cast, dynamic_cast,const_cast,reinterpret_cast，傻傻分不清？

C++砖家说：一句话，尽量少用转型，强制类型转换是C Style，如果你的C++代码需要类型强转，你需要去考虑是否设计有问题。

**## 理解字节对齐**

字节对齐能让存储器访问速度更快。

字节对齐跟cpu架构相关，有些cpu访问特定类型的数据必须在一定地址对齐的储存器位置，否则会触发异常。

字节对齐的另一个影响是调整结构体成员变量的定义顺序，有可能减少结构体大小，这在某些情况下，能节省内存。

## 牢记3 rules和5 rules，当然C++11又多了&&的copy ctor和op=版本

只在需要接管的时候才自定义operator=和copy constructor，如果编译器提供的默认版本工作的很好，不要去自找麻烦，自定义的版本勿忘拷贝每一个成分，如果要接管就要处理好。

**## 组合优先于继承，继承是一种最强的类间关系**

典型的适配器模式有类适配器和对象适配器，一般而言，建议用对象适配的方式，而非用基于继承的类适配方式，比如STL中的queue/stack的实现就是基于对象适配。

**## 减少依赖，注意隔离**

- 最大限度的减少文件间的依赖关系，用前向声明拆解相互依赖。

- 了解pimpl技术。

- 头文件要自给自足，不要图省事all.h，不要包含不必要的头文件，也不要把该包含的头文件推给user去包含，一句话，头文件包含要不多不少刚刚好。

**## 严格配对（RAII）**

打开的句柄要关闭，加锁/解锁，new/delete，new\[\]/delete\[\]，malloc/free要配对，可以使用RAII技术防止资源泄露，编写符合规范的代码

Valgrind对程序的内存使用方式有期望，需要干净的释放，所以规范编程才能写出valgrind干净的代码，不然再好的工具碰到不按规划写的代码也是武功尽废啊。

**## 理解多继承潜在的问题，慎用多继承**

多继承会存在菱形继承的问题，多个基类有相同成员变量会有问题，需要谨慎对待。

**## 有多态用法抽象基类的析构函数要加virtual关键字**

主要是为了基类的析构函数能得到正确的调用。

virtual dtor跟普通虚函数一样，基类指针指向子类对象的时候，delete ptr，根据虚函数特征，如果析构函数是普通函数，那么就调用ptr显式（基类）类型的析构函数；如果析构函数是virtual，则会调用子类的析构函数，然后再调用基类析构函数。

**## 避免在构造函数和析构函数里调用虚函数**

构造函数里，对象并没有完全构建好，此时调用虚函数不一定能正确绑定，析构亦如此。

从输入流获取数据，要做好数据不够的处理，要加try catch；没有被吞咽的exception，会被传播

从网络数据流读取数据，从数据库恢复数据都需要注意这个问题。

**## 协议尽量不要传float，如果传float要了解NaN的概念，要做好检查，避免恶意传播**

可以考虑用整数替代浮点，比如万分之五（5%%），就保存5。

**## 定义宏要遵循常规**

要对每个变量加括弧，有时候需要加do {} while(0)或者{}，以便能将一条宏当成一个语句。要理解宏在预处理阶段被替换，不用的时候要#undef，要防止污染别人的代码，要避免宏定义中依赖特定外部变量名。

**## 智能指针**

理解基于引用计数法的智能指针实现方式，了解所有权转移的概念，理解shared_ptr和unique_ptr的区别和适用场景。

不要误用指针，指针带来弹性，但如果没有这个需要，就不要用（比如命名全局变量能搞定，非要整一个全局变量的指针，然后在init里new，在terminate里delete，自找麻烦）。

**## 考虑用std::shared_ptr管理动态分配的对象。**

指针能带来弹性，但不要误用，它的弹性指一方面它能在运行时改变指向，可以用来做多态，另一方面对于不能固定大小的数组可以动态伸缩，但很多时候，我们对固定大小的array，也在init里new/malloc出来，其实没必要，而且会多占用sizeof(void\*)字节，而且增加一层间接访问。

**## size_t到底是个什么？我该用有符号还是无符号整数？**

size_t类型是被设计来保存系统存储器上能保存的对象的最大个数。

32位系统，一个对象最小的单位是一个字节，那2的32次方内存，最多能保存的对象数目就是4G/1字节，正好一个unsigned int能保存下来（typedef unsigned int size_t）。

同样，64位系统，unsigned long是8字节，所以size_t就是unsigned long的类型别名。

对于像索引，位置这样的变量，是用有符号还是无符号呢？像money这样的属性呢？

一句话：要讲道理，用最自然，最顺理成章的类型。比如索引不可能为负用size_t，账户可能欠钱，则money用int。比如：

``` template <class T> class vector``{ ```    ``` T& operator(size_t index) {}``}; ```

标准库给出了最好的示范，因为如果是有符号的话，你需要这样判断

if (index \< 0 || index >= max_num) throw out_of_bound();

而如果是无符号整数，你只需要判断 if (index >= max_num)，你认可吗？

**## 整型一般用int，long就很好，用short，char需要很仔细，要防止溢出**

大多数情况下，用int，long就很好，long一般等于机器字长，long能直接放到寄存器，硬件处理起来速度也更快。

很多时候，我们希望用short，char（8位整型）达到减少结构体大小的目的。但是由于字节对齐，可能并不能真正减少，而且1,2个字节的整型位数太少，一不小心就溢出了，需要特别注意。

所以，除非在db、网络这些对存储大小非常敏感的场合，我们才需要考虑是否以short，char替代int，long。

**# 六、扩展**

**## 了解c++高阶特性**

模板和泛型编程，union，bitfield，指向成员的指针，placement new，显式析构，异常机制，nested class，local class，namespace，多继承、虚继承，volatile，extern "C"等

有些高级特性只有在特定情况下才会被用到，但技多不压身，平时还是需要积累和了解，这样在需求出现时，才能从自己的知识库里拿出工具来对付它。

**## 了解C++新标准**

关注新技术，c++11/14/17、lambda，右值引用，move语义，多线程库等

c++98/03标准到c++11标准的推出历经13年，13年来程序设计语言的思想得到了很大的发展，c++11新标准吸收了很多其他语言的新特性，虽然c++11新标准主要是靠引入新的库来支持新特征，核心语言的变化较少，但新标准还是引入了move语义等核心语法层面的修改，每个CPPer都应该了解新标准。

**## OOD设计原则并不是胡扯**

- 设计模式六大原则（1）：单一职责原则

- 设计模式六大原则（2）：里氏替换原则

- 设计模式六大原则（3）：依赖倒置原则

- 设计模式六大原则（4）：接口隔离原则

- 设计模式六大原则（5）：迪米特法则

- 设计模式六大原则（6）：开闭原则

**## 熟悉常用设计模式，活学活用，不生搬硬套**

神化设计模式和反设计模式，都不是科学的态度，设计模式是软件设计的经验总结，有一定的价值；GOF书上对每一个设计模式，都用专门的段落讲它的应用场景和适用性，限制和缺陷，在正确评估得失的情况下，是鼓励使用的，但显然，你首先需要准确get到她。

![](https://res.wx.qq.com/op_res/NN_GToMiIjsXzgPzF9-74ZzwR3cA9-fv3o9eWo8f5gQWqx71CmGlY8kFxuIxZaG0TB1bFeMCmh1DGN_pWMRg0A)

**帐号已迁移**

公众号

Reads 2523

​

Comment

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/ibLeYZx8Co5JKf72TOeLcba56VknmOtKrMWnS3gyv2Z3RPZ6S28sAtAKSyozOHMDzI8LEkz8ic8eH2v4ZysDq6sQ/300?wx_fmt=png&wxfrom=18)

C语言与CPP编程

16Share7

Comment

Comment

**Comment**

暂无留言
