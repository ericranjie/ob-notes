Original RiemannChow 老周聊架构  _2022年03月21日 18:20_

## 一、前言

最近有很多读者要我出一些面试题的文章，一般我会给他一个老周整理的电子书，但有些读者反馈回来的面试题我觉得还是蛮经典的，而老周又在写系列的文章，本着对读者负责的态度，我会穿插写几篇我认为比较经典的面试题，让大家对这种经典问题不再是背八股文，而是深入底层原理以及数据结构。后续再碰到这类问题，不管哪个公司问的，你都会得心应手、从容不迫的回答。

今天这一篇腾讯的一道面试题：说一说 MySQL 中索引的底层原理，先不管哪个公司，如果老周是面试官，MySQL 这块我很可能也会用这道来考察，因为这可以考察出你对 MySQL 本质是否理解，而且现在很多人动不动就索引优化，索引底层是个啥我相信还有蛮多人不是特别了解。

没关系，跟着老周走，这一篇我们彻底拿下 MySQL 的索引。

## 二、索引是什么？

索引是对数据库表中一列或多列的值进行排序的一种结构，使用索引可提高数据库中特定数据的查询速度。

索引的目的在于提高查询效率，就像书的目录一样。一本 1000 页的书，如果你想快速找到其中的某一个知识点，在不借助目录的情况下，这个事情是不是很难完成？同样，对于数据库的表而言，索引其实就是它的“目录”。

## 三、索引的分类

常见的索引类型有：主键索引、唯一索引、普通索引、全文索引、组合索引

### 3.1 主键索引

即主索引，根据主键 pk_clolum（length）建立索引，不允许重复，不允许空值。

```sql
ALTER TABLE 'table_name' ADD PRIMARY KEY pk_index('col')；
```

### 3.2 唯一索引

用来建立索引的列的值必须是唯一的，允许空值。

```sql
ALTER TABLE 'table_name' ADD UNIQUE index_name('col')；
```

### 3.3 普通索引

MySQL 中的基本索引类型，允许在定义索引的列中插入重复值和空值。

```sql
ALTER TABLE 'table_name' ADD INDEX index_name('col')；
```

### 3.4 全文索引

全文索引类型为 FULLTEXT，用大文本对象的列构建的索引，在定义索引的列上支持值的全文查找，允许在这些索引的列中插入重复值和空值。全文索引可以在 CHAR、VARCHAR 或 TEXT 类型的列上创建。MySQL 中只有 MyISAM 存储引擎支持全文索引。

```sql
ALTER TABLE 'table_name' ADD FULLTEXT INDEX ft_index('col')；
```

### 3.5 组合索引

在表的多个字段组合上创建的索引，只有在查询条件中使用了这些字段的左边字段时，索引才会被使用。使用组合索引时遵循最左前缀原则，并且组合索引这多个列中的值不允许有空值。

```sql
ALTER TABLE 'table_name' ADD INDEX index_name('col1','col2','col3')；
```

注意了，遵循最左前缀原则很重要！！！把最常用作为检索或排序的列放在最左，依次递减，组合索引相当于建立了 col1、col1col2、col1col2col3 三个索引，而 col2 或者 col3 是不能使用索引的。

## 四、索引的常见模型

索引的出现是为了提高查询效率，但是实现索引的方式却有很多种，所以这里也就引入了索引模型的概念。可以用于提高读写效率的数据结构很多，这里我先给你介绍几种常见的数据结构，它们分别是哈希表、有序数组和搜索树。搜索树又可以分为二叉搜索树、平衡二叉树、B 树、B+ 树。

MySQL 中实现索引的方式主要有两种：BTREE 和 HASH，具体和表的存储引擎有关；MyISAM 和 InnoDB 存储引擎只支持 BTREE 索引，而 MEMROY/HEAP 存储引擎可以支持 HASH 和 BTREE 索引。

这里老周说的索引模型不仅仅针对 MySQL 的哈，常见的几种都会说一下，让大家对实现索引的方式有个更加全局的了解。

### 4.1 哈希表

哈希表是一种以键-值（key-value）存储数据的结构，我们只要输入待查找的值即 key，就可以找到其对应的值即 value。哈希的思路很简单，把值放在数组里，用一个哈希函数把 key 换算成一个确定的位置，然后把 value 放在数组的这个位置。

