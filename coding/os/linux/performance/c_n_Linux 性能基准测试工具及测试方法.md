
Linux云计算网络 _2024年03月29日 08:13_ _广东_

以下文章来源于SRE运维进阶之路 ，作者Clay

功能上线前，基准测试的重要性，这篇文章着重介绍一下「Linux 性能基准测试工具及测试方法」

还是老规矩，先请性能领域的大师布伦丹·格雷格（Brendan Gregg）登场 👏👏👏

![[Pasted image 20241024224423.png]]

linux_benchmarking_tools

整理测试指标如下图

![[Pasted image 20241024224436.png]]

> 测试环境说明：CentOS7， 4c8g

# CPU

**Super_Pi** 是一种用于计算圆周率π的程序，通常用于测试计算机性能和稳定性。它的主要用途是测量系统的单线程性能，因为它是一个单线程应用程序。

`# 安装 bc   yum -y install bc   # 测试   time echo "scale=5000; 4*a(1)" | bc -l -q &>1   `

`# 结果分析，看 real 即可，时间越短，性能越好   `

![[Pasted image 20241024224451.png]]

**sysbench** 素数计算

`# 安装 sysbench   yum -y install sysbench   # 测试方法: 启动4个线程计算10000事件所花的时间   sysbench cpu --threads=4 --events=10000 --time=0  run   `

`# 结果分析，看 total time 即可，时间越短，性能越好   `

![[Pasted image 20241024224501.png]]

# 内存

**内存带宽(stream)**

Stream测试是内存测试中业界公认的内存带宽性能测试基准工具

`# 编译安装 STREAM   yum -y install gcc gcc-gfortran   git clone https://github.com/jeffhammond/STREAM.git   cd STREAM/   make   # 指定线程数   export OMP_NUM_THREADS=1   ./stream_c.exe   `

`# 结果分析，看 Copy、Scale、Add、Triad，数值越大，性能越好   `

![[Pasted image 20241024224523.png]]

# 磁盘 IO

> ⚠️  测试时请准备裸的数据盘，测试完成后请重新格式化磁盘

测试方法和结果分析和文件 IO 测试相同，`--filename`  改为具体的数据盘即可，比如`/dev/sda` ，这里不再赘述

# 文件 IO

**磁盘读、写iops**

iops：磁盘的每秒读写次数，这个是随机读写考察的重点

`# 安装   yum -y install fio   # 测试随机读 IOPS   fio --ioengine=libaio --bs=4k --direct=1 --thread --time_based --rw=randread --filename=/home/randread.txt --runtime=60 --numjobs=1 --iodepth=1 --group_reporting --name=randread-dep1 --size=1g   # 测试随机写 IOPS   fio --ioengine=libaio --bs=4k --direct=1 --thread --time_based --rw=randwrite --filename=/home/randwrite.txt --runtime=60 --numjobs=1 --iodepth=1 --group_reporting --name=randread-dep1 --size=1g   `

`# 结果分析，看 IOPS 即可，值越大，性能越好   `

![[Pasted image 20241024224539.png]]

**磁盘读、写带宽**

bw：磁盘的吞吐量，这个是顺序读写考察的重点

`# 测试顺序读   fio --ioengine=libaio --bs=4k --direct=1 --thread --time_based --rw=read --filename=/home/read.txt --runtime=60 --numjobs=1 --iodepth=1 --group_reporting --name=randread-dep1 --size=1g   # 测试顺序写   fio --ioengine=libaio --bs=4k --direct=1 --thread --time_based --rw=write --filename=/home/write.txt --runtime=60 --numjobs=1 --iodepth=1 --group_reporting --name=randread-dep1 --size=1g   `

`# 结果分析，看 BW 即可，值越大，性能越好   `

![[Pasted image 20241024224616.png]]

> ⚠️  **因地制宜，灵活选取。在基准测试时，一定要注意根据应用程序 I/O 的特点，来具体评估指标。**
>
> 比如 etcd  磁盘性能衡量指标为：WAL 文件系统调用 fsync 的延迟分布，当 99% 样本的同步时间小于 10 毫秒就可以认为存储性能能够满足 etcd 的性能要求。
>
> `mkdir etcd-bench` `fio --rw=write --ioengine=sync --fdatasync=1 --directory=etcd-bench --size=22m --bs=2300 --name=etcd-bench`
>
> !\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

## 网络

**传输速率(pps)**

`# server & client 编译安装 netserver   wget -c "https://codeload.github.com/HewlettPackard/netperf/tar.gz/netperf-2.5.0" -O netperf-2.5.0.tar.gz   yum -y install gcc cc    tar zxvf netperf-2.5.0.tar.gz   cd netperf-netperf-2.5.0   ./configure && make && make install      # server 端启动 netserver   netserver   # 监控数据   sar -n DEV 5      # client 端测试   netperf -t UDP_STREAM -H <server ip> -l 100 -- -m 64 -R 1 &   # 监控数据   sar -n DEV 5   `

`# 结果分析，看 rxpck/s,txpck/s 值即可，值越大，性能越好   `

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**网络带宽**

`# server 端启动 netserver   netserver   # 监控数据   sar -n DEV 5       # client 端测试   netperf -t TCP_STREAM -H <server ip> -l 100 -- -m 1500 -R 1 &   # 监控数据   sar -n DEV 5   `

`# 结果分析，看 rxkB/s,txkB/s 值即可，值越大，性能越好   `

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

## Nginx

`# 安装 ab 工具   yum -y install httpd-tools      # 编译安装 wrk   git clone https://github.com/wg/wrk.git   make   cp wrk /usr/local/bin/       # 测试，-c表示并发连接数1000，-t表示线程数为2，-d 表示测试时间   wrk -t12 -c400 -d30s <URL>   `

`# 结果分析，Requests/sec 为 QPS   `

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

## 自动化压测脚本

> 压测需要大量采样，并实时观察

`git clone https://github.com/clay-wangzhi/bench.git   bash bench.sh   `

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

更多测试方法，详见 https://github.com/clay-wangzhi/bench

![](http://mmbiz.qpic.cn/mmbiz_png/1TDxR6xkRSFD7ibPKdP9JOqxFxp199B6LLCw80xbTAicxIQPgpsq5ib8kvx7eKfq6erLb1CAT4ZKa3RSnl9IWfELw/300?wx_fmt=png&wxfrom=19)

**Linux云计算网络**

专注于 「Linux」 「云计算」「网络」技术栈，分享的干货涉及到 Linux、网络、虚拟化、Docker、Kubernetes、SDN、Python、Go、编程等，后台回复「1024」，送你一套 10T 资源学习大礼包，期待与你相遇。

93篇原创内容

公众号

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

阅读 1144

​

写留言

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/1TDxR6xkRSFD7ibPKdP9JOqxFxp199B6LLCw80xbTAicxIQPgpsq5ib8kvx7eKfq6erLb1CAT4ZKa3RSnl9IWfELw/300?wx_fmt=png&wxfrom=18)

Linux云计算网络

41074

写留言

写留言

**留言**

暂无留言
