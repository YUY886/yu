---
course: 江协科技 STM32入门教程-2023版
chapter: 03-1 GPIO输出
type: 分集笔记
source: https://www.bilibili.com/video/BV1th411z7sn?p=5
tags:
  - STM32
  - 江协科技
  - GPIO
  - 推挽输出
  - 开漏输出
status: draft
verify: 官方源码+接线图+课件
updated: 2026-09-26
---

# 03-1 GPIO 输出

> [!abstract] 这一集只解决一个问题
> **怎么让一个引脚输出高电平或低电平？**
>
> 答：开时钟 → `GPIO_Init()` 把引脚配成**推挽输出** → 用 `GPIO_SetBits()` / `GPIO_ResetBits()` 写 0 或 1。
>
> 但这一集真正值钱的是**位结构框图**和**8 种工作模式表**——后面 3-2、3-3、3-4 以及整门课所有外设的引脚配置，都是在这张表里挑一行。
>
> 下一集 [[江协STM32 03-2 LED闪烁与流水灯与蜂鸣器]] ｜ 章索引 [[江协STM32 03 GPIO（章索引）]]

## 1 GPIO 是什么

**GPIO = General Purpose Input Output，通用输入输出口**。它就是一个可以由程序控制的引脚，能输出高低电平，也能读回外界的高低电平。

三条必须先记住的硬事实：

- **引脚电平范围 0 ~ 3.3V**。0V 是低电平（逻辑 0），3.3V 是高电平（逻辑 1）。
- **输出最高只能是 3.3V**，因为芯片供电就是 3.3V，输出不可能超过电源电压。
- **部分引脚可容忍 5V 输入**。数据手册引脚定义表里带 **FT（Five-volt Tolerant，5V 容忍）** 标记的引脚，输入 5V 也认定为高电平，且不会损坏；不带 FT 的引脚只能接 3.3V。注意这是**输入**能力，输出依然是 3.3V。

两种用途：

| 方向 | 干什么 | 典型场景 |
| --- | --- | --- |
| 输出 | 控制端口输出高低电平 | 驱动 LED、控制蜂鸣器、继电器；模拟通信协议（I2C / SPI / 单总线）的输出时序 |
| 输入 | 读取端口的高低电平或电压 | 读按键、读模块的数字输出（光敏/热敏模块）、模拟输入配合 ADC 采电压、模拟通信协议的接收 |

> [!tip] 为什么这门课先讲 GPIO
> 大部分外设（UART、SPI、I2C、PWM）本质上都是"按时序拉着某根引脚高高低低"。GPIO 讲清楚了，后面就只是"谁来拉"的问题。

## 2 GPIO 的基本结构

```text
        内核                       APB2 外设总线
         │                              │
         └──────────► [ GPIOA / GPIOB / ... 寄存器组 ] ──► 驱动器 ──► PA0 ~ PA15
```

- **所有 GPIO 都挂在 APB2 总线上**（不是 APB1），所以开时钟用 `RCC_APB2PeriphClockCmd()`。
- 每个 GPIO 外设叫 GPIOA / GPIOB / ……，各有 **16 个引脚**，编号 0 ~ 15，如 PA0 ~ PA15。
- 内部由**寄存器 + 驱动器**两部分组成。寄存器（一段特殊存储器）负责存数据，内核通过 APB2 总线读写它；**驱动器负责增大驱动能力**——寄存器只管存 0/1，真要去点灯得靠驱动器出电流。
- 寄存器**每一位对应一个引脚**：输出寄存器写 1 → 引脚输出高电平；输入寄存器读到 1 → 引脚当前是高电平。
- STM32 是 32 位机，寄存器也是 32 位，但端口只有 16 位，所以**只有低 16 位有效，高 16 位没用**。

## 3 GPIO 位结构框图（本集核心）

参考手册里那张"某一位"的电路图，右边是引脚，左边是寄存器，中间是驱动器。整体分成上（输入）下（输出）两部分。

```text
                     VDD / VDD_FT
                        │
                   ┌────┴────┐  保护二极管（上）
        ┌──────────┤▶│        │
        │          └─────────┘
        │   ┌─── 上拉电阻 ───┐        ┌── 输入数据寄存器
   IO ──┼───┤  (弱上拉)      ├──►[施密特触发器]──┴──► 复用功能输入
  引脚  │   └─── 下拉电阻 ───┘        └──────────────► ADC（模拟输入，取在触发器之前）
        │          ┌─────────┐
        │          │▶│        │  保护二极管（下）
        │          └────┬────┘
        │             VSS
        │
        │   ┌──────────────┐
        └───┤ 输出数据寄存器 ├──┐
            └──────────────┘  │  ┌─── 位设置/清除寄存器（单独改某一位）
                              ├──┴──►[ P-MOS ]──► VDD
                              │      [ N-MOS ]──► VSS
                              └──────────────────────► IO 引脚
```

