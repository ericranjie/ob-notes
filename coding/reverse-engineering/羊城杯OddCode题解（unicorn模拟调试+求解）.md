# 

34r7hm4n 看雪学苑

_2021年09月30日 18:05_

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/kR3MmOkZ9r7NJD6Jl8AP0f6ic2u1FE9uwLemaaJWFzra2zWJnoPGKEkMUUIXJvnp1yzHeCdvew2NVGA4pIfiauOA/640?wx_fmt=jpeg&wxfrom=13&tp=wxpic)

本文为看雪论坛优秀文章\
看雪论坛作者ID：34r7hm4n

首先恭喜0x401 Team首次在CTF比赛中夺得第一名，顺便和学弟AK了逆向，战队能取得今天的成绩离不开队员的努力。但是不得不承认另一个原因是，很多强队的火力都被隔壁RCTF吸引了，我们还需要继续努力：

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

自从上次强网杯unicorn那题以来我就一直对unicorn很感兴趣，但平时又没有用unicorn解决实际问题的场景，这次羊城杯总算碰到了，借此机会学习一下unicorn。OddCode这题有大量花指令和垃圾跳转，手动分析几乎不可能，如果使用unicorn模拟执行会方便很多。

比赛时一直爆肝到凌晨5点才把这题弄出来（被屑出题人的大小写坑了3个小时），本文的解法是我赛后优化之后的解法。

1

# **概览-32位模式部分**

首先这是一个32位的可执行文件：\
!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)\
没有main函数，程序直接从start函数开始执行，首先是一段校验输入格式的代码：

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

一个很奇怪的远跳转，一开始在IDA看了半天没明白是什么意思，用WinDbg调试后才发现IDA的反汇编有问题：

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

实际上是一个远跳转到33:2E5310这个地址，远跳转有一个隐形的操作，他会将代码段寄存器CS设置为跳转到的这个段对应的段选择子，这里执行完了远跳转之后，CS的值被置为0x33：

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

这里涉及一个我之前折腾自制操作系统时接触到的一个知识点——在Windows中，程序可以通过修改代码段寄存器切换32位模式和64位模式，当CS为0x33时，CPU按64位模式执行指令，当CS为0x23,时，CPU按32位模式执行指令。执行完这个远跳转后，程序跳转到2E5310这个地址（也就是下一条指令），CPU切换到64位模式执行，所以接下来的代码都要按64位模式解析。

这个技术的一个典型应用是在恶意代码领域，参考：天堂之门（Heaven’s Gate）技术的详细分析\_（https://www.freebuf.com/articles/web/209983.html）\_

切换到64位模式后，执行sub_2E1010函数：\
!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)\
接下来一段代码的作用是将CS的值改回0x23，切回32位模式：\
!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)\
最后根据sub_2E1010函数的返回值判断输入是否正确，所以本题的关键是sub_2E1010函数：\
!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

# 

2

# **概览-64位模式部分**

先到回到这个部分，远跳转前的两个lea语句相当于是传递参数，把input存入esi，把key存入edi：

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

key是一个16字节的数组，推测之后加密或者校验输入的时候会用上：\
!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)\
从sub_2E1010函数开始的代码要用64位模式查看：\
!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

从这里开始有大量的垃圾代码和花指令，手动分析了半天都没找到关键代码，于是我决定用unicorn写一个模拟调试器帮我找到关键代码。

3

# **unicorn模拟调试**

最开始看到用unicorn实现调试器的思路是在这篇文章：汇编与反汇编神器Unicorn\_（https://www.52pojie.cn/thread-1026209-1-1.html）\_。里面用到的调试器代码貌似出自无名侠：\
!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)\
我们也来写个简单的调试器来模拟64位代码的执行，并且实现一个tracer，用来跟踪代码块执行的轨迹：

```
from unicorn import *
```

unicorn可以hook代码块执行，但是会被花指令干扰，所以这里通过hook指令执行，再判断当前的地址是否与上次执行的地址+上一条指令的长度是否相等来判断是否发生了代码块跳转：

```
def trace(self, mu, address, size, data):
```

模拟执行的过程中会莫名其妙报错，所以直接加了一个try，最后打印出来的轨迹如下：

