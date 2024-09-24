# 

邀你一起成为开源贡献者 Linux

 _2021年10月25日 07:30_

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/W9DqKgFsc6ibLjdvIVnMdJheN6ibrszhWemvVgqYSGOjUKXFY50oHpsLdeiavLXZlhHrc5jRqojBuljy8IfjpxSng/640?wx_fmt=jpeg&wxfrom=13&tp=wxpic)

**导读：**使用此 Linux 命令保持日志文件更新。　 　 　 　 　 　 　 　 　 　 　 　 　 　 　 　 　 　 　 　

本文字数：6614，阅读时长大约：7分钟

  

https://linux.cn/article-13909-1.html  
作者：Ayush Sharma  
译者：perfiffer  

日志非常适合找出应用程序在做什么或对可能的问题进行故障排除。几乎我们处理的每个应用程序都会生成日志，我们希望我们自己开发的应用程序也生成日志。日志越详细，我们拥有的信息就越多。但放任不管，日志可能会增长到无法管理的大小，反过来，它们可能会成为它们自己的问题。因此，最好将它们进行裁剪，保留我们需要的那些，并将其余的归档。

![图片](https://mmbiz.qpic.cn/mmbiz_svg/B2EfAOZfS1hMttabQMHUHW5aM6gZHoRYcdcCjlX3sdibTF0XcFadA2VPOe1D8UWbS5RpzlvnOZUvzacjag2UJD0RTNHib9ahian/640?wx_fmt=svg&wxfrom=13)

基本功能

[logrotate](https://mp.weixin.qq.com/s?__biz=MzI1NDQwNDYyMg==&mid=2247489789&idx=1&sn=7500321728af0d8c5f5151c3845cdfe8&chksm=e9c4e99cdeb3608a6d3b749bb3bc66c737ec8eaecc5603524089dd9e94ccdc6faf3d163591ee&mpshare=1&scene=24&srcid=1025kFWafMvcAvoEiACZhRIZ&sharer_sharetime=1635119945491&sharer_shareid=5fb9813bfe9ffc983435bfc8d8c5e9ca&key=daf9bdc5abc4e8d0668e174ed925bd13abe2cea1549df1f7cac581c5f507d808682c6f629fbba4c51e87694f93df32c968fe1e2e61d91b72910ba96dfb132423bf788597b89c84e3eac1864e93887ccde5ba38a58e313db180ddf8755e6348f6035349c510be67dda07301264e3a4fdd308f8259fed3190726737abca97a422a&ascene=0&uin=MTEwNTU1MjgwMw%3D%3D&devicetype=Windows+11+x64&version=63090b19&lang=zh_CN&countrycode=CN&exportkey=n_ChQIAhIQSytgNVaWUYq3EC%2F6TddbUxLmAQIE97dBBAEAAAAAAPPWNWuHIBIAAAAOpnltbLcz9gKNyK89dVj0V%2BacR5uyLA%2FbYIt3pc6ww8fw5Zalb41o5Pe7ngEoTZbuDaSnibcMGwVSrA6DWoQq5XRqKUwIKcvhKVlq7efEbIK%2BHlY2dx4ctjcsEGX1M0U6VgzxfyknRTxKNB4H8%2BFbtq265tU5AmDzy%2FsHLTuHcy4saZ1af0CoAL2Wtc0vswjonkFmA7z8pAQaDryUyIWQBRhkoyhvF5yp36GxpyJeJrA0EbwqJ%2FlrhLGs2n626aQlidy08JLu3%2FvAwiYRsiAd&acctmode=0&pass_ticket=1zGvpvMXxh4d8ZJ3X5Oqa%2FzKt7RL1IFJFAl3BFEERDVkEv2a3H8saYAEMsyXHJFo&wx_header=1&fasttmpl_type=0&fasttmpl_fullversion=7351805-zh_CN-zip&fasttmpl_flag=1) 实用程序在管理日志方面非常出色。它可以轮转日志、压缩日志、通过电子邮件发送日志、删除日志、归档日志，并在你需要时开始记录最新的。

运行 [logrotate](https://mp.weixin.qq.com/s?__biz=MzI1NDQwNDYyMg==&mid=2247489789&idx=1&sn=7500321728af0d8c5f5151c3845cdfe8&chksm=e9c4e99cdeb3608a6d3b749bb3bc66c737ec8eaecc5603524089dd9e94ccdc6faf3d163591ee&mpshare=1&scene=24&srcid=1025kFWafMvcAvoEiACZhRIZ&sharer_sharetime=1635119945491&sharer_shareid=5fb9813bfe9ffc983435bfc8d8c5e9ca&key=daf9bdc5abc4e8d0668e174ed925bd13abe2cea1549df1f7cac581c5f507d808682c6f629fbba4c51e87694f93df32c968fe1e2e61d91b72910ba96dfb132423bf788597b89c84e3eac1864e93887ccde5ba38a58e313db180ddf8755e6348f6035349c510be67dda07301264e3a4fdd308f8259fed3190726737abca97a422a&ascene=0&uin=MTEwNTU1MjgwMw%3D%3D&devicetype=Windows+11+x64&version=63090b19&lang=zh_CN&countrycode=CN&exportkey=n_ChQIAhIQSytgNVaWUYq3EC%2F6TddbUxLmAQIE97dBBAEAAAAAAPPWNWuHIBIAAAAOpnltbLcz9gKNyK89dVj0V%2BacR5uyLA%2FbYIt3pc6ww8fw5Zalb41o5Pe7ngEoTZbuDaSnibcMGwVSrA6DWoQq5XRqKUwIKcvhKVlq7efEbIK%2BHlY2dx4ctjcsEGX1M0U6VgzxfyknRTxKNB4H8%2BFbtq265tU5AmDzy%2FsHLTuHcy4saZ1af0CoAL2Wtc0vswjonkFmA7z8pAQaDryUyIWQBRhkoyhvF5yp36GxpyJeJrA0EbwqJ%2FlrhLGs2n626aQlidy08JLu3%2FvAwiYRsiAd&acctmode=0&pass_ticket=1zGvpvMXxh4d8ZJ3X5Oqa%2FzKt7RL1IFJFAl3BFEERDVkEv2a3H8saYAEMsyXHJFo&wx_header=1&fasttmpl_type=0&fasttmpl_fullversion=7351805-zh_CN-zip&fasttmpl_flag=1) 非常简单——只需要运行 [logrotate -vs state-file config-file](https://mp.weixin.qq.com/s?__biz=MzI1NDQwNDYyMg==&mid=2247489789&idx=1&sn=7500321728af0d8c5f5151c3845cdfe8&chksm=e9c4e99cdeb3608a6d3b749bb3bc66c737ec8eaecc5603524089dd9e94ccdc6faf3d163591ee&mpshare=1&scene=24&srcid=1025kFWafMvcAvoEiACZhRIZ&sharer_sharetime=1635119945491&sharer_shareid=5fb9813bfe9ffc983435bfc8d8c5e9ca&key=daf9bdc5abc4e8d0668e174ed925bd13abe2cea1549df1f7cac581c5f507d808682c6f629fbba4c51e87694f93df32c968fe1e2e61d91b72910ba96dfb132423bf788597b89c84e3eac1864e93887ccde5ba38a58e313db180ddf8755e6348f6035349c510be67dda07301264e3a4fdd308f8259fed3190726737abca97a422a&ascene=0&uin=MTEwNTU1MjgwMw%3D%3D&devicetype=Windows+11+x64&version=63090b19&lang=zh_CN&countrycode=CN&exportkey=n_ChQIAhIQSytgNVaWUYq3EC%2F6TddbUxLmAQIE97dBBAEAAAAAAPPWNWuHIBIAAAAOpnltbLcz9gKNyK89dVj0V%2BacR5uyLA%2FbYIt3pc6ww8fw5Zalb41o5Pe7ngEoTZbuDaSnibcMGwVSrA6DWoQq5XRqKUwIKcvhKVlq7efEbIK%2BHlY2dx4ctjcsEGX1M0U6VgzxfyknRTxKNB4H8%2BFbtq265tU5AmDzy%2FsHLTuHcy4saZ1af0CoAL2Wtc0vswjonkFmA7z8pAQaDryUyIWQBRhkoyhvF5yp36GxpyJeJrA0EbwqJ%2FlrhLGs2n626aQlidy08JLu3%2FvAwiYRsiAd&acctmode=0&pass_ticket=1zGvpvMXxh4d8ZJ3X5Oqa%2FzKt7RL1IFJFAl3BFEERDVkEv2a3H8saYAEMsyXHJFo&wx_header=1&fasttmpl_type=0&fasttmpl_fullversion=7351805-zh_CN-zip&fasttmpl_flag=1)。在上面的命令中，`v` 选项开启详细模式，`s` 指定一个状态文件，最后的 `config-file` 是配置文件，你可以指定需要做什么。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

实战演练

让我们看看在我们的系统上静默运行的 [logrotate](https://mp.weixin.qq.com/s?__biz=MzI1NDQwNDYyMg==&mid=2247489789&idx=1&sn=7500321728af0d8c5f5151c3845cdfe8&chksm=e9c4e99cdeb3608a6d3b749bb3bc66c737ec8eaecc5603524089dd9e94ccdc6faf3d163591ee&mpshare=1&scene=24&srcid=1025kFWafMvcAvoEiACZhRIZ&sharer_sharetime=1635119945491&sharer_shareid=5fb9813bfe9ffc983435bfc8d8c5e9ca&key=daf9bdc5abc4e8d0668e174ed925bd13abe2cea1549df1f7cac581c5f507d808682c6f629fbba4c51e87694f93df32c968fe1e2e61d91b72910ba96dfb132423bf788597b89c84e3eac1864e93887ccde5ba38a58e313db180ddf8755e6348f6035349c510be67dda07301264e3a4fdd308f8259fed3190726737abca97a422a&ascene=0&uin=MTEwNTU1MjgwMw%3D%3D&devicetype=Windows+11+x64&version=63090b19&lang=zh_CN&countrycode=CN&exportkey=n_ChQIAhIQSytgNVaWUYq3EC%2F6TddbUxLmAQIE97dBBAEAAAAAAPPWNWuHIBIAAAAOpnltbLcz9gKNyK89dVj0V%2BacR5uyLA%2FbYIt3pc6ww8fw5Zalb41o5Pe7ngEoTZbuDaSnibcMGwVSrA6DWoQq5XRqKUwIKcvhKVlq7efEbIK%2BHlY2dx4ctjcsEGX1M0U6VgzxfyknRTxKNB4H8%2BFbtq265tU5AmDzy%2FsHLTuHcy4saZ1af0CoAL2Wtc0vswjonkFmA7z8pAQaDryUyIWQBRhkoyhvF5yp36GxpyJeJrA0EbwqJ%2FlrhLGs2n626aQlidy08JLu3%2FvAwiYRsiAd&acctmode=0&pass_ticket=1zGvpvMXxh4d8ZJ3X5Oqa%2FzKt7RL1IFJFAl3BFEERDVkEv2a3H8saYAEMsyXHJFo&wx_header=1&fasttmpl_type=0&fasttmpl_fullversion=7351805-zh_CN-zip&fasttmpl_flag=1) 配置，它管理我们在 `/var/log` 目录中找到的大量日志。查看该目录中的当前文件。你是否看到很多 `*.[number].gz` 文件？这就是 [logrotate](https://mp.weixin.qq.com/s?__biz=MzI1NDQwNDYyMg==&mid=2247489789&idx=1&sn=7500321728af0d8c5f5151c3845cdfe8&chksm=e9c4e99cdeb3608a6d3b749bb3bc66c737ec8eaecc5603524089dd9e94ccdc6faf3d163591ee&mpshare=1&scene=24&srcid=1025kFWafMvcAvoEiACZhRIZ&sharer_sharetime=1635119945491&sharer_shareid=5fb9813bfe9ffc983435bfc8d8c5e9ca&key=daf9bdc5abc4e8d0668e174ed925bd13abe2cea1549df1f7cac581c5f507d808682c6f629fbba4c51e87694f93df32c968fe1e2e61d91b72910ba96dfb132423bf788597b89c84e3eac1864e93887ccde5ba38a58e313db180ddf8755e6348f6035349c510be67dda07301264e3a4fdd308f8259fed3190726737abca97a422a&ascene=0&uin=MTEwNTU1MjgwMw%3D%3D&devicetype=Windows+11+x64&version=63090b19&lang=zh_CN&countrycode=CN&exportkey=n_ChQIAhIQSytgNVaWUYq3EC%2F6TddbUxLmAQIE97dBBAEAAAAAAPPWNWuHIBIAAAAOpnltbLcz9gKNyK89dVj0V%2BacR5uyLA%2FbYIt3pc6ww8fw5Zalb41o5Pe7ngEoTZbuDaSnibcMGwVSrA6DWoQq5XRqKUwIKcvhKVlq7efEbIK%2BHlY2dx4ctjcsEGX1M0U6VgzxfyknRTxKNB4H8%2BFbtq265tU5AmDzy%2FsHLTuHcy4saZ1af0CoAL2Wtc0vswjonkFmA7z8pAQaDryUyIWQBRhkoyhvF5yp36GxpyJeJrA0EbwqJ%2FlrhLGs2n626aQlidy08JLu3%2FvAwiYRsiAd&acctmode=0&pass_ticket=1zGvpvMXxh4d8ZJ3X5Oqa%2FzKt7RL1IFJFAl3BFEERDVkEv2a3H8saYAEMsyXHJFo&wx_header=1&fasttmpl_type=0&fasttmpl_fullversion=7351805-zh_CN-zip&fasttmpl_flag=1) 正在做的。你可以在 `/etc/logrotate.d/rsyslog` 下找到此配置文件。我的配置文件如下：

1. `/var/log/syslog`
    
2. `{`
    
3.         `rotate 7`
    
4.         `daily`
    
5.         `missingok`
    
6.         `notifempty`
    
7.         `delaycompress`
    
8.         `compress`
    
9.         `postrotate`
    
10.                 `reload rsyslog > /dev/null 2>&1 || true`
    
11.         `endscript`
    
12. `}`
    

14. `/var/log/mail.info`
    
15. `/var/log/mail.warn`
    
16. `/var/log/mail.err`
    
17. `/var/log/mail.log`
    
18. `/var/log/daemon.log`
    
19. `/var/log/kern.log`
    
20. `/var/log/auth.log`
    
21. `/var/log/user.log`
    
22. `/var/log/lpr.log`
    
23. `/var/log/cron.log`
    
24. `/var/log/debug`
    
25. `/var/log/messages`
    

27. `{`
    
28.         `rotate 4`
    
29.         `weekly`
    
30.         `missingok`
    
31.         `notifempty`
    
32.         `compress`
    
33.         `delaycompress`
    
34.         `sharedscripts`
    
35.         `postrotate`
    
36.                 `reload rsyslog > /dev/null 2>&1 || true`
    
37.         `endscript`
    
38. `}`
    

该文件首先定义了轮转 `/var/log/syslog` 文件的说明，这些说明包含在后面的花括号中。以下是它们的含义：

◈ `rotate 7`: 保留最近 7 次轮转的日志。然后开始删除超出的。

◈ `daily`: 每天轮转日志，与 `rotate 7` 一起使用，这意味着日志将保留过去 7 天。其它选项是每周、每月、每年。还有一个大小参数，如果日志文件的大小增加超过指定的限制（例如，大小 10k、大小 10M、大小 10G 等），则将轮转日志文件。如果未指定任何内容，日志将在运行 [logrotate](https://mp.weixin.qq.com/s?__biz=MzI1NDQwNDYyMg==&mid=2247489789&idx=1&sn=7500321728af0d8c5f5151c3845cdfe8&chksm=e9c4e99cdeb3608a6d3b749bb3bc66c737ec8eaecc5603524089dd9e94ccdc6faf3d163591ee&mpshare=1&scene=24&srcid=1025kFWafMvcAvoEiACZhRIZ&sharer_sharetime=1635119945491&sharer_shareid=5fb9813bfe9ffc983435bfc8d8c5e9ca&key=daf9bdc5abc4e8d0668e174ed925bd13abe2cea1549df1f7cac581c5f507d808682c6f629fbba4c51e87694f93df32c968fe1e2e61d91b72910ba96dfb132423bf788597b89c84e3eac1864e93887ccde5ba38a58e313db180ddf8755e6348f6035349c510be67dda07301264e3a4fdd308f8259fed3190726737abca97a422a&ascene=0&uin=MTEwNTU1MjgwMw%3D%3D&devicetype=Windows+11+x64&version=63090b19&lang=zh_CN&countrycode=CN&exportkey=n_ChQIAhIQSytgNVaWUYq3EC%2F6TddbUxLmAQIE97dBBAEAAAAAAPPWNWuHIBIAAAAOpnltbLcz9gKNyK89dVj0V%2BacR5uyLA%2FbYIt3pc6ww8fw5Zalb41o5Pe7ngEoTZbuDaSnibcMGwVSrA6DWoQq5XRqKUwIKcvhKVlq7efEbIK%2BHlY2dx4ctjcsEGX1M0U6VgzxfyknRTxKNB4H8%2BFbtq265tU5AmDzy%2FsHLTuHcy4saZ1af0CoAL2Wtc0vswjonkFmA7z8pAQaDryUyIWQBRhkoyhvF5yp36GxpyJeJrA0EbwqJ%2FlrhLGs2n626aQlidy08JLu3%2FvAwiYRsiAd&acctmode=0&pass_ticket=1zGvpvMXxh4d8ZJ3X5Oqa%2FzKt7RL1IFJFAl3BFEERDVkEv2a3H8saYAEMsyXHJFo&wx_header=1&fasttmpl_type=0&fasttmpl_fullversion=7351805-zh_CN-zip&fasttmpl_flag=1) 时轮转。你甚至可以在 cron 中运行 [logrotate](https://mp.weixin.qq.com/s?__biz=MzI1NDQwNDYyMg==&mid=2247489789&idx=1&sn=7500321728af0d8c5f5151c3845cdfe8&chksm=e9c4e99cdeb3608a6d3b749bb3bc66c737ec8eaecc5603524089dd9e94ccdc6faf3d163591ee&mpshare=1&scene=24&srcid=1025kFWafMvcAvoEiACZhRIZ&sharer_sharetime=1635119945491&sharer_shareid=5fb9813bfe9ffc983435bfc8d8c5e9ca&key=daf9bdc5abc4e8d0668e174ed925bd13abe2cea1549df1f7cac581c5f507d808682c6f629fbba4c51e87694f93df32c968fe1e2e61d91b72910ba96dfb132423bf788597b89c84e3eac1864e93887ccde5ba38a58e313db180ddf8755e6348f6035349c510be67dda07301264e3a4fdd308f8259fed3190726737abca97a422a&ascene=0&uin=MTEwNTU1MjgwMw%3D%3D&devicetype=Windows+11+x64&version=63090b19&lang=zh_CN&countrycode=CN&exportkey=n_ChQIAhIQSytgNVaWUYq3EC%2F6TddbUxLmAQIE97dBBAEAAAAAAPPWNWuHIBIAAAAOpnltbLcz9gKNyK89dVj0V%2BacR5uyLA%2FbYIt3pc6ww8fw5Zalb41o5Pe7ngEoTZbuDaSnibcMGwVSrA6DWoQq5XRqKUwIKcvhKVlq7efEbIK%2BHlY2dx4ctjcsEGX1M0U6VgzxfyknRTxKNB4H8%2BFbtq265tU5AmDzy%2FsHLTuHcy4saZ1af0CoAL2Wtc0vswjonkFmA7z8pAQaDryUyIWQBRhkoyhvF5yp36GxpyJeJrA0EbwqJ%2FlrhLGs2n626aQlidy08JLu3%2FvAwiYRsiAd&acctmode=0&pass_ticket=1zGvpvMXxh4d8ZJ3X5Oqa%2FzKt7RL1IFJFAl3BFEERDVkEv2a3H8saYAEMsyXHJFo&wx_header=1&fasttmpl_type=0&fasttmpl_fullversion=7351805-zh_CN-zip&fasttmpl_flag=1) 以便在更具体的时间间隔内使用它。

◈ `missingok`: 如果日志文件缺失也没关系。不要惊慌。

◈ `notifempty`: 日志文件为空时不轮转。

◈ `compress`: 开启压缩，使用 `nocompress` 关闭它。

◈ `delaycompress`: 如果压缩已打开，则将压缩延迟到下一次轮转。这允许至少存在一个轮转但未压缩的文件。如果你希望昨天的日志保持未压缩以便进行故障排除，那么此配置会很有用。如果某些程序在重新启动/重新加载之前可能仍然写入旧文件，这也很有帮助，例如 Apache。

◈ `postrotate/endscript`: 轮转后运行此部分中的脚本。有助于做清理工作。还有一个 `prerotate/endscript` 用于在轮转开始之前执行操作。

你能弄清楚下一节对上面配置中提到的所有文件做了什么吗？第二节中唯一多出的参数是 `sharedscripts`，它告诉 [logrotate](https://mp.weixin.qq.com/s?__biz=MzI1NDQwNDYyMg==&mid=2247489789&idx=1&sn=7500321728af0d8c5f5151c3845cdfe8&chksm=e9c4e99cdeb3608a6d3b749bb3bc66c737ec8eaecc5603524089dd9e94ccdc6faf3d163591ee&mpshare=1&scene=24&srcid=1025kFWafMvcAvoEiACZhRIZ&sharer_sharetime=1635119945491&sharer_shareid=5fb9813bfe9ffc983435bfc8d8c5e9ca&key=daf9bdc5abc4e8d0668e174ed925bd13abe2cea1549df1f7cac581c5f507d808682c6f629fbba4c51e87694f93df32c968fe1e2e61d91b72910ba96dfb132423bf788597b89c84e3eac1864e93887ccde5ba38a58e313db180ddf8755e6348f6035349c510be67dda07301264e3a4fdd308f8259fed3190726737abca97a422a&ascene=0&uin=MTEwNTU1MjgwMw%3D%3D&devicetype=Windows+11+x64&version=63090b19&lang=zh_CN&countrycode=CN&exportkey=n_ChQIAhIQSytgNVaWUYq3EC%2F6TddbUxLmAQIE97dBBAEAAAAAAPPWNWuHIBIAAAAOpnltbLcz9gKNyK89dVj0V%2BacR5uyLA%2FbYIt3pc6ww8fw5Zalb41o5Pe7ngEoTZbuDaSnibcMGwVSrA6DWoQq5XRqKUwIKcvhKVlq7efEbIK%2BHlY2dx4ctjcsEGX1M0U6VgzxfyknRTxKNB4H8%2BFbtq265tU5AmDzy%2FsHLTuHcy4saZ1af0CoAL2Wtc0vswjonkFmA7z8pAQaDryUyIWQBRhkoyhvF5yp36GxpyJeJrA0EbwqJ%2FlrhLGs2n626aQlidy08JLu3%2FvAwiYRsiAd&acctmode=0&pass_ticket=1zGvpvMXxh4d8ZJ3X5Oqa%2FzKt7RL1IFJFAl3BFEERDVkEv2a3H8saYAEMsyXHJFo&wx_header=1&fasttmpl_type=0&fasttmpl_fullversion=7351805-zh_CN-zip&fasttmpl_flag=1) 在所有日志轮转完成之前不要运行 `postrotate/endscript` 中的部分。它可以防止脚本在每一次轮转时执行，只在最后一次轮转完成时执行。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

看点新的东西

我使用下面的配置来处理我系统上的 `Nginx` 的访问和错误日志。

1. `/var/log/nginx/access.log`
    
2. `/var/log/nginx/error.log  {`
    
3.         `size 1`
    
4.         `missingok`
    
5.         `notifempty`
    
6.         `create 544 www-data adm`
    
7.         `rotate 30`
    
8.         `compress`
    
9.         `delaycompress`
    
10.         `dateext`
    
11.         `dateformat -%Y-%m-%d-%s`
    
12.         `sharedscripts`
    
13.         `extension .log`
    
14.         `postrotate`
    
15.                 `service nginx reload`
    
16.         `endscript`
    
17. `}`
    

上面的脚本可以使用如下命令运行：

1. `logrotate -vs state-file /tmp/logrotate`
    

第一次运行该命令会给出以下输出：

1. `reading config file /tmp/logrotate`
    
2. `extension is now .log`
    

4. `Handling 1 logs`
    

6. `rotating pattern: /var/log/nginx/access.log`
    
7. `/var/log/nginx/error.log   1 bytes (30 rotations)`
    
8. `empty log files are not rotated, old logs are removed`
    
9. `considering log /var/log/nginx/access.log`
    
10.   `log needs rotating`
    
11. `considering log /var/log/nginx/error.log`
    
12.   `log does not need rotating`
    
13. `rotating log /var/log/nginx/access.log, log-&gt;rotateCount is 30`
    
14. `Converted ' -%Y-%m-%d-%s' -&gt; '-%Y-%m-%d-%s'`
    
15. `dateext suffix '-2021-08-27-1485508250'`
    
16. `glob pattern '-[0-9][0-9][0-9][0-9]-[0-9][0-9]-[0-9][0-9]-[0-9][0-9][0-9][0-9][0-9][0-9][0-9][0-9][0-9][0-9]'`
    
17. `glob finding logs to compress failed`
    
18. `glob finding old rotated logs failed`
    
19. `renaming /var/log/nginx/access.log to /var/log/nginx/access-2021-08-27-1485508250.log`
    
20. `creating new /var/log/nginx/access.log mode = 0544 uid = 33 gid = 4`
    
21. `running postrotate script`
    
22. `* Reloading nginx configuration nginx`
    

第二次运行它：

1. `reading config file /tmp/logrotate`
    
2. `extension is now .log`
    

4. `Handling 1 logs`
    

6. `rotating pattern: /var/log/nginx/access.log`
    
7. `/var/log/nginx/error.log   1 bytes (30 rotations)`
    
8. `empty log files are not rotated, old logs are removed`
    
9. `considering log /var/log/nginx/access.log`
    
10.   `log needs rotating`
    
11. `considering log /var/log/nginx/error.log`
    
12.   `log does not need rotating`
    
13. `rotating log /var/log/nginx/access.log, log-&gt;rotateCount is 30`
    
14. `Converted ' -%Y-%m-%d-%s' -&gt; '-%Y-%m-%d-%s'`
    
15. `dateext suffix '-2021-08-27-1485508280'`
    
16. `glob pattern '-[0-9][0-9][0-9][0-9]-[0-9][0-9]-[0-9][0-9]-[0-9][0-9][0-9][0-9][0-9][0-9][0-9][0-9][0-9][0-9]'`
    
17. `compressing log with: /bin/gzip`
    
18. `renaming /var/log/nginx/access.log to /var/log/nginx/access-2021-08-27-1485508280.log`
    
19. `creating new /var/log/nginx/access.log mode = 0544 uid = 33 gid = 4`
    
20. `running postrotate script`
    
21. `* Reloading nginx configuration nginx`
    

第三次运行它：

1. `reading config file /tmp/logrotate`
    
2. `extension is now .log`
    

4. `Handling 1 logs`
    

6. `rotating pattern: /var/log/nginx/access.log`
    
7. `/var/log/nginx/error.log   1 bytes (30 rotations)`
    
8. `empty log files are not rotated, old logs are removed`
    
9. `considering log /var/log/nginx/access.log`
    
10.   `log needs rotating`
    
11. `considering log /var/log/nginx/error.log`
    
12.   `log does not need rotating`
    
13. `rotating log /var/log/nginx/access.log, log-&gt;rotateCount is 30`
    
14. `Converted ' -%Y-%m-%d-%s' -&gt; '-%Y-%m-%d-%s'`
    
15. `dateext suffix '-2021-08-27-1485508316'`
    
16. `glob pattern '-[0-9][0-9][0-9][0-9]-[0-9][0-9]-[0-9][0-9]-[0-9][0-9][0-9][0-9][0-9][0-9][0-9][0-9][0-9][0-9]'`
    
17. `compressing log with: /bin/gzip`
    
18. `renaming /var/log/nginx/access.log to /var/log/nginx/access-2021-08-27-1485508316.log`
    
19. `creating new /var/log/nginx/access.log mode = 0544 uid = 33 gid = 4`
    
20. `running postrotate script`
    
21. `* Reloading nginx configuration nginx`
    

状态文件的内容如下所示：

1. `logrotate state -- version 2`
    
2. `"/var/log/nginx/error.log" 2021-08-27-9:0:0`
    
3. `"/var/log/nginx/access.log" 2021-08-27-9:11:56`
    

◈ [下载 Linux logrotate 备忘单](https://mp.weixin.qq.com/s?__biz=MzI1NDQwNDYyMg==&mid=2247489789&idx=1&sn=7500321728af0d8c5f5151c3845cdfe8&chksm=e9c4e99cdeb3608a6d3b749bb3bc66c737ec8eaecc5603524089dd9e94ccdc6faf3d163591ee&mpshare=1&scene=24&srcid=1025kFWafMvcAvoEiACZhRIZ&sharer_sharetime=1635119945491&sharer_shareid=5fb9813bfe9ffc983435bfc8d8c5e9ca&key=daf9bdc5abc4e8d0668e174ed925bd13abe2cea1549df1f7cac581c5f507d808682c6f629fbba4c51e87694f93df32c968fe1e2e61d91b72910ba96dfb132423bf788597b89c84e3eac1864e93887ccde5ba38a58e313db180ddf8755e6348f6035349c510be67dda07301264e3a4fdd308f8259fed3190726737abca97a422a&ascene=0&uin=MTEwNTU1MjgwMw%3D%3D&devicetype=Windows+11+x64&version=63090b19&lang=zh_CN&countrycode=CN&exportkey=n_ChQIAhIQSytgNVaWUYq3EC%2F6TddbUxLmAQIE97dBBAEAAAAAAPPWNWuHIBIAAAAOpnltbLcz9gKNyK89dVj0V%2BacR5uyLA%2FbYIt3pc6ww8fw5Zalb41o5Pe7ngEoTZbuDaSnibcMGwVSrA6DWoQq5XRqKUwIKcvhKVlq7efEbIK%2BHlY2dx4ctjcsEGX1M0U6VgzxfyknRTxKNB4H8%2BFbtq265tU5AmDzy%2FsHLTuHcy4saZ1af0CoAL2Wtc0vswjonkFmA7z8pAQaDryUyIWQBRhkoyhvF5yp36GxpyJeJrA0EbwqJ%2FlrhLGs2n626aQlidy08JLu3%2FvAwiYRsiAd&acctmode=0&pass_ticket=1zGvpvMXxh4d8ZJ3X5Oqa%2FzKt7RL1IFJFAl3BFEERDVkEv2a3H8saYAEMsyXHJFo&wx_header=1&fasttmpl_type=0&fasttmpl_fullversion=7351805-zh_CN-zip&fasttmpl_flag=1)🔗 opensource.com

本文首发于[作者个人博客](https://mp.weixin.qq.com/s?__biz=MzI1NDQwNDYyMg==&mid=2247489789&idx=1&sn=7500321728af0d8c5f5151c3845cdfe8&chksm=e9c4e99cdeb3608a6d3b749bb3bc66c737ec8eaecc5603524089dd9e94ccdc6faf3d163591ee&mpshare=1&scene=24&srcid=1025kFWafMvcAvoEiACZhRIZ&sharer_sharetime=1635119945491&sharer_shareid=5fb9813bfe9ffc983435bfc8d8c5e9ca&key=daf9bdc5abc4e8d0668e174ed925bd13abe2cea1549df1f7cac581c5f507d808682c6f629fbba4c51e87694f93df32c968fe1e2e61d91b72910ba96dfb132423bf788597b89c84e3eac1864e93887ccde5ba38a58e313db180ddf8755e6348f6035349c510be67dda07301264e3a4fdd308f8259fed3190726737abca97a422a&ascene=0&uin=MTEwNTU1MjgwMw%3D%3D&devicetype=Windows+11+x64&version=63090b19&lang=zh_CN&countrycode=CN&exportkey=n_ChQIAhIQSytgNVaWUYq3EC%2F6TddbUxLmAQIE97dBBAEAAAAAAPPWNWuHIBIAAAAOpnltbLcz9gKNyK89dVj0V%2BacR5uyLA%2FbYIt3pc6ww8fw5Zalb41o5Pe7ngEoTZbuDaSnibcMGwVSrA6DWoQq5XRqKUwIKcvhKVlq7efEbIK%2BHlY2dx4ctjcsEGX1M0U6VgzxfyknRTxKNB4H8%2BFbtq265tU5AmDzy%2FsHLTuHcy4saZ1af0CoAL2Wtc0vswjonkFmA7z8pAQaDryUyIWQBRhkoyhvF5yp36GxpyJeJrA0EbwqJ%2FlrhLGs2n626aQlidy08JLu3%2FvAwiYRsiAd&acctmode=0&pass_ticket=1zGvpvMXxh4d8ZJ3X5Oqa%2FzKt7RL1IFJFAl3BFEERDVkEv2a3H8saYAEMsyXHJFo&wx_header=1&fasttmpl_type=0&fasttmpl_fullversion=7351805-zh_CN-zip&fasttmpl_flag=1)🔗 notes.ayushsharma.in，经授权改编。

---

via: [https://opensource.com/article/21/10/linux-logrotate](https://mp.weixin.qq.com/s?__biz=MzI1NDQwNDYyMg==&mid=2247489789&idx=1&sn=7500321728af0d8c5f5151c3845cdfe8&chksm=e9c4e99cdeb3608a6d3b749bb3bc66c737ec8eaecc5603524089dd9e94ccdc6faf3d163591ee&mpshare=1&scene=24&srcid=1025kFWafMvcAvoEiACZhRIZ&sharer_sharetime=1635119945491&sharer_shareid=5fb9813bfe9ffc983435bfc8d8c5e9ca&key=daf9bdc5abc4e8d0668e174ed925bd13abe2cea1549df1f7cac581c5f507d808682c6f629fbba4c51e87694f93df32c968fe1e2e61d91b72910ba96dfb132423bf788597b89c84e3eac1864e93887ccde5ba38a58e313db180ddf8755e6348f6035349c510be67dda07301264e3a4fdd308f8259fed3190726737abca97a422a&ascene=0&uin=MTEwNTU1MjgwMw%3D%3D&devicetype=Windows+11+x64&version=63090b19&lang=zh_CN&countrycode=CN&exportkey=n_ChQIAhIQSytgNVaWUYq3EC%2F6TddbUxLmAQIE97dBBAEAAAAAAPPWNWuHIBIAAAAOpnltbLcz9gKNyK89dVj0V%2BacR5uyLA%2FbYIt3pc6ww8fw5Zalb41o5Pe7ngEoTZbuDaSnibcMGwVSrA6DWoQq5XRqKUwIKcvhKVlq7efEbIK%2BHlY2dx4ctjcsEGX1M0U6VgzxfyknRTxKNB4H8%2BFbtq265tU5AmDzy%2FsHLTuHcy4saZ1af0CoAL2Wtc0vswjonkFmA7z8pAQaDryUyIWQBRhkoyhvF5yp36GxpyJeJrA0EbwqJ%2FlrhLGs2n626aQlidy08JLu3%2FvAwiYRsiAd&acctmode=0&pass_ticket=1zGvpvMXxh4d8ZJ3X5Oqa%2FzKt7RL1IFJFAl3BFEERDVkEv2a3H8saYAEMsyXHJFo&wx_header=1&fasttmpl_type=0&fasttmpl_fullversion=7351805-zh_CN-zip&fasttmpl_flag=1)

作者：[Ayush Sharma](https://mp.weixin.qq.com/s?__biz=MzI1NDQwNDYyMg==&mid=2247489789&idx=1&sn=7500321728af0d8c5f5151c3845cdfe8&chksm=e9c4e99cdeb3608a6d3b749bb3bc66c737ec8eaecc5603524089dd9e94ccdc6faf3d163591ee&mpshare=1&scene=24&srcid=1025kFWafMvcAvoEiACZhRIZ&sharer_sharetime=1635119945491&sharer_shareid=5fb9813bfe9ffc983435bfc8d8c5e9ca&key=daf9bdc5abc4e8d0668e174ed925bd13abe2cea1549df1f7cac581c5f507d808682c6f629fbba4c51e87694f93df32c968fe1e2e61d91b72910ba96dfb132423bf788597b89c84e3eac1864e93887ccde5ba38a58e313db180ddf8755e6348f6035349c510be67dda07301264e3a4fdd308f8259fed3190726737abca97a422a&ascene=0&uin=MTEwNTU1MjgwMw%3D%3D&devicetype=Windows+11+x64&version=63090b19&lang=zh_CN&countrycode=CN&exportkey=n_ChQIAhIQSytgNVaWUYq3EC%2F6TddbUxLmAQIE97dBBAEAAAAAAPPWNWuHIBIAAAAOpnltbLcz9gKNyK89dVj0V%2BacR5uyLA%2FbYIt3pc6ww8fw5Zalb41o5Pe7ngEoTZbuDaSnibcMGwVSrA6DWoQq5XRqKUwIKcvhKVlq7efEbIK%2BHlY2dx4ctjcsEGX1M0U6VgzxfyknRTxKNB4H8%2BFbtq265tU5AmDzy%2FsHLTuHcy4saZ1af0CoAL2Wtc0vswjonkFmA7z8pAQaDryUyIWQBRhkoyhvF5yp36GxpyJeJrA0EbwqJ%2FlrhLGs2n626aQlidy08JLu3%2FvAwiYRsiAd&acctmode=0&pass_ticket=1zGvpvMXxh4d8ZJ3X5Oqa%2FzKt7RL1IFJFAl3BFEERDVkEv2a3H8saYAEMsyXHJFo&wx_header=1&fasttmpl_type=0&fasttmpl_fullversion=7351805-zh_CN-zip&fasttmpl_flag=1) 选题：[lujun9972](https://mp.weixin.qq.com/s?__biz=MzI1NDQwNDYyMg==&mid=2247489789&idx=1&sn=7500321728af0d8c5f5151c3845cdfe8&chksm=e9c4e99cdeb3608a6d3b749bb3bc66c737ec8eaecc5603524089dd9e94ccdc6faf3d163591ee&mpshare=1&scene=24&srcid=1025kFWafMvcAvoEiACZhRIZ&sharer_sharetime=1635119945491&sharer_shareid=5fb9813bfe9ffc983435bfc8d8c5e9ca&key=daf9bdc5abc4e8d0668e174ed925bd13abe2cea1549df1f7cac581c5f507d808682c6f629fbba4c51e87694f93df32c968fe1e2e61d91b72910ba96dfb132423bf788597b89c84e3eac1864e93887ccde5ba38a58e313db180ddf8755e6348f6035349c510be67dda07301264e3a4fdd308f8259fed3190726737abca97a422a&ascene=0&uin=MTEwNTU1MjgwMw%3D%3D&devicetype=Windows+11+x64&version=63090b19&lang=zh_CN&countrycode=CN&exportkey=n_ChQIAhIQSytgNVaWUYq3EC%2F6TddbUxLmAQIE97dBBAEAAAAAAPPWNWuHIBIAAAAOpnltbLcz9gKNyK89dVj0V%2BacR5uyLA%2FbYIt3pc6ww8fw5Zalb41o5Pe7ngEoTZbuDaSnibcMGwVSrA6DWoQq5XRqKUwIKcvhKVlq7efEbIK%2BHlY2dx4ctjcsEGX1M0U6VgzxfyknRTxKNB4H8%2BFbtq265tU5AmDzy%2FsHLTuHcy4saZ1af0CoAL2Wtc0vswjonkFmA7z8pAQaDryUyIWQBRhkoyhvF5yp36GxpyJeJrA0EbwqJ%2FlrhLGs2n626aQlidy08JLu3%2FvAwiYRsiAd&acctmode=0&pass_ticket=1zGvpvMXxh4d8ZJ3X5Oqa%2FzKt7RL1IFJFAl3BFEERDVkEv2a3H8saYAEMsyXHJFo&wx_header=1&fasttmpl_type=0&fasttmpl_fullversion=7351805-zh_CN-zip&fasttmpl_flag=1) 译者：[perfiffer](https://mp.weixin.qq.com/s?__biz=MzI1NDQwNDYyMg==&mid=2247489789&idx=1&sn=7500321728af0d8c5f5151c3845cdfe8&chksm=e9c4e99cdeb3608a6d3b749bb3bc66c737ec8eaecc5603524089dd9e94ccdc6faf3d163591ee&mpshare=1&scene=24&srcid=1025kFWafMvcAvoEiACZhRIZ&sharer_sharetime=1635119945491&sharer_shareid=5fb9813bfe9ffc983435bfc8d8c5e9ca&key=daf9bdc5abc4e8d0668e174ed925bd13abe2cea1549df1f7cac581c5f507d808682c6f629fbba4c51e87694f93df32c968fe1e2e61d91b72910ba96dfb132423bf788597b89c84e3eac1864e93887ccde5ba38a58e313db180ddf8755e6348f6035349c510be67dda07301264e3a4fdd308f8259fed3190726737abca97a422a&ascene=0&uin=MTEwNTU1MjgwMw%3D%3D&devicetype=Windows+11+x64&version=63090b19&lang=zh_CN&countrycode=CN&exportkey=n_ChQIAhIQSytgNVaWUYq3EC%2F6TddbUxLmAQIE97dBBAEAAAAAAPPWNWuHIBIAAAAOpnltbLcz9gKNyK89dVj0V%2BacR5uyLA%2FbYIt3pc6ww8fw5Zalb41o5Pe7ngEoTZbuDaSnibcMGwVSrA6DWoQq5XRqKUwIKcvhKVlq7efEbIK%2BHlY2dx4ctjcsEGX1M0U6VgzxfyknRTxKNB4H8%2BFbtq265tU5AmDzy%2FsHLTuHcy4saZ1af0CoAL2Wtc0vswjonkFmA7z8pAQaDryUyIWQBRhkoyhvF5yp36GxpyJeJrA0EbwqJ%2FlrhLGs2n626aQlidy08JLu3%2FvAwiYRsiAd&acctmode=0&pass_ticket=1zGvpvMXxh4d8ZJ3X5Oqa%2FzKt7RL1IFJFAl3BFEERDVkEv2a3H8saYAEMsyXHJFo&wx_header=1&fasttmpl_type=0&fasttmpl_fullversion=7351805-zh_CN-zip&fasttmpl_flag=1) 校对：[wxy](https://mp.weixin.qq.com/s?__biz=MzI1NDQwNDYyMg==&mid=2247489789&idx=1&sn=7500321728af0d8c5f5151c3845cdfe8&chksm=e9c4e99cdeb3608a6d3b749bb3bc66c737ec8eaecc5603524089dd9e94ccdc6faf3d163591ee&mpshare=1&scene=24&srcid=1025kFWafMvcAvoEiACZhRIZ&sharer_sharetime=1635119945491&sharer_shareid=5fb9813bfe9ffc983435bfc8d8c5e9ca&key=daf9bdc5abc4e8d0668e174ed925bd13abe2cea1549df1f7cac581c5f507d808682c6f629fbba4c51e87694f93df32c968fe1e2e61d91b72910ba96dfb132423bf788597b89c84e3eac1864e93887ccde5ba38a58e313db180ddf8755e6348f6035349c510be67dda07301264e3a4fdd308f8259fed3190726737abca97a422a&ascene=0&uin=MTEwNTU1MjgwMw%3D%3D&devicetype=Windows+11+x64&version=63090b19&lang=zh_CN&countrycode=CN&exportkey=n_ChQIAhIQSytgNVaWUYq3EC%2F6TddbUxLmAQIE97dBBAEAAAAAAPPWNWuHIBIAAAAOpnltbLcz9gKNyK89dVj0V%2BacR5uyLA%2FbYIt3pc6ww8fw5Zalb41o5Pe7ngEoTZbuDaSnibcMGwVSrA6DWoQq5XRqKUwIKcvhKVlq7efEbIK%2BHlY2dx4ctjcsEGX1M0U6VgzxfyknRTxKNB4H8%2BFbtq265tU5AmDzy%2FsHLTuHcy4saZ1af0CoAL2Wtc0vswjonkFmA7z8pAQaDryUyIWQBRhkoyhvF5yp36GxpyJeJrA0EbwqJ%2FlrhLGs2n626aQlidy08JLu3%2FvAwiYRsiAd&acctmode=0&pass_ticket=1zGvpvMXxh4d8ZJ3X5Oqa%2FzKt7RL1IFJFAl3BFEERDVkEv2a3H8saYAEMsyXHJFo&wx_header=1&fasttmpl_type=0&fasttmpl_fullversion=7351805-zh_CN-zip&fasttmpl_flag=1)

本文由 [LCTT](https://mp.weixin.qq.com/s?__biz=MzI1NDQwNDYyMg==&mid=2247489789&idx=1&sn=7500321728af0d8c5f5151c3845cdfe8&chksm=e9c4e99cdeb3608a6d3b749bb3bc66c737ec8eaecc5603524089dd9e94ccdc6faf3d163591ee&mpshare=1&scene=24&srcid=1025kFWafMvcAvoEiACZhRIZ&sharer_sharetime=1635119945491&sharer_shareid=5fb9813bfe9ffc983435bfc8d8c5e9ca&key=daf9bdc5abc4e8d0668e174ed925bd13abe2cea1549df1f7cac581c5f507d808682c6f629fbba4c51e87694f93df32c968fe1e2e61d91b72910ba96dfb132423bf788597b89c84e3eac1864e93887ccde5ba38a58e313db180ddf8755e6348f6035349c510be67dda07301264e3a4fdd308f8259fed3190726737abca97a422a&ascene=0&uin=MTEwNTU1MjgwMw%3D%3D&devicetype=Windows+11+x64&version=63090b19&lang=zh_CN&countrycode=CN&exportkey=n_ChQIAhIQSytgNVaWUYq3EC%2F6TddbUxLmAQIE97dBBAEAAAAAAPPWNWuHIBIAAAAOpnltbLcz9gKNyK89dVj0V%2BacR5uyLA%2FbYIt3pc6ww8fw5Zalb41o5Pe7ngEoTZbuDaSnibcMGwVSrA6DWoQq5XRqKUwIKcvhKVlq7efEbIK%2BHlY2dx4ctjcsEGX1M0U6VgzxfyknRTxKNB4H8%2BFbtq265tU5AmDzy%2FsHLTuHcy4saZ1af0CoAL2Wtc0vswjonkFmA7z8pAQaDryUyIWQBRhkoyhvF5yp36GxpyJeJrA0EbwqJ%2FlrhLGs2n626aQlidy08JLu3%2FvAwiYRsiAd&acctmode=0&pass_ticket=1zGvpvMXxh4d8ZJ3X5Oqa%2FzKt7RL1IFJFAl3BFEERDVkEv2a3H8saYAEMsyXHJFo&wx_header=1&fasttmpl_type=0&fasttmpl_fullversion=7351805-zh_CN-zip&fasttmpl_flag=1) 原创编译，[Linux中国](https://mp.weixin.qq.com/s?__biz=MzI1NDQwNDYyMg==&mid=2247489789&idx=1&sn=7500321728af0d8c5f5151c3845cdfe8&chksm=e9c4e99cdeb3608a6d3b749bb3bc66c737ec8eaecc5603524089dd9e94ccdc6faf3d163591ee&mpshare=1&scene=24&srcid=1025kFWafMvcAvoEiACZhRIZ&sharer_sharetime=1635119945491&sharer_shareid=5fb9813bfe9ffc983435bfc8d8c5e9ca&key=daf9bdc5abc4e8d0668e174ed925bd13abe2cea1549df1f7cac581c5f507d808682c6f629fbba4c51e87694f93df32c968fe1e2e61d91b72910ba96dfb132423bf788597b89c84e3eac1864e93887ccde5ba38a58e313db180ddf8755e6348f6035349c510be67dda07301264e3a4fdd308f8259fed3190726737abca97a422a&ascene=0&uin=MTEwNTU1MjgwMw%3D%3D&devicetype=Windows+11+x64&version=63090b19&lang=zh_CN&countrycode=CN&exportkey=n_ChQIAhIQSytgNVaWUYq3EC%2F6TddbUxLmAQIE97dBBAEAAAAAAPPWNWuHIBIAAAAOpnltbLcz9gKNyK89dVj0V%2BacR5uyLA%2FbYIt3pc6ww8fw5Zalb41o5Pe7ngEoTZbuDaSnibcMGwVSrA6DWoQq5XRqKUwIKcvhKVlq7efEbIK%2BHlY2dx4ctjcsEGX1M0U6VgzxfyknRTxKNB4H8%2BFbtq265tU5AmDzy%2FsHLTuHcy4saZ1af0CoAL2Wtc0vswjonkFmA7z8pAQaDryUyIWQBRhkoyhvF5yp36GxpyJeJrA0EbwqJ%2FlrhLGs2n626aQlidy08JLu3%2FvAwiYRsiAd&acctmode=0&pass_ticket=1zGvpvMXxh4d8ZJ3X5Oqa%2FzKt7RL1IFJFAl3BFEERDVkEv2a3H8saYAEMsyXHJFo&wx_header=1&fasttmpl_type=0&fasttmpl_fullversion=7351805-zh_CN-zip&fasttmpl_flag=1) 荣誉推出

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)  

欢迎遵照 CC-BY-NC-SA 协议规定转载，

如需转载，请在文章下留言 “转载：公众号名称”，

我们将为您添加白名单，授权“转载文章时可以修改”。

[![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)](https://mp.weixin.qq.com/s?__biz=MjM5NjQ4MjYwMQ==&mid=2664637017&idx=1&sn=418656faabf31e5a8ba301425bea0db0&scene=21#wechat_redirect)

  

阅读原文

阅读 1149

​