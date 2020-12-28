# 第一次尝试

## 寄存器

我们来看看 `SysTick` 这个外设-每个 Cortex-M 处理器内核都有的简单计时器. 通常情况下, 你可以在芯片制造商的*数据手册*中找到这些信息, 但是此示例对于所有的 Arm Cortex-M 内核都是通用的, 让我们在[ARM参考手册]中进行查找. 我们看到有四个寄存器:

[ARM参考手册]: http://infocenter.arm.com/help/topic/com.arm.doc.dui0553a/Babieigh.html

| Offset | Name        | Description                 | Width  |
|--------|-------------|-----------------------------|--------|
| 0x00   | SYST_CSR    | Control and Status Register | 32 bits|
| 0x04   | SYST_RVR    | Reload Value Register       | 32 bits|
| 0x08   | SYST_CVR    | Current Value Register      | 32 bits|
| 0x0C   | SYST_CALIB  | Calibration Value Register  | 32 bits|

## C怎么做

在Rust中, 我们可以使用与 C 完全相同的方式来表示寄存器的合集-使用 `struct` .

```rust
#[repr(C)]
struct SysTick {
    pub csr: u32,
    pub rvr: u32,
    pub cvr: u32,
    pub calib: u32,
}
```

限定符 `#[repr(C)]` 告诉 Rust 编译器像 C 一样对这个 struct 布局. 这非常重要, 因为 Rust 会对 struct 的字段进行重新排序, 而 C 不会. 你可以想象一下, 如果 Rust 对其进行了重新排序, 我们进行调试找 BUG 会多难! 使用这个限定符之后, 我们就有四个32位字段, 它们与上表相对应. 但是, 当然仅有这个 `struct` 是不够的, 我们还需要一个变量.

```rust
let systick = 0xE000_E010 as *mut SysTick;
let time = unsafe { (*systick).cvr };
```

## 有序访问

现在, 我们碰到了一堆问题.

1. 我们需要用到 unsafe 来访问我们的外设.
2. 我们没有办法确定那个寄存器是只读的或可读可写的.
3. 代码中的任何部分都可以通过这个结构来访问硬件.
4. 最重要的, 现在它还不能用...

现在的问题是编译器很聪明. 如果你向同一块内存做两次写入, 一前一后, 编译器会注意到这个操作, 然后优化掉第一次写入. 在 C 中, 我们可以把这个变量标记为 `volatile` 来确保每次读写操作都会准确发生. 在 Rust 中, 我们将 *指针* 标记为 volatile, 而不是变量.

```rust
let systick = unsafe { &mut *(0xE000_E010 as *mut SysTick) };
let time = unsafe { core::ptr::read_volatile(&mut systick.cvr) };
```

所以, 我们修复了上面四个问题之一, 但是我们却写出了更 `unsafe` 的代码! 幸运的是, 这由第三方的库能帮我们 - [`volatile_register`].

[`volatile_register`]: https://crates.io/crates/volatile_register

```rust
use volatile_register::{RW, RO};

#[repr(C)]
struct SysTick {
    pub csr: RW<u32>,
    pub rvr: RW<u32>,
    pub cvr: RW<u32>,
    pub calib: RO<u32>,
}

fn get_systick() -> &'static mut SysTick {
    unsafe { &mut *(0xE000_E010 as *mut SysTick) }
}

fn get_time() -> u32 {
    let systick = get_systick();
    systick.cvr.read()
}
```

现在, 读取写入都通过 `read` 和 `write` 安排妥当了. 虽然写入还是 `unsafe` , 不过现在, 硬件现在是一堆可变的状态, 编译器没法知道这些操作是不是安全的, 所以这是一个不错的默认位置.

## Rust风格的外壳

我们需要用一个更高级的 API 来封装一下这个 `struct` 来让能让我们安全使用. 作为驱动作者, 我们人工确定 `unsafe` 代码是否正确, 然后给我们的用户提供一个 `safe` 的 API来让我们的用户不去担心这些.

一个栗子:

```rust
use volatile_register::{RW, RO};

pub struct SystemTimer {
    p: &'static mut RegisterBlock
}

#[repr(C)]
struct RegisterBlock {
    pub csr: RW<u32>,
    pub rvr: RW<u32>,
    pub cvr: RW<u32>,
    pub calib: RO<u32>,
}

impl SystemTimer {
    pub fn new() -> SystemTimer {
        SystemTimer {
            p: unsafe { &mut *(0xE000_E010 as *mut RegisterBlock) }
        }
    }

    pub fn get_time(&self) -> u32 {
        self.p.cvr.read()
    }

    pub fn set_reload(&mut self, reload_value: u32) {
        unsafe { self.p.rvr.write(reload_value) }
    }
}

pub fn example_usage() -> String {
    let mut st = SystemTimer::new();
    st.set_reload(0x00FF_FFFF);
    format!("Time is now 0x{:08x}", st.get_time())
}
```

现在的问题是, 如下这样的代码是被编译器完完全全接受的:

```rust
fn thread1() {
    let mut st = SystemTimer::new();
    st.set_reload(2000);
}

fn thread2() {
    let mut st = SystemTimer::new();
    st.set_reload(1000);
}
```

我们对 `set_reload` 函数的 `&mut self` 参数可以确保没有其他对*这个*特定的 `SystemTimer` 的引用, 但是它们并不阻止用户创建第二个指向同一个外设的变量! 如果作者很努力的发现所有这样"重复"的却动, 那这样的写法会有作用, 但是一旦代码分散到多个模块, 驱动, 开发者之中, 那出现错误会越来越容易.
