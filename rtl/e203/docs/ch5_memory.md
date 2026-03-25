# 第5章：存储子系统（TCM + SRAM）

> **本章目标**：了解 e203 的"记忆"是如何工作的。ITCM 和 DTCM 是什么、SRAM 的时序是怎样的、ECC 是怎么检错纠错的，以及如何通过配置参数调整存储容量。

---

## 5.1 为什么用 TCM 而不是 Cache？

### 嵌入式场景的需求

在 IoT、工业控制等嵌入式场景中，有两个关键需求经常冲突：

1. **实时性**：中断响应必须在确定的时间内完成（称为"实时保证"）
2. **低功耗**：电池供电，每毫瓦都要省

**Cache（缓存）** 的问题：
- Cache 命中（hit）：1-2 拍
- Cache 缺失（miss）：10-100 拍（去外部 DDR 取数据）
- → 延迟**不确定**，无法保证实时性！

**TCM（Tightly Coupled Memory，紧耦合存储器）** 的优势：
- 访问延迟**固定为 1 拍**，无论什么情况
- 完全可预测，满足实时系统要求
- 离 CPU 核心极近，功耗低于外部 DRAM

| 对比项 | Cache | TCM |
|--------|-------|-----|
| 容量 | 可以很大（配外部DRAM） | 较小（片上SRAM，受面积限制）|
| 延迟 | 不确定（1~100拍） | **固定 1 拍** |
| 实时性 | 差（难以保证最差情况）| **好** |
| 功耗 | 较高（维护替换策略电路）| 低 |
| 适用场景 | 通用处理器、运行大程序 | 嵌入式实时系统 |

e203 的选择：**不要 Cache，只用 TCM**——牺牲面向大内存的效率，换取确定性延迟和低功耗。

---

## 5.2 ITCM：指令紧耦合存储器

### 为什么是 64bit 宽？

ITCM 以 **64bit（8字节）** 为一行存储数据，而不是普通的 32bit。原因：

```
标准 32bit 宽 SRAM：
  每次取指读 32bit = 1条指令
  2条指令需要 2次 SRAM 访问

e203 64bit 宽 ITCM：
  每次取指读 64bit = 2条 32bit 指令（或 4条 16bit 压缩指令）
  连续执行2条指令只需 1次 SRAM 访问 → 节省功耗！
```

这是 e203 低功耗设计的体现：减少 SRAM 激活次数。

### ITCM 的容量计算

配置参数：`E203_CFG_ITCM_ADDR_WIDTH`（默认 16，表示 64KB）

```
ITCM 容量 = 2^addr_width 字节

addr_width = 16 →  2^16  = 64KB    （默认配置）
addr_width = 20 →  2^20  = 1MB
addr_width = 21 →  2^21  = 2MB

SRAM 行数（深度）= 容量字节数 / 8（每行 8 字节 = 64bit）
  depth = 2^addr_width / 8 = 2^(addr_width-3)

addr_width = 16 → depth = 2^13 = 8192  行
addr_width = 20 → depth = 2^17 = 131072 行
```

对应 e203_defines.v 中的宏：

