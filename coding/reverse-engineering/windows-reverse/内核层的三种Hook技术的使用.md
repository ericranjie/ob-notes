
1900 看雪学苑

 _2021年11月20日 18:01_

![Image](https://mmbiz.qpic.cn/mmbiz_jpg/kR3MmOkZ9r5e3c4eFLapuwvK5XCO0HibvtpRhf3UdynIRH8K8qNfgPml1xbou36eFfR7iciba49PTfsskxAU3gVww/640?wx_fmt=jpeg&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)  

本文为看雪论坛优秀文章  
看雪论坛作者ID：1900

  

  

1

  

**前言**

  

本文是在Win7 X86系统上进行实验，实验的内容是在内核层通过三种不同的Hook方法来实现进程保护，让运行的程序不被其他的程序关闭。这里用来被保护的程序是一个简单的HelloWord弹窗程序，程序名是demo.exe。

  

![Image](https://mmbiz.qpic.cn/mmbiz_jpg/kR3MmOkZ9r5e3c4eFLapuwvK5XCO0Hibvb7AtWyfyhn4IW1TPP2gfokk52TqDK7nkHtxV3VkgTkP4mFC4uINOBw/640?wx_fmt=jpeg&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

  

  

2

  

**实现原理**

  

一个程序要想关闭一个进程，首先就要获取这个进程的句柄，并且获得的这个句柄还需要具备进程关闭的访问掩码。也就是说，在用户层我们想要关闭一个进程，首先就需要使用以下的代码来获得一个进程的句柄。

```
HANDLE hProcess = OpenProcess(PROCESS_TERMINATE, FALSE, dwPid);
```

  

其中第一个参数PROCESS_TERMINATE就是让我们获得的进程句柄具有关闭进程的访问掩码，它在文档中的定义如下：  

```
#define PROCESS_TERMINATE                  (0x0001)
```

  

那也就是说，如果HOOK了打开进程的函数，就可以在我们编写的函数中判断所打开的进程是否是我们要保护的进程，以及所要获得的进程句柄是否是要具备进程关闭的访问掩码，如果是的话就把它去掉。这个时候，程序如果使用这个被删除掉进程关闭访问掩码的句柄来关闭进程，就会失败，这就完成了对进程的保护作用。

  

而在用户层所调用的OpenProcess到内核层以后，其实在内核中是调用ZwOpenProcess，所以只要在内核中监控这个函数就可以完成实验。该函数在文档中的定义如下：

```
NTSTATUS
```

  

参数2所代表的就是要打开的这个进程要具备的访问掩码，通过对它的判断与修改就可以实现进程保护。  

  

参数4的一个指向CLIENT_ID的指针，CLIENT_ID在文档中的定义如下：

```
typedef struct _CLIENT_ID {
```

  

其中的UniqueProcess代表的就是进程的PID，根据它就可以用来判断是否是要保护的进程。

  

  

3

  

**代码框架**

  

由于三个实验只是HOOK的方法的代码不同，而其他的比如用户层与内核层的通信等等是相同的。所以为了避免重复，这里先给出代码框架，后续的HOOK代码只需要加进去就好。  

  

用户层所要做的事情就是：

- 安装和卸载驱动
    
- 根据要保护的进程名获取进程PID，以传给内核层供内核层使用
    
- 根据用户输入选择安装或者卸载HOOK，向内核层发送相应的IoControlCode并把返回结果告诉用户
    

  

用户层的代码如下：

```
#include <cstdio>
```

  

而驱动程序就要接受用户层传下来的IoControlCode来进行Hook或者UnHook，并把返回结果传回用户层，具体代码如下：  

```
#include <ntifs.h>
```

#   

#   

4

  

**InlineHook**

  

InlineHook的实现原理是：将原函数要执行的代码改为jmp跳转指令，跳转到我们要执行的代码中执行中。而在我们的代码中，为了保证Hook以后原本的功能可以实现就需要恢复被修改的字节然后再次调用，可是在多线程的环境中频繁的修改与删除很容易带来错误，造成蓝屏。  

  

为了避免这种情况，可以采取热补丁的技术来实现，首先看看NtOpenProcess函数的反汇编结果。

  

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

可以看到在函数最开始执行了一句mov edi, edi这么一个无意义的指令，而这个指令对应的硬编码是两字节0x8BFF，而在这个函数上面有5个字节的0x90。所以可以通过修改最开始的两个字节实现短跳转跳到上面的5个字节的0x90的开始地址执行，而这五个字节的0x90就可以修改为跳转到要真正执行的我们的函数的地址。  

  

而在我们的函数中，如果要执行原来的代码，只需要直接跳转到NtOpenProcess函数2个字节偏移处，也就是略过最开始的mov edi, edi这条指令，直接从push ebp开始执行，这样就不用频繁的对原来的函数进行修改。

  

这里还有一点需要注意，由于页保护的存在，所以不能直接通过函数地址对函数中的字节进行修改。需要通过内存描述符(MDL)描述一块包含该内存区域的起始地址，拥有者进程，字节数量，标记等信息的内存区域，并通过将它修改具有可写的属性来实现对函数的读写。  

  

具体实现代码如下：

```
VOID NtMyOpenProcess(PHANDLE ProcessHandle,
```

#   

#   

5

  

**SysenterHook**

  

程序员使用OpenProcess打开进程的时候，这个函数其实是Kernel32.dll的导出函数，而在Kernel32.dll中这个函数的作用就是检测参数，随后将参数入栈以后在调用ntdll.dll中的ZwOpenProcess，该函数的具体实现如下：  

```
.text:77F05D88 ; NTSTATUS __stdcall ZwOpenProcess(PHANDLE ProcessHandle, ACCESS_MASK DesiredAccess, POBJECT_ATTRIBUTES ObjectAttributes, PCLIENT_ID ClientId)
```

  

可以看到这个函数首先将0xBE赋给eax，此时这个0xBE就叫做调用号，它用来指定在内核中你要调用的那个内核函数是哪个。随后对0x7FFE0300这个地址中保存的地址进行了调用。  

  

而0x7FFE03000这个地址中究竟保存的是什么就要取决于你的CPU是否支持快速调用，如果支持这个地址中保存的就是ntdll.dll中的KiFastCallSystemCall，如果不支持那么保存的就是ntdll.dll中的KiIntSystemCall。

  

而检测系统是否支持快速调用的方法是，为eax赋值为1，随后调用cpuid指令，那么处理器的信息就会被放到ecx和edx寄存器中，此时判断edx的第11位是否为1，如果为1那么当前CPU就支持快速调用，否则不支持。检测代码如下：

```
BOOL CanSysenter()
```

  

而要实现SysenterHook的电脑，它的CPU是要支持快速调用的，所以此时继续跟进ntdll.dll中的KiFastCallSystemCall，函数实现如下：  

```
.text:77F070B0 public KiFastSystemCall
```

  

可以看到，函数首先将esp赋值给edx，也就是说此时edx指向的是包含返回地址和参数的栈地址。但要注意，由于调用了ZwOpenProcess以后又调用了KiFastSystemCall。所以此时edx所指的地址保存了两个返回地址，具体如下图所示。  

  

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

随后函数会调用sysenter指令，此时这个指令就会从特殊模块寄存器(MSRs)寄存器中取出在内核中要运行的代码地址以及要设置的代码段等信息取出，并将它们赋给相应的寄存器。MSRs中包含的不同功能的寄存器有上百种，但是与sysenter配合的MSRs一共有下表的三个：  

  

|   |   |   |
|---|---|---|
|名称|偏移|功能|
|SYSENTER_CS_MSR|0x174|指定切换到内核层的CS|
|SYSENTER_ESP_MSR|0x175|指定切换到内核层的ESP|
|SYSENTER_EIP_MSR|0x176|指定切换到内核层的EIP|

  

那也就是说，进入内核层要执行的代码是有SYSENTER_EIP_MSR来指定的，而程序员可以用rdmsr/wrmsr来分别实现对SYSENTER_EIP_MSR的读取和修改。当使用rdmsr指令的时候，程序会将ecx所指的MSRs的指定寄存器中的值写入edx:eax，而调用wrmsr的时候，程序会将edx:eax的值写入到ecx所指的MSRs寄存器中。

  

那也就是说可以通过将ecx赋值为0x176，再将eax赋值为要调用的函数，就可以实现SysenterHook。当程序进入内核的时候，首先执行的就是我们所指定的函数。具体的实现代码如下：  

```
VOID MyKiFastCallEntry();  //Hook以后要执行的函数
```

#   

  

6

  

**SSDT Hook**

  

上面说过，进入内核的之前，程序会给eax赋值一个调用号，用来在进入内核以后找到要执行的函数。而找到的办法其实是通过找系统服务描述服表来找到的，在这张表中就保存了系统的所有函数，系统就是通过这张表来调用相应的函数。这张表的地址在32位的系统中是导出的，可以直接获取。而获取了该表所保存的内容如下：

```
typedef struct _KSERVICE_TABLE_DESCRIPTOR
```

  

打开进程的函数在ntoskrnl中，称为系统服务表，这是一个KSYSTEM_SERVICE_TABLE的变量，定义如下：

```
typedef struct _KSYSTEM_SERVICE_TABLE
```

  

其中的pServiceTableBase指向的地址就是一张保存了各个函数地址的表，通过替换表中相应位置的内容就可以实现HOOK。因为当用户层的函数调用进入到内核的时候，系统通过这张表找到的函数地址就是我们指定的地址。

  

另外这张表同样是因为页保护的存在而不可写，上面介绍了通过MDL的方式映射一块内存来进行写操作。其实还有另一个办法，那就是修改CR0寄存器中的WP位(第16位)。当它为0的时候，页保护就会被关闭，这样就可以实现对SSDT表的修改。

  

以下是具体的代码实现：

```
#pragma pack(1)
```

#   

  

7

  

**实验结果**

  

首先是成功Hook以后，可以看到通过任务管理器关闭demo.exe进程的时候会失败。

  

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

而当卸载了钩子以后就可以顺利关闭进程。

  

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

  

  

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E) 

  

**看雪ID：1900**

https://bbs.pediy.com/user-home-835440.htm

*本文由看雪论坛 1900 原创，转载请注明来自看雪社区

  

  

[![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458402059&idx=1&sn=b16e5d69f4bbb055f1930ae242a5e58a&chksm=b18f0d8186f88497d8460db4deaf29aa4c1b62b3c61edd094c7871291ed36f489c370e8113a7&scene=21#wechat_redirect)

  

[![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)](https://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458387399&idx=2&sn=38495add2a3a3677b2c436581c07e432&scene=21#wechat_redirect)

  

**#** **往期推荐**

1.[记一次头铁的病毒样本分析过程](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458405156&idx=1&sn=c2061cbc0d160c86568497d663d40dbd&chksm=b18f79ae86f8f0b8c0072240342b03b737b21deb89fd1f37a26a7f58f5e407437fc92633997b&scene=21#wechat_redirect)  

2.[通过CmRegisterCallback学习注册表监控与反注册表监控](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458405024&idx=1&sn=a8617e12ff4d8e39d4fe89dae2cdf577&chksm=b18f782a86f8f13c05b91d743d6a09035c01d8a24b8d1c012cc68ccb2ff72d8bb45151b4a7fb&scene=21#wechat_redirect)

3.[BCTF2018-houseofatum-Writeup题解](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458402442&idx=1&sn=65f4c412b3495b27e168d94e8adf230f&chksm=b18f0e0086f887169bf7566dafa1b5cce2800a245159721b976d49da3d44d684357f526389a0&scene=21#wechat_redirect)

4.[利用__libc_csu_init控制64位寄存器](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458402059&idx=2&sn=b93b65d6acaecbf0b48d5a2f54b24b62&chksm=b18f0d8186f884973ba64d2f37776acf439f7db53858304f18980e4501cd20ebdf1fb4172de5&scene=21#wechat_redirect)

5.[一个堆题inndy_notepad的练习笔记](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458401180&idx=2&sn=7569d3aaaafe683224e12d1384554287&chksm=b18f091686f880009c3f4eb92e84b79bf2adfa615d8d28e8742a134199376d5cd715f55145f2&scene=21#wechat_redirect)

6.[源码编译——Xposed源码编译详解](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458400885&idx=2&sn=10b7d83e238295e020c13ba181093d3b&chksm=b18f08ff86f881e90b70b1fd5f772f433a26964fc096b28fd8e90da17aa71f5a31bfee6028eb&scene=21#wechat_redirect)

  

  

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

公众号ID：ikanxue  

官方微博：看雪安全

商务合作：wsc@kanxue.com

  

  

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**球分享**

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**球点赞**

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**球在看**

  

  

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

点击“阅读原文”，了解更多！

Read more

Reads 5781

​

Comment

**留言 4**

- Unison
    
    2021年11月20日
    
    Like1
    
    x64内联不了，pg。。。得改进啊，vt兼容性又那个，能不能着重考虑x64系统先
    
- ω
    
    2021年11月21日
    
    Like
    
    佛了，ObRegisterCallbacks不是早就提供功能了吗，搞什么破坏内核的inline hook![[奸笑]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
- 拉开一帆风顺🥕
    
    2021年11月20日
    
    Like
    
    长见识了，第二种hook没有学过，只会一和三，不过我也想问一下一下评论区那个关于64位inlinehook的问题，在64位下按照文章给出的my函数的定义方式，会不会出现因为my函数和原函数距离过远而无法跳转过去的问题? (之前自己的inlinehook框架的时候当时是有这个问题的，做了一个二次跳转处理才解决)
    
- V
    
    2021年11月20日
    
    Like
    
    copy忍者前来报到![[呲牙]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    

已无更多数据

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/1UG7KPNHN8EGLfh77kFmnicd9WOic2ibvhCibFdB4bL4srJCgo2wnvdoXLxpIvAkfCmmcptXZB0qKWMoIP8iaibYN2FA/300?wx_fmt=png&wxfrom=18)

看雪学苑

2637

4

Comment

**留言 4**

- Unison
    
    2021年11月20日
    
    Like1
    
    x64内联不了，pg。。。得改进啊，vt兼容性又那个，能不能着重考虑x64系统先
    
- ω
    
    2021年11月21日
    
    Like
    
    佛了，ObRegisterCallbacks不是早就提供功能了吗，搞什么破坏内核的inline hook![[奸笑]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
- 拉开一帆风顺🥕
    
    2021年11月20日
    
    Like
    
    长见识了，第二种hook没有学过，只会一和三，不过我也想问一下一下评论区那个关于64位inlinehook的问题，在64位下按照文章给出的my函数的定义方式，会不会出现因为my函数和原函数距离过远而无法跳转过去的问题? (之前自己的inlinehook框架的时候当时是有这个问题的，做了一个二次跳转处理才解决)
    
- V
    
    2021年11月20日
    
    Like
    
    copy忍者前来报到![[呲牙]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    

已无更多数据