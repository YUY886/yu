---
course: 江协科技 STM32入门教程-2023版
chapter: 11-4 SPI通信外设
type: 分集笔记
source: https://www.bilibili.com/video/BV1th411z7sn?p=39
tags:
  - STM32
  - 江协科技
  - SPI
  - SPI外设
  - 硬件SPI
  - 状态标志
status: draft
verify: 官方源码+接线图+课件
updated: 2026-09-26
---

# 11-4 SPI 通信外设

> [!abstract] 这一集只解决一个问题
> **11-3 用 GPIO 一位一位翻出来的 SPI 时序，STM32 内部其实有一整套硬件电路替你干——它长什么样、要怎么配？**
>
> 答：SPI 外设内部是「**发送缓冲区 → 移位寄存器 → 接收缓冲区**」三件套，加一个**波特率发生器**和一个**主控制电路**。CPU 只需要做两件事：**写一次发送寄存器**、**等标志位**。配置入口是一个九成员的 `SPI_InitTypeDef`。
>
> 上一集 [[江协STM32 11-3 软件SPI读写W25Q64]] ｜ 下一集 [[江协STM32 11-5 硬件SPI读写W25Q64]] ｜ 章索引 [[江协STM32 11 SPI（章索引）]]

## 1 软件 SPI 的代价

11-3 的 `MySPI_SwapByte()` 里，真正搬数据的是这个循环（官方 `11-1 软件SPI读写W25Q64\Hardware\MySPI.c` 第 108~116 行原文）：

```c
	for (i = 0; i < 8; i ++)						//循环8次，依次交换每一位数据
	{
		/*两个!可以对数据进行两次逻辑取反，作用是把非0值统一转换为1，即：!!(0) = 0，!!(非0) = 1*/
		MySPI_W_MOSI(!!(ByteSend & (0x80 >> i)));	//使用掩码的方式取出ByteSend的指定一位数据并写入到MOSI线
		MySPI_W_SCK(1);								//拉高SCK，上升沿移出数据
		if (MySPI_R_MISO()){ByteReceive |= (0x80 >> i);}	//读取MISO数据，并存储到Byte变量
															//当MISO为1时，置变量指定位为1，当MISO为0时，不做处理，指定位为默认的初值0
		MySPI_W_SCK(0);								//拉低SCK，下降沿移入数据
	}
```

数一数 CPU 的动作：

| 每轮循环里的操作 | 次数 |
| --- | --- |
| 写 MOSI（`MySPI_W_MOSI`） | 1 |
| 写 SCK 拉高（`MySPI_W_SCK(1)`） | 1 |
| 读 MISO（`MySPI_R_MISO`） | 1 |
| 写 SCK 拉低（`MySPI_W_SCK(0)`） | 1 |
| **合计** | **4 次 GPIO 操作 / 轮** |

也就是说：**一个字节 = 8 轮循环 = 32 次 GPIO 操作**，而且每一位的时序由 CPU 亲手掐。速率上不去、CPU 被占着，任何一次中断插进来都可能把时序拉长。

课件 Slide 161 第一句就是答案：

> STM32 内部集成了硬件 SPI 收发电路，可以**由硬件自动执行时钟生成、数据收发等功能，减轻 CPU 的负担**。

**这就是本集要讲的那套电路。** 11-5 会把 11-3 的 `MySPI.c` 换成它的驱动版本，而 `W25Q64.c` 一个字都不用改。

## 2 SPI 外设简介（课件 Slide 161 原文）

| 特性 | 原文表述 |
| --- | --- |
| 定位 | 内部集成的**硬件 SPI 收发电路**，硬件自动执行**时钟生成、数据收发**，减轻 CPU 负担 |
| 数据帧 | 可配置 **8 位 / 16 位**数据帧、**高位先行 / 低位先行** |
| 时钟频率 | `f_PCLK / (2, 4, 8, 16, 32, 64, 128, 256)` |
| 主从 | 支持**多主机模型**、**主或从**操作 |
| 单工 / 半双工 | 可精简为**半双工 / 单工**通信 |
| DMA | 支持 DMA |
| I2S | 兼容 I2S 协议 |
| STM32F103C8T6 的资源 | **SPI1、SPI2** |

> [!note] 分频系数为什么只有 8 档
> `f_PCLK / (2, 4, 8, 16, 32, 64, 128, 256)` 恰好是 2 的 1~8 次幂，对应 `SPI_CR1` 里 3 个位 `BR[2:0]` 的 8 种取值——框图（下一节）里能看到 `BR[2:0]` 就是从 `SPI_CR1` 接到**波特率发生器**的那三根线。

## 3 SPI 外设内部结构

### 3.1 框图

课件 Slide 162 是一张纯图页，图上标题写的是「**图209 SPI框图**」（图内右下角另有编号 `ai14744`）。图上能直接读出的部件与连线：

| 部件 | 框图上画的关系 |
| --- | --- |
| **发送缓冲区**（TDR） | 从「地址和数据总线」**写入**；输出送往移位寄存器 |
| **移位寄存器** | 一头接发送缓冲区，一头接接收缓冲区；再经 GPIO 开关控制连到 **MOSI / MISO**；旁边标着 **LSBFIRST 控制位** |
| **接收缓冲区**（RDR） | 从移位寄存器收，向「地址和数据总线」**读出** |
| **波特率发生器** | 输入是 `BR[2:0]`（来自 `SPI_CR1`），输出是 **SCK** |
| **主控制电路** | 接 **NSS** 引脚，并与通信电路相连 |
| **通信电路** | 受 `SPI_CR1` / `SPI_CR2` 各控制位控制，同时驱动 `SPI_SR` |
| **SPI_CR1** | `LSBFIRST`、`SPE`、`BR2`、`BR1`、`BR0`、`MSTR`、`CPOL`、`CPHA`；另有 `BIDIMODE`、`BIDIOE`、`CRCEN`、`CRCNEXT`、`DFF`、`RXONLY`、`SSM`、`SSI` |
| **SPI_CR2** | `TXEIE`、`RXNEIE`、`ERRIE`、`0`、`SSOE`、`TXDMAEN`、`RXDMAEN` |
| **SPI_SR** | 依次为 `BSY`、`OVR`、`MODF`、`CRCERR`、`0`、`0`、`TXE`、`RXNE` |

