# 寄存器这一存储层级，究竟为架构提供了什么？

> 一些关于 GPU 与 CPU/NPU 设计哲学融合的思考

---


最近被问了这样一个问题：如果有一个足够聪明的程序员或者编译器，知道每条指令的时延，知道每条指令用的物理寄存器在哪个 bank 上，那乱序执行还有意义吗？而且寄存器的带宽需求会很小，用 SRAM 搭就够了。

先不正面回答。先假设这个程序员和编译器真实存在。他们能通过巧妙的编排，让计算单元每次都差不多刚好处在能拿到提前发出的预取数据的时间点。那是不是我就不需要那么多寄存器了？只要寄存器能同时供给多个计算 pipe 读写就行？

第一段是 GPU 的解题思路。第二段是某些 NPU 或者类 CPU 的解题思路。这两段话指向的其实是一个问题：寄存器这一层，在架构里到底在扮演什么角色？

我认为，在不同的架构和不同的编程方式下，对寄存器的诉求是不一样的。


## 寄存器的两个属性

寄存器作为离计算单元最近的存储层，有两个属性：

**带宽**：单位时间能喂给计算单元多少数据。决定了算子利用率的上限。
**数量**（容量）：寄存器能同时存多少个值。决定了能有多少"在途"（in-flight）的指令和线程。

这两个属性服务的诉求完全不同：

- 带宽 → 确定时延下的更高算子利用率。比如 FMA 延迟是固定的 4 周期，你要有足够的带宽同时喂多个计算 pipe，让每个周期都有指令能发射出去。
- 数量 → 不定长时延下最多可并发的线程。比如一条 load 要等 400 周期才能回来，你需要足够的寄存器来保留这些还没完成的线程上下文，好切换到别的线程继续算。

这两件事听起来很像？其实可以解耦。而 GPU 和 CPU/NPU 走的就是两条不同的解耦路线。


## GPU：用海量寄存器硬扛

NVIDIA 的寄存器文件大到什么程度？Hopper H100 上，每个 SM 有 **65,536 个 32-bit 寄存器**（64K registers）[1]。每个 SM 最多能同时有 **64 个 warp（2048 个线程）**，每个线程最多用 255 个寄存器 [2]。

为什么要这么多？

GPU 的编程模型是 SIMT（Single Instruction, Multiple Threads）。每个线程在 CUDA 里是一个独立执行上下文，GPU 的 warp scheduler 切换 warp 时——零开销。因为每个线程的状态全部存在寄存器里。warp A 发了一条访存指令，要等 400 个周期的 HBM 延迟，scheduler 只需要把 PC 指针指到 warp B，warp B 的所有状态已经在寄存器里等着了。

这就是经典的延迟隐藏（Latency Hiding）：用超多线程交错执行，让计算单元一直有活干。

代价？寄存器文件大得离谱、贵得离谱。Pascal 时代寄存器文件占 SM 片上存储面积的 60% 以上 [3]。因为每个 warp 的 32 个线程可能同时访问寄存器，文件必须切成大量 bank（一般 16 个或更多），还要靠精细编排来避免 bank conflict [4]。有研究说寄存器 bank conflict 能让计算时间增加 12% [5]。

那个"足够聪明的程序员"能做什么？他可以编排 bank 分配、编排发射次序、编排 warp 切换的 timing。

但他改变不了根本事实：在 GPU 的编程模型里，**每个在途未返回数据的指令，都需要占一个寄存器**。长时延访存指令越多，要的寄存器就越多。数量和线程并行度之间是强绑定。

数据佐证一下：从 Kepler 到 Ampere，NVIDIA 一直维持最大 64 个并发 warp/SM [6]。架构师们很清楚，64 个 warp 的寄存器总量是实现延迟隐藏的底线。


## CPU/NPU：解耦，让 Buffer 分担容量

再看 CPU 和某些 NPU。

CPU 的向量寄存器（比如 AVX-512 的 32 个 512-bit 寄存器）数量很少，但读写端口多、时延短（1 cycle）。乱序执行引擎靠 ROB（重排序缓冲区）、物理寄存器重命名、发射队列这些机制，在少量物理寄存器上实现了很高的 ILP [7]。

