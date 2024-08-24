# 

原创 旭伦 阿里云开发者

 _2021年12月07日 12:17_

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/Z6bicxIx5naLj5MhnPM3ibIw64d8m316ljDqse3Q8O60lcoCQkhlVmictBjJhKsrITUrC0aSCbWo2MdgNku8P45Ew/640?wx_fmt=jpeg&wxfrom=13&tp=wxpic)

  

使用eslint和stylelint之类的工具扫描前端代码现在已经基本成为前端同学的标配。但是，业务这么复杂，指望eslint等提供的工具完全解决业务中遇到的代码问题还是不太现实的。我们一线业务同学也要有自己的写规则的能力。

  

eslint是构建在AST Parser基础上的规则扫描器，缺省情况下使用espree作为AST解析器。rules写好对于AST事件的回调，linter处理源代码之后会根据相应的事件来回调rules中的处理函数。

![图片](https://mmbiz.qpic.cn/mmbiz_png/Z6bicxIx5naJ77rkXMJmicIUkuv1wJ1ATicZqJwrJrdNCc6tyZgQ6ljwpg4ibQoNYIhCPO8x7Rs7Z4yLBr7KgdtDpg/640?wx_fmt=png&wxfrom=13&tp=wxpic)

  

另外，在进入细节之前，请思考一下：eslint的边界在哪里？哪些功能是通过eslint写规则可以做到的，哪些是用eslint无法做到的？  

  

## **一  先学会如何写规则测试**

兵马未动，测试先行。规则写出来，如何用实际代码进行测试呢？

  

所幸非常简单，直接写个json串把代码写进来就好了。

  

我们来看个no-console的例子，就是不允许代码中出现console.*语句的规则。

  

首先把规则和测试运行对象ruleTester引进来：

  

```
//------------------------------------------------------------------------------
```

  

然后我们就直接调用ruleTester的run函数就好了。有效的样例放在valid下面，无效的样例放在invalid下面，是不是很简单。

  

我们先看下有效的：

  

```
ruleTester.run("no-console", rule, {
```

  

能通过的情况比较容易，我们就直接给代码和选项就好。

  

然后是无效的：

  

```
    invalid: [
```

  

无效的要判断下出错信息是不是符合预期。

  

我们使用mocha运行下上面的测试脚本：

  

```
./node_modules/.bin/mocha tests/lib/rules/no-console.js
```

  

运行结果如下：

  

```
  no-console
```

  

如果在valid里面放一个不能通过的，则会报错，比如我们加一个：

  

```
ruleTester.run("no-console", rule, {
```

  

就会报下面的错：

  

```
  1 failing
```

  

说明我们刚加的console是会报一个messageId为unexpected，而nodeType为MemberExpression的错误。

  

我们应将其放入到invalid里面：

  

```
    invalid: [
```

  

再运行，就可以成功了：

  

```
    invalid
```

**二  规则入门**  

会跑测试之后，我们就可以写自己的规则啦。

  

我们先看下规则的模板，其实主要要提供meta对象和create方法：

  

```
module.exports = {
```

  

总体来说，一个eslint规则所能做的事情，就是写事件回调函数，在回调函数中使用context中获取的AST等信息进行分析。

  

context提供的API是比较简洁的：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

代码信息类主要我们使用getScope获取作用域的信息，getAncestors获取上一级AST节点，getDeclaredVariables获取变量表。最后的绝招是直接获取源代码getSourceCode自己分析去。

  

markVariableAsUsed用于跨文件分析，用于分析变量的使用情况。

  

report函数用于输出分析结果，比如报错信息、修改建议和自动修复的代码等。

  

这么说太抽象了，我们来看例子。

  

还以no-console为例，我们先看meta部分，这部分不涉及逻辑代码，都是一些配置：

  

```
    meta: {
```

  

我们再看no-console的回调函数，只处理一处Program:exit, 这是程序退出的事件：

  

```
        return {
```

  

### 1  获取作用域和AST信息

  

我们首先通过context.getScope()获取作用域信息。作用域与AST的对应关系如下图：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

我们前面的console语句的例子，首先拿到的都是全局作用域，举例如下：

  

```
<ref *1> GlobalScope {
```

  

具体看一下38个全局变量，复习下Javascript基础吧：

  

```
    set: Map(38) {
```

  

我们看到，所有的变量，都以一个名为set的Map中，这样我们就可以以遍历获取所有的变量。

  

针对no-console的规则，我们主要是要查找是否有叫console的变量名。于是可以这么写：

  

```
    getVariableByName(initScope, name) {
```

  

我们可以在刚才列出的38个变量中发现，console是并没有定义的变量，所以

  

```
const consoleVar = astUtils.getVariableByName(scope, "console");
```

  

的结果是null. 

  

于是我们要去查找未定义的变量，这部分是在scope.through中，果然找到了name是console的节点：

  

```
[
```

  

这样我们就可以写个检查reference的名字是不是console的函数就好：

  

```
        function isConsole(reference) {
```

  

然后用这个函数去filter scope.though中的所有未定义的变量：

  

```
scope.through.filter(isConsole);
```

  

最后一步是输出报告，针对过滤出的reference进行报告：

  

```
                    references
```

  

报告问题使用context的report函数：

  

```
        function report(reference) {
```

  

发生问题的代码行数可以从node中获取到。

  

### 2  处理特定类型的语句

  

no-console从规则书写上并不是最容易的，我们以其为例主要是这类问题最多。下面我们举一反三，看看针对其它不应该出现的语句该如何处理。

  

其中最简单的就是针对一类语句统统报错，比如no-continue规则，就是遇到ContinueStatement就报错：

  

```
module.exports = {
```

  

不允许使用debugger的no-debugger规则：

  

```
    create(context) {
```

  

不许使用with语句：

  

```
    create(context) {
```

  

在case语句中不许定义变量、函数和类：

  

```
    create(context) {
```

  

多个类型语句可以共用一个处理函数。

  

比如不许使用构造方法生成数组：

  

```
        function check(node) {
```

  

不许给类定义赋值：

  

```
    create(context) {
```

  

函数的参数不允许重名：

  

```
    create(context) {
```

  

如果事件太多的话，可以写成一个数组，这被称为选择器数组：

  

```
const allLoopTypes = ["WhileStatement", "DoWhileStatement", "ForStatement", "ForInStatement", "ForOfStatement"];
```

  

除了直接处理语句类型，还可以针对类型加上一些额外的判断。

  

比如不允许使用delete运算符：

  

```
    create(context) {
```

  

不准使用"=="和"!="运算符：

  

```
    create(context) {
```

  

不许和-0进行比较：

  

```
    create(context) {
```

  

不准给常量赋值：

  

```
    create(context) {
```

  

### 3  :exit - 语句结束事件  

  

除了语句事件之外，eslint还提供了:exit事件。

  

比如上面的例子我们使用了VariableDeclaration语句事件，我们下面看看如何使用VariableDeclaration结束时调用的VariableDeclaration:exit事件。

  

我们看一个不允许使用var定义变量的例子：

  

```
        return {
```

  

如果觉得进入和退出不好区分的话，我们来看一个不允许在非函数的块中使用var来定义变量的例子：

  

```
            BlockStatement: enterScope,
```

  

这些逻辑的作用是，进入语句块的时候调用enterScope，退出语句块的时候调用exitScope:

        

```
        function enterScope(node) {
```

  

### 4  直接使用文字信息 - Literal

  

比如不允许使用"-.7"这样省略了0的浮点数。此时使用Literal来处理纯文字信息。

  

```
    create(context) {
```

  

不准使用八进制数字：

  

```
    create(context) {
```

  

## **三  代码路径分析**

前面我们讨论的基本都是一个代码片段，现在我们把代码逻辑串起来，形成一条代码路径。

  

代码路径就不止只有顺序结构，还有分支和循环。

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

除了采用上面的事件处理方法之外，我们还可以针对CodePath事件进行处理：

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

事件onCodePathStart和onCodePathEnd用于整个路径的分析，而onCodePathSegmentStart, onCodePathSegmentEnd是CodePath中的一个片段，onCodePathSegmentLoop是循环片段。

  

我们来看一个循环的例子：

  

```
    create(context) {
```

  

## **四  提供问题自动修复的代码**

最后，我们讲讲如何给问题给供自动修复代码。

  

我们之前报告问题都是使用context.report函数，自动修复代码也是通过这个接口返回给调用者。

  

我们以将"=="和"!="替换成"==="和"!=="为例。

  

这个fix没有多少技术含量哈，就是给原来发现问题的运算符多加一个"=":

  

```
report(node, `${node.operator}=`);
```

  

最终实现时是调用了fixer的replaceText函数：

  

```
                fix(fixer) {
```

  

完整的report代码如下：

  

```
        function report(node, expectedOperator) {
```

  

Fixer支持4个添加API，2个删除API，2个替换类的API：  

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

##   

## **五  高级话题**

### 1  React JSX的支持

  

Facebook给我们封装好了框架，写起来也是蛮眼熟的。刚好之前没有举markVariableAsUsed的例子，正好一起看了：

  

```
module.exports = {
```

  

JSX的特殊之处是增加了JSXOpenElement, JSXClosingElement, JSXOpenFragment, JSXClosingFragment等处理JSX的事件。

  

### 2  TypeScript的支持

  

随着tslint合并到eslint中，TypeScript的lint功能由typescript-eslint承载。

  

因为estree只支持javascript，typescript-eslint提供兼容estree格式的parser. 

  

既然是ts的lint，自然是拥有了ts的支持，拥有了新的工具方法，其基本架构仍是和eslint一致的：

  

```
import * as ts from 'typescript';
```

  

### 3  更换ESLint的AST解析器

  

ESLint支持使用第三方AST解析器，刚好Babel也支持ESLint，于是我们就可以用@babel/eslint-parser来替换espree. 装好插件之后，修改.eslintrc.js即可：

  

```
module.exports = {
```

  

Babel自带支持TypeScript。

  

## **六  StyleLint**

说完了Eslint，我们再花一小点篇幅看下StyleLint。

  

StyleLint与Eslint的架构思想一脉相承，都是对于AST的事件分析进行处理的工具。

  

只不过css使用不同的AST Parser，比如Post CSS API, postcss-value-parser, postcss-selector-parser等。

  

我们来看个例子体感一下：

  

```
const rule = (primary) => {
```

  

也是熟悉的report函数回报，也可以支持autofix的生成。  
  

## **七  小结**

以上，我们基本将eslint规则写法的大致框架梳理清楚了。当然，实际写规刚的过程中还需要对于AST以及语言细节有比较深的了解。预祝大家通过写出适合自己业务的检查器，写出更健壮的代码。

  

---

  

## **网站架构师（CUED）培训课程**

网站架构师CUED(Cloud User Experience Design)，集项目经理、产品经理、原型设计师等多重身份于一身，帮助客户整理需求、内容及框架搭建工作，把客户的需求完整地在网站系统中实现 。需要网站架构师具备完整的逻辑能力，对行业有较深理解，协助用户完成网站的原型设计。点击阅读原文了解详情。

后端开发107

前端开发29

后端开发 · 目录

上一篇开源微服务编排框架：Netflix Conductor下一篇基于链路思想的SpringBoot单元测试快速写法

阅读原文

阅读 1.0万

​

写留言

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/Z6bicxIx5naI1jwOfnA1w4PL2LhwNia76vBRfzqaQVVVlqiaLjmWYQXHsn1FqBHhuGVcxEHjxE9tibBFBjcB352fhQ/300?wx_fmt=png&wxfrom=18)

阿里云开发者

645

写留言

写留言

**留言**

暂无留言