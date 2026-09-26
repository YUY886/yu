---
course: 江协科技 STM32入门教程-2023版
chapter: 04-2 OLED显示屏
type: 分集笔记
source: https://www.bilibili.com/video/BV1th411z7sn?p=10
tags:
  - STM32
  - 江协科技
  - OLED
  - SSD1306
  - 显示函数
status: draft
verify: 官方源码+接线图+课件
updated: 2026-09-26
---

# 04-2 OLED 显示屏

> [!abstract] 这一集只解决一个问题
> **屏已经亮了，怎么往上写字符、数字和字符串？**
> 答：`OLED.c` 一共给了 **8 个显示函数**，全部以「**第几行、第几列**」开头。屏幕被切成 **4 行 × 16 列** 的方格，每个方格放一个字符。
>
> 上一集把屏点亮了，这一集把 API 用熟——**后面每一章的调试都要靠它**。
>
> 上一集 [[江协STM32 04-1 OLED调试工具]] ｜ 下一章 [[江协STM32 05 EXTI外部中断（章索引）]] ｜ 章索引 [[江协STM32 04 OLED（章索引）]]

## 1 OLED 驱动 API 全表

课程配套 `OLED.h` 里的**全部 8 个函数**，一个不多一个不少：

| #   | 函数原型                                                                                    | 作用                              | 参数              | 典型调用                                 |
| --- | --------------------------------------------------------------------------------------- | ------------------------------- | --------------- | ------------------------------------ |
| 1   | `void OLED_Init(void)`                                                                  | 初始化（引脚 → I2C → SSD1306 命令 → 清屏） | 无               | `OLED_Init();`                       |
| 2   | `void OLED_Clear(void)`                                                                 | 清屏（全屏写 `0x00`）                  | 无               | `OLED_Clear();`                      |
| 3   | `void OLED_ShowChar(uint8_t Line, uint8_t Column, char Char)`                           | 显示 **1 个 ASCII 字符**             | 行 1~4、列 1~16、字符 | `OLED_ShowChar(1, 1, 'A');`          |
| 4   | `void OLED_ShowString(uint8_t Line, uint8_t Column, char *String)`                      | 显示**字符串**                       | 行、起始列、字符串       | `OLED_ShowString(1, 3, "Hello");`    |
| 5   | `void OLED_ShowNum(uint8_t Line, uint8_t Column, uint32_t Number, uint8_t Length)`      | 显示**无符号十进制**                    | 行、列、数、**位数**    | `OLED_ShowNum(2, 1, 12345, 5);`      |
| 6   | `void OLED_ShowSignedNum(uint8_t Line, uint8_t Column, int32_t Number, uint8_t Length)` | 显示**有符号十进制**                    | 行、列、数、位数        | `OLED_ShowSignedNum(2, 7, -66, 2);`  |
| 7   | `void OLED_ShowHexNum(uint8_t Line, uint8_t Column, uint32_t Number, uint8_t Length)`   | 显示**十六进制**                      | 行、列、数、位数        | `OLED_ShowHexNum(3, 1, 0xAA55, 4);`  |
| 8   | `void OLED_ShowBinNum(uint8_t Line, uint8_t Column, uint32_t Number, uint8_t Length)`   | 显示**二进制**                       | 行、列、数、位数        | `OLED_ShowBinNum(4, 1, 0xAA55, 16);` |

### 1.1 每个函数取值范围的官方注释

源码注释里写清了取值范围，这是**最权威的参数说明**：

| 函数 | 参数范围（摘自源码注释） |
| --- | --- |
| `OLED_ShowChar` | `Line` 1~4；`Column` 1~16；`Char` **ASCII 可见字符** |
| `OLED_ShowString` | `Line` 1~4；`Column` 1~16（起始列）；`String` ASCII 可见字符 |
| `OLED_ShowNum` | 数字 **0 ~ 4294967295**（`uint32_t`）；`Length` **1~10** |
| `OLED_ShowSignedNum` | 数字 **−2147483648 ~ 2147483647**（`int32_t`）；`Length` **1~10** |
| `OLED_ShowHexNum` | 数字 **0 ~ 0xFFFFFFFF**；`Length` **1~8** |
| `OLED_ShowBinNum` | 数字 **0 ~ 1111 1111 1111 1111**（16 位）；`Length` **1~16** |

### 1.2 `Length` 到底是什么意思

`Length` 是**要占用的字符格数**，不是"数字本身的值"，也不是"数字的位数上限"。

