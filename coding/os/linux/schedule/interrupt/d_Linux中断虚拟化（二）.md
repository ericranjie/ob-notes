# 

Original 王柏生 谢广军 Linux阅码场

_2021年12月01日 07:00_

作者简介

**王柏生**  资深技术专家，先后就职于中科院软件所、红旗Linux和百度，现任百度主任架构师。在操作系统、虚拟化技术、分布式系统、云计算、自动驾驶等相关领域耕耘多年，有着丰富的实践经验。著有畅销书《深度探索Linux操作系统》（2013年出版）。

**谢广军**  计算机专业博士，毕业于南开大学计算机系。资深技术专家，多年的IT行业工作经验。现担任百度智能云副总经理，负责云计算相关产品的研发。多年来一直从事操作系统、虚拟化技术、分布式系统、大数据、云计算等相关领域的研发工作，实践经验丰富。

本文内容节选自\*\*《深度探索Linux虚拟化技术》\*\*，已获得机械工业出版社华章公司授权。

**欢迎读者文末留言，阅码场和机械工业出版社华章公司将为每位**精彩留言**获奖用户奉送该书一本。**

**PIC虚拟化**

计算机系统有很多的外设需要服务，显然，CPU采用轮询的方式逐个询问外设是否需要服务，是非常浪费CPU的计算的，尤其是对那些并不是频繁需要服务的设备。因此，计算机科学家们设计了外设主动向CPU发起服务请求的方式，这种方式就是中断。采用中断方式后，在没有外设请求时，CPU就可以继续其他计算任务，而不是进行很多不必要的轮询，极大地提高了系统的吞吐\[1\] 在每个指令周期结束后，如果CPU关中断标识（IF）没有被设置，那么其会去检查是否有中断请求，如果有中断请求，则运行对应的中断服务程序，然后返回被中断的计算任务继续执行。

CPU不可能为每个硬件都设计专门的管脚接收中断，管脚数量的限制、电路的复杂度、灵活度等方方面面都不现实，因此，需要设计一个专门管理中断的单元。由中断管理单元接受来自外围设备的请求，确定请求的优先级，并向CPU发出中断。1981年IBM推出的第1代个人电脑PC/XT使用了一个独立的8259A作为中断控制器，自此，8259A就成为了单核时代中断芯片事实上的标准。因为可以通过软件编程对其进行控制，比如当管脚收到设备信号时，可以编程控制其发出的中断向量号，因此，中断控制器又称为可编程中断控制器（programmable interrupt controller），简称PIC。单片8259A可以连接8个外设的中断信号线，可以多片级联支持更多外设。

8259A和CPU的连接如图5所示。

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

_图5 8259A和CPU连接_

片选和地址译码器相连，当CPU准备访问8259A前，需要向地址总线发送8259A对应的地址，经过译码器后，译码器发现是8259A对应的地址，因此会拉低与8259A的CS连接的管脚的电平，从而选中8259A，通知8259A，CPU准备与其交换数据了。

8259A的D0~7管脚与CPU的数据总线相连。从CPU向8259A发送ICW和OCW，从8259A向CPU传送8259A的状态以及中断向量号，都是通过数据总线传递的。

当CPU向8259A发送ICW、OCW时，当把数据送上数据总线后，需要通知8259A读数据，CPU通过拉低WR管脚的电平的方式通知8259A，当8259A的WR管脚收到低电平后，读取数据总线的数据。类似的，CPU准备好读取8259A的状态时，拉低RD管脚通知8259A。

8259A和CPU之间的中断信号的通知使用专用的连线，8259A的管脚INTR（interrupt request）和INTA(interrupt acknowledge)分别与处理器的INTR和INTA管脚相连。8259A通过管脚INTR向CPU发送中断请求，CPU通过管脚INTA向PIC发送中断确认，告诉PIC其收到中断并且开始处理了。8259A与CPU之间的具体中断过程如下：

1）8259A的IR0~7管脚高电平有效，所以当中断源请求服务时，拉高连接IR0~7的管脚，产生中断请求。

