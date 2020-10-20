# Linux

如下是对几种发行版的安装命令

## 包

- Ubuntu 18.04 及以上 / Debian stretch 及以上

> **NOTE** `gdb-mutliarch` 是你debug你的ARM Cortex-M程序的GDB命令

<!-- Debian stretch -->
<!-- GDB 7.12 -->
<!-- OpenOCD 0.9.0 -->
<!-- QEMU 2.8.1 -->

<!-- Ubuntu 18.04 -->
<!-- GDB 8.1 -->
<!-- OpenOCD 0.10.0 -->
<!-- QEMU 2.11.1 -->

``` console
sudo apt install gdb-multiarch openocd qemu-system-arm
```

- Ubuntu 14.04 , 16.04

> **NOTE** `arm-none-eabi-gdb` 是你debug你的ARM Cortex-M程序的GDB命令

<!-- Ubuntu 14.04 -->
<!-- GDB 7.6 (!) -->
<!-- OpenOCD 0.7.0 (?) -->
<!-- QEMU 2.0.0 (?) -->

``` console
sudo apt install gdb-arm-none-eabi openocd qemu-system-arm
```

- Fedora 27 及以上

> **NOTE** `arm-none-eabi-gdb` 是你debug你的ARM Cortex-M程序的GDB命令

<!-- Fedora 27 -->
<!-- GDB 7.6 (!) -->
<!-- OpenOCD 0.10.0 -->
<!-- QEMU 2.10.2 -->

``` console
sudo dnf install arm-none-eabi-gdb openocd qemu-system-arm
```

- Arch Linux

> **NOTE** `arm-none-eabi-gdb` 是你debug你的ARM Cortex-M程序的GDB命令

``` console
sudo pacman -S arm-none-eabi-gdb qemu-arch-extra openocd
```

## udev规则

这条规则让你可以使用OpenOCD而不要root权限.

在`/etc/udev/rules.d/70-st-link.rules`创建文件, 并写入以下内容.

``` text
# STM32F3DISCOVERY rev A/B - ST-LINK/V2
ATTRS{idVendor}=="0483", ATTRS{idProduct}=="3748", TAG+="uaccess"

# STM32F3DISCOVERY rev C+ - ST-LINK/V2-1
ATTRS{idVendor}=="0483", ATTRS{idProduct}=="374b", TAG+="uaccess"
```

然后重新加载udev规则

``` console
sudo udevadm control --reload-rules
```

如果你把板子连接到了电脑, 重新连接.

使用如下命令检查权限:

``` console
lsusb
```

你应该看到如下内容:

```text
(..)
Bus 001 Device 018: ID 0483:374b STMicroelectronics ST-LINK/V2.1
(..)
```

记一下总线设备号. 用这些数字来创建如下目录`/dev/bus/usb/<bus>/<device>`. 然后链接目录:

``` console
ls -l /dev/bus/usb/001/018
```

```text
crw-------+ 1 root root 189, 17 Sep 13 12:34 /dev/bus/usb/001/018
```

```console
getfacl /dev/bus/usb/001/018 | grep user
```

```text
user::rw-
user:you:rw-
```

附加在权限后面的`+`表示存在扩展权限。`getfacl`命令告诉用户`您`可以使用这个设备。

现在阅读[下一部分]

[下一部分]: verify.md
