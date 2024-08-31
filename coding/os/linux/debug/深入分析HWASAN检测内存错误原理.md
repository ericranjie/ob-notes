
Original 字节跳动STE团队 字节跳动SYS Tech

 _2023年07月21日 19:33_ _北京_

导语：ASAN(AddressSanitizer) 是 C/C++开发者常用的内存错误检测工具，主要用于检测缓冲区溢出、访问已释放的内存等内存错误。AArch64 上提供了 Top-Byte-Ingore 硬件特性，HWASan(HardWare-assisted AddressSanitizer) 就是利用 Top-Byte-Ignore 特性实现的增强版 ASan，与 ASAN 相比 HWASan 的内存开销更低，检测到的内存错误范围更大。因此在 AArch64 平台，建议使用 HWASAN。本篇文章将深入分析 HWASAN 检测内存错误的原理，帮助大家更好地理解和使用 HWASan 来排查程序中存在的疑难内存错误。


**前言**

在字节跳动，C++语言被广泛应用在各个业务中，由于C++语言的特性，导致 C++ 程序很容易出现内存问题。ASAN 等内存检测工具在字节跳动内部已经取得了可观的收益和效果（更多内容请查看视频分享：Sanitizer 在字节跳动 C/C++ 业务中的实践：https://www.bilibili.com/video/BV1YT411Q7BU/），服务于60个业务线，近一年协助修复上百个内存缺陷。但是仍然有很大的提升空间，特别是在性能开销方面。随着 ARM 进入服务器芯片市场，ARM 架构下的一些硬件特性可以用来缓解 ASAN 工具的性能问题，利用这些硬件特性研发的 HWASAN 检测工具在超大型 C++ 服务上的检测能力还有待确认。

  
为此，STE 团队对 HWASAN 进行了深入分析，并在字节跳动 C++ 核心服务上进行了落地验证。在落地 HWASAN 过程中，修复了 HWASAN 实现中的一些关键 bug，并对易用性进行了提升。相关 patch 已经贡献到LLVM开源社区（详情请查看文末链接）。本篇文章将深入分析 HWASAN 检测内存错误的原理，帮助大家更好地理解和使用 HWASan 来排查程序中存在的疑难内存错误。


**概述**

HWASAN: **H**ard**W**are-assisted **A**ddress**San**itizer, a tool similar to AddressSanitizer, but based on partial hardware assistance and consumes much less memory.

  

这里所谓的 "partial hardware assistance" 就是指 AArch64 的 TBI (Top Byte Ignore) 特性。

  

> **TBI** (Top Byte Ignore) feature of AArch64: bits [63:56] are ignored in address translation and can be used to store a tag.

  

以如下代码举例，Linux/AArch64 下将指针 x 的 top byte 设置为 0xfe，不影响程序执行：

```
// $ cat tbi.cpp
```

AArch64 的 TBI 特性使得软件可以在 64-bit 虚拟地址的最高字节中存储任意数据，HWASAN 正是基于 TBI 这一特性设计并实现的内存错误检测工具。

  

举个例子，以下代码中存在

```
// cat test.c
```

使用 HWASAN 检测上述代码中的 heap-buffer-overflow bug：

```
$ clang -fuse-ld=lld -g -fsanitize=hwaddress ./test.c && ./a.out
```

如上所示，HWASAN 与 ASAN 相比不管是用法 (-fsanitize=hwaddress v.s. -fsanitize=address) 还是检测到错误后的报告都很相似。

  

下面对比分析 ASAN 与 HWASAN 检测内存错误的技术原理：

**ASAN (AddressSanitizer)**

- 使用 shadow memory 技术，每 8-bytes 的 application memory 对应 1-byte 的 shadow memory。
    
- 使用 redzone 来检测 buffer-overflow。不管是栈内存还是堆内存，在申请内存时都会在原本内存的两侧额外申请一定大小的内存作为 redzone，一旦访问到了 redzone 则说明发生了缓冲区溢出。
    
