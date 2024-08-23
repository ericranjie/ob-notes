# 

土豆居士 一口Linux

 _2021年11月19日 11:41_

击上方“**一口Linux**”，选择“**星标公众号**”

# 

干货福利，第一时间送达！

# ![图片](https://mmbiz.qpic.cn/mmbiz/cZV2hRpuAPiaJQXWGyC9wrUzIicibgXayrgibTYarT3A1yzttbtaO0JlV21wMqroGYT3QtPq2C7HMYsvicSB2p7dTBg/640?wx_fmt=gif&wxfrom=13&tp=wxpic "动态黑色音符")

作者：江南一散人

友情提醒：本文介绍的调试技巧非常实用，但为了讲解清楚，篇幅较长，请耐心看完，我保证你定会有收获！

# 引言

程序调试时，你是否遇到过下面几种情况：

1、经过定位，终于找到了程序中的一个BUG，满心欢喜地以为找到了root cause，便迫不及待地修改源码，然后重新编译，重新部署。但验证时却发现，真正的问题并没有解决，代码中还隐藏着更多的问题。

2、调试时，我们找到代码中一个可疑的地方，但是不能100%确定这真的就是个BUG。要想确定，只能修改源码、重新编译、重新部署，然后重新运行验证。

3、已经找到了root cause，但不确定解决方案是否能正常工作，为了验证，不得不反复地修改代码、编译、部署。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/icRxcMBeJfc9RRgr9hSjMW0lF7BWibfHu9fH9yC75cmrBXpibwiahda4j90gdH80pPYYsIju9fViaRwaCN2yVibU3xHQ/640?wx_fmt=jpeg&wxfrom=13&tp=wxpic)

  

对于大型项目，编译过程可能需要几十分钟，甚至几个小时，部署过程则更为复杂漫长！可想而知，如果调试过程中，不得不反复的修改源码，然后重新编译和部署，会是一项多么繁琐和浪费时间的事情！

那么，有没有一种更高效的调试手段，可以避免反复修改代码和编译呢？

当然有！本文将介绍一种GDB调试技巧，可以一边调试，一边修复Bug，可以在不修改代码、不重新编译的前提下即可修复BUG，验证我们的解决方案，大幅提高调试效率！

# 本文预期效果

如下图，冒泡排序程序中，有三个BUG：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

冒泡排序示例

图中已经把三个BUG都标注了出来。正常编译运行时，程序执行结果如下：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

程序执行异常

不过是普通方式执行，还是在GDB中执行，程序都异常终止，无法得到正常结果。

但是，利用本文介绍的调试技巧，可以利用GDB给这个程序制作一个“热补丁”，在不修改代码、不重新编译的前提下，解决掉程序中的三个BUG，让程序正常执行，并得到预期结果！

最终效果，如下图所示：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

打上“热补丁”后，程序正常执行

是不是很有趣呢？下面开始介绍！

# GDB Breakpoint Command Lists

GDB支持断点触发后，自动执行用户预设的一组调试命令。使用方法：

```
commands [bp_id...]  command-listend
```

其中：

- commands是GDB内置关键字
    
- bp_id是断点的ID，也就是info命令显示出来的断点Num，可以指定多个，也可以不指定。当不指定时，默认只对最近一次设置的那个断点有效。
    
- command-list是用户预设的一组命令，当bp_id指定的断点被触发时，GDB会自动执行这些命令。
    
- end表示结束。
    

这个功能适用于各种类型的断点，如breakpoint、watchpoint、catchpoint等。

# 适用场景举例

利用GDB breakpoint commands lists这个特性可以做很多有趣的事情，本文仅列举其中的几个。

1、随时随地printf，不需修改代码和重新编译

看过我之前文章的朋友，应该还记得，我介绍过GDB的动态打印(Dynamic Printf)功能，可以用dprintf命令在代码的任意地方添加动态打印断点，并自动执行格式化打印操作，从而无需修改代码和重新编译就可以在代码中任意增加日志打印信息。

利用GDB breakpoint commands lists功能，可以实现一样的功能，而且除了打印之外，还可以做其它更多的操作，比如dump内存，dump寄存器等。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

2、修改代码执行逻辑，避免修改代码和重新编译

在GDB中可以做很多有趣的事情，比如修改变量、修改寄存器、调用函数等，结合breakpoint command list功能，可以在调试的同时，修改程序执行逻辑，给程序打上“热补丁”。从而可以在调试过程中，快速修复Bug，避免重新修改代码和重新编译，大大提高程序调试的效率！

