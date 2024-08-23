# 

原创 labuladong labuladong

 _2019年12月04日 08:11_

## 

预计阅读时间： 10 分钟

今天讲讲 Union-Find 算法，也就是常说的并查集算法，主要是解决图论中「动态连通性」问题的。名词很高端，其实特别好理解，等会解释，另外这个算法的应用都非常有趣。

说起这个 Union-Find，应该算是我的「启蒙算法」了，因为《算法4》的开头就介绍了这款算法，可是把我秀翻了，感觉好精妙啊！后来刷了 LeetCode，并查集相关的算法题目都非常有意思，而且《算法4》给的解法竟然还可以进一步优化，只要加一个微小的修改就可以把时间复杂度降到 O(1)。

废话不多说，直接上干货。先解释一下什么叫动态连通性吧。

### 一、问题介绍

简单说，动态连通性其实可以抽象成给一幅图连线。比如下面这幅图，总共有 10 个节点，他们互不相连，分别用 0~9 标记：

![图片](https://mmbiz.qpic.cn/sz_mmbiz_jpg/gibkIz0MVqdHbnHaPibsAQHPibgTF6OUYzMJRzibEJI9g7FobribRc864gkDegWlvzVATWKicEiavG4o0JI7Jra2edIXA/640?wx_fmt=jpeg&wxfrom=13)

现在我们的 Union-Find 算法主要需要实现这两个 API：  

```
class UF {    /* 将 p 和 q 连接 */    public void union(int p, int q);    /* 判断 p 和 q 是否连通 */    public boolean connected(int p, int q);    /* 返回图中有多少个连通分量 */    public int count();}
```

这里所说的「连通」是一种等价关系，也就是说具有如下三个性质：

**1、自反性**：节点`p`和`p`是连通的。

**2、对称性**：如果节点`p`和`q`连通，那么`q`和`p`也连通。

**3、传递性**：如果节点`p`和`q`连通，`q`和`r`连通，那么`p`和`r`也连通。

比如说之前那幅图，0～9 任意两个**不同**的点都不连通，调用`connected`都会返回 false，连通分量为 10 个。

如果现在调用`union(0, 1)`，那么 0 和 1 被连通，连通分量降为 9 个。

再调用`union(1, 2)`，这时 0,1,2 都被连通，调用`connected(0, 2)`也会返回 true，连通分量变为 8 个。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

判断这种「等价关系」非常实用，比如说编译器判断同一个变量的不同引用，比如社交网络中的朋友圈计算等等。  

这样，你应该大概明白什么是动态连通性了，Union-Find 算法的关键就在于`union`和`connected`函数的效率。那么用什么模型来表示这幅图的连通状态呢？用什么数据结构来实现代码呢？

### 二、基本思路

注意我刚才把「模型」和具体的「数据结构」分开说，这么做是有原因的。因为我们使用森林（若干棵树）来表示图的动态连通性，用数组来具体实现这个森林。

怎么用森林来表示连通性呢？我们设定树的每个节点有一个指针指向其父节点，如果是根节点的话，这个指针指向自己。

比如说刚才那幅 10 个节点的图，一开始的时候没有相互连通，就是这样：

```
class UF {    // 记录连通分量    private int count;    // 节点 x 的节点是 parent[x]    private int[] parent;    /* 构造函数，n 为图的节点总数 */    public UF(int n) {        // 一开始互不连通        this.count = n;        // 父节点指针初始指向自己        parent = new int[n];        for (int i = 0; i < n; i++)            parent[i] = i;    }    /* 其他函数 */}
```

**![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)**

**如果某两个节点被连通，则让其中的（任意）一个节点的根节点接到另一个节点的根节点上**：

```
public void union(int p, int q) {    int rootP = find(p);    int rootQ = find(q);    if (rootP == rootQ)        return;    // 将两棵树合并为一棵    parent[rootP] = rootQ;    // parent[rootQ] = rootP 也一样    count--; // 两个分量合二为一}/* 返回某个节点 x 的根节点 */private int find(int x) {    // 根节点的 parent[x] == x    while (parent[x] != x)        x = parent[x];    return x;}/* 返回当前的连通分量个数 */public int count() {     return count;}
```

**![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)**

**这样，如果节点`p`和`q`连通的话，它们一定拥有相同的根节点**：

```
public boolean connected(int p, int q) {    int rootP = find(p);    int rootQ = find(q);    return rootP == rootQ;}
```

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

至此，Union-Find 算法就基本完成了。是不是很神奇？竟然可以这样使用数组来模拟出一个森林，如此巧妙的解决这个比较复杂的问题！

那么这个算法的复杂度是多少呢？我们发现，主要 API`connected`和`union`中的复杂度都是`find`函数造成的，所以说它们的复杂度和`find`一样。

`find`主要功能就是从某个节点向上遍历到树根，其时间复杂度就是树的高度。我们可能习惯性地认为树的高度就是`logN`，但这并不一定。**`logN`的高度只存在于平衡二叉树，对于一般的树可能出现极端不平衡的情况，使得「树」几乎退化成「链表」，树的高度最坏情况下可能变成`N`。**

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

所以说上面这种解法，`find`,`union`,`connected`的时间复杂度都是 O(N)。这个复杂度很不理想的，你想图论解决的都是诸如社交网络这样数据规模巨大的问题，对于`union`和`connected`的调用非常频繁，每次调用需要线性时间完全不可忍受。  

**问题的关键在于，如何想办法避免树的不平衡呢**？只需要略施小计即可。

### 三、平衡性优化

我们要知道哪种情况下可能出现不平衡现象，关键在于`union`过程：

```
public void union(int p, int q) {    int rootP = find(p);    int rootQ = find(q);    if (rootP == rootQ)        return;    // 将两棵树合并为一棵    parent[rootP] = rootQ;    // parent[rootQ] = rootP 也可以    count--; 
```

我们一开始就是简单粗暴的把`p`所在的树接到`q`所在的树的根节点下面，那么这里就可能出现「头重脚轻」的不平衡状况，比如下面这种局面：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

长此以往，树可能生长得很不平衡。**我们其实是希望，小一些的树接到大一些的树下面，这样就能避免头重脚轻，更平衡一些**。解决方法是额外使用一个`size`数组，记录每棵树包含的节点数，我们不妨称为「重量」：  

```
class UF {    private int count;    private int[] parent;    // 新增一个数组记录树的“重量”    private int[] size;    public UF(int n) {        this.count = n;        parent = new int[n];        // 最初每棵树只有一个节点        // 重量应该初始化 1        size = new int[n];        for (int i = 0; i < n; i++) {            parent[i] = i;            size[i] = 1;        }    }    /* 其他函数 */}
```

比如说`size[3] = 5`表示，以节点`3`为根的那棵树，总共有`5`个节点。这样我们可以修改一下`union`方法：

```
public void union(int p, int q) {    int rootP = find(p);    int rootQ = find(q);    if (rootP == rootQ)        return;    // 小树接到大树下面，较平衡    if (size[rootP] > size[rootQ]) {        parent[rootQ] = rootP;        size[rootP] += size[rootQ];    } else {        parent[rootP] = rootQ;        size[rootQ] += size[rootP];    }    count--;}
```

这样，通过比较树的重量，就可以保证树的生长相对平衡，树的高度大致在`logN`这个数量级，极大提升执行效率。

此时，`find`,`union`,`connected`的时间复杂度都下降为 O(logN)，即便数据规模上亿，所需时间也非常少。

### 四、路径压缩

这步优化特别简单，所以非常巧妙。我们能不能进一步压缩每棵树的高度，使树高始终保持为常数？

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

这样`find`就能以 O(1) 的时间找到某一节点的根节点，相应的，`connected`和`union`复杂度都下降为 O(1)。  

要做到这一点，非常简单，只需要在`find`中加一行代码：

```
private int find(int x) {    while (parent[x] != x) {        // 进行路径压缩        parent[x] = parent[parent[x]];        x = parent[x];    }    return x;}
```

这个操作有点匪夷所思，看个 GIF 就明白它的作用了（为清晰起见，这棵树比较极端）：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

可见，调用`find`函数每次向树根遍历的同时，顺手将树高缩短了，最终所有树高都不会超过 3（`union`的时候树高可能达到 3）。

PS：读者可能会问，这个 GIF 图的`find`过程完成之后，树高恰好等于 3 了，但是如果更高的树，压缩后高度依然会大于 3 呀？不能这么想。这个 GIF 的情景是我编出来方便大家理解路径压缩的，但是实际中，每次`find`都会进行路径压缩，所以树本来就不可能增长到这么高，你的这种担心应该是多余的。

### 五、最后总结

我们先来看一下完整代码：

```
class UF {    // 连通分量个数    private int count;    // 存储一棵树    private int[] parent;    // 记录树的“重量”    private int[] size;    public UF(int n) {        this.count = n;        parent = new int[n];        size = new int[n];        for (int i = 0; i < n; i++) {            parent[i] = i;            size[i] = 1;        }    }    public void union(int p, int q) {        int rootP = find(p);        int rootQ = find(q);        if (rootP == rootQ)            return;        // 小树接到大树下面，较平衡        if (size[rootP] > size[rootQ]) {            parent[rootQ] = rootP;            size[rootP] += size[rootQ];        } else {            parent[rootP] = rootQ;            size[rootQ] += size[rootP];        }        count--;    }    public boolean connected(int p, int q) {        int rootP = find(p);        int rootQ = find(q);        return rootP == rootQ;    }    private int find(int x) {        while (parent[x] != x) {            // 进行路径压缩            parent[x] = parent[parent[x]];            x = parent[x];        }        return x;    }}
```

Union-Find 算法的复杂度可以这样分析：构造函数初始化数据结构需要 O(N) 的时间和空间复杂度；连通两个节点`union`、判断两个节点的连通性`connected`、计算连通分量`count`所需的时间复杂度均为 O(1)。

至此，算法就说完了。后续可以考虑谈几道用到该算法的有趣问题，敬请期待。  

**历史文章：**

[动态规划详解（修订版）](http://mp.weixin.qq.com/s?__biz=MzAxODQxMDM0Mw==&mid=2247484731&idx=1&sn=f1db6dee2c8e70c42240aead9fd224e6&chksm=9bd7fb33aca07225bee0b23a911c30295e0b90f393af75eca377caa4598ffb203549e1768336&scene=21#wechat_redirect)  

[回溯算法详解（修订版）](http://mp.weixin.qq.com/s?__biz=MzAxODQxMDM0Mw==&mid=2247484709&idx=1&sn=1c24a5c41a5a255000532e83f38f2ce4&chksm=9bd7fb2daca0723be888b30345e2c5e64649fc31a00b05c27a0843f349e2dd9363338d0dac61&scene=21#wechat_redirect)  

[经典动态规划：高楼扔鸡蛋（进阶篇）](http://mp.weixin.qq.com/s?__biz=MzAxODQxMDM0Mw==&mid=2247484690&idx=1&sn=eea075701a5d96dd5c6e3dc6a993cac5&chksm=9bd7fb1aaca0720c58c9d9e02a8b9211a289bcea359633a95886d7808d2846898d489ce98078&scene=21#wechat_redirect)  

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

![](https://mmbiz.qlogo.cn/mmbiz_jpg/60qTNAlvA0LPRVHfPYG3EPW3kk1WnLbOtCicwubzggGh1gvac8p0jQao1ZgAkzpL8wcUflUMibTgsS9cmrict9tWA/0?wx_fmt=jpeg)

labuladong

 享受纯粹求知的乐趣 

![赞赏二维码](https://mp.weixin.qq.com/s?__biz=MzAxODQxMDM0Mw==&mid=2247484751&idx=1&sn=a873c1f51d601bac17f5078c408cc3f6&chksm=9bd7fb47aca07251dd9146e745b4cc5cfdbc527abe93767691732dfba166dfc02fbb7237ddbf&mpshare=1&scene=24&srcid=1018111iS0Nk2NCd4CJYgdvz&sharer_sharetime=1634519473473&sharer_shareid=5fb9813bfe9ffc983435bfc8d8c5e9ca&key=daf9bdc5abc4e8d0647a19c2715460a0035931387de07420eea775a3d7229c751008746da6c7fdc852f1d0a821ea3eecb697d67b729b162b723c6afe5cfbc3a2e3c03f2f465c01dff821af221227c8d924ff211e72f6e277c615daf609ebfb48c395b059dec7c0642da10b5a168ade09994f84fe82679948420214c9c9fcb68e&ascene=0&uin=MTEwNTU1MjgwMw%3D%3D&devicetype=Windows+11+x64&version=63090b19&lang=zh_CN&countrycode=CN&exportkey=n_ChQIAhIQ2FmGC4SRoPJQmzNbZG%2BophLmAQIE97dBBAEAAAAAAC5aCgbU%2B18AAAAOpnltbLcz9gKNyK89dVj0%2BAkGuR0SRd6P24patqXcaM%2FkuSI2uj6%2FDQeG4lexBiNPa33Xxxi8WRNzCWkyAaIyt3UxTBecOqomjhEsa3lyR4Sov4gR02ExRtDJArQQVJjno7ldKaQYkEahOS8378ujWjlDAQfc7ZosEoPh82%2BZRjvW4VYLjd3FKa2BVix4u3dKBX31bERPaTXDif33ZJ%2BBA9yYaSfoIAuJStQux8s%2Be6%2Fc4JFLvsYZ0blL8Mi9%2F29Tonkc1yeFZlhCftF1O45l&acctmode=0&pass_ticket=FSaxrgwTZXN9MzCuj1RWR8Uc0%2Fn%2F404yYvmJuCUjzg61IjVOhzFOVPtHXFizl5E4&wx_header=1&fasttmpl_type=0&fasttmpl_fullversion=7351805-zh_CN-zip&fasttmpl_flag=1)稀罕作者

15人喜欢

![](http://wx.qlogo.cn/mmopen/8YSWYKpVxlgZzSxzalmcVuJ7k77dZh8DOxK60eqR9yLbOmukkgjrFyianpazV2HqtGHGn5ia5Ue1XHibv0L258Fg0auEaB9nZD5/64)![](http://wx.qlogo.cn/mmopen/YCOL3hU8ffU2KW5dSDVIqk2oWP7d9jzme2lv6nrPT1F5cQialCJG5A3VPuuUe54hu4b2V8zTRWibYGrpGm3hVI7Sib2esePhz5F/64)![](http://wx.qlogo.cn/mmopen/PiajxSqBRaEIexnn88T8g9rTxWuaGolfKlWsQ7OsJVqCNvIF6O5rEibtgp9WGOYjnusNR4LEoddtdhMYaJC9Xjf9dLB4XOLaf9xHrbhJHsBTgd1Jbx1tAUNepM0x0PicguU/64)![](http://wx.qlogo.cn/mmopen/YCOL3hU8ffWaibbYL0PdJnsTP5NXWl9acuVJuF6K6cM2sUAcynnyNc5qfT9RicucsrXqjibuQcPsJEE3gj1ibmVYKQ/64)![](http://wx.qlogo.cn/mmopen/339w0m8RJ6ZKcKBeoD2RjJzsOChvfkEyYPU6icOYDyntxg97tlRm3nlbC3BIlBLlnNGtWZrzsQh8xLMJCpJxbjOfEfLicEXt4M/64)![](http://wx.qlogo.cn/mmopen/YCOL3hU8ffU2KW5dSDVIqrVibWGaCL7ermLtJsOI4FqKsPKbohVDIxj1s0q2InD2QR1hB1jKXtsricNRGLHRepeiamkmRzv7jMB/64)![](http://wx.qlogo.cn/mmopen/fgP3YjWdsKu9AgLic6DlSqCOVAFOtaznoLfxBibPqmzXdPT4qre2NNlNhTfLia8a3iaAbZxtamSj9wBqGKbv6sZ3qbOGpt9uYQ0d/64)![](http://wx.qlogo.cn/mmopen/8YSWYKpVxlgbx6nyhZsuwNEs0zPvZHIFB2KmkmdQp45AY4ZOI41H45oA0WjEMBsJP6eW5h4gL0k6VC0LicB5aNI35qT3GPwa2icfcoPDicb1jSdF6xrqd4K55FF4DvFZwf4/64)![](http://wx.qlogo.cn/mmopen/fgP3YjWdsKu9AgLic6DlSqICBaU2zfNmNER8kU3ibGWpceQj2YVNJREnHiaibOoPKZ4oxqibjsFu5aoqyW0zV4o8iaEqRKYQfEew1v/64)![](http://wx.qlogo.cn/mmopen/70iauOibibGjGNkfvmhXocb4XH7HLsYiauuRQnSIP03utOib3GSC44RBFtuVBV4zibnp1jXwqEoE3LxFHAicF9g0bq8IxLw7ZiaLDAKsZ86s97cQ9HrBfel69apyGQpKNQxmic7Wic/64)![](http://wx.qlogo.cn/mmopen/70iauOibibGjGNLwCUkIGDBGM5sZDibsibBU4r40WMccYEnFqiaqjI92icSTy4e7NG87FJgH2eSFmZVcicrOAhpWZepW2BnzYIA6hk1w/64)![](http://wx.qlogo.cn/mmopen/YCOL3hU8ffVOhibibDpX2LxVynlB7RbW3IjjtksW4nQAcFUWwoxtqS3DlYRaL6hxUph4TviaxJO7MwxQ4YR2wVrqA/64)![](http://wx.qlogo.cn/mmopen/fgP3YjWdsKu9AgLic6DlSqIiaovaVcCKs4r5zBecz5FRxic9KISrtxh4jHcEF2NiatmTicKYlpaPxfkUiaiaekvhRg6QkFlibvGhaCec/64)![](http://wx.qlogo.cn/mmopen/70iauOibibGjGMaTa06dLC0HzfAY4mw92iaGRy6Zl1opE1ybW8bX49dekNHrvoSxFESj9cqVOVXUI9NYtSl1gbwg1KEHHlLUiaUvT/64)![](http://wx.qlogo.cn/mmopen/70iauOibibGjGNMg8KbPMtxZYCNaCO8aCFzwc3BzYPysUia4x72NOictO2ibQsnbus5mLNnD5K98y6icB5vfZteZBjhkw/64)

手把手刷数据结构33

图论算法8

手把手刷数据结构 · 目录

上一篇递归思维：k 个一组反转链表下一篇Union-Find 算法怎么应用？

阅读 1.7万

​

写留言

**留言 29**

- 钱程
    
    2019年12月6日
    
    赞10
    
    想问一下 用了路径压缩是不是就不需要平衡性优化了
    
    置顶
    
    labuladong
    
    作者2019年12月6日
    
    赞22
    
    不是，相辅相成，二者都有的话，是效率最高的。如果没有平衡优化，无法避免小树下面接大树，也就会产生更多的路径需要压缩，效率肯定会有下降。
    
- lmmwbw
    
    2019年12月4日
    
    赞5
    
    东哥，find（）压缩可不可以这样理解呢？find()它不会一步到位，而是每次调用find()时都会进行一次压缩，最终的压缩结果是这颗树只有两层![[疑问]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
    置顶
    
    labuladong
    
    作者2019年12月4日
    
    赞3
    
    是这样理解的，每次调用都会压缩，但是不是「最终压缩结果只有两层」，而是一直就保持在不超过 3 层，不可能有更多的情况。 一般高度应该是 2，但当 union 的时候 A 树接到 B 树上，AB 的高度都是 2，那么接起来 B 的高度就是 3 了。但是如果对第三层节点调用 find，第三层又会慢慢压缩到第二层。 这个过程说起来有点绕，自己画个图就很容易理解为啥最多 3 层了。
    
- 5⃣4⃣1⃣®
    
    2019年12月4日
    
    赞43
    
    关注你的公众号才几个月 很欣赏你解题的思路和拓展 对我自己在刷题时帮助很大！最近刚刚拿到谷歌的offer 面试也考到了union find的题目 看到你刚好更新了我就想点进来进来评论下感谢你！ 祝看到评论的各位也面试顺利！ ![😄](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
    labuladong
    
    作者2019年12月4日
    
    赞5
    
    恭喜大佬！
    
- icecophile
    
    2019年12月4日
    
    赞20
    
    很多人遇到好老师之前都觉得自己是蠢材，感激
    
    labuladong
    
    作者2019年12月4日
    
    赞5
    
    大家都这样过来的，加油~
    
- zcx
    
    2019年12月4日
    
    赞7
    
    可以出这个的扩展文章吗![😂](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif) ，期待
    
    labuladong
    
    作者2019年12月4日
    
    赞2
    
    肯定有的
    
- 牛虻
    
    2019年12月4日
    
    赞7
    
    要啥来啥 刚好在看这个并查集
    
    labuladong
    
    作者2019年12月4日
    
    赞4
    
    厉不厉害你东哥
    
- Malphite
    
    2020年11月17日
    
    赞6
    
    东哥可以出图的专题吗 看您的文章感觉能学到很多
    
    labuladong
    
    作者2020年11月17日
    
    赞2
    
    好的
    
- yifei
    
    2020年1月7日
    
    赞6
    
    大佬太牛逼了，别的文章上来就说路径压缩一脸懵逼，还是徐徐道来比较容易懂。
    
    labuladong
    
    作者2020年1月7日
    
    赞4
    
    哈哈，那必须讲清楚😁
    
- 圈圈圆圆
    
    2019年12月5日
    
    赞6
    
    不用邻接表也不用邻接矩阵，直接用数组，巧妙是巧妙 但是太抽象了，怎么实际应用啊 ？就拿社交网络是否存在共同好友 求大佬举个实例![[呲牙]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
    labuladong
    
    作者2019年12月5日
    
    赞3
    
    后面有相关问题，不着急~
    
- KGu
    
    2019年12月5日
    
    赞6
    
    先👍后👀 最近忙，细读文章的时间越来越少。
    
    labuladong
    
    作者2019年12月5日
    
    赞1
    
    慢慢看~
    
- 最后一个老实人
    
    2019年12月4日
    
    赞6
    
    我感觉并查集会出现在我今年考研的卷子上![[坏笑]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
    labuladong
    
    作者2019年12月4日
    
    赞
    
    哈哈，不虚，后面再来点并查集的文章！
    
- 牛虻
    
    2019年12月4日
    
    赞6
    
    加油⛽️阿东，你是最胖的[强壮]
    
    labuladong
    
    作者2019年12月4日
    
    赞1
    
    哈哈哈
    
- 忙着吃饭
    
    2019年12月19日
    
    赞5
    
    感觉用递归实现find是不是思路更清晰一点
    
    labuladong
    
    作者2019年12月19日
    
    赞2
    
    看个人吧，我觉得 while 循环更清晰
    
- wyfwo
    
    2019年12月9日
    
    赞5
    
    路径压缩那里，没管每个节点的size呀，这样的话只能unoin了，如果变种的话（比如分裂一个连通图）就会有问题了吧？菜鸡拙见，大神勿喷![😂](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
    labuladong
    
    作者2019年12月9日
    
    赞1
    
    路径压缩是在 find 里面的，而 connected 和 union 都用到了 find，所以这些操作都会自动进行路径压缩。 话说回来，union-find 算法就这几个标准 API，如果变种，那就不是这个算法需要考虑的问题了。
    
- ティム シン
    
    2019年12月4日
    
    赞5
    
    好文
    
- 吃苦瓜的灰太狼
    
    2019年12月5日
    
    赞4
    
    这篇文章还没更新到labuladong.gutbook.io上啊
    
    labuladong
    
    作者2019年12月5日
    
    赞
    
    已添加
    
- yixuan
    
    2020年9月16日
    
    赞3
    
    可以用并查集生成霍夫曼树对吧，又是一道面试没做出来的题。。
    
    labuladong
    
    作者2020年9月16日
    
    赞
    
    这是两个问题吧，不一样的
    
    yixuan
    
    2020年9月16日
    
    赞
    
    但是霍夫曼树的合并和这里的合并方法有点像，不过没有深度平衡的要求。我感觉两个题都是先把每一个点作为一颗树，然后不断合并。并查集里的合并依据是连通性，霍夫曼树的合并依据是出现次数。
    
    yixuan
    
    2020年9月16日
    
    赞
    
    霍夫曼树要用真正的二叉树结构来做
    
    labuladong
    
    作者2020年9月16日
    
    赞
    
    大概明白你的意思了，霍夫曼还要个优先级队列
    
- 王海洋
    
    2021年10月14日
    
    赞1
    
    妙不可言。
    
- Komorebi
    
    2021年8月13日
    
    赞
    
    路径压缩时，部分结点的size应该会变化吧？
    
    Komorebi
    
    2021年8月13日
    
    赞
    
    是因为根节点的size不会变，其他结点的size不会用到，所以就不需要考虑？
    
    labuladong
    
    作者2021年8月14日
    
    赞1
    
    对的
    
- master
    
    2021年3月20日
    
    赞1
    
    想问下，实际应用中可能用到朋友圈中，因为没用户都是有账号id的，账号如果是随机生成的，用到数组是不是数组太大啊，用hash_map代替数组也是可以的吧![[嘿哈]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
    labuladong
    
    作者2021年3月20日
    
    赞1
    
    当然可以
    
- 热带雨林
    
    2021年2月20日
    
    赞1
    
    并查集的几个性质 使得并查集的增查 效率高（相比平衡二叉树） 1. 并查集的关联关系只增加不减少 2. 并查集是N叉树 3. 节点只关心父节点
    
    labuladong
    
    作者2021年2月21日
    
    赞
    
    ![[强]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![[强]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![[强]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
- XW
    
    2021年10月25日
    
    赞
    
    路径压缩了，其实就不用比较size了吧
    
    labuladong
    
    作者2021年10月25日
    
    赞
    
    理论上还是有细微的优化效果的，不过实际实现中可以不比较了
    
- bkpp
    
    2021年10月4日
    
    赞
    
    东哥yyds！每晚睡前一小篇
    
- 会飞的鱼
    
    2020年12月11日
    
    赞
    
    公众号后台回复关键词「进群」即可加入 LeetCode 刷题群，分享内推机会，大家一起进步~
    

已无更多数据

[](javacript:;)

![](http://mmbiz.qpic.cn/sz_mmbiz_png/gibkIz0MVqdG7UtZh6kBicXeoTkjLGOJnF62iaJkOwBWZ19xJToiaaSv5QBRCU7n3VIFoeJunOjQxd6ao862DAAkeQ/300?wx_fmt=png&wxfrom=18)

labuladong

501339

29

写留言

**留言 29**

- 钱程
    
    2019年12月6日
    
    赞10
    
    想问一下 用了路径压缩是不是就不需要平衡性优化了
    
    置顶
    
    labuladong
    
    作者2019年12月6日
    
    赞22
    
    不是，相辅相成，二者都有的话，是效率最高的。如果没有平衡优化，无法避免小树下面接大树，也就会产生更多的路径需要压缩，效率肯定会有下降。
    
- lmmwbw
    
    2019年12月4日
    
    赞5
    
    东哥，find（）压缩可不可以这样理解呢？find()它不会一步到位，而是每次调用find()时都会进行一次压缩，最终的压缩结果是这颗树只有两层![[疑问]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
    置顶
    
    labuladong
    
    作者2019年12月4日
    
    赞3
    
    是这样理解的，每次调用都会压缩，但是不是「最终压缩结果只有两层」，而是一直就保持在不超过 3 层，不可能有更多的情况。 一般高度应该是 2，但当 union 的时候 A 树接到 B 树上，AB 的高度都是 2，那么接起来 B 的高度就是 3 了。但是如果对第三层节点调用 find，第三层又会慢慢压缩到第二层。 这个过程说起来有点绕，自己画个图就很容易理解为啥最多 3 层了。
    
- 5⃣4⃣1⃣®
    
    2019年12月4日
    
    赞43
    
    关注你的公众号才几个月 很欣赏你解题的思路和拓展 对我自己在刷题时帮助很大！最近刚刚拿到谷歌的offer 面试也考到了union find的题目 看到你刚好更新了我就想点进来进来评论下感谢你！ 祝看到评论的各位也面试顺利！ ![😄](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
    labuladong
    
    作者2019年12月4日
    
    赞5
    
    恭喜大佬！
    
- icecophile
    
    2019年12月4日
    
    赞20
    
    很多人遇到好老师之前都觉得自己是蠢材，感激
    
    labuladong
    
    作者2019年12月4日
    
    赞5
    
    大家都这样过来的，加油~
    
- zcx
    
    2019年12月4日
    
    赞7
    
    可以出这个的扩展文章吗![😂](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif) ，期待
    
    labuladong
    
    作者2019年12月4日
    
    赞2
    
    肯定有的
    
- 牛虻
    
    2019年12月4日
    
    赞7
    
    要啥来啥 刚好在看这个并查集
    
    labuladong
    
    作者2019年12月4日
    
    赞4
    
    厉不厉害你东哥
    
- Malphite
    
    2020年11月17日
    
    赞6
    
    东哥可以出图的专题吗 看您的文章感觉能学到很多
    
    labuladong
    
    作者2020年11月17日
    
    赞2
    
    好的
    
- yifei
    
    2020年1月7日
    
    赞6
    
    大佬太牛逼了，别的文章上来就说路径压缩一脸懵逼，还是徐徐道来比较容易懂。
    
    labuladong
    
    作者2020年1月7日
    
    赞4
    
    哈哈，那必须讲清楚😁
    
- 圈圈圆圆
    
    2019年12月5日
    
    赞6
    
    不用邻接表也不用邻接矩阵，直接用数组，巧妙是巧妙 但是太抽象了，怎么实际应用啊 ？就拿社交网络是否存在共同好友 求大佬举个实例![[呲牙]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
    labuladong
    
    作者2019年12月5日
    
    赞3
    
    后面有相关问题，不着急~
    
- KGu
    
    2019年12月5日
    
    赞6
    
    先👍后👀 最近忙，细读文章的时间越来越少。
    
    labuladong
    
    作者2019年12月5日
    
    赞1
    
    慢慢看~
    
- 最后一个老实人
    
    2019年12月4日
    
    赞6
    
    我感觉并查集会出现在我今年考研的卷子上![[坏笑]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
    labuladong
    
    作者2019年12月4日
    
    赞
    
    哈哈，不虚，后面再来点并查集的文章！
    
- 牛虻
    
    2019年12月4日
    
    赞6
    
    加油⛽️阿东，你是最胖的[强壮]
    
    labuladong
    
    作者2019年12月4日
    
    赞1
    
    哈哈哈
    
- 忙着吃饭
    
    2019年12月19日
    
    赞5
    
    感觉用递归实现find是不是思路更清晰一点
    
    labuladong
    
    作者2019年12月19日
    
    赞2
    
    看个人吧，我觉得 while 循环更清晰
    
- wyfwo
    
    2019年12月9日
    
    赞5
    
    路径压缩那里，没管每个节点的size呀，这样的话只能unoin了，如果变种的话（比如分裂一个连通图）就会有问题了吧？菜鸡拙见，大神勿喷![😂](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
    labuladong
    
    作者2019年12月9日
    
    赞1
    
    路径压缩是在 find 里面的，而 connected 和 union 都用到了 find，所以这些操作都会自动进行路径压缩。 话说回来，union-find 算法就这几个标准 API，如果变种，那就不是这个算法需要考虑的问题了。
    
- ティム シン
    
    2019年12月4日
    
    赞5
    
    好文
    
- 吃苦瓜的灰太狼
    
    2019年12月5日
    
    赞4
    
    这篇文章还没更新到labuladong.gutbook.io上啊
    
    labuladong
    
    作者2019年12月5日
    
    赞
    
    已添加
    
- yixuan
    
    2020年9月16日
    
    赞3
    
    可以用并查集生成霍夫曼树对吧，又是一道面试没做出来的题。。
    
    labuladong
    
    作者2020年9月16日
    
    赞
    
    这是两个问题吧，不一样的
    
    yixuan
    
    2020年9月16日
    
    赞
    
    但是霍夫曼树的合并和这里的合并方法有点像，不过没有深度平衡的要求。我感觉两个题都是先把每一个点作为一颗树，然后不断合并。并查集里的合并依据是连通性，霍夫曼树的合并依据是出现次数。
    
    yixuan
    
    2020年9月16日
    
    赞
    
    霍夫曼树要用真正的二叉树结构来做
    
    labuladong
    
    作者2020年9月16日
    
    赞
    
    大概明白你的意思了，霍夫曼还要个优先级队列
    
- 王海洋
    
    2021年10月14日
    
    赞1
    
    妙不可言。
    
- Komorebi
    
    2021年8月13日
    
    赞
    
    路径压缩时，部分结点的size应该会变化吧？
    
    Komorebi
    
    2021年8月13日
    
    赞
    
    是因为根节点的size不会变，其他结点的size不会用到，所以就不需要考虑？
    
    labuladong
    
    作者2021年8月14日
    
    赞1
    
    对的
    
- master
    
    2021年3月20日
    
    赞1
    
    想问下，实际应用中可能用到朋友圈中，因为没用户都是有账号id的，账号如果是随机生成的，用到数组是不是数组太大啊，用hash_map代替数组也是可以的吧![[嘿哈]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
    labuladong
    
    作者2021年3月20日
    
    赞1
    
    当然可以
    
- 热带雨林
    
    2021年2月20日
    
    赞1
    
    并查集的几个性质 使得并查集的增查 效率高（相比平衡二叉树） 1. 并查集的关联关系只增加不减少 2. 并查集是N叉树 3. 节点只关心父节点
    
    labuladong
    
    作者2021年2月21日
    
    赞
    
    ![[强]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![[强]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![[强]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
- XW
    
    2021年10月25日
    
    赞
    
    路径压缩了，其实就不用比较size了吧
    
    labuladong
    
    作者2021年10月25日
    
    赞
    
    理论上还是有细微的优化效果的，不过实际实现中可以不比较了
    
- bkpp
    
    2021年10月4日
    
    赞
    
    东哥yyds！每晚睡前一小篇
    
- 会飞的鱼
    
    2020年12月11日
    
    赞
    
    公众号后台回复关键词「进群」即可加入 LeetCode 刷题群，分享内推机会，大家一起进步~
    

已无更多数据