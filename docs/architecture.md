# 分层架构与可移植性设计

- 状态：草案 v0.1
- 日期：2026-09-05
- 来源约束：「未来可能更换主控芯片，底层与逻辑必须分离」

## 1. 目标与非目标

**目标**：更换 MCU（或未来更换传输方式，如蓝牙）时，应用逻辑层**零改动**；芯片相关代码面积最小且**整体可替换**。

**非目标**：不做面向任意芯片的通用 BSP 框架，不过度设计。抽象只为"我们可能换的芯片"服务，边界清晰即可。

## 2. 分层总览

```
┌──────────────────────────────────────────────────────────┐
│ app/   手柄逻辑（与芯片、与 USB 栈都无关）                  │
│   gamepad.c   消抖 / 输入状态机 / 轴校准（死区·归一化）      │
│   hid.c       HID 报告描述符 + 报告组包（纯字节流处理）      │
└──────────────────────────────────────────────────────────┘
        │ 只 include 标准 C 与 port/*.h（硬性规则）
┌──────────────────────────────────────────────────────────┐
│ port/  抽象边界：接口定义 + 板级数据                        │
│   board.h        board_init / buttons_raw / axes_raw /    │
│                  tick_ms / led                             │
│   transport.h    send_report / poll（输出通道抽象）         │
│   mapping.h      逻辑控制名 ↔ 引脚/ADC 通道 常量表（纯数据） │
└──────────────────────────────────────────────────────────┘
        │ 接口的实现方（换芯只重写这一块）
┌──────────────────────────────────────────────────────────┐
│ platform/stm32f103/    芯片相关实现（LL / CMSIS / TinyUSB）│
│   main.c   时钟树 LL 初始化、主循环、中断向量               │
│   board.c  GPIO/ADC 初始化与读取（LL）                     │
│   usb.c    TinyUSB glue（tud_task、设备回调 → app）        │
│   startup / linker 脚本 / tusb_config.h                   │
└──────────────────────────────────────────────────────────┘
        │
third_party/   tinyusb (MIT) · STM32F1 CMSIS (Apache-2.0)
               · stm32f1xx HAL 仓库 —— 只取 LL 模块 (BSD-3)
```

## 3. 层职责与设计规则

| 规则 | 内容 |
|---|---|
| R1 | `app/` **零厂商依赖**：不得 include STM32/CMSIS/TinyUSB 头文件，不得引用平台类型 |
| R2 | 跨层通信只走 `port/` 接口；状态显式定义为普通 C 结构体（如 `gamepad_state`） |
| R3 | 板级差异 = **数据**：按键↔引脚、轴↔ADC 通道写进 `mapping` 常量表；改板/改引脚不动代码 |
| R4 | `platform/` 不得反向依赖 `app/` 内部实现；`app/` 对外只暴露 `gamepad_task()` 之类的周期入口 |
| R5 | USB 协议代码尽量可移植：TinyUSB 的 HID 类 API 本身跨芯片，换 MCU 只换 DCD/时钟/引脚配置 |
| R6 | 逻辑层可测试：`app/` 可被 host 端 gcc 编译，配 mock port 做单测（消抖时序、校准曲线、报告字节） |

主循环归属 `platform/`，形态固定为：

```c
while (1) {
    transport_poll();      // 驱动 USB 栈（TinyUSB tud_task）
    gamepad_task();        // 经 port 读输入 → 算状态 → 按需 send_report
}
```

## 4. 换芯演练（可移植性如何兑现）

典型路径：STM32F103 → 另一颗 TinyUSB 支持的 MCU（如 F072/F411/G4，或非 ST 的 RP2040/nRF52 等）。

**必须改的**（改动面收敛到一处）：

1. `platform/<new>/`：startup + linker 脚本；时钟树（注意 USB 48MHz 的来源与分频）；GPIO/ADC 驱动（新厂商 SDK 或 LL）——接口签名不变，只换实现
2. `tusb_config.h` 与 USB 引脚定义（TinyUSB 侧只换 DCD）
3. `mapping` 常量表（新引脚/通道号）
4. Makefile 的 `PLATFORM` 变量

**不动的**：`app/*`、`port/*.h` 接口签名、HID 描述符与报告格式、host 端单测。

**收尾验收**：逻辑层单测保持全绿 + 新板跑通里程碑 M1（枚举）。

## 5. 风险登记

- **TinyUSB × F103**：官方 `dcd_stm32_fsdev` 列入 F103 但未正式测试（风险已在里程碑 M1 前置：枚举不通过则先修 DCD 适配，失败退路为 libopencm3，仅影响 platform 层）。
- **LL/CMSIS 仅限 ST 系**：换非 ST 芯片时 platform 改用该厂商 SDK——被隔离在 platform 内，架构不受影响。
- **未来 MCU 无 TinyUSB 支持**：届时重写 `transport` 实现即可，报告协议层不动；此决策点发生时另立 ADR。

## 6. 决策记录（ADR 摘要）

| 决策 | 选择 | 理由 |
|---|---|---|
| USB 栈 | TinyUSB (MIT) | 许可最干净；HID 类现成；跨 MCU 可移植性最好，与换芯约束契合 |
| 驱动层 | STM32 LL（无 HAL） | 与 TinyUSB 的"仅需 CMSIS"完全兼容；BSD-3；按模块裁剪 |
| 时钟/初始化 | 手写 LL 代码，不用 CubeMX | 无 IDE 锁定；仓库内可见、可审 |
| 实时内核 | 裸机主循环（无 RTOS） | 1ms USB 节拍足够；未来需要时接口不变、可后加 |
| 传输抽象 | `transport` seam，v1 = USB HID | 为未来 BLE/2.4G 留位 |
| 明确排除 | ST USB 中间件 (SLA0044)、HAL | 许可不合规 / 非必要抽象层 |

## 7. 仓库结构对应关系

```
firmware/
├── Makefile          # PLATFORM 变量选择 platform/
├── app/              # gamepad.c  hid.c  （纯逻辑）
├── port/             # board.h  transport.h  mapping.h
├── platform/
│   └── stm32f103/    # main.c  board.c  usb.c  tusb_config.h  startup  linker
└── third_party/      # tinyusb、CMSIS、LL（submodule 或 vendored）
```
