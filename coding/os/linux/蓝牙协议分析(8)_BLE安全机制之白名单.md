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

作者：[wowo](http://www.wowotech.net/author/2 "runangaozhong@163.com") 发布于：2016-11-10 14:20 分类：[蓝牙](http://www.wowotech.net/sort/bluetooth)

## 1. 前言

在万物联网的时代，安全问题将会受到非常严峻的挑战（相应地，也会获得最大的关注度），因为我们身边的每一个IOT设备，都是一个处于封印状态的天眼，随时都有被开启的危险。想想下面的场景吧：

> 凌晨2点，x米手环的闹钟意外启动，将你从睡梦中惊醒，然后床头的灯光忽明忽暗……
> 
> 你的心率、血压、睡眠质量等信息，默默地被竞争对手收集着，并通过大数据分析你的情绪、健康等，随时准备给你致命一击……
> 
> 我知道你家里有几盏灯、几台电器、几个人，知道你几点睡觉几时醒来，知道你一周做过几顿饭，甚至知道你有一个xx棒、一周使用几次、每次使用多久……
> 
> ……

算了，不罗列了，有时间的话可以建个iot eyes的站点，专门收集、整理物联网安全有关的内容。这里就先言归正传。

经过前面几篇的蓝牙协议分析，我们对蓝牙（特别是蓝牙低功耗）已经有了一个比较全面的了解。随后几篇文章，我会focus在BLE的安全机制上。毕竟，知己知彼，才能攻防有度。

话说，蓝牙SIG深知物联网安全的水有多深，因此使用了大量的篇幅，定义BLE安全有关的机制，甚至可以不夸张的说，BLE协议中内容最多、最难理解的部分，非安全机制莫属。本文先从介绍最简单的----白名单机制（White list）。

## 2. 白名单机制

白名单（white list）是BLE协议中最简单、直白的一种安全机制。其原理很简单，总结如下（前面的分析文章中都有介绍）：

> 所谓的白名单，就是一组蓝牙地址；
> 
> 通过白名单，可以只允许特定的蓝牙设备（白名单中列出的）扫描（Scan）、连接（connect）我们，也可以只扫描、连接特定的蓝牙设备（白名单中列出的）。

例如，如果某个BLE设备，只需要被受信任的某几个设备扫描、连接，我们就可以把这些受信任设备的蓝牙地址加入到该设备的白名单中，这样就可以有效避免其它“流氓设备”的骚扰了。

不过呢，该机制只防君子不防小人，因为它是靠地址去过滤“流氓”的，如果有些资深流氓，伪装一下，将自己的设备地址修改为受信任设备的地址，那就惨了……

## 3. 白名单有关的HCI命令

注1：本文主要从HCI的角度分析、介绍，如非必要，不再会涉及HCI之下的BLE协议（后续的分析文章，也大抵如此）。

#### 3.1 白名单维护相关的命令

BLE协议在HCI层定义了4个和白名单维护有关的命令，分别如下：

1）LE Read White List Size Command，获取controller可保存白名单设备的个数

该命令的格式为：

|   |   |   |
|---|---|---|
|OCF|Command parameters|Return Parameters|
|0x000F||Status  <br>White_List_Size|

> Status，命令执行的结果，0为success。
> 
> White_List_Size，size，范围是1～255。

注2：由此可知，白名单是保存在controller中，由于size的范围是1～255，因此controller必须实现白名单功能（最少保存一个）。

2）LE Clear White List Command，将controller中的白名单清空

该命令的格式为：

|   |   |   |
|---|---|---|
|OCF|Command parameters|Return Parameters|
|0x0010||Status|

> Status，命令执行的结果，0为success。

3）LE Add Device To White List Command，将指定的设备添加到白名单

该命令的格式为：

|   |   |   |
|---|---|---|
|OCF|Command parameters|Return Parameters|
|0x0011|Address_type(1 byte)  <br>Address(6 bytes)|Status|

> Address_type，设备的地址类型[1]，0为Public Device Address，1为Random Device Address。
> 
> Address，设备的地址。
> 
> Status，命令执行的结果，0为success。

4）LE Remove Device From White List Command，将指定的设备从白名单中移除的命令

该命令的格式为：

|   |   |   |
|---|---|---|
|OCF|Command parameters|Return Parameters|
|0x0012|Address_type(1 byte)  <br>Address(6 bytes)|Status|

> Address_type，设备的地址类型[1]，0为Public Device Address，1为Random Device Address。
> 
> Address，设备的地址。
> 
> Status，命令执行的结果，0为success。

最后需要说明的是，当controller处于以下三个状态的时候，以上命令除“LE Read Resolving List Size Command”外，均不能执行：

> 正在advertising；
> 
> 正在scanning；
> 
> 正在connecting。

#### 3.2 白名单使用策略有关的命令

BLE设备在发起Advertising、Scanning或者Connecting操作的时候，可以通过Set Advertising Parameters、Set Scan Parameters或者LE Create Connection Command，设置Advertising、Scanning或者Connecting的过滤策略（Filter_Policy），具体如下：

1）Advertising时的白名单策略

LE Set Advertising Parameters Command的命令格式为：

|   |   |   |
|---|---|---|
|OCF|Command parameters|Return Parameters|
|0x0006|…  <br>Advertising_Filter_Policy(1 byte)|Status|

该命令的其它参数请参考[2]，Advertising_Filter_Policy的含义如下：

> 0x00，禁用白名单机制，允许任何设备连接和扫描。
> 
> 0x01，允许任何设备连接，但只允许白名单中的设备扫描（scan data中有敏感信息？）。
> 
> 0x02，允许任何设备扫描，但只允许白名单中的设备连接。
> 
> 0x03，只允许白名单中的设备扫描和连接。

2）Scanning时的白名单策略

LE Set Scan Parameters Command的命令格式为：

|   |   |   |
|---|---|---|
|OCF|Command parameters|Return Parameters|
|0x000B|…  <br>Scanning_Filter_Policy(1 byte)|Status|

该命令的其它参数请参考[2]，Scanning_Filter_Policy的含义如下：

> 0x00，禁用白名单机制，接受所有的广播包（除了那些不是给我的directed advertising packets）。
> 
> 0x01，只接受在白名单中的那些设备发送的广播包（除了那些不是给我的directed advertising packets）。
> 
> 0x02，和白名单无关，不再介绍。
> 
> 0x03，接受如下的广播包：在白名单中的那些设备发送的广播包；广播者地址为resolvable private address的directed advertising packets；给我的给我的directed advertising packets。

注3：Scanning时的白名单策略有点奇怪，既然是主动发起的，要白名单的意义就不大了吧？

3）Connecting时的白名单策略

LE Create Connection Command的命令格式为：

|   |   |   |
|---|---|---|
|OCF|Command parameters|Return Parameters|
|0x000D|…  <br>Initiator_Filter_Policy(1 byte)  <br>…|Status|

该命令的其它参数请参考[4]，Initiator_Filter_Policy的含义如下：

> 0x00，禁用白名单机制，使用Peer_Address_Type and Peer_Address指定需要连接的设备。
> 
> 0x01，连接那些在白名单中的设备，不需要提供Peer_Address_Type and Peer_Address参数。

## 4. 使用示例

## 4.1 准备工作

后续的测试需要用到如下的设备和软件：

1）蓝牙设备A，作为Advertiser，发送广播数据，接受连接。

2）蓝牙设备B，作为Scanner，扫描设备A的广播数据，发起连接。

上述的1）和2）可以是如下一种：

> 一个具有bluez（hcitool等工具）的Android手机，可能需要较旧的android版本才行；
> 
> 带有蓝牙功能的树莓派，允许Debian、Ubuntu等系统（只要不是Android就行）；
> 
> Linux PC（或者虚拟机）加上一个具有BLE功能的蓝牙适配器。

3）bluez工具集，我们需要使用其中的hcitool命令。

#### 4.2 相关的hcitool命令说明

hcitool中的一些命令，和白名单机制有关，总结如下。

1）hcitool lewlsz，获取controller白名单的size，对应3.1中的LE Read White List Size Command，该命令不需要参数，可直接使用，如下：

> root@android:/ # hcitool lewlsz  
> hcitool lewlsz  
> White list size: 26

2）hcitool lewlclr，情况controller的白名单，对应3.1中的LE Clear White List Command，该命令也不需要参数，可直接使用，如下：

> root@android:/ # hcitool lewlclr  
> hcitool lewlclr

3）hcitool lewladd，将指定设备添加到白名单中，对应3.1中的LE Add Device To White List Command，其格式如下：

> root@android:/ # hcitool lewladd --help  
> hcitool lewladd --help  
> Usage:  
>         lewladd [--random]

其中是必选项，为要添加的蓝牙设备的地址。地址有public和random两种，默认是public，如果需要添加random类型的地址，则要指定--random参数，例如：

> root@android:/ # hcitool lewladd 22:22:21:CD:F4:58  
> hcitool lewladd 22:22:21:CD:F4:58
> 
> root@android:/ # hcitool lewladd --random 11:22:33:44:55:66  
> hcitool lewladd --random 11:22:33:44:55:66

4）hcitool lewlrm，将指定设备从白名单中移除，对应3.1中的LE Remove Device From White List Command，该命令只需要蓝牙地址作为参数，如下：

> root@android:/ # hcitool lewlrm --help  
> hcitool lewlrm --help  
> Usage:  
>         lewlrm

5）hcitool lecc，连接BLE设备的命令，对应3.2中的LE Create Connection Command，可以连接指定地址的设备，也可以直接连接白名单中的设备：

> root@android:/ # hcitool lecc --help  
> hcitool lecc --help  
> Usage:  
>         lecc [--random]  
>         lecc --whitelist

一般情况下，我们都是通过hcitool lecc 的方式连接蓝牙设备，不过如果我们需要连接白名单中的设备，可直接使用如下命令：

> hcitool lecc --whitelist

6）hcitool cmd，对于其它没有直接提供hcitool命令的HCI操作，我们可以使用hcitool cmd直接发送命令，其使用方法如下：

> root@android:/ # hcitool cmd --help  
> hcitool cmd --help  
> Usage:  
>         cmd [parameters]  
> Example:  
>         cmd 0x03 0x0013 0xAA 0x0000BBCC 0xDDEE 0xFF

其中ogf、ocf和parameters可以去蓝牙spec的“HCI COMMANDS AND EVENTS”章节查询。需要注意的是，parameters可以使用各种类型（8位、16位、32位），还是很方便的。

#### 4.3 测试步骤

这里仅仅罗列一个简单的测试，步骤包括：

> 1）设备A作为Advertising设备，不使用白名单，发送正常的ADV_IND（可连接、可扫描）广播包。
> 
> 2）设备B扫描并连接设备A（应该可以正常连接）。
> 
> 3）设备A作为Advertising设备，启用白名单，设置Advertising_Filter_Policy为0x2(只允许白名单中的设备连接)，且没有把B的地址添加到白名单中。
> 
> 4）设备B扫描并连接设备A（应该不可以正常连接）。
> 
> 5）设备A把设备B添加到白名单中，其它策略保持不变。
> 
> 6）设备B扫描并连接设备A（应该可以正常连接）。

详细步骤如下（我没有测试，有问题的请大家留言告诉我）：  
1）设备A作为Advertising设备，不使用白名单，发送正常的ADV_IND（可连接、可扫描）广播包。

> # disable BLE advertising  
> hcitool cmd 0x08 0x000A 0x00  
>   
> # 设置广播参数和广播策略  
> # Advertising_Interval_Min=0x0800 (1.28 second, default)  
> # Advertising_Interval_Max=0x0800 (1.28 second, default)  
> # Advertising_Type=0x00(ADV_IND, default)  
> # Own_Address_Type=0x00(Public Device Address, default)  
> # Peer_Address_Type=0x00(Public Device Address, default)  
> # Peer_Address=00 00 00 00 00 00 (no use)  
> # Advertising_Channel_Map=0x07(all channels enabled, Default)  
> # Advertising_Filter_Policy=0x0(禁用白名单)  
> hcitool -i hci0 cmd 0x08 0x0006 0x0800 0x0800 0x00 0x00 0x00 00 00 00 00 00 00 0x07 0x00  
>   
> # enable BLE advertising  
> hcitool cmd 0x08 0x000A 0x01
> 
> # set advertising data to Eddystone UUID(可参考[3]中的介绍)  
> hcitool -i hci0 cmd 0x08 0x0008 1e 02 01 06 03 03 aa fe 17 16 aa fe 00 -10 00 01 02 03 04 05 06 07 08 09 0a 0b 0e 0f 00 00 00 00

2）设备B扫描并连接设备A（应该可以正常连接）。

> hcitool lescan  
> hcitool lecc [bdaddr of A]

3）设备A作为Advertising设备，启用白名单，设置Advertising_Filter_Policy为0x2(只允许白名单中的设备连接)，且没有把B的地址添加到白名单中。

> # disable BLE advertising  
> hcitool cmd 0x08 0x000A 0x00  
>   
> # 设置广播参数和广播策略  
> # …  
> # Advertising_Filter_Policy=0x2(只允许白名单中的设备连接)  
> hcitool -i hci0 cmd 0x08 0x0006 0x0800 0x0800 0x00 0x00 0x00 00 00 00 00 00 00 0x07 0x02
> 
> # 清空白名单  
> hcitool lewlclr  
>   
> # 随便加一个地址到白名单  
> hcitool lewladd 11:22:33:44:55:66
> 
>   
> # enable BLE advertising  
> hcitool cmd 0x08 0x000A 0x01
> 
> # set advertising data to Eddystone UUID(可参考[3]中的介绍)  
> hcitool -i hci0 cmd 0x08 0x0008 1e 02 01 06 03 03 aa fe 17 16 aa fe 00 -10 00 01 02 03 04 05 06 07 08 09 0a 0b 0e 0f 00 00 00 00  

4）设备B扫描并连接设备A（应该不可以正常连接）。

> hcitool lescan  
> hcitool lecc [bdaddr of A]

5）设备A把设备B添加到白名单中，其它策略保持不变。

> # disable BLE advertising  
> hcitool cmd 0x08 0x000A 0x00  
>   
> # 设置广播参数和广播策略  
> # …  
> # Advertising_Filter_Policy=0x2(只允许白名单中的设备连接)  
> hcitool -i hci0 cmd 0x08 0x0006 0x0800 0x0800 0x00 0x00 0x00 00 00 00 00 00 00 0x07 0x02
> 
> # 将B添加到白名单中  
> hcitool lewladd [bdaddr of B]
> 
>   
> # enable BLE advertising  
> hcitool cmd 0x08 0x000A 0x01
> 
> # set advertising data to Eddystone UUID(可参考[3]中的介绍)  
> hcitool -i hci0 cmd 0x08 0x0008 1e 02 01 06 03 03 aa fe 17 16 aa fe 00 -10 00 01 02 03 04 05 06 07 08 09 0a 0b 0e 0f 00 00 00 00  

6）设备B扫描并连接设备A（应该可以正常连接）。

> hcitool lescan  
> hcitool lecc [bdaddr of A]

## 5. 参考文档

[1] [蓝牙协议分析(6)_BLE地址类型](http://www.wowotech.net/bluetooth/ble_address_type.html)

[2] [蓝牙协议分析(5)_BLE广播通信相关的技术分析](http://www.wowotech.net/bluetooth/ble_broadcast.html)

[3] [玩转BLE(1)_Eddystone beacon](http://www.wowotech.net/bluetooth/eddystone_test.html)

[4] [蓝牙协议分析(7)_BLE连接有关的技术分析](http://www.wowotech.net/bluetooth/ble_connection.html)

[5] Core_v4.2.pdf

原创文章，转发请注明出处。蜗窝科技，[www.wowotech.net](http://www.wowotech.net/bluetooth/ble_white_list.html)。

标签: [蓝牙](http://www.wowotech.net/tag/%E8%93%9D%E7%89%99) [Bluetooth](http://www.wowotech.net/tag/Bluetooth) [BLE](http://www.wowotech.net/tag/BLE) [白名单](http://www.wowotech.net/tag/%E7%99%BD%E5%90%8D%E5%8D%95) [white_list](http://www.wowotech.net/tag/white_list)

[![](http://www.wowotech.net/content/uploadfile/201605/ef3e1463542768.png)](http://www.wowotech.net/support_us.html)

« [内存初始化代码分析（一）：identity mapping和kernel image mapping](http://www.wowotech.net/memory_management/__create_page_tables_code_analysis.html) | [X-015-KERNEL-ARM generic timer driver的移植](http://www.wowotech.net/x_project/generic_timer_porting.html)»

**评论：**

**gw**  
2019-06-04 11:20

@wowo,测试步骤内设置广播参数和广播策略这一步有问题，配置不成功，应该是  
hcitool -i hci0 cmd 0x08 0x0006 08 00 08 00 00 00 00 00 00 00 00 00 00 0x07 0x02

[回复](http://www.wowotech.net/bluetooth/ble_white_list.html#comment-7453)

**gw**  
2019-04-12 12:03

为什么我测试步骤到只允许白名单中的设备连接，我随便设置了一个白名单，但是手机可以连接上，按理说不是应该连不上吗？

[回复](http://www.wowotech.net/bluetooth/ble_white_list.html#comment-7348)

**longlong**  
2018-09-28 14:06

不知道是否有同学按照wowo大神描述的测试步骤成功地完成了这个测试呢？

[回复](http://www.wowotech.net/bluetooth/ble_white_list.html#comment-6965)

**一般首席**  
2017-11-09 17:32

好文章，学习了

[回复](http://www.wowotech.net/bluetooth/ble_white_list.html#comment-6185)

**wnnwoo**  
2017-10-16 07:30

注3：Scanning时的白名单策略有点奇怪，既然是主动发起的，要白名单的意义就不大了吧？  
=》应该可以用来省电，无关的设备Controller就不用报给Host，Host就可以继续睡觉。

[回复](http://www.wowotech.net/bluetooth/ble_white_list.html#comment-6110)

**[wowo](http://www.wowotech.net/)**  
2017-10-16 16:21

@wnnwoo：是啊，能想到的也就这一点了。

[回复](http://www.wowotech.net/bluetooth/ble_white_list.html#comment-6113)

**[wangsuyu_1](http://www.wowotech.net/)**  
2017-05-02 15:31

想请教下@wowo “Advertising时的白名单策略”中的“0x03，只允许白名单中的设备扫描和连接”有测试过吗？我用2个ti的cc2541测试发现无论我怎么设置策略，广播都能够被任意设备搜索到，谢谢！

[回复](http://www.wowotech.net/bluetooth/ble_white_list.html#comment-5510)

**[wowo](http://www.wowotech.net/)**  
2017-05-03 08:53

@wangsuyu_1：广播（adv）和扫描（scan）不是一回事。  
广播是单向的，无论采用什么措施，ble设备发送的广播包，任何设备都能收到。  
扫描是双向的，scanner发送扫描请求，advertiser回应扫描应答。  
白名单策略能控制的，仅仅是是否回应扫描应答。

[回复](http://www.wowotech.net/bluetooth/ble_white_list.html#comment-5512)

**ABC**  
2016-11-16 14:23

wowo能不能介绍下 BLE 中 pairing 和 bonding 的部分，

[回复](http://www.wowotech.net/bluetooth/ble_white_list.html#comment-4878)

**[wowo](http://www.wowotech.net/)**  
2016-11-16 21:23

@ABC：后面会讲，不过进展可能不会很快。多谢关注~

[回复](http://www.wowotech.net/bluetooth/ble_white_list.html#comment-4881)

**老陆**  
2016-11-10 17:27

哇哇哇，第一时间看到wowo的新帖，何其幸运！感谢wowo的付出！:))

[回复](http://www.wowotech.net/bluetooth/ble_white_list.html#comment-4860)

**[wowo](http://www.wowotech.net/)**  
2016-11-10 17:32

@老陆：多谢鼓励～我就是把自己看spec的东西随手记下来而已，所以不用客气，大家互相学习哈～～～

[回复](http://www.wowotech.net/bluetooth/ble_white_list.html#comment-4861)

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
    
    - [實作 spinlock on raspberry pi 2](http://www.wowotech.net/231.html)
    - [CFS调度器（4）-PELT(per entity load tracking)](http://www.wowotech.net/process_management/450.html)
    - [中断上下文中调度会怎样？](http://www.wowotech.net/process_management/schedule-in-interrupt.html)
    - [文件缓存回写简述](http://www.wowotech.net/memory_management/327.html)
    - [Linux CPU core的电源管理(1)_概述](http://www.wowotech.net/pm_subsystem/cpu_core_pm_overview.html)
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