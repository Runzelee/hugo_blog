---
author: Runze Lee
title: STM32 Rust HAL库入门笔记
date: 2026-07-26
license: CC 4.0 BY-SA
description: RM电控学习笔记
image: image/rust_hal_2.png
categories: 
     - Tech
     - STM32
tags:
     - 学习笔记
---

Rust由于其内存安全和零成本抽象的特性，在近两年非常非常火，原来只能使用C/C++的地方如操作系统、浏览器内核、嵌入式等领域都纷纷出现了Rust的实现，甚至Linux内核都允许了Rust语言的贡献。使用Trait、Generic、Impl等实现的多态、泛型、面向对象，在代码中看起来高度现代化和抽象，只要编译通过就近似保持着数学级别的内存安全证明，但实际编译成机器码也和C差不多就是寥寥几行的寄存器操作，这种现代化达到了底层开发的甜点位。

嵌入式是Rust的一个重要应用领域，STM32 Rust HAL更是重中之重，他把Rust语言的先进特性全部带进了STM32开发当中。

## 搭建项目

确保电脑已经安装了Rust工具链，这里以STM32F405为例：
```bash
rustup target add thumbv7em-none-eabihf
cargo new --bin stm32f405-rust
cd stm32f405-rust
```
配置Cargo.toml：
```toml
[package]
name = "stm32f405-rust"
version = "0.1.0"
edition = "2024"

[dependencies]
cortex-m = "0.7.7"
cortex-m-rt = "0.7.5"
panic-halt = "1.0.0"
nb = "1"

stm32f4xx-hal = {
    version = "0.23.0",
    features = ["stm32f405"]
}
```
注意这里的dependencies有五个：cortex-m类似C的CMSIS Core，负责Cortex CPU本身的相关操作，比如IRQ（Interrupt Request，中断请求）和CPU自己的寄存器等操作，不触及STM32层。cortex-m-rt的rt指runtime，做类似`startup_stm32f405xx.s`的事情，负责向量表、.bss/.data段初始化以及各种handler的处理。panic-halt是为了让程序触发panic时直接停住，因为单片机环境没有操作系统接手panic。nb是non-blocking，非阻塞，用来给一些轮询API提供表达还需等待的状态。stm32f4xx-hal则是Rust HAL库本身。

接下来则需要配置memory.x来规定各内存的起始位置和长度，C HAL（gcc）中CubeMX已经帮我们生成了链接脚本`STM32F405XX_FLASH.ld`，有类似：
```
/* Specify the memory areas */
MEMORY
{
RAM (xrw)      : ORIGIN = 0x20000000, LENGTH = 128K
CCMRAM (xrw)      : ORIGIN = 0x10000000, LENGTH = 64K
FLASH (rx)      : ORIGIN = 0x8000000, LENGTH = 1024K
}
```
直接把他扔进memory.x即可，放在仓库根目录。当然需要让链接器读取到memory.x还需要写build.rs，这是Cargo构建脚本，类似CMakeLists的作用：
```rust
use std::env;
use std::fs;
use std::path::PathBuf;

fn main() {
    let out_dir = PathBuf::from(
        env::var_os("OUT_DIR").expect("OUT_DIR not set")
    );
    fs::copy("memory.x", out_dir.join("memory.x"))
        .expect("failed to copy memory.x");
    println!("cargo:rustc-link-search={}", out_dir.display());
    println!("cargo:rerun-if-changed=memory.x");
    println!("cargo:rustc-link-arg=-Tlink.x");
    println!("cargo:rustc-link-arg=--nmagic");
}
```
可以注意到这里用了标准库std，并且使用了本地操作系统的文件系统`fs`和环境变量`env`，说明这是编译到本地运行的crate，而非用于单片机。`out_dir`是专为存放编译时中间产物的目录，这里的操作即是把memory.x复制到该目录下，很有意思的是Cargo是读取这个程序的标准输出`println!`来指定rustc的编译命令和参数。这里则是把输出目录的memory.x放入链接搜索目录，如果memory.x发生改变则重新编译运行build.rs，并指定link.x为链接脚本，link.x是cortex-m-rt包官方提供的通用链接脚本，其中依赖针对具体型号芯片的memory.x，这样就将通用链接步骤和具体内存分布解耦，远远比C的ld脚本先进。