```verilog
`define E203_ITCM_RAM_DP  (1 << (`E203_CFG_ITCM_ADDR_WIDTH - 3))  // 行数 = 8192
`define E203_ITCM_RAM_AW  (`E203_CFG_ITCM_ADDR_WIDTH - 3)         // 行地址宽度 = 13
`define E203_ITCM_DATA_WIDTH  64                                    // 数据宽度：64bit
`define E203_ITCM_RAM_DW  `E203_ITCM_DATA_WIDTH                    // = 64
`define E203_ITCM_RAM_MW  (`E203_ITCM_DATA_WIDTH/8)                // 写掩码宽度 = 8
```

### ITCM 的双访问者：IFU 和 LSU

ITCM 有**两个主人**：

```
IFU (ifu2itcm_icb_*)  ──┐
                         ├──► ITCM 控制器（仲裁）──► SRAM
LSU (lsu2itcm_icb_*)  ──┘
```

仲裁策略：**LSU 优先**（实际代码里是优先级仲裁）。

为什么？通常 IFU 连续取指，若遇到 cache miss 的话，就会有 replay 机制重新尝试。而 LSU 的访问通常是程序逻辑强依赖的（不能随便延迟），所以 LSU 优先。

---

## 5.3 DTCM：数据紧耦合存储器

### 为什么是 32bit 宽？

DTCM 只有 **32bit（4字节）** 宽，这也是刻意为之：

```
ITCM（64bit）：需要同时读两条指令，带宽要求高
DTCM（32bit）：每次访存只需要最多 4 字节，32bit 足够
              如果做成 64bit，读一个字节也要激活整个 64bit 行 → 浪费功耗
```

### DTCM 的容量计算

配置参数：`E203_CFG_DTCM_ADDR_WIDTH`（默认 16，表示 64KB）

```
DTCM 容量 = 2^addr_width 字节

SRAM 行数 = 容量字节数 / 4 = 2^(addr_width-2)

addr_width = 16 → depth = 2^14 = 16384 行
addr_width = 14 → depth = 2^12 = 4096  行 (16KB)
addr_width = 18 → depth = 2^16 = 65536 行 (256KB)
```

对应宏：

```verilog
`define E203_DTCM_RAM_DP  (1 << (`E203_CFG_DTCM_ADDR_WIDTH - 2))  // 行数 = 16384
`define E203_DTCM_RAM_AW  (`E203_CFG_DTCM_ADDR_WIDTH - 2)         // 行地址宽度 = 14
`define E203_DTCM_DATA_WIDTH  32                                    // 数据宽度：32bit
`define E203_DTCM_RAM_DW  `E203_DTCM_DATA_WIDTH                    // = 32
`define E203_DTCM_RAM_MW  (`E203_DTCM_DATA_WIDTH/8)                // 写掩码宽度 = 4
```

---

## 5.4 SRAM 物理接口：每个信号的含义

SRAM 是最基础的存储元件，其接口信号（来自 `sirv_gnrl_ram.v` 及其上层）：

```
输入信号（控制器 → SRAM）：

  clk         时钟：所有操作在上升沿执行
  cs          Chip Select，片选：必须为 1 才激活 SRAM（省功耗的开关）
  we          Write Enable：1=写操作，0=读操作
  addr[AW-1:0] 行地址（ITCM是13bit，DTCM是14bit）
  din[DW-1:0]  写入数据（ITCM是64bit，DTCM是32bit）
  wem[MW-1:0]  Write Enable Mask，写使能掩码
               每 1 bit 控制 1 个字节是否被写入
               ITCM: 8bit 掩码（控制 8 个字节）
               DTCM: 4bit 掩码（控制 4 个字节）

  sd, ds, ls   SRAM 的特殊控制引脚（Sleep/Deep Sleep/Light Sleep）
               用于 SRAM 省电模式（深睡眠时关闭内部放大器等）

输出信号（SRAM → 控制器）：

  dout[DW-1:0] 读出数据（下一个时钟上升沿才有效！）
```

### SRAM 时序：同步读，1 拍延迟

```
时钟:    ___┌──┐  ___┌──┐  ___┌──┐  ___
             │  │     │  │     │  │
   ──────────┘  └─────┘  └─────┘  └──────

cs/addr: ═════╪═══════╪═══════╪══════════
             读请求1   读请求2   读请求3

dout:         ══════╪══════╪══════╪══════
                  数据1    数据2    数据3
                  ↑
              一拍之后才出来！
```

**关键理解**：在 Cycle N 发出读请求，数据在 Cycle N+1 的上升沿之后才能稳定输出。控制器必须"记住"上一拍发出了什么请求，才能正确接收这一拍的返回数据。

这就是为什么 ITCM/DTCM 控制器内部需要维护状态机——要知道响应对应哪个请求。

### "Hold-up"特性

SRAM 有一个有用的特性：**读出的数据在下一次访问之前会一直保持稳定**（"hold-up"）。

e203 在 ift2icb 模块里利用了这个特性：

```verilog
// ifu2itcm_holdup：ITCM 输出是否仍然保持着上次读的数据
// 如果 IFU 连续取指且新 PC 在同一个 64bit 块内，
// 就不需要再访问 SRAM，直接用上次保留的数据！
wire ifu_req_lane_holdup = ifu_req_pc2itcm & ifu2itcm_holdup & (~itcm_nohold);
```

**好处**：减少 SRAM 的 cs 激活次数，进一步降低动态功耗。

---

## 5.5 ECC：错误检测与纠正

e203 的 TCM 支持可选的 ECC（Error Correction Code，错误纠正码）。这是通过在 `config.v` 中定义 `E203_CFG_HAS_ECC` 来启用的。

### 什么是 ECC？

ECC 是一种在数据存储时添加"校验位"的技术：

```
原始数据: 32bit（例如 0xDEADBEEF）
ECC编码后: 32bit 数据 + 7bit 校验位 = 39bit 存入 SRAM

读出时：
  对 39bit 数据重新计算校验码
  如果一致 → 数据正确
  如果不一致 → 
    可以定位出错的 1bit，并自动纠正（单比特错误纠正）
    如果是 2bit 错误，则报错但无法纠正
```

e203 使用的是 **Hamming Code（汉明码）**，可以：
- 纠正 1bit 错误（SEC，Single Error Correct）
- 检测 2bit 错误（DED，Double Error Detect）

### ECC 在模块层次中的位置

```
itcm_ctrl / dtcm_ctrl 控制器
        │
        ▼ （写入时）
  sirv_sram_icb_ctrl  ← 在这里进行 ECC 编码
        ↓
  SRAM（存 数据+校验位）
        ↓（读出时）
  sirv_sram_icb_ctrl  ← 在这里进行 ECC 解码和纠错
        │
        ▼
  返回给 IFU / LSU（已纠错的数据）
```

---

## 5.6 时钟门控：SRAM 的低功耗开关

SRAM 的时钟是单独门控的，不是一直开着的：

```verilog
// e203_clkgate.v：时钟门控单元
module e203_clkgate (
  input   clk_in,
  input   test_mode,  // 测试模式：强制打开时钟
  input   clock_en,   // 使能信号
  output  clk_out     // 输出时钟
);

// 在时钟低电平时，对 clock_en 采样（保证无毛刺）
always @(*)
  if (!clk_in)
    enb = (clock_en | test_mode);

// 输出 = 使能 AND 时钟
assign clk_out = enb & clk_in;
```

**为什么要在时钟低电平时采样？**

```
时钟低电平时采样 enb：确保 enb 在时钟变高前就稳定了，
不会产生"毛刺"（glitch）——一个极短的时钟脉冲

错误示范（直接 AND）：
  clk_out = clock_en & clk_in
  → 如果 clock_en 恰好在 clk_in 变高时变化，可能产生毛刺 → 错误采样！

正确做法（latch 锁存）：
  在 clk_in = 0 时更新 enb
  clk_out = enb & clk_in
  → enb 在 clk_in = 1 期间保持稳定 → 无毛刺
```

对应到 e203 的 SRAM 时钟：
- `clk_itcm_ram`：ITCM SRAM 的门控时钟
- `clk_dtcm_ram`：DTCM SRAM 的门控时钟
- 空闲时关断 SRAM 时钟 → 每拍省下 SRAM 的时钟翻转功耗

---

## 5.7 模块层次：从 ICB 到 SRAM 的完整信号链

以 ITCM 为例，展示信号从 ICB 接口一路到物理 SRAM 的完整路径：

```
IFU / LSU（ICB Master）
  ifu2itcm_icb_cmd_valid
  ifu2itcm_icb_cmd_addr[15:0]
  ifu2itcm_icb_cmd_read
        │
        ▼
e203_itcm_ctrl（ICB Slave，ITCM 控制器）
  功能：
    • 仲裁 IFU 和 LSU 的访问竞争
    • ICB 握手 → SRAM 控制信号
    • 维护 "outstanding"（已发命令，等待响应）的状态
        │
        │  itcm_ram_cs           片选（是否激活SRAM）
        │  itcm_ram_we           写使能
        │  itcm_ram_addr[12:0]   行地址（13bit，8192行）
        │  itcm_ram_wem[7:0]     写掩码（8字节）
        │  itcm_ram_din[63:0]    写入数据（64bit）
        ▼  itcm_ram_dout[63:0]   读出数据（64bit，下拍出）
e203_srams（SRAM 顶层，汇聚所有SRAM）
        │
        ▼
e203_itcm_ram（ITCM 物理 SRAM 包装）
        │
        ▼
sirv_gnrl_ram（通用 RAM 层，选择 ASIC 或 Sim 模型）
        │
        ▼
sirv_sim_ram（仿真 RAM 实现，用 Verilog reg 数组模拟）
  reg [DW-1:0] mem_r [`E203_ITCM_RAM_DP-1:0];  // 8192 × 64bit 的大数组
