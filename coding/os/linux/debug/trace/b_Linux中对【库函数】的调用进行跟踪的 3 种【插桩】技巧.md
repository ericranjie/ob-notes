程序喵大人
_2022年01月15日 19:15_
编者荐语：
非常不错的文章，分享给大家！
以下文章来源于IOT物联网小镇 ，作者道哥

目录

- 什么是插桩？
- 插桩示例代码分析
- 在编译阶段插桩
- 链接阶段插桩
- 执行阶段插桩

## 什么是插桩？

在稍微具有一点规模的代码中(C 语言)，调用第三方动态库中的函数来完成一些功能，是很常见的工作场景。

假设现在有一项任务：需要在调用某个动态库中的某个函数的之前和之后，做一些额外的处理工作。

这样的需求一般称作：插桩，也就是对于一个指定的目标函数，新建一个包装函数，来完成一些额外的功能。

![图片](https://mmbiz.qpic.cn/mmbiz_png/WC13ibsIvG3b4qYgQZaLO8MwmCicJ1iaL9W5oMY48kBRY1qItvULrCVycPiaib6eibEvOE82lzNfM48BQ11JBfB1QkXA/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

在包装函数中去调用真正的目标函数，但是在调用之前或者之后，可以做一些额外的事情。

比如：统计函数的调用次数、验证函数的输入参数是否合法等等。

关于程序插桩的官方定义，可以看一下【百度百科】中的描述：

> 1. 程序插桩，最早是由J.C. Huang 教授提出的。
> 1. 它是在保证被测程序原有逻辑完整性的基础上在程序中插入一些探针（又称为“探测仪”，本质上就是进行信息采集的代码段，可以是赋值语句或采集覆盖信息的函数调用）。
> 1. 通过探针的执行并抛出程序运行的特征数据，通过对这些数据的分析，可以获得程序的控制流和数据流信息，进而得到逻辑覆盖等动态信息，从而实现测试目的的方法。
> 1. 根据探针插入的时间可以分为目标代码插桩和源代码插桩。

这篇文章，我们就一起讨论一下：在 Linux 环境下的 C 语言开发中，可以通过哪些方法来实现插桩功能。

## 插桩示例代码分析

示例代码很简单：

```cpp
├── app.c   └── lib       ├── rd3.h       └── librd3.so   
```

假设动态库`librd3.so`是由第三方提供的，里面有一个函数：`int rd3_func(int, int);`。

`// lib/rd3.h      #ifndef _RD3_H_   #define _RD3_H_   extern int rd3_func(int, int);   #endif   `

在应用程序`app.c`中，调用了动态库中的这个函数:

![图片](https://mmbiz.qpic.cn/mmbiz_png/WC13ibsIvG3b4qYgQZaLO8MwmCicJ1iaL9WlNNwf1amd8XbUTgpgnkZ4I83Nv28Rf1icaGcpVW2wGTLxtgenPwHr7Q/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

`app.c`代码如下：

`#include <stdio.h>   #include <stdlib.h>   #include "rd3.h"      int main(int argc, char *argv[])   {       int result = rd3_func(1, 1);       printf("result = %d \n", result);       return 0;   }   `

编译：

`$ gcc -o app app.c -I./lib -L./lib -lrd3 -Wl,--rpath=./lib   `

> 1. -L./lib: 指定编译时，在 lib 目录下搜寻库文件。
>
> 1. -Wl,--rpath=./lib: 指定执行时，在 lib 目录下搜寻库文件。

生成可执行程序：`app`，执行：

`$ ./app   result = 3   `

示例代码足够简单了，称得上是`helloworld`的兄弟版本！

## 在编译阶段插桩

对函数进行插桩，基本要求是：不应该对原来的文件(app.c)进行额外的修改。

由于`app.c`文件中，已经`include "rd3.h"`了，并且调用了其中的`rd3_func(int, int)`函数。

所以我们需要新建一个假的 "rd3.h" 提供给`app.c`，并且要把函数`rd3_func(int, int)`"重导向"到一个包装函数，然后在包装函数中去调用真正的目标函数，如下图所示：
!\[\[Pasted image 20240928174711.png\]\]

"重导向"函数：可以使用宏来实现。

包装函数：新建一个`C`文件，在这个文件中，需要 `#include "lib/rd3.h"`，然后调用真正的目标文件。

完整的文件结构如下：

```cpp
├── app.c   ├── lib   │   ├── librd3.so   │   └── rd3.h   ├── rd3.h   └── rd3_wrap.c   
```

最后两个文件是新建的：`rd3.h`, `rd3_wrap.c`，它们的内容如下：

```cpp
// rd3.h      
#ifndef _LIB_WRAP_H_   
#define _LIB_WRAP_H_      // 函数“重导向”，这样的话 app.c 中才能调用 wrap_rd3_func   
#define rd3_func(a, b)   wrap_rd3_func(a, b)      // 函数声明   
extern int wrap_rd3_func(int, int);      #endif   

// rd3_wrap.c      
#include <stdio.h>   
#include <stdlib.h>      // 真正的目标函数   
#include "lib/rd3.h"      // 包装函数，被 app.c 调用   
int wrap_rd3_func(int a, int b)   {       // 在调用目标函数之前，做一些处理       
printf("before call rd3_func. do something... \n");              // 调用目标函数       int c = rd3_func(a, b);              // 在调用目标函数之后，做一些处理       printf("after call rd3_func. do something... \n");              return c;   }   
```

让`app.c 和 rd3_wrap.c`一起编译：

`$ gcc -I./ -L./lib -Wl,--rpath=./lib -o app app.c rd3_wrap.c -lrd3   `

头文件的搜索路径不能错：必须在当前目录下搜索`rd3.h`，这样的话，`app.c`中的`#include "rd3.h"` 找到的才是我们新增的那个头文件 `rd3.h`。

所以在编译指令中，第一个选项就是 -I./，表示在当前目录下搜寻头文件。

另外，由于在`rd3_wrap.c`文件中，使用`#include "lib/rd3.h"`来包含库中的头文件，因此在编译指令中，就不需要指定到`lib` 目录下去查找头文件了。

编译得到可执行程序`app`，执行一下：

`$ ./app    before call rd3_func. do something...    after call rd3_func. do something...    result = 3   `

完美！

## 链接阶段插桩

`Linux` 系统中的链接器功能是非常强大的，它提供了一个选项：`--wrap f`，可以在链接阶段进行插桩。

这个选项的作用是：告诉链接器，遇到`f`符号时解析成`__wrap_f`，在遇到`__real_f`符号时解析成`f`，正好是一对！

我们就可以利用这个属性，新建一个文件`rd3_wrap.c`，并且定义一个函数`__wrap_rd3_func(int, int)`，在这个函数中去调用`__real_rd3_func`函数。

只要在编译选项中加上`-Wl,--wrap,rd3_func`, 编译器就会：

> 1. 把 app.c 中的 rd3_func 符号，解析成 \_\_wrap_rd3_func，从而调用包装函数;
>
> 1. 把 rd3_wrap.c 中的 \_\_real_rd3_func 符号，解析成 rd3_func，从而调用真正的函数。

!\[\[Pasted image 20240928174727.png\]\]
!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

这几个符号的转换，是由链接器自动完成的！

按照这个思路，一起来测试一下。

文件目录结构如下：

`.   ├── app.c   ├── lib   │   ├── librd3.so   │   └── rd3.h   ├── rd3_wrap.c   └── rd3_wrap.h   `

`rd3_wrap.h`是被`app.c`引用的，内容如下：

`#ifndef _RD3_WRAP_H_   #define _RD3_WRAP_H_   extern int __wrap_rd3_func(int, int);   #endif   `

`rd3_wrap.c`的内容如下：

`#include <stdio.h>   #include <stdlib.h>      #include "rd3_wrap.h"      // 这里不能直接饮用 lib/rd3.h 中的函数了，而要由链接器来完成解析。   extern int __real_rd3_func(int, int);      // 包装函数   int __wrap_rd3_func(int a, int b)   {       // 在调用目标函数之前，做一些处理       printf("before call rd3_func. do something... \n");              // 调用目标函数，链接器会解析成 rd3_func。       int c = __real_rd3_func(a, b);              // 在调用目标函数之后，做一些处理       printf("after call rd3_func. do something... \n");              return c;   }   `

`rd3_wrap.c`中，不能直接去 `include "rd3.h"`，因为`lib/rd3.h`中的函数声明是`int rd3_func(int, int);`，没有`__real`前缀。

编译一下：

`$ gcc -I./lib -L./lib -Wl,--rpath=./lib -Wl,--wrap,rd3_func -o app app.c rd3_wrap.c -lrd3   `

注意：这里的头文件搜索路径仍然设置为`-I./lib`，是因为`app.c`中`include`了这个头文件。

得到可执行程序`app`，执行：

`$ ./app   before call rd3_func. do something...    before call rd3_func. do something...    result = 3   `

完美！

## 执行阶段插桩

在编译阶段插桩，新建的文件`rd3_wrap.c`是与`app.c`一起编译的，其中的包装函数名是`wrap_rd3_func`。

`app.c`中通过一个宏定义实现函数的"重导向"：`rd3_func --> wrap_rd3_func`。

我们还可以直接"霸王硬上弓"：在新建的文件`rd3_wrap.c`中，直接定义`rd3_func`函数。

然后在这个函数中通过`dlopen, dlsym`系列函数来动态的打开真正的动态库，查找其中的目标文件，然后调用真正的目标函数。

当然了，这样的话在编译`app.c`时，就不能连接`lib/librd3.so`文件了。

按照这个思路继续实践！

文件目录结构如下：

`├── app.c   ├── lib   │   ├── librd3.so   │   └── rd3.h   └── rd3_wrap.c   `

`rd3_wrap.c`文件的内容如下(一些错误检查就暂时忽略了)：

`#include <stdio.h>   #include <stdlib.h>   #include <dlfcn.h>      // 库的头文件   #include "rd3.h"      // 与目标函数签名一致的函数类型   typedef int (*pFunc)(int, int);      int rd3_func(int a, int b)   {       printf("before call rd3_func. do something... \n");              //打开动态链接库       void *handle = dlopen("./lib/librd3.so", RTLD_NOW);              // 查找库中的目标函数       pFunc pf = dlsym(handle, "rd3_func");              // 调用目标函数       int c = pf(a, b);              // 关闭动态库句柄       dlclose(handle);              printf("after call rd3_func. do something... \n");       return c;   }   `

编译包装的动态库：

`$ gcc -shared -fPIC -I./lib -o librd3_wrap.so rd3_wrap.c   `

得到包装的动态库： `librd3_wrap.so`。

编译可执行程序，需要链接包装库 `librd3_wrap.so`：

`$ gcc -I./lib -L./ -o app app.c -lrd3_wrap -ldl   `

得到可执行程序`app`，执行：

`$ ./app    before call rd3_func. do something...    after call rd3_func. do something...    result = 3   `

完美！

______________________________________________________________________

往期推荐

探索CPU的调度原理

\](https://mp.weixin.qq.com/s?\_\_biz=MzkyODE5NjU2Mw==&mid=2247491534&idx=1&sn=f17baa3b00a30a410e629b6a9701bcf5&chksm=c21d2d72f56aa464adea2cbcc531a24fcd9aa9982d9b79f6d247425248284a3b6ccc8643d23b&scene=21#wechat_redirect)

\[

防御性编程技巧

\](https://mp.weixin.qq.com/s?\_\_biz=MzkyODE5NjU2Mw==&mid=2247491525&idx=1&sn=ea45c5caf3244858a92a243a53718538&chksm=c21d2d79f56aa46f5476dd30b7d54df9532679889b5f0430c5def19a49da9142fb93a99b20f5&scene=21#wechat_redirect)

\[

C++的全链路追踪方案，稍微有点高端

\](https://mp.weixin.qq.com/s?\_\_biz=MzkyODE5NjU2Mw==&mid=2247491517&idx=1&sn=7c23fb215c7b71b93617f14eae70bdac&chksm=c21d2d01f56aa417021b4e6ef44393ef1c79d2ce1ed3cb2bc068071d6cde411088519d081b0f&scene=21#wechat_redirect)

\[

多线程程序中操作的原子性

\](https://mp.weixin.qq.com/s?\_\_biz=MzkyODE5NjU2Mw==&mid=2247491516&idx=1&sn=252f4a15253c86eeb61667a8c82fb62b&chksm=c21d2d00f56aa41663c0d58245998a8511be91e1cb002942b54a329a5f7711867734ff71cba7&scene=21#wechat_redirect)

\[

C++反射TS初探

\](https://mp.weixin.qq.com/s?\_\_biz=MzkyODE5NjU2Mw==&mid=2247491510&idx=1&sn=b7fadf0c2daf6f73ac991c758d418359&chksm=c21d2d0af56aa41c1d73657f8d2ce3ede0dc6789dd5b0855615992c9770354d44fe31f160002&scene=21#wechat_redirect)

\[

喵哥吐血整理：软件开发的51条建议

\](https://mp.weixin.qq.com/s?\_\_biz=MzkyODE5NjU2Mw==&mid=2247491489&idx=1&sn=5a5e68b50e67bf1aaad7fa9b690ae726&chksm=c21d2d1df56aa40b51b3e29a53a783740d3d3eb099b138e2fc32ca75faf9b3f5245f95fe1e79&scene=21#wechat_redirect)

\[

为什么公司宁可高薪招一个新员工，也不愿意给老员工涨一点工资？

\](https://mp.weixin.qq.com/s?\_\_biz=MzkyODE5NjU2Mw==&mid=2247491457&idx=1&sn=7433c5ec4f7829eb0f2f01bc15351deb&chksm=c21d2d3df56aa42b68067d61579d5b75722fb03946c85faa804a7d8f2e20854a5ccee92bbc65&scene=21#wechat_redirect)

\[

函数返回值的行业潜规则

\](https://mp.weixin.qq.com/s?\_\_biz=MzkyODE5NjU2Mw==&mid=2247491440&idx=1&sn=0baf7e7e28d0c4c0f6ca6b14fe0df2a7&chksm=c21d2dccf56aa4daa84dcc87f541efae82f575abfe474815d3764ace918f9638267ddac8801a&scene=21#wechat_redirect)

\[

模版定义一定要写在头文件中吗?

\](https://mp.weixin.qq.com/s?\_\_biz=MzkyODE5NjU2Mw==&mid=2247491391&idx=1&sn=149e8289e0545d5f7513661aef792f9c&chksm=c21d2d83f56aa49509c87ea188089e86d2d779de1bdd31a5d3b72eccf4c6d0ca21c9b1a54d86&scene=21#wechat_redirect)

\[

为什么建议少用if语句！

\](https://mp.weixin.qq.com/s?\_\_biz=MzkyODE5NjU2Mw==&mid=2247491362&idx=1&sn=0cafab8395443227af0879160dcf90a0&chksm=c21d2d9ef56aa4881bc639ec5025ec5ff2bc12db424f9b7a2266505cb66e72311f48de86fa99&scene=21#wechat_redirect)

\[

四万字长文，这是我见过最好的模板元编程文章！

\](https://mp.weixin.qq.com/s?\_\_biz=MzkyODE5NjU2Mw==&mid=2247491328&idx=1&sn=a9e4147a1ffe5de9c8a7f0c8d5377405&chksm=c21d2dbcf56aa4aa5d5e60e3a4ecece847cbcb777a97f437010b1300c02ab36f9ecc01dca65d&scene=21#wechat_redirect)

阅读 2148

​

写留言

**留言 3**

- 应曙

  2022年1月16日

  赞1

  插桩?怎么感觉和勾子差不多啊，是吗大佬

  程序喵大人

  作者2022年1月16日

  赞1

  这个和勾子貌似还不太一样

- 焦璐；1560181

  2022年1月15日

  赞

  好

已无更多数据

[](javacript:;)

![](http://mmbiz.qpic.cn/sz_mmbiz_png/Av3j9GXyMmGmg0HNK7fR6Peq3uf9pUoN5Qt0s0ia8IH97pvIiaADsEvy9nm3YnR5bmA13O8ic0JJOicQO0IboKGdZg/300?wx_fmt=png&wxfrom=18)

程序喵大人

27分享12

3

写留言

**留言 3**

- 应曙

  2022年1月16日

  赞1

  插桩?怎么感觉和勾子差不多啊，是吗大佬

  程序喵大人

  作者2022年1月16日

  赞1

  这个和勾子貌似还不太一样

- 焦璐；1560181

  2022年1月15日

  赞

  好

已无更多数据
