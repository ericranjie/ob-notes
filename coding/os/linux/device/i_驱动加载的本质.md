
原创 布道师Peter 人人极客社区 _2021年11月25日 08:26_

内核启动的过程中会通过函数 **do_initcalls**，将按顺序从 \_\_initcall_start 开始，到 \_\_initcall_end 结束的 section 中以函数指针的形式取出这些编译到内核的驱动模块中初始化函数起始地址，来依次完成相应的初始化。这些初始化函数由 \_\_define_initcall(level,fn) 指示编译器在编译的时候，将这些初始化函数的起始地址值按照一定的顺序放在这个section中。

\_\_initcall_start 到 \_\_initcall_end 之间的 section，通过 vmlinux.lds 可以看到：

![[Pasted image 20241021183912.png]]

宏 INIT_CALLS 中定义的这些 section 中放了一系列的函数，这些函数是用 pure_initcall，core_initcall 等宏定义的。如下所示：

```cpp
`#define INIT_CALLS_LEVEL(level)      \     VMLINUX_SYMBOL(__initcall##level##_start) = .;  \     KEEP(*(.initcall##level##.init))   \     KEEP(*(.initcall##level##s.init))   \      #define INIT_CALLS       \     VMLINUX_SYMBOL(__initcall_start) = .;   \     KEEP(*(.initcallearly.init))    \     INIT_CALLS_LEVEL(0)     \     INIT_CALLS_LEVEL(1)     \     INIT_CALLS_LEVEL(2)     \     INIT_CALLS_LEVEL(3)     \     INIT_CALLS_LEVEL(4)     \     INIT_CALLS_LEVEL(5)     \     INIT_CALLS_LEVEL(rootfs)    \     INIT_CALLS_LEVEL(6)     \     INIT_CALLS_LEVEL(7)     \     VMLINUX_SYMBOL(__initcall_end) = .;   `

`#define __define_initcall(fn, id) \    static initcall_t __initcall_name(fn, id) __used \    __attribute__((__section__(".initcall" #id ".init"))) = fn;        #define pure_initcall(fn)  __define_initcall(fn, 0)   #define core_initcall(fn)  __define_initcall(fn, 1)   #define core_initcall_sync(fn)  __define_initcall(fn, 1s)   #define postcore_initcall(fn)  __define_initcall(fn, 2)   #define postcore_initcall_sync(fn) __define_initcall(fn, 2s)   #define arch_initcall(fn)  __define_initcall(fn, 3)   #define arch_initcall_sync(fn)  __define_initcall(fn, 3s)   #define subsys_initcall(fn)  __define_initcall(fn, 4)   #define subsys_initcall_sync(fn) __define_initcall(fn, 4s)   #define fs_initcall(fn)   __define_initcall(fn, 5)   #define fs_initcall_sync(fn)  __define_initcall(fn, 5s)   #define rootfs_initcall(fn)  __define_initcall(fn, rootfs)   #define device_initcall(fn)  __define_initcall(fn, 6)   #define device_initcall_sync(fn) __define_initcall(fn, 6s)   #define late_initcall(fn)  __define_initcall(fn, 7)   #define late_initcall_sync(fn)  __define_initcall(fn, 7s)   `
```

所以，pure_initcall 定义的 initcall 函数放在 .initcall.0.init，core_initcall 定义的 initcall 函数放在 .initcall.1.init，以此类推。

## do_initcalls

现在我们看下内核启动过程中，实现驱动加载的函数。

```cpp
static void __init do_initcalls(void)   {    int level;       for (level = 0; level < ARRAY_SIZE(initcall_levels) - 1; level++)     //依次调用不同等级的初始化函数     do_initcall_level(level);   }   

static void __init do_initcall_level(int level)   {    initcall_t *fn;       strcpy(initcall_command_line, saved_command_line);    //initcall_level_names 是个数组，里面存放了各个级别的初始化函数级数名    parse_args(initcall_level_names[level],        initcall_command_line, __start___param,        __stop___param - __start___param,        level, level,        NULL, &repair_env_string);       //执行各个初始化级别的函数    for (fn = initcall_levels[level]; fn < initcall_levels[level+1]; fn++)     //初始化同一级别中的函数     do_one_initcall(*fn);   }   
```

initcall_level_names 是个数组，里面存放了各个级别的初始化函数级数名。如下所示：

```cpp
static char *initcall_level_names[] __initdata = {    "early",    "core",    "postcore",    "arch",    "subsys",    "fs",    "device",    "late",   };   
```

即轮询执行各个级别的函数，然后通过 do_one_initcall 初始化同一级别中的函数。

## module_init

以 module_init 为例子，因为：

```cpp
#define module_init(x) __initcall(x);   #define __initcall(fn) device_initcall(fn);   #define device_initcall(fn)  __define_initcall(fn, 6);   
```

所以：

`module_init(gpu_init);   `

变为：

```cpp
#define __define_initcall(gpu_init, 6) \   static initcall_t __initcall_gpu_init6 __used \   __attribute__((__section__(".initcall.6.init"))) = gpu_init;   
```

通过查找内核映射表 System.map，可以看到 \_\_initcall_gpu_init6 函数指针:

![[Pasted image 20241021183940.png]]

---

**留言 11**

- Lei.w

  2021年11月30日

  赞

  请问ko的加载流程文章什么时候出来？

  人人极客社区

  作者2021年11月30日

  赞2

  明天

- Peter

  2021年11月25日

  赞1

  大家可以加我微信：rrjike，一起学习，共同进步，还可以进群认识更多朋友![[玫瑰]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![[玫瑰]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![[玫瑰]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

- 一字一圈

  2021年11月26日

  赞

  突然结束，还有嘛![[吃瓜]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

  人人极客社区

  作者2021年11月26日

  赞

  有续集![[呲牙]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

- JSZDM

  2021年11月25日

  赞

  戛然而止

  人人极客社区

  作者2021年11月25日

  赞

  有续集

- 爔

  2021年11月25日

  赞

  以为是文章写的是加载ko

  人人极客社区

  作者2021年11月25日

  赞

  回头我把ko加进去 整个完整版

- Pineapple

  2021年11月25日

  赞

  不是写外设寄存器吗？

  人人极客社区

  作者2021年11月25日

  赞

  什么意思

已无更多数据

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/9sNwsXcN68pL55XIyzTrCHZTbIUdTibQcuzuCaYeGTXNMyn6ACmicUrpoDC0xZSap46XJ59sKysPg9Rg379f32cA/300?wx_fmt=png&wxfrom=18)

人人极客社区

1524

11

写留言

**留言 11**

- Lei.w

  2021年11月30日

  赞

  请问ko的加载流程文章什么时候出来？

  人人极客社区

  作者2021年11月30日

  赞2

  明天

- Peter

  2021年11月25日

  赞1

  大家可以加我微信：rrjike，一起学习，共同进步，还可以进群认识更多朋友![[玫瑰]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![[玫瑰]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![[玫瑰]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

- 一字一圈

  2021年11月26日

  赞

  突然结束，还有嘛![[吃瓜]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

  人人极客社区

  作者2021年11月26日

  赞

  有续集![[呲牙]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

- JSZDM

  2021年11月25日

  赞

  戛然而止

  人人极客社区

  作者2021年11月25日

  赞

  有续集

- 爔

  2021年11月25日

  赞

  以为是文章写的是加载ko

  人人极客社区

  作者2021年11月25日

  赞

  回头我把ko加进去 整个完整版

- Pineapple

  2021年11月25日

  赞

  不是写外设寄存器吗？

  人人极客社区

  作者2021年11月25日

  赞

  什么意思

已无更多数据
