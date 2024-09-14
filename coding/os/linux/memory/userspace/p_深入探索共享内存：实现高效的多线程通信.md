

原创 往事敬秋风 深度Linux

 _2024年03月15日 21:50_ _湖南_

共享内存是一种在多个进程或线程之间共享数据的机制。它允许不同的进程或线程可以直接读取和写入同一块内存区域，从而实现高效的数据共享和通信，通过使用共享内存，可以避免复制大量数据，减少了通信开销，并提高了程序的性能。常见的应用场景包括并行计算、多线程编程、分布式系统等。

![](http://mmbiz.qpic.cn/mmbiz_png/dkX7hzLPUR0Ao40RncDiakbKx1Dy4uJicoqwn5GZ5r7zSMmpwHdJt32o95wdQmPZrBW038j8oRSSQllpnOUDlmUg/300?wx_fmt=png&wxfrom=19)

**深度Linux**

拥有15年项目开发经验及丰富教学经验，曾就职国内知名企业项目经理，部门负责人等职务。研究领域：Windows&Linux平台C/C++后端开发、Linux系统内核等技术。

181篇原创内容

公众号

在使用共享内存时，需要注意对于并发访问的控制，如使用锁或其他同步机制来保证数据的一致性和安全性。此外，还需要谨慎处理资源管理问题，确保正确地释放共享内存以避免内存泄漏。

## 一、共享内存原理

共享内存是System V版本的最后一个进程间通信方式。共享内存，顾名思义就是允许两个不相关的进程访问同一个逻辑内存，共享内存是两个正在运行的进程之间共享和传递数据的一种非常有效的方式。不同进程之间共享的内存通常为同一段物理内存。进程可以将同一段物理内存连接到他们自己的地址空间中，所有的进程都可以访问共享内存中的地址。如果某个进程向共享内存写入数据，所做的改动将立即影响到可以访问同一段共享内存的任何其他进程。

特别提醒：共享内存并未提供同步机制，也就是说，在第一个进程结束对共享内存的写操作之前，并无自动机制可以阻止第二个进程开始对它进行读取，所以我们通常需要用其他的机制来同步对共享内存的访问，例如信号量。

### 1.1通信原理

在Linux中，每个进程都有属于自己的进程控制块（PCB）和地址空间（Addr Space），并且都有一个与之对应的页表，负责将进程的虚拟地址与物理地址进行映射，通过内存管理单元（MMU）进行管理。两个不同的虚拟地址通过页表映射到物理空间的同一区域，它们所指向的这块区域即共享内存。

共享内存的通信原理示意图：
![[Pasted image 20240914191904.png]]
![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

对于上图我的理解是：当两个进程通过页表将虚拟地址映射到物理地址时，在物理地址中有一块共同的内存区，即共享内存，这块内存可以被两个进程同时看到。这样当一个进程进行写操作，另一个进程读操作就可以实现进程间通信。但是，我们要确保一个进程在写的时候不能被读，因此我们使用信号量来实现同步与互斥。

对于一个共享内存，实现采用的是引用计数的原理，当进程脱离共享存储区后，计数器减一，挂架成功时，计数器加一，只有当计数器变为零时，才能被删除。当进程终止时，它所附加的共享存储区都会自动脱离。

为什么共享内存速度最快？

借助上图说明：Proc A 进程给内存中写数据， Proc B 进程从内存中读取数据，在此期间一共发生了两次复制

```c
（1）Proc A 到共享内存       （2）共享内存到 Proc B
```

因为直接在内存上操作，所以共享内存的速度也就提高了。

### 1.2接口函数以及指令

1.查看系统中的共享存储段

```c
ipcs -m
```

2.删除系统中的共享存储段

```c
ipcrm -m [shmid]
```

3.shmget ( )：创建共享内存

```c
int shmget(key_t key, size_t size, int shmflg);
```

[参数key]：由ftok生成的key标识，标识系统的唯一IPC资源。

[参数size]：需要申请共享内存的大小。在操作系统中，申请内存的最小单位为页，一页是4k字节，为了避免内存碎片，我们一般申请的内存大小为页的整数倍。

[参数shmflg]：如果要创建新的共享内存，需要使用IPC_CREAT，IPC_EXCL，如果是已经存在的，可以使用IPC_CREAT或直接传0。

[返回值]：成功时返回一个新建或已经存在的的共享内存标识符，取决于shmflg的参数。失败返回-1并设置错误码。

4.shmat ( )：挂接共享内存

```c
void *shmat(int shmid, const void *shmaddr, int shmflg);
```

[参数shmid]：共享存储段的标识符。

[参数*shmaddr]：shmaddr = 0，则存储段连接到由内核选择的第一个可以地址上（推荐使用）。

[参数shmflg]：若指定了SHM_RDONLY位，则以只读方式连接此段，否则以读写方式连接此段。

[返回值]：成功返回共享存储段的指针（虚拟地址），并且内核将使其与该共享存储段相关的shmid_ds结构中的shm_nattch计数器加1（类似于引用计数）；出错返回-1。

5.shmdt ( )：去关联共享内存

当一个进程不需要共享内存的时候，就需要去关联。该函数并不删除所指定的共享内存区，而是将之前用shmat函数连接好的共享内存区脱离目前的进程。

```c
int shmdt(const void *shmaddr);
```

[参数*shmaddr]：连接以后返回的地址。

[返回值]：成功返回0，并将shmid_ds结构体中的 shm_nattch计数器减1；出错返回-1。

6.shmctl ( )：销毁共享内存

```c
int shmctl(int shmid, int cmd, struct shmid_ds *buf);
```

[参数shmid]：共享存储段标识符。

[参数cmd]：指定的执行操作，设置为IPC_RMID时表示可以删除共享内存。

[参数*buf]：设置为NULL即可。

[返回值]：成功返回0，失败返回-1。

### 1.3模拟共享内存

我们用server来创建共享存储段，用client获取共享存储段的标识符，二者关联起来之后server将数据写入共享存储段，client从共享区读取数据。通信结束之后server与client断开与共享区的关联，并由server释放共享存储段。

comm.h

```c
//comm.h
#ifndef _COMM_H__
#define _COMM_H__ #include<stdio.h>
#include<sys/types.h>
#include<sys/ipc.h>
#include<sys/shm.h> 
#define PATHNAME "."
#define PROJ_ID 0x6666
int CreateShm(int size);int DestroyShm(int shmid);int GetShm(int size);#endif
```

comm.c

```c
//comm.c
#include"comm.h"
static int CommShm(int size,int flags){	key_t key = ftok(PATHNAME,PROJ_ID);	if(key < 0)	{		perror("ftok");		return -1;	}	int shmid = 0;	if((shmid = shmget(key,size,flags)) < 0)	{		perror("shmget");		return -2;	}	return shmid;}int DestroyShm(int shmid){	if(shmctl(shmid,IPC_RMID,NULL) < 0)	{		perror("shmctl");		return -1;	}	return 0;}int CreateShm(int size){	return CommShm(size,IPC_CREAT | IPC_EXCL | 0666);}int GetShm(int size){	return CommShm(size,IPC_CREAT);}
```

client.c

```c
//client.c
#include"comm.h"
int main(){	int shmid = GetShm(4096);	sleep(1);	char *addr = shmat(shmid,NULL,0);	sleep(2);	int i = 0;	while(i < 26)	{		addr[i] = 'A' + i;		i++;		addr[i] = 0;		sleep(1);	}	shmdt(addr);	sleep(2);	return 0;}
```

server.c

```c
//server.c
#include"comm.h" 
int main(){	int shmid = CreateShm(4096); 	char *addr = shmat(shmid,NULL,0);	sleep(2);	int i = 0;	while(i++ < 26)	{		printf("client# %s\n",addr);		sleep(1);	}	shmdt(addr);	sleep(2);	DestroyShm(shmid);	return 0;}
```

Makefile

```c
//Makefile
.PHONY:all
all:server client client:client.c comm.c	gcc -o $@ $^server:server.c comm.c	gcc -o $@ $^ .PHONY:cleanclean:rm -f client server
```

运行结果：
![[Pasted image 20240914192036.png]]
![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

（1）优点：我们可以看到使用共享内存进行进程之间的通信是非常方便的，而且函数的接口也比较简单，数据的共享还使进程间的数据不用传送，而是直接访问内存，加快了程序的效率。

（2）缺点：共享内存没有提供同步机制，这使得我们在使用共享内存进行进程之间的通信时，往往需要借助其他手段来保证进程之间的同步工作。

## 二、内存映射

mmap I/O的描述符间接说明内存映射是对文件操作。另外，mmap另外可以在无亲缘的进程之间提供共享内存区。这样，类似的两个进程之间就是可以进行了通信。

Linux提供了内存映射函数mmap, 它把文件内容映射到一段内存上(准确说是虚拟内存上，运行着进程), 通过对这段内存的读取和修改, 实现对文件的读取和修改。mmap()系统调用使得进程之间可以通过映射一个普通的文件实现共享内存。普通文件映射到进程地址空间后，进程可以像访问内存的方式对文件进行访问，不需要其他内核态的系统调用(read,write)去操作。

这里是讲设备或者硬盘存储的一块空间映射到物理内存，然后操作这块物理内存就是在操作实际的硬盘空间，不需要经过内核态传递。比如你的硬盘上有一个文件，你可以使用linux系统提供的mmap接口，将这个文件映射到进程一块虚拟地址空间，这块空间会对应一块物理内存，当你读写这块物理空间的时候，就是在读取实际的磁盘文件，就是这么直接高效。通常诸如共享库的加载都是通过内存映射的方式加载到物理内存的。

mmap系统调用并不完全是为了共享内存来设计的，它本身提供了不同于一般对普通文件的访问的方式，进程可以像读写内存一样对普通文件进行操作，IPC的共享内存是纯粹为了共享。

内存映射指的是将 ：进程中的1个虚拟内存区域 & 1个磁盘上的对象，使得二者存在映射关系。当然，也可以多个进程同时映射到一个对象上面。
![[Pasted image 20240914192045.png]]
![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

实现过程

- 内存映射的实现过程主要是通过Linux系统下的系统调用函数：mmap（）
    
- 该函数的作用 = 创建虚拟内存区域 + 与共享对象建立映射关系
    
- 其函数原型、具体使用 & 内部流程 如下
    

```c
/** * 函数原型 */ void *mmap(void *start, size_t length, int prot, int flags, int fd, off_t offset); /** * 具体使用（用户进程调用mmap（）） * 下述代码即常见了一片大小 = MAP_SIZE的接收缓存区 & 关联到共享对象中（即建立映射） */ mmap(NULL, MAP_SIZE, PROT_READ, MAP_PRIVATE, fd, 0); /** * 内部原理 * 步骤1：创建虚拟内存区域 * 步骤2：实现地址映射关系，即：进程的虚拟地址空间 ->> 共享对象 * 注：* a. 此时，该虚拟地址并没有任何数据关联到文件中，仅仅只是建立映射关系 * b. 当其中1个进程对虚拟内存写入数据时，则真正实现了数据的可见 */
```

优点

进程在读写磁盘的时候，大概的流程是：

以write 为例：进程（用户空间） -> 系统调用，进入内核 -> 将要写入的数据从用户空间拷贝到内核空间的缓存区 -> 调用磁盘驱动 -> 写在磁盘上面。

使用mmap之后进程（用户空间）--> 读写映射的内存 --> 写在磁盘上面。

（这样的优点是 避免了频繁的进入内核空间，进行系统调用，提高了效率）

### 2.1mmap系统调用

```c
 void *mmap(void *addr, size_t length, int prot, int flags,                  int fd, off_t offset);
```

这就是mmap系统调用的接口，mmap函数成功返回指向内存区域的指针，失败返回MAP_FAILED。

- addr，某个特定的地址作为起始地址，当被设置为NULL，标识系统自动分配地址。实实在在的物理区域。
    
- length说的是内存段的长度。
    
- prot是用来设定内存段的访问权限。
    

```c
PROT_READ	内存段可读
PROT_WRITE	内存段可写
PROT_EXEC	内存段可执行
PROT_NONE	内存段不能被访问
```

flags参数控制内存段内容被修改以后程序的行为。

```c
MAP_SHARED	进程间共享内存，对该内存段修改反映到映射文件中。提供了POSIX共享内存MAP_PRIVATE	内存段为调用进程所私有。对该内存段的修改不会反映到映射文件MAP_ANNOYMOUS	这段内存不是从文件映射而来的。内容被初始化为全0
MAP_FIXED	内存段必须位于start参数指定的地址处，start必须是页大小的整数倍（4K整数倍）
MAP_HUGETLB	按照大内存页面来分配内存空间
```

fd参数是用来被映射文件对应的文件描述符，通过open系统调用得到，offset设定从何处进行映射。

### 2.2mmap用于共享内存的方式

1、我们可以使用普通文件进行提供内存映射，例如，open系统调用打开一个文件，然后进行mmap操作，得到共享内存，这种方式适用于任何进程之间。

2、可以使用特殊文件进行匿名内存映射，这个相对的是具有血缘关系的进程之间，当父进程调用mmap，然后进行fork，这样父进程创建的子进程会继承父进程匿名映射后的地址空间，这样，父子进程之间就可以进行通信了。相当于是mmap的返回地址此时是父子进程同时来维护。

3、另外POSIX版本的共享内存底层也是使用了mmap。所以，共享内存在在posix上一定程度上就是指的内存映射了。

## 三、mmap和System V共享内存的比较

共享内存：
![[Pasted image 20240914192139.png]]
![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

这是System V版本的共享内存（以下我们统称为shm），下面看下mmap的：
![[Pasted image 20240914192146.png]]
![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

mmap是在磁盘上建立一个文件，每个进程地址空间中开辟出一块空间进行映射。而shm共享内存，每个进程最终会映射到同一块物理内存。shm保存在物理内存，这样读写的速度肯定要比磁盘要快，但是存储量不是特别大，相对于shm来说，mmap更加简单，调用更加方便，所以这也是大家都喜欢用的原因。

另外mmap有一个好处是当机器重启，因为mmap把文件保存在磁盘上，这个文件还保存了操作系统同步的映像，所以mmap不会丢失，但是shmget在内存里面就会丢失，总之，共享内存是在内存中创建空间，每个进程映射到此处。内存映射是创建一个文件，并且映射到每个进程开辟的空间中，但在posix中的共享内存就是指这种使用文件的方式“内存映射”。

## 四、POSIX共享内存

### 4.1 IPC机制

共享内存是最快的可用IPC形式。它允许多个不相关(无亲缘关系)的进程去访问同一部分逻辑内存。

如果需要在两个进程之间传输数据，共享内存将是一种效率极高的解决方案。一旦这样的内存区映射到共享它的进程的地址空间，这些进程间数据的传输就不再涉及内核。这样就可以减少系统调用时间，提高程序效率。

共享内存是由IPC为一个进程创建的一个特殊的地址范围，它将出现在进程的地址空间中。其他进程可以把同一段共享内存段“连接到”它们自己的地址空间里去。所有进程都可以访问共享内存中的地址。如果一个进程向这段共享内存写了数据，所做的改动会立刻被有访问同一段共享内存的其他进程看到。

要注意的是共享内存本身没有提供任何同步功能。也就是说，在第一个进程结束对共享内存的写操作之前，并没有什么自动功能能够预防第二个进程开始对它进行读操作。共享内存的访问同步问题必须由程序员负责。可选的同步方式有互斥锁、条件变量、读写锁、纪录锁、信号灯。

实际上，进程之间在共享内存时，并不总是读写少量数据后就解除映射，有新的通信时，再重新建立共享内存区域。而是保持共享区域，直到通信完毕为止。

### 4.2 POSIX共享内存API

使用POSIX共享内存需要用到下面这些API：

```c
#include <sys/types.h>
#include <sys/stat.h>        /* For mode constants */
#include <sys/mman.h>
#include <fcntl.h>           /* For O_* constants */
#include <unistd.h>
int shm_open(const char *name, int oflag, mode_t mode);int shm_unlink(const char *name);int ftruncate(int fildes, off_t length);void *mmap(void *addr, size_t len, int prot, int flags, int fildes, off_t off);int munmap(void *addr, size_t len);int close(int fildes);int fstat(int fildes, struct stat *buf);int fchown(int fildes, uid_t owner, gid_t group);int fchmod(int fildes, mode_t mode);
```

- shm_open：穿件并打开一个新的共享内存对象或者打开一个既存的共享内存对象, 与函数open的用法是类似的函数返回值是一个文件描述符,会被下面的API使用。
    
- ftruncate：设置共享内存对象的大小,新创建的共享内存对象大小为0。
    
- mmap：将共享内存对象映射到调用进程的虚拟地址空间。
    
- munmap：取消共享内存对象到调用进程的虚拟地址空间的映射。
    
- shm_unlink：删除一个共享内存对象名字。
    
- close：当shm_open函数返回的文件描述符不再使用时,使用close函数关闭它。
    
- fstat：获得共享内存对象属性的stat结构体. 结构体中会包含共享内存对象的大小(st_size), 权限(st_mode), 所有者(st_uid), 归属组 (st_gid)。
    
- fchown：改变一个共享内存对象的所有权。
    
- fchmod：改变一个共享内存对象的权限。
    

## 五、共享内存操作

(1)shmget函数创建或者打开一个共享内存，返回一个共享内存的标识符：

```
#include <stdio.h>#include <stdlib.h>#include <unistd.h>#include <sys/ipc.h>#include <sys/shm.h> int main(int argc, char const *argv[]){ //使用ftok函数获取键值key_t mykey;if((mykey=ftok(".",100))==-1){    perror( "fail to ftok");    exit(1);}//通过shmget函数创建或者打开一个共享内存，返回一个共享内存的标识符int shmid;if((shmid = shmget(mykey,500,IPC_CREAT | 0666))==-1){    perror("fail to shmget");    exit(1);}    printf( "shmid = %d\n", shmid);    system("ipcs -m");return 0;}
```

运行结果：
![[Pasted image 20240914192211.png]]
![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

(2)共享内存映射(attach)

```c
#include <sys/types.h>
#include <sys/shm.h>
void *shmat(int shmid,const void *shmaddr， int shmflg);功能:映射共享内存参数    shmid：共享内存的id    shmaddr：映射的地址，设置为NULL为系统自动分配    shmflg：标志位        0：共享内存具有可读可写权限。        SHM_RDONLY：只读。返回值:       成功：映射的地址	失败：-1
```

注意：shmat函数使用的时候第二个和第三个参数一般设为NULL和0，即系统自动指定共享内存地址，并且共享内存可读可写。

(3)解除共享内存映射(detach)

```c
#include <sys/types.h>
#include <sys/shm.h>
int shmdt ( const void *shmaddr );功能:    解除共享内存的映射参数:    shmaddr：映射的地址，shmat的返回值返回值:    成功:0	失败:-1
```

(4)共享内存的实现

测试：打开一个共享内存并放入数据（发送端）

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/ipc.h>
#include <sys/shm.h>
#include <string.h>
int main(int argc, char const *argv[]){ //使用ftok函数获取键值key_t mykey;
if((mykey=ftok(".",100))==-1){    perror( "fail to ftok");    exit(1);}//通过shmget函数创建或者打开一个共享内存，返回一个共享内存的标识符int shmid;if((shmid = shmget(mykey,500,IPC_CREAT | 0666))==-1){    perror("fail to shmget");    exit(1);}    printf( "shmid = %d\n", shmid);    system("ipcs -m");    //使用shmat函数映射共享内存的地址    //声明一个指针变量指向共享内存，则操作指针变量text，就相当于操作共享内存    char *text;    if((text = shmat(shmid,NULL,0)) == (void *)-1){    perror( "fail to shmat");    exit(1);}//通过shmat的返回值对共享内存操作strcpy(text,"hello world");//操作完毕后要接触共享内存的映射if(shmdt(text) == -1){perror("fail to shmdt");exit(1);}  system("ipcs -m");return 0;}
```

测试：打开一个共享内存并放入数据（接收端）

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/ipc.h>
#include <sys/shm.h>
#include <string.h>
int main(int argc, char const *argv[]){ //使用ftok函数获取键值key_t mykey;if((mykey=ftok(".",100))==-1){    perror( "fail to ftok");    exit(1);}//通过shmget函数创建或者打开一个共享内存，返回一个共享内存的标识符int shmid;if((shmid = shmget(mykey,500,IPC_CREAT | 0666))==-1){    perror("fail to shmget");    exit(1);}    printf( "shmid = %d\n", shmid);    system("ipcs -m"); char *text;if((text = shmat(shmid,NULL,0)) == (void*)-1){perror( "fail to shmat");exit(1);}//获取共享内存中的数据printf( "text = %s\n", text);//解除共享内存映射if(shmdt(text)==-1){perror( "fail to shmdt");exit(1);}  system("ipcs -m");return 0;}
```

运行如下：
![[Pasted image 20240914192223.png]]
![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

用结构体指针来实现：

```
写端#include <stdio.h>#include <stdlib.h>#include <unistd.h>#include <sys/ipc.h>#include <sys/shm.h>#include <string.h>typedef struct{int a;char b;}MSG;int main(int argc, char const *argv[]){ //使用ftok函数获取键值key_t mykey;if((mykey=ftok(".",100))==-1){    perror( "fail to ftok");    exit(1);}//通过shmget函数创建或者打开一个共享内存，返回一个共享内存的标识符int shmid;if((shmid = shmget(mykey,500,IPC_CREAT | 0666))==-1){    perror("fail to shmget");    exit(1);}    printf( "shmid = %d\n", shmid);    system("ipcs -m");    //使用shmat函数映射共享内存的地址    //声明一个指针变量指向共享内存，则操作指针变量text，就相当于操作共享内存    //char *text;       MSG *text; if((text = shmat(shmid,NULL,0)) == (void *)-1){    perror( "fail to shmat");    exit(1);} //通过shmat的返回值对共享内存操作//strcpy(text,"hello world");text->a=100;text->b='w';//操作完毕后要接触共享内存的映射if(shmdt(text) == -1){perror("fail to shmdt");exit(1);}  system("ipcs -m");return 0;}
```

运行如下：
![[Pasted image 20240914192234.png]]
![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

shmctl函数

```
#include <sys/ipc. h>#include <sys /shm.h>int shmct1(int shmid, int cmd, struct shmid_ds *buf);功能:    设置或者获取共享内存你的属性参数:    shmid:共享内存的id    cmd:执行操作的命令         IPC_STAT 获取共享内存的属性        IPC__SET 设置共享内存的属性        IPC_RMID 删除共享内存    shmid_ds:共享内存的属性结构体 返回值:        成功:0	失败: -1
```

测试：//通过shmctl函数删除共享内存

```
#include <stdio.h>#include <stdlib.h>#include <unistd.h>#include <sys/ipc.h>#include <sys/shm.h> int main(int argc, char const *argv[]){ //使用ftok函数获取键值key_t mykey;if((mykey=ftok(".",100))==-1){    perror( "fail to ftok");    exit(1);}//通过shmget函数创建或者打开一个共享内存，返回一个共享内存的标识符int shmid;if((shmid = shmget(mykey,500,IPC_CREAT | 0666))==-1){    perror("fail to shmget");    exit(1);}    printf( "shmid = %d\n", shmid);    system("ipcs -m");return 0;}//通过shmctl函数删除共享内存if(shmctl(shmid,IPC_RMID,NULL) == -1){    perror("fail to shmctl");    exit(1);}system( "ipcs -m");return 0;}
```

运行结果：
![[Pasted image 20240914192241.png]]
![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

2023年往期回顾

[C/C++发展方向（强烈推荐！！）](http://mp.weixin.qq.com/s?__biz=Mzg4NDQ0OTI4Ng==&mid=2247487749&idx=1&sn=e57e6f3df526b7ad78313d9428e55b6b&chksm=cfb9586cf8ced17a8c7830e380a45ce080c2b8258e145f5898503a779840a5fcfec3e8f8fa9a&scene=21#wechat_redirect)  

[Linux内核源码分析（强烈推荐收藏！](http://mp.weixin.qq.com/s?__biz=Mzg4NDQ0OTI4Ng==&mid=2247487832&idx=1&sn=bf0468e26f353306c743c4d7523ebb07&chksm=cfb95831f8ced127ca94eb61e6508732576bb150b2fb2047664f8256b3da284d0e53e2f792dc&scene=21#wechat_redirect)[）](http://mp.weixin.qq.com/s?__biz=Mzg4NDQ0OTI4Ng==&mid=2247487832&idx=1&sn=bf0468e26f353306c743c4d7523ebb07&chksm=cfb95831f8ced127ca94eb61e6508732576bb150b2fb2047664f8256b3da284d0e53e2f792dc&scene=21#wechat_redirect)  

[从菜鸟到大师，用Qt编写出惊艳世界应用](http://mp.weixin.qq.com/s?__biz=Mzg4NDQ0OTI4Ng==&mid=2247488117&idx=1&sn=a83d661165a3840fbb23d0e62b5f303a&chksm=cfb95b1cf8ced20ae63206fe25891d9a37ffe76fd695ef55b5506c83aad387d55c4032cb7e4f&scene=21#wechat_redirect)  

[存储全栈开发：构建高效的数据存储系统](http://mp.weixin.qq.com/s?__biz=Mzg4NDQ0OTI4Ng==&mid=2247487696&idx=1&sn=b5ebe830ddb6798ac5bf6db4a8d5d075&chksm=cfb959b9f8ced0af76710c070a6db2677fb359af735e79c6378e82ead570aa1ce5350a146793&scene=21#wechat_redirect)

[突破性能瓶颈：释放DPDK带来无尽潜力](http://mp.weixin.qq.com/s?__biz=Mzg4NDQ0OTI4Ng==&mid=2247488150&idx=1&sn=2cc5ace4391d4eed060fa463fc73dd92&chksm=cfb95bfff8ced2e9589eecd66e7c6e906f28626bef6c1bce3fa19ae2d1404baa25ba3ebfedaa&scene=21#wechat_redirect)

[嵌入式音视频技术：从原理到项目应用](http://mp.weixin.qq.com/s?__biz=Mzg4NDQ0OTI4Ng==&mid=2247488135&idx=1&sn=dcc56a3af5401c338944b9b94f24c699&chksm=cfb95beef8ced2f8231c647033188be3ded88cbacf7d75feab3613f4d7f8fd8e4c8106ac88cf&scene=21#wechat_redirect)

[](http://mp.weixin.qq.com/s?__biz=MzIwNzA1OTA2OQ==&mid=2657215412&idx=1&sn=d3c64ac21056b84814db9903b656853a&chksm=8c8d62a6bbfaebb09df1f727e201c401a1663596a719fa9cb4c7859158e62ebedb5bd5f13b8f&scene=21#wechat_redirect)[C++游戏开发，基于魔兽开源后端框架](http://mp.weixin.qq.com/s?__biz=Mzg4NDQ0OTI4Ng==&mid=2247486920&idx=1&sn=82ca8a72473850e431c01e7c39d8c8c0&chksm=cfb944a1f8cecdb7051863f58d334ff3dea8afae1ecd8bc693287b2f483d2ce8f94eed565253&scene=21#wechat_redirect)

linux内核108

C/C++开发92

mmap6

linux内核 · 目录

上一篇探索Linux进程：工作原理与应用实例分析下一篇Linux内核源码研读与实战演练 (版本5.6.18)

阅读 2527

​