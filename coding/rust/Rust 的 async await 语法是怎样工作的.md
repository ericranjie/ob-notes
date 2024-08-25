# 

原创 JmPotato 半际谈

 _2022年01月24日 21:53_

原文地址《Rust 的 async/await 语法是怎样工作的》[1]

从最开始的宏到现在的 Rust 关键字，距离 async/await 语法的 rfc[2] 被提出已经过去将近 4 年了。相比于回调地狱，或者类似 CPS[3]-Style 的铁索连环套娃（此处应有圣经传唱：一个 Monad 说白了不过就是自函子范畴上的一个幺半群而已），async/await 的存在无疑提供了一种良好的异步代码编写方式，它更像是把同步代码写法的异步化，让代码编写者能够最大限度的遵循同步代码编写方式，但同时提供异步的运行时表现。

不过，有言道：”哪有什么岁月静好，不过是有人替你负重前行“。想要代码写的爽，编译器一定会在背后做很多”脏活累活“。Rust 的 async/await 语法具体是怎样工作的？它又是如何将我们写的代码，转化成异步执行的呢？

先来看一段代码。

`#[inline(never)]   async fn x() -> usize {       5   }   `

再简单不过的一个 async 函数，只会返回一个 5，为了防止被编译器优化掉，我们给它加上了一个 `#[inline(never)]` 属性。这个异步函数的等价同步代码长这样：

`#[inline(never)]   fn x() -> impl Future<Output = usize> {       async { 5 }   }   `

async fn 其实就是会返回一个 Future trait[4] 的函数。不过这一步转化并没有帮助我们更深地理解 async 关键字到底做了什么。为了一探究竟，我们可以尝试看看上述代码的 HIR[5] 长什么样。HIR 是 Rust 在编译过程中的一个中间产物，在转化成更为晦涩难懂的 MIR[6] 之前，它可以帮助我们一窥编译器的小小细节。

`cargo rustc -- -Z unpretty=hir   `

输出如下（为了方便展示，我做了一些格式上的调整）：

`#[inline(never)]   async fn x()    -> /*impl Trait*/ #[lang = "from_generator"](move |mut _task_context| { { let _t = { 5 }; _t } } "lang = "from_generator"")   `

此时我们终于看到了 Rust 中异步语义实现的核心：generator。不过上面这个函数的内容还是过于贫瘠了，甚至都没有涉及到今天文章的另一个主角 await。所以我们先在 `x()` 的基础上再加一个 `y()`。

`#[inline(never)]   async fn x() -> i32 {       5   }      async fn y() -> i32 {       x().await   }   `

`y()` 也是一个异步函数，它会在内部调用 `x().await`，即在 `x()` 返回结果前 block 住自己，不进行后续的操作。虽然在本例中 `x()` 并没有任何需要等待的操作，会直接返回 5，但在实际开发中，await 可能作用在各种各样的 Future 上，诸如锁的争用，网络 I/O 等，能够在此类操作不能被立马完成时提前返回并稍后再看也是异步编程的一个核心思想。此时我们再次输出 HIR，可以发现内容果然丰富了许多。

`#[inline(never)]   async fn x()    -> /*impl Trait*/ #[lang = "from_generator"](move |mut _task_context| { { let _t = { 5 }; _t } } "lang = "from_generator"")      async fn y()    -> /*impl Trait*/ #[lang = "from_generator"](move |mut _task_context|      {        {          let _t =          {            match #[lang = "into_future"](x( "lang = "into_future""))            {              mut pinned              =>              loop {                match unsafe                {                  #[lang = "poll"](#[lang = "new_unchecked"](&mut pinned),                  #[lang = "get_context"](_task_context "lang = "get_context""))              }              {                #[lang = "Ready"] {                  0: result                }                =>                break                result,                #[lang = "Pending"] {}                =>                {                }              }              _task_context              =              (yield                ());            },          }        };        _t      }   })   `

为了方便讲解，我尝试把上述代码转化成 Rust 伪代码：

`#[inline(never)]   fn x() -> impl Future<Output = usize> {       from_generator(move |mut _task_context| {           let _t = 5;           _t       })   }      fn y() -> impl Future<Output = usize> {       from_generator(move |mut _task_context| {           let mut pinned = into_future(x());           loop {               match unsafe {                   Pin::new_unchecked(&mut pinned).poll(_task_context.get_context());               } {                   Poll::Ready(result) => break result,                   Poll::Pending => {}               }               yield           }       })   }   `

可以看到整个转化主要干了两件事情：

- 把 async 块转化成一个由 `from_generator` 方法包裹的闭包
    
- 把 await 部分转化成一个循环，调用其 poll 方法获取 Future 的运行结果
    

这里的大部分操作还是比较符合直觉的：因为遇到了需要 await 完成的操作，所以运行一个循环去不停的获取结果，完成后再继续。注意到这里，当 x 所代表的 Future 还没有就绪时（即便在本例中并不会存在这种情况），loop 的运行会来到一个 yield 语句，而非 return。在开始阐述 generator 的 yield 之前，我们不妨先来思考一下，如果这里使用了 break 或 return，会有什么问题？

