---
course: 江协科技 STM32入门教程-2023版
chapter: 09-2 USART串口外设
type: 分集笔记
source: https://www.bilibili.com/video/BV1th411z7sn?p=26
tags:
  - STM32
  - 江协科技
  - USART
  - 串口
  - 中断
status: draft
verify: 官方源码+接线图+课件
updated: 2026-09-26
---

# 09-2 USART 串口外设

> [!abstract] 这一集只解决一个问题
> **上一集讲了协议，这一集把「按协议收发」这件事交给硬件去做。**
>
> 答：STM32 内部有一个 **USART 外设**。你往**数据寄存器**里写一个字节，它自动加上起始位、停止位、算好校验、按波特率从 **TX** 引脚一位一位发出去；RX 引脚来了一帧，它自动拼成一个字节塞进数据寄存器并**举手通知你**。CPU 只跟「数据寄存器」和一个「状态位」打交道。
>
> 上一集 [[江协STM32 09-1 USART串口协议]] ｜ 章索引 [[江协STM32 09 USART串口（章索引）]] ｜ 下一集 [[江协STM32 09-3 串口发送与串口发送接收]]

## 1 USART 是什么

课件 Slide 114 的定义：

> USART（Universal Synchronous/Asynchronous Receiver/Transmitter）**通用同步/异步收发器**。USART 是 STM32 内部集成的硬件外设，**可根据数据寄存器的一个字节数据自动生成数据帧时序，从 TX 引脚发送出去，也可自动接收 RX 引脚的数据帧时序，拼接为一个字节数据，存放在数据寄存器里**。

这句话就是整集的纲。它后面还列了 USART 的能力：

| 能力 | 取值 |
| --- | --- |
| 波特率 | 自带波特率发生器，**最高达 4.5 Mbits/s** |
| 数据位长度 | **8 / 9** |
| 停止位长度 | **0.5 / 1 / 1.5 / 2** |
| 校验位 | 无校验 / 奇校验 / 偶校验 |
| 其他 | 支持同步模式、硬件流控制、DMA、智能卡、IrDA、LIN |

**STM32F103C8T6 的 USART 资源：USART1、USART2、USART3**（Slide 114）。

## 2 内部框图：一个「双缓冲 + 移位」的结构

课件 Slide 116 的「USART 基本结构」抽掉了细节，只留主干：

```text
                  ┌─────────────── 发送数据寄存器 TDR ──► 发送移位寄存器 ──► TX ──►
   PCLK2/1 ──► 波特率发生器                                          │
                  └─────────────── 接收数据寄存器 RDR ◄── 接收移位寄存器 ◄── RX ◄──
                        发送控制器 / 接收控制器 / 开关控制
```

官方参考手册的完整框图（幻灯片 Slide 115 用的就是这张图，图 248「USART 框图」）信息更全，值得对着看一遍。把它读成四块：

| 模块 | 里面有什么 | 干什么 |
| --- | --- | --- |
| **发送部分** | 发送数据寄存器 **TDR** → 发送移位寄存器 | 把并行的一个字节变成串行的位流送到 TX |
| **接收部分** | 接收移位寄存器 → 接收数据寄存器 **RDR** | 把 RX 上的串行位流拼回一个字节 |
| **波特率发生器** | **USART_BRR**（`DIV_Mantissa` / `DIV_Fraction`）、`/16`、`USARTDIV`，分「发送器波特率控制」和「接收器波特率控制」两个 | 产生收发共用的位时钟节拍 |
| **控制逻辑** | `CR1` / `CR2` / `CR3`、`SR`（状态位）、**USART 中断控制** | 配置帧格式、使能收发（`TE` / `RE`）、上报状态与中断 |

框图里还有几处细节，本课程用不到但看一眼能避免误解：

- 左侧 `IrDA SIR 编码码模块`、`nRTS` / `nCTS` 硬件数据流控、`SCLK` 同步时钟——这些就是 Slide 114 里列的「同步模式、硬件流控制、智能卡、IrDA、LIN」。**本课程一个都不用**。
- `CR1` 里的 `M` 位决定字长是 8 位还是 9 位（图 249「字长设置」就是按有没有设 `M` 位分成上下两半画的）。
- 图 249 / 250 里都出现了一句注：`LBCL` 位控制最后一个数据位的时钟脉冲——这是**同步模式**才关心的事。

> [!note] 图上标注的 `PCLKx(x=1,2)`
> USART1 用 `PCLK2`，USART2 / USART3 用 `PCLK1`。框图上写 `/16` 的那一格，就是参考手册里说的「16 倍过采样」——接收方每个位采 16 次，在其中 3 次上取多数值（图 252「起始位侦测」、图 253「检测噪声的数据采样」把 `7/16 + 6/16 + 7/16` 的画法画得很清楚）。

## 3 为什么「写 TDR 就发、读 RDR 就收」

这是整集最该想明白的一点。

从 CPU 的视角看，发的和收的**是同一个寄存器**：标准库对外的偏移量都是 `DR`（数据寄存器），`USART_SendData()` 和 `USART_ReceiveData()` 都访问它。

但在外设内部，**发送和接收走的是两套完全独立的寄存器**：

```text
   CPU 写 ──► [ 发送数据寄存器 TDR ] ──► [ 发送移位寄存器 ] ──► TX
                                                                  （物理上串行出去）

   RX ──► [ 接收移位寄存器 ] ──► [ 接收数据寄存器 RDR ] ──► CPU 读
```

关键在**移位寄存器**这一级：

| 寄存器 | 谁改它 | 变化速度 |
| --- | --- | --- |
| **TDR / RDR**（数据寄存器） | CPU 读或写，一次性搬一个字节 | **瞬间**（几个总线周期） |
| **发送 / 接收移位寄存器** | 硬件一位一位地移 | **慢**，一位一个位时间 |

