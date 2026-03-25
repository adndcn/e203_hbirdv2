# 第4章：访存单元（LSU）

> **本章目标**：理解 CPU 如何"读写内存"。Load 指令从哪条路到达 DTCM？Store 指令如何把数据写进去？字节对齐是怎么处理的？以一条 `lw x5, 0(x1)` 指令为例，把整个访存链路走一遍。

---

## 4.1 LSU 的职责与位置

### LSU 处在流水线的哪里？

LSU（Load/Store Unit，访存单元）是 EXU 的下游，专门负责执行 Load（读内存）和 Store（写内存）操作：

```
       IFU           EXU                          LSU
  ┌──────────┐  ┌───────────────────┐  ┌───────────────────────┐
  │  取指    │  │  decode           │  │                       │
  │  PC管理  │→ │  regfile 读       │  │  地址路由             │
  │  BPU预测 │  │  disp/OITF 检测   │  │  ┌→ DTCM 路径        │
  └──────────┘  │  ALU 计算         │→ │  └→ 系统总线路径      │
                │    ▼  (AGU算地址) │  │  数据对齐/扩展        │
                │  agu_icb_cmd_*   ─┼──┤  写回 lsu_o_*         │
                └───────────────────┘  └───────────────────────┘
```

LSU 的**输入**来自 EXU 中的 AGU（Address Generation Unit，内嵌在 `alu_lsuagu` 里）：
- 访存地址（`agu_icb_cmd_addr`）
- 是读还是写（`agu_icb_cmd_read`）
- 写数据（`agu_icb_cmd_wdata`）
- 访问宽度（`agu_icb_cmd_size`：0=1字节，1=2字节，2=4字节）

LSU 的**输出**回到 EXU 的长指令写回通道：
- 读到的数据（`lsu_o_wbck_wdat`）
- 用来查找 OITF 对应条目的标签（`lsu_o_wbck_itag`）
- 是否出错（`lsu_o_wbck_err`）

---

## 4.2 地址路由：去 DTCM 还是系统总线？

### 路由逻辑

当 AGU 计算出访存地址后，LSU 控制器（`e203_lsu_ctrl`）比较地址高位，判断目标：

```verilog
// 地址的高位匹配 ITCM 区域？（ITCM 也可以被 LSU 读写，用于自修改代码）
wire arbt_icb_cmd_itcm = 
  (arbt_icb_cmd_addr[`E203_ITCM_BASE_REGION] 
      == itcm_region_indic[`E203_ITCM_BASE_REGION]);

// 地址的高位匹配 DTCM 区域？
wire arbt_icb_cmd_dtcm = 
  (arbt_icb_cmd_addr[`E203_DTCM_BASE_REGION] 
      == dtcm_region_indic[`E203_DTCM_BASE_REGION]);

// 既不是 ITCM 也不是 DTCM，就走系统总线（BIU → 外设/系统内存）
wire arbt_icb_cmd_biu  = (~arbt_icb_cmd_itcm) & (~arbt_icb_cmd_dtcm);
```

路由图：

```
AGU 发出地址 0x???????
         │
         ▼
  ┌──────────────────────────────────────┐
  │  地址路由（arbt 模块）               │
  │  addr[31:16] == 0x8000 → ITCM        │
  │  addr[31:16] == 0x9000 → DTCM        │
  │  其他                  → BIU（系统）  │
  └──────────────────────────────────────┘
         │              │             │
         ▼              ▼             ▼
      ITCM           DTCM          BIU→外设
   (lsu2itcm_*)  (lsu2dtcm_*)   (lsu2biu_*)
```

### 为什么 LSU 也可以访问 ITCM？

ITCM 本来是给 IFU 取指用的，但数据加载也可以访问它。用途场景：
- **自修改代码**：程序在运行时修改自身代码（嵌入式场景中偶尔需要）
- **读取存在 ITCM 里的常量表**（read-only data section）

---

## 4.3 DTCM 访问路径详解

DTCM（Data Tightly Coupled Memory，数据紧耦合存储器）是最常用的数据存储目标。

### 完整信号链

```
EXU (ALU-AGU) 计算出地址和数据
        │
        │  agu_icb_cmd_valid/ready/addr/wdata/read/size/wmask
        ▼
  e203_lsu_ctrl (LSU 控制器)
        │
        │  dtcm_icb_cmd_valid/ready/addr[15:0]/wdata/read/wmask
        ▼
  e203_dtcm_ctrl (DTCM 控制器)
        │
        │  SRAM 读写信号（片选/地址/数据/写掩码）
        ▼
  e203_dtcm_ram → sirv_gnrl_ram → 物理 SRAM
        │
        │  rdata（1拍后返回）
        ▼
  e203_dtcm_ctrl → dtcm_icb_rsp_rdata
        │
        ▼
  e203_lsu_ctrl：数据对齐处理
        │
        │  lsu_o_wbck_wdat（处理后的结果）
        ▼
  EXU longp_wbck：写回目标寄存器
```

### DTCM 的地址宽度

DTCM 配置为 64KB（`E203_CFG_DTCM_ADDR_WIDTH = 16`），所以：
- CPU 发出的完整 32 位地址：0x9000_0000 ~ 0x9000_FFFF
- 送给 DTCM SRAM 的地址：只取低 16 位（省掉基址部分）

```verilog
// DTCM SRAM只需要区内偏移地址
dtcm_icb_cmd_addr = 地址[`E203_DTCM_ADDR_WIDTH-1:0]
                  = addr[15:0]   // DTCM内的偏移（0~64KB）
