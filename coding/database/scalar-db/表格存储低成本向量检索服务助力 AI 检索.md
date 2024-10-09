# 

Original 寻剑 阿里云开发者

_2024年09月05日 08:30_ _浙江_

![Image](https://mmbiz.qpic.cn/mmbiz_jpg/Z6bicxIx5naKrDkHRiaSlsQTMgBicQkzT4u5emBHqpzYlbAqibCLkiayf8YGdlZsxdIZ5gUyAGgH6aPOuylO8vj72bA/640?wx_fmt=jpeg&from=appmsg&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

阿里妹导读

本文阐述了阿里云表格存储（Tablestore）如何通过其向量检索服务应对大规模数据检索的需求，尤其是在成本、规模和召回率这三个关键挑战方面。

在当今 GPT 技术盛行的时代，大模型推动了向量检索技术的迅猛发展。向量检索相较于传统的基于关键词的检索方法，能够更精准地捕捉数据之间的语义关系，极大提升了信息检索的效果。特别是在自然语言处理、计算机视觉等领域，向量能够将不同模态的数据在同一空间中进行表达和检索，推动了智能推荐、内容检索、RAG 和知识库等应用的广泛普及。

阿里云表格存储（Tablestore）的多元索引提供了向量检索能力。表格存储是一款 Serverless 的分布式结构化数据存储服务，诞生于 2009 年阿里云成立时，主要特点是分布式、Serverless 开箱即用、按量付费、水平扩展和查询功能丰富和性能优秀等。

场景

这里介绍两个经典的向量数据库场景，一是新兴的 RAG 应用，二是传统的多模态搜索，通过整合向量数据库，用户可以高效和灵活的访问大规模数据。

**场景一：新兴的RAG应用**

RAG（Retrieval-Augmented Generation）是一种结合检索与生成的方法，主要用于自然语言处理任务。它通过首先从知识库中检索相关语料，再将这些信息输入给生成式模型，用于对话或文本生成。这种方法的优势在于能够同时结合事实或私域信息与生成模型的双重能力，生成更准确和丰富的文本。

﻿![Image](https://mmbiz.qpic.cn/mmbiz_jpg/Z6bicxIx5naKrDkHRiaSlsQTMgBicQkzT4ukIBKKVl8ZfpOq4ylxHhvbRvoMGTR4GEUEhjx9iaf5YY9DWSArVNVVzA/640?wx_fmt=other&from=appmsg&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)﻿

我们以智能客服助手为例，其整个系统的核心是一个基于向量数据库的 RAG 模型。此场景中，当客户通过在线聊天框提出问题时，系统首先将问题转化为向量表示，并在向量数据库中进行检索。向量数据库快速找到与客户问题相关的文档、FAQ、历史聊天记录。这些数据被送入生成模型中，模型根据检索到的信息，生成一条简洁明了的回复。与传统的基于固定规则的客服机器人相比，这种基于向量数据库的 RAG 模型能够处理更复杂、开放性的问题，并根据用户的需求提供个性化的回答。

RAG 在问答系统、内容创作、语言翻译、信息检索、对话系统等领域具备极大的优势，这些领域需要大量的背景信息和上下文理解，相比传统的基于全文检索和基于规则的方式更加准确，更加自然和符合语境。

**场景二：传统的图片、视频、文本**

传统的图片、视频、文本搜索是经典的多模态检索问题，在以网盘为代表的系统里最为常见。过去，用户只能通过关键词搜索来查找他们想要的内容，会因为关键词不准确而查询不到目标作品。因此，各类网盘开始引入基于向量数据库的多模态搜索方案。

﻿!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)﻿

在这个方案中，用户输入一段描述或上传一张图片，系统会将这些输入转化为向量。向量数据库能够在海量的数据中检索出与用户输入相似度高的图像和视频。例如，当用户上传一张风景照片时，搜索引擎能够识别出照片中的山脉、湖泊等元素，迅速找到其他类似的风景图片和视频，并将其展示在搜索结果中。此外，用户也可以在查询时使用自然语言表述，比如“寻找以秋天为主题的风景”，系统会在向量数据库中查找与秋天相关的多张图像和视频，提供更精准的搜索结果。

业界方案

在向量数据库领域，技术方案主要有两种：基于 HNSW 的图算法和聚类方法。这两种方法在高维数据的检索与存储上都有着广泛的应用，尤其是在处理大规模数据集时，它们各自有优点和局限性。

**HNSW 图算法**

HNSW 是一种基于图的数据结构，用于高效地进行最近邻搜索。它通过构建一个多层次的图来实现快速查询。在这个结构中，数据以节点形式存在，节点之间的边表示它们的相似性。HNSW 将节点分层，每一层的节点数量逐级减少，从而在查询时，可以从较高层次开始，迅速缩小搜索范围，大大加快了查找速度。

HNSW 图算法的最大优点是召回率高，性能好。但是缺点也很明显，一是数据需要全部加载在内存里面里面，成本很高。二是规模受内存限制，很难支持到千万级以上规模。

﻿!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)﻿

