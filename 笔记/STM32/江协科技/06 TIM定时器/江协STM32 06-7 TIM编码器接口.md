---
course: 江协科技 STM32入门教程-2023版
chapter: 06-7 TIM编码器接口
type: 分集笔记
source: https://www.bilibili.com/video/BV1th411z7sn?p=19
tags:
  - STM32
  - 江协科技
  - TIM
  - 编码器接口
  - 正交编码器
status: draft
verify: 官方源码+接线图+课件
updated: 2026-09-26
---

# 06-7 TIM 编码器接口

> [!abstract] 这一集只解决一个问题
> **怎么让硬件自己去数编码器转了多少格、朝哪个方向转？**
>
> 答：把编码器的 A、B 两相接到定时器的 **CH1 / CH2** 引脚，打开**编码器接口**（Encoder Interface）。此后 CNT 不再听内部时钟的，而是**每来一个边沿就走一格，往哪边走由另一相此刻的电平决定**——位置、方向、速度三个量一次全拿到。
>
> 上一集 [[江协STM32 06-6 输入捕获测频率与占空比]] ｜ 章索引 [[江协STM32 06 TIM定时器（章索引）]] ｜ 下一集 [[江协STM32 06-8 编码器接口测速]]

## 1 编码器接口是什么

一句话：**编码器接口 = 一个自带方向判断的外部计数时钟**。

它接收**增量编码器**（incremental encoder，也就是正交编码器 quadrature encoder）输出的 A、B 两相正交方波，根据两相谁先谁后，自动控制 CNT 自增或自减，从而指示：

| 想要的量 | 从哪个信号拿到 |
| --- | --- |
| 位置（position） | CNT 的**累计值** |
| 方向（direction） | CNT 变化量的**正负号** |
| 速度（speed） | 单位时间内 CNT 的**变化量** |

结构上有两个要点：

1. **每个通用定时器和高级定时器都拥有 1 个编码器接口**；基本定时器（TIM6 / TIM7）没有。
2. **两个输入引脚借用了输入捕获的通道 1 和通道 2**。也就是说，你在 6-5 / 6-6 里用的 `TIMx_CH1`、`TIMx_CH2` 那两个引脚，就是编码器接口的输入口。

以本课程用的 STM32F103 为例，编码器接在 TIM3 上：

```text
编码器 A 相 ──► PA6 ──► TIM3_CH1 ─┐
                                  ├─► 编码器接口 ──► CNT（自增/自减）
编码器 B 相 ──► PA7 ──► TIM3_CH2 ─┘
```

> [!note] TIM3_CH1 / TIM3_CH2 默认在 PA6 / PA7
> 这是 STM32F103 的**默认**引脚定义，本课程直接用默认引脚即可。
>
> 补充一点准确性：TIM3 其实还可以**重映射**——部分重映射到 `PB4 / PB5`，完全重映射到 `PC6 / PC7`；要用就得打开 AFIO 时钟并调 `GPIO_PinRemapConfig()`。本课程不涉及，知道有这回事、别把「默认引脚」当成「唯一引脚」就行。
>
> 另外 A、B 两相可以互换接线，只影响转向的正负号（见 4.2 节，也可以在软件里靠极性解决）。

### 1.1 为什么要做成硬件

编码器测速是**频繁执行、逻辑又极简单**的任务：来了边沿、看一眼另一相、加减一。这种活儿交给软件（外部中断里手动计次）会一直占着 CPU；做成硬件模块就完全不需要 CPU 参与，只有你想读结果时才去读一次 CNT。

典型场景是**电机闭环控制**：PWM 驱动电机 → 编码器测速 → PID 算控制量 → 反过来调 PWM。这条环路上速度采样频率很高，用硬件自动计数几乎是必须的。

### 1.2 定时器内部是怎么接的

把编码器接口拆成「输入」和「输出」两半看最清楚：

| 部位 | 借用了什么 | 说明 |
| --- | --- | --- |
| **输入部分** | 输入滤波器和边沿检测器 | 6-5 里配的**滤波器**继续有效，可以滤掉信号抖动 |
| **输入部分（用不到）** | 输入预分频器、CCR 寄存器、直连/交叉选择 | 这些是给输入捕获用的，编码器模式下与它们无关 |
| **输出部分** | 相当于从模式控制器 | 去控制 CNT 的**计数时钟**和**计数方向** |

用不到的那几项即使配了也不起作用——但为了参数完整，代码里还是走一遍 `TIM_ICInit`。

## 2 正交编码器原理

正交编码器输出两路方波 A 和 B，**相位差 90°**（四分之一周期）。谁超前谁，就代表往哪个方向转：

