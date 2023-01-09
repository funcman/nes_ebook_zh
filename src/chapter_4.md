# 模拟总线

 <div style="text-align:center"><img src="./images/ch4/image_1_bus_schema.png" width="60%"/></div>

CPU 使用三种总线访问内存（包括内存映射空间）：
* 地址总线承载所需位置的地址
* 控制总线通知它是读还是写访问
* 数据总线承载正在读取或写入的数据字节

总线本身不是设备；它是平台组件之间的连线。因此，我们不需要将它作为一个独立的模块来实现，因为 Rust 允许我们直接“连接”组件。
然而，它是一个方便的抽象，我们可以卸下相当多的责任来保持 CPU 代码的清洁。

 <div style="text-align:center"><img src="./images/ch4/image_2_cpu_pinout_2.png" width="50%"/></div>

在我们当前的代码中，CPU 可以直接访问 RAM 空间，并且忽略了内存映射区域。

通过引入 Bus 模块，我们可以有一个地方用于：
* 设备内通信：
    * 数据读/写
    * 将硬件中断路由到 CPU（稍后会详细介绍）
* 处理内存映射
* 协调 PPU 和 CPU 时钟周期（稍后会详细介绍）

好消息是我们不需要编写数据、控制和地址总线的全面仿真。因为它不是硬件芯片，所以没有逻辑期望来自 BUS 的任何特定行为。所以我们可以只编写协调和信号路由。

现在，我们可以实现它的基本框架：
* 访问 CPU RAM
* 镜像

镜像是 NES 试图保持尽可能便宜的副作用。它可以看作是一个地址空间被映射到另一个地址空间。

例如，在 CPU 内存映射 RAM 地址空间 **[0x000 .. 0x0800]** (2 KiB) 被镜像 3 次：
* **[0x800 .. 0x1000]**
* **[0x1000 .. 0x1800]**
* **[0x1800 .. 0x2000]**

这意味着在读取或写入时访问 0x0000 或 0x0800 或 0x1000 或 0x1800 的内存地址没有区别。

镜像的原因是 CPU RAM 只有 2 KiB 的 ram 空间，并且只有 11 位足以寻址 RAM 空间。自然地，NES 主板只有 11 个从 CPU 到 RAM 的寻址轨道。


 <div style="text-align:center"><img src="./images/ch4/image_3_cpu_ram_connection.png" width="70%"/></div>

然而，CPU 有 **[0x0000 - 0x2000]** 为 RAM 空间保留的寻址空间 - 这是 13 位。因此，在访问 RAM 时，最高 2 位无效。
换一种说法，当 CPU 请求地址**0b0001_1111_1111_1111** （13 位）时，RAM 芯片将通过地址总线仅接收 **0b111_1111_1111**（11 位）。

因此，尽管镜像看起来很浪费，但它是布线的副作用，在真正的硬件上它没有任何成本。另一方面，模拟器必须做额外的工作才能提供相同的行为。

长话短说，如果 BUS 收到 **[0x0000 … 0x2000]** 范围内的请求，则需要将最高 2 位清零

类似地，地址空间 **[0x2008 .. 0x4000]** 镜像了 PPU 寄存器 **[0x2000 .. 0x2008]** 的内存映射。这些是 BUS 将负责的仅有的两个镜像。让我们立即对其进行编码，即使我们还没有 PPU 的任何东西。

因此，让我们介绍一个新模块 Bus，它可以直接访问 RAM。

```rust
pub struct Bus {
   cpu_vram: [u8; 2048]
}

impl Bus {
   pub fn new() -> Self{
       Bus {
           cpu_vram: [0; 2048]
       }
   }
}
```

总线还将提供读/写访问：

```rust
const RAM: u16 = 0x0000;
const RAM_MIRRORS_END: u16 = 0x1FFF;
const PPU_REGISTERS: u16 = 0x2000;
const PPU_REGISTERS_MIRRORS_END: u16 = 0x3FFF;

impl Mem for Bus {
   fn mem_read(&self, addr: u16) -> u8 {
       match addr {
           RAM ..= RAM_MIRRORS_END => {
               let mirror_down_addr = addr & 0b00000111_11111111;
               self.cpu_vram[mirror_down_addr as usize]
           }
           PPU_REGISTERS ..= PPU_REGISTERS_MIRRORS_END => {
               let _mirror_down_addr = addr & 0b00100000_00000111;
               todo!("PPU is not supported yet")
           }
           _ => {
               println!("Ignoring mem access at {}", addr);
               0
           }
       }
   }

   fn mem_write(&mut self, addr: u16, data: u8) {
       match addr {
           RAM ..= RAM_MIRRORS_END => {
               let mirror_down_addr = addr & 0b11111111111;
               self.cpu_vram[mirror_down_addr as usize] = data;
           }
           PPU_REGISTERS ..= PPU_REGISTERS_MIRRORS_END => {
               let _mirror_down_addr = addr & 0b00100000_00000111;
               todo!("PPU is not supported yet");
           }
           _ => {
               println!("Ignoring mem write-access at {}", addr);
           }
       }
   }
}
```

最后一步是将 CPU 对 RAM 的直接访问替换为通过 BUS 访问

```rust
pub struct CPU {
   pub register_a: u8,
   pub register_x: u8,
   pub register_y: u8,
   pub status: CpuFlags,
   pub program_counter: u16,
   pub stack_pointer: u8,
   pub bus: Bus,
}


impl Mem for CPU {
   fn mem_read(&self, addr: u16) -> u8 {
       self.bus.mem_read(addr)
   }

   fn mem_write(&mut self, addr: u16, data: u8) {
       self.bus.mem_write(addr, data)
   }
   fn mem_read_u16(&self, pos: u16) -> u16 {
       self.bus.mem_read_u16(pos)
   }
 
   fn mem_write_u16(&mut self, pos: u16, data: u16) {
       self.bus.mem_write_u16(pos, data)
   }
}

impl CPU {
   pub fn new() -> Self {
       CPU {
           register_a: 0,
           register_x: 0,
           register_y: 0,
           stack_pointer: STACK_RESET,
           program_counter: 0,
           status: CpuFlags::from_bits_truncate(0b100100),
           bus: Bus::new(),
       }
   }
   // ...
}
```

现在就差不多了。不难，对吧？

<br/>

------

> 本章完整源代码： <a href="https://github.com/bugzmanov/nes_ebook/tree/master/code/ch4" target="_blank">GitHub</a>
