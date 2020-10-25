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

