# FlashAttention 全系梳理：从 V1 到 V4 的算法演进、差异点与参考资料

## 0. 先说结论

如果只抓主线，FlashAttention 的演进可以概括成四句话：

1. **V1（2022）**解决的是“Attention 为什么明明 FLOPs 不算离谱，却还是很慢”的问题，答案是 **HBM 读写才是瓶颈**。  
2. **V2（2023）**解决的是“V1 已经避免了大矩阵落 HBM，但 GPU 还是没吃满”的问题，答案是 **并行划分和 warp 协作方式不够好**。  
3. **V3（2024）**解决的是“H100 这种新架构上的异步 Tensor Core、TMA、FP8 没有被用起来”的问题，答案是 **要把 attention 重新设计成异步流水线**。  
4. **V4（2026）**解决的是“Blackwell 上 Tensor Core 变得更快，但 shared memory / SFU 没有同比例变快”的问题，答案是 **进一步做算法与 kernel 的协同设计，把瓶颈从 matmul 转移到 softmax、shared memory、原子操作等环节再逐个击破**。  

换句话说，这条路线的主线不是“换了一个 attention 公式”，而是：  
**在保持 attention 语义不变的前提下，持续重排计算顺序、重写并行策略、重构 kernel 流水线，让 attention 越来越贴近硬件峰值。**

这里的“语义不变”主要指 FP16/BF16 路径仍然计算标准 scaled dot-product attention，不引入稀疏、低秩或局部窗口近似。V3/V4 的 FP8 路径会引入量化误差，所以更准确的说法是：**算法结构仍是标准 attention，但低精度实现不再是实数意义上的完全精确计算**。

## 1. 时间线与版本边界

先把几个版本的边界说清楚：

| 版本 | 正式名称 | 时间 | 主要目标硬件 | 一句话定位 |
| --- | --- | --- | --- | --- |
| V1 | FlashAttention | 2022-05-27 提交，NeurIPS 2022 | A100 / Ampere | 用 IO-aware tiling + online softmax，把 attention 从“存矩阵”改成“流式算” |
| V2 | FlashAttention-2 | 2023-07-17 提交，ICLR 2024 | A100 / Ampere | 重做 thread block 与 warp 的工作划分，让 attention 更接近 GEMM 效率 |
| V3 | FlashAttention-3 | 2024-07-11 提交 | H100 / Hopper | 用 TMA、WGMMA、异步流水线与 FP8，把 Hopper 特性吃满 |
| V4 | FlashAttention-4 | 2026-03-05 提交 | B200 / GB200 / Blackwell | 面向 Blackwell 的非对称硬件扩展，继续压缩 softmax、SMEM、原子操作瓶颈 |

有两点需要特别说明：

- **“V1”是回溯式叫法。** 原始论文标题就是 *FlashAttention*，并不叫 *FlashAttention-1*。[1]
- **FlashDecoding / FlashDecoding++ 不是 V3 或 V4。** 它们是面向 decoding 场景的旁支优化路线，不是 FlashAttention 主线版本升级。

## 2. FlashAttention 之前，标准 Attention 为什么慢

标准 scaled dot-product attention 的计算顺序通常写成：

```text
S = Q K^T / sqrt(d_h) + M
P = softmax_row(S)
O = P V
```

其中 `d_h` 是每个 head 的维度，`M` 是 mask / causal mask / padding mask。对第 `i` 行：

```text
P_ij = exp(S_ij - max_j(S_ij)) / sum_t exp(S_it - max_j(S_ij))
O_i  = sum_j P_ij V_j
```

数学上这很自然，但在 GPU 上有两个致命问题：

1. **中间矩阵 `S` 和 `P` 太大。**  
   `S` 的形状是 `[N, N]`，序列一长，显存开销就直接平方增长。

2. **真正拖慢速度的往往不是 matmul，而是 HBM 读写。**  
   `QK^T` 和 `PV` 是大矩阵乘法，Tensor Core 很擅长；但一旦把 `S`、`P` 写回 HBM，再读回来做 softmax 和第二个 matmul，整个过程就会被内存系统拖慢。[1]

FlashAttention 的核心思想一直没变：  
**不通过稀疏、低秩、局部窗口去近似 attention，而是重排标准 attention 的执行顺序，尽量不把大的中间矩阵写回 HBM。**

## 3. V1：FlashAttention（2022）

### 3.1 核心问题

V1 的出发点很明确：  
**attention 的真正瓶颈不是公式本身，而是 IO。**

原论文把这种思想叫做 **IO-awareness**。[1]  
在 A100 这类 GPU 上，片上 SRAM / shared memory 的带宽远高于 HBM，但容量很小。V1 的关键就是让 Q、K、V 以 tile 的方式流过片上存储，而不是先把完整 attention matrix 落到 HBM。

### 3.2 算法核心

V1 有四个关键点。

#### 3.2.1 Tiling：按块处理 Q、K、V

V1 把 Q、K、V 切成小块。  
计算时只把当前需要的 Q block 和 K/V block 搬进片上 SRAM，在片上完成：

