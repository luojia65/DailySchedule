# 第二阶段实验日志

## 第20天

今天下午应该有开会，我有考试，可能不参加了，请了徐文浩巨佬代替我。我在几天的工作里帮了他不少忙。

## 第19天

今天试了怎么调试PLIC的中断。其实最好的方法是调用一个S层提供的函数，如果这个能委托到S层也是条思路。
处理PLIC中断暂时的思路还是先关闭硬件中断开关，处理结束后再打开来。
比较麻烦的一点是PLIC中断有占有和完成两部分，进中断函数后先占有、配置优先级这个好说，
退出中断后要完成这事儿就很麻烦了，想不到好的处理方法。后面再考虑这块怎么写。

SD卡的支持应该放在操作系统里面。外设口的速度是可以往上调整的，有个外设口SPI3竟然可以到100MHz的时钟速度，
这个后面的设计里再考虑怎么处理这东西。这个外设口还支持直接从外挂闪存里执行代码，这也是需要调整的部分。

## 第18天

### 1. k210-hal库的串行外设口支持

只写了串行外设口（SPI）的创建部分。

创建一个串行外设口，需要外设的所有权实例，串行外设的接线方法，接口的模式（是所有的串行外设口统一的），
还有数据的端序。外设口的频率是可以动态调整的，比如初始化SD卡时需要固定的频率，后面就可以提高频率
来提高通讯速度。

需要修改大量的寄存器。K210的外设口寄存器很多，可以分为几类，包括中断控制、直接内存访问（DMA)、
从机配置、控制寄存器、非标准控制寄存器、端序控制器这些。
控制寄存器其实是可以动态更改的，比如里面有数据的字节长度等等。这个要做相应的工程设计来支持。

关于k210-hal库暂时还不能在S层使用，因为它的pac部分依赖于关闭中断，这需要M层的特权。
我需要和社区朋友交流，探讨一个最好的解决方案。他们提出的方法是换用关闭S层中断来实现这一点，似乎是可以的。

### 2. 关于M层到S层的时钟

主时钟、总线时钟在M层调整，外设时钟在S层调整。这个架构是比较合理的，因为外设时钟的调整是常见的需求，
比如串口的波特率允许动态去调。M层调整之后，传递设备树到S层。S层读取设备树，直接获取Clocks结构体，
从而用于外设时钟的调整工作。Clocks结构体的设计恰好满足了这个条件。

## 第17天

今天做了一些操作系统上的一些收尾事项。

### 1. 模拟sfence.vma指令

我们希望操作系统使用1.11版本最新的特权指令集文档。然而K210提供的是1.9版本的旧文档。
这里就有一点不同：新版本的sfence.vma和旧版本的sfence.vm。他们的区别是这样的：

| 位区间 | sfence.vma (1.11) | sfence.vm (1.9) |
|:--:|:--:|:--:|
| 31..=25 | funct7 = SFENCE.VMA(0001001) | funct12 = SFENCE.VM(000100000100) |
| 24..=20 | rs2/asid | funct12 = SFENCE.VM(000100000100) |
| 19..=15 | rs1/vaddr | rs1/vaddr |
| 14..=12 | funct3 = PRIV(000) | funct3 = PRIV(000) |
| 11..=7 | rd, =0 | rd, =0 |
| 6..=0 | opcode = SYSTEM(1110011) | opcode = SYSTEM(1110011) |

输入sfence.vma的字节码，K210会提示不存在这个指令，从而陷入非法指令异常。
其实我们可以利用它的这个特性。只要陷入非法指令异常，就获得他的这个指令，
然后判断是不是sfence.vma，如果是，就执行一条sfence.vm，然后就相当于完成了刷新工作。

需要注意的是，1.9版本的特权级文档中，不会把造成陷入的指令存到mtval里。
所以我们要根据mepc的值手动访问得到这条指令。因为mepc在这时候是虚拟地址，我们不用把它转换成物理地址，
而是直接通过设置mstatus的第17位为1，来配置为通过页表转换系统访问这个地址。
这个位无论是1.9还是1.11定义都是相同的。先设置它为1，然后直接用lw、lh等等访问，然后重新设置成0就可以了。

K210的内存不能非对齐访问，比如从对到16位的地方拿32位的数据，这就会出现那种非对齐访问异常。
这个异常是可以用陷入捕获的，只要能把panic信息输出，很容易定位问题。遇到这个异常怎么办呢，
直接通过访问当前位置和当前加16位的位置，得到两个16位地址，再拼接成32位就可以了。
这里需要注意端序的问题，不过实测K210的内存访问是小端序，所以大地址在高位，小地址在低位，
位移做一做就可以了。RISC-V的指令一定是对齐到16位地址的，就不存在8位地址有关的不对齐的问题。

