# 第7章：调试系统（Debug System）

> **本章目标**：了解如何通过 JTAG 接口调试 e203 处理器。调试器（如 GDB）如何让正在运行的 CPU 暂停？如何读写寄存器和内存？

---

## 7.1 调试系统总览

当我们说"调试 CPU"，其实是在做这些事：
- **暂停**（Halt）：让 CPU 停下来
- **单步**（Step）：执行一条指令后又暂停
- **读/写寄存器**：查看 `x5` 当前的值
- **读/写内存**：查看 `0x80001000` 处的数据
- **继续**（Resume）：让 CPU 恢复正常运行

e203 的调试系统由三层构成：

```
调试器（PC上的 GDB）
    ↕ USB/网络
OpenOCD（调试中间件）
    ↕
JTAG 物理线（4-5 根信号线）
    ↕
[片上] DTM（Debug Transport Module，调试传输模块）
    ↕ debug bus（34位）
[片上] DM（Debug Module，调试模块）
    ↕ 
[片上] CPU 核心（通过 dbg_* 信号控制）
```

---

## 7.2 JTAG 接口

JTAG（Joint Test Action Group）最初是用于电路板测试的标准，后来被广泛用于芯片调试。

### 5 根物理信号线

| 信号 | 方向（相对芯片）| 含义 |
|------|------------|------|
| TCK  | 输入 | 测试时钟（Test Clock）|
| TMS  | 输入 | 测试模式选择（Test Mode Select）|
| TDI  | 输入 | 测试数据输入（Test Data In）|
| TDO  | 输出 | 测试数据输出（Test Data Out）|
| TRST | 输入 | 测试复位（Test Reset，低有效）|

JTAG 是**串行接口**：数据一位一位地通过 TDI 移入，同时从 TDO 移出。TCK 是移位时钟，TMS 控制内部状态机。

### TAP 状态机

DTM 内部有一个 **16 状态的 TAP（Test Access Port）状态机**，它在每个 TCK 上升沿根据 TMS 的值跳转：

```
                TMS=1                TMS=0
               ┌──────┐
               ↓      │
        TEST_LOGIC_RESET ──TMS=0──→ RUN_TEST_IDLE
                                         │
                                    TMS=1↓
                                    SELECT_DR ──TMS=1──→ SELECT_IR
                                         │                    │
                                    TMS=0↓               TMS=0↓
                                    CAPTURE_DR          CAPTURE_IR
                                         │                    │
                                    TMS=0↓               TMS=0↓
                                    SHIFT_DR ←──────    SHIFT_IR ←──────
                  （数据在此移入移出）↑      │               ↑      │
                                    └──────┘               └──────┘
                                    TMS=1↓               TMS=1↓
                                    EXIT1_DR             EXIT1_IR
                                         .                    .
                                    UPDATE_DR            UPDATE_IR
                                         │                    │
                                    （DR更新）            （IR更新）
```

关键状态：
- **SHIFT_DR**：数据寄存器（DR）移位，TDI 移入，TDO 移出
- **SHIFT_IR**：指令寄存器（IR）移位，写入 5 位指令码
- **UPDATE_DR/IR**：移位完成，锁存到实际寄存器

### 两类寄存器

JTAG 操作分两步：先写 IR（选择哪个 DR），再操作 DR（读写数据）。

**指令寄存器（IR，5位）**：

```verilog
localparam REG_BYPASS       = 5'b11111;  // 直通（测试用）
localparam REG_IDCODE       = 5'b00001;  // 读芯片 ID
localparam REG_DTM_INFO     = 5'b10000;  // 读 DTM 信息
localparam REG_DEBUG_ACCESS = 5'b10001;  // 访问调试总线（核心功能）
```

**调试数据寄存器（DR，34位，当 IR=REG_DEBUG_ACCESS 时）**：

```
[33:2] = data   (32位数据)
[6:2]  = addr   (5位地址，选择 DM 中的寄存器)
[1:0]  = op     (操作码：00=空，01=读，10=写，11=保留)
```

---

## 7.3 DTM（调试传输模块）

DTM 的工作：把 JTAG 串行数据流翻译成调试总线（debug bus）上的并行读写操作。

```verilog
// sirv_jtag_dtm.v 主要参数

parameter DEBUG_DATA_BITS = 34;    // DR 位宽
parameter DEBUG_ADDR_BITS = 5;     // 地址位宽（DM 寄存器地址空间）
parameter IR_BITS = 5;             // IR 位宽
```

### 完整的 JTAG 调试访问流程

以"读取 DM 的 `dmstatus` 寄存器（地址 0x11）"为例：