接下来我们就开始写src/main.rs，Rust HAL版本Helloworld：
```rust
#![no_std]
#![no_main]

use cortex_m_rt::entry;
use panic_halt as _;

use stm32f4xx_hal::pac;

#[entry]
fn main() -> ! {
    let _dp = pac::Peripherals::take().unwrap();
    loop {
    }
}
```
开头的`#![no_std]` `#![no_main]`指的是不使用Rust标准库和main函数，和之前的build.rs截然相反，因为嵌入式开发无操作系统，自然也没有常规意义上的main函数。所以，所谓的“main函数”作为程序入口交给了cortex_m_rt来处理，也就是这里的`#[entry]`，这个函数返回类型是`!`，即never type，永不返回，因为没有操作系统用于返回这个函数，也没有输入输出。`panic_halt`先前已经介绍过，将其别名为`_`则意味着不可能会主动调用这个crate中的函数方法，他只会默默承受panic并中止程序。

这里的`pac`便是大名鼎鼎的外设访问单元（Peripheral Access Crate，PAC），他用于管理单片机的所有外设寄存器并提供读写API。`take()`方法则是在安全的情况下提供外设对象的所有权。

## 时钟配置

相比C语言配置时钟，Rust HAL配置时钟可以说非常惊艳。现在以下面的配置为例，一块焊了8MHz外部晶振的板子：

![CubeMX时钟配置](image/rust_hal_1.png)

这是来自CubeMX的截图，这里可以看到使用了8MHz的HSE，PLL得到系统时钟168MHz，最后分别分频给了AHB，APB1和APB2。在CubeMX中，我们只需要设置HCLK回车就能自动计算出PLLM/N/P的合理组合达成目标，实际生成的C代码是这样的：
```c
void SystemClock_Config(void)
{
  RCC_OscInitTypeDef RCC_OscInitStruct = {0};
  RCC_ClkInitTypeDef RCC_ClkInitStruct = {0};

  /** Configure the main internal regulator output voltage
  */
  __HAL_RCC_PWR_CLK_ENABLE();
  __HAL_PWR_VOLTAGESCALING_CONFIG(PWR_REGULATOR_VOLTAGE_SCALE1);

  /** Initializes the RCC Oscillators according to the specified parameters
  * in the RCC_OscInitTypeDef structure.
  */
  RCC_OscInitStruct.OscillatorType = RCC_OSCILLATORTYPE_HSE;
  RCC_OscInitStruct.HSEState = RCC_HSE_ON;
  RCC_OscInitStruct.PLL.PLLState = RCC_PLL_ON;
  RCC_OscInitStruct.PLL.PLLSource = RCC_PLLSOURCE_HSE;
  RCC_OscInitStruct.PLL.PLLM = 4;
  RCC_OscInitStruct.PLL.PLLN = 168;
  RCC_OscInitStruct.PLL.PLLP = RCC_PLLP_DIV2;
  RCC_OscInitStruct.PLL.PLLQ = 4;
  if (HAL_RCC_OscConfig(&RCC_OscInitStruct) != HAL_OK)
  {
    Error_Handler();
  }

  /** Initializes the CPU, AHB and APB buses clocks
  */
  RCC_ClkInitStruct.ClockType = RCC_CLOCKTYPE_HCLK|RCC_CLOCKTYPE_SYSCLK
                              |RCC_CLOCKTYPE_PCLK1|RCC_CLOCKTYPE_PCLK2;
  RCC_ClkInitStruct.SYSCLKSource = RCC_SYSCLKSOURCE_PLLCLK;
  RCC_ClkInitStruct.AHBCLKDivider = RCC_SYSCLK_DIV1;
  RCC_ClkInitStruct.APB1CLKDivider = RCC_HCLK_DIV4;
  RCC_ClkInitStruct.APB2CLKDivider = RCC_HCLK_DIV2;

  if (HAL_RCC_ClockConfig(&RCC_ClkInitStruct, FLASH_LATENCY_5) != HAL_OK)
  {
    Error_Handler();
  }
}
```
可以看到，C代码硬编码了所有内容，包括HSE、PLLM/N/P/Q系数、APB1/APB2/AHB的分频系数，然后作为结构体地址传进`HAL_RCC_OscConfig()`和`HAL_RCC_ClockConfig()`，他不会验证参数计算是否正确或合理，只是直接把这些东西丢给寄存器，所以一切复杂度都给了CubeMX的图形化操作界面。相反，Rust HAL把这些都集成到了HAL自己身上：
```rust
use stm32f4xx_hal::{
    pac,
    prelude::*,
    rcc::Config,
};

let dp = pac::Peripherals::take().unwrap();

let mut rcc = dp.RCC.freeze(
    Config::hse(8.MHz())
        .sysclk(168.MHz())
        .pclk1(42.MHz())
        .pclk2(84.MHz())
);
```
看到了吗？**只需要指定HSE输入频率，期望的系统时钟、APB1、APB2目标频率就结束了！只用了一个表达式！** Rust HAL会自动计算最优的PLLM/N/P/Q系数，做完所有脏活累活，我们只需要关心目标，配合Rust可以把物理量纲本身构造成类型，极度舒适。这里的`freeze()`把当前的时钟设置固定下来用来孵化新的外设对象。

