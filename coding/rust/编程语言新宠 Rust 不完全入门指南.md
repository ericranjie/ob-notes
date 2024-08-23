# 

全栈修仙之路

 _2021年11月30日 00:03_

以下文章来源于编程界 ，作者五月君

[

![](http://wx.qlogo.cn/mmhead/Q3auHgzwzM7pmxHHZmHficO6x3r3gUWZvxdauDImm9UjNPlico4IV0aw/0)

**编程界**.

AI 的崛起势必将编程普及大众，编程界将与 AI 同行，为您带来前沿的技术资讯与思考！

](https://mp.weixin.qq.com/s?__biz=MzI2MjcxNTQ0Nw==&mid=2247497223&idx=1&sn=7fa28f10911743aca524d088365ad795&chksm=ea44575fdd33de49d1486dc0c9f383d011792bb3f7df160bbb48ca0c011171ec86449101b5ac&mpshare=1&scene=24&srcid=1208UBJjQ6kfffJJYdLyub8u&sharer_sharetime=1638923150143&sharer_shareid=5fb9813bfe9ffc983435bfc8d8c5e9ca&key=daf9bdc5abc4e8d0edad9e5e831066b66a4bfc9bcb0781d2b46e5eb0042ae1dbf1973800e299cc786fa5405d0dc8f35103d9a4b3e240da5fa8c1285ce89f98ee7103de14e0092ed1b74cbb14e95e8ff814fd4c2543d2e6d4a1f97abd5f69eee61ec04c645542d7a1c417d2e7046ad2b798e3aa3bf30e7d54de88ba27204f463a&ascene=0&uin=MTEwNTU1MjgwMw%3D%3D&devicetype=Windows+11+x64&version=63090b19&lang=zh_CN&countrycode=CN&exportkey=n_ChQIAhIQCjilua3TDGVxyiuSYvEHVhLmAQIE97dBBAEAAAAAALAWOAhzKrgAAAAOpnltbLcz9gKNyK89dVj0CTIbchQ6j1Rbwg9dIotsc8CZEP9ixoggCOoyz0MPRHsr1Fa3NA06GDkytkAh8TtVQIMbK9CI4PwvS6emWvH5ztFp2ipIrspx2c1MpoEPlLNErNBhFCh%2FasYLQlRfIhn77cmhr5qL8xfZQGppj83yO44caF3cMuK2Hf%2FGBcm0RvMJaSwupp2UXqnzO0UdJRF4w%2FzTJSY4NVlk1%2FADqcbrsnPgj3synhjTb37NQm72N2eHrbhAqirPxEXZJZc8SMA%2F&acctmode=0&pass_ticket=vaYHjiko6XbDe%2F8DEbTQqoZOnM9Exo%2BHAsZh5sHz6V9be%2BFPnfQrSaVMLsT%2BZoWl&wx_header=1&fasttmpl_type=0&fasttmpl_fullversion=7351805-zh_CN-zip&fasttmpl_flag=1#)

学习 Rust 之前，在知乎等平台也看到过一些回答，认为 Rust 学习曲线陡峭、难学，个人觉得如果有些 C/C++ 的基础其实学起来也还好，只不过 Rust 有很多独有的概念，这一点是和现有很多主流语言是不同的，需要花点时间看下。  

![](http://mmbiz.qpic.cn/mmbiz_png/jQmwTIFl1V0dLQzNJW15CVaCoNjposvTpccciaj05o5nPiaqfLRRfTQiaYFYPN41Etrrqt8jPOWukPmJWt3lYxwuA/300?wx_fmt=png&wxfrom=19)

**全栈修仙之路**

专注分享 TS、Vue3、前端架构和源码解析等技术干货。

234篇原创内容

公众号

## 脑图

一个脑图概括本文所有知识点。

![](https://mmbiz.qpic.cn/mmbiz_png/wnIMIiaEIIrj5yib6AmIftfjBUcEKh4KGC153l9vTmiaLytgfVn5JBLiaV1xB5T7Idaevdr49pKKMSrcUU5K1lPt1g/640?wx_fmt=png&wxfrom=13&tp=wxpic)

## 发展历史

Rust 语言是 Mozilla 员工 Craydon Hoare 在 2006 年创建的一个业余项目，2012 年 Mozilla 宣布推出基于 Rust 语言开发的以**内存安全性**、 **并发性**为首要原则的新浏览器引擎 Servo 这也是其首个完整的大型项目。

2015 年发布首个 Rust v1.0 版本，这是第一个重要的里程碑，近期在过去的 2020 年由于疫情原因，Mozilla 宣布裁员，涉及到一些 Rust 项目和 Rust 社区中的活跃成员，这对外界对于 Rust 猜测又增加了更多不确定性。

今年的 2 月 9 号，Rust 基金会 https://foundation.rust-lang.org/ 宣布成立，从 Mozilla 脱离出来，目前的基金会董事成员包括：亚马逊、Google、微软、Mozilla 和国内的华为，由五大科技巨头支持，对 Rust 来说总归是好事，可以为这门语言促进更好的发展，也有着更好的前景。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

## 特点

- **类型推断**：Rust 提供了强大的类型推断功能，我们可以使用 `let a = 1;` 声明一个变量，看似给 JavaScript 一样的，Rust 中类型推断的结果可能是这样的 `let a: i32 = 1;`
    
- **内存安全**：也许你已经听过了Rust 这门语言无需 GC，这也是其与现有其它语言不同的地方，即不需要像 C/C++ 一样手动申请内存、释放内存，也不需要像 Java、Go 这样有垃圾回收的语言等待系统回收，这些还是少不了一个概念**所有权。**
    
- **线程安全**：之前谈及多线程大家经常想到的一个问题通常是数据竞争，也就是多个线程访问同一变量做一些写操作时，通常会引起一些线程安全问题，在 Rust 里有一个概念**所有权**，所有权系统会将不同对象的所有者传输到不同的线程，这里面还有一些作用域的概念，多个线程不可能同时对同一个变量持有写权限操作。
    
- **范型支持**：范型是一个编程语言核心的机制了，C 语言是没有范型的而 C++ 也是通过模版实现，编译器在调用模版时自动进行类型推导，Rust 中当我们定义一个函数，如果类型存在多种情况下，即可通过范型定义，除了函数中使用之外还可以在方法、结构体和枚举中定义范型。
    
- **模式匹配**：提供的强大的模式匹配功能与 match 表达式搭配使用，可以更好的控制程序的控制流，单值匹配、多值匹配和范围匹配都可实现。
    
- ...
    

## 安装

### 在线尝试

如果你不想在本地电脑上安装，想尽快尝试下 Rust 可通过其提供的在线代码编辑器。

**https://play.rust-lang.org**

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

### 类 Unix 系统

如果使用 MacOS、Linux 或其它的类 Unix 系统，可以下载 `Rustup` 安装 Rust，`Rustup` 即是一个 Rust 安装器又是一个版本管理工具。

终端运行如下命令，根据提示完成即可。

`$ curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh   `

- Rustup 更新
    

Rust 目前的升级很频繁，如果你已安装很长一段时间，可通过 `rustup update` 命令更新最新版本。

`$ rustup update   `

- 验证安装结果
    

`$ rustc --version   rustc 1.50.0 (cb75ad5db 2021-02-10)   `

- 环境变量
    

Rust 的所有工具存在于 `~/.cargo/bin` 目录下，正常情况下安装时会配置环境变量，但是由于不同平台、shell 之间存在的差异，可能会存在一些问题，导致在终端未重启或用户未重新登陆之前，`rustup` 对环境变量的修改不生效，如果存在问题 `rustc --version` 命令就会执行失败。

- 卸载 Rust
    

`$ rustup self uninstall   `

- reference
    

- https://github.com/rust-lang/rustup
    
- https://www.rust-lang.org/zh-CN/tools/install
    

### Windows 系统

在 Windows 平台上，通过下载可执行应用程序 rustup-init.exe 安装。

- reference
    

- other-ways-to-install-rustup
    

## 编辑器

Rust 支持多种编辑器 VS CODE、SUBLIME TEXT 3、ATOM、INTELLIJ IDEA、ECLIPSE、VIM、EMACS、GEANY。

以笔者常用的 VS CODE 做一个介绍。

推荐两个 Rust 的 VS CODE 插件: rust-analyzer、rust。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

## Hello Rust！

创建一个项目 `cargo new hello-rust`

查看目录结构 `tree -a`

`├── .gitignore   ├── Cargo.toml   └── src       └── main.rs   `

看下 `Cargo.toml` 的内容，这个类似于 Node.js 中的 package.json 声明了项目所需的信息，对于 Rust 项目来说就是声明了 `Cargo` 编译程序包所需的元数据，以 `.toml` 文件格式编写。

TOML 一种新的配置文件格式。

`[package]   name = "hello-rust"   version = "0.1.0"   authors = ["五月君 <qzfweb@gmail.com>"]   edition = "2018"      # See more keys and their definitions at https://doc.rust-lang.org/cargo/reference/manifest.html      [dependencies]   `

主程序 main.rs 使用 function 的简写 `fn` 声明了一个函数，注意，里面 **println! 后面加上了符号 `!` 并不是一个函数，而是一个宏**。

`fn main() {       println!("Hello, rust!");   }   `

**编译我们的 Rust 项目**，之后会在 target/debug/ 目录下生成编译好的文件，如下所示：

`$ cargo build   $ target/debug/hello-rust   Hello, rust!   `

**Rust 的 Release 编译模式**，如下所示：

`$ cargo build --release   $ target/release/hello-rust    Hello, rust!   `

开发阶段每次编译再执行，你可能会感觉很麻烦，**Rust 允许我们使用 `cargo run` 直接编译运行**，该命令会自动帮我们做编译操作，如下所示：

``$ cargo run      Compiling hello-rust v0.1.0 (/Users/xxxx/study/hello-rust)       Finished dev [unoptimized + debuginfo] target(s) in 1.00s        Running `target/debug/hello-rust`   Hello, rust!   ``

## 数据类型

### 变量

Rust 使用 `let` 声明一个变量，通常来说变量将是可变的，但是在 **Rust 中默认设置的变量是预设不可变的**，这也是 Rust 推动你能充分利用其提供的安全性来写程序的方式之一，Rust 中鼓励你多多使用不可变的，当然如果你明确知道该变量是可变得，也是可以的。

以下我们声明的变量 num 并没有明确其类型，这也是 Rust 的特性之一**类型推断**

`fn main() {       let num = 1;       num = 2;       println!("num {}", num);   }   `

运行之后会得到一个 `cannot assign twice to immutable variable 'num'` 错误，这在编译器阶段就是不会通过的。

在变量名称前加上 `mut` 关键字，表明该变量是可变的。

`fn main() {       let mut num = 1;       println!("num {}", num);       num = 2;       println!("num {}", num);   }   `

### 常量

常量使用 const 声明，之后是不可变的，在声明时必须指定变量类型，这也是与 let 的不同，还需注意的是常量名称一定要大写，否则编译阶段也是会报错的。

`fn main() {       const NUM: i8 = 1;       println!("num {}", NUM);   }   `

### 作用域

一个变量只有在其作用域内是生效的。下例，变量 y 在花括号内即它的块级作用域内是有效的，当离开花括号如果想在外部打印，会报 **cannot find value `y` in this scope** 错误。

`fn main() {       let x = 1;       {           let y = 2; // y 在此处开始有效           println!("y {}", y);       } // 此作用域结束，y 不再有效       println!("x {} y {}", x, y);       // println!("x {}", x);   }   `

### 基本数据类型

Rust 是一个静态数据类型的语言，这意味着在编译时它就要知道变量的类型。

Rust 包含四种基本数据类型分别为：整形、浮点型、布尔型、字符型。

#### 整型

Rust 里的整型又分为带符号的整型（signed）和非带符号整型（unsigned），两者之间的区别是数字是否是负数。带符号整型的安全存储范围为  到 ，n 就是下面的长度。

非带符号整型的安全存储范围为 0 到 。isize 和 usize 是根据系统架构决定的，例如带符号整型，如果系统是 64 位，类型为 i64，如果系统是 32 位，类型为 i32。

|长度|带符号整型|非带符号整型|
|---|---|---|
|8-bit|i8|u8|
|16-bit|i16|u16|
|32-bit|i32|u32|
|64-bit|i64|u64|
|128-bit|i128|u128|
|系统架构|isize|usize|

#### 浮点型

Rust 的浮点型提供了两种数据类型 f32、f64，分别表示为 32 位与 64 位，默认情况下是 64 位。

`fn main() {       let x = 2.0; // f64       let y: f32 = 3.0; // f32       println!("x: {}, y: {}", x, y); // x: 2, y: 3   }   `

#### 布尔型

和大多数编程语言一样，Rust 中的布尔型包含两个值：true 和 false。

`fn main() {       let x = true; // bool       let y: bool = false; // bool   }   `

#### 字符型

Rust 中的字符型为一个 Unicode 码，大小为 4 bytes，char 类型使用单引号包括。

`fn main() {       let x = 'x';       let y = '😊';       println!("x: {}, y: {}", x, y); // x: x, y: 😊   }   `

### 复合类型

复合类型可以组合多个数值为一个类别，复合类型包含两种：元组（tuples）和数组（arrays）。

#### 元组

元组是将多个不同数值组合为一个复合类型的常见方法，元组拥有固定长度，一旦声明无法更改。

我们通过**解构**的方式，分别从声明的元组中取出数据，如下例所示：

`fn main() {       let tup: (i32, f64, char) = (1, 1.01, '😊');       let (x, y, z) = tup;       println!("x: {}, y: {}, z: {}", x, y, z); // x: 1, y: 1.01, z: 😊   }   `

除此之外我们还可通过数值的索引来访问元组中的数据。

`fn main() {       let tup: (i32, f64, char) = (1, 1.01, '😊');       let x = tup.0;       let y = tup.1;       let z = tup.2;       println!("x: {}, y: {}, z: {}", x, y, z); // x: 1, y: 1.01, z: 😊   }   `

#### 数组

与元组不同的是数组中的所有元素类型必须一致，Rust 中的 Array 与其它语言不太一样，因为其 Array 的长度是固定的和元组一样。

`fn main() {       let a: [i32; 5] = [1, 2, 3, 4, 5];       println!("a[0]: {}, a[4]: {}", a[0], a[1]); // a[0]: 1, a[4]: 2   }   `

## 流程控制

### if 表达式

Rust 中的 if 语句必须接收一个布尔值，不像 JavaScript 这门语言会自动转换，还可以省略括号。

`fn main() {       let number = 1;       if number < 2 {           println!("true"); // true       } else {           println!("false");       }   }   `

如果预期不是一个布尔值，编译阶段就会报错。

`fn main() {       let number = 1;       if number {           println!("true");       }   }   `

运行之后，报错如下：

``cargo run      Compiling hello-rust v0.1.0 (/Users/xxxxx/study/hello-rust)   error[E0308]: mismatched types    --> src/main.rs:3:8     |   3 |     if number {     |        ^^^^^^ expected `bool`, found integer      error: aborting due to previous error   ``

在 let 中使用 if 表达式，注意 if else 分支的数据类型要一致，因为 Rust 是静态类型，需要在编译期间确定所有的类型。

`fn main() {       let condition = true;       let num = if condition { 1 } else { 0 };       println!("num: {}", num); // 1   }   `

### loop 循环

loop 表达式会无限的循环执行代码块，如果想终止循环，可配合 break 语句使用。

`fn main() {       let mut counter = 0;       let result = loop {           counter += 1;           if counter == 10 {               break counter * 2;           }       };          println!("result: {}", result); // 20   }   `

### while 循环

使用 while 可以加上条件判断决定是否还要循环多少次，如果条件为 true 继续循环，条件为 false 则退出循环。

`fn main() {       let mut counter = 3;       while counter != 0 {           println!("counter: {}", counter);           counter -= 1;       }       println!("end");   }   `

### for 循环

使用 for 循环遍历集合元素，例如在访问一个数组时，增加了程序的安全性不会出现超出数组大小或读取长度不足的情况。

`fn main() {       let arr = ['a', 'b', 'c'];       for element in arr.iter() {           println!("element: {}", element);       }       println!("end");   }   `

在 Rust 中使用 for 循环的另一种方式。

`fn main() {       for number in (1..4).rev() {           println!("number：{}", number);       }       println!("end");   }   `

## 结构体/函数/方法/枚举

### 函数

在 Rust 代码中函数随处可见，例如我们使用的 main 函数，关于函数的几个特点总结如下：

- 使用 fn 关键字声明。
    
- 函数参数必须定义类型。
    
- 箭头 `->` 后声明返回类型，默认情况下返回最后一个表达式，注意不要有分号 `;`
    
- 也可使用 return 返回，这里要加分号 `;`。
    

`fn main() {       let res1 = multiply(2, 3);       let res2 = add(2, 3);       print!("multiply {}, add {} \n", res1, res2);   }   fn multiply(x: i32, y: i32) -> i32 {       x * y   }   fn add(x: i32, y: i32) -> i32 {       return x + y;   }   `

### 结构体

结构体是一种自定义数据类型，它由一系列属性组成（这个属性拥有自己的属性和值），结构体是数据的集合，就像面向对象编程语言中一个无方法的轻量级类，因为 Rust 本身不是一门面向对象的语言，合理的使用结构体可以使我们的程序更加的结构化。

#### 定义一个结构体

使用 struct 关键字定义一个结构体，创建一个结构体实例也很简单，如下例所示：

`struct User {       username: String,       age: i32   }   fn main() {       // 创建结构体实例 user1       let user1 = User {           username: String::from("五月君"),           age: 18       };       print!("我是: {}, 永远 {}\n", user1.username, user1.age); // 我是: 五月君, 永远 18   }   `

### 方法

方法与函数类似，使用 fn 关键字声明，拥有参数和返回值，不同的是**方法在结构体的上下文中定义，方法的第一个参数始终为 `self` 表示调用该方法的结构体实例**。

改写上面的结构体示例，在结构体 User 上定义一个方法 info 打印信息，这里用到一个关键字 `impl`，它是 implementation 的缩写。

`struct User {       username: String,       age: i32   }   impl User {       fn info(self) {           print!("我是: {}, 永远 {}\n", self.username, self.age);       }   }   fn main() {       let user1 = User {           username: String::from("五月君"),           age: 18       };       user1.info();   }   `

### 枚举

- 简单的枚举
    

`enum Language {       Go,       Rust,       JavaScript,   }   `

- 元组结构体枚举
    

`#[derive(Debug)]   enum OpenJS {       Nodejs,       React   }   enum Language {       JavaScript(OpenJS),   }   `

- 结构体枚举
    

`#[derive(Debug)]   enum IpAddrKind {       V4,       V6,   }   #[derive(Debug)]   struct IpAddr {       kind: IpAddrKind,       address: String,   }   fn main() {       let home = IpAddr {           kind: IpAddrKind::V4,           address: String::from("127.0.0.1"),       };       let loopback = IpAddr {           kind: IpAddrKind::V6,           address: String::from("::1"),       };       println!("{:#?} \n {:#?} \n", home, loopback);   }   `

## 模式匹配

Rust 提供的匹配模式允许将一个值与一系列的模式比较，并根据匹配的模式执行相应的代码块，使用表达式 match 表示。

### 定义 match 匹配模式示例

举一个例子，我们可以定义一个 Language 枚举，代表编程语言，之后定义一个函数 get_url_by_language 根据语言获取一个对应的地址，match 表达式的结果就是这个函数的结果。看起来有点像 if 表达式，但是 if 只能返回 true 或 false，match 表达式可以返回任何类型。

这个示例分为三个小知识点：

- 如果 `Go` 匹配，因为这个分支我们**仅需要返回一个值，可以不使用大括号**。
    
- 如果 `Rust` 匹配，这次我们**需要在分支中执行多行代码，可以使用大括号**。
    
- 如果 `JavaScript` 匹配，这次我们**想对匹配的模式绑定一个值，可以修改枚举的一个成员来存放数据，这种模式称为绑定值的模式**。
    

`#[derive(Debug)]   enum OpenJS {       Nodejs,       React   }   enum Language {       Go,       Rust,       JavaScript(OpenJS),   }      fn get_url_by_language (language: Language) -> String {       match language {           Language::Go => String::from("https://golang.org/"),           Language::Rust => {               println!("We are learning Rust.");               String::from("https://www.rust-lang.org/")           },           Language::JavaScript(value) => {               println!("Openjs value {:?}!", value);               String::from("https://openjsf.org/")           },       }   }      fn main() {       print!("{}\n", get_url_by_language(Language::JavaScript(OpenJS::Nodejs)));       print!("{}\n", get_url_by_language(Language::JavaScript(OpenJS::React)));       print!("{}\n", get_url_by_language(Language::Go));       print!("{}\n", get_url_by_language(Language::Rust));   }   `

### 匹配 Option 与 Some(value)

Option是 Rust 系统定义的一个枚举类型，它有两个变量：None 表示失败、Some(value) 是元组结构体，封装了一个范型类型的值 value。

`fn something(num: Option<i32>) -> Option<i32> {       match num {           None => None,           Some(value) => Some(value + 1),       }   }   fn main() {       let five = Some(5);       let six = something(five);       let none = something(None);          println!("{:?} {:?}", six, none);   }   `

Rust 匹配模式还有一个概念**匹配是穷尽的，**上例中 `None => None` 是必须写的，否则会报 `pattern None not covered` 错误，编译阶段就不会通过的。

### 一个简单的示例，看懂 Rust 多种模式匹配

- 如果写一个固定的值，即单个值匹配。
    
- 使用  `|` 符号实现多值匹配。
    
- 使用 `..=` 符号实现范围匹配，注意，之前是使用 `...` 现在该方式已废弃。
    
- `_` 符号是匹配穷进行，Rust 要检查所有被覆盖的情况。
    

`fn main() {       let week_day = 0;       match week_day {           1 ..= 4 => println!("周一至周四过的好慢啊..."),           5 => println!("哇！今天周五啦！"),           6 | 0 => println!("这两天是周末，休息啦！"),           _ => println!("每周只有 7 天，请输入正确的值...")       };   }   `

### if let 简单控制流

我们想仅在 `Some(value)` 匹配时做些处理，其它情况不想考虑，为了满足 `match` 表达式穷进性的要求，还要在写上 `_ => ()` 来匹配其它的情况，类似代码多了显然繁琐。

`fn main() {       let five = Some(5);       match five {           Some(value) => println!("{}", value),           _ => ()       }   }   `

只针对一种模式做匹配处理的场景下，可以使用 if let 语法，可以更少的代码来写，如下所示：

`fn main() {       let five = Some(5);       if let Some(value) = five {           println!("{}", value + 1);       }   }   `

## 所有权

### 什么是所有权？

所有权是 Rust 的核心特色之一，它让 Rust 无需垃圾回收即可保证内存安全。我们先记住这样一句话：**在 Rust 里每一个值都有一个唯一的所有者，如果当我们对这个值做一个赋值操作，那么这个值的所有权也将发生转移，当所有者离开作用域，这个值也会随之被销毁**。

对许多人来说这是一个全新的概念，在接下来我们慢慢来了解它。

### 内存管理方式

也不得不说下目前的内存管理方式， **一类是类似于 C 语言这样的需要手动分配、释放内存**，在 C 语言中可以使用 malloc/free 这两个函数，手动管理内存如果出现遗漏释放、过早释放、重复释放等这些问题也是最容易制造 Bug 的。**另一类是类似于 JavaScript、Java 这类的高级语言由垃圾回收机制自动管理内存, **你不需要担心有什么内容会忘记释放，也不需要担心过早的释放。

Rust 采用了第三种方式，通过所有权系统管理内存，编译器在编译时会根据一系列规则做检查，如果出现错误，例如堆上的一个值其所在的变量 x 被赋值给一个新的变量 y，如果此后的程序还在使用 x，就会报错，因为一个值只有一个所有者，下文 "复杂数据类型 — 移动" 这个小节会讲到这个示例。

### Rust 内存分配与自动释放

基本数据类型，类似于 i32、char 这些的长度都是固定已知的，程序可以轻松的分配一定的内存给它，且它们都是存储在栈上在离开所在的作用域时也会被移除栈。对于一些复杂的数据类型，例如 String 它的长度在编写时是未知的，成勋运行过程中是有可能改变它的长度的，这个类型就存储在堆上。

类似 String 类型的数据它的过程是这样的：

- 第一步：运行过程中向操作系统申请内存。
    
- 第二步：当程序处理完 String 类型时将内存返还给操作系统。
    

看一段小示例：

`fn main() {       let s1 = String::from("hello");       print!("{}", s1); // hello   }   `

如下图所示， **左侧是存储在栈上的数据，ptr 指向存放字符串内容的指针，len 是 s1 内容使用了多少字节的内存，capacity 是容量表示从操作系统获取了多少字节的内存**。**右侧是存储在堆上的数据**。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

当我们执行 `String::from("hello")` 时，这是第一步操作，实现请求所需的内存。在变量 s1 离开作用域后会被自动释放掉，这是第二步操作，但这块不需要开发者手动操作，Rust 会为我们调用一个特殊的函数 drop，在这里 String 的作者可以放置释放内存的代码。Rust 在结尾的 `}` 处自动调用 drop。

### 基本数据类型 — 赋值

声明基本数据类型 x 将其赋值给变量 y，由于这是一个已知固定大小的值，因此被放入了栈中，赋值的过程也是一个拷贝的过程，现在栈中既有 x 也有 y，程序是可以正常执行的。

`fn main() {       let x: i32 = 5;       let y: i32 = x;       print!("x {}, y {}", x, y); // x 5, y 5   }   `

### 复杂数据类型 — 移动

接下来让我们看一个复杂数据类型的赋值，和上面的示例类似，不过这次使用的 String 类型。

`fn main() {       let s1 = String::from("hello");       let s2 = s1;       print!("s1 {}, s2 {}", s1, s2);   }   `

运行之后，报错如下：

``$ cargo run      Compiling hello-rust v0.1.0 (/Users/xxx/study/hello-rust)   error[E0382]: borrow of moved value: `s1`    --> src/main.rs:4:28     |   2 |     let s1 = String::from("hello");     |         -- move occurs because `s1` has type `String`, which does not implement the `Copy` trait   3 |     let s2 = s1;     |              -- value moved here   4 |     print!("s1 {}, s2 {}", s1, s2);     |                            ^^ value borrowed here after move   ``

String 类型的数据存储在堆上，那么赋值时也并不会在堆上拷贝一份，如果真的是拷贝在运行时过程中一旦堆上数据比较大这对性能的影响也是很大的。

为了确保安全，Rust 在这种场景下有一个值得注意的细节，**当尝试拷贝被分配的内存时，Rust 会使第一个变量无效，这个过程在 Rust 中称为移动，**可以看作 s1 被移动到了 s2 当 s2 离开作用域时，它就会自动释放自己的内存，这里也再次证明一点，在 Rust 同一时间每一个值仅有一个所有者。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

### 复杂数据类型 — 拷贝

基本数据类型存储在栈上，赋值就是一个拷贝的过程，在堆上的数据当你需要拷贝时，也是可以通过一个 clone 的通用函数实现的。

`fn main() {       let s1 = String::from("hello");       let s2 = s1.clone();       print!("s1 {}, s2 {}", s1, s2); // s1 hello, s2 hello   }   `

### 所有权与函数

将值传递给函数与变量赋值类似，值的所有权会发生变化，如下示例所示，外层 s1 处是会报错的，这在编译阶段会报 **borrow of moved value: `s1`** 错误。

`fn main() {       let s1 = String::from("hello"); // s1 进入作用域       dosomething(s1); // s1 的值移动到函数 dosomething 里       print!("s1: {}", s1); // s1 在这里就不再有效了   }   fn dosomething(s: String) { // s 进入作用域       print!("dosomething->s: {}", s);   }// s 离开作用域会被自动释放掉   `

解决这个问题的一个办法是转移函数返回值的所有权。

`fn main() {       let s1 = String::from("hello");       let s = dosomething(s1);       print!("s1: {} \n", s);   }   fn dosomething(s: String) -> String {       print!("dosomething->s: {} \n", s);       s   }   `

但是这样实现起来难免啰嗦，还可以通过 **引用** 简单的实现。

### 所有权与引用、借用

#### 引用

符号 **&** 表示引用，&s1 为我们创建一个指向值 s1 的引用，但并不拥有它，所有权没有发生转移。

`fn main() {       let s1 = String::from("hello");       dosomething(&s1);       print!("s1: {} \n", s1); // s1: hello   }   fn dosomething(s: &String) {       print!("dosomething->s: {} \n", s);   } // s 离开作用域，但其没有拥有引用值的所有权，这里也不会发生什么...   `

#### 借用

引用也许还可以理解，那么借用又是什么呢？在 Rust 中我们获取引用做为函数的参数称为 **借用，这里就需要注意了，预设变量默认是不可变的，想修改引用的值还需使用可变引用，在特定作用域中数据有且只有一个可变引用，好处是在编译时即可避免数据竞争。**

`fn main() {       let mut s1 = String::from("hello");       dosomething(&mut s1);       print!("s1: {} \n", s1); // s1: hello 五月君   }   fn dosomething(s: &mut String) {       s.push_str(" 五月君");       print!("dosomething->s: {} \n", s); // dosomething->s: hello 五月君   }   `

## 范型

范型是对具体类型的一种抽象，常用在强类型编程语言中，高效的处理重复概念。例如我们定义一个函数，参数可能有多种类型的值传递，那么就不能用具体的类型来声明，可以在编写代码时使用范型来指定类型，在实例化时做为参数指明这些类型。

### 在函数中定义范型

#### 一个比较大小的范型示例

为了开启比较功能，我们要使用到 trial 标准库中定义的 `std::cmp::PartialOrd`

`fn largest<T: std::cmp::PartialOrd>(a: T, b: T) -> T {       if a > b {           a       } else {            b       }   }   fn main() {       let res1 = largest(1, 2);       let res2 = largest(1.1, 2.1);          print!("res1: {}, res2: {} \n", res1, res2);   }   `

#### 两个数相加的范型示例

`fn add<T: std::ops::Add>(a: T, b: T) -> <T>::Output {       a + b   }   fn main() {       let res1 = add(1, 2);       let res2 = add(1.1, 2.1);          print!("res1: {}, res2: {}", res1, res2);   }   `

### 在结构体中定义范型

我们通常使用 T 来标识一个范型，例如我们定义一个坐标结构体，x、y 可能同时是不同的类型，这样就需要定义两个范型参数。也要尽可能的避免在参数中定义太多的范型参数，这会让代码看起来很难阅读和理解。

`struct Point<T1, T2> {       x: T1,       y: T2   }      fn main() {       let res1 = Point { x:1, y: 2};       let res2 = Point { x:1.1, y: 2.1};       let res3 = Point { x:1, y: 2.1};   }   `

### 在方法中定义范型

必须在 impl 后声明范型 T，对应我们的示例因为有多个范型参数，所以就是 T1、T2。

`struct Point<T1, T2> {       x: T1,       y: T2   }   impl<T1, T2> Point<T1, T2> {       fn x(&self) -> &T1 { &self.x }       fn y(&self) -> &T2 { &self.y }   }   fn main() {       let res = Point { x:1, y: 2.1};       print!("res.x: {}, res.y: {} \n", res.x(), res.y());   }   `

### 在枚举中定义范型

Option 和 Result 这两个枚举都拥有范型，由 Rust 标准库提供。

`enum Option<T> {       Some(T),       None,   }   enum Result<T, E> {       Ok(T),       Err(E),   }   `

以 Resut 举例，如果我们读取一个环境变量，成功时触发 Ok 函数，失败时触发 Err 函数。

`fn main() {       match std::env::var("HOME") {           Ok(data) => print!("Data: {}", data),           Err(err) => print!("Error: {}", err)       }   }      fn main() {       match std::env::var("xxxx") {           Ok(data) => print!("Data: {}", data),           Err(err) => print!("Error: {}", err) // Error: environment variable not found       }   }   `

## trait — 定义共享的行为

trait 是告诉 Rust 编译器某种类型具有哪些可以与其它类型共享的功能，抽象的定义共享行为，简单来说就是**把方法的签名放在一起，定义实现某一目的所必须的一组行为**。与其它语言的接口（Interface）类似，也还有些区别。

### trait 定义

创建文件 person.rs 使用 trait 定义行为 Person，例如，每个人都有一个简单的介绍，在大括号内声明实现这个 trail 行为所需要的签名方法。

注意，**trait 定义的类型里只有方法签名，没有具体实现，每一个方法签名独占一行，以 `;` 结尾，pub 代表公用的，可被外部其它模块调用**。

`pub trait Person {     fn intro(&self) -> String;   }   `

### trait 实现

实现 trait 定义的行为，使用 impl 后跟自定义的 trait，for 后跟自定义的结构体，大括号内编写实现 Person 这个行为类型声明的方法签名，这点看起来和其它面向对象编程语言中的 Interface 相似。

format!() 是格式化为一个字符串返回。

`pub struct Worker {     pub name: String,     pub age: i32   }      impl Person for Worker {     fn intro(&self) -> String {       format!("My name is {}, age {}, is a worker", self.name, self.age)     }   }   `

在 main.rs 中引入 person.rs 模块。

mod 关键字用来加载需要引入的文件为我们的模块，use 表示加载 person 这个模块定义的 trait 和结构体。

`mod person;   use person::{ Person, Worker };   fn main() {       let may_jun = Worker {           name: String::from("五月君"),           age: 20       };       println!("{}", may_jun.intro());   }   `

### trait 默认实现

可以为 trait 中某些签名方法提供默认的行为，这样当某个特定的类型实现 trait 时，可以选择保留或重载签名方法提供的默认行为。

修改 person.rs，当为方法提供默认行为之后，无法在 Worker 中再次定义 intro 方法了，但是可以在 Worker 实例后调用。

`pub trait Person {     fn intro_author(&self) -> String;     fn profession(&self) -> String;     fn intro(&self) -> String {       format!("我的个人简介：{}, 职业 {}", self.intro_author(), self.profession())     }   }      pub struct Worker {     pub name: String,     pub age: i32   }      impl Person for Worker {     fn intro_author(&self) -> String {       format!("姓名: {} 年龄: {}", self.name, self.age)     }     fn profession(&self) -> String {         format!("打工人")     }   }   `

### trait 做为参数

把 trait 做为参数，例如定义 intro 方法，传入的 item 需要是实现了 Person 这个 trait 的参数类型，这样我们可以直接调用 Person 这个 trait 上定义的 intro() 方法。

`pub fn print_intro (item: impl Person) {     println!("pub fn print_intro {}", item.intro());   }   `

修改 main.rs 引入 person.rs 文件里定义的 print_intro() 执行时传入实现了 Person 这个 trait 的结构体类型。

`mod person;   use person::{ Person, Worker, print_intro };   fn main() {       let may_jun = Worker {           name: String::from("五月君"),           age: 20       };       println!("{}", may_jun.intro());       print_intro(may_jun);   }   `

### trait 做为返回值

可以定义实现了某个 trait 的类型为返回值，与上面的例子结合使用，修改 main.rs 文件。

`fn returns_intro () -> impl Person {       Worker {           name: String::from("五月君"),           age: 20       }   }   print_intro(returns_intro());   `

### Trait Bounds

Trait Bounds 适用于复杂的场景，例如方法参数中定义多个实现了 trait 的类型。

`pub fn print_intro (item1: impl Person, item2: impl Person) {}   `

Trait Bounds 与范型参数声明在一起，改写上面的例子如下所示：

`pub fn print_intro<T: Person> (item1: T, item2: T) {}   `

## 总结与思考

以上仅是关于 Rust 一些基础知识、独有概念的一些介绍，篇幅有限也还有很多内容没有写，**欢迎关注【五月君】后续也会继续分享**。

对 Rust 感兴趣的朋友推荐去看看《**The Rust Programming Language**》这本书，官方开源的 Rust 教程，写的挺好，也有中文版的翻译，想看视频的可以上 bilibili 搜索下 Rust 入门，有几个视频版本的讲解基本上也是参考的这本书，重点还是要**多思考、多实践**。

社区上会看到一些话题：“Rust 可能取代 C 语言吗？”，Rust 提供了系统编程的能力，从能力上来讲 C 的一些实现 Rust 也是有能力去做的，但是 C/C++ 这个地位是很难被取代的，生态已经很大了，也并不是所有的项目都需要重写，也是有成本在的生态、人员等也不是一蹴而就的。

编程语言只是工具，为我们实现某些业务或功能的编程工具，不要盲目互吹或黑某一门语言，例如某乎上经常看到的 “xxxx 年了，xxx 凉了吗？”，**多学习不同编程语言背后的设计思想、优势与劣势，磨练技艺、突破自我、适时选择**。

![](http://mmbiz.qpic.cn/mmbiz_png/jQmwTIFl1V0dLQzNJW15CVaCoNjposvTpccciaj05o5nPiaqfLRRfTQiaYFYPN41Etrrqt8jPOWukPmJWt3lYxwuA/300?wx_fmt=png&wxfrom=19)

**全栈修仙之路**

专注分享 TS、Vue3、前端架构和源码解析等技术干货。

234篇原创内容

公众号

## Reference

- 《The Rust Programming Language》中文翻译
    
- 《The Rust Programming Language》
    
- https://blog.rust-lang.org/2020/05/15/five-years-of-rust.html
    

阅读 1543

​

写留言

**留言 2**

- 穿山贾
    
    2021年11月30日
    
    赞
    
    没我还以为是 泛 型呢
    
    全栈修仙之路
    
    作者2021年11月30日
    
    赞
    
    泛型它也🈶
    

已无更多数据

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/jQmwTIFl1V0dLQzNJW15CVaCoNjposvTpccciaj05o5nPiaqfLRRfTQiaYFYPN41Etrrqt8jPOWukPmJWt3lYxwuA/300?wx_fmt=png&wxfrom=18)

全栈修仙之路

关注

413

2

写留言

**留言 2**

- 穿山贾
    
    2021年11月30日
    
    赞
    
    没我还以为是 泛 型呢
    
    全栈修仙之路
    
    作者2021年11月30日
    
    赞
    
    泛型它也🈶
    

已无更多数据