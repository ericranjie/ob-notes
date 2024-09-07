# 

Original 玄姐 玄姐聊AGI

 _2024年09月02日 08:18_ _北京_

▼最近直播超级多，**预约**保你有收获

![](https://wx.qlogo.cn/finderhead/oXsjGNic4LhSPI5Ie3dboPSictzdUMpFsRHj5CmbYMVhs/0)

**玄姐谈AGI**

，

09月02日 20:00 直播

已结束

AI Agent 智能客服落地实践

视频号

_**_—_0**__—_

**背景**

我们要把 **AI 大模型当做人的大脑**，因此调用 AI 大模型，相当于调用一个人，把 AI 大模型当人看，TA 懂人话、TA 说人话、TA 会直接给出结果，但结果不一定正确。

因此在 AI 大模型的推理基础上，通过 RAG、Agent、知识库、向量数据库、知识图谱等技术手段实现了真正的 AGI（通用人工智能）。这些技术到底有哪些区别和联系，下图作了横向对比，接下来我们详细剖析下。

![Image](https://mmbiz.qpic.cn/mmbiz_png/9TPn66HT933d8fvNCsdOwoefQttUpn9UyxEzTQGgsu3UAvVTXmAZRVTSqS8uMibECX8IaSmrZ4d3VEUOumL3h0w/640?wx_fmt=png&from=appmsg&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

_**_—_1**__—_

****大语言模型（LLM）****

大语言模型（LLM）是通过深度学习方法，利用庞大的文本数据集进行训练的机器学习模型，它具备生成自然流畅的语言文本以及准确理解语言文本深层语义的能力。大语言模型广泛应用于各种自然语言处理任务，包括但不限于文本分类、智能问答以及人机交互对话等，是 AI 领域的重要支柱之一。  

过去的一年中，大语言模型及其在 AI 领域的应用受到了全球科技界的广泛关注。特别值得注意的是，这些大语言模型在规模上取得了显著的增长，参数量从最初的数十亿激增到如今惊人的万亿级别。这一飞跃性的增长不仅使得大语言模型在捕捉人类语言的微妙差异上更为精准，更让它能够深入洞察人类语言的复杂本质。

随着 OpenAI GPT-4o 的发布，回顾过去的一年，大语言模型在多个方面取得了显著的进步，包括高效吸纳新知识、有效分解复杂任务以及图文精准对齐等。随着技术的不断演进和完善，大语言模型将继续拓展其应用边界，为人们带来更加智能化、个性化的服务体验，从而深刻改变我们的生活方式和生产模式。

![Image](https://mmbiz.qpic.cn/mmbiz_png/9TPn66HT933d8fvNCsdOwoefQttUpn9UttTf1e0WLzFzSgzYuWAE5fbgPiamtBjFQYNJ4U5GZIkuTUqtzHRXQdQ/640?wx_fmt=png&from=appmsg&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

**大语言模型拥有推理能力，TA 是一切应用的基石。**

_**_—_2**__—_

**检索增强生成（RAG）**

RAG（Retrieval-Augmented Generation）技术是一种集成检索与生成双重能力的知识增强方案，旨在应对复杂多变的信息查询和生成挑战。在如今的大模型时代背景下，RAG 巧妙地引入外部数据源，比如：本地知识库或企业信息库，为 AI 大模型赋予了更强大的检索和生成实力，从而显著提升了信息查询和生成的品质。  

![Image](https://mmbiz.qpic.cn/mmbiz_png/9TPn66HT933d8fvNCsdOwoefQttUpn9U2QOFZvfvgzpibMH8VTb1BicnQwnYy3TJ6Xpz6emicTYuS0AEpaicYWGubg/640?wx_fmt=png&from=appmsg&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

RAG 技术的核心在于它将先进的向量数据库与大模型的智能问答能力进行了完美结合。知识库中的信息被精心存储在向量数据库中，当接收到用户的问题时，系统能够迅速从知识库中检索出相关的知识片段。随后，这些片段会与大模型的智慧相结合，共同孕育出精确而全面的回答。这种技术的运用极大地提高了 AI 系统在处理复杂问题时的准确性和响应速度，为用户带来了更加优质和高效的体验。

**总之，RAG 技术就是给大语言模型新知识。**

_**_—_3**__—_

**Fuction Calling（函数调用）**

大模型要实现精确的函数调用（Function Calling）需要理解能力和逻辑能力，理解能力就是对用户的 Prompt 提示词能够识别意图，然后通过逻辑能力给出需要调用执行的函数，具体流程如下：  

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**1、大模型何时会调用函数 API？**

调用函数 API 在交互形式上有两种方式：第一是让用户直接选择调用函数，第二是大模型会推理判断要调用的函数 API。

**2、大模型怎么 Function Calling 调用函数 API ？**

首先把函数 API 的元信息（函数名称、函数描述、函数参数等）注册给大模型，让大模型学习函数集合，当用户查询时，大模型根据用户的 Prompt 提示词选择对应的函数 API。

**3、函数 API 谁来具体执行？**

大模型根据用户的 Prompt 请求确定具体的函数 API 后，由 Agent 负责具体的执行。

**4、函数 API 返回的内容咋处理？**

Agent 把 Function Calling 函数 API 调用返回的结果返回给大模型，大模型进一步加工处理后返回给用户最终结果。

  

_**_—_4**__—_

****智能体（Agent）****

在 AI 大模型时代，任何具备独立思考能力并能与环境进行交互的实体，都可以被抽象地描述为智能体（Agent）。这个英文词汇在 AI 领域被普遍采纳，用以指代那些能够自主活动的软件或硬件实体。在国内，我们习惯将其译为“智能体”，尽管过去也曾出现过“代理”、“代理者”或“智能主体”等译法。

智能体构建在大语言模型的推理能力基础上，对大语言模型的 Planning 规划的方案使用工具执行（Action） ，并对执行的过程进行观测（Observation），保证任务的落地执行。

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**总之，Agent 智能体 = 大语言模型的推理能力 + 使用工具行动的能力。**

**![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  


**

_**_—_5**__—_

**知识库**

对于企业而言，构建一个符合自身业务需求的知识库是至关重要的。通过RAG、微调等技术手段，我们可以将通用的大模型转变为对特定行业有着深度理解的“行业专家”，从而更好地服务于企业的具体业务需求。这样的知识库基本上适用于每个公司各行各业，包括：市场调研知识库、人力资源知识库、项目管理知识库、技术文档知识库、项目流程知识库、招标投标知识库等等。

知识库的技术架构分为两部分：

**第一、离线的知识数据向量化**

- 加载：通过文档加载器（Document Loaders）加载数据/知识库。
    
- 拆分：文本拆分器将大型文档拆分为较小的块。便于向量或和后续检索。
    
- 向量：对拆分的数据块，进行 Embedding 向量化处理。
    
- 存储：将向量化的数据块存储到向量数据库 VectorDB 中，方便进行搜索。
    

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**第二**、在线的知识检索返回****

- 检索：根据用户输入，使用检索器从存储中检索相关的 Chunk。
    
- 生成：使用包含问题和检索到的知识提示词，交给大语言模型生成答案。
    

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**总之，知识库是 AI 大模型应用的知识基础。**

_**_—_6**__—_

**向量数据库**

向量数据库是专注于存储和查询向量的系统，其向量源于文本、语音、图像等数据的向量化表示。

相较于传统数据库，向量数据库更擅长处理非结构化数据，比如：文本、图像和音频。在机器学习和深度学习中，数据通常以向量形式存在。

向量数据库凭借高效存储、索引和搜索高维数据点的能力，在处理比如：数值特征、文本或图像嵌入等复杂数据时表现出色。

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**总之，知识库的存储载体往往是向量数据库，另外在数据存储和检索上，向量数据库以向量空间模型高效存储和检索高维数据，为 AI 大模型和 Agent 智能体提供强有力的数据支持。**

_**_—_7**__—_

**知识图谱**

知识图谱是一种基于实体和关系的图结构数据库，旨在表示和管理知识。它采用结构化数据模型来存储、管理和显示人类语言知识。

知识图谱通过语义抽取建立人类语言知识间的关系，形成树状结构。实体如人、地点、组织等，具有特定属性和关系，这些关系连接着不同的实体。通过数据挖掘、信息处理和图形绘制，知识图谱揭示了知识领域的动态发展规律，为学科研究提供了有价值的参考。

医疗领域是知识图谱技术的一个广泛应用场景，它可以帮助临床诊疗、医疗数据的整合与利用，并通过实体识别、关系抽取和数据集训练，以图谱形式展示关键节点和它们之间的联系，从而支持更精准的医疗决策。

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

与此同时，在智能推荐、自然语言处理、机器学习等领域也具有广泛的应用。尤其在搜索引擎领域，它能够提高搜索的准确性，为用户提供更加精准的搜索结果。

**总之，知识图谱本质上是一种叫作语义网络的知识库，即一个具有有向图结构的知识库，其中图的结点代表实体或者概念，而图的边代表实体/概念之间的各种语义关系。**

_**_—_8**__—_

**AGI**

AGI（通用人工智能）作为 AI 发展的终极愿景，追求的是让智能系统具备像人类一样理解和处理各种复杂情况与任务的能力。在实现这一宏伟目标的过程中，**AI 大模型、Prompt Engineering、Agent 智能体、知识库、向量数据库、RAG 以及知识图谱**等技术扮演着至关重要的角色。这些技术元素在多样化的形态中相互协作，共同推动 AI 技术持续向前发展，为实现 AGI 的最终目标奠定坚实基础。

为了帮助同学们彻底掌握 AI 大模型 Agent 智能体、知识库、向量数据库、 RAG、微调私有大模型的**应用开发、部署、生产化**，今天我会带来2场直播和同学们深度剖析，请同学们点击以下**预约按钮免费预约**。

![](https://wx.qlogo.cn/finderhead/oXsjGNic4LhSPI5Ie3dboPSictzdUMpFsRHj5CmbYMVhs/0)

**玄姐谈AGI**

，

09月02日 11:30 直播

已结束

使用GraphRAG创建本地化聊天机器人

视频号

![](https://wx.qlogo.cn/finderhead/oXsjGNic4LhSPI5Ie3dboPSictzdUMpFsRHj5CmbYMVhs/0)

**玄姐谈AGI**

，

09月02日 20:00 直播

已结束

AI Agent 智能客服落地实践

视频号

  

_**_—_9**__—_

**领取 AI 大模型学习资料**

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)今天给大家搞到的是一份**大厂****内部**都在用的**『AI 大模型学习资源』****：**  

  

**▶形式：****50+场直播**实战课

****▶**费用：****原价299，本号用户0元白嫖**

▶**内容：******大模型原理、Agent、LangChain、Spring AI、RAG、向量数据库、知识库、私有大模型、算力评估...****

**扫码预约报名**

**👇****『AI 大模型学习资源』****👇**

**堪称资源界的YYDS！**![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**“得此资源，堪比1000G网盘资源”**

👇👇👇

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

本期名额有限

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

_**_—_10**__—_

****领取《AI **大模型技能图谱**》****

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**这份业界首创知识图谱和学习路线，**今天**免费**送了!

**第一步**：**长按扫码****以下视频号，你身边需要一个 AI 专家。**

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**第二步：点击****"关****注按钮"，就可关注。**

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**第三步：点击"客服“按钮，回复**“**知识图谱**”**即可领取。**

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

 _**_—__11_**__—_

****每日精选 AI 大模型知识****

玄姐谈AGI

，赞6

  

_**_—__12_**__—_

**加我微信**

**有很多不方便公开发公众号的我会直接分享在****朋友圈****，****欢迎你扫码加我个人微信来看****👇**

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

****⬇戳”阅读****原文“，立即预约！****

END

  

大模型268

RAG56

Agents46

向量数据库13

FuncitonCalling2

大模型 · 目录

上一篇一台MacBook搭建商用级RAG知识库下一篇比裁员更侮辱人的事发生了。。。

Read more

Reads 1083

​

Comment

**留言 1**

- 玄姐谈AGI
    
    北京5天前
    
    Like2
    
    点我头像，预约最新案例实战直播![[抱拳]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)![[抱拳]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)
    
    PinnedAuthor liked
    

已无更多数据

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/9TPn66HT933xt6zN4p9SK7k58GuhNMcKqTbXJg6qIwhiaJJmEjMUdciaWTXa2ZV8R72FDoOiaNStRn5sVcvgfNx9A/300?wx_fmt=png&wxfrom=18)

玄姐聊AGI

121387

1

Comment

**留言 1**

- 玄姐谈AGI
    
    北京5天前
    
    Like2
    
    点我头像，预约最新案例实战直播![[抱拳]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)![[抱拳]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)
    
    PinnedAuthor liked
    

已无更多数据