break 很好思考，loop 循环直接结束，如果 y 函数后续还有其它操作那么就会被执行——这显然不符合 await 的语义，我们需要 block 在当前的 Future 上，而不是忽略其结果继续运行后续代码。

那么 return 呢？如果这个 Future 暂时不能 await 出结果，那么我们为了应该尽快完成上层函数的 poll 操作，不 block 当前 Executor 对其他 Future 的执行，直接返回一个 Poll::Pending——到目前为止都没什么问题，但问题的关键在于，如果 `y()` 这个 Future 被 Waker 唤醒后，再次被 poll 的时候会发生什么？它会把 await 之前的所有代码都再运行一遍，这显然也不是我们想要的。不论是操作系统的线程还是 Future 这种用户态的 Task，我们想要的任务调度切换显然是需要有一个“断点续传”的基本能力。对于系统线程来说，我们知道操作系统进行线程调度时，会将上下文信息保存好，以遍后续线程再次被运行时可以通过上下文切换再次恢复运行时的状态。那么 Rust 的异步是怎么做到这一点的呢？答案就是 generator。

再来看一段代码：

`#![feature(generators, generator_trait)]      use std::ops::{Generator, GeneratorState};   use std::pin::Pin;      fn main() {       let mut generator = || {          let mut val = 1;           yield val;          val += 1;           yield val;          val += 1;           return val;       };          match Pin::new(&mut generator).resume(()) {           GeneratorState::Yielded(1) => {}           _ => panic!("unexpected value from resume"),       }       match Pin::new(&mut generator).resume(()) {           GeneratorState::Yielded(2) => {}           _ => panic!("unexpected value from resume"),       }       match Pin::new(&mut generator).resume(()) {           GeneratorState::Complete(3) => {}           _ => panic!("unexpected value from resume"),       }   }   `

可以看到 generator 拥有自己的状态，当你在通过调用 `resume()` 方法来推进其执行状态时，它不会从头来过，而是从上一次 yield 的地方继续向后执行，直到 return。上面的代码会被转换成类似下面的代码：

`#![feature(generators, generator_trait)]      use std::ops::{Generator, GeneratorState};   use std::pin::Pin;      fn main() {       let mut generator = {           enum MyGenerator {               Start,               Yield1(i32),               Yield2(i32),               Done,           }              impl Generator for MyGenerator {               type Yield = i32;               type Return = i32;                  fn resume(mut self: Pin<&mut Self>, _resume: ()) -> GeneratorState<Self::Yield, Self::Return> {                   match std::mem::replace(&mut *self, MyGenerator::Done) {                       MyGenerator::Start => {                           let val = 1;                           *self = MyGenerator::Yield1(val);                           GeneratorState::Yielded(val)                       }                          MyGenerator::Yield1(val) => {                           let new_val = val + 1;                           *self = MyGenerator::Yield2(new_val);                           GeneratorState::Yielded(new_val)                       }                          MyGenerator::Yield2(val) => {                           let new_val = val + 1;                           *self = MyGenerator::Done;                           GeneratorState::Complete(new_val)                       }                          MyGenerator::Done => {                           panic!("generator resumed after completion")                       }                   }               }           }              MyGenerator::Start       };                     match Pin::new(&mut generator).resume(()) {           GeneratorState::Yielded(1) => {}           _ => panic!("unexpected value from resume"),       }       match Pin::new(&mut generator).resume(()) {           GeneratorState::Yielded(2) => {}           _ => panic!("unexpected value from resume"),       }       match Pin::new(&mut generator).resume(()) {           GeneratorState::Complete(3) => {}           _ => panic!("unexpected value from resume"),       }   }   `

以上代码可以被正常编译通过，有兴趣的话可以到 Rust Playground[7] 亲自试一试。可以看到整体思路其实就是一个状态机，每次 yield 就是一次对 enum 实现的状态进行推进，直到最终状态被完成。过程中与状态相关的数据还会被存储到对应的枚举类型里，以遍下一次被推进时使用。你可能已经注意到一个 generator 的 `resume()` 方法和 Future 的 poll 似乎有几分神似——都要求方法的调用对象是 Pin 住的，且都会返回一个表示当前状态的枚举类型。那么回到我们最开始的 x 和 y 函数部分，对应的 generator 代码在接下来的 Rust 编译过程中，也正是会被变成一个状态机，来表示 Future 的推进状态。伪代码如下：

`struct GeneratorY {       state: i32,       task_context: Context<'static>,       future: dyn Future<Output = Vec<i32>>,   }      impl Generator for GeneratorY {       type Yield = ();       type Return = i32;          fn resume(mut self: Pin<&mut Self>, resume: ()) -> GeneratorState<Self::Yield, Self::Return> {           match self.state {               0 => {                   self.task_context = Context::new();                   self.future = into_future(x());                   self.state = 1;                   self.resume(resume)               }               1 => {                   let result = loop {                       if let Poll::Ready(result) =                           Pin::new_unchecked(self.future.get_mut()).poll(self.task_context)                       {                           break result;                       }                       return GeneratorState::Yielded(());                   };                   self.state = 2;                   GeneratorState::Complete(result)               }               _ => panic!("GeneratorY polled with an invalid state"),           }       }   }   `

