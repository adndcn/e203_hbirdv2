# 第3章：执行单元（EXU）

> **本章目标**：理解 CPU 的"大脑"如何工作。从指令进入 EXU，到最终结果写入寄存器，经历译码→寄存器读取→数据冒险检测→计算→写回全过程。以一条 ADDI 指令为例，把所有模块串联起来走一遍。

---

## 3.1 EXU 的整体结构

EXU（Execution Unit，执行单元）是 e203 的核心功能模块。整个 EXU 的内部结构如下：

```
  来自 IFU 的指令（ifu_o_ir, ifu_o_pc）
            │
            ▼
  ┌─────────────────────────────────────────────────────────────────┐
  │  e203_exu.v（执行单元顶层）                                       │
  │                                                                  │
  │  ① decode         ② regfile        ③ disp + oitf               │
  │  ──────────────   ──────────────   ──────────────────────        │
  │  指令→控制信号     读寄存器          派发+数据冒险检测             │
  │       │                │                   │                    │
  │       └────────────────┘                   │                    │
  │               │                            │                    │
  │         操作数(rs1,rs2)              ④ alu（执行）               │
  │               └────────────────────►┌──────────────────────┐    │
  │                                     │  e203_exu_alu.v       │    │
  │                                     │  ├─ rglr  普通运算    │    │
  │                                     │  ├─ bjp   跳转        │    │
  │                                     │  ├─ lsuagu 地址计算  │    │
  │                                     │  ├─ csrctrl CSR访问  │    │
  │                                     │  └─ muldiv 乘除法    │    │
  │                                     └──────────────────────┘    │
  │                                              │                   │
  │  ⑤ wbck（写回仲裁）          ⑥ longpwbck（长指令写回）            │
  │  ──────────────────          ─────────────────────              │
  │  ALU结果 vs 长指令结果         LSU/乘除 执行完成后回来           │
  │       │                                  │                       │
  │       └──────────────┬───────────────────┘                       │
  │                      ▼                                           │
  │  ⑦ 寄存器堆写端口(regfile)                                        │
  │  ──────────────────────────                                      │
  │  x0~x31 之一 ← 写入结果                                           │
  │                                                                  │
  │  ⑧ commit（提交/异常）─── 跳转错误预测 ──► 清流水线，重取指        │
  │  ⑨ csr（CSR寄存器组）                                             │
  └─────────────────────────────────────────────────────────────────┘
```

---

## 3.2 decode：把 0 和 1 翻译成控制信号

### 什么是译码？

一条 32bit 指令是一串数字，CPU 需要"看懂"它是什么操作、操作哪些寄存器、立即数是多少。这就是**译码（Decode）**的工作。

以 `add x3, x1, x2` 为例：

```
指令编码: 0000000 00010 00001 000 00011 0110011
          ───┬─── ──┬── ──┬── ─┬─ ──┬──  ──┬──
           funct7  rs2   rs1  f3   rd   opcode
           (0)    (x2)  (x1) ADD (x3)   OP(R型)

decode 输出:
  dec_rv32    = 1        （32bit指令）
  dec_rs1en   = 1        （需要读 rs1）
  dec_rs2en   = 1        （需要读 rs2）
  dec_rdwen   = 1        （需要写 rd）
  dec_rs1idx  = 5'd1     （rs1 = x1）
  dec_rs2idx  = 5'd2     （rs2 = x2）
  dec_rdidx   = 5'd3     （rd = x3）
  dec_info    = ...      （打包的操作类型：ADD）
  dec_imm     = 0        （R型无立即数）
```

### dec_info：打包的操作信息总线

所有操作类型信息被打包进 `dec_info`（宽度为 `E203_DECINFO_WIDTH` 位）：

