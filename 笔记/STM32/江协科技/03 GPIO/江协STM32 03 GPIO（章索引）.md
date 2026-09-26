---
course: 江协科技 STM32入门教程-2023版
chapter: 03 GPIO
type: 章索引
source: https://www.bilibili.com/video/BV1th411z7sn
tags:
  - STM32
  - 江协科技
  - GPIO
  - 索引
status: draft
updated: 2026-09-26
---

# 03 GPIO（章索引）

> [!info] 这一章在讲什么
> GPIO（General Purpose Input Output，通用输入输出口）是 STM32 所有外设里最简单、但**后面每一章都要用**的一个。第 3 章 4 集只做两件事：**输出高低电平**（3-1、3-2）和**读取高低电平**（3-3、3-4）。
>
> 学完这一章，你才算真正能"用程序点亮一个东西"，也才具备调试后面所有外设的手段。
>
> 上级：[[江协STM32 课程总索引]]

## 覆盖视频

| 集 | 标题 | 页码 | 时长 |
| --- | --- | --- | --- |
| 3-1 | GPIO输出 | [P5](https://www.bilibili.com/video/BV1th411z7sn?p=5) | 34:30 |
| 3-2 | LED闪烁&LED流水灯&蜂鸣器 | [P6](https://www.bilibili.com/video/BV1th411z7sn?p=6) | 39:10 |
| 3-3 | GPIO输入 | [P7](https://www.bilibili.com/video/BV1th411z7sn?p=7) | 43:59 |
| 3-4 | 按键控制LED&光敏传感器控制蜂鸣器 | [P8](https://www.bilibili.com/video/BV1th411z7sn?p=8) | 33:05 |

## 知识地图

```text
              ┌─ 输出：写寄存器 ──► 引脚电平 ──► LED / 蜂鸣器（3-1、3-2）
GPIO 位结构 ──┤
              └─ 输入：读寄存器 ◄── 引脚电平 ◄── 按键 / 光敏模块（3-3、3-4）
```

- **3-1 是地基**。把 GPIO 位结构框图、8 种工作模式、常用库函数讲透，3-2 / 3-3 / 3-4 都只是在调用这几个函数。
- **3-2 是 3-1 的练习**：三个实验共用同一套配置骨架，只换引脚和控制方式。
- **3-3 / 3-4 是输入侧**：3-3 讲输入模式与读取函数，3-4 把按键、光敏传感器接到输入上做闭环。
- **输出侧的三种控制方式**（读写输出数据寄存器、位设置/清除寄存器、位带）在 3-1 讲完，后面各章都不会再重复。

## 本集一句话速览

| 集 | 一句话 | 核心库函数 |
| --- | --- | --- |
| 3-1 | 配置引脚为推挽输出，写 0/1 就能拉低拉高电平 | `RCC_APB2PeriphClockCmd` `GPIO_Init` `GPIO_SetBits` `GPIO_ResetBits` |
| 3-2 | 加上延时和按位取反，让 LED 逐个点亮、让蜂鸣器叫 | `GPIO_Write` `GPIO_WriteBit` `Delay_ms` |
| 3-3 | 配置引脚为上拉/下拉/浮空输入，读回外界电平 | `GPIO_ReadInputDataBit` `GPIO_ReadInputData` |
| 3-4 | 把按键电平接进输入、把判断结果写到输出上 | `GPIO_ReadInputDataBit` `GPIO_SetBits` |

## 子笔记

- [[江协STM32 03-1 GPIO输出]] —— GPIO 位结构框图、8 种工作模式、推挽 vs 开漏、常用库函数与点亮 LED
- [[江协STM32 03-2 LED闪烁与流水灯与蜂鸣器]] —— LED/蜂鸣器硬件电路、Delay 延时模块、三个实验完整代码
- [[江协STM32 03-3 GPIO输入]] —— 输入模式、读取函数、按键与传感器（由他人撰写）
- [[江协STM32 03-4 按键控制LED与光敏传感器]] —— 输入输出联动综合实验（由他人撰写）

## 本章速查

### GPIO 八种工作模式

| 模式名 | 库函数枚举值 | 性质 | 特征 | 典型场景 |
| --- | --- | --- | --- | --- |
| 浮空输入 | `GPIO_Mode_IN_FLOATING` | 数字输入 | 可读引脚电平；引脚悬空时电平不确定 | 外部已有上下拉、USART 接收脚 |
| 上拉输入 | `GPIO_Mode_IPU` | 数字输入 | 内部接上拉电阻，悬空默认高电平 | 按键（另一端接 GND） |
| 下拉输入 | `GPIO_Mode_IPD` | 数字输入 | 内部接下拉电阻，悬空默认低电平 | 按键（另一端接 VCC） |
| 模拟输入 | `GPIO_Mode_AIN` | 模拟输入 | GPIO 数字部分全部关闭，引脚直连 ADC | ADC 采集、低功耗 |
| 开漏输出 | `GPIO_Mode_Out_OD` | 数字输出 | 只能强输出低电平；高电平是高阻态 | 软件 I2C、电平转换（上拉到 5V） |
| 推挽输出 | `GPIO_Mode_Out_PP` | 数字输出 | 高低电平都有驱动能力 | 点灯、驱动蜂鸣器、普通 IO |
| 复用开漏输出 | `GPIO_Mode_AF_OD` | 数字输出 | 由片上外设控制，高电平高阻 | 硬件 I2C 的 SDA/SCL |
| 复用推挽输出 | `GPIO_Mode_AF_PP` | 数字输出 | 由片上外设控制，高低都有驱动能力 | PWM 输出、USART TX、SPI SCK |

### 常用库函数（`stm32f10x_gpio.h`）

| 函数 | 作用 |
| --- | --- |
| `RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOx, ENABLE)` | 开 GPIO 时钟，**用之前必开** |
| `GPIO_Init(GPIOx, &GPIO_InitStructure)` | 按结构体一次性配置引脚模式与速度 |
| `GPIO_SetBits(GPIOx, GPIO_Pin_x)` | 指定引脚输出高电平 |
| `GPIO_ResetBits(GPIOx, GPIO_Pin_x)` | 指定引脚输出低电平 |
| `GPIO_WriteBit(GPIOx, GPIO_Pin_x, BitVal)` | 按 `Bit_SET`/`Bit_RESET` 写单个引脚 |
| `GPIO_Write(GPIOx, PortVal)` | 一次写整个 16 位端口 |
| `GPIO_ReadInputDataBit(GPIOx, GPIO_Pin_x)` | 读引脚电平（输入用） |
| `GPIO_ReadOutputDataBit(GPIOx, GPIO_Pin_x)` | 读输出数据寄存器的值（输出用） |

### 引脚配置速记

```text
GPIOx 挂在 APB2 总线上（GPIOA ~ GPIOG）
每个 GPIOx 有 16 个引脚：Pin_0 ~ Pin_15，可组合如 GPIO_Pin_0 | GPIO_Pin_1，或直接用 GPIO_Pin_All
输出速度三档：GPIO_Speed_10MHz / GPIO_Speed_2MHz / GPIO_Speed_50MHz
引脚电平范围 0 ~ 3.3V；数据手册标 FT（Five-volt Tolerant）的引脚可接入 5V
```

> [!tip] 记这一章最省力的办法
> 所有 GPIO 操作都是同一个三步骨架：**开时钟 → `GPIO_Init` → 读写**。第 3 章 4 集、乃至后面 OLED、串口、I2C 的引脚初始化，都是这个骨架换参数。

## 全章统一易错点

- [ ] 忘记 `RCC_APB2PeriphClockCmd()` 开时钟 → 后面所有配置都"不报错但没反应"。
- [ ] 把不存在的引脚当存在用（如 F103C8T6 没有 GPIOF、GPIOG 的部分引脚）→ 同样不报错、没现象。
- [ ] 输入模式写了 `GPIO_Speed` 就以为有用 → 输出速度只对输出模式有效。
- [ ] 该用复用模式的场合写成普通推挽输出（如 PWM、串口 TX）→ 引脚被普通输出寄存器接管，外设波形出不来。
- [ ] 输出模式下去读 `GPIO_ReadInputDataBit()` 却读的是输出效果 → 要读输出值应该用 `GPIO_ReadOutputDataBit()`。
- [ ] 浮空输入接了悬空引脚 → 电平随机跳变，读数不可信。
- [ ] 引脚电平按 5V 算 → STM32 输出最高只有 3.3V，只有带 FT 的引脚才能承受 5V 输入。

## 相邻章节

- 上一章：[[江协STM32 02 开发环境搭建（章索引）]]（[2-1 P3](https://www.bilibili.com/video/BV1th411z7sn?p=3) / [2-2 P4](https://www.bilibili.com/video/BV1th411z7sn?p=4)）
- 下一章：[[江协STM32 04 OLED（章索引）]]（[4-1 P9](https://www.bilibili.com/video/BV1th411z7sn?p=9) / [4-2 P10](https://www.bilibili.com/video/BV1th411z7sn?p=10)）
- 上级：[[江协STM32 课程总索引]]
- 课程主页：<https://www.bilibili.com/video/BV1th411z7sn>

> [!note] 出处说明
> 本页八种模式、寄存器与库函数依据 ST 标准外设库 `stm32f10x_gpio.h` / `stm32f10x_rcc.h` 与课程公开讲义整理；页码与时长来自 B 站视频分 P 列表。未逐帧核对视频画面，若与视频有出入以视频为准。
