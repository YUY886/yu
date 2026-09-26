---
course: 江协科技 STM32入门教程-2023版
chapter: 03-4 按键控制LED&光敏传感器控制蜂鸣器
type: 分集笔记
source: https://www.bilibili.com/video/BV1th411z7sn?p=8
tags:
  - STM32
  - 江协科技
  - GPIO
  - 按键
  - 光敏传感器
status: draft
verify: 官方源码+接线图+课件
updated: 2026-09-26
---

# 03-4 按键控制 LED & 光敏传感器控制蜂鸣器

> [!abstract] 这一集解决三个问题
> **① 按键怎么控制 LED？② 怎么做到「按一下切换一次」？③ 光敏传感器怎么控制蜂鸣器？**
>
> 答：按键 PB1 / PB11 配**上拉输入**，`Key_GetNum()` 里用**延时 20 ms + 等松手**消抖并返回键码 → 主循环按 `LEDx_Turn()` 翻转 LED 状态（靠 `GPIO_ReadOutputDataBit()` 读回输出寄存器）；光敏模块把光敏电阻的**分压**送 **LM393 比较器**二值化成 **DO** 数字输出，MCU 读 DO 电平决定蜂鸣器响停。**三个实验用的全是 3-3 讲的输入模式和 3-1/3-2 讲的输出模式，没有新外设。**
>
> 上一集 [[江协STM32 03-3 GPIO输入]] ｜ 下一章 [[江协STM32 04 OLED（章索引）]] ｜ 章索引 [[江协STM32 03 GPIO（章索引）]]

## 1 三个实验与硬件接法

本集一共三个实验，硬件全部是「GPIO 输入 + GPIO 输出」的组合：

| 实验 | 输入 | 输入模式 | 输出 | 输出模式 |
| --- | --- | --- | --- | --- |
| ① 按键控制 LED | 按键 K1 / K2 | **上拉输入**（按键接 GND） | LED1 / LED2 | 推挽输出（低电平点亮） |
| ② 按键控制 LED（按一下切换一次） | 同上 | 同上 | 同上 | 同上 |
| ③ 光敏传感器控制蜂鸣器 | 光敏模块 **DO** | **上拉输入** | 蜂鸣器 | 推挽输出（低电平响） |

课程板上的实际接线（已逐条对照官方源码与官方接线图 3-4 / 3-5）：

```text
按键 K1/K2：一端接 GPIO（PB1 / PB11） ── 另一端接 GND 轨  → 引脚配「上拉输入」，按下读到 0
LED1/LED2：阳极经限流电阻接 3.3V 轨 ── 阴极接 GPIO（PA1 / PA2） → 低电平点亮（ResetBits 点亮、SetBits 熄灭）
蜂鸣器：三脚模块 VCC/GND 供电，I/O 接 GPIO（PB12）    → 低电平响（有源蜂鸣器模块）
光敏模块：VCC / GND 供电，DO ── GPIO（PB13）        → 配「上拉输入」，保证引脚不悬空
```

> [!tip] 引脚号已按官方资料核实
> 本页引脚**全部来自官方依据**：LED1/LED2 = **PA1/PA2**（`3-4 按键控制LED\Hardware\LED.c`）、按键 K1/K2 = **PB1/PB11**（`3-4\Hardware\Key.c`）、蜂鸣器 = **PB12**（`3-3 蜂鸣器\User\main.c` 与 `3-5\Hardware\Buzzer.c`）、光敏 DO = **PB13**（`3-5\Hardware\LightSensor.c`），并与官方接线图 `3-4 按键控制LED.png`、`3-5 光敏传感器控制蜂鸣器.png` 中的走线一致。
> 课程也明确说过「按键、LED 的数量和连接的端口都是随意的」，**你自己搭板时引脚号必须和实际接线一致**，否则现象就是「按了没反应」。

## 2 先做一件事：模块化编程

从这一集开始，工程里多一个 **Hardware** 文件夹，把每个硬件的驱动单独封装成 `.c` + `.h`：

```text
工程目录
├── Start      STM32 启动文件、内核与寄存器描述文件（只读，不动）
├── Library    ST 标准外设库（只读，不动）
├── User       main.c、stm32f10x_it.c、stm32f10x_conf.h
└── Hardware   LED.c/h、Key.c/h（3-4 工程）；Buzzer.c/h、LightSensor.c/h（3-5 工程）   ← 本集开始出现
```

要点：

- Keil 里新建一个 `Hardware` **组**，并把 `Hardware` 文件夹加入**头文件搜索路径**（魔术棒 → C/C++ → Include Paths），否则 `#include "Key.h"` 找不到文件。
- 每个 `.h` 都要写**防止重复包含**的宏：

```c
#ifndef __KEY_H
#define __KEY_H

/* 函数声明放这里 */

#endif
```

- `.c` 里只放实现，`.h` 里放「允许外部调用的函数声明」，主函数只管调用。

> [!tip] 为什么要模块化
> 驱动和业务逻辑分开后，主函数只剩「初始化 + 主循环」两件事，换板子只要改驱动里的引脚号。后面章节代码越来越长，这个习惯是刚需。

