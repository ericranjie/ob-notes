
原创 伟林 Linux阅码场

 _2022年02月22日 08:00_

伟林，中年码农，从事过电信、手机、安全、芯片等行业，目前依旧从事Linux方向开发工作，个人爱好Linux相关知识分享，个人微博CSDN pwl999，欢迎大家关注！

  

![](https://res.wx.qq.com/op_res/NN_GToMiIjsXzgPzF9-74ZzwR3cA9-fv3o9eWo8f5gQWqx71CmGlY8kFxuIxZaG0TB1bFeMCmh1DGN_pWMRg0A)

**帐号已迁移**

2篇原创内容

公众号

  

**文章目录**

**1、bpf()系统调用**

**1.1、bpf加载**

1.1.1、bpf内存空间分配

1.1.2、bpf verifier

1.1.3、bpf JIT/kernel interpreter

1.1.4、fd分配

**1.2、bpf map操作**

1.2.1、map的创建

1.2.2、map的查找

1.2.3、BPF_FUNC_map_lookup_elem

**1.3、obj pin**

1.3.1、bpf_obj_pin()

1.3.2、bpf_obj_get()

**2、Tracing类型的BPF程序**

**2.1、bpf程序的绑定**

**2.2、bpf程序的执行**

**3、Filter类型的BPF程序**

  

  

BPF的字面上意思Berkeley Packet Filter意味着它是从包过滤而来。如果在开始前对BPF缺乏感性的认识建议先看一下参考文档：“3.1、Berkeley Packet Filter (BPF) (Kernel Document)”、“3.2、BPF and XDP Reference Guide”。  

  

本质上它是一种**内核代码注入**的技术：

  

- 内核中实现了一个cBPF/eBPF虚拟机；
    
- 用户态可以用C来写运行的代码，再通过一个Clang&LLVM的编译器将C代码编译成BPF目标码；
    
- 用户态通过系统调用bpf()将BPF目标码注入到内核当中；
    
- 内核通过JIT(Just-In-Time)将BPF目编码转换成本地指令码；如果当前架构不支持JIT转换内核则会使用一个解析器(interpreter)来模拟运行，这种运行效率较低；
    
- 内核在packet filter和tracing等应用中提供了一系列的钩子来运行BPF代码。目前支持以下类型的BPF代码：
    
      
    

```c
static int __init register_kprobe_prog_ops(void) {   bpf_register_prog_type(&kprobe_tl);   bpf_register_prog_type(&tracepoint_tl);   bpf_register_prog_type(&perf_event_tl);   return 0; }  static int __init register_sk_filter_ops(void) {   bpf_register_prog_type(&sk_filter_type);   bpf_register_prog_type(&sched_cls_type);   bpf_register_prog_type(&sched_act_type);   bpf_register_prog_type(&xdp_type);   bpf_register_prog_type(&cg_skb_type);    return 0; }
```

  

BPF的好处在哪里？**是因为它提供了一种在不修改内核代码的情况下，可以灵活修改内核处理策略的方法。**

这在包过滤和系统tracing这种需要频繁修改规则的场合非常有用。因为如果只在用户态修改策略的话那么所有数据需要复制一份给用户态开销较大；如果在内核态修改策略的话需要修改内核代码重新编译内核，而且容易引人安全问题。BPF这种内核代码注入技术的生存空间就是它可以在这两者间取得一个平衡。

  

Systamp就是解决了这个问题得以发展的，它使用了ko的方式来实现内核代码注入(有点笨拙，但是也解决了实际问题)。

  

**Systemtap****工作原理：**是通过将脚本语句翻译成C语句，编译成内核模块。模块加载之后，将所有探测的事件以Kprobe钩子的方式挂到内核上，当任何处理器上的某个事件发生时，相应钩子上句柄就会被执行。最后，当systemtap会话结束之后，钩子从内核上取下，移除模块。整个过程用一个命令stap就可以完成。

  

既然是提供向内核注入代码的技术，那么安全问题肯定是重中之重。平时防范他人通过漏洞向内核中注入代码，这下子专门开了一个口子不是大开方便之门。所以内核指定了很多的规则来限制BPF代码，确保它的错误不会影响到内核：

  

- 一个BPF程序的代码数量不能超过BPF_MAXINSNS (4K)，它的总运行步数不能超过32K (4.9内核中这个值改成了96k)；
    
- BPF代码中禁止循环，这也是为了保证出错时不会出现死循环来hang死内核。一个BPF程序总的可能的分支数也被限制到1K；
    
- 为了限制它的作用域，BPF代码不能访问全局变量，只能访问局部变量。一个BPF程序只有512字节的堆栈。在开始时会传入一个ctx指针，BPF程序的数据访问就被限制在ctx变量和堆栈局部变量中；
    
- 如果BPF需要访问全局变量，它只能访问BPF map对象。BPF map对象是同时能被用户态、BPF程序、内核态共同访问的，BPF对map的访问通过helper function来实现；
    
- 旧版本BPF代码中不支持BPF对BPF函数的调用，所以所有的BPF函数必须声明成always_inline。在Linux内核4.16和LLVM 6.0以后，才支持BPF to BPF Calls；
    
- BPF虽然不能函数调用，但是它可以使用Tail Call机制从一个BPF程序直接跳转到另一个BPF程序。它需要通过BPF_MAP_TYPE_PROG_ARRAY类型的map来知道另一个BPF程序的指针。这种跳转的次数也是有限制的，32次；
    
- BPF程序可以调用一些内核函数来辅助做一些事情(helper function)；
    
- 有些架构(64 bit x86_64, arm64, ppc64, s390x, mips64, sparc64 and 32 bit arm)已经支持BPF的JIT，它可以高效的几乎一比一的把BPF代码转换成本机代码(因为eBPF的指令集已经做了优化，非常类似最新的arm/x86架构，ABI也类似)。如果当前架构不支持JTI只能使用内核的解析器(interpreter)来模拟运行；
    
- 内核还可以通过一些额外的手段来加固BPF的安全性(Hardening)。主要包括：把BPF代码映像和JIT代码映像的page都锁成只读，JIT编译时把常量致盲(constant blinding)，以及对bpf()系统调用的权限限制；
    

  

对BPF这些安全规则的检查主要是在BPF代码加载时，通过BPF verifier来实现的。大概分为两步：

  

- **第一步**，通过DAG(Directed Acyclic Graph 有向无环图)的DFS(Depth-first Search)深度优先算法来遍历BPF程序的代码路径，确保没有环路发生；
    
- **第二步**，逐条分析BPF每条指令的运行，对register和对stack的影响，最坏情况下是否有越界行为(对变量的访问是否越界，运行的指令数是否越界)。这里也有一个快速分析的优化方法：修剪(Pruning)。如果当前指令的当前分支的状态，和当前指令另一个已分析分支的状态相等或者是它的一个子集，那么当前指令的当前分支就不需要分析了，因为它肯定是符合规则的。
    

  

整个BPF的开发过程大概如下图所示：

  
![[Pasted image 20240920131635.png]]
![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

## 1.bpf()系统调用

  
核心代码在bpf()系统调用中，我们从入口开始分析。

  

```c
SYSCALL_DEFINE3(bpf, int, cmd, union bpf_attr __user *, uattr, unsigned int, size) {   union bpf_attr attr = {};   int err;    if (!capable(CAP_SYS_ADMIN) && sysctl_unprivileged_bpf_disabled)     return -EPERM;    if (!access_ok(VERIFY_READ, uattr, 1))     return -EFAULT;    if (size > PAGE_SIZE)  /* silly large */     return -E2BIG;    /* If we're handed a bigger struct than we know of,    * ensure all the unknown bits are 0 - i.e. new    * user-space does not rely on any kernel feature    * extensions we dont know about yet.    */   if (size > sizeof(attr)) {     unsigned char __user *addr;     unsigned char __user *end;     unsigned char val;      addr = (void __user *)uattr + sizeof(attr);     end  = (void __user *)uattr + size;      for (; addr < end; addr++) {       err = get_user(val, addr);       if (err)         return err;       if (val)         return -E2BIG;     }     size = sizeof(attr);   }    /* copy attributes from user space, may be less than sizeof(bpf_attr) */   if (copy_from_user(&attr, uattr, size) != 0)     return -EFAULT;    switch (cmd) {   case BPF_MAP_CREATE:     err = map_create(&attr);     break;   case BPF_MAP_LOOKUP_ELEM:     err = map_lookup_elem(&attr);     break;   case BPF_MAP_UPDATE_ELEM:     err = map_update_elem(&attr);     break;   case BPF_MAP_DELETE_ELEM:     err = map_delete_elem(&attr);     break;   case BPF_MAP_GET_NEXT_KEY:     err = map_get_next_key(&attr);     break;   case BPF_PROG_LOAD:     err = bpf_prog_load(&attr);     break;   case BPF_OBJ_PIN:     err = bpf_obj_pin(&attr);     break;   case BPF_OBJ_GET:     err = bpf_obj_get(&attr);     break;  #ifdef CONFIG_CGROUP_BPF   case BPF_PROG_ATTACH:     err = bpf_prog_attach(&attr);     break;   case BPF_PROG_DETACH:     err = bpf_prog_detach(&attr);     break; #endif    default:     err = -EINVAL;     break;   }    return err; }
```

  

## 1.1、bpf加载

BPF_PROG_LOAD命令负责加载一段BPF程序到内核当中：

- 拷贝程序到内核；
    
- 校验它的安全性；
    
- 如果可能对它进行JIT编译；
    
- 然后分配一个文件句柄fd给它。
    

完成这一切后，后续再把这段BPF程序挂载到需要运行的钩子上面。

  

### 1.1.1、bpf内存空间分配

  

```c
static int bpf_prog_load(union bpf_attr *attr) {   enum bpf_prog_type type = attr->prog_type;   struct bpf_prog *prog;   int err;   char license[128];   bool is_gpl;    if (CHECK_ATTR(BPF_PROG_LOAD))     return -EINVAL;    /* copy eBPF program license from user space */   /* (1.1) 根据attr->license地址，从用户空间拷贝license字符串到内核 */   if (strncpy_from_user(license, u64_to_ptr(attr->license),             sizeof(license) - 1) < 0)     return -EFAULT;   license[sizeof(license) - 1] = 0;    /* eBPF programs must be GPL compatible to use GPL-ed functions */   /* (1.2) 判断license是否符合GPL协议 */   is_gpl = license_is_gpl_compatible(license);      /* (1.3) 判断BPF的总指令数是否超过BPF_MAXINSNS(4k) */   if (attr->insn_cnt >= BPF_MAXINSNS)     return -EINVAL;      /* (1.4) 如果加载BPF_PROG_TYPE_KPROBE类型的BPF程序，指定的内核版本需要和当前内核版本匹配。         不然由于内核的改动，可能会附加到错误的地址上。      */   if (type == BPF_PROG_TYPE_KPROBE &&       attr->kern_version != LINUX_VERSION_CODE)     return -EINVAL;      /* (1.5) 对BPF_PROG_TYPE_SOCKET_FILTER和BPF_PROG_TYPE_CGROUP_SKB以外的BPF程序加载，需要管理员权限 */   if (type != BPF_PROG_TYPE_SOCKET_FILTER &&       type != BPF_PROG_TYPE_CGROUP_SKB &&       !capable(CAP_SYS_ADMIN))     return -EPERM;    /* plain bpf_prog allocation */   /* (2.1) 根据BPF指令数分配bpf_prog空间，和bpf_prog->aux空间 */   prog = bpf_prog_alloc(bpf_prog_size(attr->insn_cnt), GFP_USER);   if (!prog)     return -ENOMEM;      /* (2.2) 把整个bpf_prog空间在当前进程的memlock_limit中锁定 */   err = bpf_prog_charge_memlock(prog);   if (err)     goto free_prog_nouncharge;    prog->len = attr->insn_cnt;    err = -EFAULT;   /* (2.3) 把BPF代码从用户空间地址attr->insns，拷贝到内核空间地址prog->insns */   if (copy_from_user(prog->insns, u64_to_ptr(attr->insns),          prog->len * sizeof(struct bpf_insn)) != 0)     goto free_prog;    prog->orig_prog = NULL;   prog->jited = 0;    atomic_set(&prog->aux->refcnt, 1);   prog->gpl_compatible = is_gpl ? 1 : 0;    /* find program type: socket_filter vs tracing_filter */   /* (2.4) 根据attr->prog_type指定的type值，找到对应的bpf_prog_types，       给bpf_prog->aux->ops赋值，这个ops是一个函数操作集    */   err = find_prog_type(type, prog);   if (err < 0)     goto free_prog;    /* run eBPF verifier */   /* (3) 使用verifer对BPF程序进行合法性扫描 */   err = bpf_check(&prog, attr);   if (err < 0)     goto free_used_maps;    /* eBPF program is ready to be JITed */   /* (4) 尝试对BPF程序进行JIT转换 */   prog = bpf_prog_select_runtime(prog, &err);   if (err < 0)     goto free_used_maps;      /* (5) 给BPF程序分配一个文件句柄fd */   err = bpf_prog_new_fd(prog);   if (err < 0)     /* failed to allocate fd */     goto free_used_maps;    return err;  free_used_maps:   free_used_maps(prog->aux); free_prog:   bpf_prog_uncharge_memlock(prog); free_prog_nouncharge:   bpf_prog_free(prog);   return err; }
```

  

这其中对BPF来说有个重要的数据结构就是struct bpf_prog：

  

```c
struct bpf_prog {   u16      pages;    /* Number of allocated pages */   kmemcheck_bitfield_begin(meta);   u16      jited:1,  /* Is our filter JIT'ed? */         gpl_compatible:1, /* Is filter GPL compatible? */         cb_access:1,  /* Is control block accessed? */         dst_needed:1;  /* Do we need dst entry? */   kmemcheck_bitfield_end(meta);   u32      len;    /* Number of filter blocks */   enum bpf_prog_type  type;    /* Type of BPF program */   struct bpf_prog_aux  *aux;    /* Auxiliary fields */   struct sock_fprog_kern  *orig_prog;  /* Original BPF program */   unsigned int    (*bpf_func)(const struct sk_buff *skb,               const struct bpf_insn *filter);   /* Instructions for interpreter */   union {     struct sock_filter  insns[0];     struct bpf_insn    insnsi[0];   }; };
```

  

其中重要的成员如下：

  

- **len**：程序包含bpf指令的数量；
    
- **type**：当前bpf程序的类型(kprobe/tracepoint/perf_event/sk_filter/sched_cls/sched_act/xdp/cg_skb)；
    
- **aux**：主要用来辅助verifier校验和转换的数据；
    
- **orig_prog**：
    
- **bpf_func**：运行时BPF程序的入口。如果JIT转换成功，这里指向的就是BPF程序JIT转换后的映像；否则这里指向内核解析器(interpreter)的通用入口__bpf_prog_run()；
    
- **insnsi[]**：从用户态拷贝过来的，BPF程序原始指令的存放空间；
    

  

1.1.2、bpf verifier

  

关于verifier的步骤和规则，在“3.1、Berkeley Packet Filter (BPF) (Kernel Document)”一文的“eBPF verifier”一节有详细描述。

  

另外，在kernel/bpf/verifier.c文件的开头对eBPF verifier也有一段详细的注释：

  

```c
bpf_check()是一个静态代码分析器，它按指令遍历eBPF程序指令并更新寄存器/堆栈状态。分析条件分支的所有路径，直到'bpf_exit'指令。  1、第一步是深度优先搜索，检查程序是否为DAG(Directed Acyclic Graph 有向无环图)。它将会拒绝以下程序：   - 大于BPF_MAXINSNS条指令(BPF_MAXINSNS=4096)  - 如果出现循环(通过back-edge检测)  - 不可达的指令存在(不应该是森林，程序等于一个函数)  - 越界或畸形的跳跃  2、第二步是从第一步所有可能路径的展开。  - 因为它分析了程序所有的路径，这个分析的最大长度限制为32k个指令，即使指令总数小于4k也会受到影响，因为有太多的分支改变了堆栈/寄存器。 - 分支的分析数量被限制为1k。  在进入每条指令时，每个寄存器都有一个类型，该指令根据指令语义改变寄存器的类型：  - rule 1、如果指令是BPF_MOV64_REG(BPF_REG_1, BPF_REG_5)，则将R5的类型复制到R1。  所有寄存器都是64位的。 * R0 -返回寄存器   * R1-R5参数传递寄存器   * R6-R9被调用方保存寄存器   * R10 -帧指针只读    - rule 2、在BPF程序开始时，寄存器R1包含一个指向bpf_context的指针，类型为PTR_TO_CTX。  - rule 3、verifier跟踪指针上的算术运算：  `     BPF_MOV64_REG(BPF_REG_1, BPF_REG_10),     BPF_ALU64_IMM(BPF_ADD, BPF_REG_1, -20), `  第一条指令将R10(它具有FRAME_PTR)类型复制到R1中，第二条算术指令是匹配的模式，用于识别它想要构造一个指向堆栈中某个元素的指针。 因此，在第二条指令之后，寄存器R1的类型为PTR_TO_STACK(-20常数需要进一步的堆栈边界检查)。表示这个reg是一个指针由堆栈加上常数。  - rule 4、大多数时候寄存器都有UNKNOWN_VALUE类型，这意味着寄存器有一些值，但它不是一个有效的指针。(就像指针+指针变成了UNKNOWN_VALUE类型)  - rule 5、当verifier看到load指令或store指令时，基本寄存器的类型可以是:PTR_TO_MAP_VALUE、PTR_TO_CTX、FRAME_PTR。这是由check_mem_access()函数识别的三种指针类型。  - rule 6、PTR_TO_MAP_VALUE表示这个寄存器指向‘map元素的值’，并且可以访问[ptr, ptr + map value_size)的范围。  - rule 7、寄存器用于向函数调用传递参数，将根据函数参数约束进行检查。  ARG_PTR_TO_MAP_KEY就是这样的参数约束之一。 这意味着传递给这个函数的寄存器类型必须是PTR_TO_STACK，它将作为‘map element key的指针’在函数内部使用。  例如bpf_map_lookup_elem()的参数约束:  `    .ret_type = RET_PTR_TO_MAP_VALUE_OR_NULL,    .arg1_type = ARG_CONST_MAP_PTR,    .arg2_type = ARG_PTR_TO_MAP_KEY, `  ret_type表示该函数返回“指向map element value的指针或null”。 函数期望第一个参数是指向‘struct bpf_map’的const指针，第二个参数应该是指向stack的指针，这个指针在helper函数中用作map element key的指针。  在内核侧的helper函数如下：  `  u64 bpf_map_lookup_elem(u64 r1, u64 r2, u64 r3, u64 r4, u64 r5)  {     struct bpf_map *map = (struct bpf_map *) (unsigned long) r1;     void *key = (void *) (unsigned long) r2;     void *value;      here kernel can access 'key' and 'map' pointers safely, knowing that     [key, key + map->key_size) bytes are valid and were initialized on     the stack of eBPF program.  } `  相应的eBPF程序如下：  `     BPF_MOV64_REG(BPF_REG_2, BPF_REG_10),  // after this insn R2 type is FRAME_PTR     BPF_ALU64_IMM(BPF_ADD, BPF_REG_2, -4), // after this insn R2 type is PTR_TO_STACK     BPF_LD_MAP_FD(BPF_REG_1, map_fd),      // after this insn R1 type is CONST_PTR_TO_MAP     BPF_RAW_INSN(BPF_JMP | BPF_CALL, 0, 0, 0, BPF_FUNC_map_lookup_elem), `  这里verifier查看map_lookup_elem()的原型，看到:    - .arg1_type == ARG_CONST_MAP_PTR and R1->type == CONST_PTR_TO_MAP, 这个是ok的。现在verifier知道map key的尺寸了：R1->map_ptr->key_size。  - 然后.arg2_type == ARG_PTR_TO_MAP_KEY and R2->type == PTR_TO_STACK也是ok的。 现在verifier检测 [R2, R2 + map's key_size]是否在堆栈限制内，并且在调用之前被初始化。  - 如果可以，那么verifier允许这个BPF_CALL指令，并查看.ret_type  RET_PTR_TO_MAP_VALUE_OR_NULL，因此它设置R0->类型= PTR_TO_MAP_VALUE_OR_NULL，这意味着bpf_map_lookup_elem()函数返回map value指针或NULL。 当类型PTR_TO_MAP_VALUE_OR_NULL通过'if (reg != 0) goto +off' 指令判断时，在真分支中持有指针的寄存器将状态更改为PTR_TO_MAP_VALUE，在假分支中相同的寄存器将状态更改为CONST_IMM。看check_cond_jmp_op()的实现。 函数调用以后R0设置为返回函数类型后，将寄存器R1-R5设置为NOT_INIT，以指示它们不再可读。
```

  

原文如下：

  

```c
/* bpf_check() is a static code analyzer that walks eBPF program  * instruction by instruction and updates register/stack state.  * All paths of conditional branches are analyzed until 'bpf_exit' insn.  *  * The first pass is depth-first-search to check that the program is a DAG.  * It rejects the following programs:  * - larger than BPF_MAXINSNS insns  * - if loop is present (detected via back-edge)  * - unreachable insns exist (shouldn't be a forest. program = one function)  * - out of bounds or malformed jumps  * The second pass is all possible path descent from the 1st insn.  * Since it's analyzing all pathes through the program, the length of the  * analysis is limited to 32k insn, which may be hit even if total number of  * insn is less then 4K, but there are too many branches that change stack/regs.  * Number of 'branches to be analyzed' is limited to 1k  *  * On entry to each instruction, each register has a type, and the instruction  * changes the types of the registers depending on instruction semantics.  * If instruction is BPF_MOV64_REG(BPF_REG_1, BPF_REG_5), then type of R5 is  * copied to R1.  *  * All registers are 64-bit.  * R0 - return register  * R1-R5 argument passing registers  * R6-R9 callee saved registers  * R10 - frame pointer read-only  *  * At the start of BPF program the register R1 contains a pointer to bpf_context  * and has type PTR_TO_CTX.  *  * Verifier tracks arithmetic operations on pointers in case:  *    BPF_MOV64_REG(BPF_REG_1, BPF_REG_10),  *    BPF_ALU64_IMM(BPF_ADD, BPF_REG_1, -20),  * 1st insn copies R10 (which has FRAME_PTR) type into R1  * and 2nd arithmetic instruction is pattern matched to recognize  * that it wants to construct a pointer to some element within stack.  * So after 2nd insn, the register R1 has type PTR_TO_STACK  * (and -20 constant is saved for further stack bounds checking).  * Meaning that this reg is a pointer to stack plus known immediate constant.  *  * Most of the time the registers have UNKNOWN_VALUE type, which  * means the register has some value, but it's not a valid pointer.  * (like pointer plus pointer becomes UNKNOWN_VALUE type)  *  * When verifier sees load or store instructions the type of base register  * can be: PTR_TO_MAP_VALUE, PTR_TO_CTX, FRAME_PTR. These are three pointer  * types recognized by check_mem_access() function.  *  * PTR_TO_MAP_VALUE means that this register is pointing to 'map element value'  * and the range of [ptr, ptr + map's value_size) is accessible.  *  * registers used to pass values to function calls are checked against  * function argument constraints.  *  * ARG_PTR_TO_MAP_KEY is one of such argument constraints.  * It means that the register type passed to this function must be  * PTR_TO_STACK and it will be used inside the function as  * 'pointer to map element key'  *  * For example the argument constraints for bpf_map_lookup_elem():  *   .ret_type = RET_PTR_TO_MAP_VALUE_OR_NULL,  *   .arg1_type = ARG_CONST_MAP_PTR,  *   .arg2_type = ARG_PTR_TO_MAP_KEY,  *  * ret_type says that this function returns 'pointer to map elem value or null'  * function expects 1st argument to be a const pointer to 'struct bpf_map' and  * 2nd argument should be a pointer to stack, which will be used inside  * the helper function as a pointer to map element key.  *  * On the kernel side the helper function looks like:  * u64 bpf_map_lookup_elem(u64 r1, u64 r2, u64 r3, u64 r4, u64 r5)  * {  *    struct bpf_map *map = (struct bpf_map *) (unsigned long) r1;  *    void *key = (void *) (unsigned long) r2;  *    void *value;  *  *    here kernel can access 'key' and 'map' pointers safely, knowing that  *    [key, key + map->key_size) bytes are valid and were initialized on  *    the stack of eBPF program.  * }  *  * Corresponding eBPF program may look like:  *    BPF_MOV64_REG(BPF_REG_2, BPF_REG_10),  // after this insn R2 type is FRAME_PTR  *    BPF_ALU64_IMM(BPF_ADD, BPF_REG_2, -4), // after this insn R2 type is PTR_TO_STACK  *    BPF_LD_MAP_FD(BPF_REG_1, map_fd),      // after this insn R1 type is CONST_PTR_TO_MAP  *    BPF_RAW_INSN(BPF_JMP | BPF_CALL, 0, 0, 0, BPF_FUNC_map_lookup_elem),  * here verifier looks at prototype of map_lookup_elem() and sees:  * .arg1_type == ARG_CONST_MAP_PTR and R1->type == CONST_PTR_TO_MAP, which is ok,  * Now verifier knows that this map has key of R1->map_ptr->key_size bytes  *  * Then .arg2_type == ARG_PTR_TO_MAP_KEY and R2->type == PTR_TO_STACK, ok so far,  * Now verifier checks that [R2, R2 + map's key_size) are within stack limits  * and were initialized prior to this call.  * If it's ok, then verifier allows this BPF_CALL insn and looks at  * .ret_type which is RET_PTR_TO_MAP_VALUE_OR_NULL, so it sets  * R0->type = PTR_TO_MAP_VALUE_OR_NULL which means bpf_map_lookup_elem() function  * returns ether pointer to map value or NULL.  *  * When type PTR_TO_MAP_VALUE_OR_NULL passes through 'if (reg != 0) goto +off'  * insn, the register holding that pointer in the true branch changes state to  * PTR_TO_MAP_VALUE and the same register changes state to CONST_IMM in the false  * branch. See check_cond_jmp_op().  *  * After the call R0 is set to return type of the function and registers R1-R5  * are set to NOT_INIT to indicate that they are no longer readable.  */
```

  

BPF verifier总体代码流程如下：

  

```c
int bpf_check(struct bpf_prog **prog, union bpf_attr *attr) {   char __user *log_ubuf = NULL;   struct bpf_verifier_env *env;   int ret = -EINVAL;    if ((*prog)->len <= 0 || (*prog)->len > BPF_MAXINSNS)     return -E2BIG;    /* 'struct bpf_verifier_env' can be global, but since it's not small,    * allocate/free it every time bpf_check() is called    */   /* (3.1) 分配verifier静态扫描需要的数据结构  */   env = kzalloc(sizeof(struct bpf_verifier_env), GFP_KERNEL);   if (!env)     return -ENOMEM;    env->insn_aux_data = vzalloc(sizeof(struct bpf_insn_aux_data) *              (*prog)->len);   ret = -ENOMEM;   if (!env->insn_aux_data)     goto err_free_env;   env->prog = *prog;    /* grab the mutex to protect few globals used by verifier */   mutex_lock(&bpf_verifier_lock);      /* (3.2) 如果用户指定了attr->log_buf，说明用户需要具体的代码扫描log，这个在出错时非常有用          先在内核中分配log空间，在返回时拷贝给用户      */   if (attr->log_level || attr->log_buf || attr->log_size) {     /* user requested verbose verifier output      * and supplied buffer to store the verification trace      */     log_level = attr->log_level;     log_ubuf = (char __user *) (unsigned long) attr->log_buf;     log_size = attr->log_size;     log_len = 0;      ret = -EINVAL;     /* log_* values have to be sane */     if (log_size < 128 || log_size > UINT_MAX >> 8 ||         log_level == 0 || log_ubuf == NULL)       goto err_unlock;      ret = -ENOMEM;     log_buf = vmalloc(log_size);     if (!log_buf)       goto err_unlock;   } else {     log_level = 0;   }      /* (3.3) 把BPF程序中操作map的指令，从map_fd替换成实际的map指针          由此可见用户态的loader程序，肯定是先根据__section("maps")中定义的map调用bpf()创建map，再加载其他的程序section；      */   ret = replace_map_fd_with_map_ptr(env);   if (ret < 0)     goto skip_full_check;    env->explored_states = kcalloc(env->prog->len,                sizeof(struct bpf_verifier_state_list *),                GFP_USER);   ret = -ENOMEM;   if (!env->explored_states)     goto skip_full_check;      /* (3.4) step1、检查有没有环路 */   ret = check_cfg(env);   if (ret < 0)     goto skip_full_check;    env->allow_ptr_leaks = capable(CAP_SYS_ADMIN);      /* (3.5) step2、详细扫描BPF代码的运行过程，跟踪分析寄存器和堆栈，检查是否有不符合规则的情况出现 */   ret = do_check(env);  skip_full_check:   while (pop_stack(env, NULL) >= 0);   free_states(env);      /* (3.6) 把扫描分析出来的dead代码(就是不会运行的代码)转成nop指令 */   if (ret == 0)     sanitize_dead_code(env);      /* (3.7) 根据程序的type，转换对ctx指针成员的访问 */   if (ret == 0)     /* program is valid, convert *(u32*)(ctx + off) accesses */     ret = convert_ctx_accesses(env);      /* (3.8) 修复BPF指令中对内核helper function函数的调用，把函数编号替换成实际的函数指针 */   if (ret == 0)     ret = fixup_bpf_calls(env);    if (log_level && log_len >= log_size - 1) {     BUG_ON(log_len >= log_size);     /* verifier log exceeded user supplied buffer */     ret = -ENOSPC;     /* fall through to return what was recorded */   }      /* (3.9) 拷贝verifier log到用户空间 */   /* copy verifier log back to user space including trailing zero */   if (log_level && copy_to_user(log_ubuf, log_buf, log_len + 1) != 0) {     ret = -EFAULT;     goto free_log_buf;   }      /* (3.10) 备份BPF程序对map的引用信息，到prog->aux->used_maps中 */   if (ret == 0 && env->used_map_cnt) {     /* if program passed verifier, update used_maps in bpf_prog_info */     env->prog->aux->used_maps = kmalloc_array(env->used_map_cnt,                 sizeof(env->used_maps[0]),                 GFP_KERNEL);      if (!env->prog->aux->used_maps) {       ret = -ENOMEM;       goto free_log_buf;     }      memcpy(env->prog->aux->used_maps, env->used_maps,            sizeof(env->used_maps[0]) * env->used_map_cnt);     env->prog->aux->used_map_cnt = env->used_map_cnt;      /* program is valid. Convert pseudo bpf_ld_imm64 into generic      * bpf_ld_imm64 instructions      */     convert_pseudo_ld_imm64(env);   }  free_log_buf:   if (log_level)     vfree(log_buf);   if (!env->prog->aux->used_maps)     /* if we didn't copy map pointers into bpf_prog_info, release      * them now. Otherwise free_bpf_prog_info() will release them.      */     release_maps(env);   *prog = env->prog; err_unlock:   mutex_unlock(&bpf_verifier_lock);   vfree(env->insn_aux_data); err_free_env:   kfree(env);   return ret; }
```

  

- **1、把BPF程序中操作map的指令，从map_fd替换成实际的map指针。**
    
      
    

由此可见用户态的loader程序，肯定是先根据__section(“maps”)中定义的map调用bpf()创建map，再加载其他的程序section。

  

**符合条件**：(insn[0].code == (BPF_LD | BPF_IMM | BPF_DW)) && (insn[0]->src_reg == BPF_PSEUDO_MAP_FD) 的指令为map指针加载指针。

把原始的立即数作为fd找到对应的map指针。

把64bit的map指针拆分成两个32bit的立即数，存储到insn[0].imm、insn[1].imm中。

  

```c
static int replace_map_fd_with_map_ptr(struct bpf_verifier_env *env) {   struct bpf_insn *insn = env->prog->insnsi;   int insn_cnt = env->prog->len;   int i, j, err;      /* (3.3.1) 遍历所有BPF指令 */   for (i = 0; i < insn_cnt; i++, insn++) {     if (BPF_CLASS(insn->code) == BPF_LDX &&         (BPF_MODE(insn->code) != BPF_MEM || insn->imm != 0)) {       verbose("BPF_LDX uses reserved fields\n");       return -EINVAL;     }      if (BPF_CLASS(insn->code) == BPF_STX &&         ((BPF_MODE(insn->code) != BPF_MEM &&           BPF_MODE(insn->code) != BPF_XADD) || insn->imm != 0)) {       verbose("BPF_STX uses reserved fields\n");       return -EINVAL;     }          /* (3.3.2) 符合条件：(insn[0].code == (BPF_LD | BPF_IMM | BPF_DW)) && (insn[0]->src_reg == BPF_PSEUDO_MAP_FD)               的指令为map指针加载指针          */     if (insn[0].code == (BPF_LD | BPF_IMM | BPF_DW)) {       struct bpf_map *map;       struct fd f;        if (i == insn_cnt - 1 || insn[1].code != 0 ||           insn[1].dst_reg != 0 || insn[1].src_reg != 0 ||           insn[1].off != 0) {         verbose("invalid bpf_ld_imm64 insn\n");         return -EINVAL;       }        if (insn->src_reg == 0)         /* valid generic load 64-bit imm */         goto next_insn;        if (insn->src_reg != BPF_PSEUDO_MAP_FD) {         verbose("unrecognized bpf_ld_imm64 insn\n");         return -EINVAL;       }              /* (3.3.3) 根据指令中的立即数insn[0]->imm指定的fd，得到实际的map指针 */       f = fdget(insn->imm);       map = __bpf_map_get(f);       if (IS_ERR(map)) {         verbose("fd %d is not pointing to valid bpf_map\n",           insn->imm);         return PTR_ERR(map);       } 的·             /* (3.3.4) 检查map和当前类型BPF程序的兼容性 */       err = check_map_prog_compatibility(map, env->prog);       if (err) {         fdput(f);         return err;       }              /* (3.3.5) 把64bit的map指针拆分成两个32bit的立即数，存储到insn[0].imm、insn[1].imm中 */       /* store map pointer inside BPF_LD_IMM64 instruction */       insn[0].imm = (u32) (unsigned long) map;       insn[1].imm = ((u64) (unsigned long) map) >> 32;        /* check whether we recorded this map already */       for (j = 0; j < env->used_map_cnt; j++)         if (env->used_maps[j] == map) {           fdput(f);           goto next_insn;         }              /* (3.3.6) 一个prog最多引用64个map */       if (env->used_map_cnt >= MAX_USED_MAPS) {         fdput(f);         return -E2BIG;       }        /* hold the map. If the program is rejected by verifier,        * the map will be released by release_maps() or it        * will be used by the valid program until it's unloaded        * and all maps are released in free_bpf_prog_info()        */       map = bpf_map_inc(map, false);       if (IS_ERR(map)) {         fdput(f);         return PTR_ERR(map);       }       /* (3.3.7) 记录prog对map的引用 */       env->used_maps[env->used_map_cnt++] = map;        fdput(f); next_insn:       insn++;       i++;     }   }    /* now all pseudo BPF_LD_IMM64 instructions load valid    * 'struct bpf_map *' into a register instead of user map_fd.    * These pointers will be used later by verifier to validate map access.    */   return 0; }
```

  

- **2、Step 1、通过DAG(Directed Acyclic Graph 有向无环图)的**
    

  

DFS(Depth-first Search)深度优先算法来遍历BPF程序的代码路径，确保没有环路发生；

  

DAG的DFS算法可以参考“Graph”一文。其中最重要的概念如下图：

  
![[Pasted image 20240920132031.png]]
![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

一个图形"Graph"经过DAG的DFS算法遍历后，对每一个根节点都会形成一颗树“DFS Tree”，多个根节点得到的多棵树形成一个森林"DFS Forest"。根据搜索的结构整个“Graph”的边“Edge”可以**分成四类**：

  

- **Tree Edges**：在DFS树上的边；
    
- **Back Edges**：从子节点连向祖先节点的边(形成环)；
    
- **Forward Edges**：直接连向孙节点的边(跨子节点的连接)；
    
- **Cross Edges**：叶子之间的连接，或者树之间的连接；
    

  

对BPF verifier来说，检查BPF程序的运行路径图中是否有“Back Edges”的存在，确保程序中没有环路。

  

具体的代码如下：

  

```c
static int check_cfg(struct bpf_verifier_env *env) {   struct bpf_insn *insns = env->prog->insnsi;   int insn_cnt = env->prog->len;   int ret = 0;   int i, t;    insn_state = kcalloc(insn_cnt, sizeof(int), GFP_KERNEL);   if (!insn_state)     return -ENOMEM;    insn_stack = kcalloc(insn_cnt, sizeof(int), GFP_KERNEL);   if (!insn_stack) {     kfree(insn_state);     return -ENOMEM;   }    insn_state[0] = DISCOVERED; /* mark 1st insn as discovered */   insn_stack[0] = 0; /* 0 is the first instruction */   cur_stack = 1;      /* (3.4.1) DFS深度优先算法的循环 */ peek_stack:   if (cur_stack == 0)     goto check_state;   t = insn_stack[cur_stack - 1];      /* (3.4.2) 分支指令 */   if (BPF_CLASS(insns[t].code) == BPF_JMP) {     u8 opcode = BPF_OP(insns[t].code);          /* (3.4.2.1) 碰到BPF_EXIT指令，路径终结，开始回溯确认 */     if (opcode == BPF_EXIT) {       goto mark_explored;      /* (3.4.2.2) 碰到BPF_CALL指令，继续探索          并且把env->explored_states[]设置成STATE_LIST_MARK，标识call函数调用后需要重新跟踪计算寄存器和堆栈      */     } else if (opcode == BPF_CALL) {       ret = push_insn(t, t + 1, FALLTHROUGH, env);       if (ret == 1)         goto peek_stack;       else if (ret < 0)         goto err_free;       if (t + 1 < insn_cnt)         env->explored_states[t + 1] = STATE_LIST_MARK;      /* (3.4.2.3) 碰到BPF_JA指令，继续探索          并且把env->explored_states[]设置成STATE_LIST_MARK，标识call函数调用后需要重新跟踪计算寄存器和堆栈      */     } else if (opcode == BPF_JA) {       if (BPF_SRC(insns[t].code) != BPF_K) {         ret = -EINVAL;         goto err_free;       }       /* unconditional jump with single edge */       ret = push_insn(t, t + insns[t].off + 1,           FALLTHROUGH, env);       if (ret == 1)         goto peek_stack;       else if (ret < 0)         goto err_free;       /* tell verifier to check for equivalent states        * after every call and jump        */       if (t + 1 < insn_cnt)         env->explored_states[t + 1] = STATE_LIST_MARK;      /* (3.4.2.4) 剩下的是有条件跳转指令，首先探测条件失败路径，再探测条件成功路径          并且把env->explored_states[]设置成STATE_LIST_MARK，标识call函数调用后需要重新跟踪计算寄存器和堆栈      */     } else {       /* conditional jump with two edges */       env->explored_states[t] = STATE_LIST_MARK;        /* 条件失败路径 */       ret = push_insn(t, t + 1, FALLTHROUGH, env);       if (ret == 1)         goto peek_stack;       else if (ret < 0)         goto err_free;              /* 条件成功路径 */       ret = push_insn(t, t + insns[t].off + 1, BRANCH, env);       if (ret == 1)         goto peek_stack;       else if (ret < 0)         goto err_free;     }    /* (3.4.3) 非分支指令 */   } else {     /* all other non-branch instructions with single      * fall-through edge      */     ret = push_insn(t, t + 1, FALLTHROUGH, env);     /* (3.4.3.1) ret的含义如下         ret == 1：继续探索路径         ret == 0：已经是叶子节点了，跳转到mark_explored确认并回溯         ret < 0：探测到"back-edge"环路，或者其他错误      */     if (ret == 1)       goto peek_stack;     else if (ret < 0)       goto err_free;   }      /* (3.4.4) 确认并回溯，状态标记为EXPLORED       */ mark_explored:   insn_state[t] = EXPLORED;   if (cur_stack-- <= 0) {     verbose("pop stack internal bug\n");     ret = -EFAULT;     goto err_free;   }   goto peek_stack;      /* (3.4.5) 确认没有unreachable的指令，就是路径没法抵达 */ check_state:   for (i = 0; i < insn_cnt; i++) {     if (insn_state[i] != EXPLORED) {       verbose("unreachable insn %d\n", i);       ret = -EINVAL;       goto err_free;     }   }   ret = 0; /* cfg looks good */  err_free:   kfree(insn_state);   kfree(insn_stack);   return ret; }
```

  

- **3、step2、详细扫描BPF代码的运行过程，跟踪分析寄存器和堆栈，检查是否有不符合规则的情况出现。**
    
      
    

这段代码的具体算法就是把step1的路径重新走一遍，并且跟踪寄存器和堆栈的变化，判断最坏情况下是否有违反规则的情况出现。

在碰到指令对应explored_states[]被设置成STATE_LIST_MARK，需要给当前指令独立分配一个bpf_verifier_state_list链表，来存储这个指令在多个分支上的不同状况。

  

这里也有一个快速分析的优化方法：修剪(Pruning)。如果当前指令的当前分支的状态cur_state，和当前指令另一个已分析分支的状态(当前指令explored_states[]链表中的一个bpf_verifier_state_list成员)相等或者是它的一个子集，那么当前指令的当前分支就不需要分析了，因为它肯定是符合规则的。

  

```c
static int do_check(struct bpf_verifier_env *env) {   struct bpf_verifier_state *state = &env->cur_state;   struct bpf_insn *insns = env->prog->insnsi;   struct bpf_reg_state *regs = state->regs;   int insn_cnt = env->prog->len;   int insn_idx, prev_insn_idx = 0;   int insn_processed = 0;   bool do_print_state = false;    init_reg_state(regs);   insn_idx = 0;   env->varlen_map_value_access = false;   for (;;) {     struct bpf_insn *insn;     u8 class;     int err;      if (insn_idx >= insn_cnt) {       verbose("invalid insn idx %d insn_cnt %d\n",         insn_idx, insn_cnt);       return -EFAULT;     }      insn = &insns[insn_idx];     class = BPF_CLASS(insn->code);      if (++insn_processed > BPF_COMPLEXITY_LIMIT_INSNS) {       verbose("BPF program is too large. Proccessed %d insn\n",         insn_processed);       return -E2BIG;     }      err = is_state_visited(env, insn_idx);     if (err < 0)       return err;     if (err == 1) {       /* found equivalent state, can prune the search */       if (log_level) {         if (do_print_state)           verbose("\nfrom %d to %d: safe\n",             prev_insn_idx, insn_idx);         else           verbose("%d: safe\n", insn_idx);       }       goto process_bpf_exit;     }      if (need_resched())       cond_resched();      if (log_level && do_print_state) {       verbose("\nfrom %d to %d:", prev_insn_idx, insn_idx);       print_verifier_state(&env->cur_state);       do_print_state = false;     }      if (log_level) {       verbose("%d: ", insn_idx);       print_bpf_insn(env, insn);     }      err = ext_analyzer_insn_hook(env, insn_idx, prev_insn_idx);     if (err)       return err;      env->insn_aux_data[insn_idx].seen = true;     if (class == BPF_ALU || class == BPF_ALU64) {       err = check_alu_op(env, insn);       if (err)         return err;      } else if (class == BPF_LDX) {       enum bpf_reg_type *prev_src_type, src_reg_type;        /* check for reserved fields is already done */        /* check src operand */       err = check_reg_arg(regs, insn->src_reg, SRC_OP);       if (err)         return err;        err = check_reg_arg(regs, insn->dst_reg, DST_OP_NO_MARK);       if (err)         return err;        src_reg_type = regs[insn->src_reg].type;        /* check that memory (src_reg + off) is readable,        * the state of dst_reg will be updated by this func        */       err = check_mem_access(env, insn->src_reg, insn->off,                  BPF_SIZE(insn->code), BPF_READ,                  insn->dst_reg);       if (err)         return err;        reset_reg_range_values(regs, insn->dst_reg);       if (BPF_SIZE(insn->code) != BPF_W &&           BPF_SIZE(insn->code) != BPF_DW) {         insn_idx++;         continue;       }        prev_src_type = &env->insn_aux_data[insn_idx].ptr_type;        if (*prev_src_type == NOT_INIT) {         /* saw a valid insn          * dst_reg = *(u32 *)(src_reg + off)          * save type to validate intersecting paths          */         *prev_src_type = src_reg_type;        } else if (src_reg_type != *prev_src_type &&            (src_reg_type == PTR_TO_CTX ||             *prev_src_type == PTR_TO_CTX)) {         /* ABuser program is trying to use the same insn          * dst_reg = *(u32*) (src_reg + off)          * with different pointer types:          * src_reg == ctx in one branch and          * src_reg == stack|map in some other branch.          * Reject it.          */         verbose("same insn cannot be used with different pointers\n");         return -EINVAL;       }      } else if (class == BPF_STX) {       enum bpf_reg_type *prev_dst_type, dst_reg_type;        if (BPF_MODE(insn->code) == BPF_XADD) {         err = check_xadd(env, insn);         if (err)           return err;         insn_idx++;         continue;       }        /* check src1 operand */       err = check_reg_arg(regs, insn->src_reg, SRC_OP);       if (err)         return err;       /* check src2 operand */       err = check_reg_arg(regs, insn->dst_reg, SRC_OP);       if (err)         return err;        dst_reg_type = regs[insn->dst_reg].type;        /* check that memory (dst_reg + off) is writeable */       err = check_mem_access(env, insn->dst_reg, insn->off,                  BPF_SIZE(insn->code), BPF_WRITE,                  insn->src_reg);       if (err)         return err;        prev_dst_type = &env->insn_aux_data[insn_idx].ptr_type;        if (*prev_dst_type == NOT_INIT) {         *prev_dst_type = dst_reg_type;       } else if (dst_reg_type != *prev_dst_type &&            (dst_reg_type == PTR_TO_CTX ||             *prev_dst_type == PTR_TO_CTX)) {         verbose("same insn cannot be used with different pointers\n");         return -EINVAL;       }      } else if (class == BPF_ST) {       if (BPF_MODE(insn->code) != BPF_MEM ||           insn->src_reg != BPF_REG_0) {         verbose("BPF_ST uses reserved fields\n");         return -EINVAL;       }       /* check src operand */       err = check_reg_arg(regs, insn->dst_reg, SRC_OP);       if (err)         return err;        if (is_ctx_reg(env, insn->dst_reg)) {         verbose("BPF_ST stores into R%d context is not allowed\n",           insn->dst_reg);         return -EACCES;       }        /* check that memory (dst_reg + off) is writeable */       err = check_mem_access(env, insn->dst_reg, insn->off,                  BPF_SIZE(insn->code), BPF_WRITE,                  -1);       if (err)         return err;      } else if (class == BPF_JMP) {       u8 opcode = BPF_OP(insn->code);        if (opcode == BPF_CALL) {         if (BPF_SRC(insn->code) != BPF_K ||             insn->off != 0 ||             insn->src_reg != BPF_REG_0 ||             insn->dst_reg != BPF_REG_0) {           verbose("BPF_CALL uses reserved fields\n");           return -EINVAL;         }          err = check_call(env, insn->imm, insn_idx);         if (err)           return err;        } else if (opcode == BPF_JA) {         if (BPF_SRC(insn->code) != BPF_K ||             insn->imm != 0 ||             insn->src_reg != BPF_REG_0 ||             insn->dst_reg != BPF_REG_0) {           verbose("BPF_JA uses reserved fields\n");           return -EINVAL;         }          insn_idx += insn->off + 1;         continue;        } else if (opcode == BPF_EXIT) {         if (BPF_SRC(insn->code) != BPF_K ||             insn->imm != 0 ||             insn->src_reg != BPF_REG_0 ||             insn->dst_reg != BPF_REG_0) {           verbose("BPF_EXIT uses reserved fields\n");           return -EINVAL;         }          /* eBPF calling convetion is such that R0 is used          * to return the value from eBPF program.          * Make sure that it's readable at this time          * of bpf_exit, which means that program wrote          * something into it earlier          */         err = check_reg_arg(regs, BPF_REG_0, SRC_OP);         if (err)           return err;          if (is_pointer_value(env, BPF_REG_0)) {           verbose("R0 leaks addr as return value\n");           return -EACCES;         }  process_bpf_exit:         insn_idx = pop_stack(env, &prev_insn_idx);         if (insn_idx < 0) {           break;         } else {           do_print_state = true;           continue;         }       } else {         err = check_cond_jmp_op(env, insn, &insn_idx);         if (err)           return err;       }     } else if (class == BPF_LD) {       u8 mode = BPF_MODE(insn->code);        if (mode == BPF_ABS || mode == BPF_IND) {         err = check_ld_abs(env, insn);         if (err)           return err;        } else if (mode == BPF_IMM) {         err = check_ld_imm(env, insn);         if (err)           return err;          insn_idx++;         env->insn_aux_data[insn_idx].seen = true;       } else {         verbose("invalid BPF_LD mode\n");         return -EINVAL;       }       reset_reg_range_values(regs, insn->dst_reg);     } else {       verbose("unknown insn class %d\n", class);       return -EINVAL;     }      insn_idx++;   }    verbose("processed %d insns\n", insn_processed);   return 0; }
```

  

- **4、修复BPF指令中对内核helper function函数的调用，把函数编号替换成实际的函数指针。**
    
      
    

**符合条件**：(insn->code == (BPF_JMP | BPF_CALL)) 的指令，即是调用helper function的指令。

通用helper function的处理：根据insn->imm指定的编号找打对应的函数指针，然后再把函数指针和__bpf_call_base之间的offset，赋值到insn->imm中。

  

```c
static int fixup_bpf_calls(struct bpf_verifier_env *env) {   struct bpf_prog *prog = env->prog;   struct bpf_insn *insn = prog->insnsi;   const struct bpf_func_proto *fn;   const int insn_cnt = prog->len;   struct bpf_insn insn_buf[16];   struct bpf_prog *new_prog;   struct bpf_map *map_ptr;   int i, cnt, delta = 0;      /* (3.8.1) 遍历指令 */   for (i = 0; i < insn_cnt; i++, insn++) {        /* (3.8.2) 修复ALU指令的一个bug */     if (insn->code == (BPF_ALU | BPF_MOD | BPF_X) ||         insn->code == (BPF_ALU | BPF_DIV | BPF_X)) {       /* due to JIT bugs clear upper 32-bits of src register        * before div/mod operation        */       insn_buf[0] = BPF_MOV32_REG(insn->src_reg, insn->src_reg);       insn_buf[1] = *insn;       cnt = 2;       new_prog = bpf_patch_insn_data(env, i + delta, insn_buf, cnt);       if (!new_prog)         return -ENOMEM;        delta    += cnt - 1;       env->prog = prog = new_prog;       insn      = new_prog->insnsi + i + delta;       continue;     }          /* (3.8.3) 符合条件：(insn->code == (BPF_JMP | BPF_CALL))              的指令，即是调用helper function的指令          */     if (insn->code != (BPF_JMP | BPF_CALL))       continue;          /* (3.8.3.1) 几种特殊helper function的处理 */     if (insn->imm == BPF_FUNC_get_route_realm)       prog->dst_needed = 1;     if (insn->imm == BPF_FUNC_get_prandom_u32)       bpf_user_rnd_init_once();     if (insn->imm == BPF_FUNC_tail_call) {       /* mark bpf_tail_call as different opcode to avoid        * conditional branch in the interpeter for every normal        * call and to prevent accidental JITing by JIT compiler        * that doesn't support bpf_tail_call yet         */       insn->imm = 0;       insn->code |= BPF_X;        /* instead of changing every JIT dealing with tail_call        * emit two extra insns:        * if (index >= max_entries) goto out;        * index &= array->index_mask;        * to avoid out-of-bounds cpu speculation        */       map_ptr = env->insn_aux_data[i + delta].map_ptr;       if (!map_ptr->unpriv_array)         continue;       insn_buf[0] = BPF_JMP_IMM(BPF_JGE, BPF_REG_3,               map_ptr->max_entries, 2);       insn_buf[1] = BPF_ALU32_IMM(BPF_AND, BPF_REG_3,                 container_of(map_ptr,                  struct bpf_array,                  map)->index_mask);       insn_buf[2] = *insn;       cnt = 3;       new_prog = bpf_patch_insn_data(env, i + delta, insn_buf, cnt);       if (!new_prog)         return -ENOMEM;        delta    += cnt - 1;       env->prog = prog = new_prog;       insn      = new_prog->insnsi + i + delta;       continue;     }          /* (3.8.3.2) 通用helper function的处理：根据insn->imm指定的编号找打对应的函数指针 */     fn = prog->aux->ops->get_func_proto(insn->imm);     /* all functions that have prototype and verifier allowed      * programs to call them, must be real in-kernel functions      */     if (!fn->func) {       verbose("kernel subsystem misconfigured func %d\n",         insn->imm);       return -EFAULT;     }      /* (3.8.3.3) 然后再把函数指针和__bpf_call_base之间的offset，赋值到insn->imm中 */     insn->imm = fn->func - __bpf_call_base;   }    return 0; }
```

  

### 1.1.3、bpf JIT/kernel interpreter

在verifier验证通过以后，内核通过JIT(Just-In-Time)将BPF目编码转换成本地指令码；如果当前架构不支持JIT转换内核则会使用一个解析器(interpreter)来模拟运行，这种运行效率较低；

  

有些架构(64 bit x86_64, arm64, ppc64, s390x, mips64, sparc64 and 32 bit arm)已经支持BPF的JIT，它可以高效的几乎一比一的把BPF代码转换成本机代码(因为eBPF的指令集已经做了优化，非常类似最新的arm/x86架构，ABI也类似)。如果当前架构不支持JTI只能使用内核的解析器(interpreter)来模拟运行；

  

```c
struct bpf_prog *bpf_prog_select_runtime(struct bpf_prog *fp, int *err) { #ifndef CONFIG_BPF_JIT_ALWAYS_ON     /* (4.1) 在不支持JIT只能使用解析器(interpreter)时，BPF程序的运行入口 */   fp->bpf_func = (void *) __bpf_prog_run; #else   fp->bpf_func = (void *) __bpf_prog_ret0; #endif    /* eBPF JITs can rewrite the program in case constant    * blinding is active. However, in case of error during    * blinding, bpf_int_jit_compile() must always return a    * valid program, which in this case would simply not    * be JITed, but falls back to the interpreter.    */   /* (4.2) 尝试对BPF程序进行JIT转换 */   fp = bpf_int_jit_compile(fp); #ifdef CONFIG_BPF_JIT_ALWAYS_ON   if (!fp->jited) {     *err = -ENOTSUPP;     return fp;   } #endif   bpf_prog_lock_ro(fp);    /* The tail call compatibility check can only be done at    * this late stage as we need to determine, if we deal    * with JITed or non JITed program concatenations and not    * all eBPF JITs might immediately support all features.    */   /* (4.3) 对tail call使用的BPF_MAP_TYPE_PROG_ARRAY类型的map，进行一些检查 */   *err = bpf_check_tail_call(fp);    return fp; }
```

- **1、JIT**
    
      
    

以arm64的JIT转换为例：

```
struct bpf_prog *bpf_int_jit_compile(struct bpf_prog *prog)
```

  

JIT的核心转换分为3部分：**prologue + body + epilogue**。  
**prologue**：新增的指令，负责BPF运行堆栈的构建和运行现场的保护；  
**body**：BPF主体部分；  
**epilogue**：负责BPF运行完现场的恢复和清理；

  

- **1.1、prologue**
    

A64_：开头的是本机的相关寄存器

BPF_：开头的是BPF虚拟机的寄存器

  

整个过程还是比较巧妙的：

  

首先将A64_FP/A64_LR保存进堆栈A64_SP，然后把当前A64_SP保存进A64_FP；

继续保存callee saved registers进堆栈A64_SP：r6, r7, r8, r9, fp, tcc，然后把当前A64_SP保存进BPF_FP；

把A64_SP减去STACK_SIZE，给BPF_FP留出512字节的堆栈空间；

这样BPF程序使用的是BPF_FP开始的512字节堆栈空间，普通kernel函数使用的是A64_SP继续向下的堆栈空间，互不干扰；

  

```
static int build_prologue(struct jit_ctx *ctx)
```

  

- **1.2、body**
    

  

把BPF指令翻译成本地arm64指令：

  

```
static int build_body(struct jit_ctx *ctx)
```

  

- **1.3、epilogue**
    

  

做和prologue相反的工作，恢复和清理堆栈：

  

```
static void build_epilogue(struct jit_ctx *ctx)
```

  

- **2、interpreter**
    

  

对于不支持JIT的情况，内核只能使用一个解析器来解释prog->insnsi[]中BPF的指令含义，模拟BPF指令的运行:

  

使用“u64 stack[MAX_BPF_STACK / sizeof(u64)]”局部变量来模拟BPF堆栈空间；

使用“u64 regs[MAX_BPF_REG]”局部变量来模拟BPF寄存器；

  

```
/**
```

  

**3、BPF_PROG_RUN()**

  

不论是转换成JIT的映像，或者是使用interpreter解释器。最后BPF程序运行的时候都是使用BPF_PROG_RUN()这个宏来调用的：

  

```
ret = BPF_PROG_RUN(prog, ctx);
```

  

### 1.1.4、fd分配

对于加载到内核空间的BPF程序，最后会给它分配一个文件句柄fd，将prog存储到对应的file->private_data上。方便后续的引用。

  

```
int bpf_prog_new_fd(struct bpf_prog *prog)
```

  

## 1.2、bpf map操作

BPF map的应用场景有几种：

  

- BPF程序和用户态态的交互：BPF程序运行完，得到的结果存储到map中，供用户态访问；
    
- BPF程序内部交互：如果BPF程序内部需要用全局变量来交互，但是由于安全原因BPF程序不允许访问全局变量，可以使用map来充当全局变量；
    
- BPF Tail call：Tail call是一个BPF程序跳转到另一BPF程序，BPF程序首先通过BPF_MAP_TYPE_PROG_ARRAY类型的map来知道另一个BPF程序的指针，然后调用tail_call()的helper function来执行Tail call。
    
- BPF程序和内核态的交互：和BPF程序以外的内核程序交互，也可以使用map作为中介；
    

  

目前，支持的map种类：

  

```
static int __init register_array_map(void)
```

  

不论哪种map，对map的使用都是用"键-值“对(key-value)的形式来使用的。  

  

### 1.2.1、map的创建

如果用户态的BPF c程序有定义map，map最后会被编译进__section(“maps”)。  
用户态的loader在加载BPF程序的时候，首先会根据__section(“maps”)中的成员来调用bpf()系统调用来创建map对象。

  

```
static int map_create(union bpf_attr *attr)
```

  

- **1、BPF_MAP_TYPE_ARRAY**
    
      
    

我们以BPF_MAP_TYPE_ARRAY类型的map为例，来看看map的分配过程：

  

从用户态传过来的attr成员意义如下：

attr->map_type：map的类型；

attr->key_size：键key成员的大小；

attr->value_size：值value成员的大小；

attr->max_entries：需要存储多少个条目("键-值“对)

  

```
static const struct bpf_map_ops array_ops = {
```

  

- **2、BPF_MAP_TYPE_HASH**
    

  

我们以BPF_MAP_TYPE_HASH类型的map为例，来看看map的分配过程：

  

```
static const struct bpf_map_ops htab_ops = {
```

  

### 1.2.2、map的查找

  

查找就是通过key来找到对应的value。

  

```
static int map_lookup_elem(union bpf_attr *attr)
```

  

**1、BPF_MAP_TYPE_ARRAY**

  

BPF_MAP_TYPE_ARRAY类型的map最终调用到array_map_lookup_elem()：

  

```
static void *array_map_lookup_elem(struct bpf_map *map, void *key)
```

  

- 2、BPF_MAP_TYPE_HASH
    

  

BPF_MAP_TYPE_HASH类型的map最终调用到htab_map_lookup_elem()：

  

```
static void *htab_map_lookup_elem(struct bpf_map *map, void *key)
```

  

### 1.2.3、BPF_FUNC_map_lookup_elem

  

除了用户态空间需要通过bpf()系统调用来查找key对应的value值。BPF程序中也需要根据key查找到value的地址，然后在BPF程序中使用。BPF程序时通过调用BPF_FUNC_map_lookup_elem helper function来实现的。

  

我们以perf_event为例，看看BPF_FUNC_map_lookup_elem helper function的实现：

  

```
static const struct bpf_verifier_ops perf_event_prog_ops = {
```

  

和bpf()系统调用一样，最后调用的都是map->ops->map_lookup_elem()函数，只不过BPF程序需要返回的是value的指针，而bpf()系统调用需要返回的是value的值。

关于map的helper function，还有BPF_FUNC_map_update_elem、BPF_FUNC_map_delete_elem可以使用，原理一样。

  

## 1.3、obj pin

  

系统把bpf_prog和bpf_map都和文件句柄绑定起来。有一系列的好处：比如可以在用户态使用一系列的通用文件操作；也有一系列的坏处：因为fd生存在进程空间的，其他进程不能访问，而且一旦本进程退出，这些对象都会处于失联状态无法访问。

  

所以系统也支持把bpf对象进行全局化的声明，具体的做法是把这些对象绑定到一个专用的文件系统当中：

  

```
# ls /sys/fs/bpf/
```

  

具体分为pin操作和get操作。

  

### 1.3.1、bpf_obj_pin()

  

```
static int bpf_obj_pin(const union bpf_attr *attr)
```

  

### 1.3.2、bpf_obj_get()

  

```
static int bpf_obj_get(const union bpf_attr *attr)
```

  

## 2.Tracing类型的BPF程序

  

经过上一节的内容，bpf程序和map已经加载到内核当中了。什么时候bpf程序才能发挥它的作用呢？

这就需要bpf的应用系统把其挂载到适当的钩子上，当钩子所在点的路径被执行，钩子被触发，BPF程序得以执行。

  

目前应用bpf的子系统分为两大类：

  

- tracing：kprobe、tracepoint、perf_event
    
- filter：sk_filter、sched_cls、sched_act、xdp、cg_skb
    
      
    

我们仔细分析一下tracing类子系统应用bpf的过程，tracing类型的bpf操作都是通过perf来完成的。

  

## 2.1、bpf程序的绑定

在使用perf_event_open()系统调用创建perf_event并且返回一个文件句柄后，可以使用ioctl的PERF_EVENT_IOC_SET_BPF命令把加载好的bpf程序和当前perf_event绑定起来。

  

```
static long perf_ioctl(struct file *file, unsigned int cmd, unsigned long arg)
```

  

如上，perf_event绑定bpf_prog的规则如下：

  

- 对于PERF_TYPE_HARDWARE、PERF_TYPE_SOFTWARE类型的perf_event，需要绑定BPF_PROG_TYPE_PERF_EVENT类型的BPF prog。event->prog = prog;
    
- 对于TRACE_EVENT_FL_TRACEPOINT实现的PERF_TYPE_TRACEPOINT类型的perf_event，需要绑定BPF_PROG_TYPE_TRACEPOINT类型的BPF prog。event->tp_event->prog = prog;
    
- 对于TRACE_EVENT_FL_UKPROBE实现的PERF_TYPE_TRACEPOINT类型的perf_event，需要绑定BPF_PROG_TYPE_KPROBE类型的BPF prog。event->tp_event->prog = prog;
    

  

## 2.2、bpf程序的执行

  

因为几种perf_event的执行路径不一样，我们分开描述。

- 1、PERF_TYPE_HARDWARE、PERF_TYPE_SOFTWARE类型的perf_event。
    

```
static void bpf_overflow_handler(struct perf_event *event,
```

  

- **2、TRACE_EVENT_FL_TRACEPOINT实现的PERF_TYPE_TRACEPOINT类型的perf_event。**
    
-   
    

```
static notrace void              \
```

  

- **3、TRACE_EVENT_FL_UKPROBE实现的PERF_TYPE_TRACEPOINT类型的perf_event。**
    

kprobe类型的实现：

  

  

```
static void
```

  

kretprobe类型的实现：

  

```
static void
```

  

## 3.Filter类型的BPF程序

  

暂不分析

阅读 4008

​

写留言

**留言 6**

- 游~游~游
    
    2022年2月22日
    
    赞1
    
    很丰满的ebpf文章，赞~
    
- 乘风破浪的凌杰
    
    2022年3月5日
    
    赞
    
    终于知道这bpf机器码的jit过程了，有概念后可以帮助我化解心里的负担，多谢作者的创作和无私分享
    
- 西米宜家
    
    2022年2月23日
    
    赞
    
    期待后续，对于各种filter使用场景的介绍。![[强]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![[强]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![[强]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
- 西米宜家
    
    2022年2月23日
    
    赞
    
    伟林老师您好，感谢分享! 我在里面看到了专门针对skb-&gt;data+offset访问的优化，这个比较意外。请问，如果修改了skb的结构定义，data的偏移也会发生变化吧，那么offsetof(skb, data)的信息保存在什么地方呢？
    
- 陈乐安生
    
    2022年2月22日
    
    赞
    
    这个帖子说到点上了，epbf还是用到内核的hook机制实现trace，只不过ebpf在hook安全性和稳定性上做了重点防护。
    
- CPLee
    
    2022年2月22日
    
    赞
    
    赞，干货满满
    

已无更多数据

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/YHBSoNHqDiaH5V6ycE2TlM9McXhAbaV9aUBaTAibVhLibXH1rL4JTMb2eoQfdCicTm4wycOsopO82jG3szvYF3mTJw/300?wx_fmt=png&wxfrom=18)

Linux阅码场

25520

6

写留言

**留言 6**

- 游~游~游
    
    2022年2月22日
    
    赞1
    
    很丰满的ebpf文章，赞~
    
- 乘风破浪的凌杰
    
    2022年3月5日
    
    赞
    
    终于知道这bpf机器码的jit过程了，有概念后可以帮助我化解心里的负担，多谢作者的创作和无私分享
    
- 西米宜家
    
    2022年2月23日
    
    赞
    
    期待后续，对于各种filter使用场景的介绍。![[强]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![[强]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![[强]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
- 西米宜家
    
    2022年2月23日
    
    赞
    
    伟林老师您好，感谢分享! 我在里面看到了专门针对skb-&gt;data+offset访问的优化，这个比较意外。请问，如果修改了skb的结构定义，data的偏移也会发生变化吧，那么offsetof(skb, data)的信息保存在什么地方呢？
    
- 陈乐安生
    
    2022年2月22日
    
    赞
    
    这个帖子说到点上了，epbf还是用到内核的hook机制实现trace，只不过ebpf在hook安全性和稳定性上做了重点防护。
    
- CPLee
    
    2022年2月22日
    
    赞
    
    赞，干货满满
    

已无更多数据