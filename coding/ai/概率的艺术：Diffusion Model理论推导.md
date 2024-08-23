

CV开发者都爱看的 极市平台

 _2024年03月27日 22:01_ _广东_

↑ 点击**蓝字** 关注极市平台

![图片](https://mmbiz.qpic.cn/sz_mmbiz_gif/gYUsOT36vfoGnnrz1tAAA0UNPxQaxTGM7WpEmJcx4UPcaibm3FUZpY6Jzicic0sic2fzYicoogTIzwnbNicicos17UGibw/640?wx_fmt=gif&wxfrom=13&wx_lazy=1&wx_co=1)

作者丨Fugtemypt@知乎（已授权）

来源丨https://zhuanlan.zhihu.com/p/680813516

编辑丨极市平台

**极市导读**

 

理清Diffusion Model背后的理论基础。 >>[](http://mp.weixin.qq.com/s?__biz=MzI5MDUyMDIxNA==&mid=2247564839&idx=6&sn=ff2c1ca613b52f97af8ce4ddebe06146&chksm=ec1d13dedb6a9ac8bd166c8f79e62148465e78591f905d184136a146f58cbe88e25d6e2752dd&scene=21#wechat_redirect)[加入极市CV技术交流群，走在计算机视觉的最前沿](http://mp.weixin.qq.com/s?__biz=MzI5MDUyMDIxNA==&mid=2247618084&idx=1&sn=981fa2ed41e2eda97799ae098b7c8907&chksm=ec1de3dddb6a6acb719081ffef32f72b72e7d6416f3504bf7049594b9f34f2d6cf570654ae21&scene=21#wechat_redirect)

## 写在前面

大约在一年前的秋天，我听朋友们提到：一款名为StableDiffusion的AIGC生成图片工具正在席卷画师界。彼时的我也尝试着在本地部署了一下这个项目，并且拿它画了些奇妙的图片（doge），效果确实优秀得有些震撼。不过那时的我还没有想到去深究这款工具背后的知识，后来做科研时几次想过要彻底地学习一遍，却也都因为各种事情耽搁了。直到一年后的今天，赋闲在家，静下心来拿起纸笔推导一些数学，理清这强大工具背后坚实的理论基础，或许也是一种不错的消遣。

本文只是作者的一份学习笔记，主要内容大部分来自以下几位知乎大佬的文章：

一文解释 Diffusion Model (一) DDPM 理论推导(https://zhuanlan.zhihu.com/p/565901160)

一文解释 Diffusion Model (二) Score-based SDE 理论推导(https://zhuanlan.zhihu.com/p/589106222)

如何理解扩散模型中的SDE？(https://www.zhihu.com/question/616179189/answer/3230281054)

以及宋飏博士关于Score-Based SDE的论文：

https//arxiv.org/abs/2011.13456

以及Google Research关于Diffusion Model框架的工作：

https//arxiv.org/abs/2208.11970

## 概念介绍

扩散模型（Diffusion）是一种基于马尔科夫过程，通过多步加噪/去噪过程实现内容生成的模型。它和我们之前学过的VAE十分类似，都需要经过原图片->高斯噪声->生成图片这三步过程。

> 注：从我的个人感受来看，这几种生成模型的big-picture其实都差不多：它们都经历了原图->白噪声->新图的流程，且它们在数学上都是一个从某个概率分布里采样出实例的采样器（这个后面会解释）。而Diffusion相比于先前模型的最大区别可能就是引入了**微分**的思想，使得生成过程更加细粒度（精细）和可控。

一位dalao(https//lilianweng.github.io/posts/2021-07-11-diffusion-models/)制作了几种生成式模型的对比图如下：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

在实践中，Diffusion Model有很多种实现方法，例如基于VAE压缩latent space（隐空间）思想的Stable Diffusion，或者不压缩latent space（也就是让隐空间大小和原图保持一致）的DDPM。我们这篇文章不去讨论具体的实现细节，只是从general的理论角度梳理一遍Diffusion的整个过程。不过如果对这个过程理解得足够透彻，再去看具体的实现方式想必也不会感到困难了。

## 流程概述

Diffusion Model的训练流程可以大致分为两个阶段：第一阶段是加噪阶段，给定一张原图  ，我们会对其进行  步加噪，最终将其变为高斯噪声  ；第二阶段是去噪阶段，对于刚加完噪的  ，我们会对其进行  步去噪，最终尝试将其复原成原图  。在训练结束进行推理时，我们不进行第一阶段，而是取一个白噪声然后进行第二阶段，最后 “复原” 得到的图片就是模型生成的图片。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

这是宋飏博士论文里的插图，我们暂时可以忽略SDE，只看流程即可

> 注: 一张图片是怎么和随机变量产生联系的? 假定原图是一个  的张量，那么它在每一个位置  的值都是一个确定的数字，看起来好像和随机变量没什么关系。但是不要忘了，我们训练的对象并非一张图片，而是一整类图片。假如我们有  张人脸的图片（大小都是  )，那么现在的每一个位置  上就有  个数值，可以视作一个概率分布。这些概率分布事实上就代表了这类图片的所有特征，例如人脸在靠近屏幕中间的位置  通常是肉色的，那么这里的概率分布方差  就会较小，且RGB三通道的均值  拼起来会产生 “肉色”。

绝大多数的Diffusion Model都是基于上述流程实现的，它们的区别通常只在于“加噪”和“去噪”的方式不同。当然，仅凭抽象的框架很难深入推导模型的内容，在这里我们尝试具体一些，以比较流行的DDPM（Denoising Diffusion Probabilistic Models(https//arxiv.org/abs/2006.11239)）为例进行一些推导。

### 加噪过程/前向过程

假设我们的加噪过程已经进行了  步，现在要进行  的加噪。一个自然的想法是把图片和噪声构造成一个线性关系:

（）

其中＜

递归的形式不太好解，我们考虑能不能写成通式:

不妨假定  相互独立，根据正态分布的可加性，上式可以写成

其中 . 这个式子有点不好看，因为里面既有  又有  ，难以化简。我们是否可以尝试把这两个参数归一化，也就是令  ，以此来消掉其中的一个变量呢? 既然我们希望最终得到的图片  ，那么归一化显然不会影响我们达到这个目标，故我们尝试归一化，此时有

于是我们就可以令  ，从而  ，于是(3)式就可以改写成

（）

同理，因为有了  ，我们就可以将(1)式也改写成

（）

将上面两个式子表示成概率分布的形式，我们就有

整个前向过程可以表达为

### 去噪过程/反向过程

在去噪过程中，我们希望实现从纯噪声  到真实图片  的转换，为此我们同样考虑每一步  ，但与前向过程不同，此时我们不把它建模成一个简单的线性关系（但仍然建模成正态分布)，而是让模型通过神经网络的参数  去学习每一步去噪过程的期望和方差

事实上这看起来就是学了前向过程的反向分布，我们后面也会再提到这一点。

将所有步累积起来得到整个随机过程的概率分布

> 注：这是整个随机过程  的概率分布，不是  的概率分布。想要求  的分布，我们需要枚举所有可能的  并将它们所对应的随机过程  的概率求和（积分），这也就是所谓的全概率公式。

从概率论的视角看，我们模型的计算结果应当是  的分布  (就是上述积分之后的结果），而我们的优化目标就是希望  尽可能接近  的真实分布（比如人脸）。不过我们并不知道真实数据  的分布，但我们手里有若干个从  的真实分布里采样出来的样本  ，那么我们自然希望这些样本在我们建模出来的分布里能够具有尽量高的概率 (举个例子，假如我们有很多张人脸，但建模建成了熊猫脸的分布，那么我们的样本在模型里出现的概率就会很低)，也就是我们希望求出

(事实上这就是最大似然估计的思想，这里只是又用白话叙述了一遍)

现在我们考虑通过上面提到的积分方法求出  的表达式

我们知道，KL散度可以视作对两个概率分布度量距离。考察上面式子中的三项，第一项是在反向过程的最后一步最大化  ；第二项是在反向过程的第一步试图拉近  和  ，但这一步没有可训练的参数；第三步则是在反向过程的所有中间步都试图拉近  和  。

> 注: 可以理解为，我们希望模型在反向过程中能够尽可能准确地模拟出前向过程的每一步。我们前面提到过  ，这就是说前向过程的每一步都是一个正态分布，这个正态分布的均值包含了我们希望获得的原图特征的一个部分（微分）。所以事实上我们希望准确预测的是前向过程每一步的均值，这一点马上就会在后面提到。

现在我们已经可以通过梯度下降的办法去优化上面的式子了 (事实上我们优化的是  的一个下界，但这和对原目标函数的优化可以视作是等效的) ，但是有一个小问题：在第三项计算中我们每次需要同时求三项  才能算一个KL散度，这是否有点复杂? 能否进行一点优化?一个直接的想法是利用贝叶斯公式把  和  都改成同向的（比如都改成  ），根据

我们可以推得（仿照上面过程即可，这里就不推了)

现在我们就把第三项优化成了每次只需要算两项的形式，不过注意到  和前向过程是反着的，我们并不知道它的分布是什么，需要用贝叶斯公式算一下

现在我们再把前面建模好的  带进去，前面的 【注】提到我们希望估计的是均值，而方差并不携带信息，所以我们不妨就把  当做和 的方差完全一样 (因为优化它对我们学习图片特征没有任何帮助，所以不如不优化)。

现在我们再次考虑KL散度的计算，根据公式

可以计算得到

算了这么一堆，实际上我们只是从数学角度严格地证明了【注】中的观察是正确的。

现在考虑怎么学习这个  ，当然我们可以直接硬学，但是如果能做一点分解然后学其中的一个部分或许会更简单。

我们已经知道

在逆向过程中，我们的  是  的函数，所以上面的  那部分可以直接拿下来用，需要通过神经网络拟合的只有  那部分，也就是说

那么我们就可以算得

也就是说，我们最终学习的还是拟合原图  。

> 注：既然都是拟合原图  ，那Diffusion岂不是和之前的生成模型没有区别了? 但是值得注意的是: 我们是从  出发预测  的，而不是从啥也没有或者白噪声出发预测  的。从数学上来看，这里就是Diffusion和其他生成模型的最本质区别：它将整个生成过程进行了微分化，每次只预测一段过程而非整个过程。从直观上来看，这大大减小了预测的难度（提升了预测的质量)。后面我们还会从更加数学的角度分析这样做的优势。

但是这仍然没有达到最简形式，因为我们在前向过程中知道  ， 这意味着我们可以将  用  来表示，于是就有

那么我们又有了一种新的预测方法：预测噪声! 也就是建模

然后优化目标就变成了

预测噪声和预测  在数学上可以互相转化，但是实际操作中发现预测噪声比预测  表现要更好。

> 注1: 但是从数学直觉上来看预测噪声显然更好，因为  是一个相当复杂的分布，而噪声只是一个正态分布。
> 
>   
> 注2: 这里有的朋友可能会疑惑: 我们绕了一圈难道就是学了个正态分布生成器吗? 那我不用学不也可以生成正态分布? 请注意: 我们这篇文章里所有黑体的字母都代表一个具体的样本值，而不是一个随机变量。我们预测的不是随便一个噪声值，而是从具体值  变换到具体值  所加的那个具体值  。诚然这个值是从标准正态分布  里采样出来的一个样本，但我们根本不关心这个分布，我们只关心这个具体的样本值。也就是说，这个模型本质上是**一个能够根据输入，从标准正态分布里采样出对应值的采样器**（回忆一下GAN和VAE，其实它们也是一个**概率分布的采样器**，只是采用了不同的学习方式）。

整体的训练和推断流程如下图所示。这里的每一步训练都是随机选择一个  进行的，实际上只要保证  的均匀分布应该就可以了。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

## SDE框架下的建模

上面的内容以DDPM为例，基本概述了Diffusion的整个流程。现在我们希望站在一个更高的角度，从SDE（随机微分方程）的角度对Diffusion的流程进行一个建模。

我们将前向过程认定为一个可微的随机过程（Ito过程，名称来源于日本数学家伊藤清，这部分理论主要由他完成)  ，也就是说它满足

其中  代表维纳过程（用于描述布朗运动的随机过程）

> 注: 想要完全理解这个式子需要补充一下随机过程和随机微积分的知识，这里提供一种直觉理解的方式：一个随机过程关于时间通常是不可微的。这是因为，可微的定义是  ，但随机变量的  在  的时候仍是一个随机变量，它有小概率变得非常大，小量记号  不再适用。因此我们需要引入一个适应随机变量微分的微分量，而维纳过程作为最简单的随机过程，能够很好地满足我们的要求（这就是说，我们把正态分布视作了随机变量世界中的 " 1 " )。

于是我们重新考虑前向过程每一步的计算，离散形式下的每一步是  ，而连续形式下的每一步就应该是  ，按照式(25)的格式可以写作

其中  。

> 注: 如果没有学习过维纳过程，需要知道  （建议还是学习一下)。

那么前向过程的概率分布就可以写作

根据贝叶斯公式我们同样可以得到反向过程的概率分布

由lto引理 (随机变量函数的泰勒展开) 我们知道

于是上式可以改写为

注意到反向过程中  才是已知，而不是  。不过由于  ，直接替换即可，同时去掉小量

写成概率分布的形式就是

转换成Ito积分的形式就是

在上式中，  和  是我们自己定义的系数，唯一的未知量就是  ，我们使用神经网络拟合的也正是这个量。从数学上看，它相当于给概率空间的每个点都分配了一个梯度，沿着这个梯度一直走就能到达  的最优化目标。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

任何不同的Diffusion模型，其实都只是在式(32)中采用了不同的 ff 和 gg 实现的结果。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

公众号后台回复“数据集”获取100+深度学习各方向资源整理

![](http://mmbiz.qpic.cn/mmbiz_png/gYUsOT36vfodx8Y8MkYkPq45UCaZVlRdibEYl3ibSGKC8xaOszOvswbqFLApfHSNzuMfG5sibBdbYcTDct0SFPUjw/300?wx_fmt=png&wxfrom=19)

**极市平台**

为计算机视觉开发者提供全流程算法开发训练平台，以及大咖技术分享、社区交流、竞赛实践等丰富的内容与服务。

937篇原创内容

公众号

_极市干货_

**技术专栏：**[多模态大模型超详细解读专栏](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=MzI5MDUyMDIxNA==&action=getalbum&album_id=2918280735411683334#wechat_redirect)｜[搞懂Tranformer系列](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=MzI5MDUyMDIxNA==&action=getalbum&album_id=2090301627206303744#wechat_redirect)｜[ICCV2023论文解读](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=MzI5MDUyMDIxNA==&action=getalbum&album_id=3021109573835554818#wechat_redirect)｜[极市直播](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=MzI5MDUyMDIxNA==&action=getalbum&album_id=1425604183083892737#wechat_redirect)

**极视角动态**：[欢迎高校师生申报极视角2023年教育部产学合作协同育人项目](http://mp.weixin.qq.com/s?__biz=MzkwMjIxOTM0NA==&mid=2247499703&idx=1&sn=26efecffec277dfd4b55a0b7482cb244&chksm=c0aa6c88f7dde59e52d7d7ba295868e1209692e576c00a9952502cfdf3337cb64daa8c13b048&scene=21#wechat_redirect)｜[新视野+智慧脑，「无人机+AI」成为道路智能巡检好帮手！](http://mp.weixin.qq.com/s?__biz=MzkwMjIxOTM0NA==&mid=2247499654&idx=1&sn=5e87643e93831c4ce5014aae869957f8&chksm=c0aa6cb9f7dde5afa601b0ce3343e6fb845ff752d5451972ccfd276525ca8028ce7471ee15e3&scene=21#wechat_redirect)

**技术综述：**[四万字详解Neural ODE：用神经网络去刻画非离散的状态变化](http://mp.weixin.qq.com/s?__biz=MzI5MDUyMDIxNA==&mid=2247651748&idx=1&sn=1babe6c6a4f1adff434b482e68532518&chksm=ec127c5ddb65f54b9c086a5048b252084dca44006b08ec1e7a169006b94048f5d24091494443&scene=21#wechat_redirect)｜[transformer的细节到底是怎么样的？Transformer 连环18问！](http://mp.weixin.qq.com/s?__biz=MzI5MDUyMDIxNA==&mid=2247647297&idx=1&sn=f5ddd4238cb61f6b6f2eb50d9fbd8680&chksm=ec126db8db65e4ae95414f033fd2e465f1c56e8b3d8c41f57c0fb0d5f1d4ba20c53dbbdbbb4c&scene=21#wechat_redirect)

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**点击阅读原文进入CV社区**

**收获更多技术干货**

阅读原文

阅读 5390

​