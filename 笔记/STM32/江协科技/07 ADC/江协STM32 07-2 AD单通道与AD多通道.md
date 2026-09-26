---
course: 江协科技 STM32入门教程-2023版
chapter: 07-2 AD单通道与AD多通道
type: 分集笔记
source: https://www.bilibili.com/video/BV1th411z7sn?p=22
tags:
  - STM32
  - 江协科技
  - ADC
  - AD单通道
  - AD多通道
status: draft
verify: 官方源码+接线图+课件
updated: 2026-09-26
---

# 07-2 AD 单通道与 AD 多通道

> [!abstract] 这一集只解决一个问题
> **把 07-1 讲的原理真正跑起来——怎么读一个通道，又怎么读四个通道？**
>
> 答：单通道用「**只配序列 1、非扫描、软件触发、转完读走**」的最小配置；多通道用「**同一份 `AD_Init`，但把序列配置挪到 `AD_GetValue(通道号)` 里，每转之前重配一次**」的办法，绕开「规则组只有 1 个数据寄存器」这个限制。
>
> 两个实验的差别**只在一行**：`ADC_RegularChannelConfig()` 从 `AD_Init()` 里搬进了 `AD_GetValue()`，并多了一个形参。
>
> 上一集 [[江协STM32 07-1 ADC模数转换器]] ｜ 章索引 [[江协STM32 07 ADC（章索引）]] ｜ 下一章 [[江协STM32 08 DMA（章索引）]]

## 1 两个实验分别在做什么

| | **7-1 AD 单通道** | **7-2 AD 多通道** |
| --- | --- | --- |
| 目的 | 读**一个**模拟电压，并换算成电压值显示 | 读**四个**模拟电压，一起显示 |
| 通道数 | 1（通道 0） | 4（通道 0、1、2、3） |
| GPIO | 只初始化 `GPIO_Pin_0` | 一次初始化 `GPIO_Pin_0 \| GPIO_Pin_1 \| GPIO_Pin_2 \| GPIO_Pin_3` |
| 序列配置位置 | **在 `AD_Init()` 里**配一次 | **不在此处配置**，改到 `AD_GetValue()` 每次转换前配 |
| `AD_GetValue` 原型 | `uint16_t AD_GetValue(void)` | `uint16_t AD_GetValue(uint8_t ADC_Channel)` |
| 扫描模式 | `DISABLE` | `DISABLE` |
| DMA | **不用** | **不用** |
| 显示 | `ADValue` + `Voltage`（两行） | `AD0` / `AD1` / `AD2` / `AD3`（四行，只显示读数） |

> [!tip] 这两个实验的代码骨架是**同一套**
> 把 `AD.c` 两份文件逐行对比会发现：**只有「序列配置那一行放哪」和 `AD_GetValue` 是否有形参**这两处不同，其余 `RCC`、`GPIO`、`ADC_Init`、校准、触发、等待 EOC 全部一字不差。

## 2 接线

### 2.1 7-1：一个电位器 → PA0

| 器件 | 引脚 | 接到 |
| --- | --- | --- |
| 电位器模块 | 三个脚（蓝壳蓝色三脚） | 两端接 **3.3 V / GND**，中间滑动端 → **PA0** |
| STLINK | SWDIO / SWCLK / GND / 3.3V | 最小系统板 SWD 排针 |
| OLED 显示屏 | VCC / GND / SCL / SDA | 见「OLED 下方就近的接线图」 |

**PA0 = ADC1 的通道 0**，所以第一集固定读 `ADC_Channel_0`。

**接线是怎么确认的**：官方 `7-1\Hardware\AD.c` 第 20 行写的是 `GPIO_Pin_0`、第 25 行写的是 `ADC_Channel_0`；官方接线图 `7-1 AD单通道` 里，电位器中间脚那条绿线一直拉到最小系统板 `PA0` 所在列，与源码完全吻合。

### 2.2 7-2：电位器 + 三个传感器 → PA0~PA3

官方 `7-2\Hardware\AD.c` 第 20 行一次开了四个引脚：

```c
GPIO_InitStructure.GPIO_Pin = GPIO_Pin_0 | GPIO_Pin_1 | GPIO_Pin_2 | GPIO_Pin_3;
```

对应 `main.c` 里轮流读的四个通道：

| 通道 | 引脚 | 引脚号 | 官方 `main.c` 里的读法 |
| --- | --- | --- | --- |
| 通道 0 | **PA0** | 10 | `AD_GetValue(ADC_Channel_0)` |
| 通道 1 | **PA1** | 11 | `AD_GetValue(ADC_Channel_1)` |
| 通道 2 | **PA2** | 12 | `AD_GetValue(ADC_Channel_2)` |
| 通道 3 | **PA3** | 13 | `AD_GetValue(ADC_Channel_3)` |

引脚复用可在官方引脚定义表里对到：

| 引脚 | 引脚号 | 默认复用功能 |
| --- | --- | --- |
| PA0-WKUP | 10 | `WKUP/USART2_CTS/`**`ADC12_IN0`**`/TIM2_CH1_ETR` |
| PA1 | 11 | `USART2_RTS/`**`ADC12_IN1`**`/TIM2_CH2` |
| PA2 | 12 | `USART2_TX/`**`ADC12_IN2`**`/TIM2_CH3` |
| PA3 | 13 | `USART2_RX/`**`ADC12_IN3`**`/TIM2_CH4` |

**四个模拟信号源**（按官方接线图上模块的丝印）：

| 模块 | 敏感元件 | 接哪个脚 |
| --- | --- | --- |
| 电位器（7-1 就用过） | 可调电阻 | 三个模拟输出之一 |
| 反射式红外传感器 | 红外发射管 + 接收管 | 三个模拟输出之一 |
| 热敏传感器 | 热敏电阻 | 三个模拟输出之一 |
| 光敏传感器 | 光敏电阻 | 三个模拟输出之一 |