| 调用 | 屏幕上的结果 | 说明 |
| --- | --- | --- |
| `OLED_ShowNum(1, 1, 12345, 5)` | `12345` | 正好 5 位 |
| `OLED_ShowNum(1, 1, 7, 5)` | `00007` | **不足左侧补 `0`** |
| `OLED_ShowNum(1, 1, 123456, 5)` | `12345` | **超出被截断**（只显示前 5 个字符） |

`OLED_ShowSignedNum()` 会**额外占用 1 列**来放符号（正数显示 `+`，负数显示 `−`）：

```text
OLED_ShowSignedNum(2, 1, 66, 2);   →  +66     占 1（符号）+ 2（数字）= 3 列
OLED_ShowSignedNum(2, 1, -66, 2);  →  -66     同上
```

> [!warning] 千万不要自己给数字前面写 `0`
> 想让数字左补零，靠 `Length` 参数就够。手写 `0123` 是 **C 语言的八进制字面量**，值等于十进制的 **83**，不是 123。
> 同理 `0x` 才是十六进制前缀。

## 2 坐标：4 行 × 16 列

### 2.1 行列约定

**行和列都从 1 开始数**（不是 0），范围分别是 **1~4** 和 **1~16**：

```text
列  1  2  3  4  5  6  7  8  9 10 11 12 13 14 15 16
行 ┌──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┐
 1 │  │  │  │  │  │  │  │  │  │  │  │  │  │  │  │  │   ← Line = 1
   ├──┼──┼──┼──┼──┼──┼──┼──┼──┼──┼──┼──┼──┼──┼──┼──┤
 2 │  │  │  │  │  │  │  │  │  │  │  │  │  │  │  │  │   ← Line = 2
   ├──┼──┼──┼──┼──┼──┼──┼──┼──┼──┼──┼──┼──┼──┼──┼──┤
 3 │  │  │  │  │  │  │  │  │  │  │  │  │  │  │  │  │   ← Line = 3
   ├──┼──┼──┼──┼──┼──┼──┼──┼──┼──┼──┼──┼──┼──┼──┼──┤
 4 │  │  │  │  │  │  │  │  │  │  │  │  │  │  │  │  │   ← Line = 4
   └──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┘
     ↑
   Column = 1                                    Column = 16
```

一个方格 = **8 像素宽 × 16 像素高**，正好是字库 `OLED_F8x16` 里一个 ASCII 字符的点阵尺寸。

## 3 显示原理：页地址与列地址

### 3.1 SSD1306 内部结构

屏上其实有两套坐标系，必须分清：

| | 像素坐标 | 页/列坐标（驱动内部） |
| --- | --- | --- |
| 横向 | 0 ~ 127（共 128 列） | **列地址 X：0 ~ 127** |
| 纵向 | 0 ~ 63（共 64 行） | **页地址 Y：0 ~ 7**，每页 8 像素高 |

```text
        X: 0 ─────────────────────────► 127
      ┌───────────────────────────────────┐
 Y=0  │  Page 0  （像素行 0 ~ 7）          │
 Y=1  │  Page 1  （像素行 8 ~ 15）         │   每个页 = 8 个像素行
 Y=2  │  Page 2                           │   8 页 × 8 行 = 64 行
 Y=3  │  Page 3                           │
 Y=4  │  Page 4                           │
 Y=5  │  Page 5                           │
 Y=6  │  Page 6                           │
 Y=7  │  Page 7  （像素行 56 ~ 63）        │
      └───────────────────────────────────┘
```

**关键换算**：`OLED_ShowChar()` 里第 1~4 行、每行 16 像素高，所以**一个显示行 = 2 个页**：

```text
Line 1  →  页 0、页 1
Line 2  →  页 2、页 3
Line 3  →  页 4、页 5
Line 4  →  页 6、页 7
```

公式：**上半部分页号 = `(Line - 1) * 2`，下半部分页号 = `(Line - 1) * 2 + 1`**。

### 3.2 设置光标位置

写像素之前，要先告诉 SSD1306"接下来往哪写"。这个动作由 `OLED_SetCursor()` 完成：

```c
/**
  * @brief  OLED设置光标位置
  * @param  Y 以左上角为原点，向下方向的坐标，范围：0~7
  * @param  X 以左上角为原点，向右方向的坐标，范围：0~127
  */
void OLED_SetCursor(uint8_t Y, uint8_t X)
{
	OLED_WriteCommand(0xB0 | Y);					//设置Y位置（页地址）
	OLED_WriteCommand(0x10 | ((X & 0xF0) >> 4));	//设置X位置高4位
	OLED_WriteCommand(0x00 | (X & 0x0F));			//设置X位置低4位
}
```