### 3.1 两个保护二极管

引脚旁边上下各接一个二极管：上面接 VDD，下面接 VSS。作用是**对输入电压限幅**。

- 输入电压 **高于 3.3V**：上二极管导通，多余电流直接灌进 VDD，不进入内部电路。
- 输入电压 **低于 0V**：下二极管导通，电流从 VSS 流出，不从内部电路抽取。
- 输入在 **0 ~ 3.3V** 之间：两个二极管都不导通，对电路无影响。

> [!note] VDD 还是 VDD_FT
> 电路图上保护二极管的上端标的可能是 `VDD` 或 `VDD_FT`。**`VDD_FT` 就是"容忍 5V 端口"的那一路**——它的上保护二极管做了特殊处理，否则直接接 3.3V 的 VDD，外部一加 5V 上管就会开启并产生很大电流。这正是不带 FT 的引脚不能接 5V 的原因。

### 3.2 上拉 / 下拉电阻

两个开关控制的两个电阻，一端到 VDD（上拉）、一端到 VSS（下拉）。阻值都很大，是**弱上拉 / 弱下拉**，目的是尽量不影响正常的输入操作。

| 上拉开关 | 下拉开关 | 结果 |
| --- | --- | --- |
| 导通 | 断开 | **上拉输入**，悬空时默认高电平 |
| 断开 | 导通 | **下拉输入**，悬空时默认低电平 |
| 断开 | 断开 | **浮空输入**，悬空时电平不确定 |

存在的意义：数字端口不是 0 就是 1，但**引脚什么都不接时是浮空的**，电平极易受外界干扰而乱跳。加上拉或下拉就给了一个确定默认值。

### 3.3 施密特触发器（Schmitt Trigger）

作用：**对输入电压进行整形**。

逻辑是双阈值的：输入高于**上限阈值**瞬间输出高电平，低于**下限阈值**瞬间输出低电平；处在两个阈值之间时**输出保持不变**。中间留出的这段"回差"能有效滤掉信号毛刺，避免因为干扰导致误判。

> [!warning] "TTL 肖特基触发器"是翻译错误
> 很多中文资料（包括课程 PPT）把它标成"TTL 肖特基触发器"，实际应为**施密特触发器（Schmitt Trigger）**，跟肖特基二极管无关。按施密特触发器理解即可。

经过整形的信号写入**输入数据寄存器**，程序读它就知道引脚电平。

### 3.4 两路片上外设连线

- **模拟输入**：接在施密特触发器**之前**（因为 ADC 要的就是原始模拟量）。
- **复用功能输入**：接在触发器**之后**（因为它要的是数字量，如串口 RX）。

### 3.5 输出部分：三种改单个引脚的办法

输出数据寄存器同时控制 16 个端口，而它只能整体读写。想只改其中一位有以下三种办法：

| 办法 | 做法 | 评价 |
| --- | --- | --- |
| 读-改-写 | `GPIOA->ODR &= ~(1<<0);` 或 `\|= (1<<0);` | 麻烦、效率低，且读改写之间可能被中断打断 |
| **位设置/清除寄存器** | 往 BSRR / BRR 的对应位写 1 | **课程与库函数采用的方式**，一步到位、不影响其他位 |
| 位带（Bit-Band） | 读写别名区地址，等价于读写某一位 | 好用但需要理解地址映射，本课不展开 |

**位设置/清除寄存器（BSRR、BRR）细节**：

- `GPIOx->BSRR`：**低 16 位写 1 = 置位（输出高）**，**高 16 位写 1 = 复位（输出低）**，写 0 不影响。
- `GPIOx->BRR`（位清除寄存器）：低 16 位写 1 = 复位（输出低）。
- 只做置位用 BSRR、只做复位用 BRR 更省事；要**同时**置位和复位多位就用 BSRR 一次写完，能保证同步性（分两次写则无法保证同步）。

### 3.6 MOS 管与三种输出状态

输出控制后接两个 MOS 管（电子开关）：上面 **P-MOS 接 VDD**，下面 **N-MOS 接 VSS**。用信号控制通断，就有三种状态：

