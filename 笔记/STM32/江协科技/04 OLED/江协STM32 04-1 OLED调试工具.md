---
course: 江协科技 STM32入门教程-2023版
chapter: 04-1 OLED调试工具
type: 分集笔记
source: https://www.bilibili.com/video/BV1th411z7sn?p=9
tags:
  - STM32
  - 江协科技
  - OLED
  - I2C
  - 调试
status: draft
verify: 官方源码+接线图+课件
updated: 2026-09-26
---

# 04-1 OLED 调试工具

> [!abstract] 这一集只解决一个问题
> **写好的程序看不见内部状态，怎么才能一眼看出变量对不对？**
> 答：接一块 0.96 寸 OLED（Organic Light Emitting Diode，有机发光二极管）显示屏，把厂商写好的驱动文件加进工程，调一次 `OLED_Init()`，屏幕就亮了——**从这一集起，调试不再靠猜**。
>
> 本集**不自己写驱动**，重点是：认模块、认接线、认 I2C 时序、会移植。
>
> 上一章 [[江协STM32 03 GPIO（章索引）]] ｜ 章索引 [[江协STM32 04 OLED（章索引）]] ｜ 下一集 [[江协STM32 04-2 OLED显示屏]]

## 1 为什么需要 OLED：五种调试手段对比

先看清各手段的短板，才知道为什么要多接一块屏。

| 调试方式 | 做法 | 优点 | 短板 |
| --- | --- | --- | --- |
| **串口调试** | 调试信息发给电脑，用串口助手看 | 信息量大、能打印文字 | **要开电脑、要接线**，程序还得先把串口跑通；实时性差 |
| **显示屏调试** | 屏直接接单片机，信息打在屏上 | **随身、实时、不占串口** | 屏幕小，能显示的内容有限 |
| **Keil 调试模式** | 单步、断点、看寄存器/变量 | 能看到**全部**变量和外设寄存器 | **必须连着仿真器、程序被停下**，看不了"一直在跑"的动态过程 |
| **点灯调试法** | 在怀疑的位置插一句点灯，跑到灯就亮 | 零成本、极简单 | 只能表达"到没到"，**表达不了数值** |
| **注释/对照调试法** | 把新加的代码全部注释，逐行放开；或拿一份正常程序对比 | 适合定位"哪一行引入的 bug" | 靠二分法硬试，**慢** |

> [!tip] 调试的通用方法论
> 所有调试手段背后的思想是一样的：**缩小范围、控制变量、对比测试**。
> 所以选工具不是看哪个高级，而是看**哪一种能最快把范围缩小一半**。

**OLED 在这门课里的定位**：从第 5 章开始，几乎每一集的实验都会在屏幕上打印变量——`OLED_ShowNum()` 之于调试，相当于 `printf()` 之于 PC 编程，但它**不占用串口、不需要电脑、程序照常全速运行**。

## 2 OLED 模块硬件

### 2.1 模块规格

| 项目 | 参数 |
| --- | --- |
| 尺寸 | 0.96 寸 |
| 分辨率 | **128 × 64** 像素 |
| 驱动芯片 | **SSD1306**（兼容 SSD1315；1.3 寸版用 SH1106） |
| 供电 | 模块标注 **3 ~ 5.5 V** |
| 通信协议 | **I2C** 或 **SPI**（二选一，看买的是哪个版本） |
| 像素颜色 | 白色 / 蓝色 / 黄蓝双色 |

**黄蓝双色版**比较特别：屏幕**上面 1/4 固定黄色、下面 3/4 固定蓝色**，正好可以当"标题行 + 内容区"用。无论哪个颜色版本，**驱动方式完全一样**。

### 2.2 四针 I2C 版 vs 七针 SPI 版

这是本集第一个容易买错/接错的地方：

| | **4 针 I2C 版** | **7 针 SPI 版** |
| --- | --- | --- |
| 引脚数 | 4 | 7 |
| 通信引脚 | **SCL**（时钟）、**SDA**（数据） | SPI 的时钟、数据、片选等 |
| 占用 IO | **少**（2 根） | 多 |
| 本课程用法 | ✅ **本课程用这个** | 课程不涉及 |
| 接线自由度 | 用**软件 I2C** 时，SCL/SDA **可接任意 GPIO** | 用软件模拟时同样可任意接 |

