---
layout: post
title: "STM32-了解STM32程序框架和运行原理"
categories: STM32
tags: 单片机
author: fengsh998
typora-root-url: ..
---

## 祼机环境，即没有安装过CUBEIDE或Keil5之类的IDE

STM32是内核是Arm的CPU架构，需要Arm的编译工具链，连接见文章尾部，针对自己的操作系统下载对应的工具链。

    arm-none-eabi-gcc        //编译器
    arm-none-eabi-ld/gold    //链接器
    arm-none-eabi-objcopy    //用来生成.hex或.bin
    arm-none-eabi-size       //用来显示ELF大小
    arm-none-eabi-gdb        //调试器，可选

### 手动安装
如我的是MACOS M1 芯片架构的电脑:

下载：arm-gnu-toolchain-14.3.rel1-darwin-arm64-arm-none-eabi.pkg ，以当前最新为主。安装好之后会直接安装到Applications目录，所以需要添加环境变量。

        vi ~/.zshrc 
        
        #arm gun toolchain
        export ARM_GUN_TOOLCHAIN=/Applications/ArmGNUToolchain/14.3.rel1/arm-none-eabi/bin
        export PATH="$ARM_GUN_TOOLCHAIN:$PATH"
        
        source ~/.zshrc


当环境变量设置OK之后可以执行下面的命令进行验证安装是否成功。

    arm-none-eabi-gcc --version
    arm-none-eabi-objcopy --version
    arm-none-eabi-size --version

### 使用Homebrew安装

    brew tap ArmMbed/homebrew-formulae
    brew install arm-none-eabi-gcc


### 祼机编译，点亮例子

目录结构：

    --------ProjectFolder
    |---- startup.s        //必须的使用CubeIDE生成
    |---- stm32f103c8t6.ld //必须的使用CubeIDE生成
    |---- CMakeLists.txt
    |---- arm-nono-eabi.toolchain.cmake  //不使用IDE编译时指定工具链
    |---- main.c           //主程序文件，点灯
    |---- compiler.sh


