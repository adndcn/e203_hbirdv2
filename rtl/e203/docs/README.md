# e203 HBirdv2 RISC-V 处理器 RTL 文档

> 面向初学者的 e203 RTL 实现详解——从零开始，深入到每一根信号。

---

## 文档目录

| 章节 | 文件 | 内容 |
|------|------|------|
| **第0章** | [ch0_prerequisites.md](ch0_prerequisites.md) | 前置知识：CPU 基础、RISC-V、Verilog/RTL |
| **第1章** | [ch1_architecture.md](ch1_architecture.md) | 整体架构：模块层次、流水线、地址空间、ICB 协议 |
| **第2章** | [ch2_ifu.md](ch2_ifu.md) | 取指单元 IFU：PC、IR、地址对齐、分支预测 |
| **第3章** | [ch3_exu.md](ch3_exu.md) | 执行单元 EXU：解码、寄存器堆、OITF、ALU、提交 |
| **第4章** | [ch4_lsu.md](ch4_lsu.md) | 存取单元 LSU：地址路由、符号扩展、NICE 接口 |
| **第5章** | [ch5_memory.md](ch5_memory.md) | 存储子系统：TCM、SRAM、ECC、时钟门控 |
| **第6章** | [ch6_clock_reset_power.md](ch6_clock_reset_power.md) | 时钟/复位/功耗：门控策略、复位同步、AON 域 |
| **第7章** | [ch7_debug.md](ch7_debug.md) | 调试系统：JTAG、DTM、DM、halt/resume 机制 |
| **第8章** | [ch8_soc.md](ch8_soc.md) | SoC 集成：总线路由、CLINT、PLIC、Boot 流程 |
| **附录** | [appendix.md](appendix.md) | 快速参考：参数表、信号命名、文件速查、CSR 表 |

---

## e203 是什么？

e203（HBirdv2）是芯来科技（Nuclei System Technology）开源的 **RISC-V RV32IMAC** 处理器：

- **2级流水线**（IFU 取指 + EXU 执行）
- **TCM 架构**（紧耦合存储，无 Cache）
- **超低功耗**，面向嵌入式 MCU
- 完整开源 RTL，适合学习

---

## 如何使用本文档？

**完全初学者**：按顺序从第 0 章读起，每章都有丰富的例子和信号追踪。

**已有 Verilog 基础**：可直接从第 1 章开始，遇到细节时用附录查找。

**对某个模块感兴趣**：直接跳到对应章节，每章独立成文，不强依赖前章。

**对照阅读 RTL**：遇到信号/参数不认识时查附录 B/C，遇到宏定义查附录 A。

---

## 关键设计特点速览

| 特点 | 位置 | 章节 |
|------|------|------|
| OITF（在途指令追踪）| `e203_exu_oitf.v` | 第3章 |
| liteBPU（静态分支预测）| `e203_ifu_litebpu.v` | 第2章 |
| ICB 总线协议 | `sirv_gnrl_icbs.v` | 第1章 |
| ITCM 64bit 取指 | `e203_ifu_ift2icb.v` | 第2章、第5章 |
| `dis_ready ≠ ~oitf_full \| ret_ena` | `e203_exu_disp.v` | 第3章 |
| 无 Cache，延迟确定 | ITCM/DTCM | 第5章 |
| 异步复位同步释放 | `e203_reset_ctrl.v` | 第6章 |
| WFI 时钟门控 | `e203_clk_ctrl.v` | 第6章 |
| JTAG 16态 TAP FSM | `sirv_jtag_dtm.v` | 第7章 |
