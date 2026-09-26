---
course: 江协科技 STM32入门教程-2023版
chapter: 05-1 EXTI外部中断
type: 分集笔记
source: https://www.bilibili.com/video/BV1th411z7sn?p=11
tags:
  - STM32
  - 江协科技
  - EXTI
  - 中断
  - NVIC
  - AFIO
status: draft
updated: 2026-09-26
---

# 05-1 EXTI 外部中断

> [!abstract] 这一集只解决一个问题
> **怎么让一个引脚上的电平跳变，自动把 CPU 拽去执行我指定的那段代码？**
> 答：引脚 →（AFIO 数据选择器）→ **EXTI** 做边沿检测并置挂起标志 → **NVIC** 按优先级排队 → CPU 跳进 `EXTIx_IRQHandler()` 执行一次，然后回到主循环原地继续。
> 这一集是**纯理论 + 配置骨架**：先把「中断 / 优先级 / NVIC / EXTI / AFIO」这几个概念理清，再落地成六步配置。
>
> 上一章 [[江协STM32 04 OLED（章索引）]] ｜ 章索引 [[江协STM32 05 EXTI外部中断（章索引）]] ｜ 下一集 [[江协STM32 05-2 红外传感器计次与旋转编码器计次]]

## 1 什么是中断

### 1.1 主程序轮询 vs 中断

第 3 章读按键用的是**轮询**（polling）：主循环里一遍遍 `if (GPIO_ReadInputDataBit(...) == 0)`，CPU 被死死绑在「等」这件事上。

**中断**（interrupt）是反过来的思路：CPU 该干嘛干嘛，让外设自己盯着；一旦有情况，外设发一个「中断请求」（interrupt request）把 CPU 打断，CPU 停下当前指令、保存现场、跳去执行一段预先写好的函数，执行完再回来接着跑。

```text
轮询：  CPU ──► 读 ──► 读 ──► 读 ──► 读 ──► ...      （CPU 全耗在等待上）
中断：  CPU ──► 干活 ──► 干活 ──►[被打断]──► 中断函数 ──► 继续干活
                                  ▲
                          引脚边沿 ─┘
```

| | 轮询（polling） | 中断（interrupt） |
| --- | --- | --- |
| 谁来发现事件 | CPU 反复读 | 硬件自动检测 |
| 响应及时性 | 取决于循环一圈多久 | 几乎立即（几个时钟周期） |
| CPU 占用 | 一直占 | 只在事件发生时占 |
| 适合 | 极低频、无实时要求 | 低频但要求及时、需低功耗 |

STM32 的中断源数量庞大：每个外设（定时器、串口、ADC、EXTI……）都能产生中断。这一点在后面的章节会不断出现。

### 1.2 中断优先级的两种

多个中断同时来怎么办？靠**优先级**（priority）。Cortex-M3 的优先级分两级，含义完全不同：

| | 抢占优先级（pre-emption priority） | 响应优先级（subpriority） |
| --- | --- | --- |
| 别称 | 主优先级、抢占优先级 | 子优先级、副优先级 |
| 作用 | **能不能打断**正在执行的中断函数 | 同时到达时**谁先执行** |
| 谁能抢谁 | 数值小的高，可以抢数值大的 | 抢不了，只能排队 |
| 数值范围 | 由分组决定，见 2.2 | 由分组决定，见 2.2 |

两条判定规则，背下来：

> [!tip] 优先级判定两句话
> 1. **抢占优先级高的，可以打断抢占优先级低的**（这叫中断嵌套）。
> 2. **抢占优先级相同的，谁也不能打断谁，只能等对方跑完；同时到达时响应优先级高的先跑。**
>
> 两个值都是**数值越小优先级越高**（和直觉相反，别记反）。

### 1.3 中断嵌套

**中断嵌套**（interrupt nesting）：CPU 正在执行低抢占优先级的中断函数，此时来了一个抢占优先级更高的请求，CPU 会把这层中断函数也「暂停保存」，跳去执行更高优先级的那段，跑完再一层层回来。

```text
主程序 ──► 中断A（抢占=2）──►[中断B 抢占=1 到来]──► 中断B ──► 回到中断A ──► 回到主程序
                                   可以抢
主程序 ──► 中断A（抢占=2）──►[中断C 抢占=2 到来]──► 排队等A跑完 ──► 中断C ──► 主程序
                                   抢不了
```

### 1.4 中断服务函数

响应中断时执行的那段代码叫**中断服务函数**（ISR，Interrupt Service Routine），在 STM32 里就是以 `xxx_IRQHandler` 命名的 C 函数：

