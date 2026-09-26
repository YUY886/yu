---
course: 江协科技 STM32入门教程-2023版
chapter: 06 TIM定时器
type: 章索引
source: https://www.bilibili.com/video/BV1th411z7sn
tags:
  - STM32
  - 江协科技
  - TIM
  - 索引
status: draft
verify: 官方源码+接线图+课件
updated: 2026-09-26
---

# 06 TIM 定时器（章索引）

> [!info] 这一章在讲什么
> 一个计数器 CNT，加一个预分频器 PSC、一个自动重装器 ARR，构成了**时基单元**。第 6 章全部 8 集都是在时基单元上长出不同的「跟 CNT 打交道的方式」。
>
> 上级：[[江协STM32 课程总索引]]

## 覆盖视频

| 集数 | 标题 | 页码 | 时长 |
| --- | --- | --- | --- |
| 6-1 | TIM定时中断 | [P13](https://www.bilibili.com/video/BV1th411z7sn?p=13) | 49:37 |
| 6-2 | 定时器定时中断&定时器外部时钟 | [P14](https://www.bilibili.com/video/BV1th411z7sn?p=14) | 38:31 |
| 6-3 | TIM输出比较 | [P15](https://www.bilibili.com/video/BV1th411z7sn?p=15) | 44:22 |
| 6-4 | PWM驱动LED呼吸灯&PWM驱动舵机&PWM驱动直流电机 | [P16](https://www.bilibili.com/video/BV1th411z7sn?p=16) | 1:04:25 |
| 6-5 | TIM输入捕获 | [P17](https://www.bilibili.com/video/BV1th411z7sn?p=17) | 38:55 |
| 6-6 | 输入捕获模式测频率&PWMI模式测频率占空比 | [P18](https://www.bilibili.com/video/BV1th411z7sn?p=18) | 35:27 |
| 6-7 | TIM编码器接口 | [P19](https://www.bilibili.com/video/BV1th411z7sn?p=19) | 25:59 |
| 6-8 | 编码器接口测速 | [P20](https://www.bilibili.com/video/BV1th411z7sn?p=20) | 21:04 |

## 知识地图

一条主线，五个分支，全都挂在同一个时基单元上：

```text
                     ┌─ 定时中断（6-1）              到点了叫我一声
                     │
                     ├─ 外部时钟（6-2）              外面来一个脉冲，CNT 加 1
 时基单元 PSC/CNT/ARR ─┼─ 输出比较 / PWM（6-3、6-4）  到点了把引脚电平翻一下
                     │
                     ├─ 输入捕获（6-5、6-6）         外面来边沿了，把 CNT 抄下来
                     │
                     └─ 编码器接口（6-7、6-8）       两路方波谁先谁后，CNT 自动加减
```

- **6-1 是地基**。把时基单元、更新事件、更新中断讲透，后面每一集都只是换「谁跟 CNT 打交道」。
- **6-2 是 6-1 的变体**：只换时钟源（内部 → 外部引脚），配置骨架完全一样。
- **6-4 / 6-6 / 6-8 是应用集**：代码量大，但原理没有新增，属于前面几集的组合练习。

## 八集一句话速览

| 集 | 一句话 | 核心库函数 |
| --- | --- | --- |
| 6-1 | 让代码每隔固定时间自动跑一次 | `TIM_TimeBaseInit` `TIM_ITConfig` `NVIC_Init` |
| 6-2 | 把时钟源从内部换成 PA0 上的外部脉冲 | `TIM_ETRClockMode2Config` |
| 6-3 | CNT 和 CCR 比大小，比出个引脚电平 | `TIM_OCInit` |
| 6-4 | 用 PWM 调 LED 亮度、舵机角度、电机转速 | `TIM_SetCompareX` `TIM_CtrlPWMOutputs`（后者仅高级定时器需要） |
| 6-5 | 边沿一来就把 CNT 抄进 CCR，用来测频 | `TIM_ICInit` `TIM_SelectSlaveMode` |
| 6-6 | 一个通道测周期、一个测高电平，算出频率和占空比 | `TIM_PWMIConfig` |
| 6-7 | 硬件自动判断正反转并加减 CNT | `TIM_EncoderInterfaceConfig` |
| 6-8 | 固定闸门时间读一次 CNT 并清零，换算转速 | `TIM_GetCounter` `TIM_SetCounter` |

## 定时器类型（STM32F103）

| 类型 | 编号 | 总线 | 能力 |
| --- | --- | --- | --- |
| 基本定时器 | TIM6、TIM7 | APB1 | 定时中断、主模式触发 DAC |
| 通用定时器 | TIM2、TIM3、TIM4、TIM5 | APB1 | 基本全部 + 内外时钟源选择、输入捕获、输出比较、编码器接口、主从触发模式 |
| 高级定时器 | TIM1、TIM8 | APB2 | 通用全部 + 重复计数器、死区生成、互补输出、刹车输入 |

三种类型**向下兼容**：高级 ⊃ 通用 ⊃ 基本。

> [!warning] 先查手册再用
> 本课程用的 STM32F103C8T6 只有 **TIM1、TIM2、TIM3、TIM4**。操作一个芯片上不存在的外设不会报错，只是毫无反应——这是最容易白忙半天的坑。

## 子笔记

- [[江协STM32 06-1 TIM定时中断]] —— 时基单元、更新事件与更新中断、标准库配置六步
- [[江协STM32 06-2 定时器外部时钟]] —— 用 ETR 引脚给 CNT 喂外部脉冲
- [[江协STM32 06-3 TIM输出比较]] —— 八种输出模式、PWM 原理与参数计算
- [[江协STM32 06-4 PWM 应用]] —— 呼吸灯、舵机、TB6612 驱动直流电机
- [[江协STM32 06-5 TIM输入捕获]] —— 测频法 vs 测周法、主从触发自动清零
- [[江协STM32 06-6 输入捕获测频率与占空比]] —— 单通道测频、PWMI 测频率和占空比
- [[江协STM32 06-7 TIM编码器接口]] —— 正交编码器、倍频、硬件鉴相
- [[江协STM32 06-8 编码器接口测速]] —— 闸门时间读增量、转速换算

## 全章公式速查

设 `CK_PSC` 为定时器输入时钟（F103 上通常是 72 MHz）。

| 场景 | 公式 |
| --- | --- |
| 计数时钟 | `CK_CNT = CK_PSC / (PSC + 1)` |
| 更新频率 | `f_update = CK_PSC / ((PSC + 1) × (ARR + 1))` |
| 更新周期 | `T_update = (PSC + 1) × (ARR + 1) / CK_PSC` |
| PWM 频率 | `f_PWM = CK_PSC / ((PSC + 1) × (ARR + 1))` |
| PWM 占空比 | `Duty = CCR / (ARR + 1)` |
| PWM 分辨率 | `Reso = 1 / (ARR + 1)` |
| 输入捕获测频 | `f = f_capture / ΔCCR` |
| PWMI 测占空比 | `Duty = T_high / T_period` |
| 编码器每转计数 | `counts = 倍频数 × PPR` |
| 编码器转速 | `rpm = ΔCNT / (倍频数 × PPR) / T × 60` |

> [!tip] 记公式的窍门
> 所有公式里 `PSC` 和 `ARR` 永远是 **+1** 出现。因为寄存器存的是「实际值 − 1」：PSC 写 0 表示 1 分频，ARR 写 0 表示数 1 个数就更新。

## 全章统一易错点

- [ ] `PSC`、`ARR` 都是「实际值减 1」。
- [ ] 先确认具体 TIM 的真实输入时钟：APB1 是 36 MHz，但定时器拿到的是 72 MHz。
- [ ] 定时中断要同时配 TIM 中断、NVIC 和中断服务函数，中断里必须清标志。
- [ ] PWM 占空比是 `CCR / (ARR + 1)`，不是 CCR 本身。
- [ ] 基本定时器没有输入捕获和输出比较，别拿 TIM6/TIM7 去做 PWM。
- [ ] 输入捕获和输出比较**共用** CCR 和引脚，同一个通道不能同时做两件事。
- [ ] 输入捕获测周期要处理计数器溢出和 ±1 误差。
- [ ] 编码器模式下时基单元配的时钟和方向都失效，CNT 由编码器托管。

## 相邻章节

- 上一章：[[江协STM32 05 EXTI外部中断（章索引）]]（[5-1 P11](https://www.bilibili.com/video/BV1th411z7sn?p=11) / [5-2 P12](https://www.bilibili.com/video/BV1th411z7sn?p=12)）
- 下一章：[[江协STM32 07 ADC（章索引）]]（[7-1 P21](https://www.bilibili.com/video/BV1th411z7sn?p=21) / [7-2 P22](https://www.bilibili.com/video/BV1th411z7sn?p=22)）（待写）
- 上级：[[江协STM32 课程总索引]]
- 课程主页：<https://www.bilibili.com/video/BV1th411z7sn>

## 待核对

- [ ] 6-3 与 6-4 的视频分集标题归并方式（本章按「原理集 + 应用集」把 6-3/6-4 分别对应到 P15/P16，而官方工程目录是 6-3/6-4/6-5 三个目录）。
- [ ] 老师的口头讲解原话（本页表格与公式都有官方依据，但「老师当时怎么说的」不在源码与课件文本里）。

> [!note] 关于 B 站分 P 号与时长
> 「覆盖视频」表里的分 P 号（`p=13`~`p=20`）与八个时长**是已核实的一手数据**，来源是 B 站官方分 P 接口
> `https://api.bilibili.com/x/player/pagelist?bvid=BV1th411z7sn`（返回全部 50 个分 P 及各自 `duration` 秒数，例如 6-1 为 2977 秒 = 49:37）。
> 它不在本机官方资料包里，所以上面的「核对依据」没有列它，但它可随时重新查证，不是猜测。

## 核对记录（2026-09-26）

> [!success] 核对依据
> - 官方配套源码：`STM32Project-有注释版\` 下 `6-1 定时器定时中断`、`6-2 定时器外部时钟`、`6-3 PWM驱动LED呼吸灯`、`6-4 PWM驱动舵机`、`6-5 PWM驱动直流电机` 五个工程的 `System\Timer.c`、`Hardware\PWM.c`、`Servo.c`、`Motor.c` 与 `User\main.c`
> - 官方接线图：`ground-truth\接线图\6-1~6-5 *.png`
> - 课程课件文本：`ground-truth\课件文本.md`（Slide 52 定时器简介、53 定时器类型、57~61 时基单元与时序、63~69 输出比较与 PWM、70 舵机、72 直流电机）
> - ST 标准外设库：`STM32F10x_StdPeriph_Lib_V3.5.0\...\inc\stm32f10x_tim.h`

| 核对项 | 笔记原值 | 官方依据 | 结论 |
| --- | --- | --- | --- |
| 工程目录与集号对应 | 6-3 输出比较为理论集；6-4 对应 `6-3 PWM驱动LED呼吸灯`、`6-4 PWM驱动舵机`、`6-5 PWM驱动直流电机` | 三个工程目录实际存在且内容分别对应三个实验；`6-3 PWM驱动LED呼吸灯\Hardware\PWM.c` 里重映射代码是注释，符合「6-3 讲原理」的定位 | 一致 |
| 定时器类型表 | 基本 TIM6/TIM7 APB1、通用 TIM2~TIM5 APB1、高级 TIM1/TIM8 APB2 | 课件 Slide 53 逐字一致 | 一致 |
| 三种类型能力 | 高级 ⊃ 通用 ⊃ 基本 | 课件 Slide 53 的功能描述（「拥有通用定时器全部功能，并额外具有…」） | 一致 |
| C8T6 只有 TIM1~TIM4 | 是 | 课件 Slide 53「STM32F103C8T6 定时器资源：TIM1、TIM2、TIM3、TIM4」 | 一致 |
| 更新频率公式 | `f_update = CK_PSC / ((PSC + 1) × (ARR + 1))` | 课件 Slide 59 `CK_CNT_OV = CK_PSC / (PSC + 1) / (ARR + 1)` | 一致 |
| 计数时钟公式 | `CK_CNT = CK_PSC / (PSC + 1)` | 课件 Slide 58 | 一致 |
| PWM 三公式 | `f_PWM` / `Duty` / `Reso` | 课件 Slide 69 逐字一致 | 一致 |
| 6-1 参数 | `PSC 7200-1`、`ARR 10000-1` | `6-1 ...\System\Timer.c` 第 20~21 行 | 一致 |
| 6-2 参数 | `PSC 1-1`、`ARR 10-1` | `6-2 ...\System\Timer.c` 第 32~33 行 | 一致 |
| 6-4 呼吸灯参数 | `PSC 720-1`、`ARR 100-1` | `6-3 PWM驱动LED呼吸灯\Hardware\PWM.c` 第 34~35 行 | 一致 |
| 6-4 舵机参数 | `PSC 72-1`、`ARR 20000-1` | `6-4 PWM驱动舵机\Hardware\PWM.c` 第 29~30 行 | 一致 |
| 6-4 电机参数 | `PSC 36-1`、`ARR 100-1` | `6-5 PWM驱动直流电机\Hardware\PWM.c` 第 29~30 行 | 一致 |
| 6-2 外部时钟引脚 | PA0（TIM2_ETR） | `6-2 ...\System\Timer.c` 第 25 行注释「注意TIM2的ETR引脚固定为PA0，无法随意更改」；官方接线图绿线接 PA0 | 一致 |
| 6-4 三个实验的引脚 | 呼吸灯 PA0、舵机 PA1、电机 PA2 | `PWM.c` 第 22 行 `GPIO_Pin_0`、`6-4\PWM.c` 第 17 行 `GPIO_Pin_1`、`6-5\PWM.c` 第 17 行 `GPIO_Pin_2`；官方接线图逐一吻合 | 一致 |
| 章节分支图 | 「四个分支」，把 6-1、6-2 合为「定时中断」一支 | 6-2 是独立的时钟源分支（外部时钟模式 2） | 已修正（原为「四个分支／定时中断（6-1、6-2）」） |
| 级联 1 次后的时长 | 「约 8000 多年」 | 按「每级联一级乘 `65536²`」推：`2⁶⁴ / 72 MHz ≈ 8124 年` | 一致（复核后**维持原值**；中途曾被误改为 45.25 天，已回退） |
| 级联 2 次后的时长 | 「34 万亿年」 | `2⁹⁶ / 72 MHz ≈ 3.5 × 10¹³ 年` | 一致（复核后**维持原值**；中途曾被误改为 8124 年，已回退） |
| 公式速查表排版 | `输入捕获测频` 与 `PWMI 测占空比` 挤在同一行 | Markdown 表格每行只能有一条记录 | 已修正（原为两行合并成一行） |
| 「八集一句话速览」 | 6-4 核心库函数写 `TIM_SetCompareX` `TIM_CtrlPWMOutputs` | `TIM_CtrlPWMOutputs` 只对高级定时器必需，6-4 三个实验都用 TIM2 | 已修正（原为并列列出，易被误读为 TIM2 也要调用） |
| 视频页码与时长 | `p=13`~`p=20` 与八个时长 | B 站官方分 P 接口 `api.bilibili.com/x/player/pagelist?bvid=BV1th411z7sn`（返回全部 50 个分 P 的 `duration` 秒数） | 一致（分 P 号与时长逐个对得上） |

> [!note] 出处说明
> 本页的定时器类型与总线、时基单元与三个公式、每个实验的引脚与 PSC/ARR 参数，已逐条对照课程官方配套源码（五个工程的 `System\Timer.c`、`Hardware\PWM.c`、`Servo.c`、`Motor.c`）与官方接线图、课程课件文本核对；此前从第三方镜像（`gitee.com/KSweb/stm32f1doc`）抄来的内容已全部用本机官方材料重核。
> 分 P 号与视频时长来自 B 站官方分 P 接口，已逐个核对。
> **仍未核实**：老师的口头讲解原话与视频画面细节——这类信息不在源码、接线图、课件文本中。本页能保证「技术事实与官方材料一致」，不能保证「与老师原话逐字相同」。
