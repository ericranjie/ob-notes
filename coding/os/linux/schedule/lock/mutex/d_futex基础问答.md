
作者：[OPPO内核团队](http://www.wowotech.net/author/538) 发布于：2022-10-9 7:20 分类：[内核同步机制](http://www.wowotech.net/sort/kernel_synchronization)

# 一、什么是futex？

futex是Fast Userspace muTEX的缩写，该机制是由Rusty Russell、Hubertus Franke和Mathew Kirkwood在2.5.7版本的内核中引入，虽然名字中有互斥锁（mutex）的含义，但实际它是一种用于用户空间应用程序的通用同步工具（基于futex可以在userspace实现互斥锁、读写锁、condition variable等同步机制）。Futex组成包括：

1、内核空间的等待队列
2、用户空间层的32-bit futex word（所有平台都是32bit，包括64位平台）

在没有竞争的场景下，锁的获取和释放性能都非常高，不需要内核的参与，仅仅是通过用户空间的原子操作来修改futex word的状态即可。在有竞争的场景下，如果线程无法获取futex锁，那么把自己放入到 wait queue中（陷入内核，有系统调用的开销），而在owner task释放锁的时候，如果检测到有竞争（等待队列中有阻塞任务），就会通过系统调用来唤醒等待队列中的任务，使其恢复执行，继续去持锁。如果没有竞争，那么也无需陷入内核。

# 二、Futex用户和内核空间接口API是什么？

Futex接口函数的原型如下：

|                                                                                                                                                                                                                              |
| ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| #include<br><br>       #include<br><br>       int futex(int \*uaddr, int futex_op, int val,<br><br>                 const struct timespec *timeout,   /* or: uint32_t val2 \*/<br><br>                 int \*uaddr2, int val3); |

Futex系统调用的复杂性体现在其参数上，要理解futex需要充分理解其参数：

|   |   |
|---|---|
|参数|描述|
|uaddr|32-bit Futex word的地址<br><br>对于normal futex，内核并不关心futex word表示什么含义，对于pi futex，内核空间和用户空间按照下面的方式解释32-bit futex word：<br><br>1、owner tid（lsb 30-bit）<br><br>2、Flag（msb 2-bit）<br><br>futex word等于0的时候表示空锁状态。当owner tid设定为具体的thread id的时候表示持锁状态。如果有任务持锁失败进入等待队列，那么需要或一个FUTEX_WAITERS的flag。|
|futex_op|Futex_op包括两个部分：futex操作码（LSM 7-bit，共计最大支持128个op code）和futex flag（其他bit）。futex系统调用支持各种各样的操作码，我们下面会详细介绍。Futex flag用来定义futex操作码的行为，具体包括：<br><br>FUTEX_PRIVATE_FLAG：为了能够实现同步，futex word会在多个执行线索中共享。如果futex word仅在一个进程内共享，那么它是process-private。如果futex word在多个进程内共享，那么它process-shared。<br><br>FUTEX_CLOCK_REALTIME：有些操作码（例如FUTEX_WAIT）会有timeout参数（val2）用来定义阻塞的超时时间。这个bit用来控制timeout超时参数使用哪一个基准clock的。如果设置则使用realtime clock，否则使用monotonic clock。|
|val|和Futex操作字相关|
|Val2|和Futex操作字相关|
|Uaddr2|和Futex操作字相关|
|Val3|和Futex操作字相关|

futex系统调用支持各种各样的操作码，如下：

1、FUTEX_WAIT：如果futex word中仍然保存着参数val给定的值，那么当前线程则进入睡眠，等待FUTEX_WAKE的操作唤醒它。
2、FUTEX_WAKE：最多唤醒val个等待在futex word上的线程。Val或者等于1（唤醒1个等待线程）或者等于INT_MAX（唤醒全部等待线程）
3、FUTEX_WAIT_BITSET：同FUTEX_WAIT，只不过多提供一个mask的参数
4、FUTEX_WAKE_BITSET：同FUTEX_WAKE，只不过多提供一个mask参数用来选择唤醒哪一个waiter。
5、FUTEX_LOCK_PI：PI版本的FUTEX_WAIT
6、FUTEX_UNLOCK_PI：PI版本的FUTEX_WAKE
7、FUTEX_REQUEUE：这个操作包括唤醒和移动队列两个动作。唤醒val个等待在uaddr上的waiter，如果还有其他的waiter，那么将这些等待在uaddr的waiter转移到uaddr2的等待队列上去（最多转移val2个waiter）
8、FUTEX_CMP_REQUEUE：同上，不过需要对比val3这个uaddr的期望值。

除了futex wait和wake这样的基本操作，futex还有其他应用在复杂场景的操作码，由于在手机场景没有使用，本文不再介绍。

我们整理各个操作码的参数如下：

|   |   |   |   |   |   |
|---|---|---|---|---|---|
|操作码|uaddr|uaddr2|val|val2|val3|
|FUTEX_WAIT|Futex word|未使用|Futex word的期望值|Timeout|未使用|
|FUTEX_WAKE|Futex word|未使用|唤醒等待线程的个数|Timeout|未使用|
|FUTEX_WAIT_BITSET|Futex word|未使用|Futex word的期望值|Timeout|Bit mask|
|FUTEX_WAKE_BITSET|Futex word|未使用|唤醒等待线程的个数|Timeout|Bit mask|
|FUTEX_REQUEUE|Src futex word|Dest futex word|唤醒waiter的数量|最多requeue的waiter的数量|未使用|
|FUTEX_CMP_REQUEUE|Src futex word|Dest futex word|唤醒waiter的数量|最多requeue的waiter的数量|Uaddr中的期望值|
|FUTEX_WAKE_OP|第一个futex|第二个futex|唤醒的waiter数量（uaddr）|唤醒waiter数量（uaddr2）|如何对uaddr2进行操作|
|FUTEX_LOCK_PI|Futex word|未使用|未使用|Timeout|未使用|
|FUTEX_UNLOCK_PI|Futex word|未使用|未使用|未使用|未使用|

# 三、对于normal futex，内核中如何组织等待队列？

Futex相关的数据结构组织如下图所示：
!\[\[Pasted image 20241008190725.png\]\]

从逻辑上看，通过futex实现的互斥锁和内核中的互斥锁mutex是一样的（通过futex实现的读写锁的概念和内核的rwsem也是一样，不再赘述），只不过futex互斥锁是分裂开的：futex word和等待队列是分别在用户空间和内核空间，内核的mutex互斥锁可以讲把待队列头放置在mutex对象上，但是对于futex，我们没有对应的内核锁对象，因此我们就需要一个算法将futex word和其等待队列映射起来。为了管理挂入等待队列的futex阻塞任务，内核建立了一个hansh table如下：

|   |
|---|
|static struct {<br><br>struct futex_hash_bucket \*queues;<br><br>unsigned long            hashsize;<br><br>} \_\_futex_data|

在初始化的时候，内核会构建hashsize个futex hash bucket结构，每个bucket用来管理futex链表（hash key相同）。futex_hash_bucket数据结构定义如下：

|   |
|---|
|struct futex_hash_bucket {<br><br>atomic_t waiters;----在该队列上阻塞的任务个数<br><br>spinlock_t lock;---保护队列的锁<br><br>struct plist_head chain;----hash key相同的futex阻塞任务对象会挂在这个链表头上<br><br>} \_\_\_\_cacheline_aligned_in_smp;|

每一个等待在futex word的task都有一个futex_q对象（后文称之futex阻塞任务对象），根据其哈希值挂入不同的队列：

|   |
|---|
|struct futex_q {<br><br>struct plist_node list;---挂入队列的节点，按照优先级排序<br><br>struct task_struct \*task;---等待在futex word的任务<br><br>spinlock_t \*lock_ptr;---保护队列操作的自旋锁<br><br>union futex_key key;---计算hash key的数据结构<br><br>struct futex_pi_state \*pi_state;---futex的pi-aware是通过rt mutex实现，pi_state是记录futex关于优先级继承的所有信息，包括rt mutex 对象。<br><br>struct rt_mutex_waiter \*rt_waiter;---每个阻塞在rt mutex的任务都需要一个rt_mutex_waiter对象<br><br>union futex_key \*requeue_pi_key;---和requeue pi相关，非本文关注的内容。<br><br>u32 bitset;---在futex wake的时候用来判断是否唤醒该任务<br><br>} \_\_randomize_layout;|

通过上面的数据结构，只要有了futex word，那么我们就能根据hash key定位到其挂入的链表。当然，为了精准的匹配，还需要其futex key完全相等，具体请参考match_futex函数。关于优先级继承相关的成员后面会详细描述。

# 四、Futex wait的流程为何？

futex_wait函数的流程如下：

1、如果参数中给定了timeout，那么调用futex_setup_timer来创建一个hrtimer来打断futex wait阻塞状态。
2、通过三元组计算futex hash key，对于process-private类型的futex word，hash key是根据进程地址空间和futex word的虚拟地址来计算，也就是说三元组是( current->mm, address, 0 )。对于share类型的futex word，它会被放置到共享的内存中（通过mmap或者shmat）。在这种场景下，futex word在不同进程中有不同的虚拟地址，但是物理地址是相同的，通过地址空间中的虚拟地址来计算hash key是行不通的。因此share类型的futex word使用的三元组( inode->i_sequence, page->index, offset_within_page )这样的组合来计算hash key。具体的细节请参考get_futex_key函数
3、有了hash key，我们就可以通过这个key找到哈希表中对应的表头（后文称之hash bucket）。由于后续会把本次futex阻塞任务对象（futex_q）挂入hash bucket，因此需要上锁。
4、在真正插入链表之前还需要校验用户空间传递来的期望值是否发生了变化（表示用户空间有其他线程对该futex word进行了修改），如果保持不变，那么就可以放心插队了，否则返回EWOULDBLOCK，当然，不要忘记解锁。
5、插队动作是在futex_wait_queue_me函数中完成。插队是考虑了优先级的：对于rt线程，优先级高的排在队首，低的在队尾。对于cfs任务，不按照优先级排队，而是采用了FIFO这样的公平策略。同样的，完成插队后不要忘记解锁。
6、马上就要阻塞了，如果参数中给定了timeout，这时候就需要启动步骤1中设置的hrtimer了。
7、在真正阻塞之前，还要进一步进行验证，毕竟这时候有可能其他的执行线索（可能是其他线程的futex wake，也可能是timeout callback）完成出队操作。这时候就不能阻塞，否者这个线程可能再也无法醒来。
8、在步骤7中阻塞后，可能有多个唤醒场景：如果任务被正常唤醒（futex wake唤醒），那么其实已经完成出队的动作，这时候直接返回即可，当然，如果有启动hrtimer，我们需要取消它。
9、如果本次futex阻塞任务对象（futex_q）仍然挂在hash bucket的链表上，那表示是有异常发生，需要进行相应的处理并在当前上下文完成出队。具体有两种情况：超时或者被信号打断。
10、如果设置了超期时间，那么在当前上下文会定义hrtimer_sleeper的对象，如果的确是超期唤醒的话，在timer的上下文中会把hrtimer_sleeper中的task成员清掉（设置为NULL），通过这个可以判断是否是超期唤醒。
11、如果当前任务有pending的信号，那说明是被信号打断。如果没有pending信号，那说明是spurious wakeup，需要再尝试一次futex入队操作。
12、一般而言，如果被信号打断，直接返回ERESTARTSYS，让用户空间程序自己决定怎么后续处理就OK了。但是有一种情况例外，那就是设置了timeout（即还没有超期就被信号打断），这种场景需要restart syscall。

# 五、Futex wake的流程为何？

相比futex_wait，futex_wake就比较简单了，其核心操作就是出队和唤醒futex wait阻塞的任务，具体流程如下：

1、首先通过hash key找到对应的hash bucket，这个操作和futex_wait中是一样的。
2、hash bucket中的链表上的futex阻塞任务对象（futex_q）只是由于hash key相同而走到一起的，实际上并非一定是对应的futex word，因此我们需要遍历链表进行匹配。具体匹配的准则就是三元组完全相等。
3、三元组相等只能说明futex word是对应上了，但是futex机制也提供了用户可以控制唤醒的方法：比特匹配。在futex wait的时候，上层的应用程序可以传递bitset参数来标记自己（FUTEX_WAIT_BITSET），在futex wake的时候，应用程序会传递bitset参数来通知内核自己想要唤醒哪些线程（FUTEX_WAKE_BITSET）。对于FUTEX_WAIT和FUTEX_WAKE，bitset做了特殊处理，设置为FUTEX_BITSET_MATCH_ANY，即futex wake的时候可以唤醒任何阻塞在该futex word的线程。
4、除了bitset，futex wake还可以控制唤醒线程的个数。为了完成多个线程的唤醒，这里使用了唤醒队列（wake queue）。当找到匹配的futex_q的时候，将其从hash bucket的队列中删除，加入到唤醒队列上来。需要注意的是：在进行这些队列操作的时候需要持有hash buck的自旋锁。
5、完成指定数量的扫描之后会结束遍历，调用wake_up_q将wake queue的任务逐个唤醒。

# 六、Futex requeue是什么鬼？

在讲requeue流程之前我们需要先明白为何会有requeue这个op code。我们以java中的wait-notify机制来说明这个问题。我们有如下的java代码：

```cpp
A临界区：
Synchronized {
......
Wait（）；-----阻塞在condition varible，之前需要释放monitor lock。
......
}

B临界区：
Synchronized {
......
Notify（）；-----唤醒阻塞在wait上的线程，之后会立刻获取monitor lock
......
}
```

Java中的Wait和notify的功能是native实现，在虚拟机提供支持。Synchronized是java内嵌锁，在虚拟机对应monitor lock（互斥锁），A临界区和B临界区都由monitor lock保护，确保了只有一个线程进入。为了确保A、B临界区的先后关系（A临界区需要等待B临界区的事件通知），我们引入了condition varible。在wait-notify场景中有两个等待队列：一个是monitor lock的等待队列，另外一个是condition varible的等待队列。而对于wait而言，它需要涉及两个等待队列的操作：一个是释放monitor lock（唤醒其等待队列的任务），一个是阻塞在条件变量上（把自己挂入其等待队列）。如果没有requeue，那么这样的操作需要两次futex的系统调用，有了futex requeue，一次futex就OK了。

了解了requeue的由来，其流程也是非常的简单，特别是有了上面两节futex wait和futex wake基础。Requeue的流程如下（requeue有normal requeue和pi requeue，这里我们主要描述normal requeue的流程）：

1、Requeue涉及两个futex，分别用uaddr1和uaddr2表示。这里需要唤醒nr_wake个uaddr1上的线程，同时把其上的nr_requeue个等待任务对象转移到uaddr2对应的等待队列上。首先调用get_futex_key获取两个futex的hash key，并根据hash key找到对应的hash bucket（hash_futex函数）
2、如果是FUTEX_CMP_REQUEUE，那么我们还需要校验uaddr1中的值。需要特别说明的是：这里涉及内核空间访问用户空间的变量，读操作是一个非常复杂的过程，具体参考get_futex_value_locked函数。这些逻辑和本文的主题关系不大，就不再赘述了。
3、遍历uaddr1 等待队列上的所有等待任务对象（futex_q），将nr_wake个futex_q通过mark_wake_futex暂存在wake_q唤醒队列上。通过requeue_futex将uaddr1 等待队列上nr_requeue个futex_q对象转移到uaddr2的等待队列上。注意，这些操作需要持有两个hash bucket的自旋锁。
4、调用wake_up_q函数唤醒之前挂入唤醒队列的任务

# 七、为何futex要支持PI？

Non-PI futex引起的优先级翻转（priority inversion）问题如下图所示：

![[Pasted image 20241024184134.png]]

低优先级任务C首先持锁，这样当高优先级任务A试图持锁失败进入D状态。一般而言，C任务临界区比较短，完成之后就释放锁，任务A就可以执行了。然而，在C执行过程中，中等优先级的任务B被唤醒，抢占了任务C的执行，这时候，所有优先级在A和C之间的任务都可以抢占C的执行，从而使得任务A无法在确定的时间内获取到CPU资源。

PI futex中的PI就是priority inheritance，可以通过优先级继承的方法来解决系统中出现的优先级翻转问题。具体的方法就是当任务A持锁失败的时候，锁的owner task（即任务C）需要临时性的把优先级提升至任务A的优先级。而在释放锁的时候，将其优先级进行恢复原值。

当然，上面只是一个简单的例子，实际系统会涉及更多的锁和线程，但原理类似。对于线程，我们需要记录：

1、该线程持锁哪些锁，这些锁的top waiter是谁，对所有的top waiter按照优先级进行排序。
2、该线程阻塞在哪一把锁上

对于锁，我们需要记录：

1、该锁的owner是谁
2、阻塞在该锁的线程们（按照优先级进行排序）。注意，这里我们把优先级最高的那个阻塞线程叫做该所的top waiter。

有了这些信息，我们需要维持一个准则就OK了：一个任务的临时优先级应该提升至其持有锁的top waiter线程中最高的那个优先级。

# 八、Rt mutex的原理为何？

PI-futex是通过rt mutex来实现的，因此我们这里简单的聊一聊内核的这个PI-aware mutex。

从rt mutex的视角看任务：

![[Pasted image 20241024184121.png]]

rt_mutex_waiter用来抽象一个阻塞在rt mutex的任务：task成员指向这个任务，lock成员指向对应的rt mutex对象，tree_entry是挂入blocker红黑树的节点，rt mutex对象的waiters成员就是这颗红黑树的根节点（wait_lock成员用来保护红黑树的操作）。而owner则指向持锁的任务。需要特别说明的是waiters这个红黑树是按照任务优先级排序的，left most节点就是对应该锁的top waiter。

从任务的视角来看rt mutex：

![[Pasted image 20241024184109.png]]

为了支持rt mutex，task struct也增加了若干的成员，最重要的就是pi_waiters。由于一个任务可以持有多把锁，每把锁都有top waiter，因此和一个任务关联的top waiter也有非常多，这些top waiter形成了一个红黑树（同样也是按照优先级排序），pi_waiters成员就是这颗红黑树的根节点。这颗红黑树的left most的任务优先级就是实现优先级继承协议中规定要临时提升的优先级。pi_top_task成员指向了left most节点对应的任务对象，我们称之top pi waiter。Task struct的pi_blocked_on成员则指向其阻塞的rt_mutex_waiter对象。

有了上面的基本概念之后，我们讲一下PI chain的概念。首先看看任务和锁的基本关系，如下图所示：

![[Pasted image 20241024184052.png]]

在上面的图片中，task 1持有了Lock A和Lock B，阻塞在Lock C上。一个任务只能阻塞在一个锁上，所以红色箭头只能是从任务到锁，不能分叉。由于一个任务可以持有多把锁，所以黑色箭头会有多个锁指向一个任务，即多把锁汇聚于任务。有了这个基本的关系图之后，我们可以形成更加复杂的任务和锁的逻辑图，如下：

![[Pasted image 20241024184043.png]]

在上面这张图中有四条PI chain：

1、Lock D--->task 2
2、task 4--->Lock D--->task 2
3、Lock A--->task 1--->Lock C--->task 2
4、task 3--->Lock B--->task 1--->Lock C--->task 2

为了能够让PI正常起作用，PI chain中的任务必须维持这样的关系：处于PI chain中右端的任务的优先级必须大于等于PI chain中左端的任务们。我们以第四条PI chain为例，任务2的优先级必须大于等于任务1和任务3的优先级，而任务1的优先级必须要大于等于任务3的优先级。

# 九、PI futex和rt mutex有什么关系？

熟悉Linux的工程师都了解内核中的mutex互斥锁以及支持PI的互斥锁版本rt mutex。如果想让用户空间的互斥锁实现优先级继承的功能，那么其实不需要futex模块实现复杂的PI chain，实际上对PI状态的跟踪是通过rt mutex代理来完成的，原理图如下：

![[Pasted image 20241024184031.png]]

我们先看接口部分，normal futex使用FUTEX_WAIT和FUTEX_WAKE操作码来完成阻塞和唤醒的动作。对于PI futex而言，FUTEX_LOCK_PI用来执行上锁，而FUTEX_UNLOCK_PI用来完成解锁。这里的lock和unlock其实是对futex的代理rt mutex而言的。

无论是normal futex还是PI futex，阻塞于futex的任务都会有一个futex_q对象与之对应。对于normal futex，有了futex_q对象，挂入等待队列和将其唤醒的功能都能轻松实现。对于PI futex，我们不仅仅需要挂入队列和唤醒任务，最重要的是我们需要根据PI chain完成任务优先级的调整。为了完成这个功能，需要两个额外的对象，一个是rt_mutex_waiter，表示一个阻塞在rt mutex的任务，其rt mutex指针指向了其阻塞在哪个rt mutex上。另外一个是futex_pi_state对象，它记录了优先级翻转的信息，包括该用户空间上层锁对应的内核态的rt mutex，rt mutex的owner任务的信息等。

# 十、Pi futex逻辑过程

Pi futex主要有两个逻辑过程：通过FUTEX_LOCK_PI上锁，通过FUTEX_UNLOCK_PI完成释放锁的逻辑。

这里的“上锁”有点误导，不是“试图持锁”的意思，而是竞争上层锁失败之后，陷入内核准备进入阻塞状态。这里为了记录PI state，所以需要对代理rt mutex执行上锁的动作（基本上也是会阻塞在rt mutex上）。对于pi futex的。正常futex的部分，例如get hash key、找futex对应的hash bucket、插入hash队列等操作，这里不再描述，主要看PI futex特有的部分。

![[Pasted image 20241024183923.png]]

第一次futex lock pi稍微复杂一点，需要完成owner持锁和current task的阻塞在锁上这两个动作。注意：这里的锁指的是rt mutex。当线程持上层锁成功的时候，我们并不能同时对rt mutex持锁成功并设置owner，因此这时候并不会有futex系统调用进入内核。当第一次阻塞的时候，会通过futex系统调用把owner id传递给内核，这时候我们需要分配一个futex pi state对象创建一个rt mutex，同时建立这个rt mutex和owner task的关系：

1、挂入owner task的futex pi state链表。一个任务可以持有多把上层锁，所以需要链表管理，当然不一定每一个任务持有的上层锁都有对应的futex pi state对象，没有竞争也就不会陷入内核调用FUTEX_LOCK_PI。

2、futex pi state对象的owner成员指向对应的owner task

第二个重要的动作就是让current task去获取rt mutex，上面刚刚设定了owner，这里current task持锁的结果大概率就是会阻塞，不过我们本来就是通过这个阻塞关系来完成PI 状态的跟踪的。rt_mutex_waiter对象抽象了一个阻塞在rt mutex的任务，我们需要建立rt_mutex_waiter对象、阻塞任务和rt mutex的关系，具体包括：

1、rt_mutex_waiter对象的lock成员指向对应于的rt mutex，表示该任务阻塞在这个锁上。rt_mutex_waiter对象的task成员指向当前要阻塞的任务对象。
2、将rt_mutex_waiter对象插入rt mutex的waiters红黑树。
3、task struct的pi_blocked_on设置为该rt_mutex_waiter对象。
4、对于rt mutex而言，有了新的阻塞任务，如果优先级比目前该rt mutex的top waiter更高的话，那么需要更新owner task的top waiter，将旧的top waiter节点从红黑树中删除，将新的top waiter插入owner task的top waiter红黑树
5、根据新的top waiter更新owner task的动态优先级。一旦修改了owner task的优先级，那么其相关的PI chain都需要进行优先级调整。

第二次以及后续的FUTEX_LOCK_PI会简单一点，因为不需要新建rt mutex对象了，只需要在bucket找到第一个futex_q对象，通过其pi state指针就可以定位rt mutex了。有了rt mutex，通过上锁即可让自己阻塞在这个rt mutex上了。

FUTEX_UNLOCK_PI的流程留给读者自行分析了。

# 十一、小结

本文通过问答的形式简单的介绍了内核futex机制，它是上层同步机制的基石。在PI Futex的介绍中，我们对rt mutex浅尝辄止，读者未能领略其全貌。后续我们会出一篇关于rt mutex的文章，敬请期待。

# 参考文献：

1、linux-5.10.61内核源代码
2、linux-5.10.61\\Documentation\\locking\*

本文首发在“内核工匠”微信公众号，欢迎扫描以下二维码关注公众号获取最新Linux技术分享：

标签: [futex](http://www.wowotech.net/tag/futex)

---

« [Linux读写锁逻辑解析](http://www.wowotech.net/kernel_synchronization/rwsem.html) | [如何使能500个virtio_blk设备](http://www.wowotech.net/linux_kenrel/509.html)»

**评论：**

**[顶点软件](http://ddboke.net/)**\
2023-05-08 21:15

你写得非常清晰明了，让我很容易理解你的观点。

[回复](http://www.wowotech.net/kernel_synchronization/futex.html#comment-8782)

**王**\
2022-10-26 16:47

不错，休闲刚好学习一下

[回复](http://www.wowotech.net/kernel_synchronization/futex.html#comment-8693)

**狗子**\
2024-04-16 08:34

@王：老铁 休息就别卷了 受不了

[回复](http://www.wowotech.net/kernel_synchronization/futex.html#comment-8885)

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

  - [eMMC 原理 1 ：Flash Memory 简介](http://www.wowotech.net/basic_tech/flash_memory_intro.html)
  - [Linux kernel内存管理的基本概念](http://www.wowotech.net/memory_management/concept.html)
  - [ARMv8之Observability](http://www.wowotech.net/armv8a_arch/Observability.html)
  - [“极致”神话和产品观念](http://www.wowotech.net/tech_discuss/140.html)
  - [X-014-KERNEL-ARM GIC driver的移植](http://www.wowotech.net/x_project/gic_driver_porting.html)

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
