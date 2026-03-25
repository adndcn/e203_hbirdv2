# 第6章：时钟、复位与功耗管理

> **本章目标**：了解 e203 在"系统级"如何管理时钟和电源。时钟门控如何节省功耗？复位信号如何安全地同步？AON 低功耗域是做什么的？

---

## 6.1 时钟体系概览

e203 系统有多个时钟域，各自独立控制：

```
外部晶振 / PLL（e203_subsys_hclkgen）
        │
        ├─ core_clk      → 处理器核（IFU/EXU/LSU）
        ├─ clk_itcm_ram  → ITCM SRAM（门控）
        ├─ clk_dtcm_ram  → DTCM SRAM（门控）
        │
AON 域时钟（独立，低频）
        ├─ 32KHz RTC 时钟 → 实时时钟、看门狗
        └─ PMU 时钟       → 电源管理
```

核心设计思路：**分域门控，用到再开，不用就关**——只在有实际计算任务时才激活对应模块的时钟。

---

## 6.2 时钟门控（Clock Gating）

### 原理

数字电路的功耗主要来自两类：
- **静态功耗**：晶体管的漏电（总是存在，和工作状态无关）
- **动态功耗**：信号翻转时的充放电（翻转越多，功耗越高）

动态功耗计算公式：

```
P_dynamic = α × C × V² × f

α = 活动因子（每拍翻转的比例）
C = 节点电容
V = 电源电压
f = 时钟频率
```

减少动态功耗的最直接方式：**关掉不需要的模块的时钟**（α → 0）。

### e203 的时钟门控单元

```verilog
// e203_clkgate.v
module e203_clkgate(
  input  clk_in,
  input  test_mode,  // 测试模式：旁路门控，时钟直通
  input  clock_en,   // 使能：1=开时钟，0=关时钟
  output clk_out
);

reg enb;
// 在时钟低电平时采样使能信号（防毛刺！）
always @(*)
  if (!clk_in) enb = (clock_en | test_mode);

assign clk_out = enb & clk_in;
```

### 哪些模块有独立时钟门控？

```
e203_cpu.v 顶层控制：
  core_cgstop  → 核心流水线时钟停止标志
  tcm_cgstop   → TCM SRAM 时钟停止标志

触发条件（core_wfi 信号）：
  CPU 执行 WFI（Wait For Interrupt）指令 
  → IFU 停止取指
  → EXU 空闲后发出 core_wfi = 1
  → 关闭核心时钟，等待中断唤醒
```

**WFI（Wait For Interrupt）的低功耗流程**：

```
程序执行 WFI 指令
        ↓
EXU 的 commit 模块检测到 wfi_halt_req
        ↓
等待所有在途指令（OITF）清空
        ↓
向 IFU 发出停止请求（wfi_halt_ifu_req）
        ↓
IFU 和 EXU 同时确认停止（wfi_halt_ack）
        ↓
core_wfi = 1 → 时钟门控关断核心时钟
        ↓
（中断到来时）
中断信号唤醒时钟门控
        ↓
恢复执行，处理中断
```

---

## 6.3 复位控制（Reset Controller）

### 复位信号的问题

外部复位信号（`rst_n`，低有效）是**异步**的——它可能在时钟的任意时刻到来。这就带来一个问题：

```
理想情况（复位信号和时钟对齐）：
  CLK:  ───┌──┐  ───┌──┐  ───
            │  │     │  │
rst_n:  0───────────1──────────  （整整齐齐）
           ↑ Edge OK

实际情况（复位信号异步到来）：
  CLK:  ───┌──┐  ───┌──┐  ───
            │  │     │  │
rst_n:       ↑ ???（在时钟高低电平随机时刻变化）
```

如果某些触发器"看到"了复位，另一些因为时序差异没"看到"，就会出现**亚稳态（metastability）**——触发器的输出既不是 0 也不是 1，而是停在中间不确定的状态，可能持续一段时间才稳定，导致电路错误。

### 异步复位、同步释放（ARSR）

