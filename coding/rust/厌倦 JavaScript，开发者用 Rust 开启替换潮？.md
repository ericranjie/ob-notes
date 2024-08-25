# 

玩转VS Code

 _2021年11月28日 12:10_

作者 | Lee Robinson  

译者 | 弯月  

出品 | CSDN（ID：CSDNnews）

Rust 是一种快速、可靠且内存使用效率非常高的编程语言，连续六年被评为最受欢迎的编程语言。Rust 由 Mozilla 创建，现已被 Facebook、苹果、亚马逊、微软和 Google 等科技大公司用于系统基础设施、加密、虚拟化以及其他底层的开发。

为什么如今人们利用 Rust 来替换 JavaScript 网络生态系统中的一些工具，比如压缩器（Terser）、转译（Babel）、格式化（Prettier）、打包（webpack）、linting （ESLint） 等等？

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/Pn4Sm0RsAuhLyzpo5mEoAkiaeYsWkC5bnrDI2fRj2q1fxE6k0glfXUVqX7iaUvNnkUyXbQcuMF3BK25icSefcic6jA/640?wx_fmt=jpeg&wxfrom=13&tp=wxpic)

  

  

# **Rust 是什么？**  

Rust 是一种可帮助开发人员编写高效使用内存的快速编程语言。它是可以替代 C++ 或 C 等的现代编程语言，注重代码的安全和简洁的语法。

Rust 与 JavaScript 完全不同。JavaScript 会设法查找未使用的变量或对象，并自动从内存中清除，也就是我们常说的垃圾收集。因此，JavaScript 开发人员无需考虑手动内存管理。

然而，Rust 开发人员可以更好地控制内存分配，同时又不会像 C++ 或 Go 那样痛苦。

Rust 使用了一种相对独特的内存管理方法，它吸收了内存“所有权”的思想。大致来说，Rust 会跟踪谁可以读写内存。它知道程序何时使用内存，并在不再需要时立即释放内存。它会在编译时强制检查有关内存的规则，因此运行时几乎不可能出现内存错误。开发人员无需手动跟踪内存，编译器会自动处理。

—— Discord

#   

  

# **采用**  

除了上面提到的公司之外，Rust 还被广泛用于流行的开源库，例如：

- Firecracker (AWS)
    
- Bottlerocket (AWS)
    
- Quiche (Cloudflare)
    
- Neqo (Mozilla)
    

Rust 使得我们团队事半功倍，采用 Rust 是我们做出的最佳决策之一。Rust 的优势不仅限于性能，而且还可以提高我们的效率，Rust 注重正确性，帮助我们解决了同步操作的复杂性。我们将系统复杂的不变量编码到类型系统中，并让编译器帮我们进行检查。

—— Dropbox

#   

  

# **从 JavaScript 到 Rust**  

JavaScript 是一种使用极其广泛的编程语言，可在所有带有 Web 浏览器的设备上运行。在过去的十年中，人们围绕 JavaScript 构建了一个庞大的生态系统：

- Webpack：开发人员希望将多个 JavaScript 文件打包成一个。
    
- Babel：开发人员希望支持旧浏览器，同时还能编写现代 JavaScript。
    
- Terser：开发人员希望尽量压缩生成的文件大小。
    
- Prettier：开发人员想要一个可自行决定的代码格式化程序。
    
- ESLint：开发人员希望在部署之前发现代码中的问题。
    

人们编写了数百万行 JavaScript 代码，修复了大量 bug，为当今 Web 应用程序的开发奠定了基础。所有这些工具都是用 JavaScript 或 TypeScript 编写的。虽然这些工作非常了不起，但利用 JavaScript 实现的优化已经达到了极限。因此，人们渴望一类新工具的出现，并大幅提高 Web 构建的性能。

#   

  

# **SWC**  

SWC 创建于 2017 年，是一个基于 Rust 的可扩展平台，旨在构建下一代的开发工具。Next.js、Parcel 和Deno 等工具以及 Vercel、字节跳动、腾讯、Shopify等公司都在使用该平台。

SWC 可作为编译、压缩、打包等工具，而且功能还在不断扩展。开发人员可以用它来执行代码转换（内置或自定义转换），这些转换都是通过 Next.js 等更高级别的工具进行的。

  

  

# **Deno**

