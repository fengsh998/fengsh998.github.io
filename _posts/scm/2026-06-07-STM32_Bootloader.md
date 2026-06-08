---
layout: post
title: "BOOT0,BOOTLOADER,OTA"
categories: 单片机
tags: 启动程序
author: fengsh998
typora-root-url: ..
---

### BOOT0，BOOT1 跳线脚

BOOT0 为启动脚，接不同的电位时启动区不同。具体要看说明书reference manual手册。

<font color='red'> 决定上电后是走 **主程** 还是走 **Bootloader** </font>

用于决定MCU上电后怎么运行的跳线脚，有此芯片只有BOOT0的Pin脚，有些有BOOT0和BOOT1，这个取决于MCU。

下面以STM32G0xxx的说明书为例说明：

![img](/assets/articles/ic/bootloader/Boot0.jpg)

存储区域说明：

|区域|作用说明|谁来干|
|----|----|----|
|**Main Flash Memory**|用户程序存储区，掉电不丢失|嵌入式工程量写的程序烧在这里|
|**System Memory**| ST出厂固化的Bootloader(只读)|ST工厂写死的，改不了，也删不掉它。|
|**Eembedded SRAM**|运行时临时内存，掉电丢失|程序运行时用，不能烧录|

图中只有Boot0 Pin是物理脚：

* nBOOT1 bit, nBOOT_SEL bit, nBOOT0 bit 这三个是软件寄存器位，在芯片内部Flash的Option Bytes里
 ```
 Reference Manual（RMxxxx）→  Boot configuration 章节
                              看 Table: Boot modes
                              看 Option Bytes 默认值
 ```

* Boot0 接上接电阻到VCC时，则为1，接下拉电阻到GND时则为0，x则为任意值。

BOOT0 接（下拉电阻）GND，= 0时，上电就启动Main Flash Memory，即你开发程序烧录的主程。

BOOT0 接（上拉电阻）VCC，= 1时，上电后就从System memory 启动，这正是心片厂商出厂时默认写的Bootloader（常说的ISP串口烧录）区，这个区是只读不可写的区域，它包含一套识别串口/USB 信号的算法，负责接收你的数据并写入 Main Flash Memory。

**完整的启动流程**
```
上电
  ↓
芯片采样 BOOT0 pin（第4个 SYSCLK 上升沿）
  ↓
BOOT0 = 0          BOOT0 = 1（nBOOT_SEL=0时）
  ↓                    ↓
跳转到               跳转到
Main Flash           System Memory
（运行你的程序）      （运行ST的串口Bootloader）
                         ↓
                    等待串口接收程序
                    写入 Main Flash
                    复位后再从Flash启动
```

**如果想从Embedded SRAM启动**

```
上电
  │
  ├─ BOOT0=0 ──────────────────→ Main Flash（你的程序）
  │
  └─ BOOT0=1
       │
       ├─ nBOOT1=1 ────────────→ System Memory（ST串口Bootloader）
       │                              │
       │                              └→ 接收程序→写入Main Flash
       │
       └─ nBOOT1=0 ────────────→ Embedded SRAM（从SRAM跑，掉电丢失）
```

**SRAM 启动有什么用**

SRAM 启动在实际产品里几乎不用，主要用于：

* 调试阶段快速验证代码（不用反复擦写 Flash，Flash 有擦写寿命限制）
* 芯片 Flash 损坏时的应急手段
* 某些特殊的安全启动场景

掉电就丢失，所以每次上电都要重新把代码灌进 SRAM 才能跑，不适合正式产品。

### BootLoader 

芯片厂家在出厂时就固有的程序，只读区。默认它包含一套识别串口/USB 信号的算法，负责接收你的数据并写入 Main Flash。当你使用ST-Link烧录时，可以不需要切换BOOT0启动脚。

烧录程序：
|烧录方式|用什么接口|受BOOT0影响|
|----|----|----|
|ST-Link/SWD|SWDIO+SWCLK| 不受这个BOOT0的=0，=1 影响|
|ISP串口烧录|UART TX/RX|受影响，必须BOOT0切换到System Memory启动Bootloader|

> SWD（ST-Link 烧录）是通过 SWDIO/SWCLK 直接访问芯片的调试接口，不管 BOOT0 是 0 还 
> 是 1，不管芯片在运行还是刚上电，ST-Link 都能连上并烧录。这是硬件调试接口，绕过了启动
> 模式。
> BOOT0 影响的只是上电后 CPU 从哪里开始跑代码，和烧录通道没有任何关系。
{: .prompt-tip }


**启动运行程序及升级程序：**

**情况一：** 需要人手动操作。用 ST 内置的 System Bootloader（BOOT0=1）

