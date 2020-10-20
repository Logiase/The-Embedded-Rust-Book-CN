# QEMU

我们要开始给[LM3S6965]编程, [LM3S6965]是一个Cortex-M3微控制器.
我们选择这个作为开始是应为它能被QEMU[模拟](https://wiki.qemu.org/Documentation/Platforms/ARM#Supported_in_qemu-system-arm),所以你不必在本部分玩弄硬件,专心与工具与编程.

[LM3S6965]: http://www.ti.com/product/LM3S6965

**Important**
在本教程中我们使用"app"作为项目名.
当你看到"app"这个词的时候,你应该把它换成你给你自己的项目起的名.
或者,你就可以把你的项目名设成"app".

## 创建一个不含标准库的Rust程序

我们会使用[`cortex-m-quickstart`]项目模板来生成一个新项目.
新创建的项目会包含一个基础结构:
一个对嵌入式Rust程序的好的开始.
这个项目还额外包括一个有着几个不同例子的`example`文件夹.

[`cortex-m-quickstart`]: https://github.com/rust-embedded/cortex-m-quickstart

### 使用 `cargo-generate`

首先安装 cargo-generate

``` console
cargo install cargo-generate
```

然后生成一个新项目

```console
cargo generate --git https://github.com/rust-embedded/cortex-m-quickstart
```

```text
 Project Name: app
 Creating project called `app`...
 Done! New project created /tmp/app
```

```console
cd app
```

### 使用`git`

克隆仓库

```console
git clone https://github.com/rust-embedded/cortex-m-quickstart app
cd app
```

然后修改`Cargo.toml`中的占位符

```toml
[package]
authors = ["{{authors}}"] # "{{authors}}" -> "John Smith"
edition = "2018"
name = "{{project-name}}" # "{{project-name}}" -> "awesome-app"
version = "0.1.0"

# ..

[[bin]]
name = "{{project-name}}" # "{{project-name}}" -> "awesome-app"
test = false
bench = false
```

### 两者都不用

下载最新的`cortex-m-quickstart`然后解压

```console
curl -LO https://github.com/rust-embedded/cortex-m-quickstart/archive/master.zip
unzip master.zip
mv cortex-m-quickstart-master app
cd app
```

或者你可以打开[`cortex-m-quickstart`],然后点绿色的"Clone or download",然后选择"Download ZIP".

然后按照第二部分"使用git"中修改占位符.

## 程序概览

为了方便,`src/main.rs`中已经有了很重要的部分:

```rust,ignore
#![no_std]
#![no_main]

use panic_halt as _;

use cortex_m_rt::entry;

#[entry]
fn main() -> ! {
    loop {
        // your code goes here
    }
}
```

这个和标准的Rust程序不太一样,咱们来凑近一点看看.

`#![no_std]`声明这个程序*不会*连接到`std`标准库.
作为代替会连接到`core`

`#![no_main]`声明这个程序不会使用大部分Rust使用的main函数接口.
使用`no_main`主要原因是在`no_std`中使用`main`需要每夜版的Rust.

`use panic_halt as _;`.这个库提供一个定义panic行为的`panic_handler`.
我们会在[Panicing](panicing.md)章节中讨论更多细节.

[`#[entry]`][entry]是一个由[`cortex-m-rt`]提供的属性,用来标记程序的入口.
当我们不用标准的`main`入口我们就需要其他的方式声明程序的入口,就是`#[entry]`.

[entry]: https://docs.rs/cortex-m-rt-macros/latest/cortex_m_rt_macros/attr.entry.html
[`cortex-m-rt`]: https://crates.io/crates/cortex-m-rt

`fn main() -> !`.我们的程序*只会*运行在目标硬件上,所以我们不希望他停止!
我们使用一个[发散函数](https://doc.rust-lang.org/rust-by-example/fn/diverging.html) (`->!`符号)来确保编译期不会出问题.

## 交叉编译

下一步是为Cortex-M3架构进行*交叉*编译.
在知道编译目标时(`$TRIPLE`)使用`cargo build --target $TRIPLE`会很方便.
很幸运,模板中的`.cargo/config`已经提供了答案.

```console
tail -n6 .cargo/config
```

```toml
[build]
# Pick ONE of these compilation targets
# target = "thumbv6m-none-eabi"    # Cortex-M0 and Cortex-M0+
target = "thumbv7m-none-eabi"    # Cortex-M3
# target = "thumbv7em-none-eabi"   # Cortex-M4 and Cortex-M7 (no FPU)
# target = "thumbv7em-none-eabihf" # Cortex-M4F and Cortex-M7F (with FPU)
```

为了给Cortex-M3架构交叉编译,我们要使用`thumbv7m-none-eabi`.
这个编译目标并不是自带的,如果你没有的话,现在就装:

``` console
rustup target add thumbv7m-none-eabi
```

  如果`thumbv7m-none-eabi`已经`.cargo/config`中设为默认值,那下面这两条命令是一样的

```console
cargo build --target thumbv7m-none-eabi
cargo build
```

## 检查

现在我们在`target/thumbv7m-none-eabi/debug/app`有一个非本机的ELF二进制文件.
我们可以用`cargo-binutils`来检查它.

使用`cargo-readobj`来查看ELF头来确认这是个给ARM的二进制文件.

``` console
cargo readobj --bin app -- -file-headers
```

注意:

* `--bin app`是个`target/$TRIPLE/debug/app`的语法糖
* `--bin app`如果需要的话会重新编译

``` text
ELF Header:
  Magic:   7f 45 4c 46 01 01 01 00 00 00 00 00 00 00 00 00
  Class:                             ELF32
  Data:                              2's complement, little endian
  Version:                           1 (current)
  OS/ABI:                            UNIX - System V
  ABI Version:                       0x0
  Type:                              EXEC (Executable file)
  Machine:                           ARM
  Version:                           0x1
  Entry point address:               0x405
  Start of program headers:          52 (bytes into file)
  Start of section headers:          153204 (bytes into file)
  Flags:                             0x5000200
  Size of this header:               52 (bytes)
  Size of program headers:           32 (bytes)
  Number of program headers:         2
  Size of section headers:           40 (bytes)
  Number of section headers:         19
  Section header string table index: 18
```

`cargo-size`可以打印二进制文件中连接器的部分.

```console
cargo size --bin app --release -- -A
```

我们使用`--release`来获取优化的版本.

``` text
app  :
section             size        addr
.vector_table       1024         0x0
.text                 92       0x400
.rodata                0       0x45c
.data                  0  0x20000000
.bss                   0  0x20000000
.debug_str          2958         0x0
.debug_loc            19         0x0
.debug_abbrev        567         0x0
.debug_info         4929         0x0
.debug_ranges         40         0x0
.debug_macinfo         1         0x0
.debug_pubnames     2035         0x0
.debug_pubtypes     1892         0x0
.ARM.attributes       46         0x0
.debug_frame         100         0x0
.debug_line          867         0x0
Total              14570
```

> A refresher on ELF linker sections
>
> - `.text` contains the program instructions
> - `.rodata` contains constant values like strings
> - `.data` contains statically allocated variables whose initial values are
>   *not* zero
> - `.bss` also contains statically allocated variables whose initial values
>   *are* zero
> - `.vector_table` is a *non*-standard section that we use to store the vector
>   (interrupt) table
> - `.ARM.attributes` and the `.debug_*` sections contain metadata and will
>   *not* be loaded onto the target when flashing the binary.

**IMPORTANT**: ELF文件包含了类似Debug信息等等元数据,所以他们的在*磁盘上的大小*并*不能*
准确的反映烧录在硬件上的大小.*通常*使用`cargo-size`来检查二进制文件真正的大小

`cargo-objdump`可用于反汇编二进制文件.

```console
cargo objdump --bin app --release -- --disassemble --no-show-raw-insn --print-imm-hex
```

> **NOTE** 如果以上命令报错`Unknown command line argument`,
> 可以看看这个bug: <https://github.com/rust-embedded/book/issues/269>
>
> **NOTE** 这个根据不同系统有所区别.新版本的rustc,LLVM还有库会生成不同的二进制文件.
> 我们删节了一些说明,意识代码段变小.

```text
app:  file format ELF32-arm-little

Disassembly of section .text:
main:
     400: bl  #0x256
     404: b #-0x4 <main+0x4>

Reset:
     406: bl  #0x24e
     40a: movw  r0, #0x0
     < .. truncated any more instructions .. >

DefaultHandler_:
     656: b #-0x4 <DefaultHandler_>

UsageFault:
     657: strb  r7, [r4, #0x3]

DefaultPreInit:
     658: bx  lr

__pre_init:
     659: strb  r7, [r0, #0x1]

__nop:
     65a: bx  lr

HardFaultTrampoline:
     65c: mrs r0, msp
     660: b #-0x2 <HardFault_>

HardFault_:
     662: b #-0x4 <HardFault_>

HardFault:
     663: <unknown>
```

## 运行

下一步我们要在QEMU上运行我们的嵌入式程序!
这次我们使用`hello`这个例子来搞事.

为了方便,如下是`example/hello.rs`的源码

```rust,ignore
//! Prints "Hello, world!" on the host console using semihosting

#![no_main]
#![no_std]

use panic_halt as _;

use cortex_m_rt::entry;
use cortex_m_semihosting::{debug, hprintln};

#[entry]
fn main() -> ! {
    hprintln!("Hello, world!").unwrap();

    // exit QEMU
    // NOTE do not run this on hardware; it can corrupt OpenOCD state
    debug::exit(debug::EXIT_SUCCESS);

    loop {}
}
```

这个程序使用一个叫semihosting的东西来打印信息到*宿主机*.
等到了真正的硬件上,就需要一个调试会话才能用.

让我们来开始编译这个例子:

```console
cargo build --example hello
```

产出的二进制文件在`target/thumbv7m-none-eabi/debug/examples/hello`.

为了在QEMU上运行这个应使用如下命令:

```console
qemu-system-arm \
  -cpu cortex-m3 \
  -machine lm3s6965evb \
  -nographic \
  -semihosting-config enable=on,target=native \
  -kernel target/thumbv7m-none-eabi/debug/examples/hello
```

```text
Hello, world!
```

这条命令应该在输出信息后成功推出(exit code = 0).
在\*nix上你可以用如下命令确认:

```console
echo $?
```

```text
0
```

让我们来破解QEMU命令:

- `qemu-system-arm`.这是QEMU模拟器.有几种不同的QEMU二进制文件;这个能对ARM机器进行完整的系统仿真

- `-cpu cortex-m3`.这告诉QEMU去模拟一个Cortex-M3 CPU.指定CPU型号可以让我们捕获一些编译错误:
  例如,运行为带有硬件FPU的Cortex-M4F编译的程序回事QEMU执行过程中出错.

- `-machine lm3s6965evb`.这告诉QEMU去模拟LM3S6965EVB,一个包含LM3S6965的评估开发板

- `-nographic`.这告诉QEMu不要去启动GUI.

- `-semihosting-config (..)`.这让QEMU启动semihosting. Semihosting允许仿真设备使用主机的stdout, stderr和stdin,并且在主机上创建文件

- `-kernel $file`.这告诉QEMU运行哪个二进制文件

输入这么长的命令太麻烦了!我们可以在`.cargo/config`中配置一个自定义的运行指令.
去掉注释:

```console
head -n3 .cargo/config
```

```toml
[target.thumbv7m-none-eabi]
# uncomment this to make `cargo run` execute programs on QEMU
runner = "qemu-system-arm -cpu cortex-m3 -machine lm3s6965evb -nographic -semihosting-config enable=on,target=native -kernel"
```

这个运行器只针对`thumbv7m-none-eabi`,现在执行`cargo run`会编译程序并且使用QEMU运行.

```console
cargo run --example hello --release
```

```text
   Compiling app v0.1.0 (file:///tmp/app)
    Finished release [optimized + debuginfo] target(s) in 0.26s
     Running `qemu-system-arm -cpu cortex-m3 -machine lm3s6965evb -nographic -semihosting-config enable=on,target=native -kernel target/thumbv7m-none-eabi/release/examples/hello`
Hello, world!
```

## 调试

调试对于嵌入式开发至关重要.让我们看看它是如何完成的.

调试嵌入式设备设计*远程*调试,因为我们要调试的程序不会运行在运行调试器(GDB or LLDB)所在的机器上

远程调试涉及客户端与服务端.在QEMU设置中,客户端是GDB(LLDB)进程,服务端则是运行嵌入式应用的QEMU程序.

在本节中,我们使用已经编译好的`hello`例子.

调试的第一步是以调试模式启动QEMU:

```console
qemu-system-arm \
  -cpu cortex-m3 \
  -machine lm3s6965evb \
  -nographic \
  -semihosting-config enable=on,target=native \
  -gdb tcp::3333 \
  -S \
  -kernel target/thumbv7m-none-eabi/debug/examples/hello
```

这条命令不会在控制台上打印任何内容并会阻塞终端.这次我们额外传递两个命令行参数:

- `-gdb tcp::3333`.这条命令告诉QEMU在TCP 3333上等待GDB链接

- `-S`.这条命令告诉QEMU在开始时冻结机器.如果没有这条命令,
  还没等我们打开调试器,程序就已经运行到了末尾.

下一步我们在另一个终端中启动GDB,并让它加载示例的调试符:

```console
gdb-multiarch -q target/thumbv7m-none-eabi/debug/examples/hello
```

**注意**取决于你在安装章节安装了哪一个,你可能需要其他版本的gdb而不是`gdb-multiarch`.
这可能是`arm-none-eabi-gdb`或就是`gdb`.

然后在GDB Shell中连接到QEMU,它正在TCP3333上等待连接.

```console
target remote :3333
```

```text
Remote debugging using :3333
Reset () at $REGISTRY/cortex-m-rt-0.6.1/src/lib.rs:473
473     pub unsafe extern "C" fn Reset() -> ! {
```

你会看到该过程已暂停,并且程序计数器指向一个名为`Reset`的函数.
那就是reset handler,MCU在启动时执行的.

>  注意在某些设置中,gdb可能会提示如下信息,而不是显示`Reset () at $REGISTRY/cortex-m-rt-0.6.1/src/lib.rs:473`
>
>`core::num::bignum::Big32x40::mul_small () at src/libcore/num/bignum.rs:254`
> `    src/libcore/num/bignum.rs: No such file or directory.`
> 
> 这是一个已知的故障,你可以放心的忽略它,最有可能出现在Reset()处

这个reset handler最终调用我们的main函数.让我们使用断点与continue跳过.要设置断点的话,首先让我们用`list`看一下在哪断点.

```console
list main
```

这会展示`example/hello.rs`的源码

```text
6       use panic_halt as _;
7
8       use cortex_m_rt::entry;
9       use cortex_m_semihosting::{debug, hprintln};
10
11      #[entry]
12      fn main() -> ! {
13          hprintln!("Hello, world!").unwrap();
14
15          // exit QEMU
```

我们想要在第13行,"Hello world!"后加一个断点.我们可以使用`break`命令

```console
break 13
```

现在,我们可以使用`continue`命令让GDB运行我们的main函数

```console
continue
```

```text
Continuing.

Breakpoint 1, hello::__cortex_m_rt_main () at examples\hello.rs:13
13          hprintln!("Hello, world!").unwrap();
```

我们现在很接近输出"Hello, world!"的那一行代码.让我们用`next`继续.

``` console
next
```

```text
16          debug::exit(debug::EXIT_SUCCESS);
```

这时你应该看到"Hello, world!"已经在运行`qemu-system-arm`的终端上被打印出来了.

```text
$ qemu-system-arm (..)
Hello, world!
```

继续使用`next`会结束QEMU进程.

```console
next
```

```text
[Inferior 1 (Remote target) exited normally]
```

现在你可以退出GDB会话.

``` console
quit
```