Deno 创建于 2018 年，是一个简单、现代且安全的 JavaScript 和 TypeScript 运行时，是使用 V8 引擎和 Rust 构建的。Deno 由Node.js 的原作者编写，目标是取代 Node.js。虽然该运行时于 2018 年创建，但直到 2020 年5 月才发行第一版。

Deno 的 linter、代码格式化程序和文档生成器都是使用 SWC 构建的。

# 

  

# **esbuild**  

esbuild 创建于 2020 年 1 月，是一款JavaScript 打包器和压缩器，比使用 Go 编写的现有工具快 10～100 倍。

我在尝试创建一个构建工具，主要目标有两个：第一，打包 JavaScript、TypeScript 和 CSS；第二，向社区展示超快速的 JavaScript 构建工具。在我看来，目前的构建工具实在太慢了。

——Evan，esbuild 的创建者

在 esbuild 发布之前，使用 Go 和 Rust 等系统编程语言构建 JavaScript 工具的做法非常小众。我认为，esbuild 激发了更广泛的兴趣，人们纷纷开始尝试构建快速的开发工具。Evan 选择使用 Go，原因在于：

如果付出足够的努力，Rust 版本也可以达到非常快的运行速度。但从高层来看，使用 Go 更有趣。由于这是一个业余项目，因此我希望享受一定的乐趣。

——Evan，esbuild 的创建者

有人认为 Rust 的性能更好，但这两种语言都可以实现 Evan 影响社区的最初目标：

只需基本的优化，Rust 的性能就会远超手动调整的 Go 代码。使用 Rust 编写高效的程序非常容易，而使用 Go 则必须非常深入地研究。

—— Discord

#   

# **Rome**  

Rome 于 2020 年 8 月创建，是一个面向 JavaScript、TypeScript、HTML、JSON、Markdown 和 CSS 的 linter、编译器、打包器、测试运行器。主要目标是替换和统一整个前端开发的工具链。这款工具由 Sebastian 创建的，另外 Sebastian 还创建了 Babel。

为什么要重新编写呢？

如果想对 Babel 进行必要的修改，使其成为其他工具的坚实基础，则必须重写所有代码。这个架构来源于我在 2014 年学习解析器、AST 和编译器时做出的设计选择。

—— Sebastian

目前，Rome 是用 TypeScript 编写并在 Node.js 上运行。但他们正在用 Rust 重写。

#   

  

# **Rust + WebAssembly**  

  

WebAssembly（WASM）是一种可移植的底层语言，而 Rust 可以编译成 WASM。WASM在浏览器中运行，可与 JavaScript 互操作，如今所有的主流现代浏览器都支持 WASM。

WASM的速度远比 JS 快，但不及原生应用。根据我们的测试，如果将 Parcel 编译为 WASM，则运行速度比原生应用慢 10～20 倍。

—— Devon Govett

虽然 WASM 还不是完美的解决方案，但可以帮助开发人员创建极快的 Web 体验。如今，Rust 团队正在努力构建高质量且最尖端的 WASM 实现。这意味着，开发人员可以使用 WASM 编译 Web 应用，同时还可以享受 Rust 的性能优势。

该领域已浮现出一些早期的库和框架：

- Yew
    
- Percy
    
- Seed
    
- Sycamore
    
- Stork
    

这些基于 Rust 的 Web 框架都编译成了WASM，但它们的目标不是取代 JavaScript，而是与JavaScript 协同工作。现如今 Rust 正在朝着两个方向努力：加快现有 JavaScript 工具的运行速度；将 Web 应用编译成 WASM。

一路讨论下来，我们看到的都是 Rust。

#   

  

# **Rust 的缺点**  

Rust 的学习曲线很陡峭，抽象级别更靠近底层，超过了大多数 Web 开发人员所掌握的水平。

Rust使我们思考关系到系统编程的代码维度。让我们思考如何共享或复制内存。让我们思考真实但不太可能出现的极端情况，并确保这些情况都能得到恰当的处理。Rust 可以帮助我们编写无论从何种角度来看都非常高效的代码。

—— Tom MacWright 

此外，Rust 在网络社区中的使用仍然很小众。采用率并不高。尽管为了 JavaScript 的工具而学习 Rust 是一个入门障碍，但有趣的是，即便某个工具的代码不是很熟悉，但只要速度非常快就能赢得开发人员的青睐。速度制胜。