2）8259A需要将这些信号记录下来，因此其内部有个寄存器IRR（Interrupt Request Register），负责记录这个中断请求，针对这个例子，IRR的bit 0将被设置为1。

3）有的时候，我们会屏蔽掉某个设备的中断。换句话说，就是的当这个中断源向8259A发送信号后，8259A并不将这个中断信号发送给CPU。读者不要将屏蔽和CPU通过cli命令关中断混淆，CPU关中断时，中断还会发送给CPU，只是在关中断期间CPU不处理中断。8259A中的寄存器IMR(Interrupt Mask Register)负责记录某个中断源是否被屏蔽，比如0号中断源被设备了屏蔽，那么IMR的bit 0将被设置。那么这个IMR是谁设置的呢？当然是CPU中的操作系统。因此这一步，8259A将会检查收到的中断请求是否被屏蔽。

4）在某一个时刻，可能有多个中断请求，或者是之前存在IRR中的中断并没有被处理，8259A中积累了一些中断。某一个时刻，8259A只能向CPU发送一个中断请求，因此，当存在多个中断请求时，8259A需要判断一下中断优先级，这个单元叫做priority resolver，priority resolver将在IRR中选出优先级最高的中断。

5）选出最高优先级的中断后，8259A拉高管脚INTR的电平，向CPU发出信号。

6）当CPU执行完当前指令周期后，其将检查寄存器FLAGS的中断使能位IF（Interrupt enable flag），如果允许中断，那么将检查INTR是否有中断，如果有中断，那么将通过管脚INTR通知8259A处理器将开始处理中断。

7）8259A收到CPU发来的INTA信号后，将置位最高优先级的中断在ISR(In-Service Register)中对应的位，并清空IRR中对应的位。

8）通常，x86 CPU会发送第2次INTA，在收到第2次INTA后，8259A会将中断向量号（vector）送上数据总线D0~D7。

9）如果8259A设置为AEOI(Automatic End Of Interrupt)模式，那么8259A复位ISR中对应的bit，否则ISR中对应的bit一直保持到收到系统的中断服务程序发来的EOI命令。

我们知道，中断服务程序保存在一个数组中，数组中的每一项对应一个中断服务程序。在实模式下，这个数组称为IVT(interrupt vector table)；在保护模式下，这个数组称为IDT（Interrupt descriptor table）。

这个数组中保存的服务程序，并不是全部都是外部中断，还有处理CPU内部异常的以及软中断服务程序。x86CPU前32个中断号（0-31）留给处理器异常的，比如第0个中断号，是处理器出现除0（Divide by Zero）异常的，不能被占用。因此，假设我们计划IVT数组中第32个元素存放管脚IR0对应的ISR，那么我们初始化8259A时，通过ICW，设置起始的irq base为32，那么当8259A发出管脚IR0的中断请求时，则发出的值是32，管脚IR1对应的是33，依此类推。这个32、33就是所谓的中断向量（vector）。换句话说，中断向量就是中断服务程序在IVT/IDT中的索引。下面就是设置irq_base的代码，在初始化时，通过第2个初始化命令字（ICW2）设置：

﻿commit 85f455f7ddbed403b34b4d54b1eaf0e14126a126

KVM: Add support for in-kernel PIC emulation

﻿linux.git/drivers/kvm/i8259.c

﻿static void pic_ioport_write(void \*opaque, u32addr, u32 val)

{

…

﻿    switch(s->init_state) {

…

case 1:

s->irq_base = val & 0xf8;

…

}

}

后来，随着APIC和MSI的出现，中断向量设置的更为灵活，可以为每个PCI设置其中断向量，这个中断向量存储在PCI设备的配置空间中。

内核中抽象了一个结构体kvm_kpic_state来记录每个8259A的状态：

﻿commit 85f455f7ddbed403b34b4d54b1eaf0e14126a126

KVM: Add support for in-kernel PIC emulation

﻿struct kvm_kpic_state {

u8last_irr;  /\* edge detection \*/

u8 irr;  /\* interrupt request register \*/

u8imr;   /\* interrupt mask register \*/

u8isr;   /\* interrupt service register \*/

…

};

