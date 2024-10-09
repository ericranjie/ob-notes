# 

码哥字节 架构师社区

_2021年11月17日 11:33_

Redis 分布式锁这个话题似乎烂大街了，不管你是面试还是工作，随处可见，「码哥」为啥还写？

因为看过很多文章没有将分布式锁的各种问题讲明白，所以准备写一篇，也当做自己的学习总结。

在进入正文之前，我们先带着问题去思考：

- 什么时候需要分布式锁？

- 加、解锁的代码位置有讲究么？

- 如何避免出现死锁

- 超时时间设置多少合适呢？

- 如何避免锁被其他线程释放

- 如何实现重入锁？

- 主从架构会带来什么安全问题？

- 什么是 `Redlock`

- ……

# 什么时候用分布式锁？

> ❝
>
> 码哥，说个通俗的例子讲解下什么时候需要分布式锁呢？

**精子喷射那一刻，亿级流量冲向卵子，只有一个精子能获得与卵子结合的幸运。**

造物主为了保证只有一个「精子」能获得「卵子」的宠幸，当有一个精子进入后，卵子的外壳就会发生变化，将通道关闭把其余的精子阻挡在外。

- 亿级别的精子就好比「并发」流量；

- 卵子就好比是共享资源；

- 卵子外壳只允许一个精子进入的特殊蛋白就是一把锁。

而多节点构成的集群，就会有多个 JVM 进程，我们获得同样的效果就需要有一个中间人协调，只允许一个 JVM 中的一个线程获得操作共享资源的资格。

**分布式锁就是用来控制同一时刻，只有一个 JVM 进程中的一个线程「精子」可以访问被保护的资源「卵子」。**

