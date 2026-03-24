# 传入core的 ifu2itcm_icb_rsp_rdata 信号, 最终会传给哪些模块?

搜索文本 `ifu2itcm_icb_rsp_rdata` (`**/core/e203_core.v`)，3 个结果

读取 [](file:///d%3A/riscv/hbirdv2_git/e203_hbirdv2/rtl/e203/core/e203_core.v#385-385)，行 385 到 410

读取 [](file:///d%3A/riscv/hbirdv2_git/e203_hbirdv2/rtl/e203/core/e203_core.v#340-340)，行 340 到 395

确认是传给 `e203_ifu`，继续追踪进入 IFU 内部： 

搜索文本 `ifu2itcm_icb_rsp_rdata` (`**/core/e203_ifu.v`)，3 个结果

读取 [](file:///d%3A/riscv/hbirdv2_git/e203_hbirdv2/rtl/e203/core/e203_ifu.v#200-200)，行 200 到 230

读取 [](file:///d%3A/riscv/hbirdv2_git/e203_hbirdv2/rtl/e203/core/e203_ifu.v#175-175)，行 175 到 200

传到 `e203_ifu_ift2icb`，继续追踪： 

搜索文本 `ifu2itcm_icb_rsp_rdata` (`**/core/e203_ifu_ift2icb.v`)，8 个结果

读取 [](file:///d%3A/riscv/hbirdv2_git/e203_hbirdv2/rtl/e203/core/e203_ifu_ift2icb.v#510-510)，行 510 到 620

读取 [](file:///d%3A/riscv/hbirdv2_git/e203_hbirdv2/rtl/e203/core/e203_ifu_ift2icb.v#620-620)，行 620 到 660

搜索文本 `i_ifu_rsp_instr` (`**/core/e203_ifu_ift2icb.v`)，3 个结果

读取 [](file:///d%3A/riscv/hbirdv2_git/e203_hbirdv2/rtl/e203/core/e203_ifu_ift2icb.v#130-130)，行 130 到 165

搜索文本 `ifu_rsp_instr` (`**/core/e203_ifu.v`)，5 个结果

读取 [](file:///d%3A/riscv/hbirdv2_git/e203_hbirdv2/rtl/e203/core/e203_ifu.v#145-145)，行 145 到 170

搜索文本 `ifu_rsp_instr|ifu_o_ir` (`**/core/e203_ifu_ifetch.v`)，4 个结果

读取 [](file:///d%3A/riscv/hbirdv2_git/e203_hbirdv2/rtl/e203/core/e203_ifu_ifetch.v#240-240)，行 240 到 302

完整链路已经清晰，现在整理一下答案：

---

## `ifu2itcm_icb_rsp_rdata` 信号最终流向

信号在 **`e203_ifu_ift2icb`** 模块内被消费，**不再传给其他模块**，而是在此模块内部完成从 64-bit raw 数据到 32-bit 指令的转换。具体路径：

```
ifu2itcm_icb_rsp_rdata [63:0]  (传入 e203_ifu_ift2icb)
│
├─ 1. 高16位 → put2leftover_data[15:0]
│       → leftover_r 寄存器 (sirv_gnrl_dffl)
│         ← 用于缓存跨对齐边界的压缩指令上半部分
│
├─ 2. 低16位 → ifu_icb_rsp_rdata_lsb16[15:0]
│         ← 与 leftover_r 拼接成32位对齐指令
│
└─ 3. 按地址对齐切片 → ifu2itcm_icb_rsp_instr [31:0]
         addr[2:1]==00 → rdata[31:0]
         addr[2:1]==01 → rdata[47:16]
         addr[2:1]==10 → rdata[63:32]
              ↓
         ifu_icb_rsp_instr
              ↓
         i_ifu_rsp_instr
              ↓
         sirv_gnrl_bypbuf (bypass FIFO，深度1)
              ↓
         ifu_rsp_instr  (ift2icb 的 output port)
```

之后继续向上传递：

| 层次 | 模块 | 信号变化 |
|---|---|---|
| e203_ifu_ift2icb.v | 数据处理终点 | → `ifu_rsp_instr` (32-bit) |
| e203_ifu_ifetch.v | 锁存到 IR 寄存器 | `ifu_ir_nxt` → `ifu_ir_r` → `ifu_o_ir` |
| e203_ifu.v | 端口透传 | `ifu_o_ir` |
| e203_core.v | 传入 EXU | `ifu_o_ir` → `e203_exu` decode 阶段 |

### 关键设计点

`ift2icb` 做了两件核心事：
1. **64-bit → 32-bit 对齐**：根据 PC 的 `[2:1]` 位选择 64-bit 返回数据的哪个 32-bit 窗口
2. **跨对齐边界处理**：对于跨越 64-bit 边界的 RVC/32-bit 指令（如地址 `0x3E`），把第一拍的高 16-bit 存入 `leftover_r`，第二拍取回后拼接成完整指令