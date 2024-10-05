作者：[wowo](http://www.wowotech.net/author/2 "runangaozhong@163.com") 发布于：2014-12-6 15:40 分类：[ARMv8A Arch](http://www.wowotech.net/sort/armv8a_arch)

2013年9月11日（是的，911），在ARM公司发布UEFI 64-bit之后，ARM社区release了ARMv8A版本的ARM Architecture Reference Manual（我已经下载，感兴趣的同学可以找我要）。在[release note](http://community.arm.com/groups/processors/blog/2013/10/09/arm-architecture-reference-manual-for-armv8-a-64-bit-publicly-released)中，作者给出了这样一个设问句：“Why develop ARMv8-A?”。本文也效仿一下，以自问自答的形式，说明为什么会在博客中增加这样一个分类，以及期望达成的目的。

添加“ARMv8A Architecture”这个分类的原因包括：

1）自从ARM64bit（即ARMv8-A，也称作AARCH64）发布以来，linuxer、forion、wowo等同学，都表现除了极大的兴趣，希望能对它有个比较系统的了解。

2）未来的一段时间内，wowotech主要会focus在Linux内核分析上，由于我们的工作性质，一般都会基于ARM平台分析。kernel的很多模块，如进程管理、内存管理、中断管理、timer管理、cpuidle、cpufreq、spinlock等等，和体系结构的相关性非常大。以前的做法，是在相应的分析文章中，穿插介绍体系结构的内容。但这种做法比较零碎，不成体系，也很难理解。因而希望将ARM体系结构先关的知识，汇整在一起。

3）近些年，ARM发展很快，从最初的ARM7/ARM9（v4T或v5E），到后来的ARM11（v6），以及现在比较流行的Cortex系列（v7/v8/v8-A）。虽然ARM基本架构还维持不变，但却有非常多的新知识点。而我们这些从ARM7开始做起的“老人”，已经有很多年没有进行头脑风暴了。

4）ARMv8-A作为ARM第一款64bit的架构，在增加64bit的诸多特性的同时，保持了向前兼容，那么拿它做分析对象，就再好不过了。

5）从另一个角度看，当初的ARM7还是以“单片机”的形式出现的，在那个年代，没有强大的OS、没有复杂的软件抽象和软件分工，这就要求嵌入式软件开发人员有非常扎实的硬件功底，因而要非常熟悉ARM体系结构，这也是《ARM体系结构和编程》畅销的原因。但现在就不同了，一个Android下的Linux driver开发者，甚至不需要知道体系结构的存在……那这种现象好不好呢？从产业的角度看，好；从技术的角度看，不好！

6）wowotech的理念是享受技术，那么怎么能放过体系结构这一块肥肉呢？

理想虽然很丰满，但现实却很骨感，我们没有那么多时间，系统的去理解、分析并分享这些知识。但做总比不做好，这个分类下面的文章的基调是：Linux内核分析到某个模块（如cpuidle），需要用到体系结构的知识，就在这里记录下来。希望能积少成多，或许有一天，可以出一版新的《ARM体系结构与编程》，说不定呢！

标签: [Architecture](http://www.wowotech.net/tag/Architecture) [arm64](http://www.wowotech.net/tag/arm64) [aarch64](http://www.wowotech.net/tag/aarch64) [armv8-a](http://www.wowotech.net/tag/armv8-a)

---

« [Linux时间子系统之（十六）：clockevent](http://www.wowotech.net/timer_subsystem/clock-event.html) | [kobject在字符设备中的使用](http://www.wowotech.net/116.html)»

**评论：**

**Caeser**  
2023-09-06 10:59

写的很清晰 非常赞呀

[回复](http://www.wowotech.net/armv8a_arch/why_armv8a_arch.html#comment-8814)

**Roy**  
2022-12-08 15:45

拜阅  
linuxer、forion、wowo等同学，都表现（除）出了极大的兴趣， //除 ->出  
建议修改  
从最初的ARM7（v4），ARM9/9E（v5），到后来的ARM11（v6），

[回复](http://www.wowotech.net/armv8a_arch/why_armv8a_arch.html#comment-8714)

**我是来点赞的**  
2021-12-04 22:21

赞

[回复](http://www.wowotech.net/armv8a_arch/why_armv8a_arch.html#comment-8389)

**Even**  
2016-06-21 15:18

一个Android下的Linux driver开发者，甚至需要知道体系结构的存在。  
wowo，这句是不是少了一个 不 字？

[回复](http://www.wowotech.net/armv8a_arch/why_armv8a_arch.html#comment-4125)

**[wowo](http://www.wowotech.net/)**  
2016-06-21 15:50

@Even：多谢提醒，话说过去了好久，我自己都忘了当初想说的是什么了。呵呵

[回复](http://www.wowotech.net/armv8a_arch/why_armv8a_arch.html#comment-4126)

**lihua**  
2016-05-24 22:48

怎么我（前几天）下载的ARM® Architecture Reference Manual ARMv8, for ARMv8-A architecture profile (共5634页) 没有“Why develop ARMv8-A?”这一句呢^_^...博主，感谢你发表这些文章，我学到了好多。

[回复](http://www.wowotech.net/armv8a_arch/why_armv8a_arch.html#comment-3970)

**[linuxer](http://www.wowotech.net/)**  
2016-05-25 08:41

@lihua：不在ARM ARM文档中，在release ARM ARM文档的时候，Andrew N. Sloss写了一篇博客，其中描述了Why develop ARMv8-A这个问题。

[回复](http://www.wowotech.net/armv8a_arch/why_armv8a_arch.html#comment-3972)

**[wowo](http://www.wowotech.net/)**  
2016-05-25 08:44

@lihua：文章中有一个链接：在release note中……

[回复](http://www.wowotech.net/armv8a_arch/why_armv8a_arch.html#comment-3973)

**[superm](http://www.wowotech.net/)**  
2015-10-30 10:43

我也从事ARMv8芯片的研发，希望后面能有更多的交流和合作。

[回复](http://www.wowotech.net/armv8a_arch/why_armv8a_arch.html#comment-2896)

**[linuxer](http://www.wowotech.net/)**  
2014-12-08 12:33

wowo同学，华为已经出了64 bit的ARM处理器，你们要加快啊

[回复](http://www.wowotech.net/armv8a_arch/why_armv8a_arch.html#comment-855)

**wowo**  
2014-12-08 13:10

@linuxer：呵呵，这种事儿我也急不来啊~~~

[回复](http://www.wowotech.net/armv8a_arch/why_armv8a_arch.html#comment-856)

**[pingchangxin](http://www.wowotech.net/)**  
2014-12-08 16:26

@wowo：linuxer同鞋在哪家公司啊？

[回复](http://www.wowotech.net/armv8a_arch/why_armv8a_arch.html#comment-859)

**[tigger](http://www.wowotech.net/)**  
2014-12-08 18:05

@pingchangxin：这种问题算是比较隐私的啦，最好不要问啦。

[回复](http://www.wowotech.net/armv8a_arch/why_armv8a_arch.html#comment-861)

**[linuxer](http://www.wowotech.net/)**  
2014-12-08 18:10

@pingchangxin：我在广州的一家小公司供职，公司默默无闻。我猜想你问这个问题是看我提及华为，我不是华为的，虽然我有些向往华为海思

[回复](http://www.wowotech.net/armv8a_arch/why_armv8a_arch.html#comment-862)

**[pingchangxin](http://www.wowotech.net/)**  
2014-12-08 09:59

很赞成作总比不做好，谢谢你们的分享！

[回复](http://www.wowotech.net/armv8a_arch/why_armv8a_arch.html#comment-854)

**wowo**  
2014-12-08 13:11

@pingchangxin：多谢，我们会加油的。

[回复](http://www.wowotech.net/armv8a_arch/why_armv8a_arch.html#comment-857)

**发表评论：**

 昵称

 邮件地址 (选填)

 个人主页 (选填)

![](http://www.wowotech.net/include/lib/checkcode.php) 

- ### 站内搜索
    
       
     蜗窝站内  互联网
    
- ### 功能
    
    [留言板  
    ](http://www.wowotech.net/message_board.html)[评论列表  
    ](http://www.wowotech.net/?plugin=commentlist)[支持者列表  
    ](http://www.wowotech.net/support_list)
- ### 最新评论
    
    - Shiina  
        [一个电路（circuit）中，由于是回路，所以用电势差的概念...](http://www.wowotech.net/basic_subject/voltage.html#8926)
    - Shiina  
        [其中比较关键的点是相对位置概念和点电荷的静电势能计算。](http://www.wowotech.net/basic_subject/voltage.html#8925)
    - leelockhey  
        [你这是哪个内核版本](http://www.wowotech.net/pm_subsystem/generic_pm_architecture.html#8924)
    - ja  
        [@dream：我看完這段也有相同的想法，引用 @dream ...](http://www.wowotech.net/kernel_synchronization/spinlock.html#8922)
    - 元神高手  
        [围观首席power managerment专家](http://www.wowotech.net/pm_subsystem/device_driver_pm.html#8921)
    - 十七  
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
    
    - [DRAM 原理 2 ：DRAM Memory Organization](http://www.wowotech.net/basic_tech/309.html)
    - [linux kernel内存回收机制](http://www.wowotech.net/memory_management/233.html)
    - [蜗窝微信群问题整理](http://www.wowotech.net/tech_discuss/question_set_1.html)
    - [DRAM 原理 4 ：DRAM Timing](http://www.wowotech.net/basic_tech/330.html)
    - [futex基础问答](http://www.wowotech.net/kernel_synchronization/futex.html)
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