> [!note] 这一集把 3-2 的代码"拆"成驱动
> 3-2 的 LED 闪烁 / 流水灯 / 蜂鸣器代码是**直接写在 `main.c` 里**的。这一集先把它们搬进 `LED.c`、`Buzzer.c` 两个驱动模块（顺便把 LED 的引脚从 3-2 的 PA0~PA7 改成 **PA1、PA2**），再新建 `Key.c`、`LightSensor.c`。所以 **`LED.c/h`、`Buzzer.c/h` 也是本集新出现的文件**。

### 2.1 LED 驱动（LED.c / LED.h）

```c
#include "stm32f10x.h"                  // Device header

/**
  * 函    数：LED初始化
  * 参    数：无
  * 返 回 值：无
  */
void LED_Init(void)
{
	/*开启时钟*/
	RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOA, ENABLE);		//开启GPIOA的时钟

	/*GPIO初始化*/
	GPIO_InitTypeDef GPIO_InitStructure;
	GPIO_InitStructure.GPIO_Mode = GPIO_Mode_Out_PP;			//推挽输出
	GPIO_InitStructure.GPIO_Pin = GPIO_Pin_1 | GPIO_Pin_2;		//LED1接PA1，LED2接PA2
	GPIO_InitStructure.GPIO_Speed = GPIO_Speed_50MHz;
	GPIO_Init(GPIOA, &GPIO_InitStructure);						//将PA1和PA2引脚初始化为推挽输出

	/*设置GPIO初始化后的默认电平*/
	GPIO_SetBits(GPIOA, GPIO_Pin_1 | GPIO_Pin_2);				//默认高电平，两个LED都熄灭（低电平点亮）
}

/**
  * 函    数：LED1开启
  */
void LED1_ON(void)
{
	GPIO_ResetBits(GPIOA, GPIO_Pin_1);		//PA1输出低电平，LED1点亮
}

/**
  * 函    数：LED1关闭
  */
void LED1_OFF(void)
{
	GPIO_SetBits(GPIOA, GPIO_Pin_1);		//PA1输出高电平，LED1熄灭
}

/**
  * 函    数：LED1状态翻转
  */
void LED1_Turn(void)
{
	if (GPIO_ReadOutputDataBit(GPIOA, GPIO_Pin_1) == 0)		//读输出寄存器：当前输出低电平（灯亮）
	{
		GPIO_SetBits(GPIOA, GPIO_Pin_1);					//则改成高电平（灯灭）
	}
	else													//否则，当前输出高电平（灯灭）
	{
		GPIO_ResetBits(GPIOA, GPIO_Pin_1);					//则改成低电平（灯亮）
	}
}

/**
  * 函    数：LED2开启
  */
void LED2_ON(void)
{
	GPIO_ResetBits(GPIOA, GPIO_Pin_2);		//PA2输出低电平，LED2点亮
}

/**
  * 函    数：LED2关闭
  */
void LED2_OFF(void)
{
	GPIO_SetBits(GPIOA, GPIO_Pin_2);		//PA2输出高电平，LED2熄灭
}

/**
  * 函    数：LED2状态翻转
  */
void LED2_Turn(void)
{
	if (GPIO_ReadOutputDataBit(GPIOA, GPIO_Pin_2) == 0)		//读输出寄存器：当前输出低电平（灯亮）
	{
		GPIO_SetBits(GPIOA, GPIO_Pin_2);					//则改成高电平（灯灭）
	}
	else													//否则，当前输出高电平（灯灭）
	{
		GPIO_ResetBits(GPIOA, GPIO_Pin_2);					//则改成低电平（灯亮）
	}
}
```

```c
#ifndef __LED_H
#define __LED_H

void LED_Init(void);
void LED1_ON(void);
void LED1_OFF(void);
void LED1_Turn(void);
void LED2_ON(void);
void LED2_OFF(void);
void LED2_Turn(void);

#endif
```

### 2.2 蜂鸣器驱动（Buzzer.c / Buzzer.h）

```c
#include "stm32f10x.h"                  // Device header

/**
  * 函    数：蜂鸣器初始化
  */
void Buzzer_Init(void)
{
	/*开启时钟*/
	RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOB, ENABLE);		//开启GPIOB的时钟

	/*GPIO初始化*/
	GPIO_InitTypeDef GPIO_InitStructure;
	GPIO_InitStructure.GPIO_Mode = GPIO_Mode_Out_PP;			//推挽输出
	GPIO_InitStructure.GPIO_Pin = GPIO_Pin_12;					//蜂鸣器接PB12
	GPIO_InitStructure.GPIO_Speed = GPIO_Speed_50MHz;
	GPIO_Init(GPIOB, &GPIO_InitStructure);						//将PB12引脚初始化为推挽输出

	/*设置GPIO初始化后的默认电平*/
	GPIO_SetBits(GPIOB, GPIO_Pin_12);							//默认高电平，蜂鸣器不响（低电平响）
}

/**
  * 函    数：蜂鸣器开启
  */
void Buzzer_ON(void)
{
	GPIO_ResetBits(GPIOB, GPIO_Pin_12);		//PB12输出低电平，蜂鸣器鸣叫
}

/**
  * 函    数：蜂鸣器关闭
  */
void Buzzer_OFF(void)
{
	GPIO_SetBits(GPIOB, GPIO_Pin_12);		//PB12输出高电平，蜂鸣器停止
}

/**
  * 函    数：蜂鸣器状态翻转
  */
void Buzzer_Turn(void)
{
	if (GPIO_ReadOutputDataBit(GPIOB, GPIO_Pin_12) == 0)		//读输出寄存器：当前输出低电平
	{
		GPIO_SetBits(GPIOB, GPIO_Pin_12);						//则改成高电平
	}
	else														//否则，当前输出高电平
	{
		GPIO_ResetBits(GPIOB, GPIO_Pin_12);						//则改成低电平
	}
}
```

