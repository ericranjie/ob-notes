原创 lightcity 光城

_2022年01月08日 12:07_

![](http://mmbiz.qpic.cn/mmbiz_png/WwIcQHkD5mdEgG339CmT1EHgnaeA6eWto2IVqJNLqrVn033UFXKAnplTycPh7qQVRUia5MpC5kUTnPj44UibFKew/300?wx_fmt=png&wxfrom=19)

**光城**

本公众号旨在推送一些个人学习c++、go、python的一些历程，包括c++那些事，go那些事，python爬虫，机器学习，LeetCode、算法、实习、工作面经等。在这里我们可以共同分享，共同交流，共同学习！

635篇原创内容

公众号

# 拓扑排序

拓扑排序：对一个有向图的顶点进行"排序"。着重点在于图中各个顶点的连接关系，这种连接关系也叫拓扑关系。

有向无环图(Directed Acyclic Graph)：若一个图中不存在环，则称为有向无环图。

如果这个图不是 DAG，那么它是没有拓扑序的；如果是 DAG，那么它至少有一个拓扑序；反之，如果它存在一个拓扑序，那么这个图必定是 DAG。

## 1.207. 课程表

题目：

你这个学期必须选修 numCourses 门课程，记为 0 到 numCourses - 1 。

在选修某些课程之前需要一些先修课程。先修课程按数组 prerequisites 给出，其中 prerequisites\[i\] = \[ai, bi\] ，表示如果要学习课程 ai 则 必须 先学习课程  bi 。

例如，先修课程对 \[0, 1\] 表示：想要学习课程 0 ，你需要先完成课程 1 。请你判断是否可能完成所有课程的学习？如果可以，返回 true ；否则，返回 false 。

```
示例 1：输入：numCourses = 2, prerequisites = [[1,0]]输出：true解释：总共有 2 门课程。学习课程 1 之前，你需要完成课程 0 。这是可能的。
```

题解：

1.bfs实现

找到所有入度为0的点，从所有入度为0的节点开始出发，依次bfs，对其出度节点的入度进行减减，如果此时度为0，表示上游无依赖，可以放入队列中，依次bfs。最后，根据bfs的拓扑序判断，如果长度等于课程数量/节点数量，那么有拓扑序，否则存在环，无拓扑序。

```
class Solution {public:    bool canFinish(int numCourses, vector<vector<int>>& prerequisites) {        map<int, int> deg;        map<int, vector<int>> g;        for (auto p : prerequisites) {            deg[p[0]]++;            g[p[1]].push_back(p[0]);        }        queue<int> q;        vector<int> seq;        int i = 0;        for (; i < numCourses; i++) {            if (!deg.count(i)) {                q.push(i);                seq.push_back(i);            }        }        while (!q.empty()) {            auto x = q.front(); q.pop();            for (auto y : g[x]) {                deg[y]--;                if (deg[y] == 0) {                    seq.push_back(y);                    q.push(y);                }            }        }        return seq.size() == numCourses ? true : false;    }};
```

2.dfs实现

对所有节点进行dfs，dfs时如果有环，此时会陷入死循环，因此使用状态1表示有环，2表示正常访问。

```
class Solution {public:    map<int, vector<int>> g_;    bool canFinish(int numCourses, vector<vector<int>>& prerequisites) {        g_ = map<int, vector<int>>();        for (auto p : prerequisites) {            g_[p[1]].push_back(p[0]);        }        map<int, int> visited;        for (int i = 0; i < numCourses; i++) {            if (!dfs(i, visited)) {                return false;            }        }        return true;    }    bool dfs(int i, map<int, int>& visited) {        if (visited[i] == 1) {            return false;        }        if (visited[i] == 2) {            return true;        }        visited[i] = 1;        for (auto x : g_[i]) {            if (!dfs(x, visited)) {                return false;            }        }        visited[i] = 2;        return true;    }};
```

## 2.210. 课程表 II

题目：

现在你总共有 numCourses 门课需要选，记为 0 到 numCourses - 1。给你一个数组 prerequisites ，其中 prerequisites\[i\] = \[ai, bi\] ，表示在选修课程 ai 前 必须 先选修 bi 。

例如，想要学习课程 0 ，你需要先完成课程 1 ，我们用一个匹配来表示：\[0,1\] 。返回你为了学完所有课程所安排的学习顺序。可能会有多个正确的顺序，你只要返回 任意一种 就可以了。如果不可能完成所有课程，返回 一个空数组 。

```
示例 1：输入：numCourses = 2, prerequisites = [[1,0]]输出：[0,1]解释：总共有 2 门课程。要学习课程 1，你需要先完成课程 0。因此，正确的课程顺序为 [0,1] 。
```

题解：

本题同上述一样，从判断是否有拓扑序变为求拓扑序。

1.bfs实现

```
class Solution {public:    vector<int> findOrder(int numCourses, vector<vector<int>>& prerequisites) {        map<int, int> deg;        map<int, vector<int>> g;        for (auto p : prerequisites) {            deg[p[0]]++;            g[p[1]].push_back(p[0]);        }        queue<int> q;        vector<int> seq;        int i = 0;        for (; i < numCourses; i++) {            if (!deg.count(i)) {                q.push(i);                seq.push_back(i);            }        }        while (!q.empty()) {            auto x = q.front(); q.pop();            for (auto y : g[x]) {                deg[y]--;                if (deg[y] == 0) {                    seq.push_back(y);                    q.push(y);                }            }        }        return seq.size() == numCourses ? seq : vector<int>();    }};
```

2.dfs实现

```
class Solution {public:    map<int, vector<int>> g_;    vector<int> findOrder(int numCourses, vector<vector<int>>& prerequisites) {        g_ = map<int, vector<int>>();        for (auto p : prerequisites) {            g_[p[1]].push_back(p[0]);        }        map<int, int> visited;        stack<int> st;        for (int i = 0; i < numCourses; i++) {            if (!dfs(i, visited, st)) {                return vector<int>{};            }        }        vector<int> ans;        while (!st.empty()) {            ans.push_back(st.top()); st.pop();        }        return ans;    }    bool dfs(int i, map<int, int>& visited, stack<int>& st) {        if (visited[i] == 1) {            return false;        }        if (visited[i] == 2) {            return true;        }        visited[i] = 1;        for (auto x : g_[i]) {            if (!dfs(x, visited, st)) {                return false;            }        }        st.push(i);        visited[i] = 2;        return true;    }};
```

## 3.2127. 参加会议的最多员工数

题目：

一个公司准备组织一场会议，邀请名单上有 n 位员工。公司准备了一张 圆形 的桌子，可以坐下 任意数目 的员工。

员工编号为 0 到 n - 1 。每位员工都有一位 喜欢 的员工，每位员工 当且仅当 他被安排在喜欢员工的旁边，他才会参加会议。每位员工喜欢的员工 不会 是他自己。

给你一个下标从 0 开始的整数数组 favorite ，其中 favorite\[i\] 表示第 i 位员工喜欢的员工。请你返回参加会议的 最多员工数目 。

```
示例 1：输入：favorite = [2,2,1,2]输出：3解释：上图展示了公司邀请员工 0，1 和 2 参加会议以及他们在圆桌上的座位。没办法邀请所有员工参与会议，因为员工 2 没办法同时坐在 0，1 和 3 员工的旁边。注意，公司也可以邀请员工 1，2 和 3 参加会议。所以最多参加会议的员工数目为 3 。
```

题解：

这道题经典的基环树问题，可学习题解

> https://leetcode-cn.com/problems/maximum-employees-to-be-invited-to-a-meeting/solution/nei-xiang-ji-huan-shu-tuo-bu-pai-xu-fen-c1i1b/

分成两类，第一类为基环大小为2，那么求解的便是基环两个顶点各自链长度加上两个节点本身。第二类为基环大小大于2，求解的便是环的大小。

最终，由于一个图中只会存在基环>2或等于2，而不会都存在，那么求解的就是上述两类结果的最大值。

```
class Solution {public:    int maximumInvitations(vector<int>& g) {        int n = g.size();        // 1.预处理        vector<int> deg(n);        for (auto v : g) {            deg[v]++;        }        // 2.入度为0        queue<int> q;        for (int i = 0; i < n; i++) {            if (!deg[i]) {                q.push(i);            }        }        // 3.所有度--,计算链长度        vector<int> max_depth(n);        while (!q.empty()) {            int v = q.front(); q.pop();            max_depth[v]++;            int w = g[v];            max_depth[w] = max(max_depth[w], max_depth[v]);            if (--deg[w] == 0) {                q.push(w);            }        }        // 4.计算链长度        int max_ring_size = 0, sum_chain_size = 0;        for (int i = 0; i < n; i++) {            if (deg[i] == 0) {                continue;            }            deg[i] = 0;            int ring_size = 1;            int v = g[i];            // 从环的某个顶点出发 计算环大小            while (v != i) {                deg[v] = 0;                ++ring_size;                v = g[v];            }            if (ring_size == 2) { // 只有两个节点形成的环                sum_chain_size += max_depth[i] + max_depth[g[i]] + 2;            } else { // 多个节点形成的环                max_ring_size = max(max_ring_size, ring_size);            }        }        return max(max_ring_size, sum_chain_size);    }};
```

> 本节完~

![](https://mmbiz.qlogo.cn/mmbiz_jpg/xdatVaX8ek3UwLBhWibBLb3ATy7p1W9S5APibicPPGTu4NQK4aP7Uf8IOe0Q0EhaRYzb6U22FOYuIIDgwXHlogiblg/0?wx_fmt=jpeg)

lightcity

坚持原创

![赞赏二维码](https://mp.weixin.qq.com/s?__biz=MzI2NjYwOTAyMg==&mid=2247487515&idx=1&sn=1f0fc247003eba84ccf2b30952031bf4&chksm=ea8ad80addfd511c2db16f48566cf87f5c77abc4f29ca3e9ec50dd020f21c420de4d1c084173&mpshare=1&scene=24&srcid=0108wVlcoOW1rTFPf0Gz0q8N&sharer_sharetime=1641652313611&sharer_shareid=5fb9813bfe9ffc983435bfc8d8c5e9ca&key=daf9bdc5abc4e8d08e85f55df2b0faf841495d37f997c530147361541cc3105913dca445cd9c179983375868f39a781bcdfe6635b27641e8e021d5240cb1a62c113002421aa988fab815d92a7fbefa5ad97dece2ad7cedfc537d02512928fb372f14d69f22a5b66fd01259cb19dbb4963a650e01d233f3881eab9fcf3639c828&ascene=0&uin=MTEwNTU1MjgwMw%3D%3D&devicetype=Windows+11+x64&version=63090b19&lang=zh_CN&countrycode=CN&exportkey=n_ChQIAhIQv1mYVLKgEfJKy%2FyypxuwShLmAQIE97dBBAEAAAAAACSSBWP5vtQAAAAOpnltbLcz9gKNyK89dVj0UVZBx4OJ%2F6sWBdt9%2F6PvJv%2B5nUYDldtCcUg0Wt%2FYomhEG5Ofhl98WU26jw2a0cgZllenJy0POwSNyFtKwQqS1NEcbbUD7OZKf%2Fi7iA%2BSRzrWl2hZnwbDzfyKjN0e%2FNGDvYAXqb2jXq%2B3NYHq6LvXoNSTq25bHzLrXXsO9GZ8jitrcvWxSzb2o5TzrkCN%2B31nLlQUVz32aEcrLINbSZScp%2BniB%2FTociVKIBmi5sCYvu11UqZlb8PcTdfHP5Q%2B8dsj&acctmode=0&pass_ticket=kdBZNmTNXlvUppJWBIWgBOBJCxssAZ4pwYD8stBjAZ%2FkYiR2HLF9H9UKDwjD9Qa9&wx_header=1&fasttmpl_type=0&fasttmpl_fullversion=7350504-zh_CN-zip&fasttmpl_flag=1)喜欢作者

LeetCode69

C++23

拓扑1

LeetCode · 目录

上一篇二分解决最小最大问题

阅读 549

​
