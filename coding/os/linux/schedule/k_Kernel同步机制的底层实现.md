原创 baron 人人极客社区
 _2022年03月01日 08:28_
## 原子操作

通常我们代码中的a = a + 1这样的一行语句，翻译成汇编后蕴含着3条指令:

`ldr x0, &a   add x0,x0,#1   str x0,&a   `

> 即
> 
> (1)从内存中读取a变量到X0寄存器
> (2)X0寄存器加1
> (3)将X0写入到内存a中

既然是3条指令，那么就有可能并发，也就意味着返回的结果可能不是预期的。

然后在linux kernel的操作系统中，提供访问原子变量的函数，用来解决上述问题。其中部分原子操作的API如下：
```cpp
atomic_read()
atomic_add_return(i,v)
atomic_add(i,v)
atomic_inc(v)
atomic_add_unless(v,a,u)
atomic_inc_not_zero(v)
atomic_sub_return(i,v)
atomic_sub_and_test(i,v)
atomic_sub(i,v)
atomic_dec(v)
atomic_cmpxchg(v,old,new)
```
那么操作系统(仅仅是软件而已)是如何保证原子操作的呢？（还是得靠硬件），硬件原理是什么呢？

以上的那些API函数，在底层调用的其实都是如下__lse_atomic_add_return##name宏的封装，这段代码中最核心的也就是ldadd指令了，这是armv8.1增加的LSE（Large System Extension）feature。
```cpp
(linux/arch/arm64/include/asm/atomic_lse.h)      static inline int __lse_atomic_add_return##name(int i, atomic_t *v) \   {         \    u32 tmp;       \            \    asm volatile(       \    __LSE_PREAMBLE       \    " ldadd" #mb " %w[i], %w[tmp], %[v]\n"   \    " add %w[i], %w[i], %w[tmp]"    \    : [i] "+r" (i), [v] "+Q" (v->counter), [tmp] "=&r" (tmp) \    : "r" (v)       \    : cl);        \            \    return i;       \   }   
```
那么系统如果没有LSE扩展呢，即armv8.0，其实现的原型如下所示，这段代码中最核心的也就是ldxr、stxr指令了。

```c
(linux/arch/arm64/include/asm/atomic_ll_sc.h)      static inline void __ll_sc_atomic_##op(int i, atomic_t *v)\   {         \    unsigned long tmp;      \    int result;       \            \    asm volatile("// atomic_" #op "\n"    \    __LL_SC_FALLBACK(      \   " prfm pstl1strm, %2\n"     \   "1: ldxr %w0, %2\n"      \   " " #asm_op " %w0, %w0, %w3\n"    \   " stxr %w1, %w0, %2\n"      \   " cbnz %w1, 1b\n")      \    : "=&r" (result), "=&r" (tmp), "+Q" (v->counter)  \    : __stringify(constraint) "r" (i));    \   } 
```

那么在armv8.0之前呢，如armv7是怎样实现的？如下所示， 这段代码中最核心的也就是ldrex、strex指令了。
```cpp
(linux/arch/arm/include/asm/atomic.h)      static inline void atomic_##op(int i, atomic_t *v)   \   {         \    unsigned long tmp;          int result;       \            \    prefetchw(&v->counter);      \    __asm__ __volatile__("@ atomic_" #op "\n"   \   "1: ldrex %0, [%3]\n"      \   " " #asm_op " %0, %0, %4\n"     \   " strex %1, %0, [%3]\n"      \   " teq %1, #0\n"      \   " bne 1b"       \    : "=&r" (result), "=&r" (tmp), "+Qo" (v->counter)  \    : "r" (&v->counter), "Ir" (i)     \    : "cc");       \   }   
```
### 总结：

在很早期，使用arm的exclusive机制来实现的原子操作，exclusive相关的指令也就是ldrex、strex了，但在armv8后，exclusive机制的指令发生了变化变成了ldxr、stxr。但是又由于在一个大系统中，处理器是非常多的，竞争也激烈，使用独占的存储和加载指令可能要多次尝试才能成功，性能也就变得很差，在armv8.1为了解决该问题，增加了ldadd等相关的原子操作指令。
## spinlock 自旋锁
### 早期spinlock的设计

早期的spinlock的设计是锁的拥有者加锁时将锁的值设置为1，释放锁时将锁的值设置为0，这样做的缺点是会出现 先来抢占锁的进程一直抢占不到锁，而后来的进程可能一来 就能获取到锁。导致这个原因的是先抢占的进程和后抢占的进程在抢占锁时并没有一个先后关系，最终就是离锁所在的内存最近的cpu节点就有更多的机会抢占锁，离锁所在内存远的节点可能一直抢占不到。
### 新版spinlock设计

