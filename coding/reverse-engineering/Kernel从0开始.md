# 

PIG-007 看雪学苑

 _2021年12月23日 18:14_

![](https://mmbiz.qpic.cn/mmbiz_jpg/kR3MmOkZ9r6RCfZdjHqC2hBw0D2yFgfQKXWxevAC2pG4D1JAqGAYb0WheBVWzyJl4DMoKBaICJRMrfehMokp3g/640?wx_fmt=jpeg&wxfrom=13&tp=wxpic)  

本文为看雪论坛优秀文章  
看雪论坛作者ID：PIG-007

  

  

一

  

**简介**

  

这里就尝试用堆来解题，由于kernel的解法多种多样，这里我们从最简单的UAF入手。

给出自己设计的堆题目，存在很多的漏洞，read越界读，edit越界写，UAF，Double Free等：

```
#include <linux/module.h>
```

#   

  

二

  

**利用Cred结构体提权**

##   

## 1、正常UAF

###   

### 前置知识

  

由于是UAF漏洞，所以直接尝试再重启一个进程，这样新进程启动时就会申请一个Cred结构体(这里大小为0xa8)。而如果此时申请的结构体恰好落在我们释放过的堆块上，那么我们就可以利用UAF漏洞修改Cred结构体，将其uid和gid改为0，再利用该进程原地起shell，就能获得root权限的shell了。

这里同样需要一点前置知识，之前也写过类似的，其实就相当于修改某个进程的cred结构体中的uid和gid就能将该进程提权了，之后利用提权后的进程起shell得到的shell就是提权后的shell。

比较简单，这里直接给：

###   

### POC

```
#include <sys/types.h>
```

  

这里还需要说明的是，cred结构体大小在我编译的4.4.72中为0xa8，在不同内核版本可能不同，通常可以查看对应版本的Linux内核源码或者写个简便的C程序运行一下即可知道。

  

## 2、伪条件竞争造成的UAF(多进程)

###   

### 前置知识

  

前面我们的UAF是正常的指针未置空的UAF，但如果在程序中是add函数申请chunk，只有在关闭设备时才会释放chunk。那么这样当我们对一个设备进行操作时，只有在关闭设备时才能释放chunk，这就无法显著地造成UAF。但是如果能够对同一个设备打开两次 (操作符分别为fd1、fd2) ，申请一个堆块后，关闭掉第一个设备fd1后，就能释放该堆块。之后利用fd2继续对设备进行写操作，就能够继续修改释放掉的堆块了，这样就造成了一个UAF漏洞。同样也是利用cred结构体进行提权。

同样直接给出poc即可。

###   

### POC

```
#include <sys/types.h>
```

#   

  

三

  

**劫持tty_struct结构体**

  

这个真是调了我无敌久。

##   

## 原理

###   

### 1、函数调用链

  

entry_SYSCALL_64->SyS_write->SYSC_write->vfs_write

->__vfs_write->tty_write->do_tty_write->n_tty_write->pty_write

这里我们需要的就是劫持某个结构体，从而使得原本通过该结构体调用pty_write函数指针变为调用我们的ROP链条。

###   

### 2、劫持栈

  

由于用户空间和内核空间得返回进入需要用到栈，所以一般需要进行栈劫持，这里我们可以看到当通过ptmx进入其write函数时，rax为从tty_struct中获取的operations ops指针，而此时该指针已经被我们劫持了，所以如果有类似于mov rsp、rax之类的gadget就能将栈劫持到我们可控的operations ops指针指向的内存处，那么之后就很容易进行内核和用户空间的转换。

这里就用到常用的一个gadget。

movRspRax_decEbx_ret

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

### 3、结构体

  

这个之前也是讲过的。

```
struct tty_struct {
```

  

当我们打开ptmx设备时，会使用kmalloc申请这个tty_struct结构，如果存在一个UAF漏洞，那么就可以将该tty_struct申请为我们释放掉的一个chunk，其中重要的是int magic;和const struct tty_operations *ops;这两个结构体成员。

####   

#### magic成员

  

这个在网上很多人都直接将其设置为0，但是在某些版本中，如果直接设置为0，通常可能出现以下的错误：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

也就是magic number检测错误，这个经过调试可以发现，实际申请结构体之后是不变的：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

可以得到如下的数值：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

这个在后面的数值设置中可以用到，需要我们来调试才可以，不然其实如果直接设置为0也容易出错。

另外其中的driver可以设置为0，所以一般直接设置tty_struct[3]和tty_struct[0]即可。

####   

#### ops成员

  

这个const struct tty_operations *ops结构体指针在该做法里被劫持为我们设置fake_operations指针，有如下结构体，设置fake_operations中的write函数指针为ROP链条，就可以通过调用ptmx设备中的write函数，从而调用到我们设置的ROP链条。

```
struct tty_operations {
```

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

### 4、最终结构

####   

#### fake_tty_struct结构体

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

#### fake_tty_operation结构体

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

## POC

```
#include <sys/types.h>
```

##   

## 执行流程

  

这里还得说一下执行流程，比较不好调试。

即先是依据write函数跳转到我们最开始设置的gadget，也就是movRspRax_decEbx_ret，然后将栈劫持为fake_tty_operations。之后再跳转到pop_rax_rbx_r12_r13_rbp_ret，将ROP赋值给rax，再ret到movRspRax_decEbx_ret，再将栈劫持为ROP，之后就ret到ROP链条中的pop_rdi_ret了，之后执行流可控。

  

# ▲注意事项：

  

swapgs;ret没有时，可以用加上pop的，只要最后ret到iretq即可。同样的gadget可以相互转换。

iretq类似于ret，直接一个指令即可。

寄存器保存需要在进入内核之前。

堆块申请时的规则需要是GFP_KERNEL才行，至少GFP_DMA不行。

‍  

  

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E) 

  

