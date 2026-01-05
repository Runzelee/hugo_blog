---
author: Runze Lee
title: 宇树A1电机折腾笔记
date: 2026-01-04
license: CC 4.0 BY-SA
description: RM工程电控学习笔记
image: image/unitree_a1_1.jpg
categories: 
     - Tech
     - RoboMaster
tags:
     - 学习笔记
math: true

---

{{< figure src="/image/unitree_a1_1.jpg" width="50%" >}}

上图就是笨笨的宇树A1，这是我目前为止转过的最难转的电机。电机的说明书、SDK链接都来自[MATH-286-Pro的视频](https://www.bilibili.com/video/BV1nt421874A)提供：[宇树A1相关资料](https://pan.baidu.com/s/1sLqHb3r1d-6KKryThBg3Ww?pwd=1234)、[宇树官方SDK仓库](https://github.com/unitreerobotics/unitree_actuator_sdk)。这篇笔记分两部分，先使用SDK驱动电机，再迁移到STM32下位机上。

## 电脑SDK控制

### 变态的硬件接线

首先，驱动宇树A1需要的是RS-485格式的信号，这是一种类似CAN的**差分信号**，但和CAN不同的是并不采用总线方式，而是直接把TTL串口信号的方波复制一遍并镜像，因此一条线的串口信号就需要两条A/B线来传输，下图是我用示波器测出来的完美RS-485方波：

![示波器](image/unitree_a1_4.jpg)

我们的大致思路是，电脑接一个USB转TTL模块，下游再接一个TTL转RS-485模块，下游再接电机，这是我这次选择的硬件（队里也只有这一套）：

> USB转TTL模块：FT232H
>
> TTL转RS-485模块：MAX13488

由于宇树A1需要的波特率极高，4.8MBd，一般常用的CH340、CH341都不支持，FT232H是少有支持高波特率的USB转TTL/GPIO/I2C/SPI多功能芯片（最高支持12MBd），非常适合这一场景；MAX13488则最高支持16MBd，分别是下面两张图：



{{< figure src="/image/unitree_a1_3.jpg" width="50%" >}}
{{< figure src="/image/unitree_a1_2.jpg" width="50%" >}}

然后就是具体的接线：

| FT232H    | MAX13488 |
| --------- | -------- |
| AD0（TX） | TX       |
| AD1（RX） | RX       |
| 5V        | 5V       |
| GND       | GND      |

默认状态下，AD0就是FT232H的TX脚，AD1就是FT232H的RX脚。这里非常容易搞错的是，RX对RX，TX对TX，而**不是交叉相接**！可以这样理解，因为MAX13488并不是“对面的目标”，而是“我方的中转站”，接线正常情况下MAX13488上的TX灯会亮起，这个灯是红灯，**红灯并不代表出错反而是成功**。下面是宇树A1的接口线序：

{{< figure src="/image/unitree_a1_5.jpg" width="50%" >}}

上图方向插线的3pin口，**从左至右分别是B、A、GND**，左右两个3pin口线序完全一致，可以用于串联多个电机。直接B接B，A接A，GND接GND连到MAX13488另一端即可。

最早提到过，对于差分信号，一根线的数据应该需要一组A/B线才能传输，按照这样的逻辑，RX/TX应该需要两组A/B线，一共四条线，但我们看到MAX13488和电机侧只有一组A/B线。这是因为宇树A1仅支持半双工，而RX/TX TTL通信是全双工，MAX13488进行了转换，原先是发送接收可以同时进行，两条线互不干扰，但现在只有一条线（一组A/B只能表示一条线的信号），因此无法同时传输，这里MAX13488芯片自动处理了阻塞的逻辑。

### 环境配置

建议在Linux系统下使用SDK，更快捷方便。我在Ubuntu 22.04 LTS环境中测试通过，其他发行版可能出现各种版本问题无法编译（我使用的ArchLinux由于gcc太新就报错，其他发行版可以使用Distrobox不需要重装系统，这里不多赘述）。

```bash
git clone https://github.com/unitreerobotics/unitree_actuator_sdk
cd unitree_actuator_sdk
mkdir build
cd build
cmake ..
make
sudo ./changeID
```

七行命令就可以编译运行更改ID的程序。其中会要求你输入USB设备路径，一般情况下都是`/dev/ttyUSB0`，如果你不确认是不是这个设备，可以做一个简单的测试：

```bash
ls /dev | grep ttyUSB
```

运行这个命令，只有一个`ttyUSB0`，那就不用想了；如果有好几个，可以尝试拔掉FT232H再运行看看哪个设备消失了，消失的就是FT232H。`changeID`运行后，会输出：

```
[WARNING] SerialPort::recv, unblock version, wait time out
[WARNING] motor id=187 does not reply
Please turn the motor.
One time: id=0; Two times: id=1, Three times: id=2
ID can only be 0, 1, 2
Once finished, press 'a'
```

看到`wait time out`和`does not reply`不需要慌张，因为接收电机信息的线路一直比较抽风，我们只要保证能给电机发送信息就能正常驱动电机，包括速控和位控。这里有一个用于检测电机是否接收到广播的奇技淫巧，就是将B、A、GND反接成GND、A、B重新上电，运行`changeID`，如果电机突然锁死，内部发出声音，再按`a`回车声音消失，就是接收到了信息。

如果一切正常，就可以开始设置ID，宇树A1的ID只有0、1、2。这里的ID设置方式极度硬核，人为外力把电机完整的转一圈再按`a`就是0，两圈就是1，三圈就是2，建议像本文第一张图那样加两个螺丝，这样外力更好拧（说明书也是这么建议的）。ID设置完，需要在`example/example_a1_motor.cpp`中修改`serial()`和`cmd.id`，分别是USB设备路径和电机ID，再编译运行`example_a1_motor`，电机应该就能以默认参数速控开转。这里给一组位控的参数，n在这里是圈数：

```cpp
cmd.kp    = 0.1;
cmd.kd    = 2;
cmd.q     = n*6.28*queryGearRatio(MotorType::A1);
cmd.dq    = -3.14*queryGearRatio(MotorType::A1);
cmd.tau   = 0.0;
```



## 下位机直接控制

这里使用`STM32F405RGT6`为例直接控制宇树A1电机。对于接线，和上面电脑连接的逻辑相同，现在不过变成MCU上的RX脚接MAX13488的RX，MCU上的TX脚接MAX13488的TX。这里需要注意的是，**需要使用F4主控的USART1或USART6**，因为这两个串口挂在APB2上，支持84MHz的时钟频率，在默认采样模式（OverSampling=16）下，是5.25MBd的最高波特率，而另外几个挂在APB1上，最高2.625MBd，不满足我们4.8MBd的条件。

将完整通信协议迁移到下位机的详细代码构建流程不多赘述，这里我可以给出两个资料参考，一个是电机接收报文格式（也摘取自之前开头的宇树A1相关资料 ~~，这里的markdown格式喂给AI更方便~~ ）：

| 字节偏移 | 变量名    | 数据类型 | 说明                                        | 物理量换算公式 (发送前)     |
| :------- | :-------- | :------- | :------------------------------------------ | :-------------------------- |
| 0        | start[0]  | uint8    | 帧头，固定 `0xFE`                           | -                           |
| 1        | start[1]  | uint8    | 帧头，固定 `0xEE`                           | -                           |
| 2        | motorID   | uint8    | 目标电机ID (0~2) `0xBB` 为广播              | -                           |
| 3        | reserved  | uint8    | 预留，固定 0                                | -                           |
| 4        | mode      | uint8    | 控制模式： 0: 停转 5: 开环缓转 10: 闭环伺服 | -                           |
| 5        | ModifyBit | uint8    | 参数修改位 (通常 0)                         | -                           |
| 6        | ReadBit   | uint8    | 读取参数位 (通常 0)                         | -                           |
| 7        | reserved  | uint8    | 预留 (通常全 0)                             | -                           |
| 8-11     | Modify    | uint32   | 参数修改位 (通常全 0)                       | -                           |
| 12-13    | T         | int16    | 前馈力矩 (Nm)                               | `Target_T * 256`            |
| 14-15    | W         | int16    | 目标角速度 (rad/s)                          | `Target_W * 128`            |
| 16-19    | Pos       | int32    | 目标位置 (rad)                              | `Target_Pos * 16384/(2*PI)` |
| 20-21    | K_P       | int16    | 位置刚度 (系数)                             | `Target_Kp * 2048`          |
| 22-23    | K_W       | int16    | 速度刚度 (系数)                             | `Target_Kw * 1024`          |
| 24       | LowHz     | uint8    | 低频控制索引                                | -                           |
| 25       | LowHz     | uint8    | 低频控制值                                  | -                           |
| 26-29    | Res       | uint8[4] | 预留                                        | -                           |
| 30-33    | CRC       | uint32   | CRC32 校验码                                | 校验前 30 个字节            |

另外一个是我根据这个格式写 ~~（vibe coding）~~ 的下位机驱动代码，测试通过，[已经推送到GitHub](https://github.com/Runzelee/Unitree_A1_motor)。