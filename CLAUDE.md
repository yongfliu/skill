# RSIM - Realistic NIC 仿真器

## 项目概述
基于 SimPy 的 cycle-accurate Realistic SmartNIC/DPU 仿真器。

目标是精确建模现代高性能网卡的完整硬件 pipeline：从 Host PCIe 接口到 MAC 物理层，覆盖多种传输卸载引擎（RDMA/RoCEv2、TCP Offload、Raw Ethernet）、拥塞控制、流控与 QoS 调度。

当前已实现的核心子系统：
- **RDMA 传输引擎**: RoCEv2 RC (WRITE/READ/SEND/ATOMIC)、XRC、veRoCE (OOO DDP + SACK + 选择性重传)
- **拥塞控制**: DCQCN + Pathwise CC (per-ECMP-path 速率控制)
- **Raw Ethernet**: 零拷贝 RAW_PACKET 透传（TX CQE 由 EgressFTE 在 TM admission 后生成）
- **TCP TSO**: TX 分段卸载（Segmentation + L234 封装）；非末段清 PSH bit，确保 RSC 正确合并
- **TCP RSC**: RX 侧合并（Receive Side Coalescing）；零拷贝：每包 payload 立即 DMA 进 Host，NIC 仅缓存 RscContext (~100B) + saved_header (54B)；Mode A（header-split）/ Mode B（单 buffer）统一状态机
- **Virtio-net**: 北向接口前端，独立于 IB 队列机制。TX path: SplitVirtqPort(pipelined avail ring DMA via VRING_TAG) → VirtioEngine.run_tx(GS grant → VirtTxCmd) → VirtDMAFetch(frame DMA via VIRT_TX_TAG) → EgressFTE → wire；RX path: wire → IngressFTE → RxLookup(virt_exec_out fork) → VirtioEngine.run_rx(alloc_rx via VRING_TAG + DMAWriteJob(done_evt)) → RxDMAEngine(frame DMA via RX_EXEC_TAG=9) → host RX buffer → VirtUsedEvent → VirtioEngine.run_tx_complete；TX/RX completers 写 used ring via VRING_TAG HIF
- **FTE Pipeline**: IngressFTE（唯一协议解析点）/ EgressFTE（唯一 L234 封装点）
- **流控 & QoS**: GS Alpha Pipeline Admission + TM HQoS（Port L2 / TC L1 / Per-QP DRR+Shaper）+ veRoCE SACK

仿真拓扑：两个 NIC 实例通过 CdmiiSwitch (400G) 连接，驱动端运行 IB verbs 性能测试（ib_write_bw / ib_read_bw / ib_send_srq 等）。

## 工作流: Explore → Plan → Implement

非 trivial 任务（多文件修改、bug 调查、新功能）必须按三阶段执行，不得跳阶段直接写代码。

### 阶段 1: Explore
- **目标**: 充分理解现状，定位相关代码，收集事实
- **工具**: 用 `Explore` subagent 或直接 Grep/Read，**不写任何代码**
- **产出**: 向用户报告发现（涉及哪些文件、当前行为、根因假设）
- **检查点**: 用户确认理解正确后才进入 Plan

### 阶段 2: Plan
- **目标**: 设计修改方案 + 验证计划
- **工具**: 用 `EnterPlanMode` 或直接向用户描述方案
- **产出**:
  - 修改清单（文件:行号 + 改什么 + 为什么）
  - 验证计划（sanity 用例 + 覆盖场景 + 是否需要回归）
- **检查点**: 用户批准方案和验证计划后才进入 Implement

### 阶段 3: Implement
- **目标**: 执行方案，写代码，按验证计划跑测试
- **失败回退**: 测试 FAIL → 回到 Explore（带着失败信息重新调查）
- **Regression**: 按 Plan 中约定的范围执行，**跑回归前询问用户确认**

```
Explore → Plan → Implement → 测试
                                │
                           PASS → 更新 CLAUDE.md（无需提交）→ 完成
                           FAIL → 回到 Explore
```

### 阶段 4: 更新 CLAUDE.md（测试 PASS 后，无需 git 提交）
- **目标**: 将本次任务引入的架构变化同步到 CLAUDE.md，保持文档与代码一致
- **注意**: CLAUDE.md 已在 .gitignore 中，直接编辑保存即可，**不需要 git add/commit**
- **更新范围**（按需，不强制全覆盖）:
  - 新增/变更的模块职责 → `### 关键模块` 表格
  - 新增/变更的不变量或约束 → `### 关键约束与不变量`
  - 新增/变更的配置参数 → `## 关键性能参数` 表格
  - 新增/变更的流控机制 → 流控架构相关小节
  - 新增的测试用例 → `## 运行测试` 和回归列表
  - 若内容较多，移至 `docs/` 下独立文件，CLAUDE.md 保留 1-3 行摘要 + 链接
- **不更新**: 纯实现细节（函数签名、局部变量）、临时调试信息、已在 docs/ 详述的内容

### 豁免
- 单行修复、typo、用户给出了精确指令 → 可直接 Implement
- 纯探索/研究任务 → 只做 Explore，不需要 Plan/Implement

## 架构
```
Host Driver (SQM/RQM/CQM) <-> HIF (PCIe) <-> Transport Engine (rnic/te/)
                                                      ↕
                                             TxDPE / RxDPE
                                                      ↕
                              EgressFTE(TX) <-> TM(HQoS) <-> IngressFTE(RX)
                                                      ↕
                                               MAC <-> Wire
```

### 关键数据通路（四条主路径）

RDMA RC 传输由四条正交路径构成闭环，均从本地 NIC 视角描述：

**① TX Path（主动发送）** — 触发：Host 写 WQE + doorbell
- SQM(WQE解析) → TxEngine(GS/seg/ORD gate；写 _retrans_store；tx_track→RC) → TxDPE: Seg→DMA_READ(Host payload)→Assembly → EgressFTE(L234封装) → TM → Wire
- 产出：Wire 上的数据包

**② TX Backward（反向确认 / 销账）** — 触发：Wire 来了 ACK/NAK/SACK（无 payload）
- Wire → IngressFTE → TM → ACK bypass → RC.RcManager(滑动窗口推进；NAK/timeout→RetransCmd→TxEngine重传) → AckNotif/SackPreNotif → ME.TxAckBridge(读+删 _retrans_store) → TX CQE + seg/flight/ORD/GS credits 归还 + desc_id free
- ATOMIC 例外：TxAckBridge 执行 DMA_WRITE 将原子操作返回值写回 Host
- 产出：TX CQE（Host SQ CQ）+ 资源归还

**③ RX Path（接收交付）** — 触发：Wire 来了数据包（WRITE/SEND/READ_RESP，有 payload）
- Wire → IngressFTE(全量解析→PacketMetadata) → TM → RxDPE.RxParser → RxLookup(QPC+RWQE) → RxRouter(**数控分离**：ctx→ME.RxPSNMerge hold + PsnQuery→RC.PsnController→PsnVerdict→RxPSNMerge merge/enrich) → RxExecutor(SGL+MMU→PA) → DMAWriteJob→DMA_WRITE(Host Memory) + RxCommitEvent→CQM→RX CQE
- RscEngine(TCP RSC) 直连 RxExecutor，绕过 PsnController；Virtio 在 RxLookup 通过 virt_exec_out 分叉到 VirtioEngine，不经 RxRouter
- 产出：Host Memory 有数据 + RX CQE（Host RQ CQ）

