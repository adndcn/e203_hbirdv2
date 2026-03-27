# 第9章 IFU 代码精读

> 本章逐文件精讲 IFU 各子模块的核心 RTL 实现，比第2章更深入。  
> 重点解答两类困惑：  
> **①** "这个信号是哪个模块传来的，为什么要传？"  
> **②** "这行赋值语句到底在做什么？"  
> 建议对照源文件一起阅读。

---

## 第0节：跨模块信号溯源（先看这里）

`e203_ifu_ifetch.v` 模块的端口中，有一批信号不是 IFU 自己产生的，而是来自 EXU（执行单元）。理解这些信号的来源，是读懂 IFU 代码的前提。

下表展示完整溯源链，以 `e203_core.v` 为转发中枢（core.v 负责将 EXU 和 IFU 的端口连在一起）：

| 信号名（IFU 侧） | 在 core.v 中的 wire | 产生模块 | 功能说明 |
|---|---|---|---|
| `pipe_flush_req` | `pipe_flush_req`（core.v:354） | `e203_exu_branchslv`（分支误判）或 `e203_exu_excp`（异常/中断），经 `e203_exu_commit` → `e203_core` 传给 IFU | 通知 IFU"之前取的指令要作废，请重新从新 PC 取指" |
| `pipe_flush_add_op1` / `pipe_flush_add_op2` | 同上（core.v:355-356） | 同上 | **注意：传的是加法器操作数，不是 PC！** EXU 把操作数传给 IFU，让 IFU 用自己内部的加法器算出新 PC；这样两边共用一个加法器，省掉一块硬件面积 |
| `ifu_halt_req` | `wfi_halt_ifu_req`（core.v 内部命名）| `e203_exu_excp`（WFI 指令触发停机）→ `e203_exu_commit` → core | CPU 执行了 WFI（等待中断）指令，EXU 通知 IFU 暂停取指，等中断来了再恢复 |
| `oitf_empty` | `oitf_empty`（core.v:361） | `e203_exu_oitf`（在途指令跟踪 FIFO）→ `e203_exu` → core | BPU 预测 JALR x1（函数返回）时，必须确认没有"在途"长指令可能修改 x1；OITF 非空说明有未完成的乘法/除法/访存，可能正在写 x1 |
| `rf2ifu_x1` | `rf2ifu_x1`（core.v:362） | `e203_exu_regfile`（寄存器堆）→ `e203_exu` → core | 寄存器 x1（ra，链接寄存器）的值直接引到 IFU。BPU 预测函数返回（`jalr x0, x1, 0`）时，直接用这个值算目标地址，不用等 EXU 执行 |
| `rf2ifu_rs1` | `rf2ifu_rs1`（core.v:363） | 同上 | 任意寄存器 xN 的值，用于 JALR xN 的预测。x1 和 xN 分两路引出，是因为 x1（ra）最常见，单独处理逻辑更简单 |
| `dec2ifu_rden` | `dec2ifu_rden`（core.v:364） | `e203_exu_decode`（EXU 内 IR 级的解码输出）→ `e203_exu_disp` → exu → core | 当前在 EXU IR 级（等待执行）的指令，是否会写目标寄存器 rd？用于 BPU 的 RAW（先写后读）冒险检测 |
| `dec2ifu_rs1en` | `dec2ifu_rs1en`（core.v:365） | 同上 | EXU IR 级指令是否读 rs1？如果 IR 指令不读 rs1，说明即使它有个 rd 和 JALR 的 rs1 相同，也不会被覆盖 |
| `dec2ifu_rdidx` | `dec2ifu_rdidx`（core.v:366） | 同上 | EXU IR 级指令写的目标寄存器编号 |
| `dec2ifu_mulhsu` | `dec2ifu_mulhsu`（core.v:367） | 同上 | EXU IR 级指令是否是 MULHSU（高位无符号乘法）。用于检测乘法 B2B 序列 |
| `dec2ifu_div/rem/divu/remu` | core.v:368-371 | 同上 | EXU IR 级指令是否是除法/余数指令。用于 DIV+REM Back-to-Back 融合检测 |

> **B2B 融合是什么？** 当编译器生成 `MUL rd1, rs1, rs2` 后紧跟 `MULH rd2, rs1, rs2` 时（或 DIV 后跟 REM），这两条指令的操作数完全相同，可以复用上一条指令的中间计算结果，节省一倍执行时间。IFU 识别出这种模式后，给 EXU 打上 `muldiv_b2b` 标记，EXU 的乘除单元就会启用融合路径。

---

## 精读一：`e203_ifu.v`（顶层 wrapper）

**文件位置**：`core/e203_ifu.v`  
**职责**：组装 IFU 的两个子模块，并向外暴露统一接口

### 1.1 模块结构

```verilog
module e203_ifu(
  // ... 端口列表 ...
);
  // 内部接口：连接两个子模块
  wire ifu_req_valid;    // ifetch → ift2icb：有取指请求
  wire ifu_req_ready;    // ift2icb → ifetch：请求被接受
  wire [`E203_PC_SIZE-1:0] ifu_req_pc;        // 本次要取的地址
  wire ifu_req_seq;        // 是顺序取指（相对上一条 +2 或 +4）
  wire ifu_req_seq_rv32;   // 顺序取指时上一条是 32bit 指令
  wire [`E203_PC_SIZE-1:0] ifu_req_last_pc;   // 上一次取指地址
  wire ifu_rsp_valid;    // ift2icb → ifetch：返回了指令
  wire ifu_rsp_ready;    // ifetch → ift2icb：ifetch 能接收
  wire ifu_rsp_err;      // 总线错误标志
  wire [`E203_INSTR_SIZE-1:0] ifu_rsp_instr; // 取回的指令数据

  e203_ifu_ifetch u_e203_ifu_ifetch(...);  // 子模块1：PC 生成 + IR 寄存器
  e203_ifu_ift2icb u_e203_ifu_ift2icb(...); // 子模块2：转换为 ICB 总线协议

  assign ifu_active = 1'b1; // IFU 永远活跃（block 级别从不休眠）
