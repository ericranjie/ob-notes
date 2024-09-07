# [![开发内功修炼@张彦飞](https://kfngxl.cn/usr/themes/DUX/img/logo.jpg)开发内功修炼@张彦飞](https://kfngxl.cn/)

talk is cheap,  
show me the code!

-  [首页](http://kfngxl.cn/index.php)
-  [CPU篇](https://kfngxl.cn/index.php/category/cpu/)
-  [内存篇](https://kfngxl.cn/index.php/category/memory/)
-  [网络篇](https://kfngxl.cn/index.php/category/network/)
-  [关于](https://kfngxl.cn/index.php/about.html)
- 

# [一个likely的简单使用，背后却是对CPU工作原理的深刻理解](https://kfngxl.cn/index.php/archives/618/)

2024-03-28 [CPU篇](https://kfngxl.cn/index.php/category/cpu/) 阅读(179) 评论(0)

大家好，我是飞哥！

今天我给大家分享一个内核中常用的提升性能的小技巧。

在内核中很多地方都充斥着 likely、unlikely 这一对儿函数的使用。随便揪两处，比如在 TCP 连接建立的过程中的这两个函数。

```
//file: net/ipv4/tcp_ipv4.c
int tcp_v4_conn_request(struct sock *sk, struct sk_buff *skb)
{
    if (likely(!do_fastopen)) 
        ...
}
//file: net/ipv4/tcp_input.c
int tcp_rcv_established(struct sock *sk, ...)
{
    if (unlikely(sk->sk_rx_dst == NULL))
        ......
}
```

在刚开始看源码的时候，我也没弄懂这一对儿函数的玄机。后来才搞懂它原来对性能提升是有帮助的。今天我就来和大家聊聊 likely、unlikely 是如何帮助性能提升的。

### 1.likely 和 unlikely

咱们先来挖挖这对儿函数的底层实现。

```
//file: include/linux/compiler.h
#define likely(x)   __builtin_expect(!!(x),1)
#define unlikely(x) __builtin_expect(!!(x),0)
```

可以看到其实它们就是对 \_\_builtin_expect 的一个封装而已。\_\_builtin_expect 这个指令是 gcc 引入的。该函数作用是允许程序员将最有可能执行的分支告诉编译器，来辅助系统进行分支预测。(参见 [https://gcc.gnu.org/onlinedocs/gcc/Other-Builtins.html](https://gcc.gnu.org/onlinedocs/gcc/Other-Builtins.html))

它的用法为：\_\_builtin_expect(EXP, N)。意思是：EXP == N的概率很大。那么上面 likely 和 unlikely 这两句的具体含义就是：

- \_\_builtin_expect(!!(x),1) x 为真的可能性更大
- \_\_builtin_expect(!!(x),0) x 为假的可能性更大

当正确地使用了\_\_builtin_expect后，编译器在编译过程中会根据程序员的指令，将可能性更大的代码紧跟着前面的代码，从而减少指令跳转带来的性能上的下降。

### 2.实际动手验证

那我们实际动手写两个例子，来窥探一下编译器是如何进行优化的。为此我写了一段简单的测试代码，如下：

```
#include <stdio.h>
#define likely(x) __builtin_expect(!!(x), 1)
#define unlikely(x)  __builtin_expect(!!(x), 0)

int main(int argc, char *argv[])
{
    int n;
    n = atoi(argv[1]);

    if (likely(n == 10)){
        n = n + 2;
    } else {
        n = n - 2;
    }
    printf("%d\n", n);
    return 0;
}
```

这段代码非常简单，对用户的输入做一个 if else 判断。在 if 中使用了 likely，也就是假设这个条件为真的概率更大。那么我们来看看它编译后的汇编码来看看。

![图1.png](https://kfngxl.cn/usr/uploads/2024/03/3906238526.png "图1.png")

图中上面红框内是对 if 的汇编结果，可见它使用的是 jne 指令。它的作用是看它的上一句比较结果，如果不相等则跳转到 4004bc 处，相等的话则继续执行后面的指令。

在 jne 指令后面紧挨着的是 n = n + 2 对应的汇编代码，  
也就是说它把 `n = n + 2` 这段代码逻辑放在了紧挨着自己身边。而把 n = n - 2 的执行逻辑放在了离当前指令较远的 4004bc 处。 

我们再把 likey 换成 unlikey 看一下，发现结果是正好相反，这次把 n = n - 2 的执行逻辑放在前面了，如图。

![图2.png](https://kfngxl.cn/usr/uploads/2024/03/3303059834.png "图2.png")

注意，编译时需要加 -O2 选项，使用 objdump -S 来查看汇编指令。为了方便大家使用，我把它写到 makefile 里了，和测试代码一起放到了咱们的 Github 上了。大家想动手的可以玩玩，地址： [https://github.com/yanfeizhang/coder-kung-fu/tree/main/tests/cpu/test01](https://github.com/yanfeizhang/coder-kung-fu/tree/main/tests/cpu/test01)

### 3.性能提升原理

那这么做为什么就能够提升程序运行性能呢？原因有两个。

首先第一个原因就是 CPU 高速缓存。现代的 CPU 一般都有三级的缓存，用来解决内存访问过慢的问题。访问数据时优先从 L1 读取，读取不到再读 L2、还读不到就读 L3，最后用内存兜底。

![图3.png](https://kfngxl.cn/usr/uploads/2024/03/2734401235.png "图3.png")

图中的数据来源于飞哥早期的一次测试，参见[实际内访问延时](https://mp.weixin.qq.com/s/_-Ar944KlztzmFYdA3JXnQ)

这里要注意的是，每一次缓存数据的单位都是以一个 CacheLine **64 字节**为单位进行存储的。假如说要查询的数据在 L1 中不存在，那么 CPU 的做法是一次性从 L2 中把要访问的数据及其后面的 **64 个字节**全部缓存进来。

假如下一次再执行的时候要访问的指令在上一次已经在 L1 中存在了，那么就直接访问 L1，就不必再从 L2 来读取了。回到上面的例子，假如说我们执行完了 cmp 对比指令以后，判断确实是要进加法分支，那紧接着就会执行 jne 后面的 mov xor 等指令大概率就可以从 L1 中取到了。 

![图1.png](https://kfngxl.cn/usr/uploads/2024/03/1546856130.png "图1.png")

假如说 cmp 对比执行后，发现是要跳到 4004bc 处的指令执行。那大概率该位置处的指令还没有被加载到缓存中（实践中一个分支下可能会包含很多代码，而不是像我这个例子中简单的两三行），就避免不了从 L2 L3 甚至是速度更慢的 L3 去读取指令。

除了高速缓存这个原因以外，还有一个更底层的原理。那就是 CPU 的流水线技术。CPU 在执行程序指令的时候，并不是先执行一个，执行完了再运行下一个这样的。而是把每个指令都分成了多个阶段，并让不同指令的各步操作重叠，从而实现几条指令并行处理。

还拿上面编译出来的汇编码来举例，程序中 cmp、jne、mov 几个指令是挨着的，那么 CPU 在执行的时候实际是大概这样的。

![图4.png](https://kfngxl.cn/usr/uploads/2024/03/4136447205.png "图4.png")

当 jne 指令正在执行的时候，后面的两个 mov 指令都已经分别进入到译码和取址阶段了都。假如说分支预测失败，那么这工作就白干了。

### 4.小结

总结一下，今天分享的 likely 和 unlikely​ 其实是属于是辅助 CPU 分支预测的性能优化方法。这就是 likely 和 unlikely 背后的这点小秘密。它是为了让 CPU 的高速缓存命中率更高，同时也为了让 CPU 的流水线更好地工作。​

Linux 作为一个基础程序，在性能上真的是考虑到了极致。内核的作者们内功都是非常的深厚，都深谙计算机的底层工作原理。为了极致的性能追求精心打磨每一个细节，非常值得我们学习和借鉴。

不过这里要说的一点是，likely 和 unlikely 的概率判断务必准确。如果写反了，反而会起到反作用。  
今天分享到此结束，求转发！

写在最后，由于我的这些知识在公众号里文章比较分散，很多人似乎没有理解到我对知识组织的体系结构。而且图文也不像视频那样理解起来更直接。所以我在知识星球上规划了视频系列课程，包括**硬件原理、内存管理、进程管理、文件系统、网络管理、Golang语言、容器原理、性能观测、性能优化九大部分大约 120 节内容**，每周更新。加入方式参见[我要开始搞知识星球啦](https://mp.weixin.qq.com/s/_8ux274sY-As__Xwoqmewg)、[如何才能高效地学习技术,我投“融汇贯通”一票](https://mp.weixin.qq.com/s/z82z9jqnt08gBLYGxLHY2g)

Github：[https://github.com/yanfeizhang/coder-kung-fu](https://github.com/yanfeizhang/coder-kung-fu)  
关注公众号：微信扫描下方二维码  
![qrcode2_640.png](https://kfngxl.cn/usr/uploads/2024/05/4275823318.png "qrcode2_640.png")

本原创文章未经允许不得转载 | 当前页面：[开发内功修炼@张彦飞](https://kfngxl.cn/) » [一个likely的简单使用，背后却是对CPU工作原理的深刻理解](https://kfngxl.cn/index.php/archives/618/)

标签：没有标签

![张彦飞（@开发内功修炼）](https://secure.gravatar.com/avatar/23c60606a05a1e9b9fac9cadbd055ad7?s=50&r=g)

#### 张彦飞（@开发内功修炼）

腾讯、搜狗、字节等公司13年工作经验，著有《深入理解Linux网络》一书，个人技术公众号「开发内功修炼」

上一篇：[函数调用太多了会有性能问题吗？](https://kfngxl.cn/index.php/archives/612/ "函数调用太多了会有性能问题吗？")下一篇：[Linux 中 CPU 利用率是如何算出来的？](https://kfngxl.cn/index.php/archives/631/ "Linux 中 CPU 利用率是如何算出来的？")

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
    - 总访问量：36946次
    - 本站运营：0年168天19小时

© 2010 - 2024 [开发内功修炼@张彦飞](https://kfngxl.cn/) | [京ICP备2024054136号](http://beian.miit.gov.cn/)  
本站部分图片、文章来源于网络，版权归原作者所有，如有侵权，请联系我们删除。

- ###### 去顶部