```
dec_info 内部字段（部分）：
  dec_info[DECINFO_GRP_LSB+:DECINFO_GRP_WIDTH] = 操作组（ALU/AGU/BJP/CSR/MULDIV）
  dec_info[DECINFO_ALU_ADD]    = 1 → 是加法
  dec_info[DECINFO_ALU_SUB]    = 1 → 是减法
  dec_info[DECINFO_ALU_AND]    = 1 → 是与操作
  dec_info[DECINFO_ALU_SLT]    = 1 → 是比较（小于）
  ...
```

这个"大总线"设计避免了过多的独立信号线，所有操作信息在模块间只用一根宽总线传递。

---

## 3.3 regfile：CPU 的 32 个"草稿纸格子"

### 寄存器堆的物理结构

32 个通用寄存器，每个 32bit：

```
┌────┬────────────────────────────────────┐
│ x0 │ 0000_0000_0000_0000_0000_0000_0000_0000 │ ← 永远是 0，写入被忽略
│ x1 │ ⟨返回地址⟩                        │
│ x2 │ ⟨栈指针⟩                          │
│ x3 │ ⟨上次存入的值⟩                    │
│... │ ...                                │
│x31 │ ⟨某个值⟩                          │
└────┴────────────────────────────────────┘
```

### 读端口：2个（rs1 和 rs2）

一条指令最多需要读两个源寄存器（rs1 和 rs2）：

```verilog
// 读 rs1：组合逻辑（直接读出，无延迟）
assign rf_rs1 = rf_r[rs1idx];  // 读 rs1idx 号寄存器

// 读 rs2
assign rf_rs2 = rf_r[rs2idx];  // 读 rs2idx 号寄存器
```

BPU（分支预测）还有一个专用的 x1 读端口（直接连到 `litebpu`，用于快速读 ra 寄存器）：

```verilog
assign rf2bpu_x1  = rf_r[1];   // 专门给 BPU 用的 x1 读端口
assign rf2bpu_rs1 = rf_r[bpu_rs1idx];  // 给 JALR xN 用的通用端口
```

### 写端口：1个（rd）

写回时只需要写一个目标寄存器（rd）：

```verilog
always @(posedge clk) begin
  if (rf_wen & (rf_rdidx != 5'd0))  // 不能写 x0！
    rf_r[rf_rdidx] <= rf_wdata;
end
```

---

## 3.4 disp：派发——协调调度中心

`e203_exu_disp.v`（dispatch，派发）是 EXU 的"交通警察"：

- 接收来自 decode 的指令和操作数
- 检查数据冒险（这条指令能不能立刻执行？）
- 把指令"派发"给 ALU 执行
- 通知 OITF 登记长指令

### 什么时候不能立刻派发？

```
情况1：OITF 满了
  → 有太多长指令在"飞行中"，再派可能覆盖还没写回的结果
  → disp_i_ready = 0，让 IFU/EXU 停住等待

情况2：新指令的读寄存器（rs1/rs2）和OITF中某个长指令的写寄存器（rd）冲突
  → 例：OITF 里有 lw x5, 0(x1)（正在读内存），
          现在来了 add x3, x5, x2（需要读 x5）
  → 必须等 lw 完成写回 x5 后，add 才能读到正确的值
  → disp_i_ready = 0

情况3：收到 WFI（Wait For Interrupt）停机请求
  → 需要先清空所有在途指令，再进入低功耗等待态
```

### 数据冒险检测的核心逻辑

```verilog
// OITF 模块会输出4个冲突标志：
//   oitfrd_match_disprs1 ：新指令的rs1 和 OITF中某条指令的rd 冲突
//   oitfrd_match_disprs2 ：新指令的rs2 和 OITF中某条指令的rd 冲突
//   oitfrd_match_disprd  ：新指令的rd  和 OITF中某条指令的rd 冲突（WAW）

// 有任何冲突，就不能派发
wire disp_oitf_conflict = oitfrd_match_disprs1 
                        | oitfrd_match_disprs2
                        | oitfrd_match_disprd;

// 派发准备好的条件：无冲突 AND OITF未满 AND ALU准备好接受
assign disp_i_ready = disp_o_alu_ready & (~disp_oitf_conflict) 
                    & disp_oitf_ready & (~wfi_halt_exu_req);
```