endmodule
```

**为什么要拆成两个子模块？**

- `e203_ifu_ifetch.v`：管"逻辑" —— 什么时候取、取哪里、PC 怎么计算、分支怎么预测
- `e203_ifu_ift2icb.v`：管"通信" —— 怎么把取指请求转成 ICB 总线事务，怎么处理 64bit SRAM 的地址对齐

两个子模块通过 5 组 wire 对话：`ifu_req_*`（取指请求）和 `ifu_rsp_*`（取指响应）。这是 IFU 内部专用的简化协议，不暴露到外部。

### 1.2 外部接口分组

| 接口组 | 信号 | 方向 | 说明 |
|---|---|---|---|
| ITCM ICB 总线 | `ifu2itcm_icb_cmd_*` / `ifu2itcm_icb_rsp_*` | 向 ITCM | 取指令用的总线，cmd 发地址，rsp 回数据 |
| 系统总线 | `ifu2biu_icb_cmd_*` / `ifu2biu_icb_rsp_*` | 向 BIU | 从外部 Flash/SRAM 取指 |
| IR 输出 | `ifu_o_ir`、`ifu_o_pc`、`ifu_o_valid`… | 向 EXU | 取到的指令和 PC 传给执行单元 |
| Flush 接口 | `pipe_flush_req/ack/add_op1/add_op2` | 来自 EXU | 分支误判/异常/中断时重定向 PC |
| Halt 接口 | `ifu_halt_req`、`ifu_halt_ack` | 来自 EXU | WFI 停止/恢复握手 |
| BPU 辅助 | `oitf_empty`、`rf2ifu_x1`、`dec2ifu_*` | 来自 EXU | 见第0节溯源表 |

---

## 精读二：`e203_ifu_ifetch.v`（核心取指逻辑）

**文件位置**：`core/e203_ifu_ifetch.v`  
**职责**：PC 计算、IR 寄存器、分支预测、halt/flush 处理

### 2.1 贯穿全文的"SR 触发器模式"

ifetch.v 里几乎每一个状态都用同一套 4 行代码来描述：

```verilog
wire foo_set = <某个触发 set 的条件>;   // 什么时候把 foo 置为 1
wire foo_clr = <某个触发 clr 的条件>;   // 什么时候把 foo 清为 0
wire foo_ena = foo_set | foo_clr;        // 有变化才让 DFF 翻转（省功耗）
wire foo_nxt = foo_set | (~foo_clr);     // set 优先：两者同时为1时，取 set

sirv_gnrl_dfflr #(1) foo_dfflr(foo_ena, foo_nxt, foo_r, clk, rst_n);
//  ↑ 带使能的 D 触发器：只有 foo_ena=1 时，foo_r 才在下一个时钟沿更新为 foo_nxt
```

**`foo_nxt = foo_set | (~foo_clr)` 的真值表（4种情况）：**

| `foo_set` | `foo_clr` | `foo_ena` | `foo_nxt` | 结果 |
|:---:|:---:|:---:|:---:|---|
| 0 | 0 | 0 | 1 或 0（无所谓）| DFF 不会更新（ena=0），foo_r 保持不变 ✓ |
| 0 | 1 | 1 | 0 | 清零 ✓ |
| 1 | 0 | 1 | 1 | 置1 ✓ |
| 1 | 1 | 1 | 1 | **set 优先**，结果为 1 ✓ |

**为什么这样写而不是 `if-else`？**  
RTL 里不能写 `if-else` 来描述 DFF（那是行为描述，综合出的电路结构不可控）。  
`foo_nxt = foo_set | (~foo_clr)` 是一行纯组合逻辑 assign，综合器会直接翻译成几个逻辑门，结构完全确定。

### 2.2 握手信号

```verilog
wire ifu_req_hsked  = (ifu_req_valid  & ifu_req_ready);   // 取指请求握手成功
wire ifu_rsp_hsked  = (ifu_rsp_valid  & ifu_rsp_ready);   // 取指响应握手成功
wire ifu_ir_o_hsked = (ifu_o_valid    & ifu_o_ready);      // IFU 输出被 EXU 接收
wire pipe_flush_hsked = pipe_flush_req & pipe_flush_ack;  // flush 握手成功
```

`_hsked`（handshaked 的缩写）信号大量出现在 ifetch.v 中，每次一看到它，就知道"这一拍双方握手成功，可以发生状态转移"。

### 2.3 复位向量请求（`reset_req_r`）—— 两级设计

CPU 上电复位后，第一件事是从"复位向量地址"（`pc_rtvec`）取第一条指令。这是用两级触发器实现的：

```verilog
// ---- 第一级：捕获"复位刚刚结束"这个事件 ----
wire reset_flag_r;
sirv_gnrl_dffrs #(1) reset_flag_dffrs (1'b0, reset_flag_r, clk, rst_n);
//  ↑ sirv_gnrl_dffrs 是带"异步复位"的特殊 DFF：
//    * 当 rst_n=0（复位中）：reset_flag_r 被强制置为 0（复位值）
//    * 当 rst_n=1（复位结束）：第一个时钟上升沿，DFF 正常采样 D 输入 = 1'b0... 
//      等等，D 输入是 1'b0，为什么 reset_flag_r 会变成 1？
//      ——因为这里用的是 dffrs（reset-set），它在复位释放后的第一拍，
//        会让输出从 0 跳到 1（这是 sirv 库特定的行为：rst_n 从0→1时输出置1）
//    含义：reset_flag_r == 1  ↔  "复位刚刚结束"
```

```verilog
// ---- 第二级：把"事件"转成"持续请求"，握手成功才清除 ----
wire reset_req_r;
wire reset_req_set = (~reset_req_r) & reset_flag_r;  // 复位结束且请求还没发出
wire reset_req_clr = reset_req_r   & ifu_req_hsked;  // 取指请求握手成功了才清除

wire reset_req_ena = reset_req_set | reset_req_clr;
wire reset_req_nxt = reset_req_set | (~reset_req_clr);

sirv_gnrl_dfflr #(1) reset_req_dfflr (reset_req_ena, reset_req_nxt, reset_req_r, clk, rst_n);