| 状态 | P-MOS | N-MOS | 行为 |
| --- | --- | --- | --- |
| **推挽输出** | 有效 | 有效 | 写 1 → 上管通、下管断 → 接 VDD，输出高；写 0 → 上管断、下管通 → 接 VSS，输出低 |
| **开漏输出** | **无效** | 有效 | 写 1 → 下管断 → 引脚相当于断开（**高阻态**）；写 0 → 下管通 → 接 VSS，输出低 |
| **关闭** | 无效 | 无效 | 配成输入模式时，端口电平完全由外部信号决定 |

关键结论：

- 推挽模式下**高低电平都有强驱动能力**，所以也叫"强推输出"。此时 **STM32 对引脚有绝对控制权**，电平由它说了算。
- 开漏模式下**只有低电平有驱动能力**，高电平没有驱动能力。

## 4 八种工作模式

配置端口配置寄存器（CRL / CRH，每个引脚占 4 位：2 位 MODE + 2 位 CNF），上图的电路就会跟着变（开关通断、MOS 是否有效、数据选择器选谁），得到 8 种模式：

| 模式名 | 枚举值 | 性质 | 特征 |
| --- | --- | --- | --- |
| 浮空输入 | `GPIO_Mode_IN_FLOATING` | 数字输入 | 可读引脚电平，**引脚悬空则电平不确定** |
| 上拉输入 | `GPIO_Mode_IPU` | 数字输入 | 可读引脚电平，内部接上拉电阻，**悬空默认高电平** |
| 下拉输入 | `GPIO_Mode_IPD` | 数字输入 | 可读引脚电平，内部接下拉电阻，**悬空默认低电平** |
| 模拟输入 | `GPIO_Mode_AIN` | 模拟输入 | GPIO 数字部分失效，引脚直接接入内部 ADC |
| 开漏输出 | `GPIO_Mode_Out_OD` | 数字输出 | 可输出电平，**高电平为高阻态，低电平接 VSS** |
| 推挽输出 | `GPIO_Mode_Out_PP` | 数字输出 | 可输出电平，**高电平接 VDD，低电平接 VSS** |
| 复用开漏输出 | `GPIO_Mode_AF_OD` | 数字输出 | **由片上外设控制**，高电平高阻态，低电平接 VSS |
| 复用推挽输出 | `GPIO_Mode_AF_PP` | 数字输出 | **由片上外设控制**，高电平接 VDD，低电平接 VSS |

### 4.1 三种数字输入

浮空 / 上拉 / 下拉三者电路结构基本一样，**只有上下拉开关不同**，都属于数字输入口，都能读端口高低电平。此时输出驱动器断开，端口只能输入。

> [!warning] 浮空输入不能悬空
> 用浮空输入时，端口**一定要接一个连续的驱动源**，不能出现悬空状态，否则读数随机跳变。

### 4.2 模拟输入

ADC 的专属配置。此时输出驱动器断开、施密特触发器也关闭，整个 GPIO 的数字部分基本失效，**只有"引脚 → 片上 ADC"这一根线有效**。用 ADC 时把引脚配成模拟输入即可，其他时候一般用不到。

### 4.3 开漏输出与推挽输出

两者电路基本一样，区别就一条：**开漏让 P-MOS 失效**，推挽两个都有效。

> [!tip] 输出模式下输入也是有效的
> 输出模式下输入通道并没有被关掉（一个端口只能有一个输出，但可以有多个输入）。所以**输出模式也可以顺便读回引脚的真实电平**——这正是后面做 I2C、单总线这类"半双工"协议的基础。

### 4.4 复用开漏 / 复用推挽输出

和普通开漏、推挽差不多，区别是**引脚控制权从输出数据寄存器转移到了片上外设**。此时通用输出断开，外设（TIM、USART、SPI、I2C）来拉这根线；输入部分依然有效，外设和普通输入都能读到引脚电平。

> [!tip] 8 种模式里只有 1 种关掉了输入
> 除了**模拟输入**会关闭数字输入功能，其余 **7 种模式的输入都是有效的**。

## 5 开漏 vs 推挽

这是本集最容易考、也最容易在工程里踩坑的一对概念。