---

## 3.5 OITF：e203 的"空中交通管制"

### OITF 是什么？

OITF（Outstanding Instruction Track FIFO，未完成指令追踪 FIFO）是 e203 中最精妙的设计之一。

**问题背景**：2级流水线很简单，但有些指令（访存 Load/Store、乘除法）需要多拍才能完成。如果在等待这些"长指令"结果的同时继续取新指令，就可能出现新指令读到了旧的、还没被长指令更新的寄存器值——这叫**数据冒险（Data Hazard）**。

**OITF 的解决思路**：用一个 FIFO 记录所有"正在执行但还没写回"的长指令的信息（主要是它们要写哪个寄存器 `rd`）。每次派发新指令前，检查新指令的读寄存器是否和 OITF 里任何一条的 `rd` 冲突。

### OITF 的结构（深度=2）

```
           alc_ptr（分配指针）→ 新指令在这里入队
           ret_ptr（释放指针）→ 长指令完成从这里出队

OITF 内部：
  Entry[0]: vld=1, rdidx=x5, pc=0x80001000  ← lw x5, 0(x1) 正在执行
  Entry[1]: vld=0                             ← 空闲
                             ↑
                         最多2条并发长指令
```

### OITF 空/满的判断

```verilog
// 用分配指针（alc_ptr）和释放指针（ret_ptr）以及标志位判断：
assign oitf_empty = (ret_ptr_r == alc_ptr_r) &  (ret_ptr_flg_r == alc_ptr_flg_r);
assign oitf_full  = (ret_ptr_r == alc_ptr_r) & (~(ret_ptr_flg_r == alc_ptr_flg_r));

// 类比：像一个环形队列
//   alc == ret 且 flag 相同 → 队空
//   alc == ret 且 flag 不同 → 队满（绕了一圈回来）
```

### 为什么 dis_ready 不包含 ret_ena？

一个关键的设计决定（代码注释里也写明了）：

```verilog
// 注释原文（翻译）：
// 为了切断关键路径：
//   ALU写回valid → oitf_ret_ena → oitf_ready → dispatch_ready → alu_i_valid
// 我们把 ret_ena 排除在 dis_ready 之外

assign dis_ready = (~oitf_full);  // 只要 OITF 不满就可以派发
                                   // 不管当前拍是否有指令退出
```

**直觉解释**：假设 OITF 满了，但这一拍恰好有一条长指令执行完要退出。如果 `dis_ready` 依赖 `ret_ena`，就会形成"A 等 B、B 等 A"的组合逻辑环路，拉长了关键路径（影响最高时钟频率）。通过保守地"只要不满就派发"，绕开了这个问题。

### 冲突检测的实现

OITF 内部会对每个有效的 Entry，检查其 `rdidx` 是否和新派发指令的 `rs1idx`、`rs2idx`、`rdidx` 相同：

```verilog
for (i=0; i<`E203_OITF_DEPTH; i=i+1) begin
  assign rd_match_rs1idx[i] = vld_r[i] & rdwen_r[i] 
    & (rdidx_r[i] == disp_i_rs1idx) & disp_i_rs1en;
  assign rd_match_rs2idx[i] = vld_r[i] & rdwen_r[i] 
    & (rdidx_r[i] == disp_i_rs2idx) & disp_i_rs2en;
  assign rd_match_rdidx[i]  = vld_r[i] & rdwen_r[i] 
    & (rdidx_r[i] == disp_i_rdidx ) & disp_i_rdwen; // WAW冒险
end