wire ifu_reset_req = reset_req_r;
```

**为什么要两级？**

直接用 `reset_flag_r` 触发取指有个问题：如果那一拍 `ifu_req_ready` 恰好为 0（ITCM 没准备好），取指请求会丢失。  
`reset_req_r` 是一个"保持型"请求 —— 它**一直保持 1**，直到 `ifu_req_hsked`（取指成功握手）才清零。这保证了复位后一定能从 `pc_rtvec` 开始取指。

**`pc_rtvec` 来自哪里？**  
路径：外部引脚 → `e203_cpu_top.v` → `e203_core.v` → `e203_ifu.v` → `e203_ifu_ifetch.v`。  
默认值通常为 `0x0000_0000`（调试模式）或 `0x2000_0000`（ITCM 基地址），由 `config.v` 中的 `E203_PC_RTVEC` 参数控制。

### 2.4 停止确认（`halt_ack_r`）

```verilog
wire ifu_no_outs;  // 当前没有未完成的取指请求（详见下方）

wire halt_ack_set = ifu_halt_req       // EXU 的 WFI 停止请求来了
                  & (~halt_ack_r)      // 还没有发出 ack
                  & ifu_no_outs;       // 没有在途的 ICB 请求

wire halt_ack_clr = halt_ack_r & (~ifu_halt_req);  // EXU 撤销停止请求时清除
```

**`ifu_halt_req` 来自哪里？**  
路径：`e203_exu_excp`（当 WFI 指令被执行时产生）→ `e203_exu_commit`（管理 halt 的仲裁）→ `e203_core.v`（命名为 `wfi_halt_ifu_req`）→ `e203_ifu.v`（命名为 `ifu_halt_req`）→ 本文件。

**为什么必须等 `ifu_no_outs`？**  
如果 IFU 已经向 ITCM 发出了一个读请求，ITCM 还没响应时就宣布"停止了"，那 ITCM 的响应数据回来时 IFU 已停机，无法接收，会造成总线死锁（`rsp_valid` 永远无人消费）。

```verilog
// out_flag_r：记录"有没有在途请求"的 1bit 寄存器
wire out_flag_set = ifu_req_hsked;   // 发出请求时置1
assign out_flag_clr = ifu_rsp_hsked; // 收到响应时清0

// ifu_no_outs 的定义（注意：用 rsp_valid 而非 rsp_hsked）
assign ifu_no_outs = (~out_flag_r) | ifu_rsp_valid;
```

**为什么用 `ifu_rsp_valid` 而不是 `ifu_rsp_hsked`？**  
注释原文说：在 WFI 场景下，不能期望响应一定被握手（因为 IFU 已经在准备停机，`rsp_ready` 可能为 0），如果判断条件包含 `rsp_ready`，可能造成死锁。只要 `rsp_valid` 拉高，就说明上一条请求已经有数据回来了，可以安全停机。

### 2.5 流水线 Flush（`pipe_flush_ack = 1'b1`）

```verilog
// ---- 永远立刻接受 flush 请求 ----
assign pipe_flush_ack = 1'b1;
```

**`pipe_flush_req` 来自哪里？**  
路径：  
- 分支误判路径：`e203_exu_branchslv`（检测预测跳转的实际结果不对）→ `e203_exu_commit` → core → IFU  
- 异常/中断路径：`e203_exu_excp`（ecall/ebreak/非法指令/中断）→ `e203_exu_commit` → core → IFU

**`pipe_flush_add_op1/op2` 来自哪里，为什么传操作数而不是 PC？**  
来自 `e203_exu_commit`（内部由 `branchslv` 或 `excp` 计算好）。传操作数而不是 PC，是为了**复用 IFU 内部已有的 PC 加法器**——ifetch.v 里有一个 `pc_add_op1 + pc_add_op2` 的加法运算，当 flush 时 EXU 直接把它的操作数注入，让 IFU 用自己的加法器算出新的 PC，省掉一个加法器的硬件面积。

**为什么 ack 永远为 1？**  
如果 ack 是组合逻辑（需要 IFU 判断当前是否"能接受" flush），会在 EXU 和 IFU 之间形成一条长组合路径（EXU 计算结果 → `pipe_flush_req` → IFU 判断 → `pipe_flush_ack` → 可能回影响 EXU），时序很紧。  
直接让 ack = 1，用 `dly_flush_r` 处理"虽然 ack 了但当拍 req 通道忙"的情况：

```verilog
// dly_flush_r：延迟一拍处理的 flush 记号
wire dly_flush_set = pipe_flush_req & (~ifu_req_hsked);
//   ↑ flush 来了，但这一拍取指 req 通道握手没成功（忙着）→ 记下来

wire dly_flush_clr = dly_flush_r & ifu_req_hsked;
//   ↑ 有延迟的 flush 挂着，且这拍 req 通道通了 → 清除（flush 已处理）

wire dly_flush_ena = dly_flush_set | dly_flush_clr;
wire dly_flush_nxt = dly_flush_set | (~dly_flush_clr);

sirv_gnrl_dfflr #(1) dly_flush_dfflr (dly_flush_ena, dly_flush_nxt, dly_flush_r, clk, rst_n);

wire dly_pipe_flush_req = dly_flush_r;
wire pipe_flush_req_real = pipe_flush_req | dly_pipe_flush_req;
//   ↑ 无论是当拍的 flush 还是延迟的 flush，都当作"真实的 flush 请求"处理
```

### 2.6 IR 寄存器与功耗优化

IR（Instruction Register）存放从 ITCM 取回的指令，送给 EXU 解码：

```verilog
// IR 有效标志的 set/clr 条件
assign ir_valid_set  = ifu_rsp_hsked           // 有新指令取回
                     & (~pipe_flush_req_real)   // 没有 flush（flush 时丢弃取回的指令）
                     & (~ifu_rsp_need_replay);  // 不需要重放（本设计中恒为0）

assign ir_valid_clr  = ifu_ir_o_hsked          // EXU 接收了这条指令
                     | (pipe_flush_hsked & ir_valid_r);  // flush 来了且 IR 有效 → 清除
```