```
BOOT0拉高(=1) → 上电进 System Memory → 跑ST的串口Bootloader
→ 通过 UART/USB 接收新程序 → 写入 Main Flash → 拉低BOOT0复位 → 跑新程序
```
每次烧完都需要人把跳帽切到GND。此方式无法进行远程升级。

### OTA
**情况二：** 自己写 Bootloader 放在 Main Flash（真正的 OTA）
```
BOOT0 永远下拉（=0）
Main Flash 分两段：
  ├── 前段：自定义 Bootloader（你写的）
  └── 后段：应用程序
```
上电流程：
```
上电 → 跑自定义 Bootloader → 检查是否有新固件
         ├── 有新固件 → 从网络/蓝牙/SD卡下载 → 写入应用程序区 → 跳转运行
         └── 无新固件 → 直接跳转应用程序区运行
```
BOOT0 全程是 0，System Memory 完全用不到。<font color='red'>可实现远程OTA升级</font>

生产(量产)模式下： BOOT0 常接地，程序在 Main Flash 运行。


#### OTA 升级程序的核心逻辑图

OTA 的本质是：当前运行程序 (App A) 接收新固件并存入暂存区，然后重启进入升级程序 (User Bootloader)用户自定义的Bootloader进行覆盖更新。

**为什么必须是 User Bootloader？（对比分析）**

|特性	|System Bootloader (ST固化)|	User Bootloader (你写的)|
|----|----|----|
|存在位置|	System Memory (只读 ROM 区)|	Main Flash (你的程序区域)|
|执行时机|	当 BOOT0 引脚被拉高（或 BOOT_SEL 配置正确）时|	正常运行（BOOT0=0）时，由你的 App 主动触发跳转|
|功能限制|	非常死板。只支持厂家预定义的串口、USB、CAN 等协议。|	高度灵活。你可以支持 WiFi、蓝牙、LoRa、MQTT、HTTP 协议。|
|应用场景|	芯片出厂时烧录程序，或彻底变砖后的补救。|	OTA 远程升级。需要联网下载、解密、校验固件。|

**实现 OTA 的四个关键要素**

为了让这套程序真正工作，你必须处理好以下细节：

* **双区备份（Double Buffering）：** 不要直接覆盖当前运行的程序！你应该将新固件下载到 Flash 的另一块区域（Slot B），校验无误后再通过 Bootloader 将 Slot B 复制到 Slot A（运行区）。这样即使下载过程中断电，设备也不会变“砖”。
* **中断向量表重映射 (Vector Table Remapping)：** 由于你的主程序不再是从 0x0800 0000 启动，而是从 0x0800 4000 启动，你必须在主程序的 main() 函数开头调用 SCB->VTOR = 0x08004000;，否则中断向量会指向错误的地址。
* **CRC 校验：** 这是防止固件损坏导致设备无法启动的最后防线。在写入 Flash 之前，一定要校验固件的 CRC-32。
* **通信协议：** 你需要定义好如何通过 WiFi/蓝牙接收数据包，并实现断点续传逻辑。

结构概览：Flash 的内存映射

假设你的芯片 Flash 总大小是 128KB（地址范围 0x0800 0000 到 0x0802 0000）：

|区域|	起始地址|	内容|
|----|----|----|
|Bootloader|	0x0800 0000|	开机自启，负责判断是否要升级|
|Application|	0x0800 4000|	你的主业务程序（App）|

你需要将此逻辑放置在 Flash 的最前端（假设地址 0x0800 0000），而你的主程序从 0x0800 4000（举例）开始。

基于 STM32 标准的工程代码架构。核心代码逻辑 (C 语言实现示例)

#### 核心架构规划 (两个独立工程)

|区域名称|	起始地址|	说明|
|----|----|----|
|User Bootloader|	0x0800 0000|	引导程序 (固定)|
|Slot A (Application 运行区)|	0x0800 4000|	正在运行的 App|
|Slot B (下载区)|	0x0801 4000|	OTA 下载的新固件暂存地|


你需要创建两个 Keil 或 STM32CubeIDE 工程：（地址以芯片为准）
代码工程 A (User Bootloader): 烧录在 0x08000000。
代码工程 B (Application): 烧录在 0x08004000。