// 任何一个 Entry 匹配就报冲突
assign oitfrd_match_disprs1 = |rd_match_rs1idx;
assign oitfrd_match_disprs2 = |rd_match_rs2idx;
assign oitfrd_match_disprd  = |rd_match_rdidx;
```

---

## 3.6 ALU：计算的执行者

ALU（Arithmetic Logic Unit，算术逻辑单元）内部分了 5 个子功能块：

| 子模块 | 负责的指令类型 |
|--------|--------------|
| `alu_rglr`（普通ALU） | ADD/SUB/AND/OR/XOR/SLL/SRL/SRA/SLT/LUI/AUIPC... |
| `alu_bjp`（分支跳转） | JAL/JALR/BEQ/BNE/BLT/BGE/BLTU/BGEU |
| `alu_lsuagu`（访存地址） | LW/SW/LH/SH/LB/SB/LHU/LBU（计算地址，数据访问由LSU完成） |
| `alu_csrctrl`（CSR） | CSRRW/CSRRS/CSRRC 等 CSR 访问指令 |
| `alu_muldiv`（乘除） | MUL/MULH/DIV/DIVU/REM/REMU（多拍执行，是"长指令"） |

### ALU 是"单周期"还是"多周期"？

- `rglr`, `bjp`, `lsuagu`, `csrctrl`：**单周期**，1拍出结果
- `muldiv`：**多周期**，乘法约需 2 拍，除法约需 33 拍（32位除法逐位计算）
- 访存（Load/Store）：通过 `lsuagu` 计算地址后，交给 LSU 去实际访问内存——属于**长指令**

### ALU 的输出是长指令还是短指令？

```verilog
// i_longpipe：这条指令被标记为"长指令"吗？
// 如果是长指令，结果不会立即写回，而是等 LSU 或 muldiv 完成
output i_longpipe

// 短指令结果直接从 wbck（写回）端口出来
output alu_wbck_o_valid
output [E203_XLEN-1:0]  alu_wbck_o_wdat  // 结果数据
output [E203_RFIDX_WIDTH-1:0] alu_wbck_o_rdidx  // 写哪个寄存器
```

---

## 3.7 wbck：写回仲裁器

当多个来源都可能想写回寄存器时，需要仲裁：

```
写回竞争者：
  1. ALU（短指令）   → alu_wbck_*
  2. 长指令完成      → longp_wbck_*（来自 LSU 或 muldiv）

仲裁原则：
  • 每次只能有一个来源写回（同时只有一条指令在执行）
  • 短指令（ALU）：本拍立刻写回
  • 长指令（LSU/muldiv）：等执行完成，通过 longpwbck 通道写回

最终写入口（e203_exu_regfile 的写端口）：
  rf_wen   = alu_wbck 或 longp_wbck 的有效信号
  rf_wdat  = 相应的结果数据
  rf_rdidx = 目标寄存器编号
```

---

## 3.8 commit：提交与异常处理

### 为什么需要"提交"这一步？

在 e203 这样的简单处理器中，"执行"和"提交"基本同步。但以下情况需要特殊处理：

1. **分支预测错误**：BPU 预测会跳/不跳，但 EXU 执行后发现预测错了
2. **异常（Exception）**：指令执行时遇到非法操作（除以零、非对齐访存、非法指令...）
3. **中断（Interrupt）**：外部设备发来中断请求（定时器中断、外部中断...）

### 分支预测错误的处理

```
IFU 预测：跳到 0x80001000（执行了预测的下一条指令）
EXU 执行后发现：不应该跳！应该顺序执行
                        ↓
commit 模块发出 flush 信号：
  • 清掉 IR 寄存器（丢弃错误路径上的指令）
  • 更新 PC 为正确目标地址
  • IFU 重新从正确地址取指
```

```verilog
// 分支结果核对
wire bjp_prdt = alu_cmt_i_bjp_prdt;  // BPU 的预测结果
wire bjp_rslv = alu_cmt_i_bjp_rslv;  // EXU 的执行结果

// 预测错误：需要清流水线
wire bjp_mispredict = alu_cmt_i_bjp & (bjp_prdt ^ bjp_rslv);
```

### 中断处理流程

```
外部定时器中断到来
        ↓