课件 Slide 163 用一句话把主干又画了一遍：

> SPI 基本结构：**波特率发生器**、**数据控制器**、GPIO、SCK、MOSI、**开关控制**、GPIO、**发送数据寄存器 TDR**、**移位寄存器**、**接收数据寄存器 RDR**、MISO

把两张图合起来，硬件 SPI 的「数据流」就是一条直线：

```text
       写入                                 读出
        │                                    ▲
   ┌────┴─────┐        ┌──────────┐    ┌─────┴──────┐
   │ 发送缓冲区│───────►│ 移位寄存器│───►│ 接收缓冲区 │
   │   TDR    │        │ (8/16 位) │    │    RDR     │
   └──────────┘        └────┬─────┘    └────────────┘
                            │  ← LSBFIRST 决定从哪头移
                       ┌────┴─────┐
        SCK ◄──────────┤ GPIO 开关 │──────────► MOSI / MISO
     （波特率发生器）   └──────────┘
```

**注意框图上那对「读 / 写」箭头**：发送缓冲区只有「写入」，接收缓冲区只有「读出」，中间共用**一个**移位寄存器。这正是「SPI 收和发是同一个动作」在硬件上的样子——和 11-1 第 3 节讲的移位寄存器交换原理完全对得上。

> [!note] 这些话出自哪里
> 上面这张表里的位名、部件名、连线关系，全部是**课件 Slide 162 那张图里画出来的**（图片标题「图209 SPI框图」），不是本页的推测。图上的 `SPI_CR1` / `SPI_CR2` / `SPI_SR` 三个寄存器框里逐个列出的位名，已与 ST 标准外设库 `stm32f10x_spi.h` 的枚举定义（见第 4、6 节的行号）互相印证。
> 但本页**没有**逐像素比对《STM32F10xxx参考手册（中文）》PDF 里的同名框图，`图209` 这个编号是否就是手册里的图 209，未做确认，已记入「待核对」。

### 3.2 一次「写一个字节」硬件内部发生了什么

| 时刻 | 硬件动作 | 软件该做什么 |
| --- | --- | --- |
| ① | 发送缓冲区**空** → `TXE` 置 1 | 等 `TXE`（等它变 1） |
| ② | 软件把字节写进发送缓冲区 | `SPI_I2S_SendData(SPI1, ByteSend)` |
| ③ | 字节被搬进移位寄存器，发送缓冲区又空了 → `TXE` 再次置 1 | （若要连发，就回到 ①） |
| ④ | 波特率发生器驱动 SCK 打 8 拍，MOSI 一位一位出去，MISO 一位一位进来 | 不用管 |
| ⑤ | 8 位收满 → 接收缓冲区**非空** → `RXNE` 置 1 | 等 `RXNE` |
| ⑥ | 软件把收到的字节从接收缓冲区取走 | `SPI_I2S_ReceiveData(SPI1)` |

**整段时序由硬件打完，CPU 只做「等标志 → 读写寄存器」。** 这 6 步就是 11-5 里 `MySPI_SwapByte()` 的全部内容。

## 4 `SPI_InitTypeDef` 九个成员逐个讲

结构体声明在 ST 标准外设库 `stm32f10x_spi.h` 第 **50~81 行**。下面「本课取值」一列全部来自官方 `11-2 硬件SPI读写W25Q64\Hardware\MySPI.c` 第 **44~52 行**。

| # | 成员 | 本课取值 | 库中可选值（行号） | 这一项在配什么 |
| --- | --- | --- | --- | --- |
| 1 | `SPI_Direction` | `SPI_Direction_2Lines_FullDuplex` | `2Lines_FullDuplex`(`0x0000`)、`2Lines_RxOnly`(`0x0400`)、`1Line_Rx`(`0x8000`)、`1Line_Tx`(`0xC000`)（128~131 行） | **数据方向**：几根线、全双工还是单工。本课要两根数据线同时收发 |
| 2 | `SPI_Mode` | `SPI_Mode_Master` | `SPI_Mode_Master`(`0x0104`)、`SPI_Mode_Slave`(`0x0000`)（144~145 行） | **主 / 从**。本课 STM32 是主机 |
| 3 | `SPI_DataSize` | `SPI_DataSize_8b` | `SPI_DataSize_8b`(`0x0000`)、`SPI_DataSize_16b`(`0x0800`)（156~157 行） | **一帧几位**。W25Q64 按字节走，选 8 位 |
| 4 | `SPI_CPOL` | `SPI_CPOL_Low` | `SPI_CPOL_Low`(`0x0000`)、`SPI_CPOL_High`(`0x0002`)（168~169 行） | **时钟极性**：空闲时 SCK 是高还是低。低 = 模式 0 / 1 |
| 5 | `SPI_CPHA` | `SPI_CPHA_1Edge` | `SPI_CPHA_1Edge`(`0x0000`)、`SPI_CPHA_2Edge`(`0x0001`)（180~181 行） | **时钟相位**：第一个边沿还是第二个边沿动作。配合 CPOL 定成**模式 0** |
| 6 | `SPI_NSS` | `SPI_NSS_Soft` | `SPI_NSS_Soft`(`0x0200`)、`SPI_NSS_Hard`(`0x0000`)（192~193 行） | **NSS 谁来管**：硬件引脚，还是软件（见第 5 节） |
| 7 | `SPI_BaudRatePrescaler` | `SPI_BaudRatePrescaler_128` | `_2`(`0x0000`)、`_4`(`0x0008`)、`_8`(`0x0010`)、`_16`(`0x0018`)、`_32`(`0x0020`)、`_64`(`0x0028`)、`_128`(`0x0030`)、`_256`(`0x0038`)（204~211 行） | **波特率分频**，对应 `BR[2:0]`。本课取 128 分频 |
| 8 | `SPI_FirstBit` | `SPI_FirstBit_MSB` | `SPI_FirstBit_MSB`(`0x0000`)、`SPI_FirstBit_LSB`(`0x0080`)（228~229 行） | **先行位**：高位先发还是低位先发。与 11-3 软件版 `0x80 >> i` 一致 |
| 9 | `SPI_CRCPolynomial` | `7` | 校验：`IS_SPI_CRC_POLYNOMIAL(POLYNOMIAL) ((POLYNOMIAL) >= 0x1)`（425 行） | **CRC 多项式**，本课用不到，官方注释写「暂时用不到，给默认值 7」 |

