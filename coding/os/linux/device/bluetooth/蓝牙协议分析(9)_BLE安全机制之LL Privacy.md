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

作者：[wowo](http://www.wowotech.net/author/2 "runangaozhong@163.com") 发布于：2016-11-25 16:28 分类：[蓝牙](http://www.wowotech.net/sort/bluetooth)

## 1. 前言

在上一篇文章[1]中，我们介绍了BLE的白名单机制，这是一种通过地址进行简单的访问控制的安全机制。同时我们也提到了，这种安全机制只防君子，不防小人，试想这样一种场景：

> A设备表示只信任B、C、D设备，因此就把它们的地址加入到了自己的白名单中，表示只愿意和它们沟通。与此同时，E设备对它们的沟通非常感兴趣，但A对自己不信任啊，肿么办？
> 
> E眼珠子一转，想出一个坏主意：把自己的地址伪装成成B、C、D中任意一个（这个还是很容易办到的，随便扫描一下就得它们的地址了）就行了，嘿嘿嘿！

那么问题来了，怎么摆脱“小人E“的偷听呢？不着急，我们还有手段：“链路层的Privacy（隐私）机制”。

## 2. LL Privacy机制介绍

总结来说，LL Privacy机制是白名单（white list）机制的进阶和加强，它在白名单的基础上，将设备地址转变成Private addresses[2]地址，以降低“小人E“窃得设备地址进而进行伪装的概率。

从白名单的角度看，Non-resolvable private address和Public Device Address/Static Device Address没有任何区别，因此LL Privacy机制主要指Resolvable Private Addresses。因此，LL Privacy机制的本质是：

> 通过Resolvable Private Addresses，将在空中传输的设备地址加密，让“小人E”无法窃得，从而增加其伪装的难度。

注1：当然，LL Privacy机制可以脱离白名单机制单独使用，不过这样的话好像没什么威力。

注2：有关Resolvable Private Addresses、Identity Address、IRK（Local/Peer IRK）等概念的详细介绍，可参考“[蓝牙协议分析(6)_BLE地址类型](http://www.wowotech.net/bluetooth/ble_address_type.html)[2]”，本文将会直接使用。

## 3. Resolving List

和白名单机制上的White List类似，如果设备需要使用LL Privacy机制，则需要在Controller端保存一个Resolving List，其思路为：

1）BLE设备要按照[1]中的介绍，配置并使能白名单机制，把那些受信任设备的地址（这里为Identity Address）加入到自己的白名单中，并采用合适的白名单策略。

2）如果设备需要使用LL Privacy策略，保护自己（以及对方）的地址不被窃取，则需要将自己（local）和对方（peer）的地址和加密key保存在一个称作Resolving List的列表中。

3）Resolving List的每一个条目，都保存了一对BLE设备的key/address信息，其格式为：Local IRK | Peer IRK | Peer Device Identity Address | Address Type。

> Local IRK，本地的IRK，用于将本地设备的Identity Address转换为Resolvable Private Address。可以为0，表示本地地址直接使用Identity Address；
> 
> Peer IRK，对端的IRK，用于将对端设备的Resolvable Private Address解析回Identity Address。可以为零，表示对端地址是Identity Address；
> 
> Peer Device Identity Address、Address Type，对端设备的Identity Address及类型，用于和解析回的Identity Address进行比对。

4）Resolving List是Host通过HCI命令提供给Controller的，Controller的LL负责如下事项：

> 发送数据包[3]时：  
> 如果有AdvA需要填充，则判断Resolving List是否有非0的Local IRK，如果有，则使用Local IRK将Identity Address加密为Resolvable Private Address，填充到AdvA中。否则，直接填充Identity Address；  
> 同理，如果有InitA需要填充，则判断Resolving List是否有匹配的、非0的Peer IRK，如果有，则使用Peer IRK将对端的Identity Address加密为Resolvable Private Address，填充到InitA中。否则，直接填充Identity Address。
> 
> 接收数据包时：  
> 如果数据包中的AdvA或者InitA为普通的Identity Address，则直接做后续的处理；  
> 如果它们为Resolvable Private Address，则会遍历Resolving List中所有的“IRK | Identity Address”条目，使用IRK解出Identity Address和条目中的对比，如果匹配，则地址解析成功，可以做进一步处理。如果不匹配，且使能了白名单/LL Privacy策略，则会直接丢弃。

5）需要重点说明的是，Controller和Host之间所有的事件交互（除了Resolving List操作之外），均使用Identity Address。也就是说，设备地址的加密、解密、比对等操作，都是在controller中完成的，对上层实体（HCI之上）是透明的。

## 4. 使用场景说明

结合上面2章的解释，罗列一下LL Privacy策略有关的使用场景（大部分翻译自蓝牙spec）。

#### 4.1 设备处于广播状态（Advertising state）时

由[3]中的介绍可知，处于广播状态的BLE设备，根据需要可发送ADV_IND、ADV_DIRECT_IND、ADV_NONCONN_IND和ADV_SCAN_IND 4种类型的广播包，也就是说有4种不同的广播状态，它们的LL privacy策略有稍微的不同，下面分别描述。

1）ADV_IND

设备（Advertiser）发送ADV_IND时，其PDU（connectable undirected advertising event PDU）有一个AdvA字段，该字段的填充策略为（由controller的LL执行）：

> 检查Resolving List，查看是否存在非0的Local IRK条目，如果有，则使用Local IRK将自己的Identity Address加密为Resolvable Private Addresses，并填充到AdvA中。否则，直接使用Identity Address。

Advertiser收到连接请求时，请求者的地址会包含在PDU的InitA中，该字段的解析策略为（由controller的LL执行）：

> 如果InitA是Resolvable Private Addresses，且当前使能了地址解析功能，LL会遍历Resolving List中所有的“Peer IRK | Peer Device Identity Address | Address Type”条目，使用Peer IRK解析出Identity Address后和Peer Device Identity Address做比对，如果匹配，则解析成功，再基于具体的白名单策略，觉得是否接受连接；
> 
> 如果解析不成功，则无法建立连接；
> 
> 如果InitA不是Resolvable Private Addresses，则走正常的连接过程。

Advertiser收到扫描请求时，对ScanA的处理策略，和InitA类似。

2）ADV_DIRECT_IND

设备（Advertiser）发送ADV_DIRECT_IND 时，其PDU（connectable directed advertising event PDU）包含AdvA和InitA两个地址，它们填充策略为（由controller的LL执行）：

> 检查Resolving List，查看是否存在非0的Local IRK条目，如果有，则使用Local IRK将自己的Identity Address加密为Resolvable Private Addresses，并填充到AdvA中。否则，直接使用Identity Address；
> 
> 检查Resolving List，查看是否存在非0、和InitA的Identity Address匹配的Peer IRK条目，如果有，则使用Peer IRK将InitA的Identity Address加密为Resolvable Private Addresses，并填充到InitA中。否则，直接使用Identity Address。

Advertiser收到连接请求时，请求者的地址会包含在PDU的InitA中，该字段的解析策略为和上面ADV_IND类似，不再详细说明。

3）ADV_NONCONN_IND and ADV_SCAN_IND

这两个状态下，AdvA的填充策略和上面1）和2）一样，不再详细说明。

当Advertiser收到SCAN请求时，对ScanA的处理策略，和1）中InitA类似，不再详细说明。

#### 4.2 设备处于扫描状态（Scanning state）时

处于Scanning状态的设备（Scanner）在接收到Advertiser发送的scannable的广播包时，需要按照4.1中解析InitA的方法，解析广播包中的AdvA，并根据当前的白名单策略，进行过滤。

Scanner发送scan请求时，需要指定ScanA和AdvA两个地址。其实ScanA的填充策略和4.1中的AdvA类似，不再详细说明。而AdvA，需要和接收到的广播包的AdvA完全一样，不能有改动。

#### 4.3 设备处于连接状态（Initiating state）时

处于Initiating状态的设备（Initiator）在接收到Advertising发送的connectable的广播包时，需要按照4.1中解析InitA的方法，解析广播包中的AdvA，并根据当前的白名单策略，进行过滤。

同理，Initiator发送连接请求时，需要指定InitA和AdvA两个地址。其实InitA的填充策略和4.1中的AdvA类似，不再详细说明。而AdvA，需要和接收到的广播包的AdvA完全一样，不能有改动。

最后，Initiator在接收到Advertising发送的directed connectable广播包时，除了要解析AdvA，如果InitA是Resolvable Private Addresses，则需要使用Local IRK解析InitA。

## 5. 和LL Privacy有关的HCI命令

不太想写了，需要了解的直接去掰spec吧。

## 6. 参考文档

[1] [蓝牙协议分析(8)_BLE安全机制之白名单](http://www.wowotech.net/bluetooth/ble_white_list.html)

[2] [蓝牙协议分析(6)_BLE地址类型](http://www.wowotech.net/bluetooth/ble_address_type.html)

[3] [蓝牙协议分析(5)_BLE广播通信相关的技术分析](http://www.wowotech.net/bluetooth/ble_broadcast.html)

原创文章，转发请注明出处。蜗窝科技，[www.wowotech.net](http://www.wowotech.net/bluetooth/ble_ll_privacy.html)。

标签: [蓝牙](http://www.wowotech.net/tag/%E8%93%9D%E7%89%99) [BLE](http://www.wowotech.net/tag/BLE) [resolvable](http://www.wowotech.net/tag/resolvable) [privacy](http://www.wowotech.net/tag/privacy) [安全](http://www.wowotech.net/tag/%E5%AE%89%E5%85%A8)

[![](http://www.wowotech.net/content/uploadfile/201605/ef3e1463542768.png)](http://www.wowotech.net/support_us.html)

« [X-019-KERNEL-串口驱动开发之数据收发](http://www.wowotech.net/x_project/serial_driver_porting_4.html) | [内存初始化代码分析（三）：创建系统内存地址映射](http://www.wowotech.net/memory_management/mem_init_3.html)»

**评论：**

**k_th111**  
2020-03-26 09:16

Hi wowo,  
    请问在设备用IRK解析RPA得到localHash后，再与hash value extracted from RPA 比较时，这里使用的提取RPA是对等设备传给它的广播包里的RPA吗？那么解析表条目中的“Peer Device Identity Address | Address Type”发挥什么作用呢？这里不理解，想请教大神们。

[回复](http://www.wowotech.net/bluetooth/ble_ll_privacy.html#comment-7929)

**chcharles**  
2018-04-03 14:11

你好，涉及到利用IRK和rand生成hash比对这块，如何找到对端的IRK（比如利用index找到对应的key？），我有些疑问：  
文中提到：  
检查Resolving List，查看是否存在非0、和InitA的Identity Address匹配的Peer IRK条目。  
  
这个里的identity address是list里保存的那个peer Device Identity Address吗，这个address到底是哪种地址，BLE那几种中的哪类？  
另外，如果是Resolvable Private Address，它应该是由hash，random，和type组成，并没有peer device identity address信息，所以如何确定这个Resolvable Private Address对应的IRK呢？  
  
比较疑惑，请指教，谢谢！

[回复](http://www.wowotech.net/bluetooth/ble_ll_privacy.html#comment-6640)

**[wowo](http://www.wowotech.net/)**  
2018-04-03 16:53

@chcharles：可参考这篇文章理解地址类型：http://www.wowotech.net/bluetooth/ble_address_type.html

[回复](http://www.wowotech.net/bluetooth/ble_ll_privacy.html#comment-6642)

**chcharles**  
2018-04-04 09:56

@wowo：感谢回复。  
  
我仔细看过才请教，蓝牙地址这块我又看了一遍：  
当对端BLE设备扫描到该类型的蓝牙地址后，会使用保存在本机的IRK，和该地址中的prand，进行同样的hash运算，并将运算结果和地址中的hash字段比较，相同的时候，才进行后续的操作。这个过程称作resolve（解析），这也是Non-resolvable private address/Resolvable private address命名的由来。  
  
我的问题不是如何验证hash，我的问题是如何找到这个地址对应的IRK，因为现在我感觉没有类似Index的参数可以找到对应的IRK啊？

[回复](http://www.wowotech.net/bluetooth/ble_ll_privacy.html#comment-6644)

**[wowo](http://www.wowotech.net/)**  
2018-04-04 15:20

@chcharles：这个不是存在Resolving List中吗？：  
Resolving List的每一个条目，都保存了一对BLE设备的key/address信息，其格式为：Local IRK | Peer IRK | Peer Device Identity Address | Address Type。

[回复](http://www.wowotech.net/bluetooth/ble_ll_privacy.html#comment-6646)

**chcharles**  
2018-04-04 17:42

@wowo：是存在list，我理解这个resolving list会存多个设备的key/address信息吧……  
怎么找到对方的IRK啊？？？  
哪什么做索引？  
  
这个问题的关键在于Resolvable private address发过来，这个是个可变量，对吧，hash，random都是可变的，我接收到，怎么识别这个Resolvable private address在resolving list里对应的是哪个IRK啊？  
  
就是这个搞不懂

[回复](http://www.wowotech.net/bluetooth/ble_ll_privacy.html#comment-6648)

**[wowo](http://www.wowotech.net/)**  
2018-04-04 18:20

@chcharles：不需要知道啊。解析者只需要用自己保存的所有的remote IRK一一进行hash运算及比较，如果匹配，就是解析成功了。

[回复](http://www.wowotech.net/bluetooth/ble_ll_privacy.html#comment-6649)

**chcharles**  
2018-04-05 16:33

@wowo：哦 你的意思是穷举了……就是都遍历一遍，看哪个一样？  
这样计算效率太低了吧

**chcharles**  
2018-04-05 16:35

@wowo：另外，我想问一下这个Peer Device Identity Address是什么地址，不属于地址类型中的那几个分类啊……  
这里保存它是干什么用呢？谢谢！  
问题比较多，不好意思

**[wowo](http://www.wowotech.net/)**  
2018-01-01 21:58

@EE，多谢指正，其实这个话题已经在上面和zhay兄讨论过了，我一直打算重新整理、改正这篇文章呢，不过一直拖着没做:-(

[回复](http://www.wowotech.net/bluetooth/ble_ll_privacy.html#comment-6440)

**EE**  
2018-01-02 21:51

@wowo：不好意思，我没有认真看留言日期，以为是zhay兄讨论之后才发出你的总结，实则你是在那之前发出的，我只顾看文字顺序没顾及时间顺序了，不好意思啊：-）

[回复](http://www.wowotech.net/bluetooth/ble_ll_privacy.html#comment-6448)

**pang**  
2017-06-20 12:49

你好，  
假设一个device A作为从设备， IOS手机作为主设备，IOS手机用的是resolvable address，在他们绑定之前，那么device A 怎么知道或者获取手机的IRK呢

[回复](http://www.wowotech.net/bluetooth/ble_ll_privacy.html#comment-5711)

**zhay**  
2017-06-07 16:17

Hi wowo:  
  
我对你文中“接收数据时，我们用Peer IRK解析出Identity Address后和Peer Device Identity Address做比对，如果匹配，则解析成功”这段话的理解有异议，请问如何用Peer IRK 解析出Identity Address?  
因为你在《蓝牙协议分析(6)_BLE地址类型》一文中写到：Resolvable private address 是由IRK和Identity address进行hash运算得到的，而hash运算是不可逆的，见百度。即接收数据时，我们知道peer Resolvable private address, 是无法使用Peer IRK解析出Identity Address的。办法是用Resolving List中的peer IRK和prand也进行hash运算，再将hash运算结果和Resolvable private address中的hash段进行比较，如果匹配，则解析成功。你在《蓝牙协议分析(6)_BLE地址类型》中也是这么说的。  
所以我认为用Peer IRK解析出Identity Address是错误的说法，无法解析。且接收数据时，我们只用Peer IDK进行hash运算即可，跟peer device identity address没什么关系。而peer device identity address的应用场景只有发送数据包时，有InitA需要填充的情况下，即发送ADV_DIRECT_IND广播时加密对端设备地址时用。  
以上是我的理解，如有不对之处还请校正，THX！

[回复](http://www.wowotech.net/bluetooth/ble_ll_privacy.html#comment-5634)

**[wowo](http://www.wowotech.net/)**  
2017-06-08 08:57

@zhay：Hi zhay，  
多谢指正，你是对的，原谅我的不细致以及想当然吧！～～  
确实，Private Device Address Resolution的过程，是不需要“Peer Device Identity Address”参与的，只需要对比hash即可。  
Resolving List中之所以要保存“Peer Device Identity Address”，是为了和White List中做比对：在Resolution的过程中确定一个Resolving List中的条目，得到“Peer Device Identity Address”，并和White List中的条目比较。  
再次感谢指正，有空我修改一下文章。  
谢谢！！

[回复](http://www.wowotech.net/bluetooth/ble_ll_privacy.html#comment-5637)

**zhay**  
2017-06-08 17:50

@wowo：@wowo,多谢回复。感觉这句话也有点问题：“使用Local IRK将Identity Address加密为Resolvable Private Address”，据spec,Resolvable Private Address是由IRK和随机数prand生成的，所以我的理解是Resolvable Private Address的生成和解析只于IRK有关.  
我上段话中peer device identity address的应用场景有误，如你所说，是为了和白名单作对比。BLE设备在发起Advertising、Scanning或者Connecting操作的时候，都可以设置白名单策略。

[回复](http://www.wowotech.net/bluetooth/ble_ll_privacy.html#comment-5644)

**[wowo](http://www.wowotech.net/)**  
2017-06-09 08:34

@zhay：是的，Identity Address和Resolvable Private Address没有任何关系。

[回复](http://www.wowotech.net/bluetooth/ble_ll_privacy.html#comment-5645)

**dong**  
2017-09-03 19:59

@wowo：identity address应该是peer device的static address或者public address吧，每个device必须要有其中1个。假如两个device都使用privacy功能，就算双方在广播、连接之前都已将对方的IRK固化在程序里，可是整个广播连接过程也无法知道对方的identity address吧（只能知道双方的resolvable/non-resolvable address)，在我看来只有在pairing的phase 3阶段时候交换双方的identity address information时候才能让双方都拥有对方的identity address，求wowo指导啊。

[回复](http://www.wowotech.net/bluetooth/ble_ll_privacy.html#comment-5977)

**[wowo](http://www.wowotech.net/)**  
2017-09-04 10:17

@dong：你总结的很对啊，呵呵～～  
LL privacy是比较简单的安全机制，如果两个设备打算“更深一步的交流”，使用配对、绑定等更强大的安全机制，则本文所涉及的内容就out了。这时双方就应该坦诚相见，把自己的identity address告知对方。

[回复](http://www.wowotech.net/bluetooth/ble_ll_privacy.html#comment-5978)

**EE**  
2018-01-02 21:44

@zhay：“而peer device identity address的应用场景只有发送数据包时，有InitA需要填充的情况下，即发送ADV_DIRECT_IND广播时加密对端设备地址时用。”这句也是有错误的，InitA是否填充为peer device identity address，还要看Peer IRK 是否非0

[回复](http://www.wowotech.net/bluetooth/ble_ll_privacy.html#comment-6447)

**[wowo](http://www.wowotech.net/)**  
2018-01-04 08:48

@EE：zhay同学只是说initA需要填充peer device identity的时候，才会用到peer device identity（意思是peer device identity只在这个场景使用），所以也不是错的。关注的点不一样。

[回复](http://www.wowotech.net/bluetooth/ble_ll_privacy.html#comment-6450)

**xiongbiao.wu**  
2021-07-17 12:01

@zhay：是的，你的理解非常正确。client扫描到RPA之后的具体的计算方法过程如下：  
  
1.client扫描到RPA恰好与IPR链表中的某一项之前保存的RPA相同(则表示server的RPA地址没有变化)，直接将使用IRK结构体中的Identity Address进行白名单/黑名单比较，上报扫描结果等。  
  
client接收到RPA(address1)与保存的IRK链表中的每一个旧的RPA都不相同，此时遍历IRK链表，使用ah(irk-val， address1[3])算法，irk-val表示保存在IRK结构体中的经过第一次SMP之后拿到的对面irk值，address1[3]表示RPA地址的低三字节，进行ah运算得到一个hash值，用这个hash值和address1[0-2](RPA地址的高三字节)做比较，如果相同，则表示新扫描到的这个RPA地址设备就是首次经过SMP过程已绑定的设备。之后使用IRK结构体中的Identity Address进行白名单/黑名单比较，上报扫描结果等操作。  
  
还有，通过BlueZ代码发现，地址解析比对这些操作我认为是在host(SMP)端中完成的，并不是在controller中。

[回复](http://www.wowotech.net/bluetooth/ble_ll_privacy.html#comment-8253)

**panda**  
2017-04-07 15:13

Hi wowo,  
LL Privacy机制是白名单（white list）机制的进阶和加强。LL Privacy机制可以脱离白名单机制单独使用，不过这样的话好像没什么威力。  
---------------------------------------------------------------------------------  
这有点矛盾。LL Privacy机制是白名单的升级版本，完全覆盖了白名单的功能。所以单独使用LL Privacy机制是很自然的选择。是这样理解吗？

[回复](http://www.wowotech.net/bluetooth/ble_ll_privacy.html#comment-5431)

**[wowo](http://www.wowotech.net/)**  
2017-04-10 09:13

@panda：这里的意思是：  
LL Privacy是通过加密蓝牙地址的手段保证通信安全；  
而白名单功能是通过过滤不信任的地址，保证通信安全；  
从本质上讲，它们两个没有关系；  
但是，如果不过滤非法地址，加密自己的地址又有什么意义呢？

[回复](http://www.wowotech.net/bluetooth/ble_ll_privacy.html#comment-5437)

**panda**  
2017-04-10 09:27

@wowo：如果蓝牙地址解析不成功，则无法建立连接；这就起到了过滤的作用了。  
如果再增加白名单，相当于double check。

[回复](http://www.wowotech.net/bluetooth/ble_ll_privacy.html#comment-5440)

**[wowo](http://www.wowotech.net/)**  
2017-04-10 16:15

@panda：如果“地址解析不成功“，就“……（无法建立连接）”，后面这一段省略号，正是白名单的策略。  
也就是说，当地址解析不成功的时候，要怎么处理，正是白名单的一部分。所以无论如何，白名单都在的，呵呵～～

[回复](http://www.wowotech.net/bluetooth/ble_ll_privacy.html#comment-5444)

**panda**  
2017-04-10 19:24

@wowo：我再查看了spec（BLUETOOTH SPECIFICATION Version 4.2 [Vol 6, Part B] 4.7 RESOLVING LIST）  
The Identity Address may be in the White List.  
  
我的理解是：resolving List中的Peer_Identity_Address_Type/  
Peer_Identity_Address可以占用一条White List，也可以不占用White List，而在resolving List中单独实现类似的WhiteList。  
这可能和linklayer的具体实现相关了。  
  
我现在用的ble芯片，一条White List占用8个bytes，一条resolving List占用68Bytes。WhiteList和Resolving List应该就是独立实现的。  
  
两种说法都能成立，呵呵

[回复](http://www.wowotech.net/bluetooth/ble_ll_privacy.html#comment-5448)

**[wowo](http://www.wowotech.net/)**  
2017-04-11 13:56

@panda：抱歉，我上面的回复有些问题，我们慢慢的再梳理一下：  
1. 本设备的resolve list中，保存了一些这样的条目，Local IRK | Peer IRK | Peer Device Identity Address | Address Type  
2. 本设备收到其它设备的广播信息时，如果发现peer地址为resolvable address，则尝试遍历自己的resolve list，使用IRK解析resolvable address并和Peer Device Identity Address比较。  
3. 如果resolvable address解析成功：  
    a）如果没有使能白名单机制，则进行后续的操作。  
    b）如果使能了白名单机制，则根据具体的白名单策略（此时对比的是解析后的Peer Device Identity Address），决定后续的动作。  
4. 如果resolvable address解析不成功：  
    a）拒绝后续的操作（此时不依赖白名单机制）。  
  
回到我们这个话题上：  
LL Privacy确实可以单独存在，但二者的功能不完全相同。  
白名单的思路是：只有我认识的（放在了白名单中）才是好人，我才愿意和你玩。  
LL Privacy的思路是：只有能和我对上暗号的才是好人，我才愿意和你玩。  
二者加在一起的思路是：只有我认识的并且能和我对上暗号的，我才愿意和你玩。潜在的思路是什么，可以在实际的应用场景中挖掘。

[回复](http://www.wowotech.net/bluetooth/ble_ll_privacy.html#comment-5452)

**panda**  
2017-04-11 15:33

@wowo：梳理的很清楚，赞~  
  
使用IRK解析resolvable address并和Peer Device Identity Address比较。  
~~~~·这里已经是白名单的思路，所以privacy的思路包含了白名单的思路。LL Privacy的思路已经是“能和我对上暗号并且是我认识的，我才愿意和你玩。  
”

**EE**  
2017-12-29 12:13

@wowo：WOWO,我觉得上中2部分有些问题：“使用IRK解析resolvable address并和Peer Device Identity Address比较。”这句是有些问题，其中“使用IRK解析resolvable address”没有错，但“并和Peer Device Identity Address比较。”这个好似不对！用IRK解析resolvable address得到的是localHash，再与hash value extracted from RPA 比较，相同则表示解析成功，此时是根据这个IRK对应的条目（Local IRK | Peer IRK | Peer Device Identity Address | Address Type）来确认Peer Device Identity Address | Address Type，再根据这个地址进入白名单过滤的,spec原话：“The localHash value is then compared with the  hash value extracted from RPA .If the  localHash value matches the extracted  hash value, then the identity of the peer device has been resolved.”

**chong**  
2017-03-15 17:06

写的真好，看英文版的spec很是烧脑，看了您写的这些关于蓝牙的知识，感觉学到了很多呢，真心感谢

[回复](http://www.wowotech.net/bluetooth/ble_ll_privacy.html#comment-5320)

**miranda**  
2017-01-20 15:20

wowo好棒  
把死板的spec讲解的这么细致又通俗易懂,期待讲解ble的pairing和bonding~

[回复](http://www.wowotech.net/bluetooth/ble_ll_privacy.html#comment-5178)

**AndyPeng**  
2016-12-06 14:15

hi,wowo同志，请问蓝牙的安全机制怎么实现，即便监听者在配对开始时就监听也是无法解密加密数据？因为用蓝牙分析仪也是必须填写link key才能破解；谢谢！

[回复](http://www.wowotech.net/bluetooth/ble_ll_privacy.html#comment-4973)

**AndyPeng**  
2016-12-06 14:17

@AndyPeng：我的理解是既然监听是全程的窥视，双方没有‘秘密’，怎么实现加密，想不明白；

[回复](http://www.wowotech.net/bluetooth/ble_ll_privacy.html#comment-4974)

**[wowo](http://www.wowotech.net/)**  
2016-12-06 14:51

@AndyPeng：虽然监听是全程的窥视，但也只能窥视到空中传输的信息，但蓝牙的安全机制里面，有一个东西（pincode，或者说link key），不会在空中传输。这就保证了窥视者无法解密。

[回复](http://www.wowotech.net/bluetooth/ble_ll_privacy.html#comment-4975)

**[wowo](http://www.wowotech.net/)**  
2016-12-06 14:52

@wowo：顺便多说一句，空中传输的是加密后的密文，如果有人能从密文里解出key，就可以破解了（这是有可能的）。

[回复](http://www.wowotech.net/bluetooth/ble_ll_privacy.html#comment-4976)

**AndyPeng**  
2016-12-06 15:09

@wowo：感谢这么及时的回复，刚才也度娘了一下，也基本了解了，justwork方式在配对过程中是不提供MITM保护的；我起初误解以为justwork是受到MITM保护，因为这是无法理解的事情；谢谢！

[回复](http://www.wowotech.net/bluetooth/ble_ll_privacy.html#comment-4978)

**[chen_chuang](http://www.wowotech.net/)**  
2016-12-02 14:49

wowo大神，有没有讲解蓝牙室内定位的计划

[回复](http://www.wowotech.net/bluetooth/ble_ll_privacy.html#comment-4951)

**[wowo](http://www.wowotech.net/)**  
2016-12-03 11:10

@chen_chuang：帅锅，不要叫大神啊，不敢当啊！！  
暂时没有涉及到室内定位的打算啊，因为觉得和蓝牙的关系不大啊，室内定位的关键点在于拿到TX Power和RSSI之后的算法吧？

[回复](http://www.wowotech.net/bluetooth/ble_ll_privacy.html#comment-4955)

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
    
    - ja  
        [@dream：我看完這段也有相同的想法，引用 @dream ...](http://www.wowotech.net/kernel_synchronization/spinlock.html#8922)
    - 元神高手  
        [围观首席power managerment专家](http://www.wowotech.net/pm_subsystem/device_driver_pm.html#8921)
    - 十七  
        [内核空间的映射在系统启动时就已经设定好，并且在所有进程的页表...](http://www.wowotech.net/process_management/context-switch-arch.html#8920)
    - lw  
        [sparse模型和disconti模型没看出来有什么本质区别...](http://www.wowotech.net/memory_management/memory_model.html#8919)
    - 肥饶  
        [一个没设置好就出错](http://www.wowotech.net/linux_kenrel/516.html#8918)
    - orange  
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
    
    - [linux kernel的中断子系统之（四）：High level irq event handler](http://www.wowotech.net/irq_subsystem/High_level_irq_event_handler.html)
    - [ARM64的启动过程之（四）：打开MMU](http://www.wowotech.net/armv8a_arch/turn-on-mmu.html)
    - [一个较复杂dcache问题](http://www.wowotech.net/linux_kenrel/484.html)
    - [Linux TTY framework(1)_基本概念](http://www.wowotech.net/tty_framework/tty_concept.html)
    - [Linux2.6.23 ：sleepable RCU的实现](http://www.wowotech.net/kernel_synchronization/linux2-6-23-RCU.html)
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