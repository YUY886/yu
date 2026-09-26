---
course: 江协科技 STM32入门教程-2023版
chapter: 06-4 PWM驱动LED呼吸灯&舵机&直流电机
type: 分集笔记
source: https://www.bilibili.com/video/BV1th411z7sn?p=16
tags:
  - STM32
  - 江协科技
  - TIM
  - PWM
  - 舵机
  - 直流电机
status: draft
verify: 官方源码+接线图+课件
updated: 2026-09-26
---

# 06-4 PWM 驱动 LED 呼吸灯 & 舵机 & 直流电机

> [!abstract] 这一集只解决一个问题
> **怎么让一个引脚自己不停地输出「频率固定、高电平宽度可调」的方波，再用它去调亮度、转角、调速？**
>
> 答：**PSC/ARR 把 CNT 的循环周期定死**（这就是 PWM 周期），**CCR 决定一个周期里高电平占几拍**（这就是占空比）。运行时只改 CCR，就等于改了这个引脚输出的**平均电压**——LED 变亮、舵机转角、电机提速，都是同一个原理。
>
> 上一集 [[江协STM32 06-3 TIM输出比较]] ｜ 章索引 [[江协STM32 06 TIM定时器（章索引）]] ｜ 下一集 [[江协STM32 06-5 TIM输入捕获]]

## 1 PWM 初始化六步

三个实验的负载完全不一样，但初始化流程一模一样，只有「用哪个通道、填什么 PSC/ARR」在变。

```text
① RCC 开 TIM 时钟 + GPIO 时钟（两条总线分别开，缺一不可）
② GPIO 配成复用推挽输出 AF_PP —— 把引脚控制权从 ODR 交给定时器
③ 配时基单元（PSC / ARR / 计数模式）—— 决定 PWM 频率
④ 配输出比较单元 TIM_OCxInit（PWM1 模式 / 极性 / 输出使能 / 初始 CCR）—— 决定占空比
⑤ TIM_Cmd 启动计数器；高级定时器（TIM1/TIM8）还要 TIM_CtrlPWMOutputs 开主输出
⑥ 运行时用 TIM_SetCompareX 改 CCR —— 改亮度 / 角度 / 转速
```

| 步骤 | 关键函数 | 决定什么 | 不做的后果 |
| --- | --- | --- | --- |
| ① 开时钟 | `RCC_APB1PeriphClockCmd` / `RCC_APB2PeriphClockCmd` | 外设能不能工作 | 写寄存器全部无效，引脚不动 |
| ② 配 GPIO | `GPIO_Init` + `GPIO_Mode_AF_PP` | 引脚由谁驱动 | 配成 `Out_PP` 就是 CPU 写 ODR，定时器内部波形再好也出不来 |
| ③ 配时基 | `TIM_TimeBaseInit` | 频率 | 频率不对，舵机不认、电机啸叫 |
| ④ 配输出比较 | `TIM_OCxInit` | 模式/极性/使能/初值 | 通道没使能（`CCxE=0`）→ 引脚恒为默认电平 |
| ⑤ 启动 | `TIM_Cmd` | CNT 开始计数 | 参数都写好了但一动不动 |
| ⑥ 改 CCR | `TIM_SetCompareX` | 占空比 | 只能输出固定脉宽 |

### 1.1 三个核心公式

```text
f_PWM  = CK_PSC / (PSC + 1) / (ARR + 1)      ← 频率，由 PSC 和 ARR 共同决定
Duty   = CCR / (ARR + 1)                     ← 占空比，改 CCR 就能改
Reso   = 1 / (ARR + 1)                       ← 分辨率，ARR 越大越细腻
```

> [!tip] 反推顺序别搞反
> 先由**分辨率要求**定 `ARR`（要 1% 就是 `ARR+1 = 100`），再由**目标频率**定 `PSC`，最后按**目标占空比**写 `CCR`。先定 PSC 会来回凑数。

### 1.2 为什么这一集完全不需要中断

CNT 计数、CNT 与 CCR 比较、引脚翻转，全程由定时器硬件自动完成，**CPU 只在想改亮度/角度/转速时写一次 CCR**。所以这一集不需要：

- `TIM_ITConfig()`（中断输出控制）
- `NVIC_Init()`（内核中断配置）
- `TIMx_IRQHandler()`（中断服务函数）

这也是硬件 PWM 比「定时中断里手动翻转 GPIO」稳定的原因：软件卡一下，波形也不会抖。

> [!note] 和 6-3 的关系
> 6-3 讲的是输出比较单元的**原理**（8 种模式、CNT 与 CCR 的比较关系、极性电路），这一集是**拿它当工具用**：呼吸灯、舵机、直流电机。

## 2 实验一：PWM 驱动 LED 呼吸灯

### 2.1 硬件接线

| 信号 | 引脚 | 丝印 / 颜色 | 说明 |
| --- | --- | --- | --- |
| LED 正极 | **PA0** | — | 即 **TIM2_CH1**，复用推挽输出 |
| LED 负极 | GND | — | 共地 |

极性是「高电平点亮」的正极性接法：

```text
PA0 输出高 → LED 两端有压差 → 有电流 → 亮
PA0 输出低 → 无压差           → 无电流 → 灭
占空比变大 → 平均电流变大 → 视觉上更亮
```

> [!warning] 裸 LED 一定要串限流电阻
> 课程配套接线图画的是裸 LED 直接接 PA0，**没有画串联限流电阻**（已对照官方接线图确认）。课程资料中没有说明这一点；自己复现时应在 LED 回路串 `330 Ω ~ 1 kΩ`，如果用的是自带限流电阻的 LED 小模块，就按模块标注接线。

### 2.2 参数表

| 目标 | 寄存器 | 取值 | 得到什么 |
| --- | --- | --- | --- |
| 分辨率 1% | `ARR` | `100 - 1` | 一个周期 100 拍，CCR 加 1 就是 1% |
| 频率 1 kHz | `PSC` | `720 - 1` | `CK_CNT = 72 MHz / 720 = 100 kHz`，即**1 拍 10 µs** |
| 初始占空比 0% | `CCR` | `0` | 上电先不亮 |
| 占空比调节 | `CCR` | `0 ~ 100` | 100 拍 × 10 µs = **1 ms → 1 kHz** |

验算：

```text
f_PWM = 72 000 000 / 720 / 100 = 1000 Hz = 1 kHz
Duty  = CCR / 100      →  CCR = 37 就是 37% 占空比
```

> [!tip] 这组参数的好处
> `ARR + 1 = 100`，所以 **CCR 的数值恰好等于占空比的百分数**，写代码和看现象都直观。

### 2.3 完整代码

**PWM.c**

