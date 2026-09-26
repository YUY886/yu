---
course: 江协科技 STM32入门教程-2023版
chapter: 08-1 DMA直接存储器存取
type: 分集笔记
source: https://www.bilibili.com/video/BV1th411z7sn?p=23
tags:
  - STM32
  - 江协科技
  - DMA
  - AHB
  - 数据转运
status: draft
verify: 官方源码+接线图+课件
updated: 2026-09-26
---

# 08-1 DMA 直接存储器存取

> [!abstract] 这一集只解决一个问题
> **怎么让数据不经过 CPU，自己从一处搬到另一处？**
>
> 答：把「源地址、目标地址、搬几次」交给 **DMA**，它在 **AHB 总线**上自己完成「读 → 写 → 计数器减一」，搬完置一个标志（或产生中断）告诉 CPU 一声。
>
> 章索引 [[江协STM32 08 DMA（章索引）]] ｜ 下一集 [[江协STM32 08-2 DMA数据转运与DMA+AD多通道]]

## 1 为什么需要 DMA

CPU 搬数据是「读一次、写一次」，两步都要占 CPU 的指令周期，也要占总线。数据量大、或者数据**持续不断**地来（ADC 连续转换、串口连续收发），CPU 就会被搬运动作占满，没空干别的。

DMA 把这件事从 CPU 手里接管过来。课件 Slide 100 的原话：

> DMA（Direct Memory Access）直接存储器存取。DMA 可以提供外设和存储器或者存储器和存储器之间的高速数据传输，无须 CPU 干预，节省了 CPU 的资源。

三种可能的搬运：

| 搬运类型 | 例子 | 本课程哪里用 |
| --- | --- | --- |
| 外设 → 存储器 | ADC 数据寄存器 → SRAM 数组 | 8-2 的 DMA + AD 多通道 |
| 存储器 → 外设 | SRAM 数组 → 串口数据寄存器 | 第 9 章串口部分 |
| 存储器 → 存储器 | SRAM 数组 → SRAM 数组 | 8-2 的数据转运实验 |

> [!note] 先说清一个容易混的地方：工程编号 ≠ 视频集号
> 官方工程目录 `8-1 DMA数据转运\` 里的"8-1"是**工程编号**，这个"SRAM 数组互搬"的数据转运实验是在 **8-2 视频（P24）** 里讲、并配 `MyDMA.c` / `main.c` 整套代码的；**8-1 视频（P23）是理论集**，讲的是本页这些原理。
> 本页凡出现 `System\MyDMA.c`、`User\main.c` 的地方，指的都是 `8-1 DMA数据转运\` 这个官方工程目录里的文件。

关键区别：CPU 搬运要**逐字节执行指令**；DMA 搬运是硬件电路在总线上直接读写，CPU 只在开头配置、结尾收尾。

## 2 DMA 挂在 AHB 总线上

DMA 不是"挂在某个外设后面"的东西，它自己是**总线矩阵上的主设备**，和 Cortex-M3 内核平级。

参考手册 §3.1 系统架构（英文版 RM0008 第 46 页）列出：

| 类别 | 成员 |
| --- | --- |
| 4 个主设备 | Cortex-M3 的 DCode 总线、System 总线；**GP-DMA1 & 2（通用 DMA）** |
| 4 个从设备 | 内部 SRAM、内部 Flash、FSMC、AHB→APBx（APB1 / APB2 桥） |

也就是说 CPU 和 DMA 都能主动发起总线访问，谁在读写由总线矩阵裁决——这就是"抢总线"的由来。

课件 Slide 102 的 DMA 框图（`ppt\media\image94.png`，原图标注「图21 DMA框图」）画得更直观：

```text
Cortex-M3 ─┬─ ICode ──►[ Flash接口控制器 ]──► Flash
           ├─ DCode ──┐
           └─ 系统 ───┴─►[ 总线矩阵 ]──► SRAM
                          ▲  ▲
        ┌─ DMA1（通道1~7）┘  └─ DMA2（通道1~5）
        │    └─[ 仲裁器 ]──[ AHB 从设备 ]
        └─► DMA 请求 ◄── APB1 外设 / APB2 外设
