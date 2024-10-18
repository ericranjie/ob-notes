
安谋科技学堂 _2023年10月24日 20:00_

以下文章来源于Arm精选 ，作者baron

本文授权转自微信公众号Arm精选，本篇主要分享一个使用perf分析cache的示例。

# **一** **编写测试程序**

首先，需要编写一个测试程序，该程序会访问一些内存地址，并且这些内存地址会被缓存在Cache中。这个测试程序可以使用C语言编写，示例代码如下：

```c
#include <stdio.h> 
#include <stdlib.h> 
#include <time.h> 
#define ARRAY_SIZE 4096
#define MAX_LOOP 1000000
int main() { int i, j, sum = 0; int array[ARRAY_SIZE]; srand(time(NULL)); for (i = 0; i < ARRAY_SIZE; i++) { array[i] = rand() % 100; } for (i = 0; i < MAX_LOOP; i++) { for (j = 0; j < ARRAY_SIZE; j++) { sum += array[j]; } } printf("sum = %d\n", sum); return 0; }
```

这个测试程序会生成一个随机的整数数组，然后在一个循环中反复访问这个数组，并计算数组中所有元素的和。

# **二** **使用perf监控Cache事件**

接下来，需要使用perf监控Cache事件，以了解Cache的行为。可以使用如下命令监控L1 Dcache的miss事件：

```c
perf stat -e L1-dcache-load-misses ./test
```

这个命令会运行之前编写的测试程序，并监控程序执行期间L1 Dcache的miss事件的数量。执行结果如下：

```c
Performance counter stats for './test': 415,348      L1-dcache-load-misses 1.027751484 seconds time elapsed
```

结果显示，在测试程序执行期间，L1 Dcache一共发生了415348次miss事件。

# **三** **使用perf报告工具分析结果**

最后，需要使用perf报告工具分析结果，以更直观地了解Cache的性能和行为。可以使用如下命令生成perf报告：perf report。

这个命令会打开一个交互式的报告界面，在界面中可以看到各种性能指标的统计信息，例如Cache的命中率、失效率、访问延迟等。可以使用箭头键在报告中导航，查看不同的指标和代码行数。例如，在这个测试程序中，可以看到Cache失效率比较高，其中一个主要的原因是数组大小过大，导致Cache的容量不足。

通过上述步骤，可以使用perf分析Cache的行为，并深入了解Cache的性能和行为，从而优化程序的Cache访问策略。

______________________________________________________________________

**推荐阅读**

- [刷cache是啥意思？本质是什么？](http://mp.weixin.qq.com/s?__biz=MzU5NzczNTU3Nw==&mid=2247498296&idx=2&sn=382d4a0de0388fc6ec55101e6f261891&chksm=fe4c5bcec93bd2d8863fdfc2172b1a285015bb148ae8c81136738f7bc185613d21cc2d707129&scene=21#wechat_redirect)

- [一篇论文讲透Cache优化](http://mp.weixin.qq.com/s?__biz=MzU5NzczNTU3Nw==&mid=2247497577&idx=1&sn=aaef267b2e34b576e475558f811677ec&chksm=fe4c569fc93bdf898ee3fab3bcf259bcd7da7a1d5e66d927bfa3f9485cf3b5388a5325933082&scene=21#wechat_redirect)

- [简述cache的基本概念和使用场景](http://mp.weixin.qq.com/s?__biz=MzU5NzczNTU3Nw==&mid=2247497111&idx=1&sn=406ec23ec288830b941bc425bec2ce09&chksm=fe4c5461c93bdd77cbccb1c8fdd40b9a91e23f3d740b6251721f92df89caa7fe754471daa614&scene=21#wechat_redirect)

![图片](https://mmbiz.qpic.cn/mmbiz_gif/D4YpGeYE2TqBUpYwiaXMxjVKpAv5zwoVMPctEXA4yaMDrRHSTCtYIC0b4yeLy2EpFNzYLOVfCwIIoattf9h2BJA/640?wx_fmt=gif&tp=webp&wxfrom=5&wx_lazy=1)

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/D4YpGeYE2TqBUpYwiaXMxjVKpAv5zwoVMrLkZaRbIpu15ThQx7TwdlzWKTpNEdSjB1jiaBSRr8ygkXoOiaLkBzicpg/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

![图片](https://mmbiz.qpic.cn/mmbiz_gif/D4YpGeYE2TqBUpYwiaXMxjVKpAv5zwoVMHu7HBeGGzDzKh2NpDSRkIPy5Ihen9dY82tiahialibgBaia7Hiczrpalo3w/640?wx_fmt=gif&tp=webp&wxfrom=5&wx_lazy=1)

**长按识别二维码添加极术小姐姐微信（aijishu20），加入安谋科技学堂读者群。**

[****3100+课程使用的11种Arm教育套件点击免费下载啦，含嵌入式，芯片设计，信号处理，操作系统等。****](http://mp.weixin.qq.com/s?__biz=MzU5NzczNTU3Nw==&mid=2247498376&idx=1&sn=de95d75ec160c8214e4b6d50b671346f&chksm=fe4c5b7ec93bd26834c7c1e9e16a1f20977da8a5bb2cc2c5763cf036ec67a813e2a1c94ae4e9&scene=21#wechat_redirect)

关注安谋科技学堂

![](http://mmbiz.qpic.cn/mmbiz_png/D4YpGeYE2Too2yEibynvrnvvPntlNYyrPphf69g4j5ApibQeby7fdLZDviagIcqvKCLiael4eBUVBPTQ2icnWzZeVibg/300?wx_fmt=png&wxfrom=19)

**安谋科技学堂**

安谋科技教育计划将高等教育机构与丰富的Arm产品联系起来，为教育者、学生和研究人员提供教学资料、硬件平台、软件开发工具、IP和资源，支持将Arm技术用作教育用途。

13篇原创内容

公众号

![图片](https://mmbiz.qpic.cn/mmbiz_gif/D4YpGeYE2TqBUpYwiaXMxjVKpAv5zwoVM6k8V1xKGGibsTkhffNAdibAPmYLejhQtlz90PKggZqDOjiaONqeYzLj5g/640?wx_fmt=gif&tp=webp&wxfrom=5&wx_lazy=1) 点击下方“**阅读原文**”，立即下载Arm教育套件学习充电。

[阅读原文](https://aijishu.com/download)

​

![](https://mp.weixin.qq.com/rr?timestamp=1725920740&src=11&ver=1&signature=RhKCp1b7jT8btozXVmwLNc9blGDRcw*s0fv3w*7sWH*bdOEdKujVCFjkvDPfx7wd3bKTu5siTNFXTzc1cCUrOe2NorbRCzeUz6S*d5zgegI=)

微信扫一扫\
关注该公众号
