极客重生 极客重生
 _2022年02月17日 19:36_
 
![[Pasted image 20241007192403.png]]

原文：http://33h.co/kyq4z
# Part1 Linux性能优化
## 1 性能优化
### 性能指标

高并发和响应快对应着性能优化的两个核心指标：**吞吐**和**延时**

![图片](https://mmbiz.qpic.cn/mmbiz_png/lPwvgkMwLNiacfpvYibLHkL44j7YsLVfPEbSvsia2wKhoVVn0jwicZCicP4kTO4lDgQKkqdnOH4QwsGtoHEARToqZNg/640?wx_fmt=png&wxfrom=13&tp=wxpic)

图片来自: www.ctq6.cn

- **应用负载**角度：直接影响了产品终端的用户体验
- **系统资源**角度：资源使用率、饱和度等

**性能问题的本质**就是系统资源已经到达瓶颈，但请求的处理还不够快，无法支撑更多的请求。性能分析实际上就是找出应用或系统的瓶颈，设法去避免或缓解它们。

- 选择指标评估应用程序和系统性能
- 为应用程序和系统设置性能目标
- 进行性能基准测试
- 性能分析定位瓶颈
- 性能监控和告警

对于不同的性能问题要选取不同的性能分析工具。下面是常用的Linux Performance Tools以及对应分析的性能问题类型。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

图片来自: www.ctq6.cn

### 到底应该怎么理解"平均负载"

平均负载：单位时间内，系统处于可运行状态和不可中断状态的平均进程数，也就是平均活跃进程数。它和我们传统意义上理解的CPU使用率并没有直接关系。

其中不可中断进程是正处于内核态关键流程中的进程（如常见的等待设备的I/O响应）。**不可中断状态实际上是系统对进程和硬件设备的一种保护机制。**

### 平均负载多少时合理

实际生产环境中将系统的平均负载监控起来，根据历史数据判断负载的变化趋势。当负载存在明显升高趋势时，及时进行分析和调查。当然也可以当设置阈值（如当平均负载高于CPU数量的70%时）

现实工作中我们会经常混淆平均负载和CPU使用率的概念，其实两者并不完全对等：

- CPU密集型进程，大量CPU使用会导致平均负载升高，此时两者一致
- I/O密集型进程，等待I/O也会导致平均负载升高，此时CPU使用率并不一定高
- 大量等待CPU的进程调度会导致平均负载升高，此时CPU使用率也会比较高

**平均负载高时可能是CPU密集型进程导致，也可能是I/O繁忙导致。具体分析时可以结合mpstat/pidstat工具辅助分析负载来源**
## 2 CPU
### CPU上下文切换(上)

**CPU上下文切换**，就是把前一个任务的CPU上下文（CPU寄存器和PC）保存起来，然后加载新任务的上下文到这些寄存器和程序计数器，最后再跳转到程序计数器所指的位置，运行新任务。其中，保存下来的上下文会存储在系统内核中，待任务重新调度执行时再加载，保证原来的任务状态不受影响。

按照任务类型，CPU上下文切换分为：

- 进程上下文切换
- 线程上下文切换
- 中断上下文切换
#### 进程上下文切换

Linux进程按照等级权限将进程的运行空间分为内核空间和用户空间。从用户态向内核态转变时需要通过系统调用来完成。

一次系统调用过程其实进行了两次CPU上下文切换：

- CPU寄存器中用户态的指令位置先保存起来，CPU寄存器更新为内核态指令的位置，跳转到内核态运行内核任务；
- 系统调用结束后，CPU寄存器恢复原来保存的用户态数据，再切换到用户空间继续运行。

系统调用过程中并不会涉及虚拟内存等进程用户态资源，也不会切换进程。和传统意义上的进程上下文切换不同。因此**系统调用通常称为特权模式切换**。

进程是由内核管理和调度的，进程上下文切换只能发生在内核态。因此相比系统调用来说，在保存当前进程的内核状态和CPU寄存器之前，需要先把该进程的虚拟内存，栈保存下来。再加载新进程的内核态后，还要刷新进程的虚拟内存和用户栈。

进程只有在调度到CPU上运行时才需要切换上下文，有以下几种场景：CPU时间片轮流分配，系统资源不足导致进程挂起，进程通过sleep函数主动挂起，高优先级进程抢占时间片，硬件中断时CPU上的进程被挂起转而执行内核中的中断服务。
#### 线程上下文切换

线程上下文切换分为两种：

- 前后线程同属于一个进程，切换时虚拟内存资源不变，只需要切换线程的私有数据，寄存器等；
- 前后线程属于不同进程，与进程上下文切换相同。

同进程的线程切换消耗资源较少，这也是多线程的优势。

#### 中断上下文切换

中断上下文切换并不涉及到进程的用户态，因此中断上下文只包括内核态中断服务程序执行所必须的状态（CPU寄存器，内核堆栈，硬件中断参数等）。

**中断处理优先级比进程高，所以中断上下文切换和进程上下文切换不会同时发生**

**[Docker+K8s+Jenkins 主流技术全解视频资料](http://mp.weixin.qq.com/s?__biz=MzAwNTM5Njk3Mw==&mid=2247506337&idx=2&sn=3899a7373d1f2dde5d8c71cb17a3e51a&chksm=9b1fd923ac685035150c5efa021bdba0e278fe484f26ee56946a6310329199cddfff054defe0&scene=21#wechat_redirect)**

### CPU上下文切换(下)

通过vmstat可以查看系统总体的上下文切换情况

```c
vmstat 5         #每隔5s输出一组数据procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu----- r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st 1  0      0 103388 145412 511056    0    0    18    60    1    1  2  1 96  0  0 0  0      0 103388 145412 511076    0    0     0     2  450 1176  1  1 99  0  0 0  0      0 103388 145412 511076    0    0     0     8  429 1135  1  1 98  0  0 0  0      0 103388 145412 511076    0    0     0     0  431 1132  1  1 98  0  0 0  0      0 103388 145412 511076    0    0     0    10  467 1195  1  1 98  0  0 1  0      0 103388 145412 511076    0    0     0     2  426 1139  1  0 99  0  0 4  0      0  95184 145412 511108    0    0     0    74  500 1228  4  1 94  0  0 0  0      0 103512 145416 511076    0    0     0   455  723 1573 12  3 83  2  0
```

- cs （context switch） 每秒上下文切换次数
- in （interrupt） 每秒中断次数
- r （runnning or runnable）就绪队列的长度，正在运行和等待CPU的进程数
- b （Blocked） 处于不可中断睡眠状态的进程数

要查看每个进程的详细情况，需要使用pidstat来查看每个进程上下文切换情况

```c
pidstat -w 514时51分16秒   UID       PID   cswch/s nvcswch/s  Command14时51分21秒     0         1      0.80      0.00  systemd14时51分21秒     0         6      1.40      0.00  ksoftirqd/014时51分21秒     0         9     32.67      0.00  rcu_sched14时51分21秒     0        11      0.40      0.00  watchdog/014时51分21秒     0        32      0.20      0.00  khugepaged14时51分21秒     0       271      0.20      0.00  jbd2/vda1-814时51分21秒     0      1332      0.20      0.00  argusagent14时51分21秒     0      5265     10.02      0.00  AliSecGuard14时51分21秒     0      7439      7.82      0.00  kworker/0:214时51分21秒     0      7906      0.20      0.00  pidstat14时51分21秒     0      8346      0.20      0.00  sshd14时51分21秒     0     20654      9.82      0.00  AliYunDun14时51分21秒     0     25766      0.20      0.00  kworker/u2:114时51分21秒     0     28603      1.00      0.00  python3
```

- cswch 每秒自愿上下文切换次数 （进程无法获取所需资源导致的上下文切换）
    
- nvcswch 每秒非自愿上下文切换次数 （时间片轮流等系统强制调度）
    

```c
vmstat 1 1    #首先获取空闲系统的上下文切换次数sysbench --threads=10 --max-time=300 threads run #模拟多线程切换问题vmstat 1 1    #新终端观察上下文切换情况此时发现cs数据明显升高，同时观察其他指标：r列： 远超系统CPU个数，说明存在大量CPU竞争us和sy列：sy列占比80%，说明CPU主要被内核占用in列： 中断次数明显上升，说明中断处理也是潜在问题
```

说明运行/等待CPU的进程过多，导致大量的上下文切换，上下文切换导致系统的CPU占用率高

```c
pidstat -w -u 1  #查看到底哪个进程导致的问题
```

从结果中看出是sysbench导致CPU使用率过高，但是pidstat输出的上下文次数加起来也并不多。分析sysbench模拟的是线程的切换，因此需要在pidstat后加-t参数查看线程指标。

另外对于中断次数过多，我们可以通过/proc/interrupts文件读取

```c
watch -d cat /proc/interrupts
```

发现次数变化速度最快的是重调度中断（RES），该中断用来唤醒空闲状态的CPU来调度新的任务运行。分析还是因为过多任务的调度问题，和上下文切换分析一致。

### 某个应用的CPU使用率达到100%，怎么办？

Linux作为多任务操作系统，将CPU时间划分为很短的时间片，通过调度器轮流分配给各个任务使用。为了维护CPU时间，Linux通过事先定义的节拍率，触发时间中断，并使用全局变了jiffies记录开机以来的节拍数。时间中断发生一次该值+1.

**CPU使用率**，除了空闲时间以外的其他时间占总CPU时间的百分比。可以通过/proc/stat中的数据来计算出CPU使用率。因为/proc/stat时开机以来的节拍数累加值，计算出来的是开机以来的平均CPU使用率，一般意义不大。可以间隔取一段时间的两次值作差来计算该段时间内的平均CPU使用率。**性能分析工具给出的都是间隔一段时间的平均CPU使用率，要注意间隔时间的设置。**

CPU使用率可以通过top 或 ps来查看。分析进程的CPU问题可以通过perf，它以性能事件采样为基础，不仅可以分析系统的各种事件和内核性能，还可以用来分析指定应用程序的性能问题。

perf top / perf record / perf report （-g 开启调用关系的采样）

```c
sudo docker run --name nginx -p 10000:80 -itd feisky/nginxsudo docker run --name phpfpm -itd --network container:nginx feisky/php-fpmab -c 10 -n 100 http://XXX.XXX.XXX.XXX:10000/ #测试Nginx服务性能
```

发现此时每秒可承受请求给长少，此时将测试的请求数从100增加到10000。在另外一个终端运行top查看每个CPU的使用率。发现系统中几个php-fpm进程导致CPU使用率骤升。

接着用perf来分析具体是php-fpm中哪个函数导致该问题。

```c
perf top -g -p XXXX #对某一个php-fpm进程进行分析
```

发现其中sqrt和add_function占用CPU过多， 此时查看源码找到原来是sqrt中在发布前没有删除测试代码段，存在一个百万次的循环导致。将该无用代码删除后发现nginx负载能力明显提升

### 系统的CPU使用率很高，为什么找不到高CPU的应用？

```c
sudo docker run --name nginx -p 10000:80 -itd feisky/nginx:spsudo docker run --name phpfpm -itd --network container:nginx feisky/php-fpm:spab -c 100 -n 1000 http://XXX.XXX.XXX.XXX:10000/ #并发100个请求测试
```

实验结果中每秒请求数依旧不高，我们将并发请求数降为5后，nginx负载能力依旧很低。

此时用top和pidstat发现系统CPU使用率过高，但是并没有发现CPU使用率高的进程。

出现这种情况一般时我们分析时遗漏的什么信息，重新运行top命令并观察一会。发现就绪队列中处于Running状态的进行过多，超过了我们的并发请求次数5. 再仔细查看进程运行数据，发现nginx和php-fpm都处于sleep状态，真正处于运行的却是几个stress进程。

下一步就利用pidstat分析这几个stress进程，发现没有任何输出。用ps aux交叉验证发现依旧不存在该进程。说明不是工具的问题。再top查看发现stress进程的进程号变化了，此时有可能时以下两种原因导致：

- 进程不停的崩溃重启（如段错误/配置错误等），此时进程退出后可能又被监控系统重启；
    
- 短时进程导致，即其他应用内部通过exec调用的外面命令，这些命令一般只运行很短时间就结束，很难用top这种间隔较长的工具来发现
    

可以通过pstree来查找 stress的父进程，找出调用关系。

```c
pstree | grep stress
```

发现是php-fpm调用的该子进程，此时去查看源码可以看出每个请求都会调用一个stress命令来模拟I/O压力。之前top显示的结果是CPU使用率升高，是否真的是由该stress命令导致的，还需要继续分析。代码中给每个请求加了verbose=1的参数后可以查看stress命令的输出，在中断测试该命令结果显示stress命令运行时存在因权限问题导致的文件创建失败的bug。

此时依旧只是猜测，下一步继续通过perf工具来分析。性能报告显示确实时stress占用了大量的CPU，通过修复权限问题来优化解决即可.

### 系统中出现大量不可中断进程和僵尸进程怎么办？

#### 进程状态

- R Running/Runnable，表示进程在CPU的就绪队列中，正在运行或者等待运行；
    
- D Disk Sleep，不可中断状态睡眠，一般表示进程正在跟硬件交互，并且交互过程中不允许被其他进程中断；
    
- Z Zombie，僵尸进程，表示进程实际上已经结束，但是父进程还没有回收它的资源；
    
- S Interruptible Sleep，可中断睡眠状态，表示进程因为等待某个事件而被系统挂起，当等待事件发生则会被唤醒并进入R状态；
    
- I Idle，空闲状态，用在不可中断睡眠的内核线程上。该状态不会导致平均负载升高；
    
- T Stop/Traced，表示进程处于暂停或跟踪状态（SIGSTOP/SIGCONT， GDB调试）；
    
- X Dead，进程已经消亡，不会在top/ps中看到。
    

对于不可中断状态，一般都是在很短时间内结束，可忽略。但是如果系统或硬件发生故障，进程可能会保持不可中断状态很久，甚至系统中出现大量不可中断状态，此时需注意是否出现了I/O性能问题。

僵尸进程一般多进程应用容易遇到，父进程来不及处理子进程状态时子进程就提前退出，此时子进程就变成了僵尸进程。大量的僵尸进程会用尽PID进程号，导致新进程无法建立。

#### 磁盘O_DIRECT问题

```c
sudo docker run --privileged --name=app -itd feisky/app:iowaitps aux | grep '/app'
```

可以看到此时有多个app进程运行，状态分别时Ss+和D+。其中后面s表示进程是一个会话的领导进程，+号表示前台进程组。

其中**进程组**表示一组相互关联的进程，子进程是父进程所在组的组员。**会话**指共享同一个控制终端的一个或多个进程组。

用top查看系统资源发现：1）平均负载在逐渐增加，且1分钟内平均负载达到了CPU个数，说明系统可能已经有了性能瓶颈；2）僵尸进程比较多且在不停增加；3）us和sys CPU使用率都不高，iowait却比较高；4）每个进程CPU使用率也不高，但有两个进程处于D状态，可能在等待IO。

分析目前数据可知：iowait过高导致系统平均负载升高，僵尸进程不断增长说明有程序没能正确清理子进程资源。

用dstat来分析，因为它可以同时查看CPU和I/O两种资源的使用情况，便于对比分析。

```c
dstat 1 10    #间隔1秒输出10组数据
```

可以看到当wai（iowait）升高时磁盘请求read都会很大，说明iowait的升高和磁盘的读请求有关。接下来分析到底时哪个进程在读磁盘。

之前top查看的处于D状态的进程号，用pidstat -d -p XXX 展示进程的I/O统计数据。发现处于D状态的进程都没有任何读写操作。在用pidstat -d 查看所有进程的I/O统计数据，看到app进程在进行磁盘读操作，每秒读取32MB的数据。进程访问磁盘必须使用系统调用处于内核态，接下来重点就是找到app进程的系统调用。

```c
sudo strace -p XXX #对app进程调用进行跟踪
```

报错没有权限，因为已经时root权限了。所以遇到这种情况，首先要检查进程状态是否正常。ps命令查找该进程已经处于Z状态，即僵尸进程。

这种情况下top pidstat之类的工具无法给出更多的信息，此时像第5篇一样，用perf record -d和perf report进行分析，查看app进程调用栈。

看到app确实在通过系统调用sys_read()读取数据，并且从new_sync_read和blkdev_direct_IO看出进程时进行直接读操作，请求直接从磁盘读，没有通过缓存导致iowait升高。

通过层层分析后，root cause是app内部进行了磁盘的直接I/O。然后定位到具体代码位置进行优化即可。

#### 僵尸进程

上述优化后iowait显著下降，但是僵尸进程数量仍旧在增加。首先要定位僵尸进程的父进程，通过pstree -aps XXX，打印出该僵尸进程的调用树，发现父进程就是app进程。

查看app代码，看看子进程结束的处理是否正确（是否调用wait()/waitpid(),有没有注册SIGCHILD信号的处理函数等）。

**碰到iowait升高时，先用dstat pidstat等工具确认是否存在磁盘I/O问题，再找是哪些进程导致I/O，不能用strace直接分析进程调用时可以通过perf工具分析。**

**对于僵尸问题，用pstree找到父进程，然后看源码检查子进程结束的处理逻辑即可。**

### CPU性能指标

- CPU使用率
    

- 用户CPU使用率, 包括用户态(user)和低优先级用户态(nice). 该指标过高说明应用程序比较繁忙.
    
- 系统CPU使用率, CPU在内核态运行的时间百分比(不含中断). 该指标高说明内核比较繁忙.
    
- 等待I/O的CPU使用率, iowait, 该指标高说明系统与硬件设备I/O交互时间比较长.
    
- 软/硬中断CPU使用率, 该指标高说明系统中发生大量中断.
    
- steal CPU / guest CPU, 表示虚拟机占用的CPU百分比.
    

- 平均负载
    
    理想情况下平均负载等于逻辑CPU个数,表示每个CPU都被充分利用. 若大于则说明系统负载较重.
    
- 进程上下文切换
    
    包括无法获取资源的自愿切换和系统强制调度时的非自愿切换. 上下文切换本身是保证Linux正常运行的一项核心功能. 过多的切换则会将原本运行进程的CPU时间消耗在寄存器,内核占及虚拟内存等数据保存和恢复上
    
- CPU缓存命中率
    
    CPU缓存的复用情况,命中率越高性能越好. 其中L1/L2常用在单核,L3则用在多核中
    

### 性能工具

- 平均负载案例
    

- 先用uptime查看系统平均负载
    
- 判断负载在升高后再用mpstat和pidstat分别查看每个CPU和每个进程CPU使用情况.找出导致平均负载较高的进程.
    

- 上下文切换案例
    

- 先用vmstat查看系统上下文切换和中断次数
    
- 再用pidstat观察进程的自愿和非自愿上下文切换情况
    
- 最后通过pidstat观察线程的上下文切换情况
    

- 进程CPU使用率高案例
    

- 先用top查看系统和进程的CPU使用情况,定位到进程
    
- 再用perf top观察进程调用链,定位到具体函数
    

- 系统CPU使用率高案例
    

- 先用top查看系统和进程的CPU使用情况,top/pidstat都无法找到CPU使用率高的进程
    
- 重新审视top输出
    
- 从CPU使用率不高,但是处于Running状态的进程入手
    
- perf record/report发现短时进程导致 (execsnoop工具)
    

- 不可中断和僵尸进程案例
    

- 先用top观察iowait升高,发现大量不可中断和僵尸进程
    
- strace无法跟踪进程系统调用
    
- perf分析调用链发现根源来自磁盘直接I/O
    

- 软中断案例
    

- top观察系统软中断CPU使用率高
    
- 查看/proc/softirqs找到变化速率较快的几种软中断
    
- sar命令发现是网络小包问题
    
- tcpdump找出网络帧的类型和来源, 确定SYN FLOOD攻击导致
    

根据不同的性能指标来找合适的工具:

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

图片来自: www.ctq6.cn

在生产环境中往往开发者没有权限安装新的工具包,只能最大化利用好系统中已经安装好的工具. 因此要了解一些主流工具能够提供哪些指标分析.

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

图片来自: www.ctq6.cn

先运行几个支持指标较多的工具, 如top/vmstat/pidstat,根据它们的输出可以得出是哪种类型的性能问题. 定位到进程后再用strace/perf分析调用情况进一步分析. 如果是软中断导致用/proc/softirqs

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

图片来自: www.ctq6.cn

### CPU优化

- 应用程序优化
    

- 编译器优化: 编译阶段开启优化选项, 如gcc -O2
    
- 算法优化
    
- 异步处理: 避免程序因为等待某个资源而一直阻塞,提升程序的并发处理能力. (将轮询替换为事件通知)
    
- 多线程代替多进程: 减少上下文切换成本
    
- 善用缓存: 加快程序处理速度
    

- 系统优化
    

- CPU绑定: 将进程绑定要1个/多个CPU上,提高CPU缓存命中率,减少CPU调度带来的上下文切换
    
- CPU独占: CPU亲和性机制来分配进程
    
- 优先级调整:使用nice适当降低非核心应用的优先级
    
- 为进程设置资源显示: cgroups设置使用上限,防止由某个应用自身问题耗尽系统资源
    
- NUMA优化: CPU尽可能访问本地内存
    
- 中断负载均衡: irpbalance,将中断处理过程自动负载均衡到各个CPU上
    

- TPS、QPS、系统吞吐量的区别和理解
    

- QPS(TPS)
    
- 并发数
    
- 响应时间
    
    QPS(TPS)=并发数/平均相应时间
    
- 用户请求服务器
    
- 服务器内部处理
    
- 服务器返回给客户
    
    QPS类似TPS,但是对于一个页面的访问形成一个TPS,但是一次页面请求可能包含多次对服务器的请求,可能计入多次QPS
    
- QPS (Queries Per Second)每秒查询率,一台服务器每秒能够响应的查询次数.
    
- TPS (Transactions Per Second)每秒事务数,软件测试的结果.
    
- 系统吞吐量, 包括几个重要参数:
    

## 3内存

### Linux内存是怎么工作的

#### 内存映射

大多数计算机用的主存都是动态随机访问内存(DRAM)，只有内核才可以直接访问物理内存。Linux内核给每个进程提供了一个独立的虚拟地址空间，并且这个地址空间是连续的。这样进程就可以很方便的访问内存(虚拟内存)。

虚拟地址空间的内部分为内核空间和用户空间两部分，不同字长的处理器地址空间的范围不同。32位系统内核空间占用1G，用户空间占3G。64位系统内核空间和用户空间都是128T，分别占内存空间的最高和最低处，中间部分为未定义。

并不是所有的虚拟内存都会分配物理内存，只有实际使用的才会。分配后的物理内存通过内存映射管理。为了完成内存映射，内核为每个进程都维护了一个页表，记录虚拟地址和物理地址的映射关系。页表实际存储在CPU的内存管理单元MMU中，处理器可以直接通过硬件找出要访问的内存。

当进程访问的虚拟地址在页表中查不到时，系统会产生一个缺页异常，进入内核空间分配物理内存，更新进程页表，再返回用户空间恢复进程的运行。

MMU以页为单位管理内存，页大小4KB。为了解决页表项过多问题Linux提供了**多级页表**和**HugePage**的机制。

#### 虚拟内存空间分布

用户空间内存从低到高是五种不同的内存段：

- **只读段** 代码和常量等
    
- **数据段** 全局变量等
    
- **堆** 动态分配的内存，从低地址开始向上增长
    
- **文件映射** 动态库、共享内存等，从高地址开始向下增长
    
- **栈** 包括局部变量和函数调用的上下文等，栈的大小是固定的。一般8MB
    

#### 内存分配与回收

##### 分配

malloc对应到系统调用上有两种实现方式：

- **brk()** 针对小块内存(<128K)，通过移动堆顶位置来分配。内存释放后不立即归还内存，而是被缓存起来。
    
- **mmap()**针对大块内存(>128K)，直接用内存映射来分配，即在文件映射段找一块空闲内存分配。
    

前者的缓存可以减少缺页异常的发生，提高内存访问效率。但是由于内存没有归还系统，在内存工作繁忙时，频繁的内存分配/释放会造成内存碎片。

后者在释放时直接归还系统，所以每次mmap都会发生缺页异常。在内存工作繁忙时，频繁内存分配会导致大量缺页异常，使内核管理负担增加。

上述两种调用并没有真正分配内存，这些内存只有在首次访问时，才通过缺页异常进入内核中，由内核来分配

##### 回收

内存紧张时，系统通过以下方式来回收内存：

- 回收缓存：LRU算法回收最近最少使用的内存页面；
    
- 回收不常访问内存：把不常用的内存通过交换分区写入磁盘
    
- 杀死进程：OOM内核保护机制 （进程消耗内存越大oom_score越大，占用CPU越多oom_score越小，可以通过/proc手动调整oom_adj）
    
    ```
    echo -16 > /proc/$(pidof XXX)/oom_adj
    ```
    

#### 如何查看内存使用情况

free来查看整个系统的内存使用情况

top/ps来查看某个进程的内存使用情况

- **VIRT** 进程的虚拟内存大小
    
- **RES** 常驻内存的大小，即进程实际使用的物理内存大小，不包括swap和共享内存
    
- **SHR** 共享内存大小，与其他进程共享的内存，加载的动态链接库以及程序代码段
    
- **%MEM** 进程使用物理内存占系统总内存的百分比
    

### 怎样理解内存中的Buffer和Cache？

**buffer是对磁盘数据的缓存，cache是对文件数据的缓存，它们既会用在读请求也会用在写请求中**

### 如何利用系统缓存优化程序的运行效率

#### 缓存命中率

**缓存命中率**是指直接通过缓存获取数据的请求次数，占所有请求次数的百分比。**命中率越高说明缓存带来的收益越高，应用程序的性能也就越好。**

安装bcc包后可以通过cachestat和cachetop来监测缓存的读写命中情况。

安装pcstat后可以查看文件在内存中的缓存大小以及缓存比例

```
#首先安装Goexport GOPATH=~/goexport PATH=~/go/bin:$PATHgo get golang.org/x/sys/unixgo ge github.com/tobert/pcstat/pcstat
```

#### dd缓存加速

```
dd if=/dev/sda1 of=file bs=1M count=512 #生产一个512MB的临时文件echo 3 > /proc/sys/vm/drop_caches #清理缓存pcstat file #确定刚才生成文件不在系统缓存中，此时cached和percent都是0cachetop 5dd if=file of=/dev/null bs=1M #测试文件读取速度#此时文件读取性能为30+MB/s，查看cachetop结果发现并不是所有的读都落在磁盘上，读缓存命中率只有50%。dd if=file of=/dev/null bs=1M #重复上述读文件测试#此时文件读取性能为4+GB/s，读缓存命中率为100%pcstat file #查看文件file的缓存情况，100%全部缓存
```

#### O_DIRECT选项绕过系统缓存

```
cachetop 5sudo docker run --privileged --name=app -itd feisky/app:io-directsudo docker logs app #确认案例启动成功#实验结果表明每读32MB数据都要花0.9s，且cachetop输出中显示1024次缓存全部命中
```

但是凭感觉可知如果缓存命中读速度不应如此慢，读次数时1024，页大小为4K，五秒的时间内读取了1024*4KB数据，即每秒0.8MB，和结果中32MB相差较大。说明该案例没有充分利用缓存，怀疑系统调用设置了直接I/O标志绕过系统缓存。因此接下来观察系统调用.

```
strace -p $(pgrep app)#strace 结果可以看到openat打开磁盘分区/dev/sdb1，传入参数为O_RDONLY|O_DIRECT
```

这就解释了为什么读32MB数据那么慢，直接从磁盘读写肯定远远慢于缓存。找出问题后我们再看案例的源代码发现flags中指定了直接IO标志。删除该选项后重跑，验证性能变化。

### 内存泄漏，如何定位和处理？

对应用程序来说，动态内存的分配和回收是核心又复杂的一个逻辑功能模块。管理内存的过程中会发生各种各样的“事故”：

- 没正确回收分配的内存，导致了泄漏
    
- 访问的是已分配内存边界外的地址，导致程序异常退出
    

#### 内存的分配与回收

虚拟内存分布从低到高分别是**只读段，数据段，堆，内存映射段，栈**五部分。其中会导致内存泄漏的是：

- 堆：由应用程序自己来分配和管理，除非程序退出这些堆内存不会被系统自动释放。
    
- 内存映射段：包括动态链接库和共享内存，其中共享内存由程序自动分配和管理
    

**内存泄漏的危害比较大，这些忘记释放的内存，不仅应用程序自己不能访问，系统也不能把它们再次分配给其他应用。** 内存泄漏不断累积甚至会耗尽系统内存.

#### 如何检测内存泄漏

预先安装systat，docker，bcc

```
sudo docker run --name=app -itd feisky/app:mem-leaksudo docker logs appvmstat 3
```

可以看到free在不断下降，buffer和cache基本保持不变。说明系统的内存一致在升高。但并不能说明存在内存泄漏。此时可以通过memleak工具来跟踪系统或进程的内存分配/释放请求

```
/usr/share/bcc/tools/memleak -a -p $(pidof app)
```

从memleak输出可以看到，应用在不停地分配内存，并且这些分配的地址并没有被回收。通过调用栈看到是fibonacci函数分配的内存没有释放。定位到源码后查看源码来修复增加内存释放函数即可.

### 为什么系统的Swap变高

系统内存资源紧张时通过内存回收和OOM杀死进程来解决。其中可回收内存包括：

- 缓存/缓冲区，属于可回收资源，在文件管理中通常叫做文件页
    

- 在应用程序中通过fsync将脏页同步到磁盘
    
- 交给系统，内核线程pdflush负责这些脏页的刷新
    
- 被应用程序修改过暂时没写入磁盘的数据(脏页)，要先写入磁盘然后才能内存释放
    

- 内存映射获取的文件映射页，也可以被释放掉，下次访问时从文件重新读取
    

对于程序自动分配的堆内存，也就是我们在内存管理中的匿名页，虽然这些内存不能直接释放，但是Linux提供了Swap机制将不常访问的内存写入到磁盘来释放内存，再次访问时从磁盘读取到内存即可。

#### Swap原理

Swap本质就是把一块磁盘空间或者一个本地文件当作内存来使用，包括换入和换出两个过程：

- 换出：将进程暂时不用的内存数据存储到磁盘中，并释放这些内存
    
- 换入：进程再次访问内存时，将它们从磁盘读到内存中
    

Linux如何衡量内存资源是否紧张？

- **直接内存回收** 新的大块内存分配请求，但剩余内存不足。此时系统会回收一部分内存；
    
- **kswapd0** 内核线程定期回收内存。为了衡量内存使用情况，定义了pages_min,pages_low,pages_high三个阈值，并根据其来进行内存的回收操作。
    

- 剩余内存 < pages_min，进程可用内存耗尽了，只有内核才可以分配内存
    
- pages_min < 剩余内存 < pages_low,内存压力较大，kswapd0执行内存回收，直到剩余内存 > pages_high
    
- pages_low < 剩余内存 < pages_high，内存有一定压力，但可以满足新内存请求
    
- 剩余内存 > pages_high，说明剩余内存较多，无内存压力
    
    pages_low = pages_min _5 / 4 pages_high = pages_min_ 3 / 2
    

#### NUMA 与 SWAP

很多情况下系统剩余内存较多，但SWAP依旧升高，这是由于处理器的NUMA架构。

在NUMA架构下多个处理器划分到不同的Node，每个Node都拥有自己的本地内存空间。在分析内存的使用时应该针对每个Node单独分析

```
numactl --hardware #查看处理器在Node的分布情况，以及每个Node的内存使用情况
```

内存三个阈值可以通过/proc/zoneinfo来查看，该文件中还包括活跃和非活跃的匿名页/文件页数。

当某个Node内存不足时，系统可以从其他Node寻找空闲资源，也可以从本地内存中回收内存。通过/proc/sys/vm/zone_raclaim_mode来调整。

- 0表示既可以从其他Node寻找空闲资源，也可以从本地回收内存
    
- 1，2，4表示只回收本地内存，2表示可以会回脏数据回收内存，4表示可以用Swap方式回收内存。
    

#### swappiness

在实际回收过程中Linux根据/proc/sys/vm/swapiness选项来调整使用Swap的积极程度，从0-100，数值越大越积极使用Swap，即更倾向于回收匿名页；数值越小越消极使用Swap，即更倾向于回收文件页。

**注意：这只是调整Swap积极程度的权重，即使设置为0，当剩余内存+文件页小于页高阈值时，还是会发生Swap。**

#### Swap升高时如何定位分析

```
free #首先通过free查看swap使用情况，若swap=0表示未配置Swap#先创建并开启swapfallocate -l 8G /mnt/swapfilechmod 600 /mnt/swapfilemkswap /mnt/swapfileswapon /mnt/swapfilefree #再次执行free确保Swap配置成功dd if=/dev/sda1 of=/dev/null bs=1G count=2048 #模拟大文件读取sar -r -S 1  #查看内存各个指标变化 -r内存 -S swap#根据结果可以看出，%memused在不断增长，剩余内存kbmemfress不断减少，缓冲区kbbuffers不断增大，由此可知剩余内存不断分配给了缓冲区#一段时间之后，剩余内存很小，而缓冲区占用了大部分内存。此时Swap使用之间增大，缓冲区和剩余内存只在小范围波动停下sar命令cachetop5 #观察缓存#可以看到dd进程读写只有50%的命中率，未命中数为4w+页，说明正式dd进程导致缓冲区使用升高watch -d grep -A 15 ‘Normal’ /proc/zoneinfo #观察内存指标变化#发现升级内存在一个小范围不停的波动，低于页低阈值时会突然增大到一个大于页高阈值的值
```

说明剩余内存和缓冲区的波动变化正是由于内存回收和缓存再次分配的循环往复。有时候Swap用的多，有时候缓冲区波动更多。此时查看swappiness值为60，是一个相对中和的配置，系统会根据实际运行情况来选去合适的回收类型.

### 如何“快准狠”找到系统内存存在的问题

#### 内存性能指标

**系统内存指标**

- 已用内存/剩余内存
    
- 共享内存 （tmpfs实现）
    
- 可用内存：包括剩余内存和可回收内存
    
- 缓存：磁盘读取文件的页缓存，slab分配器中的可回收部分
    
- 缓冲区：原始磁盘块的临时存储，缓存将要写入磁盘的数据
    

**进程内存指标**

- 虚拟内存：5大部分
    
- 常驻内存：进程实际使用的物理内存，不包括Swap和共享内存
    
- 共享内存：与其他进程共享的内存，以及动态链接库和程序的代码段
    
- Swap内存：通过Swap换出到磁盘的内存
    

**缺页异常**

- 可以直接从物理内存中分配，次缺页异常
    
- 需要磁盘IO介入(如Swap)，主缺页异常。此时内存访问会慢很多
    

#### 内存性能工具

根据不同的性能指标来找合适的工具:

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

图片来自: www.ctq6.cn

内存分析工具包含的性能指标:

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

图片来自: www.ctq6.cn

#### 如何迅速分析内存的性能瓶颈

通常先运行几个覆盖面比较大的性能工具，如free，top，vmstat，pidstat等

- 先用free和top查看系统整体内存使用情况
    
- 再用vmstat和pidstat，查看一段时间的趋势，从而判断内存问题的类型
    
- 最后进行详细分析，比如内存分配分析，缓存/缓冲区分析，具体进程的内存使用分析等
    

常见的优化思路：

- 最好禁止Swap，若必须开启则尽量降低swappiness的值
    
- 减少内存的动态分配，如可以用内存池，HugePage等
    
- 尽量使用缓存和缓冲区来访问数据。如用堆栈明确声明内存空间来存储需要缓存的数据，或者用Redis外部缓存组件来优化数据的访问
    
- cgroups等方式来限制进程的内存使用情况，确保系统内存不被异常进程耗尽
    
- /proc/pid/oom_adj调整核心应用的oom_score，保证即使内存紧张核心应用也不会被OOM杀死
    

##### vmstat使用详解

vmstat命令是最常见的Linux/Unix监控工具，可以展现给定时间间隔的服务器的状态值,包括服务器的CPU使用率，内存使用，虚拟内存交换情况,IO读写情况。可以看到整个机器的CPU,内存,IO的使用情况，而不是单单看到各个进程的CPU使用率和内存使用率(使用场景不一样)。

```
vmstat 2procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu----- r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st 1  0      0 1379064 282244 11537528    0    0     3   104    0    0  3  0 97  0  0 0  0      0 1372716 282244 11537544    0    0     0    24 4893 8947  1  0 98  0  0 0  0      0 1373404 282248 11537544    0    0     0    96 5105 9278  2  0 98  0  0 0  0      0 1374168 282248 11537556    0    0     0     0 5001 9208  1  0 99  0  0 0  0      0 1376948 282248 11537564    0    0     0    80 5176 9388  2  0 98  0  0 0  0      0 1379356 282256 11537580    0    0     0   202 5474 9519  2  0 98  0  0 1  0      0 1368376 282256 11543696    0    0     0     0 5894 8940 12  0 88  0  0 1  0      0 1371936 282256 11539240    0    0     0 10554 6176 9481 14  1 85  1  0 1  0      0 1366184 282260 11542292    0    0     0  7456 6102 9983  7  1 91  0  0 1  0      0 1353040 282260 11556176    0    0     0 16924 7233 9578 18  1 80  1  0 0  0      0 1359432 282260 11549124    0    0     0 12576 5495 9271  7  0 92  1  0 0  0      0 1361744 282264 11549132    0    0     0    58 8606 15079  4  2 95  0  0 1  0      0 1367120 282264 11549140    0    0     0     2 5716 9205  8  0 92  0  0 0  0      0 1346580 282264 11562644    0    0     0    70 6416 9944 12  0 88  0  0 0  0      0 1359164 282264 11550108    0    0     0  2922 4941 8969  3  0 97  0  0 1  0      0 1353992 282264 11557044    0    0     0     0 6023 8917 15  0 84  0  0# 结果说明- r 表示运行队列(就是说多少个进程真的分配到CPU)，我测试的服务器目前CPU比较空闲，没什么程序在跑，当这个值超过了CPU数目，就会出现CPU瓶颈了。这个也和top的负载有关系，一般负载超过了3就比较高，超过了5就高，超过了10就不正常了，服务器的状态很危险。top的负载类似每秒的运行队列。如果运行队列过大，表示你的CPU很繁忙，一般会造成CPU使用率很高。- b 表示阻塞的进程,这个不多说，进程阻塞，大家懂的。- swpd 虚拟内存已使用的大小，如果大于0，表示你的机器物理内存不足了，如果不是程序内存泄露的原因，那么你该升级内存了或者把耗内存的任务迁移到其他机器。- free   空闲的物理内存的大小，我的机器内存总共8G，剩余3415M。- buff   Linux/Unix系统是用来存储，目录里面有什么内容，权限等的缓存，我本机大概占用300多M- cache cache直接用来记忆我们打开的文件,给文件做缓冲，我本机大概占用300多M(这里是Linux/Unix的聪明之处，把空闲的物理内存的一部分拿来做文件和目录的缓存，是为了提高 程序执行的性能，当程序使用内存时，buffer/cached会很快地被使用。)- si  每秒从磁盘读入虚拟内存的大小，如果这个值大于0，表示物理内存不够用或者内存泄露了，要查找耗内存进程解决掉。我的机器内存充裕，一切正常。- so  每秒虚拟内存写入磁盘的大小，如果这个值大于0，同上。- bi  块设备每秒接收的块数量，这里的块设备是指系统上所有的磁盘和其他块设备，默认块大小是1024byte，我本机上没什么IO操作，所以一直是0，但是我曾在处理拷贝大量数据(2-3T)的机器上看过可以达到140000/s，磁盘写入速度差不多140M每秒- bo 块设备每秒发送的块数量，例如我们读取文件，bo就要大于0。bi和bo一般都要接近0，不然就是IO过于频繁，需要调整。- in 每秒CPU的中断次数，包括时间中断- cs 每秒上下文切换次数，例如我们调用系统函数，就要进行上下文切换，线程的切换，也要进程上下文切换，这个值要越小越好，太大了，要考虑调低线程或者进程的数目,例如在apache和nginx这种web服务器中，我们一般做性能测试时会进行几千并发甚至几万并发的测试，选择web服务器的进程可以由进程或者线程的峰值一直下调，压测，直到cs到一个比较小的值，这个进程和线程数就是比较合适的值了。系统调用也是，每次调用系统函数，我们的代码就会进入内核空间，导致上下文切换，这个是很耗资源，也要尽量避免频繁调用系统函数。上下文切换次数过多表示你的CPU大部分浪费在上下文切换，导致CPU干正经事的时间少了，CPU没有充分利用，是不可取的。- us 用户CPU时间，我曾经在一个做加密解密很频繁的服务器上，可以看到us接近100,r运行队列达到80(机器在做压力测试，性能表现不佳)。- sy 系统CPU时间，如果太高，表示系统调用时间长，例如是IO操作频繁。- id 空闲CPU时间，一般来说，id + us + sy = 100,一般我认为id是空闲CPU使用率，us是用户CPU使用率，sy是系统CPU使用率。- wt 等待IO CPU时间
```

##### pidstat 使用详解

pidstat主要用于监控全部或指定进程占用系统资源的情况,如CPU,内存、设备IO、任务切换、线程等。

使用方法：

- pidstat –d interval times 统计各个进程的IO使用情况
    
- pidstat –u interval times 统计各个进程的CPU统计信息
    
- pidstat –r interval times 统计各个进程的内存使用信息
    
- pidstat -w interval times 统计各个进程的上下文切换
    
- p PID 指定PID
    

1、统计IO使用情况

```
pidstat -d 1 1003:02:02 PM   UID       PID   kB_rd/s   kB_wr/s kB_ccwr/s  Command03:02:03 PM     0       816      0.00    918.81      0.00  jbd2/vda1-803:02:03 PM     0      1007      0.00      3.96      0.00  AliYunDun03:02:03 PM   997      7326      0.00   1904.95    918.81  java03:02:03 PM   997      8539      0.00      3.96      0.00  java03:02:03 PM     0     16066      0.00     35.64      0.00  cmagent03:02:03 PM   UID       PID   kB_rd/s   kB_wr/s kB_ccwr/s  Command03:02:04 PM     0       816      0.00   1924.00      0.00  jbd2/vda1-803:02:04 PM   997      7326      0.00  11156.00   1888.00  java03:02:04 PM   997      8539      0.00      4.00      0.00  java
```

- UID
    
- PID
    
- kB_rd/s: 每秒进程从磁盘读取的数据量 KB 单位 read from disk each second KB
    
- kB_wr/s: 每秒进程向磁盘写的数据量 KB 单位 write to disk each second KB
    
- kB_ccwr/s: 每秒进程向磁盘写入，但是被取消的数据量，This may occur when the task truncates some dirty pagecache.
    
- iodelay: Block I/O delay, measured in clock ticks
    
- Command: 进程名 task name
    

2、统计CPU使用情况

```
# 统计CPUpidstat -u 1 1003:03:33 PM   UID       PID    %usr %system  %guest    %CPU   CPU  Command03:03:34 PM     0      2321    3.96    0.00    0.00    3.96     0  ansible03:03:34 PM     0      7110    0.00    0.99    0.00    0.99     4  pidstat03:03:34 PM   997      8539    0.99    0.00    0.00    0.99     5  java03:03:34 PM   984     15517    0.99    0.00    0.00    0.99     5  java03:03:34 PM     0     24406    0.99    0.00    0.00    0.99     5  java03:03:34 PM     0     32158    3.96    0.00    0.00    3.96     2  ansible
```

- UID
    
- PID
    
- %usr: 进程在用户空间占用 cpu 的百分比
    
- %system: 进程在内核空间占用 CPU 百分比
    
- %guest: 进程在虚拟机占用 CPU 百分比
    
- %wait: 进程等待运行的百分比
    
- %CPU: 进程占用 CPU 百分比
    
- CPU: 处理进程的 CPU 编号
    
- Command: 进程名
    

3、统计内存使用情况

```
# 统计内存pidstat -r 1 10Average:      UID       PID  minflt/s  majflt/s     VSZ    RSS   %MEM  CommandAverage:        0         1      0.20      0.00  191256   3064   0.01  systemdAverage:        0      1007      1.30      0.00  143256  22720   0.07  AliYunDunAverage:        0      6642      0.10      0.00 6301904 107680   0.33  javaAverage:      997      7326     10.89      0.00 13468904 8395848  26.04  javaAverage:        0      7795    348.15      0.00  108376   1233   0.00  pidstatAverage:      997      8539      0.50      0.00 8242256 2062228   6.40  javaAverage:      987      9518      0.20      0.00 6300944 1242924   3.85  javaAverage:        0     10280      3.70      0.00  807372   8344   0.03  aliyun-serviceAverage:      984     15517      0.40      0.00 6386464 1464572   4.54  javaAverage:        0     16066    236.46      0.00 2678332  71020   0.22  cmagentAverage:      995     20955      0.30      0.00 6312520 1408040   4.37  javaAverage:      995     20956      0.20      0.00 6093764 1505028   4.67  javaAverage:        0     23936      0.10      0.00 5302416 110804   0.34  javaAverage:        0     24406      0.70      0.00 10211672 2361304   7.32  javaAverage:        0     26870      1.40      0.00 1470212  36084   0.11  promtail
```

- UID
    
- PID
    
- Minflt/s : 每秒次缺页错误次数 （minor page faults），虚拟内存地址映射成物理内存地址产生的 page fault 次数
    
- Majflt/s : 每秒主缺页错误次数 (major page faults), 虚拟内存地址映射成物理内存地址时，相应 page 在 swap 中
    
- VSZ virtual memory usage : 该进程使用的虚拟内存 KB 单位
    
- RSS : 该进程使用的物理内存 KB 单位
    
- %MEM : 内存使用率
    
- Command : 该进程的命令 task name
    

4、查看具体进程使用情况

```
pidstat -T ALL -r -p 20955 1 1003:12:16 PM   UID       PID  minflt/s  majflt/s     VSZ    RSS   %MEM  Command03:12:17 PM   995     20955      0.00      0.00 6312520 1408040   4.37  java03:12:16 PM   UID       PID minflt-nr majflt-nr  Command03:12:17 PM   995     20955         0         0  java
```

版权归原作者所有，如有侵权，请联系删除。  

  

- END -

---

  

**看完一键三连********在看********，**转发****，********点赞****

**是对文章最大的赞赏，极客重生感谢你****![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)**

  

推荐阅读

  

[

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

定个目标｜建立自己的技术知识体系







](https://mp.weixin.qq.com/s?__biz=MzkyMTIzMTkzNA==&mid=2247570933&idx=1&sn=04bf666a7a402ed65b83020dc28dd543&chksm=c18522a4f6f2abb27ca1d09eaf646cc7eca2feea9c7f39618d3048b87634695955a8831992a9&scene=21#wechat_redirect)

  

[

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

从C10K到C10M高性能网络的探索与实践







](https://mp.weixin.qq.com/s?__biz=MzkyMTIzMTkzNA==&mid=2247570691&idx=1&sn=b449e2832a49c7201fda8ee340090160&chksm=c1852252f6f2ab44d77f1dddecdb49c297254eb1c4c4957f4c5986bd880f3ba91820fc4deb08&scene=21#wechat_redirect)

  

[

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

性能优化实战｜使用eBPF代替iptables优化服务网格数据面性能







](https://mp.weixin.qq.com/s?__biz=MzkyMTIzMTkzNA==&mid=2247571000&idx=1&sn=6514fc2b281426783e36a5069875e3d6&chksm=c1852169f6f2a87f9264b2304428f33d9b9b0f03424e48d6e229eeef0f3f5d2adf4afa0676fb&scene=21#wechat_redirect)

  

  

你好，这里是极客重生，我是阿荣，大家都叫我荣哥，从华为->外企->到互联网大厂，目前是大厂资深工程师，多次获得五星员工，多年职场经验，技术扎实，专业后端开发和后台架构设计，热爱底层技术，丰富的实战经验，分享技术的本质原理，希望帮助更多人蜕变重生，拿BAT大厂offer，培养高级工程师能力，成为技术专家，实现高薪梦想，期待你的关注！[**点击蓝字查看我的成长之路**。](http://mp.weixin.qq.com/s?__biz=MzkyMTIzMTkzNA==&mid=2247564006&idx=1&sn=8c88b0d7222ce0e310a012e90bff961f&chksm=c1850db7f6f284a133be27ad27ecd965c3c942583f69bd1a26c34ebf179b4cd6ceb3d7e787b4&scene=21#wechat_redirect)

  

校招/社招/简历/面试技巧/大厂技术栈分析/后端开发进阶/优秀开源项目/直播分享/技术视野/实战高手等, [极客星球](http://mp.weixin.qq.com/s?__biz=MzkyMTIzMTkzNA==&mid=2247568828&idx=1&sn=ec346110ca42539f5d797f94eee57e9a&chksm=c1853aedf6f2b3fbf80600f4f71059ce2eddcb7c3e900953014764a3d16d2fd77f8190303898&scene=21#wechat_redirect)希望成为最有技术价值星球，尽最大努力为星球的同学提供技术和成长帮助！详情查看->[极客星球](http://mp.weixin.qq.com/s?__biz=MzkyMTIzMTkzNA==&mid=2247568828&idx=1&sn=ec346110ca42539f5d797f94eee57e9a&chksm=c1853aedf6f2b3fbf80600f4f71059ce2eddcb7c3e900953014764a3d16d2fd77f8190303898&scene=21#wechat_redirect)  

  

![](http://mmbiz.qpic.cn/mmbiz_png/cYSwmJQric6m7W7KicZrc94icQe4d8ItHFyONKlBkGVqEAiavUicEzmEszR5aPvicKDeCMy8aAw6lPFe8AGhHQic1UKaA/300?wx_fmt=png&wxfrom=19)

**极客重生**

技术学习分享，一起进步

180篇原创内容

公众号

                                                                求点赞，在看，分享三连![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

深入理解Linux系统51

精通后端开发67

深入理解Linux系统 · 目录

上一篇内推|底层翻身的机会来了，快来看一看！下一篇eBPF在大厂的应用

阅读 4061

​

写留言

**留言 7**

- 周俊诚
    
    2022年2月18日
    
    赞1
    
    荣哥 666
    
- 乐观主义现代人
    
    2022年4月9日
    
    赞
    
    荣哥，能不能再细点，比如我知道很多看CPU数据的方法，到底多少我才觉得我这个CPU有压力，io多少的时候我知道磁盘读写跟不上导致的。我得到数据容易，关键是我看到这些数据，哪个是异常的呢。这种标准或者说你的经验值能不能说下。再比如我不管你内存多少，可用内存剩不到10%我就认为你的内存不够了，谢谢大佬
    
- arsenaler
    
    2022年2月18日
    
    赞
    
    很干，收藏![[呲牙]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
- 许玉康
    
    2022年2月18日
    
    赞
    
    6666
    
- abelian(王立强)
    
    2022年2月18日
    
    赞
    
    如果程序一直运行在用户态，比如计算一个很复杂的逻辑或者死循环，一段很长时间不会进入内核态，操作系统如何做到调度
    
- bin
    
    2022年2月17日
    
    赞
    
    荣哥太硬核了，基本涵盖全了，赶快收藏起来
    
- 凉凉
    
    2022年2月17日
    
    赞
    
    支持荣哥![[抱拳]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![[抱拳]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![[抱拳]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![[社会社会]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![[社会社会]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    

已无更多数据

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/cYSwmJQric6m7W7KicZrc94icQe4d8ItHFyONKlBkGVqEAiavUicEzmEszR5aPvicKDeCMy8aAw6lPFe8AGhHQic1UKaA/300?wx_fmt=png&wxfrom=18)

极客重生

31821

7

写留言

**留言 7**

- 周俊诚
    
    2022年2月18日
    
    赞1
    
    荣哥 666
    
- 乐观主义现代人
    
    2022年4月9日
    
    赞
    
    荣哥，能不能再细点，比如我知道很多看CPU数据的方法，到底多少我才觉得我这个CPU有压力，io多少的时候我知道磁盘读写跟不上导致的。我得到数据容易，关键是我看到这些数据，哪个是异常的呢。这种标准或者说你的经验值能不能说下。再比如我不管你内存多少，可用内存剩不到10%我就认为你的内存不够了，谢谢大佬
    
- arsenaler
    
    2022年2月18日
    
    赞
    
    很干，收藏![[呲牙]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
- 许玉康
    
    2022年2月18日
    
    赞
    
    6666
    
- abelian(王立强)
    
    2022年2月18日
    
    赞
    
    如果程序一直运行在用户态，比如计算一个很复杂的逻辑或者死循环，一段很长时间不会进入内核态，操作系统如何做到调度
    
- bin
    
    2022年2月17日
    
    赞
    
    荣哥太硬核了，基本涵盖全了，赶快收藏起来
    
- 凉凉
    
    2022年2月17日
    
    赞
    
    支持荣哥![[抱拳]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![[抱拳]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![[抱拳]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![[社会社会]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![[社会社会]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    

已无更多数据