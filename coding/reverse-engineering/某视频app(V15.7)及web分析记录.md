# 

xwtwho 看雪学苑

 _2021年10月26日 18:09_

![](https://mmbiz.qpic.cn/mmbiz_jpg/kR3MmOkZ9r6wJxjzQu2YDMy6buUicYDAHIAo11aXd66dvUuuHowIBaCj5gicLX3Fbs4SSWCRiar6AwNoic6P8YhUZw/640?wx_fmt=jpeg&wxfrom=13&tp=wxpic)  

本文为看雪论坛优秀文章  
看雪论坛作者ID：xwtwho

  

  

初衷是刷抖音太多，发现不能在点赞过的视频列表中直接搜索，就想自己实现下，把这个过程做了下记录，当学习笔记了，纯技术交流用。

**一、软硬件环境：**

**抖音android (V15.7，应用宝渠道)**

**抖音web (V16.1)**

IDA 7.5

Frida 14.2.2

Gda3.86

JEB

jadx-gui

unidbg

LineageOs 17.1 (android 10)

小米8

  

**二、流水账 (直接按时间顺序，有坑写坑)**

**1、App X-Gorgon算法定位**

开始也是直接网上搜了下，有下面这个：

_https://blog.csdn.net/weixin_48271161/article/details/108544446_

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

在这个V15.7版本里面没找到相关的so（后来发现我看的这个版本对应的是libmetasec_ml.so）。

  

还有其它一些，比如搜索X-Gorgon，hook HashMap，发现对这个版本都不好使了，那就从头开始了，自己找这个加密关键点。

先上frida，输出jni函数：

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

现在回过头看，发现当时就输出了libmetasec_ml.so jni记录（虽然不是直接的加密函数），但是当时可能是这个记录太多了，竟然没注意到，估计当时看到了也没留意，因为开始也不知道这个so是做什么的。

  

看到这个Cronet，搜了下：

Cronet网络库系列(一)：用例与原理实现详解

_https://segmentfault.com/a/1190000021095757_

_https://blog.csdn.net/u010983881/article/details/97770544/_

【Android】移动端接入Cronet实践

根据登录url找到下面代码：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

然后根据调用关系翻代码，找Cronet相关调用,浏览代码，翻到com.ttnet.org.chromium.net.impl.CronetUrlRequest，也没

发现有设置X-Gorgon头的地方（其它header倒是有设置），中间也尝试通过send反找，调用线太长，还是决定换个方式来做。

  

直接下了个cronet代码：

_https://github.com/hanpfei/chromium-net_

然后就是对着代码，确定认为会相关的几个函数，上IDA调试：

nativeCreateRequestAdapter

nativeAddRequestHeader

最后找到这个点：

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

到这里就跟libmetasec_ml.so关联起来了。

  

知道了调用点，就想直接调用libmetasec_ml.so方便调试，写了个程序来加载这个so，发现会异常，考虑到可能是上下文环境不全，就打算按正常流程来加载so，也调用JNI_OnLoad：

  

void* hMetasecSo = dlopen("/data/local/tmp/libmetasec_ml.so", RTLD_LAZY);

    typedef jint(*FUN)(JavaVM* vm, void* res);

    FUN func_onload = (FUN)dlsym(hMetasecSo, "JNI_OnLoad");

这个调用是需要JavaVM参数的，就准备加载libart来调用JNI_CreateJavaVM创建JavaVM，参考网上资料设置好参数：

  

    JavaVMInitArgs vm_args;

    JavaVMOption options[2];

    options[0].optionString = "-Djava.class.path=.";

    vm_args.version = JNI_VERSION_1_6;

    vm_args.options = options;

    vm_args.nOptions = 2;

    vm_args.ignoreUnrecognized = JNI_TRUE;

  

编译运行后，直接崩了，查看日志，提示没有设置NoSigChain。

然后查看android源码，找到对应地方看了下，是在检查创建VM的选项参数，应该是没有no-sig-chain这个参数，网上搜了下，没有找到怎么设置这个参数的，然后根据代码，结合其它属性改了下代码，增加了一条：

  

options[1].optionString = "-Xno-sig-chain";

  

android源码里面也跳过了这个相关的，编译运行，之前的错误就没有了，但是后面还是异常退出了，后来查了下，找到下面信息：

  

从Android N开始（SDK >= 24），通过dlopen打开系统私有库，或者lib库中依赖系统私有库，都会产生异常，甚至可能导致app崩溃。

  

应用可以调用/vendor/etc/public.libraries.txt和/system/etc/public.libraries.txt里面的所有so库，所以往这个文件写入自己希望被调用的so，这个库就变成共用的了，任意应用就可以找到这个so库了。

试了下上面的方法，包括把libart.so及相关的库放到其它用户目录下，还是不行，考虑到本来就是想还原运行环境的，就想直接上APK吧，还能省去自己创建VM，就写了个测试APK调用这个so：

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

编译好apk后导入到手机运行：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

Didn't find class "com.bytedance.mobsec.metasec.ml.MS"

直接参考抖音补上对应的包路径：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

现在能运行了，但是调用JNI_OnLoad会异常，准备上IDA调试，看了下流程图：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

带了llvm，看so文件尾部记录的是Apple LLVM version 10.0.1 (clang-1001.0.46.3)。

  

跟了下，一些跳转都做了处理，不是很好分析，准备上unidbg (_https://github.com/dqzg12300/unidbg_tools_，可以用这个大佬整理的)，trace代码下来分析。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

根据trace得到的指令流，发现有这种访问"/proc/self/exe"路径返回-1的情况，我是直接修改这个函数，直接返回1了，这个函数只是用来取e_machine字段的：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

后面继续trace，然后结合动态调试，发现有代码校验的地方：

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

上面都处理后，测试apk就可以正常跑完JNI_OnLoad了，后面就是主动调用加密函数测试了，直接在jni接口中调用libmetasec_ml.so：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

下面插个分支，到这里的时候，在一个群里看到信息说抖音开了web，直接去看了下。

**2、抖音web请求参数_signature算法分析**

直接访问www.douyin.com，看访问参数多了个_signature，这种格式：

&_signature=_02B4Z6wo00901qf0GiQAAIDAwkLkeQfbXMKn9B6AAMkm74。

  

多拿几个比较，发现前面一段（_02B4Z6wo00901）是前缀，后面初步判断是base64格式。

  

直接网上搜了下，找到下面这篇：

  

网络爬虫-今日头条_signature参数逆向(第一弹)_井蛙不可语于海的博客-CSDN博客_byted_acrawler

_https://blog.csdn.net/qq_39802740/article/details/104911315_

  

主要加密算法在acrawler.js，参考这个用node跑起来了，里面的实现是一个js 虚拟机，关于虚拟机还找到这篇：

  

StriveMario/jsvm: 给"某音"的js虚拟机写一个编译器 (github.com)

_https://github.com/StriveMario/jsvm_

  

不过目前版本都看起来跟这个不同了，并且本地调试算法其实也用不上，但是还是可以学习下的。

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

除了文章提到的，还有下面的几个参数要改下：

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

然后就可以调用global.byted_acrawler.init及global.byted_acrawler.sign参数了（这个后面发现这样调用的算法流程跟浏览器跑的其实是不同的）。

  

用浏览器调试也是确定有init ,sign函数的：

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

后面就是直接用node调试，因为原代码的格式都是这样的：

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

  

都是一句话写完逻辑，不方便调试，先整理了下代码，比如这种

opCode = 3 & initCode;         //  initCode % 4

  

那>2的分支就是==3了，长的代码就分割下。

  

整理后也方便加日志，输出中间数据。

  

调试中把vm的基础操作整理出来(vm_xor,vm_and等等)，特别字符串连接的指令，可以加上日志，方便定位。

  

Vm中会检查运行环境，包括下面这些：

domDetect

debuggerDetect

nodeDetect

phantomDetect

webdriverDetect

incognitoDetect

hookDetect

locationDetect

  

检查结果会参与加密，作为其中一段的因子。

当时看了几个签名数据，就有个疑问的，相同的数据，_signature总是不同的，当时想到应该有个变量因子的，但是看https提交的参数没发现这个变量，那这样就是直接记录在_signature中了。

  

直接base64解密排除前缀后的数据，类似这种：

<Buffer 7c 7e 62 a4 00 00 20 30 83 81 9d 5b 28 d0 87 e9 7c 76 e3 80 00 07 26 f8>

  

比较多个后也没发现明显的变量因子，猜到可能是时间戳，但是没找到哪个数据段是标识的时间，那就直接开始看流程了。

  

调试过程发现有getTime的调用，直接改为固定的，最后得到的_signature就不会变了：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

那就是确定跟时间戳相关了，也就是服务器可以通过这个得到时间戳或者转换过的值，来验证客户端上报的_signature是否正确了。

  

后面算法中，主要就是参数字符串（location,user_agent,param）的处理（xor,or,and等）得到各个hash值，然后类似base64加密得到加密字符串（这里不是直接得到完整明文，再最后一次base64的方式）：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

处理完后，最后2个字符是附加的校验字符，是对前面数据得到一个DWORD值的低字节：

_02B4Z6wo00f014U9W.wAAIDB4IuloLYVYYOFPV9AAIGw

2260354722 '86ba46a2'

_02B4Z6wo00f014U9W.wAAIDB4IuloLYVYYOFPV9AAIGwa2

本以为搞完了，直接跟浏览器访问的一比较，悲剧了，竟然加密结果不同，直接上浏览器调试。

  

前面已经整理了代码的，浏览器访问的时候js是直接下载的，直接修改DNS，把这个js定向到本地服务器：

127.0.0.1 sf1-ttcdn-tos.pstatp.com

这样浏览器访问的时候也是用的整理后的js了，有了前面的调试经历，这个也很快搞清楚了流程。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAADAAAAAQCAYAAABQrvyxAAACfElEQVRIS9WVO2hUURCGv20Eg1bapVC0sFNJJahFRPDRWFgoRokgQbAQLFJJdNFKFAVBNMROI1oYsBITsPDRpAgkNkmjEoggqIWNIILyywwMs3MxhevjwLLs3XPnzP+a0+L/XZuAE63U/wBwNz0bBWYAfcd1DrjYJfxrgHFgT9HLGeALUAIYAS6kl54A+lxNz3cAL7sEoCq7HdgVSCsBVMiPWrWszJ8EsBK4BjwEDgInrafRbKEIQJa5BawDViVrzQKHgAUrJOsdM3ZkrdP2n1sy203sPQC2eCOAW6NiX3U2JMuWCuTCKlYpIADDwBQgaV+ERpSXPgN/BLgHXAqgnCTlRxZ0dp+b7zMANb/TAPakbHQoMARMAB8BZ6/JQsqF9uw1dp7ayfJpL/AZeAR8AK4HABGwKxObVEC1HKhAVsOiQwFnQv66b0UO2/RRM2I8B1o5mA/M6aAbwHlgH3AWWJsAqLQOj6AyAA0TAdVzkVmtDgDR/xnAehtpss5kAKPfYqxtJ+hQHX4T2GzM5Wa11fd5g5UC3nRla7f2eA6xj1GpIB/rWzK/thDHO0FqSBkBup0AvAXumMcjgEWbJqrrFnS7us/dQhVQB+VuWaoAyL+PzQIRgCykMfbebKUGtgZl/EAFOjYXAWjPfmDalPqVAiJU2aruG9VtRwAxA1EBsf4MGATehFt5DvgE9AeDekDj2GxSQMNBt61Wk4VUT5mKI1v7SwWUAdlmm7HsFrpsDOgu0Cx2O30DTgFjDSH7XY+bMvDzIs0WWs6hq80i8r6HdznvdWVPBWAF8LXhNEl9xTLw15tXjxWAA8Bxu4DeAd+BjcBu4JVNEYX8n1g/AAC2uh6gEsDjAAAAAElFTkSuQmCC

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

多了对这个图片数据的参与，最后整理测试：

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

整体来看，web上虽然用了JS虚拟机，跟二进制的VM比起来还是弱些了，调试环境弄好后，执行流程就都比较清晰了。

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

这里web签名就基本完成了，继续app分析。

**3、继续APP算法分析**

被上个分支中断了下，思路都断了，又重新熟悉了下，继续开始跟这个加密算法。

  

在测试apk中调用libmetasec_ml.so加密函数后，返回的是NULL，调试后发现会从native调用java的情况：

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

然后参考抖音补全需要的包，里面有热更新保护相关的代码，屏蔽掉，让测试工程能跑起来就行：

//import com.bytedance.JProtect;  
//import com.bytedance.covode.number.Covode;  
//import com.meituan.robust.ChangeQuickRedirect;  
//import com.meituan.robust.PatchProxy;  
//import com.meituan.robust.PatchProxyResult;

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

反正就是各种补，模拟全之前的调用环境。

看到一些检测root相关的字符串：

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

通过分析trace日志，过滤掉一些跳转计算流程后，发现可能的关键调用，用unidbg跑的时候返回是null：

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

下面就是调试抖音进行验证了，修改对应指令为循环点，附加后循环处下断点：

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

那就确定是下面函数返回的签名字符串了：

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

对于自己APK调用时候，可以直接设置首尾地址：

//修改so的起始地址  
 *(unsigned  int *)(dwContextAddr+0x10)=0x100000;  
 //修改结束地址  
 *(unsigned  int *)(dwContextAddr+0x10)=0xFFFFFFFF;

  

跳过这个检查。

检查trace代码，发现有对代码指令的检查：

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

根据调试流程，整理调用链，补全初始化调用：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

测试app可以直接跑出结果了：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

现在是自己写的程序可以跑了，后面是用app中提供签名服务，还是撸算法出来，都方便很多了。

  

搞完测试：

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

整体感觉流程的分析难度不如 wegame。

  

这次分析过程中学习的技能点总结：

  

1、开发测试APK模拟目标的调用环境

  

2、Unidbg模拟调用so中任意地址

  

3、RSA验签过程，其它工作涉及的，之前只是使用API（

//1.明文计算sha256

//2.RSA解密密文（一般是base64解密后的字节流）

//3.比较尾部的串是否跟1的一致（因为解密后的结果是包含这个hash串，不是等于）

）

  

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E) 

  

**看雪ID：xwtwho**

https://bbs.pediy.com/user-home-44250.htm

*本文由看雪论坛 xwtwho 原创，转载请注明来自看雪社区

  

  

[![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458395632&idx=1&sn=34f2b52324f54626d84b46a430f35ed2&chksm=b18f137a86f89a6cb696f51c149732d6d1b428cde53e1fe8bb209ddc228c3774094515af9c79&scene=21#wechat_redirect)

  

[![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)](https://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458387399&idx=2&sn=38495add2a3a3677b2c436581c07e432&scene=21#wechat_redirect)

  

**#** **往期推荐**

1.[极棒项目复现：伪装成温度计的跟踪器](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458400501&idx=1&sn=463d4f92fe05c184082dea74af5a855f&chksm=b18f067f86f88f69856f1149fba1b7d32253a91f8b511374c334d05b3d626c5504c6c913da83&scene=21#wechat_redirect)  

2.[再探格式化字符串漏洞：CVE-2012-3569 ovftool.exe](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458398292&idx=1&sn=0215c401feec6c9f71592c1a177efbad&chksm=b18f1ede86f897c883c0c1ddba9f14da5d064f3d8ee59ed5345b269def8d9306bcabd812769e&scene=21#wechat_redirect)

3.[Android APP漏洞之战——Content Provider漏洞详解](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458398098&idx=1&sn=4d56f69a279f3203b6a4e96380d1a7bd&chksm=b18f1d1886f8940e14303d1960034c4945c6674549375d82a9c7d891f952051ed9f5234aa926&scene=21#wechat_redirect)

4.[钉钉邀请上台功能分析](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458398080&idx=1&sn=1438f3db7d878da9a3c72cadb9e566d6&chksm=b18f1d0a86f8941c6d850df67ee2f05a19de4c324d393f88efb890826d188ce2c79c2a6b51c8&scene=21#wechat_redirect)

5.[Android APP漏洞之战——Activity漏洞挖掘详解](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458398050&idx=2&sn=be762c14ec92d032f642c5270e47350e&chksm=b18f1de886f894fe777768f608e368a474c5e59af641ee3d2127b104f6ee6f21439423dde225&scene=21#wechat_redirect)

6.[PHP反序列化漏洞基础](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458397245&idx=2&sn=d0b40b359f56c55e4db3be6d02545de2&chksm=b18f1ab786f893a1020366fd743fbedbafe1017412bff2a927eea022da170425492025797db9&scene=21#wechat_redirect)

  

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

公众号ID：ikanxue  

官方微博：看雪安全

商务合作：wsc@kanxue.com

  

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**球分享**

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**球点赞**

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**球在看**

  

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

点击“阅读原文”，了解更多！

阅读原文

阅读 5625

​