三条命令的含义：

| 命令 | 作用 | 位分配 |
| --- | --- | --- |
| `0xB0` 按位或 `Y` | 设置**页地址** | 低 3 位放页号（0~7），高 5 位固定 |
| `0x10` 按位或列地址高 4 位 | 设置**列地址高 4 位** | 列地址被拆成两个半字节分两条命令发 |
| `0x00` 按位或列地址低 4 位 | 设置**列地址低 4 位** | 同上 |

> [!note] `OLED_SetCursor` 是"内部函数"
> 它**没有**出现在 `OLED.h` 里，属于驱动内部使用的函数（官方 `OLED.h` 只声明了 8 个显示函数）。日常写代码只需要用第 1 节表格里的 8 个 API，不用直接调它。理解它的意义在于看懂 `OLED_ShowChar()` 是怎么定位的。

### 3.3 一个字符是怎么画出来的

这是整集最核心的一段代码——**字符被拆成上下两半，分两次写入**：

```c
/**
  * @brief  OLED显示一个字符
  * @param  Line 行位置，范围：1~4
  * @param  Column 列位置，范围：1~16
  * @param  Char 要显示的一个字符，范围：ASCII可见字符
  */
void OLED_ShowChar(uint8_t Line, uint8_t Column, char Char)
{
	uint8_t i;
	OLED_SetCursor((Line - 1) * 2, (Column - 1) * 8);		//设置光标位置在上半部分
	for (i = 0; i < 8; i++)
	{
		OLED_WriteData(OLED_F8x16[Char - ' '][i]);			//显示上半部分内容
	}
	OLED_SetCursor((Line - 1) * 2 + 1, (Column - 1) * 8);	//设置光标位置在下半部分
	for (i = 0; i < 8; i++)
	{
		OLED_WriteData(OLED_F8x16[Char - ' '][i + 8]);		//显示下半部分内容
	}
}
```

逐句拆解：

| 代码 | 含义 |
| --- | --- |
| `(Column - 1) * 8` | 第 `Column` 列的**像素起点**（每列字符宽 8 像素） |
| `(Line - 1) * 2` | 第 `Line` 行的**上半页页号** |
| `(Line - 1) * 2 + 1` | 第 `Line` 行的**下半页页号** |
| `OLED_F8x16[Char - ' ']` | **字模寻址**：字库从 ASCII 空格（0x20）开始存，所以要减去 `' '` |
| `[i]`（`i` 为 0~7） | 取该字符的**上半部分** 8 个字节（上半页 8 个像素行） |
| `[i + 8]`（`i` 为 0~7） | 取该字符的**下半部分** 8 个字节 |

**字库结构**：`OLED_Font.h` 里的 `OLED_F8x16[][16]`，**每个字符占 16 个字节**，前 8 字节是上半部分、后 8 字节是下半部分。文件开头的注释也写明了：

```c
/*OLED字模库，宽8像素，高16像素*/
const uint8_t OLED_F8x16[][16]=
{
	0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00,
	0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00,//  0    ← 空格（第一个）

	0x00,0x00,0x00,0xF8,0x00,0x00,0x00,0x00,
	0x00,0x00,0x00,0x33,0x30,0x00,0x00,0x00,//! 1
	/* ... */
};
```

### 3.4 字符串就是"一个字符接一个字符"

```c
/**
  * @brief  OLED显示字符串
  * @param  Line 起始行位置，范围：1~4
  * @param  Column 起始列位置，范围：1~16
  * @param  String 要显示的字符串，范围：ASCII可见字符
  */
void OLED_ShowString(uint8_t Line, uint8_t Column, char *String)
{
	uint8_t i;
	for (i = 0; String[i] != '\0'; i++)     // 一直显示到字符串结束符 '\0'
	{
		OLED_ShowChar(Line, Column + i, String[i]);   // 每显示一个就右移一列
	}
}
```

`Column + i` 是**列号递增**，所以字符串从 `Column` 开始向右排。

> [!warning] 字符串越界不会报错，只会"跑到屏幕外"
> 从第 `Column` 列写 `n` 个字符，需要 `Column + n - 1 ≤ 16`。
> 例如 `OLED_ShowString(1, 10, "HelloWorld")`（10 个字符）会需要列 10~19，**第 17 列之后超出屏幕**——不报错，但后面的字符看不见。

