成功是急不来的。不计较眼前得失，将注意力真正着眼于正在做的事情本身，持续付出努力，才能一步步向前迈进，逐渐达到理想的目标。不着急，才能从容不迫，结果自会水到渠成。

大家好，我是程序喵！

摊牌了，不装了，其实我是程序喵辛苦工作一天还要回家编辑公众号到大半夜的老婆，希望各位大哥能踊跃转发，完成我一千阅读量的KPI（梦想），谢谢！

咳咳，有点跑题，以下是程序喵的废话，麻烦给个面子划到最后点击在看或者赞，证明我比程序喵人气高，谢谢！

**通过/proc文件系统探究虚拟内存**

我们会通过/proc文件系统找到正在运行的进程的字符串所在的虚拟内存地址，并通过更改此内存地址的内容来更改字符串内容，使你更深入了解虚拟内存这个概念！这之前先介绍下虚拟内存的定义！

**虚拟内存**

虚拟内存是一种实现在计算机软硬件之间的内存管理技术，它将程序使用到的内存地址(虚拟地址)映射到计算机内存中的物理地址，虚拟内存使得应用程序从繁琐的管理内存空间任务中解放出来，提高了内存隔离带来的安全性，虚拟内存地址通常是连续的地址空间，由操作系统的内存管理模块控制，在触发缺页中断时利用分页技术将实际的物理内存分配给虚拟内存，而且64位机器虚拟内存的空间大小远超出实际物理内存的大小，使得进程可以使用比物理内存大小更多的内存空间。

在深入研究虚拟内存前，有几个关键点：

- 每个进程都有它自己的虚拟内存
- 虚拟内存的大小取决于系统的体系结构
- 不同操作管理有着不同的管理虚拟内存的方式，但大多数操作系统的虚拟内存结构如下图：

