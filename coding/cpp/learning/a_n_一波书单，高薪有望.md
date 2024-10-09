原创 里缪 CppMore
_2023年10月27日 19:59_ _美国_

桂花浮玉，梧桐叶落，堪堪秋意浓。
因思上次书单更新乃是雪季，时秋已二转，犹未见新，理当出一期了。\
本期多为22/23年间的书籍，领域多样，深浅不一，内容颇新。

**_C++ Software Design_**
Klaus Iglberger | 2022

谈论 C++ 特性的书籍如满天繁星，但是讲如何使用 C++ 设计软件的书却指不多屈。

面向对象软件开发，设计层面上的问题使用设计模式解决，非设计层面上的问题使用架构和算法解决。设计模式源于建筑设计领域，这些思想随后被利用到软件开发领域，至 GoF 在 1994 年发布著作 《设计模式》，后便几乎没有 C++ 相关的设计模式书籍。

珠玉在前，在 C++ 领域纯讲面向对象设计模式已经没有意义，深度广度都难有突破，反而是其他语言中的相关书籍层出不穷。

设计模式是用来解决面向对象的缺陷，也就是 Subtype Polymorphism 中的问题，而 C++ 并非是单范式语言，它还支持泛型编程、面向过程编程、函数式编程。因此，后续有关优化提高设计模式的技术，都是将设计模式和其他范式结合起来，从而避开面向对象的缺陷，像是 Andrei 的经典书籍 _Modern C++ Design_，便是使用泛型编程所支持的 Parametric Polymorphism 来抽象化设计模式的表达。再后面的一些书籍与相关内容，像是 _Design Patterns in Modern C++20_ 和本人连载的一些文章，也是充分利用其他范式的特性在避开原有问题，此时的设计模式已经不是原本的设计模式。

换个范式，思想便会跟着变，有些问题不再是问题，有些习已为常却又成了问题。不过，C++ 支持多范式，完全可以混合编程，充分利用各种范式的优点。

总之，这本书值得一看，毕竟相关书籍寥寥无几，能够让你学习如何结合 Modern C++ 与传统设计模式。

**_C++ Best Practices_**
Jason Turner | 2022

条款书，篇幅短小，内容不多，对 Modern C++ 熟悉的老鸟可飘过，都是一些比较常见的东西。
作者是 Youtube  _C++ Weekly_ 的作者，此书内容也像是频道内容的合订本。
对照目录，如果有些技巧你恰巧不知道，也可以抽空快速浏览一下。

**_Beautiful C++_**
J. Davidson, K. Gregory | 2022

也属条款书，各章节由一些开发经验组合而成。
告诉你不要做什么，建议你应该怎么做。就算不看本书，很多内容大家也都知道，不过依然会有一些章节恰好是你不知道的，那么同样适合快速浏览一下，补全相关知识。
总之，本书比较适合 C++ 初/中级学习者。

**_C++ Core Guidelines Explained_**
Rainer Grimm | 2022

本书同样包含许多经验之谈，作者也是老面孔，modernscpp 的作者 Rainer Grimm。
主题覆盖很广，包含资源管理、并发、错误处理、泛型编程等等内容，同时也含有太多过于基础的东西，一些内容与前两本条款书存在交叉，所以最多只属于中级层次的书。\
一般来说，如果一本技术书籍覆盖的主题特别广泛，但作者却只有一人，基本可以断定是取广度而失深度。因为这里面的很多主题，单独拿出来都可构成一本书，寥寥数语怎能讲透？
但侧重广度的书，只要不是字典书，还是值得一读的。此类书中一般包含作者对各个主题的理解，以及作者在每一部分的开发经验，是作者自己的内容，这就够了。\
此书中文版今年刚出，名为 《C++ Core Guidelines 解析》，群友负责了部分翻译，推荐一下。

**_C++ Initialization Story_**
Bartlomiej Filipek | 2023

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

专门讲解「初始化」的 C++ 书籍，作者也不陌生。

