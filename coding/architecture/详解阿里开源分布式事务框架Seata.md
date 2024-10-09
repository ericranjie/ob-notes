# 

博文视点Broadview

_2021年11月24日 18:04_

👆点击“博文视点Broadview”，获取更多书讯

![图片](https://mmbiz.qpic.cn/mmbiz_png/PW0wIHxgg3nr1VNxfeqxVOw2nPJHVH4xeZibzPY5F4ibOuOZLMsUMrzIibGB6KMw7EurSKv6DkrtLzuhYdBa30A9Q/640?wx_fmt=png&wxfrom=13&tp=wxpic)

**Seata发展历史**

笔者于2014年开始着手解决阿里巴巴集团内部业务的分布式事务问题，从0到1研发一个支持非侵入模式（内部称之为AT模式，即自动模式）和TCC模式（内部称之为MT模式，即手动模式）的分布式事务中间件TXC（Taobao Transaction Constructor）。TXC被广泛应用于阿里巴巴集团内部业务，主要用于解决HSF服务框架下多个数据库读写间的一致性问题。在实际业务使用中，以非侵入模式为主，TCC模式为辅助。

2016年开始，这个产品以云服务的形式对外输出，名称为GTS（Global Transaction Service），服务于众多大型私有云用户和公有云用户。

2019年1月，阿里巴巴中间件团队推出了GTS开源版本 Fescar（Fast & Easy Commit And Rollback），并和开源社区一起共建开源分布式事务解决方案。Fescar的愿景是：让分布式事务的使用像本地事务的使用一样简单和高效，并逐步解决开发者们遇到的分布式事务方面的所有难题。

在Fescar开源后，蚂蚁金服加入Fescar社区参与共建，随后Fescar被改名为Seata（Simple Extensible Autonomous Transaction Architecture）。虽然Seata目前已经包含了多种事务模式，但其最吸引客户的始终是AT模式，因为技术发展趋势一定是从侵入式到非侵入式，提高研发效率。

![图片](https://mmbiz.qpic.cn/mmbiz_png/PW0wIHxgg3keKhvMtsv1TT1ziaTLLR6Rcr3FCKeJsaYdkQT1NNhhbVwtC3sAue8Kv4RCCgcf64dnibwIDNwIVOcQ/640?wx_fmt=png&wxfrom=13&tp=wxpic)

![图片](https://mmbiz.qpic.cn/mmbiz_png/pHvnwibKGVNuNor33yicchSlluicqw0nnthmWRUYWXeKaHGR7ribQFFMWkehxnSOOPuNPuGwtDYtBcGibDbLDSqXzsw/640?wx_fmt=png&wxfrom=13&tp=wxpic)

Seata已经是Github上一个大热的项目，开源两年多截止2021年8月已有两万多的“star”数和六千多的“fork”数，项目还在快速迭代中。目前Seata已成为业界主流的分布式事务解决方案。

![图片](https://mmbiz.qpic.cn/mmbiz_png/rLwc2F8FdxicwX7BD3kbZomnaTibKBt0WGH8QEkbuHqSFayuN7M8sicME9SMXgCBuIaG4icWhF9scNKYhUG4vOZ0fA/640?wx_fmt=png&wxfrom=13&tp=wxpic)

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**Seata总体架构**

Seata代码总量不大，目录结构比较简洁。

**模块组成**

Seata的目录结构如下图所示。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

各模块的说明如下。

- \*\*all：\*\*只有一个pom文件，指定了Seata依赖哪些包。

- \*\*bom：\*\*只有一个pom文件，指定了Seata依赖管理的包，即pom文件的<dependencyManagement>节点的内容。

- \*\*changes：\*\*描述了版本变更情况。

- \*\*common：\*\*通用模块。定义了通用的工具类、线程工具、异常、加载类等。

- \*\*compressor：\*\*压缩模块。定义了多种主流压缩格式，比如Zip、Gzip、7z、Lz4、Bzip2等，用来实现消息压缩功能。

- \*\*config：\*\*配置模块。用于连接和操作配置中心，支持多种主流配置中心组件，包括Nacos、Apollo、Etcd、Consul、ZooKeeper等。

- \*\*core：\*\*核心模块。定义了RPC、Netty、事件、协议、事务上下文等。

- \*\*discovery：\*\*发现模块。用于服务发现，支持多种主流的可用作微服务注册中心的组件，包括Nacos、Etcd、Eureka、Redis、ZooKeeper等。

- \*\*distribution：\*\*只有一个pom文件，用于打包发布。

- \*\*integration：\*\*整合模块。整合了Seata对多种RPC框架的支持，包括Dubbo、gRPC、SOFA-RPC等。用来实现Seata事务上下文在RPC框架的传递。

- \*\*metrics：\*\*度量模块。用于收集一些Seata运行指标数据，并导出到一些监控系统（如使用广泛的普罗米修斯）中。

- \*\*rm：\*\*资源管理器模块。定义了多种类型资源管理器（AT模式的资源管理器、TCC模式的资源管理器、Saga模式的资源管理器、XA模式的资源管理器）的公共组件。

- \*\*rm-datasource：\*\*AT模式的资源管理器模块。实现了AT模式的数据源代理、SQL语句处理等。该模块也包括对XA模式的支持。

- \*\*saga：\*\*Saga模式的资源管理器模块。用来实现对Saga模式事务的支持。

- \*\*script：\*\*脚本模块。定义了需要的脚本文件和配置文件。

- \*\*seata-spring-boot-starter：\*\*用于与Spring Boot结合，简化使用。

- \*\*serializer：\*\*序列化模块。用于Seata消息序列化和反序列化，支持多种协议，包括Seata私有序列化协议、FST、Hessian、Kryo等。

- \*\*server：\*\*服务端模块。用来实现事务协调器，维护全局事务和分支事务的状态，推进事务两阶段提交/回滚。

- \*\*spring：\*\*Spring支持模块。定义了Seata事务注解。

- \*\*sqlparser：\*\*SQL解析模块。Seata使用Druid SQL解释器。

- \*\*style：\*\*定义了代码规范。

- \*\*tcc：\*\*TCC模式资源管理器模块。用来实现对TCC事务模式的支持。

- \*\*test：\*\*测试模块。

- \*\*tm：\*\*事务协调器模块。定义了全局事务的范围、开启全局事务、提交/回滚全局事务。

**逻辑结构**

Seata有3个主要角色：TM（Transaction Manager）、RM（Resource Manager）和TC（Transaction Coordinator）。

其中，TM和RM是以SDK的形式作为Seata的客户端与业务系统集成在一起，TC作为Seata的服务端独立部署，如下图所示。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

\*\*TM：\*\*事务管理器。与TC交互，开启、提交、回滚全局事务。

\*\*RM：\*\*资源管理器。与TC交互，负责资源的相关处理，包括分支事务注册与分支事务状态上报。

\*\*TC：\*\*事务协调器。维护全局事务和分支事务的状态，推进事务两阶段处理。对于AT模式的分支事务，TM负责事务并发控制。

Seata处理分布式事务的主要流程如下图所示。

\*\*（1）\*\*TM开启全局事务（TM 向TC 开启全局事务）。

\*\*（2）\*\*事务参与者通过RM与资源交互，并注册分支事务（RM向TC注册分支事务）。

\*\*（3）\*\*事务参与者在完成资源操作后，上报分支事务状态（RM向TC上报分支事务完成状态）。

\*\*（4）\*\*TM结束全局事务，事务一阶段结束（TM向TC提交/回滚全局事务）。

\*\*（5）\*\*TC 推进事务二阶段操作（TC向RM发起二阶段提交/回滚）。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**Seata 事务模式**

Seata支持4种事务模式：AT、TCC、Saga、XA。本节做一个简要说明，后面章节会对AT模式和TCC模式进行深入剖析。

**AT模式**

AT 模式是 Seata 主推的分布式事务解决方案，对业务无侵入，真正做到了业务与事务分离，用户只需关注自己的“业务 SQL语句”。

AT模式使用起来非常简单，与完全没有使用分布式事务方案相比，业务逻辑不需要修改，只需要增加一个事务注解@GlobalTransactional即可，如下图所示。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**TCC模式**

TCC 模式需要用户根据自己的业务场景实现 try()、confirm() 和 cancel()这3个方法：事务发起方在一阶段执行try()方法，在二阶段提交执行 confirm()方法，在二阶段回滚执行cancel()方法。

在TCC模式中，Seata框架把每组TCC服务接口当作一个资源（TCC Resource）。这套TCC服务接口可以是RPC，也可以是服务内JVM调用。在业务启动时，Seata框架会自动扫描并识别出TCC服务接口的发布方和调用方：

对于发布方，则Seata框架会在业务启动时向TC注册TCC Resource。与 AT模式的DataSource Resource 一样，每个TCC Resource也会带有一个资源 ID。

对于调用方，则Seata 框架会给其加上切面，在运行时该切面会拦截所有对 TCC 服务接口的调用。每调用一次 try 接口，切面都会先向TC 注册一个分支事务，然后才去执行try()方法的业务逻辑并向TC汇报分支事务状态。

在请求链路调用完成后，发起方通知 TC 提交或回滚分布式事务，进入二阶段调用流程。此时，TC 会根据之前注册的分支事务，回调对应参与者去执行TCC 资源的 confirm() 或 cancel()方法。

Seata TCC 框架本身很简单，\*\*主要是扫描TCC服务接口、注册资源、拦截接口调用、注册分支事务、汇报分支事务状态、回调二阶段接口。\*\*对于TCC模式来说，最复杂逻辑是TCC服务接口的实现。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

用户以TCC模式接入Seata框架，最重要的是考虑如何将自己的业务模型拆成两阶段来实现。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

TCC 模式与AT模式的主要区别如下。

（1）在使用上，TCC 模式依赖用户自行实现的3个方法（try()、confirm()、cancel()）成本较大；AT 模式依赖全局事务注解和代理数据源，代码基本不需要改动，对业务无侵入、接入成本极低。

（2）TCC 模式的作用范围在应用层，本质上是实现针对某种业务逻辑的正向和反向方法；AT模式的作用范围在底层数据源，通过保存操作行记录的前、后镜像和生成反向SQL语句进行补偿操作，对上层应用透明。

（3）TCC模式事务并发控制由业务自行“加锁”，AT模式由Seata框架自动“加锁”。

**1．举例**

以“扣钱”场景为例，在接入 TCC 模式前，对账户“扣钱”，只需一条更新账户余额的 SQL 语句就能完成；但是在接入 TCC 模式之后，用户则需要考虑如何将原来一步就能完成的“扣钱”操作拆成两阶段，实现成3个方法，并且保证如果一阶段try()方法成功则二阶段 confirm()方法也一定能成功。

如下图所示，try()方法在一阶段执行，需要做资源的检查和预留。在“扣钱”场景下，try()方法要做的是检查账户余额是否充足、预留转账资金（预留的方式就是冻结A账户的转账金额）。在try()方法执行后，账户A的余额虽然还是100元，但是其中有30元已经被冻结了，不能被其他事务使用。

二阶段confirm()方法执行真正的“扣钱”操作。confirm()方法会使用try()方法冻结的金额执行账号“扣钱”。在confirm()方法执行后，账户 A 在一阶段中冻结的 30 元已经被扣除，账户 A 的余额变为 70 元 。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

如果二阶段是回滚，则需要在cancel()方法内释放一阶段try()方法冻结的 30 元，使账户A回到初始状态，100元全部可用。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

相比AT模式，TCC模式对业务代码有很强的侵入性。但是 TCC 模式没有AT模式的全局行锁，“加锁”逻辑完全需要根据业务特点制定。

在一些场景下，TCC模式的性能会比AT模式的性能更好。但多数场景下，TCC模式业务自己实现的“加锁”机制性能不会有明显优势，反而有较大劣势。建议用AT模式作为默认方案，用TCC模式作为补充方案。‍

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**Saga模式**

Saga理论出自Hector和Kenneth 1987发表的论文SAGAS。

Saga是一种补偿协议。在 Saga 模式中，在分布式事务内有多个参与者，每一个参与者都是一个冲正补偿服务，需要用户根据业务场景实现其正向操作和逆向回滚操作。

如下图所示，T1～T3都是正向的业务流程，都对应着一个冲正逆向操作C1～C3。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

在分布式事务执行过程中，会依次执行各参与者的正向操作。

如果所有正向操作均执行成功，则分布式事务提交。

如果任何一个正向操作执行失败，则分布式事务会退回去执行前面各参与者的逆向回滚操作，回滚已提交的参与者，使分布式事务回到初始状态。

Saga 模式的正向服务与补偿服务也需要业务开发者实现，因此也具有很强的业务侵入性。在Saga 模式中，分布式事务通常是由事件驱动的，在各个参与者之间是异步执行的。

Saga 模式是一种长事务解决方案，适用于业务流程长且需要保证事务最终一致性的业务系统。Saga 模式的一阶段就会提交本地事务，在无锁、长流程情况下这可以保证性能。

**Saga模式的优势：**

在一阶段提交本地数据库事务，无锁，高性能。

参与者可以采用事件驱动异步执行，高吞吐。

补偿服务即正向服务的“反向”操作，易于理解，易于实现。

\*\*Saga模式也存在很明显的缺点：\*\*在一阶段已经提交了本地数据库事务，且没有进行“预留”动作，所以不能保证隔离性，不容易进行并发控制。与AT模式和TCC模式相比，Saga 模式的适用场景有限。

**XA模式**

在XA模式中，需要在Seata定义的分布式事务范围内，利用\*\*事务资源（数据库、消息服务等）\*\*实现对XA协议的支持，以XA协议的机制来管理分支事务。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

本质上，Seata另外的3大事务模式（AT、TCC、Saga）都是补偿型的。事务处理机制构建在框架或应用中。事务资源本身对分布式事务是无感知的。

而在XA模式下，事务资源对分布式事务是可感知的。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

XA协议要求事务资源本身提供对规范和协议的支持。因为事务资源（数据库、消息队列）可感知并参与分布式事务处理过程，所以事务资源可以保障从任意视角对数据的访问进行有效隔离，满足全局数据一致性。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

XA模式是传统的分布式强一致性的解决方案，性能较低，在实际业务中使用得较少，本书不做深入探讨。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

\*本文节选自

《正本清源分布式事务之Seata（全彩）》一书

Seata作者编写，分布式事务处理

欢迎阅读本书了解更多精彩内容！

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**京东每满100-50，到手价仅需59元**

**快快扫码抢购吧！**

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

如果喜欢本文

欢迎 **在看**丨**留言**丨**分享至朋友圈** 三连

**热文推荐**

- [图论算法：稳定婚姻问题](http://mp.weixin.qq.com/s?__biz=MjM5NTk0NjMwOQ==&mid=2651143435&idx=1&sn=fc5603b9feb4c3e959ae282eeaefa430&chksm=bd0157218a76de37fffa48ddfa25582634f3f1f49327c84c1979427f366f8c3b10dc0974c9a7&scene=21#wechat_redirect)

- [60万字诚意续作《腾讯游戏开发精粹Ⅱ》正式发布](http://mp.weixin.qq.com/s?__biz=MjM5NTk0NjMwOQ==&mid=2651143374&idx=1&sn=d1479e02c2d85ee8a41016bd14fb33a1&chksm=bd0156e48a76dff264d5d9fbc0dda278aefe21454346db93ec49ffeb42b0504aa359a01e27ce&scene=21#wechat_redirect)

- [Effective C++学习笔记](http://mp.weixin.qq.com/s?__biz=MjM5NTk0NjMwOQ==&mid=2651143297&idx=1&sn=e21e3e5e8c2a417789370365a75f72ec&chksm=bd0156ab8a76dfbd4935304a3de4c14b90ff42430b969cbf4f1cd21256e29c94ffff0aa4a9f5&scene=21#wechat_redirect)

- [EDG夺冠刷屏，好多看《英雄联盟》的朋友](http://mp.weixin.qq.com/s?__biz=MjM5NTk0NjMwOQ==&mid=2651143296&idx=1&sn=87b4788c720d0ea61b548f67ed31d9e1&chksm=bd0156aa8a76dfbc5e6b91d5550f583d0b61e57666415c4972e90476f22873f0018925537559&scene=21#wechat_redirect)

______________________________________________________________________

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

▼点击阅读原文，查看本书详情~

阅读原文

阅读 1105

​
