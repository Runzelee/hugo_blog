---
author: Runze Lee
title: STM32笔记 Day 4 PWM
date: 2025-10-12
license: CC 4.0 BY-SA
description: 大一新生STM32学习笔记
image: image/category_stm32_image.jpg
categories: 
     - Tech
     - STM32
tags:
     - 学习笔记
---

在引入PWM的概念之前，我们先需要了解的是：**人眼就是一个超级积分器。**

人眼看到光，是光子打到视网膜上，激发细胞内的视紫红质分子，从而触发一连串化学反应，最终导致离子通道关闭、膜电位变化，产生大脑能理解的电信号。然而这个化学反应的过程，我们称为光信号转导通路，**往往需要几十到上百毫秒的时间才能完成**，绝不是瞬时的。这短时间内，人眼无法分辨强烈的光线闪烁，相反如果又有新的光刺激进来，反应只会叠加，大脑就会被欺骗，感觉更亮了。举个简单的例子，50ms里，第一种情况10ms灯暗40ms灯亮，第二种情况20ms灯暗30ms灯亮，即便灯的实际亮度是一致的，大脑也会认为前者比后者更亮，并且不会察觉到闪烁。所以，在极短时间内，我们看到的这个世界不是这个世界本来的样子，而是**光线强度对于时间的积分**，是加权平均后的平滑结果。

我们作为那个点灯的人，只想用单片机的能理解的高低电平来“调整灯的亮度”，我们就能通过这种欺骗大脑快速闪烁的方式来曲线救国，因此，PWM（Pulse Width Modulation，脉冲宽度调制）就诞生了。其实我对PWM并不陌生，在接触单片机前，选购手机也会看看屏幕是PWM调光还是DC（Direct Current，直流电）调光，前者就是通过这种骗大脑的频闪方式来调整亮度，而后者则是直接改变电流大小来调整，虽然对眼睛更友好，但成本高，电路驱动更复杂，买台性价比中端机，PWM就可以了。

## PWM相关基本概念

### 占空比

换个角度思考，与ADC恰恰相反，PWM是将数字信号转为模拟信号的一种方法。如果我们需要使用PWM控制LED亮度，LED引脚电平输出则是代表不断闪烁的方波，如下图：

![PWM方波](image/stm32_day4_1.webp)

上面有三个方波，区别在于每个周期内维持高电平的时间占比不同，我们将这个占比称为**占空比**（Duty Cycle）。占空比直接决定了人眼感知的LED亮度，高电平电压（即单片机VCC电压，STM32上是3.3V）乘以占空比就是人眼感知的模拟电压。例如，50%的占空比，模拟电压就是3.3*50%=1.65V，其与直接拉高1.65V呈现的LED亮度是一致的。（直接决定LED亮度的实际上是电流而非电压，但PWM极高频情况下电阻变化可忽略不计。）

### Prescaler、Counter Period、PWM模式、CCR

Prescaler和Counter Period我们在Day1的基本定时器那里已经接触过了，CNT（Counter Register，计数寄存器）每过Prescaler个时钟周期加一，直到Counter Period设定值触发中断回调。在PWM中，Prescaler和Counter Period的计数逻辑依然是这样，但唯一的差异CNT数到Counter Period后不再触发回调了，而是**CNT从0数到Counter Period的过程本身就定义为一个PWM方波周期**。

例如，时钟频率为72MHz，我们需要配置一个PWM频率为10KHz，根据我们之前已经掌握的公式：
> 定时时间=(Prescaler+1)\*(Counter Period+1)\*时钟周期

这里的定时时间就是我们要的PWM周期，为了方便计算，我们全部转换为频率：
> PWM频率=时钟频率/((Prescaler+1)\*(Counter Period+1))

那么(Prescaler+1)\*(Counter Period+1)就是时钟频率除以PWM频率，即72MHz/10KHz=72000KHz/10KHz=7200。不妨设Prescaler为72-1=71，Counter Period为100-1=99，即可满足条件。

配置完PWM周期后，我们就可以开始设置一个周期内哪个时间点开始翻转电平了。PWM提供两个模式，分别是模式1和模式2，前者是一个周期内由高电平转向低电平，另一个则反之，下面以模式1为例。我们的Counter Period设置的是99，意味着CNT从0数到99作为一个周期，因此，如果我们需要占空比50%，那我们只要让CNT数50开始从高电平翻向低电平即可。这个“50”分界线就是CCR（Compare Register，比较寄存器），在STM32CubeMX中称为Pulse（脉冲值）。

## 简单使用

在STM32中，PWM在TIM3中可用，其分为Channel 1、Channel 2和Channel 3，分别对应`PA6`、`PA7`、`PB0`三个引脚，今天以Channel 2为例。
首先启动PWM：
```c
HAL_TIM_PWM_Start(&htim3, TIM_CHANNEL_2);
```
此时是恒定占空比PWM，我仔细看`PA7`的LED，相比常亮LED，实际上眼睛还是有点不太舒服，因为PWM本来就不常用于显示恒定光。下面每过10ms改变占空比实现呼吸灯：
```c
for (int i = 0; i < 100; i++) {
	__HAL_TIM_SET_COMPARE(&htim3, TIM_CHANNEL_2, i);
	HAL_Delay(10);
}
for (int i = 100; i >= 0; i--) {
	__HAL_TIM_SET_COMPARE(&htim3, TIM_CHANNEL_2, i);
	HAL_Delay(10);
}
```
`__HAL_TIM_SET_COMPARE()`函数负责修改CCR的值，逻辑很简单，整体写在`while(1)`循环里效果更好。