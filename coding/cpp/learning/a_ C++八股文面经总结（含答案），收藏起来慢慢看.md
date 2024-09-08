

Linux开发架构之路

 _2024年05月14日 17:37_ _湖南_

### 

1. 编译内存相关

### 

1.1. C++ 程序编译过程

编译过程分为四个过程：编译（编译预处理、编译、优化），汇编，链接。

编译预处理：处理以 # 开头的指令，产生 .i 文件；

主要的处理操作如下：

- 对全部的#define进行宏展开。
    
- 处理全部的条件编译指令，比方#if、#ifdef、#elif、#else、#endif;
    
- 处理 #include 指令，这个过程是递归的，也就是说被包括的文件可能还包括其它文件;
    
- 删除全部的注释 // 和 /**/
    
- 加入行号和文件标识
    
- 保留全部的 #pragma 编译器指令
    

ps:经过预处理后的 .i 文件不包括任何宏定义，由于全部的宏已经被展开。而且包括的文件也已经被插入到 .i 文件里。

编译、优化：将源码 .cpp 文件翻译成 .s 汇编代码；

- 词法分析：将源代码的字符序列分割成一系列的记号。
    
- 语法分析：对记号进行语法分析，产生语法树。
    
- 语义分析：判断表达式是否有意义。
    
- 代码优化：
    
- 目标代码生成：生成汇编代码。
    
- 目标代码优化：
    

编译会将源代码由文本形式转换成机器语言，编译过程就是把预处理完的文件进行一系列词法分析、语法分析、语义分析以及优化后生成相应的汇编代码文件。编译后的.s是ASCII码文件。

汇编：将汇编代码 .s 翻译成机器指令的 .o 或.obj 目标文件；

- 汇编过程调用汇编器AS来完成，是用于将汇编代码转换成机器可以执行的指令，每一个汇编语句几乎都对应一条机器指令。
    
- 汇编后的.o文件是纯二进制文件。
    

链接：产生 .out 或 .exe 可运行文件

汇编程序生成的目标文件，即 .o 文件，并不会立即执行，因为可能会出现：.cpp 文件中的函数引用了另一个 .cpp文件中定义的符号或者调用了某个库文件中的函数。那链接的目的就是将这些文件对应的目标文件连接成一个整体，从而生成可执行的程序 .exe文件。

详细来说，链接是将所有的.o文件和库（动态库、静态库）链接在一起，得到可以运行的可执行文件（Windows的.exe文件或Linux的.out文件）等。它的工作就是把一些指令对其他符号地址的引用加以修正。链接过程主要包括了地址和空间分配、符号决议和重定向。

最基本的链接叫做静态链接，就是将每个模块的源代码文件编译、汇编成目标文件（Linux：.o 文件；Windows：.obj文件），然后将目标文件和库一起链接形成最后的可执行文件（.exe或.out等）。库其实就是一组目标文件的包，就是一些最常用的代码变异成目标文件后打包存放。最常见的库就是运行时库，它是支持程序运行的基本函数的集合。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

链接分为两种：

静态链接：代码从其所在的静态链接库中拷贝到最终的可执行程序中，在该程序被执行时，这些代码会被装入到该进程的虚拟地址空间中。

把目标程序运行时需要调用的函数代码直接链接到了生成的可执行文件中，程序在运行的时候不需要其他额外的库文件，且就算你去静态库把程序执行需要的库删掉也不会影响程序的运行，因为所需要的所有东西已经被链接到了链接阶段生成的可执行文件中。

Windows下以.lib为后缀，Linux下以.a为后缀。

动态链接：代码被放到动态链接库或共享对象的某个目标文件中，链接程序只是在最终的可执行程序中记录了共享对象的名字等一些信息。在程序执行时，动态链接库的全部内容会被映射到运行时相应进行的虚拟地址的空间。

动态 “动” 在了程序在执行阶段需要去寻找相应的函数代码，即在程序运行时才会将程序安装模块链接在一起

具体来说，动态链接就是把调⽤的函数所在⽂件模块（DLL ）和调⽤函数在⽂件中的位置等信息链接进目标程序，程序运⾏的时候再从 DLL 中寻找相应函数代码，因此需要相应 DLL ⽂件的⽀持 。（Windows）

包含函数重定位信息的文件，在Windows下以.dll为后缀，Linux下以.so为后缀。

二者的区别：

静态链接是 将各个模块的obj和库链接成一个完整的可执行程序；而动态链接是程序在运行的时候寻找动态库的函数符号（重定位），即DLL不必被包含在最终的exe文件中；

链接使用工具不同:

- 静态链接由称为“链接器”的工具完成；
    
- 动态链接由操作系统在程序运行时完成链接；
    

库包含限制：

- 静态链接库中不能再包含其他的动态链接库或者静态库；
    
- 动态链接库中还可以再包含其他的动态或静态链接库。
    

运行速度：

- 静态链接运行速度快（因为执行过程中不用重定位），可独立运行
    
- 动态链接运行速度慢、不可独立运行
    

二者的优缺点：

静态链接：浪费空间，每个可执行程序都会有目标文件的一个副本，这样如果目标文件进行了更新操作，就需要重新进行编译链接生成可执行程序（更新困难）；优点就是执行的时候运行速度快，因为可执行程序具备了程序运行的所有内容。

动态链接：节省内存、更新方便，但是动态链接是在程序运行时，每次执行都需要链接，相比静态链接会有一定的性能损失。

### 

1.2. C++ 内存管理

C++的内存分布模型：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

从高地址到低地址，一个程序由 内核空间、栈区、堆区、BSS段、数据段（data）、代码区组成。

常说的C++ 内存分区：栈、堆、全局/静态存储区、常量存储区、代码区。

可执行程序在运行时会多出两个区域：

- 栈：存放函数的局部变量、函数参数、返回地址等，由编译器自动分配和释放。栈从高地址向低地址增长。是一块连续的空间。栈一般分配几M大小的内存。
    
- 堆：动态申请的内存空间，就是由 malloc 分配的内存块，由程序员控制它的分配和释放，如果程序执行结束还没有释放，操作系统会自动回收。堆从低地址向高地址增长。一般可以分配几个G大小的内存。
    
- 在堆栈之间有一个 共享区（文件映射区）。
    
- 全局区/静态存储区（.BSS 段和 .data 段）：存放全局变量和静态变量，程序运行结束操作系统自动释放，在 C 语言中，程序中未初始化的全局变量和静态变量存放在.BSS 段中，已初始化的全局变量和静态变量存放在 .data 段中，C++ 中不再区分了。常量存储区（.data 段）：存放的是常量，不允许修改，程序运行结束自动释放。
    
- 代码区（.text 段）：存放程序执行代码的一块内存区域。只读，不允许修改，但可以执行。编译后的二进制文件存放在这里。代码段的头部还会包含一些只读的常量，如字符串常量字面值（注意：const变量虽然属于常量，但是本质还是变量，不存储于代码段）
    

在linux下size命令可以查看一个可执行二进制文件基本情况：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

### 

1.3. 栈和堆的区别

- 申请方式：栈是系统自动分配，堆是程序员主动申请。
    
- 申请后系统响应：分配栈空间，如果剩余空间大于申请空间则分配成功，否则分配失败栈溢出；申请堆空间，堆在内存中呈现的方式类似于链表（记录空闲地址空间的链表），在链表上寻找第一个大于申请空间的节点分配给程序，将该节点从链表中删除，大多数系统中该块空间的首地址存放的是本次分配空间的大小，便于释放，将该块空间上的剩余空间再次连接在空闲链表上。
    
- 栈在内存中是连续的一块空间（向低地址扩展）最大容量是系统预定好的，堆在内存中的空间（向高地址扩展）是不连续的。
    
- 申请效率：栈是有系统自动分配，申请效率高，但程序员无法控制；堆是由程序员主动申请，效率低，使用起来方便但是容易产生碎片。
    
- 存放的内容：栈中存放的是局部变量，函数的参数；堆中存放的内容由程序员控制。
    

此题总结：

1、申请方式的不同。 栈由系统自动分配，而堆是人为申请开辟;

2、申请大小的不同。 栈获得的空间较小，而堆获得的空间较大;

3、申请效率的不同。 栈由系统自动分配，速度较快，而堆一般速度比较慢;

4、 存储的内容不同。

栈在函数调用时，第一个进栈的是主函数中后的下一条指令（函数调用语句的下一条可执行语句）的地址，然后是函数的各个参数，在大多数的C编译器中，参数是由右往左入栈的，然后是函数中的局部变量。注意静态变量是不入栈的。 当本次函数调用结束后，局部变量先出栈，然后是参数，最后栈顶指针指向最开始存的地址，也就是主函数中的下一条指令，程序由该点继续运行。

堆：一般是在堆的头部用一个字节存放堆的大小。堆中的具体内容由程序员安排。

### 

1.4. 变量的区别

全局变量、局部变量、静态全局变量、静态局部变量的区别：

全局变量就是定义在函数外的变量。

局部变量就是函数内定义的变量。

静态变量就是加了static的变量。 例如：static int value = 1

各自存储的位置：

- 全局变量，存储在常量区（静态存储区）。
    
- 静态变量，存储在常量区（静态存储区）。
    
- 局部变量, 存储在栈区。
    

注意: 因为静态变量都在静态存储区（常量区），所以下次调用函数的时候还是能取到原来的值。

各自初始化的值：

- 局部变量, 存储在栈区。局部变量一般是不初始化的，
    
- 局部变量, 存储在栈区。全局变量和静态变量，都是初始化为0的，有一个初始值。
    
- 局部变量, 存储在栈区。如果是类变量，会调用默认构造函数初始化。
    

从作用域看：

C++ 变量根据定义的位置的不同的生命周期，具有不同的作用域，作用域可分为 6 种：全局作用域，局部作用域，语句作用域，类作用域，命名空间作用域和文件作用域。

- 全局变量：具有全局作用域。全局变量只需在一个源文件中定义，就可以作用于所有的源文件。当然，其他不包含全局变量定义的源文件需要用extern 关键字再次声明这个全局变量。会一直存在到程序结束。
    
- 静态全局变量：全局作用域+文件作用域，所以无法在其他文件中使用。它与全局变量的区别在于如果程序包含多个文件的话，它作用于定义它的文件里，不能作用到其它文件里，即被static 关键字修饰过的变量具有文件作用域。这样即使两个不同的源文件都定义了相同名字的静态全局变量，它们也是不同的变量。
    
- 局部变量：具有局部作用域。比如函数的参数，函数内的局部变量等等；它是自动对象（auto），在程序运行期间不是一直存在，而是只在函数执行期间存在，函数的一次调用执行结束后，变量被销毁，其所占用的内存也被收回。
    
- 静态局部变量：具有局部作用域。它只被初始化一次， 直到程序结束。自从第一次被初始化直到程序运行结束都一直存在，它和全局变量的区别在于全局变量对所有的函数都是可见的，而静态局部变量只对定义自己的函数体始终可见。
    

从分配内存空间看：

- 静态存储区：全局变量，静态局部变量，静态全局变量。
    
- 栈：局部变量。
    

各自的应用场景：

- 局部变量就是我们经常用的，进入函数，逐个构造，最后统一销毁。
    
- 全局变量主要是用来给不同的文件之间进行通信。
    
- 静态变量：只在本文件中使用，局部静态变量在函数内起作用，可以作为一个计数器。
    

例子：

   void func(){  
     static int count;  
     count ++;  
   }  
   int main(int argc, char** argv){  
     for(int i = 0; i < 10; i++)  
       func();  
   }

说说静态变量在代码执行的什么阶段进行初始化？

static int value  //静态变量初始化语句

对于C语言： 静态变量和全局变量均在编译期进行初始化，即初始化发生在任何代码执行之前。

对于C++： 静态变量和全局变量仅当首次被使用的时候才进行初始化。

助记： 如果你使用过C/C++你会发现，C语言要求在程序的最开头声明全部的变量，而C++则可以随时使用随时声明；这个规律是不是和答案类似呢？

**需要更多大厂面试题，扫描下方二维码领取**  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

### 

1.5. 全局变量定义在头文件中有什么问题？

如果在头文件中定义全局变量，当该头文件被多个文件 include 时，该头文件中的全局变量就会被定义多次，导致重复定义，因此不能再头文件中定义全局变量。

### 

1.6. 内存对齐

什么是内存对齐？内存对齐的原则？为什么要进行内存对齐，有什么优点？

内存对齐：编译器将程序中的每个“数据单元”安排在字的整数倍的地址指向的内存之中

内存对齐的原则：

1. 结构体变量的首地址能够被其最宽基本类型成员大小与对齐基数中的较小者所整除；
    
2. 结构体每个成员相对于结构体首地址的偏移量 （offset）都是该成员大小与对齐基数中的较小者的整数倍，如有需要编译器会在成员之间加上填充字节 （internal padding）；
    
3. 结构体的总大小为结构体最宽基本类型成员大小与对齐基数中的较小者的整数倍，如有需要编译器会在最末一个成员之后加上填充字节（trailing padding）。
    

进行内存对齐的原因：（主要是硬件设备方面的问题）

1. 某些硬件设备只能存取对齐数据，存取非对齐的数据可能会引发异常；
    
2. 某些硬件设备不能保证在存取非对齐数据的时候的操作是原子操作；
    
3. 相比于存取对齐的数据，存取非对齐的数据需要花费更多的时间；
    
4. 某些处理器虽然支持非对齐数据的访问，但会引发对齐陷阱（alignmenttrap）；
    
5. 某些硬件设备只支持简单数据指令非对齐存取，不支持复杂数据指令的非对齐存取。
    

内存对齐的优点：

1. 便于在不同的平台之间进行移植，因为有些硬件平台不能够支持任意地址的数据访问，只能在某些地址处取某些特定的数据，否则会抛出异常；
    
2. 提高内存的访问效率，因为 CPU 在读取内存时，是一块一块的读取。
    

### 

1.7. 什么是内存泄露

内存泄漏：由于疏忽或错误导致的程序未能释放已经不再使用的内存。

进一步解释：

- 并非指内存从物理上消失，而是指程序在运行过程中，由于疏忽或错误而失去了对该内存的控制，从而造成了内存的浪费。
    
- 常指堆内存泄漏，因为堆是动态分配的，而且是用户来控制的，如果使用不当，会产生内存泄漏。
    
- 使用 malloc、calloc、realloc、new 等分配内存时，使用完后要调用相应的 free 或 delete释放内存，否则这块内存就会造成内存泄漏。
    
- 指针重新赋值
    

char *p = (char *)malloc(10);  
char *p1 = (char *)malloc(10);  
p = np;

开始时，指针 p 和 p1 分别指向一块内存空间，但指针 p 被重新赋值，导致 p 初始时指向的那块内存空间无法找到，从而发生了内存泄漏。

### 

1.8. 怎么防止内存泄漏？内存泄漏检测工具的原理？

防止内存泄漏的方法：

- 内部封装：将内存的分配和释放封装到类中，在构造的时候申请内存，析构的时候释放内存。（说明：但这样做并不是最佳的做法，在类的对象复制时，程序会出现同一块内存空间释放两次的情况）
    
