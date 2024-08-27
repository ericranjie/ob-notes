
机器人网

 _2021年12月23日 18:00_

**01**

# **基本概念**  

贪心算法是指在对问题求解时，总是做出在当前看来是最好的选择。也就是说，不从整体最优上加以考虑，只做出在某种意义上的局部最优解。贪心算法不是对所有问题都能得到整体最优解，关键是贪心策略的选择，选择的贪心策略必须具备无后效性，即某个状态以前的过程不会影响以后的状态，只与当前状态有关。

  

贪心算法没有固定的算法框架，算法设计的关键是贪心策略的选择。必须注意的是，贪心算法不是对所有问题都能得到整体最优解，选择的贪心策略必须具备无后效性（即某个状态以后的过程不会影响以前的状态，只与当前状态有关。）

  

所以，对所采用的贪心策略一定要仔细分析其是否满足无后效性。

  

# **02**

# **贪心算法的基本思路**

  

解题的一般步骤是：

  

1.建立数学模型来描述问题；  
2.把求解的问题分成若干个子问题；  
3.对每一子问题求解，得到子问题的局部最优解；  
4.把子问题的局部最优解合成原来问题的一个解。

  

# **03**

# **该算法存在的问题**

  

- 不能保证求得的最后解是最佳的
    
- 不能用来求最大值或最小值的问题
    
- 只能求满足某些约束条件的可行解的范围
    

#   

# **04**

# **贪心算法适用的问题**

  
贪心策略适用的前提是：局部最优策略能导致产生全局最优解。

  
实际上，贪心算法适用的情况很少。一般对一个问题分析是否适用于贪心算法，可以先选择该问题下的几个实际数据进行分析，就可以做出判断。

  

# **05**

# **贪心选择性质**

  

所谓贪心选择性质是指所求问题的整体最优解可以通过一系列局部最优的选择，换句话说，当考虑做何种选择的时候，我们只考虑对当前问题最佳的选择而不考虑子问题的结果。这是贪心算法可行的第一个基本要素。贪心算法以迭代的方式作出相继的贪心选择，每作一次贪心选择就将所求问题简化为规模更小的子问题。对于一个具体问题，要确定它是否具有贪心选择性质，必须证明每一步所作的贪心选择最终导致问题的整体最优解。

  
当一个问题的最优解包含其子问题的最优解时，称此问题具有最优子结构性质。问题的最优子结构性质是该问题可用贪心算法求解的关键特征。

  

# **06**

# **贪心算法的实现框架**

  
从问题的某一初始解出发：  

  

```
while (朝给定总目标前进一步)
```

  
由所有解元素组合成问题的一个可行解；

  

# **07**

# **例题分析**

  

如果大家比较了解动态规划，就会发现它们之间的相似之处。最优解问题大部分都可以拆分成一个个的子问题，把解空间的遍历视作对子问题树的遍历，则以某种形式对树整个的遍历一遍就可以求出最优解，大部分情况下这是不可行的。贪心算法和动态规划本质上是对子问题树的一种修剪，两种算法要求问题都具有的一个性质就是子问题最优性(组成最优解的每一个子问题的解，对于这个子问题本身肯定也是最优的)。动态规划方法代表了这一类问题的一般解法，我们自底向上构造子问题的解，对每一个子树的根，求出下面每一个叶子的值，并且以其中的最优值作为自身的值，其它的值舍弃。而贪心算法是动态规划方法的一个特例，可以证明每一个子树的根的值不取决于下面叶子的值，而只取决于当前问题的状况。换句话说，不需要知道一个节点所有子树的情况，就可以求出这个节点的值。由于贪心算法的这个特性，它对解空间树的遍历不需要自底向上，而只需要自根开始，选择最优的路，一直走到底就可以了。  
  
话不多说，我们来看几个具体的例子慢慢理解它：  
  
**1.活动选择问题**  
  