- **名字固定**：由启动文件 `startup_stm32f10x_md.s` 里的中断向量表决定，写错一个字母就等于没写，而且**不报错**。
- **参数为空、返回值 void**：`void EXTI15_10_IRQHandler(void)`。
- **不要手动调用**：它是被硬件触发的。想手动触发要用 `EXTI_GenerateSWInterrupt()`。
- **要短**：里面不要延时、不要打印、不要长循环。正确做法是**置一个标志位，让主循环去干活**。

> [!warning] 中断函数里能做的事很有限
> 中断函数执行期间，同优先级及更低优先级的中断都得等着。在里面 `Delay_ms(100)`，就等于让整个系统卡 100 ms。

### 1.5 中断与子函数的区别

同样是一段「被别人调用」的代码，但触发机制不同：

| | 普通子函数 | 中断服务函数 |
| --- | --- | --- |
| 谁调用 | 你的代码显式调用 | 硬件自动跳转 |
| 什么时候执行 | 你说了算 | CPU 说了不算（可随时插入） |
| 现场保护 | 编译器按调用约定处理 | 硬件自动压栈/出栈部分寄存器 |
| 共享变量 | 正常 | **必须用 `volatile` 修饰**，否则编译器可能把它优化进寄存器，主循环永远读到旧值 |

## 2 NVIC 简介

### 2.1 NVIC 是什么

**NVIC** = Nested Vectored Interrupt Controller，**嵌套向量中断控制器**。它是 **Cortex-M3 内核自带的器件**，不在 STM32 的外设区里，所以：

- 它**不需要开时钟**（这一点和第 3 章的 GPIO、本章的 AFIO 完全不同，很多人在这里白开半天）。
- ST 把它对应的库函数放在 **`misc.c`**（头文件 `misc.h`）里，不在 `stm32f10x_gpio.c` / `stm32f10x_exti.c` 中。

NVIC 干三件事：**接收**各外设送来的中断请求、按**优先级**排序、把请求**上报**给 CPU 内核。

一句话记住它在信号链上的位置：

```text
外设产生事件 ──► 外设自己的中断使能位 ──► NVIC ──► CPU 内核 ──► 中断服务函数
                                        （本章主角）
```

外设侧和 NVIC 侧**两级都要开**，少一级就永远进不去中断函数。

### 2.2 优先级分组：4 个 bit 怎么分

Cortex-M3 每个中断的优先级寄存器是 **8 位**，但 STM32 只实现了**高 4 位**（低 4 位读回为 0）。这 4 位怎么切给抢占和响应，由**分组**决定：

| 分组 | 抢占优先级位数 | 响应优先级位数 | 抢占优先级取值 | 响应优先级取值 | 库函数宏 |
| --- | --- | --- | --- | --- | --- |
| 分组 0 | 0 位 | 4 位 | 0（只有一档） | 0 ~ 15 | `NVIC_PriorityGroup_0` |
| 分组 1 | 1 位 | 3 位 | 0 ~ 1 | 0 ~ 7 | `NVIC_PriorityGroup_1` |
| **分组 2** | **2 位** | **2 位** | **0 ~ 3** | **0 ~ 3** | `NVIC_PriorityGroup_2` |
| 分组 3 | 3 位 | 1 位 | 0 ~ 7 | 0 ~ 1 | `NVIC_PriorityGroup_3` |
| 分组 4 | 4 位 | 0 位 | 0 ~ 15 | 0（只有一档） | `NVIC_PriorityGroup_4` |

几点必须清楚：

- **位数相加恒等于 4**：抢占每多 1 位，响应就少 1 位。
- **分组 0 表示「没有抢占能力」**：所有中断抢占优先级都是 0，谁也不打断谁，只按响应优先级排队。
- **分组 4 表示「没有响应优先级」**：只能靠抢占优先级区分先后，先来后到。
- **课程里最常用的是分组 2**（抢占 0~3、响应 0~3），数值小、够用、写起来不容易越界。

> [!warning] 分组全局只能设置一次
> `NVIC_PriorityGroupConfig()` 写的是内核寄存器 `SCB->AIRCR` 的 `PRIGROUP` 位，**全芯片只有一套分组**。后写的覆盖先写的，结果就是「先配好的中断优先级含义被悄悄改掉」。
>
> 正确做法：**在 `main()` 最开头调用一次**，然后所有模块的 `Init()` 里只填抢占/响应优先级的**数值**，不再调用分组函数。第 6 章的 `Timer_Init()` 里那句 `NVIC_PriorityGroupConfig(NVIC_PriorityGroup_2)` 就是沿用本章的写法。

