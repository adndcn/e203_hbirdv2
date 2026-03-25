# 附录：快速参考手册

> 本附录提供 e203 RTL 的快速查询资料，适合在阅读源码时随时翻阅。

---

## A. config.v 参数完整参考表

`config.v` 是所有可配置参数的"总开关"，修改这里的宏定义就能改变芯片的功能和规格。

### A.1 基础架构参数

| 宏定义 | 默认值/状态 | 含义 | 影响范围 |
|--------|------------|------|----------|
| `E203_CFG_ADDR_SIZE_IS_32` | **已定义** | 地址总线宽 32 位 | PC、指针宽度 |
| `E203_CFG_REGNUM_IS_32` | **已定义** | 32 个通用寄存器（RV32 标准） | regfile 大小 |
| `E203_CFG_XLEN_IS_32` | **已定义** | 数据位宽 32 位 | 所有数据路径 |
| `E203_CFG_REGFILE_LATCH_BASED` | 未定义 | 寄存器堆用 latch 实现（面积小） | regfile 实现方式 |

### A.2 ITCM 参数

| 宏定义 | 默认值 | 含义 | 计算说明 |
|--------|--------|------|----------|
| `E203_CFG_HAS_ITCM` | **已定义** | 启用 ITCM | — |
| `E203_CFG_ITCM_ADDR_WIDTH` | **16** | ITCM 地址宽度 | 容量 = 2^16 = 64KB |
| `E203_ITCM_ADDR_BASE` | `0x8000_0000` | ITCM 起始地址 | — |
| `E203_ITCM_DATA_WIDTH` | **64** | ITCM 数据位宽（8字节/拍） | 每次读2条32位指令 |
| `E203_ITCM_RAM_DP` | `2^(16-3)=8192` | ITCM SRAM 深度（行数）| 深度 = 容量/位宽 = 64K×8/64 |
| `E203_ITCM_RAM_AW` | **13** | ITCM SRAM 地址位宽 | = addr_width - 3 |

> 修改 ITCM 大小只需改 `E203_CFG_ITCM_ADDR_WIDTH`：
> - 16 → 64KB，17 → 128KB，18 → 256KB，20 → 1MB

### A.3 DTCM 参数

| 宏定义 | 默认值 | 含义 | 计算说明 |
|--------|--------|------|----------|
| `E203_CFG_HAS_DTCM` | **已定义** | 启用 DTCM | — |
| `E203_CFG_DTCM_ADDR_WIDTH` | **16** | DTCM 地址宽度 | 容量 = 2^16 = 64KB |
| `E203_DTCM_ADDR_BASE` | `0x9000_0000` | DTCM 起始地址 | — |
| `E203_DTCM_DATA_WIDTH` | **32** | DTCM 数据位宽（4字节/拍）| — |
| `E203_DTCM_RAM_DP` | `2^(16-2)=16384` | DTCM SRAM 深度 | 深度 = 64K×8/32 |
| `E203_DTCM_RAM_AW` | **14** | DTCM SRAM 地址位宽 | = addr_width - 2 |

### A.4 功能开关参数

| 宏定义 | 默认值/状态 | 含义 |
|--------|------------|------|
| `E203_CFG_HAS_ECC` | **已定义** | 启用 ECC（Hamming SEC-DED 纠错码） |
| `E203_CFG_HAS_NICE` | **已定义** | 启用 NICE 协处理器接口 |
| `E203_CFG_SUPPORT_SHARE_MULDIV` | **已定义** | 启用 MUL/DIV 指令（共享实现） |
| `E203_CFG_SUPPORT_AMO` | **已定义** | 启用原子操作（A 扩展）|
| `E203_CFG_SUPPORT_MCYCLE_MINSTRET` | **已定义** | 启用 mcycle/minstret 性能计数器 |
| `E203_CFG_DEBUG_HAS_JTAG` | **已定义** | 启用 JTAG 调试接口 |
| `E203_CFG_IRQ_NEED_SYNC` | **已定义** | 中断信号需要同步（异步中断）|

### A.5 OITF 参数

| 宏定义 | 默认值 | 含义 |
|--------|--------|------|
| `E203_CFG_OITF_DEPTH_IS_2` | **已定义** | OITF 深度 = 2 |
| `E203_OITF_DEPTH` | **2** | 实际深度数值 |
| `E203_ITAG_WIDTH` | **1** | in-flight 标记位宽（深度2时=1位）|
| `E203_ASYNC_FF_LEVELS` | **2** | 异步同步器级数（复位和中断同步用）|

### A.6 地址空间参数

