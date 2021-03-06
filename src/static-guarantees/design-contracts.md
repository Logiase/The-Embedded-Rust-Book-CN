# 设计合同

再上一章, 我们写了一个*不符合*设计合同的接口. 让我们再看一下我们假设的 GPIO 寄存器配置:

| Name         | Bit Number(s) | Value | Meaning   | Notes |
| ---:         | ------------: | ----: | ------:   | ----: |
| enable       | 0             | 0     | disabled  | Disables the GPIO |
|              |               | 1     | enabled   | Enables the GPIO |
| direction    | 1             | 0     | input     | Sets the direction to Input |
|              |               | 1     | output    | Sets the direction to Output |
| input_mode   | 2..3          | 00    | hi-z      | Sets the input as high resistance |
|              |               | 01    | pull-low  | Input pin is pulled low |
|              |               | 10    | pull-high | Input pin is pulled high |
|              |               | 11    | n/a       | Invalid state. Do not set |
| output_mode  | 4             | 0     | set-low   | Output pin is driven low |
|              |               | 1     | set-high  | Output pin is driven high |
| input_status | 5             | x     | in-val    | 0 if input is < 1.5v, 1 if input >= 1.5v |

如果我们改为在使用硬件前先检查状态, 在运行时强制执行我们的设计合同, 我们可能会写出如下的替代:

```rust
/// GPIO interface
struct GpioConfig {
    /// GPIO Configuration structure generated by svd2rust
    periph: GPIO_CONFIG,
}

impl GpioConfig {
    pub fn set_enable(&mut self, is_enabled: bool) {
        self.periph.modify(|_r, w| {
            w.enable().set_bit(is_enabled)
        });
    }

    pub fn set_direction(&mut self, is_output: bool) -> Result<(), ()> {
        if self.periph.read().enable().bit_is_clear() {
            // Must be enabled to set direction
            return Err(());
        }

        self.periph.modify(|r, w| {
            w.direction().set_bit(is_output)
        });

        Ok(())
    }

    pub fn set_input_mode(&mut self, variant: InputMode) -> Result<(), ()> {
        if self.periph.read().enable().bit_is_clear() {
            // Must be enabled to set input mode
            return Err(());
        }

        if self.periph.read().direction().bit_is_set() {
            // Direction must be input
            return Err(());
        }

        self.periph.modify(|_r, w| {
            w.input_mode().variant(variant)
        });

        Ok(())
    }

    pub fn set_output_status(&mut self, is_high: bool) -> Result<(), ()> {
        if self.periph.read().enable().bit_is_clear() {
            // Must be enabled to set output status
            return Err(());
        }

        if self.periph.read().direction().bit_is_clear() {
            // Direction must be output
            return Err(());
        }

        self.periph.modify(|_r, w| {
            w.output_mode.set_bit(is_high)
        });

        Ok(())
    }

    pub fn get_input_status(&self) -> Result<bool, ()> {
        if self.periph.read().enable().bit_is_clear() {
            // Must be enabled to get status
            return Err(());
        }

        if self.periph.read().direction().bit_is_set() {
            // Direction must be input
            return Err(());
        }

        Ok(self.periph.read().input_status().bit_is_set())
    }
}
```

因为我们给硬件加了强制约束, 所以在结束的时候要进行大量的运行时检查, 这浪费时间又浪费性能, 并且让人看着难受.

## 状态类型

但是如果反过来, 我们使用 Rust 的类型系统来执行转换规则的话, 看看这个例子:

