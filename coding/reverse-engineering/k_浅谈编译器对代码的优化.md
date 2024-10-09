彼岸风 看雪学苑
_2023年01月23日 17:59_ _安徽_
本文为看雪论坛优秀文章
看雪论坛作者ID：彼岸风

在学习 Andorid 逆向的过程中，发现无论是哪种编译器，生成哪个平台的代码，其优化思路在本质上如出一辙，在 Windwos 平台所使用的技巧，在安卓平台仍然适用，不外乎乘法除法计算的优化，swtich 判定树的优化，对 if 条件句的优化，while 循环的优化等等，编写本文的目的也就是为了对已学的知识进行总结。

这里强烈推荐老钱和老张写的《C++反汇编与逆向分析技术揭秘》，本文大部分知识点都将以此书作为参考，我个人是不太喜欢看直播和视屏教程的，因为有些关键知识点可能老师一句话就带过去了，要想回来看还得来回拉进度条，但是书不一样，遇到读不懂的地方可以停下来仔细思考，想回头看也就是翻几页的事情，遇到那种写的特别细的作者，那读起来更是一种享受。

本文列举的代码和汇编只是为了更好的说明思路，并不代表实际代码，好了话不多说，进入正题。

# 优化方向

- 编译速度优化
- 执行速度优化
- 程序体积优化

> 对于 Debug 版程序，编译器为了满足单步调试需求，不会对无意义的代码进行优化。无意义的意思是没有发生传递，没有赋值到内存空间。

# 一  常见的优化类型

## **常量折叠**

更像是预处理，编译器会将所有可预见的值直接写成立即数。

```c
int n = 2 + 3 * 6; 
// 编译器在处理这段代码时会直接将变量赋予立即数+ 
// mov n, 20
```

## **常量传播**

是常量折叠的“进阶版”，编译器会扫描整个代码段，对所有非变量运算直接计算出结果。

```c
int n = 2 + 3 * 6; 
int m = n * 10; 
// mov m, 200
```

## **减少变量**

未使用即是无意义，无意义的代码都会被优化，上述的两个示例，在实际编译中是会直接被优化掉的，因为并未被用于函数传参和其他操作。

编译虽然能通过，但不会产生任何代码，因为没有传递结果，对后续的代码执行不会造成任何影响。

```c
int funtion1() {     int n = 2 + 3 * 6;     int m = n * 10;     return 0; } // 无意义的变量，这个函数被编译为汇编也将只有一句代码 // mov eax, 0    int funtion2() {     int n = 2 + 3 * 6;     int m = n * 10;     return m; } // 有意义的变量，但因为常量传播，也只有一句代码 // mov eax, 200
```

## **分支优化**

对于所有不可达的分支也会直接被裁剪。

```c
if(false) {     printf("you can't find me"); }
```

> 在书中还有更多优化示例，这里不做过多列举，其根本就是以上几种优化方式，无意义的代码将被删除，冗余的代码将会被精简，照着这种思路想就对了。得益于编译器的强大，使得再烂的代码也能保持高效。

# **二  数学计算上对算法的优化**

我将会穿插使用 x86 和 arm 汇编，主要指令都大差不差，理解意义即可。

## **加法**

加法没有任何优化空间，一个 add 指令所需的 cpu 周期本就很短，除了上述的常量折叠外，一般不会对其进行改动。

## **减法**

理论上加法和减法的指令周期是一致的，也不排除有些编译器会将减数转成补码进行相加，遇到补码也能一眼看出来，直接就可以认定这条指令为减法。

## **乘法**

### 变量乘常量

- 常量为2的幂

乘法将会被替换为执行周期更短的移位指令。

```c
int fun(int n) {     return n * 16; } // mov eax, n // shl eax, 4
```

- 常量为非2的幂

因为 thumb 和 x86 指令集的差异，安卓平台上处理的更好一些。

> 我并不推荐你把自己当成编译器，看到算式想着怎么转成汇编，而是推荐记下这种算法，看到计算过程知道怎么转成原式，当然也不追求100%还原，逻辑一致即可。

编译器会对非2的幂进行拆解，例如：

- n * 15 = n * 16 - n = n \<\< 4 - n
- n * 12 = n * 3 * 4 = (n \<\< 1 + n) \<\< 2

```c
int value = n * 15; // rsb.w r0, r1, r1, lsl #4   int value = n * 12; // add.w r0, r1, r1, lsl #1
```

