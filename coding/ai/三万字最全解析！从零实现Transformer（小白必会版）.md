

CV开发者都爱看的 极市平台

 _2024年03月29日 22:00_ _广东_

↑ 点击**蓝字** 关注极市平台

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

作者丨晚安汤姆布利多@知乎（已授权）

来源丨https://zhuanlan.zhihu.com/p/648127076

编辑丨极市平台

**极市导读**

 

拆解Transformer的工作原理以及实现方式。>>[](http://mp.weixin.qq.com/s?__biz=MzI5MDUyMDIxNA==&mid=2247564839&idx=6&sn=ff2c1ca613b52f97af8ce4ddebe06146&chksm=ec1d13dedb6a9ac8bd166c8f79e62148465e78591f905d184136a146f58cbe88e25d6e2752dd&scene=21#wechat_redirect)[加入极市CV技术交流群，走在计算机视觉的最前沿](http://mp.weixin.qq.com/s?__biz=MzI5MDUyMDIxNA==&mid=2247618084&idx=1&sn=981fa2ed41e2eda97799ae098b7c8907&chksm=ec1de3dddb6a6acb719081ffef32f72b72e7d6416f3504bf7049594b9f34f2d6cf570654ae21&scene=21#wechat_redirect)

## 前言

本文尝试从零开始实现一个Transformer，内容非常浅显易懂，看完不会的话博主倒立洗头 （倒立洗头机广告位：https://www.zhihu.com/people/da-da-12-30-83）。

这篇文章旨在**拆解Transformer的工作原理以及实现方式**，没有针对特定任务。在后续文章中，我们将尝试基于Transformer结构从零开始实现一个语音识别系统，从而对Transformer的具体应用以及基于深度学习的序列识别任务框架有一个更加深入细致的理解，链接如下。

晚安汤姆布利多：包教包会！从零实现基于Transformer的语音识别(ASR)模型(https://zhuanlan.zhihu.com/p/648133707)

博主水平有限，如有谬误欢迎各位指出，包括但不限于：描述不准确、语言不通顺、代码不正确、代码风格丑陋、缺少参考文献等。提前谢过！orz.

特别鸣谢以下用户的指正：

https://www.zhihu.com/people/liu-lin-gen-77

https://www.zhihu.com/people/jin-shu-zhi-guo-wang

https://www.zhihu.com/people/lan-an-91-12-84

https://www.zhihu.com/people/4thFever

https://www.zhihu.com/people/wo-lao-po-ye-tai-ke-ai-liao-ba

本文共有约30000字，阅读需约75分钟。公式较多，建议PC端阅读。

本文的基本结构如下：**（PC端点击左侧目录可直接跳转到对应章节）**

- Transformer简介：简单回顾Transformer的基本信息
    
- 上代码，Let's go!：手撸代码
    
- 总结：文章总结
    
- 延伸阅读：介绍一些延伸内容，比如Transformer变种、Transformer加速技巧等
    

---

## Transformer简介

Transformer最早由Ashish Vaswani等人在论文<<Attention is all you need>>[1]中提出，是一个**基于注意力机制的网络主干结构**。Transformer目前没有官方中文译名，暂时就叫Transformer吧。在该论文中，作者主要将Transformer用于机器翻译[2]任务，后来研究者们发现Transformer在自然语言处理的很多任务上都展现出了优越的性能。进一步地，研究者们尝试将Transformer应用于图像分类任务[3]，取得了初步成果，并后续在各项视觉任务中逐步取得了不俗的成绩[4]。

目前，Transformer已经成为了自然语言处理领域绝对主流的网络架构，当前大热的ChatGPT(https//chat.openai.com/)、GPT4(https//openai.com/gpt-4)、LLaMA(https//github.com/facebookresearch/llama)、Claude(https//claude.ai/)、文心一言(https//yiyan.baidu.com/)（百度打钱）等大语言模型 (Large Language Model,(https//en.wikipedia.org/wiki/Large_language_model) LLM）都以Transformer或者其变种作为主干网络，并且在计算机视觉领域也展现出了非常惊艳的效果。

> 题外话：这篇文章的起名方式掀起一股xxx is all you need的热潮，各种标题带有all you need的论文层出不穷，看得眼花缭乱-.-

虽然Transformer已经被应用于各个领域，但是其基本形式仍然主要适用于**序列建模**(Sequence Modeling[5])任务，比如唇语识别[6]、语音识别[7]、手语识别[8]、机器翻译等。以语音识别为例，Transformer接收一段说话内容对应的音频作为输入序列，然后输出该音频对应的文本内容。

介绍差不多就到这儿，让我们直接来看代码吧！

---

## 上代码，Let's Go!

### 总述

Transformer包括**编码器和解码器**两部分。

其中，编码器负责特征编码，即从原始的、比较底层的输入序列信号（以音频为例，比如原始音频中单个采样点数据构成的序列，或者音频的人工设计的MFCC、Fbank特征等）提取出抽象的、具有明显语义特征的特征信息；解码器负责从编码器得到的原始序列的特征信息中"破译"出目标序列的内容（比如从音频序列的特征中"破译"其对应的文本信息）。

编码器和解码器通常配套使用，但是编码器和解码器彼此之间在模型结构和用途上都相互独立，因此我们也可以只用编码器或者只用解码器。比如我们只能调用别人训练好的预训练编码器用于特征提取，那么此时我们只需要训练解码器。同时，编码器、解码器各自的用途并没有规定应当使用何种网络结构，比如我们可以用LSTM[9]或者时序卷积神经网络 (Temporal Convolutional Network, TCN[10]) 作为编码器的主干网络，然后使用Transformer作为解码器的主干网络等（不过实际使用过程中没人采用这么丑陋的backbone组合）。

> 上述内容只是为了让大家对于编码器解码器结构有一个更准确的认识，虽然Transformer本身采用编码器-解码器的结构，但是编码器-解码器的序列建模框架和Transformer本身并无联系。编码器-解码器相关论文参考Sequence to Sequence Learning with Neural Networks[11]。

本章内容安排如下：**（PC端点击左侧目录可直接跳转到对应章节）**

- 输入与输出：介绍输入和输出的基本格式；
    
- 注意力机制简介：介绍Transformer的核心思想和模块注意力机制；
    
- 注意力机制解释：进一步阐述Transformer中的注意力机制；
    
- 掩码机制：介绍Transformer中的掩码机制；
    
- 注意力机制完整代码：给出注意力机制的完整代码；
    
- 位置编码：介绍Transformer中的位置编码；
    
- Position-wise Feed-Forward Networks：介绍逐位置前向神经网络；
    
- 编码器：搭建编码器；
    
- 解码器：搭建解码器；
    
- 模型搭好啦！
    

从输入与输出到到Position-wise Feed-Forward Networks这几节主要逐个介绍Transformer中用到的各个模块，而编码器和解码器则介绍编解码器各自的总的模型结构。最后我们将encoder和decoder搭配使用，组成Transformer的完全体。

### 输入与输出

本文以语音识别为例，输入为  和  。 为输入特征，尺寸为 ， 为 batch-size (一个训练轮次中模型接收的样本数量)，  为当前batch中各音频序列的最大长度， 为特征维度。 为  个样本各自的序列长度，尺寸为 ,。

由于不同样本的序列长度不同，比如 “早上好! ” 的音频长度只有2秒，对应大概   ，而"你今天真好看"的音频有4秒，对应大概  。

为了让模型批量处理不同长度的音频，我们将同一个批次中的输入填充 (padding) 到同样长度。比如  ，那么  内就是 “早上好! ” 对应的样本信息，  就是填充的无用信息。这些填充的部分后续计算损失函数时会被丟掉，因此具体填充为何值对运算结果无影响，通常填零，不过也可以填其他值。

同理，输出文本也要做类似的处理，输出为  和  。 为输出的文本后验概率分布，尺寸为 ， 为batch-size，  为当前batch中的文本序列的最大长度，如"我可以咬一口吗? "对应于 ， 为可能的候选字个数，即词表大小； lens 为各个文本序列的长度，尺寸为 ,，如下图图所示的中，lens  。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

图1. 输入pading示例

### 注意力机制简介

简介

前文提到注意力机制是Transformer的主要组成部分，那么"注意力"到底是个什么东西呢？

其实注意力机制并不是一个新鲜概念，而且这一概念也不仅仅局限于深度学习领域。以我们人类为例，当我们在通过面相判断一个人的性别时，那么我们人眼的注意力可能就主要放在这个人的脸上，看鼻子、眼睛、耳朵等。当我们通过肢体动作判断运动员所从事的运动时，可能又会关注这个人手脚的肢体动作，而不关注这个人的长相。这一现象在神经网络中也存在（神经网络和人类的感知机理完全不同，仅作类比），如下图图2所示：

![图片](https://mmbiz.qpic.cn/sz_mmbiz_jpg/gYUsOT36vfrkYibbeKiaMTSSQyHwe1F7BR201ykEHiaiaKZAicISBE4DOuPdNKxO0HicnH3BzibvJSkIfxj2CLKZFsGFQ/640?wx_fmt=jpeg&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

图2. 注意力可视化示例

上图中，热力图的颜色越深表示网络对这一区域的关注程度越高。可以看到，边牧面部和前胸区域的颜色较深，但是地面、背景的树等的颜色较浅，这说明神经网络可以学到“不同区域的对于当前任务的重要性不同”。

> 注：热力图(saliency map)的画法多种多样，无固定范式，如有兴趣请自行Google

值得注意的是，上图所示的热力图是训练完成之后我们采取一些专门为了注意力可视化而设计的方法得到的，比如计算模型输出对于图像输入的每个原始像素点的梯度，然后认为梯度越大的点模型的关注度越高。但是，我们在模型训练的时候却并不一定要求模型直接学习到模型输出和输入之间的注意力，比如CNN中，我们仅仅只是通过反向传播的方式来更新卷积核的权重，但是并没有任何和注意力直接有关的约束。而Transformer，则是**直接以注意力机制为主要构成模块的主干网络**。

再举一个例子，假如我们想把“早上好！”这句中文翻译成对应的英文“Good Morning！”。我们现在把“早上好！”作为模型输入，“Good Morning！”作为模型输出，那么模型在尝试着拟合输入输出关系的时候，应当可以关注到对于输出的某一部分，输入的不同部分的重要性是不一样的。

具体来讲，“Good”和“好”的关联性最强，和“早上”以及“！”的关联性较弱；“Morning”和“早上”的关联性最强，和“好”以及“！”的关联性较弱；“！”和“！”的关联性最强，和“早上”以及“好”的关联性较弱。如下图所示，其中线条的颜色的颜色深浅表示相连的输入输出字符之间的关联性，颜色越深表示关联性越大。这一“关联性”，其实就是注意力的体现。

![图片](https://mmbiz.qpic.cn/sz_mmbiz_jpg/gYUsOT36vfrkYibbeKiaMTSSQyHwe1F7BRI9icHl2jNdAibz77hAdLINBLyqc6JI9xRpcF2VUVtEjfeOzVfhR2l0LQ/640?wx_fmt=jpeg&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

图3. 中英翻译注意力示例

好了，说了这么多。那么我们来看Transformer中的注意力机制的实现方式吧!

Transformer中用的注意力机制包括**Query (**  **)，Key (**  **)和Value** (  )三个组成部分。可以这样理解，  是我们手头已经有的所有资料，可以作为一个知识库； 是我们待查询的东西，我们希望把  中和  有关的信息都找出来; 而  是  这个知识库的钥匙，  中每个位置的信息对应于一个  。对于  中的每个位置的信息而言，如果  和对应钥匙 的匹配程度越高，那么就可以从该条信息中找到和  更多的内容。

举个例子，我们现在希望给四不像找妈妈。以四不像作为  ，以[鹿，牛，羊，驴，青蛙 ]同时作为  和  ，然后发现四不像和鹿的相似度为  、和牛的相似度为  ，和羊、驴的相似度均为  ，和青蛙的相似度为 0 ，那么最终的查询结果就是  鹿  牛  羊  驴 +0 青蛙。

从上面的描述可以看出，计算注意力的流程可以分解为以下两个步骤:

1. 计算  和  的相似度
    
2. 根据计算得到的相似度，取出  每条信息中和  有关的内容
    

Transformer中注意力的计算方法也可以大致分为上面两步。

我们以下面的例子为例。计算过程的简要图如下图所示 (美观起见，保留至两位小数)，详细分析附后。

![图片](https://mmbiz.qpic.cn/sz_mmbiz_jpg/gYUsOT36vfrkYibbeKiaMTSSQyHwe1F7BRChSL1lUhwbfJPYp5dMNDaWBnliamynKKuI3TFjthbAqM17W0ELOO3sA/640?wx_fmt=jpeg&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

图4. 注意力计算示例

 的序列长度为 3 ，即共有三个元素，  和  的序列长度为 4 ，即分别有 4 个元素，  的特征维度均为 2 ，我们现在希望计算  中和  有关的信息。 的每一个元素都是一个长度为 2 的行向量。

**计算Q和K的相似度**：既然要计算相似度，我们应当首先确定两个向量之间的相似度的度量标准。简单起见，我们直接以内积作为衡量两个向量之间相似度的方式，比如  (  表示  的第一个位置的元素，  同）

那么  和  之间的相似度为

> 向量之间的相似度衡量并没有特定标准，比如也可以用两个向量之间的余弦相似度，不过现在主流的还是直接两个向量做内积。

记  和  的相似度计算结果为 score，不难看出， score 应该是一个  的矩阵，因为 有三个元素，  有四个元素，其中第  行第  列的元素为  的第  个元素和  的第  个元素之间的相似度  。可以直接将 score 和  之间的关系写成矩阵运算的形式，结果如下:

以  为例，  和  的相似度分别为  ，以此类推。不知道大家注意到没有，这里  和  的相似度向量是不是可以看成一个概率分布? 方便起见，我们用softmax函数逐行对 score 进行归一化。归一化之后，  都是一个离散的概率分布。我们将 softmax 函数逐行归一化之后的结果记为 attention :

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/gYUsOT36vfrkYibbeKiaMTSSQyHwe1F7BR5tIIwWyLIgVQ2qWn4mibELvd4pyzBaLkNfCtRyeiaJNw5wlx6a8arQFg/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

> 上面的计算结果即为前文提到的attention，本质上就是一个概率分布，表示  和  之间的相似度

**取出V中每条信息中和**  **有关的内容**：得到  和  之间的相似度 attention 之后，我们就可以计算  中的每一个位置的元素和  中所有位置元素的注意力运算结果，记该运算结果为  ，其中  为一个  的行向量。以  为例，  和  的相似度的概率分布为  ，我们将该概率分布和  进行逐元素相乘，可以得到:

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/gYUsOT36vfrkYibbeKiaMTSSQyHwe1F7BRNkxvnaLzmhmfG9bF6P0Hl7R7kynSHGgMtSpRctkS9orH5FfBkW8libg/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

将上述整个运算过程写成矩阵相乘的形式:

  

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/gYUsOT36vfrkYibbeKiaMTSSQyHwe1F7BRyvNzjlI6IibhvbYfr9ASaxicQFqmI8JK8rJ3yLLKOZrYiboCa0pS1lHhg/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

ok，上面就是我们根据输入的  根据注意力机制得到的最终运算结果。

### 注意力机制解释

下面对注意力机制进行进一步解释。

1. 注意力机制本质上可以认为是**求一个离散概率分布的数学期望。**定义一个离散的随机变量  ，随机变量  的所有可能取值为 ， ，离散分布  ，于是注意力的计算结果就是随机变量  的数学希望  ，而离散分布  就是上文通过  和  计算得到的 softmax 归一化之后的attention。通过这里的解释，我们也可以更好地理解为什么计算 attention 的时候需要使用 softmax 函数进行归一化，归一化之后的每一个 attention 行向量都是一个离散的概率分布，具有更好的数学意义。
    
2. 注意力机制本身并没有对  和  的内容做出任何限制。比如，我们现在希望计算音频和文本之间的注意力，或者说希望从音频特征中提取和文本相关的信息，那么这个时候应该将文本特征作为  ，音频特征作为  和  (后文的交叉注意力机制)；又比如，我们希望计算文本和文本自身的注意力，那么就应该将文本特征同时作为  和  (后文的自注意力机制)。在实际应用过程中，常用的就是上文提到的交叉注意力机制和自注意力机制 (后文会进一步阐明具体用法），这两种注意力机制的计算上没有什么区别，只是  的选取稍有不同。
    
3. Transformer中还对上述注意力机制进行了改进，使用了 “多头注意力机制" (Multi-Head Attention)。多头注意力机制假设输入的特征空间可以分为互不相交的几个子空间，然后我们只需要在几个子空间内单独计算注意力，最后将多个子空间的计算结果拼接即可。举个例子，假设  的维度都是 512 ，长度都是  ，现将维度 512 的特征空间分成 8 个 64 维的子空间，每个子空间内单独计算注意力，于是每个子维度的计算结果的尺寸为  ，然后再将8个子维度的计算结果沿着特征维度拼起来，得到最终的计算结果，尺寸为  ，和输入的  的尺寸保持一致。
    

多头注意力机制的伪代码如下：

```
q_len, k_len, v_len = ...batch_size, hdim = ...head_dim = ...assert hdim % head_dim = 0assert k_len == v_lennum_head = hdim // head_dimq = tensor(batch_size, q_len, hdim)k = tensor(batch_size, k_len, hdim)v = tensor(batch_size, v_len, hdim)def multi_head_split(x):  # x: (batch_size, len, hdim)  b, l, hdim = x.size()  x = x.reshape(b, l, num_head, head_dim).transpose(1, 2)    # (b, num_head, l, dim)  return xdef multi_head_merge(x, b):  # x: (batch_ize, num_head, len, head_dim)  b, num_head, l, head_dim = x.size()  x = x.transpose(1, 2).reshape(b, l, num_head * head_dim)    #(batch_size, l, hdim)  return xq, k, v = map(multi_head_split, [q, k, v])output = MultiHeadAttention(q, k, v)      # 该函数的具体实现后文给出output = multi_head_merge(output, batch_size)
```

多头注意力机制目前已经是Transformer的标配，通常每个注意力头的维度为64，注意力头的个数为输入维度除以64。

> 在Transformer原文中，作者并没有对多头注意力机制的motivation做过多的阐述，后来也有研究发现多头并不一定比单头好，参考论文<<Are sixteen heads really better than one?>>[12]，不过目前基本都是默认用的多头注意力。

4. Transformer使用的注意力机制的完整名称叫"Scaled Dot-Product Attention"，这里的"Scaled"是指对输入进行了缩放，将输入特征维度记为  ，缩放之后的注意力计算公式 (简单起见，同时为和上文的注意力权重 attention相区分，整个公式记为  ) 为
    
    ![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)
    

相比于原始的attentioni算公式，上述公式多了一个系数  。这一做法有点类似一种正则化，避免  的数值计算结果过大，导致 softmax 向着梯度很小的区域偏移。这一点在 Transformer原文的第4页有详细阐述，本文不再鳌述，也欢迎读者在评论区补充。

### 掩码机制

本节介绍Transformer中的掩码机制，主要包括三部分：

- encoder的self attention的长度mask
    
- decoder的self attention的causal mask
    
- encoder和decoder的cross-attention的mask
    

**encoder的self attention的长度mask**

细心的读者可能会发现，在上文我们提到，序列输入的长度可能是不一样的，我们当时的处理是将同一个batch中的所有样本都padding到最长长度，这样可以解决输入变长的问题，但是这样做的话，attention的计算会不会出问题? 举个例子，当前batch的最大长度是 10 ，而当前样本的长度为4，也就是说序列最后6个位置的数据都是padding填充的，应当舍弃，然而在计算自注意力的过程中，由于  的长度都为 10 ，所有最终计算出的attention长度也是 10 ，其中包含了  和  中padding部分的6个位置的attention计算结果。关于这一点，Transformer中采取的方法为掩码机制 (masking)。

具体来说，既然我们的目的是为了让attention的计算结果不受padding的影响，那么一个比较简单的方法是，直接将padding对应位置的attention权重置为 0 即可。

我们仍然以"早上好! "为例，假设当前batch中最长的句子的长度为8，而 "早上好! " 的长度为 4，因此我们需要padding 4 个位置。那么在计算注意力时，记第  个字符和第  个字符之间的注意力权重为  ，那么  应当满足:

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

也就是说，除了当  和  都为前 4 个token以外，其余情形的attention权重均为 0 ，因为该情形下  和  总有一个为padding的部分。如下图所示，其中白色代表attention权重不为 0 ，灰色代表attention权重为 0 ，即被mask掉的部分。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

图5. 长度mask示例

上图中，子图 (a) 表示只要  和  其一为padding，那么我们就将其attention权重置为 0 ; 而子图 (b) 表示当  为padding时将对应attention权重为 0 。实际模型训练过程中使用(b)而不使用 (a)，使用 (b) 不会出错是因为  为padding的部分最终计算loss时会被过滤掉，所以  是否 mask无影响。而使用(a)时，由于有些行的所有位置都被mask掉了，这部分计算attention时容易出现  。举个例子，我们可以将"早上好! "后面的4个位置的文本字符都用一个特殊的token "" 来填充，然后再计算交叉摘损失的过程中利用 `torch.nn.functional.cross_entropy` 的 `ignore_idx` 参数设成 "" 将这部分的loss去掉，不纳入计算。

为了得到子图 (b) 所示的mask，我们只需要保留每个输入样本对应的样本长度即可，代码如下:

```
def get_len_mask(b: int, max_len: int, feat_lens: torch.Tensor, device: torch.device) -> torch.Tensor:    attn_mask = torch.ones((b, max_len, max_len), device=device)    for i in range(b):        attn_mask[i, :, :feat_lens[i]] = 0    return attn_mask.to(torch.bool)m = get_len_mask(2, 4, torch.tensor([2, 4]), "cpu")# 为了打印方便，转为intm = m.int()# 输出tensor([[[0, 0, 1, 1],         [0, 0, 1, 1],         [0, 0, 1, 1],         [0, 0, 1, 1]],        [[0, 0, 0, 0],         [0, 0, 0, 0],         [0, 0, 0, 0],         [0, 0, 0, 0]]], dtype=torch.int32)
```

在上面的例子中，当前batch中有两个样本，长度分别为2和4，可以看到，长度为2的样本的后两个位置都被mask掉了。

得到mask之后，我们应该怎么用才能将padding部分的attention权重置为 0 呢? 做法是直接将 和  的内积计算结果中被mask掉的部分的值置为-inf，这样的话，过一层softmax之后， padding部分的attention权重就是0了，这一点可以使用PyTorch的masked_fill函数实现。

```
scores = torch.matmul(Q, K.transpose(-1, -2)) / torch.sqrt(d_k)if attn_mask is not None:  scores.masked_fill_(attn_mask, -float("inf"))
```

> 在实际使用过程中，我们一般不直接置为-float("inf")，而是一个很小的负数，比如Bert[13]中用的-10000，具体原因本文不深入讨论。

**decoder的self attention的causal mask**

值得一提的是，除了上述的由于长度padding带来的mask以外，Transformer中还有一类常用的 mask，即causal mask，中文直译为 “因果掩码”。这一mask通常用在文本预测/生成相关的任务中，以语言模型建模为例，语言模型建模的基本形式为  ，其中  为待优化的模型参数，  表示一个长度为 的输入样本，  表示条件概率。直观上看，语言模型的目标是尽可能最大化输入样本  的对数似然，根据链式法则，  。在用Transformer对上述概率进行建模时，我们可以让第  个位置接受到第 1 至第  位置上的信息输入，然后输出  的预测值。

以 "早上好!" 为例子，  早上好!  早  上|ㅜㅜ  好  早上  早上好

> 在实际语言模型建模过程中，由于  不好处理，常常会直接建模  ，其中  为一个特殊的"SOS" (Start of Sentence) token，本文不深入讨论，有兴趣的读者可以参考链接。

在这一过程中，我们给模型的完整输入为  ，但是预测  时只用到了前  个输入的信息，也就是说后面的  个位置的信息我们应当mask 掉，只能根据历史信息来预测当前位置的输出。这很好理解，假如模型的输入为 “早上好”，那么应该通过 “早" 来预测“上”，通过 “早上" 来预测 "好”，而不是通过“早上好" 来预测“上"或者“好”，因为这相当于直接把ground truth告诉模型了。

和上文长度的mask类似，我们可以画出对应的causal mask的形式，如下图所示:

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

图6. causal mask示例

得到上述causal mask的代码非常简单，如下所示：

```
def get_subsequent_mask(b: int, max_len: int, device: torch.device) -> torch.Tensor:    """    Args:        b: batch-size.        max_len: the length of the whole seqeunce.        device: cuda or cpu.    """    return torch.triu(torch.ones((b, max_len, max_len), device=device), diagonal=1).to(torch.bool)     # or .to(torch.uint8)
```

> again，我们不用关心padding部分，因为padding部分最终计算loss时会被舍弃。

有读者看到这里可能会问，为什么我们刚刚在计算encoder的长度mask的时候没有考虑这种causal的关系呢？

这是因为encoder中只是起到提取特征的作用，不需要像decoder那样计算自回归的交叉熵损失函数，所以不需要额外的causal约束。理论上来说，给encoder加上causal mask也可以，但是其意义和非causal存在一定差异。对于encoder中的第 个位置而言，不加入causal mask时，该位置可看到整个序列的所有信息；加入causal mask时，该位置只能看到第个位置的信息，这一约束没有必要。

下面的encoder和decoder之间的mask也是类似，在计算encoder和decoder的cross attention的mask的时候，由于decoder可以获取到encoder的所有信息，因此我们不需要针对encoder做额外的causal mask。

**encoder和decoder的cross-attention的mask**

除了上述两类encoder中self-attention的长度mask、decoder中self-attention的causal mask以外，另一类是encoder和decoder的cross attention的mask。比如"早上好！"对应的音频特征长度为600，但是encoder当前batch中音频特征最长为800，因此在做cross-attention时，encoder特征后面的200个位置的特征应当被mask掉，如下图所示：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

图7. cross-attention mask示例

> again，不用管Q中后面4个padding的位置，计算loss时会筛掉。

得到上述mask的代码如下所示：

```
def get_enc_dec_mask(    b: int, max_feat_len: int, feat_lens: torch.Tensor, max_label_len: int, device: torch.device) -> torch.Tensor:    attn_mask = torch.zeros((b, max_label_len, max_feat_len), device=device)       # (b, seq_q, seq_k)    for i in range(b):        attn_mask[i, :, feat_lens[i]:] = 1    return attn_mask.to(torch.bool)
```

掩码机制补充：

1. 要尤其注意mask是True还是False，这一点非常容易错。通常的用法是将应该被mask掉的部分置为True，但是也有一些代码是将应该被保留的部分置为True，比如Pytorch 2.0中官方实现的scaled_dot_product_attention(https//pytorch.org/docs/stable/generated/torch.nn.functional.scaled_dot_product_attention.html)；
    
2. mask有很多等价写法，比如可以将被mask掉的部分的值设为-inf，被保留部分的值设为0，然后直接`attn+=mask`即可（参考LLaMA(https//github.com/facebookresearch/llama/blob/ef351e9cd9496c579bf9f2bb036ef11bdc5ca3d2/llama/model.py%23L299-L300)），这和masked_fill的写法等价，用一种即可；
    
3. 注意mask的数据类型，最好固定为bool；
    
4. 总体来说，实际使用过程中的mask就只有上面提到的三种，但是mask的设计本身非常灵活，可能会根据某些情形来设置特定的mask，比如预训练中常用的掩码预测任务 (参考Bert  )。又比如，我们限定第  个位置的  只能attend到  范围内的  ，那么此时mask需要做相应的更改，此处不再鳌述。
    

### 注意力机制完整代码

通过上文的推导以及解释，我们现在给出Transformer中计算注意力机制的完整代码：

```
class MultiHeadAttention(nn.Module):    def __init__(self, d_k, d_v, d_model, num_heads, p=0.):        super(MultiHeadAttention, self).__init__()        self.d_model = d_model        self.d_k = d_k        self.d_v = d_v        self.num_heads = num_heads        self.dropout = nn.Dropout(p)                # linear projections        self.W_Q = nn.Linear(d_model, d_k * num_heads)        self.W_K = nn.Linear(d_model, d_k * num_heads)        self.W_V = nn.Linear(d_model, d_v * num_heads)        self.W_out = nn.Linear(d_v * num_heads, d_model)        # Normalization        # References: <<Delving Deep into Rectifiers: Surpassing Human-Level Performance on ImageNet Classification>>        nn.init.normal_(self.W_Q.weight, mean=0, std=np.sqrt(2.0 / (d_model + d_k)))        nn.init.normal_(self.W_K.weight, mean=0, std=np.sqrt(2.0 / (d_model + d_k)))        nn.init.normal_(self.W_V.weight, mean=0, std=np.sqrt(2.0 / (d_model + d_v)))        nn.init.normal_(self.W_out.weight, mean=0, std=np.sqrt(2.0 / (d_model + d_v)))    def forward(self, Q, K, V, attn_mask, **kwargs):        N = Q.size(0)        q_len, k_len = Q.size(1), K.size(1)        d_k, d_v = self.d_k, self.d_v        num_heads = self.num_heads        # multi_head split        Q = self.W_Q(Q).view(N, -1, num_heads, d_k).transpose(1, 2)        K = self.W_K(K).view(N, -1, num_heads, d_k).transpose(1, 2)        V = self.W_V(V).view(N, -1, num_heads, d_v).transpose(1, 2)                # pre-process mask         if attn_mask is not None:            assert attn_mask.size() == (N, q_len, k_len)            attn_mask = attn_mask.unsqueeze(1).repeat(1, num_heads, 1, 1)    # broadcast            attn_mask = attn_mask.bool()        # calculate attention weight        scores = torch.matmul(Q, K.transpose(-1, -2)) / np.sqrt(d_k)        if attn_mask is not None:            scores.masked_fill_(attn_mask, -1e4)        attns = torch.softmax(scores, dim=-1)        # attention weights        attns = self.dropout(attns)        # calculate output        output = torch.matmul(attns, V)        # multi_head merge        output = output.transpose(1, 2).contiguous().reshape(N, -1, d_v * num_heads)        output = self.W_out(output)        return output
```

上述代码添加了一些细节，比如对输入的  做线性变换等等，但是核心代码段仍为 attention的计算部分。

Transformer原文中也给出了注意力机制的框架图，如下图所示：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

图8. 注意力机制框架图

### 位置编码

在Transformer中，模型输入其实隐含了前后顺序，比如 “早上好!"，我们将这句话输入模型时，潜在意思是 “早" 是第一个字， “上" 是第二个字，以此类推。同样，  (好早上) 的含义是作为条件的 "早上” 的" 早 "在前"上" 在 "后"。

然而，从attention的形式看，attention并不具备上述对位置进行编码的能力，这是因为attention 对所有输入都一视同仁，即  好  早上  好  上早  。

因此，为了保留原始输入的顺序关系，我们需要在Transformer的输入中加入表征顺序的特征， Transformer原文中位置编码的计算方式如下:

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

其中，pos为位置序号，  为特征的维度，  表示特征的第  维。

> 位置编码的细节见下文"延伸"一章的"位置编码的选择"，此处略过。

位置编码的代码如下：

```
def pos_sinusoid_embedding(seq_len, d_model):    embeddings = torch.zeros((seq_len, d_model))    for i in range(d_model):        f = torch.sin if i % 2 == 0 else torch.cos        embeddings[:, i] = f(torch.arange(0, seq_len) / np.power(1e4, 2 * (i // 2) / d_model))    return embeddings.float()
```

补充：同样作为建模时序信息的网络，为什么RNN中不需要位置编码？

在Transformer出现之前，序列任务的主干网络以RNN家族为主，比如GRU[15]、LSTM[16]等等，但是这些网络却不需要位置编码。我个人的理解是RNN的串行输入形式已经隐含了输入的位置关系，由于RNN的输入是串行的，位置靠前的信息较先输入给模型，而位置靠后的信息较后输入给模型，这种串行的输入方式可以提供关于输入的位置关系。然而，Transformer是并行训练的，直接一次性将所有输入都喂给模型，这个时候需要额外的操作来告诉模型输入的位置关系。

### Position-wise Feed-Forward Networks

前文提到，注意力机制为Transformer的主要构成模块。除了注意力机制以外，为了增强模型的表示能力，作者还提出逐位置的前向神经网络（Position-wise Feed-Forward Networks），其实说白了就是一个两层的MLP（<.<不知道为什么要起这么唬人的名字），代码如下：

```
class PoswiseFFN(nn.Module):    def __init__(self, d_model, d_ff, p=0.):        super(PoswiseFFN, self).__init__()        self.d_model = d_model        self.d_ff = d_ff        self.conv1 = nn.Conv1d(d_model, d_ff, 1, 1, 0)        self.conv2 = nn.Conv1d(d_ff, d_model, 1, 1, 0)        self.relu = nn.ReLU(inplace=True)        self.dropout = nn.Dropout(p=p)    def forward(self, X):        out = self.conv1(X.transpose(1, 2))     # (N, d_model, seq_len) -> (N, d_ff, seq_len)        out = self.relu(out)        out = self.conv2(out).transpose(1, 2)   # (N, d_ff, seq_len) -> (N, d_model, seq_len)        out = self.dropout(out)        return out
```

> 关于MLP的实现方式，上面是用的一维卷积，也可以直接通过nn.Linear实现。

### 编码器

铺垫了这么久，终于可以用上面提到的各种模块来自己搭一个Transformer了。

编码器和解码器的整体框架图如下图所示：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

图9. encoder-decoder框架图

其中，编码器的主要用途是进行信息抽取，比如语音识别中，"Inputs"为原始音频提取的Fbank或者MFCC特征，尺寸为(batch_size, seq_len, hdim)，我们现在希望从这些人工特征中得到具有高层语义的特征，于是就将其输入Transformer的encoder，进行特征编码。

编码器的主要结构包括三部分：特征编码、位置编码、N个encoder layer。其中，特征编码的用途是对原始的输入信息进行特征提取，得到**连续**向量，以作为后续输入；位置编码的用途是给特征编码加上位置信息；encoder layer的作用是实现高层语义特征的提取。每个encoder layer的模型结构完全相同，为一个多头注意力和一个MLP，再加上一些归一化和残差连接。

解码器的主要用途是根据编码器的信息推断出对应的文本是什么，比如我们现在通过encoder拿到了”早上好！“这句话对应的音频特征，我们希望让decoder具备推理能力，可以在接受这些音频特征作为输入的前提下去判断对应的文本内容是什么。

解码器的主要结构和编码器十分类似，也包括特征编码、位置编码和decoder layer三部分，只是decoder layer相比encoder layer而言多了一个和encoder输出之间的交叉注意力。

> 关于上图中的input embeddings和output embeddings，由于Transformer最早是应用于NLP中的机器翻译任务，机器翻译任务中，encoder和decoder的原始输入都是一个个离散的字符，显然无法直接作为模型输入，因此通用的做法是通过一层word embedding，将每个字符映射到一个独一无二的向量，然后再作为后续encoder/decoder的输入。
> 
>   
> 而在其他任务中，比如语音识别、唇语识别等等，由于decoder的输入仍然为离散的文本字符，因此output embeddings仍然为word embedding，但是encoder的输入是音频或者图像序列的特征，此时input embedding不再是word embedding，而是这个模态自己的特征提取器，比如通过一个 1 维的ResNet将 (batch_size,  ) 的 80 维Fbank音频特征映射到 (batch_size,  的特征，然后作为encoder的输入。

接下来我们实现一个encoder，首先从encoder layer开始。

```
class EncoderLayer(nn.Module):    def __init__(self, dim, n, dff, dropout_posffn, dropout_attn):        """ Args: dim: input dimension n: number of attention heads dff: dimention of PosFFN (Positional FeedForward) dropout_posffn: dropout ratio of PosFFN dropout_attn: dropout ratio of attention module """        assert dim % n == 0        hdim = dim // n     # dimension of each attention head        super(EncoderLayer, self).__init__()        # LayerNorm        self.norm1 = nn.LayerNorm(dim)        self.norm2 = nn.LayerNorm(dim)        # MultiHeadAttention        self.multi_head_attn = MultiHeadAttention(hdim, hdim, dim, n, dropout_attn)        # Position-wise Feedforward Neural Network        self.poswise_ffn = PoswiseFFN(dim, dff, p=dropout_posffn)    def forward(self, enc_in, attn_mask):        # reserve original input for later residual connections        residual = enc_in        # MultiHeadAttention forward        context = self.multi_head_attn(enc_in, enc_in, enc_in, attn_mask)        # residual connection and norm        out = self.norm1(residual + context)        residual = out        # position-wise feedforward        out = self.poswise_ffn(out)        # residual connection and norm        out = self.norm2(residual + out)        return out
```

完整的encoder代码如下所示：

```
class Encoder(nn.Module):    def __init__(            self, dropout_emb, dropout_posffn, dropout_attn,            num_layers, enc_dim, num_heads, dff, tgt_len,    ):        """ Args: dropout_emb: dropout ratio of Position Embeddings. dropout_posffn: dropout ratio of PosFFN. dropout_attn: dropout ratio of attention module. num_layers: number of encoder layers enc_dim: input dimension of encoder num_heads: number of attention heads dff: dimensionf of PosFFN tgt_len: the maximum length of sequences """        super(Encoder, self).__init__()        # The maximum length of input sequence        self.tgt_len = tgt_len        self.pos_emb = nn.Embedding.from_pretrained(pos_sinusoid_embedding(tgt_len, enc_dim), freeze=True)        self.emb_dropout = nn.Dropout(dropout_emb)        self.layers = nn.ModuleList(            [EncoderLayer(enc_dim, num_heads, dff, dropout_posffn, dropout_attn) for _ in range(num_layers)]        )        def forward(self, X, X_lens, mask=None):        # add position embedding        batch_size, seq_len, d_model = X.shape        out = X + self.pos_emb(torch.arange(seq_len, device=X.device))  # (batch_size, seq_len, d_model)        out = self.emb_dropout(out)        # encoder layers        for layer in self.layers:            out = layer(out, mask)        return out
```

### 解码器

首先从decoder layer开始

```
class DecoderLayer(nn.Module):    def __init__(self, dim, n, dff, dropout_posffn, dropout_attn):        """ Args: dim: input dimension n: number of attention heads dff: dimention of PosFFN (Positional FeedForward) dropout_posffn: dropout ratio of PosFFN dropout_attn: dropout ratio of attention module """        super(DecoderLayer, self).__init__()        assert dim % n == 0        hdim = dim // n        # LayerNorms        self.norm1 = nn.LayerNorm(dim)        self.norm2 = nn.LayerNorm(dim)        self.norm3 = nn.LayerNorm(dim)        # Position-wise Feed-Forward Networks        self.poswise_ffn = PoswiseFFN(dim, dff, p=dropout_posffn)        # MultiHeadAttention, both self-attention and encoder-decoder cross attention)        self.dec_attn = MultiHeadAttention(hdim, hdim, dim, n, dropout_attn)        self.enc_dec_attn = MultiHeadAttention(hdim, hdim, dim, n, dropout_attn)    def forward(self, dec_in, enc_out, dec_mask, dec_enc_mask, cache=None, freqs_cis=None):        # decoder's self-attention        residual = dec_in        context = self.dec_attn(dec_in, dec_in, dec_in, dec_mask)        dec_out = self.norm1(residual + context)        # encoder-decoder cross attention        residual = dec_out        context = self.enc_dec_attn(dec_out, enc_out, enc_out, dec_enc_mask)        dec_out = self.norm2(residual + context)        # position-wise feed-forward networks        residual = dec_out        out = self.poswise_ffn(dec_out)        dec_out = self.norm3(residual + out)        return dec_out
```

然后是decoder的完整代码

```
class Decoder(nn.Module):    def __init__(            self, dropout_emb, dropout_posffn, dropout_attn,            num_layers, dec_dim, num_heads, dff, tgt_len, tgt_vocab_size,    ):        """ Args: dropout_emb: dropout ratio of Position Embeddings. dropout_posffn: dropout ratio of PosFFN. dropout_attn: dropout ratio of attention module. num_layers: number of encoder layers dec_dim: input dimension of decoder num_heads: number of attention heads dff: dimensionf of PosFFN tgt_len: the target length to be embedded. tgt_vocab_size: the target vocabulary size. """        super(Decoder, self).__init__()        # output embedding        self.tgt_emb = nn.Embedding(tgt_vocab_size, dec_dim)        self.dropout_emb = nn.Dropout(p=dropout_emb)                            # embedding dropout        # position embedding        self.pos_emb = nn.Embedding.from_pretrained(pos_sinusoid_embedding(tgt_len, dec_dim), freeze=True)        # decoder layers        self.layers = nn.ModuleList(            [                DecoderLayer(dec_dim, num_heads, dff, dropout_posffn, dropout_attn) for _ in                range(num_layers)            ]        )    def forward(self, labels, enc_out, dec_mask, dec_enc_mask, cache=None):        # output embedding and position embedding        tgt_emb = self.tgt_emb(labels)        pos_emb = self.pos_emb(torch.arange(labels.size(1), device=labels.device))        dec_out = self.dropout_emb(tgt_emb + pos_emb)        # decoder layers        for layer in self.layers:                dec_out = layer(dec_out, enc_out, dec_mask, dec_enc_mask)        return dec_out
```

### 模型搭好啦！

到目前位置，我们已经成功搭建起了encoder和decoder模型，整个模型结构如下：

```
class Transformer(nn.Module):    def __init__(            self, frontend: nn.Module, encoder: nn.Module, decoder: nn.Module,            dec_out_dim: int, vocab: int,    ) -> None:        super().__init__()        self.frontend = frontend     # feature extractor        self.encoder = encoder        self.decoder = decoder        self.linear = nn.Linear(dec_out_dim, vocab)    def forward(self, X: torch.Tensor, X_lens: torch.Tensor, labels: torch.Tensor):        X_lens, labels = X_lens.long(), labels.long()        b = X.size(0)        device = X.device        # frontend        out = self.frontend(X)        max_feat_len = out.size(1)                            # compute after frontend because of optional subsampling        max_label_len = labels.size(1)        # encoder        enc_mask = get_len_mask(b, max_feat_len, X_lens, device)        enc_out = self.encoder(out, X_lens, enc_mask)        # decoder        dec_mask = get_subsequent_mask(b, max_label_len, device)        dec_enc_mask = get_enc_dec_mask(b, max_feat_len, X_lens, max_label_len, device)        dec_out = self.decoder(labels, enc_out, dec_mask, dec_enc_mask)        logits = self.linear(dec_out)        return logits
```

使用下面的代码进行验证

```
if __name__ == "__main__":    # constants    batch_size = 16                 # batch size    max_feat_len = 100              # the maximum length of input sequence    max_lable_len = 50              # the maximum length of output sequence    fbank_dim = 80                  # the dimension of input feature    hidden_dim = 512                # the dimension of hidden layer    vocab_size = 26                 # the size of vocabulary    # dummy data    fbank_feature = torch.randn(batch_size, max_feat_len, fbank_dim)        # input sequence    feat_lens = torch.randint(1, max_feat_len, (batch_size,))               # the length of each input sequence in the batch    labels = torch.randint(0, vocab_size, (batch_size, max_lable_len))      # output sequence    label_lens = torch.randint(1, max_label_len, (batch_size,))             # the length of each output sequence in the batch    # model    feature_extractor = nn.Linear(fbank_dim, hidden_dim)                    # alinear layer to simulate the audio feature extractor    encoder = Encoder(        dropout_emb=0.1, dropout_posffn=0.1, dropout_attn=0.,        num_layers=6, enc_dim=hidden_dim, num_heads=8, dff=2048, tgt_len=2048    )    decoder = Decoder(        dropout_emb=0.1, dropout_posffn=0.1, dropout_attn=0.,        num_layers=6, dec_dim=hidden_dim, num_heads=8, dff=2048, tgt_len=2048, tgt_vocab_size=vocab_size    )    transformer = Transformer(feature_extractor, encoder, decoder, hidden_dim, vocab_size)    # forward check    logits = transformer(fbank_feature, feat_lens, labels)    print(f"logits: {logits.shape}")     # (batch_size, max_label_len, vocab_size)    # output msg    # logits: torch.Size([16, 100, 26])
```

ok，前向过程能跑通，能跑通应该就没啥问题（...吧）

---

## 总结

本文中，我们从零开始实现了一个Transformer，对网络的细节进行了非常细致的分析，包括注意力机制、掩码机制、位置编码等等，然后基于各个子模块分别实现了Transformer的encoder和decoder，可以作为各种序列任务的主干网络。

为了方便读者阅读代码，本文的所有代码已经打包放到Github Gist，欢迎star~。

https//gist.github.com/xiabingquan/eb2ceb583a1e8858c23da23ac7a4a340

---

## 延伸阅读

### Transformer和RNN各自的优劣势

相比于Transformer，RNN不具有全局建模的能力，由于RNN的基本形式为 ，其中  表示当前位置，  表示当前位置的输入，  包含了第 1 至第  个位置的所有历史信息。可以看出，在这种形式中，第 1 至第  个位置的所有历史信息和当前位置的信息所占的地位是等同的，这种建模方式显然更加偏向于和当前位置较近的临近位置的信息。直观来看，不管输入序列多长，RNN都用一个维度不变的向量 (RNN中称为hidden state）来表示所有历史信息，这当然很可能会存在遗忘问题。举个例子，以一维的情况为例，此时  为一个  的矩阵，假设  ，于是  可以展开为:

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

> 上述例子并不严谨，比如没有考虑RNN各种激活函数，以及没有考虑  的其他情形等等，只是为了说明 “在RNN中不同位置的信息所占的地位并不对等” 这一现象。

从上面的讨论中也能看出， RNN的前向过程是串行的，即通过  和  来计算  ，然后再通过  和  来计算  ，直到序列末尾。这样的串行形式显然是不利于并行训练的。然而， RNN在推理方面相比于具有一定优势。不考虑模型性能、训练成本，单讨论推理成本的话，RNN 推理的理论时间复杂度为  ，而Transformer推理的理论时间复杂度为  ，其中 表示输出序列的长度。

以当前大火的大语言模型为例，假设我们以 “请续写诗句：白日依山尽，" 作为prompt， Transformer在推理时，首先将prompt作为输入，得到第一个输出token "黄"，然后再将 “请续写诗句：白日依山尽，黄 "作为输入，得到第二个输出token "河"，以此进行下去，直至输出完整的" 黄河入海流。欲穷干里目，更上一层楼 “。由于Transformeri算attention的时间复杂度是  ，即推理第  个输出token时需要对前  个所有tokeni算attention，因此推理成本也随着序列的长度平方级上涨。然而在RNN中，由于RNN使用一个隐向量  来表示从 1 到  所有输入的历史信息，推理过程中只需要根据隐向量的更新公式 更新隐变量  即可（然后再根据  得到第  个输出token )，每一步的计算成本都是恒定的 (都是矩阵相乘)，和输出序列的长度无关，因此RNN的推理时间复杂度为  。感兴趣的小伙伴可以了解一下RWKV(https://github.com/BlinkDL/ChatRWKV)，一个基于RNN的大语言模型，开源LLM界的一股清流...

### Transformer网络结构变种

自2017年Transformer提出以后，关于Transformer模型结构的改进层出不穷，比如语音识别的Conformer[17]、Branchformer[18]、E-Branchformer[19]等等，有兴趣的读者可以自行查阅相关文献。

### 位置编码的选择

Transformer实现位置编码的具体方式非常多，一直有新的位置编码形式被提出，包括可学习的位置编码、相对位置编码[20]、RoPE[21]、AliBi[22]等等，也有许多关于Transformer位置编码的特性的讨论，包括长程衰减等等，有兴趣的读者可以参考苏神的系列博客[23]。

**归一化方式的选择**

在CNN中，我们通常使用BatchNorm[24]作为网络的归一化模块。然而，在Transformer中，我们却使用LayerNorm[25]作为网络的归一化模块，这一差异主要是由于序列问题中输入长度的不确定性导致的。

在传统的图像分类任务中，每张图像的尺寸都一样，模型的输入尺寸为[batch_size, num_channels, height, width]，BatchNorm可以在num_channels维度对所有特征做归一化。在序列任务中，模型的输入尺寸为[batch_size, max_sequence_len, hidden_dim]，理论上讲，我们也可以在max_sequence_len这一所在维度上使用BatchNorm，即对所有位置的所有特征向量做归一化，但是这一做法存在问题，因为不同输入的序列长度不一样，有些位置的特征是padding，并不是合法的特征。

举个例子，假设batch_size为2，两个输入的序列长度分别为1和10，模型的输入尺寸为[2, 10, hidden_dim]，理论上讲，BatchNorm计算均值和方差应该针对这11个向量进行，但是由于第一个输入有9个位置都是padding，所以直接对[2, 10, hidden_dim]的模型输入做BatchNorm实际将这9个padding也纳入到了均值和方差的计算过程中，显然不对。

> "padding导致BatchNorm的均值和方差更新错误"这一问题可以通过mask解决，参考wav2vec-U[26]，但是并不常用，也不太优雅。

一个比较简单的方式是，我们在做归一化时直接不考虑序列这一维度，只在每一个位置的特征内部自己做归一化，即**针对每一位置的维度为hidden_dim的特征单独计算均值和方差**，这样的话就可以避免序列长度不同对于归一化的影响，这就是LayerNorm的做法。深度学习中的正则化方式非常多，其他的比如GroupNorm、InstanceNorm等，关于不同normalization方式的比较，可参考博客。

除了LayerNorm以外，近期大语言模型用的比较多的是RMSNorm  (Root mean square layer normalization)，计算方式如下。其中  是一个维度为  的特征向量，  为RMSNorm的输出向量,  为一个  维的可学习参数。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

简单来说，RMSNorm和LayerNorm类似，也是针对每个特征向量单独进行归一化，只是 LayerNorm的做法是zscore(https//en.wikipedia.org/wiki/Standard_score)，即减去均值除以方差，然后做一个带偏置项的线性变换  ，而RMSNorm的做法是直接对模长进行归一化，然后再做一个不带偏置项的线性变换  。

RMSNorm的主要优势在于计算量小 (只用计算向量的L2范数，不用计算均值和方差)，同时性能和LayerNorm也可比。

### 归一化位置的选择：post-norm和pre-norm

在Transformer的原始论文中，LayerNorm在attention、逐位置FFN之后，简称post-norm。比如，encoder的计算公式如下：

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

其中 MHSA 为多头自注意力机制 (Multi-Head Self Attention) 的缩写，PosFFN 为逐位置前馈神经网络 (Position-wise Feedforward Neural Networks) 的缩写。

然后，在实际训练的过程中，上述正则化方式会带来一些问题，比如Transformer稍微大一点就不 ，计算公式如下:

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

pre-norm细节请参考论文  。pre-norm和post-norm的对比图如下:

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

图10. pre-norm和post-norm对比图

通常来说，在post-norm和pre-norm都可以收敛且保证其他变量相同的前提下，post-norm的模型性能会比pre-norm稍微好一点点，但是pre-norm可以保证更好的收敛性。目前，pre-norm已经是许多序列建模框架的默认选项，比如ESPnet(https//github.com/espnet/espnet/blob/8ad0d3e564c2c483fb7cc09bc3997cc5af471580/espnet/nets/pytorch_backend/transformer/decoder_layer.py%23L46)、transformers(https//github.com/huggingface/transformers/blob/0ea42ef0f9f71deba7775ead33afa0e493823d60/src/transformers/models/gpt2/modeling_gpt2.py%23L410-L429)等。

### 加速Transformer的有关尝试

虽然Transformer可以很好地进行并行训练，但是由于attention的时间复杂度为  ，因此当序列长度很长时训练成本和推理成本都会很高。为此，研究者们进行了一系列的努力。

主要可以分为两个方向，第一是降低attention运算的理论上的时间复杂度，想办法从  降到  ，比如Linear Transformer  和Linformer  ；第二是在不改变attention理论时间复杂度的前提下，尽可能加速attention的运算，比如FlashAttention(https://github.com/Dao-AILab/flash-attention)。

关于第一点，为了降低时间复杂度，目前的相关工作几乎都或多或少用了一些近似技巧，虽然时间复杂度降下去了，但是性能有所损耗，目前业界使用不多。关于第二点，FlashAttention是一个很有代表性的做法，虽然它没有降低attention运算的时间复杂度，但是减少了GPU的SRAM (Static Random Access Memory） 和HBM (High Bandwidth Memory之间的通信开销（可以类比为CPU中，待运算的数据在磁盘中，原本需要一直在内存和硬盘之间不停倒腾数据，很费时间，现在倒腾的次数少了，时间就省下来了）。另外，最近也有一些优化大模型推理成本的一些很有意思的做法，比如PagedAttention(https//github.com/vllm-project/vllm)[32]，模拟了操作系统的页表机制，来避免GPU显存的碎片化和不合理的显存预分配，进而达到GPU显存更高效的利用，开源半年以来已经在Github斩获1w多star。

这一块之后会单独开一篇文章细讲，尤其是Flash Attention的原理和代码实现，很有意思。

参考

1. https://proceedings.neurips.cc/paper/2017/file/3f5ee243547dee91fbd053c1c4a845aa-Paper.pdf
    
2. https://arxiv.org/pdf/1409.0473.pdf
    
3. https://arxiv.org/pdf/2010.11929.pdf
    
4. http://openaccess.thecvf.com/content/CVPR2022/papers/He_Masked_Autoencoders_Are_Scalable_Vision_Le arners_CVPR_2022_paper.pdf
    
5. https://proceedings.neurips.cc/paper/2014/file/a14ac55a4f27472c5d894ec1c3c743d2-Paper.pdf
    
6. https://openaccess.thecvf.com/content_cvpr_2017/papers/Chung_Lip_Reading_Sentences_CVPR_2017_paper .pdf
    
7. https://www.nature.com/articles/s41598-022-12260-y
    
8. https://scholar.google.com/scholar_url? url=https://openaccess.thecvf.com/content_CVPR_2020/papers/Camgoz_Sign_Language_Transformers_Joint_E nd-to-End_Sign_Language_Recognition_and_Translation_CVPR_2020_paper.pdf
    
9. https://www.researchgate.net/publication/13853244_Long_Short-term_Memory
    
10. http://openaccess.thecvf.com/content_cvpr_2017/papers/Lea_Temporal_Convolutional_Networks_CVPR_20 17 paper.pdf
    
11. https://proceedings.neurips.cc/paper/2014/file/a14ac55a4f27472c5d894ec1c3c743d2-Paper.pdf
    
12. https://proceedings.neurips.cc/paper_files/paper/2019/file/2c601ad9d2ff9bc8b282670cdd54f69f-Paper.pdf
    
13. https://arxiv.org/pdf/1810.04805.pdf
    
14. https://arxiv.org/pdf/1810.04805.pdf
    
15. https://arxiv.org/pdf/1406.1078.pdf
    
16. https://www.researchgate.net/publication/13853244_Long_Short-term_Memory
    
17. https://arxiv.org/pdf/2005.08100.pdf
    
18. https://proceedings.mlr.press/v162/peng22a/peng22a.pdf
    
19. https://arxiv.org/pdf/2210.00077.pdf
    
20. https://arxiv.org/pdf/1901.02860.pdf
    
21. https://arxiv.org/pdf/2104.09864&hl=ja&sa=X&ei=5B0dZcHLGJ2h6rQPweSLOA0&scisig=AFWwaebUGjvb4 JBysy2Z117aHWfJ&oi=scholarr
    
22. https://arxiv.org/pdf/2108.12409.pdf
    
23. https://spaces.ac.cn/archives/8130
    
24. https://arxiv.org/pdf/1502.03167.pdf
    
25. https://arxiv.org/pdf/1607.06450.pdf
    
26. https://github.com/facebookresearch/fairseq/blob/da8fb630880d529ab47e53381c30ddc8ad235216/exam ples/wav2vec/unsupervised/models/wav2vec_u.py#L362-L367
    
27. https://proceedings.neurips.cc/paper/2019/file/1e8a19426224ca89e83cef47f1e7f53b-Paper.pdf
    
28. https://aclanthology.org/P19-1176.pdf
    
29. https://aclanthology.org/P19-1176.pdf
    
30. http://proceedings.mlr.press/v119/katharopoulos20a/katharopoulos20a.pdf
    
31. https://arxiv.org/pdf/2006.04768.pdf
    
32. https://arxiv.org/pdf/2309.06180.pdf
    

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

阅读 4413

​

写留言

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/gYUsOT36vfodx8Y8MkYkPq45UCaZVlRdibEYl3ibSGKC8xaOszOvswbqFLApfHSNzuMfG5sibBdbYcTDct0SFPUjw/300?wx_fmt=png&wxfrom=18)

极市平台

2772122

写留言

写留言

**留言**

暂无留言