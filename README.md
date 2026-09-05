# asidepad

基于 STM32F103C8T6 的 **USB 有线游戏手柄**：硬件 + 固件一体化项目。

## 简介

- 目标主机：PC，标准 HID 游戏手柄，**免驱**（Linux / Windows）
- 输入 v1：十字键 + A/B/X/Y + LB/RB + Start/Select（12 键）＋ 左右摇杆 2×2 轴（ADC）
- 技术栈约束（硬性）：底层全部**开源许可**（无 ST SLA0044）；**无 IDE 锁定**（arm-none-eabi-gcc + Makefile + OpenOCD）；**底层与逻辑分离**，未来可换主控芯片
- 协议与输入规格详见 [docs/spec.md](docs/spec.md)，分层架构详见 [docs/architecture.md](docs/architecture.md)

## 仓库结构

```
├── docs/          # 设计文档：spec.md（规格书）、architecture.md（分层架构/ADR）
├── hardware/      # 原理图、PCB（KiCad）
├── firmware/      # 固件：app(逻辑) / port(接口) / platform(芯片实现) / third_party
└── README.md
```

## 技术栈（全部开源）

| 组件 | 作用 | 许可证 |
|---|---|---|
| CMSIS Device | 寄存器定义 / startup / 时钟 | Apache-2.0 |
| STM32 LL（取 stm32f1xx_hal_driver 仓库中的 LL 模块） | RCC/GPIO/ADC 驱动 | BSD-3-Clause |
| TinyUSB | USB 栈 + HID 类 | MIT |
| arm-none-eabi-gcc + Makefile + OpenOCD | 编译 / 烧录 | 自由软件 |

明确不采用：ST USB Device 中间件（SLA0044）、HAL、CubeMX。

## 状态

- **立项完成**：规格草案 + 分层架构草案已落盘（docs/）
- **下一步**：里程碑 M1 —— firmware 最小工程骨架（Makefile + linker + 时钟树），TinyUSB HID 枚举跑通
- 里程碑总览见 [docs/spec.md](docs/spec.md) §5

## 参考

- [STM32CubeF1（GitHub，含许可清单）](https://github.com/STMicroelectronics/STM32CubeF1)
- [TinyUSB](https://github.com/hathach/tinyusb)
- 数据手册 / 参考手册笔记：待补充（docs/）