- **4 针版**引脚为 `GND` / `VCC` / `SCL` / `SDA`。
- **7 针版**除 `GND` / `VCC` 外的引脚是 SPI 通信引脚。

> [!warning] 关键前提：为什么"可以接任意 GPIO"
> 这句话只在使用**软件 I2C**（用两个普通 GPIO 手动翻电平来模拟时序）时成立。
> 如果换成 STM32 内部的**硬件 I2C 外设**，SCL/SDA 就**必须**落在芯片指定的 I2C 复用引脚上。本课程用的是软件 I2C，所以自由。

### 2.3 本课程驱动默认的引脚

课程配套的 `OLED.c` 里，引脚由两个宏决定：

```c
/*引脚配置*/
#define OLED_W_SCL(x)   GPIO_WriteBit(GPIOB, GPIO_Pin_8, (BitAction)(x))  //scl引脚
#define OLED_W_SDA(x)   GPIO_WriteBit(GPIOB, GPIO_Pin_9, (BitAction)(x))  //sda引脚
```

即**驱动默认把 SCL 接在 PB8、SDA 接在 PB9**。想换引脚，改宏和下面的 `OLED_I2C_Init()` 即可。

> [!note] 官方两个工程里的驱动是同一个
> `4-1 OLED显示屏\Hardware\OLED.c` 与 `1-4 OLED驱动函数模块\4针脚I2C版本\OLED.c` 的引脚配置完全一致（都是 **PB8 = SCL、PB9 = SDA**，开漏输出），没有差异。
> 另外 `1-4 OLED驱动函数模块` 下另有一个 **7 针脚 SPI 版本**的驱动，引脚不同（`D0` PB12、`D1` PB13、`RES` PB14、`DC` PB15、`CS` PA8，**推挽输出** `GPIO_Mode_Out_PP`）——课程不用这个版本。

> [!note] 关于"屏靠 IO 供电"
> 4 针模块功率很小，一种**不规范但能用**的做法是把 `VCC`/`GND` 也接到 GPIO 上、由端口直接供电（把供电脚当普通输出用）；正式项目务必接真实电源。
> **依据范围**：官方接线图 `4-1 OLED显示屏.png` 中，OLED 的 `VCC`/`GND` 是接在面包板电源轨上的，**没有**看到"由 GPIO 供电"的接法（接线图未标注电源轨来自哪个引脚）。因此"课程中还有一种接法……"这句**未在官方资料中核实**，仅作为排错提示保留。

## 3 I2C 通信基础

I2C（Inter-Integrated Circuit，集成电路总线）只用**两根线**就能挂多个设备：

| 线 | 名字 | 作用 |
| --- | --- | --- |
| **SCL** | Serial Clock，串行时钟 | 主机产生，决定每一位的节拍 |
| **SDA** | Serial Data，串行数据 | 双向数据线，**一个时钟位只传 1 bit** |

两根线都要**上拉电阻**，平时为高电平；**靠"拉低"来表达信息**——这也是后面 `OLED_I2C_Init()` 把引脚配成**开漏输出**（`GPIO_Mode_Out_OD`）的原因。

> [!note] 依据范围
> 课件《I2C 硬件电路》页写明：「SCL 和 SDA 各添加一个上拉电阻，阻值一般为 **4.7KΩ** 左右」。但**官方课件与源码都没有说明本课程这块 OLED 模块板上的上拉电阻是否为 4.7 kΩ**（有公开笔记称板载 4.7 kΩ，未见官方出处），因此本笔记不对模块板载阻值下结论。

### 3.1 三个最基本的时序

**起始条件（Start）**：SCL 为高时，SDA **由高变低**。

```text
SDA ─────┐
         └────────      高→低，且此时 SCL 为高
SCL ────────────
        （SCL 保持高电平）
```

**停止条件（Stop）**：SCL 为高时，SDA **由低变高**。

```text
SDA          ┌──────
        ─────┘          低→高，且此时 SCL 为高
SCL ────────────
```

**发送一个字节**：拉低 SCL → 放好 SDA 电平 → **拉高 SCL（从机在这时采样）** → 拉低 SCL，重复 8 次。8 位发完后，第 9 个时钟是**应答位（ACK）**，由从机拉低 SDA 表示"收到了"。