**聚类算法**

聚类是将数据集中相似的对象归为一类的过程。通过将数据进行聚类，可以将相似的向量分到同一个群组中，在检索时，可以先确定数据点属于哪个聚类，再在该聚类内进行详细搜索。这种方法的优势在于减少了需要访问的数据量，从而提升检索速度。

聚类算法的最大优点是成本低，索引性能好，但是缺点也很明显，召回率低。常见的聚类算法包括 K-means、层次聚类和 DBSCAN 等。在高维空间中，不同的聚类算法对数据的分布特点有不同的适应性，因此选择合适的聚类算法及其参数很重要，否则会出现聚类数目、聚类中心等选择不佳导致向量检索效果不理想。

﻿!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)﻿

**开源系统**

开源的向量检索产品十分丰富，分为“专用向量数据库”和“传统数据库附加向量检索功能”。专用向量数据库由于其数据库方面的特性发展相对不完善，还有很大提升空间，因此大多数用户倾向于使用传统的附加向量检索功能的数据库，比较常见的有如下一些相关产品。

- PG：通过 pgvector \[1\]插件整合了向量检索能力，索引能力包括FLAT索引提供线性暴力扫描、基于聚类的 IVFFLAT、 基于图的 HNSW。

- Redis：通过 RedisStack \[2\]提供了向量检索，索引能力包括FLAT索引提供线性暴力扫描、基于图的 HNSW。

- Elasticsearch：通过 Lucene \[3\]提供了向量检索能力，索引能力包括 FLAT 索引提供线性暴力扫描、基于图的HNSW。

问题和挑战

尽管当前向量数据库的实现方案特别多，但还是存在一些问题和挑战，尤其是在向量检索最核心的成本、规模、召回率三个方面。

**1.成本**

成本是影响用户选择向量数据库的一项重要因素。在向量数据库里，构建和维护一个向量索引需要庞大的资源，这些成本主要体现在计算资源（CPU）和 I/O 资源（内存）上。在计算资源上，基于图的算法在构建和维护图时候需要的计算资源特别高，基于聚类的算法实现简单相对来说计算资源使用较低，但是两个算法在多次覆盖插入或删除操作后，索引结构的维护成本特别高，且召回率偏低。在 I/O 资源上，因为向量索引大多数场景都是随机 I/O，特别是以 HNSW 为代表的图索引随机 I/O 问题特别严重，业界一般将整个索引以 MMAP 的方式全部加载到内存里，导致内存成本特别高。因此，在考虑为大量非结构化和半结构化的数据增加向量能力的时候，成本成为了最大的拦路虎。

**2.规模**

未来数据规模一定是会不断地增长，所以对向量数据库的规模要求会更高。尤其是在企业网盘、语义搜索、视频分析以及行业知识库等领域，预计数据规模将达到千万甚至数十亿级别。这种趋势使得向量数据库在应对大规模数据时面临更高的要求。单机系统在个人知识库和小规模文档处理上仍然有效，但在更复杂的大数据环境下，必须考虑数据库的扩展能力，以满足未来不断增长的数据需求。

向量检索系统中，支撑大规模数据需要通过分布式架构来实现。借助分布式系统，可以将数据分散存储，提升检索效率和系统可靠性。然而，实现分布式架构并非易事，其中包括数据一致性、负载均衡、高可用性等问题。此外，向量数据与标量数据的混合存储和检索也需要合理的设计，以保证系统在高并发和大规模条件下的稳定运行。