**「数据寄存器 + 移位寄存器」两级合起来叫双缓冲。** 它的价值：

- **发送侧**：CPU 把字节写进 TDR 后立刻就可以走了，硬件自己慢慢往外移。TDR 空了（`TXE = 1`）就可以写下一个字节，**不必等这一帧真正发完**。
- **接收侧**：硬件把拼好的字节放进 RDR，`RXNE` 置 1 通知 CPU。CPU 在**下一个字节到齐之前**把它读走就行——这中间有一整帧的时间可以慢慢来。

> [!tip] 一句话记住
> **TDR / RDR 是「中转箱」，移位寄存器是「传送带」。** 中转箱瞬间放/取，传送带一位一位地走。这正是 `Serial_SendByte()` 里那句 `while (USART_GetFlagStatus(USART1, USART_FLAG_TXE) == RESET);` 能成立的原因。

## 4 四个关键状态位

框图的 `SR`（状态寄存器）一格列出了全部状态位。本课程关心这四个：

| 状态位 | 全称 | 含义 | 什么时候置 1 | 怎么清 | 对应库函数 / 中断 |
| --- | --- | --- | --- | --- | --- |
| **TXE** | Transmit Data Register Empty | **发送数据寄存器空** | TDR 里的数据被搬进移位寄存器后 | **写一次 TDR 自动清**，不必手动清 | `USART_FLAG_TXE`；中断 `USART_IT_TXE` |
| **TC** | Transmission Complete | **发送完成** | 整个帧（含停止位）都从 TX 发出去了 | 先读 `SR` 再写 `DR`（软件序列） | `USART_FLAG_TC`；中断 `USART_IT_TC` |
| **RXNE** | Read Data Register Not Empty | **读数据寄存器非空** | 一帧接收完，字节已放进 RDR | **读一次 RDR 自动清** | `USART_FLAG_RXNE`；中断 `USART_IT_RXNE` |
| **IDLE** | Idle Line Detected | **总线空闲**（RX 上检测到空闲帧） | RX 上出现一个完整帧时间的空闲电平 | 读 `SR` 再读 `DR` | `USART_FLAG_IDLE`；中断 `USART_IT_IDLE` |

### 4.1 TXE 和 TC 到底差在哪

这一对最容易混：

```text
写 TDR ──► TDR 有数据            TXE = 0
                │
                └─ 搬进移位寄存器 ─► TXE = 1   ← 「可以写下一个字节了」
                       │
                       └─ 一位一位发出去，最后一帧的停止位发完 ─► TC = 1  ← 「线真的空了」
```

| | TXE | TC |
| --- | --- | --- |
| 关心的对象 | **数据寄存器**空不空 | **整条发送链路**（含移位寄存器）空不空 |
| 用途 | 连续发字节时判断「能不能写下一个」 | 发完一串数据后判断「线彻底空了」 |
| 本课程用不用 | **用**（`Serial_SendByte` 就靠它） | 课件讲了，本课程两个工程的源码里未使用 |

### 4.2 RXNE 与接收中断

课件 Slide 116 的框图中「USART 中断控制」那一格，接收侧要经过 `RXNEIE` 这个使能位；标准库里对应的就是：

```c
USART_ITConfig(USART1, USART_IT_RXNE, ENABLE);      // 使能「接收数据寄存器非空」中断
```

框架同第 5 章 EXTI、第 6 章 TIM：**外设内部使能中断 → NVIC 使能通道 → 中断服务函数里判断并清标志**。

## 5 波特率发生器与 `USARTDIV`

课件 Slide 121：

> 发送器和接收器的波特率由波特率寄存器 **BRR** 里的 **DIV** 确定。计算公式：**波特率 = f_PCLK2/1 / (16 × DIV)**

官方参考手册给出了 `BRR` 的位定义（图 251「波特率寄存器（USART_BRR）」）：

| 位 | 名称 | 含义 |
| --- | --- | --- |
| 31:16 | 保留 | 硬件强制为 0 |
| **15:4** | **`DIV_Mantissa[11:0]`** | `USARTDIV` 的**整数部分**（12 位） |
| **3:0** | **`DIV_Fraction[3:0]`** | `USARTDIV` 的**小数部分**（4 位） |

```text
USARTDIV = DIV_Mantissa + (DIV_Fraction / 16)
```

两边一致：`BRR` 里的 `DIV` 就是 `USARTDIV`，而 `波特率 = f_PCLK / (16 × USARTDIV)`，与课件公式完全等价。

### 5.1 标准库怎么算的

不用手算，`USART_Init()` 会按 `USART_BaudRate` 自动填 `BRR`。它的算法（`stm32f10x_usart.c`，16 倍过采样分支）：

```c
apbclock = RCC_ClocksStatus.PCLK2_Frequency;   /* USART1 取 PCLK2，USART2/3 取 PCLK1 */
integerdivider    = ((25 * apbclock) / (4 * (USART_InitStruct->USART_BaudRate)));
tmpreg            = (integerdivider / 100) << 4;               /* 整数部分放到 bit15:4 */
fractionaldivider = integerdivider - (100 * (tmpreg >> 4));
tmpreg           |= (((fractionaldivider * 16) + 50) / 100) & ((uint8_t)0x0F);  /* 小数部分 */
```

也就是 `25 × PCLK / (4 × baud)` 相当于把 `PCLK / (16 × baud)` 放大 100 倍，再分离整数与小数。**关键：库函数自己会去读 `PCLK2` 的当前频率**，所以只要时钟树配得对，波特率就是对的。

### 5.2 本课程 9600 的验算

`PCLK2 = 72 MHz`、目标 9600：

```text
USARTDIV = 72 000 000 / (16 × 9600) = 468.75
整数部分 = 468 = 0x1D4     小数部分 = 0.75 × 16 = 12 = 0xC
BRR = (468 << 4) | 12 = 0x1D4C
```

用库函数的算法复算一遍，结果完全相同：

