---
layout: post
title: "STM32-使用STM32CubeIDE构建一个基本的寄存器开发环境"
categories: STM32
tags: 单片机
author: fengsh998
typora-root-url: ..
---

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