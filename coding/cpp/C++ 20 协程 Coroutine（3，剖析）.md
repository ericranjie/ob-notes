
我不想种地 码砖杂役

 _2022年09月11日 09:09_ _内蒙古_

# C++ 20 协程 Coroutine（3，剖析）

最后，我们来剖析一下协程的过程。通过这个剖析，希望达到梳理协程几个重要概念的关系，把这些点串起来。所以在概念参考我们列出了相应的概念文字。

## 协程的创建

C++20协程在启动前，开始会new 一个协程状态（`coroutine state`）。然后构造协程的承诺对象（`promise`)。承诺对象（`promise`)通过`get_return_object()`构造协程的返回值`result`。这个返回值在协程第一次挂起时，赋值给调用者。然后通过`co_await promise.initial_suspend()`，决定协程初试完成后的行为。如果返回`std::suspend_always`，初始化就挂起，如果返回`std::suspend_never` ，初始化后就继续运行。(注意initial_suspend也可以返回其他协程体)

![Image](https://mmbiz.qpic.cn/mmbiz_png/pg57MH7pFC7icUJtPSiag3BKssa6tibKXZDhv3outPicFyvpibiaCjSia3tsvBzBYtKHNicgiaicibTBmuCiaWABJAEzFdUXyQ/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

## 协程的co_await

`cw_ret = co_await awaiter` 或者`cw_ret = co_await fun()`，先计算表达式fun，fun返回结果，就是一个等待体`awaiter`。系统先调用`awaiter.await_ready()`接口，看等待体是否准备好了，没准备好（`return false`）就调用`awaiter.await_suspend()`。`await_suspend`根据参数可以记录调用其的协程的的句柄。`await_suspend`的返回值为`return true` ，或者 `return void` 就会挂起协程。

后面在外部如果恢复了协程的运行，`awaiter.await_resume()`接口被调用。其返回结果，作为`co_await`的返回值。

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

## 协程的co_yield

`co_yield cy_ret;`，相当于调用`co_wait promise.yield_value(cy_ret)`，你可以在`yield_value`中记录参数`cy_ret`后面使用，`yield_value`的返回值如果是`std::suspend_always`，协程挂起，如果返回`std::suspend_never` ，协程就继续运行。

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

## 协程的co_return

`co_yield cr_ret;`，调用`promise.retun_value(cr_ret)`，如果没有返回值相当于`promise.retun_viod()`，你可以在`retun_value`中记录参数`cr_ret`后面使用。然后调用`co_await promise.final_suspend(void)`，如果返回值是`std::suspend_always`，你需要自己手动青清理`coroutine handle`，调用`handle.destroy()`。

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

> 这儿存在一个疑问，`final_suspend`，并没有真正挂起协程。看C++ 参考,里面说的也是`calls promise.final_suspend() and co_awaits the result.`。按说如果返回应该要挂起。但用VS 2022测试是不会挂起的，再探 C++20 协程文章中说的是如果返回`std::suspend_always`，需要你自己清理`coroutine handle`。存疑吧。

## 概念参考附录：

这些概念在原文第一章都有，附录在此仅供您方便参考。

### 协程状态（coroutine state）

协程状态（`coroutine state`）是协程启动开始时，new空间存放协程状态，协程状态记录协程函数的参数，协程的运行状态，变量。挂起时的断点。

注意，协程状态 (`coroutine state`)并不是就是协程函数的返回值RET。虽然我们设计的RET一般里面也有`promise`和`coroutine handle`，大家一般也是通过RET去操作协程的恢复，获取返回值。但`coroutine state`理论上还应该包含协程运行参数，断点等信息。而**协程状态** (`coroutine state`)应该是协程句柄（`coroutine handle`）对应的一个数据，而由系统管理的。

  

### 承诺对象（promise）

承诺对象的表现形式必须是`result::promise_type`，`result`为协程函数的返回值。

承诺对象是一个实现若干接口，用于辅助协程，构造协程函数返回值；提交传递`co_yield`，`co_return`的返回值。明确协程启动阶段是否立即挂起；以及协程内部发生异常时的处理方式。其接口包括：

- `auto get_return_object()` ：用于生成协程函数的返回对象。
    
- `auto initial_suspend()`：用于明确初始化后，协程函数的执行行为，返回值为等待体（`awaiter`），用`co_wait`调用其返回值。返回值为`std::suspend_always` 表示协程启动后立即挂起（不执行第一行协程函数的代码），返回`std::suspend_never` 表示协程启动后不立即挂起。（当然既然是返回等待体，你可以自己在这儿选择进行什么等待操作）
    
- `void return_value(T v)`：调用`co_return v`后会调用这个函数，可以保存`co_return`的结果
    
- `auto yield_value(T v)`：调用`co_yield`后会调用这个函数，可以保存`co_yield`的结果，其返回其返回值为`std::suspend_always`表示协程会挂起，如果返回`std::suspend_never`表示不挂起。
    
- `auto final_suspend() noexcept`：在协程退出是调用的接口，返回`std::suspend_never` ，自动销毁 coroutine state 对象。若 `final_suspend` 返回 `std::suspend_always` 则需要用户自行调用 `handle.destroy()` 进行销毁。但值得注意的是返回std::suspend_always并不会挂起协程。
    

前面我们提到在协程创建的时候，会new协程状态（`coroutine state`）。你可以通过可以在 `promise_type` 中重载 `operator new` 和 `operator delete`，使用自己的内存分配接口。（请参考再探 C++20 协程）

  

### 协程句柄（coroutine handle）

协程句柄（`coroutine handle`）是一个协程的标示，用于操作协程恢复，销毁的句柄。

协程句柄的表现形式是`std::coroutine_handle<promise_type>`，其模板参数为承诺对象（`promise`）类型。句柄有几个重要函数：

- `resume()`函数可以恢复协程。
    
- `done()`函数可以判断协程是否已经完成。返回false标示协程还没有完成，还在挂起。
    

协程句柄和承诺对象之间是可以相互转化的。

- `std::coroutine_handle<promise_type>::from_promise` ：这是一个静态函数，可以从承诺对象（promise）得到相应句柄。
    
- `std::coroutine_handle<promise_type>::promise()` 函数可以从协程句柄`coroutine handle`得到对应的承诺对象（`promise`）
    

  

### 等待体（awaiter）

co_wait 关键字会调用一个等待体对象(`awaiter`)。这个对象内部也有3个接口。根据接口co_wait  决定进行什么操作。

- `bool await_ready()`：等待体是否准备好了，返回 false ，表示协程没有准备好，立即调用await_suspend。返回true，表示已经准备好了。
    
- `auto await_suspend(std::coroutine_handle<> handle)`如果要挂起，调用的接口。其中handle参数就是调用等待体的协程，其返回值有3种可能
    

- void 同返回true
    
- bool 返回true 立即挂起，返回false 不挂起。
    
- 返回某个协程句柄（`coroutine handle`），立即恢复对应句柄的运行。
    

- `auto await_resume()` ：协程挂起后恢复时，调用的接口。返回值作为co_wait 操作的返回值。
    

**等待体**（awaiter）值得用更加详细的笔墨书写一章，我们就放一下，先了解其有2个特化类型。

- `std::suspend_never`类，不挂起的的特化等待体类型。
    
- `std::suspend_always`类，挂起的特化等待体类型。
    

前面不少接口已经用了这2个特化的类，同时也可以明白其实协程内部不少地方其实也在使用`co_wait` 关键字。

  

## 本章总结

此章讲解了协程的启动，3个关键字的细节。您可以通过这些关键概念，融合协程状态（`coroutine state`），承诺对象（`promise`），协程句柄（`coroutine handle`），**等待体**（`awaiter`）。

## 参考文档

初探 C++20 协程

再探 C++20 协程

Coroutines (C++20)

协程（coroutine）简介

The Coroutine in C++ 20 协程之诺

C++ Coroutines: Understanding operator co_await

  

  

  

Reads 148

​

Send Message

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/pg57MH7pFC5SmXunUYSRovoeKZOqGl5TlL0pcGLNCtf9dxQX6SLaUIH2aJj76yEf9IhQAzDe0jCncyyaicT94bg/300?wx_fmt=png&wxfrom=18)

码砖杂役

41Wow

Send Message