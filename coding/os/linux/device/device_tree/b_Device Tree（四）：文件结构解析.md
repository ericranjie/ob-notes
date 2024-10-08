作者：[smcdef](http://www.wowotech.net/author/531) 发布于：2017-9-24 11:08 分类：[统一设备模型](http://www.wowotech.net/sort/device_model)

**前言**

通过linuxer发表的三篇设备树的文章，我想你应该对设备已经有一个非常充分的认识了。本篇文章即作为一篇Device Tree的总结性文章，同时也作为linuxer文章的补充。本篇文章曾发表在Linuxer公众号，链接为：

[http://mp.weixin.qq.com/s/OX-aXd5MYlE_YoZ3p32qWA](http://http//mp.weixin.qq.com/s/OX-aXd5MYlE_YoZ3p32qWA)

**1. Device Tree简介**\
设备树就是描述单板资源以及设备的一种文本文件。本篇文章主要是更深层次的探讨设备文件的构成以及kernel解析设备树的原理。所以，本篇内容并不是针对没有任何设备树知识的读者。本篇文章主要针对已经使用过设备树或者对设备已经有所了解并想深层次的探究设备树的文件结构和kernel解析过程的读者。\
**2. Device Tree编译**\
Device Tree文件的格式为dts，包含的头文件格式为dtsi，dts文件是一种人可以看懂的编码格式。但是uboot和linux不能直接识别，他们只能识别二进制文件，所以需要把dts文件编译成dtb文件。dtb文件是一种可以被kernel和uboot识别的二进制文件。把dts编译成dtb文件的工具是dtc。Linux源码目录下scripts/dtc目录包含dtc工具的源码。在Linux的scripts/dtc目录下除了提供dtc工具外，也可以自己安装dtc工具，linux下执行：sudo apt-get install device-tree-compiler安装dtc工具。dtc工具的使用方法是：dtc –I dts –O dtb –o xxx.dtb xxx.dts，即可生成dts文件对应的dtb文件了。 当然了，dtc –I dtb –O dts –o xxx.dts xxx.dtb反过来即可生成dts文件。其中还提供了一个fdtdump的工具，可以dump dtb文件,方便查看信息。

**3. Device Tree头信息**\
fdtdump工具使用，Linux终端执行ftddump –h，输出以下信息：

```cpp
fdtdump -h  
Usage: fdtdump options <file>  
 Options: -[dshV]  
   -d, --debug   Dump debug information while decoding the file  
   -s, --scan    Scan for an embedded fdt in file  
   -h, --help    Print this help and exit  
   -V, --version Print version and exit
```

本文采用s5pv21_smc.dtb文件为例说明fdtdump工具的使用。Linux终端执行fdtdump –sd s5pv21_smc.dtb > s5pv21_smc.txt，打开s5pv21_smc.txt文件，部分输出信息如下所示：

```cpp
// magic:  0xd00dfeed  
// totalsize:  0xce4 (3300)  
// off_dt_struct: 0x38  
// off_dt_strings: 0xc34  
// off_mem_rsvmap: 0x28  
// version: 17  
// last_comp_version: 16  
// boot_cpuid_phys: 0x0  
// size_dt_strings: 0xb0  
// size_dt_struct: 0xbfc
```

以上信息便是Device Tree文件头信息，存储在dtb文件的开头部分。在Linux内核中使用struct fdt_header结构体描述。struct fdt_header结构体定义在scripts\\dtc\\libfdt\\fdt.h文件中。

```cpp
1. struct fdt_header {
2. 	fdt32_t magic;			     /* magic word FDT_MAGIC */
3. 	fdt32_t totalsize;		     /* total size of DT block */
4. 	fdt32_t off_dt_struct;		 /* offset to structure */
5. 	fdt32_t off_dt_strings;		 /* offset to strings */
6. 	fdt32_t off_mem_rsvmap;		 /* offset to memory reserve map */
7. 	fdt32_t version;		         /* format version */
8. 	fdt32_t last_comp_version;	 /* last compatible version */

10. 	/* version 2 fields below */
11. 	fdt32_t boot_cpuid_phys;	 /* Which physical CPU id we're booting on */
12. 	/* version 3 fields below */
13. 	fdt32_t size_dt_strings;	 /* size of the strings block */

15. 	/* version 17 fields below */
16. 	fdt32_t size_dt_struct;		 /* size of the structure block */
17. };
```

fdtdump工具的输出信息即是以上结构中每一个成员的值，struct fdt_header结构体包含了Device Tree的私有信息。例如: fdt_header.magic是fdt的魔数,固定值为0xd00dfeed，fdt_header.totalsize是fdt文件的大小。使用二进制工具打开s5pv21_smc.dtb验证。s5pv21_smc.dtb二进制文件头信息如下图所示。从下图中可以得到Device Tree的文件是以大端模式储存。并且，头部信息和fdtdump的输出信息一致。

[![](http://www.wowotech.net/content/uploadfile/201709/97111506232039.png)](http://www.wowotech.net/content/uploadfile/201709/97111506232039.png)

Device Tree中的节点信息举例如下图所示。

![](http://www.wowotech.net/content/uploadfile/201709/1dc41506232049.png)

上述.dts文件并没有什么真实的用途，但它基本表征了一个Device Tree源文件的结构：

> 1个root结点"/"；root结点下面含一系列子结点，本例中为"node@0"和"node@1"；结点"node@0"下又含有一系列子结点，本例中为"child-node@0"；各结点都有一系列属性。
>
> 这些属性可能为空，如" an-empty-property"；可能为字符串，如"a-string-property"；可能为字符串数组，如"a-string-list-property"；可能为Cells（由u32整数组成），如"second-child-property"，可能为二进制数，如"a-byte-data-property"。
>
> Device Tree源文件的结构分为header、fill_area、dt_struct及dt_string四个区域。header为头信息，fill_area为填充区域，填充数字0，dt_struct存储节点数值及名称相关信息，dt_string存储属性名。例如：a-string-property就存储在dt_string区，"A string"及node1就存储在dt_struct区域。\
> 我们可以给一个设备节点添加lable，之后可以通过&lable的形式访问这个lable，这种引用是通过phandle（pointer handle）进行的。例如，下图中的node1就是一个lable，node@0的子节点child-node@0通过&node1引用node@1节点。像是这种phandle的节点，在经过DTC工具编译之后，&node1会变成一个特殊的整型数字n，假设n值为1，那么在node@1节点下自动生成两个属性，属性如下：\
> linux,phandle = \<0x00000001>;\
> phandle = \<0x00000001>;
>
> node@0的子节点child-node@0中的a-reference-to-something = \<&node1>会变成a-reference-to-something = \< 0x00000001>。此处0x00000001就是一个phandle得值，每一个phandle都有一个独一无二的整型值，在后续kernel中通过这个特殊的数字间接找到引用的节点。通过查看fdtdump输出信息以及dtb二进制文件信息，得到struct fdt_header和文件结构之间的关系信息如所示。

[![](http://www.wowotech.net/content/uploadfile/201709/42e01506232107.png)](http://www.wowotech.net/content/uploadfile/201709/42e01506232107.png)

**4. Device Tree文件结构**\
通过以上分析，可以得到Device Tree文件结构如下图所示。dtb的头部首先存放的是fdt_header的结构体信息，接着是填充区域，填充大小为off_dt_struct – sizeof(struct fdt_header)，填充的值为0。接着就是struct fdt_property结构体的相关信息。最后是dt_string部分。

[![](http://www.wowotech.net/content/uploadfile/201709/518c1506232119.png)](http://www.wowotech.net/content/uploadfile/201709/518c1506232119.png)

Device Tree源文件的结构分为header、fill_area、dt_struct及dt_string四个区域。fill_area区域填充数值0。节点（node）信息使用struct fdt_node_header结构体描述。属性信息使用struct fdt_property结构体描述。各个结构体信息如下:

```cpp
1. struct fdt_node_header {
2. 	fdt32_t tag;
3. 	char name[0];
4. };

6. struct fdt_property {
7. 	fdt32_t tag;
8. 	fdt32_t len;
9. 	fdt32_t nameoff;
10. 	char data[0];
11. };
```

struct fdt_node_header描述节点信息，tag是标识node的起始结束等信息的标志位，name指向node名称的首地址。tag的取值如下：

```cpp
1. #define FDT_BEGIN_NODE	0x1		/* Start node: full name */
2. #define FDT_END_NODE	0x2		/* End node */
3. #define FDT_PROP	      0x3		/* Property: name off, size, content */
4. #define FDT_NOP		0x4		/* nop */
5. #define FDT_END		0x9
```

FDT_BEGIN_NODE和FDT_END_NODE标识node节点的起始和结束，FDT_PROP标识node节点下面的属性起始符，FDT_END标识Device Tree的结束标识符。因此，对于每个node节点的tag标识符一般为FDT_BEGIN_NODE，对于每个node节点下面的属性的tag标识符一般是FDT_PROP。

描述属性采用struct fdt_property描述，tag标识是属性，取值为FDT_PROP；len为属性值的长度（包括‘\\0’，单位：字节）；nameoff为属性名称存储位置相对于off_dt_strings的偏移地址。

例如：compatible = "samsung,goni", "samsung,s5pv210";compatible是属性名称，"samsung,goni", "samsung,s5pv210"是属性值。compatible属性名称字符串存放的区域是dt_string。"samsung,goni", "samsung,s5pv210"存放的位置是fdt_property.data后面。因此fdt_property.data指向该属性值。fdt_property.tag的值为属性标识，len为属性值的长度（包括‘\\0’，单位：字节）,此处len = 29。nameoff为compatible字符串的位置相对于off_dt_strings的偏移地址，即&compatible = nameoff + off_dt_strings。 dt_struct在Device Tree中的结构如下图所示。节点的嵌套也带来tag标识符的嵌套。

[![](http://www.wowotech.net/content/uploadfile/201709/52ae1506232173.png)](http://www.wowotech.net/content/uploadfile/201709/52ae1506232173.png)

**5. kernel解析Device Tree**\
Device Tree文件结构描述就以上struct fdt_header、struct fdt_node_header及struct fdt_property三个结构体描述。kernel会根据Device Tree的结构解析出kernel能够使用的struct property结构体。kernel根据Device Tree中所有的属性解析出数据填充struct property结构体。struct property结构体描述如下：

```cpp
1. struct property {
2. 	char *name;                          /* property full name */
3. 	int length;                          /* property value length */
4. 	void *value;                         /* property value */
5. 	struct property *next;             /* next property under the same node */
6. 	unsigned long _flags;
7. 	unsigned int unique_id;
8. 	struct bin_attribute attr;        /* 属性文件，与sysfs文件系统挂接 */
9. };
```

总的来说，kernel根据Device Tree的文件结构信息转换成struct property结构体，并将同一个node节点下面的所有属性通过property.next指针进行链接，形成一个单链表。\
kernel中究竟是如何解析Device Tree的呢？下面分析函数解析过程。函数调用过程如下图所示。kernel的C语言阶段的入口函数是init/main.c/stsrt_kernel()函数，在early_init_dt_scan_nodes()中会做以下三件事：

(1) 扫描/chosen或者/chose@0节点下面的bootargs属性值到boot_command_line，此外，还处理initrd相关的property，并保存在initrd_start和initrd_end这两个全局变量中；

(2) 扫描根节点下面，获取{size,address}-cells信息，并保存在dt_root_size_cells和dt_root_addr_cells全局变量中；

(3) 扫描具有device_type = “memory”属性的/memory或者/memory@0节点下面的reg属性值，并把相关信息保存在meminfo中，全局变量meminfo保存了系统内存相关的信息。

[![](http://www.wowotech.net/content/uploadfile/201709/a40d1506232259.png)](http://www.wowotech.net/content/uploadfile/201709/a40d1506232259.png)

Device Tree中的每一个node节点经过kernel处理都会生成一个struct device_node的结构体，struct device_node最终一般会被挂接到具体的struct device结构体。struct device_node结构体描述如下：

```cpp
1. struct device_node {
2. 	const char *name;              /* node的名称，取最后一次“/”和“@”之间子串 */
3. 	const char *type;              /* device_type的属性名称，没有为<NULL> */
4. 	phandle phandle;               /* phandle属性值 */
5. 	const char *full_name;        /* 指向该结构体结束的位置，存放node的路径全名，例如：/chosen */
6. 	struct fwnode_handle fwnode;

8. 	struct	property *properties;  /* 指向该节点下的第一个属性，其他属性与该属性链表相接 */
9. 	struct	property *deadprops;   /* removed properties */
10. 	struct	device_node *parent;   /* 父节点 */
11. 	struct	device_node *child;    /* 子节点 */
12. 	struct	device_node *sibling;  /* 姊妹节点，与自己同等级的node */
13. 	struct	kobject kobj;            /* sysfs文件系统目录体现 */
14. 	unsigned long _flags;          /* 当前node状态标志位，见/include/linux/of.h line124-127 */
15. 	void	*data;
16. };

18. /* flag descriptions (need to be visible even when !CONFIG_OF) */
19. #define OF_DYNAMIC        1 /* node and properties were allocated via kmalloc */
20. #define OF_DETACHED       2 /* node has been detached from the device tree*/
21. #define OF_POPULATED      3 /* device already created for the node */
22. #define OF_POPULATED_BUS 4 /* of_platform_populate recursed to children of this node */
```

struct device_node结构体中的每个成员作用已经备注了注释信息，下面分析以上信息是如何得来的。Device Tree的解析首先从unflatten_device_tree()开始，代码列出如下：

```cpp
1. /**
2.  * unflatten_device_tree - create tree of device_nodes from flat blob
3.  *
4.  * unflattens the device-tree passed by the firmware, creating the
5.  * tree of struct device_node. It also fills the "name" and "type"
6.  * pointers of the nodes so the normal device-tree walking functions
7.  * can be used.
8.  */
9. void __init unflatten_device_tree(void)
10. {
11. 	__unflatten_device_tree(initial_boot_params, &of_root,
12. 				early_init_dt_alloc_memory_arch);

14. 	/* Get pointer to "/chosen" and "/aliases" nodes for use everywhere */
15. 	of_alias_scan(early_init_dt_alloc_memory_arch);
16. }

18. /**
19.  * __unflatten_device_tree - create tree of device_nodes from flat blob
20.  *
21.  * unflattens a device-tree, creating the
22.  * tree of struct device_node. It also fills the "name" and "type"
23.  * pointers of the nodes so the normal device-tree walking functions
24.  * can be used.
25.  * @blob: The blob to expand
26.  * @mynodes: The device_node tree created by the call
27.  * @dt_alloc: An allocator that provides a virtual address to memory
28.  * for the resulting tree
29.  */
30. static void __unflatten_device_tree(const void *blob,
31. 			     struct device_node **mynodes,
32. 			     void * (*dt_alloc)(u64 size, u64 align))
33. {
34. 	unsigned long size;
35. 	int start;
36. 	void *mem;

38.     /* 省略部分不重要部分 */
39. 	/* First pass, scan for size */
40. 	start = 0;
41. 	size = (unsigned long)unflatten_dt_node(blob, NULL, &start, NULL, NULL, 0, true);
42. 	size = ALIGN(size, 4);

44. 	/* Allocate memory for the expanded device tree */
45. 	mem = dt_alloc(size + 4, __alignof__(struct device_node));
46. 	memset(mem, 0, size);

48. 	/* Second pass, do actual unflattening */
49. 	start = 0;
50. 	unflatten_dt_node(blob, mem, &start, NULL, mynodes, 0, false);
51. }
```

分析以上代码，在unflatten_device_tree()中，调用函数\_\_unflatten_device_tree()，参数initial_boot_params指向Device Tree在内存中的首地址，of_root在经过该函数处理之后，会指向根节点，early_init_dt_alloc_memory_arch是一个函数指针，为struct device_node和struct property结构体分配内存的回调函数（callback）。在\_\_unflatten_device_tree()函数中，两次调用unflatten_dt_node()函数，第一次是为了得到Device Tree转换成struct device_node和struct property结构体需要分配的内存大小，第二次调用才是具体填充每一个struct device_node和struct property结构体。unflatten_dt_node()代码列出如下：

```cpp
1. /**
2.  * unflatten_dt_node - Alloc and populate a device_node from the flat tree
3.  * @blob: The parent device tree blob
4.  * @mem: Memory chunk to use for allocating device nodes and properties
5.  * @poffset: pointer to node in flat tree
6.  * @dad: Parent struct device_node
7.  * @nodepp: The device_node tree created by the call
8.  * @fpsize: Size of the node path up at the current depth.
9.  * @dryrun: If true, do not allocate device nodes but still calculate needed
10.  * memory size
11.  */
12. static void * unflatten_dt_node(const void *blob,
13. 				void *mem,
14. 				int *poffset,
15. 				struct device_node *dad,
16. 				struct device_node **nodepp,
17. 				unsigned long fpsize,
18. 				bool dryrun)
19. {
20. 	const __be32 *p;
21. 	struct device_node *np;
22. 	struct property *pp, **prev_pp = NULL;
23. 	const char *pathp;
24. 	unsigned int l, allocl;
25. 	static int depth;
26. 	int old_depth;
27. 	int offset;
28. 	int has_name = 0;
29. 	int new_format = 0;

31. 	/* 获取node节点的name指针到pathp中 */
32. 	pathp = fdt_get_name(blob, *poffset, &l);
33. 	if (!pathp)
34. 		return mem;

36. 	allocl = ++l;

38. 	/* version 0x10 has a more compact unit name here instead of the full
39. 	 * path. we accumulate the full path size using "fpsize", we'll rebuild
40. 	 * it later. We detect this because the first character of the name is
41. 	 * not '/'.
42. 	 */
43. 	if ((*pathp) != '/') {
44. 		new_format = 1;
45. 		if (fpsize == 0) {
46. 			/* root node: special case. fpsize accounts for path
47. 			 * plus terminating zero. root node only has '/', so
48. 			 * fpsize should be 2, but we want to avoid the first
49. 			 * level nodes to have two '/' so we use fpsize 1 here
50. 			 */
51. 			fpsize = 1;
52. 			allocl = 2;
53. 			l = 1;
54. 			pathp = "";
55. 		} else {
56. 			/* account for '/' and path size minus terminal 0
57. 			 * already in 'l'
58. 			 */
59. 			fpsize += l;
60. 			allocl = fpsize;
61. 		}
62. 	}

64. 	/* 分配struct device_node内存，包括路径全称大小 */
65. 	np = unflatten_dt_alloc(&mem, sizeof(struct device_node) + allocl,
66. 				__alignof__(struct device_node));
67. 	if (!dryrun) {
68. 		char *fn;
69. 		of_node_init(np);

71. 		/* 填充full_name，full_name指向该node节点的全路径名称字符串 */
72. 		np->full_name = fn = ((char *)np) + sizeof(*np);
73. 		if (new_format) {
74. 			/* rebuild full path for new format */
75. 			if (dad && dad->parent) {
76. 				strcpy(fn, dad->full_name);
77. 				fn += strlen(fn);
78. 			}
79. 			*(fn++) = '/';
80. 		}
81. 		memcpy(fn, pathp, l);

83. 		/* 节点挂接到相应的父节点、子节点和姊妹节点 */
84. 		prev_pp = &np->properties;
85. 		if (dad != NULL) {
86. 			np->parent = dad;
87. 			np->sibling = dad->child;
88. 			dad->child = np;
89. 		}
90. 	}
91. 	/* 处理该node节点下面所有的property */
92. 	for (offset = fdt_first_property_offset(blob, *poffset);
93. 	     (offset >= 0);
94. 	     (offset = fdt_next_property_offset(blob, offset))) {
95. 		const char *pname;
96. 		u32 sz;

98. 		if (!(p = fdt_getprop_by_offset(blob, offset, &pname, &sz))) {
99. 			offset = -FDT_ERR_INTERNAL;
100. 			break;
101. 		}

103. 		if (pname == NULL) {
104. 			pr_info("Can't find property name in list !\n");
105. 			break;
106. 		}
107. 		if (strcmp(pname, "name") == 0)
108. 			has_name = 1;
109. 		pp = unflatten_dt_alloc(&mem, sizeof(struct property),
110. 					__alignof__(struct property));
111. 		if (!dryrun) {
112. 			/* We accept flattened tree phandles either in
113. 			 * ePAPR-style "phandle" properties, or the
114. 			 * legacy "linux,phandle" properties.  If both
115. 			 * appear and have different values, things
116. 			 * will get weird.  Don't do that. */

118. 			/* 处理phandle，得到phandle值 */
119. 			if ((strcmp(pname, "phandle") == 0) ||
120. 			    (strcmp(pname, "linux,phandle") == 0)) {
121. 				if (np->phandle == 0)
122. 					np->phandle = be32_to_cpup(p);
123. 			}
124. 			/* And we process the "ibm,phandle" property
125. 			 * used in pSeries dynamic device tree
126. 			 * stuff */
127. 			if (strcmp(pname, "ibm,phandle") == 0)
128. 				np->phandle = be32_to_cpup(p);
129. 			pp->name = (char *)pname;
130. 			pp->length = sz;
131. 			pp->value = (__be32 *)p;
132. 			*prev_pp = pp;
133. 			prev_pp = &pp->next;
134. 		}
135. 	}
136. 	/* with version 0x10 we may not have the name property, recreate
137. 	 * it here from the unit name if absent
138. 	 */
139. 	/* 为每个node节点添加一个name的属性 */
140. 	if (!has_name) {
141. 		const char *p1 = pathp, *ps = pathp, *pa = NULL;
142. 		int sz;

144. 		/* 属性name的value值为node节点的名称，取“/”和“@”之间的子串 */
145. 		while (*p1) {
146. 			if ((*p1) == '@')
147. 				pa = p1;
148. 			if ((*p1) == '/')
149. 				ps = p1 + 1;
150. 			p1++;
151. 		}
152. 		if (pa < ps)
153. 			pa = p1;
154. 		sz = (pa - ps) + 1;
155. 		pp = unflatten_dt_alloc(&mem, sizeof(struct property) + sz,
156. 					__alignof__(struct property));
157. 		if (!dryrun) {
158. 			pp->name = "name";
159. 			pp->length = sz;
160. 			pp->value = pp + 1;
161. 			*prev_pp = pp;
162. 			prev_pp = &pp->next;
163. 			memcpy(pp->value, ps, sz - 1);
164. 			((char *)pp->value)[sz - 1] = 0;
165. 		}
166. 	}
167. 	/* 填充device_node结构体中的name和type成员 */
168. 	if (!dryrun) {
169. 		*prev_pp = NULL;
170. 		np->name = of_get_property(np, "name", NULL);
171. 		np->type = of_get_property(np, "device_type", NULL);

173. 		if (!np->name)
174. 			np->name = "<NULL>";
175. 		if (!np->type)
176. 			np->type = "<NULL>";
177. 	}

179. 	old_depth = depth;
180. 	*poffset = fdt_next_node(blob, *poffset, &depth);
181. 	if (depth < 0)
182. 		depth = 0;
183. 	/* 递归调用node节点下面的子节点 */
184. 	while (*poffset > 0 && depth > old_depth)
185. 		mem = unflatten_dt_node(blob, mem, poffset, np, NULL,
186. 					fpsize, dryrun);

188. 	if (*poffset < 0 && *poffset != -FDT_ERR_NOTFOUND)
189. 		pr_err("unflatten: error %d processing FDT\n", *poffset);

191. 	/*
192. 	 * Reverse the child list. Some drivers assumes node order matches .dts
193. 	 * node order
194. 	 */
195. 	if (!dryrun && np->child) {
196. 		struct device_node *child = np->child;
197. 		np->child = NULL;
198. 		while (child) {
199. 			struct device_node *next = child->sibling;
200. 			child->sibling = np->child;
201. 			np->child = child;
202. 			child = next;
203. 		}
204. 	}

206. 	if (nodepp)
207. 		*nodepp = np;

209. 	return mem;
210. }
```

通过以上函数处理就得到了所有的struct device_node结构体，为每一个node都会自动添加一个名称为“name”的property，property.length的值为当前node的名称取最后一个“/”和“@”之间的子串（包括‘\\0’）。例如：/serial@e2900800，则length = 7，property.value = device_node.name = “serial”。\
**6. platform_device和device_node绑定**\
经过以上解析，Device Tree的数据已经全部解析出具体的struct device_node和struct property结构体，下面需要和具体的device进行绑定。首先讲解platform_device和device_node的绑定过程。在arch/arm/kernel/setup.c文件中，customize_machine()函数负责填充struct platform_device结构体。函数调用过程如下图所示。

[![](http://www.wowotech.net/content/uploadfile/201709/395b1506232186.png)](http://www.wowotech.net/content/uploadfile/201709/395b1506232186.png)

代码分析如下：

```cpp
1. const struct of_device_id of_default_bus_match_table[] = {
2. 	{ .compatible = "simple-bus", },
3. 	{ .compatible = "simple-mfd", },
4. #ifdef CONFIG_ARM_AMBA
5. 	{ .compatible = "arm,amba-bus", },
6. #endif /* CONFIG_ARM_AMBA */
7. 	{} /* Empty terminated list */
8. };

10. int of_platform_populate(struct device_node *root,
11. 			const struct of_device_id *matches,
12. 			const struct of_dev_auxdata *lookup,
13. 			struct device *parent)
14. {
15. 	struct device_node *child;
16. 	int rc = 0;

18. 	/* 获取根节点 */
19. 	root = root ? of_node_get(root) : of_find_node_by_path("/");
20. 	if (!root)
21. 		return -EINVAL;

23. 	/* 为根节点下面的每一个节点创建platform_device结构体 */
24. 	for_each_child_of_node(root, child) {
25. 		rc = of_platform_bus_create(child, matches, lookup, parent, true);
26. 		if (rc) {
27. 			of_node_put(child);
28. 			break;
29. 		}
30. 	}
31. 	/* 更新device_node flag标志位 */
32. 	of_node_set_flag(root, OF_POPULATED_BUS);

34. 	of_node_put(root);
35. 	return rc;
36. }

38. static int of_platform_bus_create(struct device_node *bus,
39. 				  const struct of_device_id *matches,
40. 				  const struct of_dev_auxdata *lookup,
41. 				  struct device *parent, bool strict)
42. {
43. 	const struct of_dev_auxdata *auxdata;
44. 	struct device_node *child;
45. 	struct platform_device *dev;
46. 	const char *bus_id = NULL;
47. 	void *platform_data = NULL;
48. 	int rc = 0;

50. 	/* 只有包含"compatible"属性的node节点才会生成相应的platform_device结构体 */
51. 	/* Make sure it has a compatible property */
52. 	if (strict && (!of_get_property(bus, "compatible", NULL))) {
53. 		return 0;
54. 	}
55. 	/* 省略部分代码 */
56. 	/* 
57. 	 * 针对节点下面得到status = "ok" 或者status = "okay"或者不存在status属性的
58. 	 * 节点分配内存并填充platform_device结构体
59. 	 */
60. 	dev = of_platform_device_create_pdata(bus, bus_id, platform_data, parent);
61. 	if (!dev || !of_match_node(matches, bus))
62. 		return 0;

64. 	/* 递归调用节点解析函数，为子节点继续生成platform_device结构体，前提是父节点
65. 	 * 的“compatible” = “simple-bus”，也就是匹配of_default_bus_match_table结构体中的数据
66. 	 */
67. 	for_each_child_of_node(bus, child) {
68. 		rc = of_platform_bus_create(child, matches, lookup, &dev->dev, strict);
69. 		if (rc) {
70. 			of_node_put(child);
71. 			break;
72. 		}
73. 	}
74. 	of_node_set_flag(bus, OF_POPULATED_BUS);
75. 	return rc;
76. }
```

总的来说，当of_platform_populate()函数执行完毕，kernel就为DTB中所有包含compatible属性名的第一级node创建platform_device结构体，并向平台设备总线注册设备信息。如果第一级node的compatible属性值等于“simple-bus”、“simple-mfd”或者"arm,amba-bus"的话，kernel会继续为当前node的第二级包含compatible属性的node创建platform_device结构体，并注册设备。Linux系统下的设备大多都是挂载在平台总线下的，因此在平台总线被注册后，会根据of_root节点的树结构，去寻找该总线的子节点，所有的子节点将被作为设备注册到该总线上。\
**7. i2c_client和device_node绑定**\
经过customize_machine()函数的初始化，DTB已经转换成platform_device结构体，这其中就包含i2c adapter设备，不同的SoC需要通过平台设备总线的方式自己实现i2c adapter设备的驱动。例如：i2c_adapter驱动的probe函数中会调用i2c_add_numbered_adapter()注册adapter驱动，函数流执行如下图所示。

[![](http://www.wowotech.net/content/uploadfile/201709/6d731506232188.png)](http://www.wowotech.net/content/uploadfile/201709/6d731506232188.png)

在of_i2c_register_devices()函数内部便利i2c节点下面的每一个子节点，并为子节点（status = “disable”的除外）创建i2c_client结构体，并与子节点的device_node挂接。其中i2c_client的填充是在i2c_new_device()中进行的，最后device_register()。在构建i2c_client的时候，会对node下面的compatible属性名称的厂商名字去除作为i2c_client的name。例如：compatible = “maxim,ds1338”,则i2c_client->name = “ds1338”。\
**8. Device_Tree与sysfs**\
kernel启动流程为start_kernel()→rest_init()→kernel_thread():kernel_init()→do_basic_setup()→driver_init()→of_core_init()，在of_core_init()函数中在sys/firmware/devicetree/base目录下面为设备树展开成sysfs的目录和二进制属性文件，所有的node节点就是一个目录，所有的property属性就是一个二进制属性文件。

_原创文章，转发请注明出处。蜗窝科技，[www.wowotech.net](http://www.wowotech.net/armv8a_arch/arm_concept.html)。_

标签: [设备树](http://www.wowotech.net/tag/%E8%AE%BE%E5%A4%87%E6%A0%91)

______________________________________________________________________

« [X-026-KERNEL-Linux gpio driver的移植之gpio range](http://www.wowotech.net/x_project/kernel_gpio_driver_porting_2.html) | [X-025-KERNEL-Linux gpio driver的移植之基本功能](http://www.wowotech.net/x_project/kernel_gpio_driver_porting_1.html)»

**评论：**

**orange**\
2024-07-31 10:51

点赞点赞，对linuxer的文章总结到位

[回复](http://www.wowotech.net/device_model/dt-code-file-struct-parse.html#comment-8917)

**时文杰**\
2020-10-22 15:05

在经过DTC工具编译之后，&node1会变成一个特殊的整型数字n，假设n值为1，那么在node@1节点下自动生成两个属性，属性如下：\
linux,phandle = \<0x00000001>;\
phandle = \<0x00000001>;

使用fdtdump dump之后发现只添加了phandle = \<0x00000001>这一条property;没有linux,phandle = \<0x00000001>;这个property

[回复](http://www.wowotech.net/device_model/dt-code-file-struct-parse.html#comment-8129)

**Warren**\
2020-01-20 11:16

写得很棒，感谢分享。\
有个地方需要交流一下哈，i2c_adapter是依赖于i2c platform_driver的probe函数创建的,i2c_adapter本身是注册在i2c_bus上的，没有在platform_bus上注册设备驱动，并且没有在i2c_bus上注册相应的驱动(i2c_core.c中的i2c_device_probe()有一条判断if (!client) return 0;说明不需要为i2c_adapter匹配驱动)，厂商只需要实现i2c_adapter的i2c_algorithm方法，通过i2c_client->adapter访问i2c_adapter就能实现i2c协议的数据传输。

[回复](http://www.wowotech.net/device_model/dt-code-file-struct-parse.html#comment-7855)

**mating**\
2019-08-21 19:49

dts makefile会包含很多dtbo,现在应该只会选择一个dtb，编译时如何确定用哪个dtb的

[回复](http://www.wowotech.net/device_model/dt-code-file-struct-parse.html#comment-7606)

**[smcdef](http://www.wowotech.net/)**\
2019-08-21 19:57

@mating：编译的时候可以编译多个 dtb，不用编译器选择只编译哪一个。具体使用哪个 dtb 是由 BootLoader 根据当前硬件情况判断，最终选择一个合适的 dtb，直接将地址传给 kernel。

[回复](http://www.wowotech.net/device_model/dt-code-file-struct-parse.html#comment-7608)

**idyll**\
2019-12-10 16:48

@mating：应该是配置CONFIG_DEFAULT_DEVICE_TREE

[回复](http://www.wowotech.net/device_model/dt-code-file-struct-parse.html#comment-7783)

**xtzt**\
2018-04-19 10:03

“stsrt_kernel()函数”\
应该是“start_kernel()函数”

[回复](http://www.wowotech.net/device_model/dt-code-file-struct-parse.html#comment-6681)

**[smcdef](http://www.wowotech.net/)**\
2018-04-30 23:52

@xtzt：谢谢。笔误

[回复](http://www.wowotech.net/device_model/dt-code-file-struct-parse.html#comment-6724)

**kang**\
2019-01-18 20:43

@smcdef：似乎没改完...

[回复](http://www.wowotech.net/device_model/dt-code-file-struct-parse.html#comment-7142)

**[smcdef](http://www.wowotech.net/)**\
2019-01-18 20:54

@kang：很明显的错误，大家都知道。哈哈

[回复](http://www.wowotech.net/device_model/dt-code-file-struct-parse.html#comment-7143)

**xtzt**\
2018-04-19 09:50

“an-empty-property”\
图片中写成“an-cmpty-property”

[回复](http://www.wowotech.net/device_model/dt-code-file-struct-parse.html#comment-6680)

**wason**\
2018-02-27 16:51

非常棒的文章，谢谢分享！

[回复](http://www.wowotech.net/device_model/dt-code-file-struct-parse.html#comment-6569)

**[schedule](http://www.wowotech.net/)**\
2017-10-12 10:22

ACPI 比dts 灵活多了。

[回复](http://www.wowotech.net/device_model/dt-code-file-struct-parse.html#comment-6104)

**[smcdef](http://www.wowotech.net/)**\
2018-01-19 20:09

@schedule：兄台可否与大家分享一下ACPI的相关文章呢？

[回复](http://www.wowotech.net/device_model/dt-code-file-struct-parse.html#comment-6490)

**[zeroway](http://www.github.com/54shady)**\
2017-09-30 23:00

画图工具感觉不错，不知道用的是什么？

[回复](http://www.wowotech.net/device_model/dt-code-file-struct-parse.html#comment-6087)

**[smcdef](http://www.wowotech.net/)**\
2017-10-04 10:22

@zeroway：微软visio

[回复](http://www.wowotech.net/device_model/dt-code-file-struct-parse.html#comment-6089)

**u-boter**\
2017-09-24 16:46

# 其中还提供了一个fdtdump的工具，可以反编译dtb文件……

这一段的表述（以及后面那个图），有些歧义，其实dts的编译和反编译都是都是通过dtc来完成的：\
编译----dtc -i dts -o dtb\
反编译--dtc -i dtb -o dts\
至于fdtdump，正如它的名字，应该类似hexdump之类的，只是一个查看工具而已。

[回复](http://www.wowotech.net/device_model/dt-code-file-struct-parse.html#comment-6065)

**[smcdef](http://www.wowotech.net/)**\
2017-09-24 19:19

@u-boter：是的，是我表述的有点歧义！谢谢指正！dtc工具是可以双向编译的。待会修改…

[回复](http://www.wowotech.net/device_model/dt-code-file-struct-parse.html#comment-6066)

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

  - [linux kernel内存碎片防治技术](http://www.wowotech.net/memory_management/memory-fragment.html)
  - [小printf大作用（用日志打印的方式调试程序）](http://www.wowotech.net/soft/7.html)
  - [蓝牙协议分析(10)\_BLE安全机制之LE Encryption](http://www.wowotech.net/bluetooth/le_encryption.html)
  - [CFS任务放置代码详解](http://www.wowotech.net/process_management/task_placement_detail.html)
  - [PELT算法浅析](http://www.wowotech.net/process_management/pelt.html)

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