目前，有些服务很难找到相应的 Rust 库或框架，比如身份验证、数据库、支付等。但我认为，随着 Rust 和 WASM 的采用率上升，这些问题自然就能得到解决。然而，目前还没有达到这个水平。我们需要利用现有的 JavaScript 工具，帮助我们弥合差距，并逐步享受性能的提升。

  

# **JavaScript 工具的未来**  

我相信 Rust 是 JavaScript 工具的未来。从 Next.js 12 开始，我们逐步开始过渡，用 SWC 和 Rust 取代 Babel（转译）和Terser（压缩）。为什么？

- 可扩展性：SWC 可以作为一个 crate 在 Next.js 中使用，无需建立分叉库，也无需绕开设计约束。
    
- 性能：换成 SWC，就能够将 Next.js 的快速刷新提高约 3 倍，构建速度提高约 5 倍，此外还有更多优化仍在开发中。
    
- WebAssembly：Rust 对 WASM 的支持成为了支持所有平台和推广 Next.js 开发的关键。
    
- 社区：Rust 社区和生态系统非常优秀，且在不断增长中。
    

采用 SWC 的不仅限于 Next.js，还有：

- Deno 的 linter、代码格式化程序和文档生成器都是使用 SWC 构建的。
    
- Rome 正在用 Rust 重写，并计划使用 SWC。
    
- dprint 是使用 SWC 构建的，这款代码格式化工具的速度是 Prettier 的 30倍。
    
- Parcel 使用 SWC 将整体构建性能提高了 10 倍。
    

Parcel将 SWC 作为一个库使用。以前我们使用 Babel 的解析器和用 JS 编写的自定义转换。现在，我们使用 SWC 的解析器和 Rust 的自定义转换。这项工作包括全部的 hoisting 实现、依赖项集合等。涉及范围之广类似于在 SWC 之上构建 Deno。

—— Devon Govett

目前，Rust 还处于早期阶段，一些重要的功能尚在研究中：

- 插件：对于许多 JavaScript 开发人员来说，用 Rust 编写插件并不容易。同时，公开 JavaScript 的插件系统可能会导致性能的提升荡然无存。最终的解决方案尚未出现。
    
- 打包：swcpack是一个有趣的开发领域，SWC 利用它来代替 Webpack。这款工具仍在开发中，但前景非常看好。
    
- WebAssembly：如上所述，编写Rust 代码并编译成 WASM 的前景很诱人，但仍有很多工作需要努力。
    

总的来说，我相信在未来 1～2 年内 Rust 将对 JavaScript 生态系统产生重大影响。想象一下，Next.js 中使用的所有构建工具都是用 Rust 编写的，这样就能为你提供极致的性能。此外，Next.js 还可以作为静态二进制文件通过 NPM 下载。

我心目中理想的开发世界，不外乎与此。

#   

  

# **评论**  

##   

对此，网友也发表了不同的看法：

## **评论1**

回想当初写下第一行 JavaScript 代码，已经过去16年了，然而至今我仍然对这门语言深恶痛绝。JavaScript 是那么神秘莫测，在使用 TypeScript 的时候这种感觉有过之而无不及。浏览器不兼容的问题仍然很普遍（虽然不像以前那么痛苦）。流行的库也常常没有任何文档，我不得不到处搜索。遇到有些问题时，我遍寻无果，直到有一天看到 GitHub 上的某个议题，才解开了痛苦之源。更重要的是，每天都有人在为构建流水线添加更多令人困惑而又不透明的工具。各色的打包器、加载器、编译器、转译器以及平台等等各自为政，让人眼花缭乱。

因此，我非常渴望有一天能够完全摆脱 JavaScript，换成某个更健全的语言，更完善的构建工具。

## **评论2**

JavaScript 看上去像是人们仅花了 10 天做出来的东西，目的仅仅是为了填补 HTML 缺乏的交互性。如果 Brendan Eich 知道人们今天如此使用 JavaScript，这门语言肯定会以完全不同的面貌出现。

## **评论3**

我很高兴看到 JavaScript 的工具遵循了能用=>好用=>速度快的发展过程，这才是构建软件的正确方向。如果一开始就以速度快为目标，那肯定不会有现在的百花齐放的局面。

参考链接：

https://leerob.io/blog/rust

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)  

  

  

