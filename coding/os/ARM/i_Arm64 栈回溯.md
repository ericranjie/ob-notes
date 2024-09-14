# 

Original 业余程序员plus 人人极客社区

 _2024年09月06日 08:41_ _江苏_

![](http://mmbiz.qpic.cn/mmbiz_png/9sNwsXcN68pL55XIyzTrCHZTbIUdTibQcuzuCaYeGTXNMyn6ACmicUrpoDC0xZSap46XJ59sKysPg9Rg379f32cA/300?wx_fmt=png&wxfrom=19)

**人人极客社区**

工程师们自己的Linux底层技术社区，分享体系架构、内核、网络、安全和驱动。

317篇原创内容

公众号

## AArch64栈的结构

Arm64有4种栈，分别是空增栈(Empty Ascendant Stack,EA)、空减栈(Empty Descendant Stack,ED)、满增栈(Full Ascendant Stack,FA)、满减栈(Full Descendant Stack,FD)。常用的是满减栈，Linux内核也使用满减栈。

下图是一个满减栈的示意图，高地址为栈顶，低地址为栈低，栈向低地址方向生长，如右边的箭头所示。栈指针SP指向栈底（栈低保存了数据）。每产生一次函数调用，就会在栈中形成一个栈帧，该栈总共保存了4个栈帧（Stack Frame），每个栈帧由FP、LR及栈参数（函数参数、函数局部变量等）组成。可以将栈中的所有栈帧视为一个单项链表，栈最低位置的栈帧为链表头，栈最高位置的栈帧为链表尾，整个链表使用FP索引。栈手动回溯时，可以根据FP将所有栈帧索引出来。

![Image](https://mmbiz.qpic.cn/sz_mmbiz_png/9sNwsXcN68qwDItRJicnD5HyNsOu8L3DmPEL5uGrHibIPwwzCUM8iaFNuvpWL2k7tLgUvUZ4BxtSibkUyIr50FD8aA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

## AArch64过程调用标准中寄存器的使用规则

下面是Arm64程序调用标准规定的通用寄存器的使用方法。

- 参数寄存器（X0-X7） 函数参数数量小于等于8个时，使用X0-X7传递，大于8个时，多余的使用栈传递，函数返回时返回值保存在X0中。
    
- 调用者保存的临时寄存器（X9-X15） 调用者若使用到了X9-X15寄存器，在调用子函数之前，需要将X9-X15寄存器保存到自己的栈中，子函数使用这些寄存器的时候不需要保存和恢复。
    
- 被调用者保存的寄存器（X19-X29） 被调用者若使用到这些寄存器，需要将其保存到自己的栈中，返回时从栈中恢复。
    
- 特殊用途的寄存器
    

- X8是间接结果寄存器。用于传递间接结果的地址位置，例如，函数返回一个大结构。
    
- X16-X17过程内调用暂存寄存器。。
    
- X18平台寄存器。
    
- X29是栈帧（FP）寄存器。保存了调用函数的栈帧地址。
    
- X30保存了返回地址（LR）。函数返回后跳转到该地址处运行。
    
![[Pasted image 20240911001423.png]]
![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

## 实例

下图是内核Oops时打印出的信息。第一张图片是寄存器信息，pc寄存器和sp寄存器对栈回溯有重要作用。第二张图是内核线程irq/231-dwc3栈数据的二进制转储，栈回溯就是在这些二进制数据中找到栈帧，从而找到调用的函数地址。
![[Pasted image 20240911001434.png]]
![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)
![[Pasted image 20240911001439.png]]
![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

下图是内核栈回溯的结果，发生异常函数的地址保存在异常栈中，不在内核线程irq/231-dwc3栈中。
![[Pasted image 20240911001445.png]]
![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

发生异常的函数可以根据pc寄存器得到，该函数是栈回溯的第一个函数。sp寄存器指向了第一个栈帧中的FP1寄存器，即0xffffffc0ee823b80地址，FP1向高地址偏移8字节得到LR1寄存器，即0xffffff80087369e4地址，该地址位于dwc3_ep0_stall_and_restart函数内，该函数是栈回溯的第二个函数。FP1指向了第二个栈帧的FP2，根据栈帧找到LR2，依次类推。所有的栈帧最终如下图所示，总共找到7个栈帧，因此irq/231-dwc3内核线程发生异常时总共有8个函数调用，和内核输出的函数调用关系一致。需要注意的是，代码里调用了该函数，但在栈回溯中没有找到符号，肯定是编译器优化，将该函数内联了，是否内联可以通过反汇编确认。
![[Pasted image 20240911001451.png]]
![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

ARM7

ARM · 目录

上一篇Arm64 栈回溯

Reads 1802

​

Comment

**留言 1**

- 蒋辰浩
    
    浙江Yesterday
    
    Like1
    
    这四种是栈的四种模型 和arm64有什么关系？
    

已无更多数据

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/9sNwsXcN68pL55XIyzTrCHZTbIUdTibQcuzuCaYeGTXNMyn6ACmicUrpoDC0xZSap46XJ59sKysPg9Rg379f32cA/300?wx_fmt=png&wxfrom=18)

人人极客社区

1513415

1

Comment

**留言 1**

- 蒋辰浩
    
    浙江Yesterday
    
    Like1
    
    这四种是栈的四种模型 和arm64有什么关系？
    

已无更多数据