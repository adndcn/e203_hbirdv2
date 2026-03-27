# 第10章 EXU 代码精读

> 本章对 e203 执行单元（EXU）核心源文件进行逐行精读，重点解析每条关键 `assign`/`always` 背后的设计意图。阅读本章前建议先完成第8章（EXU 架构设计）的理论学习。

---

## 第0节 跨模块信号溯源

EXU 既是信号的消费者，也是信号的生产者。读代码前，先把所有"从外面进来的线"和"往外面送出去的线"理清楚。

### 0.1 EXU 接收的跨模块信号

| 信号名 | 来源模块 | 含义 |
|---|---|---|
| `ifu_o_valid` / `ifu_o_ir` / `ifu_o_pc` | `e203_ifu_ifetch` | IFU 向 EXU 送出指令，握手 valid 端 |
| `ifu_o_misalgn` / `ifu_o_buserr` | `e203_ifu_ifetch` | IFU 取指异常标记，随指令一起带入 EXU |
| `dbg_irq_r` / `ext_irq_r` / `sft_irq_r` / `tmr_irq_r` | `e203_irq_sync` | 外部中断同步寄存器，至 excp 子模块 |
| `csr_epc_r` / `csr_dpc_r` / `csr_mtvec_r` | `e203_exu_csr` | CSR 中的 EPC / tvec，用于 mret/trap 跳转 |
| `lsu_o_valid` / `lsu_o_wbck_wdat` / `lsu_o_wbck_rdidx` | `e203_lsu_ctrl` | LSU 长流水写回，送入 `longpwbck.v` |
| `pipe_flush_ack` | `e203_ifu_ifetch` | IFU 确认已接受冲刷请求，握手 ready 端 |
| `amo_wait` | `e203_lsu_ctrl` | AMO 操作进行中，禁止 WFI 应答 |

### 0.2 EXU 向外发出的跨模块信号

| 信号名 | 目标模块 | 含义 |
|---|---|---|
| `pipe_flush_req` / `pipe_flush_add_op1/2` | `e203_ifu_ifetch` | 要求 IFU 以新 PC 冲刷流水线，op1/op2 送 IFU 复用其加法器 |
| `oitf_empty` | `e203_ifu_litebpu` 及 `e203_exu_disp` | OITF 空标志：BPU 用于解除 JALR 依赖；disp 用于 WFI/CSR/FENCE 条件 |
| `rf2ifu_x1` / `rf2ifu_rs1` | `e203_ifu_litebpu` | x1（ra）及 rs1 寄存器值，供 BPU 直接预测 JALR 目标 |
| `dec2ifu_rden` / `dec2ifu_rs1en` / `dec2ifu_rdidx` | `e203_ifu_ifetch` | 解码结果反馈给 IFU，用于 JALR 数据依赖检测与使能 rf→bpu 读 |
| `dec2ifu_mulhsu/mul/div/rem/divu/remu` | `e203_ifu_ifetch` | 当前指令是 MUL/DIV 类：IFU 用于判断 B2B（背靠背）融合 |
| `wfi_halt_ifu_req` | `e203_ifu_ifetch` | 要求 IFU 停止取指（WFI 等待中断） |
| `exu2ifu_ir_valid` | `e203_ifu_ifetch` | EXU IR 寄存器有效，IFU 才允许接受新指令 |

---

## 精读一：`e203_exu_decode.v` — 指令解码器

### 1.1 为什么预先提炼这么多 `opcode_4_2_XXX` 连线？

```verilog
// We generate the signals and reused them as much as possible to save gatecounts
wire opcode_4_2_000 = (opcode[4:2] == 3'b000);
wire opcode_4_2_001 = (opcode[4:2] == 3'b001);
...
wire opcode_6_5_00  = (opcode[6:5] == 2'b00);
wire rv32_func3_000 = (rv32_func3 == 3'b000);
wire rv32_func7_0000001 = (rv32_func7 == 7'b0000001);
```

注释已经点明——**为了节省门电路数量**。

具体原因：RISC-V 32 位指令的 opcode`[6:0]` 可以被分成 3 段：`[1:0]`（固定为 `11` 表示 32bit）、`[6:5]`（2 位大类）、`[4:2]`（3 位小类）。这种分段结构让"大类比较器"和"小类比较器"可以被所有指令类型共用。