[https://mmbiz.qpic.cn/mmbiz_png/9lFFFiaKpEr91metvR7pqkhktRGtEHTA1QiaX2JZVOibBBcwaM2WZneNcibxGhJ88C3bdWm5GuCxuWzIzNlnmkWjcQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1](https://mmbiz.qpic.cn/mmbiz_png/9lFFFiaKpEr91metvR7pqkhktRGtEHTA1QiaX2JZVOibBBcwaM2WZneNcibxGhJ88C3bdWm5GuCxuWzIzNlnmkWjcQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

virtual_memory.png

上图并不是特别详细的内存管理图，高地址其实还有内核空间等等，但这不是这篇文章的主题。从图中可以看到高地址存储着命令行参数和环境变量，之后是栈空间、堆空间和可执行程序，其中栈空间向下延申，堆空间向上增长，堆空间需要使用malloc分配，是动态分配的内存的一部分。

首先通过一个简单的C程序探究虚拟内存。

```c
#include <stdlib.h>
#include <stdio.h>
#include <string.h>

/**
 * main - 使用strdup创建一个字符串的拷贝，strdup内部会使用malloc分配空间，
 * 返回新空间的地址，这段地址空间需要外部自行使用free释放
 *
 * Return: EXIT_FAILURE if malloc failed. Otherwise EXIT_SUCCESS
 */
int main(void) {
  char *s;
  s = strdup("test_memory");
  if (s == NULL) {
    fprintf(stderr, "Can't allocate mem with malloc\\n");
    return (EXIT_FAILURE);
  }
  printf("%p\\n", (void *)s);
  return (EXIT_SUCCESS);
}
```

编译运行：

```bash
gcc -Wall -Wextra -pedantic -Werror main.c -o test; ./test
```

输出：0x88f010

我的机器是64位机器，进程的虚拟内存高地址为0xffffffffffffffff， 低地址为0x0，而0x88f010远小于0xffffffffffffffff，因此大概可以推断出被复制的字符串的地址(堆地址)是在内存低地址附近，具体可以通过/proc文件系统验证.ls /proc目录可以看到好多文件，这里主要关注/proc/[pid]/mem和/proc/[pid]/maps

### **mem & maps**

```bash
man proc

/proc/[pid]/mem
    This file can be used to access the pages of a process's memory through open(2), read(2), and lseek(2).

/proc/[pid]/maps
    A  file containing the currently mapped memory regions and their access permissions.
          See mmap(2) for some further information about memory mappings.

              The format of the file is:

       address           perms offset  dev   inode       pathname
       00400000-00452000 r-xp 00000000 08:02 173521      /usr/bin/dbus-daemon
       00651000-00652000 r--p 00051000 08:02 173521      /usr/bin/dbus-daemon
       00652000-00655000 rw-p 00052000 08:02 173521      /usr/bin/dbus-daemon
       00e03000-00e24000 rw-p 00000000 00:00 0           [heap]
       00e24000-011f7000 rw-p 00000000 00:00 0           [heap]
       ...
       35b1800000-35b1820000 r-xp 00000000 08:02 135522  /usr/lib64/ld-2.15.so
       35b1a1f000-35b1a20000 r--p 0001f000 08:02 135522  /usr/lib64/ld-2.15.so
       35b1a20000-35b1a21000 rw-p 00020000 08:02 135522  /usr/lib64/ld-2.15.so
       35b1a21000-35b1a22000 rw-p 00000000 00:00 0
       35b1c00000-35b1dac000 r-xp 00000000 08:02 135870  /usr/lib64/libc-2.15.so
       35b1dac000-35b1fac000 ---p 001ac000 08:02 135870  /usr/lib64/libc-2.15.so
       35b1fac000-35b1fb0000 r--p 001ac000 08:02 135870  /usr/lib64/libc-2.15.so
       35b1fb0000-35b1fb2000 rw-p 001b0000 08:02 135870  /usr/lib64/libc-2.15.so
       ...
       f2c6ff8c000-7f2c7078c000 rw-p 00000000 00:00 0    [stack:986]
       ...
       7fffb2c0d000-7fffb2c2e000 rw-p 00000000 00:00 0   [stack]
       7fffb2d48000-7fffb2d49000 r-xp 00000000 00:00 0   [vdso]

              The address field is the address space in the process that the mapping occupies.
          The perms field is a set of permissions:

                   r = read
                   w = write
                   x = execute
                   s = shared
                   p = private (copy on write)

              The offset field is the offset into the file/whatever;
          dev is the device (major:minor); inode is the inode on that device.   0  indicates
              that no inode is associated with the memory region,
          as would be the case with BSS (uninitialized data).

              The  pathname field will usually be the file that is backing the mapping.
          For ELF files, you can easily coordinate with the offset field
              by looking at the Offset field in the ELF program headers (readelf -l).

              There are additional helpful pseudo-paths:

                   [stack]
                          The initial process's (also known as the main thread's) stack.

                   [stack:<tid>] (since Linux 3.4)
                          A thread's stack (where the <tid> is a thread ID).
              It corresponds to the /proc/[pid]/task/[tid]/ path.

                   [vdso] The virtual dynamically linked shared object.

                   [heap] The process's heap.

              If the pathname field is blank, this is an anonymous mapping as obtained via the mmap(2) function.
          There is no easy  way  to  coordinate
              this back to a process's source, short of running it through gdb(1), strace(1), or similar.

              Under Linux 2.0 there is no field giving pathname.

```

通过mem文件可以访问和修改整个进程的内存页，通过maps可以看到进程当前已映射的内存区域，有地址和访问权限偏移量等，从maps中可以看到堆空间是在低地址而栈空间是在高地址.  从maps中可以看到heap的访问权限是rw，即可写，所以可以通过堆地址找到上个示例程序中字符串的地址，并通过修改mem文件对应地址的内容，就可以修改字符串的内容啦，程序：

```c
#include <stdlib.h>
#include <stdio.h>
#include <string.h>
#include <unistd.h>

/**
 * main - uses strdup to create a new string, loops forever-ever
 *
 * Return: EXIT_FAILURE if malloc failed. Other never returns
 */
int main(void) {
   char *s;
   unsigned long int i;

   s = strdup("test_memory");
   if (s == NULL) {
     fprintf(stderr, "Can't allocate mem with malloc\\n");
     return (EXIT_FAILURE);
   }
   i = 0;
   while (s) {
     printf("[%lu] %s (%p)\\n", i, s, (void *)s);
     sleep(1);
     i++;
   }
   return (EXIT_SUCCESS);
}
```

编译运行：

```bash
gcc -Wall -Wextra -pedantic -Werror main.c -o loop; ./loop
```

输出：

```bash
[0] test_memory (0x21dc010)
[1] test_memory (0x21dc010)
[2] test_memory (0x21dc010)
[3] test_memory (0x21dc010)
[4] test_memory (0x21dc010)
[5] test_memory (0x21dc010)
[6] test_memory (0x21dc010)
...
```

这里可以写一个脚本通过/proc文件系统找到字符串所在位置并修改其内容，相应的输出也会更改。

首先找到进程的进程号

```bash
ps aux | grep ./loop | grep -v grep
zjucad    2542  0.0  0.0   4352   636 pts/3    S+   12:28   0:00 ./loop
```

2542即为loop程序的进程号，cat /proc/2542/maps得到

```bash
00400000-00401000 r-xp 00000000 08:01 811716                             /home/zjucad/wangzhiqiang/loop
00600000-00601000 r--p 00000000 08:01 811716                             /home/zjucad/wangzhiqiang/loop
00601000-00602000 rw-p 00001000 08:01 811716                             /home/zjucad/wangzhiqiang/loop
021dc000-021fd000 rw-p 00000000 00:00 0                                  [heap]
7f2adae2a000-7f2adafea000 r-xp 00000000 08:01 8661324                    /lib/x86_64-linux-gnu/libc-2.23.so
7f2adafea000-7f2adb1ea000 ---p 001c0000 08:01 8661324                    /lib/x86_64-linux-gnu/libc-2.23.so
7f2adb1ea000-7f2adb1ee000 r--p 001c0000 08:01 8661324                    /lib/x86_64-linux-gnu/libc-2.23.so
7f2adb1ee000-7f2adb1f0000 rw-p 001c4000 08:01 8661324                    /lib/x86_64-linux-gnu/libc-2.23.so
7f2adb1f0000-7f2adb1f4000 rw-p 00000000 00:00 0
7f2adb1f4000-7f2adb21a000 r-xp 00000000 08:01 8661310                    /lib/x86_64-linux-gnu/ld-2.23.so
7f2adb3fa000-7f2adb3fd000 rw-p 00000000 00:00 0
7f2adb419000-7f2adb41a000 r--p 00025000 08:01 8661310                    /lib/x86_64-linux-gnu/ld-2.23.so
7f2adb41a000-7f2adb41b000 rw-p 00026000 08:01 8661310                    /lib/x86_64-linux-gnu/ld-2.23.so
7f2adb41b000-7f2adb41c000 rw-p 00000000 00:00 0
7ffd51bb3000-7ffd51bd4000 rw-p 00000000 00:00 0                          [stack]
7ffd51bdd000-7ffd51be0000 r--p 00000000 00:00 0                          [vvar]
7ffd51be0000-7ffd51be2000 r-xp 00000000 00:00 0                          [vdso]
ffffffffff600000-ffffffffff601000 r-xp 00000000 00:00 0                  [vsyscall]

```

看见堆地址范围021dc000-021fd000，并且可读可写，而且021dc000<0x21dc010<021fd000，这就可以确认字符串的地址在堆中，在堆中的索引是0x10(至于为什么是0x10，后面会讲到)，这时可以通过mem文件到0x21dc010地址修改内容，字符串输出的内容也会随之更改，这里通过python脚本实现此功能.

```python
#!/usr/bin/env python3
'''
Locates and replaces the first occurrence of a string in the heap
of a process

Usage: ./read_write_heap.py PID search_string replace_by_string
Where:
- PID is the pid of the target process
- search_string is the ASCII string you are looking to overwrite
- replace_by_string is the ASCII string you want to replace
  search_string with
'''

import sys

def print_usage_and_exit():
    print('Usage: {} pid search write'.format(sys.argv[0]))
    sys.exit(1)

# check usage
if len(sys.argv) != 4:
    print_usage_and_exit()

# get the pid from args
pid = int(sys.argv[1])
if pid <= 0:
    print_usage_and_exit()
search_string = str(sys.argv[2])
if search_string  == "":
    print_usage_and_exit()
write_string = str(sys.argv[3])
if search_string  == "":
    print_usage_and_exit()

# open the maps and mem files of the process
maps_filename = "/proc/{}/maps".format(pid)
print("[*] maps: {}".format(maps_filename))
mem_filename = "/proc/{}/mem".format(pid)
print("[*] mem: {}".format(mem_filename))

# try opening the maps file
try:
    maps_file = open('/proc/{}/maps'.format(pid), 'r')
except IOError as e:
    print("[ERROR] Can not open file {}:".format(maps_filename))
    print("        I/O error({}): {}".format(e.errno, e.strerror))
    sys.exit(1)

for line in maps_file:
    sline = line.split(' ')
# check if we found the heap
    if sline[-1][:-1] != "[heap]":
        continue
    print("[*] Found [heap]:")

# parse line
    addr = sline[0]
    perm = sline[1]
    offset = sline[2]
    device = sline[3]
    inode = sline[4]
    pathname = sline[-1][:-1]
    print("\\tpathname = {}".format(pathname))
    print("\\taddresses = {}".format(addr))
    print("\\tpermisions = {}".format(perm))
    print("\\toffset = {}".format(offset))
    print("\\tinode = {}".format(inode))

# check if there is read and write permission
    if perm[0] != 'r' or perm[1] != 'w':
        print("[*] {} does not have read/write permission".format(pathname))
        maps_file.close()
        exit(0)

# get start and end of the heap in the virtual memory
    addr = addr.split("-")
    if len(addr) != 2:# never trust anyone, not even your OS :)
        print("[*] Wrong addr format")
        maps_file.close()
        exit(1)
    addr_start = int(addr[0], 16)
    addr_end = int(addr[1], 16)
    print("\\tAddr start [{:x}] | end [{:x}]".format(addr_start, addr_end))

# open and read mem
    try:
        mem_file = open(mem_filename, 'rb+')
    except IOError as e:
        print("[ERROR] Can not open file {}:".format(mem_filename))
        print("        I/O error({}): {}".format(e.errno, e.strerror))
        maps_file.close()
        exit(1)

# read heap
    mem_file.seek(addr_start)
    heap = mem_file.read(addr_end - addr_start)

# find string
    try:
        i = heap.index(bytes(search_string, "ASCII"))
    except Exception:
        print("Can't find '{}'".format(search_string))
        maps_file.close()
        mem_file.close()
        exit(0)
    print("[*] Found '{}' at {:x}".format(search_string, i))

# write the new string
    print("[*] Writing '{}' at {:x}".format(write_string, addr_start + i))
    mem_file.seek(addr_start + i)
    mem_file.write(bytes(write_string, "ASCII"))

# close files
    maps_file.close()
    mem_file.close()

# there is only one heap in our example
    break
```

运行这个Python脚本

```bash
zjucad@zjucad-ONDA-H110-MINI-V3-01:~/wangzhiqiang$ sudo ./loop.py 2542 test_memory test_hello
[*] maps: /proc/2542/maps
[*] mem: /proc/2542/mem
[*] Found [heap]:
        pathname = [heap]
        addresses = 021dc000-021fd000
        permisions = rw-p
        offset = 00000000
        inode = 0
        Addr start [21dc000] | end [21fd000]
[*] Found 'test_memory' at 10
[*] Writing 'test_hello' at 21dc010
```

同时字符串输出的内容也已更改

```bash
[633] test_memory (0x21dc010)
[634] test_memory (0x21dc010)
[635] test_memory (0x21dc010)
[636] test_memory (0x21dc010)
[637] test_memory (0x21dc010)
[638] test_memory (0x21dc010)
[639] test_memory (0x21dc010)
[640] test_helloy (0x21dc010)
[641] test_helloy (0x21dc010)
[642] test_helloy (0x21dc010)
[643] test_helloy (0x21dc010)
[644] test_helloy (0x21dc010)
[645] test_helloy (0x21dc010)
```

实验成功.

### **通过实践画出虚拟内存空间分布图**

再列出内存空间分布图

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/226dc7bf-77be-41f2-b22c-c09eea12b42c/Untitled.png)

data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==

基本上每个人或多或少都了解虚拟内存的空间分布，那如何验证它呢，下面会提到.

### **堆栈空间**

首先验证栈空间的位置，我们都知道C中局部变量是存储在栈空间的，malloc分配的内存是存储在堆空间，所以可以通过打印出局部变量地址和malloc的返回内存地址的方式来验证堆栈空间在整个虚拟空间中的位置.

```cpp
#include <stdlib.h>
#include <stdio.h>
#include <string.h>

/**
 * main - print locations of various elements
 *
 * Return: EXIT_FAILURE if something failed. Otherwise EXIT_SUCCESS
 */
int main(void)
{
    int a;
    void *p;

    printf("Address of a: %p\\n", (void *)&a);
    p = malloc(98);
    if (p == NULL)
    {
        fprintf(stderr, "Can't malloc\\n");
        return (EXIT_FAILURE);
    }
    printf("Allocated space in the heap: %p\\n", p);
    return (EXIT_SUCCESS);
}
编译运行:gcc -Wall -Wextra -pedantic -Werror main.c -o test; ./test
输出：
Address of a: 0x7ffedde9c7fc
Allocated space in the heap: 0x55ca5b360670

```