## 基本外设用法

### GPIO

以PA5为例，推挽输出，CubeMX拉的一坨：
```c
GPIO_InitTypeDef GPIO_InitStruct = {0};

__HAL_RCC_GPIOA_CLK_ENABLE();

GPIO_InitStruct.Pin = GPIO_PIN_5;
GPIO_InitStruct.Mode = GPIO_MODE_OUTPUT_PP;
GPIO_InitStruct.Pull = GPIO_NOPULL;
GPIO_InitStruct.Speed = GPIO_SPEED_FREQ_LOW;

HAL_GPIO_Init(GPIOA, &GPIO_InitStruct);
```
Rust两句话：
```rust
let gpioa = dp.GPIOA.split(&mut rcc);
let mut pa5 = gpioa.pa5.into_push_pull_output();
```
或者指定默认电平：
```rust
let mut pa5 = gpioa.pa5.into_push_pull_output_in_state(PinState::Low);
```
`split()`方法很有意思，顾名思义，指的是把GPIOA的所有权从整个PAC实例里“分割出来”，传进去可变引用`&mut rcc`也说明要暂时借用并**开启对应时钟**，本质上就是`__HAL_RCC_GPIOA_CLK_ENABLE()`。`HAL_GPIO_WritePin()`/`HAL_GPIO_TogglePin()`就变成了：
```rust
pa5.set_high();
pa5.set_low();
pa5.toggle();
```
一对比，面向对象的特性立刻展现，再也不用到处用函数传指针了。外设都变成了对象，所有特性和配置都变成了类型的一部分，外设自己的操作就是自己的方法调用。

如果是GPIO Input，以PA0为例，分别对应悬浮、上拉和下拉：
```rust
let pa0 = gpioa.pa0.into_floating_input();
let pa0 = gpioa.pa0.into_pull_up_input();
let pa0 = gpioa.pa0.into_pull_down_input();
```
原来的`HAL_GPIO_ReadPin()`和`if (state == GPIO_PIN_XXX)`就变成了:
```rust
if pa0.is_high() {
    // high
}

if pa0.is_low() {
    // low
}
```
极度清爽。

### 基本定时器

