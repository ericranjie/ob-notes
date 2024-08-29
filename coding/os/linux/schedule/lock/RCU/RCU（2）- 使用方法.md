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

作者：[沙漠之狐](http://www.wowotech.net/author/535) 发布于：2019-5-24 19:36 分类：[内核同步机制](http://www.wowotech.net/sort/kernel_synchronization)

 

作者简介：余华兵，在网络通信行业工作十多年，负责IPv4协议栈、IPv6协议栈和Linux内核。在工作中看着2.6版本的专业书籍维护3.x和4.x版本的Linux内核，感觉不方便，于是自己分析4.x版本的Linux内核整理出一本书，书名叫《Linux内核深度解析》，2019年5月出版，希望对同行有帮助。

1．经典RCU

如果不关心使用的RCU是不可抢占RCU还是可抢占RCU，应该使用经典RCU的编程接口。最初的经典RCU是不可抢占RCU，后来实现了可抢占RCU，经典RCU的意思发生了变化：如果内核编译了可抢占RCU，那么经典RCU的编程接口被实现为可抢占RCU，否则被实现为不可抢占RCU。

读者使用函数rcu_read_lock()标记进入读端临界区，使用函数rcu_read_unlock()标记退出读端临界区。读端临界区可以嵌套。

在读端临界区里面应该使用宏rcu_dereference(p)访问指针，这个宏封装了数据依赖屏障，即只有阿尔法处理器需要的读内存屏障。

写者可以使用下面4个函数。

（1）使用函数synchronize_rcu()等待宽限期结束，即所有读者退出读端临界区，然后写者执行下一步操作。这个函数可能睡眠。

（2）使用函数synchronize_rcu_expedited()等待宽限期结束。和函数synchronize_rcu()的区别是：该函数会向其他处理器发送处理器间中断（Inter-Processor Interrupt，IPI）请求，强制宽限期快速结束。我们把强制快速结束的宽限期称为加速宽限期（expedited grace period），把没有强制快速结束的宽限期称为正常宽限期（normal grace period）。

（3）使用函数call_rcu()注册延后执行的回调函数，把回调函数添加到RCU回调函数链表中，立即返回，不会阻塞。函数原型如下：

void call_rcu(struct rcu_head *head, rcu_callback_t func);

struct callback_head {

     struct callback_head *next;

     void (*func)(struct callback_head *head);

} __attribute__((aligned(sizeof(void *))));

#define rcu_head callback_head

typedef void (*rcu_callback_t)(struct rcu_head *head);

（4）使用函数rcu_barrier()等待使用call_rcu注册的所有回调函数执行完。这个函数可能睡眠。

现在举例说明使用方法，假设链表节点和头节点如下：

typedef struct {

    struct list_head link;

    struct rcu_head rcu;

    int key;

    int val;

} test_entry;

struct list_head test_head;

成员“struct rcu_head rcu”：调用函数call_rcu把回调函数添加到RCU回调函数链表的时候需要使用。

读者访问链表的方法如下：

int test_read(int key, int *val_ptr)

{

     test_entry *entry;

     int found = 0;

     **rcu****_read****_lock();**

     **list****_for****_each****_entry****_rcu(entry,** **&test****_head,** **link)** **{**

           if (entry->key == key) {

                *val_ptr = entry->val;

                found = 1;

                break;

           }

     }

     **rcu****_read****_unlock();**

     return found;

}

如果只有一个写者，写者不需要使用锁，添加、更新和删除3种操作的实现方法如下。

（1）写者添加一个节点到链表尾部。

void test_add_node(test_entry *entry)

{

     **list****_add****_tail****_rcu(&entry->link,** **&test****_head);**

}

（2）写者更新一个节点。

更新的过程是：首先把旧节点复制更新，然后使用新节点替换旧节点，最后使用函数call_rcu注册回调函数，延后释放旧节点。

int test_update_node(int key, int new_val)

{

     test_entry *entry, *new_entry;

     int ret;

     ret = -ENOENT;

     list_for_each_entry(entry, &test_head, link) {

           if (entry->key == key) {

                new_entry = kmalloc(sizeof(test_entry), GFP_ATOMIC);

                if (new_entry == NULL) {

                     ret = -ENOMEM;

                     break;

                }

                ***new****_entry** **=** ***entry;**

                **new****_entry->val** **=** **new****_val;**

                **list****_replace****_rcu(&entry->list,** **&new****_entry->list);**

                **call****_rcu(&entry->rcu,** **test****_free****_node);**

                ret = 0;

                break;

           } 

     }

     return ret;

}

**static** **void** **test****_free****_node(struct** **rcu****_head** ***head)**

**{**

     **test****_entry** ***entry** **=** **container****_of(head,** **test****_entry,** **rcu);**

     **kfree(entry);**

**}**

（3）写者删除一个节点的第一种实现方法是：首先把节点从链表中删除，然后使用函数call_rcu注册回调函数，延后释放节点。

int test_del_node(int key)

{

     test_entry *entry;

     int found = 0;

     list_for_each_entry(entry, &test_head, link) {

           if (entry->key == key) {

                **list****_del****_rcu(&entry->link);**

                **call****_rcu(&entry->rcu,** **test****_free****_node);**

                found = 1;

                break; 

           } 

     }

     return found;

}

**static** **void** **test****_free****_node(struct** **rcu****_head** ***head)**

**{**

     **test****_entry** ***entry** **=** **container****_of(head,** **test****_entry,** **rcu);**

     **kfree(entry);**

**}**

（4）写者删除一个节点的第二种实现方法是：首先把节点从链表中删除，然后使用函数synchronize_rcu()等待宽限期结束，最后释放节点。

int test_del_node(int key)

{

     test_entry *entry;

     int found = 0;

     list_for_each_entry(entry, &test_head, link) {

           if (entry->key == key) {

                **list****_del****_rcu(&entry->link);**

                **synchronize****_rcu();**

                **kfree(entry);**

                found = 1;

                break; 

           } 

     }

     return found;

}

如果有多个写者，那么写者之间必须使用锁互斥，添加、更新和删除3种操作的实现方法如下。

（1）写者添加一个节点到链表尾部。假设使用自旋锁“test_lock”保护链表。

void test_add_node(test_entry *entry)

{

     **spin****_lock(&test****_lock);**

     **list****_add****_tail****_rcu(&entry->link,** **&test****_head);**

     **spin****_unlock(&test****_lock);**

}

（2）写者更新一个节点。

int test_update_node(int key, int new_val)

{

     test_entry *entry, *new_entry;

     int ret;

     ret = -ENOENT;

     **spin****_lock(&test****_lock);**

     list_for_each_entry(entry, &test_head, link) {

           if (entry->key == key) {

                new_entry = kmalloc(sizeof(test_entry), GFP_ATOMIC);

                if (new_entry == NULL) {

                     ret = -ENOMEM;

                     break;

                }

                ***new****_entry** **=** ***entry;**

                **new****_entry->val** **=** **new****_val;**

                **list****_replace****_rcu(&entry->list,** **&new****_entry->list);**

                **call****_rcu(&entry->rcu,** **test****_free****_node);**

                ret = 0;

                break;

           } 

     }

     **spin****_unlock(&test****_lock);**

     return ret;

}

**static** **void** **test****_free****_node(struct** **rcu****_head** ***head)**

**{**

     **test****_entry** ***entry** **=** **container****_of(head,** **test****_entry,** **rcu);**

     **kfree(entry);**

**}**

（3）写者删除一个节点。

int test_del_node(int key)

{

     test_entry *entry;

     int found = 0;

     **spin****_lock(&test****_lock);**

     list_for_each_entry(entry, &test_head, link) {

           if (entry->key == key) {

                **list****_del****_rcu(&entry->link);**

                **call****_rcu(&entry->rcu,** **test****_free****_node);**

                found = 1;

                break; 

           } 

     }

     **spin****_unlock(&test****_lock);**

     return found;

}

**static** **void** **test****_free****_node(struct** **rcu****_head** ***head)**

**{**

     **test****_entry** ***entry** **=** **container****_of(head,** **test****_entry,** **rcu);**

     **kfree(entry);**

**}**

2．不可抢占RCU

如果我们的需求是“不管内核是否编译了可抢占RCU，都要使用不可抢占RCU”，那么应该使用不可抢占RCU的专用编程接口。

读者使用函数rcu_read_lock_sched()标记进入读端临界区，使用函数rcu_read_unlock_ sched()标记退出读端临界区。读端临界区可以嵌套。

在读端临界区里面应该使用宏rcu_dereference_sched(p)访问指针，这个宏封装了数据依赖屏障，即只有阿尔法处理器需要的读内存屏障。

写者可以使用下面4个函数。

（1）使用函数synchronize_sched()等待宽限期结束，即所有读者退出读端临界区，然后写者执行下一步操作。这个函数可能睡眠。

（2）使用函数synchronize_sched_expedited()等待宽限期结束。和函数synchronize_sched()的区别是：该函数会向其他处理器发送处理器间中断请求，强制宽限期快速结束。

（3）使用函数call_rcu_sched()注册延后执行的回调函数，把回调函数添加到RCU回调函数链表中，立即返回，不会阻塞。

（4）使用函数rcu_barrier_sched()等待使用call_rcu_sched注册的所有回调函数执行完。这个函数可能睡眠。

3．加速版不可抢占RCU

加速版不可抢占RCU在软中断很多的情况下可以缩短宽限期。

读者使用函数rcu_read_lock_bh()标记进入读端临界区，使用函数rcu_read_unlock_bh()标记退出读端临界区。读端临界区可以嵌套。

在读端临界区里面应该使用宏rcu_dereference_bh(p)访问指针，这个宏封装了数据依赖屏障，即只有阿尔法处理器需要的读内存屏障。

写者可以使用下面4个函数。

（1）使用函数synchronize_rcu_bh()等待宽限期结束，即所有读者退出读端临界区，然后写者执行下一步操作。这个函数可能睡眠。

（2）使用函数synchronize_rcu_bh_expedited()等待宽限期结束。和函数synchronize_rcu_ bh()的区别是：该函数会向其他处理器发送处理器间中断请求，强制宽限期快速结束。

（3）使用函数call_rcu_bh注册延后执行的回调函数，把回调函数添加到RCU回调函数链表中，立即返回，不会阻塞。

（4）使用函数rcu_barrier_bh()等待使用call_rcu_bh注册的所有回调函数执行完。这个函数可能睡眠。

4．链表操作的RCU版本

RCU最常见的使用场合是保护大多数时候读的双向链表。内核实现了链表操作的RCU版本，这些操作封装了内存屏障。

内核实现了4种双向链表。

（1）链表“list_head”。

（2）链表“hlist”：和链表“list_head”相比，优势是头节点只有一个指针，节省内存。

（3）链表“hlist_nulls”。hlist_nulls是hlist的变体，区别是：链表hlist的结束符号是一个空指针；链表hlist_nulls的结束符号是“ (1UL | (((long)value) << 1)) ”，最低位为1，value是嵌入的值，比如散列桶的索引。

（4）链表“hlist_bl”。hlist_bl是hlist的变体，链表头节点的最低位作为基于位的自旋锁保护链表。

链表“list_head”常用的操作如下。

（1）list_for_each_entry_rcu(pos, head, member)

遍历链表，这个宏封装了只有阿尔法处理器需要的数据依赖屏障。

（2）void list_add_rcu(struct list_head *new, struct list_head *head)

把节点添加到链表首部。

（3）void list_add_tail_rcu(struct list_head *new, struct list_head *head)

把节点添加到链表尾部。

（4）void list_del_rcu(struct list_head *entry)

把节点从链表中删除。

（5）void list_replace_rcu(struct list_head *old, struct list_head *new)

用新节点替换旧节点。

链表“hlist”常用的操作如下。

（1）hlist_for_each_entry_rcu(pos, head, member)

遍历链表。

（2）void hlist_add_head_rcu(struct hlist_node *n, struct hlist_head *h)

把节点添加到链表首部。

（3）void hlist_add_tail_rcu(struct hlist_node *n, struct hlist_head *h)

把节点添加到链表尾部。

（4）void hlist_del_rcu(struct hlist_node *n)

把节点从链表中删除。

（5）void hlist_replace_rcu(struct hlist_node *old, struct hlist_node *new)

用新节点替换旧节点。

链表“hlist_nulls”常用的操作如下。

（1）hlist_nulls_for_each_entry_rcu(tpos, pos, head, member)

遍历链表。

（2）void hlist_nulls_add_head_rcu(struct hlist_nulls_node *n, struct hlist_nulls_head *h)

把节点添加到链表首部。

（3）void hlist_nulls_add_tail_rcu(struct hlist_nulls_node *n, struct hlist_nulls_head *h)

把节点添加到链表尾部。

（4）void hlist_nulls_del_rcu(struct hlist_nulls_node *n)

把节点从链表中删除。

链表“hlist_bl”常用的操作如下。

（1）hlist_bl_for_each_entry_rcu(tpos, pos, head, member)

遍历链表。

（2）void hlist_bl_add_head_rcu(struct hlist_bl_node *n, struct hlist_bl_head *h)

把节点添加到链表首部。

（3）void hlist_bl_del_rcu(struct hlist_bl_node *n)

把节点从链表中删除。

5．slab缓存支持RCU

创建slab缓存的时候，可以使用标志SLAB_TYPESAFE_BY_RCU（旧的名称是SLAB_ DESTROY_BY_RCU），延迟释放slab页到RCU宽限期结束，例如：

struct kmem_cache *anon_vma_cachep;

anon_vma_cachep = kmem_cache_create("anon_vma", sizeof(struct anon_vma),

                   0, **SLAB****_TYPESAFE****_BY****_RCU**|SLAB_PANIC|SLAB_ACCOUNT,

                   anon_vma_ctor);

注意：标志SLAB_TYPESAFE_BY_RCU只会延迟释放slab页到RCU宽限期结束，但是不会延迟对象的释放。当调用函数kmem_cache_free()释放对象时，对象的内存被释放了，可能立即被分配给另一个对象。如果使用散列表，新的对象可能加入不同的散列桶。所以查找对象的时候一定要小心。

针对使用函数kmalloc()从通用slab缓存分配的对象，提供了函数“kfree_rcu(ptr, rcu_ head)”来延迟释放对象到RCU宽限期结束，参数rcu_head是指针ptr指向的结构体里面类型为“struct rcu_head”的成员的名称。例如：

typedef struct {

    struct list_head link;

    struct rcu_head rcu;

    int key;

    int val;

} test_entry;

test_entry *p;

p = kmalloc(sizeof(test_entry), GFP_KERNEL);

…

kfree_rcu(p, rcu);

举例说明，假设对象从设置了标志SLAB_TYPESAFE_BY_RCU的slab缓存分配内存，对象加入散列表，散列表如下：

struct hlist_nulls_head table[TABLE_SIZE];

初始化散列桶的时候，把散列桶索引嵌入到链表结束符号中，其代码如下：

for (i = 0; i < TABLE_SIZE; i++) {

     INIT_HLIST_NULLS_HEAD(&table[i], i);

}

查找对象的算法如下：

1      head = &table[slot];

2      rcu_read_lock();

3   begin:

4      hlist_nulls_for_each_entry_rcu(obj, node, head, hash_node) {

5         if (obj->key == key) {

6            if (!atomic_inc_not_zero(obj->refcnt)) {

7               goto begin;

8            }

9   

10           if (obj->key != key) {

11               put_ref(obj);

12               goto begin;

13            }

14            goto out;

15     } 

16   

17     if (get_nulls_value(node) != slot) {

18        goto begin;

19     }

20     obj = NULL;

21   

22   out:

23      rcu_read_unlock();

第6～8行代码，找到对象以后，如果引用计数不是0，把引用计数加一；如果引用计数是0，表示对象已经被释放，应该在散列桶中重新查找对象。

第10～13行代码，把引用计数加1以后，需要重新比较关键字。如果关键字不同，说明对象的内存被释放以后立即分配给新的对象，应该在散列桶中重新查找对象。

第17～19行代码，如果遍历完一个散列桶，没有找到对象，那么需要比较链表结束符号中嵌入的散列桶索引。如果不同，说明对象的内存被释放以后立即分配给新的对象，新对象的散列值不同，加入了不同的散列桶。需要在散列桶中重新查找对象。

插入对象的算法如下：

obj = kmem_cache_alloc(cachep);

if (obj == NULL) {

     return -ENOMEM;

}

obj->key = key;

atomic_set(&obj->refcnt, 1);

lock_chain(); /* 通常是spin_lock() */

hlist_nulls_add_head_rcu(&obj->hash_node, &table[slot]);

unlock_chain(); /* 通常是spin_unlock() */

删除对象的算法如下：

if (atomic_dec_and_test(obj->refcnt)) {

    lock_chain(); /* 通常是spin_lock() */

    hlist_nulls_del_rcu(&obj->hash_node);

    unlock_chain(); /* 通常是spin_unlock() */

    kmem_cache_free(cachep, obj);

}

标签: [无锁编程](http://www.wowotech.net/tag/%E6%97%A0%E9%94%81%E7%BC%96%E7%A8%8B)

[![](http://www.wowotech.net/content/uploadfile/201605/ef3e1463542768.png)](http://www.wowotech.net/support_us.html)

« [以太网驱动的流程浅析(一)-Ifconfig主要流程](http://www.wowotech.net/linux_kenrel/464.html) | [RCU（1）- 概述](http://www.wowotech.net/kernel_synchronization/461.html)»

**评论：**

**坚强的小孩**  
2023-12-06 21:04

写的真好，干货满满

[回复](http://www.wowotech.net/kernel_synchronization/462.html#comment-8849)

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
    
    - [Linux调度器：进程优先级](http://www.wowotech.net/process_management/process-priority.html)
    - [X-022-OTHERS-git操作记录之合并远端分支的更新](http://www.wowotech.net/x_project/u_boot_merge_denx.html)
    - [KASLR](http://www.wowotech.net/memory_management/441.html)
    - [SDRAM Internals](http://www.wowotech.net/basic_tech/sdram-internals.html)
    - [Linux DMA Engine framework(2)_功能介绍及解接口分析](http://www.wowotech.net/linux_kenrel/dma_engine_api.html)
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