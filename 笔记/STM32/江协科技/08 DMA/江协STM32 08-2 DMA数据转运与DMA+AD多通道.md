---
course: 江协科技 STM32入门教程-2023版
chapter: 08-2 DMA数据转运与DMA+AD多通道
type: 分集笔记
source: https://www.bilibili.com/video/BV1th411z7sn?p=24
tags:
  - STM32
  - 江协科技
  - DMA
  - ADC
  - 多通道
  - 数据转运
status: draft
verify: 官方源码+接线图+课件
updated: 2026-09-26
---

# 08-2 DMA 数据转运与 DMA+AD 多通道

> [!abstract] 这一集把 DMA 用起来：两个实验
> **① DMA 数据转运**：把 SRAM 里的 `DataA[]` 整体搬到 `DataB[]`——**软件触发 + 存储器到存储器**，用来验证"搬运确实发生了、而且没有 CPU 参与"。
> **② DMA + AD 多通道**：让 ADC 扫描 4 个通道、DMA 自动把 4 个结果搬进 `AD_Value[]`——这就是 8-1 讲的"硬件触发 + 循环模式"的标准用法，也是第 7 章"手动切通道"的升级版。
>
> 上一集 [[江协STM32 08-1 DMA直接存储器存取]] ｜ 下一章 [[江协STM32 09 USART串口（章索引）]] ｜ 章索引 [[江协STM32 08 DMA（章索引）]]

## 1 两个实验的对照

| 项目 | 实验一 数据转运 | 实验二 DMA + AD 多通道 |
| --- | --- | --- |
| 源 | `DataA[]`（SRAM 数组） | `ADC1->DR`（外设数据寄存器） |
| 目的 | `DataB[]`（SRAM 数组） | `AD_Value[]`（SRAM 数组） |
| 触发方式 | **软件触发**（`DMA_M2M_Enable`） | **硬件触发**（ADC 转换完成） |
| 传输模式 | 正常模式 | 循环模式 |
| 数据宽度 | Byte（8 位） | HalfWord（16 位） |
| 通道 | `DMA1_Channel1` | `DMA1_Channel1` |
| 官方工程 | `8-1 DMA数据转运\` | `8-2 DMA+AD多通道\` |

两个实验用的是**同一个 DMA1 通道 1**，因为 ADC1 的硬件请求固定绑定在通道 1 上（8-1 讲的请求映射表），而实验一本来就是软件触发，用哪个通道都行——官方干脆也用了通道 1。

## 2 实验一：DMA 数据转运

### 2.1 现象与接线

接线非常简单：**只接 OLED 和 STLINK，没有任何传感器**。

| 连接 | 引脚 | 依据 |
| --- | --- | --- |
| OLED SCL | `PB8` | `Hardware\OLED.c` 第 5、16 行 |
| OLED SDA | `PB9` | `Hardware\OLED.c` 第 6、18 行 |
| STLINK | SWDIO / SWCLK / GND / 3.3V | 官方接线图 |

现象：OLED 第 1、3 行显示 `DataA`、`DataB` 两个数组的首地址（十六进制，8 位）；第 2、4 行显示两个数组的 4 个元素。`DataA` 每秒自增一次，`DataB` 一直是 `00 00 00 00`——直到 `MyDMA_Transfer()` 被调用，`DataB` 一瞬间变成和 `DataA` 一模一样。

> [!note] 官方接线图在接线相同时会复用同一张图
> `8-1 DMA数据转运.png` 与 `4-1 OLED显示屏.png`、`6-1 定时器定时中断.png` 是**同一张图**（SHA256 完全相同：`9E2250A2…9EBEEF`）。这是官方在"接线完全一致"时的正常复用，不是异常。

### 2.2 `MyDMA_Init()` 的每个成员在实际取值下的含义

`MyDMA_Init((uint32_t)DataA, (uint32_t)DataB, 4)`：

| 结构体成员 | 实际取值 | 为什么这么写 |
| --- | --- | --- |
| `DMA_PeripheralBaseAddr` | `AddrA`（即 `DataA` 的首地址） | 虽然叫"外设地址"，在 M2M 模式下它就是**源地址** |
| `DMA_PeripheralDataSize` | `DMA_PeripheralDataSize_Byte` | `DataA` 是 `uint8_t` 数组，一个元素 1 字节 |
| `DMA_PeripheralInc` | `DMA_PeripheralInc_Enable` | 源是数组，每搬一次地址要往后走 1 字节 |
| `DMA_MemoryBaseAddr` | `AddrB`（即 `DataB` 的首地址） | **目的地址** |
| `DMA_MemoryDataSize` | `DMA_MemoryDataSize_Byte` | 与源宽度对应 |
| `DMA_MemoryInc` | `DMA_MemoryInc_Enable` | 目的也是数组，同样要往后走 |
| `DMA_DIR` | `DMA_DIR_PeripheralSRC` | 方向"外设→存储器"，M2M 下照旧这么填（由 `M2M` 位决定谁自增） |
| `DMA_BufferSize` | `Size`（传进来是 4） | 转运次数 = 数组元素个数 |
| `DMA_Mode` | `DMA_Mode_Normal` | 搬完就停，不自动重装 |
| `DMA_M2M` | `DMA_M2M_Enable` | **这是实验一的关键**：没有任何外设会发请求，必须让 DMA 自己触发自己 |
| `DMA_Priority` | `DMA_Priority_Medium` | 只有一个通道在干活，优先级随便填 |
| 通道 | `DMA1_Channel1` | F103C8T6 只有 DMA1 的 7 个通道 |

### 2.3 `System\MyDMA.c`（官方原文）

```c
#include "stm32f10x.h"                  // Device header

