# 

LarryS 看雪学苑

 _2022年01月01日 17:59_

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/kR3MmOkZ9r6bzgOopMSaOu2JfqWm4zlibt6OXduRcR83CPvIE8bcvvUykGZtnIaB9DrmPlrpibho94RqiaAAEdj3g/640?wx_fmt=jpeg&wxfrom=13&tp=wxpic)  

本文为看雪论坛精华文章

看雪论坛作者ID：LarryS

  

注：资料下载地址见参考资料

这篇文章是我第一次接触V8的漏洞分析，从环境搭建开始，分析了题目中提供的代码，对涉及到的javascript和v8知识点进行了简单介绍，同时针对存在的OOB漏洞，从最终目的——写入执行shellcode——倒退分析，最终一步步达到目标，并与saelo大神提出的addrof和fakeobj概念进行了对照介绍。

除此之外，本文也提供了一种无需FQ就能够进行v8编译安装的方法：我在按照官网的步骤编译v8的时候总是因为网络不稳的原因fetch失败，几番搜索也没有找到好的方法，后来忘记在哪篇文章中看到node是自带v8的，因此想到了文中的方法，只需要简单的修改，就能实现对于v8代码的修改、编译和安装。

本文对于有经验的人来说可能颇为啰嗦，但是很适合初学者，逻辑上讲，如果你能够跟随本文的步骤，并理解其中的内容，就可以像我入门v8漏洞分析了。

  

  

1

  

**环境搭建**  

#   

# （1）确定v8版本

  

本来Github项目里面是提供了对应的chromium版本的编辑脚本的，但是VPN的网络状态一直不稳定，我在fetch的时候一直没有成功。因此我使用了一些小技巧来安装对应版本的v8。

  

之前已经把项目中提供的ubuntu virtualbox镜像下载下来了，通过chrome://version查看对应的v8版本是V8 7.0.276.3。

  

（2）下载对应的node

  

在https://nodejs.org/en/download/releases/这里找到对应的node版本，下载其源码：

```
wget https://nodejs.org/download/release/v11.1.0/node-v11.1.0.tar.gz  --no-check-certificate
```

  

（3）（可选）根据需求对v8进行修改

  

进入目录node-v11.1.0/deps/v8中，执行

```
git apply ~/attachments/addition-reducer.patch
```

  

打开文件/home/test/node-v11.1.0/deps/v8/v8.gyp，第486行列出了一系列的sources文件目录，在其中添加两项：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

（4）编译&安装

  

最后回到node根目录，下载依赖项(这个版本的node需要python2.6/2.7)，编译node。

```
sudo apt install g++ python make
```

  

此时我们就可以把node指令当作d8来使用了。

  

注意上面是编译debug版本的方法，这样在使用%DebugPrint的时候可以获得更多信息。在下面的分析过程中，我同时编译了release和debug两个版本，无论如何，编译得到的可执行程序都放在了out目录下，debug版本在Debug文件夹中，release版本在Release文件夹中，只要最终把需要的可执行文件放在/usr/local/bin里面，并做好区分就可以了。

  

（5）关于turbolizer

  

以上的编译安装都发生在我下载的ubuntu virtualbox镜像中，因为虚拟机的速度有些慢，所以我在主机上也安装了一版node，用于和漏洞细节无关的测试以及turbolizer的使用。

  

turbolizer所在目录：C:\Users\【用户名】\AppData\Roaming\npm\node_modules\turbolizer

在目录下执行：python -m SimpleHTTPServer，就可以在浏览器通过127.0.0.1:8000使用turbolizer了。

  

  

2

  

**问题代码分析**

  

在addition-reducer.patch文件中可以看到这个题目在TypedLoweringPhase阶段添加了一个DuplicateAdditionReducer，duplicate-addition-reducer.cc_（http://duplicate-addition-reducer.cc/）_的主体代码如下：

```
DuplicateAdditionReducer::DuplicateAdditionReducer(Editor* editor, Graph* graph,
```

  

可以看到，当node的操作类型是kNumberAdd的时候，会执行ReduceAddition对当前节点进行处理，从ReduceAddition的代码可以看出一共分成了四种情况：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

