PIG-007 看雪学苑
 _2021年09月16日 18:05_
本文为看雪论坛精华文章  
看雪论坛作者ID：PIG-007  

Glibc2.29及以上版本堆的利用技巧越来越复杂，简直就是神仙打架，实在学得有点头晕。并且很多时候就算我们有了复用堆块在出题人的各种围追堵截的限制下，也可能没办法getshell，所以一直在不断开发新的利用姿势。

# **一 House of KIWI**
House OF Kiwi - 安全客，安全资讯平台 (anquanke.com)
（_https://www.anquanke.com/post/id/235598_）
## **1、原理分析**

函数调用链：assert->malloc_assert->fflush->_IO_file_jumps结构体中的__IO_file_sync

调用时的寄存器为：
![[Pasted image 20241004201944.png]]
![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

那么如果可以在不同版本下劫持对应setcontext中的赋值参数，即rdi或者rdx，就可以设置寄存器来调用我们想调用的函数。
### (1)rdi和rdx互相转换

①getkeyserv_handle+576：
```c
plaintext   #注释头   mov rdx, [rdi+8] mov [rsp+0C8h+var_C8], rax call qword ptr [rdx+20h]
```

通过rdi控制rdx，同样2.29以后不同版本都不太一样，需要再调试看看，比如2.31里就是：

```c
plaintext   #注释头   mov rdx,QWORD PTR [rdi+0x8] mov QWORD PTR [rsp],rax call QWORD PTR [rdx+0x20]
```

②svcudp_reply+26:

```c
plaintext   #注释头   mov rbp, qword ptr [rdi + 0x48]; mov rax, qword ptr [rbp + 0x18]; lea r13, [rbp + 0x10]; mov dword ptr [rbp + 0x10], 0; mov rdi, r13; call qword ptr [rax + 0x28];
```

通过rdi控制rbp实现栈迁移，然后即可任意gadget了。

其中2.31版本下还是一样的，如下：

```c
plaintext   #注释头   mov rbp,QWORD PTR [rdi+0x48] mov rax,QWORD PTR [rbp+0x18] lea r13,[rbp+0x10] mov DWORD PTR [rbp+0x10],0x0 mov rdi,r13 call QWORD PTR [rax+0x28]
```

### (2)不同劫持

这里观察寄存器就可以知道，不同版本的setcontext对应的rdi和rdx，这里就劫持哪一个。另外这里的rdi为_IO_2_1_stderr结构体，是从stderr@@GLIBC_2.2.5取值过来的，也就是data段上的数据，如果可以取得ELF基地址，直接劫持该指针为chunk地址也是可以的，这样就能劫持RDI寄存器了。
![[Pasted image 20241004202135.png]]
![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

这样如果劫持__IO_file_sync函数指针为setcontext，配合劫持的rdi和rdx就可以来调用我们想调用函数从而直接getshell或者绕过orw。

## **2、触发条件**

只要assert判断出错都可以，常用以下几个：

(1)top_chunk改小，并置pre_inuse为0，当top_chunk不足分配时会触发一个assert。(该assert函数在sysmalloc函数中被调用)
![[Pasted image 20241004202154.png]]
![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

(2)largebin chunk的size中的flag位，这个不太清楚。

(3)如果是2.29及以下，因为在tcache_put和tcacheget中还存在assert的关系，所以如果可以修改掉mp.tcache_bins，将之改大，(利用largebin attack)就会触发assert。

```c
//2.29 
tcache_put (mchunkptr chunk, size_t tc_idx) {   tcache_entry *e = (tcache_entry *) chunk2mem (chunk);   assert (tc_idx < TCACHE_MAX_BINS);     /* Mark this chunk as "in the tcache" so the test in _int_free will      detect a double free.  */   e->key = tcache;     e->next = tcache->entries[tc_idx];   tcache->entries[tc_idx] = e;   ++(tcache->counts[tc_idx]); }   //2.29 tcache_get (size_t tc_idx) {   tcache_entry *e = tcache->entries[tc_idx];   assert (tc_idx < TCACHE_MAX_BINS);   assert (tcache->entries[tc_idx] > 0);   tcache->entries[tc_idx] = e->next;   --(tcache->counts[tc_idx]);   e->key = NULL;   return (void *) e; }
```

![[Pasted image 20241004202218.png]]
![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

## **3、适用条件**

如果将exit函数替换成_exit函数,最终结束的时候,则是进行了syscall来结束,并没有机会调用_IO_cleanup,若再将__malloc_hook和__free_hook给ban了,且在输入和输出都用read和write的情况下,无法hook且无法通过IO刷新缓冲区的情况下。这时候可以借用malloc出错调用malloc_assert->fflush->_IO_file_sync函数指针。且进入的时候rdx为_IO_helper_jumps_addr，rdi为_IO_2_1_stderr_addr。

# **二 House of Husk**
house-of-husk学习笔记 (juejin.cn)
（_https://juejin.cn/post/6844904117119385614_）  

## **1、原理分析**

函数调用链:

```c
printf->vfprintf->printf_positional->__parse_one_specmb->__printf_arginfo_table(spec)
```

__parse_one_specmb 函数 会调用 __printf_arginfo_table和__printf_function_table两个函数指针中对应spec索引的函数指针printf_arginfo_size_function

▲这个spec索引指针就是格式化字符的ascii码值，比如printf("%S")，那么就是S的ascii码值。当然，这个方法的前提是得有printf系列函数，并且有格式化字符。

即调用(__printf_arginfo_table+'spec'8) 和 (printf_function_table+'spec'8)这两个函数指针。

而实际情况会先调用__printf_arginfo_table中对应的spec索引的函数指针，然后调用__printf_function_table对应spec索引函数指针。

所以如果修改了__printf_arginfo_table和__printf_function_table，则需要确保对应的spec索引对应的函数指针，要么为0，要么有效。

同时如果选择这个方法，就得需要__printf_arginfo_table和__printf_function_table均不为0才行。

```c
//2.31 vfprintf-internal.c(stdio-common)
/* Use the slow path in case any printf handler is registered.  */
if (__glibc_unlikely (__printf_function_table != NULL                       || __printf_modifier_table != NULL                       || __printf_va_arg_table != NULL))     goto do_positional;     do_positional: if (__glibc_unlikely (workstart != NULL)) {     free (workstart);     workstart = NULL; } done = printf_positional (s, format, readonly_format, ap, &ap_save,                           done, nspecs_done, lead_str_end, work_buffer,                           save_errno, grouping, thousands_sep, mode_flags);
```

```c
//2.31 vfprintf-internal.c(stdio-common)   
(void) (*__printf_arginfo_table[specs[cnt].info.spec]) (&specs[cnt].info,  specs[cnt].ndata_args, &args_type[specs[cnt].data_arg],  &args_size[specs[cnt].data_arg]);     /* Call the function.  */ function_done = __printf_function_table[(size_t) spec]     (s, &specs[nspecs_done].info, ptr);
```

即如果table不为空，则调用printf_positional函数，然后如果spec不为空，则调用对应spec索引函数。但是有时候不知道printf最终会调用哪个spec，可能隐藏在哪，所以直接把干脆_printf_arginfo_table和__printf_function_table中的值全给改成one_gadget算了。

综上，得出以下条件：

```c
A.    __printf_function_table = heap_addr     __printf_arginfo_table != 0 //其中__printf_arginfo_table和__printf_function_table可以对调 B. heap_addr+'spec'*8 = one_gadget
```
  
![[Pasted image 20241004202413.png]]
![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

在2.29下可以直接用largebin attack爆破修改两个地方，当然还是需要先泄露地址的。

## **2、触发条件**

即需要printf家族函数被调用，且其中需带上格式化字符，比如%s，%x等，用来计算spec，这个和libc版本无关，相当于只针对printf家族函数进行攻击的。

## **3、适用条件**

具有printf家族函数，并且存在spec，合适地方会调用。
# **三 House of Pig**

house of pig一个新的堆利用详解 - 安全客，安全资讯平台 (anquanke.com)

## （https://www.anquanke.com/post/id/242640）  

##   

## **1、原理分析**

###   

### (1)劫持原理

  

_IO_str_overflow函数中会连续调用malloc memcpy free三个函数。并且__IO_str_overflow函数传入的参数rdi为从_IO_list_all中获取的_IO_2_1_stderr结构体的地址。所以如果我们能改掉_IO_list_all中的值就能劫持进入该函数的参数rdi。

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

所以如上所示，即劫持成功。

  

### (2)Getshell原理

  

#### ①函数流程

A.在_IO_str_overflow函数中会先申请chunk为new_buf，然后会依据rdi的值，将rdi当作_IO_FILE结构体，从该结构体中获取_IO_buf_base当作old_buf。

B.依据old_blen 和_IO_buf_base来拷贝数据到new_buf中，然后释放掉old_buf。其中old_blen 是通过_IO_buf_end减去_IO_buf_base得到的。

```
//2.31 strops.c中的_IO_str_overflow
```

  

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

#### ②劫持所需数据

所以如果在申请的new_buf包含为_free_hook，然后我们在_IO_buf_base和_IO_buf_end这里一段数据块中将system_addr放入，那么就可以将system_addr拷贝到_free_hook中。之后释放掉old_buf，如果old_buf中的头部数据为/bin/sh\x00，那么就能直接getshell了。得到以下劫持所需数据：

```
*(_IO_list_all) = chunk_addr;
```

  

但是如何使得tcachebin[tc_idx]中的Chunk为_free_hook_addr-old_blen呢，这个就用到技术。

Largebin attack + Tcache Stashing Unlink Attack，这个技术原理比较复杂，自己看吧。

通常是只能使用callo的情况下来用的，因为如果能malloc那直接从tcache中malloc出来不就完了。

然后由于_IO_str_overflow函数中的一些检查，所以有的地方还是需要修改的：

```
fake_IO_FILE = p64(0)*2
```

##   

## **2、触发条件**

(1)Libc结构被破坏的abort函数中会调用刷新

(2)调用exit()

(3)能够从main函数返回

##   

## **3、适用条件**

程序只能通过calloc来获取chunk时。

#   

  

四

  

# **House of banana**

  

house of banana - 安全客，安全资讯平台 (anquanke.com)

（_https://www.anquanke.com/post/id/222948_）

  

main_arena劫持及link_map劫持 - 安全客，安全资讯平台 (anquanke.com)

## （_https://www.anquanke.com/post/id/211331_）  

##   

## **1、原理分析**

  

函数调用链：exit()->_dl_fini->(fini_t)array[i]

```
//2.31 glibc/elf/dl_fini.c
```

  

所以如果可以使得*array[i] = one_gadget，那么就可以一键getshell。而array[i]调用时这里就有两种套路：

###   

### (1)伪造link_map结构体

直接伪造link_map结构体，将原本指向link_map的指针指向我们伪造的link_map，然后伪造其中数据，绕过检查，最后调用array[i]。这里通常利用largebin attack来将堆地址写到_rtld_global这个结构体指针中。  

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

link_map的布局通常如下：

```
#largebin attack's chunk
```

  

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

### (2)修改link_map结构体数据

修改对应link_map结构体中的数据，绕过检查，最终调用array[i]。这里就通常需要利用任意申请来申请到该结构体，然后修改其中的值，因为当调用array[i]时，传入的实际上是link_map中的某个地址，即rdx为link_map+0x30，这个不同版本好像不太一样，2.31及以上为link_map+0x38。

主要伪造以下数据：

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

这个方法常用来打ORW，因为可以我们可以直接将ROP链布置在link_map中。然而因为版本间的关系，所以数据也有点不同，实际布局：

####   

#### **2.31**

  

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

```
//docker 2.31 gadget
```

  

#### **2.29**

  

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

```
//docker 2.29 gadget
```

  

这里需要注意的是由于ld动态连接加载的事情，所以就算是同一个版本中的link_map相对于libc基地址在不同机器中也有可能是不同的，需要爆破第4，5两位，一个字节。

题外话：适用到ld动态链接库的话，如果直接patchelf的话，很可能出错的，原因未知。推荐还是用docker:

PIG-007/pwnDockerAll (github.com)

## （_https://github.com/PIG-007/pwnDockerAll_）  

  

## **2、触发条件**

(1)调用exit()

(2)能够从main函数返回

##   

## **3、适用条件**

ban掉了很多东西的时候。但是这个需要泄露地址才行的，另外由于可能需要爆破一个字节，所以如果还涉及其他的爆破就得慎重考虑一下了，别到时候爆得黄花菜都凉了。

  

  

  

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E) 

  

**看雪ID：PIG-007**

https://bbs.pediy.com/user-home-904686.htm

*本文由看雪论坛 PIG-007 原创，转载请注明来自看雪社区

  

  

[![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458389210&idx=2&sn=fb4499fcfca121f55ac826f219d1f82d&chksm=b18f3a5086f8b34692c595152584cdc648800f5e0deb782a474ace670aec74ffd2af2e754c88&scene=21#wechat_redirect)  
  

[![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)](https://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458387399&idx=2&sn=38495add2a3a3677b2c436581c07e432&scene=21#wechat_redirect)

  

**#** **往期推荐**

1. [新人PWN堆Heap总结off-by-null专场](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458392540&idx=1&sn=75525c0ee1d799edbf118b4e19b81e3a&chksm=b18f275686f8ae40bacfec040a64b645227ce1f5554cdc5e95c337b63573d1ec67a57bebec01&scene=21#wechat_redirect)  

2. [CVE-2012-3569 VMware OVF Tool格式化字符串漏洞分析](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458392532&idx=1&sn=009b3954d88c8732df14e2d442e6cf04&chksm=b18f275e86f8ae48ec217fc9b28c98cdcfa94822de52fd359c75d81de835ce23605f46fcdf79&scene=21#wechat_redirect)

3.[DASCTF八月挑战赛 re](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458392508&idx=1&sn=e60b11a27f9ef2c9b55189c8cca6dbd8&chksm=b18f273686f8ae20ba53ee897f8ae9069f2fe80eb0853184b4b0b9310596b9be0dfb81967201&scene=21#wechat_redirect)

4. [对使用系统调用进行收发包的frida hook抓包](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458392502&idx=1&sn=d3c7746f85f44c6fa3731ad54757ccc7&chksm=b18f273c86f8ae2ab110c65fb3287c3a8414c56073e517be2c95028dc843ee4ba514cb0db411&scene=21#wechat_redirect)

5. [恶意样本分析——XOR加密与进程注入](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458392501&idx=2&sn=c11111da88b6e3064eb56adcccab2c72&chksm=b18f273f86f8ae293cddae3c27f37c692a1758f9947b9fc1c99de5c3f87f2599b4db3d152714&scene=21#wechat_redirect)

6. [超级长的IE调试总结：CVE-2013-1347 IE CGenericElement UAF漏洞分析](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458392497&idx=1&sn=0ac9a4380f0847da06e1d35dc102d02b&chksm=b18f273b86f8ae2d6d97bc7cf4594365d88f787edc07a8fd8f3a2e24f7c488e7ed55b0b4d280&scene=21#wechat_redirect)

  

  

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

公众号ID：ikanxue  

官方微博：看雪安全

商务合作：wsc@kanxue.com

  

  

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**球分享**

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**球点赞**

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**球在看**

  

  

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

点击“阅读原文”，了解更多！

Read more

Reads 3089

​

Comment

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/1UG7KPNHN8EGLfh77kFmnicd9WOic2ibvhCibFdB4bL4srJCgo2wnvdoXLxpIvAkfCmmcptXZB0qKWMoIP8iaibYN2FA/300?wx_fmt=png&wxfrom=18)

看雪学苑

1114

Comment

Comment

**Comment**

暂无留言