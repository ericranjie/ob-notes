# 

CPP开发者

_2022年05月24日 11:50_ _浙江_

以下文章来源于IOT物联网小镇 ，作者道哥

\[

![](http://wx.qlogo.cn/mmhead/Q3auHgzwzM4Ity70WLCicqljHPMcjBJicfmKibafuotbvsRqaZIic6N5dQ/0)

**IOT物联网小镇**.

深入的思考 + 直白的文字 + 实用的项目经验，这是我能为您提供的、最基本的知识服务！

\](https://mp.weixin.qq.com/s?\_\_biz=MzAxNDI5NzEzNg==&mid=2651171156&idx=1&sn=383ce87003f95134997a4dc5164679f3&chksm=8064780bb713f11d45268afd5f5ea247a13141eb895b53e15344da89371364cfead43c670bfd&mpshare=1&scene=24&srcid=0524lDx7d5FLLMi2qS47ZcDN&sharer_sharetime=1653405692228&sharer_shareid=8397e53ca255d0bca170c6327d62b9af&key=daf9bdc5abc4e8d0544ee05ee7ad39b6125cbd964f9c753c17bdcf52aa5f7cf8a98e2fbd292d7eb0b9f18c6fedcd5fb9901e2ecf63003a66fc9ba2a83beef5a29591a0979a9ffb9136d975f5903df07945251e807cce9eae34aac23d9da3a1abc010a183cd8a94c9ba6e42828b219f5b4b4b2476868ea01d0dbfc40808bb9013&ascene=0&uin=MTEwNTU1MjgwMw%3D%3D&devicetype=Windows+11+x64&version=63090b19&lang=zh_CN&countrycode=CN&exportkey=n_ChQIAhIQrQXN0anwbyVK7pLhefglyxLmAQIE97dBBAEAAAAAACb1CAlh2GcAAAAOpnltbLcz9gKNyK89dVj0DBSXH12w6hecFI9oUJcyWJFlMt0mU8RT%2F%2BT30dp7WrroZ29fB4GnG0dz4ZfR6Ui5ghUxU2QScAPAEAN7NzM5sAhKMbFgy4ZNuk4L1%2B0JZXo%2BeBze24%2BhXaq0IOexC9%2FN3gP5MKZBPWw0Kl%2Bh1YtyhH0pN6ajc%2BtPYccJK7b%2FSmuLU2bbAoyZirxS9xdhw5R97Tetwn4yoFvljYcwWPviC40T29FZmVbmeLkCWO7VMGTuYEW94ke1dpAb08E9j2nn&acctmode=0&pass_ticket=sKRQ4aDPoxsDBx7BpE87KW3uGc%2BbA2HHwEdO66FzAcQW38Fr0v2OH0R5uLkBzdd0&wx_header=1&fasttmpl_type=0&fasttmpl_fullversion=7351805-zh_CN-zip&fasttmpl_flag=1#)

目录

- 动态链接要解决什么问题？

- 矛盾：代码段不可写

- 解决矛盾：增加一层间接性

- 示例代码

- b.c

- a.c

- main.c

- 编译成动态链接库

- 动态库的依赖关系

- 动态库的加载过程

- 动态链接器加载动态库

- 动态库的加载地址分析

- 符号重定位

- 全局符号表

- 全局偏移表GOT

- liba.so动态库文件的布局

- liba.so动态库的虚拟地址

- GOT表的内部结构

- 反汇编liba.so代码

在上一篇文章中，我们一起学习了`Linux`系统中 `GCC`编译器在编译可执行程序时，静态链接过程中是如何进行符号重定位的。[GCC 链接过程中的【重定位】过程分析](http://mp.weixin.qq.com/s?__biz=MzAxNDI5NzEzNg==&mid=2651170773&idx=1&sn=278b893d525205697874563dd906b78e&chksm=8064768ab713ff9c85a61338393dc5aac43506a9b55664fa50645399d907b938f09dc7fbf49c&scene=21#wechat_redirect)

为了完整性，我们这篇文章来一起探索一下：动态链接过程中是如何进行符号重定位的。

老样子，文中使用大量的【代码+图片】的方式，来真实的感受一下实际的内存模型。

文中使用了大量的图片，建议您在电脑上阅读此文。

> 关于为什么使用动态链接，这里就不展开讨论了，无非就几点：
>
> 1. 节省物理内存;
>
> 1. 可以动态更新;

## 动态链接要解决什么问题？

静态链接得到的可执行程序，被操作系统加载之后就可以执行执行。

因为在链接的时候，链接器已经把所有目标文件中的代码、数据等`Section`，都组装到可执行文件中了。

并且把代码中所有使用的外部符号(变量、函数)，都进行了重定位(即：把变量、函数的地址，都填写到代码段中需要重定位的地方)，因此可执行程序在执行的时候，不依赖于其它的外部模块即可运行。

详细的静态链接过程，请参考上一篇文章：[GCC 链接过程中的【重定位】过程分析。](http://mp.weixin.qq.com/s?__biz=MzAxNDI5NzEzNg==&mid=2651170773&idx=1&sn=278b893d525205697874563dd906b78e&chksm=8064768ab713ff9c85a61338393dc5aac43506a9b55664fa50645399d907b938f09dc7fbf49c&scene=21#wechat_redirect)

也就是说：符号重定位的过程，是直接对可执行文件进行修改。

但是对于动态链接来说，在编译阶段，仅仅是在可执行文件或者动态库中记录了一些必要的信息。

真正的重定位过程，是在这个时间点来完成的：可执行程序、动态库被加载之后，调用可执行程序的入口函数之前。

只有当所有需要被重定位的符号被解决了之后，才能开始执行程序。

既然也是重定位，与静态链接过程一样：也需要把符号的目标地址填写到代码段中需要重定位的地方。

#### 矛盾：代码段不可写

问题来了!

我们知道，在现代操作系统中，对于内存的访问是有权限控制的，一般来说：

> 代码段：可读、可执行;
>
> 数据段：可读、可写;

如果进行符号重定位，就需要对代码进行修改(填写符号的地址)，但是代码段又没有可写的权限，这是一个矛盾！

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

解决这个矛盾的方案，就是`Linux`系统中动态链接器的核心工作！

#### 解决矛盾：增加一层间接性

`David Wheeler`有一句名言:“计算机科学中的大多数问题，都可以通过增加一层间接性来解决。”

解决动态链接中的代码重定位问题，同样也可以通过增加一层间接性来解决。

既然代码段在被加载到内存中之后不可写，但是数据段是可写的。

在代码段中引用的外部符号，可以在数据段中增加一个跳板：让代码段先引用数据段中的内容，然后在重定位时，把外部符号的地址填写到数据段中对应的位置，不就解决这个矛盾了吗？!

如下图所示：

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

理解了上图的解决思路，基本上就理解了动态链接过程中重定位的核心思想。

## 示例代码

我们需要`3`个源文件来讨论动态链接中重定位的过程：`main.c`、`a.c`、`b.c`，其中的`a.c`和`b.c`被编译成动态库，然后`main.c`与这两个动态库一起动态链接成可执行程序。

它们之间的依赖关系是：

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

#### b.c

代码如下：

```
#include <stdio.h>int b = 30;void func_b(void){    printf("in func_b. b = %d \n", b);}
```

代码说明：

> 定义一个全局变量和一个全局函数，被 a.c 调用。

#### a.c

代码如下(稍微复杂一些，主要是为了探索：不同类型的符号如何处理重定位)：

```
#include <stdio.h>// 内部定义【静态】全局变量static int a1 = 10;// 内部定义【非静态】全局变量int a2 = 20;// 声明外部变量extern int b;// 声明外部函数extern void func_b(void);// 内部定义的【静态】函数static void func_a2(void){    printf("in func_a2 \n");}// 内部定义的【非静态】函数void func_a3(void){    printf("in func_a3 \n");}// 被 main 调用void func_a1(void){    printf("in func_a1 \n");    // 操作内部变量    a1 = 11;    a2 = 21;    // 操作外部变量    b  = 31;    // 调用内部函数    func_a2();    func_a3();    // 调用外部函数    func_b();}
```

代码说明：

> 1. 定义了 2 个全局变量：一个静态，一个非静态；
>
> 1. 定义了 3 个函数:
>
> > `func_a2`是静态函数，只能在本文件中调用;
>
> > `func_a1`和`func_a3`是全局函数，可以被外部调用;
>
> 3. 在 main.c 中会调用`func_a1`。

#### main.c

代码如下：

```
#include <stdio.h>#include <unistd.h>#include <dlfcn.h>// 声明外部变量extern int a2;extern void func_a1();typedef void (*pfunc)(void);int main(void){    printf("in main \n");    // 打印此进程的全局符号表    void *handle = dlopen(0, RTLD_NOW);    if (NULL == handle)    {        printf("dlopen failed! \n");        return -1;    }    printf("\n------------ main ---------------\n");    // 打印 main 中变量符号的地址    pfunc addr_main = dlsym(handle, "main");    if (NULL != addr_main)        printf("addr_main = 0x%x \n", (unsigned int)addr_main);    else        printf("get address of main failed! \n");    printf("\n------------ liba.so ---------------\n");    // 打印 liba.so 中变量符号的地址    unsigned int *addr_a1 = dlsym(handle, "a1");    if (NULL != addr_a1)        printf("addr_a1 = 0x%x \n", *addr_a1);    else        printf("get address of a1 failed! \n");    unsigned int *addr_a2 = dlsym(handle, "a2");    if (NULL != addr_a2)        printf("addr_a2 = 0x%x \n", *addr_a2);    else        printf("get address of a2 failed! \n");    // 打印 liba.so 中函数符号的地址    pfunc addr_func_a1 = dlsym(handle, "func_a1");    if (NULL != addr_func_a1)        printf("addr_func_a1 = 0x%x \n", (unsigned int)addr_func_a1);    else        printf("get address of func_a1 failed! \n");    pfunc addr_func_a2 = dlsym(handle, "func_a2");    if (NULL != addr_func_a2)        printf("addr_func_a2 = 0x%x \n", (unsigned int)addr_func_a2);    else        printf("get address of func_a2 failed! \n");    pfunc addr_func_a3 = dlsym(handle, "func_a3");    if (NULL != addr_func_a3)        printf("addr_func_a3 = 0x%x \n", (unsigned int)addr_func_a3);    else        printf("get address of func_a3 failed! \n");    printf("\n------------ libb.so ---------------\n");    // 打印 libb.so 中变量符号的地址    unsigned int *addr_b = dlsym(handle, "b");    if (NULL != addr_b)        printf("addr_b = 0x%x \n", *addr_b);    else        printf("get address of b failed! \n");    // 打印 libb.so 中函数符号的地址    pfunc addr_func_b = dlsym(handle, "func_b");    if (NULL != addr_func_b)        printf("addr_func_b = 0x%x \n", (unsigned int)addr_func_b);    else        printf("get address of func_b failed! \n");    dlclose(handle);    // 操作外部变量    a2 = 100;    // 调用外部函数    func_a1();    // 为了让进程不退出，方便查看虚拟空间中的地址信息    while(1) sleep(5);    return 0;}
```

> 纠正：代码中本来是想打印变量的地址的，但是不小心加上了 \*，变成了打印变量值。最后检查的时候才发现，所以就懒得再去修改了。

代码说明：

> 1. 利用 dlopen 函数(第一个参数传入 NULL)，来打印此进程中的一些符号信息(变量和函数);
>
> 1. 赋值给 liba.so 中的变量 a2，然后调用 liba.so 中的 func_a1 函数;

#### 编译成动态链接库

把以上几个源文件编译成动态库以及可执行程序：

```
$ gcc -m32 -fPIC --shared b.c -o libb.so$ gcc -m32 -fPIC --shared a.c -o liba.so -lb -L./$ gcc -m32 -fPIC main.c -o main -ldl -la -lb -L./
```

有几点内容说明一下：

> 1. -fPIC 参数意思是：生成位置无关代码(Position Independent Code)，这也是动态链接中的关键;
>
> 1. 既然动态库是在运行时加载，那为什么在编译的时候还需要指明?
>
> > 因为在编译的时候，需要知道每一个动态库中提供了哪些符号。Windows 中的动态库的显性的导出和导入标识，更能体现这个概念(\_\_declspec(dllexport), \_\_declspec(dllimport))。

此时，就得到了如下几个文件：

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

## 动态库的依赖关系

对于静态链接的可执行程序来说，被操作系统加载之后，可以认为直接从可执行程序的入口函数开始(也就是`ELF`文件头中指定的`e_entry`这个地址)，执行其中的指令码。

但是对于动态链接的程序来说，在执行入口函数的指令之前，必须把该程序所依赖的动态库加载到内存中，然后才能开始执行。

对于我们的实例代码来说：`main`程序依赖于`liba.so`库，而`liba.so`库又依赖于`libb.so`库。

可以用`ldd`工具来分别看一下动态库之间的依赖关系：

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

可以看出：

> 1. 在 liba.so 动态库中，记录了信息：依赖于 libb.so;
>
> 1. 在 main 可执行文件中，记录了信息：依赖于 liba.so, libb.so;

也可以使用另一个工具`patchelf`来查看一个可执行程序或者动态库，依赖于其他哪些模块。例如：

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

那么，动态库的加载是由谁来完成的呢？动态链接器！

## 动态库的加载过程

#### 动态链接器加载动态库

当执行`main`程序的时候，操作系统首先把`main`加载到内存，然后通过`.interp`段信息来查看该文件依赖哪些动态库：

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

上图中的字符串`/lib/ld-linux.so.2`，就表示`main`依赖动态链接库。

`ld-linux.so.2`也是一个动态链接库，在大部分情况下动态链接库已经被加载到内存中了(动态链接库就是为了共享)，操作系统此时只需要把动态链接库所在的物理内存，映射到 `main`进程的虚拟地址空间中就可以了，然后再把控制权交给动态链接器。

动态链接器发现：`main`依赖`liba.so`，于是它就在虚拟地址空间中找一块能放得下`liba.so`的空闲空间，然后把`liba.so`中需要加载到内存中的代码段、数据段都加载进来。

当然，在加载`liba.so`时，又会发现它依赖`libb.so`，于是又把在虚拟地址空间中找一块能放得下`libb.so`的空闲空间，把`libb.so`中的代码段、数据段等加载到内存中，示意图如下所示：

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

> 动态链接器自身也是一个动态库，而且是一个特殊的动态库：它不依赖于其他的任何动态库，因为当它被加载的时候，没有人帮它去加载依赖的动态库，否则就形成鸡生蛋、蛋生鸡的问题了。

#### 动态库的加载地址

一个进程在运行时的实际加载地址(或者说虚拟内存区域)，可以通过指令：`$ cat /proc/[进程的 pid]/maps` 读取出来。

例如：我的虚拟机中执行`main`程序时，看到的地址信息是：

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

黄色部分分别是：`main`, `liba.so`, `libb.so` 这`3`个模块的加载信息。

另外，还可以看到`c`库(`libc-2.23.so`)、动态链接器(`ld-2.23.so`)以及动态加载库`libdl-2.23.so`的虚拟地址区域，布局如下：

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

> 可以看出出来:`main`可执行程序是位于低地址，所有的动态库都位于`4G`内存空间的最后`1G`空间中。

还有另外一个指令也很好用 `$ pmap [进程的 pid]`，也可以打印出每个模块的内存地址：

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

## 符号重定位

#### 全局符号表

在之前的静态链接中学习过，链接器在扫描每一个目标文件(`.o`文件)的时候，会把每个目标文件中的符号提取出来，构成一个全局符号表。

然后在第二遍扫描的时候，查看每个目标文件中需要重定位的符号，然后在全局符号表中查找该符号被安排在什么地址，然后把这个地址填写到引用的地方，这就是静态链接时的重定位。

但是动态链接过程中的重定位，与静态链接的处理方式差别就大很多了，因为每个符号的地址只有在运行的时候才能知道它们的地址。

例如：`liba.so`引用了`libb.so`中的变量和函数，而`libb.so`中的这两个符号被加载到什么位置，直到`main`程序准备执行的时候，才能被链接器加载到内存中的某个随机的位置。

也就是说：动态链接器知道每个动态库中的代码段、数据段被加载的内存地址，因此动态链接器也会维护一个全局符号表，其中存放着每一个动态库中导出的符号以及它们的内存地址信息。

在示例代码`main.c`函数中，我们通过`dlopen`返回的句柄来打印进程中的一些全局符号的地址信息，输出内容如下：

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

> 上文已经纠错过：本来是想打印变量的地址信息，但是 printf 语句中不小心加上了型号，变成了打印变量值。

可以看到：在全局符号表中，没有找到`liba.so`中的变量`a1`和函数`func_a2`这两个符号，因为它俩都是`static`类型的，在编译成动态库的时候，没有导出到符号表中。

既然提到了符号表，就来看看这 3 个`ELF`文件中的动态符号表信息：

> 1. 动态链接库中保护两个符号表：.dynsym(动态符号表: 表示模块中符号的导出、导入关系) 和 .symtab(符号表: 表示模块中的所有符号);
>
> > .symtab 中包含了 .dynsym;
>
> 2. 由于图片太大，这里只贴出 .dynsym 动态符号表。

绿色矩形框前面的`Ndx`列是数字，表示该符号位于当前文件的哪一个段中(即：段索引);

红色矩形框前面的`Ndx`列是`UND`，表示这个符号没有找到，是一个外部符号(需要重定位);

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

#### 全局偏移表GOT

在我们的示例代码中，`liba.so`是比较特殊的，它既被`main`可执行程序所依赖，又依赖于`libb.so`。

而且，在`liba.so`中，定义了静态、动态的全局变量和函数，可以很好的概况很多种情况，因此这部分内容就主要来分析`liba.so`这个动态库。

前文说过：代码重定位需要修改代码段中的符号引用，而代码段被加载到内存中又没有可写的权限，动态链接解决这个矛盾的方案是：增加一层间接性。

例如：`liba.so`的代码中引用了`libb.so`中的变量`b`，在`liba.so`的代码段，并不是在引用的地方直接指向`libb.so`数据段中变量`b`的地址，而是指向了`liba.so`自己的数据段中的某个位置，在重定位阶段，链接器再把`libb.so`中变量`b`的地址填写到这个位置。

因为`liba.so`自己的代码段和数据段位置是相对固定的，这样的话，`liba.so`的代码段被加载到内存之后，就再也不用修改了。

而数据段中这个间接跳转的位置，就称作：全局偏移表(`GOT: Global Offset Table`)。

划重点：

`liba.so`的代码段中引用了`libb.so`中的符号`b`，既然`b`的地址需要在重定位时才能确定，那么就在数据段中开辟一块空间(称作：`GOT`表)，重定位时把`b`的地址填写到`GOT`表中。

而`liba.so`的代码段中，把`GOT`表的地址填写到引用`b`的地方，因为`GOT`表在编译阶段是可以确定的，使用的是相对地址。

这样，就可以在不修改`liba.so`代码段的前提下，动态的对符号`b`进行了重定位！

其实，在一个动态库中存在 2 个`GOT`表，分别用于重定位变量符号(`section`名称：`.got`)和函数符号( `section` 名称：`.got.plt`)。

也就是说：所有变量类型的符号重定位信息都位于`.got`中，所有函数类型的符号重定位信息都位于`.got.plt`中。

并且，在一个动态库文件中，有两个特殊的段(`.rel.dyn`和`.rel.plt`)来告诉链接器：`.got`和`.got.plt`这两个表中，有哪些符号需要进行重定位，这个问题下面会深入讨论。

#### liba.so动态库文件的布局

为了更深刻的理解`.got`和`.got.plt`这两个表，有必要来拆解一下`liba.so`动态库文件的内部结构。

通过`readelf -S liba.so`指令来看一下这个`ELF`文件中都有哪些`section`:

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

可以看到：一共有`28`个`section`，其中的`21、22`就是两个`GOT`表。

另外，从装载的角度来看，装载器并不是把这些`sections`分开来处理，而是根据不同的读写属性，把多个`section`看做一个`segment`。

再次通过指令 `readelf -l liba.so` ，来查看一下`segment`信息:

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

也就是说：

这`28`个`section`中(关注绿色线条)：

> 1. section 0 ~ 16 都是可读、可执行权限，被当做一个 segment;
>
> 1. section 17 ~ 24 都是可读、可写的权限，被动作另一个 segment;

再来重点看一下`.got`和`.got.plt`这两个`section`(关注黄色矩形框)：

可见：`.got`和`.got.plt`与数据段一样，都是可读、可写的，所以被当做同一个 `segment`被加载到内存中。

通过以上这`2`张图(红色矩形框)，可以得到`liba.so`动态库文件的内部结构如下：

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

#### liba.so动态库的虚拟地址

来继续观察`liba.so`文件`segment`信息中的`AirtAddr`列，它表示的是被加载到虚拟内存中的地址，重新贴图如下：

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

因为编译动态库时，使用了代码位置无关参数(`-fPIC`)，这里的虚拟地址从`0x0000_0000`开始。

当`liba.so`的代码段、数据段被加载到内存中时，动态链接器找到一块空闲空间，这个空间的开始地址，就相当于一个基地址。

`liba.so`中的代码段和数据段中所有的虚拟地址信息，只要加上这个基地址，就得到了实际虚拟地址。

我们还是把上图中的输出信息，画出详细的内存模型图，如下所示：

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

#### GOT表的内部结构

现在，我们已经知道了`liba.so`库的文件布局，也知道了它的虚拟地址，此时就可以来进一步的看一下`.got`和`.got.plt`这两个表的内部结构了。

从刚才的图片中看出：

> 1. .got 表的长度是 0x1c，说明有 7 个表项(每个表项占 4 个字节);
>
> 1. .got.plt 表的长度是 0x18，说明有 6 个表项;

上文已经说过，这两个表是用来重定位所有的变量和函数等符号的。

那么：`liba.so`通过什么方式来告诉动态链接器：需要对`.got`和`.got.plt`这两个表中的表项进行地址重定位呢？

在静态链接的时候，目标文件是通过两个重定位表`.rel.text`和`.rel.data`这两个段信息来告诉链接器的。

对于动态链接来说，也是通过两个重定位表来传递需要重定位的符号信息的，只不过名字有些不同：`.rel.dyn`和`.rel.plt`。

通过指令 `readelf -r liba.so`来查看重定位信息：

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

从黄色和绿色的矩形框中可以看出：

> 1. liba.so 引用了外部符号 b，类型是 R_386_GLOB_DAT，这个符号的重定位描述信息在 .rel.dyn 段中;
>
> 1. liba.so 引用了外部符号 func_b, 类型是 R_386_JUMP_SLOT，这个符号的重定位描述信息在 .rel.plt 段中;

从左侧红色的矩形框可以看出：每一个需要重定位的表项所对应的虚拟地址，画成内存模型图就是下面这样：

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

暂时只专注表项中的红色部分：`.got`表中的`b`, `.got.plt`表中的`func_b`，这两个符号都是`libb.so`中导出的。

也就是说：

`liba.so`的代码中在操作变量`b`的时候，就到`.got`表中的`0x0000_1fe8`这个地址处来获取变量`b`的真正地址；

`liba.so`的代码中在调用`func_b`函数的时候，就到`.got.plt`表中的`0x0000_200c`这个地址处来获取函数的真正地址；

#### 反汇编liba.so代码

下面就来反汇编一下`liba.so`，看一下指令码中是如何对这两个表项进行寻址的。

执行反汇编指令：`$ objdump -d liba.so`，这里只贴出`func_a1`函数的反汇编代码：

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

第一个绿色矩形框(`call 490 <__x86.get_pc_thunk.bx>`)的功能是：把下一条指令(`add`)的地址存储到`%ebx`中，也就是：

```
%ebx = 0x622
```

然后执行: `add $0x19de,%ebx`，让`%ebx`加上`0x19de`，结果就是：`%ebx = 0x2000`。

`0x2000`正是`.got.plt`表的开始地址！

看一下第`2`个绿色矩形框：

`mov -0x18(%ebx),%eax`: 先用`%ebx`减去`0x18`的结果，存储到`%eax`中，结果是：`%eax = 0x1fe8`，这个地址正是变量`b`在`.got`表中的虚拟地址。

`movl $0x1f,(%eax)`：在把`0x1f`(十进制就是`31`)，存储到`0x1fe8`表项中存储的地址所对应的内存单元中(`libb.so`的数据段中的某个位置)。

因此，当链接器进行重定位之后，`0x1fe8`表项中存储的就是变量`b`的真正地址，而上面这两步操作，就把数值`31`赋值给变量`b`了。

第`3`个绿色矩形框，是调用函数`func_b`，稍微复杂一些，跳转到符号 `func_b@plt`的地方，看一下反汇编代码：

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

`jmp`指令调用了`%ebx + 0xc`处的那个函数指针，从上面的`.got.plt`布局图中可以看出，重定位之后这个表项中存储的正是`func_b`函数的地址(`libb.so`中代码段的某个位置)，所以就正确的跳转到该函数中了。

- EOF -

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**加主页君微信，不仅C/C++技能+1**

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

主页君日常还会在个人微信分享**C/C++开发学习资源**和**技术文章精选**，不定期分享一些**有意思的活动**、**岗位内推**以及**如何用技术做业余项目**

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

加个微信，打开一扇窗

推荐阅读  点击标题可跳转

1、[80% 的 Linux 都不懂的内存问题](http://mp.weixin.qq.com/s?__biz=MzAxNDI5NzEzNg==&mid=2651170931&idx=1&sn=1637771be60a1d2fa004181979bf7846&chksm=8064792cb713f03a3575a8abef19ccf7881a22a544f592aea1f01950b4f6ba9b4a9e20f3760c&scene=21#wechat_redirect)

2、[性能提升 8450%，Linux 内核函数获大幅改进](http://mp.weixin.qq.com/s?__biz=MzAxNDI5NzEzNg==&mid=2651170836&idx=1&sn=10403b754f822ad4d8afae92f77d166c&chksm=8064794bb713f05d372b5ff67e16dd361164ae9b51d4886d0a05eefa9245f6cf052a97bec72f&scene=21#wechat_redirect)

3、[Linux 存储系列：请描述一下文件的 io 栈？](http://mp.weixin.qq.com/s?__biz=MzAxNDI5NzEzNg==&mid=2651170679&idx=1&sn=6ed391614f54635eee5c7d644eb06f8d&chksm=80647628b713ff3e3abc1ab607e42d3ab53a0d4f1e86136809d1c931f442496e61b683f4ac87&scene=21#wechat_redirect)

**关注『CPP开发者』**

看精选C/C++技术文章

![](https://res.wx.qq.com/t/fed_upload/b39ef69e-c4d6-4169-8612-5f00a84860e7/wx-avatar-default.svg)

公众号

点赞和在看就是最大的支持❤️

阅读 2921

​
