# **The Adventures of OS**

使用Rust的RISC-V操作系统
[**在Patreon上支持我!**](https://www.patreon.com/sgmarz)  [**操作系统博客**](http://osblog.stephenmarz.com/)  [**RSS订阅** ](http://osblog.stephenmarz.com/feed.rss)  [**Github** ](https://github.com/sgmarz)  [**EECS网站**](http://web.eecs.utk.edu/~smarz1)
这是[用Rust编写RISC-V操作系统](http://osblog.stephenmarz.com/index.html)系列教程中的第5章。
[目录](http://osblog.stephenmarz.com/index.html) → [第4章](http://osblog.stephenmarz.com/ch4.html) → (第5章) → [第6章](http://osblog.stephenmarz.com/ch6.html)

# 外部中断

**<span style='color:red'>2019年11月18日: 仅Patreon</span>**  
**<span style='color:red'>2019年11月25日: 公开</span>**

## 视频

[https://www.youtube.com/watch?v=99KMubPgDIU](https://www.youtube.com/watch?v=99KMubPgDIU)

## 概述

在上一章，我们讨论了CPU以及内核的中断。 在这一章，我们将讨论中断的其中一类——外部中断，这些中断表示发生了某些外部或平台中断。 例如，UART设备可能刚刚填满了它的缓冲区。

## 平台级中断控制器(The Platform-Level Interrupt Controller)

平台级中断控制器 (PLIC) 通过CPU上的一个引脚——EI (外部中断) 引脚，来路由所有信号，通过 `mie` 寄存器中的机器外部中断启用 (`meie`) 位可以启用该引脚。

每当我们看到该引脚已被触发（外部中断待处理）时，我们就可以查询 PLIC 以查看是什么原因造成的。 此外，我们可以将 PLIC 配置为优先考虑中断源或完全禁用某些源，同时启用其他源。

## PLIC 寄存器

PLIC 是一个通过 MMIO 控制的中断控制器。 有几个与 PLIC 相关的寄存器：

```
寄存器              地址            描述
Priority           0x0c00_0000    设置特定中断源的优先级
Pending            0x0c00_1000    包含已触发的中断列表(待处理)
Enable             0x0c00_2000    启用/禁用某些中断源
Threshold          0x0c20_0000    设置中断触发的阈值
Claim(read)        0x0c20_0004    按优先级顺序返回下一个中断
Complete(write)    0x0c20_0004    完成对特定中断的处理
```

## PLIC 如何工作

PLIC 通过外部中断引脚连接到 CPU。 以下架构图来自[SiFive's Freedom Unleashed Manual](https://sifive.cdn.prismic.io/sifive%2F834354f0-08e6-423c-bf1f-0cb58ef14061_fu540-c000-v1.0.pdf)。

<div align=center>

![plic_cpu](assets/5/plic_cpu.png)

</div>

PLIC 连接到外部设备，并通过位于 PLIC 基地址（如上表所示的寄存器）的可编程接口控制它们的中断。 这意味着我们作为程序员可以控制每个中断的优先级，我们是否看到它，我们是否处理过它，等等。

上图可能有点混乱，但在 QEMU 中编程的 PLIC 要简单得多。

## PLIC 处理

系统将通过电线将 PLIC 连接到外部设备，我们通常会在技术参考文档或类似文件中获得线号。 但是对于我们来说，我在 `qemu/include/hw/riscv/virt.h` 中查看到 UART 连接到引脚 10，VIRTIO 设备是 1 到 8，PCI express 设备是 32 到 35。 你可能在想为什么在我们的 Rust 代码中，我们只有一个中断启用寄存器——因为我们不会为了我们的目的而超过中断 10 (UART)。

## 启用中断源

现在知道了设备通过哪个中断进行连接，我们通过将 `1 << id` 写入中断启用寄存器来启用该中断。 此示例的中断 ID 将为 UART 的 10。

```rust

/// 启用给定的中断 id
pub fn enable(id: u32) {
    let enables = PLIC_INT_ENABLE as *mut u32;
    let actual_id = 1 << id;
    unsafe {
        // 与 complete 和 claim 寄存器不同，plic_int_enable
        // 寄存器是一个位集(bitset)，其中 id 是位索引。
        // 该寄存器是一个 32 位寄存器，因此我们可以启用
        // 中断 31 到 1（0 硬连线到 0）。
        enables.write_volatile(enables.read_volatile() | actual_id);
    }
}

```

## 设置中断源优先级

现在已经启用了中断源，我们需要给它一个 0 到 7 的优先级。7 是最高优先级，0 是“scum”类（h/t Top Gear）——但是优先级 0 不能满足任何阈值，因此它基本上禁用了中断（请参阅下面的 PLIC 阈值）。 我们可以通过优先级寄存器设置每个中断源的优先级。

```rust

/// 将给定中断的优先级设置为给定优先级。
/// 优先级必须为 [0..7]
pub fn set_priority(id: u32, prio: u8) {
    let actual_prio = prio as u32 & 7;
    let prio_reg = PLIC_PRIORITY as *mut u32;
    unsafe {
        // 中断 id 的偏移量是：
        // PLIC_PRIORITY + 4 * id
        // 由于我们在 u32 类型上使用指针运算
        // 它会自动将 id 乘以 4。
        prio_reg.add(id as usize).write_volatile(actual_prio);
    }
}

```

## 设置 PLIC 阈值

PLIC 本身有一个全局阈值，所有中断在“启用”之前都必须通过该阈值。 这是通过阈值寄存器控制的，我们可以将值 0 到 7 写入其中。 任何小于或等于此阈值的中断优先级都无法满足障碍并被屏蔽，这实际上意味着中断被禁用。

```rust

/// 设置全局阈值，阈值可以是值 [0..7]。
/// PLIC 将屏蔽等于或低于给定阈值的任何中断。
/// 这意味着阈值 7 将屏蔽所有中断，阈值 0 将允许所有中断。
pub fn set_threshold(tsh: u8) {
    // 我们使用 tsh 是因为我们使用的是 u8
    // 但我们的最大数量是 3 位 0b111。
    // 所以，我们与 7 (0b111) 求 and 来获取最后三位。
    let actual_tsh = tsh & 7;
    let tsh_reg = PLIC_THRESHOLD as *mut u32;
    unsafe {
        tsh_reg.write_volatile(actual_tsh as u32);
    }
}

```

## 处理 PLIC 中断

PLIC 将通过异步原因(`asynchronous cause`) 11（机器外部中断）向我们的操作系统发出信号。 当处理这个中断时，我们不会知道实际上是什么导致了中断——只知道 PLIC 导致了它。 这是声明(`claim`)/完成(`complete`)过程开始的地方。

## 声明一个中断

SiFive 将声明/完成过程描述如下：

> ## 10.7 中断声明过程
> FU540-C000 hart 可以通过读取`claim/complete`寄存器（表 45）来执行中断声明，该寄存器返回最高优先级的挂起中断的 ID，如果没有挂起的中断，则返回零。 成功的声明还会原子地清除中断源上相应的pending位。
>
> FU540-C000 hart 可以随时执行声明，即使其 `mip`（表 22）寄存器中的 MEIP 位未设置。
>
> 声明操作不受优先级阈值寄存器设置的影响。
>
> ## 10.8 中断完成
> FU540-C000 hart 通过将其从声明中接收到的中断 ID 写入`claim/complete`寄存器（表 45）来发出它已完成执行中断处理程序的信号。 PLIC 不检查完成 ID 是否与该目标的最后一个声明 ID 相同。 如果完成 ID 与当前为目标启用的中断源不匹配，则g该完成会被静默忽略。

`claim`寄存器将返回按优先级排序的下一个中断。 如果寄存器返回 0，则没有挂起的中断，这不应该发生，因为我们将通过中断处理程序处理声明过程。

```rust

/// 获取下一个可用中断，这就是“claim声明”过程。
// plic 会自动按优先级排序，并将中断的 ID 交给我们。
// 例如，如果 UART 正在中断并且为下一个，我们将得到值 10。
pub fn next() -> Option {
    let claim_reg = PLIC_CLAIM as *const u32;
    let claim_no;
    // 声明寄存器填充了最高优先级的已启用中断。
    unsafe {
        claim_no = claim_reg.read_volatile();
    }
    if claim_no == 0 {
        // 中断 0 硬连线到 0，这告诉我们没有要声明的中断
        // 因此我们返回 None。
        None
    }
    else {
        // 如果到达这里，我们就会得到一个非 0 中断。
        Some(claim_no)
    }
}

```

然后，我们可以要求 PLIC 通过该声明寄存器向我们提供任何发起中断的编号。 在 Rust 中，我通过匹配将其驱动到正确的处理程序。 到目前为止，我们直接在中断上下文中处理它——这是一个坏主意，但我们还没有任何延迟任务系统。

```rust

  // 机器外部 (来自 PLIC 的中断)
  // println!("Machine external interrupt CPU#{}", hart);
  // 我们将检查下一个中断。 如果中断不可用，则将给我们 None。
  // 然而，这意味着我们得到了一个虚假的中断，除非
  // 我们得到一个来自非 PLIC 源的中断。 这是 PLIC 将 id 0 硬连线
  // 到 0 的主要原因，因此我们可以将其用作错误案例。
  if let Some(interrupt) = plic::next() {
    // 如果我们到达这里，我们就会从声明寄存器中得到一个中断。
    // PLIC 将自动为下一个中断设置优先级，因此当我们从声明中获取它时，
    // 它将是优先级顺序中的下一个。
    match interrupt {
      10 => { // 中断 10 是 UART 中断。
        // 我们通常会将其设置为在中断上下文之外进行处理，
        // 但我们在这里进行测试！ 来吧！
        // 我们还没有为 my_uart 使用单例模式，但请记住，
        // 这只是简单地包装(wrap)了 0x1000_0000 (UART)。
        let mut my_uart = uart::Uart::new(0x1000_0000);
        // 如果我们到了这里，UART 最好有一些东西！ 如果不是，会发生什么？？
        if let Some(c) = my_uart.get() {
          // 如果您认识这段代码，它曾经位于 kmain() 下的 lib.rs 中。
          // 那是因为我们需要轮询 UART 数据。
          // 既然现在我们有中断了，那就来吧！
          match c {
            8 => {
              // 这是一个退格，所以我们基本上必须写一个空格，再写一个退格：
              print!("{} {}", 8 as char, 8 as char);
            },
            10 | 13 => {
              // 换行或回车
              println!();
            },
            _ => {
              print!("{}", c as char);
            },
          }
        }

      },
      // 非UART中断在这里并且什么也不做。
      _ => {
        println!("Non-UART external interrupt: {}", interrupt);
      }
    }
    // 我们已经声明了它，所以现在假设我们已经处理了它。
    // 这将重置挂起的中断并允许 UART 再次中断。 否则，UART 将“卡住”。
    plic::complete(interrupt);
  }

```

## 告诉 PLIC 我们需要擦除(wiping)

当我们声明一个中断时，我们是在告诉 PLIC 它将被处理或正在被处理。 在此期间，PLIC 不会再监听来自同一设备的任何中断。 这是 `complete` 过程的开始。 当我们写入（而不是读取）声明寄存器时，我们给出一个中断的值来告诉 PLIC 我们已经完成了它给我们的中断。 然后，PLIC 可以重置中断触发器并等待该设备将来再次中断。 这会重置系统并让我们一遍又一遍地循环回到声明/完成。

```rust

// 通过 id 完成一个挂起的中断。 id 应该来自上面的 next() 函数。
pub fn complete(id: u32) {
    let complete_reg = PLIC_CLAIM as *mut u32;
    unsafe {
        // 我们实际上将一个 u32 写入整个 complete_register。
        // 这与声明寄存器是同一个寄存器，
        // 但它可以根据我们是在读还是在写来区分。
        complete_reg.write_volatile(id);
    }
}

```

就是这样。 PLIC 已被编程，现在它只处理我们的 UART。 请注意在 `lib.rs` 中我删除了 UART 轮询代码。 现在有了中断，我们只能在它向我们发出信号时处理 UART。 由于我们使用等待中断 (wfi) 将 HART 置于等待循环中，我们可以节省电力和 CPU 周期。

让我们将它添加到我们的 `kmain` 函数中，看看会发生什么！

```rust

// 让我们通过 PLIC 设置中断系统。
// 我们必须将阈值设置为不会屏蔽所有中断的值。
println!("Setting up interrupts and PLIC...");
// 我们降低了阈值墙，这样我们的中断就可以跃过它。
plic::set_threshold(0);
// VIRTIO = [1..8]
// UART0 = 10
// PCIE = [32..35]
// 启用 UART 中断。
plic::enable(10);
plic::set_priority(10, 1);
println!("UART interrupts have been enabled and are awaiting your command");

```

当我们运行(`make run`)这段代码时，我们得到以下信息。 我输入了底线(bottom line)，但它表明我们现在正在通过 PLIC 处理中断！

<div align=center>

![plic_works](assets/5/plic_works.png)

</div>

[目录](http://osblog.stephenmarz.com/index.html) → [第4章](http://osblog.stephenmarz.com/ch4.html) → (第5章) → [第6章](http://osblog.stephenmarz.com/ch6.html)
