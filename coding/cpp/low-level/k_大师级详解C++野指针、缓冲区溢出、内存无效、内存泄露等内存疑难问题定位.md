Original QtCpp大师 QtCpp大师
 _2024年09月01日 08:00_ _广东_

上一篇文章讲述了ASan原理，即使用**影子内存和红区**技术来提供准确和即时的内存错误检测。

这篇文章用**示例代码**详细讲述如下内存问题定位以及输出分析

- •释放后使用(空悬指针)
- •堆缓冲区溢出
- •栈缓冲区溢出
- •全局缓冲区溢出
- •在返回后使用栈内存
- •限定作用域后使用

![[Pasted image 20240909080003.png]]
# 释放后使用(空悬指针)

内存被释放后，该内存还继续被使用，导致访问异常。
```cpp
int main(int argc, char** argv){    int* array = new int[100];    delete[] array;    return array[argc];  // 异常}
```

上述代码肯定会异常，我们很容易发现，因为这是一个演示模型。
但是在大型项目中，涉及到系统，子系统，模块等等，这样的问题就很难从代码层面发现了。
那么使用ASan运行后，因为有影子内存以及红区检测技术，问题很快会被发现而且定位到，详细分析如下
![[Pasted image 20240909080014.png]]

1. 1.第一段表示heap-use-after-free类型的内存错误，在main.cpp的第六行，即`return array[argc];`语句访问了0x03603a44地址。
2. 2.第二段表示0x03603a44这个地址被哪个线程哪个文件哪行代码释放掉了，在main.cpp的第五行。
3. 3.第三段表示0x03603a44这个地址被哪个线程哪个文件哪行代码申请的，在main.cpp的第四行。
    
即不仅告知了错误在哪里，而且也告知了为什么会产生这种错误，问题就迎刃而解了，精准高效。

# 堆缓冲区溢出

内存访问发生在堆分配区域的边界之外。

```cpp
int main(int argc, char** argv){    int* array =new int[100];    array[0]=0;    int res = array[argc +100];// 异常？    delete[] array;    printf("run here");    getchar();    return res;}
```
![[Pasted image 20240909080024.png]]

1. 1.第一段表示heap-buffer-overflow类型的内存错误，在main.cpp的第七行，即`int res = array[argc + 100];`语句越界访问0x02f03bd4地址。
2. 2.第二段表示0x02f03bd4这个地址位于400字节区域右侧4个字节处，在main.cpp的第五行。
    
因此根据输出报告以及第五行、第七行代码，可迅速定位到问题并进行解决。
# 栈缓冲区溢出
内存访问发生在栈分配区域的边界之外。

```cpp
int main(int argc, char** argv){    int stack_array[100];    stack_array[1]=0;    int res = stack_array[argc +100]; // 异常？						
printf("run here");    getchar();    return res;}
```
![[Pasted image 20240909080031.png]]

1. 1.第一段表示stack-buffer-overflow类型的内存错误，在main.cpp的第七行，即`int res = stack_array[argc + 100];`语句越界访问0x0026fc2c地址。
2. 2.第二段表示0x0026fc2c地址位于线程帧中偏移量420处的栈中，在main.cpp的第四行。
    
# 全局缓冲区溢出

内存访问发生在全局分配区域的边界之外。

```cpp
int global_array[100]={-1};int main(int argc, char** argv){    int res = global_array[argc +100];// 异常？
printf("run here");    getchar();    return res;}
```

**上述代码启动运行是不会出现异常(例如程序异常退出)，但是是非常危险的代码**。

因为其隐晦的访问了非预期内存中的四个字节，非常难定位。
那么使用ASan运行后，因为有影子内存以及红区检测技术，问题很快会被发现而且定位到，详细分析如下
![[Pasted image 20240909080042.png]]

1. 1.第一段表示是global-buffer-overflow类型的内存错误，由main.cpp的第六行触发，即`int res = global_array[argc + 100];`语句访问了0x0083b254地址。
2. 2.第二段表示0x0083b254这个地址在global_array的右边，即溢出了，同时指出了global_array是在main.cpp的第三行定义的。
# 在返回后使用栈内存

在函数内把函数变量栈内存地址返回给外部使用。

```cpp
int* ptr =nullptr;void Fun(){    int local[100];    ptr =&local[0];}int main(int argc, char** argv){    Fun();    int value =*ptr;    printf("value = %d\n", value);    printf("run here");    getchar();    return value;}
```
![[Pasted image 20240909080058.png]]

1. 1.第一段表示stack-use-after-return类型的内存错误，在main.cpp的第14行，即`int value = *ptr;`语句访问0x01809000地址。
2. 2.第二段表示0x01809000是一个帧栈变量，在main.cpp的第6行，即`void Fun()`函数。
    
# 限定作用域后使用

在栈变量生命周期的范围之外对该变量地址进行访问。

```cpp
int* p =nullptr;int main(){    {        int x =0;        p =&x;    }    *p =5;    printf("run here");    getchar();    return0;}
```
![[Pasted image 20240909080106.png]]

1. 1.第一段表示stack-use-after-scope类型的内存错误，在main.cpp的第11行，即`*p = 5;`语句访问0x0055fc58地址。
2. 2.第二段表示0x0055fc58地址是一个帧栈地址，在main.cpp的第6行。
    
