Linux云计算网络
 _2021年12月04日 08:13_

## 1. 概述

在某些情况下，我们需要对于内核中的流程进行分析，虽然通过 BPF 的技术可以对于函数传入的参数和返回结果进行展示，但是在流程的调试上还是不如直接 GDB 单步调试来的直接。本文采用的编译方式如下，在一台 16 核 CentOS 7.7 的机器上进行内核源码相关的编译（主要是考虑编译效率），调试则是基于 VirtualBox 的 Ubuntu 20.04 系统中，采用 Qemu + GDB 进行单步调试，网上查看了很多文章，在最终进行单步跟踪的时候，始终不能够在断点处停止，进行过多次尝试和查询文档，最终发现需要在内核启动参数上添加 `nokaslr` ，本文是对整个搭建过程的总结。

![图片](https://mmbiz.qpic.cn/mmbiz_png/ciab8jTiab9J6CRYdmmSBQy127V98QYTF7MwkmrdeeEbhcr6hCkibTOqxpbqhSHEA3iamTpXYAau0E82tJxR4GCohw/640?wx_fmt=png&wxfrom=13&tp=wxpic)

## 2. Linux 内核编译和文件系统制作

### 2.1 Linux 内核编译

编译内核和制作文件系统在 CentOS 7.7 的机器上。源码从国内清华的源下载：http://ftp.sjtu.edu.cn/sites/ftp.kernel.org/pub/linux/kernel/， 此处选择 linux-4.19.172.tar.gz 版本。详细编译步骤如下：

```c
$ sudo yum group install "Development Tools"$ yum install ncurses-devel bison flex elfutils-libelf-devel openssl-devel $ wget http://ftp.sjtu.edu.cn/sites/ftp.kernel.org/pub/linux/kernel/v4.x/linux-4.19.172.tar.gz$ tar xzvf linux-4.19.172.tar.gz
$ cd linux-4.19.172/$ make menuconfig
```

在内核编译选项中，开启如下 “Compile the kernel with debug info”， 4.19.172 中默认已经选中：  

Kernel hacking —> Compile-time checks and compiler options —> [ ] Compile the kernel with debug info
![[Pasted image 20240928132246.png]]
![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

以上配置完成后会在当前目录生成 `.config` 文件，我们可以使用 `grep` 进行验证：

```c
# grep CONFIG_DEBUG_INFO .configCONFIG_DEBUG_INFO=y
```

接着我们进行内核编译：

```c
 $ nproc       # 查看当前的系统核数 $ make -j 12  # 或者采用 make bzImage 进行编译， -j N，表示使用多少核并行编译  # 未压缩的内核文件，这个在 gdb 的时候需要加载，用于读取 symbol 符号信息，由于包含调试信息所以比较大 $ ls -hl vmlinux -rwxr-xr-x 1 root root 449M Feb  3 14:46 vmlinux# 压缩后的镜像文件 $ ls -hl ./arch/x86_64/boot/bzImagelrwxrwxrwx 1 root root 22 Feb  3 14:47 ./arch/x86_64/boot/bzImage -> ../../x86/boot/bzImage$ ls -hl ./arch/x86/boot/bzImage-rw-r--r-- 1 root root 7.6M Feb  3 14:47 ./arch/x86/boot/bzImage
```

不同发行版本下的内核的详细编译文档可以参考这里。  

### 2.2 启动内存文件系统制作

```c
# 首先安装静态依赖，否则会有报错，参见后续的排错章节$ yum install -y glibc-static.x86_64 -y
$ wget https://busybox.net/downloads/busybox-1.32.1.tar.bz2
$ tar -xvf busybox-1.32.1.tar.bz2$ cd busybox-1.32.1/$ make menuconfig# 安装完成后生成的相关文件会在 _install 目录下$ make && make install   $ cd _install$ mkdir proc$ mkdir sys$ touch init  #  init 内容见后续章节，为内核启动的初始化程序$ vim init   # 必须设置成可执行文件$ chmod +x init  $ find . | cpio -o --format=newc > ./rootfs.imgcpio: File ./rootfs.img grew, 2758144 new bytes not copied10777 blocks$ ls -hl rootfs.img-rw-r--r-- 1 root root 5.3M Feb  2 11:23 rootfs.img
```

其中上述的 `init` 文件内容如下，打印启动日志和系统的整个启动过程花费的时间：

```c
#!/bin/sh
echo "{==DBG==} INIT SCRIPT"mkdir /tmpmount -t proc none /procmount -t sysfs none /sysmount -t debugfs none /sys/kernel/debugmount -t tmpfs none /tmpmdev -s echo -e "{==DBG==} Boot took $(cut -d' ' -f1 /proc/uptime) seconds"# normal usersetsid /bin/cttyhack setuidgid 1000 /bin/sh
```

到此为止我们已经编译了好了 Linux 内核（vmlinux 和 bzImage）和启动的内存文件系统（rootfs.img）。  

### 2.3 错误排查

在编译过程中出现以下报错：

```c
/bin/ld: cannot find -lcrypt/bin/ld: cannot find -lm/bin/ld: cannot find -lresolv/bin/ld: cannot find -lrtcollect2: error: ld returned 1 exit statusNote: if build needs additional libraries, put them in CONFIG_EXTRA_LDLIBS.Example: CONFIG_EXTRA_LDLIBS="pthread dl tirpc audit pam"
```

出错的原因是因为我们采用静态编译依赖的底层库没有安装，如果不清楚这些库有哪些 rpm 安装包提供，则可以通过 `yum provides` 命令查看，然后安装相关依赖包重新编译即可。

```c
$ yum provides */libm.a// ...glibc-static-2.17-317.el7.x86_64 : C library static libraries for -static linking.Repo        : baseMatched from:Filename    : /usr/lib64/libm.a
```

## 3. Qemu 启动内核  

在上述步骤准备好以后，我们需要在调试的 Ubuntu 20.04 的系统中安装 Qemu 工具，其中调测的 Ubuntu 系统使用 VirtualBox 安装。

```c
$ apt install qemu qemu-utils qemu-kvm virt-manager libvirt-daemon-system libvirt-clients bridge-utils
```

把上述编译好的 vmlinux、bzImage、rootfs.img 和编译的源码拷贝到我们当前 Unbuntu 机器中。  

拷贝 Linux 编译的源码主要是在 gdb 的调试过程中查看源码，其中 vmlinux 和 linux 源码处于相同的目录，本例中 vmlinux 位于 linux-4.19.172 源目录中。

```c
$ qemu-system-x86_64 -kernel ./bzImage -initrd  ./rootfs.img -append "nokaslr console=ttyS0" -s -S -nographic
```

使用上述命令启动调试，启动后会停止在界面处，并等待远程 gdb 进行调试，在使用 GDB 调试之前，可以先使用以下命令进程测试内核启动是否正常。

```c
qemu-system-x86_64 -kernel ./bzImage -initrd  ./rootfs.img -append "nokaslr console=ttyS0" -nographic
```

其中命令行中各参数如下：  

- `-kernel ./bzImage`：指定启用的内核镜像；
- `-initrd ./rootfs.img`：指定启动的内存文件系统；
- `-append "nokaslr console=ttyS0"` ：附加参数，其中 `nokaslr` 参数必须添加进来，防止内核起始地址随机化，这样会导致 gdb 断点不能命中；参数说明可以参见这里。
- `-s` ：监听在 gdb 1234 端口；
- `-S` ：表示启动后就挂起，等待 gdb 连接；
- `-nographic`：不启动图形界面，调试信息输出到终端与参数 `console=ttyS0` 组合使用；

![[Pasted image 20240928132618.png]]
## 4. GDB 调试

在使用 `qemu-system-x86_64` 命令启动内核以后，进入到我们从编译机器上拷贝过来的 Linux 内核源代码目录中，在另外一个终端我们来启动 gdb 命令：

```c
[linux-4.19.172]$ gdb (gdb) file vmlinux           # vmlinux 位于目录 linux-4.19.172 中(gdb) target remote :1234(gdb) break start_kernel     # 有些文档建议使用 hb 硬件断点，我在本地测试使用 break 也是 ok 的(gdb) c                      # 启动调试，则内核会停止在 start_kernel 函数处
```

整体运行界面如下：  
![[Pasted image 20240928132635.png]]

## 5. Eclipse 图像化调试  

我们可以通过 eclipse-cdt 进行可视化项目调试。

”File“ -> “New” -> “Project” ，然后选择 ”Makefile Project with Existing Code“ 选项，后续按照向导导入代码。
![[Pasted image 20240928132646.png]]
![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

在 “Run” -> “Debug Configurations” 选项中，创建一个 ”C/C++ Attach to Application“ 的调试选项。

- Project：选择我们刚才创建的项目名字；
- C/C++ Application：选择编译 Linux 内核带符号信息表的 vmlinux；
- Build before launching：选择 ”Disable auto build“；
- Debugger：选择 gdbserver，具体设置如下图；
- 在 Debugger 中的 Connection 信息中选择 ”TCP“，并填写端口为 ”1234“；

启动 Debug 调试，即可看到与 gdb 类似的窗口。
![[Pasted image 20240928132655.png]]

启动 ”Debug“ 调试以后的窗口如下，在 Debug 窗口栏中，设置与 gdb 调试相同的步骤即可。
![[Pasted image 20240928132702.png]]

## 6. 参考

- How to compile and install Linux Kernel 5.6.9 from source code
    
- 用qemu + gdb调试linux内核 ***
    
- QEMU+busybox 搭建Linux内核运行环境 ***
    
- QEMU+gdb调试Linux内核全过程 *
    
- linux内核编译与调试方法
    
- How to Build A Custom Linux Kernel For Qemu (2015 Edition)
    
- qemu与qemu-kvm到底什么区别
    
- 在qemu环境中用gdb调试Linux内核 *
    

原文：https://www.ebpf.top/post/qemu_gdb_busybox_debug_kernel/

  

---

后台回复“加群”，带你进入高手如云交流群

  

**推荐阅读：**

[深入理解 Cache 工作原理](http://mp.weixin.qq.com/s?__biz=MzI1OTY2MzMxOQ==&mid=2247497507&idx=1&sn=91327167900834132efcbc6129d26cd0&chksm=ea77c39bdd004a8de63a542442b5113d5cde931a2d76cff5c92b7e9f68cd561d3ed6ef7a1aa5&scene=21#wechat_redirect)

[Cilium 容器网络的落地实践](http://mp.weixin.qq.com/s?__biz=MzI1OTY2MzMxOQ==&mid=2247497237&idx=1&sn=d84b91d9e416bb8d18eee409b6993743&chksm=ea77c2addd004bbb0eda5815bbf216cff6a5054f74a25122c6e51fafd2512100e78848aad65e&scene=21#wechat_redirect)

[【中断】的本质](http://mp.weixin.qq.com/s?__biz=MzI1OTY2MzMxOQ==&mid=2247496751&idx=1&sn=dbdb208d4a9489981364fa36e916efc9&chksm=ea77c097dd004981e7358d25342f5c16e48936a2275202866334d872090692763110870136ad&scene=21#wechat_redirect)  

[图解 | Linux内存回收之LRU算法](http://mp.weixin.qq.com/s?__biz=MzI1OTY2MzMxOQ==&mid=2247496417&idx=1&sn=4267d317bb0aa5d871911f255a8bf4ad&chksm=ea77c659dd004f4f54a673830560f31851dfc819a2a62f248c7e391973bd14ab653eaf2a63b8&scene=21#wechat_redirect)  

[Linux 应用内存调试神器- ASan](http://mp.weixin.qq.com/s?__biz=MzI1OTY2MzMxOQ==&mid=2247496414&idx=1&sn=897d3d39e208652dcb969b5aca221ca1&chksm=ea77c666dd004f70ebee7b9b9d6e6ebd351aa60e3084149bfefa59bca570320ebcc7cadc6358&scene=21#wechat_redirect)

[深入理解 Cilium 的 eBPF 收发包路径](http://mp.weixin.qq.com/s?__biz=MzI1OTY2MzMxOQ==&mid=2247496345&idx=1&sn=22815aeadccc1c4a3f48a89e5426b3f3&chksm=ea77c621dd004f37ff3a9e93a64e145f55e621c02a917ba0901e8688757cc8030b4afce2ef63&scene=21#wechat_redirect)

[Page Cache和Buffer Cache关系](http://mp.weixin.qq.com/s?__biz=MzI1OTY2MzMxOQ==&mid=2247495951&idx=1&sn=8bc76e05a63b8c9c9f05c3ebe3f99b7a&chksm=ea77c5b7dd004ca18c71a163588ccacd33231a58157957abc17f1eca17e5dcb35147b273bc52&scene=21#wechat_redirect)

[深入理解DPDK程序设计|Linux网络2.0](http://mp.weixin.qq.com/s?__biz=MzI1OTY2MzMxOQ==&mid=2247495791&idx=1&sn=5d9f3bdc29e8ae72043ee63bc16ed280&chksm=ea77c4d7dd004dc1eb0cee7cba6020d33282ead83a5c7f76a82cb483e5243cd082051e355d8a&scene=21#wechat_redirect)  

[一文读懂基于Kubernetes打造的边缘计算](http://mp.weixin.qq.com/s?__biz=MzI1OTY2MzMxOQ==&mid=2247495291&idx=1&sn=0aebc6ee54af03829e15ac659db923ae&chksm=ea77dac3dd0053d5cd4216e0dc91285ff37607c792d180b946bc09783d1a2032b0dffbcb03f0&scene=21#wechat_redirect)

[网络方案 Cilium 入门教程](http://mp.weixin.qq.com/s?__biz=MzI1OTY2MzMxOQ==&mid=2247495170&idx=1&sn=54d6c659853f296fd6e6e20d44b06d9b&chksm=ea77dabadd0053ac7f72c4e742942f1f59d29000e22f9e31d7146bcf1d7d318b68a0ae0ef91e&scene=21#wechat_redirect)

[Docker  容器技术使用指南](http://mp.weixin.qq.com/s?__biz=MzI1OTY2MzMxOQ==&mid=2247494756&idx=1&sn=f7384fc8979e696d587596911dc1f06b&chksm=ea77d8dcdd0051ca7dacde28306c535508b8d97f2b21ee9a8a84e2a114325e4274e32eccc924&scene=21#wechat_redirect)

[云原生/云计算发展白皮书（附下载）](http://mp.weixin.qq.com/s?__biz=MzI1OTY2MzMxOQ==&mid=2247494647&idx=1&sn=136f21a903b0771c1548802f4737e5f8&chksm=ea77df4fdd00565996a468dac0afa936589a4cef07b71146b7d9ae563d11882859cc4c24c347&scene=21#wechat_redirect)

[使用 GDB+Qemu 调试 Linux 内核](http://mp.weixin.qq.com/s?__biz=MzI1OTY2MzMxOQ==&mid=2247493336&idx=1&sn=268fae00f4f88fe27b24796644186e9e&chksm=ea77d260dd005b76c10f75dafc38428b8357150f3fb63bc49a080fb39130d6590ddea61a98b5&scene=21#wechat_redirect)

[防火墙双机热备](http://mp.weixin.qq.com/s?__biz=MzI1OTY2MzMxOQ==&mid=2247493173&idx=1&sn=53975601d927d4a89fe90d741121605b&chksm=ea77d28ddd005b9bdd83dac0f86beab658da494c4078af37d56262933c866fcb0b752afcc4b9&scene=21#wechat_redirect)

[常见的几种网络故障案例分析与解决](http://mp.weixin.qq.com/s?__biz=MzI1OTY2MzMxOQ==&mid=2247493157&idx=1&sn=de0c263f74cb3617629e84062b6e9f45&chksm=ea77d29ddd005b8b6d2264399360cfbbec8739d8f60d3fe6980bc9f79c88cc4656072729ec19&scene=21#wechat_redirect)

[Kubernetes容器之间的通信浅谈](http://mp.weixin.qq.com/s?__biz=MzI1OTY2MzMxOQ==&mid=2247493145&idx=1&sn=c69bd59a40281c2d7e669a639e1a50cd&chksm=ea77d2a1dd005bb78bf499ea58d3b6b138647fc995c71dcfc5acaee00cd80209f43db878fdcd&scene=21#wechat_redirect)

[kube-proxy 如何与 iptables 配合使用](http://mp.weixin.qq.com/s?__biz=MzI1OTY2MzMxOQ==&mid=2247492982&idx=1&sn=2b842536b8cdff23e44e86117e3d940f&chksm=ea77d1cedd0058d82f31248808a4830cbe01077c018a952e3a9034c96badf9140387b6f011d6&scene=21#wechat_redirect)

[完美排查入侵](http://mp.weixin.qq.com/s?__biz=MzI1OTY2MzMxOQ==&mid=2247492931&idx=1&sn=523a985a5200430a7d4c71333efeb1d4&chksm=ea77d1fbdd0058ed83726455c2f16c9a9284530da4ea612a45d1ca1af96cb4e421290171030a&scene=21#wechat_redirect)

[QUIC也不是万能的](http://mp.weixin.qq.com/s?__biz=MzI1OTY2MzMxOQ==&mid=2247491959&idx=1&sn=61058136134e7da6a1ad1b9067eebb95&chksm=ea77d5cfdd005cd9261e3dc0f9689291895f0c9764aa1aa740608c0916405a5b94302f659025&scene=21#wechat_redirect)

[为什么要选择智能网卡？](http://mp.weixin.qq.com/s?__biz=MzI1OTY2MzMxOQ==&mid=2247491828&idx=1&sn=d81f41f6e09fac78ddddc287feabe502&chksm=ea77d44cdd005d5a48dc97e13f644ea24d6e9533625ce8f204d4986b6ba07f328c5574820703&scene=21#wechat_redirect)

[网络排错大讲解~](http://mp.weixin.qq.com/s?__biz=MzI1OTY2MzMxOQ==&mid=2247488574&idx=1&sn=68df1982e9f23ce42457d00ce529b012&chksm=ea742086dd03a9902adb16d2b7648149209fed6c9811c6dd05be5101b42d462cb48e269b6e9d&scene=21#wechat_redirect)

[OVS 和 OVS-DPDK 对比](http://mp.weixin.qq.com/s?__biz=MzI1OTY2MzMxOQ==&mid=2247488294&idx=2&sn=303d9baefa768f887ba36213df793025&chksm=ea74279edd03ae88bc91300a1666066881dd763879b714941120fb9e41e787c557c2930ff5db&scene=21#wechat_redirect)

[微软出品的最新K8S学习指南3.0](http://mp.weixin.qq.com/s?__biz=MzI1OTY2MzMxOQ==&mid=2247488271&idx=1&sn=a52af9aba2ea8bbc5fd4440f85ece458&chksm=ea7427b7dd03aea1dbf106326168333c651b403dd44abf4e652d5a1912377d64becabac5051a&scene=21#wechat_redirect)下载

▼

  

_**_****喜欢，就给我一个****“在看”****_**_

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**10T 技术资源大放送！包括但不限于：云计算、虚拟化、微服务、大数据、网络、**Linux、**Docker、Kubernetes、Python、Go、C/C++、Shell、PPT 等。在公众号内回复「****1024****」**，即可免费获！！****

阅读 1318

​

写留言

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/1TDxR6xkRSFD7ibPKdP9JOqxFxp199B6LLCw80xbTAicxIQPgpsq5ib8kvx7eKfq6erLb1CAT4ZKa3RSnl9IWfELw/300?wx_fmt=png&wxfrom=18)

Linux云计算网络

5分享4

写留言

写留言

**留言**

暂无留言