---
course: 江协科技 STM32入门教程-2023版
chapter: 04 OLED
type: 章索引
source: https://www.bilibili.com/video/BV1th411z7sn
tags:
  - STM32
  - 江协科技
  - OLED
  - I2C
  - 索引
status: draft
verify: 官方源码+接线图+课件
updated: 2026-09-26
---

# 04 OLED（章索引）

> [!info] 这一章在讲什么
> 前 3 章的程序只能靠 **LED 亮不亮、串口有没有输出** 来猜它跑得对不对。这一章引入一块 **0.96 寸 128×64 的 OLED 显示屏**：把变量实时打在屏幕上，从此调试不再靠猜。
>
> 全章只有 2 集，但它产出的 `OLED.c` / `OLED.h` 会作为**基础设施**被后面每一章反复 `#include`——第 5 章的中断计数值、第 6 章的 CNT 与占空比、第 7 章的 ADC 采样值，全都靠它显示。
>
> 上级：[[江协STM32 课程总索引]]

## 覆盖视频

| 集 | 标题 | 页码 | 时长 |
| --- | --- | --- | --- |
| 4-1 | OLED调试工具 | [P9](https://www.bilibili.com/video/BV1th411z7sn?p=9) | 14:01 |
| 4-2 | OLED显示屏 | [P10](https://www.bilibili.com/video/BV1th411z7sn?p=10) | 19:55 |

## 本集一句话速览

| 集 | 一句话 | 核心库函数 |
| --- | --- | --- |
| 4-1 | 把厂商写好的 OLED 驱动搬进工程，让变量第一次能"看得见" | `OLED_Init` |
| 4-2 | 用 8 个显示函数把字符、数字、字符串摆到屏幕的任意方格上 | `OLED_ShowString` `OLED_ShowNum` `OLED_ShowHexNum` |

- **两集的分工很干净**：4-1 解决"**接上线、点亮屏**"（硬件 + 通信 + 移植），几乎不写自己的逻辑；4-2 解决"**往屏上写什么**"（API + 坐标 + 字体）。
- **4-1 是本课程第一次接触通信协议**。I2C 的起始/停止/应答这套时序，到第 10 章 `10-3 软件I2C读写MPU6050` 会被完整地再讲一遍并正式封装成 `MyI2C.c`。这里先建立印象，第 10 章再回来看会觉得顺理成章。

## 子笔记

- [[江协STM32 04-1 OLED调试工具]] —— 调试手段对比、0.96 寸模块硬件规格、软件 I2C 时序、驱动移植步骤
- [[江协STM32 04-2 OLED显示屏]] —— 显示 API 全表、4 行 × 16 列坐标约定、页/列寻址与字模取模

## 本章速查

### OLED 常用显示函数速查表

参数 `Line` 为行（1~4），`Column` 为列（1~16），**行列都从 1 开始数**。

| 函数原型 | 作用 | 典型调用 | 备注 |
| --- | --- | --- | --- |
| `OLED_Init(void)` | 上电初始化 + 清屏 | `OLED_Init();` | 内部含 1000×1000 空循环上电延时与 SSD1306 命令序列 |
| `OLED_Clear(void)` | 清屏（全屏写 0x00） | `OLED_Clear();` | 逐页写满 128 个 `0x00` |
| `OLED_ShowChar(Line, Column, Char)` | 显示**1 个 ASCII 字符** | `OLED_ShowChar(1, 1, 'A');` | 字符占 1 列宽、半行高（分上下半页两次写） |
| `OLED_ShowString(Line, Column, *String)` | 显示**字符串** | `OLED_ShowString(1, 3, "Hello");` | 从 `Column` 起逐字向右排 |
| `OLED_ShowNum(Line, Column, Number, Length)` | 显示**无符号十进制** | `OLED_ShowNum(2, 1, 12345, 5);` | 不足 `Length` 格左侧自动补 `0` |
| `OLED_ShowSignedNum(Line, Column, Number, Length)` | 显示**有符号十进制** | `OLED_ShowSignedNum(2, 7, -66, 2);` | **自带 `+`/`-` 号**，会多占 1 列 |
| `OLED_ShowHexNum(Line, Column, Number, Length)` | 显示**十六进制** | `OLED_ShowHexNum(3, 1, 0xAA55, 4);` | 字母用大写 `A`~`F` |
| `OLED_ShowBinNum(Line, Column, Number, Length)` | 显示**二进制** | `OLED_ShowBinNum(4, 1, 0xAA55, 16);` | `Length` 最大 16 |

官方驱动默认引脚：**SCL = PB8、SDA = PB9**（软件 I2C，开漏输出）。

> [!warning] 别把 `Length` 当成"数字本身的值"
> 第 4 个参数是**要占用的字符格数**，和数字大小无关。`OLED_ShowNum(1, 1, 7, 5)` 显示的是 `00007` 而不是 `7`。
>
> 另外注意：**数字字面量前面不要手写 `0`**。`0123` 在 C 语言里是**八进制**（等于十进制 83），不是"补零的 123"。

### 显示函数放到哪一行哪一列

屏幕被切成 **4 行 × 16 列** 共 64 个小方格，每格放 1 个字符：

```text
列  1  2  3  4  5  6  7  8  9 10 11 12 13 14 15 16
行 ┌──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┐
 1 │  │  │  │  │  │  │  │  │  │  │  │  │  │  │  │  │
   ├──┼──┼──┼──┼──┼──┼──┼──┼──┼──┼──┼──┼──┼──┼──┼──┤
 2 │  │  │  │  │  │  │  │  │  │  │  │  │  │  │  │  │
   ├──┼──┼──┼──┼──┼──┼──┼──┼──┼──┼──┼──┼──┼──┼──┼──┤
 3 │  │  │  │  │  │  │  │  │  │  │  │  │  │  │  │  │
   ├──┼──┼──┼──┼──┼──┼──┼──┼──┼──┼──┼──┼──┼──┼──┼──┤
 4 │  │  │  │  │  │  │  │  │  │  │  │  │  │  │  │  │
   └──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┘
```

## 相邻章节

- 上一章：[[江协STM32 03 GPIO（章索引）]] —— GPIO 的八种模式是本章软件 I2C 的基础（`SCL`/`SDA` 配成**开漏输出**）
- 下一章：[[江协STM32 05 EXTI外部中断（章索引）]] —— 中断里数出来的计数值，第一时间就靠 OLED 显示
- 课程主页：<https://www.bilibili.com/video/BV1th411z7sn>

## 核对记录（2026-09-26）

> [!success] 核对依据
> 实际用到的依据（均为本机官方资料）：
> - **官方源码**：`程序源码\STM32Project-有注释版\4-1 OLED显示屏\Hardware\OLED.h`、`OLED.c`、`OLED_Font.h`；`1-4 OLED驱动函数模块\4针脚I2C版本\OLED.h`、`OLED.c`
> - **官方接线图**：`ground-truth\接线图\4-1 OLED显示屏.png`
> - **官方课件**：`ground-truth\课件文本.md`（Slide 38 OLED 简介、Slide 40 驱动函数）
> - **官方驱动页**：<https://jiangxiekeji.com/tutorial/oled.html>

| 核对项 | 笔记原值 | 官方依据 | 结论 |
| --- | --- | --- | --- |
| 速查表函数条目（8 条） | `OLED_Init`/`Clear`/`ShowChar`/`ShowString`/`ShowNum`/`ShowSignedNum`/`ShowHexNum`/`ShowBinNum` | 官方 `OLED.h` 全部 13 行，恰好这 8 条声明 | 一致 |
| 行列约定「Line 1~4、Column 1~16，都从 1 开始」 | 1~4 / 1~16，从 1 开始 | `OLED.c` 各显示函数注释：`Line 行位置，范围：1~4`、`Column 列位置，范围：1~16` | 一致 |
| `OLED_Init` 备注"内部含上电延时与 SSD1306 命令序列" | 上电延时 + 命令序列 | `OLED.c` L275~L278 的 1000×1000 空循环 + L282~L318 命令序列（补充"1000×1000 空循环"） | 一致 |
| `OLED_Clear` 备注"全屏写 0x00" | 全屏写 `0x00` | `OLED.c` L114~L125：8 页各写 128 个 `0x00` | 一致 |
| `OLED_ShowChar` 备注"字符占 1 列宽、半行高" | 1 列宽、半行高 | `OLED.c` L134~L147：分上下半页各写 8 字节 | 一致 |
| `OLED_ShowNum` 备注"不足 Length 格左侧自动补 0" | 左侧自动补 `0` | `OLED.c` L192：`OLED_Pow(10, Length-i-1)` 逐位取数 | 一致 |
| `OLED_ShowSignedNum` 备注"自带 +/- 号，会多占 1 列" | 多占 1 列 | `OLED.c` L210~L221：符号写在 `Column`，数字从 `Column + 1` 起 | 一致 |
| `OLED_ShowHexNum` 备注"字母用大写 A~F" | 大写 `A`~`F` | `OLED.c` L244：`SingleNumber - 10 + 'A'` | 一致 |
| `OLED_ShowBinNum` 备注"Length 最大 16" | 最大 16 | `OLED.c` L254 注释：`Length 要显示数字的长度，范围：1~16` | 一致 |
| 本章驱动默认引脚（原表未列） | 原表未写引脚号 | `OLED.c` L5~L6：SCL=PB8、SDA=PB9（开漏输出 `GPIO_Mode_Out_OD`） | 已补记（原表未列引脚） |
| 4-1 一句话速览「核心库函数 `OLED_Init`」 | `OLED_Init` | `OLED.c` L271 起 | 一致 |
| 4-2 一句话速览「8 个显示函数」 | 8 个 | 官方 `OLED.h` 8 条声明 | 一致 |
| 子笔记描述「4 行 × 16 列坐标约定」 | 4 行 × 16 列 | `OLED.c` 注释与课件 Slide 40 屏幕示意 | 一致 |
| 模块规格（0.96 寸、128×64） | 见 04-1 笔记 | 课件 Slide 38：「0.96 寸 OLED 模块……分辨率：128*64」 | 一致 |

> [!note] 出处说明
> 本页覆盖的集数、标题、页码与时长来自 B 站视频分 P 列表（`api.bilibili.com` 番剧/投稿详情接口）；函数速查表已对照**官方配套源码** `程序源码\STM32Project-有注释版\4-1 OLED显示屏\Hardware\OLED.h`、`OLED.c`（8 条函数原型、行列范围注释、各函数实现）与 `1-4 OLED驱动函数模块\4针脚I2C版本\OLED.h`，引脚号对照 `OLED.c` 与**官方接线图** `接线图\4-1 OLED显示屏.png`，坐标约定对照**官方课件** `ground-truth\课件文本.md` Slide 40，模块规格对照课件 Slide 38 与江协科技官方驱动页 <https://jiangxiekeji.com/tutorial/oled.html>。
> **仍未核实**：B 站分 P 的时长/页码未重新拉取核对；两篇分集笔记「待核对」小节中列出的条目（取模软件具体设置、视频讲解详略、「点灯调试法/注释对照调试法」是否出自视频等）在本页同样未核实。**未逐帧核对视频画面**，若与视频有出入以视频为准。