如果不抽象这些中间线，每一条指令的判断（如 `rv32_load`、`rv32_store`、`rv32_jalr` 等）都要各自独立用完整 7 个 bit 做比较，会生成大量重复比较逻辑。抽象为 `opcode_4_2_XXX` 等连线后，每个比较器只实例化一次，多个下游信号直接 AND 引用，综合器就不会重复面积。

**这是 RTL 设计中最基础的"公共表达式提取"优化，效果在指令字段解码这类有大量分支的逻辑中尤其显著。**

### 1.2 RISC-V 指令的两级 opcode 结构

```verilog
wire rv32_load     = opcode_6_5_00 & opcode_4_2_000 & opcode_1_0_11;
wire rv32_store    = opcode_6_5_01 & opcode_4_2_000 & opcode_1_0_11;
wire rv32_branch   = opcode_6_5_11 & opcode_4_2_000 & opcode_1_0_11;
wire rv32_jalr     = opcode_6_5_11 & opcode_4_2_001 & opcode_1_0_11;
wire rv32_jal      = opcode_6_5_11 & opcode_4_2_011 & opcode_1_0_11;
wire rv32_system   = opcode_6_5_11 & opcode_4_2_100 & opcode_1_0_11;
```

RISC-V ISA 规范：32 位指令的 `[1:0]` 恒为 `11`（区别于 16 位 RVC），`[6:5]` 表示大类，`[4:2]` 进一步细分。三段 AND 合在一起就是完整的指令类型判断：

```verilog
wire rv32 = (~(i_instr[4:2] == 3'b111)) & opcode_1_0_11;
// opcode[4:2]=111 在 RISC-V 规范中是保留的"更长指令"编码，
// 与 opcode[1:0]=11 组合意味着指令长度 ≥48bit
// e203 不支持，所以将其标记为非 rv32 以便后续产生非法指令异常
```

### 1.3 RVC 寄存器压缩映射

```verilog
wire [4:0]  rv16_rdd   = {2'b01, rv32_instr[4:2]};
wire [4:0]  rv16_rss1  = {2'b01, rv32_instr[9:7]};
wire [4:0]  rv16_rss2  = rv16_rdd;
```

RVC（压缩指令集）中，很多指令只用 3 bit 编码寄存器字段（节省指令空间），但这 3 bit 只表示 x8\~x15 这 8 个"常用寄存器"。`{2'b01, 3'bXXX}` 的拼接就是把 3 bit 扩展到 5 bit 并自动加上偏移：`01_000` = 8、`01_001` = 9 … `01_111` = 15。一条赋值语句就完成了所有 RVC 压缩寄存器的映射，不需要任何 case/if。

> **RVC 寄存器约定**：`rv16_rdd/rss1/rss2` 是"CL/CS/CA/CB 格式"中的压缩寄存器；`rv16_rd/rs1/rs2` 是"CR/CI 格式"中直接用完整 5 bit 编码的寄存器，两套命名同时存在，解码时按具体指令格式选用。

### 1.4 非法指令检测

```verilog
wire rv16_lwsp_ilgl   = rv16_lwsp & rv16_rd_x0;        // C.LWSP: rd=x0 非法
wire rv16_sxxi_shamt_ilgl = (rv16_slli | rv16_srli | rv16_srai) & (~rv16_sxxi_shamt_legl);
// shamt[5]=1 或 shamt[4:0]=0 在 RV32C 中非法
wire rv16_li_ilgl     = rv16_li & (rv16_rd_x0);         // C.LI: rd=x0 无意义
wire rv16_lui_ilgl    = rv16_lui & (rv16_rd_x0 | rv16_rd_x2 | ...);
```

非法指令检测直接嵌入解码逻辑中，与指令识别同步完成。每个 `_ilgl` 信号在输出端合并为统一的 `dec_ilegl`，随指令一起进入 EXU 流水线，最终由 `excp` 子模块处理为 Illegal Instruction 异常。

---

## 精读二：`e203_exu_regfile.v` — 寄存器堆

### 2.1 `generate for` 循环：硬件循环 vs 软件循环

