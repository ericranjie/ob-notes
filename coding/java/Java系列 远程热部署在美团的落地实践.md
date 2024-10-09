# 

原创 凯哥 占峰 李晗等 美团技术团队

_2022年03月17日 19:57_

![图片](https://mmbiz.qpic.cn/mmbiz_png/hEx03cFgUsUrXicw2VXTQTVVN5yxXWEacdY1ZdxTH195Pgibtib8EENJRMia3tzEnyVfgyfAgRibMssKqwlE186TLSw/640?wx_fmt=png&wxfrom=13&tp=wxpic)

**总第495\*\*\*\*篇**

**2022年 第012篇**

![图片](https://mmbiz.qpic.cn/mmbiz_png/hEx03cFgUsVLR21NicmyQxcmiaqQ2KOJJj2JLwgJL4KSbo7CcuMF1hLf4xFjGQiaDRhSPyERxWGChWYP47Oc4sKGA/640?wx_fmt=png&wxfrom=13&tp=wxpic "undefined")

Sonic是美团内部一款用于热部署的IDEA插件。本文主要讲述Sonic的实现细节以及底层原理，从IDEA插件到自动化部署，再到沉浸式开发产品闭环，全方位讲述了Sonic在美团的落地与实践经验。目前业界对标的产品并不多，希望本文能对从事联调/开发/测试等相关方向的同学有所帮助或启发。

- 1 前言

- 1.1 什么是热部署

- 1.2 为什么我们需要热部署

- 1.3 热部署难在哪

- 1.4 Sonic可以做什么

- 1.5 技术产品落地和推广实践经验

- 2 整体设计方案

- 2.1 Sonic结构

- 2.2 走进Agent

- 2.3 那些年JVM和HotSwap之间的相爱相杀

- 2.4 Sonic如何解决Instrumentation的局限性

- 3 Sonic热部署技术解析

- 3.1 Sonic整体架构模型

- 3.2 Sonic功能流转

- 3.3 文件监听

- 3.4 Jvm Class Reload

- 3.5 Spring Bean重载

- 3.6 Spring XML重载

- 3.7 MyBatis 热部署

- 4 总结

- 4.1 热部署功能一览

- 4.2 IDE插件集成

- 4.3 推广使用情况

Sonic是美团内部研发设计的一款用于热部署的IDEA插件，本文其实现原理及落地的一些技术细节。在阅读本文之前，建议大家先熟悉一下[Spring源码](https://www.cnblogs.com/java-chen-hao/category/1480619.html)、[Spring MVC 源码](https://blog.csdn.net/win7system/article/details/90674757) 、[Spring Boot源码](https://www.cnblogs.com/java-chen-hao/p/11829344.html) 、[Agent字节码增强](https://tech.meituan.com/2019/09/05/java-bytecode-enhancement.html)、[Javassist](https://github.com/jboss-javassist/javassist/wiki)、[Classloader](https://juejin.cn/post/6844903794627608589)等相关知识。

## 1 前言

### 1.1 什么是热部署

所谓热部署，就是在应用正在运行时升级软件，却不需要重新启动应用。对于Java应用程序来说，热部署就是在运行时更新Java类文件，同时触发Spring以及其他常用第三方框架的一系列重新加载的过程。在这个过程中不需要重新启动，并且修改的代码实时生效，好比是战斗机在空中完成加油，不需要战斗机熄火降落，一系列操作都在“运行”状态来完成。

### 1.2 为什么我们需要热部署

据了解，美团内部很多工程师每天本地重启服务高达5~12次，单次大概3~8分钟，每天向Cargo（美团内部测试环境管理工具）部署3~5次，单次时长20~45分钟，部署频繁频次高、耗时长，严重影响了系统上线的效率。而插件提供的本地和远程热部署功能，可让将代码变更“秒级”生效。一般而言，开发者日常工作主要分为开发自测和联调两个场景，下面将分别介绍热部署在每个场景中发挥的作用。

![图片](https://mmbiz.qpic.cn/mmbiz_png/hEx03cFgUsVdg254TovgwLsQdUMv10I9j1GCxXBcT0rPhbDfnd63cwibnvjUBXVnWM0qz3fOq3jScdcYcgWC13A/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

图 1

#### 1.2.1 开发自测场景

一般来讲，在用插件之前，开发者修改完代码还需等待3~8分钟启动时间，然后手动构造请求或协调上游发请求，耗时且费力。在使用完热部署插件后，修改完代码可以一键增量部署，让变更“秒级”生效，能够做到快速自测。而对于那些无法本地启动项目，也可以通过远程热部署功能使代码变更“秒级”生效。

![图片](https://mmbiz.qpic.cn/mmbiz_png/hEx03cFgUsUdXQK0EgQDaRD8Rx4lyibS5iaqjGXWxGprIj5xRCv0QaM4bUP94kzZI9O9GkBsjWGVsLRcXXlibqhVg/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

图 2

#### 1.2.2 联调场景

通常情况下，在使用插件之前，开发者修改代码经过20~35分钟的漫长部署，需要联系上游联调开发者发起请求，一直要等到远程服务器查看日志，才能确认代码生效。在使用热部署插件之后，开发者修改代码远程热部署能够秒级（2~3s）生效，开发者直接发起服务调用，可以节省大量的碎片化时间（热部署插件还具备流量回放、远程调用、远程反编译等功能，可配合进行使用）。

![图片](https://mmbiz.qpic.cn/mmbiz_png/hEx03cFgUsUdXQK0EgQDaRD8Rx4lyibS5k3Dt6PZYSQypvRY4DicnvMibQHmvhI3OiabCxHwDLw7fCMNMfJ9KzewDw/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

图 3

所以，热部署插件希望解决的痛点是：**在可控的条件内，帮助开发者减少频繁编译部署的次数，节省碎片化的时间。最终为开发者每天节约出一定量的编码时间**。

### 1.3 热部署难在哪

为什么业界目前没有好用的开源工具？因为热部署不等同于热重启，像Tomcat或者Spring Boot DevTools此类热重启模式需要重新加载项目，性能较差。增量热部署难度较大，需要兼容常用的中间件版本，需要深入启动销毁加载流程。以美团为例，我们需要对JPDA（Java Platform Debugger Architecture）、Java Agent、ASM字节码增强、Classloader、Spring框架、Spring Boot框架、MyBatis框架、Mtthrift（美团RPC框架）、Zebra（美团持久层框架）、Pigeon（美团RPC框架），MDP（美团快速开发框架）、XFrame（美团快速开发脚手架）、Crane（美团分布式任务调度框架）等众多框架和技术原理深入了解才能做到全面的兼容和支持。另外，还需要IDEA插件开发能力，形成整体的产品解决方案闭环，美团的热部署插件Sonic正是在这种背景下应运而生。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

图 4

### 1.4 Sonic可以做什么

Sonic是美团内部研发设计的一款IDEA插件，旨在通过低代码开发辅助远程/本地热部署，解决Coding、单测编写执行、自测联调等阶段的效率问题，提高开发者的编码产出效率。数据统计表明，开发者日常大概有35%时间用于编码的产出。如果想提高研发效率，要么扩大编码产出的时间占比，要么提高编码阶段的产出效率，而Sonic则聚焦提高编码阶段的产出效率。

目前，使用Sonic热部署可以解决大部分代码重复构建的问题。Sonic可以使用户在本地编写代码一键部署到远程环境，修改代码、部署、联调请求、查看日志，循环反复。如果不考虑代码修改时间，通常一个循环需要20~35分钟，而使用Sonic可以把整个时长缩短至5~10秒，而且能够给开发者带来高效沉浸式的开发体验。在实际编码工作中，多文件修改是家常便饭，Sonic对多文件的热部署能力尤为突出，它可以通过依赖分析等手段来对多文件批量进行远程热部署，并且支持Spring Bean Class、普通Class、Spring XML、MyBatis XML等多类型文件混合热部署。下面的动图就演示了多文件复查场景下的增量热部署：

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

那么跟业界现有的产品相比，Sonic有哪些优劣势呢？下面我们尝试给出几种产品的对比，仅供大家参考：

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

上表未把Sofa-Ark、Osgi、Arthas列举，此类属于插件化、模块化应用框架，以及Java在线诊断工具，核心能力非热部署。值得注意的是，Spring Boot DevTools只能应用在Spring Boot项目中，并且它不是增量热部署，而是通过Classloader迭代的方式重启项目，对大项目而言，性能上是无法接受的。虽然，JRebel支持三方插件较多，生态庞大，但是对于国产的插件不支持，例如FastJson等，同时它还存在远程热部署配置局限，对于公司内部的中间件需要个性化开发，并且是商业软件，整体的使用成本较高。

### 1.5 Sonic远程热部署落地推广的实践经验

相信大家都知道，对于技术产品的推广，尤其是开发、测试阶段使用的产品，由于远离线上环境，推动力、执行力、产品功能闭环能否做好，是决定着该产品是否能在企业内部落地并得到大多数人认可的重要的一环。此外，因为很多开发者在开发、测试阶段已逐渐形成了“固化动作”，如何改变这些用户的行为，让他们拥抱新产品，也是Sonic面临的艰巨挑战之一。我们从主动沟通、零成本（或极低成本）快速接入、自动化脚本，以及产品自动诊断、收集反馈等方向出发，践行出了四条原则。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

图 6

## 2 整体设计方案

### 2.1 Sonic结构

Sonic插件由4大部分组成，包括脚本端、插件端、Agent端，以及Sonic服务端。脚本端负责自动化构建Sonic启动参数、服务启动等集成工作；IDEA插件端集成环境为开发者提供更便捷的热部署服务；Agent端随项目启动负责热部署的功能实现；服务端则负责收集热部署信息、失败上报等统计工作。如下图所示：

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

图 7

### 2.2 走进Agent

#### 2.2.1 Instrumentation类常用API

public interface Instrumentation {

//增加一个Class 文件的转换器，转换器用于改变 Class 二进制流的数据，参数 canRetransform 设置是否允许重新转换。\
void addTransformer(ClassFileTransformer transformer, boolean canRetransform);

//在类加载之前，重新定义 Class 文件，ClassDefinition 表示对一个类新的定义，\
//如果在类加载之后，需要使用 retransformClasses 方法重新定义。addTransformer方法配置之后，后续的类加载都会被Transformer拦截。\
//对于已经加载过的类，可以执行retransformClasses来重新触发这个Transformer的拦截。类加载的字节码被修改后，除非再次被retransform，否则不会恢复。\
void addTransformer(ClassFileTransformer transformer);

//删除一个类转换器\
boolean removeTransformer(ClassFileTransformer transformer);\
\
//是否允许对class retransform\
boolean isRetransformClassesSupported();

//在类加载之后，重新定义 Class。这个很重要，该方法是1.6 之后加入的，事实上，该方法是 update 了一个类。\
void retransformClasses(Class\<?>... classes) throws UnmodifiableClassException;\
\
//是否允许对class重新定义\
boolean isRedefineClassesSupported();

//此方法用于替换类的定义，而不引用现有的类文件字节，就像从源代码重新编译以进行修复和继续调试时所做的那样。\
//在要转换现有类文件字节的地方（例如在字节码插装中），应该使用retransformClasses。\
//该方法可以修改方法体、常量池和属性值，但不能新增、删除、重命名属性或方法，也不能修改方法的签名\
void redefineClasses(ClassDefinition... definitions) throws  ClassNotFoundException, UnmodifiableClassException;

//获取已经被JVM加载的class，有className可能重复（可能存在多个classloader）\
@SuppressWarnings("rawtypes")\
Class\[\] getAllLoadedClasses();\
}

#### 2.2.2 Instrument简介

Instrument的底层实现依赖于JVMTI（JVM Tool Interface），它是JVM暴露出来的一些供用户扩展的接口集合，JVMTI是基于事件驱动的，JVM每执行到一定的逻辑就会调用一些事件的回调接口（如果存在），这些接口可以供开发者去扩展自己的逻辑。

JVMTIAgent是一个利用JVMTI暴露出来的接口提供了代理启动时加载（Agent On Load）、代理通过Attach形式加载（Agent On Attach）和代理卸载（Agent On Unload）功能的动态库。而Instrument Agent可以理解为一类JVMTIAgent动态库，别名是JPLISAgent（Java Programming Language Instrumentation Services Agent），也就是专门为Java语言编写的插桩服务提供支持的代理。

#### 2.2.3 启动时和运行时加载Instrument Agent过程

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

图 8

### 2.3 那些年JVM和HotSwap之间的“相爱相杀”

围绕着Method Body的HotSwap JVM一直在进行改进。从1.4版本开始，JPDA引入HotSwap机制（JPDA Enhancements），实现Debug时的Method Body的动态性。大家可参考文档：[enhancements1.4](https://docs.oracle.com/javase/8/docs/technotes/guides/jpda/enhancements1.4.html)。

1.5版本开始通过JVMTI实现的java.lang.instrument（Java Platform SE 8）的Premain方式，实现Agent方式的动态性（JVM启动时指定Agent）。大家可参考文档：[package-summary](https://docs.oracle.com/javase/8/docs/api/java/lang/instrument/package-summary.html)。

1.6版本又增加Agentmain方式，实现运行时动态性（通过The Attach API 绑定到具体VM）。大家可参考文档：[package-summary](https://docs.oracle.com/javase/8/docs/api/java/lang/instrument/package-summary.html) 。基本实现是通过JVMTI的retransformClass/redefineClass进行method、body级的字节码更新，ASM、CGLib基本都是围绕这些在做动态性。但是针对Class的HotSwap一直没有动作（比如Class添加method、添加field、修改继承关系等等），为什么会这样呢？因为复杂度过高，且没有很高的回报。

### 2.4 Sonic如何解决Instrumentation的局限性

由于JVM限制，JDK 7和JDK 8都不允许改类结构，比如新增字段，新增方法和修改类的父类等，这对于Spring项目来说是致命的。比如开发同学想修改一个Spring Bean，新增一个@Autowired字段，此类场景在实际应用时很多，所以Sonic对此类场景的支持必不可少。

那么，具体是如何做到的呢？这里要提一下“大名鼎鼎”的Dcevm。Dcevm（DynamicCode Evolution Virtual Machine）是Java Hostspot的补丁（严格上来说是修改），允许（并非无限制）在运行环境下修改加载的类文件。当前虚拟机只允许修改方法体（Method，Body），而Decvm可以增加、删除类属性、方法，甚至改变一个类的父类，Dcevm是一个开源项目，遵从GPL 2.0协议。更多关于Dcevm的介绍，大家可以参考：[Wuerthinger10a](https://ssw.jku.at/Research/Papers/Wuerthinger10a/Wuerthinger10a.pdf)以及[GitHub Decvm](https://github.com/dcevm/dcevm)。

值得一提的是，在美团内部，针对Dcevm的安装，Sonic已经打通[HULK](http://mp.weixin.qq.com/s?__biz=MjM5NjQ5MTI5OA==&mid=2651750633&idx=1&sn=51a4b05deac592c4ccbf2dbcf709288b&chksm=bd1259a48a65d0b2d198239c03158e4f5eeace74241f8dc7f7d361974fa1e85bfbc8d3dfab88&scene=21#wechat_redirect)，集成发布镜像即可完成（本地热部署可结合插件功能实现一键安装热部署环境）。

## 3 Sonic热部署技术解析

### 3.1 Sonic整体架构模型

上一章节我们主要介绍了Sonic的组成。下图详细介绍了Sonic在运行期间各个组成部分的工作职责，由它们形成一整套完备的技术产品落地闭环方案：

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

图 9

### 3.2 Sonic功能流转

Sonic通过NIO监听本地文件变更，触发文件变更事件，例如Class新增、Class修改、Spring Bean重载等事件流程。下图展示了一次热部署单个文件的生命周期：

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

图 10

### 3.3 文件监听

Sonic首先会在本地和远程预定义两个目录，`/var/tmp/sonic/extraClasspath`和`/var/tmp/sonic/classes`。extraClasspath为Sonic自定义的拓展Classpath URL，classes为Sonic监听的目录，当有文件变更时，通过IDEA插件来部署到远程/本地，触发Agent的监听目录，来继续下面的热加载逻辑：

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

图 11

为什么Sonic不直接替换用户ClassPath下面的资源文件呢？因为考虑到业务方WAR包的API项目、Spring Boot、Tomcat项目、Jetty项目等，都是以JAR包来启动的，这样是无法直接修改用户的Class文件的。即使是用户项目可以修改，直接操作用户的Class，也会带来一系列的安全问题。

所以，Sonic采用拓展ClassPath URL路径来实现文件的修改和新增。并且存在这么一种场景，多个业务侧的项目引入相同的JAR包，在JAR里面配置MyBatis的XML和注解。在此类情况下，Sonic没有办法直接来修改JAR包中源文件，通过拓展路径的方式可以不需要关注JAR包，来修改JAR包中某一文件和XML。同理，采用此类方法可以进行整个JAR包的热替换。下面我们简单介绍一下Sonic的核心监听器，如下图所示：

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

图 12

### 3.4 JVM Class Reload

JVM的字节码批量重载逻辑，通过新的字节码二进制流和旧的Class对象生成ClassDefinition定义，instrumentation.redefineClasses（definitions），来触发JVM重载，重载过后将触发初始化时Spring插件注册的Transfrom。接下来，我们简单讲解一下Spring是怎么重载的。

新增class Sonic如何保证可以加载到Classloader上下文中？由于项目在远程执行，所以运行环境复杂，有可能是JAR包方式启动（Spring Boot），也有可能是普通项目，也有可能是War Web项目，针对此类情况Sonic做了一层Classloader URL拓展。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

图 13

User ClassLoader是框架自定义的ClassLoader统称，例如Jetty项目是WebAppclassLoader。其中Urlclasspath为当前项目的lib文件件下，例如Spring Boot项目也是从当前项目BOOT-INF/lib/路径中加载CLass等等，不同框架的自定义位置稍有不同。所以针对此类情况，Agent必须拿到用户的自定义Classloader，如果是常规方式启动的，比如普通Spring XML项目，借助Plus（美团内部服务发布平台）发布，此类没有自定义Classloader，是默认AppClassLoader，所以Agent在用户项目启动过程中，借助字节码增强的方式来获取到真正的用户Classloader。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

图 14

找到用户使用的子Classloader之后，通过反射的方式来获取Classloader中的元素Classpath，其中ClassPath中的URL就是当前项目加载Class时需要的所有运行时Class环境，并且包括三方的JAR包依赖等。

Sonic获取到URL数组，把Sonic自定义的拓展Classpath目录加入到URL数组首位，这样当有新增Class时，Sonic只需要将Class文件复制到拓展Classpath对应的包目录下面即可，当有其他Bean依赖新增的Class时，会从当前目录下面查找类文件。

为什么不直接对Appclassloader进行加强？而是对框架的自定义Classloader进行加强？

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

图 15

考虑这样一个场景，框架自定义类加载器中有ClassA，此时用户新增ClassB需要热加载，B Class里面有A的引用关系，如果增强AppClassLoader，初始化B实例时ClassLoader。loadclass首先从UserClassLoader开始加载ClassB的字节码，依靠双亲委派原则，B被Appclassloader加载，因为B依赖类A，所以当前AppClassLoader加载B一定是加载不到的，此时会抛出ClassNotFoundException异常。所以对类加载器拓展，一定要拓展最上层的类加载器，这样才会达到使用者想要的效果。

### 3.5 Spring Bean重载

Spring Bean Reload过程中，Bean的销毁和重启流程，主要内容如下图展示：

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

图 16

首先当修改Java Class D时，通过Spring ClasspathScan扫描校验当前修改的Bean是否Sprin Bean（注解校验），然后触发销毁流程（BeanDefinitionRegistry.removeBeanDefinition），此方法会将当前Spring上下文中的Bean D和依赖Spring Bean D的Bean C一并销毁，但是作用范围仅仅在当前Spring上下文。如果C被子上下文中的Bean B依赖，就无法更新子上下文中的依赖关系，当有系统请求时，Bean B中关联的Bean C还是热部署之前的对象，所以热部署失败。

因此，在Spring初始化过程中，需要维护父子上下文的对应关系，当子上下文变时若变更范围涉及到Bean B时，需要重新更新子上下文中的依赖关系，当有多上下文关联时需要维护多上下文环境，且当前上下文环境入口需要Reload。这里的入口是指：Spring MVC Controller、Mthrift和Pigeon，对不同的流量入口，采用不同的Reload策略。RPC框架入口主要操作为解绑注册中心、重新注册、重新加载启动流程等等，对Spring MVC Controller，主要是解绑和注册URL Mappping来实现流量入口类的变化切换。

### 3.6 Spring XML重载

当用户修改/新增Spring XML时，需要对XML中所有Bean进行重载。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

图 17

重新Reload之后，将Spring销毁后重启。需要注意的是：XML修改方式改动较大，可能涉及到全局的AOP的配置以及前置和后置处理器相关的内容，影响范围为全局，所以目前只放开普通的XML Bean标签的新增/修改，其他能力酌情逐步放开。

### 3.7 MyBatis 热部署

Spring MyBatis热部署的主要处理流程是在启动期间获取所有Configuration路径，并维护它和Spring Context的对应关系，在热部署Class、XML时去匹配Configuration，从而重新加载Configuration以达到热部署的目的。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

图 18

## 4 总结

### 4.1 热部署功能一览

上一章节主要讲述了Spring Bean、Spring MVC、MyBatis的重载流程，Sonic还支持其它常用的开发框架，丰富的框架支持和兼容能力是Sonic的基石，下面列举一些Sonic支持的常用的第三方框架：

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

图19 美团内部框架以及常用开源框架

截止目前，Sonic已经支持绝大部分常用第三方框架的热加载，常规业务开发几乎无需重启服务。并且在美团内部的成功率已经高达99.9%以上，真正地让热部署来代替常规部署构建成为一种可能。

### 4.2 IDE插件集成

Sonic也提供了功能强大的IDEA插件，让用户进行沉浸式开发，远程热部署也变得更加便利。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

图 20

### 4.3 推广使用情况

截止到发稿时，Sonic在美团使用人数3000+，应用项目数量2000+。该项目还获得了美团内部2020年下半年到家研发平台“最佳效率团队”奖。

## 5 作者简介

凯哥、占峰、李晗、龚炎、程骁、玉龙等，均来自美团/到家研发平台。

## 6 参考文章

\[1\] [基于Javassist和Javaagent实现动态切面](https://www.cnblogs.com/chiangchou/p/javassist.html)

\[2\] [Spring MVC 源码解析](https://blog.csdn.net/win7system/article/details/90674757)

\[3\] [Spring IOC源码解析](https://blog.csdn.net/zhanyu1/article/details/83023854)

\[4\] [MyBatis源码解析](https://www.cnblogs.com/javazhiyin/p/12340498.html)

\[5\] [Spring Boot源码解析](https://www.cnblogs.com/java-chen-hao/p/11829344.html)

\[6\] [Spring AOP源码解析](https://javadoop.com/post/spring-aop-source)

\[7\] [Spring事务源码解析](https://www.jianshu.com/p/622f60520674)

\[8\] [Cglib源码解析](https://blog.csdn.net/lpq374606827/article/details/79392658)

\[9\] [JDK Proxy源码解析](https://www.cnblogs.com/zyl2016/p/11841492.html)

\[10\] [Dcevm简介](https://ssw.jku.at/Research/Papers/Wuerthinger10a/Wuerthinger10a.pdf)

\[11\] [字节码增强技术探索](https://tech.meituan.com/2019/09/05/java-bytecode-enhancement.html)

\[12\] [Javassist API](https://github.com/jboss-javassist/javassist/wiki)

----------  END  ----------

**也许你还想看**

**| [](http://mp.weixin.qq.com/s?__biz=MjM5NjQ5MTI5OA==&mid=2651754955&idx=1&sn=8411133d2e5f22b9e2c5a34cdc67985d&chksm=bd1248868a65c1900dd1b7203ce17159740253df2324a208ea9c71ee764e1bde1ed2616d77ce&scene=21#wechat_redirect)** [Java中9种常见的CMS GC问题分析与解决](http://mp.weixin.qq.com/s?__biz=MjM5NjQ5MTI5OA==&mid=2651754955&idx=1&sn=8411133d2e5f22b9e2c5a34cdc67985d&chksm=bd1248868a65c1900dd1b7203ce17159740253df2324a208ea9c71ee764e1bde1ed2616d77ce&scene=21#wechat_redirect)**[](http://mp.weixin.qq.com/s?__biz=MjM5NjQ5MTI5OA==&mid=2651754955&idx=1&sn=8411133d2e5f22b9e2c5a34cdc67985d&chksm=bd1248868a65c1900dd1b7203ce17159740253df2324a208ea9c71ee764e1bde1ed2616d77ce&scene=21#wechat_redirect)**

**|** [Java线程池实现原理及其在美团业务中的实践](http://mp.weixin.qq.com/s?__biz=MjM5NjQ5MTI5OA==&mid=2651751537&idx=1&sn=c50a434302cc06797828782970da190e&chksm=bd125d3c8a65d42aaf58999c89b6a4749f092441335f3c96067d2d361b9af69ad4ff1b73504c&scene=21#wechat_redirect)

**|** [Java 动态调试技术原理及实践](http://mp.weixin.qq.com/s?__biz=MjM5NjQ5MTI5OA==&mid=2651750923&idx=2&sn=55102c505cd57f185219d35b94de50d5&chksm=bd125b468a65d2505706c962af707e009105cb7f0b78bbd27c6dbcd7e895c2a7fe631953ae46&scene=21#wechat_redirect)

**阅读更多**

______________________________________________________________________

[前端](http://mp.weixin.qq.com/s?__biz=MjM5NjQ5MTI5OA==&mid=2651765958&idx=1&sn=8201546812e5a95a2bee9dffc6d12f00&chksm=bd12658b8a65ec9de2f5be1e96796dfb3c8f1a374d4b7bd91266072f557caf8118d4ddb72b07&scene=21#wechat_redirect) **|**  [](https://t.1yb.co/jo7v)[算法](http://mp.weixin.qq.com/s?__biz=MjM5NjQ5MTI5OA==&mid=2651765981&idx=1&sn=c2dd86f15dee2cbbc89e27677d985060&chksm=bd1265908a65ec86d4d08f7600d1518b61c90f6453074f9b308c96861c045712280a73751c73&scene=21#wechat_redirect) **|** [后端](http://mp.weixin.qq.com/s?__biz=MjM5NjQ5MTI5OA==&mid=2651765982&idx=1&sn=231b41f653ac7959f3e3b8213dcec2b0&chksm=bd1265938a65ec85630c546169444d56377bc2f11401d251da7ca50e5d07e353aa01580c7216&scene=21#wechat_redirect) **|** [数据](http://mp.weixin.qq.com/s?__biz=MjM5NjQ5MTI5OA==&mid=2651765964&idx=1&sn=ab6d8db147234fe57f27dd46eec40fef&chksm=bd1265818a65ec9749246dd1a2eb3bf7798772cc4d5b4283b15eae2f80bc6db63a1471a9e61e&scene=21#wechat_redirect)

[安全](http://mp.weixin.qq.com/s?__biz=MjM5NjQ5MTI5OA==&mid=2651765965&idx=1&sn=37e0c56c8b080146ce5249243bfd84d8&chksm=bd1265808a65ec96d3a2b2c87c6e27c910d49cb6b149970fb2db8bf88045a0a85fed2e6a0b84&scene=21#wechat_redirect) **|** [Android](http://mp.weixin.qq.com/s?__biz=MjM5NjQ5MTI5OA==&mid=2651765972&idx=1&sn=afe02ec92762c1ce18740d03324c4ac3&chksm=bd1265998a65ec8f10d5f58d0f3681ddfc5325137218e568e1cda3a50e427749edb5c6a7dcf5&scene=21#wechat_redirect) **|** [iOS](http://mp.weixin.qq.com/s?__biz=MjM5NjQ5MTI5OA==&mid=2651765973&idx=1&sn=32a23bf1d278dda0398f993ab60a697e&chksm=bd1265988a65ec8e630ef4d24b4946ab6bd7e66702c1d712481cf3c471468a059c470a14c30d&scene=21#wechat_redirect)  **|** [运维](http://mp.weixin.qq.com/s?__biz=MjM5NjQ5MTI5OA==&mid=2651765963&idx=1&sn=a3de9ef267d07d94118c1611776a4b28&chksm=bd1265868a65ec906592d25ad65f2a8516338d07ec3217059e6975fc131fc0107d66a8cd2612&scene=21#wechat_redirect) **|** [测试](http://mp.weixin.qq.com/s?__biz=MjM5NjQ5MTI5OA==&mid=2651765974&idx=1&sn=763c1e37d04acffd0142a2852ecfb000&chksm=bd12659b8a65ec8dfcfeb2028ef287fae7c38f134a665375ba420556ce5d2e4cf398147bd12e&scene=21#wechat_redirect)

![](http://mmbiz.qpic.cn/mmbiz_png/hEx03cFgUsVGibnsaEib3aNlqF0tOrA2RGEmNSbia2nnohE4Tpf95UyTiaSjDVbHRfY8WNBeTuLLTaVdSckkNyEx1Q/300?wx_fmt=png&wxfrom=19)

**美团技术团队**

10000+工程师，如何支撑中国领先的生活服务电子商务平台？数亿消费者、数百万商户、2000多个行业背后是哪些技术在支撑？这里是美团、大众点评、美团外卖、美团优选等技术团队对外交流的窗口。

548篇原创内容

公众号

到家14

后台38

Java4

热部署1

Spring1

到家 · 目录

上一篇美团跨端一体化富文本管理技术实践下一篇终端新玩法：“零代码”的剧本式引导

阅读 2.5万

​

写留言

**留言 29**

- 王克远

  2022年3月17日

  赞18

  期待开源

  置顶

  美团技术团队

  作者2022年3月18日

  赞9

  今后有机会会考虑开源，大家可以先关注Dcevm、Spring Loader等优秀的类似产品。

- 李爽

  2022年3月17日

  赞9

  很好用，特别是一些相对复杂的应用，热部署很方便![[社会社会]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![[社会社会]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

  置顶

- 赵泽恩

  2022年3月17日

  赞10

  sonic的推广离不开某位老哥曾经在各种大象群里疯狂发软文哈哈哈

  置顶

- 喝咖啡的猫

  2022年3月17日

  赞21

  非常期待开源![/::B](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

- 红色的红🙈🎯

  2022年3月17日

  赞9

  这文章太干了，处处是知识点，能够把各种应用框架打通，且提供idea插件闭环，很是佩服，这是一个大工程

- 丰富

  2022年3月17日

  赞9

  看起来很牛的样子，有2个问题：1. 支持proxy吗，很多公司是不允许开发机直连服务器的？2. 有计划开源吗？

- 履霜坚冰至

  2022年3月17日

  赞7

  一直在用jrebel

- 王大圆🌸

  2022年3月17日

  赞6

  凯哥最帅！！！![[皱眉]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

- 合

  2022年3月17日

  赞6

  咨询一个问题：如果业务依赖于通过某个 class 的 static 字段，且该字段是个 map，热更新以后业务逻辑能保证正常吗？热更新是否会把这个 map 中缓存的值清理掉呢？

  美团技术团队

  作者2022年3月18日

  赞4

  热更新之后不会保留原有堆内存中的数据，Map里面的值也会相应的清空，热部署关注“新”代码逻辑，不关注“旧”数据状态。

- 拖雷鸡鸭

  2022年3月17日

  赞5

  目前应该还没有开源吧，求开源推广![[嘿哈]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

- Sparkle

  2022年3月17日

  赞4

  完美解决了我们的痛点，顺便问一下有没有go版本的![[旺柴]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

- Luck

  2022年3月17日

  赞3

  凯哥牛逼，不愧是多年前就很牛逼的技术大神！

- 欢

  2022年3月17日

  赞3

  期待开源

- 🎅 🌞

  2022年4月22日

  赞2

  强烈期待开源，天下苦jvm久已

- 滚石

  2022年3月18日

  赞2

  看起来很nb的样子，批量更新会不会出现更新失败、或者短时间内不一致的情况？

  美团技术团队

  作者2022年3月18日

  赞2

  不会出现不一致的情况，但是在热部署期间如果有流量打进来会出现错误。

- 沈军禹

  2022年3月18日

  赞2

  热部署的时候，如果有请求过来怎么办？

  美团技术团队

  作者2022年3月18日

  赞2

  热部署时，流量打进来，在瞬时内，MVC服务有可能会出现404，RPC服务上游调用方可能会无法找到可用服务。

- Dong

  2022年3月18日

  赞2

  ![😂](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)确实给力，对于我们这种懒人，写完代码发到多环境，流水线自动触发部署，涉及到编译和发布！整体流程比较大的项目基本上是10-20分钟左右，导致该一行代码也是这种成本，真滴无语！提高效率yyds！但是要是能支持go和c++就行了

- PerJoker

  2022年3月17日

  赞2

  dcevm也是有一定限制的，如果仅仅是本地应用热部署，可以使用idea的hotswap agent插件，然后把本地的jvm里面的dll文件替换成dcevm的。dcevm的限制，完全可以看hot swap agent 的官方介绍。

- 番薯和米饭一起吃出辣椒的味道

  2022年3月17日

  赞2

  非常好用，节省大量部署代码的时间，提升工作效率

- alex

  2022年3月17日

  赞2

  试了一下，果然牛逼。对于不适合本地调试的大内存应用来说，简直是救星![[色]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![[色]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![[色]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

- ⊙_⊙

  2022年3月23日

  赞1

  美图的jvm文章真不错，这种基础框架真心对开发来说真是好东西。希望后面多介绍下框架架构![[微笑]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)和实现

- Winterer

  2022年3月17日

  赞1

  这是需要用dcevm替换原生jvm吗？如果用原生jvm的话有办法在运行时attach进程进行热更新吗？

  美团技术团队

  作者2022年3月18日

  赞1

  是的，需要在JDK中打补丁，用原生JDK可以修改MyBatis等不改变类结构的热更新操作，无法通过Attach的方式，因为在项目启动期间，需要字节码增强来维护一些框架的基础数据。

- 莫道

  2022年3月17日

  赞1

  听起来是浩大的工程量

- 石头剪刀布💪

  广东2023年9月1日

  赞

  太好了，太好了。如果能开源，对中国开源行业是一件大事情，对全世界的Java用户都非常有价值。。

已无更多数据

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/hEx03cFgUsVGibnsaEib3aNlqF0tOrA2RGEmNSbia2nnohE4Tpf95UyTiaSjDVbHRfY8WNBeTuLLTaVdSckkNyEx1Q/300?wx_fmt=png&wxfrom=18)

美团技术团队

1493895

29

写留言

**留言 29**

- 王克远

  2022年3月17日

  赞18

  期待开源

  置顶

  美团技术团队

  作者2022年3月18日

  赞9

  今后有机会会考虑开源，大家可以先关注Dcevm、Spring Loader等优秀的类似产品。

- 李爽

  2022年3月17日

  赞9

  很好用，特别是一些相对复杂的应用，热部署很方便![[社会社会]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![[社会社会]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

  置顶

- 赵泽恩

  2022年3月17日

  赞10

  sonic的推广离不开某位老哥曾经在各种大象群里疯狂发软文哈哈哈

  置顶

- 喝咖啡的猫

  2022年3月17日

  赞21

  非常期待开源![/::B](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

- 红色的红🙈🎯

  2022年3月17日

  赞9

  这文章太干了，处处是知识点，能够把各种应用框架打通，且提供idea插件闭环，很是佩服，这是一个大工程

- 丰富

  2022年3月17日

  赞9

  看起来很牛的样子，有2个问题：1. 支持proxy吗，很多公司是不允许开发机直连服务器的？2. 有计划开源吗？

- 履霜坚冰至

  2022年3月17日

  赞7

  一直在用jrebel

- 王大圆🌸

  2022年3月17日

  赞6

  凯哥最帅！！！![[皱眉]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

- 合

  2022年3月17日

  赞6

  咨询一个问题：如果业务依赖于通过某个 class 的 static 字段，且该字段是个 map，热更新以后业务逻辑能保证正常吗？热更新是否会把这个 map 中缓存的值清理掉呢？

  美团技术团队

  作者2022年3月18日

  赞4

  热更新之后不会保留原有堆内存中的数据，Map里面的值也会相应的清空，热部署关注“新”代码逻辑，不关注“旧”数据状态。

- 拖雷鸡鸭

  2022年3月17日

  赞5

  目前应该还没有开源吧，求开源推广![[嘿哈]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

- Sparkle

  2022年3月17日

  赞4

  完美解决了我们的痛点，顺便问一下有没有go版本的![[旺柴]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

- Luck

  2022年3月17日

  赞3

  凯哥牛逼，不愧是多年前就很牛逼的技术大神！

- 欢

  2022年3月17日

  赞3

  期待开源

- 🎅 🌞

  2022年4月22日

  赞2

  强烈期待开源，天下苦jvm久已

- 滚石

  2022年3月18日

  赞2

  看起来很nb的样子，批量更新会不会出现更新失败、或者短时间内不一致的情况？

  美团技术团队

  作者2022年3月18日

  赞2

  不会出现不一致的情况，但是在热部署期间如果有流量打进来会出现错误。

- 沈军禹

  2022年3月18日

  赞2

  热部署的时候，如果有请求过来怎么办？

  美团技术团队

  作者2022年3月18日

  赞2

  热部署时，流量打进来，在瞬时内，MVC服务有可能会出现404，RPC服务上游调用方可能会无法找到可用服务。

- Dong

  2022年3月18日

  赞2

  ![😂](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)确实给力，对于我们这种懒人，写完代码发到多环境，流水线自动触发部署，涉及到编译和发布！整体流程比较大的项目基本上是10-20分钟左右，导致该一行代码也是这种成本，真滴无语！提高效率yyds！但是要是能支持go和c++就行了

- PerJoker

  2022年3月17日

  赞2

  dcevm也是有一定限制的，如果仅仅是本地应用热部署，可以使用idea的hotswap agent插件，然后把本地的jvm里面的dll文件替换成dcevm的。dcevm的限制，完全可以看hot swap agent 的官方介绍。

- 番薯和米饭一起吃出辣椒的味道

  2022年3月17日

  赞2

  非常好用，节省大量部署代码的时间，提升工作效率

- alex

  2022年3月17日

  赞2

  试了一下，果然牛逼。对于不适合本地调试的大内存应用来说，简直是救星![[色]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![[色]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![[色]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

- ⊙_⊙

  2022年3月23日

  赞1

  美图的jvm文章真不错，这种基础框架真心对开发来说真是好东西。希望后面多介绍下框架架构![[微笑]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)和实现

- Winterer

  2022年3月17日

  赞1

  这是需要用dcevm替换原生jvm吗？如果用原生jvm的话有办法在运行时attach进程进行热更新吗？

  美团技术团队

  作者2022年3月18日

  赞1

  是的，需要在JDK中打补丁，用原生JDK可以修改MyBatis等不改变类结构的热更新操作，无法通过Attach的方式，因为在项目启动期间，需要字节码增强来维护一些框架的基础数据。

- 莫道

  2022年3月17日

  赞1

  听起来是浩大的工程量

- 石头剪刀布💪

  广东2023年9月1日

  赞

  太好了，太好了。如果能开源，对中国开源行业是一件大事情，对全世界的Java用户都非常有价值。。

已无更多数据
