# Janus4K AI Core 架构规格书

> 版本: v0.5-public
> 日期: 2026-04-26
> 状态: 对外呈现版
> 关联工作稿: `Janus4k_AS_draft.md`

---

## 1. 概述

Janus4K 是一个以 `TileReg` 为中心的 tile-centric AI Core 子系统。系统以 `Block -> TileOp -> uOp` 为执行组织层次，通过 `BCC` 完成 Block 级依赖解析、标量执行、任务派发和顺序退休，通过 `Vector Core`、`Cube`、`TMA` 完成向量计算、矩阵计算和 Tile/Memory 搬运。

核心设计原则如下：

1. `TileReg` 是 AI Core 内部共享数据 buffer，总容量为 `1 MB`。
2. Vector 侧逻辑 Tile 固定为 `4 KB`；Cube/TMA 可使用 `4 KB` 整数倍的大 Tile。
3. `512 B` 是 Vector 计算 slice、TileReg bank slice 和 `Output Buffer` entry 的统一局部粒度。
4. `2 KB` 是 TileReg 对外一次满带宽读/写窗口，由 4 个逻辑 bank 各提供 `512 B` 组成。
5. `Output Buffer` 是全局唯一的结果驻留、前递、写回仲裁和 reduction reuse 结构。
6. Block 完成反馈允许乱序返回，BCC 按 Block 顺序退休。

### 1.1 架构边界

本规格定义 Janus4K AI Core 内部的 Block 控制、Tile 数据组织、Vector/Cube/TMA 协同、TileReg 读写仲裁、Output Buffer 前递与写回语义。软件栈、编译器 IR、DDR 控制器内部协议、NoC 协议和具体 SRAM 宏实现不属于本文定义范围，但这些外部模块必须遵守本文给出的 Block descriptor、Tile 粒度、接口握手和完成语义。

### 1.2 可见行为

Janus4K 对上层软件和验证环境暴露以下稳定行为：

| 行为 | 架构定义 |
| --- | --- |
| Block 派发 | BCC 在依赖满足后将 Block 派发到且只派发到一个目标 PE |
| 数据交换 | Vector、Cube、TMA 均通过 TileReg 交换 Tile 数据 |
| Vector Tile | 每个 Vector Tile 固定 `4 KB` |
| 读写窗口 | TileReg 满带宽读写窗口固定 `2 KB` |
| 结果前递 | Output Buffer 中 forwardable 结果可在写回 TileReg 前被 consumer 使用 |
| 完成反馈 | PE 返回 `resolve + block_id`，BCC 顺序退休 |

---

## 2. 顶层架构

### 2.1 顶层模块

Janus4K 顶层由以下模块组成：

| 模块 | 职责 |
| --- | --- |
| `BCC` | Block Control Core，负责 Block descriptor 解码、Tile 依赖解析、标量 pipe 执行、任务派发、resolve 记录和顺序退休 |
| `Vector Core` | 负责 Vector TileOp/uOp 执行，主计算粒度为 `512 B/cycle` |
| `Cube` | 矩阵计算单元，使用 L0A/L0B buffer 接收 TileReg 数据，并通过脉动阵列执行矩阵类计算 |
| `TMA` | Tile Memory Access，负责 TileReg 与 DDR/Memory 之间的双向搬运 |
| `TileReg` | AI Core 共享数据 buffer，总容量 `1 MB`，由 4 个逻辑 bank 组成 |
| `Memory` | 外部或上级 memory 系统 |

### 2.2 顶层框图