通过结果可以看出堆地址空间在栈地址空间下面，整理如图:

data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==

### **可执行程序**

可执行程序也在虚拟内存中，可以通过打印main函数的地址，并与堆栈地址相比较，即可知道可执行程序地址相对于堆栈地址的分布.

```c
#include <stdlib.h>
#include <stdio.h>
#include <string.h>

/**
 * main - print locations of various elements
 *
 * Return: EXIT_FAILURE if something failed. Otherwise EXIT_SUCCESS
 */
int main(void)
{
    int a;
    void *p;

    printf("Address of a: %p\\n", (void *)&a);
    p = malloc(98);
    if (p == NULL)
    {
        fprintf(stderr, "Can't malloc\\n");
        return (EXIT_FAILURE);
    }
    printf("Allocated space in the heap: %p\\n", p);
    printf("Address of function main: %p\\n", (void *)main);
    return (EXIT_SUCCESS);
}
编译运行:gcc main.c -o test; ./test
输出：
Address of a: 0x7ffed846de2c
Allocated space in the heap: 0x561b9ee8c670
Address of function main: 0x561b9deb378a

```

由于main(0x561b9deb378a) < heap(0x561b9ee8c670) < (0x7ffed846de2c)，可以画出分布图如下:

[https://mmbiz.qpic.cn/mmbiz_png/9lFFFiaKpEr8ZWrBt9SoqIMmdjw6L2WqPy4P6RORCspJLRFjPFiaqKYeLBiaNeRjJG0zA7uqvrR50wxfC9bAqz67g/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1](https://mmbiz.qpic.cn/mmbiz_png/9lFFFiaKpEr8ZWrBt9SoqIMmdjw6L2WqPy4P6RORCspJLRFjPFiaqKYeLBiaNeRjJG0zA7uqvrR50wxfC9bAqz67g/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

virtual_memory_stack_heap_executable.png

### **命令行参数和环境变量**

程序入口main函数可以携带参数:

- **第一个参数(argc): 命令行参数的个数**
- **第二个参数(argv): 指向命令行参数数组的指针**
- **第三个参数(env): 指向环境变量数组的指针**

通过程序可以看见这些元素在虚拟内存中的位置：

```c
#include <stdlib.h>
#include <stdio.h>
#include <string.h>

/**
 * main - print locations of various elements
 *
 * Return: EXIT_FAILURE if something failed. Otherwise EXIT_SUCCESS
 */
int main(int ac, char **av, char **env)
{
        int a;
        void *p;
        int i;

        printf("Address of a: %p\\n", (void *)&a);
        p = malloc(98);
        if (p == NULL)
        {
                fprintf(stderr, "Can't malloc\\n");
                return (EXIT_FAILURE);
        }
        printf("Allocated space in the heap: %p\\n", p);
        printf("Address of function main: %p\\n", (void *)main);
        printf("First bytes of the main function:\\n\\t");
        for (i = 0; i < 15; i++)
        {
                printf("%02x ", ((unsigned char *)main)[i]);
        }
        printf("\\n");
        printf("Address of the array of arguments: %p\\n", (void *)av);
        printf("Addresses of the arguments:\\n\\t");
        for (i = 0; i < ac; i++)
        {
                printf("[%s]:%p ", av[i], av[i]);
        }
        printf("\\n");
        printf("Address of the array of environment variables: %p\\n", (void *)env);
    printf("Address of the first environment variable: %p\\n", (void *)(env[0]));
        return (EXIT_SUCCESS);
}
编译运行:gcc main.c -o test; ./test nihao hello
输出：
Address of a: 0x7ffcc154a748
Allocated space in the heap: 0x559bd1bee670
Address of function main: 0x559bd09807ca
First bytes of the main function:
        55 48 89 e5 48 83 ec 40 89 7d dc 48 89 75 d0
Address of the array of arguments: 0x7ffcc154a848
Addresses of the arguments:
        [./test]:0x7ffcc154b94f [nihao]:0x7ffcc154b956 [hello]:0x7ffcc154b95c
Address of the array of environment variables: 0x7ffcc154a868
Address of the first environment variable: 0x7ffcc154b962

```

结果如下：main(0x559bd09807ca) < heap(0x559bd1bee670) < stack(0x7ffcc154a748) < argv(0x7ffcc154a848) < env(0x7ffcc154a868) < arguments(0x7ffcc154b94f->0x7ffcc154b95c + 6)(6为hello+1('\0')) < env first(0x7ffcc154b962)可以看出所有的命令行参数都是相邻的，并且紧接着就是环境变量.

### **argv和env数组地址是相邻的吗**

上例中argv有4个元素，命令行中有三个参数，还有一个NULL指向标记数组的末尾，每个指针是8字节，8*4=32, argv(0x7ffcc154a848) + 32(0x20) = env(0x7ffcc154a868)，所以argv和env数组指针是相邻的.

### **命令行参数地址紧随环境变量地址之后吗**

首先需要获取环境变量数组的大小，环境变量数组是以NULL结束的，所以可以遍历env数组，检查是否为NULL，获取数组大小，代码如下：

```c
#include <stdlib.h>
#include <stdio.h>
#include <string.h>

/**
 * main - print locations of various elements
 *
 * Return: EXIT_FAILURE if something failed. Otherwise EXIT_SUCCESS
 */
int main(int ac, char **av, char **env)
{
     int a;
     void *p;
     int i;
     int size;

     printf("Address of a: %p\\n", (void *)&a);
     p = malloc(98);
     if (p == NULL)
     {
          fprintf(stderr, "Can't malloc\\n");
          return (EXIT_FAILURE);
     }
     printf("Allocated space in the heap: %p\\n", p);
     printf("Address of function main: %p\\n", (void *)main);
     printf("First bytes of the main function:\\n\\t");
     for (i = 0; i < 15; i++)
     {
          printf("%02x ", ((unsigned char *)main)[i]);
     }
     printf("\\n");
     printf("Address of the array of arguments: %p\\n", (void *)av);
     printf("Addresses of the arguments:\\n\\t");
     for (i = 0; i < ac; i++)
     {
          printf("[%s]:%p ", av[i], av[i]);
     }
     printf("\\n");
     printf("Address of the array of environment variables: %p\\n", (void *)env);
     printf("Address of the first environment variables:\\n");
     for (i = 0; i < 3; i++)
     {
          printf("\\t[%p]:\\"%s\\"\\n", env[i], env[i]);
     }
     /* size of the env array */
     i = 0;
     while (env[i] != NULL)
     {
          i++;
     }
     i++; /* the NULL pointer */
     size = i * sizeof(char *);
     printf("Size of the array env: %d elements -> %d bytes (0x%x)\\n", i, size, size);
     return (EXIT_SUCCESS);
}

编译运行:gcc main.c -o test; ./test nihao hello
输出：
Address of a: 0x7ffd5ebadff4
Allocated space in the heap: 0x562ba4e13670
Address of function main: 0x562ba2f1881a
First bytes of the main function:
        55 48 89 e5 48 83 ec 40 89 7d dc 48 89 75 d0
Address of the array of arguments: 0x7ffd5ebae0f8
Addresses of the arguments:
        [./test]:0x7ffd5ebae94f [nihao]:0x7ffd5ebae956 [hello]:0x7ffd5ebae95c
Address of the array of environment variables: 0x7ffd5ebae118
Address of the first environment variables:
        [0x7ffd5ebae962]:"LS_COLORS=rs=0:di=01;34:ln=01;36:mh=00:pi=40;33:so=01;35:do=01;35:bd=40;33;01:cd=40;33;01:or=40;31;01:mi=00:su=37;41:sg=30;43:ca=30;41:tw=30;42:ow=34;42:st=37;44:ex=01;32:*.tar=01;31:*.tgz=01;31:*.arc=01;31:*.arj=01;31:*.taz=01;31:*.lha=01;31:*.lz4=01;31:*.lzh=01;31:*.lzma=01;31:*.tlz=01;31:*.txz=01;31:*.tzo=01;31:*.t7z=01;31:*.zip=01;31:*.z=01;31:*.Z=01;31:*.dz=01;31:*.gz=01;31:*.lrz=01;31:*.lz=01;31:*.lzo=01;31:*.xz=01;31:*.zst=01;31:*.tzst=01;31:*.bz2=01;31:*.bz=01;31:*.tbz=01;31:*.tbz2=01;31:*.tz=01;31:*.deb=01;31:*.rpm=01;31:*.jar=01;31:*.war=01;31:*.ear=01;31:*.sar=01;31:*.rar=01;31:*.alz=01;31:*.ace=01;31:*.zoo=01;31:*.cpio=01;31:*.7z=01;31:*.rz=01;31:*.cab=01;31:*.wim=01;31:*.swm=01;31:*.dwm=01;31:*.esd=01;31:*.jpg=01;35:*.jpeg=01;35:*.mjpg=01;35:*.mjpeg=01;35:*.gif=01;35:*.bmp=01;35:*.pbm=01;35:*.pgm=01;35:*.ppm=01;35:*.tga=01;35:*.xbm=01;35:*.xpm=01;35:*.tif=01;35:*.tiff=01;35:*.png=01;35:*.svg=01;35:*.svgz=01;35:*.mng=01;35:*.pcx=01;35:*.mov=01;35:*.mpg=01;35:*.mpeg=01;35:*.m2v=01;35:*.mkv=01;35:*.webm=01;35:*.ogm=01;35:*.mp4=01;35:*.m4v=01;35:*.mp4v=01;35:*.vob=01;35:*.qt=01;35:*.nuv=01;35:*.wmv=01;35:*.asf=01;35:*.rm=01;35:*.rmvb=01;35:*.flc=01;35:*.avi=01;35:*.fli=01;35:*.flv=01;35:*.gl=01;35:*.dl=01;35:*.xcf=01;35:*.xwd=01;35:*.yuv=01;35:*.cgm=01;35:*.emf=01;35:*.ogv=01;35:*.ogx=01;35:*.aac=00;36:*.au=00;36:*.flac=00;36:*.m4a=00;36:*.mid=00;36:*.midi=00;36:*.mka=00;36:*.mp3=00;36:*.mpc=00;36:*.ogg=00;36:*.ra=00;36:*.wav=00;36:*.oga=00;36:*.opus=00;36:*.spx=00;36:*.xspf=00;36:"
        [0x7ffd5ebaef4e]:"HOSTNAME=3e8650948c0c"
        [0x7ffd5ebaef64]:"OLDPWD=/"
Size of the array env: 11 elements -> 88 bytes (0x58)

运算结果如下：
root@3e8650948c0c:/ubuntu# bc
bc 1.07.1
Copyright 1991-1994, 1997, 1998, 2000, 2004, 2006, 2008, 2012-2017 Free Software Foundation, Inc.
This is free software with ABSOLUTELY NO WARRANTY.
For details type `warranty'.
obase=16
ibase=16
58+7ffd5ebae118
(standard_in) 3: syntax error
58+7FFD5EBAE118
7FFD5EBAE170
quit

```

通过结果可知7FFD5EBAE170 != 0x7ffd5ebae94f，所以命令行参数地址不是紧随环境变量地址之后。

截至目前画出图表如下：

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/203ccb22-6e4b-492a-9915-ed2b42a9e459/Untitled.png)

data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==

### **栈内存真的向下增长吗**

可以通过调用函数来确认，如果真的是向下增长，那么调用函数的地址应该高于被调用函数地址, 代码如下：

```c
#include <stdlib.h>
#include <stdio.h>
#include <string.h>

void f(void) {
     int a;
     int b;
     int c;

     a = 98;
     b = 1024;
     c = a * b;
     printf("[f] a = %d, b = %d, c = a * b = %d\\n", a, b, c);
     printf("[f] Adresses of a: %p, b = %p, c = %p\\n", (void *)&a, (void *)&b, (void *)&c);
}

int main(int ac, char **av, char **env)
{
     int a;
     void *p;
     int i;
     int size;

     printf("Address of a: %p\\n", (void *)&a);
     p = malloc(98);
     if (p == NULL)
     {
          fprintf(stderr, "Can't malloc\\n");
          return (EXIT_FAILURE);
     }
     printf("Allocated space in the heap: %p\\n", p);
     printf("Address of function main: %p\\n", (void *)main);
     f();
     return (EXIT_SUCCESS);
}
编译运行:gcc main.c -o test; ./test
输出：
Address of a: 0x7ffefc75083c
Allocated space in the heap: 0x564d46318670
Address of function main: 0x564d45b9880e
[f] a = 98, b = 1024, c = a * b = 100352
[f] Adresses of a: 0x7ffefc7507ec, b = 0x7ffefc7507f0, c = 0x7ffefc7507f4
```

结果可知: f{a} 0x7ffefc7507ec < main{a} 0x7ffefc75083c

可画图如下：

[https://mmbiz.qpic.cn/mmbiz_png/9lFFFiaKpEr8ZWrBt9SoqIMmdjw6L2WqPAqIqGjw2icLSic85QJUdU3If9WC9XsGpTf4I5RoOngLKNDU7aq1GMSHg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1](https://mmbiz.qpic.cn/mmbiz_png/9lFFFiaKpEr8ZWrBt9SoqIMmdjw6L2WqPAqIqGjw2icLSic85QJUdU3If9WC9XsGpTf4I5RoOngLKNDU7aq1GMSHg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

其实也可以写一个简单的代码，通过查看/proc文件系统中map内容来查看内存分布，这里就不举例啦.

### **堆内存(malloc)**

### **malloc**

malloc是常用的动态分配内存的函数，malloc申请的内存分配在堆中，注意malloc是glibc函数，不是系统调用.

man malloc:

```c
[...] allocate dynamic memory[...]
void *malloc(size_t size);
[...]
The malloc() function allocates size bytes and returns a pointer to the allocated memory.

```

### **不调用malloc，就不会有堆空间[heap]**

看一段不调用malloc的代码

```c
#include <stdlib.h>
#include <stdio.h>

/**
 * main - do nothing
 *
 * Return: EXIT_FAILURE if something failed. Otherwise EXIT_SUCCESS
 */
int main(void)
{
    getchar();
    return (EXIT_SUCCESS);
}
编译运行：gcc test.c -o 2; ./2
step 1 : ps aux | grep \\ \\./2$
输出：
zjucad    3023  0.0  0.0   4352   788 pts/3    S+   13:58   0:00 ./2
step 2 : /proc/3023/maps
输出：
00400000-00401000 r-xp 00000000 08:01 811723                             /home/zjucad/wangzhiqiang/2
00600000-00601000 r--p 00000000 08:01 811723                             /home/zjucad/wangzhiqiang/2
00601000-00602000 rw-p 00001000 08:01 811723                             /home/zjucad/wangzhiqiang/2
007a4000-007c5000 rw-p 00000000 00:00 0                                  [heap]
7f954ca02000-7f954cbc2000 r-xp 00000000 08:01 8661324                    /lib/x86_64-linux-gnu/libc-2.23.so
7f954cbc2000-7f954cdc2000 ---p 001c0000 08:01 8661324                    /lib/x86_64-linux-gnu/libc-2.23.so
7f954cdc2000-7f954cdc6000 r--p 001c0000 08:01 8661324                    /lib/x86_64-linux-gnu/libc-2.23.so
7f954cdc6000-7f954cdc8000 rw-p 001c4000 08:01 8661324                    /lib/x86_64-linux-gnu/libc-2.23.so
7f954cdc8000-7f954cdcc000 rw-p 00000000 00:00 0
7f954cdcc000-7f954cdf2000 r-xp 00000000 08:01 8661310                    /lib/x86_64-linux-gnu/ld-2.23.so
7f954cfd2000-7f954cfd5000 rw-p 00000000 00:00 0
7f954cff1000-7f954cff2000 r--p 00025000 08:01 8661310                    /lib/x86_64-linux-gnu/ld-2.23.so
7f954cff2000-7f954cff3000 rw-p 00026000 08:01 8661310                    /lib/x86_64-linux-gnu/ld-2.23.so
7f954cff3000-7f954cff4000 rw-p 00000000 00:00 0
7ffed68a1000-7ffed68c2000 rw-p 00000000 00:00 0                          [stack]
7ffed690e000-7ffed6911000 r--p 00000000 00:00 0                          [vvar]
7ffed6911000-7ffed6913000 r-xp 00000000 00:00 0                          [vdso]
ffffffffff600000-ffffffffff601000 r-xp 00000000 00:00 0                  [vsyscall]

```

可以看到，如果不调用malloc，maps中就没有[heap]

下面运行一个带有malloc的程序

```c
#include <stdio.h>
#include <stdlib.h>

/**
 * main - prints the malloc returned address
 *
 * Return: EXIT_FAILURE if something failed. Otherwise EXIT_SUCCESS
 */
int main(void)
{
    void *p;

    p = malloc(1);
    printf("%p\\n", p);
    getchar();
    return (EXIT_SUCCESS);
}
编译运行：gcc test.c -o 3; ./3
输出：0xcc7010
验证步骤及输出：
zjucad@zjucad-ONDA-H110-MINI-V3-01:~/wangzhiqiang$ ps aux | grep \\ \\./3$
zjucad    3113  0.0  0.0   4352   644 pts/3    S+   14:06   0:00 ./3
zjucad@zjucad-ONDA-H110-MINI-V3-01:~/wangzhiqiang$ cat /proc/3113/maps
00400000-00401000 r-xp 00000000 08:01 811726                             /home/zjucad/wangzhiqiang/3
00600000-00601000 r--p 00000000 08:01 811726                             /home/zjucad/wangzhiqiang/3
00601000-00602000 rw-p 00001000 08:01 811726                             /home/zjucad/wangzhiqiang/3
00cc7000-00ce8000 rw-p 00000000 00:00 0                                  [heap]
7fc7e9128000-7fc7e92e8000 r-xp 00000000 08:01 8661324                    /lib/x86_64-linux-gnu/libc-2.23.so
7fc7e92e8000-7fc7e94e8000 ---p 001c0000 08:01 8661324                    /lib/x86_64-linux-gnu/libc-2.23.so
7fc7e94e8000-7fc7e94ec000 r--p 001c0000 08:01 8661324                    /lib/x86_64-linux-gnu/libc-2.23.so
7fc7e94ec000-7fc7e94ee000 rw-p 001c4000 08:01 8661324                    /lib/x86_64-linux-gnu/libc-2.23.so
7fc7e94ee000-7fc7e94f2000 rw-p 00000000 00:00 0
7fc7e94f2000-7fc7e9518000 r-xp 00000000 08:01 8661310                    /lib/x86_64-linux-gnu/ld-2.23.so
7fc7e96f8000-7fc7e96fb000 rw-p 00000000 00:00 0
7fc7e9717000-7fc7e9718000 r--p 00025000 08:01 8661310                    /lib/x86_64-linux-gnu/ld-2.23.so
7fc7e9718000-7fc7e9719000 rw-p 00026000 08:01 8661310                    /lib/x86_64-linux-gnu/ld-2.23.so
7fc7e9719000-7fc7e971a000 rw-p 00000000 00:00 0
7ffc91c18000-7ffc91c39000 rw-p 00000000 00:00 0                          [stack]
7ffc91d5f000-7ffc91d62000 r--p 00000000 00:00 0                          [vvar]
7ffc91d62000-7ffc91d64000 r-xp 00000000 00:00 0                          [vdso]
ffffffffff600000-ffffffffff601000 r-xp 00000000 00:00 0                  [vsyscall]

```

程序中带有malloc，那maps中就有[heap]段，并且malloc返回的地址在heap的地址段中，但是返回的地址却不再heap的最开始地址上，相差了0x10字节，为什么呢?看下面：

### **strace, brk, sbrk**

malloc不是系统调用，它是一个正常函数，它必须调用某些系统调用才可以操作堆内存，通过使用strace工具可以追踪进程的系统调用和信号，为了确认系统调用是malloc产生的，所以在malloc前后添加write系统调用方便定位问题。

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

/**
 * main - let's find out which syscall malloc is using
 *
 * Return: EXIT_FAILURE if something failed. Otherwise EXIT_SUCCESS
 */
int main(void)
{
    void *p;

    write(1, "BEFORE MALLOC\\n", 14);
    p = malloc(1);
    write(1, "AFTER MALLOC\\n", 13);
    printf("%p\\n", p);
    getchar();
    return (EXIT_SUCCESS);
}
编译运行：gcc test.c -o 4
zjucad@zjucad-ONDA-H110-MINI-V3-01:~/wangzhiqiang$ strace ./4
execve("./4", ["./4"], [/* 34 vars */]) = 0
brk(NULL)                               = 0x781000
access("/etc/ld.so.nohwcap", F_OK)      = -1 ENOENT (No such file or directory)
access("/etc/ld.so.preload", R_OK)      = -1 ENOENT (No such file or directory)
open("/etc/ld.so.cache", O_RDONLY|O_CLOEXEC) = 3
fstat(3, {st_mode=S_IFREG|0644, st_size=111450, ...}) = 0
mmap(NULL, 111450, PROT_READ, MAP_PRIVATE, 3, 0) = 0x7f37720fa000
close(3)                                = 0
access("/etc/ld.so.nohwcap", F_OK)      = -1 ENOENT (No such file or directory)
open("/lib/x86_64-linux-gnu/libc.so.6", O_RDONLY|O_CLOEXEC) = 3
read(3, "\\177ELF\\2\\1\\1\\3\\0\\0\\0\\0\\0\\0\\0\\0\\3\\0>\\0\\1\\0\\0\\0P\\t\\2\\0\\0\\0\\0\\0"..., 832) = 832
fstat(3, {st_mode=S_IFREG|0755, st_size=1868984, ...}) = 0
mmap(NULL, 4096, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7f37720f9000
mmap(NULL, 3971488, PROT_READ|PROT_EXEC, MAP_PRIVATE|MAP_DENYWRITE, 3, 0) = 0x7f3771b27000
mprotect(0x7f3771ce7000, 2097152, PROT_NONE) = 0
mmap(0x7f3771ee7000, 24576, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x1c0000) = 0x7f3771ee7000
mmap(0x7f3771eed000, 14752, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_ANONYMOUS, -1, 0) = 0x7f3771eed000
close(3)                                = 0
mmap(NULL, 4096, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7f37720f8000
mmap(NULL, 4096, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7f37720f7000
arch_prctl(ARCH_SET_FS, 0x7f37720f8700) = 0
mprotect(0x7f3771ee7000, 16384, PROT_READ) = 0
mprotect(0x600000, 4096, PROT_READ)     = 0
mprotect(0x7f3772116000, 4096, PROT_READ) = 0
munmap(0x7f37720fa000, 111450)          = 0
write(1, "BEFORE MALLOC\\n", 14BEFORE MALLOC
)         = 14
brk(NULL)                               = 0x781000
brk(0x7a2000)                           = 0x7a2000
write(1, "AFTER MALLOC\\n", 13AFTER MALLOC
)          = 13
fstat(1, {st_mode=S_IFCHR|0620, st_rdev=makedev(136, 3), ...}) = 0
write(1, "0x781010\\n", 90x781010
)               = 9
fstat(0, {st_mode=S_IFCHR|0620, st_rdev=makedev(136, 3), ...}) = 0

```

最后几行的输出可知，malloc主要调用brk系统调用来操作堆内存.

```c
man brk
...
       int brk(void *addr);
       void *sbrk(intptr_t increment);
...
DESCRIPTION
       brk() and sbrk() change the location of the program  break,  which  defines
       the end of the process's data segment (i.e., the program break is the first
       location after the end of the uninitialized data segment).  Increasing  the
       program  break has the effect of allocating memory to the process; decreas‐
       ing the break deallocates memory.

       brk() sets the end of the data segment to the value specified by addr, when
       that  value  is  reasonable,  the system has enough memory, and the process
       does not exceed its maximum data size (see setrlimit(2)).

       sbrk() increments the program's data space  by  increment  bytes.   Calling
       sbrk()  with  an increment of 0 can be used to find the current location of
       the program break.

```

程序中断是虚拟内存中程序数据段结束后的第一个位置的地址，malloc通过调用brk或者sbrk，增加程序中断的值就可以创建新空间来动态分配内存，首次调用brk会返回当前程序中断的地址，第二次调用brk也会返回程序中断的地址，可以发现第二次brk返回地址大于第一次brk返回地址，brk就是通过增加程序中断地址的方式来分配内存，可以看出现在的堆地址范围是0x781000-0x7a2000，通过cat /proc/[pid]/maps也可以验证，此处就不贴上实际验证的结果啦。

### **多次malloc**

如果多次malloc会出现什么现象呢，代码如下：

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

/**
 * main - many calls to malloc
 *
 * Return: EXIT_FAILURE if something failed. Otherwise EXIT_SUCCESS
 */
int main(void)
{
    void *p;

    write(1, "BEFORE MALLOC #0\\n", 17);
    p = malloc(1024);
    write(1, "AFTER MALLOC #0\\n", 16);
    printf("%p\\n", p);

    write(1, "BEFORE MALLOC #1\\n", 17);
    p = malloc(1024);
    write(1, "AFTER MALLOC #1\\n", 16);
    printf("%p\\n", p);

    write(1, "BEFORE MALLOC #2\\n", 17);
    p = malloc(1024);
    write(1, "AFTER MALLOC #2\\n", 16);
    printf("%p\\n", p);

    write(1, "BEFORE MALLOC #3\\n", 17);
    p = malloc(1024);
    write(1, "AFTER MALLOC #3\\n", 16);
    printf("%p\\n", p);

    getchar();
    return (EXIT_SUCCESS);
}
编译运行：gcc test.c -o 5; strace ./5
摘要输出结果如下：
write(1, "BEFORE MALLOC #0\\n", 17BEFORE MALLOC#0
)      = 17
brk(NULL)                               = 0x561605c7a000
brk(0x561605c9b000)                     = 0x561605c9b000
write(1, "AFTER MALLOC #0\\n", 16AFTER MALLOC#0
)       = 16
fstat(1, {st_mode=S_IFCHR|0620, st_rdev=makedev(136, 0), ...}) = 0
write(1, "0x561605c7a260\\n", 150x561605c7a260
)        = 15
write(1, "BEFORE MALLOC #1\\n", 17BEFORE MALLOC#1
)      = 17
write(1, "AFTER MALLOC #1\\n", 16AFTER MALLOC#1
)       = 16
write(1, "0x561605c7aa80\\n", 150x561605c7aa80
)        = 15
write(1, "BEFORE MALLOC #2\\n", 17BEFORE MALLOC#2
)      = 17
write(1, "AFTER MALLOC #2\\n", 16AFTER MALLOC#2
)       = 16
write(1, "0x561605c7ae90\\n", 150x561605c7ae90
)        = 15
write(1, "BEFORE MALLOC #3\\n", 17BEFORE MALLOC#3
)      = 17
write(1, "AFTER MALLOC #3\\n", 16AFTER MALLOC#3
)       = 16
write(1, "0x561605c7b2a0\\n", 150x561605c7b2a0
)        = 15
fstat(0, {st_mode=S_IFCHR|0620, st_rdev=makedev(136, 0), ...}) = 0

```

可以发现并不是每次调用malloc都会触发brk系统调用，首次调用malloc，内部会通过brk系统调用更改程序中断地址，分配出一大块内存空间，后续再调用malloc，malloc内部会优先使用之前分配出来的内存空间，直到内部内存空间已经不够再次分配给外部时才会再次触发brk系统调用.

### **0x10 那丢失的16字节是什么**

上面分析可以看见程序第一次调用malloc返回的地址并不是heap段的首地址，而是相差了0x10个字节，那这16个字节究竟是什么，可以通过程序打印出这前16个字节的内容.

```c
编译运行：gcc test.c -o test;./test
输出：
0x5589436ce260
bytes at 0x5589436ce250:
00 00 00 00 00 00 00 00 11 04 00 00 00 00 00 00
0x5589436cea80
bytes at 0x5589436cea70:
00 00 00 00 00 00 00 00 11 08 00 00 00 00 00 00
0x5589436cf290
bytes at 0x5589436cf280:
00 00 00 00 00 00 00 00 11 0c 00 00 00 00 00 00
0x5589436cfea0
bytes at 0x5589436cfe90:
00 00 00 00 00 00 00 00 11 10 00 00 00 00 00 00
0x5589436d0eb0
bytes at 0x5589436d0ea0:
00 00 00 00 00 00 00 00 11 14 00 00 00 00 00 00
0x5589436d22c0
bytes at 0x5589436d22b0:
00 00 00 00 00 00 00 00 11 18 00 00 00 00 00 00
0x5589436d3ad0
bytes at 0x5589436d3ac0:
00 00 00 00 00 00 00 00 11 1c 00 00 00 00 00 00
0x5589436d56e0
bytes at 0x5589436d56d0:
00 00 00 00 00 00 00 00 11 20 00 00 00 00 00 00
0x5589436d76f0
bytes at 0x5589436d76e0:
00 00 00 00 00 00 00 00 11 24 00 00 00 00 00 00
0x5589436d9b00
bytes at 0x5589436d9af0:
00 00 00 00 00 00 00 00 11 28 00 00 00 00 00 00

```

可以看出规律：这16个字节相当于malloc出来的地址的头，包含一些信息，目前可以看出它包括已经分配的地址空间的大小，第一次malloc申请了0x400(1024)字节，可以发现11 04 00 00 00 00 00 00大于0x400(1024)，这8个字节表示数字 0x 00 00 00 00 00 00 04 11 = 0x400(1024) + 0x10(头的大小16) + 1(后面会说明它的含义)，可以发现每次调用malloc，这前8个字节代表的含义都是malloc字节数+16+1.

可以猜测，malloc内部会把这前16个字节强转成某种数据结构，数据结构包含某些信息，最主要的是已经分配的字节数，尽管我们不了解具体结构，但是也可以通过代码操作这16个字节验证我们上面总结的规律是否正确，注意代码中不调用free释放内存.

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

/**
 * pmem - print mem
 * @p: memory address to start printing from
 * @bytes: number of bytes to print
 *
 * Return: nothing
 */
void pmem(void *p, unsigned int bytes)
{
    unsigned char *ptr;
    unsigned int i;

    ptr = (unsigned char *)p;
    for (i = 0; i < bytes; i++)
    {
        if (i != 0)
        {
            printf(" ");
        }
        printf("%02x", *(ptr + i));
    }
    printf("\\n");
}

/**
 * main - confirm the source code
 *
 * Return: EXIT_FAILURE if something failed. Otherwise EXIT_SUCCESS
 */
int main(void)
{
    void *p;
    int i;
    size_t size_of_the_chunk;
    size_t size_of_the_previous_chunk;
    void *chunks[10];

    for (i = 0; i < 10; i++)
    {
        p = malloc(1024 * (i + 1));
        chunks[i] = (void *)((char *)p - 0x10);
        printf("%p\\n", p);
    }
    free((char *)(chunks[3]) + 0x10);
    free((char *)(chunks[7]) + 0x10);
    for (i = 0; i < 10; i++)
    {
        p = chunks[i];
        printf("chunks[%d]: ", i);
        pmem(p, 0x10);
        size_of_the_chunk = *((size_t *)((char *)p + 8)) - 1;
        size_of_the_previous_chunk = *((size_t *)((char *)p));
        printf("chunks[%d]: %p, size = %li, prev = %li\\n",
              i, p, size_of_the_chunk, size_of_the_previous_chunk);
    }
    return (EXIT_SUCCESS);
}
编译运行输出：
root@3e8650948c0c:/ubuntu# gcc test.c -o test
root@3e8650948c0c:/ubuntu# ./test
0x559721de4260
0x559721de4a80
0x559721de5290
0x559721de5ea0
0x559721de6eb0
0x559721de82c0
0x559721de9ad0
0x559721deb6e0
0x559721ded6f0
0x559721defb00
chunks[0]: 00 00 00 00 00 00 00 00 11 04 00 00 00 00 00 00
chunks[0]: 0x559721de4250, size = 1040, prev = 0
chunks[1]: 00 00 00 00 00 00 00 00 11 08 00 00 00 00 00 00
chunks[1]: 0x559721de4a70, size = 2064, prev = 0
chunks[2]: 00 00 00 00 00 00 00 00 11 0c 00 00 00 00 00 00
chunks[2]: 0x559721de5280, size = 3088, prev = 0
chunks[3]: 00 00 00 00 00 00 00 00 11 10 00 00 00 00 00 00
chunks[3]: 0x559721de5e90, size = 4112, prev = 0
chunks[4]: 10 10 00 00 00 00 00 00 10 14 00 00 00 00 00 00
chunks[4]: 0x559721de6ea0, size = 5135, prev = 4112
chunks[5]: 00 00 00 00 00 00 00 00 11 18 00 00 00 00 00 00
chunks[5]: 0x559721de82b0, size = 6160, prev = 0
chunks[6]: 00 00 00 00 00 00 00 00 11 1c 00 00 00 00 00 00
chunks[6]: 0x559721de9ac0, size = 7184, prev = 0
chunks[7]: 00 00 00 00 00 00 00 00 11 20 00 00 00 00 00 00
chunks[7]: 0x559721deb6d0, size = 8208, prev = 0
chunks[8]: 10 20 00 00 00 00 00 00 10 24 00 00 00 00 00 00
chunks[8]: 0x559721ded6e0, size = 9231, prev = 8208
chunks[9]: 00 00 00 00 00 00 00 00 11 28 00 00 00 00 00 00
chunks[9]: 0x559721defaf0, size = 10256, prev = 0

```

结果可以看出，malloc返回的地址往前的16个字节可以表示已经分配的内存大小, 如图：

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/813ef440-4066-466d-90b9-51474db97a51/Untitled.png)

