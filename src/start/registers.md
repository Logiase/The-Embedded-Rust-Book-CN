# 内存映射寄存器

嵌入式系统只能通过执行常规的Rust代码并在RAM中移动数据来达到目标.
如果我们想从外部读取信息到系统,或从系统中获取信息(例如,点亮LED,检测到按钮按下,或者与总线上某种设备进行通信),那我们必须要接触外设和"内存映射寄存器"

您可能会发现,已经在以下级别之一编写了访问微控制器外围设备所需的代码:

<p align="center">
<img title="Common crates" src="../assets/crates.png">
</p>

- Micro-architecture Crate - 这种库可处理你使用的mcu的通用部分,以及使用该内核的所有mcu的通用的外设.例如,[cortex-m]可以提供启用禁用中断的功能,这些功能对所有Cortex-m处理器都适用.他还可以让你访问所有基于Cortex-m微控制器所带的'SysTick'外设.
- Peripheral Access Crate (PAC) - 这种库是根据你的mcu型号来提供一个对内存包装寄存器的简单包装.例如[tm4c123x]对应Texas Instruments Tiva-C TM4C123系列,[stm32f30x]对应ST-Micro STM32F30x系列.在这里你将按照mcu的参考手册中给出的每个外设的操作说明直接操作寄存器.
- HAL Crate - 这些库给实现[embedded-hal]的一些trait,来为你的mcu提供一个更加用户友好的API.例如,这些库可能会提供一个`Serial` Struct,它的构造函数使用适当GPIO与波特率,并提供某种`write_byte`函数来发送数据.有关嵌入式HAL的更多信息,请参考[移植]
- Board Crate - 这些库通过预先配置好的外设和GPIO引脚来让你使用特定的开发板,像是[stm32f3-discovery]对STM32F3DISCOVERY,这些库比HAL库更进一步.

[cortex-m]: https://crates.io/crates/cortex-m
[tm4c123x]: https://crates.io/crates/tm4c123x
[stm32f30x]: https://crates.io/crates/stm32f30x
[embedded-hal]: https://crates.io/crates/embedded-hal
[移植]: ../portability/index.md
[stm32f3-discovery]: https://crates.io/crates/stm32f3-discovery
[Discovery]: https://rust-embedded.github.io/discovery/

## Board Crate

如果你在嵌入式系统方面是个萌新,拿使用Board Crate是一个很好的起点.
他们很好的抽象了我们在学习过程中会遇到的硬件细节,并简化像是开关LED的操作.
他们暴露的函数在不同开发板之间差别很大.由于本书旨在不涉及硬件的细节,所以本书不会使用board crate.

如果你想使用STM32F3DISCOVERY进行试验,那很推荐你去看一看[stm32f3-discovery] board crate,这个库提供了一些列功能,包括开关LED,使用指南针,蓝牙等.
[Discovery]这本书提供了一个使用这个board crate很好的介绍.

但是如果你使用一个没有board crate的系统,或者你需要使用现有board crate没有提供的功能,请从底部开始阅读micro-architecture.

## Micro-architecture crate

让我们看一下所有基于Cortex-M的微控制器共有的SysTick外设.我们可以在[cortex-m]中找到一个非常非常低级的API,我们能这么用:

```rust,ignore
#![no_std]
#![no_main]
use cortex_m::peripheral::{syst, Peripherals};
use cortex_m_rt::entry;
use panic_halt as _;

#[entry]
fn main() -> ! {
    let peripherals = Peripherals::take().unwrap();
    let mut systick = peripherals.SYST;
    systick.set_clock_source(syst::SystClkSource::Core);
    systick.set_reload(1_000);
    systick.clear_current();
    systick.enable_counter();
    while !systick.has_wrapped() {
        // Loop
    }

    loop {}
}
```

SYST struct的函数与ARM Technical Reference Manual定义的很相似.
此API中没有没有关于`延迟X毫秒`的函数 - 我们得使用`while`循环
来大致的实现这个功能.注意,我们在调用`Peripherals::take()`函数前,
我们没法使用`SYST` - 这是一个特殊的历程,可以确保整个程序中只有一个`SYST`.
关于更多,可以参考[Peripherals]章节

[Peripherals]: ../peripherals/index.md

## 使用Peripheral Access Crate (PAC)

如果我们把自己束缚在Cortex-M自带的基本外设上,那注定我们的嵌入式之路是走不远的.
在某个时候,我们需要编写一些特定于我们正在使用的硬件的代码.
在这个实例中,先假设我们有一个德州仪器(TI)的TM4C123,一个有256KiB闪存,80MHz的中等的Cortex-M4微控制器.
我们打算使用[tm4c123x]库来玩这块芯片.

```rust,ignore
#![no_std]
#![no_main]

use panic_halt as _; // panic handler

use cortex_m_rt::entry;
use tm4c123x;

#[entry]
pub fn init() -> (Delay, Leds) {
    let cp = cortex_m::Peripherals::take().unwrap();
    let p = tm4c123x::Peripherals::take().unwrap();

    let pwm = p.PWM0;
    pwm.ctl.write(|w| w.globalsync0().clear_bit());
    // Mode = 1 => Count up/down mode
    pwm._2_ctl.write(|w| w.enable().set_bit().mode().set_bit());
    pwm._2_gena.write(|w| w.actcmpau().zero().actcmpad().one());
    // 528 cycles (264 up and down) = 4 loops per video line (2112 cycles)
    pwm._2_load.write(|w| unsafe { w.load().bits(263) });
    pwm._2_cmpa.write(|w| unsafe { w.compa().bits(64) });
    pwm.enable.write(|w| w.pwm4en().set_bit());
}

```
