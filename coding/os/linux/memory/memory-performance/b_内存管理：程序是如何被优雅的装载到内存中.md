
一口Linux _2021年11月18日 11:50_

The following article is from 假装懂编程 Author 康师傅

内存作为计算机中一项比较重要的资源，它的主要作用就是解决CPU和磁盘之间速度的鸿沟，但是由于内存条是需要插入到主板上的，因此对于一台计算机来说，由于物理限制，它的内存不可能无限大的。我们知道我们写的代码最终是要从磁盘被加载到内存中的，然后再被CPU执行，不知道你有没有想过，为什么一些大型游戏大到10几G，却可以在只有8G内存的电脑上运行？甚至在玩游戏期间，我们还可以聊微信、听音乐...，这么多进程看着同时在运行，它们在内存中是如何被管理的？带着这些疑问我们来看看计算系统内存管理那些事。

# 内存的交换技术

如果我们的内存可以无限大，那么我们担忧的问题就不会存在，但是实际情况是往往我们的机器上会同时运行多个进程，这些进程小到需要几十兆内存，大到可能需要上百兆内存，当许许多多这些进程想要同时加载到内存的时候是不可能的，但是从我们用户的角度来看，似乎这些进程确实都在运行呀，这是怎么回事？

这就引入要说的交换技术了，从字面的意思来看，我想你应该猜到了，它会把某个内存中的进程交换出去。当我们的进程空闲的时候，其他的进程又需要被运行，然而很不幸，此时没有足够的内存空间了，这时候怎么办呢？似乎刚刚那个空闲的进程有种占着茅坑不拉屎的感觉，于是可以把这个空闲的进程从内存中交换到磁盘上去，这时候就会空出多余的空间来让这个新的进程运行，当这个换出去的空闲进程又需要被运行的时候，那么它就会被再次交换进内存中。通过这种技术，可以让有限的内存空间运行更多的进程，进程之间不停来回交换，看着好像都可以运行。
![[Pasted image 20240928103709.png]]

如图所示，一开始进程A被换入内存中，所幸还剩余的内存空间比较多，然后进程B也被换入内存中，但是剩余的空间比较少了，这时候进程C想要被换入到内存中，但是发现空间不够了，这时候会把已经运行一段时间的进程A换到磁盘中去，然后调入进程C。

![[Pasted image 20240928103717.png]]

## 内存碎片

通过这种交换技术，交替的换入和换出进程可以达到小内存可以运行更多的进程，但是这似乎也产生了一些问题，不知道你发现了没有，在进程C换入进来之后，在进程B和进程C之间有段较小的内存空间，并且进程B之上也有段较小的内存空间，说实话，这些小空间可能永远没法装载对应大小的程序，那么它们就浪费了，在某些情况下，可能会产生更多这种内存碎片。
![[Pasted image 20240928103724.png]]
!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)如果想要节约内存，那么就得用到内存紧凑的技术了，即把所有的进程都向下移动，这样所有的碎片就会连接在一起变成一段更大的连续内存空间了。
!\[\[Pasted image 20240928103734.png\]\]
!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)但是这个移动的开销基本和当前内存中的活跃进程成正比，据统计，一台16G内存的计算机可以每8ns复制8个字节，它紧凑全部的内存大概需要16s，所以通常不会进行紧凑这个操作，因为它耗费的CPU时间还是比较大的。

## 动态增长

其实上面说的进程装载算是比较理想的了，正常来说，一个进程被创建或者被换入的时候，它占用多大的空间就分配多大的内存，但是如果我们的进程需要的空间是动态增长的，那就麻烦了，比如我们的程序在运行期间的for循环可能会利用到某个临时变量来存放目标数据（例如以下变量a，随着程序的运行是会越来越大的）：

`var a []int64   for i:= 0;i <= 1000000;i++{     if i%2 == 0{      a = append(a,i) //a是不断增大的     }   }   `

当需要增长的时候：

1. 如果进程的邻居是空闲区那还好，可以把该空闲区分配给进程

1. 如果进程的邻居是另一个进程，那么解决的办法只能把增长的进程移动到一个更大的空闲内存中，但是万一没有更大的内存空间，那么就要触发换出，把一个或者多个进程换出去来提供更多的内存空间，很明显这个开销不小。

为了解决进程空间动态增长的问题，我们可以提前多给一些空间，比如进程本身需要10M，我们多给2M，这样如果进程发生增长的时候，可以利用这2M空间，当然前提是这2M空间够用，如果不够用还是得触发同样的移动、换出逻辑。

