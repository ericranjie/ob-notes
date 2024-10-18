
baron Arm精选 _2024年03月29日 07:42_ _上海_

> ### 目录
>
> - 1、DynamIQ架构中L1 cache的替换策略(以cortex-A710为例)
>
> - 2、core cache的替换策略(以cortex-A710为例)
>
> - 2.1、L1 data cache 遵从MESI协议
>
> - 2.2、L1 instruction cache 没有遵从MESI协议
>
> - 2.3、MESI协议的介绍
>
> - 3、cluster cache 之间的替换策略
>
> - 4、总结
>
> - 参考

> 思考:\
> 1、L1 cache的替换策略是什么，L2和L3的呢\
> 2、哪些的替换策略是由硬件决定的(定死的，软件不可更改的)，哪些的替换策略是软件可以配置的？\
> 3、在经典的 DynamIQ架构 中，数据是什么时候存在L1 cache，什么时候存进L2 cache，什么时候又存进L3 cache，以及他们的替换策略是怎样的？比如什么时候数据只在L1？什么时候数据只在L2？什么时候数据只在L3？还有一些组合，比如什么时候数组同时在L1和L3，而L2没有？这一切的规则是怎样定义的？\
> 说明：\
> 本文讨论经典的DynamIQ的cache架构，忽略 big.LITTLE的cache架构

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/72OMRpZ5hV13KJCtnGJnicQjayDuOpVD6BZCSj1ypeqCGdpbeia57NJ3PfUSVXFKAtS6XpkkByARAv5ZLF4Hib9Ng/640?wx_fmt=png&from=appmsg&wxfrom=13)

______________________________________________________________________

#### 1、DynamIQ架构中L1 cache的替换策略(以cortex-A710为例)

我们先看一下DynamIQ架构中的cache中新增的几个概念：

- (1) Strictly inclusive: 所有存在L1 cache中的数据，必然也存在L2 cache中

- (2) Weakly inclusive: 当miss的时候，数据会被同时缓存到L1和L2，但在之后，L2中的数据可能会被替换

- (3) Fully exclusive: 当miss的时候，数据只会缓存到L1

其实inclusive/exclusive属性描述的正是是 L1和L2之间的替换策略，这部分是硬件定死的，软件不可更改的。