```

---

## 4.4 数据对齐：按宽度访问 32bit DTCM

### 什么是"非对齐访问"？

DTCM 每次返回 32bit（4字节）数据。但程序可能只需要读 1 个字节或 2 个字节，且地址可能不是 4 字节对齐的：

```
DTCM 某个 32bit 字（地址 0x9000_0004）：
  字节位置:  [addr+3]  [addr+2]  [addr+1]  [addr+0]
  bit位置:   [31:24]   [23:16]   [15:8]    [7:0]
  示例数据:  0xDE       0xAD      0xBE      0xEF

  LW  0x9000_0004 → 读完整32bit → 0xDEADBEEF              ✓ 4字节对齐
  LH  0x9000_0004 → 读低16bit  → 0xBEEF（需要零扩展/符号扩展）
  LH  0x9000_0006 → 读高16bit  → 0xDEAD
  LB  0x9000_0005 → 读第[15:8] → 0xBE（需要符号扩展为32bit）
  LBU 0x9000_0005 → 读第[15:8] → 0x000000BE（零扩展，无符号）
```

### 写掩码（wmask）机制

对于 Store 操作，wmask 控制写哪几个字节：

```
SW  (写4字节)：wmask = 4'b1111   → 所有字节都写
SH  写到偏移0 ：wmask = 4'b0011   → 只写低2字节
SH  写到偏移2 ：wmask = 4'b1100   → 只写高2字节
SB  写到偏移0 ：wmask = 4'b0001   → 只写第0字节
SB  写到偏移1 ：wmask = 4'b0010   → 只写第1字节
SB  写到偏移2 ：wmask = 4'b0100   → 只写第2字节
SB  写到偏移3 ：wmask = 4'b1000   → 只写第3字节
```

wmask 的计算依赖访问地址的低 2 位（`addr[1:0]`）和访问宽度（`size`）：

```verilog
// AGU 中的 wmask 生成（简化版）
assign agu_icb_cmd_wmask =
    ({4{size == 2'b00}} & (4'b0001 << addr[1:0]))  // 字节访问：移位选择字节
  | ({4{size == 2'b01}} & (4'b0011 << {addr[1],1'b0}))  // 半字访问：2bit对齐
  | ({4{size == 2'b10}} & 4'b1111);                // 字访问：全写
```

### 读数据的选择与符号扩展

Load 返回的是 32bit SRAM 数据，但程序可能只需要其中的 8bit 或 16bit，还可能需要符号扩展：

```
LB  src_data[7:0] →  符号扩展到32bit  → {{24{data[7]}},  data[7:0]}
LBU src_data[7:0] →  零扩展到32bit    → {24'b0,          data[7:0]}
LH  src_data[15:0]→  符号扩展到32bit  → {{16{data[15]}}, data[15:0]}
LHU src_data[15:0]→  零扩展到32bit    → {16'b0,          data[15:0]}
LW  全部32bit     →  直接使用          → data[31:0]
```

`usign`（无符号标志）信号控制做零扩展还是符号扩展，随请求打包在 ICB 的 `usr` 字段里，响应回来时再取出使用。

---

## 4.5 用户识别标签（itag）：怎么把响应和 OITF 对应起来

LSU 是**异步**的：发出请求和收到响应可能不在同一拍。需要一种机制把"某次响应"和"对应的 OITF 条目"关联起来。

解决方案：用 `itag`（Instruction Tag，指令标签）：

```
发请求时：
  打包 agu_icb_cmd_itag = OITF 里分配的指针（0 或 1）
  这个 itag 随着 ICB 命令被记录在 LSU 内部的 FIFO 里

收响应时：
  从 FIFO 中取出对应的 itag
  lsu_o_wbck_itag = itag（哪个 OITF 条目可以退出了）

EXU longp_wbck：
  用 itag 找到 OITF 对应条目的寄存器索引（rdidx）
  把数据写入该寄存器然后退出 OITF
```

---

## 4.6 NICE 协处理器接口（扩展访存）

e203 支持一个可选的 NICE（Nuclei Instruction Co-design Extension）协处理器接口。NICE 协处理器可以自定义指令，并通过 LSU 通道访问内存：

```
NICE 协处理器                LSU 控制器
    │                           │
    │  nice_icb_cmd_*    ───►   │  （与 AGU 仲裁，NICE 优先级更高）
    │  nice_icb_rsp_*    ◄───   │
    │                           │
```

仲裁策略（代码注释说明）：NICE 优先级高于 AGU（CPU 自身的 Load/Store）。这是因为 NICE 协处理器可能对时序有更严格需求。

---

## 4.7 信号追踪：`lw x5, 0(x1)` 的完整路径

**假设**：`x1 = 0x90000010`（指向 DTCM 内地址 0x90000010）

```
──────── Step 1: EXU AGU 计算地址 ─────────────────────────────
指令: lw x5, 0(x1)
rs1_val = x1 = 0x90000010
imm     = 0
地址    = 0x90000010 + 0 = 0x90000010

agu_icb_cmd_valid = 1
agu_icb_cmd_addr  = 32'h90000010
agu_icb_cmd_read  = 1              （是 Load，读操作）
agu_icb_cmd_size  = 2'b10          （word = 4字节）
agu_icb_cmd_wmask = 4'b1111        （读操作，wmask 无效，但传 all-1）
agu_icb_cmd_usign = 0              （有符号，LW 是有符号读）
agu_icb_cmd_itag  = 1'b0           （OITF 指针 = 0）

──────── Step 2: LSU 地址路由 ─────────────────────────────────
addr[31:16] = 16'h9000 → 匹配 DTCM 基址高位
arbt_icb_cmd_dtcm = 1
arbt_icb_cmd_biu  = 0              （不走系统总线）

──────── Step 3: 发给 DTCM 控制器 ─────────────────────────────
dtcm_icb_cmd_valid = 1
dtcm_icb_cmd_addr  = 16'h0010      （只取低16位偏移）
dtcm_icb_cmd_read  = 1

──────── Step 4: DTCM SRAM 响应（1拍后）───────────────────────
dtcm_icb_rsp_valid = 1
dtcm_icb_rsp_rdata = 32'h12345678  （内存里存的数据）

──────── Step 5: 数据选择（宽度=word，不需要截取）────────────
size = 2'b10（4字节），addr[1:0] = 2'b00（4字节对齐）
结果 = 32'h12345678（直接使用，无需扩展）

──────── Step 6: LSU 向 EXU 报告完成 ──────────────────────────
lsu_o_valid      = 1
lsu_o_wbck_wdat  = 32'h12345678
lsu_o_wbck_itag  = 1'b0            （对应 OITF[0]）
lsu_o_wbck_err   = 0               （无错误）

──────── Step 7: EXU longp_wbck 写回 ──────────────────────────
OITF[0].rdidx = 5'd5               （之前登记：lw 要写 x5）
rf_wen   = 1
rf_rdidx = 5'd5
rf_wdat  = 32'h12345678

──────── Step 8: OITF 退出 ────────────────────────────────────
OITF[0].vld = 0                    （条目清空，OITF 重新变空）
ret_ena = 1
```

---

## 4.8 Store 操作的不同点

Store（写内存）和 Load 的主要区别：

1. **没有写回阶段**：Store 不需要把数据写回寄存器（rd 为 x0 或没有 rd）
2. **但 OITF 仍然记录**：等 Store 确认完成（保证写操作已达内存），才退出 OITF，保证程序有序性
3. **写掩码很重要**：必须精确控制写哪几个字节

Store 示例（`sw x6, 4(x1)`）：

```
agu_icb_cmd_addr  = x1 + 4
agu_icb_cmd_read  = 0           （写操作）
agu_icb_cmd_wdata = x6 的值    （要写入的数据）
agu_icb_cmd_wmask = 4'b1111    （SW 写全部4字节）
agu_icb_cmd_size  = 2'b10      （4字节）

DTCM 收到后：
  WEM（写使能掩码）= agu_icb_cmd_wmask = 4'b1111
  4个字节全部更新

lsu_o_wbck_err 回报：存储错误（非对齐/非法地址）
                       若无错，ret_ena → OITF 退出
```

---

## 4.9 异常处理：非法地址和非对齐

如果访存地址非法或非对齐，LSU 会向 commit 模块报告访存异常：

```verilog
lsu_o_cmt_buserr = 1       // 总线访存错误
lsu_o_cmt_badaddr = addr   // 出错的地址（存入 mtval CSR）
lsu_o_cmt_ld = 1           // 是 Load 操作
lsu_o_cmt_st = 0           // 还是 Store 操作
```

commit 模块收到 `buserr` 后：
- 将 `mcause` 设为对应错误码（Load 访问错误 = 5，Store 访问错误 = 7）
- 将 `mtval` 设为出错地址
- 跳转到 `mtvec`（异常处理程序）

---

## 本章小结

| 概念 | 关键点 | 信号 |
|------|--------|------|
| 地址路由 | 比较高位判断目标（DTCM/ITCM/BIU） | `arbt_icb_cmd_dtcm/itcm/biu` |
| 访问宽度 | size[1:0]：0=1B, 1=2B, 2=4B | `agu_icb_cmd_size` |
| 写掩码 | wmask[3:0]：每位控制1字节 | `dtcm_icb_cmd_wmask` |
| itag | 把响应和 OITF 条目对应起来 | `agu_icb_cmd_itag` |
| 符号扩展 | LB/LH 需扩展到32bit | `agu_icb_cmd_usign` |
| 长指令写回 | LSU 完成后通过 longp_wbck 路径 | `lsu_o_wbck_*` |
| NICE | 协处理器可使用同一 LSU 通道 | `nice_icb_*` |

> **下一章**：存储子系统——ITCM 和 DTCM 的物理 SRAM 如何组织，时序是什么样的，ECC 如何工作，以及容量如何通过配置参数自定义。
