# 页粒度的内存分配

这是[用 Rust 编写 RISC-V 操作系统](http://osblog.stephenmarz.com/index.html)系列教程中的第 3.1 章。

[目录](index.md) → [第 2 章](ch2.md) → 第 3.1 章 → [第 3.2 章](ch3.2.md)

**<span style='color:red'>2019 年 10 月 15 日：仅Patreon</span>**

**<span style='color:red'>2019 年 10 月 22 日：公开</span>**

## 视频

本章博文的视频在 YouTube 上：[https://www.youtube.com/watch?v=SZ6GMxLkbx0](https://www.youtube.com/watch?v=SZ6GMxLkbx0)。

## 概述

当我们启动时，`.elf` 文件内的所有内容都会自动加载到内存中。在代码中可见，我们为整个系统分配了 128 兆字节空间。因此，我们需要处理除去所有声明的全局变量和代码之外的空间。

在管理内存时，我们本质上需要处理三个部分：1）页粒度的分配，2）字节粒度的分配，以及 3）内存管理单元的编程。

## 以页为单位的分配

以页为单位的分配意味着每一次分配所提供的内存大小是一个页。在大多数架构中，最小的页大小是 4096 字节。RISC-V 也不例外。我们将使用 SV39 MMU 分页系统，这些将在第 3 部分中深入讨论。

我们有多种方法可以一次分配页面。由于我们知道每一次分配的大小，分配方案就很直白了。许多操作系统使用一个链表进行分配，其中指向链表头部的指针是第一个未分配的页面，然后每个未分配的页面用 8 个字节来存储下一个指针，指向下一个页面的内存地址。这些字节是存储在页面本身之中的，所以不需要额外的内存的开销。

这个系统的缺点是，我们必须跟踪每一个被分配的页面，因为一个指针就是一个页面。

```rust
struct FreePages {
    struct FreePages *next;
};
```

## 描述符的分配

我们的操作系统将使用描述符。我们可以分配连续的页面，而只存储链表头的指针。然而要做到这一点，我们需要为每个 4096 字节大小的页额外分配一个字节。这一个字节将包含两个标志：1）这个页面是否被占用？ 2）这是不是连续分配的最后一个页面？

```rust
// These are the page flags, we represent this
// as a u8, since the Page stores this flag.
#[repr(u8)]
pub enum PageBits {
    Empty = 0,
    Taken = 1 << 0,
    Last = 1 << 1,
}

pub struct Page {
    flags: u8,
}
```

在这里，我们需要为每个页面分配一个 `Page` 结构体。幸运的是，通过升级过的链接器脚本，我们能够获得堆的地址和大小。

```linker
/*
Finally, our heap starts right after the kernel stack. This heap will be used mainly
to dole out memory for user-space applications. However, in some circumstances, it will
be used for kernel memory as well.

We don't align here because we let the kernel determine how it wants to do this.
*/
PROVIDE(_heap_start = _stack_end);
PROVIDE(_heap_size = _memory_end - _heap_start);
```

我们可以简单的地计算出堆中的总页数：`let num_pages = HEAP_SIZE / PAGE_SIZE;`，其中 `HEAP_SIZE` 是一个全局符号，值为 `_heap_size`，页大小是一个常数为 4096。最终我们必须分配 `num_pages` 数量的 `Page` 描述符来存储分配信息。

## 页面分配

分配是通过首先在描述符链表中搜索一个连续、空闲的页面块来完成的。我们接收想要分配的页数量作为 `alloc` 的参数。如果我们找到一段空闲的页，我们就把所有这些 `Page` 的描述符都设置为已占用，然后将最后一个页的"Last"位置位。这可以帮助我们在释放分配的页面时，跟踪最后一个页面的位置。

```rust
pub fn alloc(pages: usize) -> *mut u8 {
    // We have to find a contiguous allocation of pages
    assert!(pages > 0);
    unsafe {
        // We create a Page structure for each page on the heap. We
        // actually might have more since HEAP_SIZE moves and so does
        // the size of our structure, but we'll only waste a few bytes.
        let num_pages = HEAP_SIZE / PAGE_SIZE;
        let ptr = HEAP_START as *mut Page;
        for i in 0..num_pages - pages {
            let mut found = false;
            // Check to see if this Page is free. If so, we have our
            // first candidate memory address.
            if (*ptr.add(i)).is_free() {
                // It was FREE! Yay!
                found = true;
                for j in i..i + pages {
                    // Now check to see if we have a
                    // contiguous allocation for all of the
                    // request pages. If not, we should
                    // check somewhere else.
                    if (*ptr.add(j)).is_taken() {
                        found = false;
                        break;
                    }
                }
            }
    // ........................................
```

我们可以简单算出哪个描述符指向哪个页面，因为第一个描述符就是第一页，第二个描述符是第二页，...，第 n 个描述符是第 n 页。这种一对一的关系简化了数学运算：`ALLOC_START + PAGE_SIZE * i` 。在这个公式中，`ALLOC_START` 是第一个可用页面的地址，`PAGE_SIZE` 是常数 4096，而 `i` 是指第 i 个页面描述符（从0开始）。

## 页面释放

首先我们必须把上面的等式倒过来，计算出这个页面对应的是哪个描述符。我们可以使用这个式子：`HEAP_START + (ptr - ALLOC_START) / PAGE_SIZE`。我们接收一个指向可使用页面的指针 `ptr`。我们从该指针中减去可使用的堆顶部地址，以获得该 ptr 的绝对位置。由于每次分配都正好是 4096 字节，我们再除去页面大小来得到描述符下标。然后我们再加上 `HEAP_START` 变量，这样就得到了描述符的位置。

```rust
pub fn dealloc(ptr: *mut u8) {
    // Make sure we don't try to free a null pointer.
    assert!(!ptr.is_null());
    unsafe {
        let addr =
            HEAP_START + (ptr as usize - ALLOC_START) / PAGE_SIZE;
        // Make sure that the address makes sense. The address we
        // calculate here is the page structure, not the HEAP address!
        assert!(addr >= HEAP_START && addr < HEAP_START + HEAP_SIZE);
        let mut p = addr as *mut Page;
        // Keep clearing pages until we hit the last page.
        while (*p).is_taken() && !(*p).is_last() {
            (*p).clear();
            p = p.add(1);
        }
        // If the following assertion fails, it is most likely
        // caused by a double-free.
        assert!(
                (*p).is_last() == true,
                "Possible double-free detected! (Not taken found \
                    before last)"
        );
        // If we get here, we've taken care of all previous pages and
        // we are on the last page.
        (*p).clear();
    }
}
```

在这段代码中，我们可以看到，我们一直释放空间直到最后一个被分配的页。如果我们在到最后一个分配的页面之前遇到了一个没有被占用的页面，那么我们的堆就乱套了。通常情况下，当一个指针被释放超过一次时就会发生这种情况（常称为双重释放）。在这种情况下我们可以添加一个 `assert!` 语句来帮助捕捉这些类型的问题，但我们必须使用“可能”一词，因为我们不能确定为什么在出现非占用页之前没有遇到最后一个被分配的页。

## 分配清零的页面

这些 4096 字节的页将分配给内核和用户应用使用。注意当我们释放一个页面时，我们并不清除内存内容，相反我们只清除描述符。在上述代码中这表现为 `(*p).clear()`。这段代码只清除了描述符对应的位，而不是整个页的 4096 字节。

安全起见，我们将创建一个辅助函数，分配并清除这些页面。

```rust
/// Allocate and zero a page or multiple pages
/// pages: the number of pages to allocate
/// Each page is PAGE_SIZE which is calculated as 1 << PAGE_ORDER
/// On RISC-V, this typically will be 4,096 bytes.
pub fn zalloc(pages: usize) -> *mut u8 {
    // Allocate and zero a page.
    // First, let's get the allocation
    let ret = alloc(pages);
    if !ret.is_null() {
        let size = (PAGE_SIZE * pages) / 8;
        let big_ptr = ret as *mut u64;
        for i in 0..size {
            // We use big_ptr so that we can force an
            // sd (store doubleword) instruction rather than
            // the sb. This means 8x fewer stores than before.
            // Typically we have to be concerned about remaining
            // bytes, but fortunately 4096 % 8 = 0, so we
            // won't have any remaining bytes.
            unsafe {
                (*big_ptr.add(i)) = 0;
            }
        }
    }
}
```

我们简单地调用 `alloc` 来完成那些繁重的工作——分配页面描述符并设置标志位。然后我们进入 `zalloc` 的后半部分，即将页面清零。幸运的是，我们可以使用 8 字节的指针来一次性存储 8 字节的 0 ，因为 `4096 / 8 = 512`，而 512 是一个整数。因此，8 字节的指针只需 512 轮循环即可完全覆盖一个 4096 字节的页面。

## 测试

我创建了一个函数，通过查看所有占用的描述符来打印出页分配表。

```rust
/// Print all page allocations
/// This is mainly used for debugging.
pub fn print_page_allocations() {
    unsafe {
        let num_pages = HEAP_SIZE / PAGE_SIZE;
        let mut beg = HEAP_START as *const Page;
        let end = beg.add(num_pages);
        let alloc_beg = ALLOC_START;
        let alloc_end = ALLOC_START + num_pages * PAGE_SIZE;
        println!();
        println!(
                    "PAGE ALLOCATION TABLE\nMETA: {:p} -> {:p}\nPHYS: \
                    0x{:x} -> 0x{:x}",
                    beg, end, alloc_beg, alloc_end
        );
        println!("~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~");
        let mut num = 0;
        while beg < end {
            if (*beg).is_taken() {
                let start = beg as usize;
                let memaddr = ALLOC_START
                                + (start - HEAP_START)
                                * PAGE_SIZE;
                print!("0x{:x} => ", memaddr);
                loop {
                    num += 1;
                    if (*beg).is_last() {
                        let end = beg as usize;
                        let memaddr = ALLOC_START
                                        + (end
                                            - HEAP_START)
                                        * PAGE_SIZE
                                        + PAGE_SIZE - 1;
                        print!(
                                "0x{:x}: {:>3} page(s)",
                                memaddr,
                                (end - start + 1)
                        );
                        println!(".");
                        break;
                    }
                    beg = beg.add(1);
                }
            }
            beg = beg.add(1);
        }
        println!("~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~");
        println!(
                    "Allocated: {:>6} pages ({:>10} bytes).",
                    num,
                    num * PAGE_SIZE
        );
        println!(
                    "Free     : {:>6} pages ({:>10} bytes).",
                    num_pages - num,
                    (num_pages - num) * PAGE_SIZE
        );
        println!();
    }
}
```

然后，让我们来分配一些页面! 我已经分配了一些单页和一段较大的 64 页空间。你会看到以下打印出的内容。

![ ](assets/3.1/page_allocations.png)

## 未来

对于较小的数据结构，页粒度的分配器会浪费大量的内存。例如，一个存储整数的链表结点将需要 4 字节 + 8 字节 = 12 字节的内存。4 字节用于整数，8 字节用于下一个指针。然而我们必须动态分配一整页。因此，我们有可能浪费 4096 - 12 = 4084 字节。所以，我们可以做的是更精细管理每个页面的分配，这个以字节为单位的分配器将很像 C++ 中 `malloc` 的实现方式。

当我们开始分配用户空间的进程时，我们需要保护内核内存不被用户的应用程序错误地写入，因此我们将使用内存管理单元，特别地，我们将使用Sv39 方案，它的文档在 [RISC-V 特权规范第 4.4章](https://github.com/riscv/riscv-isa-manual)中。

本章还将关注两个部分：2）内存管理单元（MMU）和 3）Rust 中 `alloc` crate 的字节粒度内存分配器，它包含数据结构，如链表和二叉树。我们还可以使用这个 crate 来存储和创建 `String` 类型。

[目录](index.md) → [第 2 章](ch2.md) → 第 3.1 章 → [第 3.2 章](ch3.2.md)
