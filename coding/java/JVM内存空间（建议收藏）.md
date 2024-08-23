# 

冰河 冰河技术

 _2022年01月20日 10:31_

**大家好，我是冰河~~**

今天，跟大家一起聊聊关于JVM内存空间的话题，这也是一线互联网大厂面试中经常被问及的问题，冰河建议小伙伴们收藏后经常拿出来翻阅，重在理解。好了，不多说了，开始今天的正题。  

![](http://mmbiz.qpic.cn/mmbiz_png/2hHcUic5FEwEsQmOPMjJS0EZKCJ66T4VwM2ia7E7Dxj8pWyco8fSSt6EeUOTeRHM64TE2MGkxHibrXYibUALgGLncA/300?wx_fmt=png&wxfrom=19)

**冰河技术**

分享各种编程语言、开发技术、分布式与微服务架构、分布式数据库、分布式事务、云原生、大数据与云计算技术和渗透技术。另外，还会分享各种面试题和面试技巧。

673篇原创内容

公众号

点击上方卡片关注我

JVM会把内存划分成不同的数据区域，那加载的类是分配到哪里呢？下图是内存的各个区域，包括：方法区、堆、虚拟机栈、本地方法栈、程序计数器。

![Image](https://mmbiz.qpic.cn/mmbiz_png/2hHcUic5FEwFOdDxgmD49E6wUENWn9HpoI6IKH4HT192R7ecicV5M3DEEpJJggpBFsuTBrXfBYSaUSPI2U9KozJg/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

# 方法区

方法区用于存储已被虚拟机加载的类信息、常量、静态变量、即时编译器编译后的代码等数据。类的加载中提到了类加载的五个阶段。在加载阶段，会将字节流所代表的静态存储结构转化为方法区的运行时数据结构，在准备阶段，会将变量所使用的内存都将在方法区中进行分配。

# 程序计数器

来一个简单的代码，计算(1+2)*3并返回

`public int cal() {       int a = 1;       int b = 2;       int c = 3;       return (a + b) * c;   }   `

这段代码在加载到虚拟机的时候，就变成了以下的字节码，虚拟机执行的时候，就会一行行执行。

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

java是多线程的，在线程切换回来后，它需要知道原先的执行位置在哪里。用来记录这个执行位置的，就是程序计数器，为了保证线程间的计数器相互不影响，这个内存区域是线程私有的。

# 虚拟机栈

虚拟机栈也是线程私有的，生命周期与线程相同。每个线程都有自己的虚拟机栈，如果这个线程执行了一个方法，就会创建一个栈帧，方法从调用直至执行完成的过程，就对应着一个栈帧在虚拟机栈中入栈到出栈的过程。比如下面的例子，fun1调用fun2，fun2调用fun3，fun3创建Hello对象。

`public void fun1() {       fun2();   }      public void fun2() {       fun3();   }      public void fun3() {       Hello hello = new Hello();   }   `

调用的时候，流程图如下：

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

执行完成的时候，流程图如下：

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

每一个栈帧都包括了局部变量表、操作数栈、动态连接、方法返回地址和一些额外的附加信息。局部变量主要是存放方法参数以及方法内部定义的局部变量，操作数栈是一个后入先出栈，当方法刚刚开始执行的时候，这个方法的操作数栈是空的，在方法的执行过程中，会有各种字节码指令往操作数栈中写入和提取内容，也就是出栈/入栈操作。

我们通过上面(1+2)*3的例子，把方法区、程序计数器、虚拟机栈的协同工作理一下。首先通过javap查看它的字节码，经过类加载器加载后，此时这个字节码存在方法区中。stack表示栈深度是2，locals是本地变量的slot个数，args_size是入参的个数，默认是this。栈的深度、本地变量个数，入参个数，都是在编译器决定的。

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

如下图，指令的位置是方法区，局部变量和操作数栈的位置是虚拟机栈，程序计数器就在程序计数器（这个下面的图就不再重复）。当执行偏地址为0的指令的时候，程序计数器为0，局部变量第一个值是this，当前的指令就是方法区`0:iconst_1`,指令iconst_1就是把int常量值1进栈，这个1就到了虚拟机栈的操作数栈中。

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

当执行偏地址为1的指令的时候，程序计数器为1，把操作数栈的值赋值到局部变量，此时操作数栈清空了，局部变量多了一个1，这条指令执行完，就是对应上面int a=1的语句。

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

另外b，c两个语句的赋值，对应着2，3，4，5指令，这边不再重复。执行完5后，如下图所示：

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

执行6的时候，是执行iload_1，就是把第二个int型局部变量压入栈顶，这里的变量是1。

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

执行7的时候，是执行iload_2，就是把第三个int型局部变量压入栈顶，这里的变量是2。

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

执行8的时候，是iadd语句，指的是栈顶的两个int型元素出栈，得到结果后再压入栈顶。

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

执行9的时候，把栈顶的元素3，赋值到第五个局部变量。

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

执行到11的时候，把第五个局部变量值压入栈顶，执行到13的时候，把第四个局部变量值压入栈顶，执行14的时候，栈顶的两个int型元素出栈，相乘后的结果入栈，执行15的时候，从当前方法返回当前栈顶int型元素。这些与上面的相加差不多，就不再赘述了。

# 堆

堆内存区域的唯一目的就是存放对象实例，几乎所有的对象实例都在这里分配内存。比如上面的fun1调用fun2，fun2调用fun3，fun3创建Hello对象。fun3方法中创建对象时，就是在堆中创建的，并且把地址赋值给fun3的局部变量。Java堆中还可以细分为：新生代和老年代；新生代还细分为Eden空间、From Survivor空间、To Survivor空间。

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

# 总结

整体流程如下，先把java文件编译成class文件，通过类加载器加载到方法区。线程调用方法的时候，会创建一个栈帧，读取方法区的字节码执行指令，执行指令的时候，会把执行的位置记录在程序计数器中，如果创建对象，会在堆内存中创建，方法执行完，这个栈帧就会出栈。

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

# 内存参数

- -XX:PermSize：永久代内存容量。
    
- -XX:MaxPermSize：永久代最大内存容量。
    
- -Xss：栈内存容量。
    
- -Xms：堆内存容量。
    
- -Xmx：堆最大内存容量，通常和-Xms设置一样，防止运行时扩容产生的影响。
    
- -Xmn：新生代内存容量，老年代就是堆内存容量-新生代内存容量
    
- -XX:SurvivorRatio=8：新生代还细分为Eden空间、From Survivor空间、To Survivor空间，设置为8代表Eden空间：From Survivor空间：To Survivor空间=8：1：1，比如新生代有10M，那Eden空间占8M，From Survivor空间、To Survivor空间各占1M。
    

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**好了，今天就到这儿吧，我是冰河，我们下期见~~**

_转自：segmentfault.com/a/1190000038322634_

![](http://mmbiz.qpic.cn/mmbiz_png/2hHcUic5FEwEsQmOPMjJS0EZKCJ66T4VwM2ia7E7Dxj8pWyco8fSSt6EeUOTeRHM64TE2MGkxHibrXYibUALgGLncA/300?wx_fmt=png&wxfrom=19)

**冰河技术**

分享各种编程语言、开发技术、分布式与微服务架构、分布式数据库、分布式事务、云原生、大数据与云计算技术和渗透技术。另外，还会分享各种面试题和面试技巧。

673篇原创内容

公众号

点击上方卡片关注我

精通JVM系列18

精通JVM系列 · 目录

上一篇类的加载过程（建议收藏）下一篇JVM堆内存分配机制（建议收藏）

Read more

Reads 1752

​

Comment

**留言 12**

- 冰河
    
    2022年1月20日
    
    Like4
    
    一起玩转JVM，冲鸭～～
    
    Pinned
    
- 清水
    
    2022年1月27日
    
    Like
    
    方法区名字由来是啥，它又没有存类的方法
    
- 飞的更高
    
    2022年1月25日
    
    Like
    
    这的确是冰河时代的JVM空间了
    
- HappyIsVeryGood
    
    2022年1月21日
    
    Like
    
    冰河大哥，有必要去看汇编语言吗，熟悉了汇编语言是不是会了解java更透彻?mysql,nginx都是用c语言写的，那是不是学习了c语言才能对mysql，nginx性能调优更好的理解![[破涕为笑]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)
    
    冰河技术
    
    Author2022年1月21日
    
    Like
    
    不知道有没有必要，反正我是学过汇编和c，学完这些确实能够从底层去思考问题
    
    HappyIsVeryGood
    
    2022年1月24日
    
    Like
    
    感觉要是往高级工程师发展的话，是要学![[流泪]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)
    
- forever
    
    2022年1月20日
    
    Like
    
    虽然不得不说，冰河大佬越来越刁了
    
- Gibson·朋飞🔥魔术丝合伙人
    
    2022年1月20日
    
    Like
    
    大佬 最近高产呀
    
    冰河技术
    
    Author2022年1月20日
    
    Like
    
    更新频率低了
    
- 小沈同学呀
    
    2022年1月20日
    
    Like
    
    冲鸭
    
- 简
    
    2022年1月20日
    
    Like
    
    ![[加油]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)
    
- BigBigWolf
    
    2022年1月20日
    
    Like
    
    大佬可不可以讲下jni线程中断相关的内容呢![[可怜]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)
    

已无更多数据

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/2hHcUic5FEwEsQmOPMjJS0EZKCJ66T4VwM2ia7E7Dxj8pWyco8fSSt6EeUOTeRHM64TE2MGkxHibrXYibUALgGLncA/300?wx_fmt=png&wxfrom=18)

冰河技术

1517

12

Comment

**留言 12**

- 冰河
    
    2022年1月20日
    
    Like4
    
    一起玩转JVM，冲鸭～～
    
    Pinned
    
- 清水
    
    2022年1月27日
    
    Like
    
    方法区名字由来是啥，它又没有存类的方法
    
- 飞的更高
    
    2022年1月25日
    
    Like
    
    这的确是冰河时代的JVM空间了
    
- HappyIsVeryGood
    
    2022年1月21日
    
    Like
    
    冰河大哥，有必要去看汇编语言吗，熟悉了汇编语言是不是会了解java更透彻?mysql,nginx都是用c语言写的，那是不是学习了c语言才能对mysql，nginx性能调优更好的理解![[破涕为笑]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)
    
    冰河技术
    
    Author2022年1月21日
    
    Like
    
    不知道有没有必要，反正我是学过汇编和c，学完这些确实能够从底层去思考问题
    
    HappyIsVeryGood
    
    2022年1月24日
    
    Like
    
    感觉要是往高级工程师发展的话，是要学![[流泪]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)
    
- forever
    
    2022年1月20日
    
    Like
    
    虽然不得不说，冰河大佬越来越刁了
    
- Gibson·朋飞🔥魔术丝合伙人
    
    2022年1月20日
    
    Like
    
    大佬 最近高产呀
    
    冰河技术
    
    Author2022年1月20日
    
    Like
    
    更新频率低了
    
- 小沈同学呀
    
    2022年1月20日
    
    Like
    
    冲鸭
    
- 简
    
    2022年1月20日
    
    Like
    
    ![[加油]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)
    
- BigBigWolf
    
    2022年1月20日
    
    Like
    
    大佬可不可以讲下jni线程中断相关的内容呢![[可怜]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)
    

已无更多数据