```
['0x2e1010', '0x2e3634', '0x2e3e1d', '0x2e389c', '0x2e3d9e', '0x2e3b8e', '0x2e37ae', '0x2e3f3a', '0x2e4ee5', '0x2e51ad', '0x2e45f9', '0x2e4e03', '0x2e3c8f', '0x2e4cf1', '0x2e4e96', '0x2e3d49', '0x2e3641', '0x2e4ca8', '0x2e49fd', '0x2e5109', '0x2e4e16', '0x2e382a', '0x2e48f1', '0x2e3ec2', '0x2e4567', '0x2e3a7e', '0x2e4ae0', '0x2e3718', '0x2e402f', '0x2e4ba1', '0x2e4263', '0x2e4441', '0x2e4af2', '0x2e42f7', '0x2e5163', '0x2e3dd1', '0x2e49b7', '0x2e4907', '0x2e4ddb', '0x2e2896', '0x2e2e08', '0x2e35a4', '0x2e2bd2', '0x2e32a2', '0x2e2cf2', '0x2e296d', '0x2e2eb6', '0x2e3391', '0x2e2f9b', '0x2e2ff8', '0x2e2b83', '0x2e3082', '0x2e2ab3', '0x2e333e', '0x2e2ee9', '0x2e2bc5', '0x2e3519', '0x2e3447', '0x2e31a1', '0x2e33fa', '0x2e2bba', '0x2e3623', '0x2e2b95', '0x2e2e99', '0x2e308d', '0x2e33a0', '0x2e3473', '0x2e35ac', '0x2e2b21', '0x2e2980', '0x2e341d', '0x2e31d4', '0x2e32ab', '0x2e30e2', '0x2e289c', '0x2e2acb', '0x2e30f4', '0x2e34f8', '0x2e3176', '0x2e2e5d', '0x2e2cfe', '0x2e2bfb', '0x2e2f15', '0x2e2c6e', '0x2e2ea5', '0x2e305d', '0x2e2f91', '0x2e3267', '0x2e3210', '0x2e324a', '0x2e330f', '0x2e32d9', '0x2e2e78', '0x2e2924', '0x2e34d5', '0x2e2c19', '0x2e3121', '0x2e2907', '0x2e2a75', '0x2e332e', '0x2e2dc9', '0x2e2edc', '0x2e353d', '0x2e2c2f', '0x2e2cd4', '0x2e28e4', '0x2e2b6c', '0x2e3481', '0x2e294b', '0x2e2b40', '0x2e2e83', '0x2e2f4d', '0x2e31f8', '0x2e4df6', '0x2e4177', '0x2e496d', '0x2e37a1', '0x2e3a3a', '0x2e4d76', '0x2e3e38', '0x2e45bc', '0x2e3f86', '0x2e3df5', '0x2e4242', '0x2e3aee', '0x2e5039', '0x2e3ff8', '0x2e4cb9', '0x2e48a1', '0x2e4135', '0x2e3d05', '0x2e4bd9', '0x2e3c0e', '0x2e5133', '0x2e42d7', '0x2e4bff', '0x2e39fe', '0x2e50a8', '0x2e4a2f', '0x2e4e6a', '0x2e43f6', '0x2e401d', '0x2e43a1', '0x2e4b95', '0x2e37d5', '0x2e404d', '0x2e37c6', '0x2e46b3', '0x2e5120', '0x2e5013', '0x2e5075', '0x2e4673', '0x2e45e1', '0x2e3ba2', '0x2e4802', '0x2e481c', '0x2e38d6', '0x2e4f11', '0x2e4494', '0x2e41f1', '0x2e3853', '0x2e504d', '0x2e4529', '0x2e50df', '0x2e3671', '0x2e3968', '0x2e3741', '0x2e4074', '0x2e368e', '0x2e4ffb', '0x2e4c86', '0x2e491f', '0x2e432b', '0x2e3e8c', '0x2e3f97', '0x2e38e5', '0x2e44bc', '0x2e444e', '0x2e3a48', '0x2e39c9', '0x2e46d2', '0x2e3982', '0x2e3eed', '0x2e4682', '0x2e3d7c', '0x2e3eb6', '0x2e3c25', '0x2e4390', '0x2e462c', '0x2e4957', '0x2e4a0c', '0x2e486e', '0x2e493b', '0x2e4479', '0x2e4760', '0x2e4ed5', '0x2e4eb6', '0x2e4d52', '0x2e39a8', '0x2e41bb', '0x2e4e48', '0x2e39b4', '0x2e513e', '0x2e41a4', '0x2e473a', '0x2e4abe', '0x2e47d8', '0x2e4650', '0x2e51b7', '0x2e4367', '0x2e3b75', '0x2e3c63', '0x2e4542', '0x2e487f', '0x2e4b79', '0x2e4ccc', '0x2e3cc8', '0x2e4d28', '0x2e36f1', '0x2e4a7b', '0x2e3cd3', '0x2e3e98', '0x2e4f28', '0x2e3847', '0x2e38ac', '0x2e365c', '0x2e454f', '0x2e3944', '0x2e4105', '0x2e4506', '0x2e4bb6', '0x2e3893', '0x2e4c71', '0x2e3839', '0x2e4f3b', '0x2e3bca', '0x2e3795', '0x2e3b16', '0x2e40c9', '0x2e3d3c', '0x2e3afe', '0x2e5230', '0x2e419c']
```

这么多代码块一个个去手动分析不太现实，于是再加一个hook来hook输入和key的访问操作，来帮助我们找到了访问了输入和key的指令所在的代码块，加上：

```
mu.hook_add(UC_HOOK_MEM_READ, self.hook_mem_read)
```

输出：

```
Read input[8] at 0x2e326d
```

通过内存访问hook我们得到了几个很重要的信息：

- 读取输入的地址

- 读取key的地址

- 输入可能恒是2字节一组进行加密后比较

- 当前比对失败后程序不会继续比对剩下的部分