**推荐阅读：**

- [全宇宙首本 VS Code 中文书，来了！](http://mp.weixin.qq.com/s?__biz=MzU1NjgwNTExNQ==&mid=2247485774&idx=1&sn=fba4ad7f8ad9bf339f9190f1d5fb6662&chksm=fc3e37dacb49beccf20cb3d7230a176802ea9401d52ca77e7cab003c7ce3896ca49bfb61e4a8&scene=21#wechat_redirect)  
    
- [Code Runner for VS Code 突破 1000 万下载量！支持运行超过 40 种语言](http://mp.weixin.qq.com/s?__biz=MzU1NjgwNTExNQ==&mid=2247484584&idx=1&sn=436d47aea9b88a7fb0ec24d77f06d725&chksm=fc3e3a3ccb49b32a1be2d365f58423791140c670c3f78d68eeb6ba45d69aec6334408e4649b9&scene=21#wechat_redirect)
    
- [微软也爱 Python！VS Code Python 全新发布！Jupyter Notebook 原生支持终于来了！](http://mp.weixin.qq.com/s?__biz=MzU1NjgwNTExNQ==&mid=2247484345&idx=1&sn=f541c502d69c4520a7ed50a141bb3386&chksm=fc3e3d2dcb49b43b668ab46c888e1433a0c177b92dd5adc91fa82766f42bd3c20beb07a731a9&scene=21#wechat_redirect)  
    
- [微软也爱 Java！微软在 SpringOne 大会上宣布 Azure Spring Cloud 云服务！](http://mp.weixin.qq.com/s?__biz=MzU1NjgwNTExNQ==&mid=2247484340&idx=1&sn=6e03e88e42f326aeab6e8dfa535aad16&chksm=fc3e3d20cb49b4366c58a29a7c9cf27f795e9ac2014e89e46810c5fb6ec2ba4c7de7f5170c48&scene=21#wechat_redirect)
    
- [在微软（Microsoft）工作是怎样一番体验？](http://mp.weixin.qq.com/s?__biz=MzIwODE4Nzg2NQ==&mid=2650548708&idx=1&sn=389ac3e83d6c3cd7a54164581b0a3284&chksm=8f0e7152b879f844b80626296324dbf26ebefe82c48525d558617e6ef09a570bdb8d6033102f&scene=21#wechat_redirect)  
    
- [微软内推，长期有效](http://mp.weixin.qq.com/s?__biz=MzIwODE4Nzg2NQ==&mid=2650548833&idx=1&sn=91c40a5133435a9b9b3d989e03af69e1&chksm=8f0e72d7b879fbc1afd382a36600552da15d4e90c9515f8109c701866d39e2ee8c0fafab16e1&scene=21#wechat_redirect)
    
- [代码编辑器横评：为什么 VS Code 能拔得头筹](http://mp.weixin.qq.com/s?__biz=MzU1NjgwNTExNQ==&mid=2247483834&idx=1&sn=c836662c60bf035e726bb82345453d4f&chksm=fc3e3f2ecb49b638bb58eb39df7646b0b180d43008f6bf9e9fadcf9390f79e8f4e0f7b4d3ec2&scene=21#wechat_redirect)
    
- [知否知否，VS Code 不止开源](http://mp.weixin.qq.com/s?__biz=MzU1NjgwNTExNQ==&mid=2247483756&idx=1&sn=d4faca9d8f69b67e6e1833431531c831&chksm=fc3e3ff8cb49b6eebe63083a2080d9d304d38604fbb608544249f5dda3311760f7efa6367c35&scene=21#wechat_redirect)
    
- [那些年，我们一起追的 VS Code](http://mp.weixin.qq.com/s?__biz=MzU1NjgwNTExNQ==&mid=2247483798&idx=1&sn=751467ecb2fc58d6aa7824ff123ef8cb&chksm=fc3e3f02cb49b6148ac69c87e24972da9b016a8257b0144ab565c20c9eb755cb2d673f233b00&scene=21#wechat_redirect)
    

  

**玩转VS Code**

VS Code **·** 编程开发 **·** 业界资讯

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

rust4

javascript13

编程语言3

rust · 目录

上一篇Rust 审核团队“一夜之间”集体辞职：开源社区治理话题再被热议下一篇我的 JavaScript 比你的 Rust 更快

阅读 2654

​