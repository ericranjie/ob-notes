C++11 之前，C++ 语言没有对并发编程提供语言级别的支持，这使得我们在编写可移植的并发程序时，存在诸多的不便。现在 C++11 中增加了线程以及线程相关的类，很方便地支持了并发编程，使得编写的多线程程序的可移植性得到了很大的提高。

C++11 中提供的线程类叫做 `std::thread`，基于这个类创建一个新的线程非常的简单，只需要提供线程函数或者函数对象即可，并且可以同时指定线程函数的参数。我们首先来了解一下这个类提供的一些常用 API：

## **1. 构造函数**

```cpp
// ①
thread() noexcept;
// ②
thread( thread&& other ) noexcept;
// ③
template< class Function, class... Args >
explicit thread( Function&& f, Args&&... args );
// ④
thread( const thread& ) = delete;

```

构造函数①：默认构造函，构造一个线程对象，在这个线程中不执行任何处理动作

构造函数②：移动构造函数，将 other 的线程所有权转移给新的 thread 对象。之后 other 不再表示执行线程。

构造函数③：创建线程对象，并在该线程中执行函数 f 中的业务逻辑，args 是要传递给函数 f 的参数

任务函数 f 的可选类型有很多，具体如下：

- 普通函数，类成员函数，匿名函数，仿函数（这些都是可调用对象类型）
- 可以是可调用对象包装器类型，也可以是使用绑定器绑定之后得到的类型（仿函数）

构造函数④：使用 =delete 显示删除拷贝构造，不允许线程对象之间的拷贝

## **2. 公共成员函数**

### **2.1 get_id()**

应用程序启动之后默认只有一个线程，这个线程一般称之为主线程或父线程，通过线程类创建出的线程一般称之为子线程，每个被创建出的线程实例都对应一个线程 ID，这个 ID 是唯一的，可以通过这个 ID 来区分和识别各个已经存在的线程实例，这个获取线程 ID 的函数叫做 `get_id()`，函数原型如下：

```cpp
std::thread::id get_id() const noexcept;
```

示例程序如下：

```cpp
#include <iostream>
#include <thread>
#include <chrono>
using namespace std;

void func(int num, string str)
{
    for (int i = 0; i < 10; ++i)
    {
        cout << "子线程: i = " << i << "num: "
             << num << ", str: " << str << endl;
    }
}

void func1()
{
    for (int i = 0; i < 10; ++i)
    {
        cout << "子线程: i = " << i << endl;
    }
}

int main()
{
    cout << "主线程的线程ID: " << this_thread::get_id() << endl;
    thread t(func, 520, "i love you");
    thread t1(func1);
    cout << "线程t 的线程ID: " << t.get_id() << endl;
    cout << "线程t1的线程ID: " << t1.get_id() << endl;
}

```

1. `thread t(func, 520, "i love you")`;：创建了子线程对象 t，func() 函数会在这个子线程中运行

- `func()` 是一个回调函数，线程启动之后就会执行这个任务函数，程序猿只需要实现即可
- `func()` 的参数是通过 thread 的参数进行传递的，`520,i love you` 都是调用 `func()` 需要的实参
- 线程类的构造函数③ 是一个变参函数，因此无需担心线程任务函数的参数个数问题
- 任务函数 `func()` 一般返回值指定为 `void`，因为子线程在调用这个函数的时候不会处理其返回值

1. `thread t1(func1)`;：子线程对象 t1 中的任务函数`func1()`，没有参数，因此在线程构造函数中就无需指定了 通过线程对象调用 `get_id()` 就可以知道这个子线程的线程 ID 了，`t.get_id()`，`t1.get_id()`。
1. 基于命名空间 `this_thread` 得到当前线程的线程 ID

在上面的示例程序中有一个 bug，在主线程中依次创建出两个子线程，打印两个子线程的线程 ID，最后主线程执行完毕就退出了（主线程就是执行 `main ()` 函数的那个线程）。默认情况下，主线程销毁时会将与其关联的两个子线程也一并销毁，但是这时有可能子线程中的任务还没有执行完毕，最后也就得不到我们想要的结果了。

当启动了一个线程（创建了一个 thread 对象）之后，在这个线程结束的时候（`std::terminate ()`），我们如何去回收线程所使用的资源呢？thread 库给我们两种选择：

- 加入式（join()）
- 分离式（detach()）

另外，我们必须要在线程对象销毁之前在二者之间作出选择，否则程序运行期间就会有 bug 产生。

### **2.2 join()**

