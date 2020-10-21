# 硬件

现在你应该熟悉了工具与开发过程.
在本节我们来试试真正的硬件.
该过程基本不变.让我们开始:

## 了解你的硬件

在我们开始之前你需要了解硬件的特点以便于配置项目:

- ARM内核. e.g. Cortex-M3.
- ARM内核有FPU吗? Cortex-M4**F**和Cortex-M7**F**有.
- 目标设备有多大闪存和RAM? e.g. 256KiB闪存32KiB内存.
- 闪存和RAM在的地址在多少? e.g. RAM通常位于`0x2000_0000`.

你可以在用户手册和数据手册中找到这些信息.

在本届我们使用我们的参考硬件STM32F3DISCOVERY.
这块板子有一个STM32F303VCT6.这块MCU有:

- 一个带有单精度FPU的Cortex-M4F内核
- 位于0x0800_0000的256KiB闪存
- 位于0x2000_0000的40KiB内存(还有另一个RAM区域,为了简单我们忽略)

## 配置

我们从一个新的模板实例开始.
关于如何使用`cargo-generate`请参考[上一章节QEMU]

[上一章节QEMU]: qemu.md

``` console
$ cargo generate --git https://github.com/rust-embedded/cortex-m-quickstart
 Project Name: app
 Creating project called `app`...
 Done! New project created /tmp/app

$ cd app
```

第一步是在`.cargo/config`中设置默认编译目标.

``` console
$ tail -n5 .cargo/config
```

``` toml
# Pick ONE of these compilation targets
# target = "thumbv6m-none-eabi"    # Cortex-M0 and Cortex-M0+
# target = "thumbv7m-none-eabi"    # Cortex-M3
# target = "thumbv7em-none-eabi"   # Cortex-M4 and Cortex-M7 (no FPU)
target = "thumbv7em-none-eabihf" # Cortex-M4F and Cortex-M7F (with FPU)
```

我们使用`thumbv7em-none-eabihf`,因为它包含Cortex-M4F内核.

第二步是把内存区域信息输入到`memory.x`中.

``` console
$ cat memory.x
/* Linker script for the STM32F303VCT6 */
MEMORY
{
  /* NOTE 1 K = 1 KiBi = 1024 bytes */
  FLASH : ORIGIN = 0x08000000, LENGTH = 256K
  RAM : ORIGIN = 0x20000000, LENGTH = 40K
}
```

> **NOTE**如果你因为某些原因修改了`memory.x`,并且之前做过了编译,
> 那你需要执行`cargo clean`再执行`cargo build`,因为`cargo build`并不会追踪`memory.x`的变化.

我们还用hello这个例子, 但是首先先做一点小改动.

在`examples/hello.rs`中,确保`debug::exit()`被注释掉或者删掉.它只是为了运行QEMU而存在的.

```rust,ignore
#[entry]
fn main() -> ! {
    hprintln!("Hello, world!").unwrap();

    // exit QEMU
    // NOTE do not run this on hardware; it can corrupt OpenOCD state
    // debug::exit(debug::EXIT_SUCCESS);

    loop {}
}
```

现在你可以用`cargo build`进行交叉编译,并且像之前一样用`cargo-binutils`查看信息.
`cortex-m-rt`这个库包含了一切能让你芯片运行的魔法,它很有帮助,因为几乎所有的Cortex-M CPU都可以用相同的方式引导.

``` console
$ cargo build --example hello
```

## 调试