1. 当前 Q block 与 K block 的乘法  
2. 当前 block 的 softmax 统计量更新  
3. 当前 block 对输出 O 的增量更新

这样做的效果是：  
**避免显式 materialize 整个 `S` 和 `P`。**

从复杂度看，naive 实现的额外显存主要被 `S` / `P` 占住，是 `O(N^2)`；FlashAttention 前向只需要保存输出和每行归一化统计量，额外显存可以降到 `O(N)` 级别。论文的 IO 分析还指出，在 SRAM 容量为 `M`、head dimension 为 `d_h` 的典型假设下，FlashAttention 的主要 HBM 访问量可以从标准实现的 `Theta(N^2 + N d_h)` 级别降到 `Theta(N^2 d_h^2 / M)` 级别。[1]

#### 3.2.2 Online Softmax：softmax 也按块增量做

softmax 看起来必须先看到一整行才能算，但 V1 利用了 **online softmax / safe softmax** 的增量形式。[1][2]

对一行 attention score，可以维护：

- 当前全局最大值 `m`
- 当前归一化分母 `l`

每来一个新 tile，就更新这两个统计量，并对当前输出做重缩放。  
因此 softmax 不再要求先把整行 score 写到 HBM 后再统一处理，而是可以跟 block matmul 一起流式完成。

把公式写开会更清楚。对某一行 `i`，假设当前已经处理过一部分 key/value，维护：

```text
m_i = 当前已处理 score 的最大值
l_i = sum exp(score - m_i)
a_i = sum exp(score - m_i) * V
```

当新的 K/V tile 到来，先算局部 score：

```text
S_ij = Q_i K_j^T / sqrt(d_h) + M_ij
```

然后用局部行最大值 `m_tile = max_j(S_ij)` 更新全局统计量：

```text
m_new = max(m_i, m_tile)
l_new = exp(m_i - m_new) * l_i
      + sum_j exp(S_ij - m_new)

a_new = exp(m_i - m_new) * a_i
      + sum_j exp(S_ij - m_new) * V_j

O_i = a_new / l_new
```

这组递推式的关键是 `exp(m_i - m_new)`。如果新 tile 里出现了更大的 score，旧 tile 的贡献必须按新的最大值重新缩放；如果没有出现更大值，这个因子就是 `1`。

#### 3.2.3 Kernel Fusion：把注意力链路尽量合并成一个 fused kernel

V1 的实际收益不只来自算法公式，还来自实现方式：  
把 `QK^T`、mask、softmax、dropout、`PV` 这些步骤尽量融合，减少中间张量反复落 HBM。[1]

#### 3.2.4 Backward 里用重计算换显存

前向阶段不保存完整 `P`。  
反向时，重新把对应的 Q/K/V tile 载入片上，利用前向保留的归一化信息重算局部 attention，再求梯度。[1]

前向只需要保存输出 `O` 和每行的 log-sum-exp：

```text
L_i = m_i + log(l_i)
```

反向处理某个 tile 时重算：

```text
S_ij = Q_i K_j^T / sqrt(d_h) + M_ij
P_ij = exp(S_ij - L_i)
```

然后按标准 softmax backward 公式累积梯度。设 `dO` 是输出梯度：

```text
D_i    = sum_r dO_ir * O_ir
dV_j  += sum_i P_ij * dO_i
dP_ij  = dO_i dot V_j
dS_ij  = P_ij * (dP_ij - D_i)
dQ_i  += sum_j dS_ij * K_j / sqrt(d_h)
dK_j  += sum_i dS_ij * Q_i / sqrt(d_h)
```

所以 V1 backward 的本质不是保存 attention matrix，而是保存足够小的归一化状态，并在反向阶段按 tile 重建局部 `P`。

这是一种经典的 tradeoff：

- 多做一些算术重计算
- 换来更少的 HBM 读写和更低的显存占用

### 3.3 V1 带来的变化

V1 的意义不在于“又一个更快的 kernel”，而在于它第一次把这件事讲清楚：

- **FP16/BF16 标准 attention 也可以做成线性额外显存复杂度**
- **长上下文的瓶颈不只是 FLOPs，更是内存层级**
- **attention 可以像高性能 GEMM 一样做硬件友好的分块和流水化**

论文里给出的代表性结果包括：[1]

- BERT-large（seq 512）端到端训练速度相对当时 MLPerf 1.1 记录提升 15%
- GPT-2（seq 1K）attention 部分约 3x 加速
- Long Range Arena（seq 1K-4K）约 2.4x 加速
- 在 Path-X（16K）和 Path-256（64K）上首次把 Transformer 拉到随机猜测以上

### 3.4 V1 的局限

V1 虽然解决了 IO 问题，但还没有把 GPU 并行度吃满。

主要问题有三类：

1. **thread block 并行粒度还不够理想**  
2. **warp 之间存在较多 shared memory 通信与同步**  
3. **整体效率仍明显低于高质量 GEMM kernel**

FlashAttention-2 论文回顾 V1 时给出的数字是：  
V1 在 A100 上大约只能达到 **25% 到 40% 的理论峰值 FLOPs/s**。[3]