```assembly
/* -----------------------------------------------------------------------------
 * 文件名: startup.s
 * 描述:   STM32F103C8T6 (Cortex-M3) 的最小启动文件
 * 功能:   定义中断向量表, 初始化内存段 (.data, .bss), 并跳转到 main 函数。
 * -----------------------------------------------------------------------------
 */

    .syntax unified            /* 启用统一汇编语法，允许灵活的指令格式 */
    .cpu cortex-m3             /* 目标 CPU 架构为 Cortex-M3 */
    .thumb                     /* 强制使用 Thumb 指令集 (Cortex-M 默认且高效) */

/* -----------------------------------------------------------------------------
 * 1. 中断向量表 (Interrupt Vector Table, IVT)
 * 该表必须位于 Flash 的起始地址 (0x08000000)
 * -----------------------------------------------------------------------------
 */

    .section .isr_vector,"a",%progbits  /* 定义一个名为 .isr_vector 的段，"a" 可分配，"%progbits" 包含程序数据 */
    .word _estack                       /* 向量表第 0 项: 初始栈指针 (Initial SP) ，_estack 符号由链接器脚本 (.ld) 定义*/
    .word Reset_Handler                 /* 向量表第 1 项: 复位处理程序 (Reset Handler)，MCU 上电/复位后执行的第一条指令地址 */
    
/*最小化配置: 其余所有向量都指向 Default_Handler，Cortex-M3 通常有 15 个系统异常 + 芯片特有的中断*/
    .rept 62                            /* 重复 62 次，用于填充剩余的中断向量，STM32F103 的中断向量总数通常是 64 个 */
      .word Default_Handler             /* 将所有未定义的异常/中断都指向 Default_Handler */
    .endr

/* -----------------------------------------------------------------------------
 * 2. 复位处理程序 (Reset Handler)
 * 这是启动文件最核心的部分，负责内存初始化
 * -----------------------------------------------------------------------------
 */
    .section .text.Reset_Handler        /* 定义 Reset_Handler 函数所在的文本段 */
    .weak Reset_Handler                 /* 允许 Reset_Handler 被其他文件覆盖 (弱符号) */
    .type Reset_Handler, %function      /* 声明 Reset_Handler 是一个函数 */
Reset_Handler:
    /* 2.1 复制 .data 段 (初始化数据) 从 Flash 到 RAM，.data 段存储已初始化的全局变量和静态变量的初始值   */
    ldr   r0, =_sidata                  /* r0 = Flash 中的 .data 源地址 (Source Data) */
    ldr   r1, =_sdata                   /* r1 = RAM 中的 .data 目标起始地址 (Start Data) */
    ldr   r2, =_edata                   /* r2 = RAM 中的 .data 目标结束地址 (End Data) */
copy_data_loop:
    cmp   r1, r2                        /* 比较 r1 和 r2 (当前目标地址 vs 结束地址) */
    bcs   copy_data_done                /* 如果 r1 >= r2 (即复制完成)，跳转到 done (无符号比较) */
    ldr   r3, [r0], #4                  /* 从 r0 指向的 Flash 地址加载一个 4 字节数据到 r3，r0 自动增加 4 */
    str   r3, [r1], #4                  /* 将 r3 中的数据存储到 r1 指向的 RAM 地址，r1 自动增加 4 */
    b     copy_data_loop                /* 继续循环 */
copy_data_done:

    /* 2.2 清零 .bss 段 (未初始化数据) ,.bss 段存储未初始化的全局变量和静态变量，在 C 语言标准中要求初始化为 0*/
    ldr   r0, =_sbss                    /* r0 = RAM 中的 .bss 起始地址 (Start BSS) */
    ldr   r1, =_ebss                    /* r1 = RAM 中的 .bss 结束地址 (End BSS) */
    movs  r2, #0                        /* r2 = 0 (用于存储零值) */
zero_bss_loop:
    cmp   r0, r1                        /* 比较 r0 和 r1 (当前地址 vs 结束地址) */
    bcs   zero_bss_done                 /* 如果 r0 >= r1 (即清零完成)，跳转到 done */
    str   r2, [r0], #4                  /* 将 r2 (0) 存储到 r0 指向的 RAM 地址，r0 自动增加 4 */
    b     zero_bss_loop                 /* 继续循环 */
zero_bss_done:

    /* 2.3 调用 C 语言的 main 函数 */
    bl    main                          /* Branch with Link (跳转并链接) 到 main 函数,bl 会将返回地址存入 LR 寄存器 */

    /* 2.4 如果 main 函数返回，进入无限循环 (嵌入式系统通常不返回) */
hang:
    b hang                              /* Branch (跳转) 到自身，进入死循环，防止程序跑飞 */

    .size Reset_Handler, .-Reset_Handler   /* 定义 Reset_Handler 函数的大小 */

/* -----------------------------------------------------------------------------
 * 3. 默认中断/异常处理程序 (Default Handler)
 * -----------------------------------------------------------------------------
 */
    .section .text.Default_Handler     /* 定义 Default_Handler 函数所在的文本段 */
    .weak Default_Handler              /* 允许 Default_Handler 被用户定义的函数覆盖 */
    .type Default_Handler, %function
Default_Handler:
    b Default_Handler                  /* 进入死循环，若发生未处理的中断或异常，则停在此处 */

    .size Default_Handler, .-Default_Handler  /* 定义 Default_Handler 函数的大小 */

```

连接文件，我这里取名STM32F103C8T6.ld

