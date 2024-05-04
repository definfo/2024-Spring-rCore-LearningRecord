# 2024-Spring-rCore-LearningRecord

Written on Typora & VSCodium

第一阶段暂未整理笔记（

### 2024-04-20 --- 2024-04-28

提交了 Rustlings 练习，熟悉基本的 Rust 语法，Rust 实战仍需努力。

1. 学习 MIT 6.1810(6.S081) xv6 lab，完成 Unix lab sleep & pingpong，初步理解系统编程的项目架构和管道逻辑。
2. 阅读 rCore Tutorial，配置 QEMU 环境，了解 bare-metal 启动流程和 Rust User Mode 调用 syscall 的方法。
3. 阅读 RISC-V ISA 文档以及特权指令集总结。
4. 学习 Berkeley CS61C，完成 Lab 1 & 2。

### 2024-04-29 --- 2024-05-04

开始第二阶段学习

1. 观看 Rust bare-metal 课程，~~被 Rust ASM 吓晕~~学习 RISC-V ASM 操作以及 Rust sbi_call 实现。
2. 阅读 rCore Tutorial，结合往年 Slide 和完整文档加深理解，整理 ch1 & 2 项目实现逻辑。



#### Ch1 - 应用程序与基本执行环境

> 见识下应用程序的真面目！

主要知识点：

- 应用程序执行环境、标准库依赖

- Rust 汇编语法
- 裸机（QEMU）启动流程、程序内存布局和编译流程
- 链接脚本语法

- GDB 调试
- （重要！）函数调用与栈
- SBI 调用方法

---

##### 应用程序执行环境

> ​	如下图所示，现在通用操作系统（如 Linux 等）上的应用程序运行需要下面多层次的执行环境栈的支持，图中的白色块自上而下（越往下则越靠近底层，下层作为上层的执行环境支持上层代码的运行）表示各级执行环境，黑色块则表示相邻两层执行环境之间的接口。

通过**多层抽象**实现的现代操作系统：

