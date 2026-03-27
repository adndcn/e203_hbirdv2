
---

---

# IFU 代码精读

> 本节逐文件精讲 IFU 各子模块的核心 RTL 实现。遇到精妙的设计会给出"为什么精妙"的说明和具体例子。建议在阅读时对照源文件一起看。

---

## 精读一：`e203_ifu_ifetch.v`

### 1.1 大量出现的"SR 触发器"模式

在 ifetch.v 里，几乎每一个状态寄存器都用同一套固定写法来"设置/清除"：

```verilog
wire foo_set = <触发set的条件>;
wire foo_clr = <触发clr的条件>;
wire foo_ena = foo_set | foo_clr;          // 有变化才更新DFF
wire foo_nxt = foo_set | (~foo_clr);       // set优先：同时set和clr时取set
sirv_gnrl_dfflr #(1) foo_dfflr(foo_ena, foo_nxt, foo_r, clk, rst_n);
```

**为什么精妙？** 这个模式有三个优点：

1. **使能门控省功耗**：`foo_ena = set | clr`，只有状态真的要变时 DFF 才翻转。若 set/clr 都为 0（状态不变），DFF 输入门被关断，不消耗动态功耗。

2. **set 优先语义自然**：`foo_nxt = set | (~clr)` 用一行等式表达了"set 优先"：
   - `set=1, clr=0` → nxt = 1 ✓
   - `set=0, clr=1` → nxt = 0 ✓  
   - `set=1, clr=1` → nxt = 1（set 优先） ✓
   - `set=0, clr=0` → nxt = 1 或 0（无所谓，ena=0 不会写入） ✓

3. **可读性强**：一眼就能找到什么条件置位、什么条件清零。

这套写法在 ifetch.v 里被反复使用 6 次，贯穿整个文件。

---

### 1.2 复位请求状态机（`reset_req_r`）

```verilog
// 第一级：捕获复位撤销事件
wire reset_flag_r;
sirv_gnrl_dffrs #(1) reset_flag_dffrs(1'b0, reset_flag_r, clk, rst_n);
//  ↑ 复位期间 rst_n=0，reset_flag_r 强制为0；
//    复位释放后第一个 clk 上升沿，reset_flag_r 从0→1（DFF默认采样到1'b0→1）
//    也就是说：reset_flag_r == 1  意味着"复位刚刚结束"

// 第二级：把这个"事件"转成"请求"，等到IFU真正发出取指才清除
wire reset_req_r;
wire reset_req_set = (~reset_req_r) & reset_flag_r;   // 复位刚结束且请求还没发
wire reset_req_clr = reset_req_r  & ifu_req_hsked;     // 请求握手成功才清除
wire reset_req_ena = reset_req_set | reset_req_clr;
wire reset_req_nxt = reset_req_set | (~reset_req_clr);
sirv_gnrl_dfflr #(1) reset_req_dfflr(reset_req_ena, reset_req_nxt, reset_req_r, clk, rst_n);
```

**为什么要两级？**

直接使用 `reset_flag_r` 当做"复位后取第一条指令"的触发，有一个问题：如果那一拍 IFU 的 req 通道刚好没准备好（`ifu_req_ready=0`），请求就会丢失，CPU 永远不会从正确地址开始取指。

`reset_req_r` 是一个"保持型"请求——它记住"我需要取复位向量这个地址"，直到 `ifu_req_hsked`（握手成功）才清除。这保证了复位后一定能从 `pc_rtvec` 开始执行。

---

### 1.3 暂停确认（`halt_ack_r`）

```verilog
// halt_ack 的条件更严格：必须等在途请求也清空
wire ifu_no_outs;   // 当前没有在途的取指（下节详解）

assign halt_ack_set = ifu_halt_req       // 有停止请求
                    & (~halt_ack_r)      // 还没确认过
                    & ifu_no_outs;       // 没有在途的ICB请求
assign halt_ack_clr = halt_ack_r & (~ifu_halt_req); // 停止请求撤销时清除
```

**为什么必须等 `ifu_no_outs`？**

如果 IFU 已经向 ITCM 发出了一个读请求，ITCM 还没响应，这时候就宣布"已停止"，那 ITCM 的响应回来时 IFU 可能已经被冻住，无法接收，会造成总线协议违规（rsp_valid 无人消费）。

