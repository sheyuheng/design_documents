# Janus4K 10-Minute Technical Report / 10 分钟技术汇报稿

> Source: `Janus4k_AS.md` / `Janus4k_AS_EN.md` v0.5-reviewed  
> Purpose: English presentation with Chinese counterpart.  
> Status: Simulation results are placeholders to be filled later.

---

## Timing Plan / 时间分配

| Slide | Topic | Time |
|---|---|---:|
| 1 | Architecture thesis | 0:00-0:45 |
| 2 | TileReg-centered organization | 0:45-2:15 |
| 3 | TileReg bandwidth assumptions | 2:15-3:35 |
| 4 | Vector internal datapath | 3:35-5:15 |
| 5 | TileOp to uOp execution example | 5:15-6:45 |
| 6 | Output Buffer and forwarding | 6:45-7:45 |
| 7 | Latency and backpressure assumptions | 7:45-8:55 |
| 8 | Simulation plan and closing | 8:55-10:00 |

---

## Slide 1 — Architecture Thesis / 架构主线

### English Slide Bullets

- Janus4K is a tile-centric AI Core built around a shared `TileReg`.
- The execution hierarchy is `Block -> TileOp -> uOp`.
- The key question is not peak compute alone, but whether TileReg bandwidth, Vector scheduling, and forwarding keep the pipelines fed.

### English Speaker Notes

Today I will focus on the part of Janus4K that determines whether the core can actually sustain useful throughput: the TileReg-centered data organization and the Vector execution path. Janus4K is organized as a tile-centric AI Core. At the control level, work is expressed as Blocks, then decomposed into TileOps, and finally into uOps. At the data level, Vector, Cube, and TMA all exchange data through a shared TileReg. So the main design question is not just how many arithmetic units we instantiate. The main question is whether the local storage organization, bandwidth assumptions, and forwarding mechanisms can feed those units without turning the read path into the bottleneck.

### 中文对照

今天的汇报重点是 Janus4K 中真正决定有效吞吐的部分：以 TileReg 为中心的数据组织，以及 Vector 执行路径。Janus4K 的控制层次是 `Block -> TileOp -> uOp`，数据层面则由 Vector、Cube、TMA 通过共享 `TileReg` 交换数据。因此问题不只是“算力峰值有多少”，而是 TileReg 的组织形式、带宽假设、Vector 调度和 forwarding 机制能否持续供数，避免读路径成为瓶颈。

---

## Slide 2 — TileReg-Centered Organization / 以 TileReg 为中心的组织形式

### English Slide Bullets

- `TileReg`: shared in-core data buffer, total capacity **1 MB**.
- Vector Tile: fixed **4 KB**, split into **8 x 512 B** slices.
- TileReg exposes a **2 KB** full-bandwidth window.
- A 4 KB Vector Tile has two windows: `window0` and `window1`.

```text
4 KB Vector Tile

window0: slice0 slice1 slice2 slice3  = 2 KB
window1: slice4 slice5 slice6 slice7  = 2 KB

slice size = 512 B
```

### English Speaker Notes

The first architectural anchor is TileReg. TileReg is a one-megabyte shared in-core buffer. It is the exchange point between Vector, Cube, and TMA. For the Vector side, the architectural tile size is fixed at 4 KB. Each 4 KB tile is divided into eight 512-byte slices. Those eight slices are grouped into two 2 KB windows. The reason this matters is that 512 bytes is the local execution granularity, while 2 KB is the TileReg full-bandwidth access granularity. A Vector TileOp therefore naturally decomposes into two windows, and each window decomposes into four 512-byte execution slices. This gives the design a clean mapping between storage, read bandwidth, and Vector uOp execution.

### 中文对照

第一个架构锚点是 TileReg。TileReg 是 1 MB 的核内共享数据 buffer，是 Vector、Cube、TMA 的数据交汇点。对 Vector 而言，架构级 Tile 固定为 4 KB。每个 4 KB Tile 被拆成 8 个 512 B slice，再组织成两个 2 KB window。这里的关键是：512 B 是本地执行粒度，2 KB 是 TileReg 满带宽访问粒度。因此一个 Vector TileOp 可以自然拆成两个 window，每个 window 再拆成 4 个 512 B 的执行 uOp。

---

## Slide 3 — TileReg Bandwidth Assumptions / TileReg 带宽假设

### English Slide Bullets

| Item | Assumption |
|---|---|
| TileReg logical banks | 4 banks |
| Per-bank read bandwidth | 512 B/cycle |
| Per-bank write bandwidth | 512 B/cycle |
| Full read window | 2 KB/cycle |
| Full write window | 2 KB/cycle |
| Read clients | Vector compute, Cube, TMA |
| Read arbitration | Round-Robin |

### English Speaker Notes

