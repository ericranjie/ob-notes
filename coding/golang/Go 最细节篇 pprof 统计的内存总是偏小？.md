# 
原创 奇伢 奇伢云存储

 _2022年03月28日 07:46_

  

![](https://res.wx.qq.com/op_res/NN_GToMiIjsXzgPzF9-74ZzwR3cA9-fv3o9eWo8f5gQWqx71CmGlY8kFxuIxZaG0TB1bFeMCmh1DGN_pWMRg0A)

**帐号已迁移**

1篇原创内容

公众号

坚持思考，就会很酷

  

![图片](https://mmbiz.qpic.cn/mmbiz_png/4UtmXsuLoNd3EIeY1wNNjjWValiaiacmOXLwbEMKfbVKXNiaS41MM2QgInxYAtyQ8h2TBaWo69kDLmuMg8RBiaMRJQ/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

  

  

  

![图片](https://mmbiz.qpic.cn/mmbiz_png/gBJOYg4SEEUvEqC5oO30octlCXHubelsesdolc6YJHWRf1C8UTPs3scrsXWFoCHYsntUfbpKFrDicJ5tFbLPxBw/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

Go 的内存泄漏

![图片](https://mmbiz.qpic.cn/mmbiz_png/gBJOYg4SEEUvEqC5oO30octlCXHubelsl8Q8jryjW8fJDBj8r4A48RcHDMj7ibJfRRhRWSJUUWNTqiaibDr9dmstQ/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

  

内存泄漏通常在 c/c++ 等语言常见，手工管理内存对程序猿的编程能力有较高要求。最常见的就是**分配**和**释放**没有配对使用。  

Go 是一门带 Gc 的语言，内存分配位置由编译器推断是在栈还是堆上，内存分配完全由 Go 本身把控，程序猿无法介入。程序猿在前端触发分配，后端的 runtime 的 GC 任务则不断的回收内存，从而达到一个平衡。理论上是不存在常规意义的内存泄漏的。但在程序中，还是经常见到内存占用持续升高的场景，今天就是来分享这类场景的思考。

**Go 的内存问题更多的是内存对象的不合理使用**，比如一个全局的 map ，程序猿 A 不知什么原因持续往里面添加元素，从来不删。这就是一个典型的内存泄漏（或者叫做内存不合理占用）。也就是说，Go 的内存问题基本都是**不合理的业务逻辑导致的**。

一般来说，这类内存问题其实非常好排查，怎么排查？

**使用 Go 自带的 pprof 工具**。

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

运用 pprof 利器

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

 **1**   **开启 pprof 端口**

  

导入 pprof 包即可，然后开启一个监听端口：  

`import "net/http"   import _ "net/http/pprof"      func main() {       // ...       go func() {           http.ListenAndServe("0.0.0.0:8080", nil)       }()       // ...   }      `

###   

 **2**   **pprof 排查姿势**

  

把程序运行起来，然后直接通过网络接口拿到 pporf 的数据，很方便。

`go tool pprof http://127.0.0.1:8080/debug/pprof/allocs   `

登陆之后 top 就能看到堆栈：

`root@ubuntu20:~# go tool pprof http://127.0.0.1:8080/debug/pprof/allocs      (pprof) top   Showing nodes accounting for 1468.90MB, 99.79% of 1471.93MB total   Dropped 22 nodes (cum <= 7.36MB)         flat  flat%   sum%        cum   cum%    1270.89MB 86.34% 86.34%  1468.90MB 99.79%  main.main        198MB 13.45% 99.79%      198MB 13.45%  encoding/base64.(*Encoding).EncodeToString            0     0% 99.79%      198MB 13.45%  main.EncodeGid            0     0% 99.79%  1469.90MB 99.86%  runtime.main   `

###   

 **3**   **pprof 原理其实很简单**

  

pprof 实现的原理其实很简单，就是分配释放的地方做好打点，最好就是能把分配的路径记录下来。举个例子，A 对象**分配路径**是：main -> fn1 -> fn2 -> fn3 （ 详细的原理可以参考文章：[Go内存管理深度细节](http://mp.weixin.qq.com/s?__biz=Mzg3NTU3OTgxOA==&mid=2247486735&idx=1&sn=a855d4198ee87d1db3da9029ae2c3bc9&chksm=cf3e1dcaf84994dc2966238034e585c58a9bf9e49b8330caaa8ec85c3617435bc6165d715ae5&scene=21#wechat_redirect) ），那么 go 把这个栈路径记录下来即可。这个事情就是在 mallocgc 中实现，每一次的分配都会判断是否要统计采样。

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

pprof 统计到的比 top 看到的要小？

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

主要有几个重要原因：1）pprof 有采样周期，2）管理内存+内存碎片，3）cgo 分配的内存 。

  

 **1**   **pprof 有采样周期**

  

pprof 统计到的比实际的小，**最重要的原因就是：采样的频率间隔导致的。**

**采样是有性能消耗的。** 毕竟是多出来的操作，每次还要记录堆栈开销是不可忽视的。所以只能在采样的频率上有个权衡，mallocgc 采样默认是 512 KiB，也就是说，进程每分配满 512 KiB 的数据才会记录一次分配路径。

`// runtime/mprof.go   var MemProfileRate int = 512 * 1024      // runtime/malloc.go   func mallocgc(size uintptr, typ *_type, needzero bool) unsafe.Pointer {    // ...       if rate := MemProfileRate; rate > 0 {           if rate != 1 && size < c.next_sample {            // 累积采样               c.next_sample -= size           } else {      // 内存的分配采样入口               profilealloc(mp, x, size)           }       }       // ...   }   `

所以这个采样的频率就会导致 pprof 看到的分配、在用内存都比实际的物理内存要小。

**有办法影响到加快采样的频率吗？**

Go 进程加载的时候，是有机会机会修改这个值的，通过 GODEBUG 来设置 memprofilerate 的值，就会覆盖 MemProfileRate 的默认值。

`func parsedebugvars() {          for p := gogetenv("GODEBUG"); p != ""; {           // ...           if key == "memprofilerate" {               if n, ok := atoi(value); ok {                   MemProfileRate = n               }           } else {               // ...           }       }   }   `

改成 1 的话，每一次分配都要被采样，采样的全面但是性能也是最差的。

  

 **2**   **内存有碎片率**

  

Go 使用的是 tcmalloc 的内存分配的模型，把内存 page 按照固定大小划分成小块。这种方式解决了外部碎片，但是小块内部还是有碎片的，这个 gap 也是内存差异的一部分。tcmalloc 内部碎片率整体预期控制在 12.5% 左右。

  

 **3**   **runtime 管理内存**

  

Go 运行过程中会采样统计内存的分配路径情况，还会时刻关注整体的内存数值，通过 runtime.ReadMemStats 可以获取得到。里面的信息非常之丰富，基本上看一眼就知道内存的一个大概情况，**memstat 字段详情（建议收藏）**：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

其中，**内存的 gap 主要来源于**：

1. heap 上 Idle span，分配了但是未使用的（往往出现这种情况是一波波的请求峰值导致的，冲上去就一时半会不下来）；
    
2. stack 的内存占用；
    
3. OS 分配但是是 reserved 的；
    
4. runtime 的 Gc 元数据，mcache，mspan 等管理内存；
    

这部分可大可小，大家记得关注即可。

  

 **4**   **cgo 分配的内存**

  

cgo 分配的内存无法被统计到，这个很容易理解。因为 Go 程序的统计是在 malloc.go 文件 mallocgc 这个函数中，cgo 调用的是 c 程序的代码，八杆子都打不到，它的内存用的 libc 的 malloc ，free 来管理，go 程序完全感知不到，根本没法统计。

所以，如果你的程序遇到了内存问题，并且没用到 cgo ，那么就可以快速 pass ，如果用到了，一般也是排除完前面的所有情况，再来考虑这个因素吧。因为如果真是 cgo 里面分配的内存导致的问题，相当于你要退回到原来 c 语言的排查手段，有点蛋疼。

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

一个实际的例子

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

来看一个有趣的例子分析下吧，曾经有个 Go 程序系统 top 看起来 15G，go pprof 只能看到 6G 左右，如下：  

**top 的样子**：

`[amy@centos7]$ top         PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND   499174 service   20   0   19.2g  15.1g   6932 S 635.3 24.3 496911:01 testprog   `

**pprof 看到的**：

`(pprof) top20   Showing nodes accounting for 6.08GB, 88.63% of 6.86GB total   `

**memstat 看到的**：

`sys_bytes 1.6808429272e+10    // 16029  // 系统常驻内存，单位是 M      alloc_bytes 1.0527689648e+10  // 10040  // 分配出的对象，且使用的      heap_alloc_bytes 1.0527689648e+10  // 10040  // 堆上分配出来，且在使用的   heap_idle_bytes 4.272185344e+09  // 4074  // 堆上的内存，但是还没人用，等待被使用   heap_inuse_bytes 1.154125824e+10  // 11006  // 堆内存，且在使用的   heap_released_bytes 5.06535936e+08  // 483   // 释放给 os 的堆内存   heap_sys_bytes 1.5813443584e+10  // 15080.8  // 系统占用内存      stack_inuse_bytes 2.424832e+07   // 23   // 栈上的内存   stack_sys_bytes 2.424832e+07   // 23    // 栈上的内存      mcache_inuse_bytes 55552    // 0.05  // mcache 结构的内存占用   mcache_sys_bytes 65536     // 0.06  // mcache 结构的内存占用   mspan_inuse_bytes 1.87557464e+08  // 178.8  // mspan 结构的内存占用   mspan_sys_bytes 2.31292928e+08   // 220.5  // mspan 结构的内存占用      gc_sys_bytes 6.82119168e+08   // 650  // gc 的元数据   buck_hash_sys_bytes 3.86714e+06  // 3.6   // bucket hash 表的开销   other_sys_bytes 5.3392596e+07   // 50   // 用于其他的系统分配出来的内存      next_gc_bytes 1.4926500032e+10   // 14235  // 下一次 gc 的目标内存大小   `

**上面案例数据可以给到我们几个小结论**：

1. 系统总内存占用 15+ G 左右；
    
2. Go heap 总共占用约 15 G ，其中 idle 的内存还挺大的，差不多 4G；
    

1. 说明**有过请求峰值，并且已经过去**了
    

4. mspan，mcache，gc 元数据的内存加起来也到 1G 了，这个值不小了；
    
5. 下一次 gc 后的目标是 14 G，也就是说至少有 1G 的垃圾；
    
6. pprof 采样到 6.86 G的内存分配，pprof top 可以看到 Go 的分配详情；
    

  

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

总结

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

1. Go 的内存问题好排查，用 **pprof 能解决 99% 的问题**；
    
2. pprof 采样虽然比实际的分配要少，但不影响问题的排查，看最多的就行，**有问题的会一直有问题**；
    
3. **memprofilerate** 可以影响到采样频率，但一般不建议配置；
    
4. tcmalloc 的管理方式没有外部碎片，但是有内部碎片，整体碎片率控制在 12 %左右
    
5. heap 上的 Idle span 的内存，stack 的内存，runtime 本身结构体的内存，这些内存占用有时候也不小，值得关注；
    
6. 一般我们不会把问题指向 cgo，除非你用到了 cgo 并且已经排除了所有其他方向；
    

  

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

后记

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

今天分享的是 Go 内存的一个小思考，**点赞、在看** 是对奇伢最大的支持。

  

~完～

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

往期推荐

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

  

往期推荐

  

  

[

存储架构｜Haystack 太强了！存 2600 亿图片



](https://mp.weixin.qq.com/s?__biz=Mzg3NTU3OTgxOA==&mid=2247497110&idx=1&sn=523f7d02a93d71c32df16cb038ca6ece&chksm=cf3de553f84a6c45b741a9b76ed26d069c5e41488bb7f45e68fb972731be1c658fe86be18cf2&scene=21#wechat_redirect)

[

深度细节 | Go 的 panic 的三种诞生方式



](https://mp.weixin.qq.com/s?__biz=Mzg3NTU3OTgxOA==&mid=2247493867&idx=1&sn=9fef8e55b8220976d6362658d5188618&chksm=cf3df82ef84a7138f21b6fea9e719b204b0f7adaab5632cc8aff8b6966c985300d4af2a71bcd&scene=21#wechat_redirect)

[

Go 存储基础 | 怎么使用 direct io ？



](https://mp.weixin.qq.com/s?__biz=Mzg3NTU3OTgxOA==&mid=2247494750&idx=1&sn=53c88342c2af53aa9bd8310ed5d44a1a&chksm=cf3dfc9bf84a758d2d8be3a638a2979ffc0291319858775c146c5a91cfcbea07a2015a519529&scene=21#wechat_redirect)

[

Go 并发编程 — 结构体多字段的原子操作



](https://mp.weixin.qq.com/s?__biz=Mzg3NTU3OTgxOA==&mid=2247493239&idx=1&sn=9b0a2187e30fba80643a8ed5610adf0f&chksm=cf3df6b2f84a7fa4976cdf8dcdf876e01cb1807de679c996a09ee5b2100a3c3b617c3033dc8b&scene=21#wechat_redirect)

  

  

坚持思考，方向比努力更重要。**关注我：奇伢云存储。欢迎加我好友，技术交流。**

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**欢迎加我好友，技术交流。**

![](https://res.wx.qq.com/op_res/NN_GToMiIjsXzgPzF9-74ZzwR3cA9-fv3o9eWo8f5gQWqx71CmGlY8kFxuIxZaG0TB1bFeMCmh1DGN_pWMRg0A)

**帐号已迁移**

1篇原创内容

公众号

![](https://mmbiz.qlogo.cn/mmbiz_jpg/iaeiczgvdQ9mN1mmOG1e1BzDmThWc2ibcxCAPr4rJFibqdQzyAKWqdvwaW2rdecibibYD2Cm8F8L7tySbot1DqiaWo0Bw/0?wx_fmt=jpeg)

奇伢

 你的支持，是我创作的动力。 

![赞赏二维码](https://mp.weixin.qq.com/s?__biz=MzkyOTU5MTc3NQ==&mid=2247499275&idx=1&sn=0d152befa0b1660845dcbe1dce879000&source=41&key=daf9bdc5abc4e8d0eed18c5c1655bed97fd2fea20dc507364ed1f6e49bc5b655b053093f6d1095a352de06541e0f7427313baa2ac8cee8e739def2da4a19db6fb78b4b3d163afcb3ccce1e3eea2af682482eed517a9b92dbc368b291eb55bb9169b47e78f367af5cb3f8a98c7b3eb71e0dd4c8f029d2e1dff077c11eb1b49554&ascene=0&uin=MTEwNTU1MjgwMw%3D%3D&devicetype=Windows+11+x64&version=63090b19&lang=zh_CN&countrycode=CN&exportkey=n_ChQIAhIQKelixv3HaJQPLU2idxcWmhLmAQIE97dBBAEAAAAAANNrLuBkyLQAAAAOpnltbLcz9gKNyK89dVj0kBINDcQbe2sW5JBvvnpjs0A%2BYDdua0YsW8FPHIdZmTlQoTS0GLJPnYDW%2BcXWcHF0gEUGY0oOr75%2F%2B87MMLCdwK5fCZfA0Ob1BoVsEWvyrnQc80jW%2BnZe%2BXkGn1%2FbzLGJqiYjW0Q4GOJQcp3Qc7B8KRdPFu90MnmYdTfwdEqXL4LcdRbzJFPaaoibuSFKDITtTpmJcwEqSqxR9JZ4T4rsxxwNZBGZHAMsc3n0cmB%2B7U4zBQydGVXeyx1Btq88DoiG&acctmode=0&pass_ticket=Zg1FhW5O%2BMxqfZ%2B2ndJT0pkUThRB3jCzblFdonhxE7hUmf7%2FpnRq3bBAuL5PU0hg&wx_header=1)喜欢作者

Go 最细节篇21

Go 技术专辑46

并发编程10

Go 最细节篇 · 目录

上一篇深度细节 | Go 的 panic 的三种诞生方式下一篇Go 适合 IO 密集型？并不准确！

阅读原文

阅读 1626

​

写留言

**留言 6**

- 奇伢
    
    2022年3月28日
    
    赞2
    
    Go内存问题其实非常好查，但 pprof 看到的经常比实际内存小得多？你知道这是为什么嘛？ 点赞，在看就是最好的支持。加我微信： liqingqiya2019 ，技术交流。
    
    置顶
    
- JUNLONG
    
    2022年3月28日
    
    赞6
    
    推荐一个库 github/mosn/holmes，自动监控的pprof工具
    
    奇伢云存储
    
    作者2022年3月28日
    
    赞
    
    ![[强]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![[强]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
- YouSec
    
    2022年3月28日
    
    赞1
    
    前阵子就遇到这个问题，内存消耗比预期多了很多，查看的时候top比pprof多了不少内存![[破涕为笑]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
    奇伢云存储
    
    作者2022年3月28日
    
    赞
    
    ![[愉快]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)是的，很常见。我之前也是遇到了，所以就分享一点自己的思考。
    
- 王帅
    
    2022年3月28日
    
    赞
    
    好文~感谢
    

已无更多数据

[](javacript:;)

![](http://mmbiz.qpic.cn/sz_mmbiz_png/Pe6fMET7W1sn1HSod7sy8vX2Hqicfos9njfWcj5GP7Ub5K35kOVgzwZia69byvdHu9B3UjtmZIRIssa0K4wby5eA/300?wx_fmt=png&wxfrom=18)

奇伢云存储

2416

6

写留言

**留言 6**

- 奇伢
    
    2022年3月28日
    
    赞2
    
    Go内存问题其实非常好查，但 pprof 看到的经常比实际内存小得多？你知道这是为什么嘛？ 点赞，在看就是最好的支持。加我微信： liqingqiya2019 ，技术交流。
    
    置顶
    
- JUNLONG
    
    2022年3月28日
    
    赞6
    
    推荐一个库 github/mosn/holmes，自动监控的pprof工具
    
    奇伢云存储
    
    作者2022年3月28日
    
    赞
    
    ![[强]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![[强]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
- YouSec
    
    2022年3月28日
    
    赞1
    
    前阵子就遇到这个问题，内存消耗比预期多了很多，查看的时候top比pprof多了不少内存![[破涕为笑]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
    奇伢云存储
    
    作者2022年3月28日
    
    赞
    
    ![[愉快]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)是的，很常见。我之前也是遇到了，所以就分享一点自己的思考。
    
- 王帅
    
    2022年3月28日
    
    赞
    
    好文~感谢
    

已无更多数据