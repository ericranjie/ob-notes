
嵌入式Linux _2022年06月20日 07:20_ _广东_

以下文章来源于Linux内核远航者 ，作者Linux内核远航者

# 1.开场白

- 环境：

  处理器架构：arm64
  内核源码：linux-5.11
  ubuntu版本：20.04.1
  代码阅读工具：vim+ctags+cscope

我们知道，linux系统中用户空间和内核空间是隔离的，用户空间程序不能随意的访问内核空间数据，只能通过中断或者异常的方式进入内核态，一般情况下，我们使用copy_to_user和copy_from_user等内核api来实现用户空间和内核空间的数据拷贝，但是像显存这样的设备如果也采用这样的方式就显的效率非常底下，因为用户经常需要在屏幕上进行绘制，要消除这种复制的操作就需要应用程序直接能够访问显存，但是显存被映射到内核空间，应用程序是没有访问权限的，如果显存也能同时映射到用户空间那就不需要拷贝操作了，于是字符设备中提供了mmap接口，可以将内核空间映射的那块物理内存再次映射到用户空间，这样用户空间就可以直接访问不需要任何拷贝操作，这就是我们今天要说的0拷贝技术。

下面是正常情况下用户空间和内核空间数据访问图示：

![[Pasted image 20241023190512.png]]

# 2. 体验一下

首先我们通过一个例子来感受一下：

驱动代码：

注：驱动代码中使用misc框架来实现字符设备，misc框架会处理如创建字符设备，创建设备等通用的字符设备处理，我们只需要关心我们的实际的逻辑即可（内核中大量使用misc设备框架来使用字符设备操作集如ioctl接口，像实现系统虚拟化kvm模块，实现安卓进程间通信的binder模块等）。

0copy_demo.c

```c
#include <linux/module.h>   
#include <linux/kernel.h>   
#include <linux/init.h>   
#include <linux/mm.h>   
#include <linux/miscdevice.h>         
#define MISC_DEV_MINOR 5      
static char *kbuff;         static ssize_t misc_dev_read(struct file *filep, char __user *buf, size_t count, loff_t *offset)   {       int ret;       size_t len = (count > PAGE_SIZE ? PAGE_SIZE : count);       pr_info("###### %s:%d kbuff:%s ######\n", __func__, __LINE__, kbuff);        ret = copy_to_user(buf, kbuff, len);  //这里使用copy_to_user  来进程内核空间到用户空间拷贝       return len - ret;   }      static ssize_t misc_dev_write(struct file *filep, const char __user *buf, size_t count, loff_t *offset)   {    pr_info("###### %s:%d ######\n", __func__, __LINE__);    return 0;   }      static int misc_dev_mmap(struct file *filep, struct vm_area_struct *vma)   {    int ret;    unsigned long start;       start = vma->vm_start;        ret =  remap_pfn_range(vma, start, virt_to_phys(kbuff) >> PAGE_SHIFT,      PAGE_SIZE, vma->vm_page_prot); //使用remap_pfn_range来映射物理页面到进程的虚拟内存中  virt_to_phys(kbuff) >> PAGE_SHIFT作用是将内核的虚拟地址转化为实际的物理地址页帧号  创建页表的权限为通过mmap传递的 vma->vm_page_prot   映射大小为1页       return ret;   }      static long misc_dev_ioctl(struct file *filep, unsigned int cmd, unsigned long args)   {    pr_info("###### %s:%d ######\n", __func__, __LINE__);    return 0;   }            static int misc_dev_open(struct inode *inodep, struct file *filep)   {    pr_info("###### %s:%d ######\n", __func__, __LINE__);    return 0;   }      static int misc_dev_release(struct inode *inodep, struct file *filep)   {    pr_info("###### %s:%d ######\n", __func__, __LINE__);    return 0;   }         static struct file_operations misc_dev_fops = {    .open = misc_dev_open,    .release = misc_dev_release,    .read = misc_dev_read,    .write = misc_dev_write,    .unlocked_ioctl = misc_dev_ioctl,    .mmap = misc_dev_mmap,   };      static struct miscdevice misc_dev = {    MISC_DEV_MINOR,    "misc_dev",    &misc_dev_fops,   };      static int __init misc_demo_init(void)   {    misc_register(&misc_dev);  //注册misc设备 （让misc来帮我们处理创建字符设备的通用代码，这样我们就不需要在去做这些和我们的实际逻辑无关的代码处理了）           kbuff = (char *)__get_free_page(GFP_KERNEL);  //申请一个物理页面（返回对应的内核虚拟地址，内核初始化的时候会做线性映射，将整个ddr内存映射到线性映射区，所以我们不需要做页表映射）    if (NULL == kbuff)     return -ENOMEM;       pr_info("###### %s:%d ######\n", __func__, __LINE__);    return 0;   }      static void __exit misc_demo_exit(void)   {    free_page((unsigned long)kbuff);       misc_deregister(&misc_dev);    pr_info("###### %s:%d ######\n", __func__, __LINE__);   }      module_init(misc_demo_init);   module_exit(misc_demo_exit);   MODULE_LICENSE("GPL");       
```

