 # 模拟 CPU


本章的目标是让我们的第一个 NES 游戏启动并运行。我们要玩贪吃蛇游戏。可以在[这个gist](https://gist.github.com/wkjagt/9043907)中找到带有注释的源代码。

 <div style="text-align:center"><img src="./images/ch3/snk_logo.png" width="40%"/></div>
 <div style="text-align:center"><img src="./images/ch3/snk_game.gif" width="40%"/></div>

CPU 是任何计算机系统的心脏。运行程序指令并协调所有可用的硬件模块以提供完整的体验是 CPU 的工作。尽管 PPU 和 APU 运行各自独立的电路，但它们仍然必须在 CPU 的节拍下行进，并执行 CPU 发出的命令。

在开始实施之前，我们需要简要讨论一下 CPU 可以使用哪些资源来完成其工作。

CPU 可以访问的仅有的两个资源是内存映射和 CPU 寄存器。

从编程的角度来看，内存映射只是一个 1 字节单元的连续数组。NES CPU 使用 16 位进行内存寻址，这意味着它可以寻址 65536 个不同的内存单元。

正如我们之前所见，NES 平台只有 2 KiB 的 RAM 连接到 CPU。

 <div style="text-align:center"><img src="./images/ch3/cpu_registers_memory.png" width="80%"/></div>


该 RAM 可通过 **[0x0000 … 0x2000]** 地址空间访问。

对 **[0x2000 … 0x4020]** 的访问被重定向到其他可用的 NES 硬件模块：PPU、APU、GamePad 等（稍后会详细介绍）

访问 **[0x4020 .. 0x6000]** 是一个特殊的空间，不同代的卡带使用方式不同。它可能映射到 RAM、ROM 或什么都没有。该空间由所谓的映射器控制 - 卡带上的特殊电路。我们将忽略这个空间。

如果卡带有 RAM 空间，则对 **[0x6000 .. 0x8000]** 的访问被保留给卡带上的 RAM 空间。它在 Zelda 等游戏中用于存储和检索游戏状态。我们也将忽略这个空间。

对 **[0x8000 … 0x10000]** 的访问被映射到卡带上的 Program ROM (PRG ROM) 空间。

内存访问相对较慢，NES CPU 有一些称为寄存器的内部存储插槽，访问延迟显着降低。


> | CPU Operation type  | Execution time (in CPU Cycles)  |
> |---|---|
> | Accessing only registers                         | 2        |
> | Accessing the first 255 bytes of RAM             | 3        |
> | Accessing memory space after the first 255         | 4-7  |


NES CPU 有 7 个寄存器：
* 程序计数器 (*PC*) - 保存下一条要执行的机器语言指令的地址。

* 堆栈指针 - 内存空间 [0x0100 .. 0x1FF] 用于堆栈。堆栈指针保存该空间顶部的地址。NES 堆栈（与所有堆栈一样）从上到下增长：当一个字节被压入堆栈时，SP 寄存器递减。当从堆栈中检索一个字节时，SP 寄存器递增。

* 累加器 (*A*) - 存储算术、逻辑和内存访问操作的结果。它用作某些操作的输入参数。

* 索引寄存器X (*X*) - 用作特定内存寻址模式中的偏移量（稍后会详细介绍）。可用于辅助存储需求（保存温度值、用作计数器等）

* 索引寄存器Y (*Y*) - 与寄存器 X 类似的用处。

* 处理器状态 (*P*) - 8 位寄存器表示 7 个状态标志，可以根据最后执行指令的结果设置或取消设置（例如，如果操作的结果为 0，则 Z 标志设置为 (1)，否则取消设置/擦除为（0））


每个 CPU 都带有一个预定义的硬连线指令集，该指令集定义了 CPU 可以执行的所有操作。

CPU 以机器码的形式从应用层接收指令。您可以将机器语言视为连接软件和硬件的薄层。


官方 6502 指令的完整列表：
* [https://www.nesdev.org/obelisk-6502-guide/reference.html](https://www.nesdev.org/obelisk-6502-guide/reference.html)
* [http://www.6502.org/tutorials/6502opcodes.html](http://www.6502.org/tutorials/6502opcodes.html)

我倾向于使用这两个链接。这些页面提供了可用 CPU 功能及其机器代码的完整规格。

我强烈建议在继续之前阅读这个 [关于 6502 指令的交互式教程](https://skilldrick.github.io/easy6502/) 。

 <div style="text-align:center"><img src="./images/ch3/image_4_opcodes.png" width="80%" /></div>

6502芯片是比较简单的CPU；它仅支持六种类型的命令和大约 64 个独特的命令。因为有些指令对于不同的内存寻址模式有多个版本，这导致我们要实现大约 150 个机器代码操作。

> **注意：** NES游戏机有一个基于 6502 的定制芯片 2A03，但有明显差异：
>
> - 除了官方的机器操作之外，它还有大约110个非官方的附加操作码（幸运的是，其中大约三分之一是NOP）。
> - 它自带有音频处理单元
> - 它不支持十进制的算术模式
>
> 为了保持简单，我们需要实现对256种不同机器指令的支持。
>
> 好消息是，指令之间有很多相似之处。一旦我们有了基础，我们就会不断地重复使用它们来实现整套系统。