```assembly
/* Linker script for STM32F103C8T6 (64KB flash, 20KB RAM), 入口点程序，上电后将会执行，此函数在.s启动汇编文件中定义*/
ENTRY(Reset_Handler)

/*
*  内存映射，这部分根据芯片型号定义了物理内存块的名称、起始地址和大小
*  FLASH: 程序代码和常量存储区。这是 STM32F1 系列 Flash 的标准起始地址。
*  RAM: 运行时数据存储区（堆栈、堆、全局变量）。这是 STM32F1 系列 SRAM 的标准起始地址。
*/
MEMORY
{
  FLASH (rx) : ORIGIN = 0x08000000, LENGTH = 64K
  RAM   (rwx): ORIGIN = 0x20000000, LENGTH = 20K
}

/* 堆栈定义，定义了符号 _estack，它用于 startup.s 文件中的中断向量表 */
_estack = ORIGIN(RAM) + LENGTH(RAM);

/*
*这部分定义了编译后的程序段如何分配到前面定义的 MEMORY 区域中，并定义了 startup.s 需要使用的关键符号。
*/
SECTIONS
{
  .isr_vector :
  {
    KEEP(*(.isr_vector))
  } >FLASH

  .text :
  {
    *(.text*)
    *(.rodata*)
    KEEP(*(.init))
    KEEP(*(.fini))
    . = ALIGN(4);
    _etext = .;
  } >FLASH

  .data : AT (ADDR(.text) + SIZEOF(.text))
  {
    _sidata = LOADADDR(.data);
    _sdata = .;
    *(.data*)
    . = ALIGN(4);
    _edata = .;
  } >RAM

  .bss :
  {
    _sbss = .;
    *(.bss*)
    *(COMMON)
    . = ALIGN(4);
    _ebss = .;
  } >RAM

  /* Optional debug/frame info */
  /DISCARD/ : { *(.eh_frame) }
}
``` 

上面两个文件只帮助了解，不需要编写，可以通过CubeIDE自动生成的。这两个也是STM32中必须的文件，没有的话就启动不了执行不了main函数。

工具链文件：arm-none-eabi.toolchain.cmake

```cmake
# Toolchain file for GNU Arm Embedded (arm-none-eabi) on macOS (Apple Silicon).
# Must be passed to cmake as -DCMAKE_TOOLCHAIN_FILE=... (so it is read BEFORE project())

# 目标系统定义，即编译最终运行的目标环境，无操作系统的嵌入式系统，通常设置为 Generic
set(CMAKE_SYSTEM_NAME Generic)
# 定义CPU的架构
set(CMAKE_SYSTEM_PROCESSOR arm)

# 这几行是在 macOS 或 iOS 上进行交叉编译时的关键步骤。它们清空了 macOS SDK 特定的变量，阻止 CMake 默认链接到 macOS 的库和框架，确保我们构建的是一个纯净的嵌入式二进制文件。
set(CMAKE_OSX_SYSROOT "")
set(CMAKE_OSX_DEPLOYMENT_TARGET "")
set(CMAKE_OSX_ARCHITECTURES "")

# 交叉编译器路径定义
set(CMAKE_C_COMPILER   "/Applications/ArmGNUToolchain/14.3.rel1/arm-none-eabi/bin/arm-none-eabi-gcc")
set(CMAKE_ASM_COMPILER "${CMAKE_C_COMPILER}")
# 禁用C++编译器:
set(CMAKE_CXX_COMPILER "")

# Ensure try_compile uses a static library (avoids linking with host toolchain/SDK)
set(CMAKE_TRY_COMPILE_TARGET_TYPE STATIC_LIBRARY)

# 初始编译标志定义，可参考Cortex-M 和GCC文档
set(CMAKE_C_FLAGS_INIT    "-mcpu=cortex-m3 -mthumb -ffreestanding -fno-builtin -Wall -O2 -g -nostdlib")
set(CMAKE_ASM_FLAGS_INIT  "-mcpu=cortex-m3 -mthumb")
```

CMakeLists.txt

