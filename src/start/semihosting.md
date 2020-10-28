# Semihosting

Semihosting是一种让嵌入式设备的能在宿主机上IO的机制,主要用来在主机控制台上log.
Semihosting需要一个调试会话,剩下什么都不要(甚至不需要额外的线!),所以它很方便.
下行速度非常长非常慢:取决于硬件调试器(例如ST-LINK),每个写操作都可能要几毫秒.

[`cortex-m-semihosting`]库提供一个在Cortex-M的设备上使用semihosting的API.
下面的程序是"Hello, world!"的semihosting版本:

[`cortex-m-semihosting`]: https://crates.io/crates/cortex-m-semihosting

```rust,ignore
#![no_main]
#![no_std]

use panic_halt as _;

use cortex_m_rt::entry;
use cortex_m_semihosting::hprintln;

#[entry]
fn main() -> ! {
    hprintln!("Hello, world!").unwrap();

    loop {}
}
```

如果你在这个硬件上运行程序,你会在OpenOCD日志上看到"Hello, world!".

``` console
$ openocd
(..)
Hello, world!
(..)
```

你首先要从GDB中启动semihosting:

``` console
(gdb) monitor arm semihosting enable
semihosting is enabled
```

QEMU理解semihosting操作,所以以前的程序可以在`qemu-system-arm`上运行,而不需要启动一个调试会话.注意你需要传递`-semihosting-config`参数来让QEMU启用semihosting支持;这些参数已经包含在`.cargo/config`文件中.

``` console
$ # this program will block the terminal
$ cargo run
     Running `qemu-system-arm (..)
Hello, world!
```

还有一个`exit`的semihosting操作, 用来结束QEMU进程.
重要提示: 请**不要**在硬件上使用`debug::exit`, 此函数会破坏OpenOCD会话, 只有重启才能继续调试.

```rust,ignore
#![no_main]
#![no_std]

use panic_halt as _;

use cortex_m_rt::entry;
use cortex_m_semihosting::debug;

#[entry]
fn main() -> ! {
    let roses = "blue";

    if roses == "red" {
        debug::exit(debug::EXIT_SUCCESS);
    } else {
        debug::exit(debug::EXIT_FAILURE);
    }

    loop {}
}
```

``` console
$ cargo run
     Running `qemu-system-arm (..)

$ echo $?
1
```

最后一个提示: 你可以将panic行为设置为`exit(EXIT_FAILURE)`. 这可以让你写的`no_std`测试在QEMU上运行.

为了方便, `panic-semihosting`有一个"exit"的feature, 启用它可以在`exit(EXIT_FAILURE)`后把panic信息输出到主机stderr上.

```rust,ignore
#![no_main]
#![no_std]

use panic_semihosting as _; // features = ["exit"]

use cortex_m_rt::entry;
use cortex_m_semihosting::debug;

#[entry]
fn main() -> ! {
    let roses = "blue";

    assert_eq!(roses, "red");

    loop {}
}
```

``` console
$ cargo run
     Running `qemu-system-arm (..)
panicked at 'assertion failed: `(left == right)`
  left: `"blue"`,
 right: `"red"`', examples/hello.rs:15:5

$ echo $?
1
```

**注意**: 编辑你的`Cargo.toml`以启用`panic-semihosting`的该特性:

``` toml
panic-semihosting = { version = "VERSION", features = ["exit"] }
```

有关feature的更多信息请参考[`specifying dependencies`]

[`specifying dependencies`]:
https://doc.rust-lang.org/cargo/reference/specifying-dependencies.html