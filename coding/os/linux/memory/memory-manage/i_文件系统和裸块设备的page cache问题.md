作者：[阿克曼](http://www.wowotech.net/author/533) 发布于：2018-4-28 10:16 分类：[文件系统](http://www.wowotech.net/sort/filesystem)

注：本文代码基于linux-3.18.31，此版本中块缓存已经合入页缓存。
## 普通文件的address space

文件系统读取文件一般会使用do_generic_file_read()，mapping指向普通文件的address space。如果一个文件的某一块不在page cache中，在find_get_page函数中会创建一个page，并将这个page根据index插入到这个普通文件的address space中。这也是我们熟知的过程。
```cpp
1. static ssize_t do_generic_file_read(struct file *filp, loff_t *ppos,
2.         struct iov_iter *iter, ssize_t written)
3. {
4.     struct address_space *mapping = filp->f_mapping;
5.     struct inode *inode = mapping->host;
6.     struct file_ra_state *ra = &filp->f_ra;
7.     pgoff_t index;
8.     pgoff_t last_index;
9.     pgoff_t prev_index;
10.     unsigned long offset;      /* offset into pagecache page */
11.     unsigned int prev_offset;
12.     int error = 0;

14.     index = *ppos >> PAGE_CACHE_SHIFT;
15.     prev_index = ra->prev_pos >> PAGE_CACHE_SHIFT;
16.     prev_offset = ra->prev_pos & (PAGE_CACHE_SIZE-1);
17.     last_index = (*ppos + iter->count + PAGE_CACHE_SIZE-1) >> PAGE_CACHE_SHIFT;
18.     offset = *ppos & ~PAGE_CACHE_MASK;

20.     for (;;) {
21.         struct page *page;
22.         pgoff_t end_index;
23.         loff_t isize;
24.         unsigned long nr, ret;

26.         cond_resched();
27. find_page:
28.         page = find_get_page(mapping, index);
29.         if (!page) {
30.             page_cache_sync_readahead(mapping,
31.                     ra, filp,
32.                     index, last_index - index);
33.             page = find_get_page(mapping, index);
34.             if (unlikely(page == NULL))
35.                 goto no_cached_page;
36.         }
37.        ......//此处省略约200行
38. }
```
## 块设备的address space

但是在读取文件系统元数据的时候，元数据对应的page会被加入到底层裸块设备的address space中。下面代码的bdev_mapping指向块设备的address space，调用find_get_page_flags()后，一个新的page（如果page不在这个块设备的address space）就被创建并且插入到这个块设备的address space。
```cpp
1. static struct buffer_head *
2. __find_get_block_slow(struct block_device *bdev, sector_t block)
3. {
4.     struct inode *bd_inode = bdev->bd_inode;
5.     struct address_space *bd_mapping = bd_inode->i_mapping;
6.     struct buffer_head *ret = NULL;
7.     pgoff_t index;
8.     struct buffer_head *bh;
9.     struct buffer_head *head;
10.     struct page *page;
11.     int all_mapped = 1;

13.     index = block >> (PAGE_CACHE_SHIFT - bd_inode->i_blkbits);
14.     page = find_get_page_flags(bd_mapping, index, FGP_ACCESSED);
15.     if (!page)
16.         goto out;
17.     ......//此处省略几十行
18. }
```
## 两份缓存?

前面提到的情况是正常的操作流程，属于普通文件的page放在文件的address space，元数据对应的page放在块设备的address space中，大家井水不犯河水，和平共处。但是世事难料，总有一些不按套路出牌的家伙。文件系统在块设备上欢快的跑着，如果有人绕过文件系统，直接去操作块设备上属于文件的数据块，这会出现什么情况？如果这个数据块已经在普通文件的address space中，这次直接的数据块修改能够立马体现到普通文件的缓存中吗？

答案是直接修改块设备上块会新建一个对应这个块的page，并且这个page会被加到块设备的address space中。也就是同一个数据块，在其所属的普通文件的address space中有一个对应的page。同时，在这个块设备的address space中也会有一个与其对应的page，所有的修改都更新到这个块设备address space中的page上。除非重新从磁盘上读取这一块的数据，否则普通文件的文件缓存并不会感知这一修改。
## 实验

口说无凭，实践是检验真理的唯一标准。我在这里准备了一个实验，先将一个文件的数据全部加载到page cache中，然后直接操作块设备修改这个文件的数据块，再读取文件的内容，看看有没有被修改。

为了确认一个文件的数据是否在page cache中，我先介绍一个有趣的工具---vmtouch，这个工具可以显示出一个文件有多少内容已经被加载到page cache。大家可以在github上获取到它的源码，并自行编译安装

[https://github.com/hoytech/vmtouch](https://github.com/hoytech/vmtouch)

现在开始我们的表演：

首先，我们找一个测试文件，就拿我家目录下的read.c来测试，这个文件的内容就是一些凌乱的c代码。

➜ ~ cat read.c 
```cpp
1. #include <stdio.h>
2. #include <unistd.h>
3. #include <fcntl.h>
4. #include <sys/types.h>
5. #include <sys/stat.h>

7. char buf[4096] = {0};

9. int main(int argc, char *argv[])
10. {
11. 	int fd;
12. 	if (argc != 2) {
13. 		printf("argument error.\n");
14. 		return -1;
15. 	}

17. 	fd = open(argv[1], O_RDONLY);
18. 	if (fd < 0) {
19. 		perror("open failed:");
20. 		return -1;
21. 	}

23. 	read(fd, buf, 4096);
24. 	//read(fd, buf, 4096);
25. 	close(fd);
26. }
27. ➜  ~ 
```
接着运行vmtouch，看看这个文件是否在page cache中了，由于这个文件刚才被读取过，所以文件已经全部保存在page cache中了。
```cpp
1. ➜  ~ vmtouch read.c                   
2.            Files: 1
3.      Directories: 0
4.   Resident Pages: 1/1  4K/4K  100%
5.          Elapsed: 0.000133 seconds
6. ➜  ~ 
```
然后我通过debugfs找到read.c的数据块，并且通过dd命令直接修改数据块。  
```cpp
1. Inode: 3945394   Type: regular    Mode:  0644   Flags: 0x80000
2. Generation: 659328746    Version: 0x00000000:00000001
3. User:     0   Group:     0   Project:     0   Size: 386
4. File ACL: 0
5. Links: 1   Blockcount: 8
6. Fragment:  Address: 0    Number: 0    Size: 0
7.  ctime: 0x5ad2f108:60154d80 -- Sun Apr 15 14:28:24 2018
8.  atime: 0x5ad2f108:5db2f37c -- Sun Apr 15 14:28:24 2018
9.  mtime: 0x5ad2f108:5db2f37c -- Sun Apr 15 14:28:24 2018
10. crtime: 0x5ad2f108:5db2f37c -- Sun Apr 15 14:28:24 2018
11. Size of extra inode fields: 32
12. EXTENTS:
13. (0):2681460

15. ➜  ~ dd if=/dev/zero of=/dev/sda2 seek=2681460 bs=4096 count=1 
16. 1+0 records in
17. 1+0 records out
18. 4096 bytes (4.1 kB, 4.0 KiB) copied, 0.000323738 s, 12.7 MB/s
```
修改已经完成，我们看看直接读取这个文件会怎么样。
```cpp
1. ➜  ~ cat read.c 
2. #include <stdio.h>
3. #include <unistd.h>
4. #include <fcntl.h>
5. #include <sys/types.h>
6. #include <sys/stat.h>

8. char buf[4096] = {0};

10. int main(int argc, char *argv[])
11. {
12. 	int fd;
13. 	if (argc != 2) {
14. 		printf("argument error.\n");
15. 		return -1;
16. 	}

18. 	fd = open(argv[1], O_RDONLY);
19. 	if (fd < 0) {
20. 		perror("open failed:");
21. 		return -1;
22. 	}

24. 	read(fd, buf, 4096);
25. 	//read(fd, buf, 4096);
26. 	close(fd);
27. }

29. ➜  ~ vmtouch read.c 
30.            Files: 1
31.      Directories: 0
32.   Resident Pages: 1/1  4K/4K  100%
33.          Elapsed: 0.00013 seconds
```
文件依然在page cache中，所以我们还是能够读取到文件的内容。然而当我们drop cache以后，再读取这个文件，会发现文件内容被清空。
```cpp
1. ➜  ~ vmtouch read.c 
2.            Files: 1
3.      Directories: 0
4.   Resident Pages: 1/1  4K/4K  100%
5.          Elapsed: 0.00013 seconds
6. ➜  ~ echo 3 > /proc/sys/vm/drop_caches                        
7. ➜  ~ vmtouch read.c                   
8.            Files: 1
9.      Directories: 0
10.   Resident Pages: 0/1  0/4K  0%
11.          Elapsed: 0.000679 seconds
12. ➜  ~ cat read.c
13. ➜  ~ 
```
## 总结

普通文件的数据可以保存在它的地址空间中，同时直接访问块设备中此文件的块，也会将这个文件的数据保存在块设备的地址空间中。这两份缓存相互独立，kernel并不会为这种非正常访问同步两份缓存，从而避免了同步的开销。

---

« [fixmap addresses原理](http://www.wowotech.net/memory_management/440.html) | [一次触摸屏中断调试引发的深入探究](http://www.wowotech.net/linux_kenrel/437.html)»

**评论：**

**abccdeg**  
2022-12-20 15:53

篇幅不多，但讲得非常透彻，尤其实验过程，赞！

[回复](http://www.wowotech.net/filesystem/439.html#comment-8726)

**askatwowotech**  
2021-06-15 15:31

请教一下unlink文件后这个文件对应的page frames会自动被reclaim吗? 如果会自动被reclaim是当场reclaim, 还是稍等一段时间以后被reclaim?

[回复](http://www.wowotech.net/filesystem/439.html#comment-8246)

**[bsp](http://www.wowotech.net/)**  
2021-07-16 15:24

@askatwowotech：等内存紧张时，会自动reclaim，即swap。

[回复](http://www.wowotech.net/filesystem/439.html#comment-8251)

**小学生**  
2019-08-09 12:27

楼主在文章开头说：“注：本文代码基于linux-3.18.31，此版本中块缓存已经合入页缓存。”，意思是不是说kernel-3.18.31下，块缓存和页缓存两者不再独立呢？

[回复](http://www.wowotech.net/filesystem/439.html#comment-7581)

**[colin](http://www.colins110.cn/)**  
2019-12-07 15:16

@小学生：应该指的是，在这个版本中，经过VFS对文件的读取，实际上数据存到块缓存（buffer）中，而文件对应的page cache实际指向前面的buffer吧。

[回复](http://www.wowotech.net/filesystem/439.html#comment-7777)

**fishland**  
2018-12-17 10:22

文件层面最终会“看到”修改后的内容，那么本质上也算是同步的了:D

[回复](http://www.wowotech.net/filesystem/439.html#comment-7088)

**[bsp](http://www.wowotech.net/)**  
2018-11-20 11:41

硬文，楼主的调试技巧很丰富！  
one comment: vmtouch先mmap文件，再调用mincore() to determine whether pages are resident in memory.

[回复](http://www.wowotech.net/filesystem/439.html#comment-7043)

**coderall**  
2018-08-15 11:44

这里第二种直接操作文件数据的方式已经绕过文件系统了，也就不存在你说的第二次打开等问题；再者，如果一个文件正在读写，对于数据的一致性，这个应该由应用来维护了

[回复](http://www.wowotech.net/filesystem/439.html#comment-6888)

**[狂奔的蜗牛](http://www.wowotech.net/)**  
2018-06-08 15:40

这种算不算上是一种内核漏洞呢？这个块设备已经被mount的情况下，还允许别的进程进行读写操作，是不是应该在open的时候进行检查，一些特殊情况下，比如被mount了，或者被其他操作独占了的情况下，就不允许打开操作了。

[回复](http://www.wowotech.net/filesystem/439.html#comment-6791)

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
    
    - [一例hardened_usercopy的bug解析](http://www.wowotech.net/linux_kenrel/480.html)
    - [内存初始化代码分析（一）：identity mapping和kernel image mapping](http://www.wowotech.net/memory_management/__create_page_tables_code_analysis.html)
    - [防冲突机制介绍](http://www.wowotech.net/basic_tech/103.html)
    - [Linux内核同步机制之（五）：Read/Write spin lock](http://www.wowotech.net/kernel_synchronization/rw-spinlock.html)
    - [《奔跑吧，Linux内核》已经上架预售了](http://www.wowotech.net/tech_discuss/running_kernel.html)
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