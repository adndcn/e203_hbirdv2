# 为什么叫ift_ift2icb

名字是三段缩写的组合：

**`ifu`** + **`ift`** + **`2`** + **`icb`**

| 片段 | 全称 | 含义 |
|---|---|---|
| `ifu` | Instruction Fetch Unit | 所属单元：取指单元 |
| `ift` | Instruction FeTch | 取指请求接口（模块内部的 fetch req/rsp 握手） |
| `2` | to | "转换为" |
| `icb` | Internal Chip Bus | 目标总线协议 |

所以完整含义是：**"IFU 内部的取指接口（IFT）到 ICB 总线协议的转换器"**。

这个模块的本质就是一个**协议适配桥**：
- **上游**（面向 `e203_ifu_ifetch`）：使用简单的 `ifu_req` / `ifu_rsp` 握手接口，只关心 PC 和指令
- **下游**（面向 ITCM/BIU）：使用标准 ICB 协议（`cmd_valid/ready`、`rsp_valid/ready`）

同时它还处理了 IFT 协议到 ICB 协议之间最复杂的差异：**跨 64-bit 边界的 2-拍取指**，以及 64-bit 宽 ITCM 返回数据的地址对齐切片——这些细节对上层 `ifetch` 完全透明。