The second point is the bandwidth model. TileReg is modeled as four architectural logical banks. Each bank provides 512 bytes per cycle for reads and 512 bytes per cycle for writes. Together, the four banks expose a 2 KB per-cycle full-bandwidth read window and a 2 KB per-cycle full-bandwidth write window. The document treats these as architectural visibility requirements, not as a final SRAM macro choice. Physical splitting, replication, or port implementation can still change later, but the visible contract must remain the same. Read traffic comes from three client classes: Vector compute, Cube, and TMA. These clients share the TileReg read path through Round-Robin arbitration. Each grant corresponds to one 2 KB TileReg window.

We also modeled a wider 4 KB TileReg access option. A 4 KB window doubles the visible memory-side bandwidth, but it can exceed the bandwidth that the current Vector compute resources can actually consume. In that case, the additional bandwidth is partly redundant: it increases the supply ceiling, but the execution pipes do not always turn that supply into higher throughput. In our model, the performance difference between 2 KB and 4 KB TileReg bandwidth is small for the current compute configuration. This is why the 2 KB assumption is not just a conservative simplification; it is close to the practical bandwidth point for this version.

### 中文对照

第二个重点是带宽模型。TileReg 被建模为 4 个架构级逻辑 bank，每个 bank 每拍提供 512 B 读带宽和 512 B 写带宽。4 个 bank 合起来，对外形成 2 KB/cycle 的满带宽读窗口和 2 KB/cycle 的满带宽写窗口。这里定义的是架构可见语义，不是最终 SRAM macro 选择；物理上可以采用分片、复制或多端口策略，但对外必须保持这个带宽契约。读请求来自三类 client：Vector compute、Cube、TMA，并通过 Round-Robin 仲裁共享读路径。每次 grant 对应一个 2 KB window。

我们也对更宽的 4 KB TileReg 访问窗口做了模型。4 KB window 会把可见访存侧带宽翻倍，但它可能高于当前 Vector 计算资源实际能够消耗的带宽。此时新增带宽会出现部分冗余：它提高了供数上限，但执行 pipe 不一定能把这部分供给转化成更高吞吐。在我们的模型中，当前计算配置下 2 KB 和 4 KB TileReg 带宽的性能差异不大。因此 2 KB 假设不只是保守简化，也接近这一版设计的实际有效带宽点。

---

## Slide 4 — Vector Internal Datapath / Vector 内部实现

### English Slide Bullets

```text
BCC Block
   |
   v
Vector Block / TileOp Window
   |
TileOp Decoder
   |
Operand Collector  -- Output Buffer lookup first
   |
Read Port Arbitration -> TileReg -> Global Src Buffer
   |
uOp Issue Queues
   |
FMLA/FCVT Pipe | IALU/PERM/MAC Pipe | SFU
   |
Output Buffer -> TileReg writeback
```

### English Speaker Notes

Inside the Vector Core, the data path starts when BCC dispatches a ready Vector Block. The Vector Block and TileOp Window hold the TileOp state and dependency state. The TileOp Decoder converts the Block information, including opcode, B.IOT, dtype, packing, element width, mask, and reduction attributes, into a uOp plan. The Operand Collector is important: it first checks the Output Buffer before issuing TileReg reads. If a source is already forwardable from the Output Buffer, that source does not consume TileReg read bandwidth. If it misses, the request goes to Read Port Arbitration. Returned 2 KB source windows enter the Global Src Buffer. Then uOps are issued into fixed queues and finally dispatched to dedicated execution resources.

### 中文对照

Vector Core 内部从 BCC 派发 ready Vector Block 开始。`Vector Block / TileOp Window` 保存 TileOp 状态和依赖状态。`TileOp Decoder` 根据 opcode、B.IOT、dtype、packing、element width、mask、reduction 属性生成 uOp plan。关键结构是 `Operand Collector`：它先查 Output Buffer，再决定是否发 TileReg read。若源数据已经可以从 Output Buffer forwarding，则不消耗 TileReg 读口；若 miss，则进入 Read Port Arbitration。TileReg 返回的 2 KB 源窗口先进入 Global Src Buffer，再进入 uOp Issue Queue，最后发射到固定执行单元。

---

## Slide 5 — TileOp to uOp Execution Example / TileOp 到 uOp 的执行示例

### English Slide Bullets

- One standard Vector TileOp consumes **4 KB** source Tiles.
- A three-source TileOp is executed as two 2 KB windows.
- Each 2 KB window issues **3 source reads** and then **4 x 512 B uOps**.

```text
windowN:
  read srcA[windowN] -> 2 KB
  read srcB[windowN] -> 2 KB
  read srcC[windowN] -> 2 KB
  issue 4 x 512 B uOps
```

### English Speaker Notes

