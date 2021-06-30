# 并发

并发发生在你程序的不同的部分在不同时间发生或者无需执行时。
在嵌入式开发中，包括：

* 中断处理，当相关中断发生时
* 各种多线程，当你的微处理器交换线程时
* 在某些系统中，多核处理器中每个核都可以独立运行

由于许多嵌入式应用都需要处理中断，所以并发迟早都会发生，同时也最容易出现许多奇怪难懂的 BUG 。
幸运的是， Rust 提供了许多抽象与安全保证来帮助我们写出正确的代码。

## 无并发

最简单的并发就是没有并发：你的应用就一个循环一直在运行，也没有中断。
有时候这就足够解决手头上的问题了！
典型的情况时是你的循环读取一些输入然后做一些处理进行输出。

```rust, ignore
#[entry]
fn main() {
    let peripherals = setup_peripherals();
    loop {
        let inputs = read_inputs(&peripherals);
        let outputs = process(inputs);
        write_outputs(&peripherals, outputs);
    }
}
```

因为没有并发，所以你也没必要担心在程序的不同部分分享数据或是同步外设的访问权限。
如果你能用这种方法解决问题那很好。

## 全局可变数据

不像非嵌入式的 Rust ，我们通常不会创建堆然后把对数据的引用传递给新建的线程。
相反我们的中断处理函数可能在任意时刻被调用，并且必须知道如何访问我们正在使用的内存。
在底层上这意味着我们必须静态分配可变内存，让这块内存可以被中断和主程序引用。

在 Rust 中，像 [`static mut`] 这样的变量是读写不安全的，因为没有特殊照顾的情况下，这可能会出现竞态，
即你对数据的访问在半路上被同样要访问该数据的中断打断。

[`static mut`]: https://doc.rust-lang.org/book/ch19-01-unsafe-rust.html#accessing-or-modifying-a-mutable-static-variable

举个例子，设想一下有个应用，它用一个计数器测量在一秒内一个信号有多少上升沿（频率计）：

```rust,ignore
static mut COUNTER: u32 = 0;

#[entry]
fn main() -> ! {
    set_timer_1hz();
    let mut last_state = false;
    loop {
        let state = read_signal_level();
        if state && !last_state {
            // DANGER - Not actually safe! Could cause data races.
            unsafe { COUNTER += 1 };
        }
        last_state = state;
    }
}

#[interrupt]
fn timer() {
    unsafe { COUNTER = 0; }
}
```

定时器中断每秒把计数器归零。
同时主循环还在不停的测量信号，有一个上升沿就 +1 。
我们使用 `unsafe` 来操作 `COUNTER` ，因为它是个 `static mut` ，这意味着我们向编译器保证我们不会做任何未定义行为。
你能看出来这有竞态吗？
`COUNTER` 的增加 _不是_ 原子的 -- 事实上，在绝大多数嵌入式平台上，这个操作会被分成读取、增加、保存三个步骤。
如果中断发生在读取之后，保存之前，那清零的操作就会被忽略，我们便会一个周期计两次。

## 临界区

那么，我们要怎么做？一个简单的方法是使用 _临界区_ ，在这中断被关闭。
通过在 `main` 中使用一个临界区来包裹 `COUNTER` 我们可以保证在我们完成增加 `COUNTER` 前定时器中断不会触发。

```rust,ignore
static mut COUNTER: u32 = 0;

#[entry]
fn main() -> ! {
    set_timer_1hz();
    let mut last_state = false;
    loop {
        let state = read_signal_level();
        if state && !last_state {
            // New critical section ensures synchronised access to COUNTER
            cortex_m::interrupt::free(|_| {
                unsafe { COUNTER += 1 };
            });
        }
        last_state = state;
    }
}

#[interrupt]
fn timer() {
    unsafe { COUNTER = 0; }
}
```

在这个例子中，我们使用 `cortex_m::interrupt::free` ，其他平台也有类似的步骤。
这等效于禁用中断，执行代码，重启中断。

注意我们不需要在中断中使用临界区，因为：

* 对 `COUNTER` 写0不会导致竞态，因为我们没读它
* 它不可能被 `main` 中断

如果 `COUNTER` 被多个中断处理函数 _共用_ ，那么每个中断可能都需要一个临界区。

这解决了我们眼前的问题，但是我们还是得写一堆需要仔细考虑的不安全代码，而且也有可能写了没必要的临界区。
因为每个临界区都暂时的停止了中断，所以会有一些额外的代码大小，还增加了中断的延迟与中断处理的时间。
这是不是个问题取决于你的系统，但我们应该避免。