为了解决这个spinlock的不公平问题，linux 2.6.25内核以后，spinlock采用了一种"FIFO ticket-based"算法的spinlock机制，可以很好的实现先来先抢占的思想。具体的做法如下：

1. spinlock的核心字段有owner和next，在初始时，owner=next=0
2. 当第一个进程抢占spinlock时，会在进程函数本地保存下next的值，也就是next=0，并将spinlock的next字段加1；
3. 当获取spinlock的进程的本地next和spinlock的owner相等时，该进程就获取到spinlock;
4. 由于第一个进程本地的next=0，并且spinlock的owner为0，所以第一个CPU获取到spinlock；
5. 接着当第二个进程抢占spinlock，此时spinlock的next值为1，保存到本地，然后将spinlock的next字段加1。而spinlock的owner字段依然为0，第二个进程的本地next 不等于spinlock的owner，所以一直自旋等待spinlock；
6. 第三个进程抢占spinlock，得到本地next值为2，然后将spinlock的next字段加1。此时spinlock的owner字段还是为0，所以第三个进程自旋等待。
7. 当第一个进程处理完临界区以后，就释放spinlock，执行的操作是将spinlock的owner字段加1；
8. 由于第二个进程和第三个进程都还在等待spinlock，他们会不停第获取spinlock的owner字段，并和自己本地的next值进行比较。当第二个进程发现自己的next值和spinlock的owner字段相等时（此时next == owner == 2），第二个进程就获取到spinlock。第三个进程的本地next值是3，和spinlock的owner字段不相等，所以继续等待；
9. 只有在第二个进程释放了spinlock，就会将spinlock的owner字段加1，第三个进程才有机会获取spinlock。

我在举个例子，如下：
![[Pasted image 20240923215552.png]]

> T1 : 进程1调用spin_lock，此时next=0, owner=0获得该锁，在arch_spin_lock()底层实现中，会next++
> 
> T2 : 进程2调用spin_lock，此时next=1, owner=0没有获得该锁，while(1)中调用wfe指令standby在那里，等待owner==next成立.
> 
> T3 : 进程3调用spin_lock，此时next=2, owner=0没有获得该锁，while(1)中调用wfe指令standby在那里，等待owner==next成立.
> 
> T4&T5 : 进程1调用spin_unlock，此时owner++，即owner=1，接着调用sev指令，让进程2和进程3退出standby状态，走while(1)流程，重新检查owner==next条件。此时进程2条件成立，进程3继续等待。进程2获得该锁，进程3继续等待。
### Linux Kernel中的SpinLock的实现
![[Pasted image 20240923215558.png]]

```c
(linux/include/linux/spinlock.h)      static __always_inline void spin_unlock(spinlock_t *lock)   {    raw_spin_unlock(&lock->rlock);   }         static __always_inline void spin_lock(spinlock_t *lock)   {    raw_spin_lock(&lock->rlock);   }   

(linux/include/linux/spinlock.h)      #define raw_spin_lock_irq(lock)  _raw_spin_lock_irq(lock)   #define raw_spin_lock_bh(lock)  _raw_spin_lock_bh(lock)   #define raw_spin_unlock(lock)  _raw_spin_unlock(lock)   #define raw_spin_unlock_irq(lock) _raw_spin_unlock_irq(lock)         #define raw_spin_lock(lock) _raw_spin_lock(lock)   

(linux/kernel/locking/spinlock.c)      #ifdef CONFIG_UNINLINE_SPIN_UNLOCK   void __lockfunc _raw_spin_unlock(raw_spinlock_t *lock)   {    __raw_spin_unlock(lock);   }   EXPORT_SYMBOL(_raw_spin_unlock);   #endif      #ifndef CONFIG_INLINE_SPIN_LOCK   void __lockfunc _raw_spin_lock(raw_spinlock_t *lock)   {    __raw_spin_lock(lock);   }   EXPORT_SYMBOL(_raw_spin_lock);   #endif 

(linux/include/linux/spinlock_api_smp.h)      static inline void __raw_spin_unlock(raw_spinlock_t *lock)   {    spin_release(&lock->dep_map, _RET_IP_);    do_raw_spin_unlock(lock);    preempt_enable();   }      static inline void __raw_spin_lock(raw_spinlock_t *lock)   {    preempt_disable();    spin_acquire(&lock->dep_map, 0, 0, _RET_IP_);    LOCK_CONTENDED(lock, do_raw_spin_trylock, do_raw_spin_lock);   }

(linux/include/linux/spinlock.h)      static inline void do_raw_spin_unlock(raw_spinlock_t *lock) __releases(lock)   {    mmiowb_spin_unlock();    arch_spin_unlock(&lock->raw_lock);    __release(lock);   }      static inline void do_raw_spin_lock(raw_spinlock_t *lock) __acquires(lock)   {    __acquire(lock);    arch_spin_lock(&lock->raw_lock);    mmiowb_spin_lock();   }   
```

