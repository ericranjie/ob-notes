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

作者：[wowo](http://www.wowotech.net/author/2 "runangaozhong@163.com") 发布于：2017-9-13 22:18 分类：[X Project](http://www.wowotech.net/sort/x_project)

## 1. 前言

本文将基于本站GPIO subsystem\[1\]相关的文章，结合”[X Project](http://www.wowotech.net/sort/x_project)”的开发过程，实现一个简单的gpio driver，并利用gpiolib提供的sysfs api进行简单的测试，进而加深对gpio相关概念的理解。

注1：本文后续的描述，kernel基于本站“[X Project](http://www.wowotech.net/sort/x_project)”所使用的kernel版本，硬件基于 ”[X Project](http://www.wowotech.net/sort/x_project)”所使用的“Bubbugum-96”平台\[2\]。

## 2. 添加gpio driver的驱动框架

kernel中所有驱动的开发都是一个套路，包括如下几个步骤：

a）新建driver文件（例如drivers/gpio/gpio-owl.c）并从其它地方copy一个架子（例如从drivers/pinctrl/pinctrl-owl.c中，并将pinctrl替换为gpio）。

b）修改drivers/gpio/Kconfig和drivers/gpio/Makefile，将这个新的driver添加到编译体系中：

> $ git diff drivers/gpio/\
> ...\
> --- a/drivers/gpio/Kconfig\
> +++ b/drivers/gpio/Kconfig\
> @@ -342,6 +342,12 @@ config GPIO_OMAP\
> help\
> Say yes here to enable GPIO support for TI OMAP SoCs.\
> +config GPIO_OWL\
> +       bool "GPIO driver for Actions OWL soc"\
> +       default y if ARCH_OWL\
> +       help\
> +         This selects the gpio driver for Actions OWL soc.\
> +\
> ...
>
> --- a/drivers/gpio/Makefile\
> +++ b/drivers/gpio/Makefile\
> ...\
> +obj-$(CONFIG_GPIO_OWL)         += gpio-owl.o\
> obj-$(CONFIG_GPIO_PCA953X)     += gpio-pca953x.o\
> ...

注2：参考drivers/gpio/Kconfig中有关的注释，为了使我们的driver看着更专业、更规范，我们需要以字母为顺序把我们新的driver放到合适的位置：

> config GPIO_GENERIC\
> tristate
>
> # put drivers in the right section, in alphabetical order
>
> # This symbol is selected by both I2C and SPI expanders
>
> config GPIO_MAX730X\
> tristate

c）修改arch/arm64/Kconfig.platforms，选中ARCH_REQUIRE_GPIOLIB配置项，这样就会默认编译gpio有关的代码。

> vim@ubuntu:~/work/xprj/linux$ git diff arch/arm64/Kconfig.platforms\
> diff --git a/arch/arm64/Kconfig.platforms b/arch/arm64/Kconfig.platforms\
> index b03d2538..48c4ae2 100644\
> --- a/arch/arm64/Kconfig.platforms\
> +++ b/arch/arm64/Kconfig.platforms\
> @@ -178,6 +178,7 @@ config ARCH_OWL\
> bool "Actions OWL 64-bit SoC Family"\
> select PINCTRL\
> select PINMUX\
> +       select ARCH_REQUIRE_GPIOLIB\
> help\
> This enables support for Actions OWL based SoCs like the S900.

d）编译kernel，发现新的driver已经编译进来了

> vim@ubuntu:~/work/xprj/build$ make kernel\
> CHK     include/generated/timeconst.h\
> CHK     include/generated/asm-offsets.h\
> CALL    /home/vim/work/xprj/linux/scripts/checksyscalls.sh\
> CHK     include/generated/compile.h\
> CC      drivers/gpio/gpio-owl.o\
> LD      drivers/gpio/built-in.o\
> CC      drivers/irqchip/irqchip.o\
> CC      drivers/irqchip/irq-gic.o

至此GPIO driver的驱动框架已经成型（具体的patch如下），后续根据实际的硬件，往里面填充东西使其慢慢完善即可。

> [https://github.com/wowotechX/linux/commit/2e8b86d7f2e4e892d4a9e1d82cbaa6d3051cfa56](https://github.com/wowotechX/linux/commit/2e8b86d7f2e4e892d4a9e1d82cbaa6d3051cfa56)

## 3. 添加gpiochip

由\[3\]中的介绍可知，编写gpio driver驱动的主要工作就是使用gpiochip抽象gpio控制器，一般情况下，有两种抽象策略：

> a）使用一个gpiochip抽象并管理系统中所有的gpio。
>
> b）每个bank（有关bank的概念，熟悉gpio的同学应该都了解，这里就不解释了）使用一个gpiochip抽象（一个实现，多个实例）。

至于使用哪种策略，主要是由代码编写的方便与否决定的，可以自行发挥，这里以第二种为例。

gpiochip的实现比较简单，主要包括（可参考后面的patch)：

a）实现request和free接口，可交给pinctrl\[4\]（可以先什么都不做）。

b）实现direction_input、direction_output、get 、set等接口。

c）根据实际情况，填充其它字段。

> patch：[https://github.com/wowotechX/linux/commit/d05448f039f961f09226f8ef30cba6b706aa8c15](https://github.com/wowotechX/linux/commit/d05448f039f961f09226f8ef30cba6b706aa8c15)

## 4. 结合gpiolib提供的sysfs api调试

实现基本的功能之后，我们可以借助gpiolib提供的sysfs api进行简单的验证和测试，步骤如下：

#### 4.1 打开SYSFS配置项（CONFIG_SYSFS）和GPIO_SYSFS配置项（CONFIG_GPIO_SYSFS）

> make kernel-config
>
> File systems  --->\
> Pseudo filesystems  --->\
> d\[\*\] sysfs file system support
>
> Device Drivers  --->\
> -*- GPIO Support  --->\
> \[*\]   /sys/class/gpio/... (sysfs interface)

结果如下：

> --- a/arch/arm64/configs/xprj_defconfig\
> +++ b/arch/arm64/configs/xprj_defconfig\
> ...\
> +CONFIG_GPIO_SYSFS=y
>
> # CONFIG_HWMON is not set
>
> ...\
> -# CONFIG_SYSFS is not set
>
> # CONFIG_MISC_FILESYSTEMS is not set

#### 4.2 修改dts，增加gpio的node

注2：“Bubbugum-96”平台\[2\]平台中，pinctrl和gpio是同一个模块，控制寄存器都在一起，这里偷懒，不想区分哪个是pinctrl的、哪个是gpio的了，就使用相同的寄存器范围就行了。不过呢，这一来我们在代码中就不能使用devm_ioremap_resource进行寄存器映射（devm_xxx类的接口具有排他性），要换成ioremap，具体请参考下面的改动。

> +       gpioa: gpio@0xe01b0000 {\
> +               compatible = "actions,s900-gpio";\
> +               reg = \<0 0="" 12="" 0xe01b0000="">;\
> +       };\
> +       gpiob: [gpio@0xe01b000c](mailto:gpio@0xe01b000c) {\
> +               compatible = "actions,s900-gpio";\
> +               reg = \<0 0="" 12="" 0xe01b000c="">;\
> +       };\
> ...

> --- a/drivers/pinctrl/pinctrl-owl.c\
> +++ b/drivers/pinctrl/pinctrl-owl.c\
> ...\
> -       priv->membase = devm_ioremap_resource(dev, resource); \
> +      priv->membase = ioremap(resource->start, resource_size(resource));

以上4.1和4.2的修改可参考下面的patch：[https://github.com/wowotechX/linux/commit/3c8fa7f1076cc6786166b7c3c742ce19ec640c53](https://github.com/wowotechX/linux/commit/3c8fa7f1076cc6786166b7c3c742ce19ec640c53)

#### 4.3 编译并在板子上运行

跑起来后发现有如下错误：

> \[    0.218562\] gpio gpiochip1: GPIO integer space overlap, cannot add chip\
> \[    0.225124\] gpiochip_add_data: GPIOs 0..31 (owl-gpio) failed to register\
> \[    0.231781\] owl_gpio e01b000c.gpio: Failed to add gpiochip\
> \[    0.237249\] owl_gpio: probe of e01b000c.gpio failed with error -16\
> \[    0.243437\] owl_gpio e01b0018.gpio: owl_gpio_probe\
> \[    0.248187\] gpio gpiochip1: GPIO integer space overlap, cannot add chip\
> \[    0.254749\] gpiochip_add_data: GPIOs 0..31 (owl-gpio) failed to register\
> \[    0.261437\] owl_gpio e01b0018.gpio: Failed to add gpiochip\
> \[    0.266906\] owl_gpio: probe of e01b0018.gpio failed with error -16

总结来说，原因有二：

1） 每一个gpio bank（4.2中的dts node）要对应一个gpiochip结构，我们上面的移植忽略了这个事情。

2）每一个gpiochip的base要小心维护，要在整个gpio subsystem中唯一，因为sysfs api以base命名相应的gpiochip（可参考后面的介绍），并以base和nrgpio计算逻辑上的gpio number，我们可以简单的将base设置为0、32、64、等等。

修改的方法如下：

> diff --git a/arch/arm64/boot/dts/actions/s900-bubblegum.dts b/arch/arm64/boot/dts/actions/s900-bubblegum.dts\
> index 2c959d2..6119dbc 100644\
> --- a/arch/arm64/boot/dts/actions/s900-bubblegum.dts\
> +++ b/arch/arm64/boot/dts/actions/s900-bubblegum.dts\
> @@ -55,26 +55,32 @@\
> gpioa: gpio@0xe01b0000 {\
> compatible = "actions,s900-gpio";\
> reg = \<0 0="" 12="" 0xe01b0000="">;\
> +               base = \<0>;\
> };\
> gpiob: gpio@0xe01b000c {\
> compatible = "actions,s900-gpio";\
> reg = \<0 0="" 12="" 0xe01b000c="">;\
> +               base = \<32>;\
> };            \
> ...
>
> diff --git a/drivers/gpio/gpio-owl.c b/drivers/gpio/gpio-owl.c\
> index f71bc7a..e32e292 100644\
> --- a/drivers/gpio/gpio-owl.c\
> +++ b/drivers/gpio/gpio-owl.c\
> @@ -20,6 +20,7 @@\
> #include\
> #include\
> #include\
> +#include\
> #include\
> #include
>
> @@ -34,6 +35,8 @@
>
> struct owl_gpio_priv {\
> void \_\_iomem            \*membase;\
> +\
> +       struct gpio_chip        gpiochip;\
> };
>
> ...
>
> @@ -166,7 +169,10 @@ static int owl_gpio_probe(struct platform_device \*pdev)\
> return PTR_ERR(priv->membase);\
> }
>
> -       ret = devm_gpiochip_add_data(dev, &owl_gpio_chip, priv);\
> +       memcpy(&priv->gpiochip, &owl_gpio_chip, sizeof(struct gpio_chip));\
> +       of_property_read_u32(dev->of_node, "base", &priv->gpiochip.base);\
> +\
> +       ret = devm_gpiochip_add_data(dev, &priv->gpiochip, priv);\
> if (ret \< 0) {\
> dev_err(dev, "Failed to add gpiochip\\n");\
> return ret;

修改后再次编译运行，已经可以正确probe了：

> \[    0.209124\] owl_gpio e01b0000.gpio: owl_gpio_probe\
> \[    0.213968\] owl_gpio e01b000c.gpio: owl_gpio_probe\
> \[    0.218718\] owl_gpio e01b0018.gpio: owl_gpio_probe\
> \[    0.223468\] owl_gpio e01b0024.gpio: owl_gpio_probe\
> \[    0.228249\] owl_gpio e01b0030.gpio: owl_gpio_probe\
> \[    0.232999\] owl_gpio e01b00f0.gpio: owl_gpio_probe

#### 4.4 通过sysfs api控制gpio点亮、熄灭LED灯

由于当前测试环境（bubblegum96）的rootfs的是在ramdisk中，因此在系统启动后，我们需要手动创建/sys目录并挂载sysfs，如下：

> mkdir /sys\
> mount -t sysfs /sys /sys

sysfs挂在后可以在gpio class下看到gpiolib提供的sysfs api，如下：

> ls /sys/class/gpio\
> export       gpiochip128  gpiochip32   gpiochip96\
> gpiochip0    gpiochip160  gpiochip64   unexport

参考\[5\]中的介绍，我们可以将需要使用的gpio number（逻辑number）写入到export文件，这样就可以将该gpio申请出来，并体现在/sys/class/gpio/目录中。以GPIOA19（LED0）为例（gpio的base是0，因此逻辑number也是19）：

> / # echo 19 > /sys/class/gpio/export\
> sh: write error: No such device

又出现错误，经过排查，原来我们的request接口还没有正确实现，没关系，先返回OK（0）：

> static int owl_gpio_request(struct gpio_chip \*chip, unsigned offset)\
> {\
> \- return pinctrl_request_gpio(offset);\
> \+ //return pinctrl_request_gpio(offset);\
> \+ return 0;\
> }\
> ...

再次测试，OK：

> echo 19 > /sys/class/gpio/export\
> / # ls /sys/class/gpio\
> lass/gpio\
> export       gpiochip0    gpiochip160  gpiochip64   unexport\
> gpio19       gpiochip128  gpiochip32   gpiochip96

然后我们可以向“/sys/class/gpio/gpio19/direction ”中写入“out”字符串，将gpio的方向设置为输出，然后向/sys/class/gpio/gpio19/value中写入“1”或者“0”控制gpio的输出电平，如下：

> / # echo out > /sys/class/gpio/gpio19/direction\
> echo out > /sys/class/gpio/gpio19/di\[   36.780156\] owl_gpio e01b0000.gpio: offset 19, value 0\
> rection
>
> / # echo 1 > /sys/class/gpio/gpio19/value\
> echo 1 > /sys/class/gpio/gpio19/val\[   54.310562\] owl_gpio e01b0000.gpio: offset 19, value 1\
> ue\
> / # echo 0 > /sys/class/gpio/gpio19/value
>
> > /sys/class/gpio/gpio19/valu\[   60.714062\] owl_gpio e01b0000.gpio: offset 19, value 0\
> > e

然后观察LED灯，检查是否可以正确控制，如果可以则测试成功。

以上patch可参考：[https://github.com/wowotechX/linux/commit/30fcb89aabc7eef0043034262f76d3ebe4a9c789](https://github.com/wowotechX/linux/commit/30fcb89aabc7eef0043034262f76d3ebe4a9c789 "https://github.com/wowotechX/linux/commit/30fcb89aabc7eef0043034262f76d3ebe4a9c789")

## 5. 参考文档

\[1\] [http://www.wowotech.net/sort/gpio_subsystem](http://www.wowotech.net/sort/gpio_subsystem "http://www.wowotech.net/sort/gpio_subsystem")

\[2\] [https://github.com/96boards/documentation/blob/master/ConsumerEdition/Bubblegum-96/HardwareDocs/SoC_bubblegum96.pdf](https://github.com/96boards/documentation/blob/master/ConsumerEdition/Bubblegum-96/HardwareDocs/SoC_bubblegum96.pdf)

\[3\] [linux内核中的GPIO系统之（1）：软件框架](http://www.wowotech.net/gpio_subsystem/io-port-control.html)

\[4\] [linux内核中的GPIO系统之（5）：gpio subsysem和pinctrl subsystem之间的耦合](http://www.wowotech.net/gpio_subsystem/pinctrl-and-gpio.html)

\[5\] Documentation/gpio/sysfs.txt

原创文章，转发请注明出处。蜗窝科技，[www.wowotech.net](http://www.wowotech.net/x_project/kernel_gpio_driver_porting_1.html)。

标签: [sysfs](http://www.wowotech.net/tag/sysfs) [driver](http://www.wowotech.net/tag/driver) [GPIO](http://www.wowotech.net/tag/GPIO) [porting](http://www.wowotech.net/tag/porting) [gpiolib](http://www.wowotech.net/tag/gpiolib)

[![](http://www.wowotech.net/content/uploadfile/201605/ef3e1463542768.png)](http://www.wowotech.net/support_us.html)

« [Device Tree（四）：文件结构解析](http://www.wowotech.net/device_model/dt-code-file-struct-parse.html) | [蓝牙协议分析(11)\_BLE安全机制之SM](http://www.wowotech.net/bluetooth/le_security_manager.html)»

**评论：**

**小白**\
2017-09-21 00:22

我看了代码，devm_ioremap_resource具有排他性，这个devm_ioremap好像是没有排他性的!

[回复](http://www.wowotech.net/x_project/kernel_gpio_driver_porting_1.html#comment-6050)

**[wowo](http://www.wowotech.net/)**\
2017-09-21 10:18

@小白：多谢提醒，好像是这个样子的，devm_ioremap_resource中会调用\_\_request_region申请一个“busy resource region”，所谓的busy，就是这段memory区域只能自己使用。用devm_ioremap/ioremap取代就没有这个限制了。

/\*\*

- \_\_request_region - create a new busy resource region
- @parent: parent resource descriptor
- @start: resource start address
- @n: resource region size
- @name: reserving caller's ID string
- @flags: IO resource flags                                                    \
  \*/                                                                            \
  struct resource * \_\_request_region(struct resource \*parent,                    \
  resource_size_t start, resource_size_t n,    \
  const char \*name, int flags)

[回复](http://www.wowotech.net/x_project/kernel_gpio_driver_porting_1.html#comment-6051)

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

  - [X-006-UBOOT-pinctrl driver移植(Bubblegum-96平台)](http://www.wowotech.net/x_project/bubblegum_uboot_pinctrl.html)
  - [我眼中的可穿戴设备](http://www.wowotech.net/tech_discuss/106.html)
  - [ARM64的启动过程之（六）：异常向量表的设定](http://www.wowotech.net/armv8a_arch/238.html)
  - [Linux设备模型(1)\_基本概念](http://www.wowotech.net/device_model/13.html)
  - [Common Clock Framework系统结构](http://www.wowotech.net/pm_subsystem/ccf-arch.html)

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
