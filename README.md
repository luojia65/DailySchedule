# DailySchedule

2020年六月

| 周一 | 周二 | 周三 | 周四 | 周五 | 周六 | 周日 |
|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
|    |    |    | 25 | 26 | 27 | 28 |
| 29 | 30 |    |    |    |    |    |

## 第1天

### 1. 尝试提出适配SBI到Rust的思路

Rust嵌入式已经提出No-RTOS下的外设标准称为[embedded-hal](https://docs.rs/embedded-hal/1.0.0-alpha.1/embedded_hal/index.html)。这个标准在没有操作系统的裸机开发下非常好用、省力，也便于快速适配和实现各种外设；但是没有给出标准的设备树搭建方法。
笔者建议Rust+RISC-V标准的操作系统，在统一的设备树标准下实现。RISC-V提出了统一的SBI标准，它会返回核的HartID以及设备树的位置，以便给出统一的抽象。
OpenSBI是SBI的实现之一。rCore社区给出了OpenSBI的适配库`opensbi-rt`。建议未来的社区此种适配库分为两部分：`sbi-rt`运行时和`sbi`底层标准库。（OpenSBI是一个实现，SBI是一个标准，建议使用SBI作为名称。）
`sbi-rt`将用于搭建bootstack等简单的运行时操作，建议不包含动态内存部分（alloc库），更适合操作系统自己挑选动态内存库。`sbi`包含所有SBI标准的调用，包括`println!`以及所有的SBI方法。

此类设计模式在嵌入式Rust社区已经十分成熟，目前出现了`cortex-m`和`cortex-m-rt`、`riscv`和`riscv-rt`等库，都十分常用，但通常搭配组成M特权模式上的运行时库。预期的`sbi-rt`可能称为S特权模式上的运行时库。如果有机会，和社区成员沟通后，未来有机会将预期的`sbi-rt`合并到`riscv-rt`。

SBI部分的实现可以由QEMU等模拟器提供，在真实机器上使用时，也可以由真实机器的固件提供。这个固件实现有很多，RISC-V基金会提供了参考的OpenSBI实现，当然也已经有更多的实现（[比如这些](https://github.com/riscv/riscv-sbi-doc/blob/master/riscv-sbi.adoc#sbi-implementation-ids)）。
我们可以基于社区成熟的embedded-hal外设标准库，搭建自己的实现。甚至可以建设或许能称为`rustsbi`的库，这个库提供一个方便的构造器，可以运用Rust语言灵活和零抽象开销的特点，从embedded-hal外设直接搭建设备树。加上简单的适配，我们可以运用这个库，在数十行代码内编写M模式下的SBI程序，这样就能直接在操作系统套用`sbi-rt`库了。
如果库做得好，甚至可以在已有的“SBI Implementation IDs”列表内占用一个单独的Implementation ID，以便进一步推广我们的SBI实现库。

目前Rust已有的嵌入式开发生态由PAC、HAL、应用组成，PAC代表由SVD等描述文件生成的外设访问crate，HAL代表硬件中间层；在此之上还有[RTIC](https://rtic.rs)等成熟的裸机中断调度库，可以完成No-RTOS场合的开发过程。
我们需要补上存在OS（如RTOS）时的生态部分。这样的生态如果能够搭建完成，Rust开发OS在标准层级的最后一环将被打通。相信会有更多的人，无论是学习还是生产开发，都能灵活运用共同的标准更好地完成任务。

给出一段理想状况下的代码：

```rust
// M特权层
lazy_static! { SBI_INSTANCE = ... };

#[riscv_rt::entry]
fn main() -> ! {
    // 假设只有一个Hart在运行程序
    // 1. 从PAC（外设访问）库、HAL（硬件中间层）库配置裸机环境
    let dp = pac::Peripherals::take();
    let clocks = dp.CLOCKS.constrain(/*...*/);
    let mut serial = dp.SERIAL.configure(/*...*/);
    // 2. 从embedded-hal裸机环境创建rustsbi实例
    SBI_INSTANCE = rustsbi::InstanceBuilder::new()
        .stdio(EmbeddedHalSerial(&mut serial))
        // ...其它的SBI外设
        .build().unwrap(); // 设备树将由rustsbi生成
    // 3. 修改mideleg、medeleg寄存器，将中断委托给rustsbi（略）
    // 4. 启动SBI
    SBI_INSTANCE.execute()
}

// S特权层
use sbi::println;

#[sbi_rt::entry]
fn main(hart_id: usize, dtb: &DeviceTree) {
    println!("操作系统启动！");
    // todo: 调度器；加载用户程序
}

// U特权层

fn main() {
    println!("Hello world!")
}
```

### 2. 完善社区的相关依赖库

在`riscv`库增加一个寄存器。提交了Pull Request：[rust-embedded/riscv#52](https://github.com/rust-embedded/riscv/pull/52)。未来可能要把页表支持放到这个库里面，等待以后成熟定型的工程设计出现，现在不着急。

添加一些Rust常用的API风格，充分利用Rust宏的灵活特性。提交了Pull Request：[rcore-os/opensbi-rt#1](https://github.com/rcore-os/opensbi-rt/pulls)。希望以后能把SBI的功能分开成单独的库，只有运行时部分保留在这个crate中。
