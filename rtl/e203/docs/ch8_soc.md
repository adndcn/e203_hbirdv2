# 第8章：SoC 集成与外设

> **本章目标**：了解 e203 CPU 核心如何连接到整个芯片系统（SoC）。包括总线互连、中断控制器、定时器和常见外设。

---

## 8.1 SoC 模块层次

```
e203_soc_top（芯片顶层）
  │
  ├─ e203_subsys_top（CPU 子系统，核心部分）
  │    ├─ e203_cpu_top（CPU 核 + ITCM/DTCM SRAM）
  │    │    ├─ e203_cpu（IFU + EXU）
  │    │    └─ e203_srams（ITCM + DTCM SRAM 控制）
  │    │
  │    ├─ sirv_debug_module（调试模块）
  │    │
  │    ├─ [系统总线 ICB 路由]
  │    │    ├─ sirv_clint_top   @0x0200_0000（定时器+软中断）
  │    │    ├─ sirv_plic_top    @0x0C00_0000（外部中断控制器）
  │    │    ├─ sirv_mrom_top    @0x0001_0000（Mask ROM）
  │    │    └─ 外部 ICB 总线接口（连接 SoC 外设）
  │    │
  │    └─ sirv_aon_top（Always-On 域）
  │         ├─ RTC（实时时钟）
  │         ├─ PMU（电源管理）
  │         └─ 看门狗（WDOG）
  │
  └─ 片上外设（在 SoC 层面连接）
       ├─ GPIO（通用 I/O）
       ├─ SPI / QSPI（Flash 接口）
       ├─ UART
       └─ ...
```

---

## 8.2 ICB 总线互连结构（fab）

### 为什么需要总线 fabric？

CPU 只有一套 ICB 接口（来自 BIU），但芯片上有很多外设，各自有独立的 ICB 接口。需要"分线器"把 CPU 的一条总线路由到多个外设：

```
CPU BIU ICB（主机：只有1个）
         ↓
    sirv_icb1to2_bus
    ┌────────────────┐
    │ 地址匹配逻辑   │
    │ O0_BASE_ADDR   │
    └──────┬─────────┘
           ├── sysmem ICB（ITCM/DTCM/外部存储）
           └── sysper ICB（外设）
                    ↓
              sirv_icb1to8_bus
              ├── CLINT        @0x0200_0000
              ├── PLIC         @0x0C00_0000
              ├── Debug Module @0x0000_0000（内部地址）
              ├── MROM         @0x0001_0000
              └── ...
```

### 1-to-2 总线的地址路由逻辑

```verilog
// sirv_icb1to2_bus.v 核心逻辑（简化）

// 示例：O0_BASE_ADDR = 0x2000_0000，O0_BASE_REGION_LSB = 28
// 意思是：地址[31:28] == 4'h2 → 路由到 O0

assign sel_o0 = (i_icb_cmd_addr[AW-1:O0_BASE_REGION_LSB] 
                 == O0_BASE_ADDR[AW-1:O0_BASE_REGION_LSB]);
assign sel_o1 = ~sel_o0;

// 命令only发到选中的端口
assign o0_icb_cmd_valid = i_icb_cmd_valid & sel_o0;
assign o1_icb_cmd_valid = i_icb_cmd_valid & sel_o1;

// 主机ready = 被选中端口的ready
assign i_icb_cmd_ready = sel_o0 ? o0_icb_cmd_ready : o1_icb_cmd_ready;

// 响应回来：记录当时选了哪个端口（by FIFO），选择对应的rsp
assign i_icb_rsp_rdata = (rsp_sel == 0) ? o0_icb_rsp_rdata : o1_icb_rsp_rdata;
```

关键点：**命令通道选择一次就写入 FIFO 记录，响应通道按 FIFO 顺序还原**——这保证了即使有多笔在途请求，响应也不会乱序。

### 完整的地址空间分布

