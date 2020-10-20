# Windows

## `arm-none-eabi-gdb`

ARM为Windows提供了`.exe`安装器. 从这[openocd]: https://xpack.github.io/openocd/下载[here][gcc], 然后跟着说i名安装.
在安装结束之前选择"添加到环境变量"选项. 然后验证工具已经在`%PATH%`中:

``` console
$ arm-none-eabi-gdb -v
GNU gdb (GNU Tools for Arm Embedded Processors 7-2018-q2-update) 8.1.0.20180315-git
(..)
```

[gcc]: https://developer.arm.com/open-source/gnu-toolchain/gnu-rm/downloads

## OpenOCD

对Windows现在还没有官方的二进制文件, 但是如果你不想自己编译, xPack项目提供了一个二进制文件. [here][openocd]. 跟着安装说明. 然后更新你的`%PATH%`环境变量. (如果你使用快速安装`C:\Users\USERNAME\AppData\Roaming\xPacks\@xpack-dev-tools\openocd\0.10.0-13.1\.content\bin\`)

[openocd]: https://xpack.github.io/openocd/

验证OpenOCD已经在`%PATH%`中:

``` console
$ openocd -v
Open On-Chip Debugger 0.10.0
(..)
```

## QEMU

从[官方网站][qemu]获取QEMU. 

[qemu]: https://www.qemu.org/download/#windows

## ST-LINK USB驱动

你还需要安装[USB驱动], 否则OpenOCD不能工作. 跟着安装说明并且保证你安装了正确的版本(32位或64位).

[USB驱动]: http://www.st.com/en/embedded-software/stsw-link009.html

这就是全部了, [下一部分]

[下一部分]: verify.md