```

---

## 5.8 仿真模型：sirv_sim_ram

在仿真环境中，没有真实的 SRAM 芯片，用 Verilog `reg` 数组模拟：

```verilog
module sirv_sim_ram #(
  parameter DW = 64,      // 数据宽度
  parameter AW = 13,      // 地址宽度
  parameter DP = 8192,    // 深度（行数）
  parameter MW = 8        // 写掩码宽度
) (
  input clk,
  input cs, we,
  input [AW-1:0] addr,
  input [DW-1:0] din,
  input [MW-1:0] wem,
  output reg [DW-1:0] dout  // 注意！是 reg，时序输出
);

  reg [DW-1:0] mem_r [0:DP-1];  // 整个 SRAM 的存储阵列

  always @(posedge clk) begin
    if (cs & we) begin    // 写操作
      for (i = 0; i < MW; i = i + 1)
        if (wem[i])       // 逐字节检查掩码
          mem_r[addr][i*8+7 -: 8] <= din[i*8+7 -: 8];
    end
  end

  always @(posedge clk) begin
    if (cs & ~we)         // 读操作
      dout <= mem_r[addr]; // 注意：非阻塞赋值，下一拍才输出
  end
```

---

## 5.9 容量配置速查表

修改 `config.v` 中的参数可以调整 TCM 容量：

### ITCM 容量配置

```verilog
// 选择一个，其余注释掉：
`define E203_CFG_ITCM_ADDR_WIDTH  16   // → 64KB    (8192行×64bit)
//`define E203_CFG_ITCM_ADDR_WIDTH  20 // → 1MB  (131072行×64bit)
//`define E203_CFG_ITCM_ADDR_WIDTH  21 // → 2MB  (262144行×64bit)
```

### DTCM 容量配置

```verilog
// 选择一个，其余注释掉：
`define E203_CFG_DTCM_ADDR_WIDTH  14   // → 16KB   (4096行×32bit)
`define E203_CFG_DTCM_ADDR_WIDTH  16   // → 64KB  (16384行×32bit)  ← 当前配置
//`define E203_CFG_DTCM_ADDR_WIDTH  18 // → 256KB (65536行×32bit)
//`define E203_CFG_DTCM_ADDR_WIDTH  20 // → 1MB  (262144行×32bit)
```

### 面积与容量的权衡

```
容量越大：
  ✓ 可以存放更大的程序/数据
  ✗ SRAM 面积线性增加
  ✗ 静态漏电流增加（功耗↑）
  ✗ 时序可能变差（更大的 bitcell 阵列，读访问时间↑）