```text
正转：A 超前 B 90°                     反转：B 超前 A 90°
A(TI1)  ‾‾‾‾____‾‾‾‾____          A(TI1)  ‾‾‾‾____‾‾‾‾____
B(TI2)  __‾‾‾‾____‾‾‾‾__          B(TI2)  ‾‾____‾‾‾‾____‾‾
```

判断方法非常朴素：**出现一个边沿时，去看另一相此刻是高还是低。**

- 一个周期里 A、B 一共产生 **4 个边沿**（A 的上升/下降 + B 的上升/下降），所以一圈下来能数到 4 个数。
- 单相输出（只有 A）只能测速度和位置，**测不出方向**；必须两相都有才能判方向——这也正是"正交"的意义。
- 编码器的**线数**（每转脉冲数，PPR，Pulses Per Revolution）指的是**单相**每转输出的脉冲个数，不是四相计数后的个数。

## 3 三种模式与倍频

库函数提供了三个模式宏，区别就是「数哪些边沿」：

| 模式宏 | 数哪些边沿 | 每转计数 | 俗称 |
| --- | --- | --- | --- |
| `TIM_EncoderMode_TI1` | 只数 TI1（A 相）的上升沿 + 下降沿 | `2 × PPR` | **2 倍频** |
| `TIM_EncoderMode_TI2` | 只数 TI2（B 相）的上升沿 + 下降沿 | `2 × PPR` | **2 倍频** |
| `TIM_EncoderMode_TI12` | TI1、TI2 的 4 个边沿全数 | `4 × PPR` | **4 倍频** |

```text
TI1  模式：  只看 A 相的两次跳变        → 每周期 +2
TI2  模式：  只看 B 相的两次跳变        → 每周期 +2
TI12 模式：  A、B 四个跳变全看          → 每周期 +4   ← 分辨率最高，本课程用这个
```

> [!note] 「倍频」这个词是标准库/参考手册口径，课件里没有
> 课件 Slide 83《工作模式》与 Slide 84/85《实例（均不反相）》《实例（TI1 反相）》**只给了模式名与实例波形，没有出现「倍频」「线数」「PPR」这些术语**（已 grep 全 207 页课件文本确认）。「2 倍频 / 4 倍频」的说法来自标准库对 `TIM_EncoderMode_*` 的说明——`stm32f10x_tim.c` 第 1250~1252 行：`TIM_EncoderMode_TI1: Counter counts on TI1FP1 edge depending on TI2FP2 level.`、`TI2: … on TI2FP2 edge depending on TI1FP1 level.`、`TI12: … on both TI1FP1 and TI2FP2 edges…`。本表是据此整理的**补充**，不是课件原文。

> [!warning] 为什么没有「一倍频」
> 编码器模式下边沿检测器**上升沿和下降沿都有效**（这是它的设计前提，见下一节），所以只要是它管的通道，一次跳变就一定会被数到，最少也是 2 倍频。想要"只数 A 相上升沿"的 1 倍频，库里没有对应模式，得自己用外部中断或输入捕获实现。
>
> 倍频数越高，**分辨率越高**，但同样转速下 CNT 涨得也越快，更容易溢出——6-8 算转速时这个数会出现在分母上。

不管用哪个模式，**方向永远由另一相的电平决定**，这也是 2 倍频模式仍然能识别正反转的原因。

## 4 极性：在编码器模式下含义变了

这是本集最容易懵的一点，单独拎出来说。

在 6-5 输入捕获里，`TIM_ICPolarity_Rising` / `TIM_ICPolarity_Falling` 的意思是"**捕获哪个边沿**"。

在编码器模式里，**上升沿和下降沿都有效**，于是这个参数换了个含义：

| 参数 | 输入捕获里的含义 | 编码器模式下的含义 |
| --- | --- | --- |
| `TIM_ICPolarity_Rising` | 只捕获上升沿 | 信号**直通**，高低电平**不反相** |
| `TIM_ICPolarity_Falling` | 只捕获下降沿 | 信号**经过一个非门**，高低电平**反相** |

也就是说，这两个宏现在选的是**要不要插一个非门**，而不是选边沿。名字骗人，但看一眼真值表就通了。

### 4.1 A/B 状态与 CNT 增减的真值表

下面这张表是编码器接口的核心判决逻辑。**判据是「触发计数的那个边沿」+「另一相此刻的电平」**：

| 触发计数的边沿 | 另一相的电平 | 判定 | CNT 动作 |
| --- | --- | --- | --- |
| A（TI1）上升沿 | B（TI2）= 低 | 正转 | **+1** |
| A（TI1）下降沿 | B（TI2）= 低 | 反转 | **−1** |
| A（TI1）上升沿 | B（TI2）= 高 | 反转 | **−1** |
| A（TI1）下降沿 | B（TI2）= 高 | 正转 | **+1** |
| B（TI2）上升沿 | A（TI1）= 高 | 正转 | **+1** |
| B（TI2）下降沿 | A（TI1）= 高 | 反转 | **−1** |
| B（TI2）上升沿 | A（TI1）= 低 | 反转 | **−1** |
| B（TI2）下降沿 | A（TI1）= 低 | 正转 | **+1** |

