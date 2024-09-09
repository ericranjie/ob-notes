# 

程序员解析

 _2021年10月14日 16:00_

公众号关注 “程序员解析”

设为“星标”，重磅干货，第一时间送达

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/ow6przZuPIENb0m5iawutIf90N2Ub3dcPuP2KXHJvaR1Fv2FnicTuOy3KcHuIEJbd9lUyOibeXqW8tEhoJGL98qOw/640?wx_fmt=jpeg&wxfrom=13&tp=wxpic)定义

场景：一家网上商城做商品限量秒杀。

## 1 单机环境下的锁

将商品的数量存到Redis中。每个用户抢购前都需要到Redis中查询商品数量（代替mysql数据库。不考虑事务），如果商品数量大于0，则证明商品有库存。然后我们在进行库存扣减和接下来的操作。

因为多线程并发问题，我们不得不在get()方法内部使用同步代码块。这样可以保证查询库存和减库存操作的原子性。

```java
package springbootdemo.demo.controller;/* * @auther 顶风少年 * @mail dfsn19970313@foxmail.com * @date 2020-01-13 11:19 * @notify * @version 1.0 */import org.springframework.beans.factory.annotation.Autowired;import org.springframework.data.redis.core.RedisTemplate;import org.springframework.web.bind.annotation.GetMapping;import org.springframework.web.bind.annotation.RestController;@RestControllerpublic class RedisLock  {    @Autowired    private RedisTemplate<String, String> redisTemplate;    @GetMapping(value = "buy")    public String get() {        synchronized (this) {            String phone = redisTemplate.opsForValue().get("phone");            Integer count = Integer.valueOf(phone);            if (count > 0) {                redisTemplate.opsForValue().set("phone", String.valueOf(count - 1));                System.out.println("抢到了" + count + "号商品");            }return "";        }    }}
```

## 2 分布式情况下使用Redis锁。

但是由于业务上升，并发数量变大。公司不得不将原有系统复制一份，放到新的服务器。然后使用nginx做负载均衡。为了模拟高并发环境这里使用了 Apache JMeter工具。

很明显，现在的线程锁不管用了。于是我们需要换一把锁，这把锁必须和两套系统没有任何的耦合度。

使用Redies的API如果key不存在，则设置一个key。这个key就是我们现在使用的一把锁。每个线程到此处，先设置锁，如果设置锁失败，则表明当前有线程获取到了锁，就返回。

最后我们为了减库存和其他业务抛出异常，而没有释放锁。把释放锁的操作放到了finally代码块中。看起来是比较完美了。

```java
package springbootdemo.demo.controller;/* * @auther 顶风少年 * @mail dfsn19970313@foxmail.com * @date 2020-01-13 11:19 * @notify * @version 1.0 */import org.springframework.beans.factory.annotation.Autowired;import org.springframework.data.redis.core.RedisTemplate;import org.springframework.web.bind.annotation.GetMapping;import org.springframework.web.bind.annotation.RestController;@RestControllerpublic class RedisLock {    @Autowired    private RedisTemplate<String, String> redisTemplate;    @GetMapping(value = "buy")    public String get() {        Boolean phoneLock = redisTemplate.opsForValue().setIfAbsent("phoneLock", "");        if (!phoneLock) {            return "";        }        try{            String phone = redisTemplate.opsForValue().get("phone");            Integer count = Integer.valueOf(phone);            if (count > 0) {                redisTemplate.opsForValue().set("phone", String.valueOf(count - 1));                System.out.println("抢到了" + count + "号商品");            }        }finally {            redisTemplate.delete("phoneLock");        }        return "";    }}
```

## 3 一台服务宕机，导致无法释放锁

如果try中抛出了异常，进入finally,这把锁依然会释放，不会影响其他线程获取锁，那么如果在finally也抛出了异常，或者在finally中服务直接关闭了，那其他的服务再也获取不到锁。最终导致商品卖不出去。

