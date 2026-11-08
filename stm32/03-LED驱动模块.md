# 03 · LED 驱动模块（模块化编程第一课）💡

> 日期：2026-11-__ ｜ 板子：STM32F103C8T6 最小系统板（LED 接 PA1 / PA2，低电平点亮）｜ 下载器：ST-Link ｜ IDE：Keil MDK5

## 一、实验目标

把 LED 的"初始化 + 亮灭 + 翻转"封装成一个独立的 **LED.c 驱动模块**——main.c 里只管调用 `LED1_ON()` 这类函数，不用再关心寄存器细节。这就是**模块化编程**：每个硬件一个 .c/.h 文件，像搭积木一样组程序。

## 二、完整代码（含逐行注释）

```c
#include "stm32f10x.h"    // 设备头文件：库里所有函数、结构体、常量都声明在这里

/* ============ 初始化：整个模块的"开机仪式"，main 里最先调用一次 ============ */
void LED_Init(void)
{
    // ① 开时钟：给 GPIOA 通电（不写这行，后面全白干——新手第一大坑）
    RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOA, ENABLE);

    // ② 定义一张"配置单"（结构体变量），接下来往里填三个空
    GPIO_InitTypeDef GPIO_Initstructure;

    GPIO_Initstructure.GPIO_Mode  = GPIO_Mode_Out_PP;         // 模式：推挽输出
    GPIO_Initstructure.GPIO_Pin   = GPIO_Pin_1 | GPIO_Pin_2;  // 同时选中 1、2 号引脚
    GPIO_Initstructure.GPIO_Speed = GPIO_Speed_50MHz;         // 输出速度 50MHz

    // ③ 交表：库函数按配置单改写寄存器
    GPIO_Init(GPIOA, &GPIO_Initstructure);

    // ④ 上电默认：两个灯都先灭（低电平点亮的灯，置高即灭）——给系统确定的初始状态
    GPIO_SetBits(GPIOA, GPIO_Pin_1 | GPIO_Pin_2);
}

/* ============ LED1 的三个动作 ============ */

void LED1_ON(void)                    // 亮：输出低电平
{
    GPIO_ResetBits(GPIOA, GPIO_Pin_1);
}

void LED1_OFF(void)                   // 灭：输出高电平
{
    GPIO_SetBits(GPIOA, GPIO_Pin_1);
}

void LED1_Turn(void)                  // 翻转：亮的变灭、灭的变亮
{
    // 读当前引脚电平：0（亮）→ 置高拍灭；1（灭）→ 置低按亮
    if (GPIO_ReadInputDataBit(GPIOA, GPIO_Pin_1) == 0)
    {
        GPIO_SetBits(GPIOA, GPIO_Pin_1);
    }
    else
    {
        GPIO_ResetBits(GPIOA, GPIO_Pin_1);
    }
}

/* ============ LED2 的三个动作（同 LED1，引脚换 Pin_2） ============ */

void LED2_ON(void)  { GPIO_ResetBits(GPIOA, GPIO_Pin_2); }
void LED2_OFF(void) { GPIO_SetBits(GPIOA, GPIO_Pin_2);   }

void LED2_Turn(void)
{
    if (GPIO_ReadInputDataBit(GPIOA, GPIO_Pin_2) == 0)
        GPIO_SetBits(GPIOA, GPIO_Pin_2);
    else
        GPIO_ResetBits(GPIOA, GPIO_Pin_2);
}
```

## 三、三个值得多想一秒的点

1. **为什么 Init 结尾要 SetBits（先灭灯）？**
   芯片上电后引脚是悬空状态，灯可能乱亮。初始化最后统一置高，保证上电第一刻就是确定的灭——"给系统确定的初始状态"是工程师的基本功。

2. **输出引脚居然也能"读"？**
   `GPIO_ReadInputDataBit` 名字带 Input，却用在输出引脚上——这不算 bug：芯片内部的输入寄存器（IDR）会**如实记录引脚当前实际电平**，不管这电平是谁驱动的。"先读当前状态 → 反着写"= 翻转。这个思路以后在按键消抖、状态机里到处都是。

3. **`GPIO_Pin_1 | GPIO_Pin_2` 的 `|`——位运算日常上岗**
   `Pin_1` = 0x0002，`Pin_2` = 0x0004，一"或" = 0x0006（两位同时选中）。**想同时操作几个引脚，就用 `|` 拼起来**。

## 四、进阶彩蛋：一行版翻转（学完 BitAction 后）

```c
GPIO_WriteBit(GPIOA, GPIO_Pin_1, (BitAction)!GPIO_ReadInputDataBit(GPIOA, GPIO_Pin_1));
// 读当前电平 → ! 取反 → 强转 BitAction → 写入。与上面的 if/else 完全等价
```

## 五、踩坑记录

- ________（写你的真实经历：比如忘了在 main 里调 LED_Init 导致灯不亮、.h 文件忘了 include 函数报未声明等）

## 六、收获

1. 第一次把代码按功能拆成独立模块（LED.c）——main.c 从此只写"业务逻辑"
2. 位运算（`|` 选多个引脚）成了日常工具
3. 理解了"读-判断-写"的状态翻转模式

---
*相关笔记：[01 · 点灯实验](01-点灯.md) · [02 · 流水灯与蜂鸣器](02-流水灯蜂鸣器.md)*