![[Pasted image 20240928103744.png]]

# 空闲的内存如何管理

前面我们说到内存的交换技术，交换技术的目的是腾出空闲内存来，那么我们是如何知道一块内存是被使用了，还是空闲的？因此需要一套机制来区分出空闲内存和已使用内存，一般操作系统对内存管理的方式有两种：位图法和链表法。

## 位图法

先说位图法，没错，位图法采用比特位的方式来管理我们的内存，每块内存都有位置，我们用一个比特位来表示：

1. 如果某块内存被使用了，那么比特位为1

1. 如果某块内存是空闲的，那么比特位为0

这里的某块内存具体是多大得看操作系统是如何管理的，它可能是一个字节、几个字节甚至几千个字节，但是这些不是重点，重点是我们要知道内存被这样分割了。

![[Pasted image 20240928103754.png]]

位图法的优点就是清晰明确，某个内存块的状态可以通过位图快速的知道，因为它的时间复杂度是O(1)，当然它的缺点也很明显，就是需要占用太多的空间，尤其是管理的内存块越小的时候。更糟糕的是，进程分配的空间不一定是内存块的整数倍，那么最后一个内存块中一定是有浪费的。

![[Pasted image 20240928103804.png]]

如图，进程A和进程B都占用的最后一个内存块的一部分，那么对于最后一个内存块，它的另一部分一定是浪费的。

## 链表法

相比位图法，链表法对空间的利用更加合理，我相信你应该已经猜到了，链表法简单理解就是把使用的和空闲的内存用链表的方式连接起来，那么对于每个链表的元素节点来说，他应该具备以下特点：

1. 应该知道每个节点是空闲的还是被使用的

1. 每个节点都应该知道当前节点的内存的开始地址和结束地址

针对这些特点，最终内存对应的链表节点大概是这样的：

![[Pasted image 20240928103812.png]]

p代表这个节点对应的内存空间是被使用的，H代表这个节点对应的内存空间是空闲的，start代表这块内存空间的开始地址，length代表的是这块内存的长度，最后还有指向邻居节点的pre和next指针。

因此对于一个进程来说，它与邻居的组合有四种：

1. 它的前后节点都不是空闲的

1. 它的前一个节点是空闲的，它的后一个节点也不是空闲的

1. 它的前一个节点不是空闲的，它的后一个节点是空闲的

1. 它的前后节点都是空闲的

当一个内存节点被换出或者说进程结束后，那么它对应的内存就是空闲的，此时如果它的邻居也是空闲的，就会发生合并，即两块空闲的内存块合并成一个大的空闲内存块。

![[Pasted image 20240928103819.png]]

ok，通过链表的方式把我们的内存给管理起来了，接下来就是当创建一个进程或者从磁盘换入一个进程的时候，如何从链表中找到一块合适的内存空间？

##### 首次适应算法

其实想要找到空闲内存空间最简单的办法就是顺着链表找到第一个满足需要内存大小的节点，如果找到的第一个空闲内存块和我们需要的内存空间是一样大小的，那么就直接利用，但是这太理想了，现实情况大部分可能是找到的第一个目标内存块要比我们的需要的内存空间要大一些，这时候呢，会把这个空闲内存空间分成两块，一块正好使用，一块继续充当空闲内存块。

![[Pasted image 20240928103831.png]]

一个需要3M内存的进程，会把4M的空间拆分成3M和1M。

### 下次适配算法

和首次适应算法很相似，在找到目标内存块后，会记录下位置，这样下次需要再次查找内存块的时候，会从这个位置开始找，而不用从链表的头节点开始寻找，这个算法存在的问题就是，如果标记的位置之前有合适的内存块，那么就会被跳过。

![[Pasted image 20240928103839.png]]

一个需要2M内存的进程，在5这个位置找到了合适的空间，下次如果需要这1M的内存会从5这个位置开始，然后会在7这个位置找到合适的空间，但是会跳过1这个位置。

##### 最佳适配算法

相比首次适应算法，最佳适配算法的区别就是：不是找到第一个合适的内存块就停止，而是会继续向后找，并且每次都可能要检索到链表的尾部，因为它要找到最合适那个内存块，什么是最合适的内存块呢？如果刚好大小一致，则一定是最合适的，如果没有大小一致的，那么能容得下进程的那个最小的内存块就是最合适的，可以看出最佳适配算法的平均检索时间相对是要慢的，同时可能会造成很多小的碎片。