### 4.1 结构体成员顺序 ≠ 赋值顺序

`SPI_InitTypeDef` 里成员的**声明顺序**是 `Direction → Mode → DataSize → CPOL → CPHA → NSS → BaudRatePrescaler → FirstBit → CRCPolynomial`（第 52~80 行），而官方代码的**赋值顺序**是 `Mode → Direction → DataSize → FirstBit → BaudRatePrescaler → CPOL → CPHA → NSS → CRCPolynomial`（第 44~52 行）。

**两者不一致完全没关系**：这里只是给结构体的成员逐个赋值，怎么排都一样，最后统一交给 `SPI_Init(SPI1, &SPI_InitStructure)` 一次生效（`stm32f10x_spi.h` 第 447 行声明 `void SPI_Init(SPI_TypeDef* SPIx, SPI_InitTypeDef* SPI_InitStruct);`）。

### 4.2 本课 SCK 频率算得出来

`SPI1` 挂在 **APB2** 上（官方源码用 `RCC_APB2PeriphClockCmd(RCC_APB2Periph_SPI1, ENABLE)`，`stm32f10x_rcc.h` 第 508 行），官方工程启动文件把系统时钟配成 72 MHz、APB2 不分频，所以 `f_PCLK = 72 MHz`：

```text
SCK = 72 MHz / 128 = 562.5 kHz
```

（依据：`Start\system_stm32f10x.c` 的 `SetSysClockTo72()`，第 1051/1056 行 `RCC_CFGR_PLLMULL9` + HSE；第 1025 行 `RCC_CFGR_PPRE2_DIV1`；`Start\stm32f10x.h` 第 119 行 `HSE_VALUE = 8000000`。**这是本页按官方工程配置算出来的**，课件 Slide 161 只给了 `f_PCLK / (2…256)` 这个公式。）

## 5 NSS：软件 NSS 与硬件 NSS

### 5.1 库头文件怎么描述这两者

`stm32f10x_spi.h` 第 67~69 行对 `SPI_NSS` 成员的注释原文：

> Specifies whether the NSS signal is managed by **hardware (NSS pin)** or by **software using the SSI bit**.

也就是说：

| 取值 | 谁在管 NSS | 框图上的痕迹 |
| --- | --- | --- |
| `SPI_NSS_Hard`（`0x0000`） | **NSS 引脚**（硬件） | 框图里 **NSS 引脚 → 主控制电路** 那根线起作用 |
| `SPI_NSS_Soft`（`0x0200`） | **软件**，通过 **SSI 位** | 框图里 `SPI_CR1` 的 **SSM / SSI** 两个位 |

配套还有两个库函数（`stm32f10x_spi.h` 第 457~458 行）：

```c
void SPI_NSSInternalSoftwareConfig(SPI_TypeDef* SPIx, uint16_t SPI_NSSInternalSoft);
void SPI_SSOutputCmd(SPI_TypeDef* SPIx, FunctionalState NewState);
```

`SPI_NSSInternalSoft_Set`(`0x0100`) / `SPI_NSSInternalSoft_Reset`(`0xFEFF`) 定义在第 347~348 行。

### 5.2 本课用的是哪个

**`SPI_NSS_Soft`**（官方 `11-2\MySPI.c` 第 51 行），而且**片选根本不经过 SPI 外设**——它由一根**普通 GPIO（PA4）**手动拉低拉高。官方源码第 4 行的函数头注释写得很明白：

> 函    数：SPI写SS引脚电平，**SS仍由软件模拟**

具体写法就是「普通推挽输出 + `GPIO_WriteBit`」：

```c
/**
  * 函    数：SPI写SS引脚电平，SS仍由软件模拟
  * 参    数：BitValue 协议层传入的当前需要写入SS的电平，范围0~1
  * 返 回 值：无
  * 注意事项：此函数需要用户实现内容，当BitValue为0时，需要置SS为低电平，当BitValue为1时，需要置SS为高电平
  */
void MySPI_W_SS(uint8_t BitValue)
{
	GPIO_WriteBit(GPIOA, GPIO_Pin_4, (BitAction)BitValue);		//根据BitValue，设置SS引脚的电平
}
```

```c
	/*GPIO初始化*/
	GPIO_InitTypeDef GPIO_InitStructure;
	GPIO_InitStructure.GPIO_Mode = GPIO_Mode_Out_PP;
	GPIO_InitStructure.GPIO_Pin = GPIO_Pin_4;
	GPIO_InitStructure.GPIO_Speed = GPIO_Speed_50MHz;
	GPIO_Init(GPIOA, &GPIO_InitStructure);					//将PA4引脚初始化为推挽输出
```

注意第 2 段里的 `GPIO_Mode_Out_PP`：**PA4 配的是「普通」推挽，不是复用推挽**——这一点是「片选不归外设管」在代码里最直接的痕迹（第 8 节把四个脚摆在一起对比）。

起始 / 终止仍然是 11-3 那两个函数（官方 `11-2\MySPI.c` 第 67~80 行原文），一行都没变：

```c
/**
  * 函    数：SPI起始
  * 参    数：无
  * 返 回 值：无
  */
void MySPI_Start(void)
{
	MySPI_W_SS(0);				//拉低SS，开始时序
}

/**
  * 函    数：SPI终止
  * 参    数：无
  * 返 回 值：无
  */
void MySPI_Stop(void)
{
	MySPI_W_SS(1);				//拉高SS，终止时序
}
```