其中第四种情况会发生reduce：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

#   

#   

3

  

**漏洞分析**

  

那么上面这样的reduce会导致怎样的问题出现呢？想要知道这一点，首先要了解javascript中的数据表示。

  

## **3.1 IEEE 754标准的64位浮点格式**

  

在javascript中，数字只有一种类型(Number)，它并不像C那样有int、short、long、float、double等类型，所有的数字都采用IEEE754标准定义的64位浮点格式表示。其格式为：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

使用以下公式转换成真实的数值：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

在表达整数的时候，以上格式可以表示的最大安全整数为：

```
max_value = 1 * (1 + 0.1111...1111) * 2^52  // 小数点后有52个'1'
```

  

也就是：

```
d8> Math.pow(2, 53) - 1
```

  

那么如果在这个最大值上再加1会发生什么呢？

```
// max_value的二进制表示
```

  

准确的说，根据参考资料3，IEEE 754标准在2^52~2^53之间的精度是1，在2^53~2^54之间精度是2（只能表示偶数），在2^51~2^52之间精度是0.5，以此类推。

  

## **3.2 x+1+1与x+2**

  

虽然在上面的二进制表示中，max_value+2得到的二进制结果转换为十进制是9007199254740994，但这并不是在javascript中的实际结果。因为IEEE 754无法表示9007199254740993这个值，因此9007199254740991 + 2在javascript中由于精度丢失，得到的结果是9007199254740992。

如果对以下代码进行优化（这里我使用的仍旧是普通版本的v8）：

```
function opt_me() {
```

  

在Turbolizer中得到sea of nodes：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

可以看到javascript无法对9007199254740992进行递增，得到的永远都是同样的值，而直接+2则可以得到正确的数值。

也就是说，当pareng_left的数值是Number.MAX_SAFE_INTEGER + 1的时候，下图中的reduce是错误的：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

## **3.3 OOB**

  

根据上面的推断，当数值处于9007199254740992这个临界点的时候，DuplicateAdditionReducer的做法会出现问题。那么这会导致什么结果呢？先写一段简单的代码：

```
function try1(x) {
```

  

首先定义函数try1，其中包含了：

- 一个长度为4的数组，用于实现越界读；  
    
- 变量y，会根据参数x取9007199254740992或者9007199254740989，其中9007199254740992是为了之后+1+1操作触发漏洞，9007199254740989这个值需要保证完成+1+1后不等于9007199254740992（当然加法之前相等也不行，否则TurboFan会将y优化成一个常数，try1的返回值是固定的，所有过程都优化掉了），且和9007199254740990的差为正数；  
    
- y+1+1操作用于触发漏洞，y的可能取值为9007199254740992或9007199254740991，经过TypedLoweringPhase的DuplicateAdditionReducer之后，可能取值变成了9007199254740994或者9007199254740991；  
    
- 减法操作，得到索引值，可能取值为2或1，经过DuplicateAdditionReducer后，可能取值变成了4（超过了数组索引值范围）或1。  
    
- 返回bug_arr[y]，发生越界读。
    

  

执行上述代码，得到：

```
root@test-vm:/home/test/ctf2018# node --allow-natives-syntax try1.js
```

  

说明try1(true)确实发生了越界读。

我们可以在turbolizer中直观的看到问题所在，在typer阶段，经过两次SpeculativeNumberAdd操作之后，结果的最大值仍旧为9007199254740992。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

而到了typed lowering阶段，经过DuplicateAdditionReducer之后，可以看到原本的两个SpeculativeNumberAdd被简化成了一个NumberAdd，加数变成了2，但是结果并没有更新，因此最后的CheckBound得到的索引边界范围仍旧在[1,2]之间。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

到达escape analysis阶段的时候，可以看到CheckBound节点的索引是Range(1,2)，长度是Range(4,4)：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

而紧接着的simplified lowering阶段会对CheckBound节点进行检查：

```
// simplified-lowering.cc
```

  

注意到上面代码中的判断(index_type.Min() >= 0.0 && index_type.Max() < length_type.Min())，如果这个条件成立，CheckBound节点就会被替换掉。而在此例中，index_type.Max()==2，length_type.Min()==4，条件成立。