需要注意，虽然临界区保证不会发生中断，但是它并不能在多核系统上做出同样的保证！
即使没有中断，其他的核心也可以访问你操作的核的内存。如果你使用多核系统，那么你需要更强的同步原语。

## 原子操作

在一些平台上，我们可以使用特殊的原子指令，为读取-修改-保存操作提供保证。
针对 Cortex-M: `thumbv6`(Cortex-M0, Cortex-M0+) 只提供原子读和原子写， `thumbv7`(Cortex-M3 及以上 ) 提供完整的比较交换（CAS）操作。
这些 CAS 指令提供了消耗严重的禁用中断的替代方法：我们直接增加，大多数时候会成功，但如果被中断，它会自动尝试重新增加。
即使是多核系统，这些操作仍然是安全的。

```rust,ignore
use core::sync::atomic::{AtomicUsize, Ordering};

static COUNTER: AtomicUsize = AtomicUsize::new(0);

#[entry]
fn main() -> ! {
    set_timer_1hz();
    let mut last_state = false;
    loop {
        let state = read_signal_level();
        if state && !last_state {
            // Use `fetch_add` to atomically add 1 to COUNTER
            COUNTER.fetch_add(1, Ordering::Relaxed);
        }
        last_state = state;
    }
}

#[interrupt]
fn timer() {
    // Use `store` to write 0 directly to COUNTER
    COUNTER.store(0, Ordering::Relaxed)
}
```

这次 `COUNTER` 是一个安全的 `static` 变量。多亏 `AtomicUsize` 类型， `COUNTER` 能从中断和主循环中不停用中断安全修改。
如果可行的话这是个更好的方法 -- 但它取决于你的系统支不支持。

关于 [`Ordering`] 的说明：
这影响编译器和硬件对指令的重新排序方式，也会对缓存产生影响。
如果单核的话， `Relaxed` 就够了，也是效率最高的方法。
更严格的排序会让编译器围绕原子操作生成内存屏障；
根据你使用的原子操作的对象选择是否使用。原子模型很复杂，在这里不做介绍。

如果想了解更多有关原子与排序的内容，请看 [nomicon] 。

[`Ordering`]: https://doc.rust-lang.org/core/sync/atomic/enum.Ordering.html
[nomicon]: https://doc.rust-lang.org/nomicon/atomics.html

## 抽象、发送和同步

上面的方法都不是很让人满意。他们需要使用 `unsafe` ，所以我们得非常仔细的检查，很反人类。
在 Rust 中我们有更好的解决办法！

我们可以把 `COUNTER` 抽象成一个我们可以在哪都能用的安全的接口。
在这个例子中，我们使用临界区，但你也可以用原子操作做到相同的功能。

```rust,ignore
use core::cell::UnsafeCell;
use cortex_m::interrupt;

// Our counter is just a wrapper around UnsafeCell<u32>, which is the heart
// of interior mutability in Rust. By using interior mutability, we can have
// COUNTER be `static` instead of `static mut`, but still able to mutate
// its counter value.
struct CSCounter(UnsafeCell<u32>);

const CS_COUNTER_INIT: CSCounter = CSCounter(UnsafeCell::new(0));

impl CSCounter {
    pub fn reset(&self, _cs: &interrupt::CriticalSection) {
        // By requiring a CriticalSection be passed in, we know we must
        // be operating inside a CriticalSection, and so can confidently
        // use this unsafe block (required to call UnsafeCell::get).
        unsafe { *self.0.get() = 0 };
    }

    pub fn increment(&self, _cs: &interrupt::CriticalSection) {
        unsafe { *self.0.get() += 1 };
    }
}

// Required to allow static CSCounter. See explanation below.
unsafe impl Sync for CSCounter {}

// COUNTER is no longer `mut` as it uses interior mutability;
// therefore it also no longer requires unsafe blocks to access.
static COUNTER: CSCounter = CS_COUNTER_INIT;

#[entry]
fn main() -> ! {
    set_timer_1hz();
    let mut last_state = false;
    loop {
        let state = read_signal_level();
        if state && !last_state {
            // No unsafe here!
            interrupt::free(|cs| COUNTER.increment(cs));
        }
        last_state = state;
    }
}

#[interrupt]
fn timer() {
    // We do need to enter a critical section here just to obtain a valid
    // cs token, even though we know no other interrupt could pre-empt
    // this one.
    interrupt::free(|cs| COUNTER.reset(cs));

    // We could use unsafe code to generate a fake CriticalSection if we
    // really wanted to, avoiding the overhead:
    // let cs = unsafe { interrupt::CriticalSection::new() };
}
```

