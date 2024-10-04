
作者：[wowo](http://www.wowotech.net/author/2 "runangaozhong@163.com") 发布于：2017-3-28 11:52 分类：[蓝牙](http://www.wowotech.net/sort/bluetooth)

## 1. 前言

前面文章介绍了两种BLE的安全机制：白名单[4]和LL privacy[3]。说实话，在这危机四伏的年代，这两种“捂着脸讲话（其它人不知道是谁在讲话，因而不能插话、不能假传圣旨，但讲话的内容却听得一清二楚）”的方法，实在是小儿科。对于物联网的应用场景来说，要做到安全，就必须对传输的数据进行加密，这就是LE Encryption要完成的事情（当然，只针对面向连接的数据），具体请参考本文的介绍。

## 2. 基本概念

从字面理解，Encryption是一个名词，意思是“加密术”，因此LE Encryption就是“BLE所使用的加密技术”的意思。了解加密概念的同学应该都知道，通信过程中的加密无非就是如下简单的流程：

> 数据发送方在需要发送数据的时候，按照一定的加密算法，将数据加密；
> 
> 数据接收方在接收到数据的时候，按照等同的解密算法，将数据解密。

因此，对LE Encryption来说，需要考虑的事情无非就两条：

> 1）加密/解密的事情，需要在协议的哪个层次去做？
> 
> 2）使用什么样的加密/解密算法？

对问题1来说，很好回答：无论是从安全的角度，还是从通信效率的角度，都应该由链路层（LL，Link Layer[1]）在发送/接收数据的时候，完成加密/解密动作（具体可参考）。而问题2呢，到底要使用什么的算法，这是蓝牙标准化组织的事情，我们在本文只需要了解最终的结论即可（具体可参考）。

## 3. packet的加密/解密过程

LE加密/解密的过程如下面图片1所示：

[![LE加密解密](http://www.wowotech.net/content/uploadfile/201703/e7b0bba362d92ccda866ce74cffcada720170328035225.gif "LE加密解密")](http://www.wowotech.net/content/uploadfile/201703/fb35af6a4e96b398c26682018083d25a20170328035225.gif)

图片1 LE加密/解密过程

> Master/Slave的LE Host会保存一个LTK（Long Term Key，至少128bits），对BLE用户（或者应用）来说，这个Key是所有加密/解密动作源头；
> 
> 每当为某个LE连接启动加密传输的时候，Master和Slave的LL会协商生成一个128bits的随机数SKD（Session Key Diversifier，128bits），并以它为输入，以LTK为key，通过Encryption Engine加密生成SessionKey；
> 
> 每当有明文数据包需要发送的时候，需要对明文进行加密。加密的过程，是以明文数据包为输入，以SessionKey为Key，同样通过Encryption Engine加密生成密文数据包；
> 
> 同样，每当收到密文数据包的时候，需要对密文解密。解密的过程，是以密文数据包为输入，以SessionKey为Key，同样通过Encryption Engine解密生成明文数据包。

理解了加密/解密的过程之后，问题随之而来：

1）LTK是怎么来的？

2）SKD是个什么东西？怎么来的？

3）Encryption Engine又什么东西呢？

对于问题1，需要由SMP解答（也就是我们常说的配对过程），具体可参考后续的文章。问题3比较单纯，就是一个由Bluetooth specifiction所规定的加密算法，后面会单独写一篇文章介绍。而问题2，则涉及到LE Encryption的操作流程，具体可参考后面第4章的介绍。

## 4. Encryption Procedure

LE Encryption的过程主要由Link Layer控制（具体可参考“BLUETOOTH SPECIFICATION Version 4.2 [Vol 6, Part B] 5.1.3 Encryption Procedure”）：当连接建立之后，Link Layer可以应Host的请求，使能对数据包的Encryption操作，过程如下（具体可参考“BLUETOOTH SPECIFICATION Version 4.2 [Vol 6, Part D]  6.6 START ENCRYPTION”）：

[![LE encryption](http://www.wowotech.net/content/uploadfile/201703/0a35e18ba0960f0f8e6468123668b5ab20170328035226.gif "LE encryption")](http://www.wowotech.net/content/uploadfile/201703/77a80593c0ea43f0ed5d3ee944b0365320170328035226.gif)

图片2 Start Encryption

1）Host A发送LE Start Encryption HCI命令，请求Link Layer启动加密。该命令的格式如下：

> |   |   |   |   |
> |---|---|---|---|
> |Command|OCF|Command parameters|Return Parameters|
> |HCI_LE_Start_Encryption|0x0019|Connection_Handle  <br>Random_Number  <br>Encrypted_Diversifier  <br>Long_Term_Key||
> 
> Connection_Handle，连接句柄；  
> Random_Number和Encrypted_Diversifier分别简称为Rand和EDIV（Rand是一个64bits的随机数，EDIV是一个16bits的Diversifier），它们在LE Legacy Pairing的过程中，用于在多个LTK标识某一个具体的LTK。而在新的LE Secure Connections Pairing过程中，则不再使用（赋值为0即可）。关于LE的配对过程，可参考后面SMP的分析文章，这里不再详细描述；  
> Long_Term_Key，保存在Host段的加密key。

2）LL A收到Host的加密请求之后，会向LL B发送LL_ENC_REQ PDU以请求加密，该PDU的格式为：

> |   |   |   |   |
> |---|---|---|---|
> |Rand (8 octets)|EDIV (2 octets)|SKDm (8 octets)|IVm (4 octets)|
> 
> Rand和EDIV就是上面的Random_Number和Encrypted_Diversifier；  
> SKDm（session key diversifier ），是一个64bits的随机数，用于和SKDs一起，生成本次加密的SessionKey；  
> IVm（initialization vector ），一个32bits的随机数，和IVs一起组成64bits的IV，后面Encryption Engine会使用。

3）LL B收到LL_ENC_REQ PDU之后，会向Host B发送LE Long Term Key Request HCI Event，该Event的格式为：

> |   |   |   |
> |---|---|---|
> |Event|Event code|Event Parameters|
> |LE Long Term Key Request|0x3E|Subevent_Code  <br>Connection_Handle  <br>Random_Number  <br>Encryption_Diversifier|
> 
> Subevent_Code为0x05；  
> Connection_Handle，连接句柄；  
> Random_Number和Encrypted_Diversifier就是LL_ENC_REQ PDU中的Rand和EDIV，最初是由Host A通过LE Start Encryption命令发送过来的。

4）如果Host B能提供LTK，则通过LE Long Term Key Request Reply HCI命令，将LTK提供给LL B，该命令的格式为：

> |   |   |   |   |
> |---|---|---|---|
> |Command|OCF|Command parameters|Return Parameters|
> |HCI_LE_Long_Term_Key_  <br>Request_Reply|0x001A|Connection_Handle  <br>Long_Term_Key|status  <br>Connection_Handle|

5）LL B收到LTK之后，会向LL A回应一个LL_ENC_ RSP PDU，该PDU的格式为：

> |   |   |
> |---|---|
> |SKDs (8 octets)|IVs (4 octets)|
> 
> SKDs和LL A的SDKm共同组成128bits的SKD；  
> IVs和LL A的IVm共同组成64bits的IV。

6）LL A收到LL_ENC_ RSP PDU之后，可以向LL B发送LL_START_ENC_REQ PDU，开启Encryption，LL B则回应LL_START_ENC_RSP PDU，这两个PDU均不携带任何的参数。

加密start之后，双方就可以安全的通信了。当然，LE encryption还提供了其它诸如暂停加密（LL_PAUSE_ENC_REQ/LL_PAUSE_ENC_RSP）、重启加密等过程，比较简单，这里就不详细介绍了，具体可参考BLE Spec[2]中的相关描述。

## 5. 参考文档

[1] [蓝牙协议分析(3)_蓝牙低功耗(BLE)协议栈介绍](http://www.wowotech.net/bluetooth/ble_stack_overview.html)

[2] Core_v4.2.pdf

[3] [蓝牙协议分析(9)_BLE安全机制之LL Privacy](http://www.wowotech.net/bluetooth/ble_ll_privacy.html)

[4] [蓝牙协议分析(8)_BLE安全机制之白名单](http://www.wowotech.net/bluetooth/ble_white_list.html)

原创文章，转发请注明出处。蜗窝科技，[www.wowotech.net](http://www.wowotech.net/bluetooth/le_encryption.html)。

标签: [蓝牙](http://www.wowotech.net/tag/%E8%93%9D%E7%89%99) [Bluetooth](http://www.wowotech.net/tag/Bluetooth) [BLE](http://www.wowotech.net/tag/BLE) [安全](http://www.wowotech.net/tag/%E5%AE%89%E5%85%A8) [Encryption](http://www.wowotech.net/tag/Encryption)

---

« [Linux DMA Engine framework(1)_概述](http://www.wowotech.net/linux_kenrel/dma_engine_overview.html) | [中断上下文中调度会怎样？](http://www.wowotech.net/process_management/schedule-in-interrupt.html)»

**评论：**

**bright**  
2019-03-12 11:23

Hi Wowo  
最近接觸BLE  
請教一下 https://blog.bluetooth.com/bluetooth-pairing-part-4  
所謂的LTK 是用DHKey再算出來的是嘛?  為什麼已經算出共享金鑰DHkey了還要再算出LTK?  
DHkey不能用來建立加密通道嗎?

[回复](http://www.wowotech.net/bluetooth/le_encryption.html#comment-7271)

**vellt**  
2018-08-09 17:33

请问什么时候能写些GAP层Security有关的定义的介绍

[回复](http://www.wowotech.net/bluetooth/le_encryption.html#comment-6877)

**EE**  
2018-03-11 16:01

wowo,最近好哦，又来请教了！  
   1、如果两台设备间进行了绑定，当每次重连时如果链路设置了安全模式（数据要加密），那么加密是不是要进行加密规程，比如发LL_ENC_REQ，那么发第一条LL_ENC_REQ时是没有加密的但从机设置了安全模式（数据要加密）要求第一请求都要加密，这个时候从机是不是要拒绝呢？  
   2、加密、认证等安全方案对LL Control PDU有用吗？  
   3、一台从机能绑定多少个中央主机，主机能绑定多少个从机？

[回复](http://www.wowotech.net/bluetooth/le_encryption.html#comment-6605)

**[wowo](http://www.wowotech.net/)**  
2018-03-13 16:17

@EE：1. 加密是以链路为单位的，所以如果断了之后，就不会拒绝非加密的LL_ENC_REQ请求。  
2. 看看spec吧，我记得不太清楚。  
3. 原则上可以很多，多的不用去考虑，但实际上会受到资源的限制。

[回复](http://www.wowotech.net/bluetooth/le_encryption.html#comment-6608)

**EE**  
2018-03-13 17:51

@wowo：1、如果如你所说，那加密模式设置就没有意思了，因为没加密也可以向从机发数据？！  
2、spec上没找到  
3、谢谢  
加问一条：配对请求在链路层是用数据控制通道还是用数据通道了？？

[回复](http://www.wowotech.net/bluetooth/le_encryption.html#comment-6611)

**cc**  
2023-04-04 22:10

@EE：控制

[回复](http://www.wowotech.net/bluetooth/le_encryption.html#comment-8764)

**EE**  
2017-12-07 16:35

wowo请问，如果做了净荷数据加密，MIC就一定要加上吗？还有就是MIC而言，如果一个数据有四块组成，块1，2，3，4，第一次发了块1，2并带上了MIC，这个MIC是对数据块1，2的真实性进行验证，如果此时第二个传输的包数据块3，4被攻击者替换成A，B数据其它的算法和密钥和第一个数据包的一样从而计算得到的第二个包的MIC，此时接收方接到第二个包时会不会出错，第二个问题我其实想问的是，如果一个数据包较大需要几次才能传完，那么每一次传输的MIC只对本次包的数据真正性做校验，还是说，后面每一个未传完的包中的MI是否都都相临的上一个包中的未数据有关？如果后续某一个包的数据全部替换再来计算MIC接收方会不会报算?

[回复](http://www.wowotech.net/bluetooth/le_encryption.html#comment-6302)

**EE**  
2017-12-07 16:37

@EE：wowo请问，如果做了净荷数据加密，MIC就一定要加上吗？还有就是MIC而言，如果一个数据有四块组成，块1，2，3，4，第一次发了块1，2并带上了MIC，这个MIC是对数据块1，2的真实性进行验证，如果此时第二个传输的包数据块3，4被攻击者替换成A，B数据其它的算法和密钥和第一个数据包的一样从而计算得到的第二个包的MIC，此时接收方接到第二个包时会不会出错，第二个问题我其实想问的是，如果一个数据包较大需要几次才能传完，那么每一次传输的MIC只对本次包的数据真正性做校验，还是说，后面每一个未传完的包中的MIC是否都都相临的上一个包中的未数据有关？如果后续某一个包的数据全部替换再来计算MIC接收方会不会报错?

[回复](http://www.wowotech.net/bluetooth/le_encryption.html#comment-6303)

**[wowo](http://www.wowotech.net/)**  
2017-12-08 10:39

@EE：MIC（Message Authentication Code）其实是用来的鉴权的，和加密是两个维度的事情。MIC一般是根据PDU等信息使用对称加密得到的（“The MIC shall be calculated over the Data Channel PDU’s Payload field and the first octet of the header (Part B, Section 2.4) with the NESN, SN and MD bits masked to zero.”）。  
得到MIC之后，DATA PDU + MIC，然后去做后续的加密过程。  
所以每一包的MIC，只和这个包有关。

[回复](http://www.wowotech.net/bluetooth/le_encryption.html#comment-6318)

**[wowo](http://www.wowotech.net/)**  
2017-12-08 10:39

@EE：MIC（Message Authentication Code）其实是用来的鉴权的，和加密是两个维度的事情。MIC一般是根据PDU等信息使用对称加密得到的（“The MIC shall be calculated over the Data Channel PDU’s Payload field and the first octet of the header (Part B, Section 2.4) with the NESN, SN and MD bits masked to zero.”）。  
得到MIC之后，DATA PDU + MIC，然后去做后续的加密过程。  
所以每一包的MIC，只和这个包有关。

[回复](http://www.wowotech.net/bluetooth/le_encryption.html#comment-6319)

**EE**  
2017-12-08 11:57

@wowo：是我表达不详细，不是MIC（Message Authentication Code），而是MIC（Message Integrity Check (MIC)）消息完整性校验，用于验证数据包的发送者，会用到AES，将数据分块，一个块的输出将用于一个块的输入，各个块串联起来用来校验信息完整性的，如果“每一包的MIC，只和这个包有关”如果是这样那MIC和CRC功能上没有区别了，我现不明白：如果一个需要分包发送的数据，第二个包数据整体发现了改变（MIC和CRC都是根据这个改变的数据计算得到，所以接收方不会报错），如果是这样蓝牙有没有机制甄别区来？第一个包（数据A由多个包组成）的MIC计算是从length字段,到header字段,再到这个包（不是指数据A，是指数据A的第一个包也即本次包）中最后的净荷块最后得出MIC，第二个包发送时也会第一个包一样计算吗也从包的length字段,到header字段再到数据块，还是说第一个包中某此数据也参与第二个数据包的MIC计算？如果不参与怎么确保第二个数据与第一个数据包的相关性（安全角度上说）

[回复](http://www.wowotech.net/bluetooth/le_encryption.html#comment-6321)

**[wowo](http://www.wowotech.net/)**  
2017-12-08 14:07

@EE：MAC和MIC是一个东西，只是叫法不同（为了不和网络里面那个MAC混淆）。维基上有一个图，可以很好的解释MIC的功能：  
https://en.wikipedia.org/wiki/File:MAC.svg  
其实就是：  
发送方：key+msg----MIC算法---->MIC  
接收方：key+msg----MIC算法---->MIC，然后对比packet里面的MIC和计算得到的MIC。  
和CRC的区别是，收发双方拥有同一个key（对称加密）。  
  
最后，怎么做到鉴权呢？其实很简单：  
MIC算法所使用的msg是加密前的msg，因此必须将其正确解密得到原始msg后，才能算出合法的MIC。  
数据伪造方随便伪造一个数据，由于不知道具体的加密信息，是无法做出来一个可以让接收方正确解密的MIC的。

[回复](http://www.wowotech.net/bluetooth/le_encryption.html#comment-6322)

**EE**  
2017-12-08 16:07

@wowo：估计一时半会是弄不明白，看看spec后再来重温问答

[回复](http://www.wowotech.net/bluetooth/le_encryption.html#comment-6327)

**EE**  
2017-12-11 21:57

@EE：wowo,请看BLUETOOTH SPECIFICATION Version 4.2 [Vol 6, Part B] // 2.4 DATA CHANNEL PDU：The Data Channel PDU has a 16 bit header, a variable size payload, and may include a Message Integrity Check (MIC) field.The MIC field shall not be included in an un-encrypted Link Layer connection, or in an encrypted Link Layer connection with a data channel PDU with a zero length Payload.The MIC field shall be included in an encrypted Link Layer connection, with a data channel PDU with a non-zero length Payload and shall be calculated as specified in [Vol. 6] Part E, Section 1.MIC只能在加密的链路中，这是不是说明了MIC与链路加密有关，或者说什么时候启动了MIC需要建立在链路加密之上？

[回复](http://www.wowotech.net/bluetooth/le_encryption.html#comment-6341)

**[wowo](http://www.wowotech.net/)**  
2017-12-12 15:50

@EE：是的，你看看我上一条回复，其实就是这个意思。如果没有加密，MIC完全就是一个摆设，没啥用。

**方言**  
2017-11-28 15:11

Wowo,"6)LL A收到LL_ENC_ RSP PDU之后，可以向LL B发送LL_START_ENC_REQ PDU"这段话，根据图，好像有误。是不是应该改为"6)LL A收到LL_ENC_ RSP PDU之后，其实已经得到自己的SessionKey,而LL B这个时候也计算得到了LTK和最终的SessionKey, 所以LL B主动发送LL_START_ENC_REQ PDU"。

[回复](http://www.wowotech.net/bluetooth/le_encryption.html#comment-6250)

**[wowo](http://www.wowotech.net/)**  
2017-11-29 09:19

@方言：多谢提醒，这一段描述确实有2个问题（稍后有空的话修改一下）：  
1. LL B回应LL_ENC_RSP，不需要等Host B的信息，因为这个PDU只包含salve的SDK和IV，LL完全可以搞定。  
2. LL_START_ENC_REQ请求是LL B主动发的。不过这一段可以存疑：LL A是不是也可以主动发？spec没有提（没说可以，也没说不可以）。  
  
谢谢～～～～

[回复](http://www.wowotech.net/bluetooth/le_encryption.html#comment-6255)

**sukey**  
2017-09-04 14:47

你好，请问一下BLE协议下有没有类似于tcpdump和iptables的工具呢 谢谢

[回复](http://www.wowotech.net/bluetooth/le_encryption.html#comment-5984)

**[wowo](http://www.wowotech.net/)**  
2017-09-04 17:13

@sukey：首先呢，工具不是协议定义的，而是具体的平台为了方便协议的开发和调试做出来的。  
其次呢，回到这个问题本身，你列举的这两个工具分别属于TCP层和IP层，而这两个层和BLE没有关系（你可以把BLE看作ISO 7层协议里面的物理层和链路层）。  
最后呢，其实我们可以在BLE之上增加TCP和IP，例如传说中的6LoWPAN。这时，你想用什么工具，都是okay的，前提要在相应的平台下用平台所提供的工具。

[回复](http://www.wowotech.net/bluetooth/le_encryption.html#comment-5985)

**三九感冒灵**  
2017-07-24 10:36

希望可以抽时间继续更新，很期待

[回复](http://www.wowotech.net/bluetooth/le_encryption.html#comment-5840)

**[wowo](http://www.wowotech.net/)**  
2017-07-25 08:46

@三九感冒灵：有时间的话会的，谢谢关注。

[回复](http://www.wowotech.net/bluetooth/le_encryption.html#comment-5844)

**Reashion**  
2017-07-13 11:53

好东西

[回复](http://www.wowotech.net/bluetooth/le_encryption.html#comment-5801)

**zx**  
2017-05-09 17:18

看spec，我感觉encryption在BLE里是放到了host里，而BR/EDR则是由controller完成

[回复](http://www.wowotech.net/bluetooth/le_encryption.html#comment-5528)

**[wowo](http://www.wowotech.net/)**  
2017-05-09 19:48

@zx：何以见得呢？你可以把你看到的东西贴出来，讨论讨论。

[回复](http://www.wowotech.net/bluetooth/le_encryption.html#comment-5529)

**zx**  
2017-05-15 15:48

@wowo：sorry,是我看错了，应该是BLE把除了encryption和privacy的一部分放在controller，其它的flow放到了host

[回复](http://www.wowotech.net/bluetooth/le_encryption.html#comment-5549)

**[wowo](http://www.wowotech.net/)**  
2017-05-15 16:16

@zx：是的。加密的事情，不能交给Host，不然安全性就得不到保证。

[回复](http://www.wowotech.net/bluetooth/le_encryption.html#comment-5550)

**ronald**  
2017-04-17 23:05

看你对蓝牙协议好熟悉，请问怎么熟悉一套协议了，我看蓝牙协议的时候，有点摸不着重点

[回复](http://www.wowotech.net/bluetooth/le_encryption.html#comment-5473)

**[wowo](http://www.wowotech.net/)**  
2017-04-19 08:58

@ronald：协议这种东西，用的多了自然熟了，如果不怎么用，看着会很枯燥无味的。

[回复](http://www.wowotech.net/bluetooth/le_encryption.html#comment-5474)

1 [2](http://www.wowotech.net/bluetooth/le_encryption.html/comment-page-2#comments)

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
    
    - [Linux时间子系统之（十四）：tick broadcast framework](http://www.wowotech.net/timer_subsystem/tick-broadcast-framework.html)
    - [Linux DMA Engine framework(1)_概述](http://www.wowotech.net/linux_kenrel/dma_engine_overview.html)
    - [Linux调度器：用户空间接口](http://www.wowotech.net/process_management/scheduler-API.html)
    - [linux cpufreq framework(1)_概述](http://www.wowotech.net/pm_subsystem/cpufreq_overview.html)
    - [perfbook memory barrier（14.2章节）中文翻译（下）](http://www.wowotech.net/kernel_synchronization/perfbook-memory-barrier-2.html)
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