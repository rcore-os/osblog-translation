# 掌控 RISC-V

这是[用 Rust 编写 RISC-V 操作系统](http://osblog.stephenmarz.com/index.html)系列教程中的第 1 章。

[目录](index.md) → [第 0 章](ch0.md) → 第 1 章 → [第 2 章](ch2.md)

**<span style='color:red'>2019年9月27日</span>**

## 概述

启动并进入 RISC-V 系统很简单。在各种方法中，我将要介绍我自己的方法——从物理内存地址 `0x8000_0000` 开始。幸运的是， QEMU 可以读取 ELF 文件，所以它知道应该把我们的代码运行在哪个地址上。在整个过程中，我们将通过查看 QEMU 中包含的`qemu/hw/riscv/virt.c`源代码来获取一些信息。首先，先来看一下内存映射：

```c
static const struct MemmapEntry {
    hwaddr base;
    hwaddr size;
} virt_memmap[] = {
    [VIRT_DEBUG] =       {        0x0,         0x100 },
    [VIRT_MROM] =        {     0x1000,       0x11000 },
    [VIRT_TEST] =        {   0x100000,        0x1000 },
    [VIRT_CLINT] =       {  0x2000000,       0x10000 },
    [VIRT_PLIC] =        {  0xc000000,     0x4000000 },
    [VIRT_UART0] =       { 0x10000000,         0x100 },
    [VIRT_VIRTIO] =      { 0x10001000,        0x1000 },
    [VIRT_DRAM] =        { 0x80000000,           0x0 },
    [VIRT_PCIE_MMIO] =   { 0x40000000,    0x40000000 },
    [VIRT_PCIE_PIO] =    { 0x03000000,    0x00010000 },
    [VIRT_PCIE_ECAM] =   { 0x30000000,    0x10000000 },
};
```

由此可见，我们的机器从 DRAM(VIRT_DRAM) 的第0字节，地址 0x8000_0000 开始。当我们再往前走一点，我们将对CLINT（0x200_0000）、PLIC（0xc00_0000）、UART（0x1000_0000）和VIRTIO（0x1000_1000）编程。现在不要担心这些是什么意思，我们只需要看看接下来要做什么!

完成这些后，我们需要在RISC-V汇编中完成以下工作：

1. 选择一个CPU引导加载程序（通常是id#0）；
2. 将BSS部分清零；
3. 开始Rust!

RISC-V汇编类似于MIPS汇编，除了我们不需要给我们的寄存器加前缀。所有的指令都来自于RISC-V规范，你可以在以下网站获取：[https://github.com/riscv/riscv-isa-manual](https://github.com/riscv/riscv-isa-manual)。我们的编写对象是RV64GC（RISC-V 64位，一般扩展和压缩指令扩展）。

## 选择一个引导加载程序 (bootloader)

这个时候我们不考虑并行性、条件竞争或其他此类问题。相反，我们只让我们的一个CPU核心（RISC-V中称为HARTs[硬件线程]）做所有的工作。现在我们首先要深入研究特权级规范，来弄清我们要讨论的是哪个寄存器。因此，请在[https://github.com/riscv/riscv-isa-manual](https://github.com/riscv/riscv-isa-manual/)下载。
我们将从3.1.5章中开始（Hart ID寄存器 mhartid）。这个寄存器将告诉我们我们的hart编号。根据规范，我们必须有一个hart id #0。所以，我们以这个ID来启动。

在你的`src/asm/`目录下创建一个`boot.S`文件。我们将在此启动并进入Rust的编写。

```nasm
# boot.S
# bootloader for SoS
# Stephen Marz
# 8 February 2019
.option norvc
.section .data

.section .text.init
.global _start
_start:
    # Any hardware threads (hart) that are not bootstrapping
    # need to wait for an IPI
    csrr    t0, mhartid
    bnez    t0, 3f
    # SATP should be zero, but let's make sure
    csrw    satp, zero
.option push
.option norelax
    la      gp, _global_pointer
.option pop

3:
    wfi
    j   3b
```

这里csrr的意思是 "控制状态寄存器读取"，利用csrr我们把hart标识符读到寄存器t0中，检查它是否为零。如果不是，我们就把它送去循环等待。之后，我们将监管者地址转换和保护（satp）寄存器设置为0，这就是我们最终控制MMU的方式。由于我们还没有对虚拟内存的需求，我们用csrw（控制状态寄存器写入）写入0来把它禁用掉。一些板子的复位向量会将mhartid加载到该板子的a0寄存器中，但有些板子或许不会这么做，所以我选择从最可靠的地方来获取hart ID。

## 清除BSS

全局的、未初始化的变量的初值都是0，因为这些变量是在BSS段分配的。对于操作系统而言，我们要保证内存全是0。幸运的是，在我们的链接器脚本中定义了`_bss_start`和`_bss_end`这两个字段，分别告诉我们BSS部分的开始和结束位置。因此，我们在`.option pop`和`3`之间添加以下内容：

```nasm
# The BSS section is expected to be zero
    la      a0, _bss_start
    la      a1, _bss_end
    bgeu    a0, a1, 2f
1:
    sd      zero, (a0)
    addi    a0, a0, 8
    bltu    a0, a1, 1b
2:
```

在这里，我们使用sd（双字[64位]存储）将0存到内存地址a0，该地址逐渐向_bss_end移动。

### 进入Rust编程

由于许多人都不喜欢编写大量的汇编，我们尽快跳入Rust，虽然也会有人认为用Rust编程是最难的部分。我们的Rust不会那么难，不会一直和借用检查器(borrow checker)较劲。

为了进入Rust程序并使CPU处于一个确定的模式，我们将使用mret指令，这是一个陷入返回函数，允许我们将mstatus寄存器设置为我们的特权模式。因此，我们在boot.S中加入以下内容。

```nasm
# Control registers, set the stack, mstatus, mepc,
# and mtvec to return to the main function.
    # li    t5, 0xffff;
    # csrw  medeleg, t5
    # csrw  mideleg, t5
    la      sp, _stack
# We use mret here so that the mstatus register
# is properly updated.
    li      t0, (0b11 << 11) | (1 << 7) | (1 << 3)
    csrw    mstatus, t0
    la      t1, kmain
    csrw    mepc, t1
    la      t2, asm_trap_vector
    csrw    mtvec, t2
    li      t3, (1 << 3) | (1 << 7) | (1 << 11)
    csrw    mie, t3
    la      ra, 4f
    mret
4:
    wfi
    j   4b
```

这段代码很长，中间还有些注释。我们要做的是将[12:11]位设置为11，也就是 "机器模式(machine mode)"。这将使我们能够访问所有的指令和寄存器。当然，我们或许已经处于这一模式了，但还是做一次比较好。

\> 之后位[7]和位[3]将在总体上启用中断。然而，我们仍然需要通过`mie`（机器中断使能）寄存器启用特定的中断，这一点在最后几行实现。

mepc寄存器是 "机器异常程序计数器"，它是我们要返回的内存地址。符号`kmain`是在Rust中定义的，是我们离开汇编、进入Rust的“逃生”通道。

mtvec（机器陷入向量）是一个内核函数，每当有陷入出现时就会被调用，比如系统调用、非法指令或者时钟中断。

在我们完成Rust的主函数后，我们将ra（返回地址）置为等待状态。然后，`mret`指令将我们刚才所做的一切，通过mepc寄存器跳回，这就是我们最终进入Rust的地方!

(2019年9月29日添加）我们已经参考了`asm_trap_vector`，但我们还没有实际编写它。不过我们很快就会开工，现在先在 `src/asm/` 下创建一个名为 `trap.S` 的文件，并在其中添加以下内容。

```nasm
# trap.S
# Assembly-level trap handler.
.section .text
.global asm_trap_vector
asm_trap_vector:
    # We get here when the CPU is interrupted
    # for any reason.
    mret
```

## 裸机 Rust 的世界！

现在我们已经进入了rust。首先我们需要编辑 `lib.rs`，这个文件是由cargo命令为我们创建的。不要改变 lib.rs 的名称，否则 cargo 将永远不知道我们在干什么。lib.rs将是我们的入口点与我们用来导入其他Rust模块的工具。不要把`kmain`看成是一段执行代码。相反，它将初始化我们所需要的一切，然后引起 "大爆炸"，也就是说让所有代码都开始运行。操作系统主要是异步的，我们将使用时钟中断来指示内核开始运行，所以我们不能使用平时习惯的单线程编程方法。

当第一次打开`lib.rs`时，清空文件，因为里面没有我们需要用于内核的东西。我们需要自己定义一些东西来满足Rust的要求。由于我们不会使用标准库（标准库自己就依赖于内核，不能被用于我们的内核构建），我们必须首先定义`abort`和`panic_handler`再去做别的事情。就像这样：

```rust
// Steve Operating System
// Stephen Marz
// 21 Sep 2019
#![no_std]
#![feature(panic_info_message,asm)]

// ///////////////////////////////////
// / RUST MACROS
// ///////////////////////////////////
#[macro_export]
macro_rules! print
{
    ($($args:tt)+) => ({

    });
}
#[macro_export]
macro_rules! println
{
    () => ({
        print!("\r\n")
    });
    ($fmt:expr) => ({
        print!(concat!($fmt, "\r\n"))
    });
    ($fmt:expr, $($args:tt)+) => ({
        print!(concat!($fmt, "\r\n"), $($args)+)
    });
}

// ///////////////////////////////////
// / LANGUAGE STRUCTURES / FUNCTIONS
// ///////////////////////////////////
#[no_mangle]
extern "C" fn eh_personality() {}
#[panic_handler]
fn panic(info: &core::panic::PanicInfo) -> ! {
    print!("Aborting: ");
    if let Some(p) = info.location() {
        println!(
                    "line {}, file {}: {}",
                    p.line(),
                    p.file(),
                    info.message().unwrap()
        );
    }
    else {
        println!("no information available.");
    }
    abort();
}
#[no_mangle]
extern "C"
fn abort() -> ! {
    loop {
        unsafe {
            // The asm! syntax has changed in Rust.
            // For the old, you can use llvm_asm!, but the
            // new syntax kicks ass--when we actually get to use it.
            asm!("wfi");
        }
    }
}
```

我们使用`#![no_std]`来告诉Rust不会使用标准库。然后我们要求Rust允许我们的代码使用 panic 信息和内嵌式汇编功能。第一件要做的事是创建一个空的`eh_personality`函数。`#[no_mangle]`关闭了Rust的名称处理功能，所以这个符号确实是eh_personality。然后，`extern "C "`告诉Rust使用C风格的ABI。

之后，`#[panic_handler]`告诉Rust，我们定义的下一个函数将是我们的panic处理程序。Rust调用panic有几个原因，我们将用我们的断言隐式地调用它。我让这个函数做的是打印出产生panic的源文件和行号。虽说我们还没有实现`print！`或`println!`，但我们知道Rust中print和println的格式。顺便提一下，`->！`意味着这个函数不会返回。如果Rust检测到它可以返回，编译器会报错。

最后，我们编写`abort`函数。它要做的就是不断循环`wfi`（等待中断）指令。这使它正在运行的hart关闭，直到另一个中断发生。

### 我们真正进入了RUST!

我们已经正式进入rust，所以我们需要写出`boot.S`中指定的入口点，也就是`kmain`。所以，在我们的`lib.rs`代码中添加：

```rust
#[no_mangle]
extern "C"
fn kmain() {
    // Main should initialize all sub-systems and get
    // ready to start scheduling. The last thing this
    // should do is start the timer.
}
```

当kmain返回时，它碰到wfi循环并挂起。这是我们所期望的，因为我们还没有告诉内核要做什么。

那么基本已经完工了，我们进入了Rust之中。不幸的是，在我们实现`print！`之前不会看到任何打印到屏幕上的东西。但是，现在的代码至少应该是能够正常编译的！一般来说，优秀的作家会以一些引言或结束语来结束，但我并不是一位优秀的作家。

当你输入`make run`时，你的操作系统将尝试启动并进入Rust。然而由于我们还没有编写与操作系统通信的驱动程序，所以什么也不会发生。输入CTRL-A+'x'退出模拟器。同时，你也可以通过输入CTRL-A+'c'来查看你所在的位置。你现在是在QEMU的控制台。输入 "info registers "来查看模拟器在你的操作系统中的位置。

[第0章](ch0.md) → 第1章 → [第2章](ch2.md)
