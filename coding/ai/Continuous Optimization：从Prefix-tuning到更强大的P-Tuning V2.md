Original 李国趸 PaperWeekly

_2021年10月18日 18:56_

![Image](https://mmbiz.qpic.cn/mmbiz_gif/Psho9dm7oDHKVtfYDubjKdZRUjAfBQQicXjoZWJ3qnK42ooD4eeJUfJBM4SSZVa2RE5lO0j6rWwzliby0j9u4bDg/640?wx_fmt=gif&tp=wxpic&wxfrom=5&wx_lazy=1)

**©作者 **|**** 李国趸

**单位 **|**** 浙江大学硕士生

**研究方向 **|**** 少样本学习

![Image](https://mmbiz.qpic.cn/mmbiz_png/Psho9dm7oDGhKg9nnSz5qQrwKvXibt3wulOVRfC18yCkd6xXqGq22h6QUk8chptF0fnQ4uXeZtAktYMrWwG2SyQ/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

**Prefix-Tuning**

![Image](https://mmbiz.qpic.cn/mmbiz_png/VBcD02jFhgl2bQjUAJOrCRHCtvxHFyXmXRg4m3NtuSNbr0qykQ0ibtRPELicxRJqkbgyRHiawibrzwfjByCEH8Lic9Q/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

**论文标题：**

Prefix-Tuning: Optimizing Continuous Prompts for Generation

**论文链接：**

https://arxiv.org/abs/2101.00190

**代码链接：**

https://github.com/XiangLi1999/PrefixTuning

**1.1 动机**

在 prefix-tuning 之前的工作主要是人工设计模版或者自动化搜索模版，问题在于最终的性能对人工设计的模版的变化特别敏感，加一个词或者少一个词，或者变动位置啥的都会造成比较大的变化。自动化搜索模版成本也比较高，同时以前这种离散化的 token 的搜索出来的结果可能并不是最优的。

传统的微调范式利用预训练模型去对不同的下游任务进行微调，对每个任务都要保存一份微调后的模型断点，一方面微调整个模型耗时长，另一方面也会占很多存储空间。

基于上述两点，prefix-tuning 提出固定预训练 LM 不动，为 LM 添加可训练，任务特定的前缀，这样子就可以为不同任务保存不同的前缀，微调成本也小，同时这种 prefix 实际就是连续的可微的 virtual token，相比离散的 token，更好优化，效果更好。

### **1.2 方法**

这篇主要研究对象是 GPT-2 和 BART 做生成任务：table-to-text 和 summarization。

![Image](https://mmbiz.qpic.cn/mmbiz_png/VBcD02jFhgl2bQjUAJOrCRHCtvxHFyXmcqkHH8oNPRw67wPLShb9J3ymxgzsmMTGWaYiaTZzsGHgEVAIKM9ANHg/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

针对自回归 LM 来说，添加前缀的直觉是：合适的上下文能够在 fixed LM 的情况下去引导生成（比如 GPT-3 的 in context learning）；针对编码器-解码器模型来说，编码器和解码器的输入都增加了前缀，其直觉是：编码器端增加前缀是为了引导输入部分的编码（指导输入端要提取何种信息），解码端增加前缀是为了引导后续 token 的生成。

Deep Continuous Prompt：一般直觉的做法是在词嵌入层，把离散的 token 变成可训练的 virtual token，这样可训练的参数虽然最少，但是效果肯定没有在每一层都加入可微调的参数更好。所以这篇是在每层都加入了可训练的参数，优化的时候就是优化所有层的前缀。

套 MLP 来优化：作者发现直接优化可训练的参数会导致不稳定和性能下降，套了一个 MLP 映射矩阵，训练完以后，最后推理使用的还是映射以后的矩阵。

![Image](https://mmbiz.qpic.cn/mmbiz_png/VBcD02jFhgl2bQjUAJOrCRHCtvxHFyXmCNENKowCwiabku9Pb0eW8Xk3BoDujoNxDhGzetvyCRD5FZ6XaHOejFw/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

![Image](https://mmbiz.qpic.cn/mmbiz_jpg/VBcD02jFhgl2bQjUAJOrCRHCtvxHFyXmHm3pJyc574KLATs0nNT8MMUtMY39IJZgiaDsLlpHfLtMU3HB1O9xsxQ/640?wx_fmt=jpeg&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

### **1.3 实验**

设置了几个基线：微调，微调上面的两层，adapter-tuning：加 adapter layer。效果优于轻量微调的基线，和微调的基线差不多，在低资源情况下，效果优于微调。

在低资源的摘要生成上，prefix-tuning 相比微调生成的结果更加忠实于输入。这几个轻量微调的基线都能不错地泛化到 unseen 的 domain 上，这说明保留预训练 LM 的参数确实有用。

更长的前缀意味着更多的可微调参数，效果也变好，不过长度还是有阈值限制的（table-to-text 是 10，summarization 是 200）。

discrete prompting \< embedding-only ablation（只有词嵌入层的 prefix 是可训练的参数）\< prefix-tuning。这里也提到离散 token 优化的上限其实就是 embedding-only ablation。

连续可微的参数作为前缀比作为中缀更好一点。前缀的初始化方式在低资源情况有很大影响。随机初始化导致低性能和高方差，使用任务相关的词来初始化更好点：例如“摘要”和“表格到文本”。

相比 adapter tuning，prefix tuning 参数更少，但是效果更好，原因可能是尽量保证预训练 LM 的完整性。

### **1.4 总结**

- template design：continuous

- verbalizer design：none tuning

- strategy：Fixed-LM Prompt Tuning

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

P-tuning V1

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**论文标题：**

GPT Understands, Too

**论文链接：**

https://arxiv.org/abs/2103.10385

**代码链接：**

https://github.com/THUDM/P-tuning

**2.1 动机**

一个刻板印象是 GPT 不适合理解类任务，这篇就是去思考这种刻板印象是否正确。GPT-3 采用人工构造的模版来做 in context learning，人工设计的模版的变化特别敏感，加一个词或者少一个词，或者变动位置啥的都会造成比较大的变化（这里作者做了一个简单的验证实验，具体看论文）。近来的自动化搜索模版工作成本也比较高，同时以前这种离散化的 token 的搜索出来的结果可能并不是最优的。

和 prefix-tuning 差不多，反正是基于这两点去设计了一种连续可微的模版。

**2.2 方法**

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

相比 prefix-tuning，这里加了可微的 virtual token，但是仅限于输入，没有在每层加；另外 virtual token 的位置也不一定是前缀，插入的位置是可选的。这里的出发点实际是把传统人工设计模版中的真实 token 替换成可微的 virtual token。

优化这种 virtual token 的挑战：经过预训练的 LM 的词嵌入已经变得高度离散，如果随机初始化 virtual token，容易优化到局部最优值；这些 virtual token 理论是应该有相关关联的，如何建模这种关联也是问题。当然实际在实验中，作者发现的是用一个 prompt encoder 来编码收敛更快，效果更好。也就是说，用一个 LSTM+MLP 去编码这些 virtual token 以后，再输入到模型。

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

作者还发现其实可以加一些 anchor token，比如文本蕴含任务：输入是前提和假设，判断是否蕴含。一个连续化的模版是：【前提的输入】【continuous tokens】【假设的输入】【continuous tokens】【MASK】，作者提出加一个 anchor token：【?】效果会更好，也就是【前提的输入】【continuous tokens】?【假设的输入】【continuous tokens】【MASK】。

至于 continuous tokens 插入位置，token 的数量是多少，应该也有影响，作者这里采用的是用 3 个 continuous token 去分割开多个输入，比如上面这里文本蕴含的例子。

**2.3 实验**

这里测试了两个经典的 benchmark：一个是用来 knowledge probe 的 LAMA，实际就是做完形填空的预测，比如：Obama was born in【MASK】，让模型去预测；一个是 NLU 的 benchmark：superGLUE。

knowledge probe：为了评测预训练模型的效果，所以评测的时候是 fixed LM 的，训练连续的 prompt 时也是 fixed LM 的。注：LAMA-29k 是覆盖了 GPT2 和 BERT 词表的测试集。

- 在 LAMA 上，P-tuning + 各种 PLM 相比传统的手工模版最少都提升了 20 个百分点；

- p-tuning 效果优于利用手工设计的模版来预测的模型，也优于基于梯度来搜索的离散模版；

- p-tuning 效果优于或相当于微调的模型，可能原因是微调容易造成灾难性遗忘的问题，P-tuning 不会改变预训练模型的参数，而是通过寻找更好的连续提示来唤起存储的知识；

- p-tuning 对 GPT 这种单向模型相比传统微调的增益是很大的，也能作用于超大规模的，无法微调的模型。

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**2.3.1 superGLUE**

\*\*supervised setting：\*\*整个训练集训练，整个验证集进行模型选择和网格的超参数搜索，注意这里实际是微调整个模型+prompt；对于 bert 模型，p-tuning 在 5 个任务上优于标准微调，其他两个任务除外，原因可能是这两个任务训练集特别大，这时候微调优势更大。相比传统微调，p-tuning 能够提高 gpt 在 nlu 任务上的表现，和 bert 的效果持平甚至更好，打破了以前的刻板印象。

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

\*\*few-shot setting：\*\*在 fewGLUE 上评测，给定了 32 个训练样本，另外 32 个验证样本是从 superGLUE 中找的，超参数直接用的 PET 这篇的，微调 3500steps。

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

\*\*基线 PET 的设置：\*\*为确保公平比较，PET 在 32 个验证集样本情况 下实验，同时移除所有辅助技术（数据增强、集成和蒸馏），由于使用了多个 prompt，所以报告的是平均性能和最佳性能。

P-tuning 在所有任务上的手动提示始终优于 PET 和 PET-best，同时在某些任务上 P-tuning 还优于采用了这些辅助技术的 PET 模型，也就是 iPET。相比GPT-3，P-tuning（albert-xxlarge-v2）在多个任务上还优于 GPT-3。这些证明了两点：一是 P-tuning 能够搜索到比手工更好的模版，二是 P-tuning 提供了对一些难以微调的大模型的新利用方式。

**手工制定的 prompt v.s. p-tuning：**

- few-shot 的表现与 prompt 的语义、格式、语法没有明显的相关性。人类认为合理的 prompt 不一定对语言模型有效。其次，手工制定的 prompt 的微小变化会导致显着的性能差异。预训练语言模型对提示的选择非常敏感；

- 与采用相同 anchor 的人工模版相比，P-tuning 有明显优势；

- 对于只有少量的验证集样本，很难选择出很好的手工制定的 prompt。

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**2.4 总结**

- template design：continuous

- verbalizer design：none

- tuning strategy：superGLUE 上是 Prompt+LM Tuning，LAMA 上是 Fixed LM Prompt Tuning

- 在同时微调 Prompt+LM 的前提下，supervised setting 中无论是 GPT2 还是 BERT 模型，P-tuning 在大部分情况下都显著提升了传统 Fine-tuning 的效果

- 在同时微调 Prompt+LM 的前提下，few-shot setting 中 P-tuning 全面优于经典的 PET 算法以及 GPT-3 的 in context learning

- 相比于传统的 few-shot setting，提出了一种更实际的设定：32 个 dev 样本；同时强调对于只有少量的验证集样本，很难选择出很好的手工制定的 prompt；而 P-tuning 相比人工 prompt 有显著的优势；

- 在 LAMA 测试中采用的 Fixed LM Prompt Tuning 为超大模型微调提供了范式。

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**P-tuning V2**

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**论文标题：**

P-Tuning v2: Prompt Tuning Can Be Comparable to Fine-tuning Universally Across Scales and Tasks

**论文链接：**

https://arxiv.org/abs/2110.07602

**代码链接：**

https://github.com/thudm/p-tuning-v2

### **3.1 动机**

\*\*规模通用性：\*\*在 Fixed LM Prompt Tuning 并采用全量数据的前提下，Prompt Tuning (The Power of Scale for Parameter-Efficient Prompt Tuning) 被证明能够匹敌 Fine-tuning 的效果，而只需要很少的参数微调：但是要求是 10B 以上的参数量的预训练模型，以及特殊的初始化技巧等。对于普通模型，能不能在 Fixed LM Prompt Tuning+全量数据情况下匹敌 Fine-tuning？

\*\*任务通用性：\*\*尽管 P-tuning 在 SuperGLUE 上表现很好，对于一些比较难的 token-level 的任务表现就差强人意了，比如阅读理解和NER，当然现在也有一些工作在用 prompt 做序列标注（template-NER，lightNER，template-free NER）。还有一个问题是，不是所有标签都有明确的语义，verbalizer 这边映射的 label words 都是有具体含义的，对于一些没有 label 语义的分类任务应该怎么办，比如用户评论的聚类等。

为了解决上面两个痛点，提出了 P-tuning 的 V2 版本。

### **3.2 方法**

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

\*\*Deep Prompt Tuning on NLU：\*\*采用 Prefix-tuning 的做法，在输入前面的每层加入可微调的参数。

\*\*去掉重参数化的编码器：\*\*以前的方法利用重参数化功能来提高训练速度和鲁棒性（例如，用于 prefix-tunning 的 MLP 和用于 P-tuning 的 LSTM）。在 P-tuning v2 中，作者发现重参数化的改进很小，尤其是对于较小的模型，同时还会影响模型的表现。

\*\*可选的多任务学习：\*\*Deep Prompt Tuning 的优化难题可以通过增加额外的任务数据或者无标注数据来缓解，同时可微调的 prefix continuous prompt 也可以用来做跨任务的共享知识。比如说，在 NER 中，可以同时训练多个数据集，不同数据集使用不同的顶层 classifer，但是 prefix continuous prompt 是共享的。

\*\*回归传统的 CLS 和 token label classifier：\*\*主要是为了解决一些没有语义的标签的问题。

**3.3 实验**

注意：这里的 setting 是全量数据+Fixed LM Prompt Tuning。

**规模通用性：**

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

对于简单的 NLU 任务，例如 SST-2（单句分类），P-tuning & prompt-tuning 在较小的规模上没有显示出明显的优势。当涉及到自然语言在推理和问答方面的复杂挑战时，它们的表现可能会很差。

相反，P-tuning v2 在较小规模的所有任务中与微调的性能相匹配。P-tuning v2 在 RTE 中还明显优于微调，尤其是对于 BERT。

在更大规模的参数上（2B 到 10B），P-tuning & prompt-tuning 和微调之间的差距逐渐缩小。在 10B 尺度上，prompt-tuning 变得比微调更具竞争力。当然，P-tuning v2 总是在所有规模上可以媲美微调的效果，而只需微调 0.1% 的特定任务参数。

**任务通用性：**

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

在 NER，QA 和 SRL 三个任务上测试：

- NER：对于多任务设置，结合三个数据集的训练集进行训练。对每个数据集使用不同的线性分类器，同时共享 continuous prompt 的参数；

- QA：对于多任务设置，混合 SQuAD 1.1 和 2.0 的数据集进行训练，不管是否有不可回答的问题；

- SRL：同 NER 的多任务设置。

从表里可以看到 P-tuning V2 的优势以及多任务学习的优势（除了 QA，作者认为是混合了不可回答的问题的原因）。

**3.4 总结**

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

- template design：continuous

- verbalizer design：none

- tuning strategy：Fixed LM Prompt Tuning + 全量数据

- P-tuning V2 在不同规模的预训练模型上都能取得媲美 fine-tuning 的表现，同时也能在一些比较难的 NLU 任务上取得更好的效果，未来研究的一个很强的 baseline。

**特别鸣谢**

感谢 TCCI 天桥脑科学研究院对于 PaperWeekly 的支持。TCCI 关注大脑探知、大脑功能和大脑健康。

**更多阅读**

[!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)](http://mp.weixin.qq.com/s?__biz=MzIwMTc4ODE0Mw==&mid=2247540033&idx=2&sn=f9a9dba9c1e38b5fc58e65541bccc42f&chksm=96ea82c1a19d0bd71aad804f1143e4681c981f4381f60290c9057f3f6de9f685d30fe6bece6e&scene=21#wechat_redirect)

[!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)](http://mp.weixin.qq.com/s?__biz=MzIwMTc4ODE0Mw==&mid=2247539867&idx=3&sn=adf2c3706d0341d1836f736fb4d8d6a6&chksm=96ea831ba19d0a0d2f14e495ea8f9012244b92da22166e3edd586559f5b349880527a6358b98&scene=21#wechat_redirect)

[!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)](http://mp.weixin.qq.com/s?__biz=MzIwMTc4ODE0Mw==&mid=2247538771&idx=1&sn=557e99c3997daf19cf81f674d7a7fd09&chksm=96ea87d3a19d0ec5259bc17dc4ea8396cd85daa0d13aa4384a0e04db2781b1f4609032a96fe4&scene=21#wechat_redirect)

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**#投 稿 通 道#**

**让你的文字被更多人看到**

如何才能让更多的优质内容以更短路径到达读者群体，缩短读者寻找优质内容的成本呢？**答案就是：你不认识的人。**

总有一些你不认识的人，知道你想知道的东西。PaperWeekly 或许可以成为一座桥梁，促使不同背景、不同方向的学者和学术灵感相互碰撞，迸发出更多的可能性。

PaperWeekly 鼓励高校实验室或个人，在我们的平台上分享各类优质内容，可以是**最新论文解读**，也可以是**学术热点剖析**、**科研心得**或**竞赛经验讲解**等。我们的目的只有一个，让知识真正流动起来。

📝 **稿件基本要求：**

• 文章确系个人**原创作品**，未曾在公开渠道发表，如为其他平台已发表或待发表的文章，请明确标注

• 稿件建议以 **markdown** 格式撰写，文中配图以附件形式发送，要求图片清晰，无版权问题

• PaperWeekly 尊重原作者署名权，并将为每篇被采纳的原创首发稿件，提供**业内具有竞争力稿酬**，具体依据文章阅读量和文章质量阶梯制结算

📬 **投稿通道：**

• 投稿邮箱：hr@paperweekly.site

• 来稿请备注即时联系方式（微信），以便我们在稿件选用的第一时间联系作者

• 您也可以直接添加小编微信（**pwbot02**）快速投稿，备注：姓名-投稿

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**△长按添加PaperWeekly小编**

🔍

现在，在\*\*「知乎」\*\*也能找到我们了

进入知乎首页搜索\*\*「PaperWeekly」\*\*

点击\*\*「关注」\*\*订阅我们的专栏吧

·

![](http://mmbiz.qpic.cn/mmbiz_png/VBcD02jFhgklOsJKfNKCYFCiaBOjViaarib352vjdQc2vvcV7BEicdEsZJEonTkeZMsqh3nx2s1NzAUmsRNHM7Og3Q/300?wx_fmt=png&wxfrom=19)

**PaperWeekly**

PaperWeekly是一个推荐、解读、讨论和报道人工智能前沿论文成果的学术平台，致力于让国内外优秀科研工作得到更为广泛的传播和认可。社区：http://paperweek.ly | 微博：@PaperWeekly

1730篇原创内容

公众号

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

预训练模型177

自然语言处理412

预训练模型 · 目录

上一篇EMNLP 2021 | ST-ToD：小样本场景下的任务型对话预训练下一篇从多篇2021年顶会论文看多模态预训练模型最新研究进展

Read more

Reads 5176

​

Comment

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/VBcD02jFhgklOsJKfNKCYFCiaBOjViaarib352vjdQc2vvcV7BEicdEsZJEonTkeZMsqh3nx2s1NzAUmsRNHM7Og3Q/300?wx_fmt=png&wxfrom=18)

PaperWeekly

13Share10

Comment

Comment

**Comment**

暂无留言