### 3.5 数字显示是怎么算出来的

数字显示的思路：**逐位取出十进制数字，再转成对应字符**。

```c
/* 求 X 的 Y 次方，用于"取第几位" */
uint32_t OLED_Pow(uint32_t X, uint32_t Y)
{
	uint32_t Result = 1;
	while (Y--)
	{
		Result *= X;
	}
	return Result;
}

void OLED_ShowNum(uint8_t Line, uint8_t Column, uint32_t Number, uint8_t Length)
{
	uint8_t i;
	for (i = 0; i < Length; i++)
	{
		OLED_ShowChar(Line, Column + i, Number / OLED_Pow(10, Length - i - 1) % 10 + '0');
	}
}
```

核心那一行的意思是：

```text
Number / 10^(Length-i-1) % 10 + '0'
   │           │           │      └─ 数字 0~9 → 字符 '0'~'9'（'0' 的 ASCII 是 0x30）
   │           │           └─ 只留个位，得到 0~9
   │           └─ 把这一位挪到个位
   └─ 要显示的数值
```

`+ '0'` 是**数字转字符**的标准写法；反过来 `Char - '0'` 就是字符转数字。

十六进制版多了一步"10 以上用字母"：

```c
SingleNumber = Number / OLED_Pow(16, Length - i - 1) % 16;
if (SingleNumber < 10)
	OLED_ShowChar(Line, Column + i, SingleNumber + '0');        // 0~9
else
	OLED_ShowChar(Line, Column + i, SingleNumber - 10 + 'A');   // 10~15 → A~F
```

> [!tip] 一句话总结四个数字函数
> 全是同一个套路：**按进制逐位取出 → 转成字符 → 调 `OLED_ShowChar()` 摆到 `Column + i` 列**。
> 十进制用 `Pow(10, …) % 10`，十六进制用 `Pow(16, …) % 16`，二进制用 `Pow(2, …) % 2`。

## 4 关于「显存缓冲区」和「取模软件」

> [!warning] 本课程这一集**没有** `OLED_GRAM` / `OLED_Update`
> 有些教程（以及江协科技后来单独发布的**新版** 0.96 寸 OLED 驱动库）使用**显存缓冲区 + 统一刷新**的架构，形如：
>
> ```c
> uint8_t OLED_GRAM[8][128];   // 8 页 × 128 列，每字节 8 个像素
> /* 所有显示函数先改 GRAM，最后调用一次 OLED_Update() 才真正写屏 */
> OLED_Update();
> ```
>
> 这种设计的优点是**画面不撕裂、支持局部刷新**。
>
> 但**本课程（2023 版）这一集用的驱动是"直写"模式**：`OLED_ShowChar()` 内部直接 `OLED_WriteData()`，数据**立刻通过 I2C 发到屏上**，既没有 `OLED_GRAM`，也**没有 `OLED_Update()` 函数**。
>
> 两种写法不要混用：在课程的驱动里调 `OLED_Update()` 会**编译报错**（函数不存在）。

### 4.1 课程驱动 vs 江协科技新版驱动

这里容易踩坑，明确区分一下：

| | **本课程驱动**（2023 版教程配套） | **江协科技新版驱动**（官网单独发布） |
| --- | --- | --- |
| 显示方式 | 直写屏幕（无缓冲区） | **显存缓冲区 + `OLED_Update()`** |
| 坐标 | 行列 1~16 / 1~4，`uint8_t` | 像素坐标，新版改为 `int16_t`，**支持负坐标平滑移入移出** |
| 字号 | 固定 8×16 | 可选（如 `OLED_8X16`、`OLED_6X8`） |
| 汉字 | ❌ 无 `OLED_ShowChinese` | ✅ 有（V2.0 起移除，改为 `OLED_ShowString` 中英文混写） |
| 图片 | ❌ 无 `OLED_ShowImage` | ✅ 有 `OLED_ShowImage(x, y, w, h, 数组名)` |
| 浮点数 | ❌ 无 | ✅ 有 `OLED_ShowFloatNum` |
| 中英混排 | 只支持 ASCII | V2.0 起 `OLED_ShowString` / `OLED_Printf` 支持 |
| 字符集 | 固定 | `OLED_CHARSET_UTF8` / `OLED_CHARSET_GB2312` 可选 |

