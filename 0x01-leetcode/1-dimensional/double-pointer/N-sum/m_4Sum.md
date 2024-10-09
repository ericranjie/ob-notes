Given an array _S_ of _n_ integers, are there elements _a_ , _b_ , _c_ , and _d_ in _S_ such that _a_ + _b_ + _c_ + _d_ = target? Find all unique quadruplets in the array which gives the sum of target.

**Note:**

- Elements in a quadruplet ( _a_ , _b_ , _c_ , _d_ ) must be in non-descending order. (ie, _a_ ≤ _b_ ≤ _c_ ≤ _d_ )
- The solution set must not contain duplicate quadruplets.

```cpp
For example, given array S = {1 0 -1 0 -2 2}, and target =
 0.
A solution set is:
(-1,  0, 0, 1)
(-2, -1, 1, 2)
(-2,  0, 0, 2)
```

LeetCode 中关于数字之和还有其他几道，分别是 [Two Sum](http://www.cnblogs.com/grandyang/p/4130379.html) ，[3Sum](http://www.cnblogs.com/grandyang/p/4481576.html) ，[3Sum Closest](http://www.cnblogs.com/grandyang/p/4510984.html)，虽然难度在递增，但是整体的套路都是一样的，在这里为了避免重复项，我们使用了 STL 中的 TreeSet，其特点是不能有重复，如果新加入的数在 TreeSet 中原本就存在的话，插入操作就会失败，这样能很好的避免的重复项的存在。此题的 O(n^3) 解法的思路跟 [3Sum](http://www.cnblogs.com/grandyang/p/4481576.html) 基本没啥区别，就是多加了一层 for循环，其他的都一样，代码如下：

解法一：

```cpp
// TreeSet-Solutions:
class Solution {
 public:
  vector<vector<int>> fourSum(vector<int> &nums,
    int target) {
    set<vector<int>> res; // 结果向量的集合 TreeSet
    sort(nums.begin(), nums.end()); // 先从小到大 排个序
    for (int i = 0; i < int(nums.size() - 3); ++i) { // 遍历 定下i 去除最后3个数
      for (int j = i + 1; j < int(nums.size() - 2); ++j) { // 遍历 定下j 去除2个数
        if (j > i + 1 && nums[j] == nums[j - 1]) // j和前面重复
          continue; // 跳过 去重
        int left = j + 1, right = nums.size() - 1; // 左右边界双指针 定下最后2个数
        while (left < right) { // 循环持续条件
          int sum = nums[i] + nums[j] + nums[left] +
            nums[right]; // 计算四数之和
          if (sum == target) {
            vector<int> out{nums[i], nums[j], nums[left],
              nums[right]}; // Construct 建立一个临时结果向量
            res.insert(out); // 加入结果向量集合
            ++left; --right; // 左右同时缩小
          } else if (sum < target) // 和小于目标值
            ++left; // 左边界向右
          else // 和大于目标值 
            --right; // 右边界向左
        }
      }
    }
    return vector<vector<int>>(res.begin(), res.end());
  }
};
```

但是毕竟用 TreeSet 来进行去重复的处理还是有些取巧，可能在 Java 中就不能这么做，那么还是来看一种比较正统的做法吧，手动进行去重复处理。主要可以进行的有三个地方，首先在两个 for 循环下可以各放一个，因为一旦当前的数字跟上面处理过的数字相同了，那么找下来肯定还是重复的。之后就是当 sum 等于 target 的时候了，在将四个数字加入结果 res 之后，left 和 right 都需要去重复处理，分别像各自的方面遍历即可，参见代码如下：

解法二：

```cpp
// Manual-Decoupling-Solution:
class Solution {
 public:
  vector<vector<int>> fourSum(vector<int> &nums,
    int target) {
    vector<vector<int>> res;
    int n = nums.size();
    sort(nums.begin(), nums.end());
    for (int i = 0; i < n - 3; ++i) {
      if (i > 0 && nums[i] == nums[i - 1]) continue; // 手动去重
      for (int j = i + 1; j < n - 2; ++j) {
        if (j > i + 1 && nums[j] == nums[j - 1]) continue; // 手动去重
        int left = j + 1, right = n - 1;
        while (left < right) {
          int sum = nums[i] + nums[j] + nums[left] +
            nums[right];
          if (sum == target) {
            vector<int> out{nums[i], nums[j], nums[left],
              nums[right]};
            res.push_back(out);
            while (left < right && nums[left] ==
              nums[left + 1]) ++left; // 去重
            while (left < right && nums[right] ==
              nums[right - 1]) --right; // 去重
            ++left; --right;
          } else if (sum < target) ++left;
          else --right;
        }
      }
    }
    return res;
  }
};
```

Github 同步地址：

[#18](https://github.com/grandyang/leetcode/issues/18)

类似题目：

[Two Sum](http://www.cnblogs.com/grandyang/p/4130379.html)

[3Sum](http://www.cnblogs.com/grandyang/p/4481576.html)

[4Sum II](http://www.cnblogs.com/grandyang/p/6073317.html)

参考资料：

[https://leetcode.com/problems/4sum/](https://leetcode.com/problems/4sum/)

[https://leetcode.com/problems/4sum/discuss/8549/My-16ms-c%2B%2B-code](https://leetcode.com/problems/4sum/discuss/8549/My-16ms-c%2B%2B-code)

[](<https://leetcode.com/problems/4sum/discuss/8575/Clean-accepted-java-O(n3)-solution-based-on-3sum>)[https://leetcode.com/problems/4sum/discuss/8575/Clean-accepted-java-O(n3)-solution-based-on-3sum](<https://leetcode.com/problems/4sum/discuss/8575/Clean-accepted-java-O(n3)-solution-based-on-3sum>)

[LeetCode All in One 题目讲解汇总(持续更新中...)](http://www.cnblogs.com/grandyang/p/4606334.html)