必须等所有在途的请求都回来处理完，才能安全宣布停止。

---

### 1.4 `pipe_flush_ack = 1'b1`：永远立刻接受 flush 请求

```verilog
// 来自 EXU 分支误判或异常的 flush 请求
assign pipe_flush_ack = 1'b1;   // 永远立刻 ack！

// 但万一IFU那一拍无法立即处理（比如req通道忙），就用延迟锁存器记住
wire dly_flush_r;
assign dly_flush_set = pipe_flush_req & (~ifu_req_hsked);  // flush来了但req没握手
assign dly_flush_clr = dly_flush_r & ifu_req_hsked;        // 延迟的flush发出去之后清除
```

**精妙分析：**

正常的握手逻辑是"能处理就 ack，不能处理就拉低 ready"。但这里反其道而行，永远立刻 ack，再用一个寄存器 `dly_flush_r` 记住"虽然ack了但还没真正处理"。

**为什么这样设计？** EXU 发出 `pipe_flush_req` 时，EXU 和 IFU 之间形成一条组合逻辑路径：

```
EXU 计算跳转目标
   → pipe_flush_req 拉高
   → IFU 看到 req，判断能否处理，生成 ack
   → ack 回到 EXU ...（可能影响 EXU 的下拍逻辑）
```

如果 ack 是组合逻辑（非寄存器），这条路径很长，时序紧张。`pipe_flush_ack = 1'b1` 把这条路径从关键路径上拿掉，用 `dly_flush_r` 推迟一拍处理，换来了时序余量。

---

### 1.5 IR 高低 16bit 分开控制（功耗优化）

```verilog
// 低 16bit 每次有新指令就更新
wire ir_lo_ena = ir_valid_set;

// 高 16bit 只有确认是 32bit 指令时才更新
wire ir_hi_ena = ir_valid_set & minidec_rv32;

sirv_gnrl_dfflr #(16) ifu_hi_ir_dfflr(ir_hi_ena, ifu_ir_nxt[31:16], ifu_ir_r[31:16], clk, rst_n);
sirv_gnrl_dfflr #(16) ifu_lo_ir_dfflr(ir_lo_ena, ifu_ir_nxt[15: 0], ifu_ir_r[15: 0], clk, rst_n);
```

**功耗计算举例：**

假设 50% 的指令是 16bit 压缩指令（RVC），如果不优化：
- 每拍 IR 寄存器翻转：32bit × α （翻转率）

加了这个优化后，执行 RVC 指令时高 16bit DFF 不翻转：
- 平均 16 × α + 16 × α × 0.5 = 24 × α（节省 25% 的 IR 动态功耗）

对深嵌入式 MCU 来说，这类雕花式节省积少成多。

---

### 1.6 "CAM 搜索"：检测 JALR 对 IR 中指令的 rs1 依赖

```verilog
// IR 中存的是 EXU 正在处理的指令
// dec2ifu_rden/rdidx：IR 那条指令是否写寄存器，写哪个
wire ir_rden  = dec2ifu_rden;
wire [`E203_RFIDX_WIDTH-1:0] ir_rdidx = dec2ifu_rdidx;

