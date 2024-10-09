# 

Linux内核远航者

_2021年11月22日 09:25_

The following article is from Linux驿站 Author 微信号szyhb1981

\[

![](http://wx.qlogo.cn/mmhead/Q3auHgzwzM7wDWibpgiaVDGposVS3MtC6Uic5uDpgWRykALUjhjia6Rk3g/0)

**Linux驿站**.

分享Linux用户空间和内核的编程技术以及技术原理。本人把自己学习Linux内核的笔记写成专业书籍《Linux内核深度解析》，2019年5月出版。

\](https://mp.weixin.qq.com/s?\_\_biz=MzAwMDQ1MjAzOQ==&mid=2247484419&idx=1&sn=425e4e0a074a12ee4ebdb1ab06462ce1&chksm=9ae9f0abad9e79bd228655d1e27703f804baa84979c65f16122979fd8e587bc914e718205395&mpshare=1&scene=24&srcid=1122MTpdve5P3c3cJBe7mpoL&sharer_sharetime=1637556057645&sharer_shareid=5fb9813bfe9ffc983435bfc8d8c5e9ca&key=daf9bdc5abc4e8d0fa48ef0775759c774c85dc9b987058ab7031b4de738746787c903ebb71dc178f1634ab761d00a3a00a6c9fdde3905f6642d4cc0ead9345137e849e5c68dae409721e1678d4b4abbc06f41304294f041adb71940be465e1a98379b3d258f21e210349ea6eed0f753033b88c918f8e1d64ae742f3e20251c26&ascene=14&uin=MTEwNTU1MjgwMw%3D%3D&devicetype=iMac+MacBookAir10%2C1+OSX+OSX+14.6.1+build(23G93)&version=13080710&nettype=WIFI&lang=en&session_us=gh_f52274b628f1&countrycode=CN&fontScale=100&exportkey=n_ChQIAhIQNZ9MvJhJymp5PvKSVLxkqhKUAgIE97dBBAEAAAAAABqPJT84n6cAAAAOpnltbLcz9gKNyK89dVj0iA7w9Ob%2FUOhZq1Bit0Rr32aU3%2FfIJOvyAYDMIh3qnIjfSTWStftRANmzf1%2BwAmBJPrXgYjJteL7I%2BEIj4dM%2BVflSLunOk3JLcilwzlig%2FgV5yUR7f5nopchZJexbes9G9ymnZKthvIyvMiGoXrovwXwIvCzShHXroTdqD2B0odXzrEdQAFwmnbqFX1kdA%2FJACTtAkLScZ3Q1sWJTzQ79AbtHY8Ifu7dTXtdwe6LgQyAlefjSQOx%2BvOJwJDmYlDQAIUNJFTh9y7hF8%2FyPbmE0Z5Y5l5yGvdLPvIe0i9YAd6f9kOE4KjcharlqXQxdsg%3D%3D&acctmode=0&pass_ticket=gQie3l%2F0Q3Ac8Iy0Q%2FgxDw7PzDCMinNkwHM0JpBDugQe%2Bwk7KQFIOmOvJsCrsiv0&wx_header=0#)

Linux 5.12引入一个新的内存错误检测工具：KFENCE（Kernel Electric-Fence，内核电子栅栏）。KFENCE是一个低开销的、基于采样的内存错误检测工具。KFENCE检测越界访问、释放后使用和非法释放（包括重复释放和释放的起始地址不是分配的起始地址）这3种错误。

KFENCE和KASAN是互补的。KASAN可以检测KFENCE支持的所有缺陷种类。KASAN依靠编译器插桩，对每个内存访问都检查地址的合法性，更精确，但是导致内核的性能下降，所以KASAN只适合测试环境。KFENCE使用采样的方法，牺牲了精度，但是性能开销几乎为零，它被设计为在产品内核中使用，发现在测试环境中测试用例没有执行的代码路径中的缺陷。

目前只有x86_64和ARM64两种架构支持KFENCE。

# **1.   使用方法**

KFENCE的配置宏如下。

(1)     配置宏CONFIG_KFENCE。

(2)     配置宏CONFIG_KFENCE_SAMPLE_INTERVAL，用来指定采样间隔，单位是毫秒，默认值是100毫秒。把这个配置宏设置为0表示默认禁止KFENCE。

(3)     配置宏KFENCE_NUM_OBJECTS，用来指定KFENCE内存池里面的对象数量，取值范围是1~65535，默认值是255。

可以使用内核启动参数“kfence.sample_interval”在启动时指定采样间隔，单位是毫秒，设置为0表示禁止KFENCE。

KFENCE通过debugfs文件系统提供了一些调试信息，如下。

(1)     文件“/sys/kernel/debug/kfence/stats”提供运行时的统计值。

(2)     文件“/sys/kernel/debug/kfence/objects”提供通过KFENCE分配的对象的列表，包括已经释放但是受保护的对象。

# **2.   技术原理**

KFENCE使用一个固定长度的内存池，如图2.1所示。配置宏CONFIG_KFENCE_NUM_OBJECTS指定对象的数量。每个对象需要2页，一页用来存放对象自身，另一页用作警戒页（guard page）。对象页和警戒页交替出现，每个对象页被两个警戒页包围。内存池的长度是“（对象数量 + 1）× 2 ×页长度”。第1页不是必需的，增加这一页是因为分配偶数个物理页可以简化把对象页地址转换为对象索引的计算。

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

图2.1 KFENCE内存池的布局

在采样间隔到期以后，下一次从SLAB分配器（或者SLUB分配器）分配内存的时候，从KFENCE内存池分配一个对象（只支持分配长度不超过一页），如果内存池用完了，那么返回空指针，由SLAB分配器分配。

如果访问对象的时候越界访问到警戒页，那么触发页错误异常。在页错误异常处理程序里面，KFENCE拦截页错误异常，报告一个越界访问，如果开启了“panic_on_warn”（通过内核启动参数“panic_on_warn”开启，或者执行命令“echo 1 > /proc/sys/kernel/panic_on_warn”开启），那么重启设备，否则把正在访问的警戒页设置为可以访问，让出错的代码继续执行。

为了检测出在对象页里面的越界写，KFENCE使用红色区域。对象页有2种布局，如下。

(1)如图2.2所示，对象在对象页的前半部分，红色区域在对象页的后半部分。这种布局有利于检测左越界，如果向左越界访问左边的警戒页，就会触发页错误异常。

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

图2.2 对象在对象页的前半部分

(2)如图2.3所示，对象在对象页的后半部分，红色区域在对象页的前半部分。这种布局有利于检测右越界，如果向右越界访问右边的警戒页，就会触发页错误异常。

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

图2.3 对象在对象页的后半部分

KFENCE在每次分配对象的时候，随机选择一种布局，并且用特定的字符填充红色区域。释放对象的时候，检查红色区域里面的字符是否变化，如果变化，那么报告错误。

释放一个KFENCE对象的时候，KFENCE把对象页设置为不可访问，并且把对象标记为空闲。继续访问这个对象就会触发一个页错误异常，KFENCE报告一个“释放后使用”错误。为了增加检测出“释放后使用”的机会，KFENCE把空闲对象插入空闲链表的尾部，让最早释放的空闲对象先被分配出去。

Reads 923

​