课程配套代码里这个函数写得很直白：

```c
/**
  * @brief  I2C发送一个字节
  * @param  Byte 要发送的一个字节
  */
void OLED_I2C_SendByte(uint8_t Byte)
{
	uint8_t i;
	for (i = 0; i < 8; i++)
	{
		OLED_W_SDA(!!(Byte & (0x80 >> i)));   // 从最高位开始，把第 i 位放上 SDA
		OLED_W_SCL(1);                        // 拉高 SCL，从机此刻采样这一位
		OLED_W_SCL(0);                        // 拉低 SCL，准备下一位
	}
	OLED_W_SCL(1);	//额外的一个时钟，不处理应答信号
	OLED_W_SCL(0);
}
```

> [!warning] 本课程"不读应答"
> 上面最后两行**只补了一个时钟节拍，没有去读 SDA 上的 ACK**。因为 SSD1306 在串行模式下**只写不读**，课程为了代码简单就跳过了应答判断。
> 这不是 I2C 的完整实现。到第 10 章 `10-3 软件I2C读写MPU6050` 时，因为要**读**数据，就必须老老实实把 ACK 判断和"读一个字节"补齐。

### 3.2 SSD1306 怎么区分"命令"和"数据"

I2C 只有一根数据线，没地方再拉一根 `D/C` 线。SSD1306 的约定是：**在起始条件后连发两个字节**——先发**从机地址（Slave Address）**，再发一个**控制字节**，用它的值来区分后面跟的是命令还是数据。

```c
/**
  * @brief  OLED写命令
  * @param  Command 要写入的命令
  */
void OLED_WriteCommand(uint8_t Command)
{
	OLED_I2C_Start();
	OLED_I2C_SendByte(0x78);		//从机地址
	OLED_I2C_SendByte(0x00);		//写命令
	OLED_I2C_SendByte(Command);
	OLED_I2C_Stop();
}

/**
  * @brief  OLED写数据
  * @param  Data 要写入的数据
  */
void OLED_WriteData(uint8_t Data)
{
	OLED_I2C_Start();
	OLED_I2C_SendByte(0x78);		//从机地址
	OLED_I2C_SendByte(0x40);		//写数据
	OLED_I2C_SendByte(Data);
	OLED_I2C_Stop();
}
```

| 第 2 个字节 | 含义 |
| --- | --- |
| `0x00` | 后面跟的是**命令**（Command） |
| `0x40` | 后面跟的是**数据**（Data） |

> [!note] 从机地址 `0x78` 是怎么来的
> SSD1306 数据手册给的从机地址是 **`0x3C`**（`D/C` 脚接地）或 **`0x3D`**（接高电平）。`0x78` 是 **`0x3C` 左移一位**后的形式——I2C 传输时地址占 7 位、第 8 位是读写方向位，所以"传输用的字节"比"手册上的地址"大一位。模块 PCB 上标注的地址是 `0x3C`/`0x3D`，代码里写的是移位后的 `0x78`/`0x7A`，**两者说的是同一个设备**，不要以为是矛盾。

## 4 驱动代码的移植步骤

移植（Porting）就是把厂商给的驱动文件"搬"进自己的工程。**三步**：

### 4.1 第一步：把文件放进工程

厂商提供 **3 个文件**，全部放进工程的 `Hardware` 文件夹：

| 文件 | 内容 |
| --- | --- |
| `OLED.c` | 引脚宏定义、`OLED_I2C_Init()`、I2C 时序、显示函数、`OLED_Init()` 初始化命令序列 |
| `OLED.h` | 8 个显示函数的声明 |
| `OLED_Font.h` | 字模库 `OLED_F8x16[][]`，ASCII 可见字符的 8×16 点阵 |

然后在 Keil 的工程树里，把 `OLED.c` **加入编译**（只加 `.c`，`.h` 会被自动包含）。

### 4.2 第二步：接线与（可选）改引脚

按默认接线把 4 针接到 **PB8（SCL）/ PB9（SDA）**，`VCC`、`GND` 接电源（官方接线图中该两脚接到面包板电源轨）。若改用其他引脚，需要同时修改两处：

