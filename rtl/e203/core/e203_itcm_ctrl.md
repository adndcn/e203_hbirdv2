# e203_itcm_ctrl 如何从srams中读取数据, 并送入到e203_core中, 这之间的信号走向

搜索正则表达式 `itcm_ram_|itcm_icb_|e203_itcm_ctrl|e203_core`，20 个结果

读取 [](file:///d%3A/riscv/hbirdv2_git/e203_hbirdv2/rtl/e203/core/e203_cpu.v#395-395)，行 395 到 450

读取 [](file:///d%3A/riscv/hbirdv2_git/e203_hbirdv2/rtl/e203/core/e203_cpu.v#750-750)，行 750 到 870

搜索正则表达式 `itcm_ram_dout|itcm_ram_cs|itcm_ram_addr|ifu2itcm_icb_rsp_rdata|ram_dout`，20 个结果

搜索正则表达式 `ifu2itcm_icb_rsp|lsu2itcm_icb_rsp`，20 个结果

读取 [](file:///d%3A/riscv/hbirdv2_git/e203_hbirdv2/rtl/e203/core/e203_itcm_ctrl.v#490-490)，行 490 到 545

现在信号流路径完全清晰。下面是完整分析：

---

### 读数据信号走向（以 IFU 取指为例）

```
e203_core (IFU)                 e203_itcm_ctrl              e203_cpu_top (wire)        e203_srams → e203_itcm_ram → sirv_sim_ram
    │                                │                              │                          │
    │── ifu2itcm_icb_cmd_valid ──>  │                              │                          │
    │── ifu2itcm_icb_cmd_addr  ──>  │                              │                          │
    │                                │── itcm_ram_cs   ──────────> wire ──────────────────>  .cs
    │                                │── itcm_ram_addr ──────────> wire ──────────────────>  .addr
    │                                │── itcm_ram_we=0 ──────────> wire ──────────────────>  .we
    │                                │                              │                          │
    │                                │<── itcm_ram_dout <───────── wire <──────────────────  .dout
    │                                │  (sram_icb_rsp_rdata)        │                          │
    │<── ifu2itcm_icb_rsp_valid <──  │                              │                          │
    │<── ifu2itcm_icb_rsp_rdata <──  │  (assign from sram_icb_rsp_rdata)
```

---

### 关键信号链（按层次）

**① e203_core → e203_itcm_ctrl（ICB cmd 请求）**

| 信号 | 方向 | 含义 |
|------|------|------|
| `ifu2itcm_icb_cmd_valid` | IFU→ctrl | 取指请求有效 |
| `ifu2itcm_icb_cmd_addr` | IFU→ctrl | 取指地址 |
| `ifu2itcm_icb_cmd_read` | 固定=`1'b1` | 永远只读 |

**② e203_itcm_ctrl → e203_srams（RAM 物理接口）**

ctrl 内部通过 `sirv_sram_icb_ctrl` 子模块将 ICB 协议转换为 SRAM 同步接口：

| 信号 | 方向 | 含义 |
|------|------|------|
| `itcm_ram_cs` | ctrl→sram | 片选 |
| `itcm_ram_we` | ctrl→sram | 写使能（读时为0） |
| `itcm_ram_addr` | ctrl→sram | 行地址（13bit） |
| `itcm_ram_din` | ctrl→sram | 写数据（读时无效） |
| `itcm_ram_dout` | sram→ctrl | **读出的64bit数据** |

**③ e203_itcm_ctrl → e203_core（ICB rsp 响应）**

ctrl 内部直接连线：
```verilog
assign ifu2itcm_icb_rsp_rdata = sram_icb_rsp_rdata;  // sram_icb_rsp_rdata = itcm_ram_dout
assign ifu2itcm_icb_rsp_valid = sram_icb_rsp_valid & sram_icb_rsp_ifu;
```

| 信号 | 方向 | 含义 |
|------|------|------|
| `ifu2itcm_icb_rsp_valid` | ctrl→IFU | 数据就绪 |
| `ifu2itcm_icb_rsp_rdata` | ctrl→IFU | **64bit 指令数据** |
| `ifu2itcm_icb_rsp_err` | ctrl→IFU | 错误标志（固定为0） |

---

### 时序关键点

SRAM 是**同步读**（1拍延迟）：
- T0：`itcm_ram_cs=1, addr` 驱动到 SRAM
- T1：`itcm_ram_dout` 输出有效数据 → `ifu2itcm_icb_rsp_rdata` → IFU

`e203_itcm_ctrl` 内部通过 `sram_icb_rsp_ifu` 标志区分当前响应是来自 IFU 还是 LSU 的访问，从而将 `itcm_ram_dout` 路由到正确的 ICB rsp 通道。