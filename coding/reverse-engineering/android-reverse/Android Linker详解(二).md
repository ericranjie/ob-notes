# 

r0ysue 看雪学苑

 _2021年11月21日 18:05_

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/kR3MmOkZ9r7dibrdGMbvb6FdqicOAriagz3L2E60m2u0qRRxzhkzjZMAHAOZy27LmZhllOWxcK4us4GGlMg4wCibYA/640?wx_fmt=jpeg&wxfrom=13&tp=wxpic)  

看雪论坛作者ID：r0ysue

  

  

1

  

**简介**

  

接上篇Linker源码详解(一)，本文继续来分析Linker的链接过程。为了更好的理解Unidbg的原理，我们需要了解很多细节。虽然一个模拟二进制执行框架的弊端很多，但也是未来二进制分析的一个很好的思路。

上篇文章我们讲解了Linker的装载，将So文件按PT_LOAD段的指示来将So加载到内存，那么我们这篇文章就来分析一下加载完之后又干了什么呢？

##   

  

2

  

**So的链接**  

  

_http://androidxref.com/4.4.4_r1/xref/bionic/linker/linker.cpp#702_

```
`static soinfo``*` `load_library(const char``*` `name) {`

    `/``/``...`

    `ElfReader elf_reader(name, fd);`

    `if` `(!elf_reader.Load()) {`

        `return` `NULL;`

    `}`

    `const char``*` `bname` `=` `strrchr(name,` `'/'``);`

    `soinfo``*` `si` `=` `soinfo_alloc(bname ? bname` `+` `1` `: name);`

    `if` `(si` `=``=` `NULL) {`

        `return` `NULL;`

    `}`

    `si``-``>base` `=` `elf_reader.load_start();`

    `si``-``>size` `=` `elf_reader.load_size();`

    `si``-``>load_bias` `=` `elf_reader.load_bias();`

    `si``-``>flags` `=` `0``;`

    `si``-``>entry` `=` `0``;`

    `si``-``>dynamic` `=` `NULL;`

    `si``-``>phnum` `=` `elf_reader.phdr_count();`

    `si``-``>phdr` `=` `elf_reader.loaded_phdr();`

    `return` `si;`

`}`
```

  

上篇我们进入了elf_reader.Load()函数，阅读了Linker的装载源码，当装载结束后，对soinfo结构体进行赋值(So文件的头信息/装载的结果)，并插入到链表，接着我们回到上层函数继续看。

  

_http://androidxref.com/4.4.4_r1/xref/bionic/linker/linker.cpp#751_

```
`static soinfo``*` `find_library_internal(const char``*` `name) {`

  `/``/``...`

  `si` `=` `load_library(name);`

  `if` `(si` `=``=` `NULL) {`

    `return` `NULL;`

  `}`

  `/``/` `At this point we know that whatever` `is` `loaded @ base` `is` `a valid ELF`

  `/``/` `shared library whose segments are properly mapped` `in``.`

  `TRACE(``"[ init_library base=0x%08x sz=0x%08x name='%s' ]"``,`

        `si``-``>base, si``-``>size, si``-``>name);`

  `if` `(!soinfo_link_image(si)) {`

    `munmap(reinterpret_cast<void``*``>(si``-``>base), si``-``>size);`

    `soinfo_free(si);`

    `return` `NULL;`

  `}`

  `return` `si;`

`}`
```

  

我们从上面这个函数中看到，当调用了load_library函数之后，又调用了soinfo_link_image这个函数。这个函数也就是我们今天分析的一个主要入口--链接。

下面的这个函数很长，我给大家把不相关的代码去掉，大家先通过注释来看一遍这个函数在干什么。

  

_http://androidxref.com/4.4.4_r1/xref/bionic/linker/linker.cpp#1303_

