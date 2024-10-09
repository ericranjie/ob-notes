ScUpax0s 看雪学苑
_2022年02月11日 17:58_\
本文为看雪论坛精华文章
看雪论坛作者ID：ScUpax0s

# **环境搭建**

最终要产生这样一个环境：

![图片](https://mmbiz.qpic.cn/mmbiz_png/kR3MmOkZ9r7ouW9ItrawF5VVR7IWmjmRoJXicpibru3qUaU2lJQOyRbQN1qQLRibzic5Xaek06icicwk04gA3d3PgBFw/640?wx_fmt=png&wxfrom=13&tp=wxpic)

目前看到网上的貌似都是双机调试，但是双机还是比较麻烦，于是最好就是用QEMU来做。

1、首先编译对应的内核，得到vmlinux，bzImage。

注意，在编译内核时一定要打开：CONFIG_OVERLAY_FS=y 因为docker需要对应的文件系统的支持。

打开 CONFIG_GDB_SCRIPTS=y , CONFIG_DEBUG_INFO=y 如果后续要调试内核的话。

2、然后利用syzkaller的 create-image.sh。

修改添加对cgroup的挂载：

```c
echo 'debugfs /sys/kernel/debug debugfs defaults 0 0' | sudo tee -a $DIR/etc/fstab echo 'securityfs /sys/kernel/security securityfs defaults 0 0' | sudo tee -a $DIR/etc/fstab echo 'configfs /sys/kernel/config/ configfs defaults 0 0' | sudo tee -a $DIR/etc/fstab echo 'binfmt_misc /proc/sys/fs/binfmt_misc binfmt_misc defaults 0 0' | sudo tee -a $DIR/etc/fstab echo 'tmpfs /sys/fs/cgroup cgroup defaults 0 0' | sudo tee -a $DIR/etc/fstab
```

执行：./create-image.sh 生成文件系统。

3、qemu启动脚本：

```c
qemu-system-x86_64 \ -drive file=./stretch.img,format=raw \ -m 256 \ -net nic \ -net user,host=10.0.2.10,hostfwd=tcp::23505-:22 \ -enable-kvm \ -kernel ./bzImage \ -append "console=ttyS0 root=/dev/sda earlyprintk=serial" \ -nographic \ -pidfile vm.pid \
```

4、启动起来之后的连接命令：

```c
ssh-keygen -f "/root/.ssh/known_hosts" -R "[localhost]:23505"
ssh -i ./stretch.id_rsa -p 23505 root@localhost
```

5、最终可以通过ssh进入系统。

```c
root@syzkaller:~#
```

6、修改登陆密码：

```c
apt install docker-ce   
vim /etc/ssh/sshd_config
PermitRootLogin yes
passwd root
poweroff
```

如果第一步失败就按照：

_https://docs.docker.com/engine/install/debian/_
https://stackoverflow.com/questions/48002345/docker-ce-depends-libseccomp2-2-3-0-but-2-2-3-3ubuntu3-is-to-be-installe\_

7、重新登陆确认：

```bash
Debian GNU/Linux 9 syzkaller ttyS0   syzkaller login: root Password: Unable to get valid context for root Last login: Fri Dec  3 14:09:17 UTC 2021 from 10.0.2.10 on pts/0 Linux syzkaller 5.16.0-rc1 #3 SMP PREEMPT Wed Dec 1 09:46:37 PST 2021 x86_64   The programs included with the Debian GNU/Linux system are free software; the exact distribution terms for each program are described in the individual files in /usr/share/doc/*/copyright.   
Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent permitted by applicable law. root@syzkaller:~#
```

8、安装对应的docker-ce，方法：

_https://docs.docker.com/engine/install/debian/_

9、这里有一个官方提供的用于检查对应的docker运行时环境的。

_https://github.com/moby/moby/blob/master/contrib/check-config.sh_

使用方式：

```c
./check-config.sh /path/to/kernel/.config
```

如果启动不起来什么的，就运行一下这个脚本check一下环境

然后开启对应的需要的CONFIG，重新编译即可。

**如果以上步骤之后，还是无法直接运行dockerd，那么请按照下面的步骤由runc来直接手动创建容器。**

```bash
# 1. Get rootfs(out of VM) 
docker export $(docker create busybox) -o busybox.tar   # 2. put rootfs into VM's .img root@ubuntu:~/container# mount -o loop ./stretch.img /mnt/chroot/ root@ubuntu:~/container# cp ./busybox.tar /mnt/chroot/root/ root@ubuntu:~/container# umount /mnt/chroot/   # 3. Finally, boot the QEMU, and untar the busybox.tar in ~ to rootfs/ root@syzkaller:~# cd rootfs/ root@syzkaller:~/rootfs# pwd /root/rootfs root@syzkaller:~/rootfs# ls bin  dev  etc  home  proc  root  sys  tmp  usr  var   # 4. Generate OCI config docker-runc spec root@syzkaller:~# ls config.json  rootfs   # 5. Run manually, docker-runc run <ContainerName> root@syzkaller:~# docker-runc run guoziyi / # ls bin   dev   etc   home  proc  root  sys   tmp   usr   var / # id uid=0(root) gid=0(root) / # ps -ef PID   USER     TIME  COMMAND     1 root      0:00 sh     7 root      0:00 ps -ef / # exit   # 6. vim config.json "root": {     "path":"root",     "readonly":"false" }
```

10、最终效果：

首先我们在runc启动的容器中起一个top进程：

```
Mem: 261032K used, 1438232K free, 16624K shrd, 6996K buff, 101932K cached
```

可以看到，此时容器内部有两个进程，一个是1号进程sh，一个是6号进程top。

然后我们在容器外运行一下 pstree。

```
root@syzkaller:~# pstree -pl
```

可以看到，容器内的1号进程实际上就是容器外的507号进程映射进来的。他是docker-runc的子进程。同样的，容器内的top进程也是507的子进程。

# **从内核的角度看一个容器内进程**

在本部分我们通过从host机上使用gdb(gef)来target remote到qemu的内核，然后去观察对应的container的process。

这里推荐使用gef插件，感觉比pwndbg快得多。

我个人对于内核的配置如下（1 year expire）：

_https://paste.ubuntu.com/p/wMvftKv2bV/_

首先需要明确的一点是，容器内进程的本质接近于一个宿主机上命名空间、资源、文件系统隔离的受限进程。

我们关注task_struct中的一些结构：

```
/* task_struct member predeclarations (sorted alphabetically): */
```

可以看到，作为一个进程，在内核的PCB中维护了对应的结构，其中两个非常重要的一个是涉及工作目录or路径的 fs，一个是涉及namespace的nsproxy，最后是涉及资源限制的css_set。

我们可以观察下这些个结构体。

**struct nsproxy**

```
/*
```

**struct fs_struct**

```
struct fs_struct {
```

```
struct fs_struct
```

可以看到容器内的一号进程的工作目录：

```
gef➤  p ((struct task_struct *)0xffff88800c1b8000)->fs->root->dentry->d_name
```

**fs：**

```
# container init
```

**namespace：**

```
gef➤  p *(struct nsproxy*)$t->nsproxy
```

可以看到，从命名空间的角度上来说，容器内一号进程的ns和虚拟机自身的一号进程的ns是不一样的。uts、ipc、mnt、pid等都是新的。

**cred：**

容器进程

```
gef➤  p *$t->cred
```

host进程：

```
gef➤  p *$init->cred
```

可以看到虽然id都是0，但是他们的Cap是不一样的，host的init进程有完全的CAP，但是容器的init进程只有很少的Capability。

```
root@ubuntu:~/container/module_for_container# capsh --decode=0x20000420
```

## 

## **从内核漏洞到容器逃逸**

从内核打到容器逃逸其实原理也很简单，就是把当前docker里的sh进程的nsproxy和fs都切换成宿主机进程的（最好是宿主机init进程）。想达到这个效果要满足以下的条件：

1、能通过遍历拿到宿主机某一个进程（最好是init进程）的task_struct。

2、能**读写**对应的task_struct的数据，修改当前进程的fs和nsproxy（比如直接把对应的指针改成指向init进程的，或者调用switch_task_namespaces 切换ns）

在实际测试中，当我们把进程的task_struct的fs改成对应的 init_fs 。就已经能实现一个基本的逃逸了。

有了这个基础后，其实很容易想到，其实我们可以让容器进程逃逸到任意别的进程的ns和fs中，只要把对应的信息切换过去就行。

更进一步的，我们可以尝试寻找如何对于 init_fs 进行堆喷。

## 

## **从内核特性到容器逃逸**

这一部分与上一部分区别很大的是，我们并不是直接通过内核漏洞来完成容器逃逸的整个过程，而是通过一些Linux Feature来恶意的完成容器的逃逸。

由于eBPF本身是内核态的模块，可以用于几乎无差别的Hook，那么一个朴素的想法就是，通过eBPF去hook一些跑在用户态的并且可以执行命令的服务（用户态进程），然后来进行一个容器外的命令执行。又或者通过eBPF来对直接的读内核态的敏感数据造成泄露，辅助逃逸或者信息的泄露。

#### 

#### **libbpf**

_https://github.com/libbpf/libbpf_

libbpf这个项目本身类似于

首先我们需要对应的btf支持：

> If your kernel doesn't come with BTF built-in, you'll need to build custom kernel. You'll need:
>
> pahole 1.16+ tool (part of dwarves package), which performs DWARF to BTF conversion;
>
> kernel built with CONFIG_DEBUG_INFO_BTF=y option;

check一下：

```
root@syzkaller:~# ls -la /sys/kernel/btf/vmlinux
```

#### 

#### **libbpf-bootstrap**

_https://github.com/libbpf/libbpf-bootstrap_

```
git clone https://github.com/libbpf/libbpf-bootstrap.git
```

接下来创建对应的hello文件：

```
/* cat hello.bpf.c */
```

```
/* cat hello.c */
```

更新当前目录下对应的Makefile中的：

```
APPS = minimal bootstrap uprobe kprobe fentry hello
```

最后运行 make，然后运行 ./hello

输出：

```
root@ubuntu:~/libbpf-bootstrap/examples/c# ./hello
```

说明成功。

#### 

#### **eBPF用户态程序基本结构**

_https://facebookmicrosites.github.io/bpf/blog/2020/02/20/bcc-to-libbpf-howto-guide.html_

```
/* cat hello.bpf.c */
```

首先，bpf.h 中主要定义了一堆define和struct。

bpf_helpers.h 中主要是一些helper macros和functions。

```
/*
```

SEC是用来指定对应的类型，libbpf会根据上下文来解释然后放置到elf_bpf的不同的sections上。

在 hello.c 中主要是三个函数：

```
#define DEBUGFS "/sys/kernel/debug/tracing/"
```

这里通过 libbpf_set_print(libbpf_print_fn); 指定了bpf debug的输出标准。利用vfprintf从stderr输出。

接下来设置对应的rlimit by：

```
/* Bump RLIMIT_MEMLOCK to allow BPF sub-system to do anything */
```

```
/* set rlimit (required for every app) */
```

可以看到这里将值设置成最大。

最后有个 read_trace_pipe(); 读出log信息：

```
/* read trace logs from debug fs */
```

此外，还有一些别的函数。我们注意到在hello.c中有个：

```
#include "hello.skel.h"
```

这个文件应该是在编译过程中 Generate BPF skeletons 产生的。

```
/* Open BPF application */
```

```
root@ubuntu:~/libbpf-bootstrap/examples/c# cd .output/ && ls
```

在 .output 目录之下。

#### 

#### **通过evil eBPF劫持高权限进程完成逃逸**

11月份的时候，腾讯蓝军的同学发了一篇很有意思的文章：_https://security.tencent.com/index.php/blog/msg/206_

这篇文章的后续还是有很多可以研究的点。链接中的文章没有放出完整的代码，我这里完整的实现了这个完整的evil eBPF程序，代码在下面，大家可以参考一下。

我的实现代码：_https://github.com/OrangeGzY/Eebpf-kit/blob/main/libbpf-bootstrap/examples/c/hello.bpf.c_

##### 

##### **cron基本介绍与流程**

```
man cron
```

在ubuntu上我们使用的是 Vixie Cron

_https://www.runoob.com/w3cnote/linux-crontab-tasks.html_

```
root         800       1  0 02:17 ?        00:00:00 /usr/sbin/cron -f
```

_https://github.com/vixie/cron/tree/master_

在源码中其实我们主要关注 _https://github.com/vixie/cron/blob/master/database.c_ 的 load_database 函数。

```
#define CRONDIR        "/var/spool/cron"
```

可以看到是过了四个check然后直接调用：process_crontab 。

首先前两个判断，用stat获取对应的 SPOOL_DIR 和 SYSCRONTAB 文件的**最后一次修改时间**，放入对应的stat中。

```
struct stat
```

第三个判断old_db的mtim与 TMAX(statbuf.st_mtim, syscron_stat.st_mtim) 比是否发生了变化，其实就是是否更新。TMAX(statbuf.st_mtim, syscron_stat.st_mtim) 这里是取了两个文件最大（最新）的更新时间。最后通过new_db记录最新时间。

最后只要保证syscron的最新修改时间不是ts_zero，即可进入：process_crontab("root", NULL, SYSCRONTAB, &syscron_stat,&new_db, old_db);

```
const struct timespec ts_zero = {.tv_sec = 0L, .tv_nsec = 0L}
```

在 process_crontab 中：

```
// tabname = "/etc/crontab"
```

可以看到首先用fstat判断了一下，然后判断crontab是否更新，最终load_user。

可以看一下user对应的结构体：

```
typedef    struct _user {
```

可以看到里面维护了对应的user的crontab的cmd。

最终我们的job会被加入队列中，由 job_runqueue() 调用 do_command(j->e, j->u) 运行。

```
typedef    struct _job {
```

##### 

##### **Hook程序分析**

首先从 sys_enter 去Hook对应的系统调用。获取当前的syscall id，从进程 commandline 获取对应的文件名（比较是否为cron），然后根据我们捕捉到的不同系统调用再分配不同的处理函数。

相应的，我们也在对称的位置，即每个syscall退出时进行hook，主要涉及到了对于返回值的修改。

```
// When we enter syscall
```

```
// When we exit syscall
```

**handle_enter_stat(ctx)**

stat进入。

首先从rdi中读出文件名到缓冲区，然后确保文件名是 /etc/crontab 或者 crontabs。

接下来拿到当前的pid、文件名，存在全局变量中。

然后很关键的一步是通过rsi获取对应的 statbuf 结构（struct stat）的地址，也放入全局变量。

```
/*
```

这里主要是要Capture到对应的cron进程中对两个文件名判断的地方。

**handle_exit_stat()**

stat退出。

我们此时的目标是bypass后面的两个TEQUAL，让cron检测到文件的更新，然后立刻去调用 process_crontab("root", NULL, SYSCRONTAB, &syscron_stat,&new_db, old_db)。

```
static __inline int handle_exit_stat(){
```

**open**

open -> open64 -> openat

所以我们最终要对openat进行hook。

```
int openat(int dirfd, const char *pathname, int flags);
```

在enter时我们要保存+判断rsi中的参数。在退出时，保存open_fd。

```
// int openat(int  dirfd , const char * pathname
```

```
static __inline int handle_exit_openat(struct bpf_raw_tracepoint_args *ctx){
```

ok，现在我们已经有了对应的fd了。

**fstat**

```
int fstat(int fd, struct stat *statbuf);
```

```
// int fstat(int fd, struct stat *statbuf);
```

```
static __inline int handle_exit_fstat(){
```

**read**

```
// read(int fd, void *buf, size_t count);
```

```
static __inline int handle_exit_read(struct bpf_raw_tracepoint_args *ctx){
```

##### 

##### 在Docker中测试，首先构建相应的环境。

```
FROM ubuntu:20.04   
```

```
docker build -t .
```

在docker里直接运行对应文件即可。

可以从cron的log进行观测：

```
journalctl -f -u cron
```

最终在docker外执行了命令。

#### 

#### **通过eBPF劫持sshd进程**

在这篇文章之后，其实很容易想到，既然可以通过eBPF来对其他的进程的系统调用进行劫持，那有没有可能做一些其他的事情，比如尝试针对一些除了crond以外的其他的**用户态高权限进程**做一些事情，其实一个比较容易想到的就是sshd进程。

而事实上，通过eBPF确实可以实针对sshd的劫持。实现原理类似于上面的crontab hook，最终达到的效果包括但不仅限于：

1、patch掉原有用户的密码。

2、修改掉一个低权限用户为一个高权限登录的用户。

3、用一个不存在的用户直接登录。

然而作为一个pwn手/二进制选手，其实我不太清楚这个到底有什么作用，但是他确实是可以做到这样一个效果。。。我的实现代码可以在：_https://github.com/OrangeGzY/Eebpf-kit/blob/main/libbpf-bootstrap/examples/c/esshd.bpf.c_ 中找到。

#### 

#### **可能是潜在的地址泄露风险**

从另一个角度，如果我们能够hijack掉一些内核函数的调用，配合一些用户态的技巧，其实是很容易实现一个信息泄露的。比如泄露一些内核函数/全局结构的地址之类的。不过这个的使用条件类似于从crontab注入逃逸命令的条件，对权限要求比较苛刻。感觉只有在特定的容器环境内才会起作用。代码可以看：

_https://github.com/OrangeGzY/Eebpf-kit/blob/main/libbpf-bootstrap/examples/c/kprobe.bpf.c_

_https://github.com/OrangeGzY/Eebpf-kit/blob/main/libbpf-bootstrap/examples/c/spray.c_

效果：

```
root@ubuntu:~/Eebpf-kit/libbpf-bootstrap/examples/c# ./spray
```

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**看雪ID：ScUpax0s**

https://bbs.pediy.com/user-home-876323.htm

\*本文由看雪论坛 ScUpax0s 原创，转载请注明来自看雪社区

[!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458409596&idx=3&sn=658012a9debc76c73efed03342d41358&chksm=b18f6af686f8e3e09180522b243def309e265532cfadfdbf5becd7d9ec90f4505136959ebdfe&scene=21#wechat_redirect)

**#** **往期推荐**

1.[CVE-2019-10999复现学习](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458428260&idx=1&sn=fa3d9b2378eb0de3278c4b99496de336&chksm=b18f93ee86f81af81e6dba1d284e353edef9deb83f48b9e197d366da3b43025072b9b323d0fc&scene=21#wechat_redirect)

2.[内核漏洞学习-HEVD-UninitializedStackVariable](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458428150&idx=1&sn=69ab7abf5eb047bb7fd372dc0d45d3bd&chksm=b18f927c86f81b6a2641e45a1274153a01d13477c69742acb07ef9af6cb034b93c8e3aa2fe08&scene=21#wechat_redirect)

3.[记录一次vmp2.xdemo的分析](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458428142&idx=1&sn=28df738183d948cd2270f63c3f874dde&chksm=b18f926486f81b72d4cc7862f2304b3719a07a603055adddd9e482d78bed72cb2900b3c00eae&scene=21#wechat_redirect)

4.[内核漏洞学习-HEVD-NullPointerDereference](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458428141&idx=1&sn=80846d1b5260dae1b7dab121c07d0478&chksm=b18f926786f81b7101888bca96d5b69c2910f1c954a9e3a2b312a74766975072b6ece7f1ea7c&scene=21#wechat_redirect)

5.[Golang版本简易fuzzer及debugger实践](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458428140&idx=1&sn=2c8313469054f1bf41d0249f2ba1b177&chksm=b18f926686f81b70a4007461f693d06f3a59b062ea7b35fb8e4c75e7808f76f3dae79d94a29f&scene=21#wechat_redirect)

6.[Windows内核逆向——\<中断处理 从硬件机制到用户驱动接管>](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458428139&idx=1&sn=ac793292c14fe8126588dc7a612b7de3&chksm=b18f926186f81b77104d546de4b803829c3218d51c5d2c8116be9d4a3c3379091bc67ed941d0&scene=21#wechat_redirect)

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**球分享**

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**球点赞**

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**球在看**

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

点击“阅读原文”，了解更多！

阅读原文

阅读 2135

​

写留言

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/1UG7KPNHN8EGLfh77kFmnicd9WOic2ibvhCibFdB4bL4srJCgo2wnvdoXLxpIvAkfCmmcptXZB0qKWMoIP8iaibYN2FA/300?wx_fmt=png&wxfrom=18)

看雪学苑

824

写留言

写留言

**留言**

暂无留言