因此经过simplified lowering，得到：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

可以看到减法操作之后的CheckBound节点已经不见了，而TurboFan仍旧认为数组的索引范围最大值为2。

  

## **3.4 扩大OOB范围**

  

之前的代码可以看成一个基础模板：

```
function try1(x) {
```

  

必须要保证以下条件：

```
-1 < normal_value - start_value + 2 < 4
```

  

因此该模板下，可以读取到的最大偏移(9007199254740994-start_value)是5：

```
function try1(x) {
```

  

结果：

```
root@test-vm:/home/test/ctf2018# node --allow-natives-syntax try1.js
```

  

那么要如何扩大读取的范围呢？

在3.1小结关于IEEE 754标准的介绍中，我们提到它的精度问题，“IEEE 754标准在2^52~2^53之间的精度是1，在2^53~2^54之间精度是2（只能表示偶数）”，可以想见，当数值范围在2^54~2^55的时候，它的精度应该是4：

```
d8> x = Math.pow(2, 54)
```

  

因此我们可以通过增大数值的方式扩大可读写范围。或者，在参考资料2中，也提到可以通过乘法的方式扩大这一范围：

```
d8> x = Math.pow(2, 53)
```

  

通过上面的手段，理论上来讲我们可以读写bug_arr数组后的任意空间数据。但是如何利用这一能力呢？

  

  

4

  

**利用OOB W/R的方法**

##   

## **4.1 前置知识**

###   

### 4.1.1 Pointer Tagging

  

参考资料5，6

在进行具体的调试分析之前，需要对v8中表示javascript对象的方式有所了解。

在v8中，所有的javascript值无论类型如何，都用对象表示，并且保存在堆中，这样就可以使用指针的方式索引所有值。但是存在一种情况，就是在短时间内使用频率特别高的小的整数，例如循环中使用的索引值。为了避免每次使用的时候都重新分配一个新的number对象，V8使用了一种叫做pointer tagging的技巧来处理这种情况。

具体来说，V8使用两种形式表示javascript中的值，一种叫做Smi(Samll Integer)，一种叫做HeapObject。

在64位的机器上，Smi的最低位固定为0，高32位用于表示真正的数值，其余位为0；HeapObject的最低位固定为1，其余的63位用来存放地址。

正是由于这种机制，在下面调试分析的过程中就出现了需要将地址减1的情况。

  

### 4.1.2 数组相关结构

  

参考资料5，9

每次在javascript中使用var arr = []或者var arr = new Array()的时候，就会创建一个JSArray对象以及一个FixedArray或者FixedDoubleArray对象。如果数组中的元素是整数，就创建FixedArray，否则创建FixedDoubleArray。可以把JSArray看作是数组的头部，而FixedArray/FixedDoubleArray中保存了真正的数组元素。这三个结构在内存中的构成如下：

```
// JSArray
```

  

在JSArray和FixedArray中分别存在一个length属性，FixedArray中的length属性相当于数组的capacity，是系统为数组预分配的空间大小，而真正和代码中的索引值相关的是存在于JSArray中的length属性。

除了上面的这三种对象之外，每次在javascript中使用var buf = new ArrayBuffer(3)的时候，就会创建一个JSArrayBuffer对象。与JSArray不同之处在于，ArrayBuffer中存在一个叫做backing pointer的东西：

```
// JSArrayBuffer
```

  

注意到这里面既有一个Pointer to Elements，也有一个Pointer to Backing Store，前者和JSArray中的意义相同，而backing pointer所指向的虽然也是数组中的数据，但是它所指向的内容是纯二进制数据，不具备类型信息。因此在使用的时候，可以通过typed array objects或者DataView用具体的数据类型表达ArrayBuffer中的数据。这一特性也对接下来的漏洞利用十分有帮助。

  

## **4.2 利用方法分析**

  

倒推：执行shellcode ← 找到一块可执行内存并写入shellcode ← WebAssembly函数代码所在内存可执行 ← 获取函数对象所在地址 + 写入shellcode

根据上面的倒退，我们需要具有获取对象地址以及任意写（读）的能力。

此次分析的漏洞让我们可以读写bug_arr数组后可变长的一段内存，所以：

  