e203 使用**同步化复位（Synchronizer）**：通过 2 级 D 触发器链，把异步复位信号同步到系统时钟域：

```verilog
// e203_reset_ctrl.v
// 2级（或更多）同步器链
reg [RST_SYNC_LEVEL-1:0] rst_sync_r;

always @(posedge clk or negedge rst_n)
begin
  if (rst_n == 1'b0)
    rst_sync_r <= {RST_SYNC_LEVEL{1'b0}};  // 异步复位：立刻清零
  else
    rst_sync_r <= {rst_sync_r[RST_SYNC_LEVEL-2:0], 1'b1};  // 从右移入1
end

// 输出：经过几拍延迟的稳定复位信号
assign rst_sync_n = rst_sync_r[RST_SYNC_LEVEL-1];
```

**原理可视化**：

```
外部 rst_n 拉高（释放复位）后：
  Cycle0:  rst_sync_r = 00（仍然复位中）
  Cycle1:  rst_sync_r = 01
  Cycle2:  rst_sync_r = 11（输出 rst_sync_n = 1，复位释放）
```

复位释放被推迟了 2 拍，确保所有触发器同步地退出复位状态——这就是"同步释放"。

---

## 6.4 AON 域：芯片的"心脏起搏器"

### 什么是 AON 域？

AON（Always-On，始终开启）域是芯片里**不会被关掉的部分**，即使主处理器已经睡眠。它运行在独立的低频时钟（通常 32KHz）下，功耗极低。

```
正常运行模式：
  主域（core + TCM + 外设）: 工作中     功耗 ~5mW
  AON 域（RTC + PMU + WDOG）: 工作中    功耗 ~几μW

深睡眠模式（主域掉电）：
  主域：关闭                            功耗 ≈ 0
  AON 域（RTC + PMU + WDOG）: 仍运行   功耗 ~几μW

唤醒条件：
  RTC 定时到 → 唤醒主域
  外部中断   → 唤醒主域
  看门狗超时 → 触发复位

总睡眠功耗 ≈ AON 域功耗 ≈ 几微瓦！
```

### AON 域包含什么？

```
sirv_aon_top（e203_subsys/perips/ 中）
  ├─ sirv_rtc        实时时钟（RTC）：精确计时，可触发中断
  ├─ sirv_pmu_core   电源管理单元（PMU）：控制各子域的上下电
  ├─ sirv_aon_porrst 上电复位（Power-On Reset）：芯片上电时产生初始复位
  └─ 低功耗时钟生成器
```

---

## 6.5 e203 的低功耗设计理念总结

e203 把低功耗设计贯穿到了每一个层次：

```
电路层面：
  ✓ 时钟门控单元（clkgate）
  ✓ SRAM 独立时钟门控
  ✓ IR 高位分开单独更新（减少 DFF 翻转）

微架构层面：
  ✓ 2级流水线（比5级流水线少 3 倍的触发器）
  ✓ ITCM 64bit 读（每2条指令只读1次 SRAM）
  ✓ 顺序执行避免乱序的控制电路
  ✓ ITCM hold-up 避免重复 SRAM 访问

系统层面：
  ✓ WFI 指令进入低功耗等待
  ✓ AON 域独立于主域，深睡眠时只有 AON 工作
  ✓ 无 Cache（Cache 的置换和维护逻辑本身耗电）
  ✓ TCM 延迟确定不需要预取冒险的功耗
```

---

## 本章小结

| 机制 | 功能 | 关键信号/模块 |
|------|------|-------------|
| 时钟门控 | 不用的模块关时钟 | `e203_clkgate.v`，`clk_itcm_ram`  |
| WFI | 空闲时停核心时钟 | `core_wfi`, `wfi_halt_*` |
| 复位同步器 | 异步复位→同步释放，防亚稳态 | `e203_reset_ctrl.v`，`rst_sync_r` |
| AON 域 | 深睡眠下维持计时和唤醒能力 | `sirv_aon_top`，RTC/PMU |
