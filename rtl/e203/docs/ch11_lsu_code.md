# 第11章 LSU 代码精读

> 本章对 e203 存取单元（LSU）核心源文件进行逐行精读，重点解析地址路由、写数据对齐、应答分拆、符号扩展等关键逻辑的设计意图。阅读本章前建议先完成第4章（LSU 架构设计）的理论学习。

---

## 第0节 跨模块信号溯源

### 0.1 LSU 接收的跨模块信号

| 信号名 | 来源模块 | 含义 |
|---|---|---|
| `agu_icb_cmd_*` | `e203_exu_alu_lsuagu` | AGU 向 LSU 发送访存命令（地址/数据/掩码/类型） |
| `agu_icb_cmd_usign` | `e203_exu_alu_lsuagu` | 是否无符号扩展（LBU/LHU 与 LB/LH 的区别） |
| `agu_icb_cmd_itag` | `e203_exu_alu.v`（通过 disp）| 指令标签：OITF 中的 slot 编号，写回时对应到 rd |
| `agu_icb_cmd_back2agu` | `e203_exu_alu_lsuagu` | AMO/非对齐指令需要把响应回发给 AGU，而非直接写回 |
| `itcm_region_indic` | `e203_itcm_ctrl` | ITCM 地址区域标识（高位基地址），用于路由 |
| `dtcm_region_indic` | `e203_dtcm_ctrl` | DTCM 地址区域标识，用于路由 |
| `nice_mem_holdup` | `e203_exu_nice` | NICE 协处理器占用内存总线，暂停 AGU 访存 |
| `commit_mret` / `commit_trap` | `e203_exu_commit` | mret/trap 发生时，需要清除 excl（LR/SC 预约） |

### 0.2 LSU 向外发出的跨模块信号

| 信号名 | 目标模块 | 含义 |
|---|---|---|
| `lsu_o_valid` / `lsu_o_wbck_wdat` | `e203_exu_longpwbck` | LSU 完成后送写回数据给长流水写回模块 |
| `lsu_o_wbck_itag` | `e203_exu_longpwbck` | 写回对应的 OITF 标签，用于 OITF 退出 |
| `lsu_o_wbck_err` | `e203_exu_longpwbck` | 总线错误标志，写回模块据此决定是否触发异常 |
| `lsu_o_cmt_buserr / badaddr` | `e203_exu_commit`（通过 longp_excp）| 访存错误信息，进入异常处理 |
| `amo_wait` | `e203_exu_disp` | AMO 进行中，禁止 WFI 应答 |
| `itcm_icb_cmd_*` | `e203_itcm_ctrl` | LSU 向 ITCM 发出 ICB 访存命令（load/store 代码区数据） |
| `dtcm_icb_cmd_*` | `e203_dtcm_ctrl` | LSU 向 DTCM 发出 ICB 访存命令 |
| `biu_icb_cmd_*` | `e203_biu` | LSU 向外部总线发出 ICB 命令（访问外设/DRAM） |

---

## 精读一：`e203_lsu.v` — LSU 顶层包装

`e203_lsu.v` 是 LSU 的顶层包装模块，主要职责是：
1. 将 `e203_exu_alu_lsuagu.v` 产生的 AGU ICB 命令透传给 `e203_lsu_ctrl.v`
2. 将 NICE 协处理器的 ICB 命令合并进来
3. 对各个下游存储器（ITCM/DTCM/BIU）的 ICB 接口进行汇集

结构上非常干净：`e203_lsu.v` 本身几乎没有逻辑，全是连线（instantiation + port mapping），核心逻辑全在 `e203_lsu_ctrl.v` 中。

---

## 精读二：`e203_exu_alu_lsuagu.v` — 地址生成单元（AGU）

AGU 负责计算访存地址并把访存命令发往 LSU-ctrl。它和 ALU 共用数据通路（adder/shifter），体现的正是 e203 "节省面积"的一贯策略。

### 2.1 地址计算完全复用 ALU 加法器