﻿struct kvm_pic {

structkvm_kpic_state pics\[2\]; /\* 0 is master pic, 1 is slave pic\*/

irq_request_func \*irq_request;

void\*irq_request_opaque;

intoutput;   /\* intr from master PIC \*/

structkvm_io_device dev;

};

1片8259A只能连接最多8个外设，如果需要支持更多外设，需要多片8259A级联。在结构体kvm_pic中，我们看到有2片8259A：pic\[0\]和pic\[1\]。KVM定义了结构体kvm_kpic_state记录8259A的状态，其中包括我们之前提到的IRR、IMR、ISR等等。

### **1 虚拟设备向PIC发送中断请求**

如同物理外设请求中断时拉高与8259A连接的管脚的电压，虚拟设备请求中断的方式是通过一个API告诉虚拟的8259A芯片中断请求，以kvmtool中的virtio blk虚拟设备为例：

﻿commit 4155ba8cda055b7831489e4c4a412b073493115b

kvm: Fix virtio block device support some more

﻿kvmtool.git/blk-virtio.c

﻿static bool blk_virtio_out(…)

{

…

﻿  caseVIRTIO_PCI_QUEUE_NOTIFY: {

…

while(queue->vring.avail->idx != queue->last_avail_idx) {

if(!blk_virtio_read(self, queue))

return false;

}

kvm\_\_irq_line(self, VIRTIO_BLK_IRQ, 1);

break;

}

…

}

当Guest内核的块设备驱动发出I/O通知VIRTIO_PCI_QUEUE_NOTIFY时，将触发CPU从Guest模式切换到Host模式，KVM中的块模拟设备开始I/O操作，比如访问保存Guest文件系统的镜像文件。virtio blk这个提交，块设备的I/O处理是同步的，也就是说，一直要等到文件操作完成，才会向Guest发送中断，返回Guest。当然同步阻塞在这里是不合理的，而是应该马上返回Guest，这样Guest可以执行其他的任务，虚拟设备完成I/O操作后，再通知Guest，这是kvmtool初期的实现，后来已经改进为异步的方式。代码中在一个while循环处理完设备驱动的I/O请求后，调用了函数kvm\_\_irq_line，irq_line对应8259A的管脚IR0~7，其代码如下：
```cpp
﻿commit 4155ba8cda055b7831489e4c4a412b073493115b
kvm: Fix virtio block device support some more
kvmtool.git/kvm.c

void kvm\_\_irq_line(struct kvm \*self, int irq, intlevel) {
structkvm_irq_level irq_level;
irq_level= (struct kvm_irq_level) {
{
.irq    = irq,
},
.level    = level,
};

if(ioctl(self->vm_fd, KVM_IRQ_LINE, &irq_level) \< 0)
die_perror("KVM_IRQ_LINE failed");
}
```
函数kvm\_\_irq_line将irq number和管脚电平信息，这里是1，表示拉高电平了，封装到结构体kvm_irq_level中，传递给内核中的KVM模块：
```cpp
﻿﻿commit 85f455f7ddbed403b34b4d54b1eaf0e14126a126
KVM: Add support for in-kernel PIC emulation
linux.git/drivers/kvm/kvm_main.c
﻿static long kvm_vm_ioctl(…) {
…
﻿  caseKVM_IRQ_LINE: {
…
kvm_pic_set_irq(pic_irqchip(kvm),
irq_event.irq,
irq_event.level);
…
break;
}
…
}
```
KVM模块将kvmtool中组织的中断信息从用户空间复制到内核空间中，然后调用虚拟8259A的模块中提供的API kvm_pic_set_irq，向8259A发出中断请求。

### **2 记录中断到IRR**

中断处理需要一个过程，从外设发出请求，一直到ISR处理完成发出EOI。而且可能中断来了并不能马上处理，或者之前已经累加了一些中断，大家需要排队依次请求CPU处理，等等，因此，需要一些寄存器来记录这些状态。当外设中断请求到来时，8259A首先需要将他们记录下来，这个寄存器就是IRR(Interrupt Request Register)，8259A用他来记录有哪些pending的中断需要处理。