![[Pasted image 20240928103848.png]]

假设现在进程需要2M的内存，那么最佳适配算法会在检索到3号位置（3M）后，继续向后检索，最终会选择5号位置的空闲内存块。

##### 最差适配算法

我们知道最佳适配算法中最佳的意思是找到一个最贴近真实大小的空闲内存块，但是这会造成很多细小的碎片，这些细小的碎片一般情况下，如果没有进行内存紧凑，那么大概率是浪费的，为了避免这种情况，就出现了这个最差适配算法，这个算法它和最佳适配算法是反着来的，它每次尝试分配最大的可用空闲区，因为这样的话，理论上剩余的空闲区也是比较大的，内存碎片不会那么小，还能得到重复利用。

![[Pasted image 20240928103856.png]]

一个需要1.5M的进程，在最差适配算法情况下，不会选择3号（2M）内存空闲块，而是会选择更大的5号（3M）内存空闲块。

### 快速适配算法

上面的几种算法都有一个共同的特点：空闲内存块和已使用内存块是共用的一个链表，这会有什么问题呢？正常来说，我要查找一个空闲块，我并不需要检索已经被使用的内存块，所以如果能把已使用的和未使用的分开，然后用两个链表分别维护，那么上面的算法无论哪种，速度都将得到提升，并且节点也不需要P和M来标记状态了。但是分开也有缺点，如果进程终止或者被换出，那么对应的内存块需要从已使用的链表中删掉然后加入到未使用的链表中，这个开销是要稍微大点的。当然对于未使用的链表如果是排序的，那么首次适应算法和最佳适应算法是一样快的。

快速适配算法就是利用了这个特点，这个算法会为那些常用大小的空闲块维护单独的链表，比如有4K的空闲链表、8K的空闲链表...，如果要分配一个7K的内存空间，那么既可以选择两个4K的，也可以选择一个8K的。

![[Pasted image 20240928103903.png]]

它的优点很明显，在查找一个指定大小的空闲区会很快速，但是一个进程终止或被换出时，会寻找它的相邻块查看是否可以合并，这个过程相对较慢，如果不合并的话，那么同样也会产生很多的小空闲区，它们可能无法被利用，造成浪费。

# 虚拟内存：小内存运行大程序

可能你看到小内存运行大程序比较诧异，因为上面不是说到了吗？只要把空闲的进程换出去，把需要运行的进程再换进来不就行了吗？内存交换技术似乎解决了，这里需要注意的是，首先内存交换技术在空间不够的情况下需要把进程换出到磁盘上，然后从磁盘上换入新进程，看到磁盘你可能明白了，很慢。其次，你发现没，换入换出的是整个进程，我们知道进程也是由一块一块代码组成的，也就是许许多多的机器指令，对于内存交换技术来说，一个进程下的所有指令要么全部进内存，要么全部不进内存。看到这里你可能觉得这不是正常吗？好的，别急，我们接着往下看。

后来出现了更牛逼的技术：虚拟内存。它的基本思想就是，每个程序拥有自己的地址空间，尤其注意后面的自己的地址空间，然后这个空间可以被分割成多个块，每一个块我们称之为页（page）或者叫页面，对于这些页来说，它们的地址是连续的，同时它们的地址是虚拟的，并不是真正的物理内存地址，那怎么办？程序运行需要读到真正的物理内存地址，别跟我玩虚的，这就需要一套映射机制，然后MMU出现了，MMU全称叫做：Memory Managment Unit，即内存管理单元，正常来说，CPU读某个内存地址数据的时候，会把对应的地址发到内存总线上，但是在虚拟内存的情况下，直接发到内存总线上肯定是找不到对应的内存地址的，这时候CPU会把虚拟地址告诉MMU，让MMU帮我们找到对应的内存地址，没错，MMU就是一个地址转换的中转站。

![[Pasted image 20240928103915.png]]

程序地址分页的好处是：

1. 对于程序来说，不需要像内存交换那样把所有的指令都加载到内存中才能运行，可以单独运行某一页的指令

1. 当进程的某一页不在内存中的时候，CPU会在这个页加载到内存的过程中去执行其他的进程。