我们再去查阅 `ARMV9 cortex-A710 trm`手册，查看该core的cache类型，得知：\
![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/72OMRpZ5hV13KJCtnGJnicQjayDuOpVD66ZY733QqUxonWibDReqCcIXN34RoBn75weenmOyxz7b9zhV37IPibviag/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

- L1 I-cache和L2之间是 weakly inclusive的

- L1 D-cache和L2之间是 strictly inclusive的

也就是说：

- 当发生D-cache发生miss时，数据缓存到L1 D-cache的时候，也会被缓存到L2 Cache中，当L2 Cache被替换时，L1 D-cache也会跟着被替换

- 当发生I-cache发生miss时，数据缓存到L1 I-cache的时候，也会被缓存到L2 Cache中，当L2 Cache被替换时，L1 I- cache不会被替换

再次总结 ：L1 和 L2之间的cache的替换策略，I-cache和D-cache可以是不同的策略，每一个core都有每一个core的做法，请查阅你使用core的手册。

#### 2、core cache的替换策略(以cortex-A710为例)

为了能够将DynamIQ架构和bit.LITTLE架构的cache放在一起介绍，我们将DynamIQ架构中的L1/L2 cache看做成一个单元统称Core cache，bit.LITTLE架构中的L1 Cache也称之为core cache.\
DynamIQ架构中DSU中的L3 cache称之为cluster cache，bit.LITTLE架构中SCU中的L2 cache也称之为cluster cache。

##### 2.1、L1 data cache 遵从MESI协议

在L1 data cache TAG中，有记录MESI相关比特， 然后将一个core内的cache看做是一个整体，core与core之间的缓存一致性，就由DSU执行MESI协议来维护\
![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/72OMRpZ5hV13KJCtnGJnicQjayDuOpVD6nRpqF8jXqT9ztNorQzWFM7zlkbtjwUxGYsesTsBibo1N4SJAjAJnM1Q/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

##### 2.2、L1 instruction cache 没有遵从MESI协议

因为对于Instruction cache来说，都是只读的，cpu不会改写I-cache中的数据，所以也就不需要硬件维护多核之间缓存的不一致\
![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/72OMRpZ5hV13KJCtnGJnicQjayDuOpVD61Ooyn4icK0tZIvYZRicMLK3wtqHCUxdBctp2Miay3ZFvkhbxuEEG3lT5g/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

##### 2.3、MESI协议的介绍

MESI这四种状态:\
![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/72OMRpZ5hV13KJCtnGJnicQjayDuOpVD6UnFiaPxsqaktL4JvnGRcHduHvOub2YdKJLZTwuOoqjALq15NrMe9kNg/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)\
MESI状态之间的切换:\
![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/72OMRpZ5hV13KJCtnGJnicQjayDuOpVD6saufwNFua26QFt4UQhs4XPCHLhAuRrTIkzaYY5dCGcFneXO6uQ6RxA/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

> Events:\
> RH = Read Hit\
> RMS = Read miss, shared\
> RME = Read miss, exclusive\
> WH = Write hit\
> WM = Write miss\
> SHR = Snoop hit on read\
> SHI = Snoop hit on invalidate\
> LRU = LRU replacement
>
> Bus Transactions:\
> Push = Write cache line back to memory\
> Invalidate = Broadcast invalidate\
> Read = Read cache line from memory

#### 3、cluster cache 之间的替换策略

说实话，`core cache / cluster cache /` 这个名字可能不好，感觉叫`private cache 和 share cache`也会更好，我也不知道官方一般使用哪个，反正我们能理解其意思即可吧。

那么他们之间的替换策略是怎样的呢？

我们知道MMU的页表中的表项中，管理者每一块内存的属性，其实就是cache属性，也就是缓存策略。\
其中就有cacheable和shareable、Inner和Outer的概念。如下是针对 DynamIQ 架构做出的总结，注意哦，仅仅是针对 DynamIQ 架构的cache。

- 如果将block的内存属性配置成Non-cacheable，那么数据就不会被缓存到cache，那么所有observer看到的内存是一致的，也就说此时也相当于Outer Shareable。\
  其实官方文档，也有这一句的描述：\
  在B2.7.2章节 “Data accesses to memory locations are coherent for all observers in the system, and correspondingly are treated as being Outer Shareable”

- 如果将block的内存属性配置成write-through cacheable 或 write-back cacheable，那么数据会被缓存cache中。write-through和write-back是缓存策略。

- 如果将block的内存属性配置成 non-shareable, 那么core0访问该内存时，数据缓存的到Core0的L1 D-cache / L2 cache （将L1/L2看做一个整体，直接说数据会缓存到core0的private cache更好），不会缓存到其它cache中。

- 如果将block的内存属性配置成 inner-shareable, 那么core0访问该内存时，数据只会缓存到core 0的L1 D-cache / L2 cache和 DSU L3 cache，不会缓存到System Cache中(当然如果有system cache的话 ) ， （注意这里MESI协议其作用了）此时core0的cache TAG中的MESI状态是E， 接着如果这个时候core1也去读该数据，那么数据也会被缓存core1的L1 D-cache / L2 cache 和DSU0的L3 cache(白字黑字，绝不瞎说，请参见文末的`[1] DSU TRM片段`)， 此时core0和core1的MESI状态都是S

- 如果将block的内存属性配置成 outer-shareable, 那么core0访问该内存时，数据会缓存到core 0的L1 D-cache / L2 cache 、cluster0的DSU L3 cache 、 System Cache中， core0的MESI状态为E。如果core1再去读的话，则也会缓存到core1的L1 D-cache / L2 cache，此时core0和core1的MESI都是S。这个时候，如果core7也去读的话，数据还会被缓存到cluster1的DSU L3 cache. 至于DSU0和DSU1之间的一致性，非MESI维护，具体怎么维护的请看DSU手册，本文不展开讨论。

||Non-cacheable|write-through  <br>cacheable|write-back  <br>cacheable|
|:--|:--|:--|:--|
|non-shareable|数据不会缓存到cache  <br>（对于观察则而言，又相当于outer-shareable）|core0访问该内存时，数据缓存的到Core0的L1 D-cache / L2 cache （将L1/L2看做一个整体，直接说数据会缓存到core0的private cache更好），不会缓存到其它cache中|同左侧|
|inner-shareable|数据不会缓存到cache  <br>（对于观察则而言，又相当于outer-shareable）|core0访问该内存时，数据只会缓存到core 0的L1 D-cache / L2 cache和 DSU L3 cache，不会缓存到System Cache中(当然如果有system cache的话 ) ， （注意这里MESI协议其作用了）此时core0的cache TAG中的MESI状态是E， 接着如果这个时候core1也去读该数据，那么数据也会被缓存core1的L1 D-cache / L2 cache 和DSU0的L3 cache， 此时core0和core1的MESI状态都是S|同左侧|
|outer-shareable|数据不会缓存到cache  <br>（对于观察则而言，又相当于outer-shareable）|core0访问该内存时，数据会缓存到core 0的L1 D-cache / L2 cache 、cluster0的DSU L3 cache 、 System Cache中， core0的MESI状态为E。如果core1再去读的话，则也会缓存到core1的L1 D-cache / L2 cache，此时core0和core1的MESI都是S  <br>思考:那么此时core7去读取会怎样？|同左侧|

#### 4、总结

- dynamIQ 架构中 L1和L2之间的替换策略，是由core的inclusive/exclusive的硬件特性决定的，软无法更改

- core cache之间的替换策略，是由SCU(或DSU)执行的MESI协议中定义的，软件也无法更改。

- cluster cache之间的替换策略，是由于MMU页表中的内存属性定义的(innor/outer/cacheable/shareable)，软件可以修改

______________________________________________________________________

#### 参考

- \[1\] DSU TRM片段\
  !\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

最新上架

☞[【课程】8天入门ARM架构](http://mp.weixin.qq.com/s?__biz=MzU4NDg4MzY3OA==&mid=2247493218&idx=1&sn=6e75e48d7443139d8e3a889e140c7bfa&chksm=fd91aad3cae623c5320add2c05cd012469ade81196ec0489882a51f10ca1fab8b08fc1147e94&scene=21#wechat_redirect)

☞[【课程】《8天入门Trustzone/TEE/安全架构》限时128元](http://mp.weixin.qq.com/s?__biz=MzU4NDg4MzY3OA==&mid=2247493212&idx=1&sn=8c6e210c1f51c6329874119c7bdcb7f1&chksm=fd91aaedcae623fbeaa5c3a2f65ad64798728fb683cb32a96a7be62e64deddd4bd4af5a29bb9&scene=21#wechat_redirect)

经典课程

☞【会员】[Arm精选课堂-铂金VIP介绍](http://mp.weixin.qq.com/s?__biz=MzU4NDg4MzY3OA==&mid=2247491516&idx=3&sn=ff9c68ec46e94a124748d31fb58174b2&chksm=fd92530dcae5da1bc5052d12f7bcdca099659bf380b2ed87f38062e56699d79ffa1acdcc04ca&scene=21#wechat_redirect) !\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

☞【课程】[《ARMv8/ARMv9架构从入门到精通》-二期 视频课程](http://mp.weixin.qq.com/s?__biz=MzU4NDg4MzY3OA==&mid=2247491518&idx=1&sn=387d78d7b276878e0ad595c57e19ef69&chksm=fd92530fcae5da1979d1346f7fdac7c3a475010068c3477a6fa9319bdee3416d7cb0208da78c&scene=21#wechat_redirect)  !\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

☞【课程】[Trustzone/TEE/安全从入门到精通 - 标配版 视频课程](https://mp.weixin.qq.com/s?__biz=MzU4NDg4MzY3OA==&mid=2247491516&idx=5&sn=56ce2413f056553cfeb5090a1ac01f9d&chksm=fd92530dcae5da1b7c04bdab9dfab485791a0ca6c2e7cb4a50579706d6a5d58d1149dee8906d&scene=21&token=56753480&lang=zh_CN#wechat_redirect) !\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

☞【课程】[Secureboot从入门到精通（二期）](http://mp.weixin.qq.com/s?__biz=MzU4NDg4MzY3OA==&mid=2247491516&idx=6&sn=f121f77f1513cc8d2da6b4ec422d4339&chksm=fd92530dcae5da1bab5b645f2a360c73ebe7b4c895ace567a03c6376ae8b66b2baf2d426fe40&scene=21#wechat_redirect) !\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

☞【课程】[CA/TA开发从入门到精通](http://mp.weixin.qq.com/s?__biz=MzU4NDg4MzY3OA==&mid=2247491581&idx=1&sn=37eb8e3c20abb6a23d92fca7b04b8836&chksm=fd92534ccae5da5a75f866927ada7ab9b120597d0ac8e127d7689ee9ce78950b672965a27098&scene=21#wechat_redirect)

☞【课程】[《ATF架构开发精讲》-视频课](http://mp.weixin.qq.com/s?__biz=MzU4NDg4MzY3OA==&mid=2247491529&idx=1&sn=eba2ff9b68b2d8b132bb93caf29854b9&chksm=fd925378cae5da6e490f270037c21a7520c4c5a1e4931b6ac24955cab3ef848c8c3a4b6bccc3&scene=21#wechat_redirect)

☞【课程】[《optee系统开发精讲》-视频课](http://mp.weixin.qq.com/s?__biz=MzU4NDg4MzY3OA==&mid=2247491540&idx=1&sn=6f78af5f2463ab4b7e243d661c27ba60&chksm=fd925365cae5da73a7bfd7dd69b8222dca3f548258aa9cdbe32929286bb1a9393508077414ba&scene=21#wechat_redirect)

☞【课程】[ATF/optee/hafnium/linux/xen代码精读](http://mp.weixin.qq.com/s?__biz=MzU4NDg4MzY3OA==&mid=2247491571&idx=1&sn=eda514ebff87115e53db4ecd3f369ba0&chksm=fd925342cae5da5446e5ca5fdf9d38a357c6d078b2317a66abd9df488cad9ba4875f6ca53992&scene=21#wechat_redirect)

☞[【课程】HSM/HSE/SHE/SE/Crypto engine大扫盲](http://mp.weixin.qq.com/s?__biz=MzU4NDg4MzY3OA==&mid=2247491614&idx=1&sn=c6513342b0176d9dc9dabe0323edc939&chksm=fd91acafcae625b9dbdeb208c12efc1c496e79fd2189e86c043ae260490e1013c1f3981eea39&scene=21#wechat_redirect)

[☞【课程】Keystore/keymaster/keymint的介绍](http://mp.weixin.qq.com/s?__biz=MzU4NDg4MzY3OA==&mid=2247491635&idx=1&sn=941da5e4b5d712ac5c7a0484a645bd90&chksm=fd91ac82cae6259467453b5299e77c28d50b3091bb76d72fd5bde6354c06fd7b5a8847745cf6&scene=21#wechat_redirect)

[☞【课程】Arm微架构课和Arm MMU/SMMU讨论课](http://mp.weixin.qq.com/s?__biz=MzU4NDg4MzY3OA==&mid=2247491665&idx=1&sn=de03b52689a8a1221db41751607a4282&chksm=fd91ace0cae625f641204b17e8f36aad5bbca282793f9894fcd8ba408b16e9b27ea8c5119e32&scene=21#wechat_redirect)

[☞【课程】TEE扫盲课程-TEE/安全面试课程](http://mp.weixin.qq.com/s?__biz=MzU4NDg4MzY3OA==&mid=2247491654&idx=1&sn=74131b259db8677f70266e755e52829f&chksm=fd91acf7cae625e122a4e7343dbe51a3956944e3e19c569e4abf67ccb3527eab511fff2ad9e8&scene=21#wechat_redirect)

☞【指导】[\[指南\]  观看我们课程的4种方式](http://mp.weixin.qq.com/s?__biz=MzU4NDg4MzY3OA==&mid=2247490740&idx=2&sn=a1dd6e971b5b18e854d313647e1ecdc3&chksm=fd925005cae5d913e1c3e2a91cc9387814107cd088c1a80f38c477d58dd0abce0ad2ff437cf3&scene=21#wechat_redirect)

☞【指导】[Arm视频课程一期和二期有什么区别？](http://mp.weixin.qq.com/s?__biz=MzU4NDg4MzY3OA==&mid=2247490740&idx=3&sn=35aba1e36c80f8b44da83ab81fadca0d&chksm=fd925005cae5d91308716aae78ee6a5816e1c8fbc4edb4f9221a01fae39aa1ba751f6feda7eb&scene=21#wechat_redirect)

☞【指导】[Trustzone/TEE/安全从入门到精通--回放版、标准版、高配版有什么区别](http://mp.weixin.qq.com/s?__biz=MzU4NDg4MzY3OA==&mid=2247490740&idx=4&sn=f619d3d64007dd0be73799b7ea0614fb&chksm=fd925005cae5d91325b8d3f014e0e2f32bc70e674e525c4977b1c8563f50262187f7173fdf71&scene=21#wechat_redirect)

#### 【店铺地址】

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

#### 【客服咨询】

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

![](http://mmbiz.qpic.cn/sz_mmbiz_png/72OMRpZ5hV00GOqhBiag6YJIfArytJkI1xGDuAwB6cumchNVevDk9T1PvfhVdicTkFge7XpJy6mvTJT2YFYzGYnw/300?wx_fmt=png&wxfrom=19)

**Arm精选**

ARMv8/ARMv9架构、SOC架构、Trustzone/TEE安全、终端安全、SOC安全、ARM安全、ATF、OPTEE等

591篇原创内容

公众号

阅读 606

​

发消息

[](javacript:;)

![](http://mmbiz.qpic.cn/sz_mmbiz_png/72OMRpZ5hV00GOqhBiag6YJIfArytJkI1xGDuAwB6cumchNVevDk9T1PvfhVdicTkFge7XpJy6mvTJT2YFYzGYnw/300?wx_fmt=png&wxfrom=18)

Arm精选

赞74在看

发消息
