
jmpcall 看雪学苑

 _2023年05月11日 17:59_ _上海_

![Image](https://mmbiz.qpic.cn/sz_mmbiz_jpg/1UG7KPNHN8EDceFY27zBguZnDM7wTaqnTkpas6Da21Opo4CCOLWlYxSpNGCVibIVic3C1hl8j41N7M8hP7tDwXtg/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

本文为看雪论坛优秀文章

看雪论坛作者ID：jmpcall

  

  

一  

  

必备知识

###   

### 1.1. Makefile基础语法

  

如果还不熟悉Makefile语法，建议先系统的学习一下，特别是以下几点：  

  

(1) Makefile哪些部分包含的是shell语句：  
编译规则中的指令部分  
${shell XX}，var != XX中的XX部分  
$(if …, XX, XX)中的XX部分  
  

(2) 变量展开：  
=（延迟赋值）、:=（立即赋值）、!=（值为shell命令）、?=（条件赋值）、+=（追加）  
  

(3) include：将指定的其它Makefile内容，展开到当前Makefile  
-f/-C：嵌套执行指定（目录中的）Makefile  
执行一个Makefile，并不是从第一行开始执行，而是从指定或默认的编译目标开始执行（位置目标编译规则之前的赋值语句，只在相应变量需要被使用时才会执行），其中，Makefile（包括include内容）中的第一个目标，为默认目标，如果make命令行中没有指定编译目标，则执行默认目标。  
  

(4) 自动推导依赖文件  
  

(5) 根据文件时间戳、中间文件（.d、.cmd），判断依赖更新，决定是否需要重新编译  
  

(6) 重要的内置函数：  
$(wildcard pattern)  
$(patsubst pattern, replacement, text)  
$(strip string)  
$(filter pattern, text)  
$(filter-out pattern, text)  
$(call func, args..)  
…  
  

(7) 自动推导变量：  
$@：编译目标  
$<：依赖列表中的第一个依赖对象  
$^：依赖列表中的所有对象  
$?：依赖文件列表中所有有更新的文件

Makefile教程可以参考以下这2个：  
深入解析Makefile系列：https://zhuanlan.zhihu.com/p/362640343（简约，直指核心）  
跟我一起写makefile（陈皓）：https://blog.csdn.net/whitefish520/article/details/103968609（精典，超级详细）

###   

### 1.2. Kbuild内置函数

  

Linux内核源码包含一套Makefile程序，本文基于Linux-5.2.5内核源码分析，其中包括top Makefile，scripts/目录下的Makefile、Makefile.build、Makefile.lib、Kbuild.include、Makefile.modpost、kconfig/Makefile等,以及其它目录下的很多子Makefile，统称为Kbuild。Kbuild是按照框架设计思路实现的，使得内核自身包含或外部提供的大量驱动模块，只需要按照Kbuild框架的约定，各自提供一个简单的Makefile即可编译。  
  

所以，理解内核或驱动文件的编译过程，其实就是要理解Kbuild这套Makefile程序的实现逻辑，既然是程序，就免不了会定义一些函数，由于很多关键的流程，都使用了$(build)和$(if_changed)，所以以下先单独介绍（本文分析的Makefile内容，来自Linux-5.2.5内核源码）：

  

### 1.2.1. $(build)


◆使用形式：$(Q)$(MAKE) $(build)=xx目录 [编译目标]

![[Pasted image 20240829154814.png]]
  

◆build内部过程  

![[Pasted image 20240829154837.png]]


◆build作用概括  
  

以下是$(build)的使用形式，以及每个部分的作用：

![[Pasted image 20240829154905.png]]

###   

### 1.2.2. $(if_changed)

◆使用形式：$(call if_changed, xx) 
![[Pasted image 20240829154938.png]]


◆if_changed内部过程  

![[Pasted image 20240829154958.png]]


◆if_changed作用概括

![[Pasted image 20240829155013.png]]
  
以下是$(if_changed)的使用形式，及其参数的含义：  
![[Pasted image 20240829155021.png]]
  

二  编译外部模块

###   

### 2.1. 涉及Makefile内容
![[Pasted image 20240829155056.png]]
  
### 2.2. 概要流程
![[Pasted image 20240829155110.png]]
  
###   

### 2.3. 详细流程
![[Pasted image 20240829155124.png]]


三  make menuconfig

###   

### 3.1. 涉及Makefile内容

![[Pasted image 20240829155208.png]]

  

### 3.2. 概要流程
![[Pasted image 20240829155221.png]]
  



###   

### 3.3. 详细流程
![[Pasted image 20240829155235.png]]
  


  

  

四  

  

Make [all/_all/modules]

###   

### 4.1. 涉及Makefile内容

  

make命令行指定all/_all/modules目标，或者不指定目标时，是为了生成vmlinux文件，而vmlinux目标间接依赖prepare目标，且prepare目标编译规则展开内容比较多，所以以下分开介绍：

  

◆vmlinux目标  

![[Pasted image 20240829155255.png]]


  

◆prepare目标  

![[Pasted image 20240829155311.png]]


###   

### 4.2. 概要流程



◆vmlinux目标
![[Pasted image 20240829155328.png]]
  


  

◆prepare目标
![[Pasted image 20240829155355.png]]
  


  

### 4.3. 详细流程

  

◆vmlinux目标  

![[Pasted image 20240829155410.png]]



  

◆vmlinuz目标

![[Pasted image 20240829155424.png]]



  

◆prepare目标  

![[Pasted image 20240829155437.png]]


  

  

五  

  

参考

  

linux Kbuild详解系列：

_https://zhuanlan.zhihu.com/p/362640343_  

跟我一起写makefile：_https://blog.csdn.net/whitefish520/article/details/103968609_

---

_点击文末 阅读原文 获取附件：文中部分图片，根据"执行流程.txt"内容截取，其余已经打包到"图.zip"文件_。

  

  

  

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

**看雪ID：jmpcall**

https://bbs.kanxue.com/user-home-815036.htm

*本文由看雪论坛 jmpcall 原创，转载请注明来自看雪社区

  

[![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458499288&idx=1&sn=b2b9cd6ff7388a8658d254e13c72f9ad&chksm=b18e885286f9014436a590f2531fda167be67e1e227ea395812968e828932bd44eade34b0dbf&scene=21#wechat_redirect)

  

**#** **往期推荐**

1、[在 Windows下搭建LLVM 使用环境](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458500602&idx=1&sn=4bcc2af3c62e79403737ce6eb197effc&chksm=b18e8d7086f9046631a74245c89d5029c542976f21a98982b34dd59c0bda4624d49d1d0d246b&scene=21#wechat_redirect)

2、[深入学习smali语法](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458500599&idx=1&sn=8afbdf12634cbf147b7ca67986002161&chksm=b18e8d7d86f9046b55ff3f6868bd6e1133092b7b4ec7a0d5e115e1ad0a4bd0cb5004a6bb06d1&scene=21#wechat_redirect)

3、[安卓加固脱壳分享](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458500598&idx=1&sn=d783cb03dc6a3c1a9f9465c5053bbbee&chksm=b18e8d7c86f9046a67659f598242acb74c822aaf04529433c5ec2ccff14adeafa4f45abc2b33&scene=21#wechat_redirect)

4、[Flutter 逆向初探](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458500574&idx=1&sn=06344a7d18a72530077fbc8f93a40d8f&chksm=b18e8d5486f904424874d7308e840523ebfb2db20811d99e4b0249d42fa8e38c4e80c3f622c6&scene=21#wechat_redirect)

5、[一个简单实践理解栈空间转移](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458500315&idx=1&sn=19b12ab150dd49325f93ae9d73aef0c4&chksm=b18e8c5186f90547f3b615b160d803a320c103d9d892c7253253db41124ac6993d83d13c5789&scene=21#wechat_redirect)

6、[记一次某盾手游加固的脱壳与修复](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458500165&idx=1&sn=b16710232d3c2799c4177710f0ea6d41&chksm=b18e8ccf86f905d9a0b6c2c40997e9b859241a4d7f798c4aeab21352b0a72b6135afce349262&scene=21#wechat_redirect)

  

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

  

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**球分享**

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**球点赞**

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**球在看**

Read more

Reads 6008

​

Comment

**留言 7**

- Yolo
    
    上海2023年5月11日
    
    Like2
    
    牛逼
    
-  FTD
    
    广东2023年5月12日
    
    Like1
    
    反人类![[恐惧]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
- Dominic
    
    浙江2023年5月12日
    
    Like1
    
    眼花缭乱，谢谢
    
- 龙先生　　　 　　　　　　༽
    
    北京2023年5月12日
    
    Like
    
    截图图片有文档吗？麻烦给一下链接
    
- .a_c.
    
    广西2023年5月11日
    
    Like
    
    学习![[笑脸]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
- mr.zcb
    
    江苏2023年5月11日
    
    Like
    
    不明觉厉
    
- Bruce
    
    江苏2023年5月11日
    
    Like
    
    正需要这个👆🏻
    

已无更多数据

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/1UG7KPNHN8EGLfh77kFmnicd9WOic2ibvhCibFdB4bL4srJCgo2wnvdoXLxpIvAkfCmmcptXZB0qKWMoIP8iaibYN2FA/300?wx_fmt=png&wxfrom=18)

看雪学苑

31510

7

Comment

**留言 7**

- Yolo
    
    上海2023年5月11日
    
    Like2
    
    牛逼
    
-  FTD
    
    广东2023年5月12日
    
    Like1
    
    反人类![[恐惧]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
- Dominic
    
    浙江2023年5月12日
    
    Like1
    
    眼花缭乱，谢谢
    
- 龙先生　　　 　　　　　　༽
    
    北京2023年5月12日
    
    Like
    
    截图图片有文档吗？麻烦给一下链接
    
- .a_c.
    
    广西2023年5月11日
    
    Like
    
    学习![[笑脸]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
- mr.zcb
    
    江苏2023年5月11日
    
    Like
    
    不明觉厉
    
- Bruce
    
    江苏2023年5月11日
    
    Like
    
    正需要这个👆🏻
    

已无更多数据