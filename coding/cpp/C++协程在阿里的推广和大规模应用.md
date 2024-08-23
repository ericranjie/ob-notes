

原创 马骏 CPP与系统软件联盟

 _2022年06月06日 18:58_ _上海_

  

  

  

**点击蓝字**

**关注我们**

  

  

**特|邀|作|者**

**马骏**

**阿里云智能 C/C++编译器技术主管**

拥有多年编译器开发和性能优化经验，熟悉GCC LLVM的Middle&Back End. 先后在阿里巴巴编译器团队 负责集团C/C++ workload的性能优化，推动AutoFDO优化的上线。参与GCC/LLVM社区C++20标准中协程部分的开发，并在阿里推广和大规模落地。

  

**01**

**协程介绍**

  

  

  

![图片](https://mmbiz.qpic.cn/mmbiz_png/YQ55UxynrohYUDlfkibLUlgb6rQpZa4VppV8BPDWYh7l4XHk8KsSgco44WDW5VUDSkY1sqibHKVHFbFkicc7qx6pQ/640?wx_fmt=png&wxfrom=13&tp=wxpic)

  

协程并不是一个新的概念，协程是一种程序组件，是由子例程的概念泛化而来的。像 C++ 理解的 lambda、函数。正常的C++方法只有一个入口点，正常只返回一次。协程的话规定可以有多个入口点，并且可以在指定的位置挂起和恢复执行。需要实现上述的概念需要以下两点：一是我们在控制权离开的时候需要暂停协程执行，再次回来的时候需要从我们离开的位置继续执行。第二点就是协程内部的本地数据需要做持久化。无论是我们理解的现有的协程库还是C++20的协程都是围绕这两点方面来实现的。

  

![图片](https://mmbiz.qpic.cn/mmbiz_png/YQ55UxynrohYUDlfkibLUlgb6rQpZa4VpdlyqNc7pPjX6DrZq0RwYxqPkrb8iaAlP8HicWO08mHLnAAO4Jpp8pfBA/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

  

接下来，我们来看为什么需要协程？初听上去好像除了多次执行、多次恢复好像也没有什么其他的地方。简单来说就是一个高吞吐尤其是在当今时代的一个CPU，我们CPU的执行速度相对于我们的缓存和IO是非常快的。这种情况下面如何来提升我们系统的吞吐率，在这种场景下，我们的协程就派上用场了。传统的提高高吞吐的方法为多线程，相对于多线程来说的话，协程是一个用户态的thread，它没有多线程的锁，也没有嵌入到内核态的切换，所以它的执行效率理论上是高于多线程的。相对于异步回调，我们的协程对于开发者的思路是同步的一种编程模式来写的，流程上面能更好的理解代码。协程的编程模式是让我们避免了callback的模式。

  

![图片](https://mmbiz.qpic.cn/mmbiz_png/YQ55UxynrohYUDlfkibLUlgb6rQpZa4VpVmY3rQqyLticXLLHaIC9dMKpsaFZkrYlUY3KJ0zt7w9zAXLz7PP8QwQ/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

  

协程主要有：阿里内部用C++开发的libeasy库，腾讯开发的libco，boost的coroutine，go开发的 go coroutine 以及 C#开发的 await 等等。

  

这么多的协程库有它的什么区别和相似之处，主要分为以下三点：

  

**1、控制传递（Contorl-transfer）：对称/非对称 (return-to-caller)**

协程挂起的时候是否一定要返回到调用者那里，如果是一定要返回到调用者那里，则这种协程是非对称的，一般来说这种非对称协程需要跟调度器进行绑定，由调度器进行接管。对称的则是当我挂起恢复时，可以直接调用其他的协程，这种对于我们编程来说是方便了很多。

  

**2、是否作为语言的第一类（First-class）对象提供：语言特性&编译器支持**

能否充分的利用语言特性和编译器支持，来开发出协程的编程模式和性能。

  

**3、栈式（Stackful）/无栈（Stackless）构造：是否在内部的嵌套调用中挂起（性能&挂起位置）**

有栈的话需要有一个协程栈作为我们在暂停协程时来拷贝栈信息，对本地数据进行持久化。有栈和无栈的另外一个区别：能否只能在我的最顶层进行挂起和恢复。有栈可以在任意一层进行挂起和恢复。无栈则只能在顶层做挂起和恢复。有栈和无栈决定了我们编程方式的灵活性以及最终的性能。有栈的切换是需要代价的。

  

**02**

**C++20协程**

  

  

  

![图片](https://mmbiz.qpic.cn/mmbiz_png/YQ55UxynrohYUDlfkibLUlgb6rQpZa4Vpew3rdEsnXiacnianFJwvuFEQ1dznmlEc9jhoSZW569Gr6bWt9ic3gm2Jw/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

  

协程是 C++ 20 语言中 Big four 之一。C++20的协程是一个对称，语言第一类支持，并且是无栈特性的一类协程。相对于libeasy是一个非对称的，语言不支持的且有栈的一种协程机制，C++ 20 协程是跟它完全相反的。如果函数中使用了 C++ 20 中的三个方法co_wait、co_yield、co_return，则可以称这个函数是一个协程函数，但是背后是一个非常复杂和灵活的事情。

  

![图片](https://mmbiz.qpic.cn/mmbiz_png/YQ55UxynrohYUDlfkibLUlgb6rQpZa4VpQPXJ06aVQcHpW6ZEaUPV5HgbxWlepM1QibjYNBqfMq7IVwP5gibvicbrw/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

  

我们用一个例子来看最终编译会如何处理和生成出怎么样的代码。大概的意思是我们的代码会执行到func函数，同时执行到 co_wait 的行为这停止，它会移交控制权，当我再次恢复的时候会继续执行这个‘Coroutine’这个输出。Resume的返回值会存放一个coro_handle，这个handle会通过 resume destroy 来恢复和析构我们的协程。

  

这个方法中没有返回值和返回的语句，但是我们写了一个Resumable的返回对象，看上去很诡异，我们先build一下，不出意外会报错。报错告诉我们：我们的Resumable中缺少了一个promise_type这个member。

  

我们大概想下，我们如何完善Resumable来让我们的程序跑起来，coroutine handle需要放在我们的协程对象里面，用来控制我们的协程恢复和销毁，以及一些状态的判断。另外一个需要我们编译器报错时提供给我们的 promise_type 结构体来做一些额外的协程函数中的控制。

  

![图片](https://mmbiz.qpic.cn/mmbiz_png/YQ55UxynrohYUDlfkibLUlgb6rQpZa4Vp2zqayF7LFwaVsKic3xEAEScS0bbvDnkIy4svoHr2ACckvM7BU6dd99A/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

  

这就涉及到我们编译器对协程的第一次改造。这里面其实可以理解为是源到源的转换（Source to Source）之前的func函数，编译器处理的时候会给它插入一些操作。首先我们看到原有的func body会被放在一个 try catch 中。程序开始时，用operator new创建一个堆，这个堆就满足我们第二个能力，做数据的持久化。C++ 20协程中是用堆来做数据的持久化的。接下来看到需要构建promise_type结构体，我们的协程对象都是从promise_type来获得的，最终通过promise返回一个Resumable。

  

![图片](https://mmbiz.qpic.cn/mmbiz_png/YQ55UxynrohYUDlfkibLUlgb6rQpZa4VpI1ib8ricNiaa1IiayxkmjRZlIoF3EYAqhNZCZbRVOeyMqI9lz6QplTeoJA/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

  

接下来我们来看func如何来构造多入口的这种能力。基于上面的实现，我们来实现Resumable，包含一个协程句柄coro_handle和我们的Resumable协程对象是一一关联的。

  

其中 resume 函数是用来判断这个handle是否被执行完成，如果没有则会一直进行恢复，直到它析构或者被执行完成。promise_type大概需要五个函数，可以简单return ture或false。

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E "幻灯片11.png")

  

我们除了生成刚才说的初始化的代码和返回的代码，还会额外多生成两个co_wait promise.initial_suspend() 和 co_wait promise.final_suspend()，以及我们自己写的co_wait std::experimental::suspend_aways()。co_wait关键字在C++标准里面是用来规定我们这个函数恢复和暂停的逻辑和行为。co_wait表示我这个点有可能会暂停和恢复。暂停点怎么执行是需要一系列的条件判断的，这些会比有栈协程好很多。在 C++ 20 协程中，我们可以通过不同的条件作判断，判断程序的状态来实现同步或异步状态的混合。

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

我们最后转换出来的case就变成右边的4个部分，我们做了一系列的初始化，有一些暂停点和恢复点，最后返回和销毁。

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E "幻灯片16.png")

  

第一次改造，我们的执行流程就如上图右边的形式。好像也不符合多入口多出口的形式，而且也不知道如何恢复和暂停。

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

接下来，我们就讲如何进行恢复和暂停。co_wait std::experimental 表达式规则，在C++标准里面需要通过一系列的规则来转成avaitable的对象。avaitable对象其实是一个Awaiter的struct。Awaiter需要有三个方法：await_ready()、await_suspend、await_resume。

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

这个就是我们的第二次改造，其实是对co_wait std::experimental 进行扩展。await_ready 来判断我们当前协程是否已经ready，如果ready则不执行暂停操作，继续往下执行。如果没有ready则会调用 await_suspend 这个函数。这里扩展出来三个函数，它可以返回void，void表示一定要暂停，一定要挂起。也可以返回布尔值，布尔值也是和 await_ready() 一样根据返回的状态来判断是否做挂起。

  

最后还有比较特殊的一点是C++ 协程有一个对称能力，await_suspend() 可以返回另外一个coroutine_handle，并立马去恢复这样一个 coroutine，那C++ 协程的对称能力就是在这个暂停点可以不去挂起，而是继续执行另外一个协程的恢复。

  

假设我们现在这儿写了一个void, 那await_suspend() 执行完了以后就 return_to_the_caller()。Return to the caller 回去以后，handler可能再通过 Resume 再回来，回到resume_point 继续执行。首先进来其实是要执行await_resume，这也是await里面的一个状态，一般是用来跟外界去做一些数据交互的。因为这里写的 co_await 的expression没有返回值，实际上它是可以带返回值的，通过 await_resume把最终的一个值返回来。这就是我们第二次源到源的转换。

 ![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

这张图里的实黑线表示的是不会脱离这样一个协程函数，会一直执行到 initial_suspend 这个 point; 然后再考虑有可能根据 co_await 扩展出来的那三个method去判断是否有可能会跳出，有可能会在这挂起，还是有可能会继续往下执行。虚线表示从 suspend point 到下一个 resume point恢复点，可能会挂起，也可能继续在里面执行。其中 suspend_always 是 C++ 20 STL提供了它的实现。其实只要把 await_ready() 设成一个false, 把 await_suspend() 返回值设成一个void, 把await_resume()也设成一个void, 这就是suspend_always在标准库里面的实现。final_suspend其实是和 initial_suspend一样的状态。

  

这里可以看到有一个初始化的代码，我们创建了堆，也能够有数据持久化能力，也有三个恢复点和暂停点，也有最终的析构函数。看上去好像条件都满足了，但其实还是一个普通 C++ 函数，只有一个入口和一个出口，并没有实现多入口、多出口的能力，那么编译器还要做最后一次改造。

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

最后一次改造也是最重要的一次改造。我们把上面的一个函数拆成三个函数：我们创建的 create 这部分代码单独做一个函数；这部分中间包含了所有的暂停点和恢复点的作为单独一部分；destroy 最终的资源回收以及一些对象析构额外作为一个函数。

  

Ramp函数，所谓 func 起始函数，当真正生成完了以后去调用 fun 函数，其实真正调用的是 fun.remp() 这个函数。之前说创建了一个堆，然后把所有本地的变量、elements 都copy到这样的堆上去，去给 coroutine_handler 赋值。这里早早就把coroutine 对象 resume返回回去了，但在返回之前需要 resume一下 resume()函数，表示一定要执行到第一个暂停点为止才能挂起。resume函数把之前那几个暂停点和恢复点做了简单的排序，用一个大的 switch-case 写了一个简单的状态机，每次根据当前恢复点的 index 值执行对应的代码。可以看到case 0其实表示恢复点从一开始一直执行到 initial suspend point 就退出了。当外界通过 coroutine handle去resume, 第二次进来时 index 就是1了，就开始执行case 1部分，这样来实现真正的func.resume多入口、多出口的功能，并且没有依赖任何一个 Runtime 机制，完全是通过 C++代码的转换来实现这样的一个功能。

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

可以看到这样一个简单的函数经过三次转换，第一次把协程函数做一次扩展，第二次把 co_await expression再做一次扩展；第三次把扩展后的一个程序转成这样的一个格式，来做这样的一个执行，首先正常调用的是 func.ramp() 函数，后续 resume 都是通过 func.resume(）去做恢复的。根据恢复点的不同去执行不同的代码，最终执行完成以后或者说在任何一个暂停点里想放弃这个协程的话，可以调destroy函数去销毁这个协程。

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E "幻灯片19.png")

接下来再补齐这个代码的一个测试用例。测试用例其实很简单，就是调用一个 func 函数，返回 Resumable。Resumable 调它的一个 resume函数，然后不断地去执行，直到结束为止。那协程代码里面有一个隐含的，当我的状态是所有 suspend 的点都已经执行完了以后，自动会去执行到析构代码里去，也就实现了一个RAII 的状态。你可以看到最终输出的是一个 Hello Coroutine。

  

**03**

**编译器相关工作**

  

  

  

上面介绍的基本都是编译器正常的流程，它是一步步转换的。这里面我其实没有介绍一些优化方面的事情。其实 C++ 协程优化方面的事情也很值得讲，而且也是很重要的一个点，简单讲一下。

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

我们的 C++ 协程最终转换出来的分两部分，协程函数的展开和 co_await expression的展开，这些其实都是在我们的抽象语法树里去做更多的扩展，类似于source-to-source transformation。我相信大家可能有些人会知道。另外一部分是多入口的一个做法，也就是把协程函数去做拆分，让状态机的一个机制去满足多入口、多出口。

  

这个点的选择很微妙，就是GCC和LLVM这个点其实标准里面没有规定，编译器只要根据标准的要求自己去做实现。GGC和LLVM在这里的实现差别是很大的。GCC 在AST里生成 IR 的时候就已经把这个协程函数拆开，有一点不好的是在拆成 IR 的时候其实没有做任何的优化。它的策略是把函数已经拆开了以后内联并加上一些特定的协程相关的分析来完成这样的一个优化。这样就把拆分的策略转化成一个内联以及内联以后优化的策略。

  

而LLVM则是另外一种思想，LLVM的协程是基于 IR 去做拆分的，所以在编译器 pipeline 相当靠后的阶段。它的一个思想是如果这个协程函数有一些冗余的计算，我希望走一遍 pipeline。这个编译器已经帮我做了充分的优化，在这个基础上我再做 Split，然后再对拆分完以后的三个函数分别去做优化。这两个策略其实导致了我们后续去选择支持 GCC 还是 LLVM 也产生了很大的影响。

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

介绍一下我们的工作，我们是C/C++的一个编译器团队。我们从去年的十月份开始就开始支持 C++ 20 协程相关的工作。一开始其实是在GCC里面，我们和社区合作主要是对 C++ 协程做支持，没有任何优化方面的事，只是把所有协程的代码case跑通，最终其实是在GCC-10上面去做release。这主要集中在前端，就像前面说的，GCC选择其实是在 AST 转 IR 的阶段就已经做了Split，所以我们当时的工作主要集中在 C/C++ Frontend。后续的优化虽然很早拆出来了，避免了一些其他问题，但导致有些优化很难直接去利用 inline 或现有的 pipeline 做一些事，还是需要加很多针对 coroutine 相关特性的分析。比如加一些额外的 IPA 分析或者 IPO 优化，让协程拆分完以后能获得比较好的性能。

  

吴老师也说过 operator new其实是一个比较大的隐患，C++ 委员会因为这个协程的这个标准就争辩过。因为 C++ 协程并不是所见即所得的，后面隐藏了好多好多东西，最重要的一个点就是operator new, 这个东西对于性能的影响是相当大的。GCC现在由于没有加任何的优化，而且优化加起来可能会相当麻烦，导致现在 GCC 的 C++协程的性能其实不太好，跑跑简单的测试用例可以 ，但跑真正的 workload 可能影响会比较大。所以我们后面就转到了LLVM，当时是 LLVM 10版本，其实已经有比较完善的支撑，性能看上去也还不错。所以我们的工作主要是去完善优化和改善 C++ 协程的能力，让它能够在我们真正做 workload 时具有高吞吐、低延迟的这样一个高性能。

  

我们内部支持Clang-8，但是会做一系列自己的backport，以及一些额外的还没有upstream的一些功能。Clang 11 对协程的特性支持应该会比较稳定，而且性能也相当的不错。因为我们大部分的patch都已经upstream了。我们主要做的工作是针对 operator new申请出来的frame去做一些优化，能删则删，能把它合并就把它合并。LLVM里面有一个CoroElide的优化 , 我们是利用这个协程 coroutine handler逃逸的分析，结合这个CoroElide去做Elision。如果真正消不掉，还可以去优化协程帧的size。也就是说，我们希望和正常栈复用的情况一样，用stack parallelism的算法去优化协程帧，并且现在协程调试可能会有一定的难度，我们在一点一点地改进协程调试的信息，以及 multi_thread 涉及一些 coroutine await suspend 以后会出现额外的 use-after-free 情况，以及对称调用里会涉及一些尾调用，这些种种其实都是对于协程的改善和优化。

  

**04**

**future_lite协程库**

  

  

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

讲完编译器实践方面，我们在这个基础上其实还做了一个future_lite 协程库，原因也很清楚了，因为现在 C++ 20 的标准里面其实没有 coroutine library，C++ 23 可能会支持，但仅仅是可能，因为目前争议比较大。第三方库我们也有 cppcoro 库和 folly 库。我们自己实现这样一个 future_lite库其实是借鉴了Facebook 的 folly，但加上了很多我们自己的特性。

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E) 

我们主要的组件，Lazy<T> 借鉴的是惰性执行的这样一个思想，也就是说一个 Lazy 的协程是等待执行准备好才做后续的操作，所以这个 Lazy 既是 coroutine handler，也是 awaiter，它完全可以嵌套起来去做调用，后续我有两个例子会讲一下。future/promise 实现的功能和 STL feature 和 promise 完全相同的功能 , 但是是协程化的一个组件，也具有我们自己调度器的接口，并且具有其他一些功能，比如说像批量执行、generator以及一些 Mutex 的异步的一些配套实现。

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

上图右边是我们自己写的 generator，每次会 co_yield一个值。co_yield 和 co_await 是同一回事，只是做了简单的封装，能够调额外的 API。左边展示我们用得比较多的一个例子，一个普通函数 func 会调用一个 task2，并且等待 task2 完成。task2是通过我们指定的调度器或执行器去执行，task2 里面又调了一个 getvalue()，getvalue()也是一个协程函数，它返回的也是 Lazy<T>，并且等待 gettvalue() 完成。getvalue() 里面写了co_return co_await ValueAmaiter()，它去等待 ValueAmaiter执行。ValueAmaiter 起了一个thread, 去恢复 getvalue() 的执行。所以你可以看到这里面对称的构造是当 getvalue() 挂起的时候又通过另外一个线程去恢复 getvalue() 的执行，然后co return 把值返回回来。所以我们可以看到 task2 这里的 x 的值是10，这里就是 await_resume 返回的值的传递。

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E) 

上图是一些简单的性能库的对比。大家也说了用 C++ 主要其实为了性能。那我们 C++ future_lite 的性能主要是和第三方 GitHub 的 cppcoro，以及我们自己内部的有栈协程去做对比。场景1是 UT 的一个例子，场景2、场景3、场景4都是从真实的Workload里抽出来的，比如说线程池加上海量的协程任务的执行。前面展示的协程调用链的执行，以及我们真正场景里用的批量分组执行的任务。可以看到我们 future_lite 其实比 cppcoro 都要快很多，而且在我们的场景2里比 libeasy 也要快很多。

  

**05**

**业务落地**

  

  

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E) 

最后讲一下我们的落地，我们目前在集团的搜索和交互分析里面都已经大规模落地了。采用的方式是从查询层到计算引擎层，再到存储层的链路的改造，是基于 C++ 20的协程，加上我们的future_lite库，加上我们内部自己开发的调度器，再加上最终访问存储的异步IO，实现全链路的同步和异步混合的这样一个改造。可以看到上图是其中一个 service 的情况，我们上线之前，Rt 基本上在50~100，有高有低，因为我们并不确定这回拿到数据是在 cache 里，还是真正地去打IO。我们升级完了以后 Rt 基本上全部都稳定在 25 左右，Timeout qps 基本直接就全变成0了。这里我其实没有写到的另一个好处就是之前同步和异步混合的这种场景，我们CPU其实是打不上来的，也就是说在压测或者真实场景里，我们的CPU最多可能就跑到50%、60%就已经到头了，但采用我们协程的这个方法，我们的高吞吐是完全能够上来的，CPU的利用率能够达到100%。

  

**06**

**未来**

  

  

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E) 

未来的规划还是两方面，我们应该还是会在LLVM方面继续做相关的工作，不光是协程方面，虽然我这里只列了协程。比如说继续去完善我们 DebugInfo 调试方面的一些痛点，CoroElide 帧消除的PAAS 也会有额外的一些扩展空间，因为我们现在的 LLVM 社区是基于inline 去做的，那有可能我们会去参加 IPO 过程间优化，能够增强 CoroElide。第三点也是大规模 multi_thread，其实是带 thread pool 或者说 executor 的情况下，去对于我们程序的stack 和 协程 frame 的 use-after-free 和 safety 做check。库方面我们明年下半年应该会考虑开源，功能上面我们会继续去完善有栈和无栈的结合，同步和异步的混合模式，以及我们刚才列的 Lazy 和 future/promise 的一个混合，同时我们考虑支持更多的IO 和 network 接口，无论是同步还是异步的，能够支持更多的场景。

  

**课程预告**

**![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)**

本课程紧密结合现代C++的泛型编程和面向对象两大编程范式最新发展，深入讲解现代软件设计的经典思想和设计原则，对GOF传统模式进行大幅度更新，融合前沿的优秀软件设计方法、惯用法和模式实践，帮助软件开发人员建立优秀的软件设计和架构能力。

  

**· 总课时20课时（每课时50分钟）**

**· 周末班：**共 10 天，每天 2 课时，每周六、日

**· 具体日期：**7月2-3、9-10、16-17、23-24、30-31日 晚20:00-21:40

  

课

程

咨

询

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**扫码登记，了解开课详情**

  

▼点击“**阅读原文**”了解课程详情

阅读原文

阅读 2315

​

发消息

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/YQ55UxynrohLM1OUXLiakCrhCmWppA6iclWuJYBgV8dMAkvT3O9EOF2CeA1PGZeVEYSm8d7v93oibPgzFeES7v3BA/300?wx_fmt=png&wxfrom=18)

CPP与系统软件联盟

14212

发消息