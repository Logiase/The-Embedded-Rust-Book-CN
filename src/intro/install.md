# 安装工具

此页包含与操作系统无关的工具安装说明:

### Rust 工具链

按照[https://rustup.rs](https://rustup.rs)的教程安装rustup.

**NOTE** 确保你有`1.31`或以上版本的编译器. `rustc -V`应该返回一个更新的版本.

``` console
$ rustc -V
rustc 1.31.1 (b6c32da9b 2018-12-18)
```

为了带宽和磁盘的使用情况, 默认安装只支持本机编译.
要想添加ARM Cortex-M的交叉编译支持, 应该选择下面其一的编译目标.
对于STM32F3DISCOVERY开发板, 应该使用`thumbv7em-none-eabihf`

Cortex-M0, M0+, M1 (ARMv6-M 架构):
``` console
$ rustup target add thumbv6m-none-eabi
```

Cortex-M3 (ARMv7-M 架构):
``` console
$ rustup target add thumbv7m-none-eabi
```

没有硬浮点的Cortex-M4 and M7 (ARMv7E-M 架构):
``` console
$ rustup target add thumbv7em-none-eabi
```

有硬浮点的Cortex-M4F and M7F (ARMv7E-M 架构):
``` console
$ rustup target add thumbv7em-none-eabihf
```

Cortex-M23 (ARMv8-M 架构):
``` console
$ rustup target add thumbv8m.base-none-eabi
```

Cortex-M33 and M35P (ARMv8-M 架构):
``` console
$ rustup target add thumbv8m.main-none-eabi
```

有硬浮点的Cortex-M33F and M35PF (ARMv8-M 架构):
``` console
$ rustup target add thumbv8m.main-none-eabihf
```

### `cargo-binutils`

``` console
$ cargo install cargo-binutils

$ rustup component add llvm-tools-preview
```

### `cargo-generate`

我们后面用这个来生成项目

``` console
$ cargo install cargo-generate
```

### OS相关安装

现在跟着这些教程安装:

- [Linux](install/linux.md)
- [Windows](install/windows.md)
- [macOS](install/macos.md)