这是《算法导论》上的例子，也是一个非常经典的问题。有n个需要在同一天使用同一个教室的活动a1,a2,…,an，教室同一时刻只能由一个活动使用。每个活动ai都有一个开始时间si和结束时间fi 。一旦被选择后，活动ai就占据半开时间区间[si,fi)。如果[si,fi]和[sj,fj]互不重叠，ai和aj两个活动就可以被安排在这一天。该问题就是要安排这些活动使得尽量多的活动能不冲突的举行。例如下图所示的活动集合S，其中各项活动按照结束时间单调递增排序。  
  
![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)  
  
考虑使用贪心算法的解法。为了方便，我们用不同颜色的线条代表每个活动，线条的长度就是活动所占据的时间段，蓝色的线条表示我们已经选择的活动；红色的线条表示我们没有选择的活动。  
  
如果我们每次都选择开始时间最早的活动，不能得到最优解：  
  
![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)  
  
如果我们每次都选择持续时间最短的活动，不能得到最优解：  
  
![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)  
  
可以用数学归纳法证明，我们的贪心策略应该是每次选取结束时间最早的活动。直观上也很好理解，按这种方法选择相容活动为未安排活动留下尽可能多的时间。这也是把各项活动按照结束时间单调递增排序的原因。  
  

```
#include<cstdio>
```

  
**2.钱币找零问题**  
  
这个问题在我们的日常生活中就更加普遍了。假设1元、2元、5元、10元、20元、50元、100元的纸币分别有c0, c1, c2, c3, c4, c5, c6张。现在要用这些钱来支付K元，至少要用多少张纸币？用贪心算法的思想，很显然，每一步尽可能用面值大的纸币即可。在日常生活中我们自然而然也是这么做的。在程序中已经事先将Value按照从小到大的顺序排好。  
  

```
#include<iostream>
```

**  
3.再论背包问题**  
  
在从零开始学动态规划中我们已经谈过三种最基本的背包问题：零一背包，部分背包，完全背包。很容易证明，背包问题不能使用贪心算法。然而我们考虑这样一种背包问题：在选择物品i装入背包时，可以选择物品的一部分，而不一定要全部装入背包。这时便可以使用贪心算法求解了。计算每种物品的单位重量价值作为贪心选择的依据指标，选择单位重量价值最高的物品，将尽可能多的该物品装入背包，依此策略一直地进行下去，直到背包装满为止。在零一背包问题中贪心选择之所以不能得到最优解原因是贪心选择无法保证最终能将背包装满，部分闲置的背包空间使每公斤背包空间的价值降低了。在程序中已经事先将单位重量价值按照从大到小的顺序排好。  
  

```
    #include<iostream>   
```

**  
4.多机调度问题**  
  
n个作业组成的作业集，可由m台相同机器加工处理。要求给出一种作业调度方案，使所给的n个作业在尽可能短的时间内由m台机器加工处理完成。作业不能拆分成更小的子作业；每个作业均可在任何一台机器上加工处理。这个问题是NP完全问题，还没有有效的解法(求最优解)，但是可以用贪心选择策略设计出较好的近似算法(求次优解)。当n<=m时，只要将作业时间区间分配给作业即可；当n>m时，首先将n个作业从大到小排序，然后依此顺序将作业分配给空闲的处理机。也就是说从剩下的作业中，选择需要处理时间最长的，然后依次选择处理时间次长的，直到所有的作业全部处理完毕，或者机器不能再处理其他作业为止。如果我们每次是将需要处理时间最短的作业分配给空闲的机器，那么可能就会出现其它所有作业都处理完了只剩所需时间最长的作业在处理的情况，这样势必效率较低。在下面的代码中没有讨论n和m的大小关系，把这两种情况合二为一了。  
  

```
#include<iostream>  
```

**  
5.小船过河问题**  
  
POJ1700是一道经典的贪心算法例题。题目大意是只有一艘船，能乘2人，船的运行速度为2人中较慢一人的速度，过去后还需一个人把船划回来，问把n个人运到对岸，最少需要多久。先将所有人过河所需的时间按照升序排序，我们考虑把单独过河所需要时间最多的两个旅行者送到对岸去，有两种方式：  
  
