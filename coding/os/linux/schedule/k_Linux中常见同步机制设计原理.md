作者：[smcdef](http://www.wowotech.net/author/531) 发布于：2018-6-9 16:19 分类：[内核同步机制](http://www.wowotech.net/sort/kernel_synchronization)

## 引言

今天谈谈linux中常见并发访问的保护机制设计原理。为什么要写这篇文章呢？其实想帮助自己及读者更深入的了解背后的原理（据可靠消息，锁的实现经常出现在笔试环节。既可以考察面试者对锁的原理的理解，又可以考察面试者编程技能）。我们抛开linux中汇编代码。用C语言为大家呈现背后实现的原理。同时，文章中的代码都没有考虑并发情况（例如某些操作需要原子性，或者数据需要保护等）。

> 注：部分代码都是根据ARM64架构汇编代码翻译成C语言并经过精简（例如：spin lock、read-write lock）。也有部分代码实现是为了呈现背后设计的原理自己编写的，而不是精简linux中实现的代码（例如mutex）。

## 自旋锁（spin lock）

自旋锁是linux中使用非常频繁的锁，原理简单。当进程A申请锁成功后，进程B申请锁就会失败，但是不会调度，原地自旋。就在原地转到天昏地暗只为等到进程A释放锁。由于不会睡眠和调度的特性，在中断上下文中，数据的保护一般都是选择自旋锁。如果有多个进程去申请锁。当第一个申请锁成功的线程在释放的时候，其他进程是竞争的关系。因此是一种不公平。所以现在的linux采用的是排队机制。先到先得。谁先申请，谁就先得到锁。

### 原理

举个例子，大家应该都去过银行办业务吧。银行的办事大厅一般会有几个窗口同步进行。今天很不巧，只有一个窗口提供服务。现在的银行服务都是采用取号排队，叫号服务的方式。当你去银行办理业务的时候，首先会去取号机器领取小票，上面写着你排多少号。然后你就可以排队等待了。一般还会有个显示屏，上面会显示一个数字（例如："请xxx号到1号窗口办理"），代表当前可以被服务顾客的排队号码。每办理完一个顾客的业务，显示屏上面的数字都会增加1。等待的顾客都会对比自己手上写的编号和显示屏上面是否一致，如果一致的话，就可以去前台办理业务了。现在早上刚开业，顾客A是今天的第一个顾客，去取号机器领取0号（next计数）小票，然后看到显示屏上显示0（owner计数），顾客A就知道现在轮到自己办理业务了。顾客A到前台办理业务（持有锁）中，顾客B来了。同样，顾客B去取号机器拿到1号（next计数）小票。然后乖乖的坐在旁边等候。顾客A依然在办理业务中，此时顾客C也来了。顾客C去取号机器拿到2号（next计数）小票。顾客C也乖乖的找个座位继续等待。终于，顾客A的业务办完了（释放锁）。然后，显示屏上面显示1（owner计数）。顾客B和C都对比显示屏上面的数字和自己手中小票的数字是否相等。顾客B终于可以办理业务了（持有锁）。顾客C依然等待中。顾客B的业务办完了（释放锁）。然后，显示屏上面显示2（owner计数）。顾客C终于开始办理业务（持有锁）。顾客C的业务办完了（释放锁）。3个顾客都办完了业务离开了。只留下一个银行柜台服务员。最终，显示屏上面显示3（owner计数）。取号机器的下一个排队号也是3号（next计数）。无人办理业务（锁是释放状态）。

!\[\[Pasted image 20241008185413.png\]\]

linux中针对每一个spin lock会有两个计数。分别是next和owner（初始值为0）。进程A申请锁时，会判断next和owner的值是否相等。如果相等就代表锁可以申请成功，否则原地自旋。直到owner和next的值相等才会退出自旋。假设进程A申请锁成功，然后会next加1。此时owner值为0，next值为1。进程B也申请锁，保存next得值到局部变量tmp（tmp = 1）中。由于next和owner值不相等，因此原地自旋读取owner的值，判断owner和tmp是否相等，直到相等退出自旋状态。当然next的值还是加1，变成2。进程A释放锁，此时会将owner的值加1，那么此时B进程的owner和tmp的值都是1，因此B进程获得锁。当B进程释放锁后，同样会将owner的值加1。最后owner和next都等于2，代表没有进程持有锁。next就是一个记录申请锁的次数，而owner是持有锁进程的计数值。

### 实现

我们首先定义描述自旋锁的结构体arch_spinlock_t。

```cpp
1. typedef struct {
2. 	union {
3. 		unsigned int slock;
4. 		struct __raw_tickets {
5. 			unsigned short owner;
6. 			unsigned short next;
7. 		} tickets;
8. 	};
9. } arch_spinlock_t; 
```

如上面的原理描述，我们需要两个计数，分别是owner和next。slock所占内存区域覆盖owner和next（据说C语言学好的都能看得懂）。下面实现申请锁操作 arch_spin_lock。

```cpp
1. static inline void arch_spin_lock(arch_spinlock_t *lock)
2. {
3. 	arch_spinlock_t old_lock;

5. 	old_lock.slock = lock->slock;                             /* 1 */
6. 	lock->tickets.next++;                                     /* 2 */
7. 	while (old_lock.tickets.next != old_lock.tickets.owner) { /* 3 */
8. 		wfe();                                                /* 4 */
9. 		old_lock.tickets.owner = lock->tickets.owner;         /* 5 */
10. 	}
11. } 
```

> 1. 继续上面的举例。顾客从取号机器得到排队号。
> 1. 取号机器更新下个顾客将要拿到的排队号。
> 1. 看一下显示屏，判断是否轮到自己了。
> 1. wfe()函数是指ARM64架构的WFE(wait for event)汇编指令。WFE是让ARM核进入低功耗模式的指令。当进程拿不到锁的时候，原地自旋不如cpu睡眠。节能。睡下去之后，什么时候醒来呢？就是等到持有锁的进程释放的时候，醒过来判断是否可以持有锁。如果不能获得锁，继续睡眠即可。这里就相当于顾客先小憩一会，等到广播下一位排队者的时候，醒来看看是不是自己。
> 1. 前台已经为上一个顾客办理完成业务，剩下排队的顾客都要抬头看一下显示屏是不是轮到自己了。

释放锁的操作就非常简单了。还记得上面银行办理业务的例子吗？释放锁的操作仅仅是显示屏上面的排队号加1。我们仅仅需要将owner计数加1即可。arch_spin_unlock实现如下。

```cpp
1. static inline void arch_spin_unlock(arch_spinlock_t *lock)
2. {
3. 	lock->tickets.owner++;
4. 	sev();
5. } 
```

> sev()函数是指ARM64架构的SEV汇编指令。当进程无法获取锁的时候会使用WFE指令使CPU睡眠。现在释放锁了，自然要唤醒所有睡眠的CPU醒来检查自己是不是可以获取锁。

## 信号量（semaphore）

信号量（semaphore）是进程间通信处理同步互斥的机制。是在多线程环境下使用的一种措施，它负责协调各个进程，以保证他们能够正确、合理的使用公共资源。 它和spin lock最大的不同之处就是：无法获取信号量的进程可以睡眠，因此会导致系统调度。

### 原理

信号量一般可以用来标记可用资源的个数。老规矩，还是举个例子。假设图书馆有2本《C语言从入门到放弃》书籍。A同学想学C语言，于是发现这本书特别的好。于是就去学校的图书馆借书，A同学成功的从图书馆借走一本。这时，A同学室友B同学发现A同学竟然在偷偷的学习武功秘籍（C语言）。于是，B同学也去借一本。此时，图书馆已经没有书了。C同学也想借这本书，可能是这本书太火了。图书馆管理员告诉C同学，图书馆这本书都被借走了。如果有同学换回来，会第一时间通知你。于是，管理员就把C同学的信息登记先来，以备后续通知C同学来借书。所以，C同学只能悲伤的走了（如果是自旋锁的原理的话，那么C同学将会端个小板凳坐在图书馆，一直要等到A同学或者B同学还书并借走）。

### 实现

为了记录可用资源的数量，我们肯定需要一个count计数，标记当前可用资源数量。当然还要一个可以像图书管理员一样的笔记本功能。用来记录等待借书的同学。所以，一个双向链表即可。因此只需要一个count计数和等待进程的链表头即可。描述信号量的结构体如下。

```cpp
1. struct semaphore {
2. 	unsigned int count;
3. 	struct list_head wait_list;
4. }; 
```

在linux中，每个进程就相当于是每个借书的同学。通知一个同学，就相当于唤醒这个进程。因此，我们还需要一个结构体记录当前的进程信息（task_struct）。

```cpp
1. struct semaphore_waiter {
2. 	struct list_head list;
3. 	struct task_struct *task;
4. }; 
```

struct semaphore_waiter的list成员是当进程无法获取信号量的时候挂入semaphore的wait_list成员。task成员就是记录后续被唤醒的进程信息。

一切准备就绪，现在就可以实现信号量的申请函数。

```cpp
1. void down(struct semaphore *sem)
2. {
3. 	struct semaphore_waiter waiter;

5. 	if (sem->count > 0) {
6. 		sem->count--;                               /* 1 */
7. 		return;
8. 	}

10. 	waiter.task = current;                          /* 2 */
11. 	list_add_tail(&waiter.list, &sem->wait_list);   /* 2 */
12. 	schedule();                                     /* 3 */
13. } 
```

> 1. 如果信号量标记的资源还有剩余，自然可以成功获取信号量。只需要递减可用资源计数。
> 1. 既然无法获取信号量，就需要将当前进程挂入信号量的等待队列链表上。
> 1. schedule()主要是触发任务调度的示意函数，主动让出CPU使用权。在让出之前，需要将当前进程从运行队列上移除。

释放信号的实现也是比较简单。实现如下。

```cpp
1. void up(struct semaphore *sem)
2. {
3. 	struct semaphore_waiter waiter;

5. 	if (list_empty(&sem->wait_list)) {
6. 		sem->count++;                              /* 1 */
7. 		return;
8. 	}

10. 	waiter = list_first_entry(&sem->wait_list, struct semaphore_waiter, list);
11. 	list_del(&waiter->list);                       /* 2 */
12. 	wake_up_process(waiter->task);                 /* 2 */
13. } 
```

> 1. 如果等待链表没有进程，那么自然只需要增加资源计数。
> 1. 从等待进程链表头取出第一个进程，并从链表上移除。然后就是唤醒该进程。

## 读写锁（read-write lock）

不管是自旋锁还是信号量在同一时间只能有一个进程进入临界区。对于有些情况，我们是可以区分读写操作的。因此，我们希望对于读操作的进程可以并发进行。对于写操作只限于一个进程进入临界区。而这种同步机制就是读写锁。读写锁一般具有以下几种性质。

- 同一时间有且仅有一个写进程进入临界区。
- 在没有写进程进入临界区的时候，同时可以有多个读进程进入临界区。
- 读进程和写进程不可以同时进入临界区。

读写锁有两种，一种是信号量类型，另一种是spin lock类型。下面以spin lock类型讲解。

### 原理

老规矩，还是举个例子理解读写锁。我绞尽脑汁才想到一个比较贴切的例子。这个例子来源于生活。我发现公司一般都会有保洁阿姨打扫厕所。如果以男厕所为例的话，我觉得男士进入厕所就相当于读者进入临界区。因为可以有多个男士进厕所。而保洁阿姨进入男士厕所就相当于写者进入临界区。假设A男士发现保洁阿姨不在打扫厕所，就进入厕所。随后B和C同时也进入厕所。然后保洁阿姨准备打扫厕所，发现有男士在厕所里面，因此只能在门口等待。ABC都离开了厕所。保洁阿姨迅速进入厕所打扫。然后D男士去上厕所，发现保洁阿姨在里面。灰溜溜的出来了在门口等着。现在体会到了写者（保洁阿姨）具有排他性，读者（男士）可以并发进入临界区了吧。

既然我们允许多个读者进入临界区，因此我们需要一个计数统计读者的个数。同时，由于写者永远只存在一个进入临界区，因此只需要一个bit标记是否有写进程进入临界区。所以，我们可以将两个计数合二为一。只需要1个unsigned int类型即可。最高位（bit31）代表是否有写者进入临界区，低31位（0~30bit）统计读者个数。

```c
 
+----+-------------------------------------------------+| 31 | 30                                            0 |+----+-------------------------------------------------+  |                    |  |                    +----> [0:30] Read Thread Counter  +-------------------------> [31]  Write Thread Counter
```

### 实现

描述读写锁只需要1个变量即可，因此我们可以定义读写锁的结构体如下。

```cpp
1. typedef struct {
2. 	volatile unsigned int lock;
3. } arch_rwlock_t; 
```

既然区分读写操作，因此肯定会有两个申请锁函数，分别是读和写。首先，我们看一下read_lock操作的实现。

```cpp
1. static inline void arch_read_lock(arch_rwlock_t *rw)
2. {
3. 	unsigned int tmp;

5. 	sevl();                                   /* 1 */
6. 	do {
7. 		wfe();
8. 		tmp = rw->lock;
9. 		tmp++;                                /* 2 */
10. 	} while(tmp & (1 << 31));                 /* 3 */
11. 	rw->lock = tmp;
12. } 
```

> 1. sevl()函数是ARM64架构中SEVL汇编指令。SEVL和SEV的区别是，SEVL仅仅修改本地CPU的PE寄存器值，这样下面的WFE指令第一次执行的时候不会睡眠。
> 1. 增加读者计数，最后会更新到rw->lock中。
> 1. 更新rw->lock前提是没有写者，因此这里会判断是否有写者已经进入临界区（判断方法是rw->lock变量bit31的值）。如果，有写者已经进入临界区，就在这里循环，并WFE指令睡眠。类似上面介绍的spin lock实现。

当读进程离开临界区的时候会调用read_unlock释放锁。read_unlock实现如下。

```cpp
1. static inline void arch_read_unlock(arch_rwlock_t *rw)
2. {
3. 	rw->lock--;
4. 	sev();
5. } 
```

> 实现很简单，和spin_unlock如出一辙。递减读者计数，然后使用SEV指令唤醒所有的CPU，检查等待状态的进程是否可以获取锁。

读操作看完了，我们看看写操作是如何实现的。arch_write_lock实现如下。

```cpp
1. static inline void arch_write_lock(arch_rwlock_t *rw)
2. {
3. 	unsigned int tmp;

5. 	sevl();
6. 	do {
7. 		wfe();
8. 		tmp = rw->lock;
9. 	} while(tmp);                       /* 1 */
10. 	rw->lock = 1 << 31;                 /* 2 */
11. } 
```

> 1. 由于写者是排他的（读者和写者都不能有），因此这里只有rw->lock的值为0，当前的写者才可以进入临界区。
> 1. 置位rw->lock的bit31，代表有写者进入临界区。

当写进程离开临界区的时候会调用write_unlock释放锁。write_unlock实现如下。

```cpp
1. static inline void arch_write_unlock(arch_rwlock_t *rw)
2. {
3. 	rw->lock = 0;                     /* 1 */
4. 	sev();                            /* 2 */
5. } 
```

> 1. 同样由于写者是排他的，因此只需要将rw->lock置0即可。代表没有任何进程进入临界区。毕竟是因为同一时间只能有一个写者进入临界区，当这个写者离开临界区的时候，肯定是意味着现在没有任何进程进入临界区。
> 1. 使用SEV指令唤醒所有的CPU，检查等待状态的进程是否可以获取锁。

以上的代码实现其实会导致写进程饿死现象。例如，A、B、C三个进程进入读临界区，D进程尝试获得写锁，此时只能等待A、B、C三个进程退出临界区。如果在退出之前又有F、G进程进入读临界区，那么将出现D进程饿死现象。

## 互斥量（mutex）

前文提到的semaphore在初始化count计数的时候，可以分为计数信号量和互斥信号量（二值信号量）。mutex和初始化计数为1的二值信号量有很大的相似之处。他们都可以用做资源互斥。但是mutex却有一个特殊的地方：只有持锁者才能解锁。但是，二值信号量却可以在一个进程中获取信号量，在另一个进程中释放信号量。如果是应用在嵌入式应用的RTOS，针对mutex的实现还会考虑优先级反转问题。

### 原理

既然mutex是一种二值信号量，因此就不需要像semaphore那样需要一个count计数。由于mutex具有“持锁者才能解锁”的特点，所以我们需要一个变量owner记录持锁进程。释放锁的时候必须是同一个进程才能释放。当然也需要一个链表头，主要用来便利睡眠等待的进程。原理和semaphore及其相似，因此在代码上也有体现。

### 实现

mutex的实现代码和linux中实现会有差异，但是依然可以为你呈现设计的原理。下面的设计代码更像是部分RTOS中的代码。mutex和semaphore一样，我们需要两个类似的结构体分别描述mutex。

```cpp
1. struct mutex_waiter {
2. 	struct list_head   list;
3. 	struct task_struct *task;
4. };

6. struct mutex {
7.     long   owner;
8.     struct list_head wait_list;
9. }; 
```

struct mutex_waiter的list成员是当进程无法获取互斥量的时候挂入mutex的wait_list链表。

首先实现申请互斥量的函数。

```cpp
1. void mutex_take(struct mutex *mutex)
2. {
3. 	struct mutex_waiter waiter;

5. 	if (!mutex->owner) {
6. 		mutex->owner = (long)current;               /* 1 */
7. 		return;
8. 	}

10. 	waiter.task = current;
11. 	list_add_tail(&waiter.list, &mutex->wait_list); /* 2 */
12. 	schedule();                                     /* 2 */
13. }  
```

> 1. 当mutex->owner的值为0的时候，代表没有任何进程持有锁。因此可以直接申请成功。然后，记录当前申请锁进程的task_struct。
> 1. 既然不能获取互斥量，自然就需要睡眠等待，挂入等待链表。

互斥量的释放代码实现也同样和semaphore有很多相似之处。不信，你看。

```c
 
int mutex_release(struct mutex *mutex){	struct mutex_waiter *waiter; 	if (mutex->owner != (long)current)                         /* 1 */		return -1; 	if (list_empty(&mutex->wait_list)) {		mutex->owner = 0;                                      /* 2 */		return 0;	} 	waiter = list_first_entry(&mutex->wait_list, struct mutex_waiter, list);	list_del(&waiter->list);	mutex->owner = (long)waiter->task;                         /* 3 */	wake_up_process(waiter->task);                             /* 4 */ 	return 0;}
```

> 1. mutex具有“持锁者才能解锁”的特点就是在这行代码体现。
> 1. 如果等待链表没有进程，那么自然只需要将mutex->owner置0，代表没有锁是释放状态。
> 1. mutex->owner的值改成当前可以持锁进程的task_struct。
> 1. 从等待进程链表取出第一个进程，并从链表上移除。然后就是唤醒该进程。

标签: [spin](http://www.wowotech.net/tag/spin) [lock](http://www.wowotech.net/tag/lock) [mutex](http://www.wowotech.net/tag/mutex) [rw_lock](http://www.wowotech.net/tag/rw_lock)

______________________________________________________________________

« [per-entity load tracking](http://www.wowotech.net/process_management/PELT.html) | [KASLR](http://www.wowotech.net/memory_management/441.html)»

**评论：**

**wangxingxing**\
2020-07-30 10:33

void \_\_sched mutex_lock(struct mutex \*lock)\
{\
might_sleep();

if (!\_\_mutex_trylock_fast(lock))\
\_\_mutex_lock_slowpath(lock);\
}\
调用\_\_mutex_lock_slowpath申请锁之前会调用\_\_mutex_trylock_fast检查所是否可以被快速获取，能否解释一下\_\_mutex_trylock_fast的实现原理，谢谢。

[回复](http://www.wowotech.net/kernel_synchronization/445.html#comment-8074)

**crash**\
2019-08-20 09:37

spin_lock获取不到锁资源spin等待期间会被中断切走吗

[回复](http://www.wowotech.net/kernel_synchronization/445.html#comment-7598)

**[smcdef](http://www.wowotech.net/)**\
2019-08-21 19:55

@crash：spin lock 会关闭抢占，所以当前进程不会被切换。但是中断是打开的，所以可以响应中断，中断退出后继续返回这里 spin。

[回复](http://www.wowotech.net/kernel_synchronization/445.html#comment-7607)

**crash**\
2019-08-22 08:25

@smcdef：那spin期间被中断切走了，借助上面形象的例子，下一个等待的顾客来了个电话中断了，正好这时候窗口的业务办好释放了锁，那排在后面的也得等他？

[回复](http://www.wowotech.net/kernel_synchronization/445.html#comment-7610)

**[smcdef](http://www.wowotech.net/)**\
2020-01-04 18:08

@crash：那是自然

[回复](http://www.wowotech.net/kernel_synchronization/445.html#comment-7814)

**crash**\
2018-11-14 09:11

wowo,初学者有两点疑问：\
1、spinlock对自身ticket,owner变量是不是也遵循read/modify/write的规则，具体还是通过ldrex\\strex实现？\
2、semaphore的down方法实现中用了raw_spin_lock_irqsave来保护count，为什么要加上禁中断，semaphore不会用在中断的啊

[回复](http://www.wowotech.net/kernel_synchronization/445.html#comment-7030)

**[linuxer](http://www.wowotech.net/)**\
2018-11-14 09:27

@crash：1、当然，对（ticket，owner）的访问也需要时原子操作，是不是ldrex/strex需要进一步看底层实现，ARMV8之前是ldrex/strex，ARMV8.1之后，ARM平台提供了原子指令，底层也许可以有不同的实现，但是我还没有时间跟踪\
2、中断上下文可以调用down_trylock

[回复](http://www.wowotech.net/kernel_synchronization/445.html#comment-7031)

**crash**\
2018-11-14 09:51

@linuxer：回复神速，感谢。\
问个题外话，armv8这些ldrex/strex指令，包括您提到的V8.1原子指令，这些arm资讯你们是从哪个网站获取或是订阅了什么推送的吗?

[回复](http://www.wowotech.net/kernel_synchronization/445.html#comment-7032)

**linux爱好者**\
2018-09-10 12:31

弱弱的问下，struct semaphore_waiter waiter;这样写在函数体内，函数down结束后生命周期结束，会不会有问题？

[回复](http://www.wowotech.net/kernel_synchronization/445.html#comment-6940)

**[smcdef](http://www.wowotech.net/)**\
2018-09-10 18:02

@linux爱好者：不会有问题，如果无法获取semaphore ，down函数根本没有返回，而是调用schedule()函数让出cpu使用权了。等到down函数结束生命周期后，其实已经获取semaphore 了。

[回复](http://www.wowotech.net/kernel_synchronization/445.html#comment-6943)

**1**\
2018-09-04 16:39

大佬 写写顺序锁和RCU...

[回复](http://www.wowotech.net/kernel_synchronization/445.html#comment-6929)

**aa**\
2018-09-04 14:29

好好好!看湿了

[回复](http://www.wowotech.net/kernel_synchronization/445.html#comment-6928)

**hai**\
2018-08-21 15:40

大神，看到你说，mutex和二值信号量的区别在于mutex必须是同一个进程来释放，文章里也是很清楚的通过对比ower和current表现出来，其实网上也有很多这种说法，但是我具体去看代码的时候找了很久，也没有找到哪里有对比ower和current的地方，你帖的代码应该是你总结出来的吧，请问你这里是在哪里看到的内核代码从而总结出来的？

[回复](http://www.wowotech.net/kernel_synchronization/445.html#comment-6904)

**[smcdef](http://www.wowotech.net/)**\
2018-08-22 13:50

@hai：文章开头说明了mutex部分是我自己理解写出的，给你造成的困惑，实在抱歉。如果你想看些简单的实现可以参考RTOS的实现，例如rt-thread。如果你想探究linux的实现，我看了一下mutex_unlock函数（linux-4.15）。在使能CONFIG_DEBUG_MUTEXES之后，会有下面一句代码：\
DEBUG_LOCKS_WARN_ON(\_\_owner_task(owner) != current);\
说明linux认为这是一种不正常的行为。

[回复](http://www.wowotech.net/kernel_synchronization/445.html#comment-6906)

**zero**\
2019-05-14 21:09

@hai：哪些情况下需要一个线程加down，另外一个线程up？

[回复](http://www.wowotech.net/kernel_synchronization/445.html#comment-7419)

**lcm**\
2018-08-14 17:41

大神，看过你以前专门写spinlock文章。最新的内核的spin_lock机制有改变吗？非SMP原来的机制都是1.如果临界区只是在进程里，那么就是spin_lock,即关闭cpu抢占。2.如果是中断中含有临界区，就需要关中断。现在的机制看是加入了排队，取号步骤，如果取不到号让cpu进入低功耗自旋模式。现在这种机制是不是只是针对SMP的呢？

[回复](http://www.wowotech.net/kernel_synchronization/445.html#comment-6885)

**[smcdef](http://www.wowotech.net/)**\
2018-08-14 22:13

@lcm：文章写的就是针对linux4.13代码。针对UP系统，spin_lock只需要preempt_disable即可。SMP系统除了preempt_disable之外，还需要申请锁。

[回复](http://www.wowotech.net/kernel_synchronization/445.html#comment-6886)

**bingchuan**\
2018-08-09 17:23

read_write lock的write进程被饿死常见吗？感觉read_write lock和RCU的原理是一样的。另外，这节的最后一句话有个错别字，推->退，强迫症，没办法-v-

[回复](http://www.wowotech.net/kernel_synchronization/445.html#comment-6876)

**[smcdef](http://www.wowotech.net/)**\
2018-08-09 22:55

@bingchuan：感谢指出，错别字已经修改。\
read_write lock的write进程被饿死常见吗？\
其实会不会饿死，取决于你的应用场景。及其频繁read穿插write的话，是可能饿死的。保洁阿姨在厕所门口站着，陆陆续续有人进厕所，你说这位阿姨是不是一直在等待呢？

[回复](http://www.wowotech.net/kernel_synchronization/445.html#comment-6879)

**hai**\
2018-08-03 10:15

请问一下，信号量的count值代表着多少个临界区资源，只要count不为0，进程来了就可以访问，这是不是意味着有多个临界区，因为临界区一次只能一个进程或线程访问

[回复](http://www.wowotech.net/kernel_synchronization/445.html#comment-6864)

**[smcdef](http://www.wowotech.net/)**\
2018-08-04 13:01

@hai：你说说的情况应该是信号量的count为1（二进制信号量）的情况，也就是只有一个临界区。但是信号量的count同样可以用来表示资源的数量。例如，银行办事大厅有5个服务台，那么这个count就可以是5，同时可以有5个顾客办理业务。第6个人就只能等待。

[回复](http://www.wowotech.net/kernel_synchronization/445.html#comment-6867)

1 [2](http://www.wowotech.net/kernel_synchronization/445.html/comment-page-2#comments)

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

  - [我的bash shell学习笔记](http://www.wowotech.net/linux_application/134.html)
  - [ARMv8-a架构简介](http://www.wowotech.net/armv8a_arch/armv8-a_overview.html)
  - [Linux电源管理(15)\_PM OPP Interface](http://www.wowotech.net/pm_subsystem/pm_opp.html)
  - [perfbook memory barrier（14.2章节）的中文翻译（上）](http://www.wowotech.net/kernel_synchronization/memory-barrier-1.html)
  - [Atomic operation in aarch64](http://www.wowotech.net/armv8a_arch/492.html)

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
