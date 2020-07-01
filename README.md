# DailySchedule

2020年六月

| 周一 | 周二 | 周三 | 周四 | 周五 | 周六 | 周日 |
|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
|    |    |    | 25 | 26 | 27 | 28 |
| 29 | 30 |    |    |    |    |    |

2020年七月

| 周一 | 周二 | 周三 | 周四 | 周五 | 周六 | 周日 |
|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
|    |    | 1  | 2  | 3  | 4  | 5  |
| 6  | 7  | 8  | 9  | 10 | 11 | 12 |
| 13 | 14 | 15 | 16 | 17 | 18 | 19 |
| 20 | 21 | 22 | 23 | 24 | 25 | 26 |
| 27 | 28 | 29 | 30 | 31 |    |    |

[第一天（2020年6月25日）](#第1天)

[第二天（2020年6月26日）](#第2天)

[第三天（2020年6月27日）](#第3天)

[第四天（2020年6月28日）](#第4天)

[第五天（2020年6月29日）](#第5天)

[第六天（2020年6月30日）](#第6天)

[第七天（2020年7月1日）](#第7天)

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
`#[interrupt]`宏还能检查函数导出的符号是否正确（是否为6个可能的符号名称之一），因为如果导出名称写错了（通常是少打了单词的s、er或者别的），是相当难发现的错误。使用这个宏能帮助操作系统作者在编译期找到错误。

编写`#[interrupt]`宏出现了一些名称冲突问题。因为往常的rust嵌入式生态会把这部分运行时分为两块：在PAC中，实现了叫做`interrupt`的枚举类型；在`cortex-m-rt`类型的库中实现一个宏，它也叫做`interrupt`。如果做到同一个库里面，就会出现可能的名称冲突。
这可能是`cortex-m`架构在rust设计中，会把核心的中断和外设的中断一起设计，放在SVD文件生成的PAC库里面，所以绕过了问题。
社区的`riscv`库可能存在潜在的问题，因为`riscv`核心的中断有统一的标准，但机器M层上，`riscv`架构在M层的中断控制器实现有很多种，不会放到标准里面去，所以这个问题可能暂时还不明显。
我们基于S层的中断控制器，M层外设中断是由SBI完成的，S层不需要兼容不同的M层中断控制器，所以我们把枚举类型放到其它命名空间；或者如果将来放到一个命名空间，改用其它名字即可。

今天暂时没有设计`#[exception]`宏。`riscv`的trap有一些寄存器是需要保存的，有一些也许是不需要保存的（需要查资料）；修改CSR寄存器（比如sepc）的情况也需要仔细斟酌。
在这之后，才能确定函数的签名（输入类型、输出类型），才可以制作用于检查类型和名称的宏。可能需要查找资料，确定这些内容后，才能继续做下去。

制作了参考设计：[这里](https://github.com/luojia65/spicy-os)。这里实现了S模式的定时器中断处理器，还有ebreak异常的处理器。其中中断处理器使用`#[interrupt]`实现。因为移除了共同的运行时部分（到opensbi-rt里面），操作系统本身的代码非常简洁。
参考设计里`wrapping_add`是为了防止时间计数器溢出（虽然要等很久很久才能看到这个现象……以防万一吧）。

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

{
    println!("hello world!"); // 主程序只会运行一次
    loop {}
}

// 主程序只会运行一次，我们可以定义某种意义的“静态”变量，这个用let语句
// 就可以实现。

// 中断上下文：
{
    static mut TICKS: usize = 0;

    *TICKS += 1; // 中断发生第一次
    if *TICKS % 100 == 0 { println!("100 ticks!"); }

    *TICKS += 1; // 中断发生第二次
    if *TICKS % 100 == 0 { println!("100 ticks!"); }

    *TICKS += 1; // 中断发生第三次
    if *TICKS % 100 == 0 { println!("100 ticks!"); }
}

// 中断可能发生多次，但是因为同一个中断在同一个Hart上不可能同时发生，
// 所以多次运行的中断过程，仍然可以看作在同一个上下文内。
// 这之后，访问静态的TICKS就可以看作是安全的。
// 如果还用let语句，作用域就只在一次中断过程内，第二次中断就不能再访问第一次中断
// 保存的let变量。当然，一次中断结束后，let语句的变量将会被Drop。
```

这对于存储一些只有中断能访问的静态变量很有用，比如这里定时器的`TICKS`变量。
实现这个功能，就在宏里面遍历函数开头的所有`static mut a: T = b`，换成了`let a: &mut T = b`；这样就可以在函数内安全使用了。

### 2. 还需要完成的事情

RISC-V的trap分为中断（Interrupt）和异常（Exception）两部分。今天完成了trap上下文的保存，但设计函数部分，只完成了中断（Interrupt）的设计；而且上下文保存也有细节要斟酌，比如需要确认保存、还原哪些寄存器。
需要学习后面的内容，再和社区好友、朋友交流，确定异常（Exception）处理函数的签名、trap保存上下文的具体设计。

`r0`库是一个非常小的现代编程语言运行时，它包含清零bss段、预先加载data段等最简单的功能。需要考虑把它放入opensbi-rt中。

汇编语言方面有一些比较细的问题，比如32位和64位架构指令不同，代码里还有一两处需要讨论如何修改（保留栈空间，需要保留4n或者8n个的问题）。学习来说暂时是能用了。未来如果`asm!`宏稳定了，逐步把现有的`llvm_asm!`替换成更rusty的`asm!`。

明天争取实现`#[pre_init]`宏，这个宏对预加载动态内存管理器是很有用的。

## 第3天

### 1. 切分opensbi-rt库

感谢学长的帮助，今天把opensbi-rt切分为了riscv-sbi-rt和riscv-sbi两个库。
riscv-sbi-rt库包含S模式下最小的SBI运行时；riscv-sbi库提供了SBI提供的标准调用实现。
这两个库未来如果做得好可以推到crates.io上去。
添加了一些SBI函数。测试下来似乎除了legacy，似乎没有被qemu支持？很奇怪，明天再试试看。

今天课业任务实在非常多，就先写到这儿了。

## 第4天

写了一白天的作业，这个学期算是结束了，可以专心写点操作系统了。

### 1. 制作#[pre_init]宏

这回把这件事情做完了，就是在运行时入口加一个`__pre_init`函数，它只会被一个核调用一次。
考虑后期可能移植到异构多核处理器，需要再加一个`_mp_hook`函数，确保只在一个核内返回true，
让初始化内存的核返回true。在`#[pre_init]`标注的函数内，我们可以初始化动态内存模块
（就是需要alloc库的模块），因为这个函数只会被一个核执行，保证它不会在多hart环境下被执行很多次。

`#[pre_init]`函数是unsafe函数，这是因为在这个函数执行之后，运行时才会调用内存初始化函数，
所以这里包括静态变量都是没有初始化的，可能读出意想不到的值，一些操作将是内存不安全的，
标一个unsafe开发者会注意这一点。

给一个一般使用`#[pre_init]`的例子：

```rust
#[pre_init]
unsafe fn before_main() {
    // 这里的代码会在main函数之前被执行。初始化内存分配器啊，这种，就可以放这里
    // 无论有多少个hart，都只会被执行一次。（相比于main函数，main会被每个hart分别执行一次）
}
```

`#[pre_init]`的设计在Rust的很多运行时库里都有，各种运行时，但是应用的运行时是没有的。
这是因为习惯上来讲，用户应用的内存应当由操作系统初始化，而不是应用自己初始化。（这句话说得可能不科学，需要翻阅资料）

### 2. 还需要做的事情

按照Rust社区开发运行时库的经验，`#[pre_init]`应该在内存初始化之前完成。
这里还要搞清楚`.data`段和`.bss`段的初始化过程。如果没有OS，包括S特权层运行时（OS运行时）和M特权层运行时
（裸金属运行时）的情况，这两个段都需要程序自己初始化。（应该是这样吧……）
Rust有一个很好的运行时段初始化库叫做`r0`，提供了初始化这两个段的函数实现，方便运行时库调用。
需要给出一个不初始化`.data`段会出事情的例子。应该说static变量使用会出问题，但是需要找一个合适的Rust例子。

如果用`r0`的话，需要给定`_sbss`、`_ebss`、`_sdata`、`_edata`、`_sidata`这些符号，
也许就需要把操作系统的链接器脚本包含到运行时里面了。这样操作系统需要给出内存区域的别名就可以了。
对写操作系统还是很方便的，不过学习怎么写就不方便了。可以在学完怎么手捏一个运行时后，再学习如何使用已有的运行时。

## 第5天

### 1. 分页内存系统

这个是操作系统的常规内容了。跟着教程做了一遍，需要更多的时间理清楚各个地址的关系。

### 2. 完善运行时

改善链接器脚本设计。现在把链接器脚本的`SECTION`部分移动到运行时里面，这里导出到`>REGION_XXX`为标记，
操作系统只需要在它的链接器脚本里添加`REGION_ALIAS`，就可以无缝使用运行时的相关内容。
操作系统可以根据设备规定定义内存块，放在`MEMORY`部分，然后把所有的程序模块输出到定义的`MEMORY`就可以了。
这个设计在嵌入式Rust里已经相当成熟，无论是单核、多核还是ARM、MIPS、RISC-V，都可以使用这样的设计模式，
实践证明能节省大量的开发时间。比如这样：

```rust
MEMORY {
    /* 起始地址 */
    DRAM : ORIGIN = 0x0000000080000000, LENGTH = 128M
}

/* 通常情况下都是从REGION_TEXT区域的起始位置运行的，但qemu的opensbi规定了入口位置，就把程序放在这里 */
PROVIDE(_stext = 0x0000000080200000);

REGION_ALIAS("REGION_TEXT", DRAM);
REGION_ALIAS("REGION_RODATA", DRAM);
REGION_ALIAS("REGION_DATA", DRAM);
REGION_ALIAS("REGION_BSS", DRAM);
REGION_ALIAS("REGION_STACK", DRAM);
```

QEMU SBI定义的内存块起始位置是固定的，这个似乎不容易在运行时确定，因为地址至少在烧录固件的时候已经明确。
这里需要在`REGION_ALIAS`里面给定需要放到的内存块名称，因为很多真实设备的内存块不止一个，还不一定是连续的，
甚至有可能是多重映射（比如一个内存快不经高速缓存和经高速缓存，可以通过不同的内存地址访问来区分）。
内存块的属性也会不同，比如闪存和SRAM的速度和性质就不一样，适合放置不同的程序段。
如果未来操作系统迁移到其它硬件，只需查阅手册，提供每个程序段对应的内存位置就可以了。

关于链接器脚本命名和引用的问题。咨询社区成员后，社区的工作经验一般这样命名：`.ld`文件代表完整的链接器脚本，
而`.x`代表脚本的一部分。单独的`.x`文件不能置入链接器工作，因为它可能缺少必要的信息，需要其它`.x`脚本一起
配合使用，才能构成完整的链接器脚本信息。这里把原来的`sbi.ld`拆分成`sbi64.x`和操作系统自己实现的`.x`文件。

关于`r0`库。phil-opp博客文章的有提到，像是Rust的高级语言运行时，都是从“零运行时”开始的；
这个零运行时或者是操作系统提供的，通常叫`crt0`，或者是运行时库提供的。
高级语言运行时至少应当初始化数据段、清零bss段。我们不用自己撸一个，可以使用现成的`r0`库。
`r0`库是一个大量rust运行时库常用的依赖库，由Rust-Embedded的四个小组共同维护，已经运用在大量嵌入式Rust项目中，
设计已经相当成熟。使用`r0`库主要需要两个函数：`init_data`和`zero_bss`，恰好对应最小运行时的两个条件。
因为`r0`库不能选择使用`usize`长度初始化，我们分别使用`u32`和`u64`类型初始化，
在程序里用`#[cfg(target_pointer_width = "32")]`这样的代码条件读取链接器脚本提供的`_sbss`等等地址，
读取为`u32`或者`u64`类型，然后直接传入这两个函数完成初始化。
初始化过程只能由一个Hart完成，我们将它加入`pre_init`之后，这样恰好`pre_init`就能在初始化之前完成部分过程。

关于`r0`的数据对齐问题。`r0`初始化数据是按单位复制的，比如`u32`就是4字节一单位，`u64`就是8字节一单位。
`r0`没有提供`usize`长度初始化（需要询问社区这样设计的目的），可能想到的方法是提供两个不同的链接器脚本，
一个将`.data`、`.bss`按4字节对齐，一个按8字节对齐，这样初始化的速度较好。初始化速度在整个操作系统启动的过程
中影响不大，无论XLEN全部处理成4字节对齐也不是不可以。这里就提供这两种设计思路。

为了判断由哪个Hart完成，这里引入`_mp_hook`函数。这个函数返回一个`bool`，需要保证只在一个Hart返回true，
其它Hart都返回false。返回true的Hart将处理`pre_init`函数和`r0`库的初始化过程。
运行时给出一个默认实现`default_mp_hook`，而且定义这个符号是弱符号，这样就允许操作系统重写函数，委派自己需要的核。
原来的代码这里直接判断hart编号是否为0；有些非对称设计的情况，需要用其它Hart启动，就可以重写函数。
默认实现里还有一个待斟酌的地方：是否需要为不参与初始化的Hart执行`wfi`函数。虽然合法的`wfi`函数实现之一是`nop`，
但实现了`wfi`的情况也略有区别。如果不执行，相当于所有的核将会直接执行main函数。如果执行，只有初始化核会继续运行，
它需要使用S层的软件中断`sbi_send_ipi`来启动对应的核，但需要实现软件中断的处理函数。
需要参考成熟的设计再做考虑。

今天做了一个内联汇编的小优化。`.equ XLEN, 8`是我从rCore项目里学到的代码，因为内联汇编里用#define会出现问题，
所以可以用这个来代替。可以在32、64位系统下条件编译，然后再共享一段共同的代码，这样可以节省代码量。

### 3. 提供建议：Cargo、`build.rs`文件与链接器脚本

不管是`.x`还是`.ld`，目前使用它们的方法还只能在`.cargo/config`里面添加这样的代码：

```toml
[target.riscv64imac-unknown-none-elf]
rustflags = [
    "-C", "link-arg=-Tspicy-os/linker64.x",
    "-C", "link-arg=-Tsbi64.x",
]
```

会发现这部分非常复杂，还需要自己查找链接器文件的名字，这就非常头疼，调试也不好调。无论如何我们`build.rs`
文件还是要用的，因为这样我们检查项目（cargo check）的时候，或者修改了链接器文件的时候，我们把文件添加
到`build.rs`里面，就能提示Rust重新运行脚本并编译，避免我们改了代码Rust游不识别的问题。比如这样：

```rust
fn main() {
    let out_dir = PathBuf::from(env::var("OUT_DIR").unwrap());

    // Put the linker script somewhere the linker can find it
    fs::File::create(out_dir.join("sbi64.x"))
        .unwrap()
        .write_all(include_bytes!("sbi64.x"))
        .unwrap();
    println!("cargo:rustc-link-search={}", out_dir.display());

    println!("cargo:rerun-if-changed=build.rs");
    println!("cargo:rerun-if-changed=sbi.ld");
}
```

所以找链接器文件最好的办法还是在build.rs文件里提示Rust编译器。翻阅The Cargo Book第3.8章，发现了
`cargo:rustc-flags=FLAGS`配置，但是它只支持`-L`或`-l`，不支持`-T`来添加链接器脚本。
如果使用另一个配置，效果也不好，因为它只支持`cdylib`，和我们的目的不符。
翻阅issues，发现cargo工具链[有人交过这个提案了](https://github.com/rust-lang/cargo/pull/7811)。
如果这个有进展，我们就可以在`build.rs`里面添加：

```rust
println!("cargo:rustc-bin-link-arg=-Tsbi.ld");
```

就不需要再动`.cargo/config`了。甚至可以根据32位和64位目标自动选择合适的链接器脚本，这都是`build.rs`
能做到的。

### 4. 关于`#[linkage = "weak"]`

这个如果做到运行时库的函数里，就不需要再在链接器脚本里`PROVIDE`了。而且原来的函数如果被重写，也不会被编译进去，
这就十分舒服，可以省一些空间。但我翻阅issues，发现这个功能的实现暂时基于llvm，因为Rust编译器也有别的后端
（比如wasm的以前叫Cranelift，现在似乎改名了，总之不是llvm），如果做起来这些后端可能有点难适配。
我的想法是Rust语言是否能添加一个新的`#[weak]`标签，希望达到相同的效果，而且也不是llvm特有的。

Issue地址在[这里](https://github.com/rust-lang/rust/issues/29603)。好像2015年就遇到这个问题了，
现在还没有个好的解决方案。我暂时的解决方案是，不要在Rust里写这些函数，放在汇编代码里面。
或者占一点空间就占一点，为默认的实现起一个新的名字，不放`weak`标签，然后在链接器脚本定义`PROVIDE`，
这样运行时就可以检测到用户覆盖的函数，或者找不到就用默认的函数实现。

如果这个issue有一天被实现，那最好用这个issue提供的方法，这就会比现在临时的解决方法要好很多。

### 5. 待解决的问题

今天调试的过程遇到很多诡异的现象。比如在debug下运行竟然会出现非法指令异常，然后换成release竟然异常就没了。
这个在我尝试把内存分配器从运行时移动到操作系统后，调用Box::new函数里会出现。暂时Rust调试qemu操作系统的方法
还需要摸索，这一点调试非常困难，就很奇怪。以前我做K210硬件可能出现过类似的问题，就是串口的println!，
在debug模式下会莫名其妙死机（可能就是进异常了），然后删除这个println!之后就没有再出故障。
今天忘记记录了，有一个情况竟然会执行std::hint::unreachable_unchecked，这让我非常意外，如果下次遇到这样
的情况需要注意记录，以便排查。本来要完成移除内存分配器，改让操作系统自行选择，过两天继续寻找解决方案。

另外就是在IDE环境下不容易用`rust-analyzer`做实时检查。这个工具在vscode上常用，它似乎是通过`test`特性实现
检查的，但是在没有`std`的目标下不存在`test`框架，在`.cargo/config`定义目标后就出现无法检查的问题。
Phil-opp的博客提供了一个[自定义测试框架的方法](https://github.com/phil-opp/blog_os/blob/788d6a7e2295cc45cf327637c0ba58638d5f1346/blog/content/second-edition/posts/04-testing/index.md#custom-test-frameworks)。但这个方法[一直没有稳定](https://github.com/rust-lang/rust/blob/9ebf47851a357faa4cd97f4b1dc7835f6376e639/src/doc/unstable-book/src/language-features/custom-test-frameworks.md)，最好等到稳定后再考虑此种思路。
现在实时检查几乎很难使用，如果不在`.cargo/config`定义目标，它默认使用当前系统，但`riscv`定义的结构和汇编代码
就会被归为语法错误。如果在文件里定义目标，又因为没有test框架不能使用。暂时还是不好解决的问题。

今天多次添加`REGION_HEAP`段又删减，是因为还没有考虑它和物理内存的联系。
后续的过程里尝试继续彻底摸清楚物理内存下的操作机制，希望给出一个较好的设计。

### 6. 一些关于烧录的想法

以前我曾经尝试自制烧录软件，最终社区制作的[`probe.rs`](https://probe.rs)和`cargo-flash`完成度远远超过
我的实现，就使用他们的烧录软件了。我发现制作操作系统过程中，SBI实现部分和操作系统部分是完全不同的两块内存内容，
很可能是互不干扰的，甚至可以考虑分别烧录。未来为一个新硬件开发操作系统，先实现SBI，把SBI部分烧录进去。
然后套用已有的操作系统，直接编译后烧录，而且不需要覆盖先前SBI占用的部分。然后实现应用程序，如果要硬编码到SoC里面
（而不是从SD卡这样的地方读取），可以直接烧到相应的地址，不需要覆盖SBI和操作系统部分。
这样就能节省很大一部分的烧录时间，加快开发和迭代流程，也是多层级开发可能带来的好处。

## 第6天

### 1. 实现页表和内核重映射

RISC-V这部分据说以前标准有一些问题，现在的标准应该是已经解决了。今天跟着教程把流程过了一遍。
虽然一直到晚上还会出现Page Fault，这部分问题明天继续处理。

### 2. 关于Rust标准库的Range迭代器

遇到Rust的问题一个是标准库的`Range`通常下没实现`Iterator`，标准库里没有稳定的做法是给出一个`Step`
trait。我尝试做一个归纳，对于类型T中的变量t，一旦实现这个trait，就等效于：

1. 对任意t属于T，t的前驱有唯一的定义或者为None，后继也有唯一的定义或者None
2. 有且仅有一个t属于T，它的前驱是None，定义这个t为类型的“起点”
3. t的后继定义为“第1后继”，当n是自然数，t的“第n后继”定义为t的“第n-1后继”的后继，当“第n-1后继“不为None；同理定义t的“第n前驱”。且满足对任意t2是t1的“第n后继”，都有t1是t2的“第n前驱”，反之亦然
4. 对任意t1和t2都属于T，定义t2和t1的“距离”：设n是自然数，t2和t1的“距离”为n，当且仅当t2是t1的“第n后继”

则由规则1可以单步遍历Range，有前驱的特性可以反向单步遍历。由规则2，类型里的所有值不会是一棵树，不会有环，也不会是多条链。
再由规则3，可以跳步遍历和反向跳步遍历（`nth`)。由规则4，可以求得Range剩余的元素数。
满足了这些规则，就有点像某种离散值区间的描述，如果描述得没有问题，可能就是目前Step trait给出的抽象方法了。
由Rust规范给出的迭代器的定义，如果实现了单步遍历就能实现Iterator，当然可选的跳步遍历能提高速度，
可选的剩余元素数可以给编译器更好地提示（`size_hint`)。如果实现了反向遍历就能实现DoubleEndedIterator，
当然反向跳步遍历也能提高速度。
相同地思路，还能知道这个迭代器一定是FusedIterator，就是它是有终点的，这里就不描述了。
所以一旦满足Step，就能方便地遍历一个离散值区间，这就和操作系统开发里的地址空间能套上使用了。

这个Step约束[很早就有人提出来了](https://github.com/rust-lang/rust/issues/42168)，
也许是因为`for i in 0..5`这样最简单的Rust语法中其实也隐含了这样的Step iterator思想。
目前还只在Rust标准库内部使用，用来为最简单的primitive type实现迭代器。
目前替代的方法是自己制作一个Range结构体；或者在Range外面套一层壳，为它实现迭代器。
如果这个约束的实现稳定化有进展，我们可以用来实现物理地址、虚拟地址以及两种页号的遍历工作，
就能剩下一些思考量，这比替代方法要好很多。

### 3. 解决了很多奇怪的问题

昨天写运行时的时候遇到一些奇怪的问题，比如莫名其妙的非法指令异常、LoadFault异常。我发现原因可能是这样的：
RISC-V为了节省代码量，为我们提供了`gp`寄存器，因为地址很高的情况下需要好多条指令才能加载地址数（尤其是64位下），
我们就能首先保存`gp`的值，然后就能用基于`gp`的寻址了，这样只需要`ld`或者`sd`一条指令。
这个寻址能寻找正负0x800个字节的信息，所以最好把它放在数据段基址加上0x800的位置上。
这样负0x800就能映射到数据段基址，正0x800就能映射到0x1000的位置上。比如我们这样做：

```rust
    /* .data 字段 */
    .data : ALIGN(4K) {
        _sidata = LOADADDR(.data);
        _sdata = .;
        /* Must be called __global_pointer$ for linker relaxations to work. */
        PROVIDE(__global_pointer$ = . + 0x800); /* <-- 这里 <-- */
        /* 要链接的文件的 .data 字段集中放在这里 */
        *(.sdata .sdata.* .sdata2 .sdata2.*);
        *(.data .data.*)
        . = ALIGN(4K);
        _edata = .;
    } > REGION_DATA
```

这部分的处理是链接器完成的。但是如果我们没定义`gp`，它的值也许是任意的，在模拟器里值可能是零。
猜测是因为没有禁用relax（`.option norelax`），代码仍然根据`gp`的基准计算偏移，可能会出现问题，
这个就需要复现昨天的问题再调查原因了。

（不过后来这样的问题再也没在我的电脑上出现……也许需要特定的代码才会出现，如果后续有朋友开始做这块内容，
遇到了这个问题，尝试把`gp`的定义和实现都这样加上去看看。）

另外，定义`gp`的值之后，需要在运行时加入这样的代码：

```text
    .option push
    .option norelax
    la gp, __global_pointer$
    .option pop
```

禁用relax，是因为`gp`本身没法根据自己计算偏移。这样`gp`的值就能在运行的时候加载到寄存器了。

### 4. 多核钩子、切割栈设计

这里参照社区库的设计，为每个Hart切割属于自己的栈。原来栈的大小和位置都是在汇编代码里写死的，现在不用写死了。
这部分Hart的数量（`_max_hart_id`）和每个Hart栈的大小（`_hart_stack_size`）都能在OS的链接器脚本里定义，
而且运行时库会帮你计算空间够不够。运行时的汇编代码会判断Hart的数量，如果数量不对就会停机（abort）。

计算每个Hart的栈地址需要使用乘法。有乘法M指令集的核好说，没有M指令集的核上就用循环加法代替，Hart的数量也不会多，
占用的时间不会很长。

多核钩子`_mp_hook`的设计也参照了社区的库。标准库给出的设计只会在Hart 0返回true，将完成初始化过程和main函数，别的核都停机。
用户如果想使用多核系统，应该覆写这个函数，虽然也只在第0个hart返回true，但其它hart不再直接停机。
而是清除自己的软件中断标记，再等待第0个核调用软件中断，让它启动。这里给出简单的`mp_hook`实现：

```rust
#[export_name = "_mp_hook"]
pub extern fn mp_hook(hartid: usize, _dtb: usize) -> bool {
    if hartid == 0 {
        true
    } else {
        unsafe {
            sbi::legacy::clear_ipi();
            sie::set_ssoft();
            loop {
                riscv::asm::wfi();
                if sip::read().ssoft() {
                    break;
                }
            }
            sie::clear_ssoft();
            sbi::legacy::clear_ipi();
        }
        false
    }
}
```

清除软件中断标记部分需要调用`riscv-sbi`的函数。当第0个核完成main函数的启动步骤后，main函数需要这样运行：

```rust
#[entry]
fn main(hartid: usize, dtb: usize) {
    if hartid == 0 {
        // 1. 完成只运行一次的初始化过程（略）
        // 2. 把别的核唤醒
        let hart_mask = 0b1110; // todo 修改设计
        sbi::legacy::send_ipi(hart_mask);
    }
    // 3. 初始化完成后，所有核都会运行的代码（略）
}
```

运行后，所有核都会运行应该运行的代码了。其中编号非零的核只会运行3，而零号核会运行所有的1、2和3部分。
这里`send_ipi`函数的设计应该接受一个位集（bitset），而不是一个数字，需要后续时间的实践后再重新考虑。
只运行一次的初始化过程包括动态内存的初始化（比如预加载`HEAP_ALLOCATOR`）。这些只需要运行一次，而且不适合
放到`pre_init`里面，因为它们可能操作未加载的静态变量，但是这些变量在随后`r0`的初始化过程被覆盖了，
相当于内存分配器又回到了没有初始化的状态。所以应该在`main`函数里初始化动态内存分配器，而不是前几天设想的`pre_init`
函数里面。

### 5. 还需要做的事情

今天尝试把内存分配器从运行时库里移出来，解决了奇怪的问题后成功了。后续再考虑要不要把内存分配的错误处理器也拿出来
（就是那个`oom`函数）。内核内存溢出如何处理可能是涉及用户交互和工程学的问题，这个需要做一定经验后，再决定哪种设计更方便。

稍微吐槽一下`riscv`库的`satp`寄存器。因为`docs.rs`网站编译文档的时候不会编译到RISC-V目标下（都是些linux啊这种目标），
它`#[cfg(riscv32)]`和`#[cfg(riscv64)]`都是不满足的。这样就导致`satp`寄存器的实现内容其实很丰富，但是在`docs.rs`
下就只有一点点内容，还可能有些诧异为什么要设计成安全的函数（而不是unsafe），但翻阅代码才发现函数全都是齐全的，
就是在`#[cfg(riscv64)]`下面了，`docs.rs`文档不显示。建议修改为`#[cfg(not(riscv32))]`，这样`docs.rs`
上面文档就有了（别的寄存器就是这样设计的，毕竟RV128这事儿还远着）。下次需要修改`riscv`这个包的时候，顺便完成这项小工作。
或者可以建议`docs.rs`添加一些功能，允许我们给定它需要编译的目标，而不是只能在服务器支持的目标里面选。

暂时分页内存部分的设计，还有一些是写死在运行时里面的。明天要把分页内存部分完成，相关的设计尽量修改收尾，然后有时间就往下一章推。

## 第7天

### 1. 关于运行时`_start`部分的地址映射

这两天频繁出现的问题和这个有关系。我们需要注意两个地址，一个是`pc`，一个是`sp`。
可能和`ra`也是有关系的，但是考虑到操作系统的代码不需要返回，这里就不考虑main函数的`ra`了。

进入main函数之前，`pc`和`sp`都应该是虚拟地址`0xffffffff80xxxxxxx`。
如果`pc`或者`sp`还是物理地址的空间`0x0000000080xxxxxxx`，代码还是能运行的，
动态内存和内存页分配器也是能运行的。这似乎是因为我们在初始页里搭建了`0x0000000080xxxxxx`
到它自身的映射，为了解决教程里题到“尴尬的情况”；但这个映射在`_start`里面页表刷新后依然存在。
所以这个部分`pc`和`sp`假如还是物理地址，依然可以运行下去。

问题出现在分页内存系统实现。如果`pc`和`sp`还是物理地址，我们分完的页都是基于链接器脚本的。
链接器脚本的设定都是虚拟地址`0xffffffff80xxxxxxx`的区间，也就是说，`_stext`、
`_etext`这些全都是虚拟地址的区间，我们分页保护的也是这些区间（而不是物理地址区间）。
这时候，分页系统不再包含初始页面里`0x0000000080xxxxxxx`到它自己的映射。
此时刷新TLB后，假如`pc`或`sp`还是物理地址，就将在页表中不存在，会出现Page Fault。
有时候也许会注意到Page Fault还会是嵌套的，这是因为我们提供了异常（Exception）处理器，
它会基于`sp`保存当前异常的上下文。然而，如果`sp`不在虚拟地址空间，保存上下文这个动作本身
也是无法完成的，因为比如`sd x8, 8(sp)`指令保存上下文的值，就会用到`sp`。
所以这里`sp`产生的错误会一直嵌套，无法进入异常处理在上下文保存后的正文部分。

所以我们在进入`main`函数前，需要明白物理地址到自己的映射是为了执行“尴尬的代码”的。
我们需要确保`pc`和`sp`都在虚拟地址空间里，才能进入`main`函数。
如果`sp`暂时保存在物理地址空间，也是能进去的，不过需要在页表系统刷新快表后，手动调整`sp`
的值。考虑到刷新快表次数会有很多，很难判断`sp`是第一次调整还是后续的调整（第一次需要
一个加减法指令，后续无需指令），写代码就会非常麻烦。所以建议在`main`函数之前就调好，
这样操作系统里面的函数就不需要担心这个了。

归纳一个一般的规律：如果出现嵌套的分页错误，或者访问了莫名其妙的地址，
一般的方法（输出调试等）非常难以定位问题，而且可能花费大量时间。
开发者更需要首先考虑的是`sp`和`pc`是虚拟地址还是物理地址。
建议使用gdb配合qemu下断点，观察地址的变化，在进入main函数前检查`sp`和`pc`的值。
最好的设计是在运行时中全部调整为虚拟地址后，才能进入`main`函数。
如果不符合这一点，就需要调整运行时的设计思路。

### 2. 阅读OpenSBI源码

运行时可以返回SBI吗？似乎不能，虽然它给`ra`设了一个值，但是用`ret`跳转到这个地址，
（当然，首先要先清空`satp`寄存器，然后刷新TLB，让页表恢复到直接映射物理地址的状态）
用GDB观察，它似乎只会在`0x80000000`定义给SBI的地址里相互跳转，没有观察到输出或者特殊的行为。
OpenSBI对应的源码可能在[这里](https://github.com/riscv/opensbi/blob/51f0e4a0533fe8b5d713379ab3a6cb676add82da/firmware/fw_base.S)。
暂时这部分设计为不可返回的（返回Never类型），只能由运行时调用SBI退出。

猜测其它SBI设计可能和此类设计相似，都是不允许操作系统的main函数返回，需要调用SBI函数退出。
需要阅读更多代码后才能下结论。

在这之外，观察到OpenSBI会初始化它自己占用的bss段和data段。这也许说明我们在操作系统运行时层面，
也应该清空操作系统的bss、初始化操作系统的data，这部分关于`r0`的设计应该是没有问题的。

### 3. 关于Make和Just

两者都是很好的构建工具。经过这几天开发操作系统的实践经验，Just的脚本`justfile`
不需要特别注意Tab还是空格，也不需要添加`.PHONY`，语法保留Make的熟悉程度同时，也和Rust比较相似。
编写Make脚本时，好不容易把脚本写完了，才发现Tab和空格的使用还有规则，又要查资料改好一会儿。
然后好不容易改好了，又发现还有`.PHONY`这种很容易忘了写的东西。
所以相对Make脚本，Just脚本和Make语法非常接近，几乎可以说是无缝衔接。
Just脚本经过长期的工程调整，不需要学习复杂的使用规则，编写脚本速度更快。
尤其是开发者着急地要添加一些内容的时候，不需要被琐碎的问题打扰，效率更高。

Just还有一个好处就是兼容所有的平台。比如Windows上是没内置Make的，想用Make就不得不安装
非常庞大的msys、wsl或者minGW等等环境。相比之下，在所有平台安装Just的方法是很简单的。
只需要输入以下代码：

```bash
cargo install just
```

就可以在任何已经安装Rust的电脑上安装Just。

鉴于以上因素，这里强烈建议在后续的项目中使用Just代替Make，一是在全平台的兼容性，二是开发效率更高。
Just的项目页面在这里：[https://github.com/casey/just](https://github.com/casey/just)。

这里给出适用于本次操作系统开发的Just脚本：

```makefile
target := "riscv64imac-unknown-none-elf"
mode := "debug"
kernel_file := "target/" + target + "/" + mode + "/spicy-os"
bin_file := "target/" + target + "/" + mode + "/kernel.bin"

objdump := "rust-objdump --arch-name=riscv64"
objcopy := "rust-objcopy --binary-architecture=riscv64"
size := "rust-size"

build: kernel
    @{{objcopy}} {{kernel_file}} --strip-all -O binary {{bin_file}}

kernel:
    @cargo build --target={{target}}

qemu: build
    @qemu-system-riscv64 \
            -machine virt \
            -nographic \
            -bios default \
            -device loader,file={{bin_file}},addr=0x80200000 \
            -smp threads=1

run: build qemu

asm: build
    @{{objdump}} -D {{kernel_file}} | less

size: build
    @{{size}} -A -x {{kernel_file}}
```

在项目根目录下，保存为名称为`justfile`的文件就可以了。

然后如果我们要运行内核，只需要输入：

```shell
just run
```

就和`make run`比较相似。类似地，还可以执行`just build`、`just asm`和`just size`等等指令。

如果还需要GDB调试支持，还要添加以下内容：

```makefile
gdb := "riscv64-unknown-elf-gdb"

debug: build
    @qemu-system-riscv64 \
            -machine virt \
            -nographic \
            -bios default \
            -device loader,file={{bin_file}},addr=0x80200000 \
            -smp threads=1 \
            -gdb tcp::11111 -S
gdb: build
    @gdb --eval-command="file {{kernel_file}}" --eval-command="target remote localhost:11111"
```

两个端口号`11111`只需要相同就可以了。
除非同时调试两个操作系统，一般保证这个端口暂时没有被占用即可。

### 4. 还需要做的事情

目前运行时里对内核初始分页还只是简单的包装。需要考虑这部分的工程设计，需要导出函数给操作系统完成，
还是我们提供一个固定的实现。尤其是在main函数之前调整`sp`和`pc`的值，这两个很重要。

后续还需要在更多时间中，提炼分页系统最好的工程设计。这部分本来设想放到运行时库里面，
可能最最基础的数据结构（页表项PTE这种）可以放进去，实现还是由操作系统自己完成。
等成熟的设计出现后，再考虑这一点。
