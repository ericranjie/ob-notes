# 

r0ysue 看雪学苑

 _2021年11月21日 18:05_

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/kR3MmOkZ9r7dibrdGMbvb6FdqicOAriagz3L2E60m2u0qRRxzhkzjZMAHAOZy27LmZhllOWxcK4us4GGlMg4wCibYA/640?wx_fmt=jpeg&wxfrom=13&tp=wxpic)  

看雪论坛作者ID：r0ysue

  

  

1

  

**简介**

  

接上篇Linker源码详解(一)，本文继续来分析Linker的链接过程。为了更好的理解Unidbg的原理，我们需要了解很多细节。虽然一个模拟二进制执行框架的弊端很多，但也是未来二进制分析的一个很好的思路。

上篇文章我们讲解了Linker的装载，将So文件按PT_LOAD段的指示来将So加载到内存，那么我们这篇文章就来分析一下加载完之后又干了什么呢？

##   

  

2

  

**So的链接**  

  

_http://androidxref.com/4.4.4_r1/xref/bionic/linker/linker.cpp#702_

```
static soinfo* load_library(const char* name) {
```

  

上篇我们进入了elf_reader.Load()函数，阅读了Linker的装载源码，当装载结束后，对soinfo结构体进行赋值(So文件的头信息/装载的结果)，并插入到链表，接着我们回到上层函数继续看。

  

_http://androidxref.com/4.4.4_r1/xref/bionic/linker/linker.cpp#751_

```
static soinfo* find_library_internal(const char* name) {
```

  

我们从上面这个函数中看到，当调用了load_library函数之后，又调用了soinfo_link_image这个函数。这个函数也就是我们今天分析的一个主要入口--链接。

下面的这个函数很长，我给大家把不相关的代码去掉，大家先通过注释来看一遍这个函数在干什么。

  

_http://androidxref.com/4.4.4_r1/xref/bionic/linker/linker.cpp#1303_

```
static bool soinfo_link_image(soinfo* si) {
```

  

上面的函数虽然很长，但是它想表达的意思很简单，我们再来回顾下它干了什么事情：

- 解析Dynamic段信息
    
- 处理依赖
    
- 准备进行重定位
    

##   

  

3

  

**So重定位**  

  

下面我们就来分析它的soinfo_relocate函数，我们看到它调用了两次，只不过入参不同，分别是我们的重定位表和PLT重定位表。

  

_http://androidxref.com/4.4.4_r1/xref/bionic/linker/linker.cpp#848_

```
static int soinfo_relocate(soinfo* si, Elf32_Rel* rel, unsigned count,
```

  

上面这个函数就是在处理重定位相关的信息了，我们看到从Dynamic段中拿到的跟重定位相关的表，会经过这个函数来处理，将So本身的地址引用进行重定位，使其可以正常运行。其实在32位So中，需要处理的重定位类型并不是很多，就4种类型需要处理，而且还有两种处理方式相同。

现在So就重定位完成了，现在So已经就可以跑起来了，下面我们就来看看从Dynamic段中拿到的各种初始化函数是怎么处理的，还记得吧。

我们回到do_dlopen函数。

  

_http://androidxref.com/4.4.4_r1/xref/bionic/linker/linker.cpp#823_

```
soinfo* do_dlopen(const char* name, int flags) {
```

  

此时我们的find_library函数已经处理完了，So已经被装载且链接过了，最后一步它调用了soinfo的CallConstructors函数，我们来看看这个函数处理了什么。

  
_http://androidxref.com/4.4.4_r1/xref/bionic/linker/linker.cpp#1192_

```
void soinfo::CallConstructors() {
```

  

_http://androidxref.com/4.4.4_r1/xref/bionic/linker/linker.cpp#1172_

```
void soinfo::CallFunction(const char* function_name UNUSED, linker_function_t function) {
```

  

至此，Linker就分析结束了。

##   

##   

4

  

**总结**  

  

我们在最后说一个Unidbg细节的bug，但是现在已经被修复了，就是作为一个扩展吧。我们来看下面一段Unidbg加载So的代码。