每个模块都从 `AO`（模拟输出）脚取电压，`DO` 脚**悬空不接**（DO 是数字量，本实验要读模拟量）。

> [!warning] 哪个模块接哪个通道，官方材料里没有写明
> 官方源码只写了「读通道 0/1/2/3」，**没有说明哪个模块接哪个引脚**；官方接线图上四条模拟输出线走到面包板后交织在一起，本次按像素追踪**无法唯一确定**对应关系（详见 `## 待核对`）。
>
> **不确定接线时，用这个办法现场确认**：下好程序后，OLED 上四行分别显示 AD0~AD3。用手遮住某个传感器的敏感元件，**哪一行的读数明显变化，那个传感器就接在那个通道上**。电位器则是转旋钮，读数跟着变的那一路就是它。

> [!note] 本集接线图没有被官方复用
> 官方 `1-1 接线图\` 里有些实验因为接线完全相同而**共用同一张图**（例如 `7-2 AD多通道` 与 `8-2 DMA+AD多通道` 就是同一张）。本批两集不属于这种情况：`7-1 AD单通道`（2600×990）只有电位器，`7-2 AD多通道`（2600×1371）是电位器加三个传感器模块，**两张图内容不同**。

## 3 单通道：`AD_Init()` 完整代码

官方 `7-1 AD单通道\Hardware\AD.c` 全文（**照抄，未做任何改写**）：

```c
#include "stm32f10x.h"                  // Device header

/**
  * 函    数：AD初始化
  * 参    数：无
  * 返 回 值：无
  */
void AD_Init(void)
{
	/*开启时钟*/
	RCC_APB2PeriphClockCmd(RCC_APB2Periph_ADC1, ENABLE);	//开启ADC1的时钟
	RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOA, ENABLE);	//开启GPIOA的时钟
	
	/*设置ADC时钟*/
	RCC_ADCCLKConfig(RCC_PCLK2_Div6);						//选择时钟6分频，ADCCLK = 72MHz / 6 = 12MHz
	
	/*GPIO初始化*/
	GPIO_InitTypeDef GPIO_InitStructure;
	GPIO_InitStructure.GPIO_Mode = GPIO_Mode_AIN;
	GPIO_InitStructure.GPIO_Pin = GPIO_Pin_0;
	GPIO_InitStructure.GPIO_Speed = GPIO_Speed_50MHz;
	GPIO_Init(GPIOA, &GPIO_InitStructure);					//将PA0引脚初始化为模拟输入
	
	/*规则组通道配置*/
	ADC_RegularChannelConfig(ADC1, ADC_Channel_0, 1, ADC_SampleTime_55Cycles5);		//规则组序列1的位置，配置为通道0
	
	/*ADC初始化*/
	ADC_InitTypeDef ADC_InitStructure;						//定义结构体变量
	ADC_InitStructure.ADC_Mode = ADC_Mode_Independent;		//模式，选择独立模式，即单独使用ADC1
	ADC_InitStructure.ADC_DataAlign = ADC_DataAlign_Right;	//数据对齐，选择右对齐
	ADC_InitStructure.ADC_ExternalTrigConv = ADC_ExternalTrigConv_None;	//外部触发，使用软件触发，不需要外部触发
	ADC_InitStructure.ADC_ContinuousConvMode = DISABLE;		//连续转换，失能，每转换一次规则组序列后停止
	ADC_InitStructure.ADC_ScanConvMode = DISABLE;			//扫描模式，失能，只转换规则组的序列1这一个位置
	ADC_InitStructure.ADC_NbrOfChannel = 1;					//通道数，为1，仅在扫描模式下，才需要指定大于1的数，在非扫描模式下，只能是1
	ADC_Init(ADC1, &ADC_InitStructure);						//将结构体变量交给ADC_Init，配置ADC1
	
	/*ADC使能*/
	ADC_Cmd(ADC1, ENABLE);									//使能ADC1，ADC开始运行
	
	/*ADC校准*/
	ADC_ResetCalibration(ADC1);								//固定流程，内部有电路会自动执行校准
	while (ADC_GetResetCalibrationStatus(ADC1) == SET);
	ADC_StartCalibration(ADC1);
	while (ADC_GetCalibrationStatus(ADC1) == SET);
}

/**
  * 函    数：获取AD转换的值
  * 参    数：无
  * 返 回 值：AD转换的值，范围：0~4095
  */
uint16_t AD_GetValue(void)
{
	ADC_SoftwareStartConvCmd(ADC1, ENABLE);					//软件触发AD转换一次
	while (ADC_GetFlagStatus(ADC1, ADC_FLAG_EOC) == RESET);	//等待EOC标志位，即等待AD转换结束
	return ADC_GetConversionValue(ADC1);					//读数据寄存器，得到AD转换的结果
}
```

`AD.h`：

```c
#ifndef __AD_H
#define __AD_H

void AD_Init(void);
uint16_t AD_GetValue(void);

#endif
```

### 3.1 逐段拆解

```text
① 开时钟      RCC_APB2Periph_ADC1 / RCC_APB2Periph_GPIOA   —— ADC1 和 GPIOA 都挂 APB2
② 配 ADC 时钟  RCC_ADCCLKConfig(RCC_PCLK2_Div6)             —— ADCCLK = 72MHz/6 = 12MHz
③ 配 GPIO      GPIO_Mode_AIN（模拟输入）                     —— 引脚直接接入内部 ADC
④ 配规则组序列 ADC_RegularChannelConfig(序列1 = 通道0, 55.5 周期)
⑤ 配 ADC       ADC_Init：独立/右对齐/软件触发/非连续/非扫描/1 通道
⑥ 使能 ADC     ADC_Cmd(ENABLE)
⑦ 校准         复位校准→等→启动校准→等
```

### 3.2 关键点解释

**② 为什么用 6 分频。** `PCLK2 = 72 MHz`，6 分频得 **12 MHz**，正是源码注释写明的值（`//ADCCLK = 72MHz / 6 = 12MHz`）。分频可选项只有 `Div2 / Div4 / Div6 / Div8` 四种。

