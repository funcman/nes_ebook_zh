# 实现其余的 CPU 指令


 <div style="text-align:center"><img src="./images/ch3.3/image_1_how_to_draw_owl.png" width="60%"/></div>

实现其余 6502 CPU 指令应该相对简单。我不会详细介绍所有这些。

只是一些注释：
* 从逻辑流程的角度来看， **ADC** 可能是最复杂的指令。请注意，该规范包含有关可以完全跳过的十进制模式的详细信息，因为该芯片的 Ricoh 修改不支持十进制模式。
> 本文详细介绍了如何在 6502 中实现二进制算术：[6502 溢出标志以数学方式解释](http://www.righto.com/2012/12/the-6502-overflow-flag-explained.html)
>
>对于好奇和勇敢的人，可以看一下：[6502 CPU的溢出标志在硅层面的解释](http://www.righto.com/2013/01/a-small-part-of-6502-chip-explained.html)

* 实现 ADC 后，实现 **SBC** 变得微不足道，因为
`A - B = A + (-B)`.
以及 `-B = !B + 1`

* **PHP** 、 **PLP** 和 **RTI** 必须处理 [2位B-flag](http://wiki.nesdev.com/w/index.php/Status_flags#The_B_flag)。除了中断执行之外，这些是唯一直接影响（或直接受其影响）**状态寄存器P**的第 5 位的命令

* 大部分的跳转和跳转操作都可以通过简单的修改 **program_counter** 寄存器来实现。但是，请注意不要在同一指令解释周期内递增寄存器。

如果你卡住了，你可以随时在这里查看 6502 指令集的实现：<link to code>


<br/>

------

> 本章完整源代码： <a href="https://github.com/bugzmanov/nes_ebook/tree/master/code/ch3.3" target="_blank">GitHub</a>
