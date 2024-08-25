# 

原创 damyxu 腾讯技术工程

 _2021年12月13日 18:02_

![图片](https://mmbiz.qpic.cn/mmbiz_gif/j3gficicyOvasIjZpiaTNIPReJVWEJf7UGpmokI3LL4NbQDb8fO48fYROmYPXUhXFN8IdDqPcI1gA6OfSLsQHxB4w/640?wx_fmt=gif&wxfrom=13&tp=wxpic)

  

作者：damyxu，腾讯 PCG 前端开发工程师

> iframe是一个天然的微前端方案，但受限于跨域的严格限制而无法很好的应用，本文介绍一种基于 iframe 的全新微前端方案，继承iframe的优点，补足 iframe 的缺点，让 iframe 焕发新生。

  

### 背景

前端开发中我们对`iframe`已经非常熟悉了，那么`iframe`的作用是什么？可以归纳如下：

在一个`web`应用中可以**独立**的运行另一个`web`应用

这个概念已经和[微前端](https://micro-frontends.org/)不谋而合，相对于目前配置复杂、高适配成本的微前端方案来说，采用`iframe`方案具有一些显著的**优点**：

- **非常简单**，使用没有任何心智负担
    
- **隔离完美**，无论是 js、css、dom 都完全隔离开来
    
- **多应用激活**，页面上可以摆放多个`iframe`来组合业务
    

但是开发者又厌恶使用`iframe`，因为**缺点**也非常明显：

- **路由状态丢失**，刷新一下，iframe 的 url 状态就丢失了
    
- **dom 割裂严重**，弹窗只能在 iframe 内部展示，无法覆盖全局
    
- **通信非常困难**，只能通过 postmessage 传递序列化的消息
    
- **白屏时间太长**，对于[SPA 应用](https://zh.wikipedia.org/wiki/%E5%8D%95%E9%A1%B5%E5%BA%94%E7%94%A8)应用来说无法接受
    

能否打造一个完美的`iframe`，保留所有的优点的同时，解决掉所有的缺点呢？

### 无界方案

无界微前端框架通过继承`iframe`的优点，解决`iframe`的缺点，打造一个接近完美的`iframe`方案。

来看无界如何一步一步的解决`iframe`的问题，假设我们有 A 应用，想要加载 B 应用：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

在应用 A 中构造一个`shadow`和`iframe`，然后将应用 B 的`html`写入`shadow`中，`js`运行在`iframe`中，**注意`iframe`的`url`**，`iframe`保持和主应用同域但是保留子应用的路径信息，这样子应用的`js`可以运行在`iframe`的`location`和`history`中保持路由正确。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

image-20211206160113792

在`iframe`中拦截`document`对象，统一将`dom`指向`shadowRoot`，此时比如新建元素、弹窗或者冒泡组件就可以正常约束在`shadowRoot`内部。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

接下来的三步分别解决`iframe`的三个缺点：

- ✅ dom 割裂严重的问题，主应用提供一个容器给到`shadowRoot`插拔，`shadowRoot`内部的弹窗也就可以覆盖到整个应用 A
    
- ✅ 路由状态丢失的问题，浏览器的前进后退可以天然的作用到`iframe`上，此时监听`iframe`的路由变化并同步到主应用，如果刷新浏览器，就可以从 url 读回保存的路由
    
- ✅ 通信非常困难的问题，`iframe`和主应用是同域的，天然的共享内存通信，而且无界提供了一个去中心化的事件机制
    

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

将这套机制封装进`wujie`框架：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

我们可以发现：

- ✅ 首次白屏的问题，`wujie`实例可以提前实例化，包括`shadowRoot`、`iframe`的创建、`js`的执行，这样极大的加快子应用第一次打开的时间
    
- ✅ 切换白屏的问题，一旦`wujie`实例可以缓存下来，子应用的切换成本变的极低，如果采用保活模式，那么相当于`shadowRoot`的插拔
    

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

image-20211206160227875

由于子应用完全独立的运行在`iframe`内，路由依赖`iframe`的`location`和`history`，我们还可以在一张页面上同时激活多个子应用，由于`iframe`和主应用处于同一个[top-level browsing context](https://html.spec.whatwg.org/multipage/browsers.html#top-level-browsing-context)，因此浏览器前进、后退都可以作用到到子应用：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

image-20211206160244704

通过以上方法，无界方案可以做到：

- ✅ **非常简单**，使用没有任何心智负担
    
- ✅ **隔离完美**，无论是 js、css、dom 都完全隔离开来
    
- ✅ **多应用激活**，页面上可以摆放多个`iframe`来组合业务
    
- **路由状态丢失**，刷新一下，iframe 的 url 状态就丢失了
    
- **dom 割裂严重**，弹窗只能在 iframe 内部展示，无法覆盖全局
    
- **通信非常困难**，只能通过 postmessage 传递序列化的消息
    
- **白屏时间太长**，对于[SPA 应用](https://zh.wikipedia.org/wiki/%E5%8D%95%E9%A1%B5%E5%BA%94%E7%94%A8)应用来说无法接受
    

### 使用无界

如果主应用是`vue`框架：

**安装**

`` `npm i @tencent/wujie-vue -S`    ``

**引入**

`mport WujieVue from "@tencent/wujie-vue";   Vue.use(WujieVue);   `

**使用**

`<WujieVue     width="100%"     height="100%"     name="xxx"     url="xxx"     :sync="true"     :fetch="fetch"     :props="props"     @xxx="handleXXX"   ></WujieVue>   `

其他框架也会在近期上线

### 适配成本

无界的适配成本非常低

对于主应用无需做任何改造

对于子应用：

- 前提，必须开放跨域配置，因为子应用是在主应用域内请求和运行的
    
- 对`webpack`应用，修改动态加载路径
    
- 如果子应用保活模式则无需进一步修改，非保活则需要将实例化挂载到无界生命周期内
    

`if (window.__POWERED_BY_WUJIE__) {     let instance;     window.__WUJIE_MOUNT = () => {       instance = new Vue({ router, render: (h) => h(App) }).$mount("#app");     };     window.__WUJIE_UNMOUNT = () => {       instance.$destroy();     };   } else {     new Vue({ router, render: (h) => h(App) }).$mount("#app");   }   `

### 实现细节

#### 实现一个纯净的 iframe

子应用运行在一个和主应用同域的`iframe`中，设置`src`为替换了主域名`host`的子应用`url`，子应用路由只取`location`的`pathname`和`hash`

但是一旦设置`src`后，`iframe`由于同域，会加载主应用的`html`、`js`，所以必须在`iframe`实例化完成并且还没有加载完`html`时中断加载，防止污染子应用

此时可以采用轮询监听`document.readyState`状态来及时中断，对于一些浏览器比如`safari`状态不准确，可以在`wujie`主动抛错来防止有主应用的`js`运行

#### iframe 数据劫持和注入

子应用的代码 `code` 在 `iframe` 内部访问 `window`，`document`、`location` 都被劫持到相应的 `proxy`，并且还会注入`$wujie`对象供子应用调用

``const script = `(function(window, self, global, document, location, $wujie) {       ${code}\n     }).bind(window.__WUJIE.proxy)(       window.__WUJIE.proxy,       window.__WUJIE.proxy,       window.__WUJIE.proxy,       window.__WUJIE.proxy.document,       window.__WUJIE.proxy.location,       window.__WUJIE.provide     );`;   ``

#### iframe 和 shadowRoot 副作用的处理

`iframe` 内部的副作用处理在初始化`iframe`时进行，主要分为如下几部

`/**    * 1、location劫持后的数据修改回来，防止跨域错误    * 2、同步路由到主应用    */   patchIframeHistory(iframeWindow, appPublicPath, mainPublicPath);   /**    * 对window.addEventListener进行劫持，比如resize事件必须是监听主应用的    */   patchIframeEvents(iframeWindow);   /**    * 注入私有变量    */   patchIframeVariable(iframeWindow, appPublicPath);   /**    * 将有DOM副作用的统一在此修改，比如mutationObserver必须调用主应用的    */   patchIframeDomEffect(iframeWindow);   /**    * 子应用前进后退，同步路由到主应用    */   syncIframeUrlToWindow(iframeWindow);   `

`ShadowRoot` 内部的副作用必须进行处理，主要处理的就是`shadowRoot`的`head`和`body`

  `shadowRoot.head.appendChild = getOverwrittenAppendChildOrInsertBefore({       rawDOMAppendOrInsertBefore: rawHeadAppendChild     }) as typeof rawHeadAppendChild     shadowRoot.head.insertBefore = getOverwrittenAppendChildOrInsertBefore({       rawDOMAppendOrInsertBefore: rawHeadInsertBefore as any     }) as typeof rawHeadInsertBefore     shadowRoot.body.appendChild = getOverwrittenAppendChildOrInsertBefore({       rawDOMAppendOrInsertBefore: rawBodyAppendChild     }) as typeof rawBodyAppendChild     shadowRoot.body.insertBefore = getOverwrittenAppendChildOrInsertBefore({       rawDOMAppendOrInsertBefore: rawBodyInsertBefore as any     }) as typeof rawBodyInsertBefore`

`getOverwrittenAppendChildOrInsertBefore`主要是处理四种类型标签：

- **`link/style`标签**
    
    收集`stylesheetElement`并用于子应用重新激活重建样式
    
- **`script`标签**
    
    动态插入的`script`标签必须从`ShadowRoot`转移至`iframe`内部执行
    
- **`iframe`标签**
    
    修复`iframe`的指向对应子应用`iframe`的`window`
    

#### iframe 的 document 改造

由于`js`在`iframe`运行需要和`shadowRoot`，包括元素创建、事件绑定等等，将`iframe`的`document`进行劫持：

- 所有的元素的查询全部代理到`shadowRoot`内去查询
    
- `head`和`body`代理到`shadowRoot`的对应`html`元素上
    

#### iframe 的 location 改造

将`iframe`的`location`进行劫持：

- 由于`iframe`的`url`的`host`是主应用的，所以需要将`host`改回子应用自己的
    
- 对于`location.href`特殊逻辑的处理
    

### 总结

通过上面原理以及细节的阐述，我们可以得出无界微前端框架的几点优势：

- **多应用同时激活在线**框架具备同时激活多应用，并保持这些应用路由同步的能力
    
- **组件式的使用方式**无需注册，更无需路由适配，在组件内使用，跟随组件装载、卸载
    
- **应用级别的 keep-alive**子应用开启保活模式后，应用发生切换时整个子应用的状态可以保存下来不丢失，结合预执行模式可以获得类似`ssr`的打开体验
    
- **纯净无污染**
    

- 无界利用`iframe`和`ShadowRoot`来搭建天然的`js`隔离沙箱和`css`隔离沙箱
    
- 利用`iframe`的`history`和主应用的`history`在同一个[top-level browsing context](https://html.spec.whatwg.org/multipage/browsers.html#top-level-browsing-context)来搭建天然的路由同步机制
    
- 副作用局限在沙箱内部，子应用切换无需任何清理工作，没有额外的切换成本
    

- **性能和体积兼具**
    

- 子应用执行性能和原生一致，子应用实例`instance`运行在`iframe`的`window`上下文中，避免`with(proxyWindow){code}`这样指定代码执行上下文导致的性能下降，但是多了实例化`iframe`的一次性的开销，可以通过`proloadApp`提前实例化
    
- 包体积只有`11kb`，非常轻量，借助`iframe`和`ShadowRoot`来实现沙箱，极大的减小了代码量
    

- **开箱即用**不管是样式的兼容、路由的处理、弹窗的处理、热更新的加载，子应用完成接入即可开箱即用无需额外处理，应用接入成本也极低
    

相应的也有所不足：

- 内存占用较高，为了降低子应用的白屏时间，将未激活子应用的`shadowRoot`和`iframe`常驻内存并且保活模式下每张页面都需要独占一个`wujie`实例，内存开销较大
    
- 兼容性一般，目前用到了浏览器的`shadowRoot`和`proxy`能力，并且没有做降级方案
    
- `iframe`劫持`document`到`shadowRoot`时，某些第三方库可能无法兼容导致穿透
    

  

**近期好文：**  

[微信支付团队精益研发实践总结](http://mp.weixin.qq.com/s?__biz=MjM5ODYwMjI2MA==&mid=2649766177&idx=1&sn=14f0343184b317b40678d49d0bb83ecd&chksm=beccaa5a89bb234c1ccb1dbaac2f8bd699afe2954aa5427cb5ed0351ea934af2dc988ca0e676&scene=21#wechat_redirect)  

[梳理正则表达式发展史](http://mp.weixin.qq.com/s?__biz=MjM5ODYwMjI2MA==&mid=2649766143&idx=1&sn=c1c2e0bc00b0244470390285407976d0&chksm=beccab8489bb2292b50571f5c96ab9d56a8d046622838357ace15aaa2f8ede911ac767bc9b84&scene=21#wechat_redirect)  

[程序员妈妈的“work-life balance”，直面想象中的困难](http://mp.weixin.qq.com/s?__biz=MjM5ODYwMjI2MA==&mid=2649766110&idx=1&sn=39793d74a238a15ce0eac2733ec0a9e3&chksm=beccaba589bb22b3089efced190fa5d57ab25f079fb931f534259a341a78754a01aff81344f7&scene=21#wechat_redirect)  

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

**程序员教你做咖啡**  

阅读 8152

​