如果对象正好放在bug_arr的后面，就可以通过这个漏洞获取到对象的地址；

  

如果bug_arr后面有一个指针，就可以通过这个漏洞修改指针的数值，从而对其指向的内存进行读写。

  

那么我们要怎么做呢？

##   

## **4.3 更简单的OOB方式**

  

在获取能力之前，我们需要先对OOB进行优化，因为目前这种通过小心构造数值，利用漏洞访问数组范围外数据的方式十分复杂，无法灵活地进行数据访问。

因此我们可以在bug_arr后再定义一个数组(oob_arr)，利用漏洞访问并修改oob_arr中JSArray的length属性，之后，就可以通过oob_arr[idx]的方式访问到oob_arr后面的大片数据了。这种通过索引进行访问的方式显然十分便捷。

下面通过调试的方式确定这一方法的可行性。

###   

### 4.3.1 调试方法

  

1. js脚本的修改
    

1. 在适当位置添加%DebugPrint，输出感兴趣的对象的信息；
    
2. 脚本最后添加一个无限循环语句
    

3. node —allow-natives-syntax 执行脚本
    

1. 从%DebugPrint的输出中获得对象地址等信息，用于后续调试；
    
2. 由于无限循环语句，进程挂起；
    

5. 通过ps -a | grep node的方式找到该进程的PID；
    
6. 使用gdb attach [pid]或者其他调试器附加到该进程，开始调试。
    

###   

###   

### 4.3.2 实验代码

  

数值转换借用了参考资料2中的方法：

```
let ab = new ArrayBuffer(8);
```

###   

### 4.3.3 调试分析

  

执行上面的脚本，得到输出：

```
root@test-vm:/home/test/ctf2018# node --allow-natives-syntax  try2.js
```

  

这里我没有使用debug版本，因为只需要JSArray的地址就可以了。

然后使用gdb调试，检查0x3e7e89d3c159-1处的数据（注意这里由于pointer tagging，进行了减1操作）：

```
(gdb) x/20gx 0x3e7e89d3c159-1
```

  

有了前面前置知识的介绍，我们知道bug_arr数组元素位于0x00001ca19adc6409-1的位置：

```
(gdb) x/20gx 0x000012b40ca867c9-1
```

  

注意到bug_arr和oob_arr在内存中是彼此紧邻的，这样当bug_arr的长度为5，而我们获得的索引值是6时，就会读取到bug_arr数组末尾的第二个元素，也就是oob_arr FixedDoubleArray中的length属性。

而oob_arr的JSArray位于FixedDoubleArray的后面，我们需要覆盖更远的范围。

  

### 4.3.4 覆写长度属性

  

根据上面的调试输出，要覆写的length属性距离bug_arr的起始元素的偏移为bug_arr.length + 2 + oob_arr.length + 3，必须合理设置代码中的数值，才能让程序正好覆盖到length属性上：

```
function try2(x) {
```

  

得到输出结果：

```
Index is 12
```

  

注意到oob_arr的长度已经变成了100。现在我们可以很方便的通过oob_arr索引后面的数据了。

  

## **4.4 获取对象地址的能力**

  

正如之前所说，只要把对象放在oob_arr数组后面，就可以通过漏洞获取到该对象的地址。为了方便定位，我们在对象前面放置一个MARKER，两者组合放在一个数组中。

```
let OOB_LENGTH_64 = 0x0000006400000000;
```

  

得到输出：

```
test@test-vm:~/ctf2018$ node --allow-natives-syntax try2.js
```

  

通过调试查看其内存：

```
(gdb) x/4gx 0x3d07774cffe0
```

  

可以看到这段代码确实输出了对象的地址0x0000356298e93df1 - 1。

##   

## **4.5 任意地址读写的能力**

  

前面说过了，这个能力需要在oob_arr后面放一个指针，然后通过修改指针来实现“任意地址”读写的功能。理论上来说，指针的实现可以通过各种数组实现，因为它们在偏移0x10的位置都有一个Pointer to Elements。但是这里我们选择ArrayBuffer，因为它有backing pointer，这个指针指向的内存是纯二进制数据，可以按照各种数据形式进行访问，十分方便。

