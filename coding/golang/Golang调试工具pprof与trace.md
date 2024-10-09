原创 lishinho lishinho杂货铺

_2022年06月19日 19:31_ _上海_

Go语言原生支持对于程序运行时重要指标和特征的分析，以及事件的追踪。

pprof工具可实现对于指标和特征的分析，通过分析可以查找到程序中的错误，比如内存泄露，race冲突，协程泄漏，cpu利用率过高等；也可以对程序进行优化。

trace工具是以事件为基础的信息追踪，可反映一段时间内程序的变化，比如频繁的gc及协程调度等。

在生产环境中，由于go语言运行时的指标不对外暴露，需要通过一系列工具分析查询。而且在很多情形下，我们往往也不会直接通过http接口调用的方式在生产环境中直接分析特征文件，需要将特征文件取样下载到本地之后再解析分析。

01

—

关于pprof

pprof是谷歌推出的golang可视化和分析数据的工具，该工具在golang安装时即存在：https://github.com/google/pprof/blob/main/doc/README.md

pprof 读取 profile.proto 格式的分析样本集合并生成报告以可视化和帮助分析数据，生成文本和图形报告。pprof提供的功能在 runtime/pprof 的包中, 也提供了 http的接口, 参考 net/http/pprof。

pprof的总的入口是http://host:port/debug/pprof , 内存, cpu, mutex 和 block 的检测都是在这个入口的分类。

#堆内存特征分析

在容器里获取特征文件：

```
curl -o heap.out http://localhost:{port}/debug/pprof/heap
```

下载到本地后用pprof工具分析

```
go tool pprof heap.out
```

当执行pprof分析堆内存的特征文件时，默认的类型为inner_space，代表分析程序正在使用的内存。

全部支持的类型如下：

```
inuse_space: 正在使用的内存大小
```

如果是内存泄露, 可以根据 inner_space 的占比和变化判断, 但是也不是万能的, 有些时候, 一些 inuse_space 是因为高流量导致的. 对于频繁创建的一些对象 可以在 alloc_space和 alloc_object 判断, 这种场景可以通过复用对象池来减少没意义的分配。

下面介绍一些常用的指令：

top会列出默认以flat列排序的序列：

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

flat指当前函数统计的值，比如在inner_space模式下，则代表当前函数分配的正在使用的堆内存大小。

cum指当前函数及其调用的一系列的flat的和。

flat只包含当前函数的栈帧信息，不包括其调用的函数的栈帧信息。

如果切到了alloc_objects模式，所以表示分配对象的次数。

通过指定-cum参数，top指令可以通过对比得到大致的问题函数调用关系：

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

通过list指令可以找出函数具体内存/对象分配发生在哪一行。

通过tree指令可以打印函数的调用链，能够得到具体函数的调用堆栈信息。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

以上可以看出，teenagersUpdate函数频繁的创建出一些对象，其调用和占总分配次数的30%左右,并可以看出在调用栈的哪里可能有问题。

在pprof交互行中输入help，可以看到更多指令和使用方法：

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

#CPU分析

在容器里获取特征文件：

```
curl -o profile.out http://localhost:{port}/debug/pprof/profile
```

有时候我们希望指定采样的时候, 可以在 http的请求参数后面添加: ?seconds=60

下载到本地后用pprof工具分析

```
go tool pprof profile.out
```

和堆内存同样的指令，分析cpu文件:

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

top20可以看占用cpu的前20位函数，tree可以打出调用栈

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

cpu和内存的情况都可以搭配火焰图一起看。

#其它可以通过pprof分析的场景

pprof协程栈分析：

```
go tool pprof http://localhost:{port}/debug/pprof/goroutine
```

base基准分析：

```
go tool pprof -base pprof.goroutine.001.pb.gz pprof.goroutine.002.pb.gz 
```

mutext堵塞分析

```
go tool pprof http://localhost:{port}/debug/pprof/mutex
```

#火焰图和性能图

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

火焰图

从Go11之后，火焰图已经内置到pprof分析工具中，用于分析堆内存和cpu的使用情况,分析要点：

1.最下方的root代表程序的开始，其它的框都代表一个函数

2.火焰图每一层中的函数都是平级的，下层函数是其对应的上层函数的字函数。

3.函数调用栈越长，火焰就越高

4.框越长，颜色越深，代表当前函数占用内存越多

5.可以单击任何框，查看该函数更详细的信息

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

性能图

关于节点的说明：

红色节点代表当前cum值很大，灰色节点代表当前cum值很小

节点里较大的字体表示较大的当前值，较小的字体表示较小当前值

关于箭头的说明：