这是本文重点讲解的场景，稍后会演示如何利用这个功能，在GDB调试的过程中修复掉上文冒泡排序程序中的三个Bug。

3、实现自动化调试，提高调试效率

这个功能，结合GDB支持的脚本功能，以及自定义命令功能，可以实现调试自动化。

这涉及到GDB的很多其它知识，篇幅有限，不再展开讨论，以后更新专门文章讲解！感兴趣的童鞋，不妨右上角关注一下！

# 给冒泡排序打上“热补丁”

现在，我们利用GDB breakpoint command lists功能，给文中的冒泡排序程序打上“热补丁”，演示如何在不修改源码、不重新编译的前提下，解决掉程序中的3个BUG。

再看一下示例程序：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

编译一下：

```
gcc -g bubble.c -o bubble
```

先用GDB加载运行一下：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

程序运行异常，符合我们的预期。

下面我们依次解决冒泡排序程序中的3个BUG。

1、解决第一个BUG

先解决第22行的BUG，也就是传递给了bubble_sort()错误的数组长度。

我们知道，在x64上，函数参数优先采用寄存器传递。那么，我们有这么几种方式可以选择：

1. 把断点设置在bubble_sort()入口第一条指令，然后直接修改存放数组长度n的那个寄存器中的值。
    
2. 把断点设置在bubble_sort()入口处(不必是第一条指令)，在第7行for循环之前，把存放数组长度的变量n的值改掉。
    
3. 把断点设置在main()函数第22行，也就是调用bubble_sort()的地方，然后以正确的参数手动调用bubble_sort()函数，并利用GDB的jump命令，跳过第22行代码的执行。
    

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

考虑到有些童鞋对x64 CPU不是非常了解，或者对GDB的jump命令不熟悉，我们采用第2种方式。而且，这种方式也更简单通用。

我们先给bubble_sort()函数设置断点，然后利用commands命令预设一条命令，把变量n的值修改为10。命令如下：

```
b bubble_sortcommands 1  set var n=10end
```

设置完之后，用run命令开始运行程序。结果如下：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

bubble_sort()处的断点被触发后，程序暂停，用print命令查看变量n的值，已经被修改成了正确的值：10。

可见，我们的设置是有效的。

断点触发后，让程序自动恢复执行

那么，在bubble_sort()处断点被触发，变量n的值被修改之后，如何让程序自动恢复执行呢？

很简单，只需要在预设的命令中添加一个continue命令就可以了。为了证明我们的设置确实是生效的，我们在修改变量n的前后，各添加一个格式化打印语句，把变量n的值打印出来：

```
b bubble_sortcommands 1  printf "The original value of n is %d\n",n  set var n=10  printf "Current value of n is %d\n",n  continueend
```

结果如下图：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

解决第一个BUG

从运行结果可以看出，断点被触发后，我们预设的语句被正确执行，变量n的值被修改为10，然后程序自动恢复执行。

到此，第一个BUG已经解决了。

2、解决第二个BUG

下面，我们解决第7行代码中的数组访问越界错误：数组的元素个数是n，但是bubble_sort()中第一个for循环的终止条件是i<=n，明显会造成访问越界，正确的条件应该是i<n。

要解决这个BUG也很简单，只需要在执行第8行代码之前，判断如果i的值等于n，就跳出循环。对于这个简单的程序，我们直接从bubble_sort()函数return就可以了。

命令如下：

```
b 8 if i==ncommand 2  printf "i = %d, n = %d\n",i,n  return  continueend
```

在第8行设置条件断点，当i==n时断点被触发，然后自动把i和n的值打印出来，再行return命令，从bubble_sort()返回，然后continue命令自动恢复程序执行。

执行结果如下图：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

解决第二个BUG

3、解决第三个BUG

下面，解决最后一个BUG，第23行数组访问越界错误。

命令如下：

```
b 24 if i==10commands 3  printf "i=%d, exit from for loop!\n",i  jump 26  continueend
```

与第二个BUG类似，在第24行设置条件断点，当==10时触发断点，然后退出循环，让程序跳转到第26行继续执行。

执行结果如下图所示：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

解决第三个BUG

从图中可以看出，三个断点全部被触发，并且预设的命令都正常执行。

我们终于得到了正确的执行结果！

虽然，现在程序可以正常执行了，但是每次手动输入命令还是比较麻烦的。我之前文章介绍过，GDB支持调试脚本，从脚本中加载并执行调试命令。

下面，我们利用GDB脚本，来制作我们的“热补丁”脚本。

# 制作“热补丁”脚本

