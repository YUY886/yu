---
course: 江协科技 STM32入门教程-2023版
chapter: 09-5 串口收发HEX数据包与文本数据包
type: 分集笔记
source: https://www.bilibili.com/video/BV1th411z7sn?p=29
tags:
  - STM32
  - 江协科技
  - USART
  - HEX数据包
  - 文本数据包
  - 状态机
status: draft
verify: 官方源码+接线图+课件
updated: 2026-09-26
---

# 09-5 串口收发 HEX 数据包 & 串口收发文本数据包

> [!abstract] 这一集只解决一个问题
> **怎么把 09-4 讲的数据包格式和状态机，真正写成能跑的代码？**
> 
> 答：两个实验各写一版 `Serial.c`——**HEX 版**（`9-3 串口收发HEX数据包`）用 `0xFF` 包头、`0xFE` 包尾、固定 4 字节数据；**文本版**（`9-4 串口收发文本数据包`）用 `'@'` 包头、`'\r\n'` 包尾、长度可变。两版的中断服务函数里都用 **`static` 状态机变量 `RxState` + `pRxPacket`** 把字节流切成数据包。
> 
> 上一集 [[江协STM32 09-4 USART串口数据包]] ｜ 下一章 [[江协STM32 10 I2C（章索引）]] ｜ 章索引 [[江协STM32 09 USART串口（章索引）]]

## 1 两个实验总览

| | **HEX 数据包版**（官方工程 `9-3 串口收发HEX数据包`） | **文本数据包版**（官方工程 `9-4 串口收发文本数据包`） |
| --- | --- | --- |
| 包格式 | `0xFF` + **固定 4 字节数据** + `0xFE` | `'@'` + **可变长字符数据** + `'\r\n'` |
| 接收数组 | `uint8_t Serial_RxPacket[4]` | `char Serial_RxPacket[100]` |
| 发送数组 | `uint8_t Serial_TxPacket[4]` | **没有**（回应用 `Serial_SendString` 直接发字符串） |
| 发送函数 | 有 `Serial_SendPacket()` | **没有** |
| 取数据方式 | `Serial_GetRxFlag()` + 外部 `Serial_RxPacket[]` | 直接读外部变量 `Serial_RxFlag`，用完自己清 0 |
| 额外外设 | **按键**（`Key.c`，PB1 / PB11），按键 1 触发发一包 | **LED**（`LED.c`，PA1 / PA2），收到指令控制 LED |
| 主循环行为 | 收到包 → **把收到的 4 个字节显示在 OLED 上** | 收到包 → 用 `strcmp` **当命令解析**，回传结果字符串 |
| 串口参数 | 9600、8 位数据、1 位停止、无校验（与 9-1/9-2 完全相同） | 同左 |

> [!warning] 两版的 `Serial.c` 差别很大，**必须逐集照抄，不要互相混用**
> 最容易踩的坑：把 HEX 版的 `Serial_SendPacket()` 抄到文本版、或者把文本版的 `char Serial_RxPacket[100]` 抄到 HEX 版。两版的**全局变量类型、缓冲区大小、状态机迁移条件**都不一样。

> [!note] 任务提示里的 `Serial_GetRxPacket()` 在官方源码中不存在
> 官方 `9-3` 与 `9-4` 两个工程的 `Serial.c` / `Serial.h` 里**都没有** `Serial_GetRxPacket()` 这个函数。实际做法是：
> - **9-3（HEX 版）**：`Serial_GetRxFlag()` 取标志位，数据直接从 `extern uint8_t Serial_RxPacket[]` 里读；
> - **9-4（文本版）**：`Serial.h` 里直接 `extern char Serial_RxPacket[]` 和 `extern uint8_t Serial_RxFlag`，主循环判 `Serial_RxFlag == 1` 并**自己手动清 0**。
> 
> 本页一律以官方源码为准，已记入「待核对」。

## 2 接线

**串口部分与 9-1 / 9-2 完全一致**（官方对相同接线复用同一张接线图，这是正常情况，不是错误）：

| 单片机引脚 | 接到模块的哪一脚 |
| --- | --- |
| **PA9**（USART1_TX） | 模块 **RXD** |
| **PA10**（USART1_RX） | 模块 **TXD** |
| GND | 模块 **GND** |

两个实验各自再多一个外设（引脚以官方源码为准）：

| 实验 | 额外外设 | 引脚 | 来源 |
| --- | --- | --- | --- |
| HEX 版 | 按键 ×2 | **PB1**（按键 1）、**PB11**（按键 2），均上拉输入 | `9-3 串口收发HEX数据包\Hardware\Key.c` 第 17 行 |
| 文本版 | LED ×2 | **PA1**（LED1）、**PA2**（LED2），推挽输出、默认置高（低电平点亮） | `9-4 串口收发文本数据包\Hardware\LED.c` 第 16、21 行 |

## 3 HEX 数据包版（`9-3 串口收发HEX数据包`）

### 3.1 `Serial.c` 全文