应用代码：test.c

`#include <stdio.h>   #include <string.h>   #include <unistd.h>   #include <sys/types.h>   #include <sys/stat.h>   #include <fcntl.h>   #include <sys/mman.h>            int main(int argc, char **argv)   {        int fd;    char *ptr;    char buff[32];       fd = open("/dev/misc_dev", O_RDWR);  //打开字符设备    if (fd < 0) {     perror("fail to open");     return -1;    }         ptr = mmap(NULL, 4096, PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0); //映射字符设备到进程的地址空间  权限为可读可写 映射为共享  大小为一个页面    if (ptr == MAP_FAILED) {     perror("fail to mmap");     return -1;    }          memcpy(ptr, "hello world!!!", 15);   //写mmap映射的内存  直接操作，不需要进行特权级别的陷入！          if(read(fd, buff, 15) == -1) {  //读接口  来读取映射的内存，这里会进行内核空间到用户空间的数据拷贝 （需要调用系统调用 在内核空间进行拷贝，然后才能访问）     perror("fail to read");     return -1;    }    puts(buff);         pause();    return 0;   }      `

Makefile文件：

```cpp
export ARCH=arm64   
export CROSS_COMPILE=aarch64-linux-gnu-      KERNEL_DIR ?= ~/kernel/linux-5.11   obj-m := 0copy_demo.o      modules:    $(MAKE) -C $(KERNEL_DIR) M=$(PWD) modules      app:    aarch64-linux-gnu-gcc test.c -o test    cp test $(KERNEL_DIR)/kmodules      clean:    $(MAKE) -C $(KERNEL_DIR) M=$(PWD) clean      install:    cp *.ko $(KERNEL_DIR)/kmodules      
```

编译驱动代码和应用代码，然后拷贝到qemu中运行：