```java
package springbootdemo.demo.controller;/* * @auther 顶风少年 * @mail dfsn19970313@foxmail.com * @date 2020-01-13 11:19 * @notify * @version 1.0 */import org.springframework.beans.factory.annotation.Autowired;import org.springframework.data.redis.core.RedisTemplate;import org.springframework.web.bind.annotation.GetMapping;import org.springframework.web.bind.annotation.RestController;@RestControllerpublic class RedisLock {    @Autowired    private RedisTemplate<String, String> redisTemplate;    @GetMapping(value = "buy")    public String get() {        int i = 0;        Boolean phoneLock = redisTemplate.opsForValue().setIfAbsent("phoneLock", "");        if (!phoneLock) {            return "";        }        try {            String phone = redisTemplate.opsForValue().get("phone");            Integer count = Integer.valueOf(phone);            if (count > 0) {                i = count;                redisTemplate.opsForValue().set("phone", String.valueOf(count - 1));                System.out.println("抢到了" + count + "号商品");            }        } finally {            if (i == 20) {                System.exit(0);            }            redisTemplate.delete("phoneLock");        }        return "";    }}
```

## 4 给每一把锁加上过期时间

问题就出现在如果出现意外，这把锁无法释放。这里我们在引入Redis的API，对key进行过期时间的设置。这样如果拿到锁的线程，在任何情况下没有来得及释放锁，当Redis的key时间到，也会自动释放锁。但是这样还是存在问题

如果在key过期后，锁释放了，但是当前线程没有执行完毕。那么其他线程就会拿到锁，继续抢购商品，而这个较慢的线程则会在执行完毕后，释放别人的锁。导致锁失效!

```java
package springbootdemo.demo.controller;/* * @auther 顶风少年 * @mail dfsn19970313@foxmail.com * @date 2020-01-13 11:19 * @notify * @version 1.0 */import javafx.concurrent.Task;import org.springframework.beans.factory.annotation.Autowired;import org.springframework.data.redis.core.RedisTemplate;import org.springframework.web.bind.annotation.GetMapping;import org.springframework.web.bind.annotation.RestController;import java.util.Timer;import java.util.TimerTask;import java.util.concurrent.TimeUnit;@RestControllerpublic class RedisLock {    @Autowired    private RedisTemplate<String, String> redisTemplate;    @GetMapping(value = "buy")    public String get() {        Boolean phoneLock = redisTemplate.opsForValue().setIfAbsent("phoneLock", "", 3, TimeUnit.SECONDS);        if (!phoneLock) {            return "";        }        try {            String phone = redisTemplate.opsForValue().get("phone");            Integer count = Integer.valueOf(phone);            if (count > 0) {                try {                    Thread.sleep(99999999999L);                } catch (Exception e) {                }                redisTemplate.opsForValue().set("phone", String.valueOf(count - 1));                System.out.println("抢到了" + count + "号商品");            }        } finally {                      redisTemplate.delete("phoneLock");        }        return "";    }}
```

## 5 延长锁的过期时间，解决锁失效

问题的出现就是，当一条线程的key已经过期，但是这个线程的任务确确实实没有执行完毕，这个交易没有结束。但是锁没了。现在我们必须对锁的时间进行延长。在判断商品有库存时，第一时间创建一个线程不停的给key续命

防止key过期。然后在交易结束后，停止定时器，释放锁。

```java
package springbootdemo.demo.controller;/* * @auther 顶风少年 * @mail dfsn19970313@foxmail.com * @date 2020-01-13 11:19 * @notify * @version 1.0 */import org.springframework.beans.factory.annotation.Autowired;import org.springframework.data.redis.core.RedisTemplate;import org.springframework.web.bind.annotation.GetMapping;import org.springframework.web.bind.annotation.RestController;import java.util.Timer;import java.util.TimerTask;import java.util.concurrent.TimeUnit;@RestControllerpublic class RedisLock {    @Autowired    private RedisTemplate<String, String> redisTemplate;    @GetMapping(value = "buy")    public String get() {        Boolean phoneLock = redisTemplate.opsForValue().setIfAbsent("phoneLock", "", 3, TimeUnit.SECONDS);        if (!phoneLock) {            return "";        }        Timer timer = null;        try {            String phone = redisTemplate.opsForValue().get("phone");            Integer count = Integer.valueOf(phone);            if (count > 0) {                timer = new Timer();                timer.schedule(new TimerTask() {                    @Override                    public void run() {                        redisTemplate.opsForValue().set("phoneLock", "", 3, TimeUnit.SECONDS);                    }                }, 0, 1);                redisTemplate.opsForValue().set("phone", String.valueOf(count - 1));                System.out.println("抢到了" + count + "号商品");            }        } finally {            if (timer != null) {                timer.cancel();            }            redisTemplate.delete("phoneLock");        }        return "";    }}
```

