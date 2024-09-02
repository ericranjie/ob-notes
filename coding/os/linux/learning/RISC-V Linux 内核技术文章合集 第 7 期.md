
泰晓科技

 _2023年06月08日 20:09_ _广东_

## 更新时间：2023.06.04  

## 简介

自 2022 年 3 月份开始，泰晓科技 Linux 技术社区启动了一项 “**RISC-V Linux 内核技术调研活动**”，致力于 RISC-V Linux 内核及周边技术生态的建设：

- 首页
    

- https://tinylab.org/riscv-linux
    

- 仓库
    

- https://gitee.com/tinylab/riscv-linux
    

该活动持续产出了多方面的成果，包括技术分析文章、在线技术直播 & 视频分享以及各类提交进上游社区的 patch，也陆续孵化了多个开源项目。

- 文章
    

- https://tinylab.org
    

- 视频
    

- https://space.bilibili.com/687228362
    

- 项目
    

- https://gitee.com/tinylab
    

为方便读者查阅，本文将持续汇总已发表到公众号的文章合集，可关注公众号收看后续更新，初步计划每个月更新一期，敬请期待！

## 文章合集

### 技术总结

每周二和周四连载 RISC-V Linux 内核技术调研活动中的技术文章成果。截止于 2023.06.04，累计已评审并合并 136 篇，等待陆续连载中。

  

- 硬件规范
    

