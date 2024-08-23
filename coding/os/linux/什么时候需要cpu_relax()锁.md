# 

宋宝华 Linux内核之旅

 _2021年09月27日 11:32_

![](https://res.wx.qq.com/op_res/NN_GToMiIjsXzgPzF9-74ZzwR3cA9-fv3o9eWo8f5gQWqx71CmGlY8kFxuIxZaG0TB1bFeMCmh1DGN_pWMRg0A)

**帐号已迁移**

2篇原创内容

公众号

一个最典型的要使用pu_relax()锁的场景是忙等待（也就是死循环等一个事情的发生），在内核里面有大量的代码，比如等寄存器状态：  

![Image](https://mmbiz.qpic.cn/mmbiz_png/Ass1lsY6byvfdRKf3Kb75YMB5zRLI42icczvUbxPqrpCDjvVSbiaBNRuzSWCjDw0Y1cODnqXcmMteq6K3BPkbgsQ/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

比如做延迟：  

![Image](https://mmbiz.qpic.cn/mmbiz_png/Ass1lsY6byvfdRKf3Kb75YMB5zRLI42icpxthvKpEQpzHnQPkSaffKW0ElLyia5VYHM6NCGO5oPXfG9ZPytyc1dw/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

简单来说，你如果在内核里面写了忙等待的代码，都没有在循环里面加个cpu_relax()的话，这基本上是一种比较幼稚的表现。

  

根据内核文档volatile-considered-harmful，cpu_relax()的描述：

![Image](https://mmbiz.qpic.cn/mmbiz_png/Ass1lsY6byvfdRKf3Kb75YMB5zRLI42ickpJZDBmr8Szx5yZHtViczIwX2HKfB2DMTHMZRKIrJ6HuhemeVMuut3A/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

由此可见cpu_relax()至少具备三大功能：  

1. 帮着省电；
    
2. 如果是超线程CPU的机器，可以让渡CPU给其他的线程；因为我这个CPU目前没什么正经事情干，等的时间不如放慢节奏，让别人多干点；
    
3. 它是一个编译屏障，让volatile变地在内核基本没什么必要。
    

当然，cpu_relax()的具体实现与体系架构相关，不同的体系架构实现不一样，可能只完成了上面3个功能中的1个，2个而不是全部。

  

我们看看它在ARM64的实现：  

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

其中"memory"的部分是服务于屏障功能，而前面的yield符合SMT系统让渡的语义。在典型的超线程处理器中，每个超线程不是一个独立的core，所以两个或者多个超线程之间仍然在竞争一些资源，如果其中一个人调用了yield，那么它会在争抢中放慢节奏，而旁边的那个兄弟会抢地更多。

  

PowerPC的实现则是调节hardware multi-threading的优先级：

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

当然，硬件如果不具备这种超线程能力的话，cpu_relax()可以简单地是一个编译屏障，比如arch/alpha/include/asm/processor.h中：

```
#define cpu_relax()     barrier()
```

  

在arch/x86/include/asm/vdso/processor.h的实现中：  

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

REP NOP这种pause操作既可以省点，有可以避免忙等的CPU疯狂去抢总线访问内存。如果纯粹地不加pause暗示的忙等，疯狂执行指令，应该是很耗电的，忙等中拼命访问内存，总线冲突也大。  

  

总之，不管具体的体系架构怎么实现，忙等里面都适合加cpu_relax()，毕竟内核多数的代码要求是跨平台的。

Reads 1371

​

People who liked this content also liked

2分钟搞懂如何计算uart速率

我关注的号

一口Linux

不喜欢

不看的原因

OK

- 内容低质
- 不看此公众号内容

![](https://mmbiz.qpic.cn/sz_mmbiz_jpg/icRxcMBeJfc8DMm25x4srjKQvGmCQdE4x0qC4oicP3fjqeEViblIoqSKFzD2xLrqtUMcmbArnGibKsnrIYhuozUqzA/0?wx_fmt=jpeg)

虚拟内存、用户空间 、内核空间、用户态、内核态

我关注的号

Linux开发架构之路

不喜欢

不看的原因

OK

- 内容低质
- 不看此公众号内容

![](https://mmbiz.qpic.cn/sz_mmbiz_jpg/8pECVbqIO0wsszqy9Yd9HRlAqk50XMhaAicWG3zVOFKOuv66bsb5UuCPtIJ57VvKvMrjIbuxPKbIYjUdr4CgMkA/0?wx_fmt=jpeg)

C++那些事之小项目实战-进程间通信

我关注的号

光城

不喜欢

不看的原因

OK

- 内容低质
- 不看此公众号内容

![](https://mmbiz.qpic.cn/mmbiz_jpg/WwIcQHkD5md68tBbPeZfkvMal97PkwqFKlaSLSC0lVWvvRhIZVqTEsic5lyWouibOticjHjwALXXp62thvGxZ7qAw/0?wx_fmt=jpeg&tp=wxpic)

Comment

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/SeWfibBcBT0EibtIWVNvshnuWMN1AoJw3poFIsbpaIVyZibCCqwBUR21rcDfrQgoqYzaYNdS14IIXzvmzvibdDa5Rw/300?wx_fmt=png&wxfrom=18)

Linux内核之旅

3ShareWow

Comment

Comment

**Comment**

暂无留言