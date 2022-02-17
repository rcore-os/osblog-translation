# 内存管理单元

这是[用 Rust 编写 RISC-V 操作系统](http://osblog.stephenmarz.com/index.html)系列教程中的第 3.2 章。

[目录](index.md) → [第 3.1 章](ch3.1.md) → 第 3.2 章 → [第 4 章](ch4.md)

**<span style='color:red'>2019 年 10 月 23 日：仅Patreon</span>**

**<span style='color:red'>2019 年 10 月 30 日：公开</span>**

**<span style='color:red'>于 2019 年 11 月 4 日更新</span>**

## 视频

本章博文的视频在 YouTube 上：[https://www.youtube.com/watch?v=-9sRy5UPWJk](https://www.youtube.com/watch?v=-9sRy5UPWJk)

## 概述

从操作系统的角度来看，我们有一个从某个起始地址到某个结束地址的大内存池。我们可以使用链接脚本（`virt.lds` 文件）提供表示起始和终止地址的符号。

一些内存将用于全局变量、堆栈空间等。但是，我们使用链接脚本来帮助我们处理这些不断移动的东西。其余空闲的内存成为我们的“堆”空间。然而，操作系统中的堆并不完全是 `malloc`/`free` 所使用的堆。

## 分页

分页是一个通过硬件（通常称为内存管理单元或 MMU）将虚拟地址翻译为物理地址的系统。MMU 通过读取名为 SATP（Supervisor Address Translation and Protection）的特权寄存器来执行地址翻译。该寄存器控制 MMU 的使能，设置地址空间标识符，并设置第一级页表所在的物理内存地址。

我们可以将这些表放在 RAM 中的任何位置，只要地址最后 12 位为 0。这是因为 SATP 中没有提供页表的最后 12 位。事实上，为了获得实际的内存地址，我们从 SATP 中取出 44 位并将其左移 12 位，同时在低 12 位填充 0。这组成了一个 56 位的地址——而不是 64 位。

本质上我们只需要关注以下三点：(1) 虚拟地址，(2) 物理地址，以及 (3) 页表项。这些都列在 [RISC-V 特权规范第 4.4章](https://github.com/riscv/riscv-isa-manual)中。

RISC-V 有不同的分页模式。HiFive Unleashed 使用 Sv39 模式，这意味着虚拟地址为 39 位。在这种情况下，虚拟地址无法使用完整的 64 位内存空间。其他的分页模式包括使用 32 位虚拟地址的 Sv32 和使用 48 位虚拟地址的 Sv48。

## 虚拟地址

Sv39 虚拟地址包含三个 9 位索引。由于 \\( 2^9 = 512 \\)，每级页表都包含 512 个页表项，每个页表项正好是 8 字节（64 位），如下所示。 Sv39 虚拟地址包含 VPN\[x\]。 VPN 代表“虚拟页号”，它本质上是一个包含 512 个 8 字节条目的数组的索引。因此，MMU 不是直接将虚拟地址转换为物理地址，而是通过一系列页表。使用 Sv39 系统，我们的页表可以三个级别，每个级别是一张包含 512 个 8 字节项的表。

![ ](assets/3.2/sv39_virtual_address.png)

## 页表项

操作系统会写入一个页表项来控制 MMU 的工作方式。我们可以更改地址的翻译方式，也可以设置某些位来保护页面。页面入口如下图所示：

![ ](assets/3.2/sv39_pte.png)

## 物理地址

物理地址实际上是 56 位。因此，一个 39 位的虚拟地址可以转换为一个 56 位的物理地址。显然，这允许我们将相同的虚拟地址映射到不同的物理地址——就像我们在创建用户进程时映射地址的方式一样。物理地址是通过获取 PPN\[2:0\] 并将它们按照以下格式拼接来形成的。可以注意到页面偏移量是直接从虚拟地址复制到物理地址的。

![ ](assets/3.2/sv39_physical_address.png)

## SATP 寄存器

所有的地址翻译都从如下所示的SATP寄存器开始，具体描述在 [RISC-V 特权规范第 4.1.12 章](https://github.com/riscv/riscv-isa-manual)中。SATP 寄存器的示意图如下。

![ ](assets/3.2/satp.png)

我们可以看到 PPN（物理页号）是一个 44 位的值。但是，该值在存储之前已右移 12 位，这本质上是将实际地址除以 4096 (\\( 2^12 = 4096 \\))。因此，实际地址为 `PPN << 12`。

## 页面错误

MMU 通过三个不同的分页错误向 CPU 发出地址翻译错误信号：(1) 指令、(2) 加载和 (3) 存储。当 MMU 引发页面错误时，在取指阶段发生指令分页错误。这可能意味着页表项无效或没有将 X（执行）位设置为 1 或其他原因。

页面错误由 CPU 捕获， `mcause` 或 `scause` 寄存器中存有异常编号。 12 表示指令页面错误、13 表示加载页面错误、 15 表示存储页面错误。操作系统可以决定如何处理。在大多数情况下，正确的处理方式是杀死有问题的进程。但是，有一些高级的系统机制，例如写时复制（Copy-on-Write, CoW），是通过捕获页面错误来实现的。在这些情况下，页面错误不是一个问题，而是一个特性。

## 虚拟地址翻译

举个例子。虚拟地址 `0x7d_beef_cafe` ，正好是 39 位。如果地址长于 39 位， MMU 很可能会产生一个页面错误。

二进制：0b0111_1101_1011_1110_1110_1111_1100_1010_1111_1110

| VPN\[2\]        | VPN\[1\]        | VPN\[0\]        | 12 位偏移          |
| --------------- | --------------- | --------------- | ------------------ |
| \[1_1111_0110\] | \[1_1111_0111\] | \[0_1111_1100\] | \[1010_1111_1110\] |
| 502             | 503             | 252             |

上面的地址将按照以下步骤进行翻译：

1. 读取 SATP 寄存器并找到第 2 级页表的首项地址 (`PPN << 12`)。
2. 在首项地址上加上偏移量\*大小，其中偏移量为 `VPN[2] = 0b1_1111_0110 = 502 * 8 = SATP + 4016`
3. 读取此条目。
4. 如果 V(valid)=0，产生页面错误
5. 如果该条目的 R、W 或 X 位为 1，则该页表项为叶项，否则为枝干项。
6. PPN \[2\] | PPN\[1\] | PPN\[0\] 地址指示下一个页表在物理内存中的位置。
7. 从第 2 步重复，直到找到叶项。
8. 叶项表示 PPN\[2\]、PPN\[1\] 和 PPN\[0\] 告诉 MMU 实际物理地址是什么。

## 每一级页表都可以成为叶项

像大多数内存管理单元一样，MMU 可以翻译更粗粒度的页面。例如，Intel/AMD x86_64 系统可以在 PDP 处停止，这可以提供 1GB 的粒度，而在 PD 处停止将提供 2MB 的粒度。这同样适用于 RISC-V 系统。不过，RISC-V 采用了一种巧妙的方法来区分叶项和枝干项。

在 Intel/AMD 机器上，上级页表控制下级页表。如果某个上级页表是只读的，那么之后的任何枝干项都不可写。 RISC-V 认为这很愚蠢，所以他们的设计是这样的：如果 R、W、X（读、写、执行）保护位中的任何一个被置位，则该页表项是一个页项。

如果一个页表项 (PTE) 是叶子，则 PPN\[2:0\] 表示物理地址。但是，如果一个 PTE 是枝干项，则 PPN\[2:0\] 描述可以在物理 RAM 中找到下一个页表的位置。需要注意的一点是，只有 PPN[2:leaf-level] 将用于开发物理地址。例如，如果第 2 级（顶层）的页表条目是叶子，那么只有 PPN[2] 对物理地址有贡献。 VPN[1] 被复制到 PPN[1]，VPN[0] 被复制到 PPN[0]，并且页偏移被复制，正常。

## 对MMU的编程

在 Rust 中，我创建了三个对 MMU 进行编程的函数： `map` 、 `unmap` 和 `virt_to_phys` 。我们需要能够手动遍历页表，因为当用户空间的应用程序进行系统调用时，每个指针都是指向虚拟内存地址的指针--这与我们在内核中的地址不一样。因此，我们必须一路深入到一个物理地址，以使用户空间应用程序和内核使用的是同一个内存地址。

我使用了一些结构体来简化页表项的编程。

```rust
pub struct Table {
    pub entries: [Entry; 512],
}

impl Table {
    pub fn len() -> usize {
        512
    }
}
```

第一个结构体是 `Table` 。它描述了一级 4096 字节的页表。这个大小来自 512 个 8 字节的页表项。一个页表项被描述为：

```rust
pub struct Entry {
    pub entry: i64,
}

impl Entry {
    pub fn is_valid(&self) -> bool {
        self.get_entry() & EntryBits::Valid.val() != 0
    }

    // 首位 (下标 #0) 是有效位 (Valid, V)
    pub fn is_invalid(&self) -> bool {
        !self.is_valid()
    }

    // 页项的 RWX 位至少有一位被置位
    pub fn is_leaf(&self) -> bool {
        self.get_entry() & 0xe != 0
    }

    pub fn is_branch(&self) -> bool {
        !self.is_leaf()
    }

    pub fn set_entry(&mut self, entry: i64) {
        self.entry = entry;
    }

    pub fn get_entry(&self) -> i64 {
        self.entry
    }
}
```

本质上我只是把 `i64` 数据类型重命名为 `Entry`，这样我就可以给它添加一些辅助函数。

`map` 函数接收一个页表根的可变引用，一个虚拟地址，一个物理地址，保护位，以及这个地址应该被映射到哪个级别。通常情况下，我们将所有的页面映射到 0 级，也就是 4KiB 级。然而，我们可以用 2 级表示比 1GiB 页，1 表示 2MiB 页，或 0 表示 4KiB 页。

```rust
pub fn map(root: &mut Table, vaddr: usize, paddr: usize, bits: i64, level: usize) {
    // 确保 RWX 至少有一位被置位，否则会导致内存泄漏并且产生一个页错误
    assert!(bits & 0xe != 0);
    // 从虚拟地址中提取 VPN 。虚拟地址中, 每段 VPN 都恰好是 9 位，
    // 所以我们用 0x1ff = 0b1_1111_1111 (9 位) 作为掩码
    let vpn = [
                // VPN[0] = vaddr[20:12]
                (vaddr >> 12) & 0x1ff,
                // VPN[1] = vaddr[29:21]
                (vaddr >> 21) & 0x1ff,
                // VPN[2] = vaddr[38:30]
                (vaddr >> 30) & 0x1ff,
    ];

    // 与虚拟地址类似，提取物理页号 (PPN)。但是 PPN[2] 是 26 位而不是 9 位，
    // 这是不同之处。因此我们使用
    // 0x3ff_ffff = 0b11_1111_1111_1111_1111_1111_1111 (26 位).
    let ppn = [
                // PPN[0] = paddr[20:12]
                (paddr >> 12) & 0x1ff,
                // PPN[1] = paddr[29:21]
                (paddr >> 21) & 0x1ff,
                // PPN[2] = paddr[55:30]
                (paddr >> 30) & 0x3ff_ffff,
    ];
```

我们在这里做的第一件事是分解虚拟地址和物理地址。注意，我们并不关心页面偏移量--这是因为我们并不存储页面偏移量。相反，当 MMU 翻译一个虚拟地址时，它直接将页面偏移量复制到物理地址中，形成一个完整的 56 位物理地址。

```rust
// We will use this as a floating reference so that we can set
// individual entries as we walk the table.
let mut v = &mut root.entries[vpn[2]];
// Now, we're going to traverse the page table and set the bits
// properly. We expect the root to be valid, however we're required to
// create anything beyond the root.
// In Rust, we create a range iterator using the .. operator.
// The .rev() will reverse the iteration since we need to start with
// VPN[2] The .. operator is inclusive on start but exclusive on end.
// So, (0..2) will iterate 0 and 1.
for i in (level..2).rev() {
    if !v.is_valid() {
        // Allocate a page
        let page = zalloc(1);
        // The page is already aligned by 4,096, so store it
        // directly The page is stored in the entry shifted
        // right by 2 places.
        v.set_entry(
                    (page as i64 >> 2)
                    | EntryBits::Valid.val(),
        );
    }
    let entry = ((v.get_entry() & !0x3ff) << 2) as *mut Entry;
    v = unsafe { entry.add(vpn[i]).as_mut().unwrap() };
}
// When we get here, we should be at VPN[0] and v should be pointing to
// our entry.
// The entry structure is Figure 4.18 in the RISC-V Privileged
// Specification
let entry = (ppn[2] << 28) as i64 |   // PPN[2] = [53:28]
            (ppn[1] << 19) as i64 |   // PPN[1] = [27:19]
            (ppn[0] << 10) as i64 |   // PPN[0] = [18:10]
            bits |                    // Specified bits, such as User, Read, Write, etc
            EntryBits::Valid.val();   // Valid bit
            // Set the entry. V should be set to the correct pointer by the loop
            // above.
v.set_entry(entry);
```

在上面的代码中，我们用 `zalloc` 分配了一个新的页面。幸运的是，RISC-V 的每级页表正好是 4096 字节（512个页表项，每个 8 字节），这正是我们使用 `zalloc` 分配的大小。

我的学生面临的一个挑战是，如果你看一下，PPN\[2:0\] 在 PTE 和物理地址中是一样的，只是它们的位置不同。由于某些原因，PPN\[2:0\] 被向右移了两位，而物理地址从第 10 位开始。而在物理地址中是从第 12 位开始的。然而，使用我上面采取的方法，即单独对 PPN 本身进行编码，我们应该不会有什么问题。

另一个有趣的方面是，无论我们在哪一级页表，我总是对 PPN\[2:0\] 进行编码。这对页表项来说还好，但它可能会造成混乱。记住，如果我们停在第 1 级，那么 PPN\[0\] 就不会被从 PTE 中复制出来。相反，PPN\[0\] 是从VPN\[0\] 中复制的。这是由 MMU 硬件执行的。最后，我们只是添加了比页表项可能需要的更多的信息。

## 释放 MMU 表

我们将为每个用户空间的应用程序至少分配一个页。然而，我计划只允许用户空间的应用程序使用 4KiB 大小的页。因此，我们至少需要为一个地址准备三个页表项。这意味着，如果我们不重复使用这些页，内存很快就会耗尽。为了释放内存，我们将使用 `unmap` 。

```rust
pub fn unmap(root: &mut Table) {
    // Start with level 2
    for lv2 in 0..Table::len() {
        let ref entry_lv2 = root.entries[lv2];
        if entry_lv2.is_valid() && entry_lv2.is_branch() {
            // This is a valid entry, so drill down and free.
            let memaddr_lv1 = (entry_lv2.get_entry() & !0x3ff) << 2;
            let table_lv1 = unsafe {
                // Make table_lv1 a mutable reference instead of a pointer.
                (memaddr_lv1 as *mut Table).as_mut().unwrap()
            };
            for lv1 in 0..Table::len() {
                let ref entry_lv1 = table_lv1.entries[lv1];
                if entry_lv1.is_valid() && entry_lv1.is_branch()
                {
                    let memaddr_lv0 = (entry_lv1.get_entry()
                                        & !0x3ff) << 2;
                    // The next level is level 0, which
                    // cannot have branches, therefore,
                    // we free here.
                    dealloc(memaddr_lv0 as *mut u8);
                }
            }
            dealloc(memaddr_lv1 as *mut u8);
        }
    }
}
```

就像 `map` 函数一样，这个函数假设了一个合法的页表根（第 2 级页表）。我们把它的可变引用传递到 `unmap` 函数中。因此，这段代码背后的逻辑是，我们从最低一级（第 0 级）开始释放空间，然后一路回到最高级（第 2 级）。上述代码本质上是使用迭代方式实现了一个递归函数。请注意，我没有释放页表根本身。由于我们得到了一个通用的的 `Table` 结构体的引用，我们可以传入一个 1 级表来释放一个更大的表。

## 遍历页表

最后，我们需要手动遍历页表。这被用于从虚拟内存地址复制数据。由于虚拟内存地址在内核和用户进程之间是不同的，我们需要通过物理地址进行转化。唯一的方法是将虚拟内存地址转换为对应的物理地址。与硬件 MMU 不同，我们可以将任何页表传递给这个函数，无论这个表目前是否正被 MMU 使用。

```rust
pub fn virt_to_phys(root: &Table, vaddr: usize) -> Option {
    // Walk the page table pointed to by root
    let vpn = [
                // VPN[0] = vaddr[20:12]
                (vaddr >> 12) & 0x1ff,
                // VPN[1] = vaddr[29:21]
                (vaddr >> 21) & 0x1ff,
                // VPN[2] = vaddr[38:30]
                (vaddr >> 30) & 0x1ff,
    ];

    let mut v = &root.entries[vpn[2]];
    for i in (0..=2).rev() {
        if v.is_invalid() {
            // This is an invalid entry, page fault.
            break;
        }
        else if v.is_leaf() {
            // According to RISC-V, a leaf can be at any level.

            // The offset mask masks off the PPN. Each PPN is 9
            // bits and they start at bit #12. So, our formula
            // 12 + i * 9
            let off_mask = (1 << (12 + i * 9)) - 1;
            let vaddr_pgoff = vaddr & off_mask;
            let addr = ((v.get_entry() << 2) as usize) & !off_mask;
            return Some(addr | vaddr_pgoff);
        }
        // Set v to the next entry which is pointed to by this
        // entry. However, the address was shifted right by 2 places
        // when stored in the page table entry, so we shift it left
        // to get it back into place.
        let entry = ((v.get_entry() & !0x3ff) << 2) as *const Entry;
        // We do i - 1 here, however we should get None or Some() above
        // before we do 0 - 1 = -1.
        v = unsafe { entry.add(vpn[i - 1]).as_ref().unwrap() };
    }

    // If we get here, we've exhausted all valid tables and haven't
    // found a leaf.
    None
}
```

在上面的 Rust 函数中，我们得到了一个对页表的常引用。我们返回一个 `Option` ，它是一个枚举类型，要么是 `None` （用于指示页面错误），要么是 `Some()`（用于返回物理地址）。

使用 RISC-V 内存管理单元时有几种情况会产生页面错误。一个是遇到了一个无效的页表项（V 位为 0）。另一个是在第 0 级有枝干项。由于 0 级是最低的级别，如果我们发现一个更低的级别（比如 -1？），就会发生页面故障。

## A 和 D 位

如果你读过 RISC-V 的特权规范，D（dirty）和 A（accessed）位可以有两种不同的含义。

对于应用处理器来说，更传统的含义是，当内存被读取或写入时，CPU 都会将 A 位设置为 1 ，而内存被写入时 CPU 会将 D 位设置为 1。然而，RISC-V 规范本质上也允许硬件将这些作为冗余的 R 和 W 位--至少我是这么理解的。换句话说，由我们内核程序员来控制 A 和 D 位，而不是硬件。

HiFive Unleashed 开发板使用的是后者，即如果 D 位（用于存储）或 A 位（用于加载）在页表项中为 0，MMU 会抛出一个页面错误，所以让我们来仔细看看。

1. 如果你试图写入一个 D 位为 0 的页面，MMU 将抛出一个页面错误，这与 W 位为 0 的情况很相似。
2. 如果你试图读取一个 A 位为 0 的页面，MMU 将抛出一个页面错误，这与 R 位为 0 的情况很相似。

![ ](assets/3.2/sv39_mmu_ad_bits.png)

RISC-V 特权规范，第4.3.1章

简而言之，我们有两种选择：（a）软件控制或（b）硬件控制。

## 内核的地址映射

我们将切换到内核态，这样可以让内核内存使用 MMU，不过我们首先需要映射一切我们需要的内容。在 RISC-V 中，机器态使用物理内存地址；而如果我们打开 MMU（将 MODE 字段设置为 8），那么我们可以在内核或用户态下使用 MMU。

要做到这一点，我们要对内核中需要的一切进行恒等映射（虚拟地址=物理地址），包括程序代码、全局段、UART MMIO 地址等等。首先，我用 Rust 写了一个函数，它将帮助我恒等映射一段地址。

```rust
pub fn id_map_range(root: &mut page::Table,
    start: usize,
    end: usize,
    bits: i64)
{
    let mut memaddr = start & !(page::PAGE_SIZE - 1);
    let num_kb_pages = (page::align_val(end, 12)
        - memaddr)
        / page::PAGE_SIZE;

    // I named this num_kb_pages for future expansion when
    // I decide to allow for GiB (2^30) and 2MiB (2^21) page
    // sizes. However, the overlapping memory regions are causing
    // nightmares.
    for _ in 0..num_kb_pages {
    page::map(root, memaddr, memaddr, bits, 0);
    memaddr += 1 << 12;
    }
}
```

如你所见，这个函数调用了我们的 `page::map` 函数，它将一个虚拟地址映射到一个物理地址。我们得到的根表是一个可变引用，可以把它直接传递给 map 函数。

现在，我们需要用这个函数来映射我们的程序代码段和稍后设备驱动所需的所有的 MMIO 地址。

```rust
page::init();
kmem::init();

// Map heap allocations
let root_ptr = kmem::get_page_table();
let root_u = root_ptr as usize;
let mut root = unsafe { root_ptr.as_mut().unwrap() };
let kheap_head = kmem::get_head() as usize;
let total_pages = kmem::get_num_allocations();
println!();
println!();
unsafe {
    println!("TEXT:   0x{:x} -> 0x{:x}", TEXT_START, TEXT_END);
    println!("RODATA: 0x{:x} -> 0x{:x}", RODATA_START, RODATA_END);
    println!("DATA:   0x{:x} -> 0x{:x}", DATA_START, DATA_END);
    println!("BSS:    0x{:x} -> 0x{:x}", BSS_START, BSS_END);
    println!("STACK:  0x{:x} -> 0x{:x}", KERNEL_STACK_START, KERNEL_STACK_END);
    println!("HEAP:   0x{:x} -> 0x{:x}", kheap_head, kheap_head + total_pages * 4096);
}
id_map_range(
                &mut root,
                kheap_head,
                kheap_head + total_pages * 4096,
                page::EntryBits::ReadWrite.val(),
);
unsafe {
    // Map heap descriptors
    let num_pages = HEAP_SIZE / page::PAGE_SIZE;
    id_map_range(&mut root,
                    HEAP_START,
                    HEAP_START + num_pages,
                    page::EntryBits::ReadWrite.val()
    );
    // Map executable section
    id_map_range(
                    &mut root,
                    TEXT_START,
                    TEXT_END,
                    page::EntryBits::ReadExecute.val(),
    );
    // Map rodata section
    // We put the ROdata section into the text section, so they can
    // potentially overlap however, we only care that it's read
    // only.
    id_map_range(
                    &mut root,
                    RODATA_START,
                    RODATA_END,
                    page::EntryBits::ReadExecute.val(),
    );
    // Map data section
    id_map_range(
                    &mut root,
                    DATA_START,
                    DATA_END,
                    page::EntryBits::ReadWrite.val(),
    );
    // Map bss section
    id_map_range(
                    &mut root,
                    BSS_START,
                    BSS_END,
                    page::EntryBits::ReadWrite.val(),
    );
    // Map kernel stack
    id_map_range(
                    &mut root,
                    KERNEL_STACK_START,
                    KERNEL_STACK_END,
                    page::EntryBits::ReadWrite.val(),
    );
}

// UART
page::map(
            &mut root,
            0x1000_0000,
            0x1000_0000,
            page::EntryBits::ReadWrite.val(),
            0
);
```

这有很多代码，因此我需要编写 `id_map_range` 函数。在这段代码中，我们首先初始化我们在第 3.1 章中创建的页分配系统。然后，我们创建一个页表根，它正好需要1页（4096字节）。来回转换内存地址的类型的意义在于，我们最终需要将一个物理地址写入 SATP（Supervisor Address Translation and Protection，监管者地址翻译与保护）寄存器以使 MMU 正常运行。

现在我们有了页表根，我们需要把它写进 SATP 寄存器的 PPN 字段。PPN 本质上是页表根的 56 位物理地址的前 44 位。我们要做的是把页表根的地址向右移 12 位，这本质上是把地址除以一个页面的大小。

![ ](assets/3.2/satp.png)

```rust
let root_ppn = root_u >> 12;
let satp_val = 8 << 60 | root_ppn;
unsafe {
    asm!("csrw satp, $0" :: "r"(satp_val));
}
```

上面的代码使用了 `asm!` 宏，它允许我们在 Rust 中直接编写汇编代码。在这里，我们使用 `csrw` 指令，意思是 "控制和状态寄存器-写入"。 `8 << 60` 把数值 8 放在 SATP 寄存器的 MODE 字段中，表示 Sv39 分页模式。

## 开启MMU

仅在 MMU 模式字段中写入 8 并没有完全打开 MMU。原因是我们处于 3 号 CPU 模式，也就是机器模式。所以，我们需要切换到“监督者”模式，也就是模式 1 。我们通过在 `mstatus` 寄存器的 MPP（Machine Previous Privilege）字段（第 11 和 12 位）中写入二进制数字 01 来实现这一点。

我们切换 SATP 寄存器时需要谨慎，因为页表会被缓存。切换的次数越多，我们就需要刷新缓存：旧的出来，新的进去。当用户进程开始切入和切出上下文时，我们需要同步页表。如果你现在对此有兴趣，请看 [RISC-V 的特权规范第4.2.1章](https://github.com/riscv/riscv-isa-manual)，不过当我们开始运行进程时，我们会详细介绍这一点。

![ ](assets/3.2/mstatus.png)

```assembly
    li t0, (1 << 11) | (1 << 5)
    csrw mstatus, t0

    la t1, kmain
    csrw mepc, t1

    mret
```

我们在上面的代码中所做的是启用中断，并将第 11 位开始的数值置为 1（MPP=01）。当我们执行 `mret` 时，我们进入了一个叫 `kmain` 的 Rust 函数。现在我们处于监督者模式并且完全启用了 MMU。

我们还没有处理任何页面错误，所以要小心！如果你跑到地址空间以外的地方，你的内核就会挂起宕机。