```verilog
// IR 高低 16bit 分开控制（功耗优化核心）
wire ir_hi_ena = ir_valid_set & minidec_rv32;   // 只有 32bit 指令才更新高 16bit
wire ir_lo_ena = ir_valid_set;                  // 低 16bit 每次指令来都更新

sirv_gnrl_dfflr #(16) ifu_hi_ir_dfflr (ir_hi_ena, ifu_ir_nxt[31:16], ifu_ir_r[31:16], clk, rst_n);
sirv_gnrl_dfflr #(16) ifu_lo_ir_dfflr (ir_lo_ena, ifu_ir_nxt[15: 0], ifu_ir_r[15: 0], clk, rst_n);
```

**为什么要分开控制？**  
如果是 16bit RVC 指令（C扩展压缩指令），高 16bit 是无效数据。每次都更新高 16bit 等于白白让 16 个 DFF 翻转，消耗动态功耗。  
用 `minidec_rv32` 做使能，仅 32bit 指令才更新高半部分，平均节省约 50% 的 IR 寄存器翻转功耗。

**`minidec_rv32` 来自哪里？**  
来自 `e203_ifu_minidec`（本文件实例化的预译码模块），它对当前指令即时解码，判断是 16bit 还是 32bit（`instr[1:0] == 2'b11` 且不是 `111` 类型时为 32bit）。

### 2.7 rs1idx / rs2idx 寄存器

```verilog
wire ir_rs1idx_ena = 
    (minidec_fpu & ir_valid_set & minidec_fpu_rs1en & (~minidec_fpu_rs1fpu)) 
    // ↑ FPU 指令且 rs1 不是浮点寄存器（如果 rs1 是浮点寄存器，不更新整数 rs1idx）
  | ((~minidec_fpu) & ir_valid_set & minidec_rs1en)
    // ↑ 非 FPU 指令，且 rs1 字段有效（有些指令没有 rs1，如 LUI）
  | bpu2rf_rs1_ena;
    // ↑ BPU 要读 xN 寄存器（JALR xN 预测），也要更新 rs1idx
```

**`bpu2rf_rs1_ena` 来自哪里？**  
来自 `e203_ifu_litebpu`（本文件实例化的 BPU 模块）。当 BPU 决定发起"读 xN 寄存器"操作时，拉高这个信号，同时 IFU 也要把 rs1idx 锁进 DFF，供后续逻辑使用。

> **本设计中无 FPU**（`E203_HAS_FPU` 宏未定义），所以 `minidec_fpu = 1'b0`，FPU 分支的代码对应到 0，上面 `ir_rs1idx_ena` 简化为：`(ir_valid_set & minidec_rs1en) | bpu2rf_rs1_ena`。

### 2.8 JALR 依赖检测（CAM 比对）

```verilog
wire ir_empty = ~ir_valid_r;                     // EXU 的 IR 槽是否已空
wire ir_rs1en = dec2ifu_rs1en;                   // IR 中的指令是否读 rs1
wire ir_rden  = dec2ifu_rden;                    // IR 中的指令是否写 rd
wire [`E203_RFIDX_WIDTH-1:0] ir_rdidx = dec2ifu_rdidx;  // IR 中指令的目标寄存器编号

wire jalr_rs1idx_cam_irrdidx = 
    ir_rden                              // IR 指令会写 rd
    & (minidec_jalr_rs1idx == ir_rdidx) // 且 rd 恰好是新 JALR 的 rs1
    & ir_valid_r;                        // 且 IR 槽确实有指令
```

**这是一个微型 CAM（Content Addressable Memory）查表：**  
用当前新取的 JALR 指令的 rs1 索引，和 EXU IR 级指令的 rd 索引做比较。  
如果相等，说明 EXU 还没执行完那条指令就打算用它写的结果，存在 RAW（先写后读）冒险，BPU 不能直接用寄存器堆里的旧值来预测 JALR 目标。

**`dec2ifu_rden`、`dec2ifu_rdidx`、`dec2ifu_rs1en` 来自哪里？**  
路径：`e203_exu_decode`（对 IR 寄存器的指令进行完整解码，产生 `dec_rdwen`、`dec_rdidx`、`dec_rs1en`）→ `e203_exu_disp`（直接透传）→ `e203_exu`（顶层） → `e203_core.v`（命名为 `dec2ifu_*`）→ `e203_ifu.v` → `e203_ifu_ifetch.v`。

### 2.9 乘除 Back-to-Back 融合检测

```verilog
assign ifu_muldiv_b2b_nxt = 
  (
    // 融合条件1：乘法序列 —— 新指令是 MUL，且 EXU IR 级是 MULH/MULHU/MULHSU
    ( minidec_mul & dec2ifu_mulhsu)
    // 融合条件2：除法/余数序列
  | ( minidec_div  & dec2ifu_rem)    // 新是 DIV，旧是 REM
  | ( minidec_rem  & dec2ifu_div)    // 新是 REM，旧是 DIV
  | ( minidec_divu & dec2ifu_remu)   // 无符号版本
  | ( minidec_remu & dec2ifu_divu)
  )
  // 且两条指令的操作数相同（rs1/rs2 索引一样），才能共享计算中间结果
  & (ir_rs1idx_r == ir_rs1idx_nxt)   // 上条指令的 rs1 == 新指令的 rs1
  & (ir_rs2idx_r == ir_rs2idx_nxt)   // 上条指令的 rs2 == 新指令的 rs2
  // 且目标寄存器不和源寄存器冲突（rd != rs1/rs2，否则会覆盖要复用的值）
  & (~(ir_rs1idx_r == ir_rdidx))
  & (~(ir_rs2idx_r == ir_rdidx));
