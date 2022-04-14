# 容器

我们经常会在我们的应用中使用动态的数据结构（或者叫集合、容器）。
`std` 为我们提供了一系列常用的容器：[`Vec`], [`String`], [`HashMap`]等等。
这些在`std`中实现的容器都用了全局的内存分配器。

[`Vec`]: https://doc.rust-lang.org/std/vec/struct.Vec.html
[`String`]: https://doc.rust-lang.org/std/string/struct.String.html
[`HashMap`]: https://doc.rust-lang.org/std/collections/struct.HashMap.html

但是在`Core`的定义中没有内存分配器的实现，它们随编译器`alloc`库中附带。

如果你需要使用容器，除了自己实现一个内存分配器外，也可以考虑一下*固定容量*的容器，比如[`heapless`]

[`heapless`]: https://crates.io/crates/heapless

在本章节中，我们会介绍并对比这两种实现方式。

## 使用`alloc`

`alloc`默认随Rust附带，不用在`Cargo.toml`中声明就可以使用。

```rust,ignore
#![feature(alloc)]

extern crate alloc;

use alloc::vec::Vec;
```

为了使用这些容器你首先需要使用`global_allocator`来给你的程序标记使用的全局内存分配器，分配器需要实现 [`GlobalAlloc`] trait。

[`GlobalAlloc`]: https://doc.rust-lang.org/core/alloc/trait.GlobalAlloc.html

为了使本章节尽量保持完整清晰，让我们实现一个简单的bump pointer allocator作为全局内存分配器。
但是*强烈*建议你在你的程序中使用一个经过实际测试的库来替代这个简陋的分配器。

``` rust,ignore
// Bump pointer allocator implementation
extern crate cortex_m;
use core::alloc::GlobalAlloc;
use core::ptr;
use cortex_m::interrupt;
// Bump pointer allocator for *single* core systems
struct BumpPointerAlloc {
    head: UnsafeCell<usize>,
    end: usize,
}
unsafe impl Sync for BumpPointerAlloc {}
unsafe impl GlobalAlloc for BumpPointerAlloc {
    unsafe fn alloc(&self, layout: Layout) -> *mut u8 {
        // `interrupt::free` is a critical section that makes our allocator safe
        // to use from within interrupts
        interrupt::free(|_| {
            let head = self.head.get();
            let size = layout.size();
            let align = layout.align();
            let align_mask = !(align - 1);
            // move start up to the next alignment boundary
            let start = (*head + align - 1) & align_mask;
            if start + size > self.end {
                // a null pointer signal an Out Of Memory condition
                ptr::null_mut()
            } else {
                *head = start + size;
                start as *mut u8
            }
        })
    }
    unsafe fn dealloc(&self, _: *mut u8, _: Layout) {
        // this allocator never deallocates memory
    }
}
// Declaration of the global memory allocator
// NOTE the user must ensure that the memory region `[0x2000_0100, 0x2000_0200]`
// is not used by other parts of the program
#[global_allocator]
static HEAP: BumpPointerAlloc = BumpPointerAlloc {
    head: UnsafeCell::new(0x2000_0100),
    end: 0x2000_0200,
};
```

除了选择全局内存分配器，我们还需要使用*不稳定标签*`alloc_error_handler`定义内存溢出错误的处理办法。

``` rust,ignore
#![feature(alloc_error_handler)]
use cortex_m::asm;
#[alloc_error_handler]
fn on_oom(_layout: Layout) -> ! {
    asm::bkpt();
    loop {}
}
```

这些都完成之后，我们就可以使用在`alloc`中定义的容器了。

```rust,ignore
#[entry]
fn main() -> ! {
    let mut xs = Vec::new();
    xs.push(42);
    assert!(xs.pop(), Some(42));
    loop {
        // ..
    }
}
```

这些容器和`std`中的定义完全一样。

## 使用`heapless`

`heapless`不需要初始化因为它并不依赖全局内存分配器，直接使用就可以：

