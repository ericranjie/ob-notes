
Golang技术分享

 _2022年01月05日 08:01_

编者荐语：

反射是一个在业务开发中不易接触到的语言特性，很多童鞋或许还未能掌握。本文通过实现一个SQL语句生成器工具，循序渐进地讲解了反射的核心知识点，非常适合阅读学习。

以下文章来源于网管叨bi叨 ，作者KevinYan11

[

![](http://wx.qlogo.cn/mmhead/Q3auHgzwzM7xEq7qxG9ickUKJd2MD9A9sEe6YhvCFeZd8jImnffzq9g/0)

**网管叨bi叨**.

分享软件开发和系统架构设计基础、Go 语言和Kubernetes。

](https://mp.weixin.qq.com/s?__biz=MzkyMzIyNjIxMQ==&mid=2247485793&idx=1&sn=c467fff326f420f4c357baf123fcc97f&chksm=c1e9106df69e997be3d14f2b9d6e89f02c747027d6d1316ae437d8f9e42b92db68cf23b84944&mpshare=1&scene=24&srcid=0105UdOLeSnNHtU5MfC8vmkO&sharer_sharetime=1641361923434&sharer_shareid=5fb9813bfe9ffc983435bfc8d8c5e9ca&key=daf9bdc5abc4e8d0644ef99587dcfd3587d5d251c743309e4bc4aa1310dd8c8a3ce8edaf223a3117d1747413d31208c51140c02a4d279de1a2978f563782c02f7618ffe5414013638f5ac10c47455332a747292fadc198a8f94c2764f4a4f43d28984eb5b54fd67f7a1285d415e000bb5ecb2ec59c173770879581d6f306d229&ascene=0&uin=MTEwNTU1MjgwMw%3D%3D&devicetype=Windows+11+x64&version=63090b19&lang=zh_CN&countrycode=CN&exportkey=n_ChQIAhIQI7ntgCKtyHhlkWCkQMuA0hLmAQIE97dBBAEAAAAAAJzVLGitgP0AAAAOpnltbLcz9gKNyK89dVj0%2BCH0Q1yq2ipcI1Pysbtvje4SNuB%2B3d7biDEmrUjK7hnYKuoU8H2S%2BiY93VTiYe7jjEsag%2BphoTBIJodSYFLTDqJ2yGkMH6LvI9ze5JaManF1fzkoj%2FJQA%2FMrMicyqpg0%2Fz9%2FUvVc%2FO4pdHd%2FVyP8yCAdcrkO%2F3RXVtnC7VuKHpPWbG4DOORT2ppx4DRej%2BoMP4m%2FnHg3VYcM8HViUtLIjqV0GnOQ3BOqaJjzd%2ByccGFyNBoTXJc7SFa6Eun%2FXAX4&acctmode=0&pass_ticket=Gq40St6FppqHckohUpl5IAx%2F1RtKy4OgI%2FZ1KOFnrl0LiuVcdurHh8r5PD3RUcde&wx_header=1&fasttmpl_type=0&fasttmpl_fullversion=7350504-zh_CN-zip&fasttmpl_flag=1#)

今天来聊一个平时用的不多，但是很多框架或者基础库会用到的语言特性--反射，反射并不是`Go`语言独有的能力，其他编程语言都有。这篇文章的目标是简单地给大家梳理一下反射的应用场景和使用方法。

我们平时写代码能接触到与反射联系比较紧密的一个东西是结构体字段的标签，这个我准备放在后面的文章再梳理。

我准备通过用反射搞一个通用的`SQL`构造器的例子，带大家掌握反射这个知识点。这个是看了国外一个博主写的例子，觉得思路很好，我又对其进行了改进，让构造器的实现更丰富了些。

> 本文的思路参考自：https://golangbot.com/reflection/ ，本文内容并非只是对原文的简单翻译，具体看下面的内容吧~！

## 什么是反射

反射是程序在运行时检查其变量和值并找到它们类型的能力。听起来比较笼统，接下来我通过文章的例子一步步带你认识反射。

## 为什么需要反射

当学习反射的时候，每个人首先会想到的问题都是 “为什么我们要在运行时检查变量的类型呢，程序里的变量在定义的时候我们不都已经给他们指定好类型了吗？” 确实是这样的，但也并非总是如此，看到这你可能心里会想，大哥，你在说什么呢，em... 还是先写一个简单的程序，解释一下。

`package main      import (         "fmt"   )      func main() {         i := 10       fmt.Printf("%d %T", i, i)   }   `

在上面的程序里， 变量`i`的类型在编译时是已知的，我们在下一行打印了它的值和类型。

现在让我们理解一下 ”在运行时知道变量的类型的必要“。假设我们要编写一个简单的函数，它将一个结构体作为参数，并使用这个参数创建一个`SQL`插入语句。

考虑一下下面这个程序

`package main      import (         "fmt"   )      type order struct {         ordId      int       customerId int   }      func main() {         o := order{           ordId:      1234,           customerId: 567,       }       fmt.Println(o)   }   `

我们需要写一个接收上面定义的结构体`o`作为参数，返回类似`INSERT INTO order VALUES(1234, 567)`这样的`SQL`语句。这个函数定义写来很容易，比如像下面这样。

`package main      import (         "fmt"   )      type order struct {         ordId      int       customerId int   }      func createQuery(o order) string {         i := fmt.Sprintf("INSERT INTO order VALUES(%d, %d)", o.ordId, o.customerId)       return i   }      func main() {         o := order{           ordId:      1234,           customerId: 567,       }       fmt.Println(createQuery(o))   }   `

上面例子的`createQuery`使用参数`o` 的`ordId`和`customerId`字段创建SQL。

现在让我们将我们的`SQL`创建函数定义地更抽象些，下面还是用程序附带说明举一个案例，比如我们想泛化我们的`SQL`创建函数使其适用于任何结构体。

`package main      type order struct {         ordId      int       customerId int   }      type employee struct {         name string       id int       address string       salary int       country string   }      func createQuery(q interface{}) string {     }   `

现在我们的目标是，改造`createQuery`函数，让它能接受任何结构作为参数并基于结构字段创建`INSERT` 语句。比如如果传给`createQuery`的参数不再是`order`类型的结构体，而是`employee`类型的结构体时

 `e := employee {           name: "Naveen",           id: 565,           address: "Science Park Road, Singapore",           salary: 90000,           country: "Singapore",       }`

那它应该返回的`INSERT`语句应该是

`INSERT INTO employee (name, id, address, salary, country)    VALUES("Naveen", 565, "Science Park Road, Singapore", 90000, "Singapore")`  

由于`createQuery` 函数要适用于任何结构体，因此它需要一个 `interface{}`类型的参数。为了说明问题，简单起见，我们假定`createQuery`函数只处理包含`string` 和 `int` 类型字段的结构体。

编写这个`createQuery`函数的唯一方法是检查在运行时传递给它的参数的类型，找到它的字段，然后创建SQL。这里就是需要反射发挥用的地方啦。在后续步骤中，我们将学习如何使用`Go`语言的反射包来实现这一点。

## Go语言的反射包

`Go`语言自带的`reflect`包实现了在运行时进行反射的功能，这个包可以帮助识别一个`interface{}`类型变量其底层的具体类型和值。我们的`createQuery`函数接收到一个`interface{}`类型的实参后，需要根据这个实参的底层类型和值去创建并返回`INSERT`语句，这正是反射包的作用所在。

在开始编写我们的通用`SQL`生成器函数之前，我们需要先了解一下`reflect`包中我们会用到的几个类型和方法，接下来我们先逐个学习一下。

### reflect.Type 和 reflect.Value

经过反射后`interface{}`类型的变量的底层具体类型由`reflect.Type`表示，底层值由`reflect.Value`表示。`reflect`包里有两个函数`reflect.TypeOf()` 和`reflect.ValueOf()` 分别能将`interface{}`类型的变量转换为`reflect.Type`和`reflect.Value`。这两种类型是创建我们的`SQL`生成器函数的基础。

让我们写一个简单的例子来理解这两种类型。

`package main      import (         "fmt"       "reflect"   )      type order struct {         ordId      int       customerId int   }      func createQuery(q interface{}) {         t := reflect.TypeOf(q)       v := reflect.ValueOf(q)       fmt.Println("Type ", t)       fmt.Println("Value ", v)         }   func main() {         o := order{           ordId:      456,           customerId: 56,       }       createQuery(o)      }   `

上面的程序会输出：

`Type  main.order     Value  {456 56}` 

上面的程序里`createQuery`函数接收一个`interface{}`类型的实参，然后把实参传给了`reflect.Typeof`和`reflect.Valueof` 函数的调用。从输出，我们可以看到程序输出了`interface{}`类型实参对应的底层具体类型和值。

### Go语言反射的三法则

这里插播一下反射的三法则，他们是：

1. 从接口值可以反射出反射对象。
    
2. 从反射对象可反射出接口值。
    
3. 要修改反射对象，其值必须可设置。
    

反射的第一条法则是，我们能够吧`Go`中的接口类型变量转换成反射对象，上面提到的`reflect.TypeOf`和 `reflect.ValueOf` 就是完成的这种转换。第二条指的是我们能把反射类型的变量再转换回到接口类型，最后一条则是与反射值是否可以被更改有关。三法则详细的说明可以去看看德莱文大神写的文章 [Go反射的实现原理](https://mp.weixin.qq.com/s?__biz=MzU5NTAzNjc3Mg==&mid=2247483962&idx=1&sn=e13df5c5e016215302205f5ec8fbb857&scene=21#wechat_redirect)，文章开头就有对三法则说明的图解，再次膜拜。

下面我们接着继续了解完成我们的SQL生成器需要的反射知识。

### reflect.Kind

`reflect`包中还有一个非常重要的类型，`reflect.Kind`。

`reflect.Kind`和`reflect.Type`类型可能看起来很相似，从命名上也是，Kind和Type在英文的一些Phrase是可以互转使用的，不过在反射这块它们有挺大区别，从下面的程序中可以清楚地看到。

`package main   import (         "fmt"       "reflect"   )      type order struct {         ordId      int       customerId int   }      func createQuery(q interface{}) {         t := reflect.TypeOf(q)       k := t.Kind()       fmt.Println("Type ", t)       fmt.Println("Kind ", k)         }   func main() {         o := order{           ordId:      456,           customerId: 56,       }       createQuery(o)      }   `

上面的程序会输出

`Type  main.order     Kind  struct`  

通过输出让我们清楚了两者之间的区别。`reflect.Type` 表示接口的实际类型，即本例中`main.order` 而`Kind`表示类型的所属的种类，即`main.order`是一个「struct」类型，类似的类型`map[string]string`的Kind就该是「map」。

### 反射获取结构体字段的方法

我们可以通过`reflect.StructField`类型的方法来获取结构体下字段的类型属性。`reflect.StructField`可以通过`reflect.Type`提供的下面两种方式拿到。

`// 获取一个结构体内的字段数量   NumField() int   // 根据 index 获取结构体内字段的类型对象   Field(i int) StructField   // 根据字段名获取结构体内字段的类型对象   FieldByName(name string) (StructField, bool)   `

`reflect.structField`是一个struct类型，通过它我们又能在反射里知道字段的基本类型、Tag、是否已导出等属性。

`type StructField struct {    Name string    Type      Type      // field type    Tag       StructTag // field tag string     ......   }   `

与`reflect.Type`提供的获取`Field`信息的方法相对应，`reflect.Value`也提供了获取`Field`值的方法。

`func (v Value) Field(i int) Value {   ...   }      func (v Value) FieldByName(name string) Value {   ...   }   `

这块需要注意，不然容易迷惑。下面我们尝试一下通过反射拿到`order`结构体类型的字段名和值

`package main      import (    "fmt"    "reflect"   )      type order struct {    ordId      int    customerId int   }      func createQuery(q interface{}) {    t := reflect.TypeOf(q)    if t.Kind() != reflect.Struct {     panic("unsupported argument type!")    }    v := reflect.ValueOf(q)    for i:=0; i < t.NumField(); i++ {     fmt.Println("FieldName:", t.Field(i).Name, "FiledType:", t.Field(i).Type,      "FiledValue:", v.Field(i))    }      }   func main() {    o := order{     ordId:      456,     customerId: 56,    }    createQuery(o)      }   `

上面的程序会输出：

`FieldName: ordId FiledType: int FiledValue: 456   FieldName: customerId FiledType: int FiledValue: 56   `

除了获取结构体字段名称和值之外，还能获取结构体字段的Tag，这个放在后面的文章我再总结吧，不然篇幅就太长了。

### reflect.Value转换成实际值

现在离完成我们的SQL生成器还差最后一步，即还需要把`reflect.Value`转换成实际类型的值，`reflect.Value`实现了一系列`Int()`, `String()`，`Float()`这样的方法来完成其到实际类型值的转换。

## 用反射搞一个SQL生成器

上面我们已经了解完写这个SQL生成器函数前所有的必备知识点啦，接下来就把他们串起来，加工完成`createQuery`函数。

这个SQL生成器完整的实现和测试代码如下：

`package main      import (    "fmt"    "reflect"   )      type order struct {    ordId      int    customerId int   }      type employee struct {    name    string    id      int    address string    salary  int    country string   }      func createQuery(q interface{}) string {    t := reflect.TypeOf(q)    v := reflect.ValueOf(q)    if v.Kind() != reflect.Struct {     panic("unsupported argument type!")    }    tableName := t.Name() // 通过结构体类型提取出SQL的表名    sql := fmt.Sprintf("INSERT INTO %s ", tableName)    columns := "("    values := "VALUES ("    for i := 0; i < v.NumField(); i++ {     // 注意reflect.Value 也实现了NumField,Kind这些方法     // 这里的v.Field(i).Kind()等价于t.Field(i).Type.Kind()     switch v.Field(i).Kind() {     case reflect.Int:      if i == 0 {       columns += fmt.Sprintf("%s", t.Field(i).Name)       values += fmt.Sprintf("%d", v.Field(i).Int())      } else {       columns += fmt.Sprintf(", %s", t.Field(i).Name)       values += fmt.Sprintf(", %d", v.Field(i).Int())      }     case reflect.String:      if i == 0 {       columns += fmt.Sprintf("%s", t.Field(i).Name)       values += fmt.Sprintf("'%s'", v.Field(i).String())      } else {       columns += fmt.Sprintf(", %s", t.Field(i).Name)       values += fmt.Sprintf(", '%s'", v.Field(i).String())      }     }    }    columns += "); "    values += "); "    sql += columns + values    fmt.Println(sql)    return sql   }      func main() {    o := order{     ordId:      456,     customerId: 56,    }    createQuery(o)       e := employee{     name:    "Naveen",     id:      565,     address: "Coimbatore",     salary:  90000,     country: "India",    }    createQuery(e)   }   `

同学们可以把代码拿到本地运行一下，上面的例子会根据传递给函数不同的结构体实参，输出对应的标准`SQL`插入语句

`INSERT INTO order (ordId, customerId); VALUES (456, 56);    INSERT INTO employee (name, id, address, salary, country); VALUES ('Naveen', 565, 'Coimbatore', 90000, 'India');` 

## 总结

这篇文章通过利用反射完成一个实际应用来教会大家`Go`语言反射的基本使用方法，虽然反射看起来挺强大，但使用反射编写清晰且可维护的代码非常困难，应尽可能避免，仅在绝对必要时才使用。

我的看法是如果是要写业务代码，根本不需要使用反射，如果要写类似`encoding/json`，`gorm`这些样的库倒是可以利用反射的强大功能简化库使用者的编码难度。

![](http://mmbiz.qpic.cn/mmbiz_png/z4pQ0O5h0f4pGhLz2xEbRFHnAQon2QLYgbBibCJo1ibJHesLWshPJeRibateRtAqkaf6BgjlbhYiaxHLq6Zu07CRPw/300?wx_fmt=png&wxfrom=19)

**网管叨bi叨**

分享软件开发和系统架构设计基础、Go 语言和Kubernetes。

296篇原创内容

公众号

阅读 1135

​