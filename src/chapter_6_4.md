# 渲染静态屏幕

此时，CPU 和 PPU 功能齐全，相互协调工作。
如果我们将游戏加载到我们的模拟器中，游戏将执行并且很可能会进入演示模式。

问题是我们看不到里面发生了什么。还记得我们是如何截取贪吃蛇游戏的执行来读取游戏画面状态的吗？然后让它通过 SDL2 画布渲染？我们将不得不在这里做类似的事情。只是NES使用的数据格式稍微复杂一些。

PPU 必须处理两类对象：

<div style="text-align:center"><img src="./images/ch6.4/image_8_bg_sprites_game.png" width="80%"/></div>

两者都是使用 CHR 瓦片构建的，我们在上一章中已经讨论过。
事实上，相同的图块既可以用于背景，也可以用于精灵。

NES 使用不同的内存空间来保存这些类别。可能的转换集也不同。


## 渲染背景

<!-- <div style="text-align:center"><img src="./images/ch6.4/image_1_pacman_bg.png" width="30%"/></div> -->

三个主内存部分负责后台的状态：
- Pattern Table - 来自 CHR ROM 的 2 组图块之一
- Nametable - 存储在 VRAM 中的屏幕状态
- Palette table - 有关像素真实颜色的信息，存储在内部 PPU 内存中

NES 屏幕背景屏幕由 960 个图块（一个图块为 8x8 像素：`256 / 8 * 240 / 8 = 960`）
每个图块在称为 Nametable 的空间中由 VRAM 中的一个字节表示。

<div style="text-align:center"><img src="./images/ch6.4/image_2_nametable.png" width="100%"/></div>

> 在可命名的 PPU 中使用一个字节只能寻址模式表中单个 bank 中的 256 个元素。
> 控制寄存器决定两个组中的哪一个应该用于背景（以及哪一个应该用于精灵）。
> <div style="text-align:left"><img src="./images/ch6.4/image_3_control_register_highlight.png" width="50%"/></div>

除了瓦片的 960 字节外，命名表还包含 64 字节用于指定调色板，我们将在后面讨论。总的来说，单个帧定义为 1024 字节 (960 + 64)。PPU VRAM 可以同时保存两个命名表——两个帧的状态。

PPU 地址空间中存在的两个附加命名表必须映射到现有表或磁带上的额外 RAM 空间。 
[更多细节](http://wiki.nesdev.com/w/index.php/Mirroring)。

名称表在程序执行期间由 CPU 填充（使用我们已经实现的 Addr 和 Data 寄存器）。这完全由游戏代码决定。我们需要做的就是读取 VRAM 的正确部分。

绘制当前背景的算法：
1) 确定当前屏幕正在使用哪个命名表（通过从控制寄存器中读取位 0 和位 1）
2) 确定哪个 CHR ROM bank 用于背景图块（通过从控制寄存器读取位 4）
3) 从指定的 nametable 中读取 960 个字节并绘制一个 32x30 的基于 tile 的屏幕

我们向 ```render``` 添加新的 `渲染` 模块：

```rust
pub mod frame;
pub mod palette;

use crate::ppu::NesPPU;
use frame::Frame;

pub fn render(ppu: &NesPPU, frame: &mut Frame) {
   let bank = ppu.ctrl.bknd_pattern_addr();

   for i in 0..0x03c0 { // just for now, lets use the first nametable
       let tile = ppu.vram[i] as u16;
       let tile_x = i % 32;
       let tile_y = i / 32;
       let tile = &ppu.chr_rom[(bank + tile * 16) as usize..=(bank + tile * 16 + 15) as usize];

       for y in 0..=7 {
           let mut upper = tile[y];
           let mut lower = tile[y + 8];

           for x in (0..=7).rev() {
               let value = (1 & upper) << 1 | (1 & lower);
               upper = upper >> 1;
               lower = lower >> 1;
               let rgb = match value {
                   0 => palette::SYSTEM_PALLETE[0x01],
                   1 => palette::SYSTEM_PALLETE[0x23],
                   2 => palette::SYSTEM_PALLETE[0x27],
                   3 => palette::SYSTEM_PALLETE[0x30],
                   _ => panic!("can't be"),
               };
               frame.set_pixel(tile_x*8 + x, tile_y*8 + y, rgb)
           }
       }
   }
}

```