uint16_t MyDMA_Size;					//定义全局变量，用于记住Init函数的Size，供Transfer函数使用

/**
  * 函    数：DMA初始化
  * 参    数：AddrA 原数组的首地址
  * 参    数：AddrB 目的数组的首地址
  * 参    数：Size 转运的数据大小（转运次数）
  * 返 回 值：无
  */
void MyDMA_Init(uint32_t AddrA, uint32_t AddrB, uint16_t Size)
{
	MyDMA_Size = Size;					//将Size写入到全局变量，记住参数Size
	
	/*开启时钟*/
	RCC_AHBPeriphClockCmd(RCC_AHBPeriph_DMA1, ENABLE);						//开启DMA的时钟
	
	/*DMA初始化*/
	DMA_InitTypeDef DMA_InitStructure;										//定义结构体变量
	DMA_InitStructure.DMA_PeripheralBaseAddr = AddrA;						//外设基地址，给定形参AddrA
	DMA_InitStructure.DMA_PeripheralDataSize = DMA_PeripheralDataSize_Byte;	//外设数据宽度，选择字节
	DMA_InitStructure.DMA_PeripheralInc = DMA_PeripheralInc_Enable;			//外设地址自增，选择使能
	DMA_InitStructure.DMA_MemoryBaseAddr = AddrB;							//存储器基地址，给定形参AddrB
	DMA_InitStructure.DMA_MemoryDataSize = DMA_MemoryDataSize_Byte;			//存储器数据宽度，选择字节
	DMA_InitStructure.DMA_MemoryInc = DMA_MemoryInc_Enable;					//存储器地址自增，选择使能
	DMA_InitStructure.DMA_DIR = DMA_DIR_PeripheralSRC;						//数据传输方向，选择由外设到存储器
	DMA_InitStructure.DMA_BufferSize = Size;								//转运的数据大小（转运次数）
	DMA_InitStructure.DMA_Mode = DMA_Mode_Normal;							//模式，选择正常模式
	DMA_InitStructure.DMA_M2M = DMA_M2M_Enable;								//存储器到存储器，选择使能
	DMA_InitStructure.DMA_Priority = DMA_Priority_Medium;					//优先级，选择中等
	DMA_Init(DMA1_Channel1, &DMA_InitStructure);							//将结构体变量交给DMA_Init，配置DMA1的通道1
	
	/*DMA使能*/
	DMA_Cmd(DMA1_Channel1, DISABLE);	//这里先不给使能，初始化后不会立刻工作，等后续调用Transfer后，再开始
}

/**
  * 函    数：启动DMA数据转运
  * 参    数：无
  * 返 回 值：无
  */
void MyDMA_Transfer(void)
{
	DMA_Cmd(DMA1_Channel1, DISABLE);					//DMA失能，在写入传输计数器之前，需要DMA暂停工作
	DMA_SetCurrDataCounter(DMA1_Channel1, MyDMA_Size);	//写入传输计数器，指定将要转运的次数
	DMA_Cmd(DMA1_Channel1, ENABLE);						//DMA使能，开始工作
	
	while (DMA_GetFlagStatus(DMA1_FLAG_TC1) == RESET);	//等待DMA工作完成
	DMA_ClearFlag(DMA1_FLAG_TC1);						//清除工作完成标志位
}
```

### 2.4 `System\MyDMA.h`（官方原文）

```c
#ifndef __MYDMA_H
#define __MYDMA_H

void MyDMA_Init(uint32_t AddrA, uint32_t AddrB, uint16_t Size);
void MyDMA_Transfer(void);

#endif
```

### 2.5 `User\main.c`（官方原文）

```c
#include "stm32f10x.h"                  // Device header
#include "Delay.h"
#include "OLED.h"
#include "MyDMA.h"

uint8_t DataA[] = {0x01, 0x02, 0x03, 0x04};				//定义测试数组DataA，为数据源
uint8_t DataB[] = {0, 0, 0, 0};							//定义测试数组DataB，为数据目的地