```c
#include "stm32f10x.h"                  // Device header

/**
  * 函    数：PWM初始化（TIM2_CH1 → PA0，1kHz，1% 分辨率）
  * 参    数：无
  * 返 回 值：无
  */
void PWM_Init(void)
{
	/* ① 开启时钟：TIM2 挂 APB1，GPIOA 挂 APB2，两条总线都要开 */
	RCC_APB1PeriphClockCmd(RCC_APB1Periph_TIM2, ENABLE);   // 开启TIM2的时钟
	RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOA, ENABLE);  // 开启GPIOA的时钟

	/* ② GPIO 初始化：受外设控制的引脚必须配成复用推挽，控制权交给定时器 */
	GPIO_InitTypeDef GPIO_InitStructure;
	GPIO_InitStructure.GPIO_Mode = GPIO_Mode_AF_PP;        // 复用推挽输出
	GPIO_InitStructure.GPIO_Pin = GPIO_Pin_0;              // TIM2_CH1 默认在 PA0
	GPIO_InitStructure.GPIO_Speed = GPIO_Speed_50MHz;      // 输出速度档位，不是 PWM 频率
	GPIO_Init(GPIOA, &GPIO_InitStructure);

	/* 选择时基单元时钟源：内部时钟（TIM 复位后默认就是它，写出来是为了显式） */
	TIM_InternalClockConfig(TIM2);

	/* ③ 配置时基单元：PSC 和 ARR 一起决定 PWM 频率 */
	TIM_TimeBaseInitTypeDef TIM_TimeBaseInitStructure;
	TIM_TimeBaseInitStructure.TIM_ClockDivision = TIM_CKD_DIV1;      // 滤波器采样分频，与 PSC 无关
	TIM_TimeBaseInitStructure.TIM_CounterMode = TIM_CounterMode_Up;  // 向上计数，PWM 最常用
	TIM_TimeBaseInitStructure.TIM_Period = 100 - 1;                  // ARR：100 拍一个周期
	TIM_TimeBaseInitStructure.TIM_Prescaler = 720 - 1;               // PSC：72MHz/720 = 100kHz
	TIM_TimeBaseInitStructure.TIM_RepetitionCounter = 0;             // 仅高级定时器有效
	TIM_TimeBaseInit(TIM2, &TIM_TimeBaseInitStructure);

	/* ④ 配置输出比较单元：模式、极性、使能、初始 CCR */
	TIM_OCInitTypeDef TIM_OCInitStructure;
	TIM_OCStructInit(&TIM_OCInitStructure);                          // 先赋默认值，避免未赋值成员带随机值
	TIM_OCInitStructure.TIM_OCMode = TIM_OCMode_PWM1;                // PWM模式1：CNT<CCR 时输出有效电平
	TIM_OCInitStructure.TIM_OCPolarity = TIM_OCPolarity_High;        // 有效电平为高（LED 高电平点亮）
	TIM_OCInitStructure.TIM_OutputState = TIM_OutputState_Enable;    // 打开 CC1 通道输出
	TIM_OCInitStructure.TIM_Pulse = 0;                               // 初始 CCR = 0，即 0% 占空比
	TIM_OC1Init(TIM2, &TIM_OCInitStructure);                         // 配置通道1

	/* ⑤ 启动定时器：不使能 CNT 一动不动 */
	TIM_Cmd(TIM2, ENABLE);
}

/**
  * 函    数：PWM设置CCR1
  * 参    数：Compare 要写入的 CCR 的值，范围 0~100
  * 返 回 值：无
  * 注意事项：占空比 Duty = CCR / (ARR + 1)，此函数只改 CCR，不直接是占空比
  */
void PWM_SetCompare1(uint16_t Compare)
{
	TIM_SetCompare1(TIM2, Compare);   // 运行中改 CCR1，波形下一个周期就变
}
```

**PWM.h**

```c
#ifndef __PWM_H
#define __PWM_H

void PWM_Init(void);
void PWM_SetCompare1(uint16_t Compare);

#endif
```

**main.c**

```c
#include "stm32f10x.h"                  // Device header
#include "Delay.h"
#include "OLED.h"
#include "PWM.h"

uint8_t i;                              // for 循环变量

int main(void)
{
	OLED_Init();                        // OLED 初始化
	PWM_Init();                         // PWM 初始化，引脚开始输出 1kHz 方波

	while (1)
	{
		for (i = 0; i <= 100; i++)
		{
			PWM_SetCompare1(i);         // CCR 从 0 加到 100，占空比渐大，LED 渐亮
			Delay_ms(10);               // 每级停 10ms，100 级约 1s 走完
		}
		for (i = 0; i <= 100; i++)
		{
			PWM_SetCompare1(100 - i);   // CCR 从 100 减到 0，占空比渐小，LED 渐暗
			Delay_ms(10);
		}
	}
}
```

### 2.4 现象与原理

现象：LED 由暗到亮、再由亮到暗循环，一圈约 2 s，像是「呼吸」。

为什么改占空比就等于改亮度：

```text
1 kHz 周期 1ms，人眼完全跟不上 → 只看到「平均效果」（视觉暂留 / 惯性系统）
平均电压 ≈ 3.3V × 占空比
CCR=0   → 0%   → 平均电压 0V    → 灭
CCR=50  → 50%  → 平均电压 1.65V → 半亮
CCR=100 → 100% → 平均电压 3.3V  → 全亮
```

这就是 PWM 的本质：**用只有高/低两种状态的数字引脚，等效输出一个连续可调的模拟电压**。同样的道理用在电机上就是调速（机械惯性），用在电容/RC 滤波上就得到真正的模拟电压。

## 3 实验二：PWM 驱动 SG90 舵机

### 3.1 舵机的输入要求

舵机（Servo）内部是「电机 + 减速齿轮 + 电位器 + 控制电路」的闭环，**单片机只负责送命令，不负责出力气**：PWM 在这里只是一根通信线，功率由舵机自己从电源取。

| 项目 | 要求 |
| --- | --- |
| 信号周期 | **20 ms**，即 **50 Hz** |
| 高电平宽度 | **0.5 ms ~ 2.5 ms** |
| 对应角度 | 0.5 ms → **0°**，2.5 ms → **180°**（线性） |

> [!warning] 周期不能随便改
> 舵机认的是「周期 20 ms 里高电平有多宽」，不是认占空比百分比。所以**频率必须先是 50 Hz**，然后再去调脉宽。（1 ms 高电平在 20 ms 周期里是 5%，但在 1 ms 周期里就是 100%，完全不是一回事。）

### 3.2 三根线怎么接

