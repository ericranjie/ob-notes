# 

康荣 字节跳动技术团队

_2024年08月24日 11:10_

# 从 TPC-C 到 TPC-E

在数据库评测领域， TPC-C 可能是最出名的OLTP 基准测试（benchmark）之一了。各大数据库产品为展现其性能强大，纷纷在 TPC-C 性能榜上你方唱罢我登场。Oracle 一度独占鳌头，阿里 OceanBase、 腾讯 TD-SQL 也轮番登顶。达梦、TiDB、TBase 等等也纷纷用 TPC-C 作为自身产品的性能衡量标准。不仅如此，TPC-C 也在许多下游任务中频繁亮相，例如参数调优任务、负载预测任务、索引推荐任务等等。

然而，TPC-C 作为一个1992 年推出的 OLTP benchmark，库表结构、事务类型、业务场景都显得“过于简单”了。为了应对数据库领域的发展，TPC 委员会在 2007 年推出了下一代 OLTP 基准测试：TPC-E。

在 TPC-E 官网 上，官方开宗明义：**「TPC-E 比」** **「TPC-C」** **「更加复杂，因为它的事务类型更多样化、库表结构和执行结构更复杂」**：

> ❝
>
> TPC-E is more complex than previous OLTP benchmarks such as TPC-C because of its diverse transaction types, more complex database and overall execution structure.
>
> ❞

TPC-E 相比 TPC-C 的复杂性是显而易见的，我们仅列举一些：

||TPC-C|TPC-E|
|---|---|---|
|模拟场景|简单的批发商系统|复杂的证券交易所系统|
|事务类型|5 种|12 种|
|库表|9 张表|33 张表|
|数据生成|随机数，均匀分布|真实数据规律，有倾斜（skew）|
|复杂 join|最多 2 表 join|最多 7 表 join|
|读写比|1.9:1|9.7:1（读负载比例更高）|

相比 TPC-C 威名赫赫，TPC-E 由于其复杂而显得小众，在工业界和学术界并没有被广泛地用于性能测试。然而在 TPC-C 已经被研究透彻、各大厂商的评测中纷纷“过度优化”的如今，TPC-E 基准测试不失为一种新的、良好的补充。

本文接下来你会看到：

1. **「概览全貌」**：对 TPC-E 做一份详细的讲解，展现 TPC-E 的场景、库表与事务全貌。

1. **「实践挑战」**：借助 “MySQL 索引优化” 这一场景，展现 TPC-E 对现有技术带来的新的挑战。

1. **「原理解析」**：深入SQL 级别，完全拆解 TPC-E 的 12 种事务类型，知其然也知其所以然。

1. **「轻松上手」**：绕过 Github 暗坑，在 MySQL 上编译和运行 TPC-E。

# 概览全貌