```c
#ifndef __BUZZER_H
#define __BUZZER_H

void Buzzer_Init(void);
void Buzzer_ON(void);
void Buzzer_OFF(void);
void Buzzer_Turn(void);

#endif
```

## 3 实验一：按键控制 LED

### 3.1 Key.c / Key.h

```c
#include "stm32f10x.h"                  // Device header
#include "Delay.h"

/**
  * 函    数：按键初始化
  * 参    数：无
  * 返 回 值：无
  */
void Key_Init(void)
{
	/*开启时钟*/
	RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOB, ENABLE);		//开启GPIOB的时钟

	/*GPIO初始化*/
	GPIO_InitTypeDef GPIO_InitStructure;
	GPIO_InitStructure.GPIO_Mode = GPIO_Mode_IPU;				//上拉输入：按键另一端接GND，松手=1，按下=0
	GPIO_InitStructure.GPIO_Pin = GPIO_Pin_1 | GPIO_Pin_11;		//按位或，一次选中PB1和PB11两个引脚
	GPIO_InitStructure.GPIO_Speed = GPIO_Speed_50MHz;			//输入模式下速度无意义，照模板写
	GPIO_Init(GPIOB, &GPIO_InitStructure);						//将PB1和PB11引脚初始化为上拉输入
}

/**
  * 函    数：按键获取键码
  * 参    数：无
  * 返 回 值：按下按键的键码值，范围：0~2，返回0代表没有按键按下
  * 注意事项：此函数是阻塞式操作，当按键按住不放时，函数会卡住，直到按键松手
  */
uint8_t Key_GetNum(void)
{
	uint8_t KeyNum = 0;		//定义变量，默认键码值为0

	if (GPIO_ReadInputDataBit(GPIOB, GPIO_Pin_1) == 0)			//读PB1输入寄存器的状态，如果为0，则代表按键1按下
	{
		Delay_ms(20);											//延时消抖：跳过按下瞬间的抖动区
		while (GPIO_ReadInputDataBit(GPIOB, GPIO_Pin_1) == 0);	//二次确认并等待按键松手
		Delay_ms(20);											//延时消抖：松手瞬间同样有抖动
		KeyNum = 1;												//置键码为1
	}

	if (GPIO_ReadInputDataBit(GPIOB, GPIO_Pin_11) == 0)			//读PB11输入寄存器的状态，如果为0，则代表按键2按下
	{
		Delay_ms(20);											//延时消抖
		while (GPIO_ReadInputDataBit(GPIOB, GPIO_Pin_11) == 0);	//二次确认并等待按键松手
		Delay_ms(20);											//延时消抖
		KeyNum = 2;												//置键码为2
	}

	return KeyNum;			//返回键码值，如果没有按键按下，键码为默认值0
}
```

```c
#ifndef __KEY_H
#define __KEY_H

void Key_Init(void);
uint8_t Key_GetNum(void);

#endif
```

> [!note] 本集新增的输入侧驱动
> 按官方工程划分：**`LED.c/h`、`Key.c/h` 在 `3-4 按键控制LED\Hardware\`**，**`Buzzer.c/h`、`LightSensor.c/h` 在 `3-5 光敏传感器控制蜂鸣器\Hardware\`**（3-5 工程里同时保留了 LED.c/h、Key.c/h）。主函数里只保留「初始化 + 主循环」。

### 3.2 main.c（实验一 + 实验二共用）

```c
#include "stm32f10x.h"                  // Device header
#include "Delay.h"
#include "LED.h"
#include "Key.h"

uint8_t KeyNum;		//定义用于接收按键键码的变量

int main(void)
{
	/*模块初始化*/
	LED_Init();		//LED初始化
	Key_Init();		//按键初始化

	while (1)
	{
		KeyNum = Key_GetNum();		//获取按键键码

		if (KeyNum == 1)			//按键1按下
		{
			LED1_Turn();			//LED1状态翻转（按一下切换一次）
		}

		if (KeyNum == 2)			//按键2按下
		{
			LED2_Turn();			//LED2状态翻转（按一下切换一次）
		}
	}
}
```

现象：按一下 K1，LED1 亮灭翻转一次；按一下 K2，LED2 亮灭翻转一次。

## 4 实验二：「按一下切换一次」是怎么实现的

这是本集最容易踩坑的地方，拆开看是两件事。

### 4.1 坑在哪：没有消抖 = 按一下切换好几次

主循环一秒能跑几十万圈，按键抖动期间引脚电平会反复跳变，**一次按下会被读到很多次**，于是 LED 闪成一团。所以「按一下切换一次」= **消抖** + **状态翻转**两件事的组合。

### 4.2 状态翻转：靠 `GPIO_ReadOutputDataBit()` 读回输出寄存器

```c
/**
  * 函    数：LED1状态翻转
  * 参    数：无
  * 返 回 值：无
  */