1.最快的和次快的过河，然后最快的将船划回来；次慢的和最慢的过河，然后次快的将船划回来，所需时间为：t[0]+2*t[1]+t[n-1]；  
  
2.最快的和最慢的过河，然后最快的将船划回来，最快的和次慢的过河，然后最快的将船划回来，所需时间为：2*t[0]+t[n-2]+t[n-1]。  
  
算一下就知道，除此之外的其它情况用的时间一定更多。每次都运送耗时最长的两人而不影响其它人，问题具有贪心子结构的性质。  
  
AC代码：  
  

```
#include<iostream>
```

**  
6.区间覆盖问题**  
  
POJ1328是一道经典的贪心算法例题。题目大意是假设海岸线是一条无限延伸的直线。陆地在海岸线的一侧，而海洋在另一侧。每一个小的岛屿是海洋上的一个点。雷达坐落于海岸线上，只能覆盖d距离，所以如果小岛能够被覆盖到的话，它们之间的距离最多为d。题目要求计算出能够覆盖给出的所有岛屿的最少雷达数目。对于每个小岛，我们可以计算出一个雷达所在位置的区间。  
  
![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)  
  
问题转化为如何用尽可能少的点覆盖这些区间。先将所有区间按照左端点大小排序，初始时需要一个点。如果两个区间相交而不重合，我们什么都不需要做；如果一个区间完全包含于另外一个区间，我们需要更新区间的右端点；如果两个区间不相交，我们需要增加点并更新右端点。  
  
AC代码：  
  

```
#include<cmath>
```

**  
7.销售比赛**  
  
在学校OJ上做的一道比较好的题，这里码一下。假设有偶数天，要求每天必须买一件物品或者卖一件物品，只能选择一种操作并且不能不选，开始手上没有这种物品。现在给你每天的物品价格表，要求计算最大收益。首先要明白，第一天必须买，最后一天必须卖，并且最后手上没有物品。那么除了第一天和最后一天之外我们每次取两天，小的买大的卖，并且把卖的价格放进一个最小堆。如果买的价格比堆顶还大，就交换。这样我们保证了卖的价格总是大于买的价格，一定能取得最大收益。  
  

```
#include<queue>
```

  
下面我们结合数据结构中的知识讲解几个例子。  
  
**8.Huffman编码**  
  
这同样是《算法导论》上的例子。Huffman编码是广泛用于数据文件压缩的十分有效的编码方法。我们可以有多种方式表示文件中的信息，如果用01串表示字符，采用定长编码表示，则需要3位表示一个字符，整个文件编码需要300000位；采用变长编码表示，给频率高的字符较短的编码，频率低的字符较长的编码，达到整体编码减少的目的，则整个文件编码需要(45×1+13×3+12×3+16×3+9×4+5×4)×1000=224000位，由此可见，变长码比定长码方案好，总码长减小约25%。  
  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  
对每一个字符规定一个01串作为其代码，并要求任一字符的代码都不是其他字符代码的前缀，这种编码称为前缀码。可能无前缀码是一个更好的名字，但是前缀码是一致认可的标准术语。编码的前缀性质可以使译码非常简单：例如001011101可以唯一的分解为0,0,101,1101，因而其译码为aabe。译码过程需要方便的取出编码的前缀，为此可以用二叉树作为前缀码的数据结构：树叶表示给定字符；从树根到树叶的路径当作该字符的前缀码；代码中每一位的0或1分别作为指示某节点到左儿子或右儿子的路标。  
  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  
从上图可以看出，最优前缀编码码的二叉树总是一棵完全二叉树，而定长编码的二叉树不是一棵完全二叉树。给定编码字符集C及频率分布f，C的一个前缀码编码方案对应于一棵二叉树T。字符c在树T中的深度记为dT(c)，dT(c)也是字符c的前缀码长。则平均码长定义为：  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  
使平均码长达到最小的前缀码编码方案称为C的最优前缀码。       
  
