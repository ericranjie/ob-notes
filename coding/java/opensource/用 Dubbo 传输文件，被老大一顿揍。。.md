# 

点击关注 👉 Java面试那些事儿

_2021年10月16日 10:34_

大家好，我是D哥

点击关注下方公众号，Java面试资料都在这里![图片](https://mmbiz.qpic.cn/mmbiz_png/b96CibCt70iaajvl7fD4ZCicMcjhXMp1v6UibM134tIsO1j5yqHyNhh9arj090oAL7zGhRJRq6cFqFOlDZMleLl4pw/640?wx_fmt=png&wxfrom=13&tp=wxpic)

![](http://mmbiz.qpic.cn/mmbiz_png/R5ic1icyNBNd5jvclGgwOZI5CIazqQm21GGq2VvGjrjTbbRiaic3TlrI3BX4Snpz6ibmO1ukibWDsVqj1bBN8cV54h3w/300?wx_fmt=png&wxfrom=19)

**Java面试那些事儿**

回复 java ，领取Java面试题。分享AI编程，Java教程，Java面试辅导，Java编程视频，Java下载，Java技术栈，AI工具，Java开源项目，Java简历模板，Java招聘，Java实战，Java面试经验，IDEA教程。

305篇原创内容

公众号

作者：空无

来源：juejin.cn/post/6963642641506369566

**# 背景**

[公司之前有一个 Dubbo 服务，其内部封装了腾讯云的对象存储服务 SDK，目的是统一管理这种三方服务的SDK，其他系统直接调用这个对象存储的 Dubbo 服务。](http://mp.weixin.qq.com/s?__biz=MzIzMzgxOTQ5NA==&mid=2247557735&idx=1&sn=b66951ef2091bf5abb199e0ea45b3a3e&chksm=e8fc7e6edf8bf778aa14d4ea7f51c3856183d4072a51c5a4251df36b20705e5c3a8c251e63d6&scene=21#wechat_redirect)

[这样可以避免因平台 SDK 出现不兼容的大版本更新，从而导致公司所有系统修改跟着升级的问题。](http://mp.weixin.qq.com/s?__biz=MzIzMzgxOTQ5NA==&mid=2247557735&idx=1&sn=b66951ef2091bf5abb199e0ea45b3a3e&chksm=e8fc7e6edf8bf778aa14d4ea7f51c3856183d4072a51c5a4251df36b20705e5c3a8c251e63d6&scene=21#wechat_redirect)

想法是好的，不过这种做法并不合适，因为 Dubbo 并不适合传输文件。好在这个系统在上线不久就没人用废弃了……

虽然系统废弃了，不过就这个 Dubbo 上传文件的主题还是可以详细分析下，聊聊它到底为什么不适合传文件。

**# Dubbo 怎么传文件？**

难道这样直接传 File 吗？

```
void sendPhoto(File photo);
```

当然不行！Dubbo 只是将对象进行序列化然后传输，而 File 对象就算序列化也无法处理文件的数据，所以只能直接发送文件内容：

```
void sendPhoto(byte[] photo);
```

但这样就会导致 consumer 端需要一次性读取完整的文件内容至内存中，再大的内存也扛不住这样玩。而且 provider 端在接受数据解析报文时，也需要一次性将 byte\[\] 读取至内存中，也是一样有内存占用过高问题。

**# 单连接模型问题**

除了内存占用问题之外，Dubbo（这里指 Dubbo 协议）的单连接模型也不适合文件传输。

Dubbo 协议默认是单连接的模型，即一个 provider 的所有请求都是用一个 TCP 连接。默认使用 Netty 来进行传输，而 Netty 中为了保证 Channel 线程安全，会将写入事件进行排队处理。那么在单连接下，多个请求都会使用同一个连接，也就是同一个 Channel 进行写入数据；当多个请求同时写入时，如果某个报文过大，会导致 Channel 一直在发送这个报文，其他请求的报文写入事件会进行排队，迟迟无法发送，数据都没有发送过去，那么其他的 consumer 也自然会处于阻塞等待响应的状态中，一直无法返回了。

所以在单连接下，如果报文过大，会导致 Netty 的写入事件处理阻塞，无法及时的将数据发送至服务端，从而造成请求白白阻塞的问题。

那既然单连接模型有这么大的缺点，为什么 Dubbo 还要采用单连接呢？

因为省资源啊，TCP 连接这种资源可是很宝贵的，如果单连接可以满足绝大多数场景，那么完全不需要为每个请求准备一个连接。

Dubbo 文档中也提到了单连接设计的原因：

> 因为服务的现状大都是服务提供者少，通常只有几台机器，而服务的消费者多，可能整个网站都在访问该服务，比如 Morgan 的提供者只有 6 台提供者，却有上百台消费者，每天有 1.5 亿次调用，如果采用常规的 hessian 服务，服务提供者很容易就被压跨，通过单一连接，保证单一消费者不会压死提供者，长连接，减少连接握手验证等，并使用异步 IO，复用线程池，防止 C10K 问题。

虽然 Dubbo 协议默认单连接模型，但还是可以设置多连接的：

```
<dubbo:service connections="1"/>
```

不过多连接下，连接和请求并不是一一对应的，而是一个轮询的机制。如下图所示，当配置了N个连接时，对于每一个 Provider 实例都会维护多个连接，在执行请求时会通过轮询的机制，为每次请求分配不同的连接

[!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)](http://mp.weixin.qq.com/s?__biz=MzIzMzgxOTQ5NA==&mid=2247557735&idx=3&sn=217e5a93a2c453e37af409fc2e0d8924&chksm=e8fc7e6edf8bf7787e8cc053a3d92e0581b0da09d33b267f6d8bbcc8b65ec26bd48d8ad1ef86&scene=21#wechat_redirect)

**# 为什么 HTTP 协议“适合”传文件？**

其实这么说并不严谨，并不是 HTTP 协议适合传文件，Dubbo 还支持 HTTP 协议呢（虽然是半残品），一样不适合传文件。

Dubbo 这类 RPC 框架为了满足“调用本地方法像调用远程一样”，必须将数据序列化成语言里的对象，但这样一来就导致无法处理 File 这种形式的对象了。

如果跳出 Dubbo 这种 RPC 框架特性的限制，单独看 HTTP 协议的话，是很适合传输文件的。因为对于 Client 来说，只需要将报文发送至 Server，比如要传输的文件在本地的话，那我完全可以每次只读取文件的一个 Buffer 大小，然后将这个 Buffer 的数据使用 Socket 发送即可；在这种方式下，同时存在于内存中的数据，只会有一个 Buffer 大小，不会有 Dubbo 那样将全部数据读取至内存的问题。

如下图所示，Client 每次只从1GB 文件中读取 4K 大小的 Buffer 数据，然后用 Socket 发送，直至将文件完全读取并发送成功。那么这种方式下对于单次传输来说，内存始终都是只有 4K buffer 大小的占用，并不会像 Dubbo 那样一次性全部读取为 byte\[\] 再发送。

[!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)](http://mp.weixin.qq.com/s?__biz=MzIzMzgxOTQ5NA==&mid=2247557735&idx=3&sn=217e5a93a2c453e37af409fc2e0d8924&chksm=e8fc7e6edf8bf7787e8cc053a3d92e0581b0da09d33b267f6d8bbcc8b65ec26bd48d8ad1ef86&scene=21#wechat_redirect)

对于 Server 端也是一样，Server 端也并不用一次性将所有报文读取至内存中，在解析 Header 中的 Content-Length 后，直接包装一个 InputStream，在这个 InputStream 内部进行读取 Socket Buffer 的数据即可，一样不会有内存占用问题

那既然 HTTP 协议“适合”传输文件，Spring Cloud 的标配 RPC 客户端 - Feign 在传输文件上又会有什么问题呢？

**# Feign 适合传输文件吗**

Feign 其实并不能算一套 RPC 框架，它只是一个 Http Client 而已。在使用 Feign 时，Server 可以是任意的 Http Server，比如实现 Servlet 的 Tomcat/Jetty/Undertow，或者是其他语言的 Apache Server 等等。

而一般用 Feign 时，都是在 Spring Cloud 全家桶环境下，服务端往往是默认的 Tomcat。而 Tomcat 在读取文件报文（form-data）时，会先将报文暂存至磁盘，然后通过 FileItem 读取磁盘中的报文内容。所以在对于 Server 端来说，不会一次性将完整的报文数据读取至内存中，也就不会有内存占用过高的问题。

Feign 中上传文件有以下几种方式：

```
interface SomeApi {
```

Feign 中将参数的编码/序列化抽象为一个 Encoder，对于 HTTP 协议的文件上传也提供了一个 feign-form 模块，该模块中提供了一些 FormEncoder。可无论哪种 FormEncoder 最后都是通过 Feign 封装的 Output 对象进行输出，不过这个 Output 对象却不是那种包装 Socket InputStream 作为中转发送，而是直接作为一个数据的载体，用一个 ByteArrayOutputStream 来存储编码完成的数据。

所以无论怎么定义 FormEncoder，最后数据都会写入到这个 Output 的 ByteArrayOutputStream 中，仍然会将所有数据完整的读取至内存中，一样会有内存占用高的问题。

```
@RequiredArgsConstructor
```

但好在 Feign 只是个 HTTP Client，Server 端还是“增量”读取的，对于 Server 端来说不会有这个内存问题。

**# 总结**

其实 Dubbo 不光是不适合传输文件，大报文场景下都不太合适，Dubbo 的设计更适合小业务报文的传输（默认报文大小只有8MB）。

所以如果有文件上传的场景，尽可能的用客户端直传的方式吧，友好又节省资源！

**!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)技术交流群!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)**

D哥也了一个技术群，主要针对一些新的技术和开源项目值不值得去研究和IDEA使用的“骚操作”，有兴趣入群的同学，可以长扫描区域二维码，一定要注意事项：**城市+昵称+技术方向**，根据格式备注，可快速通过。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

▲长按扫描

**!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)热门推荐：**

- [震惊！Go 标准库源码中竟然包含色情网站！](http://mp.weixin.qq.com/s?__biz=MzIzMzgxOTQ5NA==&mid=2247557735&idx=1&sn=b66951ef2091bf5abb199e0ea45b3a3e&chksm=e8fc7e6edf8bf778aa14d4ea7f51c3856183d4072a51c5a4251df36b20705e5c3a8c251e63d6&scene=21#wechat_redirect)

- [再见MybatisPlus，阿里推出新ORM框架！](http://mp.weixin.qq.com/s?__biz=MzIzMzgxOTQ5NA==&mid=2247557735&idx=3&sn=217e5a93a2c453e37af409fc2e0d8924&chksm=e8fc7e6edf8bf7787e8cc053a3d92e0581b0da09d33b267f6d8bbcc8b65ec26bd48d8ad1ef86&scene=21#wechat_redirect)

- [Java 数组中new Object\[5\]语句是否创建了5个对象？](http://mp.weixin.qq.com/s?__biz=MzIzMzgxOTQ5NA==&mid=2247557735&idx=4&sn=1d31ca50a3ad86ddbdabd72e5ba22375&chksm=e8fc7e6edf8bf77819e04382ac3d2621db394777736b708c30a8f901c8beaa26644cc2da65a6&scene=21#wechat_redirect)

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

阅读 3658

​

写留言

**留言 1**

- 陈良良

  2021年10月16日

  赞

  所以还是客户端穿好了文件后，给服务端瑜哥文件路径参数吗？

已无更多数据

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/R5ic1icyNBNd5jvclGgwOZI5CIazqQm21GGq2VvGjrjTbbRiaic3TlrI3BX4Snpz6ibmO1ukibWDsVqj1bBN8cV54h3w/300?wx_fmt=png&wxfrom=18)

Java面试那些事儿

4分享5

1

写留言

**留言 1**

- 陈良良

  2021年10月16日

  赞

  所以还是客户端穿好了文件后，给服务端瑜哥文件路径参数吗？

已无更多数据