void LED1_Turn(void)
{
	if (GPIO_ReadOutputDataBit(GPIOA, GPIO_Pin_1) == 0)		//读输出数据寄存器：当前PA1输出的是低电平（灯亮）
	{
		GPIO_SetBits(GPIOA, GPIO_Pin_1);					//则改成高电平（灯灭）
	}
	else													//否则，即当前输出高电平（灯灭）
	{
		GPIO_ResetBits(GPIOA, GPIO_Pin_1);					//则改成低电平（灯亮）
	}
}
```

**关键：这里用的是 `GPIO_ReadOutputDataBit()`（读 ODR），不是 `GPIO_ReadInputDataBit()`（读 IDR）。** 因为我们要知道的是「自己刚才把它输出成了什么」，好取反；这不需要知道引脚上真实的电压。

> [!tip] 状态翻转不一定非要标志位
> 「状态」这件事，**输出寄存器 ODR 本身就在记着**（3-3 讲的四个读取函数的用途之一）。所以 `LED1_Turn()` 直接读 ODR 取反即可，干净且不需要额外的全局变量。只有在**引脚状态不掌握在自己手里**（比如开漏 + 外部上拉）时才需要单独的软件标志位。

### 4.3 保证「一次按下只算一次」：等松手（课程做法）

`Key_GetNum()` 里的这一行起了决定性作用：

```c
while (GPIO_ReadInputDataBit(GPIOB, GPIO_Pin_1) == 0);	//不松手就一直在这死等
```

只有**松手之后**函数才会返回 1，因此**按住不放时 `Key_GetNum()` 只返回一次非 0 值**，主循环里 `LED1_Turn()` 也就只执行一次。代价是函数是**阻塞**的——按着不放，主循环就停在这里。

### 4.4 标志位法：非阻塞的等价实现

课程在后续章节处理「按键既要点动又要长按」时，会改成**标志位（状态标记）**的写法。效果一样是「按一下切换一次」，但主循环不会被卡住：

```c
uint8_t Key_Flag = 0;	//标志位：0 = 上一次按下已经处理完（按键已释放），1 = 正在按住且已处理过

int main(void)
{
	LED_Init();
	Key_Init();

	while (1)
	{
		if (GPIO_ReadInputDataBit(GPIOB, GPIO_Pin_1) == 0)	//当前读到按键按下
		{
			if (Key_Flag == 0)								//标志位为0，说明这是一次「新的按下」
			{
				Delay_ms(20);								//延时消抖
				if (GPIO_ReadInputDataBit(GPIOB, GPIO_Pin_1) == 0)	//二次确认仍为按下
				{
					LED1_Turn();							//状态翻转，只执行这一次
					Key_Flag = 1;							//置标志位：按住不放也不会再翻转
				}
			}
		}
		else												//当前读到按键松开
		{
			Key_Flag = 0;									//清标志位，为下一次按下做准备
		}
	}
}
```

**标志位的本质**：它记住的是「这次按下我处理过了没有」。只要不松手，`Key_Flag` 一直是 1，翻转就只发生一次；松手后清 0，下一次按下才重新有效。

| | 键码 + 等松手（课程 `Key_GetNum`） | 标志位（4.4 写法） |
| --- | --- | --- |
| 一次按下触发次数 | 1 次 | 1 次 |
| 是否阻塞主循环 | **阻塞**（按住不放就卡住） | **非阻塞** |
| 代码复杂度 | 低，封装成 `Key_GetNum()` 复用 | 略高，每个按键要有自己的标志位 |
| 可扩展性 | 难以扩展长按、连按 | 容易扩展长按、双击 |

> [!warning] 标志位必须「轮到松手才清」
> 如果把 `Key_Flag = 0` 写在按下分支里（而不是松开分支），标志位立刻被清掉，下一圈主循环又会翻转一次——就退化成了「按住就狂闪」，等于没写。

## 5 实验三：光敏传感器控制蜂鸣器

### 5.1 光敏传感器模块原理：分压 → 比较器 → 二值化

光敏电阻（photoresistor / LDR）的阻值随光照变化：**光线越强，阻值越小**；光线越暗，阻值越大。

模块内部的基本链路：

```text
        VCC
         │
      [ R1 ]  定值电阻（上拉，与光敏电阻分压）
         │
         ├──────────► AO  模拟输出（分压点电压，连续的模拟量）
         │
      [ N1 ]  光敏电阻（下拉，阻值随光照变化）
         │
        GND

   分压点电压 ──►[ LM393 电压比较器 ]──► DO  数字输出（二值化后的 0/1）
                      ▲
                 [ 电位器 ] 提供比较阈值（参考电压）
```

- **分压**：R1 与光敏电阻 N1 串联在 VCC 与 GND 之间，中间节点的电压就是 AO。用「上下拉弹簧」的直觉理解：**哪边阻值小，哪边拉力强，节点电压就偏向哪边**。这里 N1 是下拉，所以**光线越强 → N1 阻值越小 → 下拉越强 → 分压点电压越低**。
- **二值化**：LM393 是**电压比较器（voltage comparator）**，本质是一个开环运放。当同相输入端电压 > 反相输入端电压时输出瞬间拉到 VCC，反之输出瞬间拉到 GND。它把连续的模拟电压变成干净的 0/1，这就是「二值化」。
- **滤波电容**：分压输出处还并了一个滤波电容，用来滤掉干扰、让输出波形平滑（分析电路时可以先把它抹掉）。

> [!note] 这段电路原理说明的依据
> 课件文本 Slide 30 的原文只讲到：「传感器模块：传感器元件（光敏电阻 / 热敏电阻 / 红外接收管等）的电阻会随外界模拟量的变化而变化，通过与定值电阻分压即可得到模拟电压输出，再通过电压比较器进行二值化即可得到数字电压输出」——**「分压 → 电压比较器 → 二值化」这条链路是课件原文**。
> 下面具体到 **LM393 型号**、定值电阻、滤波电容与电位器的电路细节，**在课件文本与官方源码中均未见**（官方 `模块资料.zip\4-光敏电阻传感器\` 内有模块说明书与电路图，本次未能读取），故按「常见模块电路」的通行讲法描述，未当作官方结论，见 `## 待核对`。