> [!tip] 一个能佐证「PA4 没交给外设」的细节
> 引脚定义表第 16 行写着 **PA4 的默认复用功能是 `SPI1_NSS`**。但官方硬件版工程**偏偏没把 PA4 配成复用**，而是配成 `GPIO_Mode_Out_PP`（普通推挽）。这就说明：**本课的片选走的是「软件 NSS + 普通 GPIO」这条路，PA4 上的 `SPI1_NSS` 复用功能没有被启用。**
>
> 好处是省事（一个引脚想怎么拉就怎么拉），代价是每次通信前后都得自己记得翻它。

> [!warning] 硬件 NSS 的细节，本机官方材料里只有一句注解
> 库头文件只说了「由硬件（NSS 引脚）管理」，**没有**展开「从模式下 NSS 低电平才被选中」「主模式下 NSS 引脚的电平要求 / 多主机冲突检测（MODF）」这些规则。框图上只能看到 **NSS 引脚接到主控制电路**、`SPI_SR` 里有 **`MODF`** 位、`SPI_CR2` 里有 **`SSOE`** 位。更细的行为本页不写成肯定句，已记入「待核对」。

## 6 三个状态标志与判定顺序

### 6.1 库里的定义

| 标志 | 值 | 声明行 | 含义 |
| --- | --- | --- | --- |
| `SPI_I2S_FLAG_TXE` | `0x0002` | 第 405 行 | 发送缓冲区**空**（可以写下一个） |
| `SPI_I2S_FLAG_RXNE` | `0x0001` | 第 404 行 | 接收缓冲区**非空**（可以读走一个） |
| `SPI_I2S_FLAG_BSY` | `0x0080` | 第 411 行 | 总线**忙** |

读标志的函数是 `FlagStatus SPI_I2S_GetFlagStatus(SPI_TypeDef* SPIx, uint16_t SPI_I2S_FLAG);`（第 465 行）。

> [!note] 清标志只能清 `CRCERR`
> 库头文件第 412 行：`#define IS_SPI_I2S_CLEAR_FLAG(FLAG) (((FLAG) == SPI_FLAG_CRCERR))`——`SPI_I2S_ClearFlag()` **只允许传 `SPI_FLAG_CRCERR`**。
> 这反过来说明：**`TXE` / `RXNE` / `BSY` 都不是「用函数清」的标志**。参考手册的时序图（下面 6.2）标注得很清楚——`TXE` 是「由硬件设置并由**软件清除**」，而这个「软件清除」就是**写一次发送寄存器**；`RXNE` 的「软件清除」就是**读一次接收寄存器**。所以本课代码里看不到任何清标志的语句。

### 6.2 两份官方时序图的标注

课件 Slide 164 / 165 是两张纯图页，图内标题分别是：

| 课件页 | 图内标题 | 关键标注 |
| --- | --- | --- |
| Slide 164 | 「图213 主模式、全双工模式下(BIDIMODE=0 并且 RXONLY=0)连续传输时，TXE/RXNE/BSY 的变化示意图」 | `TXE` 标志**由硬件设置并由软件清除**；`BSY` 标志**由硬件设置**、末尾**由硬件清除**；`RXNE` 标志**由硬件设置**、**由软件清除**；图下方的软件流程块写着「软件写入 0xF1 至 SPI_DR」「软件等待 RXNE=1 然后从 SPI_DR 读出 0xA1」等 |
| Slide 165 | 「图218 非连续传输发送(BIDIMODE=0 并且 RXONLY=0)时，TXE/BSY 变化示意图」 | 软件流程块写着「**软件等待 TXE=1**，但是较晚写入 0xF2 至 SPI_DR」；最后一个流程块写着「**软件等待 BSY=0**」 |

### 6.3 官方代码就是这样写的

官方 `11-2 硬件SPI读写W25Q64\Hardware\MySPI.c` 第 87~96 行：

```c
uint8_t MySPI_SwapByte(uint8_t ByteSend)
{
	while (SPI_I2S_GetFlagStatus(SPI1, SPI_I2S_FLAG_TXE) != SET);	//等待发送数据寄存器空
	
	SPI_I2S_SendData(SPI1, ByteSend);								//写入数据到发送数据寄存器，开始产生时序
	
	while (SPI_I2S_GetFlagStatus(SPI1, SPI_I2S_FLAG_RXNE) != SET);	//等待接收数据寄存器非空
	
	return SPI_I2S_ReceiveData(SPI1);								//读取接收到的数据并返回
}
```

四步顺序：**等 TXE → 写数据 → 等 RXNE → 读数据**。

### 6.4 为什么是「先 TXE，后 BSY」，反过来不行

先把两个标志各回答什么问题分清楚：

| 标志 | 回答的问题 | 用在哪一步 |
| --- | --- | --- |
| `TXE` | **能不能写下一个字节**（发送缓冲区空了吗） | 写数据**之前** |
| `RXNE` | **有没有收到一个字节**（接收缓冲区满了吗） | 读数据**之前** |
| `BSY` | **现在到底还在不在传**（移位寄存器空了吗） | 一次通信**结束之后** |

**顺序不能反过来，理由有三条：**

1. **等 BSY 起不到「能不能写」的作用。** 在还没写任何数据、上一次传输也已经结束的时候，`BSY` **本来就是 0**，「先等 BSY=0」这一步会**立刻通过**，等于什么都没判。真正决定「能不能写下一个」的只有 `TXE`。
2. **官方时序图把 BSY 放在最后。** Slide 165（图218）的软件流程块，最后的动作才是「**软件等待 BSY=0**」；中间的每一个字节，流程块写的都是「**软件等待 TXE=1** … 然后写入 SPI_DR」。顺序在图上是画死了的。
3. **BSY 是整段传输的「总闸」，不是单字节的「闸」。** Slide 164（图213）里 `BSY` 由硬件在传输开始时置 1、**一直保持到最后一个字节结束才由硬件清 0**（图上标「由硬件设置」「由硬件清除」）；而 `TXE` 在传输过程中会**反复**置位。把「等 BSY=0」塞进每个字节之间，就等于要求**整段传输全部结束**才允许写下一字节——连续传输被强行切成非连续，硬件的缓冲也就白做了。