**看雪ID：PIG-007**

https://bbs.pediy.com/user-home-904686.htm

*本文由看雪论坛 PIG-007 原创，转载请注明来自看雪社区

  

  

[![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458410583&idx=1&sn=fdb990e6485358004e86955cafd40a1c&chksm=b18f6edd86f8e7cb8e16efc6222fa50bfd7db02e40213370743bac836e8ca6112d30fcc5393a&scene=21#wechat_redirect)

  

[![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458409596&idx=3&sn=658012a9debc76c73efed03342d41358&chksm=b18f6af686f8e3e09180522b243def309e265532cfadfdbf5becd7d9ec90f4505136959ebdfe&scene=21#wechat_redirect)  

  

**#** **往期推荐**

1.[强网拟态线上mobile的两道wp](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458410675&idx=1&sn=e015ec78d467a2afdb3ccad3f868e6ec&chksm=b18f6e3986f8e72f921a3fc3113de57766fd66b2c46fa587fa5f5139f698b86166f0fd42c2c0&scene=21#wechat_redirect)  

2.[某钱包转账付款算法分析篇](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458410651&idx=1&sn=070a3637c7f9d78e188d2b4ca7e52c81&chksm=b18f6e1186f8e707496c08a09c6c8dd7685ef81eaca6d7e9a54305779281aca232a981e7f207&scene=21#wechat_redirect)

3.[通过PsSetLoadImageNotifyRoutine学习模块监控与反模块监控](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458410644&idx=1&sn=e36fcc9b2b7a1efffb84bc3673b248b6&chksm=b18f6e1e86f8e708e321843fbdc68404b28db00aa48fed2c974a402a7708db095f94820c3da1&scene=21#wechat_redirect)

4.[Kernel从0开始](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458410637&idx=1&sn=05a08370689a55cc0fa996954f18f172&chksm=b18f6e0786f8e711d69dc942c397be40e72fae11446f14d5673ec20d324b2a9ab2eda16c27d9&scene=21#wechat_redirect)

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

阅读 8068

​

写留言

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/1UG7KPNHN8EGLfh77kFmnicd9WOic2ibvhCibFdB4bL4srJCgo2wnvdoXLxpIvAkfCmmcptXZB0qKWMoIP8iaibYN2FA/300?wx_fmt=png&wxfrom=18)

看雪学苑

8分享5

写留言

写留言

**留言**

暂无留言