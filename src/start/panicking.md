# 恐慌

Panicking是Rust语言的核心一部分.
像是索引一样的内置操作会在运行时检查安全性.
当尝试超出索引范围时, 结果就会导致panic.

在标准库之中, 恐慌有一个定义的行为: 除非用户选择在出现恐慌时结束程序, 否则它将展开出现恐慌行为的线程的堆栈.

然而, 在没有使用标准库的程序中, 恐慌行为没有定义.
可以使用`#[panic_handler]`来指定恐慌行为.
这个函数在整个程序的语法树中只能出现*一次*, 并且必须有如下标志: `fn(&PanicInfo) -> !`, [`PanicInfo`]是一个包含了恐慌位置信息的结构体.

[`PanicInfo`]: https://doc.rust-lang.org/core/panic/struct.PanicInfo.html

鉴于嵌入式系统的范围从面型用户到所以安全至关重要(不崩溃), 所以没有任何一种适用于全部情况的恐慌行为, 但是有很多经常使用的.
这些库中已经定义了`#[panic_handler]`函数. 这有几个例子:

- [`panic-abort`]. 恐慌使指令停止运行.
- [`panic-halt`]. 恐慌使当前程序或线程进入一个无限循环.
- [`panic-itm`]. 使用ITM(Cortex-M的一种外设)记录恐慌消息.
- [`panic-semihosting`]. 使用semihosting技术将恐慌信息输出到主机上.

  
[`panic-abort`]: https://crates.io/crates/panic-abort
[`panic-halt`]: https://crates.io/crates/panic-halt
[`panic-itm`]: https://crates.io/crates/panic-itm
[`panic-semihosting`]: https://crates.io/crates/panic-semihosting

你可以在crates.io上使用关键词[`panic-handler`]搜索到更多库.

[`panic-handler`]: https://crates.io/keywords/panic-handler

程序可以通过简单的链接到其中一个库来选择一个恐慌行为.
恐慌行为在应用程序的源代码中表示为一行代码这一事实不仅可用作文档, 而且还可以根据编译配置文件用于更改恐慌行为.
例如:

``` rust,ignore
#![no_main]
#![no_std]

// dev profile: easier to debug panics; can put a breakpoint on `rust_begin_unwind`
#[cfg(debug_assertions)]
use panic_halt as _;

// release profile: minimize the binary size of the application
#[cfg(not(debug_assertions))]
use panic_abort as _;

// ..
```

在这个例子中我们选择在开发时(`cargo build`)我们选择`panic-halt`, 但在发布时(`cargo build --release`)我们选择`panic-abort`.

> `use panic_abort as _;`使用`use`语句来确保在最终二进制产物中`panic_abort`被包含进去, 同时也让编译器知道我们不会使用其中的任何内容.
> 没有`as _`的话, 编译器会给我们一个Warn来告诉我们有个没有使用的导入库.
> 又是你会看见`extern crate panic_abort`, 这是一个在2018版本之前的旧版本的写法, 现在仅仅应用于"sysroot"库(那些随着Rust一起发布的库), 像是 `proc_macro`, `alloc`, `std`, 还有 `test`.

## 一个例子

这有一个试图越界数组的例子.
最终结果会导致恐慌.

```rust,ignore
#![no_main]
#![no_std]

use panic_semihosting as _;

use cortex_m_rt::entry;

#[entry]
fn main() -> ! {
    let xs = [0, 1, 2];
    let i = xs.len() + 1;
    let _y = xs[i]; // out of bounds access

    loop {}
}
```

这个例子选择使用semihosting技术输出信息到主机的`panic-semihosting`.

``` console
$ cargo run
     Running `qemu-system-arm -cpu cortex-m3 -machine lm3s6965evb (..)
panicked at 'index out of bounds: the len is 3 but the index is 4', src/main.rs:12:13
```

你可以试着把恐慌行为改成`panic-halt`来确认一下还会不会有信息输出.
