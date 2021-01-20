# 状态机编程

[typestate] 的概念描述了将当前的状态编码为一个类型. 尽管这听起来不可思议, 但如果你在 Rust 中使用了 [Builder Pattern] (建造者模式), 你就已经在用状态机编程了!

[typestates]: https://en.wikipedia.org/wiki/Typestate_analysis
[Builder Pattern]: https://doc.rust-lang.org/1.0.0/style/ownership/builders.html

```rust
pub mod foo_module {
    #[derive(Debug)]
    pub struct Foo {
        inner: u32,
    }

    pub struct FooBuilder {
        a: u32,
        b: u32,
    }

    impl FooBuilder {
        pub fn new(starter: u32) -> Self {
            Self {
                a: starter,
                b: starter,
            }
        }

        pub fn double_a(self) -> Self {
            Self {
                a: self.a * 2,
                b: self.b,
            }
        }

        pub fn into_foo(self) -> Foo {
            Foo {
                inner: self.a + self.b,
            }
        }
    }
}

fn main() {
    let x = foo_module::FooBuilder::new(10)
        .double_a()
        .into_foo();

    println!("{:#?}", x);
}
```

在这个例子中, 没有一个直接创建 `Foo` 对象的方法. 我们必须先创建一个 `FooBuilder`, 然后正确的初始化它才能得到我们想要的 `Foo` 对象.

这个简单的小例子编码了两种状态:

- `FooBuilder` 代表了一个"未配置", "在配置中"的状态
- `Foo` 代表了一个"配置完成", "准备使用"的状态

## 强类型

因为 Rust 有一个 [Strong Type System] (强类型系统), 所以没有什么花里胡哨的办法去直接创建一个 `Foo` 的实例, 或者不用 `into_foo()` 方法把一个 `FooBuilder` 转变为 `Foo`. 另外, 调用 `into_foo()` 方法会消费掉原来的 `FooBuilder`, 意思是你没法再用它去创建一个新实例.

[Strong Type System]: https://en.wikipedia.org/wiki/Strong_and_weak_typing

强类型系统让我们能把我们的系统状态表示为类型, 并且可以用方法来把由一种状态到另一种状态所需的步骤描述出来. 通过创建一个 `FooBuilder`, 并把它转变为 `Foo` 对象, 我们完成了一个基本的状态机.
