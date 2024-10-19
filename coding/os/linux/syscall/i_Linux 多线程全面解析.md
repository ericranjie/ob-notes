
马哥Linux运维  _2023年07月28日 23:28_

引入：

在传统的Unix模型中，当一个进程需要由另一个实体执行某件事时，该进程派生（fork）一个子进程，让子进程去进行处理。Unix下的大多数网络服务器程序都是这么编写的，即父进程接受连接，派生子进程，子进程处理与客户的交互。

虽然这种模型很多年来使用得很好，但是fork时有一些问题：

- fork是昂贵的。内存映像要从父进程拷贝到子进程，所有描述字要在子进程中复制等等。目前有的Unix实现使用一种叫做写时拷贝（copy－on－write）的技术，可避免父进程数据空间向子进程的拷贝。尽管有这种优化技术，fork仍然是昂贵的。

- fork子进程后，需要用进程间通信（IPC）在父子进程之间传递信息。Fork之前的信息容易传递，因为子进程从一开始就有父进程数据空间及所有描述字的拷贝。但是从子进程返回信息给父进程需要做更多的工作。

线程有助于解决这两个问题。线程有时被称为轻权进程（lightweight process），因为线程比进程“轻权”，一般来说，创建一个线程要比创建一个进程快10～100倍。

一个进程中的所有线程共享相同的全局内存，这使得线程很容易共享信息，但是这种简易性也带来了同步问题。

一个进程中的所有线程不仅共享全局变量，而且共享：进程指令、大多数数据、打开的文件（如描述字）、信号处理程序和信号处置、当前工作目录、用户ID和组ID。但是每个线程有自己的线程ID、寄存器集合（包括程序计数器和栈指针）、栈（用于存放局部变量和返回地址）、error、信号掩码、优先级。

在Linux中线程编程符合Posix.1标准，称为Pthreads。所有的pthread函数都以pthread_开头。在调用它们前均要包括pthread.h头文件，一个函数库libpthread实现。

# 1.线程基础介绍：

- 数据结构：

`pthread_t:线程的ID   pthread_attr_t:线程的属性   `

- 操作函数：

`pthread_create()：创建一个线程   pthread_exit()：终止当前线程   pthread_cancel()：中断另外一个线程的运行   pthread_join()：阻塞当前的线程，直到另外一个线程运行结束   pthread_attr_init()：初始化线程的属性   pthread_attr_setdetachstate()：设置脱离状态的属性（决定这个线程在终止时是否可以被结合）   pthread_attr_getdetachstate()：获取脱离状态的属性   pthread_attr_destroy()：删除线程的属性   pthread_kill()：向线程发送一个信号   `

- 同步函数：

`用于 mutex 和条件变量   pthread_mutex_init()初始化互斥锁   pthread_mutex_destroy()删除互斥锁   pthread_mutex_lock()：占有互斥锁（阻塞操作）   pthread_mutex_trylock()：试图占有互斥锁（不阻塞操作）。即，当互斥锁空闲时，将占有该锁；否则，立即返回。   pthread_mutex_unlock():释放互斥锁   pthread_cond_init()：初始化条件变量   pthread_cond_destroy()：销毁条件变量   pthread_cond_signal():唤醒第一个调用pthread_cond_wait()而进入睡眠的线程   pthread_cond_wait():等待条件变量的特殊条件发生   Thread-local storage（或者以Pthreads术语，称作线程特有数据）：   pthread_key_create():分配用于标识进程中线程特定数据的键   pthread_setspecific():为指定线程特定数据键设置线程特定绑定   pthread_getspecific():获取调用线程的键绑定，并将该绑定存储在 value 指向的位置中   pthread_key_delete():销毁现有线程特定数据键   pthread_attr_getschedparam();获取线程优先级   pthread_attr_setschedparam();设置线程优先级   `

## 2.概念：

线程的组成部分：

Thread ID  线程ID

Stack  栈

Policy  优先级

Signal mask  信号码

Errno  错误码

Thread-Specific Data 特殊数据

## 3.线程定义

