
作者：[wowo](http://www.wowotech.net/author/2 "runangaozhong@163.com") 发布于：2017-3-11 11:42 分类：[蓝牙](http://www.wowotech.net/sort/bluetooth)

## 1. 前言

在物联网时代，有一个问题肯定会让人头疼（现在已经初露端倪了）：

> 物联网中的IOT设备有两个主要特点：  
> 1）简单小巧（不具备复杂的人机交互接口，需要手机等终端设备辅助完成配置、控制等功能）。  
> 2）数量和种类繁多（消费者面对的可是数量众多的不同厂家、不同类型的设备）。
> 
> 基于这两个特点，手机等终端设备一般通过APP（或APK）对IOT设备进行控制，不同厂家的不同设备，通常需要不同的APP/APK。于是出现了这样的结果：  
> 如果你家里有一个小米yeelight的床头灯，你需要安装一个yeelight的APP/APK；  
> 如果你手上戴一个百度智能手环，你需要安装一个百度智能手环APP/APK；  
> 如果你跑步时戴一个xxx运动蓝牙耳机，你需要安装一个xxx运动蓝牙耳机APP/APK；  
> 如果你要骑摩拜单车，你需要安装一个摩拜单车APP/APK；  
> 如果你也行骑一下优拜单车，你需要安装一个优拜单车APP/APK；  
> ……（要列的话，我觉得永远都列不完，大家可以想象一下，这是不是一个灾难？？）

既然有问题，肯定就有人试图解决，于是微信IoT、阿里小智、亚马逊IoT等等，互联网大佬们就各显神通了。最终谁能称霸天下，现在还不得而知，本文也不去讨论这些。

作为蓝牙BLE的介绍文章，本文将以微信IoT的“微信蓝牙精简协议”为例，通过“把一个蓝牙适配器模拟成微信计步器”，分别从BLE技术（怎样注册一个GATT service）和微信IoT（微信物联网平台的思路和想法）两个角度，窥一窥IoT江湖的冰山一角（权当开阔眼界了）。

## 2. 微信蓝牙精简协议简介

微信蓝牙精简协议的思路很简单（具体可参考“[http://iot.weixin.qq.com/wiki/new/index.html?page=4-3](http://iot.weixin.qq.com/wiki/new/index.html?page=4-3 "http://iot.weixin.qq.com/wiki/new/index.html?page=4-3")“），如下图：

[![微信蓝牙精简协议-框图](http://www.wowotech.net/content/uploadfile/201703/8f462a23fdb34fdb6b8d38d5969921c420170311034149.gif "微信蓝牙精简协议-框图")](http://www.wowotech.net/content/uploadfile/201703/e9b5ae4516aafc95f622f0acb2b333ac20170311034148.gif)

图片1 微信蓝牙精简协议框架

总结来说，就是：

> 通过微信（可以运行在手机、平板、电脑、等硬件上面）统一和IoT设备交互；
> 
> 交互的介质是BLE协议（可以摇身变为其它无线协议）；
> 
> 交互的过程可由服务器协助完成（服务器可选择微信服务器或者第三方服务器）。

虽然思路简单，背后却有深深的“不怀好意”，因为：

> 1）IoT设备想要和微信（或者微信背后的服务器）交互，就必须遵守微信定义的协议；
> 
> 2）IoT设备通过微信和服务的交互（无论是第三方服务器，还是微信服务器），都必须遵守微信定义的协议，并经由微信服务器转发。
> 
> 基于上面两点，物联网中最重要的两个环节：设备和数据，都掌握在微信的手中了。恐怖！！

到目前为止，“微信蓝牙精简协议”有一个比较成熟的应用实例：微信计步器，其工作流程为：

> 1）在带有计步功能的BLE设备上，按照协议规定的方式，基于BLE GATT，实现相应的Profile、Service、Attribute。
> 
> 2）在微信公众平台，为该设备申请一个唯一的设备ID，并和设备的MAC地址一起，在公众平台上注册、授权。
> 
> 3）在微信的微信运动小程序中，扫描并添加该设备，即可通过BLE读取设备的计步信息。
> 
> 4）微信运动读取计步信息后，可以进行后续的动作（例如在朋友圈刷排名等）。

以上步骤，后面章节将会使用一个蓝牙适配器（模拟蓝牙计步器），配合自己的手机微信，进行演示说明。

## 3. 将蓝牙适配器模拟成一个计步器

#### 3.1 环境准备

后续实验基于如下的硬件、软件环境（其它环境也okay，不过我没有测试）。

> 运行Ubuntu为例的PC、笔记本（或者开发板），包含蓝牙4.0（及以上）功能（自带或者使用蓝牙适配器）；
> 
> Ubuntu中安装有较新版本的Bluez协议栈及工具集，以及其它必须的软禁包（具体步骤略，碰到问题的时候可以一点点解决）；
> 
> 具有BLE功能的手机（上面有可以正常使用的微信）；

#### 3.2 搭建Golang开发环境

在包含Bluez（及相关工具集）的Linux OS中，使用Bluez的工具，可以完成大部分的BLE实验，例如：

> hcitool、bluetoothctl等工具，可以进行BLE设备的扫描、连接、配对、广播等操作；
> 
> hcitool可以发送HCI command，设置BLE的广播数据；
> 
> gatttool可以在GATT层面，完成GATT profile的连接、service attribute的读写等操作；
> 
> 等等。

但有一个功能点，不是很容易实现，就是如何自定义一个GATT profile（要知道，BLE 90%以上的功能都是通过GATT实现的，如果做到这一点，学习BLE就非常方便了）。当然，BLE不复杂，我们可以直接使用BLuez提供的API，自行开发。如果是产品开发，无可厚非，但是如果仅仅只做个实验，这样大张旗鼓的就不划算了，最好能有一些开源的工具。确实有，我搜集、尝试过几种工具：

> 1）使用bluez的测试代码（bluez-5.37/test/example-gatt-server），是一个python脚本。这个家伙对环境依赖度比较高，很难用，最终没有用起来。
> 
> 2）一个由nodejs脚本写的工具（[https://github.com/luluxie/weixin-iot](https://github.com/luluxie/weixin-iot "https://github.com/luluxie/weixin-iot")）。基本功能还好，不过本人对js不太熟，就算了。
> 
> 3）一个由Go语言写的工具（[https://github.com/paypal/gatt）](https://github.com/paypal/gatt "https://github.com/paypal/gatt")。功能OK，关键还是新奇的Go语言，果断使用～～

这就是搭建Golang开发环境的原因。至于步骤，很简单（以Ubuntu为例）：

> 1）安装Go语言  
> sudo apt install golang
> 
> 2）创建GOPATH环境变量  
> mkdir ~/gopath  
> export GOPATH=$HOME/gopath    #为了方便，可以放到~/.profile中

#### 3.3 代码准备

在github上，从[https://github.com/paypal/gatt](https://github.com/paypal/gatt)中clone一个仓库，clone的仓库地址为[https://github.com/wowotech/gatt](https://github.com/wowotech/gatt)。clone后使用go get命令，下载代码到本地：

> vim@ubuntu:~$ go get github.com/wowotech/gatt

下载后代码的位置为：
```sh
vim@ubuntu:~$ ls $GOPATH/src/github.com/wowotech/ 
gatt
```
然后，手动将代码中所有的“github.com/paypal”类型的import修改为“github.com/wowotech”（fuck Go！！）

> cd $GOPATH/src/github.com/wowotech/gatt
> 
> find . -name "*.go" | xargs sed -i 's/paypal/wowotech/g'

提交记录如下：

> [https://github.com/wowotech/gatt/commit/1f1c23e94cb7e1b54078a568a60cf1aaef7de195](https://github.com/wowotech/gatt/commit/1f1c23e94cb7e1b54078a568a60cf1aaef7de195)

#### 3.4 编译并运行一个示例文件.

（略）。

#### 3.5 支持“微信蓝牙精简协议”

参考examples/server.go以及蓝牙精简协议的定义[1]，新建weixin.go，增加微信蓝牙精简协议。最终的文件如下（代码很简单，熟悉BLE GATT协议的同学，很容易看懂，就不过多解释了）：

> [https://github.com/wowotech/gatt/commit/942e7480ad1664dabacdb5561fa98a831621f7de](https://github.com/wowotech/gatt/commit/942e7480ad1664dabacdb5561fa98a831621f7de "https://github.com/wowotech/gatt/commit/942e7480ad1664dabacdb5561fa98a831621f7de")

注1：可以通过代码中如下的定义修改计步的步数（想刷爆微信运动朋友圈的同学注意了，嘿嘿！！）：

> + // steps little endian
> 
> + // 01(steps) 10 27 00(0x002710 = 10000)
> 
> + // http://iot.weixin.qq.com/wiki/new/index.html?page=4-3
> 
> + steps := []byte{ 0x01, 0x5c, 0x74, 0x01 }

#### 3.6 编译、运行、测试

编译：

> vim@ubuntu:~/gopath/src/github.com/wowotech/gatt/examples$ go build weixin.go

运行：

> vim@ubuntu:~/gopath/src/github.com/wowotech/gatt/examples$ sudo ./weixin
> 
> [sudo] password for vim:
> 
> 2017/03/04 01:03:13 dev: hci0 up  
> 2017/03/04 01:03:14 dev: hci0 down  
> 2017/03/04 01:03:14 dev: hci0 opened  
> State: PoweredOn  
> 2017/03/04 01:03:14 BD Addr: 5C:F3:70:6A:BA:27  
> 2017/03/04 01:03:14 Generating attribute table:  
> 2017/03/04 01:03:14 handle type props secure pvt value  
> 2017/03/04 01:03:14 0x0001 0x2800 0x02 0x00 *gatt.Service [ E7 FE ]  
> 2017/03/04 01:03:14 0x0002 0x2803 0x32 0x00 *gatt.Characteristic [ 32 03 00 A1 FE ]  
> 2017/03/04 01:03:14 0x0003 0xfea1 0x32 0x00 *gatt.Characteristic [ ]  
> 2017/03/04 01:03:14 0x0004 0x2902 0x0E 0x00 *gatt.Descriptor [ 00 00 ]  
> 2017/03/04 01:03:14 0x0005 0x2803 0x3E 0x00 *gatt.Characteristic [ 3E 06 00 A2 FE ]  
> 2017/03/04 01:03:14 0x0006 0xfea2 0x3E 0x00 *gatt.Characteristic [ ]  
> 2017/03/04 01:03:14 0x0007 0x2902 0x0E 0x00 *gatt.Descriptor [ 00 00 ]  
> 2017/03/04 01:03:14 0x0008 0x2803 0x02 0x00 *gatt.Characteristic [ 02 09 00 C9 FE ]  
> 2017/03/04 01:03:14 0x0009 0xfec9 0x02 0x00 *gatt.Characteristic [ ]

然后在手机下载AirSyncDebugger2.3.0.apk（[http://iot.weixin.qq.com/wiki/new/index.html?page=4-3](http://iot.weixin.qq.com/wiki/new/index.html?page=4-3)的底部）测试精简协议是否okay：

> 打开APK---->精简协议---->计步器测试

测试界面如下（Android平台）：

[![wx-蓝牙精简协议-AirSyncDebugger](http://www.wowotech.net/content/uploadfile/201703/3c26d09fdcbc5720c2b118f92ed4f38020170311035024.gif "wx-蓝牙精简协议-AirSyncDebugger")](http://www.wowotech.net/content/uploadfile/201703/801a5f17b152b1884cbefeb7070633aa20170311035023.gif)

## 4. 在微信公众平台注册并授权设备

第2章成功模拟出来一个遵守“微信蓝牙精简协议”的计步器之后，我们需要登录微信的公众平台（可以使用测试帐号），注册一个产品类别，并授权该设备，之后就可以在微信运动界面使用这个设备了。

#### 4.1 登录“微信公众平台接口测试帐号”

打开下面链接：

[http://mp.weixin.qq.com/debug/cgi-bin/sandbox?t=sandbox/login](http://mp.weixin.qq.com/debug/cgi-bin/sandbox?t=sandbox/login)

点击登录按钮，用自己微信的扫一扫登录，登录后的界面如下：

[![wx-iot-登录界面](http://www.wowotech.net/content/uploadfile/201703/e4718e03a9ad108655faed51f1e3dede20170311035025.gif "wx-iot-登录界面")](http://www.wowotech.net/content/uploadfile/201703/69e83b9ff1d4742ac740b292f959c31b20170311035025.gif)

注意图中红色涂抹的三个信息（要记录下来，后面有用）：

> 微信号：  
> appID：  
> appsecret：

#### 4.2 开启设备功能接口

将上面的登录界面往下拉，找到“功能服务”，“设备功能”，点击“开启”，开启设备功能接口，如下图：

[![wx-iot-设备功能接口](http://www.wowotech.net/content/uploadfile/201703/ee1b2bd43a61ff7fd99414ee2bead83f20170311035026.gif "wx-iot-设备功能接口")](http://www.wowotech.net/content/uploadfile/201703/4002748d55e3ef83c015c851b734186a20170311035026.gif)

#### 4.3 添加产品

设备功能接口开启后，会出现“设置”按钮，点击进入下一个界面，进行设备功能管理。在该界面，点击“添加产品”按钮，为我们的计步器设备添加一个产品类别：

[![wx-iot-添加产品](http://www.wowotech.net/content/uploadfile/201703/cff110489efab3a74063cb7141d04b0120170311035027.gif "wx-iot-添加产品")](http://www.wowotech.net/content/uploadfile/201703/ff8d489cdce08a88e7d8ad2726ff3cb120170311035027.gif)

按照提示填入相关的信息（需要注意红色圈出的地方）：

[![wx-iot-基础资料登记1](http://www.wowotech.net/content/uploadfile/201703/093f0d51fc9f6e69a12def20df1458bc20170311035028.gif "wx-iot-基础资料登记1")](http://www.wowotech.net/content/uploadfile/201703/d7fe96cb32bbffb6e02c8c967b18771d20170311035028.gif)

[![wx-iot-基础资料登记2](http://www.wowotech.net/content/uploadfile/201703/c0b150e4eb768306de5287a9c4d326bc20170311035029.gif "wx-iot-基础资料登记2")](http://www.wowotech.net/content/uploadfile/201703/a8875994cfd2124bce3218a0a8ab693920170311035029.gif)

点击“下一步”进入“产品能力登记”界面：

[![wx-iot-产品能力登记](http://www.wowotech.net/content/uploadfile/201703/4a49dae6018c8ebce4825ed96309728420170311035031.gif "wx-iot-产品能力登记")](http://www.wowotech.net/content/uploadfile/201703/0f9a0a839a289727c4635cb514e9fc3020170311035030.gif)

最后，点击“添加”按钮，将产品添加进去。添加成功后，进入如下的界面（注意其中红色涂抹的那一串数字，是产品的ID，记下来，后面有用）：

[![wx-iot-product_id](http://www.wowotech.net/content/uploadfile/201703/3870a5c05d6d6889e4d5c3791c08c06220170311035032.gif "wx-iot-product_id")](http://www.wowotech.net/content/uploadfile/201703/3ffe94ae28e19280739652c4001596bd20170311035031.gif)

#### 4.4 授权设备

**4.4.1 获取access_token**

根据上面得到的appID和appsecret，在浏览器里面输入如下指令，获取访问微信公众平台的token（令牌）：

https://api.weixin.qq.com/cgi-bin/token?grant_type=client_credential&appid=xxxxxxxxxx&secret=xxxxxxxx

注意红色部分要修改为实际内容（具体可参考4.1章节获取的信息）。

浏览器会返回如下内容：

> {"access_token":"xxxxxxxxxxxxxxxxxxx","expires_in":7200}

有两个字段，access_token（记录下来，后面有用）和有效时间（7200秒，很长了）。

**4.4.2 为设备生成一个唯一ID（device_id）及二维码（通过微信扫描即可添加设备）**

在浏览器输入如下指令：

https://api.weixin.qq.com/device/getqrcode?access_token=xxxxxxxxxxxx&product_id=xxxxx

其中access_token为4.4.1中获取的，product_id为4.3中获取的。

浏览器的返回如下：

> {"base_resp":{"errcode":0,"errmsg":"ok"},"deviceid":"xxxxxx","qrticket":"http:\/\/we.qq.com\/d\/AQArGYez06UDOcqKNe1jqWeeLhiF7fIKebEP5hiT"}

deviceid为新生成的唯一ID，记录下来，后面有用。

qrticket是设备的二维码，随便找一个二维码生成的网站（例如[http://cli.im/](http://cli.im/)），就可得到图片二维码，如下（注意需要将浏览器返回的转义字符改回正常）：

[![wx-iot-二维码获取](http://www.wowotech.net/content/uploadfile/201703/01d97c6ad1a29f9062fc9f07e86e7e3920170311035033.gif "wx-iot-二维码获取")](http://www.wowotech.net/content/uploadfile/201703/0c6bf00bed43e4241ec6e687538bb70120170311035032.gif)

**4.4.3 设备鉴权**

这一步要借助微信公众平台的debug页面了，单纯的浏览器无法搞定。打开如下的debug地址：

> [http://mp.weixin.qq.com/debug/](http://mp.weixin.qq.com/debug/)

找到如下界面：

[![wx-iot-设备授权](http://www.wowotech.net/content/uploadfile/201703/da6a58a408e7a54c932090e2aa29e86320170311035034.gif "wx-iot-设备授权")](http://www.wowotech.net/content/uploadfile/201703/357b8e0799ce33e8c3e797eb70d703b620170311035034.gif)

body输入的内容如下：

> {  
>     "device_num":"1",  
>     "device_list":[   
>      {  
>         "id":" xxxxxxxx",  
>         "mac":"5CF3706ABA27",   
>         "connect_protocol":"3",  
>         "auth_key":"",  
>         "close_strategy":"1",   
>         "conn_strategy":"5",  
>         "crypt_method":"0",  
>         "auth_ver":"0",  
>         "manu_mac_pos":"-1",  
>         "ser_mac_pos":"-2",  
>         "ble_simple_protocol": "1"  
>     }  
>     ],  
>     "op_type":"0",  
>     "product_id": "xxxxx"  
> }

注意上面红色的字段，id是4.4.2中获得的deviceid，mac是蓝牙设备的物理地址，product_id是4.3中获得产品ID的。

点击“检查问题”，看到如下的成功信息，说明授权成功了：

[![wx-iot-授权成功](http://www.wowotech.net/content/uploadfile/201703/8a3078de65dc1a410fdc1c91d43b1f8020170311035036.gif "wx-iot-授权成功")](http://www.wowotech.net/content/uploadfile/201703/fb1f3ab18cbce5295ae795d63717098720170311035035.gif)

## 5. 使用手机微信绑定设备

使用微信扫一扫，扫描刚才获得的二维码，点击“绑定设备”，即可添加设备，如下：

[![wx-扫一扫绑定设备](http://www.wowotech.net/content/uploadfile/201703/0150f57a6dee327744e45127586facd920170311035037.gif "wx-扫一扫绑定设备")](http://www.wowotech.net/content/uploadfile/201703/7357f550baefad98e82cafcb85730aa420170311035036.gif)

绑定成功后，设备显示未连接，如下：

[![wx-绑定设备未连接](http://www.wowotech.net/content/uploadfile/201703/872eb8b21a7c8d53bb8e56032fffb40020170311034209.gif "wx-绑定设备未连接")](http://www.wowotech.net/content/uploadfile/201703/89c57c8ec0ba4e5fdedb924120b32f3720170311034208.gif)

接下来要连接设备。对iOS来说，在设置界面无法主动连接BLE设备，必须有应用程序自行连接，这里以微信运动为例，搜索并连接刚才授权的那个设备，即可看到计步数据的更新。步骤如下面图片（不再解释了，大家可以自己试试）：

[![wx-微信运动-设置](http://www.wowotech.net/content/uploadfile/201703/b97837a961548c4ccc592a5e766149c120170311034210.gif "wx-微信运动-设置")](http://www.wowotech.net/content/uploadfile/201703/16b8b149232eafea146907597829b2d120170311034209.gif)

[![wx-微信运动-添加数据来源](http://www.wowotech.net/content/uploadfile/201703/695e217b9f8275de82a4e7ff44a7c49a20170311034212.gif "wx-微信运动-添加数据来源")](http://www.wowotech.net/content/uploadfile/201703/c4a4bd51dc3c6f9427733731f1bc58cf20170311034211.gif)

[![wx-微信运动-搜索并添加设备](http://www.wowotech.net/content/uploadfile/201703/7eb66f0f2812121a53a04c06ca39729c20170311034213.gif "wx-微信运动-搜索并添加设备")](http://www.wowotech.net/content/uploadfile/201703/6cd07beddddb64e7ac5d1fd466d75d1b20170311034212.gif)

## 6. 参考文档

[1] 微信蓝牙精简协议，[http://iot.weixin.qq.com/wiki/new/index.html?page=4-3](http://iot.weixin.qq.com/wiki/new/index.html?page=4-3 "http://iot.weixin.qq.com/wiki/new/index.html?page=4-3")

[2] [https://github.com/paypal/gatt](https://github.com/paypal/gatt)

_原创文章，转发请注明出处。蜗窝科技，[www.wowotech.net](http://www.wowotech.net/bluetooth/weixin_ble_1.html)。_

标签: [蓝牙](http://www.wowotech.net/tag/%E8%93%9D%E7%89%99) [Bluetooth](http://www.wowotech.net/tag/Bluetooth) [BLE](http://www.wowotech.net/tag/BLE) [golang](http://www.wowotech.net/tag/golang) [微信](http://www.wowotech.net/tag/%E5%BE%AE%E4%BF%A1) [airsync](http://www.wowotech.net/tag/airsync) [iot](http://www.wowotech.net/tag/iot)

---

« [Linux调度器：进程优先级](http://www.wowotech.net/process_management/process-priority.html) | [Linux调度器：用户空间接口](http://www.wowotech.net/process_management/scheduler-API.html)»

**评论：**

**骆凉皮**  
2018-04-01 20:12

小白路过，膜拜大神

[回复](http://www.wowotech.net/bluetooth/weixin_ble_1.html#comment-6631)

**single-coffee**  
2017-12-14 15:29

你好窝窝.  
我使用 bluez-5.47 在一个raspberry3b上.然后我想用我的Android 手机连接这个ble.我在我的raspberry 3b上面使用了  blue-5.47/test/example-gatt-server .可是 我使用google play的ble测试应用."ble scanner" 看不到 example-gatt-server  创建的服务.但是我使用我同事的iphone手机可以连接.可以看到服务.    
窝窝.有空帮我看看  ...  ... 万分感谢.好人一生平安  
测试手机  iphone 6,8  
         小米4 6 max2

[回复](http://www.wowotech.net/bluetooth/weixin_ble_1.html#comment-6363)

**[wowo](http://www.wowotech.net/)**  
2017-12-14 18:15

@single-coffee：android对ble的支持不是特别好，有一些奇奇怪怪的问题很正常，你可以多试试几个ble测试应用。

[回复](http://www.wowotech.net/bluetooth/weixin_ble_1.html#comment-6365)

**cuiran**  
2017-10-27 14:37

"接下来要连接设备。对iOS来说，在设置界面无法主动连接BLE设备，必须有应用程序自行连接，这里以微信运动为例，搜索并连接刚才授权的那个设备，即可看到计步数据的更新。步骤如下面图片（不再解释了，大家可以自己试试）："  对Android 设备来说，设置界面能主动连接BLE设备吗?

[回复](http://www.wowotech.net/bluetooth/weixin_ble_1.html#comment-6147)

**[wowo](http://www.wowotech.net/)**  
2017-10-27 16:55

@cuiran：不好说，要看手机的实现。不过可以通过ble测试工具来测试，例如iOS下的LighBlue。

[回复](http://www.wowotech.net/bluetooth/weixin_ble_1.html#comment-6148)

**Reashy**  
2017-07-14 10:31

使用bluez的测试代码（bluez-5.37/test/example-gatt-server），这个例子在那里有？

[回复](http://www.wowotech.net/bluetooth/weixin_ble_1.html#comment-5804)

**[wowo](http://www.wowotech.net/)**  
2017-07-14 10:47

@Reashy：你是指源码吗？bluez软件包里面有。不过我也没有用过。

[回复](http://www.wowotech.net/bluetooth/weixin_ble_1.html#comment-5806)

**mossad**  
2017-06-19 10:48

请问支持蓝牙4.0的笔记本可以在虚拟机上完成嘛，还是说要额外的有一个usb ble dongle  
  
我的环境是 ubuntu 12.04 ，bluez 5.37 ，go 1.8.3  
  
按照步骤编译完成 执行 sudo ./weixin 却提示，按说笔记本蓝牙适配器是dell 1705 bluetooth，应该支持bluetooth 4.0，旧版本的bluez也被我purge后重新编译安装了，不知道还有什么需要升级还是有啥其他的问题 - -：  
    2017/06/19 10:32:11 dev: hci0 does not support LE  
    2017/06/19 10:32:11 Failed to open device, err: no supported devices available  
  
执行 hciconfig -a：  
  
  hci0:    Type: BR/EDR  Bus: USB  
    BD Address: 9C:D2:1E:7A:8A:6C  ACL MTU: 8192:128  SCO MTU: 64:128  
    UP RUNNING  
    RX bytes:5318 acl:0 sco:0 events:97 errors:0  
    TX bytes:364 acl:0 sco:0 commands:56 errors:0  
    Features: 0xff 0xff 0x8f 0xfe 0x83 0xe1 0x08 0x80  
    Packet type: DM1 DM3 DM5 DH1 DH3 DH5 HV1 HV2 HV3  
    Link policy: RSWITCH HOLD SNIFF PARK  
    Link mode: SLAVE ACCEPT  
    Name: 'Virtual Bluetooth Adapter'  
    Class: 0x000100  
    Service Classes: Unspecified  
    Device Class: Computer, Uncategorized  
    HCI Version: 2.1 (0x4)  Revision: 0x100  
    LMP Version: 2.1 (0x4)  Subversion: 0x100  
    Manufacturer: not assigned (6502)  
  
我是小白，还望 wowo 有空可以指点一下，谢谢啦

[回复](http://www.wowotech.net/bluetooth/weixin_ble_1.html#comment-5694)

**mossad**  
2017-06-19 17:03

@mossad：实在没找到解决办法,于是换用 ubuntu 14.04 , 并把蓝牙设备加载到虚拟机中,安装配置好golang 1.8.1 后,发现 gatttool 存在,就试着用自带的bluez,发现 $ sudo ./weixin 可以成功运行,后面的步骤按照 wowo 大大的说明都成功了.  
  
唯独在最后的时候,微信运动中添加数据来源时,无法搜索到设备,但是AirSyncDebugger2.3.0可以检测到,计步测试的调试信息也未见error.  
  
使用手机红米 note2 和 平板 nexus 7都是这样.   - -.  
  
难过...T_T

[回复](http://www.wowotech.net/bluetooth/weixin_ble_1.html#comment-5698)

**[wowo](http://www.wowotech.net/)**  
2017-06-19 17:13

@mossad：AirSyncDebugger2.3.0能否成功的，再检查一下微信授权的部分，一般都是授权的问题。

[回复](http://www.wowotech.net/bluetooth/weixin_ble_1.html#comment-5700)

**mossad**  
2017-06-19 18:11

@wowo：确实是授权的问题，重新再添加一个设备后就可以扫描到了，谢谢 wowo 大大～

[回复](http://www.wowotech.net/bluetooth/weixin_ble_1.html#comment-5702)

**饭吃到一半**  
2017-04-11 18:20

经测试好用，但是微信运动里一堆数据来源怎么删除掉，取消关注测试公众号也没用啊。。。

[回复](http://www.wowotech.net/bluetooth/weixin_ble_1.html#comment-5457)

**[wowo](http://www.wowotech.net/)**  
2017-04-11 18:46

@饭吃到一半：我也不知道怎么删，这是微信运动设计的问题。

[回复](http://www.wowotech.net/bluetooth/weixin_ble_1.html#comment-5459)

**bbigcd**  
2017-04-14 11:14

@wowo：添加过设备之后，在微信的我->设置里面的隐私与通用之间会出现一个“设备”里面记录的是添加过的设备

[回复](http://www.wowotech.net/bluetooth/weixin_ble_1.html#comment-5469)

**徐君**  
2017-03-14 16:52

已经检测出来了，谢谢！

[回复](http://www.wowotech.net/bluetooth/weixin_ble_1.html#comment-5315)

**徐君**  
2017-03-14 16:34

报错信息是：  
result=false,Has no 0xfee7 or standard service in broadcast record!  
广播包：02 01 06 0D FF 00 0A 12 34 56 78 9A BC 03 03 E7 FE 0A 09 4D 42 31 32 2D 31 32 33 34 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00

[回复](http://www.wowotech.net/bluetooth/weixin_ble_1.html#comment-5314)

**[wowo](http://www.wowotech.net/)**  
2017-03-14 16:54

@徐君：我们可以慢慢解析这个广播数据，看看怎么回事（参考如下的链接）：  
http://www.wowotech.net/bluetooth/ble_broadcast.html  
http://iot.weixin.qq.com/wiki/new/index.html?page=4-3  
https://www.bluetooth.com/specifications/assigned-numbers/generic-access-profile  
  
02 01 06（OK）  
    Data的长度是02；Data是01 06；AD Type是01（Flags）；AD Data是06，表明支持General Discoverable Mode、不支持BR/EDR。  
  
0D FF 00 0A 12 34 56 78 9A BC 03 03 E7 FE（0D这个长度有问题！！）  
    Data的长度是0D，Data是FF 00 0A 12 34 56 78 9A BC 03 03 E7 FE；  
    AD Type是FF（厂商自定义），AD Data是00 0A 12 34 56 78 9A BC 03 03 E7 FE；（参考http://iot.weixin.qq.com/wiki/new/index.html?page=4-3）  
        00 0A：厂商ID  
        12 34 56 78 9A BC：MAC地址  
          
        03 03 E7 FE（service信息，但是service的长度包在0D这个长度里面了）。  
  
0A 09 4D 42 31 32 2D 31 32 33 34（OK）  
    Data的长度是0A，Data是09 4D 42 31 32 2D 31 32 33 34；  
    AD Type是09（Complete Local Name），AD Data是4D 42 31 32 2D 31 32 33 34（MB12-1234）

[回复](http://www.wowotech.net/bluetooth/weixin_ble_1.html#comment-5316)

**徐君**  
2017-03-14 14:46

可否发个邮件给我，想请教一下这方面的问题。  
517204655@qq.com

[回复](http://www.wowotech.net/bluetooth/weixin_ble_1.html#comment-5311)

**[wowo](http://www.wowotech.net/)**  
2017-03-14 16:17

@徐君：这里有联系方式：http://www.wowotech.net/contact_us.html

[回复](http://www.wowotech.net/bluetooth/weixin_ble_1.html#comment-5313)

**徐君**  
2017-03-14 14:43

你好，我现在也想玩一下微信蓝牙，但是我的第一个蓝牙数据广播就没办法通过检测，所以想请教一下。

[回复](http://www.wowotech.net/bluetooth/weixin_ble_1.html#comment-5310)

**[wowo](http://www.wowotech.net/)**  
2017-03-14 16:17

@徐君：可以看看广播数据是什么，对一下微信的协议，看看少哪些东西（一般是MAC地址不对）。或者点一下AirSyncDebugger2.3.0界面右边的问号，看看原因。

[回复](http://www.wowotech.net/bluetooth/weixin_ble_1.html#comment-5312)

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
    
    - [建立讨论区的原因](http://www.wowotech.net/73.html)
    - [通过点亮LED的方法调试嵌入式代码](http://www.wowotech.net/soft/debug_using_led.html)
    - [Linux内核同步机制之（四）：spin lock](http://www.wowotech.net/kernel_synchronization/spinlock.html)
    - [DMB DSB ISB以及SMP CPU 乱序](http://www.wowotech.net/77.html)
    - [串口通信技术浅析](http://www.wowotech.net/basic_tech/serial_intro.html)
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