```
地址范围                  大小   连接目标
0x0000_0000~0x0000_FFFF   64KB   调试域（Debug ROM/RAM）
0x0001_0000~0x0001_7FFF   32KB   Mask ROM（上电 Boot 代码）
0x0200_0000~0x0200_BFFF   48KB   CLINT（定时器+软中断）
0x0C00_0000~0x0FFF_FFFF   64MB   PLIC（外部中断仲裁）
0x8000_0000~0x8000_FFFF   64KB   ITCM（代码+数据）
0x9000_0000~0x9000_FFFF   64KB   DTCM（数据）
0x1000_0000~...                   SoC 外设（GPIO/UART/SPI/...)
```

---

## 8.3 CLINT——核内中断控制器

CLINT（Core-Local INTerrupt，核本地中断控制器）负责**软件中断**和**计时器中断**，这是 RISC-V 规范强制要求的。

### CLINT 包含什么？

```
CLINT 寄存器（映射在 0x0200_0000 起始）

偏移 0x0000：msip（Machine Software Interrupt Pending）
  [0]  = 1 → 触发软件中断（写1产生中断，写0清除）
  应用：多核通信中，一个核通知另一个核

偏移 0x4000：mtimecmp（64位计时器比较值）
  当 mtime >= mtimecmp 时，触发定时器中断

偏移 0xBFF8：mtime（64位机器时间计数器）
  由 RTC tick 驱动，每个 tick 加1，只增不减
  应用：实现 RTOS 的时间片调度
```

### 定时器中断示例

```c
// 用 CLINT 实现 1ms 定时中断（假设 RTC 32768Hz）
// 每 32.768 个 tick ≈ 1ms，取整为 33

void set_timer_interrupt(uint64_t delta_ticks) {
    volatile uint64_t *mtime    = (uint64_t*)0x0200BFF8;
    volatile uint64_t *mtimecmp = (uint64_t*)0x02004000;
    *mtimecmp = *mtime + delta_ticks;
}
// 在中断服务例程里重新设置 mtimecmp 就实现了周期定时
```

### CLINT 时钟同步细节

```verilog
// sirv_clint_top.v 中的 RTC 边沿检测
wire io_rtcToggle_r; 
sirv_gnrl_dffr #(1) io_rtcToggle_dffr(io_rtcToggle, io_rtcToggle_r, clk, rst_n);
wire io_rtcToggle_edge = io_rtcToggle ^ io_rtcToggle_r;
wire io_rtcTick = io_rtcToggle_edge;
```

RTC 时钟（32KHz）可能不是系统时钟（16MHz）的整数倍，不能直接用 RTC 信号。AON 域把 RTC 时钟的每个周期变成一个"toggle"的边沿事件传到主域，主域检测这个 toggle 信号的变化来计数——从而安全地跨时钟域。

---

## 8.4 PLIC——平台级中断控制器

PLIC（Platform-Level Interrupt Controller）负责处理来自**片外外设**的中断请求，如 UART 收到数据、GPIO 边沿等。

### PLIC 的工作原理

```
外设 0~16（每个都可能同时产生中断）
        ↓
   中断优先级寄存器（每个外设一个，3位）
        ↓
   门控（中断使能寄存器，每个外设1位）
        ↓
   仲裁逻辑（选出优先级最高且使能的中断源）
        ↓
   阈值寄存器（优先级 > 阈值才报给 CPU）
        ↓
   io_harts_0_0 → CPU 的 meip（machine external interrupt pending）
```

### PLIC 的 claim/complete 流程

```
1. 中断发生 → PLIC 向 CPU 报告 meip=1
2. CPU 跳转到中断向量，保存现场
3. CPU 读 PLIC.claim 寄存器（@0x0C20_0004）
   → PLIC 返回最高优先级中断源 ID（如7=UART0_RX）
   → 同时自动清除该中断的 pending 位
4. CPU 处理中断（如读 UART 数据寄存器）
5. CPU 写 PLIC.complete 寄存器（写入中断源 ID）
   → PLIC 知道这个中断已处理完，可以再次激活
6. CPU 恢复现场，执行 mret 返回
```

