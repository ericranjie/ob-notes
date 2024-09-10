# 

原创 小林coding 小林coding

 _2021年12月17日 17:38_

大家好，我是小林。

「为什么 MySQL 采用 B+ 树作为索引？」这句话，是不是在面试时经常出现。

要解释这个问题，其实不单单要从数据结构的角度出发，还要考虑磁盘 I/O 操作次数，因为 MySQL 的数据是存储在磁盘中的嘛。

这次，就跟大家一层一层的分析这个问题，图中包含大量的动图来帮助大家理解，相信看完你就拿捏这道题目了！

![图片](https://mmbiz.qpic.cn/mmbiz_png/J0g14CUwaZcz9O5KfqlqMpm7icDcyaekMB26Bias6DiaeL66Wt7mwpQE2cQAkLfbFHOtMgrSaa4NYibkhGnQUGLMlg/640?wx_fmt=png&wxfrom=13&tp=wxpic)

### 怎样的索引的数据结构是好的？  

MySQL 的数据是持久化的，意味着数据（索引+记录）是保存到磁盘上的，因为这样即使设备断电了，数据也不会丢失。

磁盘是一个慢的离谱的存储设备，有多离谱呢？

人家内存的访问速度是纳秒级别的，而磁盘访问的速度是毫秒级别的，也就是说读取同样大小的数据，磁盘中读取的速度比从内存中读取的速度要慢上万倍，甚至几十万倍。

磁盘读写的最小单位是**扇区**，扇区的大小只有 `512B` 大小，操作系统一次会读写多个扇区，所以**操作系统的最小读写单位是块（Block）。Linux 中的块大小为 `4KB`**，也就是一次磁盘  I/O 操作会直接读写 8 个扇区。

由于数据库的索引是保存到磁盘上的，因此当我们通过索引查找某行数据的时候，就需要先从磁盘读取索引到内存，再通过索引从磁盘中找到某行数据，然后读入到内存，也就是说查询过程中会发生多次磁盘 I/O，而磁盘 I/O 次数越多，所消耗的时间也就越大。

所以，我们希望索引的数据结构能在尽可能少的磁盘的 I/O 操作中完成查询工作，因为磁盘  I/O 操作越少，所消耗的时间也就越小。

另外，MySQL 是支持范围查找的，所以索引的数据结构不仅要能高效地查询某一个记录，而且也要能高效地执行范围查找。

所以，要设计一个适合 MySQL 索引的数据结构，至少满足以下要求：

- 能在尽可能少的磁盘的 I/O 操作中完成查询工作；
    
- 要能高效地查询某一个记录，也要能高效地执行范围查找；
    

分析完要求后，我们针对每一个数据结构分析一下。

### 什么是二分查找？

索引数据最好能按顺序排列，这样可以使用「二分查找法」高效定位数据。

假设我们现在用数组来存储索引，比如下面有一个排序的数组，如果要从中找出数字 3，最简单办法就是从头依次遍历查询，这种方法的时间复杂度是 O(n)，查询效率并不高。因为该数组是有序的，所以我们可以采用二分查找法，比如下面这张采用二分法的查询过程图：

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/J0g14CUwaZdmUOEoN9jh0JZndxiaQHXaAEUoGsg3kYwugMwtGdeDtoo8X0ibJZd3GbQ6f2vUk4IH4TFnLQkiaHMWQ/640?wx_fmt=jpeg&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

可以看到，二分查找法每次都把查询的范围减半，这样时间复杂度就降到了 O(logn)，但是每次查找都需要不断计算中间位置。

### 什么是二分查找树？

用数组来实现线性排序的数据虽然简单好用，但是插入新元素的时候性能太低。

因为插入一个元素，需要将这个元素之后的所有元素后移一位，如果这个操作发生在磁盘中呢？这必然是灾难性的。因为磁盘的速度比内存慢几十万倍，所以我们不能用一种线性结构将磁盘排序。

其次，有序的数组在使用二分查找的时候，每次查找都要不断计算中间的位置。

那我们能不能设计一个非线形且天然适合二分查找的数据结构呢？

有的，请看下图这个神奇的操作，找到所有二分查找中用到的所有中间节点，把他们用指针连起来，并将最中间的节点作为根节点。

![图片](https://mmbiz.qpic.cn/mmbiz_gif/J0g14CUwaZdmUOEoN9jh0JZndxiaQHXaAcmyXtk6NNI5PAXyGnBh1Is7srJqjf1sOAW6jmHC7ZbDZfRaHiaMm9Ng/640?wx_fmt=gif&tp=wxpic&wxfrom=5&wx_lazy=1)

怎么样？是不是变成了二叉树，不过它不是普通的二叉树，它是一个**二叉查找树**。

**二叉查找树的特点是一个节点的左子树的所有节点都小于这个节点，右子树的所有节点都大于这个节点**，这样我们在查询数据时，不需要计算中间节点的位置了，只需将查找的数据与节点的数据进行比较。

假设，我们查找索引值为 key 的节点：

1. 如果 key 大于根节点，则在右子树中进行查找；
    
2. 如果 key 小于根节点，则在左子树中进行查找；
    
3. 如果 key 等于根节点，也就是找到了这个节点，返回根节点即可。
    

二叉查找树查找某个节点的动图演示如下，比如要查找节点 3 ：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

另外，二叉查找树解决了插入新节点的问题，因为二叉查找树是一个跳跃结构，不必连续排列。这样在插入的时候，新节点可以放在任何位置，不会像线性结构那样插入一个元素，所有元素都需要向后排列。

下面是二叉查找树插入某个节点的动图演示：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

因此，二叉查找树解决了连续结构插入新元素开销很大的问题，同时又保持着天然的二分结构。

那是不是二叉查找树就可以作为索引的数据结构了呢？

不行不行，二叉查找树存在一个极端情况，会导致它变成一个瘸子！

**当每次插入的元素都是二叉查找树中最大的元素，二叉查找树就会退化成了一条链表，查找数据的时间复杂度变成了 O(n)**，如下动图演示：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

由于树是存储在磁盘中的，访问每个节点，都对应一次磁盘 I/O 操作（_假设一个节点的大小「小于」操作系统的最小读写单位块的大小_），也就是说**树的高度就等于每次查询数据时磁盘 IO 操作的次数**，所以树的高度越高，就会影响查询性能。

二叉查找树由于存在退化成链表的可能性，会使得查询操作的时间复杂度从 O(logn)降低为 O(n)。

而且会随着插入的元素越多，树的高度也变高，意味着需要磁盘 IO 操作的次数就越多，这样导致查询性能严重下降，再加上不能范围查询，所以不适合作为数据库的索引结构。

### 什么是自平衡二叉树？

为了解决二叉查找树会在极端情况下退化成链表的问题，后面就有人提出**平衡二叉查找树（AVL 树）**。

主要是在二叉查找树的基础上增加了一些条件约束：**每个节点的左子树和右子树的高度差不能超过 1**。也就是说节点的左子树和右子树仍然为平衡二叉树，这样查询操作的时间复杂度就会一直维持在 O(logn) 。

下图是每次插入的元素都是平衡二叉查找树中最大的元素，可以看到，它会维持自平衡：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

除了平衡二叉查找树，还有很多自平衡的二叉树，比如红黑树，它也是通过一些约束条件来达到自平衡，不过红黑树的约束条件比较复杂，不是本篇的重点重点，大家可以看《数据结构》相关的书籍来了解红黑树的约束条件。

下面是红黑树插入节点的过程，这左旋右旋的操作，就是为了自平衡。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**不管平衡二叉查找树还是红黑树，都会随着插入的元素增多，而导致树的高度变高，这就意味着磁盘 I/O 操作次数多，会影响整体数据查询的效率**。

比如，下面这个平衡二叉查找树的高度为 5，那么在访问最底部的节点时，就需要磁盘 5 次 I/O 操作。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

根本原因是因为它们都是二叉树，也就是每个节点只能保存 2 个子节点 ，如果我们把二叉树改成 M 叉树（M>2）呢？

比如，当 M=3 时，在同样的节点个数情况下，三叉树比二叉树的树高要矮。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

因此，**当树的节点越多的时候，并且树的分叉数 M 越大的时候，M 叉树的高度会远小于二叉树的高度**。

### 什么是 B 树

自平衡二叉树虽然能保持查询操作的时间复杂度在O(logn)，但是因为它本质上是一个二叉树，每个节点只能有 2 个子节点，那么当节点个数越多的时候，树的高度也会相应变高，这样就会增加磁盘的 I/O 次数，从而影响数据查询的效率。

为了解决降低树的高度的问题，后面就出来了 B 树，它不再限制一个节点就只能有 2 个子节点，而是允许 M 个子节点 (M>2)，从而降低树的高度。

B 树的每一个节点最多可以包括 M 个子节点，M 称为 B 树的阶，所以 B 树就是一个多叉树。

假设 M = 3，那么就是一棵 3 阶的 B 树，特点就是每个节点最多有 2 个（M-1个）数据和最多有 3 个（M个）子节点，超过这些要求的话，就会分裂节点，比如下面的的动图：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

我们来看看一棵 3 阶的 B 树的查询过程是怎样的？

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

假设我们在上图一棵 3 阶的 B 树中要查找的索引值是 9 的记录那么步骤可以分为以下几步：

1. 与根节点的索引(4，8）进行比较，9 大于 8，那么往右边的子节点走；
    
2. 然后该子节点的索引为（10，12），因为 9 小于 10，所以会往该节点的左边子节点走；
    
3. 走到索引为9的节点，然后我们找到了索引值 9 的节点。
    

可以看到，一棵 3 阶的 B 树在查询叶子节点中的数据时，由于树的高度是 3 ，所以在查询过程中会发生 3 次磁盘 I/O 操作。

而如果同样的节点数量在平衡二叉树的场景下，树的高度就会很高，意味着磁盘 I/O 操作会更多。所以，B 树在数据查询中比平衡二叉树效率要高。

但是 B 树的每个节点都包含数据（索引+记录），而用户的记录数据的大小很有可能远远超过了索引数据，这就需要花费更多的磁盘 I/O 操作次数来读到「有用的索引数据」。

而且，在我们查询位于底层的某个节点（比如 A 记录）过程中，「非 A 记录节点」里的记录数据会从磁盘加载到内存，但是这些记录数据是没用的，我们只是想读取这些节点的索引数据来做比较查询，而「非 A 记录节点」里的记录数据对我们是没用的，这样不仅增多磁盘 I/O 操作次数，也占用内存资源。

另外，如果使用 B 树来做范围查询的话，需要使用中序遍历，这会涉及多个节点的磁盘 I/O  问题，从而导致整体速度下降。

### 什么是 B+ 树？

B+ 树就是对 B 树做了一个升级，MySQL 中索引的数据结构就是采用了 B+ 树，B+ 树结构如下图：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

B+ 树与 B 树差异的点，主要是以下这几点：

- 叶子节点（最底部的节点）才会存放实际数据（索引+记录），非叶子节点只会存放索引；
    
- 所有索引都会在叶子节点出现，叶子节点之间构成一个有序链表；
    
- 非叶子节点的索引也会同时存在在子节点中，并且是在子节点中所有索引的最大（或最小）。
    
- 非叶子节点中有多少个子节点，就有多少个索引；
    

下面通过三个方面，比较下 B+ 和 B 树的性能区别。

#### 1、单点查询

B 树进行单个索引查询时，最快可以在 O(1) 的时间代价内就查到，而从平均时间代价来看，会比 B+ 树稍快一些。

但是 B 树的查询波动会比较大，因为每个节点即存索引又存记录，所以有时候访问到了非叶子节点就可以找到索引，而有时需要访问到叶子节点才能找到索引。

**B+ 树的非叶子节点不存放实际的记录数据，仅存放索引，因此数据量相同的情况下，相比存储即存索引又存记录的 B 树，B+树的非叶子节点可以存放更多的索引，因此 B+ 树可以比 B 树更「矮胖」，查询底层节点的磁盘 I/O次数会更少**。

#### 2、插入和删除效率

B+ 树有大量的冗余节点，这样使得删除一个节点的时候，可以直接从叶子节点中删除，甚至可以不动非叶子节点，这样删除非常快，

比如下面这个动图是删除 B+ 树某个叶子节点节点的过程：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

> 注意，：B+ 树对于非叶子节点的子节点和索引的个数，定义方式可能会有不同，有的是说非叶子节点的子节点的个数为 M 阶，而索引的个数为 M-1（这个是维基百科里的定义），因此我本文关于 B+ 树的动图都是基于这个。但是我在前面介绍 B+ 树与 B+ 树的差异时，说的是「非叶子节点中有多少个子节点，就有多少个索引」，主要是 MySQL 用到的 B+ 树就是这个特性。

甚至，B+ 树在删除根节点的时候，由于存在冗余的节点，所以不会发生复杂的树的变形，比如下面这个动图是删除 B+ 树根节点的过程：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

B 树则不同，B 树没有冗余节点，删除节点的时候非常复杂，比如删除根节点中的数据，可能涉及复杂的树的变形，比如下面这个动图是删除 B 树根节点的过程：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

B+ 树的插入也是一样，有冗余节点，插入可能存在节点的分裂（如果节点饱和），但是最多只涉及树的一条路径。而且 B+ 树会自动平衡，不需要像更多复杂的算法，类似红黑树的旋转操作等。  

因此，**B+ 树的插入和删除效率更高**。

#### 3、范围查询

B 树和 B+ 树等值查询原理基本一致，先从根节点查找，然后对比目标数据的范围，最后递归的进入子节点查找。

因为 **B+ 树所有叶子节点间还有一个链表进行连接，这种设计对范围查找非常有帮助**，比如说我们想知道 12 月 1 日和 12 月 12 日之间的订单，这个时候可以先查找到 12 月 1 日所在的叶子节点，然后利用链表向右遍历，直到找到 12 月12 日的节点，这样就不需要从根节点查询了，进一步节省查询需要的时间。

而 B 树没有将所有叶子节点用链表串联起来的结构，因此只能通过树的遍历来完成范围查询，这会涉及多个节点的磁盘 I/O 操作，范围查询效率不如 B+ 树。

因此，存在大量范围检索的场景，适合使用 B+树，比如数据库。而对于大量的单个索引查询的场景，可以考虑 B 树，比如 nosql 的MongoDB。

#### MySQL 中的 B+ 树

MySQL 的存储方式根据存储引擎的不同而不同，我们最常用的就是 Innodb 存储引擎，它就是采用了 B+ 树作为了索引的数据结构。

下图就是 Innodb 里的 B+ 树：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E "图片")

但是 Innodb 使用的  B+ 树有一些特别的点，比如：

- B+ 树的叶子节点之间是用「双向链表」进行连接，这样的好处是既能向右遍历，也能向左遍历。
    
- B+ 树点节点内容是数据页，数据页里存放了用户的记录以及各种信息，每个数据页默认大小是 16 KB。
    

Innodb 根据索引类型不同，分为聚集和二级索引。他们区别在于，聚集索引的叶子节点存放的是实际数据，所有完整的用户记录都存放在聚集索引的叶子节点，而二级索引的叶子节点存放的是主键值，而不是实际数据。

因为表的数据都是存放在聚集索引的叶子节点里，所以 InnoDB 存储引擎一定会为表创建一个聚集索引，且由于数据在物理上只会保存一份，所以聚簇索引只能有一个，而二级索引可以创建多个。

更多关于 Innodb 的 B+ 树，可以看我之前写的这篇：[从数据页的角度看 B+ 树](https://mp.weixin.qq.com/s?__biz=MzUxODAzNDg4NQ==&mid=2247502059&idx=1&sn=ccbee22bda8c3d6a98237be769a7c89c&scene=21#wechat_redirect)。

### 总结

MySQL 是会将数据持久化在硬盘，而存储功能是由 MySQL 存储引擎实现的，所以讨论 MySQL 使用哪种数据结构作为索引，实际上是在讨论存储引使用哪种数据结构作为索引，InnoDB 是 MySQL 默认的存储引擎，它就是采用了 B+ 树作为索引的数据结构。

要设计一个 MySQL 的索引数据结构，不仅仅考虑数据结构增删改的时间复杂度，更重要的是要考虑磁盘 I/0 的操作次数。因为索引和记录都是存放在硬盘，硬盘是一个非常慢的存储设备，我们在查询数据的时候，最好能在尽可能少的磁盘 I/0 的操作次数内完成。

二分查找树虽然是一个天然的二分结构，能很好的利用二分查找快速定位数据，但是它存在一种极端的情况，每当插入的元素都是树内最大的元素，就会导致二分查找树退化成一个链表，此时查询复杂度就会从 O(logn)降低为 O(n)。

为了解决二分查找树退化成链表的问题，就出现了自平衡二叉树，保证了查询操作的时间复杂度就会一直维持在 O(logn) 。但是它本质上还是一个二叉树，每个节点只能有 2 个子节点，随着元素的增多，树的高度会越来越高。

而树的高度决定于磁盘  I/O 操作的次数，因为树是存储在磁盘中的，访问每个节点，都对应一次磁盘 I/O 操作，也就是说树的高度就等于每次查询数据时磁盘 IO 操作的次数，所以树的高度越高，就会影响查询性能。

B 树和 B+ 都是通过多叉树的方式，会将树的高度变矮，所以这两个数据结构非常适合检索存于磁盘中的数据。

但是 MySQL 默认的存储引擎 InnoDB 采用的是 B+ 作为索引的数据结构，原因有：

- B+ 树的非叶子节点不存放实际的记录数据，仅存放索引，因此数据量相同的情况下，相比存储即存索引又存记录的 B 树，B+树的非叶子节点可以存放更多的索引，因此 B+ 树可以比 B 树更「矮胖」，查询底层节点的磁盘 I/O次数会更少。
    
- B+ 树有大量的冗余节点（所有非叶子节点都是冗余索引），这些冗余索引让 B+ 树在插入、删除的效率都更高，比如删除根节点的时候，不会像 B 树那样会发生复杂的树的变化；
    
- B+ 树叶子节点之间用链表连接了起来，有利于范围查询，而 B 树要实现范围查询，因此只能通过树的遍历来完成范围查询，这会涉及多个节点的磁盘 I/O 操作，范围查询效率不如 B+ 树。
    

完！

![](http://mmbiz.qpic.cn/mmbiz_png/J0g14CUwaZfTwwjfpJhXgIrYMgtVcLhQQBVb02clZfKicbxaibSTNJqXe9Zu8ydiavZKJWJAIhKcnD9hBuKU92JZQ/300?wx_fmt=png&wxfrom=19)

**小林coding**

专注图解计算机基础，让天下没有难懂的八股文！刷题网站：xiaolincoding.com

471篇原创内容

公众号

图解MySQL34

图解MySQL · 目录

上一篇换一个角度看 B+ 树下一篇谁还没经历过死锁呢

阅读 1.3万

​

写留言

**留言 58**

- 小林
    
    2021年12月17日
    
    赞33
    
    补充一句：没有B减树这个玩意，书上的「b-树」指的是b树，中间那个符号就是普通的横杠，不是减的意思![[破涕为笑]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
    置顶
    
- 王 先 生 ¹
    
    2021年12月17日
    
    赞45
    
    你怕不是个直男吧，正确的回答难道不应该是 ： “MYSQL为什么喜欢B+树 ，就像我为什么喜欢你一样。不是因为你有多完美，而是因为在我眼里你是最好最适合最独一无二的。”
    
    小林coding
    
    作者2021年12月17日
    
    赞4
    
    ![[强]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
- Curry
    
    2021年12月17日
    
    赞20
    
    红黑树 安排一下![[吃瓜]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
    小林coding
    
    作者2021年12月17日
    
    赞5
    
    这届读者这么严格了吗![[旺柴]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
- 志田
    
    2021年12月17日
    
    赞8
    
    你肯定没有女朋友，女朋友不会问这种问题。![[奸笑]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
- 丢丢原野
    
    2021年12月17日
    
    赞6
    
    全文没有提到女朋友，所以你没有女朋友
    
    小林coding
    
    作者2021年12月17日
    
    赞7
    
    ![[旺柴]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)大意了吧，封面就是我女朋友，骑着电动车载我
    
- 程序员贺同学
    
    2021年12月17日
    
    赞5
    
    所以女朋友知道答案了吗![[奸笑]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
    小林coding
    
    作者2021年12月17日
    
    赞4
    
    已经让她在背着了
    
- 认真
    
    2021年12月17日
    
    赞5
    
    是你自己new出来的女朋友吧![[可怜]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
    小林coding
    
    作者2021年12月17日
    
    赞1
    
    new object()
    
- 水草
    
    2021年12月17日
    
    赞4
    
    封面就在开车
    
- modo
    
    2021年12月17日
    
    赞4
    
    点进来吃瓜的，但是全文没有女朋友![[Emm]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
- 会微笑的糖果
    
    2021年12月18日
    
    赞3
    
    笑死，女朋友只会问我#FF7F00和#FF6EC7哪个好看
    
- 冰河
    
    2021年12月17日
    
    赞1
    
    林总，这些动图咋弄的呀
    
    小林coding
    
    作者2021年12月17日
    
    赞3
    
    这个网站：https://www.cs.usfca.edu/~galles/visualization/Algorithms.html
    
- ZhangYX
    
    2021年12月17日
    
    赞3
    
    女朋友问的问题比面试官还刁难人![[撇嘴]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
    小林coding
    
    作者2021年12月17日
    
    赞2
    
    这面试必问的吧。。
    
- 友
    
    2021年12月17日
    
    赞2
    
    马上要面试了 准备恶补一下
    
- Rafael'Kuai
    
    2021年12月17日
    
    赞2
    
    由于树是存储在磁盘中的，访问每个节点，都对应一次磁盘 I/O 操作（假设一个节点的大小「小于」操作系统的最小读写单位块的大小）。那应该也有可能一个节点的大小[远小于]操作系统的最小读写单位块的大小，而导致操作系统一次读写可以读取多个节点吧？![[打脸]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
    小林coding
    
    作者2021年12月17日
    
    赞1
    
    嗯嗯是的，如果节点刚好是连续存储的话，
    
- SnailClimb
    
    2021年12月17日
    
    赞2
    
    硬核！三连了
    
    小林coding
    
    作者2021年12月17日
    
    赞1
    
    ![[奸笑]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)g哥，杠杠的
    
- Hui
    
    2021年12月17日
    
    赞2
    
    电瓶车违规带人 罚80![[旺柴]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
    小林coding
    
    作者2021年12月17日
    
    赞1
    
    就你精明！![[奸笑]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
- 快乐星球
    
    2021年12月17日
    
    赞1
    
    【也就是一次磁盘 I/O 操作会直接读写 8 个扇区。】 这里是最少读写8扇区吧
    
    小林coding
    
    作者2021年12月17日
    
    赞2
    
    「最少读写8扇区」，有可以读9个扇区的意思，感觉不太对。
    
- 暗夜大魔王
    
    2021年12月17日
    
    赞2
    
    所以女朋友知道答案了吗![[呲牙]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
- 辉
    
    2021年12月17日
    
    赞2
    
    笑着画 x 哭着画 √
    
- 雪人
    
    2022年1月27日
    
    赞1
    
    以前别人对我说：你心里没点儿B shu吗？我当时一脸懵逼，现在回想起来，原来他是想跟我探讨一下技术问题![[旺柴]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)！PS：这是我看二叉树看得最明白的一次。。。
    
    小林coding
    
    作者2022年1月27日
    
    赞
    
    哈哈可以
    
- Cruise Zhao
    
    2022年1月6日
    
    赞
    
    「而且会随着插入的元素越多，树的高度也变高，意味着需要磁盘 IO 操作的次数就越多，这样导致查询性能严重下降，再加上不能范围查询，所以不适合作为数据库的索引结构。」 二叉查找树这一节，二叉查找树不能范围查询么？还是范围查询的效率比较低？下面可是说了B树可以范围查询的丫
    
    小林coding
    
    作者2022年1月6日
    
    赞1
    
    之前写的不太对，二叉查找树中序遍历可以输出递增的序列，支持简单范围查询，但是范围查询效率很低。
    
- 小军
    
    2021年12月18日
    
    赞1
    
    来个关于 MVCC的
    
- 山雨欲来
    
    2021年12月17日
    
    赞1
    
    求林大哥安排红黑树
    
- Jeff-BlindCat
    
    2021年12月17日
    
    赞1
    
    什么树不树的，你们竟然牵手竟然来牵手打击无数…，普法小知识注意带好安全帽，安全驾驶。![[机智]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
- wangchen
    
    2021年12月17日
    
    赞1
    
    赞了再看，反正背不下来
    
- !
    
    2021年12月17日
    
    赞1
    
    开车？![[旺柴]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
- 言寸
    
    2021年12月17日
    
    赞1
    
    6啊![😱](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
- Ethan
    
    2021年12月17日
    
    赞1
    
    你的女朋友想学红黑树了咋办
    
- TZTolerant_十点不睡_
    
    2021年12月17日
    
    赞1
    
    回复志田：格局小了![[旺柴]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
- 斑马还没有睡
    
    2021年12月17日
    
    赞1
    
    我是一楼吧
    
- 当山水深入繁华。
    
    浙江2022年5月25日
    
    赞
    
    作者大大，有个问题。一个mysql的页是16KB，磁盘I/O一次是4KB的话，那不是得进行4次磁盘I/O才能读取一页么？但是一直以来看到的文章都在说一层就是一次磁盘IO（是我理解的不对么？）
    
    当山水深入繁华。
    
    浙江2022年5月25日
    
    赞
    
    当然，也提到过磁盘IO会进行预读，所以是一次预读就会预读出一个16kb的页的单位么
    
    小林coding
    
    作者2022年5月25日
    
    赞
    
    磁盘 io 最小读写单位是4kb，但是并不是说每次磁盘io只能读4kb。如果磁盘的数据是顺序的，那么就可以通过一次顺序io读取多个扇区（1个扇区512字节）。
    
    小林coding
    
    作者2022年5月25日
    
    赞
    
    操作系统是以页为单位来管理内存，它可以一次加载整数倍的页，而 innoDB 的页大小为 16KB，刚好是操作系统页（4KB）的 4 倍，所以可以指定在读取的起始地址连续读取 4 个操作系统页，即 16 KB。
    
- 宋野
    
    2022年2月24日
    
    赞
    
    所有非叶子节点都是冗余索引，这句话是啥意思
    
    小林coding
    
    作者2022年2月24日
    
    赞
    
    非叶子节点的索性值，在叶子节点索性值也有，因为非叶子节点的索引值是叶子节点中最小的索引值，冗余就是副本的意思。
    
- 陵游龙胆👊
    
    2021年12月23日
    
    赞
    
    看到文中“注意”内容，很想了解一下维基百科中B+树定义和mysql索引中B+树实现区别的原因，问了dba同学也没搞这么细过
    
    小林coding
    
    作者2021年12月23日
    
    赞
    
    我也不知道哈哈，不过感觉mysql的b+树看起来容易理解一点
    
- 嗨⃰，人⃰
    
    2021年12月21日
    
    赞
    
    感觉不同的文章对B树的解释都不一样：B(balance)树就是平衡树 B(binary-search)树= 二叉搜索树
    
    小林coding
    
    作者2021年12月21日
    
    赞
    
    维基百科上b树的意思是平衡树，我觉得这个才是正确的。
    
- 程序员石头
    
    2021年12月18日
    
    赞
    
    有的动图有点虚啊![[呲牙]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
    小林coding
    
    作者2021年12月18日
    
    赞
    
    哈哈是啊，没办法，一个网站搞定，他的动物就很小。
    
- 认真的雪
    
    2021年12月18日
    
    赞
    
    那为什么不写个·B树。怕老司机曲解吗![[机智]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
- 友
    
    2021年12月17日
    
    赞
    
    由于数据库的索引是保存到磁盘上的，因此当我们通过索引查找某行数据的时候，就需要先从磁盘读取索引到内存，再通过索引从磁盘中找到某行数据，然后读入到内存，也就是说查询过程中会发生多次磁盘 I/O，而磁盘 I/O 次数越多，所消耗的时间也就越大。 —----------- 这个地方 我在想 把索引文件读取到内存是一次性拉取吗（应该是的呀） 然后再通过索引从磁盘找到数据是回表吗
    
    小林coding
    
    作者2021年12月17日
    
    赞
    
    不是回表，回表是使用二级索引查询时，查询的数据不在二级索引数里，这时就需要回聚族索引查到对应的数据。 文中这里是想表达，通过索引找到对应的存储数据的磁盘块。
    

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/J0g14CUwaZfTwwjfpJhXgIrYMgtVcLhQQBVb02clZfKicbxaibSTNJqXe9Zu8ydiavZKJWJAIhKcnD9hBuKU92JZQ/300?wx_fmt=png&wxfrom=18)

小林coding

关注

944230

58

写留言

**留言 58**

- 小林
    
    2021年12月17日
    
    赞33
    
    补充一句：没有B减树这个玩意，书上的「b-树」指的是b树，中间那个符号就是普通的横杠，不是减的意思![[破涕为笑]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
    置顶
    
- 王 先 生 ¹
    
    2021年12月17日
    
    赞45
    
    你怕不是个直男吧，正确的回答难道不应该是 ： “MYSQL为什么喜欢B+树 ，就像我为什么喜欢你一样。不是因为你有多完美，而是因为在我眼里你是最好最适合最独一无二的。”
    
    小林coding
    
    作者2021年12月17日
    
    赞4
    
    ![[强]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
- Curry
    
    2021年12月17日
    
    赞20
    
    红黑树 安排一下![[吃瓜]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
    小林coding
    
    作者2021年12月17日
    
    赞5
    
    这届读者这么严格了吗![[旺柴]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
- 志田
    
    2021年12月17日
    
    赞8
    
    你肯定没有女朋友，女朋友不会问这种问题。![[奸笑]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
- 丢丢原野
    
    2021年12月17日
    
    赞6
    
    全文没有提到女朋友，所以你没有女朋友
    
    小林coding
    
    作者2021年12月17日
    
    赞7
    
    ![[旺柴]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)大意了吧，封面就是我女朋友，骑着电动车载我
    
- 程序员贺同学
    
    2021年12月17日
    
    赞5
    
    所以女朋友知道答案了吗![[奸笑]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
    小林coding
    
    作者2021年12月17日
    
    赞4
    
    已经让她在背着了
    
- 认真
    
    2021年12月17日
    
    赞5
    
    是你自己new出来的女朋友吧![[可怜]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
    小林coding
    
    作者2021年12月17日
    
    赞1
    
    new object()
    
- 水草
    
    2021年12月17日
    
    赞4
    
    封面就在开车
    
- modo
    
    2021年12月17日
    
    赞4
    
    点进来吃瓜的，但是全文没有女朋友![[Emm]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
- 会微笑的糖果
    
    2021年12月18日
    
    赞3
    
    笑死，女朋友只会问我#FF7F00和#FF6EC7哪个好看
    
- 冰河
    
    2021年12月17日
    
    赞1
    
    林总，这些动图咋弄的呀
    
    小林coding
    
    作者2021年12月17日
    
    赞3
    
    这个网站：https://www.cs.usfca.edu/~galles/visualization/Algorithms.html
    
- ZhangYX
    
    2021年12月17日
    
    赞3
    
    女朋友问的问题比面试官还刁难人![[撇嘴]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
    小林coding
    
    作者2021年12月17日
    
    赞2
    
    这面试必问的吧。。
    
- 友
    
    2021年12月17日
    
    赞2
    
    马上要面试了 准备恶补一下
    
- Rafael'Kuai
    
    2021年12月17日
    
    赞2
    
    由于树是存储在磁盘中的，访问每个节点，都对应一次磁盘 I/O 操作（假设一个节点的大小「小于」操作系统的最小读写单位块的大小）。那应该也有可能一个节点的大小[远小于]操作系统的最小读写单位块的大小，而导致操作系统一次读写可以读取多个节点吧？![[打脸]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
    小林coding
    
    作者2021年12月17日
    
    赞1
    
    嗯嗯是的，如果节点刚好是连续存储的话，
    
- SnailClimb
    
    2021年12月17日
    
    赞2
    
    硬核！三连了
    
    小林coding
    
    作者2021年12月17日
    
    赞1
    
    ![[奸笑]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)g哥，杠杠的
    
- Hui
    
    2021年12月17日
    
    赞2
    
    电瓶车违规带人 罚80![[旺柴]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
    小林coding
    
    作者2021年12月17日
    
    赞1
    
    就你精明！![[奸笑]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
- 快乐星球
    
    2021年12月17日
    
    赞1
    
    【也就是一次磁盘 I/O 操作会直接读写 8 个扇区。】 这里是最少读写8扇区吧
    
    小林coding
    
    作者2021年12月17日
    
    赞2
    
    「最少读写8扇区」，有可以读9个扇区的意思，感觉不太对。
    
- 暗夜大魔王
    
    2021年12月17日
    
    赞2
    
    所以女朋友知道答案了吗![[呲牙]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
- 辉
    
    2021年12月17日
    
    赞2
    
    笑着画 x 哭着画 √
    
- 雪人
    
    2022年1月27日
    
    赞1
    
    以前别人对我说：你心里没点儿B shu吗？我当时一脸懵逼，现在回想起来，原来他是想跟我探讨一下技术问题![[旺柴]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)！PS：这是我看二叉树看得最明白的一次。。。
    
    小林coding
    
    作者2022年1月27日
    
    赞
    
    哈哈可以
    
- Cruise Zhao
    
    2022年1月6日
    
    赞
    
    「而且会随着插入的元素越多，树的高度也变高，意味着需要磁盘 IO 操作的次数就越多，这样导致查询性能严重下降，再加上不能范围查询，所以不适合作为数据库的索引结构。」 二叉查找树这一节，二叉查找树不能范围查询么？还是范围查询的效率比较低？下面可是说了B树可以范围查询的丫
    
    小林coding
    
    作者2022年1月6日
    
    赞1
    
    之前写的不太对，二叉查找树中序遍历可以输出递增的序列，支持简单范围查询，但是范围查询效率很低。
    
- 小军
    
    2021年12月18日
    
    赞1
    
    来个关于 MVCC的
    
- 山雨欲来
    
    2021年12月17日
    
    赞1
    
    求林大哥安排红黑树
    
- Jeff-BlindCat
    
    2021年12月17日
    
    赞1
    
    什么树不树的，你们竟然牵手竟然来牵手打击无数…，普法小知识注意带好安全帽，安全驾驶。![[机智]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
- wangchen
    
    2021年12月17日
    
    赞1
    
    赞了再看，反正背不下来
    
- !
    
    2021年12月17日
    
    赞1
    
    开车？![[旺柴]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
- 言寸
    
    2021年12月17日
    
    赞1
    
    6啊![😱](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
- Ethan
    
    2021年12月17日
    
    赞1
    
    你的女朋友想学红黑树了咋办
    
- TZTolerant_十点不睡_
    
    2021年12月17日
    
    赞1
    
    回复志田：格局小了![[旺柴]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
- 斑马还没有睡
    
    2021年12月17日
    
    赞1
    
    我是一楼吧
    
- 当山水深入繁华。
    
    浙江2022年5月25日
    
    赞
    
    作者大大，有个问题。一个mysql的页是16KB，磁盘I/O一次是4KB的话，那不是得进行4次磁盘I/O才能读取一页么？但是一直以来看到的文章都在说一层就是一次磁盘IO（是我理解的不对么？）
    
    当山水深入繁华。
    
    浙江2022年5月25日
    
    赞
    
    当然，也提到过磁盘IO会进行预读，所以是一次预读就会预读出一个16kb的页的单位么
    
    小林coding
    
    作者2022年5月25日
    
    赞
    
    磁盘 io 最小读写单位是4kb，但是并不是说每次磁盘io只能读4kb。如果磁盘的数据是顺序的，那么就可以通过一次顺序io读取多个扇区（1个扇区512字节）。
    
    小林coding
    
    作者2022年5月25日
    
    赞
    
    操作系统是以页为单位来管理内存，它可以一次加载整数倍的页，而 innoDB 的页大小为 16KB，刚好是操作系统页（4KB）的 4 倍，所以可以指定在读取的起始地址连续读取 4 个操作系统页，即 16 KB。
    
- 宋野
    
    2022年2月24日
    
    赞
    
    所有非叶子节点都是冗余索引，这句话是啥意思
    
    小林coding
    
    作者2022年2月24日
    
    赞
    
    非叶子节点的索性值，在叶子节点索性值也有，因为非叶子节点的索引值是叶子节点中最小的索引值，冗余就是副本的意思。
    
- 陵游龙胆👊
    
    2021年12月23日
    
    赞
    
    看到文中“注意”内容，很想了解一下维基百科中B+树定义和mysql索引中B+树实现区别的原因，问了dba同学也没搞这么细过
    
    小林coding
    
    作者2021年12月23日
    
    赞
    
    我也不知道哈哈，不过感觉mysql的b+树看起来容易理解一点
    
- 嗨⃰，人⃰
    
    2021年12月21日
    
    赞
    
    感觉不同的文章对B树的解释都不一样：B(balance)树就是平衡树 B(binary-search)树= 二叉搜索树
    
    小林coding
    
    作者2021年12月21日
    
    赞
    
    维基百科上b树的意思是平衡树，我觉得这个才是正确的。
    
- 程序员石头
    
    2021年12月18日
    
    赞
    
    有的动图有点虚啊![[呲牙]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
    小林coding
    
    作者2021年12月18日
    
    赞
    
    哈哈是啊，没办法，一个网站搞定，他的动物就很小。
    
- 认真的雪
    
    2021年12月18日
    
    赞
    
    那为什么不写个·B树。怕老司机曲解吗![[机智]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
- 友
    
    2021年12月17日
    
    赞
    
    由于数据库的索引是保存到磁盘上的，因此当我们通过索引查找某行数据的时候，就需要先从磁盘读取索引到内存，再通过索引从磁盘中找到某行数据，然后读入到内存，也就是说查询过程中会发生多次磁盘 I/O，而磁盘 I/O 次数越多，所消耗的时间也就越大。 —----------- 这个地方 我在想 把索引文件读取到内存是一次性拉取吗（应该是的呀） 然后再通过索引从磁盘找到数据是回表吗
    
    小林coding
    
    作者2021年12月17日
    
    赞
    
    不是回表，回表是使用二级索引查询时，查询的数据不在二级索引数里，这时就需要回聚族索引查到对应的数据。 文中这里是想表达，通过索引找到对应的存储数据的磁盘块。