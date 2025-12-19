---
layout: post
title: "STM32-LED灯字模(练习一)"
categories: STM32
tags: 单片机
author: fengsh998
typora-root-url: ..
---

**环境说明：**
* 使用芯片(STM32F103C8T6)
* proteus 9 
* 开发环境STM32CubeIDE
* 基于寄存器开发

使用Proteus进行仿真显示。仿真模拟电路搭建如图8*8的LED

![img](/assets/articles/ic/stm32/ledMatrix/led-1.jpg)

设置仿真芯片属性配置OSC Frequency 为 72000000 即72MHz外部晶振

![img](/assets/articles/ic/stm32/ledMatrix/led-2.jpg)


阳极是: PA1 ~ PA8

阴极是: PB0 ~ PB7

采用的是行扫描

gpio_config.h
```c
#ifndef INC_GPIO_CONFIG_H_
#define INC_GPIO_CONFIG_H_

#include "stm32f103xb.h"

/* =============================PIN脚定义 =========================== */
#define Pin_0  	0
#define Pin_1  	1
#define Pin_2  	2
#define Pin_3  	3
#define Pin_4  	4
#define Pin_5  	5
#define Pin_6  	6
#define Pin_7  	7
#define Pin_8  	8
#define Pin_9  	9
#define Pin_10  10
#define Pin_11  11
#define Pin_12  12
#define Pin_13  13
#define Pin_14  14
#define Pin_15  15

#define High 	1
#define Low  	0

/* ==================== 配置模式宏定义（F1系列专用） ==================== */
//  CNF[1:0] + MODE[1:0] 组合
#define GPIO_MODE_IN_FLOATING     0x04U   // 0100 : 悬空输入（推荐默认）
#define GPIO_MODE_IN_PULLUP       0x08U   // 1000 : 上拉输入
#define GPIO_MODE_IN_PULLDOWN     0x08U   // 1000 : 下拉输入（写ODR=0实现下拉）
#define GPIO_MODE_IN_ANALOG       0x00U   // 0000 : 模拟输入

#define GPIO_MODE_OUT_10MHZ_PP    0x01U   // 0001 : 推挽输出 10MHz
#define GPIO_MODE_OUT_10MHZ_OD    0x05U   // 0101 : 开漏输出 10MHz
#define GPIO_MODE_OUT_10MHZ_AFPP  0x09U   // 1001 : 复用推挽 10MHz
#define GPIO_MODE_OUT_10MHZ_AFOD  0x0DU   // 1101 : 复用开漏 10MHz

#define GPIO_MODE_OUT_2MHZ_PP     0x02U   // 0010
#define GPIO_MODE_OUT_2MHZ_OD     0x06U   // 0110
#define GPIO_MODE_OUT_2MHZ_AFPP   0x0AU
#define GPIO_MODE_OUT_2MHZ_AFOD   0x0EU

#define GPIO_MODE_OUT_50MHZ_PP    0x03U   // 0011 : 最常用！
#define GPIO_MODE_OUT_50MHZ_OD    0x07U   // 0111
#define GPIO_MODE_OUT_50MHZ_AFPP  0x0BU   // 1011
#define GPIO_MODE_OUT_50MHZ_AFOD  0x0FU   // 1111

/**
 * 内部带上了设置对应区的总线使能，不需要外部再设置使能
 *
 * GPIOx: GPIO_TypeDef* GPIO[A~G]区的结构体引用
 * pins: uint8_t[] 0～15脚数组
 * count: uint16_t 数组长度
 * Mode: uint8_t 模式如GPIO_MODE_OUT_50MHZ_PP
 */
void GPIO_PinsConfig(GPIO_TypeDef* GPIOx, const uint8_t pins[], uint16_t count, uint8_t Mode);
void GPIO_PinsConfigForPP50(GPIO_TypeDef* GPIOx, const uint8_t pins[], uint16_t count);

/**
 * GPIOx: A~G 组
 * pins:具体的pin脚如要设置1，3脚则写成 [1,3]
 * count: 数组长度
 * val: 只能0,和1，0表示低电平，1表示高电平，其它值时均被忽略
 * @note   内部自动使用 BSRR/BRR（原子操作，中断安全），性能最高
 */
void GPIO_PinsSetArray(GPIO_TypeDef *GPIOx, const uint8_t pins[], uint16_t count,
		uint8_t val);
void GPIO_PinsSet(GPIO_TypeDef *GPIOx, uint8_t pin, uint8_t val);

#endif
```

