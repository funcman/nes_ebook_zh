# 模拟 PPU

图片处理单元是最难模仿的，因为它处理游戏中最复杂的方面：渲染屏幕状态。NES PPU 有很多怪癖。虽然不一定需要模拟其中一些，但其他一些对于拥有可玩环境至关重要。
64KiB 的空间并不大，NES 平台设计人员试图尽可能多地从中挤出。使用 CHR ROM 数据意味着使用压缩数据格式。它需要大量的位算术、解压缩和解析。

 <div style="text-align:center"><img src="./images/ch6/image_1_ppu_failures.png" width="60%"/></div>


我们将使用四个主要步骤创建 PPU 模拟器：
* 模拟寄存器和 NMI 中断
* 从 CHR ROM 解析和绘制图块
* 渲染 PPU 状态：
    * 渲染背景图块
    * 渲染精灵
* 实现卷动

第一步与模拟 CPU 非常相似。
在第三个之后，就可以玩静态屏幕游戏了：
- [Pac-Man](https://en.wikipedia.org/wiki/Pac-Man)
- [Donkey Kong](https://en.wikipedia.org/wiki/Donkey_Kong)
- [Balloon Fight](https://en.wikipedia.org/wiki/Balloon_Fight)

当我们完成卷轴后，我们可以玩诸如[Super Mario Bros](https://en.wikipedia.org/wiki/Super_Mario_Bros)之类的平台游戏。

所以让我们开始吧。