我们把 `unsafe` 的代码移到了我们精心设计好的抽象中，现在我们的应用不包含任何 `unsafe` 的部分。

这个设计要求我们传入一个 `CriticalSection` 标记：这些标记只能由 `interrupt::free` 安全生成，
所以通过要求传入一个标志，我们保证这个操作实在临界区执行的，而不用自己去操作。
这个保证由编译器提供：不会在运行时有任何关于 `cs` 的开销。
如果我们有多个计数器，它们也可以使用相同的 `cs` ，不用嵌套多个临界区。

这也引出了 Rust 中一个重要的话题： [`Send` and `Sync`] traits 。
总结一下， 实现 Send 的类型可以被安全的转移到另一个线程，
而实现 Sync 的可以安全的在多个线程中共用。
在嵌入式开发中，我们把中断视为新开线程，所以主代码块与中断共用的变量一定实现 Sync 。

[`Send` and `Sync`]: https://doc.rust-lang.org/nomicon/send-and-sync.html

对于 Rust 中的大多数类型，这两个 traits 通常由编译器自动派生。
然而，因为 `CSCounter` 包含一个 [`UnsafeCell`] ，它并不 Sync ，
所以我们没法声明一个 `static CSCounter` ： `static` _一定_ 是修饰 Sync 的，因为能被多线程共用。

[`UnsafeCell`]: https://doc.rust-lang.org/core/cell/struct.UnsafeCell.html

为了让编译器知道 `CSCounter` 事实上多线程共用是安全的，我们主动为它加上 Sync 。
与之前用的临界区一样，它只在单核系统上安全。

## 互斥量

我们针对计数器问题创造了一种抽象，同时还有很多用于并发的通用的抽象。

一种 _同步原语_ 叫互斥（mutex）， mutual exclusion 的缩写。
这种结构确保对变量的独占访问，如我们的计数器。
一个线程可以尝试去 _锁_ （或 _需求_ ）这个互斥锁，然后要么马上成功，要么等锁被用完，要么返回一个没法上锁的错误。
当该线程持有这个锁时，它能够访问这个受保护的数据。
当线程结束时，它 _解锁_ （或 _释放_ ）这个互斥锁，以便让其他线程上锁。
在 Rust 里，我们通常使用 [`Drop`] trait 来修饰 Unlock ，以确保互斥量超出作用域时能正确释放锁。

[`Drop`]: https://doc.rust-lang.org/core/ops/trait.Drop.html

把中断和互斥量用在一起可能有点难：中断中通常来说都不能阻塞，并且在中断中阻塞等待主循环解锁是不可能的，
会发生 _死锁_ （主线程因为等待中断结束而不会解锁）。
死锁是不安全的，即使在没有 `unsafe` 的 Rust 中也有可能发生。

为了避免这种情况的发生，我们可以实现一个需要临界区来上锁的互斥量，就像例子一样。
只要临界区和锁生命周期一样我们就可以保证我们独占被包装的变量，甚至不需要管互斥量锁没锁。

`cortex-m` 库已经帮我们完成了这些！我们可以用这种方法来写我们的计数器：

```rust,ignore
use core::cell::Cell;
use cortex_m::interrupt::Mutex;

static COUNTER: Mutex<Cell<u32>> = Mutex::new(Cell::new(0));

#[entry]
fn main() -> ! {
    set_timer_1hz();
    let mut last_state = false;
    loop {
        let state = read_signal_level();
        if state && !last_state {
            interrupt::free(|cs|
                COUNTER.borrow(cs).set(COUNTER.borrow(cs).get() + 1));
        }
        last_state = state;
    }
}

#[interrupt]
fn timer() {
    // We still need to enter a critical section here to satisfy the Mutex.
    interrupt::free(|cs| COUNTER.borrow(cs).set(0));
}
```

我们现在使用 `Cell` ，它与他的兄弟 `RefCell` 共同提供安全的内部可变性。
我们已经见过 `UnsafeCell` 了，他是 Rust 内部可变性的最底层：它允许你获取多个它的可变引用，但只能在 unsafe 中。
一个 `Cell` 和 `UnsafeCell` 差不多，但是它提供一个安全的接口：
它只允许获取当前值的一个复制或者替换它，而不允许引用，并且因为它不 Sync ，它没法在线程中共用。
这些特性意味着我们能安全使用，但我们没法直接用 `static` 修饰它，因为 `static` 只能用在 Sync 身上。

