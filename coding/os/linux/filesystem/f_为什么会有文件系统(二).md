
作者：[驴肉火烧](http://www.wowotech.net/author/529 "13608003927@163.com") 发布于：2017-6-27 21:06 分类：[文件系统](http://www.wowotech.net/sort/filesystem)

距我将全套盗墓笔记成功保存在8MB空间里已经过去了19天58分钟32秒，我渐渐发觉更高、更快、更强的绝不限于奥运精神，也充分体现了人类贪婪的本质，无尽的需求催生出这光怪陆离的大千世界。

就在今天下午，我得到一个通知，要么继续使用连续的存储空间，但是只能有4MB，要么去使用不连续的存储空间，总量可以仍然是8MB，那一刻，我的内心反而是平静的，因为我知道，这就是现实，一个不够优秀的系统是无法满足各种刁钻的需求的，并且我并不想丢掉一半的盗墓笔记，所以我必须使用不连续的存储空间，一个不算坏的消息是，就算是不连续，但是每块最小也有2048字节，并且连续的存储空间是2048字节对齐的，还有什么好说的，撸起袖子加油干，这很2017。

当时我的脑海中，浮现出了星空的图像，天顶中每颗闪烁的星代表的就是一段文字，我要怎么将它们串在一起呢？我想，首先要解决的是识别问题，即眼前的这颗星属于哪本书？是的，我需要星的索引信息，每条索引信息对应着一段可存储的空间，记录空间在硬盘中的偏移，长度，内容是属于哪本书，对应内容在书内的偏移，这样通过索引信息就可以在硬盘中找到存储着的盗墓笔记的片段了，于是有了如下的设计，

![[Pasted image 20241018115637.png]]

book_name用来存储书名，hd_ofs存储这段存储空间在硬盘中的偏移，file_ofs存储这段存储空间存储的内容在书中的偏移，chunk_len存储这段存储空间的长度，看起来是能工作的，那么这样的设计够不够好呢，答案显然是需要拿出工匠精神再来打磨一下了。

book_name，这里看起来很糟糕，如果书名很长则无法存储完整，如果书名很短则浪费了存储空间，这里真的需要存储一个书名吗？按照我的需求，盗墓笔记全套是8本书，那么第一本书，我这里记录1即可，依次则是2,3,4,...，我只需要数字就可以进行区分，于是新的设计出现了

![[Pasted image 20241018115649.png]]

但是，新的问题又出现了，我能够通过一个个的index对象找到数据块，但是我该如何找到这些index对象呢？由于每个index对象占用12字节，那么将index搓堆存在一个只存储index的数据块内，那么一个块能存170个index，就像下面这样

![[Pasted image 20241018115700.png]]

很好，现在有了一个index块，那么170个index最多只能映射(170 * 2048)字节(340KB)的内容，可我要存储的盗墓笔记不止这么点内容，所以还需要更多的index块

![[Pasted image 20241018115709.png]]

很好，现在有了更多的index块，我能通过index找到想要看的内容，但是index块也是不连续的，我要如何找到index块在哪里呢？其实，我对之前每个数据块填充170个index对象已经感觉难受了，因为170个index对象只使用了2040字节，这样一个数据块就有8字节的浪费，如果这8字节用来存储另一个index块在硬盘中的偏移位置，那么index块之间就能串联在一起，而我要做的就是找到那个入口

![[Pasted image 20241018115757.png]]

经过了两顿烧烤的谈判，我终于赢得了硬盘第1024个数据块的永久使用权，于是第1024数据块就成为了串起整部盗墓笔记的那个入口

(未完待续)

---

« [linux内核中的GPIO系统之（4）：pinctrl驱动的理解和总结](http://www.wowotech.net/gpio_subsystem/pinctrl-driver-summary.html) | [Linux DMA Engine framework(3)\_dma controller驱动](http://www.wowotech.net/linux_kenrel/dma_controller_driver.html)»

**评论：**

**xinhe**\
2022-09-28 09:12

mark, 很顺理成章的讲解

[回复](http://www.wowotech.net/filesystem/396.html#comment-8677)

**JIMMI**\
2020-09-01 16:28

搬小马扎坐等3上线

[回复](http://www.wowotech.net/filesystem/396.html#comment-8106)

**单手骑车**\
2018-12-20 15:36

很通俗易懂，为啥后面没有更新了呢？好可惜

[回复](http://www.wowotech.net/filesystem/396.html#comment-7092)

**tt**\
2018-11-14 18:40

写的很好，期待您继续分享

[回复](http://www.wowotech.net/filesystem/396.html#comment-7038)

**Nicole**\
2018-07-26 10:29

期待接下来的精彩故事

[回复](http://www.wowotech.net/filesystem/396.html#comment-6853)

**Anyly**\
2018-06-27 17:45

距离为什么会有文件系统二已过去整整一年，这么精彩的剧难道不继续拍下去了？

[回复](http://www.wowotech.net/filesystem/396.html#comment-6822)

**spinningwatt**\
2018-06-13 23:02

写的很好，很想看接下来的故事。

[回复](http://www.wowotech.net/filesystem/396.html#comment-6800)

**向往**\
2017-09-07 20:42

蜗窝，我想问一下nand 或nor上跑的文件系统，比如jffs2、或者ubi走block调度层吗？另外它们有swapd机制吗

[回复](http://www.wowotech.net/filesystem/396.html#comment-6002)

**[wowo](http://www.wowotech.net/)**\
2017-09-08 09:13

@向往：我们还没有系统性的整理文件系统的文章啊，所以这几个问题暂时没法回答哦，抱歉哈～～

[回复](http://www.wowotech.net/filesystem/396.html#comment-6006)

**[linuxer](http://www.wowotech.net/)**\
2017-06-29 09:41

^\_^，有点王家卫的感觉，我喜欢！

[回复](http://www.wowotech.net/filesystem/396.html#comment-5758)

**[驴肉火烧](http://www.wowotech.net/)**\
2017-06-29 21:23

@linuxer：感谢能喜欢，哈哈

[回复](http://www.wowotech.net/filesystem/396.html#comment-5762)

**[wowo](http://www.wowotech.net/)**\
2017-06-29 08:35

畅快的文字！好文章啊！！赞～～～

[回复](http://www.wowotech.net/filesystem/396.html#comment-5756)

**[驴肉火烧](http://www.wowotech.net/)**\
2017-06-29 21:24

@wowo：非常感谢，你们的文章都非常棒

[回复](http://www.wowotech.net/filesystem/396.html#comment-5763)

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

  - ja\
    [@dream：我看完這段也有相同的想法，引用 @dream ...](http://www.wowotech.net/kernel_synchronization/spinlock.html#8922)
  - 元神高手\
    [围观首席power managerment专家](http://www.wowotech.net/pm_subsystem/device_driver_pm.html#8921)
  - 十七\
    [内核空间的映射在系统启动时就已经设定好，并且在所有进程的页表...](http://www.wowotech.net/process_management/context-switch-arch.html#8920)
  - lw\
    [sparse模型和disconti模型没看出来有什么本质区别...](http://www.wowotech.net/memory_management/memory_model.html#8919)
  - 肥饶\
    [一个没设置好就出错](http://www.wowotech.net/linux_kenrel/516.html#8918)
  - orange\
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

  - [X-016-KERNEL-串口驱动开发之驱动框架](http://www.wowotech.net/x_project/serial_driver_porting_1.html)
  - [以太网驱动的流程浅析(四)-以太网驱动probe流程](http://www.wowotech.net/linux_kenrel/469.html)
  - [从“码农”说起](http://www.wowotech.net/tech_discuss/111.html)
  - [显示技术介绍(2)\_电子显示的前世今生](http://www.wowotech.net/display/display_tech_intro.html)
  - [Debian下的WiFi实验（一）：通过无线网卡连接AP](http://www.wowotech.net/linux_application/wifi-test-1.html)

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