**③ 为什么是 `GPIO_Mode_AIN` 而不是浮空输入。** 课件（第 3 章 GPIO 模式表，Slide 87）对模拟输入的说明是：「**GPIO 无效，引脚直接接入内部 ADC**」。配成 `AIN` 后，引脚上的数字输入通道被切断，模拟电压原封不动送进 ADC；若配成浮空输入，施密特触发器会把电压「整形」成 0/1，读数就废了。注意 `GPIO_Speed` 在模拟输入下其实不起作用，官方源码照模板填了 `50MHz`。

**④ 序列位置为什么写 `1` 不写 `0`。** `ADC_RegularChannelConfig` 的第三个参数 `Rank` 是「序列里的第几个位置」，**从 1 开始编号**。

**④ 为什么采样时间选 55.5 周期。** 这是可选里比较长的一档，给采样电容留足充电时间，对电位器/传感器这类信号源最省心。总转换时间为 `(55.5 + 12.5) / 12MHz ≈ 5.67 µs`，远快于主循环里 `Delay_ms(100)` 的节奏。

**⑥ 使能必须在校准之前。** 课件 Slide 98 原文说「启动校准前，ADC 必须处于关电状态超过至少两个 ADC 时钟周期」——`ADC_Cmd(ADC1, ENABLE)` 完成的就是这个「上电」动作，所以它排在 `ADC_ResetCalibration` 前面。

**⑦ 两个 `while` 不能省。** 校准由硬件自己跑，CPU 只能轮询状态位等它。少了等待，校准没结束就开始转换，读出的前几个值不准。

**`AD_GetValue` 里没有清 EOC 标志。** 官方代码在 `while` 等到 EOC 后**直接读 `ADC_GetConversionValue()`**，读数据寄存器这个动作本身就清掉了 EOC，所以不需要额外的 `ADC_ClearFlag`。

## 4 单通道：`main.c` 与电压换算

官方 `7-1 AD单通道\User\main.c` 全文：

```c
#include "stm32f10x.h"                  // Device header
#include "Delay.h"
#include "OLED.h"
#include "AD.h"

uint16_t ADValue;			//定义AD值变量
float Voltage;				//定义电压变量

int main(void)
{
	/*模块初始化*/
	OLED_Init();			//OLED初始化
	AD_Init();				//AD初始化
	
	/*显示静态字符串*/
	OLED_ShowString(1, 1, "ADValue:");
	OLED_ShowString(2, 1, "Voltage:0.00V");
	
	while (1)
	{
		ADValue = AD_GetValue();					//获取AD转换的值
		Voltage = (float)ADValue / 4095 * 3.3;		//将AD值线性变换到0~3.3的范围，表示电压
		
		OLED_ShowNum(1, 9, ADValue, 4);				//显示AD值
		OLED_ShowNum(2, 9, Voltage, 1);				//显示电压值的整数部分
		OLED_ShowNum(2, 11, (uint16_t)(Voltage * 100) % 100, 2);	//显示电压值的小数部分
		
		Delay_ms(100);			//延时100ms，手动增加一些转换的间隔时间
	}
}
```

### 4.1 电压换算公式

**官方源码第 22 行写的是**：

```c
Voltage = (float)ADValue / 4095 * 3.3;      //将AD值线性变换到0~3.3的范围，表示电压
```

也就是说，**官方工程用的是「除以 4095」**。这是个**实测事实**，照抄即可。

而 07-1 讲分辨率时的常见写法是「除以 4096」：

```c
Voltage = ADValue / 4096.0f * 3.3f;         //按「4096 个档位」的写法
```

| 写法 | 依据 | 当 `ADValue = 4095` 时算出的电压 |
| --- | --- | --- |
| `/ 4095`（官方源码） | 「读数 4095 应当对应满量程 3.3 V」，即按**最大读数**归一化 | 3.3000 V |
| `/ 4096`（按档位数） | 12 位共 4096 个档位，每档 1 LSB，按**档位数**归一化 | 3.2992 V |

两者最大只差 `3.3 × (1/4095 − 1/4096) ≈ 0.0002 V`（0.2 mV，不到 1 LSB），**在 OLED 保留两位小数的显示精度下完全看不出来**。

> [!note] 本页怎么写
> **代码一律照官方原文**，所以例子里的电压换算写 `/ 4095`。
> 同时把两种写法的差别摆出来：这不是「错误」，而是同一个公式的两种归一化约定。**关键是要知道自己用的是哪一种，别在同一份工程里混用。**

### 4.2 显示格式：怎么把 `Voltage` 拆成整数和小数

OLED 只有 `OLED_ShowNum(行, 列, 数字, 长度)` 这个函数，**不能直接显示小数**。官方用了一个巧办法，把 `Voltage` 拆成两段分别显示：

```c
OLED_ShowString(2, 1, "Voltage:0.00V");                     // 先打印固定模板，"0.00" 是占位
OLED_ShowNum(2, 9, Voltage, 1);                             // 第 9 列写整数部分（1 位）
OLED_ShowNum(2, 11, (uint16_t)(Voltage * 100) % 100, 2);    // 第 11 列写小数部分（2 位）
```

逐段看：