```

**`dec2ifu_mulhsu/div/rem/divu/remu` 来自哪里？**  
来自 `e203_exu_decode`，路径和 `dec2ifu_rden` 相同（见 2.8 节）。  
EXU IR 级已经解码出来的指令类型，IFU 拿来和当前新取指令（来自 minidec）对比，判断是否构成 B2B 对。

**`minidec_mul/div/rem/divu/remu` 来自哪里？**  
来自 `u_e203_ifu_minidec`（本文件中实例化的 minidec 模块），对当前新取回的指令（`ifu_ir_nxt`）做即时预译码。

### 2.10 PC 计算优先级链

```verilog
wire [`E203_PC_SIZE-1:0] pc_add_op1 = 
    pipe_flush_req      ? pipe_flush_add_op1 : // 最高优先级：flush（来自 EXU）
    dly_pipe_flush_req  ? pc_r               : // 延迟 flush：用当前 pc_r（对应 pc_r+0）
    ifetch_replay_req   ? pc_r               : // 重取（本设计恒为0，不触发）
    bjp_req             ? prdt_pc_add_op1    : // BPU 预测跳转
    ifu_reset_req       ? pc_rtvec           : // 复位：从复位向量开始
                          pc_r;               // 默认：顺序递增（当前 PC）

wire [`E203_PC_SIZE-1:0] pc_add_op2 = 
    pipe_flush_req      ? pipe_flush_add_op2 : // flush 操作数2（来自 EXU）
    dly_pipe_flush_req  ? `E203_PC_SIZE'b0   : // dly_flush：+0（保持 pc_r）
    ifetch_replay_req   ? `E203_PC_SIZE'b0   : // replay：+0
    bjp_req             ? prdt_pc_add_op2    : // BPU 预测偏移量
    ifu_reset_req       ? `E203_PC_SIZE'b0   : // 复位：pc_rtvec + 0
                          pc_incr_ofst;        // 顺序：+2（RVC）或 +4（RV32）

// pc_incr_ofst 的计算
wire [2:0] pc_incr_ofst = minidec_rv32 ? 3'd4 : 3'd2;
```

**优先级规则（从高到低）：**

| 优先级 | 条件 | 含义 |
|:---:|---|---|
| 1 | `pipe_flush_req` | EXU 触发的 flush（分支误判/异常），用 EXU 传来的操作数 |
| 2 | `dly_pipe_flush_req` | 延迟了一拍的 flush，重定向到 `pc_r`（当前 PC） |
| 3 | `ifetch_replay_req` | 本设计恒为 0，预留给 replay 功能 |
| 4 | `bjp_req` | BPU 预测跳转，用 litebpu 算出的操作数 |
| 5 | `ifu_reset_req` | 复位请求，跳到 `pc_rtvec` |
| 6 | （默认）| 顺序取指，PC += 2 或 4 |

```verilog
// 最终 PC 值：加法结果，强制最低 bit = 0（RISC-V 所有合法地址 bit[0] = 0）
assign pc_nxt_pre = pc_add_op1 + pc_add_op2;
assign pc_nxt = {pc_nxt_pre[`E203_PC_SIZE-1:1], 1'b0};
//               ↑ 保留高位                       ↑ 最低位强制为 0

// PC 寄存器：只在取指请求握手成功，或 flush 握手成功时更新
wire pc_ena = ifu_req_hsked | pipe_flush_hsked;
sirv_gnrl_dfflr #(`E203_PC_SIZE) pc_dfflr (pc_ena, pc_nxt, pc_r, clk, rst_n);
```

### 2.11 取指请求有效条件

```verilog
// 以下四条任一满足时，有新取指请求
wire ifu_new_req = (~bpu_wait)        // BPU 不要求等待（无 JALR 依赖）
                 & (~ifu_halt_req)    // EXU 没有发出 halt 请求
                 & (~reset_flag_r)    // 不在复位瞬间
                 & (~ifu_rsp_need_replay); // 不需要重取（本设计恒为0）

wire ifu_req_valid_pre = ifu_new_req          // 正常新请求
                       | ifu_reset_req        // 复位后取第一条
                       | pipe_flush_req_real  // flush 后重定向
                       | ifetch_replay_req;   // replay（恒为0）

// 还需要满足"无在途请求"或"在途请求当拍收到响应"
wire new_req_condi = (~out_flag_r) | out_flag_clr;

