---
layout: post
title: "STM32-使用STM32CubeIDE构建一个基本的寄存器开发环境"
categories: STM32
tags: 单片机
author: fengsh998
typora-root-url: ..
---

前言：
现在的CubeIDE直接创建的STM32工程都是直接上Hal库了的，有时候想学习寄存器的环境下编写代码，所以就有了以下的文章

# 使用CubeIDE来构建一个寄存器的编程环境

**CubeIDE版本：**

Version: 1.19.0

Build: 25607_20250703_0907 (UTC)

**<font color="red">项目工程以STM32F103C8T6为例说明！</font>**

正常使用CubeIDE生成的STM32Project工程时，会自动帮我生成Hal库的工程目录结构，因此想要学习使用寄存器进行开发的，在HAL库环境上混着用的话很容易调用错。因此改造一下HAL的STM32工程。步聚：（新建STM32project的过程不再复述了）

* **1、删除Hal库文件**

删除hal库相关的文件: stm32f1xx_hal_conf.h，stm32f1xx_it.h, stm32f1xx_hal_map.c, stm32f1xx_it.c, STM32F1xx_HAL_Driver,及projectname.ioc

![img](/assets/articles/ic/stm32/cuberegister/del_env.jpg)

删除未引用的头文件目录：

![img](/assets/articles/ic/stm32/cuberegister/del_inc.jpg)

删除完后编译一下报错：

![img](/assets/articles/ic/stm32/cuberegister/del_err.jpg)

* **2、解决报错**

删除项目中设置的USE_HAL_DRIVER宏

![img](/assets/articles/ic/stm32/cuberegister/del_pro.jpg)

解决main.h头文件中的引用，把原来引用hal库的头文件注释，换成直接引用stm32f1xx.h

![img](/assets/articles/ic/stm32/cuberegister/del_mainh.jpg)

重新配置RCC，因此原来是用HAL库配置的，现在HAL库被删除了，所以必须修改RCC为寄存器的方式配置。

![img](/assets/articles/ic/stm32/cuberegister/del_rcc.jpg)

最后即可成功编译，剩下的就是专门使用和寄存器来写代码了，所以该芯片的相关寄存器宏都在stm32f103xb.h中定义了。

![img](/assets/articles/ic/stm32/cuberegister/del_ok.jpg)


# 编写一个点灯示例（STM32F103C8T6的开发版PC13为LED灯控制脚）

遵守STM32的开发步聚：时钟配置 -> 使能 -> 能力项配置 -> 操作 -> 循环

main.c 零实现时代码框架

```c
#include "main.h"

// 配置系统时钟
void SystemClock_Config(void);
// 配置外部8MHz的高速晶振到72MHz运行频率
void SystemClock_HSE_Config(void);
// 配置RTC的外部32.768kHz的晶振时钟
void SystemClock_LSE_Config(void);

int main(void)
{
	SystemClock_Config();
	while (1)
	{
	}
}

void SystemClock_Config(void)
{
	SystemClock_HSE_Config();
	SystemClock_LSE_Config();
}

// 8MHz 晶振时钟
void SystemClock_HSE_Config(void) {
	//TODO 
}

// LSE RTC(Real-Time Clock) 其标准频率为 32.768 kHz
void SystemClock_LSE_Config(void) {
	//TODO
}

void Error_Handler(void)
{
  __disable_irq();
  while (1)
  {
  }
}

#ifdef USE_FULL_ASSERT
void assert_failed(uint8_t *file, uint32_t line)
{

}
#endif
```

## 实现8MHz的外部高速晶振配置

用到的寄存器: RCC->CR, FLASH->ACR, RCC->CFGR 三个寄存器，从RM0008的参考手册中可以查阅到对应的使用说明。

RCC配置步聚:

![img](/assets/articles/ic/stm32/cuberegister/reg_rcc_rc_1.jpg)

![img](/assets/articles/ic/stm32/cuberegister/reg_rcc_rc_2.jpg)

![img](/assets/articles/ic/stm32/cuberegister/reg_rcc_rc_3.jpg)

**1、配置RCC的HSE使能，并等待就绪**

