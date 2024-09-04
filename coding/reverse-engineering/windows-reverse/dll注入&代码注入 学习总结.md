# 

pyikaaaa 看雪学苑

 _2021年10月11日 18:01_

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/kR3MmOkZ9r5KlJCc9CTC9enyibl0icLBSJ2wyLEG21fygTkptVoI80uzcBDGLvVu74l0EzBsIxDlQYibA371IWnAg/640?wx_fmt=jpeg&wxfrom=13&tp=wxpic)  

本文为看雪论坛优秀文章  
看雪论坛作者ID：pyikaaaa

  

## **CreateRemoteThread**

  

思路：在目标进程中申请一块内存并向其中写DLL路径，然后调用 CreateRemoteThread ，（在自己进程中 创建远程线程到到目标进程）在目标进程中创建一个线程。LoadLibrary()”函数作为线程的启动函数，来加载待注入的DLL文件 ，LoadLibrary()参数 就是存放DLL路径的内存指针。这时需要目标进程的4个权限（PROCESS_CREATE_THREAD,PROCESS_QUERY_INFORMATION,PROCESS_VM_OPERATION,PROCESS_VM_WRITE）

```
    //计算DLL路径名所需的字节数
```

## **RtlCreateUserThread**

  

RtlCreateUserThread()”调用“NtCreateThreadEx(),这意味着“RtlCreateUserThread()”是“NtCreateThreadEx()”的一个小型封装函数。

```
    HANDLE hProcess = OpenProcess(PROCESS_ALL_ACCESS, FALSE, dwProcessId);
```

  

总结：

openprocess 获得目标进程句柄

getprocaddress 获得loadlibrary地址

getprocaddress 获得RtlCreateUserThread地址

获得dll文件==路径==大小

virtualalloc 在目标进程中开辟路径大小的空间

writeprocess写dll路径名进内存

bStatus = (BOOL)RtlCreateUserThread(  
hProcess,  
NULL,  
0,  
0,  
0,  
0,  
LoadLibraryAddress,  
lpBaseAddress, 存有dll路径的内存地址 指针类型  
&hRemoteThread,  
NULL);

##   

## **NtCreateThreadEx**

```
    memset(&ntbuffer, 0, sizeof(NtCreateThreadExBuffer));
```

  

总结：openprocess 获得目标进程句柄

getprocaddress 获得loadlibrary地址

getprocaddress 获得NtCreateThreadEx地址

获得dll文件==路径==大小

virtualalloc 在目标进程中开辟路径大小的空间

writeprocess写dll路径名进内存

利用NtCreateThreadEx 进行 dll注入

以上三种远程线程注入函数的区别：

CreateRemoteThread 和RtlCreateUserThread都调用 NtCreateThreadEx创建线程实体。

RtlCreateUserThread不需要csrss验证登记 需要自己结束自己 而CreateRemoteThread 不一样，不用自己结束自己。

线程函数不由createthread执行 而是kernal32！baseThreadStart 或者 kernal32！baseThreadInitThunk 执行，结束后 还会调用 exitthread 和 rtlexituserthread 结束线程自身 。

##   

## **ZwCreateThreadEx**

  

同理，与CreateRemoteThread或RtlCreateUserThread或NtCreateThreadEx用法类似，也是创建远程线程实现注入。

  

**反射式dll注入**  

  

在别人的内存里调用自己编写的dll导出函数 ，自己dll导出函数里实现自我加载（加载PE的整个过程），少了使用LoadLibrary的过程。

反射式注入方式并没有通过LoadLibrary等API来完成DLL的装载，DLL并没有在操作系统中”注册”自己的存在，因此ProcessExplorer等软件也无法检测出进程加载了该DLL。

```
//LoadRemoteLibraryR 函数说明
```

  

LoadRemoteLibraryR核心代码

```
            //检查库是否有ReflectiveLoader
```

  

总结：