三根线的**功能**是固定的：电源正极、电源负极、控制信号。**线色不是全世界统一标准**——市面上常见两种配色：一种是「红 = 正、棕 = 负、橙 = 信号」，另一种是「红 = 正、黑 = 负、黄/白 = 信号」。下表是按功能列出的（配色只是最常见的参考）：

| 舵机线色（最常见） | 功能 | 电压/信号 | 接到哪 |
| --- | --- | --- | --- |
| **红色** | 电源正极 VCC | 按舵机自身额定电压（SG90 常见 4.8 V ~ 6.0 V，**课程资料中未见该参数**） | 独立 5V 电源正极（或开发板 5V，见下方注意事项） |
| **棕色或黑色** | 电源负极 GND | 0 V 公共地 | 电源负极，**必须与单片机共地** |
| **橙色 / 黄色 / 白色** | 控制信号 SIG | 3.3V/5V PWM，周期 20 ms | 单片机 **PA1**（TIM2_CH2），官方接线图为橙色线接 PA1 |

> [!danger] 三根线绝不能接错
> 颜色不是全世界统一标准：常见有两种配色，**棕色/黑色都是负极**，橙/黄/白都是信号线。接错（尤其是把电源正负极接反）可能瞬间击穿舵机内部电路。上电前先用万用表确认。
>
> **推荐上电顺序**：先接 GND（先建立公共地），再接信号线，最后接 VCC。（这条顺序建议来自通用工程实践，**课程官方资料中未见**。）

#### 为什么舵机电源要独立供

- SG90 空载电流不大，但**堵转/带载时电流会冲到几百 mA 甚至更高**。如果从单片机的 5V/3.3V 引脚取电，会把电源电压拉低，轻则舵机抖动、乱转，重则单片机复位。
- 多个舵机同时动时尤其明显——必须用独立稳压电源（如 5V/2A 以上），绝不能都从控制器 5V 引脚取电。
- 电源容量不足的典型表现就是**抖动**，可在舵机电源引脚处并一只 100~470 µF 电解电容缓解。

#### 为什么必须共地

PWM 是靠「电压高低」表示 0 和 1 的，而电压必须有**共同参考点**。单片机的 GND 和舵机电源的 GND 不连在一起，两边对「高电平」的理解就没有共同基准，信号会紊乱、舵机不响应甚至乱转。

### 3.3 参数表与 50 Hz 算法

```text
PWM 频率必须是 50 Hz → 周期 20 ms

先定 ARR（要 1 个计数对应 1 µs，脉宽好算）：
  PSC = 72 - 1  →  CK_CNT = 72 MHz / 72 = 1 MHz  →  1 拍 = 1 µs
  ARR = 20000 - 1  →  20000 拍 × 1 µs = 20 ms  →  50 Hz ✓
```

| 目标 | 寄存器 | 取值 | 得到什么 |
| --- | --- | --- | --- |
| 1 拍 = 1 µs | `PSC` | `72 - 1` | `CK_CNT = 1 MHz` |
| 周期 20 ms | `ARR` | `20000 - 1` | 20000 拍 = 20 ms → **50 Hz** |
| 脉宽 0.5 ms | `CCR` | `500` | 500 拍 × 1 µs |
| 脉宽 2.5 ms | `CCR` | `2500` | 2500 拍 × 1 µs |
| 脉宽范围 | `CCR` | `500 ~ 2500` | 对应 0° ~ 180° |

> [!tip] 这组参数最妙的地方
> `PSC = 72-1` 让 **1 个计数 = 1 µs**，于是 **CCR 的数值就是高电平的微秒数**。`CCR = 1500` 就是 1.5 ms（舵机中位 90°），看一眼就知道脉宽多少。

### 3.4 角度换算

既然 0.5 ms ↔ 0°，2.5 ms ↔ 180°，而且是线性的：

```text
脉宽(µs) = 500 + 角度 / 180 × (2500 - 500)
        = 角度 / 180 × 2000 + 500        ← 这就是 CCR
```

| 角度 | 计算 | CCR（= 脉宽 µs） | 对应占空比 |
| --- | --- | --- | --- |
| 0° | 0/180×2000+500 | **500** | 2.5% |
| 45° | 45/180×2000+500 | **1000** | 5% |
| 90° | 90/180×2000+500 | **1500** | 7.5% |
| 135° | 135/180×2000+500 | **2000** | 10% |
| 180° | 180/180×2000+500 | **2500** | 12.5% |

> [!note] 这个公式只在「1 拍 = 1 µs、周期 20 ms」时成立
> `×2000` 来自 2.5ms−0.5ms=2ms=2000 拍，`+500` 来自 0.5ms=500 拍。换成别的 PSC/ARR，这两个数都要重算。

### 3.5 完整代码

**PWM.c**（只把实验一的 CH1 换成 CH2）

```c
#include "stm32f10x.h"                  // Device header

/**
  * 函    数：PWM初始化（TIM2_CH2 → PA1，50Hz，CCR 单位就是 µs）
  * 参    数：无
  * 返 回 值：无
  */
void PWM_Init(void)
{
	/* ① 开启时钟 */
	RCC_APB1PeriphClockCmd(RCC_APB1Periph_TIM2, ENABLE);   // 开启TIM2的时钟
	RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOA, ENABLE);  // 开启GPIOA的时钟

	/* ② GPIO 复用推挽输出，注意引脚换成了 PA1 */
	GPIO_InitTypeDef GPIO_InitStructure;
	GPIO_InitStructure.GPIO_Mode = GPIO_Mode_AF_PP;
	GPIO_InitStructure.GPIO_Pin = GPIO_Pin_1;              // TIM2_CH2 默认在 PA1
	GPIO_InitStructure.GPIO_Speed = GPIO_Speed_50MHz;
	GPIO_Init(GPIOA, &GPIO_InitStructure);

	TIM_InternalClockConfig(TIM2);                          // 内部时钟

	/* ③ 时基单元：1 拍 = 1µs，20000 拍 = 20ms = 50Hz */
	TIM_TimeBaseInitTypeDef TIM_TimeBaseInitStructure;
	TIM_TimeBaseInitStructure.TIM_ClockDivision = TIM_CKD_DIV1;
	TIM_TimeBaseInitStructure.TIM_CounterMode = TIM_CounterMode_Up;
	TIM_TimeBaseInitStructure.TIM_Period = 20000 - 1;       // ARR
	TIM_TimeBaseInitStructure.TIM_Prescaler = 72 - 1;       // PSC：72MHz/72 = 1MHz
	TIM_TimeBaseInitStructure.TIM_RepetitionCounter = 0;
	TIM_TimeBaseInit(TIM2, &TIM_TimeBaseInitStructure);

	/* ④ 输出比较单元 */
	TIM_OCInitTypeDef TIM_OCInitStructure;
	TIM_OCStructInit(&TIM_OCInitStructure);
	TIM_OCInitStructure.TIM_OCMode = TIM_OCMode_PWM1;
	TIM_OCInitStructure.TIM_OCPolarity = TIM_OCPolarity_High;      // 高电平为有效脉宽
	TIM_OCInitStructure.TIM_OutputState = TIM_OutputState_Enable;
	TIM_OCInitStructure.TIM_Pulse = 0;                             // 初始 CCR = 0，不发脉冲
	TIM_OC2Init(TIM2, &TIM_OCInitStructure);                       // 配置通道2

	/* ⑤ 启动定时器 */
	TIM_Cmd(TIM2, ENABLE);
}

/**
  * 函    数：PWM设置CCR2
  * 参    数：Compare 要写入的 CCR 的值，范围 500~2500（单位 µs）
  * 返 回 值：无
  */
void PWM_SetCompare2(uint16_t Compare)
{
	TIM_SetCompare2(TIM2, Compare);
}
```

