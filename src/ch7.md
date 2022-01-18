# **The Adventures of OS**

使用Rust的RISC-V操作系统  
[**在Patreon上支持我!**](https://www.patreon.com/sgmarz)  [**操作系统博客**](http://osblog.stephenmarz.com/)  [**RSS订阅** ](http://osblog.stephenmarz.com/feed.rss)  [**Github** ](https://github.com/sgmarz)  [**EECS网站**](http://web.eecs.utk.edu/~smarz1)  
这是[用Rust编写RISC-V操作系统](http://osblog.stephenmarz.com/index.html)系列教程中的第7章。  
[目录](http://osblog.stephenmarz.com/index.html) → [第6章](http://osblog.stephenmarz.com/ch6.html) → (第7章) → [第8章](http://osblog.stephenmarz.com/ch8.html)

# 系统调用

**<span style='color:red'>2020年1月23日: 仅Patreon</span>**  
**<span style='color:red'>2020年1月29日: 公开</span>**

## 视频

[https://www.youtube.com/watch?v=6GW_jgkdGPw](https://www.youtube.com/watch?v=6GW_jgkdGPw)

## 概述

系统调用是非特权用户应用程序向内核请求服务的一种方式。 在 RISC-V 架构中，我们使用 `ecall` 指令调用，这将导致 CPU 停止它正在做的事情，提升特权模式，然后跳转到存储在 `mtvec`（机器陷入向量）寄存器中的任何函数处理程序，请记住这是处理所有陷入的“漏斗”，包括我们的系统调用。

我们必须设置处理系统调用的约定，可以使用已经存在的约定，这样可以与库进行交互，例如 newlib。 但是，让我们把它变成我们的！ 我们可以规定系统调用号是多少，以及当我们执行系统调用时它们将在哪里。

## 系统调用程序

我们通常只需要在处于较低权限模式时执行系统调用，如果我们在内核中，我们已经可以访问大多数特权系统，这使我们实际上不需要进入系统调用。

我们的系统调用是通过同步陷入#8 到达的，这是用户模式 ecall 的原因(cause)。 因此在我们的#8 处理程序中，我们将数据转发到我们的系统调用处理程序。 我们将完全使用 Rust，我们还需要能够操作程序计数器。 想想看，我们的`exit`系统调用必须能够移动到另一个进程，所以我们通过 `mepc`（机器异常程序计数器）寄存器来操作它。

## Rust 系统调用

与往常一样，请确保使用以下内容导入系统调用代码。

```rust

pub mod syscall;

```

这将进入您的 `lib.rs` 文件。

## 顺序与编号

一些库已经有它们希望您的系统调用保持的顺序，但是我们将为我们的简单应用程序创建自己的“C 库”，因此只要我们保持一致，就可以继续下去。

想想我们如何才能做到这一点。 每当我们执行 ecall 指令时，CPU 都会提升权限并跳转到陷阱向量，我们如何发送数据呢？ ARM 架构允许您将数字编码到他们的 `svc`（superviser 调用）指令中，然而许多操作系统放弃了这种实现，那么我们该怎么做呢？

答案是：寄存器。 我们在 RISC-V 架构中有大量的寄存器，所以我们几乎不像在 x86 时代那样受到限制。 对于我们的系统调用约定，我们将系统调用的编号放入第一个参数寄存器 `a0`，后续参数将进入 a1、a2、a3、...、a7，然后我们将使用相同的 a0 寄存器进行返回。

这与常规函数的调用约定相同，因此它将与我们已经知道的内容很好地交互。 在 RISC-V 中，我们可以使用伪指令 `call` 来进行正常的函数调用，或者使用 `ecall` 来进行 superviser 调用。 一致性是关键。Consistency is cey, or Konsistency is Key. Hmm.. Consistency is key isn't consistently sounding the 'k' :(.

## 实现系统调用

我们将同步原因(cause) #8 重定向到我们的系统调用 Rust 代码。

```rust

8 => {
  // 来自用户模式的环境（系统）调用
  println!("E-call from User mode! CPU#{} -> 0x{:08x}", hart, epc);
  return_pc = do_syscall(return_pc, frame);
},

```

大多数操作系统使用函数指针构建一个表，但我在这里使用 Rust 的 `match` 语句。 我没有进行任何性能计算，但我不认为一个比另一个有明显的优势。 再说一次，不要引用我的话，我还没有实际测试过。

如您所见，我们收到一个新的程序计数器，它是我们返回时要执行的指令的地址，系统调用函数必须至少将其加 4，因为 `ecall` 指令实际上是导致同步中断的原因，如果我们不移动程序计数器，我们就会一遍又一遍地执行 `ecall` 指令。 幸运的是，与 x86 不同的是，除了 16 位压缩指令外，所有指令都是 32 位的，但 `ecall` 始终是 32 位的，因为它没有压缩形式。

## 神说：要有系统调用！

让我们看一下 Rust 中的代码。

```rust

pub fn do_syscall(mepc: usize, frame: *mut TrapFrame) -> usize {
    let syscall_number;
    unsafe {
        // A0 是 X10，所以它的寄存器号是 10。
        syscall_number = (*frame).regs[10];
    }
    match syscall_number {
        0 => {
            // Exit
            println!("You called the exit system call!");
            mepc + 4
        },
      _ => {
            print!("Unknown syscall number {}", syscall_number);
            mepc + 4
        }
    }
}

```

我们首先需要的是 A0 的值，它是系统调用号。 由于它在陷入处理程序阶段存储在上下文中，我们可以直接从内存中检索它。 我们必须把它放在一个 `unsafe` 上下文中，因为我们正在取消引用一个原始 (raw) 指针，它可能是也可能不是准确的内存地址，由于 Rust 不能保证它是，我们需要把它放在一个不安全的块中。

这对非Rustaceans来说可能很有趣。

```rust

let syscall_number;
unsafe {
    // A0 是 X10，所以它的寄存器号是 10。
    syscall_number = (*frame).regs[10];
}

```

我正在创建一个名为 `syscall_number` 的变量，但由于在进入不安全块之前我无法获得它的值，所以它只是一个占位符。 事实上，在我们给它一个值之前，Rust 都不会给它一个类型。 你会注意到我没有对变量类型施加任何限制，所以我让 Rust 来决定。

我为什么这样做？ unsafe 块创建了一个新块，因此它创建了一个新范围 (scope)，但是我希望 `syscall_number` 在不安全上下文之外包含一个不可变值，这就是为什么我决定这样做。 从技术上讲，我可以使用 `let syscall_number: u64;` 来约束数据类型，但这不是必需的，因为只要我们将变量设置为等于某个值，Rust 就会评估数据类型。

## 现在我们干嘛？

我们将编写调度程序，以便我们的系统调用实际上可以做一些事情——是的，以技术上最准确的方式做事情！ 例如，我们可能需要将进程推迟到一定时间后（有点像 `sleep()` 的工作方式），或者我们需要关闭进程（很像 `exit()` 的工作方式），写点东西到控制台怎么样——是的，我们也需要那个。

因此，接下来我们将添加进程和必要的系统调用，我们没有做出任何未来的预测，这可能很危险，但我们正在实现我们的操作系统，因为我们发现了一个不可行的解决方案。 希望这将使您了解操作系统怎样实现为所有应用程序的一切！

[目录](http://osblog.stephenmarz.com/index.html) → [第6章](http://osblog.stephenmarz.com/ch6.html) → (第7章) → [第8章](http://osblog.stephenmarz.com/ch8.html)