| 宏定义 | 默认值 | 对应地址范围 |
|--------|--------|------------|
| `E203_ITCM_ADDR_BASE` | `0x8000_0000` | ITCM: 0x8000_0000 ~ 0x8000_FFFF |
| `E203_DTCM_ADDR_BASE` | `0x9000_0000` | DTCM: 0x9000_0000 ~ 0x9000_FFFF |
| `E203_CLINT_ADDR_BASE` | `0x0200_0000` | CLINT: 0x0200_0000 ~ 0x0200_FFFF |
| `E203_PLIC_ADDR_BASE` | `0x0C00_0000` | PLIC: 0x0C00_0000 ~ 0x0CFF_FFFF |
| `E203_PPI_ADDR_BASE` | `0x1000_0000` | 外设: 0x1000_0000 ~ 0x1FFF_FFFF |
| `E203_FIO_ADDR_BASE` | `0xF000_0000` | 快速 IO: 0xF000_0000 ~ 0xFFFF_FFFF |

---

## B. 信号命名规范

e203 的信号命名遵循严格的统一规范，理解规范后读代码速度会大大提升。

### B.1 ICB 总线信号命名

```
格式：<发送方>2<接收方>_icb_<cmd|rsp>_<字段>

示例：
  ifu2itcm_icb_cmd_valid   = IFU→ITCM 的命令通道 valid
  ifu2itcm_icb_cmd_addr    = IFU→ITCM 的命令地址
  ifu2itcm_icb_rsp_rdata  = ITCM→IFU 的响应读数据
  lsu2dtcm_icb_cmd_wmask  = LSU→DTCM 的写字节使能
```

### B.2 握手信号规范

| 后缀 | 含义 | 方向 |
|------|------|------|
| `_valid` | 数据有效（发送方控制）| 发送方→接收方 |
| `_ready` | 可以接收（接收方控制）| 接收方→发送方 |
| `_bits` 或具体字段 | 数据内容 | 发送方→接收方 |

握手规则：**valid AND ready = 传输成功（握手）**

### B.3 其他常见后缀

| 后缀 | 含义 |
|------|------|
| `_r` | 寄存器输出（DFF 的 Q 端）|
| `_nxt` | 寄存器的下一值（D 端）|
| `_ena` | 使能信号（enable）|
| `_req` / `_ack` | 请求/应答对 |
| `_vld` | valid 的缩写 |
| `_rdy` | ready 的缩写 |
| `_i` | 输入（input）|
| `_o` | 输出（output）|
| `_n` | 低有效（active low）|
| `_w` | 组合逻辑线（wire）|

### B.4 模块内部信号命名示例

```verilog
// e203_exu_oitf.v 中的典型名称：
  oitf_empty         // OITF 为空标志
  oitf_full          // OITF 满标志
  alc_ptr_r          // 分配指针（寄存器）
  ret_ptr_r          // 释放指针（寄存器）
  oitf_ret_ena       // OITF 释放使能
  dis_ready          // 分发就绪
```

---

## C. RTL 文件功能速查表

### C.1 核心流水线模块

| 文件 | 隶属层次 | 功能一句话描述 |
|------|---------|--------------|
| `e203_ifu.v` | IFU 顶层 | IFU 的模块互连包装 |
| `e203_ifu_ifetch.v` | IFU | PC 管理、取指总控，IR 寄存器 |
| `e203_ifu_ift2icb.v` | IFU | IR 到 ICB 命令的转换，地址对齐，leftover 缓冲 |
| `e203_ifu_minidec.v` | IFU | 指令预解码（仅判断16/32位和跳转，用于 liteBPU）|
| `e203_ifu_litebpu.v` | IFU | 轻量分支预测器（静态规则）|
| `e203_exu.v` | EXU 顶层 | EXU 的模块互连包装 |
| `e203_exu_decode.v` | EXU | 指令完整解码 → dec_info 总线 |
| `e203_exu_regfile.v` | EXU | 32×32 通用寄存器堆（2 读 1 写）|
| `e203_exu_disp.v` | EXU | 指令分发（RAW/WAW 相关性检测，OITF 管理）|
| `e203_exu_oitf.v` | EXU | OITF FIFO（追踪在途长延迟指令）|
| `e203_exu_alu.v` | EXU | ALU 顶层（选择子单元，汇集结果）|
| `e203_exu_alu_rglr.v` | EXU/ALU | 普通 ALU 运算（ADD/AND/OR/SHI等）|
| `e203_exu_alu_bjp.v` | EXU/ALU | 分支/跳转（BEQ/JAL/JALR 等）|
| `e203_exu_alu_lsuagu.v` | EXU/ALU | 存取地址生成（AGU，计算 rs1+offset）|
| `e203_exu_alu_csrctrl.v` | EXU/ALU | CSR 读写指令 |
| `e203_exu_alu_muldiv.v` | EXU/ALU | 乘除法（MUL/DIV/REM 等）|
| `e203_exu_alu_dpath.v` | EXU/ALU | ALU 数据路径（加减器、移位器、比较器）|
| `e203_exu_wbck.v` | EXU | 写回仲裁（短流水优先于长流水）|
| `e203_exu_commit.v` | EXU | 提交/退休（分支跳转冲刷、异常/中断处理）|
| `e203_exu_branchslv.v` | EXU | 分支解析（比较实际结果 vs 预测结果）|
| `e203_exu_csr.v` | EXU | CSR 寄存器堆（mstatus/mtvec/mepc等）|
| `e203_exu_excp.v` | EXU | 异常生成（非法指令、地址对齐等）|
| `e203_exu_longpwbck.v` | EXU | 长流水写回（LSU/MULDIV 完成后）|
| `e203_exu_nice.v` | EXU | NICE 协处理器接口 |
| `e203_lsu.v` | LSU 顶层 | LSU 模块包装 |
| `e203_lsu_ctrl.v` | LSU | 加载/存储控制（地址路由、符号扩展、wmask）|