```

## 3 STM32F103C8T6 的 DMA 资源

课件 Slide 100 原文：**12 个独立可配置的通道：DMA1（7 个通道）、DMA2（5 个通道）；每个通道都支持软件触发和特定的硬件触发。** 并明确标注：

> **STM32F103C8T6 DMA 资源：DMA1（7 个通道）**

| 控制器 | 通道数 | F103C8T6 上有吗 |
| --- | --- | --- |
| DMA1 | 7（通道 1~7） | **有** |
| DMA2 | 5（通道 1~5） | **没有** |

参考手册在 DMA2 一节直接写了限制：DMA2 及其相关请求只在**高密度、XL 密度和互联型**器件上可用（RM0008 第 281 页 "The DMA2 controller and its relative requests are available only in high-density, XL-density and connectivity line devices."）。

> [!warning] 用不存在的 DMA2 不会报错
> 代码里写 `DMA2_Channel1` 编译得过、下载也不报错，只是**一点反应都没有**。标准库的 `IS_DMA_ALL_PERIPH` 里虽然列了 DMA2（`stm32f10x_dma.h` 第 102~106 行），那是给大容量芯片准备的。F103C8T6 上只用 DMA1。

## 4 DMA1 请求映射表

每个通道能响应哪些外设请求是**固定**的，不能随便改。多个请求进入同一个通道前是**逻辑或**关系：

> 参考手册 §13.3.7（RM0008 第 280 页）：The 7 requests from the peripherals (TIMx[1,2,3,4], ADC1, SPI1, SPI/I2S2, I2Cx[1,2] and USARTx[1,2,3]) are simply logically ORed before entering the DMA1, **this means that only one request must be enabled at a time.**

| 通道 | 可选的硬件请求（同一时刻只能使能一个） |
| --- | --- |
| 通道 1 | ADC1、TIM2_CH3、TIM4_CH1 |
| 通道 2 | SPI1_RX、USART3_TX、TIM1_CH1、TIM2_UP、TIM3_CH3 |
| 通道 3 | SPI1_TX、USART3_RX、TIM1_CH2、TIM3_CH4、TIM3_UP |
| 通道 4 | SPI2/I2S2_RX、USART1_TX、I2C2_TX、TIM1_CH4、TIM1_TRIG、TIM1_COM、TIM4_CH2 |
| 通道 5 | SPI2/I2S2_TX、USART1_RX、I2C2_RX、TIM1_UP、TIM2_CH1、TIM4_CH3 |
| 通道 6 | USART2_RX、I2C1_TX、TIM1_CH3、TIM3_CH1、TIM3_TRIG |
| 通道 7 | USART2_TX、I2C1_RX、TIM2_CH2、TIM2_CH4、TIM4_UP |

常查的几个：

| 外设请求 | 走哪个通道 | 本课程用在哪 |
| --- | --- | --- |
| ADC1 | DMA1_Channel1 | 8-2 的 DMA + AD 多通道 |
| SPI1_TX | DMA1_Channel3 | — |
| SPI1_RX | DMA1_Channel2 | — |
| USART1_TX | DMA1_Channel4 | 第 9 章串口发送 |
| USART1_RX | DMA1_Channel5 | 第 9 章串口接收 |

> [!note] 外设自己还有一个"总开关"
> 光把 DMA 通道配好还不够——**外设自己的 DMA 请求位**也要打开。参考手册 §13.3.7 同一段写着：The peripheral DMA requests can be independently activated/de-activated by programming the DMA control bit in the registers of the corresponding peripheral.
> 库函数里就是 `ADC_DMACmd(ADC1, ENABLE)`、`USART_DMACmd()` 这一类。忘了它，DMA 配得再对也等不到请求。

## 5 三大要素：外设地址、存储器地址、传输计数器

DMA 通道要干的活，说白了就是回答三个问题：

| 要素 | 寄存器 | 库函数成员 | 含义 |
| --- | --- | --- | --- |
| 从哪搬 / 搬到哪（外设侧） | `DMA_CPARx` | `DMA_PeripheralBaseAddr` | 外设数据寄存器的地址 |
| 搬到哪 / 从哪搬（存储器侧） | `DMA_CMARx` | `DMA_MemoryBaseAddr` | 存储器（SRAM / Flash）数组的地址 |
| 搬几次 | `DMA_CNDTRx` | `DMA_BufferSize` | 传输计数器，最多 65535 |

参考手册 §13.3.3（RM0008 第 276 页）把一次 DMA 传输拆成三个动作：

1. 从**外设数据寄存器**或**存储器**里把数据读出来（地址来自内部当前地址寄存器，首次传输用 `DMA_CPARx` / `DMA_CMARx` 里编程的基地址）；
2. 把读到的数据写到外设数据寄存器或存储器（同样走内部当前地址寄存器）；
3. `DMA_CNDTRx` **后递减**（post-decrementing），它装的是"还剩多少次没搬"。

> [!tip] 计数器是"还剩多少"，不是"搬了多少"
> `DMA_CNDTRx` 从设定值往下减到 0，搬运就结束。所以"搬 4 次"就写 4。标准库的 `DMA_GetCurrDataCounter()` 读回来的也是**剩余次数**。

## 6 DMA 基本结构（对照课件框图）

课件 Slide 103「DMA 基本结构」给出的框图元素（原页是矢量图，文字标签为）：

```text
外设寄存器 / SRAM / Flash
   ├─ 外设：  起始地址、数据宽度、地址是否自增
   ├─ 存储器：起始地址、数据宽度、地址是否自增
   ├─ 方向
   ├─ 硬件触发 ─┐
   ├─ 软件触发 ─┴─► M2M ─►[ 传输计数器 ]─►[ 自动重装器 ]
   └─ 开关控制（0 / 1）
