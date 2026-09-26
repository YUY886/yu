---
course: 江协科技 STM32入门教程-2023版
chapter: 06 TIM定时器
source: https://www.bilibili.com/video/BV1th411z7sn?p=13
tags:
  - STM32
  - TIM
  - 定时器
  - PWM
  - 输入捕获
  - 编码器
status: draft
---

# 06 TIM 定时器

> [!info] 覆盖视频
> [6-1] TIM定时中断、[6-2] 定时器定时中断&定时器外部时钟、[6-3] TIM输出比较、[6-4] PWM应用、[6-5] TIM输入捕获、[6-6] 输入捕获测频率/占空比、[6-7] TIM编码器接口、[6-8] 编码器接口测速。

## 1 定时器分类

| 类型 | 常见定时器 | 主要能力 |
| --- | --- | --- |
| 基本定时器 | TIM6、TIM7 | 计数、更新事件/中断，常用于定时和触发 DAC |
| 通用定时器 | TIM2~TIM5 | 定时中断、外部时钟、PWM、输入捕获、编码器接口 |
| 高级定时器 | TIM1、TIM8 | 通用定时器能力 + 互补 PWM、死区、刹车等电机控制功能 |

本课程主要使用 STM32F103C8T6，常见可用定时器是 TIM1、TIM2、TIM3、TIM4。

## 2 时基单元

定时器的核心是“计数器”：

```text
计数时钟 CK_CNT --> CNT 从 0 计数到 ARR --> 产生更新事件/更新中断 --> 重新从 0 计数
```

关键寄存器：

- `CNT`：当前计数值，通常向上计数。
- `ARR`：自动重装载值，计数器达到 ARR 后，下一次计数会触发更新事件并回到 0。
- `PSC`：预分频器，把输入时钟降低为 `CK_CNT`。
- `RCR`：重复计数器，主要用于高级定时器，可以让若干次溢出后才产生一次更新事件。

计算公式：

$$
f_{CK\_CNT}=\frac{f_{CK\_PSC}}{PSC+1}
$$

$$
f_{update}=\frac{f_{CK\_PSC}}{(PSC+1)(ARR+1)}
$$

$$
T_{update}=\frac{(PSC+1)(ARR+1)}{f_{CK\_PSC}}
$$

以 72 MHz 定时器时钟为例：

```c
PSC = 7200 - 1;   // 72MHz / 7200 = 10kHz，1个计数节拍 = 0.1ms
ARR = 10000 - 1;  // 10000个节拍 = 1s
```

所以该配置的定时周期是 1 秒。

> [!note]
>`PSC` 和 `ARR` 实际都有影子寄存器，新写入的值通常在更新事件后才会真正生效，这样能避免计数过程中值突然变化导致波形异常。

## 3 定时中断流程

以 TIM2 为例，标准库下的常用步骤：

1. 使能 TIM2 时钟。
2. 配置 `TIM_TimeBaseInitTypeDef`，设置预分频、自动重装载值、计数模式等。
3. 清一次更新标志，避免刚使能中断就立刻进入一次中断。
4. 使能更新中断 `TIM_IT_Update`。
5. 配置 NVIC。
6. 启动定时器。
7. 在中断服务函数里判断标志、处理任务、清除标志。