```c
#include "stm32f10x.h"                  // Device header
#include <stdio.h>
#include <stdarg.h>

uint8_t Serial_TxPacket[4];				//定义发送数据包数组，数据包格式：FF 01 02 03 04 FE
uint8_t Serial_RxPacket[4];				//定义接收数据包数组
uint8_t Serial_RxFlag;					//定义接收数据包标志位

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
  * 函    数：串口发送一个数组
  * 参    数：Array 要发送数组的首地址
  * 参    数：Length 要发送数组的长度
  * 返 回 值：无
  */
void Serial_SendArray(uint8_t *Array, uint16_t Length)
{
	uint16_t i;
	for (i = 0; i < Length; i ++)		//遍历数组
	{
		Serial_SendByte(Array[i]);		//依次调用Serial_SendByte发送每个字节数据
	}
}

/**
  * 函    数：串口发送一个字符串
  * 参    数：String 要发送字符串的首地址
  * 返 回 值：无
  */
void Serial_SendString(char *String)
{
	uint8_t i;
	for (i = 0; String[i] != '\0'; i ++)//遍历字符数组（字符串），遇到字符串结束标志位后停止
	{
		Serial_SendByte(String[i]);		//依次调用Serial_SendByte发送每个字节数据
	}
}

/**
  * 函    数：次方函数（内部使用）
  * 返 回 值：返回值等于X的Y次方
  */
uint32_t Serial_Pow(uint32_t X, uint32_t Y)
{
	uint32_t Result = 1;	//设置结果初值为1
	while (Y --)			//执行Y次
	{
		Result *= X;		//将X累乘到结果
	}
	return Result;
}

/**
  * 函    数：串口发送数字
  * 参    数：Number 要发送的数字，范围：0~4294967295
  * 参    数：Length 要发送数字的长度，范围：0~10
  * 返 回 值：无
  */
void Serial_SendNumber(uint32_t Number, uint8_t Length)
{
	uint8_t i;
	for (i = 0; i < Length; i ++)		//根据数字长度遍历数字的每一位
	{
		Serial_SendByte(Number / Serial_Pow(10, Length - i - 1) % 10 + '0');	//依次调用Serial_SendByte发送每位数字
	}
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
  * 函    数：自己封装的prinf函数
  * 参    数：format 格式化字符串
  * 参    数：... 可变的参数列表
  * 返 回 值：无
  */
void Serial_Printf(char *format, ...)
{
	char String[100];				//定义字符数组
	va_list arg;					//定义可变参数列表数据类型的变量arg
	va_start(arg, format);			//从format开始，接收参数列表到arg变量
	vsprintf(String, format, arg);	//使用vsprintf打印格式化字符串和参数列表到字符数组中
	va_end(arg);					//结束变量arg
	Serial_SendString(String);		//串口发送字符数组（字符串）
}

/**
  * 函    数：串口发送数据包
  * 参    数：无
  * 返 回 值：无
  * 说    明：调用此函数后，Serial_TxPacket数组的内容将加上包头（FF）包尾（FE）后，作为数据包发送出去
  */
void Serial_SendPacket(void)
{
	Serial_SendByte(0xFF);
	Serial_SendArray(Serial_TxPacket, 4);
	Serial_SendByte(0xFE);
}

/**
  * 函    数：获取串口接收数据包标志位
  * 参    数：无
  * 返 回 值：串口接收数据包标志位，范围：0~1，接收到数据包后，标志位置1，读取后标志位自动清零
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
  * 函    数：USART1中断函数
  * 参    数：无
  * 返 回 值：无
  * 注意事项：此函数为中断函数，无需调用，中断触发后自动执行
  *           函数名为预留的指定名称，可以从启动文件复制
  *           请确保函数名正确，不能有任何差异，否则中断函数将不能进入
  */
void USART1_IRQHandler(void)
{
	static uint8_t RxState = 0;		//定义表示当前状态机状态的静态变量
	static uint8_t pRxPacket = 0;	//定义表示当前接收数据位置的静态变量
	if (USART_GetITStatus(USART1, USART_IT_RXNE) == SET)		//判断是否是USART1的接收事件触发的中断
	{
		uint8_t RxData = USART_ReceiveData(USART1);				//读取数据寄存器，存放在接收的数据变量
		
		/*使用状态机的思路，依次处理数据包的不同部分*/
		
		/*当前状态为0，接收数据包包头*/
		if (RxState == 0)
		{
			if (RxData == 0xFF)			//如果数据确实是包头
			{
				RxState = 1;			//置下一个状态
				pRxPacket = 0;			//数据包的位置归零
			}
		}
		/*当前状态为1，接收数据包数据*/
		else if (RxState == 1)
		{
			Serial_RxPacket[pRxPacket] = RxData;	//将数据存入数据包数组的指定位置
			pRxPacket ++;				//数据包的位置自增
			if (pRxPacket >= 4)			//如果收够4个数据
			{
				RxState = 2;			//置下一个状态
			}
		}
		/*当前状态为2，接收数据包包尾*/
		else if (RxState == 2)
		{
			if (RxData == 0xFE)			//如果数据确实是包尾部
			{
				RxState = 0;			//状态归0
				Serial_RxFlag = 1;		//接收数据包标志位置1，成功接收一个数据包
			}
		}
		
		USART_ClearITPendingBit(USART1, USART_IT_RXNE);		//清除标志位
	}
}
```

### 3.2 `Serial.h`

```c
#ifndef __SERIAL_H
#define __SERIAL_H

#include <stdio.h>

extern uint8_t Serial_TxPacket[];
extern uint8_t Serial_RxPacket[];

void Serial_Init(void);
void Serial_SendByte(uint8_t Byte);
void Serial_SendArray(uint8_t *Array, uint16_t Length);
void Serial_SendString(char *String);
void Serial_SendNumber(uint32_t Number, uint8_t Length);
void Serial_Printf(char *format, ...);

void Serial_SendPacket(void);
uint8_t Serial_GetRxFlag(void);

#endif
```

**两个数组用 `extern` 声明、直接暴露给外部**：`main.c` 里既能写 `Serial_TxPacket[0] = 0x01;`（填要发的数据），也能读 `Serial_RxPacket[0]`（取收到的数据）。**没有** getter 函数。

### 3.3 `main.c`

