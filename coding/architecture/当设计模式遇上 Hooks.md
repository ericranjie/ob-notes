# 

原创 阿晨 阿里云开发者

 _2021年08月31日 07:58_

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

##   

## **一  前言**

「设计模式」是一个老生常谈的话题，但更多是集中在面向对象语言领域，如 C++，Java 等。前端领域对于设计模式的探讨热度并不是很高，很多人觉得对于 JavaScript 这种典型的面向过程的语言来说，设计模式的价值很难体现。之前我持有类似的观点，对于设计模式的理解仅停留在概念层面，没有深入去了解其在前端工程中的实践。近期阅读了《 JavaScript 设计模式与开发实践》一书，书中介绍了 15 种常见设计模式和基本的设计原则，以及如何使用 JavaScript 优雅实现并应用于实际工程中。碰巧前不久团队举行了一场关于 Hooks 的辩论赛，而 Hooks 的核心思想在于函数式编程，于是决定探究一下「设计模式是否有助于我们写出更优雅的 Hooks 」这一话题。

  

**二  为什么是设计模式**

在逆袭武侠剧中，主人公向第一位师父请教武艺时，最开始老师父只会让主人公挑水、扎马步等基本功，主人公这时总是会诸般抱怨，但又由于某些客观原因又不得不坚持，之后开始学习真正的武艺时才顿悟之前老师父的一番苦心，夯实基础后学习武艺突飞猛进，最终成为一代大侠。对于我们开发者来说，「数据结构」和「设计模式」就是老师父所教的基本功，它不一定能够让我们走得更快，但一定可以让我们走得更稳、更远，有助于我们写出高可靠且易于维护的代码，避免日后被 “挖坟”。

  

在 Hooks 发布以来，饱受诟病的一点就是维护成本激增，特别是对于成员能力水平差距较大的团队来说。即便一开始由经验老到的同学搭建整个项目框架，一旦交由新人维护一段时间后，大概率也会变得面目全非，更不用说让新人使用 Hooks 开发从零到一的工程。我理解这是由于 Hooks 的高度灵活性所导致的，Class Component 尚有一系列生命周期方法来约束，而 Hooks 除了 API 参数上的约束，也仅有 “只在最顶层使用 Hook” “只在 React 函数中调用 Hook” 两条强制规则。另一方面自定义 Hook 提高组件逻辑复用率的同时，也导致经验不足的开发者在抽象时缺少设计。Class Component 中对于逻辑的抽象通常会抽象为纯函数，而 Hooks 的封装则可能携带各种副作用（useEffect），出现 bug 时排查成本较大。

  

那么既然「设计模式」是一种基本功，而「Hooks」是一种新招式，那我们就尝试从设计模式出发，攻克新招式。

  

**三  有哪些经典的设计模式**  

  

在正式进入话题之前，我们先简单回顾一下那些快被我们遗忘的经典设计模式和设计原则。日常中，我们提到设计原则都会将其简化为「SOLID」，对应于单一职责原则（Single Responsibility Principle）、开放/封闭原则（Open Closed Principle）、里氏替代原则（Liskov Substitution Principle）、最小知道原则（Law of Demeter）、接口隔离原则（Interface Segregation Principle）和依赖倒置原则（Dependence Inversion Principle）。设计模式又包括了单例模式、策略模式、代理模式、迭代器模式、发布-订阅模式、命令模式、组合模式、模版方法模式、亨元模式、职责链模式、中介者模式、装饰者模式、状态模式、适配器模式等。

  

关于设计原则和设计模式有很多优秀的讲解文章，这里就不过多赘述了。

  

**四  1 + 1 > 2**

  

1  非得用 useContext 吗  

  

在 React Hook 工程中，一旦涉及到全局状态管理，我们的直觉会是使用 useContext。举个例子，假设工程中需要根据灰度接口返回的信息，决定某些组件是否进行渲染。由于整个工程共享一份灰度配置，我们很容易就想到将其作为一个全局状态，在工程初始化时调用异步接口获取并进行初始化，然后在组件内部使用 useContext 来获取。

  

```
// context.js
```

  

但是 createContext 的使用会造成一旦全局状态发生变更，Provider 下的所有组件都会进行重新渲染，哪怕它没有消费 context 下的任何信息。

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

仔细想想，这种场景和设计模式中的“发布-订阅模式”有着异曲同工之处，我们可以自己定义一个全局状态实例 GrayState，在 App 组件中初始化值，在子组件中订阅该实例的变化，也能够达到相同的效果，并且仅订阅了 GrayState 变化的组件会进行重新渲染。

  

```
// GrayState.js
```

  

最终实现的效果是一致的，不同的是获取灰度状态后，仅仅依赖灰度配置信息的 GrayComponent 进行了重新渲染。

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

考虑更好复用的话我们还可以将对 Status 监听的部分抽象为一个自定义 Hook：

  

```
// useStatus.js
```

  

当然，借助 redux 也是能够做到按需重新渲染，但如果项目中并没有大量全局状态的情况下，使用 redux 就显得有点杀鸡用牛刀了。 

  

2  useState 还是 useReducer

  

Hooks 初学者常常会感慨 “我开发中只用到 useStateuseEffect，其它钩子似乎不怎么需要的样子”。这种感慨源于对 Hooks 的理解还不够透彻。useCallbackuseMemo 是一种在必要时刻才使用的性能优化钩子，平常接触较少也是有可能的，但 useReducer 却值得我们重视。在官方的解释中，useReducer 是 useState 的替代方案，什么情况下值得替代呢，这里同样以一个例子来分析。

  

举个状态模式中最为常见的例子 —— 音乐播放器的顺序切换器。

  

```
function Mode() {
```

  