从RCC_CR的文档中可以看到，Bit[15:0]即低16位是HSI的配置(内部时钟默认配置)，而现在我们需要使用的是外部晶振，所以只需要关心Bit[31:16]的位说明。

要使外部配置生效，则先让RCC的HSE使能生效，查手册可知第16bit是HSEON，0关闭HSE,1开启HSE。

则： `RCC->CR |= (1U << 16)` 或写成 `RCC->CR |= RCC_CR_HSEON;`

开启使能后还不能直接配置其它参数还要等READY就绪，看第17bit的HSERDY位，所以需要等待(HSERDY=0没有准备就绪，1则准备就绪)

`while (!(RCC->CR & (1U << 17)));` 或写成 `while (!(RCC->CR & RCC_CR_HSERDY));`

**2、配置Flash访问控制**

接着比较关建的Flash访问控制配置，在配置 RCC（复位和时钟控制）时，需要配置 FLASH_ACR (Flash Access Control Register)，其核心原因是确保 CPU 能够以系统时钟 (SYSCLK) 设定的高速率正确地从 FLASH 存储器中读取指令和数据。

![img](/assets/articles/ic/stm32/cuberegister/reg_flash_acr.jpg)

从手册中可以看到FLASH_ACR的寄存器只用到低5位，同时因为STM32F103C8T6的稳定状态下最高工作频率为72MHz(这是芯片设计工艺决定的)，从手册说明中可见如果是48MHz < SYSCLK < 72MHz 时，设置为2，即0b010.

APB 总线速度限制：

* STM32F103 的 APB1 总线（连接低速外设，如定时器、UART 等）被设计限制在最高 36 MHz。
* STM32F103 的 APB2 总线（连接高速外设，如 GPIO、ADC 等）最高可达 72 MHz。
* 这些总线限制了外设的最高工作频率，也间接限定了整个系统的最高时钟。

        FLASH->ACR &= ~((uint32_t)0x07); //清除LATENCY,Bit[2:0]
	    FLASH->ACR |= (uint32_t)0x02; //设置为2个等待周期,48MHz < SysClock < 72MHz
	    // 设置使能预取缓冲区PRFTBE:(Prefetch Buffer) = 1
	    FLASH->ACR |= (1U << 4);
	    
	    上面三句其实和下面这一句是等价的
	    FLASH->ACR = 0x12; // 00010010
	    
	    但为写法阅读更加明朗，可以通过定义好的宏来进行设置如还可以写为：
	    FLASH->ACR = FLASH_ACR_LATENCY_1 | FLASH_ACR_PRFTBE; //最终值也是0x12
	    
**3、配置分频器,AHB总线，APB1，APB2总线配置(寄存器RCC_CFGR)**

![img](/assets/articles/ic/stm32/cuberegister/reg_rcc_cfgr_1.jpg)
	    
![img](/assets/articles/ic/stm32/cuberegister/reg_rcc_cfgr_2.jpg)

![img](/assets/articles/ic/stm32/cuberegister/reg_rcc_cfgr_3.jpg)

AHB总线配置从手册中可以看到(HPRE)Bit[7:4]的位是对AHB的分频配置,当设置为0xxx时则不分频，对于芯片STM32F103C8T6来说，AHB通常不分频，所以：

`RCC->CFGR &= ~(0xFU << 4);   //AHB(HCLK) 不分频`

APB1 则对Bit[10:8]3位进行配置，由于APB1从总结架构来看控制着USART2, Timers 2-7, I2C1，通常只有36MHz频率运行，所以需要2分频。则

`RCC->CFGR |= (0x4U << 8);    //APB1(PCLK1) 最高36MHz 需分频`

同样APB2的总线设置Bit[13:11]设置为0xx时不分频，即仍然以72MHz运行

`RCC->CFGR &= ~(0x7U << 11);  //APB2(PLCK2) 最高72MHz 不分频`

**4、配置PLL锁相环，参考手册可知Bit 16设置为1时开启锁相环倍频源选择(HSI,HSE).**

`RCC->CFGR |= (1U << 16); //选择 HSE 作为 PLL 输入源 (Bit 16),选择 PLL 源并设置倍频系数`

