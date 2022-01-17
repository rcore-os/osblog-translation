# 页面粒度的内存分配

## 视频

这个帖子的视频在 YouTube 上：[https://www.youtube.com/watch?v=SZ6GMxLkbx0](https://www.youtube.com/watch?v=SZ6GMxLkbx0)。

## 概述

当我们启动时，.elf文件内的所有内容都会自动加载到内存中。在代码中可见，我们为整个系统分配了 128 兆字节空间。因此，我们需要处理除去所有声明的全局变量和代码之外的空间。

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

在这里，我们需要为每个页面分配一个 Page 结构体。幸运的是，通过升级过的链接器脚本，我们能够获得堆的地址和大小。

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

我们可以简单的地计算出堆中的总页数：`let num_pages = HEAP_SIZE / PAGE_SIZE;`，其中 HEAP\_SIZE 是一个全局符号，值为 \_heap\_size，页大小是一个常数为 4096。最终我们必须分配 num_pages 数量的 `Page` 描述符来存储分配信息。
