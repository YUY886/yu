---
course: 江协科技 STM32入门教程-2023版
chapter: 06-1 TIM定时中断
type: 分集笔记
source: https://www.bilibili.com/video/BV1th411z7sn?p=13
tags:
  - STM32
  - 江协科技
  - TIM
  - 定时中断
  - NVIC
status: draft
updated: 2026-09-26
---

# 06-1 TIM 定时中断

> [!abstract] 这一集只解决一个问题
> **怎么让一段代码每隔固定时间自动执行一次？**
>
> 答：让定时器到点产生**更新事件** → 事件经**中断输出控制**送到 **NVIC** → CPU 跳进 `TIMx_IRQHandler` 执行一次，然后回到主循环。
>
> 上一章 [[江协STM32 06 TIM定时器（章索引）|第 6 章索引]] ｜ 下一集 [[江协STM32 06-2 定时器外部时钟]]

## 1 为什么需要定时中断

`Delay_ms()` 那种软件延时是**阻塞**的：等着的时候 CPU 什么都干不了，延时时长还会被中断打断而变长。

定时中断是硬件在数数，CPU 该干嘛干嘛，到点了硬件举手喊一声。适合：

- 周期性任务：LED 闪烁、按键扫描、周期采样
- 需要**准确**时间基准的场合：秒表、时钟、电机换相
- 需要**同时**跑好几件事的程序

## 2 时基单元：三个寄存器

这是整章的核心，务必看懂。

```text
CK_PSC ──►[ 预分频器 PSC ]──► CK_CNT ──►[ 计数器 CNT ]──► CNT == ARR ──► 更新事件
                                        ▲                    │
                                        └────── 清零重来 ─────┘
                                           [ 自动重装器 ARR ] 给 CNT 提供比较目标
```

| 寄存器 | 名字 | 作用 | 位宽 |
| --- | --- | --- | --- |
| **PSC** | 预分频器 | 把输入时钟降速，得到计数节拍 `CK_CNT` | 16 位 |
| **CNT** | 计数器 | 每个节拍加 1，就是「数到几了」 | 16 位 |
| **ARR** | 自动重装寄存器 | 计数目标值，数到它就更新并归零 | 16 位 |

### 2.1 PSC 为什么写「分频系数 − 1」

```text
PSC = 0  → 1 分频 → CK_CNT = CK_PSC           (72 MHz)
PSC = 1  → 2 分频 → CK_CNT = CK_PSC / 2       (36 MHz)
PSC = 2  → 3 分频 → CK_CNT = CK_PSC / 3       (24 MHz)
```

所以：**实际分频系数 = PSC + 1**，`PSC` 最大写 65535，也就是最大 65536 分频。

### 2.2 CNT 数到 ARR 之后还要多等一拍

关键细节：**CNT == ARR 的那一拍不清零**，要等**下一个时钟上升沿**到来才清零，同时产生更新事件。

```text
CNT:  0 → 1 → 2 → ... → ARR → 0 → 1 → ...
                          ↑
                     这里只是一个节拍，下一拍才归零
```

所以一个完整计数周期包含 `ARR + 1` 个计数节拍，不是 `ARR` 个。**公式里永远是 `ARR + 1`**。

### 2.3 ARR 是上限，但计数模式不止一种

| 计数模式 | 行为 | 支持者 |
| --- | --- | --- |
| 向上计数 | 0 → ARR，归零，循环 | 基本、通用、高级 |
| 向下计数 | ARR → 0，重装 ARR，循环 | 通用、高级 |
| 中央对齐 | 0 → ARR → 0，在两端各产生一次更新 | 通用、高级 |

基本定时器（TIM6/TIM7）**只有向上计数**。实际用得最多的也是向上计数。

## 3 更新事件 ≠ 更新中断

这一对概念极易混淆，但分清楚了整章都顺：

|          | 更新事件（Update Event）  | 更新中断（Update Interrupt） |
| -------- | ------------------- | ---------------------- |
| 是什么      | 硬件内部产生的一个信号         | 把事件「上报」给 CPU 的通道       |
| 由谁产生     | CNT 计满 ARR 时自动产生    | 需要 `TIM_ITConfig()` 使能 |
| 作用       | 触发其他电路（如 DAC、其他定时器） | 跳进 `TIMx_IRQHandler`   |
| 是否占用 CPU | 不占                  | 占                      |

**事件照常产生，只是你要不要让它打断 CPU。** 后面 6-3 输出比较里「事件驱动硬件自动化」就是靠事件不靠中断。

要真正进中断，四个条件缺一不可：

1. 定时器产生更新事件
2. `TIM_ITConfig(TIMx, TIM_IT_Update, ENABLE)` 使能了中断输出
3. NVIC 里对应通道已使能
4. CPU 全局中断没被关（`__enable_irq()`）

## 4 定时能力有多大