HSE的分频选择Bit 17

`RCC->CFGR &= ~(1U << 17); //如果 HSE 是 8 MHz，HSE 输入 PLL 前不分频 (Bit 17 = 0) 或二分频 (Bit 17 = 1)通常不分频`

HSE 的分频倍数设置Bit[21:18],8MHz的晶振要想运行在72MHz则需要9倍。则设置为0111

`RCC->CFGR |= (0x7U << 18); //设置 PLL 倍频因子 (Bits 21:18)。例如，要达到 72 MHz (8 MHz * 9), 设置倍频为 *9`

**5、使能PLL并等待其准备就绪**

        RCC->CR |= (1U << 24); //使能 PLL (PLLON): 置位 RCC_CR 寄存器的 PLLON 位 (Bit 24)
	    //等待 PLL 稳定 (PLLRDY): 持续检查 RCC_CR 寄存器的 PLLRDY 位 (Bit 25) 是否为 '1'
	    while (!(RCC->CR & (1U << 25)));

切换系统时钟源到PLL

        RCC->CFGR |= (0x2U << 0);//设置 RCC_CFGR 寄存器的 SW 位 (Bits 1:0) 为 0b10，选择 PLL 作为系统时钟 (SYSCLK)
	    // 等待系统时钟切换完成
	    while ((RCC->CFGR & (0x3U << 2)) != (0x2U << 2));
	    

完整的配置函数：

```c
// 8MHz 晶振时钟
void SystemClock_HSE_Config(void) {
	//外部高速晶振配置(8MHz)
	//1、使能HSE 设置CR寄存器第16位为1时开启外部时钟
	RCC->CR |= RCC_CR_HSEON; // 或直接写(1U << 16);
	//并等待HSE准备就绪
	while (!(RCC->CR & RCC_CR_HSERDY)); // 或 while (!(RCC->CR & (1U << 17)));

	//2、配置FLASH延迟（对于 72 MHz 的系统时钟，通常需要设置 2 个等待周期）
	// 设置等待周期(Latency)
	FLASH->ACR &= ~((uint32_t)0x07); //清除LATENCY,Bit[2:0]
	FLASH->ACR |= (uint32_t)0x02; //设置为2个等待周期,48MHz < SysClock < 72MHz
	// 设置使能预取缓冲区PRFTBE:(Prefetch Buffer) = 1
	FLASH->ACR |= (1U << 4);

	//3、配置分频器(CFGR),AHB，APB1，APB2
	RCC->CFGR &= ~(0xFU << 4);   //AHB(HCLK) 不分频
	RCC->CFGR |= (0x4U << 8);    //APB1(PCLK1) 最高36MHz 需分频
	RCC->CFGR &= ~(0x7U << 11);  //APB2(PLCK2) 最高72MHz 不分频

	//4、配置PLL(锁相环)
	//PLL 源选择
	RCC->CFGR |= (1U << 16); //选择 HSE 作为 PLL 输入源 (Bit 16),选择 PLL 源并设置倍频系数
	//HSE 分频选择
	RCC->CFGR &= ~(1U << 17); //如果 HSE 是 8 MHz，HSE 输入 PLL 前不分频 (Bit 17 = 0) 或二分频 (Bit 17 = 1)通常不分频
	//PLL 倍频选择
	RCC->CFGR |= (0x7U << 18); //设置 PLL 倍频因子 (Bits 21:18)。例如，要达到 72 MHz (8 MHz * 9), 设置倍频为 *9

	//5、使能PLL并等待其准备就绪
	RCC->CR |= (1U << 24); //使能 PLL (PLLON): 置位 RCC_CR 寄存器的 PLLON 位 (Bit 24)
	//等待 PLL 稳定 (PLLRDY): 持续检查 RCC_CR 寄存器的 PLLRDY 位 (Bit 25) 是否为 '1'
	while (!(RCC->CR & (1U << 25)));
	// 切换系统时钟源到PLL
	RCC->CFGR |= (0x2U << 0);//设置 RCC_CFGR 寄存器的 SW 位 (Bits 1:0) 为 0b10，选择 PLL 作为系统时钟 (SYSCLK)
	// 等待系统时钟切换完成
	while ((RCC->CFGR & (0x3U << 2)) != (0x2U << 2));

}
```