1） pthread_t pthread_ID  ,用于标识一个线程，不能单纯看成整数，可能是结构体，与实现有关

2） pthread_equal函数用于比较两个pthread_t是否相等

`＃include <pthread.h>      int pthread_equal(pthread_t tid1,pthread_t tid2)   `

3）pthread_self函数用于获得本线程的thread id

`＃include <pthread.h>      pthread _t pthread_self(void);   `

## 4.线程的创建

1）  创建线程调用pthread_create函数：

`1 ＃include <pthread.h>   2     3 int pthread_create(   4        pthread_t*restrict tidp,   5        constpthread_attr_t*restrict attr,   6        void*(*start_rtn)(void*),void*restrict arg);   `

参数说明：

- pthread_t \*restrict tidp：返回最后创建出来的Thread的Thread ID

- const pthread_attr_t \*restrict attr：指定线程的Attributes，后面会讲道，现在可以用NULL

- void \*(\*start_rtn)(void \*)：指定线程函数指针，该函数返回一个void _，参数也为void_

- void \*restrict arg：传入给线程函数的参数

- 返回错误值。

一个进程中的每个线程都由一个线程ID（thread ID）标识，其数据类型是pthread_t（常常是unsigned int）。如果新的线程创建成功，其ID将通过tid指针返回。

每个线程都有很多属性：优先级、起始栈大小、是否应该是一个守护线程等等，当创建线程时，我们可通过初始化一个pthread_attr_t变量说明这些属性以覆盖缺省值。我们通常使用缺省值，在这种情况下，我们将attr参数说明为空指针。

最后，当创建一个线程时，我们要说明一个它将执行的函数。线程以调用该函数开始，然后或者显式地终止（调用pthread_exit）或者隐式地终止（让该函数返回）。函数的地址由func参数指定，该函数的调用参数是一个指针arg，如果我们需要多个调用参数，我们必须将它们打包成一个结构，然后将其地址当作唯一的参数传递给起始函数。

在func和arg的声明中，func函数取一个通用指针（void ＊）参数，并返回一个通用指针（void ＊），这就使得我们可以传递一个指针（指向任何我们想要指向的东西）给线程，由线程返回一个指针（同样指向任何我们想要指向的东西）。调用成功，返回0，出错时返回正Exxx值。

2）   pthread函数在出错的时候不会设置errno，而是直接返回错误值

3）  在Linux 系统下面，在老的内核中，由于Thread也被看作是一种特殊，可共享地址空间和资源的Process，因此在同一个Process中创建的不同 Thread具有不同的Process ID（调用getpid获得）。而在新的2.6内核之中，Linux采用了NPTL(Native POSIX Thread Library)线程模型（可以参考http://en.wikipedia.org/wiki/Native_POSIX_Thread_Library和http://www-128.ibm.com/developerworks/linux/library/l-threading.html?ca=dgr-lnxw07LinuxThreadsAndNPTL），在该线程模型下同一进程下不同线程调用getpid返回同一个PID。

4）  不能对创建的新线程和当前创建者线程的运行顺序作出任何假设

## 5.线程的退出

- exit, \_Exit, \_exit用于中止当前进程，而非线程

- 中止线程可以有三种方式：

  a.   在线程函数中return

  b.   被同一进程中的另外的线程Cancel掉

  c.   线程调用pthread_exit函数

- pthread_exit和pthread_join函数的用法：

  a.   线程A调用pthread_join(B, &rval_ptr)，被Block，进入Detached状态（如果已经进入Detached状态，则pthread_join函数返回EINVAL）。如果对B的结束代码不感兴趣，rval_ptr可以传NULL。

  b.   线程B调用pthread_exit(rval_ptr)，退出线程B，结束代码为rval_ptr。注意rval_ptr指向的内存的生命周期，不应该指向B的Stack中的数据。

  c.   线程A恢复运行，pthread_join函数调用结束，线程B的结束代码被保存到rval_ptr参数中去。如果线程B被Cancel，那么rval_ptr的值就是PTHREAD_CANCELLED。