在自己进程内存中heapalloc，将dll文件 readfile进heapalloc出的内存中，openprocess 获得进程句柄。

LoadRemoteLibraryR 函数获得dll入口函数的地址，并且利用远程线程注入rtlcreateuserprocess 实现 对dll入口函数的调用。

{获得dl文件的入口点偏移 ：GetReflectiveLoaderOffset(lpBuffer);//lpbuffer:堆内存的指针 指向存有dll文件的堆内存空间

为映像分配内存 virtualalloc，writeprocessmemory 映像写进目标进程内存，函数真实地址是 分配的内存首地址加上函数在dll文件中的偏移。

远程线程函数注入 call

}

  

**SetWindowsHookE**  

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

```
    DWORD dwThreadId = getThreadID(dwProcessId);   
```

##   

## **APC注入**

  

### APC 异步过程调用

  

异步过程调用是一种能在特定线程环境中异步执行的系统机制。

MSDN说,要使用例如线程调用SignalObjectAndWait、WaitForSingleObjectE、WaitForMultipleObjectsEx、SleepEx等等等这些函数才会触发。

使用QueueUserAPC函数插入APC函数，QueueUserAPC内部调用的是NtQueueApcThread，再内部是KiUserApcDispatcher。  

  

攻击者可以将恶意代码作为一个APC函数插入APC队列（调用QueueUserAPC或NtQueueApcThread），而这段恶意代码一般实现加载DLL的操作，实现DLL注入。

###   

### **注入方法的原理：**

  

1、当对面程序执行到某一个上面的等待函数的时候,系统会产生一个中断。

2、当线程唤醒的时候,这个线程会优先去Apc队列中调用回调函数。

3、我们利用QueueUserApc,往这个队列中插入一个回调。

4、插入回调的时候,把插入的回调地址改为LoadLibrary,插入的参数我们使用VirtualAllocEx申请内存,并且写入进去，写入的是Dll的路径。

```
//1.查找窗口
```

  

```
DWORD QueueUserAPC(
```

  

PAPCFUNCpfnAPC ：这个函数将在指定线程执行an alertable wait operation操作时被调用。

##   

## **AppLint_DLLs 注册表项注入**

  

### 1、概述：

  

这种注入方式有他的弊端，那就是注入的程序必须加载user32.dll，也就说通常是那些GUI程序才可以。原因是，每当启动一个GUI程序，他都会扫描注册表的AppInit_DLLs项，看看其中有没有制定的目标库，如果有，那就在程序运行时，首先主动加载该动态库，所以我们只需要将Dll完整路径写在这个位置，就可完成注入。

  

值得一提的是，这样的Dll默认加载机制，，携带恶意代码的Dll也会在此处注册，所以Win7之后，Windows在同一个路径下，增加了一个表项LoadAppInit_DLLs，它的默认值是0，也就是说不管你在AppInit_DLLs注册什么Dll，那都没用，不会被默认加载，所以想要顺利完成注入，需要完成这个LoadAppInit_DLLs的修改，将其修改为1。

###   

### 2、流程：

  

HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Windows下找到AppInit_DLLs和LoadAppInit_DLLs项。  

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