我们可以通过oob_arr[idx]把可执行内存的地址赋值给backing pointer，然后再利用typed array objects访问backing pointer指向的内存，按照自己想要的格式写入shellcode。

代码如下：

```
let AB_LENGTH = 0x100;
```

  

得到输出：

```
test@test-vm:~/ctf2018$ node --allow-natives-syntax try2.js
```

  

由于对Math对象的map进行了篡改，修改成了非法的值，因此引用的时候发生了错误。

在调试器中查看内存：

```
(gdb) x/4gx 0x1f1c989d0668
```

  

可以看到位于obj_arr[1]的对象地址0x000007bfc3593df1所指向的内存，首位确实被修改成了0x4141414141414141。

  

## **4.6 利用WebAssembly获取可执行内存地址**

###   

### 4.6.1 获取WebAssembly编译后函数

  

到目前为止我们已经具有了获取对象地址以及向任意地址写数据的能力，接下来需要确定的是向哪里写数据。

根据参考资料8，v8的JIT代码页有一个写保护，它会根据标志位在RW和RX权限之间转换，因此无法用shellcode覆盖已编译的函数代码页。

但是除了javascript之外，v8还会编译WebAssembly，而它的代码页的写保护默认是关闭的，因此已编译的WebAssembly代码页具有RWX权限。

参考资料10提供了一个网页，可以把你编写的C语言代码转换为WebAssembly，生成对应的二进制文件，并提供了加载执行这段二进制代码的方法：

```
// 这里保存二进制代码
```

###   

### 4.6.2 分析源码得到jump table所在偏移

  

但是我们想要的可执行内存并不是f的地址，按照参考资料8，它的结构是这样的：

```
- JSFunction ->
```

  

之后jump table start + function index就会到达这个函数的jump table entry，它指向的就是RWX的内存，也是这个函数的入口点。

由于我们这里只有一个函数，因此只要获得jump table start address就可以了，它的首位就保存了函数的入口点，也就是可执行内存的地址。

参考资料2就是预先确定了这几个位置的偏移，然后不断地读取数据，最终得到可执行内存的地址；而参考资料10直接通过WasmInstanceObject pointer获得jump table start address。

上面所示结构以及各个数据的偏移（首位偏移为0）可以这样得到：

首先是一些头部信息大小：

```
// node/v8/src/objects.h
```

  

然后从JS_FUNCTION结构开始看：

```
// node/v8/src/objects.h
```

  

可以看到kSharedFunctionInfoOffset在首位，再加上前面的JSObject::kHeaderSize，kSharedFunctionInfoOffset的偏移是3。

```
// /node/v8/src/objects/shared-function-info.h
```

  

kFunctionDataOffset在第二位，但是第一位的大小是0，所以实际应该是在第一位，再加上前面的HeapObject::kHeaderSize，kFunctionDataOffset的偏移是1。

```
// /node/v8/src/wasm/wasm-objects.h
```

  

kInstanceOffset在第二位，再加上前面的HeapObject::kHeaderSize，kInstanceOffset的偏移是2。

```
// /node/v8/src/wasm/wasm-objects.h
```

  

kJumpTableStartOffset在第28位，因为前面有一个大小为0，所以实际上是在第27位，再加上前面的JSObject::kHeaderSize，kJumpTableStartOffset的偏移是29。

整理偏移值得到：

```
kSharedFunctionInfoOffset: 3
```

###   

### 4.6.3 偏移值验证

  

代码：

```
var wasm_code = new Uint8Array([0,97,115,109,1,0,0,0,1,133,128,128,128,0,1,96,0,1,127,3,130,128,128,128,0,1,0,4,132,128,128,128,0,1,112,0,0,5,131,128,128,128,0,1,0,1,6,129,128,128,128,0,0,7,145,128,128,128,0,2,6,109,101,109,111,114,121,2,0,4,109,97,105,110,0,0,10,138,128,128,128,0,1,132,128,128,128,0,0,65,42,11]);
```

  

得到输出：

```
test@test-vm:~/ctf2018$ node_debug --allow-natives-syntax wasm_test.js
```

  

