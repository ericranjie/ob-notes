
Linux内核之旅

 _2021年11月12日 15:33_

The following article is from 内核工匠 Author Stephen Fang

![](http://wx.qlogo.cn/mmhead/Q3auHgzwzM6RLUCnKO3ibq4vBV8CTFqV5ibnt5e1aeIn8ZRoxrtproqA/0)

**内核工匠**.

分享Linux内核相关黑科技、技术文章、技术资讯和精选教程

](https://mp.weixin.qq.com/s?__biz=MzI3NzA5MzUxNA==&mid=2664610401&idx=1&sn=90101e53ea57a7cea9486c5faf1d4086&chksm=f04d9784c73a1e92109b84d794ebc36d811be3bddd5cee93d5ceddda95bb2c82e132cfa02cb6&mpshare=1&scene=24&srcid=1112Y0MVVt2FmflHBL9mFfE5&sharer_sharetime=1636704742364&sharer_shareid=5fb9813bfe9ffc983435bfc8d8c5e9ca&key=daf9bdc5abc4e8d03ce983d15cb959cc1cd389623acfee5e75cc0cfc90f80c259b86ef227ed1459b478cb0c6e226ab3ec498195345fe2e4387d61edcf9f2ecbddcba3f89e1859a843265d5405abd024cf43f94160b9cb08d324b6700594178ded52cba38830d14a6c499bfca254701e292b9e1a957d637a1ca010becd6609385&ascene=14&uin=MTEwNTU1MjgwMw%3D%3D&devicetype=iMac+MacBookAir10%2C1+OSX+OSX+14.6.1+build(23G93)&version=13080710&nettype=WIFI&lang=en&session_us=gh_3e85a6de261f&countrycode=CN&fontScale=100&exportkey=n_ChQIAhIQZW1rcYQKYzEYBtTbv9PtaBKUAgIE97dBBAEAAAAAAE1DCj5VWSIAAAAOpnltbLcz9gKNyK89dVj0aER9T6jytG%2BncKWncrnyaB%2FpYBK72OGEEfh3eMHbC7QWlQNMQkp7Y4XqwKTWkMuBXl6Fw5ptfHnpTMMD%2F6y2ocxG4ENrlyWtGmEnrSkWV%2BQdB7W7Vd8c4hfGZQvwrVSYLo9NpEoJeXbAUdCUBPgsgh4kWNlz8paQVWJDKqxlSI0QyUl9ZnlUvrEZOIBAo9%2BEFO2Z30ODIzPHKCH2zZGAQDzXPOk07k8SGKWJhNF395z4ReqNXikFeOfgRQ76Wo%2FvbRstXsNMnqjMLcO%2FqpVAZfBgqfiQMZGdTH4QDBm2XL7%2FbjAyF11RBHmpJG6GzA%3D%3D&acctmode=0&pass_ticket=BYSpaxEC6TUmYfPAUirDQs5AbssVRvdKp1ApHrku%2F%2Bz6INFA3czlX2i9Bc4Zk3G5&wx_header=0#)

  

**一、引言**

作为一名基民(韭菜)，对于“做时间的朋友”这一金句一定耳熟能详（深恶痛绝）。各大知名基金经理教育我们，想做好投资，需要跟时间“做朋友”，对于认准的基金要坚定的长期持有。先不论在投资方面这句话是否适用，但它在linux开发领域却有一定的道理。作为一名linux内核开发者，如果想要了解一段程序执行的是否高效，一个很关键的指标就是它占用了cpu多长时间。“做时间的朋友”，了解linux系统中形形色色的cpu时间，是linux系统工程师需要掌握的重要技能之一，接下来我们就来简单的聊一聊这些linux时间守护者们。

  

**二、各司其职****的时间****守护者们**

**1. 总揽全局的老大哥cputime**

  

通过读取/proc/stat，我们可以看到cpu使用时间的分类显示:

  

![Image](https://mmbiz.qpic.cn/mmbiz_png/d4hoYJlxOjNw55bWznUvicLoARic5lWibAzC09eZYa7gZBAoFO1aFS3CGpGI4E6FLxYo5icRPSdz7pHVmThibHziaOtw/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

  

内核对应的时间类型定义在头文件include/linux/kernel_stat.h，上图中cpu[0...7]后的数值跟这些类型依次对应：

  

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

它们分别代表如下含义：

  

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

在linux系统中,cputime模块具有重要的意义。它记录了设备中所有cpu在各个状态下经过的时间。我们所熟悉的top工具就是用cputime换算出的cpu利用率。

  

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

  

**2. 功耗****分析的好帮手****cpufreq stats：**

cpufreq_stats模块的开启需要使能CONFIG_CPU_FREQ_STAT宏。当系统使能该特性后，cpufreq driver sysfs下生成stats目录：

  

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

其中time_in_state节点，表示的是该cpufreq policy内分别处于各个频点的时间，单位为jiffies。由于该系统内核HZ设置为100, 所以每个jiffies为10ms。如上图所示，policy0在691200这个频点上经过了101310ms。有了这个功能，我们就能获取每个cluster上cpu运行最多的频点是哪些，进而针对性的对系统功耗进行优化。

**3. **人多势众****的小兄弟cpufreq_times****：****

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

cpufreq times由procfs下的proc/[pid] /time_in_state节点来呈现，该节点记录了该进程(线程)在各个cpufreq policy的各个频点下驻留的时间, 单位为clock_t。clock_t是由USER_HZ来决定，该系统中USER_HZ为100，则clock_t代表10ms。如上图所示，进程1（一般为init进程），在cpufreq policy0的691200频点下运行了40ms，其他的频点情况依次类推。由于每个进程的派生，都会生成一个新的/proc/[pid]目录，所以这个节点也是cpu时间家族中数量最多的成员，说一句”人多势众”再合适不过。

  

  

**4. cpu睡眠质量****记录者cpuidle time****：**

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

cpuidle time模块的工作就是记录每个cpu在各层”梦境”中睡了多久，即每次开机以来，每个核在每个c state(idle状态称之为c state)下的时长。通过cpuidle driver sysfs中的time节点展示，单位为us。如上图所示，显示了该设备的cpu0在cpuidle state0/state1分别驻留了1868513998 us / 8196819678 us。

  

  

**三、各模块的工作原理**

**1. cpu****time:**

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

图1 cputime更新流程图

  

cputime模块代码位于kernel/sched/cputime.c。

当每次timer中断来临时，kernel经过由中断处理函数调用到irqtime_account_process_tick()（需要使能特性宏CONFIG_IRQ_TIME_ACCOUNTING，将irq/softirq的统计囊括其中）。通过判断当前task是否为softirq/user tick/idle进程/guest系统进程/内核进程，将经历的cpu时间（通常为1个tick）添加到第一章所述对应的类型中去。

  

1）示例代码：

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

  

**2. cpufreq****_times:**

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

图2 cpufreq_times更新示意图

  

cpufreq_times模块代码位于drivers/cpufreq/cpufreq_times.c。它的更新涉及到其他两个模块:cpufreq driver与cputime。

  

当cpufreq policy频率改变时，cpufreq driver通过cpufreq_notify_transition(普通调频模式)或者cpufreq_driver_fast_switch（快速调频模式）调用cpufreq_times_record_transition函数，通知cpufreq_times模块当前该policy处于哪一个频点。

  

当cputime模块接收到timer中断后，会调用cpufreq_acct_update_power（），将该tick添加到cpufreq_times模块当前任务及当前频点的统计上。

**3. **cpufreq_stats****：****  

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

图3 cpufreq_stats更新示意图

  

cpufreq_stats模块代码位于drivers/cpufreq/cpufreq_stats.c。它的更新有些类似于cpufreq_times, 但与其不同的是只涉及cpufreq driver一个外部模块。

  

当cpufreq policy频率改变时，cpufreq driver通过cpufreq_notify_transition(普通调频模式)或者cpufreq_driver_fast_switch（快速调频模式）调用cpufreq_times_record_transition函数调用cpufreq_stats_record_transition函数，通知cpufreq_stats模块此刻发生调频以及要切换到哪一个目标频点。

1. 示例代码：
    

  

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

cpufreq_state模块则调用cpufreq_stats_update获取当前jiffies,并与上一次更新时的jiffies相减，最后将差值添加到上个频点的时间统计中。

  

  

**4. cpu****idle time:**

cpuidle time模块代码在drivers/cpuidle/cpuidle.c。

当某个cpu runqueue上没有runnable状态的任务时，该cpu调度到idle进程。idle的流程在这里不再赘述，经过层层调用，最后执行到cpuidle_enter_state(）函数。

1）示例代码：

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

  

通过local_clock()记录下进入idle(调用target_state->enter()这个回调函数)前后时间点, 取其差值将其保存到cpuidle_state_usage结构体中的time成员变量中。

**四、总结**

linux系统中时间相关的模块数不胜数，上述的四个模块不过是冰山一角。深入了解这些时间统计的意义及实现原理对于系统性能功耗优化有着很重要的意义。

  

  

参考：

https://lwn.net/Kernel/

  

  

  

  

  

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**长按关注**

**内核工匠微信**

  

Linux 内核黑科技 | 技术文章 | 精选教程

Reads 2058

​

Comment

**留言 2**

- 黑胡子杰克
    
    2021年11月12日
    
    Like1
    
    自由和科学应该是一脉相承的，现在很理解那些当年对网络安全担忧并开始提出TLS 以及DNS over TLS的计算机科学家和工程师们致敬。这些人对未来的洞察是超越时代的
    
- Culaccino
    
    2021年11月12日
    
    Like
    
    看到了使能这个翻译![[捂脸]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![[捂脸]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    

已无更多数据

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/SeWfibBcBT0EibtIWVNvshnuWMN1AoJw3poFIsbpaIVyZibCCqwBUR21rcDfrQgoqYzaYNdS14IIXzvmzvibdDa5Rw/300?wx_fmt=png&wxfrom=18)

Linux内核之旅

6Share7

2

Comment

**留言 2**

- 黑胡子杰克
    
    2021年11月12日
    
    Like1
    
    自由和科学应该是一脉相承的，现在很理解那些当年对网络安全担忧并开始提出TLS 以及DNS over TLS的计算机科学家和工程师们致敬。这些人对未来的洞察是超越时代的
    
- Culaccino
    
    2021年11月12日
    
    Like
    
    看到了使能这个翻译![[捂脸]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![[捂脸]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    

已无更多数据