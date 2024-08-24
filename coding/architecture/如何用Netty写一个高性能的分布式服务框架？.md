# 

石杉的架构笔记

 _2021年12月08日 09:00_

以下文章来源于阿里技术 ，作者家纯

[

![](http://wx.qlogo.cn/mmhead/Q3auHgzwzM4yGEW4Je6O6aLExtOx3rQQXxibBoiawVa1y9ncdIvz8aWA/0)

**阿里技术**.

阿里巴巴官方技术号，关于阿里的技术创新均呈现于此。

](https://mp.weixin.qq.com/s?__biz=MzU0OTk3ODQ3Ng==&mid=2247509492&idx=1&sn=12189ec40fc28d94010ca7a723b4fb12&chksm=fba54df7ccd2c4e151d27778c903b327fae825d58a68dce3a254a370d939f2fee8134e3c6339&mpshare=1&scene=24&srcid=12085QpqHRktUBonWQ7x7xpd&sharer_sharetime=1638970926966&sharer_shareid=5fb9813bfe9ffc983435bfc8d8c5e9ca&key=daf9bdc5abc4e8d07131d4d57fd29686b363ffb6019421d45d0142106928c86bbe3e69c9cb5adfcb5c83f0ec7615a356ae2bfb7957825370bd112ccd2ad8af6c94b96c2bd3c9cf1c1b8654f621c90f0f59a7e43aa0ef49213430c2ec3364b1a0ba404d86011c4f1999ef86239f696ddb4b55fa4daa7c56377a715483cf6adee0&ascene=0&uin=MTEwNTU1MjgwMw%3D%3D&devicetype=Windows+11+x64&version=63090b19&lang=zh_CN&countrycode=CN&exportkey=n_ChQIAhIQ0ghstyLwmLOARsX2e3y7PhLmAQIE97dBBAEAAAAAAC1gNg17%2B08AAAAOpnltbLcz9gKNyK89dVj0RPLLaGZj58rXll3XcHtX3yHxXZ2sUH4bAA66gmxoDEWxDPG0PDFzWIkmNEeXwscXtXgcgyyZsHoBt6Z4k7vFd%2Fw44cZMZJyCfU4F8rZQf74e2zsuE%2Fg2bXOkrZB%2F5KfECxul72ndrYZPnLDgoCw7efRU9fk3nsT8yJA0LHoACE6KJclPRLkgsrFzIL8O0oCN4Cxc7q82v3EF0G%2BoILtgBxnyZIHtLHcUrWWuu8iJRJxzqBsKY2oHI2qTCNO2n1Dy&acctmode=0&pass_ticket=arGK91VNGeM2bay%2B1qgIKnLwh217X3OIXb0ySW0L1P8UUl8bpjVlJVT7fv0WQuZt&wx_header=1&fasttmpl_type=0&fasttmpl_fullversion=7351805-zh_CN-zip&fasttmpl_flag=1#)

点击上方蓝色“石杉的架构笔记”，选择“设为星标”

回复“PDF”获取独家整理的学习资料！

![图片](https://mmbiz.qpic.cn/mmbiz_png/1J6IbIcPCLaW5XycOGDYFcpTv0H3PVbticUqZp6zb57v4GEx3qwp9vjDD2D6NtDB1ibQcdMiaBiaKME64tbaxPTmTA/640?wx_fmt=png&wxfrom=13&tp=wxpic)

![图片](https://mmbiz.qpic.cn/mmbiz_png/1J6IbIcPCLaW5XycOGDYFcpTv0H3PVbtdMDFfAI3hiaylH3JsdFiaZ4UibibFb6a4mDY8t1oD1r91mLHNYIPr1gToA/640?wx_fmt=png&wxfrom=13&tp=wxpic)

![图片](https://mmbiz.qpic.cn/mmbiz_png/1J6IbIcPCLaW5XycOGDYFcpTv0H3PVbt6y0ldN6Qj8y6Wb0mAKQsic2481R59ib2wVVb0Cuz62dicZ8icHI88mMzpw/640?wx_fmt=png&wxfrom=13&tp=wxpic)

  

![图片](https://mmbiz.qpic.cn/mmbiz_png/1J6IbIcPCLZibTYKUOYdFVrgjDrBzkDHwymep3YEvG7WjqePNGcESLfr53gvTZoTgkLcjRIbl9a0exwwLS3ghMg/640?wx_fmt=png&wxfrom=13&tp=wxpic)  

代码托管仓库：https://gitee.com/suzhou-mopdila-information/ruyuan-dfs

![图片](https://mmbiz.qpic.cn/mmbiz_png/c6gqmhWiafyraCOeb13lWRHKRDxOnqibg5ZxwV0hs3wIJa5qwByVh9J5FfvPdTKQJASYsBiahgiaZcKJribYsKOib1rg/640?wx_fmt=png&wxfrom=13&tp=wxpic)

长按扫描上方免费参加

  

![图片](https://mmbiz.qpic.cn/mmbiz_png/1J6IbIcPCLanG2nzjVZov1VibmPXR7lDibFjRdTvH1iaDz9ic4H3xRZCz7BM563rFFfvRukzX7uhOqXc8yDhEZqicFQ/640?wx_fmt=png&wxfrom=13&tp=wxpic)

来源：公众号【阿里技术】

**| 什么是 Netty? 能做什么?**

  

- Netty 是一个致力于创建高性能网络应用程序的成熟的 IO 框架。
    

  

- 相比较与直接使用底层的 Java IO API，你不需要先成为网络专家就可以基于 Netty 去构建复杂的网络应用。
    

  

- 业界常见的涉及到网络通信的相关中间件大部分基于 Netty 实现网络层。
    

  

**| 设计一个分布式服务框架**

  

**· Architecture**

  

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/Z6bicxIx5naIohPtQJfcNCrXMkbmXiaia6LDoz4PxKXAkneiaF6DQllZdavgWmIfL4ibQ8GOicrVfhHia4IEKKIAE7b1A/640?wx_fmt=jpeg&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

  

**· 远程调用的流程**

  

- 启动服务端(服务提供者)并发布服务到注册中心。
    

  

- 启动客户端(服务消费者)并去注册中心订阅感兴趣的服务。
    

  

- 客户端收到注册中心推送的服务地址列表。
    

  

- 调用者发起调用，Proxy从服务地址列表中选择一个地址并将请求信息 <group，providerName，version>，methodName，args[] 等信息序列化为字节数组并通过网络发送到该地址上。
    

  

- 服务端收到收到并反序列化请求信息，根据 <group，providerName，version> 从本地服务字典里查找到对应providerObject，再根据 <methodName，args[]> 通过反射调用指定方法，并将方法返回值序列化为字节数组返回给客户端。
    

  

- 客户端收到响应信息再反序列化为 Java 对象后由 Proxy 返回给方法调用者。
    

  

以上流程对方法调用者是透明的，一切看起来就像本地调用一样。

  

**· 远程调用客户端图解**

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

重要概念：RPC三元组 <ID，Request，Response>。

  

PS: 若是 netty4.x 的线程模型，IO Thread(worker) —> Map<InvokeId，Future> 代替全局 Map 能更好的避免线程竞争。

  

**· 远程调用服务端图解**

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

**· 远程调用传输层图解**

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

**· 设计传输层协议栈**

  

**协议头**

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

**协议体**

  

1）metadata: <group，providerName，version>

  

2）methodName

  

3）parameterTypes[] 真的需要吗?

  

（a）有什么问题?

  

- 反序列化时 ClassLoader.loadClass() 潜在锁竞争。
    
- 协议体码流大小。
    
- 泛化调用多了参数类型。
    

  

（b）能解决吗?

  

- Java方法静态分派规则参考JLS <Java语言规范> $15.12.2.5 Choosing the Most Specific Method 章节。
    

  

（c）args[]

  

（d）其他：traceId，appName…

  

**| 一些Features&好的实践&压榨性能**

  

**· 创建客户端代理对象**

  

1）Proxy 做什么?

  

- 集群容错 —> 负载均衡 —> 网络
    

  

2）有哪些创建 Proxy 的方式?

  

- jdk proxy/javassist/cglib/asm/bytebuddy
    

  

3）要注意的：

  

- 注意拦截toString，equals，hashCode等方法避免远程调用。
    

  

4）推荐的(bytebuddy)：

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

**· 优雅的同步/异步调用**

  

- 先往上翻再看看“远程调用客户端图解”
    
- 再往下翻翻看看 Failover 如何处理更好
    
- 思考下如何拿到 future?
    

  

**· 单播/组播**

  

- 消息派发器
    
- FutureGroup
    

  

**· 泛化调用**

  

- Object $invoke(String methodName，Object... args)
    
- parameterTypes[]
    

  

**· 序列化/反序列化**

  

协议 header 标记 serializer type，同时支持多种。

  

**· 可扩展性**

  

Java SPI：

  

- java.util.ServiceLoader
    
- META-INF/services/com.xxx.Xxx
    

  

**· 服务级别线程池隔离**

  

要挂你先挂，别拉着我。

  

**· 责任链模式的拦截器**

  

太多扩展需要从这里起步。

  

**· 指标度量(Metrics)**

  

**· 链路追踪**

  

OpenTracing

  

**· 注册中心**

  

**· 流控(应用级别/服务级别)**

  

要有能方便接入第三方流控中间件的扩展能力。

  

**· Provider线程池满了怎么办?**

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

**· 软负载均衡**

  

1）加权随机 (二分法，不要遍历)

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

2）加权轮训(最大公约数)

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

3）最小负载

  

4）一致性 hash (有状态服务场景)

  

5）其他

  

注意：要有预热逻辑。

  

**· 集群容错**

  

1）Fail-fast

  

2）Failover

  

异步调用怎么处理?

  

- Bad
    

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

- Better
    

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

3）Fail-safe

  

4）Fail-back

  

5）Forking

  

6）其他

  

**· 如何压榨性能(Don’t trust it，Test it)**

  

1）ASM 写个 FastMethodAccessor 来代替服务端那个反射调用

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

2）序列化/反序列化

  

在业务线程中序列化/反序列化，避免占用 IO 线程：

  

- 序列化/反序列化占用数量极少的 IO 线程时间片。
    

  

- 反序列化常常会涉及到 Class 的加载，loadClass 有一把锁竞争严重(可通过 JMC 观察一下)。
    

  

选择高效的序列化/反序列化框架：

  

- 如kryo/protobuf/protostuff/hessian/fastjson/…
    

  

选择只是第一步，它(序列化框架)做的不好的，去扩展和优化之：

  

- 传统的序列化/反序列化+写入/读取网络的流程：java对象--> byte[] -->堆外内存 / 堆外内存--> byte[] -->java对象。
    

  

- 优化：省去 byte[] 环节，直接 读/写 堆外内存，这需要扩展对应的序列化框架。
    

  

- String 编码/解码优化。
    

  

- Varint 优化：多次 writeByte 合并为 writeShort/writeInt/writeLong。
    

  

- Protostuff 优化举例：UnsafeNioBufInput 直接读堆外内存/UnsafeNioBufOutput 直接写堆外内存。
    

  

3）IO 线程绑定 CPU

  

4）同步阻塞调用的客户端和容易成为瓶颈，客户端协程：

  

- Java层面可选的并不多，暂时也都不完美。
    

  

|   |   |
|---|---|
|name|description|
|kilim|编译期间字节码增强|
|quasar agent|动态字节码增强|
|ali_wisp|ali_jvm 在底层直接实现|

  

5）Netty Native Transport & PooledByteBufAllocator：

  

- 减小GC带来的波动。
    

  

6）尽快释放 IO 线程去做他该做的事情，尽量减少线程上下文切换。

  

**| Why Netty?**

  

**· BIO vs NIO**

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

**· Java 原生 NIO API 从入门到放弃**

  

**复杂度高**

  

- API复杂难懂，入门困。
    
- 粘包/半包问题费神。
    
- 需超强的并发/异步编程功底，否则很难写出高效稳定的实现。
    

  

**稳定性差，坑多且深**

  

- 调试困难，偶尔遭遇匪夷所思极难重现的bug，边哭边查是常有的事儿。
    

  

- linux 下 EPollArrayWrapper.epollWait 直接返回导致空轮训进而导致 100% cpu 的 bug 一直也没解决利索，Netty帮你 work around (通过rebuilding selector)。
    

  

**NIO代码实现方面的一些缺点**

  

1）Selector.selectedKeys() 产生太多垃圾

  

Netty 修改了 sun.nio.ch.SelectorImpl 的实现，使用双数组代替 HashSet 存储来 selectedKeys：

  

- 相比HashSet(迭代器，包装对象等)少了一些垃圾的产生(help GC)。
    
- 轻微的性能收益(1~2%)。
    

  

Nio 的代码到处是 synchronized (比如 allocate direct buffer 和 Selector.wakeup() )：

  

- 对于 allocate direct buffer，Netty 的 pooledBytebuf 有前置 TLAB(Thread-local allocation buffer)可有效的减少去竞争锁。
    

  

- wakeup 调用多了锁竞争严重并且开销非常大(开销大原因: 为了在 select 线程外跟 select 线程通信，linux 平台上用一对 pipe，windows 由于 pipe 句柄不能放入 fd_set，只能委曲求全用两个 tcp 连接模拟)，wakeup 调用少了容易导致 select 时不必要的阻塞(如果懵逼了就直接用 Netty 吧，Netty中有对应的优化逻辑)。
    

  

- Netty Native Transport 中锁少了很多。
    

  

2）fdToKey 映射

  

- EPollSelectorImpl#fdToKey 维持着所有连接的 fd(描述符)对应 SelectionKey 的映射，是个 HashMap。
    

  

- 每个 worker 线程有一个 selector，也就是每个 worker 有一个 fdToKey，这些 fdToKey 大致均分了所有连接。
    

  

- 想象一下单机 hold 几十万的连接的场景，HashMap 从默认 size=16，一步一步 rehash...
    

  

3）Selector在linux 平台是 Epoll LT 实现

  

- Netty Native Transport支持Epoll ET。
    

  

4）Direct Buffers 事实上还是由 GC 管理

  

- DirectByteBuffer.cleaner 这个虚引用负责 free direct memory，DirectByteBuffer 只是个壳子，这个壳子如果坚强的活下去熬过新生代的年龄限制最终晋升到老年代将是一件让人伤心的事情…
    

  

- 无法申请到足够的 direct memory 会显式触发 GC，Bits.reserveMemory() -> { System.gc() }，首先因为 GC 中断整个进程不说，代码中还 sleep 100 毫秒，醒了要是发现还不行就 OOM。
    

  

- 更糟的是如果你听信了个别<XX优化宝典>谗言设置了-XX:+DisableExplicitGC 参数，悲剧会静悄悄的发生...
    

  

- Netty的UnpooledUnsafeNoCleanerDirectByteBuf 去掉了 cleaner，由 Netty 框架维护引用计数来实时的去释放。
    

  

**| Netty 的真实面目**

  

**· Netty 中几个重要概念及其关系**

  

**EventLoop**

  

- 一个 Selector。
    

  

- 一个任务队列(mpsc_queue: 多生产者单消费者 lock-free)。
    

  

- 一个延迟任务队列(delay_queue: 一个二叉堆结构的优先级队列，复杂度为O(log n))。
    

  

- EventLoop 绑定了一个 Thread，这直接避免了pipeline 中的线程竞争。
    

  

**Boss: mainReactor 角色，Worker: subReactor 角色**

  

- Boss 和 Worker 共用 EventLoop 的代码逻辑，Boss 处理 accept 事件，Worker 处理 read，write 等事件。
    

  

- Boss 监听并 accept 连接(channel)后以轮训的方式将 channel 交给 Worker，Worker 负责处理此 channel 后续的read/write 等 IO 事件。
    

  

- 在不 bind 多端口的情况下 BossEventLoopGroup 中只需要包含一个 EventLoop，也只能用上一个，多了没用。
    

  

- WorkerEventLoopGroup 中一般包含多个 EventLoop，经验值一般为 cpu cores * 2(根据场景测试找出最佳值才是王道)。
    

  

- Channel 分两大类 ServerChannel 和 Channel，ServerChannel 对应着监听套接字(ServerSocketChannel)，Channel 对应着一个网络连接。
    

  

**· Netty4 Thread Model**

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

**· ChannelPipeline**

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

**· Pooling&reuse**

  

**PooledByteBufAllocator**

  

- 基于 jemalloc paper (3.x)
    

  

- ThreadLocal caches for lock free：这个做法导致曾经有坑——申请(Bytebuf)线程与归还(Bytebuf)线程不是同一个导致内存泄漏，后来用一个mpsc_queue解决，代价就是牺牲了一点点性能。
    

  

- Different size classes。
    

  

**Recycler**

  

- ThreadLocal + Stack。
    

  

- 曾经有坑，申请(元素)线程与归还(元素)线程不是同一个导致内存泄漏。
    

  

- 后来改进为不同线程归还元素的时候放入一个 WeakOrderQueue 中并关联到 stack 上，下次 pop 时如果 stack 为空则先扫描所有关联到当前 stack 上的 weakOrderQueue。
    

  

- WeakOrderQueue 是多个数组的链表，每个数组默认size=16。
    

  

- 存在的问题：思考一下老年代对象引用新生代对象对 GC 的影响？
    

  

**· Netty Native Transport**

  

相比 Nio 创建更少的对象，更小的 GC 压力。

  

针对 linux 平台优化，一些 specific features：

  

- SO_REUSEPORT - 端口复用(允许多个 socket 监听同一个 IP+端口，与 RPS/RFS 协作，可进一步提升性能)：可把 RPS/RFS 模糊的理解为在软件层面模拟多队列网卡，并提供负载均衡能力，避免网卡收包发包的中断集中的一个 CPU core 上而影响性能。
    

  

- TCP_FASTOPEN - 3次握手时也用来交换数据。
    

  

- EDGE_TRIGGERED (支持Epoll ET是重点)。
    

  

- Unix 域套接字(同一台机器上的进程间通信，比如Service Mesh)。
    

  

**· 多路复用简介**

  

**select/poll**

  

- 本身的实现机制上的限制(采用轮询方式检测就绪事件，时间复杂度: O(n)，每次还要将臃肿的 fd_set 在用户空间和内核空间拷贝来拷贝去)，并发连接越大，性能越差。
    

  

- poll 相比 select 没有很大差异，只是取消了最大文件描述符个数的限制。
    

  

- select/poll 都是 LT 模式。
    

  

**epoll**

  

- 采用回调方式检测就绪事件，时间复杂度: O(1)，每次 epoll_wait 调用只返回已就绪的文件描述符。
    

  

- epoll 支持 LT 和 ET 模式。
    

  

**· 稍微深入了解一点 Epoll**

  

**LT vs ET**

  

概念：

  

- LT：level-triggered 水平触发
    
- ET：edge-triggered 边沿触发
    

  

可读：

  

- buffer 不为空的时候 fd 的 events 中对应的可读状态就被置为1，否则为0。
    

  

可写：

  

- buffer 中有空间可写的时候 fd 的 events 中对应的可写状态就被置为1，否则为0。
    

  

图解：

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

**epoll 三个方法简介**

  

1）主要代码：linux-2.6.11.12/fs/eventpoll.c

  

2）int epoll_create(int size)

  

创建 rb-tree(红黑树)和 ready-list (就绪链表)：

  

- 红黑树O(logN)，平衡效率和内存占用，在容量需求不能确定并可能量很大的情况下红黑树是最佳选择。
    

  

- size参数已经没什么意义，早期epoll实现是hash表，所以需要size参数。
    

  

3）int epoll_ctl(int epfd，int op，int fd，struct epoll_event *event)

  

- 把epitem放入rb-tree并向内核中断处理程序注册ep_poll_callback，callback触发时把该epitem放进ready-list。
    

  

4）int epoll_wait(int epfd，struct epoll_event * events，int maxevents，int timeout)

  

- ready-list —> events[]。
    

  

**epoll 的数据结构**

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

**epoll_wait 工作流程概述** 

  

对照代码：linux-2.6.11.12/fs/eventpoll.c：

  

1）epoll_wait 调用 ep_poll

  

- 当 rdlist(ready-list) 为空(无就绪fd)时挂起当前线程,直到 rdlist 不为空时线程才被唤醒。
    

  

2）文件描述符 fd 的 events 状态改变

  

- buffer由不可读变为可读或由不可写变为可写，导致相应fd上的回调函数ep_poll_callback被触发。
    

  

3）ep_poll_callback 被触发

  

- 将相应fd对应epitem加入rdlist，导致rdlist不空，线程被唤醒，epoll_wait得以继续执行。
    

  

4）执行 ep_events_transfer 函数

  

- 将rdlist中的epitem拷贝到txlist中，并将rdlist清空。
    

  

- 如果是epoll LT，并且fd.events状态没有改变(比如buffer中数据没读完并不会改变状态)，会再重新将epitem放回rdlist。
    

  

5）执行 ep_send_events 函数

  

- 扫描txlist中的每个epitem，调用其关联fd对应的poll方法取得较新的events。
    

  

- 将取得的events和相应的fd发送到用户空间。
    

  

**· Netty 的最佳实践**

  

1）业务线程池必要性

  

- 业务逻辑尤其是阻塞时间较长的逻辑，不要占用netty的IO线程，dispatch到业务线程池中去。
    

  

2）WriteBufferWaterMark

  

- 注意默认的高低水位线设置(32K~64K)，根据场景适当调整(可以思考一下如何利用它)。
    

  

3）重写 MessageSizeEstimator 来反应真实的高低水位线

  

- 默认实现不能计算对象size，由于write时还没路过任何一个outboundHandler就已经开始计算message size，此时对象还没有被encode成Bytebuf，所以size计算肯定是不准确的(偏低)。
    

  

4）注意EventLoop#ioRatio的设置(默认50)

  

- 这是EventLoop执行IO任务和非IO任务的一个时间比例上的控制。
    

  

5）空闲链路检测用谁调度?

  

- Netty4.x默认使用IO线程调度，使用eventLoop的delayQueue，一个二叉堆实现的优先级队列，复杂度为O(log N)，每个worker处理自己的链路监测，有助于减少上下文切换，但是网络IO操作与idle会相互影响。
    

  

- 如果总的连接数小，比如几万以内，上面的实现并没什么问题，连接数大建议用HashedWheelTimer实现一个IdleStateHandler，HashedWheelTimer复杂度为 O(1)，同时可以让网络IO操作和idle互不影响，但有上下文切换开销。
    

  

6）使用ctx.writeAndFlush还是channel.writeAndFlush?

  

- ctx.write直接走到下一个outbound handler，注意别让它违背你的初衷绕过了空闲链路检测。
    

  

- channel.write从末尾开始倒着向前挨个路过pipeline中的所有outbound handlers。
    

  

7）使用Bytebuf.forEachByte() 来代替循环 ByteBuf.readByte()的遍历操作，避免rangeCheck()

  

8）使用CompositeByteBuf来避免不必要的内存拷贝

  

- 缺点是索引计算时间复杂度高，请根据自己场景衡量。
    

  

9）如果要读一个int，用Bytebuf.readInt()，不要Bytebuf.readBytes(buf，0，4)

  

- 这能避免一次memory copy (long，short等同理)。
    

  

10）配置

UnpooledUnsafeNoCleanerDirectByteBuf来代替jdk的DirectByteBuf，让netty框架基于引用计数来释放堆外内存

  

io.netty.maxDirectMemory：

  

- < 0: 不使用cleaner，netty方面直接继承jdk设置的最大direct memory size，(jdk的direct memory size是独立的，这将导致总的direct memory size将是jdk配置的2倍)。
    

  

- == 0: 使用cleaner，netty方面不设置最大direct memory size。
    

  

> 0：不使用cleaner，并且这个参数将直接限制netty的最大direct memory size，(jdk的direct memory size是独立的，不受此参数限制)。

  

11）最佳连接数

  

- 一条连接有瓶颈，无法有效利用cpu，连接太多也白扯，最佳实践是根据自己场景测试。
    

  

12）使用PooledBytebuf时要善于利用 -Dio.netty.leakDetection.level 参数

  

- 四种级别：DISABLED(禁用)，SIMPLE(简单)，ADVANCED(高级)，PARANOID(偏执)。
    

  

- SIMPLE，ADVANCED采样率相同，不到1%(按位与操作 mask ==128 - 1)。
    

  

- 默认是SIMPLE级别，开销不大。
    

  

- 出现泄漏时日志会出现“LEAK: ”字样，请时不时grep下日志，一旦出现“LEAK: ”立刻改为ADVANCED级别再跑，可以报告泄漏对象在哪被访问的。
    

  

- PARANOID：测试的时候建议使用这个级别，100%采样。
    

  

13）Channel.attr()，将自己的对象attach到channel上

  

- 拉链法实现的线程安全的hash表，也是分段锁(只锁链表头)，只有hash冲突的情况下才有锁竞争(类似ConcurrentHashMapV8版本)。
    

  

- 默认hash表只有4个桶，使用不要太任性。
    

  

**· 从 Netty 源码中学到的代码技巧**

  

1）海量对象场景中

AtomicIntegerFieldUpdater --> AtomicInteger

  

- Java中对象头12 bytes(开启压缩指针的情况下)，又因为Java对象按照8字节对齐，所以对象最小16 bytes，AtomicInteger大小为16 bytes，AtomicLong大小为 24 bytes。
    

  

- AtomicIntegerFieldUpdater作为static field去操作volatile int。
    

  

2）FastThreadLocal，相比jdk的实现更快

  

- 线性探测的Hash表 —> index原子自增的裸数组存储。
    

  

3）IntObjectHashMap / LongObjectHashMap …

  

- Integer—> int
    
- Node[] —> 裸数组
    

  

4）RecyclableArrayList

  

- 基于前面说的Recycler，频繁new ArrayList的场景可考虑。
    

  

5）JCTools

  

- 一些jdk没有的 SPSC/MPSC/SPMC/MPMC 无锁并发队以及NonblockingHashMap(可以对比ConcurrentHashMapV6/V8)
    

---

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

阅读 4539

​