```
`static` `bool` `soinfo_link_image(soinfo``*` `si) {`

    `/``/``拿到地址、段表指针、段表数`

    `Elf32_Addr base` `=` `si``-``>load_bias;`

    `const Elf32_Phdr` `*``phdr` `=` `si``-``>phdr;`

    `int` `phnum` `=` `si``-``>phnum;`

    `/``/``...`

    `size_t dynamic_count;`

    `Elf32_Word dynamic_flags;`

    `/``/``这个函数很简单，就是遍历段表，找到类型为PT_DYNAMIC的段`

    `phdr_table_get_dynamic_section(phdr, phnum, base, &si``-``>dynamic,`

                                   `&dynamic_count, &dynamic_flags);`

    `if` `(si``-``>dynamic` `=``=` `NULL) {`

        `if` `(!relocating_linker) {`

            `DL_ERR(``"missing PT_DYNAMIC in \"%s\""``, si``-``>name);`

        `}`

        `return` `false;`

    `}` 

`#ifdef ANDROID_ARM_LINKER`

    `/``/``异常相关，有兴趣的同学可以看看`

    `(void) phdr_table_get_arm_exidx(phdr, phnum, base,`

                                    `&si``-``>ARM_exidx, &si``-``>ARM_exidx_count);`

`#endif`

    `/``/``上面我们解析到了Dynamic段的地址跟数量，下面就开始遍历Dynamic信息`

    `uint32_t needed_count` `=` `0``;`

    `/``/``DT_NULL表示结束`

    `for` `(Elf32_Dyn``*` `d` `=` `si``-``>dynamic; d``-``>d_tag !``=` `DT_NULL;` `+``+``d) {`

        `DEBUG(``"d = %p, d[0](tag) = 0x%08x d[1](val) = 0x%08x"``, d, d``-``>d_tag, d``-``>d_un.d_val);`

        `switch(d``-``>d_tag){`

        `case DT_HASH:`

            `/``/``哈希表`

            `si``-``>nbucket` `=` `((unsigned` `*``) (base` `+` `d``-``>d_un.d_ptr))[``0``];`

            `si``-``>nchain` `=` `((unsigned` `*``) (base` `+` `d``-``>d_un.d_ptr))[``1``];`

            `si``-``>bucket` `=` `(unsigned` `*``) (base` `+` `d``-``>d_un.d_ptr` `+` `8``);`

            `si``-``>chain` `=` `(unsigned` `*``) (base` `+` `d``-``>d_un.d_ptr` `+` `8` `+` `si``-``>nbucket` `*` `4``);`

            `break``;`

        `case DT_STRTAB:`

            `/``/``字符串表`

            `si``-``>strtab` `=` `(const char` `*``) (base` `+` `d``-``>d_un.d_ptr);`

            `break``;`

        `case DT_SYMTAB:`

            `/``/``符号表`

            `si``-``>symtab` `=` `(Elf32_Sym` `*``) (base` `+` `d``-``>d_un.d_ptr);`

            `break``;`

        `case DT_PLTREL:`

            `/``/``未处理`

            `if` `(d``-``>d_un.d_val !``=` `DT_REL) {`

                `DL_ERR(``"unsupported DT_RELA in \"%s\""``, si``-``>name);`

                `return` `false;`

            `}`

            `break``;`

        `case DT_JMPREL:`

            `/``/``PLT重定位表`

            `si``-``>plt_rel` `=` `(Elf32_Rel``*``) (base` `+` `d``-``>d_un.d_ptr);`

            `break``;`

        `case DT_PLTRELSZ:`

            `/``/``PLT重定位表大小`

            `si``-``>plt_rel_count` `=` `d``-``>d_un.d_val` `/` `sizeof(Elf32_Rel);`

            `break``;`

        `case DT_REL:`

            `/``/``重定位表`

            `si``-``>rel` `=` `(Elf32_Rel``*``) (base` `+` `d``-``>d_un.d_ptr);`

            `break``;`

        `case DT_RELSZ:`

            `/``/``重定位表大小`

            `si``-``>rel_count` `=` `d``-``>d_un.d_val` `/` `sizeof(Elf32_Rel);`

            `break``;`

        `case DT_PLTGOT:`

            `/``/``GOT全局偏移表，跟PLT延时绑定相关，此处未处理，在Unidbg中也没有处理此项`

            `si``-``>plt_got` `=` `(unsigned` `*``)(base` `+` `d``-``>d_un.d_ptr);`

            `break``;`

        `case DT_DEBUG:`

            `/``/``调试相关, Unidbg未处理，不必理会`

            `if` `((dynamic_flags & PF_W) !``=` `0``) {`

                `d``-``>d_un.d_val` `=` `(``int``) &_r_debug;`

            `}`

            `break``;`

         `case DT_RELA:`

            `/``/``RELA表跟REL表在Unidbg中的处理方案是相同的，这两个值有哪个就用哪个，RELA只是比REL表多了一个adden常量`

            `DL_ERR(``"unsupported DT_RELA in \"%s\""``, si``-``>name);`

            `return` `false;`

        `case DT_INIT:`

            `/``/``初始化函数`

            `si``-``>init_func` `=` `reinterpret_cast<linker_function_t>(base` `+` `d``-``>d_un.d_ptr);`

            `DEBUG(``"%s constructors (DT_INIT) found at %p"``, si``-``>name, si``-``>init_func);`

            `break``;`

        `case DT_FINI:`

            `/``/``析构函数`

            `si``-``>fini_func` `=` `reinterpret_cast<linker_function_t>(base` `+` `d``-``>d_un.d_ptr);`

            `DEBUG(``"%s destructors (DT_FINI) found at %p"``, si``-``>name, si``-``>fini_func);`

            `break``;`

        `case DT_INIT_ARRAY:`

            `/``/``init.array 初始化函数列表，后面我们会看到这些初始化函数的调用顺序`

            `si``-``>init_array` `=` `reinterpret_cast<linker_function_t``*``>(base` `+` `d``-``>d_un.d_ptr);`

            `DEBUG(``"%s constructors (DT_INIT_ARRAY) found at %p"``, si``-``>name, si``-``>init_array);`

            `break``;`

        `case DT_INIT_ARRAYSZ:`

            `/``/``init.array 大小`

            `si``-``>init_array_count` `=` `((unsigned)d``-``>d_un.d_val)` `/` `sizeof(Elf32_Addr);`

            `break``;`

        `case DT_FINI_ARRAY:`

            `/``/``析构函数列表`

            `si``-``>fini_array` `=` `reinterpret_cast<linker_function_t``*``>(base` `+` `d``-``>d_un.d_ptr);`

            `DEBUG(``"%s destructors (DT_FINI_ARRAY) found at %p"``, si``-``>name, si``-``>fini_array);`

            `break``;`

        `case DT_FINI_ARRAYSZ:`

            `/``/``fini.array 大小`

            `si``-``>fini_array_count` `=` `((unsigned)d``-``>d_un.d_val)` `/` `sizeof(Elf32_Addr);`

            `break``;`

        `case DT_PREINIT_ARRAY:`

            `/``/``也是初始化函数，但是跟init.array不同，这个段大多只出现在可执行文件中，在So中我选择了忽略`

            `si``-``>preinit_array` `=` `reinterpret_cast<linker_function_t``*``>(base` `+` `d``-``>d_un.d_ptr);`

            `DEBUG(``"%s constructors (DT_PREINIT_ARRAY) found at %p"``, si``-``>name, si``-``>preinit_array);`

            `break``;`

        `case DT_PREINIT_ARRAYSZ:`

            `/``/``preinit 列表大小`

            `si``-``>preinit_array_count` `=` `((unsigned)d``-``>d_un.d_val)` `/` `sizeof(Elf32_Addr);`

            `break``;`

        `case DT_TEXTREL:`

            `si``-``>has_text_relocations` `=` `true;`

            `break``;`

        `case DT_SYMBOLIC:`

            `si``-``>has_DT_SYMBOLIC` `=` `true;`

            `break``;`

        `case DT_NEEDED:`

            `/``/``当前So的依赖`

            `+``+``needed_count;`

            `break``;`

`#if defined DT_FLAGS`

        `/``/` `TODO: why` `is` `DT_FLAGS` `not` `defined?`

        `case DT_FLAGS:`

            `if` `(d``-``>d_un.d_val & DF_TEXTREL) {`

                `si``-``>has_text_relocations` `=` `true;`

            `}`

            `if` `(d``-``>d_un.d_val & DF_SYMBOLIC) {`

                `si``-``>has_DT_SYMBOLIC` `=` `true;`

            `}`

            `break``;`

`#endif`

        `}`

    `}`

    `/``/``... Sanity checks.`

    `/``/``至此，Dynamic段的信息就解析完毕了，其中想表达的信息也被处理后放到了soinfo中，后面直接就可以拿来用了`

    `/``/` `开辟依赖库的soinfo空间，准备处理依赖`

    `soinfo``*``*` `needed` `=` `(soinfo``*``*``) alloca((``1` `+` `needed_count)` `*` `sizeof(soinfo``*``));`

    `soinfo``*``*` `pneeded` `=` `needed;`

    `/``/``再次遍历Dynamic段`

    `for` `(Elf32_Dyn``*` `d` `=` `si``-``>dynamic; d``-``>d_tag !``=` `DT_NULL;` `+``+``d) {`

        `if` `(d``-``>d_tag` `=``=` `DT_NEEDED) {`

            `/``/``查找DT_NEEDED项`

            `const char``*` `library_name` `=` `si``-``>strtab` `+` `d``-``>d_un.d_val;`

            `DEBUG(``"%s needs %s"``, si``-``>name, library_name);`

            `/``/``进行依赖处理，跟加载so一样的路线，还是已加载直接返回，未加载进行查找加载`

            `soinfo``*` `lsi` `=` `find_library(library_name);`

            `if` `(lsi` `=``=` `NULL) {`

                `strlcpy(tmp_err_buf, linker_get_error_buffer(), sizeof(tmp_err_buf));`

                `DL_ERR(``"could not load library \"%s\" needed by \"%s\"; caused by %s"``,`

                       `library_name, si``-``>name, tmp_err_buf);`

                `return` `false;`

            `}`

            `*``pneeded``+``+` `=` `lsi;`

        `}`

    `}`

    `*``pneeded` `=` `NULL;`

    `/``/``至此依赖库也已经加载完毕`

    `/``/``处理重定位`

    `if` `(si``-``>plt_rel !``=` `NULL) {`

        `DEBUG(``"[ relocating %s plt ]"``, si``-``>name );`

        `if` `(soinfo_relocate(si, si``-``>plt_rel, si``-``>plt_rel_count, needed)) {`

            `return` `false;`

        `}`

    `}`

    `if` `(si``-``>rel !``=` `NULL) {`

        `DEBUG(``"[ relocating %s ]"``, si``-``>name );`

        `if` `(soinfo_relocate(si, si``-``>rel, si``-``>rel_count, needed)) {`

            `return` `false;`

        `}`

    `}`

    `/``/``设置soinfo的LINKED标志，表示已进行链接`

    `si``-``>flags |``=` `FLAG_LINKED;`

    `DEBUG(``"[ finished linking %s ]"``, si``-``>name);`

    `/``/``...`

    `return` `true;`

`}`
```

  

上面的函数虽然很长，但是它想表达的意思很简单，我们再来回顾下它干了什么事情：

- 解析Dynamic段信息
    
- 处理依赖
    
- 准备进行重定位
    

##   

  

3

  

**So重定位**  

  

下面我们就来分析它的soinfo_relocate函数，我们看到它调用了两次，只不过入参不同，分别是我们的重定位表和PLT重定位表。

  

_http://androidxref.com/4.4.4_r1/xref/bionic/linker/linker.cpp#848_

```
`static` `int` `soinfo_relocate(soinfo``*` `si, Elf32_Rel``*` `rel, unsigned count,`

                           `soinfo``*` `needed[])`

`{`

    `/``/``拿到符号表和字符串表，定义一些变量`

    `Elf32_Sym``*` `symtab` `=` `si``-``>symtab;`

    `const char``*` `strtab` `=` `si``-``>strtab;`

    `Elf32_Sym``*` `s;`

    `Elf32_Rel``*` `start` `=` `rel;`

    `soinfo``*` `lsi;`

    `/``/``遍历重定位表`

    `for` `(size_t idx` `=` `0``; idx < count;` `+``+``idx,` `+``+``rel) {`

        `/``/``拿到重定位类型`

        `unsigned` `type` `=` `ELF32_R_TYPE(rel``-``>r_info);`

        `/``/``拿到重定位符号`

        `unsigned sym` `=` `ELF32_R_SYM(rel``-``>r_info);`

        `/``/``计算需要重定位的地址`

        `Elf32_Addr reloc` `=` `static_cast<Elf32_Addr>(rel``-``>r_offset` `+` `si``-``>load_bias);`

        `Elf32_Addr sym_addr` `=` `0``;`

        `char``*` `sym_name` `=` `NULL;`

        `DEBUG(``"Processing '%s' relocation at index %d"``, si``-``>name, idx);`

        `if` `(``type` `=``=` `0``) {` `/``/` `R_``*``_NONE`

            `continue``;`

        `}`

        `if` `(sym !``=` `0``) {`

            `/``/``如果sym不为``0``，说明重定位需要用到符号，先来找符号，拿到符号名`

            `sym_name` `=` `(char` `*``)(strtab` `+` `symtab[sym].st_name);`

            `/``/``下面这个函数大家有兴趣的可以看一下，就是根据符号名来从依赖so中查找所需要的符号`

            `s` `=` `soinfo_do_lookup(si, sym_name, &lsi, needed);`

            `if` `(s` `=``=` `NULL) {`

                `/``/``如果没找到，就用本身So的符号`

                `s` `=` `&symtab[sym];`

                `if` `(ELF32_ST_BIND(s``-``>st_info) !``=` `STB_WEAK) {`

                    `DL_ERR(``"cannot locate symbol \"%s\" referenced by \"%s\"..."``, sym_name, si``-``>name);`

                    `return` `-``1``;`

                `}`

                `switch (``type``) {`

                    `/``/``下面是如果符号不为外部符号，就只能为以下几种类型`

`#if defined(ANDROID_ARM_LINKER)`

                `case R_ARM_JUMP_SLOT:`

                `case R_ARM_GLOB_DAT:`

                `case R_ARM_ABS32:`

                `case R_ARM_RELATIVE:`    `/``*` `Don't care.` `*``/`

`#endif /* ANDROID_*_LINKER */`

                    `/``*` `sym_addr was initialized to be zero above` `or` `relocation`

                       `code below does` `not` `care about value of sym_addr.`

                       `No need to do anything.`  `*``/`

                    `break``;`

`#if defined(ANDROID_ARM_LINKER)`

                `case R_ARM_COPY:`

                    `/``*` `Fall through.  Can't really copy` `if` `weak symbol` `is`

                       `not` `found` `in` `run``-``time.`  `*``/`

`#endif /* ANDROID_ARM_LINKER */`

                `default:`

                    `DL_ERR(``"unknown weak reloc type %d @ %p (%d)"``,`

                                 `type``, rel, (``int``) (rel` `-` `start));`

                    `return` `-``1``;`

                `}`

            `}` `else` `{`

                `/``/``如果我们找到了外部符号，取到外部符号的地址`

                `sym_addr` `=` `static_cast<Elf32_Addr>(s``-``>st_value` `+` `lsi``-``>load_bias);`

            `}`

            `count_relocation(kRelocSymbol);`

        `}` `else` `{`

            `/``/``如果sym为``0``，就说明当前重定位用不到符号`

            `s` `=` `NULL;`

        `}`

        `/``/``下面根据重定位类型来处理重定位`

        `switch(``type``){`

`#if defined(ANDROID_ARM_LINKER)`

        `case R_ARM_JUMP_SLOT:`

            `count_relocation(kRelocAbsolute);`

            `MARK(rel``-``>r_offset);`

            `TRACE_TYPE(RELO,` `"RELO JMP_SLOT %08x <- %08x %s"``, reloc, sym_addr, sym_name);`

            `/``/``直接将需要重定位的地方，写入获取到的符号地址`

            `*``reinterpret_cast<Elf32_Addr``*``>(reloc)` `=` `sym_addr;`

            `break``;`

        `case R_ARM_GLOB_DAT:`

            `count_relocation(kRelocAbsolute);`

            `MARK(rel``-``>r_offset);`

            `TRACE_TYPE(RELO,` `"RELO GLOB_DAT %08x <- %08x %s"``, reloc, sym_addr, sym_name);`

            `/``/``直接将需要重定位的地方，写入获取到的符号地址，与R_ARM_JUMP_SLOT相同`

            `*``reinterpret_cast<Elf32_Addr``*``>(reloc)` `=` `sym_addr;`

            `break``;`

        `case R_ARM_ABS32:`

            `count_relocation(kRelocAbsolute);`

            `MARK(rel``-``>r_offset);`

            `TRACE_TYPE(RELO,` `"RELO ABS %08x <- %08x %s"``, reloc, sym_addr, sym_name);`

            `/``/``先读出需要重定位地方的数据，将其和符号地址相加，写入需要重定位的地方`

            `*``reinterpret_cast<Elf32_Addr``*``>(reloc)` `+``=` `sym_addr;`

            `break``;`

        `case R_ARM_REL32:`

            `count_relocation(kRelocRelative);`

            `MARK(rel``-``>r_offset);`

            `TRACE_TYPE(RELO,` `"RELO REL32 %08x <- %08x - %08x %s"``,`

                       `reloc, sym_addr, rel``-``>r_offset, sym_name);`

            `/``/``先读出需要重定位地方的数据，将其和符号地址相加，再与重定位的地址相减，重定位的写入需要重定位的地方。此处Unidbg并未处理，也可忽略，应该是用不到的`

            `*``reinterpret_cast<Elf32_Addr``*``>(reloc)` `+``=` `sym_addr` `-` `rel``-``>r_offset;`

            `break``;`

`#endif /* ANDROID_*_LINKER */`

`#if defined(ANDROID_ARM_LINKER)`

        `case R_ARM_RELATIVE:`

`#endif /* ANDROID_*_LINKER */`

            `count_relocation(kRelocRelative);`

            `MARK(rel``-``>r_offset);`

            `if` `(sym) {`

                `DL_ERR(``"odd RELATIVE form..."``);`

                `return` `-``1``;`

            `}`

            `TRACE_TYPE(RELO,` `"RELO RELATIVE %08x <- +%08x"``, reloc, si``-``>base);`

            `/``/``先读出需要重定位地方的数据，将其和So的基址相加，写入需要重定位的地方`

            `*``reinterpret_cast<Elf32_Addr``*``>(reloc)` `+``=` `si``-``>base;`

            `break``;`

`#ifdef ANDROID_ARM_LINKER`

        `case R_ARM_COPY:`

            `/``/``.. 进行了一些错误处理`

            `break``;`

`#endif /* ANDROID_ARM_LINKER */`

        `default:`

            `DL_ERR(``"unknown reloc type %d @ %p (%d)"``,`

                   `type``, rel, (``int``) (rel` `-` `start));`

            `return` `-``1``;`

        `}`

    `}`

    `return` `0``;`

`}`
```

  

上面这个函数就是在处理重定位相关的信息了，我们看到从Dynamic段中拿到的跟重定位相关的表，会经过这个函数来处理，将So本身的地址引用进行重定位，使其可以正常运行。其实在32位So中，需要处理的重定位类型并不是很多，就4种类型需要处理，而且还有两种处理方式相同。

现在So就重定位完成了，现在So已经就可以跑起来了，下面我们就来看看从Dynamic段中拿到的各种初始化函数是怎么处理的，还记得吧。

我们回到do_dlopen函数。

  

_http://androidxref.com/4.4.4_r1/xref/bionic/linker/linker.cpp#823_

```
`soinfo``*` `do_dlopen(const char``*` `name,` `int` `flags) {`

  `if` `((flags & ~(RTLD_NOW|RTLD_LAZY|RTLD_LOCAL|RTLD_GLOBAL)) !``=` `0``) {`

    `DL_ERR(``"invalid flags to dlopen: %x"``, flags);`

    `return` `NULL;`

  `}`

  `set_soinfo_pool_protection(PROT_READ | PROT_WRITE);`

  `soinfo``*` `si` `=` `find_library(name);`

  `if` `(si !``=` `NULL) {`

    `si``-``>CallConstructors();`

  `}`

  `set_soinfo_pool_protection(PROT_READ);`

  `return` `si;`

`}`
```

  

此时我们的find_library函数已经处理完了，So已经被装载且链接过了，最后一步它调用了soinfo的CallConstructors函数，我们来看看这个函数处理了什么。

  
_http://androidxref.com/4.4.4_r1/xref/bionic/linker/linker.cpp#1192_

```
`void soinfo::CallConstructors() {`

  `if` `(constructors_called) {`

    `return``;`

  `}`

  `constructors_called` `=` `true;`

  `if` `((flags & FLAG_EXE)` `=``=` `0` `&& preinit_array !``=` `NULL) {`

    `/``/` `The GNU dynamic linker silently ignores these, but we warn the developer.`

    `PRINT``(``"\"%s\": ignoring %d-entry DT_PREINIT_ARRAY in shared library!"``,`

          `name, preinit_array_count);`

  `}`

  `/``/``如果Dynamic段不为空，先处理依赖库的初始化`

  `if` `(dynamic !``=` `NULL) {`

    `for` `(Elf32_Dyn``*` `d` `=` `dynamic; d``-``>d_tag !``=` `DT_NULL;` `+``+``d) {`

      `if` `(d``-``>d_tag` `=``=` `DT_NEEDED) {`

        `const char``*` `library_name` `=` `strtab` `+` `d``-``>d_un.d_val;`

        `TRACE(``"\"%s\": calling constructors in DT_NEEDED \"%s\""``, name, library_name);`

        `find_loaded_library(library_name)``-``>CallConstructors();`

      `}`

    `}`

  `}`

  `TRACE(``"\"%s\": calling constructors"``, name);`

  `/``/``我们来看下面一句英文注释，非常重要。他说如果DT_INIT和DT_INIT_ARRAY都存在，DT_INIT应该在DT_INIT_ARRAY之前被调用`

  `/``/` `DT_INIT should be called before DT_INIT_ARRAY` `if` `both are present.`

  `/``/``下面就是在调用两者，CallArray只是在循环调用CallFunction，我们看一下CallFunction`

  `CallFunction(``"DT_INIT"``, init_func);`

  `CallArray(``"DT_INIT_ARRAY"``, init_array, init_array_count, false);`

`}`
```

  

_http://androidxref.com/4.4.4_r1/xref/bionic/linker/linker.cpp#1172_

```
`void soinfo::CallFunction(const char``*` `function_name UNUSED, linker_function_t function) {`

  `if` `(function` `=``=` `NULL || reinterpret_cast<uintptr_t>(function)` `=``=``static_cast<uintptr_t>(``-``1``)) {`

    `return``;`

  `}`

  `TRACE(``"[ Calling %s @ %p for '%s' ]"``, function_name, function, name);`

  `/``/``在这里被调用了，其他没啥好说的`

  `function();`

  `TRACE(``"[ Done calling %s @ %p for '%s' ]"``, function_name, function, name);`

  `/``/` `The function may have called dlopen(``3``)` `or` `dlclose(``3``), so we need to ensure our data structures`

  `/``/` `are still writable. This happens with our debug malloc (see http:``/``/``b``/``7941716``).`

  `set_soinfo_pool_protection(PROT_READ | PROT_WRITE);`

`}`
```

  

至此，Linker就分析结束了。

##   

##   

4

  

**总结**  

  

我们在最后说一个Unidbg细节的bug，但是现在已经被修复了，就是作为一个扩展吧。我们来看下面一段Unidbg加载So的代码。

```
if (elfFile.file_type == ElfFile.FT_DYN) { // not executable
```

  

如果我们细心的阅读Linker的源码，就会发现Unidbg这里处理的是不恰当的。在本文的最后，我们看到了初始化函数的调用，是DT_INIT函数先被执行，后面再处理DT_INIT_ARRAY，而Unidbg这里就是将他们都添加到一个List，再一起调用。

  

这样就会产生一个问题，在某些加壳的So中，它的DT_INIT_ARRAY是在DT_INIT函数执行之后，才会有值的(进行修复)，所以按照Unidbg这个写法就无法执行INIT_ARRAY或部分INIT_ARRAY无法执行。处理方法也很简单，注释在上面了，只需要让DT_INIT先执行就可以了。

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E) 

  

**看雪ID：r0ysue**

https://bbs.pediy.com/user-home-799845.htm

*本文由看雪论坛 r0ysue 原创，转载请注明来自看雪社区

  

  

[![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458404582&idx=1&sn=3e6e984a236b1e26baaafbf350fd1cb7&chksm=b18f766c86f8ff7a4b8c367aa1e0b04aceac47b283c0fb3d38a448521adb6974847b9fa597a6&scene=21#wechat_redirect)

  

[![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)](https://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458387399&idx=2&sn=38495add2a3a3677b2c436581c07e432&scene=21#wechat_redirect)  

  

**#** **往期推荐**

1.[记一次头铁的病毒样本分析过程](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458405156&idx=1&sn=c2061cbc0d160c86568497d663d40dbd&chksm=b18f79ae86f8f0b8c0072240342b03b737b21deb89fd1f37a26a7f58f5e407437fc92633997b&scene=21#wechat_redirect)  

2.[通过CmRegisterCallback学习注册表监控与反注册表监控](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458405024&idx=1&sn=a8617e12ff4d8e39d4fe89dae2cdf577&chksm=b18f782a86f8f13c05b91d743d6a09035c01d8a24b8d1c012cc68ccb2ff72d8bb45151b4a7fb&scene=21#wechat_redirect)

3.[全网最详细CVE-2014-0502 Adobe Flash Player双重释放漏洞分析](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458402566&idx=1&sn=6a8289758711c348b36f8526808747c7&chksm=b18f0f8c86f8869a416d9af04a202105779673e3997f5381b33fe58cb767314a4789e4623230&scene=21#wechat_redirect)

4.[基于linker实现so加壳补充-------从dex中加载so](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458402519&idx=1&sn=a42e81b669d38f9ff8ed1a8d941b3f2f&chksm=b18f0e5d86f8874b091e0c767b6d3e07c79a993eff4cb1554cf16d63308fc2f84511ac80f613&scene=21#wechat_redirect)

5.[一题house_of_storm的简单利用](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458402444&idx=1&sn=9baeafab94b844b2bb4a7c91f6609edc&chksm=b18f0e0686f88710326cc868dfbaddded2c8f13d48a99fdbced9f688b0b671c2782ce3d3921a&scene=21#wechat_redirect)

6.[BCTF2018-houseofatum-Writeup题解](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458402442&idx=1&sn=65f4c412b3495b27e168d94e8adf230f&chksm=b18f0e0086f887169bf7566dafa1b5cce2800a245159721b976d49da3d44d684357f526389a0&scene=21#wechat_redirect)

  

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

公众号ID：ikanxue  

官方微博：看雪安全

商务合作：wsc@kanxue.com

  

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**球分享**

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**球点赞**

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**球在看**

  

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

点击“阅读原文”，了解更多！

阅读原文

阅读 3842

​

写留言

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/1UG7KPNHN8EGLfh77kFmnicd9WOic2ibvhCibFdB4bL4srJCgo2wnvdoXLxpIvAkfCmmcptXZB0qKWMoIP8iaibYN2FA/300?wx_fmt=png&wxfrom=18)

看雪学苑

415

写留言

写留言

**留言**

暂无留言