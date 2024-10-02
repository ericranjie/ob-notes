我们说的 Modern C++，一般指的是 C++11 及以后的标准，从 C++ 11 开始，Modern C++ 引入了大量的实用的特性，主要是两大方面，学习的时候也可以从这两大方面学习：

1. 增强或者改善的语法特性；
2. 新增的或者改善的 STL 库。

我们来看几个具体的案例：

### **案例 1：统一的类成员初始化语法与 std::initializer_list：**

在 C++98/03 中，假设我们要初始化一个类数组类型的成员（例如常用的清零操作），我们需要这么写：

```cpp
 class A
 {
 public:
     A()
     {
         //初始化arr
         arr[0] = 0;
         arr[1] = 0;
         arr[2] = 0;
         arr[3] = 0;
     }

 public:
     int arr[4];
 };
```

假设数组 arr 较长，我们可以使用循环或者借助 memset 函数去初始化，代码如下：

```cpp
 class A
 {
 public:
     A()
     {
         //使用循环初始化arr
         for (int i = 0; i < 4; i++)
             arr[i] = 0;
     }

 public:
     int arr[4];
 };

 class A
 {
 public:
     A()
     {
         //使用memset初始化arr
         memset(arr, 0, sizeof(arr));
     }

 public:
     int arr[4];
 };
```

但是，我们知道，在 C++98/08 中我们可以直接通过赋值操作来初始化一个数组的：

```cpp
 int arr[4] = { 0 };
```

但是对于作为类的成员变量的数组元素，C++98/03 是不允许我们这么做的。

到 C++11 中全部放开并统一了，在 C++11 中我们也可以使用这样的语法是初始化数组：

```cpp
 class A
 {
 public:
     //在 C++11中可以使用大括号语法初始化数组类型的成员变量
     A() : arr{0}
     {
     }

 public:
     int arr[4];
 };
```

如果你有兴趣，我们可以更进一步：

在 C++ 98/03 标准中，对类的成员必须使用 static const 修饰，而且类型必须是整型 （包括 bool、 char、 int、 long 等），这样才能使用这种初始化语法：

```cpp
 //C++98/03 在类定义处初始化成员变量
 class A
 {
 public:
     //T 的类型必须是整型，且必须使用 static const 修饰
     static const T t = 某个整型值;
 };
```

在 C++11 标准中就没有这种限制了，我们可以使用花括号（即{}）对任意类型的变 量进行初始化，而且不用是 static 类型：

```cpp
 //C++ 11 在类定义处初始化成员变量
 class A
 {
 public:
     //有没有一种Java初始化类成员变量的即视感^ _ ^
     bool ma{true};
     int mb{2019};
     std::string mc{"helloworld"};
 };
```

当然，在实际开发中，建议还是将这些成员变量的初始化统一写到构造函数的初始化列表中，方便阅读和维护代码。

### **案例 2：注解标签**

C++ 14 引入了 [[deprecated]] 标签来表示一个函数或者类型等已被弃用，在使用这些被弃用的函数或者类型并编译时， 编译器会给出相应的警告， 有的编译器直接生成编译错误：

```cpp
 [[deprecated]] void funcX();
```

这个标签在实际开发中非常有用，尤其在设计一些库代码时，如果库作者希望某个函数或者类型不想再被用户使用，则可以使用该标注标记。当然，我们也可以使用如下语法给出编译时的具体警告或者出错信息：

```cpp
 [[deprecated("use funY instead")]] void funcX();
```

有如下代码：

```cpp
#include <iostream>
 [[deprecated("use funcY instead")]] void funcX()
 {
     //实现省略
 }

 int main()
 {
     funcX();
     return 0;
 }
```

若在 main 函数中调用被标记为 deprecated 的函数 funcX，则在 gcc/g++7.3 中编译时会得到如下警告信息：

```cpp
 [root@myaliyun testmybook]# g++ -g -o test_attributes test_attributes.cpp
 test_attributes.cpp: In function ‘int main()’:
 test_attributes.cpp:10:11: warning: ‘void funcX()’ is deprecated: use funcY instead
 [-Wdeprecated-declarations]
 funcX();
 ^
 test_attributes.cpp:3:42: note: declared here
 [[deprecated("use funcY instead")]] void funcX()
```