用第 2 节的两张波形图代入验算：正转时，A 的上升沿、A 的下降沿、B 的上升沿、B 的下降沿**四处都判为 +1**，一个周期净增 4；反转时四处都判为 −1，一个周期净减 4。表是自洽的。

> [!note] 课件上这张表是**两张**分开的
> 课件 Slide 81《正交编码器》把它拆成「正转」「反转」两张四行表，**行标题只有「边沿」「另一相状态」两列，没有「判定」与「CNT 动作」列**：
>
> - 正转：`A 相↑ / B 相低电平`、`A 相↓ / B 相高电平`、`B 相↑ / A 相高电平`、`B 相↓ / A 相低电平`
> - 反转：`A 相↑ / B 相高电平`、`A 相↓ / B 相低电平`、`B 相↑ / A 相低电平`、`B 相↓ / A 相高电平`
>
> 本页把它们合并成一张八行表，并补上「判定 / CNT 动作」两列（+1 / −1）。**这四组对应关系与课件逐字一致**，补出的两列是按波形自洽验算的结果。

- `TIM_EncoderMode_TI1` 模式只用到表格上半部分（A 的两种边沿）；
- `TIM_EncoderMode_TI2` 模式只用下半部分；
- `TIM_EncoderMode_TI12` 模式上下都生效。

### 4.2 反相的效果

因为反相会把「边沿方向」和「另一相电平」同时颠倒，所以结论很干净：

| 配置 | 正转时 CNT | 反转时 CNT |
| --- | --- | --- |
| 均不反相（`Rising, Rising`） | 自增 | 自减 |
| TI1 反相（`Falling, Rising`） | 自减 | 自增 |
| TI2 反相（`Rising, Falling`） | 自减 | 自增 |
| 两相都反相（`Falling, Falling`） | 自增 | 自减 |

两条推论：

1. **反相任意一相 = 把正转和反转的判定整体互换**，也就是 CNT 的正负号反了。
2. **两相都反相 = 没有反相**。所以"接线时 A/B 接反了"这件事，既可以用交换接线解决，也可以用"只反相一相"或"两相都反相"在软件里解决——这是个很方便的补救手段。

### 4.3 为什么这种设计能抗噪声

因为**判决依赖另一相的电平，而毛刺只出现在一相上**。设 B 一直为低，A 上有一个毛刺：

```text
理想情况：A 干净地上升一次，B = 低
          → A 上升沿(+1)                      净 +1   ✔

有毛刺时：A 在 0/1 之间来回抖 6 次，B 仍为低
          → A 上升+1、下降-1、上升+1、
            下降-1、上升+1、下降-1            净  0   ✔
```

同一个方向上"上去"和"下来"的边沿**必然成对出现**，一对就是一个 `+1` 和一个 `−1`，互相抵消。所以毛刺带来的计数抖动是 `+ − + −` 来回摆，最终净计数不受影响。

> [!tip] 一句话记住
> **输入滤波器管「信号干不干净」，正交判决管「毛刺算不算数」。** 前者靠 `TIM_ICFilter` 硬件低通，后者是编码器接口的天然属性。

## 5 关键认知：CNT 完全交给编码器托管

这是本集最该记住的一条：

> **一旦进入编码器模式，时基单元里配的「内部时钟」和「计数方向」就都不起作用了。**

| 配置项 | 编码器模式下的状态 |
| --- | --- |
| 内部时钟（72 MHz） | **不起作用**。计数时钟变成了编码器的边沿信号 |
| `TIM_CounterMode_Up` / `_Down` | **不起作用**。计数方向由编码器接口决定 |
| `TIM_Prescaler`（PSC） | 仍然起作用，通常写 `1 - 1`（不分频） |
| `TIM_Period`（ARR） | 仍然起作用，决定 CNT 数到多少算满 |

换句话讲：编码器接口就是一个**带有方向选择的外部时钟**，它把时基单元的输入整个接管了。

> [!warning] 所以 `TIM_InternalClockConfig()` 也不用调
> 6-1 / 6-2 里那句"选择时基单元时钟源"在编码器模式下是多余的——时钟源已经被硬件改接到编码器上了。代码里不写它，行为不变。

## 6 标准库配置六步

```text
① RCC 开启 TIM 时钟和 GPIO 时钟
② GPIO 把 CH1 / CH2 对应的两个引脚配成输入模式
③ 配置时基单元（PSC 不分频，ARR 给最大 65535）
④ TIM_ICInit 配置 CH1 / CH2 的滤波器和极性
⑤ TIM_EncoderInterfaceConfig 配置编码器模式与两个通道是否反相
⑥ TIM_Cmd 启动计数器
```