此类书一般都是小大靡遗，各种概念纤悉洞了，聚焦于庞大知识库的某一局部网络进行探析。在阅读之前，很多知识点你在其他地方也已经掌握，它的作用在于为你补充可能遗漏的细枝末节。

总体来说，此类书籍阅读难度不大，也无需从头到尾细看一遍，对哪个部分的初始化细节感兴趣，跳过去专项查看即可。

# **_The Art of Writing Efficient Programs_**

Fedor G. Pikus | 2021

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

关键词是「高效」，所以这本书跟程序性能有关。

涉及性能测试的方法、CPU 与内存的相关知识、并发并行、编译器优化等等内容，覆盖面广，深度尚可，还算不错。

本书适合对性能感兴趣的朋友阅读，需要具备一定的 C++ 基础。

# **_Software Architecture with C++_**

A. Ostrowski, P. Gaczkowski | 2021

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

架构方面的书不好写，需要很深的经验，是以此类书籍寥寥无几。

本书也是，虽然名为软件架构，但是讲架构时都是讲概念，蜻蜓点水，具体怎样实现不见写，反倒是不打紧的地方才有代码。

像是概念堆砌之物，对于概念科普有一定积极作用，问题在于，学完你也不知道怎么用。

# **_Large-Scale C++: Process and Architecture, Volume 1_**

John Lakos | 2019

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

本书讨论了大规模 C++ 项目的架构、开发过程，给出了关于如何组织和管理大型 C++ 项目的经验。

1000 来页，一人编写，就已经说明很多内容是概念拼凑，没能做到详略得当，致使篇幅非常庞大，阅读起来需要一定的 C++ 基础。

如果你没有构建大规模项目的经验，读读此书还是能有一定帮助的。

# **_Modern CMake for C++_**

Rafal Swidzinski | 2022

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

学个 C++ 还得学 CMake，难度不亚于学习一门新语言，但都在用，不学怎么行？

本书全面地介绍了 Modern CMake 的编写技术，若想系统学习一下，可以直接入手，当作一门新的编程语言来学习。

书是永远读不完的，建议是有哪部分的需求了，查找相关的编写手法，不要上来就从头读到尾，长期递进完善。

# **_C++ High Performance 2nd ed._**

Bjorn Andrist, Viktor Sehr, Ben Garney (Foreword) | 2020

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

又是雷声大、雨点小的性能书。

主题覆盖也相当广泛，移动语义、错误处理、性能测试、数据结构、算法、Ranges/Views，内存管理、编译期编程、惰性评估、并发、并行…… 如果真是一本比较深入的书，仅是两人编写不可能覆盖如此多的主题，许多都是其他地方也能看到的内容。

这种书一般都颇有厚度，然而独有的观点不多，只适合查漏补缺。

**_Functional Programming in C++_**

Ivan Čukić | 2018

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

讲函数式编程的 C++ 书籍，仅此一本。

Modern C++ 的简易性，许多都源自函数式编程思想。在写新特性相关的文章时，很多时候都能见到「越来越像 Python 了」、「这不是 Rust 吗」等等这样的评论，其实范式就那么几种，只要背后的思想相同，展现出来的语法也都是大差不差。

如果你感觉 C++ 越不像 C++ 了，那只是你默认它是 OOP 和 GP。C++ 是多范式语言，每个时期都有不同的侧重点，最早是 OOP，随后是 GP，现在则是 FP。

偏见源于缺少了解，本书虽说不够深入，有些比喻也略失本意，但却能让你整体地了解一下 FP 相关概念在 C++ 中的表达形式。

**_The Book of Modern C++_**

2023

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

不用多说了吧！

C++ 领域目前最新的书，含 C++23，多位优秀作者的文章合集，内容致广大而尽精微。虽然近些年也推荐了不少书，但也少有质量能比得上这本合集的，就是不能出版，只供各位小范围内阅读。

这只是第一版，第二版会进一步完善补充，应该年底能出来。

第二版属于打印版，供大家自行打纸质版收藏阅读。第一版先不要打印。

# **《順勢溝通》**

張忘形 | 2022

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

