# 

码农桃花源

_2021年09月01日 09:00_

编者荐语：

奇伢的文章很有特色，文字浅显易懂，图画得漂亮，总结地很到位。尤其是你在知晓了很多细节后，再看他的总结文章总是感觉很爽。

以下文章来源于奇伢云存储 ，作者奇伢

\[

![](http://wx.qlogo.cn/mmhead/Q3auHgzwzM4hc7tFZP4icTrg4hc2Ic8ibCyiay1WymOVzcVbVWibLVSblw/0)

**奇伢云存储**.

云存储深耕之路，专注于对象存储，块存储，云计算领域。坚持撰写有思考的技术文章。

\](https://mp.weixin.qq.com/s?\_\_biz=MjM5MDUwNTQwMQ==&mid=2257487403&idx=1&sn=735c9190f691e71254603ee3968909b7&chksm=a539e6fd924e6feb35a9a6652c5413357c8a1e30cc815fa5ceccd7981f08cd2b33fff2ca1d20&mpshare=1&scene=24&srcid=090159tIKKrenurG3Gmrzto9&sharer_sharetime=1630507070340&sharer_shareid=5fb9813bfe9ffc983435bfc8d8c5e9ca&key=daf9bdc5abc4e8d09f1c44a1b7d4ebefa50ec8e4c6f29d97659d962f4067d8d963bddde87009d5ea9ddb3f3f25d4b20f13e6aa0188b1ca10d6788a31dafa89442d682bf17b9104bafd8e8e236c3b2c399a997ddc03e0e96e4973a9810442cde9cda7020925c8f24e75412d16b1c741b31a47f8a998cba30ef11bab9d60155a69&ascene=0&uin=MTEwNTU1MjgwMw%3D%3D&devicetype=Windows+11+x64&version=63090b19&lang=zh_CN&countrycode=CN&exportkey=n_ChQIAhIQ5yNI5TIZ667TfD8iG2Et4BLmAQIE97dBBAEAAAAAAMFVGQn4xNkAAAAOpnltbLcz9gKNyK89dVj0gqPWhNcG4NDncJFYgDm1FyPDcUt%2BVvtWDopzXMsk1L6zKunCQEZf4E3I3WJ8HxNzD7npAhU7gi1PyIy4iysmDcvkWpugTpTAeBj6cr1LTXWJDI3ovIB%2BMgE%2F4Qhbdx1I%2FWqt74dtwO6oRUnxTO22prvhPZQNz6Rz7iBW1Km8JoOfzT8yTcEcnUEUk4X2FqMVo36ia3djIouzbwWTozYMOuPjH1%2B%2B1pGyQjQwdhtA%2FpfSLYuwenKzLGtKEUaAzk5c&acctmode=0&pass_ticket=nQz2vkaGBTH5VXZP3ORMSLJlExvD1ElXNf%2BfVxIcH9XV17Upfm6ZNAH5u8sA%2BFXT&wx_header=1&fasttmpl_type=0&fasttmpl_fullversion=7351805-zh_CN-zip&fasttmpl_flag=1#)

坚持思考，就会很酷

![图片](https://mmbiz.qpic.cn/mmbiz_png/4UtmXsuLoNdtbJoXjWrOTJUKsljrickDho4nf1XZLSPTmmDZc01s8crUexw4cgeXAMkITvzZ0lPVX3fic303cMpA/640?wx_fmt=png&wxfrom=13&tp=wxpic)

![图片](https://mmbiz.qpic.cn/mmbiz_png/E7PuB8occXpoqnvYsd3GsYicWxmuCvI2p5mSKsxCognzURYY27N4Z1AUlCXytekruoDOia3GM5w8iadrELIRk4OKg/640?wx_fmt=png&wxfrom=13&tp=wxpic)

为什么 panic 值得思考？

![图片](https://mmbiz.qpic.cn/mmbiz_png/fMeqxiaIMiaH6OPjyIRmVSYPsDpIcBqcaRCGpZFXHqCVfldMeUMM0r8e5lEmTjvfNrbw7UtCFoXQ18E5XYeADxhw/640?wx_fmt=png&wxfrom=13&tp=wxpic)

初学 Go 的时候，心里常常很多疑问，有时候看似懂了的问题，其实是是而非。

panic 究竟是啥？看似显而易见的问题，但是却回答不出个所以然来。奇伢分两个章节来彻底搞懂 panic 的知识：

- 姿势篇：摸清楚 panic 的诞生，它不是石头里蹦出来的，总结有三种姿势；

- 原理篇：彻底搞明白 panic 的内部原理，理解 panic 的深层原理；

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

panic 的三种姿势

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

什么时候会产生 panic ？

我们先从“形”来学习。从程序猿的角度来看，可以分为主动和被动方式，被动的方式有两种，如下：

**主动方式**：

- 程序猿主动调用 `panic( )` 函数；

**被动的方式**：

- 编译器的隐藏代码触发；

- 内核发送给进程信号触发 ；

## 

编译器的隐藏代码

Go 之所以简单又强大，编译器居功至伟。非常多的事情是编译器帮程序猿做了的，逻辑补充，内存的逃逸分析等等。

**包括 panic 的抛出！**

举个非常典型的例子：整数算法**除零**会发生 panic，怎么做到的？

看一段极简代码：

`func divzero(a, b int) int {       c := a/b       return c   }   `

上面函数就会有除零的风险，当 b 等于 0 的时候，程序就会触发  panic，然后退出，如下：

`root@ubuntu:~/code/gopher/src/panic# ./test_zero       panic: runtime error: integer divide by zero      goroutine 1 [running]:   main.zero(0x64, 0x0, 0x0)    /root/code/gopher/src/panic/test_zero.go:6 +0x52   `

**问题来了：程序怎么触发的 panic ？**

**代码面前无秘密。**

可代码看不出啥呀，不就是一行 `c := a/b` 嘛？

奇伢说的是**汇编代码**。**因为这段隐藏起来的逻辑，是编译器帮你加的。**

用 dlv 调试断点到 `divzero` 函数，然后执行 `disassemble` ，你就能看到秘密了。奇伢截取部分汇编，并备注了下：

`(dlv) disassemble   TEXT main.zero(SB) /root/code/gopher/src/panic/test_zero.go          // 判断 b 是否等于 0        test_zero.go:6  0x4aa3c1    4885c9          test rcx, rcx       // 不等于 0 就跳转到 0x4aa3c8 执行指令，否则就往下执行       test_zero.go:6  0x4aa3c4    7502            jnz 0x4aa3c8       // 执行到这里，就说明 b 是 0 值，就跳转到 0x4aa3ed ，也就是 call $runtime.panicdivide   =>  test_zero.go:6  0x4aa3c6    eb25            jmp 0x4aa3ed       test_zero.go:6  0x4aa3c8    4883f9ff        cmp rcx, -0x1       test_zero.go:6  0x4aa3cc    7407            jz 0x4aa3d5       test_zero.go:6  0x4aa3ce    4899            cqo       test_zero.go:6  0x4aa3d0    48f7f9          idiv rcx       // ...       test_zero.go:7  0x4aa3ec    c3              ret       // 看到神奇的函数了嘛 ！       test_zero.go:6  0x4aa3ed    e8ee27f8ff      call $runtime.panicdivide`

看到秘密的函数了吗？

**编译器偷偷加上了**一段 `if/else` 的判断逻辑，并且还给加了 `runtime.panicdivide`  的代码。

1. 如果 b == 0 ，那么跳转执行函数 `runtime.panicdivide` ；

再来看一眼 `panicdivide` 函数，这是一段极简的封装：

`// runtime/panic.go   func panicdivide() {       panicCheck2("integer divide by zero")       panic(divideError)   }   `

看到了不，这里面调用的就是 `panic()` 函数。

除零触发的 panic 就是这样来的，它不是石头里蹦出来的，而是编译器多加的逻辑判断保证了除数为 0 的时候，触发 panic 函数。

**划重点：编译器加的隐藏逻辑，调用了抛出 panic 的函数。Go 的编译器才是真大佬！**

## 

进程信号触发

最典型的是**非法地址访问**，比如， nil 指针 访问会触发 panic，怎么做到的？

看一个极简的例子：

`func nilptr(b *int) int {       c := *b       return c   }   `

当调用 `nilptr( nil )` 的时候，将会导致进程异常退出：

`root@ubuntu:~/code/gopher/src/panic# ./test_nil       panic: runtime error: invalid memory address or nil pointer dereference   [signal SIGSEGV: segmentation violation code=0x1 addr=0x0 pc=0x4aa3bc]      goroutine 1 [running]:   main.nilptr(0x0, 0x0)    /root/code/gopher/src/panic/test_nil.go:6 +0x1c   `

**问题来了：这里的 panic 又是怎么形成的呢？**

在 Go 进程启动的时候会注册默认的信号处理程序（ `sigtramp` ）

在 cpu 访问到 0 地址会触发 page fault 异常，这是一个非法地址，内核会发送 `SIGSEGV` 信号给进程，所以当收到 `SIGSEGV` 信号的时候，就会让 `sigtramp` 函数来处理，最终调用到 `panic` 函数 ：

`// 信号处理函数回调   sigtramp （纯汇编代码）     -> sigtrampgo （ signal_unix.go ）       -> sighandler  （ signal_sighandler.go ）          -> preparePanic （ signal_amd64x.go ）                -> sigpanic （ signal_unix.go ）               -> panicmem                  -> panic (内存段错误)   `

在 `sigpanic` 函数中会调用到 `panicmem`  ，在这个里面就会调用 panic 函数，从而走上了 Go 自己的 panic 之路。

`panicmem` 和 `panicdivide` 类似，都是对 `panic( )` 的极简封装：

`func panicmem() {       panicCheck2("invalid memory address or nil pointer dereference")       panic(memoryError)   }   `

**划重点：这种方式是通过信号软中断的方式来走到 Go 注册的信号处理逻辑，从而调用到 `panic( )`  的函数。**

童鞋可能会好奇，信号处理的逻辑什么时候注册进去的？

在进程初始化的时候，创建 M0（线程）的时候用系统调用 `sigaction` 给信号注册处理函数为 `sigtramp` ，调用栈如下：

`mstartm0 （proc.go）       -> initsig (signal_unix.go:113)           -> setsig （os_linux.go）   `

这样的话，以后触发了信号软中断，就能调用到 Go 的信号处理函数，从而进行语言层面的 panic 处理 。

总的来说，这个是从系统层面到特定语言层面的处理转变。

## 

程序猿主动

第三种方式，就是程序猿自己主动调用 `panic` 抛出来的。

`func main() {       panic("panic test")   }   `

简单的函数调用，这个超简单的。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

聊聊 panic 到底是什么？

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

现在我们摸透了 panic 产生的姿势，以上三种方式，无论哪一种都归一到 `panic( )` 这个函数调用。**所以有一点很明确：panic 这个东西是语言层面的处理逻辑。**

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

panic 发生之后，如果 Go 不做任何特殊处理，默认行为是打印堆栈，退出程序。

现在回到最本源的问题：panic 到底是什么？

这里不纠结概念，只描述几个简单的事实：

1. `panic( )` 函数内部会产生一个关键的数据结构体 `_panic` ，并且挂接到 goroutine 之上；

1. `panic( )` 函数内部会执行 `_defer` 函数链条，并针对 `_panic` 的状态进行**对应的处理**；

什么叫做 `panic( )` 的对应的处理？

循环执行 goroutine 上面的 `_defer` 函数链，如果执行完了都还没有恢复 `_panic` 的状态，那就没得办法了，**退出进程，打印堆栈**。

如果在 goroutine 的 `_defer` 链上，有个朋友 recover 了一下，把这个 `_panic` 标记成恢复，那事情就到此为止，就从这个 `_defer` 函数执行后续正常代码即可，走 `deferreturn` 的逻辑。

**所以，panic 是什么 ？**

**小奇伢认为，它就是个特殊函数调用，仅此而已。**

有多特殊？

限于篇幅，此处不表，下篇剖析其深度原理。

给读者朋友先思考几个小问题，下次我们一起揭秘：

- panic 究竟是啥？是一个结构体？还是一个函数？

- 为什么 panic 会让 Go 进程退出的 ？

- 为什么 recover 一定要放在 defer 里面才生效？

- 为什么 recover 已经放在 defer 里面，但是进程还是没有恢复？

- 为什么 panic 之后，还能再 panic ？有啥影响？

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

总结

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

1. panic 产生的三大姿势：**程序猿主动，编译器辅助逻辑，软中断信号触发**；

1. 无论哪一种姿势，最终都是归一到 `panic( )` 函数的处理，**panic 只是语言层面的处理逻辑**；

1. panic 发生之后，如果不做处理，默认行为是打印 panic 原因，打印堆栈，进程退出；

## 

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

后记

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

本篇姿势篇，下篇原理篇。**更多干货**在路上，敬请期待。**点赞、在看** 是对奇伢最大的支持，感谢大家！

~完～

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

往期推荐

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

往期推荐

\[

Go 存储基础 — “文件”被偷偷修改？来，给它装个监控！

\](http://mp.weixin.qq.com/s?\_\_biz=Mzg3NTU3OTgxOA==&mid=2247493695&idx=1&sn=1dbb729c9d50afaf2e868a3be70ded93&chksm=cf3df8faf84a71ecc0b7ddab09fa2a4dac7db7acbad2fb8e1104de8f3ae41764b9f29c19e89e&scene=21#wechat_redirect)

\[

Go 并发编程 — 结构体多字段的原子操作

\](http://mp.weixin.qq.com/s?\_\_biz=Mzg3NTU3OTgxOA==&mid=2247493239&idx=1&sn=9b0a2187e30fba80643a8ed5610adf0f&chksm=cf3df6b2f84a7fa4976cdf8dcdf876e01cb1807de679c996a09ee5b2100a3c3b617c3033dc8b&scene=21#wechat_redirect)

\[

Golang 最细节篇 — 解密 defer 原理，究竟背着程序猿做了多少事情？

\](http://mp.weixin.qq.com/s?\_\_biz=Mzg3NTU3OTgxOA==&mid=2247486776&idx=1&sn=acd0b940bd4dd939552cfc410b021949&chksm=cf3e1dfdf84994ebccd479c23945508dfcfa43e6c089fe0bf852a4a6ba5001f9abfd2258eabb&scene=21#wechat_redirect)

\[

深入剖析 defer 原理篇 —— 函数调用的原理？

\](http://mp.weixin.qq.com/s?\_\_biz=Mzg3NTU3OTgxOA==&mid=2247486774&idx=1&sn=3b59ac2efc97b7bbebbde366d0ee4ea0&chksm=cf3e1df3f84994e5c988bc00d369c12884f75a3234912cccd04c53142c334b0488118ff43ea0&scene=21#wechat_redirect)

坚持思考，方向比努力更重要。**关注我：奇伢云存储**

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

![](https://res.wx.qq.com/t/fed_upload/b39ef69e-c4d6-4169-8612-5f00a84860e7/wx-avatar-default.svg)

公众号

阅读 983

​

写留言

**留言 2**

- .

  2021年9月1日

  赞

  Good ! 学习时候认为就是抛异常，远远不止这么简单！

- yongxinz

  2021年9月1日

  赞

  学到了

已无更多数据

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/ASQrEXvmx62PpTb2V6eBBF4lNBnFP1zsL9CJcqsECwriaKh9NoHqqiaIJyhgudorYzkgtDvDvRX6bLp7CPMKufibg/300?wx_fmt=png&wxfrom=18)

码农桃花源

13分享4

2

写留言

**留言 2**

- .

  2021年9月1日

  赞

  Good ! 学习时候认为就是抛异常，远远不止这么简单！

- yongxinz

  2021年9月1日

  赞

  学到了

已无更多数据