**Servo.c**

```c
#include "stm32f10x.h"                  // Device header
#include "PWM.h"

/**
  * 函    数：舵机初始化
  * 参    数：无
  * 返 回 值：无
  */
void Servo_Init(void)
{
	PWM_Init();                         // 初始化底层 PWM
}

/**
  * 函    数：舵机设置角度
  * 参    数：Angle 要设置的舵机角度，范围 0~180
  * 返 回 值：无
  */
void Servo_SetAngle(float Angle)
{
	/* 角度线性映射到脉宽：0°→500，180°→2500，单位 µs(即 CCR 值) */
	PWM_SetCompare2(Angle / 180 * 2000 + 500);
}
```

**Servo.h**

```c
#ifndef __SERVO_H
#define __SERVO_H

void Servo_Init(void);
void Servo_SetAngle(float Angle);

#endif
```

**main.c**（按键控制角度，OLED 显示）

```c
#include "stm32f10x.h"                  // Device header
#include "Delay.h"
#include "OLED.h"
#include "Servo.h"
#include "Key.h"

uint8_t KeyNum;                         // 接收键码
float Angle;                            // 角度变量，全局默认初值 0

int main(void)
{
	OLED_Init();                        // OLED 初始化
	Servo_Init();                       // 舵机初始化
	Key_Init();                         // 按键初始化

	OLED_ShowString(1, 1, "Angle:");    // 显示静态字符

	while (1)
	{
		KeyNum = Key_GetNum();          // 获取按键键码
		if (KeyNum == 1)                // 按键1按下
		{
			Angle += 30;                // 角度每次加 30°
			if (Angle > 180)            // 超过 180°
			{
				Angle = 0;              // 回到 0°，循环
			}
		}
		Servo_SetAngle(Angle);          // 每次都按当前角度刷新 PWM
		OLED_ShowNum(1, 7, Angle, 3);   // OLED 显示当前角度
	}
}
```

### 3.6 现象

每按一次按键，OLED 上的角度 +30°，舵机转轴跟着转到对应角度：0° → 30° → 60° → … → 180° → 回到 0°，如此循环。

## 4 实验三：PWM 驱动直流电机（TB6612）

### 4.1 为什么 GPIO 不能直接驱动电机

- 直流电机是**大功率器件**，额定电流远超 STM32 单个 GPIO 的能力上限（几十 mA 量级），直接接上去要么转不动，要么把引脚/芯片烧掉。
- 电机转动时还会产生**反向电动势**和很大的**启动浪涌电流**，必须由专门的驱动 IC 做功率放大、方向控制和保护。
- 所以要用 **TB6612FNG**：双路 H 桥，一片能驱动两台直流电机，支持正转、反转、短制动、停止四种状态。

分工很清晰：

```text
STM32  →  只出「逻辑信号」：方向 + 速度指令（小电流）
TB6612 →  出「功率」：按指令把电机电源接到电机两端（大电流）
```

### 4.2 TB6612 引脚与接线

TB6612 模块引脚（**以官方接线图上的模块丝印为准**，本集接线图模块丝印为 `PWMA / AIN2 / AIN1 / STBY / BIN1 / BIN2 / PWMB / GND` 与 `VM / VCC / GND / AO1 / AO2 / BO2 / BO1 / GND`）：

| 模块丝印 | 作用 |
| --- | --- |
| `PWMA` | A 通道 PWM 输入 → 控制**速度** |
| `AIN1` / `AIN2` | A 通道逻辑输入 → 控制**方向** |
| `STBY` | 待机引脚：接低电平待机（不工作），**接高电平才开始工作**；官方接线图里直接接 3.3V |
| `VM` | 电机电源正极（**TB6612 数据手册不在本机 ground-truth 材料里**，通用资料给出的上限 15 V 此处未经官方核对；本集由外接电池盒供电） |
| `VCC` | 逻辑电源正极，接 **3.3 V**（官方接线图接 3.3V） |
| `GND` | 地 |
| `AO1` / `AO2` | A 通道输出，接**电机两根线** |
| `BIN1` / `BIN2` / `PWMB` / `BO1` / `BO2` | B 通道，本实验只驱动一台电机，用不到 |

与 STM32 的接线（**引脚以官方接线图为准**，并与官方源码一一对应）：

| 信号 | STM32 引脚 | TB6612 丝印 | 说明 |
| --- | --- | --- | --- |
| PWM 速度 | **PA2**（TIM2_CH3） | `PWMA` | 复用推挽输出，代码里用 `TIM_SetCompare3` |
| 方向 1 | **PA4** | `AIN1` | 普通推挽输出 `GPIO_Mode_Out_PP` |
| 方向 2 | **PA5** | `AIN2` | 普通推挽输出 `GPIO_Mode_Out_PP` |
| 逻辑电源 | 3.3V | `VCC` | 逻辑供电 |
| 待机控制 | 3.3V（不待机） | `STBY` | 必须为高电平，模块才工作 |
| 地 | GND | `GND` | **与电机电源共地** |
| 电机电源 | 电池盒正极 | `VM` | 电机功率由这里出（官方接线图里是外接电池盒） |
| 电机 | — | `AO1` / `AO2` | 接电机两根线，不分极性（只影响转向） |

### 4.3 控制逻辑

`AIN1` / `AIN2` 的高低电平组合决定运行状态（下表是 TB6612 的通用真值表，**TB6612 数据手册不在本机 ground-truth 材料里**；此处只核对与官方代码不冲突）：

| `AIN1` | `AIN2` | 电机状态 |
| --- | --- | --- |
| 0 | 0 | 停止（滑行） |
| 1 | 0 | 正转 |
| 0 | 1 | 反转 |
| 1 | 1 | 短制动 |