> Java 开发者对这个标注应该再熟悉不过了。在 Java 中使用@Deprecated 标注可以达到同样的效果，这大概是 C++标准委员“拖欠”广大 C++开发者太久的一个特性吧。

C++ 17 提供了三个实用注解：[[fallthrough]]、 [[nodiscard]] 和 [[maybe_unused]]，这里 逐一介绍它们的用法。

[[fallthrough]] 用于 switch-case 语句中，在某个 case 分支执行完毕后如果没有 break 语句，则编译器可能会给出一条警告。但有时这可能是开发者有意为之的。为了让编译器明确知道开发者的意图，可以在需要某个 case 分支被“贯穿”的地方（上一个 case 没有break 语句）显式设置 [[fallthrough]] 标记。代码示例如下：

```cpp
 switch (type)
 {
 case 1:
     func1();
     //这个位置缺少 break 语句，且没有 fallthrough 标注，
     //可能是一个逻辑错误，在编译时编译器可能会给出警告，以提醒修改

 case 2:
     func2();
     //这里也缺少 break 语句，但是使用了 fallthrough 标注，
     //说明是开发者有意为之的，编译器不会给出任何警告
     [[fallthrough]];

 case 3:
     func3();
 }
```

> 注意：在 gcc/g++中， [[fallthrough]] 后面的分号不是必需的，在 Visual Studio 中必须加上分号，否则无法编译通过。

熟悉 Golang 的读者，可能对 fallthrough 这一语法特性非常熟悉， Golang 中在 switch-case 后加上 fallthrough，是一个常用的告诉编译器意图的语法规则。代码示例如下：

```cpp
 //以下是 Golang 语法
 s := "abcd"
 switch s[3] {
     case 'a':
         fmt.Println("The integer was <= 4")
         fallthrough

     case 'b':
         fmt.Println("The integer was <= 5")
         fallthrough

     case 'c':
         fmt.Println("The integer was <= 6")

     default:
         fmt.Println("default case")
 }
```

[[nodiscard]] 一般用于修饰函数，告诉函数调用者必须关注该函数的返回值（即不能丢弃该函数的返回值）。如果函数调用者未将该函数的返回值赋值给一个变量，则编译器会给出一个警告。例如，假设有一个网络连接函数 connect，我们通过返回值明确说明了连接是否建立成功，则为了防止调用者在使用时直接将该值丢弃，我们可以将该函数使用 [[nodiscard]] 标记：

```cpp
 [[nodiscard]] int connect(const char* address, short port)
 {
     //实现省略
 }

 int main()
 {
     //忽略了connect函数的返回值，编译器会给出一个警告
     connect("127.0.0.1", 8888);
     return 0;
 }
```

在 C++ 20 中，对于诸如 operator new()、 std::allocate()等库函数均使用了 [[nodiscard]] 进行标记，以强调必须使用这些函数的返回值。

再来看另外一个标记。

在通常情况下，编译器会对程序代码中未使用的函数或变量给出警告，另一些编译器干脆不允许通过编译。在 C++ 17 之前，程序员为了消除这些未使用的变量带来的编译警告或者错误，要么修改编译器的警告选项设置，要么定义一个类似于 UNREFERENCED_PARAMETER 的宏来显式调用这些未使用的变量一次，以消除编译警告或错误：

```cpp
#define UNREFERENCED_PARAMETER(x) x

 int APIENTRY wWinMain(HINSTANCE hInstance, HINSTANCE hPrevInstance, LPWSTR lpCmdLine, int nCmdShow)
 {
     //C++17之前为了消除编译器对未使用的变量hPrevInstance、lpCmdLine给出的警告，我们可以这么做
     UNREFERENCED_PARAMETER(hPrevInstance);
     UNREFERENCED_PARAMETER(lpCmdLine);
     //无关代码省略
 }
```

以上代码节选自一个标准 Win32 程序的结构，其中的函数参数 hPrevInstance 和 lpCmdLine 一般不会被用到，编译器会给出警告。为了消除这类警告，这里定义了一个宏 UNREFERENCED_PARAMETER 并进行调用，造成这两个参数被使用的假象。

C++17 有了 [[maybe_unused]] 注解之后，我们就再也不需要这类宏来“欺骗”编译器了。以上代码使用该注解后可以修改如下：

```cpp
 int APIENTRY wWinMain(HINSTANCE hInstance,
                       [[maybe_unused]] HINSTANCE hPrevInstance,
                       [[maybe_unused]] LPWSTR lpCmdLine,
                       int nCmdShow)
 {
     //无关代码省略
 }
```