## RTC 32.768kHz的晶振配置

用到寄存器RCC->APB1ENR, PWR->CR, RCC->BDCR

![img](/assets/articles/ic/stm32/cuberegister/reg_rcc_apb1enr_1.jpg)

![img](/assets/articles/ic/stm32/cuberegister/reg_rcc_apb1enr_2.jpg)

![img](/assets/articles/ic/stm32/cuberegister/reg_rcc_apb1enr_3.jpg)

![img](/assets/articles/ic/stm32/cuberegister/reg_rcc_pwr_cr_1.jpg)

![img](/assets/articles/ic/stm32/cuberegister/reg_rcc_pwr_cr_2.jpg)

1、使能电源和备份域时钟

        //置位 RCC_APB1ENR 寄存器的 PWREN 位 (Bit 28) 以使能电源接口时钟。这是访问备份域寄存器 (包括 RTC 和 LSE) 的前提
	    RCC->APB1ENR |= (1U << 28);
	    //置位 PWR_CR 寄存器的 DBP 位 (Bit 8) 以使能对 备份域 (Backup Domain) 的写访问
	    PWR->CR |= (1U << 8);

2、使能LSE

![img](/assets/articles/ic/stm32/cuberegister/reg_rcc_bdcr_1.jpg)

![img](/assets/articles/ic/stm32/cuberegister/reg_rcc_bdcr_2.jpg)

置位 RCC_BDCR 寄存器的 LSEON 位 (Bit 0) 为1时开启LSE使能

`RCC->BDCR |= (1U << 0);`

等待LSE准备就绪，持续检查 RCC_BDCR 寄存器的 LSERDY 位 (Bit 1) 是否为 '1'

`while (!(RCC->BDCR & (1U << 1)));`

3、配置RTC时钟源(可选)，选择 LSE 作为 RTC 的时钟源

        //设置 RTC 时钟源 (RTCSEL): 设置 RCC_BDCR 寄存器的 RTCSEL 位 (Bits 9:8) 为 0b01 (LSE)
	    RCC->BDCR &= ~(0x3U << 8); //清除RTCSEL
	    RCC->BDCR |= (0x1U << 8);  //设置为LSE

4、使能RTC时钟(可选)

`RCC->BDCR |= (1U << 15); //置位 RCC_BDCR 寄存器的 RTCEN 位 (Bit 15) 以使能 RTC 时钟`

完整的配置函数

```c
// LSE RTC(Real-Time Clock) 其标准频率为 32.768 kHz
void SystemClock_LSE_Config(void) {
	//1、使能电源和备份域时钟
	//置位 RCC_APB1ENR 寄存器的 PWREN 位 (Bit 28) 以使能电源接口时钟。这是访问备份域寄存器 (包括 RTC 和 LSE) 的前提
	RCC->APB1ENR |= (1U << 28);
	//置位 PWR_CR 寄存器的 DBP 位 (Bit 8) 以使能对 备份域 (Backup Domain) 的写访问
	PWR->CR |= (1U << 8);

	//2、使能LSE
	//置位 RCC_BDCR 寄存器的 LSEON 位 (Bit 0)
	RCC->BDCR |= (1U << 0);
	//3、等待LSE准备就绪，持续检查 RCC_BDCR 寄存器的 LSERDY 位 (Bit 1) 是否为 '1'
	while (!(RCC->BDCR & (1U << 1)));
	//4、配置RTC时钟源(可选)，选择 LSE 作为 RTC 的时钟源
	//设置 RTC 时钟源 (RTCSEL): 设置 RCC_BDCR 寄存器的 RTCSEL 位 (Bits 9:8) 为 0b01 (LSE)
	RCC->BDCR &= ~(0x3U << 8); //清除RTCSEL
	RCC->BDCR |= (0x1U << 8);  //设置为LSE

	//5、使能RTC时钟(可选)
	RCC->BDCR |= (1U << 15); //置位 RCC_BDCR 寄存器的 RTCEN 位 (Bit 15) 以使能 RTC 时钟

}

//使用头文件宏定义的写法和上面的效果是一样的
void SystemClock_LSE_Config_Def(void) {
	//1、使能电源和备份域时钟
	RCC->APB1ENR |= RCC_APB1ENR_PWREN;
	PWR->CR |= PWR_CR_DBP;
	//2、使能LSE,并等待就绪
	RCC->BDCR |= RCC_BDCR_LSEON;
	while (!(RCC->BDCR & RCC_BDCR_LSERDY));
	//3、配置RTC时钟源，使能RTC时钟
	// 清除 RTCSEL 和 RTCEN 位
	RCC->BDCR &= ~(RCC_BDCR_RTCSEL | RCC_BDCR_RTCEN);
	// 设置 RTCSEL 为 LSE，并设置 RTCEN
	RCC->BDCR |= (RCC_BDCR_RTCSEL_LSE | RCC_BDCR_RTCEN);
}
```

