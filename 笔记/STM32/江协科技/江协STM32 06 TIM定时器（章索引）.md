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
updated: 2026-09-26
---

# 06 TIM 定时器（章索引）

> [!info] 这一章在讲什么
> 一个计数器 CNT，加一个预分频器 PSC、一个自动重装器 ARR，构成了**时基单元**。第 6 章全部 8 集都是在时基单元上长出不同的「跟 CNT 打交道的方式」。

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

一条主线，四个分支，全都挂在同一个时基单元上：

```text
                     ┌─ 定时中断（6-1、6-2）        到点了叫我一声
                     │
 时基单元 PSC/CNT/ARR ─┼─ 输出比较 / PWM（6-3、6-4）  到点了把引脚电平翻一下
                     │
                     ├─ 输入捕获（6-5、6-6）         外面来边沿了，把 CNT 抄下来
                     │
                     └─ 编码器接口（6-7、6-8）       两路方波谁先谁后，CNT 自动加减
```

- **6-1 是地基**。把时基单元、更新事件、更新中断讲透，后面每一集都只是换「谁跟 CNT 打交道」。
- **6-2 是 6-1 的变体**：只换时钟源（内部 → 外部引脚），配置骨架完全一样。
- **6-4 / 6-6 / 6-8 是应用集**：代码量大，但原理没有新增，属于前面几集的组合练习。

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
- [[江协STM32 06-3 TIM输出比较]]（待写）
- [[江协STM32 06-4 PWM 应用]]（待写）
- [[江协STM32 06-5 TIM输入捕获]]（待写）
- [[江协STM32 06-6 输入捕获测频率与占空比]]（待写）
- [[江协STM32 06-7 TIM编码器接口]]（待写）
- [[江协STM32 06-8 编码器接口测速]]（待写）

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

> [!tip] 记公式的窍门
> 所有公式里 `PSC` 和 `ARR` 永远是 **+1** 出现。因为寄存器存的是「实际值 − 1」：PSC 写 0 表示 1 分频，ARR 写 0 表示数 1 个数就更新。

## 相邻章节

- 上一章：EXTI 外部中断（[5-1](https://www.bilibili.com/video/BV1th411z7sn?p=11) / [5-2](https://www.bilibili.com/video/BV1th411z7sn?p=12)）
- 下一章：ADC 模数转换器（[7-1](https://www.bilibili.com/video/BV1th411z7sn?p=21) / [7-2](https://www.bilibili.com/video/BV1th411z7sn?p=22)）
- 课程主页：<https://www.bilibili.com/video/BV1th411z7sn>

> [!note] 出处说明
> 本页参数与公式依据课程公开目录、配套讲义与示例代码整理；页码与时长来自 B 站视频分 P 列表。未逐帧核对视频画面，若与视频有出入以视频为准。