| 表达式 | 结果 | 说明 |
| --- | --- | --- |
| `Voltage` | `2.3456` | 直接交给 `OLED_ShowNum` 会自动转成整数 `2` |
| `Voltage * 100` | `234.56` | 小数点后两位「搬」到整数部分 |
| `(uint16_t)(...)` | `234` | 截断取整，得到整数位+两位小数拼成的数 |
| `% 100` | `34` | 取后两位，就是小数部分 |

列位置的排布（模板 `Voltage:0.00V`）：

```text
列:  1  2  3  4  5  6  7  8  9  10 11 12 13 14
     V  o  l  t  a  g  e  :  0  .  0  0  V
                            ↑     ↑
                    整数(列9)   小数两位(列11-12)
```

`0.00` 是模板里预先画好的「小数点占位」，真正的小数点在第 10 列，**永远不会被覆盖**，所以每次刷新只需要改列 9 和列 11~12。

> [!warning] 为什么能这么写
> 因为 `Delay_ms(100)` 让数据每 100 ms 才变一次，而且整数部分只会是 0~3 这四种情况之一，**模板 `Voltage:0.00V` 永远不会出现「位数变了模板对不上」的问题**。这是个「够用就好」的取巧写法，写别的显示数据时要谨慎。

### 4.3 现象

转动电位器旋钮：

- `ADValue` 在 **0 ~ 4095** 之间变化
- `Voltage` 在 **0.00 ~ 3.30** 之间跟着变化，两者线性对应

## 5 多通道：为什么要每次重配通道

### 5.1 问题的根源：规则组只有 1 个数据寄存器

07-1 讲过：**规则组只有 1 个数据寄存器** `ADC_DR`（课件 Slide 89 标注「规则组结果 ×1」）。

于是「扫描模式 + 4 个通道」会发生这种事：

```text
触发 → 转通道0 → 结果写进 ADC_DR
     → 转通道1 → 结果【覆盖】ADC_DR     ← 通道0 的结果没了
     → 转通道2 → 结果【覆盖】ADC_DR
     → 转通道3 → 结果【覆盖】ADC_DR → EOC
读一次 ADC_DR，只能拿到【通道3】的结果
```

要拿到全部四个值，只有两条路：

1. **每转完一个通道立刻读走**，再换下一个通道——**本集用的就是这条**。
2. **用 DMA 自动把每个结果搬到内存数组**——留到第 8 章 8-2。

### 5.2 官方做法：把序列配置搬进 `AD_GetValue()`

官方 `7-2\Hardware\AD.c` 里，`AD_Init()` 中原本配置规则组序列那一行被**删掉了**，原位留下一句注释：

```c
	/*不在此处配置规则组序列，而是在每次AD转换前配置，这样可以灵活更改AD转换的通道*/
```

然后 `AD_GetValue()` 多了一个形参，并在**触发转换之前**重新指定通道：

```c
/**
  * 函    数：获取AD转换的值
  * 参    数：ADC_Channel 指定AD转换的通道，范围：ADC_Channel_x，其中x可以是0/1/2/3
  * 返 回 值：AD转换的值，范围：0~4095
  */
uint16_t AD_GetValue(uint8_t ADC_Channel)
{
	ADC_RegularChannelConfig(ADC1, ADC_Channel, 1, ADC_SampleTime_55Cycles5);	//在每次转换前，根据函数形参灵活更改规则组的通道1
	ADC_SoftwareStartConvCmd(ADC1, ENABLE);					//软件触发AD转换一次
	while (ADC_GetFlagStatus(ADC1, ADC_FLAG_EOC) == RESET);	//等待EOC标志位，即等待AD转换结束
	return ADC_GetConversionValue(ADC1);					//读数据寄存器，得到AD转换的结果
}
```

执行时的时间线：

```text
AD_GetValue(通道0): 配序列1=通道0 → 触发 → 等EOC → 读走  → 返回 AD0
AD_GetValue(通道1): 配序列1=通道1 → 触发 → 等EOC → 读走  → 返回 AD1
AD_GetValue(通道2): 配序列1=通道2 → 触发 → 等EOC → 读走  → 返回 AD2
AD_GetValue(通道3): 配序列1=通道3 → 触发 → 等EOC → 读走  → 返回 AD3
```

**每次都是「配 → 触发 → 等 → 读」四步一组，读完立刻取走，所以永远不存在覆盖问题。**

### 5.3 本集为什么不用扫描模式、不用 DMA

| 办法 | 扫描模式 | 通道数 | DMA | 本集是否用 |
| --- | --- | --- | --- | --- |
| 每次重配序列 + 单独触发 | `DISABLE` | `1` | 不需要 | **用这个** |
| 扫描模式 + DMA 搬运 | `ENABLE` | = 通道个数 | 需要 | 不用（8-2 才讲） |

官方 `7-2\Hardware\AD.c` 里的 `ADC_Init` 配置和 7-1 **一个字都没改**：

```c
	ADC_InitStructure.ADC_ContinuousConvMode = DISABLE;		//连续转换，失能，每转换一次规则组序列后停止
	ADC_InitStructure.ADC_ScanConvMode = DISABLE;			//扫描模式，失能，只转换规则组的序列1这一个位置
	ADC_InitStructure.ADC_NbrOfChannel = 1;					//通道数，为1，仅在扫描模式下，才需要指定大于1的数，在非扫描模式下，只能是1
```

三条互相印证：**非扫描 + 通道数 1 + 序列只有位置 1**，所以每次触发只转一个通道，正好配上「每次重配序列」的用法。