int main(void)
{
	/*模块初始化*/
	OLED_Init();				//OLED初始化
	
	MyDMA_Init((uint32_t)DataA, (uint32_t)DataB, 4);	//DMA初始化，把源数组和目的数组的地址传入
	
	/*显示静态字符串*/
	OLED_ShowString(1, 1, "DataA");
	OLED_ShowString(3, 1, "DataB");
	
	/*显示数组的首地址*/
	OLED_ShowHexNum(1, 8, (uint32_t)DataA, 8);
	OLED_ShowHexNum(3, 8, (uint32_t)DataB, 8);
		
	while (1)
	{
		DataA[0] ++;		//变换测试数据
		DataA[1] ++;
		DataA[2] ++;
		DataA[3] ++;
		
		OLED_ShowHexNum(2, 1, DataA[0], 2);		//显示数组DataA
		OLED_ShowHexNum(2, 4, DataA[1], 2);
		OLED_ShowHexNum(2, 7, DataA[2], 2);
		OLED_ShowHexNum(2, 10, DataA[3], 2);
		OLED_ShowHexNum(4, 1, DataB[0], 2);		//显示数组DataB
		OLED_ShowHexNum(4, 4, DataB[1], 2);
		OLED_ShowHexNum(4, 7, DataB[2], 2);
		OLED_ShowHexNum(4, 10, DataB[3], 2);
		
		Delay_ms(1000);		//延时1s，观察转运前的现象
		
		MyDMA_Transfer();	//使用DMA转运数组，从DataA转运到DataB
		
		OLED_ShowHexNum(2, 1, DataA[0], 2);		//显示数组DataA
		OLED_ShowHexNum(2, 4, DataA[1], 2);
		OLED_ShowHexNum(2, 7, DataA[2], 2);
		OLED_ShowHexNum(2, 10, DataA[3], 2);
		OLED_ShowHexNum(4, 1, DataB[0], 2);		//显示数组DataB
		OLED_ShowHexNum(4, 4, DataB[1], 2);
		OLED_ShowHexNum(4, 7, DataB[2], 2);
		OLED_ShowHexNum(4, 10, DataB[3], 2);

		Delay_ms(1000);		//延时1s，观察转运后的现象
	}
}
```

### 2.6 `MyDMA_Transfer()` 里三步各自为什么必要

| 语句 | 为什么 |
| --- | --- |
| `DMA_Cmd(DMA1_Channel1, DISABLE)` | 标准库对 `DMA_SetCurrDataCounter()` 的注释写明它**只能在通道关闭时使用**（`stm32f10x_dma.c` 第 350 行）；参考手册也说非循环模式下要重装计数器必须先关闭通道 |
| `DMA_SetCurrDataCounter(..., MyDMA_Size)` | 上一次搬完计数器已经减到 0，通道不再响应请求，必须把 `MyDMA_Size`（=4）重新装回去。这就是"把 `Size` 存进全局变量"的原因 |
| `DMA_Cmd(DMA1_Channel1, ENABLE)` | M2M 模式下 EN 置 1 立刻开搬（RM0008 第 277 页 "as soon as it is enabled"） |
| `while (DMA_GetFlagStatus(DMA1_FLAG_TC1) == RESET);` | 等传输完成标志 TC1。软件触发没有中断，只能轮询 |
| `DMA_ClearFlag(DMA1_FLAG_TC1)` | 清标志，否则下一次 `while` 会立刻通过、看起来像"没搬就返回" |

## 3 实验二：DMA + AD 多通道

### 3.1 为什么多通道 AD 必须用 DMA

ADC1 只有一个数据寄存器 `ADC1->DR`。扫描模式下一轮要转换 4 个通道，**每转完一个通道，`ADC1->DR` 就被新结果覆盖一次**。如果还像 7-2 那样"等 EOC 再读走"，就必须在两次转换之间及时把数据取走，否则前一通道的结果就被冲掉了。

DMA 正好干这件事：ADC 每转换完一个通道产生一次 DMA 请求，DMA 就把 `ADC1->DR` 的内容搬到数组的下一个位置。CPU 完全不用管，主循环随时读数组就行。课件 Slide 107「ADC 扫描模式 + DMA」画的就是这条链路：

```text
通道0..通道6 ──►[ 序列1..序列16 ]──► ADC_DR ──►(EOC/触发)──►[ DMA ]──► SRAM 数组 uint16_t ADValue[7];
                                          ▲                        ▲
                                     ADC 硬件触发 DMA          DMA 外设地址 → DMA 存储器地址
```

### 3.2 接线与引脚

| 引脚 | ADC 通道 | 接什么 | 依据 |
| --- | --- | --- | --- |
| `PA0` | `ADC_Channel_0` | 电位器 | 官方接线图 `8-2 DMA+AD多通道.png` 连线追踪 |
| `PA1` | `ADC_Channel_1` | 光敏传感器模块（AO） | 同上 |
| `PA2` | `ADC_Channel_2` | 热敏传感器模块（AO） | 同上 |
| `PA3` | `ADC_Channel_3` | 反射式红外传感器模块（AO） | 同上 |
| `PB8` / `PB9` | — | OLED SCL / SDA | `Hardware\OLED.c` 第 5~6 行 |

引脚定义表里这四脚都带 `ADC12_INx` 复用：`PA0` = `ADC12_IN0`、`PA1` = `ADC12_IN1`、`PA2` = `ADC12_IN2`、`PA3` = `ADC12_IN3`（`引脚定义_xlsx原文.txt` 第 12~15 行）。

> [!note] 接线图是复用图，不是异常
> `8-2 DMA+AD多通道.png` 与 `7-2 AD多通道.png` 是**同一张图**（SHA256 完全相同：`E541A84A…C6978A`）。两个实验的硬件接线本来就一模一样，官方直接复用了同一张图。

### 3.3 `Hardware\AD.c`（官方原文）

```c
#include "stm32f10x.h"                  // Device header