### 2.3 优先级数值与分组的关系

同一个「抢占 = 1、响应 = 1」，在不同分组下含义不同：

```text
分组 2（2 + 2 位）：抢占 0~3、响应 0~3   ← 写 1、1 合法
分组 1（1 + 3 位）：抢占 0~1、响应 0~7   ← 写 1、1 合法，但语义变了
分组 4（4 + 0 位）：抢占 0~15、响应 0    ← 响应优先级只能写 0，写 1 属于越界
```

**超出范围不会报错，硬件只取低位**——这是最难查的一类 bug。所以分组一旦定了，抢占/响应的取值上限就定了。

### 2.4 NVIC 库函数与 IRQChannel 取值

标准库只有两个常用函数：

| 函数 | 作用 |
| --- | --- |
| `NVIC_PriorityGroupConfig(uint32_t NVIC_PriorityGroup)` | 设置优先级分组，**全工程调用一次** |
| `NVIC_Init(NVIC_InitTypeDef* NVIC_InitStruct)` | 配置某一路中断通道（通道号、开关、两个优先级） |

`NVIC_InitTypeDef` 四个成员：

```c
typedef struct
{
  uint8_t NVIC_IRQChannel;                    // 中断通道，取 IRQn_Type 枚举值，如 EXTI15_10_IRQn
  uint8_t NVIC_IRQChannelPreemptionPriority;  // 抢占优先级
  uint8_t NVIC_IRQChannelSubPriority;         // 响应优先级
  FunctionalState NVIC_IRQChannelCmd;         // ENABLE / DISABLE
} NVIC_InitTypeDef;
```

通道号来自 `IRQn_Type` 枚举（在 `stm32f10x.h` 中，内容与 `irqh` 表一致），全部成员都是**负数或非负整数**：Cortex-M3 内核异常是负数（`SysTick_IRQn = -1` 等），STM32 外设从 0 开始编号。常用几个：

| 外设 | 枚举值 | 数值 |
| --- | --- | --- |
| `EXTI0_IRQn` | EXTI Line 0 中断 | 6 |
| `EXTI1_IRQn` | EXTI Line 1 中断 | 7 |
| `EXTI2_IRQn` | EXTI Line 2 中断 | 8 |
| `EXTI3_IRQn` | EXTI Line 3 中断 | 9 |
| `EXTI4_IRQn` | EXTI Line 4 中断 | 10 |
| `EXTI9_5_IRQn` | EXTI Line 9~5 中断（**5 条线共用一个**） | 23 |
| `EXTI15_10_IRQn` | EXTI Line 15~10 中断（**6 条线共用一个**） | 40 |
| `TIM2_IRQn` | TIM2 全局中断 | 28 |

> [!note] 为什么 5~9、10~15 要合并
> 中断向量表只有那么多号。为了省中断号，Line0~Line4 各占一个，**Line5~9 共用 `EXTI9_5_IRQn`，Line10~15 共用 `EXTI15_10_IRQn`**。
>
> 后果：共用的那几路中断进了**同一个函数**，所以函数里**必须先判断是哪条线触发的**——这就是 `EXTI_GetITStatus()` 存在的意义。

## 3 EXTI 简介

**EXTI** = External Interrupt/Event Controller，**外部中断/事件控制器**。它管的是「**边沿**」，不是「电平」——只有电平**发生变化**的那一刻才会触发。

### 3.1 EXTI 能监听的触发源

STM32F103 的 EXTI 一共 **20 条线**（`EXTI_Line0` ~ `EXTI_Line19`），来源分三类：

| 类别 | 线路 | 说明 |
| --- | --- | --- |
| **GPIO 引脚** | Line0 ~ Line15 | **本章主角**。每条线对应「某个端口的第 n 号引脚」 |
| 外设事件 | Line16 PVD、Line17 RTC 闹钟、Line18 USB 唤醒、Line19 以太网唤醒 | 第 13 章 PWR 会用到 PVD |
| 软件触发 | 任意已配置的线 | `EXTI_GenerateSWInterrupt(EXTI_LineX)` |

也就是说，虽然名字里带「外部」，但 EXTI 不只服务 GPIO——**第 6 章的定时器中断其实没经过 EXTI**，它走的是「中断输出控制」直接到 NVIC。所以「凡是中断都要过 EXTI」是错的。

### 3.2 EXTI 与 GPIO 引脚的对应关系

这是本集最核心、也最容易懵的一张图：**EXTI 的线号 = 引脚号**，而**具体是哪个端口（A/B/C）由 AFIO 的数据选择器决定**。

