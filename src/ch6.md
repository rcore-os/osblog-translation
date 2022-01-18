# **The Adventures of OS**

使用Rust的RISC-V操作系统  
[**在Patreon上支持我!**](https://www.patreon.com/sgmarz)  [**操作系统博客**](http://osblog.stephenmarz.com/)  [**RSS订阅** ](http://osblog.stephenmarz.com/feed.rss)  [**Github** ](https://github.com/sgmarz)  [**EECS网站**](http://web.eecs.utk.edu/~smarz1)  
这是[用Rust编写RISC-V操作系统](http://osblog.stephenmarz.com/index.html)系列教程中的第6章。  
[目录](http://osblog.stephenmarz.com/index.html) → [第5章](http://osblog.stephenmarz.com/ch5.html) → (第6章) → [第7章](http://osblog.stephenmarz.com/ch7.html)

# 进程的内存

**<span style='color:red'>2019年12月08日: 仅Patreon</span>**  
**<span style='color:red'>2019年12月23日: 公开</span>**

## 概述

进程是操作系统的重点。 我们想开始“做点事”，我们将把它融入一个进程并让它能够运行。 我们将在未来随着向进程添加功能来更新进程结构体，现在我们需要一个程序计数器（正在执行的指令，`pc`）和一个用于本地内存的栈。

我们不会为进程创建标准库。 在本章中，我们将编写内核函数并将它们包装到一个进程中。 当我们开始创建我们的用户进程时，我们需要从块设备中读取并开始执行指令。 这是一种相当不错的方式，因为我们将需要系统调用等等。

## 进程结构体

每个进程都必须包含一些信息，以便我们可以运行它。 我们需要知道一些信息，例如所有的寄存器、MMU 表和进程的状态。

这个进程结构体尚未完成，我曾想过现在就完成，但我认为逐步完成它会帮助我们理解需要什么以及为什么该部分需要存储在进程结构体中。

```rust

pub enum ProcessState {
  Running,
  Sleeping,
  Waiting,
  Dead,
}

#[repr(C)]
pub struct Process {
  frame:           TrapFrame,
  stack:           *mut u8,
  program_counter: usize,
  pid:             u16,
  root:            *mut Table,
  state:           ProcessState,
}

```

该进程包含一个栈(`stack`)，我们将把它提供给 `sp`（栈指针）寄存器，但是这个`stack`显示的是栈的顶部，所以我们需要添加大小(`size`)。 请记住，一个栈是从高内存增长到低内存的（用减法分配并用加法释放）。

程序计数器(`program_counter`)将包含下一个要执行的指令的地址，我们可以通过`mepc`寄存器（机器异常程序计数器）得到它。 当我们使用上下文切换定时器或非法指令亦或是作为对 UART 输入的响应来中断我们的进程时，CPU 会将即将执行的指令放入 `mepc`，我们可以通过返回这里重新开始我们的过程并再次开始执行指令。

## 陷入帧(Trap Frame)

我们目前使用的是单处理器系统，我们以后会把它升级成一个多 hart 系统，但对现在来说就这样会更有意义。 陷入帧允许我们在通过内核处理进程时冻结进程，这在来回切换时是必须的，你能想象你刚刚设置了寄存器它就被内核修改了吗？

进程由其上下文定义。 `TrapFrame` 显示了上下文在 CPU 上的样子：(1) 通用寄存器（有 32 个），(2) 浮点寄存器（也有 32 个），(3) MMU，(4) 一个栈 用来处理这个进程的中断上下文。 我还存储了 hartid，这样我们就不会同时在两个 hart 上运行相同的进程。

```rust

#[repr(C)]
#[derive(Clone, Copy)]
pub struct TrapFrame {
  pub regs:       [usize; 32], // 0 - 255
  pub fregs:      [usize; 32], // 256 - 511
  pub satp:       usize,       // 512 - 519
  pub trap_stack: *mut u8,     // 520
  pub hartid:     usize,       // 528
}

```

## 创建一个进程

我们需要为进程的栈和进程结构体本身分配内存。 我们将使用 MMU，因此我们可以为所有进程提供一个已知的起始地址，我选择了 0x2000_0000。 当我们创建外部进程并编译它们时，我们需要知道起始内存地址，由于这是虚拟内存，它本质上可以是我们想要的任何值。

```rust

impl Process {
  pub fn new_default(func: fn()) -> Self {
    let func_addr = func as usize;
    // 当我们开始进行多 hart 处理时，
    // 我们会将下面的 NEXT_PID 转换为一个原子(atomic)增量
    // 现在我们需要一个进程，让它可以工作，之后再改进它！
    let mut ret_proc =
      Process { frame:           TrapFrame::zero(),
                stack:           alloc(STACK_PAGES),
                program_counter: PROCESS_STARTING_ADDR,
                pid:             unsafe { NEXT_PID },
                root:            zalloc(1) as *mut Table,
                state:           ProcessState::Waiting,
                data:            ProcessData::zero(), };
    unsafe {
      NEXT_PID += 1;
    }
    // 现在我们将栈指针移动到分配的底部。
    // 规范显示寄存器 x2 (2) 是栈指针。
    // 我们可以使用 ret_proc.stack.add，
    // 但这是一个不安全的函数，需要一个不安全块。
    // 所以，先转换成usize再加上PAGE_SIZE比较好。
    // 我们还需要设置栈调整，使其位于内存的底部并且远离堆分配。
    ret_proc.frame.regs[2] = STACK_ADDR + PAGE_SIZE * STACK_PAGES;
    // 在 MMU 上映射栈
    let pt;
    unsafe {
      pt = &mut *ret_proc.root;
    }
    let saddr = ret_proc.stack as usize;
    // 我们需要将栈映射到用户进程的虚拟内存。
    // 这有点麻烦，因为我们还需要映射函数代码。
    for i in 0..STACK_PAGES {
      let addr = i * PAGE_SIZE;
      map(
          pt,
          STACK_ADDR + addr,
          saddr + addr,
          EntryBits::UserReadWrite.val(),
          0,
      );
    }
    // 在 MMU 上映射程序计数器
    map(
        pt,
        PROCESS_STARTING_ADDR,
        func_addr,
        EntryBits::UserReadExecute.val(),
        0,
    );
    map(
        pt,
        PROCESS_STARTING_ADDR + 0x1001,
        func_addr + 0x1001,
        EntryBits::UserReadExecute.val(),
        0,
    );
    ret_proc
  }
}

impl Drop for Process {
  // 由于我们将 Process 的所有权存储在链表中，
  // 我们可以使其在被删除时自动释放。
  fn drop(&mut self) {
    // 我们将栈分配为一个页面。
    dealloc(self.stack);
    // 这是不安全的，但它处于丢弃(drop)阶段，所以我们不会再使用它。
    unsafe {
      // 请记住，unmap会取消映射除根以外的所有级别的页表。
      // 它还释放与表关联的内存。
      unmap(&mut *self.root);
    }
    dealloc(self.root as *mut u8);
  }
}

```

现在`Process` 结构的实现相当简单，Rust 要求我们对所有字段都有一些值，所以我们创建这些辅助函数来做到这一点。

当我们创建一个进程时，我们并没有真正“创建”它，相反，我们分配内存来保存进程的元数据。 目前，所有进程代码都存储为一个 Rust 函数，之后我们将从块设备加载二进制指令并以这种方式执行。 请注意，我们需要做的就是创建一个栈。 我为一个栈分配了 2 个页面，给了栈 4096 * 2 = 8192 字节的内存，这很小，但对于我们正在做的事情来说应该绰绰有余。

## CPU 级别的进程

每当我们遇到trap时，都会处理 CPU 级别的进程。 我们将使用 CLINT 计时器作为我们的上下文切换计时器。 现在我已经为每个进程分配了 1 整秒的时间，这对于正常操作来说太慢了，但它使调试变得更容易，因为我们可以看到一个进程执行的步骤。

每次 CLINT 计时器命中时，我们都将存储当前上下文，跳转到 Rust（通过 trap.rs 中的 m_trap），然后处理需要做的事情。 我选择使用机器模式来处理中断，因为我们不必担心切换 MMU（它在机器模式下关闭）或遇到递归错误。

CLINT 定时器通过 MMIO 控制如下：

```rust

unsafe {
  let mtimecmp = 0x0200_4000 as *mut u64;
  let mtime = 0x0200_bff8 as *const u64;
  // QEMU 给出的频率是 10_000_000 Hz，
  // 因此这会将下一个中断设置为从现在开始一秒后触发。
  mtimecmp.write_volatile(mtime.read_volatile() + 10_000_000);
}

```

`mtimecmp` 是存储未来时间的寄存器，当 `mtime` 达到这个值时，它会以“机器定时器中断”的原因中断 CPU，我们可以使用这个周期性计时器来中断我们的进程并切换到另一个。 这是一种称为“时间切片”的技术，我们将 CPU 时间的切片分配给每个进程。 定时器中断的速度越快，我们在给定时间内可以处理的进程就越多，但是每个进程只能执行一小部分指令。 此外，每个中断都会执行上下文切换（m_trap_handler）代码。

定时器频率有点像一门艺术，Linux 在将其设定为 1,000Hz（每秒一千次中断）时深有体会，这意味着每个进程能够运行 1/1000 秒。 诚然，以当今处理器的速度，这可以执行大量的指令，但在过去，这是有争议的。

## 带陷入帧(Trap Frame)的陷入处理程序(Trap Handler)

我们的陷入处理程序必须能够处理不同的陷入帧，因为当我们需要命中陷入处理程序时，任何进程（甚至内核）都可能在运行。

```armasm

m_trap_vector:
# 这里所有的寄存器都是易失的，我们需要在做任何事情之前保存它们。
csrrw	t6, mscratch, t6
# csrrw 将自动将 t6 到 mscratch 中并将 mscratch 的旧值交换为 t6。
# 这很好，因为我们只是切换了值并且没有破坏任何东西——所有的都是原子的！
# 在 cpu.rs 中我们有一个结构：
#  32 gp regs		0
#  32 fp regs		256
#  SATP register	512
#  Trap stack       520
#  CPU HARTID		528
# 我们使用 t6 作为临时寄存器，因为它是最底层的寄存器 (x31)
.set 	i, 1
.rept	30
  save_gp	%i
  .set	i, i+1
.endr

# 保存实际的 t6 寄存器，我们将其交换到 mscratch
mv		t5, t6
csrr	t6, mscratch
save_gp 31, t5

# 将内核陷入帧(kernel trap frame)恢复到 mscratch
csrw	mscratch, t5

# 准备好进入 Rust (trap.rs)
# 我们不想写入用户栈或任何在这有关的东西。
csrr	a0, mepc
csrr	a1, mtval
csrr	a2, mcause
csrr	a3, mhartid
csrr	a4, mstatus
mv		a5, t5
ld		sp, 520(a5)
call	m_trap

# 当我们到达这里时，我们已经从 m_trap 返回，恢复寄存器并返回。
# m_trap 将通过 a0 返回返回地址。

csrw	mepc, a0

# 现在将陷入帧加载回 t6
csrr	t6, mscratch

# 恢复所有通用寄存器
.set	i, 1
.rept	31
  load_gp %i
  .set	i, i+1
.endr

# 由于我们从 i = 1 开始运行此循环 31 次，
# 因此最后一个循环将 t6 加载回其原始值。

mret  

```

我们必须手动操作 `TrapFrame` 字段，因此了解偏移量很重要。 每当我们更改 `TrapFrame` 结构时，我们都需要确保我们的汇编代码反映了这种更改。

在汇编代码中，我们可以看到我们做的第一件事就是冻结当前正在运行的进程。 我们使用 `mscratch` 寄存器来保存当前正在执行的进程（或内核——它使用 KERNEL_TRAP_FRAME）的 TrapFrame，`csrrw` 指令将自动存储 `t6` 寄存器的值并将旧值（陷入帧的内存地址）返回到 t6。 请记住，我们必须保持所有寄存器的原始值，这是一个很好的方法！

出于方便，我们使用 t6 寄存器。 它是 32 号寄存器（索引 31），所以它是我们循环保存或恢复寄存器时的最后一个寄存器。

如上所示，我使用 GNU 的 .altmacro 的宏循环来保存和恢复寄存器，很多时候您会看到所有 32 个寄存器都被保存或恢复，但我这样做显著地简化了代码。

``` armasm

csrr	a0, mepc
csrr	a1, mtval
csrr	a2, mcause
csrr	a3, mhartid
csrr	a4, mstatus
mv		a5, t5
ld		sp, 520(a5)
call	m_trap

```

这部分陷入处理程序用于将参数转发给 rust 函数 `m_trap`，如下所示：

```rust

extern "C" fn m_trap(epc: usize,
    tval: usize,
    cause: usize,
    hart: usize,
    status: usize,
    frame: *mut TrapFrame) -> usize

```

按照 ABI 约定，a0 是 epc，a1 是 tval，a2 是 cause，a3 是 hart，a4 是 status，a5 是指向陷入帧的指针。 您还会注意到我们返回了一个 usize，这用于返回要执行的下一条指令的内存地址。 请注意，调用 `m_trap` 之后的下一条指令是将 a0 寄存器加载到 mepc 中。 同样按照 ABI 约定，Rust 的返回值存储在 a0 中。

## 第一个进程——Init

我们的调度算法（稍后）将始终需要至少一个进程，就像 Linux 一样，我们将把这个进程称为 `init`。

本章的重点是展示一个进程在抽象视图中的样子，我将通过添加调度程序并显示实际运行的进程来在本章基础上进一步展示，这将是我们操作系统的基础。 想要让进程实际执行操作，我们需要做的另一件事是实现系统调用，因此您可以期待它！

[目录](http://osblog.stephenmarz.com/index.html) → [第5章](http://osblog.stephenmarz.com/ch5.html) → (第6章) → [第7章](http://osblog.stephenmarz.com/ch7.html)