两个函数原型如下：

`＃include <pthread.h>       void pthread_exit(void*rval_ptr);       int pthread_join(pthread_t thread,void**rval_ptr);   `

该函数等待一个线程终止。把线程和进程相比，pthread_creat类似于fork，而 pthread_join类似于waitpid。我们必须要等待线程的tid，很可惜，我们没有办法等待任意一个线程结束。如果status指针非空，线程的返回值（一个指向某个对象的指针）将存放在status指向的位置。

- 一个Thread可以要求另外一个Thread被Cancel，通过调用pthread_cancel函数：

`＃include <pthread.h>       void pthread_cancel(pthread_t tid)   `

该函数会使指定线程如同调用了pthread_exit(PTHREAD_CANCELLED)。不过，指定线程可以选择忽略或者进行自己的处理，在后面会讲到。此外，该函数不会导致Block，只是发送Cancel这个请求。

- 线程可以安排在它退出的时候，某些函数自动被调用，类似atexit()函数。需要调用如下函数：

`＃include <pthread.h>       void pthread_cleanup_push(void(*rtn)(void*),void*arg);   void pthread_cleanup_pop(int execute);   `

这两个函数维护一个函数指针的Stack，可以把函数指针和函数参数值push/pop。执行的顺序则是从栈顶到栈底，也就是和push的顺序相反。

在下面情况下pthread_cleanup_push所指定的thread cleanup handlers会被调用：

a.   调用pthread_exit

b.   相应cancel请求

c.   以非0参数调用pthread_cleanup_pop()。（如果以0调用pthread_cleanup_pop()，那么handler不会被调用

有一个比较怪异的要求是，由于这两个函数可能由宏的方式来实现，因此这两个函数的调用必须得是在同一个Scope之中，并且配对，因为在pthread_cleanup_push的实现中可能有一个{，而 pthread_cleanup_pop可能有一个}。

因此，一般情况下，这两个函数是用于处理意外情况用的，举例如下：

`void*thread_func(void*arg)   {       pthread_cleanup_push(cleanup,“handler”)           // do something           Pthread_cleanup_pop(0);       return((void*)0)；   }   `

- 进程函数和线程函数的相关性：

|**Process Primitive**|**Thread Primitive**|**Description**|
|---|---|---|
|fork|pthread_create|创建新的控制流|
|exit|pthread_exit|退出已有的控制流|
|waitpid|pthread_join|等待控制流并获得结束代码|
|atexit|pthread_cleanup_push|注册在控制流退出时候被调用的函数|
|getpid|pthread_self|获得控制流的id|
|abort|pthread_cancel|请求非正常退出|

- 缺省情况下，一个线程A的结束状态被保存下来直到pthread_join为该线程被调用过，也就是说即使线程A已经结束，只要没有线程B调用 pthread_join(A)，A的退出状态则一直被保存。而当线程处于Detached状态之时，当线程退出的时候，其资源可以立刻被回收，那么这个退出状态也丢失了。在这个状态下，无法为该线程调用pthread_join函数。我们可以通过调用pthread_detach函数来使指定线程进入 Detach状态：

`＃include <pthread.h>   int pthread_detach(pthread_t tid);   `

通过修改调用pthread_create函数的attr参数，我们可以指定一个线程在创建之后立刻就进入Detached状态

## 6.线程同步

- 互斥量：Mutex

  各个现成向同一个文件顺序写入数据，最后得到的结果是不可想象的。所以用互斥锁来保证一段时间内只有一个线程在执行一段代码。

  a.   用于互斥访问

  b.   类型：pthread_mutex_t，必须被初始化为PTHREAD_MUTEX_INITIALIZER

（用于静态分配的mutex，等价于 pthread_mutex_init(…, NULL)）或者调用pthread_mutex_init。Mutex也应该用pthread_mutex_destroy来销毁。这两个函数原型如下：

`＃include <pthread.h>       int pthread_mutex_init(          pthread_mutex_t*restrict mutex,          constpthread_mutexattr_t*restrict attr)       int pthread_mutex_destroy(pthread_mutex_t*mutex);   `