> 注意：我们仍然使用从系统调色板中随机挑选的颜色来查看形状

同样，我们需要拦截程序执行以读取屏幕状态。
在真实控制台上，PPU 在每个 PPU 时钟周期绘制一个像素。但是，我们可以走捷径。与其在每个 PPU 时钟滴答上读取部分屏幕状态，我们可以等到全屏准备好并一口气读取。

> **警告** 这是一个相当大的简化，它限制了可以在模拟器上玩的游戏类型。</br><br/>更高级的游戏使用了很多技巧来丰富游戏体验。
> 例如，更改框架中间的滚动 (<a href="https://wiki.nesdev.com/w/index.php/PPU_scrolling#Split_X_scroll">s拆分滚动</a>) 或更改调色板颜色。<br/><br/>
> 这种简化不会对第一代 NES 游戏产生太大影响。然而，大多数 NES 游戏在 PPU 仿真中需要更高的准确性。

在真实控制台上，PPU 在 0 - 240 条扫描线期间主动在电视屏幕上绘制屏幕状态；在扫描线 241 - 262 期间，CPU 正在为下一帧更新 PPU 的状态，然后循环重复。

一种拦截方法是在 NMI 中断之后立即读取屏幕状态 - 当 PPU 完成渲染当前帧时，但在 CPU 开始创建下一个帧之前。

首先让我们为总线添加回调，每次 PPU 触发 NMI 时都会调用它：

```rust
ub struct Bus<'call> {
   cpu_vram: [u8; 2048],
   prg_rom: Vec<u8>,
   ppu: NesPPU,

   cycles: usize,
   gameloop_callback: Box<dyn FnMut(&NesPPU) + 'call>,

}

impl<'a> Bus<'a> {
   pub fn new<'call, F>(rom: Rom, gameloop_callback: F) -> Bus<'call>
   where
       F: FnMut(&NesPPU) + 'call,
   {
       let ppu = NesPPU::new(rom.chr_rom, rom.screen_mirroring);

       Bus {
           cpu_vram: [0; 2048],
           prg_rom: rom.prg_rom,
           ppu: ppu,
           cycles: 0,
           gameloop_callback: Box::from(gameloop_callback),
       }
   }
}
```

然后让我们调整 ```tick``` 函数：

```rust
impl<'a> Bus<'a> {
//..
   pub fn tick(&mut self, cycles: u8) {
        self.cycles += cycles as usize;

        let nmi_before = self.ppu.nmi_interrupt.is_some();
        self.ppu.tick(cycles *3);
        let nmi_after = self.ppu.nmi_interrupt.is_some();
        
        if !nmi_before && nmi_after {
            (self.gameloop_callback)(&self.ppu, &mut self.joypad1);
        }
   }
}
```

然后我们可以连接游戏循环、中断回调和渲染函数：

```rust
fn main() {
   // init sdl2…

   //load the game
   let bytes: Vec<u8> = std::fs::read("game.nes").unwrap();
   let rom = Rom::new(&bytes).unwrap();

   let mut frame = Frame::new();

   // the game cycle
   let bus = Bus::new(rom, move |ppu: &NesPPU| {
       render::render(ppu, &mut frame);
       texture.update(None, &frame.data, 256 * 3).unwrap();

       canvas.copy(&texture, None, None).unwrap();

       canvas.present();
       for event in event_pump.poll_iter() {
           match event {
             Event::Quit { .. }
             | Event::KeyDown {
                 keycode: Some(Keycode::Escape),
                 ..
             } => std::process::exit(0),
             _ => { /* do nothing */ }
           }
        }
   });

   let mut cpu = CPU::new(bus);

   cpu.reset();
   cpu.run();
}
```