uint16_t AD_Value[4];					//定义用于存放AD转换结果的全局数组

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
	RCC_AHBPeriphClockCmd(RCC_AHBPeriph_DMA1, ENABLE);		//开启DMA1的时钟
	
	/*设置ADC时钟*/
	RCC_ADCCLKConfig(RCC_PCLK2_Div6);						//选择时钟6分频，ADCCLK = 72MHz / 6 = 12MHz
	
	/*GPIO初始化*/
	GPIO_InitTypeDef GPIO_InitStructure;
	GPIO_InitStructure.GPIO_Mode = GPIO_Mode_AIN;
	GPIO_InitStructure.GPIO_Pin = GPIO_Pin_0 | GPIO_Pin_1 | GPIO_Pin_2 | GPIO_Pin_3;
	GPIO_InitStructure.GPIO_Speed = GPIO_Speed_50MHz;
	GPIO_Init(GPIOA, &GPIO_InitStructure);					//将PA0、PA1、PA2和PA3引脚初始化为模拟输入
	
	/*规则组通道配置*/
	ADC_RegularChannelConfig(ADC1, ADC_Channel_0, 1, ADC_SampleTime_55Cycles5);	//规则组序列1的位置，配置为通道0
	ADC_RegularChannelConfig(ADC1, ADC_Channel_1, 2, ADC_SampleTime_55Cycles5);	//规则组序列2的位置，配置为通道1
	ADC_RegularChannelConfig(ADC1, ADC_Channel_2, 3, ADC_SampleTime_55Cycles5);	//规则组序列3的位置，配置为通道2
	ADC_RegularChannelConfig(ADC1, ADC_Channel_3, 4, ADC_SampleTime_55Cycles5);	//规则组序列4的位置，配置为通道3
	
	/*ADC初始化*/
	ADC_InitTypeDef ADC_InitStructure;											//定义结构体变量
	ADC_InitStructure.ADC_Mode = ADC_Mode_Independent;							//模式，选择独立模式，即单独使用ADC1
	ADC_InitStructure.ADC_DataAlign = ADC_DataAlign_Right;						//数据对齐，选择右对齐
	ADC_InitStructure.ADC_ExternalTrigConv = ADC_ExternalTrigConv_None;			//外部触发，使用软件触发，不需要外部触发
	ADC_InitStructure.ADC_ContinuousConvMode = ENABLE;							//连续转换，使能，每转换一次规则组序列后立刻开始下一次转换
	ADC_InitStructure.ADC_ScanConvMode = ENABLE;								//扫描模式，使能，扫描规则组的序列，扫描数量由ADC_NbrOfChannel确定
	ADC_InitStructure.ADC_NbrOfChannel = 4;										//通道数，为4，扫描规则组的前4个通道
	ADC_Init(ADC1, &ADC_InitStructure);											//将结构体变量交给ADC_Init，配置ADC1
	
	/*DMA初始化*/
	DMA_InitTypeDef DMA_InitStructure;											//定义结构体变量
	DMA_InitStructure.DMA_PeripheralBaseAddr = (uint32_t)&ADC1->DR;				//外设基地址，给定形参AddrA
	DMA_InitStructure.DMA_PeripheralDataSize = DMA_PeripheralDataSize_HalfWord;	//外设数据宽度，选择半字，对应16为的ADC数据寄存器
	DMA_InitStructure.DMA_PeripheralInc = DMA_PeripheralInc_Disable;			//外设地址自增，选择失能，始终以ADC数据寄存器为源
	DMA_InitStructure.DMA_MemoryBaseAddr = (uint32_t)AD_Value;					//存储器基地址，给定存放AD转换结果的全局数组AD_Value
	DMA_InitStructure.DMA_MemoryDataSize = DMA_MemoryDataSize_HalfWord;			//存储器数据宽度，选择半字，与源数据宽度对应
	DMA_InitStructure.DMA_MemoryInc = DMA_MemoryInc_Enable;						//存储器地址自增，选择使能，每次转运后，数组移到下一个位置
	DMA_InitStructure.DMA_DIR = DMA_DIR_PeripheralSRC;							//数据传输方向，选择由外设到存储器，ADC数据寄存器转到数组
	DMA_InitStructure.DMA_BufferSize = 4;										//转运的数据大小（转运次数），与ADC通道数一致
	DMA_InitStructure.DMA_Mode = DMA_Mode_Circular;								//模式，选择循环模式，与ADC的连续转换一致
	DMA_InitStructure.DMA_M2M = DMA_M2M_Disable;								//存储器到存储器，选择失能，数据由ADC外设触发转运到存储器
	DMA_InitStructure.DMA_Priority = DMA_Priority_Medium;						//优先级，选择中等
	DMA_Init(DMA1_Channel1, &DMA_InitStructure);								//将结构体变量交给DMA_Init，配置DMA1的通道1
	
	/*DMA和ADC使能*/
	DMA_Cmd(DMA1_Channel1, ENABLE);							//DMA1的通道1使能
	ADC_DMACmd(ADC1, ENABLE);								//ADC1触发DMA1的信号使能
	ADC_Cmd(ADC1, ENABLE);									//ADC1使能
	
	/*ADC校准*/
	ADC_ResetCalibration(ADC1);								//固定流程，内部有电路会自动执行校准
	while (ADC_GetResetCalibrationStatus(ADC1) == SET);
	ADC_StartCalibration(ADC1);
	while (ADC_GetCalibrationStatus(ADC1) == SET);
	
	/*ADC触发*/
	ADC_SoftwareStartConvCmd(ADC1, ENABLE);	//软件触发ADC开始工作，由于ADC处于连续转换模式，故触发一次后ADC就可以一直连续不断地工作
}
```

### 3.4 `ADC_InitStructure` 每个成员的含义

| 成员 | 取值 | 含义 |
| --- | --- | --- |
| `ADC_Mode` | `ADC_Mode_Independent` | 独立模式，只用 ADC1 |
| `ADC_DataAlign` | `ADC_DataAlign_Right` | 转换结果右对齐，所以低 12 位就是 0~4095 |
| `ADC_ExternalTrigConv` | `ADC_ExternalTrigConv_None` | 不用外部触发（这里指不用定时器等硬件触发 ADC），改由软件触发 |
| `ADC_ContinuousConvMode` | `ENABLE` | 连续转换：扫完一轮规则组序列后立刻开始下一轮 |
| `ADC_ScanConvMode` | `ENABLE` | 扫描模式：按序列 1→2→3→4 依次转换 |
| `ADC_NbrOfChannel` | `4` | 只扫规则组前 4 个位置 |

规则组序列用 `ADC_RegularChannelConfig(ADC1, 通道, 序列位置, 采样时间)` 一次配好 4 个位置——这是和 7-2 最大的写法差别（7-2 每次转换前改序列 1，见第 4 节）。

### 3.5 `DMA_InitStructure` 每个成员的含义

| 成员 | 取值 | 含义 |
| --- | --- | --- |
| `DMA_PeripheralBaseAddr` | `(uint32_t)&ADC1->DR` | 源固定为 ADC1 数据寄存器 |
| `DMA_PeripheralDataSize` | `DMA_PeripheralDataSize_HalfWord` | ADC 结果 12 位、右对齐，按 16 位（半字）取 |
| `DMA_PeripheralInc` | `DMA_PeripheralInc_Disable` | **源地址不自增**：每次都从同一个 `ADC1->DR` 读 |
| `DMA_MemoryBaseAddr` | `(uint32_t)AD_Value` | 目的：存放结果的全局数组 |
| `DMA_MemoryDataSize` | `DMA_MemoryDataSize_HalfWord` | 与源宽度一致，避免对齐转换 |
| `DMA_MemoryInc` | `DMA_MemoryInc_Enable` | **目的地址自增**：每搬一次数组往后挪一个 `uint16_t` |
| `DMA_DIR` | `DMA_DIR_PeripheralSRC` | 外设是源：`ADC1->DR` → `AD_Value[]` |
| `DMA_BufferSize` | `4` | 转运次数 = ADC 通道数，正好装一轮扫描 |
| `DMA_Mode` | `DMA_Mode_Circular` | 循环模式：搬完 4 次自动重装计数器，下一轮接着来 |
| `DMA_M2M` | `DMA_M2M_Disable` | 不能开：数据由 ADC 硬件请求触发（而且 MEM2MEM 与循环模式不能同时用） |
| `DMA_Priority` | `DMA_Priority_Medium` | 只有一个通道，中等优先级即可 |

> [!warning] `DMA_M2M` 必须失能
> 这里既要用**循环模式**，又要由 **ADC 硬件请求**触发，两条都要求 `DMA_M2M_Disable`：
> 一是参考手册明确 M2M 不能和循环模式同时使用；二是开了 M2M 通道一使能就自己开搬，根本不等 ADC。

### 3.6 使能顺序

```text
DMA_Cmd(DMA1_Channel1, ENABLE)   ① DMA 通道先就绪
ADC_DMACmd(ADC1, ENABLE)         ② 再打开 ADC 向 DMA 发请求的开关
ADC_Cmd(ADC1, ENABLE)            ③ 最后开 ADC
ADC_ResetCalibration / 校准       ④ 固定校准流程
ADC_SoftwareStartConvCmd         ⑤ 软件触发一次，之后连续不断
```

**顺序不能反**：先开 ADC 再开 DMA，ADC 可能已经转换完成甚至覆盖了 `ADC1->DR`，第一次的结果就丢了。而且 `ADC_DMACmd()` 打开的是"外设自己的 DMA 请求位"——8-1 讲过，这个开关不开，DMA 配得再对也等不到请求。

> [!tip] 校准为什么必须在 `ADC_Cmd(ENABLE)` 之后
> 参考手册 §11.4 校准（RM0008 第 222 页）原文：**Before starting a calibration, the ADC must have been in power-on state (ADON bit = '1') for at least two ADC clock cycles.**
> 也就是说，开始校准前 ADON 必须已经是 1 并且保持至少两个 ADC 时钟周期。官方代码先 `ADC_Cmd(ADC1, ENABLE)`（置 ADON）再 `ADC_ResetCalibration()`，顺序正好符合这条要求。
>
> 课件 Slide 98 把这句中文写成「启动校准前，ADC 必须处于**关电**状态超过至少两个 ADC 时钟周期」——按参考手册英文原文应为「**上电**状态（ADON = 1）」。**以参考手册为准，官方源码的顺序是对的。**（中文版参考手册同一句的措辞本页未核对，见「待核对」。）

### 3.7 `Hardware\AD.h` 与 `User\main.c`（官方原文）

```c
#ifndef __AD_H
#define __AD_H