```text
25 × 72 000 000 / (4 × 9600) = 46 875
46875 / 100 = 468            整数部分 ✔
46875 - 46800 = 75           → (75 × 16 + 50) / 100 = 12  小数部分 ✔
```

> [!warning] 高波特率时小数位会「不够用」
> 小数部分只有 **4 位**，也就是最多 1/16 的分辨率。波特率越高，`USARTDIV` 越小，1/16 造成的相对误差越大。这就是参考手册框图上注明「如果 `TE` 或 `RE` 被分别禁止，波特计数器停止计数」之外，另一个「为什么波特率越高越要小心时钟」的原因。

## 6 `USART_InitTypeDef` 六个成员

`stm32f10x_usart.h` 第 50~76 行：

```c
typedef struct
{
  uint32_t USART_BaudRate;            /* 波特率 */
  uint16_t USART_WordLength;          /* 字长（数据位） */
  uint16_t USART_StopBits;            /* 停止位 */
  uint16_t USART_Parity;              /* 校验位 */
  uint16_t USART_Mode;                /* 发送 / 接收模式 */
  uint16_t USART_HardwareFlowControl; /* 硬件流控制 */
} USART_InitTypeDef;
```

| 成员 | 本课程取值 | 可选值（`stm32f10x_usart.h`） | 说明 |
| --- | --- | --- | --- |
| `USART_BaudRate` | `9600` | 任意合法值（库函数据此自己填 `BRR`） | 不是寄存器值，是「多少 bps」 |
| `USART_WordLength` | `USART_WordLength_8b` | `_8b`(0x0000) / `_9b`(0x1000) | 数据位数 |
| `USART_StopBits` | `USART_StopBits_1` | `_1`(0x0000) / `_0_5`(0x1000) / `_2`(0x2000) / `_1_5`(0x3000) | 停止位数 |
| `USART_Parity` | `USART_Parity_No` | `_No`(0x0000) / `_Even`(0x0400) / `_Odd`(0x0600) | 开了校验位会占 MSB |
| `USART_Mode` | 9-1：`USART_Mode_Tx`<br>9-2：`USART_Mode_Tx \| USART_Mode_Rx` | `_Rx`(0x0004) / `_Tx`(0x0008)，可用 `\|` 组合 | 决定 `RE` / `TE` 两位 |
| `USART_HardwareFlowControl` | `USART_HardwareFlowControl_None` | `_None` / `_RTS` / `_CTS` / `_RTS_CTS` | 本课程不用，填 `None` |

> [!tip] `USART_Mode` 是「按位或」组合的
> `USART_Mode_Rx` = `0x0004`、`USART_Mode_Tx` = `0x0008`，本身就占了不同的位，所以 `USART_Mode_Tx | USART_Mode_Rx` 才是全双工。9-1 只要发，就只写 `USART_Mode_Tx`；9-2 要收，才把两个都写上。

## 7 时钟：为什么是 APB2，以及 GPIO 复用

### 7.1 USART1 挂在 APB2

标准库 `stm32f10x_rcc.h`：

```c
#define RCC_APB2Periph_USART1   ((uint32_t)0x00004000)   /* APB2 */
#define RCC_APB1Periph_USART2   ((uint32_t)0x00020000)   /* APB1 */
#define RCC_APB1Periph_USART3   ((uint32_t)0x00040000)   /* APB1 */
```

| 外设 | 总线 | 时钟使能函数 | 时钟源 |
| --- | --- | --- | --- |
| **USART1** | **APB2** | `RCC_APB2PeriphClockCmd(RCC_APB2Periph_USART1, ENABLE)` | `PCLK2`（本课程 72 MHz） |
| USART2 | APB1 | `RCC_APB1PeriphClockCmd(RCC_APB1Periph_USART2, ENABLE)` | `PCLK1` |
| USART3 | APB1 | `RCC_APB1PeriphClockCmd(RCC_APB1Periph_USART3, ENABLE)` | `PCLK1` |

**开错函数就完全不工作**（而且不报错），所以这一条要记牢：**只有 USART1 在 APB2，USART2/3 在 APB1。**

> [!warning] 别把「外设时钟」和「引脚上的时钟」搞混
> `RCC_APB2PeriphClockCmd(RCC_APB2Periph_USART1, ENABLE)` 开的是 **USART1 外设的工作时钟**（它同时也给波特率发生器提供了 `PCLK2` 的基准）。引脚属于 GPIOA，还要**另外**开 `RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOA, ENABLE)`。两个都要开。

### 7.2 引脚：PA9 / PA10

官方引脚定义表（`ground-truth\引脚定义_xlsx原文.txt`）逐格写明：

| 引脚号 | 名称 | 主功能 | 默认复用功能 | 重定义功能 |
| --- | --- | --- | --- | --- |
| 30 | PA9 | PA9 | **USART1_TX** / TIM1_CH2 | — |
| 31 | PA10 | PA10 | **USART1_RX** / TIM1_CH3 | — |
| 12 | PA2 | PA2 | **USART2_TX** / ADC12_IN2 / TIM2_CH3 | — |
| 13 | PA3 | PA3 | **USART2_RX** / ADC12_IN3 / TIM2_CH4 | — |
| 21 | PB10 | PB10 | I2C2_SCL / **USART3_TX** | TIM2_CH3 |
| 22 | PB11 | PB11 | I2C2_SDA / **USART3_RX** | TIM2_CH4 |
| 42 | PB6 | PB6 | I2C1_SCL / TIM4_CH1 | **USART1_TX** |
| 43 | PB7 | PB7 | I2C1_SDA / TIM4_CH2 | **USART1_RX** |

可见 USART1 的 `PA9` / `PA10` 属于**默认复用功能**——**不需要**开 AFIO 时钟，也**不需要** `GPIO_PinRemapConfig()`。只有搬到 `PB6` / `PB7` 才要做那两件事（参考手册框图上 `AFIO_MAPR` 的 `USART1_REMAP` 位就是干这个的，`stm32f10x.h` 里有 `AFIO_MAPR_USART1_REMAP`）。

