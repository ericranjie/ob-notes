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

作者：[wowo](http://www.wowotech.net/author/2 "runangaozhong@163.com") 发布于：2017-6-27 22:30 分类：[GPIO子系统](http://www.wowotech.net/sort/gpio_subsystem)

## 1. 前言

本站之前的三篇文章[1][2][3]介绍了pin controller（对应的pin controller subsystem）、gpio controller（对应的GPIO subsystem）有关的基本概念，包括pin multiplexing、pin configuration等等。本文将基于这些文章，单纯地从pin controller driver的角度（屏蔽掉pinctrl core的实现细节），理解pinctrl subsystem的设计思想，并掌握pinctrl驱动的移植和实现方法。

## 2. pin controller的概念和软件抽象

相信每一个嵌入式从业人员，都知道“pin（管脚）”是什么东西（就不赘述了）。由于SoC系统越来越复杂、集成度越来越高，SoC中pin的数量也越来越多、功能也越来越复杂，这就对如何管理、使用这些pins提出了挑战。因此，用于管理这些pins的硬件模块（pin controller）就出现了。相应地，linux kernel也出现了对应的驱动（pin controller driver）。

Kernel pinctrl core使用struct pinctrl_desc抽象一个pin controller，该结构的定义如下（先贴在这里，后面会围绕这个抽象一步步展开）：

|   |
|---|
|\|   \|<br>\|---\|<br>\|/* include/linux/pinctrl/pinctrl.h */  <br>struct pinctrl_desc {  <br>        const char *name;  <br>        const struct pinctrl_pin_desc *pins;  <br>        unsigned int npins;  <br>        const struct pinctrl_ops *pctlops;  <br>        const struct pinmux_ops *pmxops;  <br>        const struct pinconf_ops *confops;  <br>        struct module *owner;  <br>#ifdef CONFIG_GENERIC_PINCONF  <br>        unsigned int num_custom_params;  <br>        const struct pinconf_generic_params *custom_params;  <br>        const struct pin_config_item *custom_conf_items;  <br>#endif  <br>};\||

注1：本文后续的描述基于本站“[X Project](http://www.wowotech.net/sort/x_project)”所使用的kernel版本[4]。  
注2：本文很多的表述（特别是例子），都是引用kernel的document[5]（写的很好，可以耐心看看）。

#### 2.1 Pin

kernel的pin controller子系统要想管理好系统的pin资源，第一个要搞明白的问题就是：系统中到底有多少个pin？用软件语言来表述就是：要把系统中所有的pin描述出来，并建立索引。这由上面struct pinctrl_desc结构中pins和npins来完成。

对pinctrl core来说，它只关心系统中有多少个pin，并使用自然数为这些pin编号，后续的操作，都是以这些编号为操作对象。至于编号怎样和具体的pin对应上，完全是pinctrl driver自己的事情。

因此，pinctrl driver需要根据实际情况，将系统中所有的pin组织成一个struct pinctrl_pin_desc类型的数组，该类型的定义为：

|   |
|---|
|/**  <br>* struct pinctrl_pin_desc - boards/machines provide information on their  <br>  * pins, pads or other muxable units in this struct  <br>  * @number: unique pin number from the global pin number space  <br>  * @name: a name for this pin  <br>  * @drv_data: driver-defined per-pin data. pinctrl core does not touch this  <br>  */  <br>struct pinctrl_pin_desc {  <br>        unsigned number;  <br>        const char *name;  <br>        void *drv_data;  <br>};|

number和name完全由driver自己决定，不过要遵循有利于代码编写、有利于理解等原则。另外，为了便于driver的编写，可以在drv_data中保存driver的私有数据结构（可以包含相关的寄存器偏移等信息）。

注3：[5]中有个例子，大家可以参考理解。

#### 2.2 Pin groups

在SoC系统中，有时需要将很多pin组合在一起，以实现特定的功能，例如SPI接口、I2C接口等。因此pin controller需要以group为单位，访问、控制多个pin，这就是pin groups。相应地，pin controller subsystem需要提供一些机制，来获取系统中到底有多少groups、每个groups包含哪些pins、等等。

因此，pinctrl core在struct pinctrl_ops中抽象出三个回调函数，用来获取pin groups相关信息，如下：

|   |
|---|
|struct pinctrl_ops {  <br>        int (*get_groups_count) (struct pinctrl_dev *pctldev);  <br>        const char *(*get_group_name) (struct pinctrl_dev *pctldev,  <br>                                        unsigned selector);  <br>        int (*get_group_pins) (struct pinctrl_dev *pctldev,  <br>                               unsigned selector,  <br>                               const unsigned **pins,  <br>                               unsigned *num_pins);  <br>        void (*pin_dbg_show) (struct pinctrl_dev *pctldev, struct seq_file *s,  <br>                          unsigned offset);  <br>        int (*dt_node_to_map) (struct pinctrl_dev *pctldev,  <br>                               struct device_node *np_config,  <br>                               struct pinctrl_map **map, unsigned *num_maps);  <br>        void (*dt_free_map) (struct pinctrl_dev *pctldev,  <br>                             struct pinctrl_map *map, unsigned num_maps);  <br>};|

> get_groups_count，获取系统中pin groups的个数，后续的操作，将以相应的索引为单位（类似数组的下标，个数为数组的大小）。
> 
> get_group_name，获取指定group（由索引selector指定）的名称。
> 
> get_group_pins，获取指定group的所有pins（由索引selector指定），结果保存在pins（指针数组）和num_pins（指针）中。
> 
> 注4：dt_node_to_map用于将device tree中的pin state信息转换为pin map，具体可参考后面第3章、第4章的介绍。

当然，最终的group信息是由pinctrl driver提供的，至于driver怎么组织这些group，那是driver自己的事情了，[4]中有一个例子，大家可以参考（在编写pinctrl driver的时候直接copy然后rename即可）。  

#### 2.3. Pin configuration(对象是pin或者pin group）

2.1和2.2中介绍了pinctrl subsystem中的操作对象（pin or pin group）以及抽象方法。嵌入式系统的工程师都知道，SoC中的管脚有些属性可以配置，例如上拉、下拉、高阻、驱动能力等。pinctrl subsystem使用pin configuration来封装这些功能，具体体现在struct pinconf_ops数据结构中，如下：

|   |
|---|
|struct pinconf_ops {  <br>#ifdef CONFIG_GENERIC_PINCONF  <br>         bool is_generic;  <br>#endif  <br>        int (*pin_config_get) (struct pinctrl_dev *pctldev,  <br>                               unsigned pin,  <br>                               unsigned long *config);  <br>        int (*pin_config_set) (struct pinctrl_dev *pctldev,  <br>                               unsigned pin,  <br>                                unsigned long *configs,  <br>                                unsigned num_configs);  <br>        int (*pin_config_group_get) (struct pinctrl_dev *pctldev,  <br>                                      unsigned selector,  <br>                                      unsigned long *config);  <br>        int (*pin_config_group_set) (struct pinctrl_dev *pctldev,  <br>                                      unsigned selector,  <br>                                      unsigned long *configs,  <br>                                     unsigned num_configs);  <br>        int (*pin_config_dbg_parse_modify) (struct pinctrl_dev *pctldev,  <br>                                            const char *arg,  <br>                                            unsigned long *config);  <br>        void (*pin_config_dbg_show) (struct pinctrl_dev *pctldev,  <br>                                      struct seq_file *s,  <br>                                      unsigned offset);  <br>        void (*pin_config_group_dbg_show) (struct pinctrl_dev *pctldev,  <br>                                            struct seq_file *s,  <br>                                            unsigned selector);  <br>        void (*pin_config_config_dbg_show) (struct pinctrl_dev *pctldev,  <br>                                             struct seq_file *s,  <br>                                             unsigned long config);  <br>};|

> pin_config_get，获取指定pin（管脚的编号，由2.1中pin的注册信息获得）当前配置，保存在config指针中（配置的具体含义，只有pinctrl driver自己知道，下同）。
> 
> pin_config_set，设置指定pin的配置（可以同时配置多个config，具体意义要由相应pinctrl driver解释）。
> 
> pin_config_group_get、pin_config_group_set，获取或者设置指定pin group的配置项。
> 
> 剩下的是一些debug用的api，不再说明（用得着的时候，再研究也不迟）。

关于pinconf，有一点一定要想明白：

> **kernel pinctrl subsystem并不关心configuration的具体内容是什么，它只提供pin configuration get/set的通用机制，至于get到的东西，以及set的东西，到底是什么，是pinctrl driver自己的事情。后面结合pin map和pin state的介绍（3.2小节），就能更好地理解这种设计了。**

#### 2.4. Pin multiplexing(对象是pin或者pin group)

为了照顾不同类型的产品、不同的应用场景，SoC中的很多管脚可以配置为不同的功能，例如A2和B5两个管脚，既可以当作普通的GPIO使用，又可以配置为I2C0的的SCL和SDA，也可以配置为UART5的TX和RX，这称作管脚的复用（pin multiplexing，简称为pinmux）。kernel pinctrl subsystem使用struct pinmux_ops来抽象pinmux有关的操作，如下：

|   |
|---|
|struct pinmux_ops {  <br>         int (*request) (struct pinctrl_dev *pctldev, unsigned offset);  <br>        int (*free) (struct pinctrl_dev *pctldev, unsigned offset);  <br>        int (*get_functions_count) (struct pinctrl_dev *pctldev);  <br>        const char *(*get_function_name) (struct pinctrl_dev *pctldev,  <br>                                           unsigned selector);  <br>        int (*get_function_groups) (struct pinctrl_dev *pctldev,  <br>                                  unsigned selector,  <br>                                  const char * const **groups,  <br>                                  unsigned *num_groups);  <br>        int (*set_mux) (struct pinctrl_dev *pctldev, unsigned func_selector,  <br>                        unsigned group_selector);  <br>        int (*gpio_request_enable) (struct pinctrl_dev *pctldev,  <br>                                    struct pinctrl_gpio_range *range,  <br>                                     unsigned offset);  <br>        void (*gpio_disable_free) (struct pinctrl_dev *pctldev,  <br>                                   struct pinctrl_gpio_range *range,  <br>                                    unsigned offset);  <br>        int (*gpio_set_direction) (struct pinctrl_dev *pctldev,  <br>                                   struct pinctrl_gpio_range *range,  <br>                                    unsigned offset,  <br>                                   bool input);  <br>        bool strict;  <br>};|

注5：function的概念：

- 为了管理SoC的管脚复用，pinctrl subsystem抽象出function的概念，用来表示I2C0、UART5等功能。pin（或者pin group）所对应的function一经确定，它（们）的管脚复用状态也就确定了
- 和2.2中描述的pin group类似，pinctrl core不关心function的具体形态，只要求pinctrl driver将SoC的所有可能的function枚举出来（格式自行定义，不过需要有编号、名称等内容，可参考[5]中的例子），并注册给pinctrl core。后续pinctrl core将会通过function的索引，访问、控制相应的function
- 另外，有一点大家应该很熟悉：在SoC的设计中，同一个function（如I2C0），可能可以map到不同的pin（或者pin group）上

理解了function的概念之后，struct pinmux_ops中的API就简单了：

> get_functions_count，获取系统中function的个数。
> 
> get_function_name，获取指定function的名称。
> 
> get_function_groups，获取指定function所占用的pin group（可以有多个）。
> 
> set_mux，将指定的pin group（group_selector）设置为指定的function（func_selector）。
> 
> request，检查某个pin是否已作它用，用于管脚复用时的互斥（避免多个功能同时使用某个pin而不知道，导致奇怪的错误）。
> 
> free，request的反操作。
> 
> gpio_request_enable、gpio_disable_free、gpio_set_direction，gpio有关的操作，等到gpio有关的文章中再说明。
> 
> strict，为true时，说明该pin controller不允许某个pin作为gpio和其它功能同时使用。

## 3. pinctrl subsystem的控制逻辑

第2章以struct pinctrl_desc为引子，介绍了pinctrl subsystem中有关pin controller的概念抽象，包括pin、pin group、pinconf、pinmux、pinmux function、等等，相当于从provider的角度理解pinctrl subsystem。那么，问题来了，怎么使用pinctrl subsystem提供的功能控制管脚的配置以及功能复用呢？这看似需要由consumer（例如各个外设的驱动）自行处理，实际上却不尽然：

> 1）前面讲了，由于pinctrl subsystem的特殊性，对于pin configuration以及pin multiplexing而言，要怎么配置、怎么复用，只有pinctrl driver自己知道。同理，各个consumer也是云里雾里，啥都搞不清楚（想象各位编写设备驱动需要用到pinctrl的时候的心情吧！）。
> 
> 2）那这样的配置有道理吗？有！记得我们在[6]中提到过，对一个确定的产品来说，某个设备所使用的pinctrl功能（function）、以及所对应的pin（或者pin group）、还有这些pin（或者pin group）的属性配置，基本上在产品设计的时候就确定好了，consumer没必要（也不想）关心技术细节。因此pinctrl driver就要多做一些事情，帮助consumer厘清pin有关资源的使用情况，并在这些设备需要使用的时候（例如probe时），一声令下，将资源准备好。
> 
> 3）因此，pinctrl subsystem的设计理念就是：不需要consumer关心pin controller的技术细节，只需要在特定的时候向pinctrl driver发出一些简单的指令，告诉pinctrl driver自己的需求即可（例如我在运行时需要使用这样一组配置，在休眠时使用那样一组配置）。
> 
> 4）最后，需求的细节（例如需要使用哪些pin、配置为什么功能、等等），要怎么确定呢？一般是通过machine的配置文件、具体版型的device tree等，告诉pinctrl subsystem，以便在需要的时候使用。

下面小节我们将会根据这些思路，进行更为详细的分析。

#### 3.1 pin state

根据前面的描述，pinctrl driver抽象出来了一些离散的对象：pin（pin group）、function、configuration，并实现了这些对象的控制和配置方式。然后我们回到某一个具体的device上（如SPI2）：

> 该device在某一状态下（如工作状态、休眠状态、等等），所使用的pin（pin group）、pin（pin group）的function和configuration，是唯一确定的。

把上面的话颠倒过来说，就是：

> pin（pin group）以及相应的function和configuration的组合，可以确定一个设备的一个“状态”。

这个状态在pinctrl subsystem中就称作pin state。而pinctrl driver和具体板型有关的部分，需要负责枚举该板型下所有device（当然，特指那些需要pin资源的device）的所有可能的状态，并详细定义这些状态需要使用的pin（或pin group），以及这些pin（或pin group）需要配置为哪种function、哪种配置项。这些状态确定之后，consumer（device driver）就好办了，直接发号施令就行了：

> 喂，pinctrl subsystem，帮忙将我的xxx state激活。

pinctrl subsystem接收到指令后，找到该state的相关信息（pin、function和configuration），并调用pinctrl driver提供的相应API（参考第2章中struct pinctrl_desc有关的内容），控制pin controller即可。

#### 3.2 pin map

在pinctrl subsystem中，pin state有关的信息是通过pin map收集，相关的数据结构如下：

|   |
|---|
|/* include/linux/pinctrl/machine.h */<br><br>struct pinctrl_map {  <br>        const char *dev_name;  <br>        const char *name;  <br>        enum pinctrl_map_type type;  <br>        const char *ctrl_dev_name;  <br>        union {  <br>                struct pinctrl_map_mux mux;  <br>                 struct pinctrl_map_configs configs;  <br>        } data;  <br>};|

> dev_name，device的名称。
> 
> name，pin state的名称。
> 
> ctrl_dev_name，pin controller device的名字。
> 
> type，该map的类型，包括PIN_MAP_TYPE_MUX_GROUP（配置管脚复用）、PIN_MAP_TYPE_CONFIGS_PIN（配置pin）、PIN_MAP_TYPE_CONFIGS_GROUP（配置pin group）、PIN_MAP_TYPE_DUMMY_STATE（不需要任何配置，仅仅为了表示state的存在。
> 
> data，该map需要用到的数据项，是一个联合体，如果map的类型是PIN_MAP_TYPE_CONFIGS_GROUP，则为struct pinctrl_map_mux类型的变量；如果map的类型是PIN_MAP_TYPE_CONFIGS_PIN或者PIN_MAP_TYPE_CONFIGS_GROUP，则为struct pinctrl_map_configs类型的变量。

struct pinctrl_map_mux的定义如下：

|   |
|---|
|struct pinctrl_map_mux {  <br>         const char *group;  <br>        const char *function;  <br>};|

> group，group的名字，指明该map所涉及的pin group。
> 
> function，function的名字，表示该map需要将group配置为哪种function。

struct pinctrl_map_configs的定义如下：

|   |
|---|
|struct pinctrl_map_configs {  <br>        const char *group_or_pin;  <br>        unsigned long *configs;  <br>        unsigned num_configs;  <br>};|

> group_or_pin，pin或者pin group的名字。
> 
> configs，configuration数组，指明要将该group_or_pin配置成“神马样子”。
> 
> num_configs，配置项的个数。
> 
> 注6：讲到这里，应该理解为什么2.3小结中struct pinconf_ops中的api，都不知道configuration到底是什么东西了吧？因为都是pinctrl driver自己安排好的，自产自销，外人（pinctrl subsystem以及consumers）没必要理解！

最后，某一个device的某一种pin state，可以由多个不同类型的map entry组合而成，举例如下[5]：

|   |
|---|
|static struct pinctrl_map mapping[] __initdata = {  <br>        PIN_MAP_MUX_GROUP("foo-i2c.0", PINCTRL_STATE_DEFAULT, "pinctrl-foo", "i2c0", "i2c0"),  <br>        PIN_MAP_CONFIGS_GROUP("foo-i2c.0", PINCTRL_STATE_DEFAULT, "pinctrl-foo", "i2c0", i2c_grp_configs),  <br>        PIN_MAP_CONFIGS_PIN("foo-i2c.0", PINCTRL_STATE_DEFAULT, "pinctrl-foo", "i2c0scl", i2c_pin_configs),  <br>        PIN_MAP_CONFIGS_PIN("foo-i2c.0", PINCTRL_STATE_DEFAULT, "pinctrl-foo", "i2c0sda", i2c_pin_configs),  <br>};|

> 这是一个mapping数组，包含4个map entry，定义了"foo-i2c.0"设备的一个pin state（PINCTRL_STATE_DEFAULT，"default"），该state由一个PIN_MAP_TYPE_MUX_GROUP entry、一个PIN_MAP_TYPE_CONFIGS_GROUP entry以及两个PIN_MAP_TYPE_CONFIGS_PIN entry组成（这些entry的具体含义，大家可参考[5]以及相应的source code理解，这里不再详细说明）。

#### 3.3 通过dts生成pin map

在旧时代，kernel的bsp工程师需要在machine有关的代码中，静态的定义pin map数组（类似于3.2小节中的例子），这一个非常繁琐且不易维护的过程。不过当kernel引入device tree之后，事情就简单了很多：

> pinctrl driver确定了pin map各个字段的格式之后，就可以在dts文件中维护pin state以及相应的mapping table。pinctrl core在初始化的时候，会读取并解析dts，并生成pin map。
> 
> 而各个consumer，可以在自己的dts node中，直接引用pinctrl driver定义的pin state，并在设备驱动的相应的位置，调用pinctrl subsystem提供的API，active或者deactive这些state。

至于dts中pin map描述的格式是什么，则完全由pinctrl driver自己决定，因为，最终的解析工作（dts to map）也是它自己做的（具体可参考后面第4章的介绍）。

## 4. pinctrl subsystem的整体流程

通过前面几章的分析，我们对pinctrl subsystem有了一个比较全面的认识，这里以pinctrl整个使用流程为例，简单的总结一下。

1）pinctrl driver根据pin controller的实际情况，实现struct pinctrl_desc（包括pin/pin group的抽象，function的抽象，pinconf、pinmux的operation API实现，dt_node_to_map的实现，等等），并注册到kernel中。

2）pinctrl driver在pin controller的dts node中，根据自己定义的格式，描述每个device的所有pin state。大致的形式如下（具体可参考kernel中的代码，照葫芦总能画出来瓢~~~）：

>         pinctrl_xxx {                               /* the dts node for pin controller */  
>                 ...  
>                 xxx_state_xxx: xxx_xxx {  /* dts node for xxx device's "xxx state" */  
>                         xxx_pinmux {             /* pinmux entry */  
>                                 xxx = xxxx;  
>                                 xxx = xxxxxxx;  
>                                 ...  
>                         };  
>                         xxx_pinconf {            /* pinconf entry */  
>                                 xxx = xxxx;  
>                                 xxx = xxxxxxx;  
>                                 ...  
>                         };  
>                         xxx_pinconf {  
>                                 xxx = xxxx;  
>                                 xxx = xxxxxxx;  
>                                 ...  
>                         };  
>                        ...  
>                };  
>                ...  
>         };

3）相应的consumer driver可以在自己的dts node中，引用pinctrl driver所定义的pin state，例如：

>                xxx_device: [xxx@xxxxxxxx](mailto:xxx@xxxxxxxx) {  
>                         compatible = "xxx,xxxx";  
>                         ...  
>                         pinctrl-names = "default";  
>                         pinctrl-0 = <&xxx_state_xxx>;  
>                         ...  
>                 };

4）consumer driver在需要的时候，可以调用pinctrl_get/devm_pinctrl_get接口，获得一个pinctrl handle（struct pinctrl类型的指针）。pinctrl subsystem在pinctrl get的过程中，解析consumer device的dts node，找到相应的pin state，进行调用pinctrl driver提供的dt_node_to_map API，解析pin state并转换为pin map。以driver probe时为例，调用过程如下（大家可以自己去看代码）：

> probe  
>     devm_pinctrl_get or pinctrl_get  
>         create_pinctrl(drivers/pinctrl/core.c)  
>             pinctrl_dt_to_map(drivers/pinctrl/devicetree.c)  
>                 dt_to_map_one_config  
>                     pctlops->dt_node_to_map

5）consumer获得pinctrl handle之后，可以调用pinctrl subsystem提供的API（例如pinctrl_select_state），使自己的某个pin state生效。pinctrl subsystem进而调用pinctrl driver提供的各种回调函数，配置pin controller的硬件。  
注7：具体细节就不再描述了，后续开发pinctrl driver的时候，可以再着重说明。

## 5. 参考文档

[1] [linux内核中的GPIO系统之（1）：软件框架](http://www.wowotech.net/gpio_subsystem/io-port-control.html)

[2] [linux内核中的GPIO系统之（2）：pin control subsystem](http://www.wowotech.net/gpio_subsystem/pin-control-subsystem.html)

[3] [Linux内核中的GPIO系统之（3）：pin controller driver代码分析](http://www.wowotech.net/gpio_subsystem/pin-controller-driver.html)

[4] [https://github.com/wowotechX/linux](https://github.com/wowotechX/linux)

[5] [Documentation/pinctrl.txt](https://github.com/wowotechX/linux/blob/x_integration/Documentation/pinctrl.txt)

[6] [X-006-UBOOT-pinctrl driver移植(Bubblegum-96平台)](http://www.wowotech.net/x_project/bubblegum_uboot_pinctrl.html)

原创文章，转发请注明出处。蜗窝科技，[www.wowotech.net](http://www.wowotech.net/gpio_subsystem/pinctrl-driver-summary.html)。

标签: [Linux](http://www.wowotech.net/tag/Linux) [driver](http://www.wowotech.net/tag/driver) [GPIO](http://www.wowotech.net/tag/GPIO) [subsystem](http://www.wowotech.net/tag/subsystem) [pinctrl](http://www.wowotech.net/tag/pinctrl)

[![](http://www.wowotech.net/content/uploadfile/201605/ef3e1463542768.png)](http://www.wowotech.net/support_us.html)

« [CMA模块学习笔记](http://www.wowotech.net/memory_management/cma.html) | [为什么会有文件系统(二)](http://www.wowotech.net/filesystem/396.html)»

**评论：**

**果粒橙特仑苏**  
2021-09-06 22:40

感谢窝窝，醍醐灌顶！

[回复](http://www.wowotech.net/gpio_subsystem/pinctrl-driver-summary.html#comment-8306)

**Waiwai**  
2019-06-12 13:59

你好，请教一个问题：  
文中提到 Pin configuration(对象是pin或者pin group）， 我想这个很好理解，有config_set, config_group_set等接口区分， 但Pin multiplexing(对象是pin或者pin group) 是如何体现的呢？

[回复](http://www.wowotech.net/gpio_subsystem/pinctrl-driver-summary.html#comment-7468)

**好运气**  
2018-01-16 10:25

data，该map需要用到的数据项，是一个联合体，如果map的类型是PIN_MAP_TYPE_CONFIGS_GROUP，则为struct pinctrl_map_mux类型的变量...  
  
这个应该是个笔误，应该是：如果map的类型是PIN_MAP_TYPE_MUX_GROUP，则为struct pinctrl_map_mux类型的变量

[回复](http://www.wowotech.net/gpio_subsystem/pinctrl-driver-summary.html#comment-6479)

**随风律动**  
2017-08-08 09:40

对pinctrl的配置的描述很透彻呢！  
  
我这遇到个问题，将一个模块编成.ko insmod，设置的管脚复用等配置可以正常的运行；可是rmmod之后重新insmod该驱动，在配置管脚时，获取的map->data.configs.num_configs和map->data.configs.configs信息会有一定概率获取错误，查看对应地址，看到map参数会在重新insmod之后概率性被覆盖。  
  
不知道各位大大有没有什么想法和建议吗？

[回复](http://www.wowotech.net/gpio_subsystem/pinctrl-driver-summary.html#comment-5886)

**[wowo](http://www.wowotech.net/)**  
2017-08-08 14:00

@随风律动：这些map信息都是在probe（insmod）的时候动态生成的，按理不会有问题，是不是相关的退出机制不完善（这个模块或者pinctrl driver），重点检查一下。  
PS：问题太细节了，估计也帮不到什么，见谅～～～

[回复](http://www.wowotech.net/gpio_subsystem/pinctrl-driver-summary.html#comment-5891)

**凤凰院胸针**  
2017-06-29 16:47

茅塞顿开！不知后续会不会还有GPIO subsystem的介绍？ 以及博主对入手ARMV8的开发板有什么推荐？

[回复](http://www.wowotech.net/gpio_subsystem/pinctrl-driver-summary.html#comment-5759)

**[wowo](http://www.wowotech.net/)**  
2017-06-29 18:33

@凤凰院胸针：我会顺着站内“X Project”的过程去写，所以GPIO很快就会总结了:-)  
至于开发板，我研究的不多，你可以去网上看看～～

[回复](http://www.wowotech.net/gpio_subsystem/pinctrl-driver-summary.html#comment-5761)

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
    
    - ja  
        [@dream：我看完這段也有相同的想法，引用 @dream ...](http://www.wowotech.net/kernel_synchronization/spinlock.html#8922)
    - 元神高手  
        [围观首席power managerment专家](http://www.wowotech.net/pm_subsystem/device_driver_pm.html#8921)
    - 十七  
        [内核空间的映射在系统启动时就已经设定好，并且在所有进程的页表...](http://www.wowotech.net/process_management/context-switch-arch.html#8920)
    - lw  
        [sparse模型和disconti模型没看出来有什么本质区别...](http://www.wowotech.net/memory_management/memory_model.html#8919)
    - 肥饶  
        [一个没设置好就出错](http://www.wowotech.net/linux_kenrel/516.html#8918)
    - orange  
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
    
    - [u-boot启动流程分析(1)_平台相关部分](http://www.wowotech.net/u-boot/boot_flow_1.html)
    - [X-008-UBOOT-支持命令行(Bubblegum-96平台)](http://www.wowotech.net/x_project/bubblegum_uboot_cmdline.html)
    - [mmap在arm64背后的那个深坑](http://www.wowotech.net/linux_kenrel/516.html)
    - [KASLR](http://www.wowotech.net/memory_management/441.html)
    - [串口通信技术浅析](http://www.wowotech.net/basic_tech/serial_intro.html)
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