### 5.2 DO 数字输出 vs AO 模拟输出

| | **DO**（Digital Output） | **AO**（Analog Output） |
| --- | --- | --- |
| 信号性质 | 二值化后的 **0 / 1** | 分压得到的**连续模拟电压** |
| 由谁产生 | LM393 比较器的输出 | R1 与光敏电阻分压的中间节点 |
| 接单片机的什么 | 任意 **GPIO 输入**（本实验用上拉输入） | 必须接 **ADC 引脚**（后面 ADC 章节才用到） |
| 能告诉单片机什么 | 「比阈值亮 / 比阈值暗」这一条信息 | 具体的光照强度数值 |
| 本实验用哪个 | ✅ **用 DO** | ❌ 不用（要 ADC 才能读） |

> [!note] 本实验只读 DO
> 本集只是「让光线暗到一定程度就响」，只需要一个是/否，所以用 DO 接普通 GPIO 就够了。AO 要等学过 ADC 才能用——那时可以读回具体的光照强度。

### 5.3 输出电平与遮挡的关系、为什么要调电位器

- **现象**：模块上电后指示灯亮；**遮住光敏电阻（变暗）时，输出指示灯灭**，代表 DO 输出**高电平**；松开（变亮）时输出指示灯亮，代表 DO 输出**低电平**。
- **原因**：变暗 → 光敏电阻阻值变大 → 下拉变弱 → 分压点电压升高 → 比较器翻转，DO 输出高电平。
  **所以：光线暗 → DO = 1；光线强 → DO = 0。**
- **电位器的作用**：比较器有两个输入——一个是分压点的电压，另一个是**电位器分压出来的参考电压（阈值）**。调电位器就是**改阈值**，也就是改「多暗才算暗」。
- **为什么非调不可**：不同环境（白天/夜晚、室内/室外、不同光源）下光敏电阻的分压值差别很大。阈值固定时，太灵敏会「有点风吹草动就响」，太迟钝会「遮死了也不响」。所以模块必须留一个可调电位器来适配现场光强。

> [!tip] 被遮挡才响，还是被照亮才响？
> 本实验 `LightSensor_Get() == 1` 代表**光线暗**，于是 `Buzzer_ON()`，即**遮住光敏电阻蜂鸣器响**。如果你想让逻辑反过来（照亮才响），把 `main()` 里的判断条件写成 `== 0` 即可，硬件不用动。

### 5.4 LightSensor.c / LightSensor.h

```c
#include "stm32f10x.h"                  // Device header

/**
  * 函    数：光敏传感器初始化
  * 参    数：无
  * 返 回 值：无
  */
void LightSensor_Init(void)
{
	/*开启时钟*/
	RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOB, ENABLE);		//开启GPIOB的时钟

	/*GPIO初始化*/
	GPIO_InitTypeDef GPIO_InitStructure;
	GPIO_InitStructure.GPIO_Mode = GPIO_Mode_IPU;				//上拉输入：模块没插/没上电时，引脚也是确定的高电平，不会乱触发
	GPIO_InitStructure.GPIO_Pin = GPIO_Pin_13;					//DO接在PB13
	GPIO_InitStructure.GPIO_Speed = GPIO_Speed_50MHz;			//输入模式下速度无意义
	GPIO_Init(GPIOB, &GPIO_InitStructure);						//将PB13引脚初始化为上拉输入
}

/**
  * 函    数：获取当前光敏传感器输出的高低电平
  * 参    数：无
  * 返 回 值：光敏传感器输出的高低电平，范围：0/1
  */
uint8_t LightSensor_Get(void)
{
	return GPIO_ReadInputDataBit(GPIOB, GPIO_Pin_13);			//返回PB13输入寄存器的状态，1=光线暗，0=光线亮
}
```

```c
#ifndef __LIGHT_SENSOR_H
#define __LIGHT_SENSOR_H

void LightSensor_Init(void);
uint8_t LightSensor_Get(void);

#endif
```

> [!note] 上拉输入还是浮空输入
> 模块的 DO 是**推挽式的比较器输出**，能主动输出高低电平，所以引脚本身不会悬空。课程代码用**上拉输入**，并补充说：**如果模块始终都插在端口上，用浮空输入也可以，只要保证引脚不悬空就行**。用上拉输入更保险——模块没接或没上电时，引脚默认是高电平，不会因悬空乱跳。

### 5.5 main.c（实验三）