data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==

注意上述是没有调用free释放内存的结果，然而malloc只用了8个字节表示已经分配的内存大小，那么另外8个字节被用来表示什么含义呢，看下malloc函数的注释:

```c
1055 /*
1056       malloc_chunk details:
1057
1058        (The following includes lightly edited explanations by Colin Plumb.)
1059
1060        Chunks of memory are maintained using a `boundary tag' method as
1061        described in e.g., Knuth or Standish.  (See the paper by Paul
1062        Wilson <ftp://ftp.cs.utexas.edu/pub/garbage/allocsrv.ps> for a
1063        survey of such techniques.)  Sizes of free chunks are stored both
1064        in the front of each chunk and at the end.  This makes
1065        consolidating fragmented chunks into bigger chunks very fast.  The
1066        size fields also hold bits representing whether chunks are free or
1067        in use.
1068
1069        An allocated chunk looks like this:
1070
1071
1072        chunk-> +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
1073                |             Size of previous chunk, if unallocated (P clear)  |
1074                +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
1075                |             Size of chunk, in bytes                     |A|M|P|
1076          mem-> +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
1077                |             User data starts here...                          .
1078                .                                                               .
1079                .             (malloc_usable_size() bytes)                      .
1080                .                                                               |
1081    nextchunk-> +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
1082                |             (size of chunk, but used for application data)    |
1083                +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
1084                |             Size of next chunk, in bytes                |A|0|1|
1085                +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
1086
1087        Where "chunk" is the front of the chunk for the purpose of most of
1088        the malloc code, but "mem" is the pointer that is returned to the
1089        user.  "Nextchunk" is the beginning of the next contiguous chunk.