至此，已完成最基本的时钟配置！

> HSE 和 LSE 的配置主要完成了以下任务：
> 
>	HSE (高速外部晶振) 配置：
>
>	启动并稳定外部高速晶振（如 8 MHz）。
>
>	通过 PLL（锁相环） 将时钟频率倍增（例如到 72 MHz），形成 SYSCLK (系统时钟)。
>
>	配置 AHB, APB1, APB2 三条主要总线的分频系数，确定了 CPU、SRAM、DMA 以及大部分外设的最高工作频率。
>
>	LSE (低速外部晶振) 配置：
>
>	启动并稳定外部低速晶振（32.768 kHz）。
>
>	将该时钟源分配给 RTC（实时时钟）。
{: .prompt-tips }


> 如果需要使有外设还需要：
> 
>	使能外设时钟
>
>	这是最重要的一步。即使系统时钟已经启动，外设本身的时钟门控还是关闭的，以节省功耗。
>
>	配置步骤： 必须写入 RCC 的使能寄存器：
>
>	APB2 外设： 例如 GPIOA, USART1, SPI1 等，需要置位 RCC_APB2ENR 寄存器中对应的位。
>	APB1 外设： 例如 USART2, Timers 2-7, I2C1 等，需要置位 RCC_APB1ENR 寄存器中对应的位。
>	AHB 外设： 例如 DMA, FLASH, SRAM 等，需要置位 RCC_AHBENR 寄存器中对应的位。
>	示例： 如果要使用 GPIOA，必须设置 RCC->APB2ENR |= (1U << 2);（Bit 2 是 IOPFEN，即 GPIOA 时钟使能）。
{: .prompt-danger }


## 编写一个延时函数

systick.h
```c
#ifndef INC_SYSTICK_H_
#define INC_SYSTICK_H_

#include <stdio.h>
#include <stdint.h>
#include <errno.h>

void SysTick_Init_ms(void);
void SysTick_Handler(void);
void delay_ms(uint32_t ms);

#endif 
```

