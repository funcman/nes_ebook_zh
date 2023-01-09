# 模拟 NMI 中断

中断是 CPU 中断顺序执行流程并对需要立即关注的事件做出反应（“参加中断”）的机制。

我们已经实现了一种受支持的中断——RESET 信号。该中断通知 CPU 插入了新的卡带并且 CPU 需要执行复位子程序。

 <div style="text-align:center"><img src="./images/ch6.2/image_4_broadcast_interrupted.png" width="30%"/></div>


PPU 通过另一个中断信号 NMI（不可屏蔽中断）传达它正在进入帧的 VBLANK 阶段。
从高层次的角度来看，这意味着两件事：
- PPU 完成渲染当前帧
- CPU 可以安全地访问 PPU 内存以更新下一帧的状态。

> VBLANK 阶段之所以独特，是因为 PPU 在渲染可见扫描线的同时，它一直在使用内部缓冲区和内存。对 IO 寄存器的外部访问可能会损坏这些缓冲区中的数据并导致明显的图形故障。

与其他中断不同，CPU 不能忽略 NMI。 **状态寄存器P** 中的 **禁用中断** 标志对 CPU 处理它的方式没有影响。
但是，CPU 可能会通过重置 PPU 控制寄存器中的第 7 位来指示 PPU 不触发 NMI。

## 时钟周期

NMI 中断与 PPU 时钟周期紧密相关：
* PPU 每帧渲染 262 条扫描线
* 每条扫描线持续 341 个 PPU 时钟周期
* 进入扫描线 241 后，PPU 触发 NMI 中断
* PPU 时钟周期比 CPU 时钟周期快 3 倍

没有什么比 NESDev wiki 更能提供[逐行计时的详细信息g](http://wiki.nesdev.com/w/index.php/PPU_rendering#Line-by-line_timing)了

但为了简化，
 * 每个 PPU 帧占用 ```341*262=89342 PPU 时钟周期```
 * CPU 保证每次中断都能接收到 NMI ```~29780 CPU 周期```

> **注意：** PPU 周期和 CPU 周期不是一回事

在 NES 平台上，所有组件都独立并行运行。这使得NES成为一个分布式系统。协调必须由游戏开发者根据指令的时间规格仔细设计。我只能想象这个手动过程是多么乏味。

模拟器可以采用多种方法来模拟这种行为：
1) 为每个组件分配一个线程并为每条指令模拟适当的时序。我不知道有任何模拟器可以做到这一点。模拟正确的时间是一项艰巨的任务。其次，这种方法需要分配比作业所需更多的硬件资源（PPU、CPU 和 APU 将需要 3 个线程，并且可能会占用主机上的 3 个内核）

2) 通过在每个组件中一次推进一个时钟周期，在一个线程中按顺序执行所有组件。这类似于创建一个绿色线程运行时并使用一个专用的操作系统线程来运行这个运行时。这将需要大量投资来创建绿色线程运行时。