F103 上定时器输入时钟 `CK_PSC` 通常是 72 MHz。PSC 和 ARR 都拉满：

```text
f_update = 72 000 000 / 65536 / 65536 ≈ 0.01676 Hz
T_update = 1 / 0.01676 ≈ 59.65 s
```

所以单个定时器**最长约 59.65 秒**，接近 1 分钟。

想更长有两个办法：

- **级联**：一个定时器的更新事件（TRGO）接到另一个定时器的 ITR 输入当时钟，两个 65536 相乘，时长变成约 8000 多年。
- **重复计数器 RCR**：只有高级定时器有，再乘一个 65536。

> [!tip] 指数爆炸
> 级联第二级就到「八千多年」，第三级到「34 万亿年」。物理上没意义，但说明「定时不够长」时正确解法是级联而不是死循环。

## 5 周期计算与验算

### 5.1 公式

```text
CK_CNT   = CK_PSC / (PSC + 1)
f_update = CK_PSC / ((PSC + 1) × (ARR + 1))
T_update = (PSC + 1) × (ARR + 1) / CK_PSC
```

### 5.2 本集例程参数

| 参数 | 取值 | 理由 |
| --- | --- | --- |
| 定时器 | TIM2 | 通用定时器，挂 APB1 |
| `PSC` | `7200 - 1` | 72 MHz / 7200 = **10 kHz**，即一个节拍 0.1 ms |
| `ARR` | `10000 - 1` | 10000 拍 × 0.1 ms = **1 s** |
| 计数模式 | 向上 | |

验算：

```text
f_update = 72 000 000 / (7200 × 10000) = 1 Hz
T_update = 1 s
```

> [!warning] 先确认定时器时钟，别想当然
> F103 里 APB1 是 36 MHz，但**只要 APB 预分频系数 ≠ 1，定时器时钟就是 APB 时钟的 2 倍**，所以 TIM2 拿到的是 72 MHz 而不是 36 MHz。换芯片、改主频、改时钟树之后，这条一定要重新算。

## 6 标准库配置六步

```text
① RCC 开启 TIM 时钟
② 选择时基单元时钟源（定时中断 → 内部时钟）
③ 配置时基单元 PSC / ARR / 计数模式
④ 使能更新中断（中断输出控制）
⑤ 配置 NVIC 通道和优先级
⑥ TIM_Cmd 启动计数器
```

### 6.1 Timer.c

```c
#include "stm32f10x.h"

/**
  * 定时器初始化：TIM2 用内部 72MHz 时钟，每 1s 产生一次更新中断
  */
void Timer_Init(void)
{
	/* ① RCC 开启时钟：时钟一开，定时器基准时钟和外设工作时钟就同步打开了 */
	RCC_APB1PeriphClockCmd(RCC_APB1Periph_TIM2, ENABLE);

	/* ② 选择时基单元时钟源：定时中断用内部时钟 */
	TIM_InternalClockConfig(TIM2);

	/* ③ 配置时基单元 */
	TIM_TimeBaseInitTypeDef TIM_TimeBaseInitStructure;
	TIM_TimeBaseInitStructure.TIM_ClockDivision     = TIM_CKD_DIV1;       // 数字滤波采样分频，与 PSC 无关
	TIM_TimeBaseInitStructure.TIM_CounterMode       = TIM_CounterMode_Up; // 向上计数
	TIM_TimeBaseInitStructure.TIM_Period            = 10000 - 1;          // ARR
	TIM_TimeBaseInitStructure.TIM_Prescaler         = 7200 - 1;           // PSC
	TIM_TimeBaseInitStructure.TIM_RepetitionCounter = 0;                  // 仅高级定时器有效
	TIM_TimeBaseInit(TIM2, &TIM_TimeBaseInitStructure);

	/* 手动清一次更新标志：否则计数器刚初始化完就立刻进一次中断 */
	TIM_ClearFlag(TIM2, TIM_FLAG_Update);

	/* ④ 中断输出控制：允许更新中断送到 NVIC */
	TIM_ITConfig(TIM2, TIM_IT_Update, ENABLE);

	/* ⑤ 配置 NVIC。NVIC 是内核外设，库函数被 ST 放在 misc.h 里 */
	NVIC_PriorityGroupConfig(NVIC_PriorityGroup_2);

	NVIC_InitTypeDef NVIC_InitStructure;
	NVIC_InitStructure.NVIC_IRQChannel                   = TIM2_IRQn;
	NVIC_InitStructure.NVIC_IRQChannelPreemptionPriority = 2;
	NVIC_InitStructure.NVIC_IRQChannelSubPriority        = 1;
	NVIC_InitStructure.NVIC_IRQChannelCmd                = ENABLE;
	NVIC_Init(&NVIC_InitStructure);

	/* ⑥ 运行控制：不使能计数器，定时器一动不动 */
	TIM_Cmd(TIM2, ENABLE);
}
```

### 6.2 Timer.h