怎么做到的？靠核内 Buffer。

CPU 的 Load/Store Queue、MSHR、各级缓存，本质上是"大容量、低带宽"的角色——负责把访存操作发出去、存预取回来的数据。而真正的架构寄存器只服务"我立刻就要读写的数据"，数量少但带宽极高。

某些 NPU 架构把这个思路放得更大了。引入核内暂存存储器（Scratchpad / Buffer）后，架构变成了三层：

```
计算单元 ←→ 寄存器文件（少、快）
                 ↑
          核内 Buffer（大、慢）
                 ↑
           HBM / 全局内存
```

Buffer 层承担"容量"诉求——launch 长时延操作、暂存中间结果、解耦访存和计算。寄存器层专注"带宽"诉求——在确定时延下给多个 pipe 高速喂数据。

这个思路最极致的体现是 NVIDIA Hopper 引入的 **TMA（Tensor Memory Accelerator）和 Warp Specialization** [8]。Hopper 上你可以让一组 warp 专门做访存（用 TMA 把数据异步搬到共享内存），另一组专门做计算（在寄存器里做 Tensor Core 和 Vector Core 的运算）。两组 warp 通过 Shared Memory（Buffer）交互。这不就是 CPU 式的 Buffer + 寄存器分层架构吗。

Hopper Tuning Guide 明确说了：**TMA 让开发者可以写 warp-specialized 的代码，特定 warp 负责各层间的数据搬运，其他 warp 只在 SM 内部操作本地数据** [9]。这已经不是"每个线程各自为战"的 SIMT 了，而是多专家协同——访存专家、矩阵计算专家、向量计算专家各干各的。


## 带宽和数量真的能解耦吗？

回到最初的问题。寄存器的两个属性既然服务于两个不同的诉求，那能不能引入一个同级的 Buffer 层来承担"数量"的诉求？

答案是：看架构和编程模型，完全可以。

反正是，我增加数量只是想把这些长时延操作 launch 出去，实际上可能不需要对它们频繁读写。那我不如加一个 Buffer。等到我真的需要多指令并发访问数据时，再用数量少但带宽高的寄存器完成计算。

| 属性 | 寄存器（高带宽） | Buffer（大容量） |
|------|:----------------:|:----------------:|
| 时延 | 1 cycle | 几个 cycle |
| 带宽 | 极高（多端口） | 中等（单/双端口） |
| 容量 | 小（~128-256 个） | 大（64-256 KB） |
| 服务场景 | 多指令并行读写 | Launch 长时延操作 |

寄存器用少量但贵的多端口 SRAM 做高带宽，Buffer 用大量但便宜的单/双端口 SRAM 做大容量。解耦之后：

1. 寄存器回归本职工作：只做高带宽、单周期的数据供给
2. Buffer 用 SRAM 做大容量，面积和功耗远低于同容量的多端口寄存器
3. 并行度从"线程级"变成"任务级"——访存、计算、控制流各干各的
4. 乱序+多发射仍然是自然选择：当计算单元需要同时拿到 FMA、CVT、EXP、INT ALU 不同指令的操作数时，乱序和多发射就是提高 IPC 的正解

这个方向学术界也在探索。MICRO 2015 的《GPU Register File Virtualization》提了一个想法：编译器利用寄存器生命周期信息，在 warp 之间"虚拟化"物理寄存器 [10]。MICRO 2011 的《A Compile-Time Managed Multi-Level Register File Hierarchy》直接搞了寄存器文件分层、用 Buffer 分担容量 [11]。


## 两个聪明程序员的终点

这两种编程哲学走到了两个方向，对应了两个极聪明的程序员。

**第一种**：充分挖掘细粒度线程并行度，编排 bank conflict，用海量 warp 打满计算效率。CUDA 的标量线程抽象被用到极致。

但上限在哪？当计算不是瓶颈，数据搬运和算子多样化变成瓶颈时，纯靠多线程延迟隐藏会遇到算子利用率的门槛。一个 warp 做 FMA 时，另一个 warp 也只能做 FMA——都在跑同一个 kernel。不同算子类型（FMA、CVT、EXP、INT）如果在时间上是串行的，硬件利用率就上不去。