```text
                    AFIO 数据选择器
PA0 ─┐
PB0 ─┼──► 选择器0 ──► EXTI Line0  ──► EXTI0_IRQn
PC0 ─┘                （三选一）

PA1 ─┐
PB1 ─┼──► 选择器1 ──► EXTI Line1  ──► EXTI1_IRQn
PC1 ─┘

  ...                        ...

PA15─┐
PB15─┼──► 选择器15 ─► EXTI Line15 ──► EXTI15_10_IRQn
PC15─┘
```

规则总结成三句话：

1. **线号必须等于引脚号**。PB14 只能接 Line14，不能接 Line13。
2. **每条线的端口由 AFIO 的 16 个数据选择器各选一个**，选择器是「多选一」的开关。
3. **所以 PA0 / PB0 / PC0 三者只能选一个接 Line0**，另外两个和 EXTI 无关。

> [!warning] 为什么 PA0、PB0、PC0 不能同时用 EXTI0
> 因为 **EXTI0 这条线在芯片内部只有一根**，AFIO 的选择器是「16 选 1」而不是「16 路并行」。你把选择器拨到 PB0，PA0 的信号就进不来。
>
> 想同时监控 PA0 和 PB0？只能：把其中一个**改到别的线号**（如 PB1），或者在同一个中断函数里**软件轮询**另一个引脚——后者已经失去中断的意义了。
>
> 反过来，**同端口不同号是安全的**：PA0 用 Line0、PA1 用 Line1，互不冲突；而 **PA0 + PB5 这种「不同端口、不同线号」也没问题**。

## 4 EXTI 内部结构框图

把 EXTI 拆成「边沿检测 → 挂起 → 屏蔽 → 输出」四段来看：

```text
                                    ┌──────────────────────────────────────────┐
输入线 ──► [边沿检测电路] ──►┌──► [请求挂起寄存器 PR] ──► [中断屏蔽寄存器 IMR] ──┼──► NVIC ──► CPU
         (上升沿/下降沿)    │        (置 1 保持)                    │          │
                           │                                       │ 中断方式 │
                           └──────────────────────────────► [事件屏蔽寄存器 EMR] ─► 脉冲发生器 ─► 其他外设
                                                                              （事件方式，不惊动 CPU）
```

### 4.1 边沿检测电路

输入信号同时送进**上升沿检测电路**和**下降沿检测电路**，两路各自输出一个窄脉冲：

```text
输入：   ______┌────────────┐______
               ↑上升沿        ↑下降沿
上升沿脉冲：___┌┐___________________
下降沿脉冲：____________┌┐__________
```

- 两条边沿检测电路的输出**或**在一起，任一有效就置位挂起寄存器。
- 用哪个边沿由 `EXTI_Trigger_Rising` / `EXTI_Trigger_Falling` / `EXTI_Trigger_Rising_Falling` 选择。
- 正因如此，**EXTI 只认跳变、不认电平**：按下按钮保持低电平，只会触发一次（下降沿那一次）。

### 4.2 请求挂起寄存器

**请求挂起寄存器**（Pending Register，`EXTI_PR`）是 20 位，每一位对应一条线。

- 边沿到来 → 对应位置 **1**，并且**保持 1**，直到软件写 1 清除（写 1 清零，不是写 0）。
- 这个「保持」就是 `EXTI_ClearITPendingBit()` 存在的理由：不清，中断标志一直是 1，函数会被反复进入。
- 挂起寄存器**在中断和事件两种方式下都会被置位**——它记录的是「事件发生过」这件事本身。

### 4.3 中断屏蔽寄存器与事件屏蔽寄存器

挂起寄存器后面分成两条路，由两个屏蔽寄存器（Mask Register）控制开关：

| 寄存器 | 全称 | 控制的东西 | 库函数里的对应 |
| --- | --- | --- | --- |
| **IMR** | Interrupt Mask Register，中断屏蔽寄存器 | 允许该线**向 NVIC 发中断请求** | `EXTI_Mode_Interrupt` |
| **EMR** | Event Mask Register，事件屏蔽寄存器 | 允许该线**产生事件脉冲** | `EXTI_Mode_Event` |

> [!note] `EXTI_Init` 里只选了一个 Mode
> `EXTI_Mode_Interrupt`（值 `0x00`）和 `EXTI_Mode_Event`（值 `0x04`）是同一字段的两个取值，标准库的 `EXTI_Init()` 每次只按你给的那一个模式去写 IMR 或 EMR。所以**一次 `EXTI_Init()` 只能二选一**；想同时要中断和事件，得自己再补一句寄存器操作或再调一次。

### 4.4 中断方式 vs 事件方式