它正在工作！漂~ 亮~~。


<div style="text-align:center"><img src="./images/ch6.4/image_4_pacman_result.png" width="30%"/></div>

现在让我们修复颜色。

## 使用颜色

NES游戏机可以在电视屏幕上生成 52 种不同的颜色。这些颜色构成了控制台的硬连线系统调色板。

但是，单个屏幕只能同时使用 25 种颜色：13 种背景颜色和 12 种用于精灵。

NES 有内存 RAM 来存储调色板设置。
该空间分为 8 个调色板表：4 个用于背景，4 个用于精灵。每个调色板包含三种颜色。
请记住，图块中的像素是使用 2 位编码的 - 这是 4 个可能的值。0b00 是一个特殊的。

> *背景*图块的 **0b00** 表示使用通用背景颜色（存储在 **0x3F00**）。
>
> 对于精灵 - **0b00** 意味着像素是透明的


<div style="text-align:center"><img src="./images/ch6.4/image_5_palette_table.png" width="100%"/></div>

可以仅使用调色板表中的一个调色板来绘制单个图块。
对于背景图块，每个名称表的最后 64 字节保留用于将特定调色板分配给背景的一部分。此部分称为属性表。

属性表中的一个字节控制 4 个相邻元图块的调色板。（元瓦片是由 2x2 瓦片组成的空间）
换句话说，1 个字节控制哪些调色板用于 4x4 瓦片块或 32x32 像素
一个字节被分成四个 2 位块，每个块分配一个背景四个相邻瓷砖的调色板。

<div style="text-align:center"><img src="./images/ch6.4/image_6_attribute_table.png" width="70%"/></div>

首先，让我们提取由其在屏幕上的行和列位置指定的背景图块的调色板：

```rust
fn bg_pallette(ppu: &NesPPU, tile_column: usize, tile_row : usize) -> [u8;4] {
   let attr_table_idx = tile_row / 4 * 8 +  tile_column / 4;
   let attr_byte = ppu.vram[0x3c0 + attr_table_idx];  // note: still using hardcoded first nametable

   let pallet_idx = match (tile_column %4 / 2, tile_row % 4 / 2) {
       (0,0) => attr_byte & 0b11,
       (1,0) => (attr_byte >> 2) & 0b11,
       (0,1) => (attr_byte >> 4) & 0b11,
       (1,1) => (attr_byte >> 6) & 0b11,
       (_,_) => panic!("should not happen"),
   };

   let pallete_start: usize = 1 + (pallet_idx as usize)*4;
   [ppu.palette_table[0], ppu.palette_table[pallete_start], ppu.palette_table[pallete_start+1], ppu.palette_table[pallete_start+2]]
}
```

只需将我们的颜色查找 `render` 函数从使用随机选择的颜色重新连接到实际颜色即可：

```rust
pub fn render(ppu: &NesPPU, frame: &mut Frame) {
   let bank = ppu.ctrl.bknd_pattern_addr();

   for i in 0..0x3c0 {
       let tile = ppu.vram[i] as u16;
       let tile_column = i % 32;
       let tile_row = i / 32;
       let tile = &ppu.chr_rom[(bank + tile * 16) as usize..=(bank + tile * 16 + 15) as usize];
       let palette = bg_pallette(ppu, tile_column, tile_row);

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
               frame.set_pixel(tile_column * 8 + x, tile_row * 8 + y, rgb)
           }
       }
   }
}
```

就是这样。

## 渲染精灵。

渲染精灵有点相似，但更容易一些。
NES 有一个内部 RAM，用于存储帧中所有精灵的状态，即所谓的对象属性存储器 (OAM)。