箭头越粗代表当前的路径消耗了越多的资源，箭头越细代表当前的路径消耗越少的资源。

虚线箭头表示两个节点间的某些节点已被忽略，为间接调用；实线箭头表示两个节点之间为直接调用。

02

—

关于trace

trace工具可以查看信息包括：协程的创建开始和结束，协程的堵塞（系统调用，锁，通道），网络IO相关时间，GC相关事件，系统调用事件等。

获取trace文件：

```
curl -o trace.out http://localhost:{port}/debug/pprof/trace?seconds=30s
```

使用trace工具对获取文件分析：

```
go tool trace trace.out
```

执行后会默认打开浏览器，显示超链接信息：

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

这里简述两个比较常用的，goroutine analysis可以看在取样时间内的以方法维度的goroutine集合：

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

选中点击之后能看到该方法创建的gouroutine的信息综述：

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

选中其中一个goroutine可以看到针对该go routine的view trace信息可视化界面：

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

在view trace链接跳转的界面可以追溯该goroutine出现的上下文，究竟发生了什么：

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

界面的几个信息说明：

title:协程名字

start:协程开始的时间

wall duration:协程持续时间

start stack trace:协程开始时的栈追踪

end stack trace:协程结束时的栈追踪

event:协程产生的事件信息

同时也可以通过view trace界面分析gc的次数以及异常情况。

![](https://mmbiz.qlogo.cn/mmbiz_jpg/FmrqRd6Iib5AEBwCF3o02u0DOKRyz8EiaEqbpo8xGMXFDy624v1qwvyeDCdhhYTia3wursqFXdtuqrzQicdomuImIQ/0?wx_fmt=jpeg)

lishinho

![赞赏二维码](https://mp.weixin.qq.com/s?__biz=MzI2MjUzMDY4Nw==&mid=2247484526&idx=1&sn=57b539b146edb77a23f1a196a4cc4a80&chksm=ea48f77edd3f7e6868ba82987cc02088e14ce9a8c68ced85bf40c99cc8e8b5d7250c18af5a49&mpshare=1&scene=24&srcid=0619fUTgBvFF7gjbTEQJ8vBx&sharer_sharetime=1655638737210&sharer_shareid=8397e53ca255d0bca170c6327d62b9af&key=daf9bdc5abc4e8d04561728726fa7ca8bad40fa3299fb3eaf17fecb69ccbb7a5102c98a13f4032e709ce28c31383472adb0d84fd56beefc5e245d228f6252ca6e8a798f4322d547216e10910a0a7e81236b5cbd79747b1a556b3780af28aed0dde2134d3700134343bf5c799b6f179418c2b14a517cabb4a9e2aca823e25c36c&ascene=0&uin=MTEwNTU1MjgwMw%3D%3D&devicetype=Windows+11+x64&version=63090b19&lang=zh_CN&countrycode=CN&exportkey=n_ChQIAhIQnYoxjfwh6msVjXGxEs9XUBLmAQIE97dBBAEAAAAAANxJMPNO8xQAAAAOpnltbLcz9gKNyK89dVj0cuNn5aYGupHOiZgDLzxW8KSzxy3yo3WQ8Ppkl%2FIq81q8uG2%2F0JvYMlmUxiNd93x8EqyGiTzNnVH4LmHHC5aTAZxu4KukMgDbK1gXiV8Wg3vCFNln92btZCHagcC1P%2F4ux1%2BAS6FTXB9sUi%2FUcGfu3e8AyVoPnOdziGuPjIA77wx5PAABQhTnjIqXRCDSXLiOnDcRAOn4dgQcxh010%2Bya%2FICwG%2Fd6s208l7vjDjc6kOtt5EZvmboZKDa1kgQscV36&acctmode=0&pass_ticket=oq7Ip9GxRc6anhFgCAMS7Eh4aBpx47tHHQ7SEOPZQiQiXvCxPa%2B030ZKfNtOn7eU&wx_header=1&fasttmpl_type=0&fasttmpl_fullversion=7350504-zh_CN-zip&fasttmpl_flag=1)喜欢作者

golang54

技术92

协程6

golang · 目录

上一篇当golang struct遇上mutex下一篇golang sync.pool浅析

阅读 119

​

发消息

[](javacript:;)

![](http://mmbiz.qpic.cn/sz_mmbiz_png/rKRiaIVfYTgDUu5MplxMVVVOBKuYZjtyNJtmiaqQTibRlI45ZibUyGo8aCyGrtyf6U3cVoagja65icOIhFfwI1dvnSA/300?wx_fmt=png&wxfrom=18)

lishinho杂货铺

221

发消息
