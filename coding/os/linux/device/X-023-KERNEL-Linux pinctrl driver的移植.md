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

作者：[wowo](http://www.wowotech.net/author/2 "runangaozhong@163.com") 发布于：2017-7-14 21:58 分类：[X Project](http://www.wowotech.net/sort/x_project)

## 1. 前言

本文是“[linux内核中的GPIO系统之（4）：pinctrl驱动的理解和总结](http://www.wowotech.net/gpio_subsystem/pinctrl-driver-summary.html)”的一个实例，结合”[X Project](http://www.wowotech.net/sort/x_project)”的开发过程，介绍pinctrl driver的移植步骤，进而加深对pinctrl framework的理解。

注1：本文后续的描述，kernel基于本站“[X Project](http://www.wowotech.net/sort/x_project)”所使用的kernel版本\[4\]，硬件基于 ”[X Project](http://www.wowotech.net/sort/x_project)”所使用的“Bubbugum-96”平台。

## 2. pinctrl driver的移植步骤

#### 2.1 添加pinctrl driver在驱动模型中的框架

在linux kernel中，pin controller也是一个普通的platform device，因此需要基于platform设备的框架（可参考\[1\]）添加自身的驱动模型。对于这些单纯的设备，可以直接用一些标准化的接口来完成，如下（都是轻车熟路的东西，不再解释了）：

---->drivers/pinctrl/pinctrl-owl.c

|   |
|---|
|/*============================================================================  <br>*                                                               platform driver  <br>*==========================================================================*/  <br>static const struct of_device_id owl_pinctrl_of_match\[\] = {  <br>        {  <br>                .compatible = "actions,s900-pinctrl",  <br>        },  <br>        {  <br>         },  <br>};  <br>MODULE_DEVICE_TABLE(of, owl_pinctrl_of_match);  <br>  <br>static int \_\_init owl_pinctrl_probe(struct platform_device \*pdev)  <br>{   <br>        return 0;            <br>}  <br>  <br>static void \_\_exit owl_pinctrl_remove(struct platform_device \*pdev)  <br>{  <br>}  <br>  <br>static struct platform_driver owl_pinctrl_platform_driver = {  <br>        .probe          = owl_pinctrl_probe,  <br>        .remove         = owl_pinctrl_remove,  <br>        .driver         = {  <br>                .name   = "owl_pinctrl",  <br>                .of_match_table = owl_pinctrl_of_match,  <br>        },  <br>};  <br>module_platform_driver(owl_pinctrl_platform_driver);  <br>  <br>MODULE_ALIAS("platform driver: owl_pinctrl");  <br>MODULE_DESCRIPTION("pinctrl driver for owl seria soc(s900, etc.)");  <br>MODULE_AUTHOR("wowo<wowo@wowotech.net>");  <br>MODULE_LICENSE("GPL v2");|

（另外，需要编辑drivers/pinctrl中的Kconfig和Makefile文件，加入我们新的pinctrl driver，并更改”[X Project](http://www.wowotech.net/sort/x_project)”的kernel配置项，加入该driver的编译，具体步骤略，代码可参考如下的patch：[https://github.com/wowotechX/linux/commit/45ca29b3680db7b6d62735439a0ef93ab9797537](https://github.com/wowotechX/linux/commit/45ca29b3680db7b6d62735439a0ef93ab9797537 "https://github.com/wowotechX/linux/commit/45ca29b3680db7b6d62735439a0ef93ab9797537"),

[https://github.com/wowotechX/linux/commit/749f114564765025d907618d7306bc668c0a0a93](https://github.com/wowotechX/linux/commit/749f114564765025d907618d7306bc668c0a0a93 "https://github.com/wowotechX/linux/commit/749f114564765025d907618d7306bc668c0a0a93")）。

#### 2.2 添加pinctrl driver自身的框架

参考“Documentation/pinctrl.txt\[2\]"中的示例，创建一个struct pinctrl_desc变量，并在probe中调用pinctrl_register注册到kernel中。

另外，为了在同一个driver中支持多个soc，可以将struct pinctrl_desc变量的指针保存在每个soc的match table中，并在probe中借助of_device_get_match_data将其获取出来（这是device tree时代的惯用伎俩，大家可以多使用），如下黄色背景所示：

|   |
|---|
|static const struct of_device_id owl_pinctrl_of_match\[\] = {  <br>        {  <br>                 .compatible = "actions,s900-pinctrl",  <br>                .data = &s900_desc,  <br>        },  <br>        {  <br>        },  <br>};<br><br>...<br><br>static int \_\_init owl_pinctrl_probe(struct platform_device \*pdev)  <br>{  <br>        ...<br><br>         desc = of_device_get_match_data(dev);  <br>        if (desc == NULL) {  <br>                  dev_err(dev, "device get match data failed\\n");  <br>                 return -ENODEV;  <br>        }<br><br>         pctl = pinctrl_register(desc, dev, NULL);  <br>        ...  <br>}|

可以直接从Documentation/pinctrl.txt中copy过来，将foo换成自己平台的代号（如这里的s900），然后在owl_pinctrl_probe中调用pinctrl_register注册即可。

以上代码可参考如下的patch：[https://github.com/wowotechX/linux/commit/9ce89fe7e5d68f2d84dab39af0665acfc3b62751](https://github.com/wowotechX/linux/commit/9ce89fe7e5d68f2d84dab39af0665acfc3b62751 "https://github.com/wowotechX/linux/commit/9ce89fe7e5d68f2d84dab39af0665acfc3b62751")

#### 2.3 定义管脚（pin）

driver框架确定了之后，就简单了，根据\[3\]中的介绍，一个一个的添加各种软件抽象就行了。例如可以根据自己平台的实际情况，定义管脚。

注2：这里以“Bubbugum-96”平台为例，硬件有关的说明可参考之前的文章。另外，为了说明pinctrl driver的编写过程，这里简化一下，暂时只关心UART5有关的pin，其它的后面用的时候再加就行了。

根据原理图，uart5有关的pin脚包括（好东西，足够我们把pinctrl的原理展示完了）：

> GPIOA25/UART5_RX/SENS0_VSYNC/PWM2（A7）\
> GPIOA27/UART5_TX/SENS0_HSYNC/PWM2（D8）\
> GPIOA28/UART2_RX/UART5_RX/SD0_D0（C5）\
> GPIOA29/UART2_TX/UART5_TX/SD0_D1（B6）\
> GPIOF6/UART5_RX/UART3_RTSB（G2）\
> GPIOF7/UART5_TX/UART3_CTSB（G1）

名字，好办，就用A7、D8之类的就行了，编号怎么办呢？

先随便定吧（后面可能会根据寄存器的特性调整），直接用字母（转成数字，A是0，B是1，等等）乘以10加数字（数字好像是从1开始，那就数字减一），即：

> A7 = （1 – 1）\* 10 + （7 – 1）= 6\
> D8 =  (4 – 1) * 10 + (8 – 1) = 37\
> C5 = (3 – 1) * 10 + (5 - 1) = 24\
> B6 = (2 – 1) * 10 + (6 – 1) = 15\
> G2 = (7 – 1) * 10 + (2 – 1) = 61\
> G1 = (7 – 1) * 10 + (1 – 1) =60

最终定义出来的管脚如下：

|   |
|---|
|static const struct pinctrl_pin_desc s900_pins\[\] = {  <br>        PINCTRL_PIN(6, "A7"),  <br>        PINCTRL_PIN(15, "B6"),  <br>         PINCTRL_PIN(24, "C5"),  <br>        PINCTRL_PIN(37, "D8"),  <br>         PINCTRL_PIN(60, "G1"),  <br>        PINCTRL_PIN(61, "G2"),  <br>};|

#### 2.4 定义pin group

暂时不考虑GPIO，上面6个pin可以分解出如下groups（麻雀虽小五脏俱全啊！不过简单起见，这里只关注UART5有关的group，因为其它group可能会涉及更多的pin）：

> uart5_0_group，包括A7和D8两个pin，由bubblegum96的spec\[4\]可知，当MFP_CTL1 寄存器（0xE01B0044）的bit28:26和bit25:23分别为001（UART5_RX）和100（UART5_TX）时，该group生效；
>
> uart5_1_group，包括C5和B6两个pin，由bubblegum96的spec\[4\]可知，当MFP_CTL2 寄存器（0xE01B0048）的bit19:17和bit16:14分别为101（UART5_RX）和101（UART5_TX）时，该group生效；
>
> uart5_2_group，包括G2和G1两个pin，由bubblegum96的spec\[4\]可知，当MFP_CTL2 寄存器（0xE01B0048）的bit21和bit20分别为1（UART5_RX）和1（UART5_TX）时，该group生效。

最后参考\[2\]中的例子，定义并注册struct pinctrl_ops变量，具体可参考如下patch：[https://github.com/wowotechX/linux/commit/478f3fcf2fc950daa6fb85eb302f943a427f0878](https://github.com/wowotechX/linux/commit/478f3fcf2fc950daa6fb85eb302f943a427f0878 "https://github.com/wowotechX/linux/commit/478f3fcf2fc950daa6fb85eb302f943a427f0878")

#### 2.5 Pin configuration的设计

参考\[4\]中“5.7.3.24 PAD_PULLCTL0 ”~“5.7.3.35 PAD_SR2 ”章节有关pin configuration的描述，发现s900的pinconf有点特殊，是以特定的功能（function，例如UART0_RX）为单位进行控制的，也就是说，对某一个功能来说，可能映射到不同的管脚上，但对它的配置，都是由相同的寄存器控制的。

基于这样的硬件框架，怎么去抽象pinconf的operations，值得思考。不过由于本文的例子（uart5）没有可配置的选项，所以我们就先略过这里。后面有机会再补回来吧（不影响我们对pinctrl的整体理解）。

#### 2.6 Pin multiplexing的设计

同理，参考\[2\]中的示例，首先抽象出Pin multiplexing中的functions，本文的例子中，暂时只有一个：uart5，定义如下：

|   |
|---|
|/\* drivers/pinctrl/pinctrl-owl.c \*/<br><br>static const char * const uart5_groups\[\] = {  <br>        "uart5_0_grp", "uart5_1_grp", "uart5_2_grp"  <br>};<br><br>static const struct owl_pmx_func s900_functions\[\] = {  <br>        {  <br>                .name = "uart5",  <br>                .groups = uart5_groups,  <br>                 .num_groups = ARRAY_SIZE(uart5_groups),  <br>         },  <br>};|

> 该function可以映射到三个不同的group中："uart5_0_grp", "uart5_1_grp", "uart5_2_grp"（它们的名字要和2.4中定义的groups name相同）。

然后，定义一个struct pinmux_ops变量，并实现相应的回调函数，如下：

|   |
|---|
|static struct pinmux_ops s900_pmxops = {  <br>         .get_functions_count = s900_get_functions_count,  <br>        .get_function_name = s900_get_fname,  <br>        .get_function_groups = s900_get_groups,  <br>        .set_mux = s900_set_mux,  <br>        .strict = true,  <br>};|

上面get_functions_count 、get_function_name以及get_function_groups三个函数比较简单，这里重点提一下set_mux，它的原型如下：

> int (\*set_mux) (struct pinctrl_dev \*pctldev, unsigned func_selector,\
> unsigned group_selector);

有两个参数：func_selector，用于指定function的number（就是s900_functions中的数组index）；group_selector，用于指定group的number（就是2.4中介绍的s900_groups的数组index）。pinctrl driver拿到这两个参数之后，怎么去配置寄存器使某个group的某个function生效呢？要根据具体的pin controller的硬件情况来定。以本文作为示例的s900为例，该SoC的Pin multiplexing的设置，并不需要function和group的一一对应，以uart5 function的group0为例，参考2.4中的介绍：

> 只要把MFP_CTL1 寄存器（0xE01B0044）的bit28:26和bit25:23分别设置为001（UART5_RX）和100（UART5_TX），就可以使uart5的该group生效。

也就是说，s900 SoC可以单独以group为单位进行pinmux配置。那么我们在代码中可以把相应的寄存器配置信息和groups的定义绑定，在pinctrl core调用.set_mux回调函数的时候，忽略 func_selector，直接通过group_selector获取相应的寄存器信息后，配置寄存器即可，如下：

> +struct owl_pmx_reginfo {\
> +       const unsigned \*offset;\
> +       const unsigned \*mask;\
> +       const unsigned \*val;\
> +\
> +       unsigned num_entries;\
> +};\
> +\
> struct owl_pinctrl_group {\
> const char \*name;\
> const unsigned \*pins;\
> unsigned num_pins;\
> +\
> +       const struct owl_pmx_reginfo \*reginfo;\
> +};
>
> ......
>
> static const struct owl_pinctrl_group s900_groups\[\] = {\
> {\
> .name = "uart5_0_grp",\
> .pins = s900_uart5_0_pins,\
> .num_pins = ARRAY_SIZE(s900_uart5_0_pins),\
> +               .reginfo = &s900_uart5_0_reginfo,\
> },\
> {\
> .name = "uart5_1_grp",\
> .pins = s900_uart5_1_pins,\
> .num_pins = ARRAY_SIZE(s900_uart5_1_pins),\
> +               .reginfo = &s900_uart5_1_reginfo,\
> },\
> {\
> .name = "uart5_2_grp",\
> .pins = s900_uart5_2_pins,\
> .num_pins = ARRAY_SIZE(s900_uart5_2_pins),\
> +               .reginfo = &s900_uart5_2_reginfo,\
> },\
> };
>
> ...
>
> +static int s900_set_mux(struct pinctrl_dev \*pctldev, unsigned selector,\
> +                       unsigned group)\
> +{\
> +       int i;\
> +       uint32_t val;\
> +\
> +       struct owl_pinctrl_priv \*priv = pctldev->driver_data;\
> +\
> +       const struct owl_pmx_reginfo \*reginfo;\
> +\
> +       reginfo = s900_groups\[gourp\].reginfo;\
> +\
> +       for (i = 0; i \< reginfo->num_entries; i++) {\
> +               val = readl(priv->mem_base + reginfo->offset\[i\]);\
> +               val &= ~reginfo->mask\[i\];\
> +               val |= reginfo->val\[i\];\
> +               writel(val, priv->mem_base + reginfo->offset\[i\]);\
> +       }\
> +\
> +       return 0;\
> +}

以上代码比较直接、简单，又和具体的硬件平台有关，我就不详细解释了，大家在编写自己平台的pinctrl driver时，可以具体情况具体对待。

以上可参考如下patch：[https://github.com/wowotechX/linux/commit/52d6b8337a31267fd9b158d5f1f9d9590a16f92d](https://github.com/wowotechX/linux/commit/52d6b8337a31267fd9b158d5f1f9d9590a16f92d "https://github.com/wowotechX/linux/commit/52d6b8337a31267fd9b158d5f1f9d9590a16f92d")。

#### 2.7 dt_node_to_map的设计

上面的pinmux设计完成之后，我们需要提供一种机制，让consumer（这里为serial driver）可以方便的使用。这里介绍利用device tree的方法----步骤有三：

1）定义pinctrl device的dts node，并包含一个包含uart5 function的子节点（节点的格式可以自行决定，因为是自己解析），例如：

|   |
|---|
|--- a/arch/arm64/boot/dts/actions/s900-bubblegum.dts  <br>+++ b/arch/arm64/boot/dts/actions/s900-bubblegum.dts  <br>@@ -39,9 +39,25 @@  <br>                 clock-frequency = \<24000000>;  <br>        };<br><br>+       pinctrl@0xe01b0000 {  <br>+               compatible = "actions,s900-pinctrl";  <br>+               reg = \<0 0="" 0xe01b0000="" 0x550="">;  <br>+  <br>+               uart5_state_default: uart5_default {  <br>+                       pinmux {  <br>+                               function = "uart5";  <br>+                               group = "uart5_0_grp";  <br>+                       };  <br>+                       /\* others, TODO \*/  <br>+               };  <br>+       };|

> 这里子节点的名字为uart5_default（别名为uart5_state_default，以便consumer device的dts node引用），代表了一个pin state（相关概念可参考\[3\]中的介绍）；
>
> 简单期间，该state里面包含一个entry----pinmux，该entry有两个关键字，function和group，都是字符串类型的。

2）在pinctrl driver中，实现.dt_node_to_map和.dt_free_map回调函数，其中.dt_node_to_map用于将dts node中的信息转换为pin map，示例如下：

|   |
|---|
|+static int s900_dt_node_to_map(struct pinctrl_dev \*pctldev,  <br>+                              struct device_node \*np_config,  <br>+                              struct pinctrl_map \*\*map,  <br>+                              unsigned \*num_maps)  <br>+{  <br>+       int ret, child_cnt;  <br>+  <br>+       const char \*function;  <br>+       const char \*group;  <br>+  <br>+       struct device \*dev = pctldev->dev;  <br>+       struct device_node \*np;  <br>+  <br>+       dev_dbg(dev, "%s\\n", __func__);  <br>+  <br>+       \*map = NULL;  <br>+       \*num_maps = 0;  <br>+  <br>+       child_cnt = of_get_child_count(np_config);  <br>+       dev_dbg(dev, "child_cnt %d\\n", child_cnt);  <br>+  <br>+       if (child_cnt == 0)  <br>+               return 0;  <br>+  <br>+       \*map = kzalloc(sizeof(struct pinctrl_map) * child_cnt, GFP_KERNEL);  <br>+       if (\*map == NULL) {  <br>+               dev_dbg(dev, "malloc failed\\n");  <br>+               return -ENOMEM;  <br>+       }  <br>+  <br>+       for_each_child_of_node(np_config, np) {  <br>+               ret = of_property_read_string(np, "function", &function);  <br>+               if (ret != 0)  <br>+                       continue;  <br>+  <br>+               ret = of_property_read_string(np, "group", &group);  <br>+               if (ret != 0)  <br>+                       continue;  <br>+  <br>+               dev_dbg(dev, "got a pinmux entry: %s-%s\\n", function, group);  <br>+  <br>+               (\*map)\[\*num_maps\].type = PIN_MAP_TYPE_MUX_GROUP;  <br>+               (\*map)\[\*num_maps\].data.mux.function = function;  <br>+               (\*map)\[\*num_maps\].data.mux.group = group;  <br>+               (\*num_maps)++;  <br>+       }  <br>+  <br>+       return 0;  <br>+}|

> 这个例子比较简单：
>
> 参数主要包括：np_config，dts节点指针，对应上面的“uart5_state_default”；map，pinctrl map指针的指针，需要pinctrl driver申请空间；num_maps，map个数的指针，driver可以修改该指针的值，告诉上层软件最终的map个数。
>
> 得到dts node指针之后，我们可以遍历该node下的所有子节点，并检查对应的子节点下是否有有效的function和group字符串，如果有，则代表找到了一个有效的entry，将相应的信息（function name和group name）保存在maps数组中的一个entry即可。

3）在对应的consumer的dts node中，引用上述的pinctrl state，如下：

|   |
|---|
|serial5: serial@e012a000 {  <br>                compatible = "actions,s900-serial";  <br>                reg = \<0 0="" 0xe012a000="" 0x2000="">;  /\* UART5_BASE \*/  <br>                interrupts = ;  <br>+  <br>+               pinctrl-names = "default";  <br>+               pinctrl-0 = \<&uart5_state_default>;  <br>        };|

> 上面pinctrl-names指定为"defualt"，这样可以在设备probe的时候由kernel自行获取相应的pinctrl state并使之生效。
>
> 然后在pinctrl-0字段中填入pinctrl state的地址：&uart5_state_default。
>
> 最后serial driver在probe的时候，将会按照\[3\]中介绍过程（probe-->devm_pinctrl_get or pinctrl_get-->create_pinctrl-->pinctrl_dt_to_map-->dt_to_map_one_config-->pctlops->dt_node_to_map-->s900_dt_node_to_map），将该设备对应的pinctrl state保存下来，并通过pinctrl_lookup_state查找名称为"default“的state，调用pinctrl_select_state，使其生效。

以上代码可参考如下patch：[https://github.com/wowotechX/linux/commit/7c7b7b948b6f4eb25ef7926c89eb1199e07a7bc9](https://github.com/wowotechX/linux/commit/7c7b7b948b6f4eb25ef7926c89eb1199e07a7bc9 "https://github.com/wowotechX/linux/commit/7c7b7b948b6f4eb25ef7926c89eb1199e07a7bc9")。

## 3. 编译与调试

完成上面的移植工作之后，我们增加一些debug信息，并使能该driver的DEBUG宏，如下（具体可参考“[https://github.com/wowotechX/linux/commit/3acb915db41a4e87defa8b4ea412d92feff62c57](https://github.com/wowotechX/linux/commit/3acb915db41a4e87defa8b4ea412d92feff62c57 "https://github.com/wowotechX/linux/commit/3acb915db41a4e87defa8b4ea412d92feff62c57")”）：

|   |
|---|
|--- a/drivers/pinctrl/pinctrl-owl.c  <br>+++ b/drivers/pinctrl/pinctrl-owl.c  <br>@@ -13,6 +13,8 @@  <br>  * You should have received a copy of the GNU General Public License  <br>  * along with this program.  If not, see \<[http://www.gnu.org/licenses/](http://www.gnu.org/licenses/)>.  <br>  \*/  <br>+#define DEBUG  <br>+  <br>  #include  <br>  #include  <br>  #include  <br>@@ -143,17 +145,23 @@ static const struct owl_pinctrl_group s900_groups\[\] = {<br><br>static int s900_get_groups_count(struct pinctrl_dev \*pctldev)  <br>  {  <br>+       dev_dbg(pctldev->dev, "%s\\n", __func__);  <br>+  <br>         return ARRAY_SIZE(s900_groups);  <br>  }  <br>...|

然后编译、运行起来，得到如下的log：

> \[    0.080093\] DMA: preallocated 256 KiB pool for atomic allocations\
> \[    0.086499\] clocksource: Switched to clocksource arch_sys_counter\
> \[    0.092437\] Unpacking initramfs...\
> \[    0.177156\] Freeing initrd memory: 1044K (ffffffc07feaf000 - ffffffc07ffb4000)\
> \[    0.182031\] workingset: timestamp_bits=60 max_order=19 bucket_order=0\
> \[    0.188093\] owl_pinctrl e01b0000.pinctrl: owl_pinctrl_probe\
> \[    0.193593\] owl_pinctrl e01b0000.pinctrl: s900_get_functions_count\
> \[    0.199749\] owl_pinctrl e01b0000.pinctrl: s900_get_fname, selector 0\
> \[    0.206093\] owl_serial_init\
> \[    0.208874\] owl_pinctrl e01b0000.pinctrl: s900_dt_node_to_map\
> \[    0.214562\] owl_pinctrl e01b0000.pinctrl: child_cnt 1\
> \[    0.219593\] owl_pinctrl e01b0000.pinctrl: got a pinmux entry: uart5-uart5_0_grp\
> \[    0.226874\] owl_pinctrl e01b0000.pinctrl: s900_get_functions_count\
> \[    0.232999\] owl_pinctrl e01b0000.pinctrl: s900_get_fname, selector 0\
> \[    0.239343\] owl_pinctrl e01b0000.pinctrl: s900_get_groups, selector 0\
> \[    0.245749\] owl_pinctrl e01b0000.pinctrl: s900_get_groups_count\
> \[    0.251656\] owl_pinctrl e01b0000.pinctrl: s900_get_group_name, selector 0\
> \[    0.258406\] owl_pinctrl e01b0000.pinctrl: s900_get_group_pins, selector 0\
> \[    0.265156\] owl_pinctrl e01b0000.pinctrl: s900_set_mux, selector 0, group 0\
> \[    0.272093\] owl_pinctrl e01b0000.pinctrl:  offset 44, mask 1c000000, val 4000000\
> \[    0.279468\] owl_pinctrl e01b0000.pinctrl:  offset 44, mask 3800000, val 2000000\
> \[    0.286749\] owl_serial e012a000.serial: owl_serial_probe\
> \[    0.292062\] e012a000.serial: ttyS5 at MMIO 0xe012a000 (irq = 5, base_baud = 0) is a unknown\
> \[    0.300343\] console \[ttyS5\] enabled\
> \[    0.300343\] console \[ttyS5\] enabled

大功告成！！后续其它的功能，在需要的时候慢慢完善就行了~~

## 4. 总结

经过上面的步骤，我们开发了一个简单的pinctrl driver，并通过consumer的dts node将其绑定。这样consumer driver在probe的时候，会自动调用pinctrl subsystem提供的API，获取一个default state，并使其生效。

当然，本文所描述的pinctrl driver只算一个简单的demo（只有uart5有关的管脚），实际的项目开发过程中要比这复杂的多，但万变不离其宗，仅仅是代码量和编程技巧的差异而已。

## 5. 参考文档

\[1\] [Linux设备模型(8)\_platform设备](http://www.wowotech.net/device_model/platform_device.html)

\[2\] [Documentation/pinctrl.txt](https://github.com/wowotechX/linux/blob/x_integration/Documentation/pinctrl.txt)

\[3\] [linux内核中的GPIO系统之（4）：pinctrl驱动的理解和结](http://www.wowotech.net/gpio_subsystem/pinctrl-driver-summary.html)

\[4\] [https://github.com/96boards/documentation/blob/master/ConsumerEdition/Bubblegum-96/HardwareDocs/SoC_bubblegum96.pdf](https://github.com/96boards/documentation/blob/master/ConsumerEdition/Bubblegum-96/HardwareDocs/SoC_bubblegum96.pdf "https://github.com/96boards/documentation/blob/master/ConsumerEdition/Bubblegum-96/HardwareDocs/SoC_bubblegum96.pdf")

\[5\] [https://github.com/96boards/documentation/blob/master/ConsumerEdition/Bubblegum-96/HardwareDocs/Schematics_Bubblegum96.pdf](https://github.com/96boards/documentation/blob/master/ConsumerEdition/Bubblegum-96/HardwareDocs/Schematics_Bubblegum96.pdf "https://github.com/96boards/documentation/blob/master/ConsumerEdition/Bubblegum-96/HardwareDocs/Schematics_Bubblegum96.pdf")

原创文章，转发请注明出处。蜗窝科技，[www.wowotech.net](http://www.wowotech.net/x_project/kernel_pinctrl_driver_porting.html)。

标签: [Linux](http://www.wowotech.net/tag/Linux) [Kernel](http://www.wowotech.net/tag/Kernel) [driver](http://www.wowotech.net/tag/driver) [porting](http://www.wowotech.net/tag/porting) [pinctrl](http://www.wowotech.net/tag/pinctrl)

[![](http://www.wowotech.net/content/uploadfile/201605/ef3e1463542768.png)](http://www.wowotech.net/support_us.html)

« [Dynamic DMA mapping Guide](http://www.wowotech.net/memory_management/DMA-Mapping-api.html) | [蜗窝微信群问题整理](http://www.wowotech.net/tech_discuss/question_set_1.html)»

**发表评论：**

昵称

邮件地址 (选填)

个人主页 (选填)

![](http://www.wowotech.net/include/lib/checkcode.php)

- ### 站内搜索

  蜗窝站内  互联网

- ### 功能

  [留言板\
  ](http://www.wowotech.net/message_board.html)[评论列表\
  ](http://www.wowotech.net/?plugin=commentlist)[支持者列表\
  ](http://www.wowotech.net/support_list)

- ### 最新评论

  - ja\
    [@dream：我看完這段也有相同的想法，引用 @dream ...](http://www.wowotech.net/kernel_synchronization/spinlock.html#8922)
  - 元神高手\
    [围观首席power managerment专家](http://www.wowotech.net/pm_subsystem/device_driver_pm.html#8921)
  - 十七\
    [内核空间的映射在系统启动时就已经设定好，并且在所有进程的页表...](http://www.wowotech.net/process_management/context-switch-arch.html#8920)
  - lw\
    [sparse模型和disconti模型没看出来有什么本质区别...](http://www.wowotech.net/memory_management/memory_model.html#8919)
  - 肥饶\
    [一个没设置好就出错](http://www.wowotech.net/linux_kenrel/516.html#8918)
  - orange\
    [点赞点赞，对linuxer的文章总结到位](http://www.wowotech.net/device_model/dt-code-file-struct-parse.html#8917)

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

  - [小printf大作用（用日志打印的方式调试程序）](http://www.wowotech.net/soft/7.html)
  - [SLUB DEBUG原理](http://www.wowotech.net/memory_management/427.html)
  - [Fix-Mapped Addresses](http://www.wowotech.net/memory_management/fixmap.html)
  - [eMMC 原理 4 ：总线协议](http://www.wowotech.net/basic_tech/emmc_bus_protocol.html)
  - [Linux电源管理(13)\_Driver的电源管理](http://www.wowotech.net/pm_subsystem/driver_pm.html)

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