Huffman编码的构造方法：先合并最小频率的2个字符对应的子树，计算合并后的子树的频率；重新排序各个子树；对上述排序后的子树序列进行合并；重复上述过程，将全部结点合并成1棵完整的二叉树；对二叉树中的边赋予0、1，得到各字符的变长编码。  
  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  
POJ3253一道就是利用这一思想的典型例题。题目大意是有把一块无限长的木板锯成几块给定长度的小木板，每次锯都需要一定费用，费用就是当前锯的木板的长度。给定各个要求的小木板的长度以及小木板的个数，求最小的费用。以要求3块长度分别为5,8,5的木板为例：先从无限长的木板上锯下长度为21的木板，花费21；再从长度为21的木板上锯下长度为5的木板，花费5；再从长度为16的木板上锯下长度为8的木板，花费8；总花费=21+5+8=34。利用Huffman思想，要使总费用最小，那么每次只选取最小长度的两块木板相加，再把这些和累加到总费用中即可。为了提高效率，使用优先队列优化，并且还要注意使用long long int保存结果。  
  
AC代码：  
  

```
#include<queue>
```

**  
9.Dijkstra算法**  
  
Dijkstra算法是由E.W.Dijkstra于1959年提出，是目前公认的最好的求解最短路径的方法，使用的条件是图中不能存在负边。算法解决的是单个源点到其他顶点的最短路径问题，其主要特点是每次迭代时选择的下一个顶点是标记点之外距离源点最近的顶点，简单的说就是bfs+贪心算法的思想。  
  

```
#include<iostream>
```

**  
10.最小生成树算法**  
  
设一个网络表示为无向连通带权图G =(V, E) , E中每条边(v,w)的权为c[v][w]。如果G的子图G’是一棵包含G的所有顶点的树，则称G’为G的生成树。生成树的代价是指生成树上各边权的总和，在G的所有生成树中，耗费最小的生成树称为G的最小生成树。例如在设计通信网络时，用图的顶点表示城市，用边(v,w)的权c[v][w]表示建立城市v和城市w之间的通信线路所需的费用，最小生成树给出建立通信网络的最经济方案。  
  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  
构造最小生成树的Kruskal算法和Prim算法都利用了MST(最小生成树)性质：设顶点集U是V的真子集(可以任意选取)，如果(u,v)∈E为横跨点集U和V—U的边，即u∈U，v∈V- U，并且在所有这样的边中，(u,v)的权c[u][v]最小，则一定存在G的一棵最小生成树，它以(u,v)为其中一条边。  
  
![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)  
![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)  
  
使用反证法可以很简单的证明此性质。假设对G的任意一个最小生成树T，针对点集U和V—U，(u,v)∈E为横跨这2个点集的最小权边，T不包含该最小权边<u, v>，但T包括节点u和v。将<u,v>添加到树T中，树T将变为含回路的子图，并且该回路上有一条不同于<u,v>的边<u’,v’>,u’∈U,v’∈V-U。将<u’,v’>删去，得到另一个树T’，即树T’是通过将T中的边<u’,v’>替换为<u,v>得到的。由于这2条边的耗费满足c[u][v]≤c[u’][v’]，故即T’耗费≤T的耗费，这与T是任意最小生成树的假设相矛盾，从而得证。  
  
![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)  
  
Prim算法每一步都选择连接U和V-U的权值最小的边加入生成树。  
  
![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)  
  

```
#include<iostream>
```

  
  
Kruskal算法每一步直接将权值最小的不成环的边加入生成树，我们借助并查集这一数据结构可以完美实现它。  
  
![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)  
  

```
#include<iostream>
```

_来源 ：CSDN，简书等_

  

- END-  
  
**关注@面包板社区  
学习更多机器人、电子电气、电路、嵌入式技术**  

