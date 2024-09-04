

原创 捡田螺的小男孩 捡田螺的小男孩

 _2024年03月18日 09:14_ _广东_

## 前言

synchronized，是解决并发情况下数据同步访问问题的一把利刃。那么synchronized的底层原理是什么呢？下面我们来一层一层剥开它的心，就像剥洋葱一样，看个究竟。

![](http://mmbiz.qpic.cn/mmbiz_png/m2jCBpUlqWSUp1N5WBmiaHA6yGicBmUTfv255ZW1ZnJTIRTcuPlbXhqS5MkrlgGicESS3VfZicCBRibxXIrbVAAk52g/300?wx_fmt=png&wxfrom=19)

**田螺程序人生**

主要聊程序员职业生涯~~

4篇原创内容

公众号

## Synchronized的使用场景

synchronized关键字可以作用于方法或者代码块，最主要有以下几种使用方式，如图：![图片](https://mmbiz.qpic.cn/mmbiz_png/sMmr4XOCBzHicicQVicafhZWI9mb87VYncO0luAl3MeBHxY5bRiaCkCxDGuicpRWHeQhibOmEIC8ruS970X4pUGv68iag/640?wx_fmt=png&wxfrom=13&tp=wxpic)

**接下来，我们先剥开synchronized的第一层，反编译其作用的代码块以及方法**。

### synchronized作用于代码块

1. `public class SynchronizedTest {`
    
2.   
    
3.     `public void doSth(){`
    
4.         `synchronized (SynchronizedTest.class){`
    
5.             `System.out.println("test Synchronized" );`
    
6.         `}`
    
7.     `}`
    
8. `}`
    

反编译，可得：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

由图可得，添加了synchronized关键字的代码块，多了两个指令**monitorenter、monitorexit**。即JVM使用monitorenter和monitorexit两个指令实现同步，monitorenter、monitorexit又是怎样保证同步的呢？我们等下剥第二层继续探索。

### synchronized作用于方法

1.  `public synchronized void doSth(){`
    
2.             `System.out.println("test Synchronized method" );`
    
3.     `}`
    

反编译，可得：![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

由图可得，添加了synchronized关键字的方法，多了**ACCSYNCHRONIZED**标记。即JVM通过在方法访问标识符(flags)中加入ACCSYNCHRONIZED来实现同步功能。

## monitorenter、monitorexit、ACC_SYNCHRONIZED

剥完第一层，反编译synchronized的方法以及代码块，我们已经知道synchronized是通过monitorenter、monitorexit、ACC_SYNCHRONIZED实现同步的，它们三作用都是啥呢？我们接着剥第二层：

### monitorenter

monitorenter指令介绍

> Each object is associated with a monitor. A monitor is locked if and only if it has an owner. The thread that executes monitorenter attempts to gain ownership of the monitor associated with objectref, as follows:
> 
> > If the entry count of the monitor associated with objectref is zero, the thread enters the monitor and sets its entry count to one. The thread is then the owner of the monitor.
> > 
> > If the thread already owns the monitor associated with objectref, it reenters the monitor, incrementing its entry count.
> > 
> > If another thread already owns the monitor associated with objectref, the thread blocks until the monitor's entry count is zero, then tries again to gain ownership.

谷歌翻译一下，如下：

> 每个对象都与一个**monitor** 相关联。当且仅当拥有所有者时（被拥有），monitor才会被锁定。执行到monitorenter指令的线程，会尝试去获得对应的monitor，如下：
> 
> > 每个对象维护着一个记录着被锁次数的计数器, 对象未被锁定时，该计数器为0。线程进入monitor（执行monitorenter指令）时，会把计数器设置为1.
> > 
> > 当同一个线程再次获得该对象的锁的时候，计数器再次自增.
> > 
> > 当其他线程想获得该monitor的时候，就会阻塞，直到计数器为0才能成功。

可以看一下以下的图，便于理解用：![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

### monitorexit

monitorexit指令介绍

> The thread that executes monitorexit must be the owner of the monitor associated with the instance referenced by objectref.
> 
> The thread decrements the entry count of the monitor associated with objectref. If as a result the value of the entry count is zero, the thread exits the monitor and is no longer its owner. Other threads that are blocking to enter the monitor are allowed to attempt to do so.

谷歌翻译一下，如下：

> monitor的拥有者线程才能执行 monitorexit指令。
> 
> 线程执行monitorexit指令，就会让monitor的计数器减一。如果计数器为0，表明该线程不再拥有monitor。其他线程就允许尝试去获得该monitor了。

可以看一下以下的图，便于理解用：![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

### ACC_SYNCHRONIZED

ACC_SYNCHRONIZED介绍

> Method-level synchronization is performed implicitly, as part of method invocation and return. A synchronized method is distinguished in the run-time constant pool’s methodinfo structure by the ACCSYNCHRONIZED flag, which is checked by the method invocation instructions. When invoking a method for which ACC_SYNCHRONIZED is set, the executing thread enters a monitor, invokes the method itself, and exits the monitor whether the method invocation completes normally or abruptly. During the time the executing thread owns the monitor, no other thread may enter it. If an exception is thrown during invocation of the synchronized method and the synchronized method does not handle the exception, the monitor for the method is automatically exited before the exception is rethrown out of the synchronized method.

谷歌翻译一下，如下：

> 方法级别的同步是隐式的，作为方法调用的一部分。同步方法的常量池中会有一个ACC_SYNCHRONIZED标志。
> 
> 当调用一个设置了ACC_SYNCHRONIZED标志的方法，执行线程需要先获得monitor锁，然后开始执行方法，方法执行之后再释放monitor锁，当方法不管是正常return还是抛出异常都会释放对应的monitor锁。
> 
> 在这期间，如果其他线程来请求执行方法，会因为无法获得监视器锁而被阻断住。
> 
> 如果在方法执行过程中，发生了异常，并且方法内部并没有处理该异常，那么在异常被抛到方法外面之前监视器锁会被自动释放。

可以看一下这个流程图：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

### Synchronized第二层的总结

- 同步代码块是通过monitorenter和monitorexit来实现，当线程执行到monitorenter的时候要先获得monitor锁，才能执行后面的方法。当线程执行到monitorexit的时候则要释放锁。
    
- 同步方法是通过中设置ACCSYNCHRONIZED标志来实现，当线程执行有ACCSYNCHRONI标志的方法，需要获得monitor锁。
    
- 每个对象维护一个加锁计数器，为0表示可以被其他线程获得锁，不为0时，只有当前锁的线程才能再次获得锁。
    
- 同步方法和同步代码块底层都是通过monitor来实现同步的。
    
- 每个对象都与一个monitor相关联，线程可以占有或者释放monitor。
    

好的，剥到这里，我们还有一些不清楚的地方，**monitor是什么呢，为什么它可以实现同步呢？对象又是怎样跟monitor关联**的呢？客观别急，我们继续剥下一层，请往下看。

## monitor监视器

montor到底是什么呢？**我们接下来剥开Synchronized的第三层，monitor是什么？** 它可以理解为一种**同步工具**，或者说是**同步机制**，它通常被描述成一个对象。操作系统的**管程**是概念原理，**ObjectMonitor**是它的原理实现。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

### 操作系统的管程

- 管程 (英语：Monitors，也称为监视器) 是一种程序结构，结构内的多个子程序（对象或模块）形成的多个工作线程互斥访问共享资源。
    
- 这些共享资源一般是硬件设备或一群变量。管程实现了在一个时间点，最多只有一个线程在执行管程的某个子程序。
    
- 与那些通过修改数据结构实现互斥访问的并发程序设计相比，管程实现很大程度上简化了程序设计。
    
- 管程提供了一种机制，线程可以临时放弃互斥访问，等待某些条件得到满足后，重新获得执行权恢复它的互斥访问。
    

### ObjectMonitor

#### ObjectMonitor数据结构

在Java虚拟机（HotSpot）中，Monitor（管程）是由ObjectMonitor实现的，其主要数据结构如下：

1.  `ObjectMonitor() {`
    
2.     `_header       = NULL;`
    
3.     `_count        = 0; // 记录个数`
    
4.     `_waiters      = 0,`
    
5.     `_recursions   = 0;`
    
6.     `_object       = NULL;`
    
7.     `_owner        = NULL;`
    
8.     `_WaitSet      = NULL;  // 处于wait状态的线程，会被加入到_WaitSet`
    
9.     `_WaitSetLock  = 0 ;`
    
10.     `_Responsible  = NULL ;`
    
11.     `_succ         = NULL ;`
    
12.     `_cxq          = NULL ;`
    
13.     `FreeNext      = NULL ;`
    
14.     `_EntryList    = NULL ;  // 处于等待锁block状态的线程，会被加入到该列表`
    
15.     `_SpinFreq     = 0 ;`
    
16.     `_SpinClock    = 0 ;`
    
17.     `OwnerIsThread = 0 ;`
    
18.   `}`
    

#### ObjectMonitor关键字

ObjectMonitor中几个关键字段的含义如图所示：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

#### 工作机理

Java Monitor 的工作机理如图所示：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

- 想要获取monitor的线程,首先会进入_EntryList队列。
    
- 当某个线程获取到对象的monitor后,进入Owner区域，设置为当前线程,同时计数器count加1。
    
- 如果线程调用了wait()方法，则会进入WaitSet队列。它会释放monitor锁，即将owner赋值为null,count自减1,进入WaitSet队列阻塞等待。
    
- 如果其他线程调用 notify() / notifyAll() ，会唤醒WaitSet中的某个线程，该线程再次尝试获取monitor锁，成功即进入Owner区域。
    
- 同步方法执行完毕了，线程退出临界区，会将monitor的owner设为null，并释放监视锁。
    

为了形象生动一点，举个例子：

1.   `synchronized(this){  //进入_EntryList队列`
    
2.             `doSth();`
    
3.             `this.wait();  //进入_WaitSet队列`
    
4.         `}`
    

OK，我们又剥开一层，知道了monitor是什么了，那**么对象又是怎样跟monitor关联**呢？各位帅哥美女们，我们接着往下看，去剥下一层。

## 对象与monitor关联

对象是如何跟monitor关联的呢？直接先看图：![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

看完上图，其实对象跟monitor怎样关联，我们已经有个大概认识了，接下来我们分**对象内存布局，对象头，MarkWord**一层层继续往下探讨。

### 对象的内存布局

在HotSpot虚拟机中,对象在内存中存储的布局可以分为3块区域：对象头（Header），实例数据（Instance Data）和对象填充（Padding）。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

- **实例数据**：对象真正存储的有效信息，存放类的属性数据信息，包括父类的属性信息；
    
- **对齐填充**：由于虚拟机要求 对象起始地址必须是8字节的整数倍。填充数据不是必须存在的，仅仅是为了字节对齐。
    
- **对象头**：Hotspot虚拟机的对象头主要包括两部分数据：Mark Word（标记字段）、Class Pointer（类型指针）。
    

### 对象头

对象头主要包括两部分数据：Mark Word（标记字段）、Class Pointer（类型指针）。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

- **Class Pointer**:是对象指向它的类元数据的指针，虚拟机通过这个指针来确定这个对象是哪个类的实例
    
- **Mark Word** : 用于存储对象自身的运行时数据，它是实现轻量级锁和偏向锁的关键。
    

### Mark word

Mark Word 用于存储对象自身的运行时数据，如哈希码（HashCode）、GC分代年龄、锁状态标志、线程持有的锁、偏向线程 ID、偏向时间戳等。

在32位的HotSpot虚拟机中，如果对象处于未被锁定的状态下，那么Mark Word的32bit空间里的25位用于存储对象哈希码，4bit用于存储对象分代年龄，2bit用于存储锁标志位，1bit固定为0，表示非偏向锁。其他状态如下图所示：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

- 前面分析可知，monitor特点是互斥进行，你再喵一下上图，**重量级锁，指向互斥量的指针**。
    
- 其实synchronized是**重量级锁**，也就是说Synchronized的对象锁，Mark Word锁标识位为10，其中指针指向的是Monitor对象的起始地址。
    
- 顿时，是不是感觉柳暗花明又一村啦！对象与monitor怎么关联的？答案：**Mark Word重量级锁，指针指向monitor地址**。
    

### Synchronized剥开第四层小总结

对象与monitor怎么关联？

- 对象里有对象头
    
- 对象头里面有Mark Word
    
- Mark Word指针指向了monitor
    

## 锁优化

事实上，只有在JDK1.6之前，synchronized的实现才会直接调用ObjectMonitor的enter和exit，这种锁被称之为重量级锁。**一个重量级锁，为啥还要经常使用它呢？** 从JDK6开始，HotSpot虚拟机开发团队对Java中的锁进行优化，如增加了适应性自旋、锁消除、锁粗化、轻量级锁和偏向锁等优化策略。

### 自旋锁

**何为自旋锁？**

自旋锁是指当一个线程尝试获取某个锁时，如果该锁已被其他线程占用，就一直循环检测锁是否被释放，而不是进入线程挂起或睡眠状态。

**为何需要自旋锁？**

线程的阻塞和唤醒需要CPU从用户态转为核心态，频繁的阻塞和唤醒显然对CPU来说苦不吭言。其实很多时候，锁状态只持续很短一段时间，为了这段短暂的光阴，频繁去阻塞和唤醒线程肯定不值得。因此自旋锁应运而生。

**自旋锁应用场景**

自旋锁适用于锁保护的临界区很小的情况，临界区很小的话，锁占用的时间就很短。

**自旋锁一些思考**

在这里，我想谈谈，**为什么ConcurrentHashMap放弃分段锁，而使用CAS自旋方式**，其实也是这个道理。

### 锁消除

**何为锁消除？**

锁削除是指虚拟机即时编译器在运行时，对一些代码上要求同步，但是被检测到不可能存在共享数据竞争的锁进行削除。

**锁消除一些思考**

在这里，我想引申到日常代码开发中，有一些开发者，在没并发情况下，也使用加锁。如没并发可能，直接上来就ConcurrentHashMap。

### 锁粗化

**何为锁租化？**

锁粗话概念比较好理解，就是将多个连续的加锁、解锁操作连接在一起，扩展成一个范围更大的锁。

**为何需要锁租化？**

在使用同步锁的时候，需要让同步块的作用范围尽可能小—仅在共享数据的实际作用域中才进行同步，这样做的目的是 为了使需要同步的操作数量尽可能缩小，如果存在锁竞争，那么等待锁的线程也能尽快拿到锁。**但是如果一系列的连续加锁解锁操作，可能会导致不必要的性能损耗，所以引入锁粗话的概念。**

**锁租化比喻思考**

举个例子，买门票进动物园。老师带一群小朋友去参观，验票员如果知道他们是个集体，就可以把他们看成一个整体（锁租化），一次性验票过，而不需要一个个找他们验票。

## 总结

我们直接以一张Synchronized洋葱图作为总结吧，如果你愿意一层一层剥开我的心。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

## 参考与感谢

- Synchronized之管程 
    
- 深入理解多线程（一）——Synchronized的实现原理 
    
- 《深入理解Java虚拟机》
    
- 深入理解多线程（五）—— Java虚拟机的锁优化技术 
    

![](http://mmbiz.qpic.cn/mmbiz_png/m2jCBpUlqWSUp1N5WBmiaHA6yGicBmUTfv255ZW1ZnJTIRTcuPlbXhqS5MkrlgGicESS3VfZicCBRibxXIrbVAAk52g/300?wx_fmt=png&wxfrom=19)

**田螺程序人生**

主要聊程序员职业生涯~~

4篇原创内容

公众号

这个是我的新号，后面够500人关注，开始写程序员人生相关的，大家可以点个关注~~  

![](https://mmbiz.qlogo.cn/mmbiz_jpg/Laz3IPXGxKFDbjVBdTj4fllfbKQCoibokML0BNoFdchgZDmu34t414yz9OYgaBEQTurQW6icHNuxNuG7n8pvQAOg/0?wx_fmt=jpeg)

捡田螺的小男孩

![赞赏二维码](https://mp.weixin.qq.com/s?__biz=MzkyMzU5Mzk1NQ==&mid=2247508925&idx=1&sn=965159a3c0ca1767543c1cb4a13e7775&chksm=c1e05e31f697d727d083686231797133a8619f94ba2f1b3faa309f3e63ab926c20849771e813&mpshare=1&scene=24&srcid=0318WbUsyRzha4kD2TgfzhTP&sharer_shareinfo=f0539d342eb91c7da8eef3d70156309b&sharer_shareinfo_first=f0539d342eb91c7da8eef3d70156309b&key=daf9bdc5abc4e8d0bbaa17d248f0be8159618a796b11e2e03c8137f5438c06f133f37811b51ab8e77ae79a0e233676cd70b9c8e667c72bfcbd41fbcf759251f0082a250d8aad797f953ef3b1f37ae2dc618b87ee826c08f29273440f9eb5cb9b6049fac48c4870789681f51df93a9bf222089325c68eb73f0ef7099537517b81&ascene=0&uin=MTEwNTU1MjgwMw%3D%3D&devicetype=Windows+11+x64&version=63090b19&lang=zh_CN&countrycode=CN&exportkey=n_ChQIAhIQjtmZBz1tgk96LsxqtgxgohLmAQIE97dBBAEAAAAAALe3K76%2BqUUAAAAOpnltbLcz9gKNyK89dVj0Ap2%2FWLOkVfwLBahLYk7QHNdixnyLghoSgL8pVmNDGQxtR8GbEjLJCgkoh5Gkg4MhFsI2cS%2FkcTN%2FovQ4uBfWTUO9YCCQGCX2RDCrK9bjw0rdthAcj8jKAixTwHnlK%2F8NrY%2BsdJ%2BVAu4xLMexjMltyCR1UlgO%2Bx%2FOlVPObURtY6rz1bZo%2FIEvtAMnEyackZ1Bo4kKgcSkhHQaet%2F%2Fwo%2B6piuSmFwockXplmlO26l%2FrFnZ27slo%2FfZ7rW6kJuDMM%2BV&acctmode=0&pass_ticket=t0gTipSWBjQPf4E%2Fon9f6sNiXVy1%2B57cKWSBA%2FBD%2BOoVPwQwqWtc61D02mCl4gKS&wx_header=1&fasttmpl_type=0&fasttmpl_fullversion=7350504-zh_CN-zip&fasttmpl_flag=1)喜欢作者

阅读 2676

​

写留言

[](javacript:;)

![](http://mmbiz.qpic.cn/sz_mmbiz_png/cEDP0gXG22n22jFlEqLy1BBlJ0yJXicTGDibTmjqib1j2yZKibo0BicvWYKToCChBqIEJFSdyF2DR5Wya71XCrHTPnQ/300?wx_fmt=png&wxfrom=18)

捡田螺的小男孩

132363

写留言

写留言

**留言**

暂无留言