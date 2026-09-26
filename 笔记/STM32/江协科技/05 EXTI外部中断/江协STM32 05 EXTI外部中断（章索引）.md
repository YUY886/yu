---
course: 江协科技 STM32入门教程-2023版
chapter: 05 EXTI外部中断
type: 章索引
source: https://www.bilibili.com/video/BV1th411z7sn
tags:
  - STM32
  - 江协科技
  - EXTI
  - 中断
  - NVIC
  - 索引
status: draft
updated: 2026-09-26
---

# 05 EXTI 外部中断（章索引）

> [!info] 这一章在讲什么
> 前四章的程序都是「CPU 从头到尾顺着跑」，这一章第一次让 **CPU 被外设打断**：引脚上电平一跳变，硬件就把 CPU 拽去执行一段指定的函数，执行完再回来接着跑。
>
> 这一章篇幅不长（2 集），但它是**第 6 章的地基**——第 6 章 8 集里每一集的定时中断，用的 `NVIC_PriorityGroupConfig`、`NVIC_Init`、`IRQHandler` 写法全部在本章 5-1 讲完。
>
> 上级：[[江协STM32 课程总索引]]

## 覆盖视频

| 集 | 标题 | 页码 | 时长 |
| --- | --- | --- | --- |
| 5-1 | EXTI外部中断 | [P11](https://www.bilibili.com/video/BV1th411z7sn?p=11) | 41:58 |
| 5-2 | 对射式红外传感器计次&旋转编码器计次 | [P12](https://www.bilibili.com/video/BV1th411z7sn?p=12) | 49:31 |

## 本集一句话速览

| 集 | 一句话 | 核心库函数 |
| --- | --- | --- |
| 5-1 | 引脚电平一跳变，CPU 就跳进指定的函数跑一趟 | `GPIO_EXTILineConfig` `EXTI_Init` `NVIC_PriorityGroupConfig` `NVIC_Init` |
| 5-2 | 红外管的遮挡、编码器的转动，都靠外部中断数出来 | `EXTI_GetITStatus` `EXTI_ClearITPendingBit` |

- **5-1 是纯理论 + 配置骨架**：中断系统、NVIC、EXTI 内部结构、AFIO、六步配置流程。看这一集的目标不是背函数，而是画出「引脚 → AFIO → EXTI → NVIC → CPU」这条信号链。
- **5-2 是应用集**：用两个传感器把 5-1 的骨架跑一遍。代码量比 5-1 大，但新原理只有「编码器怎么判方向」这一条。

## 子笔记

- [[江协STM32 05-1 EXTI外部中断]] —— 中断与中断优先级、NVIC 分组、EXTI 内部结构、AFIO、配置六步
- [[江协STM32 05-2 红外传感器计次与旋转编码器计次]] —— 对射式红外管计次、旋转编码器手动鉴相、GPIO 中断函数表

## 本章速查

### NVIC 优先级分组表

Cortex-M3 给每个中断留了 **4 个 bit** 的优先级位，`NVIC_PriorityGroupConfig()` 决定这 4 个 bit 怎么切给**抢占优先级**（pre-emption priority）和**响应优先级**（subpriority）。

| 分组 | 抢占优先级位数 | 响应优先级位数 | 抢占优先级取值 | 响应优先级取值 | 库函数宏 |
| --- | --- | --- | --- | --- | --- |
| 分组 0 | 0 位 | 4 位 | 0 | 0 ~ 15 | `NVIC_PriorityGroup_0` |
| 分组 1 | 1 位 | 3 位 | 0 ~ 1 | 0 ~ 7 | `NVIC_PriorityGroup_1` |
| **分组 2** | **2 位** | **2 位** | **0 ~ 3** | **0 ~ 3** | `NVIC_PriorityGroup_2` |
| 分组 3 | 3 位 | 1 位 | 0 ~ 7 | 0 ~ 1 | `NVIC_PriorityGroup_3` |
| 分组 4 | 4 位 | 0 位 | 0 ~ 15 | 0 | `NVIC_PriorityGroup_4` |

> [!warning] 分组只能设置一次
> `NVIC_PriorityGroupConfig()` 改的是内核寄存器 `SCB->AIRCR` 里的分组位，**整个芯片只认最后写进去的那一组**。多处调用、每组写法还不一样，工程越大越难查。
>
> 惯例：**在 `main()` 开头调用一次**，各个模块的 `Init()` 里只管写抢占/响应优先级的数值。

### EXTI 库函数表

标准外设库把 EXTI 的函数放在 `stm32f10x_exti.c` 里，一共 8 个：

| 函数 | 用途 | 备注 |
| --- | --- | --- |
| `EXTI_DeInit()` | 把 EXTI 全部寄存器恢复复位值 | 模板函数，少用 |
| `EXTI_Init(EXTI_InitTypeDef*)` | 按结构体配置中断线 | **主力函数** |
| `EXTI_StructInit(EXTI_InitTypeDef*)` | 给结构体填默认值 | 模板函数 |
| `EXTI_GenerateSWInterrupt(uint32_t EXTI_Line)` | 软件触发一次外部中断 | 不接硬件也能测中断函数 |
| `EXTI_GetFlagStatus(uint32_t EXTI_Line)` | 读标志位（不判断中断是否使能） | **主程序**里用 |
| `EXTI_ClearFlag(uint32_t EXTI_Line)` | 清标志位 | **主程序**里用 |
| `EXTI_GetITStatus(uint32_t EXTI_Line)` | 读中断标志位（会判断中断使能） | **中断函数**里用 |
| `EXTI_ClearITPendingBit(uint32_t EXTI_Line)` | 清中断挂起位 | **中断函数**里用，必须清 |

配套的 AFIO 函数只有一个，但名字里没有 AFIO：

| 函数 | 用途 |
| --- | --- |
| `GPIO_EXTILineConfig(uint8_t GPIO_PortSource, uint8_t GPIO_PinSource)` | 配置 AFIO 数据选择器，把某个引脚接到某条 EXTI 线上 |

## 与第 6 章的衔接

一句话：**第 6 章的定时中断，只是把触发源从「引脚上的边沿」换成了「定时器的更新事件」，后面 NVIC、中断服务函数、清标志这三件事一模一样。**

```text
第 5 章：  引脚边沿 ──► EXTI ──► NVIC ──► EXTIx_IRQHandler
第 6 章：  更新事件 ──► 中断输出控制 ──► NVIC ──► TIMx_IRQHandler
                        （EXTI_Init 那一层换成了 TIM_ITConfig）
```

| 环节 | 第 5 章 EXTI | 第 6 章 TIM |
| --- | --- | --- |
| 触发源 | GPIO 引脚电平跳变 | 计数器 CNT 计满 ARR（更新事件） |
| 触发源选择 | `GPIO_EXTILineConfig()`（走 AFIO） | `TIM_InternalClockConfig()` 等时钟源选择 |
| 「使能上报」那一级 | `EXTI_Init()` 里 `EXTI_LineCmd = ENABLE` | `TIM_ITConfig(TIMx, TIM_IT_Update, ENABLE)` |
| 往外设时钟 | `RCC_APB2PeriphClockCmd(RCC_APB2Periph_AFIO, ...)` | 不需要（定时器自成一体） |
| NVIC | `NVIC_PriorityGroupConfig()` + `NVIC_Init()` | **完全相同** |
| 中断服务函数 | `EXTI15_10_IRQHandler()` | `TIM2_IRQHandler()` |
| 清标志 | `EXTI_ClearITPendingBit()` | `TIM_ClearITPendingBit()` |

所以把 5-1 的「配置六步」记熟，第 6 章只需要记「第 ②③ 步换成时基单元」这一处差别。详见 [[江协STM32 06 TIM定时器（章索引）]] 与 [[江协STM32 06-1 TIM定时中断]]。

## 全章统一易错点

- [ ] 只开 GPIO 时钟、忘了开 **AFIO 时钟** → `GPIO_EXTILineConfig()` 写进去不生效，中断永远不来。
- [ ] GPIO 配成推挽/开漏**输出** → 输入通道不工作，中断不来。（EXTI 要配**上拉/下拉/浮空输入**）
- [ ] 中断函数里忘了 `EXTI_ClearITPendingBit()` → 出不来中断，主循环卡死。
- [ ] 多个引脚挤在同一条 EXTI 线上（如 PA0 与 PB0）→ 硬件上不可能同时用，只能二选一。
- [ ] `NVIC_PriorityGroupConfig()` 在多个模块里各调一次且分组不一致 → 优先级行为诡异。
- [ ] 在中断函数里做延时、打印、长循环 → 阻塞同优先级和低优先级中断。

## 相邻章节

- 上一章：[[江协STM32 04 OLED（章索引）]] —— OLED 是本章之后所有实验的「显示屏」，中断里数的数全靠它显示
- 下一章：[[江协STM32 06 TIM定时器（章索引）]] —— 直接复用本章的 NVIC 与中断服务函数写法
- 课程主页：<https://www.bilibili.com/video/BV1th411z7sn>

> [!note] 出处说明
> 本页覆盖的视频标题、页码与时长来自 B 站视频分 P 列表；分组表、EXTI 库函数表依据 STM32F10x 标准外设库头文件（`misc.h`、`stm32f10x_exti.h`）与课程配套源码整理。**未逐帧核对视频画面**，若与视频有出入以视频为准；每篇分集笔记末尾附有「待核对」清单。
