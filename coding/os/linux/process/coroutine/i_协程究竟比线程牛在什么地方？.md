
2024-03-28 [CPU篇](https://kfngxl.cn/index.php/category/cpu/) 阅读(222) 评论(0)

前文中中我们用实验的方式验证了Linux进程和线程的上下文切换开销，大约是3-5us之间。当运行在一般的计算机程序时，这个开销确实不算大。但是海量互联网服务端和一般的计算机程序相比，特点是：

- 高并发：每秒钟需要处理成千上万的用户请求
- 周期短：每个用户处理耗时越短越好，经常是ms级别的
- 高网络IO：经常需要从其它机器上进行网络IO、如Redis、Mysql等等
- 低计算：一般CPU密集型的计算操作并不多

即使3-5us的开销，如果上下文切换量特别大的话，也仍然会显得是有那么一些性能低下。例如之前的Web Server之Apache，就是这种模型下的软件产品。（其实当时Linux操作系统在设计的时候，目标是一个通用的操作系统，并不是专门针对服务端高并发来设计的）

为了避免频繁的上下文切换，还有一种异步非阻塞的开发模型。那就是用一个进程或线程去接收一大堆用户的请求，然后通过IO多路复用的方式来提高性能（进程或线程不阻塞，省去了上下文切换的开销）。Nginx和Node Js就是这种模型的典型代表产品。平心而论，从程序运行效率上来，这种模型最为机器友好，运行效率是最高的（比下面提到的协程开发模型要好）。所以Nginx已经取代了Apache成为了Web Server里的首选。但是这种编程模型的问题在于开发不友好，说白了就是过于机器化，离进程概念被抽象出来的初衷背道而驰。人类正常的线性思维被打乱，应用层开发们被逼得以非人类的思维去编写代码，代码调试也变得异常困难。

于是就有一些聪明的脑袋们继续在应用层又动起了主意，设计出了不需要进程/线程上下文切换的“线程”，协程。用协程去处理高并发的应用场景，既能够符合进程涉及的初衷，让开发者们用人类正常的线性的思维去处理自己的业务，也同样能够省去昂贵的进程/线程上下文切换的开销。因此可以说，协程就是Linux处理海量请求应用场景里的进程模型的一个很好的的补丁。

背景介绍完了，那么我想说的是，毕竟协程的封装虽然轻量，但是毕竟还是需要引入了一些额外的代价的。那么我们来看看这些额外的代价具体多小吧。

# 协程开销测试

## 1、协程切换CPU开销

测试代码参见[test05](https://kfngxl.cn/index.php/archives/611/tests/test05/src/main/main.go)，测试过程是不断在协程之间让出CPU。核心代码如下：

```go
func cal()  {
    for i :=0 ; i<1000000 ;i++{
        runtime.Gosched()
    }
}

func main() {
    runtime.GOMAXPROCS(1)

    currentTime:=time.Now()
    fmt.Println(currentTime)

    go cal()  
    for i :=0 ; i<1000000 ;i++{
        runtime.Gosched()
    }

    currentTime=time.Now()
    fmt.Println(currentTime)
}
```

好了，让我们编译运行一下：

```c
# cd tests/test05/src/main/;  
# go build  
# ./main  
2019-08-08 22:35:13.415197171 +0800 CST m=+0.000286059
2019-08-08 22:35:13.655035993 +0800 CST m=+0.240124923
```

平均每次协程切换的开销是（655035993-415197171)/2000000=120ns。相对于前面文章测得的进程切换开销大约3.5us，大约是其的三十分之一。比系统调用的造成的开销还要低。

## 2、协程内存开销

在空间上，协程初始化创建的时候为其分配的栈有2KB。而线程栈要比这个数字大的多，可以通过ulimit 命令查看，一般都在几兆，作者的机器上是10M。如果对每个用户创建一个协程去处理，100万并发用户请求只需要2G内存就够了，而如果用线程模型则需要10T。

```c
# ulimit -a  
stack size              (kbytes, -s) 10240  
```

# 本节结论

协程由于是在用户态来完成上下文切换的，所以切换耗时只有区区100ns多一些，比进程切换要高30倍。单个协程需要的栈内存也足够小，只需要2KB。所以，近几年来协程大火，在互联网后端的高并发场景里大放光彩。

无论是空间还是时间性能都比进程（线程）好这么多，那么Linus为啥不把它在操作系统里实现了多好？

实际上协程并不是一个新玩意，在上个世纪60年代的时候就已经有人提出了。操作系统的一个主要设计目标是实时性，对优先级比较高的进程是会抢占当前占用CPU的进程。但是协程无法实现这一点，还得依赖于使用CPU的一方主动释放，与操作系统的实现目的不相吻合。协程的高效是以牺牲了可抢占性为代价的。

> 扩展：由于go的协程调用起来太方便了，所以一些go的程序员就很随意地go来go去。要知道go这条指令在切换到协程之前，得先把协程创建出来。而一次创建加上调度开销就涨到400ns，差不多相当于一次系统调用的耗时了。虽然协程很高效，但是也不要乱用，否则go祖师爷Rob Pike花大精力优化出来的性能，被你随意一go又给葬送掉了。

写在最后，由于我的这些知识在公众号里文章比较分散，很多人似乎没有理解到我对知识组织的体系结构。而且图文也不像视频那样理解起来更直接。所以我在知识星球上规划了视频系列课程，包括**硬件原理、内存管理、进程管理、文件系统、网络管理、Golang语言、容器原理、性能观测、性能优化九大部分大约 120 节内容**，每周更新。加入方式参见[我要开始搞知识星球啦](https://mp.weixin.qq.com/s/_8ux274sY-As__Xwoqmewg)、[如何才能高效地学习技术,我投“融汇贯通”一票](https://mp.weixin.qq.com/s/z82z9jqnt08gBLYGxLHY2g)

Github：[https://github.com/yanfeizhang/coder-kung-fu](https://github.com/yanfeizhang/coder-kung-fu)\