- 使用 quarantine 检测 use-after-free。应用程序执行 delete 或 free 释放内存时，并不真正释放，而是放在一个暂存区 (quarantine) 中，一旦访问了位于 quarantine 中的内存则说明访问了已释放的内存。
    
- 每 1-byte 的 shadow memory 编码表示对应的 8-byte application memory 的信息，每次访问 application memory 之前 ASAN 都会检查对应的 shadow memory 判断本次内存访问是否合法。例如：shadow memory 的值为 0xfd 表示对应的 8-bytes application memory 是 freed memory，所以当访问的 application memory 其 shadow memory byte 为 0xfd 就说明此时访问的是已经释放的内存了即 use-after-free；shadow memory 为 0xfa 表示相应的 application memory 是堆内存的 redzone，所以当访问到的 appllcation memory 其 shadow memory 为 0xfa 就说明此时访问的是堆内存附近的 redzone 发生了堆缓冲区溢出错误即 heap-buffer-overflow
    

**HWASAN (HardWare-assisted AddressSanitizer)**

- 同样使用 shadow memory 技术，不过与 ASAN 不同的是：HWASAN 每 16-bytes 的 application memory 对应 1-byte 的 shadow memory。
    
- 不依赖 redzone 检测 buffer-overflow，不依赖 quarantine 检测 use-after-free，仅基于 TBI 特性就能检测 buffer-overflow 和 use-after-free。
    
- 举例说明 HWASAN 检测内存错误的原理
    
- 因为 HWASAN 会将每 16-bytes 的 application memory 都对应 1-byte 的 shadow tag，所以 HWASAN 会将申请的内存都对齐到 16-bytes，因此下图中 new char[20] 实际申请的内存是 32-bytes。
    
- HWASAN 会生成一个随机 tag 保存在 operator new 返回的指针 p 的 top byte 中，同时会将 tag 保存在 p 指向的内存对应 shadow memory 中。
    

为了方便说明，下图中用不同的颜色表示不同的 tag，绿色表示 tag 0xa，蓝色表示 tag 0xb，紫色表示 tag 0xc。

- **检测 heap-buffer-overflow**：假设 HWASAN 为 new char[20] 生成的 tag 为 0xa 即绿色，所以指针 p 的 top byte 为 0xa。执行 delete[] p 释放内存时，HWASAN 将这块释放的内存 retag 为紫色，即将这块释放的内存对应的 shadow memory 从绿色修改为紫色。在通过 p[0] 访问内存时，HWASAN 会检查保存在指针 p 的 tag 与 p[0] 指向的内存所对应的 shadow memory 中保存的 tag 是否一致。显然保存在指针 p 的 tag 是绿色 而p[0] 指向的内存所对应的 shadow memory 中保存的 tag 是紫色，即 tag 是不匹配的，这说明访问 p[0] 时存在内存错误。
    

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

- **检测 use-after-free**：假设 HWASAN 为 new char[20] 生成的 tag 为 0xa 即绿色，所以指针 p 的 top byte 为 0xa。在通过 p[32] 访问内存时，HWASAN 会检查保存在指针 p 的 tag 与 p[32] 指向的内存所对应的 shadow memory 中保存的 tag 是否一致。显然保存在指针 p 的 tag 是绿色 而p[32] 指向的内存所对应的 shadow memory 中保存的 tag 是蓝色，即 tag 是不匹配的，这说明访问 p[32] 时存在内存错误。
    

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

**算法**

- shadow memory：每 16-bytes 的 application memory 对应 1-byte 的 shadow memory。
    
- 每个 heap/stack/global 内存对象都被对齐到 16-bytes，这样每个 heap/stack/global 内存对象至少对应的 1-byte shadow memory。
    
- 为每一个 heap/stack/global 内存对象生成一个 1-byte 的随机 tag，将该随机 tag 保存到指向这些内存对象的指针的 top byte 中，同样将该随机 tag 保存到这些内存对象对应的 shadow memory 中。
    
