# 

原创 Linux内核远航者 Linux内核远航者

 _2021年12月09日 08:08_

# 1.开场白

环境：

处理器架构：arm64 

内核源码：linux-5.10.50 

ubuntu版本：20.04.1 

代码阅读工具：vim+ctags+cscope

  

  

Linux内核由于存在page cache, 一般修改的文件数据并不会马上同步到磁盘，会缓存在内存的page cache中，我们把这种和磁盘数据不一致的页称为脏页，脏页会在合适的时机同步到磁盘。为了回写page cache中的脏页，需要标记页为脏。

  

脏页跟踪是指内核如何在合适的时机记录文件页为脏，以便内核在进行脏页回写时，知道将哪些页面回写到磁盘。匿名页不需要跟踪脏页，因为不需要同步到磁盘；私有文件页也不需要跟踪脏页，因为映射的时候，可写页会映射为只读，写访问会发生写时复制，转变为匿名页；所以只有共享的文件页需要跟踪脏页。跟踪有两个层面：一个是页表项记录，一个是页描述符记录。

  

访问文件页有两种方式：一种是通过mmap映射文件，一种是通过文件系统的write接口操作文件，本文将对这两种方式进行讲解。在Linux内核中，因为跟踪脏页会涉及到文件回写、缺页异常、反向映射等技术，所以本文也重点讲解在Linux内核中如何跟踪脏页。

# 2.mmap映射的文件页

基本过程如下：

1）通过mmap映射共享文件。

2）第一次访问文件页时，发生缺页后读文件页到page cache, 如果是写访问则设置相应进程的页表项为脏、可写。

3）脏页回写时，会通过反向映射机制，查找映射这个页的每一个vma, 设置相应进程的页表项为只读，清脏标记。

4）假如第二次写访问这个文件页时，脏页的处理有两种情况：

- page cache中的文件页还未回写到磁盘（3步骤之前）， 此刻，这个文件页依然是脏页。因为相应进程的页表项为脏、可写，所以可以直接写这个页。
    
- page cache中的文件页已经回写到磁盘（3步骤之后）， 此刻，这个文件页不再是脏页。因为页表项为只读，所以写访问会发生写时复制缺页异常，异常处理中将处理共享文件页映射，重新将相应进程的页表项为设置为脏、可写。
    

  

分析如下：

## 2.1 第一次写访问文件页时

如果是mmap映射文件页，在没有填充页表情况下，写访问会发生转换表错误类型的缺页异常。

//mm/memory.c 

handle_pte_fault

    ->do_fault 

        ->do_shared_fault 

            ->__do_fault  //读文件页到page cache

                ->do_page_mkwrite 

                    ->vmf->vma->vm_ops->page_mkwrite() 

                        ->filemap_page_mkwrite, //对于ext2

                                ->set_page_dirty(page) 

                                    ->__set_page_dirty_buffers  
                                        ->__set_page_dirty//page cache中标记页为脏 

                                            ->TestSetPageDirty(page) //设置页描述符脏标记

                ->finish_fault  //设置页表项

                    ->alloc_set_pte

                        ->if (write) 

                            entry = maybe_mkwrite(pte_mkdirty(entry), vma) //设置页表项脏、可写

## 2.2 脏页回写时

//mm/page-writeback.c 

write_cache_pages

->**clear_page_dirty_for_io**(page) //对于回写的每一个页

    ->page_mkclean(page) //清脏标记  mm/rmap.c 

        ->page_mkclean_one //**反向映射**查找这个页的每个vma，调用清脏标记和写保护处理

                ->entry = **pte_wrprotect**(entry);     //写保护处理，设置只读  
                    entry = **pte_mkclean**(entry); //清脏标记 set_pte_at(vma->vm_mm, address, pte, entry) //设置到页表项中

    ->**TestClearPageDirty**(page) //清页描述符脏标记

## 2.3 第二次写访问文件页时

1）脏页还没有回写时（**确切的说是调用clear_page_dirty_for_io之前），**页描述符已经设置了脏标记，页表项已经设置了脏标记、可写。