irq_sync 模块：对中断信号做时钟域同步（tmr_irq_r）
        ↓
commit 模块检测到 tmr_irq_r=1 且 mie/mtie 开关打开
        ↓
保存现场：
  mepc    ← 当前 PC（中断发生时的指令地址）
  mcause  ← 0x80000007（机器定时器中断原因码）
  mstatus ← 更新全局中断使能位
        ↓
PC 跳转到 mtvec（中断向量表基址）
        ↓
执行中断服务程序（ISR）
        ↓
mret 返回：PC ← mepc
```

---

## 3.9 CSR：控制状态寄存器

CSR（Control and Status Registers）是 RISC-V 中一组特殊用途的寄存器，不在通用寄存器堆里，需要用专门的 CSR 指令读写。

e203 实现的主要 CSR（Machine 级别）：

| CSR 名称 | 地址 | 功能 |
|---------|------|------|
| `mstatus` | 0x300 | 全局状态：中断使能(MIE)、特权级等 |
| `mtvec` | 0x305 | 中断/异常处理入口地址 |
| `mepc` | 0x341 | 发生异常时保存的 PC |
| `mcause` | 0x342 | 异常/中断原因码 |
| `mie` | 0x304 | 中断使能掩码（哪些中断允许响应） |
| `mip` | 0x344 | 中断挂起状态（哪些中断正在等待） |
| `mcycle` | 0xB00 | 时钟周期计数器（性能分析用） |
| `minstret` | 0xB02 | 已完成指令计数器 |
| `mscratch` | 0x340 | 临时寄存器（中断处理时保存数据用） |

访问 CSR 的指令（以 `csrrw x1, mstatus, x2` 为例）：
```
csrrw  x1, mstatus, x2
  ↕       ↕          ↕
 rd      CSR地址     rs1（新值）
原子操作：把 mstatus 的旧值写入 x1，同时把 x2 的值写入 mstatus
```

---

## 3.10 信号追踪：一条 ADDI 指令的完整生命周期

以 `addi x10, x0, 5`（等同于 `x10 = 0 + 5 = 5`）为例，追踪整个 EXU 执行过程：

### 指令编码

```
addi x10, x0, 5
  → 指令格式: I型（立即数操作)
  → 编码: 0000000_00101_00000_000_01010_0010011
              imm[11:0]  rs1   f3   rd    opcode
              (+5)       (x0)  ADDI (x10) OP-IMM
  → 十六进制: 0x00500513
```

### 各阶段信号值

```
─────────────────── Step 1: 指令进入EXU ───────────────────────
ifu_o_valid   = 1
ifu_o_ir      = 32'h00500513   // addi x10, x0, 5
ifu_o_pc      = 0x80000000

─────────────────── Step 2: decode 译码 ────────────────────────
dec_rv32      = 1              // 32bit指令
dec_rs1en     = 1              // 需要读 rs1
dec_rs2en     = 0              // 不需要读 rs2（立即数指令）
dec_rdwen     = 1              // 需要写 rd
dec_rs1idx    = 5'd0           // rs1 = x0
dec_rs2idx    = 5'd0           // rs2 未使用
dec_rdidx     = 5'd10          // rd = x10
dec_imm       = 32'd5          // 立即数 = 5
dec_info[ADD] = 1              // 操作类型：加法
dec_info[GRP] = ALU_GRP        // 操作组：普通ALU

─────────────────── Step 3: regfile 读取 ───────────────────────
rs1idx = 0 → 读 x0 = 32'h00000000（x0永远是0）

─────────────────── Step 4: disp 检测 ──────────────────────────
oitfrd_match_disprs1 = 0       // x0 不在OITF中（也没有指令会写x0）
oitfrd_match_disprd  = 0       // x10 不在OITF中
disp_oitf_conflict   = 0       // 无冲突
disp_i_ready         = 1       // 可以派发！

