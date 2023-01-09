# 渲染 CHR ROM 切片

PPU 上的地址空间 **[0x0 .. 0x2000]** 是为 CHR ROM 保留的，其中包含游戏的视觉图形数据。

那是 *8 KiB* 的数据。这就是 NES 卡带的第一个版本中的全部内容。

视觉数据打包在所谓的图块中：一个 8 x 8 像素的图像，最多可以使用 4 种颜色。（准确地说，背景瓦片可以有4种颜色，一个精灵瓦片可以有3种颜色，0b00用来表示一个像素应该是透明的）

```bash
8 * 8 * 2 (用2比特来编码颜色) = 128 bits = 16个字节来编排一个瓦片
```

8 KiB / 128 位 = 512 块。即，每个卡带总共包含 512 个图块，分为 2 页/组。组并没有真正的名字，历史上它们被称为“左”和“右”。


<div style="text-align:center"><img src="./images/ch6.3/image_1_mario_tiles.png" width="50%"/></div>

8 像素 x 8 像素是一个很小的尺寸，无法以这种方式呈现。NES 游戏中的大多数对象由多个图块组成。

 <div style="text-align:center"><img src="./images/ch6.3/image_2_8bit_drawings.png" width="30%" ><br/><a href="https://twitter.com/johanvinet">Johan Vinet</a> 的 8x8 像素艺术<br/><a href="https://twitter.com/PixelProspector/status/1097565152940044293">[查看]</a> </div>

CHR 格式的棘手之处在于图块本身不包含任何颜色信息。平铺中的每个像素都使用 2 位进行编码，在调色板中声明颜色索引，而不是颜色本身。

> 如果 NES 对每个像素使用流行的 RGB 格式，则单个图块将占用 8 8 24 = 192 个字节。它需要 96 KiB 的 CHR ROM 空间来容纳 512 个切片。

像素的真实颜色是在渲染阶段通过使用所谓的调色板决定的，稍后会详细介绍。

通过读取 CHR ROM，不可能得出颜色，只能得出形状。

<div style="text-align:center"><img src="./images/ch6.3/image_3_chr_content.png" width="30%"/></div>

<div style="text-align:center"><img src="./images/ch6.3/image_4_color_code.png" width="50%"/></div>

令人惊讶的是，一个像素的 2 位没有编码在同一个字节中。使用 16 个字节描述一个 tile。每一行都使用 2 个字节进行编码，这些字节彼此相隔 8 个字节。
需要指出的是，为了计算左上角像素的颜色索引，我们需要读取字节 0x0000 的第 7 位和字节 0x0008 的第 7 位，要获得同一行中的下一个像素，我们需要读取同一行中的第 6 位字节等。


<div style="text-align:center"><img src="./images/ch6.3/image_5_16bytes_of_a_tile.png" width="50%"/></div>


## 调色板

在渲染 CHR ROM 内容之前，我们需要简要讨论一下 NES 可用的颜色。
不同版本的 PPU 芯片的 52 种硬连线颜色的系统级调色板略有不同。

