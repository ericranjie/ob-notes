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

作者：[安庆](http://www.wowotech.net/author/539 "oppo混合云内核&虚拟化负责人，架构并孵化了oppo的云游戏，云手机等产品。") 发布于：2021-5-8 9:08 分类：[Linux内核分析](http://www.wowotech.net/sort/linux_kenrel)

  

# 容器内就获取个cpu利用率，怎么就占用单核100%了呢

## 背景：这个是在centos7 + lxcfs 和jdk11 的环境上复现的

## 目前这个bug已经合入到了开源社区，

## 链接为 https://github.com/openjdk/jdk/pull/4378

  

  

### 下面列一下我们是怎么排查并解这个问题的。

  

#### 一、故障现象

oppo内核团队接到jvm的兄弟甘兄发来的一个案例,

现象是java的热点一直是如下函数，占比很高：

```

at sun.management.OperatingSystemImpl.getProcessCpuLoad(Native Method)

```

我们授权后登陆oppo云宿主，发现top显示如下：

```

   PID USER      PR  NI    VIRT    RES    SHR S %CPU %MEM     TIME+ COMMAND                                                                       

617248 service   20   0   11.9g   8.6g   9300 R 93.8  2.3   3272:09 java                                                                          

179526 service   20   0   11.9g   8.6g   9300 R 93.8  2.3  77462:17 java     

```

  

#### 二、故障现象分析

  

java的线程，能把cpu使用率打这么高，常见一般是gc或者跑jni的死循环，

从jvm团队反馈的热点看，这个线程应该在一直获取cpu利用率，而不是以前常见的问题。

从perf的角度看，该线程一直在触发read调用。

然后strace查看read的返回值，打印如下：

```

read(360, 0x7f6f9000b000, 4096)         = -1 ENOTCONN (Transport endpoint is not connected)

read(360, 0x7f6f9000b000, 4096)         = -1 ENOTCONN (Transport endpoint is not connected)

read(360, 0x7f6f9000b000, 4096)         = -1 ENOTCONN (Transport endpoint is not connected)

read(360, 0x7f6f9000b000, 4096)         = -1 ENOTCONN (Transport endpoint is not connected)

read(360, 0x7f6f9000b000, 4096)         = -1 ENOTCONN (Transport endpoint is not connected)

read(360, 0x7f6f9000b000, 4096)         = -1 ENOTCONN (Transport endpoint is not connected)

```

接着查看360这个fd到底是什么类型：

```

# ll /proc/22345/fd/360

lr-x------ 1 service service 64 Feb  4 11:24 /proc/22345/fd/360 -> /proc/stat

```

发现是在读取/proc/stat ,和上面的java热点读取cpu利用率是能吻合的。

  

根据/proc/stat在容器内的挂载点，

```

# cat /proc/mounts  |grep stat

tmpfs /proc/timer_stats tmpfs rw,nosuid,size=65536k,mode=755 0 0

lxcfs /proc/stat fuse.lxcfs rw,nosuid,nodev,relatime,user_id=0,group_id=0,allow_other 0 0

lxcfs /proc/stat fuse.lxcfs rw,nosuid,nodev,relatime,user_id=0,group_id=0,allow_other 0 0

lxcfs /proc/diskstats fuse.lxcfs rw,nosuid,nodev,relatime,user_id=0,group_id=0,allow_other 0 0

```

很奇怪的是，发现有两个挂载点都是/proc/stat，

尝试手工读取一下这个文件：

```

# cat /proc/stat

cpu  12236554 0 0 9808893 0 0 0 0 0 0

cpu0 10915814 0 0 20934 0 0 0 0 0 0

cpu1 1320740 0 0 9787959 0 0 0 0 0 0

```

说明bash访问这个挂载点没有问题，难道bash访问的挂载点和出问题的java线程访问的挂载点不是同一个？

查看对应java打开的fd的super_block和mount，

```

crash> files 22345 |grep -w 360

360 ffff914f93efb400 ffff91541a16d440 ffff9154c7e29500 REG  /rootfs/proc/stat/proc/stat

crash> inode.i_sb ffff9154c7e29500

  i_sb = 0xffff9158797bf000

```

然后查看对应进程归属mount的namespace中的挂载点信息：

```

crash> mount -n 22345 |grep lxcfs

ffff91518f28b000 ffff9158797bf000 fuse   lxcfs     /rootfs/proc/stat

ffff911fb0209800 ffff914b3668e800 fuse   lxcfs     /rootfs/var/lib/baymax/var/lib/baymax/lxcfs

ffff915535becc00 ffff914b3668e800 fuse   lxcfs     /rootfs/proc/cpuinfo

ffff915876b45380 ffff914b3668e800 fuse   lxcfs     /rootfs/proc/meminfo

ffff912415b56e80 ffff914b3668e800 fuse   lxcfs     /rootfs/proc/uptime

ffff91558260f300 ffff914b3668e800 fuse   lxcfs     /rootfs/proc/stat/proc/stat

ffff911f52a6bc00 ffff914b3668e800 fuse   lxcfs     /rootfs/proc/diskstats

ffff914d24617180 ffff914b3668e800 fuse   lxcfs     /rootfs/proc/swaps

ffff911de87d0180 ffff914b3668e800 fuse   lxcfs     /rootfs/sys/devices/system/cpu/online

```

由于cat操作较短，写一个简单的demo代码，open 之后不关闭并且read 对应的/proc/stat路径，发现它

访问的挂载点的super_blcok是 ffff914b3668e800，由于代码极为简单，在此不列出。

  

也就是进一步地确认了，出问题的java进程访问的是一个出问题的挂载点。

那为什么访问这个挂载点会返回ENOTCONN呢？

```

crash> struct file.private_data ffff914f93efb400

  private_data = 0xffff911e57b49b80---根据fuse模块代码，这个是一个fuse_file

  

crash> fuse_file.fc 0xffff911e57b49b80

  fc = 0xffff9158797be000

crash> fuse_conn.connected 0xffff9158797be000

  connected = 0

```

果然对应的fuse_conn的连接状态是0，从打点看，是在fuse_get_req返回的：

调用链如下：

```

sys_read-->vfs_read-->fuse_direct_read-->__fuse_direct_read-->fuse_direct_io

    --->fuse_get_req--->__fuse_get_req

  

static struct fuse_req *__fuse_get_req(struct fuse_conn *fc, unsigned npages,

               bool for_background)

{

...

  err = -ENOTCONN;

  if (!fc->connected)

    goto out;

...

}

```

  

下面要定位的就是，为什么出问题的java线程会没有判断read的返回值，而不停重试呢？

gdb跟踪一下：

```

(gdb) bt

#0  0x00007fec3a56910d in read () at ../sysdeps/unix/syscall-template.S:81

#1  0x00007fec3a4f5c04 in _IO_new_file_underflow (fp=0x7fec0c01ae00) at fileops.c:593

#2  0x00007fec3a4f6dd2 in __GI__IO_default_uflow (fp=0x7fec0c01ae00) at genops.c:414

#3  0x00007fec3a4f163e in _IO_getc (fp=0x7fec0c01ae00) at getc.c:39

#4  0x00007febeacc6738 in get_totalticks.constprop.4 () from /usr/local/paas-agent/HeyTap-Linux-x86_64-11.0.7/lib/libmanagement_ext.so

#5  0x00007febeacc6e47 in Java_com_sun_management_internal_OperatingSystemImpl_getSystemCpuLoad0 ()

   from /usr/local/paas-agent/HeyTap-Linux-x86_64-11.0.7/lib/libmanagement_ext.so

```

确认了用户态在循环进入 _IO_getc的流程，然后jvm的甘兄开始走查代码，发现有一个代码非常可疑：

UnixOperatingSystem.c：get_totalticks 函数，会有一个循环如下：

```

static void next_line(FILE *f) {

    while (fgetc(f) != '\n');

}

```

  

从fuse模块的代码看，fc->connected = 0的地方并不是很多，基本都是abort或者fuse_put_super调用导致，通过对照java线程升高的时间点以及查看journal的日志，进一步fuse的挂载点失效有用户态文件系统lxcfs退出，而挂载点还有inode在访问导致，导致又无法及时地回收这个super_block,新的挂载使用的是：

```

remount_container() {

    export f=/proc/stat    && test -f /var/lib/baymax/lxcfs/$f && (umount $f;  mount --bind /var/lib/baymax/lxcfs/$f $f)

```

使用了bind模式，具体bind的mount的特性可以参照网上的资料。

并且在journal日志中，查看到了关键的：

```

: umount: /proc/stat: target is busy

```

这行日志表明，当时卸载时出现失败，不是所有的访问全部关闭。

  

#### 三、故障复现

1.针对以上的怀疑点，写了一个很简单的demo如下：

```

#include <stdio.h>

#include <unistd.h>

#include <stdlib.h>

#define SCNd64 "I64d"

#define DEC_64 "%"SCNd64

  

static void next_line(FILE *f) {

    while (fgetc(f) != '\n');//出错的关键点

}

  

#define SET_VALUE_GDB 10

int main(int argc,char* argv[])

{

  unsigned long i =1;

  unsigned long j=0;

  FILE * f=NULL;

  FILE         *fh;

  unsigned long        userTicks, niceTicks, systemTicks, idleTicks;

  unsigned long        iowTicks = 0, irqTicks = 0, sirqTicks= 0;

  int             n;

  

  if((fh = fopen("/proc/stat", "r")) == NULL) {

    return -1;

  }

  

    n = fscanf(fh, "cpu " DEC_64 " " DEC_64 " " DEC_64 " " DEC_64 " " DEC_64 " "

                   DEC_64 " " DEC_64,

           &userTicks, &niceTicks, &systemTicks, &idleTicks,

           &iowTicks, &irqTicks, &sirqTicks);

  

    // Move to next line

    while(i!=SET_VALUE_GDB)----------如果挂载点不变，则走这个大循环，单核cpu接近40%

    {

       next_line(fh);//挂载点一旦变化，返回 ENOTCONN，则走这个小循环，单核cpu会接近100%

       j++;

    }

   fclose(fh);

   return 0;

}

```

执行以上代码，获得结果如下：

```

#gcc -g -o caq.o caq.c

一开始单独运行./caq.o，会看到cpu占用如下：

628957 root      20   0    8468    612    484 R  32.5  0.0  18:40.92 caq.o 

发现cpu占用率时32.5左右，

此时挂载点信息如下：

crash> mount -n 628957 |grep lxcfs

ffff88a5a2619800 ffff88a1ab25f800 fuse   lxcfs     /rootfs/proc/stat

ffff88cf53417000 ffff88a4dd622800 fuse   lxcfs     /rootfs/var/lib/baymax/var/lib/baymax/lxcfs

ffff88a272f8c600 ffff88a4dd622800 fuse   lxcfs     /rootfs/proc/cpuinfo

ffff88a257b28900 ffff88a4dd622800 fuse   lxcfs     /rootfs/proc/meminfo

ffff88a5aff40300 ffff88a4dd622800 fuse   lxcfs     /rootfs/proc/uptime

ffff88a3db2bd680 ffff88a4dd622800 fuse   lxcfs     /rootfs/proc/stat/proc/stat

ffff88a2836ba400 ffff88a4dd622800 fuse   lxcfs     /rootfs/proc/diskstats

ffff88bcb361b600 ffff88a4dd622800 fuse   lxcfs     /rootfs/proc/swaps

ffff88776e623480 ffff88a4dd622800 fuse   lxcfs     /rootfs/sys/devices/system/cpu/online

  

由于没有关闭/proc/stat的fd，也就是进行大循环，然后这个时候重启lxcfs挂载：

#systemctl restart lxcfs

重启之后，发现挂载点信息如下：

crash> mount -n 628957 |grep lxcfs

ffff88a5a2619800 ffff88a1ab25f800 fuse   lxcfs     /rootfs/proc/stat

ffff88a3db2bd680 ffff88a4dd622800 fuse   lxcfs     /rootfs/proc/stat/proc/stat------------这个挂载点，由于fd未关闭，所以卸载肯定失败，可以看到super_block是重启前的

ffff887795a8f600 ffff88a53b6c6800 fuse   lxcfs     /rootfs/var/lib/baymax/var/lib/baymax/lxcfs

ffff88a25472ae80 ffff88a53b6c6800 fuse   lxcfs     /rootfs/proc/cpuinfo

ffff88cf75ff1e00 ffff88a53b6c6800 fuse   lxcfs     /rootfs/proc/meminfo

ffff88a257b2ad00 ffff88a53b6c6800 fuse   lxcfs     /rootfs/proc/uptime

ffff88cf798f0d80 ffff88a53b6c6800 fuse   lxcfs     /rootfs/proc/stat/proc/stat/proc/stat--------bind模式挂载，会新生成一个/proc/stat

ffff88cf36ff2880 ffff88a53b6c6800 fuse   lxcfs     /rootfs/proc/diskstats

ffff88cf798f1f80 ffff88a53b6c6800 fuse   lxcfs     /rootfs/proc/swaps

ffff88a53f295980 ffff88a53b6c6800 fuse   lxcfs     /rootfs/sys/devices/system/cpu/online

cpu立刻打到接近100%

628957 root      20   0    8468    612    484 R  98.8  0.0  18:40.92 caq.o     

```

#### 四、故障规避或解决

  

我们的解决方案是：

  

1.在fgetc 中增加返回值的判断，不能无脑死循环。

  

2.由于lxcfs在3.1.2 的版本会有段错误导致退出的bug，尽量升级到4.0.0以上版本，不过这个涉及到libfuse动态库的版本适配。

  

3.在lxcfs等使用fuse的用户态文件系统进行remount的时候，如果遇到umount失败，则延迟加重试，超过次数通过日志告警系统抛出告警。

  
  

  

[![](http://www.wowotech.net/content/uploadfile/201605/ef3e1463542768.png)](http://www.wowotech.net/support_us.html)

« [一个较复杂dcache问题](http://www.wowotech.net/linux_kenrel/484.html) | [关于numa loadbance的死锁分析](http://www.wowotech.net/linux_kenrel/482.html)»

**评论：**

**linzai**  
2024-03-12 10:52

大佬你好，你在文中多次使用crash，但是crash是需要dump文件，你的dump文件是将云宿主机通过/proc/sys/sysrq-trigger直接生成的吗？

[回复](http://www.wowotech.net/linux_kenrel/483.html#comment-8871)

**liuxu**  
2021-09-01 12:54

大佬确实屌，分析的很透彻，平时都很少注意文件系统被umount的情况

[回复](http://www.wowotech.net/linux_kenrel/483.html#comment-8284)

**萌**  
2021-05-08 13:45

一脸蒙蔽的来，一脸懵逼的走。

[回复](http://www.wowotech.net/linux_kenrel/483.html#comment-8227)

**[安庆](http://www.wowotech.net/)**  
2021-05-08 14:31

@萌：

[回复](http://www.wowotech.net/linux_kenrel/483.html#comment-8228)

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
    
    - [linux内核中的GPIO系统之（4）：pinctrl驱动的理解和总结](http://www.wowotech.net/gpio_subsystem/pinctrl-driver-summary.html)
    - [linux cpufreq framework(4)_cpufreq governor](http://www.wowotech.net/pm_subsystem/cpufreq_governor.html)
    - [使用pxe方式安装系统](http://www.wowotech.net/linux_application/288.html)
    - [一个技术男眼中的Smartisan T1手机](http://www.wowotech.net/tech_discuss/112.html)
    - [基本电路概念之（一）：什么是电压？](http://www.wowotech.net/basic_subject/voltage.html)
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