> [!note] 官方代码只用到「正转」和「反转」两行
> `Motor.c` 里 `Speed >= 0` 分支写 (1, 0)，`else` 分支写 (0, 1)，正好对应上表的正转与反转；「停止」和「短制动」两行本课程没有用到，**也未经课程资料核对**。

> [!note] 正反转的「正」是相对的
> 上面的正反转是以「`AO1` 接电机正极、`AO2` 接电机负极」为前提的。把电机两根线对调，正反转就反过来了。

`PWMA` 上的 PWM 占空比决定转速：占空比大 → 电机两端平均电压高 → 转得快。

> [!warning] PWM 极性会影响「数值大 = 快还是慢」
> 如果极性配成低电平有效，那 `TIM_SetCompare3` 写的值越大，输出高电平时间反而越短，电机越慢。本集用 `TIM_OCPolarity_High`，所以**数值大 = 占空比大 = 转得快**。

### 4.4 参数表

| 目标 | 寄存器 | 取值 | 得到什么 |
| --- | --- | --- | --- |
| 频率 20 kHz | `PSC` | `36 - 1` | `CK_CNT = 72 MHz / 36 = 2 MHz` |
| | `ARR` | `100 - 1` | `2 MHz / 100 = 20 kHz` |
| 速度 0~100% | `CCR` | `0 ~ 100` | CCR 数值恰好等于速度百分数 |

验算：

```text
f_PWM = 72 000 000 / 36 / 100 = 20 000 Hz = 20 kHz
Duty  = CCR / 100    →  CCR = 60 就是 60% 占空比
```

> [!tip] 电机 PWM 为什么用 20 kHz
> 20 kHz 已经在人耳听阈上限附近，可以避开电机在音频段（几百 Hz ~ 几 kHz）的**啸叫**，同时保留 1% 的调速分辨率。舵机要 50 Hz 是因为它内部要解调脉宽，电机没有这个约束，可以放心用高频。

### 4.5 完整代码

**PWM.c**（换成 CH3 / PA2，PSC 改成 36-1）

```c
#include "stm32f10x.h"                  // Device header

/**
  * 函    数：PWM初始化（TIM2_CH3 → PA2，20kHz，CCR 0~100 即速度百分数）
  * 参    数：无
  * 返 回 值：无
  */
void PWM_Init(void)
{
	/* ① 开启时钟 */
	RCC_APB1PeriphClockCmd(RCC_APB1Periph_TIM2, ENABLE);   // 开启TIM2的时钟
	RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOA, ENABLE);  // 开启GPIOA的时钟

	/* ② GPIO 复用推挽输出，引脚换成 PA2 */
	GPIO_InitTypeDef GPIO_InitStructure;
	GPIO_InitStructure.GPIO_Mode = GPIO_Mode_AF_PP;
	GPIO_InitStructure.GPIO_Pin = GPIO_Pin_2;              // TIM2_CH3 默认在 PA2
	GPIO_InitStructure.GPIO_Speed = GPIO_Speed_50MHz;
	GPIO_Init(GPIOA, &GPIO_InitStructure);

	TIM_InternalClockConfig(TIM2);                          // 内部时钟

	/* ③ 时基单元：2MHz 计数，100 拍一个周期 → 20kHz */
	TIM_TimeBaseInitTypeDef TIM_TimeBaseInitStructure;
	TIM_TimeBaseInitStructure.TIM_ClockDivision = TIM_CKD_DIV1;
	TIM_TimeBaseInitStructure.TIM_CounterMode = TIM_CounterMode_Up;
	TIM_TimeBaseInitStructure.TIM_Period = 100 - 1;         // ARR
	TIM_TimeBaseInitStructure.TIM_Prescaler = 36 - 1;       // PSC：72MHz/36 = 2MHz
	TIM_TimeBaseInitStructure.TIM_RepetitionCounter = 0;
	TIM_TimeBaseInit(TIM2, &TIM_TimeBaseInitStructure);

	/* ④ 输出比较单元 */
	TIM_OCInitTypeDef TIM_OCInitStructure;
	TIM_OCStructInit(&TIM_OCInitStructure);
	TIM_OCInitStructure.TIM_OCMode = TIM_OCMode_PWM1;
	TIM_OCInitStructure.TIM_OCPolarity = TIM_OCPolarity_High;      // 数值大 = 占空比大 = 转得快
	TIM_OCInitStructure.TIM_OutputState = TIM_OutputState_Enable;
	TIM_OCInitStructure.TIM_Pulse = 0;                             // 初始 CCR = 0，电机先不转
	TIM_OC3Init(TIM2, &TIM_OCInitStructure);                       // 配置通道3

	/* ⑤ 启动定时器 */
	TIM_Cmd(TIM2, ENABLE);
}

/**
  * 函    数：PWM设置CCR3
  * 参    数：Compare 要写入的 CCR 的值，范围 0~100
  * 返 回 值：无
  */
void PWM_SetCompare3(uint16_t Compare)
{
	TIM_SetCompare3(TIM2, Compare);
}
```

**Motor.c**

```c
#include "stm32f10x.h"                  // Device header
#include "PWM.h"

/**
  * 函    数：直流电机初始化
  * 参    数：无
  * 返 回 值：无
  * 说    明：PA4/PA5 是普通推挽输出（方向），PWM 速度信号由 PWM_Init 配在 PA2
  */
void Motor_Init(void)
{
	RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOA, ENABLE);  // 开启GPIOA的时钟

	GPIO_InitTypeDef GPIO_InitStructure;
	GPIO_InitStructure.GPIO_Mode = GPIO_Mode_Out_PP;        // 方向脚是普通推挽，CPU 直接写电平
	GPIO_InitStructure.GPIO_Pin = GPIO_Pin_4 | GPIO_Pin_5;  // PA4→AIN1，PA5→AIN2
	GPIO_InitStructure.GPIO_Speed = GPIO_Speed_50MHz;
	GPIO_Init(GPIOA, &GPIO_InitStructure);

	PWM_Init();                                            // 初始化速度用的 PWM 通道
}

/**
  * 函    数：直流电机设置速度
  * 参    数：Speed 要设置的速度，范围 -100~100，正数正转，负数反转
  * 返 回 值：无
  */
void Motor_SetSpeed(int8_t Speed)
{
	if (Speed >= 0)                         // 正数：正转
	{
		GPIO_SetBits(GPIOA, GPIO_Pin_4);    // AIN1 = 1
		GPIO_ResetBits(GPIOA, GPIO_Pin_5);  // AIN2 = 0
		PWM_SetCompare3(Speed);             // 速度就是占空比
	}
	else                                    // 负数：反转
	{
		GPIO_ResetBits(GPIOA, GPIO_Pin_4);  // AIN1 = 0
		GPIO_SetBits(GPIOA, GPIO_Pin_5);    // AIN2 = 1
		PWM_SetCompare3(-Speed);            // 取绝对值当占空比
	}
}
```