> [!warning] 但官方这段代码里并没有等 BSY
> 本课的 `MySPI_SwapByte()` **只等了 `TXE` 和 `RXNE`**，全程没有出现 `SPI_I2S_FLAG_BSY`。最后一步「等 RXNE」其实已经在第 8 个时钟结束、字节收满时才返回，所以照本课这样写就能跑通。
> 「`BSY` 用在拉高 SS 之前，确保最后一个字节彻底移位完毕」这条原则出自 Slide 165 的图注；**本课代码没有实现它**，本页把两者都写出来，不把官方没写的代码当成官方写法。

> [!tip] 顺序记成一句话
> **写之前看 TXE，读之前看 RXNE，整段结束再看 BSY。**

## 7 `SendData` / `ReceiveData`：为什么发一个就收一个

库里两个函数的原型（`stm32f10x_spi.h` 第 455~456 行）：

```c
void SPI_I2S_SendData(SPI_TypeDef* SPIx, uint16_t Data);
uint16_t SPI_I2S_ReceiveData(SPI_TypeDef* SPIx);
```

| 观察 | 说明 |
| --- | --- |
| 参数和返回值都是 **`uint16_t`** | 因为数据帧可以是 8 位也可以是 16 位；本课配 `SPI_DataSize_8b`，只用到低 8 位 |
| **写 `SPI_I2S_SendData` 就「开始产生时序」** | 官方注释原文：「写入数据到发送数据寄存器，**开始产生时序**」——SCK 是硬件在这一刻开始打的，不是软件去翻的 |
| 收发是**两个**缓冲区、**一个**移位寄存器 | 框图（第 3 节）里发送缓冲区只有「写入」、接收缓冲区只有「读出」，中间共用移位寄存器。所以**每次写出去一个字节，必然同时换回来一个字节** |

**这就是「为什么读数据时要发 `0xFF`」的根本原因。** 11-3 的 `W25Q64_ReadData()` 里那一行

```c
DataArray[i] = MySPI_SwapByte(W25Q64_DUMMY_BYTE);	//依次在起始地址后读取数据
```

在硬件版里等价于「**写一个 dummy 换一个真数据**」。这不是 W25Q64 的要求，而是 **SPI 全双工结构**的要求：移位寄存器要转，就得有东西给它转；出来的那 8 位才是你要的数据。

## 8 GPIO 怎么配：三脚复用推挽 + 一脚上拉输入

用硬件 SPI 时，引脚不再由普通 GPIO 输出寄存器控制，而是要交给 SPI 外设。官方 `11-2\MySPI.c` 第 22~40 行：

| 引脚 | 信号 | 方向 | `GPIO_Mode` | 官方源码行 |
| --- | --- | --- | --- | --- |
| PA4 | SS（片选） | 主机输出 | **`GPIO_Mode_Out_PP`**（普通推挽，**不**复用） | 第 27~30 行 |
| PA5 | SCK | 主机输出 | **`GPIO_Mode_AF_PP`**（复用推挽） | 第 32~35 行 |
| PA7 | MOSI | 主机输出 | **`GPIO_Mode_AF_PP`**（复用推挽） | 第 32~35 行 |
| PA6 | MISO | 主机输入 | **`GPIO_Mode_IPU`**（上拉输入） | 第 37~40 行 |

```c
	/*GPIO初始化*/
	GPIO_InitTypeDef GPIO_InitStructure;
	GPIO_InitStructure.GPIO_Mode = GPIO_Mode_Out_PP;
	GPIO_InitStructure.GPIO_Pin = GPIO_Pin_4;
	GPIO_InitStructure.GPIO_Speed = GPIO_Speed_50MHz;
	GPIO_Init(GPIOA, &GPIO_InitStructure);					//将PA4引脚初始化为推挽输出
	
	GPIO_InitStructure.GPIO_Mode = GPIO_Mode_AF_PP;
	GPIO_InitStructure.GPIO_Pin = GPIO_Pin_5 | GPIO_Pin_7;
	GPIO_InitStructure.GPIO_Speed = GPIO_Speed_50MHz;
	GPIO_Init(GPIOA, &GPIO_InitStructure);					//将PA5和PA7引脚初始化为复用推挽输出
	
	GPIO_InitStructure.GPIO_Mode = GPIO_Mode_IPU;
	GPIO_InitStructure.GPIO_Pin = GPIO_Pin_6;
	GPIO_InitStructure.GPIO_Speed = GPIO_Speed_50MHz;
	GPIO_Init(GPIOA, &GPIO_InitStructure);					//将PA6引脚初始化为上拉输入
```

要点只有三句：

- **要交给外设的脚（SCK / MOSI）配 `AF_PP`**，也就是「复用推挽输出」。这两个脚在软件版里配的是 `Out_PP`，硬件版改成了 `AF_PP`——**官方两个工程在这一格上的差异，就是「引脚归谁管」的直接证据**；
- **MISO 仍然是上拉输入**（和软件版一样，`GPIO_Mode_IPU`），它永远不由 STM32 输出；
- **片选 PA4 仍然是普通的 `Out_PP`**，因为它压根不归 SPI 外设管（见第 5 节）。

时钟要开两个（第 22~23 行）：

```c
	RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOA, ENABLE);	//开启GPIOA的时钟
	RCC_APB2PeriphClockCmd(RCC_APB2Periph_SPI1, ENABLE);	//开启SPI1的时钟
```

软件版**只开 GPIOA**，硬件版**多开一个 SPI1**——这也是「外设参与进来了」的标志。

## 9 硬件 SPI 相比软件 SPI 的优势与坑

### 9.1 优势

