# [蜗窝科技](http://www.wowotech.net/)

### 慢下来，享受技术。

[![](http://www.wowotech.net/content/uploadfile/201401/top-1389777175.jpg)](http://www.wowotech.net/)

- [博客](http://www.wowotech.net/)
- [项目](http://www.wowotech.net/sort/project)
- [关于蜗窝](http://www.wowotech.net/about.html)
- [联系我们](http://www.wowotech.net/contact_us.html)
- [支持与合作](http://www.wowotech.net/support_us.html)
- [登录](http://www.wowotech.net/admin)

﻿

## 

作者：[wowo](http://www.wowotech.net/author/2 "runangaozhong@163.com") 发布于：2014-3-14 18:31 分类：[统一设备模型](http://www.wowotech.net/sort/device_model)

#### 1. 前言

sysfs是一个基于RAM的文件系统，它和Kobject一起，可以将Kernel的数据结构导出到用户空间，以文件目录结构的形式，提供对这些数据结构（以及数据结构的属性）的访问支持。

sysfs具备文件系统的所有属性，而本文主要侧重其设备模型的特性,因此不会涉及过多的文件系统实现细节，而只介绍sysfs在Linux设备模型中的作用和使用方法。具体包括：

- sysfs和Kobject的关系
- attribute的概念
- sysfs的文件系统操作接口

#### 2. sysfs和Kobject的关系

在"[Linux设备模型_Kobject](http://www.wowotech.net/linux_kenrel/kobject.html)”文章中，有提到过，每一个Kobject，都会对应sysfs中的一个目录。因此在将Kobject添加到Kernel时，create_dir接口会调用sysfs文件系统的创建目录接口，创建和Kobject对应的目录，相关的代码如下：

```cpp
 1: /* lib/kobject.c, line 47 */
 2: static int create_dir(struct kobject *kobj)
 3: {
 4:     int error = 0;
 5:     error = sysfs_create_dir(kobj);
 6:     if (!error) {
 7:         error = populate_dir(kobj);
 8:     if (error)
 9:         sysfs_remove_dir(kobj);
 10:     }   
 11:     return error;
 12: }
 13:  
 14: /* fs/sysfs/dir.c, line 736 */
 15: **
 16: *  sysfs_create_dir - create a directory for an object.
 17: *  @kobj:      object we're creating directory for. 
 18: */
 19: int sysfs_create_dir(struct kobject * kobj)
 20: {
 21:     enum kobj_ns_type type;
 22:     struct sysfs_dirent *parent_sd, *sd;
 23:     const void *ns = NULL;
 24:     int error = 0;
 25:     ...
 26: }
```

#### 3. attribute

##### 3.1 attribute的功能概述

在sysfs中，为什么会有attribute的概念呢？其实它是对应kobject而言的，指的是kobject的“属性”。我们知道，

sysfs中的目录描述了kobject，而kobject是特定数据类型变量（如struct device）的体现。因此kobject的属性，就是这些变量的属性。它可以是任何东西，名称、一个内部变量、一个字符串等等。而attribute，在sysfs文件系统中是以文件的形式提供的，即：kobject的所有属性，都在它对应的sysfs目录下以文件的形式呈现。这些文件一般是可读、写的，而kernel中定义了这些属性的模块，会根据用户空间的读写操作，记录和返回这些attribute的值。

总结一下：所谓的attibute，就是内核空间和用户空间进行信息交互的一种方法。例如某个driver定义了一个变量，却希望用户空间程序可以修改该变量，以控制driver的运行行为，那么就可以将该变量以sysfs attribute的形式开放出来。

Linux内核中，attribute分为普通的attribute和二进制attribute，如下：

```cpp
 1: /* include/linux/sysfs.h, line 26 */
 2: struct attribute {
 3:     const char *name;
 4:     umode_t         mode;
 5: #ifdef CONFIG_DEBUG_LOCK_ALLOC
 6:     bool ignore_lockdep:1;
 7:     struct lock_class_key   *key;
 8:     struct lock_class_key   skey;
 9: #endif
 10: };
 11:  
 12: /* include/linux/sysfs.h, line 100 */
 13: struct bin_attribute {
 14:     struct attribute    attr;
 15:     size_t          size;
 16:     void *private;
 17:     ssize_t (*read)(struct file *, struct kobject *, struct bin_attribute *,
 18:                     char *, loff_t, size_t);
 19:     ssize_t (*write)(struct file *,struct kobject *, struct bin_attribute *,
 20:                     char *, loff_t, size_t);
 21:     int (*mmap)(struct file *, struct kobject *, struct bin_attribute *attr,
 22:                     struct vm_area_struct *vma);
 23: };
```

struct attribute为普通的attribute，使用该attribute生成的sysfs文件，只能用字符串的形式读写（后面会说为什么）。而struct bin_attribute在struct attribute的基础上，增加了read、write等函数，因此它所生成的sysfs文件可以用任何方式读写。

说完基本概念，我们要问两个问题：

Kernel怎么把attribute变成sysfs中的文件呢？
用户空间对sysfs的文件进行的读写操作，怎么传递给Kernel呢？

下面来看看这个过程。

##### 3.2 attibute文件的创建

在linux内核中，attibute文件的创建是由fs/sysfs/file.c中sysfs_create_file接口完成的，该接口的实现没有什么特殊之处，大多是文件系统相关的操作，和设备模型没有太多的关系，这里先略过不提。

##### 3.3 attibute文件的read和write

看到3.1章节struct attribute的原型时，也许我们会犯嘀咕，该结构很简单啊，name表示文件名称，mode表示文件模式，其它的字段都是内核用于debug Kernel Lock的，那文件操作的接口在哪里呢？

不着急，我们去fs/sysfs目录下看看sysfs相关的代码逻辑。

所有的文件系统，都会定义一个struct file_operations变量，用于描述本文件系统的操作接口，sysfs也不例外：

```cpp
 1: /* fs/sysfs/file.c, line 472 */
 2: const struct file_operations sysfs_file_operations = {
 3:     .read       = sysfs_read_file,
 4:     .write      = sysfs_write_file,
 5:     .llseek     = generic_file_llseek,
 6:     .open       = sysfs_open_file,
 7:     .release    = sysfs_release,
 8:     .poll       = sysfs_poll,
 9: };
```

attribute文件的read操作，会由VFS转到sysfs_file_operations的read（也就是sysfs_read_file）接口上，让我们大概看一下该接口的处理逻辑。

```cpp
 1: /* fs/sysfs/file.c, line 127 */
 2: static ssize_t
 3: sysfs_read_file(struct file *file, char __user *buf, size_t count, loff_t *ppos)
 4: {
 5:     struct sysfs_buffer * buffer = file->private_data;
 6:     ssize_t retval = 0;
 7:  
 8:     mutex_lock(&buffer->mutex);
 9:     if (buffer->needs_read_fill || *ppos == 0) {
 10:        retval = fill_read_buffer(file->f_path.dentry,buffer);
 11:        if (retval)
 12:            goto out;
 13:    }
 14: ...
 15: }
 16: /* fs/sysfs/file.c, line 67 */
 17: static int fill_read_buffer(struct dentry * dentry, struct sysfs_buffer * buffer)
 18: {           
 19:    struct sysfs_dirent *attr_sd = dentry->d_fsdata;
 20:    struct kobject *kobj = attr_sd->s_parent->s_dir.kobj;
 21:    const struct sysfs_ops * ops = buffer->ops;
 22:    ...        
 23:    count = ops->show(kobj, attr_sd->s_attr.attr, buffer->page);
 24:    ...
 25: }
```

> read处理看着很简单，sysfs_read_file从file指针中取一个私有指针（注：大家可以稍微留一下心，私有数据的概念，在VFS中使用是非常普遍的），转换为一个struct sysfs_buffer类型的指针，以此为参数（buffer），转身就调用fill_read_buffer接口。
>
> 而fill_read_buffer接口，直接从buffer指针中取出一个struct sysfs_ops指针，调用该指针的show函数，即完成了文件的read操作。
>
> 那么后续呢？当然是由ops->show接口接着处理咯。而具体怎么处理，就是其它模块（例如某个driver）的事了，sysfs不再关心（其实，Linux大多的核心代码，都是只提供架构和机制，具体的实现，也就是苦力，留给那些码农吧！这就是设计的魅力）。
>
> 不过还没完，这个struct sysfs_ops指针哪来的？好吧，我们再看看open(sysfs_open_file)接口吧。

```cpp
 1: /* fs/sysfs/file.c, line 326 */
 2: static int sysfs_open_file(struct inode *inode, struct file *file)
 3: {
 4:     struct sysfs_dirent *attr_sd = file->f_path.dentry->d_fsdata;
 5:     struct kobject *kobj = attr_sd->s_parent->s_dir.kobj;
 6:     struct sysfs_buffer *buffer;
 7:     const struct sysfs_ops *ops;
 8:     int error = -EACCES;
 9:  
 10:    /* need attr_sd for attr and ops, its parent for kobj */
 11:    if (!sysfs_get_active(attr_sd))
 12:    return -ENODEV;
 13:  
 14:    /* every kobject with an attribute needs a ktype assigned */
 15:    if (kobj->ktype && kobj->ktype->sysfs_ops)
 16:        ops = kobj->ktype->sysfs_ops;
 17:    else {
 18:        WARN(1, KERN_ERR "missing sysfs attribute operations for "
 19:            "kobject: %s\n", kobject_name(kobj));
 20:        goto err_out;
 21:    }
 22:  
 23:    ...
 24:  
 25:    buffer = kzalloc(sizeof(struct sysfs_buffer), GFP_KERNEL);
 26:    if (!buffer)
 27:        goto err_out;
 28:  
 29:    mutex_init(&buffer->mutex);
 30:    buffer->needs_read_fill = 1;
 31:    buffer->ops = ops;
 32:    file->private_data = buffer;
 33:    ...
 34: }
```

> 哦，原来和ktype有关系。这个指针是从该attribute所从属的kobject中拿的。再去看一下"[Linux设备模型_Kobject](http://www.wowotech.net/linux_kenrel/kobject.html)”中ktype的定义，还真有一个struct sysfs_ops的指针。
>
> 我们注意一下14行的注释以及其后代码逻辑，如果从属的kobject（就是attribute文件所在的目录）没有ktype，或者没有ktype->sysfs_ops指针，是不允许它注册任何attribute的！
>
> 经过确认后，sysfs_open_file从ktype中取出struct sysfs_ops指针，并在随后的代码逻辑中，分配一个struct sysfs_buffer类型的指针（buffer），并把struct sysfs_ops指针保存在其中，随后（注意哦），把buffer指针交给file的private_data，随后read/write等接口便可以取出使用。嗯！惯用伎俩！

顺便看一下struct sysfs_ops吧，我想你已经能够猜到了。

```cpp
 1: /* include/linux/sysfs.h, line 124 */
 2: struct sysfs_ops {
 3:     ssize_t (*show)(struct kobject *, struct attribute *,char *);
 4:     ssize_t (*store)(struct kobject *,struct attribute *,const char *, size_t);
 5:     const void *(*namespace)(struct kobject *, const struct attribute *);
 6: };
```

attribute文件的write过程和read类似，这里就不再多说。另外，上面只分析了普通attribute的逻辑，而二进制类型的呢？也类似，去看看fs/sysfs/bin.c吧，这里也不说了。

讲到这里，应该已经结束了，事实却不是如此。上面read/write的数据流，只到kobject（也就是目录）级别哦，而真正需要操作的是attribute（文件）啊！这中间一定还有一层转换！确实，不过又交给其它模块了。 下面我们通过一个例子，来说明如何转换的。

#### 4. sysfs在设备模型中的应用总结

让我们通过设备模型class.c中有关sysfs的实现，来总结一下sysfs的应用方式。

首先，在class.c中，定义了Class所需的ktype以及sysfs_ops类型的变量，如下：

```cpp
 1: /* drivers/base/class.c, line 86 */
 2: static const struct sysfs_ops class_sysfs_ops = {
 3:     .show      = class_attr_show,
 4:     .store     = class_attr_store,
 5:     .namespace = class_attr_namespace,
 6: };  
 7: 
 8: static struct kobj_type class_ktype = {
 9:     .sysfs_ops  = &class_sysfs_ops,
 10:    .release    = class_release,
 11:    .child_ns_type  = class_child_ns_type,
 12: };
```

由前面章节的描述可知，所有class_type的Kobject下面的attribute文件的读写操作，都会交给class_attr_show和class_attr_store两个接口处理。以class_attr_show为例：

```cpp
 1: /* drivers/base/class.c, line 24 */
 2: #define to_class_attr(_attr) container_of(_attr, struct class_attribute, attr)
 3:  
 4: static ssize_t class_attr_show(struct kobject *kobj, struct attribute *attr,
 5: char *buf)
 6: {   
 7:     struct class_attribute *class_attr = to_class_attr(attr);
 8:     struct subsys_private *cp = to_subsys_private(kobj);
 9:     ssize_t ret = -EIO;
 10:  
 11:    if (class_attr->show)
 12:    ret = class_attr->show(cp->class, class_attr, buf);
 13:    return ret;
 14: }
```

该接口使用container_of从struct attribute类型的指针中取得一个class模块的自定义指针：struct class_attribute，该指针中包含了class模块自身的show和store接口。下面是struct class_attribute的声明：

```cpp
 1: /* include/linux/device.h, line 399 */
 2: struct class_attribute {
 3:     struct attribute attr;
 4:     ssize_t (*show)(struct class *class, struct class_attribute *attr,
 5:                     char *buf);
 6:     ssize_t (*store)(struct class *class, struct class_attribute *attr,
 7:                     const char *buf, size_t count);
 8:     const void *(*namespace)(struct class *class,
 9:                                 const struct class_attribute *attr); 
 10: };
```

因此，所有需要使用attribute的模块，都不会直接定义struct attribute变量，而是通过一个自定义的数据结构，该数据结构的一个成员是struct attribute类型的变量，并提供show和store回调函数。然后在该模块ktype所对应的struct sysfs_ops变量中，实现该本模块整体的show和store函数，并在被调用时，转接到自定义数据结构（struct class_attribute）中的show和store函数中。这样，每个atrribute文件，实际上对应到一个自定义数据结构变量中了。

_原创文章，转发请注明出处。蜗窝科技，[www.wowotech.net](http://www.wowotech.net/linux_kenrel/dm_sysfs.html)。_

标签: [Linux](http://www.wowotech.net/tag/Linux) [Kernel](http://www.wowotech.net/tag/Kernel) [内核](http://www.wowotech.net/tag/%E5%86%85%E6%A0%B8) [设备模型](http://www.wowotech.net/tag/%E8%AE%BE%E5%A4%87%E6%A8%A1%E5%9E%8B) [sysfs](http://www.wowotech.net/tag/sysfs)

______________________________________________________________________

« [process credentials](http://www.wowotech.net/process_management/19.html) | [蓝牙协议中LQ和RSSI的原理及应用场景](http://www.wowotech.net/bluetooth/lqi_rssi.html)»

**评论：**

**xiaotonga**\
2023-09-19 10:46

文中介绍的比较清楚，用户对kernel空间特定数据属性访问是通过sysfs来实现，具体形式为读写设备文件属性。文中对calss.c进行分析说明但对device driver实现kobj->ktype->sysfs_ops->show调用逻辑理解不是特别清楚；\
结合kernel driver的实现补充，进一步理解其调用关系；\
调用栈：user（read(dev_file)）->syscall(read)->vfs(file_operations.read)->sysfs(sysfs_file_operation.read->fill_read_buffer->sysfs_ops.show)->driver(xxx_sysfs_ops.show)

1.vfs提供统一文件操作接口，sysfs需要实现vfs文件操作结构file_operations,即sysfs_file_operations;

2.sysfs_file_operations中sysfs file相关操作接口,负责sysfs文件及其属性的打开、读、写等

/\*

const struct file_operations sysfs_file_operations = {

.read = sysfs_read_file,

.write = sysfs_write_file,

.llseek = generic_file_llseek,

.open = sysfs_open_file,

.release = sysfs_release,

.poll = sysfs_poll,

};

\*/

4.对于“每一个Kobject，都会对应sysfs中的一个目录”，说明sysfs_ops通过kobject与device driver关联；

5.“该变量以sysfs attribute的形式开放出来”：通过sysfs接口开放attribute给用户，用户设置atrribute来控制driver函数调用或功能

6..sysfs_ops中相关函数怎么关联至driver中sysfs_ops实现呢？

通过kobj对象来关联具体device driver：

sysfs_read_file->fill_read_buffer.sysfs_ops.ops->show(kobj,attr_sd->s_attr.attr, buffer->page)->driver.show()

对dma中sysfs_ops回调函数的实现如下：

（linux/v2.6.39.4/source/fs/sysfs/inode.c#L269）\
sys_file_operations：主要用户操作sysfs_kobj_attr;

```C
  
///linux/v2.6.39.4/source/drivers/dma/ioat/dma.c#L1142  
const struct sysfs_ops ioat_sysfs_ops = {  
.show = ioat_attr_show,  
  
};  
  
//sysfs_ops回调dma的ioat_attr_show，具体实现功能由dma driver负责。  
static ssize_t  
ioat_attr_show(struct kobject *kobj, struct attribute *attr, char *page)  
  
{  
  
struct ioat_sysfs_entry *entry;  
struct ioat_chan_common *chan;  
  
entry = container_of(attr, struct ioat_sysfs_entry, attr);  
  
chan = container_of(kobj, struct ioat_chan_common, kobj);  
  
  
if (!entry->show)  
  
return -EIO;  
  
return entry->show(&chan->common, page);  
  
}  
  
```

[回复](http://www.wowotech.net/device_model/dm_sysfs.html#comment-8827)

**爱德华**\
2020-08-12 14:40

使用sysfs_create_file要注意一点：如果没有传入device的kobj，而是自己手动创建的kobj，就不要使用 DEVICE_ATTR 宏去定义attr，而应当使用struct kobj_attribute去定义attr，因为show和store传参形式是在ktype中定义的，如果使用DEVICE_ATTR并且使用自己创建的kobj，无法填入内核自带的device_ktype，因为drivers\\base\\core.c中，该ktype用static限制了作用域：\
static struct kobj_type device_ktype ，

除非自己再手动定义一个ktype，构造和device_ktype 相同的dev_attr_show和dev_attr_store，但是那样就显得多此一举了。

[回复](http://www.wowotech.net/device_model/dm_sysfs.html#comment-8090)

**落花非雪**\
2019-10-14 15:42

你好，不知道您怎么称呼。就叫你大神吧。我这边想请教下你一个问题：目前我在做一个mipi显示这块，有一个问题就是mipi显示这块的mipi-dsi显示控制器层驱动的某些方法只开放给内核使用了。我现在想江这些方法（读写mipi屏寄存器的方法）开放给用户空间使用，不知道有没有好的方法，目前驱动层是开发板官方提供的，点屏也是可以的只是通过内核设备树去配置点屏的初始化代码，这部分执行都是内核做的。现在就想用户空间就可以将点屏的初始化代码通过驱动层方法给执行下去。

[回复](http://www.wowotech.net/device_model/dm_sysfs.html#comment-7695)

**wmc**\
2019-07-08 16:56

蜗窝我有一个不明白的地方，sysfs_ops中的show和store是对kobject创建的目录读写吗？比如device_attribute这个结构体中的show和store是对属性文件读写，他们之间有什么关系吗？小白提的问题可能比较乱。

[回复](http://www.wowotech.net/device_model/dm_sysfs.html#comment-7515)

**木子**\
2019-03-06 15:08

linux 4.18 的内核中找不到sysfs_file_operations。是不是被sysfs_file_ops替换了呢？

[回复](http://www.wowotech.net/device_model/dm_sysfs.html#comment-7226)

**爱德华**\
2020-08-13 00:11

@木子：从linux3.14开始，sysfs已经采用新的kernfs框架实现。蜗窝这篇文章参考的是3.13之前的内核代码

[回复](http://www.wowotech.net/device_model/dm_sysfs.html#comment-8094)

**chunyan**\
2018-08-13 15:25

s/attibute/attribute

好像这里有typo？

[回复](http://www.wowotech.net/device_model/dm_sysfs.html#comment-6882)

**kobj**\
2017-06-10 09:16

为什么会出现如kobject_attribute、class_attribute、device_attribute之类的结构呢？不直接使用attribute吗？

[回复](http://www.wowotech.net/device_model/dm_sysfs.html#comment-5648)

**[wowo](http://www.wowotech.net/)**\
2017-06-10 14:28

@kobj：你可以自己推演一下：attribute就这么简单，连show、store都没有，怎么办呢？

[回复](http://www.wowotech.net/device_model/dm_sysfs.html#comment-5649)

**kobj**\
2017-06-10 15:39

@wowo：因为sysfs_ops中已经有了show和store接口了，按文中的意思是不是因为sysfs_ops中提供的接口只能处理所有属性的公共部分，所以提出了比如xxx_attribute这样的结构体来实现各自具体的show和store功能？如果是的话，那么sysfs_ops中的show和store是不是就没用了呢？

[回复](http://www.wowotech.net/device_model/dm_sysfs.html#comment-5651)

**[wowo](http://www.wowotech.net/)**\
2017-06-12 11:15

@kobj：你可以把sysfs_ops中的show和store看作一个桥梁，将文件的read/write，绕道具体attribute的show/store之上。

[回复](http://www.wowotech.net/device_model/dm_sysfs.html#comment-5655)

**miroor**\
2018-08-07 18:50

@wowo：是否可以理解为就像面向对象那样，attribute作为kobject_attribute、class_attribute、device_attribute……的基类？

[回复](http://www.wowotech.net/device_model/dm_sysfs.html#comment-6872)

**ask_dir**\
2019-03-13 16:58

@wowo：@wowo:是不是可以这样理解，定义device_attr之类结构体时先将其中的attr和sysfs_ops注册到kobject的ktype中去，然后读或者写时，vfs会调用ktype的sysfs_ops

[回复](http://www.wowotech.net/device_model/dm_sysfs.html#comment-7273)

**[wowo](http://www.wowotech.net/)**\
2019-03-13 19:23

@ask_dir：是的，vfs看到的都是统一的API，不用关心具体的实现。

[回复](http://www.wowotech.net/device_model/dm_sysfs.html#comment-7274)

**[callme_friend](http://www.wowotech.net/)**\
2016-04-18 22:05

着重看了下sysfs文件系统，有点疑问。\
sysfs在创建目录文件和普通文件时，有所区别：创建目录时，创建了sysfs_dirent、inode、dentry；而创建普通文件时，只创建了sysfs_dirent。总之，创建目录或文件时，已经创建了sysfs_dirent结构。  \
而在打开目录文件和普通文件时，应该是分别调用sysfs_dir_open()、sysfs_open_file()。\
在sysfs_dir_open()中，居然又新建了sysfs_dirent结构！！\
static int sysfs_dir_open(struct inode \*inode, struct file \*file)\
{\
file->private_data = sysfs_new_dirent(parent_sd, NULL);\
}

static int sysfs_open_file(struct inode \*inode, struct file \*file)\
{\
file->private_data = buffer;\
}\
不知为何，对于目录文件，为何要新建两次sysfs_dirent呢？

[回复](http://www.wowotech.net/device_model/dm_sysfs.html#comment-3833)

**[wowo](http://www.wowotech.net/)**\
2016-04-19 10:10

@callme_friend：这种做法应该是很旧的kernel版本吧？现在稍微新一点的版本都没有这样的代码了。\
如果要追究原因，建议以file->private_data为突破口，看看它是做什么的，为什么需要new dirent。\
我手头上没有旧的kernel版本，就先不帮你看了。

[回复](http://www.wowotech.net/device_model/dm_sysfs.html#comment-3837)

**passerby**\
2015-07-29 11:05

@wowo，sysfs_create_bin_file创建的节点是怎么用的？现在想用这个节点，但是不太清楚bin file这种节点和普通的file节点间的却别，和如何使用的

[回复](http://www.wowotech.net/device_model/dm_sysfs.html#comment-2349)

**passerby**\
2015-07-29 11:07

@passerby：现在有个驱动中创建了bin file的节点，我在shell中需要怎么操作这个节点？想普通的sysfs file一样echo或者read吗？还是其他的？

[回复](http://www.wowotech.net/device_model/dm_sysfs.html#comment-2350)

**[wowo](http://www.wowotech.net/)**\
2015-07-29 11:21

@passerby：普通的节点，是字符串形式的，例如读一个数据，它的值是1，你读出来的就是“1”这个字符串。\
但bin file的话，读出来的真的就是1。\
你要知道自己为什么需要bin格式的，例如为了dump出某一段内存，内存的数据是一幅图片。你应该不希望它变成一大堆无意义的字符串，所以使用bin file，直接拿出来，加一个header，就可以用图片查看工具看了。

[回复](http://www.wowotech.net/device_model/dm_sysfs.html#comment-2351)

**passerby**\
2015-07-29 11:23

@wowo：嗯，谢谢了。

[回复](http://www.wowotech.net/device_model/dm_sysfs.html#comment-2352)

**[wowo](http://www.wowotech.net/)**\
2014-12-04 22:47

puppypyb在《Linux设备模型(5)\_device和device driver》中评论非常对，这篇文章写的太过简单、模糊了。甚至一个很重要的概念（attribute group）都没有提。\
另外，“4. sysfs在设备模型中的应用总结”中的例子，如果换成device_attribute会更好，它在driver开发过程中会经常使用。\
很多人写driver，需要export出来一些信息的时候，总会胡来，直接自定义attribute（或group），然后调用sysfs_create_file（或者sysfs_create_group），导致/sys/devices\
/中的目录、文件乱飞（在wowo工作的team中，这种事很常见）。\
这种情况下，使用DEVICE_ATTR（struct device_attribute），然后调用 device_create_file创建文件，是比较规范的一种做法，这样创建出来的attribute文件，都会集中在"/sys/devices/xxx/driver/"目录下（其中xxx为driver名）。

总之，设备模型系列的文章，wowo只是把自己写明白了，远远没有达到让大家明白的程度，有时间要再重写几遍。

[回复](http://www.wowotech.net/device_model/dm_sysfs.html#comment-811)

**爱德华**\
2020-08-12 14:11

@wowo：之所以不用device_create_file，而直接用sysfs_create_file，原因有两种：一是血的人自己没用过device_create_file，看到别人用sysfs_create_file就照抄sysfs_create_file，二是虽然自己知道应该使用device_create_file，但是嫌device_create_file生成的路径太长了，想偷懒省事，用sysfs_create_file的话直接可以在/sys/下操作节点，不用cd到/sys/device/xxx/yyy/zzz去操作。

[回复](http://www.wowotech.net/device_model/dm_sysfs.html#comment-8089)

**爱德华**\
2020-08-12 14:50

@wowo：其实用sysfs_create_file也是可以的，只要传入&dev->kobj，并且使用struct device_attribute定义attr，效果和device_create_file时一样的

[回复](http://www.wowotech.net/device_model/dm_sysfs.html#comment-8091)

**调试者**\
2020-09-02 18:58

@爱德华：您好，关于struct device_attribute的使用有点疑问，不知能否帮忙解答一二。假设我在一个驱动里静态定义并初始化了一个struct device_attribute dev变量，然后调用kobject_create_and_add和sysfs_create_file（传入&dev->kobj），那么当我echo文件节点时，内核大致的路径应该是：dynamic_kobj_ktype -> kobj_sysfs_ops -> kobj_attr_store -> dev下具体的store回调。但是在kobj_attr_store中调用的是struct kobj_attribute里的store回调，其原型是ssize_t (\*store)(struct kobject \*kobj, struct kobj_attribute \*attr, const char \*buf, size_t count);，而struct device_attribute里的store回调原型是ssize_t (\*store)(struct device \*dev, struct device_attribute \*attr, const char \*buf, size_t count);两个store指针指向的函数类型明显不一样，调用流程却可以正常的走下去，这种情况想不到比较合理的解释。

[回复](http://www.wowotech.net/device_model/dm_sysfs.html#comment-8107)

1 [2](http://www.wowotech.net/device_model/dm_sysfs.html/comment-page-2#comments)

**发表评论：**

昵称

邮件地址 (选填)

个人主页 (选填)

![](http://www.wowotech.net/include/lib/checkcode.php)

- ### 站内搜索

  蜗窝站内  互联网

- ### 功能

  [留言板\
  ](http://www.wowotech.net/message_board.html)[评论列表\
  ](http://www.wowotech.net/?plugin=commentlist)[支持者列表\
  ](http://www.wowotech.net/support_list)

- ### 最新评论

  - Shiina\
    [一个电路（circuit）中，由于是回路，所以用电势差的概念...](http://www.wowotech.net/basic_subject/voltage.html#8926)
  - Shiina\
    [其中比较关键的点是相对位置概念和点电荷的静电势能计算。](http://www.wowotech.net/basic_subject/voltage.html#8925)
  - leelockhey\
    [你这是哪个内核版本](http://www.wowotech.net/pm_subsystem/generic_pm_architecture.html#8924)
  - ja\
    [@dream：我看完這段也有相同的想法，引用 @dream ...](http://www.wowotech.net/kernel_synchronization/spinlock.html#8922)
  - 元神高手\
    [围观首席power managerment专家](http://www.wowotech.net/pm_subsystem/device_driver_pm.html#8921)
  - 十七\
    [内核空间的映射在系统启动时就已经设定好，并且在所有进程的页表...](http://www.wowotech.net/process_management/context-switch-arch.html#8920)

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

  - [Linux vm运行参数之（二）：OOM相关的参数](http://www.wowotech.net/memory_management/oom.html)
  - [计算机科学基础知识（四）: 动态库和位置无关代码](http://www.wowotech.net/basic_subject/pic.html)
  - [Process Creation（一）](http://www.wowotech.net/process_management/Process-Creation-1.html)
  - [中断唤醒系统流程](http://www.wowotech.net/irq_subsystem/irq_handle_procedure.html)
  - [Linux电源管理(2)\_Generic PM之基本概念和软件架构](http://www.wowotech.net/pm_subsystem/generic_pm_architecture.html)

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