[`Cell`]: https://doc.rust-lang.org/core/cell/struct.Cell.html

那为什么上面的例子能用？ `Mutex<T>` 为任何实现 Send 的 `T` 实现 Sync。
这么做是安全的因为它只在临界区中允许访问它的内容。
因此我们能得到一个完全不用 unsafe 的安全计数器。

对于像 `u32` 这样的简单结构很棒，但是不能 Copy 的复杂类型呢？
嵌入式开发中一个很常见的示例是外设，它通常是不能 Copy 的。
因此我们可以使用 `RefCell` 。

## 分享外设

使用 `svd2rust` 和相关抽象生成设备库通过强制外设只有一个实例保证了安全。
但是也给从主线程与中断中操作外设造成了困难。

为了安全的分享外设权限，我们可以像之前一样使用 `Mutex` 。
我们还要用到 [`RefCell`] ，它有一个运行时检查来确保一次只给出一个外设的引用。
相比普通的 `Cell` 有着更多的开销，但因为我们提供引用而不是副本，我们必须确保同时只能存在一个。

[`RefCell`]: https://doc.rust-lang.org/core/cell/struct.RefCell.html

最后，我们还需要考虑怎么在主线程初始化后把外设移动到共享变量中。
为此我们可以使用 `Option` 类型，初始化为 `None` 然后再设置为外设的实例。

```rust,ignore
use core::cell::RefCell;
use cortex_m::interrupt::{self, Mutex};
use stm32f4::stm32f405;

static MY_GPIO: Mutex<RefCell<Option<stm32f405::GPIOA>>> =
    Mutex::new(RefCell::new(None));

#[entry]
fn main() -> ! {
    // Obtain the peripheral singletons and configure it.
    // This example is from an svd2rust-generated crate, but
    // most embedded device crates will be similar.
    let dp = stm32f405::Peripherals::take().unwrap();
    let gpioa = &dp.GPIOA;

    // Some sort of configuration function.
    // Assume it sets PA0 to an input and PA1 to an output.
    configure_gpio(gpioa);

    // Store the GPIOA in the mutex, moving it.
    interrupt::free(|cs| MY_GPIO.borrow(cs).replace(Some(dp.GPIOA)));
    // We can no longer use `gpioa` or `dp.GPIOA`, and instead have to
    // access it via the mutex.

    // Be careful to enable the interrupt only after setting MY_GPIO:
    // otherwise the interrupt might fire while it still contains None,
    // and as-written (with `unwrap()`), it would panic.
    set_timer_1hz();
    let mut last_state = false;
    loop {
        // We'll now read state as a digital input, via the mutex
        let state = interrupt::free(|cs| {
            let gpioa = MY_GPIO.borrow(cs).borrow();
            gpioa.as_ref().unwrap().idr.read().idr0().bit_is_set()
        });

        if state && !last_state {
            // Set PA1 high if we've seen a rising edge on PA0.
            interrupt::free(|cs| {
                let gpioa = MY_GPIO.borrow(cs).borrow();
                gpioa.as_ref().unwrap().odr.modify(|_, w| w.odr1().set_bit());
            });
        }
        last_state = state;
    }
}

#[interrupt]
fn timer() {
    // This time in the interrupt we'll just clear PA0.
    interrupt::free(|cs| {
        // We can use `unwrap()` because we know the interrupt wasn't enabled
        // until after MY_GPIO was set; otherwise we should handle the potential
        // for a None value.
        let gpioa = MY_GPIO.borrow(cs).borrow();
        gpioa.as_ref().unwrap().odr.modify(|_, w| w.odr1().clear_bit());
    });
}
```

需要考虑的内容很多，让我们来挑出重要的几行。

```rust,ignore
static MY_GPIO: Mutex<RefCell<Option<stm32f405::GPIOA>>> =
    Mutex::new(RefCell::new(None));
```

我们的共享变量现在是一个 `Mutex` 套娃 `RefCell` 套娃 `Option` 。
 `Mutex` 确保我们仅能够在临界区有访问权限，来让本来不 Sync 的 `RefCell` Sync 。
 `RefCell` 通过引用为我们提供了内部可变性，让我们能够用我们的 `GPIOA` 。
 `Option` 让我们能够初始化一个空值然后再把我们的变量塞进去。
我们没法静态访问外设实例，只有在运行时可以，所以这是必须的。