**④ RX Respond（响应生成）** — 触发：③ 处理完后（RxCommitEvent→RC.event_chan）
- RC.RespExecutor(PSN状态机 + ECN/RNR检测 + 响应类型决策) → RespDecision → RC.RespBuilder →
  - ACK / NAK / SACK / CNP：bypass_ctrl_out → TM → Wire（纯控制包，无 DMA，快速旁路）
  - READ Response：ReadRespWork → ME.RxRspBridge → TxCmd → TxDPE: DMA_READ(本地 Host Memory，读对方请求的数据) → EgressFTE → TM → Wire
- 产出：Wire 上的响应包

**READ 操作串联全部四条路径**（INI 侧视角）：
① INI 发 READ request（小包，无 payload）→ TGT ③ 收到 → TGT ④ 发 READ Response（含数据）→ INI ③ DMA 写 Host → INI ② TX CQE + ORD release

**RAW 路径**（非 RDMA）: TX: EgressFTE 直通全帧 + TX CQE；RX: IngressFTE mac_to_qpn 查表 → RxDPE RAW fast path

### 关键模块
| 模块 | 路径 | 职责 |
|------|------|------|
| GS | `rnic/gs/` | 全局调度器 (gs.py + qc.py): Alpha Pipeline Admission, TM XOFF/XON 反压, GS Budget, Byte-Fair Grant |
| MsgEngine | `rnic/te/me/` | 消息基础设施容器（**纯 RDMA/TCP，无 Virtio**）: **TxEngine**①(TX WQE执行 + 写 `_retrans_store`)，**TxAckBridge**②(AckNotif/SackPreNotif→读+删 `_retrans_store`→CQE+credits+ATOMIC writeback+desc_id free)，**RxRspBridge**④(ReadRespWork→TxCmd→TxDPE，预扣 GS pipeline credit)，RxLookup(**virt_exec_out fork**：Virtio包直接分叉→VirtioEngine，不进 RxRouter)，RxRouter(**纯二路**：RSC|RDMA 数控分离)，**RxPSNMerge**(持有 ExecContext，等待 RC verdict，合并后转发)，RxExecutor，RscEngine(直连 RxExecutor 绕过 PSN gate)。RC/TCP 共用。4 个模块各对应一条路径：TxEngine①，TxAckBridge②，RxExecutor③，RxRspBridge④ |
| RcTransport | `rnic/te/rc/` | RC 可靠传输控制容器: **RcManager(纯滑动窗口状态机**：Retrans Buffer+ACK retire+重传命令；发 AckNotif/SackPreNotif 通知 ME 释放资源；不再持有 MMU/HIF/CQE/credit-release)，RespExecutor(PSN状态机+ACK/NAK/SACK/CNP决策), RespBuilder(ACK/NAK/SACK/CNP包构造，**不含** READ_RESP), **PsnController**(纯控制平面：接收 PsnQuery(64B metadata)，产出 PsnVerdict(64B scalars)；RC enrichment: READ_RESP reth填充+veRoCE rqmsn_map; PSN过滤+多包flow offset追踪；不接触 ExecContext/payload/SGL)。创建并拥有 4 个协议状态对象（`seg_credit`/`flight_window`/`ord_tracker`/`recovery`），top.py 将其传给 MsgEngine 共享 |
| TxDPE | `rnic/dpe/txdpe/` | TX 数据通路引擎: Segmentation, DMA Fetch, Assembly, EgressFTE。chan_virt_tx_cmd 作为 TE→DPE 接口 channel（VirtDMAFetch 读取）|
| RxDPE | `rnic/dpe/rxdpe/` | RX 数据通路引擎 (纯 DMA 执行层): RxParser(L234 strip) + RxDMAEngine(统一 DMA write, RX_EXEC_TAG=9) + run_dispatcher(RxCommitEvent路由)。无协议知识，无状态 |
| IngressFTE | `rnic/fte/ingress_fte.py` | **唯一协议解析点**: 全量解析 BTH+RETH/AETH/AtomicETH/XRCETH/veRoCE扩展头 → 填满 PacketMetadata；TM/RxDPE 完全协议无感知 |
| EgressFTE | `rnic/fte/egress_fte.py` | TX 出口封装: 从 PacketDesc (src_mac/dst_mac/src_ip/dst_ip) 构造 L234 头；RAW_PACKET 直通 + TX CQE |
| TM | `rnic/tm/` | 纯排队调度器（无协议感知）: HQoS TxScheduler (Port L2 → TC L1 → Per-QP L0/DRR), BufferManager, AdmissionCtrl, Rate-Proportional Per-QP XOFF/XON → GS |
| ICM | `rnic/icm/` | 内部上下文存储 (QPC cache 层级) |
| HIF | `rnic/hif/` | 主机接口 (PCIe DMA, DBR Scanner) |
| SQM | `rnic/qm/sqm/` | 发送队列管理器 (Prefetch, WQE Parse, Desc Pool Write) |
| RQM | `rnic/qm/rqm/` | 接收队列管理器 (SRQ prefetch/replenish) |
| SplitVirtqPort | `rnic/qm/vring/virtq_port.py` | **纯 QM 层**（类比 SQM，无 GS 交互）: 吸收原 VringPrefetcher + VringDescCache + VringDispatcher + TX/RX completers 全部逻辑。doorbell → pipelined 3-step DMA(avail_idx→ring_slot→desc_table) → 内部 `_tx_queue`；`dequeue_tx()` 返回 simpy Store.get() event；`alloc_rx()` generator（ring walk via VRING_TAG）；`complete_tx/complete_rx` 写 used ring via VRING_TAG HIF；`_tx_depth[qpn]` 背压计数器（上限 64/QPN）|
| VirtioEngine | `rnic/te/virtio/virtio_engine.py` | **TE 层 Virtio 独立容器**（peer to MsgEngine）: `run_tx()` — VirtqDescMsg → GS grant → VirtTxCmd；`run_rx()` — ExecContext(来自 RxLookup virt_exec_out) → alloc_rx → DMAWriteJob(done_evt) → complete_rx；`run_tx_complete()` — VirtUsedEvent(来自 EgressFTE) → complete_tx。暴露 `register_virtio_qp()` 和 `notify_tx()` 供 driver 调用（兼容旧 API）|
| VirtDMAFetch | `rnic/dpe/txdpe/virt_dma_fetch.py` | TxDPE 子组件: 从 VirtTxCmd 取 frame_pa，HIF DMA_READ(VIRT_TX_TAG)，注入 PendingFetchEntry → chan_asm2fte |
| RxDMAEngine | `rnic/dpe/rxdpe/dma_engine.py` | RxDPE 统一 DMA 写引擎：接收 DMAWriteJob(from RxExecutor/RscEngine/VirtioEngine.run_rx)，fire-and-forget HIF DMA_WRITE(RX_EXEC_TAG=9)，支持 done_evt 回调，排干 hif_resp_chan |
| Top | `rnic/top/` | 顶层连线 (top.py), 配置, monitor, metrics |

### 模块职责定位

