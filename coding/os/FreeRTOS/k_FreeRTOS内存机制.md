原创 Lupei OPPO内核工匠

_2022年01月14日 17:00_

## 

## **一、FreeRTOS内存分配方式**

FreeRTOS的内存分配一般为静态分配方式以及动态分配方式。在FreeRTOS 的V9.0.0 版本之前，任务、软件定时器，信号量、互斥锁等系统资源所需的内存，只支持在系统配置的heap申请内存。在FreeRTOS 的V9.0.0 版本后，这些系统资源即允许在系统的heap中申请内存，也支持为任务、软件定时器，信号量、互斥锁等系统资源指定自定义的内存空间。

静态分配内存：以静态分配方式给任务、软件定时器，信号量、互斥锁等系统资源分配资源，不会调用freeRTOS的pvPortMalloc内存分配接口，在RAM在自定义内存空间（全局数组、全局变量等），创建任务、定时器。信号量、互斥锁等资源，将自定义的内存空间与创建的系统资源绑定。内存分布如下图所示：

![图片](https://mmbiz.qpic.cn/mmbiz_png/d4hoYJlxOjO1uCXrdFvkDLpny8JaxVQyzPQtiaoalHn3iaf9ZdL4PwaoYWTXibhpc6OMxuBCicqQrzybduPrepLUnA/640?wx_fmt=png&wxfrom=13&tp=wxpic)

动态分配内存：以动态分配方式给任务、软件定时器，信号量、互斥锁等系统资源分配资源，调用系统提供的pvPortMalloc内存分配接口，在系统的ucHeap中（使用heap_3.c的内存管理方式除外，使用heap_3.c内存方式，会直接调用c库的malloc申请空间）申请任务、信号量、队列、互斥锁等所需的内存空间。内存分布如下图所示（heap_3.c的内存管理方式除外）：

!\[\[Pasted image 20240926181152.png\]\]
!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

## **二、FreeRTOS 内存操作相关接口**

FreeRTOS的任务、任务队列、软件定时器、事件标志组、信号量、任务通知以及互斥锁等系统资源创建的时候，需要申请对应的内存，提供给这些资源对应的内存空间。

除了xTaskCreate、xTaskCreateStatic、xQueueCreate、xEventGroupCreateStatic等系统资源创建接口申请对应的资源内存外，FreeRTOS提供了以下几个接口对系统的ucHeap内存进行管理：

pvPortMalloc ：申请内存函数，申请对应的内存，系统使用不同的heap                                管理方式，申请的内存方式有所不同。

vPortFree ：内存释放函数，释放申请到的内存。

vPortInitialiseBlocks：初始化内存函数

xPortGetFreeHeapSize：获取当前未分配使用的内存大小

xPortGetMinimumEverFreeHeapSize：获取历史未分配使用的内存大小最小值

## **三、FreeRTOS 内存管理方式**

FreeRTOS提供5种内存管理实现方式，这五种方式对应在官方源码的Source\\portable\\MemMang目录下的五个.c文件，它们分别如下：

heap_1.c ：只允许分配内存，不允许释放内存，通常用于内存分配规划固定，不会删除创建好的任务、队列、定时器等系统资源，任务运行过程中不会动态申请内存的应用程序设计。

heap_2.c ：分配内存时运行最佳匹配算法，允许释放内存，但是不会把释放的内存相邻的零碎的内存合并。分配是只会在剩余的内存里面划分出所需的内存块，造成内存碎片。通常运用于动态创建任务的所需堆栈大小一样的应用中。

heap_3.c ：对c库的malloc以及free函数进行简单封装，加入线程保护，申请内存具有不确定性。

heap_4.c ：分配内存时运行最佳匹配算法，允许释放内存,并且提供了内存合并算法，会把内存碎片合并成一块大的可用内存。heap_4因存在合并算法，不会产生严重的内存碎片，适合运用于任务运行过程中需要使用pvPortMalloc以及vPortFree动态申请以及释放内存的应用中。它和heap_3一样具有不确定性，但是申请以及释放内存效率会比heap_3高。

heap_5.c ：heap_5的内存管理实现方式与heap_4基本相同，但是heap_5比heap_4增加了一个特性，heap_5允许申请释放内存是跨域，跨越不连续的内存段。比如，允许在内部的RAM以及外部的SRAM中跨域对内存进行操作。

### **1. heap_1**

heap_1的内存管理策略是5个内存管理策略中只允许申请不允许释放的内存管理策略。一旦申请到了内存，再也不会将其释放。这种管理策略可应用于对安全有较高要求的嵌入式系统中，并且此嵌入式系统具有一定确定性，创建的任务、队列、软件定时器等系统资源不会将其删除释放，一直存在系统的生命周期中。

由下面官方提供的pvPortMalloc代码可知，申请的内存数量调整到对齐字节数的整数倍前提下，在系统的ucHeap的大数组分配对应的内存。

!\[\[Pasted image 20240926181206.png\]\]
!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

从官方的提供的vPortFree代码可知，heap_1不支持释放内存。

!\[\[Pasted image 20240926181213.png\]\]
!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

### **2. heap_2**

heap_2使用链表对系统ucHeap中剩余的空闲内存块进行管理，申请内存时，会采用最佳匹配算法给调用pvPortMalloc的申请内存者分配对应的内存，如下官方提供的pvPortMalloc函数实现所示：

!\[\[Pasted image 20240926181218.png\]\]
!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

heap_2的vPortFree函数将释放的内存块直接回收到剩余内存块的链表中，不会将内存碎片进行合并。

### **3. heap_3**

heap_3申请内存时，挂起所有的FreeRTOS系统任务，调用c库的malloc函数进行动态申请内存，申请完成，继续调用xTaskResumeAll运行系统任务，如下图官方代码所示：

!\[\[Pasted image 20240926181226.png\]\]
!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

同样的，释放内存时，调用系统的vTaskSuspendAll函数挂起所有的系统任务，使用c库的free函数释放内存，释放完成Resume所有的系统任务。

### **4. heap_4**

Heap_4和heap_2一样采用链表的方式对剩余的内存进行管理，分配内存的时候同样采用最佳匹配算法搜索内存给到用户。Heap_4与heap_2不同之处在于，调用prvInsertBlockIntoFreeList函数将剩余空闲内存块压入剩余内存链表的时候，会将链表相邻的内存碎片进行合并。如官方给出的prvInsertBlockIntoFreeList函数源码所示：

!\[\[Pasted image 20240926181231.png\]\]
!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

### **5. heap_5**

heap_5内存管理方式与heap_4基本类似，在heap_4基础上加入了允许跨越不同的不连续的内存域进行内存的分配以及释放，跨越的不同的与在HeapRegion_t结构体进行定义。参考下面官方注释提供的范例所示进行定义即可：

!\[\[Pasted image 20240926181238.png\]\]
!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

## **四、小结**

FreeRTOS提供两种不同的创建系统资源（任务、队列、定时器、信号量等）的方式进行内存的分配，提供5种内存的管理方式进行系统的内存管理。基于不同的嵌入式应用场景，用户可以使用不同内存分配方式以及内存管理策略。灵活、简捷、因地制宜、按需分配管理内存。因此，FreeRTOS的内存机制可以满足大部分的嵌入式系统设计，被广泛应用于各个嵌入式的行业当中。

参考文献；

FreeRTOS官方文档:https://www.freertos.org/features.html

FreeRTOS官方代码：https://www.freertos.org/a00104.html

### 

### 

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**长按关注**

**内核工匠微信**

Linux 内核黑科技 | 技术文章 | 精选教程

阅读 1526

​