```c
#include "stm32f10x.h"                  // Device header
#include "Delay.h"
#include "Buzzer.h"
#include "LightSensor.h"

int main(void)
{
	/*模块初始化*/
	Buzzer_Init();			//蜂鸣器初始化（推挽输出，PB12，低电平响）
	LightSensor_Init();		//光敏传感器初始化（上拉输入，PB13）

	while (1)
	{
		if (LightSensor_Get() == 1)		//光敏输出1：光线比较暗（光敏电阻阻值变大，被上拉为高电平）
		{
			Buzzer_ON();				//蜂鸣器开启
		}
		else							//否则，光线较亮
		{
			Buzzer_OFF();				//蜂鸣器关闭
		}
	}
}
```

现象：用东西遮住光敏电阻，蜂鸣器响；移开遮挡，蜂鸣器停。遮挡的难易程度由模块上的电位器决定——**先调电位器，让当前环境光下不响，再用手遮住时才响**，这就是这个实验唯一的「校准」步骤。

## 6 易错点

- [ ] 按键引脚忘了配**上拉输入**（配成浮空/模拟）→ 松手时电平不定，表现为「没按也触发」或「按了没反应」。
- [ ] `Hardware` 文件夹没加入 Keil 的**头文件搜索路径** → 报 `cannot open source input file "Key.h"`。
- [ ] `.h` 忘了写 `#ifndef / #define / #endif` 防重复包含 → 多处包含时报重定义。
- [ ] 只做状态翻转不做消抖 → **按一下 LED 切换好几次**（抖动被读成多次按下）。
- [ ] `LEDx_Turn()` 里错用 `GPIO_ReadInputDataBit()` 去判断自己输出的状态 → 读的是引脚而非 ODR，开漏等场景下会判断错误。
- [ ] 标志位法把 `Key_Flag = 0` 写在**按下**分支 → 等于没写标志位，变成「按住狂闪」；必须在**松开**分支清 0。
- [ ] 把 `Key_GetNum()`（内含 `while` 等松手）放进中断服务函数 → 中断里死等，系统卡死。
- [ ] 按键/传感器/蜂鸣器的引脚号与实物接线不一致 → 功能完全不对，且很难从代码上看出来。
- [ ] 光敏模块用 **AO** 接普通 GPIO 去读 → AO 是模拟量，普通输入引脚读不出光照强度，只能读 DO（或等学过 ADC 后接 ADC 引脚）。
- [ ] 光敏模块的 VCC/GND 忘记接或接反 → DO 输出无意义（此时上拉输入至少能让引脚保持确定电平，不会乱响）。
- [ ] 光敏实验没调电位器就下结论「模块坏了」→ 阈值不合适时，遮挡与不遮挡可能都输出同一电平。

## 7 自测

1. 按键一端接 GPIO、另一端接 GND，为什么引脚必须配成上拉输入？如果配成浮空输入，现象会是什么？
2. `Key_GetNum()` 里两处 `Delay_ms(20)` 和中间那句 `while (GPIO_ReadInputDataBit(...) == 0);` 各起什么作用？为什么说它是阻塞函数？
3. `LED1_Turn()` 为什么用 `GPIO_ReadOutputDataBit()` 而不是 `GPIO_ReadInputDataBit()`？
4. 用标志位实现「按一下切换一次」时，标志位为什么必须在**按键松开**的分支里清零？写在按下分支会怎样？
5. 光敏模块的 DO 和 AO 有什么区别？遮挡光敏电阻时 DO 输出高电平还是低电平，为什么？电位器起什么作用？

> [!success]- 参考答案
> 1. 因为松手时开关断开，浮空输入会让引脚悬空、电平不确定且易受干扰。配成上拉输入后，松手时由内部弱上拉（约 40 kΩ）保持高电平，按下时被短到 GND 读到低电平。浮空输入的现象是「没按按键也乱触发」或读数随机跳变。
> 2. 第一个 `Delay_ms(20)` 是**延时消抖**，跳过按下瞬间 5~10 ms 的抖动区；`while(... == 0)` 是**二次确认并等待松手**，既确认电平确实稳定为低，又保证一次按下只返回一次非 0 键码；第二个 `Delay_ms(20)` 是躲过**松手瞬间**的抖动。因为 `while` 会在按住不放时一直死等，主循环停在这里，所以是阻塞函数。
> 3. 因为它要判断的是「**自己刚才把 PA1 输出成了什么**」，这个信息记录在**输出数据寄存器 ODR** 里，`GPIO_ReadOutputDataBit()` 读的正是 ODR；`GPIO_ReadInputDataBit()` 读的是引脚的输入电平，与「自己刚输出了什么」不是一回事。
> 4. 标志位记录的是「这次按下是否已经处理过」。若在**按下**分支清零，标志位立刻恢复为 0，下一圈主循环会再次判定为新按下并翻转一次——按住不放就变成连续翻转（狂闪）。只有等**松开**后才清 0，才能保证「一次按下 = 一次翻转」。
> 5. **DO 是比较器（LM393）二值化后的数字输出（0/1），AO 是分压点得到的连续模拟电压**；DO 接普通 GPIO 输入即可，AO 必须接 ADC 引脚。遮挡时**光线变暗 → 光敏电阻阻值变大 → 下拉变弱 → 分压点电压升高 → DO 输出高电平（1）**，所以现象是遮住时蜂鸣器响。电位器提供比较器的参考电压，也就是**调节翻转阈值**，用来适配不同环境的光照强度。