```cmake
# -----------------------------------------------------------
# 1. 最小要求和项目定义
# -----------------------------------------------------------

cmake_minimum_required(VERSION 3.13)
# 定义项目名称和所使用的语言 (C 和 汇编)
project(stm32-blinky C ASM)

# -----------------------------------------------------------
# 2. 核心编译标志定义
# -----------------------------------------------------------
# 定义用于 MCU 目标的核心编译标志列表。
# 这些标志定义了目标架构和裸机环境。
set(MCU_FLAGS_LIST -mcpu=cortex-m3 -mthumb -ffreestanding -fno-builtin)

# 定义 C 编译器的标志 -Wall: 开启所有警告，-O2: 优化等级 2，-g: 生成调试信息，-nostdlib: 不链接标准 C 库 (因为我们是裸机，使用自定义启动代码)
set(C_FLAGS ${MCU_FLAGS_LIST} -Wall -O2 -g -nostdlib)

# 定义汇编器的标志 (仅需指定架构)
set(ASM_FLAGS ${MCU_FLAGS_LIST})

# 定义链接器的标志 (最关键的部分)
# -T: 指定链接器脚本文件 (.ld)
# -Wl: 将参数传递给链接器，生成内存映射文件 (.map)
# -Wl: 启用垃圾回收，移除未使用的函数和数据
set(LINK_FLAGS ${MCU_FLAGS_LIST} -T${CMAKE_SOURCE_DIR}/stm32f103c8t6.ld -Wl,-Map=${PROJECT_NAME}.map,--cref -Wl,--gc-sections)

# -----------------------------------------------------------
# 3. 查找工具链程序
# -----------------------------------------------------------
# 查找 arm-none-eabi-objcopy 工具，用于将 .elf 文件转换为 .hex/.bin
find_program(OBJCOPY NAMES arm-none-eabi-objcopy objcopy)
if(NOT OBJCOPY)
  message(FATAL_ERROR "arm-none-eabi-objcopy not found; install GNU ARM toolchain and ensure in PATH")
endif()
find_program(SIZE_PROGRAM NAMES arm-none-eabi-size size)
if(NOT SIZE_PROGRAM)
  message(WARNING "arm-none-eabi-size not found; size output won't be available")
endif()

# -----------------------------------------------------------
# 4. 定义源代码和目标可执行文件
# -----------------------------------------------------------
# 定义所有源文件列表
set(SOURCES
    startup.s
    main.c
)

# 创建可执行目标文件，输出名为 stm32-blinky.elf
add_executable(${PROJECT_NAME}.elf ${SOURCES})

# -----------------------------------------------------------
# 5. 应用编译和链接选项
# -----------------------------------------------------------
# 将前面定义的 C 语言标志应用到目标
target_compile_options(${PROJECT_NAME}.elf PRIVATE ${C_FLAGS})

# 指定 C 语言标准为 C11
set_property(TARGET ${PROJECT_NAME}.elf PROPERTY C_STANDARD 11)

# 设置汇编器标志（对某些生成器/版本是必需的）
set_target_properties(${PROJECT_NAME}.elf PROPERTIES
  ASM_FLAGS "${ASM_FLAGS}"
)

# 链接选项: 推荐使用 target_link_options
if(POLICY CMP0079)
  target_link_options(${PROJECT_NAME}.elf PRIVATE ${LINK_FLAGS})
else()
  # 兼容旧版本 CMake 的后备方案
  set_target_properties(${PROJECT_NAME}.elf PROPERTIES LINK_FLAGS "${LINK_FLAGS}")
endif()

# -----------------------------------------------------------
# 6. 后续构建命令 (Post-Build Commands)
# -----------------------------------------------------------
# 在链接生成 .elf 文件之后执行的命令
add_custom_command(TARGET ${PROJECT_NAME}.elf POST_BUILD
  COMMAND ${OBJCOPY} -O ihex $<TARGET_FILE:${PROJECT_NAME}.elf> ${PROJECT_NAME}.hex
  COMMAND ${OBJCOPY} -O binary $<TARGET_FILE:${PROJECT_NAME}.elf> ${PROJECT_NAME}.bin
  COMMAND ${SIZE_PROGRAM} $<TARGET_FILE:${PROJECT_NAME}.elf> || true
  COMMENT "Generating HEX/BIN and printing size"
)
```

main.c