> [!tip] `ADC_NbrOfChannel` 只在扫描模式下才有意义
> 官方源码注释写得很清楚：「仅在扫描模式下，才需要指定大于1的数，在非扫描模式下，只能是1」。本集是非扫描，所以即使要读四个通道，这里也**必须**写 `1`。**通道个数是靠「调用 `AD_GetValue` 四次」实现的，不是靠这个参数。**

### 5.4 多通道 `AD.h`

```c
#ifndef __AD_H
#define __AD_H

void AD_Init(void);
uint16_t AD_GetValue(uint8_t ADC_Channel);

#endif
```

### 5.5 多通道 `main.c`

官方 `7-2 AD多通道\User\main.c` 全文：

```c
#include "stm32f10x.h"                  // Device header
#include "Delay.h"
#include "OLED.h"
#include "AD.h"

uint16_t AD0, AD1, AD2, AD3;	//定义AD值变量

int main(void)
{
	/*模块初始化*/
	OLED_Init();				//OLED初始化
	AD_Init();					//AD初始化
	
	/*显示静态字符串*/
	OLED_ShowString(1, 1, "AD0:");
	OLED_ShowString(2, 1, "AD1:");
	OLED_ShowString(3, 1, "AD2:");
	OLED_ShowString(4, 1, "AD3:");
	
	while (1)
	{
		AD0 = AD_GetValue(ADC_Channel_0);		//单次启动ADC，转换通道0
		AD1 = AD_GetValue(ADC_Channel_1);		//单次启动ADC，转换通道1
		AD2 = AD_GetValue(ADC_Channel_2);		//单次启动ADC，转换通道2
		AD3 = AD_GetValue(ADC_Channel_3);		//单次启动ADC，转换通道3
		
		OLED_ShowNum(1, 5, AD0, 4);				//显示通道0的转换结果AD0
		OLED_ShowNum(2, 5, AD1, 4);				//显示通道1的转换结果AD1
		OLED_ShowNum(3, 5, AD2, 4);				//显示通道2的转换结果AD2
		OLED_ShowNum(4, 5, AD3, 4);				//显示通道3的转换结果AD3
		
		Delay_ms(100);			//延时100ms，手动增加一些转换的间隔时间
	}
}
```

### 5.6 多通道实验的现象

OLED 四行分别显示 `AD0`~`AD3`，每行 4 位数，范围 **0~4095**：

| 操作 | 现象 |
| --- | --- |
| 遮住光敏电阻 | 对应那一行的读数明显变化 |
| 用手靠近/远离反射式红外传感器 | 对应那一行的读数明显变化 |
| 捏住热敏电阻（体温加热） | 对应那一行的读数慢慢变化 |
| 转动电位器 | 对应那一行的读数跟着转 |

四个通道**互不干扰**——这正是「每次读走再换下一个」的效果。

> [!note] 本集不显示电压
> 7-1 的 `main.c` 有 `Voltage` 变量和换算，**7-2 的 `main.c` 里没有**——四行全显示原始读数（`OLED_ShowNum(..., AD0, 4)`）。想在多通道基础上加电压显示，按 7-1 的换算法各加一份即可。

## 6 易错点

- [ ] `GPIO_Mode` 配成浮空输入或上拉输入 → 模拟电压被施密特触发器整形，读数无意义。必须是 **`GPIO_Mode_AIN`**。
- [ ] 忘了 `RCC_ADCCLKConfig(RCC_PCLK2_Div6)` → 用了默认的 2 分频（36 MHz），ADC 超速。
- [ ] 忘了 `RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOA, ENABLE)` → 只开了 ADC 时钟，GPIO 配置不生效。
- [ ] `ADC_Cmd(ADC1, ENABLE)` 写在 `ADC_ResetCalibration()` **后面** → 违反课件「启动校准前 ADC 必须处于关电状态超过两个 ADC 时钟周期」，校准无效。
- [ ] 校准的两个 `while` 等待漏写 → 校准未完成就转换，前几个读数不准。
- [ ] 把校准写进 `AD_GetValue()` → 每次转换都重跑校准，速度白白变慢。
- [ ] `ADC_RegularChannelConfig` 的 `Rank` 写成 `0` → 非法值，序列不生效（官方写 `1`）。
- [ ] **多通道忘了「每次转换前重配通道」** → 四次 `AD_GetValue` 读回来的全是同一个通道。
- [ ] **多通道沿用 7-1 的 `AD_GetValue(void)`** → 形参没加，无法指定通道。`AD.h` 里的声明也要一起改。
- [ ] 想用扫描模式读多通道，却没开 DMA、也没每次读走 → 规则组只有 1 个数据寄存器，**只能读到最后一个通道**。
- [ ] 非扫描模式下把 `ADC_NbrOfChannel` 写成 `4` → 注释写明「在非扫描模式下，只能是1」。
- [ ] 四个 GPIO 引脚分两次 `GPIO_Init` 而第二次忘了改 `GPIO_Pin` → 后一次覆盖前一次。官方是用 `|` 一次配齐。
- [ ] 电压换算里 `ADValue` 忘了强制转 `float` → `ADValue / 4095` 按整数除法算，结果恒为 0。
- [ ] 把 OLED 显示小数当成「有专门的函数」 → `OLED_ShowNum` 只能显示整数，必须像官方那样拆整数/小数两段。
- [ ] 以为「读数 4095 就是 3.3000 V、读数 0 就是 0.0000 V」→ 那是理想值；实际有失调误差和噪声，末位会跳几个字。

## 7 自测

1. 7-2 的 `AD_Init()` 和 7-1 相比，**删掉了哪一行**？为什么删？
2. 7-2 为什么不需要扫描模式、也不需要 DMA，就能读到四个通道？
3. 如果 7-1 的 `AD_GetValue(void)` 原封不动用在 7-2 里，会出现什么现象？
4. `ADC_NbrOfChannel` 在 7-2 里写的是几？为什么不是 4？
5. 把 `Voltage = (float)ADValue / 4095 * 3.3;` 写成 `Voltage = ADValue / 4095 * 3.3;` 会怎样？如果改成 `/ 4096` 呢？