assign ifu_req_valid = ifu_req_valid_pre & new_req_condi;
```

**为什么一次最多只允许一条在途请求？**  
e203 的 ITCM 是单端口 SRAM，同时只能服务一个访问者。IFU 占用 ITCM 时，LSU 就必须等待；如果 IFU 发了两条请求，管理起来复杂，得不偿失。用 `out_flag_r` 这 1 个 bit 追踪"有无在途"，极简且正确。

---

## 精读三：`e203_ifu_minidec.v`（预译码器）

**文件位置**：`core/e203_ifu_minidec.v`  
**职责**：在指令进入 IR 寄存器前，快速判断指令类型

### 3.1 预译码器的本质

```verilog
module e203_ifu_minidec(...);

  // 直接实例化完整的 EXU 解码器！
  e203_exu_decode u_e203_exu_decode(
    .i_instr      (instr),          // 输入指令
    .i_pc         (`E203_PC_SIZE'b0), // PC 给0（minidec 不需要 PC）
    .i_prdt_taken (1'b0),            // 预测信息给0
    .i_muldiv_b2b (1'b0),            // b2b 给0

    // 只接出需要的信号：
    .dec_rs1en    (dec_rs1en),  // 是否读 rs1
    .dec_rs2en    (dec_rs2en),  // 是否读 rs2
    .dec_rs1idx   (dec_rs1idx), // rs1 寄存器编号
    .dec_rs2idx   (dec_rs2idx), // rs2 寄存器编号
    .dec_rv32     (dec_rv32),   // 是 32bit 指令？
    .dec_bjp      (dec_bjp),    // 是跳转类指令？
    .dec_jal      (dec_jal),    // 是 JAL？
    .dec_jalr     (dec_jalr),   // 是 JALR？
    .dec_bxx      (dec_bxx),    // 是 Bxx 条件分支？
    .dec_bjp_imm  (dec_bjp_imm), // 跳转偏移量
    .dec_jalr_rs1idx (dec_jalr_rs1idx), // JALR 的 rs1 索引
    .dec_mul  (dec_mul), .dec_div (dec_div), // 乘除法标志

    // 其余输出悬空（不用）：
    .dec_info(),    // 完整解码控制信号包（EXU 需要，IFU 不需要）
    .dec_imm (),    // 立即数（EXU 需要，IFU 不需要）
    .dec_rdwen(),   // rd 写使能（minidec 不需要）
    // ...
  );
endmodule
```

**为什么 IFU 要用 EXU 的 decode 模块做预译码？**

1. **零维护成本**：如果单独写一套"轻量预译码"，指令集扩展时要改两个地方。直接复用 `e203_exu_decode`，只需维护一份代码。
2. **功能复盖完整**：minidec 只关心几个输出（是否 rv32、是否分支、操作数索引），把其余输出悬空（接 `()`），综合器会自动剪掉不需要的逻辑。
3. **代价**：decode 模块在 ifetch 的关键路径上再走一遍，会增加延迟。但由于 minidec 的结果只用于下一拍（IR 写使能、BPU 预测），时序通常能满足。

### 3.2 在文件中的使用位置

`minidec` 在 `e203_ifu_ifetch.v` 中实例化，对象是**刚刚取回但还没存入 IR 的指令**（`ifu_ir_nxt`，即 `ifu_rsp_instr`）：

```verilog
wire [`E203_INSTR_SIZE-1:0] ifu_ir_nxt = ifu_rsp_instr; // 取回的指令，即将存入 IR

e203_ifu_minidec u_e203_ifu_minidec (
    .instr       (ifu_ir_nxt),     // 对即将存入 IR 的指令做预译码
    .dec_rv32    (minidec_rv32),   // → 控制 ir_hi_ena（高16bit 写使能）
    .dec_bjp     (minidec_bjp),
    .dec_jal     (minidec_jal),    // → 传给 litebpu 做分支预测
    .dec_jalr    (minidec_jalr),
    .dec_bxx     (minidec_bxx),
    .dec_bjp_imm (minidec_bjp_imm),
    .dec_jalr_rs1idx (minidec_jalr_rs1idx), // → JALR 依赖检测（2.8节）
    .dec_mul     (minidec_mul),    // → B2B 融合检测（2.9节）
    // ...
);
```

---

## 精读四：`e203_ifu_litebpu.v`（轻量级分支预测）

**文件位置**：`core/e203_ifu_litebpu.v`  
**职责**：对跳转类指令做静态预测，计算预测目标 PC

### 4.1 静态预测策略

```verilog
// 预测"是否跳转"
assign prdt_taken = (dec_jal                          // JAL：无条件跳，永远预测跳
                   | dec_jalr                          // JALR：无条件跳，永远预测跳
                   | (dec_bxx & dec_bjp_imm[`E203_XLEN-1]));
//                    ↑ Bxx 条件分支：只有偏移量为负（最高位=1）才预测跳
```

**为什么 Bxx 负偏移预测跳？**  
这是经典的"向后分支预测跳"（backward branch prediction taken）策略：  
- 向后跳转（目标地址 < 当前 PC）通常是**循环回跳**，命中率极高（循环体会反复执行）
- 向前跳转（目标地址 > 当前 PC）通常是 `if` 语句的跳过，命中率偏低  
RISC-V Bxx 指令的立即数是有符号偏移量，负数 ↔ 向后跳，用 `dec_bjp_imm[XLEN-1]`（符号位）判断就行，无需任何历史统计。

### 4.2 JALR x1（函数返回）的依赖检测

```verilog
wire dec_jalr_rs1x1 = (dec_jalr_rs1idx == `E203_RFIDX_WIDTH'd1); // rs1 是 x1？

wire jalr_rs1x1_dep = dec_i_valid             // 当前有效指令
                    & dec_jalr                 // 是 JALR
                    & dec_jalr_rs1x1          // rs1 = x1（ra）
                    & ((~oitf_empty) |         // 情况1：长指令 FIFO 非空（可能有指令在写 x1）
                       (jalr_rs1idx_cam_irrdidx)); // 情况2：EXU IR 中的指令就要写 x1
```

**`oitf_empty` 来自哪里？**  
路径：`e203_exu_oitf`（OITF 为空时拉高）→ `e203_exu`（顶层）→ `e203_core.v`（命名 `oitf_empty`）→ `e203_ifu.v` → `e203_ifu_ifetch.v` → `e203_ifu_litebpu.v`。

**`jalr_rs1idx_cam_irrdidx` 来自哪里？**  
来自 `e203_ifu_ifetch.v`（内部 CAM 比较，见第 2.8 节），传入 litebpu 作为"IR 阶段是否有 x1 写冲突"的判断。

**依赖时怎么处理？**

```verilog
assign bpu_wait = jalr_rs1x1_dep   // x1 有依赖 → 等待
                | jalr_rs1xn_dep   // xN 有依赖 → 等待
                | rs1xn_rdrf_set;  // 正在"等一拍读寄存器"的状态 → 等待
```

`bpu_wait = 1` 时，`ifu_new_req` 变为 0，IFU 停止取新指令，直到依赖解除。

### 4.3 JALR xN（任意寄存器）的"等一拍读寄存器"

```verilog
wire dec_jalr_rs1xn = (~dec_jalr_rs1x0) & (~dec_jalr_rs1x1); // rs1 不是 x0 也不是 x1

wire jalr_rs1xn_dep = dec_i_valid & dec_jalr & dec_jalr_rs1xn
                    & ((~oitf_empty) | (~ir_empty));
// 只要 OITF 非空，或 EXU IR 中还有指令，就认为 xN 可能有冒险

// 一种特殊情况：依赖只来自 IR 级（OITF 已空），但 IR 指令正在被清除，
// 或 IR 指令不用 rs1 —— 这种情况可以绕过等待
wire jalr_rs1xn_dep_ir_clr = (jalr_rs1xn_dep & oitf_empty & (~ir_empty))
                            & (ir_valid_clr | (~ir_rs1en));

// "等一拍"状态寄存器：第一拍无依赖时进入，第二拍自动清除（读寄存器数据已稳定）
wire rs1xn_rdrf_r;
wire rs1xn_rdrf_set = (~rs1xn_rdrf_r)        // 还没在等
                    & dec_i_valid & dec_jalr & dec_jalr_rs1xn
                    & ((~jalr_rs1xn_dep) | jalr_rs1xn_dep_ir_clr);
wire rs1xn_rdrf_clr = rs1xn_rdrf_r;  // 等了一拍即清除

assign bpu2rf_rs1_ena = rs1xn_rdrf_set;  // 通知 regfile：下一拍请输出 xN 的值
```

**`bpu2rf_rs1_ena` 的作用路径：**  
`litebpu` → `ifetch`（连到 `bpu2rf_rs1_ena`）→ `core`（命名 `rf2ifu_rs1_ena`，不存在直接路径，实际采用 `ifu` 顶层端口）→ `regfile` 的读端口使能。  
regfile 的 `x1_r` 单独连出（`rf2ifu_x1`），其余寄存器通过通用读口（`rf2ifu_rs1`）读出。

### 4.4 预测目标 PC 的操作数

```verilog
assign prdt_pc_add_op1 = 
    (dec_bxx | dec_jal)       ? pc[`E203_PC_SIZE-1:0]       // JAL/Bxx：目标 = 当前PC + imm
    : (dec_jalr & dec_jalr_rs1x0) ? `E203_PC_SIZE'b0        // JALR x0：目标 = 0 + imm
    : (dec_jalr & dec_jalr_rs1x1) ? rf2bpu_x1[`E203_PC_SIZE-1:0]  // JALR x1：目标 = x1 + imm
    :                               rf2bpu_rs1[`E203_PC_SIZE-1:0]; // JALR xN：目标 = xN + imm

assign prdt_pc_add_op2 = dec_bjp_imm[`E203_PC_SIZE-1:0]; // 所有情况 op2 都是 imm
```

**`rf2bpu_x1` / `rf2bpu_rs1` 来自哪里？**  
路径：`e203_exu_regfile`（寄存器堆）→ `e203_exu`（顶层，命名 `rf2ifu_x1`/`rf2ifu_rs1`）→ `e203_core.v` → `e203_ifu.v`（端口 `rf2ifu_x1`/`rf2ifu_rs1`）→ `e203_ifu_ifetch.v`（直连 litebpu 的 `rf2bpu_x1`/`rf2bpu_rs1`）。  
x1 单独引出一根线，是因为函数返回（`ret` = `jalr x0, x1, 0`）极其频繁，直连 x1 不需要任何选择逻辑。

---

## 精读五：`e203_ifu_ift2icb.v`（取指→ICB 协议转换）

**文件位置**：`core/e203_ifu_ift2icb.v`  
**职责**：将 ifetch 的简化取指请求转成 ICB（Internal Chip Bus）总线事务，并处理 64bit SRAM 的地址对齐/拼接问题

### 5.1 为什么需要旁路缓冲（bypbuf）

```verilog
// 在模块的最开头，用 sirv_gnrl_bypbuf 包裹 rsp 通道
sirv_gnrl_bypbuf # (.DP(1), .DW(`E203_INSTR_SIZE+1))
  u_e203_ifetch_rsp_bypbuf(
    .i_vld (i_ifu_rsp_valid), .i_rdy (i_ifu_rsp_ready), // 内部侧
    .o_vld (ifu_rsp_valid),   .o_rdy (ifu_rsp_ready),   // 外部侧（面向 ifetch）
    .i_dat (ifu_rsp_bypbuf_i_data),
    .o_dat (ifu_rsp_bypbuf_o_data),
    ...
  );
```

**为什么要在 rsp 通道加一个缓冲？**  
源码注释原文（翻译）：

> "IFU rsp 的 ready 信号从 EXU 级生成，包含若干时序关键路径（如 ECC 错误检查等），会反压到这里的 rsp 通道。如果没有这个缓冲，ift2icb 的 rsp 通道可能因等待 IR 级被清而卡住；而 EXU 访问 BIU 或 ITCM 时，又在等 IFU 让出总线——这就死锁了。"

`sirv_gnrl_bypbuf` 是一个深度为 1 的旁路缓冲，它能接收响应数据存起来，即使下游（EXU）还没准备好，上游（ITCM）也能完成这次传输，从而打破循环依赖。

### 5.2 地址路由：ITCM vs 系统总线

```verilog
// 当前取指 PC 是否落在 ITCM 地址范围内？
wire ifu_req_pc2itcm = (ifu_req_pc[`E203_ITCM_BASE_REGION] 
                        == itcm_region_indic[`E203_ITCM_BASE_REGION]); 
```

**`itcm_region_indic` 来自哪里？**  
路径：`e203_itcm_ctrl.v`（ITCM 控制器）向外广播自己所在的地址区域，经 `e203_core.v` 传给 IFU 的 ift2icb 模块。只要 PC 的高几位和 ITCM 基地址的高几位相同，就路由到 ITCM。

```verilog
// 否则路由到系统总线
wire ifu_req_pc2mem = 1'b1
    `ifdef E203_HAS_ITCM
    & ~(ifu_req_pc2itcm)  // 不在 ITCM 范围的才走系统总线
    `endif
    ;
```

### 5.3 Lane 边界检测

ITCM 是 64bit 宽的 SRAM（一次读出 64bit = 8 字节 = 一个 Lane）：

```verilog
// 取指 PC 是否跨 64bit Lane 边界？
// 64bit ITCM：PC[2:1] == 2'b11 时，取出的 32bit 指令跨越了 64bit 对齐边界
wire ifu_req_lane_cross = (ifu_req_pc2itcm & (ifu_req_pc[2:1] == 2'b11));
//                         ↑ 例如 PC=0x06（二进制 0b0110），取 4 字节，
//                           会覆盖地址 6、7、8、9，而 8 是下一个 64bit 边界

// 取指 PC 是否在 Lane 的起始位置？
wire ifu_req_lane_begin = (ifu_req_pc2itcm & (ifu_req_pc[2:1] == 2'b00));
// PC[2:1]==0 时，取指从 64bit Lane 的最开头开始

// 上次 ITCM 的读出值是否还保持着（没有被新的访问覆盖）？
wire ifu_req_lane_holdup = (ifu_req_pc2itcm & ifu2itcm_holdup & (~itcm_nohold));
```

**`ifu2itcm_holdup` 来自哪里？**  
来自 `e203_itcm_ctrl.v`，当 ITCM 上次被 IFU 访问后、没有被 LSU 等其他主机再次访问时，保持为 1，表示"SRAM 的输出值还是上次 IFU 读的那个，还有效"。

**`itcm_nohold` 来自哪里？**  
来自 `e203_exu.v`（顶层），经 `e203_core.v` 传来，表示"强制不使用 holdup 状态"（调试或特殊情况下使用）。

### 5.4 四状态取指状态机

由于 32bit 指令可能跨 64bit Lane，有时需要两次 SRAM 访问。用一个 4 状态 FSM 管理：

```
IDLE ──(新请求)──▶ 1ST（发第1次 ICB CMD）
         │
         ├──(只需1次 or 0次访问)──▶ 直接到 IDLE 或下一个 1ST
         │
         └──(需要2次访问)──┬──(ICB CMD ready)──▶ 2ND（发第2次 CMD）
                          └──(ICB CMD not ready)──▶ WAIT2ND（等总线）
```

| 状态 | 含义 |
|---|---|
| `ICB_STATE_IDLE` | 空闲，无在途请求 |
| `ICB_STATE_1ST` | 已发第 1 次 ICB 请求，等响应 |
| `ICB_STATE_WAIT2ND` | 需要第 2 次请求，但总线暂时不可用 |
| `ICB_STATE_2ND` | 已发第 2 次 ICB 请求，等响应 |

**三种情况的路径：**

1. **不跨 Lane 且 holdup 有效**（`req_need_0uop`）：直接用上次 SRAM 的输出，不发 ICB CMD，产生"假响应"（`holdup_gen_fake_rsp_valid = 1`）
2. **不跨 Lane 或跨 Lane 但 holdup 有效**（1 uop）：发一次 ICB CMD
3. **跨 Lane 且 holdup 无效**（`req_need_2uop`）：发两次 ICB CMD，第一次响应存入 leftover buffer，第二次响应和 leftover 拼接

### 5.5 Leftover Buffer（16bit 拼接缓冲）

```verilog
wire [15:0] leftover_r;  // 16bit 缓冲，存放第 1 次 ICB 响应的高 16bit

// 最终指令拼接
assign i_ifu_rsp_instr = 
    ({32{rsp_instr_sel_leftover}} & {ifu_icb_rsp_rdata_lsb16, leftover_r})
    // ↑ 跨Lane情况：{第2次响应的低16bit, leftover的16bit} = 完整32bit指令
  | ({32{rsp_instr_sel_icb_rsp }} & ifu_icb_rsp_instr);
    // ↑ 非跨Lane情况：直接用对齐后的 ICB 响应数据
```

**具体例子（64bit ITCM，指令地址 = 0x06）：**

```
ITCM 64bit Lane 0: [地址 0x00..0x07]
                    ┌──────┬──────┬──────┬──────┐
                    │byte7 │byte6 │byte5 │byte4 │ ← 高32bit（来自第2次/同一次读）
                    │byte3 │byte2 │byte1 │byte0 │ ← 低32bit
                    └──────┴──────┴──────┴──────┘

ITCM 64bit Lane 1: [地址 0x08..0x0F]
                    ┌──────┬──────┬──────┬──────┐
                    │byte15│byte14│byte13│byte12│
                    │byte11│byte10│byte9 │byte8 │
                    └──────┴──────┴──────┴──────┘

PC = 0x06：指令占 0x06、0x07（来自 Lane 0 高位）和 0x08、0x09（来自 Lane 1 低位）
           → 需要 2 次 SRAM 读
           → 第1次读 Lane 0，取 byte7:byte6 → 存入 leftover_r
           → 第2次读 Lane 1，取 byte9:byte8 → 和 leftover 拼接
           → 最终指令 = {byte9, byte8, byte7, byte6}（低位在低地址，小端）
```

---

## 精读小结

| 文件 | 行数（约）| 核心机制 | 一句话亮点 |
|---|:---:|---|---|
| `e203_ifu.v` | 250 | 两子模块胶水层 | 内部 req/rsp 5 线协议将逻辑与通信彻底解耦 |
| `e203_ifu_ifetch.v` | 600 | PC 生成 / IR 寄存器 / flush / B2B / JALR依赖 | `pipe_flush_ack=1` 截断关键路径；高低 16bit 分开更新省功耗 |
| `e203_ifu_minidec.v` | 110 | 复用 `exu_decode` | 零额外维护成本做预译码；未用输出综合器自动剪除 |
| `e203_ifu_litebpu.v` | 110 | 静态分支预测 | 负偏移 Bxx 预测为跳，JALR x1 直读 ra，单周期预测无需 BTB/BHT |
| `e203_ifu_ift2icb.v` | 720 | 4 状态机 + leftover 缓冲 | bypbuf 打破 EXU↔IFU 死锁；跨 Lane 两次 ICB 自动拼接 |

**跨模块信号总览（IFU 侧）：**

| 信号 | 来源（最终产生模块）| 用于 IFU 哪个功能 |
|---|---|---|
| `pc_rtvec` | 外部引脚/顶层配置 | 复位后第一条指令的地址 |
| `pipe_flush_req/add_op1/2` | `e203_exu_branchslv` 或 `e203_exu_excp` | 流水线冲刷 + 目标 PC 计算 |
| `ifu_halt_req` | `e203_exu_excp`（WFI/debug） | IFU 暂停取指 |
| `oitf_empty` | `e203_exu_oitf` | BPU 判断 JALR x1/xN 的 RAW 依赖 |
| `rf2ifu_x1/rs1` | `e203_exu_regfile` | BPU 预测 JALR 目标地址 |
| `dec2ifu_rden/rs1en/rdidx` | `e203_exu_decode`（EXU IR 级） | BPU CAM 冒险检测 |
| `dec2ifu_mul/div/rem…` | `e203_exu_decode`（EXU IR 级） | 乘除 B2B 融合检测 |
| `itcm_region_indic` | `e203_itcm_ctrl` | ift2icb 地址路由（ITCM vs BIU）|
| `ifu2itcm_holdup` | `e203_itcm_ctrl` | ift2icb 判断是否可复用上次读出值 |
| `itcm_nohold` | `e203_exu`（调试相关）| 强制不复用 holdup 值 |
