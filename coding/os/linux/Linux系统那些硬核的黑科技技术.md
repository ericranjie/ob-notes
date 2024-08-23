# 

极客重生

 _2022年04月24日 18:09_

以下文章来源于Thoughtworks洞见 ，作者王张军

[

![](http://wx.qlogo.cn/mmhead/Q3auHgzwzM7AW9Hxz0PD2AyzrJjukQo8qHDaOta0plZmYZxMxU6U7w/0)

**Thoughtworks洞见**.

Thoughtworks是一家全球性的软件及咨询公司，我们致力于通过整合战略、设计和软件工程帮助企业开启流畅数字化之路，引航未来征程。

](https://mp.weixin.qq.com/s?__biz=MzkyMTIzMTkzNA==&mid=2247580328&idx=1&sn=a2559ef9dfee6b397730d49a6fb1b909&chksm=c1854df9f6f2c4ef433684ecb4b5c8890764df46d63c8474a6da1c09e6bc2846663df5bf903a&mpshare=1&scene=24&srcid=0424MZNppezGgiMJyn28iv75&sharer_sharetime=1650808214324&sharer_shareid=8397e53ca255d0bca170c6327d62b9af&key=daf9bdc5abc4e8d09fc8b2605c348ed71c6d6c1c706a95d219a2023be3eab1cd9be407a397369aa06ede5c22b3ccab3539c130adc71401eaaedc5ae43f5a422304fe2a08cf41884c1e3d15adb00d24a086b818a5c60e994a50bce915a7df099ccd8df77fc9026a48d5e8cc2aa9ebb157b60e914156b44dd7b5b779652c747e8a&ascene=0&uin=MTEwNTU1MjgwMw%3D%3D&devicetype=Windows+11+x64&version=63090b19&lang=zh_CN&countrycode=CN&exportkey=n_ChQIAhIQrlRELok9t7wriahUH%2BAU0RLmAQIE97dBBAEAAAAAAPZmMvfQNq4AAAAOpnltbLcz9gKNyK89dVj0lvGSg6iT3aJ7BtjUpDH89fNFHBsNZ6LhCowTTbVo7T4FNhshUkrbKkOMRcQXudI6fWnHUWOt3nZNlKkgSnG3ea34c1pQ46RaOOwAe1UxvbXw4VUSRW9SrLFa4v%2F7%2FNlIbrMFKRWzNVF4RV5Hkb6RxHA174LszJ9wTEWWun9u%2BZow5FHJrfuZnEW5uerEldK9VvEaFNYLNfc%2BrpSyHvrm5IiPlmvs04eSMytPIXKfAKPGff9c3DV6v4fRfw5yQv0x&acctmode=0&pass_ticket=esT5rNLpzIV%2FfZLlaYZ4fdaZNOMHmqh2L8qx08SRqqyq13M0M9UJWBQo2snErBh0&wx_header=1&fasttmpl_type=0&fasttmpl_fullversion=7351805-zh_CN-zip&fasttmpl_flag=1#)

很早之前，我在极客星球提出一个有趣的问题：

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/cYSwmJQric6nGPIcZJm6L8wxT4YZNwY6Ir0sCFSuMicgeZWebUQNABMc7gzGnI1Ad0xiaTqM6MahqtNkXOBpOZSaA/640?wx_fmt=jpeg&wxfrom=13&tp=wxpic)

这个问题涉及到类似上网行为管理，但大多数行为管理都是在windows端, 我们这里讨论微信和邮件运行在Linux系统，以后很有可能我们的办公都是在国产化操作系统之上:  

![图片](https://mmbiz.qpic.cn/mmbiz_png/cYSwmJQric6nGPIcZJm6L8wxT4YZNwY6IzkqomfSKPrnuV7ssrRETjpIhTpHpuZsVDiaiaz4lVg0HzHJ0dRzyF6yw/640?wx_fmt=png&wxfrom=13&tp=wxpic)

这里依赖相关技术差不多就是很多hacker所掌握一些**黑科技技术**，而这些技术其实在我们开发中偶尔也会用到，一般都是一些资深技术大牛掌握这些技术，如果能够精通这些技术，年薪至少百万以上 ，对于小白来说，只要爱学习和多花时间实践，也可以学会，并不是什么特别高深知识（相对于高等数学和量子力学等）. 今天这里我给大家分享一些Linux系统上重要的hook技术，大家先熟悉一下这些基础技术，后面我会针对这些技术，对上面提出的问题给出一些解决方案，不管学什么，**先熟悉和掌握依赖的基础知识**。

**正文**

### **背景**

前段时间，我们的项目组在帮客户解决一些操作系统安全领域的问题，涉及到windows，Linux，macOS三大操作系统平台。无论什么操作系统，本质上都是一个软件，任何软件在一开始设计的时候，都不能百分之百的满足人们的需求，所以操作系统也是一样，为了尽可能的满足人们需求，不得不提供一些供人们定制操作系统的机制。当然除了官方提供的一些机制，也有一些黑魔法，这些黑魔法不被推荐使用，但是有时候面对具体的业务场景，可以作为一个参考的思路。

---

**Linux中常见的拦截过滤**

本文着重介绍Linux平台上常见的拦截：

1. 用户态动态库拦截。
    
2. 内核态系统调用拦截。
    
3. 堆栈式文件系统拦截。
    
4. inline hook拦截。
    
5. LSM(Linux Security Modules)
    
6. eBPF Hook拦截。（本文新增）
    

---

**动态库劫持**

Linux上的动态库劫持主要是基于LD_PRELOAD环境变量，这个环境变量的主要作用是改变动态库的加载顺序，让用户有选择的载入不同动态库中的相同函数。但是使用不当就会引起严重的安全问题，我们可以通过它在主程序和动态连接库中加载别的动态函数，这就给我们提供了一个机会，向别人的程序注入恶意的代码。

假设有以下用户名密码验证的函数：

#include <stdio.h>  
#include <string.h>  
#include <stdlib.h>  
int main(int argc, char **argv)  
{  
char passwd[] = "password";  
if (argc < 2) {  
printf("Invalid argc!\n");  
return;  
}  
if (!strcmp(passwd, argv[1])) {  
printf("Correct Password!\n");  
return;  
}  
printf("Invalid Password!\n");  
}  

我们再写一段hookStrcmp的程序，让这个比较永远正确。

#include <stdio.h>  
int strcmp(const char *s1, const char *s2)  
{  
/* 永远返回0，表示两个字符串相等 */  
return 0;  
}

依次执行以下命令，就会使我们的hook程序先执行。

gcc -Wall -fPIC -shared -o hookStrcmp.so hookStrcmp.c  
export LD_PRELOAD=”./hookStrcmp.so”

结果会发现，我们自己写的strcmp函数优先被调用了。这是一个最简单的劫持 ，但是如果劫持了类似于geteuid/getuid/getgid，让其返回0，就相当于暴露了root权限。所以为了安全起见，一般将LD_PRELOAD环境变量禁用掉。  

---

**Linux系统调用劫持**

最近发现在4.4.0的内核中有513多个系统调用(很多都没用过)，系统调用劫持的目的是改变系统中原有的系统调用，用我们自己的程序替换原有的系统调用。Linux内核中所有的系统调用都是放在一个叫做sys_call_table的内核数组中，数组的值就表示这个系统调用服务程序的入口地址。整个系统调用的流程如下：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

当用户态发起一个系统调用时，会通过80软中断进入到syscall hander，进而进入全局的系统调用表sys_call**_**table去查找具体的系统调用，那么如果我们将这个数组中的地址改成我们自己的程序地址，就可以实现系统调用劫持。但是内核为了安全，对这种操作做了一些限制：

1. sys_call_table的符号没有导出，不能直接获取。
    
2. sys_call_table所在的内存页是只读属性的，无法直接进行修改。
    

对于以上两个问题，解决方案如下（方法不止一种）：

1. 获取sys_call_table的地址 ：grep sys_call_table /boot/System.map-uname -r
    
2. 控制页表只读属性是由CR0寄存器的WP位控制的，只要将这个位清零就可以对只读页表进行修改。
    

/* make the page writable */  
int make_rw(unsigned long address)  
{  
unsigned int level;  
pte_t *pte = lookup_address(address, &level);//查找虚拟地址所在的页表地址  
pte->pte |= _PAGE_RW;//设置页表读写属性  
return 0;  
}

/* make the page write protected */  
int make_ro(unsigned long address)  
{  
unsigned int level;  
pte_t *pte = lookup_address(address, &level);  
pte->pte &= ~_PAGE_RW;//设置只读属性  
return 0;  
}

#### 开始替换系统调用  

本文实现的是对 ls这个命令对应的系统调用，系统调用号是__NR_getdents。

static int syscall_init_module(void)  
{  
orig_getdents = sys_call_table[__NR_getdents];  
make_rw((unsigned long)sys_call_table); //修改页属性  
sys_call_table[__NR_getdents] = (unsigned long *)hacked_getdents; //设置新的系统调用地址  
make_ro((unsigned long)sys_call_table);  
return 0;  
}

#### 恢复原状

static void syscall_cleanup_module(void)  
{  
printk(KERN_ALERT "Module syscall unloaded.\n");  
make_rw((unsigned long)sys_call_table);  
sys_call_table[__NR_getdents] = (unsigned long *)orig_getdents;  
make_ro((unsigned long)sys_call_table);  
}

使用Makefile编译，insmod插入内核模块后，再执行ls时，就会进入到我们的系统调用，我们可以在hook代码中删掉某些文件，ls就不会显示这些文件，但是这些文件还是存在的。

---

**堆栈式文件系统**

Linux通过vfs虚拟文件系统来统一抽象具体的磁盘文件系统，从上到下的IO栈形成了一个堆栈式。通过对内核源码的分析，以一次读操作为例，从上到下所执行的流程如下：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

内核中采用了很多c语言形式的面向对象，也就是函数指针的形式，例如read是vfs提供用户的接口，具体底下调用的是ext2的read操作。我们只要实现VFS提供的各种接口，就可以实现一个堆栈式文件系统。Linux内核中已经集成了一些堆栈式文件系统，例如Ubuntu在安装时会提醒你是否需要加密home目录，其实就是一个堆栈式的加密文件系统（eCryptfs），原理如下：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

实现了一个堆栈式文件系统，相当于所有的读写操作都会进入到我们的文件系统，可以拿到所有的数据，就可以进行做一些拦截过滤。

以下是我实现的一个最简单的堆栈式文件系统，实现了最简单的打开、读写文件，麻雀虽小但五脏俱全。

https://github.com/wangzhangjun/wzjfs

---

**inline hook**

我们知道内核中的函数不可能把所有功能都在这个函数中全部实现，它必定要调用它的下层函数。如果这个下层函数可以得到我们想要的过滤信息内容，就可以把下层函数在上层函数中的offset替换成新的函数的offset，这样上层函数调用下层函数时，就会跳到新的函数中，在新的函数中做过滤和劫持内容的工作。所以从原理上来说，inline hook可以想hook哪里就hook哪里。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

#### inline hook 有两个重要的问题：

1. 如何定位hook点。
    
2. 如何注入hook函数入口。
    

#### 对于第一个问题:

需要有一点的内核源码经验，比如说对于read操作，源码如下：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

在这里当发起read系统调用后，就会进入到sys_read_,在_sys___read中会调用vfs_read_函数，在_vfs___read的参数中正好有我们需要过滤的信息，那么就可以把vfs_read当做一个hook点。

#### 对于第二个问题:

如何Hook？这里介绍两种方式：

**第一种方式：**直接进行二进制替换，将call指令的操作数替换为hook函数的地址。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**第二种方式：**Linux内核提供的kprobes机制。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

其原理是在hook点注入int 3(x86)的机器码，让cpu运行到这里的时候会触发sig_trap信号_，_然后将用户自定义的hook函数注入到sig_trap的回调函数中，达到触发hook函数的目的。这个其实也是调试器的原理。  

---

**LSM**

LSM是Linux Secrity Module的简称，即linux安全模块。是一种通用的Linux安全框架，具有效率高，简单易用等特点。原理如下：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

LSM在内核中做了以下工作：

1. 在特定的内核数据结构中加入安全域。
    
2. 在内核源代码中不同的关键点插入对安全钩子函数的调用。
    
3. 加入一个通用的安全系统调用。
    
4. 提供了函数允许内核模块注册为安全模块或者注销。
    
5. 将capabilities逻辑的大部分移植为一个可选的安全模块,具有可扩展性。
    

---

**ebpf**

**eBPF 是一个在内核中运行的虚拟机，它可以去运行用户。在用户态实现的这种 eBPF 的代码，在内核以本地代码的形式和速度去执行，它可以跟内核的 Trace 系统相结合，给我们提供了在线扩展内核功能的技术****。**

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**从图中可以看出，eBPF的hook点功能包括以下几部分：**

1. **可以在Storage、Network等与内核交互之间；**
    
2. **也可以在内核中的功能模块交互之间；**
    
3. **又可以在内核态与用户态交互之间；**
    
4. **更可以在用户态进程空间。**
    

  

**eBPF的功能覆盖XDP、TC、Probe、Socket等，每个功能点都能实现内核****态的篡改行为，从而使得用户态完全致盲，哪怕是基于内核模块的HIDS，一样无法感知到这些行为。针对文件操作，如果不存在用于特定需求的预定义挂钩，**

**则可以创建内核探针 (kprobe) 或用户探针 (uprobe) ，可以在内核或用户应用程序的几乎任何位置附加 eBPF 程序。我们可以针对关键函数进行hook：**

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

更多参考：  

  

[Linux网络新技术基石 |eBPF and XDP](http://mp.weixin.qq.com/s?__biz=MzkyMTIzMTkzNA==&mid=2247511570&idx=1&sn=18d5f1045b7ed6e27e1c8fb36302c3f9&chksm=c1845943f6f3d055c2533d580acb2d4258daf2a84ffba30e2cc6376b3ffdb1687773ed4c19a3&scene=21#wechat_redirect)  

  

[eBPF在大厂的应用](http://mp.weixin.qq.com/s?__biz=MzkyMTIzMTkzNA==&mid=2247575154&idx=1&sn=d918621452b74db082ff80727dc474c1&chksm=c1855123f6f2d83535a6b95fa57d8f3236121ea2f27a4c3138f124fc97a31097d9e5ca717440&scene=21#wechat_redirect)  

  

---

**适用场景**

对于以上几种Hook方式，有其不同的应用场景。

1. 动态库劫持不太完全，劫持的信息有可能满足不了我们的需求，还有可能别人在你之前劫持了，一旦禁用LD_PRELOAD就失效了。
    
2. 系统调用劫持，劫持的信息有可能满足不了我们的需求，例如不能获取struct file结构体，不能获取文件的绝对路径等。
    
3. 堆栈式文件系统，依赖于Mount,可能需要重启系统。
    
4. inline hook，灵活性高，随意Hook，即时生效无需重启，但是在不同内核版本之间通用性差，一旦某些函数发生了变化，Hook失效。
    
5. LSM，在早期的内核中，只能允许一个LSM内核模块加载，例如加载了SELinux，就不能加载其他的LSM模块，在最新的内核版本中不存在这个问题。
    
6. eBPF Hook,可以用户态编程控制，灵活性高，即时生效无需重启，但依赖内核的版本，通用性差，一旦某些函数发生了变化，Hook失效。
    

---

**总结**

篇幅有限，本文只是介绍了Linux上的拦截技术，事实上类似的审计HOOK放到任何一个系统中都是刚需，不只是kernel，我们可以看到越来越多的vm和runtime甚至包括很多web组件、前端应用都提供了更灵活的hook方式，这是透明化和实时性两个安全大趋势下最常见的解决方案。

---

掌握后台核心技术，深入理解Linux系统，基础概念深入理解，基本功修炼，疑难解答，欢迎大家加入极客星球，我们一起进步，学习是一辈子的事情，长期坚持学习，定能掌握核心技术，对星球感兴趣的,点击查看-> [极客星球](http://mp.weixin.qq.com/s?__biz=MzkyMTIzMTkzNA==&mid=2247577773&idx=1&sn=31254c3ba0a21956b46cdf63b471468f&chksm=c18547fcf6f2ceea54066e257a45e43da2bad5db23a7940618356f5e361878d4055f8c874cb7&scene=21#wechat_redirect)：  

[

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

进腾讯了｜学习技术哪家强







](https://mp.weixin.qq.com/s?__biz=MzkyMTIzMTkzNA==&mid=2247577773&idx=1&sn=31254c3ba0a21956b46cdf63b471468f&chksm=c18547fcf6f2ceea54066e257a45e43da2bad5db23a7940618356f5e361878d4055f8c874cb7&scene=21#wechat_redirect)

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)  

- END -

---

  

**看完一键三连********在看********，**转发****，********点赞****

**是对文章最大的赞赏，极客重生感谢你****![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)**

  

推荐阅读

  

[

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

定个目标｜建立自己的技术知识体系







](https://mp.weixin.qq.com/s?__biz=MzkyMTIzMTkzNA==&mid=2247570933&idx=1&sn=04bf666a7a402ed65b83020dc28dd543&chksm=c18522a4f6f2abb27ca1d09eaf646cc7eca2feea9c7f39618d3048b87634695955a8831992a9&scene=21#wechat_redirect)

  

[

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

大厂后台开发基本功修炼路线和经典资料







](https://mp.weixin.qq.com/s?__biz=MzkyMTIzMTkzNA==&mid=2247566986&idx=1&sn=40c3846a5a328ba18b73a16d329b756f&chksm=c18531dbf6f2b8cd4a93bd9aa5cad4020daf8da7a54156f754f5c2b403c58b02b40a79ed79f4&scene=21#wechat_redirect)

[

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

难走的路，从不拥挤







](https://mp.weixin.qq.com/s?__biz=MzkyMTIzMTkzNA==&mid=2247578173&idx=1&sn=2be47a3acca9ff1a56a071b58919ebd2&chksm=c185456cf6f2cc7a4659e65ded956cc691a97a65b05cec370dc209329595ae5305918982ba28&scene=21#wechat_redirect)

  

你好，这里是极客重生，我是阿荣，大家都叫我荣哥，从华为->外企->到互联网大厂，目前是大厂资深工程师，多次获得五星员工，多年职场经验，技术扎实，专业后端开发和后台架构设计，热爱底层技术，丰富的实战经验，分享技术的本质原理，希望帮助更多人蜕变重生，拿BAT大厂offer，培养高级工程师能力，成为技术专家，实现高薪梦想，期待你的关注！[**点击蓝字查看我的成长之路**。](http://mp.weixin.qq.com/s?__biz=MzkyMTIzMTkzNA==&mid=2247564006&idx=1&sn=8c88b0d7222ce0e310a012e90bff961f&chksm=c1850db7f6f284a133be27ad27ecd965c3c942583f69bd1a26c34ebf179b4cd6ceb3d7e787b4&scene=21#wechat_redirect)

  

校招/社招/简历/面试技巧/大厂技术栈分析/后端开发进阶/优秀开源项目/直播分享/技术视野/实战高手等, [极客星球](http://mp.weixin.qq.com/s?__biz=MzkyMTIzMTkzNA==&mid=2247577773&idx=1&sn=31254c3ba0a21956b46cdf63b471468f&chksm=c18547fcf6f2ceea54066e257a45e43da2bad5db23a7940618356f5e361878d4055f8c874cb7&scene=21#wechat_redirect)希望成为最有技术价值星球，尽最大努力为星球的同学提供面试，跳槽，技术成长帮助！详情查看->[极客星球](http://mp.weixin.qq.com/s?__biz=MzkyMTIzMTkzNA==&mid=2247577773&idx=1&sn=31254c3ba0a21956b46cdf63b471468f&chksm=c18547fcf6f2ceea54066e257a45e43da2bad5db23a7940618356f5e361878d4055f8c874cb7&scene=21#wechat_redirect)  

  

![](http://mmbiz.qpic.cn/mmbiz_png/cYSwmJQric6m7W7KicZrc94icQe4d8ItHFyONKlBkGVqEAiavUicEzmEszR5aPvicKDeCMy8aAw6lPFe8AGhHQic1UKaA/300?wx_fmt=png&wxfrom=19)

**极客重生**

技术学习分享，一起进步

180篇原创内容

公众号

                                                                求点赞，在看，分享三连![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

阅读 2583

修改于2022年04月24日

​

写留言

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/cYSwmJQric6m7W7KicZrc94icQe4d8ItHFyONKlBkGVqEAiavUicEzmEszR5aPvicKDeCMy8aAw6lPFe8AGhHQic1UKaA/300?wx_fmt=png&wxfrom=18)

极客重生

15分享11

写留言

写留言

**留言**

暂无留言