当然 windows 平台也不是一无是处，某些乘法会通过 lea 将两条指令合并成一条。

- n * 4 + 5 = lea edx, \[ecx * 4 + 5\]

```c
printf("%d", n * 4 + 5); // mov ecx, n // lea edx, [ecx * 4 + 5] // push edx
```

至于值为不可拆分的素数，就改用 mul 指令。

### **变量乘变量**

这一步没有什么优化空间，因为都是未知的，只能老老实实用 mul 指令。

```c
int fun(int n, int m) {     return n * m; } // mov eax, n // mov ecx, m // imul ecx
```

## **除法**

在看下面内容之前，不妨再问问自己，真的了解除法吗？除法的本质是什么？\
ok，现在是复习时间，简单总结一下以下两个问题。

- 符号问题

1. 两个无符号整数相除，结果依然是无符号
1. 两个有符号整数相除，结果依然是有符号
1. 混除，参数全被当成无符号计算，结果是无符号

- 取整问题

1. 向下取整 —— floor 函数    存在误差 => ( - a / b ) + ( a / b ) != - ( a / b ) - ( a / b )
1. 向上取整 —— ceil 函数      存在误差 => ( - a / b ) != - ( a / b )
1. 向零取整 —— 截断除法(Truncate)，可以理解为放弃小数部分，只取整数部分，可以在任何情况保持恒等，大部分语言用的都是截断除法

### 除数为无符号数

- 大数（负数）

在无符号中，负数的值是很大的，例如 -8 = 0xFFFFFFF8。
而除以这种大数，只能出现两种情况，1或 0，换个思路来想就可以写成这样：\[被除数\] >= \[除数\] ? 1 : 0
我们来看看 thumb 下是怎么优化的？

```c
UINT value = (UINT)n / -8; // cmn.w r0, #9    ; cmp r0, -9 // it hi // movhi r1, #1    ; n > -9 ? 1 : 0
```

他这里做了一个小小的变形：\[被除数\] > \[除数 - 1\] ? 1 : 0，逻辑上仍然成立。

- 2的幂

简单的移位

```c
UINT value = (UINT)n / 4; // lsrs r1, r0, #2
```

- 非2的幂

接下来就要引入一个非常魔幻的设定，magic number。说来这个魔数，依稀记得早在几年前的知乎上看到过一篇文章，讲的是雷神之锤游戏引擎就使用了这么一个魔数，那时的cpu是非常低效的，而为了避免使用除法这种 cpu 周期偏长的指令，天才的程序员们想出了各种奇技淫巧，其中最为后人津津乐道的就是游戏中对平方根倒数的优化，将计算过程等价替换为加法和移位操作，损失少量的精度来换取绝对的性能。

我们这里的魔数稍有不同，它是用来优化除法的，而且逻辑上也相对容易理解一些，废话不多说，进入正题。

对于普通除法，我们可以得到以下的换算：（x => 被除数变量，c => 除数常量，M => 魔数）
!\[\[Pasted image 20241005112543.png\]\]
!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

假设用 M 代替 2^n / c 这个 Magic 变量，于是有：
!\[\[Pasted image 20241005112547.png\]\]
!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

也就是说，除法将会被转会成 (x * M) >> n 的逻辑进行运算，至于 M 和 n 值怎么来的，我们不关心，这是编译器根据除数算出来的最优值，会尽力保证偏差达到最小，我们要做的是认出魔数和移了多少位，然后根据 m = 2^n/c 公式求得原本的除数 c = 2^n/m

> 公式来源于《C++反汇编与逆向分析技术揭秘》，真的是非常非常的细，书中整个推导过程很完整，很建议各位去仔细研读一遍

以下代码为例：

```c
printf("%u", (unsigned)argc / 3); // mov eax, 0xAAAAAAAB   ; M // mul [argc]            ; edx:eax = argc * M // shr edx, 1            ; edx = argc * M >> 32 >> 1 // push edx
```

!\[\[Pasted image 20241005112610.png\]\]
!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

这个属于总结篇，windows篇可以先看我以前的笔记：_https://note.youdao.com/s/5tc2zdgo_

**看雪ID：彼岸风**
https://bbs.pediy.com/user-home-937323.htm
\*本文由看雪论坛 彼岸风 原创，转载请注明来自看雪社区

