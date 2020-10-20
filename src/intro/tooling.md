# 工具

处理微控制器涉及到使用集中不同的工具, 因为我们要处理一个与你电脑架构不同的架构, 我们必须要在远程设备上来运行和调试程序. 

我们会使用下面列出的工具. 没指定最低版本时, 按理说任何最新版本都能用, 但是我们也列出了经过测试的版本. 

- Rust 1.31, 1.31-beta, 或带有 ARM Cortex-M 编译器的更新的工具链
- [`cargo-binutils`](https://github.com/rust-embedded/cargo-binutils) ~0.1.4
- [`qemu-system-arm`](https://www.qemu.org/). 测试版本: 3.0.0
- OpenOCD >=0.8. 测试版本: v0.9.0 and v0.10.0
- GDB with ARM support. 7.12或更高版本. 测试版本: 7.10, 7.11, 7.12 and 8.1
- [`cargo-generate`](https://github.com/ashleygwilliams/cargo-generate) 或 `git`.
  这个工具可选但是会让你学习本书更加轻松. 

下面来讲为什么我们需要这些工具. 安装说明会在下一页提及. 

## `cargo-generate` 或 `git`

裸金属应用是不标准(`no_std`)的Rust程序, 需要对链接过程做出一些调整, 以使程序的内存布局正确. 这需要一些额外的文件(像是链接器脚本)和设置(像是连接器参数). 
我们已经把这些打包成了一个模板, 这样你就只需要填写确实的信息就行(就想项目名称和目标硬件型号). 

我们的模板与`cargo-generate`兼容, `cargo-generate`是Cargo的一个子命令, 用来从模板创建新的Cargo项目. 你也可以使用`git`, `curl`, `wget`或浏览器来下载模板. 

## `cargo-binutils`

`cargo-binutils`是Cargo的一系列子命令, 用来更轻松的配合Rust工具链使用LLVM工具. 这些工具包含LLVM版本的`objdump`, `nm`和`size`, 用来检查二进制产物.

与GNU binmutils相比, 使用这些工具的优势在于, (a) 可以无视系统一键安装LLVM工具(`rustup component add llvm-tools-preview`), (b) 像`objdump`这样的工具支持所有`rustc`支持的所有架构, 从 ARM 到 x86_64 应为他们都使用了相同的LLVM后端.

## `qemu-system-arm`

QEMU是个模拟器. 在本书中, 我们使用能够模拟各种ARM系统的变体. 
我们使用QEMU来在电脑上运行嵌入式程序. 
多亏这个, 你能在没有硬件的情况下学习本书. 

## GDB

调试器是嵌入式开发中非常重要的一个组件, 因为你并不总是有足够的空间去把气质打到控制台上. 
某些情况下, 你的硬件上甚至都没有LED可以闪(呜呜呜).

通常情况下, 涉及到调试的时候, LLDB和GDB差不多, 但是我们还没找到一个与GDB的`load`命令相同功能的LLDB指令, 这个命令把程序加载到硬件上, 所以我们建议你使用GDB. 

## OpenOCD

GDB现在还不能直接通过ST-Link调试器和你的STM32F3DISCOVERY开发板沟通. 
他需要一个翻译器, Open On-Chip Debugger 缩写 OpenOCD 就是这个翻译器. 
OpenOCD是在你电脑上运行的, 可以在GDB基于TCP/IP的远程调试协议和ST-LINK基于USB的协议之间进行转换. 

OpenOCD还执行其他工作, 作为翻译的一部分, 用于调试STM32F3DISCOVERY开发板上的ARM Cortex-M处理器.

* 他知道如何与ARM CoreSight调试外围设备使用的内存映射寄存器沟通. 
  正是这些CoreSight寄存器允许:
  * 断电/观察点操作
  * 读写CPU寄存器
  * 检测CPU何时银调试而暂停
  * 在调试结束后继续CPU执行
  * 更多.
* 它还知道如何擦除和覆写mcu的flash