```

对应到寄存器就是四组：

| 框图块 | 寄存器位 | 库函数 |
| --- | --- | --- |
| 外设起始地址 | `DMA_CPARx` | `DMA_PeripheralBaseAddr` |
| 外设数据宽度 | `DMA_CCRx` 的 PSIZE | `DMA_PeripheralDataSize` |
| 外设地址自增 | `DMA_CCRx` 的 PINC | `DMA_PeripheralInc` |
| 存储器起始地址 | `DMA_CMARx` | `DMA_MemoryBaseAddr` |
| 存储器数据宽度 | `DMA_CCRx` 的 MSIZE | `DMA_MemoryDataSize` |
| 存储器地址自增 | `DMA_CCRx` 的 MINC | `DMA_MemoryInc` |
| 方向 | `DMA_CCRx` 的 DIR | `DMA_DIR` |
| 软件触发 | `DMA_CCRx` 的 MEM2MEM | `DMA_M2M` |
| 传输计数器 | `DMA_CNDTRx` | `DMA_BufferSize` |
| 自动重装器 | `DMA_CCRx` 的 CIRC | `DMA_Mode` |
| 开关控制 0/1 | `DMA_CCRx` 的 EN | `DMA_Cmd()` |

注意框图里 **"传输计数器"和"自动重装器"是分开画的两块**：计数器是 `DMA_CNDTRx`，自动重装器由 CIRC 位控制。正常模式下计数到 0 就停；循环模式下自动把初值装回计数器。

## 7 关键配置项逐个说

### 7.1 传输方向

| 取值 | 含义 |
| --- | --- |
| `DMA_DIR_PeripheralSRC` | 外设是**源**：外设 → 存储器（ADC 采集、串口接收） |
| `DMA_DIR_PeripheralDST` | 外设是**目标**：存储器 → 外设（串口发送） |

存储器到存储器（M2M）时，**方向位照旧写 `DMA_DIR_PeripheralSRC`**，由 `DMA_M2M` 位决定谁自增——官方 `MyDMA.c` 就是这么写的（`System\MyDMA.c` 第 27 行）。

### 7.2 地址是否自增

`PINC` / `MINC` 控制"每搬一次，地址要不要往后走"。参考手册 §13.3.3（第 276 页）写明：

> If incremented mode is enabled, the address of the next transfer will be the address of the previous one incremented by **1, 2 or 4** depending on the chosen data size.

也就是自增步长 = 数据宽度（字节 / 半字 / 字），不是恒为 1。

| 场景 | 外设自增 | 存储器自增 |
| --- | --- | --- |
| 存储器 ↔ 存储器 | 使能 | 使能 |
| 外设固定地址（如 `ADC1->DR`）→ 数组 | **失能** | 使能 |
| 数组 → 外设固定地址（如串口 `DR`） | **失能** | 使能 |

### 7.3 数据宽度

| 取值 | 位宽 |
| --- | --- |
| `DMA_PeripheralDataSize_Byte` / `DMA_MemoryDataSize_Byte` | 8 位 |
| `..._HalfWord` | 16 位 |
| `..._Word` | 32 位 |

`DMA_BufferSize` 的单位是"数据个数"，具体一个数是多少位，取决于传输方向那一侧的宽度设置（`stm32f10x_dma.h` 第 59~61 行注明：The data unit is equal to the configuration set in DMA_PeripheralDataSize or DMA_MemoryDataSize members **depending in the transfer direction**）。两侧宽度不一致时，DMA 会做对齐处理，见第 10 节。

### 7.4 传输模式：正常 vs 循环

| 模式 | 取值 | 行为 |
| --- | --- | --- |
| 正常模式 | `DMA_Mode_Normal` | 计数到 0 后**不再响应任何请求**，通道实际停止 |
| 循环模式 | `DMA_Mode_Circular` | 计数到 0 后**自动把初值装回计数器**，同时把当前地址寄存器重置回 `DMA_CPARx` / `DMA_CMARx` 的基地址，继续搬 |

参考手册 §13.3.3（第 277 页）原文：

> If the channel is configured in noncircular mode, no DMA request is served after the last transfer (that is once the number of data items to be transferred has reached zero). In order to reload a new number of data items to be transferred into the DMA_CNDTRx register, the DMA channel must be disabled.
>
> In circular mode, after the last transfer, the DMA_CNDTRx register is automatically reloaded with the initially programmed value. The current internal address registers are reloaded with the base address values from the DMA_CPARx/DMA_CMARx registers.

同一页还有一条容易忽略的注解：

> If a DMA channel is disabled, the DMA registers are not reset. The DMA channel registers (DMA_CCRx, DMA_CPARx and DMA_CMARx) retain the initial values programmed during the channel configuration phase.

循环模式专门用来处理循环缓冲和连续数据流，参考手册举的例子就是 **ADC 扫描模式**——8-2 正是这个用法。

### 7.5 优先级

见第 9 节（仲裁器）。库函数四个取值：`DMA_Priority_VeryHigh` / `_High` / `_Medium` / `_Low`。

### 7.6 中断与事件

每个通道有三类事件，各自有独立的使能位（RM0008 第 279 页 Table 77）：

| 事件 | 标志位 | 使能位 | 库函数 |
| --- | --- | --- | --- |
| 半传输完成 | HTIF | HTIE | `DMA_IT_HT` |
| 传输完成 | TCIF | TCIE | `DMA_IT_TC` |
| 传输错误 | TEIF | TEIE | `DMA_IT_TE` |

另外还有一个**总标志** `GL`（global），是前三者的"或"（`DMA1_FLAG_GL1` 等）。

传输错误怎么来的？参考手册 §13.3.5（第 279 页）：读写**保留地址空间**会产生传输错误；出错时硬件会自动把该通道的 EN 位清 0（通道被强制关闭），并置 TEIF。

## 8 软件触发 vs 硬件触发

| 项目 | 硬件触发 | 软件触发 |
| --- | --- | --- |
| 触发源 | 外设的 DMA 请求（ADC、USART、SPI、定时器…） | 无，通道一使能就开搬 |
| 寄存器位 | — | `DMA_CCRx` 的 **MEM2MEM** |
| 库函数 | `DMA_M2M_Disable` | `DMA_M2M_Enable` |
| 典型场景 | 外设 ↔ 存储器 | 存储器 ↔ 存储器 |

参考手册 §13.3.3（第 277 页）：

> The DMA channels can also work without being triggered by a request from a peripheral. This mode is called Memory to Memory mode. If the MEM2MEM bit in the DMA_CCRx register is set, then the channel initiates transfers as soon as it is enabled by software by setting the Enable bit (EN) in the DMA_CCRx register. The transfer stops once the DMA_CNDTRx register reaches zero.

"**as soon as it is enabled**"这句话很关键：M2M 模式下只要 EN 置 1 就立刻开搬，所以初始化时必须**先别使能**，等要搬的时候再使能（官方 `MyDMA_Init` 末尾就是 `DMA_Cmd(DMA1_Channel1, DISABLE)`）。

> [!warning] MEM2MEM 和循环模式不能同时用
> 参考手册 §13.3.3（第 278 页）明确：**Memory to Memory mode may not be used at the same time as Circular mode.**
> 标准库头文件里也原样抄了这句提示（`stm32f10x_dma.h` 第 77~78 行：The circular buffer mode cannot be used if the memory-to-memory data transfer is configured on the selected Channel）。
> 所以「存储器到存储器 + 自动重装」这种组合是不存在的。

## 9 仲裁器：谁先用总线

多个通道同时要搬数据时，谁先上总线由**仲裁器**决定。参考手册 §13.3.2（第 276 页）：

> The arbiter manages the channel requests based on their priority and launches the peripheral/memory access sequences. The priorities are managed in two stages:
> - **Software**: each channel priority can be configured in the DMA_CCRx register. There are four levels: Very high priority / High priority / Medium priority / Low priority
> - **Hardware**: if 2 requests have the same software priority level, the channel with the **lowest number** will get priority versus the channel with the highest number. For example, channel 2 gets priority over channel 4.
>
> Note: In high-density, XL-density and connectivity line devices, the DMA1 controller has priority over the DMA2 controller.

课件 Slide 104 的映射图（`ppt\media\image95.png`，原图标注「图22 DMA1请求映像」）右侧就画了这条"**固定的硬件优先级**"，箭头从通道 1 往下走到通道 7，标注"高优先级 → 低优先级"。

```text
软件优先级（PL，4 级） ────┐
                          ├──► 仲裁 ──► 谁先占用总线