调试过程看起来有些不同了.事实上,第一步根据目标设备不同也有不同.这一节中我们会展示在STM32DISCOBVERY上debug的步骤.这仅供参考,有关设备的调试请参考[the Debugonomicon](https://github.com/rust-embedded/debugonomicon).

和以前一样,我们进行远程调试,客户端是GDB.但是这次服务端则是OpenOCD.

像之前在[验证安装]中所做的一样,将板子连接到电脑,然后检查ST-LINK.

[验证安装]: ../intro/install/verify.md

在终端上运行OpenOCD以连接到ST-LINK.从模板的根目录运行此命令;`OpenOCD`会使用`openocd.cfg`,这里面声明了使用什么接口,连接什么设备.

``` console
$ cat openocd.cfg
```

``` text
# Sample OpenOCD configuration for the STM32F3DISCOVERY development board

# Depending on the hardware revision you got you'll have to pick ONE of these
# interfaces. At any time only one interface should be commented out.

# Revision C (newer revision)
source [find interface/stlink.cfg]

# Revision A and B (older revisions)
# source [find interface/stlink-v2.cfg]

source [find target/stm32f3x.cfg]
```

> **NOTE** 如果你在用旧版本的DISCOVERY板子,你应该修改一下`openocd.cfg`来使用`interface/stlink-v2.cfg`

``` console
$ openocd
Open On-Chip Debugger 0.10.0
Licensed under GNU GPL v2
For bug reports, read
        http://openocd.org/doc/doxygen/bugs.html
Info : auto-selecting first available session transport "hla_swd". To override use 'transport select <transport>'.
adapter speed: 1000 kHz
adapter_nsrst_delay: 100
Info : The selected transport took over low-level target control. The results might differ compared to plain JTAG/SWD
none separate
Info : Unable to match requested speed 1000 kHz, using 950 kHz
Info : Unable to match requested speed 1000 kHz, using 950 kHz
Info : clock speed 950 kHz
Info : STLINK v2 JTAG v27 API v2 SWIM v15 VID 0x0483 PID 0x374B
Info : using stlink api v2
Info : Target voltage: 2.913879
Info : stm32f3x.cpu: hardware has 6 breakpoints, 4 watchpoints
```

同样在根目录下另起一个终端运行GDB.

``` console
$ <gdb> -q target/thumbv7em-none-eabihf/debug/examples/hello
```

先一步连接GDB到OpenOCD.

``` console
(gdb) target remote :3333
Remote debugging using :3333
0x00000000 in ?? ()
```

现在使用`load`命令*烧录*程序到mcu.

``` console
(gdb) load
Loading section .vector_table, size 0x400 lma 0x8000000
Loading section .text, size 0x1e70 lma 0x8000400
Loading section .rodata, size 0x61c lma 0x8002270
Start address 0x800144e, load size 10380
Transfer rate: 17 KB/sec, 3460 bytes/write.
```

现在程序被加载了.之前程序使用semihosting,因此在我们进行任何semihosting操作时,应该先告诉OpenOCD启用semihosting.可以使用`monitor`命令.

``` console
(gdb) monitor arm semihosting enable
semihosting is enabled
```

> 你也可以使用`monitor help`查看所用OpenOCD命令.

前之前那样给`main`加断点并执行`continue`

``` console
(gdb) break main
Breakpoint 1 at 0x8000d18: file examples/hello.rs, line 15.

(gdb) continue
Continuing.
Note: automatically using hardware breakpoints for read-only addresses.

Breakpoint 1, main () at examples/hello.rs:15
15          let mut stdout = hio::hstdout().unwrap();
```

> **NOTE** 如果在发出上面的`continue`命令后GDB阻塞了终端而不是到达断点，则你可能要仔细检查一下是否已为您的设备正确设置了`memory.x`文件中的存储区域信息(起始位置和长度).

使用`next`继续程序,应该会有和之前相同的结果.

``` console
(gdb) next
16          writeln!(stdout, "Hello, world!").unwrap();

(gdb) next
19          debug::exit(debug::EXIT_SUCCESS);
```

在这我们应该看到在OpenOCD的控制台上出现了"Hello, world!"

``` console
$ openocd
(..)
Info : halted: PC: 0x08000e6c
Hello, world!
Info : halted: PC: 0x08000d62
Info : halted: PC: 0x08000d64
Info : halted: PC: 0x08000d66
Info : halted: PC: 0x08000d6a
Info : halted: PC: 0x08000a0c
Info : halted: PC: 0x08000d70
Info : halted: PC: 0x08000d72
```