### 7.3 GPIO 模式：TX 复用推挽，RX 输入

官方 `Serial.c`（9-2 版，收发都配）：

```c
GPIO_InitStructure.GPIO_Mode = GPIO_Mode_AF_PP;   // 复用推挽输出
GPIO_InitStructure.GPIO_Pin  = GPIO_Pin_9;         // PA9 = USART1_TX
GPIO_InitStructure.GPIO_Speed = GPIO_Speed_50MHz;
GPIO_Init(GPIOA, &GPIO_InitStructure);             // 将PA9引脚初始化为复用推挽输出

GPIO_InitStructure.GPIO_Mode = GPIO_Mode_IPU;      // 上拉输入
GPIO_InitStructure.GPIO_Pin  = GPIO_Pin_10;        // PA10 = USART1_RX
GPIO_InitStructure.GPIO_Speed = GPIO_Speed_50MHz;
GPIO_Init(GPIOA, &GPIO_InitStructure);             // 将PA10引脚初始化为上拉输入
```

| 引脚 | 方向 | 配置 | 为什么 |
| --- | --- | --- | --- |
| **PA9（TX）** | 输出 | `GPIO_Mode_AF_PP` **复用推挽** | 引脚的控制权必须交给 USART 外设。普通推挽输出时引脚由输出数据寄存器 `ODR` 控制，USART 内部波形再对也出不来；**复用**模式下 `ODR` 被断开，控制权才交给片上外设 |
| **PA10（RX）** | 输入 | `GPIO_Mode_IPU` **上拉输入** | 接收引脚是「读」的，不是「写」的。串口**空闲电平是高**，配上拉可以让引脚在悬空（没接东西）时就处于空闲态，避免悬空引入的随机跳变被当成起始位（乱进中断） |

> [!note] 官方两个工程在 GPIO 上的差别
> - `9-1 串口发送\User\main.c` → `Serial_Init()` 里**只配了 PA9**，`USART_Mode` 也只写 `USART_Mode_Tx`。这一集只需要发。
> - `9-2 串口发送+接收\Hardware\Serial.c` 里才**加上** PA10 的 `GPIO_Mode_IPU`，`USART_Mode` 改成 `USART_Mode_Tx | USART_Mode_Rx`，并加上 `RXNE` 中断与 NVIC。
>
> 也就是说：**9-1 的 `Serial.c` 是 9-2 的真子集**，9-2 是在 9-1 上「加接收」。

## 8 标准库配置步骤

```text
① RCC 开 USART1 时钟（APB2）+ GPIOA 时钟
② GPIO：TX → 复用推挽 AF_PP；RX（收发才有）→ 上拉输入 IPU
③ 填 USART_InitTypeDef（波特率 / 字长 / 停止位 / 校验 / 模式 / 流控）
④ USART_Init
⑤ 要收数据 → USART_ITConfig(USART1, USART_IT_RXNE, ENABLE)
⑥ 要收数据 → NVIC 分组 + NVIC_Init(USART1_IRQn)
⑦ USART_Cmd(USART1, ENABLE)
```

### 8.1 Serial.c / Serial.h（9-2 版，含接收）