For a concrete example, consider a standard three-source Vector TileOp. Each source Tile is 4 KB. The operation is split into two 2 KB windows. For each window, the Operand Collector needs source A, source B, and source C. If none of the sources hits in the Output Buffer, this window produces three 2 KB TileReg read requests. After all sources for the window are ready, the window is split into four 512-byte uOps. Across the full 4 KB TileOp, that means two windows, six 2 KB source reads in the worst case, and eight 512-byte compute uOps. This is the baseline case that simulation should later compare against forwarding-hit cases.

### 中文对照

以标准三源 Vector TileOp 为例，每个源 Tile 是 4 KB。执行时先拆成两个 2 KB window。每个 window 需要 srcA、srcB、srcC 三个源；如果三个源都没有命中 Output Buffer，则该 window 会产生三次 2 KB TileReg 读请求。等该 window 的所有源都 ready 后，再拆成 4 个 512 B uOp。完整 4 KB TileOp 就是两个 window，最坏情况下需要六次 2 KB 源读，并执行 8 个 512 B compute uOp。后续仿真可以把这个 baseline 和 forwarding hit 情况对比。

---

## Slide 6 — Execution Units and Output Buffer / 执行单元与 Output Buffer

### English Slide Bullets

| Execution path | Input granularity | Output granularity | Typical ops |
|---|---:|---:|---|
| `FMLA/FCVT Pipe` | 512 B/cycle | 512 B entry | FMLA, FCVT/CVT |
| `IALU/PERM/MAC Pipe` | 512 B/cycle | 512 B entry | IALU, PERM, MAC, reduction |
| `SFU` | 256 B/cycle | aggregated 512 B entry | EXP, DIV |

- Output Buffer supports forwarding, writeback arbitration, and reduction reuse.

### English Speaker Notes

The Vector execution resources are intentionally partitioned. There are two 512-byte main execution paths: one for FMLA and FCVT-style operations, and one for integer, permutation, MAC, compare, and reduction-style operations. In addition, there is an SFU path for operations like EXP and DIV. The SFU consumes 256 bytes per cycle, and two consecutive 256-byte results are aggregated into one 512-byte Output Buffer entry. The Output Buffer is the global structure between execution and TileReg writeback. It stores 512-byte result entries, supports producer-consumer forwarding, supports cross-pipe forwarding, and arbitrates writeback to TileReg. Reduction intermediates can remain resident in the Output Buffer instead of being written back after every step.

### 中文对照

Vector 的执行资源是固定分区的。两条 512 B 主执行路径分别覆盖 FMLA/FCVT 类操作，以及 IALU/PERM/MAC/compare/reduction 类操作。另外还有 SFU 路径，处理 EXP、DIV 等特殊函数。SFU 每拍消费 256 B，两个连续 256 B 结果聚合成一个 512 B Output Buffer entry。Output Buffer 位于执行结果和 TileReg 写回之间，是全局结构：它保存 512 B 结果 entry，支持 producer-consumer forwarding、跨 pipe forwarding，并参与 TileReg 写回仲裁。Reduction 中间结果可以驻留在 Output Buffer 中，不必每一步都写回 TileReg。

---

## Slide 7 — Latency and Backpressure Assumptions / Latency 与反压假设

### English Slide Bullets

| Assumption | Value / Rule |
|---|---|
| Read path normal propagation | ~2 cycles |
| Read path under congestion | ~2-3 cycles |
| Global Src Buffer | 6 entries, 2 KB each |
| Reduction dependency example | 4 cycles for TMAX-style chain |
| Flow control | valid/ready, no data/tag drop |

### English Speaker Notes

The latency model is intentionally simple at this stage. The critical read path is modeled as Read Port Arbitration to TileReg to Global Src Buffer. Normal propagation is assumed to be about two cycles. With routing congestion or detour, the current assumption is two to three cycles. To tolerate this variability, the Global Src Buffer is modeled as six entries, each entry holding one 2 KB source window. For reduction-style dependency chains such as TMAX or colmax, the current dependency distance assumption is four cycles. These numbers are not final physical signoff numbers. They are microarchitecture assumptions for scheduling, buffering, and simulation. The flow-control rule is strict: every stage uses valid/ready semantics, and no TileOp, uOp, data, or tag may be dropped under backpressure.

### 中文对照

当前 latency 模型保持简化。关键读路径被建模为 `Read Port Arbitration -> TileReg -> Global Src Buffer`。正常传播延迟假设约 2 cycles；在布线拥塞或绕线情况下，目前假设为 2 到 3 cycles。为了吸收这个波动，Global Src Buffer 按 6 个 entry 建模，每个 entry 保存一个 2 KB 源窗口。对 TMAX/colmax 这类 reduction 依赖链，目前使用 4 cycles 的依赖距离假设。这些数字不是最终物理签核结果，而是用于调度、缓冲和仿真的微架构假设。流控规则是严格的：各级使用 valid/ready，反压时不能丢 TileOp、uOp、data 或 tag。