**Motor.h**

```c
#ifndef __MOTOR_H
#define __MOTOR_H

void Motor_Init(void);
void Motor_SetSpeed(int8_t Speed);

#endif
```

**main.c**（按键每按一次加速 20，超过 100 就换向并从 -100 重新递增）

```c
#include "stm32f10x.h"                  // Device header
#include "Delay.h"
#include "OLED.h"
#include "Motor.h"
#include "Key.h"

uint8_t KeyNum;                         // 接收键码
int8_t Speed;                           // 速度变量，-100~100

int main(void)
{
	OLED_Init();                        // OLED 初始化
	Motor_Init();                       // 电机初始化
	Key_Init();                         // 按键初始化

	OLED_ShowString(1, 1, "Speed:");    // 显示静态字符

	while (1)
	{
		KeyNum = Key_GetNum();          // 获取按键键码
		if (KeyNum == 1)                // 按键1按下
		{
			Speed += 20;                // 速度每次加 20
			if (Speed > 100)            // 加到超过 100
			{
				Speed = -100;           // 跳到 -100，即反向全速重新开始
			}
		}
		Motor_SetSpeed(Speed);          // 按当前速度刷新方向脚和 PWM
		OLED_ShowSignedNum(1, 7, Speed, 3);  // OLED 显示带符号的速度
	}
}
```

### 4.6 现象

每按一次按键，`Speed` 加 20，电机转速随之逐级提升：`20 → 40 → … → 100`；再按一次时 `Speed` 已经超过 100，于是跳到 `-100`（反向全速），然后继续 `-80 → -60 → … → 0 → 20 …`，如此循环，实现「正转加速到满速 → 反向全速重新加速」的正反转循环。OLED 上的 `Speed` 同步显示当前速度与符号。

## 5 常用函数速查

| 函数 | 作用 | 用在什么时候 |
| --- | --- | --- |
| `TIM_OC1Init()` ~ `TIM_OC4Init()` | 配置对应通道的输出比较单元：模式（PWM1/PWM2）、极性、输出使能、初始 CCR | **初始化时**，通道 1 用 `TIM_OC1Init`，通道 2 用 `TIM_OC2Init`，以此类推 |
| `TIM_OC1PreloadConfig()` ~ `TIM_OC4PreloadConfig()` | 使能/关闭该通道 CCR 的**预装载**（影子寄存器）：使能后改 CCR 的新值要等下一次更新事件才生效 | 想让占空比在周期边界统一变化、避免半周期突变时开启 |
| `TIM_SetCompare1()` ~ `TIM_SetCompare4()` | **运行中**直接写对应通道的 CCR 值 | 改占空比：呼吸灯改亮度、舵机改角度、电机改速度，最常用 |
| `TIM_CtrlPWMOutputs()` | 高级定时器（TIM1/TIM8）的**主输出使能**（MOE 位） | 高级定时器必须调用，否则引脚没有波形；**通用定时器 TIM2 不需要** |
| `TIM_ARRPreloadConfig()` | 使能/关闭 **ARR 的预装载**（ARPE 位）：控制改 ARR 是立即生效还是等更新事件 | 运行中动态改频率时要注意；默认不启用 |
| `TIM_OCStructInit()` | 给 OC 结构体所有成员赋默认值 | 每次配 OC 之前都该调用，避免未赋值成员带随机值 |
| `TIM_Cmd()` | 启停计数器（CR1.CEN） | 初始化最后一步 |

> [!tip] 一句话记忆
> **初始化用 `TIM_OCxInit`，运行时用 `TIM_SetComparex`；高级定时器多一行 `TIM_CtrlPWMOutputs`。**

## 6 易错点

- [ ] 只开了 TIM 时钟，忘了开 GPIO 时钟（TIM2 在 APB1，GPIOA 在 APB2，两个函数不一样）→ 引脚毫无反应。
- [ ] GPIO 配成了 `GPIO_Mode_Out_PP` 而不是 `GPIO_Mode_AF_PP` → 定时器内部波形正常，但引脚不受定时器控制。
- [ ] `PSC` / `ARR` 忘了减 1 → 频率差一档（`ARR+1` 才是一个周期的拍数）。
- [ ] **把舵机的 50 Hz 和呼吸灯的 1 kHz 参数混用** → 舵机不转或疯狂抖动。
- [ ] 舵机三根线接错（把信号线接到 VCC、电源接反）→ 舵机不动作、抖动，甚至烧毁。
- [ ] 舵机电源**没有和单片机共地** → 信号没有共同参考点，舵机乱转或不响应。
- [ ] 舵机从单片机 5V/3.3V 引脚取电、或多个舵机共用一个弱电源 → 电压被拉低，舵机抖动甚至单片机复位。
- [ ] TB6612 的 `STBY` 忘了接高电平 → 电机完全不动，但代码看起来全对（最难查的一类）。
- [ ] `VCC`（逻辑电源 3.3V）和 `VM`（电机电源）分不清，把电机电源接到 VCC → 烧模块。
- [ ] 电机电源和单片机**不共地** → 逻辑电平没有参考，电机乱转或不转。
- [ ] 想用 GPIO 直接驱动电机 → 转不动或烧引脚，必须经 TB6612 这类驱动。
- [ ] `TIM_OCPolarity` 配成 Low 却没意识到 → `SetCompare` 数值变大反而变暗/变慢。
- [ ] 用高级定时器（TIM1/TIM8）做 PWM 时忘了 `TIM_CtrlPWMOutputs()` → 通用定时器能跑，换到 TIM1 就没波形。
- [ ] 在中断/循环里频繁重配整条初始化流程（`TIM_TimeBaseInit` 等）来改占空比 → 改 CCR 只用 `TIM_SetCompareX` 就够了。

## 7 自测

1. 呼吸灯实验里 `PSC = 720 - 1`、`ARR = 100 - 1`，请算出 PWM 频率和 1 个计数拍的时间；为什么说「CCR 的数值就是占空比的百分数」？
2. 舵机要求周期 20 ms，例程用 `PSC = 72 - 1`、`ARR = 20000 - 1`，请说明这两个值分别是为了满足什么条件；`CCR = 1500` 对应多少毫秒、多少度？
3. `Servo_SetAngle(90)` 最终写进 CCR 的值是多少？写出你的计算过程。
4. 直流电机实验中 `AIN1 = 1, AIN2 = 0` 和 `AIN1 = 0, AIN2 = 1` 分别是什么状态？改变转向应该改 `CCR` 还是改这两个引脚？
5. 为什么呼吸灯用 1 kHz、电机用 20 kHz 都可以，而舵机必须用 50 Hz？如果把舵机的 PWM 也改成 1 kHz 会发生什么？