gpio_config.c
```c
/**
 U : unsigned int
 UL : unsigned long

 GPIOA->CRL |= 0x03; //PA0
 GPIOA->CRL |= (0x03 << 4); //PA1
 GPIOA->CRL |= (0x03 << 8); //PA2
 GPIOA->CRL |= (0x03 << 12); //PA3
 GPIOA->CRL |= (0x03 << 16); //PA4
 GPIOA->CRL |= (0x03 << 20); //PA5
 GPIOA->CRL |= (0x03 << 24); //PA6
 GPIOA->CRL |= (0x03 << 28); //PA7

 GPIOA->CRH |= 0x03; //PA8

 //效率最高的写法
 GPIOA->CRL = 0x33333333;   // PA0~PA7  推挽 50MHz
 GPIOA->CRH = 0x33333333;   // PA8~PA15 推挽 50MHz

 0x10 = CNF01 MODE00

 */

#include "gpio_config.h"

void GPIO_PinsConfig(GPIO_TypeDef *GPIOx, const uint8_t pins[], uint16_t count,
		uint8_t Mode) {
	uint32_t crl = 0x04040404U;  // 默认全悬空输入 0~7Pin
	uint32_t crh = 0x04040404U;  // 8~15pin
	uint16_t pull_mask = 0;

	// 使能时钟（同之前）
	if (GPIOx == GPIOA)
		RCC->APB2ENR |= RCC_APB2ENR_IOPAEN;
	else if (GPIOx == GPIOB)
		RCC->APB2ENR |= RCC_APB2ENR_IOPBEN;
	else if (GPIOx == GPIOC)
		RCC->APB2ENR |= RCC_APB2ENR_IOPCEN;
	else if (GPIOx == GPIOD)
		RCC->APB2ENR |= RCC_APB2ENR_IOPDEN;
	else if (GPIOx == GPIOE)
		RCC->APB2ENR |= RCC_APB2ENR_IOPEEN;
#if defined(GPIOG)
    else if(GPIOx == GPIOG) RCC->APB2ENR |= RCC_APB2ENR_IOPGEN;
    #endif

	for (uint16_t i = 0; i < count; i++) {
		uint8_t pin = pins[i];
		if (pin > 15)
			continue;  // 防呆

		if (pin < 8) {
			crl &= ~(0xFU << (pin * 4));
			crl |= (uint32_t) Mode << (pin * 4);
		} else {
			crh &= ~(0xFU << ((pin - 8) * 4));
			crh |= (uint32_t) Mode << ((pin - 8) * 4);
		}

		// 记录需要上拉/下拉的引脚
		if (Mode == GPIO_MODE_IN_PULLUP)
			pull_mask |= (1U << pin);
		if (Mode == GPIO_MODE_IN_PULLDOWN)
			pull_mask &= ~(1U << pin);
	}

	GPIOx->CRL = crl;
	GPIOx->CRH = crh;

	// 只有在输入模式下才是必须的， 统一设置上拉/下拉（只写一次 ODR，比循环写快 10 倍）
	if (Mode == GPIO_MODE_IN_PULLUP)
		GPIOx->ODR |= pull_mask;
	if (Mode == GPIO_MODE_IN_PULLDOWN)
		GPIOx->ODR &= ~pull_mask;
}

void GPIO_PinsConfigForPP50(GPIO_TypeDef *GPIOx, const uint8_t pins[],
		uint16_t count) {
	GPIO_PinsConfig(GPIOx, pins, count, GPIO_MODE_OUT_50MHZ_PP);
}

void GPIO_PinsSetArray(GPIO_TypeDef *GPIOx, const uint8_t pins[], uint16_t count,
		uint8_t val) {
	if (val != 0 && val != 1) {
		return;              // 参数非法，直接返回（防呆）
	}

	/* 1. 先把所有要操作的引脚整理成 set/clr 掩码 */
	for (uint16_t i = 0; i < count; i++) {
		uint8_t pin = pins[i];
		if (pin > 15)
			continue;                // 非法引脚直接跳过

		GPIO_PinsSet(GPIOx,pin, val);
	}
}

void GPIO_PinsSet(GPIO_TypeDef *GPIOx, uint8_t pin, uint8_t val) {
	uint32_t set_mask = 0;   // 需要置 1 的位
	uint32_t clr_mask = 0;   // 需要置 0 的位

	if (val == 1) {
		set_mask |= (1UL << pin);          // 要置高
	} else {
		clr_mask |= (1UL << pin);          // 要置低
	}

	/* 2. 优先使用 BRR 寄存器（部分芯片有，速度更快） */
#if defined(GPIO_BRR_BR0)
	if (clr_mask) {
		GPIOx->BRR = clr_mask;                 // 有 BRR 寄存器 → 最快
	}
#else
	/* 没有 BRR 寄存器时用 BSRR 高16位 */
	if (clr_mask) {
	   GPIOx->BSRR = clr_mask << 16;
	}
#endif

	/* 3. 置高永远走 BSRR 低16位（所有芯片都支持） */
	if (set_mask) {
		GPIOx->BSRR = set_mask;
	}
}
```