### C.2 存储子系统

| 文件 | 功能 |
|------|------|
| `e203_itcm_ctrl.v` | ITCM 控制器（ICB→SRAM接口，IFU/LSU 仲裁）|
| `e203_dtcm_ctrl.v` | DTCM 控制器（ICB→SRAM接口，ECC处理）|
| `e203_itcm_ram.v` | ITCM SRAM 封装（调用 sirv_gnrl_ram）|
| `e203_dtcm_ram.v` | DTCM SRAM 封装 |
| `e203_srams.v` | ITCM+DTCM SRAM 顶层（含时钟门控）|
| `sirv_gnrl_ram.v` | 通用 SRAM 模型（仿真用行为模型）|
| `sirv_sim_ram.v` | 仿真 SRAM（$readmemh 加载初始值）|

### C.3 系统基础设施

| 文件 | 功能 |
|------|------|
| `e203_clkgate.v` | 时钟门控单元（latch+AND，防毛刺）|
| `e203_reset_ctrl.v` | 复位控制（异步复位同步释放）|
| `e203_biu.v` | 总线接口单元（IFU/LSU 的外部 ICB 出口）|
| `e203_clk_ctrl.v` | 时钟控制（WFI 停核）|
| `e203_irq_sync.v` | 中断信号同步器（跨时钟域） |
| `e203_extend_csr.v` | 扩展 CSR（厂商自定义 CSR）|

### C.4 通用 IP 模块（general/）

| 文件 | 功能 |
|------|------|
| `sirv_gnrl_dffs.v` | **所有 DFF 类型定义**（最基础模块！）|
| `sirv_gnrl_bufs.v` | 缓冲器（带反压的 FIFO 缓冲）|
| `sirv_gnrl_icbs.v` | ICB 工具函数（ICB 握手状态机）|
| `sirv_gnrl_xchecker.v` | 断言检查（仿真时检测 X 值传播）|
| `sirv_1cyc_sram_ctrl.v` | 1周期 SRAM 控制器 |
| `sirv_sram_icb_ctrl.v` | SRAM ICB 控制（通用）|

### C.5 调试系统（debug/）

| 文件 | 功能 |
|------|------|
| `sirv_jtag_dtm.v` | JTAG DTM（TAP 状态机，串行→并行转换）|
| `sirv_debug_module.v` | 调试模块（halt/resume/读写寄存器/内存）|
| `sirv_debug_rom.v` | Debug ROM（固化的 halt/resume 序列）|
| `sirv_debug_ram.v` | Debug RAM（临时抽象命令程序）|
| `sirv_debug_csr.v` | 调试 CSR（dcsr/dpc/dscratch）|

### C.6 外设（perips/）

| 文件 | 功能 |
|------|------|
| `sirv_clint_top.v` | CLINT（mtime/mtimecmp/msip）|
| `sirv_clint.v` | CLINT 核心逻辑 |
| `sirv_plic_top.v` | PLIC 顶层（16路外设中断仲裁）|
| `sirv_plic_man.v` | PLIC 管理器（优先级/使能/阈值/claim）|
| `sirv_aon_top.v` | AON 域顶层（RTC+PMU+WDOG）|
| `sirv_rtc.v` | 实时时钟（RTC）|
| `sirv_pmu_core.v` | 电源管理单元（PMU）|
| `sirv_flash_qspi_top.v` | QSPI Flash 控制器 |
| `sirv_mrom_top.v` | Mask ROM（Boot ROM）|

