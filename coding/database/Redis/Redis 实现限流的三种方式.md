# 

玄姐聊AGI

_2021年11月27日 21:00_

![](https://res.wx.qq.com/t/fed_upload/b39ef69e-c4d6-4169-8612-5f00a84860e7/wx-avatar-default.svg)

公众号

- 第一种：基于Redis的setnx的操作

- 第二种：基于Redis的数据结构zset

- 第三种：基于Redis的令牌桶算法

______________________________________________________________________

面对越来越多的高并发场景，限流显示的尤为重要。

当然，限流有许多种实现的方式，Redis具有很强大的功能，我用Redis实践了三种的实现方式，可以较为简单的实现其方式。Redis不仅仅是可以做限流，还可以做数据统计，附近的人等功能，这些可能会后续写到。

## 第一种：基于Redis的setnx的操作

我们在使用Redis的分布式锁的时候，大家都知道是依靠了setnx的指令，在CAS（Compare and swap）的操作的时候，同时给指定的key设置了过期实践（expire），我们在限流的主要目的就是为了在单位时间内，有且仅有N数量的请求能够访问我的代码程序。所以依靠setnx可以很轻松的做到这方面的功能。

比如我们需要在10秒内限定20个请求，那么我们在setnx的时候可以设置过期时间10，当请求的setnx数量达到20时候即达到了限流效果。代码比较简单就不做展示了。

当然这种做法的弊端是很多的，比如当统计1-10秒的时候，无法统计2-11秒之内，如果需要统计N秒内的M个请求，那么我们的Redis中需要保持N个key等等问题

## 第二种：基于Redis的数据结构zset

其实限流涉及的最主要的就是滑动窗口，上面也提到1-10怎么变成2-11。其实也就是起始值和末端值都各+1即可。

而我们如果用Redis的list数据结构可以轻而易举的实现该功能

我们可以将请求打造成一个zset数组，当每一次请求进来的时候，value保持唯一，可以用UUID生成，而score可以用当前时间戳表示，因为score我们可以用来计算当前时间戳之内有多少的请求数量。而zset数据结构也提供了range方法让我们可以很轻易的获取到2个时间戳内有多少请求

代码如下

```java
public Response limitFlow(){        Long currentTime = new Date().getTime();        System.out.println(currentTime);        if(redisTemplate.hasKey("limit")) {            Integer count = redisTemplate.opsForZSet().rangeByScore("limit", currentTime -  intervalTime, currentTime).size();        // intervalTime是限流的时间             System.out.println(count);            if (count != null && count > 5) {                return Response.ok("每分钟最多只能访问5次");            }        }        redisTemplate.opsForZSet().add("limit",UUID.randomUUID().toString(),currentTime);        return Response.ok("访问成功");    }
```

通过上述代码可以做到滑动窗口的效果，并且能保证每N秒内至多M个请求，缺点就是zset的数据结构会越来越大。实现方式相对也是比较简单的。

## 第三种：基于Redis的令牌桶算法

提到限流就不得不提到令牌桶算法了。

令牌桶算法提及到输入速率和输出速率，当输出速率大于输入速率，那么就是超出流量限制了。

也就是说我们每访问一次请求的时候，可以从Redis中获取一个令牌，如果拿到令牌了，那就说明没超出限制，而如果拿不到，则结果相反。

依靠上述的思想，我们可以结合Redis的List数据结构很轻易的做到这样的代码，只是简单实现

依靠List的leftPop来获取令牌

```java
// 输出令牌public Response limitFlow2(Long id){        Object result = redisTemplate.opsForList().leftPop("limit_list");        if(result == null){            return Response.ok("当前令牌桶中无令牌");        }        return Response.ok(articleDescription2);    }
```

再依靠Java的定时任务，定时往List中rightPush令牌，当然令牌也需要唯一性，所以我这里还是用UUID进行了生成

```
// 10S的速率往令牌桶中添加UUID，只为保证唯一性    @Scheduled(fixedDelay = 10_000,initialDelay = 0)    public void setIntervalTimeTask(){        redisTemplate.opsForList().rightPush("limit_list",UUID.randomUUID().toString());    }
```

综上，代码实现起始都不是很难，针对这些限流方式我们可以在AOP或者filter中加入以上代码，用来做到接口的限流，最终保护你的网站。

Redis其实还有很多其他的用处，他的作用不仅仅是缓存，分布式锁的作用。他的数据结构也不仅仅是只有String，Hash，List，Set，Zset。有兴趣的可以后续了解下他的GeoHash算法；BitMap，HLL以及布隆过滤器数据(Redis4.0之后加入，可以用Docker直接安装redislabs/rebloom)结构。

> 作者：Leonis丶L
>
> 来源：blog.csdn.net/lmx125254/article/details/90700118

![](https://res.wx.qq.com/t/fed_upload/b39ef69e-c4d6-4169-8612-5f00a84860e7/wx-avatar-default.svg)

公众号

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

阅读 3321

​