## 6 使用Redisson简化代码

在步骤5我们的代码已经很完善了，不会出现高并发问题。但是代码确过于冗余，我们为了使用Redis锁，我们需要设置一个定长的key，然后当购买完成后，将key删除。但为了防止key提前过期，我们不得不新增一个线程执行定时任务。

下面我们可以使用Redissson框架简化代码。`getLock()`方法代替了Redis的`setIfAbsent()`，`lock()`设置过期时间。最终我们在交易结束后释放锁。延长锁的操作则有Redisson框架替我们完成，它会使用轮询去查看key是否过期，

在交易没有完成时，自动重设Redis的key过期时间

```java
package springbootdemo.demo.controller;/* * @auther 顶风少年 * @mail dfsn19970313@foxmail.com * @date 2020-01-13 11:19 * @notify * @version 1.0 */import org.redisson.Redisson;import org.redisson.api.RLock;import org.springframework.beans.factory.annotation.Autowired;import org.springframework.data.redis.core.RedisTemplate;import org.springframework.web.bind.annotation.GetMapping;import org.springframework.web.bind.annotation.RestController;import java.util.Timer;import java.util.TimerTask;import java.util.concurrent.TimeUnit;@RestControllerpublic class RedissonLock {    @Autowired    private RedisTemplate<String, String> redisTemplate;    @Autowired    private Redisson redisson;    @GetMapping(value = "buy2")    public String get() {        RLock phoneLock = redisson.getLock("phoneLock");        phoneLock.lock(3, TimeUnit.SECONDS);        try {            String phone = redisTemplate.opsForValue().get("phone");            Integer count = Integer.valueOf(phone);            if (count > 0) {                redisTemplate.opsForValue().set("phone", String.valueOf(count - 1));                System.out.println("抢到了" + count + "号商品");            }        } finally {            phoneLock.unlock();        }        return "";    }}
```