## 4. V2：FlashAttention-2（2023）

如果说 V1 解决的是“别把大矩阵写回 HBM”，那 V2 解决的是：  
**不写回 HBM 之后，为什么 attention 还是不够快？**

V2 的答案是：  
**并行划分方式不对，warp 之间通信太多，non-matmul FLOPs 也偏多。**[3]

### 4.1 V2 的三大改进

#### 4.1.1 减少 non-matmul FLOPs

V2 明确提出一个非常重要的观点：  
在现代 GPU 上，**一个 non-matmul FLOP 往往比一个 matmul FLOP 贵得多**。[3]

论文以 A100 为例指出：

- FP16/BF16 matmul 的理论峰值约为 312 TFLOPs/s
- FP32 非矩阵乘运算峰值约为 19.5 TFLOPs/s

所以 V2 对 online softmax 的更新公式做了小改动：

- 前向里维护未缩放的输出累积量
- 反向只保存 `logsumexp`，不再同时保存 row max 与 sum(exp)

这类变化数学上不大，但在 kernel 层能显著减少标量运算和访存压力。

更具体地说，V1 里常见写法会在每个 K/V tile 之后形成当前的归一化输出 `O_i`。这会频繁做除法和重缩放。V2 更偏向维护未归一化累加器 `a_i`，把最后的归一化推迟到本行所有 K/V tile 都处理完：

```text
for each K/V tile:
    m_new = max(m_i, max_j(S_ij))
    alpha = exp(m_i - m_new)
    p_ij  = exp(S_ij - m_new)

    l_i = alpha * l_i + sum_j p_ij
    a_i = alpha * a_i + sum_j p_ij * V_j
    m_i = m_new

O_i = a_i / l_i
L_i = m_i + log(l_i)
```

这和 V1 在数学上等价，但把多次“规范化后的输出更新”改成“未规范化 numerator 累加 + 末尾一次除法”。在 GPU 上，这类 non-matmul 操作无法像 Tensor Core GEMM 一样便宜，因此减少它们会直接反映到 kernel 效率上。

#### 4.1.2 重新做 thread block 级并行

V1 主要按 batch 和 head 并行。  
当 batch 小、head 少、sequence 很长时，GPU 上可调度的 thread block 不够多，occupancy 会掉下来。

V2 的做法是：

- 前向把单个 head 的 **query sequence length** 拆成更多 Q block，让长序列也能产生足够多的 thread block
- 反向围绕 **key/value block** 与梯度归约重新安排并行度，使 `dK` / `dV` / `dQ` 的 partial sum 更适合 GPU 执行

这样即使单个 head 很长，也能拆成多个 thread block 去跑，提高 SM 占用率。[3]

#### 4.1.3 重新做 warp 内分工：从 split-K 转向 split-Q

这是 V2 最关键的工程变化之一。

V1 里常见的做法是让不同 warp 分担 K 方向的工作，然后在 shared memory 里做归约。  
问题是：

- warp 先各自算一部分
- 再把中间结果写 shared memory
- 然后同步
- 再读回来做合并

这会带来大量额外 shared memory 读写和同步。

V2 改成让不同 warp 分担 **Q 方向** 的工作，即每个 warp 负责不同输出切片，但共享 K/V。  
这样每个 warp 能独立地产生自己那部分输出，**不再需要跨 warp 的中间结果归约**。[3]

这一步本质上是在回答一个问题：  
**attention kernel 不只是“分块”，还要“分工对”。**

可以把 V1/V2 的 warp 分工差异简化成下面这张表：

| 维度 | V1 倾向的 sliced-K | V2 倾向的 sliced-Q |
| --- | --- | --- |
| warp 分到什么 | 不同 K/V 切片 | 不同 Q / 输出行切片 |
| Q 的使用方式 | Q 在 warp 间共享 | 每个 warp 处理自己的 Q 子块 |
| K/V 的使用方式 | 各 warp 处理不同 K/V 子块 | K/V 在 warp 间共享 |
| 是否需要跨 warp 合并 | 需要合并 partial output | 基本不需要合并 partial output |
| 主要收益 | 复用 Q | 减少 shared memory 读写和同步 |

### 4.2 V2 的效果

V2 相比 V1 的关键收益包括：[3]

- 相比 V1 约 **2x** 加速
- 在 A100 上达到 **50% 到 73%** 理论峰值 FLOPs/s
- GPT 风格模型训练时，单卡 A100 最多到 **225 TFLOPs/s**
- 更接近高质量 GEMM 的效率

这也是 FlashAttention 系列第一次真正把“attention kernel 接近 GEMM 效率”变成现实。

### 4.3 V2 的局限

V2 基本还是围绕 Ampere 时代写的。

它没有系统利用 Hopper 上的新硬件特性，例如：

- TMA（Tensor Memory Accelerator）
- WGMMA（warpgroup-level MMA）
- 更强的异步执行能力
- FP8 Tensor Core 路径