c.   pthread_mutex_lock 用于Lock Mutex，如果Mutex已经被Lock，该函数调用会Block直到Mutex被Unlock，然后该函数会Lock Mutex并返回。pthread_mutex_trylock类似，只是当Mutex被Lock的时候不会Block，而是返回一个错误值EBUSY。

pthread_mutex_unlock则是unlock一个mutex。这三个函数原型如下：

`＃include <pthread.h>       int pthread_mutex_lock(pthread_mutex_t*mutex);       int pthread_mutex_trylock(pthread_mutex_t*mutex);       int pthread_mutex_unlock(pthread_mutex_t*mutex);   `

d.   举例说明

`void reader_function (void);   void writer_function (void);   char buffer;   int buffer_has_item=0;   pthread_mutex_t mutex;   struct timespec delay;   void main (void)   {   pthread_t reader;   /* 定义延迟时间*/   delay.tv_sec =2;   delay.tv_nec =0;   /* 用默认属性初始化一个互斥锁对象*/   pthread_mutex_init (&mutex,NULL);   pthread_create(&reader, pthread_attr_default,(void*)&reader_function), NULL);   writer_function();   }   void writer_function (void){   while(1){   /* 锁定互斥锁*/   pthread_mutex_lock (&mutex);   if(buffer_has_item==0){   buffer=make_new_item();   buffer_has_item=1;   }   /* 打开互斥锁*/   pthread_mutex_unlock(&mutex);   pthread_delay_np(&delay);   }   }   void reader_function(void){   while(1){   pthread_mutex_lock(&mutex);   if(buffer_has_item==1){   consume_item(buffer);   buffer_has_item=0;   }   pthread_mutex_unlock(&mutex);   pthread_delay_np(&delay);   }   }   `

需要注意的是在使用互斥锁的过程中很有可能会出现死锁：两个线程试图同时占用两个资源，并按不同的次序锁定相应的互斥锁，例如两个线程都需要锁定互斥锁1和互斥锁2，a线程先锁定互斥锁1，b 线程先锁定互斥锁2，这时就出现了死锁。

此时我们可以使用函数 pthread_mutex_trylock，它是函数pthread_mutex_lock的非阻塞版本，当它发现死锁不可避免时，它会返回相应的信息，程序员可以针对死锁做出相应的处理。另外不同的互斥锁类型对死锁的处理不一样，但最主要的还是要程序员自己在程序设计注意这一点

- 读写锁：Reader-Writer Locks

  a.   多个线程可以同时获得读锁(Reader-Writer lock in read mode)，但是只有一个线程能够获得写锁(Reader-writer lock in write mode)

  b.   读写锁有三种状态

  i.  一个或者多个线程获得读锁，其他线程无法获得写锁

  ii.  一个线程获得写锁，其他线程无法获得读锁

  iii.  没有线程获得此读写锁

  c.   类型为pthread_rwlock_t

  d.   创建和关闭方法如下：

`＃include <pthread.h>       int pthread_rwlock_init(          pthread_rwlock_t*restrict rwlock,          constpthread_rwlockattr_t*restrict attr)       int pthread_rwlock_destroy(pthread_rwlock_t*rwlock);   `

e.   获得读写锁的方法如下：

`＃include <pthread.h>       int pthread_rwlock_rdlock(pthread_rwlock_t*rwlock);       int pthread_rwlock_wrlock(pthread_rwlock_t*rwlock);       int pthread_rwlock_unlock(pthread_rwlock_t*rwlock);       int pthread_rwlock_tryrdlock(pthread_rwlock_t*rwlock);       int pthread_rwlock_trywrlock(pthread_rwlock_t*rwlock);`

pthread_rwlock_rdlock：获得读锁

pthread_rwlock_wrlock：获得写锁

pthread_rwlock_unlock：释放锁，不管是读锁还是写锁都是调用此函数

