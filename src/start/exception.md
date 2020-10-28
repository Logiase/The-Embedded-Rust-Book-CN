# 异常

异常与中断是一种硬件机制, 处理器通过该机制来来异步处理事件或错误(例如执行无效指令).
异常意味着抢占, 涉及异常处理程序, 这些子处理程序使为了响应触发事件的信号而执行的子线程.

`cortex-m-rt`库提供了一个[`exception`]这个属性用来定义异常处理函数.

[`exception`]: https://docs.rs/cortex-m-rt-macros/latest/cortex_m_rt_macros/attr.exception.html

``` rust,ignore
// Exception handler for the SysTick (System Timer) exception
#[exception]
fn SysTick() {
    // ..
}
```

除了`exception`这个属性, 这个函数看着就像一个普通函数, 但有一点不同: `exception`处理函数不能被普通程序调用.
跟着前一个例子, 调用`SysTick`会导致编译错误.

此行为是可以预期的, 并且需要提供一个特性: 定义在`exception`处理中的`static mut`变量必须能够*安全*使用.

``` rust,ignore
#[exception]
fn SysTick() {
    static mut COUNT: u32 = 0;

    // `COUNT` has transformed to type `&mut u32` and it's safe to use
    *COUNT += 1;
}
```

和你知道的一样, 在函数中使用`static mut`变量让其[*不可重入*](https://en.wikipedia.org/wiki/Reentrancy_(computing)).
从多个异常或中断函数中, 或从`main`和一个或多个异常或中断处理函数中直接或间接调用可重入函数是未定义行为.

安全Rust必须不能导致未定义行为, 所以可重入函数必须要标记为`unsafe`.
但是我只是告诉我们的`exception`处理函数可以安全的使用`static mut`变量.
这怎么可能? 但这是可能的, 因为`exception`处理不能被软件调用, 所以也就不可能重入.