嵌入式应用推荐原则：
  只配置"够用"的 TCM，不要为了"以防万用"而过度分配
```

---

## 本章小结

| 概念 | 要点 | 配置参数 |
|------|------|---------|
| TCM vs Cache | TCM 延迟固定，适合实时系统 | — |
| ITCM | 64bit 宽，1拍延迟，双访问者（IFU+LSU） | `E203_CFG_ITCM_ADDR_WIDTH` |
| DTCM | 32bit 宽，1拍延迟，仅 LSU 访问 | `E203_CFG_DTCM_ADDR_WIDTH` |
| SRAM 接口 | cs+we+addr+din/dout+wem | `itcm_ram_*`, `dtcm_ram_*` |
| 写掩码 | 字节级控制（wem），ITCM 8bit，DTCM 4bit | — |
| Hold-up | SRAM 数据保持，可节省重复访问 | `ifu2itcm_holdup` |
| 时钟门控 | SRAM 不用时关时钟，省功耗 | `clk_itcm_ram/clk_dtcm_ram` |
| ECC | 可选，单比特纠错+双比特检测 | `E203_CFG_HAS_ECC` |

> **下一章**：时钟、复位与功耗——e203 如何在系统级管理电源，AON 域是怎么工作的，以及整个处理器的功耗设计理念。