```text
                         ┌──────────────────────────────┐
                         │             BCC              │
                         │ Block descriptor decode      │
                         │ Tile dependency scheduling   │
                         │ GPR + scalar pipes           │
                         │ resolve + in-order retire    │
                         └───────┬──────────┬───────────┘
                                 │          │
            vector block         │          │ block
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

`TileReg` 位于 Vector/Cube/TMA 的数据交汇点。Vector Core、Cube 和 TMA 通过读写仲裁共享 TileReg 端口，数据在 TileReg、Global Src Buffer、Output Buffer 以及各执行单元之间流动。

### 2.4 控制路径与数据路径

Janus4K 将控制路径和数据路径分离：

| 路径 | 起点 | 终点 | 内容 |
| --- | --- | --- | --- |
| Block control | BCC | Vector/Cube/TMA | `block_id`、opcode、target PE、Tile 描述、dtype/packing、memory ordering |
| Resolve control | Vector/Cube/TMA | BCC | `resolve_valid`、`resolve_block_id`、完成状态 |
| Tile read | Vector/Cube/TMA | TileReg | `2 KB` 读窗口请求 |
| Tile return | TileReg | Global Src Buffer / Cube / TMA | `2 KB` 读返回数据和 tag |
| Result writeback | Output Buffer / Cube / TMA | TileReg | `512 B` 或聚合后的写回数据 |
| Memory transfer | TMA | Memory | `256 B` beat load/store |

控制路径必须保持 Block 身份。数据路径必须保持 Tile ID、slice/window ID、tag 和 mask/dtype 语义，使前递、写回、resolve 和退休能够正确关联。

### 2.5 单目标 PE 派发

每个 Block 在 `BStart` 中携带唯一 `target PE`。BCC 不复制同一个 Block 到多个 PE 执行。需要多个 PE 协同完成的工作由多个 Block 表达，每个 Block 独立建立 Tile 输入输出依赖，并独立返回 `resolve + block_id`。

---

## 3. 数据模型

### 3.1 Tile

| 项目 | 定义 |
| --- | --- |
| Vector Tile | 固定 `4 KB` |
| Cube/TMA Tile | `4 KB` 的整数倍 |
| Tile slice | `512 B` |
| TileReg 访问窗口 | `2 KB` |
| Tile 地址组织 | 单个 `4 KB` Tile 在 TileReg 内地址连续 |

一个 `4 KB` Vector Tile 由 8 个 `512 B` slice 组成。TileReg 以 512B slice 编号映射到 4 个逻辑 bank，使任意 PE 能够无 bank conflict 地读出同一 Tile 的一个 `2 KB` 连续窗口。

Tile 内 slice 和 window 组织固定如下：

| Slice | Byte range | 2KB window | 用途 |
| --- | --- | --- | --- |
| slice0 | `[0, 512)` | window0 | Vector 第 0 个 512B 计算片段 |
| slice1 | `[512, 1024)` | window0 | Vector 第 1 个 512B 计算片段 |
| slice2 | `[1024, 1536)` | window0 | Vector 第 2 个 512B 计算片段 |
| slice3 | `[1536, 2048)` | window0 | Vector 第 3 个 512B 计算片段 |
| slice4 | `[2048, 2560)` | window1 | Vector 第 4 个 512B 计算片段 |
| slice5 | `[2560, 3072)` | window1 | Vector 第 5 个 512B 计算片段 |
| slice6 | `[3072, 3584)` | window1 | Vector 第 6 个 512B 计算片段 |
| slice7 | `[3584, 4096)` | window1 | Vector 第 7 个 512B 计算片段 |

完整 Vector TileOp 覆盖 window0 和 window1。部分 Tile 操作仍使用 `512 B` slice 作为最小执行粒度，并使用 `2 KB` window 作为 TileReg 满带宽读写粒度。

### 3.2 TileReg

TileReg 是 1MB 级共享数据 buffer：

| 属性 | 值 |
| --- | --- |
| 总容量 | `1 MB` |
| 逻辑 bank 数 | `4` |
| 每 bank 读带宽 | `512 B/cycle` |
| 每 bank 写带宽 | `512 B/cycle` |
| bank 端口 | dual-port |
| 满带宽读窗口 | `2 KB/cycle` |
| 满带宽写窗口 | `2 KB/cycle` |

读写冲突通过仲裁解决。TileReg 的逻辑 bank 是架构级 bank；物理 SRAM 分片、bank 复制或端口实现由 RTL/后端实现选择，但必须保持上述可见带宽和无冲突窗口语义。

TileReg 请求携带以下逻辑身份：

| 字段 | 语义 |
| --- | --- |
| `tile_id` | TileReg 中的 Tile 编号 |
| `window_id` | `2 KB` window 编号，Vector Tile 内为 0 或 1 |
| `slice_base` | `512 B` slice 起点 |
| `bytes` | 请求字节数，满带宽窗口为 `2048` |
| `client` | Vector compute、Cube 或 TMA |
| `tag` | 返回数据、wakeup、写回和调试匹配用身份 |

TileReg 保证同一 `tile_id + window_id` 的 `2 KB` 读可以在一次 grant 后以无 bank conflict 的方式返回。写回可以按 `512 B` entry 写入，也可以在实现中聚合为更宽写窗口；架构可见结果必须保持 slice 顺序和 byte mask 正确。

### 3.3 Block Descriptor

Block descriptor 由 BCC 解码。每个 Block descriptor 以 `BStart` 开始，并由一条或多条 BCC 指令描述输入、输出和搬运属性。

| 指令 | 字段 | 语义 |
| --- | --- | --- |
| `BStart` | opcode, target PE, dtype/packing, attrs | 标记 Block 起始、操作类别、目标 PE 和数据格式 |
| `B.IOT` | 3 src Tile + 1 dst Tile | 绑定 Tile 输入输出 |
| `TLOAD` | memory addr, dst Tile, attrs | TMA load 类 Tile 搬运指令 |

约束：

1. 一个 Block 只能派发给一个目标 PE。
2. 一个 Block 的 dst Tile 最多为 4 个。
3. 一个 Block 的 src Tile 描述字段最多为 2 组。
4. 单条 `B.IOT` 描述 `3 src + 1 dst`；多条 `B.IOT` 组合表达多输入/多输出 Block。
5. `BStart` 携带 dtype/packing，Vector TileOp/uOp 使用该信息解释 element width 和数据布局。

`BStart` 的逻辑字段包括：

| 字段 | 语义 |
| --- | --- |
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

Block 之间通过 Tile 输入输出建立依赖。普通计算 Block 不建立 flag 依赖，也不建立普通 memory side-effect 依赖。TMA 的 `TLOAD`/store 类操作执行内存保序。

`GPR` 用于不同 Block 之间的标量值交互。标量值依赖由 BCC scalar pipe 和 GPR 维护。

---

## 4. BCC

### 4.1 职责

`BCC` 是 Janus4K 的 Block 级控制核心，负责：

1. 解码 `BStart/B.IOT/TLOAD`。
2. 建立 Tile 输入输出依赖。
3. 执行 BCC scalar pipe 指令。
4. 将 ready Block 派发到 Vector Core、Cube 或 TMA。
5. 接收 PE 返回的 `resolve + block_id`。
6. 支持 resolve 乱序返回和 Block 顺序退休。

### 4.2 Scalar Pipe

BCC 内部包含 GPR 和三类 scalar pipe：

| 结构 | 功能 |
| --- | --- |
| `GPR` | 通用标量寄存器文件，用于 Block 间标量值交互 |
| `AB pipe` | ALU + branch，支持 2-pick |
| `AM pipe` | ALU + multicycle，支持标量乘法 |
| `LSU` | 标量 load/store |

### 4.3 Resolve 与退休

PE 完成 Block 后向 BCC 返回 `resolve + block_id`。resolve 可以乱序返回，BCC 在 retire 结构中记录完成状态，并按 Block 顺序退休。退休时释放 Tile 依赖、GPR 依赖和后续 Block 可见状态。TMA Block 在满足内存保序后才能退休。

---

## 5. Vector Core

### 5.1 结构

Vector Core 是 Janus4K 中最主要的可编程向量执行单元。它接收 BCC 派发的 Vector Block，将 Block 内 TileOp 拆成 uOp，并通过固定执行资源完成计算。

| 结构 | 职责 |
| --- | --- |
| `Vector Block / TileOp Window` | 接收 Vector Block，保存 TileOp 状态，维护 Tile 级依赖和 wakeup |
| `TileOp Decoder` | 根据 opcode、dtype/packing、element width、mask 和 reduction 属性生成 uOp 序列 |
| `Operand Collector` | 对源 Tile 先查 Output Buffer，再为 miss 源生成 TileReg read request |
| `Read Port Arbitration` | 在 Vector compute、Cube、TMA 之间对 TileReg 读请求执行 RR 仲裁 |
| `Global Src Buffer` | 保存 TileReg 返回的 `2 KB` 源数据窗口 |
| `uOp Issue Queue` | 按执行类别维护 ready uOp，并向固定执行单元发射 |
| `FMLA/FCVT Pipe` | 执行 FMLA 和 FCVT/CVT 类操作 |
| `IALU/PERM/MAC Pipe` | 执行 IALU、PERM、MAC、compare 和 reduction 类操作 |
| `SFU` | 执行 EXP/DIV 等特殊函数，吞吐为 `256 B/cycle` |
| `Output Buffer` | 保存执行结果，支持 forwarding、2KB lock、writeback arbitration 和 reduction reuse |

### 5.2 Vector Pipeline 总览

Vector Core pipeline 分为前端、取数、发射、执行、结果和完成六段：

| 阶段 | 名称 | 主要结构 | 输入 | 输出 |
| --- | --- | --- | --- | --- |
| V0 | Block accept | Vector Block | BCC Vector Block | TileOp window entry |
| V1 | TileOp decode | TileOp Decoder | opcode、B.IOT、dtype/packing、mask | uOp plan |
| V2 | Source collect | Operand Collector | src Tile desc、Output Buffer tag | forward hit 或 read request |
| V3 | Read arbitration | Read Port Arbitration | Vector/Cube/TMA read request | 2KB read grant |
| V4 | TileReg read return | TileReg + Global Src Buffer | 2KB read window | source window ready |
| V5 | uOp issue | uOp Issue Queue | ready uOp + source window | pipe/SFU issue |
| V6 | Execute | FMLA/FCVT、IALU/PERM/MAC、SFU | 512B/256B operand slice | result entry |
| V7 | Result buffer | Output Buffer | result entry | forward hit / writeback request |
| V8 | Completion | Vector Block | TileOp done bitmap | `resolve + block_id` |

Pipeline 支持多个 TileOp 同时驻留。不同 TileOp 可以处在不同阶段；同一个 TileOp 的两个 `2 KB` window 可以流水化处理。V3 读仲裁、V5 uOp issue 和 V7 Output Buffer 写入均可独立产生反压。

### 5.3 TileOp 到 uOp 的拆分

每个 Vector TileOp 的源输入是 `4 KB srcTile`。标准逐元素三源操作按两个 `2 KB` window 拆分，每个 window 再拆成 4 个 `512 B` uOp：

```text
TileOp(4KB)
  window0: slice0, slice1, slice2, slice3
  window1: slice4, slice5, slice6, slice7