![Image](https://mmbiz.qpic.cn/sz_mmbiz_png/5EcwYhllQOhIogOQhodD9em5ZG7AlpW4TibCyg86iaibBHkPtnl7RIjbTYPFKwUlt3iazahwAdyMspibrcCW3wQfBew/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

TPC-E 的场景是股票交易，涉及客户、经纪行和市场三种角色的复杂交互

TPC-E（Transaction Processing Performance Council - E）是一个模拟复杂在线交易处理（OLTP）环境的基准测试。它通过一系列事务来模拟一个股票经纪行的日常业务活动，这些活动涉及客户账户管理、交易执行以及与金融市场的互动。整个业务场景中包含了客户、经纪商、市场数据和后台处理等关键要素。

这里我们从角色（Role）、事务和关系表三部分来展现 TPC-E 全貌。

## 3种角色

TPC-E 模拟的是证券交易所，证券交易的买卖过程会涉及到下面三种角色：

- **「Brokerage（经纪行）」** ：在 TPC-E 基准测试中，经纪行的角色通常由 `Customer Emulator`（客户模拟器）组件扮演。它模拟了客户与经纪行的交互，包括提交交易请求、查询账户信息、执行市场分析等。经纪行角色负责处理客户的交易订单，管理客户账户，并提供市场数据。

- 注意，事务有一种类别是 Brokerage initiated，但代码中并没有单独的 broker emulator，因为 broker 通常是应答customer 的要求，broker 参与的事务就放到 CE 中模拟

- **「Customer（客户）」** ：客户角色代表了实际使用经纪行服务的个人或机构投资者。在 TPC-E 中，客户角色通过 `Customer Emulator` 组件模拟，执行各种交易活动，如买卖证券、查询持仓情况、查看市场动态等。客户角色的目的是评估经纪行提供的服务和交易平台的性能。

- **「Market（市场）」** ：市场角色在 TPC-E 中由 `Market Exchange Emulator`（市场交易所模拟器）组件扮演。它模拟了股票市场的实际运作，包括股票价格的变动、交易的执行、市场数据的发布等。市场角色为经纪行和客户提供了交易的场所和必要的市场信息。

这三个角色在 TPC-E 中的主要区别在于它们在交易过程中的职责和功能。经纪行负责处理交易和客户账户，客户负责发起交易和查询，而市场则提供了交易发生的环境和数据。

## 12 种事务：一个故事

TPC-E 共包含了 12 种类型的事务，为了便于理解，让我们用一个故事串讲一下。

在一个充满活力的交易日，客户们忙碌地通过经纪行的交易平台进行股票买卖。他们首先会检查自己的账户情况，了解自己的资产和持仓（`Customer-Position` 事务），然后根据市场动态（`Market-Feed` 事务）和特定证券的详细信息（`Security-Detail` 事务）来制定交易策略。在做出决策之前，他们可能会监控市场趋势（`Market-Watch` 事务），或者回顾过去的交易记录（`Trade-Lookup` 事务），以分析证券的历史表现。经纪行管理者会生成不同经纪商的交易报告，用于评估各个经纪商的表现（`Broker-Volume` 事务）。一旦客户决定买卖某只股票，他们会下达交易指令（`Trade-Order` 事务）。这些指令会被提交到市场交易所，并在交易完成后收到交易结果（`Trade-Result` 事务）。这些结果包括了交易的最终确认、成交价格以及可能的税务影响。客户可以通过查看交易状态（`Trade-Status` 事务）来跟踪他们的交易是否成功执行。在交易过程中，客户可能会需要更新或修改他们的交易指令（`Trade-Update` 事务）。同时，为了保持数据的准确性和最新性，经纪行会定期进行数据维护（`Data-Maintenance` 事务），包括更新客户账户信息、税务信息以及市场数据。在交易日结束时，经纪行需要清理数据库，取消任何未完成或错误的交易（`Trade-Cleanup` 事务），以确保第二天的交易能够顺利进行。这个过程包括从数据库中移除所有挂起的交易请求，更新交易历史记录，并确保所有交易数据都是最新的。

## 33 张关系表

TPC-E 共涉及 33 张表：

- **「Customer Tables」**：9 张表，描述了与客户相关的表，包括账户信息（`CUSTOMER_ACCOUNT`）、税务信息（`CUSTOMER_TAXRATE`）等。

- **「Broker Tables」**：9 张表，与经纪商相关的表，如经纪商（`BROKER`）、现金交易（`CASH_TRANSACTION`）、费用（`CHARGE`）等。

- **「Market Tables」**：11 张表，与市场相关的表，如公司（`COMPANY`）、每日市场数据（`DAILY_MARKET`）、交易所（`EXCHANGE`）等。

- **「Dimension Tables」**：维度表，如地址（`ADDRESS`）、状态类型（`STATUS_TYPE`）、税率（`TAXRATE`）等。

TPC 委员会公布的 TPC-E 标准文件（pdf）中事无巨细的讲解了 TPC-E 各方面内容，其中2.2.4 ~ 2.2.7 描述库表设计，感兴趣的同学可以深入了解下：

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

## 衡量标准：tpsE

TPC-E 衡量的标准是 tpsE（transactions- per-second-E，每秒成交量）。在 TPC-E 对真实场景的模拟中，用户和经纪商可能经过许多次的观望、选择、评估，才会达成一笔交易。因此，TPC-E 的性能取决于 `Trade-Result` 事务完成的数量。例如，如果一个客户执行了一项交易，并且该交易被成功处理（即交易请求被接受并执行，Trade-Result + 1），那么这将被视为完成了一个 tpsE。仅仅查看订单或执行其他非交易类型的操作通常不会计算在内。Trade-Result 事务与全部事务的比例基本稳定（例如 10%），也意味着 tpsE 基本可以反映数据库执行的事务总量。考虑到TPC-E 的事务通常较为复杂（单个事务会包含数十条 SQL），在我们执行 TPC-E 测试时，尽管最终显示的 tpsE 只有 100 上下，但实际执行的 SQL 已经超过数十万条。

# 原理解析：深入 SQL 看事务

TPC-E 比 TPC-C 的复杂体现在事务的复杂。TPC-C 包含 5 种事务，SQL 模板共 29 条，而 TPC-E 包含 12 种事务，SQL 模板超过 120 条。在一些复杂的 TPC-E 事务中（例如 Trade-Order），包含 6 个阶段（称为 Frame），每个阶段中会执行多轮”子事务“。由此，在各种任务（参数调优、规格调优、索引推荐）走到深水区后，对事务细节的了解就很有必要了。

下面我们会逐一分析各个事务的事务逻辑概述和 SQL 细节。必要的地方我们会结合 TPCE 负载发生器的源码进行解析。

## 事务分类

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

TPC-E 的事务可以按照它们的功能和特征进行分类。根据文档中的描述，这些事务主要可以分为以下几类：

1. **「客户发起的事务（Customer Initiated）」** ：

1. 这些事务模拟了客户与系统交互的场景，如查询账户信息、执行交易等。

1. 例如：Customer-Position（客户持仓查询）、Market-Watch（市场观察）、一部分 Trade-Lookup（交易查询）、Security-Detail（证券详情查询）、Trade-Order（交易委托）、Trade-Status（交易状态查询）、一部分 Trade-Update（交易更新）。

1. **「经纪商发起的事务（Brokerage Initiated）」** ：

1. 这些事务模拟了经纪商内部处理的场景，如生成报告、管理账户等。

1. 例如：Broker-Volume（经纪商成交量）。一部分 Trade-Lookup；一部分 Trade-Update

1. **「市场触发的事务（Market Triggered）」** ：

1. 这些事务模拟了市场活动对系统的影响，如市场数据更新、市场动态跟踪等。

1. 例如：Market-Feed（市场数据更新）、Trade-Result

1. **「其他」**：

1. Trade-Cleanup、Data-Maintenance

我们结合 Github 源码进行分析。tpce-mysql 中，`DBConnection.h` 文件包含几个 enum，可以作为印证，如下：

`/*   Customer Emulator System Under Test   由用户   */   enum eCESUTStmt   {           // Customer-Position 有2 阶段、4 sql。文档是3 阶段（Frame），但第三阶段只有 commit ，其他有意义的 sql 是对得上的。       CESUT_STMT_CPF1_1,       // Market-Watch（市场观察）       CESUT_STMT_MWF1_1a,       // Security-Detail（证券详情查询）       CESUT_STMT_SDF1_1,       // Trade-Lookup（交易查询），非常巨大的事务           CESUT_STMT_TLF1_1,           // Trade-Order（交易委托）           CESUT_STMT_TOF1_1,           //Trade-Status（交易状态查询）           CESUT_STMT_TSF1_1,           // Trade-Update（交易更新）           CESUT_STMT_TUF1_1,    }      /*   Market Exchange Emulator SUT   */   enum eMEESUTStmt   {       // 极其巨大的事务       MEESUT_STMT_TRF1_1,       // Market-Feed（市场数据更新）       MEESUT_STMT_MFF1_1,   };      /*   Data Maintenance SUT   */   enum eDMSUTStmt   {       // Trade-Cleanup，开测前初始化；       DMSUT_STMT_TCF1_2,   };      // 其他无对应代码 enum 的：   // Broker-Volume（经纪商成交量）：只有一个 frame、一句 sql，无 enum`

除了上述分类，事务还可以根据它们的读写特性进行区分：

- **「读事务（Read-Only）」** ：这类事务主要涉及数据的读取，不会导致数据的修改。例如，客户查询账户信息（Customer-Position）或查看市场数据（Market-Watch）。

- **「读写事务（Read-Write）」** ：这类事务既涉及数据的读取也涉及数据的写入，可能会改变数据库的状态。例如，执行交易（Trade-Order）会创建新的交易记录，更新客户账户（Trade-Update）会改变账户的持仓信息。

- **「写事务（Write-Only）」** ：这类事务主要涉及数据的写入，不涉及数据的读取。例如，数据维护（Data-Maintenance）事务可能会更新或删除数据库中的记录。

概括来看：

1. **「Broker-Volume (BV)」** - 模拟\*\*「经纪行」\*\*内部业务处理，例如生成关于不同经纪人业绩、潜力的报告。

1. **「Customer-Position (CP)」** - 模拟\*\*「客户」\*\*查询其账户的持仓情况。根据所有资产的当前市场价值总结其账户价值。

1. **「Market-Feed (MF)」** - 模拟跟踪当前市场活动，处理来自\*\*「市场交易所」\*\*的“股票行情”数据。

1. **「Market-Watch (MW)」** - 允许\*\*「客户」\*\*跟踪一组证券的当前日常趋势（上涨或下跌），基于客户的当前持仓、观察列表或特定行业。

1. **「Security-Detail (SD)」** - 模拟\*\*「客户」\*\*访问特定证券（`Security`）的详细信息，如进行研究以决定是否执行交易。

1. **「Trade-Lookup (TL)」** - 模拟信息检索，以回答关于一组交易的问题，可能涉及市场分析、交易历史审查或特定客户持仓分析。

1. **「Trade-Order (TO)」** - 模拟\*\*「客户、经纪人」\*\* 或授权第三方购买或出售证券的过程，包括验证授权、执行市场价买卖、限价买卖以及提供财务影响估计。

1. **「Trade-Result (TR)」** - 模拟完成股票市场交易的过程，更新客户持仓，记录交易结果和历史信息。这是由 **「market 市场交易所」** 负责记录的

1. **「Trade-Status (TS)」** - 提供特定交易集合的状态更新，模拟\*\*「客户」\*\*查看其账户的最近交易活动摘要。

1. **「Trade-Update (TU)」** - 模拟对一组交易进行轻微修正或更新，类似于\*\*「客户」**或**「经纪人」\*\*审查交易并进行小的编辑修正。

1. **「Data-Maintenance (DM)」** - 模拟对主要静态数据进行定期修改，如更新参考数据。

1. **「Trade-Cleanup (TC)」** - 用于取消数据库中任何待处理或已提交的交易，通常在测试运行前将数据库恢复到已知状态。

## Broker-Volume

**「Broker-Volume 事务逻辑概述」** 在 TPC-E 基准测试的第 3.3.1 章节中，Broker-Volume 事务是一个典型的读操作，它模拟了经纪行内部生成经纪人业绩报告的场景。这个事务的核心目标是计算每个经纪人在特定时间段内的交易量，这通常涉及到对挂单限价订单（TRADE_REQUEST）的汇总分析。

**「SQL 细节」** Broker-Volume 事务的 SQL 查询设计要实现以下目标：

1. **「选择经纪人列表」**：确定需要生成报告的经纪人。

1. **「检索挂单限价订单」**：从 TRADE_REQUEST 表中检索每个经纪人的订单信息。

1. **「计算总交易量」**：对每个经纪人的订单数量和价格进行计算，得出总交易量。

1. **「排序结果」**：将经纪人按照总交易量降序排列，以便展示业绩最好的经纪人。

以下是 Broker-Volume 事务的 SQL 伪代码：

`-- Broker-Volume 事务的 SQL 查询   SELECT b_name, SUM(tr_qty * tr_bid_price) -- 经纪人的总交易量   FROM trade_request, sector, industry, company, broker, security   WHERE tr_b_id = b_id -- 经纪人表，通过经纪人ID关联    AND tr_s_symb = s_symb  -- 行业表，通过证券符号关联    AND s_co_id = co_id -- 行业表，通过行业ID关联    AND co_in_id = in_id -- 确保公司表中的国家ID与行业表中的国家ID匹配    AND sc_id = in_sc_id -- 确保行业表中的公司ID与公司表中的ID匹配    AND b_name IN (%s..) -- 经纪人名称列表，这里 %s.. 是一个占位符，表示一系列经纪人名称    AND sc_name = '%s' -- 行业名称，这里 '%s' 是一个占位符，表示特定的行业名称   GROUP BY b_name   ORDER BY 2 DESC -- 按总交易量降序排列   `

在这个查询中，我们使用了多个 JOIN 操作来关联不同的表，确保我们能够获取每个经纪人的交易请求信息。我们通过 WHERE 子句过滤出特定经纪人和特定行业的交易请求。然后，我们使用 GROUP BY 对经纪人名称进行分组，并计算每个经纪人的总交易量。最后，我们使用 ORDER BY 对结果进行降序排列，以便展示交易量最高的经纪人。

## Customer-Position

客户位置（Customer-Position）由`EGenDriverCE`调用。它由三个 frame 组成（frame 2和3是相互排斥的）。客户由客户ID（`customer ID`）或客户税号（`customer tax ID`）指定。如果转入交易的 `customer ID` 为0，则使用客户税ID来查找客户ID。检索有关客户个人资料的详细信息。此外，对于每个客户的账户，将退还该账户的现金余额和账户中所有持有的当前市场总值。如果请求交易活动的历史记录，则检索客户帐户中随机选择的帐户的最新十笔交易的信息。

**「事务逻辑概述」**

Customer-Position 事务模拟了客户查询其账户持仓情况的场景。这个事务通过检索客户资料、账户余额、持仓详情以及最近的交易历史，为客户提供了一个全面的账户状态报告。在技术博客中，我们将详细探讨这个事务的每个阶段，以及它们在 SQL 中的具体实现。

**「Frame/sql 注解」**

在 Frame 1 中，我们首先设置了事务的隔离级别为 READ COMMITTED，这确保了事务在读取数据时的一致性。接着，我们执行了两个 SQL 查询来获取客户信息。

`-- 设置事务隔离级别   SET TRANSACTION ISOLATION LEVEL READ COMMITTED;      -- 根据税号查询客户ID   SELECT c_id FROM customer WHERE c_tax_id = _latin1'970AM8516RE955';      -- 获取客户详细信息   SELECT        c_st_id, c_l_name, c_f_name, c_m_name, c_gndr, c_tier,        DATE_FORMAT(c_dob,'%Y-%m-%d'), c_ad_id,        c_ctry_1, c_area_1, c_local_1, c_ext_1,        c_ctry_2, c_area_2, c_local_2, c_ext_2,        c_ctry_3, c_area_3, c_local_3, c_ext_3,        c_email_1, c_email_2    FROM customer    WHERE c_id = 4300001491;   `

Frame 2 仅在 `get_history` 参数为 TRUE 时执行。这个 Frame 负责检索客户最近的交易历史。这里我们使用了两个 SQL 查询：

`-- 查询客户账户的前10个持仓及其总价值   SELECT        ca_id, ca_bal, COALESCE(SUM(hs_qty * lt_price),0) AS price_sum    FROM        customer_account        LEFT OUTER JOIN holding_summary ON hs_ca_id = ca_id, last_trade    WHERE        ca_c_id = 4300001491 AND lt_s_symb = hs_symb    GROUP BY        ca_id, ca_bal    ORDER BY        price_sum ASC    LIMIT 10;      -- 查询客户最近的30条交易历史记录   SELECT        t_id, t_s_symb, t_qty, st_name, DATE_FORMAT(th_dts,'%Y-%m-%d %H:%i:%s.%f')    FROM        (SELECT t_id AS id FROM trade WHERE t_ca_id = 43000014904 ORDER BY t_dts DESC LIMIT 10) AS t,        trade, trade_history, status_type    FORCE INDEX(PRIMARY)    WHERE        t_id = id AND th_t_id = t_id AND st_id = th_st_id    ORDER BY        th_dts DESC    LIMIT 30;   `

Frame 3 包含了一个 `COMMIT` 语句，用于提交事务，确保之前的所有更改都被保存到数据库中。

`-- 提交事务   COMMIT;   `

## Market-Feed

**「事务逻辑概述」**

Market-Feed 事务在 TPC-E 基准测试中扮演着模拟市场数据更新的角色。这个事务的目的是处理市场交易所的最新交易信息，这些信息通常包括股票的最后成交价格、成交量和成交时间。包含 1 个 frame

**「Frame/sql 注解」** 设置事务隔离级别

`-- 设置事务隔离级别为可重复读，确保在事务期间读取的数据保持一致   SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;   `

更新最后成交信息

`-- 更新 last_trade 表，模拟市场交易所的最新成交信息   UPDATE last_trade    SET        lt_price = '2.93399999999999999e+01', -- 设置最新的成交价格       lt_vol = lt_vol + '100', -- 增加成交量       lt_dts = '2024-02-27 20:48:17' -- 更新成交时间   WHERE        lt_s_symb = 'CLYS'; -- 指定特定的证券符号   `

查询待处理的交易请求

`-- 查询 trade_request 表，找出与最新成交信息相关的待处理交易请求   SELECT        tr_t_id, tr_bid_price, tr_tt_id, tr_qty   FROM        trade_request   WHERE        tr_s_symb = 'CLYS' -- 指定证券符号       AND (           (tr_tt_id = 'TSL' AND tr_bid_price >= '2.93399999999999999e+01') -- 买入限价单，且报价大于等于最新成交价           OR (tr_tt_id = 'TLS' AND tr_bid_price <= '2.93399999999999999e+01') -- 卖出限价单，且报价小于等于最新成交价           OR (tr_tt_id = 'TLB' AND tr_bid_price >= '2.93399999999999999e+01') -- 买入止损单，且报价大于等于最新成交价       );   `

提交事务

`-- 提交事务，确保所有更改都被保存   COMMIT;   `

## Market-Watch

Market-Watch 事务是由客户执行的，用于监控市场的整体表现。这个事务通过比较选定证券集合在\*\*「特定日期的收盘价」**与**「当前市场价格」\*\*的百分比变化来实现。这个集合可能基于客户的当前持仓、潜在证券观察列表或特定行业。Market-Watch 事务包含 1 个 Frame，该 Frame 执行一个 SQL 查询来计算市值变化。

`-- 设置事务隔离级别为 READ COMMITTED，确保事务在读取数据时不会受到其他并发事务的影响   SET TRANSACTION ISOLATION LEVEL READ COMMITTED;      -- 执行查询，计算市值变化   -- 这个查询涉及到多个表的连接，包括 watch_item, watch_list, last_trade, security, 和 daily_market   -- 它计算了在特定日期（'2004-12-31'）的收盘价（dm_close）和当前价格（lt_price）的总和   -- 通过比较这两个总和，可以得到市值的百分比变化   SELECT        COALESCE(SUM(s_num_out * dm_close), 0) AS market_cap_change, -- 计算特定日期的市值       COALESCE(SUM(s_num_out * lt_price), 0) AS current_market_cap -- 计算当前市值   FROM        watch_item,        watch_list,        last_trade,        security,        daily_market   WHERE        wl_c_id = '4300000678' -- 指定客户ID       AND wi_wl_id = wl_id -- 确保 watch_item 和 watch_list 的关联ID匹配       AND dm_s_symb = wi_symb -- 确保证券符号匹配       AND dm_date = '2004-12-31' -- 指定比较的日期       AND lt_s_symb = dm_s_symb -- 确保 last_trade 中的证券符号与 daily_market 中的匹配       AND s_symb = dm_s_sym; -- 确保 security 表中的证券符号与 daily_market 中的匹配      -- 关闭语句，结束查询   Close stmt;   `

## Security-Detail

**「事务逻辑概述」**

Security-Detail 事务旨在模拟客户在决定是否执行交易前对特定证券进行详细研究的过程。这个事务由 EGenDriverCE 触发，并且只包含 **「1个 Frame」**。事务会返回关于给定证券的详细信息，包括公司信息、竞争对手列表、当前和历史财务数据，以及关于公司的最新新闻条目。

**「Frame/sql 注解」**

`-- 设置事务隔离级别为 READ COMMITTED，确保事务在读取数据时的一致性   SET TRANSACTION ISOLATION LEVEL READ COMMITTED;      -- 查询证券和公司详细信息   SELECT        s_name, co_id, co_name, co_sp_rate, co_ceo, co_desc,        DATE_FORMAT(co_open_date,'%Y-%m-%d'), co_st_id,        ca.ad_line1, ca.ad_line2, zca.zc_town, zca.zc_div, ca.ad_zc_code, ca.ad_ctry,        s_num_out, DATE_FORMAT(s_start_date,'%Y-%m-%d'),        DATE_FORMAT(s_exch_date,'%Y-%m-%d'), s_pe, s_52wk_high,        DATE_FORMAT(s_52wk_high_date,'%Y-%m-%d'), s_52wk_low,        DATE_FORMAT(s_52wk_low_date,'%Y-%m-%d'), s_dividend, s_yield,        zea.zc_div, ea.ad_ctry, ea.ad_line1, ea.ad_line2, zea.zc_town,        ea.ad_zc_code, ex_close, ex_desc, ex_name, ex_num_symb, ex_open    FROM        security, company, address ca, address ea, zip_code zca, zip_code zea, exchange    WHERE        s_symb = _latin1'XTRM'        AND co_id = s_co_id        AND ca.ad_id = co_ad_id        AND ea.ad_id = ex_ad_id        AND ex_id = s_ex_id        AND ca.ad_zc_code = zca.zc_code        AND ea.ad_zc_code = zea.zc_code;      -- 查询公司竞争对手信息   SELECT        co_name, in_name    FROM        company_competitor, company, industry    WHERE        cp_co_id = 4300000566        AND co_id = cp_comp_co_id        AND in_id = cp_in_id    LIMIT 3;      -- 查询公司财务数据   SELECT        fi_year, fi_qtr, DATE_FORMAT(fi_qtr_start_date,'%Y-%m-%d'),        fi_revenue, fi_net_earn, fi_basic_eps, fi_dilut_eps,        fi_margin, fi_inventory, fi_assets, fi_liability,        fi_out_basic, fi_out_dilut    FROM        financial    WHERE        fi_co_id = 4300000566    ORDER BY        fi_year ASC, fi_qtr    LIMIT 20;      -- 查询证券市场历史数据   SELECT        DATE_FORMAT(dm_date,'%Y-%m-%d'), dm_close, dm_high, dm_low, dm_vol    FROM        daily_market    WHERE        dm_s_symb = _latin1'XTRM'        AND dm_date >= _latin1'2000-08-12'    ORDER BY        dm_date ASC    LIMIT 15;      -- 查询最后一笔交易信息   SELECT        lt_price, lt_open_price, lt_vol    FROM        last_trade    WHERE        lt_s_symb = _latin1'XTRM';      -- 查询公司最新新闻条目   SELECT        DATE_FORMAT(ni_dts, '%Y-%m-%d %H:%i:%s.%f'), ni_source, ni_author, ni_headline, ni_summary    FROM        news_xref, news_item    WHERE        ni_id = nx_ni_id        AND nx_co_id = 4300000566    LIMIT 2;      -- 提交事务，确保所有查询结果被正确处理   COMMIT;   `

## Trade-Lookup

Trade-Lookup包含 4 个 frame，实际上包含了多个数据库意义上的“事务”，broker 和customer 分别执行两个 frame，这些甚至不在一个进程中执行完毕。因此不在通过 sql 解释，而是概述其设计逻辑。

Trade-Lookup 事务是 TPC-E 基准测试中的一个\*\*「关键组成部分」**，它模拟了**「客户」**或**「经纪人」\*\*为了回答关于一组交易的问题而进行的信息检索过程。这个事务涵盖了多种场景，包括进行`市场分析`、`回顾账户最近的交易记录`、`分析特定证券的过去表现`以及`分析特定客户持仓的历史`。

Trade-Lookup 事务由 EGenDriverCE 触发，并且包含四个互斥的 Frame。每个 Frame 都采用不同的技术来查找历史交易数据。

1. **「Frame 1」** ：Frame 1 接受一组交易 ID 的列表。对于列表中的每个交易 ID，系统会返回相关的交易信息。这允许用户查询特定的交易详情，可能是为了验证交易记录或进行详细的交易分析。

1. **「Frame 2」** ：Frame 2 接受客户账户 ID、开始时间戳、结束时间戳以及交易数量（N）作为输入。它会返回在指定时间范围内（包括开始和结束时间戳）的前 N 笔交易信息。这个 Frame 适用于用户想要了解特定账户在一定时间窗口内的交易活动。

1. **「Frame 3」** ：Frame 3 接受证券符号、开始时间戳、结束时间戳以及交易数量（N）作为输入。它会返回在指定时间范围内（包括开始和结束时间戳）的前 N 笔特定证券的交易信息。这个 Frame 用于分析特定证券的市场表现和交易活动。

1. **「Frame 4」** ：Frame 4 接受客户账户 ID 和一个时间戳作为输入。它会识别出在指定时间戳或之后该客户账户的第一笔交易，并返回最多 20 条与这笔交易 ID 相关的持仓历史变更记录。这些历史变更记录包括由这笔交易对之前交易创建的持仓所做的更改，以及后续交易对由此交易创建的任何持仓所做的更改。

部分 sql：

`    -- 3.3.6 Trade-Lookup   Query        SET TRANSACTION ISOLATION LEVEL READ COMMITTED   -- F1   Execute        SELECT t_bid_price, t_exec_name, t_is_cash, tt_is_mrkt, t_trade_price FROM trade, trade_type WHERE t_id = '200000005238470' AND t_tt_id = tt_id   Execute        SELECT se_amt, DATE_FORMAT(se_cash_due_date, '%Y-%m-%d'), se_cash_type FROM settlement WHERE se_t_id = '200000005238470'   Execute        SELECT ct_amt, DATE_FORMAT(ct_dts, '%Y-%m-%d %H:%i:%s.%f'), ct_name FROM cash_transaction WHERE ct_t_id = '200000005238470'   Execute        SELECT DATE_FORMAT(th_dts, '%Y-%m-%d %H:%i:%s.%f'), th_st_id FROM trade_history WHERE th_t_id = '200000005238470' ORDER BY th_dts LIMIT 3      -- F2   Query        SELECT t_bid_price, t_exec_name, t_is_cash, tt_is_mrkt, t_trade_price FROM trade, trade_type WHERE t_id = 200000005236617 AND t_tt_id = tt_id   Query        SELECT se_amt, DATE_FORMAT(se_cash_due_date, '%Y-%m-%d'), se_cash_type FROM settlement WHERE se_t_id = 200000005236617   Query        SELECT ct_amt, DATE_FORMAT(ct_dts, '%Y-%m-%d %H:%i:%s.%f'), ct_name FROM cash_transaction WHERE ct_t_id = 200000005236617   Query        SELECT DATE_FORMAT(th_dts, '%Y-%m-%d %H:%i:%s.%f'), th_st_id FROM trade_history WHERE th_t_id = 200000005236617 ORDER BY th_dts LIMIT 3   Query        COMMIT      -- F3   -- F4    `

## Trade-Order

> ❝
>
> 代码中，第 5、6 步是 rollback或 commit，其余四个步骤请参考 `TOF1_1 ~ TOF4_2`
>
> ❞

Trade-Order 事务由 EGenDriverCE 执行，它包含\*\*「六个 Frame」\*\*，是非常巨大的事务。这个事务模拟了客户、经纪人或授权第三方买卖证券的过程，包括验证交易执行者的授权、估算交易的财务影响以及提交或取消交易。

1. **「获取客户信息」**：事务首先使用传入的账户 ID 获取客户、客户账户和账户经纪人的信息。这是为了确保后续操作能够在正确的账户上下文中进行。

1. **「验证执行者」**：接下来，事务会验证执行交易的人是否具有适当的授权。如果执行者未获授权，事务将回滚。在基准测试执行期间，CE 总是生成授权的执行者。

1. **「估算交易影响」**：事务的下一步是估算执行交易的总体财务影响。对于限价单，使用请求的价格进行估算；对于市价单，使用当前市场价值。估算过程包括评估交易对现有持仓的影响，计算可能实现的利润的资本收益税，以及计算行政费用和经纪人佣金。如果是保证金交易，还会评估客户账户的总资产。

1. **「记录订单」**：使用上述信息记录订单。这一步骤确保了交易的详细信息被正确地保存在系统中，以便后续处理。

1. **「提交或回滚」**：在完成所有处理后，事务会根据一定的比例选择提交或回滚。这模拟了实际交易中可能出现的取消订单或错误条件。所有其他事务则被提交。

1. **「发送交易到 MEE」**：对于成功提交的市价订单，EGenTxnHarness 会将交易发送到适当的 MEE。这是模拟交易流程的最后一步，确保交易能够被市场交易所处理。

## Trade-Result

Trade-Result 事务由 EGenDriverMEE 执行，它包含\*\*「六个 Frame」\*\*。这个事务模拟了完成股票市场交易的过程，即经纪行从市场交易所接收到交易的最终确认和价格。客户的持仓将根据交易的完成情况进行更新，同时生成的估计数据（如经纪人佣金等）将被实际数值替换，并记录交易的历史信息以供后续参考。

1. **「获取交易信息」**：事务的第一步是使用传入的交易 ID 获取交易的相关信息。这包括客户的账户 ID，用于进一步查询账户信息。

1. **「更新客户持仓」**：接下来，根据交易的类型（买入或卖出）、涉及的股票数量以及客户当前的持仓情况（多头或空头），更新客户的持仓。这可能涉及清算现有持仓以覆盖销售，或者在购买股票时使用现有空头持仓。

1. **「计算税款」**：如果交易实现利润且利润需要缴税，将计算应缴税款。

1. **「计算经纪人佣金」**：计算经纪人的佣金，并将所有与交易相关的信息记录下来。

1. **「提交交易记录」**：最后，为交易创建结算记录，并在交易不是保证金交易的情况下更新客户的账户余额。

这个事务的设计确保了交易完成后所有必要的更新和记录都能被正确处理，反映了实际金融系统中交易结算的复杂性。在基准测试中，它有助于评估系统在处理交易结果时的性能和准确性。

下面 sql 是一个例子，但这个例子只走了一部分分支，例如 F2、F3 有一些就没有走到。

`    SET TRANSACTION ISOLATION LEVEL SERIALIZABLE   -- F1   SELECT t_ca_id, t_tt_id, t_s_symb, t_qty, t_chrg, t_lifo, t_is_cash FROM trade WHERE t_id = 200000014584794   SELECT tt_name, tt_is_sell, tt_is_mrkt FROM trade_type WHERE tt_id = _latin1'TMS'   SELECT hs_qty FROM holding_summary WHERE hs_ca_id = 43000012441 AND hs_s_symb = _latin1'BDGPRB'   -- F2   SELECT ca_b_id, ca_c_id, ca_tax_st FROM customer_account WHERE ca_id = 43000012441   -- TRF2_2a INSERT, or TRF2_2b UPDATE   UPDATE holding_summary SET hs_qty = 8700 WHERE hs_ca_id = 43000012441 AND hs_s_symb = _latin1'BDGPRB'   -- MEESUT_STMT_TRF2_3a ASC, or TRF2_3b DESC   SELECT h_t_id, h_qty, h_price FROM holding WHERE h_ca_id = '43000012441' AND h_s_symb = 'BDGPRB' ORDER BY h_dts ASC   -- TRF2_4   INSERT INTO holding_history(hh_h_t_id, hh_t_id, hh_before_qty, hh_after_qty) VALUES('200000013784914', '200000014584794', '700', '600')   -- TRF2_5a or TRF2_5b (DELETE)   UPDATE holding SET h_qty = '600' WHERE h_t_id = '200000013784914'   -- TRF3_1 在这个事务中 miss，如果要寻找，其他事务中可以搜到   -- SELECT sum(tx_rate) FROM taxrate, customer_taxrate WHERE tx_id = cx_tx_id AND cx_c_id = ?   -- TRF4_1   SELECT s_ex_id, s_name FROM security WHERE s_symb = _latin1'BDGPRB'   -- TRF4_2   SELECT c_tier FROM customer WHERE c_id = 4300001245   -- TRF4_3   SELECT cr_rate FROM commission_rate WHERE cr_c_tier = 1 AND cr_tt_id = _latin1'TMS' AND cr_ex_id = _latin1'NASDAQ' AND cr_from_qty <= 100 AND cr_to_qty >= 100   -- TRF5_1   UPDATE trade SET t_comm = 1.14299999999999997e+01, t_dts = _latin1'2024-02-27 20:48:15.000000', t_st_id = _latin1'CMPT', t_trade_price = 2.85799999999999983e+01 WHERE t_id = 200000014584794   -- TRF5_2   INSERT INTO trade_history(th_t_id, th_dts, th_st_id) VALUES(200000014584794, _latin1'2024-02-27 20:48:15.000000', _latin1'CMPT')   -- TRF5_3   UPDATE broker SET b_comm_total = b_comm_total + 1.14299999999999997e+01, b_num_trades = b_num_trades + 1 WHERE b_id = 4300000017   -- TRF6_1   INSERT INTO settlement(se_t_id, se_cash_type, se_cash_due_date, se_amt) VALUES(200000014584794, _latin1'Cash Account', _latin1'2024-02-29', 2.84157000000000016e+03)   -- TRF6_2   UPDATE customer_account SET ca_bal = ca_bal + 2.84157000000000016e+03 WHERE ca_id = 43000012441   -- TRF6_3   INSERT INTO cash_transaction(ct_dts, ct_t_id, ct_amt, ct_name) VALUES(_latin1'2024-02-27 20:48:15.000000', 200000014584794, 2.84157000000000016e+03, _latin1'Market-Sell 100 shared of PREF_B of Bandag, Inc.')   -- TRF6_4   SELECT ca_bal FROM customer_account WHERE ca_id = 43000012441   COMMIT    `

## Trade-Status

Trade-Status 事务由 EGenDriverCE 执行，它包含一个 Frame。这个事务模拟了客户查看其账户最近交易活动摘要的过程，通常是为了回顾最近的交易记录。

1. **「Frame 1」**：这个 Frame 负责检索给定账户 ID 的最近 50 笔交易的状态信息。这包括交易 ID、交易时间、状态名称、交易类型名称、证券符号、交易数量、执行交易的人员名称、交易费用、证券名称以及交易所名称。

`-- 设置事务隔离级别为 READ COMMITTED，确保事务在读取数据时的一致性   SET TRANSACTION ISOLATION LEVEL READ COMMITTED;      -- 执行查询，获取最近 50 笔交易的状态信息   SELECT        t_id, DATE_FORMAT(t_dts,'%Y-%m-%d %H:%i:%s.%f'), st_name, tt_name, t_s_symb, t_qty, t_exec_name, t_chrg, s_name, ex_name    FROM        trade, status_type, trade_type, security, exchange    WHERE        t_ca_id = '43000003162' -- 指定客户账户 ID       AND st_id = t_st_id -- 确保交易状态与交易 ID 匹配       AND tt_id = t_tt_id -- 确保交易类型与交易 ID 匹配       AND s_symb = t_s_symb -- 确保证券符号与交易 ID 匹配       AND ex_id = s_ex_id -- 确保交易所与证券符号匹配   ORDER BY        t_dts DESC -- 按交易时间降序排列   LIMIT 50; -- 限制结果为最近的 50 笔交易      -- 关闭语句   Close stmt;      -- 执行查询，获取客户、账户和经纪人的详细信息   SELECT        c_l_name, c_f_name, b_name    FROM        customer_account, customer, broker    WHERE        ca_id = '43000003162' -- 指定客户账户 ID       AND c_id = ca_c_id -- 确保客户账户与客户 ID 匹配       AND b_id = ca_b_id; -- 确保经纪人 ID 与客户账户匹配      -- 关闭语句   Close stmt;      -- 提交事务，确保所有查询结果被正确处理   Query COMMIT;   `

在这个事务中，首先设置了事务的隔离级别，然后执行了两个查询。第一个查询用于获取交易状态信息，第二个查询用于获取与交易相关的客户、账户和经纪人的详细信息。

## Trade-Update

Trade-Update 事务由 EGenDriverCE 执行，它包含\*\*「三个互斥的 Frame」\*\*。每个 Frame 使用不同的技术来查找和更新历史交易数据。

**「Frame 1」**

- 接受一组交易 ID 的列表。

- 返回列表中每个交易的信息。

- 对于每个交易，修改执行者的名称。

`-- 查询特定交易 ID 的执行者名字   SELECT t_exec_name FROM trade WHERE t_id = '200000001949399';      -- 更新执行者名字   UPDATE trade SET t_exec_name = 'Jessica X Lowery' WHERE t_id = '200000001949399';      -- 查询交易相关信息   SELECT t_bid_price, t_exec_name, t_is_cash, tt_is_mrkt, t_trade_price FROM trade, trade_type WHERE t_id = '200000001949399' AND t_tt_id = tt_id;      -- 查询结算信息   SELECT se_amt, DATE_FORMAT(se_cash_due_date, '%Y-%m-%d'), se_cash_type FROM settlement WHERE se_t_id = '200000001949399';      -- 查询现金交易信息   SELECT ct_amt, DATE_FORMAT(ct_dts, '%Y-%m-%d %H:%i:%s.%f'), ct_name FROM cash_transaction WHERE ct_t_id = '200000001949399';      -- 查询交易历史记录   SELECT DATE_FORMAT(th_dts, '%Y-%m-%d %H:%i:%s.%f'), th_st_id FROM trade_history WHERE th_t_id = '200000001949399' ORDER BY th_dts LIMIT 3;      -- 查询另一个交易 ID 的执行者名字   SELECT t_exec_name FROM trade WHERE t_id = 200000000135883;      -- 更新执行者名字   UPDATE trade SET t_exec_name = _latin1'Roxann Kniffen' WHERE t_id = 200000000135883;      -- 提交事务   COMMIT;   `

**「Frame 2」**

- 接受客户账户 ID、开始时间戳、结束时间戳和交易数量（N）作为输入。

- 返回指定客户账户在指定时间范围内的前 N 笔交易信息。

- 修改每笔交易的结算现金类型。

`-- 设置事务隔离级别为可重复读   SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;      -- 查询特定客户账户 ID 在指定时间范围内的交易信息   SELECT t_bid_price, t_exec_name, t_is_cash, t_id, t_trade_price FROM trade WHERE t_ca_id = 43000008818 AND t_dts >= _latin1'2005-01-27 13:24:52.109000' AND t_dts <= _latin1'2005-03-14 09:15:00.000000' ORDER BY t_dts ASC LIMIT 20;      -- 对于每笔交易，更新结算类型为 'Cash'   -- 下面的语句会重复多组   SELECT se_cash_type FROM settlement WHERE se_t_id = 200000002704863;   UPDATE settlement SET se_cash_type = 'Cash' WHERE se_t_id = 200000002704863;      -- 查询并返回与特定交易 ID 相关的结算信息   SELECT se_amt, DATE_FORMAT(se_cash_due_date, '%Y-%m-%d'), se_cash_type FROM settlement WHERE se_t_id = 200000002704863;      -- 查询并返回与特定交易 ID 相关的现金交易信息   SELECT ct_amt, DATE_FORMAT(ct_dts, '%Y-%m-%d %H:%i:%s.%f'), ct_name FROM cash_transaction WHERE ct_t_id = 200000002704863;      -- 查询并返回与特定交易 ID 相关的交易历史记录   SELECT DATE_FORMAT(th_dts, '%Y-%m-%d %H:%i:%s.%f'), th_st_id FROM trade_history WHERE th_t_id = 200000002704863 ORDER BY th_dts LIMIT 3;      -- 提交事务   Query COMMIT;   `

**「Frame 3」**

- 接受证券符号、开始时间戳、结束时间戳和交易数量（N）作为输入。

- 返回给定证券在指定时间范围内的前 N 笔交易信息。

- 对于现金交易，修改交易描述。

`-- 查询特定证券符号在指定时间范围内的交易信息   SELECT t_ca_id, t_exec_name, t_is_cash, t_trade_price, t_qty, s_name, DATE_FORMAT(t_dts, '%Y-%m-%d %H:%i:%s.%f'), t_id, t_tt_id, tt_name FROM trade, trade_type FORCE INDEX(PRIMARY), security WHERE t_s_symb = _latin1'AMGN' AND t_dts >= _latin1'2005-02-09 16:05:31.891000' AND t_dts <= _latin1'2005-03-14 09:15:00.000000' AND tt_id = t_tt_id AND s_symb = t_symb ORDER BY t_dts ASC LIMIT 20;      -- 对于每笔现金交易，更新交易描述   -- 下面的语句会重复多组   SELECT se_amt, DATE_FORMAT(se_cash_due_date, '%Y-%m-%d'), se_cash_type FROM settlement WHERE se_t_id = 200000004055564;   SELECT ct_name FROM cash_transaction WHERE ct_t_id = 200000004055564;   UPDATE cash_transaction SET ct_name = _latin1'Limit-Sell 400 Shares of COMMON of Amgen, Inc.' WHERE ct_t_id = 200000004055564;      -- 查询并返回与特定交易 ID 相关的结算信息   SELECT ct_amt, DATE_FORMAT(ct_dts, '%Y-%m-%d %H:%i:%s.%f'), ct_name FROM cash_transaction WHERE ct_t_id = 200000004055564;      -- 查询并返回与特定交易 ID 相关的交易历史记录   SELECT DATE_FORMAT(th_dts, '%Y-%m-%d %H:%i:%s.%f'), th_st_id FROM trade_history WHERE th_t_id = 200000004055564 ORDER BY th_dts ASC LIMIT 3;   `

## Data-Maintenance

Data-Maintenance 只有一个 frame，但是这个 frame 非常复杂。可能是由于 time triggered，因此 tpce_50k_sorted_id_time.csv 中并未出现。Data-Maintenance 事务由 EGenDriverDM 执行，它包含一个 Frame。这个事务模拟了对数据库中主要用作参考的静态数据进行定期修改的过程。

**「Frame 1」**

- 这个 Frame 负责执行数据维护操作，这些操作包括更新账户权限、地址信息、公司信用评级、客户电子邮件地址、客户税率、市场数据、交易所描述、财务数据、新闻项、证券交易日期、税率以及观察列表中的证券符号。

- 每次运行这个事务时，EGenTxnHarness 会提供要修改的表的名称作为输入。

- 事务会根据提供的表名选择下一个要修改的表，这意味着每个表大约每十二分钟只会被修改一次。

- 对于每个表，事务会执行特定的更新操作，例如更改信用评级、电子邮件地址、税率等，以保持数据的时效性和准确性。

## Trade-Cleanup

Trade-Cleanup 事务由 EGenDriverDM 执行，它包含一个 Frame。这个事务的目的是清理数据库中的挂起或已提交的交易，以便在测试运行之前将数据库恢复到已知状态。

仅在测试开始时执行一次。

**「Frame 1」**

- 设置事务隔离级别为 READ COMMITTED，确保事务在读取数据时的一致性。

- 查询 `trade_request` 表，获取所有待处理交易的交易 ID。

- 对于每个待处理的交易，执行以下步骤：

- 在 `trade_history` 表中插入一条新记录，表示交易已被提交（`SBMT` 表示提交）。

- 更新 `trade` 表，将交易状态设置为已取消（`CNCL`），并记录当前的日期和时间。

- 再次在 `trade_history` 表中插入一条新记录，记录交易的取消状态。

这个过程确保了所有未完成的交易都被正确地标记和记录，以便在测试运行开始时数据库处于一个干净的状态。

`-- 设置事务隔离级别为 READ COMMITTED   SET TRANSACTION ISOLATION LEVEL READ COMMITTED;      -- 选择 trade_request 表中的所有交易 ID 并排序   SELECT tr_t_id FROM trade_request ORDER BY tr_t_id;      -- 为每个交易 ID 插入一条记录到 trade_history 表，表示交易已提交   INSERT INTO trade_history (th_t_id, th_dts, th_st_id) VALUES ('200000014582105', '2024-02-27 20:48:13', 'SBMT');      -- 重复多次，为每个交易 ID 更新 trade 表，设置状态为已取消，并记录时间   UPDATE trade SET t_st_id = 'CNCL', t_dts = '2024-02-27 20:48:13' WHERE t_id = '200000014582105';      -- 为已取消的交易插入一条记录到 trade_history 表   INSERT INTO trade_history (th_t_id, th_dts, th_st_id) VALUES (200000014582105, _latin1'2024-02-27 20:48:13', _latin1'CNCL');      -- 如果有其他交易 ID，也执行相同的插入和更新操作   -- 例如：   INSERT INTO trade_history (th_t_id, th_dts, th_st_id) VALUES (200000014582119, _latin1'2024-02-27 20:48:13', _latin1'SBMT');   `

# 轻松上手：TPCE for MySQL

TPCE 起初只有面向PostgreSQL 的版本，Percona 公司贡献了针对 MySQL的版本：https://github.com/Percona-Lab/tpce-mysql。

这个版本仍然存在编译问题，建议通过下面的改版来安装 tpce-mysql：https://github.com/VincentS/tpcemysql

下面是 Debian 系统的安装过程。首先安装 tpcemysql 的依赖项：

`       # 安装 unixodbc   sudo apt-get install unixodbc unixodbc-dev      # 安装 mysql8 的驱动：   wget https://downloads.mysql.com/archives/get/p/10/file/mysql-connector-odbc_8.0.20-1debian9_amd64.deb      sudo dpkg -i mysql-connector-odbc_8.0.20-1debian9_amd64.deb   sudo apt-get install -f    `

tpcemysql 需要通过 odbc 连接 mysql，因此配置 odbc ：

`    # 设置 odbc 环境变量 /etc/odbcinst.ini   # 若[MySQL ODBC 8.0 Driver]已经存在，则需要先删除，避免重复   cat /etc/odbcinst.ini   [MySQL ODBC 8.0 Driver]   Description=MySQL ODBC 8.0 Driver   Driver=/usr/lib/x86_64-linux-gnu/odbc/libmyodbc8w.so   Setup=/usr/lib/x86_64-linux-gnu/odbc/libmyodbc8w.so      # 设置 odbc 连接信息   cat /etc/odbc.ini   [MySQLServer_ODBC_NAME]   Description=My MySQL tpce   Driver=MySQL ODBC 8.0 Driver   Server=xxx.xxx.xxx.xxx   Port=3308   User=root   Password=password   Database=tpce   Option=3    `

接下来编译 tpce-mysql：

`git clone git@github.com:VincentS/tpcemysql.git      cd tpce_mysql      mkdir flat_out   cd prj   make clean      # 修改 makefile   # 将 CCFLAGS=-g -O2 -Wall  -D__STDC_CONSTANT_MACROS -D__STDC_FORMAT_MACROS -DHANA_ODBC -DUSE_PREPARE 中的 -DHANA_ODBC 修改为 -DMYSQL_ODBC      cp Makefile.Mysql Makefile      make   `

生成 + 导入 tpce 数据。

`cd ~/tpcemysql      # 生成数据，生成后，数据会写入 flat_out，等待 LOAD DATA INFILE   ./bin/EGenLoader -i flat_in -o flat_out -c 2000 -t 2000 -f 200 -w 50      cd scripts/mysql/      # 首先在 mysql 中创建一个空库 tpce      # 步骤 1：建表   mysql --local-infile=1 -h 127.0.0.1 -uroot -ppassword -P 3308 -Dtpce < 1_create_table.sql      # 导入数据等等后续操作与步骤 1 类似   mysql --local-infile=1 -h 127.0.0.1 -uroot -ppassword -P 3308 -Dtpce < 2_load_data.sql   mysql --local-infile=1 -h 127.0.0.1 -uroot -ppassword -P 3308 -Dtpce < 3_create_index.sql   mysql --local-infile=1 -h 127.0.0.1 -uroot -ppassword -P 3308 -Dtpce < 4_create_fk.sql   mysql --local-infile=1 -h 127.0.0.1 -uroot -ppassword -P 3308 -Dtpce < 5_create_sequence.sql   `

运行：

`cd ~/tpcemysql      ./bin/EGenSimpleTest -c 2000 -a 2000 -f 200 -d 50 -l 200 -e flat_in -j tpce  -U root -P password -r 10 -u 10 -t 90 -D MySQLServer_ODBC_NAME   `

# 实践挑战：给 TPCE 推荐索引

为 MySQL 推荐索引是很常见的优化手段。对于 OLAP 或 OLTP 业务场景都有重要意义。其中，OLAP 业务的难点在于对复杂 join 关系、复杂操作子（子函数、GROUP BY、单值或范围查询）的理解，而 OLTP 业务的难点在于【慢 SQL + 基础 SQL】的综合理解。

TPCC 和 TPCE benchmark 自身提供了较为合理的普通索引、唯一键索引（UK）和外键索引（FK），我们将 benchmark 标准索引组合成为 GT（Ground Truth），这是索引推荐算法致力于达到的目标。我们对比了流行的友商开源算法Soar和字节跳动自研算法的SQLBrain的推荐效果。

下面的测试结果展现了 TPC-E 的意义：**「TPC-E 显然是难度更大、挑战性更高的基准测试。」** 由于 TPC-C 过于简单，Soar 和 SQLBrain 算法都可以达到不错的效果（超过 GT 性能的 95%），**「测试不出差距」**。但是 TPC-E 上两种方法拉开了差距。Soar 推荐的索引仅能达到 14.4 tpsE（GT 性能的16% 左右），而 SQLBrain 仍可以达到 GT 性能的 95% 以上。

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

为字节跳动ByteBrain团队自研的MySQL索引推荐系统 **「SQLBrain」** 打个广告：**「SQLBrain」** 在 TPC-E 的推荐效果达到 Ground Truth 的\*\*「98%」\*\*（对比流行的开源工具 Soar 推荐效果仅达到 **「16%」**），已经在字节跳动的业务中接入了近x万个MySQL实例，覆盖电商、财经、国际支付、直播、广告等多种业务。相关技术正在准备开源，敬请期待。🌷

# 总结

TPC-E 可以被视为 TPC-C 的强化升级版，引入了更复杂的事务、更复杂的关系表和执行逻辑，增大了 OLTP Benchmark 的挑战性。在 TPC-C 过于简单、已经被充分优化的今天，TPC-E 作为 一种更复杂的 OLTP Benchmark，可以在索引推荐、性能调参等领域展现作用、挖掘各种算法技术的能力瓶颈。

# 参考文献

3. TPC-E 官网: https://www.tpc.org/tpce/

1. TPC-E pdf 规范：https://www.tpc.org/TPC_Documents_Current_Versions/pdf/TPC-E_v1.14.0.pdf

1. Chen, Shimin, et al. "TPC-E vs. TPC-C: Characterizing the new TPC-E benchmark via an I/O comparison study." ACM Sigmod Record 39.3 (2011): 5-10.

1. Tözün, Pınar, et al. "From A to E: analyzing TPC's OLTP benchmarks: the obsolete, the ubiquitous, the unexplored." EDBT. 2013.

Reads 2454

​

Comment

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/5EcwYhllQOhkoWTP1gVm0Lqs480XOARyoSYjPEsRVCSF35cbWIp6cliaYic8KUfNfiaSjVnruzTQUTCA0lmv9vUmw/300?wx_fmt=png&wxfrom=18)

字节跳动技术团队

221024

Comment

Comment

**Comment**

暂无留言