> [!warning] 对照表的依据范围
> **左列「本课程驱动」**已核对本机官方源码：`4-1 OLED显示屏\Hardware\OLED.h` 恰好 8 条声明，`OLED.c` 中**不存在** `OLED_ShowChinese`、`OLED_ShowImage`、`OLED_Printf`、`OLED_GRAM`、`OLED_Update`。
> **右列「新版驱动」**的依据是江协科技官方驱动页 <https://jiangxiekeji.com/tutorial/oled.html> 的「适用器件 / 程序亮点 / 更新动态」，**本机没有新版驱动的代码**，因此右列的行（除已在该页明确写出的"汉字、图片、绘图""V1.2 坐标改 int16_t""V2.0 删除 OLED_ShowChinese""字符集宏"外）属于**未在官方资料中核实**，请以官方页面与新版代码为准。

**新版驱动的版本演进**（摘自官方页面）：

- `V1.0`（2023.11.22）：首次发布
- `V1.1`（2023.12.8）：`OLED_Init` 后加入清屏，防花屏；修复 `OLED_ShowFloatNum` 浮点 Bug
- `V1.2`（2024.4.24）：坐标类型 `uint8_t` → `int16_t`，支持负坐标实现**平滑移入移出**
- `V2.0`（2024.10.20）：**删除 `OLED_ShowChinese`**；`OLED_ShowString`/`OLED_Printf` 支持中英文混写；`OLED_Data.h` 增加字符集宏

### 4.2 取模软件（点阵字模提取）

要显示字库里**没有**的内容（汉字、图片、大号字符），就得用取模软件生成点阵数组。常用工具是 **PCtoLCD2002**。

**通用流程**：

1. 在软件里输入汉字/导入 BMP 图片
2. 设置**点阵大小**（如 16×16 汉字、8×16 字符、32×32 图片）
3. 设置取模**格式参数**
4. 生成 C 语言数组，复制进工程

**几个关键选项的含义**：

| 选项 | 取值 | 说明 |
| --- | --- | --- |
| **阴码 / 阳码** | 阴码 / 阳码 | 阴码：**亮点为 1**（本课程驱动需要这种）；阳码：亮点为 0 |
| **逐行式 / 逐列式** | — | 取模的扫描方向。SSD1306 按**页**组织数据（一页内 8 个像素纵向排列），驱动代码的取字节顺序决定了该选哪种 |
| **逆向 / 顺向** | — | 低位在前还是高位在前 |
| **输出格式** | C51 格式 | 生成 `{0x00, 0x01, …}` 这种可直接粘进 C 文件的数组 |

> [!warning] 取模设置必须和驱动的取字节方式对齐
> 如果取模方向或高低位设反了，字模数组**不会报错**，只会在屏幕上显示成**乱码、镜像或上下颠倒**。
> 判断方法：先用一个已知的字（如 `A`）对照 `OLED_Font.h` 里已有的点阵，**同样的设置**才是对的。

## 5 完整示例：在指定位置显示字符串和数字

```c
#include "stm32f10x.h"      // Device header
#include "OLED.h"           // OLED 驱动头文件

int main(void)
{
	OLED_Init();                                    // 初始化 OLED

	/* ---- 第 1 行：字符串 ---- */
	OLED_ShowString(1, 1, "Hello");                 // 从第 1 行第 1 列开始显示 "Hello"（占列 1~5）
	OLED_ShowString(1, 7, "World");                 // 从第 1 行第 7 列开始显示 "World"（占列 7~11）

	/* ---- 第 2 行：数字 ---- */
	OLED_ShowString(2, 1, "Num:");                  // 第 2 行第 1~4 列显示提示文字
	OLED_ShowNum(2, 6, 12345, 5);                   // 第 2 行第 6 列起，占 5 格 → 显示 12345

	/* ---- 第 3 行：有符号数 ---- */
	OLED_ShowString(3, 1, "Temp:");                 // 第 3 行提示
	OLED_ShowSignedNum(3, 7, -66, 2);               // 第 3 行第 7 列起 → -66（符号占 1 列 + 2 位数字）

	/* ---- 第 4 行：十六进制 / 二进制 ---- */
	OLED_ShowString(4, 1, "Hex:");                  // 第 4 行提示
	OLED_ShowHexNum(4, 6, 0xAA55, 4);               // 第 4 行第 6 列起 → AA55

	while (1)
	{
		/* 静态内容写一次就够，显示会一直保持到被覆盖或清屏 */
	}
}
```