**#** **往期推荐**

1.[CVE-2022-21882提权漏洞学习笔记](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458471430&idx=1&sn=6a47d0c5c8f3f6204548e80977ecd059&chksm=b18e7c8c86f9f59a88d9b8e83c8297e0ef65034a73436998ab835531baadaa51f3d630793b95&scene=21#wechat_redirect)

2.[wibu证书 - 初探](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458471429&idx=1&sn=a85188de9b9697fd1b9e708bb8bb1fdb&chksm=b18e7c8f86f9f59933d6cbf0040ed796f06e37b23f17f1ae842eb22257de02338e1a8d751f6b&scene=21#wechat_redirect)

3.[win10 1909逆向之APIC中断和实验](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458471421&idx=2&sn=e83cf7220dc1c4c06a2efc78593e30cc&chksm=b18e7b7786f9f2614ecce34e23be7f71a3d3516766aabda8f25ae41c81ef359a2c245503cf86&scene=21#wechat_redirect)

4.[EMET下EAF机制分析以及模拟实现](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458468723&idx=2&sn=5a830d04185d80e1b6cfa639dc6c6c15&chksm=b18e71f986f9f8ef5b3c2fec51f69751e63a5d6bdbadf43b49728ba05606fc4ac63fda378c92&scene=21#wechat_redirect)

5.[sql注入学习分享](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458468108&idx=1&sn=42c8ec155e13e3882cf4aeb60cdbb982&chksm=b18e0f8686f98690c9792298abb04dd243862ff8effd545dc668c7b1c682aaacf9797d899e97&scene=21#wechat_redirect)

6.[V8 Array.prototype.concat函数出现过的issues和他们的POC们](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458468074&idx=2&sn=06eb27c1649bd4e3a3e43a46a9500add&chksm=b18e0e6086f9877644ba0de33658232f99213d1b1b074342260031cb529c1b7ad1b89b2e0204&scene=21#wechat_redirect)

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**球分享**

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**球点赞**

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**球在看**

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

点击“阅读原文”，了解更多！

Read more

Reads 5187

​

Comment

**留言 5**

- 井江💓开儿

  浙江2023年1月23日

  Like3

  感觉所谓优化余地的产生都是由于从高级语言层转换到相关平台架构汇编层实现理解的不同产生的

- unituniverse

  福建2023年1月24日

  Like1

  文章还行但是想来更正下：其实吧常量表达式优化根本不是编译器那边自己打算做的而是C++标准强制要求的。这点对C可能影响不大但对C++来说就太重要了，直接决定了这个表达式可以出现在任何只（划重点）允许常量出现的地方，比如case、模板参数、枚举常量定义值、数组长度定义

- 一卡车香菜

  上海2023年1月28日

  Like

  漂亮！之前看到gcc编译的除法和取余，居然先imul一个数，然后移位…用计算器一算结果一样，大吃一惊💪

- Gww

  江苏2023年1月23日

  Like

  写的太好了

- 余老师

  重庆2023年1月23日

  Like

  这个文章写的真好，点赞

已无更多数据

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/1UG7KPNHN8EGLfh77kFmnicd9WOic2ibvhCibFdB4bL4srJCgo2wnvdoXLxpIvAkfCmmcptXZB0qKWMoIP8iaibYN2FA/300?wx_fmt=png&wxfrom=18)

看雪学苑

3428

5

Comment

**留言 5**

- 井江💓开儿

  浙江2023年1月23日

  Like3

  感觉所谓优化余地的产生都是由于从高级语言层转换到相关平台架构汇编层实现理解的不同产生的

- unituniverse

  福建2023年1月24日

  Like1

  文章还行但是想来更正下：其实吧常量表达式优化根本不是编译器那边自己打算做的而是C++标准强制要求的。这点对C可能影响不大但对C++来说就太重要了，直接决定了这个表达式可以出现在任何只（划重点）允许常量出现的地方，比如case、模板参数、枚举常量定义值、数组长度定义

- 一卡车香菜

  上海2023年1月28日

  Like

  漂亮！之前看到gcc编译的除法和取余，居然先imul一个数，然后移位…用计算器一算结果一样，大吃一惊💪

- Gww

  江苏2023年1月23日

  Like

  写的太好了

- 余老师

  重庆2023年1月23日

  Like

  这个文章写的真好，点赞

已无更多数据