所以到了 H100 时代，V2 又遇到了新的天花板。  
FlashAttention-3 论文指出，V2 在 H100 上大约只能到 **35% 利用率**。[4]

## 5. V3：FlashAttention-3（2024）

V3 的本质不是“再调一遍 block size”，而是：  
**针对 Hopper 的硬件模型，重写 attention 的流水线。**

Hopper 带来了三个对 attention 很关键的新能力：

1. **TMA**：更高效地在 global memory 与 shared memory 之间搬 tile[5][6]  
2. **WGMMA**：异步 warpgroup 级矩阵乘，Tensor Core 吞吐更高[4][7]  
3. **FP8 Tensor Core**：低精度吞吐显著提高，但要处理精度问题[4]

### 5.1 V3 的三个核心改进

#### 5.1.1 Producer-Consumer Warp Specialization

V3 把 CTA 内的 warps 分成两类：[4]

- **Producer warps**：专门发起 TMA，把 K/V tile 搬到 shared memory
- **Consumer warps**：专门发起 WGMMA，做矩阵乘和后续计算

这就是典型的 **warp specialization**。  
它的意义是让数据搬运与矩阵计算可以并行重叠，而不是由同一批线程串行完成所有工作。

V3 还借助 Hopper 的 `setmaxnreg`，把更多寄存器给真正做 MMA 的 consumer warps，用更少寄存器给 producer warps。[4]

#### 5.1.2 GEMM 与 Softmax 的异步重叠

这是 V3 最精彩的部分。

在 attention 里，除了两个大 GEMM，softmax 里的 `exp` 也很慢。  
Tri Dao 在官方博文里给出的解释很直白：[8]

- H100 FP16 matmul 峰值约 989 TFLOPs/s
- 特殊函数（如 exponential）吞吐只有约 3.9 TFLOPs/s

因此即使 GEMM 很快，softmax 也会卡住整条流水线。

V3 的策略是：

- 让一个 warpgroup 在做 GEMM 时
- 另一个 warpgroup 去做 softmax
- 通过 ping-pong 调度和更细粒度流水线，把这两类操作尽量重叠起来

这背后的核心思想是：  
**既然 Tensor Core 和 MUFU/SFU 不是同一类资源，那就尽量同时用，而不是轮流用。**

#### 5.1.3 FP8：不仅更快，还要尽量稳

V3 是 FlashAttention 主线第一次系统引入 FP8 路径。[4]

但 FP8 不能只“开开 Tensor Core”就结束，因为 attention 对异常值非常敏感。  
V3 为了兼顾吞吐与精度，做了两件事：

1. **Block Quantization**  
   不是整张 Q/K/V 张量共用一个 scale，而是按 block 量化，减少局部尺度不一致带来的误差。

2. **Incoherent Processing**  
   在量化前对 Q/K 施加随机正交变换（实际实现可用 Hadamard + 随机符号），把 outlier“打散”，降低量化误差。[4][8]

这一步背后的公式也很直接。若 `R` 是正交矩阵，满足：

```text
R R^T = I
```

则：

```text
(Q R) (K R)^T = Q R R^T K^T = Q K^T
```

也就是说，在无限精度下这个变换不改变 attention score；它的目的不是改 attention 公式，而是在 FP8 量化前把某些维度上的 outlier 分散到更多维度里。配合 block scale，可以写成：

```text
Q_fp8 = quantize((Q R) / s_Q_block)
K_fp8 = quantize((K R) / s_K_block)
Q R   ~= s_Q_block * Q_fp8
K R   ~= s_K_block * K_fp8
```

误差主要来自 `quantize`，不是来自正交变换本身。

V3 论文报告，在有 outlier 的情况下，这套 FP8 attention 相比 baseline FP8 attention 的数值误差可降低 **2.6x**。[4]

#### 5.1.4 V3 前向流水线可以怎样理解

V3 的前向不是简单的“load -> GEMM -> softmax -> GEMM -> store”串行链路，而是把多个 tile 的阶段错开执行。一个概念化的时间片可以写成：

```text
time t:
  producer warps:
      TMA load K[t+1], V[t+1] -> shared memory buffer[next]

  consumer warpgroup A:
      WGMMA: Q * K[t]^T -> S[t]

  consumer warpgroup B / SFU:
      online softmax update for S[t-1]
      P[t-1], m, l = update_softmax(m, l, S[t-1])

  consumer warpgroup C:
      WGMMA: P[t-1] * V[t-1] -> output accumulator a

  epilogue:
      when all tiles are consumed, O = a / l
```

这里的重点不是某个阶段单独更快，而是 **TMA load、WGMMA、softmax/SFU、第二个 WGMMA 尽量不要互相空等**。Hopper 的异步指令和 warp specialization 给了这个调度空间，V3 的主要工作就是把 attention 的依赖关系改写成能利用这个空间的 pipeline。

### 5.2 V3 的效果

V3 论文给出的代表性数据是：[4]

- H100 上相对 V2 提升 **1.5x 到 2.0x**
- FP16 最多到 **740 TFLOPs/s**，约 **75% 利用率**
- FP8 接近 **1.2 PFLOPs/s**
- FP8 数值误差相对基线降低 **2.6x**