```c
/* Bare-metal blinking LED on PC13 (BluePill) for STM32F103C8T6.
   No HAL, direct register access. */
#include <stdint.h>

/* Register addresses (STM32F1) */
#define RCC_BASE        0x40021000UL
#define GPIOC_BASE      0x40011000UL

#define RCC_APB2ENR     (*(volatile uint32_t *)(RCC_BASE + 0x18))
#define GPIOC_CRH       (*(volatile uint32_t *)(GPIOC_BASE + 0x04))
#define GPIOC_BSRR      (*(volatile uint32_t *)(GPIOC_BASE + 0x10))
#define GPIOC_BRR       (*(volatile uint32_t *)(GPIOC_BASE + 0x14))
#define GPIOC_ODR       (*(volatile uint32_t *)(GPIOC_BASE + 0x0C))

/* Pin */
#define LED_PIN 13

/* Simple delay (busy loop) */
static void delay(volatile uint32_t d) {
    while (d--) {
        /* prevent optimizing away */
        __asm__ volatile("nop");
    }
}

int main(void) {
    /* Enable clock for GPIOC: APB2ENR IOPCEN bit = 4 */
    RCC_APB2ENR |= (1U << 4);

    /* Configure PC13 as output, push-pull, 2 MHz (MODE = 10, CNF = 00)
       PC13 is in CRH (pins 8..15), each pin has 4 bits */
    uint32_t shift = (LED_PIN - 8) * 4;
    GPIOC_CRH &= ~(0xFU << shift);         /* clear bits */
    GPIOC_CRH |=  (0x2U << shift);         /* MODE=10 (output 2MHz), CNF=00 */

    while (1) {
        /* Set PC13 = 1 (BSRR lower) -> LED off on some BluePill wiring; adjust as needed */
        GPIOC_BSRR = (1U << LED_PIN);
        delay(200000);

        /* Reset PC13 */
        GPIOC_BRR = (1U << LED_PIN);
        delay(200000);
    }

    /* should never reach */
    return 0;
}
```

compiler.sh

```shell
#!/bin/bash

# 确保在项目根目录运行此脚本
if [ ! -f "CMakeLists.txt" ]; then
    echo "错误：请在包含 CMakeLists.txt 的项目根目录运行此脚本。"
    exit 1
fi

echo "--- 1. 清理旧的构建目录 ---"
rm -rf build

echo "--- 2. 创建并进入新的构建目录 ---"
mkdir build
cd build

# 假设您的工具链文件名为 arm-none-eabi.toolchain.cmake
TOOLCHAIN_FILE="../arm-none-eabi.toolchain.cmake"

if [ ! -f "$TOOLCHAIN_FILE" ]; then
    echo "错误：找不到工具链文件 $TOOLCHAIN_FILE。请检查路径和名称。"
    cd ..
    exit 1
fi

echo "--- 3. 运行 CMake 配置 (交叉编译) ---"
cmake .. -DCMAKE_TOOLCHAIN_FILE="$TOOLCHAIN_FILE" || { echo "CMake 配置失败"; cd ..; exit 1; }

echo "--- 4. 执行编译 (Verbose 模式) ---"
cmake --build . --verbose || { echo "编译失败"; cd ..; exit 1; }

echo "--- 编译完成！---"
echo "您可以在 'build' 目录中找到 stm32-blinky.elf, stm32-blinky.hex 和 stm32-blinky.bin 文件。"

cd ..
```

通过编译脚本可以编译出.elf,.hex,.bin文件，可以烧录到芯片上进行测试。

<font color="red">说明：</font>

代码中并没有配置系统时钟和时钟控制，是因为默认情况下不配置时钟STM32F103C8T6内部默认有(HSI)8MHz的时钟振荡，在没有配置的情况下，会自动启用。虽说8M能跑，但太慢了，而且还不能使用外设。由于上面的示例虽说延时的精度也够，但只是点个LED够了。




[【arm Gun 工具链】](https://developer.arm.com/Tools%20and%20Software/GNU%20Toolchain)

[【arm Gun Toolchain 下载】](https://developer.arm.com/downloads/-/arm-gnu-toolchain-downloads)