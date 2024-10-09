一口Linux

_2024年09月11日 20:29_ _河北_

The following article is from 小麦大叔 Author 菜刀和小麦

**小麦大叔**.

分享嵌入式技术

\](https://mp.weixin.qq.com/s?\_\_biz=MzUxMjEyNDgyNw==&mid=2247521395&idx=1&sn=edcf78ad6f55049f0456a981b57b5fbb&chksm=f83afb08f4bab2f9de35446f7adfc90282e6064d90afe1b7b4017a0e7c4aaa8f2e6991628b18&mpshare=1&scene=24&srcid=0912siBXMx2Le01uiJ6Un5vX&sharer_shareinfo=95417ed2f6e7ea07ca18b30209e7c342&sharer_shareinfo_first=95417ed2f6e7ea07ca18b30209e7c342&key=daf9bdc5abc4e8d06d2437884305d14825034f27c1f0c0fd606904a10c616e1e202be04121b2e34de4aec9de81da38e269bec38cf4bb63de07c58729bc50fb39c728c41088a0b480fce84e31bb4c905cf47abb51da522947b4564779c467324c630787a88042c1e51f81de5f32d3d9522d055d37a2bba989fd6f530942e2287f&ascene=14&uin=MTEwNTU1MjgwMw%3D%3D&devicetype=iMac+MacBookAir10%2C1+OSX+OSX+15.0+build(24A335)&version=13080810&nettype=WIFI&lang=en&session_us=gh_fc2c47bdbd29&countrycode=CN&fontScale=100&exportkey=n_ChQIAhIQgXqeI7jxZx82RNRjTkatBBKTAgIE97dBBAEAAAAAAOyqDTkGt9YAAAAOpnltbLcz9gKNyK89dVj0BJbpC8sc3x0%2FPHbthRotqZ8SmZbiTd%2Fpkp4PvkR1xnBu5u1FZQNCGXBDJI2%2FddTbjjAh8%2FWZIA3phPkVlz3oA1K%2FH2LIXwQmzDOIdCpmmZ3Y2YOiWsWm55mw85cgNSF6szhsBkjfwbg8blH6ck%2F9gc4svJaS9y%2Fewd0sqYIfh7fbjoI7ibW81B97VQlpgKPEBxkKIHrEls4q8%2F2JRNnU6FcU5xPd4%2BWzdpghq3UTXYwJRuuEP%2B5rIYxYNoyano%2BClNYJlsBnP63rA%2BZj%2BnpzhIE4h3n626r2ICjGd3KxRcG5oAFDS7Cxvcv1DIE5&acctmode=0&pass_ticket=OdFmDwMrcsilapaYBmb7zUeBEkV7Fll82J2HuFQcbIbGTlEz2GccePvnXxcyEsKF&wx_header=0#)

大家好，我是小麦，最近做了一个物联网的项目，顺便总结一下MQTT协议。大家都知道，MQTT协议在物联网中很常用，如果你对此还不是很了解，相信这篇文章可以带你入门。

- mqtt协议

- 1 MQTT协议特点

- 发布和订阅

- QoS（Quality of Service levels）

- 2 MQTT 数据包结构

- 2.1 MQTT固定头

- 2.2 MQTT可变头 / Variable header

- 2.3 Payload消息体

- 3 环境搭建

- 3.1 MQTT服务器搭建

- 3.2 MQTT Client

- 4 总结

## mqtt协议

MQTT（Message Queuing Telemetry Transport，消息队列遥测传输协议），是一种基于`发布/订阅`（`publish/subscribe`）模式的“轻量级”通讯协议，该协议构建于TCP/IP协议上，由IBM在1999年发布。

MQTT最大优点在于，**用极少的代码和有限的带宽，为连接远程设备提供实时可靠的消息服务**。

作为一种低开销、低带宽占用的即时通讯协议，使其在物联网、小型设备、移动应用等方面有较广泛的应用。

## 1 MQTT协议特点

MQTT是一个基于**客户端-服务器**的消息发布/订阅传输协议。

MQTT协议是轻量、简单、开放和易于实现的，这些特点使它适用范围非常广泛。在很多情况下，包括受限的环境中，如：机器与机器（M2M）通信和物联网（IoT）。

其在，通过卫星链路通信传感器、偶尔拨号的医疗设备、智能家居、及一些小型化设备中已广泛使用。

MQTT协议当前版本为，2014年发布的MQTT v3.1.1。除标准版外，还有一个简化版`MQTT-SN`，该协议主要针对嵌入式设备，这些设备一般工作于TCP/IP网络，如：ZigBee。

MQTT 与 HTTP 一样，MQTT 运行在传输控制协议/互联网协议 (TCP/IP) 堆栈之上。

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

MQTT OSI

### 发布和订阅

`MQTT`使用的发布/订阅消息模式，它提供了一对多的消息分发机制，从而实现与应用程序的解耦。

这是一种消息传递模式，**消息不是直接从发送器发送到接收器**（即点对点），而是由`MQTT server`（或称为 MQTT Broker）分发的。

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**MQTT 服务器是发布-订阅架构的核心**。

它可以非常简单地在Raspberry Pi或NAS等单板计算机上实现，当然也可以在大型机或 Internet 服务器上实现。

服务器分发消息，因此必须是发布者，但绝不是订阅者！

客户端可以发布消息（发送方）、订阅消息（接收方）或两者兼而有之。

客户端（也称为节点）是一种智能设备，如微控制器或具有 TCP/IP 堆栈和实现 MQTT 协议的软件的计算机。

消息在允许过滤的主题下发布。主题是分层划分的 UTF-8 字符串。不同的主题级别用斜杠`/`作为分隔符号。

我们来看看下面的设置。

- 光伏发电站是发布者（`Publisher`）。

- 主要主题（`Topic`）级别是`"PV"`，这个工厂发布两个子级别`"sunshine"`和`"data"`；

- `"PV/sunshine"`是一个布尔值（true/fault，也可以是 1/0），充电站需要它来知道是否应该装载电动汽车（仅在阳光普照时 :)）。