## 8 待核对

- [ ] 光敏模块的**具体型号与丝印**、电位器的**阻值**，以及 LM393 这一型号是否确实为课程套件所用（官方 `模块资料.zip\4-光敏电阻传感器\` 内有 `光敏电阻传感器模块电路图.pdf`、`光敏电阻传感器模块使用说明书3线制.pdf`，本次核对**未能提取这两个文件的内容**）。
- [ ] 光敏模块的滤波电容、定值电阻 R1 的具体参数（来源只讲到「有滤波电容」「有定值电阻分压」，未给数值）。
- [ ] 模块上输出指示灯的接法（本页按「DO 为低时指示灯亮」解释「遮挡时指示灯灭」这一现象，未核对模块原理图）。
- [ ] 视频 3-4 中实验二「按一下切换一次」的**讲解原话**；本页 `Key_GetNum()` 等松手 + `LEDx_Turn()` 读 ODR 的代码已与官方源码逐行核对，但视频讲解未逐帧核对；4.4 的**标志位法**为等价扩展写法（官方 3-4 源码中无此写法）。
- [ ] 视频中该模块化重构的讲解原话（`LED.c/h`、`Key.c/h` 收在 `3-4` 工程、`Buzzer.c/h`、`LightSensor.c/h` 收在 `3-5` 工程，这两个工程都没有 `Hardware\` 之外的驱动写法，事实已由官方源码确认）。
- [ ] 蜂鸣器模块内部的三极管型号与电路参数（`模块资料.zip\3-有源蜂鸣器\有源蜂鸣器模块原理图.png` 未能提取）。

## 核对记录（2026-09-26）

> [!success] 核对依据
> - 官方源码：`STM32Project-有注释版\3-4 按键控制LED\Hardware\LED.c`、`LED.h`、`Key.c`、`Key.h`、`User\main.c`；`3-5 光敏传感器控制蜂鸣器\Hardware\LightSensor.c`、`LightSensor.h`、`Buzzer.c`、`Buzzer.h`、`User\main.c`；`3-3 蜂鸣器\User\main.c`；`1-3 Delay函数模块\Delay.c`
> - 官方接线图：`接线图\3-4 按键控制LED.png`、`3-5 光敏传感器控制蜂鸣器.png`、`3-2 LED流水灯.png`
> - 课件文本：`课件文本.md`（Slide 26 LED 和蜂鸣器简介、Slide 29 按键简介、Slide 30 传感器模块简介）
> - 官方模块资料：`模块资料.zip\3-有源蜂鸣器\`（目录名）、`模块资料.zip\4-光敏电阻传感器\`（目录结构）

| 核对项 | 笔记原值 | 官方依据 | 结论 |
| --- | --- | --- | --- |
| 按键引脚 | PB1 / PB11 | 官方 `3-4\Hardware\Key.c` 第 12、17 行：`RCC_APB2Periph_GPIOB` + `GPIO_Pin_1 \| GPIO_Pin_11` | 一致 |
| 按键输入模式 | 上拉输入 | 官方 `Key.c` 第 16 行：`GPIO_Mode_IPU` | 一致 |
| 按键另一端接 GND | 一端接 GPIO、另一端接 GND | 接线图 `3-4 按键控制LED.png` 中两只按键一端接 GND 轨，另一端经导线接 PA/PB 排针侧 | 一致 |
| LED 引脚 | PA1 / PA2 | 官方 `3-4\Hardware\LED.c` 第 11、16 行：`RCC_APB2Periph_GPIOA` + `GPIO_Pin_1 \| GPIO_Pin_2` | 一致 |
| LED 驱动方式 | 推挽输出、低电平点亮 | 官方 `LED.c` 第 15 行 `GPIO_Mode_Out_PP`；`LED1_ON()` 用 `GPIO_ResetBits`（注释「设置 PA1 引脚为低电平」）；初始化后 `GPIO_SetBits` 拉高两灯（第 21 行） | 一致 |
| LED 阳极接 VCC、串限流电阻 | 见第 1 节接线块 | 接线图 `3-4 按键控制LED.png`：两只 LED 与电阻在面包板上，LED 阳极接正电源轨，阴极经电阻接排针（与 3-2 布置相同） | 一致 |
| 蜂鸣器引脚 | PB12 | 官方 `3-3 蜂鸣器\User\main.c`（`GPIOB` + `GPIO_Pin_12`）与 `3-5\Hardware\Buzzer.c` 第 11、16 行 | 一致 |
| 蜂鸣器低电平响 | 低电平响、高电平停 | 官方 `Buzzer.c`：`Buzzer_ON()` = `GPIO_ResetBits(GPIOB, GPIO_Pin_12)`，`Buzzer_OFF()` = `GPIO_SetBits` | 一致 |
| 蜂鸣器有源 / 无源 | 低电平响（有源蜂鸣器） | 官方 `模块资料.zip` 中模块目录名为 **`3-有源蜂鸣器`**；接线图 `3-3 蜂鸣器.png` 中为 VCC/GND/I-O 三脚模块 | 一致 |
| 光敏 DO 引脚 | PB13 | 官方 `3-5\Hardware\LightSensor.c` 第 11、16 行：`RCC_APB2Periph_GPIOB` + `GPIO_Pin_13`；`LightSensor_Get()` 读 `GPIOB, GPIO_Pin_13` | 一致 |
| 光敏引脚模式 | 上拉输入 | 官方 `LightSensor.c` 第 15 行：`GPIO_Mode_IPU` | 一致 |
| 光敏只接 DO、不接 AO | 见 5.2 节表格 | 接线图 `3-5 光敏传感器控制蜂鸣器.png` 中模块的 **AO 脚悬空**，只接 DO（绿）/ GND（蓝）/ VCC（红） | 一致 |
| 光敏现象：遮光 → DO=1 → 蜂鸣器响 | 光线暗 → DO=1 | 官方 `3-5\User\main.c` 第 14~21 行：`if (LightSensor_Get() == 1) Buzzer_ON();`，即读到 1 就响 | 一致 |
| `Key_GetNum()` 消抖写法与键码 | 两次 `Delay_ms(20)`，中间 `while(...==0);`，返回 1/2/0 | 官方 `Key.c` 第 30~48 行逐行一致，注释「延时消抖」/「等待按键松手」也一致 | 一致 |
| `Key_GetNum()` 是阻塞函数 | 按住不放会卡住 | 官方 `Key.c` 第 26 行函数注释**原文**：「此函数是阻塞式操作，当按键按住不放时，函数会卡住，直到按键松手」 | 一致 |
| `LED1_Turn()` 读 ODR 取反 | `GPIO_ReadOutputDataBit` 读输出寄存器 | 官方 `LED.c` 第 51 行 `if (GPIO_ReadOutputDataBit(GPIOA, GPIO_Pin_1) == 0)` | 一致 |
| `LED_Init()` 默认拉高（两灯初始熄灭） | 默认高电平，两个 LED 都熄灭 | 官方 `LED.c` 第 21 行 `GPIO_SetBits(GPIOA, GPIO_Pin_1 \| GPIO_Pin_2)` | 一致 |
| `main.c` 实验一/二结构 | `Key_GetNum()` 取键码 → `LED1_Turn()` / `LED2_Turn()` | 官方 `3-4\User\main.c` 第 11~27 行完全一致 | 一致 |
| `main.c` 实验三结构 | `LightSensor_Get() == 1` → `Buzzer_ON()`，否则 `Buzzer_OFF()` | 官方 `3-5\User\main.c` 第 9~22 行完全一致 | 一致 |
| `LED.c/h`、`Buzzer.c/h` 属本集新出现的驱动文件 | 见 2 节 callout | 官方 `3-4 按键控制LED\Hardware\` 下只有 `LED.c/h`、`Key.c/h`；`Buzzer.c/h`、`LightSensor.c/h` 在 `3-5\Hardware\` 下 | 一致（3-2 的 LED/蜂鸣器代码确实写在 `main.c` 内，`3-2 LED流水灯\`、`3-3 蜂鸣器\` 均无 `Hardware\` 目录） |
| 3-2 与 3-4 的 LED 引脚不同不是矛盾 | 「不是前后矛盾」 | `3-2 LED流水灯` 用 PA0~PA7（8 路），`3-4\Hardware\LED.c` 用 PA1/PA2（2 路），官方源码同时成立 | 一致（该说法本身正确） |
| 引脚号来源 | 原为「来自课程配套笔记（阿齐Archie 系列）的第三方复述与第三方复现代码，未逐帧核对视频接线图」 | 现全部改为官方源码 + 官方接线图 | 已修正（原为第三方笔记/复现代码） |
| 光敏模块电路细节（LM393、定值电阻、滤波电容、电位器） | 见 5.1 节电路图 | 课件文本 Slide 30 只有「分压 → 电压比较器 → 二值化」的文字简介，**未见 LM393 型号与元件参数**；官方源码中亦无 | 一致（「分压+比较器二值化」为课件原文，具体元件型号官方资料中未见，已加说明） |
| 光敏模块电源接线 | VCC / GND 供电 | 接线图 `3-5 光敏传感器控制蜂鸣器.png`：模块 VCC→红、GND→蓝、DO→绿 | 一致 |
| 4.4 标志位法 | 非阻塞等价实现 | 官方 3-4 源码中**无标志位写法**（只有 `Key_GetNum()` 等松手版），本页已标为等价扩展写法 | 一致（明确标注为扩展写法，非官方代码） |

> [!note] 出处说明
> 本页三个实验的接线、`LED.c/h`、`Key.c/h`、`Buzzer.c/h`、`LightSensor.c/h` 与两份 `main.c` 代码，已**逐行对照课程官方配套源码**（`STM32Project-有注释版\3-4 按键控制LED\`、`3-5 光敏传感器控制蜂鸣器\`、`3-3 蜂鸣器\`、`1-3 Delay函数模块\`），引脚与接线对照 **官方接线图 `接线图\3-4 按键控制LED.png`、`3-5 光敏传感器控制蜂鸣器.png`、`3-2 LED流水灯.png`**，原理表述对照 **课程课件 `课件文本.md`（Slide 26/29/30）**，模块型号对照 **官方 `模块资料.zip`**。**仍未核实**：光敏模块的 LM393 型号与电位器阻值、滤波电容与定值电阻 R1 的参数、模块输出指示灯的接法（对应官方 `模块资料.zip\4-光敏电阻传感器\` 中的说明书与电路图，本次未能提取其内容）、蜂鸣器模块内部三极管参数，以及视频中的讲解原话。