### **案例 3：final、 override 关键字和 =default、 =delete 语法**

### **3.1 final 关键字**

在 C++11 之前，我们没有特别好的方法阻止一个类被其他类继承，到了 C++11 有了 final 关键字我们就可以做到了。final 关键字修饰一个类，这个类将不允许被继承，这在其他语言（如 Java）中早就实现了。在 C++ 11 中， final 关键字要写在类名的后面，这在其他语言中是写在 class 关键字前面的。示例如下：

```cpp
 class A final
 {
 };

 class B : A
 {
 };
```

由于类 A 被声明成 final， B 继承 A， 所以编译器会报如下错误提示类 A 不能被继承：

```cpp
 error C3246: 'B' : cannot inherit from 'A' as it has been declared as 'final'
```

### **3.2 override 关键字**

C++98/03 语法规定，在父类中加了 virtual 关键字的方法可以被子类重写，子类重写该方法时可以加或不加 virtual 关键字，例如下面这样：

```cpp
 class A
 {
 protected:
     virtual void func(int a, int b)
     {
     }
 };

 class B : A
 {
 protected:
     virtual void func(int a, int b)
     {
     }
 };

 class C : B
 {
 protected:
     void func(int a, int b)
     {
     }
 };
```

这种宽松的规定可能会带来以下两个问题。

- 当我们阅读代码时，无论子类重写的方法是否添加了 virtual 关键字，我们都无法 直观地确定该方法是否是重写的父类方法。
- 如果我们在子类中不小心写错了需要重写的方法的函数签名（可能是参数类型、 个数或返回值类型），这个方法就会变成一个独立的方法，这可能会违背我们重写 父类某个方法的初衷，而编译器在编译时并不会检查到这个错误。

为了解决以上两个问题， C++11 引进了 override 关键字，其实 override 关键字并不是新语法，在 Java 等其他编程语言中早就支持。类方法被 override 关键字修饰，表明该方法重写了父类的同名方法，加了该关键字后，编译器会在编译阶段做相应的检查，如果其父类不存在相同签名格式的类方法，编译器就会给出相应的错误提示。

情形一，父类不存在，子类标记了 override 的方法：

```cpp
 class A
 {
 };

 class B : A
 {
 protected:
     void func(int k, int d) override
     {
     }
 };
```

由于在父类 A 中没有 func 方法，所以编译器会提示错误：

```cpp
 error C3668: 'B::func' : method with override specifier 'override' did not override
 any base class methods
```

情形二，父类存在，子类标记了 override 的方法，但函数签名不一致：

```cpp
 class A
 {
 protected:
     virtual int func(int k, int d)
     {
         return 0;
     }
 };

 class B : A
 {
 protected:
     virtual void func(int k, int d) override
     {
     }
 };
```

上述代码编译器会报同样的错误。正确的代码如下：

```cpp
 class A
 {
 protected:
     virtual void func(int k, int d)
     {
     }
 };

 class B : A
 {
 protected:
     virtual void func(int k, int d) override
     {
     }
 };
```

### **3.3 default 语法**

如果一个 C++类没有显式给出构造函数、析构函数、拷贝构造函数、 operator= 这几类函数的实现，则在需要它们时，编译器会自动生成；或者，在给出这些函数的声明时，如果没有给出其实现，则编译器在链接时会报错。如果使用=default 标记这类函数，则编译器会给出默认的实现。来看一个例子：

```cpp
 class A
 {
 };

 int main()
 {
     A a;
     return 0;
 }
```

这样的代码是可以编译通过的，因为编译器默认生成 A 的一个无参构造函数，假设我们现在向 A 提供一个有参构造函数：

```cpp
 class A
 {
 public:
     A(int i)
     {
     }
 };

 int main()
 {
     A a;
     return 0;
 }
```

这时，编译器就不会自动生成默认的无参构造函数了，这段代码会编译出错，提示 A 没有合适的无参构造函数：

```cpp
 error C2512: 'A' : no appropriate default constructor available
```

我们这时可以手动为 A 加上无参构造函数， 也可以使用=default 语法强行让编译器自己生成：

```cpp
 class A
 {
 public:
     A() = default;
     A(int i)
     {
     }
 };

 int main()
 {
     A a;
     return 0;
 }
```