在调试器中查看0x15e5d1c8141内存，并根据前面得到的偏移值跟随验证：

```
(gdb) x/4gx 0x15e5d1c8140   // JS_FUNCTION
```

  

可以看到最终得到的数值0x00001b3e42795000确实是一个4KB对齐的地址。

  

### 4.6.4 获得可执行内存的代码

  

我在最初实验的时候，还没有认真进行上面的分析，只是按照参考资料10的方法，先通过%DebugPrint输出WasmInstanceObject的地址，然后在调试器中查看其后的大片内存，找到4KB对齐的地址(十六进制时低三位均为0)所在的偏移，然后直接确定了jump table start address。

代码如下：

```
let AB_LENGTH = 0x100;
```

  

输出结果：

```
test@test-vm:~/ctf2018$ node --allow-natives-syntax try2.js
```

  

输出结果正常！

  

## **4.7 写入shellcode，触发！**

  

接下来这事儿就很简单了，我们已经知道怎么获取对象地址，怎么读写任意地址，也知道要写入的位置，只要把所有东西凑到一起就可以了。

这次给出完整的代码：

```
let shellcode=[0x90909090,0x90909090,0x782fb848,0x636c6163,0x48500000,0x73752fb8,0x69622f72,0x8948506e,0xc03148e7,0x89485750,0xd23148e6,0x3ac0c748,0x50000030,0x4944b848,0x414c5053,0x48503d59,0x3148e289,0x485250c0,0xc748e289,0x00003bc0,0x050f00];
```

  

成功弹出计算器：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

##   

## **4.8 关于fakeobj和addrof**

  

saelo的文章_(http://www.phrack.org/papers/attacking_javascript_engines.html)_详细地介绍了如何对OOB类型的漏洞进行利用，提到了通过fakeobj和addrof原语实现任意地址读写（addrof原语接收一个对象参数并返回其地址，fakeobj原语接收一个地址参数并返回一个位于该地址的假的对象），从而写入shellcode并执行。由于这篇文章针对的是Webkit的引擎JavaScriptCore，我对此并不十分了解，因此没有仔细看这篇文章，也就没放在参考资料中。

参考资料10中对于该利用方法在v8上的应用给出了很好的示例介绍，这篇资料中专门定义了addrof和fakeobj函数，然后通过这两个函数又定义了任意读和任意写函数，个人觉得是这种漏洞利用方法的模板化范例了。

不过由于此次分析的漏洞特性，不需要也不方便单独定义出fakeobj和addrof函数式。一方面这次的漏洞需要很精细地挑选数值，实现一次OOB，虽然OOB的长度可变，但每组确定的数值导致的OOB长度是固定的，这种情况下单独定义函数写出来的代码会很乱；另一方面，由于这次漏洞可以覆盖bug_arr数组后可变长的一段内存，我们可以直接利用这一点覆盖后方数组的长度属性，从而获得一种更灵活的OOB访问方式，最终也实现了类似addrof和fakeobj的功能，从而实现了任意地址读写。

具体来说，代码中的addrof部分很明显，就是下面这段代码：

```
// get address of wasm_instance
```

  

将想要获得地址的对象放在oob_arr后方的一个固定位置，然后通过越界读就可以获得其地址了。

fakeobj的功能不太明显，它并不是一段代码，而是buf_arr本身，可以说buf_arr就是一个fake object。因为每次我们想在某个地址上读写数据的时候，就用这个地址替换buf_arr的backing pointer，这样就算是得到了一个“假的”对象了，然后再通过typed array objects对目标地址进行读写：

```
// read the number at jump_table_ptr
```

#   

  

5

  

**总结**

  

这次的漏洞分析花费了我近一个月的时间，一开始想要分析的并不是这个CTF题目，而是一个V8的CVE漏洞，结果发现自己什么都不懂，然后就从参考资料一路点击，最终定位到了这道题目。

除了基础资料之外，看的第一篇针对性文章是参考资料2，然后就陷入了fakeobj的怪圈（那时候我还不知道这是什么东西），最终通过参考资料10的解释才逐渐理出头绪，虽然方法相同，但是你会发现我的代码和参考资料2还是有很大差别的。我之所以添加了最后的4.8小节，一个很重要的原因就是要把一开始我的疑惑给理清楚。

在阅读这些参考资料的过程中，我发现saelo的文章真的是一座里程碑，所有OOB的漏洞最后都提到了这篇文章，因此一定要把它加到我的待阅清单里。

最后十分感谢参考资料中的所有文章作者。

如果内容有错误，欢迎指正 (^_^)

#   

  

6

  

**参考资料**

#   

# 1、Github项目（题目信息及资料下载）

_https://github.com/google/google-ctf/tree/master/2018/finals/pwn-just-in-time_

#   

# 2、Introduction to TurboFan（关键文章！！！）

# _https://doar-e.github.io/blog/2019/01/28/introduction-to-turbofan/_

  

# 3、Double-precision floating-point format（基础知识）

# _https://en.wikipedia.org/wiki/Double-precision_floating-point_format_

  

# 4、Google CTF justintime exploit（参考，没有仔细看）

# _https://eternalsakura13.com/2018/11/19/justintime/_

  

# 5、V8 Objects and Their Structures（基础知识）

# _https://pwnbykenny.com/2020/07/05/v8-objects-and-their-structures/_

  

# 6、An Introduction to Speculative Optimization in V8（只看了前半部分，v8基础）

# _https://ponyfoo.com/articles/an-introduction-to-speculative-optimization-in-v8_

  

# 7、Chrome V8文档（一些不了解的琐碎的知识点是在这里学习的）

# _https://v8.dev/blog_

  

# 8、Exploiting the Math.expm1 typing bug in V8（关键文章！！！）

# _https://abiondo.me/2019/01/02/exploiting-math-expm1-v8_

  

# 9、ArrayBuffer（关于ArrayBuffer的介绍）

# _https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/ArrayBuffer_

  

# 10、Exploiting v8: *CTF 2019 oob-v8（关键文章！！！个人觉得对于addrof和fakeobj介绍的很好）

_https://faraz.faith/2019-12-13-starctf-oob-v8-indepth/_

  

  

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E) 

  

