# 

随风而行aa 看雪学苑

_2021年10月18日 18:05_

!\[\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

本文为看雪论坛精华文章\
看雪论坛作者ID：随风而行aa

1

# **简介**

最近在总结Android APP漏洞挖掘方面的知识，上篇帖子Android漏洞挖掘三板斧——drozer+Inspeckage(Xposed)+MobSF向大家初步的介绍了Android APP漏洞挖掘过程中常见的工具，这里也是我平时使用过程中比较常用的三套件，今天我们来逐步学习和复现Android中 Activity漏洞挖掘部分知识，每个漏洞挖掘部分，我们都会选择具有代表性的样本案例给大家演示。

2

# **Activity漏洞初步介绍**

### 1.Activity基本介绍

在学习Activity的漏洞挖掘之前，我们先对Activity的基本运行原理有一个初步的认识。

#### （1）Intent 调用Activity

首先，我们要启动Activity，完成各个Activity之间的交互，我们需要使用Android中一个重要的组件Intent。

```
Intent是各个组件之间交互的一种重要方式，它不仅可以指明当前组件想要执行的动作，而且还能在各组件之间传递数据。Intent一般可用于启动Activity、启动Service、发送广播等场景。
```

Intent一般分为显式Intent和隐私Intent：

显示Intent打开Activity：

```
Intent intent = new Intent(MainActivity.class,SecondActivity.class); //实例化Intent对象
```

隐式Intent打开Activity:

隐式Intent并不指明启动那个Activity而是指定一系列的action和category，然后由系统去分析找到合适的Activity并打开，action和category一般在AndroidManifest中指定。

```
<activity android:name=".SecondActivity">
```

只有<action>和<category>中的内容能够匹配上Intent中指定的action和category时，这个活动才能响应Intent。

```
Intent intent = Intent("com.example.test.ACTION_START")；
```

我们这里只传入了ACTION_START，这是因为android.intent.category.DEFAULT是一种默认的category，在调用startActivity()时会自动将这个category添加到Intent中，注意：Intent中只能添加一个action，但是可以添加多个category。

对于含多个category情况，我们可以使用addCategory()方法来添加一个category。

```
intent.addCategory("com.example.test.MY_CATEGORY");
```

隐私Intent打开程序外Activity：

例如我们调用系统的浏览器去打开百度网址：

```
Intent intent = new Intent(Intent.ACTION_VIEW);
```

Intent.ACTION_VIEW是系统内置的动作，然后将https://www.baidu.com通过Uri.parse()转换成Uri对象，传递给intent.setData(Uri uri)函数。

与此对应，我们在<intent-filter>中配置<data>标签，用于更加精确指定当前活动能够响应什么类型的数据：

```
android:scheme：用于指定数据的协议部分，如https
```

只有当<data>标签中指定的内容和Intent中携带的data完全一致时，当前Activity才能响应该Intent。下面我们通过设置data，让它也能响应打开网页的Intent。

```
<activity android:name=".SecondActivity">
```

我们就能通过隐式Intent的方法打开外部Activity。

```
Intent intent = new Intent(Intent.ACTION_VIEW);
```

#### 

#### （2）Activity中传递数据

向下一个活动传递数据：

Intent传递字符串：

```
Intent intent = new Intent(MainActivity.class,SecondActivity.class);
```

Intent接收字符串：

```
Intent intent = getIntent();
```

返回数据给上一个活动：

Android 在返回一个活动可以通过Back键，也可以使用startActivityForResult()方法来启动活动，该方法在活动销毁时能返回一个结果给上一个活动。

```
Intent intent = new Intent(MainActivity.class,SecondActivity.class);
```

我们在SecondActivity中返回数据：

```
Intent intent = new Intent();
```

当活动销毁后，就会回调到上一个活动，所以我们需要在MainActivity中接收。

```
@Override
```

如果我们要实现Back返回MainActivity，我们需要在SecondActivity中重写onBackPressed()方法。

```
@Override
```

#### 

#### （3）Activity的生命周期

Activity类中定义了7个回调方法，覆盖了Activity声明周期的每一个环节：

```
onCreate()：在Activity第一次创建时调用
```

生命周期调用图：

!\[\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

```
我们可以将活动分为3中生存期：
```

#### 

#### （4）Activity的启动模式

我们这里之所以要介绍Activity的启动模式，是因为Activity界面劫持就是根据Activity的运行特点所实现的。

Activity一共有四种启动模式：standard模式、singleTop模式、singleTask模式、singleInstance模式。下面我们简单介绍一下：

standard模式

如果不显示指定启动模式，那么Activity的启动模式就是standard，在该模式下不管Activity栈中有无Activity，均会创建一个新的Activity并入栈，并处于栈顶的位置。

singleTop模式

```
（1）启动一个Activity，这个Activity位于栈顶，则不会重新创建Activity,而直接使用，此时也不会调用Activity的onCreate()，因为并没有重新创建Activity
```

singleTask模式

```
如果准备启动的ActivityA的启动模式为singleTask的话，那么会先从栈中查找是否存在ActivityA的实例：
```

singleInstance模式

```
指定singleInstance模式的Activity会启动一个新的返回栈来管理这个Activity（其实如果singleTask模式指定了不同的taskAffinity，也会启动一个新的返回栈
```

### 

### 2.Activity 漏洞种类和危害

我们在上文中详细介绍了Activity的运行原理，接下来我们了解一些Activity的漏洞种类和应用的安全场景。

#### （1）Activity的漏洞种类

!\[\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

#### （2）Activity安全场景和危害

```
Activity的组件导出，一般会导致的问题：Android Browser Intent Scheme URLs的攻击手段
```

## 

3

# **Activity漏洞原理分析和复现**

### 1、越权绕过

#### （1）原理介绍

```
在Android系统中，Activity默认是不导出的，如果设置了exported = "true" 这样的关键值或者是添加了<intent-filter>这样的属性，那么此时Activity是导出的，就会导致越权绕过或者是泄露敏感信息等安全风险。
```

#### 

#### （2）漏洞复现

样本 sieve.apk drozer.apk

!\[\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

首先，我们需要配置drozer的基本环境，具体配置操作，参考：Android漏洞挖掘三板斧——drozer+Inspeckage(Xposed)+MobSF\_（https://bbs.pediy.com/thread-269196.htm）\_

手机端打开代理，开启31415端口。

```
adb forward tcp:31415 tcp:31415
```

!\[\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

!\[\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

我们尝试使用drozer去越权绕过该界面，首先，我们先列出程序中所有的APP 包：‍

```
run app.package.list
```

我们通过查询字段，可以快速定位到sieve的包名。

!\[\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

然后，我们去查询目标应用的攻击面：

```
run app.package.attacksurface com.mwr.example.sieve
```

!\[\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

我们可以看出，有三个activity是被导出的，我们再具体查询暴露activity的信息。

```
run app.activity.info -a  com.mwr.example.sieve
```

!\[\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

说明我们可以通过强制跳转其他两个界面，来实现越权绕过。

```
run app.activity.start --component com.mwr.example.sieve com.mwr.example.sieve.PWList
```

!\[\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

说明我们成功的实现了越权绕过。

#### （3）防护策略

```
防护策略：
```

### 

### 2、钓鱼欺诈/Activity劫持

#### 

#### （1）原理介绍

```
原理介绍：
```

!\[\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

!\[\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

#### （2）漏洞复现

在进行Android 界面劫持过程中，我发现根据Android版本的变化情况，目前不同Android版本实现的功能代码有一定的差异性，再经过多次的学习和总结后，我复现而且改进了针对Android 6.0界面劫持的功能代码，并对不同版本的页面劫持做了一个初步的总结，下面是具体实验的详细过程:

首先我们新建一个服务类HijackingService.class，然后在MainActivity里面启动这个服务类：

```
public class MainActivity extends AppCompatActivity {
```

我们可以看到程序一旦启动，就会启动HijackingService.class。

然后我们编写一个HijackingApplication类，主要负责添加劫持类别，清除劫持类别，判断是否已经劫持。

```
public class HijackingApplication{ 
```

说明：这个类的主要功能是，保存已经劫持过的包名，防止我们多次劫持增加暴露风险。

我们为了实现开机启动服务，新建一个广播类：

```
public class HijackingReciver extends BroadcastReceiver {
```

然后我们编写劫持类 HijackingService.class。

```
private boolean hasStart = false;
```

我们编写劫持类中，最关键的就是如何获取当前的前台进程和遍历正在运行的进程，这也是Android版本更新后，导致不同版本劫持差异的主要原因，对这里我做了一个初步的总结：

```
注意：
```

我们编写获取当前目标进程的代码：

```
public class ForegroundProcess {
```

我们继续编写劫持替换的测试类：

```
public class SecondActivity extends AppCompatActivity {
```

最后在我们的配置文件中加入相应的权限和配置信息：

```
<?xml version="1.0" encoding="utf-8"?>
```

我们需要将服务的时间设置成6秒，避免程序界面还未加载就劫持了。

效果演示：

我们编写劫持类安装，打开：

!\[\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

我们可以发现劫持类在后台运行：

!\[\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

我们打开目标程序：

!\[\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

等待5秒，然后劫持成功，这个时间我们可以在代码段调整：

!\[\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

这样我们成功完成了对目标程序劫持，这里我只编写了一个简易的界面，大家可以编写更加复杂的界面，这主要是针对Android 6.0平台的劫持，各位也可以试试其他版本的平台。

#### （3）安全防护

```
如果真的爆发了这种恶意程序，我们并不能在启动程序时每一次都那么小心去查看判断当前在运行的是哪一个程序，当android:noHistory="true"时上面的方法也无效
```

!\[\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

通过它提示程序进入后台来提示用户。

### 

### 3、隐私启动Intent包含敏感数据

#### 

#### （1）原理介绍

```
1.背景知识：Intent可分为隐私(implicitly)和显式(explicitly)两种
```

!\[\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

```
我们有一个应用A,采用Intent隐式传递,它的动作是"X",此时还有一个应用B,动作也是X,我们在启动的时候，通过Intent隐式传递，就会同时弹出两个界面，我们就不知道到底启动A还是B
```

因为现在这种漏洞在Android版本更新后，基本很少出现了，所以这里就不做复现和安全防护了。

### 4、拒绝服务攻击

#### （1）原理介绍

```
原理介绍：
```

提到拒绝服务攻击，我们就不得不讲一下Android外部程序的调用方法：

```
总结：
```

#### 

#### （2）漏洞复现

我们查看一个目标应用的AndroidManifest.xml文件：

```
<activity android:label="@string/app_name" android:name=".MainLoginActivity" android:excludeFromRecents="true" android:launchMode="singleTask" android:windowSoftInputMode="adjustUnspecified|stateVisible|adjustResize">
```

我们编写一个简易的APP程序，对目标程序进行拒绝服务攻击。

```
Intent intent = new Intent();
```

这里我们传入一个空字符，使其产生错误。

当然我们还可以使用我们的神器drozer来进行攻击。

!\[\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

远程拒绝服务攻击：

参考网址：http://rui0.cn/archives/30

还有其他类型的拒绝服务攻击，大家可以参考博客：

#### 

#### 参考网址_https://blog.csdn.net/myboyer/article/details/44940811utm_term=Activity%E6%8B%92%E7%BB%9D%E6%9C%8D%E5%8A%A1&utm_medium=distribute.pc_aggpage_search_result.none-task-blog-2~all~sobaiduweb~default-1-44940811&spm=3001.4430\_

#### 

#### （3）安全防护

```
安全防护：
```

## 

4

# **实验总结**

写到这里，这个帖子总算写完了，对Android的Activity漏洞挖掘的总结过程中，我又再一次将Android 的Activity组件运行的基本原理熟悉了一遍，学习就是不断的总结提高把，可能在编写的过程中，还存在很多不足地方，就请各位大佬指教了。

## **参考网址：**

```
Android 第一行代码
```

!\[\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**看雪ID：随风而行aa**

https://bbs.pediy.com/user-home-905443.htm

\*本文由看雪论坛 随风而行aa 原创，转载请注明来自看雪社区

[!\[\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458395632&idx=1&sn=34f2b52324f54626d84b46a430f35ed2&chksm=b18f137a86f89a6cb696f51c149732d6d1b428cde53e1fe8bb209ddc228c3774094515af9c79&scene=21#wechat_redirect)

[!\[\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)](https://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458387399&idx=2&sn=38495add2a3a3677b2c436581c07e432&scene=21#wechat_redirect)

**#** **往期推荐**

1.[少量虚假控制流混淆后的算法还原案例](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458397992&idx=1&sn=f2947a899d2e97478db9a0fad48151c2&chksm=b18f1da286f894b4d6609f30ec8e32a9fb250c9f2f4b406509065e33aab0e80bd894d6f3bbbf&scene=21#wechat_redirect)

2.[ollvm反混淆学习](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458397340&idx=2&sn=f06ad669a6f38e4d556ff043d4aa9517&chksm=b18f1a1686f893007e3241cff499add9db697828732c7a92543b3011cff4bfec61e9f43bd11f&scene=21#wechat_redirect)

3.[殊途同归的CVE-2012-0774 TrueType字体整数溢出漏洞分析](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458397272&idx=1&sn=9c179559e63b86fd4d985957fdcf025c&chksm=b18f1ad286f893c43956da0099db1350f9155cfc4bf09386e34963210649a0e97a723376ad6e&scene=21#wechat_redirect)

4.[PHP反序列化漏洞基础](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458397245&idx=2&sn=d0b40b359f56c55e4db3be6d02545de2&chksm=b18f1ab786f893a1020366fd743fbedbafe1017412bff2a927eea022da170425492025797db9&scene=21#wechat_redirect)

5.[栈溢出漏洞利用（绕过ASLR）](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458396195&idx=2&sn=3517949e465cddf16e347f801bfebe1f&chksm=b18f16a986f89fbf85b00510bee4577728c95cc58d9be79a2b5f09896c5c44f691c637791853&scene=21#wechat_redirect)

6.[大杀器Unidbg真正的威力](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458395762&idx=2&sn=f9662663e130096ffd01c86658813742&chksm=b18f14f886f89dee2fbc6e37a4aae1252e578f2a4abeae40b238a3f850069bd25c10a226000e&scene=21#wechat_redirect)

!\[\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

公众号ID：ikanxue

官方微博：看雪安全

商务合作：wsc@kanxue.com

!\[\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**球分享**

!\[\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**球点赞**

!\[\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**球在看**

!\[\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

点击“阅读原文”，了解更多！

阅读原文

阅读 2366

​