这说明从 V2 到 V3，优化重点已经不是“少写一次 HBM”这种一级改进，而是：

- 更充分利用 Hopper 的异步硬件
- 让 GEMM、softmax、load/store 真正形成流水化重叠

### 5.3 V3 的局限

V3 的问题也很清楚：

1. **高度依赖 Hopper 特性**  
   它不是给 A100 写的，而是给 H100 写的。

2. **寄存器压力更高**  
   更深的流水线、更大的 tile、更复杂的重叠都在消耗寄存器。[4]

3. **当 Tensor Core 再继续变快时，softmax / shared memory 这些“非 matmul 瓶颈”会更突出**

这正是 V4 出现的背景。

## 6. V4：FlashAttention-4（2026）

V4 是截至 **2026-04-30** 已公开的最新主线版本。  
论文提交时间是 **2026-03-05**。[9]

V4 面对的问题和 V3 明显不同。  
它不再是“怎么用上 Hopper 的新能力”，而是：

**在 Blackwell 上，Tensor Core 变得更快了，但 shared memory 带宽、指数函数单元、一般 ALU 没有等比例变快。attention 的瓶颈重新转移了。**[9][10]

Tri Dao 在 FA4 博文里把这件事概括为 **asymmetric hardware scaling**。[10]

### 6.1 V4 的五个关键改进

#### 6.1.1 重新设计前后向流水线，吃满 Blackwell 的异步 MMA

Blackwell 引入了新的 `tcgen05.mma` / UMMA 路径，CUTLASS 文档给出的说明是：

- Blackwell SM100 新的 `tcgen05.mma` 指令相对 Hopper 的 WGMMA 可达到 **2x 到 4x** 的吞吐提升[11]
- 支持 `cta_group::[1|2]` 的执行模式[11]

FA4 围绕这个能力重新做了流水线设计：

- 利用 **fully asynchronous MMA**
- 使用更大的 tile
- 尽量让 Tensor Core、softmax、内存访问三类资源重叠执行[9][10]

V4 的重点已经从“让 GEMM 更快”转成“让更快的 GEMM 不被别的阶段饿住”。

一个简化的 V4 前向流水线可以理解成：

```text
P0: TMA / shared memory 准备下一个 K/V tile
P1: UMMA / tcgen05.mma 计算 QK^T tile
P2: online softmax 统计量更新，必要时由 correction warpgroup 做 rescale
P3: 将 P 分阶段放入 TMEM，并尽早触发 P * V 的 UMMA
P4: epilogue，完成最终归一化和写回
```

和 V3 相比，V4 的压力不只是“把 P0/P1/P2/P3 重叠起来”，还要处理 Blackwell 上 Tensor Core 吞吐继续提高后带来的新失衡：如果 softmax、shared memory、TMEM、cluster 通信和 atomic reduction 跟不上，Tensor Core 会更频繁地等待。

#### 6.1.2 用软件模拟部分指数计算，压低 softmax 瓶颈

Blackwell 上 matmul 更快以后，softmax 里的 `exp` 更容易成为短板。  
因此 V4 在前向里加入了两项很有代表性的优化：[9][10]

1. **software-emulated exponential**  
   用多项式近似，把一部分指数计算分摊到 FMA 单元，而不是全部压在硬件 MUFU/SFU 上。

2. **conditional online softmax rescaling**  
   只在必要的时候做重缩放，减少不必要的标量与特殊函数开销。

这说明 V4 已经不满足于“把 softmax 藏在 GEMM 的阴影里”，而是开始直接改 softmax 这段本身的执行方式。

conditional rescaling 对应的是 online softmax 递推里的这个因子：

```text
alpha = exp(m_old - m_new)
```

V4 不是简单地“只要最大值变化就立刻重缩放”，而是引入阈值 `tau`，把小幅 rescale 从 critical path 上移走。概念上可以写成：

```text
delta = m_new - m_old

if delta > tau:
    a_new = exp(m_old - m_new) * a_old
          + sum_j exp(S_ij - m_new) * V_j
else:
    a_new = a_old
          + sum_j exp(S_ij - m_old) * V_j
```

最终仍使用真实的最终统计量做归一化，因此跳过小幅 rescale 不是改变 softmax 定义，而是把部分向量重缩放操作从主路径上拿掉。V4 把这类条件路径显式纳入 kernel 设计，减少不必要的 `exp`、乘法和向量更新。

software-emulated exponential 则可以理解为把一部分：

```text
exp(x)
```

改写成适合 FMA/ALU 执行的近似形式，例如先做换底：

```text
exp(x) = 2^(x * log2(e))
       = 2^(n + r)
       = 2^n * 2^r
```

其中 `n` 是整数部分，`r` 是小范围余数，再用多项式近似 `2^r`。这样做的目的不是改变 softmax 数学定义，而是在可控误差下把部分压力从 MUFU/SFU 转移到其它执行资源。

