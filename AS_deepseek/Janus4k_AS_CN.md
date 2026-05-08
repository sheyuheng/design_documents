# Janus4K AI Core 架构规格书

> **版本:** v0.5-enhanced
> **日期:** 2026-04-27
> **状态:** 对外呈现版（中文增强版）
> **英文对照:** `Janus4k_AS_EN.md`（同目录）
> **前一版本:** `Janus4k_AS_EN.md` v0.5-public (2026-04-26)
> **工作稿来源:** `Janus4k_AS_draft.md` v0.4-draft

---

## 目录

1. [概述](#1-概述)
2. [顶层架构](#2-顶层架构)
3. [数据模型](#3-数据模型)
4. [BCC — Block Control Core](#4-bcc--block-control-core)
5. [Vector Core](#5-vector-core)
6. [Output Buffer](#6-output-buffer)
7. [TileReg 仲裁与缓冲](#7-tilereg-仲裁与缓冲)
8. [Cube](#8-cube)
9. [TMA](#9-tma)
10. [Vector 执行流程](#10-vector-执行流程)
11. [接口定义](#11-接口定义)
12. [参数表](#12-参数表)
13. [调度、唤醒与发射模型](#13-调度唤醒与发射模型)
14. [反压与流控](#14-反压与流控)
15. [物理实现考量](#15-物理实现考量)
16. [性能目标与瓶颈](#16-性能目标与瓶颈)
17. [验证覆盖](#17-验证覆盖)
18. [术语表](#18-术语表)

---

## 1. 概述

Janus4K 是一个以共享 `TileReg` 为中心的 tile-centric AI Core 子系统。执行组织层次为 `Block → TileOp → uOp`。`BCC` 负责 Block 级依赖解析、标量执行、任务派发、resolve 跟踪和顺序退休。`Vector Core`、`Cube` 和 `TMA` 分别执行向量计算、矩阵计算和 Tile/Memory 搬运。

### 1.1 核心设计原则

1. **`TileReg`** 是 AI Core 内部共享数据 buffer，总容量 **1 MB**。
2. **Vector Tile** 固定为 **4 KB**。Cube/TMA 可使用 4 KB 整数倍的大 Tile。
3. **512 B** 是 Vector 计算 slice、TileReg bank slice 和 `Output Buffer` entry 的统一局部粒度。
4. **2 KB** 是 TileReg 对外一次满带宽读/写窗口，由 4 个逻辑 bank 各提供 512 B 组成。
5. **`Output Buffer`** 是全局唯一的结果驻留、前递、写回仲裁和 reduction reuse 结构。
6. Block 完成反馈（resolve）允许**乱序返回**，BCC 按 Block **顺序退休**。

### 1.2 架构边界

本规格定义 Janus4K AI Core 内部的 Block 控制、Tile 数据组织、Vector/Cube/TMA 协同、TileReg 读写仲裁、Output Buffer 前递与写回语义。软件栈、编译器 IR、DDR 控制器内部协议、NoC 协议和具体 SRAM 宏实现**不在本文定义范围内**。外部模块必须遵守本文给出的 Block descriptor、Tile 粒度、接口握手和完成语义。

### 1.3 可见行为

Janus4K 对上层软件和验证环境暴露以下稳定行为：

| 行为 | 架构定义 |
|---|---|
| Block 派发 | BCC 在依赖满足后将 Block 派发到且只派发到一个目标 PE |
| 数据交换 | Vector、Cube、TMA 均通过 TileReg 交换 Tile 数据 |
| Vector Tile | 每个 Vector Tile 固定 **4 KB** |
| 读写窗口 | TileReg 满带宽读写窗口固定 **2 KB** |
| 结果前递 | Output Buffer 中 forwardable 结果可在写回 TileReg 前被 consumer 使用 |
| 完成反馈 | PE 返回 `resolve + block_id`；BCC 顺序退休 |

### 1.4 设计目标

1. 以 **TileReg** 为中心统一承载向量执行、Cube 矩阵计算和 TMA 搬运的数据交换。
2. 支持 Block 描述符拆成 BCC 指令：`BStart`、`B.IOT (3 src + 1 dst)` 和 TMA `TLOAD`。
3. 在多周期执行和多级数据依赖下，通过 **Output Buffer** 和 **Data Forwarding** 尽量减少 bubble。
4. 将"数据就绪"、"依赖满足"和"执行流水可用"三类问题解耦处理，降低单点阻塞。
5. 通过双执行流水将不同类型算子拆分到不同功能路径，提升实际 compute-unit utilization。
6. 承认读路径拥塞会显著拉长供数时延，并通过缓冲和调度隐藏这一时延。
7. 在小容量、近执行体的本地数据结构中维持热工作集，避免每次都走更远、更慢的层级。

---

## 2. 顶层架构

### 2.1 顶层模块

| 模块 | 职责 |
|---|---|
| `BCC` | Block Control Core。负责 Block descriptor 解码、Tile 依赖解析、标量 pipe 执行、任务派发、resolve 跟踪和顺序退休。 |
| `Vector Core` | 执行 Vector TileOp/uOp，主计算粒度 **512 B/cycle**。 |
| `Cube` | 矩阵计算单元。使用 L0A/L0B buffer 接收 TileReg 数据，通过脉动阵列消费。 |
| `TMA` | Tile Memory Access 单元。负责 TileReg 与 DDR/Memory 之间的双向搬运。 |
| `TileReg` | AI Core 共享数据 buffer，总容量 **1 MB**，4 个逻辑 bank。 |
| `Memory` | 外部或上级 memory 系统。 |

### 2.2 顶层框图

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

### 2.3 数据中心化组织

`TileReg` 位于 Vector/Cube/TMA 的数据交汇点。Vector Core、Cube 和 TMA 通过读写仲裁共享 TileReg 端口。数据在 TileReg、Global Src Buffer、Output Buffer 以及各执行单元之间流动。

### 2.4 控制路径与数据路径

Janus4K 将控制路径和数据路径分离：

| 路径 | 起点 | 终点 | 内容 |
|---|---|---|---|
| Block control | BCC | Vector/Cube/TMA | `block_id`、opcode、target PE、Tile 描述、dtype/packing、memory ordering |
| Resolve control | Vector/Cube/TMA | BCC | `resolve_valid`、`resolve_block_id`、完成状态 |
| Tile read | Vector/Cube/TMA | TileReg | 2 KB 读窗口请求 |
| Tile return | TileReg | Global Src Buffer / Cube / TMA | 2 KB 读返回数据和 tag |
| Result writeback | Output Buffer / Cube / TMA | TileReg | 512 B 或聚合后的写回数据 |
| Memory transfer | TMA | Memory | 256 B beat load/store |

控制路径必须保持 Block 身份。数据路径必须保持 Tile ID、slice/window ID、tag 和 mask/dtype 语义，使前递、写回、resolve 和退休能够正确关联。

### 2.5 单目标 PE 派发

每个 Block 在 `BStart` 中携带唯一 `target PE`。BCC **不**复制同一个 Block 到多个 PE 执行。需要多个 PE 协同完成的工作由多个 Block 表达，每个 Block 独立建立 Tile 输入输出依赖，并独立返回 `resolve + block_id`。

---

## 3. 数据模型

### 3.1 Tile

| 项目 | 定义 |
|---|---|
| Vector Tile | 固定 **4 KB** |
| Cube/TMA Tile | 4 KB 的整数倍 |
| Tile slice | **512 B** |
| TileReg 访问窗口 | **2 KB** |
| Tile 地址组织 | 单个 4 KB Tile 在 TileReg 内地址连续 |

一个 **4 KB** Vector Tile 由 8 个 **512 B** slice 组成。TileReg 以 512B slice 编号映射到 4 个逻辑 bank，使任意 PE 能够无 bank conflict 地读出同一 Tile 的一个 2 KB 连续窗口。

#### Tile Slice 和 Window 组织

| Slice | Byte Range | 2KB Window | 用途 |
|---|---|---|---|
| slice0 | `[0, 512)` | window0 | Vector 第 0 个 512B 计算片段 |
| slice1 | `[512, 1024)` | window0 | Vector 第 1 个 512B 计算片段 |
| slice2 | `[1024, 1536)` | window0 | Vector 第 2 个 512B 计算片段 |
| slice3 | `[1536, 2048)` | window0 | Vector 第 3 个 512B 计算片段 |
| slice4 | `[2048, 2560)` | window1 | Vector 第 4 个 512B 计算片段 |
| slice5 | `[2560, 3072)` | window1 | Vector 第 5 个 512B 计算片段 |
| slice6 | `[3072, 3584)` | window1 | Vector 第 6 个 512B 计算片段 |
| slice7 | `[3584, 4096)` | window1 | Vector 第 7 个 512B 计算片段 |

完整 Vector TileOp 覆盖 window0 和 window1。部分 Tile 操作仍使用 **512 B** slice 作为最小执行粒度，**2 KB** window 作为 TileReg 满带宽读写粒度。

### 3.2 TileReg

TileReg 是 1MB 共享数据 buffer：

| 属性 | 值 |
|---|---|
| 总容量 | **1 MB** |
| 逻辑 bank 数 | **4** |
| 每 bank 读带宽 | 512 B/cycle |
| 每 bank 写带宽 | 512 B/cycle |
| bank 端口 | dual-port |
| 满带宽读窗口 | 2 KB/cycle |
| 满带宽写窗口 | 2 KB/cycle |

读写冲突通过仲裁解决。TileReg 的逻辑 bank 是**架构级 bank**；物理 SRAM 分片、bank 复制或端口实现由 RTL/后端实现选择，但必须保持上述可见带宽和无冲突窗口语义。

#### TileReg 请求字段

| 字段 | 语义 |
|---|---|
| `tile_id` | TileReg 中的 Tile 编号 |
| `window_id` | 2 KB window 编号（Vector Tile 内为 0 或 1） |
| `slice_base` | 512 B slice 起点 |
| `bytes` | 请求字节数（满带宽窗口为 2048） |
| `client` | Vector compute、Cube 或 TMA |
| `tag` | 返回数据、wakeup、写回和调试匹配用身份 |

TileReg 保证同一 `tile_id + window_id` 的 **2 KB** 读可以在一次 grant 后以无 bank conflict 的方式返回。写回可以按 512 B entry 写入，也可以在实现中聚合为更宽写窗口；架构可见结果必须保持 slice 顺序和 byte mask 正确。

### 3.3 Block Descriptor

Block descriptor 由 BCC 解码。每个 Block descriptor 以 `BStart` 开始，并由一条或多条 BCC 指令描述输入、输出和搬运属性。

#### BCC 指令

| 指令 | 字段 | 语义 |
|---|---|---|
| `BStart` | opcode, target PE, dtype/packing, attrs | 标记 Block 起始、操作类别、目标 PE 和数据格式 |
| `B.IOT` | 3 src Tile + 1 dst Tile | 绑定 Tile 输入输出 |
| `TLOAD` | memory addr, dst Tile, attrs | TMA load 类 Tile 搬运指令 |

#### 约束

1. 一个 Block **只能**派发给一个目标 PE。
2. 一个 Block 的 dst Tile 最多为 **4** 个。
3. 一个 Block 的 src Tile 描述字段最多为 **2** 组。
4. 单条 `B.IOT` 描述 `3 src + 1 dst`；多条 `B.IOT` 组合表达多输入/多输出 Block。
5. `BStart` 携带 dtype/packing，Vector TileOp/uOp 使用该信息解释 element width 和数据布局。

#### BStart 逻辑字段

| 字段 | 语义 |
|---|---|
| `block_id` | Block 唯一身份 |
| `opcode` | Block 操作码 |
| `target_pe` | Vector、Cube 或 TMA |
| `dtype_pack` | 数据类型和 packing |
| `elem_width` | element width |
| `mask_en` | 是否使用 mask Tile |
| `reduction_en` | 是否为 reduction 类操作 |
| `memory_order_attr` | TMA memory ordering 属性 |

`B.IOT` 展开后形成 Tile 依赖表项。每个表项包含源 Tile、目的 Tile、读写属性、mask 属性和目标 PE。BCC 以这些表项建立 Block 间依赖，并在 Block 退休时释放对应依赖。

### 3.4 依赖模型

Block 之间通过 Tile 输入输出建立依赖。规则如下：

- **普通计算 Block** 不建立 flag 依赖，也不建立普通 memory side-effect 依赖。
- **TMA 的 `TLOAD`/store 操作** 执行内存保序。
- **GPR** 用于不同 Block 之间的标量值交互。标量值依赖由 BCC scalar pipe 和 GPR 维护。
- **不引入 flag 依赖**。
- Block 退休时释放 Tile 依赖、GPR 依赖以及后续 Block 可见的状态。

---

## 4. BCC — Block Control Core

### 4.1 职责

`BCC` 是 Janus4K 的 Block 级控制核心，负责：

1. 解码 `BStart` / `B.IOT` / `TLOAD`。
2. 建立 Tile 输入输出依赖追踪。
3. 执行 BCC scalar pipe 指令。
4. 将 ready Block 派发到 Vector Core、Cube 或 TMA。
5. 接收 PE 返回的 `resolve + block_id`。
6. 支持 resolve 乱序返回和 Block 顺序退休。

### 4.2 Scalar Pipe

BCC 内部包含 GPR 和三类 scalar pipe：

| 结构 | 功能 |
|---|---|
| `GPR` | 通用标量寄存器文件，用于 Block 间标量值交互 |
| `AB pipe` | ALU + branch，支持 2-pick |
| `AM pipe` | ALU + multicycle，支持标量乘法 |
| `LSU` | 标量 load/store |

一个 Block 写入 GPR 的标量值可以被后续 Block 读取使用，提供轻量级的跨 Block 通信机制。

### 4.3 Resolve 与退休

PE 完成 Block 后向 BCC 返回 `resolve + block_id`。协议如下：

1. **resolve 可以乱序返回**。后派发的 Block 可能先完成。
2. BCC 在 retirement 结构中记录完成状态。
3. BCC **严格按程序顺序**退休。
4. 退休时释放：
   - Tile 依赖（Tile ID 对后续 Block 可用）
   - GPR 依赖
   - 后续 Block 可见状态
5. TMA Block 在满足内存保序约束后才能退休。

---

## 5. Vector Core

### 5.1 Vector Core 结构

Vector Core 是 Janus4K 中最主要的可编程向量执行单元。它接收 BCC 派发的 Vector Block，将 Block 内 TileOp 拆成 uOp，并通过固定执行资源完成计算。

| 结构 | 职责 |
|---|---|
| `Vector Block / TileOp Window` | 接收 Vector Block，保存 TileOp 状态，维护 Tile 级依赖和 wakeup |
| `TileOp Decoder` | 根据 opcode、dtype/packing、element width、mask 和 reduction 属性生成 uOp 序列 |
| `Operand Collector` | 对源 Tile 先查 Output Buffer，再为 miss 源生成 TileReg read request |
| `Read Port Arbitration` | 在 Vector compute、Cube、TMA 之间对 TileReg 读请求执行 RR 仲裁 |
| `Global Src Buffer` | 保存 TileReg 返回的 **2 KB** 源数据窗口，深度 ~6 entries |
| `uOp Issue Queue` | 按执行类别维护 ready uOp，并向固定执行单元发射 |
| `FMLA/FCVT Pipe` | 执行 FMLA 和 FCVT/CVT 类操作 |
| `IALU/PERM/MAC Pipe` | 执行 IALU、PERM、MAC、compare 和 reduction 类操作 |
| `SFU` | 执行 EXP/DIV 等特殊函数，吞吐 **256 B/cycle** |
| `Output Buffer` | 保存执行结果，支持 forwarding、2KB lock、writeback arbitration 和 reduction reuse |

### 5.2 Vector Pipeline 总览

Vector Core pipeline 分为前端、取数、发射、执行、结果和完成八段：

| 阶段 | 名称 | 主要结构 | 输入 | 输出 |
|---|---|---|---|---|
| V0 | Block accept | Vector Block | BCC Vector Block | TileOp window entry |
| V1 | TileOp decode | TileOp Decoder | opcode, B.IOT, dtype/packing, mask | uOp plan |
| V2 | Source collect | Operand Collector | src Tile desc, Output Buffer tag | forward hit 或 read request |
| V3 | Read arbitration | Read Port Arbitration | Vector/Cube/TMA read request | 2KB read grant |
| V4 | TileReg read return | TileReg + Global Src Buffer | 2KB read window | source window ready |
| V5 | uOp issue | uOp Issue Queue | ready uOp + source window | pipe/SFU issue |
| V6 | Execute | FMLA/FCVT, IALU/PERM/MAC, SFU | 512B/256B operand slice | result entry |
| V7 | Result buffer | Output Buffer | result entry | forward hit / writeback request |
| V8 | Completion | Vector Block | TileOp done bitmap | `resolve + block_id` |

**Pipeline 关键特性：**

- 多个 TileOp 可以同时驻留在 pipeline 中。
- 不同 TileOp 可以处在不同阶段。
- 同一个 TileOp 的两个 2 KB window 可以流水化处理。
- V3（读仲裁）、V5（uOp issue）、V7（Output Buffer 写入）可以独立产生反压。

### 5.3 TileOp 到 uOp 的拆分

每个 Vector TileOp 的源输入是 **4 KB srcTile**。标准逐元素三源操作按两个 2 KB window 拆分，每个 window 再拆成 4 个 512 B uOp：

```text
TileOp(4KB)
  window0: slice0, slice1, slice2, slice3
  window1: slice4, slice5, slice6, slice7
```

#### 每 Window 的读取和计算序列

三源 TileOp 的每个 window 固定执行三次源读，随后执行计算 uOp：

```text
windowN:
  read srcA[windowN]  -> 2KB
  read srcB[windowN]  -> 2KB
  read srcC[windowN]  -> 2KB
  issue 4 x 512B uOp
```

#### uOp 描述字段

| 字段 | 语义 |
|---|---|
| `block_id` | 所属 Block |
| `tileop_id` | 所属 TileOp |
| `uop_id` | TileOp 内 uOp 序号 |
| `opclass` | FMLA/FCVT、IALU/PERM/MAC 或 SFU |
| `src_desc[0..2]` | 源 Tile/window/slice 描述 |
| `mask_tile_desc` | 可选 mask Tile |
| `dst_desc` | 输出 Tile/window/slice 描述 |
| `dtype_pack` | BStart 携带的数据类型与 packing |
| `elem_width` | element width |
| `window_id` | 2KB window 编号 |
| `slice_id` | 512B slice 编号 |
| `forwardable` | 结果是否允许进入 Output Buffer 后被 forwarding |
| `reduction_ctx` | reduction 链上下文 |

**各 opclass 细节：**

- **逐元素操作** (FMLA/FCVT/IALU/PERM/MAC)：以 **512 B** slice 粒度执行。
- **SFU 操作** (EXP/DIV)：以 2 KB window 读取源数据，执行侧以 **256 B/cycle** 消费。两个连续 256B 结果组合成一个 512 B Output Buffer entry。
- **Reduction 操作**：按 reduction 方向、element width 和 reduction tree 生成 uOp，允许中间结果驻留在 Output Buffer。

### 5.4 取数 Pipeline

Operand Collector 实现**先查 Output Buffer，再决定读 TileReg** 的策略：

```text
src desc
  ├─ Output Buffer hit  -> lock 2KB window -> source ready (forwarding)
  └─ Output Buffer miss -> enqueue TileReg read request -> wait grant
```

#### 取数规则

| 规则 | 定义 |
|---|---|
| Lookup 优先级 | Output Buffer lookup 先于 TileReg read request |
| Hit 行为 | hit 源不消耗 TileReg 读口，对应 2KB Output Buffer window 被**锁定** |
| Miss 行为 | miss 源进入 Read Port Arbitration |
| Grant 粒度 | 每次 grant 读取一个 **2 KB** window |
| 三源顺序 | Vector 三源 window 固定按 **srcA → srcB → srcC** 发起读请求 |
| 返回位置 | TileReg 返回数据进入 Global Src Buffer |
| Wakeup | 三个源 window 都 ready 后唤醒对应 uOp group |

#### Global Src Buffer

- Entry 粒度：**2 KB**
- 深度：**~6 entries**（小缓冲用于隐藏 TileReg 读延迟）
- 一个三源 window 最多需要三个 entry（srcA、srcB、srcC）
- 来自 Output Buffer forwarding 的源**不**占用 Global Src Buffer entry

### 5.5 uOp Issue Queue

uOp Issue Queue 分为三类固定队列，各有专属执行单元：

| Queue | 执行单元 | 覆盖操作 |
|---|---|---|
| `FMLA/FCVT Queue` | `FMLA/FCVT Pipe` | FMLA、FCVT/CVT |
| `IALU/PERM/MAC Queue` | `IALU/PERM/MAC Pipe` | IALU、PERM、MAC、compare、reduction、TMAX/colmax |
| `SFU Queue` | `SFU` | EXP、DIV 等特殊函数 |

#### uOp Ready 条件

uOp 进入 ready 状态需要满足：
1. 所有源 window 已 ready。
2. mask Tile 已 ready 或该 uOp 不使用 mask。
3. 目标执行单元可接收。
4. Output Buffer 对该结果有可分配 entry 或可形成 backpressure。
5. reduction uOp 的前序 reduction_ctx 已 ready。

**执行单元隔离：** 三类执行资源**不共享**执行单元。`opclass` 到执行单元的映射固定。调度器不将 SFU uOp 发射到普通 Vector pipe，也不将 FMLA/FCVT uOp 发射到 IALU/PERM/MAC Pipe。

### 5.6 执行 Pipeline

Vector 执行侧包含两条 512B 主执行路径和一条 256B SFU 路径：

| 执行路径 | 输入粒度 | 输出粒度 | 典型操作 | 结果去向 |
|---|---|---|---|---|
| `FMLA/FCVT Pipe` | 512 B/cycle | 512 B entry | FMLA、FCVT/CVT | Output Buffer |
| `IALU/PERM/MAC Pipe` | 512 B/cycle | 512 B entry | IALU、PERM、MAC、TMAX/colmax | Output Buffer |
| `SFU` | 256 B/cycle | 聚合为 512 B entry | EXP、DIV | Output Buffer |

普通 512B pipe 的结果以 **512 B** entry 写入 Output Buffer。SFU 每拍消费 **256 B**，两个连续 256B 结果组合成一个 **512 B** Output Buffer entry。

执行单元输出必须携带 `block_id/tileop_id/uop_id/dst_desc/slice_id/tag`，用于 Output Buffer lookup、writeback 和完成统计。

### 5.7 典型三源 TileOp 时序

标准三源 4 KB TileOp 的读数和计算顺序如下：

| 步骤 | 动作 | 数据量 | 说明 |
|---|---|---|---|
| 1 | window0 srcA read | 2KB | RR 仲裁 |
| 2 | window0 srcB read | 2KB | RR 仲裁 |
| 3 | window0 srcC read | 2KB | RR 仲裁 |
| 4 | window0 compute | 4 × 512B | 发射 4 个 512B uOp |
| 5 | window1 srcA read | 2KB | RR 仲裁 |
| 6 | window1 srcB read | 2KB | RR 仲裁 |
| 7 | window1 srcC read | 2KB | RR 仲裁 |
| 8 | window1 compute | 4 × 512B | 发射 4 个 512B uOp |
| 9 | result residency/writeback | 8 × 512B | 结果进入 Output Buffer，按依赖和写口状态写回 |

**优化：** 当某个源在 Output Buffer 命中时，对应 read 步骤被 forwarding 替代，不发起 TileReg read。

### 5.8 Wakeup 与依赖

Vector Core 使用两类 wakeup：

| Wakeup 类型 | 来源 | 作用 |
|---|---|---|
| External wakeup | TileReg read grant/return | 唤醒等待 TileReg 数据的 uOp group |
| Internal wakeup | Output Buffer result ready | 唤醒 producer-consumer 或 reduction 链上的后继 uOp |

Output Buffer hit 的 consumer 在锁定对应 2KB window 后进入 source-ready 状态。Reduction 链上的后继 uOp 可以直接消费 Output Buffer 中的中间结果，不需要等待中间结果写回 TileReg。

### 5.9 Vector 反压

Vector pipeline 每级都保持 **valid/ready** 语义。任一级无法接收时，上游保持 payload 稳定，不丢失 tag 和数据。

| 阻塞点 | 触发条件 | 反压方向 |
|---|---|---|
| TileOp Window full | 无可用 TileOp entry | BCC 停止派发新的 Vector Block |
| Operand Collector busy | 源 lookup 或 read request 队列满 | TileOp 暂停生成新 uOp |
| Read Arbitration pending | RR 未 grant | uOp group 保持等待源状态 |
| Global Src Buffer full | TileReg 返回无 entry 可写 | TileReg read 返回路径反压，抑制新 read grant |
| uOp Queue full | 对应 opclass queue 无 entry | Operand Collector 暂停 dispatch ready uOp |
| Execute busy | 目标执行单元不可接收 | uOp Queue 保持 ready uOp |
| Output Buffer full/locked | 无可写 result entry 或目标 window 锁定 | Execute 输出反压，抑制 issue |
| TileReg write busy | 写口仲裁未 grant | Output Buffer 保持未写回 entry |

---

## 6. Output Buffer

`Output Buffer` 是位于执行结果和 TileReg 写回之间的**全局唯一结构**。它提供：

1. **结果存储** — 512 B entry 粒度。
2. **Producer-consumer forwarding** — 依赖源命中后避免 TileReg 读。
3. **跨 Pipe forwarding** — Pipe0/Pipe1/SFU 结果可被任一 pipe 消费。
4. **TileReg 写口仲裁** — 解决 Output Buffer、Cube、TMA 之间的写冲突。
5. **Reduction 中间结果复用** — 结果在多个 reduction 步骤间保持驻留。
6. **结果驻留** — entry 可以在写回前持续存在。

### 6.1 Output Buffer Entry 字段

| 字段 | 语义 |
|---|---|
| `valid` | entry 是否有效 |
| `tag` | forwarding/writeback 匹配 tag |
| `tile_id` | 目标 Tile |
| `window_id` | 2KB window |
| `slice_id` | 512B slice |
| `data` | 512B 结果 |
| `byte_mask` | byte 有效 mask |
| `forwardable` | 是否允许 forwarding |
| `writeback_required` | 是否需要写回 TileReg |
| `locked` | 所属 2KB window 是否被 consumer 锁定 |
| `reduction_ctx` | reduction 链上下文 |

### 6.2 锁定与写回协议

1. 当 TileOp 查到源数据在 Output Buffer 中时，对应的 **2 KB 数据窗口被锁定**。
2. 锁定期间，该窗口**不允许被覆盖或释放**。
3. consumer 使用完成后，窗口**解锁**。
4. 未被锁定的 entry 每拍尝试发出 TileReg 写回请求。
5. 写回请求经过 TileReg 写口仲裁。
6. Reduction 中间结果可长期驻留在 Output Buffer 中，不需要每步写回 TileReg。
7. Reduction 最终结果在满足写回条件后写回目标 Tile。

### 6.3 Output Buffer 为什么必须优先于写回存在

没有 Output Buffer：
- 链式依赖会大量回退到 TileReg 读。
- 多周期结果无法被快速消费。
- 写口冲突会直接堵住执行尾部。
- 列方向 reduction（如 `colmax`）会出现明显 bubble。

---

## 7. TileReg 仲裁与缓冲

### 7.1 Read Port Arbitration

TileReg 读仲裁接收三类 client 请求：
1. `Vector compute`
2. `Cube`
3. `TMA`

#### 仲裁策略

- 三类 client 之间采用 **Round-Robin (RR)** 仲裁。
- 每次 grant 对应一个 **2 KB** TileReg 读窗口。
- 未获 grant 的请求保持 pending，后续 RR 轮次继续参与。
- RR 状态更新以实际 grant 为准，空队列不能破坏有效请求的公平性。

#### 时序特性

| 条件 | 延迟 |
|---|---|
| 正常传播 | ~2 cycles |
| 布线拥塞（绕线） | ~2–3 cycles |

### 7.2 Global Src Buffer

Global Src Buffer 位于 TileReg 返回路径上：

| 属性 | 值 |
|---|---|
| Entry 粒度 | **2 KB** |
| 深度 | **~6 entries** |
| 用途 | 吸收 Read Port Arbitration → TileReg 的可变延迟；解耦 TileReg 供给节奏与执行消耗节奏 |

进入执行单元前，2 KB 数据窗口可拆成 4 个 512 B slice。

### 7.3 写口仲裁

TileReg 写冲突通过写口仲裁解决。写请求来源包括：
- `Output Buffer`
- `Cube`
- `TMA`

**保证：**
- 数据、tag、tile_id、slice_id 的一致性得到保持。
- forwarding-visible 结果不会因写口阻塞而丢失。
- 未被锁定的 Output Buffer entry 每拍尝试发出写回请求。
- 写仲裁不破坏依赖可见性所需的结果顺序。

---

## 8. Cube

Cube 是 Janus4K 的矩阵计算单元。

| 属性 | 定义 |
|---|---|
| 数据接口 | 输入输出以 Tile 表示（可使用多个 4 KB Tile 组成的大 Tile） |
| 内部缓冲 | L0A/L0B buffers — 从 TileReg 读入数据后存入本地 buffer，再经脉动阵列消费 |
| 消费粒度 | 内部脉动阵列决定，不受外部 512 B slice 约束 |
| 完成反馈 | 完成后向 BCC 返回 `resolve + block_id` |

Cube 遵循以下架构规则：
1. 通过共享读仲裁从 TileReg 读取数据（与 Vector compute 和 TMA 共享 RR 仲裁）。
2. 数据进入 L0A/L0B buffer，解耦 TileReg 仲裁时序与阵列消费节奏。
3. 可使用 4 KB 整数倍的大 Tile。
4. 结果写回 TileReg（参与写口仲裁）。
5. 精确阵列规模、L0A/L0B 深度、累加器宽度和输出 layout 不在本文定义范围内。

---

## 9. TMA

TMA 负责 TileReg 与 DDR/Memory 之间的双向数据搬运。

| 属性 | 定义 |
|---|---|
| 方向 | 双向 load/store |
| TileReg 单次请求上限 | **2 KB** |
| Memory 侧 beat | **256 B** |
| 读写口 | 分离 |
| 队列 | load/store 共享调度队列 |
| 保序 | TLOAD/store 执行内存保序 |

### 9.1 TMA 细节

- **TMA load:** 将 memory 数据搬入 TileReg。
- **TMA store:** 将 TileReg 数据写回 memory。
- **完成反馈:** 完成后向 BCC 返回 `resolve + block_id`。
- **内存保序:** TMA 操作必须遵守 `BStart.memory_order_attr` 指定的保序语义。TMA Block 只有在其保序约束满足后才能退休。
- **TileReg 请求:** 单次 TMA 到 TileReg 的交互限制为 2 KB，与 TileReg 满带宽窗口大小一致。

---

## 10. Vector 执行流程

完整的 Vector 执行流程如下：

1. **BCC 解码** Block descriptor，确认目标 PE 为 Vector Core。
2. **BCC 检查依赖**是否满足（通过 Tile 输入输出关系）。
3. **BCC 派发** ready Vector Block 给 Vector Block / TileOp Window。
4. **TileOp Decoder** 根据 `BStart/B.IOT`、dtype/packing、element width、mask 和 opclass 生成 uOp plan。
5. **Operand Collector** 对每个源执行 Output Buffer lookup。
6. **Output Buffer hit** → 锁定对应 2KB window，通过 forwarding 满足。
7. **Output Buffer miss** → 向 Read Port Arbitration 发起 TileReg read request。
8. **Read Port Arbitration** 按 RR grant 2 KB read window。
9. **TileReg 返回数据**进入 Global Src Buffer。
10. 对应 window 的**所有源 ready 后**，4 个 512B uOp 进入固定 uOp queue。
11. **uOp Issue Queue** 根据 opclass 发射到 FMLA/FCVT、IALU/PERM/MAC 或 SFU。
12. **执行单元**输出 512 B Output Buffer entry（SFU 以两个 256B beat 聚合为 512B entry）。
13. **Output Buffer** 根据 tag 唤醒后继 consumer，或向 TileReg 写回未锁定 entry。
14. 所有 window 和 slice 完成后更新 **TileOp done 状态**。
15. Block 内所有 TileOp 完成后，**Vector Core 向 BCC 返回** `resolve + block_id`。
16. **BCC 按顺序退休**已 resolve 的 Block。

---

## 11. 接口定义

除非特别说明，所有接口默认采用 **valid/ready** 握手：

```text
transfer = valid & ready
```

- `valid`：发送方驱动，表示 payload 有效。
- `ready`：接收方驱动，表示本拍可接收 payload。
- 当 `valid=1` 且 `ready=0` 时，发送方必须保持 payload 稳定。
- 每个事务必须携带至少一个身份字段（`block_id`、`tileop_id`、`uop_id` 或 `tag`）。

### 11.1 BCC 到 PE

| 信号 | 方向 | 含义 |
|---|---|---|
| `block_valid` | BCC → PE | Block 有效 |
| `block_ready` | PE → BCC | 目标 PE 可接收 |
| `block_id` | BCC → PE | Block 标识 |
| `block_target` | BCC → PE | Vector/Cube/TMA 目标 |
| `block_opcode` | BCC → PE | Block 操作类别 |
| `block_bstart` | BCC → PE | BStart 解码结果 |
| `block_iot_vec` | BCC → PE | B.IOT 展开的 Tile 输入输出绑定 |
| `block_attr` | BCC → PE | dtype/packing、element width、reduction、memory ordering 等属性 |

### 11.2 PE 到 BCC

| 信号 | 方向 | 含义 |
|---|---|---|
| `resolve_valid` | PE → BCC | Block 完成反馈有效 |
| `resolve_ready` | BCC → PE | BCC 可接收 resolve |
| `resolve_block_id` | PE → BCC | 完成的 Block ID |
| `resolve_status` | PE → BCC | 完成状态 |

### 11.3 Operand Collector 到 Read Port Arbitration

| 信号 | 方向 | 含义 |
|---|---|---|
| `rd_req_valid` | requester → arb | TileReg 读请求有效 |
| `rd_req_ready` | arb → requester | 仲裁器可接收 |
| `rd_req_client` | requester → arb | Vector compute/Cube/TMA |
| `rd_req_tile_id` | requester → arb | Tile ID |
| `rd_req_slice_base` | requester → arb | 512B slice 或 2KB window 起点 |
| `rd_req_bytes` | requester → arb | 请求字节数，标准 **2 KB** |
| `rd_req_tag` | requester → arb | 返回数据匹配 tag |

### 11.4 Execute/SFU 到 Output Buffer

| 信号 | 方向 | 含义 |
|---|---|---|
| `ob_valid` | exec → OB | 执行结果有效 |
| `ob_ready` | OB → exec | Output Buffer 可接收 |
| `ob_tag` | exec → OB | forwarding/writeback 匹配 tag |
| `ob_data` | exec → OB | **512 B** 结果 entry |
| `ob_forwardable` | exec → OB | 结果允许 forwarding |
| `ob_writeback_req` | exec → OB | 结果需要写回 TileReg |
| `ob_reduction_ctx` | exec → OB | reduction 上下文 |

### 11.5 TMA

| 信号 | 方向 | 含义 |
|---|---|---|
| `tma_cmd_valid` | BCC → TMA | TMA 命令有效 |
| `tma_cmd_ready` | TMA → BCC | TMA 可接收命令 |
| `tma_cmd_is_store` | BCC → TMA | load/store 方向 |
| `tma_tile_id` | BCC → TMA | TileReg Tile ID |
| `tma_tile_bytes` | BCC → TMA | TileReg 单次请求字节数，最大 **2 KB** |
| `tma_mem_addr` | BCC → TMA | memory 地址 |
| `tma_mem_beat_bytes` | BCC → TMA | memory beat，固定 **256 B** |
| `tma_order_tag` | BCC → TMA | 内存保序 tag |

### 11.6 Operand Collector / Src Buffer 到 uOp Issue Queue

| 信号 | 方向 | 含义 |
|---|---|---|
| `uop_valid` | → uOpQ | uOp 已满足最小入队条件 |
| `uop_ready` | uOpQ → | uOp Issue Queue 可接收 |
| `uop_payload` | → uOpQ | uOp 描述字段（见 §5.3） |
| `uop_src_ready_mask` | → uOpQ | 三个源是否已 ready（3-bit mask） |
| `uop_forward_hit_mask` | → uOpQ | 哪些源来自 Output Buffer forwarding |

### 11.7 Output Buffer 到 TileReg（写回）

| 信号 | 方向 | 含义 |
|---|---|---|
| `tr_wr_valid` | → TileReg | 写回请求有效 |
| `tr_wr_ready` | TileReg → | TileReg 写口可接收 |
| `tr_wr_tile_id` | → TileReg | 写回目标 Tile |
| `tr_wr_slice_id` | → TileReg | 写回目标 512B slice |
| `tr_wr_data` | → TileReg | 512B 写回数据 |
| `tr_wr_mask` | → TileReg | 字节/片段写 mask |
| `tr_wr_tag` | → TileReg | 调试/完成跟踪 tag |

---

## 12. 参数表

| 参数 | 值 | 说明 |
|---|---|---|
| `TILEREG_BYTES` | 1 MB | TileReg 总容量 |
| `TILE_BYTES` | 4096 | Vector Tile 容量 |
| `TILE_SLICE_BYTES` | 512 | Vector slice / Output Buffer entry |
| `TILEREG_BANKS` | 4 | TileReg 逻辑 bank 数 |
| `TILEREG_BANK_BYTES_PER_CYCLE` | 512 | 每 bank 每拍读/写带宽 |
| `TILEREG_RW_BYTES_PER_CYCLE` | 2048 | TileReg 满带宽读/写窗口 |
| `GLOBAL_SRCBUF_ENTRY_BYTES` | 2048 | Global Src Buffer entry 粒度 |
| `GLOBAL_SRCBUF_ENTRIES` | 6 | Global Src Buffer 深度 |
| `OUTPUT_BUFFER_ENTRY_BYTES` | 512 | Output Buffer entry 粒度 |
| `MAX_DST_TILES_PER_BLOCK` | 4 | Block dst Tile 上限 |
| `MAX_SRC_TILE_DESC_GROUPS_PER_BLOCK` | 2 | Block src Tile 描述字段组上限 |
| `B_IOT_SRC_TILES` | 3 | 单条 B.IOT src Tile 数 |
| `B_IOT_DST_TILES` | 1 | 单条 B.IOT dst Tile 数 |
| `NUM_EXEC_CLASSES` | 3 | FMLA/FCVT、IALU/PERM/MAC、SFU |
| `SFU_BYTES_PER_CYCLE` | 256 | SFU EXP/DIV 带宽 |
| `TMA_REQ_BYTES` | 2048 | TMA TileReg 单次请求上限 |
| `TMA_MEM_BEAT_BYTES` | 256 | TMA memory 侧 beat |
| `READ_ARB_POLICY` | RR | TileReg 读仲裁策略 |
| `READ_PROP_LATENCY` | 2 cycles | TileReg 读路径正常传播延迟 |
| `READ_CONGESTED_LATENCY` | 2–3 cycles | 拥塞后的读路径延迟 |
| `REDUCE_DEP_LATENCY` | 4 cycles | 示例 reduction 依赖距离（TMAX） |

---

## 13. 调度、唤醒与发射模型

### 13.1 两级调度

| 级别 | 粒度 | 管理方 | 用途 |
|---|---|---|---|
| TileOp 级 | 粗粒度（Tile） | TileOp Issue Queue | 追踪 Tile 依赖和 wakeup 状态 |
| uOp 级 | 细粒度（slice） | uOp Issue Queue | 将 ready uOp 发射到执行单元 |

### 13.2 Wakeup 类型

| Wakeup | 来源 | 作用 |
|---|---|---|
| External wakeup | TileReg read grant → data return | 唤醒等待 TileReg 数据的 uOp group |
| Internal wakeup | Output Buffer result ready | 唤醒 producer-consumer 和 reduction 后继 uOp |

### 13.3 调度状态机

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
                    └──(long latency)──> keep scoreboard active
```

### 13.4 Resolve 与退休协议

1. PE 完成 Block → 向 BCC 发送 `resolve_valid` + `resolve_block_id`。
2. BCC 在 reorder/retire 结构中记录 resolve 状态。
3. 退休**按顺序进行** — 一个 Block 退休需要满足：
   - 自身已 resolve。
   - 所有更早的 Block 均已退休。
   - （TMA 侧）内存保序约束已满足。
4. 退休时释放 Tile 依赖、GPR 依赖和后续 Block 可见状态。

---

## 14. 反压与流控

| 阻塞点 | 触发条件 | 上游反应 | 下游影响 |
|---|---|---|---|
| TileOp Issue Queue full | 无可用 TileOp entry | BCC 停止派发新 Vector Block | 不产生新的 src0/src1/src2 |
| Operand Collector busy | 源 lookup 或 read request 队列满 | TileOp 暂停新 uOp 生成 | uOp Issue Queue 不接收新 uOp |
| Read Arbitration pending | RR 未 grant | uOp group 保持等待源 | Wakeup 延迟到 grant |
| Global Src Buffer full | TileReg 返回无可用 entry | TileReg 返回路径反压，抑制新 read grant | 读路径停顿 |
| uOp Queue full | 目标 opclass queue 满 | Operand Collector 暂停 ready uOp dispatch | TileOp 可能继续等待 |
| Execute busy | 目标执行单元忙 | uOp Queue 保持 ready uOp | — |
| Output Buffer full/locked | 无可用 result entry 或目标 window 锁定 | Execute 输出反压，抑制 issue | 反压传播到 issue |
| TileReg write busy | 写口仲裁未 grant | Output Buffer 保持未写回 entry | — |

**最低约束：** 任何缓冲满时不允许丢失 TileOp、uOp、data 或 tag。如果停顿影响依赖可见性，必须优先保持 Output Buffer 中的 forwarding entry 有效。

---

## 15. 物理实现考量

### 15.1 关键时序路径

图中唯一直接被注释为物理敏感的路径：

```text
Read Port Arbitration → Tile Register File → Global Src Buffer
```

- 正常传播：~2 cycles
- 布线拥塞/绕线：~2–3 cycles
- 建议：Arbitration 紧邻 TileReg，Global Src Buffer 靠近 TileReg 出口

### 15.2 建议相对布局

1. `Read Port Arbitration` 紧邻 `TileReg`。
2. `Global Src Buffer` 放在 TileReg 执行侧出口。
3. `FMLA/FCVT Pipe`、`IALU/PERM/MAC Pipe` 和 `SFU` 水平展开。
4. Pipe 本地 Src/Output/Forward 结构紧贴 Pipe 入口/出口。
5. 全局 `Output Buffer` 靠近 `Operand Collector / dispatch` 区域，便于依赖查询。

### 15.3 预期实现挑战

1. **TileReg 扇出** — 1 MB buffer 需同时为三个 client 提供高带宽。
2. **读仲裁控制线** — 控制信号与数据线交织可能导致拥塞。
3. **Pipe 跨距** — 两条执行 pipe 拉得过长，共享结构连接过远。
4. **多端口化** — 过度多端口化会导致面积、时序和功耗失控。

### 15.4 规避方向

- 在进入 RTL 前正式化 TileReg banking 策略。
- 定义 Global Src Buffer entry 释放和 tag 匹配规则。
- 组织 Output Buffer tag 以高效 CAM lookup。
- 明确定义读写冲突解决语义。

---

## 16. 性能目标与瓶颈

### 16.1 双 Pipe 目标

双执行 pipeline 的核心目标是**提升 compute-unit utilization**（而非峰值吞吐）。这意味着：
- 不同 opclass 应映射到不同 pipe。
- 长时延功能单元不应阻塞轻量算子。
- 通过 Output Buffer 实现跨 pipe forwarding。

### 16.2 预期瓶颈

1. **Read Port Arbitration** — Vector/Cube/TMA 混合流量下的公平性。
2. **TileReg 端口复杂度** — 4-bank 512B/cycle dual-port 的物理实现。
3. **Global Src Buffer 深度** — ~6 entries 能否充分吸收 2–3 cycle 读延迟波动。
4. **Output Buffer 容量** — 能否同时覆盖 forwarding、跨 pipe 数据、reduction 复用和延迟写回。
5. **负载平衡** — FMLA/FCVT、IALU/PERM/MAC、SFU 的固定分区是否匹配实际 workload。

### 16.3 浪费意识

图中明确标注了"可能浪费"：静态分配的端口未必能被所有 workload 吃满。不同数据路径之间可能需要共享/复用策略，否则会出现一边拥塞、一边闲置。

---

## 17. 验证覆盖

验证应覆盖以下架构行为：

### 17.1 BCC 测试

1. `BStart/B.IOT/TLOAD` 解码和单目标 PE 派发。
2. BCC scalar pipe 行为：AB pipe (ALU+branch 2-pick)、AM pipe (multicycle/multiply)、LSU (标量 load/store)。
3. GPR 跨 Block 标量值交互。
4. resolve 乱序接收与 BCC 顺序退休。
5. Tile 依赖追踪和退休时释放。
6. TMA 内存保序约束。

### 17.2 Vector Core 测试

7. Vector 4 KB 三源 TileOp 的两轮 2 KB 读窗口和 8 个 512 B slice 执行。
8. Output Buffer hit 不消耗 TileReg 读口。
9. Output Buffer 2 KB lock/unlock 和未锁定 entry 写回仲裁。
10. Reduction 中间结果驻留 Output Buffer（不要求每步写回 TileReg）。
11. Vector/Cube/TMA 并发请求下的 RR 读仲裁。
12. TileReg 逻辑 bank 下 2 KB 连续读窗口无 bank conflict。
13. SFU EXP/DIV 256 B/cycle 和 mask Tile 输入。
14. uOp opclass → 执行单元固定映射（§5.5）。
15. 通过全局 Output Buffer 的跨 pipe forwarding。

### 17.3 TMA 和 Cube 测试

16. TMA 2 KB TileReg 请求、256 B memory beat、双向 load/store。
17. TMA 混合 load/store 序列的内存保序。
18. Cube L0A/L0B buffering 和大 Tile 输入输出。

### 17.4 Pipeline 和反压测试

19. Vector pipeline 反压传播：TileOp Window → Operand Collector → Read Arbitration → Global Src Buffer → uOp Queue → Execute → Output Buffer → TileReg writeback。
20. Global Src Buffer 满 → TileReg 返回路径反压。
21. Output Buffer、Cube、TMA 同时写入时的写口仲裁。
22. resolve 乱序返回，retirement 保持顺序。

### 17.5 验证原则

- `block_id/tileop_id/uop_id/tag` 在读请求、数据返回、wakeup、执行、写回链路中保持一致。
- Output Buffer hit 时**不消耗** TileReg 读口。
- Output Buffer 满时**不丢弃** forwarding-visible 结果。
- 在 4-cycle producer-consumer 延迟下，TMAX 依赖链不应产生不必要的 bubble。

---

## 18. 术语表

| 术语 | 含义 |
|---|---|
| BCC | Block Control Core — Block 级控制、依赖解析、标量执行、派发和退休 |
| Block | BCC 看到的任务粒度，派发到单个 PE |
| TileOp | Vector Core 的 tile 级操作（4 KB 源 Tile） |
| uOp | 可发射到执行单元的细粒度操作（512 B slice） |
| TileReg | Janus4K 共享 Tile register/buffer（1 MB、4 banks、2 KB window） |
| Output Buffer | 全局结果驻留、前递、写回仲裁和 reduction reuse 结构 |
| Global Src Buffer | TileReg 读返回后的 2 KB entry 缓冲（~6 entries） |
| BStart | Block descriptor 起始指令（opcode、target PE、dtype/packing、attrs） |
| B.IOT | Tile 输入输出绑定指令（3 src + 1 dst） |
| TLOAD | TMA load 类 Tile 搬运指令 |
| GPR | BCC 标量通用寄存器文件 |
| SFU | Special Function Unit（EXP、DIV，256 B/cycle） |
| Resolve | PE 到 BCC 的 Block 完成反馈（允许乱序） |
| RR | Round-Robin — TileReg 读请求仲裁策略 |
| L0A/L0B | Cube 内部接收 TileReg 源数据的 buffer |
| Dtype/pack | 数据类型与 packing 格式（由 BStart 携带） |
| Mask Tile | TileOp 的可选 mask 输入 |
| TMAX/colmax | 示例 reduction/compare TileOp，展示 4-cycle 依赖延迟 |

---

## 附录 A：版本演进对比

| 方面 | v0.4-draft (AS_draft) | v0.5-public (AS_EN) | v0.5-enhanced（本文） |
|---|---|---|---|
| Pipeline 阶段 | 隐含 | V0–V8 标注 | V0–V8 含详细时序和依赖 |
| uOp 字段 | 草案（含 TBD 位宽） | 列出 | 结构化，含按 opclass 的细节 |
| 依赖模型 | 引用 | §3.4 明确规则 | 扩展了 flag/side-effect 说明 |
| Output Buffer 协议 | lock/writeback 描述 | 描述 | §6.2 lock/unlock 状态机 |
| 时序模型 | ~2cy / ~2-3cy / 4cy | 列出数字 | 集成到参数表 + 关键路径分析 |
| 反压 | 列出 | 列出 | 完整表格含上下游影响 |
| 物理实现 | 建议 | 无 | §15 含 floorplan 和风险分析 |
| 调度模型 | 隐含 | 无 | §13 两级调度 + 状态机 |
| 性能瓶颈 | 隐含 | 无 | §16 明确瓶颈分析 |

---

## 附录 B：实现参考参数（Python 格式）

这些参数应集中到共享参数文件（如 `janus4k_params.py`），确保 RTL、仿真模型和文档使用一致的常量：

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
