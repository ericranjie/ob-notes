## 导语

c++ 内存管理学习自侯捷。

下面是本次对C++内存管理一些笔记。

## 1.四种内存分配与释放

[https://mmbiz.qpic.cn/mmbiz_png/WwIcQHkD5mcpGpZWC00BdwOPWk1hdJFtx40yAjnzrSPV9VnqhgSfpTRVvaBu92mL4UpQrrrYyv0L0KWNfljoUg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1](https://mmbiz.qpic.cn/mmbiz_png/WwIcQHkD5mcpGpZWC00BdwOPWk1hdJFtx40yAjnzrSPV9VnqhgSfpTRVvaBu92mL4UpQrrrYyv0L0KWNfljoUg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==

在编程时可以通过上图的几种方法直接或间接地操作内存。下面将介绍四种C++内存操作方法：

对于GNU C：四种分配与释放方式如下：

```cpp
   // C函数
   void *p1 = malloc(512);
   *(int *) p1 = 100;
   cout << *(int *) p1 << endl;
   free(p1);

   // C++表达式
   int *p2 = new int(10);
   cout << *p2 << endl;
   delete p2;

   // C++函数 实际上等价于上述malloc与free
   void *p3 = ::operator new(512);
   *(int *) p3 = 103;
   cout << *(int *) p3 << endl;
  ::operator delete(p3);

   //C++标准库
   printf("hello gcc %d\\n", __GNUC__);
#ifdef __GNUC__
// 以下函数都是non-static,一定要通过object调用，以下分配7个单元，而不是7个字节
   int *p4 = allocator<int>().allocate(7);
   *p4 = 9;
   cout << *p4 << endl;
   allocator<int>().deallocate((int *) p4, 7);

   /**
    * void *p = alloc::allocate(512); 分配512bytes
    * alloc::deallocate(p,512);
    */
   // __pool_alloc等价于之前的alloc 9个单元
   int *p5 = __gnu_cxx::__pool_alloc<int>().allocate(9);
   *p5 = 10;
   cout << *p5 << endl;
   __gnu_cxx::__pool_alloc<int>().deallocate((int *) p5, 9);
#endif
```

## 2.new/delete表达式

### 2.1 new表达式

当使用`operator new`：

```cpp
// 下面这个是new expression,而operator new 是函数
Complex* pc = new Complex(1,2);
```

上述会被编译器转为：

```cpp
Complex *pc;
try {
// operator new 实现自 new_op.cc
   void* mem = operator new(sizeof(Complex)); //allocate 分配内存
   pc = static_cast<Complex*>(mem);    // cast 转型 以符合对应的类型，这里对应为Complex*
   pc->Complex::Complex(1,2); // construct
   // 注意：只有编译器才可以像上面那样直接呼叫ctor 欲直接调用ctor可通用placement new: new(p) Complex(1,2);
}
catch(std::bad_alloc) {
   // 若allocation失败就不执行constructor
}
```

new操作背后编译器做的事：

- 第一步通过operator new()操作分配一个目标类型的内存大小，这里是Complex的大小；
- 第二步通过static_cast将得到的内存块强制转换为目标类型指针，这里是Complex*
- 第三版调用目标类型的构造方法，但是需要注意的是，直接通过pc->Complex::Complex(1, 2)这样的方法调用构造函数只有编译器可以做，用户这样做将产生错误。

**注意：operator new()操作的内部是调用了malloc()函数。**

`operator new()`具体实现源代码见：

> [https://github.com/gcc-mirror/gcc/blob/master/libstdc%2B%2B-v3/libsupc%2B%2B/new_op.cc](https://github.com/gcc-mirror/gcc/blob/master/libstdc%2B%2B-v3/libsupc%2B%2B/new_op.cc)

### 2.2 delete表达式

对于上述delete调用，

```cpp
delete pc;
pc->~Complex();  //先析构
operator delete(pc);   //然后释放内存
```

delete操作步骤：

- 第一步调用了对象的析构函数
- 第二步通过operator delete()函数释放内存，本质上也是调用了free函数。

`operator delete()`具体实现源代码见：

> [https://github.com/gcc-mirror/gcc/blob/master/libstdc%2B%2B-v3/libsupc%2B%2B/del_op.cc](https://github.com/gcc-mirror/gcc/blob/master/libstdc%2B%2B-v3/libsupc%2B%2B/del_op.cc)

## 3.array new/array delete

### 3.1 array

[https://mmbiz.qpic.cn/mmbiz_png/WwIcQHkD5mcpGpZWC00BdwOPWk1hdJFtJPlF3k3FRBhIJxJcnvOvA8XAugOSgQ5ACD5baOMWukfytax2hFsNNw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1](https://mmbiz.qpic.cn/mmbiz_png/WwIcQHkD5mcpGpZWC00BdwOPWk1hdJFtJPlF3k3FRBhIJxJcnvOvA8XAugOSgQ5ACD5baOMWukfytax2hFsNNw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

上图主要展示的是关于array new内存分配的大致情况。

当new一个数组对象时（例如 new Complex[3]），编译器将分配一块内存，这块内存首部是关于对象内存分配的一些标记，然后下面会分配三个连续的对象内存，在使用delete释放内存时需要使用delete[]。

**什么情况下发生内存泄露？**

如果不使用delete[]，只是使用delete只会将分配的三块内存空间释放，但不会调用对象的析构函数，如果对象内部还使用了new指向其他空间，如果指向的该空间里的对象的析构函数没有意义，那么不会造成问题，**如果有意义，那么由于该部分对象析构函数不会调用，那么将会导致内存泄漏**。

图中new string[3]便是一个例子，虽然str[0]、str[1]、str[2]被析构了，但只是调用了str[0]的析构函数，其他对象的析构函数不被调用，这里就会出问题。

其中的cookie保存的是delete[]里面的数据，比如delete几次。

### 3.2 演示数组对象创建与析构过程

[https://mmbiz.qpic.cn/mmbiz_png/WwIcQHkD5mcpGpZWC00BdwOPWk1hdJFtvFRa2vZK0FzYYozhxGhwwX1VQpFKpLLhZlBaWicuo05hoGtQKtQ2h8w/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1](https://mmbiz.qpic.cn/mmbiz_png/WwIcQHkD5mcpGpZWC00BdwOPWk1hdJFtvFRa2vZK0FzYYozhxGhwwX1VQpFKpLLhZlBaWicuo05hoGtQKtQ2h8w/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

构造函数调用顺序是按照构建对象顺序来执行的，但是析构函数执行却相反。

构造函数：自上而下；析构函数：自下而上。

### 3.3 malloc基本构成

[https://mmbiz.qpic.cn/mmbiz_png/WwIcQHkD5mcpGpZWC00BdwOPWk1hdJFtZOzmrH3zfvrOJnMAMRuIyu7RqBhoXNmwKpA2iaLCWgXfH9IEB9ia2geg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1](https://mmbiz.qpic.cn/mmbiz_png/WwIcQHkD5mcpGpZWC00BdwOPWk1hdJFtZOzmrH3zfvrOJnMAMRuIyu7RqBhoXNmwKpA2iaLCWgXfH9IEB9ia2geg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

如果使用new分配十个内存的int，内存空间如上图所示，首先内存块会有一个头和尾，黄色部分为debug信息，灰色部分才是真正使用到的内存，蓝色部分的12bytes是为了让该内存块以16字节对齐。在这个例子中delete pi和delete[] pi效果是一样的，因为int没有析构函数。但是如果释放的对象的析构函数有意义，array delet就必须采用delete[]，否则发生内存泄露。

## 4.placement new

[https://mmbiz.qpic.cn/mmbiz_png/WwIcQHkD5mcpGpZWC00BdwOPWk1hdJFt4CQXXlzPiaicGCPGpNLDqL5Znjibv8YkZaBibuJbxLhHOb3uvRBicjxj8Ww/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1](https://mmbiz.qpic.cn/mmbiz_png/WwIcQHkD5mcpGpZWC00BdwOPWk1hdJFt4CQXXlzPiaicGCPGpNLDqL5Znjibv8YkZaBibuJbxLhHOb3uvRBicjxj8Ww/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

```cpp
char *buf = new char[sizeof(Complex) * 3];
Complex *pc = new(buf)Complex(1, 2);
delete[]buf;
```

上述被编译器编译为：

```cpp
Complex *pc;
try
   void* mem = operator new(sizeof(Complex),buf); //allocate
   pc= static_cast<Complex*>(mem);//cast
   pc->Complex::Complex(1,2);//construct
} catch (std::bad_alloc) {
   // 若allocation失败就不执行construct
}
```

值得注意的是，这里采用的`operator new`有两个参数，我们在下面源码中：

> [https://github.com/gcc-mirror/gcc/blob/master/libstdc%2B%2B-v3/libsupc%2B%2B/new](https://github.com/gcc-mirror/gcc/blob/master/libstdc%2B%2B-v3/libsupc%2B%2B/new)

看到：

```cpp
_GLIBCXX_NODISCARD inline void* operator new(std::size_t, void* __p) _GLIBCXX_USE_NOEXCEPT
{ return __p; }
```

因此得出，没有做任何事，直接返回buf， 因此placement new 就等同于调用构造函数。也没有所谓的operator delete ,因为placement new根本没有分配memory。

## 5.重载

### 5.1 C++内存分配的途径

[https://mmbiz.qpic.cn/mmbiz_png/WwIcQHkD5mcpGpZWC00BdwOPWk1hdJFtBArZy8fkQuhZUO2Kib1CuvvgmcCYSfQvibVHnkG1Pmxr54SExueuwTJQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1](https://mmbiz.qpic.cn/mmbiz_png/WwIcQHkD5mcpGpZWC00BdwOPWk1hdJFtBArZy8fkQuhZUO2Kib1CuvvgmcCYSfQvibVHnkG1Pmxr54SExueuwTJQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

如果是正常情况下，调用new之后走的是第二条路线，如果在类中重载了operator new()，那么走的是第一条路线，但最后还是要调用到系统的::operator new()函数，这在后续的例子中会体现。

[https://mmbiz.qpic.cn/mmbiz_png/WwIcQHkD5mcpGpZWC00BdwOPWk1hdJFtuH8evibpAkM8NuREc3jticVqDGEyyylF4Bibp8Fos2qJ0e6AdoBgJy1Cw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1](https://mmbiz.qpic.cn/mmbiz_png/WwIcQHkD5mcpGpZWC00BdwOPWk1hdJFtuH8evibpAkM8NuREc3jticVqDGEyyylF4Bibp8Fos2qJ0e6AdoBgJy1Cw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

对于GNU C，背后使用的allocate()函数最后也是调用了系统的::operator new()函数。

### 5.2 重载new 和 delete

[https://mmbiz.qpic.cn/mmbiz_png/WwIcQHkD5mcpGpZWC00BdwOPWk1hdJFtViaRHTwI85iaG23aCmCOQ2tUTNFYiceaAEq75rISxg3odYibwBG27ibcJQg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1](https://mmbiz.qpic.cn/mmbiz_png/WwIcQHkD5mcpGpZWC00BdwOPWk1hdJFtViaRHTwI85iaG23aCmCOQ2tUTNFYiceaAEq75rISxg3odYibwBG27ibcJQg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

上面这张图演示了如何重载系统的::operator new()函数，该方法最后也是模拟了系统的做法，效果和系统的方法一样，但一般不推荐重载::operator new()函数，因为它对全局有影响，如果使用不当将造成很大的问题。

[https://mmbiz.qpic.cn/mmbiz_png/WwIcQHkD5mcpGpZWC00BdwOPWk1hdJFtr3pAlxdKxCaUDgiaPn1SPOVL05sBVs9DQmwCbrprmfTMZia8WiaZbAn9A/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1](https://mmbiz.qpic.cn/mmbiz_png/WwIcQHkD5mcpGpZWC00BdwOPWk1hdJFtr3pAlxdKxCaUDgiaPn1SPOVL05sBVs9DQmwCbrprmfTMZia8WiaZbAn9A/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

如果是在类中重载operator new()方法，那么该方法有N多种形式，但必须保证函数参数列表第一个参数是size_t类型变量；对于operator delete()，第一个参数必须是void* 类型，第二个size_t是可选项，可以去掉。

data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==

对于operator new[]和operator delete[]函数的重载，和前面类似。

## **6.pre-class allocator1**

前面把基本元素的重载元素学完了，例如：new、operator new、array new等等。万事俱备，现在可以开始一个class进行内存管理。

对于malloc来说，大家都有一个误解，以为它很慢，其实它不慢，后面会讲到。无论如何，减少malloc的调用次数，总是很好的，所以设计class者，可以先挖一块，只使用一次malloc，使用者使用，就只需要调用一次malloc，这样就是一个小型的内存管理。

除了降低malloc次数之外，还需要降低cookie用量。前面提到一次malloc需要一组(两个)cookie，总共8字节。

所以，如果一次要1000个大小，这1000个切下来，都是不带cookie，只有1000个一整包上下带cookie。所以内存池的设计就是一整块，一个池塘。这一大块设计不但要提升速度，而且要降低浪费率。所以内存管理目标就是，一个是速度，一个是空间。

每次挖一大块，需要指针把他们穿起来，如下图右边链表结构，基于这个考量，下面例子中设计了next指针。此时碰到了一个困惑：多设计了一个指针，去除了cookie，却膨胀率100%(int i 占4字节，指针也是4字节)。

使用者使用new的时候，就会被接管到`operator new`这个函数来，delete类似。

**分配：**`operator new`就是挖一大块，里面主要做的就是指针操作与转型。其中`freeStore`指向头，`operator new`返回的就是`freeStore`表头。

**回收：**当使用者delete一个Scree，就会先调用析构函数，然后调用释放内存函数，`operator delete`接管了这个任务，接收到一个指针。就把这个链表回收到单向链表之中。单向链表始终都有一个头，所以回收动作最快放在链表开头。

[https://mmbiz.qpic.cn/mmbiz_png/WwIcQHkD5mcpGpZWC00BdwOPWk1hdJFt78MP1RmsD7nEPWIayicy4bPxkIUfgfhX1FLXSMR6Bk9ibiaCQkbQFupTQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1](https://mmbiz.qpic.cn/mmbiz_png/WwIcQHkD5mcpGpZWC00BdwOPWk1hdJFt78MP1RmsD7nEPWIayicy4bPxkIUfgfhX1FLXSMR6Bk9ibiaCQkbQFupTQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

[https://mmbiz.qpic.cn/mmbiz_png/WwIcQHkD5mcpGpZWC00BdwOPWk1hdJFtBWqcErodV6LPZQfQnIeJDgnhW5y3COHG2WWfsOwWRKGpXKJEfsZo7g/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1](https://mmbiz.qpic.cn/mmbiz_png/WwIcQHkD5mcpGpZWC00BdwOPWk1hdJFtBWqcErodV6LPZQfQnIeJDgnhW5y3COHG2WWfsOwWRKGpXKJEfsZo7g/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

## **7.pre-class allocator2**

data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==

这里与上述不同之处在于使用union设计，这里带来了一个观念：`嵌入式指针`，embedding pointer。

分配与释放同前面6。

嵌入式指针：rep占16字节，next占前8字节。

```cpp
union {
   AirplaneRep rep;  //此針對 used object
   Airplane* next;   //此針對 free list
};
```

借用一个东西的前8字节当指针用，这样整体上可以节省空间，这是一个很好的想法，在内存管理中都是这么来用。

最后，6与7中的`operator delete`并没有free掉，只是回收到单向链表中。这样子好？

这种当然不好，技术难点非常高，后面谈！虽然没有还给操作系统，但不能说它内存泄露，因为这些都在它的"手上"。

[https://mmbiz.qpic.cn/mmbiz_png/WwIcQHkD5mcpGpZWC00BdwOPWk1hdJFtTITGyHVPn6v9xdFEoQPRFyXIFwTRlTyybc0DiapcUofggYu14picSlvA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1](https://mmbiz.qpic.cn/mmbiz_png/WwIcQHkD5mcpGpZWC00BdwOPWk1hdJFtTITGyHVPn6v9xdFEoQPRFyXIFwTRlTyybc0DiapcUofggYu14picSlvA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

## **8.static allocator3**

**不要把内存分配与回收写在各个class中，而要把它们集中在一个allocator中！**

[https://mmbiz.qpic.cn/mmbiz_png/WwIcQHkD5mcpGpZWC00BdwOPWk1hdJFtMdx2GSDomjFqPn8LB3B7RQjgTicrupibPeCEQg9fkDdMOibTrc0bZF4Bw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1](https://mmbiz.qpic.cn/mmbiz_png/WwIcQHkD5mcpGpZWC00BdwOPWk1hdJFtMdx2GSDomjFqPn8LB3B7RQjgTicrupibPeCEQg9fkDdMOibTrc0bZF4Bw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

在前面设计中，每次都需要重载相应的函数，内部处理一些逻辑，重复代码量多，我们可以将这些包装起来，使它容易被重复使用。以下展示一个作法：每个allocator object都是个分配器，在allocator设计了allocate与deallocate两个函数。，它内部设计如下：

```cpp
class allocator
{
private:
   struct obj {
       struct obj* next;  //embedded pointer
  };
public:
   void* allocate(size_t);
   void  deallocate(void*, size_t);
   void  check();

private:
   obj* freeStore = nullptr;
   const int CHUNK = 5; //小一點方便觀察 标准库里面是20
};
```

[https://mmbiz.qpic.cn/mmbiz_png/WwIcQHkD5mcpGpZWC00BdwOPWk1hdJFtFXWFysdroiaK45MDLwJYjSp0a4icDkRzGCQV5iavTSonefyyYvoayhAUw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1](https://mmbiz.qpic.cn/mmbiz_png/WwIcQHkD5mcpGpZWC00BdwOPWk1hdJFtFXWFysdroiaK45MDLwJYjSp0a4icDkRzGCQV5iavTSonefyyYvoayhAUw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

其他类，例如：Foo和Goo，当需要allocator这种内存管理池，只需要写出下面两个函数：

```cpp
static void* operator new(size_t size)
{
   return myAlloc.allocate(size);
}
static void  operator delete(void* pdead, size_t size)
{
   return myAlloc.deallocate(pdead, size);
}
```

然后把内部做的动作交给myAlloc。myAlloc是专门为Foo或者Goo之类的服务的，可以设计为静态 ：

```cpp
static allocator myAlloc;
```

想象成里面有一根指针指向一条链表，专门为自己服务。

这里实现同前面的实现。

```cpp
void* allocator::allocate(size_t size)
{
   obj* p;

   if (!freeStore) {
       //linked list 是空的，所以攫取一大塊 memory
       size_t chunk = CHUNK * size;
       freeStore = p = (obj*)malloc(chunk);

       //cout << "empty. malloc: " << chunk << " " << p << endl;

       //將分配得來的一大塊當做 linked list 般小塊小塊串接起來
       for (int i = 0; i < (CHUNK - 1); ++i) {  //沒寫很漂亮, 不是重點無所謂.
           p->next = (obj*)((char*)p + size);
           p = p->next;
      }
       p->next = nullptr;  //last
  }
   p = freeStore;
   freeStore = freeStore->next;

   //cout << "p= " << p << " freeStore= " << freeStore << endl;

   return p;
}
```

同前面实现：

```cpp
void allocator::deallocate(void* p, size_t)
{
   //將 deleted object 收回插入 free list 前端
  ((obj*)p)->next = freeStore;
   freeStore = (obj*)p;
}
```

这样设计好之后，任何一个class要使用它，这种写法比较干净，application classes不再需内存分配纠缠不清，所有相关细节交给allocator去操心。

## **9.macro for static allocator4**

之前的几个版本都是在类的内部重载了operator new()和operator delete()函数，这些版本都将分配内存的工作放在这些函数中，但现在的这个版本将这些分配内存的操作放在了allocator类中，这就渐渐接近了标准库的方法。

从上面的代码中可以看到，两个类Foo和Goo中operator new()和operator delete()函数等很多部分代码类似，于是可以使用**宏**来将这些高度相似的代码提取出来，**简化类的内部结构**，但最后达到的结果是一样的。

```cpp
//DECLARE_POOL_ALLOC -- used in class definition
#define DECLARE_POOL_ALLOC() \\
public:\\
   void* operator new(size_t size) { \\
       return myAlloc.allocate(size); \\
   } \\
   void operator delete(void* p) { \\
       myAlloc.deallocate(p, 0); \\
   } \\
protected: \\
   static light::allocator myAlloc;

//IMPLEMENT_POOL_ALLOC -- used in class implementation
#define IMPLEMENT_POOL_ALLOC(class_name) \\
light::allocator class_name::myAlloc;
```

Foo、Goo:

```cpp
class Foo {
DECLARE_POOL_ALLOC()
public:
   long L;
   string str;
public:
   Foo(long l): L(l) {
  }
};

IMPLEMENT_POOL_ALLOC(Foo)

class Goo {
DECLARE_POOL_ALLOC()
public:
   complex<double> c;
   string str;
public:
   Goo(const complex<double> x): c(x) {
  }
};

IMPLEMENT_POOL_ALLOC(Goo)
```

## **10.global allocator**

前面设计了版本1、2、3、 4。

版本1：最简单，版本2：加上了embedding pointer，版本3：把内存的动作抽取到class中，版本4：设计一个macro。

上面我们自己定义的分配器使用了一条链表来管理内存的，但标准库却用了多条链表来管理，这在后续会详细介绍:

[https://mmbiz.qpic.cn/mmbiz_png/WwIcQHkD5mcpGpZWC00BdwOPWk1hdJFtIyeNt3qnO7vOmUMTfTs0E3sO3kCp0m6sgCKojPPpIDtBxCP3Ioia2nQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1](https://mmbiz.qpic.cn/mmbiz_png/WwIcQHkD5mcpGpZWC00BdwOPWk1hdJFtIyeNt3qnO7vOmUMTfTs0E3sO3kCp0m6sgCKojPPpIDtBxCP3Ioia2nQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

## 11.new handler

当operator new无法满足某一内存分配需求时，它会抛出std::bad_alloc exception。某些编译器则返回0，你可以另编译器那么做：`new(nothrow) Foo;`

在抛出异常之前，它会调用一个客户指定的错误处理函数，也就是所谓的new-handler。

客户通过调用set_new_handler来设置new-handler：

```cpp
namespace std {
typedef void (*new_handler)();
new_handler set_new_handler(new_handler p) throw();
}
```

set_new_handler返回之前设置的new_handler。

当operator new无法满足内存申请时，它会不断调用new-handler函数，直到找到足够内存。因此，一个设计良好的new-handler必须做以下事：

a：让更多内存可被使用，以便使operator new下一次分配内存能够成功。实现方法之一就是程序一开始就分配一大块内存，而后当new-handler第一次被调用时，将它们还给程序使用；

b：安装另一个new-handler：如果目前的new-handler无法获得更多内存，并且它直到另外哪个new-handler有此能力，则当前的new-handler可以安装那个new-handler以替换自己，下次当operator new调用new-handler时，就是调用最新的那个。

c：卸载new-handler，一旦没有设置new-handler，则operator new就会在无法分配内存时抛异常；

d：抛出bad_alloc异常；

e：不返回，直接调用abort或exit。

**c++ 设计是为了给我们一个机会，因为一旦内存不足，整个软件也不能运作，所以它借这个机会通知你，也就是通过`set_new_handler`调用我们的函数，由我们来决定怎么办。**

现在回过头看`operator new`源码：

如果malloc没有成功，handler函数会循环调用，除非我们将handler设置为空，或者在handler中抛出异常。

```cpp
operator new (std::size_t sz) _GLIBCXX_THROW (std::bad_alloc)
{
 void *p;

 /* malloc (0) is unpredictable; avoid it. */
 if (__builtin_expect (sz == 0, false))
   sz = 1;

 while ((p = malloc (sz)) == 0)
{
     new_handler handler = std::get_new_handler ();
     if (! handler) //利用NULL，跑出错误异常
     _GLIBCXX_THROW_OR_ABORT(bad_alloc());
     handler ();  // 重新设定为原来的函数
}

 return p;
}
```

例子：

```cpp
#include <new>
#include <iostream>
#include <cassert>

using namespace std;

void noMoreMemory() {
   cerr<<"out of memory";
   abort();
}

int main() {
   set_new_handler(noMoreMemory);
   int *p=new int[900000000000000];
   assert(p);
}
```

输出：

```bash
out of memory
```

## 12.=default和=delete

(=default与=delete) it is not only for constructors and assignments, but also applies to `operator new/new[]`, `operator delete/delete[]` and their overloads.

解释一下，=default和=delete不仅适用于构造函数和赋值，还适用于`operator new / new []`，`operator delete / delete []`及其重载。

C++ 的类有四类特殊成员函数，它们分别是：默认构造函数、析构函数、拷贝构造函数以及拷贝赋值运算符。这些类的特殊成员函数负责创建、初始化、销毁，或者拷贝类的对象。如果程序员没有显式地为一个类定义某个特殊成员函数，而又需要用到该特殊成员函数时，则编译器会隐式的为这个类生成一个默认的特殊成员函数。

**（1）C++11 标准引入了一个新特性："=default"函数。**

程序员只需在函数声明后加上“=default;”，就可将该函数声明为 "=default"函数，编译器将为显式声明的 "=default"函数自动生成函数体。

```cpp
class X {
 public:
  X() = default;
}
```

- "=default"函数特性仅适用于类的特殊成员函数，且该特殊成员函数没有默认参数。

```cpp
class X1
{
 public:
  int f() = default;      // err , 函数 f() 非类 X 的特殊成员函数
  X1(int, int) = default;  // err , 构造函数 X1(int, int) 非 X 的特殊成员函数
  X1(int = 1) = default;   // err , 默认构造函数 X1(int=1) 含有默认参数
};
```

- "=default"函数既可以在类体里（inline）定义，也可以在类体外（out-of-line）定义。

```cpp
class X2
{
 public:
  X2() = default; //Inline defaulted 默认构造函数
  X2(const X&);
  X2& operator = (const X&);
  ~X2() = default;  //Inline defaulted 析构函数
};

X2::X2(const X&) = default;  //Out-of-line defaulted 拷贝构造函数
X2& X2::operator= (const X2&) = default;   //Out-of-line defaulted 拷贝赋值操作符
```

（2）**为了能够让程序员显式的禁用某个函数，C++11 标准引入了一个新特性："=delete"函数。程序员只需在函数声明后上“=delete;”，就可将该函数禁用。**

```cpp
class X3
{
 public:
  X3();
  X3(const X3&) = delete;  // 声明拷贝构造函数为 deleted 函数
  X3& operator = (const X3 &) = delete; // 声明拷贝赋值操作符为 deleted 函数
};
```

- "=delete"函数特性还可用于禁用类的某些转换构造函数，从而避免不期望的类型转换

```cpp
class X4
{
 public:
  X4(double) {}
  X4(int) = delete;
};
```

- "=delete"函数特性还可以用来禁用某些用户自定义的类的 new 操作符，从而避免在自由存储区创建类的对象

```cpp
class X5
{
 public:
  void *operator new(size_t) = delete;
  void *operator new[](size_t) = delete;
};
```

回到侯老师课上，见下面两个ppt:

[https://mmbiz.qpic.cn/mmbiz_png/WwIcQHkD5mcpGpZWC00BdwOPWk1hdJFtA39jcjuLZ9VnCq8S4TfuoK0vzYmZdVYSnrUT1rJJLOS8d4Af2pcp0w/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1](https://mmbiz.qpic.cn/mmbiz_png/WwIcQHkD5mcpGpZWC00BdwOPWk1hdJFtA39jcjuLZ9VnCq8S4TfuoK0vzYmZdVYSnrUT1rJJLOS8d4Af2pcp0w/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

[https://mmbiz.qpic.cn/mmbiz_png/WwIcQHkD5mcpGpZWC00BdwOPWk1hdJFtwfN0kSeqbtDfJxoTiasHSkOK6Kj9Iruc2mVDPUKcsOkzHt54fNVdtww/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1](https://mmbiz.qpic.cn/mmbiz_png/WwIcQHkD5mcpGpZWC00BdwOPWk1hdJFtwfN0kSeqbtDfJxoTiasHSkOK6Kj9Iruc2mVDPUKcsOkzHt54fNVdtww/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

首先使用了=default对`operator new`与`operator delete`，由于=defalult不能使用在这些函数上面，在侯老师代码中，将这两行注释掉了，保留了=delete的代码，所以在右侧输出，使用new没问题，使用new[]被禁用，自然报错，第二个是`operator new`与`operator delete`被禁用，因此new被禁用，报错，new[]正常。

参考资料：[https://www.cnblogs.com/lsgxeva/p/7787438.html](https://www.cnblogs.com/lsgxeva/p/7787438.html)