```c
#include "stm32f10x.h"                  // Device header
#include <stdio.h>
#include <stdarg.h>

uint8_t Serial_RxData;		//定义串口接收的数据变量
uint8_t Serial_RxFlag;		//定义串口接收的标志位变量

/**
  * 函    数：串口初始化
  * 参    数：无
  * 返 回 值：无
  */
void Serial_Init(void)
{
	/*开启时钟*/
	RCC_APB2PeriphClockCmd(RCC_APB2Periph_USART1, ENABLE);	//开启USART1的时钟
	RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOA, ENABLE);	//开启GPIOA的时钟

	/*GPIO初始化*/
	GPIO_InitTypeDef GPIO_InitStructure;
	GPIO_InitStructure.GPIO_Mode = GPIO_Mode_AF_PP;
	GPIO_InitStructure.GPIO_Pin = GPIO_Pin_9;
	GPIO_InitStructure.GPIO_Speed = GPIO_Speed_50MHz;
	GPIO_Init(GPIOA, &GPIO_InitStructure);					//将PA9引脚初始化为复用推挽输出

	GPIO_InitStructure.GPIO_Mode = GPIO_Mode_IPU;
	GPIO_InitStructure.GPIO_Pin = GPIO_Pin_10;
	GPIO_InitStructure.GPIO_Speed = GPIO_Speed_50MHz;
	GPIO_Init(GPIOA, &GPIO_InitStructure);					//将PA10引脚初始化为上拉输入

	/*USART初始化*/
	USART_InitTypeDef USART_InitStructure;					//定义结构体变量
	USART_InitStructure.USART_BaudRate = 9600;				//波特率
	USART_InitStructure.USART_HardwareFlowControl = USART_HardwareFlowControl_None;	//硬件流控制，不需要
	USART_InitStructure.USART_Mode = USART_Mode_Tx | USART_Mode_Rx;	//模式，发送模式和接收模式均选择
	USART_InitStructure.USART_Parity = USART_Parity_No;		//奇偶校验，不需要
	USART_InitStructure.USART_StopBits = USART_StopBits_1;	//停止位，选择1位
	USART_InitStructure.USART_WordLength = USART_WordLength_8b;		//字长，选择8位
	USART_Init(USART1, &USART_InitStructure);				//将结构体变量交给USART_Init，配置USART1

	/*中断输出配置*/
	USART_ITConfig(USART1, USART_IT_RXNE, ENABLE);			//开启串口接收数据的中断

	/*NVIC中断分组*/
	NVIC_PriorityGroupConfig(NVIC_PriorityGroup_2);			//配置NVIC为分组2

	/*NVIC配置*/
	NVIC_InitTypeDef NVIC_InitStructure;					//定义结构体变量
	NVIC_InitStructure.NVIC_IRQChannel = USART1_IRQn;		//选择配置NVIC的USART1线
	NVIC_InitStructure.NVIC_IRQChannelCmd = ENABLE;			//指定NVIC线路使能
	NVIC_InitStructure.NVIC_IRQChannelPreemptionPriority = 1;		//指定NVIC线路的抢占优先级为1
	NVIC_InitStructure.NVIC_IRQChannelSubPriority = 1;		//指定NVIC线路的响应优先级为1
	NVIC_Init(&NVIC_InitStructure);							//将结构体变量交给NVIC_Init，配置NVIC外设

	/*USART使能*/
	USART_Cmd(USART1, ENABLE);								//使能USART1，串口开始运行
}

/**
  * 函    数：串口发送一个字节
  * 参    数：Byte 要发送的一个字节
  * 返 回 值：无
  */
void Serial_SendByte(uint8_t Byte)
{
	USART_SendData(USART1, Byte);		//将字节数据写入数据寄存器，写入后USART自动生成时序波形
	while (USART_GetFlagStatus(USART1, USART_FLAG_TXE) == RESET);	//等待发送完成
	/*下次写入数据寄存器会自动清除发送完成标志位，故此循环后，无需清除标志位*/
}

/**
  * 函    数：使用printf需要重定向的底层函数
  * 参    数：保持原始格式即可，无需变动
  * 返 回 值：保持原始格式即可，无需变动
  */
int fputc(int ch, FILE *f)
{
	Serial_SendByte(ch);			//将printf的底层重定向到自己的发送字节函数
	return ch;
}

/**
  * 函    数：获取串口接收标志位
  * 参    数：无
  * 返 回 值：串口接收标志位，范围：0~1，接收到数据后，标志位置1，读取后标志位自动清零
  */
uint8_t Serial_GetRxFlag(void)
{
	if (Serial_RxFlag == 1)			//如果标志位为1
	{
		Serial_RxFlag = 0;
		return 1;					//则返回1，并自动清零标志位
	}
	return 0;						//如果标志位为0，则返回0
}

/**
  * 函    数：获取串口接收的数据
  * 参    数：无
  * 返 回 值：接收的数据，范围：0~255
  */
uint8_t Serial_GetRxData(void)
{
	return Serial_RxData;			//返回接收的数据变量
}

/**
  * 函    数：USART1中断函数
  * 参    数：无
  * 返 回 值：无
  * 注意事项：此函数为中断函数，无需调用，中断触发后自动执行
  *           函数名为预留的指定名称，可以从启动文件复制
  *           请确保函数名正确，不能有任何差异，否则中断函数将不能进入
  */
void USART1_IRQHandler(void)
{
	if (USART_GetITStatus(USART1, USART_IT_RXNE) == SET)		//判断是否是USART1的接收事件触发的中断
	{
		Serial_RxData = USART_ReceiveData(USART1);				//读取数据寄存器，存放在接收的数据变量
		Serial_RxFlag = 1;										//置接收标志位变量为1
		USART_ClearITPendingBit(USART1, USART_IT_RXNE);			//清除USART1的RXNE标志位
																//读取数据寄存器会自动清除此标志位
																//如果已经读取了数据寄存器，也可以不执行此代码
	}
}
```

`Serial.h` 对应地多了两行（本节完整源码见官方工程 `9-2 串口发送+接收\Hardware\Serial.c` / `Serial.h`；本页略去与 9-1 完全相同的 `Serial_SendArray` / `Serial_SendString` / `Serial_SendNumber` / `Serial_Printf`，那四个函数 9-1 与 9-2 逐字相同）：

```c
void Serial_Init(void);
void Serial_SendByte(uint8_t Byte);
void Serial_SendArray(uint8_t *Array, uint16_t Length);
void Serial_SendString(char *String);
void Serial_SendNumber(uint32_t Number, uint8_t Length);
void Serial_Printf(char *format, ...);

uint8_t Serial_GetRxFlag(void);
uint8_t Serial_GetRxData(void);
```

### 8.2 main.c：回环测试

官方 `9-2 串口发送+接收\User\main.c`：

```c
#include "stm32f10x.h"                  // Device header
#include "Delay.h"
#include "OLED.h"
#include "Serial.h"

uint8_t RxData;			//定义用于接收串口数据的变量

int main(void)
{
	/*模块初始化*/
	OLED_Init();		//OLED初始化

	/*显示静态字符串*/
	OLED_ShowString(1, 1, "RxData:");

	/*串口初始化*/
	Serial_Init();		//串口初始化

	while (1)
	{
		if (Serial_GetRxFlag() == 1)			//检查串口接收数据的标志位
		{
			RxData = Serial_GetRxData();		//获取串口接收的数据
			Serial_SendByte(RxData);			//串口将收到的数据回传回去，用于测试
			OLED_ShowHexNum(1, 8, RxData, 2);	//显示串口接收的数据
		}
	}
}
```

现象：串口助手发什么，OLED 上就显示什么（十六进制两位），同时串口助手也收到同样的字节——**收到就原样发回去**，所以叫回环 / 回声测试。

> [!tip] 中断只搬数据，主循环才干活
> 注意 `USART1_IRQHandler()` 里**只做了三件事**：读走数据、置标志位、清标志。所有耗时的事（发回串口、刷 OLED）都留给主循环。这是中断编程的通用纪律（第 5、6 章反复强调过）——中断里干多了会阻塞其他中断，还会拖长本次响应。

### 8.3 9-1 与 9-2 的源码差异一览