```verilog
genvar i;
generate
  for (i=0; i<`E203_RFREG_NUM; i=i+1) begin: regfile
    if (i == 0) begin: rf0
      assign rf_wen[i] = 1'b0;
      assign rf_r[i]   = `E203_XLEN'b0;
    end else begin: rfno0
      assign rf_wen[i] = wbck_dest_wen & (wbck_dest_idx == i);
      `ifdef E203_REGFILE_LATCH_BASED
      ...
      `else
      sirv_gnrl_dffl #(`E203_XLEN) rf_dffl (rf_wen[i], wbck_dest_dat, rf_r[i], clk);
      `endif
    end
  end
endgenerate
```

`generate for` 是 Verilog 的 **elaboration 时展开**，不是运行时循环。综合后这里生成的是 31 组独立的寄存器电路（x1\~x31），而不是一个时序循环。初学者常见误解是把它当成软件 for 循环，实际上每轮迭代都在描述一块独立硬件。

`genvar i` 在 Verilog 中只在 `generate for` 内部有效，不占任何时序资源。

### 2.2 x0 硬连接为零

```verilog
if (i == 0) begin: rf0
  assign rf_wen[i] = 1'b0;   // 写使能永远禁止
  assign rf_r[i]   = `E203_XLEN'b0;  // 读出值永远是 0
end
```

RISC-V ISA 规定 x0 是零寄存器，硬件写入无效，读出恒为 0。这里直接用 `assign` 常量实现，综合器不会为 x0 生成任何存储单元（触发器/锁存器），自动优化掉。

### 2.3 DFF 模式 vs Latch 模式

```verilog
`ifdef E203_REGFILE_LATCH_BASED //{
  // 使用 clock-gated latch（时钟门控锁存器）
  // 好处：功耗更低（比 DFF 少约 40%，因为 latch 体积更小）
  // 坏处：时序分析更复杂，不建议在 FPGA 上使用
  sirv_gnrl_ltch #(`E203_XLEN) rf_ltch (rf_wen[i] ? clk : 1'b0, wbck_dest_dat, rf_r[i]);
`else //}{
  // 使用普通 DFF，FPGA/ASIC 通用
  sirv_gnrl_dffl #(`E203_XLEN) rf_dffl (rf_wen[i], wbck_dest_dat, rf_r[i], clk);
`endif //}
```

e203 的电源优化选项：在实际 ASIC tape-out 时，寄存器堆通常使用 latch-based 方案，可以节约面积和功耗；FPGA 验证时用 DFF 模式（FPGA 上 latch 综合效果差）。这个宏开关让同一套 RTL 适配两种场景。

### 2.4 数组下标读：自动推断多路选择器

```verilog
assign read_src1_dat = rf_r[read_src1_idx];
assign read_src2_dat = rf_r[read_src2_idx];
```

`rf_r` 是一个 32 元素的 wire 数组（`wire [XLEN-1:0] rf_r[0:31]`）。用变量下标 `rf_r[idx]` 访问时，综合器会自动推断出一个 32:1 多路选择器树（MUX tree）——不需要程序员手写 32 路 case 语句。

### 2.5 x1 单独引出给 IFU

```verilog
assign x1_r = rf_r[1];
```

x1（ra，return address）寄存器被单独引出，目的是送给 IFU 的 liteBPU 预测 JALR（`ret`/`jalr ra`）的跳转目标。详见第9章 liteBPU 精读。这条连线使 IFU 不需要完整读端口就能直接获得 ra 的值，是一个精心设计的优化路径。

---

## 精读三：`e203_exu_oitf.v` — 未完成指令跟踪 FIFO

OITF（Outstanding Instructions Track FIFO）跟踪所有还未完成写回的"长流水"指令（MUL/DIV/LSU），用于检测 RAW/WAW 数据依赖。

### 3.1 标志位+指针 FIFO——区分满与空

经典 FIFO 只用读写指针无法区分"满"还是"空"（两种状态下 read\_ptr == write\_ptr）。e203 的解法是加一对翻转标志位：

```verilog
wire alc_ptr_flg_r;   // 分配指针每次绕回（wrap）时翻转
wire ret_ptr_flg_r;   // 退出指针每次绕回时翻转

assign oitf_empty = (ret_ptr_r == alc_ptr_r) &  (ret_ptr_flg_r == alc_ptr_flg_r);
assign oitf_full  = (ret_ptr_r == alc_ptr_r) & ~(ret_ptr_flg_r == alc_ptr_flg_r);
```