---

## Slide 8 — Simulation Plan and Closing / 仿真计划与总结

### English Slide Bullets

Simulation results to be added:

| Case | Metric placeholders |
|---|---|
| Three-source TileOp, no forwarding | cycles/TileOp, effective B/cycle |
| Output Buffer forwarding hit | TileReg read reduction, speedup |
| Vector/Cube/TMA RR contention | fairness, wait cycles |
| Global Src Buffer pressure | occupancy, stall cycles |
| SFU EXP/DIV path | 256 B/cycle utilization |
| TMAX/reduction chain | dependency bubbles, Output Buffer reuse |
| 2 KB vs 4 KB TileReg window | performance delta, unused bandwidth |

### English Speaker Notes

The simulation section will fill in the quantitative evidence. The first baseline should be the three-source TileOp without forwarding, because it stresses the TileReg read path with six 2 KB source reads per full 4 KB TileOp. The second case should enable Output Buffer forwarding and measure how many TileReg reads are eliminated. Then we should test Round-Robin fairness under mixed Vector, Cube, and TMA traffic, Global Src Buffer occupancy under read latency variation, SFU utilization for EXP and DIV, and reduction chains such as TMAX. We should also include the 2 KB versus 4 KB TileReg-window model. The current modeling result is that moving to 4 KB bandwidth gives only a small performance gain, because the memory-side supply becomes higher than what the current Vector compute resources can consume. The expected conclusion is that Janus4K's performance depends on three linked contracts: TileReg must provide enough window bandwidth, Vector must decompose TileOps into schedulable 512-byte uOps, and Output Buffer must reduce unnecessary TileReg traffic through forwarding.

### 中文对照

仿真部分后续补定量证据。第一个 baseline 应该是无 forwarding 的三源 TileOp，因为它会对 TileReg 读路径施压：一个完整 4 KB TileOp 最坏需要六次 2 KB 源读。第二个 case 应该打开 Output Buffer forwarding，衡量减少了多少 TileReg read。然后测试 Vector/Cube/TMA 混合流量下的 RR 公平性、Global Src Buffer 在读延迟波动下的 occupancy 和 stall、SFU 在 EXP/DIV 上的利用率，以及 TMAX 这类 reduction 依赖链。还应加入 2 KB 与 4 KB TileReg window 的模型对比。目前模型结论是，提升到 4 KB 带宽只带来很小性能收益，因为访存侧供给已经高于当前 Vector 计算资源实际可消耗的带宽。预期结论是：Janus4K 的性能依赖三个相互绑定的契约：TileReg 提供足够的 window 带宽，Vector 将 TileOp 拆成可调度的 512 B uOp，Output Buffer 通过 forwarding 减少不必要的 TileReg 访问。

---

## One-Minute Backup Summary / 1 分钟备用总结

### English

Janus4K is best understood as a TileReg-centered AI Core. TileReg is a 1 MB shared buffer with four logical banks. The visible bandwidth contract is 512 B/cycle per bank and a 2 KB/cycle full read or write window. We also modeled a 4 KB window, but the current result shows limited performance gain because that bandwidth is higher than what the current Vector compute resources typically consume. Vector Tiles are 4 KB and split into two 2 KB windows and eight 512 B slices. The Vector Core decodes Blocks into TileOps and uOps, checks the Output Buffer before issuing TileReg reads, buffers returned 2 KB windows in the Global Src Buffer, and issues 512 B uOps into fixed execution queues. The current latency assumptions are about 2 cycles for normal read propagation, 2-3 cycles under congestion, and 4 cycles for a TMAX-style reduction dependency. Simulation will validate whether forwarding and buffering are enough to keep the Vector pipelines fed.

### 中文

Janus4K 可以理解为以 TileReg 为中心的 AI Core。TileReg 是 1 MB 共享 buffer，包含 4 个逻辑 bank；架构可见带宽契约是每 bank 512 B/cycle，对外形成 2 KB/cycle 的满带宽读/写窗口。我们也建模了 4 KB window，但当前结果显示性能收益有限，因为该带宽已经高于当前 Vector 计算资源通常能够消耗的带宽。Vector Tile 为 4 KB，拆成两个 2 KB window 和 8 个 512 B slice。Vector Core 将 Block 解码为 TileOp 和 uOp，先查 Output Buffer 再决定是否发 TileReg read，返回的 2 KB window 进入 Global Src Buffer，然后以 512 B uOp 发射到固定执行队列。当前 latency 假设是正常读路径约 2 cycles，拥塞时 2-3 cycles，TMAX 类 reduction 依赖约 4 cycles。后续仿真要验证 forwarding 和 buffering 是否足以持续供给 Vector pipeline。