> [!success]- 参考答案
> 1. 删掉了 `AD_Init()` 里的那一行 `ADC_RegularChannelConfig(ADC1, ADC_Channel_0, 1, ADC_SampleTime_55Cycles5);`，原位只留注释「不在此处配置规则组序列，而是在每次AD转换前配置」。删的原因：**一个固定的序列配置只能对应一个通道**，要灵活换通道就得把配置挪到每次转换之前。
> 2. 因为 `AD_GetValue(通道)` 把「**配序列 → 软件触发 → 等 EOC → 立刻读走**」做成了一个闭环：每次只转一个通道，转完马上把 `ADC_DR` 里的值取走，所以永远不会被下一个通道覆盖。四条通道就是调用四次，各自独立。
> 3. `AD_GetValue()` 没有形参，无法指定通道，四次调用读回来的**全是 `AD_Init` 里配的那个通道（通道 0）**，OLED 上 AD0~AD3 四行数字完全相同；转动电位器时四行一起变，遮住光敏传感器则四行都不动。
> 4. 写的是 **`1`**。因为 `ADC_ScanConvMode = DISABLE`（非扫描模式），官方注释写明「通道数，为1，仅在扫描模式下，才需要指定大于1的数，在非扫描模式下，只能是1」。四个通道是靠调用 `AD_GetValue` 四次实现的，与这个参数无关。
> 5. 写成 `ADValue / 4095 * 3.3`：`ADValue` 是 `uint16_t`，`ADValue / 4095` 走**整数除法**，`ADValue` 小于 4095 时结果为 0，于是 `Voltage` **恒为 0.00**（只有满量程时才是 3.3）。**必须写成 `(float)ADValue / 4095 * 3.3`**。
>    改成 `/ 4096`：能正常工作，数值与 `/ 4095` **最大只差约 0.2 mV**（不到 1 LSB），OLED 两位小数看不出区别。两种归一化约定都行，**别混用**即可。

## 8 待核对

- [ ] **三个传感器模块分别接在 PA0~PA3 的哪个脚**。官方 `7-2\Hardware\AD.c` 第 20 行只写了 `GPIO_Pin_0 | GPIO_Pin_1 | GPIO_Pin_2 | GPIO_Pin_3`，`main.c` 只写了读 `ADC_Channel_0~3`，**都没有把「哪个模块」和「哪个通道」对应起来**；官方接线图 `7-2 AD多通道` 上三条 AO 走线在面包板区域交叉汇入同一组连线，本次以像素追踪**无法唯一确定**。本页因此没有写成肯定句，并给出「遮住某个传感器、看哪一行读数变化」的现场判别办法。
- [ ] **7-1 电位器接 PA0 是本批唯一确定的模块↔引脚对应**（依据：`7-1\Hardware\AD.c` 第 20 行 `GPIO_Pin_0` + 第 25 行 `ADC_Channel_0`；接线图 `7-1 AD单通道` 中电位器中间脚绿线拉到 PA0 列）。7-2 沿用了这个电位器，但同样没有官方文字确认它在 7-2 里仍占通道 0。
- [ ] **官方源码 `/ 4095` 是老师的刻意选择还是笔误**。课件 Slide 86 只给了「转换结果范围 0~4095」，**没有给电压换算公式**；课件里也查不到 `/4095` 或 `/4096` 的写法。本页按铁律**照抄源码原文写 `/ 4095`**，同时把 `/ 4096` 的写法与两者差异（< 1 LSB）并列说明，未改动依据。
- [ ] **`AD_GetValue()` 读到 EOC 后不清标志是否在所有情况下都安全**。官方代码依赖「读 `ADC_DR` 自动清 EOC」这一硬件行为，源码中没有 `ADC_ClearFlag(ADC1, ADC_FLAG_EOC)`。库函数 `ADC_ClearFlag` 在 `stm32f10x_adc.h` 第 461 行确实存在，但官方未调用。
- [ ] 视频里老师对本集接线顺序与「哪个模块接哪个脚」的口头说明（这类信息不在源码、接线图、课件文本中）。

## 核对记录（2026-09-26）

> [!success] 核对依据
> - 官方配套源码：`STM32Project-有注释版\7-1 AD单通道\Hardware\AD.c`（57 行）、`AD.h`（7 行）、`User\main.c`（30 行）；`7-2 AD多通道\Hardware\AD.c`（57 行）、`AD.h`（7 行）、`User\main.c`（34 行）
> - 官方接线图：`ground-truth\接线图\7-1 AD单通道.png`（2600×990）、`7-2 AD多通道.png`（2600×1371）；并由官方原图 `STM32Project-有注释版\1-1 接线图\7-1 AD单通道.jpg`（8946×3405）、`7-2 AD多通道.jpg`（8946×4718）放大复核
> - 引脚定义表：`ground-truth\引脚定义_xlsx原文.txt`
> - 课程课件文本：`ground-truth\课件文本.md`（Slide 86 ADC 简介、89 ADC 基本结构、90 输入通道、97 转换时间、98 校准）；第 3 章 GPIO 模式表（Slide 87）
> - ST 标准外设库：`STM32F10x_StdPeriph_Lib_V3.5.0\...\inc\stm32f10x_adc.h`、`stm32f10x_rcc.h`、`stm32f10x_gpio.h`