大规模必然带来更高的成本挑战。向量检索高昂的成本限制了很多传统数据库的扩展性，使其难以适应较大的向量检索数据量。以传统的 MySQL 和 PostgreSQL 这些单机数据库为例，为了保持最佳性能，通常单表规模较小，特别是向量场景下，对机器规格要求会更高，需要更多的 CPU 和内存配置。Elasticsearch 作为搜索领域的行业翘楚也支持向量检索，支持暴力扫描和 HNSW 算法，虽然标量数据可以支持到千亿级别，但是向量数据由于其基于内存的实现，很难支撑到特别大的规模。

**3.召回率**

召回率是衡量向量数据库查询效果的重要指标。基于 HNSW 的图算法通常表现相对较好，但在某些情况下可能无法保证高召回率，尤其是当数据规模变大和分布不均匀时导致出现噪声，以及查询时候会出现参数不对导致的局部最优问题，这些都可能导致召回率下降。基于聚类的方案的召回率因使用方式而异，在第一阶段聚类中心找对的情况下，召回率会非常高，然而过于依赖聚类中心可能导致“聚类过细”，从而使得召回率反而下降，如果聚类的类簇个数较少时候，会出现单个类簇的向量数据量太大导致查询性能较差，依赖一些高成本的 GPU 硬件进行加速，整体上而言，聚类的召回率会比 HNSW 图算法更低。

在一些高召回率的场景下，虽然聚类可以解决规模和成本问题，但是解决不了召回率问题，那么有没有一个系统既能支持大规模数据、高召回率，还能低成本较低呢？

表格存储的向量检索服务

表格存储针对上述在向量检索领域遇到的成本、规模、召回率等挑战，发布了低成本、大规模、高性能、高召回率的向量检索服务，其能以较低成本支持千亿规模向量数据的存储和检索，接下来介绍一下为什么可以做到上述这些优势。

﻿!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)﻿

**1.DiskANN**

表格存储向量检索在原有的 DiskANN 算法之上进行优化，提供大规模和高性能的向量检索服务。前期，我们也先对用户提供了基于 HNSW 的图算法，但是发现明显的性能问题，HNSW 对内存要求很高，很难支撑大规模数据，当内存不足时，HNSW 访问磁盘的性能极差。表格存储上的用户大多数据量相对较大，HNSW 的规模和成本问题不符合我们产品的定位，我们希望可以让更多的数据可以低成本的使用上向量能力，因此我们后续废弃掉了 HNSW 算法，使用 DiskAnn 重新实现了向量检索能力。﻿

DiskAnn 该算法专为磁盘设计，其内部为磁盘友好的层次化图结构，能够处理远超内存容量的数据集。在表格存储内部，我们只需要将图索引的 10% 的数据放进内存，剩余 90% 的数据放在磁盘上就能达到和 HNSW 相近的高召回率和性能，但是内存成本只有 HNSW 的 10%，这个就是低成本的原因。在规模上，经过引入 DiskANN 和表格存储的算法和分布式框架优化，最终能支持到千亿级别向量数据的存储和检索。

但是 DiskAnn 需要很高的 I/O 能力，对磁盘要求很高，大部分基于磁盘的 DiskAnn 会遇到被 I/O 能力限制的问题，那我们是怎么解决的了？这个就要看我们的存算分离架构了。

**2.存算分离**

表格存储使用了存储计算分离架构，底层使用阿里云自研的高性能高可靠地分布式文件系统盘古，计算和存储支持独立的弹性伸缩，不带来额外费用前提下默认支持 3AZ 能力，为用户提供极致的弹性能力和容灾能力。

传统的以本地磁盘为媒介的向量数据库，在访问磁盘上的图索引时候会存在 I/O 瓶颈，IOPS 受限，很容易出现单块磁盘热点，很难发挥 DiskANN 的真正实力。表格存储使用的分布式文件系统则不存在该问题，所有 I/O 都是离散的，不会出现热点，解决了 DiskANN 的I/O瓶颈问题。

