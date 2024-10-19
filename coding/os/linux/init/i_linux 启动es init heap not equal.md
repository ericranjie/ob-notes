
[mob6454cc73e9a6](https://blog.51cto.com/u_16099296)2024-09-08 09:02:57

我整理的一些关于【IT人转架构设计】的项目学习资料+视频（附讲解～～）和大家一起分享、学习一下：

[https://d.51cto.com/bLN8S1](https://d.51cto.com/bLN8S1)

继2019.8.1

四．Linux常用命令

![linux 启动es init heap not equal_硬链接](https://s2.51cto.com/images/blog/202409/08014500_66dc911c82fdb56640.png?x-oss-process=image/watermark,size_16,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_30,g_se,x_10,y_10,shadow_20,type_ZmFuZ3poZW5naGVpdGk=/resize,m_fixed,w_1184)

注：学习命令要尽量减少与计算机的交互

2.系统的启动

![linux 启动es init heap not equal_硬链接_02](https://s2.51cto.com/images/blog/202409/08014500_66dc911c94e9920039.png?x-oss-process=image/watermark,size_16,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_30,g_se,x_10,y_10,shadow_20,type_ZmFuZ3poZW5naGVpdGk=/resize,m_fixed,w_1184)

至此，将内核程序加载完成（即kernel）,但并不能运行。

init（初始程序）

Init将操作系统分为0-6七个级别，每一个级别都会运行对应的应用程序。

init程序会指定默认启动级别：

![linux 启动es init heap not equal_操作系统_03](https://s2.51cto.com/images/blog/202409/08014500_66dc911cabed657875.png?x-oss-process=image/watermark,size_16,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_30,g_se,x_10,y_10,shadow_20,type_ZmFuZ3poZW5naGVpdGk=/resize,m_fixed,w_1184)

读取默认级别：3或5

Chkconfig命令：指定服务开机时在哪个级别上启动或关闭

rc.\*（\*后匹配的是0-6）；rc.\*d每一个级别所需启动的进程

rc.local系统启动最后读取这个文件，管理员可将需要开机就进行的命令添加在这里。

扩展：

command \[选项\] \[参数\]

\[\]中括号代表可有可无，指定实现命令的某个特定功能；

\<>代码命令执行的对象，若没有加\<>,则不能省略。

-h(或--help)：查看帮助，

长整形选项（如-help），不可以合并使用；短整型选项（如-h），可以合并使用；

--list：列出所有进程；

--level：等级（可省略），--level 345 Name \<on/off>指定启动级别。

扩展：命令 子命令 \[选项\] \[参数\]

3.环境变量

变量：一段被命名的内存空间；分为全局变量和局部变量。

环境变量都以$符号调用，如echo$PATH。

提问：若命令不再PATH路径下，如何解决？

答：第一种方法，通过绝对路径来运行该命令即可。

绝对路径执行的是命令本身，而有些命名，是系统默认的别名（可自己添加）（别名alias）；\\ls—使用反斜线直接运行；which command—查找命令的绝对路径

扩展：

PATH=$PATH:/***/***；

Which：找到命令所存放的地方；

env:查看当前用户的环境变量；

存储单位：bit(0/1)—B（8bit=1B）—kb—M—G—T(以1024为换算单位)

4.常用的Linux命令的基本使用

- Ls常用选项：

-l：查看文件详细信息（属性信息）

-i：inode（属性信息）

-h：human（人类可读）

-F：显示文件后的标记（用来区分文件类型）；

-r：倒叙显示文件内容；

-a：显示所有文件（包括隐藏文件）；

$pwd:默认有一个这个环境变量。

例：以-ll读取的属性信息：

Lrwxrwxrwx. 1 root root 14 Aug 1 00:13 system-release   centos-release

L—链接文件；-—普通文件；d—目录文件；

r—读取；w—写；x—运行；.—特殊权限位；

root—文件所属主；root—文件所属组；

system-release   centos-release—链接文件名

以-li读取的属性信息：

976706 –rw-r-----. 1 root root 3.2k Jul 31 2014 sudo-ldap.conf

3.2k—文件大小；Jul 31 2014—修改的时间；sudo-ldap.conf—文件名

扩展：

存储设备必须安装文件系统；

格式化操作就是在安装文件系统；

Windows系统下常见的文件系统格式有NTFS和FAT32；

Linux下的文件系统格式有ext4。

提问：硬链接与软连接的区别有哪些？

答：第一点，软连接和源文件的inode节点号不同，进而指向的block也不同，软连接block中存放了源文件的路径名。

第二点，不能对目录创建硬链接，不能对不同文件系统创建硬链接，不能对不存在的文件创建硬链接；可以对目录创建软连接，可以跨文件系统创建软连接，可以对不存在的文件创建软连接。

第三点，删除硬链接文件或者删除源文件任意之一，文件实体并未被删除，只有删除了源文件和所有对应的硬链接文件，文件实体才会被删除；删除源文件，软链接依然存在，但无法访问源文件内容。

- Cd常用选项：

![linux 启动es init heap not equal_硬链接_04](https://s2.51cto.com/images/blog/202409/08014500_66dc911cc3c9247605.png?x-oss-process=image/watermark,size_16,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_30,g_se,x_10,y_10,shadow_20,type_ZmFuZ3poZW5naGVpdGk=/resize,m_fixed,w_1184)

- Touch—修改时间戳

![linux 启动es init heap not equal_操作系统_05](https://s2.51cto.com/images/blog/202409/08014500_66dc911cdbb0e76044.png?x-oss-process=image/watermark,size_16,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_30,g_se,x_10,y_10,shadow_20,type_ZmFuZ3poZW5naGVpdGk=/resize,m_fixed,w_1184)

\[\[CC\]YY\]MMDDhhmm\[.ss\]代表2019年 08月 01日 16时 36分 10秒

- Mkdir—创建文件夹

-p：递归创建多级子目录；-v：详细显示创建过程；

Tree：以树状形式显示目录及文件结构

![linux 启动es init heap not equal_硬链接_06](https://s2.51cto.com/images/blog/202409/08014501_66dc911d1472531240.png?x-oss-process=image/watermark,size_16,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_30,g_se,x_10,y_10,shadow_20,type_ZmFuZ3poZW5naGVpdGk=/resize,m_fixed,w_1184)

-L NUM—查看多少级子目录

![linux 启动es init heap not equal_硬链接_07](https://s2.51cto.com/images/blog/202409/08014501_66dc911d430b972938.png?x-oss-process=image/watermark,size_16,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_30,g_se,x_10,y_10,shadow_20,type_ZmFuZ3poZW5naGVpdGk=/resize,m_fixed,w_1184)

-d—只显示目录文件

- Rm—删除文件

![linux 启动es init heap not equal_文件系统_08](https://s2.51cto.com/images/blog/202409/08014501_66dc911d54ea88206.png?x-oss-process=image/watermark,size_16,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_30,g_se,x_10,y_10,shadow_20,type_ZmFuZ3poZW5naGVpdGk=/resize,m_fixed,w_1184)

扩展：

默认情况下不适用rm删除目录——rmdir只能删除空目录；

将需要删除的文件或目录移动到/tmp目录下即可（/tmp目录为临时目录，30天未被访问的文件会自动删除）；

如果必须删除一些文件，可通过find匹配出来后，再行删除。

整理的一些关于【IT人转架构设计】的项目学习资料+视频（附讲解～～），需要自取：

[https://d.51cto.com/bLN8S1](https://d.51cto.com/bLN8S1)