// 当前新来的 JALR 指令，它的 rs1 编号
wire [`E203_RFIDX_WIDTH-1:0] minidec_jalr_rs1idx;

// CAM 比较：IR 里指令的 rd 和 JALR 的 rs1 是不是同一个寄存器？
wire jalr_rs1idx_cam_irrdidx = ir_rden                   // IR 确实要写寄存器
                             & (minidec_jalr_rs1idx == ir_rdidx)  // 且是同一个
                             & ir_valid_r;               // IR 有效
```

**CAM（Content Addressable Memory）** 本来是一种硬件查找表，这里借用这个概念：用组合逻辑比较器"搜索"IR 里的目的寄存器编号是否等于当前 JALR 的源寄存器编号。

**举个例子：**

```asm
add  x1, x2, x3   ; 写 x1  ← 这条在 IR 里（ir_rdidx = 1）
jalr x0, x1, 0    ; 读 x1  ← 这条刚被 IFU 解码
```

`jalr_rs1idx_cam_irrdidx = 1`（有依赖！），BPU 等待 IR 中的 `add` 指令完成写回 x1 之后，才能读 x1 做分支预测。

---

### 1.7 乘除法 back-to-back 融合

```verilog
assign ifu_muldiv_b2b_nxt = 
    (
        (minidec_mul & dec2ifu_mulhsu)    // 当前是 MUL，上一条是 MULH/MULHU/MULHSU
      | (minidec_div  & dec2ifu_rem )    // 当前是 DIV，上一条是 REM
      | (minidec_rem  & dec2ifu_div )    // 当前是 REM，上一条是 DIV
      | (minidec_divu & dec2ifu_remu)
      | (minidec_remu & dec2ifu_divu)
    )
    & (ir_rs1idx_r == ir_rs1idx_nxt)     // 两条指令 rs1 相同
    & (ir_rs2idx_r == ir_rs2idx_nxt)     // 两条指令 rs2 相同
    & (~(ir_rs1idx_r == ir_rdidx))       // 上条指令的 rd 不等于 rs1（无 WAR）
    & (~(ir_rs2idx_r == ir_rdidx))       // 上条指令的 rd 不等于 rs2（无 WAR）
```

**为什么 MULH+MUL 可以融合？**

RISC-V 32位乘法结果是 64 位，但寄存器只有 32 位，所以编译器会生成两条相邻指令：
```asm
mulh  rd_hi, rs1, rs2   ; 得到积的高32位
mul   rd_lo, rs1, rs2   ; 得到积的低32位
```
这两条指令源操作数完全相同、运算过程高度重叠，硬件可以把两次乘法"合并"成一次宽乘法，只用一个乘法器就算出完整 64 位结果，节省一次长指令周期。

同理 DIV + REM 也可以一次除法得到商和余数。

---

### 1.8 PC 下一值的优先级链

```verilog
wire [`E203_PC_SIZE-1:0] pc_add_op1 = 
    pipe_flush_req     ? pipe_flush_add_op1 :  // 优先级最高：flush（分支误判/异常）
    dly_pipe_flush_req ? pc_r               :  // 延迟的flush：用保存的旧PC
    ifetch_replay_req  ? pc_r               :  // replay：重取当前地址
    bjp_req            ? prdt_pc_add_op1    :  // 分支预测：用BPU算出的目标
    ifu_reset_req      ? pc_rtvec           :  // 复位：从复位向量取
                         pc_r;                 // 默认：PC自增（顺序取指）

