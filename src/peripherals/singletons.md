# 单例

> 在软件工程中, 单例模式是一种限制一个类只能存在一种的设计模式
>
> *Wikipedia: [Singleton Pattern]*

[Singleton Pattern]: https://en.wikipedia.org/wiki/Singleton_pattern

## 但是我们为什么直接用全局变量?

我们可以把任何都设成一个全局变量, 像这样:

```rust
static mut THE_SERIAL_PORT: SerialPort = SerialPort;

fn main() {
    let _ = unsafe {
        THE_SERIAL_PORT.read_speed();
    };
}
```

但是这有一些问题. 在 Rust 中, 与全局变量交互是 `unsafe` 的. 这些变量始终对你的程序可见, 这意味这引用检查器不能帮你追踪引用与所有权.

## 我们在 Rust 中怎么做?

代替将外设做为全局变量, 我们决定创建一个叫做 `PERIPHERALS` 的全局变量, 它包含我们每个外设的可空引用 `Option<T>`.

```rust,ignore
struct Peripherals {
    serial: Option<SerialPort>,
}
impl Peripherals {
    fn take_serial(&mut self) -> SerialPort {
        let p = replace(&mut self.serial, None);
        p.unwrap()
    }
}
static mut PERIPHERALS: Peripherals = Peripherals {
    serial: Some(SerialPort),
};
```

这个结构允许我们获取每个外设的单一实例. 如果我们多次使用 `take_serail()` , 我们的程序就会 panic !

```rust,ignore
fn main() {
    let serial_1 = unsafe { PERIPHERALS.take_serial() };
    // This panics!
    // let serial_2 = unsafe { PERIPHERALS.take_serial() };
}
```

尽管这么交互还是 `unsafe` , 但是我们一旦拿到它持有的 `SerialPort` , 我们就不需要在使用 `unsafe` 或者 `PERIPHERALS` 了.

这有很小的运行时开销, 因为我们必须将 `SerialPort` 包装在一个 `Option<T>` 中, 并且需要调用一次 `take_serial()`, 但是, 这一点点成本能够让我们在剩余所有过程中使用引用检查器来检查我们的程序.

## 已有的库支持

尽管我们在前面创建了我们自己的 `Peripherals` , 但是你没必要再自己的代码中这么些, `cortex-m` 库中包含了一个叫 `singleton!()` 的宏, 它会帮你.

```rust
#[macro_use(singleton)]
extern crate cortex_m;

fn main() {
    // OK if `main` is executed only once
    let x: &'static mut bool =
        singleton!(: bool = false).unwrap();
}
```

[cortex_m docs](https://docs.rs/cortex-m/latest/cortex_m/macro.singleton.html)

另外, 如果你使用[`cortex-m-rtic`](https://github.com/rtic-rs/cortex-m-rtic), 那它会帮你抽象这个定义和获取外围设备的步骤, 直接给你外设, 而不是你定义的 `Option<T>`.

```rust
// cortex-m-rtic v0.5.x
#[rtic::app(device = lm3s6965, peripherals = true)]
const APP: () = {
    #[init]
    fn init(cx: init::Context) {
        static mut X: u32 = 0;
         
        // Cortex-M peripherals
        let core: cortex_m::Peripherals = cx.core;
        
        // Device specific peripherals
        let device: lm3s6965::Peripherals = cx.device;
    }
}
```

## 但是为什么?

但是这些单例如何在我们的代码中产生明显的不同?

```rust
impl SerialPort {
    const SER_PORT_SPEED_REG: *mut u32 = 0x4000_1000 as _;

    fn read_speed(
        &self // <------ This is really, really important
    ) -> u32 {
        unsafe {
            ptr::read_volatile(Self::SER_PORT_SPEED_REG)
        }
    }
}
```

这有两个重要因素:

* 因为哦我们在使用单例, 所以我们只有一种方法来获取一个 `SerialPort`
* 为了使用 `read_speed()` 函数, 我们必须有 `SerialPort`的所有权或他的引用

这两个因素加在一起意味着我们只有在满足条件的情况下才能访问硬件, 意味着我们在任何时候都不能对同一硬件有多个可变引用!

```rust
fn main() {
    // missing reference to `self`! Won't work.
    // SerialPort::read_speed();

    let serial_1 = unsafe { PERIPHERALS.take_serial() };

    // you can only read what you have access to
    let _ = serial_1.read_speed();
}
```

## 把你的硬件看成数据

另外, 由于某些引用是可变的, 有些是不可变的, 因此可以查看某个函数或方法时候有潜在的可能修改硬件的状态. 例如:

这允许修改硬件设置:

```rust
fn setup_spi_port(
    spi: &mut SpiPort,
    cs_pin: &mut GpioPin
) -> Result<()> {
    // ...
}
```

这不允许:

```rust,ignore
fn read_button(gpio: &GpioPin) -> bool {
    // ...
}
```

者能够让我们在**编译时**(而不是运行时)确定代码是否能够修改硬件状态. 需要注意的是, 这通常仅仅在一个应用中可行, 但是对于裸金属系统, 我们的代码通常只会编译为一个应用, 所以不受限制.
