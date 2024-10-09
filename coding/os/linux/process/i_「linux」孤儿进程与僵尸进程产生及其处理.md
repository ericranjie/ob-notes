土豆居士 一口Linux
_2021年12月10日 11:50_

在探讨这个问题之前，我们先来弄清什么是进程。

进程(Process)是计算机中的程序关于某数据集合上的一次运行活动，是系统进行资源分配和调度的基本单位，是操作系统结构的基础。程序是指令、数据及其组织形式的描述，进程是程序的实体。进程是一个具有独立功能的程序关于某个数据集合的一次运行活动。它可以申请和拥有系统资源，是一个动态的概念，是一个活动的实体。它不只是程序的代码，还包括当前的活动，通过程序计数器的值和处理寄存器的内容来表示。通俗点讲，进程是一段程序的执行过程，是个动态概念。

# 一：进程状态

![图片](https://mmbiz.qpic.cn/mmbiz_png/icRxcMBeJfcibNZrQ7MJaddQtgYRQuaAHMoDEn4PcEicQvcvibZibQT3hU8YrYocic5s1cgasV6GcLe041C6nIecMj3w/640?wx_fmt=png&wxfrom=13&tp=wxpic)

程序运行必须加载在内存中，当有过多的就绪态或阻塞态进程在内存中没有运行，因为内存很小，有可能不足。系统需要把他们移动到内存外磁盘中，称为挂起状态。就绪状态的进程挂起就是挂起就绪状态，阻塞进程挂起就称为阻塞挂起状态。

每个进程的产生都有自己的唯一的ID号(pid)，并且附带有一个它父进程的ID号(ppid)。进程死亡时，ID被回收。

进程间靠优先级获得CPU资源，时间片段轮换来更新优先级，以保证不会一个进程占据CPU时间过长。每个进程都得到轮换运行，因为这个时间非常短，所以给我们就好像是系统在同时运行好多进程。

# 二：僵尸进程

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

那么什么称为僵尸进程呢？

即子进程先于父进程退出后，子进程的PCB需要其父进程释放，但是父进程并没有释放子进程的PCB，这样的子进程就称为僵尸进程，僵尸进程实际上是一个已经死掉的进程。我们用代码来看一下

```
#include<stdio.h>#include<stdlib.h>#include<unistd.h>#include<string.h>#include<assert.h>#include<sys/types.h> int main(){	pid_t pid=fork(); 	if(pid==0)  //子进程	{     	printf("child id is %d\n",getpid());		printf("parent id is %d\n",getppid());	}	else  //父进程不退出，使子进程成为僵尸进程	{        while(1)		{}	}	exit(0);}
```

我们将它挂在后台执行，可以看到结果，用ps可以看到子进程后有一个<defunct> ,defunct是已死的，僵尸的意思,可以看出这时的子进程已经是一个僵尸进程了。因为子进程已经结束，而其父进程并未释放其PCB，所以产生了这个僵尸进程。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

我们也可以用ps -aux | grep pid 查看进程状态

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

一个进程在调用exit命令结束自己的生命的时候，其实它并没有真正的被销毁，而是留下一个称为僵尸进程（Zombie）的数据结构（系统调用exit，它的作用是使进程退出，但也仅仅限于将一个正常的进程变成一个僵尸进程，并不能将其完全销毁）。在Linux进程的状态中，僵尸进程是非常特殊的一种，它已经放弃了几乎所有内存空间，没有任何可执行代码，也不能被调度，仅仅在进程列表中保留一个位置，记载该进程的退出状态等信息供其他进程收集，除此之外，僵尸进程不再占有任何内存空间。这个僵尸进程需要它的父进程来为它收尸，如果他的父进程没有处理这个僵尸进程的措施，那么它就一直保持僵尸状态，如果这时父进程结束了，那么init进程自动会接手这个子进程，为它收尸，它还是能被清除的。但是如果如果父进程是一个循环，不会结束，那么子进程就会一直保持僵尸状态，这就是为什么系统中有时会有很多的僵尸进程。

试想一下，如果有大量的僵尸进程驻在系统之中，必然消耗大量的系统资源。但是系统资源是有限的，因此当僵尸进程达到一定数目时，系统因缺乏资源而导致奔溃。所以在实际编程中，避免和防范僵尸进程的产生显得尤为重要。

# 三：孤儿进程

一个父进程退出，而它的一个或多个子进程还在运行，那么那些子进程将成为孤儿进程。孤儿进程将被init进程(进程号为1)所收养，并由init进程对它们完成状态收集工作。

子进程死亡需要父进程来处理，那么意味着正常的进程应该是子进程先于父进程死亡。当父进程先于子进程死亡时，子进程死亡时没父进程处理，这个死亡的子进程就是孤儿进程。

但孤儿进程与僵尸进程不同的是，由于父进程已经死亡，系统会帮助父进程回收处理孤儿进程。所以孤儿进程实际上是不占用资源的，因为它终究是被系统回收了。不会像僵尸进程那样占用ID,损害运行系统。

下来我们上代码看看：

```
#include<stdio.h>#include<stdlib.h>#include<unistd.h>#include<string.h>#include<assert.h>#include<sys/types.h> int main(){	pid_t pid=fork(); 	if(pid==0)	{		printf("child ppid is %d\n",getppid());		sleep(10);     //为了让父进程先结束		printf("child ppid is %d\n",getppid());	}	else	{		printf("parent id is %d\n",getpid());	} 	exit(0);}
```

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

从执行结果来看，此时由pid == 4168父进程创建的子进程，其输出的父进程pid == 1，说明当其为孤儿进程时被init进程回收，最终并不会占用资源，这就是为什么要将孤儿进程分配给init进程。

# 四：僵尸进程处理方式

任何一个子进程(init除外)在exit()之后，并非马上就消失掉，而是留下一个称为僵尸进程(Zombie)的数据结构，等待父进程处理。这是每个子进程在结束时都要经过的阶段。如果子进程在exit()之后，父进程没有来得及处理，这时用ps命令就能看到子进程的状态是“defunct”。如果父进程能及时处理，可能用ps命令就来不及看到子进程的僵尸状态，但这并不等于子进程不经过僵尸状态。如果父进程在子进程结束之前退出，则子进程将由init接管。init将会以父进程的身份对僵尸状态的子进程进行处理。所以孤儿进程不会占资源，僵尸进程会占用资源危害系统。我们应当避免僵尸进程的出现。

解决方式如下：

1）：一种比较暴力的做法是将其父进程杀死，那么它的子进程，即僵尸进程会变成孤儿进程，由系统来回收。但是这种做法在大多数情况下都是不可取的，如父进程是一个服务器程序，如果为了回收其子进程的资源，而杀死服务器程序，那么将导致整个服务器崩溃，得不偿失。显然这种回收进程的方式是不可取的，但其也有一定的存在意义。

2）：SIGCHLD信号处理

我们都知道wait函数是用来处理僵尸进程的，但是进程一旦调用了wait，就立即阻塞自己，由wait自动分析是否当前进程的某个子进程已经退出，如果让它找到了这样一个已经变成僵尸的子进程，wait就会收集这个子进程的信息，并把它彻底销毁后返回；如果没有找到这样一个子进程，wait就会一直阻塞在这里，直到有一个出现为止。我们先来看看wait函数的定义

```
#include <sys/types.h> /* 提供类型pid_t的定义，实际就是int型 */
```

参数status用来保存被收集进程退出时的一些状态，它是一个指向int类型的指针。但如果我们对这个子进程是如何死掉的毫不在意，只想把这个僵尸进程消灭掉，（事实上绝大多数情况下，我们都会这样想），我们就可以设定这个参数为NULL，就象下面这样：pid=wait(NULL)；如果成功，wait会返回被收集的子进程的进程ID，如果调用进程没有子进程，调用就会失败，此时wait返回-1，同时errno被置为ECHILD。

由于调用wait之后，就必须阻塞，直到有子进程结束，所以，这样来说是非常不高效的，我们的父进程难道要一直等待你子进程完成，最后才能执行自己的代码吗？难道就不能我父进程执行自己的代码，你子进程什么时候完成我就什么时候去处理你，不用一直等你？当然是有这种方式了。

实际上当子进程终止时，内核就会向它的父进程发送一个SIGCHLD信号，父进程可以选择忽略该信号，也可以提供一个接收到信号以后的处理函数。对于这种信号的系统默认动作是忽略它。我们不希望有过多的僵尸进程产生，所以当父进程接收到SIGCHLD信号后就应该调用 wait 或 waitpid 函数对子进程进行善后处理，释放子进程占用的资源。

下面是一个处理僵尸进程的简单的例子：

```
#include<stdio.h>#include<stdlib.h>#include<unistd.h>#include<string.h>#include<assert.h>#include<sys/types.h>#include<sys/wait.h>#include<signal.h> void deal_child(int num){	printf("deal_child into\n");	wait(NULL);}int main(){	signal(SIGCHLD,deal_child);	pid_t pid=fork();	int i; 	if(pid==0)	{		printf("child is running\n");		sleep(2);		printf("child will end\n");	}	else	{		sleep(1);   //让子进程先执行		printf("parent is running\n");		sleep(10);    //一旦被打断就不能再进入睡眠		printf("sleep 10 s over\n");		sleep(5);		printf("sleep 5s over\n");	} 	exit(0);}
```

进行测试后确定了是在父进程睡眠10s时子进程结束，父进程接收到了SIGCHLD信号，调用了deal_child函数，释放了子进程的PCB后又回到自己本身的代码中执行。我们看看运行结果

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

说到这里，我们再来看看signal函数（不是阻塞函数）

signal（参数1，参数2）；

参数1：我们要进行处理的信号。系统的信号我们可以再终端键入 kill -l查看(共64个)。其实这些信号是系统定义的宏。

参数2：我们处理的方式（是系统默认还是忽略还是捕获）。

```
eg: signal(SIGINT ,SIG_ING ); //SIG_ING 代表忽略SIGINT信号
```

```
eg:signal(SIGINT,SIG_DFL); //SIGINT信号代表由InterruptKey产生，通常是CTRL +C或者是DELETE。
```

发送给所有ForeGroundGroup的进程。

SIG_DFL代表执行系统默认操作，其实对于大多数信号的系统默认动作是终止该进程。这与不写此处理函数是一样的。

我们也可以给参数2传递一个信号处理函数的地址，但是这个信号处理函数需要其返回值为void，并且默认自带一个int类型参数

这个int就是你所传递的第一个信号参数的值（你用kill -l可以查看）

我们测试了一下，如果创建了5个子进程，但是销毁的时候仍然有两个仍是僵尸进程，这又是为什么呢？

这是因为当5个进程同时终止的时候，内核都会向父进程发送SIGCHLD信号，而父进程此时有可能仍然处于信号处理的deal_child函数中，那么在处理完之前，中间接收到的SIGCHLD信号就会丢失，内核并没有使用队列等方式来存储同一种信号

所以为了解决这一问题，我们需要调用waitpid函数来清理子进程。

```
void deal_child(int sig_no) {    for (;;) {        if (waitpid(-1, NULL, WNOHANG) == 0)            break;    }  }
```

这样的话，只有检验没有僵尸进程，他才会返回0，这样就可以确保所有的僵尸进程都被杀死了。

end

**一口Linux**

**关注，回复【****1024****】海量Linux资料赠送**

![](http://mmbiz.qpic.cn/mmbiz_png/icRxcMBeJfc8535w2vKlsLPf5hwdMjpYrzuVCHx0rcQmvv8rYqTFtIyic5qErtciaibqaIOWgeKkDsOMeae4HciaUaw/300?wx_fmt=png&wxfrom=19)

**一口Linux**

《从零开始学ARM》作者，一起学习嵌入式，Linux，网络，驱动，arm知识。

300篇原创内容

公众号

**精彩文章合集**

文章推荐

☞【专辑】[ARM](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=MzUxMjEyNDgyNw==&action=getalbum&album_id=1614665559315382276#wechat_redirect)

☞【专辑】[粉丝问答](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=MzUxMjEyNDgyNw==&action=getalbum&album_id=1629876820810465283#wechat_redirect)

☞【专辑】[所有原创](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=MzUxMjEyNDgyNw==&action=getalbum&album_id=1479949091139813387#wechat_redirect)

☞【专辑】[linux](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=MzUxMjEyNDgyNw==&action=getalbum&album_id=1507350615537025026#wechat_redirect)入门

☞【专辑】[计算机网络](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=MzUxMjEyNDgyNw==&action=getalbum&album_id=1598710257097179137#wechat_redirect)

☞【专辑】[Linux驱动](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=MzUxMjEyNDgyNw==&action=getalbum&album_id=1502410824114569216#wechat_redirect)

☞【干货】[嵌入式驱动工程师学习路线](http://mp.weixin.qq.com/s?__biz=MzUxMjEyNDgyNw==&mid=2247496985&idx=1&sn=c3d5e8406ff328be92d3ef4814108cd0&chksm=f96b87edce1c0efb6f60a6a0088c714087e4a908db1938c44251cdd5175462160e26d50baf24&scene=21#wechat_redirect)

☞【干货】[Linux嵌入式所有知识点-思维导图](http://mp.weixin.qq.com/s?__biz=MzUxMjEyNDgyNw==&mid=2247497822&idx=1&sn=1e2aed9294f95ae43b1ad057c2262980&chksm=f96b8aaace1c03bc2c9b0c3a94c023062f15e9ccdea20cd76fd38967b8f2eaad4dfd28e1ca3d&scene=21#wechat_redirect)

点击“**阅读原文**”查看更多分享，欢迎**点分享、收藏、点赞、在看**

linux134

linux · 目录

上一篇面试必问：进程和线程的区别（从操作系统层次理解）下一篇从0实现基于Linux socket聊天室-增加数据加密功能-6

阅读原文

阅读 2258

​

写留言

**留言 4**

- 堇

  2021年12月10日

  赞1

  进程：你骂谁？![[奸笑]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

  一口Linux

  作者2021年12月10日

  赞1

  ![[偷笑]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![[偷笑]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![[偷笑]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

- sure

  2022年2月19日

  赞

  docker container里面没有指定init程序，有时候就会产生僵尸进程。加上SIGCHLD处理函数和waitpid还是有好处的。

  一口Linux

  作者2022年2月19日

  赞

  是的![[强]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![[强]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

已无更多数据

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/icRxcMBeJfc8535w2vKlsLPf5hwdMjpYrzuVCHx0rcQmvv8rYqTFtIyic5qErtciaibqaIOWgeKkDsOMeae4HciaUaw/300?wx_fmt=png&wxfrom=18)

一口Linux

6分享4

4

写留言

**留言 4**

- 堇

  2021年12月10日

  赞1

  进程：你骂谁？![[奸笑]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

  一口Linux

  作者2021年12月10日

  赞1

  ![[偷笑]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![[偷笑]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![[偷笑]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

- sure

  2022年2月19日

  赞

  docker container里面没有指定init程序，有时候就会产生僵尸进程。加上SIGCHLD处理函数和waitpid还是有好处的。

  一口Linux

  作者2022年2月19日

  赞

  是的![[强]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![[强]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

已无更多数据
