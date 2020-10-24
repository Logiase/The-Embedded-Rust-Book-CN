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

除了我们调用`tm4c123x::Peripherals::take()`外,我们使用`PWM0`外设的方法是和`SYST`相同的.
因为此库是使用[svd2rust]自动生成的,所以我们访问寄存器需要闭包参数,而不是数字参数.
尽管这看起来很多,但是rust编译器会执行一堆检查,然后生成的机器码与我们手写的汇编非常接近!
自动生成的代码无法确定特定寄存器的所有参数(例如,如果SVD定义寄存器有32bit,但并没有说明其中的某些位有什么特殊功能),所以被标记为`unsafe`.
我们可以在上面这个例子中看到如何使用`bits()`的子函数`load`,`compa`.

### 读取

`read()`函数会返回一个包含有制造商SVD文件定义的寄存器各个子段的只读权限的对象.
你可以在[tm4c123x documentation][tm4c123x documentation R]中特定外设,特定寄存器的特殊`R`返回值类型中的所有可用函数.

```rust,ignore
if pwm.ctl.read().globalsync0().is_set() {
    // Do a thing
}
```

### 写入

`write()`函数需要一个只有一个参数的闭包参数.我们叫他`w`.
这个参数有该设备制造商SVD文件定义的寄存器所有子段的读写权限.
你也可以在[tm4c123x documentation][tm4c123x Documentation W]中找到针对该芯片该外设该寄存器`w`的所有函数.
请注意,我们未设置的所有子字段都将被设置为我们的默认值-寄存器中的所有现有内容都将丢失.

```rust,ignore
pwm.ctl.write(|w| w.globalsync0().clear_bit());
```

### 修改

如果我们想修改寄存器中某一子段的值而不修改其他的,我们可以使用`modify()`函数.
该函数需要一个包括两个参数的闭包参数,一个用来读,一个用来写.我们经常叫`r`和`w`.
`r`可以用来查看当前寄存器中的内容,`w`可以用来修改寄存器中的值.

```rust,ignore
pwm.ctl.modify(|r, w| w.globalsync0().clear_bit());
```

`modify`函数在这真的展现了闭包的强大.在`C`中,我们先要把值读取到几个临时变量中,然后做修改,然后再写回去.这意味着会存在很大错误范围:

```C
uint32_t temp = pwm0.ctl.read();
temp |= PWM0_CTL_GLOBALSYNC0;
pwm0.ctl.write(temp);
uint32_t temp2 = pwm0.enable.read();
temp2 |= PWM0_ENABLE_PWM4EN;
pwm0.enable.write(temp); // Uh oh! Wrong variable!
```

[svd2rust]: https://crates.io/crates/svd2rust
[tm4c123x documentation R]: https://docs.rs/tm4c123x/0.7.0/tm4c123x/pwm0/ctl/struct.R.html
[tm4c123x documentation W]: https://docs.rs/tm4c123x/0.7.0/tm4c123x/pwm0/ctl/struct.W.html

## 使用 HAL(硬件抽象层) 库

芯片的HAL库通常通过为PAC暴露的原始结构来实现自定义trait.通常这个trait会为单独的外设定义一个叫`constrain()`的函数,为类似GPIO这样有多个引脚的外设定义`split()`函数.此函数包装最原始的结构,然后提供拥有一个高级的API的对象.
这个API可以做很多事情,例如串口的`new`需要借用`Clock`结构,`Clock`只能通过配置PLL设置时钟频率获得.
通过这种方法,在没有创建配置时钟或没法将波特率与始终速率对应起来之前没法创建一个串口对象.
一些库甚至为GPIO引脚定义了特殊的trait,需要用户选择引脚的正确状态(或者说,选择合适的复用功能).都不要运行时花销.

让我们看个例子:

```rust,ignore
#![no_std]
#![no_main]

use panic_halt as _; // panic handler

use cortex_m_rt::entry;
use tm4c123x_hal as hal;
use tm4c123x_hal::prelude::*;
use tm4c123x_hal::serial::{NewlineMode, Serial};
use tm4c123x_hal::sysctl;

#[entry]
fn main() -> ! {
    let p = hal::Peripherals::take().unwrap();
    let cp = hal::CorePeripherals::take().unwrap();

    // Wrap up the SYSCTL struct into an object with a higher-layer API
    let mut sc = p.SYSCTL.constrain();
    // Pick our oscillation settings
    sc.clock_setup.oscillator = sysctl::Oscillator::Main(
        sysctl::CrystalFrequency::_16mhz,
        sysctl::SystemClock::UsePll(sysctl::PllOutputFrequency::_80_00mhz),
    );
    // Configure the PLL with those settings
    let clocks = sc.clock_setup.freeze();

    // Wrap up the GPIO_PORTA struct into an object with a higher-layer API.
    // Note it needs to borrow `sc.power_control` so it can power up the GPIO
    // peripheral automatically.
    let mut porta = p.GPIO_PORTA.split(&sc.power_control);

    // Activate the UART.
    let uart = Serial::uart0(
        p.UART0,
        // The transmit pin
        porta
            .pa1
            .into_af_push_pull::<hal::gpio::AF1>(&mut porta.control),
        // The receive pin
        porta
            .pa0
            .into_af_push_pull::<hal::gpio::AF1>(&mut porta.control),
        // No RTS or CTS required
        (),
        (),
        // The baud rate
        115200_u32.bps(),
        // Output handling
        NewlineMode::SwapLFtoCRLF,
        // We need the clock rates to calculate the baud rate divisors
        &clocks,
        // We need this to power up the UART peripheral
        &sc.power_control,
    );

    loop {
        writeln!(uart, "Hello, World!\r\n").unwrap();
    }
}
```
