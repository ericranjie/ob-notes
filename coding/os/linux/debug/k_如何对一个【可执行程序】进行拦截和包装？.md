# 

Original 道哥 IOT物联网小镇  _2022年06月17日 07:55_ _江苏_

![Image](https://mmbiz.qpic.cn/mmbiz_gif/WC13ibsIvG3Y8MTyd4ia2QsjDhz1XzpjgOxWRlefNadnEbZvYFK9nVKjbjgTDZPibD2CTpglrCicGpyvBeZlSJtJRQ/640?wx_fmt=gif&tp=webp&wxfrom=5&wx_lazy=1)

> 作  者：道哥，10+年嵌入式开发老兵，专注于：C/C++、嵌入式、Linux。
>
> 关注下方公众号，回复【书籍】，获取 Linux、嵌入式领域经典书籍；回复【PDF】，获取所有原创文章( PDF 格式)。

别人的经验，我们的阶梯！

之前层写过一篇文章，讨论如何对一个库中的函数进行拦截和封装，也就是所谓的插桩。

文章的链接是：[Linux中对【库函数】的调用进行跟踪的 3 种【插桩】技巧](https://mp.weixin.qq.com/s?__biz=MzA3MzAwODYyNQ==&mid=2247487152&idx=1&sn=e5cf23a6de4995ba32058423f4edbb2f&scene=21#wechat_redirect)

文中一共讨论了`3`种方法，来实现对【函数】进行拦截：

> 1. 在编译阶段插桩;
>
> 1. 在链接阶段插桩;
>
> 1. 在执行阶段插桩;

昨天一个网友提了另外一个问题：如何对一个可执行程序进行拦截？

他提出了一个实际的示例：

`Ubuntu 18.04`操作系统中，重启指令`/sbin/reboot`是一个软链接，链接到可执行程序`/bin/systemctl`，那么是否可以在执行`systemctl`之前，做一些其它的事情(例如：保持一些应用程序的状态数据)？

> Ubuntn18.04 中使用 systemd 来管理系统的所有 Service;
>
> 除了 reboot 指令，还有其它几个指令也是软链接到 /bin/systemctl;

![Image](https://mmbiz.qpic.cn/mmbiz_png/WC13ibsIvG3Y8MTyd4ia2QsjDhz1XzpjgORELibZEdm1TOiaDnyNib054FBg4V0UWO0w8MaU0QmicUXNz0HQ6f7XYe2g/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

这里就引出一个问题了：

既然上面这`6`个命令都链接到`systemctl`，那么当`systemctl`被执行的时候，它是如何知道它是被哪一个命令调用的呢？

看一下源码就知道了：通过参数 argv\[0\] 来获得的。

我们知道，`main`函数通过`argc`和`argv[]`来获取所有的参数，如下：

`// 测试文件：test1.c      #include <stdio.h>      int main(int argc, char *argv[])   {       printf("argc = %d \n", argc);       for (int i = 0; i < argc; i++)            printf("argv[%d] = %s \n", i, argv[i]);       return 0;   }   `

编译、执行一下：

`$ gcc test1.c -o test1   $ ./test1 aaa bbb   argc = 3    argv[0] = ./test1    argv[1] = aaa    argv[2] = bbb   `

可以看到：`argv[0] = ./test1`，因为我们是在命令行直接调用`test`可执行程序的，这很容易理解。

那么：如果`test`是被一个软链接调用的呢？

测试一下，创建软链接：

`$ ln -s test1 link1   `

![Image](https://mmbiz.qpic.cn/mmbiz_png/WC13ibsIvG3Y8MTyd4ia2QsjDhz1XzpjgO9qpp5A5wfR8eY87cyZLBHicXfoqFibZNNtdrpcr8W4C44FhAYZnBOPog/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

执行一下：

![Image](https://mmbiz.qpic.cn/mmbiz_png/WC13ibsIvG3Y8MTyd4ia2QsjDhz1XzpjgOvibZ2Tl8pFyV9PhhicJMvpleoCCDrbDK2ZmeK7d7SF5iaUbZZ5vpfiaaDQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

此时，`argv[0] = ./link1`。

也就是说：第一个参数存放的是软链接文件路径，systemctl 的道理也是如此！

知道了这个原理，那我们就可以在`reboot`与`systemc`之间横叉一刀，增加一个中间可执行文件：

![Image](https://mmbiz.qpic.cn/mmbiz_png/WC13ibsIvG3Y8MTyd4ia2QsjDhz1XzpjgOWB91tZDHrMuPkxAmd5hYwibb1srHaib0xFrlf2Qksrbp6ibYoSMso1EmQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

为了便于描述，我们把这个中间文件创建为脚本`pre_systemctl.sh`，然后把`root`软链接到这个脚本。

注意：在理解原理之前，建议不要直接用 reboot 等系统命令进行操作，可以自己写一些测试程序，例如上面的 test。

操作如下：

`$ cd /sbin   $ sudo rm root   $ sudo touch pre_systemctl.sh   $ sudo chmod +x pre_systemctl.sh   $ sudo ln -s pre_systemctl.sh reboot   `

![Image](https://mmbiz.qpic.cn/mmbiz_png/WC13ibsIvG3Y8MTyd4ia2QsjDhz1XzpjgO35AybKpEHZxAibYwzubcVCPBC50aeJwgiczJXdpZTYnIjtqCNJVHs6Dg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

创建了`pre_systemctl.sh`脚本之后，并且把`reboot`软链接到它，在脚本中输入如下内容：

![Image](https://mmbiz.qpic.cn/mmbiz_png/WC13ibsIvG3Y8MTyd4ia2QsjDhz1XzpjgOlkKf06eUwial9icZfzDgOmDqbRgQhXGpAAe8Hcmqeic4ia06Ssu3Dla9mg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

此时，在命令行中执行`reboot`命令，就会执行这个脚本，并且这个脚本也能够正确的把`/sbin/root`作为第`0`个参数传递给`/bin/systemctl`，如下图所示：

![Image](https://mmbiz.qpic.cn/mmbiz_png/WC13ibsIvG3Y8MTyd4ia2QsjDhz1XzpjgOJNia3vDjuXh0zgNbQ1HdJMe3CFmhLDoPox9pqnu5w4RyGsbCic0siatIw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

在这个脚本中，可以在执行`systemctl`之前，做任何需要关机前需要处理的一些事情。

问题似乎是解决了，但是好像还有一个问题：

如果用户在执行命令时输入了一些其它的参数，这个脚本程序也应该透明的把这些参数传递给 systemctl 才可以！

为了便于观察，我们在脚本中多打印个参数，并通过`exec`来启动`systemctl`，并且强制把参数`$0`设置为`systemctl`的第`0`个参数：

![Image](https://mmbiz.qpic.cn/mmbiz_png/WC13ibsIvG3Y8MTyd4ia2QsjDhz1XzpjgO0VibAhI8vKfUQOOibRiaFH507c8L7UlqZp1RHgPpk7GgdsiaDs0uaVlBmQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

这个脚本文件中的重点是最后一条命令：

`exec -a $0 /bin/systemctl $*   `

此时，在命令行中执行`reboot`指令，输出如下：

![Image](https://mmbiz.qpic.cn/mmbiz_png/WC13ibsIvG3Y8MTyd4ia2QsjDhz1XzpjgOW0z19fUG9EibgMOmCibJWSVB2CprddAvze1yH08Ypg9uY7jlKcNcCqng/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

如此调用`systemctl`，就解决了刚才提出的问题，而且通过 `$*`，可以把任意多个参数透明的传递下去。

这里的关键还是 exec 的参数 -a ，看一下它的指令说明：

`exec [-cl] [-a name] [command [arguments ...]] [redirection ...]   `

这里还有一个更详细的说明：

![Image](https://mmbiz.qpic.cn/mmbiz_png/WC13ibsIvG3Y8MTyd4ia2QsjDhz1XzpjgOibrOeZvqsNvfYfGzDGiaItOtN6oiaiaGGvTlalibr94h3VFOwxCzV6PXjibA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

------ End ------

以上就是今天分享的内容，感谢您的阅读！

既然看到这里了，如果觉得不错，请您随手点个【赞】和【在看】吧!

如果转载本文，文末务必注明：“转自微信公众号：IOT物联网小镇”。

推荐阅读

[【1】C语言指针-从底层原理到花式技巧，用图文和代码讲解透彻](https://mp.weixin.qq.com/s?__biz=MzA3MzAwODYyNQ==&mid=2247484230&idx=1&sn=da68a36ddeb0dcab1b4ed685130ccf2e&scene=21#wechat_redirect)

[【2】GCC 链接过程中的【重定位】过程分析](https://mp.weixin.qq.com/s?__biz=MzA3MzAwODYyNQ==&mid=2247487353&idx=1&sn=533701348cbdf80ed48aee2195312d92&scene=21#wechat_redirect)

[【3】Linux 动态链接过程中的【重定位】底层原理](https://mp.weixin.qq.com/s?__biz=MzA3MzAwODYyNQ==&mid=2247487393&idx=1&sn=0b31d24d6bd182a26f1b04a3483646d7&scene=21#wechat_redirect)

[【4】原来gdb的底层调试原理这么简单](https://mp.weixin.qq.com/s?__biz=MzA3MzAwODYyNQ==&mid=2247483851&idx=1&sn=653c764eb480dff47eee96403ec5f821&scene=21#wechat_redirect)

[【5】gcc编译时，链接器安排的【虚拟地址】是如何计算出来的？](https://mp.weixin.qq.com/s?__biz=MzA3MzAwODYyNQ==&mid=2247487318&idx=1&sn=571f25a1d1fa62f906590053d959001a&scene=21#wechat_redirect)

[【6】Linux中对【库函数】的调用进行跟踪的3种【插桩】技巧](https://mp.weixin.qq.com/s?__biz=MzA3MzAwODYyNQ==&mid=2247487152&idx=1&sn=e5cf23a6de4995ba32058423f4edbb2f&scene=21#wechat_redirect)

![Image](https://mmbiz.qpic.cn/mmbiz/WC13ibsIvG3Y8MTyd4ia2QsjDhz1XzpjgO2pHiblXVOpibDojdhad1hKASFouuT2W0uyYPict30mGNeWEz8uooicygJA/640?wx_fmt=bmp&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

星标公众号，第一时间看文章！

​

![](https://mp.weixin.qq.com/rr?timestamp=1727516797&src=11&ver=1&signature=vTjFqx4nK-bb5xX52UiEZfoT-h5pWuaLWlJHOj-bwaypZWfjFik0-LGhvNXajI8RjHpLkKfeCw4iATqrbHPf2SSyUXVPzYZaRd0txYe46oY=)

Scan to Follow