#### 6.1.3 用 TMEM 和 2-CTA MMA 解决 backward 的 shared memory 与 atomic 瓶颈

这是 V4 相比 V3 最大的结构性升级之一。

FA4 博文给出的关键信息是：[10]

- B200 每个 SM 有 **256 KB TMEM**
- 2-CTA MMA 允许同一 cluster 内两个 CTA 协作执行一个 MMA
- 这样可以减少 operand B 的 shared memory 流量，并把部分中间结果放在 TMEM 中

在 backward 里，V4 利用：

- **TMEM** 存中间结果，减轻 shared memory 压力
- **2-CTA MMA** 减少 shared memory traffic
- **DSMEM 交换** 解决 reduction 轴与 tile 切分冲突
- 同时把 `dQ` 的全局 atomic reduction 次数大约减半[10]

如果说 V3 的关键词是“异步重叠”，那 V4 在 backward 里更像是：  
**把共享内存、Tensor Memory、cluster 内通信、原子归约一起纳入整体调度。**

backward 的数学核心仍然是标准 softmax backward：

```text
D_i    = sum_r dO_ir * O_ir
dP_ij  = dO_i dot V_j
dS_ij  = P_ij * (dP_ij - D_i)
dQ_i  += sum_j dS_ij * K_j / sqrt(d_h)
dK_j  += sum_i dS_ij * Q_i / sqrt(d_h)
dV_j  += sum_i P_ij * dO_i
```

V4 优化的是这些张量乘、归约和写回如何在 Blackwell 上组织。尤其是 `dQ_i` 会从多个 K/V tile 得到 partial sum，如果切分方式处理不好，就容易产生大量 global atomic。V4 通过 TMEM、2-CTA MMA 和 cluster 内交换把更多 partial reduction 留在片上完成，再减少写回 HBM 时的原子归约压力。

#### 6.1.4 实现方式升级到 CuTe-DSL

V4 不只是算法升级，也显著改变了实现方式。

官方论文和博文都强调：FA4 完全使用 **CuTe-DSL** 编写。[9][10]

这意味着：

- kernel 逻辑写在 Python DSL 中
- DSL lower 到 PTX
- 再由 CUDA toolchain 编译成目标机器码

给出的收益是：

- **编译时间缩短约 20x 到 30x**[9][10]

这点很重要，因为当 kernel 已经越来越接近硬件极限时，  
**开发效率、可维护性和快速试错能力** 也开始成为“性能工程”的一部分。

#### 6.1.5 确定性模式和调度

FA4 还补了两个容易被忽略但对训练很实用的点。[10]

第一是 **deterministic backward**。  
`dQ` 的 global atomic accumulation 会带来非确定性，FA4 提供确定性模式，用 semaphore-style lock 和 memory fence 固定归约顺序，代价是吞吐会下降。官方博文给出的经验数据是，确定性 backward 仍可达到非确定性版本约 **85% 到 90%** 的吞吐。

第二是 **tile scheduling**。  
causal mask 和 variable sequence length 会让不同 tile 的工作量不同。如果 grid 按固定顺序线性扫，很容易出现尾部负载不均。FA4 通过 grid swizzle、longest-processing-time-first / shortest-processing-time-first 等调度策略减轻 load imbalance。这个点不是 attention 数学变化，但会明显影响端到端 kernel 利用率。

### 6.2 V4 的效果

FA4 论文给出的 headline numbers 是：[9]

- B200 上 BF16 最多到 **1613 TFLOPs/s**
- 利用率约 **71%**
- 相对 cuDNN 9.13 最多 **1.3x**
- 相对 Triton 最多 **2.7x**

Tri Dao 博文还补充了一个很有价值的现实信息：[10]

- 自 FA4 初始代码发布后的几个月里，cuDNN 也吸收了不少相同思路
- 最新版 cuDNN 的性能已经与 FA4 比较接近

这也从侧面说明：  
FlashAttention 系列不仅是“自己快”，还在持续塑造整个 attention kernel 的工业实现方式。

## 7. V1 到 V4 的核心差异点

下面给一个尽量抓主线的对比表。

| 维度 | V1 | V2 | V3 | V4 |
| --- | --- | --- | --- | --- |
| 主要问题 | HBM 读写过多，attention matrix materialization 太贵 | GPU 并行度不足，warp 通信多 | Hopper 新硬件没被充分利用 | Blackwell 上非 matmul 瓶颈重新凸显 |
| 核心思想 | IO-aware tiling + online softmax | 更好的 thread block / warp 工作划分 | 异步执行 + warp specialization + FP8 | 算法与 kernel 协同设计，继续重构 softmax / memory / backward |
| 前向主循环 | 按 Q/K/V tile 流式计算 | 保留分块，但减少 non-matmul FLOPs | Producer-consumer 流水线，GEMM 与 softmax 重叠 | 重新设计前向流水线，利用 fully async MMA 和更大 tile |
| warp 分工 | 仍有较多跨 warp 通信 | 从 split-K 倾向转向 split-Q，减少通信 | producer warp 负责 TMA，consumer warp 负责 WGMMA | 更进一步结合 TMEM、2-CTA、cluster 级能力 |
| 低精度 | 不是重点 | 不是重点 | 正式支持 FP8，并针对误差做 block quantization + incoherent processing | 继续面向更快 Tensor Core 路径，并处理 softmax / memory 新瓶颈 |
| backward 思路 | 重计算替代存完整 `P` | 与 V1 相近，但保存 `logsumexp` 等更简化 | 继承异步设计 | 重点优化 shared memory traffic、TMEM 使用和原子归约 |
| 目标硬件 | Ampere | Ampere | Hopper | Blackwell（实现上也支持 Hopper/Blackwell 的 CuTeDSL 路线） |
| 实现关键词 | CUDA fused kernel | 更优 work partitioning | TMA, WGMMA, `setmaxnreg`, FP8 | `tcgen05.mma`, TMEM, 2-CTA MMA, CuTe-DSL, deterministic backward, tile scheduling |