多个 key 值经过哈希函数的换算，会出现同一个值的情况。处理这种情况的一种方法是，拉出一个链表。有没有发现，其实 HashMap 就是这样的数据结构。

假设，你现在维护着一个用户 id 和姓名的表，需要根据用户 id 查找对应的名字，这时对应的哈希索引的示意图如下所示：

![Image](https://mmbiz.qpic.cn/mmbiz_png/80icw67Ot0qJsEtic2EMiasxzVhLYQmMgcQ5FNibrpaft2Lgec9icn7ZcibGtB32Az8UzmvLr2mM0tPCulxqTU0ZLJcw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1 "在这里插入图片描述")

从上图不难发现，user_id2 和 user_id3 经过哈希函数计算出来的值都是 4，这就是哈希冲突，但没有关系，后面会拉出一个链表出来。当你要查询 user_id2 姓名的时候，先经过哈希函数得到 4，再按顺序遍历，找到 name2 。

这里要提一下这个点，user_idn 的值并不是递增的，这样有个好处，新增用户的时候会很快，只需要往后追加；但也有缺点，因为不是有序的，所以哈希索引做区间查询的速度是很慢的。

所以，哈希表这种结构适用于只有等值查询的场景，比如 Memcached 及其他一些 NoSQL 引擎。

### 4.2 有序数组

上面的哈希表不适合范围查询，而有序数组在等值查询和范围查询场景中的性能就都非常优秀。

![Image](https://mmbiz.qpic.cn/mmbiz_png/80icw67Ot0qJsEtic2EMiasxzVhLYQmMgcQiaVMAA49Ha4ibyuibmAdjSdua8TT209183vCyQzWm2BG7gt78Q0zb1iccQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1 "在这里插入图片描述")

我们来分别看看有序数组针对等值查询与范围查询的时间复杂度，等值查询比如你要查 user_id2 对应的名字，用二分法就可以快速得到，这个时间复杂度是 O(log(N))。范围查询比如你要查询 \[user_id2, user_id5\] 之间的用户，同样利用二分查找先找到 user_id2（如果不存在 user_id2，就找到大于 user_id2 的第一个用户），然后向右遍历，直到查到第一个大于 user_id5，退出循环。

查询场景确实很优秀，但有序数组也有缺点，更新数据的时候那就有点麻烦了，比如你往中间插入一条用户，该用户后面所有的记录都要往后移动，成本太高了。

所以，有序数组索引只适用于静态存储引擎，比如你要保存的是历史某一年某个城市的所有人口信息，这类不会再修改的历史数据。

### 4.3 二叉树

二叉树是一棵树，其中每个节点都不能有多余两个的儿子。二叉树也是最经典的树结构，下面讲的各种树都是基于二叉树改进而来的。

我们先来看下二叉树的特点：

- 二叉树的平均时间复杂度为 O(log(N))

- 每个节点最多只能有两个子节点

一般二叉树：

![Image](https://mmbiz.qpic.cn/mmbiz_png/80icw67Ot0qJsEtic2EMiasxzVhLYQmMgcQurwfEibXKfzScKhzN1W4XvdJzLS0ff6rQfdAzrxIa0FNLkEHyvxNmrw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1 "在这里插入图片描述")

最坏情形的二叉树，出现链化的情况：

![Image](https://mmbiz.qpic.cn/mmbiz_png/80icw67Ot0qJsEtic2EMiasxzVhLYQmMgcQ3J8Gs5KHHUmAHE16fPQIed6zCVR5orfT0QcYOAXXU2A9RXggUh4rHA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1 "在这里插入图片描述")

实现：

因为一个二叉树节点最多有两个子节点，所以我们可以保存直接链接到它们的链。树节点的声明在结构上类似于双链表的声明，在声明中，节点就是由 element（元素）的信息加上两个到其它节点的引用（left 和 right）组成的结构。如下：

```java
public class BinaryNode {
Object element; // The data in the mode
BinaryNode left; // Left child
BinaryNode right; // Right child
}
```

网上有很多说常见的索引使用的数据结构是二叉树，其实不够严谨，二叉树在索引的应用主要是二叉搜索树以及后续演进的平衡二叉树、B 树、B+ 树等。二叉树其实有许多与搜索无关的重要应用，比如在编译器的设计等领域。

### 4.4 二叉查找树

查找树 ADT（Abstract Data Type）也就是二叉查找树，它是索引使用的数据结构中的一个应用。

我们先来看下二叉查找树的特点：

- 二叉查找树的平均时间复杂度为 O(log(N))

- 每个节点最多只能有两个子节点

- 左子节点的值小于本节点的值，右子节点的值大于本节点的值。

我们来看下如下结构：

![Image](https://mmbiz.qpic.cn/mmbiz_png/80icw67Ot0qJsEtic2EMiasxzVhLYQmMgcQlxoVN50qxFn0GkpcMTKhQYpUd9NqEKV0nVBYRVMWNPO1ly6ibh7YrZw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1 "在这里插入图片描述")

根据二叉查找树的特点，我们很快就能知道，最左边的图是二叉查找树；中间的树则不是，因为在为 6 的根节点左边出现了一个为 7 的节点，不满足二叉树第三点的特性；右边的树则是链化的二叉查找树。

实现：

二叉查找树要求所有的项都能够排序，我们以 Java 语言为例，可以实现 Comparable 接口来实现这个排序的性质，树中的两项总可以使用 compareTo 方法进行比较。结构如下：

```java
public class BinarySearchTree<T extends Comparable<? super T>> {
private static class BinaryNode<T> {
T element; // The data in the mode
BinaryNode<T> left; // Left child
BinaryNode<T> right; // Right child
BinaryNode(T t) {
this(t, null, null);
}
BinaryNode(T t, BinaryNode<T> lt, BinaryNode<T> rt) {
this.element = t;
this.left = lt;
this.right = rt;
}
}    // ...
}
```

### 4.5 平衡二叉查找树

平衡二叉查找树也称 AVL 树（Adelson-Velsky-Landis Tree），平衡二叉查找树的出现就是为了解决上面的二叉查找树可能出现链化的场景。

我们先来看下平衡二叉查找树的特点：

- 二叉查找树的平均时间复杂度为 O(log(N))

- 每个节点最多只能有两个子节点

- 每个节点的左右子树高度差不超过 1

我们来看下如下结构：

![Image](https://mmbiz.qpic.cn/mmbiz_png/80icw67Ot0qJsEtic2EMiasxzVhLYQmMgcQymqTdnmb9ESw2CtFFRPbSuOfO8A7EQbIuwXbgwD6BANeJc1YYSEr3g/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1 "在这里插入图片描述")

左边的是 AVL 树，它的任何节点的两个子树的高度差别都 \<=1；而右边的不是AVL树，因为 7 的两棵子树的高度相差为 2（以 2 为根节点的树的高度是 3，而以 8 为根节点的树的高度是 1）。

实现：

```java
public class AVLTree<T extends Comparable<? super T>> {
private AVLTreeNode<T> rootNode; // root node
private static class AVLTreeNode<T> {
T element; // The data in the mode
BinaryNode<T> left; // Left child
BinaryNode<T> right; // Right child
int height; // Height
AVLTreeNode(T t) {            this(t, null, null);        }        AVLTreeNode(T t, BinaryNode<T> lt, BinaryNode<T> rt) {            this.element = t;            this.left = lt;            this.right = rt;            this.height = 0;        }    }    // ...
}
```

平衡二叉查找树通过旋转可以使每个节点的左右子树高度差不超过 1，也就是避免链化问题。磁盘的 IO 由树高决定，数据量越多，遍历次数越多，IO 次数就越多，就越慢。

### 4.6 BTree

上面的平衡二叉查找树可能由于数据量比较大的时候，树高就会很高，IO 次数也就越多，这样的话会影响性能。所以，B 树就是为了解决即使数据量很大也能让 IO 次数很少的一种树。

BTree 是平衡搜索多叉树，设树的度为2d（d>1），高度为 h，那么 BTree 要满足以下条件：

- 每个叶子节点的高度一样，等于 h。

- 每个非叶子节点由 n-1 个 key 和 n 个指针 point 组成，其中 d\<=n\<=2d，key 和 point 相互间隔，节点两端一定是 key。

- 叶子节点指针都为 null

- 非叶子节点都是 \[key，data\] 二元组，其中 key 表示作为索引的键，data 为键值所在行的数据。

BTree 结构如下：

![Image](https://mmbiz.qpic.cn/mmbiz_png/80icw67Ot0qJsEtic2EMiasxzVhLYQmMgcQc4VaUN8b5vMpnG32NZEWk4sHstq4evniaibiaTQicVDtIKEePJQIMuVKFA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1 "在这里插入图片描述")

BTree 可以使用二分查找的查找方式，查找复杂度为 h * O(log(N))，一般来说树的高度是很小的，一般为 3 左右，因此 BTree 是一个非常高效的查找结构。

### 4.7 B+Tree

B+Tree 是 BTree 的一个变种，设 d 为树的度数，h 为树的高度，B+Tree 和 BTree 的不同主要在于：

- B+Tree 中的非叶子结点不存储数据，只存储键值。

- B+Tree 的叶子节点没有指针，所有键值都会出现在叶子节点上，且 key 存储的键值对应 data 数据的物理地址。

- B+Tree 的每个非叶子节点由 n 个键值 key 和 n 个指针 point 组成

B+Tree 的结构如下：

![Image](https://mmbiz.qpic.cn/mmbiz_png/80icw67Ot0qJsEtic2EMiasxzVhLYQmMgcQNBVGn20kGkKVyryELDWDcAn0n4YCNgBNH2ClZzCt6ZmCNc5ERESGjw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1 "在这里插入图片描述")

### 4.8 带有顺序访问指针的 B+Tree

一般在数据库系统或文件系统中使用的 B+Tree 结构都在经典 B+Tree 的基础上进行了优化，增加了顺序访问指针。

![Image](https://mmbiz.qpic.cn/mmbiz_png/80icw67Ot0qJsEtic2EMiasxzVhLYQmMgcQMDhrS3gics04QeExl90ZZsrva0L7Kr4p8V0qyXdzaGc2vK3r1Zfrw4A/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1 "在这里插入图片描述")

如上图所示，在 B+Tree 的每个叶子节点增加一个指向相邻叶子节点的指针，就形成了带有顺序访问指针的 B+Tree。做这个优化的目的是为了提高区间访问的性能，如果要查询 key 为从 18 到 49 的所有数据记录，当找到 18 后，只需顺着节点和指针顺序遍历就可以一次性访问到所有数据节点，极大提到了区间查询效率。

## 五、MySQL 为什么使用 B+Tree

一般来说，索引本身也很大，不可能全部存储在内存中，因此索引往往以索引文件的形式存储在磁盘上。这样的话，索引查找过程中就要产生磁盘 I/O 消耗，相对于内存存取，I/O 存取的消耗要高几个数量级，所以评价一个数据结构作为索引的优劣最重要的指标就是在查找过程中磁盘 I/O 操作次数的渐进复杂度。换句话说，索引的结构组织要尽量减少查找过程中磁盘 I/O 的存取次数。

### 5.1 磁盘 IO 与预读

上面说到了访问磁盘，那么这里先简单介绍一下磁盘 IO 和预读，磁盘读取数据靠的是机械运动，每次读取数据花费的时间可以分为寻道时间、旋转延迟、传输时间三个部分，寻道时间指的是磁臂移动到指定磁道所需要的时间，主流磁盘一般在 5ms 以下；旋转延迟就是我们经常听说的磁盘转速，比如一个磁盘 7200 转，表示每分钟能转 7200 次，也就是说 1 秒钟能转 120 次，旋转延迟就是1/120/2 = 4.17ms；传输时间指的是从磁盘读出或将数据写入磁盘的时间，一般在零点几毫秒，相对于前两个时间可以忽略不计。那么访问一次磁盘的时间，即一次磁盘 IO 的时间约等于 5+4.17 = 9ms 左右，听起来还挺不错的，但要知道一台 500 -MIPS 的机器每秒可以执行 5 亿条指令，因为指令依靠的是电的性质，换句话说执行一次 IO 的时间可以执行 40 万条指令，数据库动辄十万百万乃至千万级数据，每次 9 毫秒的时间，显然是个灾难。下图是计算机硬件延迟的对比图，供大家参考：

![Image](https://mmbiz.qpic.cn/mmbiz_png/80icw67Ot0qJsEtic2EMiasxzVhLYQmMgcQWxKsKAcboAaUibicNWBPXqEVHB0pOAjvGR5FYGRM9p3xiaib85znWL2kSg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1 "在这里插入图片描述")

考虑到磁盘 IO 是非常高昂的操作，计算机操作系统做了一些优化，当一次 IO 时，不光把当前磁盘地址的数据，而是把相邻的数据也都读取到内存缓冲区内，因为局部预读性原理告诉我们，当计算机访问一个地址的数据的时候，与其相邻的数据也会很快被访问到。每一次 IO 读取的数据我们称之为一页（page）。具体一页有多大数据跟操作系统有关，一般为 4k 或 8k，也就是我们读取一页内的数据时候，实际上才发生了一次 IO，这个理论对于索引的数据结构设计非常有帮助。

### 5.2 B-/+ Tree索引的性能分析

上文说过一般使用磁盘 I/O 次数评价索引结构的优劣。先从 B-Tree 分析，根据 B-Tree 的定义，可知检索一次最多需要访问 h 个节点。数据库系统的设计者巧妙利用了磁盘预读原理，将一个节点的大小设为等于一个页，这样每个节点只需要一次 I/O 就可以完全载入。为了达到这个目的，在实际实现 B-Tree 还需要使用如下技巧：

每次新建节点时，直接申请一个页的空间，这样就保证一个节点物理上也存储在一个页里，加之计算机存储分配都是按页对齐的，就实现了一个 node 只需一次 I/O。

B-Tree 中一次检索最多需要 h-1 次 I/O（根节点常驻内存），渐进复杂度O(h)=O(logdN)O(h)=O(logdN)。一般实际应用中，出度 d 是非常大的数字，通常超过 100，因此 h 非常小（通常不超过 3 ）。（h 表示树的高度 & 出度 d 表示的是树的度，即树中各个节点的度的最大值）

综上所述，用 B-Tree 作为索引结构效率是非常高的。

而红黑树这种结构，h 明显要深的多。由于逻辑上很近的节点（父子）物理上可能很远，无法利用局部性，所以红黑树的 I/O 渐进复杂度也为 O(h)，效率明显比 B-Tree 差很多。

上文还说过，B+Tree 更适合外存索引，原因和内节点出度 d 有关。从上面分析可以看到，d 越大索引的性能越好，而出度的上限取决于节点内 key 和 data 的大小：

```java
dmax=floor(pagesize/(keysize+datasize+pointsize))
```

floor 表示向下取整。由于 B+Tree 内节点去掉了 data 域，因此可以拥有更大的出度，拥有更好的性能。

## 六、MySQL 的存储引擎

MyISAM 和 InnoDB 存储引擎虽然索引结构都是 B+Tree，但实现上还是有些差别，我们一起来看下：

### 6.1 MyISAM

MyISAM 引擎使用 B+Tree 作为索引结构，叶节点的 data 域存放的是数据记录的地址。下图是 MyISAM 索引的原理图：

![Image](https://mmbiz.qpic.cn/mmbiz_jpg/80icw67Ot0qJsEtic2EMiasxzVhLYQmMgcQqPVV1MJTc8QN69DzM9tibEFWiaECdFxSXxz9C96ia6m6ng1jTMz9y7ibrg/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1 "> 这里是引用")

这里设表一共有三列，假设我们以 Col1 为主键，则上图是一个 MyISAM 表的主索引（Primary key）示意。可以看出 MyISAM 的索引文件仅仅保存数据记录的地址。在 MyISAM 中，主索引和辅助索引（Secondary key）在结构上没有任何区别，只是主索引要求 key 是唯一的，而辅助索引的 key 可以重复。如果我们在 Col2 上建立一个辅助索引，则此索引的结构如下图所示：

![Image](https://mmbiz.qpic.cn/mmbiz_jpg/80icw67Ot0qJsEtic2EMiasxzVhLYQmMgcQXUIDJ0BdNicabg5FrwbDI9NKf7AcJOyWywXCYBTH0u5GyORrjSMQkGg/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1 "在这里插入图片描述")

同样是一棵 B+ 树，data 域保存数据记录的地址。因此，MyISAM 中索引检索的算法为首先按照 B+Tree 搜索算法搜索索引，如果指定的 Key 存在，则取出其 data 域的值，然后以 data 域的值为地址，读取相应数据记录。

MyISAM 的索引方式也叫做“非聚集”的，之所以这么称呼是为了与 InnoDB 的聚集索引区分。

### 6.2 InnoDB

虽然 InnoDB 也使用 B+Tree 作为索引结构，但具体实现方式却与 MyISAM 截然不同。

第一个重大区别是 InnoDB 的数据文件本身就是索引文件。从上文知道，MyISAM 索引文件和数据文件是分离的，索引文件仅保存数据记录的地址。而在 InnoDB 中，表数据文件本身就是按 B+Tree 组织的一个索引结构，这棵树的叶节点 data 域保存了完整的数据记录。这个索引的 key 是数据表的主键，因此 InnoDB 表数据文件本身就是主索引。

![Image](https://mmbiz.qpic.cn/mmbiz_png/80icw67Ot0qJsEtic2EMiasxzVhLYQmMgcQfgibDDKCzSKymzjSmVkrbecFxv6lYxKEelhT3tSC0B71LX5FnerYnBA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1 "> 这里是引用")

上图是 InnoDB 主索引（同时也是数据文件）的示意图，可以看到叶节点包含了完整的数据记录。这种索引叫做聚集索引。因为 InnoDB 的数据文件本身要按主键聚集，所以 InnoDB 要求表必须有主键（MyISAM 可以没有），如果没有显式指定，则 MySQL 系统会自动选择一个可以唯一标识数据记录的列作为主键，如果不存在这种列，则 MySQL 自动为 InnoDB 表生成一个隐含字段作为主键，这个字段长度为 6 个字节，类型为长整型。

第二个与 MyISAM 索引的不同是 InnoDB 的辅助索引 data 域存储相应记录主键的值而不是地址。换句话说，InnoDB 的所有辅助索引都引用主键作为 data 域。例如，下图为定义在 Col3 上的一个辅助索引：

![Image](https://mmbiz.qpic.cn/mmbiz_png/80icw67Ot0qJsEtic2EMiasxzVhLYQmMgcQe1exouM77R8icianXCHqzW9TliaqMD2GBketXTwRGnmqHrAMiaZNAVfJ1g/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1 "> 这里是引用")

这里以英文字符的 ASCII 码作为比较准则。聚集索引这种实现方式使得按主键的搜索十分高效，但是辅助索引搜索需要检索两遍索引：首先检索辅助索引获得主键，然后用主键到主索引中检索获得记录。

### 6.3 问题

这里你有可能会问了，为什么 InnoDB 只在主键索引树的叶子节点存储了具体数据，但是其他索引树却不存具体数据呢，而要多此一举先找到主键，再在主键索引树找到对应的数据呢?

其实很简单，因为 InnoDB 需要节省存储空间。一个表里可能有很多个索引，InnoDB 都会给每个加了索引的字段生成索引树，如果每个字段的索引树都存储了具体数据，那么这个表的索引数据文件就变得非常巨大（数据极度冗余了）。从节约磁盘空间的角度来说，真的没有必要每个字段索引树都存具体数据，通过这种看似“多此一举”的步骤，在牺牲较少查询的性能下节省了巨大的磁盘空间，这是非常有值得的。

### 6.4 MyISAM 与 InnoDB 对比

- InnoDB 支持事务，MyISAM 不支持事务。这是 MySQL 将默认存储引擎从 MyISAM 变成 InnoDB 的重要原因之一。

- InnoDB 支持外键，而 MyISAM 不支持。对一个包含外键的 InnoDB 表转为 MYISAM 会失败。

- InnoDB 是聚集索引，MyISAM 是非聚集索引。聚簇索引的文件存放在主键索引的叶子节点上，因此 InnoDB 必须要有主键，通过主键索引效率很高。但是辅助索引需要两次查询，先查询到主键，然后再通过主键查询到数据。因此，主键不应该过大，因为主键太大，其他索引也都会很大。而 MyISAM 是非聚集索引，数据文件是分离的，索引保存的是数据文件的指针。主键索引和辅助索引是独立的。

- InnoDB 不保存表的具体行数，执行 select count(\*) from table 时需要全表扫描。而 MyISAM 用一个变量保存了整个表的行数，执行上述语句时只需要读出该变量即可，速度很快。

- InnoDB 最小的锁粒度是行锁，MyISAM 最小的锁粒度是表锁。一个更新语句会锁住整张表，导致其他查询和更新都会被阻塞，因此并发访问受限。这也是 MySQL 将默认存储引擎从 MyISAM 变成 InnoDB 的重要原因之一。