```rust
/// GPIO interface
struct GpioConfig<ENABLED, DIRECTION, MODE> {
    /// GPIO Configuration structure generated by svd2rust
    periph: GPIO_CONFIG,
    enabled: ENABLED,
    direction: DIRECTION,
    mode: MODE,
}

// Type states for MODE in GpioConfig
struct Disabled;
struct Enabled;
struct Output;
struct Input;
struct PulledLow;
struct PulledHigh;
struct HighZ;
struct DontCare;

/// These functions may be used on any GPIO Pin
impl<EN, DIR, IN_MODE> GpioConfig<EN, DIR, IN_MODE> {
    pub fn into_disabled(self) -> GpioConfig<Disabled, DontCare, DontCare> {
        self.periph.modify(|_r, w| w.enable.disabled());
        GpioConfig {
            periph: self.periph,
            enabled: Disabled,
            direction: DontCare,
            mode: DontCare,
        }
    }

    pub fn into_enabled_input(self) -> GpioConfig<Enabled, Input, HighZ> {
        self.periph.modify(|_r, w| {
            w.enable.enabled()
             .direction.input()
             .input_mode.high_z()
        });
        GpioConfig {
            periph: self.periph,
            enabled: Enabled,
            direction: Input,
            mode: HighZ,
        }
    }

    pub fn into_enabled_output(self) -> GpioConfig<Enabled, Output, DontCare> {
        self.periph.modify(|_r, w| {
            w.enable.enabled()
             .direction.output()
             .input_mode.set_high()
        });
        GpioConfig {
            periph: self.periph,
            enabled: Enabled,
            direction: Output,
            mode: DontCare,
        }
    }
}

/// This function may be used on an Output Pin
impl GpioConfig<Enabled, Output, DontCare> {
    pub fn set_bit(&mut self, set_high: bool) {
        self.periph.modify(|_r, w| w.output_mode.set_bit(set_high));
    }
}

/// These methods may be used on any enabled input GPIO
impl<IN_MODE> GpioConfig<Enabled, Input, IN_MODE> {
    pub fn bit_is_set(&self) -> bool {
        self.periph.read().input_status.bit_is_set()
    }

    pub fn into_input_high_z(self) -> GpioConfig<Enabled, Input, HighZ> {
        self.periph.modify(|_r, w| w.input_mode().high_z());
        GpioConfig {
            periph: self.periph,
            enabled: Enabled,
            direction: Input,
            mode: HighZ,
        }
    }

    pub fn into_input_pull_down(self) -> GpioConfig<Enabled, Input, PulledLow> {
        self.periph.modify(|_r, w| w.input_mode().pull_low());
        GpioConfig {
            periph: self.periph,
            enabled: Enabled,
            direction: Input,
            mode: PulledLow,
        }
    }

    pub fn into_input_pull_up(self) -> GpioConfig<Enabled, Input, PulledHigh> {
        self.periph.modify(|_r, w| w.input_mode().pull_high());
        GpioConfig {
            periph: self.periph,
            enabled: Enabled,
            direction: Input,
            mode: PulledHigh,
        }
    }
}
```

现在让我们看看这段代码怎么用:

```rust
/*
 * Example 1: Unconfigured to High-Z input
 */
let pin: GpioConfig<Disabled, _, _> = get_gpio();

// Can't do this, pin isn't enabled!
// pin.into_input_pull_down();

// Now turn the pin from unconfigured to a high-z input
let input_pin = pin.into_enabled_input();

// Read from the pin
let pin_state = input_pin.bit_is_set();

// Can't do this, input pins don't have this interface!
// input_pin.set_bit(true);

/*
 * Example 2: High-Z input to Pulled Low input
 */
let pulled_low = input_pin.into_input_pull_down();
let pin_state = pulled_low.bit_is_set();

/*
 * Example 3: Pulled Low input to Output, set high
 */
let output_pin = pulled_low.into_enabled_output();
output_pin.set_bit(true);

// Can't do this, output pins don't have this interface!
// output_pin.into_input_pull_down();
```

这绝对是存储引脚状态的方便方法, 但是我们为什么要这么做? 为什么这么做比我们写一个 `GpioConfig` 的 `enum` 要好?

## 编译时函数安全

因为我们在编译时完全执行设计约束, 所以不会产生运行时成本. 当引脚处于输入状态时, 无法设置输出模式. 相反, 你必须通过改变状态来把它转换为输出引脚, 然后设置输出模式. 正因为如此, 在编译时检查状态, 不会造成运行时的性能损失.

而且, 因为这些状态是由类型系统强制约束的, 所以使用者不会出错, 如果他们尝试做一些非法的状态转换, 编译就无法通过!
