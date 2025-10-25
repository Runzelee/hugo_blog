---
author: Runze Lee
title: STM32笔记 Day 5 CAN
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

作为RM战队的新人，前几天比较赶进度，忙着上手了DJI电机，就没怎么写笔记。今天终于有时间，我们则以GM6020电机作为出发点，初步讲讲CAN通信、PID单环与双环控制，今天先讲CAN。

我们之前接触过UART、SPI、I²C协议，其中UART用于一对一通信，而SPI和I²C都是一主多从。其本质上就是主机发起对话，从机回答，就像私信聊天，协调效率并不高。1980年代初，汽车电子化刚刚起步，发动机、变速箱、安全气囊、ABS等等模块都需要实时相互通信，这种“私信聊天”最终会导致一台汽车变成一个“电线森林”，极难维护。于是，汽车工程师们提出一个想法，能不能**让所有模块共用一条通信线，谁有消息就在这条线上广播，别人则按需接收**，此时，CAN（Controller Area Network，控制器局域网）就诞生了。由于CAN抗干扰、便宜、安全可靠，其被扩展应用到工业自动化、医疗、船舶等等领域，成为现代电子社会的一大基石。

CAN通信的知识体系非常庞杂，我们今天的目标是先通过CAN给电机发送恒定的电压指令让其转起来，只会提及那些实际使用不可绕过的核心概念。因此，这篇笔记使用一个与以往相反的方式，就是从接口调用出发来解释概念快速上手。

## 接线和模式

前面提到，CAN有**让所有模块共用一条通信线**的特点，这条通信线就是CAN总线，分为CAN_H和CAN_L，其使用**差分信号**通信，即CAN_H和CAN_L两根线电压一高一低作为逻辑0，相同则作为逻辑1，如果有干扰两根线也同步被干扰，相对逻辑不变，可以提升信号质量。两根总线使用两个120Ω的电阻相连接形成回路，不同设备则连接到总线上：

![CAN总线](image/stm32_day5_2.jpg)

实践中我们连接GM6020电机只需将CAN_H和CAN_L两根线分别连接板子和电机的CAN_H和CAN_L脚即可。理论上来说STM32本身只有CAN_RX和CAN_TX脚，要连接到CAN总线中间需要CAN收发器芯片，将单片机能识别的高低电平转换成差分信号。 ~~（学长设计的板子已经集成好了）~~

CAN通信有四种模式，常规模式、静默模式、回环模式、静默回环模式，静默为向总线只收不发，回环是向总线只发不收，静默回环则是彻底与总线隔离，自收自发，要与电机通信，常规模式即可。

## 时钟配置

CAN通信中一个比特位的数据传输的时间固定，这一小段时间被分为同步段、传播段、相位缓冲段1、相位缓冲段2四个部分。这里一个`时钟周期/Prescaler`的时间段称为一个TQ（Time Quanta, 时间量子）。

![比特时间](image/stm32_day5_1.png)

其中，同步段用于通信对齐，时间恒占一个TQ；传播段用于消耗信号物理传输的时间；两个相位缓冲段用于等待信号稳定，相当于容错缓冲；而夹在两个相位缓冲段之间的便是时间采样点，这个时间点就用于**观测这个比特位是1还是0**。传播段与相位缓冲段1合称为BS1（Bit Segment 1，比特段1），相位缓冲段2称为BS2（Bit Segment 2，比特段2），BS1和BS2所占的TQ数量可以在STM32CubeMX中调整，因此一个比特位的完整时间就是`(1+BS1+BS2)*TQ`，而波特率就是每秒能传输多少个比特位，即`1/((1+BS1+BS2)*TQ)`，单位是bps，和Hz同量纲（1/s）。

例如，时钟频率是42MHz，GM6020需要的波特率是1Mbps。我们依然把`波特率=1/((1+BS1+BS2)*TQ)`和`TQ=时钟周期/Prescaler`全部转换成频率来表示，则有：