| | 推挽输出（Push-Pull） | 开漏输出（Open-Drain） |
| --- | --- | --- |
| P-MOS | 有效 | 无效 |
| 输出高电平 | 主动接到 VDD，**有驱动能力** | **高阻态**，靠外部上拉电阻拉高 |
| 输出低电平 | 主动接到 VSS，有驱动能力 | 主动接到 VSS，有驱动能力 |
| 能否输出 5V | 不能（供电只有 3.3V） | **能**（外部上拉到 5V 即可） |
| 能否多个设备并在一根线上 | 不能，会"打架" | **能**，天然线与 |
| 典型用途 | 点灯、驱动蜂鸣器、PWM、串口 TX | I2C 的 SDA/SCL、电平转换、共享总线 |

### 5.1 为什么开漏能"线与"

一根线上挂多个开漏输出时：

- 任何**一个**设备输出低电平，就把线拉低 → 线上是低电平。
- **全部**设备都输出高电平（都是高阻）时，线才被上拉电阻拉高 → 线上是高电平。

也就是 **"有低则低，全高才高"**，相当于把所有输出做了**逻辑与**，这就是"线与（wired-AND）"。而推挽输出如果两个设备一个拉高一个拉低，就会形成 VDD 到 VSS 的直通大电流，把管子烧掉——所以**共享总线绝不能用推挽**。

### 5.2 为什么 I2C 必须用开漏

I2C 总线上有主设备也有从设备，谁都可以把 SCL/SDA 拉低：

- 需要**线与**来判仲裁、做时钟同步（Clock Stretching）——从设备拉低 SCL 就能拖住主机；
- 需要**多机共享**一根线而不打架；
- 总线上通常外接 4.7kΩ 一类上拉电阻，正是因为开漏高电平本身没有驱动能力。

所以 I2C 引脚必须是**开漏**。用硬件 I2C 外设时配 `GPIO_Mode_AF_OD`；用 GPIO 软件模拟 I2C 时配 `GPIO_Mode_Out_OD`。

### 5.3 开漏还能做电平转换

引脚外接一个上拉电阻到 **5V**：

- 输出低 → 内部 N-MOS 直接接 VSS，引脚为 0V；
- 输出高 → 下管断开，由外部上拉电阻把引脚拉到 **5V**。

这样就用一个 3.3V 的芯片输出了 **5V 电平**，用来兼容 5V 器件，这是开漏最实用的附加功能。

## 6 GPIO 相关寄存器速览

| 寄存器 | 名称 | 作用 |
| --- | --- | --- |
| `CRL` | 端口配置低寄存器 | 配置 Pin 0 ~ 7，每引脚 4 位（2 位 MODE + 2 位 CNF） |
| `CRH` | 端口配置高寄存器 | 配置 Pin 8 ~ 15，同样每引脚 4 位 |
| `IDR` | 端口输入数据寄存器 | 低 16 位对应 16 个引脚的电平 |
| `ODR` | 端口输出数据寄存器 | 低 16 位对应 16 个引脚的输出值 |
| `BSRR` | 端口位设置/清除寄存器 | 低 16 位置位，高 16 位复位，写 1 生效、写 0 无影响 |
| `BRR` | 端口位清除寄存器 | 低 16 位复位 |
| `LCKR` | 端口配置锁定寄存器 | 锁定引脚配置防止误改，本课基本不用 |

**输出速度（MODE 位）**：可限制引脚的最大翻转速度，目的是兼顾低功耗与稳定性。三档：`GPIO_Speed_10MHz` / `GPIO_Speed_2MHz` / `GPIO_Speed_50MHz`。**要求不高时配 50MHz 即可。**

## 7 常用库函数

### 7.1 RCC 时钟控制（`stm32f10x_rcc.h`）

```c
/* 三条总线各一个，按外设挂在哪条总线选哪个 */
void RCC_AHBPeriphClockCmd(uint32_t RCC_AHBPeriph,  FunctionalState NewState);
void RCC_APB2PeriphClockCmd(uint32_t RCC_APB2Periph, FunctionalState NewState);  // GPIO 用这个
void RCC_APB1PeriphClockCmd(uint32_t RCC_APB1Periph, FunctionalState NewState);
```

GPIO 在 APB2 上，所以开局必写：

```c
RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOA, ENABLE);   // 开 GPIOA 的时钟
```

> [!warning] 不开时钟 = 一切白干
> 使用任何外设（不只是 GPIO）前都必须开时钟，否则对外设的所有操作都无效，而且**不会报错、不会警告**，只是毫无现象。这是新手最常见的坑。

### 7.2 GPIO 库函数（`stm32f10x_gpio.h`）