这时可以直接写访问文件页，不会发生缺页。

2）脏页已经回写时（**确切的说是调用clear_page_dirty_for_io之后**），页描述符已经清除了脏标记，页表项已经清除了脏标记，且只读。

这时写访问文件页会发生写时复制缺页异常（访问权限错误缺页）。

调用链如下：

//mm/memory.c 

handle_pte_fault 

->if (vmf->flags & FAULT_FLAG_WRITE) { //vma可写

            if (!pte_write(entry)) //页表项没有可写属性 return do_wp_page(vmf) //写时复制缺页异常处理

                    do_wp_page 

                            ->} else if (unlikely((vma->vm_flags & (VM_WRITE|VM_SHARED)) == (VM_WRITE|VM_SHARED))) { //是共享可写的文件映射vma 

                                        return   wp_page_shared(vmf);

                                        ->do_page_mkwrite 

                                            ->vmf->vma->vm_ops->page_mkwrite()

                                                ->filemap_page_mkwrite, //对于ext2                                                              ->set_page_dirty(page)

                                                               ->__set_page_dirty_buffers  //page cache中标记页为脏 

                                                                     ->TestSetPageDirty(page) //设置页描述符脏标记

                                                  ->finish_mkwrite_fault

                                                        ->wp_page_reuse

                                                                    ->entry = maybe_mkwrite(pte_mkdirty(entry), vma) //重新设置页表项脏、可写

## 2.4 再次写访问

重复上面步骤。

# 3.write接口操作的文件页

由于通过write接口访问文件页时，会读取文件页到page cache,不会映射到任何进程地址空间，所有这种方式跟踪脏页是**通过设置/清除页描述符脏标记**来实现。

## 3.1 第一次写访问文件页时

会首先读文件页到page cache，然后将用户空间写缓冲区数据写到page cache，调用链如下：

ext2_file_write_iter //fs/ext2/file.c 

->generic_file_write_iter //mm/filemap.c 

    ->__generic_file_write_iter 

        ->generic_perform_write 

            ->a_ops->write_begin() //写之前处理 分配page cache页                                               ->iov_iter_copy_from_user_atomic //户空间写缓冲区数据写到page cache页       -> a_ops->write_end() //写之后处理

                    ->block_write_end 

                            ->__block_commit_write

                                    ->mark_buffer_dirty

                                            if (!TestSetPageDirty(page)) {  //设置页描述符脏标记                                                        ->__set_page_dirty  //设置页为脏（设置页描述符脏标记）

## 3.2 脏页回写时

write_cache_pages  //mm/page-writeback.c 

->clear_page_dirty_for_io 

        ->TestClearPageDirty(page) //清除页描述符脏标记

## 3.3 第二次写访问文件页时

脏页回写之前，页描述符脏标志位依然被置位，等待回写, 不需要设置页描述符脏标志位。

脏页回写之后，页描述符脏标志位是清零的，文件写页调用链会设置页描述符脏标志位。

# 4.总结

1）对于mmap映射的共享文件页，因为这个文件页可能会被多个进程共享到多个vma中，所以通过页表项的脏标志位来跟踪脏页：第一次写访问发生缺页异常会读文件页到page cache中并设置进程的页表项的脏标志，回写之前（clear_page_dirty_for_io完成之前），页表项的脏标志是置位的，回写的时候（clear_page_dirty_for_io的调用）会通过反向映射机制将所有映射这个页的页表项的脏标志位清零并设置只读权限，回写之后（clear_page_dirty_for_io完成之后），再次的写访问会发生写时复制缺页异常，再次设置页表项的脏标志位，如此重复，从而跟踪了脏页。

2）对于直接通过write接口访问的文件页，因为这个文件页只会被读取到page cache中，并没有映射到任何进程地址空间，进程写访问是通过copy_from_user的方式，所以通过页描述符记录脏页。回写之前（clear_page_dirty_for_io完成之前），写文件的时候通过文件系统的写文件的调用链会设置页描述符脏标志位，回写的时候（clear_page_dirty_for_io的调用）会清除页描述符脏标志位，回写之后（clear_page_dirty_for_io完成之后），再次通过write接口写访问时，再次通过文件系统的写文件的调用链会再次设置页描述符脏标志位，如此重复，从而跟踪了脏页。

  

![](https://mmbiz.qlogo.cn/mmbiz_jpg/0yLh80tof6iaI7kU6IF9lzBibbRPG0CtLdoKrW3aq73t2gVBV49Qp8jamjZbjGpn5p0v5fgY9XkSlhI9PAroicbag/0?wx_fmt=jpeg)

Linux内核远航者

![赞赏二维码](https://mp.weixin.qq.com/s?__biz=MzAwMDQ1MjAzOQ==&mid=2247484423&idx=1&sn=d11c389576a04da87e15c7ce2ea90492&chksm=9ae9f0afad9e79b954d7fffe71b20f9f7b4b0c447e4b10b8229ebd2b8dc35631e8e3fb271759&mpshare=1&scene=24&srcid=1209MB0kL5dTFjGhCgueX1tD&sharer_sharetime=1639047496218&sharer_shareid=5fb9813bfe9ffc983435bfc8d8c5e9ca&key=daf9bdc5abc4e8d0bf801bf3d1e152dff223eef3cb801223c0a680e5ec7794d027166b97fcbbb99c5bf36e8ae38b0adc7d98b60c5660cdd195dd5e0bf036b191e19f4ed9cc831c14ad925c5bb16e73aeaa1345c9f4a028faad5d6541e77660901b48fd4f82accef502a0bd7e19777ff429666a909ae787c55cfec522181d3c05&ascene=0&uin=MTEwNTU1MjgwMw%3D%3D&devicetype=Windows+11+x64&version=63090b19&lang=zh_CN&countrycode=CN&exportkey=n_ChQIAhIQ%2FL56j7%2BfTGyY5uxijsr%2FpRLmAQIE97dBBAEAAAAAAEM1JbWFs6IAAAAOpnltbLcz9gKNyK89dVj0G0LOfU0pvgkzmzEdeN9%2FUo%2FdpqmAXAve6Enl7krIL%2FsIKFq1cxynMJ9RopOV%2BiajQaqwgPJOTfRip7w4BtGpRijmOJ2YPkTSwcsslzf06Z%2FUjT3evEr%2Fca1Mb0YXDvXNJmf55E%2FBwe67LLuHTJgjyujLXwgtm%2F9%2Fdlpse9GFEGZ4afy7OQadbc%2FPGaLI2fCw5F2mieVXwnNWQU%2FrsncaSwJaJhf5t6UQ9ovDtCrbB1m6bbbu%2B1ES%2ByIvUTY8MLpT&acctmode=0&pass_ticket=cyvH2aY6GI8OsVDJR268zPVqHImTkd90X5xjaDd9vtb5WC6Ic%2FJjMdq6BIYaBtER&wx_header=1&fasttmpl_type=0&fasttmpl_fullversion=7351805-zh_CN-zip&fasttmpl_flag=1)喜欢作者

3人喜欢

![](http://wx.qlogo.cn/mmopen/DmTSLTdleeuBWaicSjSJZASKmO33eoVTRHMDuhKWA2DrVmAIqm82u6Xf1nUhtxXJdFwHEYSicPh8ROXvpD4uBkHOVKulVPe4EO/64)![](http://wx.qlogo.cn/mmopen/dibSQM70XVQLHhj5HqCEqD7Chk169cM3oFW2Xz74Vo5aTAojdu00LXezdSSic5pNebjT2iamFChHrXIV5T596M3DFgYf6a7kICU/64)![](http://wx.qlogo.cn/mmopen/PiajxSqBRaEKFgPiabvpQVapoibDLdzNIKxbodeTsjBOJyMTbt4kcczXZpA6gib4d0IqLCZtufpPePD4dusXAPCXic08p3mU0WBKekLvpWE7Dy3Op4wZGibrwfRcFDtET1yIQ8/64)

阅读 1429

​