

二木先生 看雪学苑

 _2024年04月26日 17:59_ _上海_

  

一  

  

ARM汇编

  

一般的我们的电脑都是X86架构的机器，这里我们使用clang来对我们的文件进行编译，自己编译文件，放入ida中对照着学习更容易理解一点。  

  

  

二

  

关于clang的一些指令：

  

### 使用clang直接编译成可执行文件

  

//将我们的文件编译成ARMv5架构的文件（32位）  
clang -target armv-linux-android21 demo.c -o demo  
//将我们的文件编译成ARMv7-A架构的文件 (32位)  
clang -target armv7a-linux-android21 demo.c -o demo  
//将我们的文件编译成AArch64 架构的文件（64位）  
clang -target aarch64-linux-android21 1.cpp -o demo  

##   

## 分开步骤进行编译

  

### 预处理

clang -target arm-linux-android21 -E demo.c -o demo.i  

###   

### 编译（生成汇编文件）

clang -target arm-linux-android21eabi -S demo.i -o demo.s  

###   

### 汇编（生成未资源重定位的二进制文件）

clang -target arm-linux-android21eabi -S demo.s -o demo.o  

###   

### 链接（生成可执行文件）

clang -target arm-linux-android21eabi -S demo.o -o demo  

##   

## clang编译thumb架构的文件

clang -target arm-linux-android21 -S -mthumb demo.c -o demo.s  

###   

### 针对于ARM与thumb的区别

  

我们学习ARM架构的时候，我们常常会听说从arm状态转为thumb状态。这里记录一下龙去脉。

  

随着ARM架构的发展，在许多方面都有所用到，但是我们ARM的指令集都是32位，或者64位的，随着手机等设备对32位处理器需求的不断增加，功耗和成本都变得十分关键。如何减少程序占用空间大小的问题亟待解决，这个时候，arm公司就推出了ARM7TDMI处理器，在其上面就能支持了16位指令集，即thumb。thumb出现的目的是实现更高的代码密度，本质上是对我们的ARM指令的扩充。

  

ARM处理器具有从ARM状态到Thumb状态和从Thumb状态到ARM状态的无缝转换能力，可以根据需要动态切换指令集。

  

一些区别：

◆ARM 指令集提供了完整的 32 位指令，用于执行各种操作，包括算术运算、逻辑操作、数据传输等。而thumb没有很完整的指令集，有些情况下需要切换到arm下进行执行。

◆thumb现在来说一般是16, 32位的指令集，而ARM现在一般是32，64位的指令。

  

总得来说，thumb就是实现一个更高性能，更高代码密度的ARM指令分支。

  

  

三  

  

ARM寄存器

##   

## Arm-v7架构

  

ARM共有37个寄存器，都是32位长度的寄存器（具体可以看后面的工作模式图）。

  
37个寄存器中其中前31个（0~30）是通用寄存器（模式不同，使用的时候有所区别），最后2个(31,32)是专用寄存器（`sp` 寄存器和 `pc` 寄存器)。5个固定用作5种异常模式下的SPSR。

  

◆**CPSR**：当前程序**状态寄存器**，通常不直接操作，除非进行异常处理或任务切换。

◆**SPSR**：在异常处理中使用，**保存异常发生时的CPSR值**。

  

### 通用寄存器

  

ARM架构的通用寄存器是用于执行大多数指令的寄存器，可存储临时数据、地址和中间计算结果。

  

**R0-R12：通用目的寄存器**

  

◆**R0-R3** 通常用作指令操作数，传递函数参数和返回值。

◆**R4-R11** 通常用作局部变量存储，称为"callee-saved"寄存器，函数使用前需保存值，返回前需恢复。

◆**R12** 有时称为Intra-Procedure-call scratch register（IP），在某函数调用约定中用作临时存储。FP：栈帧基址寄存器，即R12

  

**R13 (SP)：堆栈指针寄存器**

  