![../_images/app-software-stack.png](https://rcore-os.cn/rCore-Tutorial-Book-v3/_images/app-software-stack.png)

> 计算机科学中遇到的所有问题都可通过增加一层抽象来解决。
>
> All problems in computer science can be solved by another level of indirection。
>
> – 计算机科学家 David Wheeler

编译需要指定可执行文件的**目标平台**，一般为三元组(arch-vendor-OS[-lib])形式。

例如：

```cmd
$ rustc --print target-list | grep riscv
riscv32gc-unknown-linux-gnu
riscv32gc-unknown-linux-musl
riscv32i-unknown-none-elf
riscv32im-risc0-zkvm-elf
riscv32im-unknown-none-elf
riscv32ima-unknown-none-elf
riscv32imac-esp-espidf
riscv32imac-unknown-none-elf
riscv32imac-unknown-xous-elf
riscv32imafc-esp-espidf
riscv32imafc-unknown-none-elf
riscv32imc-esp-espidf
riscv32imc-unknown-none-elf
riscv64-linux-android
riscv64gc-unknown-freebsd
riscv64gc-unknown-fuchsia
riscv64gc-unknown-hermit
riscv64gc-unknown-linux-gnu
riscv64gc-unknown-linux-musl
riscv64gc-unknown-netbsd
riscv64gc-unknown-none-elf
riscv64gc-unknown-openbsd
riscv64imac-unknown-none-elf
```

riscv(32/64)(imafdc) : RISC-V [32-bit/64-bit] platform with [integer, multiply, atomic, float, double, compression] instruction extension

g = imafd



gnu : GNU libc

elf : no standard runtime library but could produce executable and linkable file



Rust std lib 类似于 GNU libc，需要操作系统支持。Rust core lib 则不需要操作系统支持，可以在裸机程序上使用。

---



##### 程序内存布局

在我们将源代码编译为可执行文件之后，它就会变成一个看似充满了杂乱无章的字节的一个文件。但我们知道这些字节至少可以分成代码和数据两部分，在程序运行起来的时候它们的功能并不相同：代码部分由一条条可以被 CPU 解码并执行的指令组成，而数据部分只是被 CPU 视作可读写的内存空间。事实上我们还可以根据其功能进一步把两个部分划分为更小的单位： **段** (Section) 。不同的段会被编译器放置在内存不同的位置上，这构成了程序的 **内存布局** (Memory Layout)。一种典型的程序相对内存布局如下所示：

![../_images/MemoryLayout.png](https://rcore-os.cn/rCore-Tutorial-Book-v3/_images/MemoryLayout.png)

一种典型的程序相对内存布局

在上图中可以看到，代码部分只有代码段 `.text` 一个段，存放程序的所有汇编代码。而数据部分则还可以继续细化：

- 已初始化数据段保存程序中那些已初始化的全局数据，分为 `.rodata` 和 `.data` 两部分。前者存放只读的全局数据，通常是一些常数或者是 常量字符串等；而后者存放可修改的全局数据。
- 未初始化数据段 `.bss` 保存程序中那些未初始化的全局数据，通常由程序的加载者代为进行零初始化，即将这块区域逐字节清零；
- **堆** （heap）区域用来存放程序运行时动态分配的数据，如 C/C++ 中的 malloc/new 分配到的数据本体就放在堆区域，它向高地址增长；
- **栈** （stack）区域不仅用作函数调用上下文的保存与恢复，每个函数作用域内的局部变量也被编译器放在它的栈帧内，它向低地址增长。

**局部变量与全局变量**

在一个函数的视角中，它能够访问的变量包括以下几种：

- 函数的输入参数和局部变量：保存在一些寄存器或是该函数的栈帧里面，如果是在栈帧里面的话是基于当前栈指针加上一个偏移量来访问的；
- 全局变量：保存在数据段 `.data` 和 `.bss` 中，某些情况下 gp(x3) 寄存器保存两个数据段中间的一个位置，于是全局变量是基于 gp 加上一个偏移量来访问的。
- 堆上的动态变量：本体被保存在堆上，大小在运行时才能确定。而我们只能 *直接* 访问栈上或者全局数据段中的 **编译期确定大小** 的变量。因此我们需要通过一个运行时分配内存得到的一个指向堆上数据的指针来访问它，指针的位宽确实在编译期就能够确定。该指针即可以作为局部变量放在栈帧里面，也可以作为全局变量放在全局数据段中。

（此处为简化内核态原理，暂未引入虚拟内存地址和页表机制，程序直接访问物理地址）

---



##### 编译流程

从源代码得到可执行文件的编译流程可被细化为多个阶段（虽然输入一条命令便可将它们全部完成）：

1. **编译器** (Compiler) 将每个源文件从某门高级编程语言转化为汇编语言，注意此时源文件仍然是一个 ASCII 或其他编码的文本文件；
2. **汇编器** (Assembler) 将上一步的每个源文件中的文本格式的指令转化为机器码，得到一个二进制的 **目标文件** (Object File)；
3. **链接器** (Linker) 将上一步得到的所有目标文件以及一些可能的外部目标文件链接在一起形成一个完整的可执行文件。

汇编器输出的每个目标文件都有一个独立的程序内存布局，它描述了目标文件内各段所在的位置。而链接器所做的事情是将所有输入的目标文件整合成一个整体的内存布局。在此期间链接器主要完成两件事情：

- 第一件事情是将来自不同目标文件的段在目标内存布局中重新排布。如下图所示，在链接过程中，分别来自于目标文件 `1.o` 和 `2.o` 段被按照段的功能进行分类，相同功能的段被排在一起放在拼装后的目标文件 `output.o` 中。注意到，目标文件 `1.o` 和 `2.o` 的内存布局是存在冲突的，同一个地址在不同的内存布局中存放不同的内容。而在合并后的内存布局中，这些冲突被消除。

![../_images/link-sections.png](https://rcore-os.cn/rCore-Tutorial-Book-v3/_images/link-sections.png)

来自不同目标文件的段的重新排布

- 第二件事情是将符号替换为具体地址。这里的符号指什么呢？我们知道，在我们进行模块化编程的时候，每个模块都会提供一些向其他模块公开的全局变量、函数等供其他模块访问，也会访问其他模块向它公开的内容。要访问一个变量或者调用一个函数，在源代码级别我们只需知道它们的名字即可，这些名字被我们称为符号。取决于符号来自于模块内部还是其他模块，我们还可以进一步将符号分成内部符号和外部符号。然而，在机器码级别（也即在目标文件或可执行文件中）我们并不是通过符号来找到索引我们想要访问的变量或函数，而是直接通过变量或函数的地址。例如，如果想调用一个函数，那么在指令的机器码中我们可以找到函数入口的绝对地址或者相对于当前 PC 的相对地址。

  那么，符号何时被替换为具体地址呢？因为符号对应的变量或函数都是放在某个段里面的固定位置（如全局变量往往放在 `.bss` 或者 `.data` 段中，而函数则放在 `.text`  段中），所以我们需要等待符号所在的段确定了它们在内存布局中的位置之后才能知道它们确切的地址。当一个模块被转化为目标文件之后，它的内部符号就已经在目标文件中被转化为具体的地址了，因为目标文件给出了模块的内存布局，也就意味着模块内的各个段的位置已经被确定了。然而，此时模块所用到的外部符号的地址无法确定。我们需要将这些外部符号记录下来，放在目标文件一个名为符号表（Symbol table）的区域内。由于后续可能还需要重定位，内部符号也同样需要被记录在符号表中。

  外部符号需要等到链接的时候才能被转化为具体地址。假设模块 1 用到了模块 2  提供的内容，当两个模块的目标文件链接到一起的时候，它们的内存布局会被合并，也就意味着两个模块的各个段的位置均被确定下来。此时，模块 1  用到的来自模块 2  的外部符号可以被转化为具体地址。同时我们还需要注意：两个模块的段在合并后的内存布局中被重新排布，其最终的位置有可能和它们在模块自身的局部内存布局中的位置相比已经发生了变化。因此，每个模块的内部符号的地址也有可能会发生变化，我们也需要进行修正。上面的过程被称为重定位（Relocation），这个过程形象一些来说很像拼图：由于模块 1 用到了模块 2 的内容，因此二者分别相当于一块凹进和凸出一部分的拼图，正因如此我们可以将它们无缝地拼接到一起。

上面我们简单介绍了程序内存布局和编译流程特别是链接过程的相关知识。那么如何得到一个能够在 Qemu 上成功运行的内核镜像呢？

首先我们需要通过链接脚本调整内核可执行文件的内存布局，使得内核被执行的第一条指令位于地址 `0x80200000` 处，同时代码段所在的地址应低于其他段。这是因为 Qemu 物理内存中低于 `0x80200000` 的区域并未分配给内核，而是主要由 RustSBI  使用。

其次，我们需要将内核可执行文件中的元数据丢掉得到内核镜像，此内核镜像仅包含实际会用到的代码和数据。这则是因为 Qemu  的加载功能过于简单直接，它直接将输入的文件逐字节拷贝到物理内存中，因此也可以说这一步是我们在帮助 Qemu  手动将可执行文件加载到物理内存中。

下一节我们将成功生成内核镜像并在 Qemu 上验证控制权被转移到内核。

---



借助链接脚本 `linker.ld` 指定各段文件在内存中的排布方式

语法：

```
OUTPUT_ARCH(riscv) // program target architecture
ENTRY(_start) // program entry symbol
BASE_ADDRESS = 0x80200000; // set static base memory address

SECTIONS // combine input sections
{
    . = BASE_ADDRESS; // set current address to BASE_ADDRESS
    skernel = .; // kernel start symbol

    stext = .;
    // match sections in mode <ObjectFile>(SectionName)
    .text : {
        *(.text.entry)
        *(.text .text.*)
    }

    . = ALIGN(4K); // align the current address to a 4K boundary (详见内存对齐)
    etext = .;
    srodata = .;
    .rodata : {
        *(.rodata .rodata.*)
        *(.srodata .srodata.*)
    }

    . = ALIGN(4K);
    erodata = .;
    sdata = .;
    .data : {
        *(.data .data.*)
        *(.sdata .sdata.*)
    }

    . = ALIGN(4K);
    edata = .;
    .bss : {
        *(.bss.stack)
        sbss = .;
        *(.bss .bss.*)
        *(.sbss .sbss.*)
    }

    . = ALIGN(4K);
    ebss = .;
    ekernel = .; // kernel end symbol

	// discard error handling sections
    /DISCARD/ : {
        *(.eh_frame)
    }
}
```