除此之外，表格存储在盘古文件系统之上，搭建了自主可控的 BlockCache 缓存系统来缓存索引文件，使用高效的缓存机制和数据预取策略减少 I/O，提高整体性能。相比传统的基于本地磁盘的 MMAP 加载方式，MMAP 的资源使用不受控，不方便进行隔离和管理，会存在 Cache 竞争和污染问题。

**3.混合索引：Flat、PQ、DiskANN、标量索引**

表格存储向量检索支持多种索引：Flat、PQ、DiskANN、倒排索引等。这些索引类型不需要用户选择，会根据用户写入模式和数据规模自适应选择最佳方案，实现高效的向量检索，大幅降低用户的参数选择困扰，以及避免参数错误导致的性能不足或者召回率低的问题。无论是实时性要求高的小数据量应用，还是对存储和计算资源敏感的大规模数据量应用，我们的解决方案都能提供出色的检索能力。

﻿!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)﻿

- Flat：最简单和直接的索引，其存储了用户的原始向量，后续可以通过线性暴力扫描的方式进行检索。此方式仅适合小规模数据，尽管时间复杂度较高，但在数据量较小时，可以提供 100% 准确的结果。

- PQ（Product Quantization）：PQ 可以将高维数据压缩为低维特征，通过量化的方式显著减少内存使用并加快检索速度，尤其在处理大规模数据时，PQ 能够更有效地平衡准确性和性能。

- DiskANN：专为大规模数据设计的索引，它将向量存储在磁盘上，通过高效的图索引和聚类算法，实现近似最近邻搜索。这一方法不仅大大降低了内存占用，还提高了检索速度，特别适合规模较大的场景。

- 标量索引：用于支持非向量部分的标量数据的查询。表格存储的多元索引在传统标量索引场景下能力十分出众，支持字符串、数字、日期、地理位置、数组、Json 嵌套等各种类型的索引，能持完成任意列查询、多字段自由组合查询、地理位置查询、全文检索、模糊查询、前缀查询、嵌套查询、去重、排序等丰富的功能。

**4.极高的写入吞吐**

表格存储具备极高的向量索引写入吞吐能力，该能力得益于不同的构建策略和远端构建能力。

﻿!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)﻿

#### 构建策略

表格存储的向量索引使用了不同的索引构建策略，在不同规模和场景下自动选择，无需用户关心，避免参数选择困扰。不同的构建策略可以减少 LSM-Tree 架构下的写入放大问题，提升写入吞吐能力，并节省用户成本。

- 数据量少时，仅构建 Flat 索引。

- 数据量中等时，仅构建 Flat 和 PQ 索引。

- 数据量多时，构建 Flat、PQ、DiskANN 等多种索引。

- 后台 Compaction：构建 Flat、PQ、DiskANN 等多种索引。

#### 远端构建（Remote Compaction）

每个集群有独立的远端构建资源组，可以将向量索引的构建任务下放到远端节点执行。由于向量索引的构建是 CPU 密集型任务，本地的计算节点无法跑满 CPU 去构建向量索引，否则会影响普通的写入和查询，并且给计算节点带来极大的稳定性风险。将向量构建放在远端执行则可以更加安全和灵活，向量索引的构建资源也可以和计算节点资源分离，实现弹性伸缩和多租户共享，提高构建性能，节省构建成本。

通过使用远端构建，带来了以下的显著效果：

1.可扩展的极限吞吐能力。向量索引的构建不再局限于本机的 CPU 和内存资源，因此可以支持更大的极限写入 TPS，理论上只要远端节点资源够用，可以不断线性扩展计算资源来提高写入吞吐。

2.最大化的资源利用率。在不引入额外 CPU 资源消耗情况下，将远端构建节点的 CPU 跑满，构建耗时可减少 2~3 倍以上，通常线上集群会有更多公共的远端构建节点，可以支撑用户更短的构建时间。

3.大幅减少毛刺。之前在后台构建向量索引的同时进行普通查询，因 CPU 和内存资源消耗极高，会给普通查询带来较高的查询毛刺，使用远端构建后可以将查询毛刺降低 2 个量级，毛刺从秒级别降低到十毫秒级别。