> [!success]- 参考答案
> 1. `f = 72 000 000 / 720 / 100 = 1000 Hz = 1 kHz`；`CK_CNT = 72 MHz / 720 = 100 kHz`，所以 1 拍 = `1/100kHz = 10 µs`。因为 `ARR + 1 = 100`，`Duty = CCR / 100`，所以 CCR 写 37 就是 37%。
> 2. `PSC = 72 - 1` 是为了让 `CK_CNT = 72 MHz / 72 = 1 MHz`，即 **1 个计数 = 1 µs**；`ARR = 20000 - 1` 是为了凑够 20000 拍 = 20 ms，得到 50 Hz。`CCR = 1500` 就是 1500 µs = **1.5 ms**，正好是 0.5~2.5 ms 的中点，对应 **90°**。
> 3. `CCR = Angle / 180 * 2000 + 500 = 90 / 180 * 2000 + 500 = 1000 + 500 = 1500`。
> 4. `AIN1=1, AIN2=0` 正转；`AIN1=0, AIN2=1` 反转（以 AO1 接电机正极、AO2 接负极为前提）。**改转向改 AIN1/AIN2 这两个 GPIO**；改转速才改 CCR（`TIM_SetCompare3`）。
> 5. 呼吸灯靠人眼视觉暂留、电机靠转子机械惯性，两者都是「惯性系统」，只要频率足够高（远高于人眼/机械能跟随的速度）就只看到平均效果，所以 1 kHz 和 20 kHz 都行（电机用 20 kHz 还能避开啸叫）。**舵机不行**：它内部电路专门解调「20 ms 周期里高电平有多宽」这个约定，周期一变，同样的脉宽被解释成完全不同的角度，舵机会乱转或抖动不响应。

## 8 待核对

- [ ] 舵机三线配色中「红 = 正、棕 = 负、橙 = 信号」这一具体配色的官方出处（本集接线图用橙线接 PA1，但模块丝印只标了功能名，未标颜色代码）。
- [ ] SG90 的额定电压与堵转电流（课程资料中未见；正文只写了「常见 4.8 V ~ 6.0 V」并已标注该参数出处不明）。
- [ ] 舵机在课程板上的供电方式：究竟是从开发板 5V 取电还是外部电池盒，是否需要并 100~470 µF 电容（官方接线图中信号与电源接法可见，但供电来源未标注）。
- [ ] 直流电机实验里电机电源（`VM`）的电压等级与电池盒节数（接线图只画出电池盒，未标电压）。
- [ ] 视频中按键（Key）模块的具体引脚分配与键码映射，以及 OLED 显示格式的最终画面。
- [ ] 电机 PWM 频率选择 20 kHz（`PSC = 36 - 1`）时老师的原话解释。
- [ ] 老师对本集三组 PSC/ARR 取值过程的板书讲解（课件只给出三个公式与算例刻度「0 30 99」）。

## 核对记录（2026-09-26）

> [!success] 核对依据
> - 官方配套源码：`STM32Project-有注释版\6-3 PWM驱动LED呼吸灯\Hardware\PWM.c`、`PWM.h`、`User\main.c`；`6-4 PWM驱动舵机\Hardware\PWM.c`、`PWM.h`、`Hardware\Servo.c`、`Servo.h`、`User\main.c`；`6-5 PWM驱动直流电机\Hardware\PWM.c`、`PWM.h`、`Hardware\Motor.c`、`Motor.h`、`User\main.c`
> - 官方接线图：`ground-truth\接线图\6-3 PWM驱动LED呼吸灯.png`、`6-4 PWM驱动舵机.png`、`6-5 PWM驱动直流电机.png`
> - 课程课件文本：`ground-truth\课件文本.md`（Slide 63~70 输出比较与 PWM、舵机简介；Slide 72 直流电机及驱动简介）
> - 引脚定义表：`ground-truth\F103C8T6引脚定义_缩略.png`
> - ST 标准外设库：`STM32F10x_StdPeriph_Lib_V3.5.0\...\inc\stm32f10x_tim.h`

