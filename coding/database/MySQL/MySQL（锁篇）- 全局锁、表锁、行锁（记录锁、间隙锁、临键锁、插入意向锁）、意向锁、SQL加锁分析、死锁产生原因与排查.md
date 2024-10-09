[](https://juejin.cn/user/4022488531210768/posts)

2024-05-146,144阅读31分钟

专栏：

Mysql

# 1. 一条Update语句

sql

代码解读

复制代码

`update tab_user set name='曹操' where id=1;`

执行流程：

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/75c123c5fbef4562b703d050db814475~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.avis#?w=1308&h=443&s=147105&e=png&b=fcf4f3)

# 2. MySQL锁介绍

在实际的数据库系统中，每时每刻都在发生锁定，当某个用户在修改一部分数据的时候，MySQL会通过锁定防止其他用户读取同一条数据。

在处理并发读或者写的时候，通过实现一个由两种类型的锁组成的锁系统来解决问题。两种锁通常被称为**共享锁（shared lock）和排它锁（exclusive lock）**，也叫**读锁（read lock）和写锁（write lock）。**

读锁是共享的，是互相不阻塞的。多个客户端在同一时刻可以同时读取同一个资源，而不互相干扰。写锁则是排他的，也就是说一个写锁会阻塞其他的写锁和读锁，这是出于安全策略的考虑，只有这样才能确保在给定的时间内，只有一个用户能执行写入，并防止其他用户读取正在写入的同一个资源。

## 2.1 锁分类

**按照颗粒度分：**

- 全局锁：锁整个database，由MySQL的SQL Layer层（核心服务层）实现。
- 表级锁：锁某个table，由MySQL的SQL Layer层实现。
- 行级锁：锁某Row的索引，也可锁定行索引之间的间隙，由存储引擎实现【InnoDB】

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/92a6a6c776af4fe3ae89a858e9562791~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.avis#?w=626&h=488&s=215575&e=png&b=fcfbfb)

**按锁功能分**

- **共享锁Shared Lock（S锁，也叫做读锁）**
  - 加了读锁的记录，允许其他事务再加读锁
  - 加锁方式：select ... lock in share mode
- **排它锁Exclusive Lock（X锁，也叫写锁）**
  - 加了写锁的记录，不允许其他事务再加读锁或者写锁
  - 加锁方式：select ... for update

## 2.2 全局锁

全局锁是**对整个数据库实例加锁**，**加锁后整个实例就处于只读状态**，后续DML的写语句，DDL语句，已经更新操作的事务提交语句都会被阻塞。其典型的使用场景是做**全库的逻辑备份**，对所有的表进行锁定，从而获取一致性视图，保证数据的完整性。

加全局锁的命令为：

`flush tables with read lock;`

释放全局锁的命令为：

`unlock tables;`

或者断开加锁session的连接，自动释放全局锁。

说到全局备份的事情，还是很危险的。因为如果主库加上全局锁，则整个数据库将不能写入，备份期间影响业务运行，如果在从库上加全局锁，则会导致不能执行主库同步过来的操作，造成主从延迟。

对于innoDb这种支持事务的存储引擎，使用mysqldump备份时可以使用-single-transction参数，利用mvcc提供一致性视图，而不是用全局锁，不影响业务的正常运行。而对于有MyISAM这种不支持事务的表，就只能通过全局锁获得一致性视图，对应的mysqldump参数为-lock-all-tables;

举个例子：

sql

代码解读

复制代码

`# 提交请求锁定所有数据库的所有表，以保障数据的一致性，全局锁【LBCC】 mysqldump -uroot -p --host=localhost --all-databases --lock-all-tables > /root/db.sql # 一致性视图【MVCC】 mysqldump -uroot -p --host=localhost --all-databases --single-transaction > /root/db.sql`

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5b6151f789f6440ea8738d9657caf2f9~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.avis#?w=1724&h=86&s=137458&e=png&b=2e5267)

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7e6d309d44ba4ae6a3a6699567aac4fa~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.avis#?w=1772&h=82&s=135363&e=png&b=2b4d60)

# 3. 表级锁

## 3.1 什么是表级锁？

- 表读锁（Table Read Lock）：阻塞对当前表的写，但不阻塞读
- 表写锁（Table Write Lock）：阻塞对当前表的读和写
- 元数据锁（meta data lock, MDL）：不需要显式指定，在访问表时会被自动加上，作用保证读写的正确性
  - 当对表做增删改查操作时加元数据读锁
  - 当对表做结构变更操作的时候加元数据写锁
- 自增锁（ATUO-INC Lock）：一种特殊的表级锁，自增列事务性插入操作时产生

## 3.2 表读锁、写锁

### 3.2.1 表锁相关命令

Mysql实现的表级锁定的争用状态变量；

sql

代码解读

复制代码

`# 查看表锁定状态 mysql > show status like 'table%';`

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/fb31b7e6252c4bafbe77a2464b67ba4c~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.avis#?w=868&h=426&s=260926&e=png&b=040404)

- table_locks_immediate: 产生表级锁定的次数；
- table_locks_waited: 出现表级锁定争用而发生等待的次数;

表锁有两种表现形式：

- 表读锁（Table Read Lock）
- 表写锁（Table Write Lock）

手动添加表锁：

sql

代码解读

复制代码

`lock table 表名称 read(write), 表名称2 read(write), 其他； # 举例 lock table t read; #为表t加读锁 lock table t weite; #为表t加写锁`

查看表锁情况：

sql

代码解读

复制代码

`show open tables;`

删除表锁：

sql

代码解读

复制代码

`unlock tables;`

### 3.2.2 表锁演示

1. 环境准备

sql

代码解读

复制代码

`CREATE TABLE mylock ( id int(11) NOT NULL AUTO_INCREMENT, NAME varchar(20) DEFAULT NULL, PRIMARY KEY (id) ); INSERT INTO mylock (id,NAME) VALUES (1, 'a'); INSERT INTO mylock (id,NAME) VALUES (2, 'b'); INSERT INTO mylock (id,NAME) VALUES (3, 'c'); INSERT INTO mylock (id,NAME) VALUES (4, 'd');`

2. 读锁演示：mylock表加read锁【读阻塞写】

|时间|session01|session02|
|---|---|---|
|T1|连接数据库||
|T2|获得mylock的Read Lock锁定：lock table mylock read;|连接数据库|
|T3|当前session可以查询该表记录：select * from mylock;|其他session也可以查询该表记录： select * from mylock;|
|T4|当前session不能查询其他没有锁定表的记录：select * from t;|其他session可以查询或更新未锁定的表： update t set c = '张飞' where id =1;|
|T5|当前session插入或更新锁定的表会提示错误：insert into mylock(name) values('e');|其他session插入或更新锁定的表会一直等待获取锁： insert into mylock(name) values('e');|
|T6|释放锁：unlock tables;|插入成功|

sql

代码解读

复制代码

`-- Session01 # 获得表mylock的Read Lock锁定： lock table mylock read; # 当前Session可以查询该表记录： select * from mylock; # 当前Session不能查询其他没有锁定的表： select * from t; # 当前Session插入或更新锁定的表会提示错误： insert into mylock (name) values('e'); # 释放锁： unlock tables; -- Session02 # 其他Session也可以查询该表的记录： select * from mylock; # 其他Session可以查询或更新未锁定的表： update t set name='张飞' where id=1; # 其他Session插入或更新锁定表会一直等待获取锁： insert into mylock (name) values('e');`

3. 写锁演示：mylock表加write锁【写阻塞读】

|时间|session01|session02|
|---|---|---|
|T1|连接数据库|待session01开启锁之后，session02再获取连接|
|T2|获得mylock的Write Lock锁定：lock table mylock write;||
|T3|当前session对锁定表的查询+更新+插入操作都可以执行：select * from mylock where id =1; insert into mylock(name) values('e');|连接数据库|
|T4||其他session对锁定的表查询被阻塞，需要等待锁释放：select * from mylock where id=1;|
|T5|释放锁：unlock tables;|获得锁，返回查询结果|

sql

代码解读

复制代码

`-- Session01 # 获得表mylock的write锁： lock table mylock write; # 当前session对锁定表的查询+更新+插入操作都可以执行： select * from mylock where id=1; insert into mylock (name) values('e'); # 释放锁： unlock tables; -- Session02 # 注意：待session1开启锁后，session2再获取连接 # 其他session对锁定表的查询被阻塞，需要等待锁被释放 select * from mylock where id=1; # 获得锁，返回查询结果：`

查询操作在客户端可以正常查询，navcat会阻塞![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/91068118e9ad44abb4bec43cf83e4eec~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.avis#?w=654&h=470&s=317785&e=png&b=2b4c61)

## 3.3 元数据锁

### 3.3.1 元数据锁介绍

元数据锁不需要显式指定，在访问一个表的时候会被自动加上，锁的作用是保证读写的正确定。

可以想像一下：如果一个查询正在遍历一个表中的数据，而执行期间另一个线程对这个表的表结构做变更，删了一列，那么查询线程拿到的结果跟表结构对不上，肯定是不行的。

因此，**Mysql5.5版本中引入了元数据锁**，当一个表做增删改查的时候，**加元数据读锁**；当要对表结构做变更操作的时候，**加元数据写锁**。

- **读锁是共享的，是互相不阻塞的**：因此你可以有多个线程同时对一张表加读锁，保证数据在读取的时候不会被其他线程修改。
- **写锁则是排他的**：也就是说一个写锁会阻塞其他的写锁和读锁，用来保证变更表结构操作的安全性。因此，如果有两个线程要同时给一个表加字段，其中一个要等另一个执行完才能开始执行。

### 3.3.2 元数据锁演示

|时间|session01|session02|
|---|---|---|
|T1|开始事务：begin||
|T2|加元数据读锁 select * from mylock;|修改表结构：alter table mylock add f int;|
|T3|提交/回滚事务：commit/rollback释放锁||
|T4||获取锁，修改完成|

sql

代码解读

复制代码

`-- Session01 # 开启事务： begin; # 加元数据读锁： select * from mylock; # 提交/回滚事务： commit; # 释放锁 -- Session02 # 修改表结构： alter table mylock add f int; # 获取锁，修改完成`

# 3.4 自增锁（AUTO-INC LOCK）

自增锁是一种特殊的表级锁，发生涉及AUTO_INCREMENT列的事务性插入操作时产生。

# 4. 行级锁【重点！！！】

## 4.1 什么是行级锁？

Mysql的**行级锁**， 是由**存储引擎来实现的**，这里我们主要讲解**innoDb**的行级锁。**InnoDb行锁是通过给索引上的索引项加锁来实现的**，因此InnoDB这种行锁实现特点：

**只有通过索引条件检索的数据，Inn oDb才能使用行级锁，否则，InnoDB将使用表锁！**

- InnoDB的行级锁，按照**锁定范围**来说，分为4种：

  - **记录锁**（Record Locks）：锁定索引中的一条记录
  - **间隙锁**（Gap Locks）：要么锁住索引记录中间的值，要么锁住第一个索引记录前面的值或者最后一个索引记录后面的值。
  - **临键锁**（Next-Key Locks）：是索引记录上的记录锁和在索引记录之间的间隙锁的组合（间隙锁+记录锁）
  - **插入意向锁**（Insert Intention Locks）：做insert操作时添加的对记录ID的锁

- InnoDB的行级锁，按照**功能**来说，分为两种：

  - 读锁：允许一个事务去读一行，阻止其他事务更新目标行数据。同时阻止其他事务加写锁，但不允许其他事务加读锁。
  - 写锁：允许获得排他锁的事务更新数据，阻止其他事务获取或修改数据。同时阻止其他事务加读锁和写锁。

**如果加行级锁？**

- **对于Update、delete、insert语句InnoDB会自动给涉及数据集加写锁。**
- **对于普通的select语句，InnoDb不会加任何锁**
- 事务可以通过以下语句手动给记录集加读锁或写锁

案例：

sql

代码解读

复制代码

`` CREATE TABLE `t1_simple` ( `id` int(11) NOT NULL, `pubtime` int(11) NULL DEFAULT NULL, PRIMARY KEY (`id`) USING BTREE, INDEX `idx_pu`(`pubtime`) USING BTREE ) ENGINE = InnoDB; INSERT INTO `t1_simple` VALUES (1, 10); INSERT INTO `t1_simple` VALUES (4, 3); INSERT INTO `t1_simple` VALUES (6, 100); INSERT INTO `t1_simple` VALUES (8, 5); INSERT INTO `t1_simple` VALUES (10, 1); INSERT INTO `t1_simple` VALUES (100, 20); ``

**添加读锁**

`select * from t1_simple WHERE id = 1 lock in share mode;`

**添加写锁**

`select * from t1_simple WHERE id = 1 for update;`

## 4.2 行锁四兄弟：记录、间隙、临键、插入意向

### 4.2.1 记录锁

记录锁仅仅锁住索引记录的一行，在单条索引记录上加锁。记录锁锁住的永远是索引，而非数据本身，即使该表上没有任何显式索引，那么innodb会在后台创建一个隐藏的聚簇索引，那么锁住的就是这个隐藏的聚簇索引。

举个例子：

sql

代码解读

复制代码

`-- 加记录读锁 select * from t1_simple where id = 1 lock in share mode; -- 加记录写锁 select * from t1_simple where id = 1 for update; -- 新增，修改，删除加记录写锁 insert into t1_simple values (1, 22); update t1_simple set pubtime=33 where id =1; delete from t1_simple where id =1;`

### 4.2.2 间隙锁

1. 间隙锁（Gap Locks），仅仅锁住一个索引区间（开区间，不包含双端端点）
1. 在索引记录之间的间隙中加锁，或者是在某个索引记录之前或者之后加锁，并不包含该索引记录本身。
1. **间隙锁可用于防止幻读**，保证**索引间隙**不会被插入数据
1. 在可重复读（REPETABLE READ）这个隔离级别下生效。

**主键ID索引的行锁区间划分图**

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0e1d4c379a044721a62c48114c8b1730~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.avis#?w=952&h=460&s=78061&e=png&b=f8f8f8)

session01执行：

sql

代码解读

复制代码

`begin; select * from t1_simple where id > 4 for update; -- 加间隙锁 -- 间隙锁区间(4,100+) commit;`

session02执行：

sql

代码解读

复制代码

`begin; insert into t1_simple values (7,100); -- 阻塞 insert into t1_simple values (3,100); -- 成功 commit;`

### 4.2.3 临键锁

1. 临键锁（Next—Key Locks）相当于记录锁+间隙锁 **【左开右闭区间】**，例如(5,8\]
1. **默认情况下，innodb使用临键锁来锁定记录。** 但是在不同场景中会退化

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/02e20f002ca04eda8d4bd057a6c600fe~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.avis#?w=962&h=672&s=94769&e=png&b=f9f9f9)

- 默认情况下，InnoDB使用临键锁来锁定记录，但会在不同场景中退化
- 场景01-**唯一性字段等值（=）且记录存在，退化为记录锁**
- 场景02-**唯一性字段等值（=）且记录不存在，退化为间隙锁**
- 场景03-**唯一性字段范围（\< >），还是临键锁**
- 场景04-**非唯一性字段，默认是临键锁**

session1执行：

sql

代码解读

复制代码

`begin; select * from t1_simple where pubtime = 20 for update; -- 临键锁区间(10,20],(20,100] commit;`

session2执行：

sql

代码解读

复制代码

`begin; insert into t1_simple values (16, 19); -- 阻塞 select * from t1_simple where pubtime = 20 for update; -- 阻塞 insert into t1_simple values (16, 50); -- 阻塞 insert into t1_simple values (16, 101); -- 成功 commit;`

### 4.2.4 插入意向锁

间隙锁可以帮助我们在一定程度上解决幻读的问题，但是间隙锁就是最佳的解决思路了吗，还有没有优化空间？

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/48bc834ab50c460a9851ab17077cc6bf~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.avis#?w=948&h=298&s=198048&e=png&b=fcfcfc)

sql

代码解读

复制代码

`insert into t1_simple values (60, 200); -- 阻塞 insert into t1_simple values (70, 300); -- 阻塞`

按照之前关于间隙锁的知识分析，此时间隙锁的范围是(11,99)，意思是这个范围的id都不可以插入。如果是这样的话，数据插入效率太低，锁范围比较大，很容易发生锁冲突怎么办？

插入意向锁就是解决这个问题的。

**什么是插入意向锁？**

1. 插入意向锁是一种在INSERT操作之前设置的一种特殊的间隙锁。
1. 插入意向锁表示了一种插入意图，即当多个不同的事务，同时往同一个索引的同一个间隙插入数据的时候，他们互相之间无需等待，即不会阻塞。
1. 插入意向锁不会阻止插入意向锁，但是**插入意向锁会阻止其他间隙写锁（排他锁）、记录锁**

session01执行

sql

代码解读

复制代码

`begin; insert into t1_simple values (60, 200); -- 插入意向锁区间(10,100) commit; begin; select * from t1_simple where id > 10 for update; -- 临键锁（区间）写锁区间(10,100+) commit;`

session02执行

sql

代码解读

复制代码

`begin; insert into t1_simple values (70, 300); -- 没有发生阻塞 -- 插入意向锁区间(10,100) commit; -- 说明两个插入意向锁之间是兼容的，可以共存! begin; insert into t1_simple values (90, 300); -- 被阻塞，阻塞的原因在于，插入意向锁和其他写锁之间是互斥的！ commit;`

趁着阻塞，在新会话中，通过 `show engine innodb status\G` 指令，可以看到加锁日志信息，重点看 TRANSACTION：

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/618eaf841ccb4da7807caded794542d4~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.avis#?w=2412&h=814&s=1995672&e=png&b=2e5165)

在输出的内容中，框选中的地方，清楚的表明了插入意向锁（insert intention）的存在。

## 4.3 加锁规则【非常重要】

**主键索引**

- 等值条件：
  - 命中：加记录锁
  - 未命中，加间隙锁
- 范围条件：
  - 命中，包含where条件的临键区间，加临键锁
  - 未命中，加间隙锁

**辅助索引**

- 等值条件：
  - 命中：命中记录的辅助索引项，回表主键索引项加记录锁，辅助索引项两侧加间隙锁
  - 未命中：加间隙锁
- 范围条件
  - 命中：包含where条件的临键区间加临键锁。命中记录回表主键索引项加记录锁
  - 未命中：加间隙锁

## 4.4 意向锁

### 4.4.1 什么是意向锁？

相当于存储引擎级别的表锁。

InnoDB也实现了表锁，也就是意向锁。意向锁是Mysql内部使用的，不需要用户干预。**意向锁和行锁可以共存**，意向锁的主要作用是为了**全表更新数据时提升性能**。否则在全表更新数据的时候，需要先检索该改为是否某些记录上有行锁，那么将是一件非常繁琐且耗时的操作。

举个例子：

事务A修改user表的记录r, 会给记录r上一把行级的**写锁**，同时会给user表上一把**意向写锁（IX）**，这时事务B要给user表**上一个表级的写锁就会被阻塞**。意向锁通过这种方式实现了行锁和表锁共存，且满足事务隔离性的要求。

当我们需要加一个写锁时，需要根据意向锁区判断表里有没有数据行被锁定；

1. 如果行锁，则需要遍历每一行去确认。
1. 如果表锁，则只需要判断一次即可知道没有有数据行被锁定，提升性能。

### 4.4.2 作用

- 表明“某个事务正在某些行持有了锁、或该事务准备去持有锁”
- 意向锁的存在是为了协调行锁和表锁的关系，支持多粒度（表锁和行锁）的锁共存

### 4.4.2 意向锁和读锁、写锁的兼容关系

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e0e7c15899294f8aa1bc501ddb2862ef~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.avis#?w=1102&h=358&s=191080&e=png&b=f6efe9)

- 意向锁相互兼容：因为IX、IS只是表明申请更低层次级别元素（page、记录）的X、S操作。
- 表级S锁和X、IX锁不兼容：因为上了表级S锁后，不允许其他事务再加X锁。
- 表级X锁和IS、IX、S、X不兼容：因为上了表级X锁之后，会修改数据。

> 注意：上了行级写锁后，行级写锁不会因为有别的事务上了意向写锁而阻塞，一个Mysql是允许多个行级写锁同时存在的，只要他们不是针对相同的数据。

## 4.5 锁相关的参数

InnoDB所使用的**行级锁定**争用状态查看：

sql

代码解读

复制代码

`show status like 'innodb_row_lock%';`

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ec539f27db9c4db3876eb61eec7ada60~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.avis#?w=740&h=446&s=298306&e=png&b=2b4c61)

- Innodb_row_lock_current_waits：当前正在等待锁定的数量
- **Innodb_row_lock_time**：从系统启动到现在锁定总时间长度
- **Innodb_row_lock_time_avg**：每次等待所花费的平均时间
- Innodb_row_lock_time_max：从系统启动到现在等待最长的一次所花费的时间
- **Innodb_row_lock_waits**：系统启动到现在总共等待的次数

比较重要的是：

- Innodb_row_lock_time_avg（等待平均时长）
- Innodb_row_lock_waits（等待总次数）
- Innodb_row_lock_time（等待总时长）

尤其是当等待次数很高，而且每次等待时长也不小的时候，我们就需要分析系统中为什么会有如此多的等待，然后根据分析结果着手执行优化计划。

查看事务、锁的sql:

sql

代码解读

复制代码

`# 查看锁的SQL select * from information_schema.innodb_locks; select * from information_schema.innodb_lock_waits; # 查看事务SQL select * from information_schema.innodb_trx; # 查看未关闭的事务详情 SELECT a.trx_id,a.trx_state,a.trx_started,a.trx_query, b.ID,b.USER,b.DB,b.COMMAND,b.TIME,b.STATE,b.INFO, c.PROCESSLIST_USER,c.PROCESSLIST_HOST,c.PROCESSLIST_DB,d.SQL_TEXT FROM information_schema.INNODB_TRX a LEFT JOIN information_schema.PROCESSLIST b ON a.trx_mysql_thread_id = b.id AND b.COMMAND = 'Sleep' LEFT JOIN performance_schema.threads c ON b.id = c.PROCESSLIST_ID LEFT JOIN performance_schema.events_statements_current d ON d.THREAD_ID = c.THREAD_ID;`

# 5. 行锁分析实战

sql

代码解读

复制代码

`# 分析下下面两条简单的SQL，判断他们加的什么锁？ # SQL1 select * from t1 where id = 10; # SQL2 delete from t1 where id = 10;`

针对这个问题，我们通常能想到的答案是：

- SQL1: 不加锁，因为Mysql是多版本并发处理的，读不加锁。
- SQL2: 对ID=10的记录加写锁（走主键索引）

这个答案对吗？ 不一定，因为已知条件不足，这个问题没有答案。 判断这个问题，需要一些前提，前提不同，答案也不相同。

- **前提一**：id列是不是主键？

- **前提二**：当前系统的隔离级别是什么？

- **前提三**：ID列如果不是主键，那么ID列上有索引吗？

- **前提四**：ID列上如果有索引，那么这个索引是唯一索引吗？

- **前提五**：两个SQL的执行计划是什么？索引扫描？全表扫描？

- **读已提交【RC】隔离级别**

  - 组合一：id列是主键
  - 组合二：id列是二级唯一索引
  - 组合三：id列是二级非唯一索引
  - 组合四：id列上没有索引

- **可重复读【RR】隔离级别**

  - 组合五：id列是主键
  - 组合六：id列是二级唯一索引
  - 组合七：id列是二级非唯一索引
  - 组合八：id列上没有索引

- **组合九：Serializable隔离级别**

## 5.1 读已提交RC

> 前面8种组合下，也就是RC、RR的隔离级别下：SQL1 select操作都是不加锁的，采用的是快照读。 因此下面讨论的主要是SQL2 delete操作的加锁

### 5.1.1 组合一：ID是主键

这个组合最简单：ID是主键，RC隔离级别；给定SQL：`delete from t1 where id = 10;` 只需要将主键上ID = 10 的数据加上写锁就行了。

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8620ff3f7f874feb8935387c999ea98a~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.avis#?w=1018&h=734&s=74630&e=png&b=f9f9f9) **结论：RC隔离级别，ID是主键，此SQL只需要在id=10的记录上加写锁即可；**

### 5.1.2 组合二：ID唯一索引

这个组合，ID不是主键，而是一个Unique的二级索键值。那么在RC的隔离级别下，`delete from t1 where id = 10;`需要加什么锁呢？

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d30a5bc82c4e47ff9d3db8488305d70f~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.avis#?w=1100&h=702&s=96095&e=png&b=fafafa)

此组合中，id是unique索引，而主键是name列。此时，加锁的情况由于组合一有所不同。由于id是unique索引，因此delete语句会选择走id列的索引进行where条件的过滤，在找到id=10的记录后，首先会将unique索引上的id=10索引记录加上写锁，同时，会根据读取到的name列，回主键索引(聚簇索引)，然后将聚簇索引上的name = ‘d’ 对应的主键索引项加**写锁**。

**为什么聚簇索引上的记录也要加锁？** 试想一下，如果并发的一个SQL，是通过主键索引来更新：update t1 set id = 100 where name = ‘a’; 此时，如果delete语句没有将主键索引上的记录加锁，那么并发的update就会感知不到delete语句的存在，违背了同一记录上的更新/删除需要串行执行的约束。

**结论：若id列是unique列，其上有unique索引。那么SQL需要加两个写锁，一个对应于id unique索引上的id = 10的记录，另一把锁对应于聚簇索引上的【name=’d’,id=10】的记录。**

### 5.1.3 组合三：ID非唯一索引

相对于组合一、二，组合三又发生了变化，隔离级别仍旧是RC不变，但是id列上的约束又降低了，id列不再唯一，只有一个普通的索引。假设delete from t1 where id = 10; 语句，仍旧选择id列上的索引进行过滤where条件，那么此时会持有哪些锁？

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/952c8562e38d4b84946edbfad642e7c1~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.avis#?w=1014&h=1212&s=139084&e=png&b=f8f8f8)

根据此图，可以看到，首先，id列索引上，满足id = 10查询条件的记录，均已加锁。同时，这些记录对应的主键索引上的记录也都加上了锁。与组合二唯一的区别在于，组合二最多只有一个满足等值查询的记录，而组合三会将所有满足查询条件的记录都加锁。

**结论：若id列上有非唯一索引，那么对应的所有满足SQL查询条件的记录，都会被加锁。同时，这些记录在主键索引上的记录，也会被加锁。**

### 5.1.4 组合四：ID无索引

相对于前面三个组合，这是一个比较特殊的情况。id列上没有索引，where id = 10;这个过滤条件，没法通过索引进行过滤，那么只能走**全表扫描做过滤**

对应于这个组合，SQL会加什么锁？或者是换句话说，全表扫描时，会加什么锁？

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a743c4c44bcf452b9a82b6e037630042~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.avis#?w=1034&h=754&s=97220&e=png&b=f4f4f4)

**由于id列上没有索引，因此只能走聚簇索引，进行全部扫描。** 从图中可以看到，**满足删除条件的记录有两条，但是，聚簇索引上所有的记录，都被加上了写锁。无论记录是否满足条件，全部被加上写锁。** 既不是加表锁，也不是在满足条件的记录上加行锁。

有人可能会问？为什么不是只在满足条件的记录上加锁呢？ 这是由于MySQL的实现决定的。**如果一个条件无法通过索引快速过滤，那么存储引擎层面就会将所有记录加锁后返回，然后由MySQL Server层进行过滤。** 因此也就把所有的记录，都锁上了。

> 注：在实际的实现中，MySQL有一些改进，在MySQL Server过滤条件，发现不满足后，会调用 unlock_row方法，把不满足条件的记录放锁。这样做，保证了最后只会持有满足条件记录上的锁，但是每条记录的加锁操作还是不能省略的。

**结论：** **若id列上没有索引，SQL会走聚簇索引的全扫描进行过滤**，由于过滤是由MySQL Server层面进行的。**因此每条记录，无论是否满足条件，都会被加上写锁。** 但是，为了效率考量，**MySQL做了优化，对于不满足条件的记录，会在判断后放锁，最终持有的，是满足条件的记录上的锁，但是不满足条件的记录上的加锁/放锁动作不会省略。**

## 5.2 可重复读RR

### 5.2.1 组合五：ID主键

**结论：与组合一是一致的。ID=10的数据加写锁**

### 5.2.2 组合六：ID唯一索引

**结论：与组合二是一致的。ID=10的数据加写锁，聚簇索引上ID=10的数据也加写锁，两把锁。**

### 5.2.3 组合七：ID非唯一索引

**RC隔离级别允许幻读，而RR隔离级别，不允许存在幻读**

那么RR隔离级别下，如何防止幻读呢？ - **间隙锁**

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ce27b670edb74526916cbec50610612f~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.avis#?w=1074&h=706&s=281948&e=png&b=faf6f5)

相对于组合三最大的区别在于，组合七中多了一个间隙锁。**其实这个多出来的间隙锁，就是RR隔离级别，相对于RC隔离级别，不会出现幻读的关键。**

> 所谓幻读，就是同一个事务，连续做两次当前读 (例如：select * from t1 where id = 10 for update;)，那么这两次当前读返回的是完全相同的记录 (记录数量一致，记录本身也一致)，第二次的当前读，不会比第一次返回更多的记录 (幻象)。记录本身的一致性是可重复性，使用MVCC来解决。

> **如何保证两次当前读返回一致的记录？** 那就需要在第一次当前读与第二次当前读之间，其他的事务不会插入新的满足条件的记录并提交。为了实现这个功能，间隙锁应运而生。

**结论**：RR隔离级别下，ID列上有一个非唯一索引，对应SQL：`delete from t1 where id = 10;` 首先：**通过ID索引定位到第一条满足查询条件的记录，对该记录上写锁，加GAP的间隙锁，然后加主键索引上记录的写锁，然后返回；然后读取吓一跳，重复进行。直到进行到第一条不满足条件的记录\[11,f\]，此时，不需要加记录写锁，但是仍旧需要加间隙锁，最后返回结束。**

### 5.2.4 组合八：ID无索引

RR情况下，ID无索引：`delete from t1 where id = 10;` 只能走全表扫描

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/18b714a340ab4908a6b9fc66ca5782c9~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.avis#?w=1072&h=1002&s=191116&e=png&b=f6f6f6)

如图，**聚簇索引上的所有记录都被加上了写锁**，其次，\*\*聚簇索引每条记录的间隙，也同时被加上了间隙锁。\*\*这个实例表，总共6条数据，一共需要6个记录锁，7个间隙锁。那么如果是1000万条记录呢？

在这种情况下，这个表上，除了不加锁的快照读，其他任何加锁的并发SQL都不能被执行，不能更新，不能删除，不能插入，全表被锁死。

当前，跟组合四类似，这种情况下，Mysql也做了优化，就是所谓的semi-consistent read。semi-consistent read开启的情况下，对于不满足查询条件的记录，Mysql会提前放锁。针对上面的这个用例，就是除了记录\[d,10\],\[g,10\]之外，其他的记录锁都会被释放，同时不加间隙锁。

**semi-consistent read如何触发？** 要么是RC隔离级别；要么是RR隔离级别，同时设置了**innodb_locks_unsafe_for_binlog 参数。**

结论：在RR隔离级别下，如果进行全表扫描的当前读，会锁上表中所有的记录，同时锁上聚簇索引内的所有间隙，杜绝所有的并发更新、删除、插入操作。当前，也可以通过触发semi-consistent read，来缓解加锁开销与并发影响。但是semi-consistent read本身也会带来其他问题，不建议使用。（可能会造成主从数据库数据的不一致）

### 5.3 组合九：串形化Serializable

对于SQL2来说，Serializable隔离级别与RR隔离级别组合八情况完全一致，因此不做介绍。 `delete from t1 where id = 10;`

Serializable隔离级别，影响的是SQL1这条SQL： `select * from t1 where id = 10;`

在RC，RR隔离级别下，都是快照读，不加锁。但是在Serializable隔离级别，SQL1会加读锁，也就是说快照读不复存在，**MVCC并发控制降级为LBCC。**

结论： 在MySQL/InnoDB中，所谓的读不加锁，并不适用于所有的情况，而是隔离级别相关的。Serializable隔离级别，读不加锁就不再成立，所有的读操作，都是当前读。

## 5.4 复杂SQL加锁分析

sql

代码解读

复制代码

`delete from t1 where pubtime > 1 and pubtime < 20 and userid='hero' and commit is not null;`

索引如图所示：

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d0e1450b94064e1fb5d6ef538cb09fc6~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.avis#?w=1086&h=720&s=274848&e=png&b=f7f6f6)

假设在RR的隔离级别下，同时假设SQL走的idx_t1_pu索引

在详细分析这条SQL的加锁情况前，还需要一个知识储备，那就是一个SQL中的where条件如何拆分？

- **index key**: pubtime > 1 and pubtime \< 20。此条件，用于确定SQL在idx_t1_pu索引上的查询范围。
- **index Filter**: userid = 'hero'. 此条件，可以在idx_t1_pu索引上进行过滤，但不属于index key。
- **Table Filter**：comment is not null。此条件，在idx_t1_pu索引上无法过滤，只能在SQL- Layer上过滤。

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4b420060893b4758975c8ba29359c829~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.avis#?w=1090&h=636&s=284346&e=png&b=f9f7f7)

从图中可以看出，在RR隔离级别下，有index key所确定的范围被加上了间隙锁；Index Filter锁给定的条件视Mysql版本而定【图中红色箭头标出的写锁是否要加，跟ICP有关】

- 不支持ICP：Index Filter在Mysql Server层过滤，不满足Index Filter的记录，也要加上记录写锁
- 支持ICP：则在Index上过滤，则不满足Index Filter的记录，无需加记录写锁。

而Table Filter对应的过滤条件，则在聚簇索引中读取后，在Mysql Server层过滤，因此聚簇索引上也需要写锁。最后，选取出了一条满足条件的记录\[8, hero, d, 5, handsome\]，但是加锁的数量，要远远大于满足条件的记录数量。

**结论：**

在RR隔离级别下，针对一个复杂SQL，首先需要提取其where条件

- **index key确定的范围，需要加上间隙锁**
- **index Filter过滤条件，视是否支持ICP，若支持ICP，则不满足Index Filter的记录，不加写锁，否则需要写锁**
- **Table Filter过滤条件：无论是否满足，都需要加写锁。**

# 6. 死锁原理

深入理解Mysql如何加锁，有几个比较重要的作用：

- 可以根据Mysql的加锁规则，写出不会发生死锁的SQL
- 可以根据Mysql的加锁规则，定位出线上产生死锁的原因
- 可以根据Mysql的加锁规则，透过现象看本质，理解数据库层面阻塞执行的根本原因

## 6.1 什么是死锁？

情况1:

sql

代码解读

复制代码

`` CREATE TABLE `t1_deadlock` ( `id` int(11) NOT NULL, `name` varchar(100) DEFAULT NULL, `age` int(11) NOT NULL, `address` varchar(255) DEFAULT NULL, PRIMARY KEY (`id`), KEY `idx_age` (`age`) USING BTREE, KEY `idx_name` (`name`) USING BTREE ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4; Insert into t1_deadlock(id,name,age,address) values (1,'刘备',18,'蜀国'); Insert into t1_deadlock(id,name,age,address) values (2,'关羽',17,'蜀国'); Insert into t1_deadlock(id,name,age,address) values (3,'张飞',16,'蜀国'); Insert into t1_deadlock(id,name,age,address) values (4,'关羽',16,'蜀国'); Insert into t1_deadlock(id,name,age,address) values (5,'诸葛亮',35,'蜀国'); Insert into t1_deadlock(id,name,age,address) values (6,'曹孟德',32,'魏国'); ``

产生死锁的两个例子：

- 一个是两个session的两条SQL产生死锁
- 一个是两个session的一条SQL产生死锁

|时刻|session01|session02|
|---|---|---|
|T1|begin;|begin;|
|T2|select * from t1_deadlock where id=1 for update;||
|T3||delete from t1_deadlock where id = 5;|
|T4|update t1_deadlock set name='qqq' where id =5||
|T5|死锁|delete from t1_deadlock where id = 1;|
|T6|commit;|commit;|

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/346bd688f4814fe6aaea5edec654a731~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.avis#?w=834&h=494&s=174683&e=png&b=fcfcfc)

sql

代码解读

复制代码

`-- Session01 begin; select * from t1_deadlock where id=1 for update;	 update t1_deadlock set name='qqq' where id=5; commit; -- Session02 begin; delete from t1_deadlock where id=5; delete from t1_deadlock where id=1; -- 死锁 commit;`

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/fb9e5b5d0c754c98ab23363be0c14847~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.avis#?w=1246&h=450&s=548185&e=png&b=2c4e62)

情况02:

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1ca3ace559854033b9ec36b4eb45e52f~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.avis#?w=740&h=536&s=169116&e=png&b=fbf7f6)

sql

代码解读

复制代码

`` CREATE TABLE `t1_deadlock03` ( `id` int(11) NOT NULL AUTO_INCREMENT, `cnt` varchar(32) DEFAULT NULL, PRIMARY KEY (`id`), UNIQUE index `idx_cnt` (`cnt`) USING BTREE ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4; insert into t1_deadlock03(id,cnt) values (1,'abc-130-sz'); -- Session01 begin; delete from t1_deadlock03 where cnt='abc-130-sz'; insert into t1_deadlock03(cnt) values ('abc-130-sz'); -- 在加写锁之前会先加读锁 commit; -- Session01 begin; delete from t1_deadlock03 where cnt='abc-130-sz'; commit; ``

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5a4efd56c23543e691377d13212d846b~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.avis#?w=1290&h=264&s=377295&e=png&b=1e3648)

**结论：**

死锁的发生与否，并不在于事务中有多少条SQL，**【死锁的关键在于】** ：两个（或以上）的session【加锁的顺序不一致】

查询最近一次死锁日志：`show engine innodb status`

perl

代码解读

复制代码

`` ------------------------ LATEST DETECTED DEADLOCK ------------------------ 2024-05-14 10:49:51 0x7f8c2c06f700 *** (1) TRANSACTION: TRANSACTION 2428, ACTIVE 4 sec starting index read mysql tables in use 1, locked 1 LOCK WAIT 2 lock struct(s), heap size 1136, 1 row lock(s) MySQL thread id 207, OS thread handle 140239876904704, query id 721 localhost root updating delete from t1_deadlock03 where cnt='abc-130-sz' *** (1) WAITING FOR THIS LOCK TO BE GRANTED: RECORD LOCKS space id 44 page no 4 n bits 72 index idx_cnt of table `hello`.`t1_deadlock03` trx id 2428 lock_mode X waiting Record lock, heap no 2 PHYSICAL RECORD: n_fields 2; compact format; info bits 32  0: len 10; hex 6162632d3133302d737a; asc abc-130-sz;;  1: len 4; hex 80000004; asc     ;; *** (2) TRANSACTION: TRANSACTION 2427, ACTIVE 8 sec inserting mysql tables in use 1, locked 1 4 lock struct(s), heap size 1136, 3 row lock(s), undo log entries 2 MySQL thread id 208, OS thread handle 140240010802944, query id 722 localhost root update insert into t1_deadlock03(cnt) values ('abc-130-sz') *** (2) HOLDS THE LOCK(S): RECORD LOCKS space id 44 page no 4 n bits 72 index idx_cnt of table `hello`.`t1_deadlock03` trx id 2427 lock_mode X locks rec but not gap Record lock, heap no 2 PHYSICAL RECORD: n_fields 2; compact format; info bits 32  0: len 10; hex 6162632d3133302d737a; asc abc-130-sz;;  1: len 4; hex 80000004; asc     ;; *** (2) WAITING FOR THIS LOCK TO BE GRANTED: RECORD LOCKS space id 44 page no 4 n bits 72 index idx_cnt of table `hello`.`t1_deadlock03` trx id 2427 lock mode S waiting Record lock, heap no 2 PHYSICAL RECORD: n_fields 2; compact format; info bits 32  0: len 10; hex 6162632d3133302d737a; asc abc-130-sz;;  1: len 4; hex 80000004; asc     ;; *** WE ROLL BACK TRANSACTION (1) ``

## 6.2 如何避免死锁？

Mysql默认会主动探知死锁，并回滚某一个影响最小的事务，等另一个事务执行完成之后，再重新执行该事务。

1. 注意程序的逻辑：根本的原因是程序逻辑的顺序交叠，最常见的是交叉更新
1. 保持事务的轻量：越是轻量的事务，占有越少的资源，这样发生死锁的几率越小
1. 提高运行的速度：避免使用子查询，尽量使用主键等
1. 尽量快提交事务，减少持有锁的时间：越早提交事务，锁就越早释放

作者：Aaron学习之路\
链接：https://juejin.cn/post/7368375416929239092\
来源：稀土掘金\
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。