**5.自适应的查询策略**

#### 索引选择

在一个索引内部，数据是组织在 Segment 这样的最小单元上。在上述的写入介绍中，我们了解到向量检索服务在索引构建时候使用了自适应的策略，每个 Segment 的向量索引类型会不同，在数据量较少时候只有 Flat 原始向量索引，在数据量较多时候会额外构建 PQ 索引，在数据量较大时候会再额外构建 DiskANN 索引。因此在查询时候，会根据每个 Segment 的索引情况进行选择，优先选择已经存在的索引。在所有索引都存在的情况下，需要结合用户查询模式来动态选择，具体的策略见下述的混合查询优化。

﻿!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)﻿

#### 混合查询优化

表格存储支持同时查询标量和向量，在标量和向量同时查询的场景提供了多样化的查询策略，旨在提升用户体验并降低使用复杂度。系统会分析索引特征、数据规模、查询命中规模、查询模式等，能够自适应地选择最优的查询策略。当前支持如下 4 种查询策略：

- Brute Force 线性暴力扫描。

- Pre Filtering 前置筛选标量。

- Post Filtering 后置筛选标量。

- Single Stage Filtering 遍历图索引同时筛选标量。

﻿!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)﻿

性能指标

该测试使用 GIST1M 数据集，100 万行 960 维数据，表格存储和其它开源产品使用相同配置和参数下进行测试。

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

如何使用

您可以使用控制台或者 SDK 使用表格存储的向量检索服务。

- 控制台：登录表格存储控制台\[4\]，完成创建实例、表后，再创建包含向量字段的多元索引，通过控制台导入数据后，可以通过索引管理界面的搜索功能进行在线查询，具体参考文档向量检索介绍与使用\[5\]。

- SDK：您可以通过 Java SDK\[6\]、Go SDK\[7\]、Python SDK\[8\]或Node.js SDK\[9\]使用向量检索功能，具体参考文档向量检索介绍与使用\[10\]。

如果您需要将文本通过词嵌入（Embedding）模型转化为向量，可以参考如下文档：

- ﻿使用开源模型将Tablestore数据转成向量﻿\[11\]

- ﻿使用云服务将Tablestore数据转成向量﻿\[12\]

**参考链接：**

\[1\]https://github.com/pgvector/pgvector

\[2\]https://redis.io/docs/latest/develop/get-started/vector-database/

\[3\]https://github.com/apache/lucene/

\[4\]https://otsnext.console.aliyun.com/

\[5\]https://help.aliyun.com/zh/tablestore/knn-vector-query

\[6\]https://help.aliyun.com/zh/tablestore/developer-reference/vector-retrieval-of-search-index-by-using-java-sdk

\[7\]https://help.aliyun.com/zh/tablestore/developer-reference/vector-retrieval-by-using-go-sdk

\[8\]https://help.aliyun.com/zh/tablestore/developer-reference/vector-retrieval-by-using-python-sdk

\[9\]https://help.aliyun.com/zh/tablestore/developer-reference/vector-retrieval-by-using-nodejs-sdk

\[10\]https://help.aliyun.com/zh/tablestore/knn-vector-query

\[11\]https://help.aliyun.com/zh/tablestore/use-a-open-source-model-to-convert-data-from-tablestore-into-vector

\[12\]https://help.aliyun.com/zh/tablestore/use-the-dashscope-to-convert-the-data-in-the-tablestore-into-a-vector

**分布式训练和部署Llama 2模型**

灵骏支持业界各类流行的开源大语言模型，包括Llama2系列、Bloom系列、Falcon系列、GLM/ChatGLM系列，以及领域大模型galactica等的高效训练和部署。本方案整体可用于企业样本标注、创意文本生成、智能对话助手、文本类创作辅助等场景。

点击阅读原文查看详情。

Read more

Reads 3633

​

Comment

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/Z6bicxIx5naI1jwOfnA1w4PL2LhwNia76vBRfzqaQVVVlqiaLjmWYQXHsn1FqBHhuGVcxEHjxE9tibBFBjcB352fhQ/300?wx_fmt=png&wxfrom=18)

阿里云开发者

202064

Comment

Comment

**Comment**

暂无留言