```c
void Timer_Init(void)
{
    RCC_APB1PeriphClockCmd(RCC_APB1Periph_TIM2, ENABLE);

    TIM_TimeBaseInitTypeDef TIM_TimeBaseInitStructure;
    TIM_TimeBaseInitStructure.TIM_ClockDivision = TIM_CKD_DIV1;
    TIM_TimeBaseInitStructure.TIM_CounterMode = TIM_CounterMode_Up;
    TIM_TimeBaseInitStructure.TIM_Period = 10000 - 1;       // ARR
    TIM_TimeBaseInitStructure.TIM_Prescaler = 7200 - 1;     // PSC
    TIM_TimeBaseInitStructure.TIM_RepetitionCounter = 0;
    TIM_TimeBaseInit(TIM2, &TIM_TimeBaseInitStructure);

    TIM_ClearFlag(TIM2, TIM_FLAG_Update);
    TIM_ITConfig(TIM2, TIM_IT_Update, ENABLE);

    NVIC_PriorityGroupConfig(NVIC_PriorityGroup_2);

    NVIC_InitTypeDef NVIC_InitStructure;
    NVIC_InitStructure.NVIC_IRQChannel = TIM2_IRQn;
    NVIC_InitStructure.NVIC_IRQChannelCmd = ENABLE;
    NVIC_InitStructure.NVIC_IRQChannelPreemptionPriority = 2;
    NVIC_InitStructure.NVIC_IRQChannelSubPriority = 2;
    NVIC_Init(&NVIC_InitStructure);

    TIM_Cmd(TIM2, ENABLE);
}

void TIM2_IRQHandler(void)
{
    if (TIM_GetITStatus(TIM2, TIM_IT_Update) == SET)
    {
        TIM_ClearITPendingBit(TIM2, TIM_IT_Update);
        // 每 1s 执行一次的任务
    }
}
```

## 4 外部时钟

定时器除了使用内部 72 MHz 时钟，也可以使用外部信号作为计数时钟。常见方式是使用 ETR 引脚：

```text
ETR 引脚输入 --> 极性选择 --> 滤波 --> 分频 --> 作为定时器计数时钟
```

外部时钟常用于对外部脉冲计数。配置要点：

- 先把对应 ETR 引脚配置成输入模式。
- 使用外部时钟模式 2 时，通常调用类似 `TIM_ETRClockMode2Config()` 的函数。
- 之后仍然像普通定时器一样设置 `PSC`、`ARR`，只是计数时钟来源变成了外部信号。
- 可以用 `TIM_GetCounter(TIMx)` 读取当前计数值。

## 5 输出比较与 PWM

输出比较模块会把 `CNT` 和比较值 `CCR` 连续比较，然后控制通道引脚输出电平。

### PWM 模式

向上计数时：

- PWM 模式 1：`CNT < CCR` 时输出有效电平，`CNT >= CCR` 时输出无效电平。
- PWM 模式 2：与 PWM 模式 1 相反。

PWM 频率：

$$
f_{PWM}=\frac{f_{CK\_PSC}}{(PSC+1)(ARR+1)}
$$

占空比：

$$
Duty=\frac{CCR}{ARR+1}
$$

配置步骤：

1. 使能定时器时钟。
2. 把 PWM 输出引脚配置为复用推挽输出。
3. 配置时基单元，确定 PWM 频率。
4. 配置 `TIM_OCInitTypeDef`，选择 PWM 模式、输出极性、比较通道。
5. 设置 `CCR`，控制占空比。
6. 启动定时器和对应输出通道。

常用函数：

```c
TIM_OC3Init(TIM2, &TIM_OCInitStructure);
TIM_OC3PreloadConfig(TIM2, TIM_OCPreload_Enable);
TIM_SetCompare3(TIM2, compare_value);  // 修改占空比
```

### 常见 PWM 应用

| 应用 | 典型参数 |
| --- | --- |
| LED 呼吸灯 | 改变 CCR，控制 LED 平均电流，从而改变亮度 |
| 舵机 | 周期 20ms，即 50Hz；脉宽约 0.5ms~2.5ms 对应不同角度 |
| 直流电机 | 改变 CCR 占空比，控制平均电压，实现调速 |

## 6 输入捕获

输入捕获会在指定边沿到来时，把当前 `CNT` 值自动存入 `CCR`，因此可以精确测量信号的时间间隔。

能测量的内容：

- 两个上升沿之间的时间：测信号周期/频率。
- 上升沿和下降沿之间的时间：测高电平时间。
- 高低电平时间结合：测占空比。

测量公式：

$$
T=\frac{\Delta CCR}{f_{capture}}
$$

$$
f=\frac{f_{capture}}{\Delta CCR}
$$

其中 `f_capture` 是输入捕获使用的计数时钟频率。

### 频率测量

常用方法：

1. 捕获第一次上升沿，记录 `CCR1`。
2. 捕获下一次上升沿，记录 `CCR2`。
3. 计算两次捕获差值：

$$
\Delta CCR = CCR2 - CCR1
$$