systick.c
```c
#include "systick.h"
#include "stm32f103xb.h"

// 定义一个全局变量，用于记录毫秒数，uint32_t最大值2^32−1=4,294,967,295
/**
 * 溢出所需时间: g_ms_ticks 每 1 毫秒 (ms) 增加 1。我们可以计算出它从 0 溢出到最大值需要多长时间即溢出时间=最大值×每步时间溢出时间=4,294,967,295 ms
 * 约为49.7天
 */
volatile uint32_t g_ms_ticks = 0;

#define RELOAD_VALUE 71999 // (72000000 / 1000) - 1

void SysTick_Init_ms(void)
{
    // 1. 设置重载值：SysTick->LOAD 寄存器
    // SysTick 是一个 24 位定时器，最大值为 0xFFFFFF
    SysTick->LOAD = RELOAD_VALUE;

    // 2. 清除当前值：SysTick->VAL 寄存器
    // 写入任何值都会清零计数器
    SysTick->VAL = 0;

    // 3. 配置控制寄存器：SysTick->CTRL 寄存器
    // (1U << 2) | (1U << 1) | (1U << 0)
    // CLKSOURCE (Bit 2) = 1：使用 AHB 时钟 (72MHz)
    // TICKINT   (Bit 1) = 1：使能 SysTick 中断
    // ENABLE    (Bit 0) = 1：使能 SysTick 定时器
    SysTick->CTRL = (1U << 2) | (1U << 1) | (1U << 0);

    // 4.设置中断优先级
    NVIC_SetPriority(SysTick_IRQn, 0);
}

/**
  * @brief  SysTick 中断服务函数
  * @param  None
  * @retval None
  */
void SysTick_Handler(void)
{
    // 增加全局毫秒计数器
    g_ms_ticks++;
}

/**
  * @brief  毫秒级延时函数
  * @param  ms: 要延时的毫秒数 (1ms ~ 4294967295ms)
  * @retval None
  */
void delay_ms(uint32_t ms)
{
    uint32_t start_time = g_ms_ticks; // 记录延时开始时的滴答数

    // 循环等待，直到当前滴答数 - 开始滴答数 >= 设定的延时毫秒数
    // 由于 g_ms_ticks 是 volatile 变量，每次都会重新从内存读取，确保正确性
    // 由于无符号整数的数学特性，即使发生了回绕，这种减法操作（时间差）仍然是正确的，天然免疫于 SysTick 溢出
    while ((g_ms_ticks - start_time) < ms)
    {
        // 处理器进入低功耗模式或空转，等待中断
        // 在实际应用中，这里可能只是一个空循环 (NOP)，或者等待其他任务调度
    }
}
```

## 使能GPIO，配置GPIO口，进行操作

在STM32F103C8T6的开发板上，PC13脚正好是LED灯脚，同时由于GPIO是由APB2总线控制，所在要使能APB2总线

    //GPIO 使能配置
	RCC->APB2ENR |= RCC_APB2ENR_IOPCEN;
	GPIOC->CRH   &= ~(GPIO_CRH_MODE13 | GPIO_CRH_CNF13);
	GPIOC->CRH   |= GPIO_CRH_MODE13_1;   // 10MHz 输出推挽

`GPIOC->BSRR = GPIO_BSRR_BR13;   // LED 灭`

`GPIOC->BSRR = GPIO_BSRR_BS13;   // LED 亮`

完整的点亮代码main.c