```c
/* ① 改引脚宏 —— 决定"翻哪个引脚" */
#define OLED_W_SCL(x)   GPIO_WriteBit(GPIOB, GPIO_Pin_8, (BitAction)(x))
#define OLED_W_SDA(x)   GPIO_WriteBit(GPIOB, GPIO_Pin_9, (BitAction)(x))

/* ② 改端口初始化 —— 决定"初始化哪个引脚" */
void OLED_I2C_Init(void)
{
    RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOB, ENABLE);   // 开 GPIOB 时钟

	GPIO_InitTypeDef GPIO_InitStructure;
 	GPIO_InitStructure.GPIO_Mode = GPIO_Mode_Out_OD;        // 开漏输出，I2C 的硬件要求
	GPIO_InitStructure.GPIO_Speed = GPIO_Speed_50MHz;
	GPIO_InitStructure.GPIO_Pin = GPIO_Pin_8;               // SCL
 	GPIO_Init(GPIOB, &GPIO_InitStructure);
	GPIO_InitStructure.GPIO_Pin = GPIO_Pin_9;               // SDA
 	GPIO_Init(GPIOB, &GPIO_InitStructure);

	OLED_W_SCL(1);                                          // 空闲时两根线都是高电平
	OLED_W_SDA(1);
}
```

> [!warning] 改引脚要**两处一起改**
> `#define` 决定运行时翻哪个引脚，`GPIO_Init` 决定哪个引脚被配成开漏输出。
> 只改一处 → 要么翻不动（引脚没初始化），要么翻的是别的引脚（宏没改）。这是移植最常见的哑火原因。

### 4.3 第三步：在自己的代码里调用

```c
#include "stm32f10x.h"      // Device header
#include "OLED.h"           // ① 引用驱动头文件

int main(void)
{
	OLED_Init();            // ② 初始化 OLED（内部含上电延时、SSD1306 命令序列、清屏）

	OLED_ShowChar(1, 1, 'A');   // ③ 在第 1 行第 1 列显示字符 A
	                            //    字符要用【单引号】，双引号是字符串
	while (1)
	{

	}
}
```

现象：**屏幕第 1 行第 1 列出现一个 `A`**。

> [!note] 为什么 `OLED_Init()` 里要"先延时再发命令"
> `OLED_Init()` 开头有一段空循环延时：
>
> ```c
> 	for (i = 0; i < 1000; i++)			//上电延时
> 	{
> 		for (j = 0; j < 1000; j++);
> 	}
> ```
>
> SSD1306 上电后需要一段时间才能稳定接收命令；**上电太早发命令会导致初始化失败（黑屏或花屏）**。这段延时就是等它。

## 5 完整代码

`main.c`（本集的最小可运行版本）：

```c
#include "stm32f10x.h"      // Device header
#include "OLED.h"           // OLED 驱动头文件

int main(void)
{
	OLED_Init();            // 初始化：引脚 → I2C 时序 → SSD1306 命令序列 → 清屏

	OLED_ShowChar(1, 1, 'A');   // 第 1 行第 1 列，显示字符 'A'

	while (1)               // 显示内容会一直保持，主循环空转即可
	{

	}
}
```

`OLED.h`（厂商提供的声明，共 8 个函数，以下为官方 `Hardware\OLED.h` 原文）：

```c
#ifndef __OLED_H
#define __OLED_H

void OLED_Init(void);//初始化oled
void OLED_Clear(void);
void OLED_ShowChar(uint8_t Line, uint8_t Column, char Char);
void OLED_ShowString(uint8_t Line, uint8_t Column, char *String);
void OLED_ShowNum(uint8_t Line, uint8_t Column, uint32_t Number, uint8_t Length);
void OLED_ShowSignedNum(uint8_t Line, uint8_t Column, int32_t Number, uint8_t Length);
void OLED_ShowHexNum(uint8_t Line, uint8_t Column, uint32_t Number, uint8_t Length);
void OLED_ShowBinNum(uint8_t Line, uint8_t Column, uint32_t Number, uint8_t Length);

#endif
```

## 6 易错点

