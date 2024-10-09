# 

宋宝华 Jeff Labs

_2021年12月08日 20:33_

本文目录

不知道自己不知道！

指针何时指针？

指针何时是整数？

物理地址是指针？

模糊地带

绝世好代码？

昨天我犯了一个错误把指针和整数“混淆”的错误，幸得队友王童鞋指正，今早起床，我把这个心得花一点时间记录下来。

大抵掌握一个技术或者知识都是这三个阶段：

- 不知道自己不知道；

- 知道自己不知道；

- 知道自己知道。

比较难突破的是“不知道自己不知道”的阶段，因为“不知道自己不知道”，所以才往往特别自信，觉得“老子天下第一”。基本上，本文要记录的一个小点，也是一个我从“不知道自己不知道”到“知道自己知道”的过程。

我们都知道（？？？），指针和整数在C语言里面是两种不同含义的：

- 指针：主要是为了方便引用（Dereferencing）一个内存地址, Dereferencing is used to access or manipulate data contained in memory location pointed to by a pointer。所以指针的目的其实就是为了这样的读写操作：

```
*p = a;
```

- 整数：整数是一个数值，它的主要目的是为了加减等计算、比对、做数组下标、做索引之类的。它的目的不是为了引用一个内存。指针和整数（这里主要是unsigned long，因为unsigned long的位数一般等于CPU可寻址的内存地址位数）本身是八竿子打不着的，但是它们之间的一个有趣联系是：

  **如果我们只是关心这个地址的值，而不是关心通过这个地址去访问内存，这个时候，内核经常喜欢用unsigned long代替指针。**

我们下面来看2个不同的场景：

**指针是指针？**

```
copy_from_user(void *to, 
```

在这2个函数里面，void \_\_user \*from，void \_\_user \*to都清楚地表明用户空间的虚拟地址是一个指针。这2个函数这样做的原因是非常清晰的，我就是要去Dereference用户空间的地址，进行内存拷贝的，所以它的目的是为了通过指针来访问内存。

类似的例子比如file_operations里面read、write什么的：

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**指针是整数？**

```
/**             
```

注释清楚地写明，start是用户态的起始地址：

**@start:      starting user address**

所以，本质上，和copy_from_user()里面的void \_\_user \*from一样的，但是这里它用的是unsigned long start ！！！不是void \_\_user * start ！！！

原因非常清楚，get_user_pages()只关心start这个数值本身，它用于去运算、查找、比对。它不是要：

```
*start = 100;
```

类似的例子还有：

```
long pin_user_pages(
```

更加不要提著名的find_vma()：

```
/* Look up the first VMA which satisfies  addr < vm_end,  
```

它根据addr用户态地址，在进程的mm里面去找到addr位于的VMA。显然，这个时候，它的目的是为了完成addr与进程每个VMA起始和结束地址的比多。

这个时候，我们来看看VMA结构体的长相，就更加有意思了。我们都知道，VMA是为了记录进程每一段虚拟地址空间的（比如代码段、数据段、堆、栈、mmap等）：

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

然后我们看看VMA的定义：

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

看到没有，vm_start和vm_end都是妥妥的unsigned long啊！！！

我于是试图弄清楚这么做的**科学依据**是什么，发现LDD3里面赫然写着这么一段话（LDD3第11章289页）：

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

它的科学依据是，既然你不是为了dereferencing，我就让你dereferencing不了，免得你又跑去dereferencing，从而导致bug。有的人说，我强制转化unsigned long为指针，不就可以访问了吗？

你不是还是需要强制转换不是？你强制转换之前，会想一下，\*\*这个地方指针为啥是个整数呢？你想明白了，说不定就不去访问了。\*\*这样它实际达到了震慑心灵的效果。

到这里，我们谈的都还是虚拟地址，那么下面我们来谈下物理地址。

**物理地址是指针？**

\*\*在一个有MMU的系统中，物理地址从来都不是指针。\*\*物理地址，从骨子里就是一个整数！！！我记得之前经常有人往内核发patch，把物理地址用个指针

\*p来描述，这是错到根子里面的事情，所以每次都被骂地狗血淋头。

因为你根本不可能用物理地址去Dereferencing 什么东西。物理地址在内核的描述是：

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

它要么是一个32位的整数，要么是一个64位的整数。

那么，物理地址什么时候是一个指针呢？在我还是一个小屁孩在大学玩《仙剑奇侠传》和《轩辕剑：天之痕》的时候，我那个时候玩单片机，单片机里面没有MMU，所以也没虚拟地址的概念，都是妥妥地通过物理地址“指针”来访问内存的。

所以，如果一个人，一辈子都是玩单片机，它肯定会觉得我这篇文章在胡扯，因为他还是一个“不知道自己不知道”的阶段。

**模糊地带**

这里面仍然有一些模糊地带，比如\_\_get_free_page()、\_\_get_free_pages()这样的API，返回的也是unsigned long而不是指针：

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

往死里作也要把它弄成unsigned long!!!

实际上，内核要有时候需要访问\_\_get_free_page()返回的内存，此前它需要进行强制类型转换：

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

这看起来是不是特别地“精分”？折腾来折腾去，折腾什么鬼呢？这一篇文章有解释：An (unsigned) long story about page allocation

https://lwn.net/Articles/669015/

统计表明，90%以上的情况下，\_\_get_free_page()返回的unsigned long都会被强制转化为指针！！！但是这个返回unsigned long是在历史的第一天，Linux的0.01就这样了。Al Viro <viro@ZenIV.linux.org.uk>童鞋的这个patchset企图去改：

https://lwn.net/Articles/668852/

但是改动实在太多了，改了接近600个文件，对此Linus的态度是：

"No way in hell do we suddenly change the semantics of an interface that has been around from basically day #1."

所以Linus的建议是，你真的需要一个指针的时候，你还是去调用kmalloc()吧：

```
*kmalloc(size_t size, gfp_t flags);
```

**绝世好代码**

很多工程师喜欢较真，就是必须在0和1之间做一个选择。这个选择有时候真的很难，所以Linus的意思是，0和1踏马地都不要，我不去跟你争这个0和1的问题，我给你第三条路。这里，我看出了Linus的大智若愚啊！

我个人在工程里面对无意义的0和1的争论也也没什么好感，感觉在浪费我的时间。我对事情的看法是，争一个0.7或者0.3就OK了。0.7就是真方向，0.3就是假方向。

“绝世好代码”是不存在的，“水至清则无鱼”，往死里争反而陷入了钻牛角尖。等你写出绝对完美代码的时候，黄花菜早就歇了。别忘了，还有最重要的一招，“天下武功，无坚不破，唯快不破。”

阅读 1322

​