```verilog
// 地址计算：rs1 + imm
assign agu_req_alu_op1 = icb_sta_is_idle ? agu_i_rs1 : ...;
assign agu_req_alu_op2 = icb_sta_is_idle ? agu_addr_gen_op2 : ...;
// agu_addr_gen_op2 = ofst0 ? 0 : imm
// ofst0 是 AMO/LR/SC 的特殊处理：AMO 不加偏移，地址就是 rs1

assign agu_icb_cmd_addr = agu_req_alu_res[`E203_ADDR_SIZE-1:0];
// agu_req_alu_res 来自 ALU datapath 的加法器输出
// AGU 根本没有独立的加法器！
```

模块注释原文明确说明：  
*"for single-issue machine, seems the AGU must be shared with ALU, otherwise it wasted the area for no points"*

这是 e203 面积优化的核心：单发射机器每次只有一条指令在执行，AGU 和 ALU 不会同时工作，共用一个加法器完全没有冲突，却节省了一块完整加法器电路。

### 2.2 `agu_i_longpipe` 对齐访存才是长流水

```verilog
assign agu_i_longpipe = agu_i_algnldst;  // 对齐 load/store 走长流水
```

对齐的 load/store 指令不能在 ALU commit 阶段立即写回（因为数据在内存里）——它们必须发出 ICB 命令，等 SRAM/外设响应。响应可能延迟若干周期，所以必须进入 OITF 形成"长流水"等待写回。

AMO 指令虽然也访存，但它走状态机（多周期），在 AGU 内部自己处理写回，不通过长流水路径。

### 2.3 写数据的"重复填充"模式

```verilog
wire [`E203_XLEN-1:0] algnst_wdata =
    ({`E203_XLEN{agu_i_size_b }} & {4{agu_i_rs2[ 7:0]}})   // 字节→4份填满32位
  | ({`E203_XLEN{agu_i_size_hw}} & {2{agu_i_rs2[15:0]}})   // 半字→2份填满32位
  | ({`E203_XLEN{agu_i_size_w }} & {1{agu_i_rs2[31:0]}});  // 字→直接32位
