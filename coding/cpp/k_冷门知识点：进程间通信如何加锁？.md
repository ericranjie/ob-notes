原创 程序喵大人 程序喵大人
 _2022年02月07日 08:59_

这是[每日一题]栏目的第三题，如图：

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/JeibBY5FJRBHBFpA2wMy79B3fibuVMFFgHicqLRRBOlxCMY7yjLTEZsRE94gibT99jKAtm1qmgicI81LekqwOEy9VRg/640?wx_fmt=jpeg&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

关于进程间的通信方式估计大多数人都知道，这也是常见的面试八股文之一。
个人认为这种面试题没什么意义，无非就是答几个关键词而已，更深入的可能面试官和面试者都不太了解。
关于进程间通信方式和优缺点我之前在【这篇文章】中有过介绍，感兴趣的可以移步去看哈。
进程间通信有一种[共享内存]方式，大家有没有想过，这种通信方式中如何解决数据竞争问题？
我们可能自然而然的就会想到用锁。但我们平时使用的锁都是用于解决线程间数据竞争问题，貌似没有看到过它用在进程中，那怎么办？
我找到了两种方法，信号量和互斥锁。
直接给大家贴代码吧，首先是信号量方式：

```c
 #include <fcntl.h> 
 #include <pthread.h> 
 #include <semaphore.h> 
 #include <stdio.h> 
 #include <stdlib.h> 
 #include <sys/mman.h> 
 #include <sys/stat.h> 
 #include <sys/types.h> 
 #include <sys/wait.h> 
 #include <unistd.h>  
 constexpr int kMappingSize = 4096;  void sem() {     const char* mapname = "/mapname";     int mapfd = shm_open(mapname, O_RDWR | O_CREAT, S_IRUSR | S_IWUSR);      MEOW_DEFER {         if (mapfd > 0) {             close(mapfd);             mapfd = 0;         }         shm_unlink(mapname);     };      if (mapfd == -1) {         perror("shm_open failed \n");         exit(EXIT_FAILURE);     }      if (ftruncate(mapfd, kMappingSize) == -1) {         perror("ftruncate failed \n");         exit(EXIT_FAILURE);     }      void* sp = mmap(nullptr, kMappingSize, PROT_READ | PROT_WRITE, MAP_SHARED, mapfd, 0);     if (!sp) {         perror("mmap failed \n");         exit(EXIT_FAILURE);     }      sem_t* mutex = (sem_t*)sp;      if (sem_init(mutex, 1, 1) != 0) {         perror("sem_init failed \n");         exit(EXIT_FAILURE);     }      MEOW_DEFER { sem_destroy(mutex); };      int* num = (int*)((char*)sp + sizeof(sem_t));     int cid, proc_count = 0, max_proc_count = 8;     for (int i = 0; i < max_proc_count; ++i) {         cid = fork();         if (cid == -1) {             perror("fork failed \n");             continue;         }         if (cid == 0) {             sem_wait(mutex);             (*num)++;             printf("process %d : %d \n", getpid(), *num);             sem_post(mutex);              if (munmap(sp, kMappingSize) == -1) {                 perror("munmap failed\n");             }             close(mapfd);             exit(EXIT_SUCCESS);         }         ++proc_count;     }      int stat;     while (proc_count--) {         cid = wait(&stat);         if (cid == -1) {             perror("wait failed \n");             break;         }     }      printf("ok \n"); }
```


代码中的MEOW_DEFER我在之前的RAII相关文章中介绍过，它内部的函数会在生命周期结束后触发。它的核心函数其实就是下面这四个：

```c
int sem_init(sem_t *sem,int pshared,unsigned int value); 
int sem_post(sem_t *sem); 
int sem_wait(sem_t *sem); 
int sem_destroy(sem_t *sem);
```

具体含义大家应该看名字就知道，这里的重点就是sem_init中的pshared参数，该参数为1表示可在进程间共享，为0表示只在进程内部共享。

第二种方式是使用锁，即pthread_mutex_t，可是pthread_mutex不是用作线程间数据竞争的吗，怎么能用在进程间呢？

我也是最近才知道，可以给它配置一个属性，示例代码如下：

```c
pthread_mutex_t* mutex; 
pthread_mutexattr_t mutexattr;  
pthread_mutexattr_init(&mutexattr); pthread_mutexattr_setpshared(&mutexattr, PTHREAD_PROCESS_SHARED); pthread_mutex_init(mutex, &mutexattr);
```

它的默认属性是进程内私有，但是如果给它配置成PTHREAD_PROCESS_SHARED，它就可以用在进程间通信中。