这是一本不错的入门级沟通书，不似心理学那样深入，也不像话术书那般浅显，作者是以道在讲术。

说话和沟通只是一个媒介，真正重要的是理解对方的信念系统，人所说的话反应了自身的信念。信念是服务于自我的，而不是事实。

书中从过程、界限、情绪、人格特征和互动关键词这几个方面介绍了沟通的概念、原理与应用。沟通是寻求共识，而不是彼此控制，很多烦恼都是人际关系的烦恼，许多悲剧都是不会沟通的结果。

好好说话，好好沟通。相信通过本书，能让你更好地理解自己，理解他人，与人更好地交流。

# **《大腦先生的一天》**

Marbles: The Brain Store著 王婉卉译 | 2017

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

一本关于脑神经科学的小书。按一天内的不同时间，分成早上、白天、晚上三个 Part，每 Part 再分成各个小章，以生活中的例子引出各种有趣的脑科学概念。

比如作者在 7:00 AM 提到晨型人和夜型人，谈到起床一小时后，晨型人比夜型人拥有更高的皮质醇浓度，因此晨型人清醒得更快。生理时钟通过增加「褪黑激素」荷尔蒙或降低体温以使人想睡觉，青少年体内的褪黑激素和体温下降都比一般人要晚，所以晚上更有活力，也更容易当个夜猫子，早上就是起不来。

在 8:05 AM 提到任何类型的记忆都会经历三个基本过程：编码、存储、提取。大脑中的海马体结构负责将事实塞进脑袋，杏仁核结构是大脑的情绪学习中心，掌管着恐惧和欲望，能够辅助记忆，长期记忆可以透过联想提取和线索提取。

在 1:00 PM 提到拖延并不是能力上的缺陷，而是任务要求和动机之间的落差。可以通过欺骗心智，只要完成就奖励自己，来减少拖延。

对抗下午的疲倦，最好的办法是增加自我成就感。

腺苷在人思考和行动时会累积，睡眠能够让大脑自行清除腺苷。咖啡因之所以有提神的效果，部分原因就是咖啡因分子能阻挡腺苷对大脑的影响。

……

不是什么烧脑的书，相当于科普读物，娱乐放松时翻个一会就能读完。

# **《遥远的救世主**\*\*》\*\*

豆豆 | 2005

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

非常经典的一本小说，讲的其实就是知行合一，达到道与术的统一。

全书的精华在五台山论道这一部分。借主角丁元英和智玄大师之口谈觉性，佛便是觉性，每个人都有觉性，但并不是每个人都能觉悟，人有生老病死，觉性没有实体，可不生不灭。每个人都有觉悟的机会，有人忽然想通了，便是觉醒，人需要不断修行，不断实践，因为佛法无量，是没有边界的，觉性也是无量的，并不存在圆满之说。

简单来说，人都有可能觉悟，但只有生存受到严重威胁时，苦过、疼过、痛过，才会产生觉悟的机会。此时将从内部击碎你从小受环境影响而建立的思维之墙，把旧系统推导重建，以获新生。此即为觉悟。你之所以不改变，只是因为当前的处境还过得去，还没被逼到觉悟的境地。

非觉悟者的人生没有遇到一些大的劫难，或是彼时有所觉悟，劫难过后享受舒适区的人性又占据了主动权。若也想达到觉悟的境界，须靠修行。修行的人懂得许多道理，但缺乏实践，此时不能说懂得道理，只能说是听过这个道理，知与行是一体的，不存在只是知道却做不到的情况，做不到就是没有真正知道，此即所谓之有信无证。此时只是模仿已经悟到这个道理的人的习惯，来假装达到人家的境界。

悟道是从下而上的顺序，觉悟者在波涛汹涌的生活中被迫产生了一些思考，经由总结、归纳，抽象成为一个道理。他们在悟到道理之前就已经经历过，被动做到了实践。修行者没有经历这些生活的折磨，只是从书上或是其他地方听到过这些道理，想要理解，却又没有经历，当然是似懂非懂，自然也无法由内而外地改变自己的行为，所以懂得了许多道理也依旧过不好这一生。破局之法就是去经历，在无数经历中去重新思考这些道理，只要补上经历，不论是修行还是觉悟，最后达到的境界是一样的。此为觉者由心生律，修者以行制律。

