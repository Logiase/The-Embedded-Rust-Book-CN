# 零成本抽象

类型状态是零成本抽象的一个非常棒的例子, 将一些确定的行为转移到编译步骤. 这些状态不包含实际数据, 而是用来标记. 因为他们不包含数据, 所以在运行时, 他们在内存中并不存在.

```rust
use core::mem::size_of;

let _ = size_of::<Enabled>();    // == 0
let _ = size_of::<Input>();      // == 0
let _ = size_of::<PulledHigh>(); // == 0
let _ = size_of::<GpioConfig<Enabled, Input, PulledHigh>>(); // == 0
```

## 零大小类型

```rust
struct Enabled;
```

像这样定义的类型叫做 Zero Sized Types (零大小类型), 因为他们并不包含实际的数据. 尽管这些类型在编译时实际存在, 你能复制, 移动, 引用他们. 但是优化器会完完全全的删掉他们.

在这段代码中:

```rust
pub fn into_input_high_z(self) -> GpioConfig<Enabled, Input, HighZ> {
    self.periph.modify(|_r, w| w.input_mode().high_z());
    GpioConfig {
        periph: self.periph,
        enabled: Enabled,
        direction: Input,
        mode: HighZ,
    }
}
```

我们返回的 `GpioConfig` 在运行时是不存在的. 调用此函数通常会简化为单个汇编指令 - 把一个固定的值写入一个固定的寄存器. 这意味着我们开发的状态类型接口是一个零成本抽象, 他不占用 CPU, RAM, 或代码空间来追踪 `GpioConfig` 的状态然后呈现为与直接访问寄存器相同的汇编代码.

## 嵌套

通常来说, 这些抽象你可以随便套, 只要使用的都是零大小类型, 整个结构在运行时就不会存在.

对于复杂或深度嵌套的结构, 定义所有的状态组合会非常麻烦, 在这种情况下, 使用宏会很舒服.
