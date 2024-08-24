# 

原创 奇伢 奇伢云存储

 _2021年09月29日 07:46_

![](https://res.wx.qq.com/op_res/NN_GToMiIjsXzgPzF9-74ZzwR3cA9-fv3o9eWo8f5gQWqx71CmGlY8kFxuIxZaG0TB1bFeMCmh1DGN_pWMRg0A)

**帐号已迁移**

1篇原创内容

公众号

坚持思考，就会很酷

  

  

![图片](https://mmbiz.qpic.cn/mmbiz_png/4UtmXsuLoNcdYKVZsqs93HldMUekK1a89Q8HPOibyItpxU2Gib2Qbh1bIKrMaDpPmsBwcZXgh5rfLUWIkcIjj6Mg/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

  

Go 存储编程怎么使用 O_DIRECT 模式？今天就分享这个存储细节，

之前提过很多次，操作系统的 IO 过文件系统的时候，**默认是会使用到 page cache，并且采用的是 write back 的方式，系统异步刷盘的。**由于是异步的，如果在数据还未刷盘之前，掉电的话就会导致数据丢失。

如果想要明确数据写到磁盘有两种方式：要么就每次写完主动 sync 一把，要么就使用 direct io 的方式，指明每一笔 io 数据都要写到磁盘才返回。

**那么在 Go 里面怎么使用 direct io 呢？**

有同学可能会说，那还不简单，open 文件的时候 flag 用 O_DIRECT 嘛，然后。。。

**是吗？有这么简单吗？**提两个问题，童鞋们可以先思考下：

1. O_DIRECT 这个定义在 Go 标准库的哪个文件？
    
2. direct io 需要 io 大小和偏移扇区对齐，且还要满足内存 buffer 地址的对齐，这个怎么做到？
    

  

![图片](https://mmbiz.qpic.cn/mmbiz_png/gBJOYg4SEEUvEqC5oO30octlCXHubelsesdolc6YJHWRf1C8UTPs3scrsXWFoCHYsntUfbpKFrDicJ5tFbLPxBw/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

O_DIRECT 的知识点

![图片](https://mmbiz.qpic.cn/mmbiz_png/gBJOYg4SEEUvEqC5oO30octlCXHubelsl8Q8jryjW8fJDBj8r4A48RcHDMj7ibJfRRhRWSJUUWNTqiaibDr9dmstQ/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

  

在此之前，先回顾 O_DIRECT 相关的知识。direct io 也就是常说的 DIO，是在 Open 的时候通过 flag 来指定 O_DIRECT 参数，之后的数据的 write/read 都是绕过 page cache，直接和磁盘操作，从而避免了掉电丢数据的尴尬局面，同时也让应用层可以自己决定内存的使用（避免不必要的 cache 消耗）。  

**direct io 一般解决两个问题：**

1. 数据落盘，确保掉电不丢失；
    
2. 减少内核 page cache 的内存使用，业务层自己控制内存，更加灵活；
    

**direct io 模式需要用户保证对齐规则，否则 IO 会报错，有 3 个需要对齐的规则：**

1. IO 的大小必须扇区大小（512字节）对齐
    
2. IO 偏移按照扇区大小对齐；
    
3. 内存 buffer 的地址也必须是扇区对齐；
    

**direct io 模式却不对齐会怎样？**

读写报错呗，会抛出“无效参数”的错误。

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

思考问题

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

为什么 Go 的 O_DIRECT 知识点值得一提？  

以下按照两层意思分析思考。

  

 **1**   **第一层意思：O_DIRECT  平台不兼容**

  

**划重点：Go 标准库 os 中的是没有 O_DIRECT 这个参数的。**

为什么呢？

Go os 库实现的是各个操作系统兼容的实现，direct io 这个在不同的操作系统下实现形态不一样。其实 `O_DIRECT` 这个 `Open` flag 参数本就是只存在于 linux 系统。

以下才是各个平台兼容的 Open 参数 ( os/file.go )。

```
const (   // Exactly one of O_RDONLY, O_WRONLY, or O_RDWR must be specified.   O_RDONLY int = syscall.O_RDONLY // open the file read-only.   O_WRONLY int = syscall.O_WRONLY // open the file write-only.   O_RDWR   int = syscall.O_RDWR   // open the file read-write.   // The remaining values may be or'ed in to control behavior.   O_APPEND int = syscall.O_APPEND // append data to the file when writing.   O_CREATE int = syscall.O_CREAT  // create a new file if none exists.   O_EXCL   int = syscall.O_EXCL   // used with O_CREATE, file must not exist.   O_SYNC   int = syscall.O_SYNC   // open for synchronous I/O.   O_TRUNC  int = syscall.O_TRUNC  // truncate regular writable file when opened.)
```

发现了吗？O_DIRECT 根本不在其中。O_DIRECT 其实是和系统平台强相关的一个参数。

问题来了，那么 O_DIRECT 定义在那里？

跟操作系统强相关的自然是定义在 syscall 库中：

```
// syscall/zerrors_linux_amd64.goconst (    // ...    O_DIRECT         = 0x4000)
```

怎么打开文件呢？

```
// +build linux// 指明在 linux 平台系统编译fp := os.OpenFile(name, syscall.O_DIRECT|flag, perm)
```

###   

 **2**   **第二层意思：Go 无法精确控制内存分配地址**

  

标准库或者内置函数没有提供让你分配对齐内存的函数。

direct io 必须要满足 3 种对齐规则：io 偏移扇区对齐，长度扇区对齐，内存 buffer 地址扇区对齐。前两个还比较好满足，但是分配的内存地址作为一个小程序员无法精确控制。

先对比回忆下 c 语言，libc 库是调用 `posix_memalign` 直接分配出符合要求的内存块。go 里面怎么做？

先问个问题：Go 里面怎么分配 buffer 内存？

io 的 buffer 其实就是字节数组嘛，很好回答，最常见自然是用 make 来分配，如下：

```
buffer := make([]byte, 4096)
```

那这个地址是对齐的吗？

**答案是：不确定。**

那怎么才能获取到对齐的地址呢？

**划重点：方法很简单，就是先分配一个比预期要大的内存块，然后在这个内存块里找对齐位置。** 这是一个任何语言皆通用的方法，在 Go 里也是可用的。

什么意思？

比如，我现在需要一个 4096 大小的内存块，要求地址按照 512 对齐，可以这样做：

1. 先分配要给 4096 + 512 大小的内存块，假设得到的地址是 p1 ;
    
2. 然后在 [ p1, p1+512 ] 这个地址范围找，一定能找到 512 对齐的地址（这个能理解吗？），假设这个地址是 p2 ;
    
3. 返回 p2 这个地址给用户使用，用户能正常使用 [ p2, p2 + 4096 ] 这个范围的内存块而不越界；
    

**以上就是基本原理了**，童鞋理解了不？下面看下代码怎么写。

```
const (    AlignSize = 512)// 在 block 这个字节数组首地址，往后找，找到符合 AlignSize 对齐的地址，并返回// 这里用到位操作，速度很快；func alignment(block []byte, AlignSize int) int {   return int(uintptr(unsafe.Pointer(&block[0])) & uintptr(AlignSize-1))}// 分配 BlockSize 大小的内存块// 地址按照 512 对齐func AlignedBlock(BlockSize int) []byte {   // 分配一个，分配大小比实际需要的稍大   block := make([]byte, BlockSize+AlignSize)   // 计算这个 block 内存块往后多少偏移，地址才能对齐到 512    a := alignment(block, AlignSize)   offset := 0   if a != 0 {      offset = AlignSize - a   }   // 偏移指定位置，生成一个新的 block，这个 block 将满足地址对齐 512；   block = block[offset : offset+BlockSize]   if BlockSize != 0 {      // 最后做一次校验       a = alignment(block, AlignSize)      if a != 0 {         log.Fatal("Failed to align block")      }   }      return block}
```

所以，通过以上 AlignedBlock 函数分配出来的内存一定是 512 地址对齐的。

有啥缺点吗？

**浪费空间嘛。** 命名需要 4k 内存，实际分配了 4k+512 。

  

 **3**   **我太懒了，一行代码都不愿多写，有开源的库吗？**

  

还真有，推荐个：https://github.com/ncw/directio ，内部实现极其简单，就是上面的一样。

使用姿势很简单：

步骤一：O_DIRECT 模式打开文件：

```
// 创建句柄fp, err := directio.OpenFile(file, os.O_RDONLY, 0666)
```

封装关键在于：O_DIRECT 是从 syscall 库获取的。

步骤二：读数据

```
// 创建地址按照 4k 对齐的内存块buffer := directio.AlignedBlock(directio.BlockSize)// 把文件数据读到内存块中_, err := io.ReadFull(fp, buffer)
```

关键在于：buffer 必须是特制的 [ ]byte 数组，而不能仅仅根据 make([ ]byte, 512 ) 这样去创建，因为仅仅是 make 无法保证地址对齐。

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

总结

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

1. direct io 必须满足 io 大小，偏移，内存 buffer 地址三者都扇区对齐；
    
2. O_DIRECT 不在 os 库，而在于操作系统相关的 syscall 库；
    
3. Go 中无法直接使用 make 来分配对齐内存，一般的做法是分配一块大一点的内存，然后在里面找到对齐的地址即可；
    

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

后记

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

很小的知识点，但是很实用。**点赞、在看** 是对奇伢最大的支持。

~完～

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

往期推荐

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

  

往期推荐

  

  

[

深度细节 | Go 的 panic 的三种诞生方式



](http://mp.weixin.qq.com/s?__biz=Mzg3NTU3OTgxOA==&mid=2247493867&idx=1&sn=9fef8e55b8220976d6362658d5188618&chksm=cf3df82ef84a7138f21b6fea9e719b204b0f7adaab5632cc8aff8b6966c985300d4af2a71bcd&scene=21#wechat_redirect)

[

Go 存储基础 — “文件”被偷偷修改？来，给它装个监控！



](http://mp.weixin.qq.com/s?__biz=Mzg3NTU3OTgxOA==&mid=2247493695&idx=1&sn=1dbb729c9d50afaf2e868a3be70ded93&chksm=cf3df8faf84a71ecc0b7ddab09fa2a4dac7db7acbad2fb8e1104de8f3ae41764b9f29c19e89e&scene=21#wechat_redirect)

[

Go 存储基础 — 内存结构体怎么写入文件？



](http://mp.weixin.qq.com/s?__biz=Mzg3NTU3OTgxOA==&mid=2247490768&idx=1&sn=ff12c3823968c75b8ee3d7027254a614&chksm=cf3e0c15f8498503df5575eaceca3c87da8807e044f633f8ba79b62bfd6b296a82ec7354c907&scene=21#wechat_redirect)

[

Go 存储基础 — 文件 IO 的姿势



](http://mp.weixin.qq.com/s?__biz=Mzg3NTU3OTgxOA==&mid=2247489085&idx=1&sn=1eb51ea0a00a00a7a62f232221bafaf8&chksm=cf3e06f8f8498fee063670f21bd0a756ef821e359709ed62720b48927ad18eeb0757399036a8&scene=21#wechat_redirect)

  

坚持思考，方向比努力更重要。**关注我：奇伢云存储**

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

![](https://res.wx.qq.com/op_res/NN_GToMiIjsXzgPzF9-74ZzwR3cA9-fv3o9eWo8f5gQWqx71CmGlY8kFxuIxZaG0TB1bFeMCmh1DGN_pWMRg0A)

**帐号已迁移**

1篇原创内容

公众号

![](https://mmbiz.qlogo.cn/mmbiz_jpg/iaeiczgvdQ9mN1mmOG1e1BzDmThWc2ibcxCAPr4rJFibqdQzyAKWqdvwaW2rdecibibYD2Cm8F8L7tySbot1DqiaWo0Bw/0?wx_fmt=jpeg)

奇伢

 你的支持，是我创作的动力。 

![赞赏二维码](https://mp.weixin.qq.com/s?__biz=MzkyOTU5MTc3NQ==&mid=2247499233&idx=1&sn=12b1aa16cdf27654eedc8abbca8fc7ec&source=41&key=daf9bdc5abc4e8d031fcbc71fe8b80f98cbeb5b00f7e513f8a0d460732fb8f32c177fa9b306605ab619b574570e5e960bb240aa411a1cdb5876ba16cdbd53c54320dafe6c411af5712c1d57deff47f2a68f9dee76e92d8f3a068fb765b3a2c82d8cf5a91e3bf2895c8fd74b114e0968acecb093068b3ceca475c891c6e2a29a9&ascene=0&uin=MTEwNTU1MjgwMw%3D%3D&devicetype=Windows+11+x64&version=63090b19&lang=zh_CN&countrycode=CN&exportkey=n_ChQIAhIQ5OulQoqBDKYzIV1W1bztMBLmAQIE97dBBAEAAAAAADalEZOPMToAAAAOpnltbLcz9gKNyK89dVj03J0FWQVeR0Iw6iy216NXHSpgp8sB9aMkDqKi75Y8nnpAdlCP7C5UJQRhVggqRT8o4a7ZrlMEONnRoLGI2j5oefeVa62aSHvKcV35oh1BpJdlXDbpY1I4CMrmsVvrhmS7omVEsScEnjlj1UavHiWCoFgZimVhZReD4Rv0UxhvEHeCHbEcF3ynMQ03DQgQMTvr0JH1spS9CGMk%2BWhtO35GPiVlxfyaMl0IuUEfwvlul2JFArBCXchQAC4qlU2Pm8b6&acctmode=0&pass_ticket=aoAYRkEZWexfk2H06Wmfm9h%2BiNM2%2B8FO%2FNS3LT0yhv6V%2F6goa1Ii6dSuSraxlrF9&wx_header=1)喜欢作者

2人喜欢

![](http://wx.qlogo.cn/mmopen/XLf2naQUn7JV9VkEicLvHHEhSk3ZxuGU8GBUDFRmxauJXPNicrM2cN5YWo49L9gKd0pWcynw1VcwP3kEME7iclwPfkngL97LwEU/64)![](http://wx.qlogo.cn/mmopen/Q3auHgzwzM6PUv6almrP1YPV3eIgiaib6fHpPledujKeGwL6ric5HnlUEdbc7AiarkicjibyXP1BmXLRInMWRyWL8kKS05c2ZdxF8e3k39BbDzCKSvWGTUkWjk34Cx1zy6CRBvowdoib0H9zsA/64)

Go 技术专辑46

存储技术31

Go 技术专辑 · 目录

上一篇深度细节 | Go 的 panic 的秘密都在这下一篇Go 最细节篇｜pprof 统计的内存总是偏小？

阅读 1694

​

写留言

**留言 4**

- 奇伢
    
    2021年9月29日
    
    赞5
    
    Go 里面使用 direct io，你学fei了吗？ 点赞，在看就是最好的支持。加我微信： liqingqiya2019，拉你进群，讨论 Go 编程细节。
    
    置顶
    
- Υκω.Shredder
    
    2021年9月29日
    
    赞3
    
    用cgo封一个
    
- 一只黄羊
    
    2022年10月26日
    
    赞
    
    内存buffer与扇区对齐啥意思？基础知识匮乏![[流泪]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![[流泪]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![[流泪]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
    奇伢云存储
    
    作者2022年10月26日
    
    赞
    
    一个扇区常规是512字节。所谓对齐，就是它的整数倍。
    

已无更多数据

[](javacript:;)

![](http://mmbiz.qpic.cn/sz_mmbiz_png/Pe6fMET7W1sn1HSod7sy8vX2Hqicfos9njfWcj5GP7Ub5K35kOVgzwZia69byvdHu9B3UjtmZIRIssa0K4wby5eA/300?wx_fmt=png&wxfrom=18)

奇伢云存储

41112

4

写留言

**留言 4**

- 奇伢
    
    2021年9月29日
    
    赞5
    
    Go 里面使用 direct io，你学fei了吗？ 点赞，在看就是最好的支持。加我微信： liqingqiya2019，拉你进群，讨论 Go 编程细节。
    
    置顶
    
- Υκω.Shredder
    
    2021年9月29日
    
    赞3
    
    用cgo封一个
    
- 一只黄羊
    
    2022年10月26日
    
    赞
    
    内存buffer与扇区对齐啥意思？基础知识匮乏![[流泪]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![[流泪]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![[流泪]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
    奇伢云存储
    
    作者2022年10月26日
    
    赞
    
    一个扇区常规是512字节。所谓对齐，就是它的整数倍。
    

已无更多数据