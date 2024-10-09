腾讯程序员 腾讯技术工程

_2022年09月01日 18:00_ _广东_

![图片](https://mmbiz.qpic.cn/mmbiz_gif/j3gficicyOvasIjZpiaTNIPReJVWEJf7UGpmokI3LL4NbQDb8fO48fYROmYPXUhXFN8IdDqPcI1gA6OfSLsQHxB4w/640?wx_fmt=gif&wxfrom=13&tp=wxpic)

作者：phongchen，腾讯 IEG 后台开发工程师

> 2022 年 3 月，广大人民期盼已久的支持的泛型的 go1.18 发布了。但是目前基于泛型的容器实现还不多。我实现了一套类似 C++中 STL 的容器和算法库。其中有序的 Map 选择用跳表来实现，并优化到了相当好的性能。在此分享一下优化的思路和心得，供大家参考借鉴，如果发现有错误也欢迎指出。

### 一、背景

首先为标题党致歉，不过确实没吹牛 😊。

最近一年我所负责的业务系统中，用 Go 语言的实现的占了至少 70%的比例，因此 Review 了大量的 Go 代码，也看了很多相关的技术资料。但是我确实没怎么写过 Go 的代码，因为一直以来我不太喜欢 Go 语言的主要有两点，一个是错误处理，另一个就是泛型。因此先前还是以写 C++和 Python 代码为主，再加上一些 Markdown 文档什么的。

2022 年 3 月，Go1.18 发布，支持了泛型，我也打算自己一边学习，一边写一些有实际价值的 Go 代码。

前几周孩子放假回老家，家里没人打扰了，调研了一下有没有类似 C++中 STL 的泛型库，发现要么很薄弱要么根本就不支持泛型。于是就花了几个周末和一些晚上的时间，写了个基于泛型的容器和算法库，暂且起名叫[stl4go](https://github.com/chen3feng/stl4go)（👏 加 ⭐，🙏）。其中的有序 Map 我没有选择红黑树而是用了跳表，花了一些时间用了一些手法优化，测试了一下，基本上可以说是全 GitHub 上能找到的最快的 Go 的实现了。

### 二、跳表是什么

跳表（[skiplist](http://en.wikipedia.org/wiki/Skip_list)）是一种随机化的数据， 由 William Pugh 在论文[《Skip lists: a probabilistic alternative to balanced trees》](http://www.cl.cam.ac.uk/teaching/0506/Algorithms/skiplists.pdf)中提出， 跳表以有序的方式在层次化的链表中保存元素， 效率和平衡树媲美 —— 查找、删除、添加等操作都可以在 O(logN)期望时间下完成， 综合能力相当于平衡二叉树，并且比起平衡树来说， 跳跃表的实现要简单直观得多，核心功能在 200 行以内即可实现，遍历的时间复杂度是 O(N)，代码简单，空间上也比较节省，因此在挺多的场景得到应用。比如[Redis 的 Sorted Set](https://redis.io/docs/data-types/sorted-sets/)、[LevelDB](https://github.com/google/leveldb/blob/main/db/skiplist.h)，详细原理和算法请移步下面这篇文章：[Skip List--跳表（全网最详细的跳表文章没有之一）](https://www.jianshu.com/p/9d8296562806)，不再赘述。

完整代码见：\
[https://github.com/chen3feng/stl4go/blob/master/skiplist.go](https://github.com/chen3feng/stl4go/blob/master/skiplist.go)

附带单元测试和性能测试。

SkipList 用于需要有序的场合，在不需要有序的场景下，go 自带的 map 容器依然是优先选择。

### 三、接口设计

#### （一）创建函数

`主要考虑可以用 <、== 比较的类型，对一不可以的，需要支持自定义的比较函数。// 对于Key可以用 < 和 == 运算符比较的类型，调这个函数来创建   func NewSkipList[K Ordered, V any]() *SkipList[K, V]      // 其他情况，需要自定义Key类型的比较函数   func NewSkipListFunc[K any, V any](keyCmp CompareFn[K]) *SkipList[K, V]      // 从一个map来构建，仅为方便写Literal，go没法对自定义类型使用初始化列表。   func NewSkipListFromMap[K Ordered, V any](m map[K]V) *SkipList[K, V]   `

#### （二）主要方法

`IsEmpty() bool // 表是否为空   Len() int // 返回表中元素的个数   Clear() // 清空跳表   Has(K) bool // 检查跳表中是否存在指定的key   Find(K) *V // Finds element with specific key.   Insert(K, V) // Inserts a key-value pair in to the container or replace existing value.   Remove(K) bool // Remove element with specific key.   `

还有迭代器和遍历区间查找等功能与本主题关系不大略去。

可以看得出，完全可以满足有序 Map 容器的要求。

#### （三）节点定义

虽然不少讲跳表原理示意图会把每层的索引节点单独列出来：

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

出处：[Skip List--跳表（全网最详细的跳表文章没有之一）](https://www.jianshu.com/p/9d8296562806)

但是一般的实现都会把索引节点实现为最底层节点的一个数组，这样每个元素只需要一个节点，节省了单独的索引节点的存储开销，也提高了性能。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

因此节点定义如下：

`type skipListNode[K any, V any] struct {       key K       value V       // 指向下一个节点的指针的数组，不同深度的节点长度不同，[0]表示最下层       next []*skipListNode[K, V]   }   `

### 四、代码优化

代码并非完全从头开始写的，我是以 liyue201@github.com 的 gostl 的实现为基础的。

https://github.com/liyue201/gostl/blob/master/ds/skiplist/skiplist.go

这个实现比较简洁，只有 200 多行代码，支持自定义数据类型比较，但是不支持泛型。

我在他的基础上做了一系列的算法和内存分配等方面的优化，并增加了迭代器、区间查找等功能。

#### （一）算法优化

##### 1.生成随机 Level 的优化

每次跳表插入元素时，需要随机生成一个本次的层数，最朴素的实现方式是抛硬币：

`func randomLevel() int {       level := 0       for math.Float64() < 0.5 {           level++       }       return level   }   `

也就是根据连续获得正面的次数来决定层数。

Redis 里的算法类似，只不过用的是 1/4 的级数差，索引少一半，可以节省一些内存：

https://github.com/redis/redis/blob/7.0/src/t_zset.c#L118-L128

`/* Returns a random level for the new skiplist node we are going to create.    * The return value of this function is between 1 and ZSKIPLIST_MAXLEVEL    * (both inclusive), with a powerlaw-alike distribution where higher    * levels are less likely to be returned. */   int zslRandomLevel(void) {       static const int threshold = ZSKIPLIST_P*RAND_MAX;       int level = 1;       while (random() < threshold)           level += 1;       return (level<ZSKIPLIST_MAXLEVEL) ? level : ZSKIPLIST_MAXLEVEL;   }   `

简单直白。但是存在两个问题：

- math.Float64() （以及任何全局随机函数）内部为共享的随机数生成器对象，每次调用都会加锁解锁，在竞争情况下性能下降很厉害。详情参见源代码 https://cs.opensource.google/go/go/+/refs/tags/go1.19:src/math/rand/rand.go

- 多次生成随机数。下面我们将看到，其实只用生成一次就可以了。

所以在 gostl 的实现中，改用了生成一个某范围内的随机数，根据其均匀分布的特点，来计算 level：

`func (sl *Skiplist) randomLevel() int {       total := uint64(1)<<uint64(sl.maxLevel) - 1 // 2^n-1       k := sl.rander.Uint64() % total       levelN := uint64(1) << (uint64(sl.maxLevel) - 1)          level := 1       for total -= levelN; total > k; level++ {           levelN >>= 1           total -= levelN       }       return level   }   `

这些for循环有些拗口，改写一下就更清晰了：

```
level := 0for k < total {    levelN >>= 1    total -= levelN    level++    }
```

也就是生成的随机数

也就是生成的随机数越小，level 越高，比如 maxLevel 为 10 时，total=1023，那么：

- 512\<k\<1023 之间的概率为 1/2，level=1

- 256\<k\<511 之间的概率为 1/4，level=2

- 128\<k\<255 之间的概率为 1/8，level=3

- ...

当 level 比较高时，循环次数就会增加。不过可以观察到在生成的随机二进制中，数值增减一半正好等于改变一个 bit 位，因此我改用直接调用 math/bits 里的 Len64()函数来计算生成的随机数的最小位数的方式来实现：

`func (sl *SkipList[K, V]) randomLevel() int {       total := uint64(1)<<uint64(skipListMaxLevel) - 1 // 2^n-1       k := sl.rander.Uint64() % total       return skipListMaxLevel - bits.Len64(k) + 1   }   `

而 Len64 函数是用查表实现的，相当的快：

https://github.com/golang/go/blob/go1.19/src/math/bits/bits.go#L330-L345

`// Len64 returns the minimum number of bits required to represent x;   // the result is 0 for x == 0.   func Len64(x uint64) (n int) {       // ...       return n + int(len8tab[x])   }   `

这样当 level>1 时，时间开销就从循环变成固定开销，会快一点点。

##### **2.自适应 Level**

很多实现都把 level 硬编码成全局或者实例级别的常量，比如在 gostl 中就是如此。

sl.maxLevel 是一个实例级别的固定常量，跳表创建后便不再修改，因此有两个问题：

- 当实际元素很少时，查找函数中循环的前几次 cur 变量基本上都是空指针，白白浪费时间查找，所以他的实现里 defaultMaxLevel 设置的很小。

- 由于默认的 maxLevel 很小，只有 10，插入 1024 个元素后，最上层基本上就接近平衡二叉树的情况了，如果再继续插入大量的元素，每层索引节点数量都快速增加，性能急剧下降。如果在构造时就根据预估容量设置一个足够大的 maxLevel 既可避免这个问题，但是很多时候这个数不是那么好预估，而且用起来不方便，漏了设置又可能会导致意料之外的性能恶化。

因此我把 level 设计为根据元素的个数动态自适应调整：

- 设置一个 level 成员记录最高的 level 值

- 当插入元素时，如果出现了更高的层，再插入后就调大 level

- 当删除元素时，如果最顶层的索引变空了，就减少 level。

通过这种方式，就解决了上述问题。

`// Insert inserts a key-value pair into the skiplist.   // If the key is already in the skip list, it's value will be updated.   func (sl *SkipList[K, V]) Insert(key K, value V) {       // 处理key已存在的情况，略去          level := sl.randomLevel()       node = newSkipListNode(level, key, value)          // 插入链表，略去          if level > sl.level {           // Increase the level           for i := sl.level; i < level; i++ {               sl.head.next[i] = node           }           sl.level = level       }       sl.len++   }   `

另外为了防止万一在一开始元素个数很小时就生成了很大的随机 level，在 randomLevel 里做了一下限制，最大允许生成的 level 为 log2(Len())+2。2 是个拍脑袋决定的余量。

##### 3.插入删除优化

插入时如果 key 不存在或者删除时节点存在，都需要找到每层索引中的前一个节点，放入 prevs 数组返回，用于插入或者删除节点后各层链表的重新组织。

gostl 的实现中，是先在 findPrevNodes 函数里的循环中得到所有的 prevs，然后再比较\[0\]层的值来判断 key 是否相等决定更新或者返回。

[https://github.com/liyue201/gostl/blob/e5590f19a43ac53f35893c7c679b37d967c4859c/ds/skiplist/skiplist.go#L186-L201](https://github.com/liyue201/gostl/blob/e5590f19a43ac53f35893c7c679b37d967c4859c/ds/skiplist/skiplist.go#L186-L201)

这个函数会从顶层遍历到最底层：

`func (sl *Skiplist) findPrevNodes(key interface{}) []*Node {       prevs := sl.prevNodesCache       prev := &sl.head       for i := sl.maxLevel - 1; i >= 0; i-- {           if sl.head.next[i] != nil {               for next := prev.next[i]; next != nil; next = next.next[i] {                   if sl.keyCmp(next.key, key) >= 0 {                       break                   }                   prev = &next.Node               }           }           prevs[i] = prev       }       return prevs   }   `

插入时再取最底层的节点的下一个进一步比较是否相等

`// Insert inserts a key-value pair into the skiplist   func (sl *Skiplist) Insert(key, value interface{}) {       prevs := sl.findPrevNodes(key)          if prevs[0].next[0] != nil && sl.keyCmp(prevs[0].next[0].key, key) == 0 {           // 如果相等，其实prevs就没用了，但是findPrevNodes里依然进行了查询           // same key, update value           prevs[0].next[0].value = value           return       }       ...   }   `

但是再插入 key 时如果已经节点存在，或者删除 key 时节点不存在，是不需要调整每层节点的，前面辛辛苦苦查找的 prevs 就没用了。

我在这里做了个优化，在这种情况下提前返回，不再继续找所有的 prevs。以插入为例：

`// findInsertPoint returns (*node, nil) to the existed node if the key exists,   // or (nil, []*node) to the previous nodes if the key doesn't exist   func (sl *skipListOrdered[K, V]) findInsertPoint(key K) (*skipListNode[K, V], []*skipListNode[K, V]) {       prevs := sl.prevsCache[0:sl.level]       prev := &sl.head       for i := sl.level - 1; i >= 0; i-- {           for next := prev.next[i]; next != nil; next = next.next[i] {               if next.key == key {                   // Key 已经存在，停止搜索                   return next, nil               }               if next.key > key {                   // All other node in this level must be greater than the key,                   // search the next level.                   break               }               prev = next           }           prevs[i] = prev       }       return nil, prevs   }   `

node和prevs只会有一个不空：

`// Insert inserts a key-value pair into the skiplist.   // If the key is already in the skip list, it's value will be updated.   func (sl *SkipList[K, V]) Insert(key K, value V) {       node, prevs := sl.impl.findInsertPoint(key)       if node != nil {           // Already exist, update the value           node.value = value           return       }       // 生成及插入新节点，略去   }`

删除操作的优化方式类似，不再赘述。

##### **4.数据类型特有的优化**

对于 Ordered 类型的跳表，如果是升序的，可以直接用 NewSkipList 来创建。对于用得较少的降序或者 Key 是不可比较的类型，就需要通过传入的比较函数来比较 Key。

一开始的实现为了简化，对于 Ordered 的 SkipList，内部是通过调用 SkipListFunc 来实现的，这样可以节省不少代码，实现起来很简单。

但是 Benchmark 时，跑不过一些较快地实现。分析主要原因就在比较函数的函数调用上。以查找为例：

`// Get returns the value associated with the passed key if the key is in the skiplist, otherwise returns nil   func (sl *Skiplist) Get(key interface{}) interface{} {       var pre = &sl.head       for i := sl.maxLevel - 1; i >= 0; i-- {           cur := pre.next[i]           for ; cur != nil; cur = cur.next[i] {               cmpRet := sl.keyCmp(cur.key, key)               if cmpRet == 0 {                   return cur.value               }               if cmpRet > 0 {                   break               }               pre = &cur.Node           }       }       return nil   }   `

在 C++中，比较函数可以是无状态的函数对象，其()运算符是可以 inline 的。但是在 Go 中，比较函数只能是函数指针，sl.keyCmp 调用无法被 inline。因此对简单的类型，这部分开销占的比例很大。

我一开始用的优化手法是在运行期间，根据硬编码的 key 的类型，进行类型转换后调优化的实现

https://github.com/chen3feng/stl4go/commit/1d444f1530cc43c99a978dcf0b1d7f83bcb575ee#diff-37795d4525025da36a8f77e3e5d0b3f6593fd121960e1d563008a6700fb08473

这种方式虽然凑效但是代码很丑陋，用到了硬编码的类型列表，运行期类型 switch 等机制，甚至还需要代码生成。

后来我摸索出通过同一个接口，根据 Key 是作为 Ordered 还是通过 Func 的方式来比较，来提供了不同实现的方式，就更优雅地解决了这个问题，不需要任何强制类型转换：

`type skipListImpl[K any, V any] interface {       findNode(key K) *skipListNode[K, V]       lowerBound(key K) *skipListNode[K, V]       upperBound(key K) *skipListNode[K, V]       findInsertPoint(key K) (*skipListNode[K, V], []*skipListNode[K, V])       findRemovePoint(key K) (*skipListNode[K, V], []*skipListNode[K, V])   }      // NewSkipList creates a new SkipList for Ordered key type.   func NewSkipList[K Ordered, V any]() *SkipList[K, V] {       sl := skipListOrdered[K, V]{}       sl.init()       sl.impl = (skipListImpl[K, V])(&sl)       return &sl.SkipList   }      // NewSkipListFunc creates a new SkipList with specified compare function keyCmp.   func NewSkipListFunc[K any, V any](keyCmp CompareFn[K]) *SkipList[K, V] {       sl := skipListFunc[K, V]{}       sl.init()       sl.keyCmp = keyCmp       sl.impl = skipListImpl[K, V](&sl)       return &sl.SkipList   }   `

对于 Len()、IsEmpty()等，则不放进接口里，有利于编译器 inline 优化。

#### （二）内存分配优化

无论是理论上还是实测，内存分配对性能的影响还是挺大的。Go 不像 Java 和 C#的堆内存分配那么简单，因此应当减少不必要的内存分配。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

来源：[Go 生态下的字节跳动大规模微服务性能优化实践](https://my.oschina.net/u/5632822/blog/5565821)

##### 1.Cache Prevs 节点

在插入时如果节点先前不存在，或者删除时节点存在，那么就需要获得所有层的指向该位置的节点数组，这倒不是我原创的，因为看到的好几个实现中都采用了在 SkipList 实例级别预先分配一个 slice 的办法，经测试比起每次都创建 slice 返回确实有相当明显的性能提升。

##### 2.节点分配优化

不同 level 的节点数据类型是相同的，但是其 next 指针数组的长度不同，一些简单粗暴的实现是设置为固定的最大深度，由于跳表中绝大多数节点都只落在最低几层，浪费了较多的内存。

另外一种做法是改为动态分配，那么就多一次内存分配。

我的做法是根据不同的深度，定义不同的结构体，额外包含一个相应长度的 nexts 节点指针数组，然后在 node 的 next 切片指向这个数组，可以就减少一次内存分配。并且由于 nexts 数组和 node 的地址是在一起的，cache 局部性也更好。

https://github.com/chen3feng/stl4go/blob/master/skiplist_newnode.go

`// newSkipListNode creates a new node initialized with specified key, value and next slice.   func newSkipListNode[K any, V any](level int, key K, value V) *skipListNode[K, V] {       switch level {       case 1:           n := struct {               head skipListNode[K, V]               nexts [1]*skipListNode[K, V]           }{head: skipListNode[K, V]{key, value, nil}}           n.head.next = n.nexts[:]           return &n.head       case 2:           n := struct {               head skipListNode[K, V]               nexts [2]*skipListNode[K, V]           }{head: skipListNode[K, V]{key, value, nil}}           n.head.next = n.nexts[:]           return &n.head       // 一直到 case 40 ...       }   }   `

这么多啰嗦的代码显然不适合手写，是弄个 bash 脚本通过 go generate 生成的。

https://github.com/chen3feng/stl4go/blob/master/skiplist_newnode_generate.sh

另外我在调试这段代码时发现 go 的 switch case 语句即使对简单的全数值居然也是通过[二分法](https://github.com/golang/go/blob/8ee9bca2729ead81da6bf5a18b87767ff396d1b7/src/cmd/compile/internal/gc/swt.go#L375)而非 C++常用的[跳转表](https://sites.google.com/site/arch1utep/jump-tables)来实现的。不过估计是因为有更耗时的内存分配的原因，尝试把 case 1，2 等单独拿出来也没有提升，因此估计这里对性能没有影响。如果 case 非常多的话可以考虑对最常见的 case 单独处理，或者用函数指针数组来优化。

### 五、C++实现

类似的代码在 C++中由于支持模板非类型参数，可以简单不少：

`template <typename K, typename V>   struct Node {     K key;     V value;     size_t level;     Node* nexts[0];     SkipListNode(key, V value) : level(level), key(std::move(key)), value(std::move(value)) {}   };      template <typename K, typename V, int N> // 注意 N 可以作为模板参数   struct NodeN : public Node {     NodeN(K key, V value) : Node(N, key, value) {}     Node* nexts[N] = {};   };      Node* NewNode(int level, K key, V value) {     switch (level) {       case 1: return new NodeN<K, V, 1>(key, value);       case 2: return new NodeN<K, V, 2>(key, value);       case 3: return new NodeN<K, V, 3>(key, value);       ...     }   }   `

用 C（当然在 C++中也可以用）的[flexible array](https://en.wikipedia.org/wiki/Flexible_array_member)代码则会更简单一些：

`Node* NewNode(int level, K key, V value) {     auto p = malloc(sizeof(Node*) + level * sizeof(Node*));     return new(p) Node(std::move(key), std::move(value));   }   `

而且由于 C 和 C++中的 next 数组不需要通过切片（相当于指针）来指向 nexts 数组，少了一次内存寻址，理论上性能更好一些。

C++实现为 Go 代码的手工转译，功能未做充分的验证，仅供对比评测，代码在：

[https://github.com/chen3feng/skiplist-survey/tree/master/skiplist](https://github.com/chen3feng/skiplist-survey/tree/master/skiplist)

### 六、Benchmark

sean-public@github.com 实现了一个以 float64 为 key 的跳表：\
[https://github.com/sean-public/fast-skiplist](https://github.com/sean-public/fast-skiplist)

并和其他实现做了个比较证明自己的最快：

https://github.com/sean-public/skiplist-survey

我在他的基础上添加了一些其他的实现和我的实现，做了 benchmark，上述优化的数据类型优化就是基于此评测结果做的。

[https://github.com/chen3feng/skiplist-survey](https://github.com/chen3feng/skiplist-survey)

以下是部分评测结果，数值越小越好：

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

虽然也有少量指标不是最快的，但是总体上在大部分指标上，超越了我在 github 上找到的其他实现。并且大部分其他实现 key 只支持 int64 或者 float64，使得无法用于 string 等类型。

另外也对 C++的实现测了一下性能：

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

发现 Go 的实现性能很多指标基本接近 C++，其中 Delete 反而更快一些，是因为 C++在删除时要析构节点并释放内存，而 Go 采用 GC 的方式延后旁路处理。

### 七、心得

1）go1.18 引入的泛型还可以，虽然功能上不算很强大，但是已经能满足挺大一部分的需求。我们组现在正在升级到 go1.19，很快就能用得上。\
2）Go 的开发生态还是不错的，github 上大量垂手可得的库，VS Code 高度集成，各种便利的工具，这是我写 C++代码很难体验到的。大部分优化是基于 benchmark test 来做的。\
3）很多编程语言需要的基础知识都是相通的，打好基础，学习新技术并不太难。\
4）跳出自己的舒适区，多学习一些编程语言开阔视野涨见识，有利于持续提高自己的技术能力。

**参考资料**

1. [Wikipedia -- Skip list](https://en.wikipedia.org/wiki/Skip_list)

1. [Go 生态下的字节跳动大规模微服务性能优化实践](https://my.oschina.net/u/5632822/blog/5565821)

1. [Skip List--跳表（全网最详细的跳表文章没有之一）](https://www.jianshu.com/p/9d8296562806)

1. https://github.com/liyue201/gostl/blob/master/ds/skiplist/skiplist.go

1. https://github.com/sean-public/skiplist-survey

**欢迎观看腾讯程序员最新视频**

因视频号作者隐私设置暂无法查看

腾讯程序员

阅读 7824

​

写留言

[](javacript:;)

![](http://mmbiz.qpic.cn/sz_mmbiz_png/j3gficicyOvauPPfL7J2AVERiaoMJy9NBIwbJE2ZRJX7FZ2Dx7IibtTwdlqYSqTZTCsXkDS2jvNF8wWJKcibxXtOHng/300?wx_fmt=png&wxfrom=18)

腾讯技术工程

691022

写留言

写留言

**留言**

暂无留言
