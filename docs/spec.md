# asidepad 规格书 (SPEC)

- 状态：草案 v0.1（立项）
- 日期：2026-09-05

## 1. 产品定位

基于 STM32F103C8T6 的 **USB 有线游戏手柄**，目标主机为 PC，以**标准 HID 游戏手柄**免驱识别（Linux / Windows）。

设计约束（来自项目决策，均为硬性要求）：

1. **全部底层代码开源**，无专有许可组件（规避 ST SLA0044）。
2. **不使用特定 IDE**：arm-none-eabi-gcc + Makefile + OpenOCD，任意编辑器。
3. **底层与逻辑分离**：未来可能更换主控芯片，应用逻辑层不得依赖任何厂商/芯片代码（见 `architecture.md`）。

## 2. 硬件规格（v1 目标）

| 项目 | 规格 |
|---|---|
| MCU | STM32F103C8T6（Cortex-M3 @72MHz，64KB Flash / 20KB RAM，LQFP48） |
| 连接 | USB 2.0 FS 设备，PA11/PA12；**板级必须外接 D+ 1.5kΩ 上拉至 3.3V**（F103 无内部上拉） |
| 时钟 | 8MHz HSE → PLL ×9 = 72MHz；USB 时钟经 USBPRE 取 PLL/1.5 = **48MHz**；APB1 = 36MHz、APB2 = 72MHz |
| 供电 | USB VBUS 5V → 板载 LDO 3.3V |
| 数字输入 | D-Pad ×4（上/下/左/右）、A/B/X/Y、LB/RB、Start/Select —— 共 **12 键**，GPIO 直读 + 软件消抖 |
| 模拟输入 | 左/右摇杆各 X/Y = **4 路 ADC 通道**；ADC 时钟 ≤ 14MHz（APB2 分频 6 → 12MHz） |
| 预留 | L3/R3（摇杆按压）、LT/RT 线性扳机（另 2 路 ADC）、震动马达（PWM）——v1 可选 |
| 指示 | 状态 LED 1–2 颗（枚举成功指示） |

IO 预算（C8T6 可用 GPIO 充足；ADC 外部通道 PA0–PA7 + PB0–PB1 共 10 路，摇杆 4 路 + 扳机 2 路绰绰有余）。

## 3. 主机兼容目标

- HID 描述符：Usage Page 0x01 (Generic Desktop)、Usage 0x05 (Game Pad)。
- 报告格式：按钮位图 + 4 × 轴值（起步 8bit/轴，可升级 16bit）。
- 回报率：全速中断端点 1ms 轮询，目标 **1000 Hz**。
- Linux 经 evdev/js 免驱；Windows 识别为系统 HID 游戏手柄。
- **未决**：是否后续增加 XInput 兼容层（微软私有协议，需模拟 360 手柄描述符，另立 ADR）。

## 4. 软件规格

- 语言/工具链：C11，`arm-none-eabi-gcc`，Makefile，OpenOCD + ST-Link。
- 组件（全开源）：CMSIS（Apache-2.0）+ STM32 **LL**（BSD-3-Clause，仅编译用到的模块）+ **TinyUSB**（MIT）。
- 明确不采用：ST USB Device 中间件（SLA0044）、HAL、CubeMX 生成代码。
- 分层与移植规则：见 `docs/architecture.md`。
- 回报路径：应用层主循环节拍由 USB 轮询驱动（1ms 量级），不引入 RTOS（决策记录见架构文档）。

## 5. 里程碑（firmware）

| # | 内容 | 验收标准 |
|---|---|---|
| M1 | 最小 USB 枚举：TinyUSB HID 设备跑通 | 主机识别为游戏手柄（Linux `lsusb`/`dmesg` 或 Win 设备管理器），板载 LED 亮 |
| M2 | 按键上报：12 键映射到 HID 报告 | `evtest` 或浏览器 Gamepad API 实时显示按键 |
| M3 | 摇杆上报：4 轴 ADC + 中位校准/死区 | 校准工具读数平滑，中位输出 0 |
| M4 | 手感打磨：消抖参数、轴曲线；可选 L3/R3/LT/RT | 主观可用性测试 |
| M5 | 硬件迭代：原理图 → PCB → 打样 | 自研板全功能通过 |

> 注：M1–M4 用 Blue Pill 类最小板即可推进，不必等自研 PCB。

## 6. 未决问题 (Open)

- [ ] 摇杆器件：电位器（便宜、起步）vs 霍尔（寿命/精度，电路不同）——v1 建议电位器
- [ ] L3/R3、线性扳机、震动马达是否进 v1
- [ ] 手柄结构与外壳（3D 打印？）
- [ ] 项目自身开源许可证（候选：MIT）
- [ ] 是否做 XInput 兼容层
