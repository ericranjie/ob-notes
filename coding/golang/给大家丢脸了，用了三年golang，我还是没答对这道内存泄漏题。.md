# 

CppGuide

_2024年08月23日 14:58_

The following article is from 小白debug Author 9號小白

\[

![](http://wx.qlogo.cn/mmhead/Q3auHgzwzM7bria5qRFgU2FgpCj58q2kCZB9fZVicFiakFkeLtkhtXjSg/0)

**小白debug**.

答应我，关注我之后要好好学技术，不要只是偷我的表情包。。。

\](https://mp.weixin.qq.com/s?\_\_biz=Mzk0MjUwNDE2OA==&mid=2247499639&idx=1&sn=e47e48b01a5b624f75db5bf6b449e1d9&chksm=c39c271f0f7579261f9c60e13b13e6259c192e9b84759737e3e00b782892d77695c387213604&mpshare=1&scene=24&srcid=0824YeTTzsVNmD3FQlPmgobq&sharer_shareinfo=744ddbd47352e1d6933d10dc04ceceb1&sharer_shareinfo_first=744ddbd47352e1d6933d10dc04ceceb1&key=daf9bdc5abc4e8d0e1ed04fbe6ed475b6c1491502c2654aafbfa1f963fea69bb5963bbb5132da8b0dc00a70394862206b65fa8bab262ce122872573ff841a65fcfd734b3d9878c86d383e776255a884146ccedd0d5238def9d9ddb7bd31e9b05acecdce6c16dc7e48c52628855af4682b8bd8f3dc22708d0ae82eb3de177074c&ascene=14&uin=MTEwNTU1MjgwMw%3D%3D&devicetype=iMac+MacBookAir10%2C1+OSX+OSX+14.6.1+build(23G93)&version=13080710&nettype=WIFI&lang=en&session_us=gh_aa193139e730&countrycode=CN&fontScale=100&exportkey=n_ChQIAhIQtaSxzsCU5uUSQdyeAdNOMRKUAgIE97dBBAEAAAAAABUEFGfe8dkAAAAOpnltbLcz9gKNyK89dVj03x4XmbAAsD2dNn8oG43TCxUPtb9qC3qPn6P5%2FrKrDfpoHRwU7bO55JoBQp0AUrOqe444jZ52So1N04TEKwOcQL3oJn4msdmeYOlhAUuvN7XTCWIy84KQRaLMAviC3T3KCm%2FgLXOLPtYMJrSZaedJijMJhSdE56Y%2B3WHnwP7kO4iJ6HDk8AkPWRhU4JDZReQDYAL7%2Flvdscz70trx%2BpHTKllU9WeNzSWpXFuFqFbDtj0AViYb8lZeMk%2FuohDbjBvlCKl%2F%2BfoTwNFBBYYw36MyWiiP64w%2Fy2k2BdoqPVyBkMjw1rZWQpS091x8Mk0feA%3D%3D&acctmode=0&pass_ticket=04%2BENuG9NSdggau7ewIoyS%2FZ5IU8FOH4BGb9BKEHzycqBuQQr30frP1z67SwBtTy&wx_header=0#)

问题

`package main      import (    "fmt"    "io/ioutil"    "net/http"    "runtime"   )      func main() {    num := 6    for index := 0; index < num; index++ {     resp, _ := http.Get("https://www.baidu.com")     _, _ = ioutil.ReadAll(resp.Body)    }    fmt.Printf("此时goroutine个数= %d\n", runtime.NumGoroutine())   }      `

**上面这道题在不执行`resp.Body.Close()`的情况下，泄漏了吗？如果泄漏，泄漏了多少个`goroutine`?**

## 怎么答

- **不进行`resp.Body.Close()`，泄漏是一定的**。但是泄漏的`goroutine`个数就让我迷糊了。由于执行了**6遍**，每次泄漏一个**读和写goroutine**，就是**12个goroutine**，加上`main函数`本身也是一个`goroutine`，所以答案是**13**.

- 然而执行程序，发现**答案是3**，出入有点大，为什么呢？

## 解释

- 我们直接看源码。`golang` 的 `http` 包。

`http.Get()   ![](https://imgkr2.cn-bj.ufileos.com/94738734-9402-475a-b41b-cb443f431f2f.html?UCloudPublicKey=TOKEN_8d8b72be-579a-4e83-bfd0-5f6ce1546f13&Signature=Ceu9w6I4hvRLxVykLhh8IMwBbZ4%253D&Expires=1605828258)      -- DefaultClient.Get   ----func (c *Client) do(req *Request)   ------func send(ireq *Request, rt RoundTripper, deadline time.Time)   -------- resp, didTimeout, err = send(req, c.transport(), deadline)    // 以上代码在 go/1.12.7/libexec/src/net/http/client:174       func (c *Client) transport() RoundTripper {    if c.Transport != nil {     return c.Transport    }    return DefaultTransport   }   `

- 说明 `http.Get` 默认使用 `DefaultTransport` 管理连接。

##### `DefaultTransport` 是干嘛的呢？

`// It establishes network connections as needed   // and caches them for reuse by subsequent calls.   `

- `DefaultTransport` 的作用是根据需要建立网络连接并缓存它们以供后续调用重用。

##### 那么 `DefaultTransport` 什么时候会建立连接呢？

接着上面的代码堆栈往下翻

`func send(ireq *Request, rt RoundTripper, deadline time.Time)    --resp, err = rt.RoundTrip(req) // 以上代码在 go/1.12.7/libexec/src/net/http/client:250   func (t *Transport) RoundTrip(req *http.Request)   func (t *Transport) roundTrip(req *Request)   func (t *Transport) getConn(treq *transportRequest, cm connectMethod)   func (t *Transport) dialConn(ctx context.Context, cm connectMethod) (*persistConn, error) {       ...    go pconn.readLoop()  // 启动一个读goroutine    go pconn.writeLoop() // 启动一个写goroutine    return pconn, nil   }   `

- 一次建立连接，就会启动一个`读goroutine`和`写goroutine`。这就是为什么一次`http.Get()`会泄漏`两个goroutine`的来源。

- 泄漏的来源知道了，也知道是因为没有执行`close`

##### 那为什么不执行 `close` 会泄漏呢？

- 回到刚刚启动的`读goroutine` 的 `readLoop()` 代码里

`func (pc *persistConn) readLoop() {    alive := true    for alive {           ...     // Before looping back to the top of this function and peeking on     // the bufio.Reader, wait for the caller goroutine to finish     // reading the response body. (or for cancelation or death)     select {     case bodyEOF := <-waitForBodyRead:      pc.t.setReqCanceler(rc.req, nil) // before pc might return to idle pool      alive = alive &&       bodyEOF &&       !pc.sawEOF &&       pc.wroteRequest() &&       tryPutIdleConn(trace)      if bodyEOF {       eofc <- struct{}{}      }     case <-rc.req.Cancel:      alive = false      pc.t.CancelRequest(rc.req)     case <-rc.req.Context().Done():      alive = false      pc.t.cancelRequest(rc.req, rc.req.Context().Err())     case <-pc.closech:      alive = false           }           ...    }   }      `

- 简单来说`readLoop`就是一个死循环，只要`alive`为`true`，`goroutine`就会一直存在

- `select` 里面是 `goroutine` **有可能**退出的场景：

- `body` 被读取完毕或`body`关闭

- `request` 主动 `cancel`

- `request` 的 `context Done` 状态 `true`

- 当前的 `persistConn` 关闭

其中第一个 `body` 被读取完或关闭这个 `case`:

`alive = alive &&       bodyEOF &&       !pc.sawEOF &&       pc.wroteRequest() &&       tryPutIdleConn(trace)   `

`bodyEOF` 来源于到一个通道 `waitForBodyRead`，这个字段的 `true` 和 `false` 直接决定了 `alive` 变量的值（`alive=true`那`读goroutine`继续活着，循环，否则退出`goroutine`）。

##### 那么这个通道的值是从哪里过来的呢？

`// go/1.12.7/libexec/src/net/http/transport.go: 1758     body := &bodyEOFSignal{      body: resp.Body,      earlyCloseFn: func() error {       waitForBodyRead <- false       <-eofc // will be closed by deferred call at the end of the function       return nil         },      fn: func(err error) error {       isEOF := err == io.EOF       waitForBodyRead <- isEOF       if isEOF {        <-eofc // see comment above eofc declaration       } else if err != nil {        if cerr := pc.canceled(); cerr != nil {         return cerr        }       }       return err      },     }   `

- 如果执行 `earlyCloseFn` ，`waitForBodyRead` 通道输入的是 `false`，`alive` 也会是 `false`，那 `readLoop()` 这个 `goroutine` 就会退出。

- 如果执行 `fn` ，其中包括正常情况下 `body` 读完数据抛出 `io.EOF` 时的 `case`，`waitForBodyRead` 通道输入的是 `true`，那 `alive` 会是 `true`，那么 `readLoop()` 这个 `goroutine` 就不会退出，同时还顺便执行了 `tryPutIdleConn(trace)` 。

`// tryPutIdleConn adds pconn to the list of idle persistent connections awaiting   // a new request.   // If pconn is no longer needed or not in a good state, tryPutIdleConn returns   // an error explaining why it wasn't registered.   // tryPutIdleConn does not close pconn. Use putOrCloseIdleConn instead for that.   func (t *Transport) tryPutIdleConn(pconn *persistConn) error   `

- `tryPutIdleConn` 将 `pconn` 添加到等待新请求的空闲持久连接列表中，也就是之前说的连接会复用。

##### 那么问题又来了，什么时候会执行这个 `fn` 和 `earlyCloseFn` 呢？

`func (es *bodyEOFSignal) Close() error {    es.mu.Lock()    defer es.mu.Unlock()    if es.closed {     return nil    }    es.closed = true    if es.earlyCloseFn != nil && es.rerr != io.EOF {     return es.earlyCloseFn() // 关闭时执行 earlyCloseFn    }    err := es.body.Close()    return es.condfn(err)   }   `

- 上面这个其实就是我们比较熟悉的 `resp.Body.Close()` ,在里面会执行 `earlyCloseFn`，也就是此时 `readLoop()` 里的 `waitForBodyRead` 通道输入的是 `false`，`alive` 也会是 `false`，那 `readLoop()` 这个 `goroutine` 就会退出，`goroutine` 不会泄露。

`b, err = ioutil.ReadAll(resp.Body)   --func ReadAll(r io.Reader)    ----func readAll(r io.Reader, capacity int64)    ------func (b *Buffer) ReadFrom(r io.Reader)         // go/1.12.7/libexec/src/bytes/buffer.go:207   func (b *Buffer) ReadFrom(r io.Reader) (n int64, err error) {    for {     ...     m, e := r.Read(b.buf[i:cap(b.buf)])  // 看这里，是body在执行read方法     ...    }   }   `

- 这个`read`，其实就是 `bodyEOFSignal` 里的

`func (es *bodyEOFSignal) Read(p []byte) (n int, err error) {    ...    n, err = es.body.Read(p)    if err != nil {     ...        // 这里会有一个io.EOF的报错，意思是读完了     err = es.condfn(err)    }    return   }         func (es *bodyEOFSignal) condfn(err error) error {    if es.fn == nil {     return err    }    err = es.fn(err)  // 这了执行了 fn    es.fn = nil    return err   }   `

- 上面这个其实就是我们比较熟悉的读取 `body` 里的内容。`ioutil.ReadAll()` ,在读完 `body` 的内容时会执行 `fn`，也就是此时 `readLoop()` 里的 `waitForBodyRead` 通道输入的是 `true`，`alive` 也会是 `true`，那 `readLoop()` 这个 `goroutine` 就不会退出，`goroutine` 会泄露，然后执行 `tryPutIdleConn(trace)` 把连接放回池子里复用。

## 总结

- 所以结论呼之欲出了，虽然执行了 `6` 次循环，而且每次都没有执行 `Body.Close()` ,就是因为执行了`ioutil.ReadAll()`把内容都读出来了，连接得以复用，因此只泄漏了一个`读goroutine`和一个`写goroutine`，最后加上`main goroutine`，所以答案就是`3个goroutine`。

- 从另外一个角度说，正常情况下我们的代码都会执行 `ioutil.ReadAll()`，但如果此时忘了 `resp.Body.Close()`，确实会导致泄漏。但如果你**调用的域名一直是同一个**的话，那么只会泄漏一个 `读goroutine` 和一个`写goroutine`，**这就是为什么代码明明不规范但却看不到明显内存泄漏的原因**。

- 那么问题又来了，为什么上面要特意强调是同一个域名呢？改天，回头，以后有空再说吧。

**相关阅读**

- [C/C++多线程编程专栏](<https://mp.weixin.qq.com/mp/appmsgalbum?__biz=Mzk0MjUwNDE2OA==&action=getalbum&album_id=3225971014569017344&uin=&key=&devicetype=iMac+Mac14%2C9+OSX+OSX+14.1.2+build(23B92)&version=13080612&lang=zh_CN&nettype=WIFI&ascene=7&session_us=gh_aa193139e730&fontScale=100>)

- [网络编程重难点分析专栏](<https://mp.weixin.qq.com/mp/appmsgalbum?__biz=Mzk0MjUwNDE2OA==&action=getalbum&album_id=3267921377052049408&uin=&key=&devicetype=iMac+Mac14%2C9+OSX+OSX+14.1.2+build(23B92)&version=13080612&lang=zh_CN&nettype=WIFI&ascene=7&session_us=gh_aa193139e730&fontScale=100>)

- [高性能通信协议设计专栏](<https://mp.weixin.qq.com/mp/appmsgalbum?__biz=Mzk0MjUwNDE2OA==&action=getalbum&album_id=3276199780771414023&uin=&key=&devicetype=iMac+Mac14%2C9+OSX+OSX+14.1.2+build(23B92)&version=13080612&lang=zh_CN&nettype=WIFI&ascene=7&session_us=gh_aa193139e730&fontScale=100>)

- [推荐一波新版优质 Modern C++书籍](http://mp.weixin.qq.com/s?__biz=Mzk0MjUwNDE2OA==&mid=2247499455&idx=1&sn=56e0c1a2e6cebe02e9b381d691ee611e&chksm=c2c09f38f5b7162e897a8efee1c289d962a07d055e9e958951b363f0d377c77588f57155e2d8&scene=21#wechat_redirect)

- [高级 C++ 开发综合岗面试题，能挑战否？](http://mp.weixin.qq.com/s?__biz=Mzk0MjUwNDE2OA==&mid=2247499559&idx=1&sn=5f60c7cfa07c6e6a52c3d2cf8dcbcffe&chksm=c2c09ea0f5b717b6449581cf1565deccdce295b9d142b5b048166265408d5a74e3dd3b5f0559&scene=21#wechat_redirect)

- [大型开源 FTP 软件 FileZilla 源码分析](http://mp.weixin.qq.com/s?__biz=Mzk0MjUwNDE2OA==&mid=2247499465&idx=1&sn=f72c14841b77c69a6ae65fa6fad209e2&chksm=c2c09f4ef5b716589e74597438ce2cb7eee390ca8b2907a974530661e6b2c5afb03274912280&scene=21#wechat_redirect)

- [](http://mp.weixin.qq.com/s?__biz=Mzk0MjUwNDE2OA==&mid=2247499465&idx=1&sn=f72c14841b77c69a6ae65fa6fad209e2&chksm=c2c09f4ef5b716589e74597438ce2cb7eee390ca8b2907a974530661e6b2c5afb03274912280&scene=21#wechat_redirect)[如何尽快适应大型 C++ 项目？](http://mp.weixin.qq.com/s?__biz=Mzk0MjUwNDE2OA==&mid=2247499555&idx=1&sn=fc0b86a694a023a61b7640406f13919e&chksm=c2c09ea4f5b717b2c94cfba5f2067fd7a7386c93b94b3cbde10331a65382f1b1bf06b5919ef5&scene=21#wechat_redirect)

- [如果你想低成本的快速提高开发水平，推荐这个](http://mp.weixin.qq.com/s?__biz=Mzk0MjUwNDE2OA==&mid=2247499473&idx=1&sn=c9dce8726b24b393a2820d3edb06fe4c&chksm=c2c09f56f5b71640341cd63d42ee1914d89a9778c9278abe5a05e0c697cc039bfd9db2a27528&scene=21#wechat_redirect)

**小方目前在一家知名外企做 C++ 架构方面的工作。我建立 高质量 C++ 后端开发微信技术交流群，群里高手如云，不定期分享编程技术，也会提供一些求职和内推资源。**

**现在限时开放中，需要加群交流的同学可以加微信 cppxiaofang，备注“加群”。**

![](http://mmbiz.qpic.cn/sz_mmbiz_png/GSweNIrkicYvM1mIwPctlYONEDKJwUfRZ57uAkVR59MpX1cVnmmnyPZ5O9OCuys78Sy6fOEncwfgWpgCo9Tibeag/300?wx_fmt=png&wxfrom=19)

**CppGuide**

专注于高质量高性能C++开发，站点：cppguide.cn

306篇原创内容

公众号

**关注我，更多的优质内容第一时间阅读**

Reads 1059

​

Comment

**留言 2**

- 网管

  上海Yesterday

  Like

  同一个域名链接才能复用呀

- ThatJin

  广东Yesterday

  Like

  就算循环内部结尾处添加 resp.Body.Close()，结果打印还是：此时goroutine个数= 3

已无更多数据

[](javacript:;)

![](http://mmbiz.qpic.cn/sz_mmbiz_png/GSweNIrkicYvM1mIwPctlYONEDKJwUfRZ57uAkVR59MpX1cVnmmnyPZ5O9OCuys78Sy6fOEncwfgWpgCo9Tibeag/300?wx_fmt=png&wxfrom=18)

CppGuide

436Wow

2

Comment

**留言 2**

- 网管

  上海Yesterday

  Like

  同一个域名链接才能复用呀

- ThatJin

  广东Yesterday

  Like

  就算循环内部结尾处添加 resp.Body.Close()，结果打印还是：此时goroutine个数= 3

已无更多数据