### 6.1 第 ② 步：GPIO 模式怎么选

编码器的 A/B 是**推挽输出**，所以 MCU 这边必须是**输入**，绝不能配成输出（两边会"打架"）。三种输入模式怎么挑：

| 外部模块空闲时的电平 | 选哪种 | 宏 |
| --- | --- | --- |
| 默认输出**高**电平 | 上拉输入 | `GPIO_Mode_IPU` |
| 默认输出**低**电平 | 下拉输入 | `GPIO_Mode_IPD` |
| 不确定 / 输出功率很小 | 浮空输入 | `GPIO_Mode_IN_FLOATING` |

原则只有一句：**和外部模块的默认状态保持一致，防止默认电平打架。**

- 一般编码器空闲时输出高电平，所以**上拉输入用得最多**。
- 浮空输入的缺点：引脚悬空时没有默认电平，容易受噪声干扰来回跳变。

### 6.2 第 ④ ⑤ 步：顺序不能颠倒

```text
TIM_ICInit(通道1)  ┐
TIM_ICInit(通道2)  ├─► 必须先做
TIM_EncoderInterfaceConfig(...)  ─► 必须后做
```

原因：`TIM_EncoderInterfaceConfig()` 内部会**改写 CCER 里两个通道的极性位（CC1P / CC2P）**。查 `stm32f10x_tim.c` 第 1294~1296 行，它清掉 `TIM_CCER_CC1P | TIM_CCER_CC2P` 后写入传入的两个极性；而 `TIM_ICInit` 也会写对应通道的极性位。所以顺序必须是「先两个 `TIM_ICInit`，后 `TIM_EncoderInterfaceConfig`」，否则**后调的输入捕获会把极性覆盖掉，反相参数就白设了**。
>
> 官方 `Encoder.c` 的注释原文：「此函数必须在输入捕获初始化之后进行，否则输入捕获的配置会覆盖此函数的部分配置」。注意被覆盖的**只是极性位**，滤波器（`TIM_ICFilter`）由 `TIM_ICInit` 写进 CCMR 的 ICF 位，`TIM_EncoderInterfaceConfig` 不碰它。

## 7 完整代码

### 7.1 Encoder.c（官方 `6-8 编码器接口测速\Hardware\Encoder.c` 原文）

```c
#include "stm32f10x.h"

/**
  * 函    数：编码器初始化
  * 参    数：无
  * 返 回 值：无
  */
void Encoder_Init(void)
{
	/*开启时钟*/
	RCC_APB1PeriphClockCmd(RCC_APB1Periph_TIM3, ENABLE);			//开启TIM3的时钟
	RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOA, ENABLE);			//开启GPIOA的时钟
	
	/*GPIO初始化*/
	GPIO_InitTypeDef GPIO_InitStructure;
	GPIO_InitStructure.GPIO_Mode = GPIO_Mode_IPU;
	GPIO_InitStructure.GPIO_Pin = GPIO_Pin_6 | GPIO_Pin_7;
	GPIO_InitStructure.GPIO_Speed = GPIO_Speed_50MHz;
	GPIO_Init(GPIOA, &GPIO_InitStructure);							//将PA6和PA7引脚初始化为上拉输入
	
	/*时基单元初始化*/
	TIM_TimeBaseInitTypeDef TIM_TimeBaseInitStructure;				//定义结构体变量
	TIM_TimeBaseInitStructure.TIM_ClockDivision = TIM_CKD_DIV1;     //时钟分频，选择不分频，此参数用于配置滤波器时钟，不影响时基单元功能
	TIM_TimeBaseInitStructure.TIM_CounterMode = TIM_CounterMode_Up; //计数器模式，选择向上计数
	TIM_TimeBaseInitStructure.TIM_Period = 65536 - 1;               //计数周期，即ARR的值
	TIM_TimeBaseInitStructure.TIM_Prescaler = 1 - 1;                //预分频器，即PSC的值
	TIM_TimeBaseInitStructure.TIM_RepetitionCounter = 0;            //重复计数器，高级定时器才会用到
	TIM_TimeBaseInit(TIM3, &TIM_TimeBaseInitStructure);             //将结构体变量交给TIM_TimeBaseInit，配置TIM3的时基单元
	
	/*输入捕获初始化*/
	TIM_ICInitTypeDef TIM_ICInitStructure;							//定义结构体变量
	TIM_ICStructInit(&TIM_ICInitStructure);							//结构体初始化，若结构体没有完整赋值
																	//则最好执行此函数，给结构体所有成员都赋一个默认值
																	//避免结构体初值不确定的问题
	TIM_ICInitStructure.TIM_Channel = TIM_Channel_1;				//选择配置定时器通道1
	TIM_ICInitStructure.TIM_ICFilter = 0xF;							//输入滤波器参数，可以过滤信号抖动
	TIM_ICInit(TIM3, &TIM_ICInitStructure);							//将结构体变量交给TIM_ICInit，配置TIM3的输入捕获通道
	TIM_ICInitStructure.TIM_Channel = TIM_Channel_2;				//选择配置定时器通道2
	TIM_ICInitStructure.TIM_ICFilter = 0xF;							//输入滤波器参数，可以过滤信号抖动
	TIM_ICInit(TIM3, &TIM_ICInitStructure);							//将结构体变量交给TIM_ICInit，配置TIM3的输入捕获通道
	
	/*编码器接口配置*/
	TIM_EncoderInterfaceConfig(TIM3, TIM_EncoderMode_TI12, TIM_ICPolarity_Rising, TIM_ICPolarity_Rising);
																	//配置编码器模式以及两个输入通道是否反相
																	//注意此时参数的Rising和Falling已经不代表上升沿和下降沿了，而是代表是否反相
																	//此函数必须在输入捕获初始化之后进行，否则输入捕获的配置会覆盖此函数的部分配置
	
	/*TIM使能*/
	TIM_Cmd(TIM3, ENABLE);			//使能TIM3，定时器开始运行
}

/**
  * 函    数：获取编码器的增量值
  * 参    数：无
  * 返 回 值：自上此调用此函数后，编码器的增量值
  */
int16_t Encoder_Get(void)
{
	/*使用Temp变量作为中继，目的是返回CNT后将其清零*/
	int16_t Temp;
	Temp = TIM_GetCounter(TIM3);
	TIM_SetCounter(TIM3, 0);
	return Temp;
}
```