=default 最大的作用可能是在开发中简化了构造函数中没有实际初始化代码的写法，尤其是声明和实现分别属于.h 和.cpp 文件。例如，对于类 A，其头文件为 a.h，其实现文件为 a.cpp，则正常情况下我们需要在 a.cpp 文件中写其构造函数和析构函数的实现（可能没有实际的构造和析构逻辑）：

```cpp
 //a.h
 class A
 {
 public:
     A();
     ~A();
 };

 //a.cpp
#include "a.h"

 A::A()
 {
 }

 A::~A()
 {
 }

```

可以发现，即使在 A 的构造函数和析构函数中什么逻辑也没有，我们还是不得不在 a.cpp 中写上构造函数和析构函数的实现。有了=default 关键字，我们就可以在 a.h 中直接写成：

```cpp
 //a.h
 class A
 {
 public:
     A() = default;
     ~A() = default;
 };

 //a.cpp
#include "a.h"
 //在 cpp 文件中就不用再写 A 的构造函数和析构函数的实现了

```

### **3.4 =delete 语法**

既然有强制让编译器生成构造函数、析构函数、拷贝构造函数、 operator=的语法，那么也应该有禁止编译器生成这些函数的语法，没错，就是 =delete。在 C++ 98/03 规范中， 如果我们想让一个类不能被拷贝（即不能调用其拷贝构造函数），则可以将其拷贝构造函数和 operator=函数定义成 private 的：

```cpp
 class A
 {
 public:
     A() = default;
     ~A() = default;

 private:
     A(const A& a)
     {
     }

     A& operator =(const A& a)
     {
     }
 };

 int main()
 {
     A a1;
     A a2(a1);
     A a3;
     a3 = a1;
     return 0;
 }

```

通过以上代码利用 a1 构造 a2 时，编译器会提示错误：

```cpp
 error C2248: 'A::A' : cannot access private member declared in class 'A'
 error C2248: 'A::operator =' : cannot access private member declared in class 'A'

```

我们利用这种方式间接实现了一个类不能被拷贝的功能，这也是继承自 boost::noncopyable 的类不能被拷贝的实现原理。现在有了=delete语法，我们直接使用该语法禁止编译器生成这两个函数即可：

```cpp
 class A
 {
 public:
     A() = default;
     ~A() = default;
 public:
     A(const A& a) = delete;
     A& operator =(const A& a) = delete;
 };

 int main()
 {
     A a1;
     //A a2(a1);
     A a3;
     //a3 = a1;
     return 0;
 }
```

一般在一些工具类中， 我们不需要用到构造函数、 析构函数、 拷贝构造函数、 operator= 这 4 个函数，为了防止编译器自己生成，同时为了减小生成的可执行文件的体积，建议使用=delete 语法禁止编译器为这 4 个函数生成默认的实现代码，例如：

```cpp
 //这是一个字符转码工具类
 class EncodeUtil
 {
 public:
     static std::wstring AnsiiToUnicode(const std::string& strAnsii);
     static std::string UnicodeToAnsii(const std::wstring& strUnicode);
     static std::string AnsiiToUtf8(const std::string& strAnsii);
     static std::string Utf8ToAnsii(const std::string& strUtf8);
     static std::string UnicodeToUtf8(const std::wstring& strUnicode);
     static std::wstring Utf8ToUnicode(const std::string& strUtf8);

 private:
     EncodeUtil() = delete;
     ~EncodeUtil() = delete;
     EncodeUtil(const EncodeUtil& rhs) = delete;
     EncodeUtil& operator=(const EncodeUtil& rhs) = delete;
 };

```

### **案例 4：对多线程的支持**

我们来看一个稍微复杂一点的例子。

在 C++11 之前，由于 C++98/03 本身缺乏对线程和线程同步原语的支持，我们要写一个生产者消费者逻辑要这么写。

在 Windows 上：