- 智能指针：智能指针是 C++ 中已经对内存泄漏封装好了一个工具，可以直接拿来使用，将在下一个问题中对智能指针进行详细的解释。
    

VS下内存泄漏的检测方法（CRT）：

在debug模式下以F5运行：

#define CRTDBG_MAP_ALLOC    
#include <stdlib.h>    
#include <crtdbg.h>    
//在入口函数中包含 _CrtDumpMemoryLeaks();    
//即可检测到内存泄露  
   
//以如下测试函数为例：  
int main(){  
	char* pChars = new char[10];  
	_CrtDumpMemoryLeaks();  
	return 0;  
}

### 

1.9. 智能指针有哪几种？智能指针的实现原理？

智能指针是为了解决动态内存分配时忘记释放内存导致的内存泄漏以及多次释放同一块内存空间而提出的。C++11 中封装在了 #include < memory > 头文件中。

C++11 引入了 3 个智能指针类型：

std::unique_ptr ：独占资源所有权的指针。 std::shared_ptr ：共享资源所有权的指针。 std::weak_ptr ：共享资源的观察者，需要和 std::shared_ptr 一起使用，不影响资源的生命周期。

注：std::auto_ptr 已被废弃。

共享指针（shared_ptr）：资源可以被多个指针共享，使用计数机制表明资源被几个指针共享。通过 use_count() 查看资源的所有者的个数，可以通过 unique_ptr、weak_ptr 来构造，调用 release() 释放资源的所有权，计数减一，当计数减为 0 时，会自动释放内存空间，从而避免了内存泄漏。

独占指针（unique_ptr）：独享所有权的智能指针，资源只能被一个指针占有，该指针不能拷贝构造和赋值。但可以进行移动构造和移动赋值构造（调用move() 函数），即一个 unique_ptr 对象赋值给另一个 unique_ptr 对象，可以通过该方法进行赋值。

弱指针（weak_ptr）：指向 shared_ptr 指向的对象，能够解决由shared_ptr带来的循环引用问题。

智能指针的实现原理： 计数原理。

### 

1.10 智能指针应用举例

unique_ptr

unique_ptr 的使用比较简单，也是用得比较多的智能指针。当我们独占资源的所有权的时候，可以使用 unique_ptr 对资源进行管理——离开 unique_ptr 对象的作用域时，会自动释放资源。这是很基本的 RAII 思想。

- 自动管理内存
    

使用裸指针时，要记得释放内存。

{  
    int* p = new int(100);  
    // ...  
    delete p;  // 要记得释放内存  
}

使用 unique_ptr 自动管理内存。

{  
    std::unique_ptr<int> uptr = std::make_unique<int>(200);  
    //...  
    // 离开 uptr 的作用域的时候自动释放内存  
}

- unique_ptr 是 move-only 的，也是实现将一个 unique_ptr 对象赋值给另一个 unique_ptr 对象的方法
    

{  
    std::unique_ptr<int> uptr = std::make_unique<int>(200);  
    std::unique_ptr<int> uptr1 = uptr;  // 编译错误，std::unique_ptr<T> 是 move-only 的  
  
    std::unique_ptr<int> uptr2 = std::move(uptr);  
    assert(uptr == nullptr);  
}

- unique_ptr 可以指向一个数组
    

{  
    std::unique_ptr<int[]> uptr = std::make_unique<int[]>(10);  
    for (int i = 0; i < 10; i++) {  
        uptr[i] = i * i;  
    }     
    for (int i = 0; i < 10; i++) {  
        std::cout << uptr[i] << std::endl; //0 1 4 9 ...81  
    }     
}  
也可以用向量：  
unique_ptr<vector<int>> p (new vector<int>(5, 6)); //n = 5, value = 6  
std::cout << *p->begin() << endl;//6

shared_ptr

- shared_ptr 其实就是对资源做引用计数——当引用计数 sptr.use_count() 为 0的时候，自动释放资源。其中，assert(p);用于判断指针内容是否非空，空指针nullptr 与什么未指向的野指针过不了assert
    

{  
    std::shared_ptr<int> sptr = std::make_shared<int>(200);  
    assert(sptr.use_count() == 1);  // 此时引用计数为 1  
    {     
        std::shared_ptr<int> sptr1 = sptr;  
        assert(sptr.get() == sptr1.get());  
        assert(sptr.use_count() == 2);   // sptr 和 sptr1 共享资源，引用计数为 2  
    }     
    assert(sptr.use_count() == 1);   // sptr1 已经释放  
}  
// use_count 为 0 时自动释放内存

和 unique_ptr 一样，shared_ptr 也可以指向数组和自定义 deleter。

{  
    // C++20 才支持 std::make_shared<int[]>  
    // std::shared_ptr<int[]> sptr = std::make_shared<int[]>(100);  
    std::shared_ptr<int[]> sptr(new int[10]);  
    for (int i = 0; i < 10; i++) {  
        sptr[i] = i * i;  
    }     
    for (int i = 0; i < 10; i++) {  
        std::cout << sptr[i] << std::endl;  
    }     
}

附：

一个 shared_ptr 对象的内存开销要比裸指针和无自定义 deleter 的 unique_ptr 对象略大。

无自定义 deleter 的 unique_ptr 只需要将裸指针用 RAII 的手法封装好就行，无需保存其它信息，所以它的开销和裸指针是一样的。如果有自定义 deleter，还需要保存 deleter 的信息。

shared_ptr 需要维护的信息有两部分：

1. 指向共享资源的指针。
    
2. 引用计数等共享资源的控制信息——实现上是维护一个指向控制信息的指针。
    

所以，shared_ptr 对象需要保存两个指针。shared_ptr 的 的 deleter 是保存在控制信息中，所以，是否有自定义 deleter 不影响 shared_ptr 对象的大小。

当我们创建一个 shared_ptr 时，其实现一般如下：

std::shared_ptr<T> sptr1(new T);  
最好使用make_shared实现：  
shared_ptr<string> p1 = make_shared<string>(10, '9');  
shared_ptr<int> p2 = make_shared<int>(42);

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

复制一个 shared_ptr ：

std::shared_ptr<T> sptr2 = sptr1;

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

为什么控制信息和每个 shared_ptr 对象都需要保存指向共享资源的指针？可不可以去掉 shared_ptr 对象中指向共享资源的指针，以节省内存开销？

答案是：不能。 因为 shared_ptr 对象中的指针指向的对象不一定和控制块中的指针指向的对象一样。

来看一个例子。

struct Fruit {  
    int juice;  
};  
  
struct Vegetable {  
    int fiber;  
};  
  
struct Tomato : public Fruit, Vegetable {  
    int sauce;  
};  
  
 // 由于继承的存在，shared_ptr 可能指向基类对象  
std::shared_ptr<Tomato> tomato = std::make_shared<Tomato>();  
std::shared_ptr<Fruit> fruit = tomato;  
std::shared_ptr<Vegetable> vegetable = tomato;

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

此外，在使用 shared_ptr 时，会涉及两次内存分配：一次分配共享资源对象；一次分配控制块。C++ 标准库提供了 make_shared 函数来创建一个 shared_ptr 对象，只需要一次内存分配，所以推荐用make_shared 函数来创建对象。

weak_ptr

weak_ptr 要与 shared_ptr 一起使用。 一个 weak_ptr 对象看做是 shared_ptr 对象管理的资源的观察者，它不影响共享资源的生命周期：

1. 如果需要使用 weak_ptr 正在观察的资源，可以将 weak_ptr 提升为 shared_ptr。
    
2. 当 shared_ptr 管理的资源被释放时，weak_ptr 会自动变成 nullptr。
    

void Observe(std::weak_ptr<int> wptr) {  
    if (auto sptr = wptr.lock()) {  
        std::cout << "value: " << *sptr << std::endl;  
    } else {  
        std::cout << "wptr lock fail" << std::endl;  
    }  
}  
  
std::weak_ptr<int> wptr;  
{  
    auto sptr = std::make_shared<int>(111);  
    wptr = sptr;  
    Observe(wptr);  // sptr 指向的资源没被释放，wptr 可以成功提升为 shared_ptr  
}  
Observe(wptr);  // sptr 指向的资源已被释放，wptr 无法提升为 shared_ptr

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

当 shared_ptr 析构并释放共享资源的时候，只要 weak_ptr 对象还存在，控制块就会保留，weak_ptr 可以通过控制块观察到对象是否存活。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

### 

1.11 一个 unique_ptr 怎么赋值给另一个 unique_ptr 对象？

借助 std::move() 可以实现将一个 unique_ptr 对象赋值给另一个 unique_ptr 对象，其目的是实现所有权的转移。

// A 作为一个类   
std::unique_ptr<A> ptr1(new A());  
std::unique_ptr<A> ptr2 = std::move(ptr1);

### 

1.12 使用智能指针会出现什么问题？怎么解决？

智能指针可能出现的问题：循环引用

比如定义了两个类 Parent、Child，在两个类中分别定义另一个类的对象的共享指针，由于在程序结束后，两个指针相互指向对方的内存空间，导致内存无法释放。

循环引用的解决方法： weak_ptr

循环引用：该被调用的析构函数没有被调用，从而出现了内存泄漏。

weak_ptr 对被 shared_ptr 管理的对象存在非拥有性（弱）引用，在访问所引用的对象前必须先转化为 shared_ptr；

weak_ptr 用来打断 shared_ptr 所管理对象的循环引用问题，若这种环被孤立（没有指向环中的外部共享指针），shared_ptr 引用计数无法抵达 0，内存被泄露；令环中的指针之一为弱指针可以避免该情况；

weak_ptr 用来表达临时所有权的概念，当某个对象只有存在时才需要被访问，而且随时可能被他人删除，可以用 weak_ptr 跟踪该对象；需要获得所有权时将其转化为 shared_ptr，此时如果原来的 shared_ptr 被销毁，则该对象的生命期被延长至这个临时的 shared_ptr 同样被销毁。

### 

1.13 VS检测内存泄漏，定位泄漏代码位置方法

检查方法： 在main函数最后面一行，加上一句_CrtDumpMemoryLeaks()。调试程序，自然关闭程序让其退出（不要定制调试），查看输出：

