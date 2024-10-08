作者：[OPPO内核团队](http://www.wowotech.net/author/538) 发布于：2022-5-10 5:55 分类：[内核同步机制](http://www.wowotech.net/sort/kernel_synchronization)

一、Mutex锁简介

在linux内核中，互斥量（mutex，即mutual exclusion）是一种保证串行化的睡眠锁机制。和spinlock的语义类似，都是允许一个执行线索进入临界区，不同的是当无法获得锁的时候，spinlock原地自旋，而mutex则是选择挂起当前线程，进入阻塞状态。正因为如此，mutex无法在中断上下文使用。和mutex更类似的机制（无法获得锁时都会阻塞）是binary semaphores，当然，mutex有更严格的使用规则：

1、只有mutex的owner可以才可以释放锁

2、不可以多次释放同一把锁

3、不允许重复获取同一把锁，否则会死锁

4、必须使用mutex初始化API来完成锁的初始化，不能使用类似memset或者memcp之类的函数进行mutex初始化

5、不可以多次重复对mutex锁进行初始化

6、线程退出后必须释放自己持有的所有mutex锁

当配置了DEBUG_MUTEXES的时候，内核会对上面的规则进行检查，防止用户误用mutex，产生各种问题。

下面是一个简单的mutex工作原理图：

![](http://www.wowotech.net/content/uploadfile/202205/b6c31652133568.png)

传统的mutex只需要一个状态标记和一个等待队列就OK了，等待队列中是一个个阻塞的线程，thread owner当前持有mutex，当它离开临界区释放锁的时候，会唤醒等待队列中第一个线程（top waiter），这时候top waiter会去竞争持锁，如果成功，那么从等待队列中摘下，成为owner。如果失败，继续保持阻塞状态，等待owner释放锁的时候唤醒它。在owner task持锁过程中，如果有新的任务来竞争mutex，那么就会进入阻塞状态并插入等待队列的尾部。

相对于传统的mutex，linux内核进行了一些乐观自旋的优化，也就是说当线程持锁失败的时候，可以选择在mutex状态标记上自旋，等待owner释放锁，也可以选择进入阻塞状态并挂入等待队列。具体如何选择是在自旋等待的时间开销和进程上下文切换的开销之间进行平衡。此外为了防止多个线程自旋带来的性能问题，mutex的乐观自旋机制还引入了MCS锁，后面章节我们会详细描述。

二、数据结构

1、互斥量对象

互斥量对象用struct mutex来抽象，其成员描述如下：

|   |   |
|---|---|
|成员|描述|
|atomic_long_t   owner|这个成员有两个作用：<br><br>1、记录该mutex对象被哪一个task持有（struct task_struct *）。如果等于NULL表示还没有被任何一个任务持有。<br><br>2、由于task struct地址是L1_CACHE_BYTES对齐的，因此这个成员的有若干的LSB可以被用于标记状态（在ARM64平台上，L1_CACHE_BYTES是64字节，因此LSB 6-bit会用于保存mutex状态信息）。具体的状态包括：<br><br>MUTEX_FLAG_WAITERS：wait_list非空，即有任务阻塞在该mutex锁上，owner在unlock的时候必须要执行唤醒动作。<br><br>MUTEX_FLAG_HANDOFF：为了防止mutex等待队列的任务饿死，在唤醒的时候top waiter会设置该flag（由于乐观自旋的task不断的插入，唤醒的top waiter也未必一定会获取到锁）。有了这个设定后，锁的owner 在解锁的时候会将该锁转交给top waiter，而不是唤醒top waiter去竞争锁。<br><br>MUTEX_FLAG_PICKUP：该flag表示mutex已经万事俱备（即完成了转交），只等待top waiter来持锁。|
|spinlock_t   wait_lock|用于保护等待链表wait_list操作的自旋锁|
|struct optimistic_spin_queue osq|在配置了MUTEX_SPIN_ON_OWNER的时候，mutex支持乐观自旋机制，osq成员就是乐观自旋需要持有的MCS锁队列。数据结构optimistic_spin_queue只有一个tail的成员，如果等于0，那么说明是一个空队列，没有乐观自旋的任务。否则tail指向一个节点是optimistic_spin_node对象队列的尾部（tail并不是一个指针，它是一个cpu number，optimistic_spin_node对象是per-cpu的，有了cpu number就可以定位到其optimistic_spin_node对象）。|
|struct list_head wait_list|Mutex是睡眠锁，当无法获取锁并且不具备乐观自旋条件的时候需要挂入这个等待队列，等待锁的owner释放锁。|
|void *magic|和debug相关的成员|
|Struct lockdep_map dep_map|和debug相关的成员|

大部分的成员都非常好理解，除了osq这个成员，其工作原理示意图如下：

![](http://www.wowotech.net/content/uploadfile/202205/71421652133610.png)

字如其名，Optimistic spin queue就是乐观自旋队列的意思，也就是形成一组处于自旋状态的任务队列。和等待队列不一样，这个队列中的任务都是当前正在执行的任务。Osq并没有直接将这些任务的task struct形成队列结构，而是把per-CPU的mcs lock对象串联形成队列。Mcs lock中有cpu number，通过这些cpu number可以定位到指定cpu上的current thread，也就定位到了自旋的任务。

虽然都是自旋，但是自旋方式并不一样。Osq队列中的头部节点是持有osq锁的，只有该任务处于对mutex的owner进行乐观自旋的状态（我们称之mutex乐观自旋）。Osq队列中的其他节点都是自旋在自己的mcs lock上（我们称之mcs乐观自旋）。当头部的mcs lock释放掉后（结束mutex乐观自旋，持有了mutex锁），它会将mcs lock传递给下一个节点，从而让spinner队列上的任务一个个的按顺序进入mutex的乐观自旋，从而避免了cache-line bouncing带来的性能开销。

2、等待任务对象

由于是sleep lock，我们需要把等待的任务挂入队列。在内核中，Struct mutex_waiter用来抽象等待mutex的任务，其成员描述如下：

|   |   |
|---|---|
|成员|描述|
|struct list_head list|挂入mutex等待队列（wait_list成员）的节点|
|struct task_struct *task|等待该mutex的任务|
|struct ww_acquire_ctx *ww_ctx|说明wait/wound mutex上下文，本文不会深入讲解。|
|void *magic|和debug相关的成员|

3、MCS锁对象

在linux内核中，我们对睡眠锁（例如mutex、rwsem）进行了乐观自旋的优化，这涉及到MCS lock，struct optimistic_spin_node用来抽象乐观自旋的MCS lock，其成员描述如下：

|   |   |
|---|---|
|成员|描述|
|struct optimistic_spin_node *next, *prev|通过这两个成员我们可以把MCS锁串成一个双向链表，睡眠锁对象会指向这个MCS lock链表。对于mutex而言，osq就是定位MCS锁链表尾部的成员（最近试图持锁的那个）。而MSC锁链表头部的成员就是有资格乐观自旋在mutex的那个。|
|int locked|MCS lock的状态，1表示持锁，0表示未持锁|
|int cpu|MCS lock是per cpu的，这个成员表示该MCS 锁对应的cpu id|

三、外部接口

Mutex模块的外部接口API如下：

|   |   |
|---|---|
|接口API|描述|
|初始化接口API|
|mutex_init|初始化mutex对象，该mutex对象已经由调用者定义。|
|__mutex_init|基本类似mutex_init，但允许更灵活的设置debug信息|
|DEFINE_MUTEX|定义并初始化一个mutex对象|
|__MUTEX_INITIALIZER|当mutex嵌入其他数据结构中的时候，通过该API可以初始化数据结构中内嵌的mutex对象|
|获取mutex锁的API|
|mutex_lock|获取mutex锁的接口，如果不成功，进入D状态|
|mutex_lock_interruptible|获取mutex锁的接口，如果不成功，进入S状态|
|mutex_lock_killable|获取mutex锁的接口，如果不成功，进入KILLABLE状态|
|mutex_lock_io|类似mutex_lock，但增加标记iowait的状态，即未能成功获取锁的时候，进入的是io wait D状态|
|mutex_trylock|尝试获取mutex锁的接口，如果不成功，不阻塞，返回0|
|mutex_trylock_recursive|同mutex_trylock，但增加了当前线程重复持锁的检查|
|atomic_dec_and_mutex_lock|结合原子操作的持锁接口。当减一导致原子变量等于0并且成功持锁后返回true，否则返回false。|
|释放mutex锁的API|
|mutex_unlock|释放mutex，离开临界区。|
|获取mutex锁状态的API|
|mutex_is_locked|判断当前mutex锁的状态，如果已经被其他线程持有，那么返回true，否则返回false。|

四、尝试获取锁

和mutex_lock不一样，mutex_trylock只是尝试获取锁，如果成功，那么自然是好的，直接返回true，如果失败，也不会阻塞，只是返回false就可以了。代码主逻辑在__mutex_trylock_or_owner函数中，如下：

|   |
|---|
|owner = atomic_long_read(&lock->owner);<br><br>for (;;) {-------A<br><br>unsigned long old, flags = __owner_flags(owner);<br><br>unsigned long task = owner & ~MUTEX_FLAGS;<br><br>if (task) {<br><br>if (likely(task != curr))<br><br>break;--------------B<br><br>if (likely(!(flags & MUTEX_FLAG_PICKUP)))<br><br>break;--------------C<br><br>flags &= ~MUTEX_FLAG_PICKUP;<br><br>}<br><br>flags &= ~MUTEX_FLAG_HANDOFF;------D<br><br>old = atomic_long_cmpxchg_acquire(&lock->owner, owner, curr \| flags);<br><br>if (old == owner)<br><br>return NULL;------E<br><br>owner = old;<br><br>}<br><br>return __owner_task(owner);|

A、对于mutex的owner成员，它是一个原子变量，我们采用了大量的原子操作来访问或者更新它。然而判断持锁需要一连串的操作，我们并没有采用同步机制（例如自旋锁）来保护这一段的对owner成员操作，因此，我们这些操作放到一个for循环中，在操作的结尾处会判断是否有其他线程插入修改了owner成员，如果中间有其他线程插入，那么就需要重新来过。

B、如果task非空（task变量保存了owner中去掉flag部分的任务指针），并且也不等于current thread，那么说明mutex锁被其他线程持有，还没有释放锁（也有可能在是否锁的时候，把锁直接转交给了其他线程），因此直接break跳出循环，持锁失败。

C、如果task等于current thread，而且设置了MUTEX_FLAG_PICKUP的标记，那么说明持锁线程已经把该mutex锁转交给了本线程，等待本线程来拾取。如果没有MUTEX_FLAG_PICKUP标记，那么也是直接break跳出循环，递归持锁失败。

D、有两种情况会走到这里的时候，一种情况是task为空，说明该mutex锁处于unlocked状态。另外一种情况是task非空，等于current thread，并且mutex发生了handoff，该锁被转交给当前试图持锁的线程。无论哪种情况，都可以去执行持锁操作了。

E、调用atomic_long_cmpxchg_acquire尝试获取锁，如果成功获取了锁（没有其他线程插入修改owner这个原子变量），返回NULL。如果owner发生了变化，说明中间有其他线程插入，那么重新来过。

五、获取mutex锁

    mutex_lock代码如下：

|   |
|---|
|might_sleep();<br><br>if (!__mutex_trylock_fast(lock))<br><br>__mutex_lock_slowpath(lock);|

这里的might_sleep说明调用mutex_lock函数有可能会因为未能获取到mutex锁而进入阻塞状态。在原子上下文中（中断上下文、软中断上下文、持有自旋锁、禁止抢占等），我们不能调用可以引起阻塞的函数，因此在might_sleep函数中嵌入了这个检查，当原子上下文中调用mutex_lock函数的时候，内核会打印出内核栈的信息，从而定位这个异常。当然，这个功能是在设置CONFIG_DEBUG_ATOMIC_SLEEP选项的情况下才生效的，如果没有设置这个选项，might_sleep函数退化为might_resched函数。在配置了抢占式内核（CONFIG_PREEMPT）或者非抢占式内核（CONFIG_PREEMPT_NONE）的情况下，might_resched是空函数。在配置了主动抢占式内核（CONFIG_PREEMPT_VOLUNTARY）的情况下，might_resched会调用_cond_resched函数来主动触发一次抢占。主动抢占式内核通过在might_sleep函数中增加了潜在的调度点实现了比非抢占式内核更好的延迟特性，同时确保抢占带来的进程切换开销低于抢占式内核。

Mutex是一种睡眠锁，如果未能获取锁，那么当前线程会阻塞。不过也许我们试图获取的mutex还处于空闲状态，因此通过__mutex_trylock_fast来尝试获取mutex（mutex_lock的快速路径）：

|   |
|---|
|if (atomic_long_try_cmpxchg_acquire(&lock->owner, &zero, curr))<br><br>return true;<br><br>return false;<br><br>static __always_inline bool<br><br>atomic_long_try_cmpxchg_acquire(atomic_long_t *v, long *old, long new)|

atomic_long_try_cmpxchg_acquire函数有三个参数，从左到右分别是value指针，old指针和new。该函数会对比*value和*old指针中的数值，如果相等执行赋值*value=new同时返回true。如果不相等，不执行赋值操作，直接返回false。

如果lock->owner的值等于0（即不仅task struct地址等于0，所有的flag也要等于0），那么将当前线程的task struct的指针赋值给lock->owner，表示该mutex锁已经被当前线程持有。如果lock->owner的值不等于0，表示该mutex锁已经被其他线程持有或者锁正在传递给top waiter线程，当前线程需要阻塞等待。需要特别说明的是上面描述的操作（比较和赋值）都是原子操作，不能有任何指令插入其中。

在未能获取mutex锁的情况下，我们需要调用__mutex_lock_slowpath函数进入慢速路径。由于会进入睡眠，因此这里需要明确当前线程需要处于的阻塞状态，主要有三种状态：D状态、S状态和KILLABLE。当调用不同的持锁API的时候，当前线程可以处于各种不同的状态。对于mutex_lock（大部分场景）当前线程会进入D状态。主要的代码逻辑在__mutex_lock_common函数中，我们分段解读（省略wait/wound和调试部分的代码）：

|   |
|---|
|preempt_disable();<br><br>if (__mutex_trylock(lock) \|<br><br>    mutex_optimistic_spin(lock, ww_ctx, NULL)) {<br><br>preempt_enable();<br><br>return 0;<br><br>}|

__mutex_trylock用来再次尝试获取锁，mutex_optimistic_spin则是mutex乐观自旋（Optimistic spinning）部分的代码。这两个操作只要有其一能成功获取mutex锁，那么就直接返回了。由于没有进入阻塞状态，因此这个路径也叫做中速路径。

__mutex_trylock在上一节已经讲解了，不再赘述。乐观自旋的思路是因为mutex锁可能是被其他CPU上正在执行中的线程持有，如果临界区比较短，那么有可能该mutex锁很快就被释放。这时候，与其进行一次上下文切换，还不如自旋等待，毕竟上下文切换的开销也是不小的。乐观自旋机制底层使用的是MCS锁，具体的细节我们会在其他文档中描述。

慢速路径的代码如下（省略部分代码）：

|   |
|---|
|__mutex_add_waiter(lock, &waiter, &lock->wait_list);---------A<br><br>waiter.task = current;<br><br>set_current_state(state);<br><br>for (;;) {<br><br>schedule_preempt_disabled();---------B<br><br>if ( !first) {<br><br>first = __mutex_waiter_is_first(lock, &waiter);<br><br>if (first)<br><br>__mutex_set_flag(lock, MUTEX_FLAG_HANDOFF);------C<br><br>}<br><br>set_current_state(state);<br><br>if (__mutex_trylock(lock) \|<br><br>   (first && mutex_optimistic_spin(lock, ww_ctx, &waiter)))<br><br>break;---------------D<br><br>}|

A、所谓慢速路径其实就是阻塞当前线程，这里将current task挂入mutex的等待队列的尾部。这样的操作让所有等待mutex的任务按照时间的先后顺序排列起来，当mutex被释放的时候，会首先唤醒队首的任务，即最先等待的任务最先被唤醒。此外，在向空队列插入第一个任务的时候，会给mutex flag设置上MUTEX_FLAG_WAITERS标记，表示已经有任务在等待这个mutex锁了。

B、进入阻塞状态，触发一次调度。由于目前执行上下文处于关闭抢占状态，因此这里的调度使用了关闭抢占版本的schedule函数。

C、该任务被唤醒之后，如果是等待队列中的第一个任务，即top waiter，那么需要给该mutex设置MUTEX_FLAG_HANDOFF，这样即便本次唤醒后无法获取到mutex（有些在该mutex上乐观自旋的任务可能会抢先获得锁），那么下一次owner释放锁的时候，看到这个handoff标记也会进行锁的交接，不再是大家抢来抢去。通过这个机制，我们可以防止spinner队列中的任务抢占CPU资源，饿死waiter队列中的任务。

D、如果获取到mutex，那么就退出循环，否则继续进入阻塞状态等待。如果是队列中的第一个waiter，那么如果__mutex_trylock失败，那么就进入乐观自旋过程，这样会有更大的机会成功获取mutex锁。

六、乐观自旋

Mutex乐观自旋的代码位于mutex_optimistic_spin函数中，进入乐观自旋函数的线程可能有下面几个结果：

1、成功获取osq锁，进入mutex乐观自旋状态，当owner释放mutex锁后，该线程结束乐观自旋，成功持有了mutex，返回true

2、未能获取osq锁，在自己的MCS锁上乐观自旋。一旦成功持锁，同步骤1

3、在MCS锁或者mcs锁乐观自旋的时候，由于各种原因（例如owner进入阻塞状态）而无法继续乐观自旋，那么mutex_optimistic_spin函数返回false，告知调用者乐观自旋失败，进入等待队列。

我们分两段来解析。首先来看第一段：

|   |
|---|
|if (!waiter) {--------------A<br><br>if (!mutex_can_spin_on_owner(lock))<br><br>goto fail;--------------B<br><br>if (!osq_lock(&lock->osq))<br><br>goto fail;---------------C<br><br>}|

调用mutex_optimistic_spin函数的场景有两个，一个是waiter等于NULL，这是发生在mutex_lock的早期，这时候试图持锁的线程还没有挂入等待队列，因此waiter等于NULL。另外一个场景是持锁未果，挂入等待队列，然后被唤醒之后的乐观自旋。这时候试图持锁的线程已经挂入等待队列，因此waiter非空。在这种场景下，刚唤醒的top waiter线程会给与优待，因此不需要持有osq锁就可以长驱直入，进入乐观自旋。

A、当waiter为空时，因为是正常路径的持锁请求，所以在乐观自旋之前需要持有osq锁，只有获得了osq锁，当前线程才能进入mutex乐观自旋的过程。否则只能是在自己的MCS锁上自旋等待。

B、是否乐观自旋等待mutex可以从两个视角思考：一方面，如果本cpu已经设置了need resched标记，那说明有其他任务想要抢占当前试图持锁的任务。那么current task何必乐观自旋呢，赶紧的去sleep为其他任务让路吧。另外一方面需要从owner的行为来判断。如果owner正在其他cpu欢畅运行，那么可以考虑进入乐观自旋过程。

C、在基于共享内存的多核计算系统中，mutex的实现是通过一个共享变量（owner成员）和一个队列来完成复杂的控制的。如果有多个cpu上的线程同时乐观自旋在这个共享变量上，那么就会出现缓存踩踏现象。为了解决这个问题，我们控制不能让太多的线程进入mutex乐观自旋状态（轮询owner成员），只有那些获取了osq锁的线程才能进入。未能持osq锁的线程会进入mcs锁的乐观自旋过程，等待osq锁的owner（当前在mutex乐观自旋）释放osq锁。关于osq锁的细节我们在其他文章中描述。

完成了持osq锁之后（或者是被唤醒的top waiter线程，它会掠过osq持锁过程），我们就可以进入mutex乐观自旋了，代码如下：

|   |
|---|
|for (;;) {<br><br>struct task_struct *owner;<br><br>owner = __mutex_trylock_or_owner(lock);<br><br>if (!owner)<br><br>break;--------A<br><br>if (!mutex_spin_on_owner(lock, owner, ww_ctx, waiter))<br><br>goto fail_unlock;--------B<br><br>}------C|

A、首先还是调用__mutex_trylock_or_owner试图获取mutex锁，如果返回的owner非空（需要注意的是：这里的owner变量不包括mutex flag部分），那么说明mutex锁还在owner task手中。如果owner是空指针，说明原来持有锁的owner已经释放锁，同时这也就说明当前线程持锁成功，因此退出乐观自旋的循环。需要注意的是在退出mutex乐观自旋后会释放osq锁，从而会让spinner队列中的下一个mcs锁自旋的任务进入mutex乐观自旋状态。

B、如果__mutex_trylock_or_owner返回了非空owner，说明当前线程获取锁失败，那么可以进入mutex乐观自旋了。所谓自旋不是自旋在spinlock上，而是不断的循环检测锁的owner task是否发生变化以及owner task的运行状态。如果owner阻塞了或者当前cpu有resched的需求（可能唤醒更高级任务），那么就停止自旋，返回false，走入fail_unlock流程。

C、如果mutex锁的owner task发生变化（例如变成NULL）则mutex_spin_on_owner函数返回true，则说明可以跳转到for循环处再次尝试获取锁并进行乐观自旋。

七、释放mutex锁

mutex_unlock的代码如下：

|   |
|---|
|if (__mutex_unlock_fast(lock))<br><br>return;<br><br>__mutex_unlock_slowpath(lock, _RET_IP_);|

如果一个线程获取了某个mutex锁之后，没有任何其他的线程试图进入临界区，那么这时候mutex的owner成员就是该线程的task struct地址，并且所有的mutex flag都是clear的。在这种情况下，将mutex的owner成员清零即可，不需要额外的操作，我们称之解锁快速路径（__mutex_unlock_fast）。当然，如果有其他线程在竞争该mutex锁，那么情况会更复杂一些，这时候我们进入慢速路径（_mutex_unlock_slowpath），慢速路径的逻辑分成两段：一段是释放mutex锁，另外一段是唤醒top waiter线程。我们首先一起看第一段的代码，如下:

|   |
|---|
|owner = atomic_long_read(&lock->owner);<br><br>for (;;) {<br><br>unsigned long old;<br><br>if (owner & MUTEX_FLAG_HANDOFF)-----A<br><br>break;<br><br>old = atomic_long_cmpxchg_release(&lock->owner, owner,<br><br>  __owner_flags(owner));------B<br><br>if (old == owner) {<br><br>if (owner & MUTEX_FLAG_WAITERS)-----C<br><br>break;<br><br>return;<br><br>}<br><br>owner = old;--------D<br><br>}|

A、如果mutex flag中设定了handoff标记，那么说明owner在释放锁的时候要主动的把锁的owner传递给top waiter，不能让后来插入的乐观自旋的线程饿死top waiter。因此这时候我们还不能放锁，需要在__mutex_handoff函数中释放锁给top waiter。

B、将owner的task struct地址部分清掉，这也就是意味着owner task放弃了持锁。这时候，如果有乐观自旋的任务在轮询mutex owner，那么它会立刻感知到锁被释放，因此可以立刻获取mutex锁。在这样的情况下，即便后面唤醒了top waiter，但为时已晚。

C、如果等待队列中有任务阻塞在这个mutex中，那么退出循环，执行慢速路径中的第二段唤醒逻辑，否则直接返回，无需唤醒其他线程。

D、在操作owner的过程中，如果有其他线程对owner进行的修改（没有同步机制保证多线程对owner的并发操作），那么重新设定owner，再次进行检测。

第二段唤醒top waiter的代码如下：

|   |
|---|
|if (!list_empty(&lock->wait_list)) {------------A<br><br>struct mutex_waiter *waiter =<br><br>list_first_entry(&lock->wait_list, struct mutex_waiter, list);<br><br>next = waiter->task;<br><br>wake_q_add(&wake_q, next);<br><br>}<br><br>if (owner & MUTEX_FLAG_HANDOFF)--------B<br><br>__mutex_handoff(lock, next);<br><br>wake_up_q(&wake_q);--------C|

A、代码执行至此，需要唤醒top waiter，或者处理将锁转交top waiter的逻辑，无论哪种情况，都需要从等待队列中找到top waiter。找到后将其加入wake queue。

B、如果有任务（一般是top waiter，参考其唤醒后的代码逻辑）请求handoff mutex，那么调用__mutex_handoff函数可以直接将owner设置为top waiter任务，然后该任务在醒来之后直接pickup即可。这相当与给了top waiter一些特权，防止由于不断的插入乐观自旋的任务而导致无法获取CPU资源。

C、唤醒top waiter任务

八、结论

本文简单的介绍了linux内核中的mutex同步机制，在移动环境中，mutex锁的性能表现不尽如人意，无论是吞吐量还是延迟。在重载的场景下，我们经常会遇到Ux线程阻塞在mutex而引起的手机卡顿问题，如何在手机平台上优化mutex锁的性能是我们OPPO内核团队一直在做的事情，也欢迎热爱技术的你积极参与。

参考文献：

1、内核源代码

2、linux-5.10.61\Documentation\scheduler\*

3、[https://zhuanlan.zhihu.com/p/364130923](http://www.wowotech.net/admin/#/_webview)

本文首发在“内核工匠”微信公众号，欢迎扫描以下二维码关注公众号获取最新Linux技术分享：

![](http://www.wowotech.net/content/uploadfile/202111/b9c21636066902.png)

标签: [mutex](http://www.wowotech.net/tag/mutex)

[![](http://www.wowotech.net/content/uploadfile/201605/ef3e1463542768.png)](http://www.wowotech.net/support_us.html)

« [Linux内核同步机制之（九）：Queued spinlock](http://www.wowotech.net/kernel_synchronization/queued_spinlock.html) | [schedutil governor情景分析](http://www.wowotech.net/process_management/schedutil_governor.html)»

**评论：**

**yanl1229**  
2022-08-09 18:10

2. 由于task struct地址是L1_CACHE_BYTES对齐的，因此这个成员的有若干的LSB可以被用于标记状态（在ARM64平台上，L1_CACHE_BYTES是64字节，因此LSB 6-bit会用于保存mutex状态信息  
  
这句话是不是描述错误了，我看4.19的内核是:  
static inline struct task_struct *__mutex_owner(struct mutex *lock)  
{  
    return (struct task_struct *)(atomic_long_read(&lock->owner) & ~0x07);  
}  
是低3位，并不是低6位.

[回复](http://www.wowotech.net/kernel_synchronization/504.html#comment-8666)

**orangeboyye**  
2022-08-10 23:58

@yanl1229：有6位可用，目前只用了3位。

[回复](http://www.wowotech.net/kernel_synchronization/504.html#comment-8668)

**发表评论：**

 昵称

 邮件地址 (选填)

 个人主页 (选填)

![](http://www.wowotech.net/include/lib/checkcode.php) 

- ### 站内搜索
    
       
     蜗窝站内  互联网
    
- ### 功能
    
    [留言板  
    ](http://www.wowotech.net/message_board.html)[评论列表  
    ](http://www.wowotech.net/?plugin=commentlist)[支持者列表  
    ](http://www.wowotech.net/support_list)
- ### 最新评论
    
    - ja  
        [@dream：我看完這段也有相同的想法，引用 @dream ...](http://www.wowotech.net/kernel_synchronization/spinlock.html#8922)
    - 元神高手  
        [围观首席power managerment专家](http://www.wowotech.net/pm_subsystem/device_driver_pm.html#8921)
    - 十七  
        [内核空间的映射在系统启动时就已经设定好，并且在所有进程的页表...](http://www.wowotech.net/process_management/context-switch-arch.html#8920)
    - lw  
        [sparse模型和disconti模型没看出来有什么本质区别...](http://www.wowotech.net/memory_management/memory_model.html#8919)
    - 肥饶  
        [一个没设置好就出错](http://www.wowotech.net/linux_kenrel/516.html#8918)
    - orange  
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
    
    - [一例hardened_usercopy的bug解析](http://www.wowotech.net/linux_kenrel/480.html)
    - [Linux kernel的中断子系统之（二）：IRQ Domain介绍](http://www.wowotech.net/irq_subsystem/irq-domain.html)
    - [Linux CPU core的电源管理(5)_cpu control及cpu hotplug](http://www.wowotech.net/pm_subsystem/cpu_hotplug.html)
    - [进程管理和终端驱动：基本概念](http://www.wowotech.net/process_management/process-tty-basic.html)
    - [我的bash shell学习笔记](http://www.wowotech.net/linux_application/134.html)
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