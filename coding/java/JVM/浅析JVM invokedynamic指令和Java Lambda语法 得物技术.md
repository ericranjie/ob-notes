Original 羊羽 得物技术

_2024年08月26日 18:30_

![Image](https://mmbiz.qpic.cn/mmbiz_gif/AAQtmjCc74DZeqm2Rc4qc7ocVLZVd8FOASKicbMfKsaziasqIDXGPt8yR8anxPO3NCF4a4DkYCACam4oNAOBmSbA/640?wx_fmt=gif&tp=wxpic&wxfrom=5&wx_lazy=1)

**目录**

一、导语

二、Java和JVM的关系

三、JVM指令：invokedynamic

四、方法句柄：MethodHandle

五、Lambda表达式简介

六、Lambda表达式实现

1. invokedynamic指令参数

2. 期望的方法名称和描述符

3. BSM方法序号

4\. BSM方法

5. BSM方法参数

6. LambdaMetafactory#metafactory

7\. 构造CallSite

8\. 二阶段调用

七、Lambda表达式性能

八、Lambda表达式和final变量

九、总结

十、附录

1\. 自动生成的Lambda2适配类

2\. 自动生成的Lambda3适配类

**一**

**导语**

尽管近年来JDK的版本发布愈发敏捷，当前最新版本号已经20+，但是日常使用中，JDK8还是占据了统治地位。

![Image](https://mmbiz.qpic.cn/mmbiz_png/AAQtmjCc74Aj6AP8PysMiaRWBKcnwu2Dd5PG4nyicx9ibBIBL0opq9szAyjr3ZV5nvibBmAzXL4HNNgw6G70EbbylA/640?wx_fmt=png&from=appmsg&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

_你发任你发，我用Java8：\_\_【Jetbrains】2023 开发者生态系统现状 - https://www.jetbrains.com/zh-cn/lp/devecosystem-2023/java/_

JDK8如此旺盛的生命力，与其优异的兼容性、稳定性和足够日常开发使用的语言特性有极大的关系，这其中最引人瞩目的语言特性莫过于Lambda表达式。

Lambda表达式语言特性引入Java语言后，赋予了Java语言更便捷的函数式编程魔力(相对匿名内部类)，同时也让其更简洁，毕竟Java代码写起来啰嗦这点一直被开发者们广泛诟病。

本文将从JVM和Java两个层面着手，和大家一起深入解析Lambda表达式。

**二**

**Java和JVM的关系**

JVM是HLLVM(高级语言虚拟机)，其参考物理计算机体系架构，设计、实现了一套特定领域虚拟指令集，即：字节码指令。利用上述虚拟指令集作为中间层，将上层高级语言和底层体系架构解耦以规避繁琐、复杂的平台兼容性问题，以实现【一次编译，处处运行】。

Java是基于JVM提供的虚拟指令集，设计、实现的一种供开发者使用的高级语言。通过配套的编译器和标准库，将文本格式的Java代码编译成符合JVM指令集规范的二进制文件，交付到JVM执行。

Java是一种运行在JVM平台上的高级语言，但是JVM平台绝不是只能运行Java语言。任何人都可以设计自己的语言语法，只要能按JVM规范编译成合法的JVM字节码，即可在JVM上运行(用Java命令)。

**计算机科学领域的任何问题，都可以通过增加一个中间层来解决。**

![Image](https://mmbiz.qpic.cn/mmbiz_jpg/AAQtmjCc74Aj6AP8PysMiaRWBKcnwu2DdGdNVSvWaCtf9ouM98ck88FINjHiaDHQJQeqEGSibHTonImzIwnY92FUw/640?wx_fmt=jpeg&from=appmsg&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

没有无源之水，Java语言层面的特性，除非是纯语法糖，不然一定离不开特定JVM特性的支撑。Lambda是Java8语言特性，那支撑它的便是JVM invokedynamic指令。

**三**

**JVM指令：invokedynamic**

在Java7之前，JVM提供了如下4种【方法调用】指令：

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

上述4种字节码指令各自有不同的使用场景，但是有一个共同的特点：**目标方法一定需要在【编译期】确定**。如下图，编译后4种指令的参数都指定了目标方法所在的类和签名以供**运行时链接、动态分派**。

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

这个特点一方面保证了JVM语言类型安全，另一方面也限制了JVM平台对动态类型高级语言的支持。比如想让JavaScript、Python等动态语言代码编译成JVM字节码运行在JVM平台上的开销会比较大，性能也会比较差。

为了解决上述问题， Java7引入了一条新的虚拟机指令：**invokedynamic**。这是自JVM 1.0以来**第一次引入新的虚拟机指令**，invokedynamic与其他 invoke\*指令不同的是它允许由**应用级的代码**来决定方法解析(链接、分派)。

所谓的【应用级的代码来决定方法解析】需要对照之前的invoke*指令来理解。之前的4种invoke*指令，在编译期就必须要明确目标方法并hardcode到字节码中，JVM在运行时直接解析、链接、动态分派硬编码指定的目标方法。而invokedynamic指令通过**回调机制**来获取需要调用的目标方法。即先调用业务自定义回调方法做方法决策(解析、链接)，再调用其返回的目标方法。笔者称之为【**两阶段调用**】。

伪代码对比如下：

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

_MethdoHandle为示意，后文有详述。_

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

伪字节码

**invokevirtual指令直接调用目标方法，invokedynamic直接调用回调方法，再调用回调方法返回的方法句柄。**

传统的invoke\*指令直接调用字节码中指定的目标方法，如`Son.testMethod1`，invokedynamic指令在调用时，先调用字节码中指定的回调方法，如`Son.dynamicMethodCallback`，然后再调用回调方法(hook)返回的方法引用。

而上述`dynamicMethodCallback`即为【**应用级的代码或者我们常说的业务代码**】，可以在不影响性能的前提下，灵活的干预JVM方法解析、链接的过程。

总结来说，所谓应用级的代码其实也是一个方法，在这里这个方法被称为**引导方法（Bootstrap Method）**，简称 **BSM**。invokedynamic执行时，BSM先被调用并返回一个 CallSite(调用点)对象，这个对象就和 invokedynamic链接在一起。以后再执行这条invokedynamic指令都不会创建新的 CallSite 对象。CallSite就是一个 MethodHandle(方法句柄)的holder，方法句柄指向一个调用点真正执行的方法。

\*\*一阶段：\*\*调用引导方法确定并缓存CallSite(MethodHandle)

\*\*二阶段：\*\*调用CallSite(MethodHandle)

字节码指令比较low level，除字节码业务插桩场景外，字节码指令序列的构造、编排一般都由【高级语言编译器】来根据语言语法规则自动完成，如javac。

某种意义上有点类似Java【动态代理】机制，都是通过调用横切来动态桥接、灵活决策目标方法。

**四**

**方法句柄：MethodHandle**

前面我们知道invokedynamic指令支持通过业务层面自定义的BSM来灵活的决策被调用的目标方法，也就是上述的【一阶段】。BSM方法的返回值就是【二阶段】调用的方法。

但是和C、Python等语言不同，Java中方法/函数不是一等公民，也就是在Java中无法将【方法变量】作为方法返回值。

为了解决这个问题，Java标准库提供了一个新的类型MethodHandle，用于实现类似C语言中的方法指针、JavaScript/Python中方法变量的能力。该API和反射API呈现的能力相似，但是性能更好。

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

上述为MethodHandle API的基本使用，该课题展开又是一篇长文。总之，我们可以用MethodHandle来作为【方法变量】，变相的将【Java方法】提升为【一等公民】，从而可以在BSM中用Java代码实现动态编排、决策，返回合适的方法指针。这也是上述**invokedynamic+BSM机制**能够成立的一个基础。

\_详见：\__秒懂Java之方法句柄(MethodHandle) （https://blog.csdn.net/ShuSheng0007/article/details/107066856）_

上述【一阶段】调用的本质就是得到一个特定的MethodHandle(方法指针/方法引用)，【二阶段】调用就是调用这个MethodHandle。

**五**

**Lambda表达式简介**

Java的Lambda表达式，是传统的【匿名内部类】特性在特定场景下的平替特性。所谓的特定场景，即我们熟知的FunctionalInterface。

当【匿名内部类】匿名实现的是一个FunctionalInterface时，可以用Lambda表达式平替。

示例如下：

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

_函数式接口(Functional Interface)就是一个有且仅有一个抽象方法，但是可以有多个非抽象方法的接口。_

_Java 不会强制要求你使用 @FunctionalInterface 注解来标记你的接口是函数式接口，然而，作为API作者，你可能倾向使用@FunctionalInterface指明特定的接口为函数式接口，这只是一个设计上的考虑，可以让用户很明显的知道一个接口是函数式接口。_

Java Lambda表达式在语法层面有两种形式：行内代码块、方法引用。

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

但是在编译产物中，行内Lambda最终会被提取到独立的静态方法中。也就是说，在字节码层面只有【方法引用】一种Lambda形式。

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

如上图反编译结果，两个行内Lambda中的代码在编译后被提取到两个自动生成的方法`lambda$main$0`、`lambda$main$1`，**后续Lambda表达式的处理流程都可以收敛，无需区分对待**。

**六**

**Lambda表达式实现**

Lambda表达式具体的实现涉及类文件结构、字节码指令结构、标准库等多个方面的内容，千头万绪。也想不出来什么通俗易懂的叙述角度，只能是枯燥的对照着字节码分析了。

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

如上图，mian方法中声明了3个Lambda表达式，反编译字节码可以看到字节码指令流如下：

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

```
 0 iconst_3
```

3个lambda表达式对应3条invokedynamic指令：

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

第一个lambda表达式比较简单且典型，后续我们以其为抓手展开分析。

**invokedynamic指令参数**

invokedynamic指令参数结构如下：

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

_jvms-6.5.invokedynamic_ _（https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-6.html#jvms-6.5.invokedynamic）_

invokedynamic指令需要指定其**期待BSM返回的方法特征**(出入参类型)和**BSM方法引用**。该参数以`CONSTANT_InvokeDynamic_info`结构存放在类文件的常量池结构中，invokedynamic用两个byte宽度的常量池索引号指定。

```
CONSTANT_InvokeDynamic_info {
```

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

对照字节码我们可知，Lambda1相关的invokedynamic指定的`CONSTANT_InvokeDynamic_info`序号为3，得到如下内容：

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**期望的方法名称和描述符**

该invokedynamic指令期望BSM0方法返回一个如下特征的方法引用：

```
IntUnaryOperator anyName()；
```

_没有入参，返回值类型为IntUnaryOperator的MethodHandle。_

为什么是返回`IntUnaryOperator`类型呢？因为IntStream的map方法需要的参数是`IntUnaryOperator`类型。

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

换句话说，该invokedynamic指令希望相应的BSM返回一个`IntUnaryOperator`的工厂方法句柄，然后invokedynamic指令再调用这个方法句柄，创建出一个map方法需要的`IntUnaryOperator`类型的参数。

**BSM方法序号**

BSM方法序号指定了当前invokedynamic指令使用的BSM方法在BSM方法表中的索引。

通俗来说，类文件中有一个数组，数组名称叫`BootstrapMethods`。其结构如下：

```
BootstrapMethods_attribute {
```

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

该invokedynamic指令指定的BSM为BSM数组中的第一个BSM。

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**BSM方法**

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**BSM方法参数**

该BSM数据结构指定了3个编译期固定的、静态的BSM方法参数：

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

第一、第三个参数指定了预期的函数式接口(FunctionInterface)的特征：入参为int、出参为int。即上述`IntUnaryOperator`。

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

第二个参数是一个静态方法引用。如上述，Lambda表达式在编译时会被提取到一个自动生成的方法中。

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**至此，invokedynamic指令具有的发起【一阶段调用】的上下文如下：**

1. 具体的一阶段调用的BSM方法：`java.lang.invoke.LambdaMetafactory#metafactory`

1. IntStream.map方法需要的参数类型：`IntUnaryOperator`

1. 编译器(javac)编译产生的包含Lambda表达式代码内容的静态方法：`lambda$main$0(I)I`

接下来就是调用`java.lang.invoke.LambdaMetafactory#metafactory`方法，传递上述必要的上下文参数，接受`metafactory`方法返回的`IntUnaryOperator applyAsInt()`类型的MethodHandle并调用该MethodHandle，继而得到IntStream.map方法需要的参数：`IntUnaryOperator`。

**LambdaMetafactory#**

**metafactory**

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

如上述，invokedynamic指令调用上述`metafactory`方法，对照字节码信息，可以得到如下具体参数表格：

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

LambdaMetafactory根据上述上下文，使用ASM库，动态生成了一个如下所示的`IntUnaryOperator`适配类，用于桥接Lambda表达式代码块到`IntUnaryOperator`类型。

_添加_`-Djdk.internal.lambda.dumpProxyClasses=.`_启动参数，JDK会将生成的适配函数式接口的类源码输出到工作目录中。_

**构造CallSite**

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

_java.lang.invoke.InnerClassLambdaMetafactory#buildCallSite_

生成FunctionalInterface适配类后，基于适配类创建`MethodHandle`。该`MethodHandle`体现的代码逻辑类似如下Java代码：

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

至此，invokedynamic【一阶段】调用已经完成，invokedynamic指令获取到了由`LambdaMetafactory#metafactory`作为BSM动态决策、动态生成的`IntUnaryOperator`**适配类的【工厂方法】(以CallSite包装的MethodHandle的形式)。**

**二阶段调用**

【一阶段调用】已经完成，返回了动态决策产生的CallSite对象，getTarget方法可以获取上述的`IntUnaryOperator`适配类的【工厂方法】。

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

至此，invokedynamic指令可以通过如下伪代码，创建IntStream.map方法需要的`IntUnaryOperator`实例。

```
IntUnaryOperator intUnaryOperator = (IntUnaryOperator)callSite.getTarget().invoke()
```

Lambda1的整个运行时解析、链接流程完成。

**七**

**Lambda表达式性能**

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

经过上述分析我们可以知道，**Lambda1**这种\*\*无状态的、没有捕获外部变量(闭包)\*\*的Lambda表达式的开销是很小的，只会在第一次调用时动态生成桥接的适配类，实例化后就通过`ConstantCallSite`缓存。后续所有的调用都不会再重新生成适配类、实例化适配类。

但是，**Lambda2**则不同，因为Lambda捕获、依赖了(闭包)外部变量`num`，那么这个表达式就是有状态的。虽然同样只是会在第一次调用时动态生成桥接的适配类，但是每一次调用都会使用`num`变量重新实例化一个新的适配类实例。这种场景下，其在性能和形式上就已经和传统的【匿名内部类】没有太大差别了。

**Lambda3**本质上和**Lambda1**一样，只不过不需要Java编译器在编译时将Lambda代码语句抽取成独立的方法。

**八**

\*\*Lambda表达式和final变量\
\*\*

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

当Lambda表达式闭包捕获的局部变量`num`在方法内可变时，编译器会提示编译错误。这不是JVM的限制，而是Java语言层面的限制。笔者认为，这种限制没有技术上的原因，而是Java语言设计者刻意的借助编译器在阻止你犯错。

假设没有这个限制，那么**Lambda表达式就变成了重构不友好的【位置相关】的代码块**。

换句话说，下面两种代码执行结果是不一样的：

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

_Lambda捕获的num的值为5；_

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

_Lambda捕获的num的值为3；_

如果没有类似的编译约束，当我们有心或无意的在复杂的业务逻辑中进行了类似的代码调整时，极易出错且难以排查。

笔者个人见解，欢迎指正。

**九**

**总结**

提笔的时候立意高远，想着要尽可能通俗详尽的写清楚所有涉及的技术点，但是越写越觉得事情不简单，最后只能是把博客标题从【深入剖析】修改为【浅析】。这块内容牵涉的面太广，笔者没有能力也没有精力介绍到事无巨细、面面俱到，只能为大家抛砖引玉，大家可以配合后文【参考资料】多梳理、多实验，同时在评论区批评指正。

1. invokedynamic指令不是业务开发者使用的。invokedynamic指令可以用来实现Lambda语法，但是它不是只能用来实现Lambda语法。这个指令对于JVM语言开发者比如Kotlin、Groovy、JRuby、Jython等会比较重要。

1. 没有捕获外部变量(闭包)的Lambda表达式性能和直接调用没有差别。

1. 捕获外部变量(闭包)的Lambda表达式性能理论上和【匿名内部类】范式一样，每次调用都会创建一个对象(**最坏情况**)。

_本文使用的反编译工具为：jclasslib Bytecode Viewer_

_（https://plugins.jetbrains.com/plugin/9248-jclasslib-bytecode-viewer）_

**十**

**附录**

**自动生成的Lambda2适配类**

```
// $FF: synthetic class
```

**自动生成的Lambda3适配类**

```
// $FF: synthetic class
```

**参考：**

- **Oracle-Java虚拟机规范(JDK8)**--_https://docs.oracle.com/javase/specs/jvms/se8/html/_

- **Oracle-Java语言规范(JDK8)**-_https://docs.oracle.com/javase/specs/jls/se8/html/index.html_

- **JVM系列之:JVM是怎么实现invokedynamic的? | HeapDump性能社区**-_https://heapdump.cn/article/3573623_

- **Java 虚拟机：JVM是怎么实现invokedynamic的？（上）**-_https://cloud.tencent.com/developer/article/1787369_

- **Java 虚拟机：JVM是怎么实现invokedynamic的？（下）**-_https://cloud.tencent.com/developer/article/1787371_

- **【stackoverflow】What is a bootstrap method?**-_https://stackoverflow.com/questions/30733557/what-is-a-bootstrap-method_

- **Java中普通lambda表达式和方法引用本质上有什么区别？**-_https://www.zhihu.com/question/51491241/answer/126232275_

- **理解 invokedynamic**-_https://juejin.cn/post/6844903503236710414_

- https://www.cnblogs.com/wade-luffy/p/6058087.html

- **09 | JVM是怎么实现invokedynamic的？（下）-深入拆解Java虚拟机-极客时间**-_https://time.geekbang.org/column/article/12574_

**往期回顾**

1. [得物App白屏优化系列｜网络篇](http://mp.weixin.qq.com/s?__biz=MzkxNTE3ODU0NA==&mid=2247529578&idx=1&sn=e50ece6abb272ee227da004d3d1713d0&chksm=c1612135f616a82390111cce4424d0b423f029cf07631526f7aa7c560eb9c039f148a84efa7c&scene=21#wechat_redirect)

2. [利用多Lora节省大模型部署成本｜得物技术](http://mp.weixin.qq.com/s?__biz=MzkxNTE3ODU0NA==&mid=2247529530&idx=1&sn=f8c295fbe9f91f4f0b857a070f98dec4&chksm=c1612165f616a87370da756cd7f356a27a6aaa1ee2f4b7b8e257ea46daf9bb5d2c7a860e7d42&scene=21#wechat_redirect)

3. [得物Flink内核探索实践](http://mp.weixin.qq.com/s?__biz=MzkxNTE3ODU0NA==&mid=2247528003&idx=1&sn=e472382b4d3cb47f3faa62f8627dabb5&chksm=c1613b1cf616b20a5782a7f8d0782770060c70864b47a5225e713299c83cc192f0cd6b4ba8f8&scene=21#wechat_redirect)

4. [链路级资损防控之资损字段防控实践｜得物技术](http://mp.weixin.qq.com/s?__biz=MzkxNTE3ODU0NA==&mid=2247527972&idx=1&sn=24fea420cd6d981b3bba7c6d9bd7da9f&chksm=c1613b7bf616b26df9d51705e58f99568d9aa81d06a91c3cfa7e2eb9250686146c134725ec76&scene=21#wechat_redirect)

5. [B端常用交互方式的量化及优化实践和指引｜得物技术](http://mp.weixin.qq.com/s?__biz=MzkxNTE3ODU0NA==&mid=2247527932&idx=1&sn=5fd72c05eec90d5286f3e859427e53a4&chksm=c16138a3f616b1b572bacbe4e9430899d2c21bcff5b5fed7caaee37d5f2312037edbe8972c44&scene=21#wechat_redirect)

文 / 羊羽

关注得物技术，每周一、三更新技术干货

要是觉得文章对你有帮助的话，欢迎评论转发点赞～

未经得物技术许可严禁转载，否则依法追究法律责任。

“

**扫码添加小助手微信**

如有任何疑问，或想要了解更多技术资讯，请添加小助手微信：

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

JVM9

Java19

JVM · 目录

上一篇解密JVM崩溃(Crash)：如何通过日志分析揭开神秘面纱｜得物技术

Reads 1651

​

Comment

**留言 3**

- 蹦蹦

  浙江13小时前

  Like

  太巧了，最近在看lamda和函数式相关的内容，debug看源码时看到方法句柄的内容一脸懵逼。这篇文章把相关内容全串起来了，太好了![[色]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

- 玖陆

  北京16小时前

  Like

  有引用链接的, 规范的技术文档 , 现在去得物 下单去![[呲牙]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

- 升宇

  上海16小时前

  Like

  可以啊，牛逼牛逼

已无更多数据

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/AAQtmjCc74AlsHDtoVyU8hqzNTGS26fV9PmHAcZ8uib1GWNJibIuBiavPdAXw9IOzjlEAYRJUNjOEme5geMNPoZ1Q/300?wx_fmt=png&wxfrom=18)

得物技术

8319670

3

Comment

**留言 3**

- 蹦蹦

  浙江13小时前

  Like

  太巧了，最近在看lamda和函数式相关的内容，debug看源码时看到方法句柄的内容一脸懵逼。这篇文章把相关内容全串起来了，太好了![[色]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

- 玖陆

  北京16小时前

  Like

  有引用链接的, 规范的技术文档 , 现在去得物 下单去![[呲牙]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

- 升宇

  上海16小时前

  Like

  可以啊，牛逼牛逼

已无更多数据
