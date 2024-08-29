
Linux爱好者

 _2021年11月29日 11:53_

以下文章来源于开发内功修炼 ，作者张彦飞allen


![](http://wx.qlogo.cn/mmhead/Q3auHgzwzM5PSTGibsjHYiaRMmoGJxNMm8VFPdHQgvCVyBuEibNnqLMwA/0)

**开发内功修炼**.

飞哥有鹅厂、搜狗 10 年多的开发工作经验。通过本号，我把多年中对于性能的一些深度思考分享给大家。

](https://mp.weixin.qq.com/s?__biz=MzAxODI5ODMwOA==&mid=2666559427&idx=2&sn=4706c45d851374d6f88c16803778d087&chksm=80dcb368b7ab3a7ec99bd596a987abf2e35b00b2bb09e0ad9044c13a1e59ffe3d5b400a3ac10&mpshare=1&scene=24&srcid=1130N2hXPhp1Hg35lmwEAiyq&sharer_sharetime=1638228644953&sharer_shareid=5fb9813bfe9ffc983435bfc8d8c5e9ca&key=daf9bdc5abc4e8d0b78ab033cf3ccdc4b03d84c63caacf55a9bce4068a6acbe553ac377f96527f2c72059526ae2cb8e6abaed3b4d78dd5146301d69ea1edda43c5882e717256cffcc7b85cd4c102f4e09fc8ad1a5c4dbbd572ee12a25ffc9aa25f75a99257337189bdca6a49068c6b264d47331e3b92961facdfe6d2e71e9beb&ascene=0&uin=MTEwNTU1MjgwMw%3D%3D&devicetype=Windows+11+x64&version=63090b19&lang=zh_CN&countrycode=CN&exportkey=n_ChQIAhIQrzYjgdHLU6IbdMTCbMB1ZxLmAQIE97dBBAEAAAAAAMb1LT8gbdEAAAAOpnltbLcz9gKNyK89dVj0K%2FYK4plQjnkTF8wKJwCgqJatdl6%2BSflWnUHuDjLFJXoWsw9C%2Bah4i%2FNUs74cRLEhkHc8ZR5ugpirkidppjaZFnMfLpIb6oRRCHId3wVx5t%2FOLzLBXdZCpQd%2Fp0RvVuzbp6BkYzJPUEzax8q%2B0xQfNbmSioaAzvfjwuMT6ALJ6vE09oVclU6Bczs4uN4CMtDvG%2B%2BrQ%2FMFJ0XnQgd98pauHSc5ZI5g0Rzr%2BdTVTdZkvnQc0VIz9YKk0i6aYPJ2xUlN&acctmode=0&pass_ticket=%2BxIPvX50XDOfnpGBwnoW9uYCQj9wmrDETIB1nUhEx%2BuJsuLI7TzVvRcG3go%2B1dJm&wx_header=1&fasttmpl_type=0&fasttmpl_fullversion=7350504-zh_CN-zip&fasttmpl_flag=1#)

关于文件系统，相信大家都不陌生。身为攻城狮的我们几乎天天都会与之打交道，但是细深剖一下，其中又有多少是我们理解深度不够的呢。那么让我们一起来看一下下面这一组 Linux 文件系统相关的问题吧：

1、机械磁盘随机读写时速度非常慢，操作系统是采用什么技巧来提高随机读写的性能的？  
2、touch 一个新的空文件占用磁盘空间吗？占用的话占用多少？  
3、新建一个空目录占用磁盘空间吗？占用多少？和新建一个文件相比，哪个占用的更大？  
4、你知道文件名是记录在磁盘的什么地方吗？  
5、文件名最长多长？受什么制约？  
6、文件名太长了会影响系统性能吗？为什么会产生影响？  
7、一个目录下最多能建立多少个文件？  
8、新建一个内容大小 1 k 的文件，实际会占用多大的磁盘空间？  
9、向操作系统发起读取文件 2 Byte 的命令，操作系统实际会读取多少呢？  
10、我们使用文件时要怎么样来能提高磁盘IO速度？

如果你能想也不用想的就回答上来百分八十的问题，那么请关掉本篇文章吧。如果不能，而且你也像作者一样对有窥探操作系统隐私的嗜好，那么就请随我一起来探索文件系统的这些有趣的地方，相信理解了这些之后对我们手中的工作会有很大的帮助。这篇文章实验所用文件系统是 ext 系的。

## 一、磁盘构成及分区

### 1、磁盘物理结构

还是先从最基本的磁盘物理结构说起吧，注意本文只讨论机械磁盘，SSD 不在本文讨论范围之内。我们人类管理任何事物总是习惯先划分出一定的结构，在此规则的基础上进行管理。军队分军、师、旅、团和营。公司分事业群、部门、中心和小组。然后对于管理磁盘，分磁盘面、磁头、磁道、柱面和扇区。

- 磁盘面：磁盘是由一叠磁盘面组成，见图。
    
- 磁头(Heads)：每个磁头对应一个磁盘面，负责该磁盘面上的数据的读写。。
    
- 磁道(Track)：每个盘面会围绕圆心划分出多个同心圆圈，每个圆圈叫做一个磁道。
    
- 柱面(Cylinders)：所有盘片上的同一位置的磁道组成的立体叫做一个柱面。
    
- 扇区(Sector)：以磁道为单位管理磁盘仍然太大，所以计算机前辈们又把每个磁道划分出了多个扇区，见下右图
    

![图片](https://mmbiz.qpic.cn/mmbiz_png/BBjAFF4hcwo2yKthDibQx0hPyQNmD6fc5Nmv5fUvsKA6CQg15C2alR97XtMCGAT6FiayewBK6AWDFtehBUrO59Iw/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

  
本人爱上 Linux 的一个原因就是只要你愿意下功夫，你就能把 Linux 的内部逻辑彻底铺开来看，这点比 Windows 要好太多了。Linux 上可以通过 fdisk 命令，来查看当前系统使用的磁盘的这些物理信息。  

![图片](https://mmbiz.qpic.cn/mmbiz_png/BBjAFF4hcwo2yKthDibQx0hPyQNmD6fc5gCUdNiczJNSVYicjiaVQ0t8M5MmUeMY0SyAIbhP3IFxeAm2D3M4pOnjWw/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

以上是我本人的一台虚拟机的磁盘物理信息。可以看出我的磁盘有 255 个 heads，也就是说共有 255 个盘面。3263 个 cylinders，也就是说每个盘面上都有 3263 个磁道， 63 sectors/track 说的是每个磁道上共有 63 个扇区。命令结果也给出了 Sector size 的值是 512 bytes。那我们动笔算一下该磁盘的大小吧。  

255 盘面  * 3263 柱面 * 63 扇区 * 每个扇区 512 bytes = 26839088640 byte。

结果是 26.8 G,和磁盘的总大小基本相符（至于fdisk给出的详细结果相差了约4M的大小，笔者也没有弄彻底明白，有兴趣的读者可以继续研究）。

不过要注意一点就是上面的盘面等数据是逻辑上的，是物理盘面映射转化而来的，这个转化关系我现在还没搜到特别好的资料。

### 2、分区

分区是操作系统对磁盘进行管理的第一步，这也是我们任何一个计算机使用者都非常熟悉的概念。例如Windows下的C、D、E、F盘。那么请思考一下，

**思考**：前面的磁盘的详细物理结构已经有了，如果让你把整块磁盘分成C、D等分区，你会怎么分呢？

- 方案一：255 个盘面，C 盘是 0-100 盘面， D 盘是 101-200 个盘面, ……
    
- 方案二：3263 个柱面，C 盘 0-1000 个柱面，D 盘 1001-20001 个柱面, ……
    

对于以上的两个方案，你会选择哪一种呢？？先说下磁盘 IO 时的过程。

- 第一步，首先是磁头径向移动来寻找数据所在的磁道。这部分时间叫寻道时间。
    
- 第二步，找到目标磁道后通过盘面旋转，将目标扇区移动到磁头的正下方。
    
- 第三步，向目标扇区读取或者写入数据。到此为止，一次磁盘 IO 完成。
    

**故单次磁盘 IO 时间** = 寻道时间 + 旋转延迟 + 存取时间。

对于**旋转延时**，现在主流服务器上经常使用的是1W转/分钟的磁盘，每旋转一周所需的时间为60*1000/10000=6ms，故其旋转延迟为（0-6ms）。对于存取时间，一般耗时较短，为零点几 ms。对于寻道时间，现代磁盘大概在 3-15 ms，其中寻道时间大小主要受磁头当前所在位置和目标磁道所在位置相对距离的影响。

  
其实采用哪一种，最主要看的是那种方式性能更快。因为同一分区下的数据经常会一起读取，假如采用第一种，那么这样磁头就需要在 3000 多个 track 间不停地跳来跳去，这样磁盘的寻道时间就会翻倍，磁盘性能就会下降。

而对于方案二，假如对于磁盘C，只需要在磁头在 1-1000 个磁道间移动就可以了，大大降低了寻道时间。（实际上分区并不是从 0 开始的，磁盘的第一个磁道对应的柱面会被用来安装引导加载程序以及磁盘分区表）。所以，方案二的分区方式可以降低磁盘 IO 时间中的寻道时间部分，所以所有的操作系统采用的都是方案二，没有用方案一的。

在Linux下使用过fdisk进行分区的话可以注意到以下信息。

![图片](https://mmbiz.qpic.cn/mmbiz_png/BBjAFF4hcwo2yKthDibQx0hPyQNmD6fc5JzmykrloxWhhIPOWGQXAFaZjUialIOEkoXBZEPyAVj0p9bjicwPu5ngg/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

这充分证明了操作系统是采用方案二的。  

回到开篇问题 1，操作系统是采用什么技巧来降低随机读写的性能问题的呢？操作系统通过按磁道对应的柱面划分分区，来降低磁盘 IO 所花费的的寻道时间 ，进而提高磁盘的读写性能。

## 二、目录与文件

### 1、引子

好了，磁盘基础都说完了，那我们正式进入主题，开始我们 Linux 文件系统相关的讨论吧。文件系统不就是目录和文件吗？这二位可是我们熟悉的不能再熟悉的家伙了。可你确认它不是你的那位熟悉的陌生人么？我先来来创建个空目录和空文件吧，查看结果如下图：

![图片](https://mmbiz.qpic.cn/mmbiz_png/BBjAFF4hcwo2yKthDibQx0hPyQNmD6fc5QrBJIjHNglWyB26fHlfiaCibibC3oEz6YgvWWiawQwtbX2m9ckGIFsraqA/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

我们都知道第五列显示的是占用的空间大小，那么我来提个几个小小的问题吧。  

- 1）为什么目录占用的空间是 4096？
    
- 2）为什么空文件占用的空间却是 0？
    
- 3）如果空文件真占用 0 byte 空间，那么该文件的文件名、创建者以及权限-rw-rw-r—等文件夹相关的信息都存到哪儿去了？
    

### 2、我就不信空文件不占用空间

为了解开这个谜底，需要借助 df 命令。输入 df –i，

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

Linux 结果中红框位置处显示的是 inodes 的相关信息，如果你对 inode 的概念不熟悉，你可以暂时把它当成一个操作系统秘密管理的一个家伙，会占用空间就行了。接下来我 touch 一个空的文件后再次 df -i。  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

虽然前面操作系统告诉我们，一个新建的空文件占用的空间是 0。但是这个实验却证明操作系统“欺骗”了我们，它消耗掉了一个 inode。那么 inode 的节点大小是多少呢，使用 dumpe2fs 命令可以帮助我们查看到这个东东的实际大小。在输出的结果中我们可以找到下面这行。  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

它告诉我们每个 inode 的大小是 256 Byte。当然这个大小每台机器都会不一样，它实际上是在系统格式化磁盘的时候决定的。  

好了，开篇第二个问题也有答案了。原来新建一个空的文件是会占用磁盘空间的，实际占用的是 256 Byte。哦，不，准确的说法应该是一个 inode size，具体的值是在格式化时决定的。

再说说新建空目录吧，前面说了新建空目录会占用4KB的磁盘空间。那么仅仅如此吗？我们同样在新建目录前后都使用df –i来监视系统 inode 的占用。

原来目录也是会占用一个 inode 节点的，第三个问题也有了答案了，新建一个空目录会占用磁盘空间 4KB + inode size。哦，这个在你的系统上也不一定是4K，它实际上一个 block size。同样在 dumpe2fs 下可以看到。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

只不过我的磁盘在格式化时采用的是 4KB 的大小，呵呵！  

### 3、神秘的空目录的4KB

前面的谜团解开了，可以作为攻城狮的我对另外一个东西产生了好奇心。就是空目录占用的那 4KB，这些空间是用来存什么的呢？好神秘呀。cd 到我们新建的目录下查看。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

我们再新建两个空的文件，再查看下目录的空间占用情况。  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

貌似，没有什么新发现。因为空文件不占用 block，所以这里显示的仍然是目录占用的 block，和之前大小没有变化。那么我继续使用 php 脚本创建 100 个文件名长度为 32Byte 的空文件。  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

这时我们发现目录占用的磁盘空间变大了，成了 3 个 Block 了。哈哈，这就解答了我们开篇的第四个问题，文件名是存在目录占用的 block 中的。接下来我又还证明了每个目录 block 中能保存的文件名个数是和文件名的长度有关的（好像有点废话的意思，不过亲手证明自己的猜想还是有点小爽的）。我又另外新建了个空目录，创建了 100 个文件名长度为 32*3 个空文件，该临时目录占用的磁盘空间如下：  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

你可能会问我为什么文件名变成了 3 倍后，占用的 block 数目为什么没有变成 3 倍。其实Linux文件系统关于文件的结构体中除了文件名以外，还有其它的一些字段的，文件名变长3倍不会导致结构体变大 3 倍的，这点可以参考 Linux 系统内核相关书籍。  

好了，到现在开篇问题 6 也有了答案了。文件名长了当然会对系统性能产生影响，因为这可能会导致更多的磁盘 IO。很多程序员都喜欢将文件命名为有意义的长串，使人一看文件名就知道用途。当然我没说这样不好，但是如果你的文件数量相当大的时候，你就要考虑你的文件名是否导致你的目录 block 占用太多了。

占用的空间倒是小事，磁盘很便宜，但是你得考虑下在目录下查找文件时操作系统的感受，操作系统可需要用你你提供的文件名进行字符串比较，而且运气不好的话需要将其名下所有 block 都搞一遍才行啊。（当然了，你的文件名长度不变态，而且数量没有达到十万数量级的话实际上这个开销也不会太大，但是这个开销你还是知道的为好）

至于开篇问题 5，文件名最长多长。实际上Linux操作系统就是为了避免程序员不节制地使用长文件名，强加了个限制，不得超过 255 byte。

另外，大家有没有经验，在目录下文件很多的时候，我们使用ls命令时会很慢。现在大家知道原因了吧，这时实际上操作系统在读取当前目录的所有 block ，如果 block 比较多的话，可能得需要多次 IO 操作才能完成这个简单的 ls 命令。

我在自己的电脑某个目录下创建了一 100W 个空文件，ls 命令 1 分钟还没出结果，被我 ctrl+c 掉了。在自己的项目中可不要这么干，虽然操作系统可以 cache 住你的目录数据，使你下次调用时会快很多，但我还是建议你单个目录下文件数目不要过万。否则你的程序在重启后首次运行时可能会出现性能不佳的情况。

好了，回到开篇问题 7，你有答案了吗？一个目录下最多能建多少个文件，这个最多其实是受限于你目录所在分区的 inode 数量，你有 100W 个 inode，你最多就可以新建 100W 个文件。但是，上面说了，单个目录下文件数量最好不要过万，否则会带来系统性能的问题。

### 4、文件的block

再做个关于文件的实验。我新建了个空目录，并在其下新建了个文件，里面只写了一个空格数据，保存后 du 命令显示如下：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

这 8K 里有 4K 是目录的，也就可以算出操作系统为只包含一个空格的文件分配了 4KB。其实文件的 block 比较简单的了，不像目录的 block 里会存很多文件系统的结构体，文件的 block 里只会保存文件的数据。上面这个实验表明，操作系统分配空间时是以 block 为最小单位。

也就是说只要你的文件数据不为空，操作系统就至少会给你分配一个 block 来存储，直到你超过了 4KB，操作系统再给你分配下一个 block，就是这样。所以对于开篇问题 8，新建一个内容大小为 1k 的文件，实际会占用 1个 block（一般为4k）和一个 inode（一般为256byte）。  

其实文件系统在向磁盘发起 IO 请求的时候，也是以 block size 为单位的。哪怕你只向操作系统发起读取文件的 2 Byte，但是操作系统会一次性给你读取 4KB 回来。因此磁盘 IO 真的是很慢，而且我们只要访问了这 2 Byte，确实很有可能接下来继续访问这 2byte 后面的内容，这也就是程序局部性原理，所以操作系统索性一次性就多读取些回来了。呵呵，这就是开篇问题9的答案。

这就像我们去逛超市，逛一次真的是很浪费时间，这可要比坑爹的磁盘 IO 也慢许多了。我们总不会逛了一圈超市就买了一个苹果就回来了吧，我们肯定会多买些东西为家里以后的需求准备着，反正买一堆东西比买一个苹果也没多花多少时间，何乐为不为呢，就是这个道理。

再说说开篇问题 10，我们攻城狮怎么样设计你的文件能提高一些 IO 速度呢？那就是如果你知道你的要新建的文件大概会占用多大的空间的话，比如 1M。那么你新建文件时就顺便和操作系统说一下，让它帮你将文件的 size 预留下来。这样实际上操作系统时会尽可能为你分配连续的 block，这样你再读取这个文件时，磁头就省去很多寻道时间了，IO 速度就显得快多了。

## 三、写在后面的话

前面我们说的都是基于我自己的文件系统，情形是一个 block size 是 4KB，一个 inode size 是 256byte，包括我虚拟机上的 inode 数量才只有 140 多万个。这些值实际上不是固定的，你完全可以在格式化你的硬盘的时候设置成其它的值。设置的原则就是看你的硬盘的容量，以及你的用途。

如果你的文件都是大于 4KB，甚至是几 M，几 G 的文件，那么建议你的 block 还是尽可能的大一点吧，这样 inode 里就能少记几个地址。

如果你的文件大部分都是1K以下的，那么确实使用4K的block会造成一点点浪费，如果你的老板对成本要求异常苛刻的话，你可以适当考虑把你的 block 设置得小一点。

另外，要关注你的文件系统的 inode。操作系统在查看目录和文件占用的磁盘空间信息时把inode节点的占用给隐藏起来了，其用意在于为用户提供一个白盒的环境，把数据占用的空间交给我们来认知，而把 inode 信息隐藏起来为了降低我们理解操作系统的难度。

而实际上，我们作为非普通用户的开发人员应该具备这个知情权。这个东东直接关系到你文件系统能创建文件数量。否则哪天等你发现线上机器磁盘还剩大把大把的空间，但就是 inode 使用光了，那时候就只有重新格式化或者迁移服务器了。这两个操作想想都觉得苦逼啊，还是能避免就尽量避免吧。

**思考题**：我们大家有个经验就是目录下小文件太多的情况下，往其它地方拷贝的话，速度会非常的慢，我们这时往往会把目录压缩一下再拷贝。现在你能说出这样做为什么会快吗？

最后我想再多说一句，这篇九年前写的文章现如今看起来还是很有实用价值。

- EOF -

推荐阅读  点击标题可跳转

1、[10 分钟看懂 Docker 和 K8S](http://mp.weixin.qq.com/s?__biz=MzAxODI5ODMwOA==&mid=2666558962&idx=1&sn=ede9eea879107df21557e1b9dcfeeb28&chksm=80dcb159b7ab384f22e68fc32d5556afd3e98c29d08757a871582d526bf5985b0c7c3fd3ec6d&scene=21#wechat_redirect)[](http://mp.weixin.qq.com/s?__biz=MzAxODI5ODMwOA==&mid=2666558962&idx=1&sn=ede9eea879107df21557e1b9dcfeeb28&chksm=80dcb159b7ab384f22e68fc32d5556afd3e98c29d08757a871582d526bf5985b0c7c3fd3ec6d&scene=21#wechat_redirect)

2、[这才是中国被卡脖子最严重的软件！](http://mp.weixin.qq.com/s?__biz=MzAxODI5ODMwOA==&mid=2666558647&idx=1&sn=f9f02d09c89ec081c61a717b3724b7fd&chksm=80dcb01cb7ab390a0d89664294542d564e7cc4d539cc6dc30ef997158d1b1ac496a21a991407&scene=21#wechat_redirect)

3、[如果让你来设计网络，你会把它弄成啥样？](http://mp.weixin.qq.com/s?__biz=MzAxODI5ODMwOA==&mid=2666558954&idx=2&sn=775617c1787bb8ab5251444c8762cd44&chksm=80dcb141b7ab38576e44e8149b6f64aab5812d557f7f50a6b8543b375c7a67acac0fe75f00ac&scene=21#wechat_redirect)

  

看完本文有收获？请分享给更多人  

推荐关注「Linux 爱好者」，提升Linux技能

![](http://mmbiz.qpic.cn/mmbiz_png/9aPYe0E1fb3sjicd8JxDra10FRIqT54Zke2sfhibTDdtdnVhv5Qh3wLHZmKPjiaD7piahMAzIH6Cnltd1Nco17Ihjw/300?wx_fmt=png&wxfrom=19)

**Linux爱好者**

点击获取《每天一个Linux命令》系列和精选Linux技术资源。「Linux爱好者」日常分享 Linux/Unix 相关内容，包括：工具资源、使用技巧、课程书籍等。

75篇原创内容

公众号

点赞和在看就是最大的支持❤️

阅读 2962

​

写留言

**留言 2**

- ![[爱心]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
    2021年11月29日
    
    赞2
    
    4tb的硬盘用来储存数据 该怎么分区呢 可不可以不分区
    
- 西红柿🍉蛋汤🍊
    
    2021年11月29日
    
    赞
    
    压缩后少寻地址
    

已无更多数据

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/9aPYe0E1fb3sjicd8JxDra10FRIqT54Zke2sfhibTDdtdnVhv5Qh3wLHZmKPjiaD7piahMAzIH6Cnltd1Nco17Ihjw/300?wx_fmt=png&wxfrom=18)

Linux爱好者

9分享2

2

写留言

**留言 2**

- ![[爱心]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
    2021年11月29日
    
    赞2
    
    4tb的硬盘用来储存数据 该怎么分区呢 可不可以不分区
    
- 西红柿🍉蛋汤🍊
    
    2021年11月29日
    
    赞
    
    压缩后少寻地址
    

已无更多数据