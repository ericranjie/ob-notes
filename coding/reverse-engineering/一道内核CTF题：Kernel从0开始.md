PIG-007 看雪学苑

_2022年01月05日 17:59_

!\[\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

本文为看雪论坛优秀文章\
看雪论坛作者ID：PIG-007

一

**多线程的真条件竞争**

先给出自己编写的题目源码，参照多方题目，主要还是0CTF2018-baby。

```
#include <linux/module.h>
```

### 

### 前置知识

主要是线程的知识。

就是如果一个变量是全局的，那么在没有锁的情况就会导致一个程序的不同线程对该全局变量进行的竞争读写操作。就像这里给出的代码，首先flag在内核模块中是静态全局的：

```
static char* flag = "flag{PIG007NBHH}";
```

### 

### 1、检测：

先检测传入的数据的地址是否是在用户空间，长度是否为flag的长度，传入的所有数据是否处在用户空间。如果都是，再判断传入的数据与flag是否一致，一致则打印flag，否则打印Wrong然后退出。

这么一看好像无法得到flag，首先flag我们不知道，是硬编码在内核模块中的。其次就算无法传入内核模块中的地址，意味着就算我们获得了内核模块中flag的地址和长度，传进去也会判定失败。

### 2、漏洞

但是这是建立在一个线程中的，如果是多个线程呢。在我们传入用户空间的某个数据地址和长度之后，先进入程序的if语句，也就是进入如下if语句。

```
if(chk_range_not_ok(userFlagAddr,flagObj->len)
```

然后在检测flag数据之前，也就是如下if语句之前，启动另外一个线程把传入的该数据地址给改成内核模块的flag的数据地址，这样就能成功打印flag了。

```
if(!strncmp(flagObj->flagUser,flag,strlen(flag)))
```

那么就直接给出POC，这里涉及到一些线程的操作，可以自己学习一下。

### 

### POC

```
// gcc -static exp.c -lpthread -o exp
```

最后如下效果：

!\[\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

需要注意的是在gcc编译的时候需要加上-lpthread多线程的参数。还有线程的回调函数必须传入至少一个参数，不管该参数在里面有没有被用到。

使用userfaultfd还不太会，之后再补。

二

**任意读写漏洞**

内核里的读写漏洞利用方式与用户态有点不太一样，利用方式也是多种多样。

## 

## 题目

首先给出下题目：

```
#include <linux/module.h>
```

可以看到传入一个结构体，包含数据指针和地址，直接任意读写。

## 

## 不同劫持

### 

### 1、劫持vdso

#### 

#### (1)前置知识

vdso的代码数据在内核里，当程序需要使用时，会将vdso的内核映射给进程，也就是相当于把内核空间的vdso原封不动地复制到用户空间，然后调用在用户空间的vdso代码。

如果我们能够将vdso给劫持为shellcode，那么当具有root权限的程序调用vdso时，就会触发我们的shellcode，而具有root权限的shellcode可想而知直接就可以。而vdso是经常会被调用的，所有只要我们劫持了vdso，大概率都会运行到我们的shellcode。

真实环境中crontab会调用vdso中的gettimeofday，且是root权限的调用。

而ctf题目中就通常可以用一个小程序来模拟调用。

```
#include <stdio.h> 
```

将这个程序编译后放到init中，用nohup挂起，即可做到root权限调用gettimeofday：‍

```
nohup /gettimeofday &
```

#### 

#### (2)获取地址

由于不同版本的linux内核中的vdso偏移不同，而题目给的vmlinux通常又没有符号表，所以需要我们自己利用任意读漏洞来测量。(如果题目没有任意读漏洞，建议可以自己编译一个对应版本的内核，然后自己写一个具备任意读的模块，加载之后测量即可，或者进行爆破，一般需要爆破一个半字节)

例如这里给出的代码示例，就可以通过如下代码来将vdso从内核中dump下来。不过这种方式dump的是映射到用户程序的vdso，虽然内容是一样的，不过vdso在内核中的偏移却没有办法确定。

```
#include <stdio.h>
```

这种方法的原理是通过寻找gettimeofday字符串来得到映射到程序中的vdso的内存页，如下：

!\[\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

之后把下面的内容都复制，放到CyberChef中，利用From Hex功能得到二进制文件。

!\[\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

然后就将得到的文件放到IDA中，即可自动解析到对应gettimeofday函数相对于vdso的函数偏移。

!\[\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

这里就当是0xb20。

之后通过任意读，类似基于本题编写的以下代码，获取vdso相对于vmlinux基址的偏移。

```
#include <string.h>
```

之后得到具体的地址。

!\[\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

那么最终的gettimeofday相对于vmlinux基地址就是：

```
gettimeofday_addr = 0xffffffff81e04b20+0xb20
```

#### 

#### (3)任意写劫持gettimeofday

最终利用任意写，将gettimeofday函数的内容改为我们的shellcode即可。

#### 

#### POC

```
// gcc -static exp.c -o exp
```

参考：

(15条消息) linux kernel pwn学习之劫持vdso_seaaseesa的博客-CSDN博客

_（https://blog.csdn.net/seaaseesa/article/details/104694219?ops_request_misc=%7B%22request%5Fid%22%3A%22163469082516780357227209%22%2C%22scm%22%3A%2220140713.130102334.pc%5Fblog.%22%7D&request_id=163469082516780357227209&biz_id=0&utm_medium=distribute.pc_search_result.none-task-blog-2~blog~first_rank_v2~rank_v29-7-104694219.pc_v2_rank_blog_default&utm_term=kernel&spm=1018.2226.3001.4450）_

还有bsauce师傅的简书：_https://www.jianshu.com/p/07994f8b2bb0_

### 

### 2、劫持cred结构体

#### 

#### (1)前置知识

这个没啥好讲的，就是覆盖uig和gid为零蛋就行，唯一需要的就是寻找cred的地址。

#### 

#### (2)寻找地址

##### 

##### ①利用prctl函数

task_srtuct结构体是每个进程都会创建的一个结构体，保存当前进程的很多内容，其中就包括当前进程的cred结构体指针。

```
//version 4.4.72
```

给出的题目编译在4.4.72内核下。

也就是可以将comm\[TASK_COMM_LEN\]设置为指定的字符串，相当于打个标记，不过不同版本中可能有所不同，不如最新版本V5.14.13的Linux内核就有所不同，其中还加了一个key结构体指针：

```
//version 5.14.13
```

然后就可以利用prctl函数的PR_SET_NAME功能来设置task_struct结构体中的comm\[TASK_COMM_LEN\]成员。

```
char target[16];
```

##### 

##### ②内存搜索定位

通过内存搜索，比对我们输入的标记字符串，可以定位comm\[TASK_COMM_LEN\]成员地址，比如设置标记字符串为"tryToFindPIG007"：

!\[\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

!\[\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

!\[\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

可以查看当前Cred结构中的内容：

!\[\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

```
//search target chr
```

##### 

##### ③任意写劫持cred

这个就不多说了，获取到cred地址之后直接写就行了。

#### 

#### POC

```
// gcc -static exp.c -o exp
```

#### 

#### ▲注意事项

不过这个如果不加判断 if ((cred||0xff00000000000000) && (real_cred == cred))搜索出来的cred可能就不是当前进程，具有一定概率性，具体原因不知，可能是搜索到了用户进程下的字符串？

!\[\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

### 3、劫持prctl函数

#### 

#### (1)函数调用链

prctl->security_task_prctl->\*prctl_hook

orderly_poweroff->\_\_orderly_poweroff->run_cmd(poweroff_cmd)-> call_usermodehelper(argv\[0\], argv, envp, UMH_WAIT_EXEC)

#### 

#### (2)前置知识

!\[\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

原本capability_hooks+440存放的是cap_task_prctl的地址。

!\[\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

但是经过我们的劫持之后存放的是orderly_poweroff的地址。

!\[\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

之前讲的prctl_hook指的就是capability_hooks+440。

这样劫持之后我们就能调用到orderly_poweroff函数了。

而orderly_poweroff函数中会调用实际\_\_orderly_poweroff函数，有如下代码：

```
//version 4.4.72
```

这里就调用到run_cmd(poweroff_cmd)，而run_cmd函数有如下代码：

```
//version 4.4.72
```

这里就调用到call_usermodehelper(argv\[0\], argv, envp, UMH_WAIT_EXEC)，这里的参数rdi中就是poweroff_cmd。所以如果我们可以劫持poweroff_cmd为我们的程序名字字符串，那么就可以调用call_usermodehelpe函数来启动我们的程序。而poweroff_cmd是一个全局变量，可以直接获取地址进行修改。

!\[\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

而call_usermodehelpe函数启动程序时是以root权限启动的，所以如果我们的程序运行/bin/sh且以root权限启动，那么就完成了提权。

#### 

#### (3)获取地址

##### 

##### ①prctl_hook

可以通过编写一个小程序，然后给security_task_prctl函数下断点，运行到call QWORD PTR\[rbx+0x18\]即可看到对应的rbx+0x18上存放的地址，将其修改为orderly_poweroff函数即可。

##### 

##### ②poweroff_cmd、orderly_poweroff

可以直接使用nm命令来获取，或者直接进入gdb打印即可。

!\[\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

此外orderly_poweroff也是一样的获取。如果无法查到，那么可以启动qemu，先设置为root权限后

cat /proc/kallsyms | grep "orderly_poweroff"即可，或者编译一个对应版本的内核进行查询。

!\[\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

▲最后fork一个子进程来触发反弹shell即可

#### 

#### POC

```
// gcc -static exp.c -o exp
```

#### 

#### 反弹Shell

```
#include <stdio.h>
```

或者直接system("chmod 777 /flag");也是获取flag的一种方式。

▲vdso的劫持一直没有复现成功过，明明已经劫持gettimeofday函数的内容为shellcode，然后也挂载了循环调用gettimeofday的程序，但是就是运行不了shellcode。

**参考：**

_1、Kernel Pwn 学习之路 - 番外 - 安全客，安全资讯平台 (anquanke.com)_

_（https://www.anquanke.com/post/id/204319#h3-12）_

_2、https://www.jianshu.com/p/07994f8b2bb0_

_3、https://blog.csdn.net/seaaseesa/article/details/104695399_

_‍_

!\[\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**看雪ID：PIG-007**

https://bbs.pediy.com/user-home-904686.htm

\*本文由看雪论坛 PIG-007 原创，转载请注明来自看雪社区

[!\[\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458412044&idx=2&sn=a6f266f86c6a52f96db32d1bce0e2ee4&chksm=b18f548686f8dd90a31f9dbccfccc0496885befc56079dc667047083d32206e5eafdb4f78881&scene=21#wechat_redirect)

**#** **往期推荐**

1.[分析屏保区TOP1的一款MacOS软件](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458413768&idx=1&sn=1f45d1c9037e45e2be1a8d9e43be3186&chksm=b18f5a4286f8d3543243647499c15116e0005a5aa8a43e7c24e31b37a763eef599aa3d3f271f&scene=21#wechat_redirect)

2.[某视频app的学习记录](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458413744&idx=1&sn=134b79e811ade3cd9aabd9e2554c8796&chksm=b18f5a3a86f8d32c916883e5356c6fffecb09b504d6bc544530315e733142dde218530b5c511&scene=21#wechat_redirect)

3.[Chrom V8分析入门——Google CTF2018 justintime分析](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458413548&idx=2&sn=9a57c3d6a96df540b62e4d041495f23c&chksm=b18f596686f8d070aa544f7366f2fed5fb9cd8a4ccc24ec1a7eadb649389556f5080fa06fde2&scene=21#wechat_redirect)

4.[内核学习-异常处理](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458413193&idx=3&sn=d9e54f5f39476109634b7319cef91205&chksm=b18f580386f8d115cc0c1f30bec88e0c2705d7d5aee020738a5233620a6ab63db202decf5269&scene=21#wechat_redirect)

5.[Typora 授权解密与剖析](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458412608&idx=1&sn=4b280d208020ceae766dd0923154baf6&chksm=b18f56ca86f8dfdcb8dc58aed2268ae7973fb1c79b00e333776f2c83d719fb69baa05271f9f1&scene=21#wechat_redirect)

6.[内核漏洞学习-HEVD-StackOverflowGS](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458412464&idx=1&sn=5a039a6ed17cd45f21ce5a62193ecadb&chksm=b18f553a86f8dc2c5250cec782b359472e738f92b3a1444d56a2e1be19c4397574c0ae545dd3&scene=21#wechat_redirect)

!\[\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

!\[\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**球分享**

!\[\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**球点赞**

!\[\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**球在看**

!\[\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

点击“阅读原文”，了解更多！

阅读原文

阅读 3092

​