先在`main()`中初始化定时器，这里以TIM2为例：
```rust
let mut timer = dp.TIM2.counter(&mut rcc);
timer.start(1.secs()).unwrap();
timer.listen(Event::Update);
unsafe {
    NVIC::unmask(Interrupt::TIM2);
}
loop {
    cortex_m::asm::wfi();
}
```
这里的`counter()`方法即把TIM2对象转换为一个基本定时器对象。和时钟配置一样，只需要指定`timer.start()`的目标中断时间就可以，这里是1秒，**不需要和CubeMX一样自己计算和配置PSC和ARR数值，一切Rust HAL都做好了**。`listen()` 则是开始监听中断溢出事件，类似C中的`HAL_TIM_Base_Start_IT(&htim2)`，接下来还需要手动开启NVIC向量表中的TIM2中断，这一点Rust编译器无法证明开中断的那一刻会不会直接触发中断从而出现共享资源冲突，因此要进入unsafe，类似`HAL_NVIC_EnableIRQ(TIM2_IRQn)`。loop循环中调用`wfi()`，即等待中断（Wait For Interrupt，WFI）。

接下来就要进入比较复杂的部分了，C HAL中全局变量满天飞，`TIM_HandleTypeDef htim2`主函数和中断同时都想拿句柄随便拿没人管，但是Rust不一样，同一时刻只能有一个可变引用，因此这里需要一系列机制来保证main函数和中断里共享全局资源还能通过Rust编译。我们开辟一个全局共享区域来存放TIM定时器对象：
```rust
static TIMER: Mutex<RefCell<Option<CounterUs<TIM2>>>> =
    Mutex::new(RefCell::new(None));
```
`CounterUs<TIM2>`是TIM Counter对象本身的类型，我们对其做了多层包装。首先是一层`Option`，因为刚上电Counter对象一定还没有创建，这里只是一个Some/None槽位来存放它；接下来是`RefCell`，Rust不允许修改不可变引用，但`RefCell`却提供方法在运行态检查不可变引用指向的对象在当时是否可以安全修改，相当于把编译时的证明推后到了运行态；最后`Mutex`是互斥锁，用来提供刚才所说的这个不可变引用，但强调必须在特定的临界区（Critical Section，CS）中才能提供，这个临界区在哪是我们后期决定的，这保证了绝对安全。

现在我们在main函数中把我们创建的TIM Counter对象放进这个共享区域：
```rust
cortex_m::interrupt::free(|cs| {
    TIMER.borrow(cs).replace(Some(timer));
});
```
这里的`interrupt::free()`方法的意思是暂时关闭中断，类似C的：
```c
__disable_irq();
/* critical section BEGIN */
g_timer = timer;
/* critical section END */
__enable_irq();
```
在C HAL中我们为了保护`timer`给全局`g_timer`赋值的过程中不会突然触发中断造成异常，会专门关一下中断再开启。Rust当中也是一样，只不过把从`__disable_irq()` 到`__enable_irq()` 这段窗口期起了个名字叫做CS临界区。`interrupt::free()`接收一个闭包，提供参数`cs`作为进入临界区的凭证，由此以来，把它传进`borrow()`方法，这是`Mutex`的方法，互斥锁就知道已经在临界区内，因此就安全提供`TIMER`共享区域的不可变引用。接下来的`replace()`则是`RefCell`的方法，其作用就是刚才提到的在运行态检查上游`Mutex`提供的不可变引用是否可以安全修改，如果可以就将其改为`Some(timer)`。这样一来，在Rust严苛的编译条件下实现了**在临时关闭中断的情况下把main函数本地的TIM Counter对象传进全局共享区域**。

然后我们就可以在中断当中拿到TIM Counter对象了，还是一样，要注意的是如果以TIM2 Counter为例这里的函数名TIM2就是固定的，不可以自定义：
```rust
#[interrupt]
fn TIM2() {
    cortex_m::interrupt::free(|cs| {
        if let Some(timer) = TIMER.borrow(cs).borrow_mut().as_mut() {
            timer.clear_flags(Flag::Update);
        }

        // Do something
    });
}
```
触发中断后依旧是先`interrupt::free()`暂时关闭中断进入CS临界区，然后开始拿对象，`borrow(cs)`依旧是从`Mutex`拿CS凭证换不可变引用，然后通过`RefCell`的`borrow_mut()`运行时把它转换成可变引用，最后`as_mut()`是`Option`的方法，用于把`Some(timer)`拿出来。`if let`确保其是`Some`而忽略`None`，拿到最终的`timer`对象后直接`clear_flags(Flag::Update)`，类似C的`__HAL_TIM_CLEAR_FLAG(htim, TIM_FLAG_UPDATE)`，清除标志位，否则会始终不停地进中断。接下来就可以写自己的业务逻辑了，就像在`HAL_TIM_PeriodElapsedCallback()`里一样。