- 在每一处内存读写之前插桩：比较保存在指针 top byte 的 tag 和保存在 shadow memory 中的 tag 是否一致，如果不一致则报错。
    

**实现**

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**shadow mapping**

HWASAN 与 ASAN 一样都使用了 shadow memory 技术。ASAN 默认使用 static shadow mapping，只有对 IOS 和 32-bit Android 平台才使用 dynamic shadow mapping。而 HWASAN 则总是使用 dynamic shadow mapping。

- ASAN: static shadow mapping。在 llvm-project/compiler-rt/lib/asan/asan_mapping.h 中预定义了不同平台下 shadow memory 的布局：HighMem, HighShadow, ShadowGap, LowShadow, LowMem 的地址区间。
    
- Linux/x86_64 下 ASAN 的 shadow mapping 如下所示：
    

```
// Typical shadow mapping on Linux/x86_64 with SHADOW_OFFSET == 0x00007fff8000:
```

- 给定 application memory 地址 addr，计算其对应的 shadow memory 地址的公式如下：
    

```
uptr MemToShadow(uptr addr) { return (addr >> 3) + 0x7fff8000; }
```

- HWASAN: dynamic shadow mapping。根据 MaxUserVirtualAddress 计算 shadow memory 所需要的总大小 shadow_size，通过 mmap(shadow_size) 得到 shadow memory 区间，再具体划分 HighMem, HighShadow, ShadowGap, LowShadow, LowMem 的地址区间。
    
- 伪算法如下（未考虑对齐）：
    

```
kHighMemEnd = GetMaxUserVirtualAddress();shadow_size = MemToShadowSize(kHighMemEnd);__hwasan_shadow_memory_dynamic_address = mmap(shadow_size);// Place the low memory first.kLowMemEnd = __hwasan_shadow_memory_dynamic_address - 1;kLowMemStart = 0;// Define the low shadow based on the already placed low memory.kLowShadowEnd = MemToShadow(kLowMemEnd);kLowShadowStart = __hwasan_shadow_memory_dynamic_address;// High shadow takes whatever memory is left up there.kHighShadowEnd = MemToShadow(kHighMemEnd);kHighShadowStart = Max(kLowMemEnd, MemToShadow(kHighShadowEnd)) + 1;// High memory starts where allocated shadow allows.kHighMemStart = ShadowToMem(kHighShadowStart);uptr MemToShadow(uptr untagged_addr) {  return (untagged_addr >> 4) + __hwasan_shadow_memory_dynamic_address;}uptr ShadowToMem(uptr shadow_addr) {  return (shadow_addr - __hwasan_shadow_memory_dynamic_address) << 4;}
```

```
uptr MemToShadow(uptr untagged_addr) {
```

- Linux/AArch64 下 HWASAN 的某种 shadow mapping 如下所示：
    

```
// Typical mapping on Linux/AArch64
```

  

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**tagging**

- stack 内存对象：在编译时插桩阶段，HWASAN 会插入代码来实现对 stack 内存对象的 tagging，将生成的随机 tag 保存在 stack 内存对象指针的 top byte，同时将随机 tag 保存这些 stack 内存对象对应的 shadow memory 中。
    
- 每个 stack 内存对象的 tag 是通过 stack_base_tag ^ RetagMask(AllocaNo) 计算得到的。stack_base_tag 对于不同的 stack frame 是不同的值，RetagMask(AllocaNo) 的实现如下（AllocaNo 可以看作是 stack 内存对象的序号）。
    

```
  static unsigned RetagMask(unsigned AllocaNo) {
```

- heap 内存对象：随机 tag 是在运行时申请 heap 内存对象时由 HWASAN 的内存分配器生成的。
    
- 当程序调用 malloc 或者 operator new 申请内存时，HWASAN 的内存分配器会将随机生成的 tag 保存在 malloc 或者 operator new 返回的指针的 top byte 中，同时将随机 tag 保存在这些 heap 内存对象对应的 shadow memory 中。
    