> [!note] 与上一版的差异（逐条）
> - `TIM_Period`：`65535` → **`65536 - 1`**（官方原文，与 6-6/6-7 的 `IC.c` 写法一致；两者等值）。
> - `Encoder_Get()` 的注释文字改为官方原文（上一版自行扩写了注释）。
> - `Encoder_Init()` 的步骤注释改为官方原文逐行对齐。

### 7.2 Encoder.h

```c
#ifndef __ENCODER_H
#define __ENCODER_H

void Encoder_Init(void);
int16_t Encoder_Get(void);

#endif
```

> [!note] `Encoder_Get()` 属于 6-8
> 6-7 是**理论集，官方没有独立工程**；`Encoder_Get()` 与 `Encoder_Init()` 一起出现在 **`6-8 编码器接口测速`** 的 `Hardware\Encoder.c` 里，且 `6-8` 是**唯一**的编码器工程。
>
> 本集讲的是**编码器接口本身**：转一格 CNT 走一格。要测**位置**，直接读 CNT 就行，不需要清零。
> `Encoder_Get()` 是**测速**时才需要的东西——因为测速要的是「固定时间内的增量」，读完必须清零。这里先把它一并列在文件里，6-8 直接复用。

### 7.3 main.c（官方 `6-8 编码器接口测速\User\main.c` 原文）

官方工程里这一集**没有独立的 6-7 工程**（6-7 是理论集），编码器接口的代码落在 `6-8 编码器接口测速` 里，而且**直接就是测速的写法**：用 TIM2 定时中断每 1 秒取一次 `Encoder_Get()`。如果想只测位置、不清零，把那一行换成直接读 CNT 即可（见下方第二段代码）。

```c
#include "stm32f10x.h"                  // Device header
#include "Delay.h"
#include "OLED.h"
#include "Timer.h"
#include "Encoder.h"

int16_t Speed;			//定义速度变量

int main(void)
{
	/*模块初始化*/
	OLED_Init();		//OLED初始化
	Timer_Init();		//定时器初始化
	Encoder_Init();		//编码器初始化
	
	/*显示静态字符串*/
	OLED_ShowString(1, 1, "Speed:");		//1行1列显示字符串Speed:
	
	while (1)
	{
		OLED_ShowSignedNum(1, 7, Speed, 5);	//不断刷新显示编码器测得的最新速度
	}
}
```

> [!note] 想只测位置（读 CNT、不清零）
> 官方代码里没有「测位置」的版本，下面这段是按 6-8 的思路改写的对照写法，**不是官方原文**：把中断里那一行换成直接读 CNT，并去掉 `Encoder_Get()` 的清零动作。
>
> ```c
> OLED_ShowSignedNum(1, 5, (int16_t)TIM_GetCounter(TIM3), 5);	//反复读CNT，绝不清零
> ```
>
> 现象：正转数值自增、反转数值自减；转动一格数值变化 **4**（TI12 = 4 倍频）。这个「4」是由模式决定的，课件没有给出「倍频」这一术语，属标准库/参考手册口径。

