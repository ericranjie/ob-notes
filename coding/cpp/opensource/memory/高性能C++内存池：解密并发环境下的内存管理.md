
原创 往事敬秋风 深度Linux

 _2024年04月14日 09:10_ _湖南_

内存池是一种内存分配方式，又被称为固定大小区块规划（fixed-size-blocks allocation）。通常我们习惯直接使用new、malloc等API申请分配内存，这样做的缺点在于：由于所申请内存块的大小不定，当频繁使用时会造成大量的内存碎片并进而降低性能。

![](http://mmbiz.qpic.cn/mmbiz_png/dkX7hzLPUR0Ao40RncDiakbKx1Dy4uJicoqwn5GZ5r7zSMmpwHdJt32o95wdQmPZrBW038j8oRSSQllpnOUDlmUg/300?wx_fmt=png&wxfrom=19)

**深度Linux**

拥有15年项目开发经验及丰富教学经验，曾就职国内知名企业项目经理，部门负责人等职务。研究领域：Windows&Linux平台C/C++后端开发、Linux系统内核等技术。

181篇原创内容

公众号

在内核中有不少地方内存分配不允许失败，作为一个在这些情况下确保分配的方式，内核开发者创建了一个已知为内存池(或者是 "mempool" )的抽象， 一个内存池真实地只是一类后备缓存，它尽力一直保持一个空闲内存列表给紧急时使用。

## 一、内存池简介

### 1.1为什么要用内存池？

C++程序默认的内存管理（new，delete，malloc，free）会频繁地在堆上分配和释放内存，导致性能的损失，产生大量的内存碎片，降低内存的利用率。默认的内存管理因为被设计的比较通用，所以在性能上并不能做到极致。因此，很多时候需要根据业务需求设计专用内存管理器，便于针对特定数据结构和使用场合的内存管理，比如：内存池。

(1)内存碎片问题

造成堆利用率很低的一个主要原因就是内存碎片化。如果有未使用的存储器，但是这块存储器不能用来满足分配的请求，这时候就会产生内存碎片化问题。内存碎片化分为内部碎片和外部碎片。

- 内碎片：内部碎片是指一个已分配的块比有效载荷大时发生的。(假设以前分配了10个大小的字节，现在只用了5个字节，则剩下的5个字节就会内碎片)。内部碎片的大小就是已经分配的块的大小和他们的有效载荷之差的和。因此内部碎片取决于以前请求内存的模式和分配器实现(对齐的规则)的模式。
    
- 外碎片：假设系统依次分配了16byte、8byte、16byte、4byte，还剩余8byte未分配。这时要分配一个24byte的空间，操作系统回收了一个上面的两个16byte，总的剩余空间有40byte，但是却不能分配出一个连续24byte的空间，这就是外碎片问题。
    
![[Pasted image 20240910233414.png]]
![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

(2)申请效率问题

例如：我们上学家里给生活费一样，假设一学期的生活费是6000块。

方式1：开学时6000块直接给你，自己保管，自己分配如何花。

方式2：每次要花钱时，联系父母，父母转钱。

同样是6000块钱，第一种方式的效率肯定更高，因为第二种方式跟父母的沟通交互成本太高了。

同样的道理，程序就像是上学的我们，操作系统就像父母，频繁申请内存的场景下，每次需要内存，都像系统申请效率必然有影响。

### 1.2内存池原理

内存池的思想是，在真正使用内存之前，预先申请分配一定数量、大小预设的内存块留作备用。当有新的内存需求时，就从内存池中分出一部分内存块，若内存块不够再继续申请新的内存，当内存释放后就回归到内存块留作后续的复用，使得内存使用效率得到提升，一般也不会产生不可控制的内存碎片。

内存池设计

算法原理：

- 预申请一个内存区chunk，将内存中按照对象大小划分成多个内存块block
    
- 维持一个空闲内存块链表，通过指针相连，标记头指针为第一个空闲块
    
- 每次新申请一个对象的空间，则将该内存块从空闲链表中去除，更新空闲链表头指针
    
- 每次释放一个对象的空间，则重新将该内存块加到空闲链表头
    
- 如果一个内存区占满了，则新开辟一个内存区，维持一个内存区的链表，同指针相连，头指针指向最新的内存区，新的内存块从该区内重新划分和申请
    

通用内存分配和释放的缺点如下：

- 使用malloc/new申请分配堆内存时系统需要根据最先匹配、最优匹配或其它算法在内存空闲块表中查找一块空闲内存；使用free/delete释放堆内存时，系统可能需要合并空闲内存块，因此会产生额外开销。
    
- 频繁使用时会产生大量内存碎片，从而降低程序运行效率。
    
- 造成内存泄漏。
    

内存池（Memory Pool)是代替直接调用malloc/free、new/delete进行内存管理的常用方法，当申请内存空间时，会从内存池中查找合适的内存块，而不是直接向操作系统申请。

内存池技术的优点如下：

- 堆内存碎片很少。
    
- 内存申请/释放比malloc/new方式快。
    
- 检查任何一个指针是否在内存池中。
    
- 写一个堆转储(Heap-Dump)到硬盘。
    
- 内存泄漏检测(memory-leak detection)，当没有释放分配的内存时，内存池(Memory Pool)会抛出一个断言(assertion)。
    

内存池可以分为不定长内存池和定长内存池两类。不定长内存池的典型实现包括Apache Portable Runtime中的apr_pool和GNU lib C中的obstack，而定长内存池的实现则有boost_pool等。对于不定长内存池，不需要为不同的数据类型创建不同的内存池，其缺点是无法将分配出的内存回收到池内；对于定长内存池，在使用完毕后，可以将内存归还到内存池中，但需要为不同类型的数据结构创建不同的内存池，需要内存的时候要从相应的内存池中申请内存。

## 二、内存池实现方案

### 2.1设计问题

我们在设计内存池的实现方案时，需要考虑到以下问题：

内存池是否可以自动增长？

如果内存池的最大空间是固定的（也就是非自动增长），那么当内存池中的内存被请求完之后，程序就无法再次从内存池请求到内存。所以需要根据程序对内存的实际使用情况来确定是否需要自动增长。

内存池的总内存占用是否只增不减？

如果内存池是自动增长的，就涉及到了“内存池的总内存占用是否是只增不减”这个问题了。试想，程序从一个自动增长的内存池中请求了1000个大小为100KB的内存片，并在使用完之后全部归还给了内存池，而且假设程序之后的逻辑最多之后请求10个100KB的内存片，那么该内存池中的900个100KB的内存片就一直处于闲置状态，程序的内存占用就一直不会降下来。对内存占用大小有要求的程序需要考虑到这一点。

内存池中内存片的大小是否固定？

如果每次从内存池中的请求的内存片的大小如果不固定，那么内存池中的每个可用内存片的大小就不一致，程序再次请求内存片的时候，内存池就需要在“匹配最佳大小的内存片”和“匹配操作时间”上作出衡量。“最佳大小的内存片”虽然可以减少内存的浪费，但可能会导致“匹配时间”变长。

内存池是否是线程安全的？

是否允许在多个线程中同时从同一个内存池中请求和归还内存片？这个线程安全可以由内存池来实现，也可以由使用者来保证。

内存片分配出去之前和归还到内存池之后，其中的内容是否需要被清除？

程序可能出现将内存片归还给内存池之后，仍然使用内存片的地址指针进行内存读写操作，这样就会导致不可预期的结果。将内容清零只能尽量的（也不一定能）将问题抛出来，但并不能解决任何问题，而且将内容清零会消耗一定的CPU时间。所以，最终最好还是需要由内存池的使用者来保证这种安全性。

是否兼容std::allocator？

STL标准库中的大多类都支持用户提供一个自定义的内存分配器，默认使用的是std::allocator，如std::string：

```cpp
typedef basic_string<char, char_traits<char>, allocator<char> > string;
```

如果我们的内存池兼容std::allocator，那么我们就可以使用我们自己的内存池来替换默认的std::allocator分配器，如：

```cpp
typedef basic_string<char, char_traits<char>, MemoryPoll<char> > mystring
```

### 2.2常见内存池实现方案

（1）固定大小缓冲池

固定大小缓冲池适用于频繁分配和释放固定大小对象的情况。

（2）dlmalloc

dlmalloc 是一个内存分配器，由Doug Lea从1987年开始编写，目前最新版本为2.8.3，由于其高效率等特点被广泛使用和研究。

（3） SGI STL内存分配器

SGI STL allocator 是目前设计最优秀的 C++ 内存分配器之一，其内部free_list[16] 数组负责管理从 8 bytes到128 bytes不同大小的内存块（ chunk ），每一个内存块都由连续的固定大小（ fixed size block ）的很多 chunk 组成，并用指针链表连接。

（4）Loki小对象分配器

Loki 分配器使用vector管理数组，可以指定 fixed size block 的大小。free blocks分布在一个连续的大内存块中，free chunks 可以根据使用情况自动增长和减少合适的数目，避免内存分配得过多或者过少。

（5）Boost object_pool

Boost object_pool 可以根据用户具体应用类的大小来分配内存块，通过维护一个free nodes的链表来管理。可以自动增加nodes块，初始32个nodes，每次增加都以两倍数向system heap要内存块。object_pool 管理的内存块需要在其对象销毁的时候才返还给 system heap 。

（6）ACE_Cached_Allocator 和 ACE_Free_List

ACE 框架中包含一个可以维护固定大小的内存块的分配器，通过在 ACE_Cached_Allocator 中定义Free_list 链表来管理一个连续的大内存块，内存块中包含多个固定大小的未使用内存区块（ free chunk），同时使用ACE_unbounded_Set维护已使用的chuncks 。

（7）TCMalloc

Google开源项目gperftools提供了内存池实现方案。TCMalloc替换了系统的malloc，更加底层优化，性能更好。

### 2.3STL内存分配器

分配器(allocator))是C ++标准库的一个组件, 主要用来处理所有给定容器(vector，list，map等)内存的分配和释放。C ++标准库提供了默认使用的通用分配器std::allocator，但开发者可以自定义分配器。

