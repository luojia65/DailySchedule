# DailySchedule

2020年六月

| 周一 | 周二 | 周三 | 周四 | 周五 | 周六 | 周日 |
|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
|    |    |    | 25 | 26 | 27 | 28 |
| 29 | 30 |    |    |    |    |    |

[第一天（2020年6月25日）](#第1天)
[第二天（2020年6月26日）](#第2天)

## 第1天

### 1. 尝试提出适配SBI到Rust的思路

Rust嵌入式已经提出No-RTOS下的外设标准称为[embedded-hal](https://docs.rs/embedded-hal/1.0.0-alpha.1/embedded_hal/index.html)。这个标准在没有操作系统的裸机开发下非常好用、省力，也便于快速适配和实现各种外设；但是没有给出标准的设备树搭建方法。
笔者建议Rust+RISC-V标准的操作系统，在统一的设备树标准下实现。RISC-V提出了统一的SBI标准，它会返回核的HartID以及设备树的位置，以便给出统一的抽象。
OpenSBI是SBI的实现之一。rCore社区给出了OpenSBI的适配库`opensbi-rt`。建议未来的社区此种适配库分为两部分：`sbi-rt`运行时和`sbi`底层标准库。（OpenSBI是一个实现，SBI是一个标准，建议使用SBI作为名称。）
`sbi-rt`将用于搭建bootstack等简单的运行时操作，建议不包含动态内存部分（alloc库），更适合操作系统自己挑选动态内存库。`sbi`包含所有SBI标准的调用，包括`println!`以及所有的SBI方法。
这样用`sbi`和`sbi-rt`，就能作为Rust+RISC-V环境下统一的设备树标准，编写完成后，就能更方便地编写操作系统。

此类设计模式在嵌入式Rust社区已经十分成熟，目前出现了`cortex-m`和`cortex-m-rt`、`riscv`和`riscv-rt`等库，都十分常用，但通常搭配组成M特权模式上的运行时库。预期的`sbi-rt`可能称为S特权模式上的运行时库。如果有机会，和社区成员沟通后，未来有机会将预期的`sbi-rt`合并到`riscv-rt`。

SBI部分的实现可以由QEMU等模拟器提供，在真实机器上使用时，也可以由真实机器的固件提供。这个固件实现有很多，RISC-V基金会提供了参考的OpenSBI实现，当然也已经有更多的实现（[比如这些](https://github.com/riscv/riscv-sbi-doc/blob/master/riscv-sbi.adoc#sbi-implementation-ids)）。
我们可以基于社区成熟的embedded-hal外设标准库，搭建自己的实现。甚至可以建设或许能称为`rustsbi`的库，这个库提供一个方便的构造器，可以运用Rust语言灵活和零抽象开销的特点，从embedded-hal外设直接搭建设备树。加上简单的适配，我们可以运用这个库，在数十行代码内编写M模式下的SBI程序，这样就能直接在操作系统套用`sbi-rt`库了。
如果库做得好，甚至可以联系RISC-V基金会，在已有的“SBI Implementation IDs”列表内占用一个单独的Implementation ID，以便进一步推广我们的SBI实现库。

目前Rust已有的嵌入式开发生态由PAC、HAL、应用组成，PAC代表由SVD等描述文件生成的外设访问crate，HAL代表硬件中间层；在此之上还有[RTIC](https://rtic.rs)等成熟的裸机中断调度库，可以完成No-RTOS场合的开发过程。
我们需要补上存在OS（如RTOS）时的生态部分。这样的生态如果能够搭建完成，Rust开发OS在标准层级的最后一环将被打通。相信会有更多的人，无论是学习还是生产开发，都能灵活运用共同的标准更好地完成任务。

或者这之后，rust支持的目标就多了个`riscv64gc-rcore-rcoreos`了呢？:) 隔壁VxWorks就在rust官方支持的目标列表里面，我们也是可以做到的。:)

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

添加一些Rust常用的API风格，充分利用Rust宏的灵活特性。提交了Pull Request：[rcore-os/opensbi-rt#1](https://github.com/rcore-os/opensbi-rt/pull/1)。希望以后能把SBI的功能分开成单独的库，只有运行时部分保留在这个crate中。

### 3. 编写能在qemu启动的实验系统

已经做好了：[这里](https://github.com/luojia65/spicy-os)。使用自己为opensbi-rt做的改进为蓝本，希望PR能被合并。尝试使用just代替make作为编译脚本，因为make并不支持所有的操作系统（比如没装MinGW的windows）。

### 4. 一些感悟

从去年到今年年初，我一直在参与翻译《Writing an OS in Rust》。翻看其他同学的作业记录，我发现会有朋友为这本书的翻译提供建议，可能提到贡献者修改地很快，可能时间很长了，我也记不清是不是我通过的（哈哈）。写这本书的适合这样的建议非常重要，我一直没有机会感谢这些同学们。未来可能和一起翻译的几位朋友考虑出版这本书，虽然是x86架构的，一些Rust的思想也比较值得学习。未来几个月如果有机会，可能考虑抽时间修订这本书的代码和更多翻译，有机会就可以开始联系出版社了，翻译不精良，希望有出版社能高抬贵手吧，哈哈。

## 第2天

### 1. 适配中断和异常系统

非常感谢王润基学长把opensbi-rt交给我维护。我添加了内核中断（interrupt）的适配代码。
主要的思路是，通过llvm_asm内联中断上下文保存部分，然后导出所有的中断为函数，允许操作系统作者提供自己的中断处理函数；然后添加了一个总的trap分发器`_start_trap_rust`，它的设计有点像别的架构的中断向量表，中断发生时，能判断中断的类型，然后调用对应的处理函数。
如果用户没有提供自己的处理函数，opensbi-rt拥有默认的处理函数，方便操作系统作者忘记的时候调试。
参照昨天的`#[entry]`宏，我编写了`#[interrupt]`宏。功能和entry类似，它检查函数的输入类型。
Rust社区已有的中断实现通常没有参数，我在分发器函数里直接判断scause的值，然后映射对应的函数并调用。
`#[interrupt]`宏还能检查函数导出的符号是否正确（是否为6个可能的符号名称之一），因为如果导出名称写错了（通常出现的是少打了单词的s、er或者别的），是相当难发现的错误。使用这个宏能帮助操作系统作者在编译期找到错误。

编写`#[interrupt]`宏出现了一些名称冲突问题。因为往常的rust嵌入式生态，把这部分运行时分为两块，在PAC中实现了叫做`interrupt`的枚举类型，然后在`cortex-m-rt`此类的库中实现一个宏，它也叫做`interrupt`。如果做到同一个库里面，就会出现可能的名称冲突。
这可能是`cortex-m`架构在rust设计中，会把核心的中断和外设的中断一起设计，放在SVD文件生成的PAC库里面，所以绕过了问题。
社区的`riscv`库可能存在潜在的问题，因为`riscv`核心的中断有统一的标准，但机器M层上，`riscv`架构在M层的中断控制器实现有很多种，不会放到标准里面去，所以这个问题可能暂时还不明显。
我们基于S层的中断控制器，M层外设中断是由SBI完成的，S层不需要兼容不同的M层中断控制器，所以我们把枚举类型放到其它命名空间；或者如果将来放到一个命名空间，改用其它名字即可。

今天暂时没有设计`#[exception]`宏。`riscv`的trap有一些寄存器是需要保存的，有一些也许是不需要保存的（需要查资料）；修改CSR寄存器（比如sepc）的情况也需要仔细斟酌。
在这之后，才能确定函数的签名（输入类型、输出类型），才可以制作用于检查类型和名称的宏。可能需要查找资料，确定这些内容后，才能继续做下去。

制作了参考设计：[这里](https://github.com/luojia65/spicy-os)。这里实现了S模式的定时器中断处理器，还有ebreak异常的处理器。其中中断处理器使用`#[interrupt]`实现。因为移除了共同的运行时部分（到opensbi-rt里面），操作系统本身的代码非常简洁。

```rust
#[opensbi_rt::interrupt]
fn SupervisorTimer() {
    static mut TICKS: usize = 0;
    sbi::legacy::set_timer(time::read64().wrapping_add(INTERVAL));
    *TICKS += 1;
    if *TICKS % 100 == 0 {
        println!("100 ticks~");
    }
}
```

我这里吸收了RTIC架构的静态可变局部变量设计。Rust里static mut的全局变量都是不安全的，因为它们可能被不同的上下文（如不同线程、单线程环境下主程序和中断）访问，可能造成数据冲突。
但是在单上下文的情况，如只有本中断能访问的变量，我们会排除不同线程（因为只有作用域只在本中断内）、主程序和中断（主程序不能访问）两种情况，所以能够安全地提供仅限本中断访问的上下文变量，如例子中的`TICKS`。
我们可以把这个过程想像成，某一个中断的上下文被主程序切断了，但忽略主程序和其它中断的情况下，同一个中断本身共享同一个上下文。这样中断的上下文是独立的，就能拥有自己的局部变量。

```rust
// 主程序上下文：

println!("hello world!"); // 主程序只会运行一次
loop {}

// 主程序只会运行一次，我们可以定义某种意义的“静态”变量，这个用let语句
// 就可以实现。

// 中断上下文：

static mut TICKS: usize = 0;

*TICSK += 1; // 中断发生第一次
if *TICKS % 100 == 0 { println!("100 ticks!"); }

*TICSK += 1; // 中断发生第二次
if *TICKS % 100 == 0 { println!("100 ticks!"); }

*TICSK += 1; // 中断发生第三次
if *TICKS % 100 == 0 { println!("100 ticks!"); }

// 中断可能发生多次，但是因为同一个中断在同一个Hart上不可能同时发生，
// 所以访问静态的TICKS可以看作在同一个上下文内。
// 如果还用let语句，作用域就只在一次中断过程内，第二次中断就不能再访问第一次中断
// 保存的let变量。当然，一次中断结束后，let语句的变量将会被Drop。
```

这对于存储一些只有中断能访问的静态变量很有用，比如这里定时器的`TICKS`变量。
实现这个功能，就在宏里面遍历函数开头的所有`static mut a: T = b`，换成了`let a: &mut T = b`；这样就可以在函数内安全使用了。

### 2. 还需要完成的事情

RISC-V的trap分为中断（Interrupt）和异常（Exception）两部分。今天完成了trap上下文的保存，但设计函数部分，只完成了中断（Interrupt）的设计；而且上下文保存也有细节要斟酌，比如需要确认保存、还原哪些寄存器。
需要学习后面的内容，再和社区好友、朋友交流，确定异常（Exception）处理函数的签名、trap保存上下文的具体设计。

汇编语言方面有一些比较细的问题，比如32位和64位架构指令不同，代码里还有一两处需要讨论如何修改（保留栈空间，需要保留4n或者8n个的问题）。学习来说暂时是能用了。未来如果`asm!`宏稳定了，逐步把现有的`llvm_asm!`替换成更rusty的`asm!`。

明天争取实现`#[pre_init]`宏，这个宏对预加载动态内存管理器是很有用的。
