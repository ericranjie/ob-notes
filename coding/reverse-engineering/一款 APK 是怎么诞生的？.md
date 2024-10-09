# 

原创 腾讯程序员 腾讯技术工程

_2021年09月28日 19:58_

![图片](https://mmbiz.qpic.cn/mmbiz_gif/j3gficicyOvasIjZpiaTNIPReJVWEJf7UGpmokI3LL4NbQDb8fO48fYROmYPXUhXFN8IdDqPcI1gA6OfSLsQHxB4w/640?wx_fmt=gif&wxfrom=13&tp=wxpic)

作者：hockeyli，腾讯 PCG 客户端开发工程师

### 一、 APK 组成解析

在开始解析 Android 构建流程之前，我们先来看下构建的最终产物 APK 的整体组成：

![](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvatVXIXkwUJKf2Aeicx3wiaJpM6uR0Lay9zAr3YV0ft0KTeQx8GJg0I7NOzzvIicQGvlFL96piaHL7KK3g/640?wx_fmt=jpeg&wxfrom=13&tp=wxpic)

APK 主要由五个部分组成，分别是：

- Dex：.class 文件处理后的产物，Android 系统的可执行文件

- Resource：资源文件，主要包括 layout、drawable、animator，通过 R.XXX.id 引用

- Assets：资源文件，通过 AssetManager 进行加载

- Library：so 库存放目录

- META-INF：APK 签名有关的信息

#### 1.1 Apk 分析工具

工欲善其事，必先利其器，既然想分析 APK 必然少不了好用的工具。

**① Android Studio 自带的 APK 分析器**

通过 APK 分析器，我们可以完成这些操作：

- 查看 APK 中文件（如 DEX 和 Android 资源文件）的绝对大小和相对大小

- 了解 DEX 文件的组成

- 快速查看 APK 中文件（如 AndroidManifest.xml）的最终版本

- 对两个 APK 进行并排比较

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvatVXIXkwUJKf2Aeicx3wiaJpMeKLjJ4TbWQHOia34geCYvvcZbFT7J0KMMibjfw4mjQxp4uDm4GX2nia3w/640?wx_fmt=jpeg&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvatVXIXkwUJKf2Aeicx3wiaJpMicGicxWqdYTKzmvVZZbXBA6Rn3IPnlOBrIDbROTvqxMiaia7Zr9ibRSVYew/640?wx_fmt=jpeg&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

**② ClassyShark** 可以做为 AS 自带 APK 分析器的补充，帮我们分析 dex 中的详细数据，以及查看 APK 中的总方法数以及各个模块的方法数分布。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvatVXIXkwUJKf2Aeicx3wiaJpMw0QOBiap0blOoFSNNKibJ9iad4MIicr9PibxxSvv017yn9OtokqP8d6bh5A/640?wx_fmt=jpeg&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvatVXIXkwUJKf2Aeicx3wiaJpMluQ2d2wUBVhfIWfic5QssypszYicBMspiaK5UqNcCpncDRKFH6gdXic3JQ/640?wx_fmt=jpeg&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

#### 1.2 Dex 知识点拓展

当我们在 Android 查看一个 APK 的时候，可以看到右上角有 Defined Methods 和 Referenced Methods，但大多数人可能不知道这两者的区别，这里简单说明下：

Defined Methods：在这个 Dex 中定义的方法；Referenced Methods：Defined Methods 以及 Defined Methods 引用到的方法。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

Android 有 64K 引用限制，当 type_ids、method_ids 或者 field_ids 超过 65536（64 * 1024）的时候，需要进行 dex 分包，为了 Dex 的数量尽可能少，我们需要尽量实现「Dex 信息有效率」的提升。

`Dex 信息有效率 = Defined Methods 数量 / Referenced Methods 数量   `

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

### 二、 构建源码导读

当我们用 Android Studio 进行安装包构建的时候，会发现其实是运行了一连串的 Task，也就是说其实是这些 task 的配合，最终构建出我们的 APK 的。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

#### 2.1 源码引入

如果我们想更了解 Android 的构建流程，对于相关的源码肯定是要有所了解的。那我们如何看到这些 Task 相关的源码呢，我们知道 Android 是用 Gradle 进行构建的，也就意味着这些 task 其实都是放在 Gradle 中，我们想看 Gradle 中源码的话，可以在 build.gradle 将 Gradle 进行编译。

`compileOnly "com.android.tools.build:gradle:3.0.1"   `

编译完之后，可以在 ApplicationTaskManager#createTasksForVariantScope 中找到创建这些 Task 相关的代码，也就意味着顺藤摸瓜找到这些 Task 的真正实现逻辑。

#### 2.2 BuildConfig Task 详解

这里以 BuildConfig 文件的生成为例，来梳理下如何查看某个 task 的代码逻辑。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

生成 BuildConfig 文件，是通过 ApplicationTaskManager 中通过 createBuildConfigTask 来创建对应的 task。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

顺着代码逻辑，我们找到最终真正实现这个逻辑的是：GenerateBuildConfig 这个 task，GenerateBuildConfig 是继承自 BaseTask，这里有个小技巧是，Task 中真正的执行逻辑都是在带着 @TaskAction 注解的方法上的，所以我们能很快找到对应的 generate() 方法。可以看到生成 BuildConfig 整体的逻辑还是比较简单的，其实就是将 build.gradle 中自带的属性以及我们自定义的属性进行读取，然后通过 JavaWriter 生成对应的 BuildConfig 文件。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

#### 2.3 获取所有 task 对应的类名

看到上面的例子，可能有些人会抛出一个疑问就是那我们怎么确定构建中执行的 task 具体对应哪个类呢，这里提供一个小技巧，其实我们可以在 taskGraph 构建完成之后，将所有 task name 以及对应的 class 进行打印。例如在 build.gradle 中加入这个代码之后，我们在运行的时候，就会把 task 所对应的类名也都一起打印出来。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

### 三、构建流程梳理

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

可以看到 Android 构建中会涉及到多个工具，我们可以通过 open $ANDROID_HOME/build-tools 来查看相关的构建工具。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

### 四、手动构建 APK

最后我们通过命令行来手动打包一个可执行的 APK，能让我们对 APK 构建的理解更加深入。首先需要准备下 代码、资源文件、AndroidManifest 这些构建 APK 的必要文件。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

① 通过 aapt2 compile 将 res 资源编译成 .flat 的二进制文件：

`aapt2 compile -o build/res.zip --dir res   `

② 通过 aapt2 link 将 .flat 和 AndroidManifest 进行连接，转化成不包含 dex 的 apk 和 R.java：

`aapt2 link build/res.zip -I $ANDROID_HOME/platforms/android-30/android.jar --java build --manifest AndroidManifest.xml -o build/app-debug.apk   `

③ 通过 javac 将 Java 文件编译成 .class 文件：

`javac -d build -cp $ANDROID_HOME/platforms/android-30/android.jar com/**/**/**/*.java   `

④ 通过 d8 将 .class 文件转化成 dex 文件：

`d8 --output build/ --lib $ANDROID_HOME/platforms/android-30/android.jar build/com/tencent/hockeyli/androidbuild/*.class   `

⑤ 合并 dex ⽂件和资源⽂件：

`zip -j build/app-debug.apk build/classes.dex   `

⑥ 对 apk 通过 apksigner 进行签名：

`apksigner sign -ks ~/.android/debug.keystore build/appdebug.apk   `

**腾讯程序员视频号最新视频**

**欢迎点赞**

腾讯程序员

，赞169

阅读 1.0万

​

写留言

[](javacript:;)

![](http://mmbiz.qpic.cn/sz_mmbiz_png/j3gficicyOvauPPfL7J2AVERiaoMJy9NBIwbJE2ZRJX7FZ2Dx7IibtTwdlqYSqTZTCsXkDS2jvNF8wWJKcibxXtOHng/300?wx_fmt=png&wxfrom=18)

腾讯技术工程

59232

写留言

写留言

**留言**

暂无留言