```cpp
 /**
  * RecvMsgTask.h
  */
 class CRecvMsgTask : public CThreadPoolTask
 {
 public:
     CRecvMsgTask(void);
     ~CRecvMsgTask(void);

 public:
     virtual int Run();
     virtual int Stop();
     virtual void TaskFinish();

     BOOL AddMsgData(CBuffer* lpMsgData);

 private:
     BOOL HandleMsg(CBuffer* lpMsg);

 private:
     HANDLE                m_hEvent;
     CRITICAL_SECTION      m_csItem;
     HANDLE                m_hSemaphore;
     std::vector<CBuffer*> m_arrItem;
 };

 /**
  * RecvMsgTask.cpp
  */
 CRecvMsgTask::CRecvMsgTask(void)
 {
     ::InitializeCriticalSection(&m_csItem);
     m_hSemaphore = ::CreateSemaphore(NULL, 0, 0x7FFFFFFF, NULL);
     m_hEvent = ::CreateEvent(NULL, TRUE, FALSE, NULL);
 }

 CRecvMsgTask::~CRecvMsgTask(void)
 {
     ::DeleteCriticalSection(&m_csItem);

     if (m_hSemaphore != NULL)
     {
         ::CloseHandle(m_hSemaphore);
         m_hSemaphore = NULL;
     }

     if (m_hEvent != NULL)
     {
         ::CloseHandle(m_hEvent);
         m_hEvent = NULL;
     }
 }

 int CRecvMsgTask::Run()
 {
     HANDLE hWaitEvent[2];
     DWORD dwIndex;
     CBuffer * lpMsg;

     hWaitEvent[0] = m_hEvent;
     hWaitEvent[1] = m_hSemaphore;

     while (1)
     {
         dwIndex = ::WaitForMultipleObjects(2, hWaitEvent, FALSE, INFINITE);

         if (dwIndex == WAIT_OBJECT_0)
             break;

         lpMsg = NULL;

         ::EnterCriticalSection(&m_csItem);
         if (m_arrItem.size() > 0)
         {
             //消费者从队列m_arrItem中取出任务执行
             lpMsg = m_arrItem[0];
             m_arrItem.erase(m_arrItem.begin() + 0);
         }
         ::LeaveCriticalSection(&m_csItem);

         if (NULL == lpMsg)
             continue;

         //处理任务
         HandleMsg(lpMsg);

         delete lpMsg;
     }

     return 0;
 }

 int CRecvMsgTask::Stop()
 {
     m_HttpClient.SetCancalEvent();
     ::SetEvent(m_hEvent);
     return 0;
 }

 void CRecvMsgTask::TaskFinish()
 {
 }

 //生产者调用这个方法将Task放入队列m_arrItem中
 BOOL CRecvMsgTask::AddMsgData(CBuffer * lpMsgData)
 {
     if (NULL == lpMsgData)
         return FALSE;

     ::EnterCriticalSection(&m_csItem);
     m_arrItem.push_back(lpMsgData);
     ::LeaveCriticalSection(&m_csItem);

     ::ReleaseSemaphore(m_hSemaphore, 1, NULL);

     return TRUE;
 }

```

在 Linux 下：

```cpp
#include <pthread.h>
#include <errno.h>
#include <unistd.h>
#include <list>
#include <semaphore.h>
#include <iostream>

 class Task
 {
 public:
     Task(int taskID)
     {
         this->taskID = taskID;
     }

     void doTask()
     {
         std::cout << "handle a task, taskID: " << taskID << ", threadID: " << pthread_self() << std::endl;
     }

 private:
     int taskID;
 };

 pthread_mutex_t  mymutex;
 std::list<Task*> tasks;
 pthread_cond_t   mycv;

 void* consumer_thread(void* param)
 {
     Task* pTask = NULL;
     while (true)
     {
         pthread_mutex_lock(&mymutex);
         while (tasks.empty())
         {
             //如果获得了互斥锁，但是条件不合适的话，pthread_cond_wait会释放锁，不往下执行。
             //当发生变化后，条件合适，pthread_cond_wait将直接获得锁。
             pthread_cond_wait(&mycv, &mymutex);
         }

         pTask = tasks.front();
         tasks.pop_front();

         pthread_mutex_unlock(&mymutex);

         if (pTask == NULL)
             continue;

         pTask->doTask();
         delete pTask;
         pTask = NULL;
     }

     return NULL;
 }

 void* producer_thread(void* param)
 {
     int taskID = 0;
     Task* pTask = NULL;

     while (true)
     {
         pTask = new Task(taskID);

         pthread_mutex_lock(&mymutex);
         tasks.push_back(pTask);
         std::cout << "produce a task, taskID: " << taskID << ", threadID: " << pthread_self() << std::endl;

         pthread_mutex_unlock(&mymutex);

         //释放信号量，通知消费者线程
         pthread_cond_signal(&mycv);

         taskID ++;

         //休眠1秒
         sleep(1);
     }

     return NULL;
 }

 int main()
 {
     pthread_mutex_init(&mymutex, NULL);
     pthread_cond_init(&mycv, NULL);

     //创建5个消费者线程
     pthread_t consumerThreadID[5];
     for (int i = 0; i < 5; ++i)
         pthread_create(&consumerThreadID[i], NULL, consumer_thread, NULL);

     //创建一个生产者线程
     pthread_t producerThreadID;
     pthread_create(&producerThreadID, NULL, producer_thread, NULL);

     pthread_join(producerThreadID, NULL);

     for (int i = 0; i < 5; ++i)
         pthread_join(consumerThreadID[i], NULL);

     pthread_cond_destroy(&mycv);
     pthread_mutex_destroy(&mymutex);

     return 0;
 }

```