在上面的实现中，可以看到模式的切换依赖于上一个状态，在“顺序播放-随机播放-循环播放”三个模式中依次切换。目前只有三种模式，可以使用简单的 if...else 方式实现，但一旦模式多了便会十分难以维护和扩展，因此，针对这种行为依赖于状态的场景，当分支增长到一定程度时，便需要考虑使用“状态模式”重新设计。

  

```
function Mode() {
```

  

React 官网中对 useReducer 的解释中提到「在某些场景下，useReducer 会比 useState 更适用，例如 state 逻辑较复杂且包含多个子值，或者下一个 state 依赖于之前的 state 等」。这里着重看一下后一个场景，「类的行为是基于它的状态改变」的“状态模式”就是一种典型的依赖上一个状态的场景，这使 useReducer 天然的适用于多状态切换的业务场景。

###   

```
/* 借助 reducer 更便捷实现状态模式 */
```

  

### 3  自定义 Hook 封装原则

  

自定义 Hook 是 React Hook 广受欢迎的重要原因，然而一个抽象不佳的自定义 Hook 却可能极大增加了维护成本。在《The Ugly Side of React Hooks 》“The hidden side effect” 章节中就列举了一个层层嵌套的副作用带来的异常排查成本。我们经常说设计原则和设计模式有助于提高代码的可维护性和可扩展性，那么有哪些原则/模式能够帮助我们优雅地封装自定义 Hooks 呢？

  

> OS：“如何优雅地封装自定义 Hooks” 是一个很大的话题，这里仅仅抛转引玉讲述几个观点。

  

**第零要义：存在数据驱动**

  

在比较 Hooks 和类组件开发模式时，常常提到的一点就是 Hooks 有助于我们在组件间实现更广泛的功能复用。于是，刚开始学习 Hooks 时，对于任何可能有复用价值的功能逻辑，我经常矫枉过正地封装成奇奇怪怪的 Hooks，比如针对「在用户关闭通知且当天第一次打开时，进行二次提醒打开」这么一个功能，我抽象了一个 useTodayFirstOpen：

  

```
//  Bad case
```

  

事实上，它并没有返回任何东西，在组件内调用时也仅仅是 useTodayFirstOpen() 。回过头来，这块功能并没有任何的外部数据流入，也没有数据流出，完全可以将其抽象为一个高阶函数，而不是一个自定义 Hooks。因此具有复用价值且与外部存在数据驱动关系的功能模块才有必要抽象为自定义 Hooks。

  

**第一要义：单一职责原则**  

  

单一职责原则（SRP）建议一个方法仅有一个引起变更的原因。自定义 Hooks 本质上就是一个抽象方法，方便实现组件内逻辑功能的复用。但是如果一个 Hooks 承担了太多职责，则可能造成某一个职责的变动会影响到另一个职责的执行，造成意想不到的后果，也增加了后续功能迭代过程中出错的概率。至于什么时候应该拆分，参照 SRP 中推荐的职责分离原则，在 Hooks 中更适合解释为「如果引起两个数据变更的原因不是同一个时，则应该将两者分离」。

  

以 useTodayFirstOpen 为例，假设外界还有 Switch 控件需要根据 status 做展示与交互：

  

```
function useTodayFirstOpen() {
```

  

假设 getUserStatus 的返回格式发生了改变，需要修改该 Hook。

  

```
function useTodayFirstOpen() {
```

  

假设再有一天，监管反馈每天二次提醒频率太高了，要求改为每周「二次提醒」，又需要再次重构该 Hook。

  

```
function useThisWeekFirstOpen() {  const [status, setStatus] = useState();  const [isThisWeekFirstOpen, setIsThisWeekFirstOpen] = useState(false);  useEffect(() => {    // 获取用户状态    // ...    // 判断今天是否首次打开    const value = window.locaStorage.getItem('isThisWeekFirstOpen');    if (!value) {      setIsTodayFirstOpen(true);    } else {      const curr = getNowDate();      setIsThisWeekFirstOpen(diffDay(curr, value) >= 7);    }  }, []);  // ...}
```

  

这显然违背了单一职责原则，此时需要考虑分离 status 和 ...FirstOpen 逻辑，使其更加通用，再以组合的形式抽象为业务 Hook。

  

```
// 用户状态管理
```

  

改造之后，如果获取/设置用户状态的接口发生变动，则修改 useUserStatus；如果二次提醒的效果需要改动（如上报日志），则修改 useSecondConfirm；如果业务上调整了二次提醒逻辑（会员不二次提醒），则仅需修改 useStatusWithSecondConfirm ，各自定义 Hooks 各司其职。

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

> 第 n + 1 要义努力探索中……

  

**五  总结**

  

数据结构与设计模式能够指导我们在开发复杂系统中寻得一条清晰的道路，那既然都说 Hooks 难以维护，那就尝试让「神」来拯救这混乱的局面。对于「设计模式是否有助于我们写出更优雅的 Hooks 」这个问题，看完前面的章节，相信你心中也有自己的答案，当然，本文并不是为了辩论「 Hooks 是否强于类开发」这一话题，如果有兴趣的话，欢迎评论留言～

  

---

  

## **阿里云开发者社区“乘风者计划”重磅来袭！诚邀千名技术博主入驻**

  

无论你是小白亦或大神，只要来社区发布3篇原创技术博文，即可获得入驻勋章和精美礼品。发文越多，可解锁更多勋章和官方流量扶持、招聘推荐，个人品牌包装、顶级大会门票、专家对话沙龙等诸多重磅权益。快来升级打怪吧！

  

  

点击阅读原文参加乘风者计划～

  

后端开发107

设计模式10

后端开发 · 目录

上一篇Pull or Push？监控系统如何选型下一篇云上应用系统数据存储架构演进

阅读原文

阅读 1.9万

​