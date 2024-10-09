# 

1900 看雪学苑

_2021年11月13日 17:59_

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

本文为看雪论坛优秀文章\
看雪论坛作者ID：1900

1

**前言**

关于如何使用PsSetCreateProcessNotifyRoutineEx来实现进程监控，请看这篇文章：

通过PsSetCreateProcessNotifyRoutineEx和PsSetCreateThreadNotifyRoutine实现进程与线程监控\_（https://bbs.pediy.com/thread-269312.htm）\_

这篇主要是讲解通过逆向PsSetCreateProcessNotifyRoutineEx来查看这个API如何实现的进程监控并以此来实现反进程监控。

这次的实验是在Win7 x86系统上进行，所以如果是其他版本的Windows系统，里面的具体实现会有不同。

2

**逆向分析PsSetCreateProcessNotifyRoutine**

首先看下PsSetCreateProcessNotifyRoutine的反汇编结果。

```
PAGE:005947DE ; NTSTATUS __stdcall PsSetCreateProcessNotifyRoutine(PCREATE_PROCESS_NOTIFY_ROUTINE NotifyRoutine, BOOLEAN Remove)
```

可以看到PsSetCreateProcessNotifyRoutine就干了一件事，就是调用PspSetCreateProcessNotifyRoutine。在对它的调用时候，压入的两个参数除了NotifyRoutine和Remove以外，还压入了一个0。事实上PsSetCreateProcessNotifyRoutine和PsSetCreateProcessNotifyRoutineEx的唯一区别就是，在压入第三个参数的时候，PsSetCreateProcessNotifyRoutine压入的是0，而PsSetCreateProcessNotifyRoutineEx压入的是1。

接着跟进PspSetCreateProcessNotifyRoutine。

```
PspSetCreateProcessNotifyRoutine proc near
```

函数首先判断压入的Remove参数是否为0，是的话则跳转。由于在增加回调的时候，Remove参数会使用FALSE，所以这个时候这个跳转会成立。接着跟进loc_594905看看。

```
PAGE:00594905 loc_594905:                             ; CODE XREF: PspSetCreateProcessNotifyRoutine+C↑j
```

在loc_594905中，函数会判断传进的第三个参数是否为0。上面提到过，这个参数其实是区别你调用的是PsSetCreateProcessNotifyRoutine还是PsSetCreateProcessNotifyRoutineEx。如果是后者，这个参数就是1，前者这个参数是0。而两者的区别只是后者会调用sub_56F869。但由于这个函数的作用对今天的主题而言没什么意义，所以这里不分析，直接接着看loc_59491E的内容。

```
PAGE:0059491E loc_59491E:                             ; CODE XREF: PspSetCreateProcessNotifyRoutine+110↑j
```

函数将要设置的回调函数的地址作为第一个参数，根据调用是否带有Ex的PsSetCreateProcessNotifyRoutine来传入第二个参数。

这里主要是调用AllocateAssign这个函数，当然这个函数名是我重命名以后的函数名。因为这个函数就是为保存回调函数地址申请一块空间，如果申请到了那么会跳转继续执行，如果没申请到就会设置返回值为没有足够的资源并且跳到函数结束执行，接下来看看AllocateAssign的反汇编结果。

```
PAGE:005946D7 AllocateAssign  proc near               ; CODE XREF: PsSetLoadImageNotifyRoutine+E↑p
```

可以看到这个函数中，首先是申请0xC大小的内存空间，然后将低4为清0，中间4位赋值为要设置的回调函数的地址，最高4位根据传入的参数来设置。

由此我们可以知道，在PsSetCreateProcessNotifyRoutine函数中，系统会分配0xC大小的内存，而内存的中间4位保存的就是要设置的回调函数的地址。根据调用的是PsSetCreateProcessNotifyRoutine还是PsSetCreateProcessNotifyRoutineEx，系统会设置最高的4位的值，如果是前者会设置位0，后者设置位1。

接着看内存看内存分配并且赋值成功以后，会执行的代码，也就是loc_59493C的代码。

```
PAGE:0059493C loc_59493C:                             ; CODE XREF: PspSetCreateProcessNotifyRoutine+13A↑j
```

可以看到，函数首先将一个数组地址赋给esi，然后将0压入栈中并且调用SetArray这个函数，在调用这个函数之前还把esi的地址，也就是这个数组的地址赋给了eax，把ebx的值赋给ecx，这个ebx的值就是在上面调用完AllocateAssign函数以后得到的分配和赋值的内存地址。因为在调用AllocateAssign函数后的返回值eax赋值给了ebx。