```

可以看出这16字节有两个含义，前8个字节表示之前的空间有多少没有被分配的字节大小，后8个字节表示当前malloc已经分配的字节大小，通过一段调用free的代码查看：

```cpp
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

/**
 * pmem - print mem
 * @p: memory address to start printing from
 * @bytes: number of bytes to print
 *
 * Return: nothing
 */
void pmem(void *p, unsigned int bytes)
{
    unsigned char *ptr;
    unsigned int i;

    ptr = (unsigned char *)p;
    for (i = 0; i < bytes; i++)
    {
        if (i != 0)
        {
            printf(" ");
        }
        printf("%02x", *(ptr + i));
    }
    printf("\\n");
}

/**
 * main - confirm the source code
 *
 * Return: EXIT_FAILURE if something failed. Otherwise EXIT_SUCCESS
 */
int main(void)
{
    void *p;
    int i;
    size_t size_of_the_chunk;
    size_t size_of_the_previous_chunk;
    void *chunks[10];

    for (i = 0; i < 10; i++)
    {
        p = malloc(1024 * (i + 1));
        chunks[i] = (void *)((char *)p - 0x10);
        printf("%p\\n", p);
    }
    free((char *)(chunks[3]) + 0x10);
    free((char *)(chunks[7]) + 0x10);
    for (i = 0; i < 10; i++)
    {
        p = chunks[i];
        printf("chunks[%d]: ", i);
        pmem(p, 0x10);
        size_of_the_chunk = *((size_t *)((char *)p + 8)) - 1;
        size_of_the_previous_chunk = *((size_t *)((char *)p));
        printf("chunks[%d]: %p, size = %li, prev = %li\\n",
              i, p, size_of_the_chunk, size_of_the_previous_chunk);
    }
    return (EXIT_SUCCESS);
}