- [ ] **买错版本**：买了 7 针 SPI 版，却照 I2C 的接法接线 → 屏幕毫无反应。
- [ ] `OLED.c` 只放进了文件夹，**没在 Keil 工程树里"Add Existing Files"** → 编译报 `undefined symbol OLED_Init`。
- [ ] 改了引脚宏但**没改 `OLED_I2C_Init()` 里的端口初始化**（或反过来）→ 屏幕不亮。
- [ ] 引脚配成**推挽输出**而不是**开漏输出** → I2C 时序不对，通信失败。（I2C 要求开漏 + 上拉）
- [ ] `VCC` 接 5 V 到**裸屏**（非模块）→ 可能烧屏。模块板上有降压，可接 5 V；裸屏只能接 3.3 V 级逻辑供电。
- [ ] 上电后**立刻**发初始化命令，没等 SSD1306 稳定 → 黑屏/花屏。
- [ ] 显示字符时用了**双引号**：`OLED_ShowChar(1, 1, "A")` → 类型不匹配，应为 `'A'`。
- [ ] 以为"接了 OLED 就一定能用"——**软件 I2C 与硬件 I2C 的引脚自由度不同**，改动前先确认用的是哪种。

## 7 自测

1. 相比串口调试和 Keil 调试模式，OLED 调试最本质的优势是什么？
2. 4 针 I2C 版和 7 针 SPI 版模块，课程用的是哪个？为什么它"可以接任意 GPIO"？
3. I2C 的起始条件和停止条件，分别是在 SCL 为什么电平时 SDA 发生什么跳变？
4. SSD1306 只有一根数据线，它是靠什么区分"这次发的是命令还是数据"？
5. 移植 OLED 驱动时，如果要换成 PA0（SCL）/ PA1（SDA），需要改哪几处？

> [!success]- 参考答案
> 1. OLED 由单片机**自己**驱动，**不需要电脑、不占串口，而且程序可以全速运行**——能一边跑一边实时看到变量的动态变化。Keil 调试模式必须停下程序、连着仿真器，串口调试要电脑配合。
> 2. 课程用的是 **4 针 I2C 版**。因为课程用**软件 I2C**（普通 GPIO 手动翻电平模拟时序），不占用芯片的硬件 I2C 外设，所以 SCL/SDA 可以落在任意 GPIO 上。若改用硬件 I2C 外设，就必须用指定的复用引脚。
> 3. **起始**：SCL 保持**高**电平时，SDA **由高变低**。**停止**：SCL 保持**高**电平时，SDA **由低变高**。
> 4. 靠地址字节后面的**第 2 个字节（控制字节）**：发 `0x00` 表示后面跟命令，发 `0x40` 表示后面跟数据。
> 5. 代码里**两处一起改**：① `#define OLED_W_SCL/SDA` 的端口与引脚号；② `OLED_I2C_Init()` 里的 `RCC_APB2PeriphClockCmd`（改为 `RCC_APB2Periph_GPIOA`）和两处 `GPIO_Init`。再确认 PA0/PA1 没有被工程里其他模块占用。

## 8 待核对

- [ ] **7 针模块那 7 个引脚的完整丝印名称与排列顺序**（本笔记只核实了"除 GND/VCC 外是 SPI 通信引脚"，未取到引脚顺序的可靠来源）。
- [ ] 本集是否讲解 Keil 调试模式的操作细节（另有公开笔记提到 `ODR` 寄存器观察、外设菜单栏等内容），需对照视频确认其归属与讲法。
- [ ] `OLED_Init()` 命令序列中每条 SSD1306 命令的讲解详略程度。
- [ ] 第 1 节表格中「**点灯调试法**」「**注释/对照调试法**」两行：官方课件《调试方式》页（Slide 37）只列了**串口调试、显示屏调试、Keil 调试模式**三种，这两项**未在官方课件/源码/接线图中核实**，需对照 4-1 视频确认是否为课上讲授内容。

## 核对记录（2026-09-26）