> 波特率=时钟频率/((1+BS1+BS2)*Prescaler)

不妨设Prescaler为3，那么1+BS1+BS2=时钟频率/(Prescaler*波特率)=42MHz/(3\*1Mbps)=14。在分配BS1和BS2的值时，我们建议将前者分配大一点，目的在于将时间采样点偏后移来保证信号更稳定。这里我们可以分配BS1=9，BS2=4，即可满足条件。

另外还有两点，其一是分配一个比特位时间`(1+BS1+BS2)*TQ`时，不能太短，否则信号来不及传输或稳定，STM32CubeMX会报错。其二时注意单片机的外部晶振，我们学校的STM32F405RGT6板子只支持8MHz的外部输入频率，这个问题困扰了很久才发现。

## 发送与接收

### 发送数据

```c
void Set_GM6020_Voltage(int16_t vol)
{
     uint8_t TxData[8] = {0}; 
     TxData[2] = (uint8_t)(vol>>8);
     TxData[3] = (uint8_t)vol;
     CAN_TxHeaderTypeDef TxHeader = {
          .DLC = 8,
          .IDE = CAN_ID_STD,    
          .RTR = CAN_RTR_DATA,
          .StdId = 0x1FF
     };
     uint32_t TxBox = CAN_TX_MAILBOX0;
     HAL_CAN_AddTxMessage(&hcan1, &TxHeader, TxData, &TxBox);
}
```
这个函数是一个给GM6020电机发送电压的最简实现。显而易见，一个8字节的数组报文`TxData`的索引2和索引3的位置分别被赋值了`vol`的高八位和低八位。根据[GM6020的说明书](https://rm-static.djicdn.com/tem/17348/RoboMaster%20GM6020%E7%9B%B4%E6%B5%81%E6%97%A0%E5%88%B7%E7%94%B5%E6%9C%BA%E4%BD%BF%E7%94%A8%E8%AF%B4%E6%98%8E20231013.pdf)，一条指令最多能控制4个电机，索引2和索引3则是ID为2的电机，电机ID（1-4）在底部开关处可调整。

`CAN_TxHeaderTypeDef`结构体则用于处理一些CAN通信的基本设置，称为**帧头**，`DLC`（Data Length Code，数据长度代码）用于指定报文的字节数，这里显然是8字节；`IDE`（Identifier Extension，标识符扩展位）用于指定数据帧类型，其分为标准帧和扩展帧，二者报文长度和格式均不同，GM6020使用标准帧，因此为`CAN_ID_STD`；`RTR`（Remote Transmission Request，远程传输请求），其分为数据帧和远程帧，数据帧是直接向对方发送数据，远程帧是要求对方发送数据，这里显然选择数据帧`CAN_RTR_DATA`；`StdId`则是指定对应的标准帧CAN ID，这是设备的**唯一识别码**，GM6020的说明书中规定使用电压控制为`0x1FF`。需要注意的是，标准帧的CAN ID最多为11位，因此范围是`0-0x7FF`。

接下来是CAN的邮箱机制。为了防止数据冲突丢帧，CAN设计了三个**发送邮箱**，数据不会直接推上总线，而是以邮箱作为载体发送，分为`CAN_TX_MAILBOX0`、`CAN_TX_MAILBOX1`、`CAN_TX_MAILBOX2`。这里选择第一个作为**优先**邮箱，如果第一个邮箱正在被占用，则会自动选择其他邮箱等待发送。最后`HAL_CAN_AddTxMessage()`函数则是正式的发送接口，传入所有上述参数，不多赘述。

### 接收数据

#### 滤波器

```c
void Configure_Filter(void)
{
     CAN_FilterTypeDef CAN_Filter = {
          .FilterFIFOAssignment = CAN_FILTER_FIFO0,
          .FilterScale = CAN_FILTERSCALE_32BIT,
          .FilterBank = 0,
          .FilterMode = CAN_FILTERMODE_IDMASK,
          .SlaveStartFilterBank = 0,
          .FilterActivation = CAN_FILTER_ENABLE,
          .FilterIdHigh =  (0x1FF << 5) & 0xFFFF,
          .FilterIdLow =   0x0000,
          .FilterMaskIdHigh = (0x7FF << 5) & 0xFFFF,
          .FilterMaskIdLow =  0x0000
     };
    HAL_CAN_ConfigFilter(&hcan1, &CAN_Filter);
}

```
单片机暴露在CAN总线上，理论上可以接受总线上的所有报文，但我们一般只希望看到我们需要的报文，这时滤波器就起到作用了。和发送时配置帧头类似，滤波器使用`CAN_FilterTypeDef`结构体进行配置。

首先是FIFO（First In First Out，先入先出队列），其与发送端的邮箱机制类似，也是一种队列处理机制。CAN提供了两个FIFO，分别为`FIFO0`和`FIFO1`，每个FIFO可以存储3条报文，按顺序接收，超过3条可能会被丢弃。这里的`FilterFIFOAssignment`就是选择对接滤波器的FIFO，这里选择`CAN_FILTER_FIFO0`。

然后从几个简单的入手，`FilterBank`指的是滤波器的编号，我使用的STM32F405RGT6共有14个滤波器（0-13），分别可以分配给CAN1和CAN2，上面选择第一个滤波器，即为0；`SlaveStartFilterBank`指从第几个滤波器开始分配给CAN2，上面的代码根本没有启用CAN2，因此可以直接设置为0；`FilterActivation`指的是是否启用该滤波器，`CAN_FILTER_ENABLE`即为启用；`FilterMode`指的是滤波器模式，分为`CAN_FILTERMODE_IDLIST`（列表模式）和`CAN_FILTERMODE_IDMASK`（掩码模式），列表模式是只有当所有位明确匹配滤波器ID时才通过，而掩码模式是指定特定几位匹配滤波器ID就可以通过，后者显然更灵活也更常用。

我们先来看看掩码模式中是怎么实现“匹配ID的特定几位”的。这是掩码模式匹配通过的充要条件：

> (接收到的ID & 掩码) == (滤波器ID & 掩码)

这个公式中，我们看到一个新的运算`&`，称为**按位与**。其定义是：两个长度相同的二进制数，两个相应的位都为1，该位的结果值才为1，否则为0。例如：

```
    0101
  & 0011
  = 0001
```
我们可以发现，这个运算达到了一个巧妙的效果，`0101`任何一位旦碰到1，就能保留原值，一旦碰到0就变成0，换句话说，这些碰到0的位被**遮掩**了。`0011`的作用就是相当于告诉`0101`，我只想知道你的后两位是几，前两位并不关心，这时，`0011`就起到了一个筛选的作用，我们将其称为**掩码**。

再回到之前那个公式，`接收到的ID & 掩码`就相当于接收到的ID筛选过的特定几位，那么`滤波器ID & 掩码`就相当于滤波器ID筛选过的特定几位，翻译成人话，就是我想让你的ID里面特定几位和我的相同，我就让你进FIFO。

了解了掩码的原理，我们就可以分析上述的“ID”本身了。这里的ID并不是前面提到的CAN ID，而是完整的32位寄存器ID，标准帧的寄存器ID构成是这样的：
| 31 ~ 21位 | 第20位 | 第19位 | 18 ~ 0位 |
|--------------|--------|--------|--------------|
| CAN ID       | IDE    | RTR    | 保留为 0     |

滤波器在匹配寄存器ID中，把一个ID一劈分成两半，即高16位和低16位，在标准帧的情况下低16位都是0，因此我们只需关心高16位：

| 16 ~ 6位 | 第5位 | 第4位 | 3 ~ 0位 |
|--------------|--------|--------|--------------|
| CAN ID       | IDE    | RTR    | 保留为 0     |

比如我们要匹配CAN ID为`0x1FF`，首先要给出掩码，我们只关心CAN ID，不关心IDE和RTR，因此应该是一个6-16位都是1，0-5位都是0的二进制数。我们可以将11位全1`0x7FF`抬高5位，正好就可以放在高16位的6-16位的位置，即`0x7FF << 5 = 0xFFE0`。接下来给出滤波器ID，同理，我们依然只需给出CAN ID位置上的匹配目标，即`0x1FF`，将其抬高5位即可，`0x1FF << 5 = 0x3FE0`。处理完这些，还有最后一步，就是需要保证给出的两个高16位ID都真的只有16位，不能多位或少位，这里有一个技巧，就是将其与16位全1`0xFFFF`进行按位与，这个操作不会改变有效16位中的任何值，但可以**截断其他位**。这样一来，我们就有了`FilterMaskIdHigh = (0x7FF << 5) & 0xFFFF`和`FilterIdHigh =  (0x1FF << 5) & 0xFFFF`。

前面提到，低16位全部保留为0，因此`FilterMaskIdLow`和`FilterIdLow`全部赋为0即可。读者在这里可能有疑惑，都是0还设置这样两个变量有什么用。其实很简单，因为上面所述的所有ID格式都是标准帧格式，扩展帧格式则不同，这里不做展开，但有一些有效量会放在低16位。就标准帧情况下，我们不难看出，ID低16位的匹配机制被**浪费**了，因此CAN便提供了第二种匹配机制，即`CAN_FILTERSCALE_16BIT`，可以赋给`FilterScale`启用。这种16位匹配机制会将`FilterMaskIdLow`和`FilterIdLow`用于映射匹配第二个ID的高16位，换句话说，就是一个滤波器可以用于匹配两个ID，效率更高。缺点是，当标准帧和扩展帧同时出现，很有可能误匹配，与其相反`CAN_FILTERSCALE_32BIT`则是32位每一位都精确匹配，就是以上所述的默认情况。

全部配置完成后，调用最终的`HAL_CAN_ConfigFilter()`传入结构体，即可大功告成。

#### 接收

```c
void HAL_CAN_RxFifo0MsgPendingCallback(CAN_HandleTypeDef* hcan)
{
  if (hcan == &hcan1)
  {
      CAN_RxHeaderTypeDef RxHeader;
      uint8_t RxData[8];
      if (HAL_CAN_GetRxMessage(hcan, CAN_RX_FIFO0, &RxHeader, RxData) == HAL_OK) 
      {
          uint16_t raw_angle = (uint16_t)(((uint16_t)RxData[0] << 8) | (uint16_t)RxData[1]);
          raw_angle &= 0x1FFF; 

          int16_t raw_vel = (int16_t)(((uint16_t)RxData[2] << 8) | (uint16_t)RxData[3]);
          float velocity = (float)raw_vel;
          
          // Do something
      }
  }
}
```
上面的代码就很简单了，实现了从GM6020读取角度和速度值。`HAL_CAN_RxFifo0MsgPendingCallback()`时FIFO0的接收回调，`HAL_CAN_GetRxMessage()`则拿回对方发送的帧头结构体和数据内容。根据GM6020的说明书，角度放在索引0和索引1的位置，范围是0-8191绝对刻度（即2^13，13位二进制数），将其与`0x1FFF`按位与目的也在于截出13位有效值；速度放在索引2和索引3的位置，单位是RPM，有正负，需要注意的是将`uint16_t`与`int16_t`强制转型。`|`是按位或运算，与按位与相反，是同位只要有一个为1，结果就为1，其可以用于高低位拼接。

## 最后的最后

要成功传出数据，还有一个要注意的点是函数调用的顺序，在`main.c`中，按照我上面定义的几个函数，应该是这样调用的：
```c
Configure_Filter()；
HAL_CAN_Start(&hcan1);

while(1){
     HAL_Delay(50);
     Set_GM6020_Voltage(8000);
}
```
这样便可以实现每隔50ms向电机发送5000电压数据开转了。需要注意就是`Configure_Filter()`要在`HAL_CAN_Start(&hcan1)`之前调用。