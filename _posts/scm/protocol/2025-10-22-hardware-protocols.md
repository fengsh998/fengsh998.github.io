---
layout: post
title: "硬件通讯总线协义-SPI、UART、RS232、RS485、I2C"
categories: 嵌入式
tags: 嵌入式
author: fengsh998
typora-root-url: ..
---


**SPI、UART、RS232、RS485、I2C**

|接口全称|通信模式|线数|最大速率|距离|抗干扰|典型用途|应用场景|
|----|----|---|----|----|---|----|----|
|SPI,Serial Peripheral Interface|全双工同步|4（主）+1（片选）|10–100+ Mbps|<1m|差|MCU 与外设（如 Flash、传感器、ADC）|高速、短距、板内|
|UART,Universal Asynchronous Receiver/Transmitter|全双工异步|2（TX/RX）|115200 bps ~ 几Mbps|<15m|中|调试、串口通信、Modbus|简单点对点、低速|
|RS232,Recommended Standard 232|全双工异步|2（+GND）|115.2 kbps（标称）|<15m|差|PC 与外设、工业仪表|PC 通信（老设备）|
|RS485,Recommended Standard 485|半双工差分|2（A/B）|10 Mbps+|1200m+|强|工业总线、楼宇、Modbus RTU|工业长距、多节点、抗干扰|
|I2C,Inter-Integrated Circuit|半双工同步|2（SDA+SCL）|100k/400k/1M/3.4M bps|<1m|中|芯片间通信（EEPROM、传感器）|芯片间、少引脚、多设备|



第1周：跑通 GPIO、UART、中断
第2周：ADC、DAC、定时器 PWM/输入捕获
第3周：I2C、SPI 外设（OLED、陀螺仪、Flash）
第4周：DMA + 中级外设（FSMC 液晶、SD 卡）
第5周以后：RTOS、USB、TCP/IP（LwIP）、触摸屏 GUI

以下这些例程几乎所有新手都跑过，强烈建议按顺序做一遍：

Nucleo-F401RE / F411RE / L476RG 闪烁 LED（GPIO 输出）
按键输入（GPIO 输入 + 外部中断 EXTI）
USART 串口打印 Hello World + 收发
ADC 单通道 + 多通道 + DMA
DAC 输出正弦波/锯齿波
定时器输出 PWM（呼吸灯、舵机）
定时器输入捕获（测频率、测脉宽）
I2C 读写 EEPROM（24C02）或传感器（如 MPU6050）
SPI 读写 Flash（W25Q64）
FSMC/FMC 驱动 TFT LCD（经典 ILI9341）
RTOS（FreeRTOS）多任务例程（几乎所有 H7/F7/L4 包里都有）
USB Virtual COM（CDC）虚拟串口
FATFS 文件系统（SD 卡 + SDIO/SPI）