```

三源 TileOp 的每个 window 固定执行三次源读：

```text
windowN:
  read srcA[windowN]  -> 2KB
  read srcB[windowN]  -> 2KB
  read srcC[windowN]  -> 2KB
  issue 4 x 512B uOp
```

uOp 描述包含以下逻辑字段：

| 字段 | 语义 |
| --- | --- |
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

逐元素 FMLA/FCVT/IALU/PERM/MAC 操作以 `512 B` 为执行 slice。SFU 操作仍以 `2 KB` window 取数，但执行侧以 `256 B/cycle` 消费。Reduction 类操作按 reduction 方向、element width 和 reduction tree 生成 uOp，允许中间结果驻留在 Output Buffer。

### 5.4 取数 pipeline

Operand Collector 对每个源先执行 Output Buffer lookup，再决定是否消耗 TileReg 读口：

```text
src desc
  ├─ Output Buffer hit  -> lock 2KB window -> source ready
  └─ Output Buffer miss -> enqueue TileReg read request -> wait grant
```

取数规则如下：

| 规则 | 定义 |
| --- | --- |
| Lookup 优先级 | Output Buffer lookup 先于 TileReg read request |
| Hit 行为 | hit 源不消耗 TileReg 读口，对应 2KB Output Buffer window 被锁定 |
| Miss 行为 | miss 源进入 Read Port Arbitration |
| Grant 粒度 | 每次 grant 读取一个 `2 KB` window |
| 三源顺序 | Vector 三源 window 固定按 srcA、srcB、srcC 发起读请求 |
| 返回位置 | TileReg 返回数据进入 Global Src Buffer |
| Wakeup | 三个源 window 都 ready 后唤醒对应 uOp group |

Global Src Buffer 的 entry 粒度为 `2 KB`。一个三源 window 最多需要三个 Global Src Buffer entry 保存 srcA、srcB、srcC 的返回数据；当某个源来自 Output Buffer forwarding 时，该源不占用 Global Src Buffer entry。

### 5.5 uOp Issue Queue

uOp Issue Queue 分为三类固定队列：

| Queue | 执行单元 | 覆盖操作 |
| --- | --- | --- |
| `FMLA/FCVT Queue` | `FMLA/FCVT Pipe` | FMLA、FCVT/CVT |
| `IALU/PERM/MAC Queue` | `IALU/PERM/MAC Pipe` | IALU、PERM、MAC、compare、reduction、TMAX/colmax |
| `SFU Queue` | `SFU` | EXP、DIV 等特殊函数 |

uOp 进入 ready 状态需要满足以下条件：

1. 所有源 window 已 ready。
2. mask Tile 已 ready 或该 uOp 不使用 mask。
3. 目标执行单元可接收。
4. Output Buffer 对该结果有可分配 entry 或可形成 backpressure。
5. reduction uOp 的前序 reduction_ctx 已 ready。

三类执行资源不共享执行单元。`opclass` 到执行单元的映射固定，调度器不将 SFU uOp 发射到普通 Vector pipe，也不将 FMLA/FCVT uOp 发射到 IALU/PERM/MAC Pipe。

### 5.6 执行 pipeline

Vector 执行侧包含两条 512B 主执行路径和一条 256B SFU 路径：

| 执行路径 | 输入粒度 | 输出粒度 | 典型操作 | 结果去向 |
| --- | --- | --- | --- | --- |
| `FMLA/FCVT Pipe` | `512 B/cycle` | `512 B entry` | FMLA、FCVT/CVT | Output Buffer |
| `IALU/PERM/MAC Pipe` | `512 B/cycle` | `512 B entry` | IALU、PERM、MAC、TMAX/colmax | Output Buffer |
| `SFU` | `256 B/cycle` | 聚合为 `512 B entry` | EXP、DIV | Output Buffer |

普通 512B pipe 的结果以 `512 B` entry 写入 Output Buffer。SFU 每拍消费 `256 B`，两个连续 256B 结果组合成一个 `512 B` Output Buffer entry。执行单元输出必须携带 `block_id/tileop_id/uop_id/dst_desc/slice_id/tag`，用于 Output Buffer lookup、writeback 和完成统计。

### 5.7 典型三源 TileOp 时序

标准三源 4KB TileOp 的读数和计算顺序如下：

| 步骤 | 动作 | 数据量 | 说明 |
| --- | --- | --- | --- |
| 1 | window0 srcA read | 2KB | 经过 RR 仲裁 |
| 2 | window0 srcB read | 2KB | 经过 RR 仲裁 |
| 3 | window0 srcC read | 2KB | 经过 RR 仲裁 |
| 4 | window0 compute | 4 x 512B | 发射 4 个 512B uOp |
| 5 | window1 srcA read | 2KB | 经过 RR 仲裁 |
| 6 | window1 srcB read | 2KB | 经过 RR 仲裁 |
| 7 | window1 srcC read | 2KB | 经过 RR 仲裁 |
| 8 | window1 compute | 4 x 512B | 发射 4 个 512B uOp |
| 9 | result residency/writeback | 8 x 512B | 结果进入 Output Buffer，按依赖和写口状态写回 |

当某个源在 Output Buffer 命中时，对应 read 步骤被 forwarding 替代，不发起 TileReg read。三个源全部 ready 后，该 window 的 4 个 uOp 可进入对应 issue queue。

### 5.8 Wakeup 与依赖

Vector Core 使用两类 wakeup：

| Wakeup | 来源 | 作用 |
| --- | --- | --- |
| External wakeup | TileReg read grant/return | 唤醒等待 TileReg 数据的 uOp group |
| Internal wakeup | Output Buffer result ready | 唤醒 producer-consumer 或 reduction 链上的后继 uOp |

Output Buffer hit 的 consumer 在锁定对应 2KB window 后进入 source-ready 状态。Reduction 链上的后继 uOp 可以直接消费 Output Buffer 中的中间结果，不需要等待中间结果写回 TileReg。

### 5.9 Vector 反压

Vector pipeline 每级都保持 valid/ready 语义。任一级无法接收时，上游保持 payload 稳定，不丢失 tag 和数据。

| 阻塞点 | 触发条件 | 反压方向 |
| --- | --- | --- |
| TileOp Window full | 无可用 TileOp entry | BCC 停止派发新的 Vector Block |
| Operand Collector busy | 源 lookup 或 read request 队列满 | TileOp 暂停生成新 uOp |
| Read Arbitration pending | RR 未 grant | uOp group 保持等待源状态 |
| Global Src Buffer full | TileReg 返回无 entry 可写 | TileReg read 返回路径反压，进一步抑制新 read grant |
| uOp Queue full | 对应 opclass queue 无 entry | Operand Collector 暂停 dispatch ready uOp |
| Execute busy | 目标执行单元不可接收 | uOp Queue 保持 ready uOp |
| Output Buffer full/locked | 无可写 result entry 或目标 window 锁定 | Execute 输出反压，进一步抑制 issue |
| TileReg write busy | 写口仲裁未 grant | Output Buffer 保持未写回 entry |

---

## 6. Output Buffer

`Output Buffer` 是全局唯一结构。它位于执行结果和 TileReg 写回之间，提供以下功能：

1. 以 `512 B` entry 粒度保存执行结果。
2. 支持 producer-consumer forwarding。
3. 支持跨 Pipe forwarding。
4. 支持 TileReg write-port arbitration。
5. 支持 reduction 中间结果复用。
6. 支持未立即写回结果的驻留和后续写回。

Output Buffer entry 的逻辑字段如下：

| 字段 | 语义 |
| --- | --- |
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

当 TileOp 查询到源数据位于 Output Buffer 中时，对应 `2 KB` 数据窗口被锁定。锁定期间该窗口不能被覆盖或释放。consumer 使用完成后解锁。未被锁定的 entry 每拍尝试发出 TileReg 写回请求，写回请求经过 TileReg 写口仲裁。

Reduction 中间结果可长期驻留在 Output Buffer 中，不要求每一步都写回 TileReg。Reduction 最终结果在满足写回条件后写回目标 Tile。

---

## 7. TileReg 仲裁与缓冲

### 7.1 Read Port Arbitration

TileReg 读仲裁接收三类 client 请求：

1. `Vector compute`
2. `Cube`
3. `TMA`

仲裁策略为 Round-Robin。每次 grant 对应一个 `2 KB` TileReg 读窗口。未获 grant 的请求保持 pending，后续 RR 轮次继续参与仲裁。

### 7.2 Global Src Buffer

Global Src Buffer 位于 TileReg 返回路径上，用于缓冲 `2 KB` 读窗口。其 entry 粒度为 `2 KB`，深度为 6 entries。进入执行单元前，`2 KB` 数据窗口可进一步拆成 4 个 `512 B` slice。

### 7.3 写口仲裁

TileReg 写冲突通过写口仲裁解决。Output Buffer、Cube 和 TMA 都可能产生写入请求。写口仲裁必须保持数据、tag、tile_id、slice_id 的一致性，并保证 forwarding-visible 结果不会因写口阻塞而丢失。

---

## 8. Cube

Cube 是 Janus4K 的矩阵计算单元。Cube 输入输出以 Tile 表示，可使用由多个 `4 KB` Tile 组成的大 Tile。Cube 从 TileReg 读入数据后写入 `L0A/L0B` buffer，内部由脉动阵列消费 L0A/L0B 数据。Cube 内部消费粒度不受 Vector `512 B` slice 限制。Cube 完成后向 BCC 返回 `resolve + block_id`。

---

## 9. TMA

TMA 负责 TileReg 与 DDR/Memory 之间的双向数据搬运。

| 属性 | 定义 |
| --- | --- |
| 方向 | load/store 双向 |
| TileReg 单次请求上限 | `2 KB` |
| Memory 侧 beat | `256 B` |
| 读写口 | 分离 |
| 队列 | load/store 共享调度队列 |
| 保序 | TLOAD/store 执行内存保序 |

TMA load 将 memory 数据搬入 TileReg。TMA store 将 TileReg 数据写回 memory。TMA 完成后向 BCC 返回 `resolve + block_id`。

---

## 10. Vector 执行流程

1. BCC 解码 Block descriptor，并确认目标 PE 为 Vector Core。
2. BCC 根据 Tile 输入输出关系判断依赖是否满足。
3. BCC 将 ready Vector Block 派发给 Vector Block / TileOp Window。
4. TileOp Decoder 根据 `BStart/B.IOT`、dtype/packing、element width、mask 和 opclass 生成 uOp plan。
5. Operand Collector 对每个源执行 Output Buffer lookup。
6. Output Buffer hit 的源锁定对应 2KB window，并通过 forwarding 满足。
7. Output Buffer miss 的源向 Read Port Arbitration 发起 TileReg read request。
8. Read Port Arbitration 按 RR grant `2 KB` read window。
9. TileReg 返回 `2 KB` 数据进入 Global Src Buffer。
10. 对应 window 的所有源 ready 后，4 个 512B uOp 进入固定 uOp queue。
11. uOp Issue Queue 根据 opclass 发射到 FMLA/FCVT、IALU/PERM/MAC 或 SFU。
12. 执行单元输出 `512 B` Output Buffer entry；SFU 输出以两个 256B beat 聚合为 512B entry。
13. Output Buffer 根据 tag 唤醒后继 consumer，或向 TileReg 写回未锁定 entry。
14. TileOp 所有 window 和 slice 完成后更新 TileOp done 状态。
15. Block 内所有 TileOp 完成后，Vector Core 向 BCC 返回 `resolve + block_id`。
16. BCC 按顺序退休已 resolve 的 Block。

---

## 11. 接口定义

### 11.1 BCC 到 PE

| 信号 | 方向 | 含义 |
| --- | --- | --- |
| `block_valid` | BCC -> PE | Block 有效 |
| `block_ready` | PE -> BCC | 目标 PE 可接收 |
| `block_id` | BCC -> PE | Block 标识 |
| `block_target` | BCC -> PE | Vector/Cube/TMA |
| `block_opcode` | BCC -> PE | Block 操作类别 |
| `block_bstart` | BCC -> PE | BStart 解码结果 |
| `block_iot_vec` | BCC -> PE | B.IOT 展开的 Tile 输入输出绑定 |
| `block_attr` | BCC -> PE | dtype/packing、element width、reduction、memory ordering 等属性 |

### 11.2 PE 到 BCC

| 信号 | 方向 | 含义 |
| --- | --- | --- |
| `resolve_valid` | PE -> BCC | Block 完成反馈有效 |
| `resolve_ready` | BCC -> PE | BCC 可接收 resolve |
| `resolve_block_id` | PE -> BCC | 完成的 Block ID |
| `resolve_status` | PE -> BCC | 完成状态 |

### 11.3 Operand Collector 到 Read Port Arbitration

| 信号 | 方向 | 含义 |
| --- | --- | --- |
| `rd_req_valid` | requester -> arb | TileReg 读请求有效 |
| `rd_req_ready` | arb -> requester | 仲裁器可接收 |
| `rd_req_client` | requester -> arb | Vector compute/Cube/TMA |
| `rd_req_tile_id` | requester -> arb | Tile ID |
| `rd_req_slice_base` | requester -> arb | 512B slice 或 2KB window 起点 |
| `rd_req_bytes` | requester -> arb | 请求字节数，标准为 `2 KB` |
| `rd_req_tag` | requester -> arb | 返回数据匹配 tag |

### 11.4 Execute/SFU 到 Output Buffer

| 信号 | 方向 | 含义 |
| --- | --- | --- |
| `ob_valid` | exec -> OB | 执行结果有效 |
| `ob_ready` | OB -> exec | Output Buffer 可接收 |
| `ob_tag` | exec -> OB | forwarding/writeback 匹配 tag |
| `ob_data` | exec -> OB | `512 B` 结果 entry |
| `ob_forwardable` | exec -> OB | 结果允许 forwarding |
| `ob_writeback_req` | exec -> OB | 结果需要写回 TileReg |
| `ob_reduction_ctx` | exec -> OB | reduction 上下文 |

### 11.5 TMA

| 信号 | 方向 | 含义 |
| --- | --- | --- |
| `tma_cmd_valid` | BCC -> TMA | TMA 命令有效 |
| `tma_cmd_ready` | TMA -> BCC | TMA 可接收命令 |
| `tma_cmd_is_store` | BCC -> TMA | load/store 方向 |
| `tma_tile_id` | BCC -> TMA | TileReg Tile ID |
| `tma_tile_bytes` | BCC -> TMA | TileReg 单次请求字节数，最大 `2 KB` |
| `tma_mem_addr` | BCC -> TMA | memory 地址 |
| `tma_mem_beat_bytes` | BCC -> TMA | memory beat，固定 `256 B` |
| `tma_order_tag` | BCC -> TMA | 内存保序 tag |

---

## 12. 参数表

| 参数 | 值 | 说明 |
| --- | --- | --- |
| `TILEREG_BYTES` | `1 MB` | TileReg 总容量 |
| `TILE_BYTES` | `4096` | Vector Tile 容量 |
| `TILE_SLICE_BYTES` | `512` | Vector slice / Output Buffer entry |
| `TILEREG_BANKS` | `4` | TileReg 逻辑 bank 数 |
| `TILEREG_BANK_BYTES_PER_CYCLE` | `512` | 每 bank 每拍读/写带宽 |
| `TILEREG_RW_BYTES_PER_CYCLE` | `2048` | TileReg 满带宽读/写窗口 |
| `GLOBAL_SRCBUF_ENTRY_BYTES` | `2048` | Global Src Buffer entry 粒度 |
| `GLOBAL_SRCBUF_ENTRIES` | `6` | Global Src Buffer 深度 |
| `OUTPUT_BUFFER_ENTRY_BYTES` | `512` | Output Buffer entry 粒度 |
| `MAX_DST_TILES_PER_BLOCK` | `4` | Block dst Tile 上限 |
| `MAX_SRC_TILE_DESC_GROUPS_PER_BLOCK` | `2` | Block src Tile 描述字段组上限 |
| `B_IOT_SRC_TILES` | `3` | 单条 B.IOT src Tile 数 |
| `B_IOT_DST_TILES` | `1` | 单条 B.IOT dst Tile 数 |
| `NUM_EXEC_CLASSES` | `3` | FMLA/FCVT、IALU/PERM/MAC、SFU |
| `SFU_BYTES_PER_CYCLE` | `256` | SFU EXP/DIV 带宽 |
| `TMA_REQ_BYTES` | `2048` | TMA TileReg 单次请求上限 |
| `TMA_MEM_BEAT_BYTES` | `256` | TMA memory 侧 beat |
| `READ_ARB_POLICY` | `RR` | TileReg 读仲裁策略 |

---

## 13. 验证覆盖

验证应覆盖以下架构行为：

1. BCC `BStart/B.IOT/TLOAD` 解码和单目标 PE 派发。
2. BCC scalar pipe、GPR 跨 Block 标量值交互、AB/AM/LSU 行为。
3. Vector `4 KB` 三源 TileOp 的两轮 `2 KB` 读窗口和 8 个 `512 B` slice 执行。
4. Output Buffer hit 不消耗 TileReg 读口。
5. Output Buffer `2 KB` lock/unlock 和未锁定 entry 写回仲裁。
6. Reduction 中间结果驻留 Output Buffer。
7. Read Port Arbitration 对 Vector/Cube/TMA 请求执行 RR。
8. TileReg 逻辑 bank 下 `2 KB` 连续读窗口无 bank conflict。
9. SFU EXP/DIV `256 B/cycle` 和 mask Tile 输入。
10. TMA `2 KB` TileReg 请求、`256 B` memory beat、双向 load/store 和内存保序。
11. Cube L0A/L0B buffering 和大 Tile 输入输出。
12. PE resolve 乱序返回与 BCC 顺序退休。

---

## 14. 术语表

| 术语 | 含义 |
| --- | --- |
| BCC | Block Control Core |
| Block | BCC 看到的任务粒度 |
| TileOp | Vector Core 的 tile 级操作 |
| uOp | 可发射到执行单元的细粒度操作 |
| TileReg | Janus4K 共享 Tile register/buffer |
| Output Buffer | 全局结果驻留、前递和写回仲裁结构 |
| Global Src Buffer | TileReg 读返回后的 `2 KB` entry 缓冲 |
| BStart | Block descriptor 起始指令 |
| B.IOT | Tile 输入输出绑定指令 |
| TLOAD | TMA load 类 Tile 搬运指令 |
| GPR | BCC 标量通用寄存器文件 |
| SFU | Special Function Unit |
| Resolve | PE 到 BCC 的 Block 完成反馈 |