最后不能忘记开头use这些类型或trait：
```rust
use core::cell::RefCell;
use cortex_m::interrupt::Mutex;
use stm32f4xx_hal::{
    pac::{interrupt, Interrupt, Peripherals, TIM2},
    prelude::*,
    rcc::Config,
    timer::{CounterUs, Event, Flag},
};
```
我们可以发现，Rust在中断处理、资源共享上由于繁重的安全约束似乎并不直观，但这并不影响我们写实际的业务逻辑。比如我们要共享一个LED灯对象进来，每秒翻转一次，也就是相同的样板代码：
```rust
// ...
static TIMER: Mutex<RefCell<Option<CounterUs<TIM2>>>> =
    Mutex::new(RefCell::new(None));
static LED: Mutex<RefCell<Option<LedPin>>> =
    Mutex::new(RefCell::new(None));

#[entry]
fn main() -> ! {
    // ...
    let gpioa = dp.GPIOA.split(&mut rcc);

    let mut led = gpioa.pa5.into_push_pull_output();
    let mut timer = dp.TIM2.counter(&mut rcc);
    timer.start(1.secs()).unwrap();
    timer.listen(Event::Update);

    cortex_m::interrupt::free(|cs| {
        TIMER.borrow(cs).replace(Some(timer));
        LED.borrow(cs).replace(Some(led)); // 给出LED
    });
    unsafe {
        cortex_m::peripheral::NVIC::unmask(Interrupt::TIM2);
    }
    loop {
        cortex_m::asm::wfi();
    }
}

#[interrupt]
fn TIM2() {
    cortex_m::interrupt::free(|cs| {
        if let Some(timer) = TIMER.borrow(cs).borrow_mut().as_mut() {
            timer.clear_flags(Flag::Update);
        }

        // 拿到LED
        if let Some(led) = LED.borrow(cs).borrow_mut().as_mut() {
            led.toggle();
        }
    });
}
```
熟练后也不比C HAL“更麻烦”，代码量最终还是少得多，还免得计算PSC/ARR。

### USART

以PA9和PA10为例，USART1，初始化：
```rust
let gpioa = dp.GPIOA.split(&mut rcc);

let pins = (
    gpioa.pa9,   // TX
    gpioa.pa10,  // RX
);

let mut serial = Serial::new(
    dp.USART1,
    pins,
    SerialConfig::default()
        .baudrate(115_200.bps()),
    &mut rcc,
)
.unwrap();
```
创建`serial`对象非常直观，依旧是直接指定波特率，不多赘述。这里的`default()`默认是8N1（8位数据位，无奇偶校验，1位停止位），如果需要9-bit模式可以使用`wordlength_9()`。打印字符串更是方便，可以使用Rust原生`core::fmt::Write`：
```rust
write!(serial, "hello").unwrap();
writeln!(serial, "value = {}", 123).unwrap();
```
`writeln!()`默认是LF换行，如果对方依赖CRLF换行，可以在末尾加上`\r`，即`writeln!(serial, "value = {}\r", 123).unwrap();`。