我觉得推广一下，可以用软件模拟所有非对齐内存的访问，不过这是很后面的事情了。暂时不管这个问题。

### 2. 关于1.9版本的sptbr（satp的位置）

本来我想通过每次刷页表sfence.vma的时候，从“satp寄存器”读最高位的MODE位，
然后填入1.9版本mstatus的VM位，达到启用页表的目的，这刚好陷入的时候在M层，就能操作这些地方。
然而经过测试，K210的sptbr寄存器只有最低20位是可以写的，也就是说高44位怎么写读回来都是0，
那就没办法读MODE位在哪了。所以我们决定无论操作系统尝试写怎样的MODE，
我们总是使用Sv39，这样在K210下是够用的。

需要注意的是1.9版本的MODE和1.11版本是不同的。Sv39在1.9下编号是9，但在1.11下编号是8。
这个是一开始没有想到的，我还以为是一样的，查了才发现问题。

做一个64位下，两个版本0x180位置CSR寄存器的对比：

| 位区间 | satp (1.11) | sptbr (1.9) |
|:--:|:--:|:--:|
| 63..=60 | MODE[0..=4] | ASID[23..=26] |
| 59..=44 | ASID[0..=16] | ASID[7..=22] |
| 43..=38 | PPN[39..=44] | ASID[0..=6] |
| 37..=0 | PPN[0..=38] | PPN[0..=38] |

K210下面，高44位永远都是0，也意味着没法使用地址空间编号功能（asid这玩意）。
我觉得这个设计可能和低20位向左移动12位只有32位有关，刚好能覆盖K210可以涉及的物理地址。
如果地址空间编号总是零，可能在写sfence.vma的时候，这一块的参数可以直接省略掉。
也就是说其实我们可以不动sptbr的值，完全把它当satp使用；或者以后再考虑这块怎么写。
这样只需要在1.9版本的mstatus里面开启页表就可以了。

### 3. 开启页表和汇编指令

这个也好说，读mstatus，先清掉28..=24位，然后写一个9进去，就代表开启Sv39。
需要注意的是，现在的Rust语言已经淘汰了sfence.vm指令，我们一定要写这个指令，
只能在这里放入对应的常数值来代替指令。比如

```rust
.word 0x10400073
```

就代替了`sfence.vm x0`。这也和我们常用的刷新指令格式`sfence.vma`对应上，
如果两个参数都没提供，就说明两个都是x0寄存器。然后把mepc加4跳过原来的指令，刷新页表操作就成功了。
这样，我们在操作系统里输入`sfence.vma`，就能无缝使用K210提供的刷新页表指令。
这时候RustSBI就能代替OpenSBI跑完所有的实验了，包括最简单的虚拟内存，到最后的用户进程啊，这些。
后面就可以开始从SD卡读取用户程序了。

## 第14-16天

做了些别的事情。学校实验室的一些工作收尾了，下一阶段做别的。

## 第13天

今天和老师同学们交流，项目的进展非常顺利。虽然很多同学要考试，后面更复杂的系统就不容易做了。
我后面要做的是在SBI实现里集成一个外接存储卡支持，试试看怎么做。
然后设备树部分，希望有时间继续看文档，或者我觉得花的时间太久了，干脆先放弃，做别的。

## 第9-12天

学校实验室有很多工作，最近要发论文了，做了非常久

## 第8天

