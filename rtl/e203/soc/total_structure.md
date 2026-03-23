搜索正则表达式 `e203_subsys_main|sirv_aon_top|module e203_subsys_top`，20 个结果

搜索正则表达式 `e203_cpu\b|e203_srams`，20 个结果

搜索正则表达式 `sirv_gnrl_ram|e203_itcm_ram|e203_dtcm_ram`，20 个结果

搜索正则表达式 `e203_ifu|e203_exu|e203_lsu|e203_biu`，20 个结果

现在信息完整了。以下是从 `e203_soc_top` 到 `e203_srams` 的完整层次结构：

```
e203_soc_top                     (soc/)           ← 芯片顶层，IO pad 接口
  │
  ├─ e203_subsys_top             (subsys/)         ← subsys 顶层
  │    ├─ e203_subsys_main       (subsys/)         ← subsys 主体
  │    │    ├─ e203_subsys_hclkgen                 ← 时钟生成（PLL）
  │    │    ├─ e203_cpu_top      (core/)           ← CPU + SRAM 顶层
  │    │    │    ├─ e203_cpu                       ← CPU 核心（IFU/EXU/LSU/BIU）
  │    │    │    └─ e203_srams   (core/)           ← ITCM/DTCM SRAM 顶层
  │    │    │         ├─ e203_itcm_ram             ← ITCM RAM 封装
  │    │    │         │    └─ sirv_gnrl_ram        ← 通用 RAM（general/）
  │    │    │         │         └─ sirv_sim_ram    ← 仿真/FPGA RAM 行为模型
  │    │    │         └─ e203_dtcm_ram             ← DTCM RAM 封装（同上）
  │    │    ├─ e203_subsys_plic                    ← 中断控制器
  │    │    ├─ e203_subsys_clint                   ← 计时器/软件中断
  │    │    ├─ e203_subsys_perips                  ← UART/SPI/GPIO 等外设
  │    │    └─ e203_subsys_mems                    ← 片上 ROM
  │    │
  │    └─ sirv_aon_top           (perips/)         ← AON 域（低功耗/RTC/WDOG/PMU）
```

**各层职责一句话总结：**

| 层次 | 职责 |
|------|------|
| `e203_soc_top` | 芯片 IO pad，连接外部引脚 |
| `e203_subsys_top` | 划分主域与 AON（Always-On）低功耗域 |
| `e203_subsys_main` | 集成 CPU、时钟、外设、中断控制器 |
| `e203_cpu_top` | CPU 核心 + TCM SRAM 实例化 |
| `e203_srams` | ITCM 和 DTCM 两块紧耦合存储器 |
| `sirv_gnrl_ram` | 根据 FPGA/仿真选择 RAM 实现 |
| `sirv_sim_ram` | 实际的同步 SRAM 行为模型 |