| 核对项 | 笔记值 | 官方依据（精确到文件名与行号） | 结论 |
| --- | --- | --- | --- |
| 7-1 引脚 | PA0 | `7-1\Hardware\AD.c` 第 20 行 `GPIO_InitStructure.GPIO_Pin = GPIO_Pin_0;`；第 22 行注释「将PA0引脚初始化为模拟输入」 | 一致 |
| 7-1 通道 | 通道 0（序列 1） | `7-1\Hardware\AD.c` 第 25 行 `ADC_RegularChannelConfig(ADC1, ADC_Channel_0, 1, ADC_SampleTime_55Cycles5);` | 一致 |
| 7-1 电位器接线 | 电位器中间脚 → PA0 | 接线图 `7-1 AD单通道`：电位器在面包板中央，滑动端绿线拉到最小系统板 PA0 列；与源码 `GPIO_Pin_0` 吻合 | 一致 |
| 7-2 引脚 | PA0、PA1、PA2、PA3 | `7-2\Hardware\AD.c` 第 20 行 `GPIO_Pin_0 \| GPIO_Pin_1 \| GPIO_Pin_2 \| GPIO_Pin_3`；第 22 行注释「将PA0、PA1、PA2和PA3引脚初始化为模拟输入」 | 一致 |
| 7-2 通道 | 通道 0、1、2、3 | `7-2\User\main.c` 第 22~25 行 `AD_GetValue(ADC_Channel_0/1/2/3)` | 一致 |
| 通道↔引脚映射 | 通道 0/1/2/3 = PA0/PA1/PA2/PA3 | `引脚定义_xlsx原文.txt` 第 12~15 行 `ADC12_IN0`~`ADC12_IN3`；课件 Slide 90 通道 0~3 对应 PA0~PA3 | 一致 |
| 7-2 用到的模拟源 | 反射式红外传感器、热敏传感器、光敏传感器 + 电位器 | 官方接线图 `7-2 AD多通道` 原图 8946×4718 放大后，三块模块丝印为「反射式红外传感器」「热敏传感器」「光敏传感器」，各带 `AO/DO/GND/VCC`；电位器来自 7-1 接线 | 一致 |
| **哪个模块接哪个通道** | **官方材料中未见** | `7-2\Hardware\AD.c` 与 `User\main.c` 均只出现通道号，未出现模块名；接线图上 AO 走线交叉汇入同一组连线，像素追踪无法唯一确定 | 已标注（未写成肯定句，已进 `## 待核对` 并给出现场判别办法） |
| 7-1 / 7-2 接线图是否同一张 | 不同 | `7-1 AD单通道.png` 2600×990（只有电位器）与 `7-2 AD多通道.png` 2600×1371（电位器 + 三模块）内容不同 | 一致 |
| ADC 时钟 | `RCC_PCLK2_Div6`，ADCCLK = 12 MHz | 两份 `Hardware\AD.c` 均为第 15 行 `RCC_ADCCLKConfig(RCC_PCLK2_Div6);` + 注释「选择时钟6分频，ADCCLK = 72MHz / 6 = 12MHz」 | 一致 |
| ADCCLK 可选值 | Div2 / Div4 / Div6 / Div8 四种 | `stm32f10x_rcc.h` 第 429~434 行四个枚举 + `IS_RCC_ADCCLK` 断言 | 一致 |
| GPIO 模式 | `GPIO_Mode_AIN` | 两份 `Hardware\AD.c` 均为第 19 行 `GPIO_InitStructure.GPIO_Mode = GPIO_Mode_AIN;`；`stm32f10x_gpio.h` 第 72 行 `GPIO_Mode_AIN = 0x0`；课件（第 3 章 GPIO 模式表）「模拟输入：GPIO 无效，引脚直接接入内部 ADC」 | 一致 |
| 采样时间 | `ADC_SampleTime_55Cycles5` | `7-1\Hardware\AD.c` 第 25 行、`7-2\Hardware\AD.c` 第 53 行；`stm32f10x_adc.h` 第 218 行 `#define ADC_SampleTime_55Cycles5 ((uint8_t)0x05)` | 一致 |
| 序列位置 Rank | `1` | 两份 `AD.c` 的 `ADC_RegularChannelConfig` 第三实参均为 `1` | 一致 |
| ADC 模式 | `ADC_Mode_Independent` | 两份 `AD.c` 第 29/28 行；`stm32f10x_adc.h` 第 94 行 `#define ADC_Mode_Independent ((uint32_t)0x00000000)` | 一致 |
| 数据对齐 | `ADC_DataAlign_Right` | 两份 `AD.c` 第 30/29 行；`stm32f10x_adc.h` 第 162 行 `#define ADC_DataAlign_Right ((uint32_t)0x00000000)` | 一致 |
| 外部触发 | `ADC_ExternalTrigConv_None`（软件触发） | 两份 `AD.c` 第 31/30 行；`stm32f10x_adc.h` 第 131 行 `#define ADC_ExternalTrigConv_None ((uint32_t)0x000E0000)`；触发函数 `ADC_SoftwareStartConvCmd` 在 `stm32f10x_adc.h` 第 438 行 | 一致 |
| 连续转换 | `DISABLE` | 两份 `AD.c` 第 32/31 行，注释「每转换一次规则组序列后停止」 | 一致 |
| 扫描模式 | `DISABLE` | 两份 `AD.c` 第 33/32 行，注释「只转换规则组的序列1这一个位置」 | 一致 |
| `ADC_NbrOfChannel` | `1` | 两份 `AD.c` 第 34/33 行，注释「仅在扫描模式下，才需要指定大于1的数，在非扫描模式下，只能是1」 | 一致 |
| 7-1 序列配置位置 | 在 `AD_Init()` 里 | `7-1\Hardware\AD.c` 第 25 行位于 `AD_Init` 内 | 一致 |
| 7-2 序列配置位置 | 在 `AD_GetValue()` 里，每次转换前 | `7-2\Hardware\AD.c` 第 24 行注释「不在此处配置规则组序列，而是在每次AD转换前配置，这样可以灵活更改AD转换的通道」；第 53 行配置语句位于 `AD_GetValue` 内 | 一致 |
| `AD_GetValue` 原型 | 7-1 为 `(void)`，7-2 为 `(uint8_t ADC_Channel)` | `7-1\Hardware\AD.h` 第 5 行 `uint16_t AD_GetValue(void);`；`7-2\Hardware\AD.h` 第 5 行 `uint16_t AD_GetValue(uint8_t ADC_Channel);` | 一致 |
| 使能与校准顺序 | `ADC_Cmd(ENABLE)` 在 `ADC_ResetCalibration` 之前 | 两份 `AD.c` 第 38/37 行（`ADC_Cmd`）在第 41/40 行（`ADC_ResetCalibration`）之前；课件 Slide 98「启动校准前，ADC 必须处于关电状态超过至少两个 ADC 时钟周期」 | 一致 |
| 校准等待 | 两个 `while` 轮询状态 | 两份 `AD.c` 第 42~44 行 / 41~43 行；库函数 `ADC_GetResetCalibrationStatus`、`ADC_GetCalibrationStatus` 在 `stm32f10x_adc.h` 第 435、437 行 | 一致 |
| 等待 EOC 的方式 | `while (ADC_GetFlagStatus(ADC1, ADC_FLAG_EOC) == RESET);` | 两份 `AD.c` 第 55/55 行；`ADC_FLAG_EOC` 在 `stm32f10x_adc.h` 第 330 行 | 一致 |
| 读到 EOC 后是否清标志 | 未显式清（依赖读 `ADC_DR` 自动清） | 两份 `AD.c` 中均无 `ADC_ClearFlag` 调用；`ADC_ClearFlag` 存在于 `stm32f10x_adc.h` 第 461 行但未被使用 | 一致（如实照抄源码，已在 `## 待核对` 记录） |
| 电压换算 | `(float)ADValue / 4095 * 3.3` | `7-1\User\main.c` 第 22 行原文 `Voltage = (float)ADValue / 4095 * 3.3;` 与注释「将AD值线性变换到0~3.3的范围，表示电压」 | 一致（本页按铁律照抄 `/ 4095`；课件无此公式，`/ 4096` 的差别已在 4.1 节并列说明，未改动依据） |
| OLED 显示格式 | 1 行 `ADValue:` + 整数 4 位；2 行模板 `Voltage:0.00V`，整数 1 位 + 小数 2 位 | `7-1\User\main.c` 第 16~17、24~26 行逐行对照 | 一致 |
| 7-2 不显示电压 | 四行全是原始读数 | `7-2\User\main.c` 第 15~18、27~30 行只调用 `OLED_ShowNum(..., AD0~AD3, 4)`，文件中无 `Voltage` 变量 | 一致 |
| 主循环延时 | `Delay_ms(100)` | `7-1\User\main.c` 第 28 行、`7-2\User\main.c` 第 32 行，注释「延时100ms，手动增加一些转换的间隔时间」 | 一致 |
| 多通道不用 DMA | 否 | `7-2\Hardware\AD.c` 中无 `ADC_DMACmd` 调用；`stm32f10x_adc.h` 第 432 行虽有 `ADC_DMACmd` 原型但未被使用 | 一致 |
| 多通道不用扫描模式 | `ADC_ScanConvMode = DISABLE` | `7-2\Hardware\AD.c` 第 32 行 | 一致（与「每次重配序列」的用法自洽） |
| 单次转换时间 | `(55.5+12.5)/12MHz ≈ 5.67 µs` | 由 `AD.c` 第 15 行 `Div6`（注释 12MHz）与第 25/53 行 `ADC_SampleTime_55Cycles5` 算出；公式依据课件 Slide 97 `T_CONV = 采样时间 + 12.5` | 一致（为本页推算，非源码原文） |
| 规则组数据寄存器数量 | 1 个 | 课件 Slide 89 框图「规则组结果 ×1」；`stm32f10x_adc.h` 第 444 行 `ADC_GetConversionValue` 返回单个 `uint16_t` | 一致 |