4. 用 `f = f_capture / ΔCCR` 得到频率。

如果定时器溢出，还需要处理溢出次数，否则两个上升沿相隔超过一个 ARR 周期时会算错。

### PWMI 测频率和占空比

同一个输入信号可以同时映射到两个通道：

- 一个通道配置为上升沿捕获，用于测周期。
- 另一个通道配置为下降沿捕获，用于测高电平时间。

计算方式：

$$
T_{period}=\frac{\Delta CCR_{rise}}{f_{capture}}
$$

$$
T_{high}=\frac{\Delta CCR_{fall}}{f_{capture}}
$$

$$
Duty=\frac{T_{high}}{T_{period}}
$$

## 7 编码器接口

TIM 的编码器接口可以自动根据两个正交编码器信号 A、B 进行计数，不需要在中断里频繁手动判断边沿。

基本原理：

```text
编码器 A/B 输入 --> TI1/TI2 --> 编码器模式 --> CNT 自动加/减
```

特点：

- A 相超前 B 相时，通常对应一个旋转方向，CNT 递增。
- B 相超前 A 相时，通常对应另一个旋转方向，CNT 递减。
- 编码器模式可以利用 A、B 之间的相位关系自动鉴相。
- 常用编码器倍频方式：不倍频、2 倍频、4 倍频。

常见配置：

```c
TIM_EncoderInterfaceConfig(
    TIM3,
    TIM_EncoderMode_TI12,
    TIM_ICPolarity_Rising,
    TIM_ICPolarity_Rising
);
```

`TIM_EncoderMode_TI12` 表示同时使用 TI1 和 TI2，对 A、B 两相都计数，属于 4 倍频模式。

## 8 编码器测速

测速的基本公式：

$$
speed=\frac{\Delta CNT}{T_{sample}}\times \frac{1}{PPR}\times \frac{60}{gear}
$$

更直观地说：

1. 每隔固定时间读取一次 `CNT`，比如 10ms、50ms。
2. 计算本次读数和上次读数的差值。
3. 处理计数器溢出或反向计数导致的回绕。
4. 把脉冲数换算成机械转角或转速。

如果使用 4 倍频，则编码器每机械周期对应的计数值是：

$$
counts = 4 \times PPR
$$

例如：

```text
编码器 PPR = 1024
编码器模式 = 4 倍频
采样时间 = 100ms
10ms 内计数增量 = 1024
```

则每秒计数增量：

$$
\frac{1024}{0.1}=10240
$$

转速：

$$
\frac{10240}{4\times1024}\times60=150rpm
$$

## 9 易错点

- `ARR` 和 `PSC` 都是“实际值 - 1”。
- 定时中断里必须清除对应标志，否则可能反复进入中断。
- 初始化时基单元后最好先清一次更新标志，再使能中断。
- PWM 占空比不是直接设置 CCR 的百分数，而是 `CCR / (ARR + 1)`。
- 输入捕获测周期时要注意定时器溢出，必要时用溢出中断或更大 ARR。
- 编码器反向转动时计数会从 0 回绕到最大值，读取差值时要做回绕处理。

## 10 相关链接

- [6-1 TIM定时中断](https://www.bilibili.com/video/BV1th411z7sn?p=13)
- [6-2 定时器定时中断&定时器外部时钟](https://www.bilibili.com/video/BV1th411z7sn?p=14)
- [6-3 TIM输出比较](https://www.bilibili.com/video/BV1th411z7sn?p=15)
- [6-4 PWM驱动LED呼吸灯&PWM驱动舵机&PWM驱动直流电机](https://www.bilibili.com/video/BV1th411z7sn?p=16)
- [6-5 TIM输入捕获](https://www.bilibili.com/video/BV1th411z7sn?p=17)
- [6-6 输入捕获模式测频率&PWMI模式测频率占空比](https://www.bilibili.com/video/BV1th411z7sn?p=18)
- [6-7 TIM编码器接口](https://www.bilibili.com/video/BV1th411z7sn?p=19)
- [6-8 编码器接口测速](https://www.bilibili.com/video/BV1th411z7sn?p=20)
