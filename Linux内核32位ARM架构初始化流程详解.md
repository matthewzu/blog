由于受到众多硬件厂商及软件巨头的支持，以及自身学术性和应用性的完美结合，Linux操作系统在各个方面尤其是嵌入式领域得到了广泛应用。同时，作为开源软件，市面上虽然已经有不少介绍Linux内核启动流程的文章和书籍，但往往都过于简陋，流于表面，或者过于分散，无法得到一个整体的印象。本文基于ARM FVP模拟器平台ARMv7 Cortext-A9处理器，专注于Linux 4.10内核启动流程中的内核初始化环节，对整个过程中涉及到的ARM和Linux内核的基本知识进行介绍，使得用户可以对整个启动流程有一个整体的概念，从而有助于解决启动过程中遇到的各种问题。

**关键词**：嵌入式系统,虚拟化,Linux内核,ARM,32位，ARMv7,Cortext-A9；内核初始化

# 1. 介绍

## 1.1 基本环境

本文使用的调试环境如下：
1. 调试器采用ARM公司的DS-5 V5.27.1；
2. 模拟器采用DS-5继承的ARM FVP（Fixed Virtual Platform，固定虚拟平台）；
3. 模拟器目标板采用ARM vexpress，处理器为Cortext-A9 4核处理器；
4. Linux内核版本为4.10；
5. 配置文件采用vexpress_defconfig；
6. 内核格式为非压缩的Image；
7. DTB文件采用DS-5自带的rtsm_ve-cortex_a9x4.dtb。