| 方面 | 软件 SPI（11-3） | 硬件 SPI（11-4 讲的这套） |
| --- | --- | --- |
| 时钟生成 | CPU 一条一条翻 GPIO | **波特率发生器**自动产生 |
| 数据搬运 | 每字节 8 轮循环、32 次 GPIO 操作 | 写一次发送寄存器，硬件移位 |
| CPU 占用 | 全程参与 | 只需「等标志 + 读写寄存器」 |
| 速率 | 受 GPIO 翻转与循环开销限制 | 由 `f_PCLK / (2…256)` 分频决定，可到 MHz 级 |
| 波形 | 每位之间被循环和中断拉长，容易不规整 | 硬件打拍，波形规整（课件 Slide 166「软件 / 硬件波形对比」） |
| 扩展 | 无 | 8/16 位帧、MSB/LSB、主/从、半双工/单工、DMA、I2S（Slide 161） |

### 9.2 坑

| 坑 | 依据 |
| --- | --- |
| **片选不会自己动** | 官方注释「SS 仍由软件模拟」；PA4 配的是普通推挽。忘了在每次通信前后翻它，波形全对但芯片不响应 |
| **SCK/MOSI 配成普通推挽（`Out_PP`）就轮不到外设管** | 软件版与硬件版在这一格上的差异（第 8 节） |
| **标志顺序不能乱** | `TXE` 在写之前、`RXNE` 在读之前（第 6.4 节） |
| **`SPI_I2S_ClearFlag` 清不了 TXE/RXNE/BSY** | 库头文件第 412 行只允许 `SPI_FLAG_CRCERR`；`TXE`/`RXNE` 靠读写寄存器清 |
| **分频选太大 / 太小时要点一下从机上限** | 课件 Slide 161 给了 8 档分频；本课取 128 分频（SCK ≈ 562.5 kHz，见 4.2） |
| **别以为「有硬件 SPI 就不用管 SS 了」** | 这正是 11-5 里 `MySPI_Start/Stop` 仍然存在的原因 |

> [!note] Slide 166 的两张抓包图本页不解读
> 课件 Slide 166「软件 / 硬件波形对比」是纯图页，放了两张逻辑分析仪抓包图（`image152.jpg` 与 `image145.jpeg`）。**课件文本里没有任何说明文字**，本页无法从官方材料确定哪一张对应软件、哪一张对应硬件，因此不对两张图下结论，已记入「待核对」。

## 10 易错点

- [ ] 以为「用了硬件 SPI，片选也归硬件管」→ 本课 `SPI_NSS_Soft` + PA4 普通 GPIO，**片选必须自己翻**。
- [ ] PA5 / PA7 配成 `GPIO_Mode_Out_PP`（照 11-3 抄）→ 硬件版这两脚要用 `GPIO_Mode_AF_PP`，否则引脚不由 SPI 外设控制。
- [ ] 忘了 `RCC_APB2PeriphClockCmd(RCC_APB2Periph_SPI1, ENABLE)` → 外设没时钟，寄存器写不进、标志不动。
- [ ] 忘了 `SPI_Cmd(SPI1, ENABLE)` → 配置完但外设没使能（框图里 `SPI_CR1` 的 `SPE` 位），一个字节也发不出去。
- [ ] `CPOL` / `CPHA` 配成模式 1 → W25Q64 用模式 0，配错就是整体错位、读回乱码。
- [ ] 先等 `BSY` 再等 `TXE` → 没写数据时 `BSY` 本来就是 0，白等；而中间字节的判据只能是 `TXE`。
- [ ] 用 `SPI_I2S_ClearFlag()` 去清 `TXE` / `RXNE` → 库里只允许清 `SPI_FLAG_CRCERR`；`TXE`/`RXNE` 是**读写寄存器**时自动清的。
- [ ] 读数据时忘了「发一个 dummy」→ SPI 是全双工，不写就收不到。
- [ ] `SPI_I2S_SendData` 的参数类型是 `uint16_t`，就以为 8 位模式要左移 → 本课是 8 位帧，直接传字节即可（官方就是这么写的）。
- [ ] 把 `SPI_InitTypeDef` 成员的**赋值顺序**当成必须和声明顺序一致 → 不必，最后统一 `SPI_Init()`。

## 11 自测

1. 课件 Slide 161 说 SPI 外设「减轻 CPU 的负担」，具体是哪些工作被硬件接过去了？
2. `SPI_InitTypeDef` 里哪两个成员合起来决定「模式几」？本课取的是什么值、对应模式几？
3. 本课的 NSS 用的是软件还是硬件？片选线实际由谁控制？依据是什么？
4. 官方 `MySPI_SwapByte()` 里两个 `while` 各等什么标志、顺序是什么？为什么不能把「等 BSY」放在「写数据」之前？
5. 用硬件 SPI 时，PA5 / PA6 / PA7 三个脚各配成什么 GPIO 模式？PA4 为什么和它们不一样？

> [!success]- 参考答案
> 1. **时钟生成**（波特率发生器）和**数据收发**（发送缓冲区 → 移位寄存器 → 接收缓冲区）都由硬件自动完成；CPU 只剩「等标志、读写寄存器」。课件 Slide 161 原文：「由硬件自动执行时钟生成、数据收发等功能，减轻 CPU 的负担」。
> 2. `SPI_CPOL` 与 `SPI_CPHA`。本课取 `SPI_CPOL_Low` + `SPI_CPHA_1Edge`，即**模式 0**（官方注释直接写「极性和相位决定选择 SPI 模式 0」，`11-2\MySPI.c` 第 49~50 行）。
> 3. 用**软件**（`SPI_NSS_Soft`，第 51 行），片选由**普通 GPIO 的 PA4** 手动拉低拉高。依据：`MySPI_W_SS()` 里 `GPIO_WriteBit(GPIOA, GPIO_Pin_4, …)`、PA4 配成 `GPIO_Mode_Out_PP`（不是复用），以及函数头注释「SS 仍由软件模拟」。
> 4. 第一个 `while` 等 `SPI_I2S_FLAG_TXE`（发送数据寄存器空，等到了才写），第二个 `while` 等 `SPI_I2S_FLAG_RXNE`（接收数据寄存器非空，等到了才读）。不能把「等 BSY」放在「写数据」之前：`BSY` 回答的是「还在不在传」，没写数据时它本来就是 0，等它等于没判；官方时序图（Slide 165）里 `BSY` 只出现在**最后**一步「软件等待 BSY=0」。
> 5. PA5、PA7 是 `GPIO_Mode_AF_PP`（复用推挽输出），PA6 是 `GPIO_Mode_IPU`（上拉输入）。PA4 是 `GPIO_Mode_Out_PP`（普通推挽），因为**片选不归 SPI 外设管**——它由软件当普通 GPIO 用，所以不能配复用。