第三、四个特点是一个伏笔，之后我们会利用这个性质对flag进行爆破。

接下来看到访问了输入的几段代码，这些代码的作用是将第一个字节读入到al，第二个字节读入到bl：\
!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)\
!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)\
!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)\
...\
从这里开始，顺着我们之前打印出的轨迹往后再分析一会还能发现这样的代码：\
!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)\
!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)\
说明程序确实是将16进制两字节的输入转换成了对应的16进制数。\
再来看到访问了key的代码块：\
!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)\
我们再修改一下trace函数，通过capstone反汇编引擎找到执行到的cmp指令和test指令的地址：

```
def trace(self, mu, address, size, data):
```

输出：

```
Instruction cmp at 0x2e3ca1
```

可以看到在读取key之后执行的cmp指令只有一个，位于2E38E7这个地址，代码如下，大致可以确定是flag加密后比较的代码，比对成功的话不会执行jnz跳转：\
!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)\
所以我们可以通过记录程序第几次执行到了2E38EF这个地址，来判断比较成功比对了几个字节，通过这种方法来爆破flag。

4

# **爆破flag**

再改一下trace函数：

```
def trace(self, mu, address, size, data):
```

爆破flag的函数get_flag：

```
def get_flag(flag, except_hit):
```

这里选择的字符集为b'1234567890abcdefABCDEF'，包括了小写的字母，比赛的时候我是根据traces手动分析加密流程，被大小写坑了几个小时。爆破结果如下：

```
SangFor{A7000000000000000000000000000000}
```

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

# 

5

# **完整exp**

```
from ctypes import addressof
```

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**看雪ID：34r7hm4n**

https://bbs.pediy.com/user-home-910514.htm

\*本文由看雪论坛 34r7hm4n 原创，转载请注明来自看雪社区

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

官网：https://www.bagevent.com/event/6334937

[!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)](https://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458387399&idx=2&sn=38495add2a3a3677b2c436581c07e432&scene=21#wechat_redirect)

**#** **往期推荐**

1.[从两道0解题看Linux内核堆上msg_msg对象扩展利用](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458394993&idx=2&sn=2083b054decba632e3a17f751e5541ee&chksm=b18f11fb86f898ed7d39d50722b69e3b8505a603b02ccb315b7b6ac18cd145e00e4b0aa5813b&scene=21#wechat_redirect)

2. [Android APP漏洞之战——Broadcast Recevier漏洞详解](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458394855&idx=2&sn=89809cfc0b8ae785cf5b0b6dcbecb871&chksm=b18f106d86f8997b07492e00f9d59db199eac9527ddc9cd214cb0259e7aa1c5c812c300aa67d&scene=21#wechat_redirect)

3.[16位实模式切换32位保护模式过程详解](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458392978&idx=2&sn=10921a9bf6c258c747315bcc91d4e18c&chksm=b18f291886f8a00e68389072d7e64fd9c93ee9daacbf236a9cb8028134519fad7740ce0f4756&scene=21#wechat_redirect)

4. [高Glibc版本下的堆骚操作解析](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458392560&idx=1&sn=73533c60d701ad5816545520bc889135&chksm=b18f277a86f8ae6ce880fd532b70bbbb70180859299f7476bd1d3033a43ea7e0b3fb6709c460&scene=21#wechat_redirect)

5.[新人PWN堆Heap总结off-by-null专场](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458392540&idx=1&sn=75525c0ee1d799edbf118b4e19b81e3a&chksm=b18f275686f8ae40bacfec040a64b645227ce1f5554cdc5e95c337b63573d1ec67a57bebec01&scene=21#wechat_redirect)

6. [CVE-2012-3569 VMware OVF Tool格式化字符串漏洞分析](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458392532&idx=1&sn=009b3954d88c8732df14e2d442e6cf04&chksm=b18f275e86f8ae48ec217fc9b28c98cdcfa94822de52fd359c75d81de835ce23605f46fcdf79&scene=21#wechat_redirect)

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

公众号ID：ikanxue

官方微博：看雪安全

商务合作：wsc@kanxue.com

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**球分享**

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**球点赞**

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**球在看**

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

点击“阅读原文”，了解更多！

阅读原文

阅读 1067

​

写留言

**留言 2**

- 杜超

  2021年9月30日

  赞

  看雪文章质量都特别好，看不过来了![[流泪]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![[流泪]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

  看雪学苑

  作者2021年9月30日

  赞

  慢慢看![[机智]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

已无更多数据

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/1UG7KPNHN8EGLfh77kFmnicd9WOic2ibvhCibFdB4bL4srJCgo2wnvdoXLxpIvAkfCmmcptXZB0qKWMoIP8iaibYN2FA/300?wx_fmt=png&wxfrom=18)

看雪学苑

412

2

写留言

**留言 2**

- 杜超

  2021年9月30日

  赞

  看雪文章质量都特别好，看不过来了![[流泪]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![[流泪]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

  看雪学苑

  作者2021年9月30日

  赞

  慢慢看![[机智]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

已无更多数据
