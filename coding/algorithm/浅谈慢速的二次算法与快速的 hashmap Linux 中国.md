# 

邀你一起成为开源贡献者 Linux

 _2021年09月18日 07:30_

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/W9DqKgFsc69AcFicooCicYiaDsMoGYT8eDA0pubs4taY7PAh0uEibjXTZgIFa6VGRGEObnk7SIDWlkwTrlJsibKAaBw/640?wx_fmt=jpeg&wxfrom=13&tp=wxpic)

**导读：**hashmap 当然不是魔法　 　 　 　 　 　 　 　 　 　 　 　 　 　 　 　 　 　 　 　 　 　 　 　

本文字数：5339，阅读时长大约：7分钟

  

https://linux.cn/article-13786-1.html  
作者：Julia Evans  
译者：unigeorge  

大家好！昨天我与一位朋友聊天，他正在准备编程面试，并试图学习一些算法基础知识。

我们聊到了二次时间(quadratic-time)与线性时间(linear-time)算法的话题，我认为在这里写这篇文章会很有趣，因为避免二次时间算法不仅在面试中很重要——有时在现实生活中了解一下也是很好的！后面我会快速解释一下什么是“二次时间算法” :)

以下是我们将要讨论的 3 件事：

1. 二次时间函数比线性时间函数慢得非常非常多

2. 有时可以通过使用 hashmap 把二次算法变成线性算法

3. 这是因为 hashmap 查找非常快（即时查询！）

我会尽量避免使用数学术语，重点关注真实的代码示例以及它们到底有多快/多慢。

