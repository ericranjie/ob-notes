
Linux开发架构之路

 _2024年03月28日 20:03_ _湖南_

### 

1. 前言

目前的文件系统五花八门，从人人皆知的 ext4、ntfs 和 xfs，到 btrfs、zfs 等小众文件系统，可谓是琳琅满目，让人难以抉择适合自身业务的文件系统。

本文总结了国外网友在 OpenBenchmark 网站上的结果，并给出不同情境下各大文件系统的性能表现与对比。

目前所涉及到的文件系统包括：

- btrfs
    
- xfs
    
- ext4
    
- f2fs
    
- reiserFS
    
- ntfs
    
- zfs
    

### 

2. Linux 4.4 Benchmark

测试基准数据来源于 Linux 4.4 File-Systems。

### 

2.1 测试环境

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/8pECVbqIO0zKFlouIlW431XAASTa8VK4OIk8lCcOuKyOA4onQibQHd1Pr9lpfIur09P6iaA0FULlLwt0YhJzlciag/640?wx_fmt=png&from=appmsg&wxfrom=13)

  

各文件系统挂载选项：
![[Pasted image 20240911093408.png]]
![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

### 

2.2 测试手段

对于文件系统的 benchmark，分为以下的测试手段：

- aio-stress：使用 SuSE 开发的异步 I/O 框架，测试异步 I/O 在高压环境下的随机写入能力。
    
- sqlite：测量 SQLite 数据库进行固定数量的 Insert 操作的写入用时，此为数值越小越好的测试。
    
- flexible IO tester：无缓存情况直写磁盘的情况下，测试硬盘对 4KB 大小的块的随机读写及顺序读写能力。
    
- fs-mark：创建众多的 1MB 文件以测试文件系统写入能力。
    
- dbench：模拟多用户读写压力的 benchmark 工具。
    
- compilebench：通过内核树的编译与更新，模拟并测试硬盘快满且文件夹存放过久后的表现。
    

### 

2.3 测试结果

测试结果的总表如下，其中标 ^ 的为同测试中最优数据，标 * 为最差数据。数据单位均为 MB/s 或秒，且仅保留整数部分。：
![[Pasted image 20240911093417.png]]
![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

各个测试的图表如下：
![[Pasted image 20240911093425.png]]
![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  
![[Pasted image 20240911093433.png]]
![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)
![[Pasted image 20240911093506.png]]
![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  
![[Pasted image 20240911093515.png]]
![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  
![[Pasted image 20240911093521.png]]
![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  
![[Pasted image 20240911093528.png]]
![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)
![[Pasted image 20240911093535.png]]
![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  
![[Pasted image 20240911093541.png]]
![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  
![[Pasted image 20240911093547.png]]
![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  
![[Pasted image 20240911093554.png]]
![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)
![[Pasted image 20240911093600.png]]
![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  
![[Pasted image 20240911093605.png]]
![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

需要C/C++ Linux服务器架构师学习资料加qun579733396获取（资料包括C/C++，Linux，golang技术，Nginx，ZeroMQ，MySQL，Redis，fastdfs，MongoDB，ZK，流媒体，CDN，P2P，K8S，Docker，TCP/IP，协程，DPDK，ffmpeg等），免费分享

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

### 

2.4 各大文件系统对比

1. 高并发读写：btrfs 最优，xfs 及 ext4 稍好，zfs 及 ntfs 最差。
    
2. 随机读写：ntfs 最优，其余相差不大。
    
3. 顺序读写：ntfs 最优，btrfs 最差，其余相差不大
    
4. 顺序多文件写入：f2fs 最优，其余相差不大。
    
5. 并发多文件写入：ext4 最优，ntfs 最差，zfs 较弱，其余相差不大。
    
6. 顺序多文件写入不同子目录：f2fs 最优，其余相差不大。
    
7. 多客户使用体验：xfs 最优，ntfs 最差，reiserFS 较弱，其余皆优秀。
    

通过此 benchmark，可以发现各个文件系统具有特性，且各有优缺点：

- btrfs 仍处于开发阶段，其 COW (Copy On Write) 机制使其对数据库的插入操作表现较差。
    
- xfs 和 ext4 的综合素质优秀，特定能力上不会过于耀眼，但是对用户来说综合体验最佳。
    
- f2fs 在多文件情境下表现出色。
    
- reiserFS 即将被 reiser4 文件系统取代，综合性能大不及其它系统。
    
- ntfs 在串行的工作模式下，其随机读写和顺序读写能力都极高；反之，其在高并发高压环境下表现不佳。
    

> 注：fio 测试中缺少 zfs 相关数据，将在之后的测试中重新进行分析。

### 

3. ZFS Ubuntu 16.04 测试

由于 zfs 相关数据在第 2 章的测试中不够完善，因此本章补充 zfs 相关的测试内容。

本章数据参考自 ZFS Ubuntu 16.04 Testing。

### 

3.1 测试环境
![[Pasted image 20240911093621.png]]
![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

各文件系统挂载选项：
![[Pasted image 20240911093626.png]]
![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

> 注：此环境与测试 2 中基本一致，可以联合查看。

### 

3.2 测试手段

对于文件系统的 benchmark，分为以下的测试手段：

- fs-mark：创建众多的 1MB 文件以测试文件系统写入能力。
    
- blogbench：模拟一个博客网站的压力场景，测试随机读写的能力。
    
- dbench：模拟多用户读写压力的 benchmark 工具。
    
- iozone：发起众多不同类型的 I/O 请求，测试文件系统的性能。
    
- compilebench：通过内核树的编译与更新，模拟并测试硬盘快满且文件夹存放过久后的表现。、
    

### 

3.3 测试结果

与 2.3 中测试样式基本一致，所使用单位为 MB/s 或其他特殊的单位：
![[Pasted image 20240926190230.png]]
![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

### 

3.4 zfs 性能分析

相较于其他常用的文件系统，zfs 在性能上并不能占得太多优势，相反地，zfs 在各方面的性能上都不及其它的文件系统。

但这并不意味着 zfs 一无是处，zfs 的使用场景主要位于多硬盘体系，其最具特色的功能在于其提供文件系统级别的 raid 阵列能力，因此仍然有不少的适用场景。

### 

4. 关于性能测试

性能测试并不代表文件系统的实际体验，在读者阅读此篇文章之前，需要明确此点。

许多文件系统并不一定专门为性能而生，它们都有自己的适用场景，每个文件系统都有其独特的设计初衷。例如：

- btrfs 和 zfs 都具有 COW(Copy On Write) 机制。COW 机制使得它们在进行修改操作时具有不俗的性能，但也带来了磁盘空间的浪费，并增加了读取时的消耗。
    
- btrfs 和 zfs 都自带 raid 功能。在数据可用性有要求的情况下，相比其他的文件系统，它们能提供开箱即用的 raid 阵列能力。
    
- btrfs 对所有数据都带有 checksum 机制，确保文件完整性。
    
- xfs 具有动态 inode 分配能力，适合大文件分配。
    
- … …
    

因地制宜，适才取用，才是程序员之道。

  

阅读 1086

​