```c
#include "stm32f10x.h"                  // Device header
#include "Delay.h"
#include "OLED.h"
#include "Serial.h"
#include "Key.h"

uint8_t KeyNum;			//定义用于接收按键键码的变量

int main(void)
{
	/*模块初始化*/
	OLED_Init();		//OLED初始化
	Key_Init();			//按键初始化
	Serial_Init();		//串口初始化
	
	/*显示静态字符串*/
	OLED_ShowString(1, 1, "TxPacket");
	OLED_ShowString(3, 1, "RxPacket");
	
	/*设置发送数据包数组的初始值，用于测试*/
	Serial_TxPacket[0] = 0x01;
	Serial_TxPacket[1] = 0x02;
	Serial_TxPacket[2] = 0x03;
	Serial_TxPacket[3] = 0x04;
	
	while (1)
	{
		KeyNum = Key_GetNum();			//获取按键键码
		if (KeyNum == 1)				//按键1按下
		{
			Serial_TxPacket[0] ++;		//测试数据自增
			Serial_TxPacket[1] ++;
			Serial_TxPacket[2] ++;
			Serial_TxPacket[3] ++;
			
			Serial_SendPacket();		//串口发送数据包Serial_TxPacket
			
			OLED_ShowHexNum(2, 1, Serial_TxPacket[0], 2);	//显示发送的数据包
			OLED_ShowHexNum(2, 4, Serial_TxPacket[1], 2);
			OLED_ShowHexNum(2, 7, Serial_TxPacket[2], 2);
			OLED_ShowHexNum(2, 10, Serial_TxPacket[3], 2);
		}
		
		if (Serial_GetRxFlag() == 1)	//如果接收到数据包
		{
			OLED_ShowHexNum(4, 1, Serial_RxPacket[0], 2);	//显示接收的数据包
			OLED_ShowHexNum(4, 4, Serial_RxPacket[1], 2);
			OLED_ShowHexNum(4, 7, Serial_RxPacket[2], 2);
			OLED_ShowHexNum(4, 10, Serial_RxPacket[3], 2);
		}
	}
}
```

### 3.4 `Serial_SendPacket()`：发送侧就看三行

```c
void Serial_SendPacket(void)
{
	Serial_SendByte(0xFF);                  // 包头
	Serial_SendArray(Serial_TxPacket, 4);   // 4 个数据字节，按原样发
	Serial_SendByte(0xFE);                  // 包尾
}
```

- **包尾固定发 `0xFE`，包头固定发 `0xFF`**，中间的 `Serial_TxPacket[]` 由主循环填。
- 官方注释把这套格式写死在数组定义旁：**「数据包格式：FF 01 02 03 04 FE」**。
- 主循环的验证方式很巧妙：**按键 1 每按一次，4 个字节各自 `++` 再发一包**——串口助手每次收到的都是 `FF 02 03 04 05 FE`、`FF 03 04 05 06 FE`……一眼就能看出包边界对不对。

### 3.5 接收侧：`RxState` 和 `pRxPacket` 逐变量讲清

中断函数开头两个变量：

```c
	static uint8_t RxState = 0;		//定义表示当前状态机状态的静态变量
	static uint8_t pRxPacket = 0;	//定义表示当前接收数据位置的静态变量
```

| 变量 | 类型 | 作用 | 为什么要 `static` |
| --- | --- | --- | --- |
| **`RxState`** | `uint8_t` | 记住**现在处于三个状态中的哪一个**：`0` = 等待包头、`1` = 接收数据、`2` = 等待包尾 | 每收到一个字节就进一次中断，**状态必须跨多次中断调用保留下来**。不加 `static`，局部变量每次进函数都被重新初始化为 0，状态机永远停在状态 0 |
| **`pRxPacket`** | `uint8_t` | 记住**这一包已经收进了几个数据字节**（也就是下一个数据该存到数组的第几个位置） | 同上；它在收到包头时被**归零**（`pRxPacket = 0`），保证每包都从数组第 0 位开始塞 |

> [!tip] `static` 在这里的确切含义
> `static` 修饰的**函数内局部变量**：只在**第一次**执行到定义处时初始化一次，之后**一直存在**（不随函数返回而销毁），下次进函数时保持上次的值。所以 `RxState` / `pRxPacket` 相当于「只有这个中断函数能看见的全局变量」——**不会被其他代码误改**，这正是状态机需要的。

**逐字节走一遍** `FF 01 02 03 04 FE`（假设之前收到过一个杂散字节 `0xAA`）：

| 收到的字节 | 进入前 `RxState` | 命中的分支 | 干了什么 | 退出后 `RxState` | `pRxPacket` | `Serial_RxFlag` |
| --- | --- | --- | --- | --- | --- | --- |
| `0xAA` | 0 | `RxState == 0` | `RxData != 0xFF` → **什么都不做**（丢弃） | 0 | 0 | 0 |
| `0xFF` | 0 | `RxState == 0` | 是包头 → `RxState = 1`、`pRxPacket = 0` | 1 | 0 | 0 |
| `0x01` | 1 | `RxState == 1` | `Serial_RxPacket[0] = 0x01`，`pRxPacket++` → 1，`1 >= 4` 不成立 | 1 | 1 | 0 |
| `0x02` | 1 | `RxState == 1` | `Serial_RxPacket[1] = 0x02`，`pRxPacket` → 2 | 1 | 2 | 0 |
| `0x03` | 1 | `RxState == 1` | `Serial_RxPacket[2] = 0x03`，`pRxPacket` → 3 | 1 | 3 | 0 |
| `0x04` | 1 | `RxState == 1` | `Serial_RxPacket[3] = 0x04`，`pRxPacket` → 4，`4 >= 4` 成立 → `RxState = 2` | 2 | 4 | 0 |
| `0xFE` | 2 | `RxState == 2` | 是包尾 → `RxState = 0`、**`Serial_RxFlag = 1`** | 0 | 4 | **1** |

进完最后一个字节，`Serial_RxPacket[]` = `{0x01, 0x02, 0x03, 0x04}`，主循环马上会用 `Serial_GetRxFlag() == 1` 把它取走并显示到 OLED 第 4 行。

> [!warning] 状态 2 里收到「不是 `0xFE`」的字节会怎样
> 官方代码在 `RxState == 2` 时**只判断 `== 0xFE`，没有 else 分支**：不是包尾就**什么都不做**，状态**留在 2**，继续等下一个字节。也就是说，一旦包尾丢失（比如发漏了 `0xFE`），状态机会**卡在状态 2**，后面所有字节都被当作「等待中的包尾」判一遍——直到真的来一个 `0xFE` 才回到状态 0（此时收到的「包」其实已经是错位的了）。
> 
> 这是官方实现的固有行为，**不是笔误**；官方资料中也没有超时复位机制（见 09-4 笔记第 5 节）。

## 4 文本数据包版（`9-4 串口收发文本数据包`）

### 4.1 `Serial.c` 全文

注意与 HEX 版的差别：**接收数组变成 `char Serial_RxPacket[100]`**（长度可变）、**没有 `Serial_TxPacket`**、**没有 `Serial_SendPacket()`**、**没有 `Serial_GetRxFlag()`**。

