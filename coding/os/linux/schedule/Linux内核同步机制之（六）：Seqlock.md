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

作者：[linuxer](http://www.wowotech.net/author/3 "linuxer") 发布于：2015-9-9 11:55 分类：[内核同步机制](http://www.wowotech.net/sort/kernel_synchronization)

一、前言

普通的spin lock对待reader和writer是一视同仁，RW spin lock给reader赋予了更高的优先级，那么有没有让writer优先的锁的机制呢？答案就是seqlock。本文主要描述linux kernel 4.0中的seqlock的机制，首先是seqlock的工作原理，如果想浅尝辄止，那么了解了概念性的东东就OK了，也就是第二章了，当然，我还是推荐普通的驱动工程师了解seqlock的API，第三章给出了一个简单的例子，了解了这些，在驱动中（或者在其他内核模块）使用seqlock就可以易如反掌了。细节是魔鬼，概念性的东西需要天才的思考，不是说就代码实现的细节就无足轻重，如果想进入seqlock的内心世界，推荐阅读第四章seqlock的代码实现，这一章和cpu体系结构相关的内容我们选择了ARM64（呵呵～～要跟上时代的步伐）。最后一章是参考资料，如果觉得本文描述不清楚，可以参考这些经典文献，在无数不眠之夜，她们给我心灵的慰籍，也愿能够给读者带来快乐。

二、工作原理

1、overview

seqlock这种锁机制是倾向writer thread，也就是说，除非有其他的writer thread进入了临界区，否则它会长驱直入，无论有多少的reader thread都不能阻挡writer的脚步。writer thread这么霸道，reader肿么办？对于seqlock，reader这一侧需要进行数据访问的过程中检测是否有并发的writer thread操作，如果检测到并发的writer，那么重新read。通过不断的retry，直到reader thread在临界区的时候，没有任何的writer thread插入即可。这样的设计对reader而言不是很公平，特别是如果writer thread负荷比较重的时候，reader thread可能会retry多次，从而导致reader thread这一侧性能的下降。

总结一下seqlock的特点：临界区只允许一个writer thread进入，在没有writer thread的情况下，reader thread可以随意进入，也就是说reader不会阻挡reader。在临界区只有有reader thread的情况下，writer thread可以立刻执行，不会等待。

2、writer thread的操作

对于writer thread，获取seqlock操作如下：

（1）获取锁（例如spin lock），该锁确保临界区只有一个writer进入。

（2）sequence counter加一

释放seqlock操作如下：

（1）释放锁，允许其他writer thread进入临界区。

（2）sequence counter加一（注意：不是减一哦，sequence counter是一个不断累加的counter）

由上面的操作可知，如果临界区没有任何的writer thread，那么sequence counter是偶数（sequence counter初始化为0），如果临界区有一个writer thread（当然，也只能有一个），那么sequence counter是奇数。

3、reader thread的操作如下：

（1）获取sequence counter的值，如果是偶数，可以进入临界区，如果是奇数，那么等待writer离开临界区（sequence counter变成偶数）。进入临界区时候的sequence counter的值我们称之old sequence counter。

（2）进入临界区，读取数据

（3）获取sequence counter的值，如果等于old sequence counter，说明一切OK，否则回到step（1）

4、适用场景。一般而言，seqlock适用于：

（1）read操作比较频繁

（2）write操作较少，但是性能要求高，不希望被reader thread阻挡（之所以要求write操作较少主要是考虑read side的性能）

（3）数据类型比较简单，但是数据的访问又无法利用原子操作来保护。我们举一个简单的例子来描述：假设需要保护的数据是一个链表，header--->A node--->B node--->C node--->null。reader thread遍历链表的过程中，将B node的指针赋给了临时变量x，这时候，中断发生了，reader thread被preempt（注意，对于seqlock，reader并没有禁止抢占）。这样在其他cpu上执行的writer thread有充足的时间释放B node的memory（注意：reader thread中的临时变量x还指向这段内存）。当read thread恢复执行，并通过x这个指针进行内存访问（例如试图通过next找到C node），悲剧发生了……

三、API示例

在kernel中，jiffies_64保存了从系统启动以来的tick数目，对该数据的访问（以及其他jiffies相关数据）需要持有jiffies_lock这个seq lock。

1、reader side代码如下：

> u64 get_jiffies_64(void)  
> {
> 
>     do {  
>         seq = read_seqbegin(&jiffies_lock);  
>         ret = jiffies_64;  
>     } while (read_seqretry(&jiffies_lock, seq));  
> }

2、writer side代码如下：

> static void tick_do_update_jiffies64(ktime_t now)  
> {  
>     write_seqlock(&jiffies_lock);
> 
> 临界区会修改jiffies_64等相关变量，具体代码略  
>     write_sequnlock(&jiffies_lock);  
> }

对照上面的代码，任何工程师都可以比着葫芦画瓢，使用seqlock来保护自己的临界区。当然，seqlock的接口API非常丰富，有兴趣的读者可以自行阅读seqlock.h文件。

四、代码实现

1、seq lock的定义

> typedef struct {  
>     struct seqcount seqcount;－－－－－－－－－－sequence counter  
>     spinlock_t lock;  
> } seqlock_t;

seq lock实际上就是spin lock ＋ sequence counter。

2、write_seqlock/write_sequnlock

> static inline void write_seqlock(seqlock_t *sl)  
> {  
>     spin_lock(&sl->lock);
> 
>     sl->sequence++;  
>     smp_wmb();  
> }

唯一需要说明的是smp_wmb这个用于SMP场合下的写内存屏障，它确保了编译器以及CPU都不会打乱sequence counter内存访问以及临界区内存访问的顺序（临界区的保护是依赖sequence counter的值，因此不能打乱其顺序）。write_sequnlock非常简单，留给大家自己看吧。

3、read_seqbegin

> static inline unsigned read_seqbegin(const seqlock_t *sl)  
> {   
>     unsigned ret;
> 
> repeat:  
>     ret = ACCESS_ONCE(sl->sequence); －－－进入临界区之前，先要获取sequenc counter的快照  
>     if (unlikely(ret & 1)) { －－－－－如果是奇数，说明有writer thread  
>         cpu_relax();  
>         goto repeat; －－－－如果有writer，那么先不要进入临界区，不断的polling sequenc counter  
>     }
> 
>     smp_rmb(); －－－确保sequenc counter和临界区的内存访问顺序  
>     return ret;  
> }

如果有writer thread，read_seqbegin函数中会有一个不断polling sequenc counter，直到其变成偶数的过程，在这个过程中，如果不加以控制，那么整体系统的性能会有损失（这里的性能指的是功耗和速度）。因此，在polling过程中，有一个cpu_relax的调用，对于ARM64，其代码是：

> static inline void cpu_relax(void)  
> {  
>         asm volatile("yield" ::: "memory");  
> }

yield指令用来告知硬件系统，本cpu上执行的指令是polling操作，没有那么急迫，如果有任何的资源冲突，本cpu可以让出控制权。

4、read_seqretry

> static inline unsigned read_seqretry(const seqlock_t *sl, unsigned start)  
> {  
>     smp_rmb();－－－确保sequenc counter和临界区的内存访问顺序  
>     return unlikely(sl->sequence != start);  
> }

start参数就是进入临界区时候的sequenc counter的快照，比对当前退出临界区的sequenc counter，如果相等，说明没有writer进入打搅reader thread，那么可以愉快的离开临界区。

还有一个比较有意思的逻辑问题：read_seqbegin为何要进行奇偶判断？把一切都推到read_seqretry中进行判断不可以吗？也就是说，为何read_seqbegin要等到没有writer thread的情况下才进入临界区？其实有writer thread也可以进入，反正在read_seqretry中可以进行奇偶以及相等判断，从而保证逻辑的正确性。当然，这样想也是对的，不过在performance上有欠缺，reader在检测到有writer thread在临界区后，仍然放reader thread进入，可能会导致writer thread的一些额外的开销（cache miss），因此，最好的方法是在read_seqbegin中拦截。

五、参考文献

1、Understanding the Linux Kernel 3rd Edition

2、Linux Kernel Development 3rd Edition

3、Perfbook ([https://www.kernel.org/pub/linux/kernel/people/paulmck/perfbook/perfbook.html](https://www.kernel.org/pub/linux/kernel/people/paulmck/perfbook/perfbook.html "https://www.kernel.org/pub/linux/kernel/people/paulmck/perfbook/perfbook.html"))

_原创文章，转发请注明出处。蜗窝科技_

标签: [Seqlock](http://www.wowotech.net/tag/Seqlock)

[![](http://www.wowotech.net/content/uploadfile/201605/ef3e1463542768.png)](http://www.wowotech.net/support_us.html)

« [Linux CPU core的电源管理(5)_cpu control及cpu hotplug](http://www.wowotech.net/pm_subsystem/cpu_hotplug.html) | [linux cpufreq framework(4)_cpufreq governor](http://www.wowotech.net/pm_subsystem/cpufreq_governor.html)»

**评论：**

**呜啦啦**  
2019-07-09 11:13

（3）数据类型比较简单，但是数据的访问又无法利用原子操作来保护。我们举一个简单的例子来描述：假设需要保护的数据是一个链表，header--->A node--->B node--->C node--->null。reader thread遍历链表的过程中，将B node的指针赋给了临时变量x，这时候，中断发生了，reader thread被preempt（注意，对于seqlock，reader并没有禁止抢占）。这样在其他cpu上执行的writer thread有充足的时间释放B node的memory（注意：reader thread中的临时变量x还指向这段内存）。当read thread恢复执行，并通过x这个指针进行内存访问（例如试图通过next找到C node），悲剧发生了……  
  
这是场景是不是不太适合seqlock，如果就是read thread校验通过认为此时临时变量是有效的，后面write thread又把对应的B node释放了，read thread试着访问B node的时候不还是会出错吗？

[回复](http://www.wowotech.net/kernel_synchronization/seqlock.html#comment-7517)

**[bsp](http://www.wowotech.net/)**  
2022-04-18 20:53

@呜啦啦：是的，我也觉得seqlock不适合保护指针。

[回复](http://www.wowotech.net/kernel_synchronization/seqlock.html#comment-8588)

**double_qiang**  
2023-03-15 14:07

@呜啦啦：不是啊，write thread把reader thread给抢占了，然后将对应的B node释放后，之后reader thread还会再重新进临界区，这时候再去遍历链表时，链表中已经不存在B node，又怎么会去访问B node呢

[回复](http://www.wowotech.net/kernel_synchronization/seqlock.html#comment-8754)

**storage**  
2023-04-28 12:37

@呜啦啦：好问题，确实是这样

[回复](http://www.wowotech.net/kernel_synchronization/seqlock.html#comment-8777)

**air_123456**  
2019-02-08 18:24

reader在检测到有writer thread在临界区后，仍然放reader thread进入，可能会导致writer thread的一些额外的开销（cache miss）     //这里为什么会导致writer thread的cache miss????

[回复](http://www.wowotech.net/kernel_synchronization/seqlock.html#comment-7178)

**sailing**  
2019-06-20 14:13

@air_123456：writer thread 在另外一个CPU上，应该没啥关系。我认为主要是此时读关键数据也是白读，如果这个无意义的读影响到了本CPU的Cache，那就更无意义了.

[回复](http://www.wowotech.net/kernel_synchronization/seqlock.html#comment-7478)

**flyfish**  
2017-10-08 23:00

现在在分析网络子系统，除了看代码之外还需要阅读大量的资料，譬如RFC，还要上班，加班，而工作又跟linux kernel没有太大关系，感觉真累

[回复](http://www.wowotech.net/kernel_synchronization/seqlock.html#comment-6091)

**[wowo](http://www.wowotech.net/)**  
2017-10-10 16:35

@flyfish：要有兴趣，喜欢就不累了～@

[回复](http://www.wowotech.net/kernel_synchronization/seqlock.html#comment-6098)

**qnh**  
2017-04-11 15:47

write_sequnlock里：  
sequence counter加一确实应该在释放锁之前，需要这个锁来保护sequence counter

[回复](http://www.wowotech.net/kernel_synchronization/seqlock.html#comment-5456)

**fanzhang**  
2017-07-30 10:06

@qnh：嗯，对，这里有一个spin_lock,确保只有一个writer.

[回复](http://www.wowotech.net/kernel_synchronization/seqlock.html#comment-5859)

**will**  
2016-08-29 19:18

释放seqlock操作如下：  
  
（1）释放锁，允许其他writer thread进入临界区。  
  
（2）sequence counter加一（注意：不是减一哦，sequence counter是一个不断累加的counter）  
  
这个加一应该在释放锁之前吧。。？？？

[回复](http://www.wowotech.net/kernel_synchronization/seqlock.html#comment-4465)

**[linuxer](http://www.wowotech.net/)**  
2016-08-30 08:35

@will：为何你觉得是加一在前呢？

[回复](http://www.wowotech.net/kernel_synchronization/seqlock.html#comment-4466)

**js_wawayu**  
2016-05-25 14:55

都是高手，佩服！

[回复](http://www.wowotech.net/kernel_synchronization/seqlock.html#comment-3974)

**[维尼](http://www.wowotech.net/)**  
2015-09-17 11:13

都3.10后内核都是64的 但是为什么大部分手机还是32的    
楼主能 讲一下 怎么看android 手机 内核位数吗  我都是print一下 long类型

[回复](http://www.wowotech.net/kernel_synchronization/seqlock.html#comment-2598)

**[linuxer](http://www.wowotech.net/)**  
2015-09-17 23:20

@维尼：linux支持多种cpu，32位的、64位的、ARM、x86、powerPC什么的都支持，只要在编译内核的时候进行配置就OK了。

[回复](http://www.wowotech.net/kernel_synchronization/seqlock.html#comment-2603)

**[维尼](http://www.wowotech.net/)**  
2015-09-18 08:57

@linuxer：现在市面是的手机貌似没有64位的内核啊

[回复](http://www.wowotech.net/kernel_synchronization/seqlock.html#comment-2606)

**hony**  
2017-04-30 12:45

@维尼：可以查看，cat /proc/cpuinfo

[回复](http://www.wowotech.net/kernel_synchronization/seqlock.html#comment-5505)

**[schedule](http://www.wowotech.net/)**  
2015-09-13 18:36

btrfs 目前看没啥前途啊，linuxer怎么看中了这个模块

[回复](http://www.wowotech.net/kernel_synchronization/seqlock.html#comment-2568)

**[linuxer](http://www.wowotech.net/)**  
2015-09-14 09:02

@schedule：原因有几个：  
1、我一直对文件系统比较感兴趣，但是之前由于搞嵌入式，因此都是一些小打小闹的文件系统（例如yaffs），一直想了解一个功能比较强大的文件系统的运作机制。  
2、有一段时间我很关注内核的更新情况（现在都没有时间去了解了），看到btrfs更新非常频繁，处于活跃期，这时候的模块有很多issue，可参与度比较高  
3、我简单的检索了一下资料，似乎btrfs有取代EXT4文件系统的趋势  
  
文件系统模块，schedule有什么推荐？

[回复](http://www.wowotech.net/kernel_synchronization/seqlock.html#comment-2569)

**[schedule](http://www.wowotech.net/)**  
2015-09-14 09:18

@linuxer：其实文件系统方面我就是菜鸟级别的。  
btrfs更新是很快，但是感觉好多东西都是在模仿ZFS，ZFS 10年前都有商业部署了，btrfs概念炒的比较厉害，但是总是感觉像付不起的阿斗，有那么点不给力。一直以为红帽会在fedora发行版上作为默认的文件系统，结果XFS 突然上位。  
  
我觉得现在想在文件系统做投入，要么搞大（像Global FS），要么搞小（F2FS）。PC或者小型机级别的FS都被云概念炒翻了。当然btrfs很多概念很先进，学习是绝对值得的。

[回复](http://www.wowotech.net/kernel_synchronization/seqlock.html#comment-2570)

**[linuxer](http://www.wowotech.net/)**  
2015-09-14 12:38

@schedule：schedule，你是做哪一行的？虽然你说你是菜鸟级别，不过感觉对fs还是比较了解的，你是搞服务器（数据中心、云计算什么的）方向的吗？

[回复](http://www.wowotech.net/kernel_synchronization/seqlock.html#comment-2572)

**[schedule](http://www.wowotech.net/)**  
2015-09-15 20:51

@linuxer：呃呃呃，我是个医生。。。从小对电脑感兴趣，没事就看看代码，

[回复](http://www.wowotech.net/kernel_synchronization/seqlock.html#comment-2585)

**[linuxer](http://www.wowotech.net/)**  
2015-09-15 22:45

@schedule：天哪，我已经晕倒......别和我说话，我想静静

[回复](http://www.wowotech.net/kernel_synchronization/seqlock.html#comment-2587)

**wowo**  
2015-09-15 22:48

@schedule：这才是真正的大神啊，我的偶像啊！把兴趣留给业余时间，不想当厨子的生物学家不是好程序员啊！羡慕……

[回复](http://www.wowotech.net/kernel_synchronization/seqlock.html#comment-2590)

**[schedule](http://www.wowotech.net/)**  
2015-09-15 23:25

@wowo：比不了你们年轻人啊，周围的朋友有爱玩飞行器的，有玩汽车的，偏偏我喜欢静心下来看看代码，就像看书似的，就是一种爱好。

**C**  
2015-12-18 16:04

@schedule：这位大哥使我想到一个澳大利亚的kernel hacker，他的正常职业是麻醉师，最早他帮内核写了一个Fair Scheduler，得到大量好评，然后Ingo眼红，跟着写了一个Complete Fair Scheduler， 然后两个scheduler互相PK看谁能进主线， 结果最后linus拍板说CFS进吧，问他为啥，他说，Fair Scheduler的哥们回复邮件太慢了。。。 别人又不是专职的，怎么可能和Ingo这些geek拼时间。

[回复](http://www.wowotech.net/kernel_synchronization/seqlock.html#comment-3258)

**[wowo](http://www.wowotech.net/)**  
2015-12-18 16:07

@C：哈哈，让我笑一会…

**[tigger](http://www.wowotech.net/)**  
2015-12-21 10:06

@schedule：医生啊。。。。。。  
哪个科室？  
听说牙科跟妇产科最挣钱

[回复](http://www.wowotech.net/kernel_synchronization/seqlock.html#comment-3261)

**lamaboy**  
2017-04-14 14:22

@tigger：其实我有个疑问，一直想清楚，医生在给人看病的时候，是不是像程序员一样先开点药试下，看看效果理想不？？ 不理想找问题再试！ 会不会是在这个过程中把病情给耽误了？？

**[leefong.chen](http://www.wowotech.net/)**  
2015-09-11 16:14

@linuxer 你在Linux maillist中提交patch了吗？模块比较多，想为Linux做点贡献，想找些志同道合的人一起分析、讨论内核某个模块，可否建立一个平台，让大家讨论最新Linux。

[回复](http://www.wowotech.net/kernel_synchronization/seqlock.html#comment-2558)

**[linuxer](http://www.wowotech.net/)**  
2015-09-11 18:31

@leefong.chen：很遗憾，我也没有提交过patch，如果想要提交patch，自身需要很强才可以，我感觉目前我的水平还不足以向内核提交patch（当然，那种外设驱动级别或者ARM下的某个machine级别的code之类的patch并不需要很高的能力，但是我没有兴趣），因此我现在就是想修炼内功，先把内核的几个基础模块（中断子系统、时间子系统、内核同步机制、进程管理、内存管理、I/O系统）研究透彻之后，然后选定一个内核模块（例如现在比较活跃的btrfs），进行深入理解，思考，看看是否有机会增加其功能或者改善其性能。  
  
BTW，你想建立平台，让志同道合的人一起分析、讨论内核某个模块，想法当然好，不过这样的人其实很少，就算是有，又忙于生计，其实是挺困难的一件事情，因此，我对这件事情不看好，很容易虎头蛇尾......

[回复](http://www.wowotech.net/kernel_synchronization/seqlock.html#comment-2559)

**[leefong.chen](http://www.wowotech.net/)**  
2015-09-11 21:06

@linuxer：中国就是这样，先要生计，再谈兴趣。什么时候学习IO系统，一起学习。

[回复](http://www.wowotech.net/kernel_synchronization/seqlock.html#comment-2560)

**mj**  
2015-09-11 07:09

什么时候也出一篇关于RCU的文章，特别是update的时机，期待中。

[回复](http://www.wowotech.net/kernel_synchronization/seqlock.html#comment-2556)

**[linuxer](http://www.wowotech.net/)**  
2015-09-11 09:04

@mj：蜗窝当然不会忽略RCU这么强大的工具，关于RCU的文档已经开了个头，不过RCU比较复杂，估计需要酝酿一段时间，^_^

[回复](http://www.wowotech.net/kernel_synchronization/seqlock.html#comment-2557)

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
    
    - [X-010-UBOOT-使用booti命令启动kernel(Bubblegum-96平台)](http://www.wowotech.net/x_project/bubblegum_uboot_booti.html)
    - [TLB flush操作](http://www.wowotech.net/memory_management/tlb-flush.html)
    - [Linux时间子系统之（十四）：tick broadcast framework](http://www.wowotech.net/timer_subsystem/tick-broadcast-framework.html)
    - [Linux cpuidle framework(1)_概述和软件架构](http://www.wowotech.net/pm_subsystem/cpuidle_overview.html)
    - [Device Tree（三）：代码分析](http://www.wowotech.net/device_model/dt-code-analysis.html)
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