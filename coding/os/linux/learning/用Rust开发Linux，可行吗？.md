# 

CSDN程序人生

_2021年11月23日 15:29_

The following article is from CSDN Author 马超

\[

![](http://wx.qlogo.cn/mmhead/Q3auHgzwzM5cvOsZy9wYacdpSLicuibpMX0ibCb5C0Z8JibGbnmPKCB0Yw/0)

**CSDN**.

成就一亿技术人

\](https://mp.weixin.qq.com/s?\_\_biz=MzkxNjI3ODAwNw==&mid=2247552599&idx=2&sn=c9288d03d192637f446f76f76e323fa8&source=41&key=daf9bdc5abc4e8d0c10c72a9116da088a4064799f8f623112a521fc7e1746e2a7f699e7cb7d1f347e9658c61819218d345fe84140530ba9496cedfcb44fd488898218688ab7e39bdc7643b303242b9edf44b2994f90c73fe568d48ec1b29ee904dca554a16447ba5fed6e1719894f44e5903e13f339c4a33b7a2f91e017da83e&ascene=14&uin=MTEwNTU1MjgwMw%3D%3D&devicetype=iMac+MacBookAir10%2C1+OSX+OSX+14.6.1+build(23G93)&version=13080710&nettype=WIFI&lang=en&session_us=gh_e8d4c70739cb&countrycode=CN&fontScale=100&exportkey=n_ChQIAhIQ9V1xqi42oZtlR8IZUD05kRKUAgIE97dBBAEAAAAAAJ6eOABGzUsAAAAOpnltbLcz9gKNyK89dVj0kkuY8NHlnCN3sRoPTb57E%2Blbwm9TXOCTINfzbiXySIPuK8h97Wa1wjJmfO0xZfZJSLpi%2BUeviESq%2FYVnlL%2Bgt2YT5uqXA7zfymLddNHeplh%2B3uyBJgCYGXL77vG2gdJ0cG6%2Fdm0Ql2Ev3daltr7chpw2mk0T2j%2BWWdsYEArlJuStPNdKFvKOGm0kbm6n3UL2YdaU4YEhToPjwZ7UMU%2FK7hU1cedWvZipUzMkWKpFLIsqcKSItFFgfXUfZA32fbLmlju6PqeUFNr6LPMklo4y8hsjQhc%2FaixqFdoBgTEBNv%2BBIRlihC1XmHrK41f8KA%3D%3D&acctmode=0&pass_ticket=%2BEurHQ2Umk6g5il8hLy8Lp2Z8c6iMif5tHDhENktOIyakpxHQKCo4oUfn8lSn34T&wx_header=0#)

![Image](https://mmbiz.qpic.cn/mmbiz_gif/1hReHaqafad4H57UlgDZZl7lILyDiaAWDsRcksUcCYeT76ibEllhuHJU9PxRtFgAQC7QPgW6qicToOuMjnSsmsErQ/640?wx_fmt=gif&tp=wxpic&wxfrom=5&wx_lazy=1)

作者 | 马超

出品 | CSDN（ID：CSDNnews）

继Python之后，Rust最近也火爆得出了圈，目前Rust在Serverless等很多云原生领域已经稳定占据了C位，那么让Rust更进一步去开发操作系统的内核，就成为很多Rust粉丝心中的终极梦想，而Rust官方也一直有想法使Rust语言成为下一代操作系统的标准，在https://github.com/rust-osdev/上各种基于Rust开发的如BootLoader等工具已经发展比较齐全了，目前相对比较成熟的Rust操作系统有基于X86架构的Redox和清华大学的基于RISC-V架构芯片的rCore OS。

不过Rust想跨越C语言这座高山却并不容易，因为C语言就是为了实现操作系统而设计的。但是C语言作为一种诞生于半个世纪之前的语言，目前的确在很多方面的理念上已经落后于时代了，C语言本质上只是对于计算机硬件行为的一种抽象，瑞奇和汤普森两位天才引入的指针概念，小小的指针也大大地推进了整个IT行业的发展。

![Image](https://mmbiz.qpic.cn/mmbiz_jpg/Pn4Sm0RsAuhBKPdm6Paibvm9xy2pN2g70QdLV1r2Yr4Fj0fT2ufTDOu6WTZf2CbicbJQuZ2piaEjFOFrgpqQ3qMUg/640?wx_fmt=jpeg&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

但是C 语言成也指针败也指针，指针的高效使得C语言成为各类语言的性能标杆，但指针带来的内存泄露，也是各大操作系统中的主要安全隐患。而Rust语言具有与C语言有着基本相同的运行性能，而且大幅增加了安全性。也有人将Rust称为安全版的C语言，因此让Rust到目前最流行的操作系统Linux当中去一显身手，也就颇为顺理成章了。

谷歌是率先在操作系统领域放弃C语言的科技巨头，也是Rust For Linux项目背后的主要推动力量，因为谷歌的Fuschia作为新一代操作系统内核Ziron中，很多关键模块都使用C++来开发。

从目前的情况来看，谷歌是不打算再用C语言作为未来操作系统的主体开发语言，虽然在Fuschia中使用了很多Rust代码，但是Fuschia毕竟还是个小众的试验性项目，如果能在真正的国民操作系统Linux上尝试Rust，才能真正试出来 Rust这门编程语言在操作系统内核开发方面到底是个什么成色。

目前Linux的创始人林纳斯对于Rust For Linux项目，也持正面看法，他表示：“我对这个项目很感兴趣，但我认为这是那些对 Rust 非常兴奋的人推动的，我想看看它然后在实践中最终如何工作。

就个人而言，我暂时不会推动 Rust 化，不过考虑到承诺的优势以及能够避免一些安全隐患，我对它持开放态度。但我也知道，有时承诺并不能实现”。在林纳斯表达这番言论以后，Rust进入Linux内核的项目也取得不少的积极进展。

**![Image](https://mmbiz.qpic.cn/mmbiz_png/1hReHaqafacwpA01Gf7IBJuia2GvxJfD8mrkN4ReHBmviaxk8aJWfAkkdLTHbrp9LelJY6ebyv3pcW5QgaouBI7A/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)**

**GPIO驱动-RFL的最新进展**

几天前一名来自于谷歌的Rust开发者Wedson Almeida Filho，发布了一个RUST版本ARM PL061 GPIO驱动，详见：https://github.com/wedsonaf。

Rust的GPIO驱动中对于Device ID表进行了构建，Rust强制开发人员决定数据类型，而且程序员管理变量的生命周期，因此驱动整个机制是类型安全的，而且更容易使用。由于代码很多，我们先把这两种语言的比较典型的对应函数实现，做一下对比：

C语言的 pl061_direction_input函数实现如下：

``` static int pl061_direction_input(struct gpio_chip *gc, unsigned offset)``{``struct pl061 *pl061 = gpiochip_get_data(gc); ```        `unsigned long flags;`        ``` unsigned char gpiodir;``    ```        `raw_spin_lock_irqsave(&pl061->lock, flags);`        `gpiodir = readb(pl061->base + GPIODIR);`        `gpiodir &= ~(BIT(offset));`        `writeb(gpiodir, pl061->base + GPIODIR);`        ``` raw_spin_unlock_irqrestore(&pl061->lock, flags);``   ``return 0;``} ```

Rust的 pl061_direction_input函数实现如下：

``` fn direction_input(data: &Ref<DeviceData>, offset: u32) -> Result {``let _guard = data.lock();``let pl061 = data.resources().ok_or(Error::ENXIO)?;``let mut gpiodir = pl061.base.readb(GPIODIR);``gpiodir &= !bit(offset);``pl061.base.writeb(gpiodir, GPIODIR);``Ok(())``} ```

通过这两种语言的对比也可以看到，Rust中的lock锁是与具体要保护的数据是有强绑定关系的，开发者要调用data.lock()将锁进行锁定，只有这样才能受锁保护的数据才能被访问，程序员在锁使用的方面很难犯错误，而C语言的代码则需要显式配对使用raw_spin_lock_irqsave(&pl061->lock,flags)；与raw_spin_unlock_irqrestore(&pl061->lock,flags)两个API才能达到类似效果，而且C语言中在加锁后，释放锁的操作是开发者必须要小心注意的问题，如果没有被及时释放锁还会造成很大的问题，因此从这方面说Rust的确是安全语言

不过正如林纳斯所说，RustFor Linux目前还处在初始的探索阶段，Rust的首要目标应该是操作系统外围的驱动程序，因为驱动和整个内核比起来体积小，而且相对独立。但是想深入到Linux的核心部分，Rust要做的事情还很多。

**![Image](https://mmbiz.qpic.cn/mmbiz_png/1hReHaqafacwpA01Gf7IBJuia2GvxJfD8M4VQKRxCQJmdvLtXk45ticpc59Expyz3cI3ic2fy9AYLyzflAiaMg1CYw/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)**

**内存模型-最大的阻碍**

由于Rust特有的变量生命周期及借用等机制，所有变量在内存中都是可移动的，这对于引用Linux的数据结构来说是会有问题的。如果要固定变量的内存地址，就需要引入unsafe的Rust代码，这样的话Rust的安全优势也将荡然无存。用上文GPIO驱动作者Filho的话来说，现有Rust驱动程序中的许多不安全代码都源于这个可移动的问题。解决这个问题主要是找到正确的抽象，但是目前这个抽象模型还没有看到。

而且内存模型的问题还远不止数据的申请、释放与内存布局，在多核时代内存模型还增加了操作系统在保证执行顺序与并发灵活性之间复杂的取舍策略。

简单的讲当下最新的编译器、操作系统及处理器等等底层技术栈，都会进行某种程度上对于代码进行重排，以获取执行效率的优化比如relu函数的实现代码

`if (x>0)    y = x;elsey = 0;`

就可能被编译器优化为以下的代码：

`y=0if (x>0)    y = x;`

也就是说y会被提前赋值。

再比如以下代码：

`x=*p;y= *(p+1);`

也很可能在执行时被CPU进行合并读取操作，也就是x与y被同时调入内存，按照CPU伪指令执行如下：

`{x,y}=Load{p,p+1}`

这些重排操作在单核情况下基本没有问题，但在现在的多核并行工作的时代就可能引发风险，在内存模型的建模机制上Rust和Linux并不相同，所以这也在很多方面造成了二者之间的融合问题，这个问题在Rust为Linux开发应用程序时几乎并不存在，但是当Rust想和C语言融为一体共同成为Linux的组成部分时就会被急剧的放大。

当然内存模型这个话题不是本文所能说清楚的，后面笔者还计划撰写文章对于Linux内存模型进行专题介绍，这里不加赘述了。

**!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)**

**panic、alloc到底如何实现**

在Linux当中一旦内核态的代码执行中出现不可恢复的错误，一般是通过panic操作来记录相关信息及调用栈，但由于Rust的内存申请与释放机制，其编译器通常会隐藏内存分配的操作，这就很可能使panic!()的调用出现问题。而且在某些驱动程序中，内存分配失败不应该直接使内核产生panic，因此Rust在申请内存失败后如果直接调用panic!，可能也是错误的。

而且在Linux标准接口中的内存分配alloc API也需要为Rust For Linux项目做好准备，像Rust中原生自带的数据类型中如Vec等，都无法通过稳定版本的Linux alloc接口分配内存，从目前非稳定版本的实现来看，实现alloc这些标准接口，很可能会大量引入很多unsafe的Rust代码，这将使Rust的价值大大降低。因此从细节上看Linux还要为Rust的入驻做更充分的准备。

**!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)**

**稳定内联汇编-Rust操作系统的必游之路**

其实即使是C语言也无法单独完成开发一整套操作系统的任务，汇编语言在很多情况下是操作系统所必须的，因为有一些关键操作必须直接调用CPU底层的指令才能执行，目前Rust在开启#!(feature(asm))的情况下倒是也可以支持内联汇编，例子如下：

`#![no_std]#![feature(asm)]pub mod bits;pub mod mutex;pub mod ia_32e;#[cfg(test)]mod tests;pub  fn nop(){unsafe{       asm!("xor %eax, %eax"       :       :       : "{eax}"       :       );    }}fn main(){unsafe{       nop();    }}`

但是目前Rust对于内联汇编语言的支持并不成熟，更不算稳定，如果无法做到直接对接汇编语言的代码，Rust的操作系统内核之路就不可能一帆风顺，因此Rust如果想成为实现操作系统的主体语言也还有很长的路要走。

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**灵魂拷问-少林与逍遥派的联手，真的会有那么好的效果吗**

有关于Rust ForLinux的项目，笔者比较担心用C语言和Rust共同编写Linux内核会使Linux的可理解性和可学习性进一步降低，甚至看起来有点不伦不类，从某种程度上讲Rust For Linux就像用中文、英文双语穿插写《论语》、《圣经》一样会令人非常难以理解，甚至会降低著作的经典程度。

正如笔者在前文《Java、Go、Rust大比拼，高并发时代谁能称雄？》中所介绍的一样，Linux之所以能取得如此的成功，很大程度上是依靠C语言有如少林般的江湖地位，C语言像少林一样佛门广开，广结善缘，无论是扫地僧还是火工头陀都是门下的弟子，也正是在众多开发者的共同努力下，Linux项目才成为了当今世界上最受欢迎，贡献人数最多、使用人数也是最多的操作系统。

但Rust难的却像是火星语，Rust中多路通道在使用之前要clone，带锁的哈希表用之前要先unwrap，种种用法和C、Java完全不同，从这个角度上讲Rust很像逍遥派，想入门非常难，但只要能出师，写的程序能通过编译，那你百分百是一位高手，所以这是一门下限很高，上限同样也很高的极致语言。

正如著名的Linux开发者Greg Kroah Hartman所说，他虽然喜欢将Rust引入内核开发的想法，但这项工作还有很长的路要走，由于Rust语言的学习成本极高，因此要求开发人员至少在5年内使用Rust编写代码是不可能的。笔者的看法基本与Hartman相同，虽然Rust For Linux前任很光明，但是道路肯定会是曲折的。

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

[☞](http://mp.weixin.qq.com/s?__biz=MzA5MzY4NTQwMA==&mid=2651034313&idx=2&sn=ce6899f9efcc891009ae112bf6351c32&chksm=8bad213ebcdaa82805b989aab2dafa514f8ada2799a069fa49dc3da36ae61f02f6d229b8c77a&scene=21#wechat_redirect)[马斯克公开支持上班“摸鱼”，允许员工坐班听音乐，还可外放！](http://mp.weixin.qq.com/s?__biz=MzA5MzY4NTQwMA==&mid=2651056576&idx=1&sn=522f24620ed8a151ba3669668d1b7b72&chksm=8bad5a37bcdad3212af23cfba2374cbf11445c60d449935d24ba9ca969375c7813640fad8639&scene=21#wechat_redirect)

[☞](http://mp.weixin.qq.com/s?__biz=MzA5MzY4NTQwMA==&mid=2651034313&idx=2&sn=ce6899f9efcc891009ae112bf6351c32&chksm=8bad213ebcdaa82805b989aab2dafa514f8ada2799a069fa49dc3da36ae61f02f6d229b8c77a&scene=21#wechat_redirect)[编程界也有修仙秘籍？程序员码字3年终得《JavaScript 百炼成仙》](http://mp.weixin.qq.com/s?__biz=MzA5MzY4NTQwMA==&mid=2651056531&idx=1&sn=ff885969659c0d0e7753bbf31c6b1a91&chksm=8bad5a64bcdad37223a3206500b5a033cf90365f56b93b99aec04174b29d58eb303f26bd85a7&scene=21#wechat_redirect)

☞[年薪高达115万元，Rust成2021年最赚钱的编程语言](http://mp.weixin.qq.com/s?__biz=MzA5MzY4NTQwMA==&mid=2651056491&idx=1&sn=90bc3048a5925aab8f54df350f390226&chksm=8bad5a9cbcdad38a5669bf97983a30f6ec8fc3b94ce6047bb74fd65ec148274f02985642a7a3&scene=21#wechat_redirect)

[☞](http://mp.weixin.qq.com/s?__biz=MzA5MzY4NTQwMA==&mid=2651034313&idx=2&sn=ce6899f9efcc891009ae112bf6351c32&chksm=8bad213ebcdaa82805b989aab2dafa514f8ada2799a069fa49dc3da36ae61f02f6d229b8c77a&scene=21#wechat_redirect)[上班摸鱼被通报、开除，国美回应：工作时间使用非正常流量，系遵循员工手册](http://mp.weixin.qq.com/s?__biz=MzA5MzY4NTQwMA==&mid=2651056475&idx=2&sn=a635d6d03a9e780572eff135da5a99ff&chksm=8bad5aacbcdad3ba93c0e6fd55868e9d2e26e03e781d9684abb25e131e1014698425ede1b4c9&scene=21#wechat_redirect)

☞[“加班真好”？知名大厂挂花式标语引热议，员工：不加班完不成任务](http://mp.weixin.qq.com/s?__biz=MzA5MzY4NTQwMA==&mid=2651056392&idx=1&sn=73583bc81b233685cfecf254f55fdeae&chksm=8bad5affbcdad3e98bb6d7c8d23f6b1fe7a8b7e3543a5089707cfc10b5d34c040074d63afd54&scene=21#wechat_redirect)

![](https://res.wx.qq.com/op_res/NN_GToMiIjsXzgPzF9-74ZzwR3cA9-fv3o9eWo8f5gQWqx71CmGlY8kFxuIxZaG0TB1bFeMCmh1DGN_pWMRg0A)

**帐号已迁移**

公众号

Reads 4962

​

Comment

**留言 5**

- 清尘

  2021年11月23日

  Like8

  所有的语言都和C比，这已经说明C语言的地位了

- 啊良梓是我

  2021年11月24日

  Like1

  C: 同志们加油！

- 半⃰夏⃰ㅤ

  2021年11月23日

  Like1

  rust是为了干掉C++的

- 小金理财客服

  2021年11月24日

  Like

  梦想真滴大。要干掉C语言。走着瞧吧![[微笑]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

- 花城

  2021年11月23日

  Like

  就...图个乐吧，没有太多的期待![[捂脸]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![[捂脸]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![[捂脸]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

已无更多数据

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/hVd1jlHjia9vlnCYH4qvibHKiahN6tS7x40PGUfjewe8uS3niagDYEDtxq9lyUQDotNdlRbyo0uErk8AEbR6O5gTibQ/300?wx_fmt=png&wxfrom=18)

CSDN程序人生

5Share2

5

Comment

**留言 5**

- 清尘

  2021年11月23日

  Like8

  所有的语言都和C比，这已经说明C语言的地位了

- 啊良梓是我

  2021年11月24日

  Like1

  C: 同志们加油！

- 半⃰夏⃰ㅤ

  2021年11月23日

  Like1

  rust是为了干掉C++的

- 小金理财客服

  2021年11月24日

  Like

  梦想真滴大。要干掉C语言。走着瞧吧![[微笑]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

- 花城

  2021年11月23日

  Like

  就...图个乐吧，没有太多的期待![[捂脸]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![[捂脸]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![[捂脸]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

已无更多数据