屏幕效果：

```text
列  1  2  3  4  5  6  7  8  9 10 11 12 13 14 15 16
 1  H  e  l  l  o        W  o  r  l  d
 2  N  u  m  :        1  2  3  4  5
 3  T  e  m  p  :           -  6  6
 4  H  e  x  :        A  A  5  5
```

### 5.1 动态刷新（后面每一章的通用写法）

需要显示的变量**一直变化**时，把显示语句放进 `while(1)` 循环里即可：

```c
// 以第 6 章的定时中断为例
while (1)
{
	OLED_ShowNum(1, 5, Num, 5);                    // Num 变了就重画一次
	OLED_ShowNum(2, 5, TIM_GetCounter(TIM2), 5);   // 实时数值，肉眼看到它在飞转
}
```

> [!tip] 显示变量时长度要写够
> 用**固定长度**（如 `5`）显示变动的数字，可以让高位补 `0` 占位，避免数字位数变化时"残留旧字符"。
> `OLED_ShowNum(1, 5, Num, 5)` 显示 `00007` → `00008`，位置稳定；若写 `Length = 1`，从 `9` 变到 `10` 时后面的字符会留在屏上。

## 6 易错点

- [ ] **行列从 0 开始数** → 实际是 **1~4 / 1~16**，写 `0` 会越界。
- [ ] `OLED_ShowChar(1, 1, "A")` 用了**双引号** → 应为单引号 `'A'`（双引号是字符串，类型不匹配）。
- [ ] 把 `Length` 当成数字的值 → 它是**占用的格数**，`OLED_ShowNum(1,1,7,5)` 显示 `00007`。
- [ ] 手写 `OLED_ShowNum(1, 1, 0123, 4)` → `0123` 是**八进制**（= 十进制 83）。
- [ ] 字符串太长**写出屏幕右边界**（`Column + 字符数 − 1 > 16`）→ 不报错，超出的看不见。
- [ ] 在有符号数上少算一列 → `OLED_ShowSignedNum` **自带 `+`/`-` 号**，会比 `Length` 多占 1 列。
- [ ] 显示变动的数字时 `Length` 写太小 → 位数变多时屏幕残留旧字符。
- [ ] 上电后没有 `OLED_Init()` 就直接显示 → 命令序列未下发，黑屏或花屏。
- [ ] 在**课程驱动**里调用 `OLED_Update()` 或 `OLED_ShowChinese()` → 函数不存在，编译报错（那是江协科技**新版**驱动的 API）。
- [ ] 取模软件的**阴码/阳码、逐行/逐列**与驱动取字节方式不匹配 → 显示乱码或上下颠倒，且不报错。
- [ ] 清屏用 `OLED_Clear()`；想**只覆盖一部分**内容就直接重写那一块，不必每次全屏清（全屏清会造成明显闪烁）。

## 7 自测

1. `OLED_ShowNum(2, 3, 42, 5)` 会在屏幕的哪几列显示什么内容？
2. `OLED_ShowSignedNum(3, 1, -66, 2)` 一共占用几列？为什么？
3. `OLED_ShowChar()` 为什么要**分两次**调用 `OLED_SetCursor()`？
4. 字模寻址写的是 `OLED_F8x16[Char - ' '][i]`，为什么要 `- ' '`？`[i]` 和 `[i + 8]` 分别对应字符的哪部分？
5. 本课程这一集的驱动里能调用 `OLED_Update()` 吗？为什么？

> [!success]- 参考答案
> 1. 第 2 行**第 3 列到第 7 列**，显示 **`00042`**。因为 `Length = 5` 表示占 5 格，不足左侧补 `0`。
> 2. 共 **3 列**。因为 `-66` 需要 1 列放符号 `-`，再加 `Length = 2` 位数字，共 `1 + 2 = 3` 列（占第 1~3 列）。
> 3. 因为一个字符的点阵是 **8 像素宽 × 16 像素高**，而 SSD1306 是按**页**（每页 8 像素高）写入的。所以先定位到**上半页**写前 8 个字节，再定位到**下半页**写后 8 个字节。
> 4. 因为字库 `OLED_F8x16` **从 ASCII 空格（`0x20`）开始存放**，数组下标 0 对应空格，所以要减去 `' '`（即 `0x20`）才能得到正确下标。`[i]`（`i` 为 0~7）是字符的**上半部分**，`[i + 8]` 是**下半部分**。
> 5. **不能**。本课程这一集的驱动是**直写模式**：`OLED_ShowChar()` 内部直接调 `OLED_WriteData()` 把数据发到屏上，**没有显存缓冲区，也没有定义 `OLED_Update()`**，调用会编译报错。`OLED_GRAM` + `OLED_Update()` 是江协科技后来单独发布的**新版** OLED 驱动的架构。