## 8 易错点

- [ ] **先调 `TIM_EncoderInterfaceConfig()`，后调 `TIM_ICInit()`** → 极性配置被输入捕获覆盖，反相参数白设。顺序必须是"先输入捕获，后编码器接口"。
- [ ] **把 `TIM_ICPolarity_Rising` 当成"只在上升沿计数"** → 实际含义是"高低电平不反相"。编码器模式下上升/下降沿都计数。
- [ ] **GPIO 配成了输出模式**（推挽/开漏）→ 引脚自己驱动电平，和编码器输出打架，读数乱跳。必须是**输入**模式。
- [ ] **忘记 `TIM_Cmd()`** → CNT 一动不动，读出来一直是 0。
- [ ] **PSC 忘了写 `1 - 1`** → 计数时钟被分频，转一格 CNT 不再走一格，测出来的位置和速度整体偏小。
- [ ] **返回类型写成 `uint16_t`** → 反转时看到的是 65535 附近的大数，而不是 `−1`。官方 `Encoder_Get()` 返回的是 `int16_t`，详见 6-8。
- [ ] **拿 TIM6 / TIM7（基本定时器）做编码器接口** → 没有这个硬件模块，配了也没反应。
- [ ] **拿 CH3 / CH4 去接编码器** → 编码器接口只用 CH1 和 CH2，CH3/CH4 接上去无效。
- [ ] **想在编码器模式下改计数方向**（改 `TIM_CounterMode_Up`）→ 方向由编码器托管，这个参数不起作用。要反转方向只能反相一相，或者把 A/B 接线对调。
- [ ] **接线时 A/B 接反了，以为是代码 bug** → 那只是正转变负转。交换 A/B 接线，或者把 `TIM_EncoderInterfaceConfig` 里任意一相改成 `Falling`，都能纠正。
- [ ] **在只有 TIM1~TIM4 的芯片（如 F103C8T6）上配 TIM5** → 不报错，但毫无反应，白忙半天。

## 9 自测

1. 编码器接口说是"借用了输入捕获的 CH1 和 CH2"，它借走的是输入捕获的哪些部分？哪些部分在编码器模式下用不到？
2. 一个 1000 线（PPR = 1000）的编码器，电机转一圈，用 `TIM_EncoderMode_TI1` 和 `TIM_EncoderMode_TI12` 分别让 CNT 变化多少？
3. 编码器模式下 `TIM_ICPolarity_Rising` 是什么含义？它和 6-5 输入捕获里的含义差别在哪？
4. 为什么编码器接口能抗噪声？一个毛刺为什么不会让最终计数值跑偏？
5. 时基单元里配的"内部时钟源"和 `TIM_CounterMode_Up`，在编码器模式下起了什么作用？
6. 为什么 `TIM_EncoderInterfaceConfig()` 一定要放在 `TIM_ICInit()` 后面？

> [!success]- 参考答案
> 1. 借走的是**输入滤波器和边沿检测器**（信号进来先过滤波、再检边沿），以及后面的编码器接口判决电路。用不到的是输入捕获的**预分频器、CCR 寄存器、直连/交叉（`TIM_ICSelection`）选择**——配了也不影响。
> 2. TI1 模式是 2 倍频：`2 × 1000 = 2000`；TI12 模式是 4 倍频：`4 × 1000 = 4000`。
> 3. 编码器模式下上升沿和下降沿**都有效**，所以 `Rising` 表示信号**直通、高低电平不反相**，`Falling` 表示**经非门、电平反相**。而在输入捕获里，Rising/Falling 表示"选上升沿还是下降沿作为捕获触发"。
> 4. 因为判决方向要"看另一相此刻的电平"，而毛刺只在一相上抖动，另一相在此期间不变。同方向的抖动必然带来**成对**的 `+1` 和 `−1`，互相抵消，净计数为 0。
> 5. **都不起作用**。进入编码器模式后，计数时钟被换成编码器的边沿信号（一个带方向选择的外部时钟），计数方向也由编码器接口决定。依然起作用的是 PSC（通常不分频）和 ARR（决定溢出点）。
> 6. 因为 `TIM_EncoderInterfaceConfig()` 内部也要写两个通道的极性位。若先调它再调 `TIM_ICInit()`，后者会把极性覆盖回默认值，反相设置就失效了。

## 10 待核对