- 当程序调用 free 或者 operator delete 释放内存时，HWASAN 的内存分配器会再次随机生成 tag ，将新生成的随机 tag 保存在正释放的 heap 内存对象对应的 shadow memory 中。
    
- global 内存对象：对于每个编译单元内的 global 内存对象，在编译时插桩阶段都会生成对应的 tag，编译单元内的第一个 global 内存对象的 tag 是对当前编译单元文件路径计算 MD5 哈希值然后取第一个字节得到的，编译单元内的后续 global 内存对象的 tag 是基于之前 global 内存对象的 tag 递增得到的。
    
- 在编译时插桩阶段 HWASAN 会插入代码来将生成的随机 tag 保存在 global 内存对象指针的 top byte，而将随机 tag 保存到这些 global 内存对象对应的 shadow memory 中则是由 HWASAN runtime 在程序启动阶段做的。对于每一个 global 内存对象，HWASAN 在编译插桩阶段都会创建一个 descriptor 保存在 "hwasan_globals" section 中，descriptor 中保存了 global 内存对象的大小以及 tag 等信息，程序启动时 HWASAN runtime 会遍历 "hwasan_globals" section 中所有的 descriptor 来设置每个 global 内存对象对应的 shadow memory tag。
    
      
    

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**short granules**

每个 heap/stack/global 内存对象都会被对齐到 16-bytes，heap/stack/global 内存对象原本大小记作 size，如果 size % 16 != 0，那么就需要 padding，heap/stack/global 内存对象最后不足 16-bytes 的部分就被称为 short granule。此时会将 tag 存储到 padding 的最后一个字节，而 padding 所在的 16-bytes 内存对应的 1-byte shadow memory 中存储的则是 short granule size 即 size % 16。

  

举例如下：

```
uint8_t buf[20];
```

uint8_t buf[20] 开启 HWASAN 后会变为：

```
uint8_t buf[32]; // 20-bytes aligned to 16-bytes -> 32-bytes
```

因为 short granules 的存在，所以在比较保存在指针 top byte 的 tag 和保存在 shadow memory 中的 tag 是否一致时，需要考虑如下两种可能：

- 保存在指针 top byte 的 tag 和保存在 shadow memory 中的 tag 相同。
    
- 保存在 shadow memory 中的 tag 实际上是 short granule size，保存在指针 top byte 的 tag 等于保存在指针指向的内存所在的 16-bytes 内存的最后一个字节的 tag。
    

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**为什么需要 short granules ？**

考虑 uint8_t buf[20]，假设代码中存在访问 buf[22] 导致的 buffer-overflow。因为 HWASAN 会将 heap/stack/global 内存对象对齐到 16-bytes，所以实际为 uint8_t buf[20] 申请的空间是 uint8_t buf[32]。

  

如果没有 short granules，那么保存在 buf 指针 top byte 的 tag 为 0xa1，保存在 buf[22] 对应的 shadow memory 中的 tag 为 0xa1，尽管访问 buf[22] 时发生了 buffer-overflow，此时 HWASAN 也检测不到，因为保存在指针 top byte 的 tag 和保存在 shadow memory 中的 tag 是否一致的。

  

有了 short granules，保存在 buf 指针 top byte 的 tag 为 0xa1，保存在 buf[22] 对应的 shadow memory 中的 tag 则是 short granule size 即 20 % 16 = 4。访问 buf[22] 时，HWASAN 发现保存在指针 top byte 的 tag 和保存在 shadow memory 中的 tag 不一致，保存在 buf[22] 对应的 shadow memory 中的 tag 是 short granule size 为 4，这意味着 buf[22] 所在的 16-bytes 内存只有前 4-bytes 是可以合法访问的，而 buf[22] 访问的却是其所在 16-bytes 内存的第 7 个 byte，说明访问 buf[22] 时发生了 buffer-overflow！

  

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**hwasan check memaccess**

