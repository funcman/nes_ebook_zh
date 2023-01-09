# PPU 卷动

在我们开始讨论卷动之前，我们需要澄清一个细节。我们已经讨论过 PPU 通过触发 NMI 中断来通知帧的状态，它告诉 CPU 当前帧的渲染已经完成。
这还不是全部。PPU 有 2 个额外的机制来告诉它的进度：

* [精灵零命中标志](https://wiki.nesdev.com/w/index.php?title=PPU_OAM&redirect=no#Sprite_zero_hits)
* [精灵溢出标志g](https://wiki.nesdev.com/w/index.php/PPU_sprite_evaluation#Sprite_overflow_bug)

两者都使用 [PPU 状态寄存器 **0x2002**](https://wiki.nesdev.com/w/index.php/PPU_registers#Status_.28.242002.29_.3C_read) 进行报告

<div style="text-align:center;"><img src="./images/ch8/image_7_sprite_0_hit.png" width="60%"/></div>

Sprite 溢出很少使用，因为它有一个导致误报和漏报的错误。
大多数具有卷动功能的游戏都使用 Sprite 0 hit。这是获取 PPU 的中帧进度状态的方法：

* 将精灵零放在特定的屏幕位置（X，Y）
* 轮询状态寄存器
* 当 sprite_zero_hit 从 0 变为 1 - CPU 知道 PPU 已完成渲染 **[0 .. Y]** 扫描线，并且在 Y 扫描线上，它已完成渲染 X 像素。

> 这是对行为的非常粗略的模拟。准确的需要检查精灵的不透明像素与背景的不透明像素碰撞。

我们需要在 PPU `tick` 函数中编码这种行为：

```rust
    pub fn tick(&mut self, cycles: u8) -> bool {
        self.cycles += cycles as usize;
        if self.cycles >= 341 {
            if self.is_sprite_0_hit(self.cycles) {
                self.status.set_sprite_zero_hit(true);
            }

            self.cycles = self.cycles - 341;
            self.scanline += 1;

            if self.scanline == 241 {
                self.status.set_vblank_status(true);
                self.status.set_sprite_zero_hit(false);
                if self.ctrl.generate_vblank_nmi() {
                    self.nmi_interrupt = Some(1);
                }
            }

            if self.scanline >= 262 {
                self.scanline = 0;
                self.nmi_interrupt = None;
                self.status.set_sprite_zero_hit(false);
                self.status.reset_vblank_status();
                return true;
            }
        }
        return false;
    }

    fn is_sprite_0_hit(&self, cycle: usize) -> bool {
        let y = self.oam_data[0] as usize;
        let x = self.oam_data[3] as usize;
        (y == self.scanline as usize) && x <= cycle && self.mask.show_sprites()
    }

```

注意：精灵零命中标志应在进入 VBLANK 状态时被清除。


## 卷动

卷动是在 NES 游戏中模拟空间运动的主要机制之一。将视野移动到静态背景上以创建在空间中移动的错觉是一个古老的想法。

<div style="text-align:center;"><img src="./images/ch8/image_1_scroll_basics.png" width="80%"/></div>

卷动在 PPU 级别上实现，仅影响背景图块（存储在名称表中的图块）的渲染。精灵（OAM 数据）不受此影响。

PPU 可以同时在内存中保存两个屏幕（记住一个名称表 - 1024 字节，PPU 有 2 KiB 的 VRAM）。这看起来并不多，但这足以解决问题。在卷动期间，视野在这两个名称表中循环，而 CPU 正忙于更新尚未可见但很快就会出现的屏幕部分。
这也意味着大多数时候，PPU 都在渲染两个名称表的一部分。

因为这会耗尽所有可用的游戏机资源，所以早期的游戏只有两种卷动选项：水平或垂直。旧游戏确定了整个游戏的卷动类型。
后来出现的游戏具有在阶段之间交替卷动的机制。最先进的游戏（如塞尔达）提供了用户可以在所有 4 个方向上“移动”的体验。

<div style="text-align:center;"><img src="./images/ch8/image_2_scroll_mirroring.png" width="60%"/></div>


最初，卷动与镜像紧密结合 - 主要是因为 NES 在硬件级别上处理从一个命名表到另一个命名表的视野溢出的方式。

对于像 Super Mario Bros (Horizontal Scroll) 或 Ice Climber (Vertical Scroll) 这样的游戏，机制完全由以下定义：
- 镜像类型（在卡带 ROM 标头中设置）
- 基本名称表地址（PPU 控制寄存器中的值）
- PPU 卷动寄存器的状态（视口的 X 和 Y 位移值，以像素为单位）
- 名称表的内容

请记住，背景屏幕由 960 个图块定义，每个图块为 8x8 像素，因为 PPU 卷动寄存器定义了像素偏移，这意味着在视口的边缘，我们可以看到图块的一部分。


<div style="text-align:center;"><img src="./images/ch8/image_3_scroll_controll.png" width="70%"/></div>


更新 PPU 内存相对昂贵，CPU 只能在 241 - 262 扫描线期间执行此操作。由于这些限制，CPU 可以每帧更新屏幕相对较薄的部分（2x30 块宽区域）。
如果我们渲染还不可见的名称表的一部分，我们可以看到世界的状态是如何在进入视口之前的几帧中出现的。

<div style="text-align:center;"><img src="./images/ch8/image_4_scroll_demo.gif" width="50%"/></div>

开始实施之前的 2 个最后注意事项：
* 切片的调色板由切片所属的名称表定义，而 **不是** 由控制寄存器中指定的基本名称表定义
* 对于水平卷动，基本名称表的内容始终位于视野的左侧（或垂直卷动时的顶部）


<div style="text-align:center;"><img src="./images/ch8/image_5_scroll_caveats.png" width="80%"/></div>

实现卷动渲染并不难，但需要注意细节。我能想到的最方便的心智模型如下：
* 对于每一帧，我们将扫描两个名称表。
* 对于每个名称表，我们将指定名称表的可见部分：

```rust
struct Rect {
   x1: usize,
   y1: usize,
   x2: usize,
   y2: usize,
}

impl Rect {
   fn new(x1: usize, y1: usize, x2: usize, y2: usize) -> Self {
       Rect {
           x1: x1,
           y1: y1,
           x2: x2,
           y2: y2,
       }
   }
}
```

* 并对每个可见像素应用移位变换 - shift_x, shift_y

> 例如，
> <div style="text-align:center;"><img src="./images/ch8/image_6_transform_example.png" width="30%"/></div>
>
> 对于名称表 **0x2400**：可见区域将被定义为 **(200, 0, 256, 240)** ，移位将是 **(-200, 0)**<br/>
> 对于名称表 **0x2000**: 可见区域是 **(0,0, 200, 240)** ，移位是 **(56, 0)**

因此，要绘制一个名称表，我们需要创建一个辅助函数：

```rust
fn render_name_table(ppu: &NesPPU, frame: &mut Frame, name_table: &[u8],
   view_port: Rect, shift_x: isize, shift_y: isize) {
   let bank = ppu.ctrl.bknd_pattern_addr();

   let attribute_table = &name_table[0x3c0.. 0x400];

   for i in 0..0x3c0 {
       let tile_column = i % 32;
       let tile_row = i / 32;
       let tile_idx = name_table[i] as u16;
       let tile = &ppu.chr_rom[(bank + tile_idx * 16) as usize..=(bank + tile_idx * 16 + 15) as usize];
       let palette = bg_pallette(ppu, attribute_table, tile_column, tile_row);

       for y in 0..=7 {
           let mut upper = tile[y];
           let mut lower = tile[y + 8];

           for x in (0..=7).rev() {
               let value = (1 & lower) << 1 | (1 & upper);
               upper = upper >> 1;
               lower = lower >> 1;
               let rgb = match value {
                   0 => palette::SYSTEM_PALLETE[ppu.palette_table[0] as usize],
                   1 => palette::SYSTEM_PALLETE[palette[1] as usize],
                   2 => palette::SYSTEM_PALLETE[palette[2] as usize],
                   3 => palette::SYSTEM_PALLETE[palette[3] as usize],
                   _ => panic!("can't be"),
               };
               let pixel_x = tile_column * 8 + x;
               let pixel_y = tile_row * 8 + y;

               if pixel_x >= view_port.x1 && pixel_x < view_port.x2 && pixel_y >= view_port.y1 && pixel_y < view_port.y2 {
                   frame.set_pixel((shift_x + pixel_x as isize) as usize, (shift_y + pixel_y as isize) as usize, rgb);
               }
           }
       }
   }
}
```


那么渲染背景就变得比较简单了：

```rust


pub fn render(ppu: &NesPPU, frame: &mut Frame) {
   let scroll_x = (ppu.scroll.scroll_x) as usize;
   let scroll_y = (ppu.scroll.scroll_y) as usize;

   let (main_nametable, second_nametable) = match (&ppu.mirroring, ppu.ctrl.nametable_addr()) {
       (Mirroring::VERTICAL, 0x2000) | (Mirroring::VERTICAL, 0x2800) => {
           (&ppu.vram[0..0x400], &ppu.vram[0x400..0x800])
       }
       (Mirroring::VERTICAL, 0x2400) | (Mirroring::VERTICAL, 0x2C00) => {
           ( &ppu.vram[0x400..0x800], &ppu.vram[0..0x400])
       }
       (_,_) => {
           panic!("Not supported mirroring type {:?}", ppu.mirroring);
       }
   };

   render_name_table(ppu, frame,
       main_nametable,
       Rect::new(scroll_x, scroll_y, 256, 240 ),
       -(scroll_x as isize), -(scroll_y as isize)
   );

    render_name_table(ppu, frame,
        second_nametable,
        Rect::new(0, 0, scroll_x, 240),
        (256 - scroll_x) as isize, 0
    );

// … render sprites
}

```

实现垂直卷动类似；我们可以重用相同的 `render_name_table` 辅助函数而无需更改。只需要弄清楚正确的 *寻址* 、 *移位*, 和 *视口* 参数。

可以在[此处](https://github.com/bugzmanov/nes_ebook/tree/master/code/ch8)找到完整定义的卷动代码

对卷动的支持意味着现在我们可以玩像超级马里奥兄弟和攀冰者这样的老平台游戏。

最后缺少的部分是 APU。

<br/>

------

> 本章完整源代码： <a href="https://github.com/bugzmanov/nes_ebook/tree/master/code/ch8" target="_blank">GitHub</a>
