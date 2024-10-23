
作者：[smcdef](http://www.wowotech.net/author/531) 发布于：2019-1-27 19:03 分类：[内存管理](http://www.wowotech.net/sort/memory_management)

# 引言

我们对copy\_{to,from}\_user()接口的使用应该是再熟悉不过吧。基本Linux书籍都会介绍它的作用。毕竟它是kernel space和user space沟通的桥梁。所有的数据交互都应该使用类似这种接口。所以，我们没有理由不知道接口的作用。但是，我也曾经有过以下疑问。

> 1. 为什么需要copy\_{to,from}\_user()，它究竟在背后为我们做了什么？
> 1. copy\_{to,from}\_user()和memcpy()的区别是什么，直接使用memcpy()可以吗？
> 1. memcpy()替代copy\_{to,from}\_user()是不是一定会有问题？

一下子找回了当年困惑的自己。我所提出的每个问题，曾经我也思考过。还不止一次的思考，每一次都有不同的想法。当然是因为从一开始就我就没有完全理解。现在又重新回到这个沉重的话题，继续思考这曾经的问题。

> 温馨提示：文章代码分析基于Linux-4.18.0，部分架构相关代码以ARM64为代表。

# 百家争鸣

针对以上问题当然是先百度。百度对于该问题的博客也是很多，足以看出这个问题肯定困惑着一大批Linux的爱好者。对于我的查阅结果来说，观点主要分成以下两种：

- copy\_{to,from}_user()比memcpy()多了传入地址合法性校验。例如是否属于用户空间地址范围。理论上说，内核空间可以直接使用用户空间传过来的指针，即使要做数据拷贝的动作，也可以直接使用memcpy()，事实上在没有MMU的体系架构上，copy_{to,from}\_user()最终的实现就是利用了memcpy()。但是对于大多数有MMU的平台，情况就有了些变化：用户空间传过来的指针是在虚拟地址空间上的，它所指向的虚拟地址空间很可能还没有真正映射到实际的物理页面上。但是这又能怎样呢？缺页导致的异常会很透明地被内核予以修复（为缺页的地址空间提交新的物理页面），访问到缺页的指令会继续运行仿佛什么都没有发生一样。但这只是用户空间缺页异常的行为，在内核空间这种缺页异常必须被显式地修复，这是由内核提供的缺页异常处理函数的设计模式决定的。其背后的思想是：在内核态，如果程序试图访问一个尚未被提交物理页面的用户空间地址，内核必须对此保持警惕而不能像用户空间那样毫无察觉。
- 如果我们确保用户态传递的指针的正确性，我们完全可以用memcpy()函数替代copy\_{to,from}\_user()。经过一些试验测试，发现使用memcpy()，程序的运行上并没有问题。因此在确保用户态指针安全的情况下，二者可以替换。

从各家博客上，观点主要集中在第一点。看起来第一点受到大家的广泛认可。但是，注重实践的人又得出了第二种观点，毕竟是实践出真知。真理究竟是是掌握在少数人手里呢？还是群众的眼睛是雪亮的呢？当然，我不否定以上任何一种观点。也不能向你保证哪种观点正确。因为，我相信即使是曾经无懈可击的理论，随着时间的推移或者特定情况的改变理论也可能不再正确。比如，牛顿的经典力学理论（好像扯得有点远）。如果要我说人话，就是：随着时间的推移，Linux的代码在不断的变化。或许以上的观点在曾经正确。当然，也可能现在还正确。下面的分析就是我的观点了。同样，大家也是需要保持怀疑的态度。下面我就抛砖引玉。

# 抛砖引玉

首先我们看下memcpy()和copy\_{to,from}\_user()的函数定义。参数几乎没有差别，都包含目的地址，源地址和需要复制的字节size。

1. static \_\_always_inline unsigned long \_\_must_check

1. copy_to_user(void \_\_user \*to, const void \*from, unsigned long n);

1. static \_\_always_inline unsigned long \_\_must_check

1. copy_from_user(void \*to, const void \_\_user \*from, unsigned long n);

1. void \*memcpy(void \*dest, const void \*src, size_t len);

但是，有一点我们肯定是知道的。那就是memcpy()没有传入地址合法性校验。而copy\_{to,from}\_user()针对传入地址进行类似下面的合法性校验（简单说点，更多校验详情可以参考代码）。

- 如果从用户空间copy数据到内核空间，用户空间地址to及to加上copy的字节长度n必须位于用户空间地址空间。
- 如果从内核空间copy数据到用户空间，当然也需要检查地址的合法性。例如，是否越界访问或者是不是代码段的数据等等。总之一切不合法地操作都需要立刻杜绝。

经过简单的对比之后，我们再看看其他的差异以及一起探讨下上面提出的2个观点。我们先从第2个观点说起。涉及实践，我还是有点相信实践出真知。从我测试的结果来说，实现结果分成两种情况。

第一种情况的结果是：使用memcpy()测试，没有出现问题，代码正常运行。测试代码如下（仅仅展示proc文件系统下file_operations对应的read接口函数）：

1. static ssize_t test_read(struct file \*file, char \_\_user \*buf,

1. ```
                        size_t len, loff_t *offset)
   ```

1. {

1. ```
   	memcpy(buf, "test\n", 5);	/* copy_to_user(buf, "test\n", 5) */
   ```

1. ```
   	return 5;
   ```

1. }

我们使用cat命令读取文件内容，cat会通过系统调用read调用test_read，并且传递的buf大小是4k。测试很顺利，结果很喜人。成功地读到了“test”字符串。看起来，第2点观点是没毛病的。但是，我们还需要继续验证和探究下去。因为第1个观点提到，“在内核空间这种缺页异常必须被显式地修复”。因此我们还需要验证的情况是：如果buf在用户空间已经分配虚拟地址空间，但是并没有建立和物理内存的具体映射关系，这种情况下会出现内核态page fault。我们首先需要创建这种条件，找到符合的buf，然后测试。这里我当然没测啦。因为有测试结论（主要是因为我懒，构造这个条件我觉得比较麻烦）。这个测试是我的一个朋友，人称宋老师的“阿助教”阿克曼大牛。他曾经做个这个实验，并且得到的结论是：即使是没有建立和物理内存的具体映射关系的buf，代码也可以正常运行。在内核态发生page fault，并被其修复（分配具体物理内存，填充页表，建立映射关系）。同时，我从代码的角度分析，结论也是如此。

经过上面的分析，看起来好像是memcpy()也可以正常使用，鉴于安全地考虑建议使用copy\_{to,from}\_user()等接口。

第二种情况的结果是：以上的测试代码并没有正常运行，并且会触发kernel oops。当然本次测试和上次测试的kernel配置选项是不一样的。这个配置项是`CONFIG_ARM64_SW_TTBR0_PAN`或者`CONFIG_ARM64_PAN`（针对ARM64平台）。两个配置选项的功能都是阻止内核态直接访问用户地址空间。只不过，CONFIG_ARM64_SW_TTBR0_PAN是软件仿真实现这种功能，而CONFIG_ARM64_PAN是硬件实现功能（ARMv8.1扩展功能）。我们以CONFIG_ARM64_SW_TTBR0_PAN作为分析对象（软件仿真才有代码提供分析）。BTW，如果硬件不支持，即使配置CONFIG_ARM64_PAN也没用，只能使用软件仿真的方法。内核Kconfig部分解释如下。如果需要访问用户空间地址需要通过类似copy\_{to,from}\_user()的接口，否则会导致kernel oops。

1. config ARM64_SW_TTBR0_PAN
1. ```
       bool "Emulate Privileged Access Never using TTBR0_EL1 switching"
   ```
1. ```
       help
   ```
1. ```
         Enabling this option prevents the kernel from accessing
   ```
1. ```
         user-space memory directly by pointing TTBR0_EL1 to a reserved
   ```
1. ```
         zeroed area and reserved ASID. The user access routines
   ```
1. ```
         restore the valid TTBR0_EL1 temporarily. 
   ```

在打开CONFIG_ARM64_SW_TTBR0_PAN的选项后，测试以上代码就会导致kernel oops。原因就是内核态直接访问了用户空间地址。因此，在这种情况我们就不可以使用memcpy()。我们别无选择，只能使用copy\_{to,from}\_user()。当然了，我们也不是没有办法使用memcpy()，但是需要额外的操作。如何操作呢？下一节为你揭晓。

# 刨根问底

既然提到了CONFIG_ARM64_SW_TTBR0_PAN的配置选项。当然我也希望了解其背后设计的原理。由于ARM64的硬件特殊设计，我们使用两个页表基地址寄存器ttbr0_el1和ttbr1_el1。处理器根据64 bit地址的高16 bit判断访问的地址属于用户空间还是内核空间。如果是用户空间地址则使用ttbr0_el1，反之使用ttbr1_el1。因此，ARM64进程切换的时候，只需要改变ttbr0_el1的值即可。ttbr1_el1可以选择不需要改变，因为所有的进程共享相同的内核空间地址。

当进程切换到内核态（中断，异常，系统调用等）后，如何才能避免内核态访问用户态地址空间呢？其实不难想出，改变ttbr0_el1的值即可，指向一段非法的映射即可。因此，我们为此准备了一份特殊的页表，该页表大小4k内存，其值全是0。当进程切换到内核态后，修改ttbr0_el1的值为该页表的地址即可保证访问用户空间地址是非法访问。因为页表的值是非法的。这个特殊的页表内存通过[链接脚本](https://elixir.bootlin.com/linux/v4.18/source/arch/arm64/kernel/vmlinux.lds.S#L228)分配。

```c#
#define RESERVED_TTBR0_SIZE	(PAGE_SIZE) SECTIONS{	reserved_ttbr0 = .;	. += RESERVED_TTBR0_SIZE;	swapper_pg_dir = .;	. += SWAPPER_DIR_SIZE;	swapper_pg_end = .;}
```

这个特殊的页表和内核页表在一起。和`swapper_pg_dir`仅仅差4k大小。`reserved_ttbr0`地址开始的4k内存空间的内容会被清零。

当我们进入内核态后会通过`__uaccess_ttbr0_disable`切换ttbr0_el1以关闭用户空间地址访问，在需要访问的时候通过`__uaccess_ttbr0_enable`打开用户空间地址访问。这两个宏定义也不复杂，就以\_\_uaccess_ttbr0_disable为例说明原理。其[定义](https://elixir.bootlin.com/linux/v4.18/source/arch/arm64/include/asm/asm-uaccess.h#L15)如下：

```c
 
.macro	__uaccess_ttbr0_disable, tmp1    mrs	\tmp1, ttbr1_el1                        // swapper_pg_dir (1)    bic	\tmp1, \tmp1, #TTBR_ASID_MASK    sub	\tmp1, \tmp1, #RESERVED_TTBR0_SIZE      // reserved_ttbr0 just before                                                // swapper_pg_dir (2)    msr	ttbr0_el1, \tmp1                        // set reserved TTBR0_EL1 (3)    isb    add	\tmp1, \tmp1, #RESERVED_TTBR0_SIZE    msr	ttbr1_el1, \tmp1                       // set reserved ASID    isb.endm
```

> 1. ttbr1_el1存储的是内核页表基地址，因此其值就是swapper_pg_dir。
> 1. swapper_pg_dir减去RESERVED_TTBR0_SIZE就是上面描述的特殊页表。
> 1. 将ttbr0_el1修改指向这个特殊的页表基地址，当然可以保证后续访问用户地址都是非法的。

\__uaccess_ttbr0_disable对应的C语言实现可以参考[这里](https://elixir.bootlin.com/linux/v4.18/source/arch/arm64/include/asm/uaccess.h#L120)。如何允许内核态访问用户空间地址呢？也很简单，就是\_\_uaccess_ttbr0_disable的反操作，给ttbr0_el1赋予合法的页表基地址。这里就不必重复了。我们现在需要知道的事实就是，在配置CONFIG_ARM64_SW_TTBR0_PAN的情况下，copy_{to,from}_user()接口会在copy之前允许内核态访问用户空间，并在copy结束之后关闭内核态访问用户空间的能力。因此，使用copy_{to,from}\_user()才是正统做法。主要体现在安全性检查及安全访问处理。这里是其比memcpy()多的第一个特性，后面还会介绍另一个重要特性。

现在我们可以解答上一节中遗留的问题。怎样才能继续使用memcpy()？现在就很简单了，在memcpy()调用之前通过[uaccess_enable_not_uao()](https://elixir.bootlin.com/linux/v4.18/source/arch/arm64/include/asm/uaccess.h#L232)允许内核态访问用户空间地址，调用memcpy()，最后通过[uaccess_disable_not_uao()](https://elixir.bootlin.com/linux/v4.18/source/arch/arm64/include/asm/uaccess.h#L227)关闭内核态访问用户空间的能力。

# 未雨绸缪

以上的测试用例都是建立在用户空间传递合法地址的基础上测试的，何为合法的用户空间地址？用户空间通过系统调用申请的虚拟地址空间包含的地址范围，即是合法的地址（不论是否分配物理页面建立映射关系）。既然要写一个接口程序，当然也要考虑程序的健壮性，我们不能假设所有的用户传递的参数都是合法的。我们应该预判非法传参情况的发生，并提前做好准备，这就是未雨绸缪。

我们首先使用memcpy()的测试用例，随机传递一个非法的地址。经过测试发现：会触发kernel oops。继续使用copy\_{to,from}\_user()替代memcpy()测试。测试发现：read()仅仅是返回错误，但不会触发kernel oops。这才是我们想要的结果。毕竟，一个应用程序不应该触发kernel oops。这种机制的实现原理是什么呢？

我们以[copy_to_user()](https://elixir.bootlin.com/linux/v4.18/source/include/linux/uaccess.h#L152)为例分析。函数调用流程如下：

```c
copy_to_user()->_copy_to_user()->raw_copy_to_user()->__arch_copy_to_user()
```

[\_\_arch_copy_to_user()](https://elixir.bootlin.com/linux/v4.18/source/arch/arm64/lib/copy_to_user.S#L65)在ARM64平台是汇编代码实现，这部分代码很关键。

```C
end	.req	x5ENTRY(__arch_copy_to_user)	uaccess_enable_not_uao x3, x4, x5	add	end, x0, x2#include "copy_template.S"	uaccess_disable_not_uao x3, x4	mov	x0, #0	retENDPROC(__arch_copy_to_user) 	.section .fixup,"ax"	.align	29998:	sub	x0, end, dst			// bytes not copied	ret	.previous
```

> 1. uaccess_enable_not_uao和uaccess_disable_not_uao是上面说到的内核态访问用户空间的开关。
> 1. copy_template.S文件是汇编实现的memcpy()的功能，稍后看看memcpy()的实现代码就清楚了。
> 1. `.section .fixup,“ax”`定义一个section，名为“.fixup”，权限是ax（‘a’可重定位的段，‘x’可执行段）。`9998`标号处的指令就是“未雨绸缪”的善后处理工作。还记得copy\_{to,from}_user()返回值的意义吗？返回0代表copy成功，否则返回剩余没有copy的字节数。这行代码就是计算剩余没有copy的字节数。当我们访问非法的用户空间地址的时候，就一定会触发page fault。这种情况下，内核态发生的page fault并返回的时候并没有修复异常，所以肯定不能返回发生异常的地址继续运行。所以，系统可以有2个选择：第1个选择是kernel oops，并给当前进程发送SIGSEGV信号；第2个选择是不返回出现异常的地址运行，而是选择一个已经修复的地址返回。如果使用的是memcpy()就只有第1个选择。但是copy_{to,from}\_user()可以有第2个选择。`.fixup`段就是为了实现这个修复功能。当copy过程中出现访问非法用户空间地址的时候，do_page_fault()返回的地址变成`9998`标号处，此时可以计算剩余未copy的字节长度，程序还可以继续执行。

对比前面分析的结果，其实\_\_arch_copy_to_user()可以近似等效如下关系。

```c
uaccess_enable_not_uao();memcpy(ubuf, kbuf, size);      ==     __arch_copy_to_user(ubuf, kbuf, size);uaccess_disable_not_uao();
```

先插播一条消息，解释[copy_template.S](https://elixir.bootlin.com/linux/v4.18/source/arch/arm64/lib/copy_template.S)为何是memcpy()。memcpy()在ARM64平台是由汇编代码实现。其定义在[arch/arm64/lib/memcpy.S](https://elixir.bootlin.com/linux/v4.18/source/arch/arm64/lib/memcpy.S)文件。

```c
 .weak memcpyENTRY(__memcpy)ENTRY(memcpy)#include "copy_template.S"	retENDPIPROC(memcpy)ENDPROC(__memcpy)
```

所以很明显，memcpy()和\_\_memcpy()函数定义是一样的。并且memcpy()函数声明是weak，因此可以重写memcpy()函数（扯得有点远）。再扯一点，为何使用汇编呢？为何不使用lib/string.c文件的[memcpy()](https://elixir.bootlin.com/linux/v4.18/source/lib/string.c#L803)函数呢？当然是为了优化memcpy() 的执行速度。lib/string.c文件的memcpy()函数是按照字节为单位进行copy（再好的硬件也会被粗糙的代码毁掉）。但是现在的处理器基本都是32或者64位，完全可以4 bytes或者8 bytes甚至16 bytes copy（考虑地址对齐的情况下）。可以明显提升执行速度。所以，ARM64平台使用汇编实现。这部分知识可以参考这篇博客[《ARM64 的 memcpy 优化与实现》](https://www.byteisland.com/arm64-%E7%9A%84-memcpy-%E6%B1%87%E7%BC%96%E5%88%86%E6%9E%90/)。

下面继续进入正题，再重复一遍：内核态访问用户空间地址，如果触发page fault，只要用户空间地址合法，内核态也会像什么也没有发生一样修复异常（分配物理内存，建立页表映射关系）。但是如果访问非法用户空间地址，就选择第2条路，尝试救赎自己。这条路就是利用`.fixup`和`__ex_table`段。如果无力回天只能给当前进程发送SIGSEGV信号。并且，轻则kernel oops，重则panic（取决于kernel配置选项CONFIG_PANIC_ON_OOPS）。在内核态访问非法用户空间地址的情况下，do_page_fault()最终会跳转`no_context`标号处的[\_\_do_kernel_fault()](https://elixir.bootlin.com/linux/v4.18/source/arch/arm64/mm/fault.c#L274)。

```c
static void __do_kernel_fault(unsigned long addr, unsigned int esr,                              struct pt_regs *regs){	/*	 * Are we prepared to handle this kernel fault?	 * We are almost certainly not prepared to handle instruction faults.	 */	if (!is_el1_instruction_abort(esr) && fixup_exception(regs))		return;	/* ... */}
```

[fixup_exception()](https://elixir.bootlin.com/linux/v4.18/source/arch/arm64/mm/extable.c#L9)继续调用[search_exception_tables()](https://elixir.bootlin.com/linux/v4.18/source/kernel/extable.c#L56)，其通过查找\_\_ex_table段。\_\_ex_table段存储exception table，每个entry存储着异常地址及其对应修复的地址。例如上述的`9998: sub x0, end, dst`指令的地址就会被找到并修改do_page_fault()函数的返回地址，以达到跳转修复的功能。其实查找过程是根据出问题的地址addr，查找\_\_ex_table段（exception table）是否有对应的exception table entry，如果有就代表可以被修复。由于32位处理器和64位处理器实现方式有差别，因此我们先从32位处理器异常表的实现原理说起。

\_\_ex_table段的首尾地址分别是`__start___ex_table`和`__stop___ex_table`（定义在[include/asm-generic/vmlinux.lds.h](https://elixir.bootlin.com/linux/v4.18/source/include/asm-generic/vmlinux.lds.h#L563)。这段内存可以看作是一个数组，数组的每个元素都是`struct exception_table_entry`类型，其记录着异常发生地址及其对应的修复地址。

```c
 exception tables__start___ex_table --> +---------------+                       |     entry     |                       +---------------+                       |     entry     |                       +---------------+                       |      ...      |                       +---------------+                       |     entry     |                       +---------------+                       |     entry     |__stop___ex_table  --> +---------------+
```

在32位处理器上，struct exception_table_entry[定义](https://elixir.bootlin.com/linux/v4.18/source/include/asm-generic/extable.h#L18)如下：

```c
struct exception_table_entry {	unsigned long insn, fixup;};
```

有一点需要明确，在32位处理器上，unsigned long是4 bytes。insn和fixup分别存储异常发生地址及其对应的修复地址。根据异常地址ex_addr查找对应的修复地址（未找到返回0），其示意代码如下：

```c
unsigned long search_fixup_addr32(unsigned long ex_addr){	const struct exception_table_entry *e; 	for (e = __start___ex_table; e < __stop___ex_table; e++)		if (ex_addr == e->insn)			return e->fixup; 	return 0;}
```

在32位处理器上，创建exception table entry相对简单。针对copy\_{to,from}\_user()汇编代码中每一处用户空间地址访问的指令都会创建一个entry，并且insn存储当前指令对应的地址，fixup存储修复指令对应的地址。

当64位处理器开始发展起来，如果我们继续使用这种方式，势必需要2倍于32位处理器的内存存储exception table（因为存储一个地址需要8 bytes）。所以，kernel换用另一种方式实现。在64处理器上，struct exception_table_entry[定义](https://elixir.bootlin.com/linux/v4.18/source/arch/arm64/include/asm/extable.h#L18)如下：

```c
struct exception_table_entry {	int insn, fixup;};
```

每个exception table entry占用的内存和32位处理器情况一样，因此内存占用不变。但是insn和fixup的意义发生变化。insn和fixup分别存储着异常发生地址及修复地址相对于当前结构体成员地址的偏移（有点拗口）。例如，根据异常地址ex_addr查找对应的修复地址（未找到返回0），其示意代码如下：

```c
1. unsigned long search_fixup_addr64(unsigned long ex_addr)
2. {
3. 	const struct exception_table_entry *e;

5. 	for (e = __start___ex_table; e < __stop___ex_table; e++)
6. 		if (ex_addr == (unsigned long)&e->insn + e->insn)
7. 			return (unsigned long)&e->fixup + e->fixup;

9. 	return 0;
10. } 
```

因此，我们的关注点就是如何去构建exception_table_entry。我们针对每个用户空间地址的内存访问都需要创建一个exception table entry，并插入\_\_ex_table段。例如下面的汇编指令（汇编指令对应的地址是随意写的，不用纠结对错。理解原理才是王道）。

```c
0xffff000000000000: ldr x1, [x0]0xffff000000000004: add x1, x1, #0x100xffff000000000008: ldr x2, [x0, #0x10]/* ... */0xffff000040000000: mov x0, #0xfffffffffffffff2    // -140xffff000040000004: ret
```

假设x0寄存器保存着用户空间地址，因此我们需要对0xffff000000000000地址的汇编指令创建一个exception table entry，并且我们期望当x0是非法用户空间地址时，跳转返回的修复地址是0xffff000040000000。为了计算简单，假设这是创建第一个entry，`__start___ex_table`值是0xffff000080000000。那么第一个exception table entry的insn和fixup成员的值分别是：0x80000000和0xbffffffc（这两个值都是负数）。因此，针对copy\_{to,from}\_user()汇编代码中每一处用户空间地址访问的指令都会创建一个entry。所以0xffff000000000008地址处的汇编指令也需要创建一个exception table entry。

所以，如果内核态访问非法用户空间地址究竟发生了什么？上面的分析流程可以总结如下：

1. 访问非法用户空间地址：

   `0xffff000000000000: ldr x1, [x0]`

1. MMU触发异常

1. CPU调用do_page_fault()

1. do_page_fault()调用search_exception_table()（regs->pc == 0xffff000000000000）

1. 查看\_\_ex_table段，寻找0xffff000000000000 并且返回修复地址0xffff000040000000

1. do_page_fault()修改函数返回地址（regs->pc = 0xffff000040000000）并返回

1. 程序继续执行，处理出错情况

1. 修改函数返回值x0 = -EFAULT (-14) 并返回（ARM64通过x0传递函数返回值）

# 总结

到了回顾总结的时候，copy\_{to,from}\_user()的思考也到此结束。我们来个总结结束此文。

- 无论是内核态还是用户态访问合法的用户空间地址，当虚拟地址并未建立物理地址的映射关系的时候，page fault的流程几乎一样，都会帮助我们申请物理内存并创建映射关系。所以这种情况下memcpy()和copy\_{to,from}\_user()是类似的。
- 当内核态访问非法用户空间地址的时候，通过`.fixup`和`__ex_table`两个段的帮助尝试修复异常。这种修复异常并不是建立地址映射关系，而是修改do_page_fault()返回地址。memcpy()由于没有创建这样的段，所以memcpy()无法做到这点。
- 在使能`CONFIG_ARM64_SW_TTBR0_PAN`或者`CONFIG_ARM64_PAN`（硬件支持的情况下才有效）的时候，我们只能使用copy\_{to,from}\_user()这种接口，直接使用memcpy()是不行的。

最后，我想说，即使在某些情况下memcpy()可以正常工作。但是，这也是不推荐的，不是良好的编程习惯。在用户空间和内核空间数据交互上，我们必须使用类似copy\_{to,from}_user()的接口。为什么类似呢？因为还有其他的接口用于内核空间和用户空间数据交互，只是没有copy_{to,from}\_user()出名。例如：{get,put}\_user()。

标签: [copy_to_user](http://www.wowotech.net/tag/copy_to_user)

---

« [内存一致性模型](http://www.wowotech.net/memory_management/456.html) | [编译乱序(Compiler Reordering)](http://www.wowotech.net/kernel_synchronization/453.html)»

**评论：**

**schspa**\
2019-04-30 14:45

最后，我想说，即使在某些情况下memcpy()可以正常工作。但是，这也是不推荐的，不是良好的编程习惯。在用户空间和内核空间数据交互上，我们必须使用类似copy\_{to,from}_user()的接口。为什么类似呢？因为还有其他的接口用于内核空间和用户空间数据交互，只是没有copy_{to,from}\_user()出名。例如：{get,put}\_user()。

\<\< 如果直接使用memcpy，会有安全漏洞，直接给了用户空间程序向内核任意地址写入，读取内容的能力。

[回复](http://www.wowotech.net/memory_management/454.html#comment-7388)

**[smcdef](http://www.wowotech.net/)**\
2019-05-02 21:14

@schspa：是啊！所以我一直不推荐使用memcpy()啊！但是，总还是有驱动开发者在用。

[回复](http://www.wowotech.net/memory_management/454.html#comment-7390)

**[紫书海](http://www.zishuhai.com/)**\
2019-03-21 11:50

如此好文章一定要留下名啊

[回复](http://www.wowotech.net/memory_management/454.html#comment-7329)

**leoxu**\
2019-03-02 03:56

pagefault处理的官方的文档，基于x86\
https://www.kernel.org/doc/Documentation/x86/exception-tables.txt

[回复](http://www.wowotech.net/memory_management/454.html#comment-7215)

**[nstcl](http://www.wowotech.net/)**\
2019-01-31 15:39

“针对copy\_{to,from}\_user()汇编代码中每一处用户空间地址访问的指令都会创建一个entry”这个创建是如何实现的呢？谢谢

[回复](http://www.wowotech.net/memory_management/454.html#comment-7172)

**[smcdef](http://www.wowotech.net/)**\
2019-01-31 19:01

@nstcl：https://elixir.bootlin.com/linux/v4.18/source/arch/arm64/lib/copy_to_user.S#L65

上面的链接是\_\_arch_copy_to_user()汇编代码实现。注意，9998 标号的位置就是fixup address。因此，需要构建 entry 就离不开这个9998。所以，你看下，\_\_arch_copy_to_user()上半部分的汇编，是一堆宏定义。注意几个包含9998数字的几个宏。

下面需要做的事情就是展开这些宏吧。

[回复](http://www.wowotech.net/memory_management/454.html#comment-7174)

**[nstcl](http://www.wowotech.net/)**\
2019-02-14 11:52

@smcdef：感谢

[回复](http://www.wowotech.net/memory_management/454.html#comment-7183)

**[linuxer](http://www.wowotech.net/)**\
2019-01-29 22:37

用memory copy取代copy\_{to,from}\_user()是绝对禁止的，没有地址检查（access_ok）会引起安全问题。在4.13内核中有一个惨痛的案例：waitpid系统调用使用了unsafe版本的copy_to_user，遗漏了access_ok，从而导致了安全漏洞，而黑客可以利用该漏洞轻松的让普通进程获取root权限。

另外，当你想使用memory copy取代copy\_{to,from}_user()的时候，你潜意识中做了一些假设。以32位系统为例，1G：3G的地址空间分配（高端1G内核空间在所有进程之间共享）并不是唯一的选择，也许4G:4G也未尝不可。假如系统配置是4G：4G，那么memory copy是无法取代copy_{to,from}\_user()的。

[回复](http://www.wowotech.net/memory_management/454.html#comment-7164)

**[smcdef](http://www.wowotech.net/)**\
2019-01-29 22:50

@linuxer：感谢，郭大侠补充。所以，我们一定要规范编程。

[回复](http://www.wowotech.net/memory_management/454.html#comment-7166)

**[bsp](http://www.wowotech.net/)**\
2019-02-01 10:39

@linuxer：大侠连评论都这么精彩，含金量爆表！\
4G:4G的分配之前都没想过，不过通过arm的（TTBR0/TTBR1）确实可以实现。\
那么弱弱的问一下，x86也可以做4G:4G的分配吗？通过什么实现呢？

[回复](http://www.wowotech.net/memory_management/454.html#comment-7175)

**[linuxer](http://www.wowotech.net/)**\
2019-02-02 19:07

@bsp：https://lwn.net/Articles/39283/\
https://lwn.net/Articles/86715/

这里有一些信息可以参考一下。4G/4G split patch是在特定场景下有意义（32bit CPU，内存配置非常大的情况），但是会引入性能问题（在系统调用的时候还需要切换地址空间），因此，现在这种地址空间分法也就几乎没有人提起了。

[回复](http://www.wowotech.net/memory_management/454.html#comment-7177)

**hzm**\
2019-01-29 21:17

访问非法用户空间地址的时候，通过.fixup修复...需要做这种修复的地方岂不是很多？一句代码就要修复一次感觉有点多呀... 楼主有无对各种fixup的总结讲解呢~

[回复](http://www.wowotech.net/memory_management/454.html#comment-7163)

**[smcdef](http://www.wowotech.net/)**\
2019-01-29 22:48

@hzm：为何会很多呢？我们只需要考虑内核空间访问用户空间的情况。而这种情况主要集中在内核空间和用户空间数据的交互接口。例如，copy_to_user()接口上，也并不是需要对每一条指令fixup，只需要对访问用户空间地址的指令fixup。所以，总的来说并不是太多。

[回复](http://www.wowotech.net/memory_management/454.html#comment-7165)

**yuan**\
2019-01-28 15:11

有一个 typo: copy\_{to,from}\_user()最终的实现就是利用了mencpy() => memcpy()

[回复](http://www.wowotech.net/memory_management/454.html#comment-7160)

**[smcdef](http://www.wowotech.net/)**\
2019-01-28 17:12

@yuan：细心，为你点个赞。

[回复](http://www.wowotech.net/memory_management/454.html#comment-7162)

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

  - [TLB flush操作](http://www.wowotech.net/memory_management/tlb-flush.html)
  - [小printf大作用（用日志打印的方式调试程序）](http://www.wowotech.net/soft/7.html)
  - [CFS任务放置代码详解](http://www.wowotech.net/process_management/task_placement_detail.html)
  - [Linux serial framework(1)\_概述](http://www.wowotech.net/comm/serial_overview.html)
  - [process credentials](http://www.wowotech.net/process_management/19.html)

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