```c
#include "stm32f10x.h"                  // Device header
#include <stdio.h>
#include <stdarg.h>

char Serial_RxPacket[100];				//定义接收数据包数组，数据包格式"@MSG\r\n"
uint8_t Serial_RxFlag;					//定义接收数据包标志位

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
  * 函    数：串口发送一个数组
  * 参    数：Array 要发送数组的首地址
  * 参    数：Length 要发送数组的长度
  * 返 回 值：无
  */
void Serial_SendArray(uint8_t *Array, uint16_t Length)
{
	uint16_t i;
	for (i = 0; i < Length; i ++)		//遍历数组
	{
		Serial_SendByte(Array[i]);		//依次调用Serial_SendByte发送每个字节数据
	}
}

/**
  * 函    数：串口发送一个字符串
  * 参    数：String 要发送字符串的首地址
  * 返 回 值：无
  */
void Serial_SendString(char *String)
{
	uint8_t i;
	for (i = 0; String[i] != '\0'; i ++)//遍历字符数组（字符串），遇到字符串结束标志位后停止
	{
		Serial_SendByte(String[i]);		//依次调用Serial_SendByte发送每个字节数据
	}
}

/**
  * 函    数：次方函数（内部使用）
  * 返 回 值：返回值等于X的Y次方
  */
uint32_t Serial_Pow(uint32_t X, uint32_t Y)
{
	uint32_t Result = 1;	//设置结果初值为1
	while (Y --)			//执行Y次
	{
		Result *= X;		//将X累乘到结果
	}
	return Result;
}

/**
  * 函    数：串口发送数字
  * 参    数：Number 要发送的数字，范围：0~4294967295
  * 参    数：Length 要发送数字的长度，范围：0~10
  * 返 回 值：无
  */
void Serial_SendNumber(uint32_t Number, uint8_t Length)
{
	uint8_t i;
	for (i = 0; i < Length; i ++)		//根据数字长度遍历数字的每一位
	{
		Serial_SendByte(Number / Serial_Pow(10, Length - i - 1) % 10 + '0');	//依次调用Serial_SendByte发送每位数字
	}
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
  * 函    数：自己封装的prinf函数
  * 参    数：format 格式化字符串
  * 参    数：... 可变的参数列表
  * 返 回 值：无
  */
void Serial_Printf(char *format, ...)
{
	char String[100];				//定义字符数组
	va_list arg;					//定义可变参数列表数据类型的变量arg
	va_start(arg, format);			//从format开始，接收参数列表到arg变量
	vsprintf(String, format, arg);	//使用vsprintf打印格式化字符串和参数列表到字符数组中
	va_end(arg);					//结束变量arg
	Serial_SendString(String);		//串口发送字符数组（字符串）
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
	static uint8_t RxState = 0;		//定义表示当前状态机状态的静态变量
	static uint8_t pRxPacket = 0;	//定义表示当前接收数据位置的静态变量
	if (USART_GetITStatus(USART1, USART_IT_RXNE) == SET)	//判断是否是USART1的接收事件触发的中断
	{
		uint8_t RxData = USART_ReceiveData(USART1);			//读取数据寄存器，存放在接收的数据变量
		
		/*使用状态机的思路，依次处理数据包的不同部分*/
		
		/*当前状态为0，接收数据包包头*/
		if (RxState == 0)
		{
			if (RxData == '@' && Serial_RxFlag == 0)		//如果数据确实是包头，并且上一个数据包已处理完毕
			{
				RxState = 1;			//置下一个状态
				pRxPacket = 0;			//数据包的位置归零
			}
		}
		/*当前状态为1，接收数据包数据，同时判断是否接收到了第一个包尾*/
		else if (RxState == 1)
		{
			if (RxData == '\r')			//如果收到第一个包尾
			{
				RxState = 2;			//置下一个状态
			}
			else						//接收到了正常的数据
			{
				Serial_RxPacket[pRxPacket] = RxData;		//将数据存入数据包数组的指定位置
				pRxPacket ++;			//数据包的位置自增
			}
		}
		/*当前状态为2，接收数据包第二个包尾*/
		else if (RxState == 2)
		{
			if (RxData == '\n')			//如果收到第二个包尾
			{
				RxState = 0;			//状态归0
				Serial_RxPacket[pRxPacket] = '\0';			//将收到的字符数据包添加一个字符串结束标志
				Serial_RxFlag = 1;		//接收数据包标志位置1，成功接收一个数据包
			}
		}
		
		USART_ClearITPendingBit(USART1, USART_IT_RXNE);		//清除标志位
	}
}
```

### 4.2 `Serial.h`

```c
#ifndef __SERIAL_H
#define __SERIAL_H

#include <stdio.h>

extern char Serial_RxPacket[];
extern uint8_t Serial_RxFlag;

void Serial_Init(void);
void Serial_SendByte(uint8_t Byte);
void Serial_SendArray(uint8_t *Array, uint16_t Length);
void Serial_SendString(char *String);
void Serial_SendNumber(uint32_t Number, uint8_t Length);
void Serial_Printf(char *format, ...);

#endif
```

> [!warning] 文本版**把标志位变量本身 `extern` 出去了**
> 和 HEX 版不一样：HEX 版用 `Serial_GetRxFlag()` 函数封装，读一次自动清 0；文本版直接把 `extern uint8_t Serial_RxFlag` 暴露出来，**主循环必须自己记得清 0**。官方 `main.c` 最后那一行 `Serial_RxFlag = 0;` 就是干这个的，注释原文是「**处理完成后，需要将接收数据包标志位清零，否则将无法接收后续数据包**」。

### 4.3 `main.c`