硬件优先级（通道号小者优先）┘
```

> [!note] 关于"DMA 和 CPU 抢总线"
> 参考手册 §3.1 说明 DMA 是总线矩阵的主设备、与 Cortex-M3 共享总线；§13.3.2 则只写了 **DMA 通道之间**的仲裁规则。
> **DMA 与 CPU 之间争用总线时怎么分配带宽**，在已核对的官方材料（课件、参考手册、标准库）里未见明确表述，本页不作肯定结论——已列入「待核对」。

## 10 数据宽度与对齐

两侧宽度不一致时 DMA 会做对齐。课件 Slide 105「数据宽度与对齐」用的就是参考手册的 Table 76（`表57 可编程的数据传输宽度和大小端操作(当PINC = MINC = 1)`，RM0008 第 278 页）。

摘几条最容易踩的（源 8 位 / 目标 16 位，转运 4 次）：

| 步骤 | 操作 |
| --- | --- |
| 1 | 读 `B0[7:0] @0x0`，写 `00B0[15:0] @0x0` |
| 2 | 读 `B1[7:0] @0x1`，写 `00B1[15:0] @0x2` |
| 3 | 读 `B2[7:0] @0x2`，写 `00B2[15:0] @0x4` |
| 4 | 读 `B3[7:0] @0x3`，写 `00B3[15:0] @0x6` |

规律：**源侧按源宽度走，目标侧按目标宽度走，各自自增各自的步长**，读到的数据在高位补 0 后写入。源宽 > 目标宽时则相反，只取低位（例如源 16 位、目标 8 位：读 `B1B0[15:0] @0x0`，写 `B0[7:0] @0x0`）。

参考手册另外提示了 AHB 外设不支持字节/半字写时的情况：DMA 会把数据在 HWDATA 总线的空闲通道上复制，例如写半字 `0xABCD` 时把总线置成 `0xABCDABCD`，写字节 `0xAB` 时置成 `0xABABABAB`（RM0008 第 278~279 页）。所以像 APB 备份寄存器这种 16 位寄存器，要用"存储器源 16 位、外设目标 32 位"的配法。

## 11 一个 DMA 通道的配置流程

参考手册 §13.3.3（第 277 页）给的六步：

```text
① 把外设寄存器地址写进 DMA_CPARx
② 把存储器地址写进 DMA_CMARx
③ 把要搬的数据个数写进 DMA_CNDTRx（每来一次请求减 1）
④ 用 DMA_CCRx 的 PL[1:0] 配优先级
⑤ 在 DMA_CCRx 里配方向、循环模式、外设/存储器自增、外设/存储器数据宽度、半传输/全传输中断
⑥ 把 DMA_CCRx 的 ENABLE 位置 1，通道开始工作
```

标准库把它包成了 `DMA_Init()` + `DMA_Cmd()`：

| 参考手册步骤 | 库函数写法 |
| --- | --- |
| ① | `DMA_InitStructure.DMA_PeripheralBaseAddr` |
| ② | `DMA_InitStructure.DMA_MemoryBaseAddr` |
| ③ | `DMA_InitStructure.DMA_BufferSize` |
| ④ | `DMA_InitStructure.DMA_Priority` |
| ⑤ | `DMA_DIR` / `DMA_Mode` / `DMA_PeripheralInc` / `DMA_MemoryInc` / `DMA_PeripheralDataSize` / `DMA_MemoryDataSize`，中断另用 `DMA_ITConfig()` |
| ⑥ | `DMA_Cmd(DMA1_ChannelX, ENABLE)` |

`DMA_Init()` 内部只改 `CCR` 的 MEM2MEM / PL / MSIZE / PSIZE / MINC / PINC / CIRC / DIR 这些位，然后依次写 `CNDTR`、`CPAR`、`CMAR`（`stm32f10x_dma.c` 第 218~250 行）。它**不动 EN 位**，所以"要不要现在就开搬"由你在 `DMA_Cmd()` 里决定。

## 12 易错点

- [ ] 想用 DMA2 → F103C8T6 上没有 DMA2，只有 DMA1 的 7 个通道。
- [ ] 只配 DMA，忘了开外设自己的 DMA 请求位（`ADC_DMACmd()`、`USART_DMACmd()`）→ DMA 干等，一次都不搬。
- [ ] 把 `DMA_BufferSize` 当成"字节数" → 它其实是**数据个数**，一个数多少位看数据宽度设置。
- [ ] 外设地址自增没关（`ADC1->DR` 这类固定地址）→ 第二次就开始读不存在的寄存器地址，直接传输错误。
- [ ] M2M 模式下初始化完就直接使能 → 数据当场被搬完，等到你想看现象时早就搬完了。官方写法是初始化时先 `DISABLE`。
- [ ] 通道还开着就调 `DMA_SetCurrDataCounter()` → 库函数注释明确要求必须先关闭（见下方核对记录）。
- [ ] 想「存储器到存储器 + 循环模式」→ 参考手册明确不允许两者同时使用。
- [ ] 正常模式下搬完了想再搬一次，却忘了重装计数器 → 计数器已经是 0，通道不再响应请求。
- [ ] 轮询完 `DMA_FLAG_TCx` 忘了清标志 → 下次等待会立刻"通过"，看起来像没搬就返回了。
- [ ] 认为「同一个通道上挂的多个请求可以同时用」→ 它们进通道前是逻辑或，同一时刻只能使能一个。

## 13 自测

1. STM32F103C8T6 有几个 DMA 控制器、几个通道？如果代码里写了 `DMA2_Channel1` 会怎样？
2. ADC1 的 DMA 请求固定走哪个通道？`USART1_TX` 呢？
3. `DMA_BufferSize = 4`，外设和存储器数据宽度都选 Byte，两侧自增都使能，一共搬几个字节？如果把**存储器自增关掉**会怎样？
4. 为什么 `MyDMA_Transfer()` 里要先 `DMA_Cmd(DISABLE)` 才能写传输计数器？
5. 想把"存储器 → 存储器"的搬运做成循环模式（自动重装、搬完从头再来），能实现吗？为什么？

> [!success]- 参考答案
> 1. 只有 DMA1，共 7 个通道（课件 Slide 100）。写 `DMA2_Channel1` 编译下载都不报错，但 F103C8T6 上没有这个控制器，**毫无反应**（参考手册注明 DMA2 只在高密度 / XL 密度 / 互联型器件上可用）。
> 2. ADC1 → `DMA1_Channel1`；`USART1_TX` → `DMA1_Channel4`（课件 Slide 104 图22；参考手册 Table 78）。
> 3. 4 次 × 1 字节 = **4 字节**。若关掉存储器自增，则 4 次都写同一个存储器地址（`DataB[0]`），最终只有第一个元素被反复覆盖，后 3 个元素保持原值不变（外设侧自增，所以源仍然依次读 `DataA[0]`~`DataA[3]`，`DataB[0]` 最后等于 `DataA[3]`）。
> 4. 标准库对 `DMA_SetCurrDataCounter()` 的注释写明它**只能在通道关闭时使用**（`stm32f10x_dma.c` 第 350 行）；参考手册也说明非循环模式下要重装计数器必须先关闭通道（RM0008 第 277 页）。
> 5. 不能。参考手册 §13.3.3 明确 "Memory to Memory mode may not be used at the same time as Circular mode."（RM0008 第 278 页），标准库头文件 `stm32f10x_dma.h` 第 77~78 行也有同样提示。要反复搬，只能在每次搬之前重新调用一次 `MyDMA_Transfer()` 重装计数器。

## 14 待核对

- [ ] 「DMA 不能把 Flash 当作搬运对象 / 不能做某种 Flash 相关搬运」这类限制说法：**官方课件（Slide 100~107）、参考手册 DMA 章节（第 13 章）、8-1 / 8-2 官方源码中均未见对应表述**。相反，参考手册 §3.1（第 46 页）把 GP-DMA1 & 2 与「内部 Flash」「内部 SRAM」并列画在同一个总线矩阵上，课件 Slide 103 也把 `Flash` 与 `SRAM` 并排列在"存储器"一侧。本条**不作为肯定结论**，留待视频原话确认。
- [ ] DMA 与 **CPU** 之间的总线争用与带宽分配细节：参考手册 §13.3.2 只写了 DMA 通道之间的仲裁器规则，未见 DMA 与 CPU 之间的仲裁表述。
- [ ] 视频里老师对「CPU 搬运的代价」的口头表述与现场实测现象（这部分不在源码、接线图与课件文本里）。
- [ ] 课件 Slide 106 的数据转运示意图画的是 `uint8_t DataA[7]; uint8_t DataB[7];`（7 个元素），而官方工程 `8-1 DMA数据转运\User\main.c` 里是 4 个元素——两者不一致的原因（大概率只是示意图随手画的），以源码为准。

## 核对记录（2026-09-26）

> [!success] 核对依据
> - 官方配套源码：`STM32Project-有注释版\8-1 DMA数据转运\System\MyDMA.c`、`System\MyDMA.h`、`User\main.c`、`Hardware\OLED.c`
> - 官方接线图：`ground-truth\接线图\8-1 DMA数据转运.png`
> - 课程课件文本：`ground-truth\课件文本.md`（Slide 100 DMA 简介、101 存储器映像、102 DMA 框图、103 DMA 基本结构、104 DMA 请求、105 数据宽度与对齐、106 数据转运+DMA）
> - 课件原始图形：`ground-truth\ppt_x\ppt\media\image94.png`（图21 DMA框图）、`image95.png`（图22 DMA1请求映像）、`image96.png`（表57 数据宽度与对齐）
> - ST 官方参考手册：`STM32F10xxx参考手册（英文）.pdf` §3.1（第 46 页 系统架构）、§13.3.1~§13.3.7（第 276~281 页）
> - ST 标准外设库 V3.5.0：`...\inc\stm32f10x_dma.h`、`...\src\stm32f10x_dma.c`
> - 引脚定义表：`ground-truth\引脚定义_xlsx原文.txt`

| 核对项 | 笔记值 | 官方依据（文件名+行号） | 结论 |
| --- | --- | --- | --- |
| DMA 定义 | 外设↔存储器 / 存储器↔存储器的高速传输，无须 CPU 干预 | 课件 Slide 100 逐字一致 | 一致 |
| 通道总数 | 12 个：DMA1（7）+ DMA2（5） | 课件 Slide 100 | 一致 |
| F103C8T6 的 DMA 资源 | 只有 DMA1（7 个通道） | 课件 Slide 100「STM32F103C8T6 DMA 资源：DMA1（7 个通道）」；参考手册第 281 页「DMA2 仅高密度/XL 密度/互联型器件可用」 | 一致 |
| DMA 的总线位置 | 总线矩阵主设备之一（GP-DMA1 & 2） | 参考手册 §3.1（第 46 页）Four masters: Cortex-M3 DCode/System bus、GP-DMA1 & 2 | 一致 |
| 存储器映像 | Flash `0x0800 0000`、SRAM `0x2000 0000`、外设寄存器 `0x4000 0000` | 课件 Slide 101 逐项一致 | 一致 |
| 请求映射表（7 个通道） | 见正文第 4 节 | 课件 Slide 104 图（`image95.png` 图22）与参考手册 Table 78（第 281 页）两张表逐格一致 | 一致 |
| ADC1 的通道 | `DMA1_Channel1` | 同上；8-2 官方源码 `Hardware\AD.c` 第 56 行也用 `DMA1_Channel1` | 一致 |
| USART1_TX 的通道 | `DMA1_Channel4` | 同上 | 一致 |
| SPI1_TX 的通道 | `DMA1_Channel3` | 同上 | 一致 |
| 同通道多请求的关系 | 进通道前逻辑或，同一时刻只能使能一个 | 参考手册 §13.3.7（第 280 页） | 一致 |
| 外设侧还要单独开 DMA 请求位 | 是（如 `ADC_DMACmd`） | 参考手册 §13.3.7（第 280 页） | 一致 |
| 一次传输的三个动作 | 读 → 写 → `DMA_CNDTRx` 后递减 | 参考手册 §13.3.3（第 276 页） | 一致 |
| 传输计数器最大次数 | 65535 | 参考手册 §13.3.3（第 276 页）The amount of data to be transferred (up to 65535) is programmable | 一致 |
| 自增步长 | 等于数据宽度（1 / 2 / 4） | 参考手册 §13.3.3（第 276 页） | 一致 |
| `DMA_BufferSize` 的单位 | 数据个数，位数看方向那侧的宽度设置 | `stm32f10x_dma.h` 第 59~61 行 | 一致 |
| 方向枚举 | `DMA_DIR_PeripheralSRC` / `DMA_DIR_PeripheralDST` | `stm32f10x_dma.h` 第 112~113 行；数据转运工程用 SRC（`System\MyDMA.c` 第 27 行） | 一致 |
| 自增枚举 | `DMA_PeripheralInc_Enable` / `DMA_MemoryInc_Enable` | `stm32f10x_dma.h` 第 124、136 行；`MyDMA.c` 第 23、26 行 | 一致 |
| 数据宽度枚举 | `DMA_*DataSize_Byte` / `_HalfWord` / `_Word` | `stm32f10x_dma.h` 第 148~150、162~164 行；数据转运工程用 Byte（`MyDMA.c` 第 22、25 行） | 一致 |
| 模式枚举 | `DMA_Mode_Normal` / `DMA_Mode_Circular` | `stm32f10x_dma.h` 第 176~177 行；数据转运工程用 Normal（`MyDMA.c` 第 29 行） | 一致 |
| M2M 枚举 | `DMA_M2M_Enable` / `DMA_M2M_Disable` | `stm32f10x_dma.h` 第 203~204 行；数据转运工程用 Enable（`MyDMA.c` 第 30 行） | 一致 |
| 优先级枚举 | VeryHigh / High / Medium / Low | `stm32f10x_dma.h` 第 187~190 行；数据转运工程用 Medium（`MyDMA.c` 第 31 行） | 一致 |
| 正常模式计数到 0 后 | 不再响应请求，要重装必须先关闭通道 | 参考手册 §13.3.3（第 277 页） | 一致 |
| 循环模式计数到 0 后 | 自动重装初值并重载基地址 | 参考手册 §13.3.3（第 277 页） | 一致 |
| 通道关闭后寄存器是否复位 | 不复位，`CCRx` / `CPARx` / `CMARx` 保持初值 | 参考手册 §13.3.3（第 277 页 Note） | 一致 |
| MEM2MEM 与循环模式 | 不能同时使用 | 参考手册 §13.3.3（第 278 页）；`stm32f10x_dma.h` 第 77~78 行 | 一致 |
| 仲裁器优先级层级 | 软件 4 级 + 硬件按通道号（编号小者优先） | 参考手册 §13.3.2（第 276 页） | 一致 |
| 课件映射图上的"固定的硬件优先级" | 通道 1 高 → 通道 7 低 | 课件 Slide 104 图（`image95.png` 右侧标注） | 一致 |
| 中断与事件 | 半传输 HTIF / 传输完成 TCIF / 传输错误 TEIF，各有独立使能位 | 参考手册 §13.3.6（第 279 页 Table 77）；`stm32f10x_dma.h` 第 215~217 行 | 一致 |
| 传输错误的来源与后果 | 读写保留地址空间触发；硬件自动清 EN 关闭通道 | 参考手册 §13.3.5（第 279 页） | 一致 |
| 数据宽度对齐规律 | 源按源宽度、目标按目标宽度各自走步长 | 参考手册 §13.3.4（第 278 页 Table 76）；课件 Slide 105 同表 | 一致 |
| `DMA_Init()` 会不会动 EN 位 | 不会（只清 MEM2MEM/PL/MSIZE/PSIZE/MINC/PINC/CIRC/DIR） | `stm32f10x_dma.c` 第 218~238 行 | 一致 |
| `DMA_SetCurrDataCounter()` 的使用条件 | 只能在通道关闭时调用 | `stm32f10x_dma.c` 第 350 行注释 "This function can only be used when the DMAy_Channelx is disabled." | 一致 |
| 数据转运工程用哪个通道 | `DMA1_Channel1` | `System\MyDMA.c` 第 32、35、45、46、47、49、50 行 | 一致 |
| 数据转运工程接线 | 只有 OLED（`PB8`=SCL、`PB9`=SDA）与 STLINK，没有任何传感器 | 官方接线图 `8-1 DMA数据转运.png`；`Hardware\OLED.c` 第 5~6 行 | 一致 |
| 数据转运工程接线图与 4-1 / 6-1 是同一张图 | 是（官方在接线完全相同时复用同一张图） | SHA256：`8-1 DMA数据转运.png` = `4-1 OLED显示屏.png` = `6-1 定时器定时中断.png` = `9E2250A2668C831077D0D1248736E2FD68C0A994813EE72794EBA769296EBEEF` | 一致（正常复用，非异常） |
| 课件 Slide 106 的数组长度 | 示意图为 7，源码为 4 | `User\main.c` 第 6~7 行 `DataA[4]` / `DataB[4]` | 以源码为准（差异已记入「待核对」） |

> [!note] 出处说明
> 本页的 DMA 资源、请求映射表、三大要素、配置项枚举值与仲裁规则，已逐条对照课程官方课件（Slide 100~107 及其原始图形 `image94/95/96.png`）、ST 官方参考手册 RM0008（§3.1 系统架构、§13.3 DMA 功能描述、Table 76/77/78）与 ST 标准外设库 V3.5.0（`stm32f10x_dma.h` / `stm32f10x_dma.c`）核对；引脚以官方接线图与官方源码为准。
> **仍未核实**：「DMA 不能把 Flash 当搬运对象」这类限制说法、DMA 与 CPU 之间的总线仲裁细节、以及老师的口头讲解原话——这些不在源码、接线图、课件文本与参考手册 DMA 章节中，已留在上面的「待核对」。