`split()`可以把`serial`分割成单独的`rx`和`tx`对象，`rx.listen()`就可以直接开始监听RX引脚SR里的RXNE位，也就是准备接收中断。类似TIM定时器中断的思路，一样unsafe下打开NVIC中断、把`rx`对象交给共享区，然后在`loop`里调用`wfi()`：
```rust
let (mut tx, mut rx) = serial.split();
rx.listen();

unsafe {
    cortex_m::peripheral::NVIC::unmask(
        pac::Interrupt::USART1
    );
}

cortex_m::interrupt::free(|cs| {
    RX.borrow(cs).replace(Some(rx));
});

loop {
    cortex_m::asm::wfi();
}
```
当然一开始要开辟共享区：
```rust
static RX: Mutex<RefCell<Option<Rx<USART1>>>> =
    Mutex::new(RefCell::new(None));
```
和TIM定时器不同的是，中断中拿到`rx`对象第一时间不是`clear_flags()`，而是`rx.read()`，语义是从RX读取一个字节，本质上就是类似`(void)USART1->DR;`读取一下DR硬件清除RXNE的传统手艺。
```rust
#[interrupt]
fn USART1() {
    cortex_m::interrupt::free(|cs| {
        if let Some(rx) = RX.borrow(cs).borrow_mut().as_mut() {
            match rx.read() {
                Ok(byte) => {
                    // 收到 byte
                }

                Err(nb::Error::WouldBlock) => {
                    // 暂无数据需要等待
                }

                Err(nb::Error::Other(_)) => {
                    // 串口错误
                }
            }
        }
    });
}
```
这里和`HAL_UART_Receive_IT()`不同的是其收完特定长度的字节就会一次性触发`HAL_UART_RxCpltCallback()`然后结束，因此每次回调最后还要再次`HAL_UART_Receive_IT()`重新启用。但`rx.read()`则是始终监听RXNE，只要收到一个字节就会进回调，不需要重新启用，但如果要连续收多字节则要自己维护缓冲区。

### CAN

CAN需要在Cargo.toml加上`bxcan`依赖，并在`stm32f4xx-hal`的feature里加上`can`：
```toml
[dependencies]
cortex-m = "0.7.7"
cortex-m-rt = "0.7.5"
panic-halt = "1.0.0"
nb = "1"
bxcan = "0.8"

stm32f4xx-hal = {
    version = "0.23.0",
    features = ["stm32f405", "can"]
}
```
首先初始化CAN外设：
```rust
let can_peripheral = Can::new(
    dp.CAN1,
    (gpioa.pa12, gpioa.pa11),
    &mut rcc,
);

let mut can = BxCan::builder(can_peripheral)
    .set_bit_timing(0x001A_0002)
    .enable();
```
这里的接口确实比较原始，`set_bit_timing()`传进去的是BSR原始16进制值，我们完全可以手动封装一个函数来指定PSC、BS1、BS2和SJW：
```rust
const fn bxcan_btr(
    prescaler: u32,
    bs1: u32,
    bs2: u32,
    sjw: u32,
) -> u32 {
    ((prescaler - 1) & 0x3FF)
        | (((bs1 - 1) & 0x0F) << 16)
        | (((bs2 - 1) & 0x07) << 20)
        | (((sjw - 1) & 0x03) << 24)
}

let mut can = BxCan::builder(can_peripheral)
    .set_bit_timing(bxcan_btr(3, 11, 2, 1))
    .enable();
```
然后配置默认FIFO和滤波器，这里`accept_all()`就是全部通过：
```rust
can.modify_filters()
    .enable_bank(
        0,
        Fifo::Fifo0,
        Mask32::accept_all(),
    );
```
如果需要特定IDLIST和IDMASK过滤，也比C HAL直观得多，`ListEntry16`在这里正好是4个16位ID填满64位的Filter Bank 0：
```rust
let filters = [
    ListEntry16::data_frames_with_id(
        StandardId::new(0x201).unwrap()
    ),
    ListEntry16::data_frames_with_id(
        StandardId::new(0x202).unwrap()
    ),
    ListEntry16::data_frames_with_id(
        StandardId::new(0x203).unwrap()
    ),
    ListEntry16::data_frames_with_id(
        StandardId::new(0x204).unwrap()
    ),
];

can.modify_filters()
    .enable_bank(
        0,
        Fifo::Fifo0,
        filters,
    );
```
IDMASK：
```rust
let filter = Mask32::frames_with_std_id(
    StandardId::new(0x201).unwrap(),
    StandardId::new(0x7FF).unwrap(),
);

can.modify_filters()
    .enable_bank(
        0,
        Fifo::Fifo0,
        filter,
    );
```
接下来发送CAN帧：
```rust
let id = StandardId::new(0x201).unwrap();

let frame = Frame::new_data(
    id,
    [1, 2, 3, 4, 5, 6, 7, 8],
);

can.transmit(&frame).unwrap();
```
结束，甚至不需要去关心发送邮箱，`bxcan`会自动处理排队。