## 12 待核对

- [ ] 课件 Slide 162 那张框图的图内标题是「**图209 SPI框图**」，但本页**没有**逐像素比对《STM32F10xxx参考手册（中文）》PDF，`图209` 是否就是手册里的图 209，未做确认。
- [ ] **硬件 NSS（`SPI_NSS_Hard`）的完整行为**：库头文件只有一句「由硬件（NSS 引脚）管理」，从模式选片、主模式下的 NSS 电平要求与多主机冲突检测（`MODF`）等细节，本机课件与源码中都没有文字说明（框图上只能看到 NSS → 主控制电路、`SPI_SR` 有 `MODF`、`SPI_CR2` 有 `SSOE`）。
- [ ] 本课 SCK 的 **562.5 kHz** 是本页按「SPI1 挂 APB2、APB2 = 72 MHz、128 分频」算出来的；课件 Slide 161 只给了 `f_PCLK / (2…256)` 这个公式，没有给具体频率。
- [ ] 课件 Slide 166 的两张逻辑分析仪抓包图（`image152.jpg`、`image145.jpeg`）**哪张是软件、哪张是硬件**，课件文本里没有说明，本页不下结论。
- [ ] 「本课代码没有等 BSY，靠等 RXNE 已经够用」是本页按 `RXNE` 的置位时机推的，**官方没有文字说明**为什么可以不写 `BSY`；而图表（Slide 165）里明确有「软件等待 BSY=0」这一步。
- [ ] 「硬件 SPI 波形更规整、速率更高」的定量数据（示波器 / 逻辑分析仪实测）本机官方材料中没有。
- [ ] 本集是**理论集**，官方没有独立工程目录；本页的配置代码全部引自 `11-2 硬件SPI读写W25Q64\Hardware\MySPI.c`（该工程完整讲解属于 11-5），老师课上是否还有别的示例代码，需要看视频。
- [ ] 老师的口头讲解原话（本页技术事实都有课件 / 源码 / 库头文件依据，但「老师当时怎么说的」不在这些材料里）。

## 核对记录（2026-09-26）

> [!success] 核对依据
> - 官方配套源码（本集为理论集，代码引自硬件版工程）：`STM32Project-有注释版\11-2 硬件SPI读写W25Q64\Hardware\MySPI.c`（96 行）、`MySPI.h`、`Start\system_stm32f10x.c`、`Start\stm32f10x.h`
> - 课程课件文本：`ground-truth\课件文本.md`（Slide 161 SPI 外设简介、163 SPI 基本结构；另 Slide 162 SPI 框图、164 主模式全双工连续传输、165 非连续传输、166 软件/硬件波形对比 为纯图页，图内文字见 `ppt_x\ppt\media\image149.png`、`image150.png`、`image151.png`、`image152.jpg`、`image145.jpeg`）
> - 官方接线图：`ground-truth\接线图\11-2 硬件SPI读写W25Q64.png`
> - 引脚定义表：`ground-truth\引脚定义_xlsx原文.txt` 第 16~19 行
> - ST 标准外设库：`STM32F10x_StdPeriph_Lib_V3.5.0\...\inc\stm32f10x_spi.h`（结构体 50~81 行、各枚举、标志 404/405/411 行、函数原型 446~468 行）、`stm32f10x_gpio.h`（62/75/77/79 行）、`stm32f10x_rcc.h`（498/508 行）