当KVM模块收到外设的请求，调用虚拟8259A的API kvm_pic_set_irq是，其第1件事就是将中断记录到IRR寄存器中：
```cpp
﻿commit 85f455f7ddbed403b34b4d54b1eaf0e14126a126
KVM: Add support for in-kernel PIC emulation
﻿linux.git/drivers/kvm/i8259.c
﻿void kvm_pic_set_irq(void \*opaque, int irq, intlevel) {
structkvm_pic \*s = opaque;
pic_set_irq1(&s->pics\[irq >> 3\], irq & 7, level);
……
}

﻿static inline void pic_set_irq1(structkvm_kpic_state \*s,
int irq, int level) {

int mask;
mask = 1\<\< irq;
if(s->elcr & mask) /\* level triggered \*/
…
else  /\* edge triggered \*/
if(level) {
if((s->last_irr & mask) == 0)
s->irr |= mask;
s->last_irr |= mask;
} else
s->last_irr &= ~mask;
}
```
信号有边缘触发和水平触发，在物理上可以理解为，8329A在前一个周期检测到管脚信号是0，当前周前检测到管脚信号是1，如果是上升沿触发模式，那么8259A就认为外设有请求了，这种触发模式就是边缘触发。对于水平触发，以高电平触发为例，当8259A检测到管脚处于高电平，则认为外设来请求了。

在虚拟8259A的结构体kvm_kpic_state中，寄存器elcr就是用来记录8259A被设置的触发模式的。参数level即相当于硬件层面的电信号，0表示低电平，1表示高电平。以边缘触发为例，当管脚收到一个低电平时，即level的值为0，代码进入else分支，结构体kvm_kpic_state中的字段last_irr中会清除该IRQ对应IRR的位，即相当于设置该中断管脚为低电平状态。当管脚收到高电平时，即level的值为1，代码进入if分支，此时8259A将判断之前该管脚的状态，也就是判断结构体kvm_kpic_state中的字段last_irr中该IRQ对应IRR的位，如果为低电平，那么则认为中断源有中断请求，将其记录到IRR中。当然，同时需要在字段last_irr记录下当前该管脚的状态。

### **3 设置中断标识**

当8259A将中断请求记录到IRR中后，下一步就是开启一个中断评估（evaluate）过程了，包括中断是否被屏蔽，多个中断请求的优先级等等，最后将通过管脚INTA通知CPU处理外部中断。我们看到这里是8259A主动发起中断过程，但是虚拟中断有些不同，中断的发起的时机不再是虚拟中断芯片主动发起，而是在每次准备切入Guest时，KVM查询中断芯片，如果有pending的中断，则执行中断注入。模拟8259A在收到中断请求后，在记录到IRR后，设置一个变量，后面在切入Guest前KVM会查询这个变量：

﻿commit 85f455f7ddbed403b34b4d54b1eaf0e14126a126

KVM: Add support for in-kernel PIC emulation

﻿linux.git/drivers/kvm/i8259.c

﻿void kvm_pic_set_irq(void \*opaque, int irq, intlevel)

{

structkvm_pic \*s = opaque;

pic_set_irq1(&s->pics\[irq >> 3\], irq & 7, level);

﻿ pic_update_irq(s);

}

﻿static void pic_update_irq(struct kvm_pic \*s)

{

…

irq =pic_get_irq(&s->pics\[0\]);

if (irq>= 0)

s->irq_request(s->irq_request_opaque, 1);

else

s->irq_request(s->irq_request_opaque, 0);

}

﻿static void pic_irq_request(void \*opaque, intlevel)

{

struct kvm\*kvm = opaque;

pic_irqchip(kvm)->output = level;

}

在函数vmx_vcpu_run中，在准备切入Guest之前将调用函数vmx_intr_assist去检查虚拟中断芯片是否有等待处理的中断，相关代码如下：

﻿﻿commit 85f455f7ddbed403b34b4d54b1eaf0e14126a126

KVM: Add support for in-kernel PIC emulation

linux.git/drivers/kvm/vmx.c

﻿static int vmx_vcpu_run(…)

{

…

vmx_intr_assist(vcpu);

…

}