!\[\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

「每一个生命，都是亿级选手中的佼佼者」，加油。

# 分布式锁入门

> ❝
>
> 分布式锁应该满足哪些特性？

1. 互斥：在任何给定时刻，只有一个客户端可以持有锁；

1. 无死锁：任何时刻都有可能获得锁，即使获取锁的客户端崩溃；

1. 容错：只要大多数 `Redis`的节点都已经启动，客户端就可以获取和释放锁。

> ❝
>
> 码哥，我可以使用 `SETNX key value` 命令是实现「互斥」特性。

这个命令来自于`SET if Not eXists`的缩写，意思是：如果 `key` 不存在，则设置 `value` 给这个`key`，否则啥都不做。

命令的返回值：

- 1：设置成功；

- 0：key 没有设置成功。

如下场景：

敲代码一天累了，想去放松按摩下肩颈。

168 号技师最抢手，大家喜欢点，所以并发量大，需要分布式锁控制。

同一时刻只允许一个「客户」预约 168 技师。

肖彩机申请 168 技师成功：

`> SETNX lock:168 1   (integer) 1 # 获取 168 技师成功   `

谢霸哥后面到，申请失败：

`> SETNX lock 2   (integer) 0 # 客户谢霸哥 2 获取失败   `

此刻，申请成功的客户就可以享受 168 技师的肩颈放松服务「共享资源」。

享受结束后，要及时释放锁，给后来者享受 168 技师的服务机会。

> ❝
>
> 肖彩机，码哥考考你如何释放锁呢？

很简单，使用 `DEL` 删除这个 `key` 就行。

`> DEL lock:168   (integer) 1   `

> ❝
>
> 码哥，你见过「龙」么？我见过，因为我被一条龙服务过。

!\[\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

肖彩机，事情可没这么简单。

这个方案存在一个存在造成锁无法释放的问题，造成该问题的场景如下：

1. 在按摩过程中突然收到线上报警，提起裤子就跑去公司了，没及时执行 `DEL` 释放锁（客户端处理业务异常，无法正确释放锁）；

1. 按摩过程中心肌梗塞嗝屁了，无法执行 `DEL`指令。

这样，这个锁就会一直占用，锁在我手里，我挂了，这样其他客户端再也拿不到这个锁了。

# 异常导致没有删除锁

> ❝
>
> 码哥，我可以在获取锁成功的时候设置一个「超时时间」

比如设定按摩服务一次 60 分钟，那么在给这个 `key` 加锁的时候设置 60 分钟过期即可：

`> SETNX lock:168 1  // 获取锁   (integer) 1   > EXPIRE lock:168 60  // 60s 自动删除   (integer) 1   `

这样，到点后锁自动释放，其他客户就可以继续享受 168 技师按摩服务了。

> ❝
>
> 谁要这么写，就糟透了。

「加锁」、「设置超时」是两个命令，他们不是原子操作。

**如果出现只执行了第一条，第二条没机会执行就会出现「超时时间」设置失败，依然出现死锁。**

> ❝
>
> 码哥，那咋办，我想被一条龙服务，不能出现死锁啊。

Redis 2.6.X 之后，官方拓展了 `SET` 命令的参数，满足了当 key 不存在则设置 value，同时设置超时时间的语义，并且满足原子性。

`SET resource_name random_value NX PX 30000   `

- NX：表示只有 `resource_name` 不存在的时候才能 `SET` 成功，从而保证只有一个客户端可以获得锁；

- PX 30000：表示这个锁有一个 30 秒自动过期时间。

## 执行时间超过锁的过期时间

> ❝
>
> 这样我能稳妥的享受一条龙服务了么？

No，还有一种场景会导致**释放别人的锁**：

1. 客户 1 获取锁成功并设置设置 30 秒超时；

1. 客户 1 因为一些原因导致执行很慢（网络问题、发生 FullGC……），过了 30 秒依然没执行完，但是锁过期「自动释放了」；

1. 客户 2 申请加锁成功；

1. 客户 1 执行完成，执行 `DEL` 释放锁指令，这个时候就把 客户 2 的锁给释放了。

有两个关键问题需要解决：

1. 如何合理设置过期时间？

1. 如何避免删除别人持有的锁。

# 正确设置锁超时

> ❝
>
> 锁的超时时间怎么计算合适呢？

这个时间不能瞎写，一般要根据在测试环境多次测试，然后压测多轮之后，比如计算出平均执行时间 200 ms。

那么锁的**超时时间就放大为平均执行时间的 3~5 倍。**

> ❝
>
> 为啥要放放大呢？

因为如果锁的操作逻辑中有网络 IO操作、JVM FullGC 等，线上的网络不会总一帆风顺，我们要给网络抖动留有缓冲时间。

> ❝
>
> 那我设置更大一点，比如设置 1 小时不是更安全？

不要钻牛角，多大算大？

设置时间过长，一旦发生宕机重启，就意味着 1 小时内，分布式锁的服务全部节点不可用。

你要让运维手动删除这个锁么？

只要运维真的不会打你。

> ❝
>
> 有没有完美的方案呢？不管时间怎么设置都不大合适。

我们可以让获得锁的线程开启一个**守护线程**，用来给快要过期的锁「续航」。

加锁的时候设置一个过期时间，同时客户端开启一个「守护线程」，定时去检测这个锁的失效时间。

如果快要过期，但是业务逻辑还没执行完成，自动对这个锁进行续期，重新设置过期时间。

> ❝
>
> 这个道理行得通，可我写不出。

别慌，已经有一个库把这些工作都封装好了他叫**Redisson**。

在使用分布式锁时，它就采用了「自动续期」的方案来避免锁过期，这个守护线程我们一般也把它叫做「看门狗」线程。

关于 Redisson 的使用与原理分析由于篇幅有限，大家可关注「码哥字节」且听下回分解。

# 避免释放别人的锁

> ❝
>
> 我要如何删除是自己加的锁呢？

出现释放别人锁的关键在于直接执行`DEL`指令，所以我们要想办法检查下这个锁是不是自己加的锁再执行删除指令。

**解铃还须系铃人**

> ❝
>
> 码哥，我在加锁的时候设置一个「唯一标识」作为 `value` 代表加锁的客户端。
>
> 在释放锁的时候，客户端将自己的「唯一标识」比如可以使用随机数作为唯一标识，与锁上的「标识」比较是否相等，匹配上则删除，否则没有权利释放锁。

伪代码如下：

`// 比对 value 与 唯一标识   if (redis.get("lock:168").equals(uuid)){      redis.del("lock:168"); //比对成功则删除    }   `

> ❝
>
> 有没有想过，这是 `GET + DEL` 指令组合而成的，这里又会涉及到原子性问题。

我们可以通过 `Lua` 脚本来实现，这样判断和删除的过程就是原子操作了。

`if redis.call("get",KEYS[1]) == ARGV[1] then       return redis.call("del",KEYS[1])   else       return 0   end   `

> ❝
>
> 一路优化下来，方案似乎比较「严谨」了，抽象出对应的模型如下。

1. 通过 `SET lock_resource_name $uuid NX PX $expire_time`，同时启动守护线程为快要过期单还没执行完毕的客户端的锁续命;

1. 客户端执行业务逻辑操作共享资源；

1. 通过 `Lua` 脚本释放锁，先 get 判断锁是否是自己加的，再执行 `DEL`。

!\[\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

# 加解锁代码位置有讲究

根据前面的分析，我们已经有了一个「相对严谨」的分布式锁了。

于是「谢霸哥」就写了如下代码将分布式锁运用到项目中，以下是伪代码逻辑：

`public void doSomething() {     redisLock.lock(); // 上锁       try {           // 处理业务           redisLock.unlock(); // 释放锁       } catch (Exception e) {           e.printStackTrace();       }   }   `

> ❝
>
> 一旦执行业务逻辑过程中抛出异常，程序就无法走下一步释放锁的流程。

**所以释放锁的代码一定要放在 `finally{}` 块中。**

加锁的位置也有问题，放在 try 外面的话，如果执行 `redisLock.lock()` 加锁异常，但是实际指令已经发送到服务端并执行，只是客户端读取响应超时，就会导致没有机会执行解锁的代码。

所以 `redisLock.lock()` 应该写在 try 代码块，这样保证一定会执行解锁逻辑。

综上所述，正确代码位置如下 ：

`public void doSomething() {      // 上锁      redisLock.lock();       try {           // 处理业务           ...       } catch (Exception e) {           e.printStackTrace();       } finally {         // 释放锁         redisLock.unlock();        }   }   `

# 实现可重入锁

> ❝
>
> 可重入锁要如何实现呢？重入之后，超时时间如何设置呢？

当一个线程执行一段代码成功获取锁之后，继续执行时，又遇到加锁的代码，可重入性就就保证线程能继续执行，而不可重入就是需要等待锁释放之后，再次获取锁成功，才能继续往下执行。

用一段代码解释可重入：

`public synchronized void a() {       b();   }   public synchronized void b() {       // pass   }   `

假设 X 线程在 a 方法获取锁之后，继续执行 b 方法，如果此时**不可重入**，线程就必须等待锁释放，再次争抢锁。

锁明明是被 X 线程拥有，却还需要等待自己释放锁，然后再去抢锁，这看起来就很奇怪，我释放我自己~

## Redis Hash 可重入锁

> ❝
>
> Redisson 类库就是通过 Redis Hash 来实现可重入锁，未来码哥会专门写一篇关于 Redisson 的使用与原理的文章……

当线程拥有锁之后，往后再遇到加锁方法，直接将加锁次数加 1，然后再执行方法逻辑。

退出加锁方法之后，加锁次数再减 1，当加锁次数为 0 时，锁才被真正的释放。

可以看到可重入锁最大特性就是计数，计算加锁的次数。

所以当可重入锁需要在分布式环境实现时，我们也就需要统计加锁次数。

### 加锁逻辑

> ❝
>
> 我们可以使用 Redis hash 结构实现，key 表示被锁的共享资源， hash 结构的 fieldKey 的 value 则保存加锁的次数。

!\[\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

通过 Lua 脚本实现原子性，假设 KEYS1 = 「lock」, ARGV「1000，uuid」：

`---- 1 代表 true   ---- 0 代表 false      if (redis.call('exists', KEYS[1]) == 0) then       redis.call('hincrby', KEYS[1], ARGV[2], 1);       redis.call('pexpire', KEYS[1], ARGV[1]);       return 1;   end ;   if (redis.call('hexists', KEYS[1], ARGV[2]) == 1) then       redis.call('hincrby', KEYS[1], ARGV[2], 1);       redis.call('pexpire', KEYS[1], ARGV[1]);       return 1;   end ;   return 0;   `

加锁代码首先使用 Redis `exists` 命令判断当前 lock 这个锁是否存在。

如果锁不存在的话，直接使用 `hincrby`创建一个键为 `lock` hash 表，并且为 Hash 表中键为 `uuid` 初始化为 0，然后再次加 1，最后再设置过期时间。

如果当前锁存在，则使用 `hexists`判断当前 `lock` 对应的 hash 表中是否存在 `uuid` 这个键，如果存在，再次使用 `hincrby` 加 1，最后再次设置过期时间。

最后如果上述两个逻辑都不符合，直接返回。

### 解锁逻辑

`-- 判断 hash set 可重入 key 的值是否等于 0   -- 如果为 0 代表 该可重入 key 不存在   if (redis.call('hexists', KEYS[1], ARGV[1]) == 0) then       return nil;   end ;   -- 计算当前可重入次数   local counter = redis.call('hincrby', KEYS[1], ARGV[1], -1);   -- 小于等于 0 代表可以解锁   if (counter > 0) then       return 0;   else       redis.call('del', KEYS[1]);       return 1;   end ;   return nil;   `

首先使用 `hexists` 判断 Redis Hash 表是否存给定的域。

如果 lock 对应 Hash 表不存在，或者 Hash 表不存在 uuid 这个 key，直接返回 `nil`。

若存在的情况下，代表当前锁被其持有，首先使用 `hincrby`使可重入次数减 1 ，然后判断计算之后可重入次数，若小于等于 0，则使用 `del` 删除这把锁。

解锁代码执行方式与加锁类似，只不过解锁的执行结果返回类型使用 `Long`。这里之所以没有跟加锁一样使用 `Boolean` ,这是因为解锁 lua 脚本中，三个返回值含义如下：

- 1 代表解锁成功，锁被释放

- 0 代表可重入次数被减 1

- `null` 代表其他线程尝试解锁，解锁失败.

# 主从架构带来的问题

> ❝
>
> 码哥，到这里分布式锁「很完美了」吧，没想到分布式锁这么多门道。

路还很远，之前分析的场景都是，锁在「单个」Redis 实例中可能产生的问题，并没有涉及到 Redis 的部署架构细节。

我们通常使用「Cluster 集群」或者「哨兵集群」的模式部署保证高可用。

这两个模式都是基于「主从架构数据同步复制」实现的数据同步，而 Redis 的主从复制默认是异步的。

我们试想下如下场景会发生什么问题：

1. 如果客户端 1 刚往 master 节点写入一个分布式锁，此时这个指令还没来得及同步到 slave 节点。

1. 此时，master 节点宕机，其中一个 slave 被选举为新 master，这时候新 master 是没有客户端 1 写入的锁，锁丢失了。

1. 此刻，客户端 2 线程来获取锁，就成功了。

虽然这个概率极低，但是我们必须得承认这个风险的存在。

> ❝
>
> Redis 的作者提出了一种解决方案，叫 Redlock（红锁）

Redis 的作者为了统一分布式锁的标准，搞了一个 Redlock，算是 Redis 官方对于实现分布式锁的指导规范，https://redis.io/topics/distlock，但是这个 Redlock 也被国外的一些分布式专家给喷了。

因为它也不完美，有“漏洞”。

# 什么是 Redlock

红锁是不是这个？

!\[\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

泡面吃多了你，`Redlock` 红锁是为了解决主从架构中当出现主从切换导致锁丢失而提出的一种算法。

大家可以看官方文档（https://redis.io/topics/distlock），以下来自官方文档的翻译。

想用使用 Redlock，官方建议部署 5 个 Redis 实例，它们之间没有任何关系，都是**一个个孤立的实例**。

**另外部署实例的数量要求是奇数。**

**一个客户端要获取锁有 5 个步骤**：

1. 客户端获取当前时间 `T1`（毫秒级别）；

1. 使用相同的 `key`和 `value`顺序尝试从 `N`个 `Redis`实例上获取锁。

- 每个请求都设置一个超时时间（毫秒级别），该超时时间要远小于锁的有效时间，这样便于快速尝试与下一个实例发送请求。

- 比如锁的自动释放时间 `10s`，则请求的超时时间可以设置 `5~50` 毫秒内，这样可以防止客户端长时间阻塞。

4. 客户端获取当前时间 `T2` 并减去步骤 1 的 `T1` 来计算出获取锁所用的时间（`T3 = T2 -T1`）。**当且仅当客户端在大多数实例（`N/2 + 1`）获取成功，且获取锁所用的总时间 T3 小于锁的有效时间，才认为加锁成功，否则加锁失败。**

1. 如果第 3 步加锁成功，则执行业务逻辑操作共享资源，**key 的真正有效时间等于有效时间减去获取锁所使用的时间（步骤 3 计算的结果）。**

1. 如果因为某些原因，获取锁失败（没有在至少 N/2+1 个Redis实例取到锁或者取锁时间已经超过了有效时间），**客户端应该在所有的 Redis 实例上进行解锁**（即便某些 Redis 实例根本就没有加锁成功）。

> ❝
>
> 事情可没这么简单，Redis 作者把这个方案提出后，受到了业界著名的分布式系统专家的**质疑**。

两人好比神仙打架，两人一来一回论据充足的对一个问题提出很多论断……

由于篇幅原因，关于 两人的争论分析以及 `Redssion` 对分布式锁的封装以及 `Redlock` 的实现我们下期再见。

预知后事如何，且听下回分解…

![](https://res.wx.qq.com/t/fed_upload/b39ef69e-c4d6-4169-8612-5f00a84860e7/wx-avatar-default.svg)

公众号

阅读 1307

​
