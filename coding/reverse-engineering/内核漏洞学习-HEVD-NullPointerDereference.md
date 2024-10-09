# 

pyikaaaa 看雪学苑

_2022年01月11日 18:01_

![图片](https://mmbiz.qpic.cn/sz_mmbiz_jpg/1UG7KPNHN8EPe1jYJj9nlXQKNV6O1rBTxNoNfkTwlOc1Lxstx60LZ0fLdvIpFDZOlickRiayg5iaosG01T1lRfVfA/640?wx_fmt=jpeg&wxfrom=13)

本文为看雪论坛优秀文章\
看雪论坛作者ID：pyikaaaa

## 

1

概述

HEVD：漏洞靶场，包含各种Windows内核漏洞的驱动程序项目，在Github上就可以找到该项目，进行相关的学习。

_Releases · hacksysteam/HackSysExtremeVulnerableDriver · GitHub_

环境准备：

Windows 7 X86 sp1 虚拟机

使用VirtualKD和windbg双机调试

HEVD 3.0+KmdManager+DubugView

## 

2

前置知识

指针的三种错误使用：

1、由指针指向的一块动态内存，在利用完后，没有释放内存，导致内存泄露

2、野指针（悬浮指针）的使用，在指针指向的内存空间使用完释放后，指针指向的内存空间已经归还给了操作系统，此时的指针成为野指针，在没有对野指针做处理的情况下，有可能对该指针再次利用导致指针引用错误而程序崩溃。

3、Null Pointer空指针的引用，对于空指针的错误引用往往是由于在引用之前没有对空指针做判断，就直接使用空指针，还有可能把空指针作为一个对象来使用，间接使用对象中的属性或是方法，而引起程序崩溃，空指针的错误使用常见于系统、服务、软件漏洞方面。

总结

free（p）后：p仍然指向那块地址，但是地址被释放，回归给系统，指针仍然可以使用，可以利用池喷射，去多次申请内昆空间，撞这个指针p指向的地址，改写这个地址内容，在调用p，执行自己写入的代码。

p=NULL 后：p 不指向任何内存地址，通常指向000 0页地址，通过ntallocvirtualmemory 申请0页地址空间。在0页地址写代码。调用null指针，执行shellcode。

使用ntallocvirtualmemory 函数申请内存：_NtAllocateVirtualMemory function (ntifs.h) - Windows drivers | Microsoft Docs_

## 

## 

3

漏洞点分析

空指针漏洞

此类漏洞利用主要集中在两种方式上：

1、利用NULL指针。

2、利用零页内存分配可用内存空间

（1）分析漏洞点

UserValue = \*(PULONG)UserBuffer;从用户模式获取value的值，如果uservalue=magicvalue的值，向缓冲区赋值，并打印信息，反之则释放缓冲区，清空指针（ NullPointerDereference = NULL;），之后，对于安全版本，对NullPointerDereference进行检查判断其是否被置空，非安全版本，未对NullPointerDereference进行检查判断，直接调用callback。

```
NTSTATUS
```

## 

4

漏洞利用

## HEVD_IOCTL_NULL_POINTER_DEREFERENCE控制码对应的派遣函数NullPointerDereferenceIoctlHandler case。

```
HEVD_IOCTL_NULL_POINTER_DEREFERENCE:
```

NullPointerDereferenceIoctlHandler函数调用TriggerNullPointerDereference触发漏洞。

```
NTSTATUS
```

（1）测试

```
#include<stdio.h>
```

当我们传入值与MagicValue值不匹配时，则会触发漏洞。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

因为uservalue=0xBAD0B0B0，所以打印出信息。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

（2）漏洞利用

官方给出的方法也是利用NtAllocateVirtualMemory函数，在0页申请内存。

该函数在指定进程的虚拟空间中申请一块内存，该块内存默认将以64kb大小对齐，所以SIZE_T RegionSize = 0x1000;

```
BOOL MapNullPage() {
```

申请成功后，将shellcode地址放入偏移四字节处，因为CallBack成员在结构体的0x4字节处，使传入值与MagicValue值不匹配，触发漏洞。

```
ULONG MagicValue = 0xBAADF00D;
```

exp，可供参考

```
#include<stdio.h>
```

payload功能：遍历进程，得到系统进程的token，把当前进程的token替换，达到提权目的。

相关内核结构体：在内核模式下，fs:\[0\]指向KPCR结构体。

```
_KPCR
```

payload：注意堆栈平衡问题。

```
 __asm {
```

运行exp提权成功：

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

流程总结：

- 申请0页内存

- 将payload放入内存任意位置

- 并在0x4地址放入payload地址

- 调用TriggerNullPointerDereference函数

- 提权启动cmd

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

E N D

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**看雪ID：pyikaaaa**

https://bbs.pediy.com/user-home-921642.htm

\*本文由看雪论坛 pyikaaaa 原创，转载请注明来自看雪社区

[!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458409596&idx=3&sn=658012a9debc76c73efed03342d41358&chksm=b18f6af686f8e3e09180522b243def309e265532cfadfdbf5becd7d9ec90f4505136959ebdfe&scene=21#wechat_redirect)

**#** **往期推荐**

1.[一个数据加密恶意样本分析](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458418146&idx=1&sn=0bea59a60b1525d2ef55134a659b5a27&chksm=b18f4b6886f8c27eb38ea10fc994bf566355d2152cb9c28e60b5b1fd8f2ff6b9dc557cc2a984&scene=21#wechat_redirect)

2.[windows64位分页机制分析-隐藏可执行内存方法](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458417107&idx=1&sn=3c1a896c96aed35811e32340facd1b58&chksm=b18f475986f8ce4fc3954717159d384cd604c737385fcab99ec5fd6a1dbe8aa31fe1d3015c54&scene=21#wechat_redirect)

3.[分析屏保区TOP1的一款MacOS软件](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458413768&idx=1&sn=1f45d1c9037e45e2be1a8d9e43be3186&chksm=b18f5a4286f8d3543243647499c15116e0005a5aa8a43e7c24e31b37a763eef599aa3d3f271f&scene=21#wechat_redirect)

4.[某视频app的学习记录](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458413744&idx=1&sn=134b79e811ade3cd9aabd9e2554c8796&chksm=b18f5a3a86f8d32c916883e5356c6fffecb09b504d6bc544530315e733142dde218530b5c511&scene=21#wechat_redirect)

5.[Chrom V8分析入门——Google CTF2018 justintime分析](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458413548&idx=2&sn=9a57c3d6a96df540b62e4d041495f23c&chksm=b18f596686f8d070aa544f7366f2fed5fb9cd8a4ccc24ec1a7eadb649389556f5080fa06fde2&scene=21#wechat_redirect)

6.[内核学习-异常处理](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458413193&idx=3&sn=d9e54f5f39476109634b7319cef91205&chksm=b18f580386f8d115cc0c1f30bec88e0c2705d7d5aee020738a5233620a6ab63db202decf5269&scene=21#wechat_redirect)

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

点击“阅读原文”了解更多

阅读原文

阅读 1533

​

写留言

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/1UG7KPNHN8EGLfh77kFmnicd9WOic2ibvhCibFdB4bL4srJCgo2wnvdoXLxpIvAkfCmmcptXZB0qKWMoIP8iaibYN2FA/300?wx_fmt=png&wxfrom=18)

看雪学苑

322

写留言

写留言

**留言**

暂无留言
