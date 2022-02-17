# The Adventures of OS

使用 Rust 的 RISC-V 操作系统

[在 Patreon 上支持我!](https://www.patreon.com/sgmarz) &emsp;
[操作系统博客](http://osblog.stephenmarz.com/) &emsp;
[RSS 订阅](http://osblog.stephenmarz.com/feed.rss) &emsp;
[Github](https://github.com/sgmarz) &emsp;
[EECS 网站](http://web.eecs.utk.edu/~smarz1) &emsp;

这是[用 Rust 编写 RISC-V 操作系统](http://osblog.stephenmarz.com/index.html)系列教程中的第0章，介绍如何正确地构建运行环境。

[目录](http://osblog.stephenmarz.com/index.html) → 第0章 → [第1章](http://osblog.stephenmarz.com/ch1.html)

## 我需要你的支持

对我来说，写这篇博客现在只是业余消遣，因为我的本职工作主要是本科生教育。我会坚持写下去，但真的很希望有人来帮忙。如果你愿意，请在[Patreon(sgmarz)](https://www.patreon.com/sgmarz) 支持我。

我刚刚起步，要做的还有很多！所以，请加入我们吧!

# 环境配置与依赖安装

**<span style='color:red;'>2019年9月27日</span>**

## 2020年更新

Rust现在对RISC-V的支持使我们能开箱即用！我们不通过工具链也能很容易地来构建项目。但在工具链之外，仍需要QEMU（模拟器），在[ch0.old.html](http://osblog.stephenmarz.com/ch0.old.html) 的旧版教程中有相关介绍。

Rust要求我们使用`rustup`命令安装一些依赖。如果你没有rustup，请在[https://www.rust-lang.org/tools/install](https://www.rust-lang.org/tools/install)下载。

```sh
rustup default nightly
rustup target add riscv64gc-unknown-none-elf
cargo install cargo-binutils
```

由于我在系统中使用了语言特性（用 `#![features]` 表示），我们必须使用 nightly 版本，即使 RISC-V 有稳定版本。

## 构建与运行

在[博客的 git 仓库 (Github)](https://github.com/sgmarz/osblog) 中你会看到一个名为 `.cargo/config` 的文件。你可以根据你的系统来编辑这个文件：

```toml
[build]
target = "riscv64gc-unknown-none-elf"
rustflags = ['-Clink-arg=-Tsrc/lds/virt.lds']

[target.riscv64gc-unknown-none-elf]
runner = "qemu-system-riscv64 -machine virt -cpu rv64 -smp 4 -m 128M -drive if=none,format=raw,file=hdd.dsk,id=foo -device virtio-blk-device,scsi=off,drive=foo -nographic -serial mon:stdio -bios none -device virtio-rng-device -device virtio-gpu-device -device virtio-net-device -device virtio-tablet-device -device virtio-keyboard-device -kernel "
```

配置文件显示了我们要构建的目标，也就是 riscv64gc 。我们还需要指定我们的链接器脚本 `src/lds/virt.lds` ，以便我们能确保正确的文件位置。最后，当我们输入 `cargo run` 时，会调用一个 "runner"，它将运行 riscv64 qemu 。另外注意在 -kernel 后面有一个空格，这是因为 cargo 会自动指定通过 Cargo.toml 配置的可执行文件。

## 完成！

这部分确实很无聊，如果配置过程一切顺利（在ArchLinux上配置正常！），接下来的部分将有趣起来。

