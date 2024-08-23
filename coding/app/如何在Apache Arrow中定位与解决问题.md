

原创 lightcity 光城

 _2024年03月27日 16:58_ _广东_

## 如何在apache Arrow定位与解决问题  

最近在执行sql时做了一些batch变更，出现了一个 crash问题，底层使用了apache arrow来实现。本节将会从0开始讲解如何调试STL源码crash问题，在这篇文章中以实际工作中resize导致crash为例，引出如何进行系统性分析，希望可以帮助大家～

在最后给社区提了一个pr，感兴趣可以去查阅。

> https://github.com/apache/arrow/pull/40817

## 背景

最近想修改一下arrow batch的大小，当调整为65536后发现crash，出现：

`terminate called after throwing an instance of 'std::length_error'     what():  vector::_M_default_append   `

然后通过捕获异常gdb找到异常位置，最后拿到堆栈，发现位置是在join里面构建哈希表侧的partition数组出了问题：

`prtn_state.key_ids.resize(num_rows_before + num_rows_new);   `

即问题转化为：**resize操作为何引发throw？**

研究了一下STL代码发现，会遇到两种场景，先把STL代码精简一下贴出来给大家看看：

 `if (__navail < __n) {      const size_type __len =      _M_check_len(__n, "vector::_M_default_append");    }      size_type _M_check_len(size_type __n, const char* __s) const {    if (max_size() - size() < __n)      __throw_length_error(__N(__s));   }`

其中最核心的就是`_M_check_len`函数，看到这个判断能想起哪两种场景呢？

- 场景1：内存确实不足了，超过了vector的max_size，此时会抛这个异常。
    
- 场景2：`__n`传递的是一个负数，由于是size_t类型，则会变为超大值，从而抛出异常。
    

场景1在我们系统当中通过查看内存不会遇到，于是转到场景2，首先是猜测是个负数，然后搞了个log包，上去测试发现确实是这个问题，可以看到rows_new变为负数了。

`part id 15, dop_ = 105,prtnid + 1 ranges = 0,prtnid ranges = 61434, part size:0, rows_new: -61434, cap: 0   `

既然这里知道原因了，那么下一步就得继续分析为何会产生负数？

num_rows_new是有分区的range决定的，下面有个公式计算产生了负数

`int num_rows_new =         locals.batch_prtn_ranges[prtn_id + 1] - locals.batch_prtn_ranges[prtn_id];   `

继续跟进找到PartitionSort的Eval，里面有几处非常需要注意：

`ARROW_DCHECK(num_rows > 0 && num_rows <= (1 << 15));   `

首先第一个是这个断言，我明明传递的是65536，明显大于这里的32768，为何没有断言成功？事后发现这里是release包，只会报warning，不会fatal。

随后继续往下看，看到了一个比较明显的类型`uint16_t`，这个玩意就是在计算sum，而要让num_rows_new为负数，只有两种可能：

- 场景1: locals.batch_prtn_ranges[prtn_id + 1] < locals.batch_prtn_ranges[prtn_id]
    
- 场景2:  locals.batch_prtn_ranges[prtn_id + 1] 是负数且locals.batch_prtn_ranges[prtn_id]是负数或者locals.batch_prtn_ranges[prtn_id + 1] 是负数且locals.batch_prtn_ranges[prtn_id]也是负数并且大于前者。
    

`uint16_t sum = 0;   for (int i = 0; i < num_prtns; ++i) {     uint16_t sum_next = sum + prtn_ranges[i + 1];     prtn_ranges[i + 1] = sum;     sum = sum_next;   }   `

看了这段代码可以知道，场景1排除了，因为是自增的，最差情况是相等，那么就只能场景2，变为负数就不用说了，又碰到了溢出问题，所以可以推测uint16_t溢出了，这个值我们知道是65535，而65536刚好超过它，所以有问题！

至此，这一轮的debug调试与分析到此结束～

---

往期干货：

[热度更新，手把手实现工业级线程池](http://mp.weixin.qq.com/s?__biz=MzI2NjYwOTAyMg==&mid=2247488519&idx=1&sn=0ffc5b91b576e3592c0b687229f14982&chksm=ea8adc16ddfd55003c8b97eb409b388d4b940b9e03a85876d50e5996c451da9697ca4cfa05c7&scene=21#wechat_redirect)

[快速拿下面试算法](http://mp.weixin.qq.com/s?__biz=MzI2NjYwOTAyMg==&mid=2247486607&idx=1&sn=d365d5d393e463d3fd3f8dddcc4704d4&chksm=ea8ac49eddfd4d88b6e972feaf043b34a0ce6c57a2c6d96996cb35270d7180515376d2ad8ca0&scene=21#wechat_redirect)

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

  

![](https://mmbiz.qlogo.cn/mmbiz_jpg/xdatVaX8ek3UwLBhWibBLb3ATy7p1W9S5APibicPPGTu4NQK4aP7Uf8IOe0Q0EhaRYzb6U22FOYuIIDgwXHlogiblg/0?wx_fmt=jpeg)

lightcity

 坚持原创 

![赞赏二维码](https://mp.weixin.qq.com/s?__biz=MzI2NjYwOTAyMg==&mid=2247489366&idx=1&sn=5320b2f52de14934c8d4985a03a5cb77&chksm=ea8adf47ddfd56516113e0f050118852b8538bb5225e5a82143b8a6f68cff8b88e5098e301b0&mpshare=1&scene=24&srcid=0327R2YSpobzCQxa1WQr2nkO&sharer_shareinfo=643ea2103a8379bf58a77b0d3d5917b0&sharer_shareinfo_first=643ea2103a8379bf58a77b0d3d5917b0&key=daf9bdc5abc4e8d05d3d2faff8b7146d34873def46eec9eec4f74f6b6ab46b95c90a76a45661d7339bf5bd32ad28a131571284b2ab4690af4e6601640524e7e831d9343dfae438ee826ae68112323ea28166eccda200c5b9d321253b717ce4f9e147967f189542f4a9beb8226872e812226e3a22afa0fe769c8de4f8ea9d4aa9&ascene=0&uin=MTEwNTU1MjgwMw%3D%3D&devicetype=Windows+11+x64&version=63090b19&lang=zh_CN&countrycode=CN&exportkey=n_ChQIAhIQBbL%2F%2BnMyMP%2F%2BkvAvFnaLWhLmAQIE97dBBAEAAAAAAGHlKFFQ5zIAAAAOpnltbLcz9gKNyK89dVj0%2BTYQ%2FgylQsXBWYZuPSD0b7NUsbzXdr%2BohzGq13XgfyhOqZNCs6WKJ5UXHS0k51dfaj7e6xydqTuBl6OE%2F3WBpFTy4IZCtS39U0svDUy2XdWwUmIdfito4oSPufnZ9ZoIuaz3qHvWABSHXTi8oxW9uvpWc8C8apCvqliULR6Y6m%2B9FzkjEI4sZMXqtmu7pz9mmGbOrTRwukkLntU3NIXHBim4nr3TlZJykvltMd2prbsYta7jzxQ0RX%2B%2BsZmdINC8&acctmode=0&pass_ticket=NEvd89quH9B7XPNZsVZcR6%2FNq03NklZu2WnKG5yDYscTUAQBRthfqHUxA49sX5ux&wx_header=1&fasttmpl_type=0&fasttmpl_fullversion=7350504-zh_CN-zip&fasttmpl_flag=1)喜欢作者

apache1

arrow4

C++那些事193

c++34

现代C++38

阅读 506

​