| 项目 | 9-1 串口发送 | 9-2 串口发送+接收 |
| --- | --- | --- |
| GPIO | 只配 PA9（`AF_PP`） | PA9（`AF_PP`）+ PA10（`IPU`） |
| `USART_Mode` | `USART_Mode_Tx` | `USART_Mode_Tx \| USART_Mode_Rx` |
| `USART_ITConfig` | 无 | `USART_IT_RXNE` |
| NVIC | 无 | `USART1_IRQn`，抢占 1 / 响应 1，分组 2 |
| `Serial.h` 接口 | 只有发送 + printf 相关 | 多 `Serial_GetRxFlag()`、`Serial_GetRxData()` |
| `USART1_IRQHandler` | 无 | 有 |
| `main.c` | 一次性发一个数组 `{0x42,0x43,0x44,0x45}` | 主循环里回环 |
| 其余 | — | `Serial.c` 其余函数与 9-1 逐字相同 |

## 9 易错点

- [ ] 开了 `RCC_APB1PeriphClockCmd(RCC_APB1Periph_USART1, ...)` → 常量名不存在，而且 USART1 在 **APB2**。
- [ ] 只开了 USART1 时钟、忘了开 GPIOA 时钟 → 引脚不工作。
- [ ] **TX 配成 `GPIO_Mode_Out_PP`（普通推挽）** → 引脚控制权在 `ODR` 手里，波形出不来。必须是 `GPIO_Mode_AF_PP`。
- [ ] **RX 配成 `GPIO_Mode_AF_PP`** → 那是输出模式，引脚不会听外面的信号。RX 要配 `IPU` 或浮空输入。
- [ ] 只有 USART1 用 `RCC_APB2Periph_*`，写成 `APB1` 的名字编译不过、写成对的函数但传错外设则无声无息。
- [ ] 写了 `USART_ITConfig()` 却忘了 `NVIC_Init()` → 死活不进中断（和第 6 章定时器同一个坑）。
- [ ] 中断服务函数名写错（比如 `USART1_IRQhandler`、`Usart1_IRQHandler`）→ 链接不报错，中断永远不进。**这个名字要和启动文件里的一致**。
- [ ] 中断里忘了清 `RXNE` → 反复进出中断。不过官方注释也提醒：**读 `DR` 本身就会自动清 `RXNE`**，所以读过了再手动清是「保险」而不是「必须」。
- [ ] 以为要等 `TC` 才能发下一个字节 → 等 `TXE` 就够了，`TXE` 来得更早，发连续字节效率更高。
- [ ] 收发共用一个 `DR` 就以为「读和写会打架」→ 外设内部是 TDR / RDR **两套**寄存器，全双工同时收发不冲突。
- [ ] `USART_Mode` 只写 `USART_Mode_Tx` 却想接收 → `RE` 没使能，RX 引脚上的数据根本不进 RDR。
- [ ] 把 `USART_Mode_Tx | USART_Mode_Rx` 当成「写一个常量」→ 它是按位或组合出来的。
- [ ] 发送循环里写成 `USART_GetITStatus(..., USART_IT_TXE)` → 那是**中断**标志的判断函数；`Serial_SendByte` 里用的是 `USART_GetFlagStatus(..., USART_FLAG_TXE)`，两者不是一回事。

## 10 自测

1. 为什么 CPU 写完 `TDR` 就能走，不用等数据真的从 TX 发出去？
2. `TXE` 和 `TC` 分别在什么时候置 1？发连续字节应该等哪个？
3. `RXNE` 怎么清？读了 `DR` 之后还要不要手动调 `USART_ClearITPendingBit()`？
4. 开 USART1 的时钟用哪个函数？USART2 呢？为什么不一样？
5. PA9 为什么必须配成 `GPIO_Mode_AF_PP`，配成 `GPIO_Mode_Out_PP` 会怎样？
6. `USARTDIV = 468.75` 时，`BRR` 该写什么？
7. `USART1_IRQHandler()` 里为什么只置标志位、不发数据不刷 OLED？

> [!success]- 参考答案
> 1. 因为 TDR 后面还有一级**发送移位寄存器**。写进 TDR 后硬件立刻把它搬进移位寄存器，然后一位一位慢慢发；TDR 空了 CPU 就可以写下一个字节。
> 2. `TXE` 在 TDR 的数据被搬进移位寄存器后置 1（「能写下一个了」）；`TC` 在整帧（含停止位）全部发完才置 1（「线真的空了」）。发连续字节**等 `TXE`**。
> 3. **读一次 `RDR`（也就是读 `DR`）就会自动清 `RXNE`**。官方代码在读完之后仍然调了 `USART_ClearITPendingBit()`，注释里说明这是「也可以不执行」的保险动作。
> 4. USART1 用 `RCC_APB2PeriphClockCmd(RCC_APB2Periph_USART1, ENABLE)`；USART2/3 用 `RCC_APB1PeriphClockCmd(...)`。因为 USART1 挂在 **APB2**，USART2/3 挂在 **APB1**。
> 5. 因为 TX 是**输出**，而且必须由 USART 外设来驱动。`GPIO_Mode_AF_PP` 是复用推挽，会把引脚控制权从 `ODR` 交给片上外设；写成 `GPIO_Mode_Out_PP`，引脚只听 `ODR`，USART 内部生成的波形到不了引脚。
> 6. 整数部分 468、小数部分 `0.75 × 16 = 12`，所以 `BRR = (468 << 4) | 12 = 0x1D4C`。
> 7. 中断应当尽量短。中断里干多了会阻塞其他中断、拉长响应时间；而且回环要发数据、刷 OLED 都很慢。正确做法是中断里只「读走数据 + 置标志」，主循环看到标志再干活。

## 11 待核对

