# OS实验报告

[仓库链接](https://github.com/luojia65/spicy-os)

这部分我一边跟进教程，一边学习操作系统的理论知识，还尝试完善相关的开发生态。

实验零主要是学习一些安装Rust和环境的方法。由于我们暂时使用nightly版本的Rust，
需要注意部分问题。Rust的安装是非常容易的，而且比C语言好的是有统一的包管理器
称作Cargo。这比往常使用C语言开发操作系统要方便不少，也要简单很多，出错了也容易
找到来源。这部分的实验非常简单，相信许多已经入门Rust的开发者都可以迅速完成。

## 1. 第一到三章

实验一、二、三主要是RISC-V架构的知识，包括操作系统通用的中断、页表等等。
这些编写每个操作系统是都需要接触的知识。

### 第一章

实验一里介绍了RISC-V的中断系统。这也包括了相关的CSR寄存器，还有一些中断机制。
RISC-V的陷入包括异常和中断，其中中断分为三个特权级、三个种类，共占用9个编号，
中断是可以通过`mstatus`里的总开关打开或关闭的。
我们在后续的实验中，应该特别注意哪些属于异常，哪些属于中断。
异常分为十几个类型，包括非法指令、页错误、读写地址错误等等，环境调用也属于异常。
RISC-V提供了特殊的`ecall`和`ebreak`指令，其中`ecall`能产生环境调用异常，
这在后续的系统调用里是非常有用的。

实验中特别介绍了时钟中断的处理方法。今天主流的指令集架构里，
几乎都专门给系统时钟提供了支持，这类设计一般都是为操作系统的时间轮片调度提供便利。
时钟中断是涉及上下文调度的，我们在涉及上下文的处理函数里，
必须保存所有31个寄存器（不考虑x0），
而且还要保存`sstatus`和`sepc`，因为切换回来之后，可能就在不同的上下文内。
如果只涉及异常处理，比如在页错误里面重新处理页表，或者系统调用，
其实只保存16个寄存器就可以了（不包含s0-s11、tp、gp和sp），
因为它们能代表我们需要恢复的上下文，其它寄存器不涉及上下文切换，意义不会变化。

实验中使用断点异常作为测试，这应该是最容易得到中断的方法。
这类异常直接就可以运行，不需要打开开关。
但是时钟的运行是需要开关打开的，需要设置`sie`和`sstatus`相关的位，
才能打开时钟的运行，从而触发时间中断。

中断系统里，所有的异常都是可以处理的。要达到这一点，我们需要编写处理函数。
这个处理函数和中断处理的模式需要放入`mtvec`里面。
中断处理有两个模式，一个是直接模式，一个是向量模式，本次实验中使用直接模式。
处理函数需要对齐到4字节。

所有的中断发生时，`sepc`的值都是下一个指令的地址。
但是异常发生时，`sepc`都是当前指令的地址。因为一个指令发生了异常，
它很可能需要在处理后重新执行一次，比如缺页异常，可以把这个页补上然后再试一遍。
`ecall`被归类到异常里面，所以需要把`sepc`加上4，来跳过原来的指令。

需要注意的是RISC-V标准提供的中断系统是不支持嵌套的。要使用嵌套中断，
可以使用赛昉提出的PLIC，也可以使用芯来提出的ECLIC，这些就看芯片是如何支持的了。
这些是M层中断，一般都在SBI里处理完成，SBI只会给我们委托一些异常，
S层的操作系统作者可能不需要考虑太深。

### 第二章

这一章讲了物理内存的探测和分配。其实也涉及了大量Rust alloc库的内容和设计方式。
其实SBI运行时会给一个设备树，从这里面是能扫描到所有的内存块，
本次实验出于简单考虑，直接把地址固化在代码里面。
还大致使用了堆分配的算法，在初始化代码里加载一个堆，这样就能用Box和Arc等等了。

需要注意的事项不多。每个段最好对齐到4K，方便后面分配初始的段内存。

### 第三章

这一章是比较重要的部分，虚拟地址到物理地址，涉及的难点是页表。

因为每个操作系统的地址空间设计都会不一样，初始的页表也会不一样。本次实验里初始页表
暂时是固化在代码里面的，也可能会有其它的设计。

RISC-V的虚拟地址涉及几个概念：虚拟地址、物理地址、虚拟页号、物理页号。
其中虚拟地址与整数寄存器宽度相等，物理地址可能大于整数寄存器的位数。
虚拟页号和物理页号都是地址的一部分，高位如果不够，都用符号扩展，第0到11位都是索引位，
剩余的都是页号的位置。无论如何，物理页号和虚拟页号都能装进整数寄存器宽度的页表项中。

页号是分成几部分的，虚拟页号的每一部分都有9位（32位下有10位）。
这个设计可能是由统一的4KB页大小决定的。
需要注意的是，如果自己设计页表，页表的级数和物理页号的对齐方法是有关的，
需要仔细查看文档后继续编写。

页表项的低10位是大量的标记，表示每个页表项的权限等等。包括最低的V位代表是否有效。
其实如果这一位是0，页表项仍然可以存放一些有用的信息，这个就另外说了。

页表都是有好多级的，包括Sv32是两级，64位下Sv39是三级，Sv48是四级。
页表的级数从高到低，最先查询的页表级数最大，最后查询的级数最小为零。
最终需要把最高级页表的物理地址存到`satp`里面。`satp`也包含页表的模式，
还有一个地址空间标识符，本次实验还没有用到，其它地方会用得到。

还有一个就是研发处理器核的人需要关心的，就是快表。页表如果还只从内存里直接查那就太慢了，
有个快表就快一点，相当于是某种缓存。
出于这种考虑，`satp`写完之后，一定要记得使用`sfence.vma`刷新快表，否则访问会出问题。

既然我们有页表了，虚拟地址也有了，这样操作系统就能编译到虚拟地址启动了。
即使编译到虚拟地址，M层的SBI运行时调用操作系统时，还是物理地址，
这时候地址的转换暂时是有效的，访问临时变量也是有效的，因为一般是按相对地址去访问。
需要建立一个初始映射，这个就和操作系统的设计有关系，把物理地址映射到虚拟地址上。
跳转到虚拟地址，需要把起始虚拟地址的值写到代码里面，用汇编指令去读取，然后跳转。
这时候还要准备一个比较小的启动栈。

初始映射有了之后就可以准备内核重映射了。这里就涉及到大量代码工作，
把内核的每个段映射到每个地址，这样就可以解引用然后运行了。

还有一个页面置换的问题，涉及到缺页异常怎么处理，然后要不要把哪个页置换到磁盘里，
把新的页拿出来到内存里面。这里其实DMA就可以派上用场了，还有很多特性。
这个到后面遇到了再解决，这章实验就做简单的了解。

页表的调试是非常难的，尤其是还没有进入虚拟地址的时候页表的调试，
这时候通常需要对照反汇编代码，反复测试后再继续。

## 2. 第四章

//todo

## 3. 第五到六章

## 题外话

其实我很想把RISC-V翻译成“第五代精简指令集技术”，因为简短性，保留外文原文。
我在编写各类文章、博客的时候，除了专有名词，尽量全部使用汉语翻译，是为了让读者理解更清晰。