# 

宋宝华 Jeff Labs

 _2022年01月10日 09:28_

  

_**原文作者：宋宝华**_

_**原文链接：https://blog.csdn.net/21cnbao/article/details/122396533**_

  

写时拷贝的原理我们没什么好赘述的，就是当P1 fork出来P2后，P1和P2会以只读的形式共享page，直到P1或者P2写这个page的内容，才发生page fault导致写的进程得到一份新的数据拷贝。下面的代码演示了它的效果：  

![图片](https://mmbiz.qpic.cn/mmbiz_png/Ass1lsY6bytQpBwvVk4KwpuT4e9m6aIDeFjMZcooZ8g8feiaibhj2zPOk0CebFY7j80Sh6iaBkNX9TjaGXiaZIVd6Q/640?wx_fmt=png&wxfrom=13&tp=wxpic)

上面的代码，执行的时候打印：

baohua@baohua-VirtualBox:~$ ./a.out 

Child process 3498, data 10

Child process 3498, data 20

Parent process 3497, data 10

子进程把10改为20后，父进程1秒后打印，得到的仍然是10。如果到这里为止，你看不懂，这篇文章不适合你这样的Linux初学者，请勿继续往下阅读。

从技术上来讲，在父进程写过数据后，子进程应该读不到父进程新写的数据；在子进程写过数据后，父进程也应该读不到子进程新写的数据。这才符合“进程是资源封装的单位”的本质定义。

如果都是上面的经典模型，那么岁月静好，与君白头偕老。但是，总会有人在花田里犯了错，破晓前仍然没有忘掉。这个COW技术，就爆出了巨大的漏洞，让父子进程间可以向对方泄露写过的新数据，成为了Linux内核的惊天大瓜。

我们先来看看是怎样的一个程序，让COW的人设崩塌了呢？

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

上面的程序，父子进程最初共享了data指向的0x1000这么大1个page的内容。然后父进程在data里面写“BORING DATA”,之后，父进程fork子进程。子进程接下来创建了一个pipe，并用vmsplice，把data指向的buffer拼接到了pipe的写端，而后子进程通过munmap()去掉data的映射，再睡眠2秒制造机会让父进程在data里面写"THIS IS SECRET"。2秒后，子进程read pipe的读端，这个时候，神奇的事情发生了，子进程读到了父进程写的秘密数据。

为什么会发生这种事情呢？魔鬼就在细节里。这里面有2个细节：

**1. 子进程munmap，导致data的mapcount减-1，这样欺骗了Linux内核，使得父进程在写THIS IS SECRET的时候，并不会发生COW，因为内核理解data只有1个进程有map，制造拷贝显然是多余的。**

**2.子进程调用vmsplice，这是一种0拷贝技术，避免管道的写端把userspace往kernel space进程拷贝。vmsplice的底层，会通过传说中的GUP(get_user_pages)技术，来增加page的引用计数，导致page不会被回收和释放。**

所以，子进程通过pipe的写端hold住了老的page，然后通过read()，把这个page经过父进程写后的新内容读出来了。这真地很神奇有木有！这个漏洞的编号是CVE-2020-29374，它的官方描述如下：

An issue was discovered in the Linux kernel before 5.7.3, related to mm/gup.c and mm/huge_memory.c. The get_user_pages (aka gup) implementation, when used for a copy-on-write page, does not properly consider the semantics of read operations and therefore can grant unintended write access, aka CID-17839856fd58.

这个瓜大地直接惊动了祖师爷Linus Torvalds发patch来进行“修复”，Linus的“修复”patch编号是17839856fd58 ("gup: document and work around 'COW can break either way' issue")。祖师爷的修复方法比较简单直接，对于任何要COW的page，如果你做GUP，哪怕你后面对这个page的行为是只读的，也要得到一份新的copy。对应前面的参考代码，其实就是子进程调用vmsplice的行为，打破了COW的常规逻辑，之后子进程read(pipe[0])的时候，读到的是新的page。

所以没有Linus的patch的时候，data的内存在父子进程分布如下：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

有了Linus的patch后，data的内存在父子进程分布如下：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

显然，这样之后，父进程写data后，写的是蓝色区域，子进程读的是黄色的区域，这样子进程是肯定读不到SECRET数据了。

**Linus是永远正确的？必须是！**当Linus把这个patch合入5.8内核的时候，人们以为故事就此结束了，却没想到瓜才刚刚开始。作为Linus内核的吃瓜群众，我们的激情从不曾磨灭，因为“吃在嘴里，甜在心里”，吃瓜的甜蜜诱惑引诱我们一步步走入Linux内核的深渊，误了一生。

redhat的Peter Xu童鞋，在2020年8月报了一个bug，直指祖师爷的patch造成了问题，因为它破坏了类似userfaultfd-wp和umapsort这样的应用程序。注意，子曾经曰过，“If a change results in user programs breaking, it's a bug in the kernel. We never EVER blame the user programs”，有图有真相：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

一个典型的umap代码仓库在：

GitHub - LLNL/umap: User-space Page Management

_https://github.com/LLNL/umap_

这种app利用userfaultfd的原理，在userspace处理page fault，从而提供userspace特定的page cache evict机制。关于userfaultfd的原理和用法，你可以阅读我之前的文章

宋宝华：论一切都是文件之匿名inode_宋宝华-CSDN博客

_https://blog.csdn.net/21cnbao/article/details/115153742_

简单来说，umap这样的程序通过3个步骤来evict page。

  (1) 用mode=WP来对即将被evict的page执行写保护，从而block对于page P的写，保持page的clean；

  (2) 把page P写入磁盘；

  (3) 通过MADV_DONTNEED来evict这个page。

其中的第2步会用到一个read形式的GUP。不过，Linus已经通过他的patch，强迫哪怕是read形式的GUP也要发生COW,这样触发了一个app完全没有预期到的page fault，导致uffd线程出错hang死。显然Linus自己break了userspace，等待他的结局是，他的patch的行为也要被revert掉。这一次仍然是Linus亲自出手，他提交了09854ba94c6a ("patch: mm: do_wp_page() simplification")，导致程序的行为再次发生了翻天覆地的变化。

前面我们提到，通过Linus的17839856fd58 ("gup: document and work around 'COW can break either way' issue") patch，子进程vmsplice的GUP行为会强迫子进程进行COW，得到新的拷贝。但是，现在Linus不这个干了，vmsplice的pipe写端还是指向老的页面，他重新选择了在父进程进行实际的写的时候，不再只是傻傻地判断page的mapcount，他还会判断是不是有人间接通过GUP等形式，增加了page的引用计数，如果是，则在父进程写的时候，进行copy-on-write，这个时候，父进程写过"THIS IS SECRET"后，data在父子进程的内存分布变成：

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

由于父进程是在新的黄色page进行写，而子进程用的是老的蓝色page，所以"THIS IS SECRET"不会泄露给子进程。Linus的最主要修改是直接变更了do_wp_page()函数，逻辑变成：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

因为GUP的行为会增加page的refcount，从而触发父进程在写data的wp的page fault里面，进行COW。所以Linus是守信用的，自己提交的patch犯的错，含泪也要revert掉。

那么故事就此结束了吗？正当所有的吃瓜群众都把西瓜皮扔到垃圾桶准备休息一阵的时候，蕾神再次以今天之锤，锤向了“花田里犯的错”。

累了，睡觉了。预知后事如何，请听下回分解。

  

阅读 375

​

写留言

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/QO9OBu0wPg0c2nEoRPjUtn2uQGibnXhXMxuKw5RwHLdVzsm6iaIE3okWLL42EIpzcPb33fS2pa8CicPrzpesewvCw/300?wx_fmt=png&wxfrom=18)

Jeff Labs

4分享1

写留言

写留言

**留言**

暂无留言