所有必要的细节都可以在相应的[NesDev wiki页面](http://wiki.nesdev.com/w/index.php/PPU_palettes#Palettes)上找到。


<div style="text-align:center"><img src="./images/ch6.3/image_6_system_palette.png" width="50%"/></div>

模拟器中使用了多种变体。有些使图片更具视觉吸引力，而另一些则使其更接近电视上生成的原始图片 NES。

我们选择哪一种并不重要，它们中的大多数都能为我们带来足够好的结果。

但是，我们仍然需要将该表编码为 SDL2 库可识别的 RGB 格式：

```rust
#[rustfmt::skip]

pub static SYSTEM_PALLETE: [(u8,u8,u8); 64] = [
   (0x80, 0x80, 0x80), (0x00, 0x3D, 0xA6), (0x00, 0x12, 0xB0), (0x44, 0x00, 0x96), (0xA1, 0x00, 0x5E),
   (0xC7, 0x00, 0x28), (0xBA, 0x06, 0x00), (0x8C, 0x17, 0x00), (0x5C, 0x2F, 0x00), (0x10, 0x45, 0x00),
   (0x05, 0x4A, 0x00), (0x00, 0x47, 0x2E), (0x00, 0x41, 0x66), (0x00, 0x00, 0x00), (0x05, 0x05, 0x05),
   (0x05, 0x05, 0x05), (0xC7, 0xC7, 0xC7), (0x00, 0x77, 0xFF), (0x21, 0x55, 0xFF), (0x82, 0x37, 0xFA),
   (0xEB, 0x2F, 0xB5), (0xFF, 0x29, 0x50), (0xFF, 0x22, 0x00), (0xD6, 0x32, 0x00), (0xC4, 0x62, 0x00),
   (0x35, 0x80, 0x00), (0x05, 0x8F, 0x00), (0x00, 0x8A, 0x55), (0x00, 0x99, 0xCC), (0x21, 0x21, 0x21),
   (0x09, 0x09, 0x09), (0x09, 0x09, 0x09), (0xFF, 0xFF, 0xFF), (0x0F, 0xD7, 0xFF), (0x69, 0xA2, 0xFF),
   (0xD4, 0x80, 0xFF), (0xFF, 0x45, 0xF3), (0xFF, 0x61, 0x8B), (0xFF, 0x88, 0x33), (0xFF, 0x9C, 0x12),
   (0xFA, 0xBC, 0x20), (0x9F, 0xE3, 0x0E), (0x2B, 0xF0, 0x35), (0x0C, 0xF0, 0xA4), (0x05, 0xFB, 0xFF),
   (0x5E, 0x5E, 0x5E), (0x0D, 0x0D, 0x0D), (0x0D, 0x0D, 0x0D), (0xFF, 0xFF, 0xFF), (0xA6, 0xFC, 0xFF),
   (0xB3, 0xEC, 0xFF), (0xDA, 0xAB, 0xEB), (0xFF, 0xA8, 0xF9), (0xFF, 0xAB, 0xB3), (0xFF, 0xD2, 0xB0),
   (0xFF, 0xEF, 0xA6), (0xFF, 0xF7, 0x9C), (0xD7, 0xE8, 0x95), (0xA6, 0xED, 0xAF), (0xA2, 0xF2, 0xDA),
   (0x99, 0xFF, 0xFC), (0xDD, 0xDD, 0xDD), (0x11, 0x11, 0x11), (0x11, 0x11, 0x11)
];
 ```

 ## 渲染 CHR Rom

要从 CHR ROM 渲染图块，我们需要获取游戏的 ROM 文件。谷歌会帮你找到很多著名经典的 ROM 转储。但是，如果您没有卡带，则下载此类 ROM 是非法的（眨眼眨眼）。
有一个网站列出了最近开发的合法自制游戏。而且有些还不错，大部分都是免费的。看看：[www.nesworld.com](http://www.nesworld.com/article.php?system=nes&data=neshomebrew)

这里需要注意的是，我们的模拟器仅支持 NES 1.0 格式。并且自制开发的游戏倾向于使用 NES 2.0。
像“Alter Ego”这样的游戏就可以了。

我会使用吃豆人，主要是因为它很容易辨认，而且我碰巧拥有这款游戏的卡带。

首先，让我们为一个框架创建一个抽象层，这样我们就不需要直接使用 SDL：

```rust
pub struct Frame {
   pub data: Vec<u8>,
}

impl Frame {
   const WIDTH: usize = 256;
   const HIGHT: usize = 240;

   pub fn new() -> Self {
       Frame {
           data: vec![0; (Frame::WIDTH) * (Frame::HIGHT) * 3],
       }
   }

   pub fn set_pixel(&mut self, x: usize, y: usize, rgb: (u8, u8, u8)) {
       let base = y * 3 * Frame::WIDTH + x * 3;
       if base + 2 < self.data.len() {
           self.data[base] = rgb.0;
           self.data[base + 1] = rgb.1;
           self.data[base + 2] = rgb.2;
       }
   }
}
```


现在我们准备好在框架上渲染图块：

```rust
fn show_tile(chr_rom: &Vec<u8>, bank: usize, tile_n: usize) ->Frame {
   assert!(bank <= 1);

   let mut frame = Frame::new();
   let bank = (bank * 0x1000) as usize;

   let tile = &chr_rom[(bank + tile_n * 16)..=(bank + tile_n * 16 + 15)];

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
           frame.set_pixel(x, y, rgb)
       }
   }

   frame
}
```

> **注意：** 目前，我们正在随机解释颜色索引。只需从系统调色板中为每个索引值选择 4 种随机颜色即可查看它的外观。

在主循环中将它们捆绑在一起：

```rust
fn main() {
   // ….init sdl2
   // ....

   //load the game
   let bytes: Vec<u8> = std::fs::read("pacman.nes").unwrap();
   let rom = Rom::new(&bytes).unwrap();

   let tile_frame = show_tile(&rom.chr_rom, 1,0);

   texture.update(None, &tile_frame.data, 256 * 3).unwrap();
   canvas.copy(&texture, None, None).unwrap();
   canvas.present();

   loop {
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
   }
}
```

结果并不那么令人印象深刻：

<div style="text-align:center"><img src="./images/ch6.3/image_7_tile_1.png" width="50%"/></div>

可能是吃豆人的后背……呃……头？谁知道。

我们可以稍微调整一下代码以从 CHR ROM 中绘制所有图块：

<div style="text-align:center"><img src="./images/ch6.3/image_8_pacman_chr_rom.png" width="80%"/></div>


啊哈！尽管颜色明显不同，但现在可以识别形状。我们可以看到部分幽灵、一些字母和一些数字。
我猜就是这样。继续...


<br/>

------

> 本章完整源代码： <a href="https://github.com/bugzmanov/nes_ebook/tree/master/code/ch6.3" target="_blank">GitHub</a>