开启接收中断还是类似的老步骤，先创建共享区：
```rust
type Can1 = bxcan::Can<HalCan<CAN1>>;

static CAN: Mutex<RefCell<Option<Can1>>> =
    Mutex::new(RefCell::new(None));
```
然后main里启动接收，unsafe开NVIC，存CAN对象，loop开`wfi()`：
```rust
can.enable_interrupt(
    Interrupt::Fifo0MessagePending,
);

unsafe {
    cortex_m::peripheral::NVIC::unmask(
        pac::Interrupt::CAN1_RX0
    );
}

cortex_m::interrupt::free(|cs| {
    CAN.borrow(cs).replace(Some(can));
});

loop {
    cortex_m::asm::wfi();
}
```
最后再从中断拿：
```rust
#[interrupt]
fn CAN1_RX0() {
    cortex_m::interrupt::free(|cs| {
        if let Some(can) = CAN.borrow(cs).borrow_mut().as_mut() {
            while let Ok(frame) = can.receive() {
                // 处理 frame
                match frame.id() {
                    bxcan::Id::Standard(id) => {
                        match id.as_raw() {
                            0x201 => {
                                // 根据ID匹配
                            }
                    
                            0x202 => {
                                // ...
                            }
                    
                            0x203 => {
                                // ...
                            }
                    
                            _ => {}
                        }
                    }
                    
                    bxcan::Id::Extended(_) => {}
                } 
            }
        }
    });
}
```
这里的`can.receive()`相当于串口的`rx.read()`，不同的是CAN接收的是完整的一帧而不是一字节。可见理解了Rust HAL的资源共享和中断机制后，所有外设都是通用的可以举一反三。

## 烧录和调试

Rust嵌入式项目中openocd不再是首选，因为有更佳的probe-rs，可以在.cargo/config.toml里配置成runner：
```toml
[build]
target = "thumbv7em-none-eabihf"

[target.thumbv7em-none-eabihf]
runner = "probe-rs run --chip STM32F405RGTx"
```
然后编译、烧录、接收日志一条龙只需要一行命令：
```bash
cargo run
```
或者可以用`cargo run --release`，则使用编译优化，类似CMake的Release Preset，调试不建议使用可能会导致单步调试到处乱跳。如果需要仅烧录：
```bash
cargo flash --release --chip STM32F405RGTx
```
此时使用`--release`更好。

调试配合probe-rs VSCode插件，这里提供一个简单的launch.json，`stm32f405-rust`为实际package名：
```json
{
    "version": "0.2.0",
    "configurations": [
        {
            "type": "probe-rs-debug",
            "request": "launch",
            "name": "Debug STM32F405",
            "chip": "STM32F405RGTx",
            "cwd": "${workspaceFolder}",

            "flashingConfig": {
                "flashingEnabled": true,
                "haltAfterReset": true
            },

            "coreConfigs": [
                {
                    "programBinary": "${workspaceFolder}/target/thumbv7em-none-eabihf/debug/stm32f405-rust"
                }
            ]
        }
    ]
}
```
此外，probe-rs还原生支持RTT等特性，可以在不占用串口等外设的情况下打印一些调试变量，这种方式也比之前用C写一大堆全局变量来Live Watch要优雅得多。

用Rust HAL写STM32绝对不仅仅是尝到了点现代语法糖，而是建立了所有权的意识，掌握Rust的基本思想对未来继续写C/C++程序也有很大帮助。Rust HAL大概率是不会在队里推广的，上手难度高、传承困难，但我应该会在27赛季我负责的部分小规模试点使用。