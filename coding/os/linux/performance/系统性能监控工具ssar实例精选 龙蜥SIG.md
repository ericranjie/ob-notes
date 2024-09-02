# 

Linux阅码场

 _2021年11月25日 07:03_

以下文章来源于OpenAnolis龙蜥 ，作者闻茂泉、李靖轩

[

![](http://wx.qlogo.cn/mmhead/Q3auHgzwzM4bZCboj0qz7G2rqBCKMh4iblVGkMN9XKUxiaECBVF6nj1g/0)

**OpenAnolis龙蜥**.

龙蜥社区（OpenAnolis）是立足中国面向国际的 Linux 服务器操作系统开源根社区及创新平台。

](https://mp.weixin.qq.com/s?__biz=Mzg2OTc0ODAzMw==&mid=2247502803&idx=1&sn=709db7f06081af73177c2d2993f03fd2&source=41&key=daf9bdc5abc4e8d05fb4e1ce7572f25187a4bdf98413b11d17b7d296f7c0a8ead604c6279a5fdc68478baa7056dee6a816daa68b46e5cdc4e37745850ad62e5357353015561bc164d341f170d52442b4b02c0bb549c68cd7a48b61bce09700aaac63756dce0f6e0cf64664b4acb121f6bafbbd8632110d1fefe34c04c6975f96&ascene=0&uin=MTEwNTU1MjgwMw%3D%3D&devicetype=Windows+11+x64&version=63090b19&lang=zh_CN&countrycode=CN&exportkey=n_ChQIAhIQtCZB%2F0EQCYtZ7uiQ0Ev0BhLmAQIE97dBBAEAAAAAAP3vDB3ExKUAAAAOpnltbLcz9gKNyK89dVj04ViatOwdSkycmFBjVcmREhqJCODXY7lk1RZ2l107J018vWUj9hem%2FgMFch4ofXHzNgUtYIuCgi%2Bhf%2BPG2SSkuh%2BeWlFKYt0V0WSLdujTDX0fepp7tDCLIlVrCXOSEAJ%2BwnxhfSgokYFjPrs3%2Bfiw2%2Faq6L47gRHiytmXavHH19yxs6bLtVnQtjBugG1L6XyKGjtEAi9b%2BbsHwH7643c9GVv7q0g7mcd34cE0Ev%2FGudYavSkKhGgKZkUJKroyzqQ%2F&acctmode=0&pass_ticket=tH4V4QcsOMmoMNwUPvizh7Af8SbvIXUpGf3p4CXyc4Fsi2qaI4b1YcA%2Bpfv8%2B5TK&wx_header=1#)

**跟踪诊断技术 SIG 致力于为操作系统生态提供系统性，工具化，并以数据为支撑的发现、跟踪和诊断问题的能力。**

**SIG目标：为龙蜥社区（OpenAnolis****）开源操作系统，提供一个全栈覆盖内核与核心组件的跟踪和诊断工具，增强龙蜥社区（OpenAnolis）全栈的可观察性和可靠性。**

**作者广成（闻茂泉）、无牙（李靖轩）分别是龙蜥社区**跟踪诊断技术 SIG** 核心成员。  
**

欢迎更多开发者加入跟踪诊断技术SIG：

网址：_https://openanolis.cn/sig/tracing_

邮件列表：_cloud-kernel@lists.openanolis.cn_

![图片](https://mmbiz.qpic.cn/mmbiz_png/xHicTWic3y1DskJOic8Wjfg87ib4ZtSljR5oF5hzgfqNVM5wus6BUic16kmrib746FgMNf6ZEKMs3TRJMVPLibbX7dXyg/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

### **前言**  

ssar 是阿里自研并贡献至龙蜥社区的一款系统性能监控工具。该工具系统开销很小，已经在公司部分业务规模化部署，**且持续稳定运行 1 年以上。**

ssar 工具近期在国内领先的操作系统开源社区及创新平台龙蜥社区开源，欢迎大家访问龙蜥社区（_**https://openanolis.cn/sig/tracing**_）下载 ssar 工具源码适用。源码中提供了相关的编译和安装步骤，还提供了具体《使用参考手册》。

ssar 工具在日常使用中也解决了很多实际的生产问题，为了能让大家更快速的了解 ssar 的用途，**这里特别从案例使用的角度挑选了工具的十大典型使用场景进行介绍。**

### **一、定位 CPU 和内存波动时的进程级因素**

日常运维中，整机 CPU 和内存经常出现较大的增长，从而引起系统不稳定。此时特别需要能回朔历史数据，查看指标波动时刻的进程级 CPU 和内存等影响因素。

**ssar 查看整机 CPU 和内存变化趋势方法**。其中 user、system 和 anonpages 分别是内核用于指代用户空间 CPU 使用、内核空间 CPU 使用和匿名页内存的指标名。

```
$ ssar -i 1 -o user,system,anonpages
```

**ssar 查看进程级 CPU 影响因素的方法**。通过 procs 子命令可以显示出 2021-09-08T18:38:00 到 2021-09-08T18:40:00 这 2 分钟内，内核态 CPU 使用最多的 3 个进程列表。pucpu、pscpu 和 pcpu 分别表示该进程的用户空间 CPU 利用率，内核空间 CPU 利用率和总 CPU 利用率。

```
$ ssar procs -f 2021-09-08T18:40:00 -r 2 -o pid,ppid,pcpu,pucpu,pscpu,cmd -k pscpu -l 3
```

**ssar 查看进程级内存影响因素的方法**。通过 procs 子命令可以显示出 2021-09-08T18:36:00 到 2021-09-08T18:38:00 这 2 分钟内，内存申请增量最多的 3 个进程列表。rss_dlt 表示 rss 内存的增量值。

```
$ ssar procs -f 2021-09-08T18:38:00 -r 2 -o pid,ppid,rss,rss_dlt,nlwp,cmd -k rss_dlt -l 3
```

### **二、跟踪单进程的CPU和内存波动详情**

通过上面的方法，发掘出了影响整机指标波动的进程，还可以进一步通过 proc 子命令更进一步追踪特定进程的 CPU 和内存变化情况。其中左尖括号（<）表示 2021-08-09T11:39:00 时刻该 sresar 进程还没有启动，右尖括号（>）表示 2021-08-09T11:47:00 时刻该 sresar 进程已经结束。

```
$ ssar proc -p $(pidof sresar)  -i1 -o collect_datetime,rss,min_flt,cmd
```

### **三、记录5秒级精度的系统load信息**

ssar 的 load5s 子命令不仅将 load 的精度提高到了 5s，还创造性增加了load5s 指标，**load5s 指标比 load1、load5 和 load15 指标更加精准**，更能反映系统当时的压力情况。这里明显可以看出传统的 load1 指标在系统负载压力消失后，还一定的滞后性，但load5s指标却可以精准的显示机器的负载压力。

```
$ ssar load5s
```

### **四、记录线程D状态异常的stack详情**

对于 load5s 值大于 CPU 核数一定倍数的情况，会触发 load 详情的采集，其中针对 D 状态线程会抽样记录内核调用栈。使用 load2p 可以显示详细的 D 状态调用栈信息。

```
$ ssar load2p -c 2021-09-08T17:25:00 --stackinfo
```

### **五、诊断整机高阶内存不足情况**

系统内存不足相关的问题，一直是日常运维中最常见的问题之一。有时候我们会从系统日志中看到类似这样的日志信息“page allocction failure: order:4”，这是 free 内存不足，并且伴随内存碎片化，内核空间申请不到高阶内存的表现。         

定位这类问题需先对整机内存不足程度做一个诊断，anonpages 指标是匿名页，memfree 指标是整机 free 内存， pgsteal_kswapd 指标是每秒异步内存回收数量，pgsteal_direct 是每秒直接内存回收数量。

```
$ ssar -i 1 -o anonpages,memfree,pgsteal_kswapd,pgsteal_direct
```

可以看到，随着系统的 anonpages 匿名页内存的增长，memfree 空闲内存逐步减少，到达一定阈值后，开始异步内存回收 pgsteal_kswapd，空闲内存进一步减少进一步引起直接内存回收 pgsteal_direct。

在这样空闲内存严重不足的前提下，可能会进一步引发内核高阶内存不足的情况发生。从数据中可以看到 2021-09-09T10:46:00 时刻的 order4 内存库存为0。

```
$ ssar -i 1 --node0
```

### **六、定位 Cgroup 级别的内存颠簸问题**

在 cgroup 层面出现内存不足时，也有一个内存颠簸问题，会引起系统严重的问题。

```
$ tsar2 --io -i 1 -I nvme0n1p1 -s rs,rarqsz,util
```

性能优越的 nvme 磁盘，有时候磁盘 util 也会被打满到 100%，可以观察到相应的磁盘读 IO 高达数万，同时单次读 IO 大小只有 10 多个 Kb 大小。造成这种情况的最常见原因之一是 cgroup 级别的内存颠簸。通过内核指标 pgmajfault 可以看到在 IO 打满的同时每秒整机主缺页中断数也非常高，这正是造成系统 IO 打满的直接原因。

```
$ ssar -i 1 -o pgmajfault
```

进一步，通过 ssar 的 procs 子命令，可以定位到发生大量主缺页中断的正是 pid 为 58532 和 46873 这 2 个 java 进程。引起 java 进程大量发生主缺页中断的原因是进程 rss 内存极度逼近 cgroup 组设置的memory.limit_in_bytes 值，导致 cgroup 组内留给 clean 状态的代码段内存空间不足以完全加载所有程序代码段。所以代码段在程序执行中，只能不断的被丢弃再从磁盘读取。

### **七、网络 TCP 重传扩展信息**

在主机网络层面，日常运维中也会受到网络丢包、重传和乱序等问题的困扰。这里也提供了几组更加深入的网络指标供诊断网络问题使用。

```
$ tsar2 --retran
```

在 tcp 重传方面，我们提供了许多相比 tcp 重传率之外，更为细致的指标。现代 Linux 系统，早已经有除 200ms 超时重传之外的多种重传的方式来加速网络抖动或者异常状态下的恢复。这里我们将 TCP 的重传进行了更细致的分类，包括快重传、尾包重传和超时重传，不同的重传对于业务的影响是完全不同的，相当多的时候，我们可能观察到快重传或者尾包重传很高，但超时重传并不多，这个时候往往并不影响业务。而一旦出现大量的超时重传，往往意味着真正的网络异常。

--tcpofo 可以让我们看到当前收到的一些乱序的情况，乱序跟重传往往是关联的，一般情况下，收到乱序报文会看到 OFOQueue 增加，而一旦看到 OFODrop，则说明收到的乱序的报文被丢失了，**这个时候很可能是我们队列中乱序报文太多**，或者我们应用进程没有及时将 tcp 接收队列中的数据包收走。

以上数据再结合 ssar 的历史数据，找到某些重传或乱序的异常点，可以帮我们很快定位一些 tcp 问题的原因。

```
$ tsar2 --tcpdrop
```

对于 TCP 的一些典型丢包和异常，可以通过 --tcpdrop/--tcperr 来观察，譬如上面我们看到在 08/09/21-16:55 时，有较多的 ListenDrops 和 ListenOverflows，说明有较多的 TCP SYN 建连请求，导致 TCP 的 accept 队列满了，这种情况说明到达服务端的新建连接请求比服务端 accept() 消费新的连接请求的速度要快，有可能是突发来了较多的建连请求，也可能是服务端长时间没有调用 accept() 来消费 accept 队列中的半连接请求。

### **八、自定义采集任意系统指标**

做驱动开发的同学很可能希望能记录自己的驱动模块中一些性能数据，ssar 工具为这种场景提供了简单和完善的采集方案。只需要驱动开发同学，将内核模块中的性能指标通过/proc/或/sys/下的伪文件方式暴露为用户空间即可。数据可以是累积值，也可以是瞬时值，例如 mydev 伪文件中有 2 个值，左侧的indicator1 是累积值，右侧的 indicator2 是瞬时值。

```
$ cat /proc/mydev
```

ssar 工具配置采集的方法比较简单。

```
$ cat /etc/ssar/sys.conf 
```

修改完配置完后，可以使用如下命令获取历史数据。

```
$ ssar -i 1 -f 2021-09-09T17:20:00 -r 60 -o 'metric=d:cfile=mydev:line=1:column=1:alias=indicator1;metric=c:cfile=mydev:line=1:column=2:alias=indicator2'
```

命令略微改变下，获取实时数据。

```
$ ssar -i 1s -f +10 -o 'metric=d|cfile=mydev|line=1|column=1|alias=indicator1;metric=c|cfile=mydev|line=1|column=2|alias=indicator2'
```

不安装ssar软件包，只拷贝 ssar 程序，也可以直接使用获取实时数据。

```
$ ./ssar -i 1s -f +10 -o 
```

对于多行或者多列指标含义相近的情况，也可以参考如下 2 个用法。

`$ ssar -o 'metric=d:cfile=stat:line=2:column=2-``11:alias=cpu0_{column};' # cpu0第2到11列数据` `$ ssar -o 'metric=d|cfile=stat|line=2-``17|column=5|alias=idle_{line};'   # cpu0到15的idle`

ssar 还支持指标不是整数的情况，支持实数和字符串形式的指标，参考如下用法。

`$ ssar -o 'metric=c:cfile=loadavg:line=1:column=1:dtype=float:alias=load1'``# load1实型数据``$ ssar -o 'metric=c|cfile=loadavg|line=1|column=4|dtype=string|alias=rp'``# rp字符串信息`

如果只想采集这个驱动指标文件mydev，ssar还提供了2个配置参数load5s_flag 和 proc_flag 来关闭load和进程级指标采集。节省不必要的磁盘存储空间开销。

### **九、诊断特定CPU中断不均情况**

特定 CPU 的 softirq 资源消耗过多，会影响此 CPU 上用于用户空间的资源使用。因此当中断分布不均时，会严重影响用户空间进程的执行。首先需要了解 softirq 资源消耗在各个 CPU 之间是否均衡。

```
$ tsar2 --cpu -I cpu,cpu0,cpu1,cpu2,cpu3,cpu4,cpu5,cpu6,cpu7 -s sirq -i 1
```

通过命令可以看到，softirq 在各个 CPU 之间分布并不均匀，cpu5 的 softirq高达百分之十几之多。针对更多核 CPU 的情况，可以使用 tsar2 cputop 子命令排序输出 softirq 占用比最高的 CPU。

tsar2 irqtop 即可快速定位引起 cpu5 的 softirq 资源占比高的原因是中断名称为 virtio-output.1 的 158 号中断，并且显示出每秒平均中断数。

```
$ tsar2 irqtop -i 1 -C 5   
```

默认情况下 CPU 级别的中断情况不开启历史采集，需要将配置文件中 interrupts 部分修改为如下配置开启。

```
{src_path='interrupts', turn=true, gzip=true},
```

如果需要获取实时数据，可将上面命令增加-l选项即可。

`$ tsar2 --cpu -I cpu,cpu0,cpu1,cpu2,cpu3,cpu4,cpu5,cpu6,cpu7 -s sirq -i 1 -l``$ tsar2 irqtop -i 1 -C 5 -l`

### **十、支持灵活的数据输出和解析**

ssar 不仅数据指标非常丰富，而且输出格式上也非常易于使用和解析。ssar 的所有数据输出，都支持支持 json 格式输出，便于 python 等高级语言的解析。使用方法就是在命令上增加一个 --api 选项。

`$ ssar -r 5 -o user,system,memfree,anonpages --api``[{"anonpages":"280824","collect_datetime":"2021-09-10T12:00:00","memfree":"181753024","system":"-","user":"-"},{"anonpages":"280812","collect_datetime":"2021-09-10T12:05:00","memfree":"181752704","system":"2.35","user":"1.22"}]`

对于 shell 等脚本解析，ssar 也提供了便利，-H 选项可以隐藏掉各种 header 信息。

`$ ssar -r 5 -o user,system,memfree,anonpages -H``2021-09-10T12:00:00          -          -  81754836     280296` `2021-09-10T12:05:00       1.29       2.57  81756712     280228`

对于数据项中出现的各种“-”、“*”和“<”等符号信息，提供-P 选项参数隐藏输出，可以让上层调用脚本减少处理异常数据的工作。

```
$ ssar -r 5 -o user,system,memfree,anonpages -H -P
```

**ssar 工具代码库地址：**

_**https://codeup.openanolis.cn/codeup/tracing_diagnosis/ssar.git**_

—— 完 ——  

**关于龙蜥社区SIG**

SIG是开放的，并争取让交付成果成为社区发行的一部分，由组内核心成员主导治理，可通过邮件列表和组内的成员进行交流。龙蜥社区SIG目前已超20个，包括跟踪诊断技术SIG、商密软件栈、高性能存储技术SIG、Java语言与虚拟机SIG、Cloud Kernel、OceanBase SIG等。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**SIG网址：https://openanolis.cn/sig**

阅读原文

阅读 2190

​

写留言

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/YHBSoNHqDiaH5V6ycE2TlM9McXhAbaV9aUBaTAibVhLibXH1rL4JTMb2eoQfdCicTm4wycOsopO82jG3szvYF3mTJw/300?wx_fmt=png&wxfrom=18)

Linux阅码场

9分享3

写留言

写留言

**留言**

暂无留言