![](http://mmbiz.qpic.cn/mmbiz_png/xML2GYBfTfm0FZOSezRG5r9gfMWSlgicqUic5P2EtR5IJ75mQzDEQ3hibVbJjLNUNWHjibOyMtAN6q8sWHib95iaulQw/300?wx_fmt=png&wxfrom=19)

**面包板社区**

分享电子技术干货，工程师福利！EET电子工程专辑、ESM国际电子商情、EDN电子技术设计官方社区。

2779篇原创内容

公众号

****#推荐阅读#****

[](http://mp.weixin.qq.com/s?__biz=MzIzMjQwNjQzNA==&mid=2247539708&idx=3&sn=4dc682a88ecf77062ba6d1ec9c060c63&chksm=e8977731dfe0fe27cb250049edb2dc962a47aecd2741e1d6aff064ca7a25b294880ad81275bb&scene=21#wechat_redirect)[**深度：嵌入式系统的软件架构设计！**](http://mp.weixin.qq.com/s?__biz=MzIzMjQwNjQzNA==&mid=2247539708&idx=3&sn=4dc682a88ecf77062ba6d1ec9c060c63&chksm=e8977731dfe0fe27cb250049edb2dc962a47aecd2741e1d6aff064ca7a25b294880ad81275bb&scene=21#wechat_redirect)  
[**嵌入式软件架构设计之分层设计**](http://mp.weixin.qq.com/s?__biz=MzIzMjQwNjQzNA==&mid=2247514164&idx=3&sn=ef66c07a31edc2e6fe699bbb9721a718&chksm=e89794f9dfe01defd2d715f5ff40ea729f15a4c3d70a3b28b7988671b72a36bd3ca58750929c&scene=21#wechat_redirect)  
[](http://mp.weixin.qq.com/s?__biz=MzA3MTIyNzIxOQ==&mid=2655553321&idx=1&sn=633f305911cae0591d2bd532753fd606&chksm=848ce515b3fb6c031b034b7e3a1f29ebc59342d2e094ef161053cd74458c4ab8bd2aff4b7c52&scene=21#wechat_redirect)[**工程师常用10大基础算法**](http://mp.weixin.qq.com/s?__biz=MzA3MTIyNzIxOQ==&mid=2655553321&idx=1&sn=633f305911cae0591d2bd532753fd606&chksm=848ce515b3fb6c031b034b7e3a1f29ebc59342d2e094ef161053cd74458c4ab8bd2aff4b7c52&scene=21#wechat_redirect)  
[**电机是怎么转起来的？**](http://mp.weixin.qq.com/s?__biz=MzA3MTIyNzIxOQ==&mid=2655553320&idx=1&sn=5a11672fc200328162ac001e002ad42b&chksm=848ce514b3fb6c02dcfbc16ffcf72513cb002316e090f3950c0ecb7c2a856064caf11eea3118&scene=21#wechat_redirect)  
[**大牛谈C语言的高级用法**](http://mp.weixin.qq.com/s?__biz=MzA3MTIyNzIxOQ==&mid=2655553284&idx=1&sn=0a9a6c88c6d79125e935466e6198f1f1&chksm=848ce538b3fb6c2e5082738594f44d68ab418531711b1315c7bf93f9c02b65076206c0a47ab7&scene=21#wechat_redirect)  
[**《电磁兼容设计》165页PPT**](http://mp.weixin.qq.com/s?__biz=MzA3MTIyNzIxOQ==&mid=2655553199&idx=1&sn=4ed61387c90f38f7cd753ffbd4215228&chksm=848ce493b3fb6d859eab8adf16e5ada750e3a3390526b3776280883125f8327575ab0e2d3d95&scene=21#wechat_redirect)  
[**无人机是如何实现避障的？**](http://mp.weixin.qq.com/s?__biz=MzA3MTIyNzIxOQ==&mid=2655553197&idx=1&sn=f57f2d692d2ce0a037eb814beb3ef975&chksm=848ce491b3fb6d871763f1a43cd7888ee0a62e138daa7766cfc614e7db247615e1b3394cb8d6&scene=21#wechat_redirect)  

**![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)点击阅读原文下载资料《算法C语言实现》**  

阅读原文

阅读 1112

​