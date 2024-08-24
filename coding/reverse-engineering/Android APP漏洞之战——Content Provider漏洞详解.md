# 

随风而行aa 看雪学苑

 _2021年10月20日 18:10_

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/kR3MmOkZ9r61GWufXMibOVQTexiaf4YUBsJeLnHnvJogQLsiauxMDiaNGtW1cdI5mRZ1OplDLM1q9KLp4y9CAfOT9g/640?wx_fmt=jpeg&wxfrom=13&tp=wxpic)  

本文为看雪论坛精华文章  
看雪论坛作者ID：随风而行aa

  

  

1

  

# **前言**

  

今天总结Android APP四大组件中Content Provider挖掘的知识，主要分为两个部分，一部分是对Android Content Provider内容提供器的原理总结，另一部分便是对Android provider机制常见的一些漏洞总结，包括一些已知的漏洞方法，和一部分案例实践。

  

  

2

  

# **Content Provider初步介绍**

  

### **1、Content Provider的基本原理**

  

### （1）Content Provider简介

  

![图片](https://mmbiz.qpic.cn/mmbiz_png/kR3MmOkZ9r42PxsNG4V0KFbfYINsXAAGEcFEzF7NPH3nueSjCqfXl16np7TIHnbG6MBLEG8j3BvN9lIgoDIvIA/640?wx_fmt=png&wxfrom=13&tp=wxpic)

  

Android中的数据存储方式：Shared Preferences、网络存储、文件存储、外部存储、SQLite,这些存储方式一般在单独的应用程序中实现数据共享，对于不同应用之间共享数据，就要借助Content Provider。

  

ContentProvider为存储和读取数据提供了统一的接口，使用表的形式来对数据进行封装，使用ContentProvider可以在不同的应用程序之间共享数据，统一数据的访问方式，保证数据的安全性。

  

### （2）Content Provider作用

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

Content Provider可以使得不同APP进程之间进行数据交互和共享，即跨进程通信。

  

### （3）URI详解

  

我们创建一个Content Provider，其他的应用可以通过使用ContentResolver来访问ContentProvider提供的数据，而ContentResolver通过uri来定位自己要访问的数据，所以我们要先了解URI

#####   

##### URI：

  

URI的介绍：

（1）定义：Uniform Resource Identifier，即统一资源标识符。

（2）作用：唯一标识ContentProvider &其中的数据。

（3）外界进程通过URL找到对应的ContentProvider &其中数据，再进行数据操作。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

（1）标准前缀：content:// ,用来说明一个Content Provider控制这些数据。

  

（2）URL的标识：com.carson.provider, 用于唯一标识这个ContentProvider，外部调用者可以根据这个标识来找到它。对于第三方程序，为了保证URL标识的一致性，必须是一个完整的、小写的类名，这个标识在元素的authorities属性中说明，一般是定义该ContentProvider的包.类的名称。

  

（3）路径：User,要操作的数据库中表的名字，或者可以自己定义，记得在使用的时候保持一致。

  

（4）记录ID：id, 如果URL中包含表示需要获取的记录ID,则返回该id对应的数据，如果没有ID,就表示返回全部。

  

构建URI的路径：

  

（1）操作User表中id为11的记录，构建数据：/User/11

  

（2）操作User表中id为11的记录的name字段：User/11/name

  

（3）操作User表中的所有记录：/User

  

（4）操作来自文件、xml或网络其他存储方式的数据，如要操作xml文件中User节点下的name字段：/User/name

  

（5）若要将一个字符串转换成URI,可以使用Uri类中的parse()方法：

    Uri uri = Uri.parse("content://com.carson.provider/User")

  

URI各部分的获取：

我们给出一个URI的样例：

_http://www.baidu.com:8080/wenku/jiatiao.html?id=123456&name=jack_

  

我们介意使用一些方法来获取URI的各个部分：

```
getScheme()：获取 Uri 中的 scheme 字符串部分，在这里是 http
```

#####   

##### MIME：

MIME是指定某个扩展名的文件用一种应用程序打开，就像用浏览器查看PDF格式的文件，浏览器会选择合适的应用打开。ContentProvider 会根据 URI 来返回 MIME 类型，ContentProvider 会返回一个包含两部分的字符串。

  

MIME 类型一般包含两部分，如：

```
text/html
```

  

分为类型和子类型，Android 遵循类似的约定来定义MIME类型，每个内容类型的 Android MIME 类型有两种形式：多条记录（集合）和单条记录。

  

- 集合记录（dir）：
    

```
vnd.android.cursor.dir/自定义
```

  

- 单条记录（item）：
    

```
vnd.android.cursor.item/自定义
```

  

vnd 表示这些类型和子类型具有非标准的、供应商特定的形式。Android中类型已经固定好了，不能更改，只能区别是集合还是单条具体记录，子类型可以按照格式自己填写，在使用 Intent 时，会用到 MIME，根据 Mimetype 打开符合条件的活动。

  

##### URI解析:

  

这里URI代表要操作的数据，我们在对数据进行获取时需要解析URI，Android提供了两个操作URI的工具类：UriMatcher 和 ContentUris。

UriMatcher：

UriMatcher类用于匹配Uri，使用步骤如下：

  

- 将需要匹配的Uri路径进行注册：
    

```
//常量UriMatcher.NO_MATCH表示不匹配任何路径的返回码
```

  

此处采用 addURI 注册了两个需要用到的 URI；注意，添加第二个 URI 时，路径后面的 id 采用了通配符形式 “#”，表示只要前面三个部分都匹配上了就 OK。

补充：

```
*:表示匹配任意长度的任意字符
```

  

- 注册完需要匹配的 Uri 后，可以使用 sMatcher.match(Uri) 方法对输入的 Uri 进行匹配，如果匹配就返回对应的匹配码，匹配码为调用 addURI() 方法时传入的第三个参数。  
    

```
switch (sMatcher.match(Uri.parse("content://com.zhang.provider.yourprovider/tablename/100"))) {
```

  

ContentUris：

ContentUris类用于操作Uri路径后面的ID部分，有两个比较实用的方法：withAppendedId(Uri uri, long id)和parseId(Uri uri)。

  

- withAppendedId(Uri uri, long id)用于为路径加上ID部分：
    

  

```
Uri uri = Uri.parse("content://com.wang.provider.myprovider/tablename");
```

  

- parseId(Uri uri)则从路径中获取ID部分：
    

```
Uri uri = Uri.parse("content://com.zhang.provider.myprovider/tablename/10")
```

  

### （4）Content Provider数据共享

  

ContentProvider是一个抽象类，我们需要开发自己的内容提供者就需要继承这个类并复写其方法：

```
ContentProvider 类主要方法的介绍：
```

  

如果操作的数据属于集合类型，那么 MIME 类型字符串应该以 vnd.android.cursor.dir/ 开头：

```
要得到所有 tablename 记录：Uri 为 content://com.wang.provider.myprovider/tablename，那么返回的MIME类型字符串应该为vnd.android.cursor.dir/table
```

  

如果要操作的数据属于非集合类型数据，那么 MIME 类型字符串应该以 vnd.android.cursor.item/ 开头：

```
要得到 id 为 10 的 tablename 记录，Uri 为 content://com.wang.provider.myprovider/tablename/10，那么返回的 MIME 类型字符串为：vnd.android.cursor.item/tablename
```

###   

### （5）Content Resolver操作数据

  

当外部应用需要对ContentProvider中的数据进行添加、删除、修改及查询操作时，可以使用ContentResolver类来完成，要获取ContentResolver对象，可以使用Activity提供getContentResolver()。

ContentResolver类提供了与ContentProvider类相同签名的四个方法：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

```
public Uri insert(Uri uri, ContentValues values)，往ContentProvider添加数据；
```

  

这些方法的第一个参数为Uri，代表要操作的ContentProvider和对其中的什么数据进行操作，其实和ContentProvider里面的方法是一样的，最终会被传到我们之前程序里面定义的ContentProvider方法。

```
假定给定的是：Uri.parse("content://com.wang.provider.myprovider/tablename/10")，
```

  

使用ContentResolver对ContentProvider中的数据进行操作：

```
ContentResolver resolver = getContentResolver();
```

  

监听数据变化：

如果ContentProvider的访问者需要知道数据发生的变化，可以在ContentProvider发生数据变化时调用getContentResolver().notifyChange(uri, null)来通知注册在此URI上的访问者。只给出类中监听部分的代码：

```
public class MyProvider extends ContentProvider {
```

  

而访问者必须使用ContentObserver对数据（数据采用uri描述）进行监听，当监听到数据变化通知时，系统就会调用ContentObserver的onChange()方法：

```
getContentResolver().registerContentObserver(Uri.parse("content://com.ljq.providers.personprovider/person"),
```

  

### （6）Content Provider使用

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

创建内容提供者的基本流程：

  

（1）创建一个扩展ContentProviderbaseclass的 Content Provider 类。

  

（2）定义将用于访问内容的内容提供者 URI 地址。

  

（3）创建自己的数据库来保存内容。通常，Android 使用 SQLite 数据库，框架需要覆盖onCreate()方法，该方法将使用 SQLite Open Helper 方法创建或打开提供者的数据库。当您的应用程序启动时，其每个内容提供程序的onCreate()处理程序在主应用程序线程上被调用。

  

（4）实现内容提供者查询以执行不同的数据库特定操作。

  

（5）最后使用 <provider> 标签在您的活动文件中注册您的内容提供者。

  

### **2、Content Provider漏洞的种类和危害**

  

Content Provoder漏洞大致可以分为：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

Content Provider漏洞的危害：

  

Android中Content Provider起到在不同的进程APP之间实现共享数据的作用，通过Binder进程间通信机制以及匿名共享内存机制来实现，但是考虑到数据的安全性，我们需要设置一定的保护权限。

  

Binder进程间通信机制突破了以应用程序为边界的权限控制，是安全可控的，数据的访问接口由数据的所有者来提供，数据提供方实现安全控制，决定数据的读写操作。

  

而content Provider组件本身提供了读取权限控制，这导致在使用过程中就会存在一些漏洞。

  

  

3

  

# **Content Provider漏洞原理分析和复现**

  

### **1、漏洞挖掘方法**

  

先检测组件的exported属性，再检测组件permission、readPermission、writePermissio对应的protectionlevel，最后再检测sdk版本。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

#### （1）查找导出Provider

  

①反编译 apk 文件，在AndroidManifest.xml中查找显示设置了android:exported="true"Content Provider。

  

②使用drozer工具，执行命令：run app.provider.info -a ddns.android.vuls。

  

#### （2）查找URI

- 反编译apk文件，在代码中查找UriMatcher.addURI，并手动拼接uri。
    

  

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

如上，可以拼接出：

```
content://ddns.vuls.AccountProvider/account
```

  

- 使用drozer工具
    

```
执行命令 run app.provider.finduri ddns.android.vuls
```

####   

#### （3）方法使用

```
1.使用adb shell查询
```

  

我们下面将结合这三种方法来对一些常见的案例进行漏洞挖掘介绍。

###   

### **2、信息泄露漏洞**

  

#### （1）原理介绍

#### content URI是一个标志provider中的数据的URI。Content URI中包含了整个provider的以符号表示的名字(它的authority)和指向一个表的名字(一个路径)。当你调用一个客户端的方法来操作一个，provider中的一个表，指向表的contentURI是参数之一，如果对ContentProvider的权限没有做好控制，就有可能导致恶意的程序通过这种方式读取APP的敏感数据。

  

#### （2）漏洞复现

案例1：盛大有你Android存在信息泄露漏洞

目标代码：

```
<provider android:name=".providers.YouNiProvider" android:process="com.snda.youni.mms" android:authorities="com.snda.youni.providers.DataStructs"/>
```

  

攻击代码：

```
private void getyouni(){
```

  

代码分析：

我们可以分析目标程序的provider的进程名和授权的的URI，我们可以根据授权的URI来构建一个URI，然后通过contentresolver去读取里面的的列表名信息，这样我们就可以获取APP中的隐私数据信息。

案例2：样例sieve.apk

我们先向apk中添加一条数据，然后保存：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

我们先使用drozer对内容提供器的路径进行扫描：

```
run scanner.provider.finduris -a <包名>
```

  

报错：drozer could not find or compile a required extension library

这是由于我们drozer2.7中代码导致的，我们需要修改相应的代码，参考网址_(https://github.com/FSecureLABS/drozer/issues/361 )_

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

我们可以对敏感数据读取：

```
run app.provider.query uri
```

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

我们就成功的将我们刚才保存的账号密码信息给获取了。  

案例3：CVE-2018-9546: Download Provider文件头信息泄露

漏洞描述：

Download Provider运行app获取下载的http请求头，但理论上APP只能访问自己下载的文件的http请求头，但Download Provider没有做好权限配置，导致heads可以被任意读取。header中会保存一些敏感数据，例如cookie等。

  

目标代码：

```
读取header的URI为：content://download/mydownloads/download_id/headers
```

  

攻击代码：

```
Uri uri = Uri.parse("content://download/mydownloads/1493/headers");
```

  

由于header的URI并未做一些防护措施，我们可以将download_id取具体的值，然后来获取里面的具体信息。

  

#### （3）安全防护

  

① minSdkVersion不低于9。

  

②不向外部app提供数据的私有content provider显示设置exported=”false”，避免组件暴露(编译api小于17时更应注意此点)。

  

③内部app通过content provid交换数据时，设置protectionLevel=”signature”验证签名。

  

④公开的content provider确保不存储敏感数据。

针对权限保护绕过防御措施：

  

①使用Context.checkCallingPermission()和Context.enforceCallingPermission()来确保调用者拥有相应的权限，防止串谋攻击(confused deputy)。

  

②可以使用如下函数，获取应用的permission保护级别是否与系统中已定义的permission保护级别一致。如果不一致，则抛出异常。

  

### **3、SQL注入漏洞**

  

#### （1）原理介绍

  

对Content Provider进行增删改查操作时，程序没有对用户的输入进行过滤，未采用参数化查询的方式，可能会导致sql注入攻击。

  

所谓的SQL注入攻击指的是攻击者可以精心构造selection参数、projection参数以及其他有效的SQL语句组成部分，实现在未授权的情况下从Content Provider获取更多信息。应该避免使用SQLiteDatabase.rawQuery()进行查询，而应该使用编译好的参数化语句。

  

使用预编译好的语句比如SQLiteStatement，不仅可以避免SQL注入，而且操作性能也大幅提高，因为其不用每次执行都进行解析。

  

另外一种方式是使用query(),insert(),update(),和delete()方法，因为这些函数也提供了参数化的语句。预编译的参数化语句，问号处可以插入或者使bindString()绑定值。从而避免SQL注入攻击。

  

#### （2）漏洞复现

案例1：安全管家客户端存在SQL注入攻击

漏洞说明：

Android版安全管家客户端contentprovider uri配置不当，导致sql注入，使得任何应用可不需要root权限下，获得和修改数据库中数据。

  

Androidmanifest文件中定义的provider：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

使用drozer扫描客户端程序存在的contentProvider uri：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

搜索到对外暴露可访问的uri：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

newapp.db结构：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

查看新安装应用的包名：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

查看白名单：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

案例2：样本sieve

我们使用drozer扫描注入的位置：

```
run scanner.provider.injection -a <包名>
```

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

然后我们执行以下命令，发现返回了报错信息，接着构造sql获取敏感数据。

```
run app.provider.query content://com.mwr.example.sieve.DBContentProvider/Passwords/ --projection "'"
```

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

列出所有表信息：

```
run app.provider.query content://com.mwr.example.sieve.DBContentProvider/Passwords/ --projection "* FROM SQLITE_MASTER WHERE type='table';--"
```

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

获取具体表信息：

```
run app.provider.query content://com.mwr.example.sieve.DBContentProvider/Passwords/ --projection "* FROM Key;--"
```

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

列出该app的表信息：

```
run scanner.provider.sqltables -a  com.mwr.example.sieve
```

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

案例3：CVE-2018-9493: Download Provider SQL注入

漏洞分析：

  

Download Provider中的以下columns是不允许被外部访问的，例如CookieData，但是利用SQL注入漏洞可以绕过这个限制。

  

projection参数存在注入漏洞，结合二分法可以爆出某些columns字段的内容。

  

目标代码：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

攻击代码：

详细可以参考该作者博客：_(https://mabin004.github.io/2019/04/15/Android-Download-Provider%E6%BC%8F%E6%B4%9E%E5%88%86%E6%9E%90/)_

  

#### （3）安全防护

①实现健壮的服务端校验。

②使用参数化查询语句，比如SQLiteStatement。

③避免使用rawQuery()。

④过滤用户的输入。

###   

### **4、目录遍历漏洞**

####   

#### （1）原理介绍

Android Content Provider存在文件目录遍历安全漏洞，该漏洞源于对外暴露Content Provider组件的应用，没有对Content Provider组件的访问进行权限控制和对访问的目标文件的Content Query Uri进行有效判断，攻击者利用该应用暴露的Content Provider的openFile()接口进行文件目录遍历以达到访问任意可读文件的目的。

  

漏洞触发的前提条件：

  

对外暴露的Content Provider组件实现了openFile()接口；

  

没有对所访问的目标文件Uri进行有效判断，如没有过滤限制如“../”可实现任意可读文件的访问的Content Query Uri。

####   

#### （2）漏洞复现

  

案例1：赶集网Android客户端Content Provider组件任意文件读取漏洞_（http://wy.zone.ci/bug_detail.php?wybug_id=wooyun-2013-044407）_

漏洞分析：

  

赶集网客户端APP的实现中定义了一个可以访问本地文件的Content Provider组件，默认的android:exported="true"，对应com.ganji.android.jobs.html5.LocalFileContentProvider，该Provider实现了openFile()接口，通过此接口可以访问内部存储app_webview目录下的数据，由于后台未能对目标文件地址进行有效判断，可以通过"../"实现目录跨越，实现对任意私有数据的访问（当然，也可以访问任意外部存储数据，只是我们更关心私有敏感数据）。

  

攻击代码：

```
public void GJContentProviderFileOperations(){
```

  

案例2：样本sieve

我们检测文件遍历漏洞：

```
run scanner.provider.traversal -a <包名>
```

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

我们读取系统文件：

```
run app.provider.read content://com.mwr.example.sieve.FileBackupProvider/etc/hosts
```

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

我们下载系统文件：

```
run app.provider.download content://com.mwr.example.sieve.FileBackupProvider/data/data/com.mwr.example.sieve/databases/database.db f:/home/database.db
```

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

案例3：

目标代码：

```
private static String IMAGE_DIRECTORY=localFile.getAbsolutePath();
```

  

我们可以从目标代码中分析，这段代码使用android.net.Uri.getLastPathSegment()从paramUri中获取文件名，然后将其放置在预定义好的目录IMAGE_DIRECTORY中，如果该URL是encoded编码后的，那么将可能导致目录遍历漏洞。

Android4.3开始，Uri.getLastPathSegment()内部实现调用Uri.getPathSegments()。

```
Uri.getPathSegments()部分代码片段： 
```

  

Uri.getPathSegments首先会通过getEncoded()获取一个路径，然后以”/“为分隔符将path分成片段，最后调用decode()方法解码。

假如我们传递encoded编码后的url给getLastPathSegment()，编码后的分隔符就变成了%2F，绕过了内部的分割规则，那么返回的就可能不是真正想要的文件了。这是API设计方面的问题，直接导致了目录遍历漏洞。

```
public String getLastPathSegment(){
```

  

为了避免这种情况导致的目录遍历漏洞，开发者应该在传递给getLastPathSegment()之前解码，采用调用两次getLastPathSegment()方法的方式，第一次调用是为了解码，第二次调用期望得到正确的值这一部分大家可以详细参考博客：_(https://tea9.xyz/post/758430476.html)_

```
private static String IMAGE_DIRECTORY=localFile.getAbsolutePath();
```

  

#### （3）安全防护

① 将不必要导出的Content Provider设置为不导出。

② 去除没有必要的openFile()接口。

③ 过滤限制跨域访问，对访问的目标文件的路径进行有效判断。

④ 设置权限来进行内部应用通过Content Provider的数据共享。

##   
  

4

  

# **实验总结**

  

本文对Content Provider内容提供器的基本原理做了一个详细讲解，然后对Provider常见的一些漏洞情况作了分析，这里面一部分漏洞来自于漏洞平台，一部分来自于网上的博客收集总结，还提供了一个样例sieve.apk，初步的实现信息泄露、SQL注入、目录遍历漏洞的基本操作方式，也介绍了一般挖掘provider漏洞的基本方法。

  

其中关于drozer的具体操作使用，大家可以参考之前的博客：Android漏洞挖掘三板斧——drozer+Inspeckage(Xposed)+MobSF_（https://bbs.pediy.com/thread-269196.htm）_，当然可能对于Provider中的漏洞介绍还不是很全面，其他的就请各位大佬指正了。

##   

  

5

  

# **参考文献**

  

**Content Provider原理介绍**

_https://www.cnblogs.com/tgyf/p/4696288.html_

_https://www.jianshu.com/p/5e13d1fec9c9_

_https://www.cnblogs.com/huansky/p/13785634.html_

_http://www.tutorialspoint.com/android/android_content_providers.htm_

  

**Content Provider漏洞挖掘**

_https://tea9.xyz/post/758430476.html_

_https://ayesawyer.github.io/2019/08/21/Android-App%E5%B8%B8%E8%A7%81%E5%AE%89%E5%85%A8%E6%BC%8F%E6%B4%9E/_

_https://wy.zone.ci/bug_detail.php?wybug_id=wooyun-2015-0156386_

_http://www.feidao.site/wordpress/?p=3295_

_http://www.hackdig.com/03/hack-19497.htm_

_https://mabin004.github.io/2019/04/15/Android-Download-Provider%E6%BC%8F%E6%B4%9E%E5%88%86%E6%9E%90/_

  

  

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E) 

  

**看雪ID：随风而行aa**

https://bbs.pediy.com/user-home-905443.htm

*本文由看雪论坛 随风而行aa 原创，转载请注明来自看雪社区

  

  

[![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458395632&idx=1&sn=34f2b52324f54626d84b46a430f35ed2&chksm=b18f137a86f89a6cb696f51c149732d6d1b428cde53e1fe8bb209ddc228c3774094515af9c79&scene=21#wechat_redirect)

  

[![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)](https://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458387399&idx=2&sn=38495add2a3a3677b2c436581c07e432&scene=21#wechat_redirect)

  

**#** **往期推荐**

1.[钉钉邀请上台功能分析](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458398080&idx=1&sn=1438f3db7d878da9a3c72cadb9e566d6&chksm=b18f1d0a86f8941c6d850df67ee2f05a19de4c324d393f88efb890826d188ce2c79c2a6b51c8&scene=21#wechat_redirect)  

2.[Android APP漏洞之战——Activity漏洞挖掘详解](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458398050&idx=2&sn=be762c14ec92d032f642c5270e47350e&chksm=b18f1de886f894fe777768f608e368a474c5e59af641ee3d2127b104f6ee6f21439423dde225&scene=21#wechat_redirect)

3.[少量虚假控制流混淆后的算法还原案例](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458397992&idx=1&sn=f2947a899d2e97478db9a0fad48151c2&chksm=b18f1da286f894b4d6609f30ec8e32a9fb250c9f2f4b406509065e33aab0e80bd894d6f3bbbf&scene=21#wechat_redirect)

4.[PHP反序列化漏洞基础](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458397245&idx=2&sn=d0b40b359f56c55e4db3be6d02545de2&chksm=b18f1ab786f893a1020366fd743fbedbafe1017412bff2a927eea022da170425492025797db9&scene=21#wechat_redirect)

5.[栈溢出漏洞利用（绕过ASLR）](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458396195&idx=2&sn=3517949e465cddf16e347f801bfebe1f&chksm=b18f16a986f89fbf85b00510bee4577728c95cc58d9be79a2b5f09896c5c44f691c637791853&scene=21#wechat_redirect)

6.[大杀器Unidbg真正的威力](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458395762&idx=2&sn=f9662663e130096ffd01c86658813742&chksm=b18f14f886f89dee2fbc6e37a4aae1252e578f2a4abeae40b238a3f850069bd25c10a226000e&scene=21#wechat_redirect)

  

  

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

阅读 2373

​