```c
#include "stm32f10x.h"                  // Device header
#include "Delay.h"
#include "OLED.h"
#include "Serial.h"
#include "LED.h"
#include "string.h"

int main(void)
{
	/*模块初始化*/
	OLED_Init();		//OLED初始化
	LED_Init();			//LED初始化
	Serial_Init();		//串口初始化
	
	/*显示静态字符串*/
	OLED_ShowString(1, 1, "TxPacket");
	OLED_ShowString(3, 1, "RxPacket");
	
	while (1)
	{
		if (Serial_RxFlag == 1)		//如果接收到数据包
		{
			OLED_ShowString(4, 1, "                ");
			OLED_ShowString(4, 1, Serial_RxPacket);				//OLED清除指定位置，并显示接收到的数据包
			
			/*将收到的数据包与预设的指令对比，以此决定将要执行的操作*/
			if (strcmp(Serial_RxPacket, "LED_ON") == 0)			//如果收到LED_ON指令
			{
				LED1_ON();										//点亮LED
				Serial_SendString("LED_ON_OK\r\n");				//串口回传一个字符串LED_ON_OK
				OLED_ShowString(2, 1, "                ");
				OLED_ShowString(2, 1, "LED_ON_OK");				//OLED清除指定位置，并显示LED_ON_OK
			}
			else if (strcmp(Serial_RxPacket, "LED_OFF") == 0)	//如果收到LED_OFF指令
			{
				LED1_OFF();										//熄灭LED
				Serial_SendString("LED_OFF_OK\r\n");			//串口回传一个字符串LED_OFF_OK
				OLED_ShowString(2, 1, "                ");
				OLED_ShowString(2, 1, "LED_OFF_OK");			//OLED清除指定位置，并显示LED_OFF_OK
			}
			else						//上述所有条件均不满足，即收到了未知指令
			{
				Serial_SendString("ERROR_COMMAND\r\n");			//串口回传一个字符串ERROR_COMMAND
				OLED_ShowString(2, 1, "                ");
				OLED_ShowString(2, 1, "ERROR_COMMAND");			//OLED清除指定位置，并显示ERROR_COMMAND
			}
			
			Serial_RxFlag = 0;			//处理完成后，需要将接收数据包标志位清零，否则将无法接收后续数据包
		}
	}
}
```

**主循环干三件事**：

1. **清屏再显示**：先写 16 个空格（`"                "`）把 OLED 上这一行擦掉，再写新字符串——否则短字符串会**盖不住**上一次的长字符串，看起来像乱码。
2. **`strcmp` 当命令解析**：`strcmp(Serial_RxPacket, "LED_ON") == 0` 才算完全相等（`strcmp` 来自 `#include "string.h"`）。之所以能直接用 `strcmp`，是因为中断里在包尾处补了 `Serial_RxPacket[pRxPacket] = '\0'`，让字符数组变成了一个**合法 C 字符串**。
3. **用完清标志**：`Serial_RxFlag = 0;` 放在 `if` 体最后。不清 0，`Serial_RxFlag` 一直是 1，中断里状态 0 的 `&& Serial_RxFlag == 0` 就永远不成立，**再也收不到下一包**。

### 4.4 接收侧：文本版状态机逐变量讲清

| 变量 | 作用 |
| --- | --- |
| **`RxState`** | 三个状态：`0` = 等待包头 `'@'`、`1` = 接收数据**并等第一个包尾 `'\r'`**、`2` = 等第二个包尾 `'\n'` |
| **`pRxPacket`** | 这一包收进了几个字符，同时也是下一个字符该写入的下标；收到 `'@'` 时归零 |
| **`Serial_RxFlag`** | 一包收完置 1；**由主循环清 0**（文本版没有自动清零的 getter） |

**逐字节走一遍** `@LED_ON\r\n`：

| 收到的字节 | 进入前 `RxState` | 命中的分支 | 干了什么 | 退出后 `RxState` | `pRxPacket` | `Serial_RxFlag` |
| --- | --- | --- | --- | --- | --- | --- |
| `'@'` | 0 | `RxState == 0` | 是包头**且 `Serial_RxFlag == 0`** → `RxState = 1`、`pRxPacket = 0` | 1 | 0 | 0 |
| `'L'` | 1 | `RxState == 1` | 不是 `'\r'` → `Serial_RxPacket[0] = 'L'`，`pRxPacket` → 1 | 1 | 1 | 0 |
| `'E'` | 1 | `RxState == 1` | `[1] = 'E'`，`pRxPacket` → 2 | 1 | 2 | 0 |
| `'D'` | 1 | `RxState == 1` | `[2] = 'D'`，`pRxPacket` → 3 | 1 | 3 | 0 |
| `'_'` | 1 | `RxState == 1` | `[3] = '_'`，`pRxPacket` → 4 | 1 | 4 | 0 |
| `'O'` | 1 | `RxState == 1` | `[4] = 'O'`，`pRxPacket` → 5 | 1 | 5 | 0 |
| `'N'` | 1 | `RxState == 1` | `[5] = 'N'`，`pRxPacket` → 6 | 1 | 6 | 0 |
| `'\r'` | 1 | `RxState == 1` | **是第一个包尾** → `RxState = 2`（**不存进数组**） | 2 | 6 | 0 |
| `'\n'` | 2 | `RxState == 2` | 是第二个包尾 → `RxState = 0`、`Serial_RxPacket[6] = '\0'`、**`Serial_RxFlag = 1`** | 0 | 6 | **1** |

结果：`Serial_RxPacket` = `"LED_ON"`（第 6 位被补上了 `'\0'`），主循环 `strcmp` 一比就中，点亮 LED 并回传 `LED_ON_OK\r\n`。

> [!tip] 状态 0 里那个 `&& Serial_RxFlag == 0` 是干什么的
> 它的作用是**「上一包还没被主循环取走时，不许开始收下一包」**。
> 
> 假设没有这个条件：主循环还没轮到处理第 1 包，第 2 包的 `'@'` 就来了，于是 `pRxPacket` 被清零、**第 1 包的数据被第 2 包覆盖**，而 `Serial_RxFlag` 一直是 1，主循环醒来后拿到的是**被覆盖后的数据**——丢了一整包还查不出原因。
> 
> 加上这个条件后，第 1 包没收走之前来的 `'@'` 会被**当成普通字节丢掉**（状态 0 里 `if` 不成立，什么都不做），相当于**主动丢包**。这是官方实现保证「不读到半包数据」的办法。