编译运行输出：
root@3e8650948c0c:/ubuntu# gcc test.c -o test
root@3e8650948c0c:/ubuntu# ./test
0x55fbebf20260
0x55fbebf20a80
0x55fbebf21290
0x55fbebf21ea0
0x55fbebf22eb0
0x55fbebf242c0
0x55fbebf25ad0
0x55fbebf276e0
0x55fbebf296f0
0x55fbebf2bb00
chunks[0]: 00 00 00 00 00 00 00 00 11 04 00 00 00 00 00 00
chunks[0]: 0x55fbebf20250, size = 1040, prev = 0
chunks[1]: 00 00 00 00 00 00 00 00 11 08 00 00 00 00 00 00
chunks[1]: 0x55fbebf20a70, size = 2064, prev = 0
chunks[2]: 00 00 00 00 00 00 00 00 11 0c 00 00 00 00 00 00
chunks[2]: 0x55fbebf21280, size = 3088, prev = 0
chunks[3]: 00 00 00 00 00 00 00 00 11 10 00 00 00 00 00 00
chunks[3]: 0x55fbebf21e90, size = 4112, prev = 0
chunks[4]: 10 10 00 00 00 00 00 00 10 14 00 00 00 00 00 00
chunks[4]: 0x55fbebf22ea0, size = 5135, prev = 4112
chunks[5]: 00 00 00 00 00 00 00 00 11 18 00 00 00 00 00 00
chunks[5]: 0x55fbebf242b0, size = 6160, prev = 0
chunks[6]: 00 00 00 00 00 00 00 00 11 1c 00 00 00 00 00 00
chunks[6]: 0x55fbebf25ac0, size = 7184, prev = 0
chunks[7]: 00 00 00 00 00 00 00 00 11 20 00 00 00 00 00 00
chunks[7]: 0x55fbebf276d0, size = 8208, prev = 0
chunks[8]: 10 20 00 00 00 00 00 00 10 24 00 00 00 00 00 00
chunks[8]: 0x55fbebf296e0, size = 9231, prev = 8208
chunks[9]: 00 00 00 00 00 00 00 00 11 28 00 00 00 00 00 00
chunks[9]: 0x55fbebf2baf0, size = 10256, prev = 0