- 充电站（EVSE）是订阅者，订阅`"PV/sunshine"`从服务器获取信息。

- `"PV/data"` 另一方面，以 kW 为单位传输工厂产生的瞬时功率，并且该主题可以例如通过计算机或平板电脑订阅，以生成一天内传输功率的图表。

这就是一个简单的MQTT的应用场景，具体如下图所示；

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

MQTT 发布和订阅

### QoS（Quality of Service levels）

服务质量是 MQTT 的一个重要特性。当我们使用 TCP/IP 时，连接已经在一定程度上受到保护。但是在无线网络中，中断和干扰很频繁，MQTT 在这里帮助避免信息丢失及其服务质量水平。这些级别在发布时使用。如果客户端发布到 MQTT 服务器，则客户端将是发送者，MQTT 服务器将是接收者。当MQTT服务器向客户端发布消息时，服务器是发送者，客户端是接收者。

**QoS  0**

这一级别会发生消息丢失或重复，消息发布依赖于底层TCP/IP网络。即：\<=1

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**QoS  1**

QoS 1 承诺消息将至少传送一次给订阅者。

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**QoS  2**

使用 QoS 2，我们保证消息仅传送到目的地一次。为此，带有唯一消息 ID 的消息会存储两次，首先来自发送者，然后是接收者。QoS 级别 2 在网络中具有最高的开销，因为在发送方和接收方之间需要两个流。

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

## 2 MQTT 数据包结构

- `固定头（Fixed header）`，存在于所有`MQTT`数据包中，表示数据包类型及数据包的分组类标识；

- `可变头（Variable header）`，存在于部分`MQTT`数据包中，数据包类型决定了可变头是否存在及其具体内容；

- `消息体（Payload）`，存在于部分`MQTT`数据包中，表示客户端收到的具体内容；

整体MQTT的消息格式如下图所示；

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

### 2.1 `MQTT`固定头

`固定头`存在于所有`MQTT`数据包中，其结构如下：

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

下面简单分析一下固定头的消息格式；

#### `MQTT`消息类型 / message type

\*\*位置：\*\*byte 1, bits 7-4。

4位的无符号值，类型如下：

|名称|值|流方向|描述|
|---|---|---|---|
|Reserved|0|不可用|保留位|
|CONNECT|1|客户端到服务器|客户端请求连接到服务器|
|CONNACK|2|服务器到客户端|连接确认|
|PUBLISH|3|双向|发布消息|
|PUBACK|4|双向|发布确认|
|PUBREC|5|双向|发布收到（保证第1部分到达）|
|PUBREL|6|双向|发布释放（保证第2部分到达）|
|PUBCOMP|7|双向|发布完成（保证第3部分到达）|
|SUBSCRIBE|8|客户端到服务器|客户端请求订阅|
|SUBACK|9|服务器到客户端|订阅确认|
|UNSUBSCRIBE|10|客户端到服务器|请求取消订阅|
|UNSUBACK|11|服务器到客户端|取消订阅确认|
|PINGREQ|12|客户端到服务器|PING请求|
|PINGRESP|13|服务器到客户端|PING应答|
|DISCONNECT|14|客户端到服务器|中断连接|
|Reserved|15|不可用|保留位|

