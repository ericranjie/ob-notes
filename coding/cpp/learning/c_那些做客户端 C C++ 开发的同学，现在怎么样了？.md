Original 张小方 CppGuide
 _2022年03月08日 14:30_

我读研的时候，沉迷于 Windows 编程而不能自拔，那个时候也和楼主有一样的困惑。毕业的时候找工作，非 Windows C/C++ 岗位不去，因为技术功底比较好，很快就成为客户端负责人。

为了说明问题，我来给你讲个案例吧。

## 一、如何开发一款类电驴客户端？

假设我们现在要开发一个类似电驴这样的软件，软件界面如下图：

![Image](https://mmbiz.qpic.cn/mmbiz_jpg/ic8RqseyjxMPO6P0hhKbZUn85JnvzicQz9O3vsVib3Ay6kooHYP6wDiceC0956CBIa62CPwEl5j8k604j72bWvWW1Q/640?wx_fmt=jpeg&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

![Image](https://mmbiz.qpic.cn/mmbiz_jpg/ic8RqseyjxMPO6P0hhKbZUn85JnvzicQz951IAmiaKNsGmibc1a65uuiahgeYNbxUWTWa7h4TGIPNxwXpO2RRn49PSQ/640?wx_fmt=jpeg&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)


如上图所示，假设操作系统选择 Windows，使用语言使用 C++，这就要求您必须熟悉 C++ 常用的语法，如果还不熟悉，就需要补充这方面的知识。

在熟悉 C++ 语法的前提下，从这款产品实现技术来看，我们的目标产品分为 UI 和网络通信部分。下面将详细介绍这两部分。
### UI 部分

对于 UI 部分，我们的认识是，这里需要使用 Windows 的窗口技术。可以直接使用原生的 Win 32 API 来制作自己的界面库，也可以选择一些熟悉的界面框架，如 MFC、WTL、Duilib、wxWidgets 等。无论您是在阅读别人的项目还是需要自己开发这样的项目，在确定了这款软件使用的 UI 库（或者使用原生 Win 32 API），您就需要对 Windows 的窗口、对话框、消息产生、派发与处理机制进行了解。同样的道理，如果不熟悉您需要补充相关的知识（关于这一点，下文不再赘述）。

接着，根据上图中的软件功能，大致分为三大模块，即资源、下载和分享。这三大块是可以使用一个 Windows Tab 控件去组织，这个时候您需要了解 Windows Tab 控件的特性。

对于资源模块，本质上是一个窗口中嵌入了一个浏览器控件（WebBrowser 控件），那么您需要了解这一个功能点的相关知识。当用户点击了某个列表中某个具体的资源，可以对其进行下载。这就又涉及到 WebBrowser 控件与 C++ 宿主程序的交互了，那么如何实现呢？可以选择使用 ActiveX 技术，也可以使用 JavaScript 与 C++ 交互技术。

再来看下载模块，当产生一个下载操作时，界面上会产生以下下载列表，每个列表项会实时显示下载进度、下载速率等参数，同时正在下载的项目也可以被暂停、停止和删除。那么这又涉及到 ListView 控件的相关功能，以及 ListView 如何与后台网络通信的逻辑交互。

分享模块是将本地资源分享到服务器或者给其他用户。界面左侧是文件系统的一个快照，那么这又涉及到如何遍历文件系统（了解枚举文件系统的 API），右侧也是一个 ListView 控件，这里不再赘述。

### 网络通信部分

网络通信部分，主要有两大块，第一个是程序启动时，与服务端的交互；第二个就是文件下载与分享的 P2P 网络。您在阅读或开发的过程中，如果对这些技术比较陌生，您需要补充这些知识，具体的就是 Socket 的各种 API 函数，以及基于这些 API 逻辑的组合。当然可能也会用到操作系统平台所特有的网络 API 函数，如 WSAAsyncSelect 网络模型。

网络编程对于已经工作了的或者时间不是很充裕的同学来说，如果想入门或者上手，不建议去读一些大部头的图书，容易坚持不下，最后放弃。

建议找一些通俗易懂又可快速实践的书，这里推荐韩国人尹圣雨写的《**TCP/IP 网络编程**》这本书，这本书尤其适合非科班出身或者网络编程小白的同学，常见的 socket API 以及网络通信模式都有介绍，且同时包括 Linux 和 Windows 两个操作系统平台。

另外一点，网络通信部分如何与 UI 部分进行数据交换，是使用队列？全局变量？或者相应的 Windows 操作平台提供的特殊通信技术，如 PostMessage 函数、管道？如果使用队列，多线程之间如何保持资源的一致性和解决资源竞态，使用 Event、CriticalSection、Mutex、Semaphore 等？

当然，笔者这里只列举了这个软件的主干部分，还有许多方方面面的细节需要考虑。这就需要读者根据自己的情况，斟酌和筛选了。您想达到什么目的，就要去学习和研究相关的代码。看懂了吗？一款 Windows 软件的生产等于下面的公式：

> **一款 C++ 软件 = C++ 语法 + 操作系统 API 函数调用**

## 二、为什么你学 Windows 编程感觉这么枯燥或者痛苦？

痛苦的原因大致有两点：

1. 学不得法
    

即未掌握 Windows 程序的规律和编码习惯，觉得 Windows 开发是一个个孤零零的 API 函数，这些 API 函数固然重要，但是他们都是成体系的，你需要结合 Windows 的各个知识点学习。

2. 没有成就感
    

我们做一件事，如果一段时间感觉不到成就感，我们也容易放弃。那么如何寻找成就感呢？如果自己能看懂甚至编写一些有意义的 Windows 软件，那肯定会对自己信心大增。

下面我将从解决以上两点来介绍。

## 三、Windows 编程的特点（规律）

### 3.1 严谨的接口设计

按操作系统的技术栈发展来看，相比较类 unix 操作系统比较短小精悍的风格，Windows 操作系统提供的函数接口以及各种函数参数的命名都是很清晰而且容易看懂的。虽然古怪的匈牙利命名法（下文将介绍）让 Windows 程序看起来有点“中世纪风格”，但另一方面增加了 Windows 程序的可读性和可理解性，后来者不断模仿。

另外一点就是 Windows 提供的函数名称、结构体类型风格都非常统一，诸如 CreateProcess、CreateThread、CreateMutex、CreateSemaphore 等。

当然也有例外，比如堆创建函数 HeapCreate，这个据《**The Old New Things**》（中文名《**Windows 编程启示录**》）的作者 Raymond Chen 说，当时设计这个堆接口 API 是另外一个部门，所以风格出现了不一致- -！）。

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

Windows 扩展的函数和结构体在原有的基础上使用 Ex，例如从 CreateWindow 到CreateWindowEx，RegisterClass 到 RegisterClassEx，WNDCLASS 到 WNDCLASSEX。而且 Windows 在新旧系统兼容性方面做的也非常优秀，即使在最新的 Windows 操作系统上使用很老的 Windows API 仍然没什么问题，虽然很多 API 或者做法不推荐在新版系统中使用，但是你使用仍然不会有什么大问题。反观，现在的一些操作系统比如 Android，低版本使用的 API 一个个被弃用，原来老的做法在新的系统上导致程序 crash 掉。

再者一个特色，就是 Windows 将大多数资源抽象成句柄（HANDLE），例如 socket、进程对象、线程对象、画笔画刷对象，甚至连像 Windows 服务这样的东西也是对象。这种抽象带来的好处就是可以以同一套逻辑去处理，比如操作一个对象必须先打开或者创建，然后使用，使用完了之后然后关闭。句柄只是内部资源的引用，通过句柄操作资源只能按照系统规定对资源特定字段进行查询和修改，保证了安全。操作部分资源类型的句柄时，如果权限不足时，会操作报错，不会因为越权而带来安全隐患。

### 3.2 匈牙利命名法

在驼峰式（例如 myBestFriend）、帕斯卡（例如 MyBestFriend ）或者连字符式（例如my_best_friend ）等诸多命名法当中，匈牙利命名法是 Windows 程序的一大特色。

虽然显得很奇怪，但是带来的便捷性和可读性不得不提。匈牙利命名法是比尔盖茨雇佣的第一代程序员**查尔斯·西蒙尼**发明的（Excel 的主要开发者），这是一个传奇性的人物。匈牙利命名法的主要思想是给程序变量加上它的类型信息，例如一个整形变量表示数值，可以叫 nNum 或 iNum，表示屏幕尺寸的叫 cxScreen、cyScreen，以 NULL 结束的字符串指针叫 pszStr（sz 的意思是 String terminated with Zero，表示以 \0 结束的字符串）。当我们在代码中看到这样的变量时我们无需查看其类型定义。虽然有人说这种命名法已经过时，在后来的很多系统设计上，很多地方可以看到匈牙利命名法。这种命名法虽然看起来很拖沓，但是表达意思却是非常清晰，尤其是在阅读别人的项目代码或者维护一些旧的项目时，别人能读懂是其他工作的前提。所以熟悉 Windows 匈牙利命名法是学好 Windows 编程的一个关键点。

### 3.3 消息机制

之所以把 Windows 消息机制单独列出来是因为基本上可以认为它是以后所有的操作系统界面的模型的滥觞，同时也是我们普通程序开发者应该学习和模仿的典范。它本质上就是一个消息队列， Windows 消息队列的实现可以参考开源版本的”Windows”—— ReactOS。这里要提一下的是，消息队列是软件产品中非常常用的结构，所以如何设计出一个好的消息队列属于开发者的基本功了。Windows 可以参考，现在流行的很多专门的消息队列框架，如 RabbitMQ，都是不错的学习资料。

ReactOS 号称开源的 Windows，其开发团队不仅保持对外的操作系统 API 与官方 Windows 一模一样（名称和特性均一样），同时内核实现也力求和官方 Windows 一模一样。ReactOS 官网地址：Front Page，《**Windows 内核情景分析**》（分为上下两册）一书通过 ReactOS 的源代码详细地介绍了 Windows 的各种内核实现机制，非常好的书，给大家推荐一下。书的作者毛德操老师也是操作系统领域的泰山北斗。

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

### 3.4 统一的用户界面使用习惯

Windows 程序除了一些自绘的界面以外，大多数界面风格、菜单位置、使用习惯等等都是统一的，例如菜单项后面有⋯（三个点）的，点击该菜单项一般都会弹出一个对话框；对于文本的编辑操作（例如复制、剪切、粘贴）等等一般统一放在”编辑”菜单下面。这也是 Windows 和 Linux 的不同，Windows 的哲学是你不会操作，Windows 教你如何操作；而 Linux 是假设你会操作。所以，你在设计Windows UI 交互的时候千万不要违背这种微软帮用户培养出来的多年用户习惯，否则很可能你的用户因为觉得产品操作麻烦而放弃。

一些应用软件为了追求一些特殊的界面效果（比如游戏界面），需要显示出不同于传统的 Windows 程序界面的样子。由于 Windows 操作系统提供了最大化的界面自绘机制，所以市面上出现了很多开源的或非开源的、收费的或非收费的界面库。核心思想其实就是调用 Windows GDI 或 GDI+ 函数进行自绘，GDI  提供的自绘接口在一些追求界面细节的精细程度上不够且 GDI 接口都是 C 接口不符合现在开发软件使用的面向对象模型的理念，所以后来微软又推出来一套基于GDI的纯面向对象的绘制接口 GDI+（GDI Plus），更不用说专门用于图形要求更高的领域的 opengl、direct3D 了。

Flamingo 的 PC 版本的界面即是我利用 GDI/GDI+ 绘制出来的，代码地址：https://github.com/balloonwj/flamingo

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

> **我为 Flamingo 专门录制了 3 部高清技术讲解视频以方便读者学习，视频中介绍了 Flamingo 的编译和部署方法、整体架构、各个模块的技术实现细节以及如何学习 Flamingo 的方法，需要视频教程下载链接：**
> 
> 链接: https://pan.baidu.com/s/1p_dBPJqgsvpLXvVyCotjUg
> 
> 提取码: ah6c

比较著名的开源的界面库像 DUILIB，这个库最先由一位德国人开发，后面中国的一些开发者接手，并产生了不同的分支。微信和百度云盘的 PC 客户端都是基于 DUILIB。收费的界面库像迅雷的bolt。这里不再列举各种界面库的名字了，无论哪种界面库其核心技术都是自绘。这里不得不说一下这里的 DUI 思想，做 Windows 界面开发，这是一种必须理解的界面绘制思想。所谓 DUI，即 Direct Draw on Parent ，即直接子控件对象直接绘制在父窗口上，也就是相当于只有父窗口一个窗口句柄。原来可能每个子控件都是窗口，Windows 消息机制决定，窗口越多，窗口的消息泵也就越多。为了减少这些消息泵，子窗口直接绘制在父窗口上，然后利用 PtInRect 这样的 API 函数去检测鼠标是否位于这些绘制出来的控件上，然后利用 PostMessage/SendMessage 这样的 API 产生消息，交给父窗口或者自己处理。

Windows 版本的微信的界面就是基于开源界面库 DUILIB 开发的，你可以在微信安装目录下找到该开源软件的 licence。

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

衡量一个界面库的优劣不仅要看这个界面库的功能，也要看它的流畅性（性能）和用户友好性。腾讯 QQ 从 2009 版本开始，采用了这种思想，到今天的 2022 版本，其界面的设计是一个经典的范例。当然界面库就该做界面库自己的工作，现在一些界面库的作者因为一定的利益驱使，在其发布的界面库里面包含了方方面面的功能，核心的界面功能不去优化，一些与界面有关的类对象，因为继承链的关系体积已经达到几十 k 甚至更大，并号称“旗舰版”。这很容易对一些初学 Windows 界面开发的新手造成误导。界面是客户端软件的脸面，也是用户体验好与差的第一衡量标准，一个用户体验好的软件界面是开发人员、产品人员和美工人员共同努力的结果。

## 四、如何学习 Windows 编程

根据前面的介绍，在了解 Windows 软件的特定和编程习惯之后，你需要挨个去学习 Windows 的各个知识点，而不是孤零零地去学习单个的 API 函数，Windows 编程包括的知识点，我这里列举了一些常用的，也是我之前招 Windows 程序员的考察范围之一：

- Windows 程序的基本原理
    
- Windows 程序风格与特点
    
- 单字符与宽字符，API 宏
    
- Windows 错误码
    
- 函数调用方式
    
- 窗口与控件
    
- Windows的消息
    
- 从 gdi/gdi+ 到界面库
    
- DUI 思想
    
- 从WIN API到 MFC/WTL 等框架
    
- 从伯克利 socket 到 Windows 事件驱动型网络
    
- Windows上的几种网络通信模型
    
- 你一定要知道的 Window 高性能网络通信模型——完成端口
    
- 进程与线程
    
- 文件操作与内存映射文件
    
- ini 文件与注册表
    
- Windows 中的句柄
    
- 内存管理
    
- TLS 技术
    
- dll 技术
    
- 钩子技术
    
- SEH
    
- Windows 服务程序
    
- COM 技术
    
- activeX 和 OLE
    
- 你一定要知道的调试工具与 dump 文件
    
- 如何制作Window安装程序
    
- 软件安全——ollydbg 与 IDA
    
- Windows开发必备工具包
    

我曾在知乎上开过一个关于 Windows 编程从入门到进阶的讲座，有兴趣的读者可以戳这里：

**画龙点睛——从Windows程序到商业产品（https://www.zhihu.com/lives/909740192015998976）**

关于 Windows 的图书，我推荐两本，这两本书是互补的。

1.《**Windows 程序设计（第五版）**》

这本书讲述了 Windows UI 相关原理的方方面面，且语言朴实、娓娓道来，犹如一位良师益友，我当初也是看这本书进入 Windows C/C++ 开发领域的；这本书的业界地位很高，可以说这本书是中国的老一代 Windows 程序员的启蒙和进阶读物。

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

2.《**Windows 核心编程（第五版）**》

此书英文版叫《**Windows via C/C++**》。

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

这本书正好与上一本相互弥补，讲述的是 Windows 非 UI 部分的运行原理，内容非常丰富，当之“核心”二字无愧，图书的作者是编写 Windows Sysinternals 套件的 Jeffrey Richter，如果你没听说过 Windows Sysinternals 套件，那你一定听说过，Process Explorer：

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

侯捷老师评价这本书是“搞 Windows 开发，需要两样资源，一是 MSDN，一本就是《Windows 核心编程》”，这本书口碑非常好，多次重印，每一版都有一些新的改动和惊喜。

**《Windows 核心编程》这本书不要指望一次性读懂，应该是不同阶段拿来读一下，慢慢会理解书中越来越多的内容。**

你可以一边学习 Windows 编程理论知识，一边阅读一些不错的 Windows 开源软件的代码，这里推荐几款我曾经看过的：

### 金山卫士

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

![](http://mmbiz.qpic.cn/mmbiz_png/OIdl32LQlqpmKgWE0wRXCiafyf8F0GZbX5ldCfUdmYLpngEDp1QfSD6RsuukHIq0llJ8yiayfCqEFibVX9n6OQwTQ/300?wx_fmt=png&wxfrom=19)

**程序员小方**

技术，生活，编码，加班，读书学习，这里是程序员小方的 IT 生活。

21篇原创内容

公众号

打开后回复“**五套源码**”，获取金山卫士源码

### 电驴

### ![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

![](http://mmbiz.qpic.cn/mmbiz_png/OIdl32LQlqpmKgWE0wRXCiafyf8F0GZbX5ldCfUdmYLpngEDp1QfSD6RsuukHIq0llJ8yiayfCqEFibVX9n6OQwTQ/300?wx_fmt=png&wxfrom=19)

**程序员小方**

技术，生活，编码，加班，读书学习，这里是程序员小方的 IT 生活。

21篇原创内容

公众号

打开后回复“**五套源码**”，获取电驴源码  

### 开源 FTP 软件 —— Filezilla  

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

![](http://mmbiz.qpic.cn/mmbiz_png/OIdl32LQlqpmKgWE0wRXCiafyf8F0GZbX5ldCfUdmYLpngEDp1QfSD6RsuukHIq0llJ8yiayfCqEFibVX9n6OQwTQ/300?wx_fmt=png&wxfrom=19)

**程序员小方**

技术，生活，编码，加班，读书学习，这里是程序员小方的 IT 生活。

21篇原创内容

公众号

打开后回复“**五套源码**”，获取Filezilla源码

### TeamTalk

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

TeamTalk 是蘑菇街开源的一款用于企业内部的即时通信工具，其代码下载地址是： 

**https://github.com/balloonwj/TeamTalk**

## 五、写在最后的话

相比较其他编程，有人说 Windows 编程已经日薄西山了，其实也不尽然。我们大多数人的工作和娱乐的电脑仍然是 PC 机和 Windows，只不过因为 Windows 上的各种软件我们已经熟悉到觉得它们存在是那么理所当然了，开发这样的软件的技术已经处于一个相对稳定和成熟的阶段。就职业发展来讲，如果你生活在二三线城市，掌握了 Windows 编程，你可以在 Windows 开发各种桌面软件，这会大大增加你的经济收入。

## 六、一些你可以利用的资源

技术面试中常见的计算机网络题，可以看这里：

**轻松搞定技术面试中常见的网络通信问题**(www.zhihu.com/lives/922110858308485120)

关于求职后端开发的一些问题可以看这里：

**如何求职 C++ 后端开发岗位**(www.zhihu.com/lives/1215948129440518144)

最后，祝你能坚持下来，学好 Windows 编程。

原创不易，如果觉得有帮助，请给 @张小方 点个赞呗～

  

**推荐阅读**

- # [我苦逼的转行计算机开发的经历](http://mp.weixin.qq.com/s?__biz=MzI0NTE4NTE4Nw==&mid=2648792863&idx=1&sn=cf0a468120381a1e3f71d62812f9862b&chksm=f14790d3c63019c549bb50bfb0ad17a8a75aa9364b1f0b829467a832af6f57d2e19cc76a6a07&scene=21#wechat_redirect)
    
- [最难调试修复的 bug 是怎样的？](http://mp.weixin.qq.com/s?__biz=MzI0NTE4NTE4Nw==&mid=2648792590&idx=1&sn=0ac92af61726a5a70c933c34daf9bdff&chksm=f14791c2c63018d4eb19445f5cd1d8cd3d8072001f7835d382f99ba0c93fce7ba3f4943a4411&scene=21#wechat_redirect)
    
- [阿里面试，拿到 P7 offer](http://mp.weixin.qq.com/s?__biz=MzI0NTE4NTE4Nw==&mid=2648792649&idx=1&sn=467aa3ca593346ef4aa9e88816965348&chksm=f1479185c6301893f0df2eab1bb4df9beec8f4fcdf21de109e9311d347a8043391e23a34a872&scene=21#wechat_redirect)
    
- [高考后，张小方成为一名程序员](http://mp.weixin.qq.com/s?__biz=MzI0NTE4NTE4Nw==&mid=2648792705&idx=1&sn=ea8ad7cea130bcda63203ea30c8dfd07&chksm=f147914dc630185b57ae061d1d53af6852e60b8dfb5811c1f935f250ab1fb01d9c0d043e33df&scene=21#wechat_redirect)
    
- [在 2021 年写一本 C++ 图书是一种什么体验？](http://mp.weixin.qq.com/s?__biz=MzI0NTE4NTE4Nw==&mid=2648792707&idx=1&sn=c7b9cc545937aa88f26e3a791c0f4b2e&chksm=f147914fc6301859480d26527d1a132c8ba92b23e0fd22489f46a44b10e9dac62f27ac7f62dd&scene=21#wechat_redirect)
    
- [网络编程到底要怎么学？](http://mp.weixin.qq.com/s?__biz=MzI0NTE4NTE4Nw==&mid=2648792716&idx=1&sn=f3a7ff324f54ba6941a4403125f57fde&chksm=f1479140c6301856a93b35b87fe2be22b558c6072b883355b2f2ea4070f465f0db12297ecd31&scene=21#wechat_redirect)
    
- [Modern C++ 有哪些真正提高开发效率的语法糖？](http://mp.weixin.qq.com/s?__biz=MzI0NTE4NTE4Nw==&mid=2648792756&idx=1&sn=49ee9c0ac943bb075b4f000e706fd857&chksm=f1479178c630186ee7968559ef8e91978a7d4b62e5c583b41d136a0e2c0ddf764b0566fc5de4&scene=21#wechat_redirect)
    
- ## [程序员面试，面试官最后说，你还有什么要问的吗？该怎么回答？](http://mp.weixin.qq.com/s?__biz=MzI0NTE4NTE4Nw==&mid=2648792763&idx=1&sn=9be653ca78404ddbf9332c1b7ae6c7fa&chksm=f1479177c63018615767f4d77d475ba570ca42eb2dbbfa53c4aea918b286f0ce4e22eb25409c&scene=21#wechat_redirect)
    
- ## [第一次亲密接触](http://mp.weixin.qq.com/s?__biz=MzI0NTE4NTE4Nw==&mid=2648792773&idx=1&sn=dbbcf20f879882591fe71913c69e28c3&chksm=f1479109c630181f1b420b050ddbe108918d4fd97a9a78bb9085762ba9de65200f1e922eb0c6&scene=21#wechat_redirect)
    
- # [能不能推荐几本 C++ 的书？](http://mp.weixin.qq.com/s?__biz=MzI0NTE4NTE4Nw==&mid=2648792788&idx=1&sn=1c05ac8095604682b2cd329189aed977&chksm=f1479118c630180e16743ee17d932a04c76dc618db2445090c7cb3b852ba98730bd8f258345d&scene=21#wechat_redirect)
    
- # [若干年后的某个夏日，我还是江南皮革厂的跑路老板，你也还是手捧奶茶的浅笑女孩。](http://mp.weixin.qq.com/s?__biz=MzI0NTE4NTE4Nw==&mid=2648792800&idx=1&sn=3c5a9e4f1491c47387e7bf78e6bcbd4a&chksm=f147912cc630183aed033efb57948eb7a1a2f71b73d1b86117447efb0a3ecfe71bbb13a234e3&scene=21#wechat_redirect)
    
- # [WebSocket 一般会用在什么实际的场合？](http://mp.weixin.qq.com/s?__biz=MzI0NTE4NTE4Nw==&mid=2648792806&idx=1&sn=d57691526e801ccaa9e339a5c1154ade&chksm=f147912ac630183cf3034a183a9222d4432c5b5b1e19d090ff103732fa93e95b28b79b0f651f&scene=21#wechat_redirect)
    
- # [大学四年、硕士三年、工作七年，我都读了哪些计算机经典书籍？](http://mp.weixin.qq.com/s?__biz=MzI0NTE4NTE4Nw==&mid=2648792838&idx=1&sn=fb2b865e6ad650781fbef6ff7045a9f0&chksm=f14790cac63019dc0f51e5a94277939c8c2d0611dc7b4d1cbc38a7ffb8f4a242c492f7e3ec31&scene=21#wechat_redirect)
    

  

![](https://res.wx.qq.com/op_res/NN_GToMiIjsXzgPzF9-74ZzwR3cA9-fv3o9eWo8f5gQWqx71CmGlY8kFxuIxZaG0TB1bFeMCmh1DGN_pWMRg0A)

**帐号已迁移**

公众号

  

**原创不易，如果觉得有帮助，请给小方点个赞呗～～**

![](https://mmbiz.qlogo.cn/mmbiz_jpg/zCbmRnz6zbHp58oZHt9WfTQZu74Qt9obHib4bgpiaW6unFZDrbj3yDM50QT6yCN4WenN0YfqaRapLparBNL3c45A/0?wx_fmt=jpeg)

张小方

 学好技术，回村娶小芳。 

![赞赏二维码](https://mp.weixin.qq.com/s?__biz=Mzk0MjUwNDE2OA==&mid=2247497580&idx=1&sn=b3ecdcdc6a8accb1e8090d23bf653fd5&source=41&key=daf9bdc5abc4e8d0d46936fa8299c176d880c7ef6b4b7461274f5877d54ba210dff637ee3f671b6bc61fcc82f323aeb8247d9376ca21ee9ebe077e4fb957cb83fd0562ac51ff8b2e1364cae998b28cf077550fa9ace3fb279ae52cf49cb67c4bd344f901950ab211372b829f96ffc1b42c28141bf0ebf21332007b8ff1911ed5&ascene=14&uin=MTEwNTU1MjgwMw%3D%3D&devicetype=iMac+MacBookAir10%2C1+OSX+OSX+14.6.1+build(23G93)&version=13080710&nettype=WIFI&lang=en&countrycode=CN&fontScale=100&exportkey=n_ChQIAhIQckPvxHWmZaAxHOVbE8GMmRKYAgIE97dBBAEAAAAAAN1tMm7hLqMAAAAOpnltbLcz9gKNyK89dVj03I8I5kuAiLVvBmdb9vmGLezFPWH%2F0rJgbYLbcw2VwgpY9ItYSLQWkK5sageNnV8W6G5FaesdVR8wocMCZBznbjCgD%2BefmghJy0nifi2n%2FVbu5CZeoAEX52GgfnWpBnEOl8bmPvDbzk8CatGH7Wv1RN4sNj8cnxauVSOwkO%2BLOKZ4kr%2BXY9uaLO455PpejnsiuaJcYpvI2c%2Fki29dKZdijj%2Fahuw5Sx8GzY37GXdtwgQO3QSa4pr3B1hNyl8Qxe6yrrXVEj7%2FH0WTr5y90D68M0z7evttNXn8leZ9gzeW3bR%2Bv9fiwV4ocrjszfxmkG5CAN4%3D&acctmode=0&pass_ticket=93R483nS%2FIhsWKAxaN03iabwd%2FQOg6t2PS%2FzCsBD1u3iCm7bALVvxsIRJP2Awnv5&wx_header=0)Like the Author

4 like(s)

![](http://wx.qlogo.cn/mmopen/TZiaoT8B8oO0GsCiaL4dU6HNPFHmw3KrQaH8YFBKDiauBVSm2icibCaqI8JO5GXErLJ2lsl6E4fh4doaicWp1qyR0buRYSWppc5xXW/64)![](http://wx.qlogo.cn/mmopen/WlbFPUDJ0ZP6Unq3B3wiahm0u0xe26epZ2CpeArTRPJjvwDgrmSyRKrIeqYiaEen5uxbC7PbhpPTKvWqyiax0TVeTibibpYOauyGPzsPSslwlbtl3J9rglsmnjHkn2rJKB7A8/64)![](http://wx.qlogo.cn/mmopen/IRUJvvhticxSx8BJogYFzrmTZaLAa51slGxHNibkEtwl2U8jtqZolvBfl3IJIJeQdbwEzlMiaC90WGhqnfXia87wOHk5AVYnnooO5LMfBicJomEn4vnhQDdnqIly9QIxAAcsv/64)![](http://wx.qlogo.cn/mmopen/IRUJvvhticxSqzAexcsTmc5MUUsKXpRF8JEtzp2EYMz9uv69HmmyhsIGiaceBADxRpWKPicZiaiatNFMTlKzOLO2K3yLLmwuDCE6b/64)

Reads 7209

​

Comment

**留言 34**

- 雨乐
    
    2022年3月8日
    
    Like5
    
    封面妹子不错![😄](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)
    
    CppGuide
    
    Author2022年3月8日
    
    Like
    
    学习要抓住重点![[抠鼻]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)
    
- nullptr
    
    2022年3月8日
    
    Like2
    
    现在搞Windows桌面开发的越来越少，岗位越来越少，学习门槛却不低，好多初学者都望而止步了。
    
    CppGuide
    
    Author2022年3月8日
    
    Like
    
    是技术比较成熟，要求高，对新人不友好了。
    
- 坚韧之内
    
    2022年3月8日
    
    Like2
    
    学完c++基础语法，可以学这个了吗？
    
    CppGuide
    
    Author2022年3月8日
    
    Like
    
    可以的。
    
- 谈山与木宁
    
    2022年3月9日
    
    Like1
    
    文章讲得很好，像我这样半路出家的，除了qt啥也不会，唉
    
    CppGuide
    
    Author2022年3月9日
    
    Like1
    
    早些年用mfc，现在不少为了跨平台，用QT，但是做客户端开发不能只会用框架，在Windows平台上还必须深入框架背后的API原理，知道如何设计框架，这才能触类旁通。
    
- Tan
    
    2022年3月8日
    
    Like1
    
    哇嘎![[憨笑]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=) 懂的 点赞
    
- sure
    
    2022年3月8日
    
    Like1
    
    封面妹子拍照姿势跟我真是一模一样![[破涕为笑]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)
    
- 又又
    
    2022年3月11日
    
    Like
    
    桌面软件的明天是web，现在则是手机app。说桌面软件弱势都是带了情怀高看了
    
    CppGuide
    
    Author2022年3月11日
    
    Like
    
    不是这样的，对效率有要求的还得选原生开发。
    
- Jack_Li
    
    2022年3月10日
    
    Like
    
    岗位太少，对大多数人来说，得不偿失。
    
- 千寻🍀
    
    2022年3月10日
    
    Like
    
    巨讨厌QT，太肥
    
    CppGuide
    
    Author2022年3月10日
    
    Like
    
    嗯，有一些人确实不太喜欢QT，跨平台必然有许多额外的东西。
    
- 安川
    
    2022年3月9日
    
    Like
    
    你对滥觞这个词的用法是不是有什么误解？
    
    CppGuide
    
    Author2022年3月10日
    
    Like
    
    你理解的应该是何意？怎么用？
    
- Husen
    
    2022年3月9日
    
    Like
    
    一直使用QT写windows程序, windows api几乎没有直接调用到, 发现文中很多东西都没有接触, 都怀疑是不是搞 windows桌面软件开发了......
    
    CppGuide
    
    Author2022年3月9日
    
    Like
    
    不想做框架或者界面程序员，背后的操作系统原理一定要搞清楚的。加油。感谢打赏！![[抱拳]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)
    
- 木木
    
    2022年3月9日
    
    Like
    
    UI库长知识了，有机会去看下，收藏先
    
    CppGuide
    
    Author2022年3月9日
    
    Like
    
    UI是做客户端一个重要方面，毕竟对用户来说，这是他们体感的第一开源，所以想做好客户端，也要在UI上有自己的知识体系和理解，不能只会拖拖拽拽。感谢打赏。![[抱拳]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)
    
- Mints
    
    2022年3月9日
    
    Like
    
    最后是都转方向了嘛![[捂脸]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)
    
- 哈喽，大猫
    
    2022年3月8日
    
    Like
    
    大佬，向您学习
    
    CppGuide
    
    Author2022年3月8日
    
    Like
    
    相互学习![[抱拳]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)
    
- 热夏
    
    2022年3月8日
    
    Like
    
    OPPO用的好像就是TeamTalk 然后自己改 实习的时候别人告诉我的
    
    CppGuide
    
    Author2022年3月8日
    
    Like
    
    ![[强]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)
    
- 北风与太阳
    
    2022年3月8日
    
    Like
    
    所以大厂客户端岗位是不是坑呢![[让我看看]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)
    
    CppGuide
    
    Author2022年3月8日
    
    Like
    
    不一定吧。
    
- 隐姓埋名做coding
    
    2022年3月8日
    
    Like
    
    方哥出品，必属精品![[强]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)![[强]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)![[强]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)![[强]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)![[强]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)![[强]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)
    
- 于佑飞
    
    2022年3月8日
    
    Like
    
    干得了服务器，写的动客户端。关键文笔也不错
    
    CppGuide
    
    Author2022年3月8日
    
    Like
    
    ![[悠闲]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)
    
- echoOo
    
    2022年3月8日
    
    Like
    
    看得我都想从linux服务端转windows客户端了![[呲牙]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)
    
    CppGuide
    
    Author2022年3月8日
    
    Like
    
    可以学一学，相辅相成，技多不压身。
    
- 一去二三里
    
    2022年3月8日
    
    Like
    
    本来是做Linux服务端，结果服务器现在用Java重做了，被迫做客户端了
    

已无更多数据

[](javacript:;)

![](http://mmbiz.qpic.cn/sz_mmbiz_png/GSweNIrkicYvM1mIwPctlYONEDKJwUfRZ57uAkVR59MpX1cVnmmnyPZ5O9OCuys78Sy6fOEncwfgWpgCo9Tibeag/300?wx_fmt=png&wxfrom=18)

CppGuide

4216

34

Comment

**留言 34**

- 雨乐
    
    2022年3月8日
    
    Like5
    
    封面妹子不错![😄](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)
    
    CppGuide
    
    Author2022年3月8日
    
    Like
    
    学习要抓住重点![[抠鼻]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)
    
- nullptr
    
    2022年3月8日
    
    Like2
    
    现在搞Windows桌面开发的越来越少，岗位越来越少，学习门槛却不低，好多初学者都望而止步了。
    
    CppGuide
    
    Author2022年3月8日
    
    Like
    
    是技术比较成熟，要求高，对新人不友好了。
    
- 坚韧之内
    
    2022年3月8日
    
    Like2
    
    学完c++基础语法，可以学这个了吗？
    
    CppGuide
    
    Author2022年3月8日
    
    Like
    
    可以的。
    
- 谈山与木宁
    
    2022年3月9日
    
    Like1
    
    文章讲得很好，像我这样半路出家的，除了qt啥也不会，唉
    
    CppGuide
    
    Author2022年3月9日
    
    Like1
    
    早些年用mfc，现在不少为了跨平台，用QT，但是做客户端开发不能只会用框架，在Windows平台上还必须深入框架背后的API原理，知道如何设计框架，这才能触类旁通。
    
- Tan
    
    2022年3月8日
    
    Like1
    
    哇嘎![[憨笑]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=) 懂的 点赞
    
- sure
    
    2022年3月8日
    
    Like1
    
    封面妹子拍照姿势跟我真是一模一样![[破涕为笑]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)
    
- 又又
    
    2022年3月11日
    
    Like
    
    桌面软件的明天是web，现在则是手机app。说桌面软件弱势都是带了情怀高看了
    
    CppGuide
    
    Author2022年3月11日
    
    Like
    
    不是这样的，对效率有要求的还得选原生开发。
    
- Jack_Li
    
    2022年3月10日
    
    Like
    
    岗位太少，对大多数人来说，得不偿失。
    
- 千寻🍀
    
    2022年3月10日
    
    Like
    
    巨讨厌QT，太肥
    
    CppGuide
    
    Author2022年3月10日
    
    Like
    
    嗯，有一些人确实不太喜欢QT，跨平台必然有许多额外的东西。
    
- 安川
    
    2022年3月9日
    
    Like
    
    你对滥觞这个词的用法是不是有什么误解？
    
    CppGuide
    
    Author2022年3月10日
    
    Like
    
    你理解的应该是何意？怎么用？
    
- Husen
    
    2022年3月9日
    
    Like
    
    一直使用QT写windows程序, windows api几乎没有直接调用到, 发现文中很多东西都没有接触, 都怀疑是不是搞 windows桌面软件开发了......
    
    CppGuide
    
    Author2022年3月9日
    
    Like
    
    不想做框架或者界面程序员，背后的操作系统原理一定要搞清楚的。加油。感谢打赏！![[抱拳]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)
    
- 木木
    
    2022年3月9日
    
    Like
    
    UI库长知识了，有机会去看下，收藏先
    
    CppGuide
    
    Author2022年3月9日
    
    Like
    
    UI是做客户端一个重要方面，毕竟对用户来说，这是他们体感的第一开源，所以想做好客户端，也要在UI上有自己的知识体系和理解，不能只会拖拖拽拽。感谢打赏。![[抱拳]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)
    
- Mints
    
    2022年3月9日
    
    Like
    
    最后是都转方向了嘛![[捂脸]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)
    
- 哈喽，大猫
    
    2022年3月8日
    
    Like
    
    大佬，向您学习
    
    CppGuide
    
    Author2022年3月8日
    
    Like
    
    相互学习![[抱拳]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)
    
- 热夏
    
    2022年3月8日
    
    Like
    
    OPPO用的好像就是TeamTalk 然后自己改 实习的时候别人告诉我的
    
    CppGuide
    
    Author2022年3月8日
    
    Like
    
    ![[强]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)
    
- 北风与太阳
    
    2022年3月8日
    
    Like
    
    所以大厂客户端岗位是不是坑呢![[让我看看]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)
    
    CppGuide
    
    Author2022年3月8日
    
    Like
    
    不一定吧。
    
- 隐姓埋名做coding
    
    2022年3月8日
    
    Like
    
    方哥出品，必属精品![[强]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)![[强]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)![[强]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)![[强]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)![[强]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)![[强]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)
    
- 于佑飞
    
    2022年3月8日
    
    Like
    
    干得了服务器，写的动客户端。关键文笔也不错
    
    CppGuide
    
    Author2022年3月8日
    
    Like
    
    ![[悠闲]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)
    
- echoOo
    
    2022年3月8日
    
    Like
    
    看得我都想从linux服务端转windows客户端了![[呲牙]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)
    
    CppGuide
    
    Author2022年3月8日
    
    Like
    
    可以学一学，相辅相成，技多不压身。
    
- 一去二三里
    
    2022年3月8日
    
    Like
    
    本来是做Linux服务端，结果服务器现在用Java重做了，被迫做客户端了
    

已无更多数据