```rust,ignore
extern crate heapless; // v0.4.x
use heapless::Vec;
use heapless::consts::*;
#[entry]
fn main() -> ! {
    let mut xs: Vec<_, U8> = Vec::new();
    xs.push(42).unwrap();
    assert_eq!(xs.pop(), Some(42));
}
```

你需要注意这两种容器的区别（`heapless`和`alloc`）。

首先你需要定义容器的最大容量，`heapless`不会重新分配内存并且有固定大小的容量，并且容量是类型的一部分。
上面的例子中，我们定义了一个只能容纳8个元素的向量`xs`（容量为8）。这个容量由`U8`（参见[`typenum`]）确定。

[`typenum`]: https://crates.io/crates/typenum

并且，`push`还有其他许多方法会返回一个`Result`。因为`heapless`容器有固定的容量，因此所有插入元素的操作都有失败的可能性。
`heapless` API通过返回一个`Result`来表示是否成功。使用`alloc`的情况下，容器会在容量满时重新分配内存。

`heapless` v0.4.x版本的容器会直接存储他们的元素，这意味着如`let x = heapless::Vec::new();`这样的操作会直接把容器分配到栈上（全在栈上），
不过想要把他们分配到`static`或者堆上（`Box<Vec<_, _>>`）也是可行的。

## 使用权衡

在选择使用堆分配自动扩容的容器或固定容量的容器时记住一下几条内容。

### 内存溢出错误处理（Out of Memory）

在容器自动扩容的时候有可能发生OOM错误：比如`alloc::Vec.push`。一些扩容操作可能悄悄失败，
一些`alloc`的容器会提供一个`try_reserve`的方法来让你检查扩容是否会失败，但你必须主动调用。

如果你只使用`heapless`容器，并且不用内存分配器那就永远不会出现OOM错误。
但是你需要处理容器容量不够的问题（指你需要手动处理所有类似`Vec.push`返回的`Result`）

OOM错误处理比`heapless`的Result难得多（`Result`只要`unwrap`就行），因为出现错误的位置可能和引发错误的位置不同。
比如`vec.reserve(1)`失败了，但真正的原因是其他容器一直在内存泄漏导致没法继续分配（safe Rust中也是可以出现内存泄漏的）。

### 内存使用

堆内存的使用情况进行分析时很难的，因为长期存活的容器容量是可以在运行时变化的。
一些操作可以增大内存占用，也有操作可以减少内存占用（比如`shrink_to_fit`）。
另外，分配器也要处理内存分片的问题来增加*可用*内存。

另一方面，如果你只用固定容量的容器，把它们存在`static`变量中然后设置一个容量，连接器会自动为你检查是否会容量不够。

另外，分配在栈上的固定大小容器可以在栈分析工具（如 [`stack-sizes`]）使用 [`-Z emit-stack-sizes`] 来分析报告。

[`-Z emit-stack-sizes`]: https://doc.rust-lang.org/beta/unstable-book/compiler-flags/emit-stack-sizes.html
[`stack-sizes`]: https://crates.io/crates/stack-sizes

然而，固定容量的容器不能被压缩大小，相比自动扩容的容器有更低的内存使用效率（实际使用内存与占用内存的比值）。

### 最慢执行时间 (WCET)

如果你在做一个时间敏感型应用，或者对时间有很高要求，那么你需要关心一下你应用不同部分的最慢执行时间。

`alloc`内的容器会重新分配内存，所以最慢执行时间会包括容器在*运行时*重新分配内存的过程。
这让最慢执行时间难以估计，例如`alloc::Vec.push`会根据已有容量和运行时容量进行扩容。

另一方面，固定容量容器从来不会重新分配内存，所以时间是可估计的。例如`heapless::Vec.push`有一个固定的时间。

### 便于使用

`alloc`需要设置一个全局内存分配器，而`heapless`就不需要。然而`heapless`需要你在初始化时指定容量。

`alloc`的API对几乎任何Rust开发者都很熟悉，`heapless`的API已经尽量和`alloc`保持一致了，但是因为它的错误处理还是会有不同，
一些开发者会觉得显式的错误处理太麻烦太多了。
