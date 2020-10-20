# 引言

欢迎阅读嵌入式Rust之书, 本书是使用Rust在如微控制器(MCU)的"裸金属"嵌入式系统上编程的引导

## 谁应使用Rust进行嵌入式开发

嵌入式Rust为任何想要在嵌入式系统上享受Rust提供的高级功能及安全性的人所提供.
(也可以看看[Who Rust Is For](https://doc.rust-lang.org/book/ch00-00-introduction.html))

## 概览

这本书的目标是:

* 让开发者开苏上手Rust嵌入式开发. 例如, 如何建立开发环境

* 分享*当前*使用Rust进行嵌入式开发的最佳实践. 例如, 如何最好地使用Rust编写更加正确的嵌入式应用

* 在某些情况下提供一个开发指南. 例如, 如何在一个项目中混用C与Rust

本书试着尽可能涵盖各种体系, 但是为了让读者与作者~~还有翻译~~更轻松, 在所有实例中都是用ARM Cortex-M架构.
但是, 本书并不建立在读者熟悉该架构的基础上, 会在需要的地方解释架构的细节.

## 这本书适合谁

本书面向具有一定嵌入式背景或者对Rust熟悉的人, 但是我们相信每个对嵌入式Rust编程感兴趣的人都可以从本书中学到东西.
对于那些没有任何经验知识的人, 建议您阅读 "先决条件" 部分并且补全缺少的知识, 以便从书中获得更多知识并且提升阅读体验.
你可以查看 "其他资源" 部分来查找你想获得的知识对应资源.

### 先决条件

* 你对使用Rust很熟悉, 并且在桌面环境Rust程序写过, 跑过, 捉过虫.
对Rust2018版本熟悉, 应为本书使用Rust 2018

* 熟悉使用其他语言, 如C, C++, Ada开发调试嵌入式系统, 熟悉如以下概念:
  * 交叉编译
  * 内存映射外设
  * 中断
  * 通用接口, 如I2C, SPI, 串口等

### 其他资源

如果你对上面提到的东西不熟, 或者你想对本书提到的一个概念有更加深刻的了解, 你可以看看下面这些资源, 会很有用.

| Topic        | Resource | Description |
|--------------|----------|-------------|
| Rust         | [Rust Book](https://doc.rust-lang.org/book/) | If you are not yet comfortable with Rust, we highly suggest reading this book. |
| Rust, Embedded | [Discovery Book](https://docs.rust-embedded.org/discovery/) | If you have never done any embedded programming, this book might be a better start |
| Rust, Embedded | [Embedded Rust Bookshelf](https://docs.rust-embedded.org) | Here you can find several other resources provided by Rust's Embedded Working Group. |
| Rust, Embedded | [Embedonomicon](https://docs.rust-embedded.org/embedonomicon/) | The nitty gritty details when doing embedded programming in Rust. |
| Rust, Embedded | [embedded FAQ](https://docs.rust-embedded.org/faq.html) | Frequently asked questions about Rust in an embedded context. |
| Interrupts | [Interrupt](https://en.wikipedia.org/wiki/Interrupt) | - |
| Memory-mapped IO/Peripherals | [Memory-mapped I/O](https://en.wikipedia.org/wiki/Memory-mapped_I/O) | - |
| SPI, UART, RS232, USB, I2C, TTL | [Stack Exchange about SPI, UART, and other interfaces](https://electronics.stackexchange.com/questions/37814/usart-uart-rs232-usb-spi-i2c-ttl-etc-what-are-all-of-these-and-how-do-th) | - |

## 怎么看这本书

这本书默认你从头看到尾. 后面的章节建立在前面的基础上, 并且前面的章节不会深挖某个细节部分, 在后面会重新探讨这个问题

这本书使用ST公司的[STM32F3DISCOVERY]开发板作为例子.
这个开发板时ARM Cortex-M架构, 尽管基于该架构的大多数CPU的基本功能都是相似的, 但是不同供应商之间的MCU的外设与其他市县细节是不同的, 并且同意供应商之间的MCU也往往有所不同.

出于这个原因, 我们建议你买一块[STM32F3DISCOVERY]开发板来跟着学习本书中的例子.

[STM32F3DISCOVERY]: http://www.st.com/en/evaluation-tools/stm32f3discovery.html

## 为本书做贡献

本书在[this repository]一起编写并且主要由[resources team]编写

[this repository]: https://github.com/rust-embedded/book
[resources team]: https://github.com/rust-embedded/wg#the-resources-team

如果你跟不住本书或是发现本书中某些部分不够清晰明白或者很难学习, 拿着就是一个BUG并且应该在[the issue tracker]被汇报

[the issue tracker]: https://github.com/rust-embedded/book/issues/

欢迎修改文字错误或是增加内容

## 中文翻译

本书为作者抽空翻译,可能有语义不通顺,如有不明白的地方也请参考[英文原版](https://rust-embedded.github.io/book/#introduction)

如果有勘误, 欢迎提出你的想法

~~同时也复习考研英语~~

本书[仓库](https://github.com/Logiase/The-Embedded-Rust-Book-CN)

时刻欢迎批评与建议

## 重用本书资源

本书在以下LICENSES下发布

* 代码示例与Cargo项目均在[MIT License]与[Apache License v2.0]下发布

* 本书的文字内容, 图片与图标均根据[CC-BY-SA v4.0]条款获得许可

[MIT License]: https://opensource.org/licenses/MIT
[Apache License v2.0]: http://www.apache.org/licenses/LICENSE-2.0
[CC-BY-SA v4.0]: https://creativecommons.org/licenses/by-sa/4.0/legalcode

太长别看系列: 如果你想在你的作品中使用我们的文字或图片, 你应该:

* 加个提醒, 像是提一下本书, 再加个链接
* 提供[CC-BY-SA v4.0]的链接
* 说明你是否对内容进行了修改, 并且用相同的协议对进行更改

另外请一定让我们知道这本书帮了你 :gift:
