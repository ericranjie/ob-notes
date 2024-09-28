
原创 JeffXie Jeff Labs

 _2022年01月15日 23:36_

内存泄漏应该怎么检测？  

比如:

event_object_trigger_callback()

```c
{           obj_data = kzalloc(sizeof(*obj_data),GFP_KERNEL);           obj_data->field= field;           obj_data->offset = offset;           obj_data->obj_type_size= obj_type_size;           …}
```

当这个函数event_object_trigger_callback执行之后，obj_data指向的数据还在使用中，此时还没有调用kfree(obj_data),我们怎么知道obj_data是否已经发生内存泄漏呢,我们也不能依据是否调用kfree来判断内存泄漏吧。

如果仅仅是以上代码，kmemleak工具在扫描过程中，obj_data会被判定为内存泄漏，因为没有其它的内存引用obj_data.如果是以下这种代码，就不会被判定为内存泄漏

```c
event_object_trigger_callback(){    obj_data = kzalloc(sizeof(*obj_data), GFP_KERNEL);    obj_data->field = field;    obj_data->offset = offset;    obj_data->obj_type_size = obj_type_size;    trigger_data = kzalloc(sizeof(*trigger_data), GFP_KERNEL);    trigger_data->private_data = obj_data;…}
```

因为此时有内存(trigger_data->private_data)引用obj_data,会被kmemleak扫描到，如果在这里你完全看懂了，后面就不用看了;-)

  

Kmemleak 把object(其实就是pointer<指针> 分为三种颜色:

```c
mm/kmemleak.c 301  * Object colors, encoded with count andmin_count: 302  * - white - orphan object, not enoughreferences to it (count < min_count) 303  * - gray - not orphan, not marked as false positive (min_count == 0) or 304  *             sufficient references to it (count >= min_count) 305  * - black - ignore, it doesn't containreferences (e.g. text section) 306  *             (min_count == -1). No function defined for this color.
```

  

**black:** 意思就是不在其它内存中扫描这部分object,也不在这部分内存中扫描其它的object.不参与整个kmemleak检测游戏。

**white:** count < min_count  (如果内存扫描之后，某个对象没有其它人引用,引用数目count 小于引用初始值min_count， 则这个object就判定为内存泄漏。

**gray:** 经过内存扫描之后，某个对象被其他人引用数count大于等于初始值min_count, 则标为gray, 说明此object没有泄漏。

Note:

一个对象，叫做一个object,就是一个pointer(指针)，在内核中使用一个数据结构(kmemleak_object)维护这个object的引用，状态，被内存分配时候分配的时间，归属的进程等信息。

```c
struct kmemleak_object {     /* 仅仅展示重要的成员*/{unsigned int flags;        /* object status flags */struct list_head object_list;struct list_head gray_list;unsigned long pointer;/* minimumnumber of a pointers found before it is considered leak */int min_count;  /* the totalnumber of pointers found pointing to this object */int count;u32 checksum;pid_t pid;char comm[TASK_COMM_LEN];} 
```

最主要的内存扫描函数 kmemleak_scan()

1.  把所有对象变成white

```c
list_for_each_entry_rcu(object,&object_list, object_list) {/* reset the reference count (whitenthe object) */       object->count= 0;if (color_gray(object) &&get_object(object))    list_add_tail(&object->gray_list,&gray_list);…}// color_gray(object->count >= object->min_count)
```

哪些对象初始化的时候就是gray(color_gray)?

```c
mm/kmemleak.ckmemleak_init()    /*register the data/bss sections */ create_object((unsigned long)_sdata, _edata -_sdata,               KMEMLEAK_GREY, GFP_ATOMIC); create_object((unsigned long)__bss_start,__bss_stop - __bss_start,               KMEMLEAK_GREY, GFP_ATOMIC);#define KMEMLEAK_GREY  0#defineKMEMLEAK_BLACK  -1static struct kmemleak_object *create_object(unsigned long ptr, 
```

说明内核数据段和bss段，初始化的时候就是gray.(不考虑在内存泄漏范围内，只参与扫描这部分内存中是否引用了其它的object)

(要哄小孩了，未完待续)  

  

  

![](https://mmbiz.qlogo.cn/mmbiz_jpg/Uq9aKmPtujWtN5RaswEJS8kFGHGyEMV4VPFxz1QoNVmmNRZrr4Tgibak8FtpSmLaMMezfHZzibEkATLywlUVYrSQ/0?wx_fmt=jpeg)

JeffXie

 谢谢你的爱 

![赞赏二维码](https://mp.weixin.qq.com/s?__biz=MzA5ODI2NzMyMQ==&mid=2458811312&idx=1&sn=8ed3fef88d3058e6e22062daf7c7279a&chksm=87eee266b0996b70233f8d4345f04ae01fa1417bcaec04a080f9b98693839c6ed5eda0ecd768&mpshare=1&scene=24&srcid=01160NEjtCmP4vgsqaE9yLOq&sharer_sharetime=1642296072312&sharer_shareid=5fb9813bfe9ffc983435bfc8d8c5e9ca&key=daf9bdc5abc4e8d07114c936249b97133ee81399c41ff4404eda3aad35f9afe6ac004d2c0b700c6afa5b91c038afb8773cfb8218ad6c5b8ff4a79670bbe0c3fde4fb83658ff47c9543132502687470948ec7d26bcde1db5c2888f7834f0429afececd016983520aaa4b22253f1e8ee43166233098869cf314e6b3f67d3af51d5&ascene=0&uin=MTEwNTU1MjgwMw%3D%3D&devicetype=Windows+11+x64&version=63090b19&lang=zh_CN&countrycode=CN&exportkey=n_ChQIAhIQXwAtaoNaAeuxMezlj5uDcBLmAQIE97dBBAEAAAAAANLTOEBQhPUAAAAOpnltbLcz9gKNyK89dVj0DWYrryqRATr6FJGJJCBXFOuErITfZKkq2bU9b4DGexmc5u7iaZgGDe4tJf4p%2B3Aoxcwnhz%2B6mxlJPmb9KODE6dsnFso9Q%2F7b2KoQJZvYWiPI50jHWyENXFuBH%2BkfK04xLnbmhM%2FIPaBQ1DdYzVHZogqU3sWZkBwS9KM9nYGV7rfXPT%2FvMEBsOLa8BcSTSbJE58EBywDSoTHP1YBJ8fP118TS6qP1%2BPk%2FfxIzRkRHidCMgbh9BV3nw%2BHKt%2BMlDuLG&acctmode=0&pass_ticket=GBQBAZCfjr402zVoeDJG8NvMRvKTC7iz8G%2F9V27j0YxeWzjYe%2BVtvlJxgqhKoZ%2Br&wx_header=1&fasttmpl_type=0&fasttmpl_fullversion=7350504-zh_CN-zip&fasttmpl_flag=1)喜欢作者

阅读 700

​

写留言

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/QO9OBu0wPg0c2nEoRPjUtn2uQGibnXhXMxuKw5RwHLdVzsm6iaIE3okWLL42EIpzcPb33fS2pa8CicPrzpesewvCw/300?wx_fmt=png&wxfrom=18)

Jeff Labs

4分享在看

写留言

写留言

**留言**

暂无留言