`join()` 字面意思是连接一个线程，意味着主动地等待线程的终止（线程阻塞）。在某个线程中通过子线程对象调用 `join()` 函数，调用这个函数的线程被阻塞，但是子线程对象中的任务函数会继续执行，当任务执行完毕之后 `join()` 会清理当前子线程中的相关资源然后返回，同时，调用该函数的线程解除阻塞继续向下执行。

再次强调，我们一定要搞清楚这个函数阻塞的是哪一个线程，函数在哪个线程中被执行，那么函数就阻塞哪个线程。该函数的函数原型如下：

```cpp
void join();
```

有了这样一个线程阻塞函数之后，就可以解决在上面测试程序中的 bug 了，如果要阻塞主线程的执行，只需要在主线程中通过子线程对象调用这个方法即可，当调用这个方法的子线程对象中的任务函数执行完毕之后，主线程的阻塞也就随之解除了。修改之后的示例代码如下：

```cpp
int main()
{
    cout << "主线程的线程ID: " << this_thread::get_id() << endl;
    thread t(func, 520, "i love you");
    thread t1(func1);
    cout << "线程t 的线程ID: " << t.get_id() << endl;
    cout << "线程t1的线程ID: " << t1.get_id() << endl;
    t.join();
    t1.join();
}
```

当主线程运行到第八行 `t.join()`;，根据子线程对象 t 的任务函数 `func()` 的执行情况，主线程会做如下处理：

- 如果任务函数 `func()` 还没执行完毕，主线程阻塞，直到任务执行完毕，主线程解除阻塞，继续向下运行
- 如果任务函数 `func()` 已经执行完毕，主线程不会阻塞，继续向下运行

同样，第 9 行的代码亦如此。

> 为了更好的理解 join() 的使用，再来给大家举一个例子，场景如下：

> 程序中一共有三个线程，其中两个子线程负责分段下载同一个文件，下载完毕之后，由主线程对这个文件进行下一步处理，那么示例程序就应该这么写：

```cpp
#include <iostream>
#include <thread>
#include <chrono>
using namespace std;

void download1()
{
    // 模拟下载, 总共耗时500ms，阻塞线程500ms
    this_thread::sleep_for(chrono::milliseconds(500));
    cout << "子线程1: " << this_thread::get_id() << ", 找到历史正文...." << endl;
}

void download2()
{
    // 模拟下载, 总共耗时300ms，阻塞线程300ms
    this_thread::sleep_for(chrono::milliseconds(300));
    cout << "子线程2: " << this_thread::get_id() << ", 找到历史正文...." << endl;
}

void doSomething()
{
    cout << "集齐历史正文, 呼叫罗宾...." << endl;
    cout << "历史正文解析中...." << endl;
    cout << "起航，前往拉夫德尔...." << endl;
    cout << "找到OnePiece, 成为海贼王, 哈哈哈!!!" << endl;
    cout << "若干年后，草帽全员卒...." << endl;
    cout << "大海贼时代再次被开启...." << endl;
}

int main()
{
    thread t1(download1);
    thread t2(download2);
    // 阻塞主线程，等待所有子线程任务执行完毕再继续向下执行
    t1.join();
    t2.join();
    doSomething();
}
```

示例程序输出的结果：

```cpp
子线程2: 72540, 找到历史正文....
子线程1: 79776, 找到历史正文....
集齐历史正文, 呼叫罗宾....
历史正文解析中....
起航，前往拉夫德尔....
找到OnePiece, 成为海贼王, 哈哈哈!!!
若干年后，草帽全员卒....
大海贼时代再次被开启....
```

在上面示例程序中最核心的处理是在主线程调用 `doSomething()`; 之前在第 35、36行通过子线程对象调用了 `join()` 方法，这样就能够保证两个子线程的任务都执行完毕了，也就是文件内容已经全部下载完成，主线程再对文件进行后续处理，如果子线程的文件没有下载完毕，主线程就去处理文件，很显然从逻辑上讲是有问题的。

### **2.3 detach()**

`detach()` 函数的作用是进行线程分离，分离主线程和创建出的子线程。在线程分离之后，主线程退出也会一并销毁创建出的所有子线程，在主线程退出之前，它可以脱离主线程继续独立的运行，任务执行完毕之后，这个子线程会自动释放自己占用的系统资源。（其实就是孩子翅膀硬了，和家里断绝关系，自己外出闯荡了，如果家里被诛九族还是会受牵连）。该函数函数原型如下：

```cpp
void detach();
```

线程分离函数没有参数也没有返回值，只需要在线程成功之后，通过线程对象调用该函数即可，继续将上面的测试程序修改一下：