─────────────────── Step 5: OITF 登记（不需要！）────────────────
addi 是短指令（单周期），不需要进入 OITF
i_longpipe = 0

─────────────────── Step 6: ALU 计算 ───────────────────────────
alu_rglr 模块：
  操作数1 = rs1_val = 0           // 来自寄存器堆 x0
  操作数2 = imm     = 5           // 来自立即数
  操作: ADD
  结果 = 0 + 5 = 5 = 32'h00000005

─────────────────── Step 7: 写回 ───────────────────────────────
alu_wbck_o_valid = 1
alu_wbck_o_wdat  = 32'h00000005  // 结果 = 5
alu_wbck_o_rdidx = 5'd10         // 写入 x10

─────────────────── Step 8: regfile 写入 ───────────────────────
rf_wen   = 1
rf_rdidx = 10                    // 写 x10
rf_wdat  = 32'h00000005          // 值 = 5

─────────────────── Step 9: commit ────────────────────────────
alu_cmt_i_bjp = 0                // 不是跳转指令，无需检查预测
无异常，无中断 → 正常提交
PC 更新为 0x80000004（PC + 4）
```

---

## 3.11 一条 LOAD 指令的特殊路径（长指令）

与 ADDI 不同，`lw x5, 0(x1)`（从内存读数据到 x5）需要多拍完成：

```
─── Step 1-4: decode + regfile + disp 同前（约 1 周期）──────────

─── Step 5: OITF 登记 ──────────────────────────────────────────
lw 是长指令（需要等内存响应）
disp_oitf_ena = 1                // 向 OITF 申请一个条目
OITF Entry[0]: vld=1, rdidx=x5, pc=0x80001000

─── Step 6: ALU 计算地址 ────────────────────────────────────────
alu_lsuagu 模块：
  基址  = rs1_val         // x1 的值，比如 0x20000000
  偏移  = imm = 0
  地址  = 0x20000000 + 0 = 0x20000000

─── Step 7: 发给 LSU ────────────────────────────────────────────
i_longpipe = 1                   // 这是长指令！
cmt_o_valid → agu_i_valid = 1   // 发给 LSU
地址 = 0x20000000，操作 = LOAD，宽度 = 4字节

─── Step 8: LSU 去 DTCM 取数 ────────────────────────────────────
（切换到第4章：LSU 的工作）
lsu2dtcm_icb_cmd_valid = 1
lsu2dtcm_icb_cmd_addr  = 0x0000（DTCM内偏移）
lsu2dtcm_icb_rsp_rdata = 32'hDEADBEEF（读到的数据）

─── Step 9: 长指令写回 ──────────────────────────────────────────
longp_wbck_o_valid = 1
longp_wbck_o_wdat  = 32'hDEADBEEF
longp_wbck_o_rdidx = 5'd5         // 写入 x5
OITF Entry[0] 退出（ret_ena = 1）
```

---

## 本章小结

| 子模块 | 关键职责 | 常见信号前缀 |
|--------|---------|------------|
| `decode` | 32bit → 控制信号 + 操作数信息 | `dec_*` |
| `regfile` | 读写32个通用寄存器 | `rf_*` |
| `disp` | 派发调度，检测冲突 | `disp_*` |
| `oitf` | 追踪长指令，防止数据冒险 | `oitf_*`, `oitfrd_*` |
| `alu` | 实际计算（5个子功能块） | `alu_*` |
| `wbck` | 写回仲裁（短指令 vs 长指令） | `*_wbck_*` |
| `commit` | 提交，处理异常/中断/分支纠错 | `cmt_*`, `commit_*` |
| `csr` | CSR 寄存器组管理 | `csr_*` |

> **下一章**：访存单元（LSU）——Load 和 Store 指令如何真正地访问内存、如何处理字节对齐、以及内存访问结果如何回到 EXU 写入寄存器。
