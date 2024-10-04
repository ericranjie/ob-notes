spy_os Linux爱好者
 _2021年08月27日 11:52_

linux平台普遍采用的DRM软件架构中，不仅包含了内核空间驱动层的代码，而且提供应用层的支撑库libdrm。libdrm基于DRI协议通过ioctl与2D图显驱动进行交互，配置图显处理器以及HDMI、MIPI、LVDS等编解码单元。

![图片](https://mmbiz.qpic.cn/mmbiz_png/9aPYe0E1fb0XkZexF5cqTWxvVB8LSXTkrvlJnhX42SBI7DOFok1pUqU11yaIcdeibfeKV9DxEhMWvtBtJWesaew/640?wx_fmt=png&wxfrom=13&tp=wxpic)

from rockchip

验证SoC的图显处理器及其他编解码模块时，可以基于libdrm modetest所提供的功能来丰富我们的verify条目。如单帧、多帧、旋转、缩放、裁剪等等。

### modetest功能及流程

#### 解析命令行参数

通过库函数getopt()处理modetest的命令行参数

![图片](https://mmbiz.qpic.cn/mmbiz_png/9aPYe0E1fb0XkZexF5cqTWxvVB8LSXTkfzTAD7G08d7tWo2zaVatWEat1K2eskVkcLRLzVj5VnbqfnRReQ5tZw/640?wx_fmt=png&wxfrom=13&tp=wxpic)

支持的命令行参数主要包括三类：

1. 查询类
2. 测试类
3. 通用选项
    
![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

与解析命令函参数有关的三个API：
```cpp
1 static int parse_connector(struct pipe_arg *pipe, const char *arg)   2 static int parse_plane(struct plane_arg *plane, const char *p)   
3 static int parse_property(struct property_arg *p, const char *arg)   4 static void parse_fill_patterns(char *arg)   
```
#### 打开DRM设备

打开DRM设备的流程如下：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

modetest只能打开static const char * const modules[]内定义的DRM驱动，默认支持的DRM驱动包括：
```cpp
 1static const char * const modules[] = {    
 2    "i915",    
 3    "amdgpu",    
 4    "radeon",    
 5    "nouveau",    
 6    "vmwgfx",    
 7    "omapdrm",    
 8    "exynos",    
 9    "tilcdc",   
 10    "msm",   
 11    "sti",   
 12    "tegra",   
 13    "imx-drm",   
 14    "rockchip",   
 15    "atmel-hlcdc",   
 16    "fsl-dcu-drm",   
 17    "vc4",   
 18    "virtio_gpu",   
 19    "mediatek",   
 20    "meson",   
 21    "pl111",   
 22    "stm",   
 23    "sun4i-drm",   
 24    "armada-drm",   
 25};
```
当我们自己的图显驱动需要使用modetest进行验证的时候，需要在这里增加驱动名字。DRM驱动的名字定义在kernel driver的drm_driver数据结构中。

 `1struct drm_driver {    2...    3    /** @major: driver major number */    4    int major;    5    /** @minor: driver minor number */    6    int minor;    7    /** @patchlevel: driver patch level */    8    int patchlevel;    9    /** @name: driver name */   10    char *name;   11    /** @desc: driver description */   12    char *desc;   13    /** @date: driver date */   14    char *date;   15...   16};`

在打开设备的过程中，若通过-M参数指定了DRM驱动名，那么打开特定驱动；若未指定DRM驱动名，那么遍历modules[]中指定的DRM驱动。

另外，若没有指定-D参数(没有指定设备名)，默认按照DRM驱动名打开DRM设备。这里面的-D参数是/dev/drixxx编号。例如-D 0，指定打开0号DRM设备。若指定了-D参数，那么首先按照-D指定设备编号来打开DRM设备。

#### 获取设备资源

 `1static struct resources *get_resources(struct device *dev)    2{    3…    4    drmModeGetCrtc();    5    drmModeGetEncoder();    6    drmModeGetConnector ();    7    drmModeGetFB();    8    drmModeGetPlane();    9…   10    drmModeObjectGetProperties(,,CRTC);   11drmModeObjectGetProperties(,,CONNECTOR);   12…   13}   14   15dump_resource(&dev, encoders);   16dump_resource(&dev, connectors);   17dump_resource(&dev, crtcs);   18dump_resource(&dev, planes);   19dump_resource(&dev, framebuffers);`

#### 配置property

若通过-w参数指定了property设置，那么执行

`1for (i = 0; i < prop_count; ++i)   2        set_property(&dev, &prop_args[i]);   `

#### DRM配置

DRM的配置根据-a参数决定是否采用原子操作，当使用原子操作时原子更新属性，然后利用drmModeAtomicCommit()统一提交配置信息；否则，依次调用各个模块的配置API，如：set_mode()、set_planes()、set_cursors()、test_page_flip()等。

下面的内容是针对原子操作的。

1.模式配置

`1static void atomic_set_mode(   2    struct device *dev,    3    struct pipe_arg *pipes,    4    unsigned int count)   5   `

2.plane配置

这部分的代码应该是最精彩的部分，里面包括了生成待显示图像数据、配置gamma系数、内存数据填充、配置fb等。

支持4种图像格式，分别是：

`1enum util_fill_pattern {   2    UTIL_PATTERN_TILES,   3    UTIL_PATTERN_PLAIN,   4    UTIL_PATTERN_SMPTE,   5    UTIL_PATTERN_GRADIENT,   6};   `

plane配置的命令行参数是:

`1-P <plane_id>@<crtc_id>:<w>x<h>[+<x>+<y>][*<scale>][@<format>]   `

进行plane配置之前首先按照上面的命令行参数逐项进行解析，最后一个format若不指定，默认使用XR24。

 `1static int parse_plane(struct plane_arg *plane, const char *p)    2{    3...    4    plane->plane_id = strtoul(p, &end, 10);    5...    6    plane->crtc_id = strtoul(p, &end, 10);    7...    8    plane->w = strtoul(p, &end, 10);    9...   10    plane->h = strtoul(p, &end, 10);   11   12    if (*end == '+' || *end == '-') {   13        plane->x = strtol(end, &end, 10);   14...   15        plane->y = strtol(end, &end, 10);   16...   17    }   18...   19    if (*end == '@') {   20        strncpy(plane->format_str, end + 1, 4);   21        plane->format_str[4] = '\0';   22    } else {   23        strcpy(plane->format_str, "XR24");   24    }   25   26    plane->fourcc = util_format_fourcc(plane->format_str);   27...   28}`

关于plane显示功能的配置包括三部分：

**1) part 1 申请内存空间**

user space的程序发起ioctl请求，kernel driver完成内存空间的分配

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**2) part 2 填充图显数据**

根据format格式向申请的内存空间填充rgb或yuv格式的图显数据

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**3) part 3 framebuffer绑定**