#### 标识位 / DUP

\*\*位置：\*\*byte 1, bits 3-0。

在不使用标识位的消息类型中，标识位被作为保留位。如果收到无效的标志时，接收端必须关闭网络连接：

|数据包|标识位|Bit 3|Bit 2|Bit 1|Bit 0|
|---|---|---|---|---|---|
|CONNECT|保留位|0|0|0|0|
|CONNACK|保留位|0|0|0|0|
|PUBLISH|MQTT 3.1.1使用|DUP1|QoS2|QoS2|RETAIN3|
|PUBACK|保留位|0|0|0|0|
|PUBREC|保留位|0|0|0|0|
|PUBREL|保留位|0|0|0|0|
|PUBCOMP|保留位|0|0|0|0|
|SUBSCRIBE|保留位|0|0|0|0|
|SUBACK|保留位|0|0|0|0|
|UNSUBSCRIBE|保留位|0|0|0|0|
|UNSUBACK|保留位|0|0|0|0|
|PINGREQ|保留位|0|0|0|0|
|PINGRESP|保留位|0|0|0|0|
|DISCONNECT|保留位|0|0|0|0|

- `DUP`：发布消息的副本。用来在保证消息的可靠传输，如果设置为 1，则在下面的变长中增加MessageId，并且需要回复确认，以保证消息传输完成，但不能用于检测消息重复发送。

- `QoS`发布消息的服务质量（前面已经做过介绍），即：保证消息传递的次数

- `00`：最多一次，即：\<=1

- `01`：至少一次，即：>=1

- `10`：一次，即：=1

- `11`：预留

- `RETAIN`：发布保留标识，表示服务器要保留这次推送的信息，如果有新的订阅者出现，就把这消息推送给它，如果设有那么推送至当前订阅者后释放。

#### 剩余长度（Remaining Length）

位置：byte 1。

固定头的第二字节用来保存变长头部和消息体的总大小的，但不是直接保存的。这一字节是可以扩展，其保存机制，前7位用于保存长度，后一部用做标识。当最后一位为 1时，表示长度不足，需要使用二个字节继续保存。例如：计算出后面的大小为0

### 2.2 `MQTT`可变头 / Variable header

`MQTT`数据包中包含一个可变头，它驻位于固定的头和负载之间。可变头的内容因数据包类型而不同，较常的应用是做为包的标识：

|Bit|7  — 0|
|---|---|
|byte 1|包标签符（MSB）|
|byte 2…|包标签符（LSB）|

很多类型数据包中都包括一个2字节的数据包标识字段，这些类型的包有：

PUBLISH (QoS > 0)、PUBACK、PUBREC、PUBREL、PUBCOMP、

SUBSCRIBE、SUBACK、UNSUBSCRIBE、UNSUBACK

### 2.3 `Payload`消息体

`Payload`消息体是`MQTT`数据包的第三部分，CONNECT、SUBSCRIBE、SUBACK、UNSUBSCRIBE四种类型的消息 有消息体：

- `CONNECT`，消息体内容主要是：客户端的ClientID、订阅的Topic、Message以及用户名和密码

- `SUBSCRIBE`，消息体内容是一系列的要订阅的主题以及`QoS`。

- `SUBACK`，消息体内容是服务器对于`SUBSCRIBE`所申请的主题及`QoS`进行确认和回复。

- `UNSUBSCRIBE`，消息体内容是要订阅的主题。

## 3 环境搭建

介绍完基础理论部分，下面在Windows平台上搭建一个简单的MQTT应用，进行简单的应用，整体架构如下图所示；

\[外链图片转存失败,源站可能有防盗链机制,建议将图片保存下来直接上传(img-ScRucIVO-1625480723109)(架构图.png)\]

### 3.1 MQTT服务器搭建

目前MQTT代理的主流平台有下面几个：

- Mosquitto：https://mosquitto.org/

- VerneMQ：https://vernemq.com/

- EMQTT：http://emqtt.io/

本文将使用 Mosquitoo 进行测试，进入到安装页面，下载自己电脑的系统所适配的程序；

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

下载页面

安装成功之后，进入到安装路径下，找到`mosquitto.exe`；

\[外链图片转存失败,源站可能有防盗链机制,建议将图片保存下来直接上传(img-YXZupgOv-1625480723111)(image-20210705171401654.png)\]

按住`Shift`，右键鼠标点击空白处，然后打开`Powershell`，正常打开一个终端软件即可；

- 输入`./mosquitto.exe -h` 可以查看相应的帮助；

- 输入`./mosquitto.exe -p 10086`，就开启了MQTT服务，监听的地址是`127.0.0.1`，端口是`10086`；