收外国男人的钱，骗中国妹子的炮？天朝竟有这样一帮「女权组织」 2018-03-19 INSIGHT视界 From 酷玩实验室 微信号：coollabs 其实我读书的时候 也曾经想过做一个女权主义者 但是后来发生了一些事情 让我选择了放弃 简单来说是这么一个事情：我发现 女权对于一些中国人来说是信仰 但是对另一些中国人来说是生意 所谓的“伪女权”“女权癌” 大概就是这么回事 尽管早就有这样的思想准备 但让我没想到的是 这两天，知乎上曝光了一件大事 还是让我三观震碎 我没想到，这些“伪女权” 竟然已经形成了黑色产业链 让人细思恐极—— 国内竟然有一群人 打着“女权主义”的名号 从事着组织卖淫的事情 在中国女生不知情的情况下 把她们卖给外国男人！事情是这样的：根据知乎用户伊利丹·怒风的爆料 他在知乎和一个伪女权主义者 吵了起来 一开始，他可能以为这只是一个 脑子比较轴的伪女权主义者 所以两人就吵了一通 本来，他以为就是撕个逼而已 没想到的是 这个伪女权主义者 可不是什么好惹的主 这个自称为“玛丽女王”的人 竟然在半个月中 持续不断地骚扰他 而最夸张的是 玛丽女王声称 自己有能力 让伊利丹的QQ号 在5天之内被封掉 到这里为止 伊利丹一直以为 他不过是碰到了一个杠精 但是万万没想到 5天之后 他的QQ号竟然真的被永久封禁了！说真的，这就有点吓人了 这个不起眼的玛丽女王 竟然还能操控别人的QQ账号被封？难不成，她真的背后有人？伊利丹这才意识到 自己好像惹到了一个组织 他去扒了扒这个玛丽女王的QQ空间 这才发现 自己简直捅出一个马蜂窝：这个人平时干的 竟然是把中国女生 卖给外国男人的皮肉生意！真的，我本来以为 我是一个见过不少套路的人 但没想到 这一套操作 真的是惊为天人 简单来说是这样的 首先，玛丽女王自称是“女权主义者” 但是实际上她的言论 宣传的却是 中国男人配不上中国女人 她甚至恶意辱骂中国男人 恨不得中国男人全部死光 连自己的爸爸都不放过 但是，这么做对她有什么好处呢？很简单 骂完中国男人以后 接下来她就说—— 既然中国男人这么差劲 那就找外国男人吧！于是，她就经常发布外国男人的介绍 看起来是一个热心的媒婆 还在各种QQ和微信群里 散播此类信息 但是看到这里 我们不难发现有点问题 看看其中这些不堪入目的措辞 这并不是普通的介绍男友啊！这简直是在拉皮条啊！果然，伊利丹发现 玛丽女王真的在 拉皮条的过程中 收外国男人的钱！下面是聊天记录实锤：而且，请注意—— 在这个过程中 她会收外国男人的钱 但是钱不给中国女生 却落到了她自己的腰包 于是一个诡异的情况出现了：中国妹子 并不知道收钱这回事 还以为是正常交友 而外国男人 却都交了钱 很可能认为自己是在买春！额，也就是说 在中国女孩不知情的情况下 她们被“卖”给了外国男人 而好处费 却全都进了玛丽女王的腰包... 我真的是没见过这种操作 这说轻了是骗炮 说重了，已经可以算是卖淫了吧？我想请熟悉刑法的朋友们看看 这个玛丽女王 至少应该算是个 介绍组织卖淫罪吧？而且，从伊利丹曝光的资料看来 这个组织规模不小 玛丽女王甚至把外国男生的信息 建了一个完整的表格 有详细的个人资料、照片 可以说 是一条非常完整的产业链 那如果按照这样操作 外国男人都是来嫖的 中国女生却不知道 还以为是要跟他们谈恋爱 那双方难道不会穿帮吗？恩，在这方面 玛丽女王早有对策 根据知乎一位 从事过这个产业的匿名用户提供的信息 针对这种情况 玛丽女王们 还会手把手地教外国男人 怎么快速摆脱女生的纠缠 怎么调教中国女生 怎么让女生觉得自己很可爱 可以说 各种套路一应俱全 甚至还可以开发票！看到这里 她们背后的产业就非常清楚了 这个玛丽女王 她根本就不是什么女权主义者 而是打着女权主义的口号 贩卖中国女生的人贩子 一方面 她们通过辱骂中国男人 吸引对外国男人感兴趣的中国女生 另一方面 她们向外国男人收钱 然后把中国女生卖给他们！图片来源：知乎@渭水徐工 而可怜的中国妹子们 还以为自己是在 追求男女平权 其实，不过是沦为了 这些老鸨的赚钱工具 伊利丹把这整个事情 写出来以后 在知乎、微博引起了巨大的关注 关于其中提到的 伊利丹的QQ被永久封禁的问题 腾讯经过核查 目前也有了结果：经调查，是玛丽女王利用伪造证据 恶意举报了伊利丹的QQ号 目前，腾讯已经将伊利丹的QQ解封 同时封禁了玛丽女王等人的 两个QQ账号 警方也就此事立案侦查了 相信很快就会有结果 这个事情算是告一段落了 但是在我看来 却有一件事让我无法释怀：为什么“女权主义”竟然会和 辱骂中国男性等同起来？为什么“和外国男人交友” 竟然还能演变成 一个免费的陪睡组织？我想，这个玛丽女王 也许只是一个 发现了恶性赚钱模式的生意人 但是在这背后隐藏的 其实是一个很深的问题：为什么有不少中国女人 越来越看不上中国男人 甚至觉得嫁给外国男人 是一种时尚？这里面的原因可能非常复杂 我这里先提供一个思路 供大家讨论：我发现 现在中国很多大型的女权组织 背后都有着西方势力的影子 她们打着女权的名号 为自己谋取暴利 为西方国家从事破坏活动 而那些真正为女性平权而奔走的人 却得不到应有的帮助 我之所以这样说 并不是信口开河 而是有充足的证据 有一个非常有名的民间女权组织 叫做“女权之声” 它一再声称 自己只是一个自发的民间组织 致力于促进男女平等的 它所有的微博账号、微信账号 全部都是由一个 叫做妇女传媒监测网络的创办的 而这个妇女传媒监测网络 有这么多媒体产品 那它的钱都是哪里来的呢？从她们介绍的合作组织里 我们可以清楚地找到 她们的资助者—— 竟然有西方的福特基金会 有人也许会问 收了西方的钱怎么了？中国的组织不能收西方的钱吗？然而，她们不只是收了西方的钱而已 女权之声组织里 有一个人叫做郑楚然 她除了女权运动之外 没有任何其他工作 表面上，是一个全职的女权工作者 在2015年的时候 她还因为寻衅滋事 被警察拘留过30多天 甚至在她被拘留的时候 希拉里还借题发挥 指责中国侵犯人权、压制民主 一个中国的小小民间组织的首领 在互联网上的粉丝还没有我多 竟然能得到希拉里这个级别的关注？我真的是惊掉了下巴 这样看来 我离希拉里也不是很远了？？而不止是希拉里 这样一个明明思想上毫无建树的人 却被西方媒体BBC评为了 全球百大思想家 图：郑楚然在王宝强事件中发表的言论 除此以外 更让人匪夷所思的 是她们平时就喜欢攻击政府 甚至于，她们还会试图分裂我们国家 比如，女权之声这个组织里 著名的女权斗士洪理达 就曾经转发著名的港独媒体 Hong Kong Free Press的言论 甚至曾公开发表过 支持藏独、港独、台独的言论 她也经常和郑楚然混在一起 我很想不通 如果她们真的只是单纯的女权主义者 为何要发表分裂国家的言论？为何要支持藏独、港独、台独？我只能说，这大概就叫 拿人家的手短，吃人家的嘴软吧 以前，我在接触中国的女权组织时 我就觉得很奇怪 她们都喜欢声称 自己是不盈利的非政府组织 但是她们无论是宣传 还是组织各类活动 都需要大量的钱 如果她们真的不盈利 那这些钱都是哪里来的呢？而这些外国的金主 他们也更加不可能是什么慈善组织 大发善心来给中国人投钱 每一分投出去的钱 一定都是要有回报的 那么，他们的回报是什么呢？他们给中国的“女权组织”投钱 能得到什么利益呢？联想到中国网络上 如火如荼的对中国男人的讨伐 我只能说，细思恐极 我绝不是危言耸听 因为我们就看不远的邻国日本 近些年来日本对于西方的崇拜 可谓深入骨髓 已经到了崇洋媚外的程度 而这其中 当然也包括对白人男性的崇拜 甚至在2016年一个瑞士白人 发了一个视频，赤裸裸的说 “在东京，只要你是白人， 做什么都可以” 视频里面他在日本便利店 随意的亲吻不认识的收银员女孩 在酒吧把不认识的日本女孩 按向自己的裤裆 而日本女孩回应的却是谄媚的笑容 我想，并不会有那么多中国人 真正被西方伪女权主义控制 但是，我们要警惕的是 别在你自己都没有察觉的时候 被别有用心的人洗了脑 更有甚者 别在你自己都不知道的情况下 被别人卖给了外国男人 还去帮他数钱 本文系授权发布，From 酷玩实验室，微信号：coollabs，欢迎分享到朋友圈，未经许可不得转载，INSIGHT视界 诚意推荐 Forwarded from Official Account 酷玩实验室 酷玩实验室 Learn More Scan QR Code via WeChat to follow Official Account 采集文章采集样式近似文章查看封

**更多精彩内容，****关注我们**

**▼▼**

![](https://res.wx.qq.com/t/fed_upload/b39ef69e-c4d6-4169-8612-5f00a84860e7/wx-avatar-default.svg)

公众号

**如果觉得这篇文章还不错**

**点击下面卡片关注我**

******点个“**在看**”吧******![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)****

阅读 318

​