这是又一堆「事件 / 中断」概念的坑，一张表分清：

| | 中断方式（Interrupt） | 事件方式（Event） |
| --- | --- | --- |
| 走哪条路 | IMR → NVIC → CPU | EMR → 脉冲发生器 → 其他外设 |
| 占 CPU 吗 | **占**，要跳进中断函数 | **不占**，CPU 完全不知情 |
| 典型用途 | 计次、按键、传感器检测 | 触发 ADC 转换、触发 DMA 搬运、级联定时器 |
| 库函数 | `EXTI_Mode_Interrupt` | `EXTI_Mode_Event` |
| 谁清标志 | 软件 `EXTI_ClearITPendingBit()` | 一般不用管 |

一句话：**中断是「通知 CPU」，事件是「通知硬件」**。第 6 章会大量使用「事件」这个概念（更新事件、TRGO 触发 ADC/DMA），这里先把地基打好。

## 5 AFIO 复用功能

### 5.1 AFIO 是什么

**AFIO** = Alternate Function I/O，**复用功能 IO**。STM32F103 的引脚数量有限，很多外设功能是**分时复用**在同一批引脚上的，AFIO 就是管这些「复用开关」的外设：

| AFIO 管的事 | 库函数 |
| --- | --- |
| **外部中断线选择**（本章用） | `GPIO_EXTILineConfig()` |
| 引脚重映射（如 TIM3 换引脚） | `GPIO_PinRemapConfig()` |
| 引脚配置锁定（防止误改） | `GPIO_PinLockConfig()` |
| 以太网媒体接口（本课程芯片无此功能） | 忽略 |

ST 没有为 AFIO 单独建 `.c` 文件，**它的库函数就写在 `stm32f10x_gpio.c` 里**——所以 `GPIO_EXTILineConfig()` 这个名字里一个 AFIO 都没有，却在配置 AFIO。别被名字骗了。

### 5.2 为什么用 EXTI 一定要开 AFIO 时钟

`GPIO_EXTILineConfig()` 写的是 AFIO 的寄存器（`AFIO_EXTICR1~4`）。**任何外设的寄存器，不打开它的时钟就写不进去**（写操作被硬件忽略，读回恒为复位值），而且**不会报错**。

```c
/* AFIO 挂在 APB2 上，和 GPIO 同一条总线 */
RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOB, ENABLE);  // ① GPIO 自己的时钟
RCC_APB2PeriphClockCmd(RCC_APB2Periph_AFIO,  ENABLE);  // ② AFIO 的时钟：少了这句，中断永远不来
```

> [!warning] 最容易漏的一句话
> 「GPIO 时钟我明明开了呀」——但 EXTI 这条路上有**两个** APB2 外设：GPIO 和 AFIO。只开 GPIO，引脚能读能写，**但 AFIO 的选择器没被拨过去，信号到不了 EXTI**。现象是：引脚电平用万用表测确实在变，程序就是不进中断。

### 5.3 `GPIO_EXTILineConfig()` 的用法

```c
void GPIO_EXTILineConfig(uint8_t GPIO_PortSource, uint8_t GPIO_PinSource);
```

| 参数 | 取值 | 说明 |
| --- | --- | --- |
| `GPIO_PortSource` | `GPIO_PortSourceGPIOA` / `...GPIOB` / `...GPIOC` / ... | **端口**，选 A/B/C |
| `GPIO_PinSource` | `GPIO_PinSource0` ~ `GPIO_PinSource15` | **线号**，注意**不是** `GPIO_Pin_0` 那种带下划线的宏 |

两个参数合起来表达：**「把哪个端口的第几号引脚，接到第几号 EXTI 线上」**。因为线号就等于 `GPIO_PinSource` 的数字，所以引脚号和线号天然一致，不需要额外指定。

```c
/* 把 GPIOB 的 14 号引脚（PB14）接到 EXTI Line14 */
GPIO_EXTILineConfig(GPIO_PortSourceGPIOB, GPIO_PinSource14);
```

> [!tip] 两组宏长得像，别写混
> - `GPIO_Pin_14` → 给 `GPIO_Init()` 用，是**位掩码**（`0x4000`）。
> - `GPIO_PinSource14` → 给 `GPIO_EXTILineConfig()` 用，是**序号**（`14`）。
>
> 写错了编译器通常不报错（都是整数），但行为全错。

## 6 配置流程六步