本节说明 HWASAN 如何在每一处内存读写之前通过插桩来比较保存在指针 top byte 的 tag 和保存在 shadow memory 中的 tag 是否一致的。

  

开启 HWASAN 后，HWAddressSanitizer instrumentation pass 会在 LLVM IR 层面进行插桩。默认情况下，HWASAN 在每一处内存读写之前添加对 llvm.hwasan.check.memaccess.shortgranules intrinsic 的调用。该 llvm.hwasan.check.memaccess.shortgranules intrinsic 会在生成汇编代码时转换为对相应函数的调用。

  

还是以如下代码为例说明：

```
// cat test.c
```

上述代码 clang -O1 -fsanitize=hwaddress test.c -S -emit-llvm 开启 HWASAN 生成的 LLVM IR 如下：

```
%5 = alloca ptr, align 8
```

llvm.hwasan.check.memaccess.shortgranules intrinsic 有三个参数：

1. __hwasan_shadow_memory_dynamic_address
    
2. 本次 memory access 访问的内存地址
    
3. 常数 AccessInfo，编码了本次 memory access 的相关信息。计算公式如下：
    

```
int64_t HWAddressSanitizer::getAccessInfo(bool IsWrite,
```

IsWrite 布尔值，0 表示本次 memory access 是读操作，1 表示本次 memory access 为 写操作。

  

AccessSizeIndex 由 __builtin_ctz(AccessSize) 计算得到。AccessSize 为 1-byte, 2-bytes, 4-bytes, 8-bytes, 16-bytes 时，AccessSizeIndex 分别为 0, 1, 2, 3, 4。

  

CompileKernel 布尔值，只有 HWASAN 用于 KernelHWAddressSanitizer 时 CompileKernel 值才为 1。

  

MatchAllTag 类型为 uint8_t，MatchAllTag 的默认值为 -1，表示没有设置 MatchAllTag。可以通过编译时选项 -mllvm -hwasan-match-all-tag=-1 进行设置。当保存在指针 top byte 的 tag 值 和保存在 shadow memory 中的 tag 不匹配时说明 HWASAN 检测到的内存错误，如果设置了 MatchAllTag，那么 HWASAN 会忽略所有 pointer tag 为 MatchAllTag 时出现的 tag mismatch。

  

Recover 布尔值，0 表示 HWASAN 检测到错误不再继续执行程序，1 表示 HWASAN 检测到错误后继续执行。Recover 默认为 0，在编译时添加参数 -fsanitize-recover=hwaddress 后 Recover 为 1。

  

上述例子 LLVM IR 中，调用 llvm.hwasan.check.memaccess.shortgranules 的第一参数就是 %__hwasan_shadow_memory_dynamic_address，第二个参数是 %8 对应源码中的 x[10] 的地址，第三个参数是常数 18 表示本次内存访问是写入 4-byte 的操作。

  

llvm.hwasan.check.memaccess.shortgranules intrinsic 在 AsmPrinter 阶段被转换为汇编代码。对于 AArch64 后端，相关的函数为 LowerHWASAN_CHECK_MEMACCESS(), emitHwasanMemaccessSymbols()，代码位于 llvm/lib/Target/AArch64/AArch64AsmPrinter.cpp。

  

llvm.hwasan.check.memaccess.shortgranules intrinsic 会根据参数不同而生成不同的汇编函数。例如上述例子中 call void @llvm.hwasan.check.memaccess.shortgranules(ptr %__hwasan_shadow_memory_dynamic_address, ptr %8, i32 18) 生成的汇编符号/函数名为 __hwasan_check_x0_18_short_v2：

- x0 表示本次 memory access 访问的内存地址保存在 X0 寄存器中
    
- 18 即本次 memory access 的 AccessInfo 的值
    