将AppInit_DLLs值修改为目标Dll完整路径。  
![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

修改LoadAppInit_DLLs值为1(意思是允许默认加载动态库)。

##   

## **AppCert DLL 注入**

  

如果有进程使用了CreateProcess、CreateProcessAsUser、CreateProcessWithLoginW、CreateProcessWithTokenW或WinExec  
函数，那么此进程会获取HKEY_LOCAL_MACHINE\System\CurrentControlSet\Control\SessionManager\AppCertDlls注册表项，此项下的dll都会被加载到此进程。

  
此技术的实现逻辑是CreateProcess、CreateProcessAsUser、CreateProcessWithLoginW、CreateProcessWithTokenW或WinExec  
函数在创建进程的时候其内部会调用BasepIsProcessAllowed函数，而BasepIsProcessAllowed则会打开AppCertDlls注册表项，将此项下的dll都加载到进程中。

  
CreateProcess过程有七个阶段，BasepIsProcessAllowed的过程发生在阶段2——创建文件映像和兼容性检查之间。

  
值得注意的是win xp-win 10 默认不存在这个注册表项，为了利用该技术需要自行创建AppCertDlls项。

补充：CreateProcess过程有七个阶段详解CreateProcess调用内核创建进程的过程 - Gotogoo - 博客园 (cnblogs.com)_（https://www.cnblogs.com/Gotogoo/p/5262536.html）_

  

**傀儡进程注入**  

  

==也是进程内存替换==

之前分析的病毒 使用傀儡进程注入(进程内存替换)达到进程隐藏的目的 ，傀儡进程是将目标进程的映射文件替换为指定的映射文件,替换后的进程称之为傀儡进程;常常有恶意程序将隐藏在自己文件内的恶意代码加载进目标进程,而在加载进目标进程之前，会利用ZwUnmpViewOfSection或者NtUnmapViewOfSection进行相关设置。

  

流程概述：

  

直接将自身代码注入傀儡进程，不需要DLL。首先用CreateProcess来创建一个挂起的IE进程，创建时候就把它挂起。然后得到它的装载基址，使用函数ZwUnmapViewOfSection来卸载这个这个基址内存空间的数据。

  
再用VirtualAllocEx来个ie进程重新分配内存空间，大小为要注入程序的大小(就是自身的imagesize)。使用WriteProcessMemory重新写IE进程的基址，就是刚才分配的内存空间的地址。再用WriteProcessMemory把自己的代码写入IE的内存空间。用SetThreadContext设置下进程状态，最后使用ResumeThread继续运行IE进程。

  

### **相关技术点**

  

#### 1、创建挂起进程

  

系统函数CreateProcessW中参数dwCreationFlgs传递CREATE_SUSPEND便可以创建一个挂起的进程，进程被创建之后系统会为它分配足够的资源和初始化必要的操作(常见的操作有:为进程分配空间,加载映像文件,创建主进程,将EIP指向代码入口点,并将主线程挂起等)。

```
CreateProcessA(strTargetProcess.c_str(),NULL,NUL NULL, FALSE,CREATE_SUSPENDED, NULL, NULL,&stSi, &stPi)
```

####   

#### 2、利得到当前的线程上下文

  

相关的API和结构信息如下：

```
BOOL WINAPI GetThreadContext(
```

  

获得线程信息代码：

```
CONTEXT stThreadContext;
```

####   

#### 3、清空目标进程的内存空间

  

目标进程被初始化后，进程的映像文件也随之被加载进对应的内存空间。傀儡进程在替换之前必须将目标进程的内容清除掉。此时要用到另外一个系统未文档化的函数NtUnmapViewOfSection，需要自行从ntdll.dll中获取。该函数需要指定的进程加载的基地址，基地址即是从第2步中的上下文取得。相关的函数说明及基地址计算方法如下：

```
NTSTATUS NtUnmapViewOfSection(
```

  

ontext.Ebx+ 8 = 基地址的地址，因此从context.Ebx + 8的地址读取4字节的内容并转化为DWORD类型，既是进程加载的基地址。

  

#### 4、重新分配空间

  

在第3步中，NtUnmapViewOfSection将原始空间清除并释放了，因此在写入傀儡进程之前需要重新在目标进程中分配大小足够的空间。需要用到跨进程内存分配函数VirtualAllocEx。

一般情况下，在写入傀儡进程之前，需要将傀儡进程对应的文件按照申请空间的首地址作为基地址进行“重定位”，这样才能保证傀儡进程的正常运行。为了避免这一步操作，可以以傀儡进程PE文件头部的建议加载基地址作为VirtualAllocEx 的lpAddress参数，申请与之对应的内存空间，然后以此地址作为基地址将傀儡进程写入目标进程，就不会存在重定位问题。关于“重定位”的原理可以自行网络查找相关资料。

  

#### 5、写入傀儡进程

  

准备工作完成后，现在开始将傀儡进程的代码写入到对应的空间中，注意写入的时候要按照傀儡进程PE文件头标明的信息进行。一般是先写入PE头，再写入PE节，如果存在附加数据还需要写入附加数据。

  

#### 6、恢复现场并运行傀儡进程

  

在第2步中，保存的线程上下文信息需要在此时就需要及时恢复了。由于目标进程和傀儡进程的入口点一般不相同，因此在恢复之前，需要更改一下其中的线程入口点，需要用到系统函数SetThreadContext。将挂起的进程开始运行需要用到函数ResumeThread。

####   

#### 7、傀儡进程创建过程总结：

  

(1) CreateProcess一个进程，并挂起，即向dwCreationFlags 参数传入CREATE_SUSPENDED；

(2) GetThreadContext获取挂起进程CONTEXT，其中，EAX为进程入口点地址，EBX指向进程PEB：

(3) ZwUnmapViewOfSection卸载挂起进程内存空间数据；

(4) VirtualAlloc分配内存空间；

(5) WriteProcessMemory将恶意代码写入分配的内存；

(6) SetThreadContext设置挂起的进程的状态；

(7) ResumeThread唤醒进程运行。

傀儡进程是恶意软件隐藏自身代码的常用方式，在调式过程中，若遇到傀儡进程，需要将创建的子进程数据从内存中dump出来，作为PE文件单独调试，dump的时机为ResumeThead调用之前，此时傀儡进程内存数据已经完全写入，进程还未正式开始运行。

若dump后文件无法运行，OD加载失败，则需要做如下修复：

(1) FileAlignment值修改为SectionAlignment值

(2) 所有section的Raw Address值修改为Virtual Address

###   

### 代码：

  

32位环境下的代码：

```
0x68, 0xCC, 0xCC, 0xCC, 0xCC,   // push 0xDEADBEEF (为返回地址占位)
```

  

64位环境：

```
0x50,                                                       // push rax (保存RAX寄存器)
```

  

在我们想目标进程注入这段代码之前，以下占位符需要修改填充：

·返回地址（代码桩执行完毕之后，线程恢复应回到的地址）

·DLL路径名称

·LoadLibrary()函数地址

而这也是进行劫持，挂起，注入和恢复线程这一系列操作的时机。

32位：

```
memcpy((void *)((unsigned long)sc + 1), &oldIP, 4);
```

  

64位：

```
memcpy(sc + 3, &oldIP, sizeof(oldIP));
```

  

32位注入核心代码：

```
HANDLE hProcess = OpenProcess(PROCESS_ALL_ACCESS, FALSE, dwProcessId);
```

  

64位同理。

  

**线程执行劫持**  

  

线程执行劫持技术和傀儡进程技术相似，傀儡进程替换的是整个进程而线程执行劫持替换的只是某一个线程。

  
线程执行劫持也需要先在RWX内存中写入payload，写入完毕后直接将线程执行地址替换为payload地址就行了：

```
HANDLE t = OpenThread(THREAD_SET_CONTEXT, FALSE, thread_id);//打开线程
```

##   

## **Atom Bombing**

  

### 1、Atom Table

  

是一个存储字符串和相应标识符的系统定义表。

应用程序将一个字符串放入一个 Atom 表中，并接收一个 16 位整数 (WORD) 作为标识 (称为 Atom)，可通过该标识访问字符串内容，实现进程间的数据交换。

  

#### 分类：

  

(1) Global Atom Table

所有应用程序可用。

当一个进程将一个字符串保存到 Global Atom Table 时，系统生成一个在系统范围内唯一的 atom，来标示该字符串。在系统范围之内所有的进程都可以通过该 atom(索引) 来获得这个字符串,从而实现进程间的数据交换。

(2) Local Atom Table

只有当前程序可用，相当于定义一个全局变量，如果程序多次使用该变量，使用 Local Atom Table 仅需要一次内存操作。

  

#### **常用 API：**

  

添加一个 Global Atom：

```
ATOM WINAPI GlobalAddAtom(In LPCTSTR lpString);
```

  

删除一个 Global Atom：

```
ATOM WINAPI GlobalDeleteAtom(In ATOM nAtom);
```

  

查找指定字符串对应的 Global Atom：

```
ATOM WINAPI GlobalFindAtom(In LPCTSTR lpString);
```

  

获取指定 atom 对应的字符串：

```
UINT WINAPI GlobalGetAtomName(
```

###   

### **Atom Bombing注入**

  

#### 1、将任意数据写入目标进程地址空间中的任意位置

  

整体思路：

使用GlobalAddAtom创建一个原子，写入不含null的字符串，让目标进程调用GlobalGetAtomNameA就可以向目标进程任意地址写入任意代码了。

细节：

自身进程通过 GlobalAddAtom 将 shellcode 添加到 Global Atom Table 中。

通过 APC 注入（使用APC中的NtQueueApcThread函数另目标进程调用GlobalGetAtomNameA。使用NtQueueApcThread而不是QueueUserAPC是因为 GlobalGetAtomNameA有三个参数，NtQueueApcThread能传递三个参数而QueueUserAPC只能传递一个参数），使目标进程调用 GlobalGetAtomName， 即可从 Global Atom Table 中获取 shellcode。

总结：

通过GlobalAddAtom函数把数据放到原子表，用APC在目标线程用GlobalGetAtomName把原子表里的数据放到远程内存地址里。

```
HANDLE th = OpenThread(THREAD_SET_CONTEXT | THREAD_QUERY_INFORMATION, FALSE,thread_id);
```

####   

#### 2、执行 shellcode

  

目标进程调用 GlobalGetAtomName 从 Global Atom Table 中获取 shellcode 后，需要先保存 shellcode 再执行。

找到一段 RW 的内存（ KERNELBASE 数据段后未使用的空间）写入数据，构造 ROP 链实现 shellcode 的执行。

ROP 链实现了以下功能：

①申请 RWX 内存

②将 shellcode 从 RW 内存处拷贝到 RWX 内存储

③执行

  

注入后需要恢复目标进程的执行。

Windows 8.1 update 3 和 Windows 10 添加了一个新的保护机制 CFG。

参数，NtQueueApcThread能传递三个参数而QueueUserAPC只能传递一个参数），使目标进程调用 GlobalGetAtomName， 即可从 Global Atom Table 中获取 shellcode。

总结：

通过GlobalAddAtom函数把数据放到原子表，用APC在目标线程用GlobalGetAtomName把原子表里的数据放到远程内存地址里。

```
HANDLE th = OpenThread(THREAD_SET_CONTEXT | THREAD_QUERY_INFORMATION, FALSE,thread_id);
```

  

  

Windows 8.1 update 3 和 Windows 10 添加了一个新的保护机制 CFG。

##   

## **利用内存映射文件实现注入**

  

内存映射文件，是由一个文件到一块内存的映射。Win32提供了允许应用程序把文件映射到一个进程的函数。

  
(CreateFileMapping)。内存映射文件与虚拟内存有些类似，通过内存映射文件可以保留一个地址空间的区域，同时将物理存储器提交给此区域，内存文件映射的物理存储器来自一个已经存在于磁盘上的文件，而且在对该文件进行操作之前必须首先对文件进行映射。使用内存映射文件处理存储于磁盘上的文件时，将不必再对文件执行I/O操作，使得内存映射文件在处理大数据量的文件时能起到相当重要的作用。

在Windows下，创建操作共享内存的API主要有CreateFileMapping、MapViewOfFile、OpenFileMapping、FlushViewOfFile、UnmapViewOfFile等。

**利用目标进程共享内存进行注入**

```
//打开共享内存
```

  

**创建共享内存进行注入**

首先，使用CreateProcess创建挂起进程。

  
利用了内存映射文件的原理，那么内存映射的一套的流程也基本都用到了，CreateFileMapping创建共享内存的内存映射对象。

  
MapViewOfFile得到该内存空间的映射地址。

  
RtlMoveMemory将shellcode与PE文件信息拷贝到内存映射对象中。

  
ZwMapViewOfSection将内存映射对象与挂起目标进程关联在一起，这样目标进程中就存在了shellcode与PE文件。

  
ZwQueryInformationThread获取目标进程主线程的入口地址。

  
CreateRemoteThread创建一个主线程。

  
最后使用QueueUserAPC向创建的主线程插入一个APC执行shellcode装载随后的PE文件。

  
恢复执行。

注意：所有代码均不是原创，此文只是学习过程进行总结。  

  

_https://github.com/BreakingMalwareResearch/atom-bombing  
_

  

_https://github.com/BreakingMalware/PowerLoaderEx_

  

  

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E) 

  

