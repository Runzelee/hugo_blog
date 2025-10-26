---
author: Runze Lee
title: STM32笔记 Day 6 PID
date: 2025-10-25
license: CC 4.0 BY-SA
description: 大一新生STM32学习笔记
image: image/category_stm32_image.jpg
categories: 
     - Tech
     - STM32
tags:
     - 学习笔记
---

在Day5中，我们已经通过CAN实现给电机发送恒定的电压值转起来了，今天我们则是要让他运行到在规定的角速度和角度。在引入PID的概念之前，我们先需要了解一个重要的知识：**要让一个物理量趋于预期值，我们要通过改变其变化率实现**。举个最简单的例子，我想让汽车在50m内停下，一定是降低速度，而速度就是位移的变化率；如果我想降低速度，那我一定想要给他一个负向加速度，这里的加速度就是速度的变化率，而这个负向加速度和我踩的刹车一定是近似成正比的。换句话说，如果位移作为参照，就是**高阶导量操作低阶导量**。

上面举的是平动的例子，旋转运动完全同理，只不过这里的位移变成了角度变化量，速度变成了角速度，而加速度对应很多量（同一个电机全都线性相关）：角加速度、扭矩、力，等等等等。不过这不重要，在GM6020电机上，我们只需知道这里的加速度和输入的电压近似成正比，这就足够了（理论上来说直接调整扭矩的应该是电流，但GM6020自带的电调已经对电流电压进行转换，最终接收CAN通信的是电压值，详见说明书）。在定性了解到高阶导量操作低阶导量后，我们就面临一个巨大的难题，**怎么定量调整高阶导量来让低阶导量达到预期？** 于是，PID（Proportional–integral–derivative controller，比例-积分-微分控制器）就诞生了。

## 数学原理

为了更好理解，我们依然举平动的例子，速度作为高阶导量，位移作为低阶导量。我们现在的目标是导入预期位移，经过某种算法，导出一个合理的速度值。从最最简单能想到的开始，位移越小，我们给的速度就越小，那就先来个正比关系吧：
$$
v = K\Delta x
$$
这里我们知道，预期位移就是目标位置与当前位置的距离差，想让他到目标位置，我们肯定希望这个位移越小越好，因此我们把它称作**误差（error）**，用小e表示，那么用e(t)来表示t时刻的误差。速度是输出值，我们将其记作u(t)，上面的公式就可以写作这样：
$$
u(t) = Ke(t)
$$
误差越大，给的速度越大；误差越小，给的速度越小。这意味着动作总有滞后性，机器完全可能在目标位置之前达到平衡，永远和目标有一段距离无法修正，我们将其称为**稳态误差**。为了修正稳态误差，我们可以把之前所有的误差累计起来，不断补偿长期的偏差，即误差的积分：
$$
u(t) = K_p e(t) + K_i \int e(t)dt
$$
积分和前面的相反，反馈比较慢，越积越多很容易过度补偿，超过目标位置，我们把这种现象称为**超调**，然后误差反向增大，会导致机器来回震荡。这时，我们就需要引入误差的变化率，让其预测未来的变化趋势来抑制超调，即微分：
$$
u(t) = K_p e(t) + K_i \int e(t)dt + K_d \frac{de(t)}{dt}
$$
这就是大名鼎鼎的PID算法的本质。当然，单片机不可能像物理世界那样计算无穷小时间或者连续的量，一定是按照时钟定时采样的离散实现，因此积分就成了过去所有采样点误差的总和，微分就成了当前误差和上次采样误差的平均变化率，于是就有下面的公式（小k指第k次采样，$\Delta t$指的是两次采样之间的时间差）：
$$
u(k) = K_p e(k) + K_i \sum_{i=0}^{k} e(i) \Delta t + K_d \frac{e(k) - e(k-1)}{\Delta t}
$$

## 单环PID

有了上面这个公式，将其变成实际代码就非常容易了：
```c
float integral = 0.0f;
float prev_error = 0.0f;

float PID_Controller(float setpoint, float measurement) {
    float error = setpoint - measurement;
    integral += error * dt;
    float derivative = (error - prev_error) / dt;
    float u = Kp * error + Ki * integral + Kd * derivative;
    prev_error = error;
    return u;
}
```
我们先考虑让电机达到人为设定的角速度`setpoint`，则需通过调整高阶导量角加速度来实现，前面提到过，同一电机角加速度与输入电压近似成正比，那么导出的`u`就应该是我们要的输入电压。`measurement`是当前角速度值，应该通过CAN实时接收得到。其他量与公式中的一一对应，经过计算，即可得出`u`，然后将`u`再通过CAN发送给电机即可，与直接给定电压相反，这种通过算法“内部循环数据”的控制方式称为**闭环控制**。这个`PID_Controller()`函数应该位于TIM时钟中断循环使用或直接放在`main.c`的`while(1)`里，后者则不要忘记`HAL_Delay()`，延迟的时间或中断溢出时间就是这里的`dt`。

