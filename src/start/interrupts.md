# 中断

中断在许多方面与异常有区别, 但是他们的操作与使用又在很大程度上相同, 并且他们也由同一中断控制器控制.
尽管异常是由Cortex-M架构定义的, 但是在中断的命名和功能上确实供应商(或芯片)确定的.

中断有很大的灵活性, 在尝试使用高级方法使用他们前, 应该考虑这些灵活性.
在本书中我们不会介绍用法, 但请牢记下面几点:

 - 中断具有可编程的优先级, 可以用来确定他们处理程序的顺序.
 - 中断可以嵌套, 可以抢占, 就是中断处理过程可能被另一个更高优先级的中断打断.
 - 通常需要处理掉触发中断的原因, 以防止无限进入中断.

运行时的常规初始化步骤始终相同:

 - 设置外设来启用中断.
 - 在中断处理器中设置优先级.
 - 在终端控制器中启用中断函数.

和异常一样, `cortex-m-rt`提供了一个[`interrupt`]属性来定义中断处理函数.
可用的中断(还有中断向量表中的位置)通常使用`svd2rust`由SVD文件自动生成.

[`interrupt`]: https://docs.rs/cortex-m-rt-macros/0.1.5/cortex_m_rt_macros/attr.interrupt.html

``` rust,ignore
// Interrupt handler for the Timer2 interrupt
#[interrupt]
fn TIM2() {
    // ..
    // Clear reason for the generated interrupt request
}
```

中断处理函数看起来就像普通函数一样(除了缺少参数).
但是由于特殊的调用约定, 他们不能直接被固件的其他部分直接调用.
但是可以在软件中生成中断请求, 以触发对中断函数的转移.

与异常处理类似, 也可以在中断处理函数中定义`static mut`变量来保持状态*安全*.

``` rust,ignore
#[interrupt]
fn TIM2() {
    static mut COUNT: u32 = 0;

    // `COUNT` has type `&mut u32` and it's safe to use
    *COUNT += 1;
}
```

有关此机制的更加详细的说明, 请参考[异常].

[异常]: ./exceptions.md