当然虚拟内存会分页，那么对应的物理内存其实也会分页，只不过物理内存对应的单元我们叫页框。页面和页框通常是一样大的。我们来看个例子，假设此时页面和页框的大小都是4K，那么对于64K的虚拟地址空间可以得到64/4=16个虚拟页面，而对于32K的物理地址空间可以得到32/4=8个页框，很明显此时的页框是不够的，总有些虚拟页面找不到对应的页框。

![[Pasted image 20240928103921.png]]

我们先来看看虚拟地址为20500对应物理地址如何被找到的：

1. 首先虚拟地址20500对应5号页面（20480-24575）

1. 5号页面的起始地址20480向后查找20个字节，就是虚拟地址的位置

1. 5号页面对应3号物理页框

1. 3号物理页框的起始地址是12288，12288+20=12308，即12308就是我们实际的目标物理地址。

但是对于虚拟地址而言，图中还有红色的区域，上面我们也说到了，总有些虚拟地址没有对应的页框，也就是这部分虚拟地址是没有对应的物理地址，当程序访问到一个未被映射的虚拟地址（红色区域）的时候，那么就会发生缺页中断，然后操作系统会找到一个最近很少使用的页框把它的内容换到磁盘上去，再把刚刚发生缺页中断的页面从磁盘读到刚刚回收的页框中去，最后修改虚拟地址到页框的映射，然后重启引起中断的指令。

最后可以发现分页机制使我们的程序更加细腻了，运行的粒度是页而不是整个进程，大大提高了效率。

### 页表

上面说到虚拟内存到物理内存有个映射，这个映射我们知道是MMU做的，但是它是如何实现的？最简单的办法就是需要有一张类似hash表的结构来查看，比如页面1对应的页框是10，那么就记录成`hash[1]=10`，但是这仅仅是定位到了页框，具体的位置还没定位到，也就是类似偏移量的数据没有。不猜了，我们直接来看看MMU是如何做到的，以一个16位的虚拟地址，并且页面和页框都是4K的情况来说，MMU会把前4位当作是索引，也就是定位到页框的序号，后12位作为偏移量，这里为什么是12位，很巧妙，因为2^12=4K，正好给每个页框里的数据上了个标号。因此我们只需要根据前4位找到对应的页框即可，然后偏移量就是后12位。找页框就是去我们即将要说的页表里去找，页表除了有页面对应的页框后，还有个标志位来表示对应的页面是否有映射到对应的页框，缺页中断就是根据这个标志位来的。

![[Pasted image 20240928103929.png]]

可以看出页表非常关键，不仅仅要知道页框、以及是否缺页，其实页表还有保护位、修改位、访问位和高速缓存禁止位。

- 保护位：指的是一个页允许什么类型的访问，常见的是用三个比特位分别表示读、写、执行。

- 修改位：有时候也称为脏位，由硬件自动设置，当一个页被修改后，也就是和磁盘的数据不一致了，那么这个位就会被标记为1，下次在页框置换的时候，需要把脏页刷回磁盘，如果这个页的标记为0，说明没有被修改，那么不需要刷回磁盘，直接把数据丢弃就行了。

- 访问位：当一个页面不论是发生读还是发生写，该页面的访问位都会设置成1，表示正在被访问，它的作用就是在发生缺页中断时，根据这个标志位优先在那些没有被访问的页面中选择淘汰其中的一个或者多个页框。

- 高速缓存禁止位：对于那些映射到设备寄存器而不是常规内存的页面而言，这个特性很重要，加入操作系统正在紧张的循环等待某个IO设备对它刚发出的指令做出响应，保证这个设备读的不是被高速缓存的副本非常重要。

![[Pasted image 20240928103936.png]]

## TLB快表加速访问

通过页表我们可以很好的实现虚拟地址到物理地址的转换，然而现代计算机至少是32位的虚拟地址，以4K为一页来说，那么对于32位的虚拟地址，它的页表项就有2^20=1048576个，无论是页表本身的大小还是检索速度，这个数字其实算是有点大了。如果是64位虚拟的地址，按照这种方式的话，页表项将大到超乎想象，更何况最重要的是每个进程都会有一个这样的页表。

我们知道如果每次都要在庞大的页表里面检索页框的话，效率一定不是很高。而且计算机的设计者们观察到这样一种现象：大多数程序总是对少量的页进行多次访问，如果能为这些经常被访问的页单独建立一个查询页表，那么速度就会大大提升，这就是快表，快表只会包含少量的页表项，通常不会超过256个，当我们要查找一个虚拟地址的时候。首先会在快表中查找，如果能找到那么就可以直接返回对应的页框，如果找不到才会去页表中查找，然后从快表中淘汰一个表项，用新找到的页替代它。

