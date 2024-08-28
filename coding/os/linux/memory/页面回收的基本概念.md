# [蜗窝科技](http://www.wowotech.net/)

### 慢下来，享受技术。

[![](http://www.wowotech.net/content/uploadfile/201401/top-1389777175.jpg)](http://www.wowotech.net/)

- [博客](http://www.wowotech.net/)
- [项目](http://www.wowotech.net/sort/project)
- [关于蜗窝](http://www.wowotech.net/about.html)
- [联系我们](http://www.wowotech.net/contact_us.html)
- [支持与合作](http://www.wowotech.net/support_us.html)
- [登录](http://www.wowotech.net/admin)

﻿

## 

作者：[linuxer](http://www.wowotech.net/author/3 "linuxer") 发布于：2017-8-25 19:01 分类：[内存管理](http://www.wowotech.net/sort/memory_management)

本文主要介绍了一些page reclaim机制中的基本概念。这份文档其实也可以看成阅读ULK第17章第一小节的一个读书笔记。虽然ULK已经读了很多遍，不过每一遍还是觉得有收获。Linux内核虽然不断在演进，但是页面回收的基本概念是不变的，所以ULK仍然值得内核发烧友仔细品味。

一、什么是page frame reclaiming？

在用户进程的内存使用上，Linux内核并没有严格的限制，其实思路是：当系统负荷小的时候，内存都是位于各种cache中，以便提高性能。当系统负荷重（进程数目非常多）的时候，cache中的内存被收回，然后用于进程地址空间的创建和映射。在这样思路的指导下，一开始，内核大手大脚，一直从伙伴系统的free list上分配page frame给用户进程或者各种kernel cache使用，但是系统内存终归是有限的，当伙伴系统的空闲内存下降到一定的水位的时候，系统会从用户进程或者kernel cache中回收page frame到伙伴系统，以便满足新的内存分配的需求，这个过程就是page frame reclaiming。一言以蔽之，page frame reclaiming就是保证系统的空闲内存在一个指定的水位之上。

二、什么时候启动page frame reclaiming？

能不能等到空闲内存用尽之后才启动page frame reclaiming呢？不行，因为在page frame reclaiming的过程中也会消耗内存（例如在页面回收过程中，我们可能会将page frame的数据交换到磁盘，因此需要分配buffer head数据结构，完成IO操作），因此我们必须在伙伴系统中保持一定的水位，以便让page frame reclaiming机制正常工作，以便回收更多的内存，让系统运转下去，否则系统会crash掉。

三、哪些场景可以触发page frame reclaiming？

当分配内存失败的时候触发页面回收是最直观的想法，然而内核的场景没有那么简单，例如在中断上下文中分配内存，这时候我们不能触发page frame reclaiming，因为这时候我们不能阻塞当前进程。此外，有些内存分配的场景是发生在持锁之后（以便对某些资源进行排他性的访问），这时候，我们也不能激活IO操作，进行内存回收。

因此，总结起来，内核在内存回收的思路就是：系统在分配内存的时候就进行检查，如果有需要就唤醒kswapd，让系统的空闲内存的数量保持在一个水平之上，以免将自己逼入绝境。如果没有办法，的确进入了绝境（分配内存失败），那么就直接触发页面回收。具体的场景包括：

1、synchronous page reclaim，即当遭遇分配内存失败的时候，一言不合，然后直接调用page frame reclaiming进行回收。例如：在分配page buffer的时候（alloc_page_buffers），如果分配不成功，直接调用free_more_memory进行内存回收。或者在调用__alloc_pages的时候，如果分配不成功，直接调用try_to_free_pages进行内存回收。当然，其实free_more_memory也是调用try_to_free_pages来实现页面回收的。

2、Suspend to disk（Hibernation）的场景。系统hibernate的时候需要大量内存，这时候会调用shrink_all_memory来回收指定数目的page frame。

3、kswapd场景。Kswapd是一个专门用来进行页面回收的内核线程。

4、slab内存分配器会定时的queue work到system_wq上去，从而会周期性的调用cache_reap来回收slab上的空闲内存。

四、页面回收的策略为何？

首先我们需要对page frame进行分类，主要分成4类：

1、 没有办法回收的page frame。包括空闲页面（已经在free list上面，也就不需要劳驾page frame reclaim机制了）、保留页面（设定了PG_reserved，例如内核正文段、数据段等等）、内核动态分配的page frame、用户进程的内核栈上的page frame、临时性的被锁定的page frame（即设定了PG_locked flag，例如在进行磁盘IO的时候）、mlocked page frame（有VM_LOCKED标识的VMA）

2、 可以交换到磁盘的page frame（swappable）。用户空间的匿名映射页面（用户进程堆和栈上的page frame）、tmpfs的page frame。

3、 可以同步到磁盘的page frame（syncable）。用户空间的文件映射（file mapped）页面，page cache中的page frame（其内容会对应到某个磁盘文件），block device的buffered cache、disk cache中的page frame（例如inode cache）

4、 可以直接释放的page frame。各种内存cache（例如 slab内存分配器）中还没有使用的那些page frame、没有使用的dentry cache。

上面的第二类和第三类有些类似， 其page frame都有后备文件或者磁盘，不过我们可以这么区分。Swappable的page frame，其数据的最终地点就是内存，其后备文件或者磁盘只是为了延伸内存的容量。Syncable的page frame，其数据的最终地点就是磁盘，内存只不过是为了加快速度而已。因此，当回收swappable的页面的时候，需要将page frame的数据保存到后备的磁盘或者文件。而当回收syncable页面的时候，要看page frame是否是dirty的，如果dirty，则需要磁盘操作，否则可以直接回收。

圈定进行页面回收的那些候选page frame很容易（即上面的2、3、4类型的page frame），但是怎么考虑页面回收的先后顺序呢？Linux内核设定的基本规则如下：

1、 尽量不要修改page table。例如回收各种没有使用的内核cache的时候，我们直接回收，根本不需要修改页表项。而用户空间进程的页面回收往往涉及将对应的pte条目修改为无效状态。

2、 除非调用mlock将page锁定，否则所有的用户空间对应的page frame都应该可以被回收。

3、 如果一个page frame被多个进程共享，那么我们需要清除所有的pte entry，之后才能回收该页面。

4、 不要回收那些最近使用（访问）过的page frame，或者说优先回收那些最近没有访问的page frame。

5、 尽量先回收那些不需要磁盘IO操作的page frame。

原创文章，转发请注明出处。蜗窝科技，[www.wowotech.net](http://www.wowotech.net/memory_management/page_reclaim_basic.html)。

标签: [页面回收](http://www.wowotech.net/tag/%E9%A1%B5%E9%9D%A2%E5%9B%9E%E6%94%B6)

[![](http://www.wowotech.net/content/uploadfile/201605/ef3e1463542768.png)](http://www.wowotech.net/support_us.html)

« [Linux reset framework](http://www.wowotech.net/pm_subsystem/reset_framework.html) | [/proc/meminfo分析（一）](http://www.wowotech.net/memory_management/meminfo_1.html)»

**评论：**

**focus**  
2022-06-25 15:26

清晰。

[回复](http://www.wowotech.net/memory_management/page_reclaim_basic.html#comment-8632)

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
    
    - ja  
        [@dream：我看完這段也有相同的想法，引用 @dream ...](http://www.wowotech.net/kernel_synchronization/spinlock.html#8922)
    - 元神高手  
        [围观首席power managerment专家](http://www.wowotech.net/pm_subsystem/device_driver_pm.html#8921)
    - 十七  
        [内核空间的映射在系统启动时就已经设定好，并且在所有进程的页表...](http://www.wowotech.net/process_management/context-switch-arch.html#8920)
    - lw  
        [sparse模型和disconti模型没看出来有什么本质区别...](http://www.wowotech.net/memory_management/memory_model.html#8919)
    - 肥饶  
        [一个没设置好就出错](http://www.wowotech.net/linux_kenrel/516.html#8918)
    - orange  
        [点赞点赞，对linuxer的文章总结到位](http://www.wowotech.net/device_model/dt-code-file-struct-parse.html#8917)
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
    
    - [Linux PWM framework(1)_简介和API描述](http://www.wowotech.net/comm/pwm_overview.html)
    - [Linux电源管理(3)_Generic PM之Reboot过程](http://www.wowotech.net/pm_subsystem/reboot.html)
    - [mellanox的网卡故障分析](http://www.wowotech.net/linux_kenrel/485.html)
    - [图解slub](http://www.wowotech.net/memory_management/426.html)
    - [MinGW下安装man工具包](http://www.wowotech.net/linux_application/8.html)
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