GNU STL除了提供默认分配器，还提供了__pool_alloc、__mt_alloc、array_allocator、malloc_allocator 内存分配器。

- __pool_alloc ：SGI内存池分配器
    
- __mt_alloc ：多线程内存池分配器
    
- array_allocator ：全局内存分配，只分配不释放，交给系统来释放
    
- malloc_allocator ：堆std::malloc和std::free进行的封装
    

## 三、内存池设计

### 3.1为什么要使用内存池

- 解决内碎片问题
    
- 由于向内存申请的内存块都是比较大的，所以能够降低外碎片问题
    
- 一次性向内存申请一块大的内存慢慢使用，避免了频繁的向内存请求内存操作，提高内存分配的效率
    
- 但是内碎片问题无法避免，只能尽可能的降低
    

### 3.2内存池的演变

最简单的内存分配器，做一个链表指向空闲内存，分配就是取出一块来，改写链表，返回，释放就是放回到链表里面，并做好归并。注意做好标记和保护，避免二次释放，还可以花点力气在如何查找最适合大小的内存快的搜索上，减少内存碎片，有空你了还可以把链表换成伙伴算法。

- 优点： 实现简单
    
- 缺点： 分配时搜索合适的内存块效率低，释放回归内存后归并消耗大，实际中不实用。
    