> [!warning] 文本版没有长度上限检查
> `pRxPacket` 是 `uint8_t`，`Serial_RxPacket` 只有 **100 字节**。官方中断里**没有** `if (pRxPacket >= 100) 复位` 之类的保护：如果对方发来一个 `'@'` 后面跟上超过 100 个字符才给 `'\r'`，`Serial_RxPacket[pRxPacket]` 就会**越界写**，把相邻内存（`Serial_RxFlag` 等）写坏。
> 
> 这是官方实现的固有情况，课堂实验（手工发短指令）不会触发；**做实际项目时务必自己加上长度检查**。

## 5 用串口助手验证

两个实验的验证方式对照（**发送/接收的模式选错是最大的坑**）：

| | HEX 数据包版 | 文本数据包版 |
| --- | --- | --- |
| 串口助手模式 | **HEX 模式** | **文本模式** |
| 波特率 / 数据位 / 停止位 / 校验 | `9600` / `8` / `1` / `无` | 同左 |
| 发什么 | `FF 01 02 03 04 FE` | `@LED_ON` 后面跟**回车+换行**（`\r\n`） |
| 单片机反应 | OLED 第 4 行显示 `01 02 03 04` | LED1 亮；OLED 第 2 行显示 `LED_ON_OK` |
| 单片机回什么 | 按键 1 按下时发一包（自增的 4 个字节） | 串口助手收到 `LED_ON_OK` + 换行 |

**HEX 版操作步骤**：

1. 打开串口助手，选择最小系统板对应的 **COM 口**，参数设成 **9600 / 8 / 1 / 无**，打开串口。
2. 把发送区切到 **HEX 模式**（十六进制发送），输入 `FF 01 02 03 04 FE`，发送。
3. 看 OLED 第 4 行 `RxPacket`：应显示 `01 02 03 04`。
4. 按一下板上的**按键 1**：OLED 第 2 行 `TxPacket` 变成 `02 03 04 05`，串口助手（HEX 显示）应收到 `FF 02 03 04 05 FE`。**再按一次**得到 `FF 03 04 05 06 FE`——包头包尾始终在正确位置，说明包边界正确。
5. 故意发一包缺尾巴的 `FF 0A 0B 0C 0D`：OLED 不会更新（包尾没等到），此时状态机停在状态 2；再补发一个 `FE`，OLED 第 4 行就会显示 `0A 0B 0C 0D`。

**文本版操作步骤**：

1. 同样打开串口，参数 **9600 / 8 / 1 / 无**。
2. 把发送区切到**文本模式**，发送 `@LED_ON`，**并在末尾附上回车换行**（多数串口助手有「发送新行 / 追加回车换行」勾选项，等价于在 `@LED_ON` 后面补 `\r\n` 两个字节）。
3. LED1 应点亮，OLED 第 2 行显示 `LED_ON_OK`，第 4 行显示 `LED_ON`；串口助手收到 `LED_ON_OK`。
4. 发 `@LED_OFF`（同样带 `\r\n`）→ LED1 熄灭，OLED 显示 `LED_OFF_OK`。
5. 发 `@ABC`（带 `\r\n`）→ 这是未知指令，串口助手收到 `ERROR_COMMAND`，OLED 第 2 行显示 `ERROR_COMMAND`。
6. **不带 `\r\n` 只发 `@LED_ON`**：包永远收不完（状态停在 1 或 2），OLED 第 4 行不动——这正是「包尾是必需的」。

> [!warning] 上面这些操作步骤的细节不属于官方材料
> 协议本身（发什么字节、回什么字符串）**全部来自官方源码**；但「串口助手怎么点、勾选项叫什么名字」取决于具体使用的串口助手软件，**课件文本与源码里都没有这部分内容**。本页按通用串口助手的行为写成操作建议，具体界面请以自己用的工具为准，已记入「待核对」。

> [!note] 回传的字符串其实不是合法文本数据包
> 官方 `main.c` 回传的是 `"LED_ON_OK\r\n"`、`"LED_OFF_OK\r\n"`、`"ERROR_COMMAND\r\n"`——**有 `\r\n` 结尾但没有 `'@'` 包头**，所以严格按本集的包格式看，它们并不是合法的文本数据包。这没有问题，因为它们是发给**电脑**看的提示信息，不是发给自己的；但如果希望对方也能用同一套状态机解析，回传内容就要写成 `"@LED_ON_OK\r\n"` 这种形式。这一点属于按源码事实的说明，官方资料中未见相关讨论。

## 易错点

- [ ] **HEX 版和文本版的 `Serial.c` 互相混抄** → 数组类型、缓冲区大小、状态机条件全对不上（HEX 版固定 4 字节 + `0xFE` 包尾；文本版 100 字节 + `'\r'` `'\n'` 双包尾）。
- [ ] `RxState` / `pRxPacket` **忘了加 `static`** → 每次进中断都被重新初始化，状态机永远停在状态 0，一个包都收不到。
- [ ] 忘了 `USART_ITConfig(USART1, USART_IT_RXNE, ENABLE)` 或忘了配 NVIC → 不进中断（和 09-3 是同一个坑）。
- [ ] 收到包头时**忘了 `pRxPacket = 0`** → 第二包的数据从上一次的下标继续写，数据错位。
- [ ] 文本版忘了在包尾处补 `Serial_RxPacket[pRxPacket] = '\0'` → `strcmp` 读到数组后面的垃圾数据，怎么都不相等。
- [ ] 文本版主循环**忘了 `Serial_RxFlag = 0;`** → 只能收第一包，之后永远收不到（状态 0 的 `&& Serial_RxFlag == 0` 不再成立）。官方注释原文已提醒这一点。
- [ ] HEX 版用 `Serial_GetRxFlag()` 之外又手动去清 `Serial_RxFlag` → 没必要，函数内部已经清了。
- [ ] OLED 显示前**忘了先写空格清残留** → 长字符串之后的短字符串显示成「尾巴没擦掉」的样子。
- [ ] 串口助手**模式选错**（HEX 包用文本模式发、或文本包用 HEX 模式发）→ 发出去的字节和协议完全对不上。
- [ ] 文本包**只发 `@LED_ON` 不带 `\r\n`** → 包尾不满，状态机永远收不完。
- [ ] 在 `'\r'` 之前插入换行或用其他换行符（只有 `\n`）→ 状态停在 1，收不完。
- [ ] 文本数据里想包含字符 `'\r'` 或 `@` → 前者会被当成第一个包尾，后者虽然只在状态 0 判定，但会让数据变长，**这两种字符不能作为文本包的数据内容**。
- [ ] 发超过 100 个字符的文本包 → **数组越界**，官方代码没有长度检查（HEX 版则被固定成 4 字节，多发的字节会被当成包尾处理）。
- [ ] 以为 `Serial_SendPacket()` 是两版都有的 → **只有 HEX 版有**，文本版直接 `Serial_SendString()`。
- [ ] 忘了按键 / LED 的初始化（`Key_Init()` / `LED_Init()`）→ 按了没反应或灯不亮。