```text
① 开 RCC 时钟：GPIO + AFIO（两个都要！）
② 配置 GPIO：上拉 / 下拉 / 浮空输入
③ 配置 AFIO：GPIO_EXTILineConfig() 选中断线
④ 配置 EXTI：EXTI_Init() 选边沿、选模式（中断 or 事件）
⑤ 配置 NVIC：选通道、使能、给抢占/响应优先级
⑥ 写中断服务函数：判断标志 → 干活 → 清标志
```

对照第 6 章的定时中断六步，可以看到**只有 ①②④ 换了名字，⑤⑥ 一字不改**：

| 步骤 | 本章 EXTI | 第 6 章 TIM |
| --- | --- | --- |
| ① 时钟 | `RCC_APB2PeriphClockCmd(GPIO / AFIO)` | `RCC_APB1PeriphClockCmd(TIM2)` |
| ② 输入配置 | GPIO 输入模式 | `TIM_InternalClockConfig()` |
| ③ 路由/单元配置 | `GPIO_EXTILineConfig()` | `TIM_TimeBaseInit()` |
| ④ 中断使能 | `EXTI_Init()` | `TIM_ITConfig()` |
| ⑤ NVIC | **相同** | **相同** |
| ⑥ 中断函数 | `EXTIx_IRQHandler` | `TIMx_IRQHandler` |

## 7 完整代码

以 5-2 的 CountSensor（红外计次）为骨架，把六步逐条落地。这里先给出**完整可编译**版本。

### 7.1 CountSensor.c

```c
#include "stm32f10x.h"                  // Device header

uint16_t CountSensor_Count;             // 中断函数里 ++，主循环里读

/**
  * 对射式红外传感器计次初始化：PB14 下降沿触发外部中断
  * 配置顺序就是六步：时钟 → GPIO → AFIO → EXTI → NVIC →（中断函数写在下面）
  */
void CountSensor_Init(void)
{
	/* ① 开时钟：GPIO 和 AFIO 都在 APB2 上，两个都必须开 */
	RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOB, ENABLE);
	RCC_APB2PeriphClockCmd(RCC_APB2Periph_AFIO,  ENABLE);

	/* ② 配置 GPIO：EXTI 要从引脚读电平，所以必须是输入模式
	      上拉输入 = 引脚内部默认拉高，外接开关/传感器只负责把它拉低 */
	GPIO_InitTypeDef GPIO_InitStructure;
	GPIO_InitStructure.GPIO_Mode  = GPIO_Mode_IPU;      // 上拉输入
	GPIO_InitStructure.GPIO_Pin   = GPIO_Pin_14;        // PB14
	GPIO_InitStructure.GPIO_Speed = GPIO_Speed_50MHz;   // 输入模式下速度无意义，填上即可
	GPIO_Init(GPIOB, &GPIO_InitStructure);

	/* ③ 配置 AFIO：把 GPIOB 的 14 号引脚接到 EXTI Line14
	      注意第二个参数是 PinSource（序号），不是 GPIO_Pin_14（位掩码） */
	GPIO_EXTILineConfig(GPIO_PortSourceGPIOB, GPIO_PinSource14);

	/* ④ 配置 EXTI：下降沿触发，中断模式 */
	EXTI_InitTypeDef EXTI_InitStructure;
	EXTI_InitStructure.EXTI_Line    = EXTI_Line14;             // 线号必须等于引脚号
	EXTI_InitStructure.EXTI_LineCmd = ENABLE;                  // 使能这条线
	EXTI_InitStructure.EXTI_Mode    = EXTI_Mode_Interrupt;     // 中断模式；事件模式是 EXTI_Mode_Event
	EXTI_InitStructure.EXTI_Trigger = EXTI_Trigger_Falling;    // 下降沿触发
	EXTI_Init(&EXTI_InitStructure);

	/* ⑤ 配置 NVIC：EXTI15_10 共用一个中断通道
	      分组一般写在 main() 里只调用一次，这里为了代码完整也写一遍 */
	NVIC_PriorityGroupConfig(NVIC_PriorityGroup_2);            // 抢占 2 位 + 响应 2 位

	NVIC_InitTypeDef NVIC_InitStructure;
	NVIC_InitStructure.NVIC_IRQChannel                   = EXTI15_10_IRQn;  // 14 号线落在 15~10 组
	NVIC_InitStructure.NVIC_IRQChannelCmd                = ENABLE;
	NVIC_InitStructure.NVIC_IRQChannelPreemptionPriority = 1;
	NVIC_InitStructure.NVIC_IRQChannelSubPriority        = 1;
	NVIC_Init(&NVIC_InitStructure);
}

/**
  * 读计数值
  */
uint16_t CountSensor_Get(void)
{
	return CountSensor_Count;
}

/**
  * ⑥ 中断服务函数：名字固定，由启动文件的中断向量表决定
  *    EXTI15_10 管着 10~15 六条线，所以必须先判断是不是 Line14 触发的
  */
void EXTI15_10_IRQHandler(void)
{
	if (EXTI_GetITStatus(EXTI_Line14) == SET)   // 判断是不是 EXTI Line14 来的
	{
		/* 再读一次引脚电平来消抖：只有确实是低电平才计数
		   如果只是毛刺跳到下降沿、此刻已经回到高电平，就不计 */
		if (GPIO_ReadInputDataBit(GPIOB, GPIO_Pin_14) == 0)
		{
			CountSensor_Count++;
		}

		EXTI_ClearITPendingBit(EXTI_Line14);    // 必须清中断标志，否则出不去
	}
}
```