注意具体实现可能对同时获得读锁的线程个数有限制，所以在调用 pthread_rwlock_rdlock的时候需要检查错误值，而另外两个pthread_rwlock_wrlock和 pthread_rwlock_unlock则一般不用检查，如果我们代码写的正确的话。

- Conditional Variable：条件变量

  互斥锁一个明显的缺点是它只有两种状态：锁定和非锁定。而条件变量通过允许线程阻塞和等待另一个线程发送信号的方法弥补了互斥锁的不足，它常和互斥锁一起使用。使用时，条件变量被用来阻塞一个线程，当条件不满足时，线程往往解开相应的互斥锁并等待条件发生变化。一旦其它的某个线程改变了条件变量，它将通知相应的条件变量唤醒一个或多个正被此条件变量阻塞的线程。这些线程将重新锁定互斥锁并重新测试条件是否满足。一般说来，条件变量被用来进行线程间的同步。

  a.   条件必须被Mutex保护起来

  b.   类型为：pthread_cond_t，必须被初始化为PTHREAD_COND_INITIALIZER（用于静态分配的条件，等价于pthread_cond_init(…, NULL)）或者调用pthread_cond_init

`＃include <pthread.h>       int pthread_cond_init(          pthread_cond_t*restrict cond,          constpthread_condxattr_t*restrict attr)       int pthread_cond_destroy(pthread_cond_t*cond);   `

c.   pthread_cond_wait 函数用于等待条件发生（=true）。pthread_cond_timedwait类似，只是当等待超时的时候返回一个错误值ETIMEDOUT。超时的时间用timespec结构指定。此外，两个函数都需要传入一个Mutex用于保护条件

`＃include <pthread.h>       int pthread_cond_wait(          pthread_cond_t*restrict cond,          pthread_mutex_t*restrict mutex);       int pthread_cond_timedwait(          pthread_cond_t*restrict cond,          pthread_mutex_t*restrict mutex,          conststruct timespec *restrict timeout);   `

一个简单例子：

`pthread_mutex_t count_lock;   pthread_cond_t count_nonzero;   unsigned count;   decrement_count (){   pthread_mutex_lock (&count_lock);   while(count==0)   pthread_cond_wait(&count_nonzero,&count_lock);   count=count -1;   pthread_mutex_unlock (&count_lock);   }   increment_count(){   pthread_mutex_lock(&count_lock);   if(count==0)   pthread_cond_signal(&count_nonzero);   count=count+1;   pthread_mutex_unlock(&count_lock);   }   `

count 值为0时， decrement函数在pthread_cond_wait处被阻塞，并打开互斥锁count_lock。此时，当调用到函数 increment_count时，pthread_cond_signal（）函数改变条件变量，告知decrement_count（）停止阻塞。

d.   timespec结构定义如下：

`struct timespec {          time_t tv_sec;       /* seconds */          long   tv_nsec;      /* nanoseconds */   };   `

注意timespec的时间是绝对时间而非相对时间，因此需要先调用gettimeofday函数获得当前时间，再转换成timespec结构，加上偏移量。

e.   有两个函数用于通知线程条件被满足（=true）：

`＃include <pthread.h>       int pthread_cond_signal(pthread_cond_t*cond);       int pthread_cond_broadcast(pthread_cond_t*cond);   `

两者的区别是前者会唤醒单个线程，而后者会唤醒多个线程。

## 7.线程属性

- 线程属性设置

我们用pthread_create函数创建一个线程，在这个线程中，我们使用默认参数，即将该函数的第二个参数设为NULL。的确，对大多数程序来说，使用默认属性就够了，但我们还是有必要来了解一下线程的有关属性。

属性结构为pthread_attr_t，它同样在头文件pthread.h中定义，属性值不能直接设置，须使用相关函数进行操作，初始化的函数为pthread_attr_init，这个函数必须在pthread_create函数之前调用。属性对象主要包括是否绑定、是否分离、

堆栈地址、堆栈大小、优先级。默认的属性为非绑定、非分离、缺省的堆栈、与父进程同样级别的优先级。

- 绑定