wire [`E203_PC_SIZE-1:0] pc_add_op2 = 
    pipe_flush_req     ? pipe_flush_add_op2 :
    dly_pipe_flush_req ? `E203_PC_SIZE'b0   :  // op2=0，PC不变（下拍重发）
    ifetch_replay_req  ? `E203_PC_SIZE'b0   :
    bjp_req            ? prdt_pc_add_op2    :
    ifu_reset_req      ? `E203_PC_SIZE'b0   :
                         pc_incr_ofst;          // 顺序：+2（RVC）或+4（RV32）
```

**优先级设计哲学：** 优先级越高的事件越紧急、代价越大，放在最前面保证不会被覆盖。flush 是"纠正错误"，一旦发生必须立刻响应，否则 CPU 会继续执行错误路径的指令。

```
pc_nxt = {(pc_add_op1 + pc_add_op2)[31:1], 1'b0}
```

**注意最后的 `1'b0`**：RISC-V 要求所有取指地址最低位为 0（2字节对齐，即使是 PC+2 也是偶数）。不论加法结果如何，直接把最低位强制清零，简单粗暴地保证对齐，这比加一个条件判断更省逻辑。

---

### 1.9 在途请求追踪（`out_flag_r`）

```verilog
// out_flag_r：当前是否有"已发出但还没收到响应"的ICB请求
wire out_flag_set = ifu_req_hsked;   // 发出请求时置1
wire out_flag_clr = ifu_rsp_hsked;   // 收到响应时清0
...
// 新请求的发送条件：没有在途请求，或者在途请求刚好这拍收到响应
wire new_req_condi = (~out_flag_r) | out_flag_clr;
assign ifu_req_valid = ifu_req_valid_pre & new_req_condi;
```

**e203 为什么只允许 1 个在途请求？**

1. ITCM 是单端口 SRAM，一次只能做一次 read（不支持流水并发）
2. 2级流水线不需要激进的预取，1个在途足够流水
3. 简化 OITF 和 hazard 检测逻辑，节省面积

有了 `out_flag_r`，就能精确知道 ITCM "是否已经在忙"，从而阻止重复发请求。

---

## 精读二：`e203_ifu_litebpu.v`

### 2.1 一行代码决定是否预测跳转

```verilog
assign prdt_taken = (dec_jal | dec_jalr | (dec_bxx & dec_bjp_imm[`E203_XLEN-1]));
```

拆解这三个条件：
- `dec_jal`：JAL 是**无条件**跳转，必然跳，直接置 1
- `dec_jalr`：JALR 也是**无条件**跳转（只是目标地址需计算），直接置 1
- `dec_bxx & dec_bjp_imm[31]`：条件跳转（Bxx），只看立即数的**符号位**

**立即数符号位 = 偏移量的正负：**

```
dec_bjp_imm[31] = 1  →  偏移量为负  →  向后跳（循环）  →  预测跳
dec_bjp_imm[31] = 0  →  偏移量为正  →  向前跳（if分支）→  预测不跳
```

用**一个 bit** 的比较完成了一个"统计学上合理"的分支预测决策，面积和时延极小。

---

### 2.2 rs1xn 两拍读寄存器

对 `JALR rs1=xN`（N ≠ 0, 1），xN 的值只能从寄存器堆里读，而寄存器堆在 EXU 里，IFU 在 EXU 前面，不能直接访问。但 e203 专门给 BPU 预留了一个寄存器堆读端口 `rf2bpu_rs1`，让 IFU 可以直接读 xN。

```verilog
// 等 xN 没有依赖后，发起一次读寄存器请求（只保持一拍：set后下拍自动clr）
wire rs1xn_rdrf_set = (~rs1xn_rdrf_r)             // 还没发过读请求
                    & dec_i_valid & dec_jalr        // 确实是 JALR xN
                    & dec_jalr_rs1xn               // rs1 != x0, x1
                    & ((~jalr_rs1xn_dep)            // 没有数据依赖
                    | jalr_rs1xn_dep_ir_clr);       // 或者依赖已被清除

wire rs1xn_rdrf_clr = rs1xn_rdrf_r;               // 下一拍立刻清除（只保持1拍）

sirv_gnrl_dfflr #(1) rs1xn_rdrf_dfflrs(rs1xn_rdrf_ena, rs1xn_rdrf_nxt, rs1xn_rdrf_r, clk, rst_n);
```

**两拍时序图：**

```
Cycle 0:  rs1xn_rdrf_set=1 → 发出读请求 bpu2rf_rs1_ena=1
          rs1xn_rdrf_r 仍为 0（DFF 还没更新）

Cycle 1:  rs1xn_rdrf_r = 1  （读请求已发出，等待结果）
          rs1xn_rdrf_clr = 1 → 请求撤销
          这一拍：rf2bpu_rs1 上已经出现 xN 的值（寄存器堆组合逻辑）
          → bpu 使用 rf2bpu_rs1 计算跳转目标 prdt_pc_add_op1
```

读请求只保持一拍的原因：寄存器堆是组合逻辑读（一拍延迟），set 后下一拍数据就已经到了，不需要继续保持使能。

---

### 2.3 `bpu_wait` 的组合

```verilog
assign bpu_wait = jalr_rs1x1_dep   // x1 有依赖，等待
                | jalr_rs1xn_dep   // xN 有依赖，等待
                | rs1xn_rdrf_set;  // 正在读 xN，等读完（等1拍）