- [ ] **USART1 最高 4.5 Mbits/s** 取自课件 Slide 114；官方数据手册中该外设的实际上限与总线/过采样设置有关，本页按课件口径写，未在数据手册中逐条复核。
- [ ] 课件 Slide 115「USART 框图」用的是官方参考手册图 248，图上所有分支（IrDA、智能卡、LIN、同步时钟）本课程均未使用；老师在视频里是否逐块讲解过，不在课件文本里。
- [ ] 老师讲解「为什么 GPIO 要配复用推挽」的原话。
- [ ] `USART_HardwareFlowControl` 中 `RTS` / `CTS` 两档对应的引脚（PA11/PA12 等）本课程未涉及，未在笔记中展开。

## 核对记录（2026-09-26）

> [!success] 核对依据
> - 官方配套源码：`STM32Project-有注释版\9-2 串口发送+接收\Hardware\Serial.c`、`Hardware\Serial.h`、`User\main.c`；对照 `9-1 串口发送\Hardware\Serial.c`、`Hardware\Serial.h`、`User\main.c`
> - 官方接线图：`ground-truth\接线图\9-1 串口发送.png`（STLINK 与 USB 转串口模块两条独立链路）
> - 课程课件文本：`ground-truth\课件文本.md`（Slide 114 USART 简介、115 USART 框图、116 USART 基本结构、121 波特率发生器）
> - 课件配图（PPT 内嵌的官方参考手册图，`ground-truth\ppt_x\ppt\media\`）：`image110.jpeg` = 图 248「USART 框图」（含 `USART_BRR` 的 `DIV_Mantissa`/`DIV_Fraction` 与 `USARTDIV = DIV_Mantissa + (DIV_Fraction / 16)`）、`image111.png` = 图 249「字长设置」、`image112.png` = 图 250「配置停止位」、`image113.png` = 图 252「起始位侦测」、`image114.png` = 图 253「检测噪声的数据采样」、`image115.png` = 图 251「波特率寄存器（USART_BRR）」
> - 官方引脚定义表：`ground-truth\引脚定义_xlsx原文.txt`
> - ST 标准外设库：`STM32F10x_StdPeriph_Lib_V3.5.0\...\inc\stm32f10x_usart.h`（`USART_InitTypeDef`、各枚举值、`USART_IT_*`、`USART_FLAG_*`、函数原型）、`...\src\stm32f10x_usart.c`（`USART_Init` 波特率分频算法）、`...\inc\stm32f10x_rcc.h`（USART 时钟常量）、`...\inc\stm32f10x_gpio.h`（`GPIO_Mode_AF_PP`/`IPU`）

| 核对项 | 笔记原值 | 官方依据 | 结论 |
| --- | --- | --- | --- |
| USART 定义 | 通用同步/异步收发器，可自动生成/拼接数据帧 | 课件 Slide 114 逐字一致 | 一致 |
| 最高波特率 | 4.5 Mbits/s | 课件 Slide 114「自带波特率发生器，最高达 4.5Mbits/s」 | 一致 |
| C8T6 的 USART 资源 | USART1、USART2、USART3 | 课件 Slide 114 逐字一致 | 一致 |
| 框图四块划分 | 发送部分 / 接收部分 / 波特率发生器 / 控制逻辑 | 课件 Slide 116「发送数据寄存器 TDR／发送移位寄存器／接收数据寄存器 RDR／接收移位寄存器／发送控制器／接收控制器／波特率发生器／PCLK2/1／开关控制」；官方参考手册图 248 | 一致 |
| 数据寄存器是两套 | TDR 与 RDR 独立，`DR` 只是同名偏移 | 官方参考手册图 248 中「发送数据寄存器(TDR)」与「接收数据寄存器(RDR)」分别处于写/读支路；`stm32f10x_usart.c` 中 `USART_SendData`/`USART_ReceiveData` 访问同一 `DR` 偏移 | 一致 |
| `TXE` 含义 | 发送数据寄存器空 | `stm32f10x_usart.h` 第 323 行 `USART_FLAG_TXE`；图 248 `SR` 一格含 `TXE` | 一致 |
| `TC` 含义 | 发送完成 | `stm32f10x_usart.h` 第 324 行 `USART_FLAG_TC`；图 248 `SR` 含 `TC` | 一致 |
| `RXNE` 含义 | 读数据寄存器非空 | `stm32f10x_usart.h` 第 325 行 `USART_FLAG_RXNE`；图 248 `SR` 含 `RXNE` | 一致 |
| `IDLE` 含义 | 总线空闲 | `stm32f10x_usart.h` 第 326 行 `USART_FLAG_IDLE`；图 248 `SR` 含 `IDLE` | 一致 |
| 清标志方式 | 写 TDR 清 `TXE`，读 RDR 清 `RXNE` | 官方 `Serial.c` 第 46、196~197 行注释：「下次写入数据寄存器会自动清除发送完成标志位」「读取数据寄存器会自动清除此标志位」 | 一致 |
| 波特率公式 | `波特率 = f_PCLK / (16 × DIV)` | 课件 Slide 121 逐字一致；`stm32f10x_usart.h` 第 54 行 `IntegerDivider = ((PCLKx) / (16 * USART_BaudRate))` | 一致 |
| `BRR` 位定义 | `DIV_Mantissa` 15:4（12 位）、`DIV_Fraction` 3:0（4 位） | 官方参考手册图 251（`image115.png`）逐位一致 | 一致 |
| `USARTDIV` 公式 | `DIV_Mantissa + (DIV_Fraction / 16)` | 图 248 右下角原文 `USARTDIV = DIV_Mantissa + (DIV_Fraction / 16)` | 一致 |
| `USART_InitTypeDef` 六个成员 | `USART_BaudRate` / `WordLength` / `StopBits` / `Parity` / `Mode` / `HardwareFlowControl` | `stm32f10x_usart.h` 第 50~76 行，成员名与顺序一致 | 一致 |
| 六个成员的枚举值 | `_8b`=0x0000 / `_1`=0x0000 / `_No`=0x0000 / `_Rx`=0x0004 / `_Tx`=0x0008 / `_None`=0x0000 | `stm32f10x_usart.h` 第 125、138、154、168~169、178 行，逐个一致 | 一致 |
| `USART_Mode` 用按位或组合 | `USART_Mode_Tx \| USART_Mode_Rx` | 官方 `9-2\Serial.c` 第 35 行原文；两个常量分占 bit2 / bit3（`stm32f10x_usart.h` 第 168~169 行） | 一致 |
| USART1 在 APB2 | 是 | `stm32f10x_rcc.h` 第 510 行 `RCC_APB2Periph_USART1`；官方 `Serial.c` 第 13 行 `RCC_APB2PeriphClockCmd` | 一致 |
| USART2 / USART3 在 APB1 | 是 | `stm32f10x_rcc.h` 第 540~541 行 `RCC_APB1Periph_USART2` / `RCC_APB1Periph_USART3` | 一致 |
| TX / RX 引脚 | PA9 / PA10 | 引脚定义表第 32~33 行 `PA9 = USART1_TX`、`PA10 = USART1_RX`；官方 `Serial.c` 第 19、22、27 行 `GPIO_Pin_9` / `GPIO_Pin_10` | 一致 |
| 其他 USART 引脚 | USART2 = PA2/PA3，USART3 = PB10/PB11 | 引脚定义表第 14~15、23~24 行 | 一致 |
| USART1 重映射引脚 | PB6 / PB7 | 引脚定义表第 44~45 行「重定义功能」列 | 一致 |
| TX 配复用推挽 | `GPIO_Mode_AF_PP` | 官方 `9-1\Serial.c` 第 18、21 行、`9-2\Serial.c` 第 21、24 行；`stm32f10x_gpio.h` 第 79 行 `GPIO_Mode_AF_PP = 0x18` | 一致 |
| RX 配输入 | `GPIO_Mode_IPU`（上拉输入） | 官方 `9-2\Serial.c` 第 26、29 行「将PA10引脚初始化为上拉输入」；`stm32f10x_gpio.h` 第 75 行 `GPIO_Mode_IPU = 0x48` | 一致 |
| 中断服务函数名 | `USART1_IRQHandler` | 官方 `9-2\Serial.c` 第 189 行；NVIC 向量枚举 `USART1_IRQn = 37`（工程 `Start\stm32f10x.h`） | 一致 |
| 中断配置 | `USART_ITConfig(USART1, USART_IT_RXNE, ENABLE)` + NVIC 分组 2、抢占 1 / 响应 1 | 官方 `9-2\Serial.c` 第 42、45、49~53 行，逐项一致 | 一致 |
| `USART_GetITStatus` / `USART_IT_RXNE` 用法 | 在 `USART1_IRQHandler` 里判断 | 官方 `9-2\Serial.c` 第 191 行；`stm32f10x_usart.h` 第 392 行原型、第 245 行 `USART_IT_RXNE` | 一致 |
| `Serial_SendByte` 等 `TXE` 等待 | `while (USART_GetFlagStatus(USART1, USART_FLAG_TXE) == RESET);` | 官方 `9-2\Serial.c` 第 66~68 行（含第 68 行注释「下次写入数据寄存器会自动清除发送完成标志位，故此循环后，无需清除标志位」） | 一致 |
| `USART_ReceiveData` 读数据 | 中断里调用 | 官方 `9-2\Serial.c` 第 193 行 | 一致 |
| 9-1 与 9-2 的 `Serial.c` 差异 | 9-1 只配 PA9 / 只 `USART_Mode_Tx` / 无中断；9-2 加 PA10 / 双模式 / `RXNE` 中断 / `USART1_IRQHandler` | 逐行比对两个工程的 `Serial.c`：9-1 共 132 行无接收相关代码，9-2 共 199 行；两者共有的发送类函数逐字相同 | 一致 |
| main.c 行为 | 9-1 在 `main()` 里发一次数组 `{0x42,0x43,0x44,0x45}`（其余发送/printf 调用在官方源码里都是注释）；9-2 主循环回环 | 官方 `9-1\User\main.c` 第 16~17 行（数组定义与 `Serial_SendArray`）、第 19~36 行为注释；官方 `9-2\User\main.c` 第 14、24、25 行 | 一致（笔误已改：**9-1 的 `Serial.c` 里没有 `main()`**，发数组是 `main.c` 干的） |
| 串口链路 | USB 转串口模块，非 STLINK 串口 | 官方接线图 `9-1 串口发送.png`：STLINK 只接 SWDIO/SWCLK/RST/3.3V/GND，USB 转串口模块单独有 RXD/TXD/GND/5V/VCC 一路 | 一致（本页据此写「本课程用 USB 转串口模块」） |

> [!note] 出处说明
> 本页的 USART 定义与能力、框图结构、`BRR` 与 `USARTDIV`、中断与 GPIO 复用等，已逐条对照课程官方配套源码（`9-1`、`9-2` 两个工程的 `Hardware\Serial.c`、`Hardware\Serial.h`、`User\main.c`）、官方接线图、课程课件文本（Slide 114~116、121）与 PPT 内嵌的官方参考手册图（图 248~253）核对；库函数名、结构体成员名与枚举值（`USART_Mode_Tx`、`USART_FLAG_TXE`、`USART_IT_RXNE`、`GPIO_Mode_AF_PP`、`GPIO_Mode_IPU`、`RCC_APB2Periph_USART1` 等）已在 ST 标准外设库 `stm32f10x_usart.h`、`stm32f10x_gpio.h`、`stm32f10x_rcc.h` 中逐个查到；引脚以 `ground-truth\引脚定义_xlsx原文.txt` 与官方 `Serial.c` 为准。
> **仍未核实**：老师关于「为什么 GPIO 要配复用推挽」等口头讲解原话、视频里的演示画面细节、以及「最高 4.5 Mbits/s」在官方数据手册中的逐条对应——这些不在源码、接线图与课件文本中。相关条目已留在上面的「待核对」。