**第二种**：把并行度拆到多个部件的计算任务，让机器做 MoE 式的架构。访存专家负责数据搬运（TMA），矩阵专家算矩阵（Tensor Core / WGMMA），向量专家算向量（CUDA Cores）。多个专家通过 Buffer（Shared Memory）交互，核内再靠乱序+多发射榨干算子性能——让 FMA、CVT、EXP、INT ALU 这些用独立算子资源的指令同时上 pipe。

这个方向的硬件基础，NVIDIA 一直在补：

- **Volta（2017）**：引入独立线程调度，让 warp 内的线程可以独立执行 [12]
- **Ampere（2020）**：硬件加速的异步拷贝和 barrier [13]
- **Hopper（2022）**：TMA + WGMMA + 8 个独立 scheduler，正式给 warp specialization 砸了一等支持 [14]
- **Blackwell（2024）**：接着强化


## 那开头那个程序员存在吗？

不存在。因为实际硬件不是编排出来的。有不可预测的 bank conflict、缓存缺失、写后读依赖、资源竞争。乱序执行、多发射、寄存器重命名，本质就是用硬件弥补编译器无法完美静态编排的缺陷。

但够聪明之后，选择就很清晰了：

- **守旧者**：只看 GPU 的海量寄存器 + 多线程，想用标量线程模型覆盖一切
- **弯道超车者**：用 CPU 思路做 NPU，重视 Buffer 和大容量暂存，但放弃了 SIMT 的灵活性和延迟隐藏
- **同行或超越者**：两者兼顾。理解两种架构各自的适用场景，在 GPU 里融入 Buffer + 乱序 + 多发射的分层设计，让寄存器的带宽和数量真的解耦

NVIDIA 自己在做的事，就是不断从两个方向吸收东西。Hopper 和 Blackwell 里，GPU 已经不是单纯的"暴力多线程机"，而是融合了 GPU 的线程并行能力、CPU 的乱序控制逻辑、专用加速器流水线的混合体。

CPU 和 GPU 的设计哲学在走向融合。谁守旧，谁超越，谁两者兼顾做真正的同行者，还不好说。但能看到两种设计有融合的机会。


## 参考来源

[1] NVIDIA H100 Tensor Core GPU Architecture Whitepaper. https://resources.nvidia.com/en-us-hopper-architecture/nvidia-h100-tensor-c
[2] NVIDIA CUDA C++ Programming Guide, Appendix. https://docs.nvidia.com/cuda/cuda-c-programming-guide/
[3] Gebhart et al., "Energy-efficient mechanisms for managing thread context in throughput processors", ISCA 2011.
[4] NVIDIA GP100 (Pascal) Whitepaper, 2016.
[5] Hayes & Zhang, "Unified On-Chip Memory Allocation for SIMT Warp Execution", PACT 2016.
[6] NVIDIA Hopper Tuning Guide, Occupancy. https://docs.nvidia.com/cuda/hopper-tuning-guide/
[7] Hennessy & Patterson, "Computer Architecture: A Quantitative Approach", 6th Ed, 2017.
[8] H100 Whitepaper, Section on TMA.
[9] NVIDIA Hopper Tuning Guide, TMA. https://docs.nvidia.com/cuda/hopper-tuning-guide/
[10] Jeon et al., "GPU Register File Virtualization", MICRO 2015. DOI: 10.1145/2830772.2830777.
[11] Gebhart et al., "A Compile-Time Managed Multi-Level Register File Hierarchy", MICRO 2011. DOI: 10.1145/2155620.2155645.
[12] NVIDIA Volta V100 Whitepaper. https://images.nvidia.com/content/volta-architecture/pdf/volta-architecture-whitepaper.pdf
[13] NVIDIA Ampere A100 Whitepaper. https://www.nvidia.com/content/dam/en-zz/Solutions/Data-Center/nvidia-ampere-architecture-whitepaper.pdf
[14] NVIDIA Hopper Tuning Guide. https://docs.nvidia.com/cuda/hopper-tuning-guide/

---

*首发知乎。欢迎讨论。*