```rust,ignore
interrupt::free(|cs| MY_GPIO.borrow(cs).replace(Some(dp.GPIOA)));
```

在临界区中，我们对互斥量使用 `borrow()` ，让我们拿到一个 `RefCell` 的引用。
使用 `replace()` 来替换 `RefCell` 中的值。

```rust,ignore
interrupt::free(|cs| {
    let gpioa = MY_GPIO.borrow(cs).borrow();
    gpioa.as_ref().unwrap().odr.modify(|_, w| w.odr1().set_bit());
});
```

最后我们能够安全并发使用 `MY_GPIO` 。临界区防止中断发生，让我们解锁互斥量。
 `RefCell` 给我们一个 `&Option<GPIOA>` ，
并且跟踪它的生命周期 -- 一旦生命周期结束， `RefCell` 将被更新以表示它不再被使用。

因为我们没法把 `GPIOA` 移出 `&Option` ，我们需要使用 `as_ref()` 转换成 `&Option<&GPIOA>` ，
让我们最终能 `unwarp()` 出能操作外设的 `&GPIOA` 。

如果我们需要一个共享资源的可变引用，那使用 `borrow_mut` 和 `deref_mut` 来替代。
下面的例子使用 TIM2 来展示。

```rust,ignore
use core::cell::RefCell;
use core::ops::DerefMut;
use cortex_m::interrupt::{self, Mutex};
use cortex_m::asm::wfi;
use stm32f4::stm32f405;

static G_TIM: Mutex<RefCell<Option<Timer<stm32::TIM2>>>> =
	Mutex::new(RefCell::new(None));

#[entry]
fn main() -> ! {
    let mut cp = cm::Peripherals::take().unwrap();
    let dp = stm32f405::Peripherals::take().unwrap();

    // Some sort of timer configuration function.
    // Assume it configures the TIM2 timer, its NVIC interrupt,
    // and finally starts the timer.
    let tim = configure_timer_interrupt(&mut cp, dp);

    interrupt::free(|cs| {
        G_TIM.borrow(cs).replace(Some(tim));
    });

    loop {
        wfi();
    }
}

#[interrupt]
fn timer() {
    interrupt::free(|cs| {
        if let Some(ref mut tim)) =  G_TIM.borrow(cs).borrow_mut().deref_mut() {
            tim.start(1.hz());
        }
    });
}

```

哇！这很安全，但也有点憨批。我们还有什么可以做的吗？

## RTIC

一种替代是 [RTIC framework] ( Real Time Interrupt-driven Concurrency )。
它强制执行静态优先级并跟踪对 `static mut` 变量（“资源”）的访问，以确保共享资源始终安全访问，
而不用进入临界区和使用引用计数（如在 `RefCell` 中）的开销。 
它有许多优点，例如保证没有死锁并提供极快的时间和内存开销。

[RTIC framework]: https://github.com/rtic-rs/cortex-m-rtic

该框架还提供了许多其他功能，如消息传递，能减少对显式共享状态的需求，还有能在指定时间调度任务的能力，可以用来实现周期性任务。
查看 [the documentation] 获取更多信息！

[the documentation]: https://rtic.rs

## 实时操作系统

嵌入式并发的另一种常见方法是实时操作系统（ RTOS ）。
虽然在 Rust 中发展还不是很好，但他们广泛应用于传统嵌入式开发。
开源项目包括 [FreeRTOS] 和 [ChibiOS] 。
这些实时操作系统为运行多个线程提供 CPU 调度的支持，包括线程让出控制（协作多任务）与基于常规计时器与中断（抢占式任务）。
 RTOS 通常提供互斥量与其他同步原语，并且经常与硬件引擎（如 DMA 控制器）进行互操作。

[FreeRTOS]: https://freertos.org/
[ChibiOS]: http://chibios.org/

在本文撰写时，还没有许多 Rust 的 RTOS 例子，但请仍然关注。

## 多核

在嵌入式系统中，多核系统越来越普遍，这给并发又增加了难度与复杂程度。
所有使用临界区的例子（包括 `cortex_m::interrupt::Mutex` ）都假设唯一能打断的线程是中断，
但在多核系统上不是这样。
相反我们需要为多核系统设计的同步原语（也叫 SMP ，symmetric multi-processing ）。

这些通常使用我们之前看到的原子指令，因为处理系统将确保在所有内核上保持原子性。

详细介绍这些主题目前超出了本书的范围，但一般模式与单核情况相同。