关于线程的绑定，牵涉到另外一个概念：轻进程（LWP：Light Weight Process）。轻进程可以理解为内核线程，它位于用户层和系统层之间。系统对线程资源的分配、对线程的控制是通过轻进程来实现的，一个轻进程可以控制一个或多个线程。

默认状况下，启动多少轻进程、哪些轻进程来控制哪些线程是由系统来控制的，这种状况即称为非绑定的。绑定状况下，则顾名思义，即某个线程固定的"绑"在一个轻进程之上。

被绑定的线程具有较高的响应速度，这是因为CPU时间片的调度是面向轻进程的，绑定的线程可以保证在需要的时候它总有一个轻进程可用。通过设置被绑定的轻进程的优先级和调度级可以使得绑定的线程满足诸如实时反应之类的要求。

设置线程绑定状态的函数为 pthread_attr_setscope，它有两个参数，第一个是指向属性结构的指针，第二个是绑定类型，它有两个取值：PTHREAD_SCOPE_SYSTEM（绑定的）和PTHREAD_SCOPE_PROCESS（非绑定的）。下面的代码即创建了一个绑定的线程。

`＃include <pthread.h>   pthread_attr_t attr;   pthread_t tid;   /*初始化属性值，均设为默认值*/   pthread_attr_init(&attr);   pthread_attr_setscope(&attr, PTHREAD_SCOPE_SYSTEM);   pthread_create(&tid,&attr,(void*) my_function, NULL);   `

- 线程分离状态

  线程的分离状态决定一个线程以什么样的方式来终止自己。非分离的线程终止时，其线程ID和退出状态将保留，直到另外一个线程调用 pthread_join.分离的线程在当它终止时，所有的资源将释放，我们不能等待它终止。

  设置线程分离状态的函数为 pthread_attr_setdetachstate（pthread_attr_t \*attr, int detachstate）

第二个参数可选为PTHREAD_CREATE_DETACHED（分离线程）或 PTHREAD \_CREATE_JOINABLE（非分离线程）。

这里要注意的一点是，如果设置一个线程为分离线程，而这个线程运行又非常快，它很可能在 pthread_create函数返回之前就终止了，它终止以后就可能将线程号和系统资源移交给其他的线程使用，这样调用pthread_create的线程就得到了错误的线程号。

要避免这种情况可以采取一定的同步措施，最简单的方法之一是可以在被创建的线程里调用 pthread_cond_timewait函数，让这个线程等待一会儿，留出足够的时间让函数pthread_create返回。设置一段等待时间，是在多线程编程里常用的方法。

- 4．优先级

它存放在结构sched_param中。用函数pthread_attr_getschedparam和函数 pthread_attr_setschedparam进行存放，一般说来，我们总是先取优先级，对取得的值修改后再存放回去。下面即是一段简单的例子。

`＃include <pthread.h>   ＃include <sched.h>   pthread_attr_t attr;pthread_t tid;   sched_param param;   int newprio=20;   /*初始化属性*/   pthread_attr_init(&attr);   /*设置优先级*/   pthread_attr_getschedparam(&attr,&param);    param.sched_priority=newprio;   pthread_attr_setschedparam(&attr,&param);   pthread_create(&tid,&attr,(void*)myfunction, myarg);`

_链接：**************https://www.cnblogs.com/Stultz-Lee/p/6702922.html**************_

_（版权归原作者所有，侵删）_

![Image](https://mmbiz.qpic.cn/mmbiz_png/2rMyvdWluHv8eFkGO2m6ia8T0B0GjM3YojG4TDD6v9CBrE5WKqNwADDsjZzfmbIBsFa6avHkPBF6L3LmfF3GG2g/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1&tp=webp)

​

![](https://mp.weixin.qq.com/rr?timestamp=1726626653&src=11&ver=1&signature=LnOQdtBhNGxyujuaO0LtsWR*0gdo880uatISbQUxeWeSDGbeKgydcHI49EsbZJeKVp*27y4gd4c1IJV0t7sPyI22OYzQfpxkuSXwuvxgCpY=)

Scan to Follow