定长内存分配器，即实现一个 FreeList，每个 FreeList 用于分配固定大小的内存块，比如用于分配 32字节对象的固定内存分配器，之类的。每个固定内存分配器里面有两个链表，OpenList 用于存储未分配的空闲对象，CloseList用于存储已分配的内存对象，那么所谓的分配就是从 OpenList 中取出一个对象放到 CloseList 里并且返回给用户，释放又是从 CloseList 移回到 OpenList。分配时如果不够，那么就需要增长 OpenList：申请一个大一点的内存块，切割成比如 64 个相同大小的对象添加到 OpenList中。这个固定内存分配器回收的时候，统一把先前向系统申请的内存块全部还给系统。

- 优点： 简单粗暴，分配和释放的效率高，解决实际中特定场景下的问题有效。
    
- 缺点： 功能单一，只能解决定长的内存需求，另外占着内存没有释放。
    
![[Pasted image 20240910233438.png]]
![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

哈希映射的FreeList池，在定长分配器的基础上，按照不同对象大小(8，16，32，64，128，256，512，1k…64K),构造十多个固定内存分配器，分配内存时根据要申请内存大小进行对齐然后查H表，决定到底由哪个分配器负责，分配后要在内存头部的 header 处写上 cookie，表示由该块内存哪一个分配器分配的，这样释放时候你才能正确归还。如果大于64K，则直接用系统的 malloc作为分配，如此以浪费内存为代价你得到了一个分配时间近似O(1)的内存分配器。这种内存池的缺点是假设某个 FreeList 如果高峰期占用了大量内存即使后面不用，也无法支援到其他内存不够的 FreeList，达不到分配均衡的效果。

- 优点：这个本质是定长内存池的改进，分配和释放的效率高。可以解决一定长度内的问题。
    
- 缺点：存在内碎片的问题，且将一块大内存切小以后，申请大内存无法使用。多线程并发场景下，锁竞争激烈，效率降低。
    

范例：sgi stl 六大组件中的空间配置器就是这种设计实现的。
![[Pasted image 20240910233443.png]]
![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

了解malloc底层原理

C 标准库函数malloc 在底层使用的是 —– 分离适配，使用这种方法，分配器维护着一个空闲链表数组，每个空闲链表被组织成某种类型的显示/隐式链表。每个链表包含大小不同的块，这些块的大小是大小类的成员，当要分配一个块时，我们确定了大小类之后，对适当的空闲链表做首次适配，查找一个合适的块，如果找到，那么可选地分割它，并将剩余的部分插入到适当的空闲链表中。如果每找到，那就搜索下一个更大的大小类的空闲链表，重复直到找到一个合适的块。如果空闲链表中没有合适的块，那么就向操作系统请求额外的堆存储器，从这个新的堆存储器中分配一个块，将剩余部分放置在适当的大小类中，当释放一个块时，我们执行合并，并将结果放在相应的空闲链表中。

- malloc优点: 使用自由链表的数组，提高分配释放效率；减少内存碎片，可以合并空闲的内存
    
- malloc缺点：为了维护隐式/显示链表需要维护一些信息，空间利用率不高；在多线程的情况下，会出现线程安全的问题，如果以加锁的方式解决，会大大降低效率。
    

## 四、内存池的具体实现

计划实现一个内存池管理的类MemoryPool，它具有如下特性：

- 内存池的总大小自动增长。
    
- 内存池中内存片的大小固定。
    
- 支持线程安全。
    
- 在内存片被归还之后，清除其中的内容。
    
- 兼容std::allocator。
    

因为内存池的内存片的大小是固定的，不涉及到需要匹配最合适大小的内存片，由于会频繁的进行插入、移除的操作，但查找比较少，故选用链表数据结构来管理内存池中的内存片。

MemoryPool中有2个链表，它们都是双向链表（设计成双向链表主要是为了在移除指定元素时，能够快速定位该元素的前后元素，从而在该元素被移除后，将其前后元素连接起来，保证链表的完整性）：

- data_element_ 记录以及分配出去的内存片。
    
- free_element_ 记录未被分配出去的内存片。
    

MemoryPool实现代码

代码中使用了std::mutex等C++11才支持的特性，所以需要编译器最低支持C++11：

```cpp
#ifndef PPX_BASE_MEMORY_POOL_H_#define PPX_BASE_MEMORY_POOL_H_#include <climits>#include <cstddef>#include <mutex>namespace ppx {    namespace base {        template <typename T, size_t BlockSize = 4096, bool ZeroOnDeallocate = true>        class MemoryPool {        public:            /* Member types */            typedef T               value_type;            typedef T*              pointer;            typedef T&              reference;            typedef const T*        const_pointer;            typedef const T&        const_reference;            typedef size_t          size_type;            typedef ptrdiff_t       difference_type;            typedef std::false_type propagate_on_container_copy_assignment;            typedef std::true_type  propagate_on_container_move_assignment;            typedef std::true_type  propagate_on_container_swap;            template <typename U> struct rebind {                typedef MemoryPool<U> other;            };            /* Member functions */            MemoryPool() noexcept;            MemoryPool(const MemoryPool& memoryPool) noexcept;            MemoryPool(MemoryPool&& memoryPool) noexcept;            template <class U> MemoryPool(const MemoryPool<U>& memoryPool) noexcept;            ~MemoryPool() noexcept;            MemoryPool& operator=(const MemoryPool& memoryPool) = delete;            MemoryPool& operator=(MemoryPool&& memoryPool) noexcept;            pointer address(reference x) const noexcept;            const_pointer address(const_reference x) const noexcept;            // Can only allocate one object at a time. n and hint are ignored
pointer allocate(size_type n = 1, const_pointer hint = 0);            void deallocate(pointer p, size_type n = 1);            size_type max_size() const noexcept;            template <class U, class... Args> void construct(U* p, Args&&... args);            template <class U> void destroy(U* p);            template <class... Args> pointer newElement(Args&&... args);            void deleteElement(pointer p);        private:            struct Element_ {                Element_* pre;                Element_* next;            };            typedef char* data_pointer;            typedef Element_ element_type;            typedef Element_* element_pointer;            element_pointer data_element_;            element_pointer free_element_;            std::recursive_mutex m_;            size_type padPointer(data_pointer p, size_type align) const noexcept;            void allocateBlock();            static_assert(BlockSize >= 2 * sizeof(element_type), "BlockSize too small.");        };        template <typename T, size_t BlockSize,  bool ZeroOnDeallocate>        inline typename MemoryPool<T, BlockSize, ZeroOnDeallocate>::size_type            MemoryPool<T, BlockSize, ZeroOnDeallocate>::padPointer(data_pointer p, size_type align)            const noexcept {            uintptr_t result = reinterpret_cast<uintptr_t>(p);            return ((align - result) % align);        }        template <typename T, size_t BlockSize,  bool ZeroOnDeallocate>        MemoryPool<T, BlockSize, ZeroOnDeallocate>::MemoryPool()            noexcept {            data_element_ = nullptr;            free_element_ = nullptr;        }        template <typename T, size_t BlockSize,  bool ZeroOnDeallocate>        MemoryPool<T, BlockSize, ZeroOnDeallocate>::MemoryPool(const MemoryPool& memoryPool)            noexcept :            MemoryPool() {        }        template <typename T, size_t BlockSize,  bool ZeroOnDeallocate>        MemoryPool<T, BlockSize, ZeroOnDeallocate>::MemoryPool(MemoryPool&& memoryPool)            noexcept {            std::lock_guard<std::recursive_mutex> lock(m_);            data_element_ = memoryPool.data_element_;            memoryPool.data_element_ = nullptr;            free_element_ = memoryPool.free_element_;            memoryPool.free_element_ = nullptr;        }        template <typename T, size_t BlockSize,  bool ZeroOnDeallocate>        template<class U>        MemoryPool<T, BlockSize, ZeroOnDeallocate>::MemoryPool(const MemoryPool<U>& memoryPool)            noexcept :            MemoryPool() {        }        template <typename T, size_t BlockSize,  bool ZeroOnDeallocate>        MemoryPool<T, BlockSize, ZeroOnDeallocate>&            MemoryPool<T, BlockSize, ZeroOnDeallocate>::operator=(MemoryPool&& memoryPool)            noexcept {            std::lock_guard<std::recursive_mutex> lock(m_);            if (this != &memoryPool) {                std::swap(data_element_, memoryPool.data_element_);                std::swap(free_element_, memoryPool.free_element_);            }            return *this;        }        template <typename T, size_t BlockSize,  bool ZeroOnDeallocate>        MemoryPool<T, BlockSize, ZeroOnDeallocate>::~MemoryPool()            noexcept {            std::lock_guard<std::recursive_mutex> lock(m_);            element_pointer curr = data_element_;            while (curr != nullptr) {                element_pointer prev = curr->next;                operator delete(reinterpret_cast<void*>(curr));                curr = prev;            }            curr = free_element_;            while (curr != nullptr) {                element_pointer prev = curr->next;                operator delete(reinterpret_cast<void*>(curr));                curr = prev;            }        }        template <typename T, size_t BlockSize,  bool ZeroOnDeallocate>        inline typename MemoryPool<T, BlockSize, ZeroOnDeallocate>::pointer            MemoryPool<T, BlockSize, ZeroOnDeallocate>::address(reference x)            const noexcept {            return &x;        }        template <typename T, size_t BlockSize,  bool ZeroOnDeallocate>        inline typename MemoryPool<T, BlockSize, ZeroOnDeallocate>::const_pointer            MemoryPool<T, BlockSize, ZeroOnDeallocate>::address(const_reference x)            const noexcept {            return &x;        }        template <typename T, size_t BlockSize,  bool ZeroOnDeallocate>        void            MemoryPool<T, BlockSize, ZeroOnDeallocate>::allocateBlock() {            // Allocate space for the new block and store a pointer to the previous one
data_pointer new_block = reinterpret_cast<data_pointer> (operator new(BlockSize));            element_pointer new_ele_pointer = reinterpret_cast<element_pointer>(new_block);            new_ele_pointer->pre = nullptr;            new_ele_pointer->next = nullptr;            if (data_element_) {                data_element_->pre = new_ele_pointer;            }            new_ele_pointer->next = data_element_;            data_element_ = new_ele_pointer;        }        template <typename T, size_t BlockSize,  bool ZeroOnDeallocate>        inline typename MemoryPool<T, BlockSize, ZeroOnDeallocate>::pointer            MemoryPool<T, BlockSize, ZeroOnDeallocate>::allocate(size_type n, const_pointer hint) {            std::lock_guard<std::recursive_mutex> lock(m_);            if (free_element_ != nullptr) {                data_pointer body =                    reinterpret_cast<data_pointer>(reinterpret_cast<data_pointer>(free_element_) + sizeof(element_type));                size_type bodyPadding = padPointer(body, alignof(element_type));                pointer result = reinterpret_cast<pointer>(reinterpret_cast<data_pointer>(body + bodyPadding));                element_pointer tmp = free_element_;                free_element_ = free_element_->next;                if (free_element_)                    free_element_->pre = nullptr;                tmp->next = data_element_;                if (data_element_)                    data_element_->pre = tmp;                tmp->pre = nullptr;                data_element_ = tmp;                return result;            }            else {                allocateBlock();                data_pointer body =                    reinterpret_cast<data_pointer>(reinterpret_cast<data_pointer>(data_element_) + sizeof(element_type));                size_type bodyPadding = padPointer(body, alignof(element_type));                pointer result = reinterpret_cast<pointer>(reinterpret_cast<data_pointer>(body + bodyPadding));                return result;            }        }        template <typename T, size_t BlockSize, bool ZeroOnDeallocate>        inline void            MemoryPool<T, BlockSize, ZeroOnDeallocate>::deallocate(pointer p, size_type n) {            std::lock_guard<std::recursive_mutex> lock(m_);            if (p != nullptr) {                element_pointer ele_p =                    reinterpret_cast<element_pointer>(reinterpret_cast<data_pointer>(p) - sizeof(element_type));                if (ZeroOnDeallocate) {                    memset(reinterpret_cast<data_pointer>(p), 0, BlockSize - sizeof(element_type));                }                if (ele_p->pre) {                    ele_p->pre->next = ele_p->next;                }                if (ele_p->next) {                    ele_p->next->pre = ele_p->pre;                }                if (ele_p->pre == nullptr) {                    data_element_ = ele_p->next;                }                ele_p->pre = nullptr;                if (free_element_) {                    ele_p->next = free_element_;                    free_element_->pre = ele_p;                }                else {                    ele_p->next = nullptr;                }                free_element_ = ele_p;            }        }        template <typename T, size_t BlockSize, bool ZeroOnDeallocate>        inline typename MemoryPool<T, BlockSize, ZeroOnDeallocate>::size_type            MemoryPool<T, BlockSize, ZeroOnDeallocate>::max_size()            const noexcept {            size_type maxBlocks = -1 / BlockSize;            return (BlockSize - sizeof(data_pointer)) / sizeof(element_type) * maxBlocks;        }        template <typename T, size_t BlockSize, bool ZeroOnDeallocate>        template <class U, class... Args>        inline void            MemoryPool<T, BlockSize, ZeroOnDeallocate>::construct(U* p, Args&&... args) {            new (p) U(std::forward<Args>(args)...);        }        template <typename T, size_t BlockSize, bool ZeroOnDeallocate>        template <class U>        inline void            MemoryPool<T, BlockSize, ZeroOnDeallocate>::destroy(U* p) {            p->~U();        }        template <typename T, size_t BlockSize, bool ZeroOnDeallocate>        template <class... Args>        inline typename MemoryPool<T, BlockSize, ZeroOnDeallocate>::pointer            MemoryPool<T, BlockSize, ZeroOnDeallocate>::newElement(Args&&... args) {            std::lock_guard<std::recursive_mutex> lock(m_);            pointer result = allocate();            construct<value_type>(result, std::forward<Args>(args)...);            return result;        }        template <typename T, size_t BlockSize, bool ZeroOnDeallocate>        inline void            MemoryPool<T, BlockSize, ZeroOnDeallocate>::deleteElement(pointer p) {            std::lock_guard<std::recursive_mutex> lock(m_);            if (p != nullptr) {                p->~value_type();                deallocate(p);            }        }    }}#endif // PPX_BASE_MEMORY_POOL_H_使用示例：
#include <iostream>
#include <thread>
using namespace std;class Apple {public:    Apple() {        id_ = 0;        cout << "Apple()" << endl;    }    Apple(int id) {        id_ = id;        cout << "Apple(" << id_ << ")" << endl;    }    ~Apple() {        cout << "~Apple()" << endl;    }    void SetId(int id) {        id_ = id;    }    int GetId() {        return id_;    }private:    int id_;};void ThreadProc(ppx::base::MemoryPool<char> *mp) {    int i = 0;    while (i++ < 100000) {        char* p0 = (char*)mp->allocate();        char* p1 = (char*)mp->allocate();        mp->deallocate(p0);        char* p2 = (char*)mp->allocate();        mp->deallocate(p1);                mp->deallocate(p2);    }}int main(){    ppx::base::MemoryPool<char> mp;    int i = 0;    while (i++ < 100000) {        char* p0 = (char*)mp.allocate();        char* p1 = (char*)mp.allocate();        mp.deallocate(p0);        char* p2 = (char*)mp.allocate();        mp.deallocate(p1);        mp.deallocate(p2);    }    std::thread th0(ThreadProc, &mp);    std::thread th1(ThreadProc, &mp);    std::thread th2(ThreadProc, &mp);    th0.join();    th1.join();    th2.join();    Apple *apple = nullptr;    {        ppx::base::MemoryPool<Apple> mp2;        apple = mp2.newElement(10);        int a = apple->GetId();        apple->SetId(10);        a = apple->GetId();        mp2.deleteElement(apple);    }    apple->SetId(12);    int b = -4 % 4;    int *a = nullptr;    {        ppx::base::MemoryPool<int, 18> mp3;        a =  mp3.allocate();        *a = 100;        //mp3.deallocate(a);        int *b =  mp3.allocate();        *b = 200;        //mp3.deallocate(b);        mp3.deallocate(a);        mp3.deallocate(b);        int *c = mp3.allocate();        *c = 300;    }    getchar();    return 0;}
```

2023年往期回顾

[C/C++发展方向（强烈推荐！！）](http://mp.weixin.qq.com/s?__biz=Mzg4NDQ0OTI4Ng==&mid=2247487749&idx=1&sn=e57e6f3df526b7ad78313d9428e55b6b&chksm=cfb9586cf8ced17a8c7830e380a45ce080c2b8258e145f5898503a779840a5fcfec3e8f8fa9a&scene=21#wechat_redirect)  

[Linux内核源码分析（强烈推荐收藏！](http://mp.weixin.qq.com/s?__biz=Mzg4NDQ0OTI4Ng==&mid=2247487832&idx=1&sn=bf0468e26f353306c743c4d7523ebb07&chksm=cfb95831f8ced127ca94eb61e6508732576bb150b2fb2047664f8256b3da284d0e53e2f792dc&scene=21#wechat_redirect)[）](http://mp.weixin.qq.com/s?__biz=Mzg4NDQ0OTI4Ng==&mid=2247487832&idx=1&sn=bf0468e26f353306c743c4d7523ebb07&chksm=cfb95831f8ced127ca94eb61e6508732576bb150b2fb2047664f8256b3da284d0e53e2f792dc&scene=21#wechat_redirect)  

[从菜鸟到大师，用Qt编写出惊艳世界应用](http://mp.weixin.qq.com/s?__biz=Mzg4NDQ0OTI4Ng==&mid=2247488117&idx=1&sn=a83d661165a3840fbb23d0e62b5f303a&chksm=cfb95b1cf8ced20ae63206fe25891d9a37ffe76fd695ef55b5506c83aad387d55c4032cb7e4f&scene=21#wechat_redirect)  

[存储全栈开发：构建高效的数据存储系统](http://mp.weixin.qq.com/s?__biz=Mzg4NDQ0OTI4Ng==&mid=2247487696&idx=1&sn=b5ebe830ddb6798ac5bf6db4a8d5d075&chksm=cfb959b9f8ced0af76710c070a6db2677fb359af735e79c6378e82ead570aa1ce5350a146793&scene=21#wechat_redirect)

[突破性能瓶颈：释放DPDK带来无尽潜力](http://mp.weixin.qq.com/s?__biz=Mzg4NDQ0OTI4Ng==&mid=2247488150&idx=1&sn=2cc5ace4391d4eed060fa463fc73dd92&chksm=cfb95bfff8ced2e9589eecd66e7c6e906f28626bef6c1bce3fa19ae2d1404baa25ba3ebfedaa&scene=21#wechat_redirect)

[嵌入式音视频技术：从原理到项目应用](http://mp.weixin.qq.com/s?__biz=Mzg4NDQ0OTI4Ng==&mid=2247488135&idx=1&sn=dcc56a3af5401c338944b9b94f24c699&chksm=cfb95beef8ced2f8231c647033188be3ded88cbacf7d75feab3613f4d7f8fd8e4c8106ac88cf&scene=21#wechat_redirect)

[](http://mp.weixin.qq.com/s?__biz=MzIwNzA1OTA2OQ==&mid=2657215412&idx=1&sn=d3c64ac21056b84814db9903b656853a&chksm=8c8d62a6bbfaebb09df1f727e201c401a1663596a719fa9cb4c7859158e62ebedb5bd5f13b8f&scene=21#wechat_redirect)[C++游戏开发，基于魔兽开源后端框架](http://mp.weixin.qq.com/s?__biz=Mzg4NDQ0OTI4Ng==&mid=2247486920&idx=1&sn=82ca8a72473850e431c01e7c39d8c8c0&chksm=cfb944a1f8cecdb7051863f58d334ff3dea8afae1ecd8bc693287b2f483d2ce8f94eed565253&scene=21#wechat_redirect)

linux内核108

linux系统31

网络编程11

linux内核 · 目录

上一篇深入了解Linux内核，实战演练与代码解析下一篇C++无锁队列的探索：多种实现方案及性能对比

阅读 1.5万

​