| 模块 | 角色定位 | 核心职责 |
|------|---------|---------|
| **GS** | 发射器 (Issuer) | 全局资源守门人。Select: TM Per-QP XOFF/XON (rate-proportional)。Grant: Alpha per-QP + TC Quota + GS Budget。READ 仅受 gs_budget 限流。模块: `gs.py`(调度) + `qc.py`(QuotaController 资源管理) |
| **SQM** | 译码器 (Decoder) | 自治 decoder。将 Raw WQEBBs 翻译为结构化 Metadata，自行分配 desc_id，无 GS 交互。Desc pool 背压提供自然流控。 |
| **vring/** | 描述符预取器 (Prefetcher) | 纯 QM 层（类比 SQM）。SplitVirtqPort: doorbell → avail_ring.idx(1 DMA) → N 并行 ring[slot] reads → N 并行 desc_table reads(每个 ring 响应到达后立即发出，硬件 pipeline) → 内部 `_tx_queue`。**无 GS 交互**，GS admission 在 VirtioEngine 侧。 |
| **MsgEngine** | 消息基础设施 | **纯 RDMA/TCP**，协议无关消息处理层，四个子模块各对应一条路径。**TxEngine①**: budget-gated WQE 执行 + ORD 控制，写 `_retrans_store[qpn][psn]`；**TxAckBridge②**: 消费 `AckNotif`(CQE生成/ATOMIC writeback/ORD release/seg·flight·gs credit释放/desc_id free) 和 `SackPreNotif`(SACK Phase 1.5 预释放)；**RxRspBridge④**: 消费 `ReadRespWork`，预扣 pipeline credit(-9)，组装 TxCmd 推 TxDPE。RX 侧：RxLookup(**Virtio fork → virt_exec_out**)→RxRouter(**纯二路**: RSC|RDMA 数控分离)→RxPSNMerge→**RxExecutor③**；RscEngine 旁路 PSN gate 直连 RxExecutor。_retrans_store 由 TxEngine 写，TxAckBridge 和 TxEngine.run_bridge_retrans 读，三方共享同一 dict 引用。 |
| **RcTransport** | RC 可靠传输控制 | 完整 RC 双向闭环。**RcManager: 纯滑动窗口状态机** — Retrans Buffer + ACK retire + 重传命令；发聚合 `AckNotif{qpn,ack_psn,atomic_payload,is_error}` 通知 ME 释放资源（不再持有 MMU/HIF/CQE/credit-release）；SACK Phase 1.5 发 `SackPreNotif{qpn,psn_list}` 给 ME 预释放；`_send_ack_notif()` 统一发送路径（含 is_error=True rollback）；`get_read_match()` 供 PsnController 同容器直接调用。RespExecutor: PSN 状态机 + ECN/BECN/CNP 决策。**PsnController**: ① RC enrichment（READ_RESP 填充 reth via get_read_match()、veRoCE rqmsn_map 管理）② PSN 过滤 ③ 多包 flow offset追踪（_flow 只存标量，无 SGL），产出 PsnVerdict 委托 ME 执行 QPC mutation 和 flow_ref 组装。创建 4 个共享状态对象：`SegCreditPool`(ME alloc→ME release via AckNotif)、`FlightWindow`(ME add→ME remove via AckNotif)、`OrdTracker`(ME alloc→ME release via AckNotif.release_ord=True)、`RecoveryTracker`(RC set→ME read)，通过 top.py 传给 MsgEngine。 |
| **DPE** | 搬运工 (Worker) | 无状态 pass-through。DMA 搬运与组包发送。Assembly 无 credit 门控。**TX credit 归还由 ME.TxAckBridge 完成**（不经 RcManager）。**RX DPE 职责**：RxParser(L234 strip) + RxDMAEngine(执行 DMAWriteJob，RX_EXEC_TAG=9) + run_dispatcher(路由 RxCommitEvent 到 CQM 或 RcManager)。所有协议决策上移至 MsgEngine/RcTransport。 |
| **IngressFTE** | 唯一协议解析器 (Parser) | **全量解析** BTH + RETH/AETH/AtomicETH/XRCETH/MSNETH/POETH/RQETH/SACKETH → `PacketMetadata` 所有字段。TM pending buffer 模式：`_run_rx_admit`(AXI 重组) → FTE(1ns) → `_run_rx_route`(路由)。RxDPE 仅消费 meta 字段，不再解析协议。 |
| **EgressFTE** | 出口封装器 (Encap) | Assembly 之后、TM 之前。从 `PacketDesc`(src_mac/dst_mac/src_ip/dst_ip) 构造 42B L234 头，拼接 IB packet 推入 TM。RAW_PACKET 直通 + 在 TM admission 后生成 TX CQE。QPC 中新增 L2/L3 寻址字段，TxEngine→Segmentation→PacketDesc 全链路透传。 |

## 当前架构状态

### 流控架构

**GS 层 (全局准入)**:

| 阶段 | 机制 | 容量 | 保护对象 | 信号/归还 | READ |
|------|------|------|---------|----------|------|
| Select | TM Per-QP XOFF/XON (rate-prop) | rate × 1us XOFF / 0.5us XON | 单 QP 垄断 TM VOQ | TM 水位信号 `(-6, qpn, flag)` | **跳过** |
| Grant.1 | Alpha per-QP (proactive) | `max(1KB, alpha × remaining / N)` | per-QP pipeline 深度公平 | TM admission `(-4, tc, qpn, bytes)` | **跳过** |
| Grant.2 | TC Quota | 32KB/TC | 防止单 TC 垄断 pipeline | TM admission `(-4, tc, qpn, bytes)` | **跳过** |
| Grant.3 | GS Budget (`gs_budget_bytes`) | 1MB | 全局网络并发 | TxAckBridge AckNotif 处理 (`(qpn, bytes)`) | 扣减 (唯一全局 gate) |

**TE 层 (本地执行)**:

| 层 | 机制 | 容量 | 保护对象 | 归还时机 |
|----|------|------|---------|---------|
| TE.0 | ORD (OrdTracker, READ only) | 32/QP | READ 并发 | TxAckBridge.run_bridge_ack_notif (release_ord=True) |
| TE.1 | Per-QP Window (FlightWindow) | 256KB | per-QP flight | TxAckBridge.run_bridge_ack_notif (AckNotif) |
| TE.2 | Per-QP Seg Credit (SegCreditPool) | 256/QP | per-QP seg 深度 | TxAckBridge.run_bridge_ack_notif / run_bridge_sack_pre |

**READ 豁免原理**: READ request 是小包 (~64B)，不消耗 TX pipeline/TC 带宽。实际数据传输在 RX 路径（远端发送 READ response）。READ 并发由 TE ORD 本地控制。

### GS Alpha Pipeline Admission

GS 使用 alpha 算法主动控制 per-QP pipeline 深度，同时用 tc_quota 限制 per-TC pipeline 深度：

```
threshold = max(min_threshold, alpha × (port_quota - port_inflight) / N_active_qps)
```

```
GS (主动准入)                Pipeline              TM (被动反馈)
┌──────────────────────┐    ┌──────────┐          ┌──────────────────────┐
│ Alpha per-QP         │    │          │          │                      │
│ tm_quota (32KB)      │    │TE→Seg→   │──pkt──> │ VOQ (per-TC/per-QP)  │
│ gs_budget (1MB)      │─grant─>│DMA→Asm │          │  Rate-prop XOFF/XON  │
│                      │    │          │<─(-4)──│  TM admission (put)   │
│ _tm_bp_set           │<─(-6)──│          │<─(-6)──│  XOFF/XON signal     │
└──────────────────────┘    └──────────┘          └──────────────────────┘
```

| 参数 | 文件 | 默认值 | 说明 |
|------|------|--------|------|
| `tm_quota_bytes` | `gs/config.py` | 32KB | Pipeline 深度（port total + per-TC cap） |
| `port_alpha` | `gs/config.py` | 1.0 | Per-QP 公平共享比例 |
| `port_alpha_min_threshold` | `gs/config.py` | 1KB | 最低阈值 = 1 MTU floor |
| `per_qp_voq_xoff_ns` | `tm/config.py` | 1000 | TM per-QP XOFF: rate × 1us |
| `per_qp_voq_xon_ns` | `tm/config.py` | 500 | TM per-QP XON: rate × 0.5us |

- **GS 主动 (proactive)**: Alpha 算法在 Grant Stage 检查 per-QP pipeline inflight，tc_quota 限制 per-TC 深度
- **TM 被动 (reactive)**: 静态水位 XOFF/XON 信号，反馈哪些 QP 的 VOQ 深度过大
- **GS Select Stage**: 检查 `_tm_bp_set`，XOFF 触发 park → XON 信号触发 unpark
- **READ**: 跳过 Alpha/TC Quota/XOFF 检查（READ request 小包，实际数据在 RX 路径）
- **Safety net**: gs_budget (1MB) 限制全局网络并发（主要保护 READ）

### 守恒闭环

- **gs_budget**: `GS_deduct = TxAckBridge.run_bridge_ack_notif(AckNotif) → gs_credit_return_chan.put((qpn, bytes))`
- **tm_quota (port_inflight + tc_quota + qp_inflight)**: `GS_deduct → TM_admission_return(-4, tc, qpn, bytes)`
- **TM Per-QP XOFF/XON**: 无 credit 守恒（TM 自主监控 VOQ 字节数，静态水位）

**WRITE 4096B 守恒**:
| 事件 | gs_budget | port_inflight | tc_quota | qp_inflight |
|------|-----------|---------------|----------|-------------|
| Grant | -4096 | +4096 | -4096 | +4096 |
| TM Admission | — | -4096 | +4096 | -4096 |
| ACK | +4096 | — | — | — |

**READ 4096B 守恒**:
| 事件 | gs_budget | port_inflight | tc_quota | qp_inflight |
|------|-----------|---------------|----------|-------------|
| Grant | -4096 | 不扣减 | 不扣减 | 不扣减 |
| ACK | +4096 | — | — | — |

**READ_RESP 4096B 守恒**:
| 事件 | gs_budget | port_inflight | tc_quota | qp_inflight |
|------|-----------|---------------|----------|-------------|
| RxRspBridge (-9) | 不扣减 | +4096 | -4096 | +4096 |
| TM Admission (-4) | — | -4096 | +4096 | -4096 |
| (无 ACK) | — | — | — | — |

### TM HQoS 调度

TM TxScheduler 实现 HQoS 树形调度：

| 层 | 机制 | 说明 |
|----|------|------|
| L2 | Port Shaper | 物理端口令牌桶（rate_gbps × 0.125 B/ns） |
| L1 | Per-TC Scheduler | SP (TC7) + RR (TC 0-6)，L1 令牌桶限速 |
| L0 | Per-QP DRR + Shaper | DRR 公平调度 + per-QP 令牌桶整形 (rate_mbps from pkt metadata) |

- **DRR**: 每 TC 内 per-QP 子队列，deficit quantum = 4096B，保证字节公平
- **Per-QP Shaper**: 令牌桶，rate 从包 metadata 动态更新，max_burst = 9000B
- **Static Per-QP XOFF/XON**: TxQueueMgr 在 `put()`/`commit_egress()` 时用静态水位检测 per-QP VOQ 深度，向 GS 发送 XOFF/XON 信号
- **tm_quota Credit Return at TM Admission**: TxQueueMgr.put() 在包进入 VOQ 时立即归还 `(-4, tc, qpn, bytes)` 给 GS（不等待物理发送）

### 共享 Credit Channel

`gs_credit_return_chan` (simpy.Store) 统一接收消息：

| 消息格式 | 来源 | 回补目标 |
|----------|------|---------|
| `(-4, tc, qpn, byte_cnt)` | TM TxQueueMgr.put() (at admission) | 归还 tc_quota + port_inflight + qp_inflight |
| `(-6, qpn, flag)` | TM TxQueueMgr | Per-QP XOFF/XON (rate-proportional, flag: 0=XOFF, 1=XON) |
| `(-9, tc, qpn, byte_cnt)` | RxRspBridge.run() (READ_RESP 发送前) | 预扣减 port_inflight + tc_quota + qp_inflight（与后续 -4 归还配对，gs_budget 不动）|
| `(qpn, byte_cnt)` | TxAckBridge.run_bridge_ack_notif (AckNotif) | gs_budget |

**已消除**: Assembly per-TC credit (由 GS Alpha + TM XOFF 替代), `(-8, 0, flag)` Port XOFF/XON, `(-7, tc, flag)` TC XOFF/XON, `(-5, qpn, bytes)` Per-QP Quota (合并入 -4), `(-1, bytes)` pipeline_credit (由 alpha 替代), `(-3, qpn, count)` ORD (TE 本地), `int(bytes)` SQM refund (SQM 自治)

### ICM 三通道架构
```
[Walker]       → chan_icm_mtt(8)   → mtt_lock(1 port)  → shared L1/L2 cache
[QPC readers]  → chan_icm_req(64)  → qpc_lock(3 ports) → shared L1/L2 cache
[dirty writes] → chan_icm_dirty(32) → no lock           → dirty_map only (O(1))
```
- L0=64/client, L1=256, L2=1024, MSHR=16
- SchedulerCache=**8192**（**必须 ≥ 活跃 QPs**，否则 arbiter 抖动洪泛 ICM）
- QPC 转发链: Prefetcher → Executor → Dispatcher → TeReq → Fetcher L0

**ICM context type_tag**:

| type_tag | 类型 | host-memory 后端 | 备注 |
|----------|------|-----------------|------|
| 0 | QPC | PCIe DMA (QPC_BASE) | read/write |
| 1 | SRQC | PCIe DMA (SRQC_BASE) | read/write |
| 2 | MTT | PCIe DMA (MTT_BASE) | read-only，不入 victim buffer |
| 3 | OooBlock | `_ooo_store` dict（进程内，零延迟）| index=qpn；RespExecutor 独占使用 |

- **OooBlock (type_tag=3)**: 替代原 `OooBlockPool`（固定 256 slot）。cache miss 直接从 `_ooo_store.get(qpn)` 取或新建空 block，无 PCIe RTT。脏写回 → `_ooo_store[qpn] = block`（同进程内存，无 DMA）。
- **`qpc.is_ooo`** 保留为快路标志：in-order 包跳过 ICM lookup；仅在首个 OOO 包到达时置位，psn_map 清空时清零。

### ConnectX-Style Doorbell (DBR)
- Driver: CPU store tail → DBR[qpn] (主机内存), burst 后 send_ding (PCIe MMIO)
- RNIC HIF: ding → DBR Scanner → DMA read 8B → dispatch doorbell

### XRC 传输
- XRC INI QP (仅 SQ) + XRC TGT QP (SRQ, 无 SQ)，`bth.tver=5` 标识
- XRCETH 存在于所有 XRC 请求 opcodes，SRQ 基础设施完全复用

### veRoCE 协议 (OOO DDP + SACK + Selective Retransmit)

veRoCE 在标准 RoCEv2 RC 基础上扩展了 Out-of-Order DDP、Selective ACK 和选择性重传，通过 `bth.tver=6` 标识。新增 opcode `RC_SACK`(0x18) 和协议头 MSNETH/POETH/RQETH/SACKETH；新增 8 个 QPC 字段（`veroce`、`sq_msn`、`sq_send_msn`、`hpsn`、`write_va`、`write_rkey`、`is_ooo`、`ooo_state_ptr`）。详见 [docs/veRoCe.md](docs/veRoCe.md)。

---

### Pathwise CC + 多路径

Per-(QP, path) DCQCN：每 QP 支持 N 条 ECMP 路径，拥塞只影响拥塞路径的速率 R_k，不误伤其他路径。`curr_rate = Σ R_k`，TM per-QP pacer 不变。

**QPC 新字段** (`common/protocol.py` `HardwareQPC`):
- `num_paths` (4b)：ECMP 路径数，0/1 = 单路径（退化为原始行为）
- `base_udp_port` (16b)：UDP src port 基址；path_id=k 的包使用 `base_udp_port + k`

**DCQCN per-path 状态** (`rnic/cc/dcqcn.py` `DCQCNState`):
- `path_rates[]`, `path_target_rates[]`, `path_alphas[]` 替代原来的单标量
- `curr_rate` 是 Python `@property`，返回 `sum(path_rates)`（向后兼容）
- `on_cnp_recv(qpn, qpc, path_id=0)`: MD 只作用于 `path_rates[path_id % N]`
- `_recovery_timer()`: per-path AI，`ai_step = R_AI / num_paths`；每路径上限 `line_rate / num_paths`
- `get_path_rates(qpn) -> list`: 供 Segmentation WRR 查询

**路径选择 — Deficit WRR** (`rnic/dpe/txdpe/segmentation.py`):
- `SegmentationUnit.__init__` 接受 `cc=None`；`self._path_deficit: dict[int, list[float]]`
- `_select_path_wrr(qpn, num_paths)`: 每次选路前 `deficit[k] += R_k / ΣR_k`，选 deficit 最大的路径并 `-= 1.0`；cc=None 或 num_paths≤1 退化为始终返回 0
- 每个 `PacketDesc` 创建时写入 `path_id` 和 `base_udp_port`（从 `TxCmd.num_paths`/`TxCmd.base_udp_port` 传入，TxEngine 从 QPC 读取填充）

**UDP src port 路径编码** (`rnic/dpe/txdpe/assembly.py`):
```python
udp_src = desc.base_udp_port + desc.path_id
full_pkt = build_rocev2_l234(len(ib_pkt), src_port=udp_src) + ib_pkt
```
`build_rocev2_l234` 新增 `src_port=0` 参数（`common/protocol.py`）。

**CNP outband 路径 echo 链** (`cc_notify_mode='outband'`，默认):
```
RX包头[34:36] → Parser(meta.udp_src_port)
             ↘ ACK bypass(TM): bypass_pkt.udp_src_port (L234剥离前提取)
RespExecutor ← (meta.udp_src_port)
  → RespDecision.cnp_udp_src
  → RespBuilder: op.udp_src_port
  → TM._bridge_ctrl_tx_loop: build_rocev2_l234(src_port=udp_src_port)
  → wire → 发送侧 Parser → RcManager._process_cnp
  → path_id = (udp_src - base_udp_port) % num_paths
  → cc.on_cnp_recv(qpn, qpc, path_id)
```

**INBAND BECN 路径** (`cc_notify_mode='inband'`):
```
RespExecutor ECN检测 → _pending_becn=True (不推单独RC_CNP)
  → decision.becn=True, decision.becn_udp_src=cnp_udp_src
  → RespBuilder: BTH b_bit=1, udp_src_port=becn_udp_src
  → wire → 发送侧 RxDPE(run_ack_bypass): info['becn']=True
  → RcManager._process_ack: 调用 _process_cnp(info)
  → cc.on_cnp_recv(qpn, qpc, path_id)
```
- `TeRespConfig.cc_notify_mode` 控制模式，默认 `'outband'`；由 `rnic.te_rc.resp_executor.cc_notify_mode` 访问
- 注意 `_pending_becn` 是独立 bool（不用 `_becn_udp_src` 判断，后者为 0 时也需生效）

**实现注意事项**:
- TM `_run_rx_route` 在发往 ACK bypass channel 前用 `meta.decap_offset` 剥离 L234，UDP src port 必须在剥离前从 `pkt.data[34:36]` 提取并附加到 `bypass_pkt.udp_src_port`
- **PacketMetadata 总线**: `TLM_Transaction.meta` 默认构造为空 `PacketMetadata()`。IngressFTE 填充后由 TM 路由；RxParser 读取 `meta.decap_offset` 跳过 L234 扫描（fallback: `meta.decap_offset==0` 时退化到常量 `ROCEV2_L234_LEN`）
- `TxDPE` 在 `top.py` 中早于 `DCQCNController` 实例化，不可在构造函数传入 `cc`；改为在 DCQCN 创建后 `self.tx_dpe.stage_seg.cc = self.dcqcn` 后赋值
- `num_paths=0/1` 时：path_id 始终为 0，`_select_path_wrr` 直接返回 0，`on_cnp_recv` 的 path_id 也为 0 → 行为与原 DCQCN 完全相同，所有回归测试 PASS

### 关键约束与不变量
- `sched_cache_capacity` 必须 ≥ 活跃 QPs（否则 ICM 洪泛，详见 [architecture-defects.md](docs/architecture-defects.md)）
- `gs_budget_bytes` 必须 ≥ wire BDP (~10us × 400Gbps ≈ 500KB)，512KB 默认值匹配
- **READ 并发控制**: GS budget 是 READ 唯一全局 gate。TE ORD 控制 per-QP READ 并发。Alpha/TC Quota/XOFF 均跳过 READ
- **gs_budget 守恒**: `GS_deduct = TxAckBridge.run_bridge_ack_notif → gs_credit_return_chan.put((qpn, bytes))`
- **pipeline 守恒**: `GS_deduct(port_inflight + tc_quota + qp_inflight) = TM_admission(-4, tc, qpn, bytes)` (WRITE/SEND: via GS grant；READ_RESP: via RxRspBridge `(-9, tc, qpn, bytes)` 预扣减)
- **GS Alpha (proactive)**: `threshold = max(min, alpha × remaining / N)` 在 Grant Stage 检查 per-QP pipeline inflight
- **TM Rate-Proportional XOFF/XON (reactive)**: 水位 = qp_rate × time_ns (1us XOFF / 0.5us XON)，TM 自主监控 per-QP VOQ 字节数，向 GS 发送 XOFF/XON 信号
- **ORD 在 ME**: `OrdTracker.alloc(qpn)` 在 TxEngine._execute_slice；`OrdTracker.release(qpn)` 在 TxAckBridge.run_bridge_ack_notif（`notif.release_ord=True`，RC `_process_read_resp` 发出）。RcManager 不再持有 OrdTracker 引用
- **Seg credit 守恒**: `alloc_seg()` 次数 = `release_seg()` 次数（per-QP + global 双重计数）。归还由 `TxAckBridge.run_bridge_ack_notif` 和 `run_bridge_sack_pre` 完成（基于 `_retrans_store[psn]['is_slice_last']` 计数）；`pre_released=True` 的条目在 ack_notif 中跳过，避免重复归还
- **TX 数据平面字段分离（SGL 预计算）**: RC 的 `unacked_window` entry 仅存控制标量（psn, length, flight_bytes, timestamp, is_last, is_slice_last, mtu, sq_cqn, wqe_idx, opcode；READ 加 read_segments）；SGL/WQE/offset/inline_data/dest_qpn/xrc_srq_num/msn/rqmsn/desc_id 存入 ME 的 `_retrans_store[qpn][psn]`（ME 内部，不跨容器）。重传走 `RetransCmd{qpn,psn,qpc}` (64B thin)，ME 从 `_retrans_store` 重建 LogicContext 并 re-DMA；READ `get_read_match()` 使用预计算的 `read_segments[(va,rkey),...]` 实现 O(1) 查表。Seg Credit Pool 全局共享 (max_seg_entries=4096)，gs_budget 天然约束实际并发
- **Budget-split WQE 排序**: TE 在 byte_budget 耗尽时 yield WQE，必须 `appendleft` 重新入队（HEAD of pending），否则违反 per-QP 包排序导致 WRITE 数据损坏
- **rate_mbps 传播**: QPC.curr_rate (= Σ path_rates[k]) → TxEngine → TxCmd → Segmentation → PacketDesc → Assembly → TLM_Transaction → TM per-QP shaper
- **SQM 自治**: SQM 自行分配 desc_id，无 GS grant 交互。GS 向 TE 发送 ByteBudgetGrant，不再经 SQM
- **GS 3 种 Parking Lot**: `_parked_on_tm_bp` (Per-QP XOFF) / `_parked_on_pipeline` (Alpha + TC Quota) / `_parked_on_budget` (GS Budget)
- SRQ 测试: 最小 `srq_depth=4096`，`alloc_timeout_ns=2000`（避免 RNR 级联）
- 稳态测试需 ≥64 msgs/QP（避免 ramp-up 伪影）
- **veRoCE OOO state 由 ICM 管理 (type_tag=3)**: OooBlock 以 qpn 为 index 存入 ICM（L2=1024），overflow 溢到 `_ooo_store` dict（进程内，无上限）；不再有固定 pool 容量限制。`is_ooo=False` 时跳过 ICM lookup（快路）
- **veRoCE SACK bitmap 范围**: 128-bit 覆盖 PSN [amsn+1, amsn+128]；超出范围的 PSN 视为 in-flight，不重传
- **veRoCE sack_acked 预释放**: SACK bitmap 确认的条目在 Phase 1.5 由 RC 发 `SackPreNotif{qpn,psn_list}` → ME 的 `run_bridge_sack_pre` 处理：标 `pre_released=True`、生成 CQE（若 `is_last`）、释放 seg/flight/gs credits。`AckNotif` 仍在 Phase 1 cumulative ACK 推进时发送（`ack_psn=max_ack_psn`），`run_bridge_ack_notif` 遇到 `pre_released=True` 时跳过 CQE+credit（只做 desc_id cleanup），避免双重释放。**不可在 Phase 1.5 发 AckNotif**——否则会提前清除 _retrans_store 中 bit=0 的空洞条目，导致空洞重传时找不到 store entry
- **veRoCE RTO timer reset**: `_selective_retransmit` 之后必须重置 RTO timer，否则 stale timer 误触发 go-back-N
- **veRoCE rqmsn_map 清理**: SEND_LAST 后必须从 `rqmsn_map[qpn]` 删除条目，否则内存持续增长。map 由 **PsnController**（RC）拥有，通过 `_update_rqmsn_map()` 维护；RxLookup（ME）不再持有任何 RC 状态
- **veRoCE WRITE VA 缓存**: `qpc.write_va` 在 WRITE_FIRST 写入，WRITE_LAST 清零；OOO 中间包依赖缓存 VA + POETH.po 定位 DMA 目标
- **TCP RSC Zero-Copy**: RscEngine 的 `payload_dma_pa` 是 payload 首字节的真实 PA（Mode B 已预加 base_offset=54），`_dma_payload` 仅追加 `coalesced_len` 偏移；禁止 NIC SRAM 缓存 payload。
- **Virtio IB 隔离**: Virtio QPN 由 `virtio_qpn=1` 标识；RxLookup 跳过 Virtio RWQE alloc，并通过 `virt_exec_out` 直接分叉 ExecContext 到 VirtioEngine（不经 RxRouter）；SQM/RQM/CQM 零修改
- **Virtio RX TM credit 归还**: RxLookup 在 Virtio 包走 `virt_exec_out` 分叉后，必须立即 `env.process(credit_chan.push(1))` 归还 TM RxScheduler credit（RxExecutor 通常在处理完每包后归还，但 Virtio 绕过了 RxExecutor）；若遗漏，`TxDPE→TM` StreamChannel 会在 2ms 后触发 Pull starvation 错误
- **vring/ 纯 QM 原则**: SplitVirtqPort 不做 GS 交互；GS demand/grant 在 VirtioEngine.run_tx() 侧。SplitVirtqPort 只管描述符预取与 used ring write-back
- **Virtio TX pipeline 不变量**: SplitVirtqPort 在 `_tx_depth[qpn]<64` 时持续预取（pipelined 3-step DMA）；VirtioEngine.run_tx 每次 dequeue_tx() 一个 VirtqDescMsg → 发一次 GS demand → 等一次 grant → 发一次 VirtTxCmd（串行，GS 天然背压）
- **Virtio GS 路径**: VirtioEngine.run_tx 必须持有 ByteBudgetGrant（`gs_demand_chan` → GS → `gs_budget_store`）才能发送 VirtTxCmd；WQEOpcode.RAW_ETH → is_raw=True → 跳过 gs_budget 扣减
- **Virtio used ring 串行**: TX/RX completer 各为单 coroutine，顺序写 used ring，无并发写冲突
- **Virtio PCIe client 两层分工**: VIRT_TX_TAG(10) NP-track — VirtDMAFetch TX frame DMA_READ；VRING_TAG(12) NP-track — SplitVirtqPort ring metadata (avail ring, desc table, used ring, RX ring walk)。Virtio RX frame DMA 合并入 RX_EXEC_TAG(9)（RxDMAEngine 统一处理，DMAWriteJob(done_evt) 机制）。两类均竞争 PCIe tags，无 synthetic latency
- **VRING_TAG shared HifClientPort**: top.py 创建单个 `HifClientPort(vring_in, vring_out)` 传给 SplitVirtqPort；seq_id 唯一标识并发读请求；writes 用 `submit()`（fire-and-forget）
- **RSC TSO PSH 前提**: `segmentation.py` 在非末段清 PSH (& ~0x08)，否则每段都会触发 flush，RSC 无合并效果
- **RSC RWQE 自分配**: RxLookup 跳过 TCP 包的 RQM alloc（TCP 非 IB 消费者）；RscEngine 在首个 TCP 包（新 RSC context 建立时）自行调用 RQM alloc，使用私有 `_rqm_resp_chan`（`resp_chan` 字段路由，不抢 RxLookup 响应）；flush 时 RxCommitEvent 携带该 RWQE 供 CQM 回收
- **RX TE/DPE 边界**: TE（MsgEngine + RcTransport）做所有协议决策 → 产出 `DMAWriteJob{pa_list}` 和 `RxCommitEvent`。DPE（RxDPE）做纯 DMA 执行：`RxDMAEngine`（`RX_EXEC_TAG=9`）是唯一 RX DMA write 执行点（RDMA + TCP RSC + Virtio 三路统一）。`hif_resp_chan`（`rx_exec_out`）由 RxDMAEngine 排干，防止 HIF hub 阻塞
- **DMA CQE 顺序**: `push(DMAWriteJob)` 之后立即 `push(RxCommitEvent)`；DMAEngine 的 fire-and-forget submit 保证 CQE 不先于 DMA write 到达 CQM
- **RX 路由架构（数控分离，ME/RC 跨容器）**: RxLookup 输出：`virtio_qpn` → `virt_exec_out` → VirtioEngine(**绕过 RxRouter**)；其余 ctx → RxRouter(纯二路：`rsc_enabled && TCP` → RscEngine→直连 _chan_exec_in→RxExecutor；其余 RDMA → **数控分离**：ctx→ME._chan_ctx_hold，PsnQuery→RC.PsnController→PsnVerdict→ME.RxPSNMerge→_chan_exec_in→RxExecutor)。**7条跨容器 channel** 在 top.py 创建：TxTrack、Retrans、PsnQuery、PsnVerdict、ReadResp、**AckNotif** (`chan_rc2me_ack_notif`)、**SackPreNotif** (`chan_rc2me_sack_pre`)；PsnController 直接调用 `rc_mgr.get_read_match()` (O(1) 标量查表)
- **READ_RESP 路由（RespBuilder → RxRspBridge）**: RespBuilder（RC）构造 `ReadRespWork` struct 推入 `chan_rc2me_read_resp` → ME.RxRspBridge.run() 接收后：(1) 发送 `(-9, tc=0, qpn, bytes)` 到 `gs_credit_return_chan` 扣减 pipeline credit（`port_inflight/tc_quota/qp_inflight`），确保后续 TM admission `(-4)` 归还时平衡；(2) 调用 `build_hdr_template()` 组装 TxCmd 推入 `chan_tx_cmd`。gs_budget 不扣减（READ response 无 ACK 归还）。这消除了 RespBuilder 对 `header_builder.py` 的依赖，并修复了原来 RespBuilder 直接推 TxCmd 导致的 GS pipeline 计数器漂移 bug。RxRspBridge 是 ④ RX Respond 在 ME 的执行侧（独立于 TxEngine，职责单一）
- **数控分离不变量**: ExecContext（含 payload_ref/SGL 引用）永不进入 RC 容器（rnic/te/rc/）；PsnController 不导入 `ExecContext` 或 `RxCommitEvent`；PsnController._flow 只存 offset/va 标量，无 SGL 引用；QPC mutation（write_va 等）通过 PsnVerdict 指令委托 ME.RxPSNMerge 执行；bypass RxCommitEvent 由 ME.RxPSNMerge 构造（语义不变，执行侧移至 ME）
- **数控分离顺序保证**: RxRouter 先 push ctx 后 push query（同 coroutine 内顺序），RC.PsnController 按 per-QP FIFO 处理，因此 verdict 到达 RxPSNMerge 时对应 ctx 必已在 holder 队列中。seq_id 是 RxRouter 分配的 per-QP 单调计数器，用于 RxPSNMerge 匹配验证
- **TE/DPE 层隔离**: `rnic/te/` 和 `rnic/dpe/` 不得跨层 import；`ExecContext`/`RxCommitEvent` 放 `common/rx_defs.py`（中立共享区）；`top.py` 是唯一跨层连线点
- **ME/RC 容器所有权**: 4 个协议状态对象（`seg_credit`/`flight_window`/`ord_tracker`/`recovery`）由 RcTransport 创建并暴露为实例属性（`rnic.te_rc.seg_credit` 等），top.py 通过独立参数传给 MsgEngine 共享；`chan_ver_to_gen`（RespExecutor→RespBuilder）封装在 RcTransport 内；`_retrans_store[qpn][psn]` 由 WqeEngine 拥有，`_execute_slice()` 写入（含 `flight_bytes`/`byte_cnt_for_cqe`/`is_last`/`is_slice_last` 等字段），`run_bridge_ack_notif()` 和 `run_bridge_sack_pre()` 消费；**TX 数据平面资源生命周期全在 ME**：CQE 生成 (`_gen_cqe_from_store`)、ATOMIC writeback (`_atomic_writeback`)、seg/flight/gs credit 释放、desc_id 释放均由 WqeEngine 完成；RC 不持有 `tx_data_pool`/`mmu_port`/`hif_port`/`cq_event_chan`；AckNotif 携带 `atomic_payload`（ATOMIC 8字节返回值）、`is_error`（rollback：跳过 CQE 但仍释放 credits）、`release_ord`（READ 完成时 ME 释放 OrdTracker slot）
- **RxLookup 协议无关**: RxLookup（ME）仅做 QPC lookup + RWQE alloc + RNR 检测，不持有任何 RC 状态。READ_RESP 填充（`reth`）和 veRoCE rqmsn_map 均由 **PsnController**（RC）通过 PsnVerdict 完成；`get_read_match()` 是 PsnController 到 RcManager 的直接函数调用（同容器，无 SimPy yield）
- **ICM client_id**: OooBlock (type_tag=3) 读写使用 `client_id='RespExecutor'`（不是旧的 `'Verifier'`）；ICM router 按 client_id 路由响应，字符串必须与组件注册名一致
- **测试直接注入约束**: 单元测试向 `chan_input`/`chan_ack_bypass` 直接注入包时，必须预填 `trans.meta`（PacketMetadata 字段），因为 RxParser 是纯 strip pass（不解析原始字节）。最低要求：`meta.opcode`、`meta.qpn`、`meta.psn`、`meta.decap_offset=ROCEV2_L234_LEN`；ECN 测试还需 `meta.ecn_marked=True`、`meta.reth`；BECN ACK bypass 还需 `meta.b_bit=True`
- **IngressFTE 是唯一协议解析点**: RX 管线中只有 IngressFTE 解析 BTH + 扩展头；MsgEngine/RcTransport/RxDPE(Parser/Lookup/Executor) 只读 `trans.meta`(PacketMetadata) 字段，不做协议解析
- **PacketMetadata 协议字段 (opcode/psn/reth/aeth/…)**: 由 IngressFTE 填充；默认值 0/None 向后兼容；RAW_PACKET 包强制 `opcode=0xFF`(synthetic RAW_ETH marker)，保证 `op_enum.is_raw()` 正确
- **EgressFTE L234 封装**: Assembly 输出 PendingFetchEntry（含 PacketDesc），EgressFTE 用 `desc.src_mac/dst_mac/src_ip/dst_ip` 构造 42B L234；字段从 QPC 经 TxCmd → PacketDesc 传播（RAW 路径留零值透传）
- **TX CQE RAW_PACKET**: 在 EgressFTE 的 `pkt_out.push` 之后生成（TM admission 后），不再在 Assembly 中生成

## 运行测试
```bash
python3 -m tests.st.ib_bw_write   <msg_size> <num_qps> <volume_per_qp> [-u]
python3 -m tests.st.ib_bw_read    <msg_size> <num_qps> <volume_per_qp> [-u]
python3 -m tests.st.ib_bw_send    <msg_size> <num_qps> <volume_per_qp> [-u]
python3 -m tests.st.ib_bw_rand    <num_qps> <volume_per_qp> [seed]
python3 -m tests.st.ib_sgl_rand   <num_qps> <volume_per_qp> [seed]
python3 -m tests.st.ib_xrc_rand   <num_qps> <volume_per_qp> [seed]
python3 -m tests.st.ib_veroce_rand <num_qps> <volume_per_qp> [seed]
```
- `volume_per_qp`: 每 QP 总字节数，支持 K/M 后缀。msgs/QP = volume / msg_size。
- `-u`: 单向模式（默认双向）。
- `ib_bw_rand`: RC 混合负载，每 QP 随机 WRITE/READ/SEND + 64B/4096B。
- `ib_xrc_rand`: XRC 混合负载（WRITE/READ/SEND），使用 XRC INI/TGT QP + SRQ。
- `ib_veroce_rand`: veRoCE 混合负载（WRITE/SEND），全 QP `veroce=True`，双向随机丢包，验证 OOO DDP + SACK + 选择性重传。
- `vr_bw_net`: Virtio-net 带宽基准，N QPs × 双向，测试 rate_mbps fair-share shaper + 环形缓冲区流控 + RX ring 补充。

回归用例 (18 项):
```bash
python3 -m tests.st.ib_bw_write    4096 64   256K
python3 -m tests.st.ib_bw_read     4096 64   256K
python3 -m tests.st.ib_bw_send     4096 64   256K
python3 -m tests.st.ib_bw_write    64   64   16K
python3 -m tests.st.ib_bw_read     64   64   16K
python3 -m tests.st.ib_bw_send     64   64   16K
python3 -m tests.st.ib_bw_rand     16   256K
python3 -m tests.st.ib_bw_rand     64   16K
python3 -m tests.st.ib_sgl_rand    64   16K
python3 -m tests.st.ib_xrc_rand    64   16K
python3 -m tests.st.ib_veroce_rand 16   256K
python3 -m tests.st.ib_pacer       4M
python3 tests/st/test_becn_inband.py
python3 tests/st/test_header_split.py
python3 tests/st/test_tcp_tso.py
python3 tests/st/test_tcp_rsc.py
python3 tests/st/test_virtio_net.py
python3 -m tests.st.vr_bw_net      1500 4    64K
```

## 关键性能参数
| 参数 | 文件 | 默认值 | 用途 |
|------|------|--------|------|
| `gs_budget_bytes` | `gs/config.py` | 1MB | GS 全局准入预算（safety net, READ 唯一全局限流） |
| `tm_quota_bytes` | `gs/config.py` | 32KB | GS pipeline 深度（port total + per-TC cap, returned at TM admission） |
| `port_alpha` | `gs/config.py` | 1.0 | Per-QP 公平共享比例 |
| `port_alpha_min_threshold` | `gs/config.py` | 1KB | Per-QP 最低阈值 = 1 MTU floor |
| `byte_budget_per_grant` | `gs/config.py` | 8KB | 每次 grant 的字节预算（字节公平调度: 4096B→2 WQE, 64B→128 WQE） |
| `per_qp_voq_xoff_ns` | `tm/config.py` | 1000 | TM per-QP XOFF 水位: rate × 1us |
| `per_qp_voq_xon_ns` | `tm/config.py` | 500 | TM per-QP XON 水位: rate × 0.5us |
| `drr_quantum` | `tm/config.py` | 4096 | TM DRR deficit quantum（bytes per round per QP） |
| `per_qp_shaper_max_burst` | `tm/config.py` | 9000 | TM Per-QP shaper burst（1 jumbo frame） |
| `sched_cache_capacity` | `gs/config.py` | 8192 | SchedulerCache 大小（必须 ≥ 活跃 QPs 避免 ICM 洪泛） |
| `tx_desc_pool_capacity` | `top/config.py` | 1024 | Desc pool 容量（SQM 自行分配, SQM→TE→RetransMgr 生命周期） |
| `burst_limit_pkts` | `te/config.py` | 64 | TxEngine 每次 slice 最大包数 |
| `max_qp_window_bytes` | `te/config.py` | 256KB | Per-QP protocol window (flight_bytes 上限) |
| `MAX_SEG_PER_QP` | `te/state.py` | 256 | Per-QP seg credit 上限（与 max_seg_entries 双重限制） |
| `max_seg_entries` | `te/config.py` | 4096 | Seg Credit Pool 全局容量（所有 QP 共享，limits in-flight TxCmds） |
| `prefetch_budget_bytes` | `sqm/config.py` | 1MB | 全局 WQE prefetch budget（限制 DMA 中未消费字节总量） |
| `pcie_max_tags` | `hif/config.py` | 512 | PCIe outstanding request tags |
| `alpha` | `tm/config.py` | 0.5 | AdmissionCtrl per-TC 队列深度限制因子 |

## GS Budget 调优
`gs_budget_bytes ≥ BDP (~500KB at 400G, 10us RTT)`。1MB 默认值限 READ 并发 ~256 (4KB msgs)。
作为 safety net 保留，主要限制 READ 并发。Pipeline 深度由 Alpha + TC Quota 控制。

## 仿真基础设施
- **StreamChannel**: 内部总线零延迟 pull（`_pull()` 无 `yield timeout`），允许 pipeline stages 同周期重叠
- **CdmiiSwitch**: 400G 物理链路模型，含 L1 开销 (26B: Preamble 8 + IFG 12 + FCS 4)。Ingress + Egress 均建模串行化延迟 (`wire_len × 8 / 400Gbps`)
- **SQM micro-TLB prefetch**: 已禁用以降低高 PPS 场景的仿真开销
- **PCIe TLP tracing**: 已移除（legacy debug 开销）

## 文档

- [架构](docs/architecture.md) — 详细模块内部、数据通路、pipeline 图
- [架构综述 v2](docs/architecture-review.md) — 重构后全量架构 review，标注 [NEW]/[CHANGED]
- [流控架构](docs/flow-control.md) — GS Alpha Pipeline Admission + TM Rate-Proportional XOFF/XON + GS Budget, Desc Pool, 调优
- [优化历史](docs/optimization-history.md) — Pipeline ROB, QPC 转发, Scheduler 重构, RNR 修复
- [ICM 架构](docs/icm-architecture.md) — ICM cache 层级, 客户端分析, 优化策略 #1-#7, miss rate 模型
- [QP 扩展分析](docs/qp-scaling.md) — 1024 QP 退化根因, Steps 1-11, 暗期分析
- [架构缺陷](docs/architecture-defects.md) — #A1-#A6 结构性问题及修复状态
- [关键通道](docs/critical-channels.md) — 关键 StreamChannel/队列溢出/下溢检测
- [XRC 传输](docs/xrc-transport.md) — XRC INI/TGT QP 设计, Driver API, 测试
- [veRoCE 协议](docs/veRoCe.md) — OOO DDP, SACK 生成, 选择性重传, Pathwise CC 集成, 配置与不变量
- [性能目标](docs/performance-targets.md) — PktRTT 分解 (Pipeline RTT + Inflight RTT), 各级 latency budget, 资源 sizing
- [设计决策](docs/decisions/) — ADR 记录（ROB, RNR, ICM cache 策略）
- [硬件真实性重构](docs/refactoring/HARDWARE_REALISM_INITIATIVE.md) — 硬件化重构计划（pipeline DMA, 协议解耦等）
- [架构审计](docs/refactoring/architecture-audit.md) — 架构审计报告
- [历史重构 Phase 1-5](docs/phase1_infrastructure.md) — 基础设施/流控/TX-RX pipeline/TE 集成各阶段实施记录
- [会话日志](docs/session_log.md) — 持续调查笔记、实验结果
- [故障排除](docs/troubleshooting.md) — 已知问题、调试技巧、经验教训