### C.7 总线互连（fab/）

| 文件 | 功能 |
|------|------|
| `sirv_icb1to2_bus.v` | ICB 1→2 路由器（地址解码分发）|
| `sirv_icb1to8_bus.v` | ICB 1→8 路由器 |
| `sirv_icb1to16_bus.v` | ICB 1→16 路由器 |

---

## D. sirv_gnrl_dffs.v 触发器类型速查

这是整个 e203 最底层的基础模块，所有状态都由这里的 DFF 单元构成：

| 模块名 | 含义 | 使用场景 |
|--------|------|----------|
| `sirv_gnrl_dff` | 普通 D 触发器，纯时钟同步，无复位 | 极少用 |
| `sirv_gnrl_dffr` | 带**同步**复位 DFF，reset→0 | 一般寄存器 |
| `sirv_gnrl_dfflr` | 带**同步**复位+**使能**（load enable）| 带条件更新的寄存器 |
| `sirv_gnrl_dfflrs` | 带**同步**复位+使能，复位值可设（S=set）| 复位到非零值 |
| `sirv_gnrl_dffrs` | 带**异步**复位 DFF | 复位控制域（慎用）|

```verilog
// 最常用的一种：sirv_gnrl_dfflrs
// l = load-enable, r = reset, s = 可设置复位值

module sirv_gnrl_dfflrs # (parameter DW = 32) (
  input  lden,       // 使能：1 才更新
  input  [DW-1:0] dnxt,  // 下一值（D 端）
  output [DW-1:0] qout,  // 输出（Q 端）
  input  clk,
  input  rst_n       // 复位（低有效）
);
reg [DW-1:0] qout_r;
always @(posedge clk or negedge rst_n)
  if (!rst_n) qout_r <= {DW{1'b1}};   // 复位到全1
  else if (lden) qout_r <= dnxt;       // 使能时更新
assign qout = qout_r;
```

---

## E. ICB 协议速查

```
cmd 通道（主机→从机）：
  _valid   锁存好了，主机说"我有数据"
  _ready   从机说"我能接收"
  _addr    32位访问地址
  _read    1=读，0=写
  _wdata   写数据（32位）
  _wmask   写字节使能（4位，每位对应1字节）
  _usr     用户附加信息（可选，如 LSU 的 itag/sign）

rsp 通道（从机→主机）：
  _valid   从机说"结果准备好了"
  _ready   主机说"我能接收结果"
  _rdata   读回的数据（32位）
  _err     访问出错标志

握手完成 = valid & ready = 1（同一拍）
cmd 和 rsp 可以不在同一拍（流水线）
```

---

## F. 常用 CSR 寄存器速查

| CSR 地址 | 名称 | 功能 |
|---------|------|------|
| `0x300` | `mstatus` | 机器状态（MIE=全局中断使能）|
| `0x301` | `misa` | ISA 支持声明（RV32IMAC）|
| `0x304` | `mie` | 中断使能掩码（MTIE/MSIE/MEIE）|
| `0x305` | `mtvec` | 中断向量基址 |
| `0x340` | `mscratch` | 特权切换临时寄存器 |
| `0x341` | `mepc` | 异常返回地址（中断/异常发生时保存 PC）|
| `0x342` | `mcause` | 中断/异常原因码 |
| `0x343` | `mtval` | 附加信息（如缺地址、非法指令码）|
| `0x344` | `mip` | 中断等待标志（只读）|
| `0xB00` | `mcycle` | 机器周期计数器（低32位）|
| `0xB02` | `minstret` | 退休指令计数器（低32位）|
| `0x7B0` | `dcsr` | 调试控制寄存器 |
| `0x7B1` | `dpc` | 调试时的 PC 保存 |

---

## G. 读代码的建议路径

**初学者推荐阅读顺序**（由浅入深）：

```
1. sirv_gnrl_dffs.v         ← 理解 DFF，一切的基础
2. e203_defines.v            ← 看懂宏定义命名规律
3. e203_ifu_ifetch.v         ← 最简单的有状态逻辑
4. e203_exu_oitf.v           ← FIFO 设计：掌握指针管理
5. e203_exu_decode.v         ← 大型 case 语句：指令解码
6. e203_exu_alu_dpath.v      ← 纯组合逻辑：加减法器
7. e203_lsu_ctrl.v           ← 访存地址路由
8. e203_itcm_ctrl.v          ← SRAM 访问仲裁
9. e203_exu_commit.v         ← 控制流：最复杂的状态机
10. e203_cpu.v               ← 顶层连线，看全局
```
