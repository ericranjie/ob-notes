# 

pyikaaaa 看雪学苑

 _2021年12月31日 18:21_

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/kR3MmOkZ9r6NkLx6w1ib1VAtF8869zN1jjTergCrdjFgAd7icia6Amr36hMaUW6pCBl3Hd7ElkaVZzfyElhiaPPz4w/640?wx_fmt=jpeg&wxfrom=13&tp=wxpic)  

本文为看雪论坛优秀文章  
看雪论坛作者ID：pyikaaaa

  

异常产生后，首先是要记录异常信息（异常的类型、异常发生的位置等），然后要寻找异常的处理函数，称为异常的分发，最后找到异常处理函数并调用，称为异常处理。

围绕 异常处理，异常分发，异常处理 展开。

异常的分类：

（1）cpu产生的异常  
![图片](https://mmbiz.qpic.cn/mmbiz_png/kR3MmOkZ9r6TKZgyKZGqyibibBAp74icx2s2UwrZvnSIg5IRNdGc413IW56D2c8cjmibAd3MypKPy9UoRs2pJ7noTA/640?wx_fmt=png&wxfrom=13&tp=wxpic)

代码执行时，CPU检测到除数为零，cpu抛出异常。

（2）软件模拟产生的异常

在C++或者是C#等一些高级语言中，在程序需要的时候也可以主动抛出异常，这种高级语言抛出的异常就是模拟产生的异常，并不是真正的异常。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

---

 **异常记录**

---

  

一

  

**CPU异常记录**

###   

### 1、CPU异常的处理流程

  

① CPU指令检测到异常（例：除零）

② 查IDT表(如下表)，执行中断处理函数

③ CommonDispatchException

④ KiDispatchException

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

找中断表的0号处理函数。

截图百度百科：  
![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

IDA打开ntoskrnl.exe 搜索字符串 _IDT 进行定位。

IDT表：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

### 2、分析中断处理函数 _KiTrap00

  

执行流程：

  

① 保存现场(将寄存器堆栈位置等信息保存到 _Trap_Frame 中)

② 调用CommonDispatchException函数

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

4018c7函数：没有对异常进行处理，函数内部调用CommonDispatchException函数。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

CommonDispatchException函数内部调用_KiDispatchExceptio，CPU这么设计的目的是为了让程序员有机会对异常进行处理。

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

总结：

  

（1）异常处理函数中并没有直接对异常进行处理，而是调用了CommonDispatchException。

（2）这样设计异常的目的是为了程序员有机会对异常进行处理。

###   

### 3、CommonDispatchException函数分析

  

该函数构造一个_EXCEPTION_RECORD结构体 并赋值。

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

CommonDispatchException主要就做了一件事情，就是将异常相关的信息存储到一个_EXCEPTION_RECORD结构体里，这个结构体的作用就是用来记录异常信息。

```
type struct _EXCEPTION_RECORD       
```

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

ebx中存储的是发生异常时的EIP，eax：0C0000094h ，这个是CPU自己在内部定义的异常代码。

ExceptionFlags异常状态：通过异常状态可以区分是CPU产生的异常还是软件模拟产生的异常。所有的CPU产生的异常这个位置的值是0，所有软件模拟产生的异常这个位置存储的值是1，如果堆栈错误里面存储的值是8，如果出现嵌套异常，里面存储的值是0x10。

ExceptionRecord下一个异常：通常是空的，但是出现嵌套异常，通过指针指向下一个异常。

  

### 4、总结

  

CPU异常的执行流程：

（1）CPU指令检测到异常

（2）查IDT表，执行中断处理函数

（3）调用CommonDispatchException（构建EXCEPTION_RECORD结构体）

（4）KiDispatchException（分发异常：目的是为了找到异常处理函数）

##   

  

二

  

**模拟异常记录**

  

示例代码：通过代码抛出异常 throw关键字

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)当通过软件抛出异常的时候，实际上就是调用了CxxThrowException。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

CxxThrowException调用（KERNEL32.DLL）RaiseException

###   

### **RaiseException分析**

  

在堆栈里构建一个EXCEPTION_RECORD结构体，然后对结构体进行赋值。

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

CPU产生的异常和软件模拟产生的异常都填充EXCEPTION_RECORD结构体，但是两者有差异。

差异：

**ExceptionCode 异常代码**

当CPU产生异常时会记录一个ErrorCode，通过查表可以查到ErrorCode具体的含义，不同的异常对应不同的错误代码，但是软件抛出的ErrorCode是根据编译环境决定的，如下图的EDX中存储的值即为当前编译环境的ErrorCode。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**ExceptionAddress 异常发生地址**

第二个区别就是ExceptionAddress，CPU异常记录的位置是真正的异常发生时的地址。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

CPU记录异常的地址是真正发生异常的地址,软件模拟产生的异常里面存储的是RaiseException这个函数的地址。

  

### **RaiseException函数分析**

  

RaiseException调用RtlRaiseException （ntdll.dll）

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

RtlRaiseException调用_ZwRaiseException

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

 _ZwRaiseException调用NtRaiseException

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

NtRaiseException调用_KiRaiseException  

  

这个函数主要做了两件事：

  

（1）把EXCEPTION_RECORD结构体的ExceptionCode最高位清零，用于区分CPU异常。

（2）调用KiDispatchException开始分发异常。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

RaiseException在初始化一个EXCEPTION_RECORD结构体之后，开始调用NTDLL中的RtlRaiseException; 之后，开始调用内核中NtRaiseException， NtRaiseException再调用另外一个内核函数KiRaiseException。接下来KiRaiseException会调用KiDispatchException开始异常的分发。

_https://www.cnblogs.com/Winston/archive/2010/03/16/1687649.html_

_http://runxinzhi.com/Ox9A82-p-5374527.html_

cpu异常和模拟异常只有在异常记录不同，调用KiDispatchException开始异常的分发后步骤都相同。

---

**异常处理**

---

  

三

  

**内核层用户处理流程**

  

异常可以发生在用户空间，也可以发生在内核空间。无论是CPU异常还是模拟异常，是用户层异常还是内核层异常，都要通过KiDispatchException函数进行分发。

```
VOID KiDispatchException(ExceptionRecord, ExceptionFrame,TrapFrame,PreviousMode, FirstChance)
```

###   

### （1）KiDispatchException函数分析

  

KiDispatchException函数执行流程总结：

① 将Trap_Frame备份到context为返回三环做准备；

② 判断先前模式 0是内核调用 1是用户层调用；

③ 判断是否是第一次调用；

④ 判断是否有内核调试器；

⑤ 如果没有内核调试器则不处理；

⑥ 调用RtlDispatchException处理异常；

⑦ 如果RtlDispatchException返回FALSE，再次判断是否有内核调试器，没有直接蓝屏。

  

(ntoskrnl.dll)

将Trap_frame备份到context为返回3环做准备(由于操作系统不知道分发的异常到底是3环的异常还是0环的异常，如果是3环的异常，进0环前的环境会被备份到Trap_Frame中，如果要在中途回到3环的话，就需要将 Trap_frame 备份到 context，为返回3环做准备)。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

如果没有内核调试器或者内核调试器不处理，都跳转到loc_4309A1。

调用_RtlDispatchException处理异常：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

如果RtlDispatchException返回FALSE，再次判断是否有内核调试器，没有直接蓝屏。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

###   

### （2）RtlDispatchException函数分析

  

RtlDispatchException函数作用：遍历异常链表，调用异常处理函数，如果异常被正确处理了，该函数返回1。如果当前异常处理函数不能处理该异常，那么调用下一个，以此类推。如果到最后也没有人处理这个异常，返回0。

RtlDispatchException调用_RtlpGetRegistrationHead

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

RtlpGetRegistrationHead将FS:0保存到eax之后返回。我们知道FS:0在零环的时候指向的是KPCR，而KPCR的第一个成员就是ExceptionList（具体信息可看以往文章apc和系统调用）。

ExceptionList，指向了一个结构体 _EXCEPTION_REGISTRATION_RECORD

_EXCEPTION_REGISTRATION_RECORD结构体必须位于当前线程的堆栈中。

```
typedef struct _EXCEPTION_REGISTRATION_RECORD
```

  

这个结构体有两个成员，第一个成员指向下一个_EXCEPTION_REGISTRATION_RECORD，如果没有下一个_EXCEPTION_REGISTRATION_RECORD结构体，那么这个地方的值是-1。第二个成员是异常处理函数。

当调用RtlDispatchException时，按顺序执行异常处理函数，若其中一个异常处理函数返回结果为真，就不再继续向下执行。

若执行完所有异常处理函数后，异常仍然没有被处理，那么就返回FALSE。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

四

  

**用户层异常处理流程**

###   

（1）上述内核层异常处理中，异常处理函数也在0环，不用切换堆栈，用户层异常发生在三环，异常处理函数也在3环，所以要切换堆栈（因为KiDispatchException在内核，从0环返到三环）回到3环执行异常处理函数。

  

（2）切换堆栈的处理方式与用户APC的执行过程几乎是一样的，惟一的区别就是执行用户APC时返回3环后执行的函数是KiUserApcDispatcher，而异常处理时返回3环后执行的函数是KiUserExceptionDispatcher。

  

（3）理解用户APC的执行过程是理解3环异常处理的关键。

  

### **分析用户层异常发生时的 KiDispatchException**

  

Trap_Frame备份到context。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

判断previousmode内核调用 还是 用户层调用，如果是用户层调用，jnz到loc_42COC2。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

当不存在内核调试器或者内核调试器没有处理时，跳转到42C10A。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

调用DbgkForwardException函数将异常发送给3环调试器。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

3环调试器如果不存在或者没有处理的话，就会开始修改寄存器，准备返回3环。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

这里eax的值是一个全局变量KeUserExceptionDispatcher；在操作系统初始化的时候，会给这个全局变量赋一个值，这个值就是ntdll.KiUserExceptionDispatcher函数。

总结：

##   

（1）_KeContextFromKframes将Trap_frame被分到context为返回3环做准备

（2）判断席安全模式，0是内核调用，1是用户层调用

（3）是否都是第一次机会

（4）否有内核调试器

（5）发送给3环调试

（6）如果3环调试器没有处理这个异常，修正EIP为KiUserExceptionDispatcher

（7）KiUserExceptionDispatcher函数执行结束：CPU异常与模拟异常返回地点不同

（8）无论通过那种方式，但线程再次回到3环时，将执行KiUserExceptionDispatcher函数

##   

  

五

  

**VEH（向量化异常处理）**

  

当用户异常产生后，内核函数KiDispatchException并不是像处理内核异常那样在0环直接处理，而是修正3环EIP为KiUserExceptionDispatcher函数后就结束了。

这样，当线程再次回到3环时，将会从KiUserExceptionDispatcher函数开始执行。

###   

### **KiUserExceptionDispatcher函数分析**

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

调用_RtlDispatchException找到当前异常的处理函数，处理成功，再次调用ZwContinue重新进入0环（回到3环时，修改了eip，将返回地址更改为当前函数，因此进入0环需要再次修正eip，ZwContinue再次进入零环目的就是把修正后的context写进trap_frame里，再次返回3环后，就会从修正后的位置再次执行）。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

没有找到当前的处理函数，跳转，调用_ZwRaiseException对异常进行第二次分发。

3环调用了RtlCallVectoredExceptionHandlers，0环没有调用此函数。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

###   

### _RtlDispatchException函数分析

  

作用：

（1）查找VEH链表(全局链表)，如果有则调用，如果没有继续查找局部链表

（2）查找SEH链表(局部链表，在堆栈中)，如果有则调用

  

注意：与内核调用时的区别！

VEH：全局链表，无论哪一个线程出现问题，都会先找这个全局链表，VEH中没有异常处理函数，在查找SEH链表。

###   

### 代码：自定义VEH

```
typedef PVOID(NTAPI *FnAddVectoredExceptionHandler)(ULONG, _EXCEPTION_POINTERS*);
```

  

这个异常处理函数的参数是一个结构体指针，有两个成员。

```
typedef struct _EXCEPTION_POINTERS
```

  

有了这个参数我们就可以捕获异常发生时的相关信息并且修改异常发生时的寄存器环境。

代码中是先判断异常代码是否为除0异常，然后修改发生异常的Eip和Ecx，接着返回异常已处理。如果不是除0异常就返回异常未处理。

  

### VEH异常的处理流程

  

① CPU捕获异常

② 通过KiDispatchException进行分发(3环异常将EIP修改为 KiUserExceptionDispatcher)

③ KiUserExceptionDispatcher调用RtlDispatchException

④ RtlDispatchException查找VEH处理函数链表，并调用相关处理函数

⑤ 代码返回到ZwContinue再次进入0环

⑥ 线程再次返回3环后，从修正的位置开始执行

  

##   

六

  

**SEH（结构化异常处理）**

  

当用户异常产生后，内核函数KiDispatchException并不是像处理内核异常那样在0环直接进行处理，而是修正3环EIP为KiUserExceptionDispatcher函数后就结束了。

这样，当线程再次回到3环时，将会从KiUserExceptionDispatcher函数开始执行。

KiUserExceptionDispatcher会调用RtlDispatchException函数来查找并调用异常处理函数，查找顺序：

① 先查全局链表：VEH

② 再查局部链表：SEH

  

SEH是线程相关的，存储在当前线程的堆栈中。

三环，FS:0指向的是TEB，TEB的第一个成员是一个子结构体_NT_TIB，这个子结构体的第一个成员就是ExceptionList，异常处理链表。

```
typedef struct _EXCEPTION_REGISTRATION_RECORD
```

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

VEH异常处理，全局的链表，想要插入异常处理函数，调用AddVectoredExceptionHandler函数；

SEH异常处理，想要插入异常处理函数的线程，需要自行插入_EXCEPTION_REGISTRATION_RECORD 这中结构。

###   

### **RtlDispatchException函数分析**

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

handler必须遵循调用约定，有自己的格式。

```
/*SEH处理函数是提供给RtlDispatchException调用的，所以需要遵循一定的格式。
```

  

自定义SEH

```
#include <stdio.h>
```

###   

### SEH异常的处理流程

① FS:[0]指向SEH链表的第一个成员；

② SEH的异常处理函数必须在当前线程的堆栈中；

③ 只有当VEH中的异常处理函数不存在或者不处理才会到SEH链表中查找。

  

##   

七

  

**编译器扩展的SEH**

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

SEH的处理流程实际上就是在当前的堆栈中挂一个链表。

使用SEH这种方式来处理异常要创建一个结构体，挂到当前TEB的ExceptionList链表里，编写异常处理函数。

编译器支持的SEH

```
_try                                   // 挂入链表
```

  

except里的过滤表达式用于异常过滤，只能有以下三个值：

  

EXCEPTION_EXECUTE_HANDLER(1) 异常已经被识别，控制流将进入到 _except模块中运行异常处理代码。

  

EXCEPTION_CONTINUE_SEARCH(0) 异常不被识别，也即当前的这个 _except模块不是这个异常错误所对应的正确的异常处理模块。系统将继续到上 _try except域中继续查找一个恰当的 _except模块。

  

EXCEPTION_CONTINUE_EXECUTION(-1) 异常被忽略，控制流将在异常出现的点之后，继续恢复运行。

  

过滤表达式只能有三种写法：

① 直接写常量值

② 表达式

③ 调用函数

  

手动挂入链表：

```
_asm
```

  

  

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E) 

  