完整代码如下：

```c
void func() {     const char* mapname = "/mapname";     int mapfd = shm_open(mapname, O_RDWR | O_CREAT, S_IRUSR | S_IWUSR);      MEOW_DEFER {         if (mapfd > 0) {             close(mapfd);             mapfd = 0;         }         shm_unlink(mapname);     };      if (mapfd == -1) {         perror("shm_open failed \n");         exit(EXIT_FAILURE);     }      if (ftruncate(mapfd, kMappingSize) == -1) {         perror("ftruncate failed \n");         exit(EXIT_FAILURE);     }      void* sp = mmap(nullptr, kMappingSize, PROT_READ | PROT_WRITE, MAP_SHARED, mapfd, 0);     if (!sp) {         perror("mmap failed \n");         exit(EXIT_FAILURE);     }      pthread_mutex_t* mutex = (pthread_mutex_t*)sp;     pthread_mutexattr_t mutexattr;      pthread_mutexattr_init(&mutexattr);     pthread_mutexattr_setpshared(&mutexattr, PTHREAD_PROCESS_SHARED);     pthread_mutex_init(mutex, &mutexattr);      MEOW_DEFER {         pthread_mutexattr_destroy(&mutexattr);         pthread_mutex_destroy(mutex);     };      int* num = (int*)((char*)sp + sizeof(pthread_mutex_t));     int cid, proc_count = 0, max_proc_count = 8;     for (int i = 0; i < max_proc_count; ++i) {         cid = fork();         if (cid == -1) {             perror("fork failed \n");             continue;         }         if (cid == 0) {             pthread_mutex_lock(mutex);             (*num)++;             printf("process %d : %d \n", getpid(), *num);             pthread_mutex_unlock(mutex);              if (munmap(sp, kMappingSize) == -1) {                 perror("munmap failed\n");             }             close(mapfd);             exit(EXIT_SUCCESS);         }         ++proc_count;     }      int stat;     while (proc_count--) {         cid = wait(&stat);         if (cid == -1) {             perror("wait failed \n");             break;         }     }      printf("ok \n"); }
```

我想这两种方式应该可以满足我们日常开发过程中的大多数需求。

锁的方式介绍完之后，可能很多朋友自然就会想到原子变量，这块我也搜索了一下。但是也不太确定C++标准中的atomic是否在进程间通信中有作用，不过看样子boost中的atomic是可以用在进程间通信中的。

其实在研究这个问题的过程中，还找到了一些很多解决办法，包括：

- Disabling Interrupts
- Lock Variables
- Strict Alternation
- Peterson's Solution
- The TSL Instruction
- Sleep and Wakeup
- Semaphores
- Mutexes
- Monitors
- Message Passing
- Barriers

这里我就不过多介绍啦，大家感兴趣的可以自行查阅资料哈。
我也整理了一下自己找到的相关pdf资料，感兴趣的可以公众号后台回复“微信”加我好友，找我领取哈。
这次的分享就到这里，希望能够帮助到大家。

     **参考链接**

- http://husharp.today/2020/12/10/IPC-and-Lock/
- https://linux.die.net/man/3/pthread_mutexattr_init
- https://www.cnblogs.com/muyi23333/articles/13533291.html
- https://man7.org/linux/man-pages/man7/sem_overview.7.html
- https://codeantenna.com/a/uXNTzONHZI
  

---

往期推荐

  

  