全书就是在告诉大家如何走出当前的困境，实现自救。救世主并不遥远，救世主就在你自己的心里，只是他们被环境所尘蔽，当前没有显现而已。

我们能够做的就是在知行合一中不断修行，不断完善自我，当自己的救世主。

书中丁元英其实一直也没有达到知行合一的境界，他空有满腹才华，知晓问题所在，却只是在怒指乾坤错，逃避问题。等到结尾，因为女主的事才算又从劫难中真正完善自身。

作者另有一本书 《天幕红尘》，虽然不是同一主角，却相当于丁元英大成之后的样子，也可一读。

# **《挽救计划**\*\*》\*\*

Andy Weir | 2021

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

科幻小说，火爆的《火星救援》也是出自这位作者之手。

小说描述了地球人 Grace 和波江座40星系外星人 Rocky 为解救各自受噬星体威胁的母星的故事。

全文双线交叉描写，一线是当前，一线是回忆，穿插揭开层层迷雾，再收线合一。作者很擅长设计悬念，每章结尾都能拉起读者读下去的兴趣，整体故事发展就是不断遇到问题，解决问题的过程。

有趣的是具体描写了外星文明的外貌，并详细描述了两个不同文明之间是如何从零建立沟通方式的。

故事层层递进，逻辑严密，扣人心弦，值得一读。

# **《****版面设计基础****》**

视觉设计研究院著 张喆译 | 2004

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

这是一本版式设计方面的老书，对业内人士来说属于入门书，对外行来说作为科普读物也不错。

书中介绍了一些版式设计的基本知识，主要包含样式的八要素：视觉度、图版率、文字跳跃率、图片跳跃率、网格拘束率、版面率、字体构成、字体印象，形态的八原则：明确主题、分开副主题、群化、明确关系、整理流向、空白运用、抵制四角、利用版心线，以及创造形态的一些诀窍：节奏、对比、重点、比例、平衡、融合。

本书只有一百来页，图文并序，内容翔实，思想也不会过时，可以一观。

**_Enhancing English Language Teching: Harnessing the Power of ChatGPT_**

Abolfazl Ghanbari | 2023

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

ChatGPT 火爆之后，相关书籍如雨后春笋般涌出，但基本都是蹭热点之作，概念堆砌，穿凿附会，值得读的实在不多。

这本书是一个小册子，一篇文章的长度，讲解如何借助 ChatGPT 提高英语能力，也没含多少技巧，但可举一反三，将整体思路推到其他领域的学习，比如编程。

对于知识比较固定的领域，借助 ChatGPT学习更有效果。类似英语，这种领域本不适合自学，因为缺乏有效的反馈途径，学习者没有办法刻意练习。之前需要请老师帮你建立及时反馈，现在则可以用 ChatGPT 完全替代，英语是其母语，指导普通学习者完全没有问题。

对于编程，初学者也可以借助它来学习，但若是询问稍微深入的内容，反馈就并不一定正确，完全使用它来学习可能会适得其反。

**_The Art of Asking ChatGPT for High-Quality Answers_**

Ibrahim John | 2023

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

这应该是 ChatGPT 相关质量最高的一本书，主要介绍如何编写咒语（Prompt）以得到更加精确的回答。

全书主要围绕TIR（Task, Instructions, Role）这个模板来介绍在各种情况下如何编写咒语才能得到想要的结果，有一定参考意义。

其实就算不学习这些技巧，一直询问、反馈、调整、优化也能够得到想要的结果，因而只是参考。

除了以上这些，还有一些群友推荐的书籍，如下列出：

- C·亚历山大著，赵冰译，《建筑的永恒之道》，2004. 出版比较早讲建筑的书，对软件架构比较有帮助。

- D. Hofstadter，E. Sander著，刘健，胡海，陈祺译， 《表象与本质》，2019. GEB 作者新出的书，探寻人类智能的本质。