> [!note] 出处说明
> 本页的**全部代码**（`7-1\Hardware\AD.c`、`AD.h`、`User\main.c`；`7-2\Hardware\AD.c`、`AD.h`、`User\main.c`）均**逐行照抄课程官方配套源码**，包括老师的中文注释，未做任何改写或「优化」；每个配置项、库函数名与枚举值都已回查 **ST 标准外设库 `stm32f10x_adc.h` / `stm32f10x_rcc.h` / `stm32f10x_gpio.h`** 并标出行号；引脚号与通道复用取自 **`引脚定义_xlsx原文.txt`**；原理性表述（校准要求、GPIO 模拟输入、规则组寄存器数量、转换时间公式）对照 **课程课件 Slide 86/89/97/98**。模块名称由 **官方接线图原图（8946 px 宽）** 放大确认。本页**未引用任何第三方笔记，也未使用 Gitee 镜像 `KSweb/stm32f1doc`**。
> **仍未核实**：「哪个传感器模块接哪个通道」官方材料里没有写明（已给出现场判别办法）、官方源码 `/ 4095` 是刻意选择还是笔误（课件无此公式，本页照抄源码原文）、以及老师的口头讲解原话。相关条目已留在上面的「待核对」。

---

**导航**：上一集 [[江协STM32 07-1 ADC模数转换器]] ｜ 下一章 [[江协STM32 08 DMA（章索引）]] ｜ 章索引 [[江协STM32 07 ADC（章索引）]]