```c
/* 初始化 */
void     GPIO_Init(GPIO_TypeDef* GPIOx, GPIO_InitTypeDef* GPIO_InitStruct);
void     GPIO_DeInit(GPIO_TypeDef* GPIOx);          // 复位到默认配置
void     GPIO_StructInit(GPIO_InitTypeDef* GPIO_InitStruct);  // 结构体填默认值

/* 读：输入用 ReadInput，输出用 ReadOutput */
uint8_t  GPIO_ReadInputDataBit(GPIO_TypeDef* GPIOx, uint16_t GPIO_Pin);
uint16_t GPIO_ReadInputData(GPIO_TypeDef* GPIOx);
uint8_t  GPIO_ReadOutputDataBit(GPIO_TypeDef* GPIOx, uint16_t GPIO_Pin);
uint16_t GPIO_ReadOutputData(GPIO_TypeDef* GPIOx);

/* 写 */
void     GPIO_SetBits(GPIO_TypeDef* GPIOx, uint16_t GPIO_Pin);       // 置高
void     GPIO_ResetBits(GPIO_TypeDef* GPIOx, uint16_t GPIO_Pin);     // 置低
void     GPIO_WriteBit(GPIO_TypeDef* GPIOx, uint16_t GPIO_Pin, BitAction BitVal);
void     GPIO_Write(GPIO_TypeDef* GPIOx, uint16_t PortVal);          // 整个端口一次写
```

`BitAction` 只有两个值：

```c
typedef enum
{
  Bit_RESET = 0,   // 低电平
  Bit_SET          // 高电平
} BitAction;
```

### 7.3 `GPIO_InitTypeDef` 三个成员

```c
typedef struct
{
  uint16_t        GPIO_Pin;    // 选哪些引脚，如 GPIO_Pin_0 或 GPIO_Pin_0 | GPIO_Pin_1 或 GPIO_Pin_All
  GPIOSpeed_TypeDef GPIO_Speed; // 输出速度：GPIO_Speed_10MHz / 2MHz / 50MHz
  GPIOMode_TypeDef  GPIO_Mode;  // 八种模式之一
} GPIO_InitTypeDef;
```

| 成员 | 取值来源 | 说明 |
| --- | --- | --- |
| `GPIO_Pin` | `GPIO_Pin_0` ~ `GPIO_Pin_15`、`GPIO_Pin_All` | 用 `\|` 可以一次选多个引脚 |
| `GPIO_Speed` | `GPIO_Speed_10MHz`、`GPIO_Speed_2MHz`、`GPIO_Speed_50MHz` | **仅对输出模式有意义**，输入模式随便写 |
| `GPIO_Mode` | 8 种模式枚举 | 见第 4 节表格 |

`GPIO_Pin_x` 的取值就是一个二进制位，**第 n 号引脚对应 `1 << n`**：

```text
GPIO_Pin_0  = 0x0001   →  0000 0000 0000 0001
GPIO_Pin_1  = 0x0002   →  0000 0000 0000 0010
GPIO_Pin_2  = 0x0004   →  0000 0000 0000 0100
GPIO_Pin_3  = 0x0008   →  0000 0000 0000 1000
...
GPIO_Pin_7  = 0x0080   →  0000 0000 1000 0000
GPIO_Pin_All= 0xFFFF   →  1111 1111 1111 1111
```

> [!tip] 为什么这套枚举值看起来"很怪"
> `GPIO_Mode_Out_PP = 0x10`、`GPIO_Mode_IPU = 0x48`，不是随便编的。低 4 位就是写进配置寄存器的 MODE + CNF 值，高 4 位是给 `GPIO_Init()` 内部判断用的标志。看个例子就明白了：
>
> ```text
> GPIO_Mode_Out_PP = 0x10  → MODE=11(输出50MHz) CNF=00(通用推挽)  = 0x_0  (低4位 0x0)
> GPIO_Mode_Out_OD = 0x14  → MODE=11            CNF=01(通用开漏)  = 0x_4
> GPIO_Mode_AF_PP  = 0x18  → MODE=11            CNF=10(复用推挽)  = 0x_8
> GPIO_Mode_AF_OD  = 0x1C  → MODE=11            CNF=11(复用开漏)  = 0x_C
> GPIO_Mode_AIN    = 0x00  → MODE=00(输入)      CNF=00(模拟)      = 0x_0
> GPIO_Mode_IPU    = 0x48  → MODE=10(输入+上下拉) CNF=10(上拉)     = 0x_8
> ```
>
> 用 `GPIO_Init()` 就行，不用背这些数。

