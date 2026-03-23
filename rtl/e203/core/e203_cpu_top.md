# e203_cpu_top 模块层级

搜索正则表达式 `^\s+e203_\w+ u_e203_`，20 个结果

搜索正则表达式 `^\s+e203_\w+ u_e203_`，20 个结果

搜索正则表达式 `^\s+e203_\w+ u_e203_`，20 个结果

搜索正则表达式 `u_e203_ifu_`，12 个结果

搜索正则表达式 `u_e203_`，20 个结果

搜索正则表达式 `u_e203_`，20 个结果

搜索正则表达式 `u_e203_`，20 个结果

搜索正则表达式 `^\s+e203_lsu_\w+ u_`，1 个结果

信息已经完整，汇总如下：

```
e203_cpu_top                              (core/e203_cpu_top.v)
  │
  ├─ e203_cpu                             (core/e203_cpu.v)
  │    ├─ e203_reset_ctrl                 (core/e203_reset_ctrl.v)       复位控制
  │    ├─ e203_clk_ctrl                   (core/e203_clk_ctrl.v)         时钟门控
  │    ├─ e203_irq_sync                   (core/e203_irq_sync.v)         中断同步
  │    ├─ e203_extend_csr                 (core/e203_extend_csr.v)       扩展CSR
  │    ├─ e203_subsys_nice_core           (subsys/e203_subsys_nice_core.v) NICE协处理器接口
  │    ├─ e203_itcm_ctrl                  (core/e203_itcm_ctrl.v)        ITCM访问控制
  │    ├─ e203_dtcm_ctrl                  (core/e203_dtcm_ctrl.v)        DTCM访问控制
  │    │
  │    └─ e203_core                       (core/e203_core.v)             流水线顶层
  │         ├─ e203_ifu                   (core/e203_ifu.v)              取指单元
  │         │    ├─ e203_ifu_ifetch       (core/e203_ifu_ifetch.v)       取指主逻辑
  │         │    │    ├─ e203_ifu_minidec (core/e203_ifu_minidec.v)      预译码
  │         │    │    └─ e203_ifu_litebpu (core/e203_ifu_litebpu.v)      轻量级分支预测
  │         │    └─ e203_ifu_ift2icb      (core/e203_ifu_ift2icb.v)      IFetch转ICB总线
  │         │
  │         ├─ e203_exu                   (core/e203_exu.v)              执行单元
  │         │    ├─ e203_exu_regfile      (core/e203_exu_regfile.v)      通用寄存器堆
  │         │    ├─ e203_exu_decode       (core/e203_exu_decode.v)       译码器
  │         │    ├─ e203_exu_disp         (core/e203_exu_disp.v)         派发/仲裁
  │         │    ├─ e203_exu_oitf         (core/e203_exu_oitf.v)         长指令跟踪表
  │         │    ├─ e203_exu_alu          (core/e203_exu_alu.v)          ALU
  │         │    │    ├─ e203_exu_alu_csrctrl  CSR读写控制
  │         │    │    ├─ e203_exu_alu_bjp      分支跳转
  │         │    │    ├─ e203_exu_alu_lsuagu   LSU地址计算
  │         │    │    ├─ e203_exu_alu_rglr     普通ALU运算
  │         │    │    ├─ e203_exu_alu_muldiv   乘除法
  │         │    │    ├─ e203_exu_alu_dpath    ALU数据通路
  │         │    │    └─ e203_exu_nice         NICE协处理器握手
  │         │    ├─ e203_exu_longpwbck    (core/e203_exu_longpwbck.v)   长指令写回
  │         │    ├─ e203_exu_wbck         (core/e203_exu_wbck.v)        写回仲裁
  │         │    ├─ e203_exu_commit       (core/e203_exu_commit.v)       提交控制
  │         │    │    ├─ e203_exu_branchslv   分支解析
  │         │    │    └─ e203_exu_excp        异常处理
  │         │    └─ e203_exu_csr          (core/e203_exu_csr.v)         CSR寄存器组
  │         │
  │         ├─ e203_lsu                   (core/e203_lsu.v)              存储访问单元
  │         │    └─ e203_lsu_ctrl         (core/e203_lsu_ctrl.v)         LSU控制逻辑
  │         │
  │         └─ e203_biu                   (core/e203_biu.v)              总线接口单元
  │
  └─ e203_srams                           (core/e203_srams.v)            SRAM顶层
       ├─ e203_itcm_ram                   (core/e203_itcm_ram.v)         ITCM RAM封装
       │    └─ sirv_gnrl_ram              (general/sirv_gnrl_ram.v)      通用RAM
       │         └─ sirv_sim_ram          (general/sirv_sim_ram.v)       RAM行为模型
       └─ e203_dtcm_ram                   (core/e203_dtcm_ram.v)         DTCM RAM封装
            └─ sirv_gnrl_ram → sirv_sim_ram  (同上)
```