# 一个 `no_std` 的Rust环境

嵌入式编程一词用于很多不同种类的涵义. 从只有几KB大小RAM与ROM的8位MCU, 到像是树莓派这样有32/64位四核Cortex-A53 cpu与1GB内存的设备.
编写代码时, 对于不同设备会有不同的限制.

有两种通用的嵌入式变成分类:

## 托管环境

这种环境与正常的PC环境相似. 这意味这你能够使用系统级接口, 类似POSIX这样的能提供给你与系统交互的原语, 像是文件系统, 网络, 内存管理, 线程等等.
你可能还会有些sysroot和RAM/ROM的限制, 可能还会有些特殊的硬件或I/O. 简而言之, 这类似在一台特殊用途的PC环境上编程.

## 裸金属

在一个裸金属环境中, 在你的程序开始之前不会有任何代码被加载.
没有OS提供我们没法使用标准库.
相反, 程序和它使用的库(Crates)可以只使用硬件(裸金属)来运行.
为了防止rust使用标准库, 我们使用`no_std`.
标准库中与平台无关的部分可以通过[libcore](https://doc.rust-lang.org/core/)获取.
libcore中也排除了在嵌入式环境中并不总是理想的东西.
这其中之一就是用于动态内存分配的内存分配器.
如果你需要这个或是其他功能, 会有库(Crates)提供.

### libstd运行时

像前面说的, 使用[libstd](https://doc.rust-lang.org/std/)需要系统支持, 但是这并不只是因为[libstd](https://doc.rust-lang.org/std/)至提供了访问OS的通用的抽象的方法, 而且它还提供了一个运行时.
这个运行时, 除了其他事情外, 还负责设置对战一处保护, 处理命令行参数还有在调用程序的main函数之前创建主线程. 这个运行时在`no_std`环境中不可用.

## 总结

`#![no_std]`是一个声明这个crate不会连接到std-crate二十core-crate的crate级别的属性.
[libcore](https://doc.rust-lang.org/core/)是std-crate的一个与平台无关的子集, 对程序将要运行在的系统上没有任何假设(需求).
因此, 它为语言原语,像是float, string和slices等提供api, 和开放的处理器特性, 像是原子操作与SIMD指令.
然而他缺少任何设计平台集成的API.
由于这些属性, no\_std与[libcore](https://doc.rust-lang.org/core/)写成的代码能不能够用于任何类型的引导(stage 0)像是加载程序, 固件还有内核.

### 概述

| feature                                                   | no\_std | std |
|-----------------------------------------------------------|--------|-----|
| 堆 (动态内存)                                              |   *    |  ✓  |
| 集合 (Vec, HashMap, etc)                                  |  **    |  ✓  |
| 堆栈溢出保护                                               |   ✘    |  ✓  |
| 初始化函数                                                 |   ✘    |  ✓  |
| libstd  可用                                               |   ✘    |  ✓  |
| libcore 可用                                               |   ✓    |  ✓  |
| 编写 固件, 内核, 引导加载器                                  |   ✓    |  ✘  |

\* 只有当你使用 `alloc` crate并且选择一个合适的分配器, 像是[alloc-cortex-m]才可用.

\** 只有当你使用 `collections` crate 并且配置一个全局默认的分配器才可用

[alloc-cortex-m]: https://github.com/rust-embedded/alloc-cortex-m

## See Also

* [RFC-1184](https://github.com/rust-lang/rfcs/blob/master/text/1184-stabilize-no_std.md)
