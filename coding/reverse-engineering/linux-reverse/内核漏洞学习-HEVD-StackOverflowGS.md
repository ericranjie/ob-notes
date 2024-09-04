# 

pyikaaaa 看雪学苑

 _2021年12月29日 18:01_

![](https://mmbiz.qpic.cn/mmbiz_jpg/kR3MmOkZ9r6NkLx6w1ib1VAtF8869zN1j0tGOpUcjd2uBlDMhx0ZvvrUHepYTeZe8KDClaZQWBuynZNCoIJ43hA/640?wx_fmt=jpeg&wxfrom=13&tp=wxpic)  

本文为看雪论坛优秀文章  
看雪论坛作者ID：pyikaaaa

  

  

1

  

**概述**

  

HEVD：漏洞靶场，包含各种Windows内核漏洞的驱动程序项目，在Github上就可以找到该项目，进行相关的学习。

Releases · hacksysteam/HackSysExtremeVulnerableDriver · GitHub_（https://github.com/hacksysteam/HackSysExtremeVulnerableDriver/releases）_

> Windows 7 X86 sp1 虚拟机 
> 
> 使用VirtualKD和windbg双机调试 
> 
> HEVD 3.0+KmdManager+DubugView

  

  

2

  

**前置知识**

###   

### （1）栈中的守护天使：GS

  

安全编译选项-GS

![图片](https://mmbiz.qpic.cn/mmbiz_png/kR3MmOkZ9r4mV9EFf8zJTpB2e7chlnjciaS1CkkdQB7DE6prwpgq66xCZeYfa10eo1HZBL7OWhia3jdAqJTHCgyg/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

GS编译选项为每个函数调用增加了一些额外的数据和操作，用以检测栈中的溢出。

  
在所有函数调用发生时，向栈帧内压入一个额外的随机 DWORD，随机数标注为“SecurityCookie”。

  
Security Cookie位于EBP之前，系统还将在.data的内存区域中存放一个Security Cookie的副本，如图：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

当栈中发生溢出时，Security Cookie将被首先淹没，之后才是EBP和返回地址。在函数返回之前，系统将执行一个额外的安全验证操作，被称做 Security check。

  
在Security Check的过程中，系统将比较栈帧中原先存放的Security Co okie和.data中副本的值，如果两者不吻合，说明栈帧中的Security Cookie已被破坏，即栈中发生了溢出。

当检测到栈中发生溢出时，系统将进入异常处理流程，函数不会被正常返回，ret 指令也不会被执行，如图：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

但是额外的数据和操作带来的直接后果就是系统性能的下降，为了将对性能的影响降到最小，编译器在编译程序的时候并不是对所有的函数都应用GS，以下情况不会应用GS。

(1）函数不包含缓冲区。

(2）函数被定义为具有变量参数列表。

(3）函数使用无保护的关键字标记。

(4）函数在第一个语句中包含内嵌汇编代码。

(5）缓冲区不是8字节类型且大小不大于4个字节。

  

除了在返回地址前添加Security Cookie外，在Visual Studio 2005及后续版本还使用了变量重排技术，在编译时根据局部变量的类型对变量在栈帧中的位置进行调整，将字符串变量移动到栈帧的高地址。这样可以防止该字符串溢出时破坏其他的局部变量。同时还会将指针参数和字符串参数复制到内存中低地址，防止函数参数被破坏。如图：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

通过GS安全编译选项，操作系统能够在运行中有效地检测并阻止绝大多数基于栈溢出的攻击。要想硬对硬地冲击GS机制，是很难成功的。让我们再来看看Security C ookie产生的细节。

① 系统以.data节的第一个双字作为Cookie的种子,或称原始Cookie(所有函数的eCookie都用这个DWORD生成)。

② 在程序每次运行时Cookie 的种子都不同，因此种子有很强的随机性。

③ 在栈桢初始化以后系统用ESP异或种子，作为当前函数的Cookie，以此作为不同函数之间的区别，并增加Cookie的随机性。

④ 在函数返回前，用ESP还原出（异或）Cookie 的种子。

  

在微软出版的Writing Secure Code一书中谈到GS选项时，作者还给出了微软内部对GS为产品所提供的安全保护的看法：

修改栈帧中函数返回地址的经典攻击将被GS机制有效遏制;

基于改写函数指针的攻击，GS很难防御;

针对异常处理机制的攻击，GS很难防御;  

GS是对栈帧的保护机制，因此很难防御堆溢出的攻击。

所以绕过GS保护有几种方法：

① 利用未被保护的内存突破GS

② 覆盖虚函数突破GS

③ 攻击异常处理突破GS

④ 同时替换栈中和.data中的Cookie突破GS

###   

### （2）攻击SEH绕过GS保护

  

GS机制并没有对S.E.H 提供保护，换句话说我们可以通过攻击程序的异常处理达到绕过GS 的目的。我们首先通过超长字符串覆盖掉异常处理函数指针，然后想办法触发一个异常，程序就会转入异常处理，由于异常处理函数指针已经被我们覆盖，那么我们就可以通过劫持S.E.H 来控制程序的后续流程。

每个SEH结构体包含两个DWORD指针：SEH链表指针next seh和异常处理函数句柄Exception handler，共八个字节，存放于栈中。如下图所示，其中SEH链表指针next seh用于指向下一个SEH结构体，异常处理函数句柄Exception handler为一个异常处理函数。

```
EXCEPTION_DISPOSITION
```

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

Visual C++为使用结构化异常处理的函数生成的标准异常堆栈帧，它看起来像下面这个样子：

```
EBP-00 _ebp
```

  

（0day第十章有详细介绍）

关于异常更详细的知识：

[原创]内核学习-异常处理-软件逆向-看雪论坛-安全社区|安全招聘|bbs.pediy.com_（https://bbs.pediy.com/thread-270045.htm）_

##   

  

3

  

**漏洞点分析**  

  

漏洞原理：栈溢出漏洞，使用过程中使用危险的函数，没有对参数进行限制  
BufferOverflowStackGS开启GS保护。

（1）加载驱动

（这部分内容直接使用上一篇分析的HEVD_BufferOverflowStack的图片了）

安装驱动程序，使用kmdManager 加载驱动程序，DebugView检测内核，可以看到加载驱动程序成功。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

windbg：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

```
lm  查看所有已加载模块
```

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

（2）分析漏洞点

对BufferOverflowStackGS.c源码进行分析，漏洞函数TriggerBufferOverflowStackGS，存在明显的栈溢出漏洞，RtlCopyMemory函数没有对KernelBuffer大小进行验证，直接将size大小的userbuffer传入缓冲区，没有进行大小的限制(例如安全版本的sizeof(KernelBuffer))，因此造成栈溢出。

  

因为开启了GS保护，所以对于BufferOverflowStackGS，不使用攻击返回地址，而攻击SEH，覆盖SEH handle ，人为构造异常，触发异常，执行异常处理函数（此时异常处理函数是我们的payload），这个步骤发生在再返回地址，检查cookie之前。

```
#define BUFFER_SIZE 512  
```

##   

  

4

  

**漏洞利用**

  

因为GS保护，在栈中有cookie，所以无法溢出到返回地址，那就溢出到SEH handle。

windbg调试：

1、在TriggerBufferOverflowStackGS函数处下断点

```
kd>bp HEVD!TriggerBufferOverflowStackGS
```

  

2、

```
kd>g  //运行
```

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

ebp:971c1ae0

3、运行exp，找到seh handle 距离kernelbuffer的偏移（后面再说exp的原理）。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

kernelbuffer:971c18b4

4、

Visual C++为使用结构化异常处理的函数生成的标准异常堆栈帧，它看起来像下面这个样子：

```
EBP-00 _ebp
```

  

EBP-0C handler函数地址=971c1ae0-0c=971c1ad4

971c1ad4-kernelbuffer:971c18b4=220, 所以 想要覆盖到handler函数地址 需要大小0x224，偏移确定了，分析下exp原理。

5、

构造exp（官方exp）

具体原理看注释：

```
DWORD WINAPI StackOverflowGSThread(LPVOID Parameter) {
```

  

HEVD_IOCTL_BUFFER_OVERFLOW_STACK_GS控制码对应的派遣函数BufferOverflowStackGSIoctlHandler。

```
DriverEntry
```

```
IrpDeviceIoCtlHandler(
```

  

BufferOverflowStackGSIoctlHandler函数调用TriggerBufferOverflowStackGS触发漏洞函数。

```
BufferOverflowStackGSIoctlHandler(
```

  

payload功能：遍历进程，得到系统进程的token，把当前进程的token替换，达到提权目的。

相关内核结构体：

在内核模式下，fs:[0]指向KPCR结构体。

```
_KPCR
```

  

payload：

```
__asm {
```

  

提权成功：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

在网上看到的一个思路，没有获取se handle到kernelbuffer的偏移，而是直接将缓冲区数据全部写成payload地址。

```
memset(pMapView, 'a', PAGE_SIZE);
```

  

测试结果，蓝屏了。  

  

  

5

  

**补丁分析**

  

对RtlCopyMemory的参数进行严格的设置，使用sizeof(kernelbuffer)。  

  

  

其他：  
刚接触内核漏洞，如果有哪写的不对，希望大佬们指出。

  

  

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E) 

  

**看雪ID：pyikaaaa**

https://bbs.pediy.com/user-home-921642.htm

*本文由看雪论坛 pyikaaaa 原创，转载请注明来自看雪社区

  

  

[![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458409596&idx=3&sn=658012a9debc76c73efed03342d41358&chksm=b18f6af686f8e3e09180522b243def309e265532cfadfdbf5becd7d9ec90f4505136959ebdfe&scene=21#wechat_redirect)  

  

**#** **往期推荐**

1.[Kernel从0开始](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458411220&idx=1&sn=bcb5872be8a7d92dee039fc565075057&chksm=b18f505e86f8d948bdad54fa350494b56fe67f2dbb8b0a3c8261b5b620f23609fc987c7fd8cb&scene=21#wechat_redirect)  

2.[强网拟态线上mobile的两道wp](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458410675&idx=1&sn=e015ec78d467a2afdb3ccad3f868e6ec&chksm=b18f6e3986f8e72f921a3fc3113de57766fd66b2c46fa587fa5f5139f698b86166f0fd42c2c0&scene=21#wechat_redirect)

3.[某钱包转账付款算法分析篇](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458410651&idx=1&sn=070a3637c7f9d78e188d2b4ca7e52c81&chksm=b18f6e1186f8e707496c08a09c6c8dd7685ef81eaca6d7e9a54305779281aca232a981e7f207&scene=21#wechat_redirect)

4.[通过PsSetLoadImageNotifyRoutine学习模块监控与反模块监控](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458410644&idx=1&sn=e36fcc9b2b7a1efffb84bc3673b248b6&chksm=b18f6e1e86f8e708e321843fbdc68404b28db00aa48fed2c974a402a7708db095f94820c3da1&scene=21#wechat_redirect)

5.[常见的几种DLL注入技术](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458410636&idx=2&sn=41c0f39c9215a19648b2775fa3c4357b&chksm=b18f6e0686f8e7104fd869ca03dbd98abd919120705d17787efda6badfd6364a637ea63a83b5&scene=21#wechat_redirect)

6.[侠盗猎车 — 玩转固定码](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458410583&idx=2&sn=f51e5e6a81fbf6961727b6a90de730e2&chksm=b18f6edd86f8e7cb1be072459aad2f9aaafe82ef1a49a228cbaf0beca87186f58ed0e6a39f0e&scene=21#wechat_redirect)

  

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

  

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**球分享**

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**球点赞**

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**球在看**

  

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

点击“阅读原文”，了解更多！

阅读原文

阅读 1658

​

写留言

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/1UG7KPNHN8EGLfh77kFmnicd9WOic2ibvhCibFdB4bL4srJCgo2wnvdoXLxpIvAkfCmmcptXZB0qKWMoIP8iaibYN2FA/300?wx_fmt=png&wxfrom=18)

看雪学苑

9分享2

写留言

写留言

**留言**

暂无留言