- [ ] 视频里编码器接口框图（输入部分 / 输出部分）的逐块讲解顺序与画面标注（课件 Slide 82《编码器接口 基本结构》只有框图标号）。
- [ ] 视频里「正转 / 反转」两张判决表的**原始排版与行标题的视觉呈现**（课件 Slide 81 的行列文字已核对，但 PPT 上的箭头/图例符号无法从文本提取）。
- [ ] 视频是否单独讨论过「一倍频」及其实现方式（课件 Slide 83/84/85 只给模式名与实例波形，未见相关表述）。
- [ ] 视频里编码器模块 A/B/ VCC/GND 四根线的**插线动作细节**（官方接线图 `6-8 编码器接口测速.png` 上 A、B 两线从编码器模块走向 MCU 板块，缩略图不足以逐孔确认；引脚以官方 `Encoder.c` 的 `GPIO_Pin_6 | GPIO_Pin_7` 为准）。

## 核对记录（2026-09-26）

> [!success] 核对依据
> - **官方配套源码**（最高优先）：`C:\Users\陈杰裕\Desktop\资料\STM32入门教程资料\程序源码\程序源码\STM32Project-有注释版\`
>   - `6-8 编码器接口测速\Hardware\Encoder.c`、`Encoder.h`、`User\main.c`、`System\Timer.c`（**6-7 是理论集，官方无独立工程**，编码器代码只在 6-8 里）
>   - `6-7 PWMI模式测频率占空比\Library\stm32f10x_tim.c`（`TIM_EncoderInterfaceConfig()` 第 1264~1304 行）、`stm32f10x_tim.h`（枚举与函数原型）
> - **官方接线图**：`ground-truth\接线图\6-8 编码器接口测速.png`
> - **课件文本**：`ground-truth\课件文本.md`（Slide 80 编码器接口简介、Slide 81 正交编码器、Slide 82 编码器接口基本结构、Slide 83 工作模式、Slide 84/85 实例）
> - **引脚定义表**：`ground-truth\F103C8T6引脚定义_缩略.png`（PA6 = TIM3_CH1、PA7 = TIM3_CH2；PB4/PB5、PC6/PC7 为重定义）

| 核对项 | 笔记原值 | 官方依据 | 结论 |
| --- | --- | --- | --- |
| 「编码器接口 = 自带方向判断的外部计数时钟」 | 位置/方向/速度三量 | 课件 Slide 80 原文：「自动控制 CNT 自增或自减，从而指示编码器的位置、旋转方向和旋转速度」 | 一致 |
| 编码器类型 | 增量（正交）编码器 | 课件 Slide 80：「可接收增量（正交）编码器的信号」 | 一致 |
| 每个高级/通用定时器 1 个编码器接口 | 同 | 课件 Slide 80：「每个高级定时器和通用定时器都拥有 1 个编码器接口」 | 一致 |
| 借用 CH1/CH2 | 同 | 课件 Slide 80：「两个输入引脚借用了输入捕获的通道 1 和通道 2」 | 一致 |
| A/B 相引脚 | PA6 / PA7 → TIM3_CH1 / CH2 | `Encoder.c`：`GPIO_Pin_6 \| GPIO_Pin_7`；引脚定义表 PA6/PA7 = TIM3_CH1/CH2；接线图同 | 一致 |
| TIM3 重映射说法 | 「部分重映射 PB4/PB5、完全重映射 PC6/PC7」 | 引脚定义表 PA6/PA7 行的「重定义功能」分别标注 `TIM3_CH1`、`TIM3_CH2`（PB4/PB5、PC6/PC7 行标注为 TIM3_CH1~CH4 的重定义） | 一致 |
| `TIM_Period` | `65535` | `Encoder.c`：`TIM_TimeBaseInitStructure.TIM_Period = 65536 - 1;` | 已修正（原为 `65535`，现改为官方原文 `65536 - 1`） |
| `TIM_Prescaler` | `1 - 1` | `Encoder.c`：`TIM_Prescaler = 1 - 1` | 一致 |
| GPIO 模式 | PA6/PA7 上拉输入 `GPIO_Mode_IPU` | `Encoder.c`：`GPIO_Mode_IPU` + `GPIO_Pin_6 \| GPIO_Pin_7` | 一致 |
| `TIM_ICStructInit` + 两次 `TIM_ICInit` 的顺序 | 先 `TIM_ICInit`（CH1、CH2），后 `TIM_EncoderInterfaceConfig` | `Encoder.c`：`TIM_ICStructInit` → `TIM_ICInit(CH1, ICF=0xF)` → `TIM_ICInit(CH2, ICF=0xF)` → `TIM_EncoderInterfaceConfig(...)` | 一致 |
| `TIM_ICFilter` | `0xF`（两个通道） | `Encoder.c`：两次都写 `TIM_ICFilter = 0xF` | 一致 |
| 编码器模式 | `TIM_EncoderMode_TI12`（TI12，两个通道均不反相） | `Encoder.c`：`TIM_EncoderInterfaceConfig(TIM3, TIM_EncoderMode_TI12, TIM_ICPolarity_Rising, TIM_ICPolarity_Rising)` | 一致 |
| 「顺序不能颠倒」的原因 | 「`TIM_EncoderInterfaceConfig()` 内部也会写两个通道的极性位」 | `stm32f10x_tim.c` 第 1294~1296 行：清 `TIM_CCER_CC1P \| TIM_CCER_CC2P` 后写入两个极性；官方注释：「此函数必须在输入捕获初始化之后进行，否则输入捕获的配置会覆盖此函数的部分配置」 | 一致（已补注「被覆盖的只是极性位，滤波器不在此列」） |
| 极性在编码器模式下的含义 | `Rising` = 直通不反相；`Falling` = 经非门反相 | `Encoder.c` 注释：「注意此时参数的Rising和Falling已经不代表上升沿和下降沿了，而是代表是否反相」 | 一致 |
| 真值表（8 行） | 合并成一张 8 行表，含「判定 / CNT 动作」两列 | 课件 Slide 81 拆成正转/反转两张 4 行表，**只有「边沿」「另一相状态」两列**，8 组对应关系与本页逐条相同 | 一致（8 组对应关系原值正确；已补注课件只有两张 4 行表、本页的判定列是自洽验算补出的） |
| 反相的效果（4 种配置） | 均不反相=自增/自减；任一相反相=互换；两相都反相=等同不反相 | 课件 Slide 84《实例（均不反相）》、Slide 85《实例（TI1 反相）》两页标题与之一致；`TIM_ICPolarity_BothEdge` 之外的组合在库里都合法 | 一致 |
| 「2 倍频/4 倍频」「线数 PPR」 | 3 种模式对应 2/2/4 倍频 | 课件全文 **grep 不到「倍频」「线数」「PPR」**；标准库 `stm32f10x_tim.c` 第 1250~1252 行只说明「counts on TI1FP1 edge depending on TI2FP2 level」等 | 一致（原值按标准库口径正确；已补 callout 明确「倍频」不是课件术语，属补充） |
| `TIM_InternalClockConfig()` 不用调 | 「编码器模式下多余」 | `Encoder.c` 全程**没有**调用 `TIM_InternalClockConfig` | 一致 |
| main.c 写法 | 「直接读 CNT，用 `OLED_ShowString(1, 1, "CNT:")`」 | 6-8 `main.c`：`OLED_ShowString(1, 1, "Speed:")` + `OLED_ShowSignedNum(1, 7, Speed, 5)`，中断里 `Speed = Encoder_Get()` | 已修正（原为自造的「测位置 main.c」；现改为官方 `main.c` 原文，并把「只读 CNT 测位置」的写法降级为对照补充） |
| `Encoder_Get()` 出现在哪一集 | 「本页按 6-8 引入处理，但为文件完整先在 6-7 给出」 | 官方只有 `6-8 编码器接口测速\Hardware\Encoder.c` 含此函数；官方无 6-7 工程 | 一致（已由官方源码核实：6-8 引入；该条从待核对移除） |
| 库函数与枚举名 | `TIM_EncoderMode_TI1/TI2/TI12`、`TIM_ICPolarity_Rising/Falling`、`TIM_EncoderInterfaceConfig`、`TIM_ICInit`、`TIM_ICStructInit`、`TIM_GetCounter`、`TIM_SetCounter`、`GPIO_Mode_IPU/IPD/IN_FLOATING` | `stm32f10x_tim.h` 第 553~938、1060~1140 行逐一命中 | 一致 |
| 「拿 TIM6/TIM7 做编码器接口」易错点 | 基本定时器没有该模块 | 课件 Slide 215 定时器类型表：基本定时器 TIM6/TIM7 只有「定时中断、主模式触发 DAC」 | 一致 |

> [!note] 出处说明
> 本页的编码器接口功能描述、正交判决关系、工作模式、配置六步、`TIM_EncoderInterfaceConfig()` 参数含义、GPIO 输入模式选择原则与示例代码，已对照**课件文本**（Slide 80~85）、**官方配套源码**（`6-8 编码器接口测速\Hardware\Encoder.c`、`Encoder.h`、`User\main.c`，以及 `stm32f10x_tim.h` / `stm32f10x_tim.c`）与**官方接线图**（`6-8 编码器接口测速.png`）、**F103C8T6 引脚定义表**逐条核对；真值表的 8 组对应关系与课件 Slide 81 逐条一致，「判定 / CNT 动作」两列按波形自洽验算补出。
> **仍未核实的是**：编码器接口框图的逐块画面讲解顺序、两张判决表在 PPT 上的视觉排版与箭头图例、「一倍频」是否被讨论、编码器四根线插线动作的逐孔细节。以上条目见 `## 待核对`。
> 「倍频 / 线数 / PPR」不是课件术语，本页已明确标注为按标准库说明整理的补充口径。