﻿static void vmx_intr_assist(struct kvm_vcpu\*vcpu)

{

…

has_ext_irq= kvm_cpu_has_interrupt(vcpu);

…

if(!has_ext_irq)

return;

interrupt_window_open =

((vmcs_readl(GUEST_RFLAGS) & X86_EFLAGS_IF) &&

(vmcs_read32(GUEST_INTERRUPTIBILITY_INFO) & 3) == 0);

﻿  if(interrupt_window_open)

vmx_inject_irq(vcpu, kvm_cpu_get_interrupt(vcpu));

…

}

其中函数kvm_cpu_has_interrupt查询8259A设置的变量output:

﻿﻿commit 85f455f7ddbed403b34b4d54b1eaf0e14126a126

KVM: Add support for in-kernel PIC emulation

﻿linux.git/drivers/kvm/irq.c

﻿int kvm_cpu_has_interrupt(struct kvm_vcpu \*v)

{

structkvm_pic \*s = pic_irqchip(v->kvm);

if(s->output)  /\* PIC \*/

return1;

return 0;

}

如果有output设置了，就说明有外部中断等待处理，然后接着判断Guest是否可以被中断，包括Guest是否中断了，Guest是否正在执行一些不能被中断的指令，如果可以注入，则调用vmx_inject_irq完成中断的注入。其中，传递给函数vmx_inject_irq的第2个参数是函数kvm_cpu_get_interrupt返回的结果，该函数获取需要注入的中断，这个过程就是中断评估过程，我们下一节讨论。

### **4 中断评估**

在上一节我们看到在执行注入前，vmx_inject_irq调用函数kvm_cpu_get_interrupt获取需要注入的中断。函数kvm_cpu_get_interrupt的核心逻辑就是中断评估（evaluate），包括：这个pending的中断有没有被屏蔽？pending的中断的优先级是否比CPU正在处理的中断优先级高？代码如下：

﻿﻿﻿commit 85f455f7ddbed403b34b4d54b1eaf0e14126a126

KVM: Add support for in-kernel PIC emulation

linux.git/drivers/kvm/irq.c

﻿int kvm_cpu_get_interrupt(struct kvm_vcpu \*v)

{

……

vector =kvm_pic_read_irq(s);

if (vector!= -1)

returnvector;

…

}

﻿linux.git/drivers/kvm/i8259.c

﻿int kvm_pic_read_irq(struct kvm_pic \*s)