可以看到每一个 Future 的本质其实都是一个 Generator，两者可以互相转换，例如 x 函数其实也是一个 Generator，它的实现会比 y 函数简单不少，毕竟只需要直接返回值，而没有额外需要 await 进行 yield 的状态。由于状态机本身就实现了 Future 方法，所以 into_future 也只是简单的进行了一个类型的转化，代码在这里[8]。具体的 Future trait 实现则在 from_generator 的过程中：

``/// Wrap a generator in a future.   ///   /// This function returns a `GenFuture` underneath, but hides it in `impl Trait` to give   /// better error messages (`impl Future` rather than `GenFuture<[closure.....]>`).   // This is `const` to avoid extra errors after we recover from `const async fn`   #[lang = "from_generator"]   #[doc(hidden)]   #[unstable(feature = "gen_future", issue = "50547")]   #[rustc_const_unstable(feature = "gen_future", issue = "50547")]   #[inline]   pub const fn from_generator<T>(gen: T) -> impl Future<Output = T::Return>   where       T: Generator<ResumeTy, Yield = ()>,   {       #[rustc_diagnostic_item = "gen_future"]       struct GenFuture<T: Generator<ResumeTy, Yield = ()>>(T);          // We rely on the fact that async/await futures are immovable in order to create       // self-referential borrows in the underlying generator.       impl<T: Generator<ResumeTy, Yield = ()>> !Unpin for GenFuture<T> {}          impl<T: Generator<ResumeTy, Yield = ()>> Future for GenFuture<T> {           type Output = T::Return;           fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {               // SAFETY: Safe because we're !Unpin + !Drop, and this is just a field projection.               let gen = unsafe { Pin::map_unchecked_mut(self, |s| &mut s.0) };                  // Resume the generator, turning the `&mut Context` into a `NonNull` raw pointer. The               // `.await` lowering will safely cast that back to a `&mut Context`.               match gen.resume(ResumeTy(NonNull::from(cx).cast::<Context<'static>>())) {                   GeneratorState::Yielded(()) => Poll::Pending,                   GeneratorState::Complete(x) => Poll::Ready(x),               }           }       }          GenFuture(gen)   }   ``

from_generator 的源代码[9]如上，可以看到 Future 转换成 Generator 后的 poll 的实现就等于进行一次 generator 的 resume，获得 `GeneratorState::Yielded` 即返回 `Poll::Pending`，获得 `GeneratorState::Complete(result)` 即返回 `Poll::Ready(result)` ，Context 则是作为 resume 的参数透传给状态机内部，整体逻辑还是非常清晰的。其中关于 Pin 的相关细节则是另一个比较繁杂的话题了，可以参考这篇博客进行学习：Rust 的 Pin 与 Unpin[10]。

# 参考

- Inside Rust's Async Transform[11]
    
- Generators and async/await[12]
    
- generators[13]
    
- How Rust optimizes async/await I[14]
    
- How Rust optimizes async/await II: Program analysis[15]
    
- Xuanwo's Note: Rust std/Future[16]
    

### 参考资料

[1]

原文地址《Rust 的 async/await 语法是怎样工作的》: _https://ipotato.me/article/70_

[2]

rfc: _https://github.com/rust-lang/rfcs/blob/master/text/2394-async_await.md_

[3]

CPS: _https://en.wikipedia.org/wiki/Continuation-passing_style_

[4]

Future trait: _https://docs.rs/rustc-std-workspace-std/latest/std/future/trait.Future.html_

[5]

HIR: _https://rustc-dev-guide.rust-lang.org/hir.html_

[6]

MIR: _https://rustc-dev-guide.rust-lang.org/mir/index.html_

[7]

Rust Playground: _https://play.rust-lang.org/?version=nightly&mode=debug&edition=2021&gist=49c5d4da4a94b7b8538457c3e4891ec2_

[8]

这里: _https://github.com/rust-lang/rust/blob/master/library/core/src/future/into_future.rs_

[9]

from_generator 的源代码: _https://github.com/rust-lang/rust/blob/42313dd29b3edb0ab453a0d43d12876ec7e48ce0/library/core/src/future/mod.rs#L65_

[10]

Rust 的 Pin 与 Unpin: _https://folyd.com/blog/rust-pin-unpin_

[11]

Inside Rust's Async Transform: _https://blag.nemo157.com/2018/12/09/inside-rusts-async-transform.html_

[12]

Generators and async/await: _https://cfsamson.github.io/books-futures-explained/4_generators_async_await.html#generators-and-asyncawait_

[13]

generators: _https://doc.rust-lang.org/beta/unstable-book/language-features/generators.html_

[14]

How Rust optimizes async/await I: _https://tmandry.gitlab.io/blog/posts/optimizing-await-1_

[15]

How Rust optimizes async/await II: Program analysis: _https://tmandry.gitlab.io/blog/posts/optimizing-await-2_

[16]

Xuanwo's Note: Rust std/Future: _https://note.xuanwo.io/#/page/rust%2Fstd%20future_

rust1

complier1

async1

阅读原文

阅读 2304

​