
扬阜 阿里云云栖号

 _2022年01月26日 19:04_

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/tMJtfgIIibWKhCho0ocwzicZnZVUth8B7PYYEKRX7YichWOicCnTaZsGEEpmsOdYfZBymrvIuLbWZpQ0dgbicwLlgyQ/640?wx_fmt=jpeg&wxfrom=13&tp=wxpic)

作者 | 扬阜

###   

### **一 问题背景**

  

#### **1 实践验证**

  

工作中使用LLDB调试器调试这一段C++多继承程序的时候，发现通过lldb print(expression命令的别名) 命令获取的指针地址和实际理解的C++的内存模型的地址不一样。那么到底是什么原因呢？程序如下：

  

class Base {  
public:  
    Base(){}  
protected:  
    float x;  
};  
class VBase {  
public:  
    VBase(){}  
    virtual void test(){};  
    virtual void foo(){};  
protected:  
    float x;  
};  
class VBaseA: public VBase {  
public:  
    VBaseA(){}  
    virtual void test(){}  
    virtual void foo(){};  
protected:  
    float x;  
};  
class VBaseB: public VBase  {  
public:  
    VBaseB(){}  
    virtual void test(){  
        printf("test \n");  
    }  
    virtual void foo(){};  
protected:  
    float x;  
};  
class VDerived : public VBaseA, public Base, public VBaseB {  
public:  
    VDerived(){}  
    virtual void test(){}  
    virtual void foo(){};  
protected:  
    float x;  
};  
int  main(int argc, char *argv[])  
{  
    VDerived *pDerived = new VDerived(); //0x0000000103407f30  
    Base  *pBase = (Base*)pDerived; //0x0000000103407f40  
    VBaseA *pvBaseA = static_cast< VBaseA*>(pDerived);//0x0000000103407f30  
    VBaseB  *pvBaseB = static_cast< VBaseB*>(pDerived);//0x0000000103407f30 这里应该为0x0000000103407f48,但是显示的是0x0000000103407f30  
    unsigned long pBaseAddressbase = (unsigned long)pBase;  
    unsigned long pvBaseAAddressbase = (unsigned long)pvBaseA;  
    unsigned long pvBaseBAddressbase = (unsigned long)pvBaseB;  
    pvBaseB->test();  
}  

  

通过lldb print命令获取的地址如下图：  

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E "image.png")

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E "image.png")

  

正常理解的C++内存模型

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E "image.png")

  

由于我使用的是x86_64的mac系统，所以指针是8字节对齐,align=8。

  

按正常的理解的C++内存模型：pDerived转换为Base 类型pBase,地址偏移了16，是没问题的。

  

pDerived转化为VBaseA，由于共用了首地址为0x0000000103407f30，一样可以理解。pDerived转化为Base，地址偏移了16个字节(sizeof(VBaseA))为0x0000000103407f40,也是符合预期的。

  

但是pDerived转化为VBase 类型pBaseB内存地址应该偏移24，为0x0000000103407f48；而不是0x0000000103407f30(对象的首地址)，这个到底是什么原因引起的的呢？

  

#### **2 验证引发的猜测**

  

对于上面的这段代码

  

Base 类中没有虚函数，VBaseB 中有虚函数test和foo，猜测如下

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E "image.png")

  

1.不含有虚函数的(不含有虚表的)基类的指针，在类型转换时编译器对地址按照实际偏移。

  

2.含有虚函数的(含有虚表的)基类指针，在类型转换时，编译器实际上没有做地址的偏移，还是指向派生类，并没有指向实际的VBaseB类型。

  

### **二 现象带来的问题**

  

1.有虚函数的(含有虚表的)基类指针，在派生类类型转换为有虚函数的基类时，编译器背后有做真实的地址偏移吗？

  

2.如果做了偏移

  

- 那C++中在通过基类指针调用派生类重写的虚函数以及通过派生类指针调用虚函数的时候，编译器是如何保证这两种调用this指针的值是一样的，以确保调用的正确性的？
    

  

- 那为什么LLDB expression获取的地址是派生类对象的首地址呢？
    

  

3.如果没有做偏移，那是如何通过派生类的指针调用基类成员变量和函数的？

  

### **三 现象核心原因**

  

1.编译器背后和普通的非虚函数继承一样，也做了指针的偏移。

  

2.做了指针偏移，C++ 中基类对象指针调用派生类对象时,编译器通过thunk技术来实现每次参数调用和参数返回this地址的调整。

  