```cpp
int main()
{
    cout << "主线程的线程ID: " << this_thread::get_id() << endl;
    thread t(func, 520, "i love you");
    thread t1(func1);
    cout << "线程t 的线程ID: " << t.get_id() << endl;
    cout << "线程t1的线程ID: " << t1.get_id() << endl;
    t.detach();
    t1.detach();
    // 让主线程休眠, 等待子线程执行完毕
    this_thread::sleep_for(chrono::seconds(5));
}
```

> 注意事项：线程分离函数 detach () 不会阻塞线程，子线程和主线程分离之后，在主线程中就不能再对这个子线程做任何控制了，比如：通过 join () 阻塞主线程等待子线程中的任务执行完毕，或者调用 get_id () 获取子线程的线程 ID。有利就有弊，鱼和熊掌不可兼得，建议使用 join ()。

### **2.5 joinable()**

`joinable()` 函数用于判断主线程和子线程是否处理关联（连接）状态，一般情况下，二者之间的关系处于关联状态，该函数返回一个布尔类型：

- 返回值为 true：主线程和子线程之间有关联（连接）关系
- 返回值为 false：主线程和子线程之间没有关联（连接）关系

```cpp
bool joinable() const noexcept;
```

示例代码如下：

```cpp
#include <iostream>
#include <thread>
#include <chrono>
using namespace std;

void foo()
{
    this_thread::sleep_for(std::chrono::seconds(1));
}

int main()
{
    thread t;
    cout << "before starting, joinable: " << t.joinable() << endl;

    t = thread(foo);
    cout << "after starting, joinable: " << t.joinable() << endl;

    t.join();
    cout << "after joining, joinable: " << t.joinable() << endl;

    thread t1(foo);
    cout << "after starting, joinable: " << t1.joinable() << endl;
    t1.detach();
    cout << "after detaching, joinable: " << t1.joinable() << endl;
}
```

示例代码打印的结果如下：

```cpp
before starting, joinable: 0
after starting, joinable: 1
after joining, joinable: 0
after starting, joinable: 1
after detaching, joinable: 0
```

基于示例代码打印的结果可以得到以下结论：

- 在创建的子线程对象的时候，如果没有指定任务函数，那么子线程不会启动，主线程和这个子线程也不会进行连接
- 在创建的子线程对象的时候，如果指定了任务函数，子线程启动并执行任务，主线程和这个子线程自动连接成功
- 子线程调用了`detach()`函数之后，父子线程分离，同时二者的连接断开，调用`joinable()`返回false
- 在子线程调用了`join()`函数，子线程中的任务函数继续执行，直到任务处理完毕，这时`join()`会清理（回收）当前子线程的相关资源，所以这个子线程和主线程的连接也就断开了，因此，调用`join()`之后再调用`joinable()`会返回false。

### **2.6 operator=**

线程中的资源是不能被复制的，因此通过 = 操作符进行赋值操作最终并不会得到两个完全相同的对象。

```cpp
// move (1)
thread& operator= (thread&& other) noexcept;
// copy [deleted] (2)
thread& operator= (const other&) = delete;
```

通过以上 = 操作符的重载声明可以得知：

- 如果 other 是一个右值，会进行资源所有权的转移
- 如果 other 不是右值，禁止拷贝，该函数被显示删除（=delete），不可用

## **3. 静态函数**

thread 线程类还提供了一个静态方法，用于获取当前计算机的 CPU 核心数，根据这个结果在程序中创建出数量相等的线程，每个线程独自占有一个 CPU 核心，这些线程就不用分时复用 CPU 时间片，此时程序的并发效率是最高的。

```cpp
static unsigned hardware_concurrency() noexcept;
```

示例代码如下：

```cpp
#include <iostream>
#include <thread>
using namespace std;

int main()
{
    int num = thread::hardware_concurrency();
    cout << "CPU number: " << num << endl;
}
```

## **4. C 线程库**

C 语言提供的线程库不论在 window 还是 Linux 操作系统中都是可以使用的，看明白了这些 C 语言中的线程函数之后会发现它和上面的 C++ 线程类使用很类似（其实就是基于面向对象的思想进行了封装），但 C++ 的线程类用起来更简单一些，链接奉上，感兴趣的可以一看。

[**C语言线程库的使用**](https://mp.weixin.qq.com/s?__biz=MzI3ODQ3OTczMw==&mid=2247491745&idx=1&sn=d995e1617ed6ad3d56de28b5be127e73&scene=21#wechat_redirect)

链接：[https://subingwen.com/cpp/thread/](https://subingwen.com/cpp/thread/)