### 7.2 CountSensor.h

```c
#ifndef __COUNT_SENSOR_H
#define __COUNT_SENSOR_H

void CountSensor_Init(void);
uint16_t CountSensor_Get(void);

#endif
```

### 7.3 main.c

```c
#include "stm32f10x.h"                  // Device header
#include "Delay.h"
#include "OLED.h"
#include "CountSensor.h"

int main(void)
{
	OLED_Init();
	CountSensor_Init();

	OLED_ShowString(1, 1, "Count:");

	while (1)
	{
		/* 主循环只管显示；计数的活儿全在中断函数里完成 */
		OLED_ShowNum(1, 7, CountSensor_Get(), 5);
	}
}
```

### 7.4 中断服务函数的固定名字

名字写错 = 中断永远不会执行，而且**编译链接都不报错**（因为那只是一个没人调用的普通函数）。可用名字一览：

| 中断向量名 | 管哪些线 | 常见触发引脚 |
| --- | --- | --- |
| `EXTI0_IRQHandler` | Line0 | Px0 |
| `EXTI1_IRQHandler` | Line1 | Px1 |
| `EXTI2_IRQHandler` | Line2 | Px2 |
| `EXTI3_IRQHandler` | Line3 | Px3 |
| `EXTI4_IRQHandler` | Line4 | Px4 |
| `EXTI9_5_IRQHandler` | Line5 ~ Line9 | Px5 ~ Px9 |
| `EXTI15_10_IRQHandler` | Line10 ~ Line15 | Px10 ~ Px15 |

> [!tip] 忘了名字怎么办
> 打开工程里的启动文件 `startup_stm32f10x_md.s`，里面 `DCD` 后面那一串 `xxx_IRQHandler` 就是全部可用名字。另外 ST 的标准库在 `stm32f10x_it.c` 里放了**弱定义**的默认实现，你自己写一个同名函数就会覆盖它。

### 7.5 不用接线也能测中断函数

不确定中断函数到底进没进、名字对不对，可以让软件触发一次：

```c
/* 主循环里调用：手动置一次 EXTI Line14 的挂起位，等价于来了一次边沿 */
EXTI_GenerateSWInterrupt(EXTI_Line14);
```

它绕过边沿检测电路，直接置挂起位，因此能把「GPIO / AFIO / 接线」这一段排除掉，用来定位问题在哪一级。

## 8 易错点

- [ ] 只开了 GPIO 时钟，**忘了 `RCC_APB2PeriphClockCmd(RCC_APB2Periph_AFIO, ENABLE)`** → AFIO 寄存器写不进去，中断永远不来。
- [ ] `GPIO_EXTILineConfig()` 第二个参数写成 `GPIO_Pin_14`（位掩码）而不是 `GPIO_PinSource14`（序号）→ 选择器选错线。
- [ ] GPIO 配成了**输出**模式 → EXTI 读不到输入电平。
- [ ] 中断函数名写错（如漏掉下划线、写成 `EXTI_15_10_IRQHandler`）→ 编译链接都不报错，中断静默失效。
- [ ] 中断函数里**忘了 `EXTI_ClearITPendingBit()`** → 挂起位一直为 1，反复进出中断，主循环卡死。
- [ ] 在 `EXTI9_5_IRQHandler` / `EXTI15_10_IRQHandler` 里**不判断是哪条线** → 误清其他线的标志，丢掉中断。
- [ ] 在 `EXTI9_5_IRQHandler` / `EXTI15_10_IRQHandler` 里写分支时，误用了主程序版的 `EXTI_GetFlagStatus()` / `EXTI_ClearFlag()` → 中断里该用 `EXTI_GetITStatus()` / `EXTI_ClearITPendingBit()`。
- [ ] `NVIC_PriorityGroupConfig()` 在多个模块里重复调用且分组不同 → 全芯片分组被最后一句覆盖，先前配的优先级含义全变。
- [ ] 抢占/响应优先级写超出当前分组范围（如分组 2 里写 5）→ 不报错，硬件只取低位。
- [ ] 同时想用 PA0 和 PB0 → 两者都要 EXTI Line0，硬件不允许。
- [ ] 在中断函数里 `Delay_ms()`、串口打印、长循环 → 阻塞同优先级和低优先级中断。
- [ ] 主循环和中断函数共享的变量没加 `volatile` → 编译器把它缓存进寄存器，主循环读到旧值。

