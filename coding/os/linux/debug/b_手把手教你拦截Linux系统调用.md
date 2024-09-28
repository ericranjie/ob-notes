
极客重生

 _2022年01月13日 18:12_

以下文章来源于Linux内核那些事 ，作者songsong001

[

![](http://wx.qlogo.cn/mmhead/Q3auHgzwzM6dwH9qJmWO5T0Rnsiaib5jEUibibJgLmkkG02PrbyOWab4UA/0)

**Linux内核那些事**.

以简单的方式介绍 Linux 内核的原理，以通俗的语言分析 Linux 内核的实现。如果你没有接触过 Linux 内核，那么就关注我们的公众号吧，我们将以图解的方式让内核更亲民...

](https://mp.weixin.qq.com/s?__biz=MzkyMTIzMTkzNA==&mid=2247567092&idx=1&sn=4061e264c76c430a4a4b9b4cddf34a62&chksm=c18531a5f6f2b8b3a083edc30d9ca2da558807c3afe79f7065e93f8a3e0989777801bcd54f30&mpshare=1&scene=24&srcid=01130NA47vHHHM3e5XsOs9KW&sharer_sharetime=1642070225452&sharer_shareid=5fb9813bfe9ffc983435bfc8d8c5e9ca&key=daf9bdc5abc4e8d0514216f68094454118befd839a218539defcfdf0e6b52179891a2c509ae9bd2f52b6de140c05ca49dffeb1564b11b9d6fc97325f724f3be3a623994b77d7f3ed3c39f9714f44a3f3a8a00562c4c19ea7e2b908e8e07835d2a5411af50fc3a9781b227878a768afa3dfbf37005cdc9752b7857e144d0df865&ascene=0&uin=MTEwNTU1MjgwMw%3D%3D&devicetype=Windows+11+x64&version=63090b19&lang=zh_CN&countrycode=CN&exportkey=n_ChQIAhIQVmTZeCF6dfNEWQH7DZJ1tRLmAQIE97dBBAEAAAAAANRfI254pf4AAAAOpnltbLcz9gKNyK89dVj0PxzbtniFc4MrnYlbLZxKE%2B%2FqZOeKaVmuhMEt3XiT5HC%2BDcyqAVzmcKHdFNyzKKOWL7H2%2Bhb%2B36EG%2Ft%2B%2FZZOmciNRUZORb15i8WdSfIbkKvlp0VDbwtdT6yJkOLxKA%2FNKrwsz8FtyC71uYX%2FK2eH87dUNbiecWvqRyQbCk99HSizNIaXxHfmgfcgyQ5Nw8WFba%2F%2B7nQai0qh5RvdSDvdjzwWMQAu06nRjXvuIV0gFQzzZQutzjBhLjDPHTBXpRf%2Ba&acctmode=0&pass_ticket=lHMwFoGBbebNVg0ZvJQ1gZc49ilgx0s%2FBHh4nSjuawvLCljkHgYo5a8YFLpm2LWi&wx_header=1&fasttmpl_type=0&fasttmpl_fullversion=7350504-zh_CN-zip&fasttmpl_flag=1#)

## 一、什么是系统调用

`系统调用` 是内核提供给应用程序使用的功能函数，由于应用程序一般运行在 `用户态`，处于用户态的进程有诸多限制（如不能进行 I/O 操作），所以有些功能必须由内核代劳完成。而内核就是通过向应用层提供 `系统调用`，来完成一些在用户态不能完成的工作。

说白了，系统调用其实就是函数调用，只不过调用的是内核态的函数。但与普通的函数调用不同，系统调用不能使用 `call` 指令来调用，而是需要使用 `软中断` 来调用。在 Linux 系统中，系统调用一般使用 `int 0x80` 指令（x86）或者 `syscall` 指令（x64）来调用。

下面我们以 `int 0x80` 指令（x86）调用方式为例，来说明系统调用的原理。

## 二、系统调用原理

在 Linux 内核中，使用 `sys_call_table` 数组来保存所有系统调用，`sys_call_table` 数组每一个元素代表着一个系统调用的入口，其定义如下：

```c
typedef void (*sys_call_ptr_t)(void);const sys_call_ptr_t sys_call_table[__NR_syscall_max+1] = {    ...};
```

当应用程序需要调用一个系统调用时，首先需要将要调用的系统调用号（也就是系统调用所在 `sys_call_table` 数组的索引）放置到 `eax` 寄存器中，然后通过使用 `int 0x80` 指令触发调用 `0x80` 号软中断服务。

`0x80` 号软中断服务，会通过以下代码来调用系统调用，如下所示：

```c
...call *sys_call_table(,%eax,8)...
```

上面的代码会根据 `eax` 寄存器中的值来调用正确的系统调用，其过程如下图所示：
![[Pasted image 20240928195526.png]]
![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

整体流程  

  
![[Pasted image 20240928195532.png]]
![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

## 三、系统调用拦截

了解了系统调用的原理后，要拦截系统调用就很简单了。那么如何拦截呢？

做法就是：我们只需要把 `sys_call_table` 数组的系统调用换成我们自己编写的函数入口即可。比如，我们想要拦截 `write()` 系统调用，那么只需要将 `sys_call_table` 数组的第一个元素换成我们编写好的函数（因为 `write()` 系统调用在 `sys_call_table` 数组的索引为1）。

要修改 `sys_call_table` 数组元素的值，步骤如下：

### 1. 获取 `sys_call_table` 数组的地址

> 要修改 `sys_call_table` 数组元素的值，一般需要通过内核模块来完成。因为用户态程序由于内存保护机制，不能改写内核态的数据。而内核模块运行在内核态，所以能够跳过这个限制。

要修改 `sys_call_table` 数组元素的值，首先要获取 `sys_call_table` 数组的虚拟内存地址（由于 `sys_call_table` 变量不是一个导出符号，所以内核模块不能直接使用）。

要获取 `sys_call_table` 数组的虚拟内存地址有两种方法：

#### 第一种方法：从 `System.map` 文件中读取

`System.map` 是一份内核符号表，包含了内核中的变量名和函数名地址，在每次编译内核时，自动生成。获取 `sys_call_table` 数组的虚拟地址使用如下命令：

```c
sudo cat /boot/System.map-`uname -r` | grep sys_call_table
```

结果如下图所示：  
  
![[Pasted image 20240928195543.png]]
![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  
从上图可知，`sys_call_table` 数组的虚拟地址为：`ffffffff818001c0`。

#### 第二种方法：通过 `kallsyms_lookup_name()` 函数来获取

从 `System.map` 文件中读取的方法不是很优雅，所以内核提供了一个名为 `kallsyms_lookup_name()` 的函数来获取内核变量和内核函数的虚拟内存地址。

`kallsyms_lookup_name()` 函数的使用很简单，只需要传入要获取虚拟内存地址的变量名即可，如下代码所示：

```c
#include <linux/kallsyms.h>void func() {    ...    unsigned long *sys_call_table;    // 获取 sys_call_table 的虚拟内存地址    
sys_call_table = (unsigned long *)kallsyms_lookup_name("sys_call_table");    ...}
```

### 2. 设置 sys_call_table 数组为可写状态

是不是获取到 `sys_call_table` 数组的虚拟地址就可以修改其元素的值呢？没那么简单。

由于 `sys_call_table` 数组处于写保护区域，并不能直接修改其内容。但有两种方法可以将写保护暂时关闭，如下：

#### 第一种方法：将 `cr0` 寄存器的第 16 位设置为零

`cr0` 控制寄存器的第 16 位是写保护位，若设置为零，则允许超级权限往内核中写入数据。这样我们可以在修改 `sys_call_table` 数组的值前，将 `cr0` 寄存器的第 16 位清零，使其可以修改 `sys_call_table` 数组的内容。当修改完后，又将那一位复原即可。

代码如下：

```c
/* * 设置cr0寄存器的第16位为0 */unsigned int clear_and_return_cr0(void){    unsigned int cr0 = 0;    unsigned int ret;    /* 将cr0寄存器的值移动到rax寄存器中，同时输出到cr0变量中 */    asm volatile ("movq %%cr0, %%rax" : "=a"(cr0));    ret = cr0;    cr0 &= 0xfffeffff;  /* 将cr0变量值中的第16位清0，将修改后的值写入cr0寄存器 */    /* 读取cr0的值到rax寄存器，再将rax寄存器的值放入cr0中 */    asm volatile ("movq %%rax, %%cr0" :: "a"(cr0));    return ret;}/* * 还原cr0寄存器的值为val */void setback_cr0(unsigned int val){    asm volatile ("movq %%rax, %%cr0" :: "a"(val));}
```

#### 第二种方法：设置虚拟地址对应页表项的读写属性

由于 `x86 CPU` 的内存保护机制是通过虚拟内存页表来实现的（可以参考这篇文章：[漫谈内存映射](https://mp.weixin.qq.com/s?__biz=MzA3NzYzODg1OA==&mid=2648464745&idx=1&sn=374b770823d0a9e9677386160e57f71c&scene=21#wechat_redirect)），所以我们只需要把 `sys_call_table` 数组的虚拟内存页表项中的保护标志位清空即可，代码如下：

```c
/* * 把虚拟内存地址设置为可写 */int make_rw(unsigned long address){    unsigned int level;    //查找虚拟地址所在的页表地址    pte_t *pte = lookup_address(address, &level);    if (pte->pte & ~_PAGE_RW)  //设置页表读写属性        pte->pte |=  _PAGE_RW;    return 0;}/* * 把虚拟内存地址设置为只读 */int make_ro(unsigned long address){    unsigned int level;    pte_t *pte = lookup_address(address, &level);    pte->pte &= ~_PAGE_RW;  //设置只读属性    return 0;}
```

### 3. 修改 `sys_call_table` 数组的内容

万事俱备，只欠东风。前面我们把准备工作都做完了，现在只需要把 `sys_call_table` 数组中的系统调用入口替换成我们编写的函数入口即可。

我们可以在内核模块初始化函数修改 `sys_call_table` 数组的值，然后在内核模块退出函数改回成原来的值即可，完整代码如下：

```c
/* * File: syscall.c */#include <linux/module.h>#include <linux/kernel.h>#include <linux/init.h>#include <linux/unistd.h>#include <linux/time.h>#include <asm/uaccess.h>#include <linux/sched.h>#include <linux/kallsyms.h>
unsigned long *sys_call_table;unsigned int clear_and_return_cr0(void);void setback_cr0(unsigned int val);static int sys_hackcall(void);unsigned long *sys_call_table = 0;/* 定义一个函数指针，用来保存原来的系统调用*/static int (*orig_syscall_saved)(void);/* * 设置cr0寄存器的第16位为0 */unsigned int clear_and_return_cr0(void){    unsigned int cr0 = 0;    unsigned int ret;    /* 将cr0寄存器的值移动到rax寄存器中，同时输出到cr0变量中 */    asm volatile ("movq %%cr0, %%rax" : "=a"(cr0));    ret = cr0;    cr0 &= 0xfffeffff;  /* 将cr0变量值中的第16位清0，将修改后的值写入cr0寄存器 */    /* 读取cr0的值到rax寄存器，再将rax寄存器的值放入cr0中 */    asm volatile ("movq %%rax, %%cr0" :: "a"(cr0));    return ret;}/* * 还原cr0寄存器的值为val */void setback_cr0(unsigned int val){    asm volatile ("movq %%rax, %%cr0" :: "a"(val));}/* * 自己编写的系统调用函数 */static int sys_hackcall(void){    printk("Hack syscall is successful!!!\n");    return 0;}/* * 模块的初始化函数，模块的入口函数，加载模块时调用 */static int __init init_hack_module(void){    int orig_cr0;    printk("Hack syscall is starting...\n");    /* 获取 sys_call_table 虚拟内存地址 */    sys_call_table = (unsigned long *)kallsyms_lookup_name("sys_call_table");    /* 保存原始系统调用 */    orig_syscall_saved = (int(*)(void))(sys_call_table[__NR_perf_event_open]);    orig_cr0 = clear_and_return_cr0(); /* 设置cr0寄存器的第16位为0 */    sys_call_table[__NR_perf_event_open] = (unsigned long)&sys_hackcall; /* 替换成我们编写的函数 */    setback_cr0(orig_cr0); /* 还原cr0寄存器的值 */    return 0;}/* * 模块退出函数，卸载模块时调用 */static void __exit exit_hack_module(void){    int orig_cr0;    orig_cr0 = clear_and_return_cr0();    sys_call_table[__NR_perf_event_open] = (unsigned long)orig_syscall_saved; /* 设置为原来的系统调用 */    setback_cr0(orig_cr0);    printk("Hack syscall is exited....\n");}module_init(init_hack_module);module_exit(exit_hack_module);MODULE_LICENSE("GPL");
```

在上面代码中，我们将 `perf_event_open()` 系统调用替换成了我们自己实现的函数。

> 注意：测试时最好使用冷门的系统调用，否则可能会导致系统崩溃。

### 4. 编写 Makefile 文件

为了编译方便，我们编写一个 Makefile 文件来进行编译，如下所示：

```c
obj-m:=syscall.oPWD:= $(shell pwd)KERNELDIR:= /lib/modules/$(shell uname -r)/buildEXTRA_CFLAGS= -O0all:    make -C $(KERNELDIR)  M=$(PWD) modulesclean:    make -C $(KERNELDIR) M=$(PWD) clean
```

要注意添加 `EXTRA_CFLAGS= -O0` 关闭 gcc 优化选项，避免插入模块出错。

### 5. 测试程序

现在，我们编写一个测试程序来测试一下系统调用拦截是否成功，代码如下：

```c
#include <syscall.h>#include <stdio.h>#include <unistd.h>
int main(void){    unsigned long ret = syscall(__NR_perf_event_open, NULL, 0, 0, 0, 0);    printf("%d\n", (int)ret);    return 0;}
```

### 6. 运行结果

#### 第一步：安装拦截内核模块

使用以下命令安装内核模块：

```c
root# insmod syscall.ko
```

然后通过 `dmesg` 命令来观察系统日志，可以看到以下输出：

```c
...[  133.564652] Hack syscall is starting...
```

这说明我们的内核模块安装成功。

#### 第二步：运行测试程序

接着，我们运行刚才编写的测试程序，然后观察系统日志，输出如下：

```c
...[  532.243714] Hack syscall is successful!!!
```

这说明拦截系统调用成功了。

![](http://mmbiz.qpic.cn/mmbiz_png/ciab8jTiab9J6dgBc9aqzhEyz7LkUJ812dSOibgAHcHicR8zE8PyD3bvkyicjTSGfFsF1racDTDviayU3Mbcra30sacw/300?wx_fmt=png&wxfrom=19)

**Linux内核那些事**

以简单的方式介绍 Linux 内核的原理，以通俗的语言分析 Linux 内核的实现。如果你没有接触过 Linux 内核，那么就关注我们的公众号吧，我们将以图解的方式让内核更亲民...

115篇原创内容

公众号

  

- END -

---

  

**看完一键三连********在看********，**转发****，********点赞****

**是对文章最大的赞赏，极客重生感谢你****![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)**

  

推荐阅读

  

  

[

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

大厂后台开发基本功修炼路线和经典资料







](https://mp.weixin.qq.com/s?__biz=MzkyMTIzMTkzNA==&mid=2247566986&idx=1&sn=40c3846a5a328ba18b73a16d329b756f&chksm=c18531dbf6f2b8cd4a93bd9aa5cad4020daf8da7a54156f754f5c2b403c58b02b40a79ed79f4&scene=21#wechat_redirect)

  

[

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

2022新年重磅技术分享|深入理解Linux操作系统







](http://mp.weixin.qq.com/s?__biz=MzkyMTIzMTkzNA==&mid=2247566121&idx=1&sn=a169fb93b7f961d8409ef4494ddd1735&chksm=c1853478f6f2bd6ee1a675d10eede0c2a435380729d9a534eff1ea6dcf2a5d2263dfd7979c11&scene=21#wechat_redirect)

  

你好，这里是极客重生，我是阿荣，大家都叫我荣哥，从华为->外企->到互联网大厂，目前是大厂资深工程师，多次获得五星员工，多年职场经验，技术扎实，专业后端开发和后台架构设计，热爱底层技术，丰富的实战经验，分享技术的本质原理，希望帮助更多人蜕变重生，拿BAT大厂offer，培养高级工程师能力，成为技术专家，实现高薪梦想，期待你的关注！[**点击蓝字查看我的成长之路**。](http://mp.weixin.qq.com/s?__biz=MzkyMTIzMTkzNA==&mid=2247564006&idx=1&sn=8c88b0d7222ce0e310a012e90bff961f&chksm=c1850db7f6f284a133be27ad27ecd965c3c942583f69bd1a26c34ebf179b4cd6ceb3d7e787b4&scene=21#wechat_redirect)

  

校招/社招/简历/面试技巧/大厂技术栈分析/后端开发进阶/优秀开源项目/直播分享/技术视野/实战高手等, [极客星球](http://mp.weixin.qq.com/s?__biz=MzkyMTIzMTkzNA==&mid=2247563979&idx=1&sn=b26f56f933d9c9e2fd1fe3260362d6e5&chksm=c1850d9af6f2848cf5e3282aa6464b79fc32d848bb8d8155f317c3be7a021a19ac937f8dd10b&scene=21#wechat_redirect)希望成为最有技术价值星球，尽最大努力为星球的同学提供技术和成长帮助！详情查看->[极客星球](http://mp.weixin.qq.com/s?__biz=MzkyMTIzMTkzNA==&mid=2247563979&idx=1&sn=b26f56f933d9c9e2fd1fe3260362d6e5&chksm=c1850d9af6f2848cf5e3282aa6464b79fc32d848bb8d8155f317c3be7a021a19ac937f8dd10b&scene=21#wechat_redirect)  

  

                                                                求点赞，在看，分享三连![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

阅读 2414

​