我们把上文中用来解决三个BUG的命令保存在一个脚本文件中：

```
vi bubble.fix
```

脚本内容如下图：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

bubble.fix 热补丁脚本

bubble.fix脚本中的命令，与上文在GDB中直接输入的命令有几个区别：

1. 删除了格式化打印信息。
    
2. 删除了commands后面的断点ID。上文讲过，commands后面的断点ID可以省略，表示对最近一次设置的断点有效。为了让脚本更加通用，每个commands都紧跟在break命令之后，因此直接省略了断点ID。
    

GDB的脚本可以通过两种方式执行：

1. 启动GDB时，用-x参数指定要执行的脚本文件。
    
2. 启动GDB后，执行source命令执行指定的脚本。
    

下面，我们用第二种方式演示一下，如下图所示：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

执行bubble.fix脚本

使用source命令加载并执行bubble.fix，然后用run命令执行程序，三个断点均被触发，且预设的命令全部被正确执行，最后程序运行正常，得到期望的结果！

我们现在可以利用我们制作的“热补丁”脚本，在不修改代码、不重新编译和部署的前提下，成功修复程序中的BUG！是不是很有趣呢？

不过，做到这种程度，还不算完美！

尽管得到了正确的结果，但程序执行时，总是会打印我们设置的断点信息，看起来还是有些视觉干扰的。

最后，我们来解决这个问题，让我们的“热补丁”更加完美！

# 优化“热补丁”脚本，隐藏断点信息

在预设的命令中，如果第一条命令是silent，断点被触发的打印信息会被屏蔽掉。

我们把bubble.fix做些修改，把silent命令加进去，如下图所示：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

最终版bubble.fix 脚本

然后，重新执行一下：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

这样，看起来，清爽多了！

到此，我们终于实现了本文的目标：一边debug，一边修复BUG，避免反复修改代码、重新编译和部署、提高调试效率！

# 结语

本文重点介绍了如何利用GDB breakpoint command lists功能，制作“调试热补丁”，修改代码BUG。还可以利用这个功能，快速验证我们的猜想和解决方案，避免反复修改代码和重新编译。

巧用GDB breakpoint command lists功能，可以做很多有趣的事情，如实现调试自动化，提高调试效率等。

end

  

  

**一口Linux** 

  

**关注，回复【****1024****】海量Linux资料赠送**

![](https://res.wx.qq.com/t/fed_upload/b39ef69e-c4d6-4169-8612-5f00a84860e7/wx-avatar-default.svg)

公众号

**精彩文章合集**  

文章推荐

☞【专辑】[ARM](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=MzUxMjEyNDgyNw==&action=getalbum&album_id=1614665559315382276#wechat_redirect)

☞【专辑】[粉丝问答](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=MzUxMjEyNDgyNw==&action=getalbum&album_id=1629876820810465283#wechat_redirect)

☞【专辑】[所有原创](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=MzUxMjEyNDgyNw==&action=getalbum&album_id=1479949091139813387#wechat_redirect)

☞【专辑】[linux](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=MzUxMjEyNDgyNw==&action=getalbum&album_id=1507350615537025026#wechat_redirect)入门

☞【专辑】[计算机网络](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=MzUxMjEyNDgyNw==&action=getalbum&album_id=1598710257097179137#wechat_redirect)

☞【专辑】[Linux驱动](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=MzUxMjEyNDgyNw==&action=getalbum&album_id=1502410824114569216#wechat_redirect)

☞【干货】[嵌入式驱动工程师学习路线](http://mp.weixin.qq.com/s?__biz=MzUxMjEyNDgyNw==&mid=2247496985&idx=1&sn=c3d5e8406ff328be92d3ef4814108cd0&chksm=f96b87edce1c0efb6f60a6a0088c714087e4a908db1938c44251cdd5175462160e26d50baf24&scene=21#wechat_redirect)

☞【干货】[Linux嵌入式所有知识点-思维导图](http://mp.weixin.qq.com/s?__biz=MzUxMjEyNDgyNw==&mid=2247497822&idx=1&sn=1e2aed9294f95ae43b1ad057c2262980&chksm=f96b8aaace1c03bc2c9b0c3a94c023062f15e9ccdea20cd76fd38967b8f2eaad4dfd28e1ca3d&scene=21#wechat_redirect)

  

点击“**阅读原文**”查看更多分享，欢迎**点分享、收藏、点赞、在看**

网络41

网络 · 目录

上一篇手把手教你如何实现一个简单的数据加解密算法下一篇NAT穿透技术、穿透原理和方法详解

阅读原文

阅读 2384

​