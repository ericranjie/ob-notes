# 

原创 土豆居士 一口Linux

 _2021年10月18日 11:50_

击上方“**一口Linux**”，选择“**星标公众号**”

# 

干货福利，第一时间送达！

# ![图片](https://mmbiz.qpic.cn/mmbiz/cZV2hRpuAPiaJQXWGyC9wrUzIicibgXayrgibTYarT3A1yzttbtaO0JlV21wMqroGYT3QtPq2C7HMYsvicSB2p7dTBg/640?wx_fmt=gif&wxfrom=13&tp=wxpic "动态黑色音符")

  

## 0. 前言

粉丝留言，想知道如何使用Makefile给多个文件和多级目录建立一个工程，必须安排！

关于Makefile的入门参考文章，可以先看这篇文章：

《[Makefile入门教程](https://mp.weixin.qq.com/s?__biz=MzUxMjEyNDgyNw==&mid=2247497099&idx=1&sn=cc1ecb9f77b13726ed7bac1cc8b9ba96&chksm=f96b877fce1c0e69ccd4e0a913bb8dce2f9217b4452083e1bfe94e803b5bba009f10db2cde41&token=1090410464&lang=zh_CN&scene=21#wechat_redirect)》

为了让大家有个更加直观的感受，一口君将之前写的一个小项目，本篇在该项目基础上进行修改。

该项目详细设计和代码，见下文：

《[从0写一个《电话号码管理系统》的C入门项目【适合初学者】](https://mp.weixin.qq.com/s?__biz=MzUxMjEyNDgyNw==&mid=2247497355&idx=1&sn=34acdc6014924551d67f3aeb477ecca5&chksm=f96b847fce1c0d698109845db171eeb37b2774b9cb7a2a392145e4ed180afc05ef8cf5c7d91d&token=1108558673&lang=zh_CN&scene=21#wechat_redirect)》

## 一、文件

好了，开始吧!

我们将该项目的所有功能函数放到以该函数名命名的c文件，同时放到对应名称的子目录中。

比如函数allfree()，存放到 allfree/allfree.c中

最终目录结构如下图所示：

 `peng@ubuntu:/mnt/hgfs/code/phone$ tree .   .   ├── allfree   │   ├── allfree.c   │   └── Makefile   ├── create   │   ├── create.c   │   └── Makefile   ├── delete   │   ├── delete.c   │   └── Makefile   ├── display   │   ├── display.c   │   └── Makefile   ├── include   │   ├── Makefile   │   └── phone.h   ├── init   │   ├── init.c   │   └── Makefile   ├── login   │   ├── login.c   │   └── Makefile   ├── main   │   ├── main.c   │   └── Makefile   ├── Makefile   ├── menu   │   ├── Makefile   │   └── menu.c   ├── scripts   │   └── Makefile   └── search       ├── Makefile       └── search.c      11 directories, 22 files`

直接看下编译结果吧：

`peng@ubuntu:/mnt/hgfs/code/phone$ make   make[1]: Entering directory '/mnt/hgfs/code/phone/allfree'   make[1]: Nothing to be done for 'all'.   make[1]: Leaving directory '/mnt/hgfs/code/phone/allfree'   make[1]: Entering directory '/mnt/hgfs/code/phone/create'   make[1]: Nothing to be done for 'all'.   make[1]: Leaving directory '/mnt/hgfs/code/phone/create'   make[1]: Entering directory '/mnt/hgfs/code/phone/delete'   make[1]: Nothing to be done for 'all'.   make[1]: Leaving directory '/mnt/hgfs/code/phone/delete'   make[1]: Entering directory '/mnt/hgfs/code/phone/display'   make[1]: Nothing to be done for 'all'.   make[1]: Leaving directory '/mnt/hgfs/code/phone/display'   make[1]: Entering directory '/mnt/hgfs/code/phone/init'   make[1]: Nothing to be done for 'all'.   make[1]: Leaving directory '/mnt/hgfs/code/phone/init'   make[1]: Entering directory '/mnt/hgfs/code/phone/login'   make[1]: Nothing to be done for 'all'.   make[1]: Leaving directory '/mnt/hgfs/code/phone/login'   make[1]: Entering directory '/mnt/hgfs/code/phone/menu'   make[1]: Nothing to be done for 'all'.   make[1]: Leaving directory '/mnt/hgfs/code/phone/menu'   make[1]: Entering directory '/mnt/hgfs/code/phone/search'   make[1]: Nothing to be done for 'all'.   make[1]: Leaving directory '/mnt/hgfs/code/phone/search'   make[1]: Entering directory '/mnt/hgfs/code/phone/main'   make[1]: Nothing to be done for 'all'.   make[1]: Leaving directory '/mnt/hgfs/code/phone/main'   gcc -Wall -O3 -o phone allfree/*.o create/*.o delete/*.o display/*.o init/*.o login/*.o menu/*.o search/*.o main/*.o -lpthread   phone make done!` 

运行结果如下：
![[Pasted image 20240929113949.png]]
![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

## 二、Makefile常用基础知识点

### [0] 符号`'@' '$' '$$' '-' '-n '`的说明

1. `'@'`  
    通常makefile会将其执行的命令行在执行前输出到屏幕上。如果将‘@’添加到命令行前，这个命令将不被make回显出来。例如：
    

`@echo  --compiling module----;  // 屏幕输出  --compiling module----   echo  --compiling module----;  // 没有@ 屏幕输出echo  --compiling module----`   

2. `' - '`
    

通常删除，创建文件如果碰到文件不存在或者已经创建，那么希望忽略掉这个错误，继续执行，就可以在命令前面添加 -，

`-rm dir；   -mkdir aaadir；   `

3. `' $ '`美元符号`$`，主要扩展打开makefile中定义的变量
    
4. `' $$ '``$$` 符号主要扩展打开makefile中定义的shell变量
    

### [1] wildcard

说明: 列出当前目录下所有符合模式“ PATTERN”格式的文件名,并且以空格分开。“ PATTERN”使用shell可识别的通配符，包括“ ?”（单字符）、“ *”（多字符）等。示例：

`$(wildcard *.c)` 

返回值为当前目录下所有.c 源文件列表。

### [2] patsubst

说明：把字串“ x.c.c bar.c”中以.c 结尾的单词替换成以.o 结尾的字符。示例：

`$(patsubst %.c,%.o,x.c.c bar.c)   `

函数的返回结果 是

 `x.c.o bar.o`

### [3] notdir

说明：去除文件名中的路径信息 示例：

`SRC = ( notdir ./src/a.c )` 

去除文件a . c 的路径信息 ， 使用 (notdir ./src/a.c) 去除文件a.c的路径信息，使用 (notdir./src/a.c)去除文件a.c的路径信息，使用(SRC)得到的是不带路径的文件名称，即a.c。

### [4] 包含头文件路径

使用-I+头文件路径的方式可以指定编译器的头文件的路径 示例：

`INCLUDES = -I./inc   `

`$(CC) -c $(INCLUDES) $(SRC)   `

### [5] addsuffix

函数名称：加后缀函数—addsuffix。语法：

`$(addsuffix SUFFIX,NAMES…)` 

函数功能：为“NAMES…”中的每一个文件名添加后缀“SUFFIX”。参数“NAMES…” 为空格分割的文件名序列，将“SUFFIX”追加到此序列的每一个文件名 的末尾。返回值：以单空格分割的添加了后缀“SUFFIX”的文件名序列。函数说明：示例：

`$(addsuffix .c,foo bar)` 

返回值为

`foo.c bar.c   `

### [6] 包含另外一个文件：include

在Makefile使用include关键字可以把别的Makefile包含进来，这很像C语言的#include，被包含的文件会原模原样的放在当前文件的包含位置。比如命令

`include file.dep   `

即把file.dep文件在当前Makefile文件中展开，亦即把file.dep文件的内容包含进当前Makefile文件

> 在 include前面可以有一些空字符，但是绝不能是[Tab]键开始。

### [7] foreach

foreach函数和别的函数非常的不一样。因为这个函数是用来做循环用的 语法是：

`$(foreach <var>,<list>,<text> )   `

这个函数的意思是，把参数中的单词逐一取出放到参数所指定的变量中，然后再执行

每一次

所以，最好是一个变量名，可以是一个表达式，而

举例:

`names := a b c d   files := $(foreach n,$(names),$(n).o)   `

上面的例子中，`$(name)`中的单词会被挨个取出，并存到变量“n”中，“`$(n).o`”每次根据“`$(n)`”计算出一个值，这些值以空格分隔，最后作为foreach函数的返回，所以，`$(files)`的值是“a.o b.o c.o d.o”。

注意，foreach中的参数是一个临时的局部变量，foreach函数执行完后，参数的变量将不在作用，其作用域只在foreach函数当中。

### [8] call

“ call”函数是唯一一个可以创建定制化参数函数的引用函数。使用这个函数可以实现对用户自己定义函数引用。我们可以将一个变量定义为一个复杂的表达式，用“ call”函数根据不同的参数对它进行展开来获得不同的结果。

函数语法：

`$(call variable,param1,param2,...)   `

函数功能：在执行时，将它的参数“ param”依次赋值给临时变量“ `$(1)`”、“ **`$(2)`**” call 函数对参数的数目没有限制，也可以没有参数值，没有参数值的“ call”没有任何实际存在的意义。执行时变量“ variable”被展开为在函数上下文有效的临时变量，变量定义中的“ `$(1)`”作为第一个参数，并将函数参数值中的第一个参数赋值给它；变量中的“ **`$(2)`**”一样被赋值为函数的第二个参数值；依此类推（变量**`$(0)`**代表变量“ variable”本身）。之后对变量“ variable” 表达式的计算值。

返回值：参数值“ param”依次替换“ **`$(1)`**”、“ **`$(2)`**”…… 之后变量“ variable”定义的表达式的计算值。

函数说明：

1. 函数中“ variable”是一个变量名，而不是变量引用。因此，通常“ call”函数中的“ variable”中不包含“ **`$`**”（当然，除非此变量名是一个计算的变量名）。
    
2. 当变量“ variable”是一个 make 内嵌的函数名时（如“ if”、“ foreach”、“ strip”等），对“ param”参数的使用需要注意，因为不合适或者不正确的参数将会导致函数的返回值难以预料。
    
3. 函数中多个“ param”之间使用逗号分割。
    
4. 变量“ variable”在定义时不能定义为直接展开式！只能定义为递归展开式。
    

函数示例：

`reverse = $(2)$(1)   foo = $(call reverse,a,b)   all:    @echo "foo=$(foo)"   `

执行结果：

`foo=ba   `

即a替代了替代了(2)

## 三、编译详细说明
![[Pasted image 20240929114001.png]]
![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

我们在根目录下执行make命令后，详细步骤如下：

1. include scripts/Makefile ：将文件替换到当前位置，
    
2. 使用默认的目标all，该目标依赖于`$(Target)``$(Target)` 在scripts/Makefile中定义了，即phone
    
3. 而`$(Target)`依赖于mm
    
4. mm这个目标会执行
    

`@ $(foreach n,$(Modules),$(call modules_make,$(n)))   `

Modules是所有的目录名字集合， foreach 会遍历字符串$(Modules)中每个词语， 每个词语会赋值给n， 同时执行语句：

`call modules_make,$(n)   `

5. modules_make 被`$(MAKE) -C $(1)`所替代，
    

**`$(MAKE)`** 有默认的名字make -C：进入子目录执行make`$(1)` ：是步骤4中`$(n)`,即每一个目录名字

最终步骤4的语句就是进入到每一个目录下，执行每一个目录下的Makefile

6. 进入某一个子目录下，执行Makefile 默认目标是all，依赖Objs
    

`Objs := $(patsubst %.c,%.o,$(Source))   `

patsubst 把字串`$ource`中以.c 结尾的单词替换成以.o 结尾的字符 而

`Source := $(wildcard ./*.c)   `

wildcard 会列举出当前目录下所有的.c文件

所以第6步最终就是将子目录下的所有的.c文件，编译生成对应文件名的.o文件

8.   
    

`$(CC) $(CFLAGS) -o $(Target) $(AllObjs) $(Libs)   `

这几个变量都在文件scripts/Makefile中定义`$(CC)` ：替换成gcc，制定编译器`$(CFLAGS)` ：替换成-Wall -O3，即编译时的优化等级`-o $(Target)`：生成可执行程序phone`$(AllObjs)` ：

`AllObjs := $(addsuffix /*.o,$(Modules))   `

addsuffix 会将 /*.o追加到`$(Modules)`中所有的词语后面，也就是我们之前在子目录下编译生成的所有的.o文件`$(Libs)` ：替换为-lpthread，即所需要的动态库

大家可以根据这个步骤，来分析一下执行make clean时，执行步骤

完整的实例程序公众号后台回复：**电话号码管理**

《电话号码管理-makefile版.rar》

end

  

  

**一口Linux** 

  

**关注，回复【****1024****】海量Linux资料赠送**

![](https://res.wx.qq.com/t/fed_upload/b39ef69e-c4d6-4169-8612-5f00a84860e7/wx-avatar-default.svg)

公众号

**精彩文章合集**

[ARM](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=MzUxMjEyNDgyNw==&action=getalbum&album_id=1614665559315382276#wechat_redirect)

[粉丝问答](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=MzUxMjEyNDgyNw==&action=getalbum&album_id=1629876820810465283#wechat_redirect)

[所有原创](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=MzUxMjEyNDgyNw==&action=getalbum&album_id=1479949091139813387#wechat_redirect)

[linux](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=MzUxMjEyNDgyNw==&action=getalbum&album_id=1507350615537025026#wechat_redirect)入门

[计算机网络](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=MzUxMjEyNDgyNw==&action=getalbum&album_id=1598710257097179137#wechat_redirect)

[Linux驱动](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=MzUxMjEyNDgyNw==&action=getalbum&album_id=1502410824114569216#wechat_redirect)

[嵌入式驱动工程师学习路线](http://mp.weixin.qq.com/s?__biz=MzUxMjEyNDgyNw==&mid=2247496985&idx=1&sn=c3d5e8406ff328be92d3ef4814108cd0&chksm=f96b87edce1c0efb6f60a6a0088c714087e4a908db1938c44251cdd5175462160e26d50baf24&scene=21#wechat_redirect)

[Linux嵌入式所有知识点-思维导图](http://mp.weixin.qq.com/s?__biz=MzUxMjEyNDgyNw==&mid=2247497822&idx=1&sn=1e2aed9294f95ae43b1ad057c2262980&chksm=f96b8aaace1c03bc2c9b0c3a94c023062f15e9ccdea20cd76fd38967b8f2eaad4dfd28e1ca3d&scene=21#wechat_redirect)

  

点击“**阅读原文**”查看更多分享，欢迎**点分享、收藏、点赞、在看**

![](https://mmbiz.qlogo.cn/mmbiz_jpg/h5qLUehhRhd5fEkYx3icmhEn7HzMF25aia45k0kTVelC8aG5ZNsXn5IXCjR3hkgg4fqM2zFRXDp3GqrbDhKoeEoQ/0?wx_fmt=jpeg)

土豆居士

 植发 

![赞赏二维码](https://mp.weixin.qq.com/s?__biz=MzUxMjEyNDgyNw==&mid=2247498748&idx=1&sn=17418a198f1edc4f089cc17537dbfb3f&chksm=f96b8908ce1c001e6533f6e85e37a88c55e656761f6c6c61e4d4eac91fc9ac23179d0ca84075&mpshare=1&scene=24&srcid=1018SnSiVBTfk25n4b34Lbab&sharer_sharetime=1634532613669&sharer_shareid=5fb9813bfe9ffc983435bfc8d8c5e9ca&key=daf9bdc5abc4e8d09fe65b34b0ca630414107d087d6eed375be1a3b1fb7dbea5a77555ea46e9ccf02b0766d0d7cfb0f06d39bb4847787f98c349eaf66bb63a7f7df6afacae2db33ccd82b514111f2ceb886cf4f7bf6fb8543ee541b222e2d3206a287566af2c563bd117c127a94658800e96f28d894ecc1a59d9bd99d965c462&ascene=0&uin=MTEwNTU1MjgwMw%3D%3D&devicetype=Windows+11+x64&version=63090b19&lang=zh_CN&countrycode=CN&exportkey=n_ChQIAhIQdh%2BIYolGQmyUgxurc7oG%2BhLmAQIE97dBBAEAAAAAAKClL6Vzk30AAAAOpnltbLcz9gKNyK89dVj03%2BllfYxZRuYJAyHQPCyxD90OSjVF4v76ED%2B5ogSZnpylelShZL90YkvEBojg%2F2RRgWZFmEFBu6qiU6YkCL%2BtFtIuebBNzp2w56VH2%2BY5biQ%2BlGgVqSLojuqb6eOtBjmeAAhy6BI4DC0qFmqyQ41BHbJAmbScMBZp3tXOZxKyZrLuwjCa6gbhu4pB%2BB2BH%2BfZngTL5xuFKE24pU9VkooLomClwutLpTjnhX5UdA2urrvvPKM205D%2BvRoNBOAR8G0b&acctmode=0&pass_ticket=VwK9XUT4FDH17qBPaQNVFvZRHl5DgShcbzLMAbRnZnRx16CYDejlQbrcAqFbxOz4&wx_header=1&fasttmpl_type=0&fasttmpl_fullversion=7351805-zh_CN-zip&fasttmpl_flag=1)喜欢作者

3人喜欢

![](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

linux134

所有原创240

linux · 目录

上一篇你见过最垃圾的代码长什么样？下一篇Linux中su，sudo，sudo su，sudo -i命令的使用和区别

阅读原文

阅读 6164

​