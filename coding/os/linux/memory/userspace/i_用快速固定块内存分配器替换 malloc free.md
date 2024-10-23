
David Lafreniere 控制工程研习 _2024年03月31日 08:26_ _北京_

原文链接：

https://www.codeproject.com/Articles/1084801/Replace-malloc-free-with-a-Fast-Fixed-Block-Memory

在本文中，C 库 malloc/free 被替换为替代的固定内存块版本 xmalloc() 和 xfree()。首先，本文会简要解释底层 Allocator 存储回收方法，然后再解释 xallocator 的工作原理。

# **介绍**

自定义固定块分配器（Custom fixed block allocators）是专门的内存管理器，用于解决全局堆的性能问题。在《高效 C++ 固定块内存分配器》一文中，我实现了一个分配器类来提高速度并消除碎片堆内存故障的可能性。在这篇最新文章中，Allocator 类被用作 xallocator 实现的基础来替换 malloc() 和 free()。

与大多数固定块分配器不同，\*\*xallocator 实现能够以完全动态的方式运行，无需预先了解块大小或块数量。分配器会为您处理所有固定块管理。它完全可移植到任何基于 PC 的系统或嵌入式系统。\*\*此外，它还可以通过内存统计信息深入了解您的动态使用情况。

在本文中，我将 C 库 malloc/free 替换为替代的固定内存块版本 xmalloc() 和 xfree()。首先，我将简要解释底层 Allocator 存储回收方法，然后介绍 xallocator 的工作原理。

# **储存回收(Storage Recycling)**

内存管理方案的基本原理是回收在对象分配期间获得的内存。\*\*一旦创建了对象的存储空间，它就永远不会返回到堆中。相反，内存被回收，允许相同类型的另一个对象重用该空间。\*\*我已经实现了一个名为 Allocator 的类来表达该技术。

当应用程序使用 Allocator 进行删除时，单个对象的内存块将被释放以供再次使用，但实际上并未释放回内存管理器。\*\*释放的块保留在称为空闲列表（free-list）的链接列表中，以便再次分配给相同类型的另一个对象。\*\*对于每个分配请求，Allocator 首先检查空闲列表中是否存在现有内存块。仅当没有可用的时才会创建新的。根据Allocator所需的行为，存储来自全局堆或具有以下三种操作模式之一的静态内存池：

1. 堆块（Heap blocks）

1. 堆池（Heap pool）

1. 静态池（Static pool）

# **堆vs池**

当空闲列表无法提供现有块时，Allocator 类能够从堆或内存池创建新块。如果使用池，则必须预先指定对象的数量。使用对象总数，创建一个足够大的池来处理最大数量的实例。**另一方面，从堆获取块内存则没有这样的数量限制——在存储允许的情况下构造尽可能多的新对象。**

堆块模式根据需要从全局堆中为单个对象分配新的内存块以满足内存请求。解除分配会将块放入空闲列表中以供以后重用。当空闲列表为空时，从堆中创建新的块可以使您不必设置对象限制。这种方法提供了类似动态的操作，因为块的数量可以在运行时扩展。缺点是在块创建期间失去确定性执行。

堆池模式从全局堆创建一个池来保存所有块。该池是在构造 Allocator 对象时使用 new 运算符创建的。然后，Allocator 在分配期间从池中提供内存块。

静态池模式使用单个内存池（通常位于静态内存中）来保存所有块。静态内存池不是由 Allocator 创建的，而是由类的用户提供的。

堆池和静态池模式提供一致的分配执行时间，因为内存管理器从不参与获取单个块。这使得新操作非常快速且具有确定性。

Allocator 构造函数控制操作模式。

```c++
class Allocator { public:     Allocator(size_t size, UINT objects=0, CHAR* memory=NULL, const CHAR* name=NULL); ...
```

有关Allocator的更多信息，请参阅“高效的 C++ 固定块内存分配器”。

# **xallocator**

xallocator 模块有六个主要 API：

- xmalloc

- xfree

- xrealloc

- xalloc_stats

- xalloc_init

- xalloc_destroy

xmalloc() 与 malloc() 等效，并且使用方式完全相同。给定多个字节，该函数返回一个指向请求大小的内存块的指针。

```c
void* memory1 = xmalloc(100);
```

内存块至少与用户请求一样大，但由于固定块分配器实现，实际上可能更大。 额外的过度分配的内存称为松弛，但通过微调块大小，可以最大限度地减少浪费，正如我将在本文后面解释的那样。

xfree() 是 free() 的 CRT 等价物。只需向 xfree() 传递一个指向先前分配的 xmalloc() 块的指针即可释放内存以供重用。

```c
xfree(memory1);
```

xrealloc() 的行为与 realloc() 相同，它扩展或收缩内存块，同时保留内存块内容。

```c++
char* memory2 = (char*)xmalloc(24);     strcpy(memory2, "TEST STRING"); memory2 = (char*)xrealloc(memory2, 124); xfree(memory2);   
```

xalloc_stats() 将分配器使用统计信息输出到标准输出流。输出可让您深入了解正在使用的分配器实例数量、正在使用的块、块大小等。

xalloc_init() 必须在任何工作线程启动之前调用一次，或者对于嵌入式系统，在操作系统启动之前调用一次。在 C++ 应用程序中，会自动为您调用此函数。但是，在某些情况下（通常在嵌入式系统上）最好手动调用 xalloc_init()，以避免自动 xalloc_init()/xalloc_destroy() 调用机制涉及的少量内存开销。

