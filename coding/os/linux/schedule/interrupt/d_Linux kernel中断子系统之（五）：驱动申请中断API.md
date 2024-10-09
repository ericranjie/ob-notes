作者：[linuxer](http://www.wowotech.net/author/3 "linuxer") 发布于：2014-9-22 18:33 分类：[中断子系统](http://www.wowotech.net/sort/irq_subsystem)

一、前言

本文主要的议题是作为一个普通的驱动工程师，在撰写自己负责的驱动的时候，如何向Linux Kernel中的中断子系统注册中断处理函数？为了理解注册中断的接口，必须了解一些中断线程化（threaded interrupt handler）的基础知识，这些在第二章描述。第三章主要描述了驱动申请 interrupt line接口API request_threaded_irq的规格。第四章是进入request_threaded_irq的实现细节，分析整个代码的执行过程。

二、和中断相关的linux实时性分析以及中断线程化的背景介绍

1、非抢占式linux内核的实时性

在遥远的过去，linux2.4之前的内核是不支持抢占特性的，具体可以参考下图：

[![sxw](http://www.wowotech.net/content/uploadfile/201409/89cc8133ade45f46ce400d3d2d74ce6f20140922103326.gif "sxw")](http://www.wowotech.net/content/uploadfile/201409/e31cc1446eefc8742e5f01648fadba8a20140922103324.gif)

事情的开始源自高优先级任务（橘色block）由于要等待外部事件（例如网络数据）而进入睡眠，调度器调度了某个低优先级的任务（紫色block）执行。该低优先级任务欢畅的执行，直到触发了一次系统调用（例如通过read()文件接口读取磁盘上的文件等）而进入了内核态。仍然是熟悉的配方，仍然是熟悉的味道，低优先级任务正在执行不会变化，只不过从user space切换到了kernel space。外部事件总是在你不想让它来的时候到来，T0时刻，高优先级任务等待的那个中断事件发生了。

中断虽然发生了，但软件不一定立刻响应，可能由于在内核态执行的某些操作不希望被外部事件打断而主动关闭了中断（或是关闭了CPU的中断，或者MASK了该IRQ number），这时候，中断信号没有立刻得到响应，软件仍然在内核态执行低优先级任务系统调用的代码。在T1时刻，内核态代码由于退出临界区而打开中断（注意：上图中的比例是不协调的，一般而言，linux kernel不会有那么长的关中断时间，上面主要是为了表示清楚，同理，从中断触发到具体中断服务程序的执行也没有那么长，都是为了表述清楚），中断一旦打开，立刻跳转到了异常向量地址，interrupt handler抢占了低优先级任务的执行，进入中断上下文（虽然这时候的current task是低优先级任务，但是中断上下文和它没有任何关系）。

从CPU开始处理中断到具体中断服务程序被执行还需要一个分发的过程。这个期间系统要做的主要操作包括确定HW interrupt ID，确定IRQ Number，ack或者mask中断，调用中断服务程序等。T0到T2之间的delay被称为中断延迟（Interrupt Latency），主要包括两部分，一部分是HW造成的delay（硬件的中断系统识别外部的中断事件并signal到CPU），另外一部分是软件原因（内核代码中由于要保护临界区而关闭中断引起的）。

该中断的服务程序执行完毕（在其执行过程中，T3时刻，会唤醒高优先级任务，让它从sleep状态进入runable状态），返回低优先级任务的系统调用现场，这时候并不存在一个抢占点，低优先级任务要完成系统调用之后，在返回用户空间的时候才出现抢占点。漫长的等待之后，T4时刻，调度器调度高优先级任务执行。有一个术语叫做任务响应时间（Task Response Time）用来描述T3到T4之间的delay。

2、抢占式linux内核的实时性

2.6内核和2.4内核显著的不同是提供了一个CONFIG_PREEMPT的选项，打开该选项后，linux kernel就支持了内核代码的抢占（当然不能在临界区），其行为如下：

[![pre](http://www.wowotech.net/content/uploadfile/201409/5d3893a4997db31f1bcbec9c8446b98020140922103329.gif "pre")](http://www.wowotech.net/content/uploadfile/201409/ed95106e2c04e02b2ba84fe038d70f2220140922103327.gif)

T0到T3的操作都是和上一节的描述一样的，不同的地方是在T4。对于2.4内核，只有返回用户空间的时候才有抢占点出现，但是对于抢占式内核而言，即便是从中断上下文返回内核空间的进程上下文，只要内核代码不在临界区内，就可以发生调度，让最高优先级的任务调度执行。

在非抢占式linux内核中，一个任务的内核态是不可以被其他进程抢占的。这里并不是说kernel space不可以被抢占，只是说进程通过系统调用陷入到内核的时候，不可以被其他的进程抢占。实际上，中断上下文当然可以抢占进程上下文（无论是内核态还是用户态），更进一步，中断上下文是拥有至高无上的权限，它甚至可以抢占其他的中断上下文。引入抢占式内核后，系统的平均任务响应时间会缩短，但是，实时性更关注的是：无论在任何的负载情况下，任务响应时间是确定的。因此，更需要关注的是worst-case的任务响应时间。这里有两个因素会影响worst case latency：

（1）为了同步，内核中总有些代码需要持有自旋锁资源，或者显式的调用preempt_disable来禁止抢占，这时候不允许抢占

（2）中断上下文（并非只是中断handler，还包括softirq、timer、tasklet）总是可以抢占进程上下文

因此，即便是打开了PREEMPT的选项，实际上linux系统的任务响应时间仍然是不确定的。一方面内核代码的临界区非常多，我们需要找到，系统中持有锁，或者禁止抢占的最大的时间片。另外一方面，在上图的T4中，能顺利的调度高优先级任务并非易事，这时候可能有触发的软中断，也可能有新来的中断，也可能某些driver的tasklet要执行，只有在没有任何bottom half的任务要执行的时候，调度器才会启动调度。

3、进一步提高linux内核的实时性

通过上一个小节的描述，相信大家都确信中断对linux 实时性的最大的敌人。那么怎么破？我曾经接触过一款RTOS，它的中断handler非常简单，就是发送一个inter-task message到该driver thread，对任何的一个驱动都是如此处理。这样，每个中断上下文都变得非常简短，而且每个中断都是一致的。在这样的设计中，外设中断的处理线程化了，然后，系统设计师要仔细的为每个系统中的task分配优先级，确保整个系统的实时性。

在Linux kernel中，一个外设的中断处理被分成top half和bottom half，top half进行最关键，最基本的处理，而比较耗时的操作被放到bottom half（softirq、tasklet）中延迟执行。虽然bottom half被延迟执行，但始终都是先于进程执行的。为何不让这些耗时的bottom half和普通进程公平竞争呢？因此，linux kernel借鉴了RTOS的某些特性，对那些耗时的驱动interrupt handler进行线程化处理，在内核的抢占点上，让线程（无论是内核线程还是用户空间创建的线程，还是驱动的interrupt thread）在一个舞台上竞争CPU。

三、request_threaded_irq的接口规格

1、输入参数描述

|   |   |
|---|---|
|输入参数|描述|
|irq|要注册handler的那个IRQ number。这里要注册的handler包括两个，一个是传统意义的中断handler，我们称之primary handler，另外一个是threaded interrupt handler|
|handler|primary handler。需要注意的是primary handler和threaded interrupt handler不能同时为空，否则会出错|
|thread_fn|threaded interrupt handler。如果该参数不是NULL，那么系统会创建一个kernel thread，调用的function就是thread_fn|
|irqflags|参见本章第三节|
|devname||
|dev_id|参见第四章，第一节中的描述。|

2、输出参数描述

0表示成功执行，负数表示各种错误原因。

3、Interrupt type flags

|   |   |
|---|---|
|flag定义|描述|
|IRQF_TRIGGER_XXX|描述该interrupt line触发类型的flag|
|IRQF_DISABLED|首先要说明的是这是一个废弃的flag，在新的内核中，该flag没有任何的作用了。具体可以参考：[Disabling IRQF_DISABLED](http://lwn.net/Articles/380931/ "http://lwn.net/Articles/380931/")  <br>旧的内核（2.6.35版本之前）认为有两种interrupt handler：slow handler和fast handle。在request irq的时候，对于fast handler，需要传递IRQF_DISABLED的参数，确保其中断处理过程中是关闭CPU的中断，因为是fast handler，执行很快，即便是关闭CPU中断不会影响系统的性能。但是，并不是每一种外设中断的handler都是那么快（例如磁盘），因此就有 slow handler的概念，说明其在中断处理过程中会耗时比较长。对于这种情况，在执行interrupt handler的时候不能关闭CPU中断，否则对系统的performance会有影响。  <br>新的内核已经不区分slow handler和fast handle，都是fast handler，都是需要关闭CPU中断的，那些需要后续处理的内容推到threaded interrupt handler中去执行。|
|IRQF_SHARED|这是flag用来描述一个interrupt line是否允许在多个设备中共享。如果中断控制器可以支持足够多的interrupt source，那么在两个外设间共享一个interrupt request line是不推荐的，毕竟有一些额外的开销（发生中断的时候要逐个询问是不是你的中断，软件上就是遍历action list），因此外设的irq handler中最好是一开始就启动判断，看看是否是自己的中断，如果不是，返回IRQ_NONE,表示这个中断不归我管。 早期PC时代，使用8259中断控制器，级联的8259最多支持15个外部中断，但是PC外设那么多，因此需要irq share。现在，ARM平台上的系统设计很少会采用外设共享IRQ方式，毕竟一般ARM SOC提供的有中断功能的GPIO非常的多，足够用的。 当然，如果确实需要两个外设共享IRQ，那也只能如此设计了。对于HW，中断控制器的一个interrupt source的引脚要接到两个外设的interrupt request line上，怎么接？直接连接可以吗？当然不行，对于低电平触发的情况，我们可以考虑用与门连接中断控制器和外设。|
|IRQF_PROBE_SHARED|IRQF_SHARED用来表示该interrupt action descriptor是允许和其他device共享一个interrupt line（IRQ number），但是实际上是否能够share还是需要其他条件：例如触发方式必须相同。有些驱动程序可能有这样的调用场景：我只是想scan一个irq table，看看哪一个是OK的，这时候，如果即便是不能和其他的驱动程序share这个interrupt line，我也没有关系，我就是想scan看看情况。这时候，caller其实可以预见sharing mismatche的发生，因此，不需要内核打印“Flags mismatch irq……“这样冗余的信息|
|IRQF_PERCPU|在SMP的架构下，中断有两种mode，一种中断是在所有processor之间共享的，也就是global的，一旦中断产生，interrupt controller可以把这个中断送达多个处理器。当然，在具体实现的时候不会同时将中断送达多个CPU，一般是软件和硬件协同处理，将中断送达一个CPU处理。但是一段时间内产生的中断可以平均（或者按照既定的策略）分配到一组CPU上。这种interrupt mode下，interrupt controller针对该中断的operational register是global的，所有的CPU看到的都是一套寄存器，一旦一个CPU ack了该中断，那么其他的CPU看到的该interupt source的状态也是已经ack的状态。  <br>和global对应的就是per cpu interrupt了，对于这种interrupt，不是processor之间共享的，而是特定属于一个CPU的。例如GIC中interrupt ID等于30的中断就是per cpu的（这个中断event被用于各个CPU的local timer），这个中断号虽然只有一个，但是，实际上控制该interrupt ID的寄存器有n组（如果系统中有n个processor），每个CPU看到的是不同的控制寄存器。在具体实现中，这些寄存器组有两种形态，一种是banked，所有CPU操作同样的寄存器地址，硬件系统会根据访问的cpu定向到不同的寄存器，另外一种是non banked，也就是说，对于该interrupt source，每个cpu都有自己独特的访问地址。|
|IRQF_NOBALANCING|这也是和multi-processor相关的一个flag。对于那些可以在多个CPU之间共享的中断，具体送达哪一个processor是有策略的，我们可以在多个CPU之间进行平衡。如果你不想让你的中断参与到irq balancing的过程中那么就设定这个flag|
|IRQF_IRQPOLL||
|IRQF_ONESHOT|one shot本身的意思的只有一次的，结合到中断这个场景，则表示中断是一次性触发的，不能嵌套。对于primary handler，当然是不会嵌套，但是对于threaded interrupt handler，我们有两种选择，一种是mask该interrupt source，另外一种是unmask该interrupt source。一旦mask住该interrupt source，那么该interrupt source的中断在整个threaded interrupt handler处理过程中都是不会再次触发的，也就是one shot了。这种handler不需要考虑重入问题。  <br>具体是否要设定one shot的flag是和硬件系统有关的，我们举一个例子，比如电池驱动，电池里面有一个电量计，是使用HDQ协议进行通信的，电池驱动会注册一个threaded interrupt handler，在这个handler中，会通过HDQ协议和电量计进行通信。对于这个handler，通过HDQ进行通信是需要一个完整的HDQ交互过程，如果中间被打断，整个通信过程会出问题，因此，这个handler就必须是one shot的。|
|IRQF_NO_SUSPEND|这个flag比较好理解，就是说在系统suspend的时候，不用disable这个中断，如果disable，可能会导致系统不能正常的resume。|
|IRQF_FORCE_RESUME|在系统resume的过程中，强制必须进行enable的动作，即便是设定了IRQF_NO_SUSPEND这个flag。这是和特定的硬件行为相关的。|
|IRQF_NO_THREAD|有些low level的interrupt是不能线程化的（例如系统timer的中断），这个flag就是起这个作用的。另外，有些级联的interrupt controller对应的IRQ也是不能线程化的（例如secondary GIC对应的IRQ），它的线程化可能会影响一大批附属于该interrupt controller的外设的中断响应延迟。|
|IRQF_EARLY_RESUME||
|IRQF_TIMER||

四、request_threaded_irq代码分析

1、request_threaded_irq主流程

> int request_threaded_irq(unsigned int irq, irq_handler_t handler,\
> irq_handler_t thread_fn, unsigned long irqflags,\
> const char \*devname, void \*dev_id)\
> { \
> if ((irqflags & IRQF_SHARED) && !dev_id)－－－－－－－－－（1）\
> return -EINVAL;
>
> desc = irq_to_desc(irq); －－－－－－－－－－－－－－－－－（2）\
> if (!desc)         return -EINVAL;
>
> if (!irq_settings_can_request(desc) || －－－－－－－－－－－－（3）\
> WARN_ON(irq_settings_is_per_cpu_devid(desc)))\
> return -EINVAL;
>
> if (!handler) { －－－－－－－－－－－－－－－－－－－－－－（4）\
> if (!thread_fn)\
> return -EINVAL;\
> handler = irq_default_primary_handler;\
> }
>
> action = kzalloc(sizeof(struct irqaction), GFP_KERNEL);
>
> action->handler = handler;\
> action->thread_fn = thread_fn;\
> action->flags = irqflags;\
> action->name = devname;\
> action->dev_id = dev_id;
>
> chip_bus_lock(desc);\
> retval = \_\_setup_irq(irq, desc, action); －－－－－－－－－－－（5）\
> chip_bus_sync_unlock(desc);\
> }

（1）对于那些需要共享的中断，在request irq的时候需要给出dev id，否则会出错退出。为何对于IRQF_SHARED的中断必须要给出dev id呢？实际上，在共享的情况下，一个IRQ number对应若干个irqaction，当操作irqaction的时候，仅仅给出IRQ number就不是非常的足够了，这时候，需要一个ID表示具体的irqaction，这里就是dev_id的作用了。我们举一个例子：

> void free_irq(unsigned int irq, void \*dev_id)

当释放一个IRQ资源的时候，不但要给出IRQ number，还要给出device ID。只有这样，才能精准的把要释放的那个irqaction 从irq action list上移除。dev_id在中断处理中有没有作用呢？我们来看看source code：

> irqreturn_t handle_irq_event_percpu(struct irq_desc \*desc, struct irqaction \*action)\
> {
>
> do {\
> irqreturn_t res; \
> res = action->handler(irq, action->dev_id);
>
> ……\
> action = action->next;\
> } while (action);
>
> ……\
> }

linux interrupt framework虽然支持中断共享，但是它并不会协助解决识别问题，它只会遍历该IRQ number上注册的irqaction的callback函数，这样，虽然只是一个外设产生的中断，linux kernel还是把所有共享的那些中断handler都逐个调用执行。为了让系统的performance不受影响，irqaction的callback函数必须在函数的最开始进行判断，是否是自己的硬件设备产生了中断（读取硬件的寄存器），如果不是，尽快的退出。

需要注意的是，这里dev_id并不能在中断触发的时候用来标识需要调用哪一个irqaction的callback函数，通过上面的代码也可以看出，dev_id有些类似一个参数传递的过程，可以把具体driver的一些硬件信息，组合成一个structure，在触发中断的时候可以把这个structure传递给中断处理函数。

（2）通过IRQ number获取对应的中断描述符。在引入CONFIG_SPARSE_IRQ选项后，这个转换变得不是那么简单了。在过去，我们会以IRQ number为index，从irq_desc这个全局数组中直接获取中断描述符。如果配置CONFIG_SPARSE_IRQ选项，则需要从radix tree中搜索。CONFIG_SPARSE_IRQ选项的更详细的介绍请参考[IRQ number和中断描述符](http://www.wowotech.net/linux_kenrel/interrupt_descriptor.html)

（3）并非系统中所有的IRQ number都可以request，有些中断描述符被标记为IRQ_NOREQUEST，标识该IRQ number不能被其他的驱动request。一般而言，这些IRQ number有特殊的作用，例如用于级联的那个IRQ number是不能request。irq_settings_can_request函数就是判断一个IRQ是否可以被request。

irq_settings_is_per_cpu_devid函数用来判断一个中断描述符是否需要传递per cpu的device ID。per cpu的中断上面已经描述的很清楚了，这里不再细述。如果一个中断描述符对应的中断 ID是per cpu的，那么在申请其handler的时候就有两种情况，一种是传递统一的dev_id参数（传入request_threaded_irq的最后一个参数），另外一种情况是针对每个CPU，传递不同的dev_id参数。在这种情况下，我们需要调用request_percpu_irq接口函数而不是request_threaded_irq。

（4）传入request_threaded_irq的primary handler和threaded handler参数有下面四种组合：

|   |   |   |
|---|---|---|
|primary handler|threaded handler|描述|
|NULL|NULL|函数出错，返回-EINVAL|
|设定|设定|正常流程。中断处理被合理的分配到primary handler和threaded handler中。|
|设定|NULL|中断处理都是在primary handler中完成|
|NULL|设定|这种情况下，系统会帮忙设定一个default的primary handler：irq_default_primary_handler，协助唤醒threaded handler线程|

（5）这部分的代码很简单，分配struct irqaction，赋值，调用\_\_setup_irq进行实际的注册过程。这里要罗嗦几句的是锁的操作，在内核中，有很多函数，有的是需要调用者自己加锁保护的，有些是不需要加锁保护的。对于这些场景，linux kernel采取了统一的策略：基本函数名字是一样的，只不过需要调用者自己加锁保护的那个函数需要增加\_\_的前缀，例如内核有有下面两个函数：setup_irq和\_\_setup_irq。这里，我们在setup irq的时候已经调用chip_bus_lock进行保护，因此使用lock free的版本\_\_setup_irq。

chip_bus_lock定义如下：

> static inline void chip_bus_lock(struct irq_desc \*desc)\
> {\
> if (unlikely(desc->irq_data.chip->irq_bus_lock))\
> desc->irq_data.chip->irq_bus_lock(&desc->irq_data);\
> }

大部分的interrupt controller并没有定义irq_bus_lock这个callback函数，因此chip_bus_lock这个函数对大多数的中断控制器而言是没有实际意义的。但是，有些interrupt controller是连接到慢速总线上的，例如一个i2c接口的IO expander芯片（这种芯片往往也提供若干有中断功能的GPIO，因此也是一个interrupt controller），在访问这种interrupt controller的时候需要lock住那个慢速bus（只能有一个client在使用I2C bus）。

2、注册irqaction

（1）nested IRQ的处理代码

在看具体的代码之前，我们首先要理解什么是nested IRQ。nested IRQ不是cascade IRQ，在[GIC代码分析](http://www.wowotech.net/linux_kenrel/gic_driver.html)中我们有描述过cascade IRQ这个概念，主要用在interrupt controller级联的情况下。为了方便大家理解，我还是给出一个具体的例子吧，具体的HW block请参考下图：

[![SOC-INT](http://www.wowotech.net/content/uploadfile/201409/f2140c5be79f3100a5c54fdf4228a51a20140922103331.gif "SOC-INT")](http://www.wowotech.net/content/uploadfile/201409/a03daab0b60c21e763d7f367c449fb8820140922103330.gif)

上图是一个两个GIC级联的例子，所有的HW block封装在了一个SOC chip中。为了方便描述，我们先进行编号：Secondary GIC的IRQ number是A，外设1的IRQ number是B，外设2的IRQ number是C。对于上面的硬件，linux kernel处理如下：

（a）IRQ A的中断描述符被设定为不能注册irqaction（不能注册specific interrupt handler，或者叫中断服务程序）

（b）IRQ A的highlevel irq-events handler（处理interrupt flow control）负责进行IRQ number的映射，在其irq domain上翻译出具体外设的IRQ number，并重新定向到外设IRQ number对应的highlevel irq-events handler。

（c）所有外设驱动的中断正常request irq，可以任意选择线程化的handler，或者只注册primary handler。

需要注意的是，对root GIC和Secondary GIC寄存器的访问非常快，因此ack、mask、EOI等操作也非常快。

我们再看看另外一个interrupt controller级联的情况：

[![nested](http://www.wowotech.net/content/uploadfile/201409/eb9047b8462ce224c9958af7e42dcc6820140922103334.gif "nested")](http://www.wowotech.net/content/uploadfile/201409/123d27cb3ef3f846a1f12b7ff969be0120140922103332.gif)

IO expander HW block提供了有中断功能的GPIO，因此它也是一个interrupt controller，有它自己的irq domain和irq chip。上图中外设1和外设2使用了IO expander上有中断功能的GPIO，它们有属于自己的IRQ number以及中断描述符。IO expander HW block的IRQ line连接到SOC内部的interrupt controller上，因此，这也是一个interrupt controller级联的情况，对于这种情况，我们是否可以采用和上面GIC级联的处理方式呢？

不行，对于GIC级联的情况，如果secondary GIC上的外设1产生了中断，整个关中断的时间是IRQ A的中断描述符的highlevel irq-events handler处理时间＋IRQ B的的中断描述符的highlevel irq-events handler处理时间＋外设1的primary handler的处理时间。所幸对root GIC和Secondary GIC寄存器的访问非常快，因此整个关中断的时间也不是非常的长。但是如果是IO expander这个情况，如果采取和上面GIC级联的处理方式一样的话，关中断的时间非常长。我们还是用外设1产生的中断为例子好了。这时候，由于IRQ B的的中断描述符的highlevel irq-events handler处理设计I2C的操作，因此时间非常的长，这时候，对于整个系统的实时性而言是致命的打击。对这种硬件情况，linux kernel处理如下：

（a）IRQ A的中断描述符的highlevel irq-events handler根据实际情况进行设定，并且允许注册irqaction。对于连接到IO expander上的外设，它是没有real time的要求的（否则也不会接到IO expander上），因此一般会进行线程化处理。由于threaded handler中涉及I2C操作，因此要设定IRQF_ONESHOT的flag。

（b）在IRQ A的中断描述符的threaded interrupt handler中进行进行IRQ number的映射，在IO expander irq domain上翻译出具体外设的IRQ number，并直接调用handle_nested_irq函数处理该IRQ。

（c）外设对应的中断描述符设定IRQ_NESTED_THREAD的flag，表明这是一个nested IRQ。nested IRQ没有highlevel irq-events handler，也没有primary handler，它的threaded interrupt handler是附着在其parent IRQ的threaded handler上的。

具体的nested IRQ的处理代码如下：

> static int \_\_setup_irq(unsigned int irq, struct irq_desc \*desc, struct irqaction \*new)\
> {
>
> ……\
> **nested = irq_settings_is_nested_thread(desc);\
> if (nested) {\
> if (!new->thread_fn) {\
> ret = -EINVAL;\
> goto out_mput;\
> }\
> new->handler = irq_nested_primary_handler;**\
> } else { \
> ……\
> }
>
> ……
>
> }

如果一个中断描述符是nested thread type的，说明这个中断描述符应该设定threaded interrupt handler（当然，内核是不会单独创建一个thread的，它是借着其parent IRQ的interrupt thread执行），否则就会出错返回。对于primary handler，它应该没有机会被调用到，当然为了调试，kernel将其设定为irq_nested_primary_handler，以便在调用的时候打印一些信息，让工程师直到发生了什么状况。

（2）forced irq threading处理

具体的forced irq threading的处理代码如下：

> static int \_\_setup_irq(unsigned int irq, struct irq_desc \*desc, struct irqaction \*new)\
> {
>
> ……\
> nested = irq_settings_is_nested_thread(desc);\
> if (nested) { \
> ……\
> } else {\
> **if (irq_settings_can_thread(desc))\
> irq_setup_forced_threading(new);**\
> }
>
> ……
>
> }

forced irq threading其实就是将系统中所有可以被线程化的中断handler全部线程化，即便你在request irq的时候，设定的是primary handler，而不是threaded handler。当然那些不能被线程化的中断（标注了IRQF_NO_THREAD的中断，例如系统timer）还是排除在外的。irq_settings_can_thread函数就是判断一个中断是否可以被线程化，如果可以的话，则调用irq_setup_forced_threading在set irq的时候强制进行线程化。具体代码如下：

> static void irq_setup_forced_threading(struct irqaction \*new)\
> {\
> if (!force_irqthreads)－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－（a）\
> return;\
> if (new->flags & (IRQF_NO_THREAD | IRQF_PERCPU | IRQF_ONESHOT))－－－－－－－（b）\
> return;
>
> new->flags |= IRQF_ONESHOT; －－－－－－－－－－－－－－－－－－－－－－－－－（d）
>
> if (!new->thread_fn) {－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－（c）\
> set_bit(IRQTF_FORCED_THREAD, &new->thread_flags);\
> new->thread_fn = new->handler;\
> new->handler = irq_default_primary_handler;\
> }\
> }

（a）系统中有一个强制线程化的选项：CONFIG_IRQ_FORCED_THREADING，如果没有打开该选项，force_irqthreads总是0，因此irq_setup_forced_threading也就没有什么作用，直接return了。如果打开了CONFIG_IRQ_FORCED_THREADING，说明系统支持强制线程化，但是具体是否对所有的中断进行强制线程化处理还是要看命令行参数threadirqs。如果kernel启动的时候没有传入该参数，那么同样的，irq_setup_forced_threading也就没有什么作用，直接return了。只有bootloader向内核传入threadirqs这个命令行参数，内核才真正在启动过程中，进行各个中断的强制线程化的操作。

（b）看到IRQF_NO_THREAD选项你可能会奇怪，前面irq_settings_can_thread函数不是检查过了吗？为何还要重复检查？其实一个中断是否可以进行线程化可以从两个层面看：一个是从底层看，也就是从中断描述符、从实际的中断硬件拓扑等方面看。另外一个是从中断子系统的用户层面看，也就是各个外设在注册自己的handler的时候是否想进行线程化处理。所有的IRQF_XXX都是从用户层面看的flag，因此如果用户通过IRQF_NO_THREAD这个flag告知kernel，该interrupt不能被线程化，那么强制线程化的机制还是尊重用户的选择的。

PER CPU的中断都是一些较为特殊的中断，不是一般意义上的外设中断，因此对PER CPU的中断不强制进行线程化。IRQF_ONESHOT选项说明该中断已经被线程化了（而且是特殊的one shot类型的），因此也是直接返回了。

（c）强制线程化只对那些没有设定thread_fn的中断进行处理，这种中断将全部的处理放在了primary interrupt handler中（当然，如果中断处理比较耗时，那么也可能会采用bottom half的机制），由于primary interrupt handler是全程关闭CPU中断的，因此可能对系统的实时性造成影响，因此考虑将其强制线程化。struct irqaction中的thread_flags是和线程相关的flag，我们给它打上IRQTF_FORCED_THREAD的标签，表明该threaded handler是被强制threaded的。new->thread_fn = new->handler这段代码表示将原来primary handler中的内容全部放到threaded handler中处理，新的primary handler被设定为default handler。

（d）强制线程化是一个和实时性相关的选项，从我的角度来看是一个很不好的设计（个人观点），各个驱动工程师在撰写自己的驱动代码的时候已经安排好了自己的上下文环境。有的是进程上下文，有的是中断上下文，在各自的上下文环境中，驱动工程师又选择了适合的内核同步机制。但是，强制线程化导致原来运行在中断上下文的primary handler现在运行在进程上下文，这有可能导致一些难以跟踪和定位的bug。

当然，作为内核的开发者，既然同意将强制线程化这个特性并入linux kernel，相信他们有他们自己的考虑。我猜测这是和一些旧的驱动代码维护相关的。linux kernel中的中断子系统的API的修改会引起非常大的震动，因为内核中成千上万的驱动都是需要调用旧的接口来申请linux kernel中断子系统的服务，对每一个驱动都进行修改是一个非常耗时的工作，为了让那些旧的驱动代码可以运行在新的中断子系统上，因此，在kernel中，实际上仍然提供了旧的request_irq接口函数，如下：

> static inline int \_\_must_check\
> request_irq(unsigned int irq, irq_handler_t handler, unsigned long flags,\
> const char \*name, void \*dev)\
> {\
> return request_threaded_irq(irq, handler, NULL, flags, name, dev);\
> }

接口是OK了，但是，新的中断子系统的思路是将中断处理分成primary handler和threaded handler，而旧的驱动代码一般是将中断处理分成top half和bottom half，如何将这部分的不同抹平？linux kernel是这样处理的（这是我个人的理解，不保证是正确的）：

（d-1）内核为那些被强制线程化的中断handler设定了IRQF_ONESHOT的标识。这是因为在旧的中断处理机制中，top half是不可重入的，强制线程化之后，强制设定IRQF_ONESHOT可以保证threaded handler是不会重入的。

（d-2）在那些被强制线程化的中断线程中，disable bottom half的处理。这是因为在旧的中断处理机制中，botton half是不可能抢占top half的执行，强制线程化之后，应该保持这一点。

（3）创建interrupt线程。代码如下：

> if (new->thread_fn && !nested) {\
> struct task_struct \*t;\
> static const struct sched_param param = {\
> .sched_priority = MAX_USER_RT_PRIO/2,\
> };
>
> t = kthread_create(irq_thread, new, "irq/%d-%s", irq,－－－－－－－－－－－－－－－－－－（a）\
> new->name);
>
> sched_setscheduler_nocheck(t, SCHED_FIFO, ¶m);
>
> get_task_struct(t);－－－－－－－－－－－－－－－－－－－－－－－－－－－（b）\
> new->thread = t;
>
> set_bit(IRQTF_AFFINITY, &new->thread_flags);－－－－－－－－－－－－－－－（c）\
> }
>
> if (!alloc_cpumask_var(&mask, GFP_KERNEL)) {－－－－－－－－－－－－－－－－（d）\
> ret = -ENOMEM;\
> goto out_thread;\
> }\
> if (desc->irq_data.chip->flags & IRQCHIP_ONESHOT_SAFE)－－－－－－－－－－－（e）\
> new->flags &= ~IRQF_ONESHOT;

（a）调用kthread_create来创建一个内核线程，并调用sched_setscheduler_nocheck来设定这个中断线程的调度策略和调度优先级。这些是和进程管理相关的内容，我们留到下一个专题再详细描述吧。

（b）调用get_task_struct可以为这个threaded handler的task struct增加一次reference count，这样，即便是该thread异常退出也可以保证它的task struct不会被释放掉。这可以保证中断系统的代码不会访问到一些被释放的内存。irqaction的thread 成员被设定为刚刚创建的task，这样，primary handler就知道唤醒哪一个中断线程了。

（c）设定IRQTF_AFFINITY的标志，在threaded handler中会检查该标志并进行IRQ affinity的设定。

（d）分配一个cpu mask的变量的内存，后面会使用到

（e）驱动工程师是撰写具体外设驱动的，他/她未必会了解到底层的一些具体的interrupt controller的信息。有些interrupt controller（例如MSI based interrupt）本质上就是就是one shot的（通过IRQCHIP_ONESHOT_SAFE标记），因此驱动工程师设定的IRQF_ONESHOT其实是画蛇添足，因此可以去掉。

（4）共享中断的检查。代码如下：

> old_ptr = &desc->action;\
> old = \*old_ptr;
>
> if (old) {\
> if (!((old->flags & new->flags) & IRQF_SHARED) ||－－－－－－－－－－－－－－－－－（a）\
> ((old->flags ^ new->flags) & IRQF_TRIGGER_MASK) ||\
> ((old->flags ^ new->flags) & IRQF_ONESHOT))\
> goto mismatch;
>
> /\* All handlers must agree on per-cpuness \*/\
> if ((old->flags & IRQF_PERCPU) != (new->flags & IRQF_PERCPU))\
> goto mismatch;
>
> /\* add new interrupt at end of irq queue \*/\
> do {－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－（b）\
> thread_mask |= old->thread_mask;\
> old_ptr = &old->next;\
> old = \*old_ptr;\
> } while (old);\
> shared = 1;\
> }

（a）old指向注册之前的action list，如果不是NULL，那么说明需要共享interrupt line。但是如果要共享，需要每一个irqaction都同意共享（IRQF_SHARED），每一个irqaction的触发方式相同（都是level trigger或者都是edge trigger），相同的oneshot类型的中断（都是one shot或者都不是），per cpu类型的相同中断（都是per cpu的中断或者都不是）。

（b）将该irqaction挂入队列的尾部。

（5）thread mask的设定。代码如下：

> if (new->flags & IRQF_ONESHOT) {\
> if (thread_mask == ~0UL) {－－－－－－－－－－－－－－－－－－－－－－－－（a）\
> ret = -EBUSY;\
> goto out_mask;\
> }\
> new->thread_mask = 1 \<\< ffz(thread_mask);
>
> } else if (new->handler == irq_default_primary_handler &&\
> !(desc->irq_data.chip->flags & IRQCHIP_ONESHOT_SAFE)) {－－－－－－－－（b）\
> ret = -EINVAL;\
> goto out_mask;\
> }

对于one shot类型的中断，我们还需要设定thread mask。如果一个one shot类型的中断只有一个threaded handler（不支持共享），那么事情就很简单（临时变量thread_mask等于0），该irqaction的thread_mask成员总是使用第一个bit来标识该irqaction。但是，如果支持共享的话，事情变得有点复杂。我们假设这个one shot类型的IRQ上有A，B和C三个irqaction，那么A，B和C三个irqaction的thread_mask成员会有不同的bit来标识自己。例如A的thread_mask成员是0x01，B的是0x02，C的是0x04，如果有更多共享的irqaction（必须是oneshot类型），那么其thread_mask成员会依次设定为0x08，0x10……

（a）在上面“共享中断的检查”这个section中，thread_mask变量保存了所有的属于该interrupt line的thread_mask，这时候，如果thread_mask变量如果是全1，那么说明irqaction list上已经有了太多的irq action（大于32或者64，和具体系统和编译器相关）。如果没有满，那么通过ffz函数找到第一个为0的bit作为该irq action的thread bit mask。

（b）irq_default_primary_handler的代码如下：

> static irqreturn_t irq_default_primary_handler(int irq, void \*dev_id)\
> {\
> return IRQ_WAKE_THREAD;\
> }

代码非常的简单，返回IRQ_WAKE_THREAD，让kernel唤醒threaded handler就OK了。使用irq_default_primary_handler虽然简单，但是有一个风险：如果是电平触发的中断，我们需要操作外设的寄存器才可以让那个asserted的电平信号消失，否则它会一直持续。一般，我们都是直接在primary中操作外设寄存器（slow bus类型的interrupt controller不行），尽早的clear interrupt，但是，对于irq_default_primary_handler，它仅仅是wakeup了threaded interrupt handler，并没有clear interrupt，这样，执行完了primary handler，外设中断仍然是asserted，一旦打开CPU中断，立刻触发下一次的中断，然后不断的循环。因此，如果注册中断的时候没有指定primary interrupt handler，并且没有设定IRQF_ONESHOT，那么系统是会报错的。当然，有一种情况可以豁免，当底层的irq chip是one shot safe的（IRQCHIP_ONESHOT_SAFE）。

（6）用户IRQ flag和底层interrupt flag的同步（TODO）

_原创文章，转发请注明出处。蜗窝科技。[http://www.wowotech.net/linux_kenrel/request_threaded_irq.html](http://www.wowotech.net/linux_kenrel/request_threaded_irq.html)_

标签: [request_threaded_irq](http://www.wowotech.net/tag/request_threaded_irq)

[![](http://www.wowotech.net/content/uploadfile/201605/ef3e1463542768.png)](http://www.wowotech.net/support_us.html)

« [Linux设备模型(9)\_device resource management](http://www.wowotech.net/device_model/device_resource_management.html) | [Linux电源管理(10)\_autosleep](http://www.wowotech.net/pm_subsystem/autosleep.html)»

**评论：**

**每天一小步**\
2020-08-04 20:44

请教一下：\
“（d-2）在那些被强制线程化的中断线程中，disable bottom half的处理。这是因为在旧的中断处理机制中，botton half是不可能抢占top half的执行，强制线程化之后，应该保持这一点。”

有两个问题：\
1）“disable bottom half的处理” 这个源码在哪里？\
2）那些被强制线程化的中断处理函数作者可能并不知道自己被强制线程化了，他有可能会在自己认为的中断上半部启动一个下半部机制来处理后面的工作，如果强制线程化机制把bottom half diable了，那本来该在下半部中处理的业务不是不能完成了吗？

[回复](http://www.wowotech.net/irq_subsystem/request_threaded_irq.html#comment-8081)

**小学生**\
2020-04-22 19:06

“由于IRQ B的的中断描述符的highlevel irq-events handler处理   设计   I2C的操作”

应该是

“由于IRQ B的的中断描述符的highlevel irq-events handler处理   涉及   I2C的操作”

[回复](http://www.wowotech.net/irq_subsystem/request_threaded_irq.html#comment-7967)

**albert**\
2020-03-10 10:39

建一个中断后如何设置高级别的优先级以保证该中断的实时性？有没办法让中断一直保持高优先级？

[回复](http://www.wowotech.net/irq_subsystem/request_threaded_irq.html#comment-7914)

**wate**\
2022-06-30 09:42

@albert：如果只从保持高优先级角度看的话，那只有让中断处理程序处于中断上下文时有最高优先级了，使用IRQF_NOTHREAD在request_thread_irq()注册时明确指定不线程化应该是可以满足高优先级的问题。否则，线程化之后会被内核高优先级线程欺负的。

[回复](http://www.wowotech.net/irq_subsystem/request_threaded_irq.html#comment-8635)

**bing**\
2019-12-20 18:32

请问，我在使用request_thread_irq()的时候，总会进入一次中断函数，看着又不像是出发了中断条件，具体需要怎么查找原因呢，感谢

[回复](http://www.wowotech.net/irq_subsystem/request_threaded_irq.html#comment-7790)

**chili**\
2019-07-29 19:03

探讨个非常有意思的问题，不知道linux的中断底层是怎么去处理的。\
背景：用linux的gpio的中断去捕获并计算一系列1ms~2ms左右的脉宽（在中断中利用ktime_get获得时间并相减获得）\
测试1：利用linux的两个gpio来测试中断latency，短接2个pin，然后利用hrtimer定期产生1ms的timer来toggle pin1的状态；然后pin2申请上升沿和下降沿中断。进入pin2中断的时间减去pin1状态改变的时间即为latency，统计多次取平均值，结果如下：\
CPU     HZ          Version         times          latency\
4\*A53   1.8G        4.14.78         10000 passes, avg. 1429 nsecs\
Pi        700mhz    3.6.11+            1024 passes. avg. 3770 nsecs\
【初步结论】linux虽然不是实时操作系统，但是中断延时似乎还是很小的，但是这里是平均值，可能平滑掉了一些延时非常大的凸点，于是做了第二个实验

测试2：利用linux的两个gpio来测试中断latency，短接2个pin，然后利用hrtimer定期产生us级别的timer来toggle pin1的状态；然后pin2申请上升沿和下降沿中断，记录下pin2连续两次进入中断时间间隔T2，这个T2应该近似于timer的周期T1，测试结果如下：\
\+ hrtimer 10 us toggle pin1引脚，pin2 中断间500000 次间隔超出周期 10 us +/- 5 us的 次数有  20-49/499977-499997 time range(4us-306us)\
\
\+ hrtimer 500 us toggle pin1引脚，pin2 中断 10000 次间隔超出周期 500 us +/- 5us的 次数有  5~8/9996~10000 time range(486 us-1006 us)\
\
\+ hrtimer 100 us toggle pin1引脚，pin2 中断 50000 次间隔超出周期 100 us +/- 5 us的 次数有  7~20/49990 time range(6 us- 619 us)\
\
\+ hrtimer 50 us toggle pin1引脚，pin2 中断 100000 次间隔超出周期 50 us +/- 5 us的 次数有  14/(100000) times  34-67 us\
\
\+ hrtimer 1000 us toggle pin1引脚，pin2 中断 5000 间隔超出周期 1000 us +/- 5 us的 次数有  5-16/5000 time range(489-1006)\
\
【结论】虽然有丢中断，但是总体来说中断延时还是很小的，us基本的，基于以上两个测试，我觉得linux的gpio中断延时还是比想象中的小，于是我打算用gpio来进行IR的解码，于是有了测试3

测试3：利用一个pin接到ir信号输出，并用单边沿触发，测试脉宽周期T（IR的T的主要有1.15ms/2.25ms/9ms/13ms等，最小的都大于1ms，基于上面测试我认为正确捕获时间应该没问题）， 但是结果如下：\
\+ 每隔3s发送一个IR信号（每隔IR信号包含33个周期的IO中断），IO捕获的T打印出来，每次捕获的T都有凸点，比如T=9us\
\+ 每隔1s发送一个IR信号，大概30%概率都有凸点\
\+ 每隔40ms发送一个IR信号，就第一个IR信号有异常的凸点，从第二个IR信号开始都是正常的T\
\+ 每隔20ms发送一个IR信号，就第一个IR信号有异常的凸点，从第二个IR信号开始都是正常的T\
【结论】中断时间越长，linux越难准确捕获到，现象类似于需要一个激活状态，前面几个不准确的中断导致后面的中断都是准确的，难道底层有runtime的中断处理优化? 另外也怀疑不是睡眠以及dvfs之类的，但是好像没有类似事件发生\
，

附录具有凸点的T数据，这个在一个单独线程中打印的，中断中把时间存好：\
\[   96.691005\] 95600663000,1012137457, d=1114\
\[   96.695204\] 95602281375,1012150404, d=1618   #此处捕获的实际时间比理论时间延迟了500ms，从此处开始后面的中断都延迟了500ms，所以后面T都是正确的\
\[   96.699311\] 95604537875,1012168456, d=2256\
\[   96.703422\] 95605666625,1012177486, d=1128

\[   96.707530\] 95606765250,1012186274, d=1098\
\[   96.711640\] 95607894375,1012195308, d=1129\
\[   96.715748\] 95609023625,1012204342, d=1129\
\[   96.719858\] 95611251125,1012222162, d=2227

\[   96.723966\] 95612381500,1012231204, d=1130\
\[   96.728075\] 95612993000,1012236097, d=611    #到此处的时候，中断没有延时，而是立即就去处理了，导致此时的时间等于理论时间，从此之后中断都是正常响应\
\[   96.732184\] 95615240000,1012254073, d=2247\
\[   96.736204\] 95616358750,1012263023, d=1118

\[   96.740313\] 95619094000,1012284904, d=2735\
\[   96.744420\] 95620848875,1012298944, d=1754\
\[   96.748542\] 95623093000,1012316897, d=2244\
\[   96.752652\] 95624213625,1012325862, d=1120

\[   96.756758\] 95626457875,1012343817, d=2244\
\[   96.760867\] 95628703625,1012361782, d=2245\
\[   96.764973\] 95630947750,1012379735, d=2244\
\[   96.769094\] 95633193500,1012397701, d=2245\
\[   96.773204\] 95635438000,1012415657, d=2244\
\[   96.777314\] 95637683625,1012433622, d=2245\
\[   96.781433\] 95639927875,1012451577, d=2244\
\[   96.785545\] 95642172875,1012469537, d=2245

\[   96.789653\] 95643292750,1012478495, d=1119\
\[   96.793775\] 95644413625,1012487462, d=1120\
\[   96.797883\] 95645532875,1012496416, d=1119\
\[   96.801991\] 95646652750,1012505375, d=1119\
\[   96.806106\] 95647772750,1012514335, d=1120\
\[   96.810216\] 95648892875,1012523297, d=1120\
\[   96.814323\] 95650012875,1012532256, d=1120\
\[   96.818444\] 95651132875,1012541216, d=1120

【IR解码结果】间隔时间按一个按键，解码会失败；但是你迅速的不断按下IR按键，解码是成功的，跟上面的数据吻合，但是始终不明白为何长时间间隔的中断会捕获不准，或者说中断latency很长

[回复](http://www.wowotech.net/irq_subsystem/request_threaded_irq.html#comment-7555)

**[bsp](http://www.wowotech.net/)**\
2019-07-30 11:24

@chili：另外也怀疑不是睡眠以及dvfs之类的，但是好像没有类似事件发生 ?\
还是检查下吧，很像睡眠引起的。\
你可以让系统busy起来，比如用户态 运行个while(1)，然后再测试你的程序。

[回复](http://www.wowotech.net/irq_subsystem/request_threaded_irq.html#comment-7556)

**chili**\
2019-08-01 14:10

@bsp：确实啊，4\*A53的ARM core，执行下面两个while后，其中2个core的cpu使用率为100%，这种情况下IR按键的识别率可以达到100%，执行一个while还是会存在误识别；\
while true;do : ;done  &\
while true;do : ;done  &

\*\*那么问题又来了，怎么避免不睡眠呢？IRQF_NO_SUSPEND这些中断申请的flag都试了不行，1ms的间隔都会睡眠，有办法规避或者查询到何时睡眠的吗？

[回复](http://www.wowotech.net/irq_subsystem/request_threaded_irq.html#comment-7564)

**chili**\
2019-08-01 15:27

@chili：OK,初步研究了下，是因为cpuidle导致，执行下面命令禁止cpuidle,就可正常快速的响应中断了\
echo 1 > /sys/devices/system/cpu/cpu0/cpuidle/state1/disable

[回复](http://www.wowotech.net/irq_subsystem/request_threaded_irq.html#comment-7565)

**floater**\
2017-12-02 14:21

请问下关于io expander这种情况相当于soc扩展了一个gpio controller，那它的中断申请中io expander上的中断号是怎么分配的呢？

[回复](http://www.wowotech.net/irq_subsystem/request_threaded_irq.html#comment-6279)

**mouse**\
2017-08-03 10:39

threaded irq和工作队列(work queue)都是在进程上下文中处理中断下半部。什么时候选工作队列，什么时候选threaded irq? 谢谢。

[回复](http://www.wowotech.net/irq_subsystem/request_threaded_irq.html#comment-5869)

**[linuxer](http://www.wowotech.net/)**\
2017-08-03 15:03

@mouse：要不要加入蜗窝微信群，和大家一起讨论这些问题？

[回复](http://www.wowotech.net/irq_subsystem/request_threaded_irq.html#comment-5870)

**[linuxer](http://www.wowotech.net/)**\
2017-08-08 08:46

@mouse：大家在微信群里讨论了这个问题，具体可以参考http://www.wowotech.net/forum/viewtopic.php?pid=795#p795

[回复](http://www.wowotech.net/irq_subsystem/request_threaded_irq.html#comment-5885)

**[heziq](http://www.wowotech.net/)**\
2017-06-29 17:25

@linuxer：\
handle_edge_irq 在这个 handler中，我没有找到怎么保证IRQF_ONESHOT这个实现。如果中断有两次，就有可能wake thread两次，thread会执行两次吗？ wake up thread函数，只是会把它状态改成running。所以，我觉得会执行一次。

[回复](http://www.wowotech.net/irq_subsystem/request_threaded_irq.html#comment-5760)

**lydfl**\
2017-06-12 19:59

Linuxer,\
您好，请问nested IRQ主要应用在什么场景？如果说为了节省中断引脚，那在父中断的处理流程里面用switch case不就可以了吗？\
在内核全局搜索了一下，这个nested irq好像使用的也不多。。。

[回复](http://www.wowotech.net/irq_subsystem/request_threaded_irq.html#comment-5663)

**maze**\
2016-08-31 17:13

我在等这个todo

[回复](http://www.wowotech.net/irq_subsystem/request_threaded_irq.html#comment-4482)

**[linuxer](http://www.wowotech.net/)**\
2016-09-01 09:21

@maze：最近在搞内存管理，估计需要等一段时间了，抱歉！！！

[回复](http://www.wowotech.net/irq_subsystem/request_threaded_irq.html#comment-4483)

1 [2](http://www.wowotech.net/irq_subsystem/request_threaded_irq.html/comment-page-2#comments) [3](http://www.wowotech.net/irq_subsystem/request_threaded_irq.html/comment-page-3#comments)

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

  - [内存初始化代码分析（一）：identity mapping和kernel image mapping](http://www.wowotech.net/memory_management/__create_page_tables_code_analysis.html)
  - [u-boot启动流程分析(1)\_平台相关部分](http://www.wowotech.net/u-boot/boot_flow_1.html)
  - [systemd：为何要创建一个新的init系统软件](http://www.wowotech.net/linux_application/why-systemd.html)
  - [X-000-PRE-开发环境搭建](http://www.wowotech.net/x_project/develop_env.html)
  - [CPU 多核指令 —— WFE 原理](http://www.wowotech.net/armv8a_arch/499.html)

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
