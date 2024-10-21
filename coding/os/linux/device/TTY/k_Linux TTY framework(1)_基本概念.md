
作者：[wowo](http://www.wowotech.net/author/2 "runangaozhong@163.com") 发布于：2016-9-18 22:55 分类：[TTY子系统](http://www.wowotech.net/sort/tty_framework)

# 1. 前言

由于串口的缘故，TTY是Linux系统中最普遍的一类设备，稍微了解Linux系统的同学，对它都不陌生。尽管如此，相信很少有人能回到这样的问题：TTY到底是什么东西？我们常常挂在嘴边的终端（terminal）、控制台（console）等概念，到底是什么意思？

本文是Linux TTY framework分析文章的第一篇，将带着上述疑问，介绍TTY有关的基本概念，为后续的TTY软件框架的分析，以及Linux serial subsystem的分析，打好基础。

# 2. 终端（terminal）

## 2.1 基本概念

在计算机或者通信系统中，终端是一个电子（或电气）设备，用于向系统输入数据（input），或者将系统接收到的数据显示出来（output），即我们常说的“人机交互设备”。

关于终端最典型的例子，就是电传打字机（Teletype）\[1\]\[2\]----一种基于电报技术的远距离信息传送器械。电传打字机通常由键盘、收发报器和印字机构等组成。发报时，按下某一字符键，就能将该字符的电码信号自动发送到信道（input）；收报时，能自动接收来自信道的电码信号，并打印出相应的字符（output）。

## 2.2 Unix终端

在计算机的世界里，键盘和显示器，是最常用的终端设备，一个用于向计算机输入信息，一个用于显示计算机的输出信息。

在大型机(mainframe)和小型机(minicomputer)的时代里，终端设备和计算机主机都同属一个整体。但到PC时代，情况发生了变化。Unix创始人肯•汤普逊和丹尼斯•里奇想让Unix成为一个多用户系统。多用户系统意味着要给每个用户配置一个终端，每个用户都要有一个显示器、一个键盘。但当时所有的计算机设备(包括显示器)价格都非常昂贵，而且键盘和主机是集成在一起的，根本没有独立的键盘。

最后他们找到了一样东西，那就是ASR33电传打字机。虽然电传打字机的用途是在电报线路上收发电报，但是它也可以作为人与计算机的接口，而且价格低廉。ASR33打字机的键盘用来输入信息，打印纸用来输出信息。所以他们把ASR33电传打字机作为终端，很多个ASR33连接到同一个主机，每个用户都可以在终端输入用户名和密码登录主机。这样他们创造了计算机历史上的第一个真正的多用户系统Unix，而ASR33成为第一个Unix终端。

## 2.3 TTY设备

由上面的介绍可知，第一个Unix终端是一个名字为ASR33的电传打字机，而电传打字机的英文单词为Teletype（或Teletypewritter），缩写为TTY。因此，该终端设备也被称为TTY设备。这就是TTY这个名称的来源，当然，在现在的Unix/Linux系统中，TTY设备已经演变为不同的意义了，后面我们会介绍演变的过程。

注1：读到这里，希望读者再仔细思索一下“设备”的概念。ASR33的电传打字机本身是一个硬件设备，在Unix/Linux系统中，这个硬件设备被抽象为“TTY设备”。

## 2.4 串口终端（Serials Terminal）

早期的TTY终端（这里暂时特指电传打字机），一般通过串口和Unix设备连接的，如下所示：

![[Pasted image 20241021180910.png]]

然后，正如你我所熟知的，我们可以把上面红色部分（电传打字机），替换为任意的具有键盘、显示器、串口的硬件设备（如另一台PC），如下：

![[Pasted image 20241021180922.png]]

因此，对Unix/Linux系统来说，只要是通过串口连接的设备，都可以作为终端设备，因而不再需要关注具体的终端形态。久而久之，终端设备、TTY设备、串口设备等概念，逐渐混在一起，就不再区分了，总结来说，在当今的Linux系统中：

> 1）TTY设备就是终端设备，终端设备就是TTY设备，无需区分。
>
> 2）所有的串口设备都是TTY设备。
>
> 3）当然，除了串口设备，也发展出来了其它形式的TTY设备，例如虚拟终端（VT）、伪终端（Pseudo Terminal）等等，这些概念本文就不展开描述了，后续会使用专门的文章分析。

# 3. 控制台（console）

了解了终端和TTY的概念之后，再来看看另一个比较熟悉的概念：console。

回到Unix系统刚刚支持多用户（2.2小节的描述）的时代，此时的PC有一个自带的、昂贵的终端（自身的键盘、显示器等），另外为了支持多用户，可以通过串口线连接多个TTY终端（Teletype）。为了彰显自带终端崇高的江湖地位，人们称它为console。

当然，“江湖地位”之说，纯属玩笑，不过从console的中文翻译-----控制台，可以看出，自带终端（console）有别于TTY终端的地方如下：

> 1）控制台（console）是昂贵的。
>
> 2）控制台（console）比TTY终端拥有更多的权限，例如用户建立、密码更改、权限分配等等，这也是“控制”的意义所在。
>
> 3）系统的运行日志、出错信息等内容，通常只会输出到控制台（console）终端中，以方便管理员进行“控制”和“管理”。

不过，随着计算机技术的发展、操作系统的改进，控制台（console）终端和普通TTY终端的界限越来越模糊，console能做的事情，普通终端也都能做了。因此，console逐渐退化，以至于在当前的Linux系统中，它仅仅保留了第三点“日志输出”的功能，这就是Linux TTY framework中console的概念（具体可参考后续文章的分析）。

# 4. 参考文章

\[1\] [https://en.wikipedia.org/wiki/Teleprinter](https://en.wikipedia.org/wiki/Teleprinter "https://en.wikipedia.org/wiki/Teleprinter")

\[2\] 电传打字机(Teletype)，[http://baike.baidu.com/view/1773688.htm](http://baike.baidu.com/view/1773688.htm "http://baike.baidu.com/view/1773688.htm")

\[3\] [你真的知道什么是终端吗？](https://www.linuxdashen.com/%E4%BD%A0%E7%9C%9F%E7%9A%84%E7%9F%A5%E9%81%93%E4%BB%80%E4%B9%88%E6%98%AF%E7%BB%88%E7%AB%AF%E5%90%97%EF%BC%9F)

\[4\] [串口通信技术浅析](http://www.wowotech.net/basic_tech/serial_intro.html)

_原创文章，转发请注明出处。蜗窝科技_，[www.wowotech.net](http://www.wowotech.net/tty_framework/tty_concept.html)。

标签: [Linux](http://www.wowotech.net/tag/Linux) [Kernel](http://www.wowotech.net/tag/Kernel) [内核](http://www.wowotech.net/tag/%E5%86%85%E6%A0%B8) [tty](http://www.wowotech.net/tag/tty) [terminal](http://www.wowotech.net/tag/terminal) [终端](http://www.wowotech.net/tag/%E7%BB%88%E7%AB%AF) [console](http://www.wowotech.net/tag/console) [控制台](http://www.wowotech.net/tag/%E6%8E%A7%E5%88%B6%E5%8F%B0)

---

« [TLB flush操作](http://www.wowotech.net/memory_management/tlb-flush.html) | [X-011-UBOOT-使用bootm命令启动kernel(Bubblegum-96平台)](http://www.wowotech.net/x_project/bubblegum_uboot_bootm.html)»

**评论：**

**feixiahn**\
2016-12-06 10:59

您好，我想问一个问题，我们如何修改串口工具登录时的root权限的密码呢？\
能解释一下如何更改吗？

[回复](http://www.wowotech.net/tty_framework/tty_concept.html#comment-4968)

**[wowo](http://www.wowotech.net/)**\
2016-12-06 13:21

@feixiahn：是在PC上修改target的密码吗（最好把问题描述清楚;-）？\
可以用openssl命令：\
openssl passwd -1\
输入两次密码后，会生成MD5加密的字符串，然后替换/etc/shadow中对应的字段就可以了。

[回复](http://www.wowotech.net/tty_framework/tty_concept.html#comment-4971)

**fexiahn**\
2016-12-07 14:39

@wowo：@wowo,谢谢，您的文章写的确实不错的

[回复](http://www.wowotech.net/tty_framework/tty_concept.html#comment-4985)

**fexiahn**\
2016-12-07 14:48

@fexiahn：如果将从代码中/etc/passwd路径改为etc_us/passwd，那么修改用户名的密码时是不是要将/etc/shadow的路径一同修改，我最近在看这部分的资料，没有搞明白/etc/passwd和/etc/shadow之间的关系，麻烦wowo帮忙分析一下，谢谢

[回复](http://www.wowotech.net/tty_framework/tty_concept.html#comment-4986)

**[wowo](http://www.wowotech.net/)**\
2016-12-08 09:23

@fexiahn：用户空间的东西，我也是一知半解，一时半会儿没办法给你讲啊，抱歉哈～～

[回复](http://www.wowotech.net/tty_framework/tty_concept.html#comment-4990)

**笨小孩**\
2016-10-26 21:22

窝窝你好，想问一下，在linux的U-BOOT启动时，有一个CONSOLE = ttySA0 ,在启动后，信息会通过串口打印到屏幕上，那应该这个ttySA0应该代表的是控制台吧，为何在进行其他的操作时，却要求提升用户权限呢？这又是为什么？

[回复](http://www.wowotech.net/tty_framework/tty_concept.html#comment-4783)

**[wowo](http://www.wowotech.net/)**\
2016-10-26 21:50

@笨小孩：是的的，console指定的是控制台。\
对于权限的事情，是这样的：\
用户空间程序是通过文件系统（字符设备，/dev/ttySA0）访问TTY设备的，只要在用户空间牵涉到文件操作，都有权限的概念。

[回复](http://www.wowotech.net/tty_framework/tty_concept.html#comment-4785)

**笨小孩**\
2016-10-27 08:50

@wowo：谢谢窝窝！

[回复](http://www.wowotech.net/tty_framework/tty_concept.html#comment-4787)

**lucifer**\
2016-10-04 15:51

还有就是 类似这种链表操作比较多的。\
static int sysfs_readdir(struct file * filp, void * dirent, filldir_t filldir)\
366{\
.......................................      \
370        struct list_head \*p, *q = &cursor->s_sibling;\
371        ino_t ino;\
372        int i = filp->f_pos;\
373\
374        .........................................*/\
389                default:\
390                        if (filp->f_pos == 2) {\
391                                list_del(q);\
392                                list_add(q, &parent_sd->s_children);\
393                        }\
394                        for (p=q->next; p!= &parent_sd->s_children; p=p->next) {\
395                                struct sysfs_dirent \*next;\
396                                const char * name;\
397                                int len;\
398\
399                                next = list_entry(p, struct sysfs_dirent,\
400                                                   s_sibling);\
401                                if (!next->s_element)\
402                                        continue;\
403\
404                                name = sysfs_get_name(next);\
405                                len = strlen(name);\
406                                if (next->s_dentry)\
407                                        ino = next->s_dentry->d_inode->i_ino;\
408                                else\
409                                        ino = iunique(sysfs_sb, 2);\
410\
411                                if (filldir(dirent, name, len, filp->f_pos, ino,\
412                                                 dt_type(next)) \< 0)\
413                                        return 0;\
414\
415                                list_del(q);\
416                                list_add(q, p);\
417                                p = q;\
418                                filp->f_pos++;\
419                        }\
420        }\
421        return 0;\
422}

=====\
我读起来迷迷糊糊的

[回复](http://www.wowotech.net/tty_framework/tty_concept.html#comment-4645)

**lucifer**\
2016-10-04 15:50

请教大奖一个基础的C语言的问题或者说对指针的理解吧\
我在2.6.10中看到\
#define subsys_set_kset(obj,\_subsys)  (obj)->subsys.kset.kobj.kset = &(\_subsys).kset\
还有down_write(&kobj->kset->subsys->rwsem);  \
怎么理解这么多的->或者. --- 什么时候该用  ‘ .’\
什么时候用->

[回复](http://www.wowotech.net/tty_framework/tty_concept.html#comment-4644)

**[wowo](http://www.wowotech.net/)**\
2016-10-04 16:28

@lucifer：->是指针，例如struct device \*p;p->name;\
.是变量，例如struct device aa; aa.name;

[回复](http://www.wowotech.net/tty_framework/tty_concept.html#comment-4647)

**[electrlife](http://www.wowotech.net/)**\
2016-09-22 15:11

这篇文章来的太及时了，前两天正准备部关于tty的相关东西，有一些概念还没有理清楚：\
terminal console vt tty pseudo terminal等，再加上control terminal, session等\
纠缠在一起！

[回复](http://www.wowotech.net/tty_framework/tty_concept.html#comment-4596)

**[维尼](http://www.wowotech.net/)**\
2016-09-20 16:35

内核 /dev下已经有好多TTY了  tty又演变成驱动了

[回复](http://www.wowotech.net/tty_framework/tty_concept.html#comment-4579)

**熊猫盼盼**\
2016-09-21 15:41

@维尼：这段时间也一直再看tty相关的东西，起因是在弄lxc/docker时，不明白ｌｘｃ/docer的ｃｏｎｓｏｌｅ/tty 是怎么实现的。 正如ｗｏｗｏ所说的，ｔｔｙ是unix/linux 下一个既熟悉又陌生的东西。　非常期待ｗｏｗｏ后续的文章，　如果能把　设备下的文件 /dev/console /dev/tty /dev/ttyN /dev/pty 之间的关系　以及　实现讲清楚，　那就更好了！

[回复](http://www.wowotech.net/tty_framework/tty_concept.html#comment-4582)

**[wowo](http://www.wowotech.net/)**\
2016-09-21 17:17

@熊猫盼盼：多谢建议，为我写后续的文章指了一条明路:-)

[回复](http://www.wowotech.net/tty_framework/tty_concept.html#comment-4584)

**wink**\
2016-11-20 21:22

@熊猫盼盼：我也是啊，

[回复](http://www.wowotech.net/tty_framework/tty_concept.html#comment-4897)

**发表评论：**

昵称

邮件地址 (选填)

个人主页 (选填)

![](http://www.wowotech.net/include/lib/checkcode.php)

- ### 站内搜索

  蜗窝站内  互联网

- ### 功能

  [留言板\
  ](http://www.wowotech.net/message_board.html)[评论列表\
  ](http://www.wowotech.net/?plugin=commentlist)[支持者列表\
  ](http://www.wowotech.net/support_list)

- ### 最新评论

  - Shiina\
    [一个电路（circuit）中，由于是回路，所以用电势差的概念...](http://www.wowotech.net/basic_subject/voltage.html#8926)
  - Shiina\
    [其中比较关键的点是相对位置概念和点电荷的静电势能计算。](http://www.wowotech.net/basic_subject/voltage.html#8925)
  - leelockhey\
    [你这是哪个内核版本](http://www.wowotech.net/pm_subsystem/generic_pm_architecture.html#8924)
  - ja\
    [@dream：我看完這段也有相同的想法，引用 @dream ...](http://www.wowotech.net/kernel_synchronization/spinlock.html#8922)
  - 元神高手\
    [围观首席power managerment专家](http://www.wowotech.net/pm_subsystem/device_driver_pm.html#8921)
  - 十七\
    [内核空间的映射在系统启动时就已经设定好，并且在所有进程的页表...](http://www.wowotech.net/process_management/context-switch-arch.html#8920)

- ### 文章分类

  - [Linux内核分析(25)](http://www.wowotech.net/sort/linux_kenrel) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=4)
    - [统一设备模型(15)](http://www.wowotech.net/sort/device_model) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=12)
    - [电源管理子系统(43)](http://www.wowotech.net/sort/pm_subsystem) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=13)
    - [中断子系统(15)](http://www.wowotech.net/sort/irq_subsystem) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=14)
    - [进程管理(31)](http://www.wowotech.net/sort/process_management) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=15)
    - [内核同步机制(26)](http://www.wowotech.net/sort/kernel_synchronization) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=16)
    - [GPIO子系统(5)](http://www.wowotech.net/sort/gpio_subsystem) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=17)
    - [时间子系统(14)](http://www.wowotech.net/sort/timer_subsystem) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=18)
    - [通信类协议(7)](http://www.wowotech.net/sort/comm) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=20)
    - [内存管理(31)](http://www.wowotech.net/sort/memory_management) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=21)
    - [图形子系统(2)](http://www.wowotech.net/sort/graphic_subsystem) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=23)
    - [文件系统(5)](http://www.wowotech.net/sort/filesystem) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=26)
    - [TTY子系统(6)](http://www.wowotech.net/sort/tty_framework) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=27)
  - [u-boot分析(3)](http://www.wowotech.net/sort/u-boot) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=25)
  - [Linux应用技巧(13)](http://www.wowotech.net/sort/linux_application) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=3)
  - [软件开发(6)](http://www.wowotech.net/sort/soft) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=1)
  - [基础技术(13)](http://www.wowotech.net/sort/basic_tech) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=6)
    - [蓝牙(16)](http://www.wowotech.net/sort/bluetooth) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=10)
    - [ARMv8A Arch(15)](http://www.wowotech.net/sort/armv8a_arch) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=19)
    - [显示(3)](http://www.wowotech.net/sort/display) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=22)
    - [USB(1)](http://www.wowotech.net/sort/usb) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=28)
  - [基础学科(10)](http://www.wowotech.net/sort/basic_subject) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=7)
  - [技术漫谈(12)](http://www.wowotech.net/sort/tech_discuss) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=8)
  - [项目专区(0)](http://www.wowotech.net/sort/project) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=9)
    - [X Project(28)](http://www.wowotech.net/sort/x_project) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=24)

- ### 随机文章

  - [Linux内核同步机制之（七）：RCU基础](http://www.wowotech.net/kernel_synchronization/rcu_fundamentals.html)
  - [GIC驱动代码分析（废弃）](http://www.wowotech.net/irq_subsystem/gic-irq-chip-driver.html)
  - [CFS任务的负载均衡（load balance）](http://www.wowotech.net/process_management/load_balance_detail.html)
  - [Windows系统结合MinGW搭建软件开发环境](http://www.wowotech.net/soft/6.html)
  - [Linux内核中的GPIO系统之（3）：pin controller driver代码分析](http://www.wowotech.net/gpio_subsystem/pin-controller-driver.html)

- ### 文章存档

  - [2024年2月(1)](http://www.wowotech.net/record/202402)
  - [2023年5月(1)](http://www.wowotech.net/record/202305)
  - [2022年10月(1)](http://www.wowotech.net/record/202210)
  - [2022年8月(1)](http://www.wowotech.net/record/202208)
  - [2022年6月(1)](http://www.wowotech.net/record/202206)
  - [2022年5月(1)](http://www.wowotech.net/record/202205)
  - [2022年4月(2)](http://www.wowotech.net/record/202204)
  - [2022年2月(2)](http://www.wowotech.net/record/202202)
  - [2021年12月(1)](http://www.wowotech.net/record/202112)
  - [2021年11月(5)](http://www.wowotech.net/record/202111)
  - [2021年7月(1)](http://www.wowotech.net/record/202107)
  - [2021年6月(1)](http://www.wowotech.net/record/202106)
  - [2021年5月(3)](http://www.wowotech.net/record/202105)
  - [2020年3月(3)](http://www.wowotech.net/record/202003)
  - [2020年2月(2)](http://www.wowotech.net/record/202002)
  - [2020年1月(3)](http://www.wowotech.net/record/202001)
  - [2019年12月(3)](http://www.wowotech.net/record/201912)
  - [2019年5月(4)](http://www.wowotech.net/record/201905)
  - [2019年3月(1)](http://www.wowotech.net/record/201903)
  - [2019年1月(3)](http://www.wowotech.net/record/201901)
  - [2018年12月(2)](http://www.wowotech.net/record/201812)
  - [2018年11月(1)](http://www.wowotech.net/record/201811)
  - [2018年10月(2)](http://www.wowotech.net/record/201810)
  - [2018年8月(1)](http://www.wowotech.net/record/201808)
  - [2018年6月(1)](http://www.wowotech.net/record/201806)
  - [2018年5月(1)](http://www.wowotech.net/record/201805)
  - [2018年4月(7)](http://www.wowotech.net/record/201804)
  - [2018年2月(4)](http://www.wowotech.net/record/201802)
  - [2018年1月(5)](http://www.wowotech.net/record/201801)
  - [2017年12月(2)](http://www.wowotech.net/record/201712)
  - [2017年11月(2)](http://www.wowotech.net/record/201711)
  - [2017年10月(1)](http://www.wowotech.net/record/201710)
  - [2017年9月(5)](http://www.wowotech.net/record/201709)
  - [2017年8月(4)](http://www.wowotech.net/record/201708)
  - [2017年7月(4)](http://www.wowotech.net/record/201707)
  - [2017年6月(3)](http://www.wowotech.net/record/201706)
  - [2017年5月(3)](http://www.wowotech.net/record/201705)
  - [2017年4月(1)](http://www.wowotech.net/record/201704)
  - [2017年3月(8)](http://www.wowotech.net/record/201703)
  - [2017年2月(6)](http://www.wowotech.net/record/201702)
  - [2017年1月(5)](http://www.wowotech.net/record/201701)
  - [2016年12月(6)](http://www.wowotech.net/record/201612)
  - [2016年11月(11)](http://www.wowotech.net/record/201611)
  - [2016年10月(9)](http://www.wowotech.net/record/201610)
  - [2016年9月(6)](http://www.wowotech.net/record/201609)
  - [2016年8月(9)](http://www.wowotech.net/record/201608)
  - [2016年7月(5)](http://www.wowotech.net/record/201607)
  - [2016年6月(8)](http://www.wowotech.net/record/201606)
  - [2016年5月(8)](http://www.wowotech.net/record/201605)
  - [2016年4月(7)](http://www.wowotech.net/record/201604)
  - [2016年3月(5)](http://www.wowotech.net/record/201603)
  - [2016年2月(5)](http://www.wowotech.net/record/201602)
  - [2016年1月(6)](http://www.wowotech.net/record/201601)
  - [2015年12月(6)](http://www.wowotech.net/record/201512)
  - [2015年11月(9)](http://www.wowotech.net/record/201511)
  - [2015年10月(9)](http://www.wowotech.net/record/201510)
  - [2015年9月(4)](http://www.wowotech.net/record/201509)
  - [2015年8月(3)](http://www.wowotech.net/record/201508)
  - [2015年7月(7)](http://www.wowotech.net/record/201507)
  - [2015年6月(3)](http://www.wowotech.net/record/201506)
  - [2015年5月(6)](http://www.wowotech.net/record/201505)
  - [2015年4月(9)](http://www.wowotech.net/record/201504)
  - [2015年3月(9)](http://www.wowotech.net/record/201503)
  - [2015年2月(6)](http://www.wowotech.net/record/201502)
  - [2015年1月(6)](http://www.wowotech.net/record/201501)
  - [2014年12月(17)](http://www.wowotech.net/record/201412)
  - [2014年11月(8)](http://www.wowotech.net/record/201411)
  - [2014年10月(9)](http://www.wowotech.net/record/201410)
  - [2014年9月(7)](http://www.wowotech.net/record/201409)
  - [2014年8月(12)](http://www.wowotech.net/record/201408)
  - [2014年7月(6)](http://www.wowotech.net/record/201407)
  - [2014年6月(6)](http://www.wowotech.net/record/201406)
  - [2014年5月(9)](http://www.wowotech.net/record/201405)
  - [2014年4月(9)](http://www.wowotech.net/record/201404)
  - [2014年3月(7)](http://www.wowotech.net/record/201403)
  - [2014年2月(3)](http://www.wowotech.net/record/201402)
  - [2014年1月(4)](http://www.wowotech.net/record/201401)

[![订阅Rss](http://www.wowotech.net/content/templates/default/images/rss.gif)](http://www.wowotech.net/rss.php "RSS订阅")

Copyright @ 2013-2015 [蜗窝科技](http://www.wowotech.net/ "wowotech") All rights reserved. Powered by [emlog](http://www.emlog.net/ "采用emlog系统")