> [!success] 核对依据
> 实际用到的依据（均为本机官方资料）：
> - **官方源码**：`程序源码\STM32Project-有注释版\4-1 OLED显示屏\Hardware\OLED.c`、`OLED.h`、`OLED_Font.h`；`1-4 OLED驱动函数模块\4针脚I2C版本\OLED.c`、`OLED.h`；`1-4 OLED驱动函数模块\7针脚SPI版本\OLED.c`
> - **官方接线图**：`ground-truth\接线图\4-1 OLED显示屏.png`
> - **官方课件**：`ground-truth\课件文本.md`（Slide 37 调试方式、Slide 38 OLED 简介、Slide 40 驱动函数）
> - **引脚定义表**：`ground-truth\F103C8T6引脚定义_缩略.png`
> - **官方驱动页**：<https://jiangxiekeji.com/tutorial/oled.html>

| 核对项 | 笔记原值 | 官方依据 | 结论 |
| --- | --- | --- | --- |
| 驱动默认 SCL 引脚 | PB6 | `4-1 OLED显示屏\Hardware\OLED.c` L5：`GPIO_WriteBit(GPIOB, GPIO_Pin_8, ...)` | 已修正（原为 PB6） |
| 驱动默认 SDA 引脚 | PB7 | 同上 L6：`GPIOB, GPIO_Pin_9` | 已修正（原为 PB7） |
| `1-4` 与 `4-1` 两个工程引脚是否一致 | 未提 | 两处 `OLED.c` 引脚宏与 `OLED_I2C_Init()` 完全相同（PB8/PB9） | 已补记（原未说明） |
| 7 针 SPI 版驱动引脚 | 未提 | `7针脚SPI版本\OLED.c` L5~L9：D0=PB12、D1=PB13、RES=PB14、DC=PB15、CS=PA8，`GPIO_Mode_Out_PP` | 已补记（原未说明） |
| 4.2 引脚示例代码 | `GPIO_Pin_6` / `GPIO_Pin_7` | 官方 `OLED_I2C_Init()`：`GPIO_Pin_8` / `GPIO_Pin_9` | 已修正（原为 Pin_6/Pin_7） |
| `GPIO_Mode` | 开漏输出 `GPIO_Mode_Out_OD` | `OLED.c` L14：`GPIO_InitStructure.GPIO_Mode = GPIO_Mode_Out_OD;` | 一致 |
| `GPIO_Speed` | 50 MHz | `OLED.c` L15：`GPIO_Speed_50MHz` | 一致 |
| I2C 发送字节写法 | `OLED_W_SDA(Byte & (0x80 >> i));` | `OLED.c` L60：`OLED_W_SDA(!!(Byte & (0x80 >> i)));` | 已修正（原缺 `!!`） |
| 不读应答（只补一个时钟） | 只补一个时钟、不处理 ACK | `OLED.c` L64：`OLED_W_SCL(1);	//额外的一个时钟，不处理应答信号` | 一致 |
| 从机地址 `0x78` | `0x78` | `OLED.c`（`OLED_WriteCommand` / `OLED_WriteData`）：`OLED_I2C_SendByte(0x78);	//从机地址` | 一致 |
| 命令/数据第 2 字节 | 命令 `0x00`、数据 `0x40` | `OLED.c`：`//写命令` 处 `0x00`、`//写数据` 处 `0x40` | 一致 |
| 函数总数与清单 | 8 个（Init/Clear/ShowChar/ShowString/ShowNum/ShowSignedNum/ShowHexNum/ShowBinNum） | 官方 `OLED.h` 全部 13 行，恰好这 8 条声明 | 一致 |
| 是否存在 `OLED_ShowChinese` / `OLED_ShowImage` / `OLED_Printf` | 未提及（`OLED_Printf` 未在本页出现） | 两个版本的 `OLED.h`、`OLED.c` 中均**无**这三个函数 | 一致 |
| 是否存在显存缓冲区 / `OLED_Update` | 未提及 | `OLED.c` 中**无** `GRAM`、无 `OLED_Update`，`OLED_ShowChar` 直接 `OLED_WriteData` 写屏 | 一致 |
| 驱动文件数量与文件名 | `OLED.c` / `OLED.h` / `OLED_Font.h` 三个 | 官方工程 `Hardware\` 下即为这三个文件 | 一致 |
| 字模库名与几何 | `OLED_F8x16[][]`，8×16 | `OLED_Font.h` L4~L5：`/*OLED字模库，宽8像素，高16像素*/`、`const uint8_t OLED_F8x16[][16]=` | 一致 |
| 字模寻址从空格起算 | 未在本页展开 | `OLED.c` L140：`OLED_F8x16[Char - ' '][i]` | 一致 |
| `OLED_Init()` 上电延时 | 1000×1000 空循环 | `OLED.c` L275~L278：`for (i = 0; i < 1000; i++) //上电延时 { for (j = 0; j < 1000; j++); }` | 一致 |
| `OLED_Init()` 末尾清屏 | 内含清屏 | `OLED.c` L320：`OLED_Clear();	//OLED清屏` | 一致 |
| 模块供电范围 | 3 ~ 5.5 V | 课件 Slide 38：「供电：3~5.5V」 | 一致 |
| 通信协议与分辨率 | I2C/SPI、128×64 | 课件 Slide 38：「通信协议：I2C/SPI，分辨率：128*64」 | 一致 |
| 模块尺寸 | 0.96 寸 | 课件 Slide 38：「0.96 寸 OLED 模块」 | 一致 |
| 驱动芯片 SSD1306（兼容 SSD1315） | SSD1306（兼容 SSD1315） | 官方驱动页「适用器件」：SSD1306 / SSD1315 | 一致 |
| 1.3 寸版用 SH1106 | SH1106 | 官方驱动页「适用器件」：1.3寸（驱动芯片：SH1106） | 一致 |
| 4 针 I2C / 7 针 SPI | 4 针 / 7 针 | 官方驱动页「适用器件」：4针脚I2C接口、7针脚SPI接口 | 一致 |
| 上拉电阻「模块板载 4.7 kΩ」 | 本课程 OLED 模块板上自带 4.7 kΩ 上拉 | 课件《I2C 硬件电路》只说「各添加一个上拉电阻，阻值一般为 4.7KΩ 左右」，**未说模块板载阻值** | 已修正（原为「模块板上自带 4.7 kΩ 上拉」，改为引用课件原文并注明模块阻值未核实） |
| 调试手段种类 | 五种（串口/显示屏/Keil/点灯/注释对照） | 课件 Slide 37 只列三种：串口调试、显示屏调试、Keil 调试模式 | 保留原文但标为待核实（见下） |
| 行列约定（Line 1~4 / Column 1~16） | 1~4 行、1~16 列 | `OLED.c` 各显示函数注释：`Line 行位置，范围：1~4`、`Column 列位置，范围：1~16` | 一致 |
| 移植「两处一起改」 | 两处 | `OLED.c` 中引脚只由引脚宏与 `OLED_I2C_Init()` 决定，改引脚确需同时改这两处 | 一致 |
| 「课程中还有一种接法是把 `VCC`/`GND` 接到 GPIO 供电」 | 原写为"课程中有这种接法" | 官方接线图中 `VCC`/`GND` 接在面包板电源轨上，未见 GPIO 供电接法 | 已修正（原为肯定句，现标为"未在官方资料中核实"，仅作排错提示） |