## 8. 从一代到下一代，究竟“改”了什么

这个问题最容易被说得太泛。下面只说最本质的变化。

### 8.1 V1 -> V2

不是换 attention 公式，而是：

- **从“避免 HBM 回写”进一步走到“把 GPU 并行划分做好”**
- **从 IO-aware 走到 work-partition-aware**

V1 已经证明标准 attention 可以流式算；V2 证明 attention kernel 还必须像高性能 GEMM 一样细抠 thread block 与 warp 协作。

### 8.2 V2 -> V3

不是再做一次 Ampere 调参，而是：

- **从同步式分块 kernel 走到异步流水线 kernel**
- **从“把 FlashAttention 搬到 H100 上”走到“按 H100 的硬件心智模型重写 FlashAttention”**

V3 把 TMA、WGMMA、warp specialization、GEMM-softmax overlap、FP8 统一进一个设计里。

### 8.3 V3 -> V4

不是简单把 Hopper 代码迁到 Blackwell，而是：

- **从利用异步 Tensor Core，进一步走到应对“非对称硬件扩展”**
- **从“matmul 足够快”转到“softmax / SMEM / atomic / scheduling 才是真短板”**

V4 是目前最明显体现“attention kernel 设计要与硬件路线图共演化”的一代。

## 9. 如何理解这条技术路线

从 V1 到 V4，可以把 FlashAttention 的演进总结成四层：

1. **V1：重排计算顺序**  
   别先存整张 attention matrix。

2. **V2：重排并行工作**  
   别让 warp 因为错误的分工而反复通信。

3. **V3：重排执行时序**  
   让数据搬运、GEMM、softmax 尽可能并行。

4. **V4：重排整个资源图**  
   Tensor Core、TMEM、SMEM、SFU、cluster、atomic 都要一起看。

因此 FlashAttention 的历史，本质上是：

**标准 attention 的数学语义不怎么变，但它的执行图在持续演化。**

## 10. 写作时建议怎么表述

如果这份材料后面要改成知乎文章、报告或分享，建议避免两个常见误区：

1. **不要把 FlashAttention 说成“稀疏/低秩/局部窗口 attention”。**  
   FP16/BF16 主线仍是标准 scaled dot-product attention，只是重排了执行顺序；FP8 路径则需要单独说明量化误差。

2. **不要把版本演进简单理解为“更快的 CUDA 优化”。**  
   真正的主线是：  
   IO-aware -> work partitioning -> asynchrony -> algorithm/kernel co-design。

## 11. 核对后的纠错与边界

这次核对后，文中需要特别固定住以下边界，避免后续写作时误读：

1. **标准 attention 公式必须包含 scaling 和 mask。**  
   `S = QK^T / sqrt(d_h) + M`，不能只写 `S = QK^T`。省略 scaling 会让公式不完整，省略 mask 会把 causal / padding 场景说丢。

2. **V1/V2 的差异不是数学公式差异。**  
   两者都利用 online softmax。V2 的关键是减少 non-matmul FLOPs、重排 thread block 级并行和把 warp 分工从 sliced-K 倾向改到 sliced-Q 倾向。

3. **“exact attention”要带精度边界。**  
   对 FP16/BF16 路径，可以说 FlashAttention 保持标准 attention 语义，不做稀疏或低秩近似；对 V3/V4 的 FP8 路径，应该说算法结构仍是标准 attention，但量化会引入数值误差。

4. **V3 的核心不是单纯使用 TMA/WGMMA，而是异步流水线。**  
   TMA、WGMMA、warp specialization、ping-pong scheduling、GEMM-softmax overlap 要一起出现，单独列硬件名不足以说明 V3。

5. **V4 的核心不是 Blackwell 上“再快一点”。**  
   论文和官方博文强调的是 asymmetric hardware scaling：Tensor Core 增速快于 SMEM、MUFU/SFU、ALU 等资源，所以 softmax、SMEM、TMEM、cluster 通信、atomic reduction 变成新的系统性瓶颈。conditional rescaling 也不是“最大值不变才跳过”，而是带阈值 `tau` 的条件路径。

