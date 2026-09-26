---
course: 江协科技 STM32入门教程-2023版
chapter: 09 USART串口
type: 章索引
source: https://www.bilibili.com/video/BV1th411z7sn
tags:
  - STM32
  - 江协科技
  - USART
  - 串口
  - 索引
status: draft
verify: 官方源码+接线图+课件
updated: 2026-09-26
---

# 09 USART 串口（章索引）

> [!info] 这一章在讲什么
> 前面 8 章打交道的都是「自己板子上的东西」——GPIO、OLED、中断、定时器、ADC、DMA。从第 9 章开始，STM32 要**和别的设备说话**。
>
> USART 就是最基础的那张嘴：**两根线（TX / RX）、一套约定的帧格式、一个事先说好的速率**。第 9 章 6 集全都长在这个外设上——先是协议，再是外设，然后是「怎么把一帧一帧的数据组织成有意义的数据包」，最后是怎么用串口下载程序。
>
> 上级：[[江协STM32 课程总索引]]

## 覆盖视频

| 集数 | 标题 | 页码 | 时长 | 笔记 |
| --- | --- | --- | --- | --- |
| 9-1 | USART串口协议 | [P25](https://www.bilibili.com/video/BV1th411z7sn?p=25) | 32:21 | [[江协STM32 09-1 USART串口协议]] |
| 9-2 | USART串口外设 | [P26](https://www.bilibili.com/video/BV1th411z7sn?p=26) | 39:51 | [[江协STM32 09-2 USART串口外设]] |
| 9-3 | 串口发送&串口发送+接收 | [P27](https://www.bilibili.com/video/BV1th411z7sn?p=27) | 1:00:01 | [[江协STM32 09-3 串口发送与串口发送接收]] |
| 9-4 | USART串口数据包 | [P28](https://www.bilibili.com/video/BV1th411z7sn?p=28) | 20:29 | [[江协STM32 09-4 USART串口数据包]] |
| 9-5 | 串口收发HEX数据包&串口收发文本数据包 | [P29](https://www.bilibili.com/video/BV1th411z7sn?p=29) | 26:27 | [[江协STM32 09-5 串口收发HEX数据包与文本数据包]] |
| 9-6 | FlyMcu串口下载&STLINK Utility | [P30](https://www.bilibili.com/video/BV1th411z7sn?p=30) | 22:00 | 本次未做 |

## 知识地图

一条主线：**先把「说什么、怎么说」定下来（协议），再让硬件替你去说（外设），最后把「说出去的一个个字」组织成有意义的话（数据包）。**

```text
                     ┌─ 9-1 协议      没有代码。电平标准 / 帧格式 / 波特率 / 接线规则
                     │
  串口要能通信 ──────┼─ 9-2 外设      USART 内部结构：TDR / RDR / 移位寄存器 / 波特率发生器 / 状态位与中断
                     │
                     ├─ 9-3 收发      把外设变成函数：Serial_Init / Serial_SendByte / 接收中断 + 回环测试
                     │
                     ├─ 9-4 数据包    一帧一个字节不够用：包头 / 包尾 / 长度 / 校验，HEX 包与文本包
                     │
                     └─ 9-5 应用      两个完整工程：收发 HEX 数据包、收发文本数据包（状态机解析）
```

- **9-1 是「规矩」**：这一集没有一行代码，但后面每一集的每一个参数（波特率、字长、停止位、校验位）都在这里被定义。
- **9-2 是「硬件」**：讲清 `TDR` / `RDR` 为什么能「写进去就发、读出来就收」，以及 `TXE` / `TC` / `RXNE` / `IDLE` 四个状态位各自管什么。
- **9-3 是「代码」**：外设配置的完整写法 + 收发函数封装，是本章唯一的第一手工程实践。
- **9-4 / 9-5 是「应用」**：不再有新的外设知识，全部是「怎么用 9-3 的函数把数据组织好、解析好」。

## 本集一句话速览

> 按任务约定沿用此标题名；下表实际覆盖本章 9-1 ~ 9-6 **每一集**的一句话速览。

| 集 | 一句话 | 核心库函数 / 关键点 |
| --- | --- | --- |
| 9-1 | 讲清电平标准、帧格式、波特率与 TX-RX 交叉接线，没有任何代码 | —（纯协议：起始位 / 数据位 / 校验位 / 停止位） |
| 9-2 | USART 硬件怎么把「一个字节」自动变成「一串电平」，以及四个状态位 | `USART_Init` `USART_Cmd` `USART_SendData` `USART_ReceiveData` `USART_GetFlagStatus` `USART_ITConfig` `USART1_IRQHandler` |
| 9-3 | 把 9-2 的外设封装成 `Serial.c` / `Serial.h`，并加上接收中断做回环测试 | `Serial_Init` `Serial_SendByte` `Serial_SendString` `Serial_Printf` |
| 9-4 | 一帧一个字节不够用，于是约定「包头 + 数据 + 包尾」的数据包 | 数据包结构、状态机思路（无新库函数） |
| 9-5 | 两个完整工程：收发 HEX 数据包、收发文本数据包 | `Serial_SendArray` `Serial_GetRxFlag` / `Serial_GetRxData` |
| 9-6 | 只讲下载工具（FlyMcu / STLINK Utility），没有新外设知识 | 本次未做 |

## 本章速查

### 速查 1 一帧数据的构成

| 字段 | 长度 | 电平 | 本课程取值 |
| --- | --- | --- | --- |
| 起始位 | 1 位 | **固定低电平** | 固定 |
| 数据位 | 8 / 9 位，**低位先行** | 高 = 1，低 = 0 | 8 位 |
| 校验位 | 0 或 1 位 | 由数据位算出 | 无校验 |
| 停止位 | 0.5 / 1 / 1.5 / 2 位 | **固定高电平** | 1 位 |

**空闲时线是高电平**，所以起始位的下降沿就是「一帧来了」。

### 速查 2 三种电平标准

| 标准 | 表示 1 | 表示 0 | 特点 |
| --- | --- | --- | --- |
| TTL | +3.3V 或 +5V | 0V | 芯片引脚直接出来就是它，单端、短距离 |
| RS-232 | −3 ~ −15V | +3 ~ +15V | 单端、**负逻辑**，电脑 DB9 就是它 |
| RS-485 | 两线压差 +2 ~ +6V | 两线压差 −2 ~ −6V | **差分信号**，抗共模干扰、可多设备、可远距离 |

三种之间**不能直连**，必须加电平转换芯片。

### 速查 3 关键公式

设 `PCLK` 为对应总线的时钟（USART1 用 `PCLK2`，USART2/3 用 `PCLK1`；本课程 `PCLK2 = 72 MHz`）。

| 场景 | 公式 |
| --- | --- |
| 位时间 | `T_bit = 1 / 波特率` |
| 一帧位数 | `1 + 数据位 + 校验位 + 停止位`（本课程 = 10） |
| 一帧时间 | `一帧位数 / 波特率`（9600 时 ≈ 1.042 ms） |
| 分频值 | `USARTDIV = PCLK / (16 × 波特率)` |
| 分频值分解 | `USARTDIV = DIV_Mantissa + (DIV_Fraction / 16)` |
| `BRR` 写入值 | `(DIV_Mantissa << 4) \| DIV_Fraction` |
| 9600 的验算 | `USARTDIV = 468.75` → `BRR = 0x1D4C` |

### 速查 4 USART 引脚与总线

| 外设 | TX | RX | 总线 | 时钟使能 |
| --- | --- | --- | --- | --- |
| **USART1** | **PA9** | **PA10** | **APB2** | `RCC_APB2PeriphClockCmd(RCC_APB2Periph_USART1, ENABLE)` |
| USART2 | PA2 | PA3 | APB1 | `RCC_APB1PeriphClockCmd(RCC_APB1Periph_USART2, ENABLE)` |
| USART3 | PB10 | PB11 | APB1 | `RCC_APB1PeriphClockCmd(RCC_APB1Periph_USART3, ENABLE)` |

USART1 的重映射引脚是 PB6 / PB7（需要开 AFIO 时钟 + `GPIO_PinRemapConfig`）；`PA9` / `PA10` 属**默认复用功能**，不需要重映射。

### 速查 5 四个状态位

| 状态位 | 含义 | 什么时候置 1 | 怎么清 | 库函数常量 |
| --- | --- | --- | --- | --- |
| `TXE` | 发送数据寄存器空 | TDR 的数据搬进移位寄存器后 | **写 TDR 自动清** | `USART_FLAG_TXE` |
| `TC` | 发送完成 | 整帧（含停止位）发完 | 读 `SR` 再写 `DR` | `USART_FLAG_TC` |
| `RXNE` | 读数据寄存器非空 | 一帧接收完 | **读 RDR 自动清** | `USART_FLAG_RXNE` |
| `IDLE` | 总线空闲 | RX 上出现空闲帧 | 读 `SR` 再读 `DR` | `USART_FLAG_IDLE` |

### 速查 6 GPIO 模式与中断

| 项目 | 配置 |
| --- | --- |
| TX 引脚（PA9） | `GPIO_Mode_AF_PP` **复用推挽**（控制权交给 USART） |
| RX 引脚（PA10） | `GPIO_Mode_IPU` **上拉输入**（官方 `Serial.c` 的写法） |
| 接收中断 | `USART_ITConfig(USART1, USART_IT_RXNE, ENABLE)` + `NVIC_Init()` |
| 中断服务函数名 | `USART1_IRQHandler`（必须与启动文件一致） |
| 判断中断来源 | `USART_GetITStatus(USART1, USART_IT_RXNE) == SET` |
| 清中断标志 | `USART_ClearITPendingBit(USART1, USART_IT_RXNE)`（读 `DR` 会自动清，这是保险） |

### 速查 7 三种「在哪儿看数据」

| 模式 | 说明 |
| --- | --- |
| HEX / 十六进制 / 二进制模式 | 以**原始数据**的形式显示 |
| 文本 / 字符模式 | 以**原始数据编码后**的形式显示 |

线上跑的永远只是 0 和 1，HEX / 文本只是上位机的显示方式。

## 子笔记

- [[江协STM32 09-1 USART串口协议]] —— 同步/异步、全双工、电平标准、一帧的构成、波特率与位时间、TX-RX 交叉接线
- [[江协STM32 09-2 USART串口外设]] —— USART 内部框图、TDR/RDR 双缓冲、四个状态位、`USARTDIV` 计算、GPIO 复用与中断配置
- [[江协STM32 09-3 串口发送与串口发送接收]] —— 把外设封装成 `Serial.c`，串口发送各函数与接收中断回环
- [[江协STM32 09-4 USART串口数据包]] —— 包头/包尾/定长与变长数据包，HEX 包与文本包
- [[江协STM32 09-5 串口收发HEX数据包与文本数据包]] —— 两个完整工程的状态机解析
- **9-6 FlyMcu串口下载&STLINK Utility —— 本次未做，没有笔记。**

## 相邻章节

- 上一章：[[江协STM32 08 DMA（章索引）]]（[8-1 P23](https://www.bilibili.com/video/BV1th411z7sn?p=23) / [8-2 P24](https://www.bilibili.com/video/BV1th411z7sn?p=24)）
- 下一章：[[江协STM32 10 I2C（章索引）]]（[10-1 P31](https://www.bilibili.com/video/BV1th411z7sn?p=31)）
- 上级：[[江协STM32 课程总索引]]
- 课程主页：<https://www.bilibili.com/video/BV1th411z7sn>

## 待核对

- [ ] 9-6「FlyMcu串口下载&STLINK Utility」本次未做，本页只按你的表列出集数、页码与时长，其内容与「有没有笔记」以总索引口径为准。
- [ ] 本章的「B 站分 P 号与时长」（P25~P30、32:21 / 39:51 / 1:00:01 / 20:29 / 26:27 / 22:00）由你提供，本次未在 B 站接口或官方资料包中重新查证。
- [ ] 老师关于「波特率误差要求」的原话（课件文本中没有这个数字）。
- [ ] 各集视频里的画面演示细节（例如串口助手的实际操作、示波器波形）。

> [!note] 关于 B 站分 P 号与时长
> 「覆盖视频」表里的分 P 号（`p=25`~`p=30`）与六个时长**由本次任务直接给出**。上一章（06 章）的同类数据来自 B 站官方分 P 接口
> `https://api.bilibili.com/x/player/pagelist?bvid=BV1th411z7sn`。本章本次未重新调用该接口核对，故已列入「待核对」。

## 核对记录（2026-09-26）

> [!success] 核对依据
> - 官方配套源码：`STM32Project-有注释版\9-1 串口发送\`（`Hardware\Serial.c`、`Hardware\Serial.h`、`User\main.c`）、`9-2 串口发送+接收\`（同上）
> - 官方接线图：`ground-truth\接线图\9-1 串口发送.png`、`ground-truth\接线图\9-2 串口发送+接收.png`
> - 课程课件文本：`ground-truth\课件文本.md`（Slide 108~114、115 USART 框图、116 USART 基本结构、117~120 数据帧与采样、121 波特率发生器、122 数据模式）
> - 课件配图（PPT 内嵌的官方参考手册图）：`ground-truth\ppt_x\ppt\media\image110.jpeg`（图 248 USART 框图）、`image111.png`（图 249 字长设置）、`image112.png`（图 250 配置停止位）、`image115.png`（图 251 USART_BRR）
> - 官方引脚定义表：`ground-truth\引脚定义_xlsx原文.txt`
> - ST 标准外设库：`STM32F10x_StdPeriph_Lib_V3.5.0\...\inc\stm32f10x_usart.h`、`stm32f10x_rcc.h`、`stm32f10x_gpio.h`

| 核对项 | 笔记原值 | 官方依据 | 结论 |
| --- | --- | --- | --- |
| 集数与页码 | 9-1~9-6 共 6 集，P25~P30 | 与「课程总索引」第 146~151 行逐行一致（总索引 9-1~9-5 有 wikilink，9-6 写「本次未做」） | 一致 |
| 9-6 标记 | 「本次未做」，无笔记 | 总索引第 52 行「已完成 ✅（9-6 未做）」、第 151 行「本次未做」 | 一致 |
| 章节区间 | 课件 Slide 108~126 为 USART 章 | `ground-truth\README.md` 第 68 行「108 ~ 126 USART」 | 一致 |
| 一帧构成表 | 起始位 / 数据位 / 校验位 / 停止位 及各自电平 | 课件 Slide 112 逐字一致 | 一致 |
| 低位先行 | 数据位低位先行 | 课件 Slide 112「低位先行」；参考手册图 249 帧结构为「起始位 \| 位0 \| … \| 位7」 | 一致 |
| 三种电平标准 | TTL / RS-232 / RS-485 的电压与 0/1 对应 | 课件 Slide 111 逐字一致 | 一致 |
| USART 引脚与总线 | USART1 = PA9/PA10 挂 APB2；USART2 = PA2/PA3 挂 APB1；USART3 = PB10/PB11 挂 APB1 | 引脚定义表第 32~33、14~15、23~24 行；`stm32f10x_rcc.h` 第 510、540、541 行；官方 `Serial.c` 第 13 行用 `RCC_APB2PeriphClockCmd` | 一致 |
| USART1 重映射引脚 | PB6 / PB7 | 引脚定义表第 44~45 行「重定义功能」列 | 一致 |
| 波特率公式 | `波特率 = PCLK / (16 × USARTDIV)` | 课件 Slide 121；`stm32f10x_usart.h` 第 54 行 | 一致 |
| 9600 的分频验算 | `USARTDIV = 468.75`、`BRR = 0x1D4C` | 按 `PCLK2 = 72 MHz` 代入；`stm32f10x_usart.c` 的 `25*apbclock/(4*baud)` 算法复算吻合 | 一致 |
| 四个状态位 | `TXE` / `TC` / `RXNE` / `IDLE` 的含义与清除方式 | `stm32f10x_usart.h` 第 323~326 行常量；官方 `Serial.c` 第 68、196~197 行关于自动清标志的注释 | 一致 |
| GPIO 模式 | TX → `GPIO_Mode_AF_PP`；RX → `GPIO_Mode_IPU` | 官方 `9-1\Serial.c` 第 18、21 行；官方 `9-2\Serial.c` 第 21、24、26、29 行（注释「将PA10引脚初始化为上拉输入」） | 一致 |
| 中断相关常量 | `USART1_IRQHandler`、`USART_GetITStatus`、`USART_IT_RXNE`、`USART_ClearITPendingBit` | 官方 `9-2\Serial.c` 第 189、191、195 行；`stm32f10x_usart.h` 第 245、392、393 行 | 一致 |
| 两个接线图内容相同 | 9-1 与 9-2 的接线图按同一套接线绘制 | 两个 PNG 文件字节数相同（均 503254 字节）、**SHA-256 完全相同**（`8f5b8f77…d0b6`）；工程源码也只是 9-1 为 9-2 的真子集，接线不变 | 一致（此处特别说明：并非漏放图片，而是两集接线确实一样） |
| 「本章速查」小节编号 | `速查 1` ~ `速查 7` | 本页 `## 本章速查` 下的小节需要编号，但章索引页正文没有数字章节序号，故用 `速查 N` 而非 `N.1`（06 章索引用的是「全章公式速查」单节写法） | 说明（排版选择，已避开与 06 章的 `2.1`、`2.2`、`5.1` 等小节号混淆） |

> [!note] 出处说明
> 本页的一帧构成、三种电平标准、HEX / 文本模式取自课程课件 Slide 108~114、122 的原文；引脚、总线与时钟使能函数取自 `ground-truth\引脚定义_xlsx原文.txt` 与 ST 标准外设库 `stm32f10x_rcc.h`；`BRR` / `USARTDIV` 与四个状态位在 `stm32f10x_usart.h`、官方参考手册图 248 / 251 中逐个查到；收发函数与中断写法逐行对照官方 `9-1`、`9-2` 两个工程的 `Hardware\Serial.c` 与 `User\main.c`。
> **仍未核实**：B 站分 P 号与时长（本次由任务直接给出，未重新查证）、老师关于「波特率误差」等口头讲解原话、视频画面细节。相关条目已留在上面的「待核对」。