## 8 待核对

- [ ] **取模软件的具体设置组合**（阴码/阳码、逐行式/逐列式、C51 格式的勾选项截图）——官方课件、源码与接线图中均无取模设置说明，需逐帧看视频确认。
- [ ] 本集视频中 `OLED_ShowChar` / `OLED_ShowNum` 内部实现的讲解详略程度，以及是否现场演示"改字库表"。
- [ ] 视频里实际演示的例程内容（显示哪些字符串/变量）与本笔记示例可能有出入。
- [ ] 第 4.1 节对照表**右列「江协科技新版驱动」**的行——新版驱动代码不在本机官方资料中，仅依据官方驱动页描述整理，详见 4.1 的说明框。

## 核对记录（2026-09-26）

> [!success] 核对依据
> 实际用到的依据（均为本机官方资料）：
> - **官方源码**：`程序源码\STM32Project-有注释版\4-1 OLED显示屏\Hardware\OLED.c`、`OLED.h`、`OLED_Font.h`；`1-4 OLED驱动函数模块\4针脚I2C版本\OLED.h`、`OLED.c`
> - **官方接线图**：`ground-truth\接线图\4-1 OLED显示屏.png`（含"OLED 下方被遮住的接线图"小图）
> - **官方课件**：`ground-truth\课件文本.md`（Slide 38 OLED 简介、Slide 39 硬件电路、Slide 40 驱动函数）
> - **引脚定义表**：`ground-truth\F103C8T6引脚定义_缩略.png`
> - **官方驱动页**：<https://jiangxiekeji.com/tutorial/oled.html>