创建一个framebuffer-DMA空间，将已准备好的图显数据内存区域与framebuffer进行绑定。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

上面的流程涉及到了3个ioctl命令码，如下：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

#### vsync

这个的本意是进行刷新率的测试，但是可以基于这个做简单的2D多帧测试，就不用引入其他视频测试工具如mplayer。

#### 原子操作统一提交commit

配置结束后统一结交给DRM驱动的ioctl API

 `1/* user space API */    2drmModeAtomicCommit(    3    dev.fd,     4    dev.req,     5    DRM_MODE_ATOMIC_ALLOW_MODESET,     6    NULL);    7    8/* kernel DRM driver    9 * drivers/gpu/drm/drm_atomic_uapi.c   10 */   11drm_mode_atomic_ioctl(   12    struct drm_device *dev,    13    void *data,    14    struct drm_file *file_priv)   15{   16…   17}`

**kernel DRM driver ioctl**

提供的ioctl API函数定义在

1. drivers/gpu/drm/drm_atomic_uapi.c
    
2. drivers/gpu/drm/drm_ioctl.c
    
3. drivers/gpu/drm/drm_auth.c
    

IOCTL COMMAND码定义在

1. include/uapi/drm/drm.h
    

假如依赖原子操作，那么modetest图形显示参数配置调用了 DRM_IOCTL(fd, DRM_IOCTL_MODE_ATOMIC, &atomic);