```c
`编译驱动模块代码：   $ make modules      编译并拷贝应用：   $ make app      拷贝驱动模块到qemu：   $ make install       加载驱动代码：   # insmod 0copy_demo.ko   [23328.532194] ###### misc_demo_init:91 ######      查看生成的设备节点：   # ls -l /dev/misc_dev    crw-rw----    1 0        0          10,   5 Apr  7 19:26 /dev/misc_dev      后台运行应用程序：   # ./test&   # [23415.280501] ###### misc_dev_open:56 ######   [23415.281052] ###### misc_dev_read:20 kbuff:hello world!!! ######   hello world!!!      查看test的pid：   # pidof test   1768         查看内存映射：   # cat /proc/1768/maps    aaaabc5a0000-aaaabc5a1000 r-xp 00000000 00:19 8666193                    /mnt/test   aaaabc5b0000-aaaabc5b1000 r--p 00000000 00:19 8666193                    /mnt/test   aaaabc5b1000-aaaabc5b2000 rw-p 00001000 00:19 8666193                    /mnt/test   aaaacf033000-aaaacf054000 rw-p 00000000 00:00 0                          [heap]   ffff8a911000-ffff8aa52000 r-xp 00000000 fe:00 152                        /lib/libc-2.27.so   ffff8aa52000-ffff8aa61000 ---p 00141000 fe:00 152                        /lib/libc-2.27.so   ffff8aa61000-ffff8aa65000 r--p 00140000 fe:00 152                        /lib/libc-2.27.so   ffff8aa65000-ffff8aa67000 rw-p 00144000 fe:00 152                        /lib/libc-2.27.so   ffff8aa67000-ffff8aa6b000 rw-p 00000000 00:00 0    ffff8aa6b000-ffff8aa88000 r-xp 00000000 fe:00 129                        /lib/ld-2.27.so   ffff8aa91000-ffff8aa92000 rw-s 00000000 00:05 152                        /dev/misc_dev      //映射设备文件到用户空间   ffff8aa92000-ffff8aa94000 rw-p 00000000 00:00 0    ffff8aa94000-ffff8aa96000 r--p 00000000 00:00 0                          [vvar]   ffff8aa96000-ffff8aa97000 r-xp 00000000 00:00 0                          [vdso]   ffff8aa97000-ffff8aa98000 r--p 0001c000 fe:00 129                        /lib/ld-2.27.so   ffff8aa98000-ffff8aa9a000 rw-p 0001d000 fe:00 129                        /lib/ld-2.27.so   ffffecb5a000-ffffecb7b000 rw-p 00000000 00:00 0                          [stack]      `
```

执行了以上步骤可以发现最终内核中出现了我在应用程序中写入的“hello world!!!“  字符串，应用程序也能成功读取到（当然本文讲解的0拷贝实现的驱动接口是mmap，而我们读取使用的是read接口，里面我们用copy_to_user来实现的，当然我们可以直接操作mmap映射的内存不需要任何拷贝操作）。

查看应用程序的内存映射发现，/dev/misc_dev设备被映射到了ffff8aa91000-ffff8aa92000这段用户空间地址范围，而且权限为rw-s（可读可写共享）。

写到这里可能大家还是有点不明白那我来解释下：

**1.用户空间不能直接访问内核空间数据（不能直接读写），一旦访问发生缺页异常，产生段错误，必须通过read这样的接口来访问，而read这样的接口会通过系统调用的方式写入到内核态，然后通过copy_to_user这样的内核api来拷贝内核空间数据到用户空间之后才能正常访问。**

**2.通过mmap这种方式之后，用户进程可以直接访问这块内存，memcpy访问的也只不过是用户空间地址，由于访问的时候已经分配好了物理页面和建立好了物理页到虚拟页的映射，所有不会发生缺页异常，也不会发生用户态到内核态的陷入动作。**

**3.用户态进程正常访问内核态数据需要首先通过系统调用等方式陷入内核，进行数据拷贝，然后再次回到用户态，用户态和内核态直接的进出需要进行上下文切换，需要2次上下文切换，需要一定的开销，而mmap映射好之后以后访问都不需要进行上下文切换。**

**4.mmap映射这种方法由于物理页面通过页面共享更加节省内存，而用户态和内核态内存拷贝需要两份物理页面。**

# 3. 实现原理

我们发现通过mmap映射之后，我们在应用程序中可以直接读写这段内存，不需要任何用户空间和内核空间的拷贝动作，大大提高了内存访问效率，那么就是是如何实现的呢？下面我们来揭开它神秘的面纱：

实现0拷贝功不可没的是mmap接口中的**remap_pfn_range**内核api，它将内核空间映射的物理内存重新映射到了用户空间，下面我们来看这个函数的实现：remap_pfn_range函数参数如下：

```cpp
int remap_pfn_range(struct vm_area_struct *vma, unsigned long addr,                            ¦   unsigned long pfn, unsigned long size, pgprot_t prot)   
```

vma为需要映射的进程的vma（进程调用mmap的时候内核会找到一个合适的vma), addr为vma中的一个起始映射地址（这是用户空间的一个虚拟地址），pfn为页帧号（在驱动的mmap接口中会将内核空间的地址转化为物理地址的页帧号），size为需要映射的大小，prot为映射的权限（一般取mmap时传递的权限如rw）

remap_pfn_range实现主要如下代码段：

```cpp
remap_pfn_range       ...       pgd = pgd_offset(mm, addr);                                            flush_cache_range(vma, addr, end);                                     do {                                                                           next = pgd_addr_end(addr, end);                                        err = remap_p4d_range(mm, pgd, addr, next,                                             pfn + (addr >> PAGE_SHIFT), prot);                     if (err)                                                                       break;                                                 } while (pgd++, addr = next, addr != end);                     
```

解释下：remap_pfn_range函数会查找进程的页表，然后填写页表，会将**映射的物理页帧号和访问权限**填写到进程的对应页表中，这会遍历进程的各级页表找到最终的页表项然后进行填写，具体过程自行查看代码。

我们需要注意的是：

**1.一般情况下，用户程序调用mmap只是申请虚拟内存（即是获得一块没有使用用户空间内存，使用vma描述），实际的物理页表都是通过进程访问的时候缺页异常的方式来申请的，但是本场景中是物理页面已经申请好了，进程访问时不会再发生缺页异常，不会申请物理页面。**

**2.同样，物理页面到用户空间虚拟页面的映射也在调用mmap的时候，驱动调用mmap接口的remap_pfn_range映射好了，也不需要在访问的时候发生缺页异常来建立映射。所以，只要用户进程通过mmap映射之后就可以正常访问，访问过程中不会发生缺页异常，映射虚拟页对应的物理页面已经在驱动中申请好映射好。**

下面给出mmap映射原理的图示：

![[Pasted image 20240914165043.png]]

# 4. 应用场景

最后，我们来看下使用**framebuffer的lcd对0拷贝的使用情况**：

```c
fbmem_init    //drivers/video/fbdev/core/fbmem.c   ->register_chrdev(FB_MAJOR, "fb", &fb_fops)  //注册framebuffer字符设备            -> struct file_operations fb_fops = {      ->.mmap =         fb_mmap               -> fb_mmap    //framebuffer的实现               ->vm_iomap_memory                   ->io_remap_pfn_range                       ->remap_pfn_range                          ->  fb_class = class_create(THIS_MODULE, "graphics")  //创建设备类   `

lcd驱动代码中会设置好最终注册framebuffer：

`xxxfb_probe   ->register_framebuffer       ->do_register_framebuffer           -> fb_info->dev = device_create(fb_class, fb_info->device,                            ¦    MKDEV(FB_MAJOR, i), NULL, "fb%d", i);  //创建设备  会出现/dev/fdx 设备节点      
```

可以看到当系统支持framebuffer设备时，在fbmem_init中会创建framebuffer设备类关联字符设备操作集fb_fops，lcd的驱动代码中会调用register_framebuffer创建framebuffer设备（就会创建出了/dev/fdx 设备节点），应用程序就可以通过mmap来映射framebuffer设备到用户空间，然后进行屏幕绘制操作，不需要任何数据拷贝。

# 5. 总结

可以看的出，通过mmap实现0拷贝非常简单，只需要在驱动的mmap接口中调用remap_pfn_range来将内核空间映射的那块物理页再次映射到用户空间即可，这就实现了用户空间和内核空间的数据共享，这和用户进程之间的共享内存机制非常相似，都需要操作进程的页表将这段物理内存映射到进程虚拟地址空间。
