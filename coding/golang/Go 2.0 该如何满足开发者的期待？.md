# 

华章计算机

 _2022年02月10日 08:03_

以下文章来源于CSDN ，作者Seth Vargo

[

![](http://wx.qlogo.cn/mmhead/Q3auHgzwzM5cvOsZy9wYacdpSLicuibpMX0ibCb5C0Z8JibGbnmPKCB0Yw/0)

**CSDN**.

成就一亿技术人

](https://mp.weixin.qq.com/s?__biz=Mzg5MDU5NTM1NQ==&mid=2247536528&idx=1&sn=adc81a7882ab01534e6e1ea296febf13&chksm=cfd83ee4f8afb7f20db7b47fd3b459f5788b099fef26dd7584d59b8cf5918191a00b0b8bb2dc&mpshare=1&scene=24&srcid=0211yGVnucp5okIynBtb68Ym&sharer_sharetime=1644515996528&sharer_shareid=5fb9813bfe9ffc983435bfc8d8c5e9ca&key=daf9bdc5abc4e8d0eb1dd72370a32044f81f323957ad5efd14319cf45ae781c23922d4d6a88117744dfbbab295b2f4c80bd0dad9e57c4f57185f75a2f3e7e7900198f1e34a175eb19c509afaee28875547476823a46bb4c6623c7873fe384b2447d14b28d09abac4f7a2e9b542941c41d9fdf0d247cf1215d8568f393f22b6cc&ascene=0&uin=MTEwNTU1MjgwMw%3D%3D&devicetype=Windows+11+x64&version=63090b19&lang=zh_CN&countrycode=CN&exportkey=n_ChQIAhIQX%2F0fwgK8PLhk6%2FahcJOhXhLmAQIE97dBBAEAAAAAAGjnAfbUhs8AAAAOpnltbLcz9gKNyK89dVj0Vgz4tkZRMawWUmOmbOzZlFSHcd72Hkv0HePOwjh83IahwEUWmTgbXvIqQeVqGCrPbulJOW60zOO65pqio1zugJpdeVet05a%2FbRxTqtlKbkPA09K0D5%2B8WWJ%2B7WcoEi54pv9Lq6Ts76Wj84R1wk6slonyhr1lcVzWPUo1vXtcixoZOUvGpvgovXd5L%2BQKdt1xWlsto6OFww%2B8Ft6JypCUXrD18yx78rmWufdGKf%2Fwie1eNLNOpGYU7K0qPI2Ll1RA&acctmode=0&pass_ticket=4%2FTWzeiXon4uFaG%2BVB5zZHCHA7UqLzLofcP05EN56go3BTZ4razlEGoYLXTZriim&wx_header=1&fasttmpl_type=0&fasttmpl_fullversion=7351805-zh_CN-zip&fasttmpl_flag=1#)

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

作者 | Seth Vargo    译者 | 弯月  

出品 | CSDN（ID：CSDNnews）

虽然 Go 是我最喜欢的编程语言之一，但它远不够完美。在过去的 10 年里，我使用 Go 构建了很多小型个人项目和大型应用程序。自 2009 年第一版发布以来，Go 有了很大变化，但我希望通过本文表达我认为 Go 仍有待改进的一些领域。

在此之前，首先声明一点：我并不是在批评 Go 开发团队或个人的贡献。我的目的是让 Go 成为最好的编程语言。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**0****1**        

现代模板引擎

  

Go的标准库有两个模板包：text/template 和 html/template。二者使用的语法大致相同，但 html/template 会处理实体转义和一些其他特定于 Web 的构造。然而不幸的是，对于一些高级的用例，这两个库都不够强大，依然需要进行大量开发。

- 编译时错误。与 Go本身不同，Go 的模板包允许将整数作为字符串传递，但会在运行时报错。这意味着，开发人员无法依赖类型系统，他们需要严格测试模板中所有可能的输入。Go 的模板包应该支持编译时类型检查。
    
- 与Go语言一致的range子句。虽然我使用 Go 已有10 年之久，但仍然不太理解 Go 模板中 range 子句的顺序，因为它有时与Go是相反的。例如，如果使用两个参数，那么模板引擎与标准库是一致的：
    

`{{ range $a, $b := .Items }} // [$a = 0,$b = "foo"]``   ``for a, b := range items { // [a = 0, b ="foo"]`

然而，当只有一个参数时，模板引擎就会返回值，而Go会返回索引：

`{{ range $a := .Items }} // [$a ="foo"]``   ``for a := range items { // [a = 0]`

Go的模板包应该与标准库相一致。

- 多提供标准功能，减少反射的使用。我认为大多数开发人员永远不需要使用反射。但是，如果想实现的功能超出了基本的加减法，那么 Go 的模板包就会强迫你使用反射，因为它的内置函数非常少，只能满足一小部分用例。
    

在编写完 Consul Template（https://github.com/hashicorp/consul-template）之后，我明显感觉到标准的 Go 模板功能不足以满足用户的需求。超过一半的问题都与使用 Go 的模板语言有关。如今，Consul Template 拥有 50 多个“辅助”功能，其中绝大多数都应该由标准模板语言提供。

不仅仅是我遇到了这个问题，Hugo 有一个广泛的辅助函数列表（https://gohugo.io/functions/），其中的绝大多数都应该由标准模板语言提供。即使在我最近的一个项目中，也无法避免使用反射。

Go的模板语言确实需要更广泛的函数集。

- 条件短路。Go 的模板语言总是在子句中对整个条件进行求值，这会产生一些非常可笑的错误（直到运行时才会显示出来。）考虑以下情况，假设 $foo 可能为 nil：
    

```
{{ if (and $foo $foo.Bar) }}
```

虽然代码看上去没问题，但是两个 and 条件都需要求值，也就是说表达式中没有短路逻辑。如果 $foo 为 nil，就会引发运行时异常。

为了解决这个问题，你必须分割条件子句：

```
{{ if $foo }} {{ if $foo.Bar }}{{ end }}
```

Go的模板语言应该像标准库一样运行，在遇到第一个真值条件后就停止。

- 特定于 Web 的小工具。多年来，我一直是一名 Ruby on Rails 开发人员，我时常感叹于用 Ruby on Rails 构建漂亮的 Web 应用程序是多么容易。然而使用 Go 的模板语言，即使是最简单的任务，比如输出句子中的每一个单词，初学者也无法完成，尤其是与 Rails 的 Enumerable#to_sentence 相比。
    

**0****2**        

range的改进：不要复制值

  

虽然文档很齐全，但 range 子句中的值被复制还是出人意料。例如，考虑以下代码：

`type Foo struct {`  `bar string``}``   ``func main() {`  `list :=[]Foo{{"A"}, {"B"}, {"C"}}``   `  `cp := make([]*Foo,len(list))`  `for i, value := rangelist {`    `cp[i] = &value`  `}``   `  `fmt.Printf("list:%q\n", list)`  `fmt.Printf("cp:%q\n", cp)``}`

cp的值是什么？[A B C] ？不好意思，你错了。实际上，cp 的值为：

`[C C C]`

这是因为 Go 的 range 子句中使用的是值的副本，而不是值本身。在 Go 2.0 中，range 子句应该通过引用传递值。此外，我还有一些关于 Go 2.0 的建议，包括改进 for-loop，在每次迭代中重新定义范围循环变量。  

  

**0****3**        

确定的select

  

在 select 语句中，如果有多个条件为真，那么究竟会执行哪个语句是不确定的。这个细微的差异会导致错误，这个问题与使用方法相似的switch语句相比更为明显，因为  switch 语句会按照写入的顺序逐个求值。

考虑以下代码，我们希望的行为是：如果系统停止，则什么也不做。否则等待 5 秒，然后超时。

`for {`  `select {`  `case <-doneCh: // or<-ctx.Done():`    `return`  `case thing :=<-thingCh:`    `// ... long-runningoperation`  `case<-time.After(5*time.Second):`    `returnfmt.Errorf("timeout")`  `}``}`

对于 select 语句，如果多个条件为真（例如 doneCh 已关闭且已超过 5 秒），则最后会执行哪个语句是不确定的行为。因此，我们不得不加上冗长的取消代码：

`for {`  `// Check here in casewe've been CPU throttled for an extended time, we need to`  `// check graceful stopor risk returning a timeout error.`  `select {`  `case <-doneCh:`    `return`  `default:`  `}``   `  `select {`  `case <-doneCh:`    `return`  `case thing :=<-thingCh:`    `// Even though thiscase won, we still might ALSO be stopped.`    `select {`    `case <-doneCh:`      `return`    `default:`    `}`    `// ...`  `default<-time.After(5*time.Second):`    `// Even though thiscase won, we still might ALSO be stopped.`    `select {`    `case <-doneCh:`      `return`    `default:`    `}`    `return fmt.Errorf("timeout")`  `}``}`

如果能够将 select 语句改成确定的，则原始代码（更简单且更容易编写）就可以按预期工作。但是，由于 select 的非确定性，我们必须不断检查占主导地位的条件。

此外，我希望看到“如果该分支通过条件判断，就执行下面的代码，否则继续下一个分支”的简写语法。当前的语法很冗长：

`select {``case <-doneCh:`  `return``default:``}`

我很想看到更简洁的检查，比如像下面这样：

```
select <-?doneCh: // not valid Go
```

  

**0****4**        

结构化日志接口

  

Go的标准库包含 log 包，可用于处理基本操作。但是，大多数生产系统都需要结构化的日志记录，而 Go 中也不乏结构化日志记录库：

●  apex/log

●  go-kit/log

●  golang/glog

●  hashicorp/go-hclog

●  inconshreveable/log15

●  rs/zerolog

●  sirupus/logrus

●  uber/zap

由于 Go 在这个领域没有给出明确的意见，因此导致了这些包的泛滥，其中大多数都拥有不兼容的功能和签名。因此，库作者不可能发出结构化日志。例如，我希望能够在 go-retry、go-envconfig 或 go-githubactions 中发出结构化日志，但这样做就会与其中某个库紧密耦合。理想情况下，我希望库的用户可以自行选择结构化日志记录解决方案，但是由于缺乏通用接口，使得这种选择非常困难。

Go标准库需要定义一个结构化的日志接口，现有的上游包都可以选择实现该接口。然后，作为库作者，我可以选择接受 log.StructuredLogger 接口，实现者可以自己选择：

`func WithLogger(l log.StructuredLogger) Option {`  `return func(f *Foo) *Foo{`    `f.logger = l`    `return f`  `}``}`

我快速整理了一个潦草的接口：

`// StructuredLogger is an interface for structured logging.``type StructuredLogger interface {`  `// Log logs a message.`  `Log(message string, fields...LogField)``   `  `// LogAt logs a messageat the provided level. Perhaps we could also have`  `// Debugf, Infof, etc,but I think that might be too limiting for the standard`  `// library.`  `LogAt(level LogLevel,message string, fields ...LogField)``   `  `// LogEntry logs acomplete log entry. See LogEntry for the default values if`  `// any fields aremissing.`  `LogEntry(entry*LogEntry)``}``   ``// LogLevel is the underlying log level.``type LogLevel uint8``   ``// LogEntry represents a single log entry.``type LogEntry struct {`  `// Level is the loglevel. If no level is provided, the default level of`  `// LevelError is used.`  `Level LogLevel``   `  `// Message is the actuallog message.`  `Message string``   `  `// Fields is the list ofstructured logging fields. If two fields have the same`  `// Name, the later onetakes precedence.`  `Fields []*LogField``}``   ``// LogField is a tuple of the named field (a string) and itsunderlying value.``type LogField struct {`  `Name  string`  `Value interface{}``}`

围绕具体的接口、如何最小化资源分配以及最大化兼容性的讨论有很多，但目标都是定义一个其他日志库可以轻松实现的接口。

回到我从事 Ruby 开发的时代，有一阵子 Ruby 的版本管理器激增，每个版本管理器的配置文件名和语法都不一样。Fletcher Nichol 写了一篇 gist，成功地说服所有 Ruby 版本管理器的维护者对 .ruby-version 进行标准化。我希望 Go 社区也能以类似的方式处理结构化日志。

  

**0****5**        

多错误处理

  

在很多情况下，尤其是后台作业或周期性任务，系统可能会并行处理多个任务或采用continue-on-error策略。在这些情况下，返回多个错误会很有帮助。标准库中没有处理错误集合的内置支持。

Go社区可以围绕多错误处理建立清晰简洁的标准库，这样不仅可以统一社区，而且还可以降低错误处理不当的风险，就好像错误打包和展开那样。

  

**0****6**        

对于error的JSON序列化处理

  

说到错误，你知不知道如果将 error 类型嵌入到结构字段中，然后将这个结构进行JSON序列化，"error"就会被序列化成{}？

`// https://play.golang.org/p/gl7BPJOgmjr``package main``   ``import (` `"encoding/json"`  `"fmt"``)``   ``type Response1 struct {`  `` Err error`json:"error"` ```}``   ``func main() {`  `v1 :=&Response1{Err: fmt.Errorf("oops")}`  `b1, err :=json.Marshal(v1)`  `if err != nil {`    `panic(err)`  `}``   `  `// got:{"error":{}}`  `// want: {"error": "oops"}`  `fmt.Println(string(b1))``}`

至少对于内置的 errorString 类型，Go应当对.Error()的结果进行序列化。或者在 Go 2.0 中，也可以在试图对 error 类型进行序列化时，如果没有定义序列化逻辑，则返回错误。

  

**0****7**        

标准库中不再有公共变量

  

仅举一个例子，http.DefaultClient 和 http.DefaultTransport 都是具有共享状态的全局变量。http.DefaultClient 没有设置超时，因此很容易引发 DOS 攻击，并造成瓶颈。许多包都会修改 http.DefaultClient 和 http.DefaultTransport，这会导致开发人员需要浪费数天来跟踪错误。  

Go2.0 应该将这些全局变量设为私有，并通过函数调用来公开它们，而这个函数的调用会返回一个唯一的已分配好的变量。或者，Go 2.0 也可以实现一种“冻结”的全局变量，这种全局变量无法被其他包修改。

从软件供应链的角度来看，这类问题也令我很担忧。如果我开发一个包，秘密地修改 http.DefaultTransport，然后使用自定义的 RoundTripper，将所有流量都转发到我的服务器，那就麻烦了。

**08**       

缓冲渲染器的原生支持

  

有些问题是因为不为人知或没有文档记录。大多数示例，包括 Go 文档中的示例，都应该按照以下行为进行JSON序列化或通过 Web 请求呈现 HTML：

`func toJSON(w http.ResponseWriter, i interface{}) {`  `if err :=json.NewEncoder(w).Encode(i); err != nil {`    `http.Error(w,"oops", http.StatusInternalServerError)`  `}``}``   ``func toHTML(w http.ResponseWriter, tmpl string, i interface{}) {`  `if err :=templates.ExecuteTemplate(w, tmpl, i); err != nil {`    `http.Error(w,"oops", http.StatusInternalServerError)`  `}``}` 

然而，对于上述两段代码，如果 i 足够大，则在发送第一个字节（和 200 状态代码）后，编码/执行就可能会失败。此时，请求是无法恢复，因为无法更改响应代码。

为了解决这个问题，广泛接受的解决方案是先渲染，然后复制到 w。这个解决方案仍然有可能引发错误（由于连接问题，写入 w 失败），但可以确保在发送第一个字节之前编码/执行成功。但是，为每个请求分配一个字节切片可能会很昂贵，因此通常都会使用缓冲池。

这种方法非常啰嗦，并且将许多不必要的复杂性推给了实现者。相反，如果 Go 能够使用 EncodePooled 之类的函数，自动处理这个缓冲池管理就好了。

  

**0****9**        

总结

  

Go是我最喜欢的编程语言之一，这就是为什么我愿意说出自己的一些批评意见。与其他编程语言一样，Go 也在不断发展。你赞同本文指出的这些问题吗？请在下方留言。

参考链接：

https://www.sethvargo.com/what-id-like-to-see-in-go-2/

  

**10**        

学习资料

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

01

NEW

**《Go语言精进之路：从新手到高手的编程思想、方法和技巧》**

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**Go圈知名架构师和布道者撰写，**

**3大Go社区力荐，**

**66个主题快速帮你写出Go风格高质量代码**

  

作者：白明

推荐阅读

  

Go入门容易，精进难，如何才能像Go开发团队那样写出符合Go思维和语言惯例的高质量代码呢？本书将从编程思维和实践技巧2个维度给出答案，帮助你快速掌握Go思维，写出Go风格的高质量代码。学完这本书，你将拥有和 Go专家一样的编程思维，写出符合Go惯例和风格的高质量代码。本书内容共分为十部分，限于篇幅，分为两册出版。

  

  
![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

02

**《Go程序设计语言》**

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**经典与权威的碰撞，打造Go语言编程圣经**

  

作者：[美] 艾伦 A. A. 多诺万（Alan A. A. Donovan）

布莱恩 W. 柯尼汉（Brian W. Kernighan）

译者：李道兵 高博 庞向才 金鑫鑫 林齐斌 

推荐阅读

  

《C程序设计语言》作者Kernighan教授与谷歌Go开发团队核心成员Donovan联合编写。凝聚大师毕生造诣，融合Go开发团队智慧，经典与权威的碰撞，打造Go语言编程圣经。学习Go语言程序设计的权威指南。

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

03

**《Head First Go语言程序设计》**

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

   

**Head First系列又一力作，零基础学Go语言不再枯燥**

作者：[美] 杰伊·麦克格瑞恩（Jay McGavren）

译者：刘红泉、王佳

推荐阅读

  

通过这本图文并茂的使用指南，你将会了解到企业希望入门级Go开发人员所知晓的惯例和技术。本书包含语法基础、条件和循环、函数、包、数组、映射、结构、封装和嵌入、接口、故障恢复、共享、自动化测试、Web应用程序等。

  

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

04

**《Go微服务实战》**

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

**给小白的Go语言微服务实战手册**

作者：刘金亮

推荐阅读

  

以实践的角度全方位介绍如何通过Go语言实现微服务模式,书中包含大量案例、代码注释详细、理论解释形象。本书面向所有工程师，即便是没有Go语言基础的Java、PHP、Python工程师也可以直接上手使用，书中对Go语言进行了全面精炼的介绍。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

[更多精彩回顾](http://mp.weixin.qq.com/s?__biz=MjM5ODE2NzE2MA==&mid=2650307630&idx=1&sn=4555f1813768252248807e9e518108ae&chksm=bec20b3989b5822f9f87f62eee12d771b2744ca896e9ace4fb24161f2deccf1b694450219c56&scene=21#wechat_redirect)

  

  

  

  

书讯 | [2月书讯（下）| 新年到，新书到！](http://mp.weixin.qq.com/s?__biz=Mzg5MDU5NTM1NQ==&mid=2247536256&idx=1&sn=8cd031760ccf77758b9e7b3c69534da7&chksm=cfd83ff4f8afb6e20763466c8c1349e49ec50c9b2a749738074c1f3f635b774d3927ff0479cc&scene=21#wechat_redirect)

书讯 | [2月书讯 （上）| 新年到，新书到！](http://mp.weixin.qq.com/s?__biz=Mzg5MDU5NTM1NQ==&mid=2247536253&idx=1&sn=22e238fd3138d5a98c33534905f0cef5&chksm=cfd83f09f8afb61f643987e9704516454f995a8b47fa533322549fd1211cc7193c32674d4223&scene=21#wechat_redirect)

资讯 | [重磅！达摩院发布2022十大科技趋势](http://mp.weixin.qq.com/s?__biz=Mzg5MDU5NTM1NQ==&mid=2247533400&idx=1&sn=fd501ceeb910807539dca77ec8ceb6b9&chksm=cfd8322cf8afbb3add3fa271959ce7bef393b23f12c869c721bc8afac5efd73fcb9395bae160&scene=21#wechat_redirect)

书单 | [6本书，读懂2022年最火的边缘计算](http://mp.weixin.qq.com/s?__biz=Mzg5MDU5NTM1NQ==&mid=2247532694&idx=1&sn=dc1895831f117ebeb22c188b760e7702&chksm=cfd831e2f8afb8f4e1f634ac48637afca1f4e43b9b63273ff257b5f38e91e2b83cb6209d7f67&scene=21#wechat_redirect)

干货 | [前端应用和产品逻辑的核心：交互流](http://mp.weixin.qq.com/s?__biz=Mzg5MDU5NTM1NQ==&mid=2247536251&idx=1&sn=e40a70c6101c200f86f831c8e2ba778e&chksm=cfd83f0ff8afb619192f40db5590609fafa1705122d9bab3ff6b46c5503dcd0fee91be9aaee0&scene=21#wechat_redirect)

收藏 | [Three.js 的 3D 粒子动画：群星送福](http://mp.weixin.qq.com/s?__biz=Mzg5MDU5NTM1NQ==&mid=2247536257&idx=1&sn=ee0fd466be495c627f63c062a3b4c0bd&chksm=cfd83ff5f8afb6e3b1c6fba6814df6d5c7762a00a8b117ab8ce22805c8368e59aa5f810893f2&scene=21#wechat_redirect)

干货 | [Java静态编译技术：突破Java“冷启动”桎梏，实现启动性能“质”的飞跃](http://mp.weixin.qq.com/s?__biz=Mzg5MDU5NTM1NQ==&mid=2247534208&idx=3&sn=0117b3585afa2dddb9c395f1301d73ac&chksm=cfd837f4f8afbee2c07fdeb7e944c07f4f246e091d3a50f5c2d0f313e9f026fa0169474eeb5f&scene=21#wechat_redirect)

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

阅读 330

​