- [RISC-V ISA Spec 简介](http://mp.weixin.qq.com/s?__biz=MzA5NDQzODQ3MQ==&mid=2648189312&idx=1&sn=9e1df15640e8f29c5769841cb29db8a3&chksm=88621ca8bf1595be07164e3ca8671c8d5c35712187dfbe742eb105f8cf16804eb82316a4ceaf&scene=21#wechat_redirect)
    
- [RISC-V 特权指令](http://mp.weixin.qq.com/s?__biz=MzA5NDQzODQ3MQ==&mid=2648189343&idx=1&sn=c2100572653ca940f0b7de99437465fc&chksm=88621cb7bf1595a116db19224b2b0b88168ca0a62aa8d065f85288c5e57d082b0fa0d476f256&scene=21#wechat_redirect)
    
- [](http://mp.weixin.qq.com/s?__biz=MzA5NDQzODQ3MQ==&mid=2648189343&idx=1&sn=c2100572653ca940f0b7de99437465fc&chksm=88621cb7bf1595a116db19224b2b0b88168ca0a62aa8d065f85288c5e57d082b0fa0d476f256&scene=21#wechat_redirect)[RISC-V 原子指令介绍](http://mp.weixin.qq.com/s?__biz=MzA5NDQzODQ3MQ==&mid=2648188801&idx=1&sn=56d1be783eaa12210a227d3edc05f8c4&chksm=886212a9bf159bbf966f9e6b2d6a5a053eedc28ecf8a738a95d534e225d2c24e5c2d09b8693d&scene=21#wechat_redirect)
    

- 引导启动
    

- [RISC-V OpenSBI 快速上手](http://mp.weixin.qq.com/s?__biz=MzA5NDQzODQ3MQ==&mid=2648188949&idx=1&sn=81bb8fbc8c11f7ece88853a0d68d3302&chksm=88621d3dbf15942b9348af18ab1b0ebb9fb8d72bc36ccc00b1f42e97a20b9487ed6aab5d581a&scene=21#wechat_redirect)
    
- [RISC-V UEFI 架构支持详解，第 1 部分 - OpenSBI/U-Boot/UEFI 简介](http://mp.weixin.qq.com/s?__biz=MzA5NDQzODQ3MQ==&mid=2648188975&idx=1&sn=275dd87264b365ca42c214a8f9d8935b&chksm=88621d07bf159411f502eb73bb04af5242c65186a094bd72894c938d5c1f21cd953466707b8e&scene=21#wechat_redirect)
    
- [RISC-V Linux 启动流程分析](http://mp.weixin.qq.com/s?__biz=MzA5NDQzODQ3MQ==&mid=2648189198&idx=1&sn=953f922c6b5b199bf24a37832b45da0b&chksm=88621c26bf159530c42609e16981c5ea2453b6246873ca465bf6be2c923d3d4c9dabf26c8483&scene=21#wechat_redirect)
    

- 内存管理
    

- [RISC-V Linux SPARSEMEM 介绍与分析](http://mp.weixin.qq.com/s?__biz=MzA5NDQzODQ3MQ==&mid=2648189875&idx=1&sn=5ea70689d37457d45ec461599203abe6&chksm=88621e9bbf15978dd232a6f0becc95d62277983944b6ac728a27f4fe6ae64f3b0f691b6c2354&scene=21#wechat_redirect)
    
- [memblock 内存分配器原理和代码分析](http://mp.weixin.qq.com/s?__biz=MzA5NDQzODQ3MQ==&mid=2648190012&idx=1&sn=aebf95cebb93f37ca1b34000d00236b1&chksm=88621914bf1590025e5ab2e24432b5668699612ab5c888c33bf2953180e0c1e1b4a806a028d3&scene=21#wechat_redirect)
    
- [RISCV MMU 概述](http://mp.weixin.qq.com/s?__biz=MzA5NDQzODQ3MQ==&mid=2648189712&idx=1&sn=34af5147eef1a8de2c4f7ffe929befa9&chksm=88621e38bf15972e37707f3520ab98003109e3460b7cdfce71814f7c7b8bdcf478ced3e1171b&scene=21#wechat_redirect)
    
- [多代 LRU（Multi-Gen LRU）文档翻译](http://mp.weixin.qq.com/s?__biz=MzA5NDQzODQ3MQ==&mid=2648191562&idx=1&sn=bd6536d7566b7f710f20c21a050100a1&chksm=88620762bf158e74cc433bef6ac7f3083c006c30f38ea35954f5971f50e48c795d7ff5c06ae9&scene=21#wechat_redirect)
    
- [](http://mp.weixin.qq.com/s?__biz=MzA5NDQzODQ3MQ==&mid=2648191604&idx=1&sn=0f3d15fc35af5fb1a47ed69e54491ab2&chksm=8862075cbf158e4aa9ed73befba316c72c5b6860c4968ea4fec22d2f3f36b2a577ed90c88f47&scene=21#wechat_redirect)[RISC-V 缺页异常处理程序分析（1）：do_page_fault() 和 handle_mm_fault()](http://mp.weixin.qq.com/s?__biz=MzA5NDQzODQ3MQ==&mid=2648191604&idx=1&sn=0f3d15fc35af5fb1a47ed69e54491ab2&chksm=8862075cbf158e4aa9ed73befba316c72c5b6860c4968ea4fec22d2f3f36b2a577ed90c88f47&scene=21#wechat_redirect)
    
- [RISC-V 缺页异常处理程序分析（2）：handle_pte_fault() 和 do_anonymous_page()](http://mp.weixin.qq.com/s?__biz=MzA5NDQzODQ3MQ==&mid=2648191670&idx=1&sn=5862a15317c3d3634807a8152501d9e8&chksm=8862079ebf158e88222fb523bc69787126623cc3c366d441dba05a7cf9e59cfaddf4cc17ffd1&scene=21#wechat_redirect)  
    
- [RISC-V 缺页异常处理程序分析（3）：文件映射缺页异常分析](http://mp.weixin.qq.com/s?__biz=MzA5NDQzODQ3MQ==&mid=2648191691&idx=1&sn=6b5c5822a8baa0f785921d3a730c21c5&chksm=886207e3bf158ef54c23963de76f2b5cd2612bb0d57860fc7afde141fb6a3d3bc0d57a5a1e69&scene=21#wechat_redirect)  
    

- 进程调度
    

- [RISC-V 架构下内核线程返回函数探究](http://mp.weixin.qq.com/s?__biz=MzA5NDQzODQ3MQ==&mid=2648189652&idx=1&sn=b060b23e5be503e8c354dad80548b9fc&chksm=88621ffcbf1596ea95029d69bda8cd17a09c1a2e59ea3b94c4605df43d16a27279caffac6d1a&scene=21#wechat_redirect)  
    
- [RISC-V Linux 进程创建与执行流程代码分析](http://mp.weixin.qq.com/s?__biz=MzA5NDQzODQ3MQ==&mid=2648189532&idx=1&sn=0d52b4a4061231d7dfec8da56fd0a9e2&chksm=88621f74bf15966219f8e6ccc5a13135164b8ad099f5ceb3815ec0528336d29f99c72e2ffc61&scene=21#wechat_redirect)
    
- [RISC-V Linux 上下文切换分析](http://mp.weixin.qq.com/s?__biz=MzA5NDQzODQ3MQ==&mid=2648189393&idx=1&sn=f33a5d8ab33df9f4b07818faca08a17e&chksm=88621cf9bf1595ef7fd4fbc5f519b2934c351e43fce0a8fce8e3889aa2cb704e6309e749615d&scene=21#wechat_redirect)
    
- [RISC-V Linux Schedule 分析](http://mp.weixin.qq.com/s?__biz=MzA5NDQzODQ3MQ==&mid=2648189444&idx=1&sn=9748aae47d6ed8082c63a1db41b9cf52&chksm=88621f2cbf15963acd474ca612461d0f8fcbf9ec66935c407f1828ceec995636463c8fb0862d&scene=21#wechat_redirect)  
    

- 中断管理
    

- [Generic entry RISC-V 补丁分析](http://mp.weixin.qq.com/s?__biz=MzA5NDQzODQ3MQ==&mid=2648190616&idx=1&sn=4278eb77da95e9aff8e741cb8077b09e&chksm=88621bb0bf1592a67916aa7b2aaf50037c367c1f338ba0e61e20e2dfbc1c4df1d09144e4b119&scene=21#wechat_redirect)  
    
- [RISC-V 中断子系统分析——硬件及其初始化](http://mp.weixin.qq.com/s?__biz=MzA5NDQzODQ3MQ==&mid=2648190213&idx=1&sn=3b2ea5a9c9d2db2624eaded60e51ce33&chksm=8862182dbf15913b782cfe6f6e053a34cf0e6573116907b39f4d70d0b93701e5b4a673571e1d&scene=21#wechat_redirect)
    
- [RISC-V 中断子系统分析——PLIC 中断处理](http://mp.weixin.qq.com/s?__biz=MzA5NDQzODQ3MQ==&mid=2648190386&idx=1&sn=5495280cc913c3a757f59046ff9fca48&chksm=8862189abf15918c345f30c09302b7c8818aa8133b76c1088bd20367542a1b9b52f31938e403&scene=21#wechat_redirect)
    
- [RISC-V 中断子系统分析——CPU 中断处理](http://mp.weixin.qq.com/s?__biz=MzA5NDQzODQ3MQ==&mid=2648190445&idx=1&sn=d2e45a9a64ca1286f9e28fb014a83d93&chksm=886218c5bf1591d3caad7eedeb0e7f8842d164a0c5738d0dc61935f57b9355e563c57da9b851&scene=21#wechat_redirect)
    
- [RISC-V 中断子系统分析——中断优先级](http://mp.weixin.qq.com/s?__biz=MzA5NDQzODQ3MQ==&mid=2648190485&idx=1&sn=dc2a498682d44ea2622c024ad720c786&chksm=88621b3dbf15922b7aa862007e2bfe6c1a1e2b92079eb0628c1b97a49e5f6d1094781a11a538&scene=21#wechat_redirect)
    

- 异常处理
    

- [RISC-V 异常处理流程介绍](http://mp.weixin.qq.com/s?__biz=MzA5NDQzODQ3MQ==&mid=2648190518&idx=1&sn=8482197c95f893877d0ee4e6d4772a17&chksm=88621b1ebf159208b04da7983f526f4649c425ef422d19ba0979ae9d48a56ac33303fd572ff5&scene=21#wechat_redirect)
    

- 系统调用
    

- [RISC-V Syscall 系列 1：什么是 Syscall ?](http://mp.weixin.qq.com/s?__biz=MzA5NDQzODQ3MQ==&mid=2648189907&idx=1&sn=608e932a5e4fa2bb0228518bce64f301&chksm=88621efbbf1597ed94a85c0b4b478de828ae917a2f670538c2f86586b2a7b7db07bb61ece0e4&scene=21#wechat_redirect)
    
- [RISC-V Syscall 系列 2：Syscall 过程分析](http://mp.weixin.qq.com/s?__biz=MzA5NDQzODQ3MQ==&mid=2648190062&idx=1&sn=33c0ea5882bb691cbd4ea2d9a4f01a77&chksm=88621946bf159050a0d5cff4da08422b3a5a097cd10635a701ca8c5a4e1d83b8c4a78c0a2b47&scene=21#wechat_redirect)
    
- [RISC-V Syscall 系列 3：什么是 vDSO？](http://mp.weixin.qq.com/s?__biz=MzA5NDQzODQ3MQ==&mid=2648190146&idx=1&sn=fe036748694d39060e50348ec9d2d8d5&chksm=886219eabf1590fcae7c71e808e086fec17f1fa4eb31417af01bb6658224f4bd5f262d7d86e1&scene=21#wechat_redirect)
    
- [RISC-V Syscall 系列 4：vDSO 实现原理分析](http://mp.weixin.qq.com/s?__biz=MzA5NDQzODQ3MQ==&mid=2648190267&idx=1&sn=3b116b59465fa95d78a299dede9790ad&chksm=88621813bf15910531e9f66abff4aadc90320b433316460b149fccd3949be5b20f37393f3f41&scene=21#wechat_redirect)
    

- 时钟管理
    

- [RISC-V timer 在 Linux 中的实现](http://mp.weixin.qq.com/s?__biz=MzA5NDQzODQ3MQ==&mid=2648189049&idx=1&sn=9bc2a2e9bb889b887c526c9d75b11762&chksm=88621d51bf1594478731e1e8362a85a8445d281c17b0d490e35cae4e981e6075ad1a202fb75b&scene=21#wechat_redirect)
    

- 内核同步
    
- SMP支持  
    
- 跟踪调试
    

- [RISC-V Linux Stacktrace 详解](http://mp.weixin.qq.com/s?__biz=MzA5NDQzODQ3MQ==&mid=2648188861&idx=1&sn=033f832978a5c9f58b6ad09343c1205d&chksm=88621295bf159b83d748a1547f5e2c3909d6fd8285b01810edc5ecdab0a5ab5a4df6ebdaf9ef&scene=21#wechat_redirect)
    
- [Linux Kfence 详解](http://mp.weixin.qq.com/s?__biz=MzA5NDQzODQ3MQ==&mid=2648189086&idx=1&sn=5efd987b59805421e630b4652223ec31&chksm=88621db6bf1594a0eca937cc5e536118050daf15281bbf97a13d397d761b5564fa809780d342&scene=21#wechat_redirect)
    
- [](http://mp.weixin.qq.com/s?__biz=MzA5NDQzODQ3MQ==&mid=2648191175&idx=1&sn=91389b133ed05af185844c6bf4c6876f&chksm=886205efbf158cf9995e08a561de56e8d83bd95c06afdef7c37ad18d411186369892d4a5a0d6&scene=21#wechat_redirect)[RISC-V Ftrace 实现原理（1）- 函数跟踪](http://mp.weixin.qq.com/s?__biz=MzA5NDQzODQ3MQ==&mid=2648191175&idx=1&sn=91389b133ed05af185844c6bf4c6876f&chksm=886205efbf158cf9995e08a561de56e8d83bd95c06afdef7c37ad18d411186369892d4a5a0d6&scene=21#wechat_redirect)
    
- [RISC-V Ftrace 实现原理（2）- 编译时原理](http://mp.weixin.qq.com/s?__biz=MzA5NDQzODQ3MQ==&mid=2648191225&idx=1&sn=62fcc1d1f524d290316ab5f06151f797&chksm=886205d1bf158cc7496a385d14ffaab196a52ca735966cac3d867288e718ec8037a391754a54&scene=21#wechat_redirect)
    
- [RISC-V Ftrace 实现原理（3）- 替换函数入口](http://mp.weixin.qq.com/s?__biz=MzA5NDQzODQ3MQ==&mid=2648191255&idx=1&sn=3557f699d400a4e1c443031f3cfe991c&chksm=8862043fbf158d29fdce638fe3e1cb5f1776a1824bb872e07aefe9e6269a62940e1517ea1d39&scene=21#wechat_redirect)
    
- [RISC-V Ftrace 实现原理（4）- 替换跟踪函数](http://mp.weixin.qq.com/s?__biz=MzA5NDQzODQ3MQ==&mid=2648191275&idx=1&sn=0fc4ad3ba8196e2c05047aea5442eeef&chksm=88620403bf158d154d61db66c003bd201100fc3109ff4dd44fc0a25b24a7c811b01195d727c1&scene=21#wechat_redirect)
    
- [RISC-V Ftrace 实现原理（5）- 动态函数图跟踪](http://mp.weixin.qq.com/s?__biz=MzA5NDQzODQ3MQ==&mid=2648191311&idx=1&sn=f4852b9825fe8b9286cd8fd56be8efeb&chksm=88620467bf158d71d6378a245b26cae6de6a709333ae2e08020b352bf1f8175e24a8268833dd&scene=21#wechat_redirect)
    
- [](http://mp.weixin.qq.com/s?__biz=MzA5NDQzODQ3MQ==&mid=2648191358&idx=1&sn=b0c8405380dbc561873a954a586a99c7&chksm=88620456bf158d40d2a0579fcc499b164b02b4d2a9127f40065567955a22ed502c51c32d8e00&scene=21#wechat_redirect)[RISC-V Ftrace 实现原理（6）- trace ring buffer](http://mp.weixin.qq.com/s?__biz=MzA5NDQzODQ3MQ==&mid=2648191358&idx=1&sn=b0c8405380dbc561873a954a586a99c7&chksm=88620456bf158d40d2a0579fcc499b164b02b4d2a9127f40065567955a22ed502c51c32d8e00&scene=21#wechat_redirect)
    
- [RISC-V Ftrace 实现原理（7）- RISC-V 架构总结](http://mp.weixin.qq.com/s?__biz=MzA5NDQzODQ3MQ==&mid=2648191391&idx=1&sn=f0d1f14b81e8ce36e7e8e8795009cdf1&chksm=886204b7bf158da1d97290accf5de1786fa0316696ea5a8425c1d9998a301c1766817f51df61&scene=21#wechat_redirect)  
    

- 内核优化
    

- [RISC-V jump_label 详解，第 1 部分：技术背景](http://mp.weixin.qq.com/s?__biz=MzA5NDQzODQ3MQ==&mid=2648189397&idx=1&sn=6680c64b4491cf661a68c1bf2397e5fa&chksm=88621cfdbf1595eb1cf830b49eb41cd77dc30b07aec3b43fca0b684db47cffe38d35cbd32bb5&scene=21#wechat_redirect)
    
- [RISC-V jump_label 详解，第 2 部分：指令编码](http://mp.weixin.qq.com/s?__biz=MzA5NDQzODQ3MQ==&mid=2648189478&idx=1&sn=e9bddd04aab6725efd24ada286fea4a0&chksm=88621f0ebf159618f2e02c6bc3afeb2863bd7d1854daf7aeb2e136c1085a06e65e6fa7c587a6&scene=21#wechat_redirect)
    
- [RISC-V jump_label 详解，第 3 部分：核心实现](http://mp.weixin.qq.com/s?__biz=MzA5NDQzODQ3MQ==&mid=2648189600&idx=1&sn=ea199628fa06b5028aaad672797da755&chksm=88621f88bf15969eb998c36a282a2bb5f11c31db43b96b22ea2b665bcebacc56476e3097ab6c&scene=21#wechat_redirect)
    
- [RISC-V jump_label 详解，第 4 部分：运行时代码改写](http://mp.weixin.qq.com/s?__biz=MzA5NDQzODQ3MQ==&mid=2648189680&idx=1&sn=c8cb7cd15f0079b18a68e90381f984a0&chksm=88621fd8bf1596cecfcc1c86f40f27ca07b4bd46c25487517d88c570ac4fb6c60968bc093870&scene=21#wechat_redirect)
    
- [RISC-V jump_label 详解，第 5 部分：优化案例](http://mp.weixin.qq.com/s?__biz=MzA5NDQzODQ3MQ==&mid=2648189778&idx=1&sn=633afd62dc8ef36ef80ed335c0a03647&chksm=88621e7abf15976ca3c793f139c673c8889438e3433612df147bcf08188d3f0e44be6cfee05a&scene=21#wechat_redirect)  
    

- 内核裁剪
    
- 内核移植
    

- [将 Linux 移植到新的处理器架构，第 1 部分：基础](http://mp.weixin.qq.com/s?__biz=MzA5NDQzODQ3MQ==&mid=2648188881&idx=1&sn=27a95fc92f8e1ab2ae348b1c2450aa83&chksm=886212f9bf159befcdfcaaf9cde059a4911ae760840e746a28d1928a76f9a10eb51a100cac97&scene=21#wechat_redirect)
    
- [将 Linux 移植到新的处理器架构，第 2 部分：早期代码](http://mp.weixin.qq.com/s?__biz=MzA5NDQzODQ3MQ==&mid=2648188920&idx=1&sn=f9dabddca29726a418823c2998d17121&chksm=886212d0bf159bc6f63eaa283951cee0793e9fd81d73d0f1f697bb7f12cd0a126aa992806124&scene=21#wechat_redirect)
    
- [将 Linux 移植到新的处理器架构，第 3 部分：收尾](http://mp.weixin.qq.com/s?__biz=MzA5NDQzODQ3MQ==&mid=2648188935&idx=1&sn=95e64ce9926ee03172dd491b4f131950&chksm=88621d2fbf159439588291bba4b5ea6ff133d31535b5a3f8f0e50d5d0a43270b4376c71975c0&scene=21#wechat_redirect)  
    

- 虚拟化
    

- [从嵌入式系统视角初次展望 RISC-V 虚拟化](http://mp.weixin.qq.com/s?__biz=MzA5NDQzODQ3MQ==&mid=2648191732&idx=1&sn=7248ea266e49f9dcf20d1155bc80656d&chksm=886207dcbf158eca7d0bbf98621e2fe267a85fbb9608ab7a2de2785667e5c8d5f0f933297f01&scene=21#wechat_redirect)
    
- [用 QEMU/Spike+KVM 运行 Host/Guest Linux](http://mp.weixin.qq.com/s?__biz=MzA5NDQzODQ3MQ==&mid=2648191777&idx=1&sn=748db64548de13a2dc77be846d0b5363&chksm=88620609bf158f1fd2fa471c2cf84f6f43465da280866b02090d66e559742c26b12728af6b8c&scene=21#wechat_redirect)
    
- [RISC-V 虚拟化模式切换简析](http://mp.weixin.qq.com/s?__biz=MzA5NDQzODQ3MQ==&mid=2648191808&idx=1&sn=fbbab40d792e15c99afedfa3ef8c02af&chksm=88620668bf158f7e2e7e9efb3f71ce1cd85cdadcca613946593c9d3cc38e538ee2df747393a0&scene=21#wechat_redirect)  
    
- [RISC-V KVM 虚拟化：用户态程序](http://mp.weixin.qq.com/s?__biz=MzA5NDQzODQ3MQ==&mid=2648191858&idx=1&sn=3b8432625dc3f983ce675040bd41a989&chksm=8862065abf158f4cbe211b654fd9035a920620894def8333310bdf3379953d72ac7fa81d1d0f&scene=21#wechat_redirect)  
    
- [RISC-V 内存虚拟化简析（一）](http://mp.weixin.qq.com/s?__biz=MzA5NDQzODQ3MQ==&mid=2648191893&idx=1&sn=7409544007eab15c36d9adf0f2df7eb5&chksm=886206bdbf158fab44822c9f0475614e4324e48b0acb356f52658ddca91fb6609cbca0f4a40d&scene=21#wechat_redirect)  
    
- [RISC-V 内存虚拟化简析（二）](http://mp.weixin.qq.com/s?__biz=MzA5NDQzODQ3MQ==&mid=2648191937&idx=1&sn=ac392ab23aaa620209953c33358f1a99&chksm=886206e9bf158fff1fea173ead81bdbaf04de787cc90c3e4dbea3d48978dfceda767c07c4a57&scene=21#wechat_redirect)  
    
- [RISC-V 架构 H 扩展中的 Trap 处理](http://mp.weixin.qq.com/s?__biz=MzA5NDQzODQ3MQ==&mid=2648191966&idx=1&sn=759371d427c42b2cb99a2e5749fbc1c6&chksm=886206f6bf158fe065151e35404d5c5757e8585d20bcfa024597a4f80f2eb5ba0f758d2c8a35&scene=21#wechat_redirect)  
    
- [RISC 内存虚拟化在 KVM 及 kvmtool 中的实现](http://mp.weixin.qq.com/s?__biz=MzA5NDQzODQ3MQ==&mid=2648191997&idx=1&sn=89a55a44af0e896b6d783379ba94be38&chksm=886206d5bf158fc39b5a7eba4bb49a8f22d8f06e4ee86758f9ebd43d7e3ba3aa11d7bc4d6dfe&scene=21#wechat_redirect)  
    

- 实时性
    
- 安全性
    
- Benchmark
    

- [当前业界 RISC-V 处理器指令级性能揭秘](http://mp.weixin.qq.com/s?__biz=MzA5NDQzODQ3MQ==&mid=2648189071&idx=1&sn=120c54897e1f5b433029c591c07869fc&chksm=88621da7bf1594b107e2a9b655129c792062d1eeca305a335e310c520fd1c48e6816fec6020b&scene=21#wechat_redirect)  
    

- Testing
    
- Linux 发行版
    

- [两分钟内体验 RISC-V Linux 系统发行版](http://mp.weixin.qq.com/s?__biz=MzA5NDQzODQ3MQ==&mid=2648188993&idx=1&sn=04ffc9da5ddbd2e993edb9bbf42c1fc1&chksm=88621d69bf15947f09012baf06d70bdbded9ddda248cd2fe24532c4bf188638cb82a1ad186c1&scene=21#wechat_redirect)
    
- [5 秒内跨架构运行 RISC-V Ubuntu 22.04+xfce4](http://mp.weixin.qq.com/s?__biz=MzA5NDQzODQ3MQ==&mid=2648189474&idx=1&sn=d59bdd85e9decc31d28f860832f431c9&chksm=88621f0abf15961c2c804e1fd88ff5224b2f0563a061385e572c67aec0e934abaf8713809fb3&scene=21#wechat_redirect)
    

- 嵌入式 Linux
    

- [使用 buildroot 构建 QEMU 和哪吒开发板的系统镜像](http://mp.weixin.qq.com/s?__biz=MzA5NDQzODQ3MQ==&mid=2648191479&idx=1&sn=4c48bd5fb73c9339f260991af50c40cc&chksm=886204dfbf158dc9201ffa53f07cb10fc7ffb5e5bc4fdcde539076c7eaf9fe2baf1ff4d14c8b&scene=21#wechat_redirect)
    
- [使用 Bitbake 和 OpenEmbedded 构建运行在 D1-H 哪吒开发板的软件](http://mp.weixin.qq.com/s?__biz=MzA5NDQzODQ3MQ==&mid=2648191448&idx=1&sn=adf185475a20f5b2e8fbf17bac750ddc&chksm=886204f0bf158de6624e318f3573ac9ee07bfb8cbee90157deb29201385b7197f5788b6ada78&scene=21#wechat_redirect)
    
- [使用 Bitbake 和 OpenEmbedded 构建运行在 RISC-V 的系统](http://mp.weixin.qq.com/s?__biz=MzA5NDQzODQ3MQ==&mid=2648191408&idx=1&sn=63fc45468a2656b336d3a84e12d6baf5&chksm=88620498bf158d8e5c6ac304858897b0f15c0816ab183e931fb36f7e83eb542879a866488e18&scene=21#wechat_redirect)
    
      
    

- CPU 设计
    
- Emulator
    

- [用纯 C 语言写一个简单的 RISC-V 模拟器（支持基础整数指令集，乘法指令集与 CSR 指令）](http://mp.weixin.qq.com/s?__biz=MzA5NDQzODQ3MQ==&mid=2648190704&idx=1&sn=fae537ff9487c18071cc7e2bb14d302a&chksm=88621bd8bf1592ce36193ec69f5cffabc0b0ed2c0f4687ef162c751370108a92747bf0d86a62&scene=21#wechat_redirect)
    
- [QEMU 启动方式分析（1）：QEMU 及 RISC-V 启动流程简介](http://mp.weixin.qq.com/s?__biz=MzA5NDQzODQ3MQ==&mid=2648190785&idx=1&sn=1ea576bb60d17ce8627ef6777e69e8de&chksm=88621a69bf15937fc5b7fa2a8ee1e194eaca15e2f0718f407f08f8d1c00e96a9f50444c0edfe&scene=21#wechat_redirect)
    
- [](http://mp.weixin.qq.com/s?__biz=MzA5NDQzODQ3MQ==&mid=2648190898&idx=1&sn=d4969d7f03d713147aafd02681a5b570&chksm=88621a9abf15938cc55f66bca5c155b06ef5730c19a1d136f5157829d11bc043f0dec9782aef&scene=21#wechat_redirect)[QEMU 启动方式分析（2）: QEMU virt平台下通过OpenSBI+U-Boot引导 RISCV64 Linux 内核](http://mp.weixin.qq.com/s?__biz=MzA5NDQzODQ3MQ==&mid=2648190898&idx=1&sn=d4969d7f03d713147aafd02681a5b570&chksm=88621a9abf15938cc55f66bca5c155b06ef5730c19a1d136f5157829d11bc043f0dec9782aef&scene=21#wechat_redirect)
    
- [QEMU 启动方式分析（3）: QEMU 代码与 RISCV virt 平台 ZSBL 分析](http://mp.weixin.qq.com/s?__biz=MzA5NDQzODQ3MQ==&mid=2648190978&idx=1&sn=de7d11c9e0eadb5e373c46705a81641e&chksm=8862052abf158c3ca8cfadfa5c6de02e29dd9ff171c80312e229fdab0192eb760267daf8b2c8&scene=21#wechat_redirect)
    
- [QEMU 启动方式分析（4）: OpenSBI 固件分析与 SBI 规范的 HSM 扩展](http://mp.weixin.qq.com/s?__biz=MzA5NDQzODQ3MQ==&mid=2648191090&idx=1&sn=ddab73b9126417ad1dcc1ffe731b2013&chksm=8862055abf158c4cc95c30d5fec477253a3c0ed6f99d5e40fa3b2cfa66be3a1748330cd6afaf&scene=21#wechat_redirect)  
    

- 硬件设计  
    
- AI 开发
    

- [RISC-V AI 开发：D1 开机入门](http://mp.weixin.qq.com/s?__biz=MzA5NDQzODQ3MQ==&mid=2648190732&idx=1&sn=3f873465b9e1af27cf031efbf0af1aa6&chksm=88621a24bf1593327ad634568ebb2868762699e81b92725c0d053c8421150cc2bfa7e32c7be7&scene=21#wechat_redirect)
    
- [RISC-V AI 开发：用 D1 进行图片采集和人体识别](http://mp.weixin.qq.com/s?__biz=MzA5NDQzODQ3MQ==&mid=2648190862&idx=1&sn=89304b1c4446cd7eecce59eabb071f46&chksm=88621aa6bf1593b0d5447a5e49fcd9721f9dd7c15f823fc2317b4e16744fbba7f3cab1f53a25&scene=21#wechat_redirect)
    
- [RISC-V AI 开发：使用 ffmpeg 和 D1 开发板进行直播推流](http://mp.weixin.qq.com/s?__biz=MzA5NDQzODQ3MQ==&mid=2648190910&idx=1&sn=524c4c0c4d129baa054aeddb796200a1&chksm=88621a96bf159380038e754dbfe25d6c8cda22b8a842d4d87499b89a0d63f5c772e703ff780b&scene=21#wechat_redirect)  
    
- [](http://mp.weixin.qq.com/s?__biz=MzA5NDQzODQ3MQ==&mid=2648190988&idx=1&sn=4550208060d7b1d4775ca1a59fdca9d2&chksm=88620524bf158c32b3eec2e9b7606d0648dc1562ff60225204d16f3d04806723bd2bb3818970&scene=21#wechat_redirect)[RISC-V AI 开发：D1 开发板实时人物检测推流的功能实现](http://mp.weixin.qq.com/s?__biz=MzA5NDQzODQ3MQ==&mid=2648190988&idx=1&sn=4550208060d7b1d4775ca1a59fdca9d2&chksm=88620524bf158c32b3eec2e9b7606d0648dc1562ff60225204d16f3d04806723bd2bb3818970&scene=21#wechat_redirect)
    

- 待开发项
    

- [RISC-V 缺失的 Linux 内核功能](http://mp.weixin.qq.com/s?__biz=MzA5NDQzODQ3MQ==&mid=2648190400&idx=1&sn=3a337384c618332c66d27a08ae9300d5&chksm=886218e8bf1591fee38ec0ad9f4574d568e9e440e93dbd00011caad0ea54fc8c487a272a4699&scene=21#wechat_redirect)
    
- [](http://mp.weixin.qq.com/s?__biz=MzA5NDQzODQ3MQ==&mid=2648191577&idx=1&sn=41479618e4ff5eb918356a21235bd728&chksm=88620771bf158e673966b1a58a33a23a04b361577fd80a2c7dd6a5136eb3e0ac9ebd8a1d95b4&scene=21#wechat_redirect)[RISC-V 缺失的 Linux 内核功能-Part2](http://mp.weixin.qq.com/s?__biz=MzA5NDQzODQ3MQ==&mid=2648191577&idx=1&sn=41479618e4ff5eb918356a21235bd728&chksm=88620771bf158e673966b1a58a33a23a04b361577fd80a2c7dd6a5136eb3e0ac9ebd8a1d95b4&scene=21#wechat_redirect)
    

- Upstream
    

- [正确使用邮件列表参与开源社区的协作](http://mp.weixin.qq.com/s?__biz=MzA5NDQzODQ3MQ==&mid=2648191527&idx=1&sn=9f0b0d1ccf1f48e482d85847fc376b92&chksm=8862070fbf158e190d7527e6c57798d0ee7b4ce66a0ce5549ae10ee5eabd371b20b6c562f2d9&scene=21#wechat_redirect)  
    

### 实验演示

实验统一采用自研 Linux Lab 开源软件和泰晓 Linux 实验盘，某宝检索“泰晓 Linux”选购，详情：https://tinylab.org/linux-lab-disk

  

- [历时 13 年，只为那 1 分钟的极致开发体验](http://mp.weixin.qq.com/s?__biz=MzA5NDQzODQ3MQ==&mid=2648191639&idx=1&sn=58cdbd70de90c33f2ffdb8ede52d7cdb&chksm=886207bfbf158ea9666dd8adc67e81bc19cc4f9ce2686974c047019fedbd4f57f1d74a1b2b67&scene=21#wechat_redirect)  
    
- [Linux Lab v1.1 来了，3s 内开展 Linux 内核实验，新增 v6.0.7 支持](http://mp.weixin.qq.com/s?__biz=MzA5NDQzODQ3MQ==&mid=2648190868&idx=1&sn=f6f26f646153e41f5ad3f14cac4ef039&chksm=88621abcbf1593aa788711f77d8292159ab5bdafd3a92182dcbd18c729e2379d83bc42950bc7&scene=21#wechat_redirect)  
    
- [在 Linux Lab Disk 开展 5 大 CPU 汇编语言实验](http://mp.weixin.qq.com/s?__biz=MzA5NDQzODQ3MQ==&mid=2648188737&idx=1&sn=bfb4f9dc00945afb6578c2dc251c4822&chksm=88621269bf159b7f019f50f693e8395c25f6ca5cdd786a5a504d4ca4c70c4766b5e52c658488&scene=21#wechat_redirect)
    
- [Linux v5.17内核刚发布，做 RISC-V 内核实验吧](http://mp.weixin.qq.com/s?__biz=MzA5NDQzODQ3MQ==&mid=2648188766&idx=1&sn=d6474be60add2c90f894090fd06fe6b1&chksm=88621276bf159b6016656070129a7b9f12bcd1f54b11812a1aeffd0004df9b9a4e349690ff5c&scene=21#wechat_redirect)
    
- [坚守 6 年后，Linux Lab 终于迎来 v1.0 版](http://mp.weixin.qq.com/s?__biz=MzA5NDQzODQ3MQ==&mid=2648189113&idx=1&sn=62c0d6d1dac656d531126eb6ae009ed6&chksm=88621d91bf1594879fb6ee08240ef80741efdaaac2adabf18566edfc35bd7f2c79e83022fd95&scene=21#wechat_redirect)
    
- [Linux Lab 开源项目迎来 10 倍以上交互性能提升](http://mp.weixin.qq.com/s?__biz=MzA5NDQzODQ3MQ==&mid=2648188453&idx=1&sn=97a804887ed17c35a5b3d287d46e5abf&chksm=8862130dbf159a1bf42f9d77e1df6030289d160c536e022a64fbc33cb18da00f17c951298b17&scene=21#wechat_redirect)
    
- [Linux Lab 发布 v1.1-rc1，新增龙芯 v5.18 支持和 QEMU v7.0.0 编译支持](http://mp.weixin.qq.com/s?__biz=MzA5NDQzODQ3MQ==&mid=2648189420&idx=1&sn=eae6b96dec9a19ccaa23762710c71e62&chksm=88621cc4bf1595d23d7b3aaa1a6663f9e79430379c947c896ada6004ba478a01d697fca32dea&scene=21#wechat_redirect)
    
- [尝试摆脱 rm -rf / 的魔咒，泰晓 Linux 实验盘发布出厂恢复功能](http://mp.weixin.qq.com/s?__biz=MzA5NDQzODQ3MQ==&mid=2648189217&idx=1&sn=1e252342558c3ea50a60b2a55fc671d0&chksm=88621c09bf15951fa181aaee53f41d532f3a763981fa7523acd96e95a6f7e2de01edec3cb06e&scene=21#wechat_redirect)
    
- [](http://mp.weixin.qq.com/s?__biz=MzA5NDQzODQ3MQ==&mid=2648189217&idx=1&sn=1e252342558c3ea50a60b2a55fc671d0&chksm=88621c09bf15951fa181aaee53f41d532f3a763981fa7523acd96e95a6f7e2de01edec3cb06e&scene=21#wechat_redirect)[RISC-V 发展迅猛，正是关注好时机](http://mp.weixin.qq.com/s?__biz=MzA5NDQzODQ3MQ==&mid=2648181665&idx=1&sn=22ccac37ebf2432b09d2833628a7aa4f&chksm=88623e89bf15b79f8347d176a7a0c7d6eb513ff317a6b7168f66f3db81ff0ace39b5ae7a773d&scene=21#wechat_redirect)
    

### 活动情况

每月发布 1 期简报，持续分享 RISC-V Linux 内核调研、开发与移植最新进展。详情：https://gitee.com/tinylab/riscv-linux

  

- [RISC-V Linux 内核技术调研简报 第 9 期](http://mp.weixin.qq.com/s?__biz=MzA5NDQzODQ3MQ==&mid=2648191790&idx=2&sn=77ae35728203186622417f5b300f0991&chksm=88620606bf158f10aebf9f1bfa2965b0896bb614f8cb357bb94100f8f812ead4bf7f6e799453&scene=21#wechat_redirect)  
    
- [RISC-V Linux 内核技术调研简报 第 8 期](http://mp.weixin.qq.com/s?__biz=MzA5NDQzODQ3MQ==&mid=2648191670&idx=3&sn=e679b5bacc76f3403abc96521cdea933&chksm=8862079ebf158e88f5de6a5dffe8ef5a85085c1fb80b6c5585510958a021f0c0ecdb9a9c8029&scene=21#wechat_redirect)  
    
- [RISC-V Linux 内核技术调研简报 第 7 期](http://mp.weixin.qq.com/s?__biz=MzA5NDQzODQ3MQ==&mid=2648191512&idx=2&sn=280bad9aa0660e228607dbab00365d02&chksm=88620730bf158e269fc512298c5a3220fe990d42c66778d50c375d7cc320d99da54035cec65c&scene=21#wechat_redirect)  
    
- [RISC-V Linux 内核技术调研简报 第 6 期](http://mp.weixin.qq.com/s?__biz=MzA5NDQzODQ3MQ==&mid=2648191370&idx=1&sn=6ff67f25168f526a0b8b19ba53f62bf3&chksm=886204a2bf158db4c861547b04e957d3a19da30f87138cb1b10eff7aa4c69e2f44e8adb28bf1&scene=21#wechat_redirect)  
    
- [RISC-V Linux 内核技术调研简报 第 5 期](http://mp.weixin.qq.com/s?__biz=MzA5NDQzODQ3MQ==&mid=2648191176&idx=1&sn=d60c81729fb11c7d300b6181a28a6560&chksm=886205e0bf158cf64084a0995a4b865470e36fc84800c5695b7ce44f25e474c7b1c13542a0f1&scene=21#wechat_redirect)  
    
- [RISC-V Linux 内核技术调研简报 第 4 期](http://mp.weixin.qq.com/s?__biz=MzA5NDQzODQ3MQ==&mid=2648190868&idx=2&sn=13b08e0f0566f77554051a2826c20849&chksm=88621abcbf1593aa4caa8fd6cfbdce14e84c961dabb2d343757ed18c2d035a0c6d865d3a619e&scene=21#wechat_redirect)  
    
- [RISC-V Linux 内核剖析活动 PR&Review 流程](http://mp.weixin.qq.com/s?__biz=MzA5NDQzODQ3MQ==&mid=2648188808&idx=1&sn=1ac20c5c113f7b8f86b2e5407c89f2b5&chksm=886212a0bf159bb6b5bff9babeb86b8ddb721f93b32463707595ed401197ef27d1f5b57af048&scene=21#wechat_redirect)  
    
- [RISC-V Linux 内核剖析活动进入第 2 阶段并开放实习与兼职岗位](http://mp.weixin.qq.com/s?__biz=MzA5NDQzODQ3MQ==&mid=2648189105&idx=1&sn=19869ad5311e1e73a0258a8180c009c4&chksm=88621d99bf15948f5b61628796c4b052656fa5aae789651b82d05bc963471358c5a113bad2e7&scene=21#wechat_redirect)
    
- [RISC-V Linux 内核兴趣小组活动简报（2）](http://mp.weixin.qq.com/s?__biz=MzA5NDQzODQ3MQ==&mid=2648188965&idx=1&sn=c0c678af11ee9865f3dce6868ea74976&chksm=88621d0dbf15941b2bffa5511d090b6e3c48577a511014332182ed38d3731d6efbe7d2d86d05&scene=21#wechat_redirect)
    
- [RISC-V Linux 内核兴趣小组活动简报（1）](http://mp.weixin.qq.com/s?__biz=MzA5NDQzODQ3MQ==&mid=2648188793&idx=1&sn=ec30ea787ad65b8492b8b3943b5dadfa&chksm=88621251bf159b47f6cdffaccb110e825bdfb6f73b8a6d38d9deb876a3b89a803a9e1540e4ca&scene=21#wechat_redirect)
    
- [RISC-V Linux 内核兴趣小组召集爱好者-ing](http://mp.weixin.qq.com/s?__biz=MzA5NDQzODQ3MQ==&mid=2648188624&idx=1&sn=76c8067aa544cc14d97c60865ccb2382&chksm=886213f8bf159aeeefc92c02670e556a7404647697e48be49477f2d54a77ce2ae1b910ead596&scene=21#wechat_redirect)
    

### 人物访谈

每月发布 1 - 2 期人物或项目访谈，旨在介绍 RISC-V & Linux 内核及周边技术栈的项目动态、开发者经验及前沿技术热点。

  

- [访谈 | RISC-V 虚拟化技术调研与分析](http://mp.weixin.qq.com/s?__biz=MzA5NDQzODQ3MQ==&mid=2648191668&idx=1&sn=8c3571ffe71a82b040eb54e61bb7d9a6&chksm=8862079cbf158e8aa33e5da23ca4c67f64e3bf8c0de5dccf21abbb5c90768fb8164a5b50b260&scene=21#wechat_redirect)  
    
- [访谈 | 我的第一笔 Patch 之 OpenSBI 篇](http://mp.weixin.qq.com/s?__biz=MzA5NDQzODQ3MQ==&mid=2648191480&idx=1&sn=13e2cf3764d85ab582b8f0dfd92f40f3&chksm=886204d0bf158dc61fea7817be12b8db3e0c03808c1f7bf770d2b5a0d3b3d4a00843af7350bb&scene=21#wechat_redirect)  
    
- [访谈 | RISC-V 开发板 AI 应用开发实践](http://mp.weixin.qq.com/s?__biz=MzA5NDQzODQ3MQ==&mid=2648191316&idx=1&sn=76c70b0f10762094175b60e2fee5f347&chksm=8862047cbf158d6a2088d67a1b8ec9ecdfbd18e384c52bba762e9743bc0d66869f37c5189f20&scene=21#wechat_redirect)  
    
- [访谈 | 我的第一笔 Patch 之 DTS 篇](http://mp.weixin.qq.com/s?__biz=MzA5NDQzODQ3MQ==&mid=2648191226&idx=1&sn=407a1476a5df586b8872849be6057340&chksm=886205d2bf158cc4a4a6546cc2ecd6d8bb1f4a25b052ee0b79d64a31005444ca91d912f7128b&scene=21#wechat_redirect)  
    
- [访谈 | 我的第一笔 Patch 之内存管理篇](http://mp.weixin.qq.com/s?__biz=MzA5NDQzODQ3MQ==&mid=2648191008&idx=1&sn=d60cbf7f9330b01bf86dc32a54bff425&chksm=88620508bf158c1e343842df698afcc33845f7bbf1e8ff50af0d503fc0d0a387d33dd9b440b9&scene=21#wechat_redirect)
    
- [访谈 | 正是参与 RISC-V 虚拟化生态好时机](http://mp.weixin.qq.com/s?__biz=MzA5NDQzODQ3MQ==&mid=2648190739&idx=1&sn=6a9f7e18b7dc6e7ea7829f9881742c64&chksm=88621a3bbf15932de6a28e4e0d5fce3bc864ec4b867e3c3a2fa790a9184d73adf90029aea66e&scene=21#wechat_redirect)
    
- [访谈 | RISC-V 让全栈开源硬件产品成为可能](http://mp.weixin.qq.com/s?__biz=MzA5NDQzODQ3MQ==&mid=2648190739&idx=2&sn=0cb5c99bc17887347e7df110c0059622&chksm=88621a3bbf15932d993ef72473e0e112723234ec64d0eb5cab5cc8b457ff0ea0e6fee215b38c&scene=21#wechat_redirect)
    
- [访谈 | QEMU 在 RISC-V 生态中举足轻重](http://mp.weixin.qq.com/s?__biz=MzA5NDQzODQ3MQ==&mid=2648190454&idx=1&sn=07b65936f78d24e0238bd3c870b2d7cd&chksm=886218debf1591c838ce754023d1eed24402e580b43a43a9e341949d300ccdb3fb3a41f90eac&scene=21#wechat_redirect)
    
- [访谈 | 中文已成为最活跃的第二内核文档语言](http://mp.weixin.qq.com/s?__biz=MzA5NDQzODQ3MQ==&mid=2648190271&idx=1&sn=1209c535e37c0bc440b99073a97b1378&chksm=88621817bf159101c20dd636faee0e92da2d257ed31ec1956b0a0aa540dd56da69101c2c01d2&scene=21#wechat_redirect)
    
- [访谈｜CTF 选手正在开发 PWN 训练专用虚拟实验室](http://mp.weixin.qq.com/s?__biz=MzA5NDQzODQ3MQ==&mid=2648190161&idx=1&sn=bfc74351f40bf0a56ec190bbb70928df&chksm=886219f9bf1590ef475de3c5b58cd019151197311f7aa4db8350829bf0990c25cb213a90db96&scene=21#wechat_redirect)
    
- [访谈 | RustSBI 作者讲述 RISC-V 开源固件的精彩开发历程](http://mp.weixin.qq.com/s?__biz=MzA5NDQzODQ3MQ==&mid=2648190110&idx=1&sn=1aeb630b487a8fc94e83afffce97b46b&chksm=886219b6bf1590a05fa47629b87a2a2a95d89bbd1d6bec920da4ceb19df78dda5595d56451da&scene=21#wechat_redirect)
    
- [访谈 | 跟程序媛一块评测 LoongArch 和 RISC-V 处理器的性能](http://mp.weixin.qq.com/s?__biz=MzA5NDQzODQ3MQ==&mid=2648189946&idx=1&sn=61ffac5bb93b68fb8dc4412b9bcaed38&chksm=88621ed2bf1597c4f2b919c5baf5396997c08dc060459c082f5ad397baa203073f879b941621&scene=21#wechat_redirect)
    
- [访谈 | Maintainer 揭秘龙芯 LoongArch 架构进入 v5.19 的背后故事](http://mp.weixin.qq.com/s?__biz=MzA5NDQzODQ3MQ==&mid=2648189805&idx=1&sn=61694f18e9ff623acac4a8b2a1d1f894&chksm=88621e45bf159753f611b1442a18c5bafbb6c98131a186c4ac8c6a0ddbc192322f65191550e1&scene=21#wechat_redirect)  
    

### 技术动态

每周日及时跟进 RISC-V & Linux 内核及周边技术栈上游官方社区的最新开发进展。从 2022.11 起，新增近 10 项内核公共特性的追踪。也可以直接看合集：https://gitee.com/tinylab/riscv-linux/tree/master/news

  

**说明**：由于公众号的推送限制，RISC-V Linux 内核技术动态栏目从第 41 期开始直接发表到社区网站上（https://tinylab.org），不再发布到公众号，请知悉。

  

- RISC-V Linux 内核及周边技术动态 第 40 期/推送失败
    
- [RISC-V Linux 内核及周边技术动态 第 39 期](http://mp.weixin.qq.com/s?__biz=MzA5NDQzODQ3MQ==&mid=2648191594&idx=1&sn=95e5dbb2f1eca99e822c4d82eaf712a9&chksm=88620742bf158e5496aaca343447961ee3fdd488f49ec23e0b25b2b1cd5f234904def0f303d3&scene=21#wechat_redirect)
    
- [RISC-V Linux 内核及周边技术动态 第 38 期](http://mp.weixin.qq.com/s?__biz=MzA5NDQzODQ3MQ==&mid=2648191557&idx=1&sn=2e346cb0f0bca10178ba37838a890689&chksm=8862076dbf158e7bd56477eababad43b9e4a6f864c0e211f00b850fb46a237acc5e5dfb69fbd&scene=21#wechat_redirect)
    
- [RISC-V Linux 内核及周边技术动态 第 37 期](http://mp.weixin.qq.com/s?__biz=MzA5NDQzODQ3MQ==&mid=2648191542&idx=1&sn=be8c59c9e768772421dbdeec7bcf8a8d&chksm=8862071ebf158e087ff87da802dc59e72fcb3bb2f2c2b929fcb2235254b3c684b1c1a4e09a5e&scene=21#wechat_redirect)
    
- [RISC-V Linux 内核及周边技术动态 第 36 期](http://mp.weixin.qq.com/s?__biz=MzA5NDQzODQ3MQ==&mid=2648191512&idx=1&sn=5f84cbffc17608ffe616c3c2745f7196&chksm=88620730bf158e26c8cb06ef727d206c3ac8ea3742193dd3aad355f574c132f3990bb573383e&scene=21#wechat_redirect)  
    
- [RISC-V Linux 内核及周边技术动态 第 35 期](http://mp.weixin.qq.com/s?__biz=MzA5NDQzODQ3MQ==&mid=2648191460&idx=1&sn=17519d29605146663c861c7a02f44b9c&chksm=886204ccbf158ddaf49b554806c71130e09d5079b453324c58d0925cfa71bec9139bbcb71484&scene=21#wechat_redirect)
    
- [RISC-V Linux 内核及周边技术动态 第 34 期](http://mp.weixin.qq.com/s?__biz=MzA5NDQzODQ3MQ==&mid=2648191429&idx=1&sn=379428b27fa335520ef92144a217f59a&chksm=886204edbf158dfba86a7d5b9aae5699e3e8fa1f6f0c2ac11c371a85b5c0cd53e2f05c98f6c7&scene=21#wechat_redirect)
    
- [RISC-V Linux 内核及周边技术动态 第 33 期](http://mp.weixin.qq.com/s?__biz=MzA5NDQzODQ3MQ==&mid=2648191406&idx=1&sn=9c27217572ea1b504d4395456426416d&chksm=88620486bf158d9063229c4e5412381490f75d75ff1abb761d6ab37704b11c9c46f8dd8b6277&scene=21#wechat_redirect)
    
- [RISC-V Linux 内核及周边技术动态 第 32 期](http://mp.weixin.qq.com/s?__biz=MzA5NDQzODQ3MQ==&mid=2648191385&idx=1&sn=18c204e6019bf4e726159cad1cac6422&chksm=886204b1bf158da7cb2fb1369be8a8c6f89d26aa9a350e1342b00bcf80369243c6261a70adf4&scene=21#wechat_redirect)  
    
- [RISC-V Linux 内核及周边技术动态 第 31 期](http://mp.weixin.qq.com/s?__biz=MzA5NDQzODQ3MQ==&mid=2648191336&idx=1&sn=0093fb9d79743012c839a474f274e8c9&chksm=88620440bf158d56c1a845d2d8f6fc270bd807f89ce81ee0ff4de0d33d34cb0a8efe13cca613&scene=21#wechat_redirect)  
    
- [RISC-V Linux 内核及周边技术动态 第 30 期](http://mp.weixin.qq.com/s?__biz=MzA5NDQzODQ3MQ==&mid=2648191321&idx=1&sn=df3bfed5d05702cbc355d04e3a6e9575&chksm=88620471bf158d67ba4fe7cbd7be753f7fc3493aea89ee37832a8e1a1b35b10230dfe6f14666&scene=21#wechat_redirect)
    
- [RISC-V Linux 内核及周边技术动态 第 29 期](http://mp.weixin.qq.com/s?__biz=MzA5NDQzODQ3MQ==&mid=2648191294&idx=1&sn=0403f0ea0ef481ca9adfc1b17bfcc663&chksm=88620416bf158d004742e5e920da4e9d740b4474b1f534c6803f0ef510814362907a51afe9da&scene=21#wechat_redirect)
    
- [RISC-V Linux 内核及周边技术动态 第 28 期](http://mp.weixin.qq.com/s?__biz=MzA5NDQzODQ3MQ==&mid=2648191277&idx=1&sn=f3af4a53afb4525047642a338e7acfc4&chksm=88620405bf158d13499f878c6a076fefb0b6b60bf015aa0fc940381fadc76a93c5637cfe8ac6&scene=21#wechat_redirect)
    
- [RISC-V Linux 内核及周边技术动态 第 27 期](http://mp.weixin.qq.com/s?__biz=MzA5NDQzODQ3MQ==&mid=2648191231&idx=1&sn=a80dcb546032a43dc7ced9663bdf74ce&chksm=886205d7bf158cc1b15da91f690d8cf74c29a96c89a40deb64a4ba48fb85caee54cb39286544&scene=21#wechat_redirect)
    
- [RISC-V Linux 内核及周边技术动态 第 26 期](http://mp.weixin.qq.com/s?__biz=MzA5NDQzODQ3MQ==&mid=2648191206&idx=1&sn=72c784e06521bc0e91c7b07d1fd76a9d&chksm=886205cebf158cd85feb95d307b106d0a5fdda0e45bfe3a35a4383b366acca65e2d275629645&scene=21#wechat_redirect)
    
- [RISC-V Linux 内核及周边技术动态 第 25 期](http://mp.weixin.qq.com/s?__biz=MzA5NDQzODQ3MQ==&mid=2648191151&idx=1&sn=dfa359cda17cf81dd4829f226a2f3d69&chksm=88620587bf158c918dc6bda574c92631d87d7df59614082b562b3dce11a2e327667c71475239&scene=21#wechat_redirect)
    
- [RISC-V Linux 内核及周边技术动态 第 24 期](http://mp.weixin.qq.com/s?__biz=MzA5NDQzODQ3MQ==&mid=2648191143&idx=1&sn=a2f45e1469ce7a47ab1cc776486968d9&chksm=8862058fbf158c99d0ed931c4212e52c9b7d4a3ca33e76003839e5172d4a28870bfaa00244ae&scene=21#wechat_redirect)  
    
- [RISC-V Linux 内核及周边技术动态 第 23 期](http://mp.weixin.qq.com/s?__biz=MzA5NDQzODQ3MQ==&mid=2648191107&idx=1&sn=2dd7205f9e0e9fcd8e15fe85b8c19046&chksm=886205abbf158cbda68c98cf3b77bfd549b86c9f65db3adb6e04ec23da8a2c07c309b8164225&scene=21#wechat_redirect)
    
- [RISV-V Linux 内核及周边技术动态 第 22 期](http://mp.weixin.qq.com/s?__biz=MzA5NDQzODQ3MQ==&mid=2648191023&idx=1&sn=b131f55a2dc6b15a08cb08bcbb54b71b&chksm=88620507bf158c11c87cbfa3ec96837a197dfe0727e9591bde930cf27f474f1035f51a5a4bac&scene=21#wechat_redirect)
    
- [RISC-V Linux 内核及周边技术动态 第 21 期](http://mp.weixin.qq.com/s?__biz=MzA5NDQzODQ3MQ==&mid=2648190930&idx=1&sn=4fdfba6feca602e02f385cc32692502b&chksm=88621afabf1593ec4f41adbdfe7c15754ce63b96e955ffa7a834500d334540bfe7a89130d908&scene=21#wechat_redirect)
    
- [RISC-V Linux 内核及周边技术动态 第 20 期](http://mp.weixin.qq.com/s?__biz=MzA5NDQzODQ3MQ==&mid=2648190884&idx=1&sn=f36c25173736e7f42d3835b0d1876af5&chksm=88621a8cbf15939aa0b00cf98fa56fa879bc9e0d9771df3e84764b7a3c60b73920a5415a9d1d&scene=21#wechat_redirect)  
    
- [RISC-V Linux 内核及周边技术动态 第 19 期](http://mp.weixin.qq.com/s?__biz=MzA5NDQzODQ3MQ==&mid=2648190744&idx=1&sn=ea440b2dfa139d2da56a8fd6e1f7c572&chksm=88621a30bf159326a139a69af6d3381fb7931256bf641c6adc3094f9aff6de9a819e2f88d93b&scene=21#wechat_redirect)  
    
- [RISC-V Linux 内核及周边技术动态 第 18 期](http://mp.weixin.qq.com/s?__biz=MzA5NDQzODQ3MQ==&mid=2648190634&idx=1&sn=89fddd66ac0d07525cf0955e19cea6c4&chksm=88621b82bf1592946ce2a283ade85383ddffba396e475e793fc3390aa76497141c448dddc026&scene=21#wechat_redirect)
    
- [RISC-V Linux 内核及周边技术动态 第 17 期](http://mp.weixin.qq.com/s?__biz=MzA5NDQzODQ3MQ==&mid=2648190535&idx=1&sn=a51bfb92dbd72b10912cffede5814c7c&chksm=88621b6fbf159279b2870af85c72acbc66efc3eda596ddafb9b8b80ab0a7a8c99a2b24e0acd5&scene=21#wechat_redirect)
    
- [RISV-V Linux 内核及周边技术动态 第 16 期](http://mp.weixin.qq.com/s?__biz=MzA5NDQzODQ3MQ==&mid=2648190501&idx=1&sn=b89dd3a6fca096cd9c1199ee456b4e48&chksm=88621b0dbf15921bb65fea35d7639d114eee632772e634814e538e22f46bdc7cacc2c3282997&scene=21#wechat_redirect)
    
- [RISC-V Linux 内核及周边技术动态 第 15 期](http://mp.weixin.qq.com/s?__biz=MzA5NDQzODQ3MQ==&mid=2648190473&idx=1&sn=227dde066b94fca4bc2edb5c05d609af&chksm=88621b21bf159237a750c8c18069734057a8e2fa21a40c419972d0f674a88b79bad842ef7b54&scene=21#wechat_redirect)
    
- [RISC-V Linux 内核及周边技术动态 第 14 期](http://mp.weixin.qq.com/s?__biz=MzA5NDQzODQ3MQ==&mid=2648190462&idx=1&sn=0b08bab7074c3dda1d1fd52a33ccf5ee&chksm=886218d6bf1591c0056f51566e5c5204d2be325888ef1a52b00e29db0b9977cd0139501c2134&scene=21#wechat_redirect)
    
- [RISC-V Linux 内核及周边技术动态 第 13 期](http://mp.weixin.qq.com/s?__biz=MzA5NDQzODQ3MQ==&mid=2648190415&idx=1&sn=c29a8eeff3ae6e4a9a33a3904a6ef54d&chksm=886218e7bf1591f1edcdb80ec2aa77a3ab579cd04f6ba79b18e9b3fb233ab3bf05ec1666e1e7&scene=21#wechat_redirect)
    
- [RISC-V Linux 内核及周边技术动态 第 12 期](http://mp.weixin.qq.com/s?__biz=MzA5NDQzODQ3MQ==&mid=2648190278&idx=1&sn=cd85fa47c03698cd17de33932384ed15&chksm=8862186ebf1591786b5f03b4a3686f06bc813c2c43ace382772712464e00ca66cd474cf87337&scene=21#wechat_redirect)
    
- [RISC-V Linux 内核及周边技术动态 第 11 期](http://mp.weixin.qq.com/s?__biz=MzA5NDQzODQ3MQ==&mid=2648190166&idx=1&sn=2824d3e5844da37117d3adcf87e39959&chksm=886219febf1590e8a444fed28f58c5e3cabc885d3a31d0907c8e761e038252ce2cbf1aa90abc&scene=21#wechat_redirect)
    
- [RISC-V Linux 内核及周边技术动态 第 10 期](http://mp.weixin.qq.com/s?__biz=MzA5NDQzODQ3MQ==&mid=2648190122&idx=1&sn=8ff348bb1cecd123b527d3e4a839bb21&chksm=88621982bf1590946b45c6f21901ae98f4b3adbb2c3a8a3ec66c51489b0e31c9fc39a38c926b&scene=21#wechat_redirect)
    
- [RISC-V Linux 内核及周边技术动态 第 9 期](http://mp.weixin.qq.com/s?__biz=MzA5NDQzODQ3MQ==&mid=2648189961&idx=1&sn=ea3cc488b3196506c7759eee4b2aac43&chksm=88621921bf15903751d109ca1f00bc2769f5f6f99d51e9fad227bcb268a25dc206782382c8a5&scene=21#wechat_redirect)
    
- [RISC-V Linux 内核及周边技术动态 第 8 期](http://mp.weixin.qq.com/s?__biz=MzA5NDQzODQ3MQ==&mid=2648189809&idx=1&sn=e482142e0d87a406783e7dfdc0356329&chksm=88621e59bf15974f08b77b53b17a2567743d6211a29d3ed19faaeac6bc00a2d717e1827a0033&scene=21#wechat_redirect)  
    
- [RISC-V Linux 内核及周边技术动态 第 7 期](http://mp.weixin.qq.com/s?__biz=MzA5NDQzODQ3MQ==&mid=2648189700&idx=1&sn=5473b1f653c2b57635e91b5be46a1c9e&chksm=88621e2cbf15973aceb97be914c084898453a62d7ab974d736b596109d9b19f652837be934a1&scene=21#wechat_redirect)
    
- [RISC-V Linux 内核及周边技术动态 第 6 期](http://mp.weixin.qq.com/s?__biz=MzA5NDQzODQ3MQ==&mid=2648189615&idx=1&sn=e57d217d93ed7fb2af9e0e71dce45b5e&chksm=88621f87bf1596913b4385e9a474e21a1263a0dba40c2d39b534e30354ea4d13078be617f9df&scene=21#wechat_redirect)
    
- [RISC-V Linux 内核及周边技术动态 第 5 期](http://mp.weixin.qq.com/s?__biz=MzA5NDQzODQ3MQ==&mid=2648189496&idx=1&sn=68f9fea0977b45f6ae248b233d8cb666&chksm=88621f10bf15960636abe1c20690f71bd1c14117a510f8731434a5b90089d7ae44e78f2ddc52&scene=21#wechat_redirect)
    
- [RISC-V Linux 内核及周边技术动态 第 4 期](http://mp.weixin.qq.com/s?__biz=MzA5NDQzODQ3MQ==&mid=2648189413&idx=1&sn=8f4f0529ab6f8d439de657d86c5265b4&chksm=88621ccdbf1595db48bfa647d3a1df71e2ca74711fd3aa655949b750dd30c8ebdb74c965e294&scene=21#wechat_redirect)
    
- [RISC-V Linux 内核及周边技术动态 第 3 期](http://mp.weixin.qq.com/s?__biz=MzA5NDQzODQ3MQ==&mid=2648189363&idx=1&sn=ecd59336ad625f036c92549dde1d5947&chksm=88621c9bbf15958d9b52e8de68c099b302d11efed03d1722a83c5a9d350451000023e22cdf38&scene=21#wechat_redirect)
    
- [RISC-V Linux 内核及周边技术动态 第 2 期](http://mp.weixin.qq.com/s?__biz=MzA5NDQzODQ3MQ==&mid=2648189333&idx=1&sn=841d2494b12232ef70b210e4e12ab983&chksm=88621cbdbf1595ab686f23aad1562a63a4ef02184ec73cf030c0e958d45793fb528e282519f8&scene=21#wechat_redirect)
    
- [RISC-V Linux 内核及周边技术动态 第 1 期](http://mp.weixin.qq.com/s?__biz=MzA5NDQzODQ3MQ==&mid=2648189244&idx=1&sn=1d35dae0f0a53be46d050589b732ece4&chksm=88621c14bf159502d28f9811de15aad6661619b09dd6728b91029c6962add1eab59e7fae114b&scene=21#wechat_redirect)  
    

## 致谢

感谢中科院软件所 PLCT 实验室对该活动的大力支持。

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

linux内核164

linux内核 · 目录

上一篇RISC-V Linux 内核技术调研简报 第 10 期下一篇RISC-V 异常处理在 KVM 中的实现

Read more

Reads 182

​

Comment

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/XXJQJDtx0ea3GDXLMgprFptA1sKUYEYt3NNtA8bBuREmchgSXogPkvzqHvAANlQ8a6Tx0yeuzh1K5ibuqtWyYTA/300?wx_fmt=png&wxfrom=18)

泰晓科技

21Wow

Comment

Comment

**Comment**

暂无留言