对于arch_spin_lock()、arch_spin_unlock()的底层实现，不同的kernel版本也一直在变化。

对于kernel4.4这个版本，还是比较好理解的，最核心的也就是ldaxr、ldaxr独占指令 ，以及stlrh release指令

```c
(linux/arch/arm64/include/asm/spinlock.h)      static inline void arch_spin_lock(arch_spinlock_t *lock)   {    unsigned int tmp;    arch_spinlock_t lockval, newval;       asm volatile(    /* Atomically increment the next ticket. */    ARM64_LSE_ATOMIC_INSN(    /* LL/SC */   " prfm pstl1strm, %3\n"   "1: ldaxr %w0, %3\n"   " add %w1, %w0, %w5\n"   " stxr %w2, %w1, %3\n"   " cbnz %w2, 1b\n",    /* LSE atomics */   " mov %w2, %w5\n"   " ldadda %w2, %w0, %3\n"   " nop\n"   " nop\n"   " nop\n"    )       /* Did we get the lock? */   " eor %w1, %w0, %w0, ror #16\n"   " cbz %w1, 3f\n"    /*     * No: spin on the owner. Send a local event to avoid missing an     * unlock before the exclusive load.     */   " sevl\n"   "2: wfe\n"   " ldaxrh %w2, %4\n"   " eor %w1, %w2, %w0, lsr #16\n"   " cbnz %w1, 2b\n"    /* We got the lock. Critical section starts here. */   "3:"    : "=&r" (lockval), "=&r" (newval), "=&r" (tmp), "+Q" (*lock)    : "Q" (lock->owner), "I" (1 << TICKET_SHIFT)    : "memory");   }         static inline void arch_spin_unlock(arch_spinlock_t *lock)   {    unsigned long tmp;       asm volatile(ARM64_LSE_ATOMIC_INSN(    /* LL/SC */    " ldrh %w1, %0\n"    " add %w1, %w1, #1\n"    " stlrh %w1, %0",    /* LSE atomics */    " mov %w1, #1\n"    " nop\n"    " staddlh %w1, %0")    : "=Q" (lock->owner), "=&r" (tmp)    :    : "memory");   }
```


---

写留言

**留言 7**

- 小永
    
    2022年3月1日
    
    赞
    
    要是再spinlock的next+1操作改为原子操作，是不是可以解决无法独占的问题呢？
    
    人人极客社区
    
    作者2022年3月1日
    
    赞
    
    超出了我的认知![😂](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
    小永
    
    2022年3月1日
    
    赞
    
    探讨而已，大佬指正![[白眼]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
    人人极客社区
    
    作者2022年3月1日
    
    赞
    
    我也不懂
    
- 小白
    
    2022年3月1日
    
    赞
    
    牛批，干货
    
- 郭磊
    
    2022年3月1日
    
    赞
    
    在spinlock的实现中，会不会同时有两个进程将spinlock的next值+1？
    
    人人极客社区
    
    作者2022年3月1日
    
    赞
    
    有可能
    

已无更多数据

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/9sNwsXcN68pL55XIyzTrCHZTbIUdTibQcuzuCaYeGTXNMyn6ACmicUrpoDC0xZSap46XJ59sKysPg9Rg379f32cA/300?wx_fmt=png&wxfrom=18)

人人极客社区

1326

7

写留言

**留言 7**

- 小永
    
    2022年3月1日
    
    赞
    
    要是再spinlock的next+1操作改为原子操作，是不是可以解决无法独占的问题呢？
    
    人人极客社区
    
    作者2022年3月1日
    
    赞
    
    超出了我的认知![😂](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
    小永
    
    2022年3月1日
    
    赞
    
    探讨而已，大佬指正![[白眼]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
    人人极客社区
    
    作者2022年3月1日
    
    赞
    
    我也不懂
    
- 小白
    
    2022年3月1日
    
    赞
    
    牛批，干货
    
- 郭磊
    
    2022年3月1日
    
    赞
    
    在spinlock的实现中，会不会同时有两个进程将spinlock的next值+1？
    
    人人极客社区
    
    作者2022年3月1日
    
    赞
    
    有可能
    

已无更多数据