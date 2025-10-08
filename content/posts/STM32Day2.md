---
author: Runze Lee
title: STM32笔记 Day 2 UART+ADC
date: 2025-10-08
license: CC 4.0 BY-SA
description: 大一新生STM32学习笔记
image: image/category_stm32_image.jpg
categories: 
     - Tech
     - STM32
tags:
     - 学习笔记
---

## UART

UART是USART（Universal Synchronous/Asynchronous Receiver/Transmitter， 通用同步/异步收发传输器）的异步（Asynchronous）模式。异步意味着不需要时钟线同步，只依赖发送端和接收端约定的相同波特率（Baud Rate）通信，其速度可能慢于同步，但简单易用，因此在日常编程中使用得更多。

### 接线

要让电脑和单片机通过UART串口通信，就需要USB-TTL（Transistor-Transistor Logic，晶体管-晶体管逻辑）转换器，即把电脑USB信号转换成单片机可以理解的高低电平，Windows端最常用的是CH340转换器。

在STM32上，USART1_TX与`PA9`复用，USART1_RX与`PA10`复用；USART2_TX与`PA2`复用，USART2_RX与`PA3`复用。这里的TX指的是发送脚，RX指的接收脚，以USART1为例：

>  STM32 ──────── CH340 <br>
  PA9  ───────> RX <br>
  PA10 <─────── TX <br>
  GND ───────── GND 

接收脚与发送脚交叉相接，`GND`脚共地，USB口接在电脑上，设置相同波特率，安装驱动和软件即可使用UART监听和发送数据给单片机。

### 标准接口

UART收发分为阻塞和非阻塞两种模式，前者是收发完数据才继续执行后续逻辑，后者则是和后续逻辑同步执行，但提供了操作完成的回调。

#### 阻塞
```c
HAL_StatusTypeDef HAL_UART_Receive(
    UART_HandleTypeDef *huart,  // &huart1/&huart2
    uint8_t *pData,             // 数据指针
    uint16_t Size,              // 数据长度
    uint32_t Timeout            // 超时时间（ms）
);

HAL_StatusTypeDef HAL_UART_Transmit(...);
```
#### 非阻塞
```c
HAL_StatusTypeDef HAL_UART_Receive_IT(
    UART_HandleTypeDef *huart,  // &huart1/&huart2
    uint8_t *pData,             // 数据指针
    uint16_t Size               // 数据长度
);

// 接收完成回调，以USART2为例
void HAL_UART_RxCpltCallback(UART_HandleTypeDef *huart) {
    if (huart->Instance == USART2) {
        // Do something
        if (pData == 0xAA) { // 假如pData是接收数据的变量
            // Do something
        }
    }
}

HAL_StatusTypeDef HAL_UART_Transmit_IT(...);
void HAL_UART_TxCpltCallback(UART_HandleTypeDef *huart) {...}
```

#### 简单案例

以阻塞方式发送字符串`Tx_str1`给USART1，10秒后超时。
```c
uint8_t Tx_str1[] = "hello world!\r\n";
HAL_UART_Transmit(&huart1, Tx_str1, sizeof(Tx_str1), 10000);
```
#### 其他注意事项

实际使用中，一般使用阻塞方式发送数据，用非阻塞方式接收数据。如果需要始终监听数据触发多次回调，则需要在回调最后重新调用`HAL_UART_Receive_IT()`，否则需要在`while(1)`中调用`HAL_UART_Receive_IT()`，前一种方案更节省资源。所有UART相关函数调用都必须放在机器生成的`MX_USART1_UART_Init()`之后（USART2同理）。

对于一些特殊的情况，我们发送的字符串需要用和数字拼接，这个时候我们可以使用`sprintf()`函数，需引入`<stdio.h>`。例如，将时分秒数据填入`str_buff`字符串：
```c
uint8_t str_buff[64];
sprintf((char *)str_buff, "%d:%d:%d LED2 is ON!\r\n", hh, mm, ss);
```
`HAL_UART_Transmit()`的发送数据参数需要`uint8_t *`，但`sprintf()`只接受`char *`，故需显式转型。


## ADC

ADC（Analog-to-Digital Converter， 模数转换器）是将自然界连续的模拟信号转换为机器可以理解的离散的数字信号的设备，其过程包括采样、保持、量化、编码，有多种实现方案。对于STM32开发，我们只需了解几个基本技术指标：

> 参考电压：指ADC所能输入模拟信号的电压范围，即量程。 <br>
 转换位数：转换后的输出结果的二进制指数范围，例如，10位ADC的输出值范围2^10=1024，即0~1023。<br>
 分辨率：ADC能够分辨的模拟信号最小变化量，可由`参考电压/2^转换位数`计算，例如，参考电压为5V（5000mV）的8位ADC的分辨率为5000/256=19.5mV。

需要记住的公式为：`模拟电压=采样值*分辨率=采样值*参考电压/2^转换位数`。模拟电压可以理解为最小变化量（分辨率）的整数倍，这个倍数就是采样值。例如参考电压为3.3V（3300mV）的12位ADC采样值为145，其模拟电压则为145*3300/2^12=116.8mV。

STM32中有ADC1、ADC2、ADC3共3个12位ADC，具有18个测量通道，可测量16个外部和2个内部信号源（内部温度和内部电压）。这2个内部信号源只能连接到ADC1，今天则以位于`PA0`脚的ADC1_IN0为例。

### 接口与转换

和UART一样，ADC读取也分为阻塞式和非阻塞式。ADC1_IN0是参考电压为3.3V（3300mV）的12位ADC，我们将采样值放入`ADC_Value`，转换得到的模拟电压放入`ADC_Volt`：

#### 阻塞

```c
uint16_t ADC_Value = 0, ADC_Volt = 0;
HAL_ADC_Start(&hadc1);

if (HAL_ADC_PollForConversion(&hadc1, 100) == HAL_OK) {  // 100ms超时
    ADC_Value = HAL_ADC_GetValue(&hadc1); 
    ADC_Volt = ADC_Value * 3300 / 4096;
}

HAL_ADC_Stop(&hadc1);
```

#### 非阻塞

```c
uint16_t ADC_Value = 0, ADC_Volt = 0;
HAL_ADC_Start_IT(&hadc1);

void HAL_ADC_ConvCpltCallback(ADC_HandleTypeDef* hadc) {
    if (hadc->Instance == ADC1) {
        ADC_Value = HAL_ADC_GetValue(&hadc1);
        ADC_Volt = ADC_Value * 3300 / 4096;
    }
}
```