**看雪ID：LarryS**

https://bbs.pediy.com/user-home-600394.htm

*本文由看雪论坛 LarryS 原创，转载请注明来自看雪社区

  

  

[![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458409596&idx=3&sn=658012a9debc76c73efed03342d41358&chksm=b18f6af686f8e3e09180522b243def309e265532cfadfdbf5becd7d9ec90f4505136959ebdfe&scene=21#wechat_redirect)  

  

**#** **往期推荐**

1.[内核学习-异常处理](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458413193&idx=3&sn=d9e54f5f39476109634b7319cef91205&chksm=b18f580386f8d115cc0c1f30bec88e0c2705d7d5aee020738a5233620a6ab63db202decf5269&scene=21#wechat_redirect)  

2.[内核漏洞学习-HEVD-StackOverflowGS](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458412464&idx=1&sn=5a039a6ed17cd45f21ce5a62193ecadb&chksm=b18f553a86f8dc2c5250cec782b359472e738f92b3a1444d56a2e1be19c4397574c0ae545dd3&scene=21#wechat_redirect)

3.[某钱包转账付款算法分析篇](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458410651&idx=1&sn=070a3637c7f9d78e188d2b4ca7e52c81&chksm=b18f6e1186f8e707496c08a09c6c8dd7685ef81eaca6d7e9a54305779281aca232a981e7f207&scene=21#wechat_redirect)

4.[通过PsSetLoadImageNotifyRoutine学习模块监控与反模块监控](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458410644&idx=1&sn=e36fcc9b2b7a1efffb84bc3673b248b6&chksm=b18f6e1e86f8e708e321843fbdc68404b28db00aa48fed2c974a402a7708db095f94820c3da1&scene=21#wechat_redirect)

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

阅读 1457

​

写留言

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/1UG7KPNHN8EGLfh77kFmnicd9WOic2ibvhCibFdB4bL4srJCgo2wnvdoXLxpIvAkfCmmcptXZB0qKWMoIP8iaibYN2FA/300?wx_fmt=png&wxfrom=18)

看雪学苑

4分享3

写留言

写留言

**留言**

暂无留言