另外，AArch64AsmPrinter 在将 llvm.hwasan.check.memaccess.shortgranules intrinsic 转换为汇编代码时，总是将 __hwasan_shadow_memory_dynamic_address 即 shadow base 保存在 X20 寄存器中。

  

__hwasan_check_x0_18_short_v2 完整的汇编代码如下：

```
__hwasan_check_x0_18_short_v2:
```

前面内容提到，HWASAN 在编译插桩时，默认情况下是在每一处内存读写之前添加对 llvm.hwasan.check.memaccess.shortgranules intrinsic 的调用，那非默认情况呢？

- 如果在编译时添加选项 -mllvm -hwasan-instrument-with-calls=true，那么编译插桩时 HWASAN 在每一处内存读写之前添加的则是对 __hwasan_[load|store][1|2|4|8|16|n] 的函数调用。
    
- 还是以本文开头的示例代码进行说明，clang -O1 -fsanitize=hwaddress -mllvm -hwasan-instrument-with-calls=true test.c -S -emit-llvm 生成的 LLVM IR 如下：
    

```
  %2 = alloca ptr, align 8
```

- __hwasan_[load|store][1|2|4|8|16|n] 函数由 HWASAN runtime 中实现，代码位于 compiler-rt/lib/hwasan/hwasan.cpp。根据本次 memory access 为读操作或写操作，调用 ___hwasan_load 或 ___hwasan_store；根据本次 memory access 读写的内存大小（字节），调用相应的版本，如本次 memory acesss 是对 4-bytes 内存的写操作，HWASAN 插入的是就对 ___hwasan_store4 的调用。
    
- __hwasan_[load|store][1|2|4|8|16|n] 函数的实现几乎一致，下面给出 ___hwasan_store4 的代码实现：
    

```
void __hwasan_store4(uptr p) {
```

- 如果在编译时添加选项 -mllvm -hwasan-inline-all-checks=true，那么编译插桩时 HWASAN 会直接在 LLVM IR 上实现检查保存在指针 top byte 的 tag 和保存在 shadow memory 中的 tag 是否匹配的逻辑。
    
- 还是以本文开头的示例代码进行说明，clang -O1 -fsanitize=hwaddress -mllvm -hwasan-inline-all-checks=true test.c -S -emit-llvm 生成的 LLVM IR 如下：
    

```
  %5 = alloca ptr, align 8
```

**开源社区贡献**

在实践 HWASAN 过程中，我们对 HWASAN 进行了一些改进和增强：修复了 HWASAN 生成报告时存在的多线程数据竞争问题，增强了多线程内存错误报告的可读性、修复使用 match-all-tag 时会产生误报的问题、并且扩展了 match-all-tag 的使用场景等。目前字节跳动 STE 团队已将相关补丁提交至 LLVM 社区，并合入社区主线，链接如下：

- https://reviews.llvm.org/D147215
    
- https://reviews.llvm.org/D147121
    
- https://reviews.llvm.org/D149252
    
- https://reviews.llvm.org/D148909
    
- https://reviews.llvm.org/D149580
    
- https://reviews.llvm.org/D149943
    

LLVM 作为一套提供编译器基础设施的开源项目，包含一系列模块化的编译器组件和工具链，欢迎更多对 LLVM 感兴趣的同学一起交流，共同推进社区的建设。

  

**参考链接**

- Hardware-assisted AddressSanitizer Design Documentation:
    
    https://arxiv.org/pdf/1802.09517.pdf
    
- https://github.com/google/sanitizers/tree/master/hwaddress-sanitizer
    

  

**热门招聘**

金三银四的季节，字节跳动STE团队诚邀您的加入！团队长期招聘，**北京、上海、深圳、杭州、US、UK** 均设岗位，以下为近期的招聘职位信息，有意向者可直接扫描海报二维码投递简历，期待与你早日相遇，在字节共赴星辰大海！若有问题可咨询小助手微信：sys_tech，岗位多多，快来砸简历吧！

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**往期精彩**