工程A的 升级跳转模块
ota_user_bootloader.c
```c
#define APP_ADDRESS 0x08004000  // 主程序存放起始地址
#define SLOT_A_ADDRESS  APP_ADDRESS //工程B主程序运行的地址
#define SLOT_B_ADDRESS  0x08014000  //OTA升级包新固件暂存地
#define MAX_APP_SIZE    0x10000    // 假设每个槽位 64KB

typedef void (*pFunction)(void); // 定义函数指针类型

// 1. 跳转到主程序的函数
void JumpToApplication(void) {
    uint32_t jumpAddress = *(__IO uint32_t*)(APP_ADDRESS + 4);
    pFunction jump_to_app = (pFunction)jumpAddress;
    
    // 初始化栈指针
    __set_MSP(*(__IO uint32_t*)APP_ADDRESS);
    jump_to_app();
}

// 2. OTA 升级流程逻辑
void RunOTAUpdate(void) {
    // 检查是否有新的固件标记
    if (CheckForNewFirmware() == SUCCESS) {
        HAL_FLASH_Unlock();

        // 1. 擦除 Slot A (运行区)
        EraseFlashSectors(SLOT_A_ADDRESS, MAX_APP_SIZE);

        // 2. 从 Slot B 拷贝数据到 Slot A
        uint32_t *src = (uint32_t *)SLOT_B_ADDRESS;
        uint32_t *dst = (uint32_t *)SLOT_A_ADDRESS;
        uint32_t size = MAX_APP_SIZE / 4;

        for (uint32_t i = 0; i < size; i++) {
          // 使用 HAL_FLASH_Program 写入每个字
          if (HAL_FLASH_Program(FLASH_TYPEPROGRAM_WORD, (uint32_t)(dst + i), src[i]) != HAL_OK) {
             // 错误处理：烧录失败
             break; 
          }
        }

        HAL_FLASH_Lock();

        // 3. 校验并重启
        if (VerifyCRC(SLOT_A_ADDRESS) == CRC_OK) {
           Clear_OTA_Flag(); // 清除升级标记，防止死循环
           NVIC_SystemReset();
        }
    }
}
```

ota_user_bootloader_main.c
```c
int main(void) {
    HAL_Init(); 

    // --- 逻辑执行流 ---
    
    // 1. 询问：有没有 OTA 任务？
    // 这个函数内部会去查 Flash 的标志位或者检测是否有新的数据
    if (CheckForNewFirmware() == SUCCESS) { 
        // 2. 如果有，执行升级（调用你刚才给的那段代码）
        RunOTAUpdate(); 
    }
    
    // 3. 如果没升级任务，或者升级刚完成，执行跳转
    JumpToApplication(); 

    while (1) { }
}
```

> 工程A：
>    ota_user_bootloader.c
>    ota_user_bootloader_main.c
> 这部分为User Bootloader部份必须烧录到0x08000000
{: .prompt-tip }


工程B的 Application应用，必须烧录到0x08004000

```c
#include "stm32g0xx_hal.h"

int main(void) {
    // 【至关重要】重定位中断向量表到 App 起始地址
    // 否则中断触发时会跳回 0x08000000
    SCB->VTOR = 0x08004000; 

    HAL_Init();
    SystemClock_Config();

    // --- 这里是真正的业务 main 函数 ---
    while (1) {
        // 执行业务逻辑
        // 如果收到 OTA 指令，将固件写入备份区，并调用 NVIC_SystemReset()
    }
}

#OTA 下载的新固件暂存地
#define SLOT_B_ADDRESS 0x08014000

// 在接收中断或处理函数中调用
void On_Data_Received(uint8_t *data, uint16_t length, uint32_t offset) {
    HAL_FLASH_Unlock();
    
    // 将数据写入 Slot B 的对应偏移位置
    // 注意：HAL_FLASH_Program 每次写入 8 字节 (Double Word) 最好
    for (uint16_t i = 0; i < length; i += 8) {
        HAL_FLASH_Program(FLASH_TYPEPROGRAM_DOUBLEWORD, SLOT_B_ADDRESS + offset + i, *(uint64_t*)(data + i));
    }
    
    HAL_FLASH_Lock();
}

// 校验 Slot B 中的数据是否完整
uint8_t Verify_SlotB_CRC(uint32_t total_size, uint32_t expected_crc) {
    uint32_t computed_crc = 0;
    
    // 初始化硬件 CRC
    __HAL_RCC_CRC_CLK_ENABLE();
    
    // 使用 HAL 库的 CRC 计算函数
    // CRC 句柄需要预先在 CubeMX 配置好
    computed_crc = HAL_CRC_Calculate(&hcrc, (uint32_t*)SLOT_B_ADDRESS, total_size / 4);
    
    if (computed_crc == expected_crc) {
        return 1; // 校验通过
    }
    return 0; // 校验失败
}

void On_Download_Finished() {
    // 1. 此时说明数据全部接收完毕
    // 2. 进行最后一次 CRC 校验
    if (Verify_SlotB_CRC(firmware_size, received_crc)) {
        
        // 3. 校验通过，写入标志位 (告诉 Bootloader 有新货)
        // 假设我们在 0x08003FFC 存储 0xAAAA5555
        HAL_FLASH_Unlock();
        HAL_FLASH_Program(FLASH_TYPEPROGRAM_WORD, 0x08003FFC, 0xAAAA5555);
        HAL_FLASH_Lock();
        
        // 4. 重启，交给 Bootloader 去干活
        NVIC_SystemReset();
    } else {
        // 校验失败，要求重发或丢弃
    }
}

```