作者：[wowo](http://www.wowotech.net/author/2 "runangaozhong@163.com") 发布于：2017-9-7 19:49 分类：[蓝牙](http://www.wowotech.net/sort/bluetooth)

## 1. 前言

注1：此SM是Security Manager的缩写，非彼SM，大家不要理解歪了！

书接上文，我们在“[蓝牙协议分析(10)\_BLE安全机制之LE Encryption](http://www.wowotech.net/bluetooth/le_encryption.html)”中介绍了BLE安全机制中的终极武器----数据加密。不过使用这把武器有个前提，那就是双方要共同拥有一个加密key（LTK，Long Term Key）。这个key至关重要，怎么生成、怎么由通信的双方共享，关系到加密的成败。因此蓝牙协议定义了一系列的复杂机制，用于处理和加密key有关的操作，这就是SM（Security Manager）。

另外，在加密链路建立之后，通信的双方可以在该链路上共享其它的key（例如在“[蓝牙协议分析(9)\_BLE安全机制之LL Privacy](http://www.wowotech.net/bluetooth/ble_ll_privacy.html)”中提到的IRK），SM也顺便定义了相应的规范。

## 2. Security Manager介绍

SM在蓝牙协议中的位置如下图：

[![SM_in_BLE_protocol](http://www.wowotech.net/content/uploadfile/201709/48d8dc6686ffabc4f8fd9b05ec11742020170907114954.gif "SM_in_BLE_protocol")](http://www.wowotech.net/content/uploadfile/201709/dfea88b9a77eb13a7187c785ca51566920170907114954.gif)

图片1 SM_in_BLE_protocol

它的主要目的是为LE设备（LE only或者BR/EDR/LE）提供建立加密连接所需的key（STK or LTK）。为了达到这个目的，它定义了如下几类规范：

> 1）将生成加密key的过程称为Pairing（配对），并详细定义了Pairing的概念、操作步骤、实现细节等。
>
> 2）定义一个密码工具箱（Cryptographic Toolbox），其中包含了配对、加密等过程中所需的各种加密算法。
>
> 3）定义一个协议（Security Manager Protocol，简称SMP），基于L2CAP连接，实现master和slave之间的配对、密码传输等操作。

## 3. Pairing（配对）

在SM的规范中，配对是指“Master和Slave通过协商确立用于加（解）密的key的过程”，主要由三个阶段组成：

[![BLE_Pairing_Phases](http://www.wowotech.net/content/uploadfile/201709/c7f44b291f09092fe43cb6d4f291efe520170907114955.gif "BLE_Pairing_Phases")](http://www.wowotech.net/content/uploadfile/201709/7c14f6017a58c1091d818827d0100b2620170907114955.gif)

图片2 LE Pairing Phases

阶段1，称作“Pairing Feature Exchange”，用于交换双方有关鉴权的需求（authentication requirements），以及双方具有怎么的人机交互能力（IO capabilities）。

阶段2，通过SMP协议进行实际的配对操作，根据阶段1 “Feature Exchange”的结果，有两种配对方法可选：LE legacy pairing和LE Secure Connections。

阶段3是可选的，经过阶段1和阶段2之后，双方已经产生了加密key，因而可以建立加密的连接。加密连接建立后，可以互相传送一些私密的信息，例如Encryption Information、Identity Information、Identity Address Information等。

#### 3.1 Pairing Feature Exchange

配对的过程总是以Pairing Request和Pairing Response的协议交互开始，通过这两个命令，配对的发起者（Initiator，总是Master）和配对的回应者（Responder，总是Slave）可以交换足够的信息，以决定在阶段2使用哪种配对方法、哪种鉴权方式、等等，具体包括：

**3.1.1 配对方法**

Master和Slave有两种可选的配对方法：LE legacy pairing和LE Secure Connections（具体可参考后面3.2和3.3章节的介绍）。从命名上看，前者是过去的方法，后者是新方法。选择的依据是：

> 当Master和Slave都支持LE Secure Connections（新方法）的时候，则使用LE Secure Connections。否则，使用LE legacy pairing。

**3.1.2 鉴权方式**

所谓的鉴权（Authentication），就是要保证执行某一操作的双方（或者多方，这里就是配对的双方）的身份的合法性，不能出现“上错花轿嫁对郎”的情况。那怎么保证呢？从本质上来说就是通过一些额外的信息，告诉对方：现在正在和你配对的是“我”，是那个你正要配对的“我”！说起来挺饶舌，没关系，看看下面的实现方法就清楚了。

对BLE来说，主要有三类鉴权的方法（其实是两种），如下：

1）由配对的双方，在配对过程之外，额外的交互一些信息，并以这些信息为输入，进行后续的配对操作。这些额外信息也称作OOB（out of band），OOB的交互过程称为OOB protocol（是一个稍微繁琐的过程，这里不在详细介绍了）。

2）让“人（human）”参与进来，例如：

> 手机A向手机B发起配对操作的时候，手机A在界面上显示一串6位数字的配对码，并将该配对码发送给手机B，手机B在界面上显示同样的配对码，并要求用户确认A和B显示的配对码是否相同，如果相同，在B设备上点击配对即可配对成功（如下如所示）。
>
> [![配对码](http://www.wowotech.net/content/uploadfile/201709/08e64e522332c09c82504d45fbeb7f2020170907114956.gif "配对码")](http://www.wowotech.net/content/uploadfile/201709/e79b28a3fa0f266f0c175e8b483f973e20170907114956.gif)
>
> 图片3 配对码

这种需要人参与的鉴权方式，在蓝牙协议里面称作MITM（man-in-the-middle）authentication，不过由于BLE设备的形态千差万别，硬件配置也各不相同，有些可以输入可以显示、有些只可输入不可显示、有些只可显示不可输入、有些即可输入也可显示，因此无法使用统一的方式进行MITM鉴权（例如没有显示的设备无法使用上面例子的方式进行鉴权）。为此Security Manager定义了多种交互方法：

2a）Passkey Entry，通过输入配对码的方式鉴权，有两种操作方法

> 用户在两个设备上输入相同的6个数字（要求两个设备都有数字输入的能力），接下来的配对过程会进行相应的校验；
>
> 一个设备（A）随机生成并显示6个数字（要求该设备有显示能力），用户记下这个数字，并在另一个设备（B）上输入。设备B在输入的同时，会通过SMP协议将输入的数字同步的传输给设备A，设备A会校验数字是否正确，以达到鉴权的目的。

2b）Numeric Comparison，两个设备自行协商生成6个数字，并显示出来（要求两个设备具有显示能力），用户比较后进行确认（一致，或者不一致，要求设备有简单的yes or no的确认能力）。

2c）Just Work，不需要用户参与，两个设备自行协商。

3）不需要鉴权，和2c中的Just work的性质一样。

**3.1.3 IO Capabilities**

由3.1.2的介绍可知，Security Manager抽象出来了三种MITM类型的鉴权方法，这三种方法是根据两个设备的IO能力，在“Pairing Feature Exchange”阶段自动选择的。IO的能力可以归纳为如下的六种：

> NoInputNoOutput\
> DisplayOnly\
> NoInputNoOutput1\
> DisplayYesNo\
> KeyboardOnly\
> KeyboardDisplay

具体可参考BT SPEC\[3\] “BLUETOOTH SPECIFICATION Version 4.2 \[Vol 3, Part H\]” “2.3.2 IO Capabilities”中的介绍。

**3.1.4 鉴权方法的选择**

在“Pairing Feature Exchange”阶段，配对的双方以下面的原则选择鉴权方法：

> 1）如果双方都支持OOB鉴权，则选择该方式（优先级最高）。
>
> 2）否则，如果双方都支持MITM鉴权，则根据双方的IO Capabilities（并结合具体的配对方法），选择合适的鉴权方式（具体可参考BT SPEC\[3\] “BLUETOOTH SPECIFICATION Version 4.2 \[Vol 3, Part H\]”“2.3.5.1 Selecting Key Generation Method”中的介绍）。
>
> 3）否则，使用Just work的方式（不再鉴权）。

#### 3.2 LE legacy pairing

LE legacy pairing的过程比较简单，总结如下（可以参考下面的流程以辅助理解）：

[![LE_legacy_pairing](http://www.wowotech.net/content/uploadfile/201709/064a02c5fe5bc8112f3debba73d9dc0020170907114957.gif "LE_legacy_pairing")](http://www.wowotech.net/content/uploadfile/201709/2816906c8f3e088a06ec62f8a8bbfd2920170907114956.gif)

图片4 LE legacy pairing过程

1）LE legacy pairing最终生成的是Short Term Key（双方共享），生成STK之后，参考\[1\]中的介绍，用STK充当LTK，并将EDIV和Rand设置为0，去建立加密连接。

2）加密连接建立之后，双方可以自行生成Long Term Key（以及相应的EDIV和Rand），并通过后续的“Transport Specific Key Distribution”将它们共享给对方，以便后面重新建立加密连接所使用：

> master和slave都要生成各自的LTK/EDIV/Rand组合，并共享给对方。因为加密链路的发起者需要知道对方的LTK/EDIV/Rand组合，而Master或者Slave都有可能重新发起连接。
>
> 另外我们可以思考一个问题（在\[1\]中就应该有这个疑问）：为什么LE legacy pairing的LTK需要EDIV/Rand信息呢？因为LTK是各自生成的，不一样，因而需要一个索引去查找某一个LTK（对比后面介绍的LE Secure Connections，LTK是直接在配对是生成的，因而就不需要这两个东西）。

3）STK的生成也比较简单，双方各提供一个随机数（MRand和SRand），并以TK为密码，执行S1加密算法即可。

4）TK是实在鉴权的过程中得到的，根据在阶段一选择的鉴权方法，TK可以是通过OOB得到，也可以是通过Passkey Entry得到，也可以是0（Just Work）。

> LE legacy pairing只能使用OOB、Passkey Entry或者Just Work三种鉴权方法（Numeric Comparison只有在LE Secure Connections时才会使用）。

#### 3.3 LE Secure Connections Pairing

LE Secure Connections pairing利用了椭圆曲线加密算法（P-256 elliptic curve），简单说明如下（具体细节可参考看蓝牙SPEC\[3\]，就不在这里罗列了）：

1）可以使用OOB、Passkey Entry、Just Work以及Numeric Comparison四种鉴权方法。其中Numeric Comparison的流程和Just Work基本一样。

2）可以直接生成LTK（双方共享），然后直接使用LTK进行后续的链路加密，以及重新连接时的加密。

#### 3.4 Transport Specific Key Distribution

加密链路建立之后，通信的双方就可以传输一些比较私密的信息，主要包括：

> Encryption Information (Long Term Key)\
> Master Identification (EDIV, Rand)\
> Identity Information (Identity Resolving Key)\
> Identity Address Information (AddrType, BD_ADDR)\
> Signing Information (Signature Key)

至于这些私密信息要怎么使用，就不在本文的讨论范围了，后续碰到的时候再介绍。

## 4. Security Manager Protocol介绍

SMP使用固定的L2CAP channel（CID为0x0006）传输Security相关的命令。它主要从如下的方面定义SM的行为（比较简单，不再详细介绍）：

1）规定L2CAP channel的特性，MTU、QoS等。

2）规定SM命令格式。

3）定义配对相关的命令，包括：

> Pairing Request\
> Pairing Response\
> Pairing Confirm\
> Pairing Random\
> Pairing Failed\
> Pairing Public Key\
> Pairing DHKey Check\
> Keypress Notification

4）定义鉴权、配对、密码交互等各个过程。

## 5. 密码工具箱介绍

为了执行鉴权、配对等过程，SM定义了一组密码工具箱，提供了十八般加密算法，由于太专业，就不在这里介绍了，感兴趣的读者直接去看spec就行了（“BLUETOOTH SPECIFICATION Version 4.2 \[Vol 3, Part H\] 2.2 CRYPTOGRAPHIC TOOLBOX”）。

## 6. Security Manager的使用

相信经过本文的介绍，大家对BLE的SM有了一定的了解，不过应该会有一个疑问：

> 这么复杂的过程，从应用角度该怎么使用呢？

放心，蓝牙协议不会给我们提供这么简陋的接口的，参考上面图片1，SM之上不是还有GAP吗？对了，真正使用SM功能之前，需要再经过GAP进行一次封装，具体可参考本站后续的文章。

## 7. 参考文档

\[1\] [蓝牙协议分析(10)\_BLE安全机制之LE Encryption](http://www.wowotech.net/bluetooth/le_encryption.html)

\[2\] [蓝牙协议分析(9)\_BLE安全机制之LL Privacy](http://www.wowotech.net/bluetooth/ble_ll_privacy.html)

\[3\] Core_v4.2.pdf

原创文章，转发请注明出处。蜗窝科技，[www.wowotech.net](http://www.wowotech.net/bluetooth/le_security_manager.html)。

标签: [蓝牙](http://www.wowotech.net/tag/%E8%93%9D%E7%89%99) [Bluetooth](http://www.wowotech.net/tag/Bluetooth) [BLE](http://www.wowotech.net/tag/BLE) [SMP](http://www.wowotech.net/tag/SMP) [配对](http://www.wowotech.net/tag/%E9%85%8D%E5%AF%B9) [pairing](http://www.wowotech.net/tag/pairing) [鉴权](http://www.wowotech.net/tag/%E9%89%B4%E6%9D%83) [authentication](http://www.wowotech.net/tag/authentication) [security](http://www.wowotech.net/tag/security)

______________________________________________________________________

« [X-025-KERNEL-Linux gpio driver的移植之基本功能](http://www.wowotech.net/x_project/kernel_gpio_driver_porting_1.html) | [Linux reset framework](http://www.wowotech.net/pm_subsystem/reset_framework.html)»

**评论：**

**strong**\
2021-12-09 10:14

上边的配对图 明明用的passkey entry，为什么又说OOB方式， 此图有错误

[回复](http://www.wowotech.net/bluetooth/le_security_manager.html#comment-8399)

**xiaoxie**\
2018-09-10 17:11

请教下wowo，OOB data能举个例子吗？比如需要app从云端获取一个secret key，这个key就是OOB data吗？

[回复](http://www.wowotech.net/bluetooth/le_security_manager.html#comment-6941)

**[w0w0](http://https//evilpan.com)**\
2020-06-03 17:53

@xiaoxie：OOB一般是指蓝牙之外的其他物理信道来交换配对所需要的信息，这个信道同时也用来发现设备，比如NFC。从云端来交换信息广义来说也算OOB，信道是802.11的WiFi，不过似乎没有这样用的。

[回复](http://www.wowotech.net/bluetooth/le_security_manager.html#comment-8011)

**[wowo](http://www.wowotech.net/)**\
2018-04-04 15:27

@EE，其实我觉得没必要把配对和角色转换这两个过程纠结在一起，这是两个很独立的过程。\
重新配对了，肯定会更新LTK；\
角色转换了，如果要重新配对，也会更新LTK；\
至于IRK，是用于地址解析的，需要在重新配对前，广播的地址需要解析，就需要提前把IRK告诉对方；\
把这几个信息揉在一起，好好想想，就是你所要的答案了~

[回复](http://www.wowotech.net/bluetooth/le_security_manager.html#comment-6647)

**EE**\
2018-04-06 11:08

@wowo：我感觉没有纠结在一起呀，重新配对肯定是会更新LTK，问题是在一次配对中密钥交换主从可以互相发送，两个LTK如何使用是这个问题，因为LTK在当前总是只使用一个不会同时使用两个，协议中既然提到了角色转换那肯定是有它的意义！！！

[回复](http://www.wowotech.net/bluetooth/le_security_manager.html#comment-6653)

**方言**\
2017-11-28 16:10

Wowo,文中提到“另外我们可以思考一个问题（在\[1\]中就应该有这个疑问）：为什么LE legacy pairing的LTK需要EDIV/Rand信息呢？因为LTK是各自生成的，不一样，因而需要一个索引去查找某一个LTK（对比后面介绍的LE Secure Connections，LTK是直接在配对是生成的，因而就不需要这两个东西）。”，说到LTK不一样。但最终数据加解密的时候，使用的应该是对称的方式，所以LTK是相等才对吧？而且整个过程中，应该是各自计算出来的，并没有空中交换，不知道我说的是否正确~

[回复](http://www.wowotech.net/bluetooth/le_security_manager.html#comment-6251)

**[wowo](http://www.wowotech.net/)**\
2017-11-29 09:25

@方言：具体某一次加密和解密，使用的LTK是一样的，但到底用的是哪一个LTK，LE legacy pairing有关的过程是不固定的。因为LTK是配对后通过STK自行生成的，所以肯定需要告诉对方，并且可能不止有一个，所以大家都需要一个表，存储不同的STK组合（EDIV/Rand就是这个表的index）。

[回复](http://www.wowotech.net/bluetooth/le_security_manager.html#comment-6256)

**[chen_chuang](http://www.wowotech.net/)**\
2017-10-23 14:15

wowo大神，何时讲讲mesh相关的知识点，不知道实际上一个蓝牙能组网多少个低功耗节点

[回复](http://www.wowotech.net/bluetooth/le_security_manager.html#comment-6135)

**[wowo](http://www.wowotech.net/)**\
2017-10-24 09:18

@chen_chuang：mesh的标准刚出来，只是看过几遍spec，还没有用过呢，所以不知道什么时候会写啊。 至于组网的节点个数，简单的讲：无数多个，因为mesh的组网简单的说就是“接收+转发”。 不过呢，最终多少个，受限于实际应用场景的需求：比如你能容忍的延迟；比如你需要的传输速率；等等。

[回复](http://www.wowotech.net/bluetooth/le_security_manager.html#comment-6139)

**dong**\
2017-09-07 23:30

“master和slave都要生成各自的LTK/EDIV/Rand组合，并共享给对方。因为加密链路的发起者需要知道对方的LTK/EDIV/Rand组合，而Master或者Slave都有可能重新发起连接。”

我在用BLE4.0的时候，好像作为Slave时候并不能主动发起START ENCRYPTION啊，Slave调用加密函数时候，用抓包软件发现Slave发出的是一个Security request，然后之前已经与之绑定过的Master收到这个Security request之后，主动发起START ENCRYPTION开始加密，所以我还是不太明白这个角色反转发起连接是怎么一回事，就是Slave如何发起START ENCRYPTION，求wowo指导指导。

[回复](http://www.wowotech.net/bluetooth/le_security_manager.html#comment-6004)

**[wowo](http://www.wowotech.net/)**\
2017-09-08 09:12

@dong：所谓的角色反转就是：前一次A是master，B是slave，下一次重连的时候，可以B是master A是slave啊。谁发起这次连接谁就是master啊（不要把连接和加密连接混在一起了，二者没啥关系）。

[回复](http://www.wowotech.net/bluetooth/le_security_manager.html#comment-6005)

**dong**\
2017-09-08 23:45

@wowo：wowo有道理，后来我想了想，这里角色调转，是指结束之前的piconet，换角色重新建立新piconet，形成“首次建立piconet就拥有slave的ltk信息”现象，直接发起加密，跳过配对过程。

[回复](http://www.wowotech.net/bluetooth/le_security_manager.html#comment-6009)

**[wowo](http://www.wowotech.net/)**\
2017-09-09 10:08

@dong：是的，就是这个意思～～

[回复](http://www.wowotech.net/bluetooth/le_security_manager.html#comment-6010)

**EE**\
2018-03-29 10:48

@wowo：wowo，来来来，帮个忙：）关于密钥分发中的the roles are reversed一段：\
The master may also provide keys to the slave device so a reconnection can be encrypted if the roles are reversed, the master’s random addresses can be resolved, or the slave can verify signed data from the master.\
1、其中the roles are reversed是不是只针对LTK，没有反转前用的是从机的LTK，反转后用的主机的LTK（此时其实已是从机角色）？\
2、其中the roles are reversed是不是只针对LTK，对CSRK，IRK都没有意义，这两个密钥只关注是谁生成的和是谁接收的跟角色没有关系，发送时对自己的数据或地址用自己生成的密钥，接收时验证或解析用分发者给的密钥，请问是不是这样的？\
3、但对于LTK如你上文所说，具体的一次加密和解密双方都用到相同LTK，而对于CSRK，IRK，如果一方没有发给对方，那么验证和解析都是单向的？\
谢谢啦：）

[回复](http://www.wowotech.net/bluetooth/le_security_manager.html#comment-6627)

**[wowo](http://www.wowotech.net/)**\
2018-04-03 16:44

@EE：抱歉回复晚了~\
首先来说，双方使用的LTK是相同的，这是加密的基础，否则就是对牛弹琴了；\
the roles are reversed的场景里面，master需要通过加密链路（基于LTK）把一些必要的信息告诉slave（例如怎么解析master的Random address），只有这样，下次角色反转的时候，slave（此时是slave，翻转后是master）才能正确解析master（此时是master，下次是slave）的地址。

[回复](http://www.wowotech.net/bluetooth/le_security_manager.html#comment-6641)

**EE**\
2018-04-04 12:55

@wowo：通读了协议栈，这方面写的不是太直接，感觉LTK与CSRKT和IRK用法上有些不同：LTK有角色转换的概念，而CSRKT和IRK没有，对于后者不管链路角色怎么变密钥总是密钥接收者验证或解析密钥发送者，如果主机和从机都要分配那么会在链路中同时用到，而前者角色变化了链路的LTK也会换悼，因为链路在加密时只使用一个LIK有别于CSRKT和IRK可以同时使用它们双方的，wowo麻烦再帮确认下我刚这些话是否与你的看法一制，如下是协议中规定的密钥分发的顺序供参考：

When using LE legacy pairing, the keys shall be distributed in the following\
order:

1. LTK by the slave
1. EDIV and Rand by the slave
1. IRK by the slave
1. BD ADDR by the slave
1. CSRK by the slave
1. LTK by the master
1. EDIV and Rand by the master
1. IRK by the master
1. BD_ADDR by the master
1. CSRK by the master

**dong**\
2017-09-07 23:06

顶一下wowo，看了一个月BLE,终于入门了，wowo帖子帮助很大。

[回复](http://www.wowotech.net/bluetooth/le_security_manager.html#comment-6003)

**发表评论：**

昵称

邮件地址 (选填)

个人主页 (选填)

![](http://www.wowotech.net/include/lib/checkcode.php)

- ### 站内搜索

  蜗窝站内  互联网

- ### 功能

  [留言板\
  ](http://www.wowotech.net/message_board.html)[评论列表\
  ](http://www.wowotech.net/?plugin=commentlist)[支持者列表\
  ](http://www.wowotech.net/support_list)

- ### 最新评论

  - ja\
    [@dream：我看完這段也有相同的想法，引用 @dream ...](http://www.wowotech.net/kernel_synchronization/spinlock.html#8922)
  - 元神高手\
    [围观首席power managerment专家](http://www.wowotech.net/pm_subsystem/device_driver_pm.html#8921)
  - 十七\
    [内核空间的映射在系统启动时就已经设定好，并且在所有进程的页表...](http://www.wowotech.net/process_management/context-switch-arch.html#8920)
  - lw\
    [sparse模型和disconti模型没看出来有什么本质区别...](http://www.wowotech.net/memory_management/memory_model.html#8919)
  - 肥饶\
    [一个没设置好就出错](http://www.wowotech.net/linux_kenrel/516.html#8918)
  - orange\
    [点赞点赞，对linuxer的文章总结到位](http://www.wowotech.net/device_model/dt-code-file-struct-parse.html#8917)

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

  - [X-018-KERNEL-串口驱动开发之serial console](http://www.wowotech.net/x_project/serial_driver_porting_3.html)
  - [ARM64的启动过程之（一）：内核第一个脚印](http://www.wowotech.net/armv8a_arch/arm64_initialize_1.html)
  - [进化论、人工智能和外星人](http://www.wowotech.net/tech_discuss/toe_ai_et.html)
  - [SDRAM Internals](http://www.wowotech.net/basic_tech/sdram-internals.html)
  - [X-006-UBOOT-pinctrl driver移植(Bubblegum-96平台)](http://www.wowotech.net/x_project/bubblegum_uboot_pinctrl.html)

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