# [深度解析C/C++之Strict Aliasing Rule](http://mp.weixin.qq.com/s?__biz=Mzg3Mjg2NjU4NA==&mid=2247484517&idx=1&sn=049fb8f6743e4caf6f9138688cd841a1&chksm=cee9f212f99e7b043523bfe80d8cb6ea1a86a5c6d2a2a14a0aa5944fbb1e116190d594ad8156&scene=21#wechat_redirect)

[深入浅出 Sanitizer Interceptor 机制](http://mp.weixin.qq.com/s?__biz=Mzg3Mjg2NjU4NA==&mid=2247483868&idx=1&sn=a85112e88cd187e27f418ff044247956&chksm=cee9f7abf99e7ebdac55e36077b4f915bccbe33a8a6ee9920136a1dc9598e2ae62bb95d3a25f&scene=21#wechat_redirect)

[Jemalloc内存分配与优化实践](http://mp.weixin.qq.com/s?__biz=Mzg3Mjg2NjU4NA==&mid=2247484507&idx=1&sn=0befa2bec46c3fe64807f5b79b2940d5&chksm=cee9f22cf99e7b3a6235b06cb0bd87c3f8e9b10c3ba989b253a97a5dc92b261cfde0b39db230&scene=21#wechat_redirect)

[DPDK内存碎片优化，性能最高提升30+倍!](http://mp.weixin.qq.com/s?__biz=Mzg3Mjg2NjU4NA==&mid=2247484462&idx=1&sn=406c59905ea57718018c602cba66f40e&chksm=cee9f259f99e7b4f7597eb6754367214a28f120c062c72c827f6e8c4225f9e54e4c65d4395d8&scene=21#wechat_redirect)

[万字长文，GPU Render Engine 详细介绍](http://mp.weixin.qq.com/s?__biz=Mzg3Mjg2NjU4NA==&mid=2247484413&idx=1&sn=d8f5f4f195b931d775b7d5934f0e3cb4&chksm=cee9f58af99e7c9c07080f22165120fcf3d321a14b943077c23649f88b27da29529db44de933&scene=21#wechat_redirect)

[DPDK Graph Pipeline框架简介与实现原理](http://mp.weixin.qq.com/s?__biz=Mzg3Mjg2NjU4NA==&mid=2247483974&idx=1&sn=25a198745ee4bec9755fe1f64f500dd8&chksm=cee9f431f99e7d27527c7a74c0fd2969c00cf904cd5a25d09dd9e2ba1adb1e2f1a3ced267940&scene=21#wechat_redirect)

[字节跳动算力监控系统的落地与实践](http://mp.weixin.qq.com/s?__biz=Mzg3Mjg2NjU4NA==&mid=2247484264&idx=1&sn=c27c08c225fcceb65a5ffc071dba3f95&chksm=cee9f51ff99e7c09a0c103bc4b362fcdcba7d8e6de74ab251afe1beb5b7af0412e619dd53f13&scene=21#wechat_redirect)

[JVM 内存使用大起底](http://mp.weixin.qq.com/s?__biz=Mzg3Mjg2NjU4NA==&mid=2247484211&idx=1&sn=c65b23b2ab50f6d56d81cfe434c8ea53&chksm=cee9f544f99e7c52d7767db2a74fda23ed7a6c06f8c7a82c1219a7e1474839bf5301f78ff38e&scene=21#wechat_redirect)

[字节跳动在PGO反馈优化技术上的探索与实践](http://mp.weixin.qq.com/s?__biz=Mzg3Mjg2NjU4NA==&mid=2247483904&idx=1&sn=335efeed20a6b42d5ee6e6543ae12018&chksm=cee9f477f99e7d6141de97d85c125979daaae14d98e73225792fa8e41cf0d6269e97d5c7a397&scene=21#wechat_redirect)

[内核优化之 PSI 篇:快速发现性能瓶颈，提高资源利用率](http://mp.weixin.qq.com/s?__biz=Mzg3Mjg2NjU4NA==&mid=2247483887&idx=1&sn=32ec7c9200a72f13ef2c1b8756ea2b8f&chksm=cee9f798f99e7e8e6e44a4f7464ee18471cd4e12def97fdf207f58d37dd317e3fafa6614e04f&scene=21#wechat_redirect)

[探索Mac mini M1的QEMU/KVM虚拟化实现](http://mp.weixin.qq.com/s?__biz=Mzg3Mjg2NjU4NA==&mid=2247483881&idx=1&sn=282483df28129da2d193239743397260&chksm=cee9f79ef99e7e884ca202597076df9915145a6c36670ae3bab2419a0da13deed072ee59e058&scene=21#wechat_redirect)

[字节跳动提出KVM内核热升级方案，效率提升5.25倍！](http://mp.weixin.qq.com/s?__biz=Mzg3Mjg2NjU4NA==&mid=2247483751&idx=1&sn=6f818349a0d6579902d9a7ca1a512cca&chksm=cee9f710f99e7e06957baac9ce25433ed8a463a417479f243136cef254073f10e018d941be4b&scene=21#wechat_redirect)

[安全容器文件服务 Virtio-fs 的高可用实践](http://mp.weixin.qq.com/s?__biz=Mzg3Mjg2NjU4NA==&mid=2247483815&idx=1&sn=92edbb33cbbbf22c80ccdf6537a2ca37&chksm=cee9f7d0f99e7ec671106416c7d5cee58497af8f427a89f5e9a2ce051277944116db069d10e3&scene=21#wechat_redirect)

[字节跳动开源高性能C++ JSON库sonic-cpp](http://mp.weixin.qq.com/s?__biz=Mzg3Mjg2NjU4NA==&mid=2247483853&idx=1&sn=b88e45bf75cd57482cf7860e669416a8&chksm=cee9f7baf99e7eacb5de6f9118a590db4bb659420bc2f4cb6c6964225409f559715ea8145186&scene=21#wechat_redirect)

[eBPF 技术实践：加速容器网络转发，耗时降低60%+](http://mp.weixin.qq.com/s?__biz=Mzg3Mjg2NjU4NA==&mid=2247483701&idx=1&sn=c99b7df5b8457f6d4f87fa18fea82eba&chksm=cee9f742f99e7e54f317d4f9eb355daf8cdceb4d3894e7601f5fc124e8b3b8567ee5f2efc46a&scene=21#wechat_redirect)

[十年磨一剑，veLinux深度解读](http://mp.weixin.qq.com/s?__biz=Mzg3Mjg2NjU4NA==&mid=2247483680&idx=1&sn=d767d17e8b28baacb25f759c02e7bd0c&chksm=cee9f757f99e7e4136580232d4c56d619feacdd939e84ed9780d010b73417c5739b223a5c4dd&scene=21#wechat_redirect)

  

**关于STE团队**

**字节跳动STE团队**（System Technologies&Engineering，系统技术与工程），一直致力于操作系统内核与虚拟化、系统基础软件与基础库的构建和性能优化、超大规模数据中心的系统稳定性和可靠性建设、新硬件与软件的协同设计等基础技术领域的研发与工程化落地，具备全面的基础软件工程能力，为字节上层业务保驾护航。同时，团队积极关注社区技术动向，拥抱开源和标准，欢迎更多同学加入我们，一起交流学习。扫描下方二维码了解职位详情，欢迎大家投递简历至huangxuechun.hr@bytedance.com 、wangan.hr@bytedance.com。

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

扫码查看职位详情

Reads 2156

​

Send Message

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/Z4txjviceemNaH5Gc7V2ltc9AB79kQxpM7pOjtTiagt1TlQtCTrJ6zyPma8xbw3sGpeecTwveKP4GLqQa2nJ49EA/300?wx_fmt=png&wxfrom=18)

字节跳动SYS Tech

5171

Send Message