调用完SetArray函数以后，会判断返回值是否为0，不为0那就是调用成功，也就是回调函数设置成功，就会跳转到函数调用成功的退出代码执行。如果为0说明并没有设置完成，随后会对edi和esi进行加4的操作并判断edi是否小于0x100，如果小于就跳上去继续执行SetArray函数。由于此时esi指向的是ProcessFuncArray数组的地址，所以加4就是指向数组中下一个元素，edi就是为了避免数组溢出而设置，根据值的大小可以猜测这个数组一共0x40个大小。

分析到这里就可以得出结论，PsSetCreateProcessNotifyRoutine函数设置进程监控的回调函数的办法就是首先申请一块内存来保存要设置的回调函数的地址，然后将这个申请到的内存的地址赋到ProcessFuncArray数组中的某个元素。

那么它是怎么判断要赋值到哪个元素呢，就要继续跟进SetArray函数。

```
PAGE:00594706 SetArray        proc near               ; CODE XREF: IoRegisterPriorityCallback+5B↑p
```

函数将数组中的对应的内容赋给eax以后跳转到loc_594759执行，接着看loc_594749处的代码。

```
PAGE:00594749 loc_594749:                             ; CODE XREF: SetArray+24↑j
```

在这里程序将得到的数组中对应的元素内容和传入的参数进行异或，然后判断是否小于7，如果小于7就跳转到对数组进行赋值的代码进行执行。而0x7对应的二进制是高29位是0，低3位是1，在跟进得到的数组内容要和0进行异或可以知道，这里是判断得到的数组中的内容高29位是不是0。如果高29位中有1个是1，那么异或以后得到的值就会大于7。

接着看loc_59472C中的代码，也就是对数组进行赋值的代码。

```
PAGE:0059472C loc_59472C:                             ; CODE XREF: SetArray+49↓j
```

这里的ecx的值就是前面用来保存回调函数地址所调用的AllocateAssign函数得到的分配的内存地址。因为上面申请到内存以后，将内存地址先赋给eax在赋给ecx，随后的代码中都没对ecx进行改变。

将这个地址赋给eax并且与7异或，那就是将低3位置1，接着就跳转到loc_594739处执行。执行的时候，这个esi就是要赋值的数组的对应元素的地址，因为在进入SetArray函数前，将数组的对应元素的地址赋给了esi后就没在改变了。而根据cmpxchg的功能可以知道，这里就是将AllocateAssign分配到的内存地址和7或运算以后的地址复制到数组中，随后跳转到loc_594751执行。那么继续看loc_594751执行的代码。

```
PAGE:00594751 loc_594751:                             ; CODE XREF: SetArray+3F↑j
```

在这里，程序首先将原来数组中保存的内容的低三位清0，接着判断得到的数据是否为0。也就是判断原来数组中的内容的高29位是不是0，如果是0说明上面的赋值成功了，程序就会跳转到函数执行成功的地方，也就是loc_5947C0处继续执行，在loc_5947C0，程序会将eax赋值为1，也就是说返回值是1。如果高29位不是0，那上面的赋值就会失败，程序就会跳转到loc_5947C4的地址继续执行，在这个地址中，程序会将eax赋值为0，也就是说返回值是0。

综上，可以得出以下的结论：

- 程序会申请一块0xC大小的内存，内存中低4为是0，中间4为是要设置的回调函数的地址，高4为根据是否是Ex函数来设置为是0还是1。

- 从整型数组ProcessArray中按顺序依次查看对应的数组的元素的位置是不是可以用来保存分配的内存地址。

- 而这个位置能不能存储这个地址，取决于这个位置中保存的内容的高29位是不是0，如果是0就用来保存申请到的地址。

# 

# 

3

**反进程监控的实现**

根据上面的分析，可以得出结论，设置这些回调函数的时候，操作系统首先会申请一个0xC大小的内存，然后将内存地址的中间4位设置为回调函数的地址。随后就会在ArrayProcess数组中存放这个分配的内存地址，当要注意这个内存地址的低3位在复制到数组之前会被置1。

那也就是说当有进程的状态发生变化的时候，操作系统就会去ProcessArray数组中遍历，如果发现对应的元素的高29位不是0就把内容取出并且将低3位置0(因为在复制到数组之中去的时候，分配的内存的低3位会被置为1)。置0以后得到的地址在+4的地方保存的就是回调函数的地址，系统就会根据这个地址在调用相应的回调函数。

所以要进行反进程监控，只需要找到这个数组，并且将这个数组中的回调函数的地址取出然后通过PsSetCreateProcessNotifyRoutine函数删除这个回调函数就可以了。

要找到这个数组首先需要找到PspSetCreateProcessNotifyRoutine函数地址，因为在这个函数中才有对这个数组进行使用，才可以找到数组地址。但因为这个函数并没有导出，所以需要通过PsSetCreateProcessNotifyRoutine函数来找到它，因为这个函数是导出函数，可以直接获取到地址。

