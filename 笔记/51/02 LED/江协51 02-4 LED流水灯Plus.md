---
course: "江协科技 51单片机教程"
chapter: "02 LED"
chapter_num: 2
episode: "02-4"
video_p: 5
title: "LED流水灯Plus"
type: 分集笔记
source: "https://www.bilibili.com/video/BV1Mb411e7re?p=5"
tags:
  - 51单片机
  - 江协科技
status: 待核对
verify: 源码已核对，上机未验证
updated: 2026-09-28
---

# 江协51 02-4 LED流水灯Plus

> [!abstract] 一句话核心
> 把延时函数参数化，流水灯就能按不同速度分段移动。

## 原理讲解

`Delay1ms(xms)` 内部按 1 ms 基准循环，外部传入不同时间。

前两颗 LED 使用 1000 ms，后续 LED 使用 100 ms，因此形成先慢后快的节奏。

## 参数与引脚

| 项目 | 值 |
| --- | --- |
| LED 端口 | P2 |
| 前 2 步 | 1000 ms |
| 后 6 步 | 100 ms |

## 完整代码

### `main.c`

```c
#include <REGX52.H>

void Delay1ms(unsigned int xms);		//@12.000MHz

void main()
{
	while(1)
	{
		P2=0xFE;//1111 1110
		Delay1ms(1000);
		P2=0xFD;//1111 1101
		Delay1ms(1000);
		P2=0xFB;//1111 1011
		Delay1ms(100);
		P2=0xF7;//1111 0111
		Delay1ms(100);
		P2=0xEF;//1110 1111
		Delay1ms(100);
		P2=0xDF;//1101 1111
		Delay1ms(100);
		P2=0xBF;//1011 1111
		Delay1ms(100);
		P2=0x7F;//0111 1111
		Delay1ms(100);
	}
}

void Delay1ms(unsigned int xms)		//@12.000MHz
{
	unsigned char i, j;
	while(xms)
	{
		i = 2;
		j = 239;
		do
		{
			while (--j);
		} while (--i);
		xms--;
	}
}
```

## 易错点

- 函数调用前要有声明或定义。
- 形参应支持较大计数值，避免溢出。

## 自测

> [!question]- 自测 1：参数化延时和固定延时相比好在哪里？
> 点击展开后，先自己完整回答，再对照本篇原理与代码。

> [!question]- 自测 2：如何改成先快后慢？
> 点击展开后，先自己完整回答，再对照本篇原理与代码。

## 待核对

- 未进行 Keil 编译检查。
- 未进行 STC-ISP 下载与硬件现象验证。

## 核对记录

| 日期 | 核对项 | 结果 | 依据/备注 |
| --- | --- | --- | --- |
| 2026-09-28 | 引脚、寄存器、时序参数与工程源码 | 已核对 | 工作区 `work/source_manifest.md` |
| - | 编译、下载、硬件现象 | 待核对 | 需要实际开发板和编译环境 |