6. **性能数字要和硬件、精度、kernel 形态绑定。**  
   V2 的 A100 数字、V3 的 H100 数字、V4 的 B200 数字不能直接跨硬件比较，也不能把 FP8 forward 的吞吐和 BF16 forward/backward 混在一起比较。

7. **FlashDecoding 不是 V3/V4。**  
   它是 decoding 场景的旁支优化路线，和本文 V1-V4 的主线版本演进需要分开叙述。

## 12. 参考文献

1. Tri Dao, Daniel Y. Fu, Stefano Ermon, Atri Rudra, Christopher Ré. *FlashAttention: Fast and Memory-Efficient Exact Attention with IO-Awareness*. arXiv:2205.14135, submitted on May 27, 2022, revised on June 23, 2022. https://arxiv.org/abs/2205.14135
2. Milakov, M., Gimelshein, N. *Online normalizer calculation for softmax*. arXiv:1805.02867, 2018. https://arxiv.org/abs/1805.02867
3. Tri Dao. *FlashAttention-2: Faster Attention with Better Parallelism and Work Partitioning*. arXiv:2307.08691, submitted on July 17, 2023; published at ICLR 2024. https://arxiv.org/abs/2307.08691
4. Jay Shah, Ganesh Bikshandi, Ying Zhang, Vijay Thakkar, Pradeep Ramani, Tri Dao. *FlashAttention-3: Fast and Accurate Attention with Asynchrony and Low-precision*. arXiv:2407.08608, submitted on July 11, 2024. https://arxiv.org/abs/2407.08608
5. NVIDIA. *CuTe TMA Tensors*. CUTLASS Documentation. 说明了 Hopper 引入的 TMA 如何在 global memory 与 shared memory 之间搬运整块 tensor tile。https://docs.nvidia.com/cutlass/4.3.0/media/docs/cpp/cute/0z_tma_tensors.html
6. NVIDIA. *NVIDIA Hopper Architecture In-Depth*. NVIDIA Technical Blog, March 22, 2022. 介绍 Hopper 的 TMA、异步事务屏障与线程块 cluster。https://developer.nvidia.com/blog/nvidia-hopper-architecture-in-depth/
7. NVIDIA. *Hopper Tuning Guide*. CUDA Documentation. 用于了解 Hopper 的 memory hierarchy、SM 组织与调优背景。https://docs.nvidia.com/cuda/hopper-tuning-guide/index.html
8. Tri Dao. *FlashAttention-3: Fast and Accurate Attention with Asynchrony and Low-precision*. 官方博文，2024-07-11。对 WGMMA、TMA、ping-pong scheduling、FP8 精度处理的解释更直观。https://tridao.me/blog/2024/flash3/
9. Ted Zadouri, Markus Hoehnerbach, Jay Shah, Timmy Liu, Vijay Thakkar, Tri Dao. *FlashAttention-4: Algorithm and Kernel Pipelining Co-Design for Asymmetric Hardware Scaling*. arXiv:2603.05451, submitted on March 5, 2026. https://arxiv.org/abs/2603.05451
10. Tri Dao. *FlashAttention-4: Algorithm and Kernel Pipelining Co-Design for Asymmetric Hardware Scaling*. 官方博文，2026-03-05。对 Blackwell 的非对称扩展、TMEM、2-CTA MMA、conditional softmax rescaling 有更直接的解释。https://tridao.me/blog/2026/flash4/
11. NVIDIA. *Blackwell SM100 GEMMs*. CUTLASS Documentation. 说明 `tcgen05.mma`、`cta_group::[1|2]` 等 Blackwell Tensor Core 新能力。https://docs.nvidia.com/cutlass/latest/media/docs/cpp/blackwell_functionality.html
12. NVIDIA. *Blackwell Tuning Guide*. CUDA Documentation, Release 12.8. 用于了解 Blackwell 的 SM、cluster、shared memory / L1 结构等。https://docs.nvidia.com/cuda/archive/12.8.0/pdf/Blackwell_Tuning_Guide.pdf
13. Dao-AILab. *flash-attention* 官方仓库 README。用于确认当前工程实现中 FlashAttention-3 beta 与 FlashAttention-4（CuTeDSL）的发布形态和安装入口。https://github.com/Dao-AILab/flash-attention

## 13. 补充资料

- Tri Dao 个人主页的 publications 页面：可以集中查看 FlashAttention 相关论文与 PDF  
  https://tridao.me/publications/
- Tri Dao 博客主页：可以集中查看 FlashAttention-3 / 4 的解释性文章  
  https://tridao.me/blog/
- FlashAttention 官方实现仓库 release 页面：可查看当前版本发布节奏  
  https://github.com/Dao-AILab/flash-attention/releases

## 14. 一句话总结

**V1 把 attention 从“存大矩阵”改成“流式标准计算”；V2 把它做得更像高性能 GEMM；V3 把它改造成 Hopper 友好的异步流水线；V4 则进一步围绕 Blackwell 的非对称硬件扩展，联合优化 softmax、TMEM、shared memory、atomic 与 kernel 开发方式。**