> [!note] 出处说明
> 本页引脚配置（**SCL = PB8、SDA = PB9**，开漏输出 `GPIO_Mode_Out_OD`、`GPIO_Speed_50MHz`）、`OLED_I2C_Init()` 与 I2C 时序代码、`OLED.h` 全部 8 条声明、`OLED_Init()` 的 1000×1000 上电延时与命令序列，已逐条对照**官方配套源码** `程序源码\STM32Project-有注释版\4-1 OLED显示屏\Hardware\OLED.c`、`OLED.h`，以及 `1-4 OLED驱动函数模块\4针脚I2C版本\OLED.c`、`OLED.h`；引脚号同时对照**官方接线图** `接线图\4-1 OLED显示屏.png`；模块规格（0.96 寸、128×64、SSD1306/SSD1315、4 针 I2C / 7 针 SPI、供电 3~5.5 V、颜色不限）对照**官方课件** Slide 38 与江协科技官方驱动页 <https://jiangxiekeji.com/tutorial/oled.html>。
> **仍未核实**：7 针 SPI 模块的引脚丝印顺序；本集视频中关于 Keil 调试模式操作细节的原话；「点灯调试法」「注释/对照调试法」两种调试方式是否出自本集视频；OLED 模块板载上拉电阻的具体阻值（官方资料中未见）。**未逐帧核对视频画面**，若与视频有出入以视频为准。