| 核对项 | 笔记原值 | 官方依据 | 结论 |
| --- | --- | --- | --- |
| SPI 外设能力表 | 8/16 位帧、MSB/LSB、`f_PCLK/(2…256)`、多主机、主/从、半双工/单工、DMA、I2S | 课件 Slide 161 逐字一致 | 一致 |
| F103C8T6 的 SPI 资源 | SPI1、SPI2 | 课件 Slide 161 逐字一致 | 一致 |
| 框图部件 | 发送缓冲区 TDR、接收缓冲区 RDR、移位寄存器、波特率发生器、主控制电路、通信电路、GPIO 开关控制 | 课件 Slide 162 图内文字（`image149.png`，标题「图209 SPI框图」）；Slide 163 文字 | 一致（转述图内文字） |
| 框图寄存器位名 | `SPI_CR1`：LSBFIRST/SPE/BR2~0/MSTR/CPOL/CPHA/BIDIMODE/BIDIOE/CRCEN/CRCNEXT/DFF/RXONLY/SSM/SSI；`SPI_CR2`：TXEIE/RXNEIE/ERRIE/SSOE/TXDMAEN/RXDMAEN；`SPI_SR`：BSY/OVR/MODF/CRCERR/TXE/RXNE | 课件 Slide 162 图内文字；`stm32f10x_spi.h` 中各枚举定义互相印证 | 一致 |
| `SPI_InitTypeDef` 成员数 | 9 个 | `stm32f10x_spi.h` 第 50~81 行 | 一致 |
| `SPI_Direction` | `SPI_Direction_2Lines_FullDuplex` | 源码第 45 行；`stm32f10x_spi.h` 第 128~131 行四个取值 | 一致 |
| `SPI_Mode` | `SPI_Mode_Master` | 源码第 44 行；`stm32f10x_spi.h` 第 144~145 行 | 一致 |
| `SPI_DataSize` | `SPI_DataSize_8b` | 源码第 46 行；`stm32f10x_spi.h` 第 156~157 行 | 一致 |
| `SPI_CPOL` | `SPI_CPOL_Low` | 源码第 49 行；`stm32f10x_spi.h` 第 168~169 行 | 一致 |
| `SPI_CPHA` | `SPI_CPHA_1Edge` | 源码第 50 行；`stm32f10x_spi.h` 第 180~181 行 | 一致 |
| `SPI_NSS` | `SPI_NSS_Soft` | 源码第 51 行；`stm32f10x_spi.h` 第 192~193 行 | 一致 |
| `SPI_BaudRatePrescaler` | `SPI_BaudRatePrescaler_128` | 源码第 48 行；`stm32f10x_spi.h` 第 204~211 行共 8 档 | 一致 |
| `SPI_FirstBit` | `SPI_FirstBit_MSB` | 源码第 47 行；`stm32f10x_spi.h` 第 228~229 行 | 一致 |
| `SPI_CRCPolynomial` | `7`，注释「暂时用不到，给默认值7」 | 源码第 52 行；`stm32f10x_spi.h` 第 425 行只校验 `>= 0x1` | 一致 |
| 结构体声明顺序与赋值顺序不同 | 是（声明 Direction 起头，赋值 Mode 起头） | `stm32f10x_spi.h` 第 52~80 行 vs 源码第 44~52 行 | 一致（本页说明不影响结果） |
| NSS 两种取值的含义 | 硬件用 NSS 引脚，软件用 SSI 位 | `stm32f10x_spi.h` 第 67~69 行注释原文 | 一致 |
| 本课用软件 NSS、片选走普通 GPIO | 是 | 源码第 51 行 `SPI_NSS_Soft`；第 9~12 行 `MySPI_W_SS` 用 `GPIO_WriteBit`；第 27 行 PA4 配 `GPIO_Mode_Out_PP`；第 4 行注释「SS仍由软件模拟」 | 一致 |
| 标志值 | TXE `0x0002`、RXNE `0x0001`、BSY `0x0080` | `stm32f10x_spi.h` 第 404、405、411 行 | 一致 |
| `SPI_I2S_ClearFlag` 只能清 CRCERR | 是 | `stm32f10x_spi.h` 第 412 行 `IS_SPI_I2S_CLEAR_FLAG` 只允许 `SPI_FLAG_CRCERR` | 一致 |
| 标志判定顺序 | 等 TXE → `SendData` → 等 RXNE → `ReceiveData` | 源码第 89~95 行，逐行一致 | 一致 |
| TXE / RXNE / BSY 的置清方式 | TXE 硬件置、软件清；RXNE 硬件置、软件清；BSY 硬件置、硬件清 | 课件 Slide 164 图内标注（`image150.png`） | 一致（转述图内文字） |
| 「先等 TXE 再等 BSY」的理由 | 见 6.4 节三条 | Slide 165 图内流程块「软件等待 TXE=1」「软件等待 BSY=0」；Slide 164 的 BSY 标注 | 一致（推理部分已注明是本页论证） |
| 本课代码是否等 BSY | **没有**，只等了 TXE 和 RXNE | 源码第 87~96 行全文无 `SPI_I2S_FLAG_BSY` | 一致（本页已显式说明） |
| 函数原型 | `SPI_I2S_SendData(SPIx, uint16_t Data)` / `uint16_t SPI_I2S_ReceiveData(SPIx)` | `stm32f10x_spi.h` 第 455~456 行 | 一致 |
| SCK 频率 | 72 MHz / 128 ≈ 562.5 kHz | 源码第 23 行 `RCC_APB2Periph_SPI1`；`stm32f10x_rcc.h` 第 508 行；`system_stm32f10x.c` 第 1051/1056/1025 行；`stm32f10x.h` 第 119 行 | 一致（本页按官方工程配置算得） |
| GPIO 模式 | PA4 `Out_PP`、PA5/PA7 `AF_PP`、PA6 `IPU` | 源码第 27~40 行 | 一致 |
| 时钟 | 开 GPIOA + SPI1 | 源码第 22~23 行 | 一致 |
| 引脚复用依据 | PA4 `SPI1_NSS`、PA5 `SPI1_SCK`、PA6 `SPI1_MISO`、PA7 `SPI1_MOSI` | `引脚定义_xlsx原文.txt` 第 16~19 行 | 一致 |
| 硬件 NSS 的具体行为 | 本页**不写**细节 | 库头文件只有一句注解，课件与源码无更多文字 | 一致（本页主动回避） |
| Slide 166 两张抓包图的归属 | 本页**不判** | 课件文本该页无文字 | 一致（本页主动回避） |

> [!note] 出处说明
> 本页的 SPI 外设能力、数据帧 / 分频 / 主从等特性逐字对照课件 Slide 161；内部结构（发送 / 接收缓冲区、移位寄存器、波特率发生器、主控制电路、通信电路、`SPI_CR1`/`CR2`/`SR` 位名）转述自课件 Slide 162 的框图（`ppt_x\ppt\media\image149.png`，图内标题「图209 SPI框图」）与 Slide 163；`TXE`/`RXNE`/`BSY` 的置清方式与判定顺序转述自课件 Slide 164 / 165 的两张时序图（`image150.png`、`image151.png`）；`SPI_InitTypeDef` 九个成员、各枚举取值、标志值、函数原型全部逐个对照 ST 标准外设库 `stm32f10x_spi.h`；GPIO 配置、SPI 初始化与 `MySPI_SwapByte` 的代码逐字抄自官方配套源码 `11-2 硬件SPI读写W25Q64\Hardware\MySPI.c`（本集是理论集，官方**没有**独立工程目录）。
> 本页**未**引用任何第三方笔记，也**未**引用 Gitee 镜像 `KSweb/stm32f1doc`。
> **仍未核实**：框图编号「图209」与参考手册 PDF 的对应关系、硬件 NSS 的完整行为、SCK 具体频率（本页算得）、Slide 166 两张抓包图的归属、「不等 BSY 也够用」的理由、波形对比的实测数据、老师课上是否另有示例代码与口头讲解原话——这些不在可查证的官方材料范围内，相关条目已留在上面的「待核对」。