```
步骤1：用 TMS 跳转到 SHIFT_IR 状态
步骤2：移入 5'b10001（REG_DEBUG_ACCESS），进入 UPDATE_IR → IR 更新

步骤3：跳转到 CAPTURE_DR（DR 自动加载当前状态）
步骤4：跳转到 SHIFT_DR，移入 34位数据：
           [33:2] = 32'h0（data，读操作 data 忽略）
           [6:2]  = 5'h11（addr = 0x11，dmstatus）
           [1:0]  = 2'b01（op = read）
       同时从 TDO 移出上次操作的结果

步骤5：跳转到 UPDATE_DR → DTM 发出调试总线请求
步骤6：DTM 等待 DM 响应（通过再次 SHIFT_DR 读取 resp 字段判断是否完成）

步骤7：再发一次读操作（op=00，空操作），从移出数据中取得结果
```

### DTM 与 DM 之间的接口

```
DTM → DM：
  dtm_req_valid  : 1   // 新请求
  dtm_req_bits   : {op, addr, data}  // 34位请求

DM → DTM：
  dtm_resp_valid : 1   // 响应ready
  dtm_resp_bits  : {resp, data}    // 2位resp + 32位数据
```

---

## 7.4 DM（调试模块）

DM 是调试系统的"核心控制器"，它通过两类接口来调试 CPU：
1. **控制信号**：直接连到 CPU 的 `dbg_*` 信号，控制暂停/恢复
2. **ICB 总线接口**：通过片上系统总线可以读写任意内存和 Debug RAM

### DM 内部的关键寄存器（地址空间 0x00~0x3F）

| 地址 | 名称 | 作用 |
|------|------|------|
| 0x10 | `dmcontrol` | 控制寄存器：halt/resume CPU |
| 0x11 | `dmstatus`  | 状态寄存器：CPU 是否停止 |
| 0x1A | `command`   | 抽象命令：读写寄存器 |
| 0x04~0x0B | `data0~7` | 命令数据缓存区 |
| 0x38 | `progbuf0`  | 程序缓冲（写小程序让 CPU 执行）|

### CPU Halt（暂停）机制

DM 通过调试中断信号 `dbg_irq` 来暂停 CPU：

```
GDB 请求 halt
    ↓
DM 设置 dmcontrol.haltreq = 1
    ↓
DM 向 CPU 发送调试中断请求（dbg_irq）
    ↓
CPU 的 commit 模块收到 dbg_irq
    ↓
CPU 完成当前指令后跳转到 Debug ROM（0x800）
    ↓
Debug ROM 执行：更新 dpc（PC 保存寄存器），设置 dcsr.cause
    ↓
CPU 进入 debug mode，停止取指
    ↓
DM 检测到 CPU halted（通过 cmt_dpc_ena 信号），设置 dmstatus.halted = 1
    ↓
DTM 通知 OpenOCD → GDB 收到"CPU 已停止"
```

### Debug ROM 和 Debug RAM

```
Debug ROM（只读，出厂固化代码）：
  地址 0x0000_0800
  包含进入/退出 debug mode 的固定代码

Debug RAM（可写，临时代码区）：
  地址 0x0000_0000（内部地址空间）
  DM 写入小程序（如"把 x5 写入 data0"）
  CPU 在 debug mode 里执行这段程序
  执行完后结果存入 DM 的 data0，GDB 读取
```

---

## 7.5 从 GDB 到 CPU 的完整路径

以 GDB 执行 `info registers` 为例：

```
用户键入 "info registers"
    ↓
GDB 向 OpenOCD 发送 "读寄存器 x0~x31" 命令
    ↓
OpenOCD 构造 JTAG 数据，通过 JTAG 线发送：
  - 写 DM.command = {type=0, regno=x0, transfer=1, write=0}
    （"抽象命令：读 x0"）
    ↓
DM 执行：生成小程序，让 CPU 执行"把 x0 的值存到 data0"
    ↓
DTM 读 DM.data0 → OpenOCD 得到 x0 的值
    ↓
重复 x1~x31
    ↓
GDB 显示所有寄存器的值
```

整个过程对用户完全透明——GDB 就好像在直接"看"CPU 内部一样。

---

## 本章小结

```
JTAG 物理层（5线串行）
    ↓  DTM 翻译
调试总线（34位并行，op/addr/data）
    ↓  DM 处理
CPU 控制信号（dbg_irq, dbg_mode, dpc, dcsr）
    ↓
CPU 在 debug mode 下执行 debug ROM/RAM 中的小程序
完成寄存器读写、内存读写、单步执行等操作
```

| 组件 | 文件 | 职责 |
|------|------|------|
| DTM | `sirv_jtag_dtm.v` | JTAG → 调试总线协议翻译 |
| DM  | `sirv_debug_module.v` | 调试控制逻辑 + 寄存器访问 |
| Debug ROM | `sirv_debug_rom.v` | CPU halt/resume 固化代码 |
| Debug RAM | `sirv_debug_ram.v` | 临时调试程序存储 |