字模的定义及显示方式

led_matrix.h
```c
#ifndef INC_LED_MATRIX_H_
#define INC_LED_MATRIX_H_

#include <stdint.h>

// 只声明，不分配存储空间
extern const uint8_t char_font_A[8][8];
extern const uint8_t char_font_B[8][8];
extern const uint8_t heart[8][8];

void LED_Init(const uint8_t row8Pins[8],const uint8_t col8Pins[8]);

/*
 * 显示 8*8 的字模
 * font: 字模数组
 * row8Pins: 8pin 行数组,接阳极
 * col8Pins: 8pin 列数组,接阴极
 *
 * 接线为阴极：PA1 为 第0 行
 * 接线为阴极：PB7 为 第0 列，如果改为PB0为第0列时，这行uint8_t val = font[i][7-j];改为uint8_t val = font[i][j];
 *
 */
void RowScanDisplay(const uint8_t font[8][8], const uint8_t row8Pins[8],const uint8_t col8Pins[8]);
/**
 * @brief  阻塞式显示某个字模一段时间
 * @param  font:      字模数组 8x8
 * @param  row_pins:  行引脚数组（共阳）
 * @param  col_pins:  列引脚数组（共阴）
 * @param  count:  显示次数
 */
void RowScanDisplay_ForTime(const uint8_t font[8][8],
                            const uint8_t row_pins[8],
                            const uint8_t col_pins[8],
                            uint32_t count);
                            

#endif
```

