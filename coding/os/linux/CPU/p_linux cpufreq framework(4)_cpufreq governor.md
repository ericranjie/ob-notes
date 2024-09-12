# [蜗窝科技](http://www.wowotech.net/)

### 慢下来，享受技术。

[![](http://www.wowotech.net/content/uploadfile/201401/top-1389777175.jpg)](http://www.wowotech.net/)

- [博客](http://www.wowotech.net/)
- [项目](http://www.wowotech.net/sort/project)
- [关于蜗窝](http://www.wowotech.net/about.html)
- [联系我们](http://www.wowotech.net/contact_us.html)
- [支持与合作](http://www.wowotech.net/support_us.html)
- [登录](http://www.wowotech.net/admin)

﻿

## 

作者：[wowo](http://www.wowotech.net/author/2 "runangaozhong@163.com") 发布于：2015-8-23 21:15 分类：[电源管理子系统](http://www.wowotech.net/sort/pm_subsystem)

## 1. 前言

由“[linux cpufreq framework(3)_cpufreq core](http://www.wowotech.net/pm_subsystem/cpufreq_core.html)”的描述可知，cpufreq policy负责设定cpu调频的一个大致范围，而cpu的具体运行频率，则需要由相应的cufreq governor决定（可自行调节频率的CPU除外，后面会再详细介绍）。那到底什么是cpufreq governor？它的运行机制是什么？这就是本文要描述的内容。

## 2. cpufreq governor的实现

### 2.1 struct cpufreq_governor

kernel通过struct cpufreq_governor抽象cpufreq governor，如下：

  1: /* include/linux/cpufreq.h */

  2: struct cpufreq_governor {

  3:         char    name[CPUFREQ_NAME_LEN];

  4:         int     initialized;

  5:         int     (*governor)     (struct cpufreq_policy *policy,

  6:                                  unsigned int event);

  7:         ssize_t (*show_setspeed)        (struct cpufreq_policy *policy,

  8:                                          char *buf);

  9:         int     (*store_setspeed)       (struct cpufreq_policy *policy,

 10:                                          unsigned int freq);

 11:         unsigned int max_transition_latency; /* HW must be able to switch to

 12:                         next freq faster than this value in nano secs or we

 13:                         will fallback to performance governor */

 14:         struct list_head        governor_list;

 15:         struct module           *owner;

 16: };

> name，该governor的名称，唯一标识某个governor。
> 
> initialized，记录governor是否已经初始化okay。
> 
> max_transition_latency，容许的最大频率切换延迟，硬件频率的切换必须小于这个值，才能满足需求。
> 
> governor_list，用于将该governor挂到一个全局的governor链表（cpufreq_governor_list）上。
> 
> show_setspeed和store_setspeed，回忆一下“[linux cpufreq framework(3)_cpufreq core](http://www.wowotech.net/pm_subsystem/cpufreq_core.html)”所描述的“scaling_setspeed ”，有些governor支持从用户空间修改频率值，此时该governor必须提供show_setspeed和store_setspeed两个回调函数，用于响应用户空间的scaling_setspeed请求。
> 
> governor，cpufreq governor的主要功能都是通过该回调函数实现，该函数借助不同的event，以状态机的形式，实现governor的启动、停止等操作，具体请参考后续的描述。

### 2.2 governor event

kernel将governor的控制方式抽象为下面的5个event，cpufreq core在合适的时机（具体参考下面第3章的介绍），以event的形式（.governor回调），控制governor完成相应的调频动作。

  1: /* include/linux/cpufreq.h */

  2: 

  3: /* Governor Events */

  4: #define CPUFREQ_GOV_START       1

  5: #define CPUFREQ_GOV_STOP        2

  6: #define CPUFREQ_GOV_LIMITS      3

  7: #define CPUFREQ_GOV_POLICY_INIT 4

  8: #define CPUFREQ_GOV_POLICY_EXIT 5

> CPUFREQ_GOV_POLICY_INIT，policy启动新的governor之前（通常在cpufreq policy刚创建或者governor改变时）发送。governor接收到这个event之后，会进行前期的准备工作，例如初始化一些必要的数据结构（如timer）等。并不是所有governor都需要这个event，后面如果有时间，我们以ondemand governor为例，再介绍它的意义。
> 
> CPUFREQ_GOV_START启动governor。
> 
> CPUFREQ_GOV_STOP、CPUFREQ_GOV_POLICY_EXIT，和前面两个event的意义相反。
> 
> CPUFREQ_GOV_LIMITS，通常在governor启动后发送，要求governor检查并修改频率值，使其在policy规定的有效范围内。

### 2.3 governor register

所有governor都是通过cpufreq_register_governor注册到kernel中的，该接口比较简单，查找是否有相同名称的governor已经注册，如果没有，将这个governor挂到全局的链表即可，如下：

  1: int cpufreq_register_governor(struct cpufreq_governor *governor)

  2: {

  3:         int err;

  4: 

  5:         if (!governor)

  6:                 return -EINVAL;

  7: 

  8:         if (cpufreq_disabled())

  9:                 return -ENODEV;

 10: 

 11:         mutex_lock(&cpufreq_governor_mutex);

 12: 

 13:         governor->initialized = 0;

 14:         err = -EBUSY;

 15:         if (__find_governor(governor->name) == NULL) {

 16:                 err = 0;

 17:                 list_add(&governor->governor_list, &cpufreq_governor_list);

 18:         }

 19: 

 20:         mutex_unlock(&cpufreq_governor_mutex);

 21:         return err;

 22: }

 23: EXPORT_SYMBOL_GPL(cpufreq_register_governor);

## 3 governor相关的调用流程

### 3.1 启动流程

“[linux cpufreq framework(3)_cpufreq core](http://www.wowotech.net/pm_subsystem/cpufreq_core.html)”中介绍过，添加cpufreq设备时，会调用cpufreq_init_policy，该接口的主要功能是为当前的cpufreq policy分配并启动一个cpufreq governor，如下：

  1: static void cpufreq_init_policy(struct cpufreq_policy *policy)

  2: {

  3:         struct cpufreq_governor *gov = NULL;

  4:         struct cpufreq_policy new_policy;

  5:         int ret = 0;

  6: 

  7:         memcpy(&new_policy, policy, sizeof(*policy));

  8: 

  9:         /* Update governor of new_policy to the governor used before hotplug */

 10:         gov = __find_governor(per_cpu(cpufreq_cpu_governor, policy->cpu));

 11:         if (gov)

 12:                 pr_debug("Restoring governor %s for cpu %d\n",

 13:                                 policy->governor->name, policy->cpu);

 14:         else

 15:                 gov = CPUFREQ_DEFAULT_GOVERNOR;

 16: 

 17:         new_policy.governor = gov;

 18: 

 19:         /* Use the default policy if its valid. */

 20:         if (cpufreq_driver->setpolicy)

 21:                 cpufreq_parse_governor(gov->name, &new_policy.policy, NULL);

 22: 

 23:         /* set default policy */

 24:         ret = cpufreq_set_policy(policy, &new_policy);

 25:         if (ret) {

 26:                 pr_debug("setting policy failed\n");

 27:                 if (cpufreq_driver->exit)

 28:                         cpufreq_driver->exit(policy);

 29:         }

 30: }

> 9~13行：首先查看是否在hotplug之前最后使用的governor（保存在per cpu的全局变量cpufreq_cpu_governor中），如果有，则直接使用这个governor。
> 
> 14~15行：如果没有，则使用默认的governor----CPUFREQ_DEFAULT_GOVERNOR，该governor在include/linux/cpufreq.h中定义，可以通过kernel配置项选择，可选的governor包括performace、powersave、userspace、ondmand和conservative五种。
> 
> 20~21行：如果cpufreq driver提供了setpolicy接口，则说明CPU可以在policy指定的有效范围内，确定具体的运行频率，因此不再需要governor确定运行频率。但如果此时的governor是performace和powersave两种，则有必要通知到cpufreq driver，以便它的setpolicy接口可以根据实际情况正确设置频率范围。怎么通知呢？通过struct cpufreq_policy结构中的policy变量（名字很费解啊！），可选的值有两个，CPUFREQ_POLICY_PERFORMANCE和CPUFREQ_POLICY_POWERSAVE。
> 
> 24行：调用cpufreq_set_policy，启动governor，代码如下。

  1: static int cpufreq_set_policy(struct cpufreq_policy *policy,

  2:                                 struct cpufreq_policy *new_policy)

  3: {

  4:         ...

  5:         if (cpufreq_driver->setpolicy) {

  6:                 policy->policy = new_policy->policy;

  7:                 pr_debug("setting range\n");

  8:                 return cpufreq_driver->setpolicy(new_policy);

  9:         }

 10: 

 11:         if (new_policy->governor == policy->governor)

 12:                 goto out;

 13: 

 14:         pr_debug("governor switch\n");

 15: 

 16:         /* save old, working values */

 17:         old_gov = policy->governor;

 18:         /* end old governor */

 19:         if (old_gov) {

 20:                 __cpufreq_governor(policy, CPUFREQ_GOV_STOP);

 21:                 up_write(&policy->rwsem);

 22:                 __cpufreq_governor(policy, CPUFREQ_GOV_POLICY_EXIT);

 23:                 down_write(&policy->rwsem);

 24:         }

 25: 

 26:         /* start new governor */

 27:         policy->governor = new_policy->governor;

 28:         if (!__cpufreq_governor(policy, CPUFREQ_GOV_POLICY_INIT)) {

 29:                 if (!__cpufreq_governor(policy, CPUFREQ_GOV_START))

 30:                         goto out;

 31: 

 32:                 up_write(&policy->rwsem);

 33:                 __cpufreq_governor(policy, CPUFREQ_GOV_POLICY_EXIT);

 34:                 down_write(&policy->rwsem);

 35:         }

 36: 

 37:         /* new governor failed, so re-start old one */

 38:         pr_debug("starting governor %s failed\n", policy->governor->name);

 39:         if (old_gov) {

 40:                 policy->governor = old_gov;

 41:                 __cpufreq_governor(policy, CPUFREQ_GOV_POLICY_INIT);

 42:                 __cpufreq_governor(policy, CPUFREQ_GOV_START);

 43:         }

 44: 

 45:         return -EINVAL;

 46: 

 47:  out:

 48:         pr_debug("governor: change or update limits\n");

 49:         return __cpufreq_governor(policy, CPUFREQ_GOV_LIMITS);

 50: }

> 5~9行，对应上面20~21行的逻辑，如果有setpolicy接口，则直接调用，不再进行后续的governor操作，因此使用CPUFREQ_POLICY_PERFORMANCE和CPUFREQ_POLICY_POWERSAVE两个值，变相的传递governor的信息。
> 
> 11~12行，如果新旧governor相同，直接返回。
> 
> 19~24行，如果存在旧的governor，停止它，流程是：  
> CPUFREQ_GOV_STOP---->CPUFREQ_GOV_POLICY_EXIT
> 
> 剩余的代码：启动新的governor，流程是：CPUFREQ_GOV_POLICY_INIT---->CPUFREQ_GOV_START---->CPUFREQ_GOV_LIMITS

### 3.2 调频流程

前面已经多次提到基于cpufreq governor的调频思路，这里再总结一下：

1）有两种类型的cpu：一种只需要给定调频范围，cpu会在该范围内自行确定运行频率；另一种需要软件指定具体的运行频率。

2）对第一种cpu，cpufreq policy中会指定频率范围policy->{min, max}，之后通过setpolicy接口，使其生效即可。

3）对第二种cpu，cpufreq policy在指定频率范围的同时，会指明使用的governor。governor在启动后，会动态的（例如启动一个timer，监测系统运行情况，并根据负荷调整频率），或者静态的（直接设置为某一个合适的频率值），设定cpu运行频率。

kernel document对这个过程有详细的解释，如下：

> Documentation\cpu-freq\governors.txt
> 
> CPU can be set to switch independently   |         CPU can only be set   
>                   within specific "limits"           |       to specific frequencies
> 
>                                  "CPUfreq policy"  
>                 consists of frequency limits (policy->{min,max})  
>                      and CPUfreq governor to be used  
>                          /                    \  
>                         /                      \  
>                        /                       the cpufreq governor decides  
>                       /                        (dynamically or statically)  
>                      /                         what target_freq to set within  
>                     /                          the limits of policy->{min,max}  
>                    /                                \  
>                   /                                  \  
>         Using the ->setpolicy call,              Using the ->target/target_index call,  
>             the limits and the                    the frequency closest  
>              "policy" is set.                     to target_freq is set.  
>                                                   It is assured that it  
>                                                   is within policy->{min,max}

## 4 常用的governor介绍

最后，我们介绍一下kernel中常见的cpufreq governor。

1）Performance

性能优先的governor，直接将cpu频率设置为policy->{min,max}中的最大值。

2）Powersave

功耗优先的governor，直接将cpu频率设置为policy->{min,max}中的最小值。

3）Userspace

由用户空间程序通过scaling_setspeed文件修改频率。

4）Ondemand

根据CPU的当前使用率，动态的调节CPU频率。

5）Conservative

类似Ondemand，不过频率调节的会平滑一下，不会忽然调整为最大值，又忽然调整为最小值。

_原创文章，转发请注明出处。蜗窝科技_，[www.wowotech.net](http://www.wowotech.net/pm_subsystem/cpufreq_governor.html)。

标签: [Linux](http://www.wowotech.net/tag/Linux) [governor](http://www.wowotech.net/tag/governor) [cpufreq](http://www.wowotech.net/tag/cpufreq)

[![](http://www.wowotech.net/content/uploadfile/201605/ef3e1463542768.png)](http://www.wowotech.net/support_us.html)

« [Linux内核同步机制之（六）：Seqlock](http://www.wowotech.net/kernel_synchronization/seqlock.html) | [Concurrency Managed Workqueue之（四）：workqueue如何处理work](http://www.wowotech.net/irq_subsystem/queue_and_handle_work.html)»

**评论：**

**wangzhe**  
2019-04-19 17:46

感谢分享！请问一下，governor控制调频的动作多长时间被唤醒一次呢？这个调度的操作是怎么实现的哈？感谢wowo

[回复](http://www.wowotech.net/pm_subsystem/cpufreq_governor.html#comment-7363)

**jiffy**  
2018-04-21 11:49

@wowo,请教大神：  
1）有两种类型的cpu：一种只需要给定调频范围，cpu会在该范围内自行确定运行频率；另一种需要软件指定具体的运行频率。  
---》对于第一种CPU自行确定运行频率，是指硬件会计算当前的loading来动态调节到某一个合适的频率吗？麻烦解惑一下，非常感谢！

[回复](http://www.wowotech.net/pm_subsystem/cpufreq_governor.html#comment-6688)

**[wowo](http://www.wowotech.net/)**  
2018-04-25 20:26

@jiffy：是的，不过这种硬件很少，市面上基本上看不到。

[回复](http://www.wowotech.net/pm_subsystem/cpufreq_governor.html#comment-6701)

**[wowo](http://www.wowotech.net/)**  
2017-06-30 16:38

@yellow, 没问题，就是这样意思。

[回复](http://www.wowotech.net/pm_subsystem/cpufreq_governor.html#comment-5765)

**yellow**  
2017-06-26 19:04

@wowo：  
  请教个问题，下面说的这句话：cpufreq policy 指定的频率范围->这个范围是从哪来的？  
是不是应该也有一套机制的？ 望指教下，谢谢。  
  
3）对第二种cpu，cpufreq policy在指定频率范围的同时，会指明使用的governor。governor在启动后，会动态的（例如启动一个timer，监测系统运行情况，并根据负荷调整频率），或者静态的（直接设置为某一个合适的频率值），设定cpu运行频率。

[回复](http://www.wowotech.net/pm_subsystem/cpufreq_governor.html#comment-5736)

**[wowo](http://www.wowotech.net/)**  
2017-06-27 10:48

@yellow：是的，这两种情况，使用的是同一套机制（就是我们的cpufreq框架啊）。区别是，第一种情况比较简单，用户空间程序根据系统情况，写入一个期望的频率范围（policy）之后，由CPU根据负荷，自行在这个频率范围里面调整而已。  
本质上，是CPU的硬件代替了软件governor。

[回复](http://www.wowotech.net/pm_subsystem/cpufreq_governor.html#comment-5740)

**yellow**  
2017-06-27 14:03

@wowo：@wowo:    
（1）用户空间程序（写入一个policy） -> （2）Governor 根据policy的范围，来选择一个具体的值 ->设置具体的频率（opp）  
  
我不理解的是用户空间程序怎么决定policy的，是要自己实现一套吗？ 如果用户空间不写入policy，那么Linux 这套gorverno机制就不起作用了？

[回复](http://www.wowotech.net/pm_subsystem/cpufreq_governor.html#comment-5743)

**linda**  
2017-06-27 18:35

@yellow：userspace只能设置policy->{min, max}范围，通过/sys/devices/system/cpu/scaling_max/min_freq实现

[回复](http://www.wowotech.net/pm_subsystem/cpufreq_governor.html#comment-5745)

**yellow**  
2017-06-27 21:26

@linda：我知道是userspace设的，但是userspace也是有一套机制去设吧，比如什么时候设sys/**节点，要设置的范围是多少。那么userspace的这套代码在哪，是开发者自己去实现的吗

[回复](http://www.wowotech.net/pm_subsystem/cpufreq_governor.html#comment-5747)

**[wowo](http://www.wowotech.net/)**  
2017-06-28 08:48

@yellow：这种情况下，CPU自己已经可以做很多事儿了（根据自己的负载调整频率），因此用户空间代码基本不需要实现什么东西。至于policy，顶多是告诉cpufreq，限制一下最大频率而已（避免不稳定）。其它的，你可以思考一下，还需要什么呢？？

[回复](http://www.wowotech.net/pm_subsystem/cpufreq_governor.html#comment-5749)

**linda**  
2017-06-28 09:35

@wowo：赞同，CPU自己根据负载在policy->{min,max}范围内调频就够了，userspace只需要根据CPU能够允许的调频范围设置policy->{min,max}

**yellow**  
2017-06-28 11:39

@wowo：可以这么理解吗  
1.policy（我目前的理解是有某些应用或者userspace才会通过节点去设置）  
2.即使没有policy，kernel也是会根据自己的负载去调节频率。只是如果有policy了，那kernel就在这个policy限制的范围内再去调整频率。  
谢谢！

**yellow**  
2017-06-30 13:17

@wowo：上面的理解对不对呀，不搞清楚总感觉又根刺不舒服，哎

**lgp802a**  
2017-03-24 20:53

感谢wowo这么详尽的分析。实际应用中，比如高通平台。频率调整需要user space参与吗？今天和同事讨论，他说高通平台需要user space的服务根据应用场景来控制。而我理解是只需要governor计算出load，根据策略调整，不需要user space参与。没搞过高通，不知道实际怎么一回事。

[回复](http://www.wowotech.net/pm_subsystem/cpufreq_governor.html#comment-5376)

**[wowo](http://www.wowotech.net/)**  
2017-03-24 21:33

@lgp802a：我也没有做过高通平台，不太了解。  
不过调频有时确实需要用户空间参与的，因为kernel虽然知道load，但不知道用户的关注点：性能还是功耗？

[回复](http://www.wowotech.net/pm_subsystem/cpufreq_governor.html#comment-5378)

**[pgalxx](http://www.wowotech.net/)**  
2016-09-04 09:55

楼主您好！我现在查看cpufreq的interactive 源码，先有一处有疑问，请教下大神哈！  
在cpufreq-interactive.c的cpufreq_interactive_timer函数里面在pcpu->target_freq = new_freq;语句之前的pcpu->target_freq应该是上一次采样时获得的频率，那此时的pcpu->target_freq与pcpu->policy->cur值是否一样的呢？谢谢！

[回复](http://www.wowotech.net/pm_subsystem/cpufreq_governor.html#comment-4509)

**[pgalxx](http://www.wowotech.net/)**  
2016-09-04 13:22

@pgalxx：追加：我现在查看的是3.10.37的源码哈。谢谢！

[回复](http://www.wowotech.net/pm_subsystem/cpufreq_governor.html#comment-4510)

**[wowo](http://www.wowotech.net/)**  
2016-09-05 09:48

@pgalxx：其实我对这些governor也不是很了解，个人的看法是（仅供参考）：  
在你所说的那个时间点，pcpu->target_freq表示之前的freq，new_freq表示当前所要调整为的freq，而pcpu->policy->cur则是用户（例如某一个调频策略）所期望的freq。  
按理说，new_freq是从pcpu->policy->cur得出的，理想情况下，二者相同，但是由于一些限制，可能会不同。  
这就是它们的关系。

[回复](http://www.wowotech.net/pm_subsystem/cpufreq_governor.html#comment-4511)

**[pgalxx](http://www.wowotech.net/)**  
2016-09-05 14:35

@wowo：@wowo 谢谢您的回复！不过我这里还有一点疑问，pcpu->policy->cur您说是用户(例如一个调频策略)所期望的freq,这个是怎么理解呢？我查看到cpufreq.c的__cpufreq_notify_transition（）函数里面在CPUFREQ_POSTCHANGE event中有对policy->cur = freqs->new;而freqs->news是这次调频的频率，所以我认为policy->cur 是系统每次调频成功后根据CPUFREQ_POSTCHANGE的事件将该次调频的频率值赋值给它，所以我认为他们应该他的值和pcpu->target_freq相同才合理，除非底层又有一层频率映射，不知我的理解是否有误呢？

[回复](http://www.wowotech.net/pm_subsystem/cpufreq_governor.html#comment-4512)

**[wowo](http://www.wowotech.net/)**  
2016-09-05 15:25

@pgalxx：你是对的，我上面的表述不是很清楚，抱歉。仔细看了一下interactive的源码，我觉得它的逻辑是这样的：  
0）先明确一点，pcpu->target_freq属于interactive governor，policy->cur属于cpufreq，new_freq是一个计算的中间过程  
1）interactive会定时收集信息，并根据旧的freq（pcpu->target_freq），计算新的cpufreq（就是new_freq）。  
2）如果interactive觉得需要调频（新旧值不同），则用新值（new_freq）更新旧值（pcpu->target_freq）  
    注：你的问题发生在赋值动作之前，因此pcpu->target_freq还是旧值。  
3）interactive会在自己的task中（cpufreq_interactive_speedchange_task），以新值（pcpu->target_freq）为参数，调用driver的调频接口（__cpufreq_driver_target），调整频率值  
    注：你所提的问题，已经有答案了----  
        此时pcpu->target_freq可能是无效值，因此实际传给driver的可能不一样，因此driver最终设定的实际值（policy->cur）可能与它不同；  
        另外，这个值就算在interactive中通过验证了，在实际的driver中，也有可能被判定为无效值，因此，实际值（policy->cur）也可能与它不同。

[回复](http://www.wowotech.net/pm_subsystem/cpufreq_governor.html#comment-4513)

**[pgalxx](http://www.wowotech.net/)**  
2016-09-06 09:54

@wowo：@wowo 感谢wowo大神能如此内心的回答！我查看interactive的task中在调用调频接口__cpufreq_driver_target之前做了如下操作：首先在所有online 的cpu中找到最大的频率（由于现在我所用的cpu是同步多核并不是异步多核所有理论上应该是所有online的cpu核的频率应该是一样的），然后判断pcpu->target_freq是否和pcpu->policy->cur相同，也就是判断如果当前需要调频的频率和当前正使用的频率是否一样，如果一样就不调频了；接着进入调频接口，在调频接口里面判断需要调频的频点是否在policy->min和policy->max之间，及再一次判断需要调频的频率是否和正在使用的频率一样如果一样就不调频了。我对这里的流程理解如下：需要调频的频率是从cpufreq中获取得来，且policy->min和policy->max均是平台cpufreq驱动调用cpufreq_frequency_table_cpuinfo接口（Freq_table.c）填充的，其填充的数据就是平台cpufreq支持的freq-volt表格里面的最大最小值。我理解为这里面需要调频的频点应该都是有效的才合理亚，无效的值无非为不在min-max之间。但是需要调频的频点都是在cpufreq支持的freq-volt里面选择出来的。这里有点不理解哈，除非是支持cpu hotplug里面讲在小核向大核切换时可能就会出现这种情况，不知我的理解是否对呢？另外我现在想通过修改cpufreq的interactive 来优化功耗，目前想到的是cpu load计算方式应该可以优化，针对这个您能否给点建议呢？谢谢！

[回复](http://www.wowotech.net/pm_subsystem/cpufreq_governor.html#comment-4516)

**[wowo](http://www.wowotech.net/)**  
2016-09-06 17:02

@pgalxx：我是觉得，有效性判断更多的是出于严谨的代码习惯：我（interactive governor）在调用driver之前，需要确保我的参数合法，同时“我”也无法保证policy->cur是否会突变（例如硬件自动调整了、大小核切换了、等等），这些不是我的管辖范围，我只是要做到自己是健壮的。  
根据功耗调频，可以有ondemand、conservative、interactive三种governor选择，你可以根据自己的应用场景，例如更care功耗还是更care性能、CPU调频生效时间的快慢、等等，选择一个。具体可以看一下这个文档：  
Documentation/cpu-freq/governors.txt

**[pgalxx](http://www.wowotech.net/)**  
2016-09-06 21:20

@pgalxx：非常感谢wowo的讲解！

**dabo**  
2015-11-25 16:37

wowo大神 对你说的调频流程里面有两种类型的cpu 不太理解。  
第一种 的cpufreq driver提供了setpolicy接口的话，就不在需要governor调频的策略？  
  
那这时候cpu如何调频的 ？由cpufreq driver驱动来实现一个调频的策略然后调用setpolicy接口？ 还是由用户上层调用setpolicy接口设置频率？

[回复](http://www.wowotech.net/pm_subsystem/cpufreq_governor.html#comment-3131)

**[wowo](http://www.wowotech.net/)**  
2015-11-25 17:05

@dabo：具体的策略，还是由governor决定的，只不过如果driver实现了setpolicy，则具体的频率调整，是CPU自己做的。driver只告诉它一个调频范围，CPU自己调整。

[回复](http://www.wowotech.net/pm_subsystem/cpufreq_governor.html#comment-3134)

**dabo**  
2015-11-26 09:43

@wowo：wowo大神 我还是不太理解。cpu 自己调整是什么意思 还不是根据负载来调整么？这样第一种和第二种 有什么区别？

[回复](http://www.wowotech.net/pm_subsystem/cpufreq_governor.html#comment-3138)

**[wowo](http://www.wowotech.net/)**  
2015-11-26 09:48

@dabo：是的，你告诉CPU一个调频的范围，它会在这个范围内，根据负载（或者其它什么），自行调整。  
不过，市面上这样的CPU很少，直接忽略就行了。

[回复](http://www.wowotech.net/pm_subsystem/cpufreq_governor.html#comment-3139)

**dabo**  
2015-11-26 14:41

@wowo：好了解了。再问大神2个问题。  
1.是不是只有在governor为Userspace的模式下，才允许用户空间修改频率。其他模式下用户空间是不能修改频率的？  
2.调频的话主要是在kernel里面进行，根据governor的策略来调整cpu频率。用户空间很少参与？

[回复](http://www.wowotech.net/pm_subsystem/cpufreq_governor.html#comment-3146)

**[wowo](http://www.wowotech.net/)**  
2015-11-26 15:02

@dabo：1. 是的。  
2. 基本生是的。但Performance、Powersave等，还是需要用户空间参与一点点的，例如提供一个范围，不能信马由缰啊~~

[回复](http://www.wowotech.net/pm_subsystem/cpufreq_governor.html#comment-3147)

**[珠海影视制作](http://www.rxmedia.cn/)**  
2015-08-27 22:33

有一阵子没来了，博主的博客还坚持更新着，非常不错哈

[回复](http://www.wowotech.net/pm_subsystem/cpufreq_governor.html#comment-2498)

**[太古神王](http://taigushenwang.org/)**  
2015-08-25 14:17

[给力]

[回复](http://www.wowotech.net/pm_subsystem/cpufreq_governor.html#comment-2491)

**[shoujixiaodao](http://www.wowotech.net/)**  
2015-08-25 11:09

写的好，学习了，cpu hotplug 哈时候写一写？多谢！

[回复](http://www.wowotech.net/pm_subsystem/cpufreq_governor.html#comment-2489)

**[wowo](http://www.wowotech.net/)**  
2015-08-25 12:12

@shoujixiaodao：过奖了，下一篇写hotplug吧，多谢关注~~~

[回复](http://www.wowotech.net/pm_subsystem/cpufreq_governor.html#comment-2490)

**发表评论：**

 昵称

 邮件地址 (选填)

 个人主页 (选填)

![](http://www.wowotech.net/include/lib/checkcode.php) 

- ### 站内搜索
    
       
     蜗窝站内  互联网
    
- ### 功能
    
    [留言板  
    ](http://www.wowotech.net/message_board.html)[评论列表  
    ](http://www.wowotech.net/?plugin=commentlist)[支持者列表  
    ](http://www.wowotech.net/support_list)
- ### 最新评论
    
    - Shiina  
        [一个电路（circuit）中，由于是回路，所以用电势差的概念...](http://www.wowotech.net/basic_subject/voltage.html#8926)
    - Shiina  
        [其中比较关键的点是相对位置概念和点电荷的静电势能计算。](http://www.wowotech.net/basic_subject/voltage.html#8925)
    - leelockhey  
        [你这是哪个内核版本](http://www.wowotech.net/pm_subsystem/generic_pm_architecture.html#8924)
    - ja  
        [@dream：我看完這段也有相同的想法，引用 @dream ...](http://www.wowotech.net/kernel_synchronization/spinlock.html#8922)
    - 元神高手  
        [围观首席power managerment专家](http://www.wowotech.net/pm_subsystem/device_driver_pm.html#8921)
    - 十七  
        [内核空间的映射在系统启动时就已经设定好，并且在所有进程的页表...](http://www.wowotech.net/process_management/context-switch-arch.html#8920)
- ### 文章分类
    
    - [Linux内核分析(25)](http://www.wowotech.net/sort/linux_kenrel) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=4)
        - [统一设备模型(15)](http://www.wowotech.net/sort/device_model) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=12)
        - [电源管理子系统(43)](http://www.wowotech.net/sort/pm_subsystem) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=13)
        - [中断子系统(15)](http://www.wowotech.net/sort/irq_subsystem) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=14)
        - [进程管理(31)](http://www.wowotech.net/sort/process_management) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=15)
        - [内核同步机制(26)](http://www.wowotech.net/sort/kernel_synchronization) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=16)
        - [GPIO子系统(5)](http://www.wowotech.net/sort/gpio_subsystem) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=17)
        - [时间子系统(14)](http://www.wowotech.net/sort/timer_subsystem) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=18)
        - [通信类协议(7)](http://www.wowotech.net/sort/comm) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=20)
        - [内存管理(31)](http://www.wowotech.net/sort/memory_management) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=21)
        - [图形子系统(2)](http://www.wowotech.net/sort/graphic_subsystem) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=23)
        - [文件系统(5)](http://www.wowotech.net/sort/filesystem) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=26)
        - [TTY子系统(6)](http://www.wowotech.net/sort/tty_framework) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=27)
    - [u-boot分析(3)](http://www.wowotech.net/sort/u-boot) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=25)
    - [Linux应用技巧(13)](http://www.wowotech.net/sort/linux_application) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=3)
    - [软件开发(6)](http://www.wowotech.net/sort/soft) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=1)
    - [基础技术(13)](http://www.wowotech.net/sort/basic_tech) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=6)
        - [蓝牙(16)](http://www.wowotech.net/sort/bluetooth) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=10)
        - [ARMv8A Arch(15)](http://www.wowotech.net/sort/armv8a_arch) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=19)
        - [显示(3)](http://www.wowotech.net/sort/display) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=22)
        - [USB(1)](http://www.wowotech.net/sort/usb) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=28)
    - [基础学科(10)](http://www.wowotech.net/sort/basic_subject) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=7)
    - [技术漫谈(12)](http://www.wowotech.net/sort/tech_discuss) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=8)
    - [项目专区(0)](http://www.wowotech.net/sort/project) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=9)
        - [X Project(28)](http://www.wowotech.net/sort/x_project) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=24)
- ### 随机文章
    
    - [一个较复杂dcache问题](http://www.wowotech.net/linux_kenrel/484.html)
    - [页面回收的基本概念](http://www.wowotech.net/memory_management/page_reclaim_basic.html)
    - [X-014-KERNEL-ARM GIC driver的移植](http://www.wowotech.net/x_project/gic_driver_porting.html)
    - [为什么会有文件系统(一)](http://www.wowotech.net/filesystem/370.html)
    - [O(n)、O(1)和CFS调度器](http://www.wowotech.net/process_management/scheduler-history.html)
- ### 文章存档
    
    - [2024年2月(1)](http://www.wowotech.net/record/202402)
    - [2023年5月(1)](http://www.wowotech.net/record/202305)
    - [2022年10月(1)](http://www.wowotech.net/record/202210)
    - [2022年8月(1)](http://www.wowotech.net/record/202208)
    - [2022年6月(1)](http://www.wowotech.net/record/202206)
    - [2022年5月(1)](http://www.wowotech.net/record/202205)
    - [2022年4月(2)](http://www.wowotech.net/record/202204)
    - [2022年2月(2)](http://www.wowotech.net/record/202202)
    - [2021年12月(1)](http://www.wowotech.net/record/202112)
    - [2021年11月(5)](http://www.wowotech.net/record/202111)
    - [2021年7月(1)](http://www.wowotech.net/record/202107)
    - [2021年6月(1)](http://www.wowotech.net/record/202106)
    - [2021年5月(3)](http://www.wowotech.net/record/202105)
    - [2020年3月(3)](http://www.wowotech.net/record/202003)
    - [2020年2月(2)](http://www.wowotech.net/record/202002)
    - [2020年1月(3)](http://www.wowotech.net/record/202001)
    - [2019年12月(3)](http://www.wowotech.net/record/201912)
    - [2019年5月(4)](http://www.wowotech.net/record/201905)
    - [2019年3月(1)](http://www.wowotech.net/record/201903)
    - [2019年1月(3)](http://www.wowotech.net/record/201901)
    - [2018年12月(2)](http://www.wowotech.net/record/201812)
    - [2018年11月(1)](http://www.wowotech.net/record/201811)
    - [2018年10月(2)](http://www.wowotech.net/record/201810)
    - [2018年8月(1)](http://www.wowotech.net/record/201808)
    - [2018年6月(1)](http://www.wowotech.net/record/201806)
    - [2018年5月(1)](http://www.wowotech.net/record/201805)
    - [2018年4月(7)](http://www.wowotech.net/record/201804)
    - [2018年2月(4)](http://www.wowotech.net/record/201802)
    - [2018年1月(5)](http://www.wowotech.net/record/201801)
    - [2017年12月(2)](http://www.wowotech.net/record/201712)
    - [2017年11月(2)](http://www.wowotech.net/record/201711)
    - [2017年10月(1)](http://www.wowotech.net/record/201710)
    - [2017年9月(5)](http://www.wowotech.net/record/201709)
    - [2017年8月(4)](http://www.wowotech.net/record/201708)
    - [2017年7月(4)](http://www.wowotech.net/record/201707)
    - [2017年6月(3)](http://www.wowotech.net/record/201706)
    - [2017年5月(3)](http://www.wowotech.net/record/201705)
    - [2017年4月(1)](http://www.wowotech.net/record/201704)
    - [2017年3月(8)](http://www.wowotech.net/record/201703)
    - [2017年2月(6)](http://www.wowotech.net/record/201702)
    - [2017年1月(5)](http://www.wowotech.net/record/201701)
    - [2016年12月(6)](http://www.wowotech.net/record/201612)
    - [2016年11月(11)](http://www.wowotech.net/record/201611)
    - [2016年10月(9)](http://www.wowotech.net/record/201610)
    - [2016年9月(6)](http://www.wowotech.net/record/201609)
    - [2016年8月(9)](http://www.wowotech.net/record/201608)
    - [2016年7月(5)](http://www.wowotech.net/record/201607)
    - [2016年6月(8)](http://www.wowotech.net/record/201606)
    - [2016年5月(8)](http://www.wowotech.net/record/201605)
    - [2016年4月(7)](http://www.wowotech.net/record/201604)
    - [2016年3月(5)](http://www.wowotech.net/record/201603)
    - [2016年2月(5)](http://www.wowotech.net/record/201602)
    - [2016年1月(6)](http://www.wowotech.net/record/201601)
    - [2015年12月(6)](http://www.wowotech.net/record/201512)
    - [2015年11月(9)](http://www.wowotech.net/record/201511)
    - [2015年10月(9)](http://www.wowotech.net/record/201510)
    - [2015年9月(4)](http://www.wowotech.net/record/201509)
    - [2015年8月(3)](http://www.wowotech.net/record/201508)
    - [2015年7月(7)](http://www.wowotech.net/record/201507)
    - [2015年6月(3)](http://www.wowotech.net/record/201506)
    - [2015年5月(6)](http://www.wowotech.net/record/201505)
    - [2015年4月(9)](http://www.wowotech.net/record/201504)
    - [2015年3月(9)](http://www.wowotech.net/record/201503)
    - [2015年2月(6)](http://www.wowotech.net/record/201502)
    - [2015年1月(6)](http://www.wowotech.net/record/201501)
    - [2014年12月(17)](http://www.wowotech.net/record/201412)
    - [2014年11月(8)](http://www.wowotech.net/record/201411)
    - [2014年10月(9)](http://www.wowotech.net/record/201410)
    - [2014年9月(7)](http://www.wowotech.net/record/201409)
    - [2014年8月(12)](http://www.wowotech.net/record/201408)
    - [2014年7月(6)](http://www.wowotech.net/record/201407)
    - [2014年6月(6)](http://www.wowotech.net/record/201406)
    - [2014年5月(9)](http://www.wowotech.net/record/201405)
    - [2014年4月(9)](http://www.wowotech.net/record/201404)
    - [2014年3月(7)](http://www.wowotech.net/record/201403)
    - [2014年2月(3)](http://www.wowotech.net/record/201402)
    - [2014年1月(4)](http://www.wowotech.net/record/201401)

[![订阅Rss](http://www.wowotech.net/content/templates/default/images/rss.gif)](http://www.wowotech.net/rss.php "RSS订阅")

Copyright @ 2013-2015 [蜗窝科技](http://www.wowotech.net/ "wowotech") All rights reserved. Powered by [emlog](http://www.emlog.net/ "采用emlog系统")