```

e203 的 SRAM/ICB 总线宽度是 32 位，最小访问粒度通过 `wmask` 字节使能来控制——而不是把数据放在特定字节位置。因此写数据必须把有效字节**重复到所有 32 位**：

- 写 `SB rs2, 5(rs1)`（字节写）：rs2 的低 8 位重复 4 次填满 32 位
- 写 `SH rs2, 4(rs1)`（半字写）：rs2 的低 16 位重复 2 次填满 32 位

真正生效的字节由写掩码 `wmask` 决定，总线远端的 SRAM 只写入掩码指示的字节，重复的数据根本不会被"多写"。这种"数据重复 + 掩码选择"是 AMBA AXI/AHB 总线的标准约定。

### 2.4 写掩码由地址低位和操作大小计算

```verilog
wire [`E203_XLEN/8-1:0] algnst_wmask =
    ({`E203_XLEN/8{agu_i_size_b }} & (4'b0001 << agu_icb_cmd_addr[1:0]))
  | ({`E203_XLEN/8{agu_i_size_hw}} & (4'b0011 << {agu_icb_cmd_addr[1],1'b0}))
  | ({`E203_XLEN/8{agu_i_size_w }} & (4'b1111));
```

掩码是 4 位（对应 32 位数据的 4 个字节），各 bit 表示"该字节是否写入"：

| 操作 | 地址 `[1:0]` | wmask 计算 | 结果示例 |
|---|---|---|---|
| SB（字节） | 0b00 | `0001 << 00` | `0001`（只写 byte0） |
| SB（字节） | 0b10 | `0001 << 10` | `0100`（只写 byte2） |
| SH（半字） | 0b00 | `0011 << 0b00` （地址[1]=0） | `0011`（byte0+byte1） |
| SH（半字） | 0b10 | `0011 << 0b10` （地址[1]=1） | `1100`（byte2+byte3） |
| SW（字） | 任意 | `1111` | `1111`（全部4字节） |

注意：半字掩码使用 `{addr[1], 1'b0}` 而不是 `addr[1:0]`，因为半字对齐时 `addr[0]` 必然为 0，只需要 `addr[1]` 来区分"低半字"还是"高半字"。

### 2.5 AMO 状态机（E203_SUPPORT_AMO）

当编译支持 AMO 时，AGU 有一个完整的多状态有限状态机：

```
IDLE → (AMO 指令 & OITF 空) →
1ST  → (收到读响应) →
AMOALU → (ALU 计算完毕) →  
AMORDY → (无条件) →
WAIT2ND → (ICB CMD ready) →
2ND  → (收到写响应) →
WBCK → (AGU commit ready) →
IDLE
```

关键设计点：
- **AMO 必须等 OITF 空**：注释解释了原因——若 OITF 中还有长流水指令出错需要冲刷，而 AMO 已经在访存中途，会形成死锁（无法冲刷，无法推进）。
- **`amo_wait = ~icb_sta_is_idle`**：任何 AMO 状态机活跃时，都会置起 `amo_wait` 信号，通知 `disp.v` 阻止 WFI 应答（中断不能在 AMO 中途响应）。

### 2.6 `flush_block`：冲刷时阻止新访存发出

```verilog
wire flush_block = flush_req & icb_sta_is_idle;
wire agu_i_load  = agu_i_info[...LOAD] & (~flush_block);
wire agu_i_store = agu_i_info[...STORE] & (~flush_block);
```

当 EXU commit 发出冲刷请求（分支错误或异常），且 AGU 状态机还处于 IDLE（尚未发出第一个 ICB），就屏蔽新的 load/store 请求，避免将一条"已被冲刷"的指令的访存发到内存。如果 AGU 状态机已经启动（非 IDLE），则必须让它完成（AMO 不可中断）。

---

## 精读三：`e203_lsu_ctrl.v` — LSU 控制核心

`e203_lsu_ctrl.v` 是 LSU 中逻辑最密集的模块，负责：
1. 仲裁多个访存来源（AGU / NICE）
2. 跟踪在途（outstanding）请求
3. 路由到正确的下游存储器（ITCM/DTCM/BIU）
4. 应答分拆回正确的来源
5. 读数据对齐和符号扩展

### 3.1 NICE 抢占优先级

```verilog
assign agu_icb_cmd_valid_pos = agu_icb_cmd_valid & (~nice_mem_holdup);
assign agu_icb_cmd_ready     = agu_icb_cmd_ready_pos & (~nice_mem_holdup);
```

当 NICE（协处理器）需要访问内存时，会置起 `nice_mem_holdup`，LSU-ctrl 就把 AGU 的 valid 屏蔽为 0，同时把 AGU 的 ready 也屏蔽为 0，使 AGU 进入等待状态。

等 NICE 释放内存后，AGU 才能再次发出命令。这是一个简单但有效的"总线请求/释放"机制。

### 3.2 仲裁器：`sirv_gnrl_icb_arbt`

```verilog
sirv_gnrl_icb_arbt # (
  .ARBT_SCHEME (0),  // 优先级仲裁（不是轮询）
  .FIFO_OUTS_NUM (`E203_LSU_OUTS_NUM),  // 在途深度
  ...
) u_lsu_icb_arbt(...);
```

`sirv_gnrl_icb_arbt` 是通用 ICB 仲裁器，支持多个主端合并为一个从端。配置为"优先级仲裁"（scheme=0），NICE 优先级高于 AGU（从 `arbt_bus_icb_cmd_valid` 的拼接顺序决定）。

仲裁器还内置一个"用户数据"（usr）随行 FIFO——AGU 发出命令时把 `{back2agu, usign, read, size, itag, addr, excl}` 打包进 `agu_icb_cmd_usr`，响应回来时从 FIFO 中取出对应的用户数据，确保即使多个请求在途，每个响应都能找到自己对应的原始命令参数。

### 3.3 分拆 FIFO（splt_fifo）：记录每个在途请求去了哪里

```verilog
assign splt_fifo_wdat = {
  arbt_icb_cmd_biu, arbt_icb_cmd_dcache,
  arbt_icb_cmd_dtcm, arbt_icb_cmd_itcm,
  ..., arbt_icb_cmd_usr
};
```

每当一个 ICB CMD 发出（`splt_fifo_wen = cmd_valid & cmd_ready`），就把"这个请求路由到了哪个目标"写进 FIFO。

**为什么需要这个 FIFO？**

ICB 的 CMD 和 RSP 通道是分离的（类似 AXI 的 AR/R 分离）。当 CMD 已经发出后，响应会在若干周期后返回。如果有多个在途请求，响应顺序和命令发出顺序相同（in-order），所以按 FIFO 顺序弹出就能知道"这个响应对应的是 ITCM 还是 DTCM"，从而决定把响应数据路由到哪条 `rsp.rdata` 线上。

每当 RSP 握手完成（`splt_fifo_ren = rsp_valid & rsp_ready`）就弹出 FIFO 头部。

### 3.4 地址路由：用高位地址匹配区域标识

```verilog
wire arbt_icb_cmd_itcm =
    (arbt_icb_cmd_addr[`E203_ITCM_BASE_REGION] == itcm_region_indic[`E203_ITCM_BASE_REGION]);
wire arbt_icb_cmd_dtcm =
    (arbt_icb_cmd_addr[`E203_DTCM_BASE_REGION] == dtcm_region_indic[`E203_DTCM_BASE_REGION]);
wire arbt_icb_cmd_biu  = (~arbt_icb_cmd_itcm) & (~arbt_icb_cmd_dtcm) & (~arbt_icb_cmd_dcache);
```

路由逻辑：比较地址的"基地址区域段"（高若干位）与运行时配置寄存器中的区域标识符，相等则命中该存储器。没有命中任何 TCM 的请求默认发往 BIU（外部总线接口），覆盖所有外设地址。

`itcm_region_indic` 来自 `e203_itcm_ctrl`，由 CPU 的配置参数决定，通常对应 ITCM 的物理基地址（如 `0x80000000`）。这种"运行时对比"而不是"编译时宏"的方式，让同一套 RTL 可以适配不同的内存映射配置。

### 3.5 防止跨目标乱序：`cmd_diff_branch`

```verilog
// 如果 LSU 支持多个在途请求（OUTS_NUM > 1），
// 必须防止新请求路由到与当前在途请求不同的目标
wire cmd_diff_branch = (~splt_fifo_empty) &
    (~({arbt_icb_cmd_biu, arbt_icb_cmd_dtcm, arbt_icb_cmd_itcm}
    == {arbt_icb_rsp_biu, arbt_icb_rsp_dtcm, arbt_icb_rsp_itcm}));

wire arbt_icb_cmd_addi_condi = (~splt_fifo_full) & (~cmd_diff_branch);
```

**为什么不能有跨目标的多个在途请求？**

假设在途队列中有一个到 DTCM 的请求，如果此时允许再发一个到 BIU 的请求，两个请求分别去了不同的下游——DTCM 可能 1 拍响应，BIU 可能 5 拍响应。但 lsu_ctrl 的响应路由逻辑是按顺序（FIFO）工作的，如果允许"跨目标"多请求，就需要处理乱序响应，大幅增加复杂度。

e203 用最简单的策略解决：**只允许同一时间所有在途请求发往同一目标**。条件 `cmd_diff_branch` 在新请求的目标与当前在队列最末的目标不同时，阻止新命令发出，直到在途队列耗尽。

对于默认深度为 1（`E203_LSU_OUTS_NUM_IS_1`）的配置，`cmd_diff_branch` 永远为 0，代码简化掉。

### 3.6 下游 ready 的简化处理：all-ready 策略

```verilog
// 理论上应该是：
// assign arbt_icb_cmd_ready_pos =
//   (cmd_biu & biu_ready) | (cmd_itcm & itcm_ready) | (cmd_dtcm & dtcm_ready);
// 但这会产生从 cmd_addr → 目标选择 → ready 的组合路径

// e203 改用：总是检查所有下游的 ready
assign all_icb_cmd_ready = biu_icb_cmd_ready
    `ifdef E203_HAS_DTCM & (dtcm_icb_cmd_ready) `endif
    `ifdef E203_HAS_ITCM & (itcm_icb_cmd_ready) `endif ;

assign arbt_icb_cmd_ready_pos = all_icb_cmd_ready;
```

注释原文：  
*"To cut the in2out path from addr to the cmd_ready signal, we just always use the simplified logic to always ask for all of the downstream components to be ready. This may impact performance a little bit in corner case, but doesn't really hurt the common case."*

**问题**：如果用"按目标选择 ready"，那么 `ready` 信号的生成路径是：`addr → 目标判断 → ready`，这是一条组合路径，会增加关键路径延迟。

**解决**：直接使用"所有下游都 ready"作为命令发出的条件。代价是极端情况下（某个目标没响应但另一个有响应）可能浪费一拍——但大多数时候 ITCM/DTCM 都是单周期就绪，BIU 也很快就绪，实际性能损失极小。

每个下游的 valid 使用了独立的 `all_icb_cmd_ready_excp_XXX` 变体（排除自身），避免 valid 通路上也产生循环。

### 3.7 响应多路选择：AND-mask 技巧再现

```verilog
assign {arbt_icb_rsp_valid, arbt_icb_rsp_err, arbt_icb_rsp_excl_ok, arbt_icb_rsp_rdata} =
    ({`E203_XLEN+3{arbt_icb_rsp_biu  }} & {biu_icb_rsp_valid,   biu_icb_rsp_err,   biu_icb_rsp_excl_ok,   biu_icb_rsp_rdata  })
  | ({`E203_XLEN+3{arbt_icb_rsp_dtcm }} & {dtcm_icb_rsp_valid,  dtcm_icb_rsp_err,  dtcm_icb_rsp_excl_ok,  dtcm_icb_rsp_rdata })
  | ({`E203_XLEN+3{arbt_icb_rsp_itcm }} & {itcm_icb_rsp_valid,  itcm_icb_rsp_err,  itcm_icb_rsp_excl_ok,  itcm_icb_rsp_rdata });
```

这是一个 3 路"one-hot AND-mask"选择器，等价于：
```
if (rsp_biu)  out = biu_rsp_data;
if (rsp_dtcm) out = dtcm_rsp_data;
if (rsp_itcm) out = itcm_rsp_data;
```

用 AND-mask + OR 替代 case/if-else，在 RTL 中是标准的 one-hot 多路选择写法。因为三个信号互斥（来自 splt_fifo 中的路由标记），多路 OR 的结果不会有两路同时有效，语义完全正确。

同理，响应 ready 也按路由分发：
```verilog
assign biu_icb_rsp_ready  = arbt_icb_rsp_biu   & arbt_icb_rsp_ready;
assign dtcm_icb_rsp_ready = arbt_icb_rsp_dtcm  & arbt_icb_rsp_ready;
assign itcm_icb_rsp_ready = arbt_icb_rsp_itcm  & arbt_icb_rsp_ready;
```

### 3.8 `back2agu` 通路：AMO 响应不直接写回寄存器

```verilog
assign lsu_o_valid       = pre_agu_icb_rsp_valid & (~pre_agu_icb_rsp_back2agu);
assign agu_icb_rsp_valid = pre_agu_icb_rsp_valid &   pre_agu_icb_rsp_back2agu;
```

普通 load/store 的响应（`back2agu = 0`）直接送到 `lsu_o_valid`，进入长流水写回。

AMO 指令的中间响应（`back2agu = 1`，对应 AMO 的第 1 次读 uop）则回送到 `agu_icb_rsp_valid`，让 AGU 状态机继续用这个读出的值做 ALU 计算（原子的 read-modify-write 中的 modify 部分），再由 AGU 状态机发出第 2 次写 uop。

`back2agu` 信号随 CMD 一起存入 `agu_icb_cmd_usr` 字段，通过仲裁器的 usr 随行 FIFO 在 RSP 出来时恢复。

### 3.9 读数据对齐与符号扩展

```verilog
// 第一步：右移对齐（把目标字节移到 [7:0] 或 [15:0]）
wire [`E203_XLEN-1:0] rdata_algn =
    pre_agu_icb_rsp_rdata >> {pre_agu_icb_rsp_addr[1:0], 3'b0};
// addr[1:0]=0b01, size=byte → 右移 8 位 → [15:8] 移到 [7:0]
// addr[1:0]=0b10, size=half → 右移 16 位 → [31:16] 移到 [15:0]

// 第二步：根据 size 和 usign 分支
wire rsp_lbu = (size == 2'b00) & (usign == 1'b1);
wire rsp_lb  = (size == 2'b00) & (usign == 1'b0);
wire rsp_lhu = (size == 2'b01) & (usign == 1'b1);
wire rsp_lh  = (size == 2'b01) & (usign == 1'b0);
wire rsp_lw  = (size == 2'b10);

assign lsu_o_wbck_wdat =
    ({32{rsp_lbu}} & {{24{        1'b0}}, rdata_algn[ 7:0]})  // 零扩展 8→32
  | ({32{rsp_lb }} & {{24{rdata_algn[7]}}, rdata_algn[ 7:0]}) // 符号扩展 8→32
  | ({32{rsp_lhu}} & {{16{        1'b0}}, rdata_algn[15:0]})  // 零扩展 16→32
  | ({32{rsp_lh }} & {{16{rdata_algn[15]}}, rdata_algn[15:0]})// 符号扩展 16→32
  | ({32{rsp_lw }} & rdata_algn[31:0]);                       // 无需扩展
```

这里出现了两层处理：

**第一层：地址对齐右移**  
SRAM 总是以 32 位整对齐方式返回数据。如果要读 `addr=0x03` 的字节，返回的 32 位数据中有效字节在 `[31:24]`（`addr[1:0]=11`，最高字节）。右移 `addr[1:0]*8 = 24` 位，把有效字节移到最低 8 位，后续处理可以统一从 `[7:0]` 读取。

**第二层：AND-mask 符号扩展**  
符号扩展（LB/LH）：`{24/16{符号位}}` 扩展高位。无符号（LBU/LHU）：`{24/16{1'b0}}` 直接填 0。

整个过程是纯组合逻辑，没有任何 case 语句，通过 AND-mask + OR 实现，综合效果和 case 语句完全一样，但更利于编写和阅读。

### 3.10 AMO LR/SC 排他标志追踪

```verilog
wire excl_flg_set = splt_fifo_wen & arbt_icb_cmd_excl & arbt_icb_cmd_read;
// LR（Load-Reserved）：excl & read → 置标志，记录地址

wire excl_flg_clr = (splt_fifo_wen & (~arbt_icb_cmd_read) & icb_cmdaddr_eq_excladdr & excl_flg_r)
                  | commit_trap | commit_mret;
// SC（Store-Conditional）到同一地址：清除标志
// 发生 trap/mret：清除标志（保证 SC 在 trap 后返回失败）

wire arbt_icb_cmd_scond_true = arbt_icb_cmd_scond & icb_cmdaddr_eq_excladdr & excl_flg_r;
// SC 成功条件：标志有效 & 地址匹配
// SC 失败时：wmask 清零（不写内存），返回值 = 1（失败标志）
// SC 成功时：wmask 正常，返回值 = 0（成功标志）
```

RISC-V 的 LR/SC 是实现 mutex/spinlock 的基础原子操作：
- **LR.W**：读内存 + 置预约（reservation）
- **SC.W**：条件写内存，若预约仍有效（没被打断）则成功写入并返回 0，否则返回 1 不写入

e203 用一个单 bit 标志 `excl_flg_r` 和一个地址寄存器 `excl_addr_r` 实现预约，串行验证（单核不需要总线级排他；多核需要用总线互联机制）。

`commit_trap | commit_mret` 清除标志的原因：trap/mret 意味着上下文切换，"预约"不应该跨上下文保留——否则在 trap handler 中的任何 store 可能意外影响返回后的 SC。

---

## 精读小结

### LSU 各子模块功能一览

| 模块 | 核心功能 | 关键设计点 |
|---|---|---|
| `e203_lsu.v` | 顶层包装，连接 AGU + NICE | 几乎只有实例化和连线，无实质逻辑 |
| `e203_exu_alu_lsuagu.v` | 地址生成、访存命令发送 | 完全复用 ALU 加法器；longpipe 判断；重复写数据；字节掩码计算；AMO 状态机 |
| `e203_lsu_ctrl.v` | 仲裁/路由/追踪/分拆/对齐 | NICE 抢占；splt_fifo 追踪路由；cmd_diff_branch 防乱序；all-ready 时序优化；back2agu；符号扩展 |

### 贯穿 LSU 的三大设计原则

| 原则 | 体现 |
|---|---|
| **共享复用** | AGU 加法器 = ALU 加法器；sbf_0/1 暂存器 = AMO leftover buffer |
| **时序切断** | all-ready 避免 addr→ready 路径；cmd_diff_branch 简化路由追踪 |
| **简洁正确** | 只支持顺序响应（无乱序）；LR/SC 单核自行校验；splt_fifo 逐项打标记 |

### LSU 访存全链路信号流

```
EXU(disp) → AGU → agu_icb_cmd_*  →  lsu_ctrl  →  ITCM_ctrl / DTCM_ctrl / BIU
                                      ↑                                      ↓
                        nice_icb_cmd_*（NICE 抢占）            ICB RSP 返回
                                                                    ↓
                                                      符号扩展 → lsu_o_wbck_wdat
                                                                    ↓
                                                      longpwbck → regfile (wbck)
                                                                    ↓
                                                      OITF 退出 (oitf_ret_ena)
```