> [!warning] 一个容易忽略的副作用
> 配置成**上拉/下拉输入**时，`GPIO_Init()` 内部还会去写 `BSRR` / `BRR` 把对应引脚的输出数据寄存器预置为 1 或 0。所以**用 `GPIO_ReadOutputDataBit()` 在上拉输入引脚上读到的可能是 1**——这是正常的，不代表引脚真有输出。

## 8 完整示例：点亮一个 LED

### 8.1 三步走

```text
① RCC 开启 GPIO 时钟          RCC_APB2PeriphClockCmd(...)
② GPIO_Init 配置引脚模式与速度  GPIO_InitTypeDef + GPIO_Init(...)
③ 用输出函数控制电平           GPIO_SetBits / GPIO_ResetBits / GPIO_WriteBit / GPIO_Write
```

### 8.2 只点亮一个 LED（PA0，低电平点亮）

```c
#include "stm32f10x.h"   // Device header

int main(void)
{
	/* ① 开启 GPIOA 的时钟：GPIO 挂在 APB2 上。
	      不开时钟，后面所有配置都不报错但完全无效 */
	RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOA, ENABLE);

	/* ② 配置 PA0 为推挽输出 */
	GPIO_InitTypeDef GPIO_InitStructure;                       // 定义结构体变量

	GPIO_InitStructure.GPIO_Mode  = GPIO_Mode_Out_PP;          // 推挽输出：高低电平都有驱动能力
	GPIO_InitStructure.GPIO_Pin   = GPIO_Pin_0;                // 第 0 号引脚，即 PA0
	GPIO_InitStructure.GPIO_Speed = GPIO_Speed_50MHz;          // 输出速度 50MHz（要求不高时够用）

	GPIO_Init(GPIOA, &GPIO_InitStructure);                     // 把结构体交给 GPIO_Init
	                                                           // 函数内部会自动去配置 CRL/CRH 等寄存器

	/* ③ 输出低电平点亮 LED（具体高低电平哪个点亮，取决于硬件接法） */
	GPIO_ResetBits(GPIOA, GPIO_Pin_0);                         // PA0 输出低电平

	while (1)
	{
		/* 主循环空转：LED 保持当前状态 */
	}
}
```

### 8.3 四种输出函数写法对照

同样一件事——把 PA0 拉低再拉高——有四种写法，效果完全一样：

```c
/* 写法 1：SetBits / ResetBits，最常用、可读性最好 */
GPIO_ResetBits(GPIOA, GPIO_Pin_0);   // PA0 = 0
GPIO_SetBits(GPIOA, GPIO_Pin_0);     // PA0 = 1

/* 写法 2：WriteBit 配 Bit_RESET / Bit_SET，适合位值由变量决定 */
GPIO_WriteBit(GPIOA, GPIO_Pin_0, Bit_RESET);   // PA0 = 0
GPIO_WriteBit(GPIOA, GPIO_Pin_0, Bit_SET);     // PA0 = 1

/* 写法 3：WriteBit 直接给 0/1，必须强制转换成 BitAction 枚举类型 */
GPIO_WriteBit(GPIOA, GPIO_Pin_0, (BitAction)0);   // PA0 = 0
GPIO_WriteBit(GPIOA, GPIO_Pin_0, (BitAction)1);   // PA0 = 1

/* 写法 4：Write 一次写整个 16 位端口，会覆盖其他所有引脚 */
GPIO_Write(GPIOA, 0xFFFE);   // 1111 1111 1111 1110，只有 PA0 为低，其余全高
```

> [!warning] `GPIO_Write()` 会连坐
> `GPIO_Write()` 是**整体写入 16 位**，你在参数里没考虑到的引脚也会被一起改掉。只想动一位就用前三种写法。

### 8.4 一个实测细节：开漏输出点不亮 LED

如果把模式从 `GPIO_Mode_Out_PP` 改成 `GPIO_Mode_Out_OD`：

- **LED 正极接引脚、负极接 GND**（高电平点亮）→ **点不亮**，因为开漏输出高电平是高阻态，没有驱动能力。
- **LED 负极接引脚、正极接 3.3V**（低电平点亮）→ **能亮**，因为开漏的低电平是有驱动能力的。

这个实验最直观地证明了第 5 节那条结论：**开漏只有低电平有驱动能力**。

## 9 易错点

