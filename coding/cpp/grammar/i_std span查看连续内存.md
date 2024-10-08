原创 程序园 昊天 程序员的园
_2024年03月25日 11:50_ _北京_

C++开发过程中，经常会涉及到数组作为函数的入参，数组传参过程中通常使用单个指针指向数组，但是需要同时传递数组的长度。这无疑增加传参的复杂度，为此C++20提出了新特性std::span，用于解决该问题。

**定义**
std::span是一种轻量级、非拥有、不分配内存的容器，用于表示一段连续内存区域的视图，提供安全、高效地访问和操作数组、容器以及其他连续内存区域。针对如上定义，可以从如下几个方面描述其基本定义：

- 视图（View）：std::span 是对连续内存区域的视图，它只引用已存在的内存区域，不拥有内存。
- 不拥有内存，即其不负责分配/释放内存。
- 引用已存在的内存，即当被引用的内存数据变化后，span同步更新
- 连续性（Continuity）：std::span 只能查看连续的内存区域，因此适用于数组、容器等连续内存的情况，即std::span不可查看std::list和std::deque等内存非连续的容器。
- 安全性（Safety）：std::span 提供了安全的边界检查，避免了指针操作中的常见错误。
- 高效性（Efficiency）：std::span 是轻量级的，没有额外的内存开销，因此在性能上具有优势。

**使用示例**
为尽可能多的展示std::span的使用示例，本文用span分别查看传统数组、malloc分配的连续内存、std::vector，并验证std::span不可用于查看非连续内存区域的std::list和std::deque。代码如下：

**查看传统数组**

```cpp
void using_classic_array() { int arr[]={1,2,3,4,5}; std::span<int> s{arr}; std::cout <<"size "<< s.size()<<"\n"; std::cout <<"size byte "<< s.size_bytes() << "\n"; for (auto & data:s) { std::cout<<data<<"\t"; } std::cout<<"\n"; arr[0] = 500;//std::span同步更新 for (auto& data : s) { std::cout << data << "\t"; } std::cout << "\n"; }
```

**查看连续内存**

```cpp
constexpr int num=100; void using_malloc() { float *f = (float*)malloc(num *sizeof(float)); memset(f,0, num*sizeof(float)); std::span<float> s(f, num); for (size_t i = 0; i < num; i++) { f[i]=(float)i/num; } std::cout << "size " << s.size() << "\n"; std::cout << "size byte " << s.size_bytes() << "\n"; for (const auto& data:s) { std::cout<<data<<"\t"; } std::cout<<"\n\n\n\n"; free(f); for (const auto& data : s) { std::cout << data << "\t"; } std::cout << "\n\n\n\n"; }
```

**查看vector**

```cpp
void using_vector() { std::vector<int> v{ 1,2,3,4,5 }; std::span<int> ss = v; std::cout << "size " << ss.size() << "\n"; std::cout << "size byte " << ss.size_bytes() << "\n"; for (auto& data : ss) { std::cout << data << "\t"; } std::cout << "\n"; v[3]=100;//std::span同步更新 for (auto& data : ss) { std::cout << data << "\t"; } std::cout << "\n"; }
```

**查看非连续内存**

```cpp
void using_non_conitnue() { std::list<int> l{1,2,3,4,5}; std::deque<int> d{ 1,2,3,4,5 }; //std::span<int> s{l,5};//编译错误 //std::span<int> sss = d;//编译错误 //std::span<int> sss{d,5};//编译错误 }
```

由如上代码可知，std::span只能用于查看连续内存区域，同时std::span内涵区域长度信息，并可以通过其size或size_bytes方法获取，也支持for循环。

**总结**

- std::span只可以用于查看连续内存区域，其不负责内存的分配和释放；
- std::span作为原有内存的引用，当原内存发生变更时，std::span可同步更新，需注意其引用内存的有效性，当被引用的内存释放后，std::span指向无效值。
- std::span提高了对于连续内存访问的便利性，增加了数组\\连续内粗传参的简洁度。

______________________________________________________________________

C++20新特性8

C++20新特性 · 目录

上一篇C++20 格式化字符串下一篇C++20 模块

阅读 2520

​