它有 256 字节的 RAM，并为每个精灵保留 4 字节。这提供了在屏幕上同时显示 64 个图块的选项（但请记住，屏幕上的单个对象通常至少包含 3-4 个图块）。

CPU 必须选择更新 OAM 表：
- 使用 OAM Addr 和 OAM Data PPUT 寄存器，一次更新一个字节。
- 通过使用 OAM DMA 从 CPU RAM 传输 256 个字节来批量更新整个表

与背景图块相比，精灵图块可以显示在 256x240 屏幕中的任何位置。每个 OAM 记录有 2 个字节为 X 和 Y 坐标保留，一个字节用于从模式表中选择一个平铺模式。剩下的字节指定应该如何绘制对象（例如，PPU 可以水平或垂直翻转同一个图块）

NES Dev Wiki 为[OAM 记录中的每个字节](http://wiki.nesdev.com/w/index.php/PPU_OAM)提供了相当可靠的规范

要渲染所有可见的精灵，我们只需要扫描 oam_data 空间并将每 4 个字节解析为一个精灵：

```rust

pub fn render(ppu: &NesPPU, frame: &mut Frame) {

//.. draw background
//draw sprites
   for i in (0..ppu.oam_data.len()).step_by(4).rev() {
       let tile_idx = ppu.oam_data[i + 1] as u16;
       let tile_x = ppu.oam_data[i + 3] as usize;
       let tile_y = ppu.oam_data[i] as usize;

       let flip_vertical = if ppu.oam_data[i + 2] >> 7 & 1 == 1 {
           true
       } else {
           false
       };
       let flip_horizontal = if ppu.oam_data[i + 2] >> 6 & 1 == 1 {
           true
       } else {
           false
       };
       let pallette_idx = ppu.oam_data[i + 2] & 0b11;
       let sprite_palette = sprite_palette(ppu, pallette_idx);
      
       let bank: u16 = ppu.ctrl.sprt_pattern_addr();

       let tile = &ppu.chr_rom[(bank + tile_idx * 16) as usize..=(bank + tile_idx * 16 + 15) as usize];


       for y in 0..=7 {
           let mut upper = tile[y];
           let mut lower = tile[y + 8];
           'ololo: for x in (0..=7).rev() {
               let value = (1 & lower) << 1 | (1 & upper);
               upper = upper >> 1;
               lower = lower >> 1;
               let rgb = match value {
                   0 => continue 'ololo, // skip coloring the pixel
                   1 => palette::SYSTEM_PALLETE[sprite_palette[1] as usize],
                   2 => palette::SYSTEM_PALLETE[sprite_palette[2] as usize],
                   3 => palette::SYSTEM_PALLETE[sprite_palette[3] as usize],
                   _ => panic!("can't be"),
               };
               match (flip_horizontal, flip_vertical) {
                   (false, false) => frame.set_pixel(tile_x + x, tile_y + y, rgb),
                   (true, false) => frame.set_pixel(tile_x + 7 - x, tile_y + y, rgb),
                   (false, true) => frame.set_pixel(tile_x + x, tile_y + 7 - y, rgb),
                   (true, true) => frame.set_pixel(tile_x + 7 - x, tile_y + 7 - y, rgb),
               }
           }
       }
   }
```

精灵调色板查找非常简单：

```rust
fn sprite_palette(ppu: &NesPPU, pallete_idx: u8) -> [u8; 4] {
    let start = 0x11 + (pallete_idx * 4) as usize;
    [
        0,
        ppu.palette_table[start],
        ppu.palette_table[start + 1],
        ppu.palette_table[start + 2],
    ]
}
```

<div style="text-align:center"><img src="./images/ch6.4/image_7_pacman_chrs.png" width="30%"/></div>


好的。现在看起来好多了。

<br/>

------

> 本章完整源代码： <a href="https://github.com/bugzmanov/nes_ebook/tree/master/code/ch6.4" target="_blank">GitHub</a>