```

程序代码通过free释放了3和7数据块的空间，所以4和8的前8个字节已经不全是0啦，和其它不同，它们表示之前数据块没有被分配的大小，也可以注意到4和8块的后8个字节不像其它块一样需要加1啦，可以得出结论，malloc通过是否加1来作为前一个数据块是否已经分配的标志，加1表示前一个数据块已经分配。所以之前的程序代码可以修改为如下形式：

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

/**
 * pmem - print mem
 * @p: memory address to start printing from
 * @bytes: number of bytes to print
 *
 * Return: nothing
 */
void pmem(void *p, unsigned int bytes)
{
    unsigned char *ptr;
    unsigned int i;

    ptr = (unsigned char *)p;
    for (i = 0; i < bytes; i++)
    {
        if (i != 0)
        {
            printf(" ");
        }
        printf("%02x", *(ptr + i));
    }
    printf("\\n");
}

/**
 * main - updating with correct checks
 *
 * Return: EXIT_FAILURE if something failed. Otherwise EXIT_SUCCESS
 */
int main(void)
{
    void *p;
    int i;
    size_t size_of_the_chunk;
    size_t size_of_the_previous_chunk;
    void *chunks[10];
    char prev_used;

    for (i = 0; i < 10; i++)
    {
        p = malloc(1024 * (i + 1));
        chunks[i] = (void *)((char *)p - 0x10);
    }
    free((char *)(chunks[3]) + 0x10);
    free((char *)(chunks[7]) + 0x10);
    for (i = 0; i < 10; i++)
    {
        p = chunks[i];
        printf("chunks[%d]: ", i);
        pmem(p, 0x10);
        size_of_the_chunk = *((size_t *)((char *)p + 8));
        prev_used = size_of_the_chunk & 1;
        size_of_the_chunk -= prev_used;
        size_of_the_previous_chunk = *((size_t *)((char *)p));
        printf("chunks[%d]: %p, size = %li, prev (%s) = %li\\n",
              i, p, size_of_the_chunk,
              (prev_used? "allocated": "unallocated"), size_of_the_previous_chunk);
    }
    return (EXIT_SUCCESS);
}
编译运行输出：
root@3e8650948c0c:/ubuntu# gcc test.c -o test
root@3e8650948c0c:/ubuntu# ./test
chunks[0]: 00 00 00 00 00 00 00 00 11 04 00 00 00 00 00 00
chunks[0]: 0x56254f888250, size = 1040, prev (allocated) = 0
chunks[1]: 00 00 00 00 00 00 00 00 11 08 00 00 00 00 00 00
chunks[1]: 0x56254f888660, size = 2064, prev (allocated) = 0
chunks[2]: 00 00 00 00 00 00 00 00 11 0c 00 00 00 00 00 00
chunks[2]: 0x56254f888e70, size = 3088, prev (allocated) = 0
chunks[3]: 00 00 00 00 00 00 00 00 11 04 00 00 00 00 00 00
chunks[3]: 0x56254f889a80, size = 1040, prev (allocated) = 0
chunks[4]: 00 0c 00 00 00 00 00 00 10 14 00 00 00 00 00 00
chunks[4]: 0x56254f88aa90, size = 5136, prev (unallocated) = 3072
chunks[5]: 00 00 00 00 00 00 00 00 11 18 00 00 00 00 00 00
chunks[5]: 0x56254f88bea0, size = 6160, prev (allocated) = 0
chunks[6]: 00 00 00 00 00 00 00 00 11 1c 00 00 00 00 00 00
chunks[6]: 0x56254f88d6b0, size = 7184, prev (allocated) = 0
chunks[7]: 00 00 00 00 00 00 00 00 11 20 00 00 00 00 00 00
chunks[7]: 0x56254f88f2c0, size = 8208, prev (allocated) = 0
chunks[8]: 10 20 00 00 00 00 00 00 10 24 00 00 00 00 00 00
chunks[8]: 0x56254f8912d0, size = 9232, prev (unallocated) = 8208
chunks[9]: 00 00 00 00 00 00 00 00 11 28 00 00 00 00 00 00
chunks[9]: 0x56254f8936e0, size = 10256, prev (allocated) = 0

```

