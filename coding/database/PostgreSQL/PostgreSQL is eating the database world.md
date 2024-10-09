原创 冯若航 非法加冯

_2024年03月16日 12:52_ _北京_

PostgreSQL isn’t just a simple relational database; it’s a data management framework with the potential to engulf the entire database realm. The trend of “Using Postgres for Everything” is no longer limited to a few elite teams but is becoming a mainstream best practice.

**中文版**：《 [PostgreSQL正在吞噬数据库世界](http://mp.weixin.qq.com/s?__biz=MzU5ODAyNTM5Ng==&mid=2247487055&idx=1&sn=9d7bd8b6d9b07478dba7f87d0a663535&chksm=fe4b3b94c93cb282f8d9c228425d01f29f753d4c9fbf661a1bf8ea4b0ec6a5bf6d7f2ee58f86&scene=21#wechat_redirect) 》

**英文版**：https://medium.com/@fengruohang/postgres-is-eating-the-database-world-157c204dcfc4

______________________________________________________________________

## OLAP's New Challenger

In a 2016 database meetup, I argued that a significant gap in the PostgreSQL ecosystem was the lack of a **sufficiently good** columnar storage engine for OLAP workloads. While PostgreSQL itself offers lots of analysis features, its performance in full-scale analysis on larger datasets doesn’t quite measure up to dedicated real-time data warehouses.

Consider **ClickBench**, an analytics performance benchmark, where we’ve documented the performance of PostgreSQL, its ecosystem extensions, and derivative databases. The untuned PostgreSQL performs poorly (**x1050**), but it can reach (**x47**) with optimization. Additionally, there are three analysis-related extensions: columnar store **Hydra** (**x42**), time-series **TimescaleDB** (**x103**), and distributed **Citus** (**x262**).

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

This performance can't be considered bad, especially compared to pure OLTP databases like MySQL and MariaDB (**x3065, x19700**); however, its third-tier performance is not "good enough," lagging behind the first-tier OLAP components like Umbra, ClickHouse, Databend, SelectDB (**x3~x4**) by an order of magnitude. It's a tough spot - not satisfying enough to use, but too good to discard.

However, the arrival of **ParadeDB** and **DuckDB** changed the game!

**ParadeDB**'s native PG extension **pg_analytics** achieves second-tier performance (**x10**), narrowing the gap to the top tier to just 3–4x. Given the additional benefits, this level of performance discrepancy is often acceptable - ACID, freshness and real-time data without ETL, no additional learning curve, no maintenance of separate services, not to mention its ElasticSearch grade full-text search capabilities.

**DuckDB** focuses on pure OLAP, pushing analysis performance to the extreme (**x3.2**) — excluding the academically focused, closed-source database Umbra, DuckDB is arguably the fastest for practical OLAP performance. It’s not a PG extension, but PostgreSQL can fully leverage DuckDB’s analysis performance boost as an embedded file database through projects like **DuckDB FDW** and **pg_quack**.

The emergence of ParadeDB and DuckDB propels PostgreSQL's analysis capabilities to the top tier of OLAP, filling the last crucial gap in its analytic performance.

______________________________________________________________________

## The Pendulum of Database Realm

The distinction between OLTP and OLAP didn’t exist at the inception of databases. The separation of OLAP data warehouses from databases emerged in the 1990s due to traditional OLTP databases struggling to support analytics scenarios' query patterns and performance demands.

For a long time, best practice in data processing involved using MySQL/PostgreSQL for OLTP workloads and syncing data to specialized OLAP systems like Greenplum, ClickHouse, Doris, Snowflake, etc., through ETL processes.

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

Like many "specialized databases," the strength of dedicated OLAP systems often lies in **performance** — achieving 1-3 orders of magnitude improvement over native PG or MySQL. The **cost**, however, is redundant data, excessive data movement, lack of agreement on data values among distributed components, extra labor expense for specialized skills, extra licensing costs, limited query language power, programmability and extensibility, limited tool integration, poor data integrity and availability compared with a complete DMBS.

However, as the saying goes, “What goes around comes around”. With [hardware improving over thirty years following Moore's Law](http://mp.weixin.qq.com/s?__biz=MzU5ODAyNTM5Ng==&mid=2247486489&idx=1&sn=f2be1be496de46ac5ca816ac39cfdf24&chksm=fe4b39c2c93cb0d4ff50dd6962370523a6271eab478fe9174c0c7a88fc88ea05fd3e51313ad3&scene=21#wechat_redirect), performance has increased exponentially while costs have plummeted. In 2024, a single x86 machine can have hundreds of cores (512 vCPU EPYC 9754 x2), several TBs of RAM, a single NVMe SSD can hold up to 64TB, and a single all-flash rack can reach 2PB; object storage like S3 offers virtually unlimited storage.

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

Hardware advancements have solved the data volume and performance issue, while database software developments (PostgreSQL, ParadeDB, DuckDB) have addressed access method challenges. This puts the fundamental assumptions of the analytics sector — the so-called “big data” industry — under scrutiny.

As DuckDB's manifesto "**Big Data is Dead**" suggests, **the era of big data is over**. Most people don't have that much data, and most data is seldom queried. The frontier of big data recedes as hardware and software evolve, rendering "big data" unnecessary for 99% of scenarios.

If 99% of use cases can now be handled on a single machine with standalone DuckDB or PostgreSQL (and its replicas), what's the point of using dedicated analytics components? If every smartphone can send and receive texts freely, what's the point of pagers? (With the caveat that North American hospitals still use pagers, indicating that maybe less than 1% of scenarios might genuinely need "big data.")

The shift in fundamental assumptions is steering the database world from a phase of diversification back to convergence, from a big bang to a mass extinction. In this process, a new era of unified, multi-modeled, super-converged databases will emerge, reuniting OLTP and OLAP. But who will lead this monumental task of reconsolidating the database field?

______________________________________________________________________

## PostgreSQL: The Database World Eater

There are a plethora of niches in the database realm: time-series, geospatial, document, search, graph, vector databases, message queues, and object databases. PostgreSQL makes its presence felt across all these domains.

A case in point is the PostGIS extension, which sets the de facto standard in geospatial databases; the TimescaleDB extension awkwardly positions “generic” time-series databases; and the vector extension, [**PGVector**](http://mp.weixin.qq.com/s?__biz=MzU5ODAyNTM5Ng==&mid=2247485589&idx=1&sn=931f2d794e9b8486f623f746db9f00cd&chksm=fe4b3d4ec93cb4584c9bb44b1f347189868b6c8367d8c3f8dd8703d1a906786a55c900c23761&scene=21#wechat_redirect), turns the [dedicated vector database niche into a punchline](http://mp.weixin.qq.com/s?__biz=MzU5ODAyNTM5Ng==&mid=2247486505&idx=1&sn=a585c9ff22a81a8efe6b87ce9bd66cb1&chksm=fe4b39f2c93cb0e4c5d46f54e7ba9309dc0d66b5ac73bfe6722cc39f3959e47ae78210aeea1f&scene=21#wechat_redirect).

This isn’t the first time; we’re witnessing it again in the oldest and largest subdomain: OLAP analytics. But PostgreSQL’s ambition doesn’t stop at OLAP; it’s eyeing the entire database world!

What makes PostgreSQL so capable? Sure, it's advanced, but so is Oracle; it's open-source, as is MySQL. PostgreSQL's edge comes from being **both advanced and open-source**, allowing it to compete with Oracle/MySQL. But its true uniqueness lies in its **extreme extensibility and thriving extension ecosystem.**

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

> Reasons users choose PostgreSQL: **Open-Source, Reliable, Extensible**

PostgreSQL isn’t just a relational database; it’s a data management framework capable of engulfing the entire database galaxy. Besides being open-source and advanced, its core competitiveness stems from **extensibility**, i.e., its infra’s reusability and extension's composability.

______________________________________________________________________

### The Magic of Extreme Extensibility

PostgreSQL allows users to develop extensions, leveraging the database's common infra to deliver features at minimal cost. For instance, the vector database extension **pgvector**, with just several thousand lines of code, is negligible in complexity compared to PostgreSQL's millions of lines. Yet, this "insignificant" extension achieves complete vector data types and indexing capabilities, outperforming lots of specialized vector databases.

Why? Because pgvector's creators didn't need to worry about the database's general additional complexities: ACID, recovery, backup & PITR, high availability, access control, monitoring, deployment, 3rd-party ecosystem tools, client drivers, etc., which require millions of lines of code to solve well. They only focused on the essential complexity of their problem.

For example, ElasticSearch was developed on the Lucene search library, while the Rust ecosystem has an improved next-gen full-text search library, Tantivy, as a Lucene alternative. ParadeDB only needs to wrap and connect it to PostgreSQL's interface to offer search services comparable to ElasticSearch. More importantly, it can stand on the shoulders of PostgreSQL, leveraging the entire PG ecosystem's united strength (e.g., mixed searches with PG Vector) to "unfairly" compete with another dedicated database.

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

> Pigsty has 234 **extensions** available. And there are 1000+ more in the ecosystem

______________________________________________________________________

The extensibility brings another huge advantage: the **composability** of extensions, allowing different extensions to work together, creating a synergistic effect where 1+1 >> 2. For instance, TimescaleDB can be combined with PostGIS for spatio-temporal data support; the BM25 extension for full-text search can be combined with the PGVector extension, providing hybrid search capabilities.

Furthermore, the **distributive** extension **Citus** can transparently transform a standalone cluster into a horizontally partitioned distributed database cluster. This capability can be orthogonally combined with other features, making PostGIS a distributed geospatial database, PGVector a distributed vector database, ParadeDB a distributed full-text search database, and so on.

______________________________________________________________________

What’s more powerful is that extensions **evolve independently**, without the cumbersome need for main branch merges and coordination. This allows for scaling — PG’s extensibility lets numerous teams explore database possibilities in parallel, with all extensions being optional, not affecting the core functionality’s reliability. Those features that are mature and robust have the chance to be stably integrated into the main branch.

PostgreSQL achieves both foundational **reliability** and **agile functionality** through the magic of **extreme extensibility**, making it an outlier in the database world and changing the game rules of the database landscape.

______________________________________________________________________

## Game Changer in the DB Arena

**The emergence of PostgreSQL has shifted the paradigms in the database domain**: Teams endeavoring to craft a “new database kernel” now face a formidable trial — how to stand out against the open-source, feature-rich Postgres. What’s their unique value proposition?

Until a revolutionary hardware breakthrough occurs, the advent of practical, new, general-purpose database kernels seems unlikely. No singular database can match the overall prowess of PG, bolstered by all its extensions — not even Oracle, given PG’s ace of being open-source and free.

A niche database product might carve out a space for itself if it can outperform PostgreSQL by an order of magnitude in specific aspects (typically performance). However, it usually doesn’t take long before the PostgreSQL ecosystem spawns open-source extension alternatives. Opting to develop a PG extension rather than a whole new database gives teams a crushing speed advantage in playing catch-up!

Following this logic, the PostgreSQL ecosystem is poised to snowball, accruing advantages and inevitably moving towards a monopoly, mirroring the Linux kernel’s status in server OS within a few years. Developer surveys and database trend reports confirm this trajectory.

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

> **StackOverflow 2023 [Survey](http://mp.weixin.qq.com/s?__biz=MzU5ODAyNTM5Ng==&mid=2247485170&idx=1&sn=657c75be06557df26e4521ce64178f14&chksm=fe4b3329c93cba3f840283c9df0e836e96a410f540e34ac9b1b68ca4d6247d5f31c94e2a41f4&scene=21#wechat_redirect): PostgreSQL, the Decathlete**\[21\]

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

> **StackOverflow's [Database Trends](http://mp.weixin.qq.com/s?__biz=MzU5ODAyNTM5Ng==&mid=2247485933&idx=3&sn=ea360aa7a59a4cd23ad5f9a9f415a0a0&chksm=fe4b3c36c93cb520bda4596136e927d7cf92c597a76c04077c256588b2428202bdb7f004c08b&scene=21#wechat_redirect) Over the Past 7 Years**

PostgreSQL has long been the favorite database in HackerNews & StackOverflow. Many new open-source projects default to PostgreSQL as their primary, if not only, database choice. And many new-gen companies are going All in PostgreSQL.

As “**Radical Simplicity: Just Use Postgres**” says, Simplifying tech stacks, reducing components, accelerating development, lowering risks, and adding more features can be achieved by **“Just Use Postgres.”** Postgres can replace many backend technologies, including MySQL, Kafka, RabbitMQ, ElasticSearch, Mongo, and Redis, effortlessly serving millions of users. [**Just Use Postgres**](http://mp.weixin.qq.com/s?__biz=MzU5ODAyNTM5Ng==&mid=2247486931&idx=1&sn=91dbe43bb6d26c760c532f4aa8d6e3cb&chksm=fe4b3808c93cb11e00194655a49bf7aa0d4d05a61a9b06ffcc57017c633de17066443ec62b6d&scene=21#wechat_redirect) is no longer limited to a few elite teams but becoming a mainstream best practice.

______________________________________________________________________

## What Else Can Be Done?

The endgame for the database domain seems predictable. But what can we do, and what should we do?

PostgreSQL is already a near-perfect database kernel for the vast majority of scenarios, making the idea of a kernel "bottleneck" absurd. Forks of PostgreSQL and MySQL that tout kernel modifications as selling points are essentially going nowhere.

This is similar to the situation with the Linux OS kernel today; despite the plethora of Linux distros, everyone opts for the same kernel. Forking the Linux kernel is seen as creating unnecessary difficulties, and the industry frowns upon it.

Accordingly, the main conflict is no longer the database kernel itself but two directions— database **extensions** and **services**! The former pertains to internal extensibility, while the latter relates to external composability. Much like the OS ecosystem, the competitive landscape will concentrate on **database distributions**. In the database domain, only those distributions centered around extensions and services stand a chance for ultimate success.

Kernel remains lukewarm, with MariaDB, the fork of MySQL’s parent, nearing delisting, while AWS, profiting from offering services and extensions on top of the free kernel, thrives. Investment has flowed into numerous PG ecosystem extensions and service distributions: Citus, TimescaleDB, Hydra, PostgresML, ParadeDB, FerretDB, StackGres, Aiven, Neon, Supabase, Tembo, PostgresAI, and our own PG distro — — [**Pigsty**](http://mp.weixin.qq.com/s?__biz=MzU5ODAyNTM5Ng==&mid=2247485518&idx=1&sn=3d5f3c753facc829b2300a15df50d237&chksm=fe4b3d95c93cb4833b8e80433cff46a893f939154be60a2a24ee96598f96b32271301abfda1f&scene=21#wechat_redirect).

______________________________________________________________________

A dilemma within the PostgreSQL ecosystem is the independent evolution of many extensions and tools, lacking a unifier to synergize them. For instance, Hydra releases its own package and Docker image, and so does PostgresML, each distributing PostgreSQL images with their own extensions and only their own. These images and packages are far from comprehensive database services like AWS RDS.

Even service providers and ecosystem integrators like AWS RDS [fall short in](http://mp.weixin.qq.com/s?__biz=MzU5ODAyNTM5Ng==&mid=2247487111&idx=1&sn=f02c3a7fc8ba9cc0919518b3c5805675&chksm=fe4b3b5cc93cb24a71d0847bcd7f1655f7466f7f7864457413d3ca80cd79df974b7b9279b9e9&scene=21#wechat_redirect) front of numerous extensions, unable to include many due to various reasons (AGPLv3 license, security challenges with multi-tenancy), thus failing to leverage the synergistic amplification potential of PostgreSQL ecosystem extensions.

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

Extensions are the soul of PostgreSQL. A Postgres without the freedom to use extensions is like cooking without salt, a giant constrained.

Addressing this issue is one of our primary goals.

______________________________________________________________________

## Our Resolution: Pigsty

Despite earlier exposure to MySQL and MSSQL, when I first used PostgreSQL in 2015, I was convinced of its future dominance in the database realm. Nearly a decade later, I’ve transitioned from a user and administrator to a contributor and developer, witnessing PG’s march toward that goal.

Interactions with diverse users revealed that the database field's shortcoming isn't the kernel anymore — PostgreSQL is already sufficient. The real issue is **leveraging the kernel’s capabilities**, which is the reason behind RDS’s booming success.

However, I believe this capability should be as accessible as free software, like the PostgreSQL kernel itself — available to every user, not just renting from cyber feudal lords.

Thus, I created **Pigsty**, a battery-included, local-first PostgreSQL distribution as an open-source RDS Alternative, which aims to harness the collective power of PostgreSQL ecosystem extensions and democratize access to production-grade database services.

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

> Pigsty stands for **P**ostgreSQL **i**n **G**reat **STY**le

We’ve defined six core propositions addressing the central issues in PostgreSQL database services: **Extensible Postgres**, **Reliable Infras**, **Observable Graphics**, **Available Services**, **Maintainable Toolbox**, and **Composable Modules**.

The initials of these value propositions offer another acronym for Pigsty:

> **P**ostgres, **I**nfras, **G**raphics, **S**ervice, **T**oolbox, **Y**ours.
>
> Your graphical Postgres infrastructure service toolbox.

**Extensible PostgreSQL** is the linchpin of this distribution. In the recently launched **Pigsty v2.6**, we integrated DuckdbFDW and ParadeDB extensions, massively boosting PostgreSQL’s analytical capabilities and ensuring every user can easily harness this power.

Our aim is to integrate the strengths within the PostgreSQL ecosystem, creating a synergistic force akin to the **Ubuntu** of the database world. I believe the kernel debate is settled, and the real competitive frontier lies here.

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

Developers, your choices will shape the future of the database world. I hope my work helps you better utilize the world’s most advanced open-source database kernel: **PostgreSQL**.

______________________________________________________________________

试水将这篇文章翻译成英文，在 HackerNews 上大受欢迎。后面准备将本号文章翻译成英文，让数据库老司机与云计算泥石流走向海外😎。欢迎大家关注我的英文博客：https://medium.com/@fengruohang

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

![](https://mmbiz.qlogo.cn/mmbiz_jpg/e5xS3Vwicpl8vEzwicmWCF70Y4HnvILGe2WlJnGNdXqQOyUq83icsPshyXHt8ytQhusdhppIKm6xs3TbYzVlHbJWA/0?wx_fmt=jpeg)

冯若航

![赞赏二维码](https://mp.weixin.qq.com/s?__biz=MzU5ODAyNTM5Ng==&mid=2247487156&idx=1&sn=dea8abae114716013dbc4f5fb978064d&chksm=fe4b3b6fc93cb27926cf450af31bc03d826da44120aadc288303ffd64301c99b3596459776b6&mpshare=1&scene=24&srcid=0316DcC4P9II7iTtB3boyEOx&sharer_shareinfo=88f7714cb21d772225e47998bd545f80&sharer_shareinfo_first=88f7714cb21d772225e47998bd545f80&key=daf9bdc5abc4e8d0700ea71c0e2deca03f4addae4b9c71eaac70f5ef25ded1a8984c90e44d63a87f332dd468bc581e2d24ae0d2100b85099ea6ad2a07f404f547135df7794de577106e860ae55f78c995f752ad2f162afcabc1687f01365a09d831a94e088328722d36f2d6fd157c81553ff0d26f1bbf93e28b187a18194bf99&ascene=0&uin=MTEwNTU1MjgwMw%3D%3D&devicetype=Windows+11+x64&version=63090b19&lang=zh_CN&countrycode=CN&exportkey=n_ChQIAhIQrI5zfcU0prX8tGaLpm5RsBLmAQIE97dBBAEAAAAAAFmhFsdAfjAAAAAOpnltbLcz9gKNyK89dVj0B2BoyTXERBLb3ZThcm5Cx8aILFI%2Fez9bjaIg6Y1NYglemewm9c6F0t7r%2BOHyNXGEveM18RFHu22NxJLhGJxSlalTXBNya1XCG2ZNWzmKIMPKDqwH%2FdMWjTW7O6guxrSuh5%2FXNIJ17tIMnf5LJ5tlIW0JDtOqIVFOLSqhH9W9gMHgLXoIEGxUlzDxTSnCMRUld1i9tSY3E2ixgknLJfqJ85oNtMELsdbarmoqzCQHgA9SCzkRbI8oHrfizhpIKRlq&acctmode=0&pass_ticket=nxkhfgI1vbCW2b%2Ff5tbTr14n3xbiRVrr1hiafUwPktOmNPfB6xzdB7CHcznHeRsl&wx_header=1&fasttmpl_type=0&fasttmpl_fullversion=7350504-zh_CN-zip&fasttmpl_flag=1)喜欢作者

PostgreSQL81

PostgreSQL · 目录

上一篇RDS阉掉了PostgreSQL的灵魂下一篇PostgreSQL会修改开源许可证吗？

阅读原文

阅读 1845

​
