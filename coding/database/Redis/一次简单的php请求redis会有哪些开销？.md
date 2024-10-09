# [![开发内功修炼@张彦飞](https://kfngxl.cn/usr/themes/DUX/img/logo.jpg)开发内功修炼@张彦飞](https://kfngxl.cn/)

talk is cheap,\
show me the code!

- [首页](http://kfngxl.cn/index.php)
- [CPU篇](https://kfngxl.cn/index.php/category/cpu/)
- [内存篇](https://kfngxl.cn/index.php/category/memory/)
- [网络篇](https://kfngxl.cn/index.php/category/network/)
- [关于](https://kfngxl.cn/index.php/about.html)
-

# [一次简单的php请求redis会有哪些开销？](https://kfngxl.cn/index.php/archives/610/)

2024-03-28 [CPU篇](https://kfngxl.cn/index.php/category/cpu/) 阅读(217) 评论(0)

我问大家一个问题，下图中一个最简单的例子，会导致哪些CPU开销产生？你是否能够说清楚？

```
<?php  
    ... 
    $redis->get('test'); 
    ...
```

这个例子一下子就把大家在我的文章里学到的东西和你的实际工作结合起来了。怎么样，是不是足够简单？就是一句php代码从redis实例中获取一个key的value值而已，相信类似的代码你天天都在写。对这句redis get实际开销的理解水平，就代表了你的内功的深度。

在前面的文章中介绍了一些和CPU相关的硬件、内核知识，把开销大户进程上下文切换、系统调用、软中断的具体开销都挨个分析了一遍。没有看过的同学请关注并翻看我以前的文章。不过，这时候我觉得有很多开发同学都有一个疑惑，仍然是觉得：“我是应用层的开发，这么底层的开销和我有什么关系？” 或者是说：“线上服务器的运维不都应该是运维的工作吗？和开发又没关系”

我想说的是，如果你只是一个初级或者中级开发工程师，这些确实没有必要了解。但是**如果你想成为一名高级、或者是资深开发工程师，那么你应该具备大致估算你手下写出的每一行代码开销的能力，要对自己代码在线上的运行开销负责！**

接下来，我们就来带大家从更深层次的方向认识到这句简单代码的开销。为了便于测试，我们对代码进行一些简单的改造，最终的实际测试文件如下。测试代码见[test07](https://kfngxl.cn/index.php/archives/610/tests/test07/main.php)

```
<?php  
    $redis = new Redis();  
    $redis->connect({某Redis server的IP}, 6339);  

    sleep(60);    
    echo "Test begin\n";    
    for($i=0; $i<10000; $i++){  
        $redis->get('test');  
    }      
    echo "Test end!\n ";      
    sleep(60);  
```

例子非常的简单，就是一句后端同学代码里经常出现的从Redis里获取了一条数据而已。那么让我们看看它到底会产生哪些开销？

# 1.系统调用

```
# strace -c php main.php  
% time     seconds  usecs/call     calls    errors syscall  
------ ----------- ----------- --------- --------- ----------------  
 97.24    0.039698           1     30003           poll  
  2.20    0.000899           0     10003           sendto  
  0.30    0.000122           0     10000           recvfrom  
  0.13    0.000053           0     10069           gettimeofday  
  0.03    0.000013           2         6           socket  
  0.03    0.000012           0       408           munmap  
  0.02    0.000008           0       657           read  
```

我们代码所调用的get函数，其实是php的一个redis扩展提供的。该扩展又会去调用Linux系统的网络库函数，库函数再去调用内核提供的系统调用。这个调用层次模型如下：

![图1.png](https://kfngxl.cn/usr/uploads/2024/03/1340210808.png "图1.png")

**从实际测试结果可见，每次get操作都需要执行多次系统调用才可完成。**

# 2、进程上下文切换

每次调用get后，如果数据没有返回。进程都是阻塞掉的，因此还会导致进程进入主动上下文切换。 我们用实验的方式来查看一下：

```
# php main.php  
```

然后再另起一个控制台，分别赶在实验开始前和实验开始后执行如下两行命令：

```
# grep ctxt /proc/14862/status  
voluntary_ctxt_switches:        4  
nonvoluntary_ctxt_switches:     43  
  
# grep ctxt /proc/14862/status  
voluntary_ctxt_switches:        10005  
nonvoluntary_ctxt_switches:     49  
```

**每次get都会导致进程进入自愿上下文切换，在网络IO密集型的应用里自愿上下文切换要比时间片到了被动切换要多的多！**

# 3、软中断

每次在redis服务器返回数据的时候，网卡都会通过软中断的方式来让内核处理数据包。因此

```
# cat /proc/softirqs  
                CPU0       CPU1       CPU2       CPU3  
      HI:          0          0          0          0  
   TIMER:  196173081  145428444  154228333  163317242  
  NET_TX:          0          0          0          0  
  NET_RX:  178159928     116073      10108     160712  

     
# cat /proc/softirqs  
                CPU0       CPU1       CPU2       CPU3  
      HI:          0          0          0          0  
   TIMER:  196173688  145428634  154228610  163317624  
  NET_TX:          0          0          0          0  
  NET_RX:  178170212     116073      10108     160712  
```

该虚机的软中断亲和性在CPU0上，178170212-178159928 = 10284（多出来的284是机器上其它的小服务）。\
**每次get请求收到数据返回的时候，内核必须要支出一次软中断的开销！**

# 总结

看似一次非常简单的redis get操作就会把所有系统态的高开销操作都涉及到了。一次进程上下文切换、一次软中断、若干次系统调用。在经过了前面多篇文章的历练，相信大家对它们的开销有了一个量化的拿捏。其实除了上面这些容易评估到的开销外，还有如L1、L2 cache miss，以及TLB cache miss这些在进程被切换掉都会造成cache命中率下降，也会导致额外开销。

所以，你以为的简单，其实不一定简单！

写在最后，由于我的这些知识在公众号里文章比较分散，很多人似乎没有理解到我对知识组织的体系结构。而且图文也不像视频那样理解起来更直接。所以我在知识星球上规划了视频系列课程，包括**硬件原理、内存管理、进程管理、文件系统、网络管理、Golang语言、容器原理、性能观测、性能优化九大部分大约 120 节内容**，每周更新。加入方式参见[我要开始搞知识星球啦](https://mp.weixin.qq.com/s/_8ux274sY-As__Xwoqmewg)、[如何才能高效地学习技术,我投“融汇贯通”一票](https://mp.weixin.qq.com/s/z82z9jqnt08gBLYGxLHY2g)

Github：[https://github.com/yanfeizhang/coder-kung-fu](https://github.com/yanfeizhang/coder-kung-fu)\
关注公众号：微信扫描下方二维码\
![qrcode2_640.png](https://kfngxl.cn/usr/uploads/2024/05/4275823318.png "qrcode2_640.png")

本原创文章未经允许不得转载 | 当前页面：[开发内功修炼@张彦飞](https://kfngxl.cn/) » [一次简单的php请求redis会有哪些开销？](https://kfngxl.cn/index.php/archives/610/)

标签：没有标签

![张彦飞（@开发内功修炼）](https://secure.gravatar.com/avatar/23c60606a05a1e9b9fac9cadbd055ad7?s=50&r=g)

#### 张彦飞（@开发内功修炼）

腾讯、搜狗、字节等公司13年工作经验，著有《深入理解Linux网络》一书，个人技术公众号「开发内功修炼」

上一篇：[一次系统调用开销到底有多大？](https://kfngxl.cn/index.php/archives/608/ "一次系统调用开销到底有多大？")下一篇：[协程究竟比线程牛在什么地方？](https://kfngxl.cn/index.php/archives/611/ "协程究竟比线程牛在什么地方？")

### 相关推荐

- [C语言竟可以调用Go语言函数，这是如何实现的？](https://kfngxl.cn/index.php/archives/810/ "C语言竟可以调用Go语言函数，这是如何实现的？")
- [理解内存的Rank、位宽以及内存颗粒内部结构](https://kfngxl.cn/index.php/archives/798/ "理解内存的Rank、位宽以及内存颗粒内部结构")
- [看懂服务器 CPU 内存支持，学会计算内存带宽](https://kfngxl.cn/index.php/archives/787/ "看懂服务器 CPU 内存支持，学会计算内存带宽")
- [磁盘开篇：扒开机械硬盘坚硬的外衣！](https://kfngxl.cn/index.php/archives/774/ "磁盘开篇：扒开机械硬盘坚硬的外衣！")
- [经典，Linux文件系统十问](https://kfngxl.cn/index.php/archives/769/ "经典，Linux文件系统十问")
- [Linux进程是如何创建出来的？](https://kfngxl.cn/index.php/archives/687/ "Linux进程是如何创建出来的？")
- [内核是如何给容器中的进程分配CPU资源的？](https://kfngxl.cn/index.php/archives/752/ "内核是如何给容器中的进程分配CPU资源的？")
- [Docker容器里进程的 pid 是如何申请出来的？](https://kfngxl.cn/index.php/archives/745/ "Docker容器里进程的 pid 是如何申请出来的？")

### 标签云

[内存硬件 （1）](https://kfngxl.cn/index.php/tag/%E5%86%85%E5%AD%98%E7%A1%AC%E4%BB%B6/)[服务器 （1）](https://kfngxl.cn/index.php/tag/%E6%9C%8D%E5%8A%A1%E5%99%A8/)[技术面试 （1）](https://kfngxl.cn/index.php/tag/%E6%8A%80%E6%9C%AF%E9%9D%A2%E8%AF%95/)[同步阻塞 （1）](https://kfngxl.cn/index.php/tag/%E5%90%8C%E6%AD%A5%E9%98%BB%E5%A1%9E/)[进程 （1）](https://kfngxl.cn/index.php/tag/%E8%BF%9B%E7%A8%8B/)

- 最新文章

- - 06-13[C语言竟可以调用Go语言函数，这是如何实现的？](https://kfngxl.cn/index.php/archives/810/ "C语言竟可以调用Go语言函数，这是如何实现的？")
    - 05-13[理解内存的Rank、位宽以及内存颗粒内部结构](https://kfngxl.cn/index.php/archives/798/ "理解内存的Rank、位宽以及内存颗粒内部结构")
    - 05-13[看懂服务器 CPU 内存支持，学会计算内存带宽](https://kfngxl.cn/index.php/archives/787/ "看懂服务器 CPU 内存支持，学会计算内存带宽")
    - 04-09[磁盘开篇：扒开机械硬盘坚硬的外衣！](https://kfngxl.cn/index.php/archives/774/ "磁盘开篇：扒开机械硬盘坚硬的外衣！")
    - 04-08[经典，Linux文件系统十问](https://kfngxl.cn/index.php/archives/769/ "经典，Linux文件系统十问")

- 站点统计

- - 文章总数：87篇
    - 分类总数：3个
    - 总访问量：36931次
    - 本站运营：0年168天18小时

© 2010 - 2024 [开发内功修炼@张彦飞](https://kfngxl.cn/) | [京ICP备2024054136号](http://beian.miit.gov.cn/)\
本站部分图片、文章来源于网络，版权归原作者所有，如有侵权，请联系我们删除。