**看雪ID：pyikaaaa**

https://bbs.pediy.com/user-home-921642.htm

*本文由看雪论坛 pyikaaaa 原创，转载请注明来自看雪社区

  

  

[![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458396801&idx=1&sn=2d13a137f3d17766558e8689d48fd341&chksm=b18f180b86f8911d25d2ff9e0c50068c4e202187e72f9b77fe1bfcfcc14ffd662805264c7dab&scene=21#wechat_redirect)

  

[![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)](https://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458387399&idx=2&sn=38495add2a3a3677b2c436581c07e432&scene=21#wechat_redirect)

  

**#** **往期推荐**

1.[栈溢出漏洞利用（绕过ASLR）](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458396195&idx=2&sn=3517949e465cddf16e347f801bfebe1f&chksm=b18f16a986f89fbf85b00510bee4577728c95cc58d9be79a2b5f09896c5c44f691c637791853&scene=21#wechat_redirect)  

2. [大杀器Unidbg真正的威力](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458395762&idx=2&sn=f9662663e130096ffd01c86658813742&chksm=b18f14f886f89dee2fbc6e37a4aae1252e578f2a4abeae40b238a3f850069bd25c10a226000e&scene=21#wechat_redirect)

3.[16位实模式切换32位保护模式过程详解](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458392978&idx=2&sn=10921a9bf6c258c747315bcc91d4e18c&chksm=b18f291886f8a00e68389072d7e64fd9c93ee9daacbf236a9cb8028134519fad7740ce0f4756&scene=21#wechat_redirect)

4. [高Glibc版本下的堆骚操作解析](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458392560&idx=1&sn=73533c60d701ad5816545520bc889135&chksm=b18f277a86f8ae6ce880fd532b70bbbb70180859299f7476bd1d3033a43ea7e0b3fb6709c460&scene=21#wechat_redirect)

5.[新人PWN堆Heap总结off-by-null专场](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458392540&idx=1&sn=75525c0ee1d799edbf118b4e19b81e3a&chksm=b18f275686f8ae40bacfec040a64b645227ce1f5554cdc5e95c337b63573d1ec67a57bebec01&scene=21#wechat_redirect)

6. [CVE-2012-3569 VMware OVF Tool格式化字符串漏洞分析](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458392532&idx=1&sn=009b3954d88c8732df14e2d442e6cf04&chksm=b18f275e86f8ae48ec217fc9b28c98cdcfa94822de52fd359c75d81de835ce23605f46fcdf79&scene=21#wechat_redirect)

  

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

公众号ID：ikanxue  

官方微博：看雪安全

商务合作：wsc@kanxue.com

  

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**球分享**

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**球点赞**

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**球在看**

  

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

点击“阅读原文”，了解更多！

阅读原文

阅读 3292

​