## 9 自测

1. 抢占优先级和响应优先级的区别是什么？抢占都是 2、响应分别是 0 和 3 的两个中断，谁能打断谁？
2. `NVIC_PriorityGroupConfig(NVIC_PriorityGroup_2)` 下，抢占优先级和响应优先级各能取哪些值？为什么这个函数只能调用一次？
3. 为什么用 EXTI 必须打开 AFIO 时钟？`GPIO_EXTILineConfig()` 这个名字里为什么没有 AFIO？
4. PA0、PB0、PC0 能不能同时用于 EXTI？为什么？如果一定要同时监控 PA0 和 PB0 该怎么办？
5. 中断服务函数里为什么必须先 `EXTI_GetITStatus()` 再 `EXTI_ClearITPendingBit()`？两者能不能换顺序？

> [!success]- 参考答案
> 1. 抢占优先级决定**能不能打断**正在执行的中断函数（高抢低）；响应优先级只在**同时到达**时决定谁先跑，抢不了。两者抢占都是 2，**谁也打断不了谁**，只能排队；同时到达时响应 0 的那个先执行。
> 2. 分组 2 = 2 位抢占 + 2 位响应，所以抢占取 0~3、响应取 0~3。该函数写的是内核 `SCB->AIRCR` 的全局分组位，**全芯片只有一套**，多处调用只会互相覆盖，所以在 `main()` 开头调用一次即可。
> 3. 因为 `GPIO_EXTILineConfig()` 实际写的是 **AFIO 的 `AFIO_EXTICR` 寄存器**，外设不开时钟寄存器就写不进去（且不报错）。ST 没有给 AFIO 单独建库文件，这些函数被放进了 `stm32f10x_gpio.c`，所以名字里带 `GPIO_` 却不含 AFIO。
> 4. 不能。EXTI Line0 在芯片内部**只有一条**，由 AFIO 的 16 选 1 数据选择器决定接到哪个端口，同一时刻只能选一个。要同时监控，得把另一个引脚换到别的线号（如 PB1 → Line1），或在同一个中断函数里用软件轮询。
> 5. 因为 `EXTI15_10` / `EXTI9_5` 是**多条线共用一个中断通道**，必须先确认是自己这条线触发的再处理；清标志位是「写 1 清零」的一次性动作，如果先清再判断，判断就恒为假，且可能误清别的线。**先判断、后清标志**。

## 10 待核对

- [ ] 视频中 `NVIC_PriorityGroupConfig()` 的**分组取值**，以及 `PreemptionPriority` / `SubPriority` 的**具体数值**。（本页代码按第三方整理的示例取了 `Group_2` / `1` / `1`，未逐帧核对视频。）
- [ ] 老师对「为什么要开 AFIO 时钟」的**画面演示方式**（是否现场演示了不开时钟的现象）。
- [ ] EXTI 内部结构框图的**逐块讲解顺序**与老师强调的重点（本页按「边沿检测 → 挂起 → 屏蔽 → 输出」组织）。
- [ ] 「中断 vs 事件」在视频里的**具体举例**（本页举的 ADC / DMA / 级联定时器例子为通用用法，非视频原话）。
- [ ] 课程板对射式红外传感器接的**具体引脚**（配套源码镜像中为 PB14，是否与视频一致未核对）。
- [ ] `NVIC_PriorityGroupConfig()` 在视频里是写在 `CountSensor_Init()` 内还是 `main()` 内。

> [!note] 出处说明
> 本页的中断/NVIC/EXTI/AFIO 概念、寄存器行为与库函数签名依据 STM32F10x 标准外设库头文件（`misc.h`、`stm32f10x_exti.h`、`stm32f10x_gpio.h`）、`IRQn_Type` 枚举表与课程配套示例源码（`CountSensor.c` / `CountSensor.h`）整理；六步流程与讲解顺序参照课程公开讲义及多份同课程公开笔记。
> **未逐帧核对视频画面**，优先级分组与优先级数值等细节均列入上方「待核对」，若与视频有出入以视频为准。
