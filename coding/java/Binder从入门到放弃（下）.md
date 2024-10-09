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

作者：[OPPO内核团队](http://www.wowotech.net/author/538) 发布于：2020-2-18 21:19

前言

Binder从入门到放弃包括了上下篇，上篇是框架部分，下篇通过几个典型的binder通信过程来呈现其实现细节，即本文。

一、启动service manager

1、流程

Service manager进程和binder驱动的交互如下：

[![wps9D0E.tmp](http://www.wowotech.net/content/uploadfile/202002/01c9f965ceaffc9db8cc54f1abfc072920200218133015.jpg "wps9D0E.tmp")](http://www.wowotech.net/content/uploadfile/202002/1fa4a2286cfa33980d50b2c07d0986af20200218133013.jpg)

在安卓系统启动过程中，init进程会启动service manager进程。service manager会打开/dev/binder设备，一个进程打开binder设备就意味着该进程会使用binder这种IPC机制，这时候，在内核态会相应的构建一个binder proc对象，来管理该进程相关的binder资源（binder ref、binder node、binder thread等）。为了方便binder内存管控，这时候还会映射一段128K的内存地址用于binder通信。之后，service manager会把自己设定为context manager。所谓context manager实际上就是一个“名字服务器”，可以完成service组件名字的解析。随后service manager会通过binder协议（BC_ENTER_LOOPER）告知驱动自己已经准备好接收请求了。最后，service manager会进入读阻塞状态，等待来自其他进程的服务请求。

完成上面的一系列操作之后，内核相关的数据结构如下所示：

[![wps9D0F.tmp](http://www.wowotech.net/content/uploadfile/202002/75605d8240c58061e43f311e7a77022020200218133027.jpg "wps9D0F.tmp")](http://www.wowotech.net/content/uploadfile/202002/9240540f6a93923a299294d734f4a2dd20200218133026.jpg)

由于Service manager也算是一个特殊的service组件，因此在内核态也有一个binder node对象与之对应。service manager和其他的service组件不同的是它没有使用线程池模型，而是一个单线程的进程，因此它在内核态只有一个binder proc和binder thread。整个系统系统只有一个binder context，系统中所有的binder proc都指向这个全局唯一的binder上下文对象。而找到了binder context也就找到了service manager对应的binder node。

binder proc使用了红黑树来管理其所属的binder thread和binder node，不过在Service manager这个场景中，binder proc只管理了一个binder thread和binder node，看起来似乎有些小题大做，不过在其他场景（例如system server）中，binder proc会创建线程池，也可能注册多个service组件。

2、相关数据结构

在内核态，每一个参与binder通信的进程都会用一个唯一的struct binder_proc对象来表示。struct binder_proc主要成员如下表所示：

|   |   |
|---|---|
|成员变量|描述|
|struct hlist_node<br><br>proc_node|系统中的所有binder proc挂入binder_procs的链表中，这个成员是挂入全局binder_procs的链表的节点|
|struct rb_root threads|binder进程对应的所有binder thread组成的红黑树，tid作为key|
|struct rb_root nodes|一个binder进程可以注册多个service组件，因此binder proc可以有很多的binder node。Binder proc对应的所有binder node组成一颗红黑树。当然对于service manager而言，它只有一个binder node。|
|struct list_head<br><br>waiting_threads|该binder进程的线程池中等待处理binder work的binder thread链表|
|int pid|进程ID|
|struct task_struct \*tsk|指向该binder进程对应的进程描述符（指向thread group leader对应的task struct）|
|struct list_head todo|需要该binder进程处理的binder work链表|
|int max_threads|线程池中运行的最大数目|
|struct binder_alloc alloc|管理binder 内存分配的数据结构|
|struct binder_context<br><br>\*context|保存binder上下文管理者的信息。通过binder context可以找到service manager对应的bind node。|

和进程抽象类似，binder proc也是管理binder资源的实体，但是真正执行binder通信的实体是binder thread。struct binder_thread主要成员如下表所示：

|   |   |
|---|---|
|成员变量|描述|
|struct binder_proc \*proc|该binder thread所属的binder proc|
|struct rb_node rb_node|挂入binder proc红黑树的节点|
|struct list_head<br><br>waiting_thread_node|无事可做的时候，binder thread会挂入binder proc的等待队列|
|int pid|Thread id|
|struct binder_transaction<br><br>\*transaction_stack|该binder thread正在处理的transaction|
|struct list_head todo|需要该binder线程处理的binder work链表|
|struct task_struct \*task|该binder thread对应的进程描述符|

Binder node是用户空间service组件对象的内核态实体对象，struct binder_node主要成员如下表所示：

|   |   |
|---|---|
|成员变量|描述|
|struct rb_node rb_node;|一个binder proc可能有多个service组件（提供多种服务），属于一个binder proc的binder node会挂入binder proc的红黑树，这个成员是嵌入红黑树的节点。|
|struct binder_proc \*proc|该binder node所属的binder proc|
|int debug_id|唯一标示该node的id，用于调试|
|struct hlist_head refs|一个service组件可能会有多个client发起服务请求，也就是说每一个client都是对binder node的一次引用，这个成员是就是保存binder ref的哈希表|
|binder_uintptr_t ptr<br><br>binder_uintptr_t cookie|指向用户空间service组件相关的信息|
|u8 sched_policy:2;<br><br>u8 inherit_rt:1;<br><br>u8 min_priority;|这些属性定义了该service组件在处理transaction的时候优先级的设定。|
|bool has_async_transaction|是否有异步通信需要处理|
|struct list_head async_todo|异步binder通信的队列|

二、client如何找到service manager？

1、流程

为了完成service组件注册，Client需要首先定位service manager组件。在client这个binder process中，我们使用handle作为地址来标记service组件。Service manager比较特殊，对任何一个binder process而言，handle等于0的那个句柄就是指向service manager组件。对内核态binder驱动而言，寻找service manager实际上就是寻找其对应的binder node。下面是一个binder client向service manager请求注册服务的过程示例，我们重点关注binder驱动如何定位service manager：

[![wps9D20.tmp](http://www.wowotech.net/content/uploadfile/202002/7b2c07d7437809f7934d57782ceb06e520200218133033.jpg "wps9D20.tmp")](http://www.wowotech.net/content/uploadfile/202002/057918451fc6860b8f96f5b9198884f120200218133028.jpg)

想要访问service manager的进程需要首先打开binder driver，这时候内核会创建该进程对应的binder proc对象，并建立binder proc和context manager的关系，这样进一步可以找到service manager对应的binder node。随后，client进程会调用mmap映射了（1M-8K）的binder内存空间。之所以映射这么怪异的内存size主要是为了有效的利用虚拟地址空间（VMA之间有4K的gap）。完成上面两步操作之后，client process就可以通过ioctl向service manager发起transaction请求了，同时告知目标对象handle等于0。

实际上这个阶段的主要工作在用户空间，主要是service manager组件代理BpServiceManager以及BpBinder的创建过程。一般的通信过程需要为组件代理对象分配一个句柄，但是service manager访问比较特殊，对于每一个进程，等于0的句柄都保留给了service manager，因此这里就不需要分配句柄这个过程了。

2、路由过程

在binder C/S通信结构中，binder client中的BpBinder找到binder server中的BBinder的过程需要如下过程：

（1）binder client用户空间中的service组件代理（BpBinder）用句柄表示要访问的server中的service组件（BBinder）

（2）对于每一个句柄，binder client内核空间使用binder ref对象与之对应

（3）binder ref对象会指向一个binder node对象

（4）binder node对象对应一个binder server进程的service组件

在我们这个场景中，binder ref是在client第一次通过ioctl和binder驱动交互时候完成的。这时候，binder驱动的binder_ioctl函数中会建立上面路由过程需要的完整的数据对象：

[![wps9D30.tmp](http://www.wowotech.net/content/uploadfile/202002/6e20d379508c56bd13c26641ec2871eb20200218133034.jpg "wps9D30.tmp")](http://www.wowotech.net/content/uploadfile/202002/cd074e8b834a166f5fe71505da77f9b120200218133034.jpg)

Service manager的路由比较特殊，没有采用binder ref--->binder node的过程。在binder驱动中，看到0号句柄自然就知道是去往service manager的请求。因此，通过binder proc--->binder context-----binder node这条路径就找到了service manager。

三、注册Service组件

1、流程

上一节描述了client如何找到service manager的过程，这是整个注册service组件的前半部分，这一节我们补全整个流程。由于client和service manager都完成了open和mmap的过程，双方都准备好，后续可以通过ioctl进行binder transaction的通信过程了，因此下面的流程图主要呈现binder transaction的流程（忽略client/server和binder驱动系统调用的细节）：

[![wps9D31.tmp](http://www.wowotech.net/content/uploadfile/202002/0e32769ff903f0c5c59481030390671820200218131938.jpg "wps9D31.tmp")](http://www.wowotech.net/content/uploadfile/202002/cf9087c1f100ce21838516ea715ab13a20200218131936.jpg)

Service manager是一个service组件管理中心，任何一个service组件都需要向service manager进行注册（add service），以便其他的APP可以通过service manager定位到该service组件（check service）。

2、数据对象综述

注册服务相关数据结构全图如下：

[![wps9D42.tmp](http://www.wowotech.net/content/uploadfile/202002/a1553166dab6e7fa66595c155d874a6e20200218133037.jpg "wps9D42.tmp")](http://www.wowotech.net/content/uploadfile/202002/580fef31db4dac1918513fb06d624cd320200218133036.jpg)

配合上面的流程，binder驱动会为client和server分别创建对应的各种数据结构对象，具体过程如下：

（1）假设我们现在准备注册A服务组件，绑定A服务组件的进程在add service这个场景下是client process，它在用户空间首先会创建了service组件对象，在递交BC_TRANSACTION的时候会携带service组件的信息（把service组件地址信息封装在flat_binder_object数据结构中）。

（2）在系统调用接口层面，我们使用ioctl（BINDER_WRITE_READ）来完成具体transaction的递交过程。具体的transaction数据封装在struct binder_write_read对象中，具体如下图所示：

[![wps9D43.tmp](http://www.wowotech.net/content/uploadfile/202002/d7994f45b23cc6bae44185d6789286e420200218133040.jpg "wps9D43.tmp")](http://www.wowotech.net/content/uploadfile/202002/e9fa8e083e46e8db11868cec72bd51b420200218133039.jpg)

（3）Binder驱动创建binder_transaction对象来控制完成本次binder transaction。首先要初始化transaction，具体包括：和谁通信（用户空间通过binder_transaction_data的target成员告知binder驱动transaction的target）、为何通信（binder_transaction_data的code）等

（4）对于每一个service组件，内核都会创建一个binder node与之对应。用户空间通过flat_binder_object这个数据结构把本次要注册的service组件扁平化，传递给binder驱动。驱动根据这个flat_binder_object创建并初始化了该service组件对应的binder node。由于是注册到service manager，也就是说service manager会有一个对本次注册组件的引用，所以需要在target proc（即service manager）中建立一个binder ref对象（指向这个要注册的binder实体）并分配一个handle。

（5）把一个BINDER_WORK_TRANSACTION_COMPLETE类型的binder work挂入client binder thread的todo list，通知client其请求的transaction已经被binder处理完毕，可以进行其他工作了（当然对于同步binder通信，client一般会通过read类型的ioctl进入阻塞态，等待server端的回应）。

（6）至此，client端已经完成了所有操作，现在我们开始进入server端的数据流了。Binder驱动会把一个BINDER_WORK_TRANSACTION类型的binder work（内嵌在binder transaction）挂入binder线程的todo list，然后唤醒它起来干活。

（7）binder server端会使用ioctl（BINDER_WRITE_READ）进入读阻塞状态，等待client的请求到来。一旦有请求到来，Service manager进程会从binder_thread_read中醒来处理队列上的binder work。所谓处理binder work其实完成client transaction的向上递交过程。具体的transaction数据封装在struct binder_write_read对象中，具体如下图所示：

[![wps9D54.tmp](http://www.wowotech.net/content/uploadfile/202002/a00afc2a8443700e31779bcd0cad82f620200218133042.jpg "wps9D54.tmp")](http://www.wowotech.net/content/uploadfile/202002/f0cacb56ea8ceedd920cd4321ca148cb20200218133042.jpg)

需要强调的一点是：在步骤2中，flat_binder_object传递的是binder node，而这里传递的是handle（即binder ref，步骤4中创建的）

（8）在Service manager进程的用户态，识别了本次transaction的code是add service，那么它会把（service name，handle）数据写入其数据库，完成服务注册。

（9）从transaction的角度看，上半场已经完成。现在开始下半场的transaction的处理，即BC_REPLY的处理。和BC_TRANSACTION处理类似，也是通过binder_ioctl ---> binder_ioctl_write_read ---> binder_thread_write ---> binder_transaction这个调用链条进入binder transaction处理流程的。

（10）和上半场类似，在这里Binder驱动同样会创建一个binder_transaction对象来控制完成本次BC_REPLY的binder transaction。通过thread->transaction_stack可以找到其对应的BC_TRANSACTION的binder transaction对象，进而找到回应给哪一个binder process和thread。后续的处理和上半场类似，这里就不再赘述了。

3、相关数据结构

struct transaction主要用来表示binder client和server之间的一次通信，该数据结构的主要成员如下表所示：

|   |   |
|---|---|
|成员变量|描述|
|work|本次transaction涉及的binder work，它会挂入target proc或者target binder thread的todo list中。|
|from|发起binder通信的线程|
|to_proc|处理binder请求的进程|
|to_thread|处理binder请求的线程|
|buffer|binder通信使用的buffer，当A向B服务请求binder通信的时候，B进程分配buffer，并copy A的数据（user space）到buffer中。这是binder通信唯一一次内存拷贝。|
|code|本次transaction的操作码。Binder server端根据操作码提供相应的服务|
|flags|本次transaction的一些属性标记|
|Priority<br><br>saved_priority|和优先级处理相关的成员|

BC_TRANSACTION、BC_REPLY、BR_TRANSACTION和BR_REPLY这四个协议码的协议数据是struct binder_transaction_data，该数据结构的主要成员如下表所示：

|   |   |
|---|---|
|成员变量|描述|
|target|本次transation去向何方？Target有两种形式，一种是本地binder实体，另外一种是表示远端binder实体的句柄。<br><br>在client向service manager发起transaction的时候，那么target.handle等于0。当该transaction到达service manager的时候，binder实体变成本地对象，因此用<br><br>Target.ptr和cookie来表示。|
|cookie|如果transaction的目的地是本地binder实体，那么这个成员保存了binder实体对象的用户空间地址|
|code|Client和service 组件之间的操作码，binder驱动不关心这个码字。|
|flags|描述transaction特性的flag。例如TF_ONE_WAY说明是同步还是异步binder通信|
|sender_pid<br><br>sender_euid|是谁发起transaction？在binder驱动中会根据当前线程设定。|
|data_size<br><br>offsets_size<br><br>data|本次transaction的数据缓冲区信息。|

flat_binder_object主要用来在进程之间传递Binder对象，该数据结构的主要成员如下表所示：

|   |   |
|---|---|
|成员变量|描述|
|hdr|用来描述Binder对象的类型，目前支持的类型有：<br><br>（1）binder实体（本地service组件）<br><br>（2）Binder句柄（远端的service组件）<br><br>（3）文件描述符<br><br>（4）......<br><br>本文主要关注前两种对象类型|
|Binder<br><br>handle|如果flat_binder_object传递的是本地service组件，那么这个联合体中的binder成员有效，指向本地service组件（用户空间对象）的一个弱引用对象的地址。<br><br>如果flat_binder_object传递的是句柄，那么这个联合体中的handle成员有效，该handle对应的binder ref指向一个binder实体对象。|
|cookie|如果传递的是binder实体，那么这个成员保存了binder实体对象（service组件）的用户空间地址|

struct binder_ref主要用来表示一个对Binder实体对象（binder node）的引用，该数据结构的主要成员如下表所示：

|   |   |
|---|---|
|成员变量|描述|
|data|这个成员最核心的数据是用户空间的句柄|
|rb_node_desc|挂入binder proc的红黑树（key是描述符，userspace的句柄）|
|rb_node_node|挂入binder proc的红黑树（key是binder node）|
|node_entry|挂入binder node的哈希表|
|proc|该binder ref属于哪一个binder proc|
|node|该binder ref引用哪一个binder node|

四、如何和Service组件通信

我们以B进程向A服务组件（位于A进程）发起服务请求为例来说明具体的操作流程。B进程不能直接请求A服务组件的服务，因为B进程唯一获知的信息是A服务组件的名字而已。由于A服务组件已经注册在案，因此service manager已经有（A服务组件名字，句柄）的记录，因此B进程可以通过下面的流程获得A服务组件的信息并建立其代理组件对象：

[![wps9D64.tmp](http://www.wowotech.net/content/uploadfile/202002/f27073bf8922a5b1a672adffa7403bfe20200218133043.jpg "wps9D64.tmp")](http://www.wowotech.net/content/uploadfile/202002/4e183b6e3533a8592ec61af0d6cc602520200218133043.jpg)

B进程首先发起BC_TRANSACTION操作，操作码是CHECK_SERVICE，数据是A服务组件的名字。Service manager找到了句柄后将其封装到BC_REPLY中。这里的句柄是service manager进程的句柄，这个句柄并不能直接被B 进程直接使用，毕竟（进程，句柄）才对应唯一的binder实体。这里的binder driver有一个很关键的操作：把service manager中句柄A转换成B client进程中的句柄B，并封装在BR_REPLY中。这时候（service manager进程，句柄A）和（B client进程，句柄B）都指向A服务组件对应的bind node对象。

一旦定位了A服务组件，那么可以继续进行如下的流程：

[![wps9D75.tmp](http://www.wowotech.net/content/uploadfile/202002/12a6d26e7fb497ca2484c36bfb698fcf20200218133045.jpg "wps9D75.tmp")](http://www.wowotech.net/content/uploadfile/202002/8e4db2f0678afe936a89f233b05fdd2620200218133045.jpg)

五、Binder内存操作

1、逻辑过程

在处理binder transaction的过程中，相关的内存操作如下所示：

[![wps9D85.tmp](http://www.wowotech.net/content/uploadfile/202002/f9ea1a0cae11f3780731eb0c43df7e6b20200218133046.jpg "wps9D85.tmp")](http://www.wowotech.net/content/uploadfile/202002/95c5e9df2164baced7a3e631ab5200ac20200218133046.jpg)

配合上面的流程，内存操作的逻辑过程如下：

（1）在binder client的用户空间中，发起transaction的一方会构建用户数据缓冲区（包括两部分：实际的数据区和offset区），把想要传递到server端的数据填充到缓冲区并封装在binder_transaction_data数据结构中。

（2）binder_transaction_data会被copy到内核态，binder驱动会根据它计算出本次需要binder通信的数据量。

（3）根据binder通信的数据量在server进程的binder VMA分配数据缓冲区（binder buffer是这个缓冲区的控制数据对象），同时根据需要也会分配对应的物理page并建立地址映射，以便用户空间可以访问这段buffer的数据。

（4）建立内核地址空间的映射，把用户空间的binder数据缓冲区拷贝到内核中，然后释放掉该映射。

（5）在把binder buffer的数据传递到server用户空间的时候，我们需要一个binder_transaction_data来描述binder通信的缓冲区数据，这个数据对象需要拷贝到用户地址空间，而binder buffer中的数据则不需要拷贝，因为在上面步骤3中已经建立了地址映射，server进程可以直接访问即可。

2、主要的数据结构

struct binder_alloc用来描述binder进程内存分配器，该数据结构的主要成员如下表所示：

|   |   |
|---|---|
|成员变量|描述|
|vma|binder内存对应的VMA|
|vma_vm_mm|binder进程对应的地址空间描述符|
|buffer|该binder proc能用于binder通信的内存地址。<br><br>该地址是mmap的用户空间虚拟地址。|
|buffers|所有的binder buffers（包括空闲的和正在使用的）|
|free_buffers|空闲binder buffers的红黑树，按照size排序|
|allocated_buffers|已经分配的binder buffers的红黑树，key是buffer address|
|free_async_space|剩余的可用于异步binder通信的内存大小。<br><br>初始化的时候配置为2M（整个binder内存的一半）|
|pages|binder内存区域对应的page们。在reclaim binder内存的时候|
|buffer_size|通过mmap映射的，用于binder通信的缓冲区大小，即binder alloc管理的整个内存的大小。|
|pid|Binder proc的pid|

struct binder_buffer用来描述一个用于binder通信的缓冲区，该数据结构的主要成员如下表所示：

|   |   |
|---|---|
|成员变量|描述|
|entry|挂入binder alloc buffer链表（buffers成员）的节点|
|rb_node|挂入binder alloc红黑树的节点：如果是空闲的buffer，挂入空闲红黑树，如果是已经分配的，挂入已分配红黑树。|
|transaction|Binder缓冲区都是用于某次binder transaction的，这个成员指向对应的transaction。|
|target_node|该buffer的去向哪一个node（service组件）|
|data_size<br><br>offsets_size|Binder缓冲区的数据区域的大小以及offset区域的大小。|
|user_data|该binder buffer的用户空间地址|

参考文献：

1、Android系统源代码情景分析，罗升阳著

2、http://gityuan.com/tags/#binder，袁辉辉的博客

标签: [binder](http://www.wowotech.net/tag/binder)

[![](http://www.wowotech.net/content/uploadfile/201605/ef3e1463542768.png)](http://www.wowotech.net/support_us.html)

« [zRAM内存压缩技术原理与应用](http://www.wowotech.net/memory_management/zram.html) | [Binder从入门到放弃（上）](http://www.wowotech.net/linux_kenrel/binder1.html)»

**评论：**

**该昵称已屏蔽**\
2020-03-26 09:34

同样是人，咋差别这么大！！！！哎...望尘莫及  看不懂

[回复](http://www.wowotech.net/binder2.html#comment-7930)

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

  - ja\
    [@dream：我看完這段也有相同的想法，引用 @dream ...](http://www.wowotech.net/kernel_synchronization/spinlock.html#8922)
  - 元神高手\
    [围观首席power managerment专家](http://www.wowotech.net/pm_subsystem/device_driver_pm.html#8921)
  - 十七\
    [内核空间的映射在系统启动时就已经设定好，并且在所有进程的页表...](http://www.wowotech.net/process_management/context-switch-arch.html#8920)
  - lw\
    [sparse模型和disconti模型没看出来有什么本质区别...](http://www.wowotech.net/memory_management/memory_model.html#8919)
  - 肥饶\
    [一个没设置好就出错](http://www.wowotech.net/linux_kenrel/516.html#8918)
  - orange\
    [点赞点赞，对linuxer的文章总结到位](http://www.wowotech.net/device_model/dt-code-file-struct-parse.html#8917)

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

  - [process identification](http://www.wowotech.net/process_management/process_identification.html)
  - [Linux kernel的中断子系统之（一）：综述](http://www.wowotech.net/irq_subsystem/interrupt_subsystem_architecture.html)
  - [Linux 2.5.43版本的RCU实现（废弃）](http://www.wowotech.net/kernel_synchronization/Linux-2-5-43-RCU.html)
  - [futex基础问答](http://www.wowotech.net/kernel_synchronization/futex.html)
  - [Linux设备模型(2)\_Kobject](http://www.wowotech.net/device_model/kobject.html)

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
