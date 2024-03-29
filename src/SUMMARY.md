# Summary

- [引言](./intro/index.md)
  - [硬件](./intro/hardware.md)
  - [`no_std`](./intro/no-std.md)
  - [工具](./intro/tooling.md)
  - [安装](./intro/install.md)
    - [Linux](./intro/install/linux.md)
    - [MacOS](./intro/install/macos.md)
    - [Windows](./intro/install/windows.md)
    - [验证安装](./intro/install/verify.md)
- [开始](./start/index.md)
  - [QEMU](./start/qemu.md)
  - [硬件](./start/harware.md)
  - [内存映射寄存器](./start/registers.md)
  - [Semihosting](./start/semihosting)
  - [Panicking](./start/panicking.md)
  - [异常](./start/exception.md)
  - [中断](./start/interrupts.md)
  - [IO](./start/io.md)
- [外设](./peripherals/index.md)
  - [Rust的第一次尝试](./peripherals/a-first-attempt.md)
  - [引用检查](./peripherals/borrowck.md)
  - [单例](./peripherals/singletons.md)
- [静态检查](./static-guarantees/index.md)
  - [状态机编程](./static-guarantees/typestate-programming.md)
  - [外设状态机](./static-guarantees/state-machines.md)
  - [设计合同](./static-guarantees/design-contracts.md)
  - [零成本抽象](./static-guarantees/zero-cost-abstractions.md)
- [可移植性](./portability/index.md)
- [并发](./concurrency/index.md)
- [容器](./collections/index.md)
- [设计模式](./design-patterns/index.md)
  - [硬件抽象层](./design-patterns/hal/index.md)
    - [Checklist](./design-patterns/hal/checklist.md)
    - [Naming](./design-patterns/hal/naming.md)
    - [Interoperability](./design-patterns/hal/interoperability.md)
    - [Predictability](./design-patterns/hal/predictability.md)
    - [GPIO](./design-patterns/hal/gpio.md)
- [Tips for embedded C developers](./c-tips/index.md)
    <!-- TODO: Define Sections -->
- [Interoperability](./interoperability/index.md)
  - [A little C with your Rust](./interoperability/c-with-rust.md)
  - [A little Rust with your C](./interoperability/rust-with-c.md)
- [Unsorted topics](./unsorted/index.md)
  - [Optimizations: The speed size tradeoff](./unsorted/speed-vs-size.md)
  - [Performing Math Functionality](./unsorted/math.md)

---

[Appendix A: Glossary](./appendix/glossary.md)