3.LLDB expression显示的是派生类对象的首地址(0x0000000103407f30),而不是偏移后基类对象的首地址(0x0000000103407f48),是由于LLDB调试器在expression向用户展示的时候,对于虚函数继承的基类指针LLDB内部会通过summary format来对要获取的结果进行格式化。summary format时，会根据当前的内存地址获取C++运行时的动态类型和地址，来向用户展示。

###   

### **四 证实结论过程**

  

#### **1 指针类型转换时编译器是否做了偏移？**

  

**汇编指令分析**

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E "image.png")

  

有虚函数的(含有虚表的)基类指针，在派生类类型转换为有虚函数的基类时,编译器背后有做真实的地址偏移吗？

  

基于上面的猜测，通过下面运行时反汇编的程序，来验证上面的猜测：

  

在开始反汇编程序之前，有一些下面要用到的汇编知识的普及。如果熟悉，可以忽略跳过。

  

注意：由于小编使用的是mac操作系统，所以处理器使用的是AT&T语法；和Intel语法不一样。

  

AT&T语法的指令是从左到右，第一个是源操作数，第二个是目的操作数，比如：

  

movl %esp, %ebp  //movl是指令名称。%则表明esp和ebp是寄存器.在AT&T语法中, 第一个是源操作数,第二个是目的操作数。  

  

而Intel指令是从右到左，第二个是源操作数，第一个是目的操作数

  

MOVQ EBP, ESP //interl手册，你会看到是没有%的intel语法, 它的操作数顺序刚好相反  

  

在x86_64的寄存器调用约定规定中

  

1.第一个参数基本上放在：RDI/edi寄存器,第二个参数：RSI/esi寄存器，第三个参数：RDX寄存器,第四个参数：RCD寄存器,第五个参数：R8寄存器,第六个参数：R9 寄存器；

  

2.如果超过六个参数在函数里就会通过栈来访问额外的参数；

  

3.函数返回值一般放在eax寄存器，或者rax寄存器。

  

下面使用的mac Unix操作系统，本文用到的汇编指令都是AT&T语法，在函数传参数时的第一个参数都放在RDI寄存器中。

  

下面是上面的main程序从开始执行到退出程序的所有汇编程序

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E "image.png")

  

通过上看的汇编代码我们发现编译器在做类型转换的时候不管是继承的基类有虚函数，还是没有虚函数，编译器都会做实际的指针偏移，偏移到实际的基类对象的地址，证明上面的猜测是错误的。编译器在类型转换的时候不区分有没有虚函数，都是实际做了偏移的。

  

**内存分析**

  

上面的猜测，后来我通过LLDB调试器提供的：memory read ptr（memory read 命令缩写 x ）得到了验证

  

(lldb) memory read pDerived  
0x103407f30: 40 40 00 00 01 00 00 00 00 00 00 00 00 00 00 00  @@..............  
0x103407f40: 10 00 00 00 00 00 00 00 60 40 00 00 01 00 00 00  ........`@......  
(lldb) memory read pvBaseB  
0x103407f48: 60 40 00 00 01 00 00 00 00 00 00 00 00 00 00 00  `@..............  
0x103407f58: de 2d 05 10 00 00 00 00 00 00 00 00 00 00 00 00  .-..............

  

我们发现不同类型的指针 在内存中确实读取到的内容分别是pDerived:0x103407f30 pvBaseB:0x103407f48内存地址都不一样；都是实际偏移后地址。

  

#### **2 虚函数调用如何保证this的值一致的呢？**

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E "image.png")

  

那既然内容中的真实地址是偏移后的，派生类重写了基类的虚函数，在通过基类指针调用派生类重新的虚函数的时候和通过派生类调用自身实现的虚函数的时候，编译器是如何保证这两种调用this指针的值是一样的，来确保调用的正确性的？

  

在网上查阅资料得知：C++在调用函数的时候， 编译器通过thunk技术对this指针的内容做了调整，使其指向正确的内存地址。那么什么是thunk技术？编译器是如何实现的呢？

  

**虚函数调用汇编指令分析**

  

通过上面main函数不难发现的pvBaseB->test() 的反汇编：

  

  pBaseB->test();  
    0x100003c84 < +244>: movq   -0x40(%rbp), %rax    //-x40存方的是pBaseB指针的内容，这里取出pBaseB指向的地址  
    0x100003c88 < +248>: movq   (%rax), %rcx         //然后将 rax的内容赋值给rcx  
    0x100003c8b < +251>: movq   %rax, %rdi           // 之后再将rax的值给到rdi寄存器：我们都知道，rdi寄存器是函数调用的第一个参数，这里的this是基类的地址  