- Robert M.Pirsig著，张国辰译，《禅与摩托车维修艺术》，2011. 哲学。

- \[德\]艾克哈特·托尔，\[德\]埃克哈特·托利，《新世界》，2012. 哲学。

- 程炼，《伦理学导论》，2017. 哲学书，书如其名，主要有三部分，分别讲伦理学的挑战、基础和理论。比较好读，特别是第一部分就提出了很多伦理学本身的挑战，比如相对主义，科学主义等等，有助于理解哲学思考的过程。

- Alex xu, _System Design Interview - An Insider's Guide (Volume 1 and 2),_ 2020(Volume 1), 2022(Volume 2). 系统设计。

- Stanley Chiang, _Hacking the System Design Interview: Real Big Tech Interview Questions and In-depth Solutions,_ 2022. 系统设计。

- Stefan Jansen, _Machine Learning for Algorithmic Trading_, 2020. 算法交易。

- Kazuo Inamori, _A Passion for Success_, 1995. 商业管理，人生哲学。

- Daniel C. Dennett, DARWIN'S DANGEROUS IDEA, 1996. 达尔文思想对其出现前的思想的冲击和产生的辩论。

- Richard W. Hamming, _The Art of Doing Science and Engineering_, 2020. 科学和工程领域的学习方法和思维方式。

一本书能够有核心论点，围绕中心发枝散叶，全文逻辑自洽，言之有物，环环相扣，而非胡拼乱凑，矫揉造作，就可算是一本好书。

知识似茂林，我们都是在林间拾取叶子的人。

愿此书单能让你更好地成长。

书单4

书单 · 目录

上一篇2021冬季书单：别停下，往前走

阅读 1279

​

写留言

**留言 15**

- 我在呼吸和想你

  山东2023年10月27日

  赞1

  有资源分享吗

  CppMore

  作者2023年10月27日

  赞1

  无~

- Mr.Renᯤ⁶ᴳ

  四川2023年10月28日

  赞

  新书《C++ Core Guidelines解析》也不错

  CppMore

  作者2023年10月28日

  赞

  提了![[机智]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

- 梦短情长

  湖北2023年10月28日

  赞

  gp是什么缩写

  CppMore

  作者2023年10月28日

  赞

  generic programming

  4条回复

- cela

  中国香港2023年10月28日

  赞

  ![[强]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

- 📷全程不笑🏀

  广东2023年10月28日

  赞

  请问怎么加群？

  CppMore

  作者2023年10月28日

  赞

  等开放

- 哼哩🦈

  广东2023年10月28日

  赞

  十月底啦 想跟群友们团聚了![[嘿哈]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

  CppMore

  作者2023年10月28日

  赞

  计划中

已无更多数据

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/9XBBCfGaPEly5AtcdkIKicpv55w2VlBiaQ7TxtJneeDmgO3mxz4onND7rW2UjfSib9KC1FtBB4U6TupnBMfenoCgQ/300?wx_fmt=png&wxfrom=18)

CppMore

261511

15

写留言

**留言 15**

- 我在呼吸和想你

  山东2023年10月27日

  赞1

  有资源分享吗

  CppMore

  作者2023年10月27日

  赞1

  无~

- Mr.Renᯤ⁶ᴳ

  四川2023年10月28日

  赞

  新书《C++ Core Guidelines解析》也不错

  CppMore

  作者2023年10月28日

  赞

  提了![[机智]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

- 梦短情长

  湖北2023年10月28日

  赞

  gp是什么缩写

  CppMore

  作者2023年10月28日

  赞

  generic programming

  4条回复

- cela

  中国香港2023年10月28日

  赞

  ![[强]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

- 📷全程不笑🏀

  广东2023年10月28日

  赞

  请问怎么加群？

  CppMore

  作者2023年10月28日

  赞

  等开放

- 哼哩🦈

  广东2023年10月28日

  赞

  十月底啦 想跟群友们团聚了![[嘿哈]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

  CppMore

  作者2023年10月28日

  赞

  计划中

已无更多数据