extern uint16_t AD_Value[4];

void AD_Init(void);

#endif
```

```c
#include "stm32f10x.h"                  // Device header
#include "Delay.h"
#include "OLED.h"
#include "AD.h"

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
		OLED_ShowNum(1, 5, AD_Value[0], 4);		//显示转换结果第0个数据
		OLED_ShowNum(2, 5, AD_Value[1], 4);		//显示转换结果第1个数据
		OLED_ShowNum(3, 5, AD_Value[2], 4);		//显示转换结果第2个数据
		OLED_ShowNum(4, 5, AD_Value[3], 4);		//显示转换结果第3个数据
		
		Delay_ms(100);							//延时100ms，手动增加一些转换的间隔时间
	}
}
```

现象：OLED 四行 `AD0:`~`AD3:` 显示 0~4095 的数字，转动电位器 / 遮挡光敏、热敏、红外传感器时对应行跟着变。主循环里**没有任何"取数据"的代码**，只是读数组——数据是 DMA 搬进来的。

## 4 与第 7 章的衔接

7-2 也是多通道，但走的是"手动切通道"的路子。两集对同一个硬件做同一件事，差别全在配置上：

| 对比项 | 7-2 AD多通道 | 8-2 DMA+AD多通道 |
| --- | --- | --- |
| `ADC_ContinuousConvMode` | `DISABLE` | `ENABLE` |
| `ADC_ScanConvMode` | `DISABLE` | `ENABLE` |
| `ADC_NbrOfChannel` | `1` | `4` |
| 规则组序列 | 每次转换前用 `ADC_RegularChannelConfig` 改序列 1 | 初始化时一次配好序列 1~4 |
| 取结果 | `while` 等 EOC 后 `ADC_GetConversionValue` | DMA 自动搬进 `AD_Value[]`，主循环直接读数组 |
| 转换节奏 | 软件触发一次转一次 | 软件触发一次后连续不断转换 |
| 与 DMA 相关 | 完全不涉及 | `ADC_DMACmd` + `DMA_Cmd` + 循环模式 |

证据：`7-2 AD多通道\Hardware\AD.c` 第 31~33 行（连续/扫描失能、通道数 1）与第 53~56 行（每次改序列、软件触发、等 EOC）；`8-2 DMA+AD多通道\Hardware\AD.c` 第 28~31 行（4 个序列一次配好）与第 38~40 行（连续/扫描使能、通道数 4）。

一句话：**第 7 章解决"怎么把 4 个通道都读到"，第 8 章解决"怎么让 CPU 不参与地读"。**

## 5 易错点

- [ ] DMA 的时钟忘了开，或者开错总线：DMA 挂 **AHB**，用 `RCC_AHBPeriphClockCmd(RCC_AHBPeriph_DMA1, ENABLE)`，不是 `RCC_APB1PeriphClockCmd`。
- [ ] 实验二忘了 `ADC_DMACmd(ADC1, ENABLE)` → DMA 配得再对也一次都不搬，`AD_Value[]` 永远是 0。
- [ ] 实验二 `DMA_PeripheralInc` 没设成 `Disable` → 源地址开始乱走，读到的是别的寄存器，直接传输错误。
- [ ] 实验二 `DMA_MemoryInc` 没设成 `Enable` → 4 个通道结果全被覆盖到 `AD_Value[0]`，屏幕上另外三行永远不变。
- [ ] 实验二 `DMA_PeripheralDataSize` / `DMA_MemoryDataSize` 写成 `Byte` → 16 位的 ADC 结果被拆成两个字节搬，数组里的数全乱。
- [ ] `DMA_BufferSize` 与 `ADC_NbrOfChannel` 不一致（例如通道数 4、BufferSize 3）→ 每一轮扫描都会错位。
- [ ] 实验二把 `DMA_M2M` 打开 → 既违反"M2M 不能与循环模式同用"，又会让通道一使能就自己乱搬。
- [ ] 实验二用正常模式 → 搬完一轮后计数器为 0、通道不再响应请求，`AD_Value[]` 从此冻结。
- [ ] 实验一忘了 `DMA_M2M_Enable` → 没有外设发请求，`while` 死等 `TC1`，程序卡死。
- [ ] 实验一直接调用 `MyDMA_Transfer()` 之前没做初始化 → 通道根本没配；或者初始化时就把通道使能了，数据在 `while(1)` 之前已经搬完一次。
- [ ] 实验一 `while (DMA_GetFlagStatus(DMA1_FLAG_TC1) == RESET);` 之后忘记 `DMA_ClearFlag()` → 下一次判断直接通过，看起来像"没搬就返回"。
- [ ] OLED 显示 AD 值时第 4 个参数（长度）写成 2 或 3 → 0~4095 最多 4 位，长度不够高位被截掉。

## 6 自测

1. 8-1 里 `MyDMA_Init((uint32_t)DataA, (uint32_t)DataB, 4)` 的三个实参分别对应 DMA 的哪三个要素？那个 `4` 是什么含义？
2. 8-1 的实验为什么必须把 `DMA_M2M` 设成 `Enable`？
3. `MyDMA_Transfer()` 里"先 `DISABLE` → 写计数器 → 再 `ENABLE`"，这三步各自为什么必要？后面的 `while` 在等什么？
4. 8-2 里为什么 `DMA_PeripheralInc` 要 `Disable`，而 `DMA_MemoryInc` 要 `Enable`？
5. 8-2 里如果把 `DMA_Mode` 改成 `DMA_Mode_Normal`，会出现什么现象？为什么？

> [!success]- 参考答案
> 1. `(uint32_t)DataA` → `DMA_PeripheralBaseAddr`（M2M 下的源地址）；`(uint32_t)DataB` → `DMA_MemoryBaseAddr`（目的地址）；`4` → `DMA_BufferSize`，即**转运次数（数据个数）**，正好等于 `DataA` / `DataB` 的元素个数。
> 2. 因为这是纯粹的 SRAM → SRAM 搬运，**没有任何外设会发 DMA 请求**。只有 `MEM2MEM` 位为 1 时，通道才会在 EN 置 1 后由软件触发自己开搬（RM0008 第 277 页）。
> 3. `DISABLE`：标准库要求写传输计数器时通道必须处于关闭状态（`stm32f10x_dma.c` 第 350 行）；写计数器：上一轮搬完后计数器已经减到 0，必须重新装回 4；`ENABLE`：M2M 模式下 EN 置 1 立刻开搬。后面的 `while` 等的是 `DMA1_FLAG_TC1`（传输完成标志），等到了再 `DMA_ClearFlag(DMA1_FLAG_TC1)` 清掉。
> 4. 因为 4 次搬运的**源永远是同一个寄存器 `ADC1->DR`**（地址不能变，否则第二次就读到别的寄存器了），而**目的是 `AD_Value[4]` 数组**，每搬一次必须往后挪一个 `uint16_t`，否则 4 个通道的结果会全部覆盖到 `AD_Value[0]`。
> 5. 现象：`AD_Value[]` 只更新一轮（4 次）就再也不变了，OLED 上四个数字**冻结**。原因：正常模式下计数器减到 0 后通道不再响应任何请求（RM0008 第 277 页），而 ADC 还在连续转换——DMA 不接活了。要恢复必须自己重装计数器。这也正是官方用循环模式"与 ADC 的连续转换一致"的原因。

## 7 待核对

- [ ] 视频里老师对两个实验现象的口头讲解（按键/延时节奏、`DataB` 变化瞬间的画面细节）——不在源码、接线图与课件文本里。
- [ ] 中文版参考手册里 ADC 校准那句的确切措辞：本页按**英文版** RM0008 §11.4（第 222 页）"in power-on state (ADON bit = '1')" 核定，并据此判断课件 Slide 98 的中文「关电状态」为表述有误；中文版参考手册同一句的原文未核对。
- [ ] 8-2 四个传感器模块与 `PA0`~`PA3` 的一一对应（电位器→PA0、光敏→PA1、热敏→PA2、反射式红外→PA3）来自对官方接线图的连线追踪；官方源码只写了 `PA0`~`PA3` 配成模拟输入并对应通道 0~3，**未写传感器名称**。

## 核对记录（2026-09-26）

> [!success] 核对依据
> - 官方配套源码：`STM32Project-有注释版\8-1 DMA数据转运\System\MyDMA.c`、`System\MyDMA.h`、`User\main.c`、`Hardware\OLED.c`；`8-2 DMA+AD多通道\Hardware\AD.c`、`Hardware\AD.h`、`User\main.c`；对照工程 `7-2 AD多通道\Hardware\AD.c`、`User\main.c`
> - 官方接线图：`ground-truth\接线图\8-1 DMA数据转运.png`、`8-2 DMA+AD多通道.png`、`7-2 AD多通道.png`
> - 课程课件文本：`ground-truth\课件文本.md`（Slide 98 ADC 校准、Slide 106 数据转运+DMA、Slide 107 ADC 扫描模式+DMA）
> - ST 官方参考手册：`STM32F10xxx参考手册（英文）.pdf` §11.4 校准（第 222 页）、§13.3.3（第 277~278 页）
> - ST 标准外设库 V3.5.0：`...\inc\stm32f10x_dma.h`、`...\src\stm32f10x_dma.c`、`...\inc\stm32f10x_adc.h`、`...\inc\stm32f10x_rcc.h`
> - 引脚定义表：`ground-truth\引脚定义_xlsx原文.txt` 第 12~15 行

| 核对项 | 笔记值 | 官方依据（文件名+行号） | 结论 |
| --- | --- | --- | --- |
| 8-1 用哪个通道 | `DMA1_Channel1` | `8-1 ...\System\MyDMA.c` 第 32、35、45~50 行 | 一致 |
| 8-1 源 / 目的地址 | `DataA` / `DataB` 的首地址 | `User\main.c` 第 6~7 行定义、第 14 行传参；`MyDMA.c` 第 21、24 行 | 一致 |
| 8-1 转运次数 | 4 | `main.c` 第 6~7 行数组各 4 个元素；第 14 行实参 4；`MyDMA.c` 第 28 行 `DMA_BufferSize = Size` | 一致 |
| 8-1 数据宽度 | 外设、存储器都是 `Byte` | `MyDMA.c` 第 22、25 行 | 一致 |
| 8-1 地址自增 | 外设、存储器都 `Enable` | `MyDMA.c` 第 23、26 行 | 一致 |
| 8-1 方向 | `DMA_DIR_PeripheralSRC` | `MyDMA.c` 第 27 行 | 一致 |
| 8-1 模式 | `DMA_Mode_Normal` | `MyDMA.c` 第 29 行 | 一致 |
| 8-1 M2M | `DMA_M2M_Enable` | `MyDMA.c` 第 30 行 | 一致 |
| 8-1 优先级 | `DMA_Priority_Medium` | `MyDMA.c` 第 31 行 | 一致 |
| 8-1 DMA 时钟总线 | AHB（`RCC_AHBPeriph_DMA1`） | `MyDMA.c` 第 17 行；`stm32f10x_rcc.h` 第 470 行 | 一致 |
| 8-1 初始化时不使能 | 是（`DMA_Cmd(..., DISABLE)`） | `MyDMA.c` 第 35 行及该行注释 | 一致 |
| 8-1 转运三步 | DISABLE → `DMA_SetCurrDataCounter` → ENABLE | `MyDMA.c` 第 45~47 行 | 一致 |
| `DMA_SetCurrDataCounter` 的前提 | 通道必须关闭 | `stm32f10x_dma.c` 第 350 行注释 | 一致 |
| 8-1 等待完成方式 | 轮询 `DMA1_FLAG_TC1` 后清标志 | `MyDMA.c` 第 49~50 行 | 一致 |
| M2M 模式下 EN 置 1 立即开搬 | 是 | RM0008 §13.3.3（第 277 页） | 一致 |
| M2M 与循环模式不能同用 | 是 | RM0008 §13.3.3（第 278 页）；`stm32f10x_dma.h` 第 77~78 行 | 一致 |
| 8-1 数组定义在函数外 | 是（全局） | `User\main.c` 第 6~7 行 | 一致 |
| 8-1 OLED 显示内容 | `DataA` / `DataB` 的首地址 + 4 个元素 | `User\main.c` 第 17~22、31~38、44~51 行 | 一致 |
| 8-1 接线 | 只有 OLED（`PB8`/`PB9`）+ STLINK | 官方接线图 `8-1 DMA数据转运.png`；`Hardware\OLED.c` 第 5~6 行 | 一致 |
| 8-1 接线图复用 | 与 `4-1 OLED显示屏`、`6-1 定时器定时中断` 同一张图 | SHA256：`9E2250A2668C831077D0D1248736E2FD68C0A994813EE72794EBA769296EBEEF` 三者相同 | 一致（官方正常复用） |
| 8-2 用哪个通道 | `DMA1_Channel1` | `8-2 ...\Hardware\AD.c` 第 56、59 行 | 一致（ADC1 的请求固定绑通道 1） |
| 8-2 外设地址 | `(uint32_t)&ADC1->DR` | `Hardware\AD.c` 第 45 行 | 一致 |
| 8-2 外设自增 | `Disable` | `AD.c` 第 47 行 | 一致 |
| 8-2 数据宽度 | 外设、存储器都 `HalfWord` | `AD.c` 第 46、49 行 | 一致 |
| 8-2 存储器地址 | `(uint32_t)AD_Value` | `AD.c` 第 3 行定义、第 48 行传参 | 一致 |
| 8-2 存储器自增 | `Enable` | `AD.c` 第 50 行 | 一致 |
| 8-2 方向 | `DMA_DIR_PeripheralSRC` | `AD.c` 第 51 行 | 一致 |
| 8-2 `DMA_BufferSize` | 4（与 ADC 通道数一致） | `AD.c` 第 52 行、第 40 行 `ADC_NbrOfChannel = 4` | 一致 |
| 8-2 模式 | `DMA_Mode_Circular` | `AD.c` 第 53 行 | 一致 |
| 8-2 M2M | `DMA_M2M_Disable` | `AD.c` 第 54 行 | 一致 |
| 8-2 优先级 | `DMA_Priority_Medium` | `AD.c` 第 55 行 | 一致 |
| 8-2 ADC 连续转换 / 扫描模式 | 都 `ENABLE` | `AD.c` 第 38~39 行 | 一致 |
| 8-2 规则组序列 | 通道 0~3 占序列 1~4，采样时间 55.5 周期 | `AD.c` 第 28~31 行 | 一致 |
| ADC 通道枚举 | `ADC_Channel_0` ~ `ADC_Channel_3` | `stm32f10x_adc.h` 第 174~177 行 | 一致 |
| 采样时间枚举 | `ADC_SampleTime_55Cycles5` | `stm32f10x_adc.h` 第 218 行 | 一致 |
| `ADC_DMACmd` 存在且用法正确 | `ADC_DMACmd(ADC1, ENABLE)` | `stm32f10x_adc.h` 第 432 行；`AD.c` 第 60 行 | 一致 |
| 8-2 使能顺序 | DMA_Cmd → ADC_DMACmd → ADC_Cmd → 校准 → 软件触发 | `AD.c` 第 59~70 行，顺序相同 | 一致 |
| 8-2 外设 DMA 请求要单独开 | 是 | RM0008 §13.3.7（第 280 页）；`AD.c` 第 60 行 | 一致 |
| 8-2 引脚 | `PA0`~`PA3` 模拟输入 | `AD.c` 第 22~25 行 | 一致 |
| 8-2 引脚复用 | `PA0`~`PA3` = `ADC12_IN0`~`ADC12_IN3` | `引脚定义_xlsx原文.txt` 第 12~15 行 | 一致 |
| 8-2 传感器与引脚的对应 | 电位器→`PA0`、光敏→`PA1`、热敏→`PA2`、反射式红外→`PA3` | 官方接线图 `8-2 DMA+AD多通道.png` 连线逐段追踪（黄/绿线落点分别为 `PA3`/`PA2`/`PA1`/`PA0` 同一列） | 一致（源码未写传感器名称，已记入「待核对」） |
| 8-2 接线图复用 | 与 `7-2 AD多通道` 同一张图 | SHA256：`E541A84A2713F420352E3CA268A49F2176A6905207E16B1FD3FD7E9524C6978A` 两者相同 | 一致（官方正常复用） |
| 8-2 主循环 | 只读 `AD_Value[0..3]` 显示，无取数据代码 | `User\main.c` 第 20~25 行 | 一致 |
| 7-2 与 8-2 的差别 | 连续/扫描失能、通道数 1、每次改序列 | `7-2 AD多通道\Hardware\AD.c` 第 31~33、53~56 行 | 一致 |
| OLED 引脚 | `PB8` = SCL、`PB9` = SDA | `Hardware\OLED.c` 第 5~6、16、18 行 | 一致 |
| 8-2 校准的前提条件 | 开始校准前 ADON 必须已为 1 并保持至少两个 ADC 时钟周期 | RM0008 §11.4（第 222 页）"Before starting a calibration, the ADC must have been in power-on state (ADON bit = '1') for at least two ADC clock cycles." | 一致（官方代码先 `ADC_Cmd` 再校准，顺序正确） |
| 课件 Slide 98 的校准中文表述 | 课件写「必须处于**关电**状态」 | 同上英文原文为 "power-on state"（**上电**状态） | 已修正（本页按参考手册写「上电状态」，并注明课件中文表述有误） |
| 课件 Slide 107 的数组名 | 课件示意图写 `uint16_t ADValue[7]`，源码是 `uint16_t AD_Value[4]` | 课件 Slide 107；`Hardware\AD.c` 第 3 行 | 以源码为准（示意图仅为原理示意） |

> [!note] 出处说明
> 本页两个实验的代码、参数、引脚与现象，已逐条对照课程官方配套源码（`8-1 DMA数据转运\`、`8-2 DMA+AD多通道\` 的 `User\main.c`、`System\MyDMA.c/.h`、`Hardware\AD.c/.h`、`Hardware\OLED.c`）、官方接线图（并已用 SHA256 确认 `8-1` 与 `4-1`/`6-1`、`8-2` 与 `7-2` 为同一张图）与课程课件（Slide 98、106、107）核对；库函数名、枚举值与结构体成员名已在 ST 标准外设库 V3.5.0 的 `stm32f10x_dma.h`、`stm32f10x_adc.h`、`stm32f10x_rcc.h` 中逐个查到。
> **仍未核实**：老师对两个实验现象的口头讲解、中文版参考手册中 ADC 校准那句的确切措辞、以及四个传感器模块与引脚的一一对应（源码未写传感器名）——已留在上面的「待核对」。