## 自测

1. `RxState` 和 `pRxPacket` 为什么必须加 `static`？不加会怎样？
2. HEX 版里 `pRxPacket` 在什么时候被归零？如果不归零会出什么问题？
3. 文本版的 `RxState == 0` 分支里那个 `&& Serial_RxFlag == 0` 起什么作用？去掉它会发生什么？
4. 文本版为什么在 `'\n'` 的处理里要写 `Serial_RxPacket[pRxPacket] = '\0'`？不写会怎样？
5. 两个实验里，`Serial_GetRxFlag()` 分别用在哪一版？另一版是怎么取数据、怎么清标志的？

> [!success]- 参考答案
> 1. 因为状态（当前处于哪个阶段）和数据位置必须**跨多次中断调用保存**。每次进中断函数，普通局部变量都会被重新创建并初始化为初值，`RxState` 会永远是 0、`pRxPacket` 永远是 0，状态机无法推进。`static` 让变量只初始化一次、之后一直保留上次的值。
> 2. 在**状态 0 收到包头（`0xFF`）时**归零（`pRxPacket = 0`）。不归零的话，第二包的数据会从上一包结束时的下标继续往后写，导致新包数据错位、上一包的旧数据残留。
> 3. 作用是「**上一包还没被主循环取走时，不允许开始接收下一包**」。去掉它，第 2 包的 `'@'` 会在主循环处理第 1 包之前把 `pRxPacket` 清零并覆盖第 1 包的数据，而标志位一直是 1，主循环最后拿到的是被覆盖的数据——静默丢包/数据错乱。
> 4. 因为主循环要用 `strcmp()` 把这个字符数组当 **C 字符串**比较，而 C 字符串必须以 `'\0'` 结尾。不写的话 `strcmp` 会继续读到数组后面的垃圾数据，永远比不相等（即使收到的确实是 `LED_ON`）。
> 5. **只有 HEX 版（`9-3`）有 `Serial_GetRxFlag()`**：它返回 1 的同时把 `Serial_RxFlag` 自动清 0，数据从 `extern uint8_t Serial_RxPacket[]` 直接读。**文本版（`9-4`）没有这个函数**：`Serial.h` 里 `extern char Serial_RxPacket[]` 和 `extern uint8_t Serial_RxFlag`，主循环判 `Serial_RxFlag == 1` 取数据，处理完**必须自己写 `Serial_RxFlag = 0;`**。

## 待核对

- [ ] 任务提示中提到的 `Serial_GetRxPacket()` **在官方 9-3 / 9-4 工程源码中不存在**；本页按源码写成 `Serial_GetRxFlag()`（HEX 版）+ 直接 `extern` 数组（两版都如此），需确认是否另有出处。
- [ ] 串口助手的具体操作界面（勾选项名称、如何追加回车换行、HEX 模式的开关位置）——官方源码与课件文本中均无此内容，本页第 5 节按通用助手行为写成操作建议。
- [ ] 视频中老师对「HEX 版按键发一包」和「文本版指令回传」的现场演示过程与口头讲解原话。
- [ ] 文本版回传字符串（`LED_ON_OK\r\n` 等）不带 `'@'` 包头、严格说不是合法文本数据包——官方资料中未见对此的说明，本页只作事实性提示。
- [ ] 文本版 `Serial_RxPacket[100]` 的越界风险、HEX 版包尾丢失后状态机卡在状态 2 的行为，官方资料中均未讨论；本页按源码事实说明，未替官方补写保护措施。
- [ ] 9-3 / 9-4 官方接线图中按键与 LED 的具体接法（本页按键 PB1/PB11、LED PA1/PA2 取自官方 `Key.c` / `LED.c`，接线图仅作旁证）。

## 核对记录（2026-09-26）

> [!success] 核对依据
> - 官方配套源码：`STM32Project-有注释版\9-3 串口收发HEX数据包\Hardware\Serial.c`、`Serial.h`、`User\main.c`、`Hardware\Key.c`
> - 官方配套源码：`STM32Project-有注释版\9-4 串口收发文本数据包\Hardware\Serial.c`、`Serial.h`、`User\main.c`、`Hardware\LED.c`
> - 官方接线图：`ground-truth\接线图\9-1 串口发送.png`、`9-2 串口发送+接收.png`（串口部分接线相同，课程作者复用同一张图）、`9-3 串口收发HEX数据包.png`、`9-4 串口收发文本数据包.png`
> - 课程课件文本：`ground-truth\课件文本.md`（Slide 122 数据模式、123 HEX 数据包、124 文本数据包、125 HEX 数据包接收、126 文本数据包接收）
> - ST 标准外设库：`STM32F10x_StdPeriph_Lib_V3.5.0\...\inc\stm32f10x_usart.h`