**注意**：
* 本文读者应具备基础的ARM架构指令集知识（工作模式、寄存器组织、寻址模式等）和C语言常识；
* 本文关注的是32位ARMv7架构启动流程，无关代码以`...`表示从而方便阅读
* DS-5和ARM FVP的使用方法可参见[ARM FVP(固定虚拟平台)Linux内核调试简明手册](http://www.jianshu.com/p/c0a9a4b9569d);
* Linux内核代码可参考[Linux源码主线](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/log/?h=v4.14-rc2)；
* ARM体系结构可参考官网[ARMv7-AR Reference Manual](https://silver.arm.com/download/ARM_and_AMBA_Architecture/AR570-DA-70000-r0p0-00rel3/DDI0406C_d_armv7ar_arm.pdf)
* ARM指令可参考ARM官网[ARM and Thumb-2 Instruction Set Quick Reference Card (Chinese)](http://infocenter.arm.com/help/topic/com.arm.doc.qrc0001mc/QRC0001_UAL.pdf)和[ARMv7-AR Reference Manual](https://silver.arm.com/download/ARM_and_AMBA_Architecture/AR570-DA-70000-r0p0-00rel3/DDI0406C_d_armv7ar_arm.pdf)附录A4 - A8；
* ARM 32位架构ABI可参考ARM官网[针对ARM体系结构的应用程序二进制接口(ABI)](http://infocenter.arm.com/help/topic/com.arm.doc.ihi0042f/IHI0042F_aapcs.pdf)；
* GNU链接器（ld）和汇编器（as）手册可参考[GNU二进制工具文档](https://sourceware.org/binutils/docs/)。

## 1.2 初始化阶段划分

Linux内核初始化划分为如下几个阶段：
1. 入口阶段，即内核入口到C语言入口start_kernel()之前的阶段，基本上使用汇编语言实现；
2. 早期初始化阶段，即start_kernel()过程，用于为kernel_init线程（PID为1）和线程调度准备最基本的运行环境;
3. 初始化阶段，即kernel_init线程的执行过程，用于进行真正的内核初始化，包括驱动、文件系统、网络协议栈等模块的真正初始化，并最终执行跟文件系统中的初始化脚本；
4. 用户空间初始化，即初始化脚本定义的用户程序执行和模块加载等，不属于本文讨论范围。

## 1.3 内核启动条件和参数

Linux内核被加载时，根据处理器架构的不同都需要遵循一定的启动条件并传入合适的参数。这些参数和要求可以从每个ARCH的文档中获得。
对于32位ARM架构，基本要求可以从Documentation\arm\Booting中获取。简单来说，跳转到内核入口之前，必须遵顼如下条件：
1. MMU和数据Cache必须处于禁止状态；
2. 指令Cache使能或禁止均可；
3. 寄存器r0必须为0；
4. 寄存器r1为可选的machine编号(定义在arch/arm/tools/mach-types中)；
5. 寄存器r2为标签化内核参数列表（atags）或二进制设备树（DTB）在系统内存中的物理地址。

**注意**：
* 寄存器r1和r2的值会第一时间被保存在特殊的寄存器中以防止被破坏，不管是压缩还是非压缩版本；
* 内核通过对比寄存器r2对应地址的内容与DTB魔数或标签化内核参数列表标志（ATAG_CORE）来确定使用何种方式传递内核参数。

# 2.入口阶段

该阶段的主要任务是：
1. 可选的内核解压缩
2. 获取处理器信息;
3. 获取内核参数；
4. 进行基本的CPU初始化；
5. 准备C语言运行环境。

32位ARM架构包含2个内核初始化入口：
* arch\arm\boot\compressed\head.S包含压缩版本内核初始化入口start();
* arch\arm\kernel\head.S包含CPU初始化入口stext()。

**注意**：压缩版本内核在完成内核解压后，仍然会跳转到真正的内核初始化入口stext()。

## 2.1 入口部分

此段代码**为后续初始化准备基本环境，即进入SVC模式并关闭中断**。

### 2.1.1 主体

本段代码定义在`arch\arm\kernel\head.S`中，作为**CPU初始化入口**。

```cpp
    __HEAD
ENTRY(stext)
    ...
    safe_svcmode_maskall r9
    ...
```
逐条解释如下：
1. __HEAD
    定义在include/linux/init.h中：
    ```
    #define __HEAD  .section    ".head.text","ax"
    ```
    根据[GNU汇编器（as）手册.section语法](https://sourceware.org/binutils/docs/as/Section.html#Section)，此处指定当前代码被链接到`.head.text`段，`a`被忽略，`x`为可执行段标志。该代码段定义在`arch/arm/kernel/vmlinux.lds.S`中：
    ```
    . = PAGE_OFFSET + TEXT_OFFSET;
    .head.text : {
        _text = .;
        HEAD_TEXT
    }
    ```
    其中， `HEAD_TEXT`定义在`include/asm-generic/vmlinux.lds.h`中，
    ```
    #define HEAD_TEXT  *(.head.text)
    ```
    根据[GNU链接器（ld）手册section语法](https://sourceware.org/binutils/docs/ld/SECTIONS.html#SECTIONS)，上述链接脚本定义了一个位于**PAGE_OFFSET + TEXT_OFFSET**即**0x80008000**的输出段，用于存放`.head.text`输入端中的代码。其中：
    * 根据`arch/arm/include/asm/memory.h`， **PAGE_OFFSET**定义为CONFIG_PAGE_OFFSET，即由`arch/arm/Kconfig`的PAGE_OFFSET配置项来决定：
        ```
        config PAGE_OFFSET
            hex
            default PHYS_OFFSET if !MMU
            default 0x40000000 if VMSPLIT_1G
            default 0x80000000 if VMSPLIT_2G
            default 0xB0000000 if VMSPLIT_3G_OPT
            default 0xC0000000
        ```
        本文采用的虚拟地址空间配置为内核空间和用户空间各2G字节的配置方法，因此此处为**0x80000000**。
    * **TEXT_OFFSET**定义在`arch/arm/Makefile`中：
        ```
        textofs-y := 0x00008000
        ...
        textofs-$(CONFIG_ARCH_AXXIA) := 0x00308000
        ...
        # The byte offset of the kernel image in RAM from the start of RAM.
        TEXT_OFFSET := $(textofs-y)
        ```
        因此此处值为**0x00008000**。
2. ENTRY(stext)
    **ENTRY**定义在`include/linux/linkage.h`中，用于定义一个所有源文件可见的全局符号`stext`：
    ```
    ...
    #include <asm/linkage.h>
    ...
    /* Some toolchains use other characters (e.g. '`') to mark new line in macro */
    #ifndef ASM_NL
    #define ASM_NL         ;
    #endif
    ...
    #ifndef __ALIGN
    #define __ALIGN        .align 4,0x90
    #define __ALIGN_STR    ".align 4,0x90"
    #endif

    #ifdef __ASSEMBLY__

    #ifndef LINKER_SCRIPT
    #define ALIGN __ALIGN
    #define ALIGN_STR __ALIGN_STR

    #ifndef ENTRY
    #define ENTRY(name) \
        .globl name ASM_NL \
        ALIGN ASM_NL \
        name:
    #endif
    #endif /* LINKER_SCRIPT */
    ```
    其中，__ALIGN使用ARM架构（`arch/arm/include/asm/linkage.h`）的定义：
    ```
    #define __ALIGN .align 0
    #define __ALIGN_STR ".align 0"
    ```
    根据[GNU汇编器（as）手册.globl语法](https://sourceware.org/binutils/docs/as/Global.html#Global)和[GNU汇编器（as）手册.align语法](https://sourceware.org/binutils/docs/as/Align.html#Align)，此处定义`stext`标号，将其声明为全局符号，不做任何对齐。
3. safe_svcmode_maskall r9
    进入SVC模式，并关闭中断

### 2.1.2 safe_svcmode_maskall

本函数定义在`arch/arm/include/asm/assembler.h`中，用于**使CPU进入SVC模式并关闭中断**。

```cpp
.macro safe_svcmode_maskall reg:req
#if __LINUX_ARM_ARCH__ >= 6 && !defined(CONFIG_CPU_V7M)
    mrs     \reg , cpsr
    eor     \reg, \reg, #HYP_MODE
    tst     \reg, #MODE_MASK
    bic     \reg , \reg , #MODE_MASK
    orr     \reg , \reg , #PSR_I_BIT | PSR_F_BIT | SVC_MODE
    ...
    bne     1f
    orr     \reg, \reg, #PSR_A_BIT
    badr    lr, 2f
    msr     spsr_cxsf, \reg
   __MSR_ELR_HYP(14)
   __ERET
1:   msr    cpsr_c, \reg
2:
#else
...
#endif
.endm
```
逐条解释如下：
1.  .macro safe_svcmode_maskall reg:req
根据[GNU汇编器（as）手册.macro语法](https://sourceware.org/binutils/docs/as/Macro.html#Macro)，本行定义了宏safe_svcmode_maskall，参数`reg`必须传递一个有效值
2.  #if __LINUX_ARM_ARCH__ >= 6 && !defined(CONFIG_CPU_V7M)
    * `__LINUX_ARM_ARCH__`由`arch/arm/Makefile`根据Kconfig选项来指定，传递顺序如下：
        > CONFIG_ARCH_VEXPRESS(vexpress_defconfig) -> ARCH_VEXPRESS(arch/arm/mach-vexpress/Kconfig) -> ARCH_MULTI_V7(arch/arm/Kconfig) -> CPU_V7(arch/arm/mm/Kconfig) -> CONFIG_CPU_32v7(arch/arm/Makefile) -> -D__LINUX_ARM_ARCH__=7(arch/arm/Makefile)
    * 根据`Documentation/kbuild/kconfig.txt`，环境变量`CONFIG_`没有指定时，默认使用`CONFIG_`作为Kconfig配置项的前缀，例如Makefile中指定`CONFIG_ = XYZ_`，那么Kconfig配置项`CPU_32v7`就会被转换成`XYZ_CPU_32v7`并保存到`include/generated/autoconf.h`和`include/config/auto.conf`中，从而被源文件和Makefile使用。
    * `arch/arm/Makefile`中相关代码如下：
        ```
        arch-$(CONFIG_CPU_32v7) =-D__LINUX_ARM_ARCH__=7 ...
        ...
        KBUILD_AFLAGS +=$(CFLAGS_ABI) $(AFLAGS_ISA) $(arch-y) $(tune-y) -include asm/unified.h -msoft-float
        ```
    * `CONFIG_CPU_V7M`用于`Cortex V7-M`系列处理器，没有定义。
3.  mrs     \reg , cpsr
    从CPSR（当前处理器状态寄存器）读取处理器状态到`reg`对应寄存器
4.  eor     \reg, \reg, #HYP_MODE
    `HYP_MODE`即`Hypervisor模式0x1a`，定义在`arch/arm/include/uapi/asm/ptrace.h`中;本条指令将处理器状态与`0x1a`进行`按位异或`操作，按照`相同输出零，不同输出一`的规则将结果按位存储在`reg`中；因此，如果处理器工作在`Hypervisor模式`，也就是说模式域的值为`0x1a`，那么`reg`值的模式域的值就是0，否则不为0，输出结果的其他域根据具体情况而定。
5.  tst     \reg, #MODE_MASK
    `MODE_MASK`即`处理器模式掩码0x1f`，定义在`arch/arm/include/uapi/asm/ptrace.h`中；本条指令将上一条指令的结果与`0x1f`执行`按位与`操作，按照`相同输出一，不同输出零`的规则将结果按位存储在`reg`中，同时更新条件标志；结合上一条指令，如果处理器工作在`Hypervisor模式`，则`reg`值为0，否则`reg`值为1。
6.  bic     \reg , \reg , #MODE_MASK

7.  orr     \reg , \reg , #PSR_I_BIT | PSR_F_BIT | SVC_MODE
8.  bne     1f
9.  orr     \reg, \reg, #PSR_A_BIT
10. badr    lr, 2f
11. msr     spsr_cxsf, \reg
12. _MSR_ELR_HYP(14)
13. __ERET
14. msr    cpsr_c, \reg


## 2.2 获取处理器信息

```cpp
    __HEAD
ENTRY(stext)
...
    @ ensure svc mode and all interrupts masked
    safe_svcmode_maskall r9

    mrc     p15, 0, r9, c0, c0      @ get processor id
    bl      __lookup_processor_type @ r5=procinfo r9=cpuid
    movs    r10, r5                 @ invalid processor (r5=0)?
    ...
    beq    __error_p                @ yes, error 'p'
    ...
```
逐条解释如下：
1. mrc     p15, 0, r9, c0, c0
    根据[ARMv7-AR Reference Manual](https://silver.arm.com/download/ARM_and_AMBA_Architecture/AR570-DA-70000-r0p0-00rel3/DDI0406C_d_armv7ar_arm.pdf)附录`A8.8.108 `，本条指令用于将协处理器寄存器的内容读取到通用目的寄存器中，并忽略了opc2，其完整格式应当是**mrc p15, 0, r9, c0, c0, 0**，其中coproc、CRn、opc1、 CRm、opc2分别为15、c0、0、c0、0；因此根据[ARMv7-AR Reference Manual](https://silver.arm.com/download/ARM_and_AMBA_Architecture/AR570-DA-70000-r0p0-00rel3/DDI0406C_d_armv7ar_arm.pdf)表`B3-42`，本指令用于访问**MIDR（Main ID Register）**；MIDR定义在[ARMv7-AR Reference Manual](https://silver.arm.com/download/ARM_and_AMBA_Architecture/AR570-DA-70000-r0p0-00rel3/DDI0406C_d_armv7ar_arm.pdf)附录`B4.1.105`，包含实现者ID（ARM、高通等）、架构类型（V4、V5等）、变种、编号、版本等信息。
2. bl      __lookup_processor_type
   根据MIDR内容从处理器信息表中查找相应的处理器；该函数定义在`arch/arm/kernel/head-common.S`中，后面会给出详细解释
3. beq    __error_p
   如果没有查找到相应的处理器信息，则跳转到`__error_p`进行错误处理

## 2.2 可选的内核解压缩过程

此段代码**计算内核映像物理地址（PHYS_OFFSET）并存储到r8中**。

```cpp
#ifndef CONFIG_XIP_KERNEL
    adr    r3, 2f
    ldmia  r3, {r4, r8}
    sub    r4, r3, r4            @ (PHYS_OFFSET - PAGE_OFFSET)
    add    r8, r8, r4            @ PHYS_OFFSET
#else
    ...
#endif
...
#ifndef CONFIG_XIP_KERNEL
2:  .long    .
    .long    PAGE_OFFSET
#endif
```
逐条解释如下：
1. adr    r3, 2f => ADR r3,{pc}+0x40
将当前指令的地址与0x40的和赋给r3,即获取标号2物理地址（运行时地址），文中为0x80008054

2. ldmia    r3, {r4, r8}
将标号2的虚地址（链接地址）和PAGE_OFFSET变量的虚地址加载到r4和r8

3. sub    r4, r3, r4            @ (PHYS_OFFSET - PAGE_OFFSET)
获取标号2虚地址和物理地址的差值

4. add    r8, r8, r4            @ PHYS_OFFSET
PAGE_OFFSET + （标号2物理地址 - 标号2虚地址） =
（PAGE_OFFSET虚地址 - 标号2虚地址） + 标号2物理地址 = PAGE_OFFSET对应的物理地址 = PHYS_OFFSET

# 2. __fixup_pv_table

此段代码基于PHYS_OFFSET（存放于r8寄存器）和pv表（__pv_table_xxx）修正stub代码中add/sub指令。

## 2.1 准备pv表
```cpp
/* __fixup_pv_table - patch the stub instructions with the delta between
 * PHYS_OFFSET and PAGE_OFFSET, which is assumed to be 16MiB aligned and
 * can be expressed by an immediate shifter operand. The stub instruction
 * has a form of '(add|sub) rd, rn, #imm'.
 */
    __HEAD
__fixup_pv_table:
    adr    r0, 1f
    ldmia    r0, {r3-r7}
    mvn    ip, #0
    subs    r3, r0, r3    @ PHYS_OFFSET - PAGE_OFFSET
    add    r4, r4, r3    @ adjust table start address
    add    r5, r5, r3    @ adjust table end address
    add    r6, r6, r3    @ adjust __pv_phys_pfn_offset address
    add    r7, r7, r3    @ adjust __pv_offset address
    mov    r0, r8, lsr #PAGE_SHIFT    @ convert to PFN
    str    r0, [r6]    @ save computed PHYS_OFFSET to __pv_phys_pfn_offset
    strcc    ip, [r7, #HIGH_OFFSET]    @ save to __pv_offset high bits
    mov    r6, r3, lsr #24    @ constant for add/sub instructions
    teq    r3, r6, lsl #24 @ must be 16MiB aligned
    ...
    bne    __error
    str    r3, [r7, #LOW_OFFSET]    @ save to __pv_offset low bits
    b    __fixup_a_pv_table
ENDPROC(__fixup_pv_table)

    .align
1:  .long    .
    .long    __pv_table_begin
    .long    __pv_table_end
2:  .long    __pv_phys_pfn_offset
    .long    __pv_offset
```
逐条解释如下：
1. adr    r0, 1f
获取标号1的物理地址（运行时地址）

2. ldmia    r0, {r3-r7}
加载标号1、__pv_table_begin、__pv_table_end、__pv_phys_pfn_offset和__pv_offset的虚地址（链接地址）

1. mvn    ip, #0
0取反即0xFFFFFFFF放入r12

    **注意**，根据[Procedure Call Standard for the ARM Architecture](http://infocenter.arm.com/help/index.jsp?topic=/com.arm.doc.subset.swdev.abi/index.html)的`5.1.1章节`，**r12寄存器用作IP寄存器**（过程内调用临时寄存器）。

4. subs    r3, r0, r3
标号1的运行时地址减去标号1的链接地址得到物理地址和虚地址的差值，同时更新标志位（类似于CMP）

    **注意**，`s`后缀用于更新状态寄存器中的标志位，以便**第12条指令**使用。

5. add    r4, r4, r3
获取__pv_table_begin的物理地址（运行时地址）

6. add    r5, r5, r3
获取___pv_table_end的物理地址（运行时地址）

7. add    r6, r6, r3
获取__pv_phys_pfn_offset的物理地址（运行时地址）

8. add    r7, r7, r3
获取__pv_offset的物理地址（运行时地址）

9. mov    r0, r8, lsr #PAGE_SHIFT
PHYS_OFFSET即PAGE_OIFFSET（映像基地址）的物理地址右移PAGE_SHIFT即12位，相当于除以页表大小4K字节(0x1000)，转换成物理页号

10. str    r0, [r6]
PHYS_OFFSET的物理页号存入__pv_phys_pfn_offset的物理地址

11. strcc    ip, [r7, #HIGH_OFFSET]
如果标号1的物理地址（运行时地址）小于虚地址（链接地址），则保存0xFFFFFFFF到__pv_offset的高32位，即将__pv_offset设置为负数

    **注意**：
    * `cc`后缀用于根据前面**第4条指令**的状态寄存器更新结果，含义是`小于`
    * 负数在计算机中用补码表示，即`取反加一`，例如`-2`的64位16进制表示为`0x0xFFFF FFFF FFFF FFFE`;
    * 。

12. mov    r6, r3, lsr #24
物理地址和虚地址的差值右移24位即除以16M字节（0x100 0000），放入r6

13. teq    r3, r6, lsl #24
检查r3与r6左移24位的值是否相等以确认物理地址和虚地址的差值对齐到16M字节，结果存入状态寄存器

14. bne    __error
不对齐则跳转到__error

15. str    r3, [r7, #LOW_OFFSET]
将物理地址和虚地址的差值保存到__pv_offset的低32位

16. b    __fixup_a_pv_table
跳转到__fixup_a_pv_table去逐条修正

## 2.2 修正表项

```cpp
    .text
__fixup_a_pv_table:
    adr    r0, 3f
    ldr    r6, [r0]
    add    r6, r6, r3
    ldr    r0, [r6, #HIGH_OFFSET]    @ pv_offset high word
    ldr    r6, [r6, #LOW_OFFSET]    @ pv_offset low word
    mov    r6, r6, lsr #24
    cmn    r0, #1
#ifdef CONFIG_THUMB2_KERNEL
    ...
#else
#ifdef CONFIG_CPU_ENDIAN_BE8
    ...
#else
    moveq    r0, #0x400000    @ set bit 22, mov to mvn instruction
#endif
    b    2f
1:  ldr    ip, [r7, r3]
#ifdef CONFIG_CPU_ENDIAN_BE8
    ...
#else
    bic    ip, ip, #0x000000ff
    tst    ip, #0xf00    @ check the rotation field
    orrne  ip, ip, r6    @ mask in offset bits 31-24
    biceq  ip, ip, #0x400000    @ clear bit 22
    orreq  ip, ip, r0    @ mask in offset bits 7-0
#endif
    str    ip, [r7, r3]
2:  cmp    r4, r5
    ldrcc  r7, [r4], #4  @ use branch for delay slot
    bcc    1b
    ret    lr
#endif
ENDPROC(__fixup_a_pv_table)

    .align
3:  .long __pv_offset
```
逐条解释如下：
1. adr    r0, 3f
加载__pv_offset的物理地址（运行时地址）

2. ldr    r6, [r0]
加载__pv_offset的链接地址到r6

3. add    r6, r6, r3
获取__pv_offset的物理地址

**注意**： r3为主体部分计算出来的物理地址和虚地址的差值

4. 1： ldr    r0, [r6, #HIGH_OFFSET]
获取__pv_offset的高32位

5. ldr    r6, [r6, #LOW_OFFSET]
获取__pv_offset的低32位

6. mov    r6, r6, lsr #24
__pv_offset的低32位右移24位，即除以16M

7. cmn    r0, #1
检查r0是否为0xFFFFFFFF,即-1，也就是标号1的运行时地址小于链接地址

8. moveq    r0, #0x400000
如果r0为0xFFFFFFFF，则赋值为0x400000，即MOV指令的R位，用于将MOV转换为MVN

9. b    2f
跳转到标号2

10. ldr    ip, [r7, r3]
将待修正指令的虚地址（链接地址）与r3相加，得到待修正指令的物理地址（运行时地址）,并从物理地址加载指令到`r12`

**注意**： r3为主体部分计算出来的物理地址和虚地址的差值

11. bic    ip, ip, #0x000000ff
清除指令7~0位，即立即数域

![](index_files/8710659.png)
![](index_files/10576898.png)
**注意**：
1. PV表中的数据处理指令均为采用上表中立即数模式的ADD/SUB指令；
2. PV表中的MOV指令为上表中的立即数移位指令，并且不需要移位。
3. PV表中没有其他指令;
4. PV表中的指令仅用于实现__virt_to_phys和__phys_to_virt地址转换函数，不能在其他代码中出现。

```cpp
#define __pv_stub(from,to,instr,type)            \
    __asm__("@ __pv_stub\n"                \
    "1:    " instr "    %0, %1, %2\n"        \
    "    .pushsection .pv_table,\"a\"\n"        \
    "    .long    1b\n"                \
    "    .popsection\n"                \
    : "=r" (to)                    \
    : "r" (from), "I" (type))

#define __pv_stub_mov_hi(t)                \
    __asm__ volatile("@ __pv_stub_mov\n"        \
    "1:    mov    %R0, %1\n"            \
    "    .pushsection .pv_table,\"a\"\n"        \
    "    .long    1b\n"                \
    "    .popsection\n"                \
    : "=r" (t)                    \
    : "I" (__PV_BITS_7_0))

#define __pv_add_carry_stub(x, y)            \
    __asm__ volatile("@ __pv_add_carry_stub\n"    \
    "1:    adds    %Q0, %1, %2\n"            \
    "    adc    %R0, %R0, #0\n"            \
    "    .pushsection .pv_table,\"a\"\n"        \
    "    .long    1b\n"                \
    "    .popsection\n"                \
    : "+r" (y)                    \
    : "r" (x), "I" (__PV_BITS_31_24)        \
    : "cc")

static inline phys_addr_t __virt_to_phys(unsigned long x)
{
    phys_addr_t t;

    if (sizeof(phys_addr_t) == 4) {
        __pv_stub(x, t, "add", __PV_BITS_31_24);
    } else {
        __pv_stub_mov_hi(t);
        __pv_add_carry_stub(x, t);
    }
    return t;
}

static inline unsigned long __phys_to_virt(phys_addr_t x)
{
    unsigned long t;

    /*
     * 'unsigned long' cast discard upper word when
     * phys_addr_t is 64 bit, and makes sure that inline
     * assembler expression receives 32 bit argument
     * in place where 'r' 32 bit operand is expected.
     */
    __pv_stub((unsigned long) x, t, "sub", __PV_BITS_31_24);
    return t;
}
```

12. tst    ip, #0xf00
检查循环移位域

13. orrne    ip, ip, r6
如果非0，则或上__pv_offset的31~24位到指令的7~0位

14. biceq    ip, ip, #0x400000
如果为0，即MOV指令，则清除22位

15. orreq    ip, ip, r0
如果为0，即MOV指令，则MOV指令的R位，用于将MOV转换为MVN

16. str    ip, [r7, r3]
保存修正后的指令

17. 2： cmp    r4, r5
比较当前PV表指针是否到达尾部___pv_table_end

18. ldrcc    r7, [r4], #4
如果没有到达尾部则从当前指针加载一个表项，即待修正指令的链接地址，同时前PV表指针前移一项

19. bcc    1b
如果没有到达尾部则跳转到标号1

20. ret    lr
返回

# 3. __create_page_tables

## 3.1 准备工作

```cpp
    /*
 * Setup the initial page tables.  We only setup the barest
 * amount which are required to get the kernel running, which
 * generally means mapping in the kernel code.
 *
 * r8 = phys_offset, r9 = cpuid, r10 = procinfo
 *
 * Returns:
 *  r0, r3, r5-r7 corrupted
 *  r4 = physical page table address
 */
__create_page_tables:
    pgtbl    r4, r8                @ page table address

    /*
     * Clear the swapper page table
     */
    mov    r0, r4
    mov    r3, #0
    add    r6, r0, #PG_DIR_SIZE
1:    str    r3, [r0], #4
    str    r3, [r0], #4
    str    r3, [r0], #4
    str    r3, [r0], #4
    teq    r0, r6
    bne    1b
...
    ldr    r7, [r10, #PROCINFO_MM_MMUFLAGS] @ mm_mmuflags
```
![](index_files/11743334.png)
1. pgtbl    r4, r8
```cpp
/*
 * swapper_pg_dir is the virtual address of the initial page table.
 * We place the page tables 16K below KERNEL_RAM_VADDR.  Therefore, we must
 * make sure that KERNEL_RAM_VADDR is correctly set.  Currently, we expect
 * the least significant 16 bits to be 0x8000, but we could probably
 * relax this restriction to KERNEL_RAM_VADDR >= PAGE_OFFSET + 0x4000.
 */
#define KERNEL_RAM_VADDR    (PAGE_OFFSET + TEXT_OFFSET)
#if (KERNEL_RAM_VADDR & 0xffff) != 0x8000
#error KERNEL_RAM_VADDR must start at 0xXXXX8000
#endif

#ifdef CONFIG_ARM_LPAE
    /* LPAE requires an additional page for the PGD */
#define PG_DIR_SIZE    0x5000
#define PMD_ORDER    3
#else
#define PG_DIR_SIZE    0x4000
#define PMD_ORDER    2
#endif

    .globl    swapper_pg_dir
    .equ    swapper_pg_dir, KERNEL_RAM_VADDR - PG_DIR_SIZE

    .macro    pgtbl, rd, phys
    add    \rd, \phys, #TEXT_OFFSET
    sub    \rd, \rd, #PG_DIR_SIZE
    .endm
```
PHYS_OFFSET加上TEXT_OFFSET，得到KERNEL_RAM_VADDR即内核入口地址的运行时地址，再减去PGD页表大小，从而得到页表基址运行时地址，存入`r4`

2. mov    r0, r4
页表基址运行时地址存入`r0`，作为指针

3. mov    r3, #0
清零r3

4. add    r6, r0, #PG_DIR_SIZE
保存PGD页表结束地址到`r6`

5. 1： str    r3, [r0], #4
清零4个字节，并递增指针

6. teq    r0, r6
检查指针是否到达页表结束

7. bne    1b
未结束则跳转到标号1

8. ldr    r7, [r10, #PROCINFO_MM_MMUFLAGS] @ mm_mmuflags
读取procinfo中的MMU标志到`r7`

## 3.2 为__turn_mmu_on函数创建临时映射

```cpp
    /*
     * Create identity mapping to cater for __enable_mmu.
     * This identity mapping will be removed by paging_init().
     */
    adr    r0, __turn_mmu_on_loc
    ldmia    r0, {r3, r5, r6}
    sub    r0, r0, r3            @ virt->phys offset
    add    r5, r5, r0            @ phys __turn_mmu_on
    add    r6, r6, r0            @ phys __turn_mmu_on_end
    mov    r5, r5, lsr #SECTION_SHIFT
    mov    r6, r6, lsr #SECTION_SHIFT

1:    orr    r3, r7, r5, lsl #SECTION_SHIFT    @ flags + kernel base
    str    r3, [r4, r5, lsl #PMD_ORDER]    @ identity mapping
    cmp    r5, r6
    addlo    r5, r5, #1            @ next section
    blo    1b
...
    .ltorg
    .align
__turn_mmu_on_loc:
    .long    .
    .long    __turn_mmu_on
    .long    __turn_mmu_on_end
```
![](index_files/11862518.png)
![](index_files/11887447.png)

1. adr    r0, __turn_mmu_on_loc => ADR r0,{pc}+0xac
将当前指令的地址与0xac的和赋给r0,即获取标号__turn_mmu_on_loc物理地址（运行时地址），文中为`0x80008138`

2. ldmia    r0, {r3, r5, r6}
将标号__turn_mmu_on_loc、__turn_mmu_on和__turn_mmu_on_end的虚地址（链接地址）加载到r3、r5和r6

3. sub    r0, r0, r3            @ virt->phys offset
获取标号__turn_mmu_on_loc虚地址和物理地址的差值

4. add    r5, r5, r0            @ phys __turn_mmu_on
__turn_mmu_on虚地址 + （__turn_mmu_on_loc物理地址 - __turn_mmu_on_loc虚地址） =
（__turn_mmu_on虚地址 - __turn_mmu_on_loc虚地址） + __turn_mmu_on_loc物理地址 = __turn_mmu_on物理地址

5. add    r6, r6, r0            @ phys __turn_mmu_on_end
计算__turn_mmu_on_end物理地址

6. mov    r5, r5, lsr #SECTION_SHIFT =》 LSR r5,r5,20
__turn_mmu_on物理地址右移20位，得到段基址`0x801`

**注意**：段大小为1M字节。

6. mov    r6, r6, lsr #SECTION_SHIFT =》 LSR r6,r65,20
__turn_mmu_on_end物理地址右移20位，得到段基址`0x801`

7. orr    r3, r7, r5, lsl #SECTION_SHIFT    => ORR r3,r7,r5,LSL #20
段基址0x801左移20位并设置MMU标志，得到页表项`0x80111C0E`

8. str    r3, [r4, r5, lsl #PMD_ORDER] => STR r3,[r4,r5,LSL #2]
页表项0x80111C0E存入`r4包含的页表基地址（0x80004000）+（r5包含的段基址 * 页表项大小）`即`0x80006004`

**注意**：每个页表项4个字节。

9. cmp    r5, r6
比较__turn_mmu_on段基址和__turn_mmu_on_end段基址，检查是否映射完成

10. addlo    r5, r5, #1
如果r5小于r6，则段基址加1，跳转到下一个段

11. blo    1b
如果r5小于r6，则跳转到标号1

## 3.3 映射内核映像（代码段、数据段、BSS段）
```cpp
/*
     * Map our RAM from the start to the end of the kernel .bss section.
     */
    add    r0, r4, #PAGE_OFFSET >> (SECTION_SHIFT - PMD_ORDER)
    ldr    r6, =(_end - 1)
    orr    r3, r8, r7
    add    r6, r4, r6, lsr #(SECTION_SHIFT - PMD_ORDER)
1:    str    r3, [r0], #1 << PMD_ORDER
    add    r3, r3, #1 << SECTION_SHIFT
    cmp    r0, r6
    bls    1b
```
![](index_files/15388344.png)
1. add    r0, r4, #PAGE_OFFSET >> (SECTION_SHIFT - PMD_ORDER)
链接地址（虚地址）右移20位，得到段地址，然后再左移2位，即乘以页表项大小（4字节），得到页表项地址，放入r0

2. ldr    r6, =(_end - 1)

得到内核结束地址的链接地址

3. orr    r3, r8, r7
PHYS_OFFSET即PAGE_OFFSET的物理地址或上MMU标志放入r3

4. add    r6, r4, r6, lsr #(SECTION_SHIFT - PMD_ORDER)
内核结束地址的链接地址转换成映射结束地址

5. 1： str    r3, [r0], #1 << PMD_ORDER
保存页表项，并更新页表项指针

6. add    r3, r3, #1 << SECTION_SHIFT
增加页表项中的物理地址基址

7. cmp    r0, r6
比较页表项指针和映射结束地址，确认是否结束

8. bls    1b
未完成则跳转到标号1

## 3.4 映射启动参数（ATAGs/DTB）
```cpp
/*
     * Then map boot params address in r2 if specified.
     * We map 2 sections in case the ATAGs/DTB crosses a section boundary.
     */
    mov    r0, r2, lsr #SECTION_SHIFT
    movs    r0, r0, lsl #SECTION_SHIFT
    subne    r3, r0, r8
    addne    r3, r3, #PAGE_OFFSET
    addne    r3, r4, r3, lsr #(SECTION_SHIFT - PMD_ORDER)
    orrne    r6, r7, r0
    strne    r6, [r3], #1 << PMD_ORDER
    addne    r6, r6, #1 << SECTION_SHIFT
    strne    r6, [r3]
```
![](index_files/17060097.png)
1. mov    r0, r2, lsr #SECTION_SHIFT
右移20位，清除启动参数地址非对齐字节

2. movs    r0, r0, lsl #SECTION_SHIFT
左移20位并更新进位标志（如果r0是0则设置Z标志，即相等）

3. subne    r3, r0, r8
如果启动参数地址（r2）非0，则减去PHYS_OFFSET即PAGE_OFFSET的物理地址以得到物理地址偏差

4. addne    r3, r3, #PAGE_OFFSET
如果启动参数地址（r2）非0，则加上PAGE_OFFSET以获取虚地址

5. addne    r3, r4, r3, lsr #(SECTION_SHIFT - PMD_ORDER)
如果启动参数地址（r2）非0，则根据虚地址计算页表项地址

6. orrne    r6, r7, r0
如果启动参数地址（r2）非0，则启动参数地址或上MMU选项

7. strne    r6, [r3], #1 << PMD_ORDER
更新页表项，并同时更新页表指针

8. addne    r6, r6, #1 << SECTION_SHIFT
如果启动参数地址（r2）非0，则启动参数地址增加1M，映射第二个页表项

9. strne    r6, [r3]
更新第二个页表项

## 3.5 映射串口
```cpp
    /*
     * Map in IO space for serial debugging.
     * This allows debug messages to be output
     * via a serial console before paging_init.
     */
    /* addruart r7, r3, r0 */
    ldr    r7, =KT_LL_UART_PADDR     @ physical
    ldr    r3, =KT_LL_UART_VADDR    @ virtual

    mov    r3, r3, lsr #SECTION_SHIFT
    mov    r3, r3, lsl #PMD_ORDER

    add    r0, r4, r3
    mov    r3, r7, lsr #SECTION_SHIFT
    ldr    r7, [r10, #PROCINFO_IO_MMUFLAGS] @ io_mmuflags
    orr    r3, r7, r3, lsl #SECTION_SHIFT
#ifdef CONFIG_ARM_LPAE
...
#else
    orr    r3, r3, #PMD_SECT_XN
    str    r3, [r0], #4
#endif
```
![](index_files/21707679.png)

1. ldr    r7, =KT_LL_UART_PADDR
从相对位置即运行时地址`0x80008160`加载串口物理地址`0x28001000`

2. ldr    r3, =KT_LL_UART_VADDR
从相对位置即运行时地址`0x80008164`加载串口虚拟地址`0xF0801000`

3. mov    r3, r3, lsr #SECTION_SHIFT
虚拟地址转换成段地址`0x00000F08`

4. mov    r3, r3, lsl #PMD_ORDER
段地址乘以4得到页表项偏移地址`0x00003C20`

5. add    r0, r4, r3
页表项地址加上页表基址得到页表项地址`0x80007C20`

6. mov    r3, r7, lsr #SECTION_SHIFT
物理地址右移20位得到段地址`0x00000280`

7. ldr    r7, [r10, #PROCINFO_IO_MMUFLAGS] @ io_mmuflags
从处理器信息`0x8040F6DC`得到IO空间MMU标志`0x00000C02`

8. orr    r3, r7, r3, lsl #SECTION_SHIFT
得到页表项内容`0x28000C02`

9. orr    r3, r3, #PMD_SECT_XN
设置访问权限，即禁止执行权限

10. str    r3, [r0], #4
保存页表项`0x28000C12`并更新页表项指针

# 4. 使能MMU

## 4.1 主体部分

```cpp
/*
     * The following calls CPU specific code in a position independent
     * manner.  See arch/arm/mm/proc-*.S for details.  r10 = base of
     * xxx_proc_info structure selected by __lookup_processor_type
     * above.
     *
     * The processor init function will be called with:
     *  r1 - machine type
     *  r2 - boot data (atags/dt) pointer
     *  r4 - translation table base (low word)
     *  r5 - translation table base (high word, if LPAE)
     *  r8 - translation table base 1 (pfn if LPAE)
     *  r9 - cpuid
     *  r13 - virtual address for __enable_mmu -> __turn_mmu_on
     *
     * On return, the CPU will be ready for the MMU to be turned on,
     * r0 will hold the CPU control register value, r1, r2, r4, and
     * r9 will be preserved.  r5 will also be preserved if LPAE.
     */
    ldr    r13, =__mmap_switched        @ address to jump to after
                        @ mmu has been enabled
    badr    lr, 1f                @ return (PIC) address
#ifdef CONFIG_ARM_LPAE
    mov    r5, #0                @ high TTBR0
    mov    r8, r4, lsr #12            @ TTBR1 is swapper_pg_dir pfn
#else
    mov    r8, r4                @ set TTBR1 to swapper_pg_dir
#endif
    ldr    r12, [r10, #PROCINFO_INITFUNC]
    add    r12, r12, r10
    ret    r12
1:    b    __enable_mmu
ENDPROC(stext)
    .ltorg
```
![](index_files/23215460.png)
1. ldr    r13, =__mmap_switched => LDR sp,[pc,#20]
从运行地址`pc+0x20`处获取__mmap_switched的虚地址（链接地址），存放到`sp`

**注意**：最后的`.ltorg`伪指令用于显式定义这条指令用到的文字池，用于存放__mmap_switched的虚地址。

2. badr    lr, 1f
将标号1所在的物理地址（运行时地址）写入`lr`寄存器

3. mov    r8, r4
将初始页表基地址放入`r8`

4. ldr    r12, [r10, #PROCINFO_INITFUNC]
读取保存在处理器信息结构中的初始化函数偏移，放入`r12`；其中`r10`中存放的是处理器信息结构地址：
    * porc-macros.s
```cpp
.macro    initfn, func, base
    .long    \func - \base
.endm
```
    * porc-v7.s
```cpp
.macro __v7_proc name, initfunc, mm_mmuflags = 0, io_mmuflags = 0, hwcaps = 0, proc_fns = v7_processor_functions
    ALT_SMP(.long    PMD_TYPE_SECT | PMD_SECT_AP_WRITE | PMD_SECT_AP_READ | \
            PMD_SECT_AF | PMD_FLAGS_SMP | \mm_mmuflags)
    ALT_UP(.long    PMD_TYPE_SECT | PMD_SECT_AP_WRITE | PMD_SECT_AP_READ | \
            PMD_SECT_AF | PMD_FLAGS_UP | \mm_mmuflags)
    .long    PMD_TYPE_SECT | PMD_SECT_AP_WRITE | \
        PMD_SECT_AP_READ | PMD_SECT_AF | \io_mmuflags
    initfn    \initfunc, \name
    .long    cpu_arch_name
    .long    cpu_elf_name
    .long    HWCAP_SWP | HWCAP_HALF | HWCAP_THUMB | HWCAP_FAST_MULT | \
        HWCAP_EDSP | HWCAP_TLS | \hwcaps
    .long    cpu_v7_name
    .long    \proc_fns
    .long    v7wbi_tlb_fns
    .long    v6_user_fns
    .long    v7_cache_fns
.endm
...
    /*
     * ARM Ltd. Cortex A9 processor.
     */
    .type   __v7_ca9mp_proc_info, #object
__v7_ca9mp_proc_info:
    .long    0x410fc090
    .long    0xff0ffff0
    __v7_proc __v7_ca9mp_proc_info, __v7_ca9mp_setup, proc_fns = ca9mp_processor_functions
    .size    __v7_ca9mp_proc_info, . - __v7_ca9mp_proc_info
```
    * procinfo.h
```cpp
struct proc_info_list {
    unsigned int        cpu_val;
    unsigned int        cpu_mask;
    unsigned long        __cpu_mm_mmu_flags;    /* used by head.S */
    unsigned long        __cpu_io_mmu_flags;    /* used by head.S */
    unsigned long        __cpu_flush;        /* used by head.S */
    const char        *arch_name;
    const char        *elf_name;
    unsigned int        elf_hwcap;
    const char        *cpu_name;
    struct processor    *proc;
    struct cpu_tlb_fns    *tlb;
    struct cpu_user_fns    *user;
    struct cpu_cache_fns    *cache;
};
```
如上所示，偏移0x10处存放的是__cpu_flush函数相对于proc_info_list的偏移。

5. add    r12, r12, r10
获取__cpu_flush函数真实地址

6. ret    r12 =》 mov pc,r12
跳转到__cpu_flush执行

## 4.2 设置操作第一阶段（即__v7_ca9mp_setup）
```cpp
/*
 *    __v7_setup
 *
 *    Initialise TLB, Caches, and MMU state ready to switch the MMU
 *    on.  Return in r0 the new CP15 C1 control register setting.
 *
 *    r1, r2, r4, r5, r9, r13 must be preserved - r13 is not a stack
 *    r4: TTBR0 (low word)
 *    r5: TTBR0 (high word if LPAE)
 *    r8: TTBR1
 *    r9: Main ID register
 *
 *    This should be able to cover all ARMv7 cores.
 *
 *    It is assumed that:
 *    - cache type register is implemented
 */
__v7_ca5mp_setup:
__v7_ca9mp_setup:
...
    mov    r10, #(1 << 0)            @ Cache/TLB ops broadcasting
    b    1f
...
__v7_b15mp_setup:
...
    mov    r10, #0
1:    adr    r0, __v7_setup_stack_ptr
    ldr    r12, [r0]
    add    r12, r12, r0            @ the local stack
    stmia    r12, {r1-r6, lr}        @ v7_invalidate_l1 touches r0-r6
    bl      v7_invalidate_l1
    ldmia    r12, {r1-r6, lr}
#ifdef CONFIG_SMP
    orr    r10, r10, #(1 << 6)        @ Enable SMP/nAMP mode
    ALT_SMP(mrc    p15, 0, r0, c1, c0, 1)
    ALT_UP(mov    r0, r10)        @ fake it for UP
    orr    r10, r10, r0            @ Set required bits
    teq    r10, r0                @ Were they already set?
    mcrne    p15, 0, r10, c1, c0, 1        @ No, update register
#endif
    b    __v7_setup_cont
...
    .align    2
__v7_setup_stack_ptr:
    .word    PHYS_RELATIVE(__v7_setup_stack, .)
ENDPROC(__v7_setup)

    .bss
    .align    2
__v7_setup_stack:
    .space    4 * 7                @ 7 registers
```

![](index_files/33015146.png)
1. mov    r10, #(1 << 0)
设置cache操作广播标志

2. b    1f
跳转到标号1，越过非a9操作

3. adr    r0, __v7_setup_stack_ptr => S:0x80119108 : ADR      r0,{pc}+0x128 ; 0x80119230
获取设置栈指针的运行时地址
![](index_files/32871844.png)

4. ldr    r12, [r0]
获取设置栈相对于设置栈指针的偏移地址

5. add    r12, r12, r0
计算栈的运行时地址
```cpp
#ifdef CONFIG_XIP_KERNEL
...
#else
#define PHYS_RELATIVE(v_data, v_text) ((v_data) - (v_text))
#endif
```

6. stmia    r12, {r1-r6, lr}
递增保存r1-r6和lr到设置栈:
![](index_files/34315571.png)

7. bl      v7_invalidate_l1
无效L1 Cache

8. ldmia    r12, {r1-r6, lr}
恢复r1-r6和lr

9. orr    r10, r10, #(1 << 6)
使能SMP/nAMP模式

10. ALT_SMP(mrc    p15, 0, r0, c1, c0, 1) => MRC p15,#0x0,r0,c1,c0,#1
读取CP15（MMU协处理器）的ACTLR寄存器

11. ALT_UP(mov    r0, r10) => 空指令

12. orr    r10, r10, r0
设置SMP位

13. teq    r10, r0
检查SMP位是否已经设置

14. mcrne    p15, 0, r10, c1, c0, 1
未设置则需要更新ACTLR

15. b    __v7_setup_cont
继续执行设置操作

## 4.3 设置操作第二阶段（即errata阶段）

```cpp
/*
 * Errata:
 *  r0, r10 available for use
 *  r1, r2, r4, r5, r9, r13: must be preserved
 *  r3: contains MIDR rX number in bits 23-20
 *  r6: contains MIDR rXpY as 8-bit XY number
 *  r9: MIDR
 */
__ca9_errata:
...
    b    __errata_finish
...
__v7_setup_cont:
    and    r0, r9, #0xff000000        @ ARM?
    teq    r0, #0x41000000
    bne    __errata_finish
    and    r3, r9, #0x00f00000        @ variant
    and    r6, r9, #0x0000000f        @ revision
    orr    r6, r6, r3, lsr #20-4        @ combine variant and revision
    ubfx    r0, r9, #4, #12            @ primary part number
...
    /* Cortex-A9 Errata */
    ldr    r10, =0x00000c09        @ Cortex-A9 primary part number
    teq    r0, r10
    beq    __ca9_errata
```
![](index_files/32795076.png)

1. and    r0, r9, #0xff000000
获取MIDR寄存器31~24位

2. teq    r0, #0x41000000
检查是否为ARM处理器

3. bne    __errata_finish
不是ARM处理器则跳过errata阶段

4. and    r3, r9, #0x00f00000
获取变种信息

5. and    r6, r9, #0x0000000f
获取revision信息

6. orr    r6, r6, r3, lsr #20-4
合并变种和revision信息

7. ubfx    r0, r9, #4, #12
获取primary part number

8. ldr    r10, =0x00000c09
设置A9 处理器primary part number

9. teq    r0, r10
比较

10. beq    __ca9_errata
跳转到A9 errata操作， 即空操作，再次跳转到__errata_finish
![](index_files/37229576.png)

## 4.4 设置操作第三阶段

```cpp
__errata_finish:
    mov    r10, #0
    mcr    p15, 0, r10, c7, c5, 0        @ I+BTB cache invalidate
#ifdef CONFIG_MMU
    mcr    p15, 0, r10, c8, c7, 0        @ invalidate I + D TLBs
    v7_ttb_setup r10, r4, r5, r8, r3    @ TTBCR, TTBRx setup
    ldr    r3, =PRRR            @ PRRR
    ldr    r6, =NMRR            @ NMRR
    mcr    p15, 0, r3, c10, c2, 0        @ write PRRR
    mcr    p15, 0, r6, c10, c2, 1        @ write NMRR
#endif
    dsb                    @ Complete invalidations
#ifndef CONFIG_ARM_THUMBEE
    mrc    p15, 0, r0, c0, c1, 0        @ read ID_PFR0 for ThumbEE
    and    r0, r0, #(0xf << 12)        @ ThumbEE enabled field
    teq    r0, #(1 << 12)            @ check if ThumbEE is present
    bne    1f
    mov    r3, #0
    mcr    p14, 6, r3, c1, c0, 0        @ Initialize TEEHBR to 0
    mrc    p14, 6, r0, c0, c0, 0        @ load TEECR
    orr    r0, r0, #1            @ set the 1st bit in order to
    mcr    p14, 6, r0, c0, c0, 0        @ stop userspace TEEHBR access
1:
#endif
    adr    r3, v7_crval
    ldmia    r3, {r3, r6}
 ARM_BE8(orr    r6, r6, #1 << 25)        @ big-endian page tables
#ifdef CONFIG_SWP_EMULATE
    orr     r3, r3, #(1 << 10)              @ set SW bit in "clear"
    bic     r6, r6, #(1 << 10)              @ clear it in "mmuset"
#endif
       mrc    p15, 0, r0, c1, c0, 0        @ read control register
    bic    r0, r0, r3            @ clear bits them
    orr    r0, r0, r6            @ set them
 THUMB(    orr    r0, r0, #1 << 30    )    @ Thumb exceptions
    ret    lr                @ return to head.S:__ret
```
![](index_files/32852796.png)

1. mov    r10, #0
清零r10

2. mcr    p15, 0, r10, c7, c5, 0
清零ICIALLU以无效所有指令Cache

3. mcr    p15, 0, r10, c8, c7, 0
清零TLBIALLH以无效ITLB Cache和DTLB Cache

4. v7_ttb_setup r10, r4, r5, r8, r3
```cpp
S:0x801191C8 : MCR      p15,#0x0,r10,c2,c0,#2
S:0x801191CC : ORR      r4,r4,#0x6a
S:0x801191D0 : ORR      r8,r8,#0x6a
S:0x801191D4 : MCR      p15,#0x0,r8,c2,c0,#1
```
清零TTBCR；TTBR0/1(swapper_pg_dir地址)加上SMP需要的cache属性（内部和外部cache的Write-Back、Write-Alloc、Sharable属性）；写入TTBR1

5. ldr    r3, =PRRR
加载PRRR（Primary Region Remap Register）的值

6. ldr    r3, =NMRR
加载NMRR（Normal Memory Remap Register）的值

7. mcr    p15, 0, r3, c10, c2, 0
写PRRR

8. mcr    p15, 0, r6, c10, c2, 1
写NMRR

9. dsb
同步

10. mrc    p15, 0, r0, c0, c1, 0
读取ID_PFR0

11. and    r0, r0, #(0xf << 12)
获取ThumbEE位

12. teq    r0, #(1 << 12)
检查ThumbEE是否支持

13. bne    1f
不支持跳转到标号1

14. mov    r3, #0
清零r3

15. mcr    p14, 6, r3, c1, c0, 0
清零TEEHBR

16. mrc    p14, 6, r0, c0, c0, 0
获取TEECR

17. orr    r0, r0, #1
禁止非特权级（用户空间）访问

18. mcr    p14, 6, r0, c0, c0, 0
更新TEECR

19. adr    r3, v7_crval
* proc-macros.s
```cpp
    .macro    crval, clear, mmuset, ucset
#ifdef CONFIG_MMU
    .word    \clear
    .word    \mmuset
#else
    .word    \clear
    .word    \ucset
#endif
    .endm
```
* proc-v7-2level.s
```cpp
    .align    2
    .type    v7_crval, #object
v7_crval:
    crval    clear=0x2120c302, mmuset=0x10c03c7d, ucset=0x00c01c7c
```
加载MMU控制寄存器默认值指针

20. ldmia    r3, {r3, r6}
加载MMU控制寄存器默认值

21. orr     r3, r3, #(1 << 10)
clear值中设置SW位，用于清除SCTLR的SW位

22. bic     r6, r6, #(1 << 10)
mmuset值中清除SW位，确保清除SCTLR的SW位

23. mrc    p15, 0, r0, c1, c0, 0
读取SCTLR

24.  bic    r0, r0, r3
从SCTLR清除clear值包含的位

25. orr    r0, r0, r6
将mmuset值包含的位赋给SCTLR

26. ret    lr
返回

## 4.5 使能MMU

```cpp
/*
 * Setup common bits before finally enabling the MMU.  Essentially
 * this is just loading the page table pointer and domain access
 * registers.  All these registers need to be preserved by the
 * processor setup function (or set in the case of r0)
 *
 *  r0  = cp#15 control register
 *  r1  = machine ID
 *  r2  = atags or dtb pointer
 *  r4  = TTBR pointer (low word)
 *  r5  = TTBR pointer (high word if LPAE)
 *  r9  = processor ID
 *  r13 = *virtual* address to jump to upon completion
 */
__enable_mmu:
#if defined(CONFIG_ALIGNMENT_TRAP) && __LINUX_ARM_ARCH__ < 6
    orr    r0, r0, #CR_A
#else
    bic    r0, r0, #CR_A
#endif
...
#ifdef CONFIG_ARM_LPAE
    mcrr    p15, 0, r4, r5, c2        @ load TTBR0
#else
    mov    r5, #DACR_INIT
    mcr    p15, 0, r5, c3, c0, 0        @ load domain access register
    mcr    p15, 0, r4, c2, c0, 0        @ load page table pointer
#endif
    b    __turn_mmu_on
ENDPROC(__enable_mmu)
```
![](index_files/27901465.png)

1. bic    r0, r0, #CR_A
关闭对齐故障检查

2. mov    r5, #DACR_INIT
获取DACR（Domain Access Control Register）初值

3. mcr    p15, 0, r5, c3, c0, 0
设置DACR

4. mcr    p15, 0, r4, c2, c0, 0
设置TTBR0

5. b    __turn_mmu_on
跳转到__turn_mmu_on

## 4.6 __turn_mmu_on（）

```cpp
/*
 * Enable the MMU.  This completely changes the structure of the visible
 * memory space.  You will not be able to trace execution through this.
 * If you have an enquiry about this, *please* check the linux-arm-kernel
 * mailing list archives BEFORE sending another post to the list.
 *
 *  r0  = cp#15 control register
 *  r1  = machine ID
 *  r2  = atags or dtb pointer
 *  r9  = processor ID
 *  r13 = *virtual* address to jump to upon completion
 *
 * other registers depend on the function called upon completion
 */
    .align    5
    .pushsection    .idmap.text, "ax"
ENTRY(__turn_mmu_on)
    mov    r0, r0
    instr_sync
    mcr    p15, 0, r0, c1, c0, 0        @ write control reg
    mrc    p15, 0, r3, c0, c0, 0        @ read id reg
    instr_sync
    mov    r3, r3
    mov    r3, r13
    ret    r3
__turn_mmu_on_end:
ENDPROC(__turn_mmu_on)
    .popsection
```
![](index_files/29744398.png)
1. mov    r0, r0
NOP指令，用于隔离前面的指令

2. instr_sync => ISB
确保指令执行完成

3. mcr    p15, 0, r0, c1, c0, 0
写入SCTLR，使能MMU

4. mrc    p15, 0, r3, c0, c0, 0
读取MIDR

5. instr_sync => ISB
确保指令执行完成

6. mov    r3, r3
NOP指令，用于隔离前面的指令

7. mov    r3, r13
将返回指针即__mmap_switched虚地址保存到r3，释放r13，即SP

8. ret    r3
跳转到__mmap_switched

## 4.7 __mmap_switched（）

```cpp
/*
 * The following fragment of code is executed with the MMU on in MMU mode,
 * and uses absolute addresses; this is not position independent.
 *
 *  r0  = cp#15 control register
 *  r1  = machine ID
 *  r2  = atags/dtb pointer
 *  r9  = processor ID
 */
    __INIT
__mmap_switched:
    adr    r3, __mmap_switched_data

    ldmia    r3!, {r4, r5, r6, r7}
    cmp    r4, r5                @ Copy data segment if needed
1:    cmpne    r5, r6
    ldrne    fp, [r4], #4
    strne    fp, [r5], #4
    bne    1b

    mov    fp, #0                @ Clear BSS (and zero fp)
1:    cmp    r6, r7
    strcc    fp, [r6],#4
    bcc    1b

 ARM(    ldmia    r3, {r4, r5, r6, r7, sp})
 THUMB(    ldmia    r3, {r4, r5, r6, r7}    )
 THUMB(    ldr    sp, [r3, #16]        )
    str    r9, [r4]            @ Save processor ID
    str    r1, [r5]            @ Save machine type
    str    r2, [r6]            @ Save atags pointer
    cmp    r7, #0
    strne    r0, [r7]            @ Save control register values
    b    start_kernel
ENDPROC(__mmap_switched)
    .align    2
    .type    __mmap_switched_data, %object
__mmap_switched_data:
    .long    __data_loc            @ r4
    .long    _sdata                @ r5
    .long    __bss_start            @ r6
    .long    _end                @ r7
    .long    processor_id            @ r4
    .long    __machine_arch_type        @ r5
    .long    __atags_pointer            @ r6
#ifdef CONFIG_CPU_CP15
    .long    cr_alignment            @ r7
#else
    .long    0                @ r7
#endif
    .long    init_thread_union + THREAD_START_SP @ sp
    .size    __mmap_switched_data, . - __mmap_switched_data
```
![](index_files/31488474.png)

1. adr    r3, __mmap_switched_data
从当前指令的相对地址获取__mmap_switched_data的虚拟地址

2. ldmia    r3!, {r4, r5, r6, r7}
加载__data_loc、_sdata、__bss_start、_end的虚地址，并将r3递增16字节：

**注意**：这些变量定义在./arch/arm/kernel/vmlinux.lds中，表示data段地址、sdata段（第一个数据段）地址、bss段地址、image结束地址

3. cmp    r4, r5
比较data段和sdata段地址

4. cmpne    r5, r6
如果data段和sdata段地址不相同，则检查sdata段是否结束

5. ldrne    fp, [r4], #4
如果sdata段未结束，则从数据段读取数据

6. strne    fp, [r5], #4
如果sdata段未结束，则将数据段读出的数据写入到sdata段

7. bne    1b
如果sdata段未结束，则跳转到标号1继续搬移数据

8. mov    fp, #0
清零r11

9. cmp    r6, r7
检查bss段是否结束

10. strcc    fp, [r6],#4
如果bss段未结束，则清零

11. bcc    1b
如果bss段未结束，则跳转到标号1

12.  ARM(    ldmia    r3, {r4, r5, r6, r7, sp})
加载processor_id、__machine_arch_type、__atags_pointer、cr_alignment和初始栈的虚地址。

**注意**：初始栈基址定义如下：
* ./arch/arm/kernel/vmlinux.lds
```
.data : AT(__data_loc) {
919   _data = .; /* address in memory */
920   _sdata = .;
921   /*
922          * first, the init task union, aligned
923          * to an 8192 byte boundary.
924          */
925   . = ALIGN(((1 << 12) << 1)); __start_init_task = .; *(.data..init_task) __end_init_task = .;
926   . = ALIGN((1 << 12)); __nosave_begin = .; *(.data..nosave) . = ALIGN((1 << 12)); __nosave_end = .;
927   . = ALIGN((1 << 6)); *(.data..cacheline_aligned)
928   . = ALIGN((1 << 6)); *(.data..read_mostly) . = ALIGN((1 << 6));
...
}
```
* init_task.h/init_task.c
```cpp
/* Attach to the init_task data structure for proper alignment */
#define __init_task_data __attribute__((__section__(".data..init_task")))
/*
 * Initial thread structure. Alignment of this is handled by a special
 * linker map entry.
 */
union thread_union init_thread_union __init_task_data = {
#ifndef CONFIG_THREAD_INFO_IN_TASK
    INIT_THREAD_INFO(init_task)
#endif
};
```
* thread_info.h
```cpp
#define THREAD_SIZE_ORDER    1
#define THREAD_SIZE        (PAGE_SIZE << THREAD_SIZE_ORDER)
#define THREAD_START_SP        (THREAD_SIZE - 8)
```

13. str    r9, [r4]            @ Save processor ID
保存处理器ID

14. str    r1, [r5]
保存引导程序传进来的machine type

15. str    r2, [r6]
保存atags指针

16. cmp    r7, #0
检查是否使用cr_alignment

17. strne    r0, [r7]            @ Save control register values
未使用则保存SCTLR的值

18. b    start_kernel
跳转到start_kernel执行
