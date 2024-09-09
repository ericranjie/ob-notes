# 

Original QtCpp大师 QtCpp大师

 _2024年09月01日 08:00_ _广东_

![Image](https://mmbiz.qpic.cn/mmbiz_png/7EROo10OA1r3I0Wm4bkwnYqPYfWEX8bgABGtd6JuWJx02Gks0MKO9nQJZYrCicCiahOBicPJRjSxrZ5aQT3VoCtyA/640?wx_fmt=png&from=appmsg&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

上一篇文章讲述了ASan原理，即使用**影子内存和红区**技术来提供准确和即时的内存错误检测。

这篇文章用**示例代码**详细讲述如下内存问题定位以及输出分析

- •释放后使用(空悬指针)
    
- •堆缓冲区溢出
    
- •栈缓冲区溢出
    
- •全局缓冲区溢出
    
- •在返回后使用栈内存
    
- •限定作用域后使用
    
-   
    
![[Pasted image 20240909080003.png]]
![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

# 释放后使用(空悬指针)

内存被释放后，该内存还继续被使用，导致访问异常。

```cpp
int main(int argc, char** argv){    int* array = new int[100];    delete[] array;    return array[argc];  // 异常}
```

上述代码肯定会异常，我们很容易发现，因为这是一个演示模型。

但是在大型项目中，涉及到系统，子系统，模块等等，这样的问题就很难从代码层面发现了。

那么使用ASan运行后，因为有影子内存以及红区检测技术，问题很快会被发现而且定位到，详细分析如下
![[Pasted image 20240909080014.png]]
![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

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
![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

1. 1.第一段表示heap-buffer-overflow类型的内存错误，在main.cpp的第七行，即`int res = array[argc + 100];`语句越界访问0x02f03bd4地址。
    
2. 2.第二段表示0x02f03bd4这个地址位于400字节区域右侧4个字节处，在main.cpp的第五行。
    

因此根据输出报告以及第五行、第七行代码，可迅速定位到问题并进行解决。

# 栈缓冲区溢出

内存访问发生在栈分配区域的边界之外。

```cpp
int main(int argc, char** argv){    int stack_array[100];    stack_array[1]=0;    int res = stack_array[argc +100];// 异常？    printf("run here");    getchar();    return res;}
```
![[Pasted image 20240909080031.png]]
![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

1. 1.第一段表示stack-buffer-overflow类型的内存错误，在main.cpp的第七行，即`int res = stack_array[argc + 100];`语句越界访问0x0026fc2c地址。
    
2. 2.第二段表示0x0026fc2c地址位于线程帧中偏移量420处的栈中，在main.cpp的第四行。
    

# 全局缓冲区溢出

内存访问发生在全局分配区域的边界之外。

```cpp
int global_array[100]={-1};int main(int argc, char** argv){    int res = global_array[argc +100];// 异常？    printf("run here");    getchar();    return res;}
```

**上述代码启动运行是不会出现异常(例如程序异常退出)，但是是非常危险的代码**。

因为其隐晦的访问了非预期内存中的四个字节，非常难定位。

那么使用ASan运行后，因为有影子内存以及红区检测技术，问题很快会被发现而且定位到，详细分析如下
![[Pasted image 20240909080042.png]]
![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

1. 1.第一段表示是global-buffer-overflow类型的内存错误，由main.cpp的第六行触发，即`int res = global_array[argc + 100];`语句访问了0x0083b254地址。
    
2. 2.第二段表示0x0083b254这个地址在global_array的右边，即溢出了，同时指出了global_array是在main.cpp的第三行定义的。
    

# 在返回后使用栈内存

在函数内把函数变量栈内存地址返回给外部使用。

```cpp
int* ptr =nullptr;void Fun(){    int local[100];    ptr =&local[0];}int main(int argc, char** argv){    Fun();    int value =*ptr;    printf("value = %d\n", value);    printf("run here");    getchar();    return value;}
```
![[Pasted image 20240909080058.png]]
![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

1. 1.第一段表示stack-use-after-return类型的内存错误，在main.cpp的第14行，即`int value = *ptr;`语句访问0x01809000地址。
    
2. 2.第二段表示0x01809000是一个帧栈变量，在main.cpp的第6行，即`void Fun()`函数。
    

# 限定作用域后使用

在栈变量生命周期的范围之外对该变量地址进行访问。

```cpp
int* p =nullptr;int main(){    {        int x =0;        p =&x;    }    *p =5;    printf("run here");    getchar();    return0;}
```
![[Pasted image 20240909080106.png]]
![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

1. 1.第一段表示stack-use-after-scope类型的内存错误，在main.cpp的第11行，即`*p = 5;`语句访问0x0055fc58地址。
    
2. 2.第二段表示0x0055fc58地址是一个帧栈地址，在main.cpp的第6行。
    

![](https://mmbiz.qlogo.cn/mmbiz_jpg/HovRRdOHarg5Emh3PxSQfLInAZqJibeBl7Pic5dVjzMb1SywQicsc2jCw3GTibWia3N4DHibicvO1IWkUyPh4Urxn9DSQ/0?wx_fmt=jpeg)

QtCpp大师

 原创伤脑，赞赏补补脑 

![赞赏二维码](https://mp.weixin.qq.com/s?__biz=MzU0NTU5Njc0Mg==&mid=2247485545&idx=1&sn=33e73631d784e94895a055de041069fa&chksm=fa0bc4420f28879c17e053f8fd58549d74abfa0859d6cadbc12d1fcec531a59b74f8c50e535b&mpshare=1&scene=24&srcid=09016KyMnuQstGXCNYWkRYRd&sharer_shareinfo=c510fce4bd8828b4101cb4b4ba62a26b&sharer_shareinfo_first=c510fce4bd8828b4101cb4b4ba62a26b&key=daf9bdc5abc4e8d0dd3ab79a262b0528ecb6eb2db6e8a7ff140b70256afa7356ce8effdb500f0c5a851221734226d3a9909f35332b4d8ea1bc2578e052a3486b17ca14a5ebb401ef4295b688a7cea3d623cfe69ec6015e8103ad0944f3b3348c5b30919a5383bc9faf7d5e8c78a0209bdce6bf1e32b52bac53cc7cf74bc819ec&ascene=14&uin=MTEwNTU1MjgwMw%3D%3D&devicetype=iMac+MacBookAir10%2C1+OSX+OSX+14.6.1+build(23G93)&version=13080710&nettype=WIFI&lang=en&session_us=gh_07df1c1aabee&countrycode=CN&fontScale=100&exportkey=n_ChQIAhIQnKgkuQFk65%2FHknkWGwl%2BKxKUAgIE97dBBAEAAAAAAEFPIA%2F7OigAAAAOpnltbLcz9gKNyK89dVj0GgRusS6ANjKDE%2Fnh1GOqW3c9g45hS2uK2ss8EtwxAQacdC%2BlvuFf%2BxV9T0iGk5uoYkVRPgLXS3BAL8Bi5lZkyxKlthIGKAiJfSEhIebNpXAL0QyzGrC4xx6mE9jWe3iuB1xhQHhJynQcsbRB3Oa49ySX%2Fd93HNIAZbbtLp61qIs6djToQc8wb1OXxMNX9C9jGUaXVrBSxvSGF4JnuUJhq3B5tw8cwquvlXCWSyWXgc1O6fBlls9Lkw0yuQzgH%2FotWTBRuMYHeXDfAdO7HCzy5STzX0g40BU9Au%2Bjpy7IEMBuS7vBf4f63VF3JHZvmQ%3D%3D&acctmode=0&pass_ticket=DFqqm4GwHKvI20RVszKXmlOOTED5BsawYndTg8WEbQx1h3Kx750fyPdOuVm1Fuk8&wx_header=0)Like the Author

Reads 359

​

Comment

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/7EROo10OA1rGO3FR7fzpomDJDZTaYK2aq3JpN5QAMQpIdanx8KMwjr78IrVAlqem0JL1qAt4V1qGcOWpX3WeNQ/300?wx_fmt=png&wxfrom=18)

QtCpp大师

7372

Comment

Comment

**Comment**

暂无留言