| 核对项 | 笔记原值 | 官方依据 | 结论 |
| --- | --- | --- | --- |
| HEX 包格式 | `0xFF` + 4 字节 + `0xFE` | `9-3\Serial.c` 第 5 行注释「数据包格式：FF 01 02 03 04 FE」、第 163~168 行 `Serial_SendPacket()`；课件 Slide 123 | 一致 |
| `Serial_TxPacket` / `Serial_RxPacket` 类型与长度 | `uint8_t [4]` / `uint8_t [4]` | `9-3\Serial.c` 第 5~6 行 | 一致 |
| HEX 版对外接口 | `extern uint8_t Serial_TxPacket[]; extern uint8_t Serial_RxPacket[];` + `Serial_SendPacket()` + `Serial_GetRxFlag()` | `9-3\Serial.h` 第 6~7、16~17 行 | 一致 |
| HEX 版**没有** `Serial_GetRxPacket()` | 是 | `9-3\Serial.c` / `Serial.h` 全文均无该符号 | 一致（任务提示中的函数名在官方源码中不存在） |
| `Serial_GetRxFlag()` 读一次自动清零 | 是 | `9-3\Serial.c` 第 175~183 行 | 一致 |
| HEX 状态机三状态与迁移 | `0` 等 `0xFF` → `1` 收数据（收够 4 个）→ `2` 等 `0xFE` → 回 `0` | `9-3\Serial.c` 第 193~234 行；课件 Slide 125 | 一致 |
| 状态 2 收到非 `0xFE` 时无 else 分支 | 是，状态停留不变 | `9-3\Serial.c` 第 223~230 行只有 `if (RxData == 0xFE)` | 一致 |
| `RxState` / `pRxPacket` 为 `static uint8_t` | 是 | `9-3\Serial.c` 第 195~196 行；`9-4\Serial.c` 第 166~167 行 | 一致 |
| HEX 版 `main.c`：按键 1 使 4 字节自增并发包 | 是 | `9-3\User\main.c` 第 28~42 行 | 一致 |
| HEX 版 `main.c`：收到包显示在 OLED 第 4 行 | 是 | `9-3\User\main.c` 第 44~50 行 | 一致 |
| HEX 版按键引脚 | PB1、PB11，上拉输入 | `9-3\Hardware\Key.c` 第 12、17 行 | 一致 |
| 文本包格式 | `'@'` + 字符数据 + `'\r\n'` | `9-4\Serial.c` 第 5 行注释原文「定义接收数据包数组，数据包格式"@MSG\r\n"」、第 177、186、199 行；课件 Slide 124 | 一致 |
| 文本版接收数组 | `char Serial_RxPacket[100]` | `9-4\Serial.c` 第 5 行 | 一致 |
| 文本版**没有** `Serial_TxPacket` / `Serial_SendPacket()` | 是 | `9-4\Serial.c` 全文无这两个符号 | 一致 |
| 文本版对外接口 | `extern char Serial_RxPacket[]; extern uint8_t Serial_RxFlag;`（**无 getter**） | `9-4\Serial.h` 第 6~7 行 | 一致 |
| 文本版状态 0 的 `&& Serial_RxFlag == 0` | 有 | `9-4\Serial.c` 第 177 行 | 一致 |
| 文本版在 `'\n'` 处补 `'\0'` | 有 | `9-4\Serial.c` 第 202 行 | 一致 |
| 文本版主循环手动清标志 | 有，`Serial_RxFlag = 0;` | `9-4\User\main.c` 第 48 行及其注释「否则将无法接收后续数据包」 | 一致 |
| 文本版主循环用 `strcmp` 解析指令 | `LED_ON` / `LED_OFF` / 其他 → `ERROR_COMMAND` | `9-4\User\main.c` 第 27、34、41~46 行 | 一致 |
| 文本版回传字符串 | `LED_ON_OK\r\n`、`LED_OFF_OK\r\n`、`ERROR_COMMAND\r\n` | `9-4\User\main.c` 第 30、37、43 行 | 一致 |
| 文本版 OLED 显示前先写 16 个空格清行 | 是 | `9-4\User\main.c` 第 23、31、38、44 行 | 一致 |
| 文本版 LED 引脚 | PA1（LED1）、PA2（LED2），推挽输出、默认置高 | `9-4\Hardware\LED.c` 第 16、21 行 | 一致 |
| 文本版越界风险 | 无长度检查，`pRxPacket` 可越界 | `9-4\Serial.c` 第 192~193 行无任何边界判断，数组仅 100 字节（第 5 行） | 一致（按源码事实说明） |
| 串口参数 | 9600 / 8 位 / 1 停止位 / 无校验 / 无流控，PA9=AF_PP、PA10=IPU，NVIC 分组 2、抢占 1、响应 1 | `9-3\Serial.c` 第 14~57 行、`9-4\Serial.c` 第 13~57 行，与 9-2 工程逐行相同 | 一致 |
| 串口接线 | PA9→模块 RXD、PA10→模块 TXD、GND↔GND | 官方接线图 9-1/9-2（课程作者对相同接线复用同一张图，属正常复用）；引脚定义表 PA9=`USART1_TX`、PA10=`USART1_RX` | 一致 |
| 状态机图形与课件一致性 | 三状态、迁移条件 | 课件 Slide 125 / 126 原文标注，与两版 `Serial.c` 逐条对应 | 一致 |
| 串口助手操作步骤 | 第 5 节 | 课件与源码中**均无**串口助手界面说明；协议字节来自源码 | 已标注（界面细节属操作建议） |

> [!note] 出处说明
> 本页的两份 `Serial.c`、两份 `Serial.h`、两份 `main.c` 全部**逐字照抄**课程官方配套源码 `STM32Project-有注释版\9-3 串口收发HEX数据包\` 与 `9-4 串口收发文本数据包\`（含老师中文注释），未做重写、精简或「优化」；状态机变量含义、迁移过程、边界情况均以源码逐行核对；包格式与状态机另与课程课件 Slide 122~126 交叉验证；按键 / LED 引脚取自官方 `Key.c` / `LED.c`；库函数名、枚举名、结构体成员名已在 ST 标准外设库 `stm32f10x_usart.h` 中逐个查到。
> **仍未核实**：`Serial_GetRxPacket()` 这一函数名的出处、串口助手的具体操作界面、老师现场演示与口头讲解原话、官方对「回传字符串不含 `@` 包头」「数组越界」「包尾丢失后卡在状态 2」是否另有说明——这些不在官方源码、接线图与课件文本中，本页只复述可查证的部分并保留在「待核对」。
