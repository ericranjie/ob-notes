Linux爱好者

_2021年11月19日 17:51_

↓推荐关注↓

![](https://res.wx.qq.com/t/fed_upload/b39ef69e-c4d6-4169-8612-5f00a84860e7/wx-avatar-default.svg)

公众号

【导读】本文作者介绍了几种服务端实现，并记录了如何迭代、让应用能够服务更大流量的操作。

1. go 语言最大的优势在于能够支持百万级高并发，最关键的在于 go 使用了比线程更轻量级的类似与协程的 goroutine，每一个 goroutine 初始分配的栈空间只有 2k，开启一个 goroutine 方式很简单：

```
func main() { go func(s string) {  fmt.Println(s) }("hello world!")}
```

关键字 go 后面加上方法，第一种处理请求的方式就是每收到一个请求，开启一个 goroutine，并且该 goroutine 处理请求，直到请求处理完成：

```
func main() { router := gin.Default() router.Handle("POST", "/submit", submit) router.Run(":8080")}func submit(ctx *gin.Context) { if err := ctx.Request.ParseForm(); err != nil {  ctx.String(http.StatusBadRequest, "%s", "failure")  return } message := ctx.PostForm("message") go func(msg string) {  fmt.Println("上传信息的处理 ", msg) }(message)}
```

这种方式最简单，但是只适用于中小流量的业务，一旦业务请求的处理内容比较多、流量比较大，程序很快就撑不住了。

2. goroutine 之间进行通信使用 channel：

```
const MAX_QUEUE = 256func main() { channel := make(chan string, MAX_QUEUE) go func(msg string) {  channel <- msg //向信道中存消息 }("ping") msg := <- channel //从信道中去消息 fmt.Println(msg)}
```

第二种方式就是使用 channel，加入缓冲队列的内容，每次收到一个请求，将一些工作放入到队列中，每次从队列中拿出工作进行处理：

```
const MAX_QUEUE = 256var channel chan stringfunc main() { go startProcessor() router := gin.Default() router.Handle("POST", "/submit", submit) router.Run(":8080")}func init() { channel = make(chan string, MAX_QUEUE)}func submit(ctx *gin.Context) { if err := ctx.Request.ParseForm(); err != nil {  ctx.String(http.StatusBadRequest, "%s", "failure")  return } message := ctx.PostForm("message") channel <- message ctx.String(http.StatusOK, "%s", "success")}func startProcessor() { for {  select {  case msg := <-channel:   fmt.Println("上传信息的处理 ", msg)  } }}
```

这种方式适用于请求访问的速率小于等于队列执行的任务速率的情况，如果请求访问的速率远远大于队列执行任务的速率，很快缓冲队列就满了，后续的请求就会阻塞。

3. 使用 Job/Worker 模式，这种可以认为是 go 使用 channel 实现了一个工作线程池，两层的通道系统，一个通道用作任务队列，一个通道用到控制处理任务时的并发量

```
const ( MAX_QUEUE = 256 MAX_WORKER = 32 MAX_WORKER_POOL_SIZE = 5)var JobQueue chan string//用来管理执行管道中的任务type Worker struct { WorkerPool chan chan string JobChannel chan string quit chan bool}func NewWorker(workerPool chan chan string) *Worker { return &Worker{  WorkerPool:workerPool,  JobChannel:make(chan string),  quit:    make(chan bool), }}//开启当前的 worker，循环上传 channel 中的 job；并同时监听 quit 退出标识位，判断是否关闭该 channelfunc (w *Worker) Start() { go func() {  for {   w.WorkerPool <- w.JobChannel   select {   case job := <-w.JobChannel:    fmt.Println(job)   case <-w.quit://收到停止信号，退出    return   }  } }()}//关闭该 channel，这里是将 channel 标识位设置为关闭，实际在 worker 执行中关闭func (w *Worker) Stop() { go func() {  w.quit <- true }()}type Dispatcher struct { WorkerPool chan chan string quit       chan bool}func NewDispatcher(maxWorkers int) *Dispatcher { pool := make(chan chan string, maxWorkers) return &Dispatcher{WorkerPool: pool}}func (d *Dispatcher) dispatcher() { for {  select {  case job := <-JobQueue:   go func(job string) {    jobChannel := <-d.WorkerPool    jobChannel <- job   }(job)  case <-d.quit:   return  } }}func (d *Dispatcher) Run() { for i := 0; i < MAX_WORKER_POOL_SIZE; i++ {  worker := NewWorker(d.WorkerPool)  worker.Start() } go d.dispatcher()}func main() { dispatcher := NewDispatcher(MAX_WORKER)//创建任务分派器 dispatcher.Run()//任务分派器开始工作 router := gin.Default() router.Handle("POST", "/submit", submit) router.Run(":8080")}func init() { JobQueue = make(chan string, MAX_QUEUE)}func submit(ctx *gin.Context) { if err := ctx.Request.ParseForm(); err != nil {  ctx.String(http.StatusBadRequest, "%s", "failure")  return } message := ctx.PostForm("message") JobQueue <- message ctx.String(http.StatusOK, "%s", "success")}
```

> 转自：github.com/guishenbumie/MyBlog/wiki

- EOF -

推荐阅读  点击标题可跳转

1、[Go 处理 PNG 图片及 Mac 图片集转 PDF](http://mp.weixin.qq.com/s?__biz=MzAxODI5ODMwOA==&mid=2666558733&idx=3&sn=8b7c71544303d8f82074ff7e12647e09&chksm=80dcb1a6b7ab38b0d44d531f20a242ec311cb63523d9d36d538b9c1287fade74c7b286a29ea4&scene=21#wechat_redirect)

2、[支持分布式的 go 实现即时通讯系统](http://mp.weixin.qq.com/s?__biz=MzAxODI5ODMwOA==&mid=2666558558&idx=3&sn=1c7ef552e7cb8c29c0418c4b76417ca0&chksm=80dcb0f5b7ab39e398935507c0a822fb9abcad54e545cfd234ebc49d537e73234a2455cd7aa8&scene=21#wechat_redirect)

3、[Golang 闭包的实现](http://mp.weixin.qq.com/s?__biz=MzAxODI5ODMwOA==&mid=2666558626&idx=3&sn=2c41d32fed14d2e69b9cbfb7720655c0&chksm=80dcb009b7ab391ffd7ba74ab3da73c4f6053466311e66071d4ee1bc467dfa85232287128910&scene=21#wechat_redirect)

看完本文有收获？请分享给更多人

推荐关注「Linux 爱好者」，提升Linux技能

![](https://res.wx.qq.com/t/fed_upload/b39ef69e-c4d6-4169-8612-5f00a84860e7/wx-avatar-default.svg)

公众号

点赞和在看就是最大的支持❤️

阅读 2895

​

写留言

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/9aPYe0E1fb3sjicd8JxDra10FRIqT54Zke2sfhibTDdtdnVhv5Qh3wLHZmKPjiaD7piahMAzIH6Cnltd1Nco17Ihjw/300?wx_fmt=png&wxfrom=18)

Linux爱好者

6分享2

写留言

写留言

**留言**

暂无留言