```

`bpu_wait = 1` 时，ifetch 中的 `ifu_new_req = 0`，IFU 不会发出新的取指请求——流水线在此处暂停一拍。

**为什么 `rs1xn_rdrf_set` 而不是 `rs1xn_rdrf_r`？**

`set` 被置为 1 的那一拍，读请求刚发出，数据还没到。下一拍（`rs1xn_rdrf_r=1`）数据到了，`clr=1`，此时 `bpu_wait=0`，IFU 立刻用数据生成目标 PC——整个等待期恰好是 1 拍，非常紧凑。

---

### 2.4 跳转目标地址计算（多路选择器）

```verilog
assign prdt_pc_add_op1 =
    (dec_bxx | dec_jal)          ? pc               // JAL/Bxx：目标 = PC + 偏移
  : (dec_jalr & dec_jalr_rs1x0)  ? `E203_PC_SIZE'b0 // JALR x0：目标 = 0 + 偏移
  : (dec_jalr & dec_jalr_rs1x1)  ? rf2bpu_x1        // JALR x1：目标 = x1 + 偏移
  :                                 rf2bpu_rs1;       // JALR xN：目标 = xN + 偏移

assign prdt_pc_add_op2 = dec_bjp_imm; // op2 统一是立即数偏移
```

**精妙处**：op2 对所有情况都是 `dec_bjp_imm`，只有 op1 变化，最终 `target = op1 + op2`。这把四种情况的地址计算**统一到同一个加法器**，不需要为每种情况单独一个加法器或多路选择器，面积更小。

---

## 精读三：`e203_ifu_ift2icb.v`

### 3.1 三种取指场景分类

ift2icb 在每次取指请求时，先判断这次取指属于哪种场景：

```verilog
// 场景1：同一个 64bit lane、需要跨越 lane 边界、但 ITCM 输出仍 hold 住
wire req_same_cross_holdup = ifu_req_lane_same & ifu_req_lane_cross & ifu_req_lane_holdup;

// 场景2：需要两次 ICB 请求（跨 lane 且旧数据不保留）
wire req_need_2uop = (ifu_req_lane_same  & ifu_req_lane_cross & (~ifu_req_lane_holdup))
                   | ((~ifu_req_lane_same) & ifu_req_lane_cross);

// 场景3：不需要 ICB 请求（同 lane、不跨界、ITCM hold 住）
wire req_need_0uop = ifu_req_lane_same & (~ifu_req_lane_cross) & ifu_req_lane_holdup;
```

三个条件词的含义：

| 条件 | 含义 |
|------|------|
| `ifu_req_lane_same` | 新 PC 和上次 PC 在同一个 64bit lane（`PC[15:3]` 相同） |
| `ifu_req_lane_cross` | PC 处于 64bit lane 的最后 2 字节（`PC[2:1]==2'b11`），需跨 lane |
| `ifu_req_lane_holdup` | ITCM SRAM 输出数据还保持着上次读出的值（未被其他访问覆盖） |

三个变量组合出三种场景，决定需要 0、1、2 次 ICB 访问。

---

### 3.2 精妙：`req_need_0uop` — 零次 ICB 直接用 SRAM 保持值

```verilog
// 连 ICB 请求都不用发！SRAM 的输出还保持着上次读的值
wire holdup_gen_fake_rsp_valid = icb_sta_is_1st & req_need_0uop_r;
```

**举例理解：**

```
上一条指令：取 PC=0x80000000 (64bit lane #0 的 bit[31:0])
              → SRAM 读出 lane#0 的所有 64bit，输出保持不变

当前指令：取 PC=0x80000004 (仍在 lane#0，且 PC[2:1]=01，不跨 lane)
             → 数据仍然是 lane#0 的那 64bit（SRAM 没被重新访问）
             → 直接从 SRAM 输出的 bit[47:16] 切出指令，无需再发 ICB！
```

SRAM 是"保持型"（Hold-up）输出：地址选通不拉高，输出维持上次读出的数据。利用这个特性，同一 64bit lane 内连续取指**每两条只需一次 SRAM 访问**。

---

### 3.3 ICB 状态机（4 状态）

```verilog
localparam ICB_STATE_IDLE    = 2'd0;  // 空闲
localparam ICB_STATE_1ST     = 2'd1;  // 已发第1次ICB请求，等响应
localparam ICB_STATE_WAIT2ND = 2'd2;  // 第1次响应到了，但第2次ICB cmd还没就绪
localparam ICB_STATE_2ND     = 2'd3;  // 已发第2次ICB请求，等响应
```

状态转移逻辑（关键路径）：