具体如下图所示；

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

### 3.2 MQTT Client

服务器搭建好了，下面就是开启客户端，进行发布和订阅，这样就可以传输相应的消息。

这里我使用的是自己编译了一个`QT mqtt client` 程序，是基于Qt的官方库进行编译的，下面打开这个软件，下一期简单介绍一下如何完成这个客户端，并设置好相应参数：

- 地址：`127.0.0.1`

- 端口：`10086`

然后订阅主题，就可以互相发送数据了，具体如下图所示；

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

结合前面的图片来看，整体的架构如下所示；

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

## 4 总结

本文简单介绍了MQTT协议的工作原理，以及相应的协议格式，简单介绍了协议的一些细节，具体举出了相应的应用场景，作者水平和能力有限，文中难免存在错误和纰漏，请大佬不吝赐教。

本期就到此结束了，我是小麦，我们下期再见。

end

**一口Linux**

**关注，回复【****1024****】海量Linux资料赠送**

**精彩文章合集**

文章推荐

☞【专辑】[ARM](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=MzUxMjEyNDgyNw==&action=getalbum&album_id=1614665559315382276#wechat_redirect)

☞【专辑】[粉丝问答](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=MzUxMjEyNDgyNw==&action=getalbum&album_id=1629876820810465283#wechat_redirect)

☞【专辑】[所有原创](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=MzUxMjEyNDgyNw==&action=getalbum&album_id=1479949091139813387#wechat_redirect)

☞【专辑】[linux](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=MzUxMjEyNDgyNw==&action=getalbum&album_id=1507350615537025026#wechat_redirect)入门

☞【专辑】[计算机网络](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=MzUxMjEyNDgyNw==&action=getalbum&album_id=1598710257097179137#wechat_redirect)

☞【专辑】[Linux驱动](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=MzUxMjEyNDgyNw==&action=getalbum&album_id=1502410824114569216#wechat_redirect)

☞【干货】[嵌入式驱动工程师学习路线](http://mp.weixin.qq.com/s?__biz=MzUxMjEyNDgyNw==&mid=2247496985&idx=1&sn=c3d5e8406ff328be92d3ef4814108cd0&chksm=f96b87edce1c0efb6f60a6a0088c714087e4a908db1938c44251cdd5175462160e26d50baf24&scene=21#wechat_redirect)

☞【干货】[Linux嵌入式所有知识点-思维导图](http://mp.weixin.qq.com/s?__biz=MzUxMjEyNDgyNw==&mid=2247497822&idx=1&sn=1e2aed9294f95ae43b1ad057c2262980&chksm=f96b8aaace1c03bc2c9b0c3a94c023062f15e9ccdea20cd76fd38967b8f2eaad4dfd28e1ca3d&scene=21#wechat_redirect)

Reads 5372

​

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/icRxcMBeJfc8535w2vKlsLPf5hwdMjpYrzuVCHx0rcQmvv8rYqTFtIyic5qErtciaibqaIOWgeKkDsOMeae4HciaUaw/300?wx_fmt=png&wxfrom=18)

一口Linux

8252638

15

Comment

**留言 15**

- 银鹿望月🦌![😱](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

  四川9/13

  Like

  还是不错的，我用麦哥介绍的方法启动MQTT服务器后，用Netassist新版的MQTT Client功能测试了一下，OK的。

  Pinned

  一口Linux

  Author9/13

  Like

  ![[强]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![[强]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![[强]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![[强]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![[强]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![[强]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)学以致用！！

- 让我聆听你的青春

  广东9/11

  Like

  有源码么？

  Pinned

  聪明男孩不绝顶

  广东9/12

  Like

  mqtt好像是开源的，lwip去实现，好像大部分物联网的都是基于这个实现

  让我聆听你的青春

  广东9/13

  Like

  回复 **聪明男孩不绝顶**：问题是不知道去哪里拿源码

  2条回复

- 光梭

  广东9/11

  Like2

  麦哥的这篇文章还是挺详细的

  一口Linux

  Author9/11

  Like2

  必须的！！

  Louie

  上海9/12

  Like

  回复 **一口Linux**：一口出品，从来没有让粉丝失望过

- Glückskind

  9/11

  Like1

  明天好好研读一下，先睡了![[Emm]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

  一口Linux

  Author9/11

  Like1

  别睡，继续卷！！

- y

  安徽9/14

  Like

  哈喽，我这边是做公众号广告投放的，想在咱家公众号上面投放广告，可以给个联系方式吗，或者这边添加我微信也可以ds9211026

- 小豆泥

  9/12

  Like

  帅

- ZXJ

  浙江9/12

  Like

  读完了，感谢

已无更多数据