![图片](https://mmbiz.qpic.cn/mmbiz_svg/B2EfAOZfS1iaS9dgibME2dgeHzAACNC34LGpDCh32g0mtDpAEzexzVMQU55u8VjrRYy0XCEzFelGyhs8LsDPQWukMQsmnribZRg/640?wx_fmt=svg&wxfrom=13)

目标问题：取两个列表的交集

我们来讨论一个简单的面试式问题：获取 2 个数字列表的交集。例如，`intersect([1,2,3], [2,4,5])` 应该返回 `[2]`。

这个问题也是有些现实应用的——你可以假设有一个真实程序，其需求正是取两个 ID 列表的交集。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

“显而易见”的解决方案：

我们来写一些获取 2 个列表交集的代码。下面是一个实现此需求的程序，命名为 `quadratic.py`。

1. `import sys`
    

3. `# 实际运行的代码`
    
4. `def intersection(list1, list2):`
    
5.     `result = []`
    
6.     `for x in list1:`
    
7.         `for y in list2:`
    
8.             `if x == y:`
    
9.                 `result.append(y)`
    
10.     `return result`
    

12. `# 一些样板，便于我们从命令行运行程序，处理不同大小的列表`
    
13. `def run(n):`
    
14.     `# 定义两个有 n+1 个元素的列表`
    
15.     `list1 = list(range(3, n)) + [2]`
    
16.     `list2 = list(range(n+1, 2*n)) + [2]`
    
17.     `# 取其交集并输出结果`
    
18.     `print(list(intersection(list1, list2)))`
    

20. `# 使用第一个命令行参数作为输入，运行程序`
    
21. `run(int(sys.argv[1]))`
    

程序名为 `quadratic.py`（LCTT 译注：“quadratic”意为“二次方的”）的原因是：如果 `list1` 和 `list2` 的大小为 `n`，那么内层循环（`if x == y`）会运行 `n^2` 次。在数学中，像 `x^2` 这样的函数就称为“二次”函数。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

`quadratic.py` 有多慢？

用一些不同长度的列表来运行这个程序，两个列表的交集总是相同的：`[2]`。

1. `$ time python3 quadratic.py 10`
    
2. `[2]`
    

4. `real    0m0.037s`
    
5. `$ time python3 quadratic.py 100`
    
6. `[2]`
    

8. `real    0m0.053s`
    
9. `$ time python3 quadratic.py 1000`
    
10. `[2]`
    

12. `real    0m0.051s`
    
13. `$ time python3 quadratic.py 10000 # 10,000`
    
14. `[2]`
    

16. `real    0m1.661s`
    

到目前为止，一切都还不错——程序仍然只花费不到 2 秒的时间。

然后运行该程序处理两个包含 100,000 个元素的列表，我不得不等待了很长时间。结果如下：

1. `$ time python3 quadratic.py 100000 # 100,000`
    
2. `[2]`
    

4. `real    2m41.059s`
    

这可以说相当慢了！总共花费了 160 秒，几乎是在 10,000 个元素上运行时（1.6 秒）的 100 倍。所以我们可以看到，在某个点之后，每次我们将列表扩大 10 倍，程序运行的时间就会增加大约 100 倍。

我没有尝试在 1,000,000 个元素上运行这个程序，因为我知道它会花费又 100 倍的时间——可能大约需要 3 个小时。我没时间这样做！

你现在大概明白了为什么二次时间算法会成为一个问题——即使是这个非常简单的程序也会很快变得非常缓慢。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

快速版：`linear.py`

好，接下来我们编写一个快速版的程序。我先给你看看程序的样子，然后再分析。

1. `import sys`
    

3. `# 实际执行的算法`
    
4. `def intersection(list1, list2):`
    
5.     `set1 = set(list1) # this is a hash set`
    
6.     `result = []`
    
7.     `for y in list2:`
    
8.         `if y in set1:`
    
9.             `result.append(y)`
    
10.     `return result`
    

12. `# 一些样板，便于我们从命令行运行程序，处理不同大小的列表`
    
13. `def run(n):`
    
14.     `# 定义两个有 n+1 个元素的列表`
    
15.     `list1 = range(3, n) + [2]`
    
16.     `list2 = range(n+1, 2*n) + [2]`
    
17.     `# 输出交集结果`
    
18.     `print(intersection(list1, list2))`
    

20. `run(int(sys.argv[1]))`
    

（这不是最惯用的 Python 使用方式，但我想在尽量避免使用太多 Python 思想的前提下编写代码，以便不了解 Python 的人能够更容易理解）

这里我们做了两件与慢速版程序不同的事：

1. 将 `list1` 转换成名为 `set1` 的 set 集合

2. 只使用一个 for 循环而不是两个

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

看看 `linear.py` 程序有多快

在讨论 为什么 这个程序快之前，我们先在一些大型列表上运行该程序，以此证明它确实是很快的。此处演示该程序依次在大小为 10 到 10,000,000 的列表上运行的过程。（请记住，我们上一个的程序在 100,000 个元素上运行时开始变得非常非常慢）

1. `$ time python3 linear.py 100`
    
2. `[2]`
    

4. `real    0m0.056s`
    
5. `$ time python3 linear.py 1000`
    
6. `[2]`
    

8. `real    0m0.036s`
    
9. `$ time python3 linear.py 10000 # 10,000`
    
10. `[2]`
    

12. `real    0m0.028s`
    
13. `$ time python3 linear.py 100000 # 100,000`
    
14. `[2]`
    

16. `real    0m0.048s <-- quadratic.py took 2 minutes in this case! we're doing it in 0.04 seconds now!!! so fast!`
    
17. `$ time python3 linear.py 1000000 # 1,000,000`
    
18. `[2]`
    

20. `real    0m0.178s`
    
21. `$ time python3 linear.py 10000000 # 10,000,000`
    
22. `[2]`
    

24. `real    0m1.560s`
    

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

在极大型列表上运行 `linear.py`

如果我们试着在一个非常非常大的列表（100 亿 / 10,000,000,000 个元素）上运行它，那么实际上会遇到另一个问题：它足够 快 了（该列表仅比花费 4.2 秒的列表大 100 倍，因此我们大概应该能在不超过 420 秒的时间内完成），但我的计算机没有足够的内存来存储列表的所有元素，因此程序在运行结束之前崩溃了。

1. `$ time python3 linear.py 10000000000`
    
2. `Traceback (most recent call last):`
    
3.   `File "/home/bork/work/homepage/linear.py", line 18, in <module>`
    
4.     `run(int(sys.argv[1]))`
    
5.   `File "/home/bork/work/homepage/linear.py", line 13, in run`
    
6.     `list1 = [1] * n + [2]`
    
7. `MemoryError`
    

9. `real    0m0.090s`
    
10. `user    0m0.034s`
    
11. `sys 0m0.018s`
    

不过本文不讨论内存使用，所以我们可以忽略这个问题。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

那么，为什么 `linear.py` 很快呢？

现在我将试着解释为什么 `linear.py` 很快。

再看一下我们的代码:

1. `def intersection(list1, list2):`
    
2.     `set1 = set(list1) # this is a hash set`
    
3.     `result = []`
    
4.     `for y in list2:`
    
5.         `if y in set1:`
    
6.             `result.append(y)`
    
7.     `return result`
    

假设 `list1` 和 `list2` 都是大约 10,000,000 个不同元素的列表，这样的元素数量可以说是很大了！

那么为什么它还能够运行得如此之快呢？因为 hashmap！！！

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

hashmap 查找是即时的（“常数级时间”）

我们看一下快速版程序中的 `if` 语句：

1. `if y in set1:`
    
2.     `result.append(y)`
    

你可能会认为如果 `set1` 包含 1000 万个元素，那么这个查找——`if y in set1` 会比 `set1` 包含 1000 个元素时慢。但事实并非如此！无论 `set1` 有多大，所需时间基本是相同的（超级快）。

这是因为 `set1` 是一个哈希集合，它是一种只有键没有值的 hashmap（hashtable）结构。

我不准备在本文中解释 为什么 hashmap 查找是即时的，但是神奇的 Vaidehi Joshi 的 [basecs](https://mp.weixin.qq.com/s?__biz=MzI1NDQwNDYyMg==&mid=2247489714&idx=1&sn=8cc4b9085c3ab12a10ebb626405f5196&chksm=e9c4e9d3deb360c56dbb84c7ad0978f11da779d306ba51d42ce705096190b29f71769876c2fc&mpshare=1&scene=24&srcid=0919rVQS83V32vNxXWBYhADx&sharer_sharetime=1631981790071&sharer_shareid=5fb9813bfe9ffc983435bfc8d8c5e9ca&key=daf9bdc5abc4e8d07e94cad3380c9996009c7423f6790952f4d46ae22ab2383cf5eb425f19556d65575c735866592166bc5267e9f33b3ac43f3a763c45443dbbd610aed25b37378cab311daeffb43361e7efdfd2a9fa36e5ebbdd97f42cf20a53caf570d4d6595706907e43d0d1dac511864347121f6f1c3d7eb3ff2495d363f&ascene=0&uin=MTEwNTU1MjgwMw%3D%3D&devicetype=Windows+11+x64&version=63090b19&lang=zh_CN&countrycode=CN&exportkey=n_ChQIAhIQ8lMyxxlpCdxL%2F7kU1368MhLeAQIE97dBBAEAAAAAAGY%2FNgRrJPUAAAAOpnltbLcz9gKNyK89dVj0gntksHXkOhJZo84bCsZQXHxDRV2okMyIC1ejhz56uGACNe6dlN6zhB4jPo0A1arG7OVIAApD9NOIaimHmErcezhip55a1%2FtJJs%2BwyRKXu6U9GTdJcXEs6uIAzXBGXZ%2BJCOpQfJvOno1FHeYhBoaZ9FS%2FI0%2FgORrBF5hPqdY7Ng9o9vLEaCJI5Twt8u8c60vrrd4wp2gEd2B2OTUsNWZjYGnLVwx0ZxUIB%2B3cF48zRpWrssY2exn1yQ%3D%3D&acctmode=0&pass_ticket=bvXp11bs1eKKOOVqJOo81t7ZdkUH0oTzA%2BE6RYr8d8L6hV6K%2BxzmCkfYqRWQp8Cm&wx_header=1&fasttmpl_type=0&fasttmpl_fullversion=7351805-zh_CN-zip&fasttmpl_flag=1)🔗 medium.com 系列中有关于 [hash table](https://mp.weixin.qq.com/s?__biz=MzI1NDQwNDYyMg==&mid=2247489714&idx=1&sn=8cc4b9085c3ab12a10ebb626405f5196&chksm=e9c4e9d3deb360c56dbb84c7ad0978f11da779d306ba51d42ce705096190b29f71769876c2fc&mpshare=1&scene=24&srcid=0919rVQS83V32vNxXWBYhADx&sharer_sharetime=1631981790071&sharer_shareid=5fb9813bfe9ffc983435bfc8d8c5e9ca&key=daf9bdc5abc4e8d07e94cad3380c9996009c7423f6790952f4d46ae22ab2383cf5eb425f19556d65575c735866592166bc5267e9f33b3ac43f3a763c45443dbbd610aed25b37378cab311daeffb43361e7efdfd2a9fa36e5ebbdd97f42cf20a53caf570d4d6595706907e43d0d1dac511864347121f6f1c3d7eb3ff2495d363f&ascene=0&uin=MTEwNTU1MjgwMw%3D%3D&devicetype=Windows+11+x64&version=63090b19&lang=zh_CN&countrycode=CN&exportkey=n_ChQIAhIQ8lMyxxlpCdxL%2F7kU1368MhLeAQIE97dBBAEAAAAAAGY%2FNgRrJPUAAAAOpnltbLcz9gKNyK89dVj0gntksHXkOhJZo84bCsZQXHxDRV2okMyIC1ejhz56uGACNe6dlN6zhB4jPo0A1arG7OVIAApD9NOIaimHmErcezhip55a1%2FtJJs%2BwyRKXu6U9GTdJcXEs6uIAzXBGXZ%2BJCOpQfJvOno1FHeYhBoaZ9FS%2FI0%2FgORrBF5hPqdY7Ng9o9vLEaCJI5Twt8u8c60vrrd4wp2gEd2B2OTUsNWZjYGnLVwx0ZxUIB%2B3cF48zRpWrssY2exn1yQ%3D%3D&acctmode=0&pass_ticket=bvXp11bs1eKKOOVqJOo81t7ZdkUH0oTzA%2BE6RYr8d8L6hV6K%2BxzmCkfYqRWQp8Cm&wx_header=1&fasttmpl_type=0&fasttmpl_fullversion=7351805-zh_CN-zip&fasttmpl_flag=1)🔗 medium.com 和 [hash 函数](https://mp.weixin.qq.com/s?__biz=MzI1NDQwNDYyMg==&mid=2247489714&idx=1&sn=8cc4b9085c3ab12a10ebb626405f5196&chksm=e9c4e9d3deb360c56dbb84c7ad0978f11da779d306ba51d42ce705096190b29f71769876c2fc&mpshare=1&scene=24&srcid=0919rVQS83V32vNxXWBYhADx&sharer_sharetime=1631981790071&sharer_shareid=5fb9813bfe9ffc983435bfc8d8c5e9ca&key=daf9bdc5abc4e8d07e94cad3380c9996009c7423f6790952f4d46ae22ab2383cf5eb425f19556d65575c735866592166bc5267e9f33b3ac43f3a763c45443dbbd610aed25b37378cab311daeffb43361e7efdfd2a9fa36e5ebbdd97f42cf20a53caf570d4d6595706907e43d0d1dac511864347121f6f1c3d7eb3ff2495d363f&ascene=0&uin=MTEwNTU1MjgwMw%3D%3D&devicetype=Windows+11+x64&version=63090b19&lang=zh_CN&countrycode=CN&exportkey=n_ChQIAhIQ8lMyxxlpCdxL%2F7kU1368MhLeAQIE97dBBAEAAAAAAGY%2FNgRrJPUAAAAOpnltbLcz9gKNyK89dVj0gntksHXkOhJZo84bCsZQXHxDRV2okMyIC1ejhz56uGACNe6dlN6zhB4jPo0A1arG7OVIAApD9NOIaimHmErcezhip55a1%2FtJJs%2BwyRKXu6U9GTdJcXEs6uIAzXBGXZ%2BJCOpQfJvOno1FHeYhBoaZ9FS%2FI0%2FgORrBF5hPqdY7Ng9o9vLEaCJI5Twt8u8c60vrrd4wp2gEd2B2OTUsNWZjYGnLVwx0ZxUIB%2B3cF48zRpWrssY2exn1yQ%3D%3D&acctmode=0&pass_ticket=bvXp11bs1eKKOOVqJOo81t7ZdkUH0oTzA%2BE6RYr8d8L6hV6K%2BxzmCkfYqRWQp8Cm&wx_header=1&fasttmpl_type=0&fasttmpl_fullversion=7351805-zh_CN-zip&fasttmpl_flag=1)🔗 medium.com 的解释，其中讨论了 hashmap 即时查找的原因。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

不经意的二次方：现实中的二次算法！

二次时间算法真的很慢，我们看到的的这个问题实际上在现实中也会遇到——Nelson Elhage 有一个很棒的博客，名为 [不经意的二次方](https://mp.weixin.qq.com/s?__biz=MzI1NDQwNDYyMg==&mid=2247489714&idx=1&sn=8cc4b9085c3ab12a10ebb626405f5196&chksm=e9c4e9d3deb360c56dbb84c7ad0978f11da779d306ba51d42ce705096190b29f71769876c2fc&mpshare=1&scene=24&srcid=0919rVQS83V32vNxXWBYhADx&sharer_sharetime=1631981790071&sharer_shareid=5fb9813bfe9ffc983435bfc8d8c5e9ca&key=daf9bdc5abc4e8d07e94cad3380c9996009c7423f6790952f4d46ae22ab2383cf5eb425f19556d65575c735866592166bc5267e9f33b3ac43f3a763c45443dbbd610aed25b37378cab311daeffb43361e7efdfd2a9fa36e5ebbdd97f42cf20a53caf570d4d6595706907e43d0d1dac511864347121f6f1c3d7eb3ff2495d363f&ascene=0&uin=MTEwNTU1MjgwMw%3D%3D&devicetype=Windows+11+x64&version=63090b19&lang=zh_CN&countrycode=CN&exportkey=n_ChQIAhIQ8lMyxxlpCdxL%2F7kU1368MhLeAQIE97dBBAEAAAAAAGY%2FNgRrJPUAAAAOpnltbLcz9gKNyK89dVj0gntksHXkOhJZo84bCsZQXHxDRV2okMyIC1ejhz56uGACNe6dlN6zhB4jPo0A1arG7OVIAApD9NOIaimHmErcezhip55a1%2FtJJs%2BwyRKXu6U9GTdJcXEs6uIAzXBGXZ%2BJCOpQfJvOno1FHeYhBoaZ9FS%2FI0%2FgORrBF5hPqdY7Ng9o9vLEaCJI5Twt8u8c60vrrd4wp2gEd2B2OTUsNWZjYGnLVwx0ZxUIB%2B3cF48zRpWrssY2exn1yQ%3D%3D&acctmode=0&pass_ticket=bvXp11bs1eKKOOVqJOo81t7ZdkUH0oTzA%2BE6RYr8d8L6hV6K%2BxzmCkfYqRWQp8Cm&wx_header=1&fasttmpl_type=0&fasttmpl_fullversion=7351805-zh_CN-zip&fasttmpl_flag=1)🔗 accidentallyquadratic.tumblr.com，其中有关于不经意以二次时间算法运行代码导致性能问题的故事。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

二次时间算法可能会“偷袭”你

关于二次时间算法的奇怪之处在于，当你在少量元素（如 1000）上运行它们时，它看起来并没有那么糟糕！没那么慢！但是如果你给它 1,000,000 个元素，它真的会花费几个小时去运行。

所以我认为它还是值得深入了解的，这样你就可以避免无意中使用二次时间算法，特别是当有一种简单的方法来编写线性时间算法（例如使用 hashmap）时。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

总是让我感到一丝神奇的 hashmap

hashmap 当然不是魔法（你可以学习一下为什么 hashmap 查找是即时的！真的很酷！），但它总是让人 感觉 有点神奇，每次我在程序中使用 hashmap 来加速，都会使我感到开心 :)

---

via: [https://jvns.ca/blog/2021/09/10/hashmaps-make-things-fast/](https://mp.weixin.qq.com/s?__biz=MzI1NDQwNDYyMg==&mid=2247489714&idx=1&sn=8cc4b9085c3ab12a10ebb626405f5196&chksm=e9c4e9d3deb360c56dbb84c7ad0978f11da779d306ba51d42ce705096190b29f71769876c2fc&mpshare=1&scene=24&srcid=0919rVQS83V32vNxXWBYhADx&sharer_sharetime=1631981790071&sharer_shareid=5fb9813bfe9ffc983435bfc8d8c5e9ca&key=daf9bdc5abc4e8d07e94cad3380c9996009c7423f6790952f4d46ae22ab2383cf5eb425f19556d65575c735866592166bc5267e9f33b3ac43f3a763c45443dbbd610aed25b37378cab311daeffb43361e7efdfd2a9fa36e5ebbdd97f42cf20a53caf570d4d6595706907e43d0d1dac511864347121f6f1c3d7eb3ff2495d363f&ascene=0&uin=MTEwNTU1MjgwMw%3D%3D&devicetype=Windows+11+x64&version=63090b19&lang=zh_CN&countrycode=CN&exportkey=n_ChQIAhIQ8lMyxxlpCdxL%2F7kU1368MhLeAQIE97dBBAEAAAAAAGY%2FNgRrJPUAAAAOpnltbLcz9gKNyK89dVj0gntksHXkOhJZo84bCsZQXHxDRV2okMyIC1ejhz56uGACNe6dlN6zhB4jPo0A1arG7OVIAApD9NOIaimHmErcezhip55a1%2FtJJs%2BwyRKXu6U9GTdJcXEs6uIAzXBGXZ%2BJCOpQfJvOno1FHeYhBoaZ9FS%2FI0%2FgORrBF5hPqdY7Ng9o9vLEaCJI5Twt8u8c60vrrd4wp2gEd2B2OTUsNWZjYGnLVwx0ZxUIB%2B3cF48zRpWrssY2exn1yQ%3D%3D&acctmode=0&pass_ticket=bvXp11bs1eKKOOVqJOo81t7ZdkUH0oTzA%2BE6RYr8d8L6hV6K%2BxzmCkfYqRWQp8Cm&wx_header=1&fasttmpl_type=0&fasttmpl_fullversion=7351805-zh_CN-zip&fasttmpl_flag=1)

作者：[Julia Evans](https://mp.weixin.qq.com/s?__biz=MzI1NDQwNDYyMg==&mid=2247489714&idx=1&sn=8cc4b9085c3ab12a10ebb626405f5196&chksm=e9c4e9d3deb360c56dbb84c7ad0978f11da779d306ba51d42ce705096190b29f71769876c2fc&mpshare=1&scene=24&srcid=0919rVQS83V32vNxXWBYhADx&sharer_sharetime=1631981790071&sharer_shareid=5fb9813bfe9ffc983435bfc8d8c5e9ca&key=daf9bdc5abc4e8d07e94cad3380c9996009c7423f6790952f4d46ae22ab2383cf5eb425f19556d65575c735866592166bc5267e9f33b3ac43f3a763c45443dbbd610aed25b37378cab311daeffb43361e7efdfd2a9fa36e5ebbdd97f42cf20a53caf570d4d6595706907e43d0d1dac511864347121f6f1c3d7eb3ff2495d363f&ascene=0&uin=MTEwNTU1MjgwMw%3D%3D&devicetype=Windows+11+x64&version=63090b19&lang=zh_CN&countrycode=CN&exportkey=n_ChQIAhIQ8lMyxxlpCdxL%2F7kU1368MhLeAQIE97dBBAEAAAAAAGY%2FNgRrJPUAAAAOpnltbLcz9gKNyK89dVj0gntksHXkOhJZo84bCsZQXHxDRV2okMyIC1ejhz56uGACNe6dlN6zhB4jPo0A1arG7OVIAApD9NOIaimHmErcezhip55a1%2FtJJs%2BwyRKXu6U9GTdJcXEs6uIAzXBGXZ%2BJCOpQfJvOno1FHeYhBoaZ9FS%2FI0%2FgORrBF5hPqdY7Ng9o9vLEaCJI5Twt8u8c60vrrd4wp2gEd2B2OTUsNWZjYGnLVwx0ZxUIB%2B3cF48zRpWrssY2exn1yQ%3D%3D&acctmode=0&pass_ticket=bvXp11bs1eKKOOVqJOo81t7ZdkUH0oTzA%2BE6RYr8d8L6hV6K%2BxzmCkfYqRWQp8Cm&wx_header=1&fasttmpl_type=0&fasttmpl_fullversion=7351805-zh_CN-zip&fasttmpl_flag=1) 选题：[lujun9972](https://mp.weixin.qq.com/s?__biz=MzI1NDQwNDYyMg==&mid=2247489714&idx=1&sn=8cc4b9085c3ab12a10ebb626405f5196&chksm=e9c4e9d3deb360c56dbb84c7ad0978f11da779d306ba51d42ce705096190b29f71769876c2fc&mpshare=1&scene=24&srcid=0919rVQS83V32vNxXWBYhADx&sharer_sharetime=1631981790071&sharer_shareid=5fb9813bfe9ffc983435bfc8d8c5e9ca&key=daf9bdc5abc4e8d07e94cad3380c9996009c7423f6790952f4d46ae22ab2383cf5eb425f19556d65575c735866592166bc5267e9f33b3ac43f3a763c45443dbbd610aed25b37378cab311daeffb43361e7efdfd2a9fa36e5ebbdd97f42cf20a53caf570d4d6595706907e43d0d1dac511864347121f6f1c3d7eb3ff2495d363f&ascene=0&uin=MTEwNTU1MjgwMw%3D%3D&devicetype=Windows+11+x64&version=63090b19&lang=zh_CN&countrycode=CN&exportkey=n_ChQIAhIQ8lMyxxlpCdxL%2F7kU1368MhLeAQIE97dBBAEAAAAAAGY%2FNgRrJPUAAAAOpnltbLcz9gKNyK89dVj0gntksHXkOhJZo84bCsZQXHxDRV2okMyIC1ejhz56uGACNe6dlN6zhB4jPo0A1arG7OVIAApD9NOIaimHmErcezhip55a1%2FtJJs%2BwyRKXu6U9GTdJcXEs6uIAzXBGXZ%2BJCOpQfJvOno1FHeYhBoaZ9FS%2FI0%2FgORrBF5hPqdY7Ng9o9vLEaCJI5Twt8u8c60vrrd4wp2gEd2B2OTUsNWZjYGnLVwx0ZxUIB%2B3cF48zRpWrssY2exn1yQ%3D%3D&acctmode=0&pass_ticket=bvXp11bs1eKKOOVqJOo81t7ZdkUH0oTzA%2BE6RYr8d8L6hV6K%2BxzmCkfYqRWQp8Cm&wx_header=1&fasttmpl_type=0&fasttmpl_fullversion=7351805-zh_CN-zip&fasttmpl_flag=1) 译者：[unigeorge](https://mp.weixin.qq.com/s?__biz=MzI1NDQwNDYyMg==&mid=2247489714&idx=1&sn=8cc4b9085c3ab12a10ebb626405f5196&chksm=e9c4e9d3deb360c56dbb84c7ad0978f11da779d306ba51d42ce705096190b29f71769876c2fc&mpshare=1&scene=24&srcid=0919rVQS83V32vNxXWBYhADx&sharer_sharetime=1631981790071&sharer_shareid=5fb9813bfe9ffc983435bfc8d8c5e9ca&key=daf9bdc5abc4e8d07e94cad3380c9996009c7423f6790952f4d46ae22ab2383cf5eb425f19556d65575c735866592166bc5267e9f33b3ac43f3a763c45443dbbd610aed25b37378cab311daeffb43361e7efdfd2a9fa36e5ebbdd97f42cf20a53caf570d4d6595706907e43d0d1dac511864347121f6f1c3d7eb3ff2495d363f&ascene=0&uin=MTEwNTU1MjgwMw%3D%3D&devicetype=Windows+11+x64&version=63090b19&lang=zh_CN&countrycode=CN&exportkey=n_ChQIAhIQ8lMyxxlpCdxL%2F7kU1368MhLeAQIE97dBBAEAAAAAAGY%2FNgRrJPUAAAAOpnltbLcz9gKNyK89dVj0gntksHXkOhJZo84bCsZQXHxDRV2okMyIC1ejhz56uGACNe6dlN6zhB4jPo0A1arG7OVIAApD9NOIaimHmErcezhip55a1%2FtJJs%2BwyRKXu6U9GTdJcXEs6uIAzXBGXZ%2BJCOpQfJvOno1FHeYhBoaZ9FS%2FI0%2FgORrBF5hPqdY7Ng9o9vLEaCJI5Twt8u8c60vrrd4wp2gEd2B2OTUsNWZjYGnLVwx0ZxUIB%2B3cF48zRpWrssY2exn1yQ%3D%3D&acctmode=0&pass_ticket=bvXp11bs1eKKOOVqJOo81t7ZdkUH0oTzA%2BE6RYr8d8L6hV6K%2BxzmCkfYqRWQp8Cm&wx_header=1&fasttmpl_type=0&fasttmpl_fullversion=7351805-zh_CN-zip&fasttmpl_flag=1) 校对：[wxy](https://mp.weixin.qq.com/s?__biz=MzI1NDQwNDYyMg==&mid=2247489714&idx=1&sn=8cc4b9085c3ab12a10ebb626405f5196&chksm=e9c4e9d3deb360c56dbb84c7ad0978f11da779d306ba51d42ce705096190b29f71769876c2fc&mpshare=1&scene=24&srcid=0919rVQS83V32vNxXWBYhADx&sharer_sharetime=1631981790071&sharer_shareid=5fb9813bfe9ffc983435bfc8d8c5e9ca&key=daf9bdc5abc4e8d07e94cad3380c9996009c7423f6790952f4d46ae22ab2383cf5eb425f19556d65575c735866592166bc5267e9f33b3ac43f3a763c45443dbbd610aed25b37378cab311daeffb43361e7efdfd2a9fa36e5ebbdd97f42cf20a53caf570d4d6595706907e43d0d1dac511864347121f6f1c3d7eb3ff2495d363f&ascene=0&uin=MTEwNTU1MjgwMw%3D%3D&devicetype=Windows+11+x64&version=63090b19&lang=zh_CN&countrycode=CN&exportkey=n_ChQIAhIQ8lMyxxlpCdxL%2F7kU1368MhLeAQIE97dBBAEAAAAAAGY%2FNgRrJPUAAAAOpnltbLcz9gKNyK89dVj0gntksHXkOhJZo84bCsZQXHxDRV2okMyIC1ejhz56uGACNe6dlN6zhB4jPo0A1arG7OVIAApD9NOIaimHmErcezhip55a1%2FtJJs%2BwyRKXu6U9GTdJcXEs6uIAzXBGXZ%2BJCOpQfJvOno1FHeYhBoaZ9FS%2FI0%2FgORrBF5hPqdY7Ng9o9vLEaCJI5Twt8u8c60vrrd4wp2gEd2B2OTUsNWZjYGnLVwx0ZxUIB%2B3cF48zRpWrssY2exn1yQ%3D%3D&acctmode=0&pass_ticket=bvXp11bs1eKKOOVqJOo81t7ZdkUH0oTzA%2BE6RYr8d8L6hV6K%2BxzmCkfYqRWQp8Cm&wx_header=1&fasttmpl_type=0&fasttmpl_fullversion=7351805-zh_CN-zip&fasttmpl_flag=1)

本文由 [LCTT](https://mp.weixin.qq.com/s?__biz=MzI1NDQwNDYyMg==&mid=2247489714&idx=1&sn=8cc4b9085c3ab12a10ebb626405f5196&chksm=e9c4e9d3deb360c56dbb84c7ad0978f11da779d306ba51d42ce705096190b29f71769876c2fc&mpshare=1&scene=24&srcid=0919rVQS83V32vNxXWBYhADx&sharer_sharetime=1631981790071&sharer_shareid=5fb9813bfe9ffc983435bfc8d8c5e9ca&key=daf9bdc5abc4e8d07e94cad3380c9996009c7423f6790952f4d46ae22ab2383cf5eb425f19556d65575c735866592166bc5267e9f33b3ac43f3a763c45443dbbd610aed25b37378cab311daeffb43361e7efdfd2a9fa36e5ebbdd97f42cf20a53caf570d4d6595706907e43d0d1dac511864347121f6f1c3d7eb3ff2495d363f&ascene=0&uin=MTEwNTU1MjgwMw%3D%3D&devicetype=Windows+11+x64&version=63090b19&lang=zh_CN&countrycode=CN&exportkey=n_ChQIAhIQ8lMyxxlpCdxL%2F7kU1368MhLeAQIE97dBBAEAAAAAAGY%2FNgRrJPUAAAAOpnltbLcz9gKNyK89dVj0gntksHXkOhJZo84bCsZQXHxDRV2okMyIC1ejhz56uGACNe6dlN6zhB4jPo0A1arG7OVIAApD9NOIaimHmErcezhip55a1%2FtJJs%2BwyRKXu6U9GTdJcXEs6uIAzXBGXZ%2BJCOpQfJvOno1FHeYhBoaZ9FS%2FI0%2FgORrBF5hPqdY7Ng9o9vLEaCJI5Twt8u8c60vrrd4wp2gEd2B2OTUsNWZjYGnLVwx0ZxUIB%2B3cF48zRpWrssY2exn1yQ%3D%3D&acctmode=0&pass_ticket=bvXp11bs1eKKOOVqJOo81t7ZdkUH0oTzA%2BE6RYr8d8L6hV6K%2BxzmCkfYqRWQp8Cm&wx_header=1&fasttmpl_type=0&fasttmpl_fullversion=7351805-zh_CN-zip&fasttmpl_flag=1) 原创编译，[Linux中国](https://mp.weixin.qq.com/s?__biz=MzI1NDQwNDYyMg==&mid=2247489714&idx=1&sn=8cc4b9085c3ab12a10ebb626405f5196&chksm=e9c4e9d3deb360c56dbb84c7ad0978f11da779d306ba51d42ce705096190b29f71769876c2fc&mpshare=1&scene=24&srcid=0919rVQS83V32vNxXWBYhADx&sharer_sharetime=1631981790071&sharer_shareid=5fb9813bfe9ffc983435bfc8d8c5e9ca&key=daf9bdc5abc4e8d07e94cad3380c9996009c7423f6790952f4d46ae22ab2383cf5eb425f19556d65575c735866592166bc5267e9f33b3ac43f3a763c45443dbbd610aed25b37378cab311daeffb43361e7efdfd2a9fa36e5ebbdd97f42cf20a53caf570d4d6595706907e43d0d1dac511864347121f6f1c3d7eb3ff2495d363f&ascene=0&uin=MTEwNTU1MjgwMw%3D%3D&devicetype=Windows+11+x64&version=63090b19&lang=zh_CN&countrycode=CN&exportkey=n_ChQIAhIQ8lMyxxlpCdxL%2F7kU1368MhLeAQIE97dBBAEAAAAAAGY%2FNgRrJPUAAAAOpnltbLcz9gKNyK89dVj0gntksHXkOhJZo84bCsZQXHxDRV2okMyIC1ejhz56uGACNe6dlN6zhB4jPo0A1arG7OVIAApD9NOIaimHmErcezhip55a1%2FtJJs%2BwyRKXu6U9GTdJcXEs6uIAzXBGXZ%2BJCOpQfJvOno1FHeYhBoaZ9FS%2FI0%2FgORrBF5hPqdY7Ng9o9vLEaCJI5Twt8u8c60vrrd4wp2gEd2B2OTUsNWZjYGnLVwx0ZxUIB%2B3cF48zRpWrssY2exn1yQ%3D%3D&acctmode=0&pass_ticket=bvXp11bs1eKKOOVqJOo81t7ZdkUH0oTzA%2BE6RYr8d8L6hV6K%2BxzmCkfYqRWQp8Cm&wx_header=1&fasttmpl_type=0&fasttmpl_fullversion=7351805-zh_CN-zip&fasttmpl_flag=1) 荣誉推出

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)  

欢迎遵照 CC-BY-NC-SA 协议规定转载，

如需转载，请在文章下留言 “转载：公众号名称”，

我们将为您添加白名单，授权“转载文章时可以修改”。

[![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)](https://mp.weixin.qq.com/s?__biz=MjM5NjQ4MjYwMQ==&mid=2664637017&idx=1&sn=418656faabf31e5a8ba301425bea0db0&scene=21#wechat_redirect)

  

阅读原文

阅读 840

​

写留言

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/eQYGxygoIB8CJDPsichXNx5N4GcUbMotRrxjQ2WfQNpb84lSPl8xQklj89XqO22icNpuDfFyctxkfoNaDibCiceXdA/300?wx_fmt=png&wxfrom=18)

Linux

赞分享1

写留言

写留言

**留言**

暂无留言