### **堆空间是向上增长吗？**

通过代码验证：

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

/**
 * main - moving the program break
 *
 * Return: EXIT_FAILURE if something failed. Otherwise EXIT_SUCCESS
 */
int main(void)
{
    int i;

    write(1, "START\\n", 6);
    malloc(1);
    getchar();
    write(1, "LOOP\\n", 5);
    for (i = 0; i < 0x25000 / 1024; i++)
    {
        malloc(1024);
    }
    write(1, "END\\n", 4);
    getchar();
    return (EXIT_SUCCESS);
}
编译运行部分摘要输出：
root@3e8650948c0c:/ubuntu# gcc test.c -o test
root@3e8650948c0c:/ubuntu# strace ./test
execve("./test", ["./test"], 0x7ffe0d7cbd80 /* 10 vars */) = 0
brk(NULL)                               = 0x555a2428f000
access("/etc/ld.so.nohwcap", F_OK)      = -1 ENOENT (No such file or directory)
access("/etc/ld.so.preload", R_OK)      = -1 ENOENT (No such file or directory)
openat(AT_FDCWD, "/etc/ld.so.cache", O_RDONLY|O_CLOEXEC) = 3
fstat(3, {st_mode=S_IFREG|0644, st_size=13722, ...}) = 0
mmap(NULL, 13722, PROT_READ, MAP_PRIVATE, 3, 0) = 0x7f6423455000
close(3)                                = 0
access("/etc/ld.so.nohwcap", F_OK)      = -1 ENOENT (No such file or directory)
openat(AT_FDCWD, "/lib/x86_64-linux-gnu/libc.so.6", O_RDONLY|O_CLOEXEC) = 3
read(3, "\\177ELF\\2\\1\\1\\3\\0\\0\\0\\0\\0\\0\\0\\0\\3\\0>\\0\\1\\0\\0\\0\\260\\34\\2\\0\\0\\0\\0\\0"..., 832) = 832
fstat(3, {st_mode=S_IFREG|0755, st_size=2030544, ...}) = 0
mmap(NULL, 8192, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7f6423453000
mmap(NULL, 4131552, PROT_READ|PROT_EXEC, MAP_PRIVATE|MAP_DENYWRITE, 3, 0) = 0x7f6422e41000
mprotect(0x7f6423028000, 2097152, PROT_NONE) = 0
mmap(0x7f6423228000, 24576, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x1e7000) = 0x7f6423228000
mmap(0x7f642322e000, 15072, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_ANONYMOUS, -1, 0) = 0x7f642322e000
close(3)                                = 0
arch_prctl(ARCH_SET_FS, 0x7f64234544c0) = 0
mprotect(0x7f6423228000, 16384, PROT_READ) = 0
mprotect(0x555a22f5f000, 4096, PROT_READ) = 0
mprotect(0x7f6423459000, 4096, PROT_READ) = 0
munmap(0x7f6423455000, 13722)           = 0
write(1, "START\\n", 6START
)                  = 6
brk(NULL)                               = 0x555a2428f000
brk(0x555a242b0000)                     = 0x555a242b0000
fstat(0, {st_mode=S_IFCHR|0620, st_rdev=makedev(136, 0), ...}) = 0
read(0,
"\\n", 1024)                     = 1
write(1, "LOOP\\n", 5LOOP
)                   = 5
brk(0x555a242d1000)                     = 0x555a242d1000
brk(0x555a242f2000)                     = 0x555a242f2000
brk(0x555a24313000)                     = 0x555a24313000
brk(0x555a24334000)                     = 0x555a24334000
brk(0x555a24355000)                     = 0x555a24355000
brk(0x555a24376000)                     = 0x555a24376000
brk(0x555a24397000)                     = 0x555a24397000
brk(0x555a243b8000)                     = 0x555a243b8000
brk(0x555a243d9000)                     = 0x555a243d9000
brk(0x555a243fa000)                     = 0x555a243fa000

```

可以看出堆空间是向上增长的.

### **随机化地址空间布局**

从开始到现在运行了好多个进程，通过查看对应进程的maps，发现每个进程的heap的起始地址和可执行程序的结束地址都不紧邻，而且差距还每次都不相同.

```bash
[3718]: 01195000 – 00602000 = b93000
[3834]: 024d6000 – 00602000 = 1ed4000
[4014]: 00e70000 – 00602000 = 86e000
[4172]: 01314000 – 00602000 = d12000
[7972]: 00901000 – 00602000 = 2ff000
```

可以看出这个差值是随机的，查看fs/binfmt_elf.c源代码

```c
if ((current->flags & PF_RANDOMIZE) && (randomize_va_space > 1)) {
                current->mm->brk = current->mm->start_brk =
                        arch_randomize_brk(current->mm);
#ifdef compat_brk_randomized
                current->brk_randomized = 1;
#endif
        }
// current->mm->brk是当前进程程序中断的地址
```

arch_randomize_brk函数在arch/x86/kernel/process.c中

```c
unsigned long arch_randomize_brk(struct mm_struct *mm)
{
        unsigned long range_end = mm->brk + 0x02000000;
        return randomize_range(mm->brk, range_end, 0) ? : mm->brk;
}
```

randomize_range函数在drivers/char/random.c中

```c
/*
 * randomize_range() returns a start address such that
 *
 *    [...... <range> .....]
 *  start                  end
 *
 * a <range> with size "len" starting at the return value is inside in the
 * area defined by [start, end], but is otherwise randomized.
 */
unsigned long
randomize_range(unsigned long start, unsigned long end, unsigned long len)
{
        unsigned long range = end - len - start;

        if (end <= start + len)
                return 0;
        return PAGE_ALIGN(get_random_int() % range + start);
}
```

可以看出上面所说的这个差值其实就是0-0x02000000中的一个随机数，这种技术称为ASLR(Address Space Layout Randomisation)，是一种计算机安全技术，随机安排虚拟内存中堆栈空间的位置，可以有效防止黑客攻击。通过以上分析，可以画出内存分布图如下：

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/4812bc9a-43fe-4ec2-b45d-d76bc42c6933/Untitled.png)

data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==

### **malloc(0)发生了什么？**

当调用malloc(0)会发生什么，代码如下：

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

/**
 * pmem - print mem
 * @p: memory address to start printing from
 * @bytes: number of bytes to print
 *
 * Return: nothing
 */
void pmem(void *p, unsigned int bytes)
{
    unsigned char *ptr;
    unsigned int i;

    ptr = (unsigned char *)p;
    for (i = 0; i < bytes; i++)
    {
        if (i != 0)
        {
            printf(" ");
        }
        printf("%02x", *(ptr + i));
    }
    printf("\\n");
}

/**
 * main - moving the program break
 *
 * Return: EXIT_FAILURE if something failed. Otherwise EXIT_SUCCESS
 */
int main(void)
{
    void *p;
    size_t size_of_the_chunk;
    char prev_used;

    p = malloc(0);
    printf("%p\\n", p);
    pmem((char *)p - 0x10, 0x10);
    size_of_the_chunk = *((size_t *)((char *)p - 8));
    prev_used = size_of_the_chunk & 1;
    size_of_the_chunk -= prev_used;
    printf("chunk size = %li bytes\\n", size_of_the_chunk);
    return (EXIT_SUCCESS);
}
编译运行输出如下：
root@3e8650948c0c:/ubuntu# gcc test.c -o test
root@3e8650948c0c:/ubuntu# ./test
0x564ece64b260
00 00 00 00 00 00 00 00 21 00 00 00 00 00 00 00
chunk size = 32 bytes
```

可以看出malloc(0)实际使用了32个字节，其中包括我们之前说的16个字节头部，然而有时候malloc(0)可能会有不同的结果输出，也有可能会返回NULL.

```bash
man malloc
NULL may also be returned by a successful call to malloc() with a size of zero
```

### **操作环境**

```
示例代码主要在两种环境下跑过：
ubuntu 16.04
gcc (Ubuntu 7.4.0-1ubuntu1~16.04~ppa1) 7.4.0

ubuntu 18.04 docker
gcc (Ubuntu 7.4.0-1ubuntu1~18.04.1) 7.4.0

```

本文是从这一系列文章翻译并结合自己理解提炼出来的，代码都自己实践过，有时间的也可以直接阅读英文原链接

Hack The Virtual Memory: C strings & /proc  Hack the Virtual Memory: drawing the VM diagram

Hack the Virtual Memory: malloc, the heap & the program break

```
欢迎关注『高性能服务器开发』公众号。如果有任何技术或者职业方面的问题需要我提供帮助，可通过这个公众号与我取得联系，此公众号不仅分享高性能服务器开发经验和故事，同时也免费为广大技术朋友提供技术答疑和职业解惑，您有任何问题都可以在微信公众号回复关键字“职业指导”。
```