
Original clouds clouds

 _2024年08月24日 13:32_

![Image](https://mmbiz.qpic.cn/mmbiz_jpg/Bh66jm0ozvYuBmCrWlIeq84BlZZSVSPaYhP0OQqsAaKO3jJkgRCAdRdJJUfN1rq11djyZfcDUcUz5fOZUsqIfw/640?wx_fmt=jpeg&from=appmsg&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

本文通过Linux内核自带的例子，来讲解eBPF Map和eBPF Loop的使用方法和原理。

内核主线

selftests/bpf/progs/test_map_lookup_percpu_elem.c

知识点

1  eBPF Map

    本实例讲解了三个最常用的eBPF map，分别是：

- **数组类型的Map**  
    

    数组类型的map，是一个数组，map的key就是数组中的索引(index)；key的大小必须正好是4个字节；它的所有元素都在内存中预先分配并设置为零值。 

![Image](https://mmbiz.qpic.cn/mmbiz_png/Bh66jm0ozvYQibbY6ZPp4rEd7I6SdZyxTvliatGpocszo69g1VarPQOI4zR6xibNzicgBGycNhsMYHfa7hZgycF6eA/640?wx_fmt=png&from=appmsg&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

```
struct {
```

- **哈希类型的Map**
    

     也是一个键值对的集合，其中每个键都映射到一个值；他通过key查找value 时，通过哈希实现，而非数组索引。

![Image](https://mmbiz.qpic.cn/mmbiz_png/Bh66jm0ozvYQibbY6ZPp4rEd7I6SdZyxTxcMCTjKshuT2O07u7Xlfn5vvG8vEfQCYibGmjBDTMA3hC5BSic1qyMrA/640?wx_fmt=png&from=appmsg&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

```
struct {
```

- **LRU哈希类型的Map**
    

   普通hash map的问题是有大小限制，超过最大数量后就无法再插入了。LRU哈希map可以解决这个问题，如果map满了，再插入时它会自动将最久未被使用(least recently used)的元素从map中移除，为新元素提供空间 。

```
struct {
```

2  eBPF Loop

    内核5.17版本，引入了一个新的辅助函数bpf_loop;  bpf_loop()通过接收一个static函数loop_fn()和iterations，可让loop_fn执行iterations次。

```
int bpf_loop(int nr, int (*fn)(__u32, void *), void *ctx, int flags);
```

- **`nr`**: 循环次数，即要遍历的元素数量。
    
- **`fn`**: 回调函数，每次迭代时会调用此函数。它的参数包括当前元素的索引和用户提供的上下文指针。
    
- **`ctx`**: 用户提供的上下文指针，将作为参数传递给回调函数。
    
- **`flags`**: 目前没有实际作用，通常设置为 0。
    

在本文讲解的eBPF程序中，`bpf_loop`用来遍历每个CPU上的映射元素。

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

源代码视频解析

源代码的解析请看视频里详细的讲解；

，时长08:46

---

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

源代码文字解释

  通过这个eBPF程序和用户空间测试程序，演示了eBPF程序如何从每CPU映射中读取数据，并计算这些数据的总和。用户空间程序设置数据并触发eBPF程序执行，然后验证计算结果是否符合预期。

---

用户空间程序

### 主要流程概述

1. **准备工作**:
    

- nr_cpus: 获取系统中可能的CPU数量。
    
- buf: 为每个CPU分配一个缓冲区，并初始化为CPU索引的值(即第i个CPU 上的值为i）。
    
- sum: 计算buf中所有元素的累加和，这个值稍后将用于验证eBPF程序的输出是否正确。
    

3. **加载 eBPF 程序**:
    

- 打开eBPF程序。
    
- 设置eBPF程序中的只读数据(my_pid 和 nr_cpus),确保eBPF程序在监控正确的进程，并知道CPU的数量。
    
- 加载eBPF程序到内核。
    
- 将eBPF程序附加到sys_enter_getuid 事件。
    

5. **更新 eBPF 映射**:
    

- 使用bpf_map_update_elem函数将用户空间中的buf数据写入到eBPF程序中的映射中，确保每个CPU上的值为其对应的索引。
    

7. **触发系统调用**:
    

- 通过执行syscall(__NR_getuid)，触发sys_enter_getuid事件，从而触发eBPF程序的执行。eBPF程序将读取三个映射中每个CPU上的值并计算总和。
    

9. **验证结果**:
    

- 使用ASSERT_EQ宏验证eBPF程序计算的总和是否与预期的sum相等。
    

11. **清理工作**:
    

- 销毁eBPF程序实例。
    
- 释放用户空间的buf缓冲区。
    

---

eBPF 程序分析

- **全局变量**：程序中定义了几个全局变量来存储从映射中读取的数据总和。
    
- **读取映射**：eBPF程序通过循环遍历每个CPU，读取每个映射中的值，并计算它们的总和。
    
- **处理系统调用**：当触发的`sys_enter_getuid`跟踪点被调用时，eBPF程序会执行上述的计算操作。
    

  

---

联系作者

有任何疑问和建议可以联系作者。  

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

![](https://mmbiz.qlogo.cn/sz_mmbiz_jpg/0xZAYAjdlib60gRb2a6ib8V0tm5R9f19zaicsEsZ7XbCqnpbHICSrcvMpRd48NTibMWdQfu89LgUiaG5HHuPyD7ZR7w/0?wx_fmt=jpeg)

clouds

![赞赏二维码](https://mp.weixin.qq.com/s?__biz=MzA3MDEwOTM2MA==&mid=2651575784&idx=1&sn=601b24297750318aab4da6a9fbea4195&chksm=8473875a6981bc8eb617647c2db55aa21b775923da8e5f95561a12495ca505dfbc886c83be85&mpshare=1&scene=24&srcid=0824thhNFzi95jg4lCRPiBz6&sharer_shareinfo=334eda780b344bb0228625ffb23599fa&sharer_shareinfo_first=334eda780b344bb0228625ffb23599fa&key=daf9bdc5abc4e8d00bb9b6ca6540a506725cc49f1e1a34275f22d9ad628d5545e02f20f2c39507d2ea572b14fc69bc1ae45e754e54527f50826f3362f832c47b880e7f35070f8dd649b702d61215289b02424db940c2ac6b7de455544a150a4b09d2c63b79bf51d8b1b6be829985dc756dfb34f8342a5558fb8c41156db1abf3&ascene=14&uin=MTEwNTU1MjgwMw%3D%3D&devicetype=iMac+MacBookAir10%2C1+OSX+OSX+14.6.1+build(23G93)&version=13080710&nettype=WIFI&lang=en&session_us=gh_77cd08fc67f0&countrycode=CN&fontScale=100&exportkey=n_ChQIAhIQAeeCIRLUvWk%2BdUeelZnQMhKUAgIE97dBBAEAAAAAAErWOiBukwoAAAAOpnltbLcz9gKNyK89dVj0tkxMnbuaPxC1goDfgt6XBXUC2Oljh6lrJaEWk%2Fr9T2M2FAN%2BydRMKJKgQiYAYDC%2Fi%2FEuH6%2Bg7VAwyfnhkplwmnNJ%2BaMFsMgMMWok8h5wA2B9iiy2Ozcaio018pZ7hrdgtGXZg4WYjc3BJP3Gd6BI2ijnpJ5vmCYw48pJwXpoRwQpEumDOf5kCjdSmJxlAbJhZ3zxGtNCokt%2F36AlhTJCmswJAAJ%2BH7qa%2FVhx65AWtxFmkdhcYl6hFQI1dfn%2B7LscGNtnLf%2FBQfNXI%2FXqf2t5bvqIlTyvpCQHUlj6vyW6xxCefZxuf4mUfObI9qXJWA%3D%3D&acctmode=0&pass_ticket=Trihsns4r5Uq%2BdN9oxrRSZNiK8jM5LT8AJOUpMCbCNUyhbrOA5Y5vyu%2B8O5ZBdzh&wx_header=0)Like the Author

Linux eBPF7

Linux eBPF · 目录

上一篇Linux内核实例解析系列-eBPF根据协议类型统计流量

Reads 60

​

Comment

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/Bh66jm0ozvabWTwwn3N6dPgFhdtF8mx2PIV2BOjSIRXOib9SOXJ0RaibcDSFibCbEWEqZgQc33Tep8TLrlh46PbWQ/300?wx_fmt=png&wxfrom=18)

clouds

Like4Wow

Comment

Comment

**Comment**

暂无留言