### e203 支持的中断源数量

```verilog
// sirv_plic_top.v
localparam PLIC_IRQ_NUM = 17;  // 16 个外设中断 + 1 个永远为0的 IRQ0
// 可以增大，最多支持 1024 个
```

---

## 8.5 中断路由总览

e203 有三类中断，对应 RISC-V 特权规范的三个 MIP 位：

```
├── msip（软件中断）← CLINT.msip[0] 写1触发
│                       CPU 处理：读 CLINT，清零并执行软件任务
│
├── mtip（定时器中断）← mtime >= mtimecmp 时触发
│                       CPU 处理：重设 mtimecmp 实现周期定时
│
└── meip（外部中断）← PLIC 仲裁后发给 CPU
                        CPU 处理：先读 PLIC claim，处理后写 complete

```

三类中断全部路由到 e203 的 IRQ 同步器（`e203_irq_sync.v`），经过 2 级 DFF 同步后进入 EXU commit 模块的中断处理逻辑。

---

## 8.6 Boot 流程（从上电到执行用户代码）

```
芯片上电
    ↓
上电复位（POR）→ 系统进入复位状态
    ↓
复位释放（rst_n 拉高）
    ↓
PC 初始值 = 0x0001_0000（Mask ROM 起始地址）
    ↓
CPU 从 Mask ROM 取第一条指令

[Mask ROM 中的 Boot 程序]：
    ├─ 检测 bootrom_n 引脚（0 = 从 ITCM 启动，1 = 从 SPI Flash 启动）
    │
    ├─ 若从 Flash 启动：
    │    1. 初始化 QSPI 控制器
    │    2. 把 Flash 前 64KB 数据拷贝到 ITCM
    │    3. 跳转到 ITCM（0x8000_0000）
    │
    └─ 若直接从 ITCM 启动：
         跳转到 0x8000_0000

[ITCM 中的用户程序]：
    1. 初始化 .bss 段（清零）
    2. 初始化 .data 段（从 Flash 拷贝）
    3. 设置 RISC-V 中断向量基址（mtvec 寄存器）
    4. 调用 main()
```

---

## 8.7 外设访问示例：让 GPIO 闪烁 LED

```c
// HBirdv2 GPIO 控制寄存器（以 gpioA 为例）
#define GPIO_BASE   0x10012000
#define GPIO_INPUT  (*(volatile uint32_t*)(GPIO_BASE + 0x00))
#define GPIO_OUTPUT (*(volatile uint32_t*)(GPIO_BASE + 0x0C))
#define GPIO_OE     (*(volatile uint32_t*)(GPIO_BASE + 0x08))  // output enable

// 让 GPIO0 输出高电平（LED 亮）
GPIO_OE     |= (1 << 0);  // GPIO0 设为输出
GPIO_OUTPUT |= (1 << 0);  // GPIO0 输出1

// 以上操作最终变成：
// CPU → EXU(AGU) → LSU → BIU ICB → SoC 系统总线 ICB
//   → sirv_icb1to8_bus 路由（地址匹配 GPIO）
//     → GPIO 寄存器写入
//       → pad 输出高电平 → LED 亮
```

整个路径从 CPU 核心出发，经过 ICB 总线，到达 GPIO 外设，驱动引脚——这就是一条完整的数据路径。

---

## 本章小结

| 组件 | 位置 | 功能 |
|------|------|------|
| 总线路由（fab） | `sirv_icb1to2_bus.v` | 地址匹配，分发到各外设 |
| CLINT | `sirv_clint_top.v` | 定时器(mtime/mtimecmp)、软中断(msip) |
| PLIC | `sirv_plic_top.v` | 外部中断仲裁、优先级管理 |
| AON | `sirv_aon_top.v` | 低功耗域：RTC/PMU/WDOG |
| Debug Module | `sirv_debug_module.v` | JTAG 调试，暂停/读写 CPU |
| Mask ROM | `sirv_mrom_top.v` | 上电 Boot 代码，不可修改 |