- [ ] 忘记 `RCC_APB2PeriphClockCmd()` → 配置全对但引脚毫无反应，且不报错。
- [ ] 把 GPIO 当 APB1 外设去开 `RCC_APB1PeriphClockCmd()` → 时钟没开，同样无反应。
- [ ] 结构体三个成员没赋值就调用 `GPIO_Init()` → 成员是随机值，引脚模式乱掉。
- [ ] 把 GPIOB 的引脚配置传给 `GPIO_Init(GPIOA, ...)` → 配的是别的端口。
- [ ] `GPIO_Pin` 与引脚的位号搞混：`GPIO_Pin_3` 是 `0x0008` 不是 `0x0003`。
- [ ] 输入模式去调 `GPIO_Speed` 以为有影响 → 对输入无效。
- [ ] 用推挽输出做 I2C / 共享总线 → 多设备互相"打架"，可能烧管子。
- [ ] 用开漏输出却忘了外接上拉电阻 → 高电平永远是高阻态，读不到高电平。
- [ ] 用了 `GPIO_Write()` 却只想着改一个引脚 → 其他 15 个引脚被一起改掉。
- [ ] 以为带 FT 的引脚能**输出** 5V → FT 只表示能**容忍** 5V 输入，输出上限仍是 3.3V。
- [ ] 浮空输入接悬空引脚 → 读数随机跳变，还容易被静电打坏。

## 10 自测

1. GPIOA 挂在哪条总线上？开时钟应该用哪个函数？
2. 推挽输出和开漏输出最本质的区别是什么？为什么 I2C 必须用开漏？
3. `GPIO_Pin_3` 的值是多少？为什么不是 `0x0003`？
4. 只想把 PA5 拉高而不影响 PA0 ~ PA4、PA6 ~ PA15，应该用哪个函数？为什么不用 `GPIO_Write()`？
5. 带 FT 标记的引脚，"容忍 5V"具体指什么？它能不能输出 5V？

> [!success]- 参考答案
> 1. 挂在 **APB2** 总线上，用 `RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOA, ENABLE)`。
> 2. 推挽的 P-MOS 有效，高低电平都能主动接到 VDD/VSS，**都有驱动能力**；开漏的 P-MOS 无效，高电平是**高阻态没有驱动能力**，只有低电平有驱动能力。I2C 总线上有多个主从设备共享一根线，需要"线与"（有低则低、全高才高）来支持多机仲裁和时钟同步，同时避免推挽直通烧管，所以必须开漏。
> 3. `0x0008`。因为端口寄存器的**第 n 位对应第 n 号引脚**，所以 `GPIO_Pin_x` 取的是 `1 << x`，即 2 的幂。
> 4. 用 `GPIO_SetBits(GPIOA, GPIO_Pin_5)`。它内部走 BSRR 寄存器，只对指定位写 1，其他位写 0 不受影响；而 `GPIO_Write()` 是整体写 16 位 ODR，会把其他引脚一起改掉。
> 5. 指该引脚**作为输入**时可以承受 5V 电压并认定为高电平，而不会损坏芯片（其保护二极管接的是 `VDD_FT`）。**输出仍然是 3.3V**，因为芯片供电只有 3.3V；要输出 5V 得用开漏外接上拉到 5V。

## 11 待核对

- [ ] 位带（Bit-Band）别名区地址的具体数值是否在视频中给出（**官方源码、接线图、课件文本中均未见**）。
- [ ] 位结构框图中 `VDD_FT` 保护电路的具体讲法、以及「TTL 肖特基触发器」是否为课件/课堂沿用叫法。**课件文本 Slide 20「GPIO 位结构」只有一张图、可提取文本里没有这些标注**，无法从可获取文本中核实（见 3.1、3.3 两处 callout）。

## 核对记录（2026-09-26）

> [!success] 核对依据
> - 固件库头文件：`STM32F10x_StdPeriph_Lib_V3.5.0\STM32F10x_StdPeriph_Lib_V3.5.0\Libraries\STM32F10x_StdPeriph_Driver\inc\stm32f10x_gpio.h`
> - 课件文本：`课件文本.md`（Slide 18 GPIO 简介、Slide 19 GPIO 基本结构、Slide 21 GPIO 模式表）
> - 官方源码：`STM32Project-有注释版\3-1 LED闪烁\User\main.c`、`3-2 LED流水灯\User\main.c`、`3-3 蜂鸣器\User\main.c`、`3-4 按键控制LED\Hardware\LED.c`、`Key.c`、`3-5 光敏传感器控制蜂鸣器\Hardware\LightSensor.c`
> - 官方接线图：`接线图\3-1 LED闪烁.png`、`3-2 LED流水灯.png`、`3-3 蜂鸣器.png`、`3-4 按键控制LED.png`
> - 引脚定义表：`F103C8T6引脚定义_缩略.png`