规则很直观：
- **空**：两个指针相等，且两个标志也相等（从来没有错位过）
- **满**：两个指针相等，但标志不同（分配指针比退出指针多走了一整圈）

这是标准的"格雷码/翻转位"FIFO 判满判空技巧，只需要 1 个额外 bit，比存计数器更省硬件。

### 3.2 关键路径切断：`dis_ready = ~oitf_full`（源码注释原文）

```verilog
// To cut down the critical loop:
//   ALU write-back valid --> oitf_ret_ena --> oitf_ready --> dispatch_ready --> alu_i_valid
// We push the dis_ready with oitf not-full instead of (oitf_not_full | oitf_ret_ena)
assign dis_ready = (~oitf_full);
```

这是整个 OITF 设计中最微妙的一处。**理论上**最激进的设计是：  
`dis_ready = ~oitf_full | oitf_ret_ena`  
——即"OITF 满了，但本周期正好有一条退出，那本周期也可以分配"。这样可以避免 OITF 满时空耗一个周期。

但**硬件上**这会形成一个组合逻辑回路：  
`alu_wb_valid → oitf_ret_ena → dis_ready → dispatch_ready → alu_i_valid → alu_wb_valid`  
这条回路跨越了 ALU、OITF、分派三个模块，时序非常紧张。

e203 的选择是**主动牺牲极端情况下 1 个周期**，换来时序裕量。这是面积/功耗/时序三角折中中常见的"保守但正确"策略。

### 3.3 CAM 匹配：`generate for` + OR 规约

```verilog
generate
  for (i=0; i<`E203_OITF_DEPTH; i=i+1) begin: oitf_entries
    assign rd_match_rs1idx[i] = vld_r[i] & rdwen_r[i] & disp_i_rs1en
        & (rdfpu_r[i] == disp_i_rs1fpu)
        & (rdidx_r[i] == disp_i_rs1idx);
    // 同理还有 rd_match_rs2idx, rd_match_rs3idx, rd_match_rdidx
  end
endgenerate