在kernel驱动中对应drm_ioctl.c中定义：DRM_IOCTL_MODE_ATOMIC, drm_mode_atomic_ioctl 而drm_mode_atomic_ioctl代码实现在drm_atomic_uapi.c中

 `1int drm_mode_atomic_ioctl(struct drm_device *dev,    2              void *data, struct drm_file *file_priv)    3{    4    struct drm_mode_atomic *arg = data;    5    ...    6    unsigned int copied_objs, copied_props;    7    struct drm_atomic_state *state;    8    struct drm_modeset_acquire_ctx ctx;    9    struct drm_out_fence_state *fence_state;   10    int ret = 0;   11    unsigned int i, j, num_fences;   12…   13}`

原子提交的用户空间代码与内核空间代码的交互流程如下：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

### modetest验证

**虚拟机版本：**

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**ubuntu版本：**

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**libdrm代码版本：**

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

ubuntu虚拟机需要切换到命令行模式，否则界面模式下会占用dri设备导致modetest无法正常执行。

进入命令行模式方法:

ctrl+alt+f1

退出命令行模式方法：

ctrl+alt+f7

切换到命令行模式下执行modetest

#### 单帧验证：

`./modetest -M vmwgfx -D 0 -a -s 34@36:1280x960  -P 32@36:1280x960 -Ftiles   `

结果

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**多帧验证：**

借助vsync这个选项修改代码，轮询显示modetest支持的4种图像格式来实现多个图像的输出

修改的代码如下：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

命令行：

`./testmode -M vmwgfx -D 0 -a -s 34@36:1280x960  -P 32@36:1280x960 -v -Ftiles   `

限于篇幅没能把libdrm的功能完全的呈现出来，关于图显系统的验证还有很多的方法，这里只是冰山的一角。欢迎补充交流

  

  

- EOF -

推荐阅读  点击标题可跳转

1、[ARM 汇编入门指南](http://mp.weixin.qq.com/s?__biz=MzAxODI5ODMwOA==&mid=2666557033&idx=3&sn=c13e58ef81c3f398a691afd846632b3a&chksm=80dcaac2b7ab23d43dbf84da00214f25ecc869c1955acfd9439e5d34d4e19ee9e6e605af7fb3&scene=21#wechat_redirect)

2、[如何每小时改变你的 Linux 桌面壁纸](http://mp.weixin.qq.com/s?__biz=MzAxODI5ODMwOA==&mid=2666557016&idx=3&sn=ac227fcc26c345f165680179cd91eb22&chksm=80dcaaf3b7ab23e5e63933a2ff5abd3ee090169eeb12cb44884c7dc6b12e8080c2089e37b085&scene=21#wechat_redirect)

3、[Linux 字节对齐的那些事](http://mp.weixin.qq.com/s?__biz=MzAxODI5ODMwOA==&mid=2666556974&idx=2&sn=54943c4765d4dcda865435c22f6acda3&chksm=80dcaa85b7ab23931d2ca82655e1077db5f644049a4c60ff6d1c5231f3b8f95b6c489cc0ac44&scene=21#wechat_redirect)

  

看完本文有收获？请分享给更多人  

推荐关注「Linux 爱好者」，提升Linux技能

![](http://mmbiz.qpic.cn/mmbiz_png/9aPYe0E1fb3sjicd8JxDra10FRIqT54Zke2sfhibTDdtdnVhv5Qh3wLHZmKPjiaD7piahMAzIH6Cnltd1Nco17Ihjw/300?wx_fmt=png&wxfrom=19)

**Linux爱好者**

点击获取《每天一个Linux命令》系列和精选Linux技术资源。「Linux爱好者」日常分享 Linux/Unix 相关内容，包括：工具资源、使用技巧、课程书籍等。

75篇原创内容

公众号

点赞和在看就是最大的支持❤️

阅读 385

​

写留言

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/9aPYe0E1fb3sjicd8JxDra10FRIqT54Zke2sfhibTDdtdnVhv5Qh3wLHZmKPjiaD7piahMAzIH6Cnltd1Nco17Ihjw/300?wx_fmt=png&wxfrom=18)

Linux爱好者

赞2在看

写留言

写留言

**留言**

暂无留言