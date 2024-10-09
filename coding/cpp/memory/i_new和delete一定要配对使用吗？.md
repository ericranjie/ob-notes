在平时资料中，我们常看到：**new和delete，new\[\]和delete\[\]一定要配对使用！**

也有人说：\*\*有时候不配对使用也不会出现问题。\*\*也许你也是只知其然，不知其所以然，然而我也有点懵了\_(¦3」∠)\_

[https://mmbiz.qpic.cn/mmbiz_gif/9lFFFiaKpEr8hHXsJKr7ByYvWNNiahicJL0TPgLgstH398FpPibDpSaeQvyBHqZrQLqOPNEC2nTaTEiclLYZaBJv8Qw/640?wx_fmt=gif&tp=webp&wxfrom=5&wx_lazy=1](https://mmbiz.qpic.cn/mmbiz_gif/9lFFFiaKpEr8hHXsJKr7ByYvWNNiahicJL0TPgLgstH398FpPibDpSaeQvyBHqZrQLqOPNEC2nTaTEiclLYZaBJv8Qw/640?wx_fmt=gif&tp=webp&wxfrom=5&wx_lazy=1)

那就研究下这个问题：

首先，看下这段配对使用代码：

```
#include <stdlib.h>#include <iostream>using namespace std;class inner {   public:    inner() { cout << "Constructing" << endl; }    ~inner() { cout << "Destructing" << endl; }};
int main(int argc, char *argv[]) {    inner *p = new inner();    inner *pa = new inner[2];
    delete p;    delete []pa;
    return 0;}
程序输出：ConstructingConstructingConstructingDestructingDestructingDestructing
```

因为new\[\]会创建一个数组，一个对象数组需要一定的空间大小，假设一个对象需要N字节大小，K个对象的数组就需要K\*N个空间来构造对象数组，但是在delete\[\]时候，如何知道数组的长度呢？

所以new\[\]会在K_N个空间的基础上，头部多申请4个字节，用于存储数组长度，这样delete\[\]时候才知道对象数组的大小，才会相应调用K次析构函数，并且释放K_N+4大小的内存。

这是我们平时编程中经常配对使用的情况，如果不配对使用呢？

**new\[\]与delete结对使用**

```
#include <stdlib.h>#include <iostream>using namespace std;class inner {   public:    inner() { cout << "Constructing" << endl; }    ~inner() { cout << "Destructing" << endl; }};
int main(int argc, char *argv[]) {    inner *p = new inner[2];    delete p;    return 0;}
程序输出：ConstructingConstructingDestructingmunmap_chunk(): invalid pointerAborted (core dumped)
```

这里发现：程序挂掉了。

并且，只调用了一次析构函数，为什么呢？

因为我们使用了delete，delete不同于delete\[\]，它认为这只是一个对象占用的空间，不是对象数组，不会访问前4个字节获取长度，所以只调用了一次析构函数。而且，最后释放内存的时候只释放了起始地址为A的内存。然而这不是这一整块内存的起始地址，整块内存的起始地址应该是A-4，释放内存如果不从内存起始地址操作就会出现断错误，所以导致程序挂掉。

关于内存知识可以看我以前的文章：

[10张图22段代码，万字长文带你搞懂虚拟内存模型和malloc内部原理](http://mp.weixin.qq.com/s?__biz=MzkyODE5NjU2Mw==&mid=2247484726&idx=1&sn=18d9dc7f8b76a2a9a0b29b39ff6dabea&chksm=c21d378af56abe9c56f3d4da55b4d2d90995bae0e1a4a13c3e5cf7f33473d2642615b445aeb1&scene=21#wechat_redirect)

**new和delete\[\]结对使用**

```
#include <stdlib.h>#include <iostream>using namespace std;class inner {   public:    inner() { cout << "Constructing" << endl; }    ~inner() { cout << "Destructing" << endl; }};
int main(int argc, char *argv[]) {    inner *p = new inner();    delete []p;    return 0;}程序输出：ConstructingDestructingDestructingDestructingDestructingDestructingDestructing...Destructingfree(): invalid pointerAborted (core dumped)
```

这里调用了不定次数的析构函数，并且挂掉，是因为在new时候没有多申请4个字节存储长度，而delete\[\]时候还会向前找4个字节获取长度，这4个字节是未定义的，所以调用了不固定次数的析构函数，释放内存的时候也释放了起始地址为A-4的内存，而正常的起始地址应该是A，所以程序挂掉。

**什么时候可以不配对使用？**

我们再来看一段代码：

```
#include <stdlib.h>#include <iostream>using namespace std;
int main() {    int *pint = new int(5);    delete[] pint;    int *pinta = new int[4];    delete pinta;    cout << "success" << endl;    return 0;}程序输出：success
```

这段代码即使不配对使用也会正常运行，这是为什么呢，因为int是内置类型，new\[\]和delete\[\]在配合int使用时知道int是内置类型，不需要析构函数，所以也就不需要多4个字节来存放数组长度，只需要直接操作内存即可。

总结

当类型为int, float等内置类型时，new、delete、new\[\]、delete\[\]不需要配对使用；

当是自定义类型时，new、delete和new\[\]、delete\[\]才需要配对使用。

当然，我们平时编程过程中，为了保证代码的可读性，以及养成良好的编程习惯，最好确保所有情况都配对使用。

其实侯捷老师的视频里也讲过这个问题，大家也可以去看看侯捷巨佬的C++视频。