```c
#ifndef __TIMER_H
#define __TIMER_H

void Timer_Init(void);

#endif
```

> [!note] 和 6-2 的差别
> 本集主循环直接用 `TIM_GetCounter(TIM2)` 读计数值，所以头文件里没有额外声明。到 6-2 视频里才把读值封装成 `Timer_GetCounter()`，头文件里多一行声明。

### 6.3 main.c

```c
#include "stm32f10x.h"
#include "Delay.h"
#include "OLED.h"
#include "Timer.h"

uint16_t Num;

int main(void)
{
	OLED_Init();
	Timer_Init();

	OLED_ShowString(1, 1, "Num:");
	OLED_ShowString(2, 1, "CNT:");

	while (1)
	{
		OLED_ShowNum(1, 5, Num, 5);                 // 中断次数，每秒 +1
		OLED_ShowNum(2, 5, TIM_GetCounter(TIM2), 5); // 当前计数值，0~9999 快速循环
	}
}

/* 定时器 2 的中断服务函数 */
void TIM2_IRQHandler(void)
{
	if (TIM_GetITStatus(TIM2, TIM_IT_Update) == SET)
	{
		Num++;
		TIM_ClearITPendingBit(TIM2, TIM_IT_Update);  // 必须清标志
	}
}
```

现象：`Num` 每秒 +1；`CNT` 以 10 kHz 的速度在 0~9999 之间飞转，肉眼只能看到它在闪。

## 7 时序图里藏的两个细节

### 7.1 PSC 有影子寄存器（缓冲器）

运行中途改 `PSC`，**不会立刻生效**。新值先写进「预分频控制寄存器」，等本次计数周期结束、产生更新事件时，才搬进真正起作用的「缓冲寄存器 / 影子寄存器」。

好处：避免一个计数周期内前半段和后半段频率不一样。

### 7.2 ARR 的预装载（ARPE）

`ARR` 同样有影子寄存器，但是否启用由 `ARPE` 位控制，库函数用 `TIM_ARRPreloadConfig()` 设置。默认不启用，改 `ARR` 立即生效——所以在中断里动态改周期要留意这一点。

> [!tip] 一句话区分
> **PSC 永远带缓冲；ARR 是否带缓冲看你开不开 ARPE。**

## 8 易错点

- [ ] `PSC` 和 `ARR` 忘了减 1 → 定时时长差一个节拍档位。
- [ ] 只调了 `TIM_ITConfig()` 却没配 NVIC → 死活不进中断。
- [ ] 中断里忘了 `TIM_ClearITPendingBit()` → 反复进出中断，主循环卡死。
- [ ] 没在使能中断前 `TIM_ClearFlag()` → 上电就白进一次中断，`Num` 从 1 开始。
- [ ] 把 `TIM_RepetitionCounter` 当成通用定时器能用的参数 → 那是高级定时器的，通用定时器写 0 即可。
- [ ] 中断服务函数里写 `Delay_ms()`、长循环、大量串口打印 → 阻塞其他中断。**正确做法是在中断里置标志位，主循环干活。**
- [ ] 以为「更新事件」就是「更新中断」→ 事件随时有，中断要使能。

## 9 自测

1. `CK_PSC = 72 MHz`，要得到 500 Hz 的更新中断，`PSC = 7200 - 1` 时 `ARR` 应该写多少？
2. 为什么 `ARR` 写 `10000 - 1` 而不是 `10000`？
3. `TIM_ITConfig()` 和 `NVIC_Init()` 各自管的是哪一段？
4. 中断里为什么必须先判断标志位再清标志位？
5. 想定时 10 分钟，用单个 TIM2 能做到吗？该怎么做？

> [!success]- 参考答案
> 1. `ARR + 1 = 72 000 000 / 7200 / 500 = 20`，所以 `ARR = 19`。
> 2. 因为计数从 0 到 ARR 共 `ARR + 1` 拍，写 `10000 - 1` 才是 10000 拍。
> 3. `TIM_ITConfig` 管外设内部「事件 → 中断请求」这一级；`NVIC_Init` 管内核里「中断请求 → CPU 响应」这一级。两级都开才进函数。
> 4. 因为该定时器的中断服务函数可能被多个中断源共用，先判断是谁触发的，避免误清其他标志丢掉中断。
> 5. 不能，单个定时器上限约 59.65 s。可以用中断里软件累加计数，或把两个定时器级联。

## 10 待核对

- [ ] 视频中 OLED 显示的实际刷新表现，以及老师对「CNT 显示抖动」的说明。
- [ ] 视频里 NVIC 优先级分组与抢占/响应优先级的具体取值。
- [ ] 时序图部分的画面讲解细节。

> [!note] 出处说明
> 本页公式、寄存器行为与示例代码依据课程公开目录、配套讲义与示例代码整理；未逐帧核对视频画面，若与视频有出入以视频为准。
