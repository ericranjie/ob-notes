# 

Linux爱好者

_2021年12月12日 20:00_

The following article is from 编程往事 Author 果冻虾仁

\[

![](http://wx.qlogo.cn/mmhead/Q3auHgzwzM7Hvg7rQmorRljlcVCzwYTttaruhY8OCBSft64AYB32Cg/0)

**编程往事**.

C++码农，brpc committer，搜广推在线工程。专注互联网后端技术分享、行业观察以及个人成长！也欢迎关注我的知乎：果冻虾仁

\](https://mp.weixin.qq.com/s?\_\_biz=MzAxODI5ODMwOA==&mid=2666559792&idx=3&sn=0fdc384891ff2c21843a490e2f0f5095&chksm=80dcbd9bb7ab348de30537b8718a419fe10a72b80c0b441a3b659b9a6d3b1518d5a7f50073de&mpshare=1&scene=24&srcid=1212UXu2rmgWgHg6OmiXEXsF&sharer_sharetime=1639315798062&sharer_shareid=5fb9813bfe9ffc983435bfc8d8c5e9ca&key=daf9bdc5abc4e8d028237f365b31c5b04ae0575ff675ee2432a9ef936bd93c8e146b8a6bf92f0dc15ecd7e44b336d2942a43e5cbdf34e0b04e530bb722edfc9b2e6623f422d9886081e18fadca3accc443e8e04dd78d49dcc4d2c484ccc3e6a3395282333cd226c8b84be629cee17a5b6be55bd9d0079dac00ede4bd919b6f88&ascene=14&uin=MTEwNTU1MjgwMw%3D%3D&devicetype=iMac+MacBookAir10%2C1+OSX+OSX+14.6.1+build(23G93)&version=13080710&nettype=WIFI&lang=en&session_us=gh_9f1efcd6f4ab&countrycode=CN&fontScale=100&exportkey=n_ChQIAhIQ48U9W0ttqV5UwYWTWIWSJRKUAgIE97dBBAEAAAAAAArxAHdmoR4AAAAOpnltbLcz9gKNyK89dVj0I%2BlVkXZHbfNH0%2F%2BDGn0C2ooGsUp4jpeQxZyZtx%2FLeqqg6AOMCoKGz%2B8oSXc2GUV1sAeMHPQnG%2FD0%2FLkTKM3KxcEKl9ADLX7vUXNwikrSkXmzHjmpjp8Y4zBcUega7KOk6MrPFvD7bUyTiSA4qNJ1nfcOJeuuanx%2Fo2mq3fTvXrJGXghAsq3hkjsT7%2BAW71iyjHP9dmx7OtTf0FsXwePJ%2FUwooVFGC0XDpIfWffvKilTudNWi2O2V7dt20psyHgWa3Foogb2HLVsR1X%2F%2FFFwiVfjGGL9wQi9FK7Au9lM%2BXgRtFIZT5%2B40GwGUVToq2A%3D%3D&acctmode=0&pass_ticket=Vxrz4OoOX%2F5vwS3XtsdfdVOagkR%2F2K0OhqP6JCJFeL75eeqHuiRGR1qaZ2z4rOWj&wx_header=0#)

# 故事的开始

某年某月的某一天，组里新来了一个工作多年的专家工程师。领导让其在我当前负责的模块上做一些优化工作。很快专家提出来很多C++语法上的修改意见。比如：

`vector<string> foo(Context* context) {       vector<string> v;       ... // 给v赋值       return v;   }   `

建议改成：

`void foo(Context* context, vector<string>& v;) {       ... // 给v赋值   }   `

其理由是按值返回STL容器对象，会产生拷贝。

我内心万马奔腾：

> - 如果我们是C++98，说这个意见，或许还能理解。但现在是2021年，项目用的C++版本是C++11，这个修改却并不正确！
>
> - 即便是C++98，编译器其实也对此有`NRVO`、`RVO`的优化，避免拷贝，只要你不去主动关闭优化，基本都能享受到。

类似的问题在StackOverflow上早有讨论。

> https://stackoverflow.com/questions/4986673/c11-rvalues-and-move-semantics-confusion-return-statement

# NRVO、RVO与 copy elision

我再来稍微展开一下，C++11开始当按值返回的时候，自动尝试使用move语义，而非拷贝语义，被称为`copy elision`（复制消除）。而在C++11之前有`RVO`（返回值优化）或`NRVO`（具名返回值优化），C++11以后也同样存在。都能提高C++函数返回时的效率，减少冗余的拷贝。举个例子这段代码：

`#include <iostream>   #include <vector>   using namespace std;   vector<int> foo(int n) {       vector<int> v;       for (int i = 1; i <= n; i++) {           v.push_back(i);       }       cout <<&v<<endl;       return v;   }      int main() {       vector<int> v = foo(10);       cout <<&v<<endl;   }   `

使用C++98和C++11分别编译：

`g++ rvo.cpp -std=c++98 -o 98.out   g++ rvo.cpp -std=c++11 -o 11.out   `

分别运行：

`./98.out   0x7ffc680bf490   0x7ffc680bf490      ./11.out   0x7ffc5e871300   0x7ffc5e871300   `

可以看出函数内的临时对象和函数外接收这个返回值的对象是同一个地址，也就是说没有产生拷贝构造。但是按C++11之前标准这里应该是拷贝构造，这一优化就是`NRVO`，当然这属于编译器厂商们自己做的优化（即使不开O1、O2这种优化，也会默认做），是非标的。注意这并不是C++11标准要求的`copy elision`。

另外提一句什么是`RVO`呢？如果是返回没有名字的匿名对象，编译器对其做同样的优化就是`RVO`。比如：

`vector<int> bar() {       int x = 0;       int y = 0;       int z = 0;       ... // 修改了x，y，z的值       return {x, y, z}   }   `

来回到之前的例子，我们关闭`NRVO`来看看，给g++加上一个参数`-fno-elide-constructors`即可。

`g++ rvo.cpp -std=c++98 -fno-elide-constructors -o 98.out   g++ rvo.cpp -std=c++11 -fno-elide-constructors -o 11.out   `

再执行看看：

`./98.out   0x7ffc0988eac0   0x7ffc0988eb00      ./11.out   0x7fff39efc750   0x7fff39efc790   `

去掉`NRVO`后，可以看到二者不是同一个对象了。但其实对于C++11的代码而言，这其中仍然有`copy elision`，也就是说会自动执行`move`语义，我们改下测试代码：

`#include <iostream>   #include <vector>   using namespace std;   vector<int> foo(int n) {       vector<int> v;       for (int i = 1; i <= n; i++) {           v.push_back(i);       }       cout << "obj stack addr: "<< &v << " in foo" <<endl;       cout << "obj data  addr: "<< v.data() << " in foo" <<endl;       return v;   }      int main() {       vector<int> v = foo(10);       cout << "obj stack addr: "<< &v << " in main" <<endl;       cout << "obj data  addr: "<< v.data() << " in main" <<endl;   }   `

然后重新携带`-fno-elide-constructors`参数分别编译执行。

`./98.out   obj stack addr: 0x7ffc1301c090 in foo   obj data addr: 0x55b81763af20 in foo   obj stack addr: 0x7ffc1301c0d0 in main   obj data addr: 0x55b81763b380 in main         ./11.out   obj stack addr: 0x7ffeb4acac30 in foo   obj data addr: 0x556ecd26ef20 in foo   obj stack addr: 0x7ffeb4acac70 in main   obj data addr: 0x556ecd26ef20 in main   `

可以看出，尽管C++11去掉了`NRVO`以后，main函数中的对象v和foo函数中的对象v不是同一个。但他们中的data()指向的数据地址是同一个。也就是说C++11开始，你用函数按值返回一个STL容器，即使没有显式地加move，也会自动按`move`语义走，进行数据指针的修改，而不会拷贝全部的数据。

当然`copy elision`并不是只针对STL容器类型啦，所有有move语义的对象类型都可以。但当没有`move`语义时，如果去掉NRVO还是会执行拷贝的。

再看个自定义类型的代码：

`#include <iostream>   #include <vector>   using namespace std;   class A {   public:       A() {           cout << this << " construct " <<endl;           _data = new int[size];       }       A(const A& a) {           cout << this << " copy from " <<&a <<endl;           _data = new int[a._len];           for (size_t i = 0; i < a._len; i++) {               this->_data[i] = a._data[i];           }       }       ~A() {           if (_data) {               delete[] _data;           }       }       bool push_back(int e) {           if (_len == size) {               return false;           }           _data[_len++] = e;           return true;       }       int* data() {           return _data;       }       size_t length() {           return _len;       }   private:       static const int size = 100;          int* _data = nullptr;       size_t _len = 0;   };   A foo(int n) {       A a;       for (int i = 1; i <= n; i++) {           a.push_back(i);       }       cout << "obj stack addr: "<< &a << " in foo" <<endl;       //cout << "obj data  addr: "<< a.data() << " in foo" <<endl;       return a;   }      int main() {       A a = foo(10);       cout << "obj stack addr: "<< &a << " in main" <<endl;       //cout << "obj data  addr: "<< a.data() << " in main" <<endl;   }   `

去掉NRVO用C++11编译。

`g++ rvo.cpp -std=c++11 -fno-elide-constructors -o 11.out   `

执行：

`./11.out   0x7ffcdca8fe80 construct   obj stack addr: 0x7ffcdca8fe80 in foo   0x7ffcdca8fec0 copy from 0x7ffcdca8fe80   0x7ffcdca8feb0 copy from 0x7ffcdca8fec0   obj stack addr: 0x7ffcdca8feb0 in main   `

可以看到由于我们自定义的类型A没有move语义，所以这里调用了拷贝构造函数，并且调用了两次。第一次是在foo函数内从具名的对象a，拷贝到临时变量作为返回值。第二次是从该返回值拷贝到main函数中的对象a。

我们来给他加上move构造函数：

`class A {   public:       A() {           cout << this << " construct " <<endl;           _data = new int[size];       }       A(const A& a) {           cout << this << " copy from " <<&a <<endl;           _data = new int[a._len];           for (size_t i = 0; i < a._len; i++) {               this->_data[i] = a._data[i];           }       }       A(A&& a) {           cout << this << " move data from " <<&a <<endl;           _data = a._data;           a._data = nullptr;           // 或使用交换           // swap(_data, a._data);       }       ~A() {           if (_data) {               delete[] _data;           }       }   ...   `

重新编译：

`g++ rvo.cpp -std=c++11 -fno-elide-constructors -o 11.out   `

然后运行：

`0x7ffe84ad74c0 construct   obj stack addr: 0x7ffe84ad74c0 in foo   0x7ffe84ad7510 move data from 0x7ffe84ad74c0   0x7ffe84ad7500 move data from 0x7ffe84ad7510   obj stack addr: 0x7ffe84ad7500 in main   `

可以看调用到了move构造函数。

# 故事的最后

听完专家的一系列修改意见之后，我觉得还是我自己优化更靠谱一些。这些语法上的问题，其实能优化的我基本都优化过了，没办法从语法上再拿到太多性能增益了。我感觉还是要从策略与逻辑入手，去寻找优化点。很快，一个月内，我连续两次给这个模块的耗时做了提升，999分位减少了60ms。接着我继续做该模块的负责人，专家被安排到其他“人力不足”的模块去帮忙了。

但自此我还是免不得多了一个习惯，在按值返回容器的函数上加一个注释：

`// It's OK in C++11!   vector<string> foo(Context* context) {       vector<string> v;       ... // 给v赋值       return v;   }   `

- EOF -

推荐阅读  点击标题可跳转

1、[清华大学：2021 元宇宙研究报告！](http://mp.weixin.qq.com/s?__biz=MzAxODI5ODMwOA==&mid=2666559203&idx=2&sn=95fabc48d7c7b77ac3e0aaf1f033d3d1&chksm=80dcb248b7ab3b5ed54dbe4ab601a1a0ca540088f8b22f99e231cf1d6f824391a51c174ef622&scene=21#wechat_redirect)[](http://mp.weixin.qq.com/s?__biz=MzAxODI5ODMwOA==&mid=2666559203&idx=2&sn=95fabc48d7c7b77ac3e0aaf1f033d3d1&chksm=80dcb248b7ab3b5ed54dbe4ab601a1a0ca540088f8b22f99e231cf1d6f824391a51c174ef622&scene=21#wechat_redirect)

2、[10 分钟看懂 Docker 和 K8S](http://mp.weixin.qq.com/s?__biz=MzAxODI5ODMwOA==&mid=2666558962&idx=1&sn=ede9eea879107df21557e1b9dcfeeb28&chksm=80dcb159b7ab384f22e68fc32d5556afd3e98c29d08757a871582d526bf5985b0c7c3fd3ec6d&scene=21#wechat_redirect)

3、[为什么腾讯/阿里不去开发被卡脖子的工业软件？](http://mp.weixin.qq.com/s?__biz=MzAxODI5ODMwOA==&mid=2666559138&idx=1&sn=fa0dca517b897bce61ea77194d4292e9&chksm=80dcb209b7ab3b1fc8e5cdb88c85f4d59f9867edbd45a1183116af87c86b8604341765eb9272&scene=21#wechat_redirect)

看完本文有收获？请分享给更多人

推荐关注「Linux 爱好者」，提升Linux技能

![](http://mmbiz.qpic.cn/mmbiz_png/9aPYe0E1fb3sjicd8JxDra10FRIqT54Zke2sfhibTDdtdnVhv5Qh3wLHZmKPjiaD7piahMAzIH6Cnltd1Nco17Ihjw/300?wx_fmt=png&wxfrom=19)

**Linux爱好者**

点击获取《每天一个Linux命令》系列和精选Linux技术资源。「Linux爱好者」日常分享 Linux/Unix 相关内容，包括：工具资源、使用技巧、课程书籍等。

75篇原创内容

公众号

点赞和在看就是最大的支持❤️

Reads 599

​

Comment

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/9aPYe0E1fb3sjicd8JxDra10FRIqT54Zke2sfhibTDdtdnVhv5Qh3wLHZmKPjiaD7piahMAzIH6Cnltd1Nco17Ihjw/300?wx_fmt=png&wxfrom=18)

Linux爱好者

6Share1

Comment

Comment

**Comment**

暂无留言