**看雪ID：pyikaaaa**

https://bbs.pediy.com/user-home-921642.htm

*本文由看雪论坛 pyikaaaa 原创，转载请注明来自看雪社区

  

  

[![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458409596&idx=3&sn=658012a9debc76c73efed03342d41358&chksm=b18f6af686f8e3e09180522b243def309e265532cfadfdbf5becd7d9ec90f4505136959ebdfe&scene=21#wechat_redirect)  

  

**#** **往期推荐**

1.[Typora 授权解密与剖析](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458412608&idx=1&sn=4b280d208020ceae766dd0923154baf6&chksm=b18f56ca86f8dfdcb8dc58aed2268ae7973fb1c79b00e333776f2c83d719fb69baa05271f9f1&scene=21#wechat_redirect)  

2.[内核漏洞学习-HEVD-StackOverflowGS](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458412464&idx=1&sn=5a039a6ed17cd45f21ce5a62193ecadb&chksm=b18f553a86f8dc2c5250cec782b359472e738f92b3a1444d56a2e1be19c4397574c0ae545dd3&scene=21#wechat_redirect)

3.[frida内存检索svc指令查找sendto和recvfrom进行hook抓包](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458412044&idx=1&sn=ef112858ca0f0ca0e103ceac6372253c&chksm=b18f548686f8dd907b435772d501200db5cf95d2bf62665a9edc09b3f20c0c62067aa108b3ee&scene=21#wechat_redirect)

4.[对ollvm的算法进行逆向分析和还原](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458412043&idx=1&sn=2dc05ccc842f0879728b2fcd89c20c25&chksm=b18f548186f8dd976bbefdbffc0b42282448a71d5f95724e2cd7b30dcf09299bdd7c679e2ad6&scene=21#wechat_redirect)

5.[从0开始实现一个简易的主动调用框架](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458412042&idx=1&sn=f15dd34551a91eb3d7732fbdb1153cd0&chksm=b18f548086f8dd96535c2b41fce61602aa03f6409be2ef684a0e9fe608926cdab85e109764db&scene=21#wechat_redirect)

6.[Kernel从0开始](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458411220&idx=1&sn=bcb5872be8a7d92dee039fc565075057&chksm=b18f505e86f8d948bdad54fa350494b56fe67f2dbb8b0a3c8261b5b620f23609fc987c7fd8cb&scene=21#wechat_redirect)

  

  

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

阅读 780

​

写留言

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/1UG7KPNHN8EGLfh77kFmnicd9WOic2ibvhCibFdB4bL4srJCgo2wnvdoXLxpIvAkfCmmcptXZB0qKWMoIP8iaibYN2FA/300?wx_fmt=png&wxfrom=18)

看雪学苑

222

写留言

写留言

**留言**

暂无留言