怎么样？上述代码如果对于新手来说，望而却步。

为了实现这样的功能在 Windows 上你需要掌握线程如何创建、线程同步对象 CriticalSection、Event、Semaphore、WaitForSingleObject/WaitForMultipleObjects 等操作系统对象和 API。

在 Linux 上需要掌握线程创建，你需要了解线程创建、互斥体、条件变量。

对于需要支持多个平台的开发，需要开发者同时熟悉上述原理并编写多套适用不同平台的代码。

C++11 的线程库改变了这个现状，现在你只需要掌握 std::thread、std::mutex、std::condition_variable 少数几个线程同步对象即可，同时使用这些对象编写出来的代码也可以跨平台。示例如下：

```cpp
#include <thread>
#include <mutex>
#include <condition_variable>
#include <list>
#include <iostream>

 class Task
 {
 public:
     Task(int taskID)
     {
         this->taskID = taskID;
     }

     void doTask()
     {
         std::cout << "handle a task, taskID: " << taskID << ", threadID: " << std::this_thread::get_id() << std::endl;
     }

 private:
     int taskID;
 };

 std::mutex                mymutex;
 std::list<Task*>          tasks;
 std::condition_variable   mycv;

 void* consumer_thread()
 {
     Task* pTask = NULL;
     while (true)
     {
         std::unique_lock<std::mutex> guard(mymutex);
         while (tasks.empty())
         {
             //如果获得了互斥锁，但是条件不合适的话，pthread_cond_wait会释放锁，不往下执行。
             //当发生变化后，条件合适，pthread_cond_wait将直接获得锁。
             mycv.wait(guard);
         }

         pTask = tasks.front();
         tasks.pop_front();

         if (pTask == NULL)
             continue;

         pTask->doTask();
         delete pTask;
         pTask = NULL;
     }

     return NULL;
 }

 void* producer_thread()
 {
     int taskID = 0;
     Task* pTask = NULL;

     while (true)
     {
         pTask = new Task(taskID);

         //使用括号减小guard锁的作用范围
         {
             std::lock_guard<std::mutex> guard(mymutex);
             tasks.push_back(pTask);
             std::cout << "produce a task, taskID: " << taskID << ", threadID: " << std::this_thread::get_id() << std::endl;
         }

         //释放信号量，通知消费者线程
         mycv.notify_one();

         taskID ++;

         //休眠1秒
         std::this_thread::sleep_for(std::chrono::seconds(1));
     }

     return NULL;
 }

 int main()
 {
     //创建5个消费者线程
     std::thread consumer1(consumer_thread);
     std::thread consumer2(consumer_thread);
     std::thread consumer3(consumer_thread);
     std::thread consumer4(consumer_thread);
     std::thread consumer5(consumer_thread);

     //创建一个生产者线程
     std::thread producer(producer_thread);

     producer.join();
     consumer1.join();
     consumer2.join();
     consumer3.join();
     consumer4.join();
     consumer5.join();

     return 0;
 }

```

感觉如何？代码既简洁又统一。

这就是 C++11 之后使用 Modern C++ 开发的效率！

C++11 之后的 C++ 更像一门新的语言。

当 C++11 的编译器发布之后（Visual Studio 2013、g++4.8），我第一时间更新了我的编译器，同时把我们的项目使用了 C++11 特性进行了改造。

当然，例子还有很多，限于文章篇幅，这里就列举 4 个案例。