->  0x100003c8e < +254>: callq  *(%rcx)              // 在这里取出rcx的地址，然后通过*(rcx) 间接调用rcx中存的地址

  

我们再跳到VDerived::test函数的汇编实现, 在这里通过lldb的命令：register read rdi 查看函数的第一个传参，也就是 this的地址，已经是派生类的地址了，不是调用前基类的地址

  

testCPPVirtualMemeory`VDerived::test:  
    0x100003e00 < +0>:  pushq  %rbp       //   栈低指针压栈     
    0x100003e01 < +1>:  movq   %rsp, %rbp //  将BP指针指向SP，因为上一级函数的栈顶指针是下一级函数的栈底指针  
    0x100003e04 < +4>:  subq   $0x10, %rsp  // 开始函数栈帧空间  
    0x100003e08 < +8>:  movq   %rdi, -0x8(%rbp)      //  将函数第一个参数入栈，也就是this 指针  
->  0x100003e0c < +12>: leaq   0x15c(%rip), %rdi         ; "test\n"    
    0x100003e13 < +19>: movb   $0x0, %al  
    0x100003e15 < +21>: callq  0x100003efc               ; symbol stub for: printf  
    0x100003e1a < +26>: addq   $0x10, %rsp //回收栈空间  
    0x100003e1e < +30>: popq   %rbp        //出栈 指回上一层 rbp  
    0x100003e1f < +31>: retq               //指向下一条命令  

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E "image.png")

  

通过上面的汇编我们分析，编译器在调用虚函数表中的函数时，是通过 *(%rcx) 间接寻址，然后中间做了某一个操作，跳到 test的实现，那么这个过程中thunk做了什么操作呢？

  

**llvm-thunk 源代码分析**

  

小编使用的IDE都使用的是LLVM编译器，于是通过翻看LLVM的源码找到了答案: 在VTableBuilder.cpp的AddMethods函数，小编找到了答案，描述如下：

  

  // Now go through all virtual member functions and add them to the current  
  // vftable. This is done by  
  //  - replacing overridden methods in their existing slots, as long as they  
  //    don't require return adjustment; calculating This adjustment if needed.  
  //  - adding new slots for methods of the current base not present in any  
  //    sub-bases;  
  //  - adding new slots for methods that require Return adjustment.  
  // We keep track of the methods visited in the sub-bases in MethodInfoMap.

  

编译器在编译的时候会判断基类的虚函数派生类有没有覆盖，如果有实现的时候，则动态替换虚函数表中的地址为派生类的地址，同时：

  

1.会计算调用时this指针的地址是否需要调整，如果需要调整的话，会为当前的方法开辟一块新的内存空间；

  

2.也会为需要this返回值的函数开辟一块新的内存空间；

  

代码如下：

  

void VFTableBuilder::AddMethods(BaseSubobject Base, unsigned BaseDepth,  
                                const CXXRecordDecl *LastVBase,  
                                BasesSetVectorTy &VisitedBases) {  
  const CXXRecordDecl *RD = Base.getBase();  
  if (!RD->isPolymorphic())  
    return;  
  
  const ASTRecordLayout &Layout = Context.getASTRecordLayout(RD);  
  
  // See if this class expands a vftable of the base we look at, which is either  
  // the one defined by the vfptr base path or the primary base of the current  
  // class.  
  const CXXRecordDecl *NextBase = nullptr, *NextLastVBase = LastVBase;  
  CharUnits NextBaseOffset;  
  if (BaseDepth < WhichVFPtr.PathToIntroducingObject.size()) {  
    NextBase = WhichVFPtr.PathToIntroducingObject[BaseDepth];  
    if (isDirectVBase(NextBase, RD)) {  
      NextLastVBase = NextBase;  
      NextBaseOffset = MostDerivedClassLayout.getVBaseClassOffset(NextBase);  
    } else {  
      NextBaseOffset =  
          Base.getBaseOffset() + Layout.getBaseClassOffset(NextBase);  
    }  
  } else if (const CXXRecordDecl *PrimaryBase = Layout.getPrimaryBase()) {  
    assert(!Layout.isPrimaryBaseVirtual() &&  
           "No primary virtual bases in this ABI");  
    NextBase = PrimaryBase;  
    NextBaseOffset = Base.getBaseOffset();  
  }  
  
  if (NextBase) {  
    AddMethods(BaseSubobject(NextBase, NextBaseOffset), BaseDepth + 1,  
               NextLastVBase, VisitedBases);  
    if (!VisitedBases.insert(NextBase))  
      llvm_unreachable("Found a duplicate primary base!");  
  }  
  
  SmallVector< const CXXMethodDecl*, 10> VirtualMethods;  
  // Put virtual methods in the proper order.  
  GroupNewVirtualOverloads(RD, VirtualMethods);  
  
  // Now go through all virtual member functions and add them to the current  
  // vftable. This is done by  
  //  - replacing overridden methods in their existing slots, as long as they  
  //    don't require return adjustment; calculating This adjustment if needed.  
  //  - adding new slots for methods of the current base not present in any  
  //    sub-bases;  
  //  - adding new slots for methods that require Return adjustment.  
  // We keep track of the methods visited in the sub-bases in MethodInfoMap.  
  for (const CXXMethodDecl *MD : VirtualMethods) {  
    FinalOverriders::OverriderInfo FinalOverrider =  
        Overriders.getOverrider(MD, Base.getBaseOffset());  
    const CXXMethodDecl *FinalOverriderMD = FinalOverrider.Method;  
    const CXXMethodDecl *OverriddenMD =  
        FindNearestOverriddenMethod(MD, VisitedBases);  
  
    ThisAdjustment ThisAdjustmentOffset;  
    bool ReturnAdjustingThunk = false, ForceReturnAdjustmentMangling = false;  
    CharUnits ThisOffset = ComputeThisOffset(FinalOverrider);  
    ThisAdjustmentOffset.NonVirtual =  
        (ThisOffset - WhichVFPtr.FullOffsetInMDC).getQuantity();  
    if ((OverriddenMD || FinalOverriderMD != MD) &&  
        WhichVFPtr.getVBaseWithVPtr())  
      CalculateVtordispAdjustment(FinalOverrider, ThisOffset,  
                                  ThisAdjustmentOffset);  
  
    unsigned VBIndex =  
        LastVBase ? VTables.getVBTableIndex(MostDerivedClass, LastVBase) : 0;  
  
    if (OverriddenMD) {  
      // If MD overrides anything in this vftable, we need to update the  
      // entries.  
      MethodInfoMapTy::iterator OverriddenMDIterator =  
          MethodInfoMap.find(OverriddenMD);  
  
      // If the overridden method went to a different vftable, skip it.  
      if (OverriddenMDIterator == MethodInfoMap.end())  
        continue;  
  
      MethodInfo &OverriddenMethodInfo = OverriddenMDIterator->second;  
  
      VBIndex = OverriddenMethodInfo.VBTableIndex;  
  
      // Let's check if the overrider requires any return adjustments.  
      // We must create a new slot if the MD's return type is not trivially  
      // convertible to the OverriddenMD's one.  
      // Once a chain of method overrides adds a return adjusting vftable slot,  
      // all subsequent overrides will also use an extra method slot.  
      ReturnAdjustingThunk = !ComputeReturnAdjustmentBaseOffset(  
                                  Context, MD, OverriddenMD).isEmpty() ||  
                             OverriddenMethodInfo.UsesExtraSlot;  
  
      if (!ReturnAdjustingThunk) {  
        // No return adjustment needed - just replace the overridden method info  
        // with the current info.  
        MethodInfo MI(VBIndex, OverriddenMethodInfo.VFTableIndex);  
        MethodInfoMap.erase(OverriddenMDIterator);  
  
        assert(!MethodInfoMap.count(MD) &&  
               "Should not have method info for this method yet!");  
        MethodInfoMap.insert(std::make_pair(MD, MI));  
        continue;  
      }  
  
      // In case we need a return adjustment, we'll add a new slot for  
      // the overrider. Mark the overridden method as shadowed by the new slot.  
      OverriddenMethodInfo.Shadowed = true;  
  
      // Force a special name mangling for a return-adjusting thunk  
      // unless the method is the final overrider without this adjustment.  
      ForceReturnAdjustmentMangling =  
          !(MD == FinalOverriderMD && ThisAdjustmentOffset.isEmpty());  
    } else if (Base.getBaseOffset() != WhichVFPtr.FullOffsetInMDC ||  
               MD->size_overridden_methods()) {  
      // Skip methods that don't belong to the vftable of the current class,  
      // e.g. each method that wasn't seen in any of the visited sub-bases  
      // but overrides multiple methods of other sub-bases.  
      continue;  
    }  
  
    // If we got here, MD is a method not seen in any of the sub-bases or  
    // it requires return adjustment. Insert the method info for this method.  
    MethodInfo MI(VBIndex,  
                  HasRTTIComponent ? Components.size() - 1 : Components.size(),  
                  ReturnAdjustingThunk);  
  
    assert(!MethodInfoMap.count(MD) &&  
           "Should not have method info for this method yet!");  
    MethodInfoMap.insert(std::make_pair(MD, MI));  
  
    // Check if this overrider needs a return adjustment.  
    // We don't want to do this for pure virtual member functions.  
    BaseOffset ReturnAdjustmentOffset;  
    ReturnAdjustment ReturnAdjustment;  
    if (!FinalOverriderMD->isPure()) {  
      ReturnAdjustmentOffset =  
          ComputeReturnAdjustmentBaseOffset(Context, FinalOverriderMD, MD);  
    }  
    if (!ReturnAdjustmentOffset.isEmpty()) {  
      ForceReturnAdjustmentMangling = true;  
      ReturnAdjustment.NonVirtual =  
          ReturnAdjustmentOffset.NonVirtualOffset.getQuantity();  
      if (ReturnAdjustmentOffset.VirtualBase) {  
        const ASTRecordLayout &DerivedLayout =  
            Context.getASTRecordLayout(ReturnAdjustmentOffset.DerivedClass);  
        ReturnAdjustment.Virtual.Microsoft.VBPtrOffset =  
            DerivedLayout.getVBPtrOffset().getQuantity();  
        ReturnAdjustment.Virtual.Microsoft.VBIndex =  
            VTables.getVBTableIndex(ReturnAdjustmentOffset.DerivedClass,  
                                    ReturnAdjustmentOffset.VirtualBase);  
      }  
    }  
  
    AddMethod(FinalOverriderMD,  
              ThunkInfo(ThisAdjustmentOffset, ReturnAdjustment,  
                        ForceReturnAdjustmentMangling ? MD : nullptr));  
  }  
}  

  

通过上面代码分析，在this 需要调整的时候，都是通过AddMethod(FinalOverriderMD，ThunkInfo(ThisAdjustmentOffset, ReturnAdjustment，ForceReturnAdjustmentMangling ? MD : nullptr))函数来添加一个ThunkInfo的结构体，ThunkInfo在结构体(实现在ABI.h)如下：

  

struct ThunkInfo {  
  /// The \c this pointer adjustment.  
  ThisAdjustment This;  
  
  /// The return adjustment.  
  ReturnAdjustment Return;  
  
  /// Holds a pointer to the overridden method this thunk is for,  
  /// if needed by the ABI to distinguish different thunks with equal  
  /// adjustments. Otherwise, null.  
  /// CAUTION: In the unlikely event you need to sort ThunkInfos, consider using  
  /// an ABI-specific comparator.  
  const CXXMethodDecl *Method;  
  
  ThunkInfo() : Method(nullptr) { }  
  
  ThunkInfo(const ThisAdjustment &This, const ReturnAdjustment &Return,  
            const CXXMethodDecl *Method = nullptr)  
      : This(This), Return(Return), Method(Method) {}  
  
  friend bool operator==(const ThunkInfo &LHS, const ThunkInfo &RHS) {  
    return LHS.This == RHS.This && LHS.Return == RHS.Return &&  
           LHS.Method == RHS.Method;  
  }  
  
  bool isEmpty() const {  
    return This.isEmpty() && Return.isEmpty() && Method == nullptr;  
  }  
};  
}  

  

Thunkinfo的结构体有一个method,存放函数的真正实现，This和Return记录this需要调整的信息，然后在生成方法的时候，根据这些信息，编译器自动插入thunk函数的信息，通过ItaniumMangleContextImpl::mangleThunk(const CXXMethodDecl *MD,const ThunkInfo &Thunk,raw_ostream &Out)的函数，我们得到了证实，函数如下：

  

（mangle和demangle：将C++源程序标识符(original C++ source identifier)转换成C++ ABI标识符(C++ ABI identifier)的过程称为mangle；相反的过程称为demangle。wiki）

  

void ItaniumMangleContextImpl::mangleThunk(const CXXMethodDecl *MD,  
                                           const ThunkInfo &Thunk,  
                                           raw_ostream &Out) {  
  //  < special-name> ::= T < call-offset> < base encoding>  
  //                      # base is the nominal target function of thunk  
  //  < special-name> ::= Tc < call-offset> < call-offset> < base encoding>  
  //                      # base is the nominal target function of thunk  
  //                      # first call-offset is 'this' adjustment  
  //                      # second call-offset is result adjustment  
  
  assert(!isa< CXXDestructorDecl>(MD) &&  
         "Use mangleCXXDtor for destructor decls!");  
  CXXNameMangler Mangler(*this, Out);  
  Mangler.getStream() << "_ZT";  
  if (!Thunk.Return.isEmpty())  
    Mangler.getStream() << 'c';  
  
  // Mangle the 'this' pointer adjustment.  
  Mangler.mangleCallOffset(Thunk.This.NonVirtual,  
                           Thunk.This.Virtual.Itanium.VCallOffsetOffset);  
  
  // Mangle the return pointer adjustment if there is one.  
  if (!Thunk.Return.isEmpty())  
    Mangler.mangleCallOffset(Thunk.Return.NonVirtual,  
                             Thunk.Return.Virtual.Itanium.VBaseOffsetOffset);  
  
  Mangler.mangleFunctionEncoding(MD);  
}  

  

**thunk 汇编指令分析**

  

至此，通过LLVM源码我们解开了thunk技术的真面目，那么我们通过反汇编程序来验证证实一下, 这里使用objdump 或者逆向利器 hopper都可以，小编使用的是hopper,汇编代码如下：

  

1.我们先来看编译器实现的thunk 版的test函数

  

派生类实现的test函数

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E "image.png")

  

编译器实现的thunk版的test函数

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E "image.png")

  

2.通过上面两张截图我们发现

  

编译器实现的thunk的test函数地址为0x100003e30

  

派生类实现的test函数地址为0x100003e00

  

下面我们来看下派生类的虚表中存的真实地址是那一个

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E "image.png")

  

通过上图我们可以看到：派生类的虚表中存的真实地址为编译器动态添加的thunk函数的地址0x100003e30。

  

上面分析的*(rcx)间接寻址：就是调用thunk函数的实现，然后在thunk中去调用真正的派生类覆盖的函数。

  

在这里我们可以确定的 thunk技术：

  

就是编译器在编译的时候,遇到调用this和返回值this需要调整的地方，动态的加入对应的thunk版的函数,在thunk函数的内部实现this的偏移调整，和调用派生类实现的虚函数；并将编译器实现的thunk函数的地址存入虚表中，而不是派生类实现的虚函数的地址。

  

**thunk 函数的内存布局**

  

也可以确定对应的内存布局如下：

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E "image.png")

  

故（继承链中不是第一个）虚函数继承的基类指针的调用顺序为：

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E "image.png")

  

**virtual-thunk 和 non-virtual-thunk**

  

注意：在这里可以看到，内存中有两份VBase,在多继承中分为普通继承、虚函数继承、虚继承。虚继承主要是为了解决上面看到的问题：在内存中同时有两份Vbase 的内存，将上面的代码改动一下就会确保内存中的实例只有一份：

  

class VBaseA: public VBase 改成 class VBaseA: public virtual VBase

  

class VBaseB: public VBase 改成 class VBaseB: public virtual VBase

  

这样内存中的VBase就只有一分内存了。

  

到这里还有问题没有解答，就是上面截图里的thunk函数类型是：

  

我们发现thunk函数是 non-virtual-thunk类型，那对应的virtual-thunk是什么类型呢？

  

在解答这个问题之前我们现看下下面的例子？

  

public A {  
    virtual void test() {  
    }  
}  
public B {  
    virtual void test1() {  
    }  
}  
public C {  
    virtual void test2() {  
    }  
}  
public D : public virtual A, public virtual B, public C {  
     virtual void test1() { // 这里实现的test1函数在 B类的虚函数表里就是virtual-trunk的类型  
     }  
     virtual void test2() { // 这里实现的test2函数在 C类的虚函数表示就是no-virtual-trunk的类型  
     }  
}  

  

虚函数继承和虚继承相结合，且该类在派生类的继承链中不是第一个基类的时候，则该派生类实现的虚函数在编译器编译的时候，虚表里存放就是virtual-trunk类型。

  

只有虚函数继承的时候，且该类在派生类的继承链中不是第一个基类的时候，则该派生类实现的虚函数在编译器编译的时候，虚表里存放就是no-virtual-trunk类型。

  

#### **3 为什么LLDB调试器显示的地址一样呢？**

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E "image.png")

  

如果做了偏移，那为什么LLDB expression显示的地址是派生类对象的首地址呢？

  

到了现在了解了什么是thunk技术，还没有一个问题没有解决：就是LLDB调试的时候，显示的this的地址是基类偏移后的(派生类的地址)，前面通过汇编分析编译器在类型转换的时候，做了真正的偏移，通过读取内存地址也发现是偏移后的真实地址，那lldb expression获取的地址为啥还是派生类的地址呢？由此可以猜测是LLDB调试器通过exppress 命令执行的时候做了类型的转换。

  

通过翻阅LLDB调试器的源码和LLDB说明文档，通过文档得知LLDB在每次拿到一个地址，需要向用户友好的展示的时候，首先需要通过summary format()进行格式化转换，格式化转化的依据是动态类型(lldb-getdynamictypeandaddress)的获取，在LLDB源码的bool ItaniumABILanguageRuntime::GetDynamicTypeAndAddress (lldb-summary-format)函数中找到了答案，代码如下

  

// For Itanium, if the type has a vtable pointer in the object, it will be at  
  // offset 0  
  // in the object.  That will point to the "address point" within the vtable  
  // (not the beginning of the  
  // vtable.)  We can then look up the symbol containing this "address point"  
  // and that symbol's name  
  // demangled will contain the full class name.  
  // The second pointer above the "address point" is the "offset_to_top".  We'll  
  // use that to get the  
  // start of the value object which holds the dynamic type.  

  

bool ItaniumABILanguageRuntime::GetDynamicTypeAndAddress(  
    ValueObject &in_value, lldb::DynamicValueType use_dynamic,  
    TypeAndOrName &class_type_or_name, Address &dynamic_address,  
    Value::ValueType &value_type) {  
  // For Itanium, if the type has a vtable pointer in the object, it will be at  
  // offset 0  
  // in the object.  That will point to the "address point" within the vtable  
  // (not the beginning of the  
  // vtable.)  We can then look up the symbol containing this "address point"  
  // and that symbol's name  
  // demangled will contain the full class name.  
  // The second pointer above the "address point" is the "offset_to_top".  We'll  
  // use that to get the  
  // start of the value object which holds the dynamic type.  
  //  
  
  class_type_or_name.Clear();  
  value_type = Value::ValueType::eValueTypeScalar;  
  
  // Only a pointer or reference type can have a different dynamic and static  
  // type:  
  if (CouldHaveDynamicValue(in_value)) {  
    // First job, pull out the address at 0 offset from the object.  
    AddressType address_type;  
    lldb::addr_t original_ptr = in_value.GetPointerValue(&address_type);  
    if (original_ptr == LLDB_INVALID_ADDRESS)  
      return false;  
  
    ExecutionContext exe_ctx(in_value.GetExecutionContextRef());  
  
    Process *process = exe_ctx.GetProcessPtr();  
  
    if (process == nullptr)  
      return false;  
  
    Status error;  
    const lldb::addr_t vtable_address_point =  
        process->ReadPointerFromMemory(original_ptr, error);  
  
    if (!error.Success() || vtable_address_point == LLDB_INVALID_ADDRESS) {  
      return false;  
    }  
  
    class_type_or_name = GetTypeInfoFromVTableAddress(in_value, original_ptr,  
                                                      vtable_address_point);  
  
    if (class_type_or_name) {  
      TypeSP type_sp = class_type_or_name.GetTypeSP();  
      // There can only be one type with a given name,  
      // so we've just found duplicate definitions, and this  
      // one will do as well as any other.  
      // We don't consider something to have a dynamic type if  
      // it is the same as the static type.  So compare against  
      // the value we were handed.  
      if (type_sp) {  
        if (ClangASTContext::AreTypesSame(in_value.GetCompilerType(),  
                                          type_sp->GetForwardCompilerType())) {  
          // The dynamic type we found was the same type,  
          // so we don't have a dynamic type here...  
          return false;  
        }  
  
        // The offset_to_top is two pointers above the vtable pointer.  
        const uint32_t addr_byte_size = process->GetAddressByteSize();  
        const lldb::addr_t offset_to_top_location =  
            vtable_address_point - 2 * addr_byte_size;  
        // Watch for underflow, offset_to_top_location should be less than  
        // vtable_address_point  
        if (offset_to_top_location >= vtable_address_point)  
          return false;  
        const int64_t offset_to_top = process->ReadSignedIntegerFromMemory(  
            offset_to_top_location, addr_byte_size, INT64_MIN, error);  
  
        if (offset_to_top == INT64_MIN)  
          return false;  
        // So the dynamic type is a value that starts at offset_to_top  
        // above the original address.  
        lldb::addr_t dynamic_addr = original_ptr + offset_to_top;  
        if (!process->GetTarget().GetSectionLoadList().ResolveLoadAddress(  
                dynamic_addr, dynamic_address)) {  
          dynamic_address.SetRawAddress(dynamic_addr);  
        }  
        return true;  
      }  
    }  
  }  
  
  return class_type_or_name.IsEmpty() == false;  
}  

  

通过上面代码分析可知，每次在通过LLDB 命令expression动态调用 指针地址的时候，LLDB 会去按照调试器默认的格式进行格式化，格式化的前提是动态获取到对应的类型和偏移后的地址；在碰到C++有虚表的时候，且不是虚表中的第一个基类指针的时候，就会使用指针上头的offset_to_top 获取到这个对应动态的类型和返回动态获取的该类型对象开始的地址。

  

### **五 总结**

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E "image.png")

  

1.上面主要验证了在指针类型转换的时候,编译器内部做了真实的地址偏移；

  

2.通过上面的分析，我们得知编译器在函数调用时通过thunk技术动态调整入参this指针和返回值this指针，保证C++调用时this的正确性；

  

3.在通过LLDB expression获取非虚函数基类指针内容时，LLDB内部通过summary format进行格式化转换，格式化转化时会进行动态类型的获取。

###   

### **六 工具篇**

  

#### **1 获取汇编程序**

  

**预处理->汇编**

clang++ -E main.cpp -o main.i  
clang++ -S main.i

  

**objdump**

  

objdump -S -C 可执行程序

  

**反汇编利器: hopper**

下载hopper，可执行程序拖入即可

  

**Xcode**

Xcode->Debug->Debug WorkFlow->Show disassembly

#### 

#### **2 导出C++内存布局**

**Clang++编译器**

  

clang++ -cc1 -emit-llvm -fdump-record-layouts -fdump-vtable-layouts main.cpp

###   

### **七 参考文献**

  

https://matklad.github.io/2017/10/21/lldb-dynamic-type.html  
https://lldb.llvm.org/use/variable.html  
https://github.com/llvm-mirror/lldb/blob/bc19e289f759c26e4840aab450443d4a85071139/source/Plugins/LanguageRuntime/CPlusPlus/ItaniumABI/ItaniumABILanguageRuntime.cpp#L185  
https://clang.llvm.org/doxygen/VTableBuilder_8cpp_source.html#l03109  
https://clang.llvm.org/doxygen/ABI_8h_source.html

  

相关技术：

  

llvm-virtual-thunk  
llvm-no-virtual-thunk  
lldb-summary-format  
lldb-getdynamictypeandaddress

  

**精****彩推荐**

  

[![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)](http://mp.weixin.qq.com/s?__biz=MzI0NTE4NjA0OQ==&mid=2658380746&idx=1&sn=9863e66db5b3c90fb3be1528b5a5cb9e&chksm=f2d560a4c5a2e9b28fdb9e2933256d9f1714576ade62788acfc95f888e46f63e5824123796b8&scene=21#wechat_redirect)

# [IT人的年夜饭，也太香了吧！](http://mp.weixin.qq.com/s?__biz=MzI0NTE4NjA0OQ==&mid=2658380746&idx=1&sn=9863e66db5b3c90fb3be1528b5a5cb9e&chksm=f2d560a4c5a2e9b28fdb9e2933256d9f1714576ade62788acfc95f888e46f63e5824123796b8&scene=21#wechat_redirect)

  

[![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)](http://mp.weixin.qq.com/s?__biz=MzI0NTE4NjA0OQ==&mid=2658380693&idx=1&sn=3ce1caf0f870a39a573845e8acc72ab7&chksm=f2d560fbc5a2e9ed1b5d5618d1f584fa567a30e08fa96fa8223c18de6e2a3336a62625da8c4a&scene=21#wechat_redirect)

# [一封来自工信部的感谢信](http://mp.weixin.qq.com/s?__biz=MzI0NTE4NjA0OQ==&mid=2658380693&idx=1&sn=3ce1caf0f870a39a573845e8acc72ab7&chksm=f2d560fbc5a2e9ed1b5d5618d1f584fa567a30e08fa96fa8223c18de6e2a3336a62625da8c4a&scene=21#wechat_redirect)

![](http://mmbiz.qpic.cn/mmbiz_png/tMJtfgIIibWJZdQ8EicOpoDF9lteAE7gGBotfOe7nZWKicaoH7URJUKISmAkBp7SiakzBrEqib5ZeZYrmIUClJUt1vQ/300?wx_fmt=png&wxfrom=19)

**阿里云云栖号**

阿里云官网内容平台，汇聚阿里云优质内容（入门、文档、案例、最佳实践、直播等）。

520篇原创内容

公众号

**点击上方  一键关注**

**从现在开始 学习技术**  

  

  

**↓ 新年来临 阿里云最新、最热、最全的活动已全面开启 🔔**  

阅读原文

阅读 621

​