当应用程序退出时，将调用 xalloc_destroy() 以清理任何动态分配的资源。在 C++ 应用程序中，当应用程序终止时会自动调用此函数。除非在仅在 C 文件中使用 xallocator 的程序中，否则绝不能手动调用 xalloc_destroy()。

现在，何时在 C++ 应用程序中调用 xalloc_init() 和 xalloc_destroy() 并不那么容易。静态对象会出现问题。如果过早调用 xalloc_destroy()，则当在程序退出时调用静态对象析构函数时，可能仍然需要 xallocator。以这个类为例：

```c++
class MyClassStatic { public:     MyClassStatic()      {          memory = xmalloc(100);      }     ~MyClassStatic()      {          xfree(memory);      } private:     void* memory; };
```

现在在文件范围内创建此类的静态实例。

```c
static MyClassStatic myClassStatic;
```

由于对象是静态的，因此 MyClassStatic 构造函数将在 main() 之前调用，这是可以的，我将在下面的“移植问题”部分中解释。然而，析构函数在 main() 退出后被调用，如果处理不当，这是不行的。问题变成了如何确定何时销毁xallocator动态分配的资源。如果在 main() 退出之前调用 xalloc_destroy()，则当 ~MyClassStatic() 尝试调用 xfree() 时，xallocator 将被销毁，从而导致错误。

解决方案的关键来自于C++标准中的一个保证：

# **引用：**

_“在同一翻译单元的命名空间范围内定义并动态初始化的静态存储持续时间的对象应按照其定义在翻译单元中出现的顺序进行初始化。”_

换句话说，static 对象构造函数的调用顺序与文件（翻译单元）中定义的顺序相同。破坏将颠倒这个顺序。因此，xallocator.h 定义了一个 XallocInitDestroy 类并创建它的 static 实例。

```c++
class XallocInitDestroy { public:     XallocInitDestroy();     ~XallocInitDestroy(); private:     static INT refCount; }; static XallocInitDestroy xallocInitDestroy;
```

构造函数跟踪创建的静态实例的总数，并在第一次构造时调用 xalloc_init()。

```c++
INT XallocInitDestroy::refCount = 0; XallocInitDestroy::XallocInitDestroy()  {      // Track how many static instances of XallocInitDestroy are created     
if (refCount++ == 0)         xalloc_init(); }
```

当最后一个实例被销毁时，析构函数会自动调用 xalloc_destroy()。

```c++
XallocDestroy::~XallocDestroy() {     // Last static instance to have destructor called?     
if (--refCount == 0)         xalloc_destroy(); }
```

当在翻译单元中包含 xallocator.h 时，将首先声明 xallocInitDestroy，因为 #include 位于用户代码之前。这意味着依赖 xallocator 的任何其他静态用户类都将在#include“xallocator.h”之后声明。这保证了在执行所有用户静态类析构函数之后调用 ~XallocInitDestroy()。使用这种技术，当程序退出时，xalloc_destroy() 会被安全地调用，而不会有 xallocator 过早销毁的危险。

XallocInitDestroy 是一个空类，因此大小为 1 字节。对于包含 xallocator.h 的每个翻译单元，此功能的成本为 1 字节，但以下情况除外。

1. 在应用程序永远不会退出的嵌入式系统上，除非使用 STATIC_POOLS 模式，否则不需要该技术。所有对 XallocInitDestroy 的引用都可以安全删除，并且永远不需要调用 xalloc_destroy()。但是，您现在必须在使用 xallocator API 之前在 main() 中手动调用 xalloc_init()。

1. 当 xallocator 包含在 C 翻译单元中时，不会创建 XallocInitDestroy 的静态实例。在这种情况下，必须在 main() 中调用 xalloc_init() 并在 main() 退出之前调用 xalloc_destroy()。

要启用或禁用自动 xallocator 初始化和销毁，请使用下面的#define：

```c
#define AUTOMATIC_XALLOCATOR_INIT_DESTROY
```

在 PC 或类似配备的高 RAM 系统上，这 1 个字节是微不足道的，但反过来又确保了程序退出期间静态类实例中 xallocator 操作的安全。它还使您不必调用 xalloc_init() 和 xalloc_destroy()，因为这是自动处理的。

# **重载new和delete**

为了使 xallocator 真正易于使用，我创建了一个宏来重载类中的 new/delete 并将内存请求路由到 xmalloc()/xfree()。只需在类定义中的任意位置添加宏 XALLOCATOR 即可。

```c++
class MyClass  {     XALLOCATOR     // remaining class definition
				};
```

使用宏，类的 new/delete 通过重载的 new/delete 将请求路由到 xallocator。

```c++
// Allocate MyClass using fixed block allocator 
MyClass* myClass = new MyClass(); delete myClass;
```

一个巧妙的技巧是将 XALLOCATOR 放置在继承层次结构的基类中，以便所有派生类都使用 xallocator 进行分配/释放。例如，假设您有一个带有基类的 GUI 库。

```c++
class GuiBase  {     XALLOCATOR     // remaining class definition 
				};
```

现在，任何 GuiBase 派生类（按钮、小部件等）在调用 new/delete 时都使用 xallocator，而无需向每个派生类添加 XALLOCATOR。这是一种强大的方法，可以使用单个宏语句为整个层次结构启用固定块分配。

原文后面还有代码部署的示例，详细可以看原文。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

阅读 127

​

发消息

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/hpoqGPEUNGbsumNmseibc6N4waFGsCCoS2A5F72qfmSfCrORcDVrkUqPnaw7YiaBiaE6pTWavHwYweNoCnSGyB2Qw/300?wx_fmt=png&wxfrom=18)

控制工程研习

5113

发消息