◆**SP** 指向当前栈顶，管理函数调用参数传递、局部变量存储和返回地址。

  

**R14 (LR)：链接寄存器**

  

◆**LR** 存储子程序调用返回地址。执行BL（Branch and Link）指令时，将PC当前值保存到LR，函数执行完毕后返回调用点。

  

**R15 (PC)：程序计数器**

  

◆**PC** 存储下一条将执行指令的地址。执行分支指令时，PC更新为新地址。

  

### CPSR (Current Program Status Register)

  

CPSR寄存器包含当前程序状态信息，包括：

  

◆**条件标志位**：零标志(Z)，负标志(N)，进位标志(C)，溢出标志(V)，通常在算术或逻辑操作后自动设置，用于条件分支指令。

◆**控制位**：中断禁用位(I)和快速中断禁用位(F)，用于控制中断使能状态。

◆**模式位**：指示当前CPU操作模式，如用户模式、系统模式、中断模式等。

  

### SPSR (Saved Program Status Register)

  

异常发生（如中断）时，当前CPSR自动保存到SPSR，以便异常处理后恢复先前状态。每种异常模式有自己的SPSR。

  

## 关于ARM-V8架构

###   

### 通用寄存器

  

Arm-v8架构有31个通用寄存器X0-X30 (64位长），而Arm-v7架构仅有16个通用寄存器R0-R15（32位长）。

![[Pasted image 20240905151128.png]]

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

### 特殊用途寄存器（X8, X16~X18, X29, X30）:（了解）

  

◆X8：间接结果寄存器。用来保存间接结果的地址。

◆子程序内部调用寄存器，ip寄存器（**I**ntra-**P**rocedure-Call Temporary Registers），IP0与IP1，可被Veneers（实现Arm/Thumb状态切换）。或是用于在子程序调用前，作为存储中间值的临时寄存器。使用时不需要保存。

◆x18：平台寄存器（**P**latform **R**egister），用于保存当前所用的平台的ABI=

◆x29：帧指针寄存器（FP），用于连接栈帧，使用时必须保存。

◆x30：链接寄存器（LR），用于保存子程序的返回地址。

  

### 寄存器的访问方式

  

64寄存器总的来说有两种不同的使用，一种就是当原本的64位的寄存器来访问，另一种就是为了兼容32位，将64位寄存器拆成32位的寄存器来使用

通用寄存器的访问方式有2种：

◆当作 32位寄存器的时候，使用 `W0` ~ `W30` 来引用它们。（数据保存在寄存器的低32位）

◆当作 64位 寄存器的时候，使用 `X0` ~ `X30` 来引用它们。

  

对于一些专用寄存器的访问方式:

**栈帧指针寄存器**:32位，使用 `WSP` 来引用，64位，使用 `SP` 来引用。

**零寄存器**:32位，使用 `WZR` 来引用，64位，使用 `ZR` 来引用。

  

  

四  

  

ARM处理器几种工作模式

![[Pasted image 20240905151141.png]]

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

###   

### 用户模式 (User Mode):

正常程序执行的默认模式。

###   

### 快速中断模式 (FIQ Mode)

用于处理快速中断请求，有额外的寄存器以减少中断服务例程的执行时间。

###   

### 普通中断模式 (IRQ Mode)

用于处理普通的中断请求。

###   

### 管理模式 (Supervisor Mode)

用于操作系统的内核级别操作，通常在系统启动或系统调用时进入。

###   

### 未定义模式 (Undefined Mode)

当执行了未定义的指令时进入此模式。在异常或中断发生时，处理器会自动切换到相应的模式，并使用该模式下的寄存器集。

###   

### 中止模式 (Abort Mode)

当发生数据或指令预取中止时进入此模式。

###   

### 系统模式 (System Mode)

用于运行操作系统的内核代码，与用户模式共享相同的寄存器。

#  五  ARM架构下的函数调用约定

  

ARM 架构下常见的函数调用约定有两种：

**AAPCS**（ARM Architecture Procedure Call Standard）

**AAPCS-VFP**（ARM Architecture Procedure Call Standard with the Vector Floating-Point extension）

##   

## AAPCS：

  

AAPCS 是 ARM 架构的默认函数调用约定，适用于大多数 ARM 架构的编译器和操作系统。它定义了函数参数的传递方式、栈的使用规则以及寄存器的分配方式。

◆参数传递：函数的**前几个参数（通常是 R0、R1、R2、R3）通过寄存器传递**，而剩余的参数通过栈传递。参数的传递顺序是**从左到右**。

◆栈的使用：栈用于保存局部变量、临时变量和函数调用过程中的状态。**栈的增长方向是从高地址向低地址。**

◆寄存器的分配：除了用于参数传递的寄存器，还有一些寄存器被用作临时寄存器或保存特定的值，如栈指针（SP）、链接寄存器（LR）和程序计数器（PC）等。

◆堆栈的管理由**被调用的函数负责**。具体来说，堆栈的管理包括参数的压栈、局部变量的分配和释放、以及函数调用过程中的保存和恢复寄存器等。

  

## AAPCS-VFP：

  

AAPCS-VFP 是在 AAPCS 的基础上添加了向量浮点扩展（Vector Floating-Point extension）的函数调用约定。它适用于需要使用浮点运算的函数。

◆浮点参数传递：浮点参数通过浮点寄存器传递，通常是 S0、S1、S2、S3 等。如果参数个数超过了浮点寄存器的数量，剩余的参数通过栈传递。

◆浮点寄存器的使用：除了用于浮点参数传递的寄存器，还有一些寄存器被用作临时寄存器或保存特定的值，如浮点链接寄存器（LR）和浮点状态寄存器（FPSCR）等。

  

  

六  

  

汇编指令

##   

## ARM中的寻址方式：

###   

### 寄存器寻址

  

将 R2 中的值移动到 R1 中。

mov r1, r2  

###   

### 立即寻址

  

将 0xFF00 立即数移动到 R0 中。

mov r0, #0xFF00  

###   

### 寄存器移位寻址

  

将 R1 左移 3 位的结果移动到 R0 中。

mov r0, r1, lsl #3  

###   

### 寄存器间接寻址

  

将 R2 指向的**地址中的值**加载到 R1 中。

ldr r1, [r2]     //r1=*r2  

###   

### 基址变址寻址

  

将 R2 加上 4 所得的**地址中的**值加载到 R1 中。

ldr r1, [r2, #4]  ; r1=*(r2+4)  

###   

### 多寄存器寻址

  

将 R1 中的值加载到 R2-R7 和 R12 中，然后将 R1 加上 32。

ldmia r1!, {r2-r7, r12}  

###   

### 堆栈寻址

  

将 R2-R7、LR 的值存储到堆栈中，然后将 SP 减去 24。

stmfd sp!, {r2-r7, lr}  

###   

### 相对寻址

  

如果条件码为 EQ，则跳转到标签 flag 处。

beq flag  
flag:  

##   

## 汇编常用的指令

##   

## 跳转指令

  

B 强制跳转

B label  //无条件跳转到标签 label 处  

  

BL 带返回的跳转，将返回地址放入LR寄存器中

BL label //跳转到标签 label 处，并将返回地址存储在 LR 中  

  

BLX 带返回和带状态的切换跳转指令 arm → thumb

BLX label  //跳转到标签 label 处，并将返回地址存储在 LR 中。可以实现 ARM 和 Thumb 之间的切换  

  

Bx 带状态的跳转

BX R1   // 跳转到地址存储在 R1 中的位置，并切换到相应的指令集模式  

##   

## 数据处理指令

  

mov 赋值

MOV R0, R1 //将 R1 中的值赋值到 R0 中。  

  

ADD 加

ADD R0, R1, R2 //将 R1 和 R2 中的值相加，然后将结果存储到 R0 中。  

  

SUB 减

SUB R0, R1, R2  //  

**AND**与

AND R0, R1, R2 //将 R1 和 R2 中的值进行按位与操作，然后将结果存储到 R0 中。  

**EOR**：异或

EOR R0, R1, R2 //将 R1 和 R2 中的值进行按位异或操作，然后将结果存储到 R0 中。  

**ORR**：或

ORR R0, R1, R2  //将 R1 和 R2 中的值进行按位或操作，然后将结果存储到 R0 中。  

**BIC**：与非 , 可以实现的 Bit Clear的功能

BIC R0, R1, #0xf  //将 R1 和立即数 #0xf 进行按位与非操作，然后将结果存储到 R0 中。  
  
//将R1   低4位清0  

##   

## 乘法指令：

  

MUL 一般乘法：

MUL  r0 r1,r2   //r0 = r1 * r2  乘法  

  

MLA 带加法的乘法:

MLA  r0,r1,r2,r3  //r0 = r1 * r2 + r3   带加法的乘法  

**由于32位寄存器 没有办法存64位的大数，因此需要两个寄存器来存储，64位的乘法操作就会发送改变**

SMULL 64位乘法:

SMULL  r0,r1 ,r2 ,r3  // r0 = (r2 * r3)的低32位   r1 = (r2 * r3)的高32位  

  

SMLAL 64位带加法的乘法

SMLAL  r0,r1 ,r2 ,r3  //r0 = (r2 * r3)的低32位 + r0    r1 = (r2 * r3)的高32位 + r1  

  

UMULL 64位无符号乘法

UMULL r0,r1 ,r2 ,r3  // r0 = (r2 * r3)的低32位   r1 = (r2 * r3)的高32位  

  

UMLAL 64位无符号带加法的乘法

UMLAL   r0,r1 ,r2 ,r3  //r0 = (r2 * r3)的低32位 + r0    r1 = (r2 * r3)的高32位 + r1  

##   

## 移位操作：

**LSL**：逻辑左移指令，将寄存器中的值向左移动指定的位数，移动时右侧空出的位补零。

LSL R0, R1, #2 // 将 R1 中的值左移 2 位，结果存储到 R0 中  

**LSR**：逻辑右移指令，将寄存器中的值向右移动指定的位数，移动时左侧空出的位补零。

LSR R0, R1, #3 // 将 R1 中的值右移 3 位，结果存储到 R0 中  

**ROR**：循环右移指令，将寄存器中的值向右移动指定的位数，移动时左侧空出的位补上右侧的位。

ROR R0, R1, #4 // 将 R1 中的值循环右移 4 位，结果存储到 R0 中  

**ASR**：算术右移指令，将寄存器中的值向右移动指定的位数，移动时左侧空出的位补上符号位。

ASR R0, R1, #5 // 将 R1 中的值算术右移 5 位，结果存储到 R0 中  

**RRX**：扩展的循环右移指令，将寄存器中的值向右移动一位，同时将 C 标志位的值作为最低位插入到左侧。

RRX R0, R1 // 将 R1 中的值扩展的循环右移 1 位，结果存储到 R0 中  

##   

## 内存访问指令：

### 为什么会分别有加载指令和存储指令？

  

ARM汇编采用RISC架构，CPU本身并不能直接的读取内存，而是需要先将内存中的内存加载进入CPU通用寄存器之中，才能被我们的CPU进行处理。

  

**ldr 读取**

ldr r0,=0x12  //将0x12赋值给r0  
ldr r0, .lable1 //获得.lable1的地址，存储在r0之中  
ldr r0,[r3] // r0 = *r3  
ldr r0,[r3,#4]  // r0 = *(r3 + 4)  
ldr r0,[r3,r2,LSL #2] // r0 = *(r3+(r2 << 2))  

**str 存储**

STR R0，[R1]，＃8      // 将R0中的字数据写入以R1为地址的存储器中，并将新地址R1＋8写入R1。  
STR R0，[R1，＃8]      // 将R0中的字数据写入以R1＋8为地址的存储器中。”  
STR  R1, [r0]         // 将r1寄存器的值，传送到地址值为r0的（存储器）内存中  

  

**ldr与 str 指令的后缀与变种**

在32位中会读4个字节 在64位中会读8个字节  
读取  
ldr    
ldrb  //读一个字节  一个字节的读取  
ldrh //读一个数 两个字节的读取  
ldm  //批量进行处理 （根据一些组合有着不同的方式，后面写）  
  
存储  
str  //四字节写入  
strb //一个字节写入  
strh //两个字节写入  
stm  // 批量处理 

##   

## 比较寄存器指令

  

`CMP`  比较两个操作数的值

CMP r1, #10   ; 比较寄存器 r1 的值和立即数 10  
BGT label     ; 如果 r1 的值大于 10，则跳转到 label 处  

  

`CMN`比较两个操作数的值的补码

CMN r2, #5    ; 比较寄存器 r2 的值的补码和立即数 5  
BLE label     ; 如果 r2 的补码值小于或等于 5，则跳转到 label 处  

  

`TST` 将两个操作数进行按位与运算，并根据结果设置条件码。

TST r3, #0xFF ; 检查寄存器 r3 的低八位是否全部为 1  
BEQ label     ; 如果 r3 的低八位全为 1，则跳转到 label 处  

`   `

`TEQ` 将两个操作数进行按位异或运算，并根据结果设置条件码。

TEQ r4, #0x80 ; 检查寄存器 r4 的值和立即数 0x80 的异或结果  
BNE label     ; 如果 r4 的值与 0x80 异或后不为零，则跳转到 label 处  

##   

## 比较值指令

##   

![[Pasted image 20240905151238.png]]

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

##   

## 堆栈的指令组合后缀

###   

### 四种栈的类型：

**空栈**：栈指针指向空位，每次存入时可以直接存入然后栈指针移动一格；而取出时需要先移动一格才能取出。

**满栈**：栈指针指向栈中最后一格数据，每次存入时需要先移动栈指针一格再存入；取出时可以直接取出，然后再移动栈指针。

**增栈**:栈指针移动时向地址增加的方向移动的栈。

**减栈**:栈指针移动时向地址减小的方向移动的栈。

###   

### 针对此衍生出了8种后缀：

**ia（increase after**:先传输，再地址+4

**ib（increase before**: 先地址+4，再传输

**da（decrease after**: 先传输，再地址-4

**db（decrease befor**: 先地址-4，再传输

fd（full decrease）:满递减堆栈

ed（empty decrease）:空递减堆栈

fa：满递增堆栈

ea：空递增堆栈

**在理解的时候我们就可以将其拆开理解，比如LDMIA 就是LDM IA 两部分理解一些例子：**

  

从地址 r0 开始读取 4 个字（word），分别存储到 r1-r4 寄存器中，然后将地址 r0 增加 4。

LDMIA r0!, {r1-r4}  

  

将地址 r0 增加 4，然后从新地址开始读取 3 个字，分别存储到 r1-r3 寄存器中。

LDMIB r0, {r1-r3}  

  

从地址 r0 开始读取 2 个字，分别存储到 r1-r2 寄存器中，然后将地址 r0 减少 4，并将读取的最后一个字存储到 lr 寄存器中。

STMFD sp!, {r1-r3, lr}  

  

将 r1-r3 寄存器中的数据存储到地址 sp 指向的存储器中，然后将 sp 减少 16（4 个字），并将 lr 寄存器中的数据存储到新的地址 sp 指向的存储器中。

LDMDB r0!, {r1-r2, lr}  

##   

## 指令中！的作用

  

一般的，当感叹号 "!" 出现在寄存器名称后面时，就表示在执行指令后会更新该寄存器的值。如果没有！，表示执行指令前不更新该寄存器的值。

ldmia	r0, {r2 - r3}  
ldmia	r0！, {r2 - r3}  

  

这里感叹号的作用就是r0的值在ldm过程中发生的增加或者减少最后写回到r0去，第二句的ldm时会改变r0的值，而第一句，没有!，运行之后不会更新 r0 的值。

  

## 指令中^的作用

  

^的作用：在目标寄存器中有pc时，会同时将spsr写入到cpsr，一般用于从异常模式返回。

ldmfd	sp!, {r0 - r6, pc}  
ldmfd	sp!, {r0 - r6, pc}^  

##   

## 汇编不同写法，对于寄存器的改变(先算括号）

  

**偏移量方法**

LDR R0,[R1, #4]  

  

实现的操作：

r0= *(r1 + 4)

  

**事先更新**

LDR R0,[R1, #4]!  

  

实现的操作：

r0 = *（r1 + 4)

r1 = r1 +4

  

**事后更新:**

LDR R0,[R1], #4  

  

实现的操作：

r0=*r1  
r1=r1+4

  

  

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

**看雪ID：二木先生**

https://bbs.kanxue.com/user-home-979211.htm

*本文为看雪论坛优秀文章，由 二木先生 原创，转载请注明来自看雪社区

  

[![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458551171&idx=1&sn=431e80c01dcbbcf125888660d147922f&chksm=b18db30986fa3a1f15c06232425f27c882e304b7be1fb3a7e564c03e76330c093cf2fe6d1625&scene=21#wechat_redirect)

  

  

**#** **往期推荐**

1、[自定义Linker实现分析之路](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458550539&idx=2&sn=e3a883e6de9929783e4920b1ae75802d&chksm=b18db18186fa38971cf9a67439421e62a1c3e1dbeb2cdc974c70ab52186fe92738ed759cf003&scene=21#wechat_redirect)

2、[逆向分析VT加持的无畏契约纯内核挂](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458550427&idx=1&sn=399ad869e9f33b368de123b079ca1ff2&chksm=b18db01186fa390707f03c65e957277ed4eb7d250bbce02130ab2d6324c0c4cd9ab837e01802&scene=21#wechat_redirect)

3、[阿里云CTF2024-暴力ENOTYOURWORLD题解](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458550386&idx=1&sn=ef197d9dc41313d624d8e297d6cc5f9a&chksm=b18db0f886fa39eedca81d2ebee9e73e689d9db0bfdcb9831d8ebe4a759a5c55f98aff2a771b&scene=21#wechat_redirect)

4、[Hypervisor From Scratch - 基本概念和配置测试环境、进入 VMX 操作](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458550275&idx=1&sn=c1b54dc12abbcb627796db92d4f9c2fc&chksm=b18db08986fa399ff036a52bbbe579808ba65111151b31af848628a464efe064e4fbd7c6c1d9&scene=21#wechat_redirect)

5、[V8漏洞利用之对象伪造漏洞利用模板](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458550274&idx=1&sn=83844418c6e1fb22d4d8c2033abdea5e&chksm=b18db08886fa399ee2927fefc6f01c0213e126ef3248a8ecc439231526e9e56e69f937a29a3c&scene=21#wechat_redirect)

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**球分享**

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**球点赞**

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**球在看**

  

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

点击阅读原文查看更多

阅读原文

阅读 2367

​

写留言

**留言 1**

- CyberPi
    
    湖北4月26日
    
    赞2
    
    为啥没有 ARM64 的架构？
    

已无更多数据

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/1UG7KPNHN8EGLfh77kFmnicd9WOic2ibvhCibFdB4bL4srJCgo2wnvdoXLxpIvAkfCmmcptXZB0qKWMoIP8iaibYN2FA/300?wx_fmt=png&wxfrom=18)

看雪学苑

181317

1

写留言

**留言 1**

- CyberPi
    
    湖北4月26日
    
    赞2
    
    为啥没有 ARM64 的架构？
    

已无更多数据