```verilog
// 从 STATE_1ST 的退出：
//   如果第一次响应要存 leftover（跨 lane 的情况），退出条件是
//   "第一次ICB响应握手"（把数据存入leftover）
//   如果第一次响应直接出结果（不跨 lane），退出条件是
//   "对 IFU 层的响应握手"
assign state_1st_exit_ena = icb_sta_is_1st & (
    ifu_icb_rsp2leftover ? ifu_icb_rsp_hsked    // 存leftover
                         : i_ifu_rsp_hsked);     // 直接送出

assign state_1st_nxt =
    (req_need_2uop_r & (~ifu_icb_cmd_ready)) ? ICB_STATE_WAIT2ND  // 需2次但cmd没就绪
  : (req_need_2uop_r &   ifu_icb_cmd_ready ) ? ICB_STATE_2ND      // 直接发第2次
  :  ifu_req_hsked                           ? ICB_STATE_1ST       // 下一条来了
  :                                            ICB_STATE_IDLE;
```

**WAIT2ND 存在的意义：** 第一次 SRAM 响应来了，需要发第二次 ICB 命令，但 ITCM 的 cmd channel 还没 ready（可能 LSU 也在竞争 ITCM）。这时不能丢掉第一次的数据，需要一个中间等待状态。leftover 寄存器承担了数据暂存职责。

---

### 3.4 leftover 缓冲的精确控制

```verilog
// leftover 的写入时机有两种：
// Case 1：holdup 场景，直接从 SRAM 输出取高16bit
wire holdup2leftover_ena = ifu_req_hsked & holdup2leftover_sel;

// Case 2：2次ICB场景，第1次ICB响应的高16bit
wire uop1st2leftover_ena = ifu_icb_rsp_hsked & uop1st2leftover_sel;

assign leftover_ena = holdup2leftover_ena | uop1st2leftover_ena;
assign leftover_nxt = put2leftover_data[15:0]; // 统一来源：SRAM输出的高16bit
```

**注意 `put2leftover_data` 的构造：**

```verilog
wire [15:0] put2leftover_data = 
    ({16{icb_cmd2itcm_r}} & ifu2itcm_icb_rsp_rdata[63:48])  // ITCM 返回的高16bit
  | ({16{icb_cmd2biu_r}}  & ifu2biu_icb_rsp_rdata  [31:16]) // 系统内存的高16bit
```

这个 `{16{icb_cmd2itcm_r}} & data` 是常见的 **one-hot 选择器**写法：

```
icb_cmd2itcm_r = 1  →  {16{1}} & data_itcm = data_itcm    ✓
icb_cmd2itcm_r = 0  →  {16{0}} & data_itcm = 16'b0        （不选中）

两者 OR 起来 = 被选中那路的数据
```

这等同于 `case` 语句，但不需要优先级仲裁，面积更小。

---

### 3.5 最终指令拼接

```verilog
// 选择最终指令的两种来源
wire rsp_instr_sel_leftover = (icb_sta_is_1st & req_same_cross_holdup_r)  // holdup跨lane
                            | icb_sta_is_2nd;    // 2次ICB的第2次响应来了

assign i_ifu_rsp_instr =
    ({32{rsp_instr_sel_leftover}} & {ifu_icb_rsp_rdata_lsb16, leftover_r})  // 高低拼接
  | ({32{rsp_instr_sel_icb_rsp }} & ifu_icb_rsp_instr);                      // 直接取对齐窗口
```

**跨 lane 拼接示意：**

```
leftover_r     =  指令低 16bit（上次/上上次读的 SRAM 高端数据）
ifu_icb_rsp_rdata[15:0]  =  指令高 16bit（这次读的 SRAM 低端数据）

拼出来：{ifu_icb_rsp_rdata[15:0], leftover_r[15:0]}
         = 指令[31:16]              + 指令[15:0]
         = 完整32bit指令 ✓
```

**先低后高、还是先高后低？** RISC-V 指令是小端（little-endian）：低地址存低字节。同时 Verilog `{}` 拼接是高位在左。所以 `{高16bit数据, 低16bit数据}` 正好对应完整的 32bit RISC-V 指令格式。

---

### 3.6 ITCM 地址的位宽截断