| 核对项 | 笔记原值 | 官方依据 | 结论 |
| --- | --- | --- | --- |
| 8 种模式枚举值 | `GPIO_Mode_Out_PP = 0x10`、`GPIO_Mode_IPU = 0x48` 等 | `stm32f10x_gpio.h` 第 72~80 行逐条给出同样数值 | 一致 |
| 「低 4 位是 MODE+CNF，高 4 位是判断标志」 | 见 7.3 节 callout | 库头文件数值与 RM0008 的 CRL/CRH 编码吻合（`Out_PP=0x10`→MODE=11/CNF=00，`IPU=0x48`→MODE=10/CNF=10） | 一致 |
| 三个速度枚举 | `GPIO_Speed_10MHz`/`2MHz`/`50MHz` | 库头文件第 60~62 行完全一致 | 一致 |
| `GPIO_InitTypeDef` 三个成员 | `GPIO_Pin` / `GPIO_Speed` / `GPIO_Mode` | 库头文件第 91~100 行，成员顺序与类型一致 | 一致 |
| `BitAction` 枚举 | `Bit_RESET = 0`、`Bit_SET` | 库头文件第 109~110 行一致 | 一致 |
| `GPIO_Pin_x = 1 << x`，`GPIO_Pin_All = 0xFFFF` | 见 7.3 节二进制表 | 库头文件第 142~143 行：`GPIO_Pin_15 = 0x8000`、`GPIO_Pin_All = 0xFFFF` | 一致 |
| 库函数原型清单 | 第 7.2 节 12 个原型 | 库头文件第 276~290 行、353~361 行逐个比对，签名与返回类型均一致 | 一致 |
| GPIO 挂 APB2，开时钟用 `RCC_APB2PeriphClockCmd` | 见第 2、7.1 节 | 课件文本 Slide 19「APB2」；官方 `3-1\User\main.c` 第 7 行正是 `RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOA, ENABLE)` | 一致 |
| 输出模式仍可读回引脚真实电平 | 见 4.3 节 callout | 官方 `LED.c` 的 `LED1_Turn()` 用 `GPIO_ReadOutputDataBit` 读 ODR 取反（读的是 ODR 而非 IDR，本页表述正确） | 一致 |
| 引脚电平 0~3.3V、FT 引脚可容忍 5V 输入、输出仍为 3.3V | 见第 1 节 | 课件文本 Slide 18「引脚电平：0V~3.3V，部分引脚可容忍 5V」；引脚定义表设 `I/O电平(FT)` 一列，PB12/PB13 等标 FT | 一致 |
| 「点亮 LED」用的引脚（8.2 节示例） | 示例写「PA0，低电平点亮」，原在待核对中标注「课程实际用哪个引脚未确认」 | 官方 `3-1 LED闪烁\User\main.c`：`GPIO_InitStructure.GPIO_Pin = GPIO_Pin_0`（PA0），`GPIO_ResetBits` 为亮；接线图 3-1 中 LED 阳极经导线接正电源轨 | 一致（依官方源码与接线图确认，已移出待核对） |
| 流水灯实测细节（开漏点不亮 LED） | 见 8.4 节 | 官方源码与接线图均为推挽输出实验，**未见开漏输出的实测演示**；该结论属 STM32 开漏电气特性推论 | 一致（原理推论，依据中无对应实验） |

> [!note] 出处说明
> 本页位结构、8 种工作模式、寄存器行为与库函数原型已对照 **ST 标准外设库 V3.5.0 `stm32f10x_gpio.h`**、**官方配套源码（`STM32Project-有注释版\3-1`~`3-5`）**、**官方接线图（`接线图\3-1`~`3-5`）**、**课程课件 `课件文本.md`（Slide 18/19/21）** 与 **引脚定义表** 逐条核对；引脚电平、FT 标记、推挽/开漏特性为 STM32F103 既定事实。**仍未核实**：位带别名区地址的具体数值、课件 PPT「GPIO 位结构」图中「TTL 肖特基触发器」这一标注（该页只有图片，无法从文本提取）、以及视频画面中的讲解原话。
