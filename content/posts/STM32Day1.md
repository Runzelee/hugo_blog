---
author: Runze Lee
title: STM32笔记 Day 1 Helloworld点灯+简单轮询按键+外部中断按键+基本定时器
date: 2025-10-05
license: CC 4.0 BY-SA
description: 大一新生STM32学习笔记
image: image/category_stm32_image.jpg
categories: 
     - Tech
     - STM32
tags:
     - 学习笔记
---

大一新生，才疏学浅，若有疏漏，请多指教。

## Helloworld点灯

毋庸置疑，单片机界的Helloworld就是点亮一颗朴素的LED灯。

### 接线

软件安装就不说了，主要说点容易搞错的，首先最麻烦的就是接线。我买的STM32F103C8T6最小系统板，需要插在面包板上调试。

![图1](image/stm32_day1_1.png)

如图我们可以看到，面包板分成两个区域，中间的Circuit Area和两侧的Rail。Circuit Area每行为通路，Rail一整纵条为通路，下面有金属片连接。Rail又分为正极电源轨（Power Rail）和负极电源轨（Ground Rail），为了方便地用家庭电路做比喻，我暂且称之为火线和地线。

GPIO（General Purpose Input/Output， 通用输入/输出口）可以向外输出高低电平信号，今天点灯用的是其中的推挽输出，支持两种接线方式：上拉和下拉。

假如我们现在需要把LED接到`PB8`引脚上，先不考虑串电阻。

> 上拉法，即PB8->LED长脚(阳极)->LED短脚(阴极)->GND <br>
> 下拉法，即3.3V->LED长脚(阳极)->LED短脚(阴极)->PB8

`3.3V`脚电势最高，`GND`脚电势最低。我们写代码控制的是PB8引脚的电平，上拉法拉高电平会和`GND`脚产生电势差，下拉法拉低电平则和`3.3V`脚产生电势差，从而控制LED灯的亮与灭。因此，两种接法导致最终电脑控制电平即`GPIO_PIN_SET`和`GPIO_PIN_RESET`的结果也是相反的。我一开始就用了下拉法，自己捣鼓半天才弄回来。

一般情况下，我们肯定选择上拉法，也符合直觉，但我们需要**保留两种接法的权利**。怎么做呢，就是把`3.3V`脚始终接在火线上，并把`GND`脚始终接在地线上。那么以后我不管要接什么负载，只要一边接在单片机引脚同一行，一边接在火线（下拉法）/地线（上拉法）上，都能形成通路。

### 点灯

接完线，点灯就很容易了。HAL（Hardware Abstraction Layer， 硬件抽象层）主要提供两个接口来实现点灯：
```c
void HAL_GPIO_TogglePin(GPIO_TypeDef *GPIOx, uint16_t GPIO_Pin)
```
```c
void HAL_GPIO_WritePin(GPIO_TypeDef *GPIOx, uint16_t GPIO_Pin, GPIO_PinState PinState)
```
前者是电平翻转，即亮变暗暗变亮，后者则是指定高低电平。其中前两个参数定位引脚，比如`PB8`就是`GPIOB, GPIO_PIN_8`，后面的`GPIO_PinState`是一个`enum`，包括`GPIO_PIN_SET`和`GPIO_PIN_RESET`，分别代表高低电平。

## 简单轮询按键

轮询按键顾名思义，就是通过反复读取引脚电平来判断是否按下按键，从而触发别的逻辑。

### 坑

这里尤其注意，我遇到一个大坑，就是按键引脚一定要设置默认电平（Pull-up/down）。否则，引脚电平将会处于悬空（floating）状态，其电平会由于外界微小的变化比如电磁波、静电、人体电容而改变，我刚开始连按键根本没按下去，手轻轻放在按键上面灯都疯狂闪烁，就是这个原因。

### 样例实现

这是基本代码：
```c
void Delay(unsigned int t){
    while(t--);
}

void Scan_Keys(){
    if(HAL_GPIO_ReadPin(GPIOC, GPIO_PIN_13) == GPIO_PIN_RESET){
        Delay(1000);
        if(HAL_GPIO_ReadPin(GPIOC, GPIO_PIN_13) == GPIO_PIN_RESET){
            HAL_GPIO_TogglePin(GPIOB, GPIO_PIN_9);
            while(HAL_GPIO_ReadPin(GPIOC, GPIO_PIN_13) == GPIO_PIN_RESET);
        }
    }
}
```
逻辑很简单，`Delay()`函数只是简单计数让单片机阻塞很短的时间，两次检验用于判断电平变化能不能被看作是人类按下了按键产生的，这叫做防抖。接着触发`PB9`脚翻转电平点灯，最后一个`while`等待到回高电平（松手）后才结束防止按下频繁闪烁。这里可以用宏定义省点字，**`Scan_Keys()`必须在预置的`while(1)`循环里调用才能保证长时间轮询**。

## 外部中断按键

STM32官方提供了中断接口和回调，避免轮询监听浪费资源。其中， EXTI (EXTernal Interrupt， 外部中断) 专门用于监听GPIO输入口的电平变化。STM32有16个外部中断源（EXTI0-EXTI15），一个中断源比如EXTI0负责`PA0`-`PG0`所有引脚的监听。16个外部中断源（即所有的引脚）仅对应7个中断服务函数。

> EXTI0、EXTI1、EXTI2、EXTI3、EXT4：专用。<br>
> EXTI5-EXT19：共用。<br>
> EXTI10-EXTI15：共用。<br>


不过使用HAL库，我们**完全不需要关心中断服务函数的事情**，因为STM32CubeMX早都帮我们处理好了。我们需要注意的是EXTI的共用情况，例如`PA1`和`PB1`脚不能共用，因为回调无法区分是其中哪个引脚电平在发生变化。

其回调使用很简单，比如我需要监听`PA1`和`PB8`引脚电平变化：
```c
void HAL_GPIO_EXTI_Callback(uint16_t GPIO_Pin) {
    if (GPIO_Pin == GPIO_PIN_1) {
        // Do something
    }
    else if (GPIO_Pin == GPIO_PIN_8) {
        // Do something
    }
}
```
还有一个小小的点，就是EXTI监听分为上升沿（Rising edge）、下降沿（Falling edge）、双边沿（Rising + Falling edge），分别对应监听电平上升、下降和所有，双边沿会导致按一次键触发两次回调。

## 基本定时器

STM32基本定时器由可编程预分频器 (Prescaler) 和主计数器 (Counter Period) 构成。Prescaler为一个0-65535之间的计数器，每过一个时钟周期加一，直到设定的阈值则会给Counter Period发一个脉冲信号。因此，定时器计时可以这样计算：
> 定时时间=(Prescaler+1)\*(Counter Period+1)\*时钟周期

例如，时钟信号1KHz，其周期就是1ms，Prescaler为9，Counter Period为999，定时时间就是1000\*10\*1=10000ms=10s。

在STM32中，基本定时器在TIM2、TIM3、TIM4、TIM5中可用，在设置好Prescaler和Counter Period值后，可以直接使用HAL提供的定时器溢出回调，比如TIM2计时器：
```c
void HAL_TIM_PeriodElapsedCallback(TIM_HandleTypeDef *htim) {
    if (htim->Instance == TIM2) {
        // Do something
    }
}
```
然后不要忘记在机器生成的`MX_TIM2_Init();`后写上`HAL_TIM_Base_Start_IT(&htim2);`，回调则会以定时时间为周期不断循环触发。

第一天收获颇丰，文章同步在[CSDN](https://blog.csdn.net/qq_29334675/article/details/152557309)。