assign oitfrd_match_disprs1 = |rd_match_rs1idx;  // 按位 OR 规约
assign oitfrd_match_disprs2 = |rd_match_rs2idx;
assign oitfrd_match_disprs3 = |rd_match_rs3idx;
assign oitfrd_match_disprd  = |rd_match_rdidx;
```

这是一个标准的 **Content-Addressable Memory（CAM）**结构：对 OITF 中每一项（最多 `E203_OITF_DEPTH` 项，默认为 2）并行检查是否匹配新指令的源寄存器，最后 OR 规约得到"存在依赖"标志。

每条匹配需要满足 4 个条件：
1. `vld_r[i]`：该项有效（OITF 中确实有一条未完成的指令）
2. `rdwen_r[i]`：该项写目标寄存器（没有目标寄存器的指令不构成 RAW）
3. `disp_i_rs1en`：新指令确实读 rs1（没有源寄存器的指令不会产生依赖）
4. 寄存器编号和 FPU 标记匹配

`|rd_match_rs1idx` 是 Verilog 的"一元 OR 规约"运算符，等价于 `rd_match_rs1idx[0] | rd_match_rs1idx[1] | ...`，可以直接对 vector 使用。

---

## 精读四：`e203_exu_disp.v` — 指令分派

分派（dispatch）模块连接"IFU 送来的指令"和"各执行单元的输入"，是流水线的控制中枢。

### 4.1 分派条件：5 个独立约束的 AND

```verilog
wire disp_condition =
      (disp_csr         ? oitf_empty : 1'b1)           // ① CSR 指令：等 OITF 排空
    & (disp_fence_fencei? oitf_empty : 1'b1)           // ② FENCE：等 OITF 排空
    & (~wfi_halt_exu_req)                               // ③ WFI 暂停请求未激活
    & (~dep)                                            // ④ 无 RAW/WAW 依赖
    & (disp_alu_longp_prdt ? disp_oitf_ready : 1'b1);  // ⑤ 长流水：OITF 未满
```

逐条解析：

**① CSR 指令必须等 OITF 排空**：CSR 可能与 MUL/DIV/LSU 的结果相关（如 `mstatus`、`mtval` 等），而且 CSR 修改具有全局副作用，必须在所有长流水指令完成后才能执行，以保证指令顺序语义。

**② FENCE/FENCE.I 必须等 OITF 排空**：FENCE 要确保之前所有 store 都已经到达内存，FENCE.I 要确保指令缓存刷新，两者都要求 LSU 没有在途操作。

**③ WFI 暂停请求**：`wfi_halt_exu_req` 由 excp 子模块发出，要求 EXU 停止分派新指令（等待中断唤醒）。

**④ 无数据依赖**：RAW（Read-After-Write）或 WAW（Write-After-Write）依赖，由 OITF CAM 检测。

**⑤ OITF 未满**：仅当新指令被预测为长流水时才需要检查，使用**预测值** `disp_alu_longp_prdt`（见 4.3 节）。

### 4.2 x0 寄存器掩码：AND 替代 2:1 MUX

```verilog
// rs1 & {32{~rs1x0}} 等价于：
//   if (rs1x0) return 0; else return rs1;
// 但用 AND 门实现，没有 2:1 MUX 的面积开销
wire [`E203_XLEN-1:0] disp_i_rs1_msked = disp_i_rs1 & {`E203_XLEN{~disp_i_rs1x0}};
wire [`E203_XLEN-1:0] disp_i_rs2_msked = disp_i_rs2 & {`E203_XLEN{~disp_i_rs2x0}};
```

当 rs1 = x0 时，寄存器值应为 0（RISC-V 零寄存器）。常规写法是：
```verilog
assign val = rs1x0 ? 0 : rs1;  // 需要 2:1 MUX
```
e203 改用 `{XLEN{~rs1x0}}` 扩展为全 0 或全 1 的掩码，再与 rs1 做 AND。综合结果完全等价，但通常比 MUX 用更少的门——对 32 位数据，少用了 32 个 MUX 单元。

这是嵌入式处理器设计中的高频技巧：**用 AND-mask 替代 2:1 MUX，适用于"条件为真时输出等于某一输入"的场景**。

### 4.3 `longp_prdt`（预测）vs `longp_real`（精确）

```verilog
// disp_condition 里用预测值检查 OITF 容量：
//   目的是提前截断逻辑，避免 longp_real 产生后再判断导致的时序问题
wire disp_condition = ... & (disp_alu_longp_prdt ? disp_oitf_ready : 1'b1);

// OITF 实际分配用精确值：
//   只有确实被发射为长流水的指令才占用 OITF 槽位
// "only when it is really dispatched as long pipe then allocate the OITF"
assign disp_oitf_ena = disp_o_alu_valid & disp_o_alu_ready & disp_alu_longp_real;
```

**预测值**（`longp_prdt`）：在解码的早期阶段，根据 opcode 猜测一条指令是否会走长流水。预测可能有误差（例如某些特殊 MUL 情况），但可以提前在 `disp_condition` 中使用，**切断组合路径**。

**精确值**（`longp_real`）：在确认指令真正走长流水后才有效，用于实际分配 OITF 槽位。正确性优先，即使晚一点得到结果也没问题（OITF 分配是在握手完成后的下一拍）。

这是 e203 中多处出现的"时序优化 vs 功能正确"分层模式：**预测值管时序，精确值管状态**。

### 4.4 WFI 应答条件

```verilog
assign wfi_halt_exu_ack = oitf_empty & (~amo_wait);
```

WFI（Wait For Interrupt）指令要求：
1. `oitf_empty`：所有长流水指令已经写回（没有悬空的 MUL/DIV/LSU）
2. `~amo_wait`：AMO 操作（原子内存操作）不在进行中

这两个条件都满足后，`excp` 子模块才会正式进入 WFI 等待状态，IFU 停止取指。

### 4.5 WAW 依赖：为什么所有指令都要检查？

```verilog
// WAW (Write-After-Write) dependency:
// The new instruction has the same write target as an outstanding instruction
assign waw_dep = oitfrd_match_disprd;
```

源码注释中提到：原始设计曾考虑只对近距离 MUL 指令检查 WAW，但 benchmark 测试显示对所有指令检查 WAW 并没有明显性能下降（因为真实程序中 WAW 非常罕见），而且简化了逻辑。

实际上 RISC-V 顺序单发射处理器中 WAW 依赖极少发生（需要 rd 相同且前一条还没写回），加上 OITF 深度很小（默认 2），这个代价几乎可以忽略。

---

## 精读五：`e203_exu_alu_dpath.v` — ALU 共享数据通路

ALU 数据通路是 e203 EXU 面积最大的单个模块，核心思想是**一套硬件，多种运算复用**。

### 5.1 所有运算单元共享同一个加法器

```verilog
// ALU（ADD/SUB）、BJP（branch 地址计算）、AGU（load/store 地址计算）、
// MULDIV 都使用同一个 adder
wire [`E203_XLEN:0] adder_res = adder_in1 + adder_in2 + adder_cin;
```

这个加法器是 33 位（比 XLEN 多 1 位），多出的最高位用于捕获进位，支持：
- ADD/ADDI：直接相加
- SUB：`adder_in2 = ~op2`，`adder_cin = 1`（补码取反加一）
- 比较/分支（BEQ/BNE/BLT 等）：通过减法的进位/符号位判断大小
- 地址计算（AUIPC、load/store）：PC + imm 或 rs1 + imm
- MUL/DIV 部分步骤：MULDIV 状态机借用此加法器完成迭代

只凑一个加法器而不是各模块独自带，可以节约大约 4 个 32 位加法器的面积。

### 5.2 右移通过左移实现：只用一个移位器

```verilog
// 右移技巧：
// 1. 将输入 op1 按位翻转（reverse bits）
// 2. 执行左移（SLL）
// 3. 将结果再次翻转，得到右移效果
wire [31:0] shifter_op1_rev = {
  op1[0],  op1[1],  op1[2],  op1[3],  op1[4],  op1[5],  op1[6],  op1[7],
  op1[8],  op1[9],  op1[10], op1[11], op1[12], op1[13], op1[14], op1[15],
  op1[16], op1[17], op1[18], op1[19], op1[20], op1[21], op1[22], op1[23],
  op1[24], op1[25], op1[26], op1[27], op1[28], op1[29], op1[30], op1[31]
};
// 左移后再翻转 = 等效右移
wire [31:0] srl_res_rev = ... ; // 左移结果翻转

// SRA（算术右移）符号位扩展：
wire [31:0] sra_res = (srl_res & eff_mask) | ({32{shifter_op1[31]}} & (~eff_mask));
// eff_mask 是有效结果位的掩码，~eff_mask 是需要填充符号位的部分
```

一个完整移位器（barrel shifter）的面积约等于一个 16 到 32 位乘法器。通过"翻转→左移→翻转"的技巧，e203 只需要一个**左移器**就实现了 SLL/SRL/SRA 三种移位，面积减半。

SRA 的实现更精妙：`eff_mask` 标记移位后结果的有效位，其余高位用 `op1[31]`（符号位）填充。AND-mask 技巧再次出现：`{32{sign_bit}} & ~eff_mask`——符号位广播到所有高位，然后只保留无效位区域。

### 5.3 门控截断技术节省动态功耗

```verilog
// 当不执行加减法时，将 adder_in1 归零，避免无用的翻转传播
wire [`E203_XLEN-1:0] adder_in1 = {`E203_XLEN{adder_addsub}} & adder_op1;
```

`adder_addsub` 是"本周期是否执行加减法"的使能信号。当它为 0 时，`adder_in1` 强制为全 0，即使 `adder_op1` 有数据改变也不会传播到加法器内部。

这种技术叫 **operand isolation**（操作数隔离），目的是减少 CMOS 门的翻转（switching activity），降低动态功耗。在低功耗 SoC 设计中非常普遍，e203 全文多处都有类似的 `{W{enable}} & data` 模式。

### 5.4 `sbf_0/sbf_1` 暂存寄存器

```verilog
// 33-bit scratch buffer，供 MULDIV 状态机用于跨周期累加
wire sbf_0_ena;
wire [`E203_XLEN:0] sbf_0_nxt;
wire [`E203_XLEN:0] sbf_0_r;
sirv_gnrl_dffl #(`E203_XLEN+1) sbf_0_dffl (sbf_0_ena, sbf_0_nxt, sbf_0_r, clk);
```

MUL/DIV 是多周期操作，需要保存中间结果。`sbf_0/sbf_1` 是两个 33 位 DFF（比 XLEN 多 1 位，用于符号扩展），被 MULDIV 状态机在每个迭代周期更新。它们属于数据通路，由使能信号 `sbf_0_ena` 控制（仅 MULDIV 进行中才写入）。

---

## 精读六：`e203_exu_wbck.v` — 写回仲裁

```verilog
// ALU（短流水）和长流水仲裁写回寄存器堆
wire wbck_ready4alu   = (~longp_wbck_i_valid); // 长流水正在写回时，ALU 必须等待
wire wbck_ready4longp = 1'b1;                  // 长流水永远优先，无条件立即写回
wire rf_wbck_o_ready  = 1'b1;                  // 寄存器堆只有 1 个写端口，永远就绪
```

### 6.1 为什么长流水（longp）永远优先？

寄存器堆只有 **1 个写端口**。当长流水（MUL/DIV/LSU）完成时，结果必须被写回，否则后续等待该结果的指令永远无法继续（被 OITF 的 RAW 依赖检测阻塞）。如果 ALU 在这时抢占写端口，就会导致整个流水线死锁。

ALU 的情况不同：ALU 是单周期的，它的结果"等一拍"完全可以接受——下一拍长流水写完再写入 ALU 结果即可。

### 6.2 `rf_wbck_o_ready = 1'b1` 的含义

寄存器堆的写端口没有任何背压（back-pressure），只要有写请求（`ena`），本拍就写入。这是因为 e203 的寄存器堆（DFF 数组）的写操作是纯组合逻辑写使能——不存在"忙碌"状态。`1'b1` 表示"寄存器堆永远准备好接受写入"。

---

## 精读七：`e203_exu_commit.v` — 提交与冲刷控制

### 7.1 两个子模块：branchslv 与 excp 分工明确

```verilog
e203_exu_branchslv u_e203_exu_branchslv(...);  // 处理分支预测错误冲刷
e203_exu_excp      u_e203_exu_excp     (...);  // 处理异常/中断冲刷
```

**`e203_exu_branchslv`（Branch Solver）**：负责检测分支预测失误（`bjp_prdt ≠ bjp_rslv`）、FENCE.I 操作、mret/dret 返回，计算正确的跳转目标并发出冲刷请求。

**`e203_exu_excp`（Exception）**：负责处理所有异常（非法指令、调用、断点、访存对齐错误等）和中断（外部/软件/时钟中断），将异常原因写入 CSR（mepc/mcause/mtval），并发出冲刷到 mtvec 的请求。

两者通过独立的冲刷请求互相隔离，在 commit 顶层进行仲裁：

```verilog
assign pipe_flush_req = excpirq_flush_req | alu_brchmis_flush_req;
assign pipe_flush_add_op1 = excpirq_flush_req ? excpirq_flush_add_op1 : alu_brchmis_flush_add_op1;
assign pipe_flush_add_op2 = excpirq_flush_req ? excpirq_flush_add_op2 : alu_brchmis_flush_add_op2;
```

**excpirq 优先于 branchslv**：当异常/中断和分支错误在同一周期同时发生时，异常处理优先（因为异常的语义比分支更紧迫，且 branchslv 的目标 PC 通常对异常处理来说意义不大）。

### 7.2 冲刷 PC 复用 IFU 加法器——节省一个加法器

```verilog
// 模块注释原文：
// "To save the gatecount, when we need to flush pipeline with new PC,
//  we want to reuse the adder in IFU, so we will not pass flush-PC
//  to IFU, instead, we pass the flush-pc-adder-op1/op2 to IFU
//  and IFU will just use its adder to caculate the flush-pc-adder-result"
output [`E203_PC_SIZE-1:0] pipe_flush_add_op1,
output [`E203_PC_SIZE-1:0] pipe_flush_add_op2,
```

冲刷目标 PC 通常不是一个存好的常数，而是一个计算结果（如 `PC + imm`、`mtvec`、`epc`）。如果 EXU commit 直接计算出完整 PC 再传给 IFU，就需要在 EXU 这里放一个专用加法器。

e203 改为传两个加法器操作数（op1/op2），让 IFU 的**已有加法器**完成计算，EXU 近乎免费地节约了一个完整加法器的面积。这个技巧在 e203 中的多处注释中都提到了"save gatecount"，是一个系统性的面积优化策略。

### 7.3 `nonflush_cmt_ena` 驱动 instret 计数器

```verilog
assign cmt_ena           = alu_cmt_i_valid & alu_cmt_i_ready;
assign cmt_instret_ena   = cmt_ena & (~alu_brchmis_flush_req);
assign nonflush_cmt_ena  = cmt_ena & (~pipe_flush_req);
```

`nonflush_cmt_ena` 是"真正完成提交的使能"：当且仅当一条指令被 EXU 处理 **且不触发任何冲刷**（既没有分支错误也没有异常）时，才算一条指令真正提交。

这个信号被送到 CSR 模块，用于递增 `minstret`（已提交指令计数器，RISC-V 规范中的性能计数器）。被冲刷掉的指令（预测错误分支、回滚）不应该计入 instret——因为从程序语义角度看，它们从未被执行。

`cmt_instret_ena`（不包含 excpirq 冲刷）和 `nonflush_cmt_ena`（不包含所有冲刷）在某些计数/统计场景下含义不同，分别单独引出。

---

## 精读小结

### EXU 各子模块功能一览

| 模块 | 核心功能 | 关键设计点 |
|---|---|---|
| `e203_exu_decode.v` | 指令解码，生成控制信息 | 预提炼比较器（opcode_4_2_XXX）节省面积；RVC 寄存器压缩映射 |
| `e203_exu_regfile.v` | 32 个整数寄存器 | generate for = 31 个独立 DFF；x0 硬连零；x1_r 专线给 BPU |
| `e203_exu_oitf.v` | 长流水指令追踪 FIFO | 标志位区分满/空；主动切断关键路径（`dis_ready=~oitf_full`）；CAM 并行匹配 |
| `e203_exu_disp.v` | 指令分派控制 | 5 约束 AND 分派条件；x0 AND-mask；longp_prdt/real 分层；WFI 双条件应答 |
| `e203_exu_alu_dpath.v` | 共享数据通路 | 单加法器复用；翻转实现右移；操作数门控省功耗；sbf 暂存器 |
| `e203_exu_wbck.v` | 写回仲裁 | 长流水永远优先；rf 写端口永远就绪 |
| `e203_exu_commit.v` | 提交与冲刷 | branchslv/excp 分工；excpirq 优先；冲刷 PC 复用 IFU 加法器；nonflush_cmt_ena→instret |

### 贯穿 EXU 的三大优化思想

| 优化目标 | 具体技术 | 出现位置 |
|---|---|---|
| **减少面积** | 共享加法器；共享移位器；复用 IFU 加法器计算冲刷 PC；预提炼 opcode 比较器 | dpath, commit, decode |
| **降低功耗** | 操作数隔离（`{W{en}} & data`）；latch-based regfile 选项；clock-gated DFF | dpath, regfile |
| **改善时序** | 主动切断 OITF 关键路径（`dis_ready=~oitf_full`）；longp_prdt 提前判断 | oitf, disp |

### 跨模块信号最终流向

```
regfile.x1_r ─────────────────────────────────────→ ifu.litebpu (JALR 预测)
regfile.x1_r ─────────────────────────────────────→ ifu.litebpu (rf2ifu_x1)
regfile.read_src1_dat ────────────────────────────→ ifu.litebpu (rf2ifu_rs1,JALR xN)
oitf.oitf_empty ──────────────────────────────────→ ifu.litebpu (解除 JALR 依赖)
oitf.oitf_empty ──────────────────────────────────→ disp (WFI/CSR/FENCE 条件)
decode.dec2ifu_* ─────────────────────────────────→ ifu.ifetch (MUL/DIV B2B 检测)
decode.dec2ifu_rdidx ─────────────────────────────→ ifu.ifetch (JALR CAM 检测)
commit.pipe_flush_req/op1/op2 ────────────────────→ ifu.ifetch (冲刷目标 PC 计算)
excp.wfi_halt_ifu_req ────────────────────────────→ ifu.ifetch (停止取指)
```