需要注意的是`Kp`、`Ki`、`Kd`三个比例系数的调节，这三个量会直接影响电机到目标角速度之间的运动行为，可以写一个串口连接电脑画出图像进行分析。建议先将后面两个赋为0只调试`Kp`，慢慢再扩展到三个量。

## 多环PID

调节完角速度，我们接下来直接调节到目标角度。我们向电机发送的是输入电压（与角加速度近似成正比），能直接改变的是角速度，而非角度。不难想到，如果我们直接让角度参与PID运算，得到的是一个所需要的角速度值；再将这个角速度值参与PID运算，得到的则是我们需要的角加速度值，即电压值。换句话说，这里的输入电压值是角度值的**二阶导**，那我们就需要嵌套两层PID运算，这种多层嵌套称为**多环PID**（双层即双环），与其相对的孤立的一层PID则称为**单环PID**。在双环PID中，角度计算角速度的PID运算称为**位置环**，角速度计算电压的PID运算称为**速度环**。下面是一个最简实现：
```c
float DualLoop_PID(float angle_target, float angle_now, float speed_now) 
{
    // ====== 位置环 ======
    float angle_error = angle_target - angle_now;

    angle_integral += angle_error * dt;
    float angle_derivative = (angle_error - angle_prev_error) / dt;

    float speed_target = Kp_angle * angle_error
                       + Ki_angle * angle_integral
                       + Kd_angle * angle_derivative;

    angle_prev_error = angle_error;


    // ====== 速度环 ======
    float speed_error = speed_target - speed_now;

    speed_integral += speed_error * dt;
    float speed_derivative = (speed_error - speed_prev_error) / dt;

    float control_output = Kp_speed * speed_error
                         + Ki_speed * speed_integral
                         + Kd_speed * speed_derivative;

    speed_prev_error = speed_error;


    // 最终输入电机的电压值
    return control_output;
}
```
我们可以看到`speed_target`同时作为位置环的输出值和速度环的输入值，这是一个完美的双重PID调用链，有两组`Kp`、`Ki`、`Kd`共六个比例系数需要调节。代码具体逻辑和先前同理，不多赘述。

## 善后工作

### 输出限幅

输出限幅就是防止算法得到的输入电压值超出电机的合法区间。GM6020说明书给出的电压值范围是[-25000,25000]，需要做简单的限制：
```c
float Limit(float x)
{
    if(x > 25000) return 25000;
    if(x < -25000) return -25000;
    return x;
}
```

### 积分限幅与积分分离

当算法长时间存在误差，电压达到最大后，积分项会始终累计误差，导致其数值越来越大变得不可控，这种情况称为Windup（积分过量）。我们可以通过两种手段对抗Windup，其一是直接进行**积分限幅**，和上面提到的输出限幅完全一致，只不过对象变成了积分项；其二是进行**积分分离**，即误差大于一定阈值后禁用积分，防止后续超调：
```c
if(fabsf(error) < error_threshold) {
     integral += error * dt;
}
```
这里的`fabsf()`函数用于给`float`类型取绝对值。
### 死区

当电机到目标角度值后，很容易超调一点点微小量，又会带动反向控制，然后又超调微小量，无限循环，造成严重的来回抖动。要解决这个问题很简单，就是当误差值小于一定阈值后就停止响应，这个停止响应的区间就称为**死区**：
```c
float DeadZone(float x, float zone)
{
    if(fabsf(x) < zone) return 0;
    return x;
}
```
### 多圈转动角度解算
GM6020的角度值是[0,8191]的绝对刻度，如果我们旋转多圈，则需要把单圈绝对角度值转换成多圈的连续角度值：
```c
float Angle_Unwrap_Encoder(uint16_t new_angle)
{
    static uint16_t last_angle = 0;
    static int32_t multi_turn_count = 0; // 圈数计数

    int32_t diff = (int32_t)new_angle - (int32_t)last_angle;

    if(diff > 4096)
        multi_turn_count--;
    else if(diff < -4096)
        multi_turn_count++;

    last_angle = new_angle;

    float continuous_angle = multi_turn_count * 8192.0f + new_angle;
    return continuous_angle; // 连续角度
}
```
`diff`就是相对上次采样的角度变化量，角度变化一旦超过-4096，说明其向上跨越了零点，到了上一圈；一旦超过+4096，说明向下跨越了零点，到了下一圈。设置简单的计数器`multi_turn_count`就可以判断出连续旋转的圈数，从而得到连续角度值。相反，连续角度值取余也可以得到旋转圈数和角度，不多赘述。