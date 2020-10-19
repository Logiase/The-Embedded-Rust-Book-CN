# 硬件

先让我们熟悉一下陪我们的开发板

## STM32F3DISCOVERY ("F3")

<p align="center">
<img title="F3" src="../assets/f3.jpg">
</p>

这块板子上都有什么?

- [STM32F303VCT6](https://www.st.com/en/microcontrollers/stm32f303vc.html) mcu. 这块芯片有:
  - 支持单精度浮点的单核ARM Cortex-M4F处理器
  - 256 KiB 闪存 (1 KiB = 10**24** bytes)
  - 48 KiB RAM
  - 很多集成外设, 如 计时器, I2C, SPI, USART
  - 通过标有"USB USER"的USB接口
- 一个[加速度传感器](https://en.wikipedia.org/wiki/Accelerometer) [LSM303DLHC](https://www.st.com/en/mems-and-sensors/lsm303dlhc.html)
- 一个[磁强计](https://en.wikipedia.org/wiki/Magnetometer) [LSM303DLHC](https://www.st.com/en/mems-and-sensors/lsm303dlhc.html)
- 一个[陀螺仪](https://en.wikipedia.org/wiki/Gyroscope) [L3GD20](https://www.pololu.com/file/0J563/L3GD20.pdf)
- 8个呈指南针排列的LED
- 第二个mcu [STM32F103](https://www.st.com/en/microcontrollers/stm32f103cb.html). 实际上是片上编程\调试器ST-LINK的一部分

关于这块板子更进一步的详细信息, 请参阅[STMicroelectronics](https://www.st.com/en/evaluation-tools/stm32f3discovery.html)

警告!: 如果你相对板子施加外部信号, 一定要小心! STM32F303VCT6引脚能承受的电压为3.3V. 更多有关信息, 请参阅用户手册中[6.2 Absolute maximum ratings section in the manual](https://www.st.com/resource/en/datasheet/stm32f303vc.pdf)