| 核对项 | 笔记原值 | 官方依据 | 结论 |
| --- | --- | --- | --- |
| 课程驱动函数总数 | 8 个 | 官方 `OLED.h` 全文 13 行，恰好 8 条函数声明 | 一致 |
| 8 条函数名与完整原型 | Init/Clear/ShowChar/ShowString/ShowNum/ShowSignedNum/ShowHexNum/ShowBinNum 及参数类型 | `OLED.h` L4~L11 逐字相同 | 一致 |
| `OLED_ShowChar` 参数范围 | Line 1~4、Column 1~16、ASCII 可见字符 | `OLED.c` L129~L131 函数注释原文 | 一致 |
| `OLED_ShowString` 参数范围 | Line 1~4、Column 1~16、ASCII 可见字符 | `OLED.c` L151~L153 函数注释原文 | 一致 |
| `OLED_ShowNum` 数值范围 / `Length` | 0~4294967295 / 1~10 | `OLED.c` L183~L184 函数注释原文 | 一致 |
| `OLED_ShowSignedNum` 数值范围 / `Length` | −2147483648~2147483647 / 1~10 | `OLED.c` L200~L201 函数注释原文 | 一致 |
| `OLED_ShowHexNum` 数值范围 / `Length` | 0~0xFFFFFFFF / 1~8 | `OLED.c` L228~L229 函数注释原文 | 一致 |
| `OLED_ShowBinNum` 数值范围 / `Length` | 0~1111 1111 1111 1111 / 1~16 | `OLED.c` L253~L254 函数注释原文 | 一致 |
| 是否存在 `OLED_Printf` | 不存在（原记为"待核对"） | `4-1` 与 `1-4` 两个版本的 `OLED.h`、`OLED.c` 全文均无 | 一致（该项已核实，从待核对移除） |
| 是否存在 `OLED_ShowChinese` / `OLED_ShowImage` | 不存在（原记为"待核对"） | 同上，官方源码中均无 | 一致（该项已核实，从待核对移除） |
| 是否存在显存缓冲区 `OLED_GRAM` / `OLED_Update` | 无，本课程为"直写"模式 | `OLED.c` 全文：`OLED_ShowChar` 内直接 `OLED_WriteData`，无任何缓冲区与刷新函数 | 一致 |
| `OLED_ShowChar` 实现（分上下半页两次写） | 与笔记代码块一致 | `OLED.c` L134~L147 原文 | 一致 |
| `OLED_ShowString` 实现（`Column + i`） | 与笔记代码块一致 | `OLED.c` L156~L163 原文 | 一致 |
| `OLED_ShowNum` 实现（`OLED_Pow(10,…)%10+'0'`） | 与笔记代码块一致 | `OLED.c` L169~L194 原文 | 一致 |
| `OLED_ShowHexNum` 分支 | `< 10` 用 `+'0'`，否则 `-10+'A'` | `OLED.c` L237~L245 原文 | 一致 |
| `OLED_SetCursor` 三条命令与位分配 | `0xB0\|Y`、`0x10\|高4位`、`0x00\|低4位` | `OLED.c` L102~L107 原文 | 一致 |
| `OLED_SetCursor` 未出现在 `OLED.h` | 是"内部函数" | 官方 `OLED.h` 中确无此声明（补充："官方 `OLED.h` 只声明了 8 个显示函数"） | 一致 |
| 字库名、几何与下标的偏移 | `OLED_F8x16[][16]`，8×16，`Char - ' '` | `OLED_Font.h` L4~L5 注释与声明；`OLED.c` L140、L145 的 `OLED_F8x16[Char - ' '][i]` | 一致 |
| `Length` 的含义与补零行为 | 占格数，不足左侧补 `0` | `OLED.c` L192 用 `OLED_Pow(10, Length-i-1)` 逐位取数 → 不足即取到 0 | 一致 |
| 有符号数多占 1 列 | 符号占 `Column`，数字从 `Column + 1` 起 | `OLED.c` L210~L221：先 `OLED_ShowChar(Line, Column, '+')`，再循环 `Column + i + 1` | 一致 |
| 一行 = 2 页的换算 | `(Line-1)*2` 与 `+1` | `OLED.c` L137、L142 | 一致 |
| 每列字符宽 8 像素 | `(Column - 1) * 8` | `OLED.c` L137、L142 | 一致 |
| 字符串越界不报错（`Column + n - 1 ≤ 16`） | 超出屏幕的部分看不见 | `OLED.c` L159~L162 循环不设边界检查 | 一致 |
| 4 行 × 16 列坐标约定与课件示意 | 1~4 / 1~16，从 1 开始 | 课件 Slide 40 左侧函数表与右侧屏幕示意（列 1~16、行 1~4） | 一致 |
| 驱动默认引脚（本页未列出，属相邻笔记内容） | 本页未写引脚号 | `4-1 OLED显示屏\Hardware\OLED.c` L5~L6：SCL=PB8、SDA=PB9 | 一致（本页无需改动） |
| 驱动是"直写"而非"带显存缓冲区 GRAM" | 直写 | `OLED.c` 全文无 GRAM/Update，见上 | 一致 |
| 新版驱动版本演进（V1.0~V2.0） | V1.0 2023.11.22 / V1.1 2023.12.8 / V1.2 2024.4.24 / V2.0 2024.10.20 | 官方驱动页「更新动态」逐条一致，**但本机无新版驱动代码** | 一致（依据为官方网页，已在 4.1 加说明框） |

> [!note] 出处说明
> 本页全部函数原型、参数范围、实现代码与字库结构，已逐行对照**官方配套源码** `程序源码\STM32Project-有注释版\4-1 OLED显示屏\Hardware\OLED.h`、`OLED.c`、`OLED_Font.h`（8 条声明的原文、`OLED_SetCursor` 的三条命令、`OLED_ShowChar/ShowString/ShowNum/ShowSignedNum/ShowHexNum/ShowBinNum` 实现、`OLED_F8x16[][16]` 字库结构与 `Char - ' '` 寻址）；「8 个函数」的结论同时与 `1-4 OLED驱动函数模块\4针脚I2C版本\OLED.h` 一致；4 行 × 16 列坐标约定对照**官方课件** Slide 40；「本课程驱动为直写、无 `OLED_GRAM`/`OLED_Update`」由上述 `OLED.c` 全文核实（除 `OLED_WriteData` 写屏外无任何缓冲区）。
> **仍未核实**：取模软件（PCtoLCD2002）的阴码/阳码、逐行/逐列、C51 格式等具体设置（官方资料中未见）；本集视频的讲解详略与实际演示例程；第 4.1 节对照表中「新版驱动」列的多项 API 细节（新版代码不在本机资料中，仅据官方驱动页 <https://jiangxiekeji.com/tutorial/oled.html> 整理）。**未逐帧核对视频画面**，若与视频有出入以视频为准。