3) 在一个线程中按顺序执行所有组件，但通过让 CPU 执行一条完整指令，计算其他组件的时钟周期预算并让它们在预算内运行。这种技术称为["追赶"](http://wiki.nesdev.com/w/index.php/Catch-up) 。<br/> <br/>例如，CPU 需要 2 个周期来执行 “LDA #$01”（操作码 0xA9），这意味着 PPU 现在可以运行 6 个 PPU 周期（PPU 时钟比 CPU 时钟快三倍） ) 和 APU 可以运行 1 个周期（APU 时钟慢两倍）

因为我们已经大部分指定了 CPU 循环，所以第三种方法是最容易实现的。当然，这将是最不准确的一个。但尽快有一些可玩的东西就足够了。

所以流程看起来像这样：

 <div style="text-align:center"><img src="./images/ch6.2/image_1_tick_flow.png" width="60%"/></div>

从 CPU 开始：

```rust
impl CPU {
   pub fn run_with_callback<F>(&mut self, mut callback: F)
   where
       F: FnMut(&mut CPU),
   {
      //...
       loop {
        // …
           self.bus.tick(opcode.cycles);

           if program_counter_state == self.program_counter {
               self.program_counter += (opcode.len - 1) as u16;
           }
   }

   }
}

```

总线应该跟踪执行周期并将滴答调用传播到 PPU，但由于 PPU 时钟比 CPU 时钟快 3 倍，它会乘以该值：


```rust
pub struct Bus {
   cpu_vram: [u8; 2048],
   prg_rom: Vec<u8>,
   ppu: NesPPU,

   cycles: usize,
}

impl Bus {
   pub fn new(rom: Rom) -> Self {
       let ppu = NesPPU::new(rom.chr_rom, rom.screen_mirroring);

       Bus {
           cpu_vram: [0; 2048],
           prg_rom: rom.prg_rom,
           ppu: ppu,
           cycles: 0,
       }
   }
   pub fn tick(&mut self, cycles: u8) {
       self.cycles += cycles as usize;
       self.ppu.tick(cycles * 3);
   }
}
```

PPU 将跟踪周期并计算应该绘制哪条扫描线：

```rust
pub struct NesPPU {
   // ...
   scanline: u16,
   cycles: usize,
}



impl NesPPU {
// …
   pub fn tick(&mut self, cycles: u8) -> bool {
       self.cycles += cycles as usize;
       if self.cycles >= 341 {
           self.cycles = self.cycles - 341;
           self.scanline += 1;

           if self.scanline == 241 {
               if self.ctrl.generate_vblank_nmi() {
                   self.status.set_vblank_status(true);
                   todo!("Should trigger NMI interrupt")
               }
           }

           if self.scanline >= 262 {
               self.scanline = 0;
               self.status.reset_vblank_status();
               return true;
           }
       }
       return false;
   }
}

```

一些关键细节仍然缺失：一些 CPU 操作根据执行流程需要可变的时钟时间。
例如，如果比较成功，条件分支操作（如 BNE）会占用额外的 CPU 周期。如果 JUMP 会导致程序计数器位于另一个内存页上，那么还有另一个 CPU 周期

> 内存页大小为 256 字节。例如，范围[0x0000 .. 0x00FF]-属于第0页，[0x0100 .. 0x01FF]属于第1页等。
> 比较地址的高字节，看看它们是否在同一页上就足够了。
我把它留给读者来弄清楚如何编码那些可能会或可能不会发生的额外滴答声。

# 中断

到目前为止，我们的依赖图看起来是单向的：

 <div style="text-align:center"><img src="./images/ch6.2/image_2_components_dag.png" width="60%"/></div>

问题是我们想要将信号从 PPU 传递到 CPU，而 Rust 并没有真正允许轻松地产生依赖循环。

克服这个问题的一种方法是将推模型替换为拉模型。CPU 可以在解释周期开始时询问是否有中断准备就绪。

```rust
impl CPU {
//...
   pub fn run_with_callback<F>(&mut self, mut callback: F)
   where
       F: FnMut(&mut CPU),
   {
       // ...
       loop {
           if let Some(_nmi) = self.bus.poll_nmi_status() {
               self.interrupt_nmi();
           }
           // …
       }
    }
}
```

最后一部分是实现中断行为。
CPU收到中断信号后：
1) 完成当前指令的执行
2) 完成当前指令的执行
3) 通过在 **状态寄存器P** 中设置 **禁用中断** 标志来禁用中断
4) 从 0xFFFA 加载中断处理程序的地址（用于 NMI）
5) 设置指向该地址的 **程序计数器**

 <div style="text-align:center"><img src="./images/ch6.2/image_3_interrupt_mem.png" width="40%"/></div>


除了扫描线位置之外，如果满足以下两个条件，PPU 将立即触发 NMI：


```rust
   fn interrupt_nmi(&mut self) {
       self.stack_push_u16(self.program_counter);
       let mut flag = self.status.clone();
       flag.set(CpuFlags::BREAK, 0);
       flag.set(CpuFlags::BREAK2, 1);

       self.stack_push(flag.bits);
       self.status.insert(CpuFlags::INTERRUPT_DISABLE);

       self.bus.tick(2);
       self.program_counter = self.mem_read_u16(0xfffA);
   }
```

除了扫描线位置之外，如果满足以下两个条件，PPU 将立即触发 NMI：
* PPU 为 VBLANK 状态
* 控制寄存器中的“生成 NMI”位从 0 更新为 1。

```rust
impl PPU for NesPPU {
// ...    
    fn write_to_ctrl(&mut self, value: u8) {
        let before_nmi_status = self.ctrl.generate_vblank_nmi();
        self.ctrl.update(value);
        if !before_nmi_status && self.ctrl.generate_vblank_nmi() && self.status.is_in_vblank() {
            self.nmi_interrupt = Some(1);
        }
    }
//..
}
```

# 其他 CPU 中断

在我们的 CPU 实现中，我们实现了操作码 **0x00** 作为 CPU 获取-解码-执行周期的返回，但实际上它应该触发 BRK 中断。这就是所谓的“软件中断”，游戏代码可以通过编程方式触发以响应事件。

NESDEV Wiki 提供了有关[CPU中断](https://wiki.nesdev.com/w/index.php/CPU_interrupts)的所有必要细节。

<br/>

------

> 本章完整源代码： <a href="https://github.com/bugzmanov/nes_ebook/tree/master/code/ch6.2" target="_blank">GitHub</a>
