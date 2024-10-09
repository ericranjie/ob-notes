# 

小夕学算法

_2021年10月19日 22:51_

以下文章来源于编程熊 ，作者编程熊

\[

![](http://wx.qlogo.cn/mmhead/Q3auHgzwzM6viahxlJQyWQeVEfkJhQ9sAXKTnINHTwaR1PtI2Jdic3Pw/0)

**编程熊**.

ACM亚洲区域赛金牌，字节跳动、旷视科技前员工，985研究生。

\](https://mp.weixin.qq.com/s?\_\_biz=MzA5NjM5NTc5NQ==&mid=2247492510&idx=2&sn=c2957900619d4272de928c5e82172793&chksm=90b214e2a7c59df4ea1b6bc1225dd319181371f6e534e38d7a33651a1319112b61a98321aec7&mpshare=1&scene=24&srcid=10217531U0JpMm6dYLbf4yjp&sharer_sharetime=1634777616029&sharer_shareid=5fb9813bfe9ffc983435bfc8d8c5e9ca&key=daf9bdc5abc4e8d0e9526b0c0f6d3460f715587418d83429b043a306c1f3d81789b206e0873e2fca206a6f90cfcaa6365830326175355976ad1273cd2cdd8786f47004ac219903836863b39513d4ac9a2009a1d8e1a5ed8ac8f3248426c0e40d12b071f12e13816926f12a8657db507be391f11de4b950240d26d83797b53c24&ascene=0&uin=MTEwNTU1MjgwMw%3D%3D&devicetype=Windows+11+x64&version=63090b19&lang=zh_CN&countrycode=CN&exportkey=n_ChQIAhIQ6sSEaqCg8e2IwSiM7oeCiRLmAQIE97dBBAEAAAAAAPUtF9NrtocAAAAOpnltbLcz9gKNyK89dVj0iTbnJ3TsLqEC3U5cFcLxL2Ot%2BagdjX%2Fded34Xk0vkZk131MYtppRyfd4KVayQZ1D177UfU9paOYa0BIJ02WcPICXze1QI6ezatVFDthANiU8ISj3V4qnfgaJlQMfICHTGHrQxJmfkOUHyK74qUdOf9z3fk0RcMdPqlJhnDeGJaw%2FZkuUmUnsjNFMuoOTSwnS8AT2tW%2BKX8XslSl9fWgtg%2BzweHtjbvOIAXUYx%2BMrZ%2BbI%2ByBtRQ4%2BCXUBG2yJRiVw&acctmode=0&pass_ticket=kxz20gAaXNSwv2LunvkdB3CPJNy87P%2FYTWGU%2BxQhIZouvKdivOmIErcWOBSJ%2FLRK&wx_header=1&fasttmpl_type=0&fasttmpl_fullversion=7351805-zh_CN-zip&fasttmpl_flag=1#)

大家好，我是编程熊。

往期文章介绍了《线性表》中的数组、链表、栈、队列，以及单调栈和滑动窗口。

[ACM金牌选手讲解LeetCode算法《线性表》](https://mp.weixin.qq.com/s?__biz=Mzg3OTY1NDA3NQ==&mid=2247485549&idx=1&sn=27f64cee0e12f78344b453ab7293acda&scene=21#wechat_redirect)

[ACM金牌选手讲解LeetCode算法《栈和队列的高级应用》](https://mp.weixin.qq.com/s?__biz=Mzg3OTY1NDA3NQ==&mid=2247485802&idx=1&sn=700cc0fccf9aa8f29fa90204d1e42ada&scene=21#wechat_redirect)

本期我们学习哈希，其主要作用是加速我们查找数据的速度。

文章将从以下几个方面展开，内容通俗易懂。

若不想了解哈希原理，直接使用哈希表刷题的话，可以直接下拉到"常见的哈希结构"部分。

![图片](https://mmbiz.qpic.cn/mmbiz_png/1RFyozRxKsfjJ5nP3icqgruHv3ueNWHG8wlP9TqwicJFlhVDGLCiaE8nDprf3G9P7VsZ1lnTkyz2LVmbicoKONpETQ/640?wx_fmt=png&wxfrom=13&tp=wxpic)

哈希

## 哈希概述

哈希表又称散列表，表现形式为将任意长度的输入，通过哈希算法变成固定长度的输出，哈希表是一种使用空间换取时间的数据结构。

通常是存储 `<key，value>`键值对，假设没有哈希表，将 `<key，value>`键值对存储在数组中，给定`key`查找的对应的`value`的时间复杂度为`O(n)`；

数组就是常见的哈希表，下标就是`key`，对应存储的值就是`value`。

通过引如哈希表，将任意长度的输入`key`转化为哈希表中的下标，将`<key，value>`键值对映射到哈希表中，进而加速给定给定`key`，查找`value`的速度，时间复杂度降低到`O(1)`。

下图 以两个键值对`<key1,value1>`、`<key2,value2>`为例，演示了哈希函数和哈希表之间的关系，以及在哈希中起到的作用。

![图片](https://mmbiz.qpic.cn/mmbiz_png/1RFyozRxKsfjJ5nP3icqgruHv3ueNWHG8ehwzZ4vbBJMxgiaAhN57A7SPoz5xnH6oe7HJllrFY3ys0KOMjDKEKNQ/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1 "哈希-编程熊")

哈希

## 哈希表基本操作

### 插入

将键值对`<key，value>`插入到哈希表中。

### 更新

若哈希表中已存在键值为`key`的键值对，更新哈希表键值对`<key，value>`。

### 删除

将键值对`<key，value>`从哈希表中删除。

### 查询

给定`key`，有两种查询方式。

1. 查找`key` 是否存在于哈希表中。

1. 查找`key`对应的`value`。

## 哈希函数

哈希函数又称散列函数，即将给定的任意长度的输入值转化为数组的索引(下标)。

如果有一个长度为`n`的数组，其可以存储`n`对键值对，对应的下标为`[0,n-1]`，通常数组的长度是大于等于键值对的数量。

因此我们需要一个哈希函数，将任意长度的输入映射到`[0,n-1]`，并且每个不同的`key`对应的数组下标一定是不一样的，即每个数组下标唯一对应一个`key`。

下图以三对`<key,value>`为例，演示了哈希函数`hash`将原始`key`，映射到数组下标的过程，具体哈希函数实现可以有很多方法，感兴趣的读者可以自行探究。

![图片](https://mmbiz.qpic.cn/mmbiz_png/1RFyozRxKsfjJ5nP3icqgruHv3ueNWHG8N0Cia8TEPd1aZOSIaHa9OU1kBpJZvgteLzUOp8RgsjTibiaEZXxDNrgFg/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

## 哈希冲突

哈希冲突的出现源于哈希函数对两个不同的键`key1`、`key2` `(key1≠key2)`，但经过哈希函数，`hash(key1)=hash(key2)`，将两个不同的`key`，映射到了同一个数组下标位置，导致了哈希冲突。

下图演示了`key1="abc"`，`key2="bcd"`，两个不同的`key`，经过哈希函数，映射到同一个数组下标`X`。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E "哈希-编程熊")

哈希冲突

### 解决哈希冲突的方法

#### 拉链法

将`hash`值相同的`key`放到一个链表中，查找时从前往后遍历链表，找到想要查找的`key`即可。

设需要插入哈希表的数组`a`长度为`n`，哈希表数组长度为`m`，则拉链法查找任意一个`key`的期望时间复杂度为`O(1+n/m)`。

下图展示了需要插入哈希表的数组`a`，哈希函数`h(x)`，使用拉链法解决哈希冲突的例子。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E "哈希-编程熊")

拉链法

#### 开放地址法

从发生冲突的位置起，按照某种规则找到哈希表中其他空闲的位置，将冲突的元素放入这个空闲的位置。

可以找到空闲位置的条件是: 哈希表的长度一定要大于存放元素的个数。

发生冲突后，以什么样的”规则“找到空闲的位置，有很多种方法:

- 线行探查法: 从冲突的位置开始，依次判断下一个位置是否空闲，直至找到空闲位置。

- 平方探查法:  从冲突的位置x开始，第一次增加`1^2`个位置，第二次增加`2^2`...，直至找到空闲的位置。

- 双散列函数探查法等等

#### 再哈希法

构造多个哈希函数，发生冲突时，更换哈希函数，直至找到空闲位置。

#### 建立公共溢出区

建立公共溢出区，在哈希表中发生哈希冲突时，将数据存储到公共溢出区。

## 常见的哈希结构

当解决问题需要快速查找一个元素/键值对，就可以考虑利用哈希表加速查找的速度。

C++中常用的哈希结构有以下三个:

- 数组

- unordered_set(集合)

- unordered_map(映射: 键值对)

|种类|底层实现|Key是否有序|Key是否可以重复|Key是否可以修改|增删查效率|
|---|---|---|---|---|---|
|std::unordered_set(集合)|哈希表|Key无序|Key不可重复|Key不可修改|O(1)|
|std::unordered_map(映射: 键值对)|哈希表|Key无序|Key不可重复|Key不可修改|O(1)|

C++标准库中的set、map底层基于红黑树，将会在后续章节中详细介绍。

### std::unordered_set用法

下面介绍常见的用法，一般可以满足刷题需要，详细见`https://zh.cppreference.com/w/cpp/container/unordered_set`。

`// 定义一个std::unordered_set   std::unordered_set q;      // 迭代器   // begin: 返回指向起始的迭代器   auto iter = q.begin();   // end: 返回指向末尾的迭代器   auto iter = q.end();      // 容量   // empty: 检查容器是否为空   bool is_empty =  q.empty();   // size: 返回容纳的元素数量   int s = q.size();      // 修改器   // clear: 清除内容   q.clear();   // insert: 插入元素或结点   q.insert(key);   // erase: 擦除元素   q.erase(key);      // 查找   // count: 返回匹配特定键的元素数量   int num = q.count(key);   // find: 寻找带有特定键的元素   auto iter = q.find(key);   // contains: 检查容器是否含有带特定键的元素   bool is_contains = q.contains(key);   `

### std::unordered_map用法

下面介绍常见的用法，一般可以满足刷题需要，详细见`https://zh.cppreference.com/w/cpp/container/unordered_map`。

`// 定义一个std::unordered_map   std::unordered_map q;      // 迭代器   // begin: 返回指向起始的迭代器   auto iter = q.begin();   // end: 返回指向末尾的迭代器   auto iter = q.end();      // 容量   // empty: 检查容器是否为空   bool is_empty =  q.empty();   // size: 返回容纳的元素数量   int s = q.size();      // 修改器   // clear: 清除内容   q.clear();   // insert: 插入元素或结点   q.insert(key);   // erase: 擦除元素   q.erase(key);      // 查找   // count: 返回匹配特定键的元素数量   int num = q.count(key);   // find: 寻找带有特定键的元素   auto iter = q.find(key);   // contains: 检查容器是否含有带特定键的元素   bool is_contains = q.contains(key);   `

## 例题

### LeetCode 1. 两数之和

#### 题意

给定一个整数数组 `nums` 和一个整数目标值 `target`，请你在该数组中找出 **和为目标值** _`target`_ 的那 **两个** 整数，并返回它们的数组下标。

#### 示例

`输入：nums = [2,7,11,15], target = 9   输出：[0,1]   解释：因为 nums[0] + nums[1] == 9 ，返回 [0, 1] 。   `

下图以示例演示一下哈希表，将数组插入到哈希表中，查找给定的`key`，即可以在`O(1)` 的时间复杂度查找到，图中`a，b，c，d`指代哈希表的下标。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E "哈希-编程熊")

示例

#### 题解

建立哈希表，key等于数组的值，value等于值所对应的下标。

然后遍历数组，每次遍历到位置`i`时，检查 `target-num[i]` 是否存在，注意`target-num[i]`的位置不能等于`i`。

#### 代码

`class Solution {       public int[] twoSum(int[] nums, int target) {           HashMap<Integer, Integer> numExist = new HashMap<Integer, Integer>();           for (int i = 0; i < nums.length; ++i) {               if (numExist.containsKey(target - nums[i])) {                   return new int[]{i, numExist.get(target - nums[i])};               }               numExist.put(nums[i], i);           }           return new int[2];       }   }   `

### LeetCode 128. 最长连续序列

#### 题意

给定一个未排序的整数数组 `nums` ，找出数字连续的最长序列（不要求序列元素在原数组中连续）的长度。

#### 示例

`输入：nums = [100,4,200,1,3,2]   输出：4   解释：最长数字连续序列是 [1, 2, 3, 4]。它的长度为 4。   `

#### 题解

**方法一**

对数组数字排序，然后遍历排序后的数组，找到最长的连续序列。

时间复杂度`O(nlogn)`

**方法二**

哈希可以快速查找一个数字。

将数组数字插入到哈希表，每次随便拿出一个，删除其连续的数字，直至找不到连续的，记录删除的长度，可以找到最长连续序列。

下图以示例展示，如何利用哈希表，找到最长连续序列。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E "哈希-编程熊")

示例

#### 代码

`class Solution {   public:       int longestConsecutive(vector<int>& nums) {           unordered_set<int> q;           for (int i = 0; i < nums.size(); i++) {               q.insert(nums[i]);           }           int ans = 0;           while (!q.empty()) {               int now = *q.begin();               q.erase(now);               int l = now - 1, r = now + 1;               while (q.find(l) != q.end()) {                   q.erase(l);                   l--;               }               while(q.find(r) != q.end()) {                   q.erase(r);                   r++;               }               l = l + 1, r = r - 1;               ans = max(ans, r - l + 1);           }           return ans;       }   };   `

### 习题推荐

1. LeetCode 217. 存在重复元素

1. LeetCode 594. 最长和谐子序列

1. LeetCode 149. 直线上最多的点数

1. LeetCode 332. 重新安排行程

我是编程熊，我们下期见。

![](http://mmbiz.qpic.cn/mmbiz_png/1RFyozRxKsdsccUEd9vlz8y38TlAldSjwtH7o3FAGpk2IvbWF354ePDxfRNibzNtdn1ibSMK2hqnHCFLPjJ9vOYw/300?wx_fmt=png&wxfrom=19)

**编程熊**

ACM亚洲区域赛金牌，字节跳动、旷视科技前员工，985研究生。

22篇原创内容

公众号

---END---

你好，我是编程熊，双非逆袭选手，校招时拿下字节跳动、旷视科技等大厂offer，ACM亚洲区域赛金牌选手，保研985的研究生，《ACM金牌选手讲解LeetCode算法系列》作者。\
点击下方【阅读原文】，直达学习算法与数据结构的宝藏仓库，欢迎star；公众号后台回复【书】，免费领取计算机学习核心资源；加星标，更快获取编程熊最新精彩文章。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

阅读原文

阅读 865

​

写留言

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/NP2dQRIKOrIGH3JlT3ZfickrsCetmiaoSvG9EpFE9gUR2vsWEDiaqQyGFwMApu48aK1G4l7hvtaucB36hHib2gImuw/300?wx_fmt=png&wxfrom=18)

小夕学算法

3分享1

写留言

写留言

**留言**

暂无留言