今天仔细阅读了设备树格式的文档。设备树它是有一个[官网](https://www.devicetree.org/)的，里面有一些很详细的内容，
包括最新的标准下载，等等。虽然标准没有规定设备树格式里具体的适配名称，但是Linux系统的主分支代码里有详细的规范。
综合代码和规范的描述，我们可以说SBI暂时的这套标准是和Unix类系统非常接近的。未来的UEFI标准可能仍然沿用PE格式。
本次的实验我们跟随Unix类的风格，尝试给出K210所有的外设等等，方便开发系统。

Linux的主分支是支持K210芯片的。K210的外设它有一个简单的设备树文件，翻阅文件，发现里面只描述了部分外设，
而且没有撰写内存管理单元的部分。我们的操作系统需要使用虚拟地址，就需要用这一部分的支持。
这包括后面的外设，一起写到新的设备树文件里面。

编译到K210上，我们还需要二进制的设备树描述。暂时我们不得不使用设备树编译器完成这个步骤，
这个编译器非常简单，实现的代码也非常少，它是一个必需品，优点很多，唯一的缺点的话它是用C语言写的，这次实验结束后再考虑这个问题。
暂时使用已有的dtc编译器。

然后就是在代码里面的设计了。可以编译用宏放进源代码里。然后处理器的主频这些，是允许代码自己修改的，
这部分可能要融合到设备树的设计里面，可以用我配置的参数定义这个设备树。暂时Linux的方法还是写得比较不灵活。

多核的启动除了软中断，还有一个使用核的监视模块完成，它有专门的核启动和核停止扩展。这个怎么设计需要再看代码才能做决定。

明天休息一天，需要做一些其它事情。

## 第7天

稍微整理了K210下的外设，然后试试怎么写设备树。这一块暂时是写死的。发现RISC-V的SBI标准，目标大致是Unix平台，
所以暂时可以默认第二个启动参数是约定俗成的规则，是设备树的二进制表示。
我们写的内核暂时基于SBI启动。发现Unix系的系统特别是Linux，能用来测试SBI标准实现是否恰当，
不过这个就停留在阅读源代码级别，编译运行即使是可以的，也需要修改Linux那边的代码。

暂时先弄清楚写死的设备树可以放在内存的哪一块。编译一下然后用`include_bytes!`放进静态全局变量里似乎是可行的。
可能考虑页表，会有内存访问权限的问题。

既然我们的系统要写内存管理单元，这一块就需要做更多考虑了。这个后面再说。

设备树里面比如处理器的主频，可以是在加载过程中规定的，这可能就是设备树里面比较灵活的部分，可以在代码里填写这一块内容。
争取花四到五天时间把设备树部分整理出来，有一些进展，然后做一些封装工作。后面休息一到两天时间，我个人有其它事情要忙。

## 第6天

今天启程回家，休息一天。

### 1. 第二阶段目标

必选项

1. 完善RustSBI，能在qemu上运行（已完成），能在rCore-Tutorial上运行
2. 设备树系统，设备树的封装

可选项

1. 从SD卡启动的内核，或者从SD卡加载应用程序，使用embedded-sdmmc
2. 基于协程的内核，使用此内核增加异步用户程序的运行速度

## 第5天

今天所有小组的同学都上台做报告。

未来我的工作是，先做rustsbi在K210上的引导操作系统，适配最新的SBI规范等等。然后如果有时间，去实现之前和向勇老师、王润基学长
谈到的异步操作系统的想法。

## 第4天

今天经过一些调试，rustsbi能在QEMU里运行了。下一步是在K210上运行。
之前调试最久的一个问题是，SBI的legacy扩展，返回值的位置在a0寄存器，
然而新的扩展全都是a1寄存器，对于唯一一个能有返回值的调试控制台读入扩展，会出现一些问题。
添加一些代码之后就好了。

K210有非常多的坑，希望能把学长的文档都看一遍之后，再想着移植到这个目标上。

下午华为的工程师来做关于欧拉操作系统的演讲。原来的内核，需要先上传再支持，接近一年的时间之后，才会适配新的芯片。
这就是为什么华为要开发欧拉系统，是在华为的芯片发布的当天，就有一个可用的操作系统。
开源社区共享共建的模式，需要具备独立演进能力。

晚上王润基学长讲了zircon内核的很多概念，包括内核对象、带权限等级的
句柄，等等。然后补上了周四和硬件有关的内容，包括UEFI等等。
UEFI这一类和SBI非常像，它应该提供更多的功能，就比如核的监视系统，比如内核已经死机了，就可以通过处理器核监视的系统复位这个核，
或者查看更多的信息。
介绍了rBoot项目，它会从存储设备里读内核的内容加载到内存，
然后配置页表、建立映射，然后设置图形输出，然后把启动信息传给内核。
还有一个trapframe-rs包包装了中断上下文保存的方法，
这些我们做完一次后，后面的开发者就不需要再考虑了，尤其是在历史包袱非常重的x86架构下，就方便调试错误。
virtio-drivers是virtio驱动，杰哥写了个驱动，然后从rCore里面抽出来了。Rust的包生态非常重要，能防止后人重复造轮子。后面的而设计里可以探索怎么和async/await结合起来。

今天可以在K210运行SBI了，但是要引导操作系统，就要把系统和SBI拼接起来。这部分回头看看dd和其它工具的文档。

## 第3天

前一天晚上和老师交流。有一个做操作系统协程的灵感，就是首先有一个线程，在线程上运行所有的协程，协程只会在它的任务完成后
交接，就不用保存上下文。然后一旦发生中断，保存的是这个线程的上下文。然后线程和协程共用一个调度器。
对于async/await生态，是一个相当好的处理方法。用网页服务器举例，最高效的方法是直接编写专用操作系统，
比较普通的方法是做一个用户层的调度器。我们如果能在操作系统里支持协程，就能性能相当高地给出一些异步运行时。

早上和向老师、小组同学交流。我继续开发rustsbi软件，先把rCore教程的系统跑在现在的SBI实现里面。
后续的SBI实现还包含很多模块，包括硬件线程监测和管理模块，还需要很多事情要做。包括跨核刷新页表也是这样的。

下午到清华伯克利深圳学院听了RIOS组织的报告。
RISC-V和arm系区别的是，标准是免费的，然后有授权的设计反而是最多的，封闭的设计可能差不多，RISC-V存在大量的免费设计。这里稍微提了一下法律的问题，如何防范专利上糟糕的问题，然后开源的核在法律上是一个新的领域。

RIOS组织正在达成一个五年目标，提出了一个称作PicoRio的开放架构。希望能减少开发者的开销，有一款文档详细的单板计算机，功率要和树莓派板子对标。
PicoRio的开发分为三个阶段，第一阶段希望实现异构多核的处理器，要实现还在草稿中的RISC-V的动态语言J扩展，跑Linux，跑Chrome OS的内核。希望在今年秋天发布第一个版本。
第二个阶段希望支持图形处理器，希望支持虚拟机监视器，希望有一个WebAssembly的运行时。第三个阶段希望有更多的工业软件和更好的性能。

J扩展正在草稿阶段，可能要增加两个功能。还在讨论过程中。
增加一些CSR，增加一些地址保护的模块。非常早期。

晚上听了Sipeed工程师的讲座。首先工程师介绍了MaixPy，它的使用方法，如何移植，常见容易有问题的地方。
MaixPy是用micropython语法，快速运行AI功能的一个软件包。
K210有硬件的神经网络加速器。使用了micropython，这能帮助新手更快地入门上手。
micropython是解释性语言，有缺点，不过它的优点是学习成本不高，有一定的开发效率。

工程师讲了micropython的一个例子，但其实使用Rust写嵌入式是更方便的。我给出下面的例子：

```rust
#[riscv_rt::entry]
fn main() {
    let dp = pac::Peripheral::take().unwrap();
    // ... 配置时钟，创建总线等等
    let mut lcd = LcdXxxx::new(spi); // 将包含初始化过程
    let sensor = SensorXxx::new(bus);
    loop {
        let img = sensor.snapshot();
        lcd.show(&img);
    }
}
```

其实也不会更复杂，而且工业上Rust语言的所有权能带来更多的便利，也不需要手动去找一些漏洞，缩短开发时间，
还能无缝使用社区丰富的库或者厂商提供的库，未来还能和async/await的生态结合上。
Micropython初学是非常快的，但是做到应用上可能就缺少一些灵活性。不过Micropython依然是非常有价值的语言和框架。
然后还有神经网络加速器的应用，和普通的流程相似，也有输入和模型，会输出结果。
模型和输入都放进处理器，然后由处理器与硬件加速器通讯，最终由处理器得到输出的结果。

大概的代码，首先调用神经网络加速器的加载函数，加载特定的模型，得到模型变量。
然后打开图片，把图片和模型变量都传入加速器，得到输出。最终输出就包含结果，可以做很多科学计算了。
模型在电脑上有几十兆的空间，但是嵌入式芯片上只有几兆的容量，就需要压缩模型。
后续还要做的过程包括简单的用户界面，分享模型的网站，移植Linux等等。
工程师简单介绍了platformIO环境的开发流程。

吴学长分享了K210编写操作系统的很多工作。有一位学长把1991年的Linux0.11版本移植成功。
王润基学长移植了rCore这个有内存管理单元的系统。后来有没有内存管理单元的Linux也可以移植。

当前移植K210的进展，首先暂时使用已有的OpenSBI。它会初始化所有的寄存器，然后输出一些CPU信息，
最终将跳转到内核。第二点是在运行的时候支持SBI调用。第三种是初始化跳转到系统，可以跳转到固定的位置，可以和内核一起编译，
也可以动态决定要跳转到的位置。
QEMU上是跳转到固定的位置的，只能跳转到特定的位置。但是在K210上最新的QEMU自带的OpenSBI不能工作，
只有旧版本的OpenSBI可以工作。这样能过lab0了。然后M到S的中断委托也是需要处理的。

中断处理有两个：PLIC，转发外部中断。CLINT，转发软件和时钟中断。

## 第1天

## 第2天

这两天进入了K210的小组，了解了一些同学。
下午听了国科大同学一生一芯处理器的报告。

简介，开发过程，经验，展望

芯片基本情况。处理器：“果壳”，9级双发射，RV64IMAC，M、S、U，L1、L2，流片内有高速缓存，chisel语言，有外设，Linux可用
110纳米工艺，200mW@350MHz，100脚封装
有板卡，运行Linux，图形界面，运行虚拟机，网络，动态语言脚本。PCIE的SSD上运行，挂载在NVME的设备上
不用SDRAM时，可以到200MHz。CoreMark：1.49/MHz

2019/9月到12月开发结束。前端开发过程包括高速缓存、Sv39到时序优化、适配系统等各个过程。
板卡设计失误，险些报废芯片，后修复。如果未来需要调试软件，很难想到问题会出现在硬件上。
采访：流片困难吗？同学们都设计前端，较少接触硬件。前端上逻辑、时序设计要求都非常高。流片的时间节点压力非常大。
流片的压力非常大，如果有问题，后面的软件就无法修改。冻结时间提前一个月。
后端和前端的优化暂时不多。四个月的时间是比较困难的，需要很多团队的支持，包括开源工具，有支持基础。
从零开始做会是比较难的，后续的推广可能不是每年都能做这么复杂的事情。

项目的复杂性是什么时候认识到的，如何应对？程序出现问题之后，需要关注自己项目的正确性和验证环节的正确性，
如果一开始只会一些简单的应用，就很容易出现一些漏洞。
复杂的项目需要注意版本管理的方法。
这个项目的复杂性在后期各个部分的验证中。前期先把核看成黑盒，但是后期需要做软件测试，需要了解核内是怎么构成的。
外设有很多种，需要了解所有的外设代码。转变心态。
用类似于面向对象的思想去处理核。慢慢进入黑盒的各个部分，再定位问题在哪个部件，最后找到问题。

Chisel强调面向对象，和Verilog的区别，类似于Java和汇编语言。
使用了一套处理器在线差分测试仿真验证的框架。因为qemu只能搭建几个周期的寄存器值等等，比对状态，使用了nemu。
验证CPU模块：核内可以差分测试，核外不行。核外可以使用回环测试，比如验证以太网模块等等，可以采用短接TX、RX的方法。
第二级缓存预取部分，发现漏洞可以解决，难的在于在没有漏洞的前提下如何提升性能。
需要使用回归测试。写脚本，遍历所有的设置。

印象最深刻的漏洞。跑了33亿条指令才出现问题。是跨页指令的问题，需要有仿真机制才能找到，定位这个问题相当困难。
GPIO的四个引脚。已经到ASIC过程，发现IO板的模型，“OEN”引脚的“N”其实表示低位有效，没有接对。变量名没起好，方言不同。
快表地址对的，取出来的地址错了。发现是软件没有刷新快表。
分支预测器对高速缓存有一些隐含条件，修复后性能就回去了。

适配操作系统的过程中有哪些困难？分为流片之前的开发和流片后的测试。
Sv39分页，内核和用户暂时没有超过1GB，共享了一个巨页，用户空间的内存能被内核发现。这其实是软件问题。
FreeRTOS没有坑，它非常小。还有一个RT-Thread，添加了RV64的支持，比如把指令lw改成ld等等。
还有一个ALIGN(4)改成ALIGN(8)的问题，属于RV32移植到RV64里面。
xv6没刷新快表，出现一些机制支持不正确。用官方的xv6跑芯片，一定要刷新快表。涉及到Linux系统的进程分支操作。
软件中断没有实现，导致远程刷新页表没有实现，远程的快表没有刷新。硬件上暂时是单核，不实现机器软中断。
如果一生一芯要支持多核，就必须实现软中断。
闪存只能读4字节，没法读8字节，而且没法写，只能擦除。放在2字节对齐的4字节指令会出问题，闪存里面只能跑4字节指令。
misa的问题。导致收到的是M模式的缺页，而不是S模式。
虚拟动态共享对象vDSO，允许在不陷入内核的前提下做一些简单的调用。