[

不要再无脑背诵面向对象三大特性了



](https://mp.weixin.qq.com/s?__biz=MzkyODE5NjU2Mw==&mid=2247492325&idx=1&sn=3e67b6475a8880be3ef6c30172d7ba64&chksm=c21ed059f569594fc4af71a41d8645ecb412173b54eda1eb228b07a4c0b60b062508dcfd34a5&scene=21#wechat_redirect)

[

继承和组合，究竟我要选哪个？



](https://mp.weixin.qq.com/s?__biz=MzkyODE5NjU2Mw==&mid=2247492297&idx=1&sn=fa42f11cb78e6b836c64d52a5dca001c&chksm=c21ed075f56959638ac858236cdf551eab241eacb544043f94f1c9e1b5bb3324c2f904d568d4&scene=21#wechat_redirect)

[

一位大佬对于 Qt 学习的最全总结（三万字干货）



](https://mp.weixin.qq.com/s?__biz=MzkyODE5NjU2Mw==&mid=2247492232&idx=2&sn=6fea3c08c478db39fc993bc084e2685d&chksm=c21ed034f569592226f61cf5a8ec9729aec38846eef166015b5d890fda4ea8e299958ccb499c&scene=21#wechat_redirect)

[

C++ Trick：右值引用、万能引用傻傻分不清楚



](https://mp.weixin.qq.com/s?__biz=MzkyODE5NjU2Mw==&mid=2247492121&idx=1&sn=bf57c4783804050b2a27f0b922aad992&chksm=c21ed0a5f56959b38b9b4fd3035fbb69da20a2a406fc79b5cc2945a5dd3d65d8c6a8bf397c9e&scene=21#wechat_redirect)

[

有没有比友元更优雅的方式



](https://mp.weixin.qq.com/s?__biz=MzkyODE5NjU2Mw==&mid=2247491888&idx=1&sn=134e460dd565676c84e6544c7ac658cb&chksm=c21ed38cf5695a9a8b482bd3620fab6fd27ab5989f9f30b2b76458583ddad5026883b9ff8872&scene=21#wechat_redirect)

[

C++服务性能优化的道与术-道篇：阿姆达尔定律



](https://mp.weixin.qq.com/s?__biz=MzkyODE5NjU2Mw==&mid=2247491868&idx=2&sn=2fcead3032e1e06736094fa7183fc6a4&chksm=c21ed3a0f5695ab60909003b4d33f1915ae2f0d0c4874ad86572a8801dfc8a4836f5ff24df04&scene=21#wechat_redirect)

[

压箱底的音视频学习资料以及面经整理



](https://mp.weixin.qq.com/s?__biz=MzkyODE5NjU2Mw==&mid=2247491854&idx=1&sn=a35ed5df522bdec5100fbc1e246e92ac&chksm=c21ed3b2f5695aa4caab21adc83724e100940ad176116778b13e463585bdb3deffd734435d28&scene=21#wechat_redirect)

[

C++ Best Practices (C++最佳实践)翻译与阅读笔记



](https://mp.weixin.qq.com/s?__biz=MzkyODE5NjU2Mw==&mid=2247491836&idx=1&sn=9cbf51c885d45829b2d3012b788f8728&chksm=c21ed240f5695b567cfb81f51865eb7cba81aeb23772de6414e18eac832b98a5a168d50fd98f&scene=21#wechat_redirect)

[

万字长文 |  STL 算法总结



](https://mp.weixin.qq.com/s?__biz=MzkyODE5NjU2Mw==&mid=2247491833&idx=2&sn=0b217d743cd62ef00ac7ce6ee24f285f&chksm=c21ed245f5695b539555663502112b71a7389fcf40509bafece03ffb7ee38ab650ecda4ee89d&scene=21#wechat_redirect)

[

2021程序喵技术年货



](https://mp.weixin.qq.com/s?__biz=MzkyODE5NjU2Mw==&mid=2247491801&idx=1&sn=08493bd9b2eb17d93ba94563f85eb433&chksm=c21ed265f5695b73e7045b886067b22cf2a60fd9666d82e7eaa7db21a9a37a38cdabe6dfd62b&scene=21#wechat_redirect)

[

【性能优化】lock-free在召回引擎中的实现



](https://mp.weixin.qq.com/s?__biz=MzkyODE5NjU2Mw==&mid=2247491655&idx=1&sn=6d5b0480adb1c90418ba898e4bd0ad9c&chksm=c21ed2fbf5695bedbf85fa11847ca0b3dcc5e7091de766d5c4314adb69e9b1f5bb5e52111707&scene=21#wechat_redirect)

  

阅读原文

阅读 2738

​

写留言

**留言 12**

- 程序喵
    
    2022年2月7日
    
    赞1
    
    介绍进程间通信的文章是这个https://mp.weixin.qq.com/s/NCl17jrOwP_A017nUqOkJQ
    
    置顶
    
    程序喵
    
    2022年2月7日
    
    赞
    
    忘记贴链接了
    
- ELI
    
    2022年2月7日
    
    赞1
    
    恰好我年前用到也研究了一下，例子中的代码只是第一步，要想正确的应用到工程里还有非常多和复杂的的知识点要梳理，特别是我还用到了条件变量。我举个例子比如进程异常退出(信号sigint或sighup等)，恰好是在临界区收到，那么优雅正确的退出就是问题，涉及的知识点有linux的进程和线程的信号处理函数(历史遗留问题)，取消点的概念以及在自己的代码里设置取消点，还有锁的mutex的robust属性用来退出的时候防止死锁，还有自己代码一定要设计好调用pthread_mutex_consistent后业务逻辑不要乱。网上的资料太少，包括英文的，进程间共享锁的程序太少。
    
    程序喵大人
    
    作者2022年2月7日
    
    赞1
    
    赞
    
    ELI
    
    2022年2月7日
    
    赞3
    
    关注一堆技术公号，没人系统写这个问题，《Linux UNIX系统编程手册》才有写到这个问题，作者是真牛，虽然有1200页，但是很多问题还是不够详细，可能作者是被编辑要求精炼了不少内容，感觉要是放开写，作者能再写1200页。比《UNIX环境高级编程》更适合做手册。
    
    程序喵大人
    
    作者2022年2月7日
    
    赞
    
    可能是因为这块知识点应用场景不多
    
- 大白
    
    2022年2月7日
    
    赞
    
    关于进程间数据竞争问题:nginx多个worker进程的管理是不是也可以算是一种方案?
    
    程序喵大人
    
    作者2022年2月7日
    
    赞1
    
    我不太了解nginx![[撇嘴]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)
    
- 雨乐
    
    2022年2月7日
    
    赞1
    
    pthread这个，确实第一次知道，感谢喵大人的输出![😂](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)
    
    程序喵大人
    
    作者2022年2月7日
    
    赞
    
    客气啦
    
- 熊猫🐼爱吃竹子🎍
    
    2022年2月8日
    
    赞
    
    赞
    
- 东东东
    
    2022年2月7日
    
    赞
    
    进程间通信目标是实现共享内存无锁队列+信号通知，很多dds中间件都实现了。
    

已无更多数据

[](javacript:;)

![](http://mmbiz.qpic.cn/sz_mmbiz_png/Av3j9GXyMmGmg0HNK7fR6Peq3uf9pUoN5Qt0s0ia8IH97pvIiaADsEvy9nm3YnR5bmA13O8ic0JJOicQO0IboKGdZg/300?wx_fmt=png&wxfrom=18)

程序喵大人

29分享11

12

写留言

**留言 12**

- 程序喵
    
    2022年2月7日
    
    赞1
    
    介绍进程间通信的文章是这个https://mp.weixin.qq.com/s/NCl17jrOwP_A017nUqOkJQ
    
    置顶
    
    程序喵
    
    2022年2月7日
    
    赞
    
    忘记贴链接了
    
- ELI
    
    2022年2月7日
    
    赞1
    
    恰好我年前用到也研究了一下，例子中的代码只是第一步，要想正确的应用到工程里还有非常多和复杂的的知识点要梳理，特别是我还用到了条件变量。我举个例子比如进程异常退出(信号sigint或sighup等)，恰好是在临界区收到，那么优雅正确的退出就是问题，涉及的知识点有linux的进程和线程的信号处理函数(历史遗留问题)，取消点的概念以及在自己的代码里设置取消点，还有锁的mutex的robust属性用来退出的时候防止死锁，还有自己代码一定要设计好调用pthread_mutex_consistent后业务逻辑不要乱。网上的资料太少，包括英文的，进程间共享锁的程序太少。
    
    程序喵大人
    
    作者2022年2月7日
    
    赞1
    
    赞
    
    ELI
    
    2022年2月7日
    
    赞3
    
    关注一堆技术公号，没人系统写这个问题，《Linux UNIX系统编程手册》才有写到这个问题，作者是真牛，虽然有1200页，但是很多问题还是不够详细，可能作者是被编辑要求精炼了不少内容，感觉要是放开写，作者能再写1200页。比《UNIX环境高级编程》更适合做手册。
    
    程序喵大人
    
    作者2022年2月7日
    
    赞
    
    可能是因为这块知识点应用场景不多
    
- 大白
    
    2022年2月7日
    
    赞
    
    关于进程间数据竞争问题:nginx多个worker进程的管理是不是也可以算是一种方案?
    
    程序喵大人
    
    作者2022年2月7日
    
    赞1
    
    我不太了解nginx![[撇嘴]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)
    
- 雨乐
    
    2022年2月7日
    
    赞1
    
    pthread这个，确实第一次知道，感谢喵大人的输出![😂](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)
    
    程序喵大人
    
    作者2022年2月7日
    
    赞
    
    客气啦
    
- 熊猫🐼爱吃竹子🎍
    
    2022年2月8日
    
    赞
    
    赞
    
- 东东东
    
    2022年2月7日
    
    赞
    
    进程间通信目标是实现共享内存无锁队列+信号通知，很多dds中间件都实现了。
    

已无更多数据