在PsSetCreateProcessNotifyRoutine中对PspSetCreateProcessNotifyRoutine的调用如下：

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

可以看到是调用地址处偏移为9的地方就是PspSetCreateProcessNotifyRoutine函数的地址，所以可以将0xE8作为特征码来查找函数地址。

得到PspSetCreateProcessNotifyRoutine函数地址以后，可以在对esi赋值为函数地址的地方找到ProcessFuncArray数组的地址如下：

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

所以得到PspSetCreateProcessNotifyRoutine函数地址以后，可以根据0xEB，0x2F和0xBE这三个特征码来定位进程数组的地址(当然这样有点太少，可以弄的多一点，这里就偷了个懒)。

根据以上内容就可以写出获取ProcessFuncArray地址的代码如下：

```
PUINT32 SearchProcessArray()
```

获取到函数地址以后，只需要遍历数组并查看相应的高29位是不是0，如果不是0说明里面保存了地址，取出这个地址将低3位置0以后在加4就得到了保存回调函数地址的地址，将这个回调函数地址取出就可以调用PsSetCreateProcessNotifyRoutine函数来进行回调函数的删除，代码如下：

```
NTSTATUS status = STATUS_SUCCESS;
```

4

**运行结果**

首先是开启进程监控以后，可以看到所有进程的创建都被监控到了，而且demo.exe进程的创建也被拦截了。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

当运行程序卸载掉回调函数以后，在创建进程就没有被监控到，并且demo.exe进程也可以成功创建。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**看雪ID：1900**

https://bbs.pediy.com/user-home-835440.htm

\*本文由看雪论坛 1900 原创，转载请注明来自看雪社区

[!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458402059&idx=1&sn=b16e5d69f4bbb055f1930ae242a5e58a&chksm=b18f0d8186f88497d8460db4deaf29aa4c1b62b3c61edd094c7871291ed36f489c370e8113a7&scene=21#wechat_redirect)

[!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)](https://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458387399&idx=2&sn=38495add2a3a3677b2c436581c07e432&scene=21#wechat_redirect)

**#** **往期推荐**

1.[分享一个基本不可能被检测到的hook方案](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458404571&idx=1&sn=ecef82fb7fd174d158e2672ebc0ce82d&chksm=b18f765186f8ff47f4c2929f430102ef1c13f5313f57c8b791701523c11da730aed7422bced5&scene=21#wechat_redirect)

2.[由2021ByteCTF引出的intent重定向浅析](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458403066&idx=2&sn=b7b8621add4f7130c5bc7768210ee855&chksm=b18f707086f8f9668ed04abaaad407637bb37d2c1bacd0294411d7607770220ecbbe2ffeae8d&scene=21#wechat_redirect)

3.[全网最详细CVE-2014-0502 Adobe Flash Player双重释放漏洞分析](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458402566&idx=1&sn=6a8289758711c348b36f8526808747c7&chksm=b18f0f8c86f8869a416d9af04a202105779673e3997f5381b33fe58cb767314a4789e4623230&scene=21#wechat_redirect)

4.[基于linker实现so加壳补充-------从dex中加载so](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458402519&idx=1&sn=a42e81b669d38f9ff8ed1a8d941b3f2f&chksm=b18f0e5d86f8874b091e0c767b6d3e07c79a993eff4cb1554cf16d63308fc2f84511ac80f613&scene=21#wechat_redirect)

5.[一题house_of_storm的简单利用](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458402444&idx=1&sn=9baeafab94b844b2bb4a7c91f6609edc&chksm=b18f0e0686f88710326cc868dfbaddded2c8f13d48a99fdbced9f688b0b671c2782ce3d3921a&scene=21#wechat_redirect)

6.[BCTF2018-houseofatum-Writeup题解](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458402442&idx=1&sn=65f4c412b3495b27e168d94e8adf230f&chksm=b18f0e0086f887169bf7566dafa1b5cce2800a245159721b976d49da3d44d684357f526389a0&scene=21#wechat_redirect)

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

公众号ID：ikanxue

官方微博：看雪安全

商务合作：wsc@kanxue.com

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**球分享**

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**球点赞**

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**球在看**

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

点击“阅读原文”，了解更多！

阅读原文

阅读 1545

​

写留言

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/1UG7KPNHN8EGLfh77kFmnicd9WOic2ibvhCibFdB4bL4srJCgo2wnvdoXLxpIvAkfCmmcptXZB0qKWMoIP8iaibYN2FA/300?wx_fmt=png&wxfrom=18)

看雪学苑

8分享2

写留言

写留言

**留言**

暂无留言