{

int irq,irq2, intno;

irq =pic_get_irq(&s->pics\[0\]);

if (irq>= 0) {

…

intno = s->pics\[0\].irq_base + irq;

} else {

…

returnintno;

}

kvm_pic_read_irq调用函数pic_get_irq获取评估后的中断，如上面黑体标识的，我们可以清楚的看到中断向量和中断管脚的关系，叠加了一个irq_base，这个irq_base就是通过初始化8259A时通过ICW设置的，完成从IRn到中断向量的转换。

一个中断芯片通常连接有多个外设，所以在某一个时刻，可能会有多个设备请求到来，这时就有一个优先处理哪个请求的问题，因此，中断就有了优先级的概念。以8259A为例，典型的有2种中断优先级模式：

1）固定优先级（Fixedpriority），即优先级是固定的，从IR0到IR7依次降低，IR0的优先级永远最高，IR7的优先级永远最低。

2）循环优先级（rotatingpriority），即当前处理完的IRn其优先级调整为最低，当前处理的优先级下个，即IRn+1，调整为优先级最高。比如，当前处理的中断是irq 2,那么紧接着irq3的优先级设置为是最高，然后依次是irq4、irq5、irq6、irq7、irq1、irq2、irq3。假设此时irq5和irq2同时来了中断，那么irq5显然会被优先处理。然后irq6被设置为优先级最高，接下来依次是irq7、irq1、irq2、irq3、irq4、irq5。

理解了循环优先级算法后，从8259A中获取最高优先级请求的代码就很容易理解了：

﻿﻿﻿﻿commit85f455f7ddbed403b34b4d54b1eaf0e14126a126

KVM: Add support for in-kernel PIC emulation

linux.git/drivers/kvm/i8259.c

﻿static int pic_get_irq(struct kvm_kpic_state \*s)

{

int mask,cur_priority, priority;

mask =s->irr & ~s->imr;

priority =get_priority(s, mask);

if(priority == 8)

return-1;

…

mask =s->isr;

…

cur_priority = get_priority(s, mask);

if(priority \< cur_priority)

/\*

\*higher priority found: an irq should be generated

\*/

return(priority + s->priority_add) & 7;

else

return-1;

}

﻿static inline int get_priority(structkvm_kpic_state \*s, int mask)

{

intpriority;

if (mask== 0)

return8;

priority =0;

while((mask & (1 \<\< ((priority + s->priority_add) & 7))) == 0)

priority++;

returnpriority;

}

函数pic_get_irq分成2部分，第1部分是从当前pending的中断中取得最高优先级的中断，当前之前需要滤掉被被屏蔽的中断。第2部分是获取正在被CPU处理的中断的优先级的中断的优先级，通过这里，读者更能具体的理解了8259A为什么需要这些寄存器记录中断的状态。然后比较2个中断的优先级，如果pending的优先级高，那么就通过拉低INTR管脚电压，向CPU发出中断请求。

再来看一下计算优先级的函数get_priority。其中变量priority_add记录的是当前最高优先级的管脚，所以逻辑上就是从当前最高的优先级管脚开始，从高向低依次检查是否有pending的中断。比如当前IR4最高，那么priority_add的值就是4。while循环，从管脚IR(4+0)开始，依次检查管脚IR(4+1)、IR(4+2)，依此类推。

### **5 中断ACK**

在物理上，CPU在准备处理一个中断请求后，将通过管脚INTA向8259A发出确认脉冲。同样，软件模拟上，也需要类似处理。在完成中断评估后，准备注入Guest前，需要向虚拟8259A执行确认状态的操作，代码如下：

﻿﻿﻿commit 85f455f7ddbed403b34b4d54b1eaf0e14126a126

KVM: Add support for in-kernel PIC emulation

linux.git/drivers/kvm/i8259.c

﻿int kvm_pic_read_irq(struct kvm_pic \*s)

{

int irq,irq2, intno;

irq =pic_get_irq(&s->pics\[0\]);

if (irq>= 0) {

pic_intack(&s->pics\[0\], irq);

…

}

static inline void pic_intack(structkvm_kpic_state \*s, int irq)

{

﻿  if(s->auto_eoi) {

…

} else

s->isr |= (1 \<\< irq);

/\*

\* Wedon't clear a level sensitive interrupt here

\*/

if(!(s->elcr & (1 \<\< irq)))

s->irr &= ~(1 \<\< irq);

}

在中断评估中，在调用函数kvm_pic_read_irq获取评估的中断结果后，马上就调用函数pic_intack完成了中断确认的动作。在收到CPU发来的中断确认后，8259A需要更新自己的状态，包括，因为中断已经开始得到服务了，所以从IRR中清除等待服务请求。另外，需要设置ISR位，记录正在被服务的中断，但是这里稍微有一点点复杂。

设置ISR表示正在服务的位，表示处理器正在处理中断。设置ISR的一个典型作用是中断的作用是当ISR处理完中断，向8259A发送EOI时，8259A知道正在处理的IRn，比如说如果8259A使用的是循环优先级，那么需要最高优先级为当前处理的IRn的下一个。

如果8259A是AEOI模式，那么就无须设置ISR了，因为中断服务程序执行完毕后不会发送EOI命令。所以在AEOI模式下（上面代码的if分支），需要将收到EOI时8259A需要处理的逻辑完成，这部分内容我们在下一节会讨论。

对于边缘触发的，进入到ISR阶段后，需要复位对于IRR。对于level trigger的，在收到中断请求后8259A处理，不过多讨论了，如果读者有兴趣，可以阅读函数pic_set_irq1中关于水平触发部分的逻辑。

### **6 关于EOI的处理**

中断服务程序执行完成后，会向8259A发送EOI，告知8259A中断处理完成。8259A在收到这个EOI时，复位ISR，如果是采用的循环优先级，还需要设置变量priority_add，使其指向当前处理IRn的下一个：

﻿﻿﻿﻿commit85f455f7ddbed403b34b4d54b1eaf0e14126a126

KVM: Add support for in-kernel PIC emulation

linux.git/drivers/kvm/i8259.c

static void pic_ioport_write(void \*opaque, u32addr, u32 val)

{

…

﻿      case1: /\* end of interrupt \*/

case5:

priority = get_priority(s, s->isr);

if(priority != 8) {

irq = (priority + s->priority_add) & 7;

s->isr &= ~(1 \<\< irq);

if(cmd == 5)

s->priority_add = (irq + 1) & 7;

pic_update_irq(s->pics_state);

}

break;

…

}

如果8259A被设置为AEOI模式，不会再收到后续中断服务程序的EOI命令，那么8259A在收到CPU的ACK后，就必须把收到EOI命令时执行的逻辑现在处理，前面看到AEOI模式不必设置ISR,所以这里也无需复位ISR，只需要调整变量priority_add，记录最高优先级位置即可：

﻿﻿﻿commit 85f455f7ddbed403b34b4d54b1eaf0e14126a126

KVM: Add support for in-kernel PIC emulation

static inline void pic_intack(structkvm_kpic_state \*s, int irq)

{

if(s->auto_eoi) {

if(s->rotate_on_auto_eoi)

s->priority_add = (irq + 1) & 7;

} else

…

}

### **7 中断注入**

对于外部中断，每个CPU在指令周期结束后，将会去检查INTR是否有中断请求。那么对于处于Guest模式的CPU，其如何知道有中断请求呢？Intel在VMCS中设置了一个字段：VM-entry interruption-information，在VM entry时CPU将会检查这个字段，这个字段格式表3-1所示。

表3-1 VM-entry interruption-information格式（部分）

|   |   |
|---|---|
|位|内容|
|7:0|中断或异常向量|
|10:8|中断类型:<br><br>0: External  interrupt<br><br>1: Reserved<br><br>2: Non-maskable  interrupt (NMI)<br><br>3: Hardware  exception<br><br>4: Software  interrupt<br><br>5: Privileged  software exception<br><br>6: Software  exception<br><br>7: Other event|
|31|是否有效\[2\]|

在VM entry前，KVM模块检查虚拟8259A中如果有pending中断需要处理，则将需要处理的中断信息写入到VMCS中的这个字段VM-entry interruption-information:

﻿commit 85f455f7ddbed403b34b4d54b1eaf0e14126a126

KVM: Add support for in-kernel PIC emulation

﻿linux.git/drivers/kvm/vmx.c

﻿static void vmx_inject_irq(struct kvm_vcpu \*vcpu,int irq)

{

…

vmcs_write32(VM_ENTRY_INTR_INFO_FIELD,

irq |INTR_TYPE_EXT_INTR | INTR_INFO_VALID_MASK);

}

前面我们看到，中断注入是在每次VM entry时，KVM模块检查8259A是否有pending的中断等待处理。这样就有可能给中断带来一定的延迟，典型如下面2类情况：

（1）CPU可能正处在Guest模式，那么就需要等待下一次VM exit 和VM entry。

（2）VCPU这个线程也许正在睡眠，比如Guest VCPU运行hlt指令时，就会切换回Host模式，线程挂起。

对于第1种情况，是多处理器系统下的一个典型情况，目标CPU的正在运行Guest。KVM需要想办法触发Guest发生一次VM exit，切换到Host。我们知道，当处于Guest模式的CPU收到外部中断时，会触发VM exit，由Host来处理这次中断。所以，KVM可以向目标CPU发送一个IPI中断，触发目标CPU发生一次VM exit。

对于第2种情况，首先需要唤醒睡眠的VCPU线程，使其进入CPU就绪队列，准备接受调度。对于多处理器系统，然后再向目标CPU发送一个“重新调度”的IPI中断，那么被唤醒的VCPU线程很快就会被调度，执行切入Guest的过程，从而完成中断注入。

所以当有中断请求时，虚拟中断芯片将主动“kick”一下目标CPU，这个“踢”的函数就是kvm_vcpu_kick：

﻿commit b6958ce44a11a9e9425d2b67a653b1ca2a27796f

KVM: Emulate hlt in the kernel

﻿linux.git/drivers/kvm/i8259.c

static void pic_irq_request(void \*opaque, intlevel)

{

…

pic_irqchip(kvm)->output = level;

if (vcpu)

kvm_vcpu_kick(vcpu);

}

如果虚拟CPU线程在睡眠，则“踢醒”他。如果目标CPU运行在Guest模式，则将其从Guest模式“踢”到Host模式，在VM entry时完成中断注入，kick的手段就是我们刚刚提到的IPI，代码如下：

﻿﻿commit b6958ce44a11a9e9425d2b67a653b1ca2a27796f

KVM: Emulate hlt in the kernel

linux.git/drivers/kvm/irq.c

﻿void kvm_vcpu_kick(struct kvm_vcpu \*vcpu)

{

intipi_pcpu = vcpu->cpu;

if(waitqueue_active(&vcpu->wq)) {

wake_up_interruptible(&vcpu->wq);

++vcpu->stat.halt_wakeup;

}

if(vcpu->guest_mode)

smp_call_function_single(ipi_pcpu, vcpu_kick_intr,

vcpu, 0, 0);

}

如果VCPU线程睡眠在等待队列上，则唤醒使其进入CPU的就绪任务队列。如果是多CPU的情况且目标CPU处于Guest模式，则需要发送核间中断。如果目标CPU正在执行Guest，那么这个IPI中断将导致VM exit，从而在下一次进入Guest时，可以注入中断。

事实上，目标CPU无须执行任何callback,也无须等待IPI返回，所以也无须使用smp_call_function_single，而是直接发送一个请求目标CPU重新调度的IPI即可，因此后来直接调用了函数smp_send_reschedule。函数smp_send_reschedule简单直接，直接发送了一个RESCHEDULE的IPI：

﻿commit 32f8840064d88cc3f6e85203aec7b6b57bebcb97

KVM: use smp_send_reschedule in kvm_vcpu_kick

﻿linux.git/arch/x86/kvm/x86.c

﻿void kvm_vcpu_kick(struct kvm_vcpu \*vcpu)

{

…

smp_send_reschedule(cpu);

…

}

﻿linux.git/arch/x86/kernel/smp.c

﻿static void native_smp_send_reschedule(int cpu)

{

…

apic->send_IPI_mask(cpumask_of(cpu), RESCHEDULE_VECTOR);

}

精彩回顾

[Linux系统虚拟化模型及障碍](http://mp.weixin.qq.com/s?__biz=MzAwMDUwNDgxOA==&mid=2652680510&idx=1&sn=db92933a82e8c848c54fbd9044077a91&chksm=810ff7a3b6787eb5f3814df0099bcab7624ff797ec92dd9477926b14d86dbfc54fe1995479d9&scene=21#wechat_redirect)

[Linux中断虚拟化（一）](http://mp.weixin.qq.com/s?__biz=MzAwMDUwNDgxOA==&mid=2652680521&idx=1&sn=07074c7bcd3d182fbaa312d1100e04d6&chksm=810ff7d4b6787ec255d0ccf67584b6b60f139c3fc2dd66dbc8d9989f4738245bd5e4d3c48d41&scene=21#wechat_redirect)

Reads 2097

​

Comment

**留言 1**

- 庭琹

  2021年12月2日

  Like1

  强烈建议，作者把所有虚拟化的知识放公众号上，为我们解读一遍！太棒了！

已无更多数据

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/YHBSoNHqDiaH5V6ycE2TlM9McXhAbaV9aUBaTAibVhLibXH1rL4JTMb2eoQfdCicTm4wycOsopO82jG3szvYF3mTJw/300?wx_fmt=png&wxfrom=18)

Linux阅码场

613

1

Comment

**留言 1**

- 庭琹

  2021年12月2日

  Like1

  强烈建议，作者把所有虚拟化的知识放公众号上，为我们解读一遍！太棒了！

已无更多数据
