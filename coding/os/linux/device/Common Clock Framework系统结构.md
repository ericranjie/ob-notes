# [蜗窝科技](http://www.wowotech.net/)

### 慢下来，享受技术。

[![](http://www.wowotech.net/content/uploadfile/201401/top-1389777175.jpg)](http://www.wowotech.net/)

- [博客](http://www.wowotech.net/)
- [项目](http://www.wowotech.net/sort/project)
- [关于蜗窝](http://www.wowotech.net/about.html)
- [联系我们](http://www.wowotech.net/contact_us.html)
- [支持与合作](http://www.wowotech.net/support_us.html)
- [登录](http://www.wowotech.net/admin)

﻿

## 

作者：[linuxer](http://www.wowotech.net/author/3 "linuxer") 发布于：2016-4-21 19:23 分类：[电源管理子系统](http://www.wowotech.net/sort/pm_subsystem)

一、前言

之前，wowo同学已经发表了关于CCF（[Common Clock Framework](http://www.wowotech.net/linux_kenrel/clk_overview.html)）的三份文档，相信大家对CCF有一定的了解了，本文就是在阅读那三份文档的基础上，针对Linux 4.4.6内核的内核代码实现，记录自己对CCF的理解，并对CCF进行系统结构层面的归纳和整理。

本文内容包括三个部分，第二章给出了整个CCF相关的block diagram图，随后在第三章对各个模块进行功能层面的描述。最后，第四章给出了各个block之间的接口描述。

另外，在阅读CCF代码的过程中，我准备用两份文档来分享我对CCF的理解。这一份是系统结构，另外一份是逻辑解析。

二、软件框架

1、CCF 相关的block diagram

CCF的框架如下图所示：

[![clk-arch](http://www.wowotech.net/content/uploadfile/201604/7af03a46e4a61486c507dd2bc68758dc20160421112336.gif "clk-arch")](http://www.wowotech.net/content/uploadfile/201604/ab50b4f724efcfbc9a68a285376d952920160421112334.gif)

上图中，位于图片中心的那个青色的Block就是CCF模块，和硬件无关，实现了通用的clock设备的逻辑。其他的软件模块都不属于CCF的范畴，但是会和CCF有接口进行通信。随后的两个章节会对各个block以及接口进行描述。

三、软件框架图中各个block的功能描述

1、HW clock device tree

Clock device是指那些能够产生clock信号、控制clock信号的设备，例如晶振、分频器什么的，本小节主要是从HW 工程师的角度来描述这些设备极其拓扑结构。系统中的clock distribution形成了类似文件系统那样的树状结构，和文件系统树不同的是：clock tree有多个根节点，形成多个clock tree，而文件系统树只有一个根节点；文件系统树的中间节点是目录，叶节点是文件，而clock tree相对会复杂一些，它包括如下的的节点：

（1）根节点一般是Oscillator（有源振荡器）或者Crystal（无源振荡器，即大家经常说的晶振）。

（2）中间节点有很多种，包括PLL（锁相环，用于提升频率的），Divider（分频器，用于降频的），mux（从多个clock path中选择一个），开关（用来控制ON/OFF的）。

（3）叶节点是使用clock做为输入的、有具体功能的HW block。

了解了clock tree的结构之后，我们来看看具体的操作。对于叶节点，或者说clock consumer而言，没有什么可以控制的，HW block只是享用这个clock source而已。这里需要注意的是：虽然HW clock的datasheet往往有clock gating的内容（或者一些针对clock source进行分频的内容，概念类似），但是，本质上负责clock gating的那个硬件模块需要独立出来，称为为clock tree中的一个中间节点。而对中间节点的设定包括：

（1）ON/OFF控制

（2）频率设定、相位控制等

（3）从众多输入的clock source中选择一个做为输出。

2、HW-specific Clock provider driver

这个模块是真正和系统中实际的clock device打交道的模块。与其说Clock provider driver，不如说是clock provider drivers（复数），主要用来驱动系统中的clock tree中的各个节点上clock device（不包括clock consumer device），只要该HW block对外提供时钟信号，那么它就是一个clock provider，就有对应的clock provider driver。

上一小节我们已经了解到，系统中的clock device非常多种，什么VCO啦、什么分频器啦，什么复用器啦，其功能各不相同，但是本质上都属于clock device，Linux kernel把这些clock HW block的特性抽取出来，用struct clk_hw 来表示，具体如下：

> struct clk_hw {  
>     struct clk_core *core;－－－－－－－－（1）  
>     struct clk *clk;－－－－－－－－－－－（2）  
>     const struct clk_init_data *init;－－－－（3）  
> };

（1）指向CCF模块中对应clock device实例。由于系统中的clk_hw和clk_core实例是一一对应的，因此，struct clk_core中也有指回clk_hw的数据成员。

（2）clk是访问clk_core的实例，每当consumer通过clk_get对CCF中的clock device（也就是clk_core）发起访问的时候都需要获取一个句柄，也就是clk。每一个用户访问都会有一个clk句柄，同样的，底层模块对其访问亦然。因此，这里clk是底层clk_hw访问clk_core的句柄实例。

（3）在底层clock provider driver初始化的过程中，会调用clk_register接口函数注册clk_hw。当然，这时候需要设定一些初始数据，而这些初始数据被抽象成一个struct clk_init_data数据结构。在初始化过程中，clk_init_data的数据被用来初始化clk_hw对应的clk_core数据结构，当初始化完成之后，clk_init_data则没有存在的意义了，具体struct clk_init_data的定义如下：

> struct clk_init_data {  
>     const char        *name;－－－－－－－－－－－－－－（1）  
>     const struct clk_ops    *ops;－－－－－－－－－－－－（2）  
>     const char        * const *parent_names;－－－－－－－（3）  
>     u8            num_parents;  
>     unsigned long        flags;－－－－－－－－－－－－－－（4）  
> };

（1）该clock设备的名字。

（2）consumer访问clock设备的时候往往是首先获取clk句柄，然后找到clk_core，之后即可通过clk_core的struct clk_ops中的callback函数可以进入具体的clock provider driver层进行具体的HW 操作。struct clk_ops这个数据结构是底层驱动必须要准备好的数据之一，在注册的时候通过clk_init_data带入CCF层。

（3）描述该clk_hw的拓扑结构，通过这样的信息，CCF层可以建立clk_core的拓扑结构来跟踪实际clock设备的拓扑。

（4）CCF级别的flag。

struct clk_hw应该是clock device的基类，所有的具体的clock device都应该由它派生出来，例如固定频率的振动器，它的数据结构是：

> struct clk_fixed_rate {  
>     struct        clk_hw hw;－－－－－－－－－－－基类  
>     unsigned long    fixed_rate;－－－－－－－－－下面是fixed rate这种clock device特有的成员  
>     unsigned long    fixed_accuracy;  
>     u8        flags;  
> };

其他的特定的clock device大概都是如此，这里就不赘述了。在构建自己硬件平台的clock provider驱动的时候，如果能够使用clock device的派生类，尽量不要使用struct clk_hw这个基类，尽量不要重复造轮子。

3、CCF模块

CCF模块，wowo的文章已经说了很多，我这里从文件的角度来说一说该模块。CCF模块的文件汇整如下：

|   |   |   |   |
|---|---|---|---|
|文件名|类别|对应的框架图中的元素|描述|
|clk.h|接口|common clock interface  <br>clk devm interface|为各种clock consumer模块提供通用的clock控制接口|
|clkdev.h|接口|clkdev interface|为各种clock provider模块提供的clkdev接口，主要用于consumer寻找CCF中的clk数据结构。|
|clk-provider.h|接口|common provider interface  <br>specific provider interface|为各种硬件相关的的clock provider模块提供注册、注销的接口。|
|clk.c|模块|clk|CCF的核心模块，对上提供查找，管理各种clk实例的功能，对内，提供管理和各种clock device（struct clk_core）的功能，对下，接收来自底层硬件的注册注销请求，维护clk、clk_core和clk_hw之间的连接。|
|clkdev.c|模块|devclk|该模块主要用来维护clk结构和其对应的clk名字之间的关系，用于clk的lookup。|
|clk-devres.c|模块|clk-devres|设备自动管理clk资源模块（参考[device resource management](http://www.wowotech.net/linux_kenrel/device_resource_management.html)）|
|clk-conf.c|模块|clk-conf|该模块主要用来实现在启动阶段的时候，通过分析DTS来设定一下default参数|
|clk-xxx.c|模块|clk-xxx|这些模块包括：clk-divider.c clk-fixed-factor.c clk-fixed-rate.c clk-gate clk-multiplier.c clk-mux.c clk-composite.c clk-fractional-divider.c clk-gpio.c。这些都是CCF模块对某一类clock设备进行抽象，方便具体clock provider driver工程师撰写driver的时候不至于从头开始。|
|drivers/clk目录其他文件|N/A|N/A|这些文件都是具体硬件平台上的各种clock provider 驱动|

此外，CCF模块的核心数据结构就有两个：

（1）struct clk_core。这个数据结构是CCF层对clock device的抽象，每一个实际的硬件clock device（struct clk_hw）都会对应一个clk_core，CCF模块负责建立整个抽象的clock tree的树状结构并维护这些数据。具体如何维护呢？这里不得不给出几个链表头的定义，如下：

> static HLIST_HEAD(clk_root_list);  
> static HLIST_HEAD(clk_orphan_list);

CCF layer有2条全局的链表：clk_root_list和clk_orphan_list。所有设置了CLK_IS_ROOT属性的clock 都会挂在clk_root_list中，而这个链表中的每一个阶段又展成一个树状结构。（这和硬件拓扑是吻合的，clock tree实际上是有多个根节点的，多条树状结构）。其它clock，如果有valid的parent ，则会挂到parent的“children”链表中，如果没有valid的parent，则会挂到clk_orphan_list中。

（2）struct clk。这个数据结构和consumer的访问有关，基本上，每一个user对clock device的访问都会创建一个访问句柄，这个句柄就是clk。不同的user访问同样的clock device的时候，虽然是同一个struct clk_core实例，但是其访问的句柄（clk）是不一样的。CCF如何管理clk呢？这里说来就话长了，在DTS还没有引入kenrel的时代，struct clk_lookup用来管理clk数据，具体该数据结构定义如下：

> struct clk_lookup {  
>     struct list_head    node;－－－－－－－挂入clocks链表  
>     const char        *dev_id;－－－－－－－dev_id和con_id是用来寻找适合的clk的  
>     const char        *con_id;  
>     struct clk        *clk;－－－－－－－－－对应的clk  
>     struct clk_hw        *clk_hw;  
> };

对于底层的clock provider驱动而言，除了调用clk_register函数注册到common clock framework中，还会调用clk_register_clkdev将该clk和一个名字捆绑起来（否则clock consumer并不知道如何定位到该clk），在CCF layer，clocks全局链表用来维护系统中所有的struct clk_lookup实例。通过这样的机制，clock consumer可以通过名字获取clk。

引入device tree之后，情况发生了一些变化。基本上每一个clock provider都会变成dts中的一个节点，也就是说，每一个clk都有一个设备树中的device node与之对应。在这种情况下，与其捆绑clk和一个“名字”，不如捆绑clk和device node，具体的数据结构如下：

> struct of_clk_provider {  
>     struct list_head link; －－－－－－挂入of_clk_providers全局链表
> 
>     struct device_node *node;－－－－该clock device的DTS节点  
>     struct clk *(*get)(struct of_phandle_args *clkspec, void *data);－－获取对应clk数据结构的函数  
>     void *data;  
> };

因此，对于底层provider driver而言，原来的clk_register + clk_register_clkdev的组合变成了clk_register + of_clk_add_provider的组合，在CCF layer保存了of_clk_providers全局链表来管理所有的DTS节点和clk的对应关系。

我们再看clock consumer这一侧：这时候，使用名字检索clk已经过时了，毕竟已经有了强大的device tree。我们可以通过clock consumer对应的struct device_node寻找为他提供clock signal那个clock设备对应的device node（clock属性和clock-names属性），当然，如果consumer有多个clock signal来源，那么在寻找的时候需要告知是要找哪一个时钟源（用connection ID标记）。当找了provider对应的device node之后，一切都变得简单了，从全局的clock provide链表中找到对应clk就OK了。在引入强大的设备树之后，clkdev模块按理说应该退出历史舞台了。

4、clock consumer driver

对于clock consumer driver而言，它其实就是调用CCF接口和DTS的接口来完成下面的功能：

（1）初始化的时候，调用common clk interface来对其clock设备（如果有需要，可能波及clock设备的上游设备）进行设定，以便让该HW block正常运作起来。

（2）根据用户需求（例如用户修改波特率），在运行时，clock consumer driver也会调用common clk interface来对其clock设备进行修改（例如修改clock rate，相位等）

（3）配合系统电源管理（TODO）。

5、设备树

具体请参考本站的相关文档。

四、接口描述

1、consumer dts interface

clock consumer的属性列表如下：

|   |   |   |
|---|---|---|
|属性|是否必须提供？|描述|
|clocks|必须|该属性描述clock consumer设备使用的clock source，或者clock input（可能不止一个哦）。  <br>该属性是一个数组，数组中每一个具体的entry对应一个clock source。而clock source是由phandle和clock specifier来描述。phandle指向一个clock provider的device node，如果该provider的#clock-cells等于0，那么说明该provider就一个output，那么就不需要clock specifier来进一步描述。如果该provider的#clock-cells不等于0，那么clock specifier必须提供，以便指明本设备到底使用provider输出时钟源的哪一路。|
|clock-names|可选|同样的，该属性也似描述设备使用的clock source信息的，也是一个数组，是一个字符串数组，每一个字符串描述一个clock source，对应着clocks中phandle和clock specifier。  <br>之所以提供clock-names这个属性其实是为了编程方便，驱动程序可以通过比较直观的clock name来找到该设备的输入时钟源信息。|
|clock-ranges|可选|该属性值为空，主要用来说明该设备的下级设备可以继承该设备的clock source。例如B设备是A设备的sub node，A设备如果有clock-ranges属性，那么B设备在寻找其clock source的时候，如果在本node定义的clock相关属性中没有能够找到，那么可以去A设备去继续寻找（也就是说，B设备会继承A设备的clock source相关的属性，也就是clocks或者clock-names这两个属性）。|

2、provider dts interface

clock provider的属性列表如下：

|   |   |   |
|---|---|---|
|属性|是否必须提供？|描述|
|#clock-cells|必须|我们上面说过了，一个HW block（clock consumer）的时钟源可以通过phandle和clock specifier来描述，phandle指向一个clock provider的device node，很明确，但是定位到clock provider并不行，因此有的provider会提供多路clock output给其他HW block使用，因此需要所谓的clock specifier来进一步描述。这里#clock-cells就是说明使用多少个cell（u32）来描述clock specifier。  <br>如果等于0，说明provider就一个clock output，不需要specifier，如果等于1，说明provider有多个clock output（能用u32标识）。等于2或者更大的情况应该不存在，一个provider不可能提供超过2^32个clock output。|
|clock-output-names|可选|如果clock provider能提供多路时钟输出，那么给每一个clock output起个适合人类阅读的名字是不错的选择，这也就是clock-output-names的目的。clock consumer中提供的clock specifier是一个index，通过这个index可以在clock-output-names属性值中找到对应的时钟源的名字。|
|clock-indices|可选|如果不提供这个属性，那么clock-output-names和index的对应关系就是0，1，2……。如果这个对应关系不是线性的，那么可以通过clock-indices属性来定义映射到clock-output-names的index。|

3、clock config interface

初始化时可以通过dts来设定clock parent以及clock rate，具体属性如下：

|   |   |   |
|---|---|---|
|属性|是否必须提供？|描述|
|assigned-clocks|可选|这个属性列出了需要进行设定的clock，其值是一个phandle＋clock specifier数组|
|assigned-clock-parent|可选|准备要设定的parent列表。“儿子”在哪里呢？assigned-clocks中定义的，注意，是一一对应的。例如：  <br>assigned-clocks：A， B，C；  <br>assigned-clock-parent：A_parent，B_parent，C_parent；|
|assigned-clock-rate|可选|要设定的频率列表，同样的，和assigned-clocks也是一一对应的。|

4、其他接口

请参考窝窝同学的文档，这里不再赘述。

五、参考文献

[1]     Documentation/devicetree/bindings/clock/clock-bindings.txt

[2]     Documentation/clk.txt

原创文章，转发请注明出处。蜗窝科技，[www.wowotech.net](http://www.wowotech.net/)。

标签: [framework](http://www.wowotech.net/tag/framework) [clock](http://www.wowotech.net/tag/clock) [common](http://www.wowotech.net/tag/common)

[![](http://www.wowotech.net/content/uploadfile/201605/ef3e1463542768.png)](http://www.wowotech.net/support_us.html)

« [使用pxe方式安装系统](http://www.wowotech.net/linux_application/288.html) | [Linux操作命令记录](http://www.wowotech.net/linux_application/linux_cmd_note.html)»

**评论：**

**[吴b](http://wu-being.blog.csdn.net/)**  
2020-04-03 19:26

我在display 平台代码中，找到pclk=devm_clk_get(dev,clk_name) 使用函数，打印出clk_name是没问题的，但是我想打印pclk的name和parent和rate，但编译提示成员没实现，有人了解是怎么回事吗？比如下面：  
  
pclk=devm_clk_get(dev,clk_name);  
printk("%s", pclk->core->name);// build error

[回复](http://www.wowotech.net/pm_subsystem/ccf-arch.html#comment-7938)

**Pythonic**  
2019-06-24 01:06

assigned-clock-parent并不一定一一对应，clock-bindings.txt里面是这么说的“To skip setting parent or rate of a clock its corresponding entry should be set to 0, or can be omitted if it is not followed by any non-zero entry.”，clock-bindings.txt中的例子：  
    uart@a000 {  
        compatible = "fsl,imx-uart";  
        reg = <0xa000 0x1000>;  
        ...  
        clocks = <&osc 0>, <&pll 1>;  
        clock-names = "baud", "register";  
  
        assigned-clocks = <&clkcon 0>, <&pll 2>;  
        assigned-clock-parents = <&pll 2>;  
        assigned-clock-rates = <0>, <460800>;  
    };

[回复](http://www.wowotech.net/pm_subsystem/ccf-arch.html#comment-7483)

**MDO3054**  
2017-12-20 20:38

“这一份是系统结构，另外一份是逻辑解析。”  
另一份在哪里？

[回复](http://www.wowotech.net/pm_subsystem/ccf-arch.html#comment-6395)

**[linuxer](http://www.wowotech.net/)**  
2017-12-21 11:09

@MDO3054：另外一份没有写，哈哈......

[回复](http://www.wowotech.net/pm_subsystem/ccf-arch.html#comment-6403)

**向往**  
2017-09-22 14:13

@蜗窝 在用户层有什么办法可以查看CCF下管理的各个时钟频率的大小吗

[回复](http://www.wowotech.net/pm_subsystem/ccf-arch.html#comment-6059)

**[wowo](http://www.wowotech.net/)**  
2017-09-22 18:51

@向往：你是指这些东西吗？  
shell@flo:/ $ ls /sys/kernel/debug/clk/  
adm0_clk  
adm0_p_clk  
afab_a_clk  
afab_acpu_a_clk  
...  
vcodec_p_clk  
vfe_axi_clk  
vfe_clk  
vfe_p_clk  
vpe_axi_clk  
vpe_clk  
vpe_p_clk  
  
shell@flo:/ $ ls /sys/kernel/debug/clk/vfe_clk/  
enable          has_hw_gating   list_rates      rate  
fmax_rates      is_local        measure

[回复](http://www.wowotech.net/pm_subsystem/ccf-arch.html#comment-6060)

**向往**  
2017-09-23 11:45

@wowo：是的，原来挂载debugfs后才能看到，谢谢蜗窝@蜗窝！

[回复](http://www.wowotech.net/pm_subsystem/ccf-arch.html#comment-6064)

**lsaxl**  
2017-09-11 16:07

驱动里大都使用clk_get  
  
为什么还要单独维护，而不使用__clk_lookup这个接口呢

[回复](http://www.wowotech.net/pm_subsystem/ccf-arch.html#comment-6023)

**lsaxl**  
2017-09-11 16:14

@lsaxl：我的意思是__clk_lookup就可以找到clk，为何还使用如下两个组合  
原来的clk_register + clk_register_clkdev的组合变成了clk_register + of_clk_add_provider的组合

[回复](http://www.wowotech.net/pm_subsystem/ccf-arch.html#comment-6024)

**[wowo](http://www.wowotech.net/)**  
2017-09-12 13:29

@lsaxl：__clk_lookup是给consumer用的api，clk_register + clk_register_clkdev或者clk_register + of_clk_add_provider是给provider用的api，我不知道你怎么会把它们搅和在一起？

[回复](http://www.wowotech.net/pm_subsystem/ccf-arch.html#comment-6030)

**DESW**  
2017-09-17 11:41

@wowo：我的意思是clk_get的实现：  
比如使用dts注册provider后consumer使用__clk_lookup找到设备  
何必再of_clk_add_provider  
  
clk_get的实现为何不基于__clk_lookup（dts方式）  
  
不过我想通了，因为有好多简略版的dts，dts中不包含实际的clk

[回复](http://www.wowotech.net/pm_subsystem/ccf-arch.html#comment-6046)

**发表评论：**

 昵称

 邮件地址 (选填)

 个人主页 (选填)

![](http://www.wowotech.net/include/lib/checkcode.php) 

- ### 站内搜索
    
       
     蜗窝站内  互联网
    
- ### 功能
    
    [留言板  
    ](http://www.wowotech.net/message_board.html)[评论列表  
    ](http://www.wowotech.net/?plugin=commentlist)[支持者列表  
    ](http://www.wowotech.net/support_list)
- ### 最新评论
    
    - Shiina  
        [一个电路（circuit）中，由于是回路，所以用电势差的概念...](http://www.wowotech.net/basic_subject/voltage.html#8926)
    - Shiina  
        [其中比较关键的点是相对位置概念和点电荷的静电势能计算。](http://www.wowotech.net/basic_subject/voltage.html#8925)
    - leelockhey  
        [你这是哪个内核版本](http://www.wowotech.net/pm_subsystem/generic_pm_architecture.html#8924)
    - ja  
        [@dream：我看完這段也有相同的想法，引用 @dream ...](http://www.wowotech.net/kernel_synchronization/spinlock.html#8922)
    - 元神高手  
        [围观首席power managerment专家](http://www.wowotech.net/pm_subsystem/device_driver_pm.html#8921)
    - 十七  
        [内核空间的映射在系统启动时就已经设定好，并且在所有进程的页表...](http://www.wowotech.net/process_management/context-switch-arch.html#8920)
- ### 文章分类
    
    - [Linux内核分析(25)](http://www.wowotech.net/sort/linux_kenrel) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=4)
        - [统一设备模型(15)](http://www.wowotech.net/sort/device_model) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=12)
        - [电源管理子系统(43)](http://www.wowotech.net/sort/pm_subsystem) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=13)
        - [中断子系统(15)](http://www.wowotech.net/sort/irq_subsystem) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=14)
        - [进程管理(31)](http://www.wowotech.net/sort/process_management) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=15)
        - [内核同步机制(26)](http://www.wowotech.net/sort/kernel_synchronization) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=16)
        - [GPIO子系统(5)](http://www.wowotech.net/sort/gpio_subsystem) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=17)
        - [时间子系统(14)](http://www.wowotech.net/sort/timer_subsystem) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=18)
        - [通信类协议(7)](http://www.wowotech.net/sort/comm) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=20)
        - [内存管理(31)](http://www.wowotech.net/sort/memory_management) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=21)
        - [图形子系统(2)](http://www.wowotech.net/sort/graphic_subsystem) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=23)
        - [文件系统(5)](http://www.wowotech.net/sort/filesystem) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=26)
        - [TTY子系统(6)](http://www.wowotech.net/sort/tty_framework) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=27)
    - [u-boot分析(3)](http://www.wowotech.net/sort/u-boot) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=25)
    - [Linux应用技巧(13)](http://www.wowotech.net/sort/linux_application) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=3)
    - [软件开发(6)](http://www.wowotech.net/sort/soft) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=1)
    - [基础技术(13)](http://www.wowotech.net/sort/basic_tech) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=6)
        - [蓝牙(16)](http://www.wowotech.net/sort/bluetooth) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=10)
        - [ARMv8A Arch(15)](http://www.wowotech.net/sort/armv8a_arch) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=19)
        - [显示(3)](http://www.wowotech.net/sort/display) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=22)
        - [USB(1)](http://www.wowotech.net/sort/usb) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=28)
    - [基础学科(10)](http://www.wowotech.net/sort/basic_subject) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=7)
    - [技术漫谈(12)](http://www.wowotech.net/sort/tech_discuss) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=8)
    - [项目专区(0)](http://www.wowotech.net/sort/project) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=9)
        - [X Project(28)](http://www.wowotech.net/sort/x_project) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=24)
- ### 随机文章
    
    - [Linux下“用户空间修改设备寄存器或者物理内存”的实现](http://www.wowotech.net/soft/186.html)
    - [MinGW下安装man工具包](http://www.wowotech.net/linux_application/8.html)
    - [X-006-UBOOT-pinctrl driver移植(Bubblegum-96平台)](http://www.wowotech.net/x_project/bubblegum_uboot_pinctrl.html)
    - [X-019-KERNEL-串口驱动开发之数据收发](http://www.wowotech.net/x_project/serial_driver_porting_4.html)
    - [Linux时间子系统之（一）：时间的基本概念](http://www.wowotech.net/timer_subsystem/time_concept.html)
- ### 文章存档
    
    - [2024年2月(1)](http://www.wowotech.net/record/202402)
    - [2023年5月(1)](http://www.wowotech.net/record/202305)
    - [2022年10月(1)](http://www.wowotech.net/record/202210)
    - [2022年8月(1)](http://www.wowotech.net/record/202208)
    - [2022年6月(1)](http://www.wowotech.net/record/202206)
    - [2022年5月(1)](http://www.wowotech.net/record/202205)
    - [2022年4月(2)](http://www.wowotech.net/record/202204)
    - [2022年2月(2)](http://www.wowotech.net/record/202202)
    - [2021年12月(1)](http://www.wowotech.net/record/202112)
    - [2021年11月(5)](http://www.wowotech.net/record/202111)
    - [2021年7月(1)](http://www.wowotech.net/record/202107)
    - [2021年6月(1)](http://www.wowotech.net/record/202106)
    - [2021年5月(3)](http://www.wowotech.net/record/202105)
    - [2020年3月(3)](http://www.wowotech.net/record/202003)
    - [2020年2月(2)](http://www.wowotech.net/record/202002)
    - [2020年1月(3)](http://www.wowotech.net/record/202001)
    - [2019年12月(3)](http://www.wowotech.net/record/201912)
    - [2019年5月(4)](http://www.wowotech.net/record/201905)
    - [2019年3月(1)](http://www.wowotech.net/record/201903)
    - [2019年1月(3)](http://www.wowotech.net/record/201901)
    - [2018年12月(2)](http://www.wowotech.net/record/201812)
    - [2018年11月(1)](http://www.wowotech.net/record/201811)
    - [2018年10月(2)](http://www.wowotech.net/record/201810)
    - [2018年8月(1)](http://www.wowotech.net/record/201808)
    - [2018年6月(1)](http://www.wowotech.net/record/201806)
    - [2018年5月(1)](http://www.wowotech.net/record/201805)
    - [2018年4月(7)](http://www.wowotech.net/record/201804)
    - [2018年2月(4)](http://www.wowotech.net/record/201802)
    - [2018年1月(5)](http://www.wowotech.net/record/201801)
    - [2017年12月(2)](http://www.wowotech.net/record/201712)
    - [2017年11月(2)](http://www.wowotech.net/record/201711)
    - [2017年10月(1)](http://www.wowotech.net/record/201710)
    - [2017年9月(5)](http://www.wowotech.net/record/201709)
    - [2017年8月(4)](http://www.wowotech.net/record/201708)
    - [2017年7月(4)](http://www.wowotech.net/record/201707)
    - [2017年6月(3)](http://www.wowotech.net/record/201706)
    - [2017年5月(3)](http://www.wowotech.net/record/201705)
    - [2017年4月(1)](http://www.wowotech.net/record/201704)
    - [2017年3月(8)](http://www.wowotech.net/record/201703)
    - [2017年2月(6)](http://www.wowotech.net/record/201702)
    - [2017年1月(5)](http://www.wowotech.net/record/201701)
    - [2016年12月(6)](http://www.wowotech.net/record/201612)
    - [2016年11月(11)](http://www.wowotech.net/record/201611)
    - [2016年10月(9)](http://www.wowotech.net/record/201610)
    - [2016年9月(6)](http://www.wowotech.net/record/201609)
    - [2016年8月(9)](http://www.wowotech.net/record/201608)
    - [2016年7月(5)](http://www.wowotech.net/record/201607)
    - [2016年6月(8)](http://www.wowotech.net/record/201606)
    - [2016年5月(8)](http://www.wowotech.net/record/201605)
    - [2016年4月(7)](http://www.wowotech.net/record/201604)
    - [2016年3月(5)](http://www.wowotech.net/record/201603)
    - [2016年2月(5)](http://www.wowotech.net/record/201602)
    - [2016年1月(6)](http://www.wowotech.net/record/201601)
    - [2015年12月(6)](http://www.wowotech.net/record/201512)
    - [2015年11月(9)](http://www.wowotech.net/record/201511)
    - [2015年10月(9)](http://www.wowotech.net/record/201510)
    - [2015年9月(4)](http://www.wowotech.net/record/201509)
    - [2015年8月(3)](http://www.wowotech.net/record/201508)
    - [2015年7月(7)](http://www.wowotech.net/record/201507)
    - [2015年6月(3)](http://www.wowotech.net/record/201506)
    - [2015年5月(6)](http://www.wowotech.net/record/201505)
    - [2015年4月(9)](http://www.wowotech.net/record/201504)
    - [2015年3月(9)](http://www.wowotech.net/record/201503)
    - [2015年2月(6)](http://www.wowotech.net/record/201502)
    - [2015年1月(6)](http://www.wowotech.net/record/201501)
    - [2014年12月(17)](http://www.wowotech.net/record/201412)
    - [2014年11月(8)](http://www.wowotech.net/record/201411)
    - [2014年10月(9)](http://www.wowotech.net/record/201410)
    - [2014年9月(7)](http://www.wowotech.net/record/201409)
    - [2014年8月(12)](http://www.wowotech.net/record/201408)
    - [2014年7月(6)](http://www.wowotech.net/record/201407)
    - [2014年6月(6)](http://www.wowotech.net/record/201406)
    - [2014年5月(9)](http://www.wowotech.net/record/201405)
    - [2014年4月(9)](http://www.wowotech.net/record/201404)
    - [2014年3月(7)](http://www.wowotech.net/record/201403)
    - [2014年2月(3)](http://www.wowotech.net/record/201402)
    - [2014年1月(4)](http://www.wowotech.net/record/201401)

[![订阅Rss](http://www.wowotech.net/content/templates/default/images/rss.gif)](http://www.wowotech.net/rss.php "RSS订阅")

Copyright @ 2013-2015 [蜗窝科技](http://www.wowotech.net/ "wowotech") All rights reserved. Powered by [emlog](http://www.emlog.net/ "采用emlog系统")