| 核对项 | 笔记原值 | 官方依据 | 结论 |
| --- | --- | --- | --- |
| 呼吸灯引脚 | PA0（TIM2_CH1） | `6-3 ...\Hardware\PWM.c` 第 22 行 `GPIO_Pin_0`、第 48 行 `TIM_OC1Init`；官方接线图红线接 PA0 | 一致 |
| 呼吸灯 PSC / ARR / CCR | `720 - 1` / `100 - 1` / 初值 `0` | `6-3 ...\PWM.c` 第 34、35、47 行 | 一致 |
| 呼吸灯频率 | 1 kHz | `72 000 000 / 720 / 100 = 1000 Hz`，与官方参数吻合 | 一致 |
| 呼吸灯主循环 | `for i=0..100` 加、再 `100-i` 减，`Delay_ms(10)` | `6-3 ...\User\main.c` 第 16~25 行完全一致 | 一致 |
| 呼吸灯 LED 接法 | 高电平点亮（正极接 PA0、负极接 GND） | 官方接线图红线接 PA0、LED 另一端经蓝线到负极轨 | 一致 |
| 呼吸灯限流电阻 | 接线图未画串联限流电阻 | 官方接线图确认无串联电阻；课件无相关说明 | 一致（已加注「课程资料中没有说明」） |
| 舵机引脚 | PA1（TIM2_CH2） | `6-4 ...\Hardware\PWM.c` 第 17 行 `GPIO_Pin_1`、第 43 行 `TIM_OC2Init`；官方接线图橙色线接 PA1 | 一致 |
| 舵机 PSC / ARR | `72 - 1` / `20000 - 1` | `6-4 ...\PWM.c` 第 29、30 行 | 一致 |
| 舵机频率 | 50 Hz（20 ms） | `72 000 000 / 72 / 20000 = 50 Hz`；课件 Slide 70「周期为 20ms，高电平宽度为 0.5ms~2.5ms」 | 一致 |
| 舵机脉宽范围 | `CCR = 500 ~ 2500`（0.5~2.5 ms，0°~180°） | 课件 Slide 70 原文给出 0.5ms~2.5ms；`PSC=72-1` 使 1 拍 = 1 µs，故 500~2500 拍 | 一致 |
| 角度换算公式 | `CCR = Angle / 180 * 2000 + 500` | `6-4 ...\Hardware\Servo.c` 第 21 行 `PWM_SetCompare2(Angle / 180 * 2000 + 500)` | 一致 |
| 角度换算表 | 0°/45°/90°/135°/180° → 500/1000/1500/2000/2500 | 按官方公式逐个代入计算，5 行全部吻合 | 一致 |
| 舵机 main.c | 按键 1 → `Angle += 30`，`> 180` 归零，`OLED_ShowNum(1, 7, Angle, 3)` | `6-4 ...\User\main.c` 第 22~32 行 | 一致 |
| 电机 PWM 引脚 | PA2（TIM2_CH3） | `6-5 ...\Hardware\PWM.c` 第 17 行 `GPIO_Pin_2`、第 43 行 `TIM_OC3Init`；官方接线图绿线接 PA2 一行 | 一致 |
| 电机 PSC / ARR | `36 - 1` / `100 - 1` | `6-5 ...\PWM.c` 第 29、30 行 | 一致 |
| 电机 PWM 频率 | 20 kHz | `72 000 000 / 36 / 100 = 20 000 Hz` | 一致 |
| 方向控制引脚 | PA4 = AIN1、PA5 = AIN2，`GPIO_Mode_Out_PP` | `6-5 ...\Hardware\Motor.c` 第 15~16 行 `GPIO_Mode_Out_PP` + `GPIO_Pin_4 \| GPIO_Pin_5`；第 32~39 行 `GPIO_SetBits/ResetBits` | 一致（原文未写这两个引脚的 GPIO 模式，已补） |
| 方向逻辑 | `Speed >= 0` → PA4 高 / PA5 低；否则反过来 | `6-5 ...\Hardware\Motor.c` 第 30~40 行 | 一致 |
| `Motor_SetSpeed` 对负数取绝对值 | `PWM_SetCompare3(-Speed)` | `6-5 ...\Hardware\Motor.c` 第 40 行 | 一致 |
| 电机 main.c | `Speed += 20`，`> 100` 时置 `-100`，`OLED_ShowSignedNum` | `6-5 ...\User\main.c` 第 25~34 行 | 一致 |
| 初始化顺序 | ①时钟 ②GPIO(AF_PP) ③时基 ④输出比较 ⑤`TIM_Cmd` | 三个工程的 `PWM.c` 顺序完全一致（GPIO 都在时基之前） | 一致 |
| `TIM_OCInitTypeDef` 成员 | `TIM_OCMode` / `TIM_OCPolarity` / `TIM_OutputState` / `TIM_Pulse` | `stm32f10x_tim.h` 第 82、95、85、92 行；另有 `TIM_OutputNState` 等 4 个高级定时器专用成员（`TIM_OCStructInit` 负责赋默认值） | 一致 |
| 呼吸灯/舵机的 `PWM_SetCompare2` 注释范围 | 「范围 500~2500」 | 官方 `6-4 ...\PWM.c` 第 51 行函数头注释写的是「范围：0~100」，而同一工程 `Servo.c` 实际传入的是 500~2500——**官方函数头注释与实际用法不一致** | 一致（笔记按 `Servo.c` 的实际用法写 500~2500） |
| TB6612 模块丝印 | `PWMA/AIN1/AIN2/STBY/VM/VCC/GND/AO1/AO2` | 官方接线图模块丝印为 `PWMA / AIN2 / AIN1 / STBY / BIN1 / BIN2 / PWMB / GND` + `VM / VCC / GND / AO1 / AO2 / BO2 / BO1 / GND` | 一致（丝印顺序与笔记的列举顺序不同，功能对应无误） |
| `STBY` 接法 | 接高电平（3.3V） | 官方接线图中 `STBY` 引脚直接接到 3.3V | 一致 |
| `VM` 电源 | 电池盒正极 | 官方接线图右下角为外接电池盒，其红色导线经正极轨接到 `VM` | 一致（电池盒节数/电压接线图未标，见「待核对」） |
| 舵机三线配色 | 红=正、棕/黑=负、橙/黄/白=信号 | 课件 Slide 70 只说「周期 20ms、高电平 0.5~2.5ms」，接线图模块标 `GND(棕) / VCC(红) / PWM(橙)` | 一致（与接线图丝印相符） |
| SG90 电压 4.8~6.0 V | 正文写「通常 4.8 V ~ 6.0 V」 | 课件与接线图均未给出该参数 | 已修正（原文为肯定句，现加注「课程资料中未见该参数」） |
| 上电顺序建议 | 「先接 GND、再接信号、最后接 VCC」 | 课件与接线图均未提及 | 已修正（原文为肯定句，现加注「课程官方资料中未见」） |
| TB6612 = 双路 H 桥 | 是，可驱动两台直流电机 | 课件 Slide 72「TB6612 是一款双路 H 桥型的直流电机驱动芯片，可以驱动两个直流电机并且控制其转速和方向」 | 一致 |
| TB6612 真值表（停止/短制动两行） | 列出了 00 停止、11 短制动 | 课件 Slide 72 只给出「双路 H 桥、可驱动两个电机并控制转速和方向」；TB6612 数据手册不在 ground-truth 材料内 | 已修正（原文为无条件肯定句，现注明仅「正转/反转」两行经官方代码印证，另两行未经课程资料核对） |
| 库函数名 | `TIM_OCxInit` / `TIM_SetCompareX` / `TIM_CtrlPWMOutputs` / `TIM_ARRPreloadConfig` / `TIM_OCxPreloadConfig` / `TIM_OCStructInit` / `TIM_Cmd` | `stm32f10x_tim.h` 第 1056~1059、1127~1130、1068、1092、1096~1099、1064、1067 行全部存在 | 一致 |

> [!note] 出处说明
> 本页的引脚（PA0 / PA1 / PA2 / PA4 / PA5）、PSC / ARR / CCR 取值、角度换算公式、方向控制逻辑与示例代码，已逐条对照课程官方配套源码（`6-3 PWM驱动LED呼吸灯`、`6-4 PWM驱动舵机`、`6-5 PWM驱动直流电机` 三个工程的 `Hardware\PWM.c`、`Servo.c`、`Motor.c` 与 `User\main.c`）、官方接线图与课程课件核对；库函数名与结构体成员名已在 ST 标准外设库 `stm32f10x_tim.h` 中逐个查到。此前依据第三方镜像整理的内容已全部用本机官方源码重核。
> **仍未核实**：舵机供电来源与去耦电容、`VM` 的电压等级、SG90 的额定电压与堵转电流、按键引脚与 OLED 显示画面、老师关于 20 kHz 与三组参数取值的原话——这些不在源码、接线图与课件文本中，已留在上面的「待核对」，正文中相应位置也已标注「课程资料中未见」。
