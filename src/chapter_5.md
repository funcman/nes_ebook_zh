# 卡带

 <div style="text-align:center"><img src="./images/ch5/image_1_a_cartridge.png" width="20%"/></div>

卡带的第一个版本相对简单。他们携带两组 ROM 存储器：用于代码的 PRG ROM 和用于可视图形的 CHR ROM。

插入控制台后，PRG ROM 连接到 CPU，CHR ROM 连接到 PPU。所以在硬件层面，CPU不能直接访问CHR ROM，PPU不能访问PRG ROM。

> 更高版本的卡带有额外的硬件：
> * mappers to provide access to extended ROM memory: both CHR ROM and PRG ROM
> * extra RAM (with a battery) to save and restore a game state

但是，我们不会使用卡带。模拟器使用包含 ROM 空间转储的文件。

ROM 转储有多种文件格式；最受欢迎的是由[Marat Fayzullin](http://fms.komkon.org)设计的 iNES

 <div style="text-align:center"><img src="./images/ch5/image_2_ines_file_format.png" width="60%"/></div>

该文件包含 3-4 个部分：
* 16 字节标头
* 可选 512 字节的所谓 Trainer，由 Famicom 复印机创建的一个数据部分，用于保存自己的映射。如果存在，我们可以跳过此部分。
* 包含 PRG ROM 代码的部分
* 包含 CHR ROM 数据的部分

标题是最有趣的部分。

 <div style="text-align:center"><img src="./images/ch5/image_3_ines_header.png" width="60%"/></div>

控制字节 1 和控制字节 2（标头中的字节 06 和 07）包含有关文件中数据的一些附加信息，但它以位打包。

|   |   |
|---|---|
| <div style="text-align:center"><img src="./images/ch5/image_4_control_byte_1.png" width="100%"/></div> | <div style="text-align:center"><img src="./images/ch5/image_5_control_byte_2.png" width="100%"/></div>       |


<br/>
我们不会涵盖和支持 iNES 2.0 格式，因为它不是很流行。但是您可以找到<a href="https://formats.kaitai.io/ines/index.html">两个 iNES 版本的正式规范</a>。

我们关心的最基本信息：
* PRG ROM
* CHR ROM
* 映射器类型
* 镜像类型：水平，垂直，4屏幕

镜像将在以下 PPU 章节中广泛介绍。
现在，我们需要弄清楚游戏使用的是哪种镜像类型。

我们将仅支持 iNES 1.0 格式和映射器 0。

映射器 0 本质上意味着 CPU 按原样读取 CHR 和 PRG ROM 的“无映射器”。

让我们定义卡带 Rom 数据结构：

```rust
#[derive(Debug, PartialEq)]
pub enum Mirroring {
   VERTICAL,
   HORIZONTAL,
   FOUR_SCREEN,
}

pub struct Rom {
   pub prg_rom: Vec<u8>,
   pub chr_rom: Vec<u8>,
   pub mapper: u8,
   pub screen_mirroring: Mirroring,
}

```

然后我们需要编写代码来解析二进制数据：

```rust
impl Rom {
   pub fn new(raw: &Vec<u8>) -> Result<Rom, String> {
       if &raw[0..4] != NES_TAG {
           return Err("File is not in iNES file format".to_string());
       }

       let mapper = (raw[7] & 0b1111_0000) | (raw[6] >> 4);

       let ines_ver = (raw[7] >> 2) & 0b11;
       if ines_ver != 0 {
           return Err("NES2.0 format is not supported".to_string());
       }

       let four_screen = raw[6] & 0b1000 != 0;
       let vertical_mirroring = raw[6] & 0b1 != 0;
       let screen_mirroring = match (four_screen, vertical_mirroring) {
           (true, _) => Mirroring::FOUR_SCREEN,
           (false, true) => Mirroring::VERTICAL,
           (false, false) => Mirroring::HORIZONTAL,
       };

       let prg_rom_size = raw[4] as usize * PRG_ROM_PAGE_SIZE;
       let chr_rom_size = raw[5] as usize * CHR_ROM_PAGE_SIZE;

       let skip_trainer = raw[6] & 0b100 != 0;

       let prg_rom_start = 16 + if skip_trainer { 512 } else { 0 };
       let chr_rom_start = prg_rom_start + prg_rom_size;

       Ok(Rom {
           prg_rom: raw[prg_rom_start..(prg_rom_start + prg_rom_size)].to_vec(),
           chr_rom: raw[chr_rom_start..(chr_rom_start + chr_rom_size)].to_vec(),
           mapper: mapper,
           screen_mirroring: screen_mirroring,
       })
   }
}

```

一如既往，别忘了测试！


接下来，将 Rom 连接到 BUS：

```rust 
pub struct Bus {
   cpu_vram: [u8; 2048],
   rom: Rom,
}

impl Bus {
   pub fn new(rom: Rom) -> Self {
       Bus {
           cpu_vram: [0; 2048],
           rom: rom,
       }
   }
   //....
}
```


最后，我们需要将地址空间 **[0x8000 … 0x10000]** 映射到卡带 PRG ROM 空间。

一个警告：PRG Rom 大小可能是 16 KiB 或 32 KiB。因为 **[0x8000 … 0x10000]** 映射区域是 32 KiB 的可寻址空间，所以需要将高 16 KiB 映射到低 16 KiB（如果游戏只有 16 KiB 的 PRG ROM）

```rust 
impl Mem for Bus {
   fn mem_read(&self, addr: u16) -> u8 {
       match addr {
           //….
           0x8000..=0xFFFF => self.read_prg_rom(addr),
       }
   }

   fn mem_write(&mut self, addr: u16, data: u8) {
       match addr {
           //...
           0x8000..=0xFFFF => {
               panic!("Attempt to write to Cartridge ROM space")
           }
       }
   }
}

impl Bus {
  // ...

   fn read_prg_rom(&self, mut addr: u16) -> u8 {
       addr -= 0x8000;
       if self.rom.prg_rom.len() == 0x4000 && addr >= 0x4000 {
           //mirror if needed
           addr = addr % 0x4000;
       }
       self.rom.prg_rom[addr as usize]
   }
}
```

你可以在[Github](https://github.com/bugzmanov/nes_ebook/blob/master/code/ch5/snake.nes?raw=true)上下载你的第一个 NES ROM 转储文件。

您将需要修改 `main` 从文件加载二进制文件的方法。


剧透警告：这是对贪吃蛇游戏的修改，具有更有趣的物理特性。游戏要求输入设备、屏幕输出和随机数生成器使用相同的内存映射。

<br/>

------

> 本章完整源代码： <a href="https://github.com/bugzmanov/nes_ebook/tree/master/code/ch5" target="_blank">GitHub</a>