```verilog
// 发给 ITCM 的 cmd addr：只取低 ITCM_ADDR_WIDTH 位
assign ifu2itcm_icb_cmd_addr = ifu_icb_cmd_addr[`E203_ITCM_ADDR_WIDTH-1:0];
//                                              ↑ 只取低16bit（ITCM是64KB）
```

这里**没有任何条件判断**。因为只有在 `ifu_icb_cmd2itcm=1` 时 `ifu2itcm_icb_cmd_valid` 才为 1，地址才会被 ITCM 采样——所以多余的高位地址位无所谓，ITCM 自然会忽略。

---

## 精读四：`e203_ifu_minidec.v`

### 4.1 只有 5 行的"模块体"

```verilog
module e203_ifu_minidec(
  input  [`E203_INSTR_SIZE-1:0] instr,
  output dec_rv32, dec_bjp, dec_jal, dec_jalr, dec_bxx,
  output [`E203_RFIDX_WIDTH-1:0] dec_jalr_rs1idx,
  output [`E203_XLEN-1:0] dec_bjp_imm,
  ...
);
  e203_exu_decode u_e203_exu_decode(
    .i_instr(instr),
    .i_pc(`E203_PC_SIZE'b0),    // PC 填 0（预译码不需要 PC 值）
    .i_prdt_taken(1'b0),        // 预测结果填 0（预译码不关心）
    .i_muldiv_b2b(1'b0),
    ...
    // 所有"不关心"的输出端口直接留空（不连接）
    .dec_misalgn(),
    .dec_buserr(),
    .dec_ilegl(),
    .dec_info(),
    .dec_imm(),
    ...
    // 只接需要的内容
    .dec_rv32(dec_rv32),
    .dec_bjp (dec_bjp ),
    .dec_jal (dec_jal ),
    ...
  );
endmodule
```

**精妙处：完全"复用不重造"。**

预译码本质上是完整译码的一个子集，如果单独实现一个 minidec 逻辑，一旦 exu_decode 修改（如增加新指令），minidec 也要同步修改，容易出错。

把 `e203_exu_decode` 直接例化，不需要的输出悬空——Verilog 悬空输出不会产生任何逻辑（综合器会直接优化掉）。这样 minidec 和 exu_decode 永远逻辑一致，不会出现"预译码和真正译码判断不一样"的微妙 bug。

悬空输出端口时看起来像是"资源浪费"，其实综合器会经过逻辑锥分析，把只驱动悬空端口的逻辑全部剪掉（dead code elimination），最终综合出来的面积和单独写一个窄预译码器基本一样大。

---

## 精读小结

| 位置 | 精妙的设计 | 解决的问题 |
|------|-----------|-----------|
| `ifetch.v` | SR 触发器模式 | 统一状态管理，功耗最小 |
| `ifetch.v` | `pipe_flush_ack = 1'b1` | 切断 EXU→IFU 组合路径，改善时序 |
| `ifetch.v` | IR 高低 16bit 分开使能 | RVC 指令减少 25% IR 翻转功耗 |
| `ifetch.v` | `jalr_rs1idx_cam_irrdidx` | 1位 CAM 精准检测 RAW 依赖 |
| `ifetch.v` | B2B乘除法检测 | 合并 MULH+MUL / DIV+REM 为单次运算 |
| `ifetch.v` | `pc_nxt[0] = 1'b0` | 强制对齐，省去conditional逻辑 |
| `litebpu.v` | `dec_bjp_imm[31]` 一位预测 | 用符号位做向前/向后跳转静态预测 |
| `litebpu.v` | 统一 op1+op2 加法器 | 四种 JALR/Bxx 共用一个加法器 |
| `litebpu.v` | `rs1xn_rdrf_r` 一拍脉冲 | 精确一拍读寄存器，不多等 |
| `ift2icb.v` | `req_need_0uop` 零次ICB | hold-up 属性复用，节省 SRAM 访问 |
| `ift2icb.v` | one-hot选择器 `{N{sel}} & data` | 无优先级仲裁，面积比 case 小 |
| `ift2icb.v` | bypbuf 深度1 FIFO | 打破 IFU→ITCM→LSU→EXU 死锁链 |
| `minidec.v` | 复用 `exu_decode`，悬空不用的输出 | 保证预译码和完整译码永远一致 |