led_matrix.c
```c
#include "led_matrix.h"
#include "gpio_config.h"
#include "systick.h"
#include "stm32f103xb.h"

const uint8_t char_font_A[8][8] = {
		// 行 0 (0x18)
		{0,0,0,1,1,0,0,0},
		// 行 1 (0x24)
		{0,0,1,0,0,1,0,0},
		// 行 2 (0x42)
		{0,1,0,0,0,0,1,0},
		// 行 3 (0x42)
		{0,1,0,0,0,0,1,0},
		// 行 4 (0x7E)
		{0,1,1,1,1,1,1,0},
		// 行 5 (0x42)
		{0,1,0,0,0,0,1,0},
		// 行 6 (0x42)
		{0,1,0,0,0,0,1,0},
		// 行 7 (0x00)
		{0,0,0,0,0,0,0,0}
};

const uint8_t char_font_B[8][8] = {
		{0,1,1,1,1,1,0,0},
		{0,1,0,0,0,0,1,0},
		{0,1,0,0,0,0,1,0},
		{0,1,1,1,1,1,0,0},
		{0,1,0,0,0,0,1,0},
		{0,1,0,0,0,0,1,0},
		{0,1,1,1,1,1,0,0},
		{0,0,0,0,0,0,0,0}
};

// 字模：心形
const uint8_t heart[8][8] = {
    {0,1,1,0,0,1,1,0},
    {1,1,1,1,1,1,1,1},
    {1,1,1,1,1,1,1,1},
    {1,1,1,1,1,1,1,1},
    {0,1,1,1,1,1,1,0},
    {0,0,1,1,1,1,0,0},
    {0,0,0,1,1,0,0,0},
    {0,0,0,0,0,0,0,0}
};

// 初始状态：全部熄灭
void LED_Init(const uint8_t row8Pins[8],const uint8_t col8Pins[8]) {
	// 熄灭所有行（阳极拉低），清零 PA1-PA8 位
	// 熄灭所有列（阴极拉高，共阴）
	for (int i = 0; i < 8; i++) {
		GPIO_PinsSet(GPIOA, row8Pins[i],Low);
		GPIO_PinsSet(GPIOB, col8Pins[i], High);
	}
}

void RowScanDisplay(const uint8_t font[8][8],const uint8_t row8Pins[8],const uint8_t col8Pins[8]) {
	int row_count = 8;
	int col_count = 8;
	for (int i = 0; i < row_count ; i++) {
		// 1. 先把所有列设置为高（不亮），防止鬼影
		for (int c = 0; c < 8; c++) {
		    GPIO_PinsSet(GPIOB, col8Pins[c], High); //把阴级至为高电平
		}
		//当前行先设置整行为高电平（阳极高）
		GPIO_PinsSet(GPIOA,row8Pins[i],High);

		for (int j = 0; j < col_count ; j++) {
			uint8_t val = font[i][7-j]; //读取[row,col]的电平，是1时设为高电平，0时设为低电平
			GPIO_PinsSet(GPIOB,col8Pins[j],val == 1 ? Low : High); //当字模对应是高电平时，则设置拉低电平，亮灯
		}
		//延时2毫秒
		//delay_ms(2); //实物电路用这个
		delay_ms_proteus(2); //使用proteus时用这个，显示清楚点。
		GPIO_PinsSet(GPIOA,row8Pins[i],Low); //灭当前行
	}
}

void RowScanDisplay_ForTime(const uint8_t font[8][8],
                            const uint8_t row_pins[8],
                            const uint8_t col_pins[8],
                            uint32_t count)
{
	for (int cycle = 0; cycle < count; cycle++) {
		RowScanDisplay(font, row_pins, col_pins);
	}
}
```

main.c
```c
#include "main.h"
#include "systick.h"
#include "gpio_config.h"
#include "led_matrix.h"

void SystemClock_Config(void);
void SystemClock_HSE_Config(void);
void SystemClock_LSE_Config(void);

//0~7脚CHL，8～15，CHR
//阳极脚PA1～PA8
const uint8_t High_Pins[8] = {Pin_1,Pin_2,Pin_3,Pin_4,Pin_5,Pin_6,Pin_7,Pin_8};
//阴极脚PB0~PB7
const uint8_t Low_Pins[8] = {Pin_0,Pin_1,Pin_2,Pin_3,Pin_4,Pin_5,Pin_6,Pin_7};

void LEDPin_initConfig(void) {
	// 配置GPIO脚为推挽输出,50MHz,并且默认都是低电平
	GPIO_PinsConfigForPP50(GPIOA,High_Pins,8);
	GPIO_PinsConfigForPP50(GPIOB,Low_Pins,8);

	LED_Init(High_Pins,Low_Pins);
}

int main(void)
{
	//系统时钟配置
	SystemClock_Config();
	//滴塔时钟初始化
	SysTick_Init_ms();
	//初始代GPIOA，GPIOB
	LEDPin_initConfig();

	//GPIO 使能配置
	RCC->APB2ENR |= RCC_APB2ENR_IOPCEN;
	//清除 PC13 的 4 个配置位，设置为浮空输入 (0000)
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

		RowScanDisplay_ForTime(char_font_A,High_Pins,Low_Pins,5);
		RowScanDisplay_ForTime(char_font_B,High_Pins,Low_Pins,5);
		RowScanDisplay_ForTime(heart,High_Pins,Low_Pins,5);
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
	// 先清除 PPRE1 位，再设置 /2 分频
	RCC->CFGR &= ~RCC_CFGR_PPRE1;
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




