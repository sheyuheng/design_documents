# Janus4K AI Core Architecture Specification

> **Version:** v0.5-enhanced
> **Date:** 2026-04-27
> **Status:** Public presentation version (enhanced English edition)
> **Previous revision:** `Janus4k_AS_EN.md` v0.5-public (2026-04-26)
> **Working draft source:** `Janus4k_AS_draft.md` v0.4-draft

---

## Table of Contents

1. [Overview](#1-overview)
2. [Top-Level Architecture](#2-top-level-architecture)
3. [Data Model](#3-data-model)
4. [BCC — Block Control Core](#4-bcc--block-control-core)
5. [Vector Core](#5-vector-core)
6. [Output Buffer](#6-output-buffer)
7. [TileReg Arbitration and Buffers](#7-tilereg-arbitration-and-buffers)
8. [Cube](#8-cube)
9. [TMA](#9-tma)
10. [Vector Execution Flow](#10-vector-execution-flow)
11. [Interfaces](#11-interfaces)
12. [Parameter Table](#12-parameter-table)
13. [Scheduling, Wakeup, and Dispatch Model](#13-scheduling-wakeup-and-dispatch-model)
14. [Backpressure and Flow Control](#14-backpressure-and-flow-control)
15. [Physical Implementation Considerations](#15-physical-implementation-considerations)
16. [Performance Objectives and Bottlenecks](#16-performance-objectives-and-bottlenecks)
17. [Verification Coverage](#17-verification-coverage)
18. [Glossary](#18-glossary)

---

## 1. Overview

Janus4K is a tile-centric AI Core subsystem organized around a shared `TileReg`. The execution hierarchy is `Block → TileOp → uOp`. `BCC` performs Block-level dependency resolution, scalar execution, task dispatch, resolve tracking, and in-order retirement. `Vector Core`, `Cube`, and `TMA` execute vector computation, matrix computation, and Tile/Memory transfer respectively.

### 1.1 Core Design Principles

1. **`TileReg`** is the shared in-core data buffer with **1 MB** total capacity.
2. A **Vector Tile** is fixed at **4 KB**. Cube and TMA may use large Tiles whose sizes are integer multiples of 4 KB.
3. **512 B** is the common local granularity for Vector compute slices, TileReg bank slices, and `Output Buffer` entries.
4. **2 KB** is the full-bandwidth TileReg read/write window, formed by 4 logical banks each providing 512 B.
5. **`Output Buffer`** is the single global structure for result residency, forwarding, writeback arbitration, and reduction reuse.
6. Block completion resolve may return **out of order**. BCC retires Blocks **in order**.

### 1.2 Architecture Boundary

This specification defines Janus4K internal Block control, Tile data organization, Vector/Cube/TMA cooperation, TileReg read/write arbitration, and Output Buffer forwarding/writeback semantics. Software stack details, compiler IR, DDR controller internals, NoC protocol, and concrete SRAM macro implementation are **outside the scope** of this document. External modules must comply with the Block descriptor, Tile granularity, interface handshakes, and completion semantics defined here.

### 1.3 Visible Behavior

Janus4K exposes the following stable behavior to software and verification environments:

| Behavior | Architectural Definition |
|---|---|
| Block dispatch | BCC dispatches a Block to exactly one target PE after dependencies are satisfied. |
| Data exchange | Vector, Cube, and TMA exchange Tile data through TileReg. |
| Vector Tile | Each Vector Tile is fixed at **4 KB**. |
| Read/write window | The full-bandwidth TileReg read/write window is fixed at **2 KB**. |
| Result forwarding | Forwardable results in Output Buffer can be consumed before TileReg writeback. |
| Completion feedback | A PE returns `resolve + block_id`; BCC retires Blocks in order. |

### 1.4 Design Goals

1. Use **TileReg** as the unified data exchange hub for vector execution, Cube matrix computation, and TMA data movement.
2. Support Block descriptors decomposed into BCC instructions: `BStart`, `B.IOT (3 src + 1 dst)`, and TMA `TLOAD`.
3. Minimize pipeline bubbles through **Output Buffer** and **Data Forwarding** across multi-cycle execution and multi-level data dependencies.
4. Decouple "data readiness", "dependency satisfaction", and "execution pipeline availability" to reduce single-point blocking.
5. Split different operator classes across dual execution pipelines to improve actual compute-unit utilization.
6. Acknowledge read-path congestion as a critical timing factor, and use buffering and scheduling to hide latency.
7. Maintain hot working sets in small, near-execution local data structures to avoid frequent long-latency accesses.

---

## 2. Top-Level Architecture

### 2.1 Top-Level Modules

| Module | Responsibility |
|---|---|
| `BCC` | Block Control Core. Decodes Block descriptors, resolves Tile dependencies, executes scalar pipes, dispatches tasks, tracks resolve, and retires Blocks in order. |
| `Vector Core` | Executes Vector TileOps/uOps with **512 B/cycle** primary compute granularity. |
| `Cube` | Matrix compute unit. Receives TileReg data into L0A/L0B buffers and consumes it through a systolic array. |
| `TMA` | Tile Memory Access unit. Moves data bidirectionally between TileReg and DDR/Memory. |
| `TileReg` | Shared AI Core data buffer with **1 MB** total capacity and 4 logical banks. |
| `Memory` | External or upper-level memory system. |

### 2.2 Top-Level Diagram

```text
                         ┌──────────────────────────────┐
                         │             BCC              │
                         │  Block descriptor decode      │
                         │  Tile dependency scheduling   │
                         │  GPR + scalar pipes           │
                         │  resolve + in-order retire    │
                         └───────┬──────────┬───────────┘
                                 │          │
            vector block         │          │ block (Cube / TMA)
                                 v          v
     ┌──────────────────┐   ┌──────────┐   ┌──────────────┐
     │   Vector Core    │   │  Cube    │   │     TMA      │
     │ 512 B/cycle      │   │ Matrix   │   │ TileReg-DDR  │
     │ compute          │   │ compute  │   │ transfer     │
     └────────┬─────────┘   └────┬─────┘   └──────┬───────┘
              │                  │                │
              └──────────────┬───┴────────────────┘
                             │
                             v
                    ┌─────────────────┐
                    │     TileReg     │
                    │ 1 MB shared     │
                    │ 4 logical banks │
                    │ 2 KB window     │
                    └────────┬────────┘
                             │
                             v
                       ┌──────────┐
                       │ Memory   │
                       └──────────┘
```

### 2.3 Data-Centric Organization

`TileReg` is the data convergence point for Vector, Cube, and TMA. Vector Core, Cube, and TMA share TileReg ports through read/write arbitration. Data flows through TileReg, Global Src Buffer, Output Buffer, and the execution units.

### 2.4 Control Path and Data Path

Janus4K separates control paths from data paths:

| Path | Source | Destination | Payload |
|---|---|---|---|
| Block control | BCC | Vector/Cube/TMA | `block_id`, opcode, target PE, Tile descriptors, dtype/packing, memory ordering |
| Resolve control | Vector/Cube/TMA | BCC | `resolve_valid`, `resolve_block_id`, completion status |
| Tile read | Vector/Cube/TMA | TileReg | 2 KB read-window request |
| Tile return | TileReg | Global Src Buffer / Cube / TMA | 2 KB read-return data and tag |
| Result writeback | Output Buffer / Cube / TMA | TileReg | 512 B result entry or aggregated writeback data |
| Memory transfer | TMA | Memory | 256 B load/store beats |

The control path preserves Block identity. The data path preserves Tile ID, slice/window ID, tag, mask, and dtype semantics, so forwarding, writeback, resolve, and retirement remain correctly associated.

### 2.5 Single-Target PE Dispatch

Each Block carries a unique `target PE` in `BStart`. BCC does **not** duplicate one Block to multiple PEs. Work that requires multiple PEs is represented as multiple Blocks. Each Block independently establishes Tile input/output dependencies and independently returns `resolve + block_id`.

---

## 3. Data Model

### 3.1 Tile

| Item | Definition |
|---|---|
| Vector Tile | Fixed **4 KB** |
| Cube/TMA Tile | Integer multiple of 4 KB |
| Tile slice | **512 B** |
| TileReg access window | **2 KB** |
| Tile address organization | A single 4 KB Tile is contiguous in TileReg |

A **4 KB** Vector Tile contains eight **512 B** slices. TileReg maps 512B slice numbers onto 4 logical banks, allowing any PE to read a continuous 2 KB window from a Tile without bank conflict.

#### Tile Slice and Window Organization

| Slice | Byte Range | 2KB Window | Usage |
|---|---|---|---|
| slice0 | `[0, 512)` | window0 | Vector compute slice 0 |
| slice1 | `[512, 1024)` | window0 | Vector compute slice 1 |
| slice2 | `[1024, 1536)` | window0 | Vector compute slice 2 |
| slice3 | `[1536, 2048)` | window0 | Vector compute slice 3 |
| slice4 | `[2048, 2560)` | window1 | Vector compute slice 4 |
| slice5 | `[2560, 3072)` | window1 | Vector compute slice 5 |
| slice6 | `[3072, 3584)` | window1 | Vector compute slice 6 |
| slice7 | `[3584, 4096)` | window1 | Vector compute slice 7 |

A full Vector TileOp covers both window0 and window1. Partial Tile operations still use **512 B** as the minimum execution slice and **2 KB** as the full-bandwidth TileReg access granularity.

### 3.2 TileReg

TileReg is the shared 1 MB data buffer:

| Attribute | Value |
|---|---|
| Total capacity | **1 MB** |
| Logical banks | **4** |
| Read bandwidth per bank | 512 B/cycle |
| Write bandwidth per bank | 512 B/cycle |
| Bank ports | dual-port |
| Full read window | 2 KB/cycle |
| Full write window | 2 KB/cycle |

Read and write conflicts are resolved through arbitration. TileReg banks are **architectural logical banks**. RTL and physical implementation may choose SRAM partitioning, replication, or porting strategies, while preserving the visible bandwidth and conflict-free window semantics.

#### TileReg Request Fields

| Field | Semantics |
|---|---|
| `tile_id` | Tile identifier in TileReg |
| `window_id` | 2 KB window ID (0 or 1 inside a Vector Tile) |
| `slice_base` | Base 512 B slice |
| `bytes` | Request size (2048 for a full-bandwidth window) |
| `client` | Vector compute, Cube, or TMA |
| `tag` | Identity for return data, wakeup, writeback, and debug matching |

TileReg guarantees that a **2 KB** read for the same `tile_id + window_id` returns without bank conflict after a grant. Writeback may be performed as 512 B entries or as an implementation-level wider write window. The architectural result preserves slice order and byte mask correctness.

### 3.3 Block Descriptor

Block descriptors are decoded by BCC. Each Block descriptor starts with `BStart` and uses one or more BCC instructions to describe inputs, outputs, and transfer attributes.

#### BCC Instructions

| Instruction | Fields | Semantics |
|---|---|---|
| `BStart` | opcode, target PE, dtype/packing, attrs | Marks Block start, operation class, target PE, and data format. |
| `B.IOT` | 3 src Tiles + 1 dst Tile | Binds input and output Tiles. |
| `TLOAD` | memory addr, dst Tile, attrs | TMA load-type Tile transfer instruction. |

#### Constraints

1. A Block is dispatched to **exactly one** target PE.
2. A Block has at most **4** destination Tiles.
3. A Block has at most **2** source Tile descriptor groups.
4. A single `B.IOT` describes `3 src + 1 dst`; multiple `B.IOT` instructions combine to express multi-input or multi-output Blocks.
5. `BStart` carries dtype/packing. Vector TileOps and uOps use this information to interpret element width and data layout.

#### BStart Logical Fields

| Field | Semantics |
|---|---|
| `block_id` | Unique Block identity |
| `opcode` | Block opcode |
| `target_pe` | Vector, Cube, or TMA |
| `dtype_pack` | Data type and packing |
| `elem_width` | Element width |
| `mask_en` | Whether a mask Tile is used |
| `reduction_en` | Whether the operation is a reduction |
| `memory_order_attr` | TMA memory-ordering attribute |

Expanded `B.IOT` entries form the Tile dependency table. Each entry contains source Tiles, destination Tiles, read/write attributes, mask attributes, and target PE. BCC uses these entries to establish inter-Block dependencies and releases them when the Block retires.

### 3.4 Dependency Model

Block dependencies are established through Tile inputs and outputs. Rules:

- **Normal compute Blocks** do not use flag dependencies and do not create normal memory side-effect dependencies.
- **TMA `TLOAD` and store operations** follow memory ordering.
- **GPR** is used for scalar value exchange across Blocks. Scalar dependencies are maintained by the BCC scalar pipes and GPR.
- **No flag dependencies** are introduced in the current model.
- Block retirement releases Tile dependencies, GPR dependencies, and state visible to later Blocks.

---

## 4. BCC — Block Control Core

### 4.1 Responsibilities

`BCC` is the Block-level control core of Janus4K. It performs:

1. `BStart` / `B.IOT` / `TLOAD` decoding.
2. Tile input/output dependency tracking.
3. BCC scalar pipe execution.
4. Dispatch of ready Blocks to Vector Core, Cube, or TMA.
5. Reception of `resolve + block_id` from PEs.
6. Out-of-order resolve tracking and in-order Block retirement.

### 4.2 Scalar Pipe

BCC contains GPR and three scalar pipe classes:

| Structure | Function |
|---|---|
| `GPR` | General-purpose scalar register file for scalar value exchange across Blocks. |
| `AB pipe` | ALU + branch pipe with 2-pick support. |
| `AM pipe` | ALU + multicycle pipe with scalar multiply support. |
| `LSU` | Scalar load/store unit. |

Scalar values written by one Block into GPR can be consumed by a later Block, providing a lightweight inter-Block communication mechanism.

### 4.3 Resolve and Retirement

After a PE completes a Block, it returns `resolve + block_id` to BCC. The resolve/retirement protocol:

1. **Resolve may return out of order.** A later-dispatched Block may complete before an earlier one.
2. BCC records completion state in the retirement structure.
3. BCC retires Blocks **strictly in program order**.
4. Retirement releases:
   - Tile dependencies (Tile IDs become available for subsequent Blocks)
   - GPR dependencies
   - State visible to later Blocks
5. TMA Blocks retire only after their memory-ordering constraints are satisfied.

---

## 5. Vector Core

### 5.1 Vector Core Structure

Vector Core is the primary programmable vector execution unit in Janus4K. It receives Vector Blocks from BCC, decomposes TileOps into uOps, and executes them on fixed execution resources.

| Structure | Responsibility |
|---|---|
| `Vector Block / TileOp Window` | Receives Vector Blocks, stores TileOp state, and maintains Tile-level dependency and wakeup state. |
| `TileOp Decoder` | Generates a uOp plan from opcode, dtype/packing, element width, mask, and reduction attributes. |
| `Operand Collector` | Looks up Output Buffer first, then generates TileReg read requests for missed sources. |
| `Read Port Arbitration` | Performs RR arbitration for TileReg read requests from Vector compute, Cube, and TMA. |
| `Global Src Buffer` | Stores **2 KB** source data windows returned by TileReg. Depth: ~6 entries. |
| `uOp Issue Queue` | Maintains ready uOps by execution class and issues them to fixed execution units. |
| `FMLA/FCVT Pipe` | Executes FMLA and FCVT/CVT operations. |
| `IALU/PERM/MAC Pipe` | Executes IALU, PERM, MAC, compare, and reduction operations. |
| `SFU` | Executes EXP/DIV and other special functions at **256 B/cycle**. |
| `Output Buffer` | Stores results and supports forwarding, 2KB locking, writeback arbitration, and reduction reuse. |

### 5.2 Vector Pipeline Overview

Vector Core pipeline is divided into front-end, source-read, issue, execute, result, and completion sections:

| Stage | Name | Main Structure | Input | Output |
|---|---|---|---|---|
| V0 | Block accept | Vector Block | BCC Vector Block | TileOp window entry |
| V1 | TileOp decode | TileOp Decoder | opcode, B.IOT, dtype/packing, mask | uOp plan |
| V2 | Source collect | Operand Collector | src Tile desc, Output Buffer tag | forward hit or read request |
| V3 | Read arbitration | Read Port Arbitration | Vector/Cube/TMA read request | 2KB read grant |
| V4 | TileReg read return | TileReg + Global Src Buffer | 2KB read window | source window ready |
| V5 | uOp issue | uOp Issue Queue | ready uOp + source window | pipe/SFU issue |
| V6 | Execute | FMLA/FCVT, IALU/PERM/MAC, SFU | 512B/256B operand slice | result entry |
| V7 | Result buffer | Output Buffer | result entry | forward hit / writeback request |
| V8 | Completion | Vector Block | TileOp done bitmap | `resolve + block_id` |

**Key pipeline properties:**

- Multiple TileOps may reside in the pipeline at the same time.
- Different TileOps may occupy different stages.
- The two 2 KB windows of one TileOp may be pipelined.
- V3 (read arbitration), V5 (uOp issue), and V7 (Output Buffer insertion) independently generate backpressure.

### 5.3 TileOp to uOp Decomposition

Each Vector TileOp source input is a **4 KB srcTile**. A standard elementwise three-source operation is split into two 2 KB windows, and each window is split into four 512 B uOps:

```text
TileOp(4KB)
  window0: slice0, slice1, slice2, slice3
  window1: slice4, slice5, slice6, slice7
```

#### Per-Window Read and Compute Sequence

Each three-source TileOp window performs three source reads in a fixed order, followed by the compute uOps:

```text
windowN:
  read srcA[windowN]  -> 2KB
  read srcB[windowN]  -> 2KB
  read srcC[windowN]  -> 2KB
  issue 4 x 512B uOp
```

#### uOp Descriptor Fields

| Field | Semantics |
|---|---|
| `block_id` | Owning Block |
| `tileop_id` | Owning TileOp |
| `uop_id` | uOp sequence inside the TileOp |
| `opclass` | FMLA/FCVT, IALU/PERM/MAC, or SFU |
| `src_desc[0..2]` | Source Tile/window/slice descriptors |
| `mask_tile_desc` | Optional mask Tile |
| `dst_desc` | Destination Tile/window/slice descriptor |
| `dtype_pack` | Data type and packing carried by BStart |
| `elem_width` | Element width |
| `window_id` | 2KB window ID |
| `slice_id` | 512B slice ID |
| `forwardable` | Whether the result may be forwarded after entering Output Buffer |
| `reduction_ctx` | Reduction-chain context |

**Details by opclass:**

- **Elementwise operations** (FMLA/FCVT/IALU/PERM/MAC): Execute at **512 B** slice granularity.
- **SFU operations** (EXP/DIV): Read data as 2 KB windows, but execution consumes data at **256 B/cycle**. Two consecutive 256B results combine into one 512 B Output Buffer entry.
- **Reduction operations**: Generate uOps according to reduction direction, element width, and reduction tree. Intermediate results may remain resident in Output Buffer.

### 5.4 Source-Read Pipeline

The Operand Collector implements a **lookup-before-read** policy:

```text
src desc
  ├─ Output Buffer hit  -> lock 2KB window -> source ready (forwarding)
  └─ Output Buffer miss -> enqueue TileReg read request -> wait grant
```

#### Source-Read Rules

| Rule | Definition |
|---|---|
| Lookup priority | Output Buffer lookup precedes TileReg read request generation. |
| Hit behavior | A hit source does not consume a TileReg read port; the matching 2KB Output Buffer window is **locked**. |
| Miss behavior | A missed source enters Read Port Arbitration. |
| Grant granularity | Each grant reads one **2 KB** window. |
| Three-source order | A Vector three-source window issues reads in **srcA → srcB → srcC** order. |
| Return location | TileReg return data enters Global Src Buffer. |
| Wakeup | A uOp group wakes up after **all source windows are ready**. |

#### Global Src Buffer

- Entry granularity: **2 KB**
- Depth: **~6 entries** (small buffer to hide TileReg read latency)
- A three-source window may occupy up to three entries (srcA, srcB, srcC).
- A source satisfied through Output Buffer forwarding does **not** allocate a Global Src Buffer entry.

### 5.5 uOp Issue Queue

The uOp Issue Queue is partitioned into three fixed queues with dedicated execution units:

| Queue | Execution Unit | Operations |
|---|---|---|
| `FMLA/FCVT Queue` | `FMLA/FCVT Pipe` | FMLA, FCVT/CVT |
| `IALU/PERM/MAC Queue` | `IALU/PERM/MAC Pipe` | IALU, PERM, MAC, compare, reduction, TMAX/colmax |
| `SFU Queue` | `SFU` | EXP, DIV, and other special functions |

#### uOp Ready Conditions

A uOp becomes ready when:
1. All source windows are ready.
2. The mask Tile is ready (or the uOp does not use a mask).
3. The target execution unit can accept the uOp.
4. Output Buffer can allocate a result entry or assert backpressure.
5. The previous `reduction_ctx` is ready for reduction uOps.

**Execution unit isolation:** The three execution resource classes do **not** share execution units. `opclass` to execution-unit mapping is fixed. The scheduler does not issue SFU uOps to normal Vector pipes and does not issue FMLA/FCVT uOps to the IALU/PERM/MAC Pipe.

### 5.6 Execute Pipeline

Vector execution includes two 512B main execution paths and one 256B SFU path:

| Execution Path | Input Granularity | Output Granularity | Typical Operations | Result Destination |
|---|---|---|---|---|
| `FMLA/FCVT Pipe` | 512 B/cycle | 512 B entry | FMLA, FCVT/CVT | Output Buffer |
| `IALU/PERM/MAC Pipe` | 512 B/cycle | 512 B entry | IALU, PERM, MAC, TMAX/colmax | Output Buffer |
| `SFU` | 256 B/cycle | Aggregated into 512 B entry | EXP, DIV | Output Buffer |

Normal 512B pipes write **512 B** entries into Output Buffer. SFU consumes **256 B** per cycle, and two consecutive 256B results are combined into one **512 B** Output Buffer entry.

Execution outputs carry `block_id/tileop_id/uop_id/dst_desc/slice_id/tag` for Output Buffer lookup, writeback, and completion accounting.

### 5.7 Typical Three-Source TileOp Timing

A standard three-source 4 KB TileOp follows this read and compute order:

| Step | Action | Data Size | Notes |
|---|---|---|---|
| 1 | window0 srcA read | 2KB | RR arbitration |
| 2 | window0 srcB read | 2KB | RR arbitration |
| 3 | window0 srcC read | 2KB | RR arbitration |
| 4 | window0 compute | 4 × 512B | Issue four 512B uOps |
| 5 | window1 srcA read | 2KB | RR arbitration |
| 6 | window1 srcB read | 2KB | RR arbitration |
| 7 | window1 srcC read | 2KB | RR arbitration |
| 8 | window1 compute | 4 × 512B | Issue four 512B uOps |
| 9 | result residency/writeback | 8 × 512B | Results enter Output Buffer, then write back |

**Optimization:** When a source hits Output Buffer, the corresponding read step is replaced by forwarding and no TileReg read is issued.

### 5.8 Wakeup and Dependency Handling

Vector Core uses two wakeup classes:

| Wakeup Type | Source | Purpose |
|---|---|---|
| External wakeup | TileReg read grant/return | Wakes uOp groups waiting for TileReg data. |
| Internal wakeup | Output Buffer result ready | Wakes producer-consumer chains or later reduction uOps. |

An Output Buffer hit consumer locks the corresponding 2KB window and enters source-ready state. Reduction successors can consume intermediate results directly from Output Buffer and do not wait for intermediate TileReg writeback.

### 5.9 Vector Backpressure

Every Vector pipeline stage follows **valid/ready** semantics. If a stage cannot accept a new payload, the upstream stage keeps payload stable and preserves tag and data.

| Blocking Point | Trigger | Backpressure Direction |
|---|---|---|
| TileOp Window full | No available TileOp entry | BCC stops dispatching new Vector Blocks. |
| Operand Collector busy | Source lookup or read request queue full | TileOp generation of new uOps pauses. |
| Read Arbitration pending | RR does not grant | uOp group waits for sources. |
| Global Src Buffer full | TileReg return has no free entry | TileReg return path applies backpressure; new read grants suppressed. |
| uOp Queue full | Target opclass queue has no free entry | Operand Collector pauses ready uOp dispatch. |
| Execute busy | Target execution unit cannot accept | uOp Queue holds ready uOps. |
| Output Buffer full/locked | No result entry available or target window locked | Execute output applies backpressure; issue suppressed. |
| TileReg write busy | Write arbitration does not grant | Output Buffer keeps unwritten entries. |

---

## 6. Output Buffer

`Output Buffer` is a **single global structure** between execution results and TileReg writeback. It provides:

1. **Result storage** at 512 B entry granularity.
2. **Producer-consumer forwarding** — dependency source hits avoid TileReg read.
3. **Cross-pipe forwarding** — Pipe0/Pipe1/SFU results can be consumed by any pipe.
4. **TileReg write-port arbitration** — resolves write conflicts between Output Buffer, Cube, and TMA.
5. **Reduction intermediate reuse** — results stay resident across multiple reduction steps.
6. **Result residency** — entries can persist before later writeback.

### 6.1 Output Buffer Entry Fields

| Field | Semantics |
|---|---|
| `valid` | Whether the entry is valid |
| `tag` | Forwarding/writeback matching tag |
| `tile_id` | Destination Tile |
| `window_id` | 2KB window |
| `slice_id` | 512B slice |
| `data` | 512B result |
| `byte_mask` | Valid byte mask |
| `forwardable` | Whether forwarding is allowed |
| `writeback_required` | Whether TileReg writeback is required |
| `locked` | Whether the owning 2KB window is locked by a consumer |
| `reduction_ctx` | Reduction-chain context |

### 6.2 Lock and Writeback Protocol

1. When a TileOp finds its source data in Output Buffer, the corresponding **2 KB data window is locked**.
2. While locked, the window **cannot be overwritten or released**.
3. After the consumer finishes, the window is **unlocked**.
4. Unlocked entries attempt TileReg writeback every cycle.
5. Writeback requests are arbitrated at the TileReg write port.
6. Reduction intermediate results may stay in Output Buffer across multiple reduction steps without intermediate TileReg writeback.
7. The final reduction result is written back after writeback conditions are satisfied.

### 6.3 Why Output Buffer Must Precede Writeback

Without Output Buffer:
- Chain dependencies would frequently fall back to TileReg reads.
- Multi-cycle results could not be consumed quickly.
- Write-port conflicts would block the execution tail.
- Column reductions (e.g., `colmax`) would exhibit significant bubbles.

---

## 7. TileReg Arbitration and Buffers

### 7.1 Read Port Arbitration

TileReg read arbitration accepts requests from three clients:
1. `Vector compute`
2. `Cube`
3. `TMA`

#### Arbitration Policy

- **Round-Robin (RR)** arbitration among the three client classes.
- Each grant corresponds to one **2 KB** TileReg read window.
- Requests that do not receive a grant remain pending and continue participating in later RR rounds.
- RR state updates based on actual grant issuance. Empty client queues must not break fairness for active requesters.

#### Timing Characteristics

| Condition | Latency |
|---|---|
| Normal propagation | ~2 cycles |
| Routing congestion (detour) | ~2–3 cycles |

### 7.2 Global Src Buffer

Global Src Buffer is located on the TileReg return path:

| Attribute | Value |
|---|---|
| Entry granularity | **2 KB** |
| Depth | **~6 entries** |
| Purpose | Absorb variable latency from Read Port Arbitration → TileReg; decouple TileReg supply rate from execution consumption rate. |

Before entering execution units, a 2 KB data window is split into four 512 B slices.

### 7.3 Write-Port Arbitration

TileReg write conflicts are resolved through write-port arbitration. Write requesters include:
- `Output Buffer`
- `Cube`
- `TMA`

**Guarantees:**
- Data, tag, tile_id, and slice_id consistency preserved.
- Forwarding-visible results are **not lost** when writeback is stalled.
- Unlocked Output Buffer entries attempt writeback every cycle.
- Write arbitration does not reorder results in a way that breaks dependency visibility.

---

## 8. Cube

Cube is the matrix compute unit of Janus4K.

| Attribute | Definition |
|---|---|
| Data interface | Inputs and outputs expressed as Tiles (may use large Tiles composed of multiple 4 KB Tiles) |
| Internal buffering | L0A/L0B buffers — reads TileReg data into these local buffers before systolic consumption |
| Consumption granularity | Internal systolic array — not constrained by the Vector 512 B slice |
| Completion | Returns `resolve + block_id` to BCC after completion |

Cube follows these architectural rules:
1. Cube reads data from TileReg via the shared read arbitration (same RR as Vector compute and TMA).
2. Data enters L0A/L0B buffers which decouple TileReg arbitration timing from array consumption timing.
3. Cube may use large Tiles sized as integer multiples of 4 KB.
4. Cube results are written back to TileReg (participating in write-port arbitration).
5. The exact array size, L0A/L0B depth, accumulator width, and output layout are beyond this spec's scope.

---

## 9. TMA

TMA performs bidirectional data movement between TileReg and DDR/Memory.

| Attribute | Definition |
|---|---|
| Direction | Bidirectional load/store |
| TileReg request limit | **2 KB** per request |
| Memory-side beat | **256 B** |
| Read/write ports | Separate |
| Queueing | Load and store share scheduling queues |
| Ordering | TLOAD/store operations maintain memory ordering |

### 9.1 TMA Details

- **TMA load:** Moves memory data into TileReg.
- **TMA store:** Writes TileReg data back to memory.
- **Completion:** Returns `resolve + block_id` to BCC after completion.
- **Memory ordering:** TMA operations must maintain the memory ordering semantics specified in `BStart.memory_order_attr`. A TMA Block retires only after its ordering constraints are satisfied.
- **TileReg requests:** Each individual TMA-to-TileReg transaction is limited to 2 KB, matching the TileReg full-bandwidth window.

---

## 10. Vector Execution Flow

The complete Vector execution flow is:

1. **BCC decodes** the Block descriptor and confirms the target PE is Vector Core.
2. **BCC checks dependency readiness** through Tile inputs and outputs.
3. **BCC dispatches** a ready Vector Block to the Vector Block / TileOp Window.
4. **TileOp Decoder** generates a uOp plan from `BStart/B.IOT`, dtype/packing, element width, mask, and opclass.
5. **Operand Collector** performs Output Buffer lookup for each source.
6. **Output Buffer hit** → lock the corresponding 2KB window and satisfy via forwarding.
7. **Output Buffer miss** → generate TileReg read requests to Read Port Arbitration.
8. **Read Port Arbitration** uses RR to grant 2 KB read windows.
9. **TileReg return data** enters Global Src Buffer.
10. After **all sources for a window are ready**, four 512B uOps enter the fixed uOp queue.
11. **uOp Issue Queue** issues uOps to FMLA/FCVT, IALU/PERM/MAC, or SFU according to opclass.
12. **Execution units** generate 512 B Output Buffer entries (SFU aggregates two 256B beats into one 512B entry).
13. **Output Buffer** wakes later consumers by tag, or writes back unlocked entries to TileReg.
14. **TileOp done state** updated after all windows and slices complete.
15. **Vector Core returns** `resolve + block_id` to BCC after all TileOps in the Block complete.
16. **BCC retires** resolved Blocks in order.

---

## 11. Interfaces

All interfaces use **valid/ready** handshake unless otherwise specified:

```text
transfer = valid & ready
```

- `valid`: driven by sender, indicates payload is valid.
- `ready`: driven by receiver, indicates ability to accept this cycle.
- When `valid=1` and `ready=0`, the sender must keep payload stable.
- Every transaction must carry at least one identity field (`block_id`, `tileop_id`, `uop_id`, or `tag`).

### 11.1 BCC to PE

| Signal | Direction | Meaning |
|---|---|---|
| `block_valid` | BCC → PE | Block is valid. |
| `block_ready` | PE → BCC | Target PE can accept the Block. |
| `block_id` | BCC → PE | Block identifier. |
| `block_target` | BCC → PE | Vector/Cube/TMA target. |
| `block_opcode` | BCC → PE | Block operation class. |
| `block_bstart` | BCC → PE | Decoded BStart payload. |
| `block_iot_vec` | BCC → PE | Tile input/output bindings expanded from B.IOT. |
| `block_attr` | BCC → PE | dtype/packing, element width, reduction, memory ordering, and related attributes. |

### 11.2 PE to BCC

| Signal | Direction | Meaning |
|---|---|---|
| `resolve_valid` | PE → BCC | Block completion resolve is valid. |
| `resolve_ready` | BCC → PE | BCC can accept resolve. |
| `resolve_block_id` | PE → BCC | Completed Block ID. |
| `resolve_status` | PE → BCC | Completion status. |

### 11.3 Operand Collector to Read Port Arbitration

| Signal | Direction | Meaning |
|---|---|---|
| `rd_req_valid` | requester → arb | TileReg read request is valid. |
| `rd_req_ready` | arb → requester | Arbiter can accept the request. |
| `rd_req_client` | requester → arb | Vector compute/Cube/TMA. |
| `rd_req_tile_id` | requester → arb | Tile ID. |
| `rd_req_slice_base` | requester → arb | 512B slice or 2KB window base. |
| `rd_req_bytes` | requester → arb | Request size, normally **2 KB**. |
| `rd_req_tag` | requester → arb | Tag for return-data matching. |

### 11.4 Execute/SFU to Output Buffer

| Signal | Direction | Meaning |
|---|---|---|
| `ob_valid` | exec → OB | Execution result is valid. |
| `ob_ready` | OB → exec | Output Buffer can accept the result. |
| `ob_tag` | exec → OB | Forwarding/writeback matching tag. |
| `ob_data` | exec → OB | **512 B** result entry. |
| `ob_forwardable` | exec → OB | Result can be forwarded. |
| `ob_writeback_req` | exec → OB | Result requires TileReg writeback. |
| `ob_reduction_ctx` | exec → OB | Reduction context. |

### 11.5 TMA

| Signal | Direction | Meaning |
|---|---|---|
| `tma_cmd_valid` | BCC → TMA | TMA command is valid. |
| `tma_cmd_ready` | TMA → BCC | TMA can accept the command. |
| `tma_cmd_is_store` | BCC → TMA | Load/store direction. |
| `tma_tile_id` | BCC → TMA | TileReg Tile ID. |
| `tma_tile_bytes` | BCC → TMA | TileReg request size, maximum **2 KB**. |
| `tma_mem_addr` | BCC → TMA | Memory address. |
| `tma_mem_beat_bytes` | BCC → TMA | Memory beat size, fixed at **256 B**. |
| `tma_order_tag` | BCC → TMA | Memory ordering tag. |

### 11.6 Operand Collector / Src Buffer to uOp Issue Queue

| Signal | Direction | Meaning |
|---|---|---|
| `uop_valid` | → uOpQ | uOp ready for minimum queue entry. |
| `uop_ready` | uOpQ → | uOp Issue Queue can accept. |
| `uop_payload` | → uOpQ | uOp descriptor fields (see §5.3). |
| `uop_src_ready_mask` | → uOpQ | Which of 3 sources are ready (3-bit mask). |
| `uop_forward_hit_mask` | → uOpQ | Which sources from Output Buffer forwarding. |

### 11.7 Output Buffer to TileReg (Writeback)

| Signal | Direction | Meaning |
|---|---|---|
| `tr_wr_valid` | → TileReg | Writeback request valid. |
| `tr_wr_ready` | TileReg → | TileReg write port can accept. |
| `tr_wr_tile_id` | → TileReg | Target Tile. |
| `tr_wr_slice_id` | → TileReg | Target 512B slice. |
| `tr_wr_data` | → TileReg | 512B writeback data. |
| `tr_wr_mask` | → TileReg | Byte/section write mask. |
| `tr_wr_tag` | → TileReg | Debug/completion tracking tag. |

---

## 12. Parameter Table

| Parameter | Value | Description |
|---|---|---|
| `TILEREG_BYTES` | 1 MB | Total TileReg capacity. |
| `TILE_BYTES` | 4096 | Vector Tile size. |
| `TILE_SLICE_BYTES` | 512 | Vector slice / Output Buffer entry size. |
| `TILEREG_BANKS` | 4 | Number of logical TileReg banks. |
| `TILEREG_BANK_BYTES_PER_CYCLE` | 512 | Read/write bandwidth per bank per cycle. |
| `TILEREG_RW_BYTES_PER_CYCLE` | 2048 | Full-bandwidth TileReg read/write window. |
| `GLOBAL_SRCBUF_ENTRY_BYTES` | 2048 | Global Src Buffer entry size. |
| `GLOBAL_SRCBUF_ENTRIES` | 6 | Global Src Buffer depth. |
| `OUTPUT_BUFFER_ENTRY_BYTES` | 512 | Output Buffer entry size. |
| `MAX_DST_TILES_PER_BLOCK` | 4 | Maximum destination Tiles per Block. |
| `MAX_SRC_TILE_DESC_GROUPS_PER_BLOCK` | 2 | Maximum source Tile descriptor groups per Block. |
| `B_IOT_SRC_TILES` | 3 | Number of source Tiles in one B.IOT. |
| `B_IOT_DST_TILES` | 1 | Number of destination Tiles in one B.IOT. |
| `NUM_EXEC_CLASSES` | 3 | FMLA/FCVT, IALU/PERM/MAC, and SFU. |
| `SFU_BYTES_PER_CYCLE` | 256 | SFU EXP/DIV bandwidth. |
| `TMA_REQ_BYTES` | 2048 | Maximum TMA TileReg request size. |
| `TMA_MEM_BEAT_BYTES` | 256 | TMA memory-side beat size. |
| `READ_ARB_POLICY` | RR | TileReg read arbitration policy. |
| `READ_PROP_LATENCY` | 2 cycles | Normal TileReg read propagation delay. |
| `READ_CONGESTED_LATENCY` | 2–3 cycles | Read path latency under routing congestion. |
| `REDUCE_DEP_LATENCY` | 4 cycles | Example reduction dependency distance (TMAX). |

---

## 13. Scheduling, Wakeup, and Dispatch Model

### 13.1 Two-Level Scheduling

Janus4K uses two levels of scheduling:

| Level | Granularity | Managed By | Purpose |
|---|---|---|---|
| TileOp-level | Coarse (Tile) | TileOp Issue Queue | Tracks Tile dependencies and wakeup state |
| uOp-level | Fine (slice) | uOp Issue Queue | Dispatches ready uOps to execution units |

### 13.2 Wakeup Types

| Wakeup | Source | Effect |
|---|---|---|
| External wakeup | TileReg read grant → data return | Wakes uOp groups waiting for TileReg data |
| Internal wakeup | Output Buffer result ready | Wakes producer-consumer and reduction successors |

### 13.3 Dispatch State Machine

```
WAIT_SRC ──(OutputBuffer hit)──────> READY
   │                                   │
   └──(miss)──> READ_PENDING ──(grant)──> DATA_RETURN ──> READY
                                                       │
READY ──(pipe_available && local_src_slot)─────────────┘
   │
   v
ISSUED ──> EXECUTING ──> OUTPUT_BUFFERED ──> WRITEBACK_DONE
                    │             │
                    │             └──(dependent hit)──> internal wakeup
                    └──(long latency)──> keep scoreboard entry active
```

### 13.4 Resolve and Retirement Protocol

1. PE completion → `resolve_valid` + `resolve_block_id` to BCC.
2. BCC records resolve status in reorder/retire structure.
3. Retirement occurs **in order** — a Block retires only when:
   - It has resolved.
   - All earlier Blocks have already retired.
   - (For TMA) Memory-ordering constraints are satisfied.
4. Retirement releases Tile dependencies, GPR dependencies, and downstream state.

---

## 14. Backpressure and Flow Control

| Blocking Point | Trigger | Upstream Reaction | Downstream Impact |
|---|---|---|---|
| TileOp Issue Queue full | No available TileOp entry | BCC stops dispatching new Vector Blocks | No new src0/src1/src2 generated |
| Operand Collector busy | Source lookup or read request queue full | TileOp pauses new uOp generation | uOp Issue Queue does not accept new uOps |
| Read Arbitration pending | RR does not grant | uOp group waits for sources | Wakeup delayed until grant |
| Global Src Buffer full | TileReg return has no free buffer entry | TileReg return path backpressures; new read grants suppressed | Read path stalls |
| uOp Queue full | Target opclass queue full | Operand Collector pauses ready uOp dispatch | TileOp may continue waiting |
| Execute busy | Target execution unit busy | uOp Queue holds ready uOps | — |
| Output Buffer full/locked | No result entry available or target window locked | Execute output backpressures; issue suppressed | Backpressure propagates to issue |
| TileReg write busy | Write arbitration does not grant | Output Buffer keeps unwritten entries | — |

**Minimum constraint:** No buffer may drop TileOp, uOp, data, or tag when full. If a stall affects dependency visibility, the Output Buffer forwarding entries must be preserved.

---

## 15. Physical Implementation Considerations

### 15.1 Critical Timing Path

The **only** path explicitly flagged as physically sensitive in the diagrams:

```text
Read Port Arbitration → Tile Register File → Global Src Buffer
```

- Normal propagation: ~2 cycles
- With routing congestion/detour: ~2–3 cycles
- Recommendation: Place arbitration logic close to TileReg; Global Src Buffer near TileReg exit.

### 15.2 Recommended Relative Floorplan

1. `Read Port Arbitration` adjacent to `TileReg`.
2. `Global Src Buffer` near TileReg execution-side exit.
3. `FMLA/FCVT Pipe`, `IALU/PERM/MAC Pipe`, and `SFU` horizontally arranged.
4. Pipe-local Src/Output/Forward structures close to each pipe's entry/exit.
5. Global `Output Buffer` near `Operand Collector / dispatch` area for efficient dependency lookups.

### 15.3 Expected Implementation Challenges

1. **TileReg fanout** — the 1 MB buffer must serve three clients simultaneously at high bandwidth.
2. **Read arbitration control lines** — control signals may interleave with data lines, causing congestion.
3. **Pipe span** — two execution pipes stretched too far could make shared-structure connections overly long.
4. **Multi-porting** — over-provisioning ports to satisfy all requesters may blow up area, timing, and power.

### 15.4 Mitigation Directions

- Formalize TileReg banking strategy before RTL.
- Define Global Src Buffer entry release and tag matching rules.
- Organize Output Buffer tags for efficient CAM lookup.
- Clearly define read/write conflict resolution semantics.

---

## 16. Performance Objectives and Bottlenecks

### 16.1 Dual-Pipeline Goal

The dual execution pipelines are designed to **improve compute-unit utilization** (not peak throughput). This means:
- Different opclasses should map to different pipes.
- Long-latency functional units should not block lightweight operators.
- Cross-pipe forwarding via Output Buffer maintains data flow.

### 16.2 Expected Bottlenecks

1. **Read Port Arbitration** — fairness and conflict resolution under mixed Vector/Cube/TMA traffic.
2. **TileReg port complexity** — physical implementation of 4-bank 512B/cycle dual-port.
3. **Global Src Buffer depth** — whether ~6 entries sufficiently absorbs 2–3 cycle read latency variation.
4. **Output Buffer capacity** — whether it can simultaneously cover forwarding, cross-pipe data movement, reduction reuse, and deferred writeback.
5. **Load balance** — whether the fixed FMLA/FCVT, IALU/PERM/MAC, SFU partition matches actual workload characteristics.

### 16.3 Waste Awareness

The diagrams explicitly note potential waste: statically allocated ports may not be saturated by all workloads. Different data paths may need sharing/steering strategies to avoid one path being congested while another sits idle.

---

## 17. Verification Coverage

Verification should cover the following architectural behavior:

### 17.1 BCC Tests

1. `BStart/B.IOT/TLOAD` decoding and single-target PE dispatch.
2. BCC scalar pipe behavior: AB pipe (ALU+branch 2-pick), AM pipe (multicycle/multiply), LSU (scalar load/store).
3. GPR scalar value exchange across Blocks.
4. Out-of-order resolve reception and in-order Block retirement.
5. Tile dependency tracking and release upon retirement.
6. TMA memory-ordering constraints for load/store serialization.

### 17.2 Vector Core Tests

7. Vector 4 KB three-source TileOp execution through two 2 KB read windows and eight 512 B compute slices.
8. Output Buffer hit → no TileReg read bandwidth consumed.
9. Output Buffer 2 KB lock/unlock behavior and unlocked entry writeback arbitration.
10. Reduction intermediate residency in Output Buffer (no intermediate TileReg writeback required).
11. RR read arbitration across Vector/Cube/TMA concurrent requests.
12. Conflict-free 2 KB continuous read windows across TileReg logical banks.
13. SFU EXP/DIV 256 B/cycle behavior and mask Tile input.
14. uOp opclass → execution unit fixed mapping (§5.5).
15. Cross-pipe forwarding through global Output Buffer.

### 17.3 TMA and Cube Tests

16. TMA 2 KB TileReg requests, 256 B memory beats, bidirectional load/store.
17. TMA memory ordering with mixed load/store sequences.
18. Cube L0A/L0B buffering and large-Tile input/output.

### 17.4 Pipeline and Backpressure Tests

19. Vector pipeline backpressure propagation: TileOp Window → Operand Collector → Read Arbitration → Global Src Buffer → uOp Queue → Execute → Output Buffer → TileReg writeback.
20. Global Src Buffer full → TileReg read return backpressure.
21. Write-port arbitration with simultaneous Output Buffer, Cube, and TMA writes.
22. Resolve returning out of order while retirement remains in order.

### 17.5 Verification Principles

- `block_id/tileop_id/uop_id/tag` must remain consistent across the entire read → execute → writeback chain.
- Output Buffer hit must **not** consume TileReg read bandwidth.
- Output Buffer full must **not** discard forwarding-visible results.
- TMAX dependency chain must not introduce unnecessary bubbles under 4-cycle producer-consumer latency.

---

## 18. Glossary

| Term | Meaning |
|---|---|
| BCC | Block Control Core — Block-level control, dependency resolution, scalar execution, dispatch, and retirement |
| Block | Task granularity visible to BCC; dispatched to a single PE |
| TileOp | Tile-level Vector Core operation (4 KB source Tile) |
| uOp | Fine-grained operation issued to an execution unit (512 B slice) |
| TileReg | Shared Janus4K Tile register/buffer (1 MB, 4 banks, 2 KB window) |
| Output Buffer | Global result residency, forwarding, writeback arbitration, and reduction reuse structure |
| Global Src Buffer | 2 KB entry buffer after TileReg read return (~6 entries) |
| BStart | Block descriptor start instruction (opcode, target PE, dtype/packing, attrs) |
| B.IOT | Tile input/output binding instruction (3 src + 1 dst) |
| TLOAD | TMA load-type Tile transfer instruction |
| GPR | BCC scalar general-purpose register file |
| SFU | Special Function Unit (EXP, DIV at 256 B/cycle) |
| Resolve | Block completion feedback from PE to BCC (out-of-order capable) |
| RR | Round-Robin — arbitration policy for TileReg read requests |
| L0A/L0B | Cube internal buffers for TileReg source data |
| Dtype/pack | Data type and packing format (carried by BStart) |
| Mask Tile | Optional Tile providing per-element masking for TileOps |
| TMAX/colmax | Example reduction/compare TileOp — demonstrates 4-cycle dependency latency |

---

## Appendix A: Comparison to Previous Versions

| Aspect | v0.4-draft (AS_draft) | v0.5-public (AS_EN) | v0.5-enhanced (this document) |
|---|---|---|---|
| Pipeline stages | Implicit | V0–V8 labeled | V0–V8 with detailed timing and dependencies |
| uOp fields | Draft (with TBD widths) | Listed | Structured with per-opclass details |
| Dependency model | Referenced | Explicit rules in §3.4 | Extended with flag/side-effect clarification |
| Output Buffer protocol | Lock/writeback described | Described | Lock/unlock state machine in §6.2 |
| Timing model | ~2cy / ~2-3cy / 4cy | Numbers listed | Integrated into parameter table and critical path analysis |
| Backpressure | Listed | Listed | Comprehensive table with upstream/downstream effects |
| Physical implementation | Suggested | Not present | §15 with floorplan and risk analysis |
| Scheduling model | Implicit | Not present | Two-level + state machine in §13 |
| Performance bottlenecks | Implied | Not present | §16 with explicit bottleneck analysis |

---

## Appendix B: Reference Parameters for Implementation

These parameters should be centralized in a shared parameter file (e.g., `janus4k_params.py`) for consistency across RTL, simulation models, and documentation:

```python
# Core data structure parameters
TILEREG_BYTES           = 1 * 1024 * 1024   # 1 MB
TILE_BYTES              = 4096              # 4 KB
TILE_SLICE_BYTES        = 512               # 512 B
TILEREG_BANKS           = 4
TILEREG_BANK_BYTES_PER_CYCLE = 512
TILEREG_RW_BYTES_PER_CYCLE  = 2048          # 2 KB

# Buffer parameters
GLOBAL_SRCBUF_ENTRIES   = 6
GLOBAL_SRCBUF_ENTRY_BYTES = 2048
OUTPUT_BUFFER_ENTRY_BYTES = 512

# Block descriptor constraints
MAX_DST_TILES_PER_BLOCK = 4
MAX_SRC_TILE_DESC_GROUPS_PER_BLOCK = 2
B_IOT_SRC_TILES         = 3
B_IOT_DST_TILES         = 1

# Execution unit parameters
NUM_EXEC_CLASSES        = 3
SFU_BYTES_PER_CYCLE     = 256

# TMA parameters
TMA_REQ_BYTES           = 2048
TMA_MEM_BEAT_BYTES      = 256

# Timing and arbitration
READ_ARB_POLICY         = "RR"
READ_PROP_LATENCY       = 2      # cycles
READ_CONGESTED_LATENCY  = 3      # cycles (upper bound)
REDUCE_DEP_LATENCY      = 4      # cycles (TMAX example)

# Interconnect
NUM_CLIENTS_READ_ARB    = 3      # Vector compute, Cube, TMA
NUM_CLIENTS_WRITE_ARB   = 3      # Output Buffer, Cube, TMA
```