```c
#include "main.h"
#include "systick.h"

void SystemClock_Config(void);
void SystemClock_HSE_Config(void);
void SystemClock_LSE_Config(void);

int main(void)
{
	//系统时钟配置
	SystemClock_Config();
	//滴塔时钟初始化
	SysTick_Init_ms();

	//GPIO 使能配置
	RCC->APB2ENR |= RCC_APB2ENR_IOPCEN;
	GPIOC->CRH   &= ~(GPIO_CRH_MODE13 | GPIO_CRH_CNF13);
	GPIOC->CRH   |= GPIO_CRH_MODE13_1;   // 10MHz 输出推挽

	while (1)
	{
		GPIOC->BSRR = GPIO_BSRR_BR13;   // LED 灭
		delay_ms(550);
		GPIOC->BSRR = GPIO_BSRR_BS13;   // LED 亮
		delay_ms(1000);
		GPIOC->BSRR = GPIO_BSRR_BR13;
		delay_ms(200);
		GPIOC->BSRR = GPIO_BSRR_BS13;
		delay_ms(200);
		GPIOC->BSRR = GPIO_BSRR_BR13;
		delay_ms(200);
		GPIOC->BSRR = GPIO_BSRR_BS13;
		delay_ms(200);
	}
}

void SystemClock_Config(void)
{
	SystemClock_HSE_Config();
	SystemClock_LSE_Config();
}

// 8MHz 晶振时钟
void SystemClock_HSE_Config(void) {
	//外部高速晶振配置(8MHz)
	//1、使能HSE 设置CR寄存器第16位为1时开启外部时钟
	RCC->CR |= RCC_CR_HSEON; // 或直接写(1U << 16);
	//并等待HSE准备就绪
	while (!(RCC->CR & RCC_CR_HSERDY)); // 或 while (!(RCC->CR & (1U << 17)));

	//2、配置FLASH延迟（对于 72 MHz 的系统时钟，通常需要设置 2 个等待周期）
	// 设置等待周期(Latency)
	FLASH->ACR &= ~((uint32_t)0x07); //清除LATENCY,Bit[2:0]
	FLASH->ACR |= (uint32_t)0x02; //设置为2个等待周期,48MHz < SysClock < 72MHz
	// 设置使能预取缓冲区PRFTBE:(Prefetch Buffer) = 1
	FLASH->ACR |= (1U << 4);

	//3、配置分频器(CFGR),AHB，APB1，APB2
	RCC->CFGR &= ~(0xFU << 4);   //AHB(HCLK) 不分频
	RCC->CFGR |= (0x4U << 8);    //APB1(PCLK1) 最高36MHz 需分频
	RCC->CFGR &= ~(0x7U << 11);  //APB2(PLCK2) 最高72MHz 不分频

	//4、配置PLL(锁相环)
	//PLL 源选择
	RCC->CFGR |= (1U << 16); //选择 HSE 作为 PLL 输入源 (Bit 16),选择 PLL 源并设置倍频系数
	//HSE 分频选择
	RCC->CFGR &= ~(1U << 17); //如果 HSE 是 8 MHz，HSE 输入 PLL 前不分频 (Bit 17 = 0) 或二分频 (Bit 17 = 1)通常不分频
	//PLL 倍频选择
	RCC->CFGR |= (0x7U << 18); //设置 PLL 倍频因子 (Bits 21:18)。例如，要达到 72 MHz (8 MHz * 9), 设置倍频为 *9

	//5、使能PLL并等待其准备就绪
	RCC->CR |= (1U << 24); //使能 PLL (PLLON): 置位 RCC_CR 寄存器的 PLLON 位 (Bit 24)
	//等待 PLL 稳定 (PLLRDY): 持续检查 RCC_CR 寄存器的 PLLRDY 位 (Bit 25) 是否为 '1'
	while (!(RCC->CR & (1U << 25)));
	// 切换系统时钟源到PLL
	RCC->CFGR |= (0x2U << 0);//设置 RCC_CFGR 寄存器的 SW 位 (Bits 1:0) 为 0b10，选择 PLL 作为系统时钟 (SYSCLK)
	// 等待系统时钟切换完成
	while ((RCC->CFGR & (0x3U << 2)) != (0x2U << 2));

}

// LSE RTC(Real-Time Clock) 其标准频率为 32.768 kHz
void SystemClock_LSE_Config(void) {
	//1、使能电源和备份域时钟
	//置位 RCC_APB1ENR 寄存器的 PWREN 位 (Bit 28) 以使能电源接口时钟。这是访问备份域寄存器 (包括 RTC 和 LSE) 的前提
	RCC->APB1ENR |= (1U << 28);
	//置位 PWR_CR 寄存器的 DBP 位 (Bit 8) 以使能对 备份域 (Backup Domain) 的写访问
	PWR->CR |= (1U << 8);

	//2、使能LSE
	//置位 RCC_BDCR 寄存器的 LSEON 位 (Bit 0)
	RCC->BDCR |= (1U << 0);
	//3、等待LSE准备就绪，持续检查 RCC_BDCR 寄存器的 LSERDY 位 (Bit 1) 是否为 '1'
	while (!(RCC->BDCR & (1U << 1)));
	//4、配置RTC时钟源(可选)，选择 LSE 作为 RTC 的时钟源
	//设置 RTC 时钟源 (RTCSEL): 设置 RCC_BDCR 寄存器的 RTCSEL 位 (Bits 9:8) 为 0b01 (LSE)
	RCC->BDCR &= ~(0x3U << 8); //清除RTCSEL
	RCC->BDCR |= (0x1U << 8);  //设置为LSE

	//5、使能RTC时钟(可选)
	RCC->BDCR |= (1U << 15); //置位 RCC_BDCR 寄存器的 RTCEN 位 (Bit 15) 以使能 RTC 时钟

}

void Error_Handler(void)
{
  __disable_irq();
  while (1)
  {
  }
}

#ifdef USE_FULL_ASSERT
void assert_failed(uint8_t *file, uint32_t line)
{

}
#endif

```