总体来说，TLB类似一个体积更小的页表缓存，它存放的都是最近被访问的页，而不是所有的页。

# 多级页表

TLB虽然一定程度上可以解决转换速度的问题，但是没有解决页表本身占用太大空间的问题。其实我们可以想想，大部分程序会使用到所有的页面吗？其实不会。一个进程在内存中的地址空间一般分为程序段、数据段和堆栈段，堆栈段在内存的结构上是从高地址向低地址增长的，其他两个是从低地址向高地址增长的。

![[Pasted image 20240928103943.png]]

可以发现中间部分是空的，也就是这部分地址是用不到的，那我们完全不需要把中间没有被使用的内存地址也引入页表呀，这就是多级页表的思想。以32位地址为例，后12位是偏移量，前20位可以拆成两个10位，我们暂且叫做顶级页表和二级页表，每10位可以表示2^10=1024个表项，因此它的结构大致如下：

![[Pasted image 20240928103951.png]]

对于顶级页表来说，中间灰色的部分就是没有被使用的内存空间。顶级页表就像我们身份证号前面几个数字，可以定位到我们是哪个城市或者县的，二级页表就像身份证中间的数字，可以定位到我们是哪个街道或者哪个村的，最后的偏移量就像我们的门牌号和姓名，通过这样的分段可以大大减少空间，我们来看个简单的例子：

如果我们不拆出顶级页表和二级页表，那么所需要的页表项就是2^20个，如果我们拆分，那么就是1个顶级页表+2^10个二级页表，两者的存储差距明显可以看出拆分后更加节省空间，这就是多级页表的好处。

当然我们的二级也可以拆成三级、四级甚至更多级，级数越多灵活性越大，但是级数越多，检索越慢，这一点是需要注意的。

end



---

**一口Linux**

《从零开始学ARM》作者，一起学习嵌入式，Linux，网络，驱动，arm知识。

300篇原创内容

公众号

**精彩文章合集**

文章推荐

☞【专辑】[ARM](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=MzUxMjEyNDgyNw==&action=getalbum&album_id=1614665559315382276#wechat_redirect)

☞【专辑】[粉丝问答](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=MzUxMjEyNDgyNw==&action=getalbum&album_id=1629876820810465283#wechat_redirect)

☞【专辑】[所有原创](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=MzUxMjEyNDgyNw==&action=getalbum&album_id=1479949091139813387#wechat_redirect)

☞【专辑】[linux](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=MzUxMjEyNDgyNw==&action=getalbum&album_id=1507350615537025026#wechat_redirect)入门

☞【专辑】[计算机网络](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=MzUxMjEyNDgyNw==&action=getalbum&album_id=1598710257097179137#wechat_redirect)

☞【专辑】[Linux驱动](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=MzUxMjEyNDgyNw==&action=getalbum&album_id=1502410824114569216#wechat_redirect)

☞【干货】[嵌入式驱动工程师学习路线](http://mp.weixin.qq.com/s?__biz=MzUxMjEyNDgyNw==&mid=2247496985&idx=1&sn=c3d5e8406ff328be92d3ef4814108cd0&chksm=f96b87edce1c0efb6f60a6a0088c714087e4a908db1938c44251cdd5175462160e26d50baf24&scene=21#wechat_redirect)

☞【干货】[Linux嵌入式所有知识点-思维导图](http://mp.weixin.qq.com/s?__biz=MzUxMjEyNDgyNw==&mid=2247497822&idx=1&sn=1e2aed9294f95ae43b1ad057c2262980&chksm=f96b8aaace1c03bc2c9b0c3a94c023062f15e9ccdea20cd76fd38967b8f2eaad4dfd28e1ca3d&scene=21#wechat_redirect)

点击“**阅读原文**”查看更多分享，欢迎**点分享、收藏、点赞、在看**

Reads 1200

​

Comment

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/icRxcMBeJfc8535w2vKlsLPf5hwdMjpYrzuVCHx0rcQmvv8rYqTFtIyic5qErtciaibqaIOWgeKkDsOMeae4HciaUaw/300?wx_fmt=png&wxfrom=18)

一口Linux

11Share3

Comment

Comment

**Comment**

暂无留言
