
原创 贺东升 Linux内核之旅

 _2022年03月12日 06:49_

本文整理自”Linux内核之旅开源社区“直播教学视频《BPF C编程入门》，主讲人为Linux内核之旅社区研三贺东升，喜欢看视频的小伙伴可以通过文章末尾的视频链接观看。  

主要内容包括：

- BPF C编程环境搭建。
    
- 介绍使用BPF Ｃ编写运行一个 hello world 程序。
    
- 通过分析源码，解释BPF CALL指令调用的内核辅助函数转换为BPF字节码的过程。
    
- 使用 bpftool 和 objdump查看BPF程序的字节码，观察JIT即时编译前后BPF字节码的变化。
    

  

bcc是BPF高度封装的框架，我们使用的时候可以直接运行脚本，这对于使用上来说是非常友好的，但对于我们理解背后的原理来说是不清晰的，因为bcc的高度封装性，把背后的一些编译的过程都给屏蔽掉了，不利于我们去理解整个BPF机制的工作原理。使用c编程的完整的过程，更贴近于eBPF的机制流程。这是我们今天选择BPF C作为编程入门介绍的原因。

简单回顾一下上次介绍的eBPF机制的框架图：

![图片](https://mmbiz.qpic.cn/mmbiz_png/SeWfibBcBT0E37XW86gsNeX7E22NH3yoeVfqZ7bMzkJGLD5jjULlHoIWbVVAtPT8qSzsFcVCr3wnlYQjoP77AfQ/640?wx_fmt=png&wxfrom=13&tp=wxpic)

如图所示，左边是BPF工具，使用BPF工具需要我们自己来编写BPF程序，BPF程序经过Clang和LLVM会编译生成一段BPF字节码，通过用户空间的加载器将BPF字节码，通过系统调用的方式加载进内核。

  

进入到内核的流程：首先验证器verifier进行一些循环的检查，还有一些安全的验证，验证通过以后就可以交由BPF虚拟机执行。eBPF机制有许多的hook点，而这些hook点是事件触发类型。在函数上挂好这些钩子以后，当函数被调用的时候，就会触发这些点，进而执行我们的BPF程序。

  

如果我们想把数据存在一个地方的话，可以通过BPF的map。在BPF程序中创建map，把我们想获取的内核数据保存在map中。在用户态调用一些接口来读或写map，把保存的数据取出来。

  

**eBPF可以帮助我们做什么？**

  

简单来说我们可以通过在用户空间写BPF程序，拿取内核中的一些数据。例如内核中某个内核函数的调用次数或者某个内核函数的执行时间。在用户态编写好BPF程序以后把它加载到内核，就可以拿到我们想要的内核数据。

  

**BPF C编程环境搭建**　

基本环境：

- 系统：Ubuntu 18.04 LTS
    

环境搭建步骤：

**1.下载内核源码**

下载的内核源码与Ubuntu 18.04的内核版本一致。首先查看当前内核版本：uname -r。

```
root@ubuntu:~# uname -r5.4.0-65-generic
```

然后在内核源码镜像站点(http://ftp.sjtu.edu.cn/sites/ftp.kernel.org/pub/linux/kernel/)

下载对应版本的内核源码包，解压在/usr/src目录下。

```
root@ubuntu:/usr/src# lslinux-5.4
```

  

**2.安装依赖项**

```
root@ubuntu:/usr/src/linux-5.4# apt install libncurses5-dev flex bison libelf-dev binutils-dev libssl-dev
```

  

**3.安装Clang和LLVM**

```
root@ubuntu:/usr/src/linux-5.4# apt install clang llvm
```

  

**4.配置源码**

在源码根目录下使用make defconfig生成 .config文件。

```
root@ubuntu:/usr/src/linux-5.4# make defconfig
```

  

**5.解决modpost：not found错误**

因为直接make M=samples/bpf时，会报缺少modules的错误。修复modpost的报错，以下两种解决方案二选一：

- 方案一(修复模块)
    
    ```
    root@ubuntu:/usr/src/linux-5.4# make modules_prepare
    ```
    
      
    
- 方案二(补全脚本)
    
    ```
    root@ubuntu:/usr/src/linux-5.4# make scripts
    ```
    

  

**6.关联内核头文件**

```
root@ubuntu:/usr/src/linux-5.4# make headers_install
```

  

**7.编译内核BP****F程序样例**

在源代码目录下执行make M=samples/bpf。

```
root@ubuntu:/usr/src/linux-5.4# make M=samples/bpf
```

此时进入/linux-5.4/samples/bpf/中，会看到生成了BPF字节码文件*_kern.o和用户态的可执行文件。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

下一步我们就可以编写我们自己的BPF程序了。整个的流程如下图：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**使用BPF Ｃ编写运行一个hello world程序**

**1.编写hello_kern.c**

```
root@ubuntu:/usr/src/linux-5.4/samples/bpf# vim hello_kern.c
```

```
#include <uapi/linux/bpf.h>#include"bpf_helpers.h"SEC("kprobe/sys_write")int bpf_prog(void *ctx){ char msg[]="hello world!\n";    bpf_trace_printk(msg,sizeof(msg)); return 0;}char _license[] SEC("license")="GPL";//注：编写BPF C程序要进入linxu-5.4/samples/bpf目录进行编辑
```

  

**2.编写hello_user.c**

```
root@ubuntu:/usr/src/linux-5.4/samples/bpf# vim hello_user.c
```

```
#include "BPF_load.h"int main(void){      if (load_BPF_file("hello_kern.o"))                 return -1;      read_trace_pipe();          return 0;}
```

  

**3.****修改Makefile，添加新增需要编译的文件**

```
root@ubuntu:/usr/src/linux-5.4/samples/bpf# vim Makefile
```

```
# List of programs to buildhostprogs -y := hello    # LibBPF dependencieshello-objs := BPF_load.o hello_user.o    #Tell kbuild to always build the programsalways += hello_kern.o//tips:所有添加项直接添加在原有内容的末尾即可。
```

  

**4.返回源码根目录make**

```
root@ubuntu:/usr/src/linux-5.4/samples/bpf# cd ../..
```

```
root@ubuntu:/usr/src/linux-5.4/samples/bpf# make M=samples/BPF
```

  

**5.运行用户态加载器**  

```
root@ubuntu:/usr/src/linux-5.4/samples/bpf# ./hello
```

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**BPF CALL指令调用的内核辅助函数转换为BPF字节码的过程**

**1.BPF** **程序的节**

看到在上一次的内容中我们提到了一个概念叫做BPF程序的节。SEC宏会把名字为括号中的字符串编译到elf的目标文件中。如图：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

节本身是elf文件中的概念。SEC宏中第一个为程序类型，第二个是我们要跟踪的函数。对于SEC宏来说的话他会把整个kprobe/sys_write当成节的名字，编译到elf的目标文件中。

通过readelf工具查看目标文件中得节头信息：

```
root@ubuntu:/usr/src/linux-5.4/samples/bpf# readelf -S hello_kern.o
```

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)  

可以看出，SEC宏的作用是把名字为括号中的字符串编译到elf的目标文件中。

  

**2.BPF 程序的字节码**

使用objdump查看BPF程序的字节码：  

```
root@ubuntu:/usr/src/linux-5.4/samples/bpf# objdump -s hello_kern.o
```

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

可以看到kern.o它是目标文件，格式为elf64，采用小端来存储。这里面就显示了我们节的一些信息。比如说我们用SEC定义的节名字叫做kprobe/sys_bpf，还有个licence许可证的节，说明我们用SEC宏把我们后面定义的一些名字编译到elf目标文件中的某个节中。现在我们应该理解了SEC宏的作用。

在宏的下面是BPF程序，它保存在名字为kprobe/sys_write的节中的。

右边图中橙色框圈起来的就是BPF字节码，是左面灰色部分圈起来的代码经过clang和llvm来生成的BPF的字节码。这段字节码可以被虚拟机解码执行。

接下来我们介绍一下**BPF****内核辅助函数调用是怎么转化为BPF****字节码的。**

首先我们来使用llvm-objdump工具来反汇编BPF字节码来看一下他的BPF汇编表示。

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

可以看到它的里面有文件格式是ELF64-BPF，第二行是我们用SEC宏定义的节的名字kprobe/sys_bpf，bpf_prog是我们定义的BPF程序的名字。

重点关注第10行，为汇编表示在汇编里面调用函数函数它是地址，但是我们可以看到6它并不是实际的地址，而是整形数，左面对应是汇编的BPF字节码。

下面我们就来进一步分析BPF内核辅助函数调用是怎么被编译成BPF字节码。

  

**从头文件入手**

```
/tools/testing/selftests/bpf/bpf_helpers.h(line 25)
```

```
/include/uapi/linux/bpf.h(line 2754)
```

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

我们首先来从两个头文件入手，也是我们最开始写BPF程序的时候两个必须包含的头文件。其中在bpf_helpers.h里面可以看到图示一段定义，函数指针的变量明正是我们在BPF程序里面调用的内核辅助函数的名字。在文件中是以函数指针的形式来定义的，我们可以看函数指针它是用以强制类型转换赋给了指针变量，我们在BPF程序中实际调用的并不是真正的内核辅助的函数接口，而是函数指针。

BPF_FUNC_map_lookup_elem()函数是在文件uapi/bpf.h文件中定义的，它不是直接定义，而是通过两个宏来定义的。

  

**宏展开过程**

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

bpf_call 123是伪汇编的表示，并不是真正的BPF指令，它对应的真正的BPF指令为BPF_EMIT_CALL(FUNC)。  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

可以看到BPF_EMIT_CALL BPF指令宏调用了内核辅助函BPF_FUNC_map_lookup_elem()。

我们就可以理解 FUNC　其实就是BPF辅助函数的id号。

我们前面那两个宏展开以后得到的枚举类型。说func 谁对应的枚举类型里面的某个整型值，替换一下后，最下面的一句就更容易我们理解。

我们用BPF指令宏的时候

接下来 因为我们在bpf程序中调用的是trace_printk函数，我们还是以trace_printk来进行举例。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

继续将指令宏展开，会得到bpf_insn结构体，即为BPF指令集的格式。

其中，BPF_CALL为真正的BPF调用指令，dst_reg为目的寄存器，sec_reg为源寄存器，off为偏移量，imm为立即数。  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

进行替换后得到如图示结构体，code表示操作码，在内核中分别定义为  

```
#define BPF_JMP 0x05#define BPF_CALL 0x80
```

按位与后得到code的值为85，目的寄存器、源寄存器和off均被初始化为0，imm为枚举结构中的整型值，即为6。  

  

进行组合

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

bpf_insn表示的BPF指令级的格式，我们把里面的成员按照字节序的方式来组装一下就可以得到我们对应BPF字节码。 我们进行组装的时候一定要按照BPF指令集定义的形式来组装。

  

查看JIT前后BPF字节码的变化  

观察JIT前后的变化：objdump&&bpftool  

1. BPF程序运行之前
    
    ```
    objdump -s hello_kern.o          //显示不直观llvm-objdump -d hello_kern.o     //显示直观
    ```
    
2. BPF程序运行之前
    
    ```
    ./bpftool prog show                          //显示加载的BPF程序./bpftool prog dump xlated id xx             //xlated模式将BPF指令翻译为汇编指令打印出来./bpftool prog dump xlated id xx opcodes     //使用opcodes修饰符可在输出中包含BPF指令的opcode
    ```
    

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

我们写的BPF程序首先通过clang它是llvm的前端，它的作用来最后会生成ll IR文件，传给llvm的后端。llvm的后端可以支持BPF类型，会生成BPF类型的字节码。生成BPF字节码以后，通过用户态的加载器通过系统调用进入到内核。

  

现在我们就把整个编译过程来理解一下

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

首先来说一下clang和llvm的关系。

先说一下llvm是什么？我们说最后要用llvm来编译 但是并不是说l lv m 它是编译器它本身并不是编译器 它只是可以说它是提供了一些开发编译器还有些解释器 些语言工具的一些库。同时ll v m我们可以理解成样那种框架 框架里面它其实还包含那些工具链的组件比如说l lc 也我们llvm的后端，或者说一些其他的工具。

  

这幅图就展示了它的编译过程

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

clang的作用 用做词法分析、语法分析还有语义分析，最关键的一步它会生成

IR文件 ，IR是llvm的中间表示 它也是一种语言，类似于底层那种汇编语言。生成IR以后会把IR文件交给llvm的后端,在后端的时候生成对应平台的机器码。

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

## BPF学习资料

### 书籍

- 《Linux内核观测技术BPF》
    
- 《BPF之巅：洞悉Linux系统和应用性能》
    
- 《Systems Performance》
    
- 《BPF Performance Tools》
    

  

本文原直播视频链接及二维码

https://www.bilibili.com/video/BV1f54y1h74r?spm_id_from=333.999.0.0

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

阅读 3084

​