```
if (elfFile.file_type == ElfFile.FT_DYN) { // not executable
```

  

如果我们细心的阅读Linker的源码，就会发现Unidbg这里处理的是不恰当的。在本文的最后，我们看到了初始化函数的调用，是DT_INIT函数先被执行，后面再处理DT_INIT_ARRAY，而Unidbg这里就是将他们都添加到一个List，再一起调用。

  

这样就会产生一个问题，在某些加壳的So中，它的DT_INIT_ARRAY是在DT_INIT函数执行之后，才会有值的(进行修复)，所以按照Unidbg这个写法就无法执行INIT_ARRAY或部分INIT_ARRAY无法执行。处理方法也很简单，注释在上面了，只需要让DT_INIT先执行就可以了。

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E) 

  

**看雪ID：r0ysue**

https://bbs.pediy.com/user-home-799845.htm

*本文由看雪论坛 r0ysue 原创，转载请注明来自看雪社区

  

  

[![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458404582&idx=1&sn=3e6e984a236b1e26baaafbf350fd1cb7&chksm=b18f766c86f8ff7a4b8c367aa1e0b04aceac47b283c0fb3d38a448521adb6974847b9fa597a6&scene=21#wechat_redirect)

  

[![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)](https://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458387399&idx=2&sn=38495add2a3a3677b2c436581c07e432&scene=21#wechat_redirect)  

  

**#** **往期推荐**

1.[记一次头铁的病毒样本分析过程](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458405156&idx=1&sn=c2061cbc0d160c86568497d663d40dbd&chksm=b18f79ae86f8f0b8c0072240342b03b737b21deb89fd1f37a26a7f58f5e407437fc92633997b&scene=21#wechat_redirect)  

2.[通过CmRegisterCallback学习注册表监控与反注册表监控](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458405024&idx=1&sn=a8617e12ff4d8e39d4fe89dae2cdf577&chksm=b18f782a86f8f13c05b91d743d6a09035c01d8a24b8d1c012cc68ccb2ff72d8bb45151b4a7fb&scene=21#wechat_redirect)

3.[全网最详细CVE-2014-0502 Adobe Flash Player双重释放漏洞分析](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458402566&idx=1&sn=6a8289758711c348b36f8526808747c7&chksm=b18f0f8c86f8869a416d9af04a202105779673e3997f5381b33fe58cb767314a4789e4623230&scene=21#wechat_redirect)

4.[基于linker实现so加壳补充-------从dex中加载so](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458402519&idx=1&sn=a42e81b669d38f9ff8ed1a8d941b3f2f&chksm=b18f0e5d86f8874b091e0c767b6d3e07c79a993eff4cb1554cf16d63308fc2f84511ac80f613&scene=21#wechat_redirect)

5.[一题house_of_storm的简单利用](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458402444&idx=1&sn=9baeafab94b844b2bb4a7c91f6609edc&chksm=b18f0e0686f88710326cc868dfbaddded2c8f13d48a99fdbced9f688b0b671c2782ce3d3921a&scene=21#wechat_redirect)

6.[BCTF2018-houseofatum-Writeup题解](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458402442&idx=1&sn=65f4c412b3495b27e168d94e8adf230f&chksm=b18f0e0086f887169bf7566dafa1b5cce2800a245159721b976d49da3d44d684357f526389a0&scene=21#wechat_redirect)

  

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

公众号ID：ikanxue  

官方微博：看雪安全

商务合作：wsc@kanxue.com

  

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**球分享**

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**球点赞**

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**球在看**

  

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

点击“阅读原文”，了解更多！

阅读原文

阅读 3842

​

写留言

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/1UG7KPNHN8EGLfh77kFmnicd9WOic2ibvhCibFdB4bL4srJCgo2wnvdoXLxpIvAkfCmmcptXZB0qKWMoIP8iaibYN2FA/300?wx_fmt=png&wxfrom=18)

看雪学苑

415

写留言

写留言

**留言**

暂无留言