Detected memory leaks!  
Dumping objects ->  
{453} normal block at 0x02432CA8, 868 bytes long.  
 Data: <404303374       > 34 30 34 33 30 33 33 37 34 00 00 00 00 00 00 00   
{447} normal block at 0x024328B0, 868 bytes long.  
 Data: <404303374       > 34 30 34 33 30 33 33 37 34 00 00 00 00 00 00 00   
{441} normal block at 0x024324B8, 868 bytes long.  
 Data: <404303374       > 34 30 34 33 30 33 33 37 34 00 00 00 00 00 00 00   
{435} normal block at 0x024320C0, 868 bytes long.  
 Data: <404303374       > 34 30 34 33 30 33 33 37 34 00 00 00 00 00 00 00   
{429} normal block at 0x02431CC8, 868 bytes long.  
 Data: <404303374       > 34 30 34 33 30 33 33 37 34 00 00 00 00 00 00 00   
{212} normal block at 0x01E1BF30, 44 bytes long.  
 Data: <`               > 60 B3 E1 01 CD CD CD CD CD CD CD CD CD CD CD CD   
{204} normal block at 0x01E1B2C8, 24 bytes long.  
 Data: <                > C8 B2 E1 01 C8 B2 E1 01 C8 B2 E1 01 CD CD CD CD   
{138} normal block at 0x01E15680, 332 bytes long.  
 Data: <                > 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00   
{137} normal block at 0x01E15628, 24 bytes long.  
 Data: <(V  (V  (V      > 28 56 E1 01 28 56 E1 01 28 56 E1 01 CD CD CD CD   
Object dump complete.

取其中一条详细说明：{453} normal block at 0x02432CA8, 868 bytes long.

被{}包围的453就是我们需要的内存泄漏定位值，868 bytes long就是说这个地方有868比特内存没有释放。

在main函数第一行加上：_CrtSetBreakAlloc(453);意思就是在申请453这块内存的位置中断。然后调试程序，……程序中断了。查看调用堆栈

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

双击我们的代码调用的最后一个函数，这里是CDbQuery::UpdateDatas()，就定位到了申请内存的代码：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

好了，我们总算知道是哪里出问题了，这块内存没有释放啊。改代码，修复好这个。然后继续…………，直到调试输出中没有normal block ，程序没有内存泄漏了。

记得加上头文件：#include <crtdbg.h>

最后要注意一点的，并不是所有normal block一定就有内存泄漏，当你的程序中有全局变量的时候，全局变量的释放示在main函数退出后，所以在main函数最后_CrtDumpMemoryLeaks（）会认为全局申请的内存没有释放，造成内存泄漏的假象。如何规避呢？我通常是把全局变量声明成指针在main函数中new 在main函数中delete，然后再调用_CrtDumpMemoryLeaks（），这样就不会误判了。

请自行查阅 Linux检测内存泄漏，定位泄漏代码位置方法

### 

1.14 深拷贝与浅拷贝

- c++默认的拷贝构造函数是浅拷贝
    

浅拷贝就是对象的数据成员之间的简单赋值，如你设计了一个类而没有提供它的复制构造函数，当用该类的一个对象去给另一个对象赋值时所执行的过程就是浅拷贝。当数据成员中没有指针时，浅拷贝是可行的；但当数据成员中有指针时，如果采用简单的浅拷贝，则两类中的两个指针将指向同一个地址，当对象快结束时，会调用两次析构函数，而导致指针悬挂现象，所以，此时，必须采用深拷贝。

- 深拷贝与浅拷贝的区别就在于深拷贝会在堆内存中另外申请空间来储存数据，而不是一个简单的赋值过程，从而也就解决了指针悬挂的问题。
    

**需要更多大厂面试题，扫描下方二维码领取**  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

### 

1.15 虚拟内存

物理内存：

物理内存实际上是CPU中能直接寻址的地址线条数。由于物理内存是有限的，例如32位平台下，寻址的大小是4G，并且是固定的。内存很快就会被分配完，于是没有得到分配资源的进程就只能等待。当一个进程执行完了以后，再将等待的进程装入内存。这种频繁的装入内存的操作是很没效率的。

虚拟内存：

- 在进程创建的时候，系统都会给每个进程分配4G的内存空间，这其实是虚拟内存空间。进程得到的这4G虚拟内存，进程自身以为是一段连续的空间，而实际上，通常被分隔成多个物理内存碎片，还有一部分存储在外部磁盘存储器上，需要的时候进行数据交换。
    

关于虚拟内存与物理内存的联系，下面这张图可以帮助我们巩固。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

虚拟内存机理及优点：

虚拟内存是如何工作的？

- 当每个进程创建的时候，内核会为进程分配4G的虚拟内存，当进程还没有开始运行时，这只是一个内存布局。实际上并不立即就把虚拟内存对应位置的程序数据和代码（比如.text .data段）拷贝到物理内存中，只是建立好虚拟内存和磁盘文件之间的映射就好（叫做存储器映射）。这个时候数据和代码还是在磁盘上的。当运行到对应的程序时，进程去寻找页表，发现页表中地址没有存放在物理内存上，而是在磁盘上，于是发生缺页异常，于是将磁盘上的数据拷贝到物理内存中。
    
- 另外在进程运行过程中，要通过malloc来动态分配内存时，也只是分配了虚拟内存，即为这块虚拟内存对应的页表项做相应设置，当进程真正访问到此数据时，才引发缺页异常。
    
- 可以认为虚拟空间都被映射到了磁盘空间中（事实上也是按需要映射到磁盘空间上，通过mmap，mmap是用来建立虚拟空间和磁盘空间的映射关系的）
    

利用虚拟内存机制的优点 ？

- 既然每个进程的内存空间都是一致而且固定的（32位平台下都是4G），所以链接器在链接可执行文件时，可以设定内存地址，而不用去管这些数据最终实际内存地址，这交给内核来完成映射关系
    
- 当不同的进程使用同一段代码时，比如库文件的代码，在物理内存中可以只存储一份这样的代码，不同进程只要将自己的虚拟内存映射过去就好了，这样可以节省物理内存
    
- 在程序需要分配连续空间的时候，只需要在虚拟内存分配连续空间，而不需要物理内存时连续的，实际上，往往物理内存都是断断续续的内存碎片。这样就可以有效地利用我们的物理内存
    

### 

2. 语言对比

### 

2.1 C++ 11 新特性

1. auto 类型推导

auto 关键字：自动类型推导，编译器会在 编译期间 通过初始值推导出变量的类型，通过 auto 定义的变量必须有初始值。

2. decltype 类型推导

decltype 关键字：decltype 是“declare type”的缩写，译为“声明类型”。和 auto 的功能一样，都用来在编译时期进行自动类型推导。如果希望从表达式中推断出要定义的变量的类型，但是不想用该表达式的值初始化变量，这时就不能再用 auto。decltype 作用是选择并返回操作数的数据类型。

区别：

auto var = val1 + val2;   
decltype(val1 + val2) var1 = 0; 

- auto 根据 = 右边的初始值 val1 + val2 推导出变量的类型，并将该初始值赋值给变量 var；decltype 根据 val1 + val2 表达式推导出变量的类型，变量的初始值和与表达式的值无关。
    
- auto 要求变量必须初始化，因为它是根据初始化的值推导出变量的类型，而 decltype 不要求，定义变量的时候可初始化也可以不初始化。
    

3. lambda 表达式lambda 表达式，又被称为 lambda 函数或者 lambda 匿名函数。

lambda匿名函数的定义:

[capture list] (parameter list) -> return type  
{  
   function body;  
};

其中：

- capture list：捕获列表，指 lambda 所在函数中定义的局部变量的列表，通常为空。
    
- return type、parameter list、function body：分别表示返回值类型、参数列表、函数体，和普通函数一样。
    

#include <iostream>  
#include <algorithm>  
using namespace std;  
  
int main(){  
    int arr[4] = {4, 2, 3, 1};  
    //对 a 数组中的元素进行升序排序  
    sort(arr, arr+4, [=](int x, int y) -> bool{ return x < y; } );  
    for(int n : arr){  
        cout << n << " ";  
    }  
    return 0;  
}

4. 范围 for 语句

for (declaration : expression){  
    statement  
}

参数的含义：

- expression：必须是一个序列，例如用花括号括起来的初始值列表、数组、vector ，string等，这些类型的共同特点是拥有能返回迭代器的 beign、end 成员。
    
- declaration：此处定义一个变量，序列中的每一个元素都能转化成该变量的类型，常用 auto 类型说明符。
    

5. 左值和右值，左值引用和右值引用左值和右值

- 左值：指表达式结束后依然存在的持久对象，可以取地址，具名变量或对象 。左值符号 &
    

通俗理解：左值是指具有对应的可由用户访问的存储单元，并且能由用户改变其值的量。如一个变量就是一个左值，因为它对应着一个存储单元，并可由编程者通过变量名访问和改变其值。

左值(Lvalue) →→ Location

表示内存中可以寻址，可以给它赋值(const类型的变量例外)

- 右值：表达式结束后就不再存在的临时对象，不可以取地址，没有名字。 右值符号 &&
    

右值Rvalue) →→ Read

表示可以知道它的值（例如常数）

通俗的讲，左值就是能够出现在赋值符号左面的东西，而右值就是那些可以出现在赋值符号右面的东西， 比如int a = b + c;，a 就是一个左值，可以对a取地址，而b+c 就是一个右值，对表达式b+c 取地址会报错。

一个典型的例子

a++： 先使用a的值，再给a加1，作为右值

// a++的实现  
int temp = a;  
a = a + 1;  
return temp;

++a ： 先加再用，作为 左值

a = a + 1;  
return a;

在C++中，临时对象不能作为左值，但是可以作为常量引用，const &。

C++ 11中的std::move可将左值引用转化成右值引用。

C++11中右值又由两个概念组成：将亡值和纯右值。

纯右值和将亡值

在C++98中，右值是纯右值，纯右值指的是临时变量值、不跟对象关联的字面量值。包括非引用的函数返回值、表达式等，比如 2、‘ch’、int func()等。将亡值是C++11新增的、与右值引用相关的表达式。

- 纯右值：非引用返回的临时变量( int func(void))、运算表达式产生的临时变量(b+c)、原始字面量(2)、lambda表达式等。
    
- 将亡值：将要被移动的对象、T&&函数返回值、std::move返回值和转换为T&&的类型的转换函数的返回值。
    

将亡值可以理解为通过“盗取”其他变量内存空间的方式获取到的值。在确保其他变量不再被使用、或即将被销毁时，通过“盗取”的方式可以避免内存空间的释放和分配，能够延长变量值的生命期。

右值引用和左值引用

- 右值引用：绑定到右值的引用，用 && 来获得右值引用，右值引用只能绑定到要销毁的对象。是对一个右值进行引用的类型，标记为T&&。因为右值不具名，是以引用的形式找到它，用引用来表示，右值引用也是引用的引用（我目前是这么想的）。
    
- 左值引用：对一个左值进行引用的类型。常规的引用一般都是左值引用
    

#include <iostream>  
#include <vector>  
using namespace std;  
int main()  
{  
    int var = 42;  
    int &l_var = var;  
    int &&r_var = var; // 错误：不能将右值引用绑定到左值上  
  
    int &&r_var2 = var + 40; // 正确：将 r_var2 绑定到求和结果上  
    return 0;  
}

引用本身不拥有所绑定对象的内存，只是该对象的一个别名，左值引用就是有名变量的别名，右值引用是不具名变量的别名。因此无论左值引用还是右值引用都必须立即进行初始化。

通过右值引用，这个将亡的右值又“重获新生”，它的生命周期与右值引用类型变量的生命周期一样，只要这个右值引用类型的变量还活着，那么这个右值临时量就会一直活着，这是一重要特性，可利用这一点会一些性能优化，避免临时对象的拷贝构造和析构。

左值引用包括常量左值引用和非常量左值引用。非常量左值引用只能接受左值，不能接受右值；常量左值引用是一个“万能”的引用类型，可以接受左值（常量左值、非常量左值）、右值。不过常量左值所引用的右值在它的“余生”中只能是只读的。

int &a = 2;       // 非常量左值引用 绑定到 右值，编译失败  
   
int b = 2;        // b 是非常量左值  
const int &c = b; // 常量左值引用 绑定到 非常量左值，编译通过  
   
const int d = 2;  // d 是常量左值  
const int &e = d; // 常量左值引用 绑定到 常量左值，编译通过  
const int &f =2;  // 常量左值引用 绑定到 右值，编译通过

右值引用通常不能绑定到任何的左值，要想绑定一个左值到右值引用，通常需要std::move()将左值强制转换为右值。比如：

int a;  
int &&r1 = a;             // 编译失败  
int &&r2 = std::move(a);  // 编译通过

简单总结：

- 左值引用, 即&i, 是一种对象类型的引用; 右值引用, 即&&i, 是一种对象值的引用;
    
- move() 函数可以把左值引用, 转换为右值引用;
    
- 左值引用是固定的引用, 右值引用是易变的引用, 只能引用字面值(literals)或临时对象(temporary object);
    
- 右值引用主要应用在移动构造器(move constructor) 和移动-赋值操作符(move-assignment operator)上面;
    

代码如下

#include <iostream>    
#include <utility>    
    
int main (void) {    
    int i = 42;    
    int &lr = i;    
    int &&rr = i*42;    
    const int &lr1 = i*42;    
    int &&rr1 = 42;    
    int &&rr2 = std::move(lr);    
    std::cout << "i = " << i << std::endl;    
    std::cout << "lr = " << lr << std::endl;    
    std::cout << "rr = " << rr << std::endl;    
    std::cout << "lr1 = " << lr1  <<std::endl;    
    std::cout << "rr1  = " << rr1  << std::endl;    
    std::cout << "rr2  = " << rr2  << std::endl;    
}  

6. 标准库 move() 函数

move() 函数：通过该函数可获得绑定到左值上的右值引用，该函数包括在 utility 头文件中。该知识点会在后续的章节中做详细的说明。

7. 智能指针相关知识已在第一章中进行了详细的说明，这里不再重复。

8. delete 函数和 default 函数

- delete 函数：= delete 表示该函数不能被调用。
    
- default 函数：= default 表示编译器生成默认的函数，例如：生成默认的构造函数。
    

#include <iostream>  
using namespace std;  
  
class A  
{  
public:  
	A() = default; // 表示使用默认的构造函数  
	~A() = default;	// 表示使用默认的析构函数  
	A(const A &) = delete; // 表示类的对象禁止拷贝构造  
	A &operator=(const A &) = delete; // 表示类的对象禁止拷贝赋值  
};  
int main()  
{  
	A ex1;  
	A ex2 = ex1; // error: use of deleted function 'A::A(const A&)'  
	A ex3;  
	ex3 = ex1; // error: use of deleted function 'A& A::operator=(const A&)'  
	return 0;  
}

### 

2.2 C 和 C++ 的区别

首先说一下面向对象和面向过程：

- 面向过程的思路：面向过程编程就是分析出解决问题的步骤，然后把这些步骤一步一步的实现，使用的时候一个一个的依次调用就可以了。
    
- 面向对象的思路：面向对象编程就是把问题分解成各个对象，建立对象的目的不是为了完成一个步骤，而是为了描述某个事物在整个解决问题的步骤中的行为。
    

举个例子：（玩五子棋）

（1）用面向过程的思想来考虑就是：开始游戏，白子先走，绘制画面，判断输赢，轮到黑子，绘制画面，判断输赢，重复前面的过程，输出最终结果。

（2）用面向对象的思想来考虑就是：黑白双方（两者的行为是一样的）、棋盘系统（负责绘制画面）、规定系统（规定输赢、犯规等）、输出系统（输出赢家）。

面向对象就是高度实物抽象化（功能划分）、面向过程就是自顶向下的编程（步骤划分）

区别和联系：

- C和C++一个典型的区别就在动态内存管理上了，C语言通过malloc和free来进行堆内存的分配和释放，而C++是通过new和delete来管理堆内存的；
    
- 强制类型转换上也不一样，C的强制类型转换使用()小括号里面加类型进行类型强转的，而C++有四种自己的类型强转方式，分别是const_cast，static_cast，reinterpret_cast和dynamic_cast；
    
- C和C++的输入输出方式也不一样，printf/scanf，和C++的cout/cin的对别，前面一组是C的库函数，后面是ostream和istream类型的对象。
    
- C++还支持namespace名字空间，可以让用户自己定义新的名字空间作用域出来，避免全局的名字冲突问题。
    
- 应用领域：C 语言主要用于嵌入式领域，驱动开发等与硬件直接打交道的领域，C++ 可以用于应用层开发，用户界面开发等与操作系统打交道的领域。
    
- C++ 既继承了 C 强大的底层操作特性，又被赋予了面向对象机制。它特性繁多，面向对象语言的多继承，对值传递与引用传递的区分以及 const 关键字，等等。
    
- C++ 对 C 的“增强”，表现在以下几个方面：类型检查更为严格。增加了面向对象的机制、泛型编程的机制（Template）、异常处理、运算符重载、标准模板库（STL）、命名空间（避免全局命名冲突）。
    

面向过程语言：

优点：性能比面向对象高，因为类调用时需要实例化，开销比较大，比较消耗资源;比如单片机、嵌入式开发、Linux/Unix等一般采用面向过程开发，性能是最重要的因素。

缺点：没有面向对象易维护、易复用、易扩展

面向对象语言：

优点：易维护、易复用、易扩展，由于面向对象有封装、继承、多态性的特性，可以设计出低耦合的系统，使系统 更加灵活、更加易于维护

缺点：性能比面向过程低

### 

2.3 Python 和 C++ 的区别

区别：

- 语言自身：Python 为脚本语言，解释执行，不需要经过编译；C++ 是一种需要编译后才能运行的语言，在特定的机器上编译后运行。
    
- 运行效率：C++ 运行效率高，安全稳定。原因：Python 代码和 C++ 最终都会变成 CPU指令来跑，但一般情况下，比如反转和合并两个字符串，Python 最终转换出来的 CPU 指令会比 C++ 多很多。首先，Python中涉及的内容比 C++ 多，经过了更多层，Python 中甚至连数字都是 object ；其次，Python 是解释执行的，和物理机CPU 之间多了解释器这层，而 C++ 是编译执行的，直接就是机器码，编译的时候编译器又可以进行一些优化。
    
- 开发效率：Python 开发效率高。原因：Python 一两句代码就能实现的功能，C++ 往往需要更多的代码才能实现。
    
- 书写格式和语法不同：Python 的语法格式不同于其 C++ 定义声明才能使用，而且极其灵活，完全面向更上层的开发者。
    

### 

3. 面向对象

### 

3.1 什么是面向对象？面向对象的三大特性

面向对象：对象是指具体的某一个事物，这些事物的抽象就是类，类中包含数据（成员变量）和动作（成员方法）。

面向对象的三大特性：

- 封装：将具体的实现过程和数据封装成一个函数，只能通过接口进行访问，降低耦合性。
    
- 继承：子类继承父类的特征和行为，子类有父类的非 private 方法或成员变量，子类可以对父类的方法进行重写，增强了类之间的耦合性，但是当父类中的成员变量、成员函数或者类本身被 final 关键字修饰时，修饰的类不能继承，修饰的成员不能重写或修改。
    
- 多态：多态就是不同继承类的对象，对同一消息做出不同的响应，基类的指针指向或绑定到派生类的对象，使得基类指针呈现不同的表现方式。
    

### 

3.2 重载、重写、隐藏的区别

重载：是指同一可访问区内被声明几个具有不同参数列（参数的类型、个数、顺序）的同名函数，根据参数列表确定调用哪个函数，重载不关心函数返回类型。

class A  
{  
public:  
    void fun(int tmp);  
    void fun(float tmp);        // 重载 参数类型不同（相对于上一个函数）  
    void fun(int tmp, float tmp1); // 重载 参数个数不同（相对于上一个函数）  
    void fun(float tmp, int tmp1); // 重载 参数顺序不同（相对于上一个函数）  
    int fun(int tmp);            // error: 'int A::fun(int)' cannot be overloaded 错误：注意重载不关心函数返回类型  
};

隐藏：是指派生类的函数屏蔽了与其同名的基类函数，主要只要同名函数，不管参数列表是否相同，基类函数都会被隐藏。

#include <iostream>  
using namespace std;  
  
class Base  
{  
public:  
    void fun(int tmp, float tmp1) { cout << "Base::fun(int tmp, float tmp1)" << endl; }  
};  
  
class Derive : public Base  
{  
public:  
    void fun(int tmp) { cout << "Derive::fun(int tmp)" << endl; } // 隐藏基类中的同名函数  
};  
  
int main()  
{  
    Derive ex;  
    ex.fun(1);       // Derive::fun(int tmp)  
    ex.fun(1, 0.01); // error: candidate expects 1 argument, 2 provided  
    return 0;  
}

说明：上述代码中 ex.fun(1, 0.01); 出现错误，说明派生类中将基类的同名函数隐藏了。若是想调用基类中的同名函数，可以加上类型名指明 ex.Base::fun(1, 0.01);，这样就可以调用基类中的同名函数。

重写(覆盖)：是指派生类中存在重新定义的函数。函数名、参数列表、返回值类型都必须同基类中被重写的函数一致，只有函数体不同。派生类调用时会调用派生类的重写函数，不会调用被重写函数。重写的基类中被重写的函数必须有 virtual 修饰。

#include <iostream>  
using namespace std;  
  
class Base  
{  
public:  
    virtual void fun(int tmp) { cout << "Base::fun(int tmp) : " << tmp << endl; }  
};  
  
class Derived : public Base  
{  
public:  
    virtual void fun(int tmp) { cout << "Derived::fun(int tmp) : " << tmp << endl; } // 重写基类中的 fun 函数  
};  
int main()  
{  
    Base *p = new Derived();  
    p->fun(3); // Derived::fun(int) : 3  
    return 0;  
}

重写和重载的区别：

- 范围区别：对于类中函数的重载或者重写而言，重载发生在同一个类的内部，重写发生在不同的类之间（子类和父类之间）。
    
- 参数区别：重载的函数需要与原函数有相同的函数名、不同的参数列表，不关注函数的返回值类型；重写的函数的函数名、参数列表和返回值类型都需要和原函数相同，父类中被重写的函数需要有 virtual 修饰。
    
- virtual 关键字：重写的函数基类中必须有 virtual关键字的修饰，重载的函数可以有 virtual 关键字的修饰也可以没有。
    

隐藏和重写，重载的区别：

- 范围区别：隐藏与重载范围不同，隐藏发生在不同类中。
    
- 参数区别：隐藏函数和被隐藏函数参数列表可以相同，也可以不同，但函数名一定相同；当参数不同时，无论基类中的函数是否被 virtual修饰，基类函数都是被隐藏，而不是重写。
    

### 

3.3 如何理解 C++ 是面向对象编程

说明：该问题最好结合自己的项目经历进行展开解释，或举一些恰当的例子，同时对比下面向过程编程。

- 面向过程编程：一种以执行程序操作的过程或函数为中心编写软件的方法。程序的数据通常存储在变量中，与这些过程是分开的。所以必须将变量传递给需要使用它们的函数。缺点：随着程序变得越来越复杂，程序数据与运行代码的分离可能会导致问题。例如，程序的规范经常会发生变化，从而需要更改数据的格式或数据结构的设计。当数据结构发生变化时，对数据进行操作的代码也必须更改为接受新的格式。查找需要更改的所有代码会为程序员带来额外的工作，并增加了使代码出现错误的机会。
    
- 面向对象编程（Object-Oriented Programming, OOP）：以创建和使用对象为中心。一个对象（Object）就是一个软件实体，它将数据和程序在一个单元中组合起来。对象的数据项，也称为其属性，存储在成员变量中。对象执行的过程被称为其成员函数。将对象的数据和过程绑定在一起则被称为封装。
    

面向对象编程进一步说明：

面向对象编程将数据成员和成员函数封装到一个类中，并声明数据成员和成员函数的访问级别（public、private、protected），以便控制类对象对数据成员和函数的访问，对数据成员起到一定的保护作用。而且在类的对象调用成员函数时，只需知道成员函数的名、参数列表以及返回值类型即可，无需了解其函数的实现原理。当类内部的数据成员或者成员函数发生改变时，不影响类外部的代码。

### 

3.4 什么是多态？多态如何实现？

多态：多态就是不同继承类的对象，对同一消息做出不同的响应，基类的指针指向或绑定到派生类的对象，使得基类指针呈现不同的表现方式。在基类的函数前加上 virtual 关键字，在派生类中重写该函数，运行时将会根据对象的实际类型来调用相应的函数。如果对象类型是派生类，就调用派生类的函数；如果对象类型是基类，就调用基类的函数。

实现方法：多态是通过虚函数实现的，虚函数的地址保存在虚函数表中，虚函数表的地址保存在含有虚函数的类的实例对象的内存空间中。

实现过程：

1. 在类中用 virtual 关键字声明的函数叫做虚函数；
    
2. 存在虚函数的类都有一个虚函数表，当创建一个该类的对象时，该对象有一个指向虚函数表的虚表指针（虚函数表和类对应的，虚表指针是和对象对应）；
    
3. 当基类指针指向派生类对象，基类指针调用虚函数时，基类指针指向派生类的虚表指针，由于该虚表指针指向派生类虚函数表，通过遍历虚表，寻找相应的虚函数。
    

### 

3.4.1 静态多态与动态多态：

静态多态：也称为编译期间的多态，编译器在编译期间完成的，编译器根据函数实参的类型(可能会进行隐式类型转换)，可推断出要调用那个函数，如果有对应的函数就调用该函数，否则出现编译错误。

动态多态（动态绑定）：即运行时的多态，在程序执行期间(非编译期)判断所引用对象的实际类型，根据其实际类型调用相应的方法。：

● 基类中必须包含虚函数，并且派生类中一定要对基类中的虚函数进行重写。

● 通过基类对象的指针或者引用调用虚函数。

#include <iostream>  
using namespace std;  
  
class Base  
{  
public:  
	virtual void fun() { cout << "Base::fun()" << endl; }  
  
	virtual void fun1() { cout << "Base::fun1()" << endl; }  
  
	virtual void fun2() { cout << "Base::fun2()" << endl; }  
};  
class Derive : public Base  
{  
public:  
	void fun() { cout << "Derive::fun()" << endl; }  
  
	virtual void D_fun1() { cout << "Derive::D_fun1()" << endl; }  
  
	virtual void D_fun2() { cout << "Derive::D_fun2()" << endl; }  
};  
int main()  
{  
	Base *p = new Derive();  
	p->fun(); // Derive::fun() 调用派生类中的虚函数  
	return 0;  
}

简单解释：当基类的指针指向派生类的对象时，通过派生类的对象的虚表指针找到虚函数表（派生类的对象虚函数表），进而找到相应的虚函数 Derive::f() 进行调用。

### 

4. 类相关

### 

4.1 什么是虚函数？什么是纯虚函数？

虚函数：被 virtual 关键字修饰的成员函数，就是虚函数。

纯虚函数：

- 纯虚函数在类中声明时，加上 =0；
    
- 含有纯虚函数的类称为抽象类（只要含有纯虚函数这个类就是抽象类），类中只有接口，没有具体的实现方法；
    
- 继承纯虚函数的派生类，如果没有完全实现基类纯虚函数，依然是抽象类，不能实例化对象。
    

说明：

- 抽象类对象不能作为函数的参数，不能创建对象，不能作为函数返回类型；
    
- 可以声明抽象类指针，可以声明抽象类的引用；
    
- 子类必须继承父类的纯虚函数，并全部实现后，才能创建子类的对象。
    

### 

4.2 虚函数和纯虚函数的区别？

- 虚函数和纯虚函数可以出现在同一个类中，该类称为抽象基类。（含有纯虚函数的类称为抽象基类）
    
- 使用方式不同：虚函数可以直接使用，纯虚函数必须在派生类中实现后才能使用；
    
- 定义形式不同：虚函数在定义时在普通函数的基础上加上 virtual 关键字，纯虚函数定义时除了加上virtual 关键字还需要加上 =0;
    
- 虚函数必须实现，否则编译器会报错；
    
- 对于实现纯虚函数的派生类，该纯虚函数在派生类中被称为虚函数，虚函数和纯虚函数都可以在派生类中重写；
    
- 析构函数最好定义为虚函数，特别是对于含有继承关系的类；析构函数可以定义为纯虚函数，此时，其所在的类为抽象基类，不能创建实例化对象。
    

### 

4.3 虚函数的实现机制

实现机制：虚函数通过虚函数表来实现。虚函数的地址保存在虚函数表中，在类的对象所在的内存空间中，保存了指向虚函数表的指针（称为“虚表指针”），通过虚表指针可以找到类对应的虚函数表。虚函数表解决了基类和派生类的继承问题和类中成员函数的覆盖问题，当用基类的指针来操作一个派生类的时候，这张虚函数表就指明了实际应该调用的函数

虚函数表相关知识点：

- 虚函数表存放的内容：类的虚函数的地址。
    
- 虚函数表建立的时间：编译阶段，即程序的编译过程中会将虚函数的地址放在虚函数表中。
    
- 虚表指针保存的位置：虚表指针存放在对象的内存空间中最前面的位置，这是为了保证正确取到虚函数的偏移量。
    

注：虚函数表和类绑定，虚表指针和对象绑定。即类的不同的对象的虚函数表是一样的，但是每个对象都有自己的虚表指针，来指向类的虚函数表。

实例： 无虚函数覆盖的情况：

#include <iostream>  
using namespace std;  
  
class Base  
{  
public:  
    virtual void B_fun1() { cout << "Base::B_fun1()" << endl; }  
    virtual void B_fun2() { cout << "Base::B_fun2()" << endl; }  
    virtual void B_fun3() { cout << "Base::B_fun3()" << endl; }  
};  
  
class Derive : public Base  
{  
public:  
    virtual void D_fun1() { cout << "Derive::D_fun1()" << endl; }  
    virtual void D_fun2() { cout << "Derive::D_fun2()" << endl; }  
    virtual void D_fun3() { cout << "Derive::D_fun3()" << endl; }  
};  
int main()  
{  
    Base *p = new Derive();  
    p->B_fun1(); // Base::B_fun1()  
    return 0;  
}

主函数中基类的指针 p 指向了派生类的对象，当调用函数 B_fun1() 时，通过派生类的虚函数表找到该函数的地址，从而完成调用。

### 

4.4 单继承和多继承的虚函数表结构

编译器处理虚函数表：

- 编译器将虚函数表的指针放在类的实例对象的内存空间中，该对象调用该类的虚函数时，通过指针找到虚函数表，根据虚函数表中存放的虚函数的地址找到对应的虚函数。
    
- 如果派生类没有重新定义基类的虚函数 A，则派生类的虚函数表中保存的是基类的虚函数 A 的地址，也就是说基类和派生类的虚函数 A 的地址是一样的。
    
- 如果派生类重写了基类的某个虚函数 B，则派生的虚函数表中保存的是重写后的虚函数 B 的地址，也就是说虚函数 B 有两个版本，分别存放在基类和派生类的虚函数表中。
    
- 如果派生类重新定义了新的虚函数 C，派生类的虚函数表保存新的虚函数 C 的地址。
    

### 

4.5 为什么构造函数不能为虚函数？

虚函数的调用需要虚函数表指针，而该指针存放在对象的内存空间中；若构造函数声明为虚函数，那么由于对象还未创建，还没有内存空间，更没有虚函数表地址用来调用虚函数——构造函数了。

### 

4.6 为什么析构函数可以为虚函数，如果不设为虚函数可能会存在什么问题？

防止内存泄露，delete p（基类）的时候，它很机智的先执行了派生类的析构函数，然后执行了基类的析构函数。

如果基类的析构函数不是虚函数，在delete p（基类）时，调用析构函数时，只会看指针的数据类型，而不会去看赋值的对象，这样就会造成内存泄露。

举例说明：

子类B继承自基类A；A *p = new B; delete p;

1） 此时，如果类A的析构函数不是虚函数，那么delete p；将会仅仅调用A的析构函数，只释放了B对象中的A部分，而派生出的新的部分未释放掉。

2） 如果类A的析构函数是虚函数，delete p; 将会先调用B的析构函数，再调用A的析构函数，释放B对象的所有空间。

补充： B *p = new B; delete p;时也是先调用B的析构函数，再调用A的析构函数。

### 

4.7 .不能声明为虚函数的有哪些

1、静态成员函数； 2、类外的普通函数； 3、构造函数； 4、友元函数虚函数是为了实现多态特性的。虚函数的调用只有在程序运行的时候才能知道到底调用的是哪个函数，其是有有如下几点需要注意：

（1） 类的构造函数不能是虚函数

构造函数是为了构造对象的，所以在调用构造函数时候必然知道是哪个对象调用了构造函数，所以构造函数不能为虚函数。

（2） 类的静态成员函数不能是虚函数

类的静态成员函数是该类共用的，与该类的对象无关，静态函数里没有this指针，所以不能为虚函数。

（3）内联函数

内联函数的目的是为了减少函数调用时间。它是把内联函数的函数体在编译器预处理的时候替换到函数调用处，这样代码运行到这里时候就不需要花时间去调用函数。inline是在编译器将函数类容替换到函数调用处，是静态编译的。而虚函数是动态调用的，在编译器并不知道需要调用的是父类还是子类的虚函数，所以不能够inline声明展开，所以编译器会忽略。

（4）友元函数友元函数与该类无关，没有this指针，所以不能为虚函数。

### 

5. 关键字库函数

### 

5.1 sizeof 和 strlen 的区别

1. strlen 是头文件 中的函数，sizeof 是 C++ 中的运算符。
    
2. strlen 测量的是字符串的实际长度（其源代码如下），以 \0 结束。而 sizeof 测量的是字符数组的分配大小。
    

strlen 源代码:  
size_t strlen(const char *str) {  
    size_t length = 0;  
    while (*str++)  
        ++length;  
    return length;  
}

  

#include <iostream>  
#include <cstring>  
  
using namespace std;  
  
int main()  
{  
    char arr[10] = "hello";  
    cout << strlen(arr) << endl; // 5  
    cout << sizeof(arr) << endl; // 10  
    return 0;  
}

3.若字符数组 arr 作为函数的形参，sizeof(arr) 中 arr 被当作字符指针来处理，strlen(arr) 中 arr 依然是字符数组，从下述程序的运行结果中就可以看出。

#include <iostream>  
#include <cstring>  
  
using namespace std;  
  
void size_of(char arr[])  
{  
    cout << sizeof(arr) << endl; // warning: 'sizeof' on array function parameter 'arr' will return size of 'char*' .  
    cout << strlen(arr) << endl;   
}  
  
int main()  
{  
    char arr[20] = "hello";  
    size_of(arr);   
    return 0;  
}  
/*  
输出结果：  
8  
5  
*/

4.strlen 本身是库函数，因此在程序运行过程中，计算长度；而 sizeof 在编译时，计算长度；

5.sizeof 的参数可以是类型，也可以是变量；strlen 的参数必须是 char* 类型的变量。

### 

5.2 lambda 表达式（匿名函数）的具体应用和使用场景

lambda 表达式的定义形式如下：

[capture list] (parameter list) -> reurn type  
{  
   function body  
}

其中：

- capture list：捕获列表，指 lambda 表达式所在函数中定义的局部变量的列表，通常为空，但如果函数体中用到了 lambda 表达式所在函数的局部变量，必须捕获该变量，即将此变量写在捕获列表中。捕获方式分为：引用捕获方式 [&]、值捕获方式 [=]。
    
- return type、parameter list、function body：分别表示返回值类型、参数列表、函数体，和普通函数一样。
    

常见使用场景：排序算法

bool compare(int& a, int& b)  
{  
    return a > b;  
}  
  
int main(void)  
{  
    int data[6] = { 3, 4, 12, 2, 1, 6 };  
    vector<int> testdata;  
    testdata.insert(testdata.begin(), data, data + 6);  
      
    // 排序算法  
    sort(testdata.begin(), testdata.end(), compare);    // 升序  
  
	// 使用lambda表达式  
	sort(testdata.begin(), testdata.end(), [](int a, int b){ return a > b; });  
  
    return 0;  
}

### 

5.3 explicit 的作用（如何避免编译器进行隐式类型转换）

作用：用来声明类构造函数是显示调用的，而非隐式调用，可以阻止调用构造函数时进行隐式转换。只可用于修饰单参构造函数，因为无参构造函数和多参构造函数本身就是显示调用的，再加上 explicit 关键字也没有什么意义。

隐式转换：

#include <iostream>  
#include <cstring>  
using namespace std;  
  
class A  
{  
public:  
    int var;  
    A(int tmp)  
    {  
        var = tmp;  
    }  
};  
int main()  
{  
    A ex = 10; // 发生了隐式转换  
    return 0;  
}

上述代码中，A ex = 10; 在编译时，进行了隐式转换，将 10 转换成 A 类型的对象，然后将该对象赋值给 ex，等同于如下操作：

为了避免隐式转换，可用 explicit 关键字进行声明：

#include <iostream>  
#include <cstring>  
using namespace std;  
  
class A  
{  
public:  
    int var;  
    explicit A(int tmp)  
    {  
        var = tmp;  
        cout << var << endl;  
    }  
};  
int main()  
{  
    A ex(100);  
    A ex1 = 10; // error: conversion from 'int' to non-scalar type 'A' requested  
    return 0;  
}

### 

5.4 C 和 C++ static 的区别

- 在 C 语言中，使用 static 可以定义局部静态变量、外部静态变量、静态函数
    
- 在 C++ 中，使用 static 可以定义局部静态变量、外部静态变量、静态函数、静态成员变量和静态成员函数。因为 C++中有类的概念，静态成员变量、静态成员函数都是与类有关的概念。
    

### 

5.5 static 的作用

作用：

static 定义静态变量，静态函数。

- 保持变量内容持久：static 作用于局部变量，改变了局部变量的生存周期，使得该变量存在于定义后直到程序运行结束的这段时间。
    
- 隐藏：static作用于全局变量和函数，改变了全局变量和函数的作用域，使得全局变量和函数只能在定义它的文件中使用，在源文件中不具有全局可见性。（注：普通全局变量和函数具有全局可见性，即其他的源文件也可以使用。）
    
- static 作用于类的成员变量和类的成员函数，使得类变量或者类成员函数和类有关，也就是说可以不定义类的对象就可以通过类访问这些静态成员。注意：类的静态成员函数中只能访问静态成员变量或者静态成员函数，不能将静态成员函数定义成虚函数。
    

### 

5.6 static 在类中使用的注意事项（定义、初始化和使用）

static 静态成员变量：

1. 静态成员变量是在类内进行声明，在类外进行定义和初始化，在类外进行定义和初始化的时候不要出现 static关键字和private、public、protected 访问规则。
    
2. 静态成员变量相当于类域中的全局变量，被类的所有对象所共享，包括派生类的对象。
    
3. 静态成员变量可以作为成员函数的参数，而普通成员变量不可以。
    

#include <iostream>  
using namespace std;  
  
class A  
{  
public:  
    static int s_var;  
    int var;  
    void fun1(int i = s_var); // 正确，静态成员变量可以作为成员函数的参数  
    void fun2(int i = var);   //  error: invalid use of non-static data member 'A::var'  
};

4.静态数据成员的类型可以是所属类的类型，而普通数据成员的类型只能是该类类型的指针或引用。

#include <iostream>  
using namespace std;  
  
class A  
{  
public:  
    static A s_var; // 正确，静态数据成员  
    A var;          // error: field 'var' has incomplete type 'A'  
    A *p;           // 正确，指针  
    A &var1;        // 正确，引用  
};

static 静态成员函数：

1. 静态成员函数不能调用非静态成员变量或者非静态成员函数，因为静态成员函数没有 this 指针。静态成员函数做为类作用域的全局函数。
    
2. 静态成员函数不能声明成虚函数（virtual）、const 函数和 volatile 函数。
    

### 

5.7 static 全局变量和普通全局变量的异同

相同点：

- 存储方式：普通全局变量和 static 全局变量都是静态存储方式。
    

不同点：

- 作用域：普通全局变量的作用域是整个源程序，当一个源程序由多个源文件组成时，普通全局变量在各个源文件中都是有效的；静态全局变量则限制了其作用域，即只在定义该变量的源文件内有效，在同一源程序的其它源文件中不能使用它。由于静态全局变量的作用域限于一个源文件内，只能为该源文件内的函数公用，因此可以避免在其他源文件中引起错误。
    
- 初始化：静态全局变量只初始化一次，防止在其他文件中使用。
    

### 

5.8 const 作用及用法

作用：

- const 修饰成员变量，定义成 const 常量，相较于宏常量，可进行类型检查，节省内存空间，提高了效率。
    
- const 修饰函数参数，使得传递过来的函数参数的值不能改变。
    
- const 修饰成员函数，使得成员函数不能修改任何类型的成员变量（mutable 修饰的变量除外），也不能调用非 const 成员函数，因为非 const 成员函数可能会修改成员变量。
    

在类中的用法：

const 成员变量：

- const 成员变量只能在类内声明、定义，在构造函数初始化列表中初始化。
    
- const 成员变量只在某个对象的生存周期内是常量，对于整个类而言却是可变的，因为类可以创建多个对象，不同类的 const 成员变量的值是不同的。因此不能在类的声明中初始化 const 成员变量，类的对象还没有创建，编译器不知道他的值。
    

const 成员函数：

- 不能修改成员变量的值，除非有 mutable 修饰；只能访问成员变量。
    
- 不能调用非常量成员函数，以防修改成员变量的值。
    

### 

5.9 define 和 const 的区别

区别：

- 编译阶段：define 是在编译预处理阶段进行替换，const 是在编译阶段确定其值。
    
- 安全性：define 定义的宏常量没有数据类型，只是进行简单的替换，不会进行类型安全的检查；const 定义的常量是有类型的，是要进行判断的，可以避免一些低级的错误。
    
- 内存占用：define 定义的宏常量，在程序中使用多少次就会进行多少次替换，内存中有多个备份，占用的是代码段的空间；const 定义的常量占用静态存储区的空间，程序运行过程中只有一份。
    
- 调试：define 定义的宏常量不能调试，因为在预编译阶段就已经进行替换了；cons定义的常量可以进行调试。
    

const 的优点：

- 有数据类型，在定义式可进行安全性检查。
    
- 可调式。
    
- 占用较少的空间。
    

### 

5.10 define 和 typedef 的区别

- 原理：#define 作为预处理指令，在编译预处理时进行替换操作，不作正确性检查，只有在编译已被展开的源程序时才会发现可能的错误并报错。typedef 是关键字，在编译时处理，有类型检查功能，用来给一个已经存在的类型一个别名，但不能在一个函数定义里面使用 typedef 。
    
- 功能：typedef 用来定义类型的别名，方便使用。#define 不仅可以为类型取别名，还可以定义常量、变量、编译开关等。
    
- 作用域：#define 没有作用域的限制，只要是之前预定义过的宏，在以后的程序中都可以使用，而 typedef 有自己的作用域。
    
- 指针的操作：typedef 和 #define 在处理指针时不完全一样
    

#include <iostream>  
#define INTPTR1 int *  
typedef int * INTPTR2;  
  
using namespace std;  
  
int main()  
{  
    INTPTR1 p1, p2; // p1: int *; p2: int  
    INTPTR2 p3, p4; // p3: int *; p4: int *  
  
    int var = 1;  
    const INTPTR1 p5 = &var; // 相当于 const int * p5; 常量指针，即不可以通过 p5 去修改 p5 指向的内容，但是 p5 可以指向其他内容。  
    const INTPTR2 p6 = &var; // 相当于 int * const p6; 指针常量，不可使 p6 再指向其他内容。  
      
    return 0;  
}

### 

5.11 用宏实现比较大小，以及两个数中的最小值

#include <iostream>  
#define MAX(X, Y) ((X)>(Y)?(X):(Y))  
#define MIN(X, Y) ((X)<(Y)?(X):(Y))  
using namespace std;  
  
int main ()  
{  
    int var1 = 10, var2 = 100;  
    cout << MAX(var1, var2) << endl;  
    cout << MIN(var1, var2) << endl;  
    return 0;  
}  
/*  
程序运行结果：  
100  
10  
*/

### 

5.12 inline 作用及使用方法

作用： inline 是一个关键字，可以用于定义内联函数。内联函数，像普通函数一样被调用，但是在调用时并不通过函数调用的机制而是直接在调用点处展开，这样可以大大减少由函数调用带来的开销，从而提高程序的运行效率。

使用方法：

类内定义成员函数默认是内联函数 在类内定义成员函数，可以不用在函数头部加 inline 关键字，因为编译器会自动将类内定义的函数（构造函数、析构函数、普通成员函数等）声明为内联函数，代码如下：

#include <iostream>  
using namespace std;  
  
class A{  
public:  
    int var;  
    A(int tmp){   
      var = tmp;  
    }      
    void fun(){   
        cout << var << endl;  
    }  
};

类外定义成员函数，若想定义为内联函数，需用关键字声明 当在类内声明函数，在类外定义函数时，如果想将该函数定义为内联函数，则可以在类内声明时不加 inline 关键字，而在类外定义函数时加上 inline 关键字。

#include <iostream>  
using namespace std;  
  
class A{  
public:  
    int var;  
    A(int tmp){   
      var = tmp;  
    }      
    void fun();  
};  
  
inline void A::fun(){  
    cout << var << end

另外，可以在声明函数和定义函数的同时加上 inline；也可以只在函数声明时加 inline，而定义函数时不加 inline。只要确保在调用该函数之前把 inline 的信息告知编译器即可。

### 

5.13 inline 函数工作原理

1. 内联函数不是在调用时发生控制转移关系，而是在编译阶段将函数体嵌入到每一个调用该函数的语句块中，编译器会将程序中出现内联函数的调用表达式用内联函数的函数体来替换。
    
2. 普通函数是将程序执行转移到被调用函数所存放的内存地址，当函数执行完后，返回到执行此函数前的地方。转移操作需要保护现场，被调函数执行完后，再恢复现场，该过程需要较大的资源开销。
    

### 

5.14 宏定义（define）和内联函数（inline）的区别

1. 内联函数是在编译时展开，而宏在编译预处理时展开；在编译的时候，内联函数直接被嵌入到目标代码中去，而宏只是一个简单的文本替换。
    
2. 内联函数是真正的函数，和普通函数调用的方法一样，在调用点处直接展开，避免了函数的参数压栈操作，减少了调用的开销。而宏定义编写较为复杂，常需要增加一些括号来避免歧义。
    
3. 宏定义只进行文本替换，不会对参数的类型、语句能否正常编译等进行检查。而内联函数是真正的函数，会对参数的类型、函数体内的语句编写是否正确等进行检查。
    

#include <iostream>  
#define MAX(a, b) ((a) > (b) ? (a) : (b))  
using namespace std;  
  
inline int fun_max(int a, int b){  
    return a > b ? a : b;  
}  
  
int main(){  
    int var = 1;  
    cout << MAX(var, 5) << endl;       
    cout << fun_max(var, 0) << endl;   
    return 0;  
}  
/*  
程序运行结果：  
5  
1  
*/

### 

5.15 new 的作用？

new 是 C++ 中的关键字，用来动态分配内存空间，实现方式如下：

int *p = new int[5];

### 

5.16 new 和 malloc 如何判断是否申请到内存？

- malloc ：成功申请到内存，返回指向该内存的指针；分配失败，返回 NULL 指针。
    
- new ：内存分配成功，返回该对象类型的指针；分配失败，抛出 bac_alloc 异常。
    

### 

5.17 delete 实现原理？delete 和 delete[] 的区别？

delete 的实现原理：

- 首先执行该对象所属类的析构函数；
    
- 进而通过调用 operator delete 的标准库函数来释放所占的内存空间。
    

delete 和 delete [] 的区别：

- delete 用来释放单个对象所占的空间，只会调用一次析构函数；
    
- delete [] 用来释放数组空间，会对数组中的每个成员都调用一次析构函数。
    

### 

5.18 new 和 malloc 的区别，delete 和 free 的区别

在使用的时候 new、delete 搭配使用，malloc、free 搭配使用。

- malloc、free 是库函数，而new、delete 是关键字。
    
- new 申请空间时，无需指定分配空间的大小，编译器会根据类型自行计算；malloc 在申请空间时，需要确定所申请空间的大小。
    
- new 申请空间时，返回的类型是对象的指针类型，无需强制类型转换，是类型安全的操作符；malloc 申请空间时，返回的是 void* 类型，需要进行强制类型的转换，转换为对象类型的指针。
    
- new 分配失败时，会抛出 bad_alloc 异常，malloc 分配失败时返回空指针。
    
- 对于自定义的类型，new 首先调用 operator new() 函数申请空间（底层通过 malloc 实现），然后调用构造函数进行初始化，最后返回自定义类型的指针；delete 首先调用析构函数，然后调用 operator delete() 释放空间（底层通过 free 实现）。malloc、free 无法进行自定义类型的对象的构造和析构。
    
- new 操作符从自由存储区上为对象动态分配内存，而 malloc 函数从堆上动态分配内存。（自由存储区不等于堆）
    

### 

5.19 malloc 的原理？malloc 的底层实现？

malloc 的原理:

- 当开辟的空间小于 128K 时，调用 brk() 函数，通过移动 _enddata 来实现；
    
- 当开辟空间大于 128K 时，调用 mmap() 函数，通过在虚拟地址空间中开辟一块内存空间来实现。
    

malloc 的底层实现：

brk() 函数实现原理：向高地址的方向移动指向数据段的高地址的指针 _enddata。

mmap 内存映射原理：

1.进程启动映射过程，并在虚拟地址空间中为映射创建虚拟映射区域；

2.调用内核空间的系统调用函数 mmap()，实现文件物理地址和进程虚拟地址的一一映射关系；

3.进程发起对这片映射空间的访问，引发缺页异常，实现文件内容到物理内存（主存）的拷贝。

### 

5.20 C 和 C++ struct 的区别？

- 在 C 语言中 struct 是用户自定义数据类型；在 C++ 中 struct 是抽象数据类型，支持成员函数的定义。
    
- C 语言中 struct 没有访问权限的设置，是一些变量的集合体，不能定义成员函数；C++ 中 struct 可以和类一样，有访问权限，并可以定义成员函数。
    
- C 语言中 struct 定义的自定义数据类型，在定义该类型的变量时，需要加上 struct 关键字，例如：struct A var;，定义 A 类型的变量；而 C++ 中，不用加该关键字，例如：A var;
    

### 

5.21 为什么有了 class 还保留 struct？

- C++ 是在 C 语言的基础上发展起来的，为了与 C 语言兼容，C++ 中保留了 struct。
    

### 

5.22 struct 和 union 的区别

说明：union 是联合体，struct 是结构体。

区别：

- 联合体和结构体都是由若干个数据类型不同的数据成员组成。使用时，联合体只有一个有效的成员；而结构体所有的成员都有效。
    
- 对联合体的不同成员赋值，将会对覆盖其他成员的值，而对于结构体的对不同成员赋值时，相互不影响。
    
- 联合体的大小为其内部所有变量的最大值，按照最大类型的倍数进行分配大小；结构体分配内存的大小遵循内存对齐原则。
    

### 

5.23 class 和 struct 的异同

- struct 和 class 都可以自定义数据类型，也支持继承操作。
    
- struct 中默认的访问级别是 public，默认的继承级别也是 public；class 中默认的访问级别是 private，默认的继承级别也是 private。
    
- 当 class 继承 struct 或者 struct 继承 class 时，默认的继承级别取决于 class 或 struct 本身，class（private 继承），struct（public 继承），即取决于派生类的默认继承级别。
    

struct A{}；  
class B : A{}; // private 继承   
struct C : B{}； // public 继承

- class 可以用于定义模板参数，struct 不能用于定义模板参数。
    

### 

5.24 volatile 的作用？是否具有原子性，对编译器有什么影响？

volatile 的作用：当对象的值可能在程序的控制或检测之外被改变时，应该将该对象声明为 violatile，告知编译器不应对这样的对象进行优化。

volatile不具有原子性。

volatile 对编译器的影响：使用该关键字后，编译器不会对相应的对象进行优化，即不会将变量从内存缓存到寄存器中，防止多个线程有可能使用内存中的变量，有可能使用寄存器中的变量，从而导致程序错误。

### 

5.25 什么情况下一定要用 volatile， 能否和 const 一起使用？

使用 volatile 关键字的场景：

- 当多个线程都会用到某一变量，并且该变量的值有可能发生改变时，需要用 volatile 关键字对该变量进行修饰；
    
- 中断服务程序中访问的变量或并行设备的硬件寄存器的变量，最好用 volatile 关键字修饰。
    

volatile 关键字和 const 关键字可以同时使用，某种类型可以既是 volatile 又是 const ，同时具有二者的属性。

### 

5.26 返回函数中静态变量的地址会发生什么？

#include <iostream>  
using namespace std;  
  
int * fun(int tmp){  
    static int var = 10;  
    var *= tmp;  
    return &var;  
}  
  
int main() {  
    cout << var *  fun(5) << endl;  
    return 0;  
}  
  
/*  
运行结果：  
50  
*/

说明：上述代码中在函数 fun 中定义了静态局部变量 var，使得离开该函数的作用域后，该变量不会销毁，返回到主函数中，该变量依然存在，从而使程序得到正确的运行结果。但是，该静态局部变量直到程序运行结束后才销毁，浪费内存空间。

### 

5.27 extern C 的作用？

extern "C"的主要作用就是为了能够正确实现C++代码调用其他C语言代码。加上extern "C"后，会指示编译器这部分代码按C语言（而不是C++）的方式进行编译。由于C++支持函数重载，因此编译器编译函数的过程中会将函数的参数类型也加到编译后的代码中，而不仅仅是函数名；而C语言并不支持函数重载，因此编译C语言代码的函数时不会带上函数的参数类型，一般只包括函数名。

举例：

// 可能出现在 C++ 头文件<cstring>中的链接指示  
extern "C"{  
    int strcmp(const char*, const char*);  
}

### 

5.28 sizeof(1==1) 在 C 和 C++ 中分别是什么结果？

C 语言代码：4

C++ 代码：1（布尔值大小）

### 

5.29 memcpy 函数的底层原理？

void *memcpy(void *dst, const void *src, size_t size)  
{  
    char *psrc;  
    char *pdst;  
  
    if (NULL == dst || NULL == src)  
    {  
        return NULL;  
    }  
  
    if ((src < dst) && (char *)src + size > (char *)dst) // 出现地址重叠的情况，自后向前拷贝  
    {  
        psrc = (char *)src + size - 1;  
        pdst = (char *)dst + size - 1;  
        while (size--)  
        {  
            *pdst-- = *psrc--;  
        }  
    }  
    else  
    {  
        psrc = (char *)src;  
        pdst = (char *)dst;  
        while (size--)  
        {  
            *pdst++ = *psrc++;  
        }  
    }  
  
    return dst;  
}

### 

5.30 strcpy 函数有什么缺陷？

strcpy 函数的缺陷：strcpy 函数不检查目的缓冲区的大小边界，而是将源字符串逐一的全部赋值给目的字符串地址起始的一块连续的内存空间，同时加上字符串终止符，会导致其他变量被覆盖。

说明：从上述代码中可以看出，变量 var 的后六位被字符串 “hello world!” 的 “d!\0” 这三个字符改变，这三个字符对应的 ascii 码的十六进制为：\0(0x00)，!(0x21)，d(0x64)。

原因：变量 arr 只分配的 10 个内存空间，通过上述程序中的地址可以看出 arr 和 var 在内存中是连续存放的，但是在调用 strcpy 函数进行拷贝时，源字符串 “hello world!” 所占的内存空间为 13，因此在拷贝的过程中会占用 var 的内存空间，导致 var的后六位被覆盖。

**需要更多大厂面试题，扫描下方二维码领取**  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

### 

5.31 auto 类型推导的原理

auto 类型推导的原理：

编译器根据初始值来推算变量的类型，要求用 auto 定义变量时必须有初始值。编译器推断出来的 auto 类型有时和初始值类型并不完全一样，编译器会适当改变结果类型使其更符合初始化规则。

### 

5.32 malloc一次性最大能申请多大内存空间

malloc并不是系统调用。因此并不是内存空间的终极管理者。最大能够申请多大空间，并不是malloc一个人能说了算的。

malloc有多种实现，不同的实现有不同的特点。一般情况下，malloc是从系统获取内存分页，然后将这些分页组织为不同大小的“块”。当用户程序申请内存的时候，如果大小没有超过“块”的大小，则malloc在内部匹配尺寸最为接近的，不小于申请的尺寸的块，记录分配信息之后，返回地址。这样做的目的是加快分配的速度，减少系统调用。

如果申请的空间很大，则malloc会直接向系统申请多个分页，将其map到用户程序的虚拟地址空间当中的一个连续的区间（这个同样是系统调用），然后返回地址。

> 分页的管理，是由操作系统完成的。现代操作系统大多支持虚拟内存，也就是不仅仅使用RAM，还使用磁盘上的交换文件作为内存空间。操作系统将这些空间按照一定大小切分成页，在一个被称为是页表的结构当中进行维护。操作系统级别的内存管理，都是以页为单位的。 页的实际存储位置由操作系统管理。当多个应用程序同时执行时，操作系统会将不活动的页从RAM当中移动到磁盘文件当中。当应用程序重新尝试访问该页的时候，再从磁盘文件读入到RAM当中，这个称为页交换。

也是因为页的这个特性，操作系统完全可以分配实际上不存在的页给malloc，直到该页被真正使用。所以说理论上malloc 1TB可能都没有问题，虽然RAM和磁盘上的交换文件（或者交换分区）加起来往往也就是几十个GB。这就是因为操作系统实际上并没有分配完整的空间给这些页，只是在页表当中记了一笔：A程序需要XXXX页，并留出相应的行，在其中记录“未确保空间”。

当应用程序真的开始尝试对malloc返回的地址进行读写的时候，如果操作系统检测到页其实还没有分配实际的空间，这个时候才会去寻找合适的空间填写进页表。如果找不到合适的空间，那么就会触发页错误（overcommit错误）。

在Linux当中，我们是可以关闭overcommit功能，要求操作系统每次分配分页的时候都确保分页可用，那么malloc可分配的最大大小就受实际RAM+交换文件/分区大小的限制。在Linux当中，为保障操作系统自身的运行，通常这个大小是RAM的一半+交换区的大小。（当然可改变设置）。

另外，单次可分配的大小和多次合计可分配的大小也不是同一个概念，因为多次分配存在内存碎片化的问题。就好像一个饭店虽然有40个座位，但是也许只需要几个人你就没办法找到一张8人桌一样。

所以，在当代操作系统当中，malloc可以分配的空间的最大大小，首先当然受机器的bit数，也就是可以直接寻址的空间大小制约；在这个基础上进一步受操作系统的策略制约；然后受实际可用资源量的制约；再者还受malloc本身算法的制约；最后还受应用程序申请方式和次数的制约。

### 

5.33 public、protected、private的区别

第一: private,public,protected的访问范围：

- private: 只能由该类中的函数、其友元函数访问，不能被任何其他访问，该类的对象也不能访问
    
- protected: 可以被该类中的函数、子类的函数、以及其友元函数访问,但不能被该类的对象访问
    
- public: 可以被该类中的函数、子类的函数、其友元函数访问，也可以由该类的对象访问
    

第二:类的继承后方法属性变化:

- 使用private继承，父类的所有方法在子类中变为private;
    
- 使用protected继承，父类的protected和public方法在子类中变为protected，而private方法不变;
    
- 使用public继承，父类中的方法属性不发生改变;
    

### 

6. 语言特性相关

### 

6.1 左值和右值的区别？左值引用和右值引用的区别，如何将左值转换成右值？

左值：指表达式结束后依然存在的持久对象。

右值：表达式结束就不再存在的临时对象。

左值和右值的区别：左值持久，右值短暂

右值引用和左值引用的区别：

- 左值引用不能绑定到要转换的表达式、字面常量或返回右值的表达式。右值引用恰好相反，可以绑定到这类表达式，但不能绑定到一个左值上。
    
- 右值引用必须绑定到右值的引用，通过 && 获得。右值引用只能绑定到一个将要销毁的对象上，因此可以自由地移动其资源。
    

std::move可以将一个左值强制转化为右值，继而可以通过右值引用使用该值，以用于移动语义。

#include <iostream>  
using namespace std;  
  
void fun1(int& tmp)   
{   
  cout << "fun1(int& tmp):" << tmp << endl;   
}   
  
void fun2(int&& tmp)   
{   
  cout << "fun2(int&& tmp)" << tmp << endl;   
}   
  
int main()   
{   
  int var = 11;   
  fun1(12); // error: cannot bind non-const lvalue reference of type 'int&' to an rvalue of type 'int'  
  fun1(var);  
  fun2(1);   
}

### 

6.2 std::move() 函数的实现原理

std::move() 函数原型：

template <typename T>  
typename remove_reference<T>::type&& move(T&& t)  
{  
	return static_cast<typename remove_reference<T>::type &&>(t);  
}

说明：引用折叠原理

- 右值传递给上述函数的形参 T&& 依然是右值，即 T&& && 相当于 T&&。
    
- 左值传递给上述函数的形参 T&& 依然是左值，即 T&& & 相当于 T&。
    

小结：通过引用折叠原理可以知道，move() 函数的形参既可以是左值也可以是右值。

remove_reference 具体实现：

//原始的，最通用的版本  
template <typename T> struct remove_reference{  
    typedef T type;  //定义 T 的类型别名为 type  
};  
   
//部分版本特例化，将用于左值引用和右值引用  
template <class T> struct remove_reference<T&> //左值引用  
{ typedef T type; }  
   
template <class T> struct remove_reference<T&&> //右值引用  
{ typedef T type; }     
    
//举例如下,下列定义的a、b、c三个变量都是int类型  
int i;  
remove_refrence<decltype(42)>::type a;             //使用原版本，  
remove_refrence<decltype(i)>::type  b;             //左值引用特例版本  
remove_refrence<decltype(std::move(i))>::type  b;  //右值引用特例版本 

举例：

int var = 10;   
  
转化过程：  
1. std::move(var) => std::move(int&& &) => 折叠后 std::move(int&)  
  
2. 此时：T 的类型为 int&，typename remove_reference<T>::type 为 int，这里使用 remove_reference 的左值引用的特例化版本  
  
3. 通过 static_cast 将 int& 强制转换为 int&&  
  
整个std::move被实例化如下  
string&& move(int& t)   
{  
    return static_cast<int&&>(t);   
}

总结： std::move() 实现原理：

- 利用引用折叠原理将右值经过 T&& 传递类型保持不变还是右值，而左值经过 T&&变为普通的左值引用，以保证模板可以传递任意实参，且保持类型不变；
    
- 然后通过 remove_refrence 移除引用，得到具体的类型 T；
    
- 最后通过 static_cast<> 进行强制类型转换，返回 T&& 右值引用。
    

### 

6.3 什么是指针？指针的大小及用法？

指针： 指向另外一种类型的复合类型。指针的大小： 在 64 位计算机中，指针占 8 个字节空间。

#include<iostream>  
  
using namespace std;  
  
int main(){  
    int *p = nullptr;  
    cout << sizeof(p) << endl; // 8  
  
    char *p1 = nullptr;  
    cout << sizeof(p1) << endl; // 8  
    return 0;  
}

指针的用法：

1.指向普通对象的指针

#include <iostream>  
  
using namespace std;  
  
class A  
{  
};  
  
int main()  
{  
    A *p = new A();  
    return 0;  
}

2.指向常量对象的指针：常量指针

#include <iostream>  
using namespace std;  
  
int main(void)  
{  
    const int c_var = 10;  
    const int * p = &c_var;  
    cout << *p << endl;  
    return 0;  
}

3.指向函数的指针：函数指针

#include <iostream>  
using namespace std;  
  
int add(int a, int b){  
    return a + b;  
}  
  
int main(void)  
{  
    int (*fun_p)(int, int);  
    fun_p = add;  
    cout << fun_p(1, 6) << endl;  
    return 0;  
}

4.指向对象成员的指针，包括指向对象成员函数的指针和指向对象成员变量的指针。 特别注意：定义指向成员函数的指针时，要标明指针所属的类。

#include <iostream>  
  
using namespace std;  
  
class A  
{  
public:  
    int var1, var2;   
    int add(){  
        return var1 + var2;  
    }  
};  
  
int main()  
{  
    A ex;  
    ex.var1 = 3;  
    ex.var2 = 4;  
    int *p = &ex.var1; // 指向对象成员变量的指针  
    cout << *p << endl;  
  
    int (A::*fun_p)();  
    fun_p = A::add; // 指向对象成员函数的指针 fun_p  
    cout << (ex.*fun_p)() << endl;  
    return 0;  
}

5.this 指针：指向类的当前对象的指针常量。

#include <iostream>  
#include <cstring>  
using namespace std;  
  
class A  
{  
public:  
    void set_name(string tmp)  
    {  
        this->name = tmp;  
    }  
    void set_age(int tmp)  
    {  
        this->age = age;  
    }  
    void set_sex(int tmp)  
    {  
        this->sex = tmp;  
    }  
    void show()  
    {  
        cout << "Name: " << this->name << endl;  
        cout << "Age: " << this->age << endl;  
        cout << "Sex: " << this->sex << endl;  
    }  
  
private:  
    string name;  
    int age;  
    int sex;  
};  
  
int main()  
{  
    A *p = new A();  
    p->set_name("Alice");  
    p->set_age(16);  
    p->set_sex(1);  
    p->show();  
  
    return 0;  
}

6.什么是野指针和悬空指针？

悬空指针： 若指针指向一块内存空间，当这块内存空间被释放后，该指针依然指向这块内存空间，此时，称该指针为“悬空指针”。

void *p = malloc(size);  
free(p);   
// 此时，p 指向的内存空间已释放， p 就是悬空指针。

野指针：

“野指针”是指不确定其指向的指针，未初始化的指针为“野指针”。

void *p;   
// 此时 p 是“野指针”。

### 

6.5 C++ 11 nullptr 比 NULL 优势

- NULL：预处理变量，是一个宏，它的值是 0，定义在头文件 中，即 #define NULL 0。
    
- nullptr：C++ 11 中的关键字，是一种特殊类型的字面值，可以被转换成任意其他类型。
    

nullptr 的优势：

1. 有类型，类型是 typdef decltype(nullptr) nullptr_t;，使用 nullptr 提高代码的健壮性。
    
2. 函数重载：因为 NULL 本质上是 0，在函数调用过程中，若出现函数重载并且传递的实参是 NULL，可能会出现，不知和哪一个函数匹配的情况；但是传递实参 nullptr 就不会出现这种情况。
    

### 

6.6 指针和引用的区别？

- 指针所指向的内存空间在程序运行过程中可以改变，而引用所绑定的对象一旦绑定就不能改变。（是否可变）
    
- 指针本身在内存中占有内存空间，引用相当于变量的别名，在内存中不占内存空间。（是否占内存）
    
- 指针可以为空，但是引用必须绑定对象。（是否可为空）
    
- 指针可以有多级，但是引用只能一级。（是否能为多级）
    

### 

6.7 常量指针和指针常量的区别

常量指针：

常量指针本质上是个指针，只不过这个指针指向的对象是常量。

特点：const 的位置在指针声明运算符 * 的左侧。只要 const 位于 * 的左侧，无论它在类型名的左边或右边，都表示指向常量的指针。（可以这样理解，* 左侧表示指针指向的对象，该对象为常量，那么该指针为常量指针。）

const int * p;  
int const * p;

注意 1：指针指向的对象不能通过这个指针来修改，也就是说常量指针可以被赋值为变量的地址，之所以叫做常量指针，是限制了通过这个指针修改变量的值。

#include <iostream>  
using namespace std;  
  
int main()  
{  
    const int c_var = 8;  
    const int *p = &c_var;   
    *p = 6;            // error: assignment of read-only location '* p'  
    return 0;  
}

注意 2：虽然常量指针指向的对象不能变化，可是因为常量指针本身是一个变量，因此，可以被重新赋值。

例如：

#include <iostream>  
using namespace std;  
  
int main(){  
    const int c_var1 = 8;  
    const int c_var2 = 8;  
    const int *p = &c_var1;   
    p = &c_var2;  
    return 0;  
}

指针常量：

指针常量的本质上是个常量，只不过这个常量的值是一个指针。

特点：const 位于指针声明操作符右侧，表明该对象本身是一个常量，* 左侧表示该指针指向的类型，即以 * 为分界线，其左侧表示指针指向的类型，右侧表示指针本身的性质。

const int var;  
int * const c_p = &var;

注意 1：指针常量的值是指针，这个值因为是常量，所以指针本身不能改变。

#include <iostream>  
using namespace std;  
  
int main()  
{  
    int var, var1;  
    int * const c_p = &var;  
    c_p = &var1; // error: assignment of read-only variable 'c_p'  
    return 0;  
}

注意 2：指针的内容可以改变。

#include <iostream>  
using namespace std;  
  
int main(){  
    int var = 3;  
    int * const c_p = &var;  
    *c_p = 12;   
    return 0;  
}

### 

6.8 函数指针和指针函数的区别

指针函数： 指针函数本质是一个函数，只不过该函数的返回值是一个指针。相对于普通函数而言，只是返回值是指针。

#include <iostream>  
using namespace std;  
  
struct Type  
{  
  int var1;  
  int var2;  
};  
  
Type * fun(int tmp1, int tmp2){  
    Type * t = new Type();  
    t->var1 = tmp1;  
    t->var2 = tmp2;  
    return t;  
}  
  
int main()  
{  
    Type *p = fun(5, 6);  
    return 0;  
}

函数指针： 函数指针本质是一个指针变量，只不过这个指针指向一个函数。函数指针即指向函数的指针。

举例：

#include <iostream>  
using namespace std;  
int fun1(int tmp1, int tmp2)  
{  
  return tmp1 * tmp2;  
}  
int fun2(int tmp1, int tmp2)  
{  
  return tmp1 / tmp2;  
}  
  
int main()  
{  
  int (*fun)(int x, int y);   
  fun = fun1;  
  cout << fun(15, 5) << endl;   
  fun = fun2;  
  cout << fun(15, 5) << endl;   
  return 0;  
}  
/*  
运行结果：  
75  
3  
*/

函数指针和指针函数的区别：

本质不同

1.指针函数本质是一个函数，其返回值为指针。

2.函数指针本质是一个指针变量，其指向一个函数。

定义形式不同

1.指针函数：int* fun(int tmp1, int tmp2); ，这里* 表示函数的返回值类型是指针类型。

2.函数指针：int (fun)(int tmp1, int tmp2);，这里 表示变量本身是指针类型。

用法不同

### 

6.9 强制类型转换有哪几种？

- static_cast：用于数据的强制类型转换，强制将一种数据类型转换为另一种数据类型。
    

1.用于基本数据类型的转换。

2.用于类层次之间的基类和派生类之间 指针或者引用 的转换（不要求必须包含虚函数，但必须是有相互联系的类），进行上行转换（派生类的指针或引用转换成基类表示）是安全的；进行下行转换（基类的指针或引用转换成派生类表示）由于没有动态类型检查，所以是不安全的，最好用 dynamic_cast 进行下行转换。

3.可以将空指针转化成目标类型的空指针。

4.可以将任何类型的表达式转化成 void 类型。

- const_cast：强制去掉常量属性，不能用于去掉变量的常量性，只能用于去除指针或引用的常量性，将常量指针转化为非常量指针或者将常量引用转化为非常量引用（注意：表达式的类型和要转化的类型是相同的）。
    
- reinterpret_cast：改变指针或引用的类型、将指针或引用转换为一个足够长度的整型、将整型转化为指针或引用类型。
    
- dynamic_cast：
    

1.其他三种都是编译时完成的，动态类型转换是在程序运行时处理的，运行时会进行类型检查。

2.只能用于带有虚函数的基类或派生类的指针或者引用对象的转换，转换成功返回指向类型的指针或引用，转换失败返回 NULL；不能用于基本数据类型的转换。

3.在向上进行转换时，即派生类类的指针转换成基类类的指针和 static_cast 效果是一样的，（注意：这里只是改变了指针的类型，指针指向的对象的类型并未发生改变）。

4.在下行转换时，基类的指针类型转化为派生类类的指针类型，只有当要转换的指针指向的对象类型和转化以后的对象类型相同时，才会转化成功。

#include <iostream>  
#include <cstring>  
  
using namespace std;  
  
class Base  
{  
public:  
    virtual void fun()  
    {  
        cout << "Base::fun()" << endl;  
    }  
};  
  
class Derive : public Base  
{  
public:  
    virtual void fun()  
    {  
        cout << "Derive::fun()" << endl;  
    }  
};  
  
int main()  
{  
    Base *p1 = new Derive();  
    Base *p2 = new Base();  
    Derive *p3 = new Derive();  
  
    //转换成功  
    p3 = dynamic_cast<Derive *>(p1);  
    if (p3 == NULL)  
    {  
        cout << "NULL" << endl;  
    }  
    else  
    {  
        cout << "NOT NULL" << endl; // 输出  
    }  
  
    //转换失败  
    p3 = dynamic_cast<Derive *>(p2);  
    if (p3 == NULL)  
    {  
        cout << "NULL" << endl; // 输出  
    }  
    else  
    {  
        cout << "NOT NULL" << endl;  
    }  
  
    return 0;  
}

### 

6.10 如何判断结构体是否相等？能否用 memcmp 函数判断结构体相等？

需要重载操作符 == 判断两个结构体是否相等，不能用函数 memcmp 来判断两个结构体是否相等，因为 memcmp 函数是逐个字节进行比较的，而结构体存在内存空间中保存时存在字节对齐，字节对齐时补的字节内容是随机的，会产生垃圾值，所以无法比较。

利用运算符重载来实现结构体对象的比较：

#include <iostream>  
  
using namespace std;  
  
struct A  
{  
    char c;  
    int val;  
    A(char c_tmp, int tmp) : c(c_tmp), val(tmp) {}  
  
    friend bool operator==(const A &tmp1, const A &tmp2); //  友元运算符重载函数  
};  
  
bool operator==(const A &tmp1, const A &tmp2)  
{  
    return (tmp1.c == tmp2.c && tmp1.val == tmp2.val);  
}  
  
int main()  
{  
    A ex1('a', 90), ex2('b', 80);  
    if (ex1 == ex2)  
        cout << "ex1 == ex2" << endl;  
    else  
        cout << "ex1 != ex2" << endl; // 输出  
    return 0;  
}

### 

6.11 参数传递时，值传递、引用传递、指针传递的区别？

参数传递的三种方式：

- 值传递：形参是实参的拷贝，函数对形参的所有操作不会影响实参。
    
- 指针传递：本质上是值传递，只不过拷贝的是指针的值，拷贝之后，实参和形参是不同的指针，通过指针可以间接的访问指针所指向的对象，从而可以修改它所指对象的值。
    
- 引用传递：当形参是引用类型时，我们说它对应的实参被引用传递。
    

### 

6.12 什么是模板？如何实现？

模板：创建类或者函数的蓝图或者公式，分为函数模板和类模板。实现方式：模板定义以关键字 template 开始，后跟一个模板参数列表。

- 模板参数列表不能为空；
    
- 模板类型参数前必须使用关键字 class 或者 typename，在模板参数列表中这两个关键字含义相同，可互换使用。
    

template <typename T, typename U, ...>

函数模板：通过定义一个函数模板，可以避免为每一种类型定义一个新函数。

- 对于函数模板而言，模板类型参数可以用来指定返回类型或函数的参数类型，以及在函数体内用于变量声明或类型转换。
    
- 函数模板实例化：当调用一个模板时，编译器用函数实参来推断模板实参，从而使用实参的类型来确定绑定到模板参数的类型
    

#include<iostream>  
using namespace std;  
  
template <typename T>  
T add_fun(const T & tmp1, const T & tmp2){  
    return tmp1 + tmp2;  
}  
  
int main(){  
    int var1, var2;  
    cin >> var1 >> var2;  
    cout << add_fun(var1, var2);  
  
    double var3, var4;  
    cin >> var3 >> var4;  
    cout << add_fun(var3, var4);  
    return 0;  
}

类模板：类似函数模板，类模板以关键字 template 开始，后跟模板参数列表。但是，编译器不能为类模板推断模板参数类型，需要在使用该类模板时，在模板名后面的尖括号中指明类型。

#include <iostream>  
using namespace std;  
  
template <typename T>  
class Complex{  
public:  
    //构造函数  
    Complex(T a, T b)  
    {  
        this->a = a;  
        this->b = b;  
    }  
  
    //运算符重载  
    Complex<T> operator+(Complex &c){  
        Complex<T> tmp(this->a + c.a, this->b + c.b);  
        cout << tmp.a << " " << tmp.b << endl;  
        return tmp;  
    }  
  
private:  
    T a;  
    T b;  
};  
  
int main(){  
    Complex<int> a(10, 20);  
    Complex<int> b(20, 30);  
    Complex<int> c = a + b;  
  
    return 0;  
}

### 

6.13 函数模板和类模板的区别？

- 实例化方式不同：函数模板实例化由编译程序在处理函数调用时自动完成，类模板实例化需要在程序中显式指定。
    
- 实例化的结果不同：函数模板实例化后是一个函数，类模板实例化后是一个类。
    
- 默认参数：类模板在模板参数列表中可以有默认参数。
    
- 特化：函数模板只能全特化；而类模板可以全特化，也可以偏特化。
    
- 调用方式不同：函数模板可以隐式调用，也可以显式调用；类模板只能显式调用。 函数模板调用方式举例：
    

#include<iostream>  
using namespace std;  
  
template <typename T>  
T add_fun(const T & tmp1, const T & tmp2){  
    return tmp1 + tmp2;  
}  
  
int main(){  
    int var1, var2;  
    cin >> var1 >> var2;  
    cout << add_fun<int>(var1, var2); // 显式调用  
  
    double var3, var4;  
    cin >> var3 >> var4;  
    cout << add_fun(var3, var4); // 隐式调用  
    return 0;  
}

### 

6.14 什么是可变参数模板？

可变参数模板：接受可变数目参数的模板函数或模板类。将可变数目的参数被称为参数包，包括模板参数包和函数参数包。

- 模板参数包：表示零个或多个模板参数；
    
- 函数参数包：表示零个或多个函数参数。
    

用省略号来指出一个模板参数或函数参数表示一个包，在模板参数列表中，class… 或 typename… 指出接下来的参数表示零个或多个类型的列表；一个类型名后面跟一个省略号表示零个或多个给定类型的非类型参数的列表。当需要知道包中有多少元素时，可以使用 sizeof… 运算符。

template <typename T, typename... Args> // Args 是模板参数包  
void foo(const T &t, const Args&... rest); // 可变参数模板，rest 是函数参数包

  

#include <iostream>  
using namespace std;  
  
template <typename T>  
void print_fun(const T &t){  
    cout << t << endl; // 最后一个元素  
}  
  
template <typename T, typename... Args>  
void print_fun(const T &t, const Args &...args){  
    cout << t << " ";  
    print_fun(args...);  
}  
  
int main(){  
    print_fun("Hello", "wolrd", "!");  
    return 0;  
}  
/*运行结果：  
Hello wolrd !  
*/

说明：可变参数函数通常是递归的，第一个版本的 print_fun 负责终止递归并打印初始调用中的最后一个实参。第二个版本的 print_fun 是可变参数版本，打印绑定到 t 的实参，并用来调用自身来打印函数参数包中的剩余值。

### 

6.15 什么是模板特化？为什么特化？

模板特化的原因：模板并非对任何模板实参都合适、都能实例化，某些情况下，通用模板的定义对特定类型不合适，可能会编译失败，或者得不到正确的结果。因此，当不希望使用模板版本时，可以定义类或者函数模板的一个特例化版

模板特化：模板参数在某种特定类型下的具体实现。分为函数模板特化和类模板特化

- 函数模板特化：将函数模板中的全部类型进行特例化，称为函数模板特化。
    
- 类模板特化：将类模板中的部分或全部类型进行特例化，称为类模板特化。
    

特化分为全特化和偏特化：

- 全特化：模板中的模板参数全部特例化。
    
- 偏特化：模板中的模板参数只确定了一部分，剩余部分需要在编译器编译时确定。
    

说明：要区分下函数重载与函数模板特化

定义函数模板的特化版本，本质上是接管了编译器的工作，为原函数模板定义了一个特殊实例，而不是函数重载，函数模板特化并不影响函数匹配。

#include <iostream>  
#include <cstring>  
using namespace std;  
//函数模板  
template <class T>  
bool compare(T t1, T t2){  
    cout << "通用版本：";  
    return t1 == t2;  
}  
  
template <> //函数模板特化  
bool compare(char *t1, char *t2){  
    cout << "特化版本：";  
    return strcmp(t1, t2) == 0;  
}在LINUX中我们可以使用mmap用来在进程虚拟内存地址空间中分配地址空间，创建和物理内存的映射关系。  
  
int main(int argc, char *argv[]){  
    char arr1[] = "hello";  
    char arr2[] = "abc";  
    cout << compare(123, 123) << endl;  
    cout << compare(arr1, arr2) << endl;  
  
    return 0;  
}  
/*  
运行结果：  
通用版本：1  
特化版本：0  
*/

### 

6.16 include " " 和 <> 的区别

include<文件名> 和 #include"文件名" 的区别:

- 查找文件的位置：include<文件名>在标准库头文件所在的目录中查找，如果没有，再到当前源文件所在目录下查找；#include"文件名" 在当前源文件所在目录中进行查找，如果没有；再到系统目录中查找。
    
- 使用习惯：对于标准库中的头文件常用 include<文件名>，对于自己定义的头文件，常用 #include"文件名"
    

### 

6.17 泛型编程如何实现？

泛型编程实现的基础：模板。模板是创建类或者函数的蓝图或者说公式，当时用一个 vector 这样的泛型，或者 find 这样的泛型函数时，编译时会转化为特定的类或者函数。

泛型编程涉及到的知识点较广，例如：容器、迭代器、算法等都是泛型编程的实现实例。面试者可选择自己掌握比较扎实的一方面进行展开。

- 容器：涉及到 STL 中的容器，例如：vector、list、map 等，可选其中熟悉底层原理的容器进行展开讲解。
    
- 迭代器：在无需知道容器底层原理的情况下，遍历容器中的元素。
    
- 模板：可参考本章节中的模板相关问题。
    

### 

6.18 C++命名空间

使用命名空间的目的是对标识符的名称进行本地化，以避免命名冲突。在C++中，变量、函数和类都是大量存在的。如果没有命名空间，这些变量、函数、类的名称将都存在于全局命名空间中，会导致很多冲突。比如，如果我们在自己的程序中定义了一个函functionA()，这将重写标准库中的functionA()函 数，这是因为这两个函数都是位于全局命名空间中的。

### 

6.19 C++ STL六大组件

为了建立数据结构和算法的一套标准，并且降低他们之间的耦合关系，以提升各自的独立性、弹性、交互操作性(相互合作性,interoperability),诞生了STL。

STL提供了六大组件，彼此之间可以组合套用，这六大组件分别是:容器、算法、迭代器、仿函数、适配器（配接器）、空间配置器。

容器：各种数据结构，如vector、list、deque、set、map等,用来存放数据，从实现角度来看，STL容器是一种class template。

算法：各种常用的算法，如sort、find、copy、for_each。从实现的角度来看，STL算法是一种function tempalte.

迭代器：扮演了容器与算法之间的胶合剂，共有五种类型，从实现角度来看，迭代器是一种将operator* , operator-> , operator++,operator–等指针相关操作予以重载的class template. 所有STL容器都附带有自己专属的迭代器，只有容器的设计者才知道如何遍历自己的元素。原生指针(native pointer)也是一种迭代器。

仿函数：行为类似函数，可作为算法的某种策略。从实现角度来看，仿函数是一种重载了operator()的class 或者class template

适配器：一种用来修饰容器或者仿函数或迭代器接口的东西。

空间配置器：负责空间的配置与管理。从实现角度看，配置器是一个实现了动态空间配置、空间管理、空间释放的class tempalte.

STL六大组件的交互关系，容器通过空间配置器取得数据存储空间，算法通过迭代器存储容器中的内容，仿函数可以协助算法完成不同的策略的变化，适配器可以修饰仿函数。

### 

7 git 分布式版本控制系统

### 

7.1 简单说一下大端、小端。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

换句话说👇：存在字符串： 高有效位 → 12 34 56 78 → 低有效位<br/>小端： 低地址位 → 7 8 56 34 12 → 高地址位<br/>大 端： 低地址位 → 12 34 56 78 → 高地址位

助 记 ：

大端模式：符合阅读习惯， 高字节存放在低地址，低字节存放在高地址；类似于把数据当作字符串顺序处理小端模式：低字节存放在低地址，高字节存放在高地址；即高地址部分权值高，低地址部分权值低

### 

7.2 什么是git？

Git 是一个开源的分布式版本控制系统，用于快速高效地处理任何或小或大的项目。最初是为了帮助管理Linux内核开发的一个开放源码的版本控制软件。

Git应用十分广泛，小到我们平时使用github网站，大到公司中多人合作的大型项目开发。 它速度快，完全分布式，允许成千上万个并行开发的分支，容灾性能强。

- git分为哪几个区？分别是什么？
    

①工作区(Workspace)是电脑中实际的目录。

②暂存区(Index)类似于缓存区域，临时保存你的改动。

③仓库区(Repository)，分为本地仓库和远程仓库。

- 通常提交代码分为几步：
    

① git add 从工作区提交到暂存区 ② git commit 从暂存区提交到本地仓库③ git push 或 git svn dcommit 从本地仓库提交到远程仓库

### 

7.3 为什么要用git？在LINUX中我们可以使用mmap用来在进程虚拟内存地址空间中分配地址空间，创建和物理内存的映射关系。

让你回答为什么用git，其实就是让你说之前的版本控制系统为什么不好。 最开始的版本控制方法一般都是采用 人工手动控制。

- 缺点：
    

千人千面，不同版本命名随意

有时无法快速辨别版本新旧

无法快速知道版本1相比版本2自己改了什么

如果要是多人开发，那命名和版本控制更为可怕

### 

7.4 简述集中式版本控制库和分布式版本控制库的区别

备份

- 集中式存在单点故障，一旦故障，没法恢复，所以备份极其重要；
    
- 分布式则更安全，出现故障可以恢复数据，每一个节点都是一个服务器。
    

服务器压力

- 集中式所有的操作都要与服务器交互，操作首先于带宽，不能移动办公；
    
- 分布式全是离线操作，不受限于带宽，可以移动办公。
    

安全性

- 集中式容易出现单点故障，遭受黑客攻击；
    
- 分布式数据、提交全部使用SHA1哈希，以保证数据完整性，甚至提交可以使用PGP签名。
    

工作模式

- 集中式合适人数不多的项目，集中管理；
    
- 分布式适合很多人的项目。
    

### 

8. 补充内容

### 

8.1 mmap基本原理和分类

在LINUX中我们可以使用mmap用来在进程虚拟内存地址空间中分配地址空间，创建和物理内存的映射关系。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

映射关系可以分为两种

1、文件映射

磁盘文件映射进程的虚拟地址空间，使用文件内容初始化物理内存。

2、匿名映射

初始化全为0的内存空间。

而对于映射关系是否共享又分为

1、私有映射(MAP_PRIVATE)

多进程间数据共享，修改不反应到磁盘实际文件，是一个copy-on-write（写时复制）的映射方式。

2、共享映射(MAP_SHARED)

多进程间数据共享，修改反应到磁盘实际文件中。

因此总结起来有4种组合

1、私有文件映射

多个进程使用同样的物理内存页进行初始化，但是各个进程对内存文件的修改不会共享，也不会反应到物理文件中

2、私有匿名映射

mmap会创建一个新的映射，各个进程不共享，这种使用主要用于分配内存(malloc分配大内存会调用mmap)。

例如开辟新进程时，会为每个进程分配虚拟的地址空间，这些虚拟地址映射的物理内存空间各个进程间读的时候共享，写的时候会copy-on-write。

3、共享文件映射

多个进程通过虚拟内存技术共享同样的物理内存空间，对内存文件 的修改会反应到实际物理文件中，他也是进程间通信(IPC)的一种机制。

4、共享匿名映射

这种机制在进行fork的时候不会采用写时复制，父子进程完全共享同样的物理内存页，这也就实现了父子进程通信(IPC).

这里值得注意的是，mmap只是在虚拟内存分配了地址空间，只有在第一次访问虚拟内存的时候才分配物理内存。

在mmap之后，并没有在将文件内容加载到物理页上，只上在虚拟内存中分配了地址空间。当进程在访问这段地址时，通过查找页表，发现虚拟内存对应的页没有在物理内存中缓存，则产生"缺页"，由内核的缺页异常处理程序处理，将文件对应内容，以页为单位(4096)加载到物理内存，注意是只加载缺页，但也会受操作系统一些调度策略影响，加载的比所需的多。

### 

8.2 RAII机制介绍

RAII全程为Resource Acquisition Is Initialization（资源获取即初始化），RAII是C++语法体系中的一种常用的合理管理资源避免出现内存泄漏的常用方法。以对象管理资源，利用的就是C++构造的对象最终会被对象的析构函数销毁的原则。RAII的做法是使用一个对象，在其构造时获取对应的资源，在对象生命期内控制对资源的访问，使之始终保持有效，最后在对象析构的时候，释放构造时获取的资源。

### 

8.3 使用RAII机制的原因

RAII是合理管理资源避免出现内存泄漏的常用方法。那么所谓的资源是如何进行定义的呢？在计算机系统中，资源是数量有限且对系统正常运行具有一定作用的元素。比如：网络套接字、互斥锁、文件句柄和内存等等，它们属于系统资源。由于系统的资源是有限的，就好比自然界的石油，铁矿一样，不是取之不尽，用之不竭的，所以，我们在编程使用系统资源时，都必须遵循一个步骤：

- 申请资源→使用资源→释放资源
    

如果在申请和使用资源后未进行资源的释放，此时就造成了资源的泄漏。资源使用后必须要将其释放。 简单举例：

#include <iostream>   
using namespace std;   
  
int main()   
{   
    int *arr = new int [10];   
    // Here, you can use the array   
    delete [] arr;   
    arr = nullptr ;   
    return 0;   
}

上述的的申请、使用、释放资源的程序较为简单，但是如果程序很复杂的时候，需要为所有的new 分配的内存delete掉，导致极度臃肿，效率下降，更可怕的是，程序的可理解性和可维护性明显降低了，当操作增多时，处理资源释放的代码就会越来越多，越来越乱。如果某一个操作发生了异常而导致释放资源的语句没有被调用，怎么办？这个时候，RAII机制就可以派上用场了。

使用RAII机制的优点

- 不需要显式地释放资源。
    
- 采用这种方式，对象所需的资源只在其生命期内始终保持有效。
    

### 

8.4 RAII机制的使用方法

由于系统的资源不具有自动释放的功能，而C++中的类具有自动调用析构函数的功能。如果把资源用类进行封装起来，对资源操作都封装在类的内部，在析构函数中进行释放资源。当定义的局部变量的生命结束时，它的析构函数就会自动的被调用，如此，就不用程序员显示的去调用释放资源的操作了。

使用RAII机制的代码示例

#include <iostream>  
#include <windows.h>  
#include <process.h>  
  
using namespace std;  
  
CRITICAL_SECTION cs;  
int gGlobal = 0;  
  
class MyLock  
{  
public:  
    MyLock()  
    {  
        EnterCriticalSection(&cs);  
    }  
  
    ~MyLock()  
    {  
        LeaveCriticalSection(&cs);  
    }  
  
private:  
    MyLock( const MyLock &);  
    MyLock operator =(const MyLock &);  
};  
  
void DoComplex(MyLock &lock )  
{  
}  
  
unsigned int __stdcall ThreadFun(PVOID pv)   
{  
    MyLock lock;  
    int *para = (int *) pv;  
  
    // I need the lock to do some complex thing  
    DoComplex(lock);  
  
    for (int i = 0; i < 10; ++i)  
    {  
        ++gGlobal;  
        cout<< "Thread " <<*para<<endl;  
        cout<<gGlobal<<endl;  
    }  
    return 0;  
}  
  
int main()  
{  
    InitializeCriticalSection(&cs);  
  
    int thread1, thread2;  
    thread1 = 1;  
    thread2 = 2;  
  
    HANDLE handle[2];  
    handle[0] = ( HANDLE )_beginthreadex(NULL , 0, ThreadFun, ( void *)&thread1, 0, NULL );  
    handle[1] = ( HANDLE )_beginthreadex(NULL , 0, ThreadFun, ( void *)&thread2, 0, NULL );  
    WaitForMultipleObjects(2, handle, TRUE , INFINITE );  
    return 0;  
}

这个例子可以说是实际项目的一个模型，当多个进程访问临界变量时，为了不出现错误的情况，需要对临界变量进行加锁；上面的例子就是使用的Windows的临界区域实现的加锁。但是，在使用CRITICAL_SECTION时，EnterCriticalSection和LeaveCriticalSection必须成对使用，很多时候，经常会忘了调用LeaveCriticalSection，此时就会发生死锁的现象。当我将对CRITICAL_SECTION的访问封装到MyLock类中时，之后，我只需要定义一个MyLock变量，而不必手动的去显示调用LeaveCriticalSection函数。

RAII总结

- RAII机制保证了异常安全，并且也为程序员在编写动态分配内存的程序时提供了安全保证。
    
- 但它也存在一些缺点，有些操作可能会抛出异常，如果放在析构函数中进行则不能将错误传递出去，那么此时析构函数就必须自己处理异常。这在某些时候是很繁琐的。
    

  

阅读 1459

​

写留言

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/8pECVbqIO0wM26DrqMOOCH0fLxcaCMw0RTe7vAiaUYITgvtM3gtFu2SJUUTX3CN62BiaQfC3BjWVlDwwKqiaCpNIg/300?wx_fmt=png&wxfrom=18)

Linux开发架构之路

91739

写留言

写留言

**留言**

暂无留言