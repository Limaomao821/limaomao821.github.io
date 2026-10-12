---
layout: default
categories: [ai-compiler, daily]
title: "LLM 智能体深入内核优化与基准测试：KernelOPT 搜索 Triton 子内核，KREX 加速共享 GPU 上的并发测量"
date: 2026-09-25
generation_mode: "deepseek"
window_since: "2026-09-24T08:00:00+08:00"
cutoff_at: "2026-09-25T08:00:00+08:00"
---

# LLM 智能体深入内核优化与基准测试：KernelOPT 搜索 Triton 子内核，KREX 加速共享 GPU 上的并发测量

## 今日概览

2026年9月25日的编译视角集中在两件事上：LLM 智能体正从单纯生成内核走向以编译产物为对象做系统性搜索与验证，同时服务于这些智能体的基准测试与软硬件基础设施也在同步演进。KernelOPT 用五个 profiling 引导的智能体和四门验证级联，在 KernelBench 上相对 torch.compile 取得 Level 1 至 3 分别为 1.40x、1.15x、1.07x 的几何平均加速；KREX 为这类 agent 提供区域粒度独占的并发基准测试运行时，相对命令粒度独占基线最高提升 3.4 倍吞吐；TileBench 给出 Triton 与 cuTile 在 B200 上的受控对照，并比较了 LLM 生成内核的 token 效率。在编译与硬件协同方向，EAAC 展示基于 MLIR 的软硬件协同设计如何把 GEMM 加速器原型端到端跑通并量化编译器信号量分配成本，MQSS-Selector 则尝试用强化学习把 MLIR pass selection 与设备选择、队列调度统一为单一学习式 selector。

## 重点变化

### 1. KernelOPT: Dispatch-Aware Agentic Search for GPU Kernel Optimization

KernelOPT 是一个 dispatch-aware 的多智能体 LLM 系统，把 PyTorch Inductor/torch.compile 编译后的模型当作结构化产物来处理。它保留 cuBLAS、cuDNN 等 vendor library 调用，只用五个 profiling 引导的 LLM 智能体去优化生成的 Triton sub-kernel；候选须依次通过静态校验、多种子正确性、模型级 float64-fallback 验证与性能门控四层验证，之后对重新拼接的模型做端到端验证，若没有候选通过全部四门则回退到编译器基线。系统接受 PyTorch nn.Modules、独立 Triton kernel 与 Helion kernel，在 250 个 KernelBench 问题上相对 torch.compile 取得 Level 1 1.40x、Level 2 1.15x、Level 3 1.07x 的几何平均加速。

**为什么重要：** 它把 LLM 内核搜索限定在编译器生成的子内核上而不是重写整个模型，既缩小了搜索空间，又保留了经过充分优化的 vendor library；四门验证级联则为智能体生成的候选提供了一层可量化的可靠性保障。这为智能体式内核优化如何嵌入现有编译栈给出了一个可复用的范式。

**涉及层级：** `kernel_codegen`、`optimization`、`benchmark`

**来源：** [原始来源 1](https://arxiv.org/abs/2609.30059v1)

**事件评分：** 74.0/100

### 2. Compiler and Hardware Co-Design for Accelerator Architectures

论文提出 EAAC（Extensible Accelerator Architecture），一套基于 MLIR 的编译器与硬件架构，目标是降低硬件与编译器协同开发的开销、加速加速器原型设计。EAAC 面向静态数据流负载，借助 MLIR 支持多种前端集成，只提供必要的数据编排编译器功能。作者用一个脉动阵列加嵌入式 RISC-V 核的 GEMM 加速器原型做了端到端编译验证，在合成全连接层负载上相对纯 RISC-V 基线取得 28x 执行时间加速，其中 GEMM 操作本身只占执行时间的 1.64%。他们还刻画了编译器的硬件信号量分配行为，指出最坏情况下所需信号量数量随指令数线性增长，并将其列为后续优化目标。

**为什么重要：** 它展示了编译器基础设施如何成为加速器原型设计的组成部分而非事后补充，为快速硬件原型提供了相对空白的起点。把信号量分配这类编译器与硬件接口问题量化出来，也给了软硬件协同开发一个具体、可度量的下一个优化方向。

**涉及层级：** `frontend`、`tensor_ir`、`hardware_backend`、`benchmark`

**来源：** [原始来源 1](https://arxiv.org/abs/2609.30099v1)

**事件评分：** 73.7/100

### 3. MQSS-Selector: RL-Guided Pass Selection for an MLIR Compilation Pipeline

MQSS-Selector 提出一种基于强化学习与深度学习的统一 selector，把 HPCQC 场景中原本分散的 device selection、compiler-pass optimization 与 job queue scheduling 整合进一个统一框架，为 MLIR 编译流水线提供 RL 引导的 pass selection。论文声称该方案可以同时优化 fidelity、compilation time 与 scheduling latency 等多目标，并根据电路特征与设备条件动态适应。

**为什么重要：** 如果设备选择、pass 选择与队列调度这些原本相互独立的决策环节能被一个学习式 selector 联合优化，编译流水线就能在资源受限的共享环境中做全局权衡，而不仅是单点调优。不过目前论文给出的是框架性描述，端到端效果还有待基准验证。

**涉及层级：** `optimization`、`runtime`

**来源：** [原始来源 1](https://arxiv.org/abs/2609.30104v1)

**事件评分：** 64.5/100

## 论文速递

### 1. TileBench: A Controlled Benchmark for Performance Evaluation and Bottleneck Diagnosis of Tile-Based Programming Models

TileBench 是一个受控 benchmark，在 NVIDIA B200 GPU 上以匹配的算子语义和可比实现结构系统评估 Triton 与 cuTile 两种 tile-based 编程模型。它包含 45 个覆盖多样 AI-kernel 模式的算子，每个任务提供 PyTorch 参考实现、验证过的 Triton 与 cuTile 实现、dtype 与输入规模扫描、默认与 autotuned 配置、roofline 指标和 profiling 引导诊断。评估发现性能差距与 workload 相关：cuTile 在 Tensor-Core/TMA 友好的 kernel 上更强，Triton 在不规则、streaming 与带宽受限算子上更强；在相同 iterative refinement 协议下，LLM 生成的 Triton kernel 的 token 效率持续高于 cuTile。项目已在 GitHub 公开。

**为什么重要：** 它为两个主流 tile-based 编程模型提供了统一且可复现的对照基准，把哪个模型更好的问题变成可以按算子类型来回答的问题。LLM 生成 kernel 的 token 效率对比，对后续代码生成智能体的目标语言选择有直接参考价值。

**涉及层级：** `kernel_codegen`、`hardware_backend`、`benchmark`

**来源：** [原始来源 1](https://arxiv.org/abs/2609.29067v1)

**事件评分：** 63.6/100

### 2. KREX: Concurrent Kernel Benchmarking on Shared GPUs via Region-Granular Exclusivity

KREX 是面向共享 GPU 上并发内核基准测试的运行时，服务于 LLM 驱动的 GPU 内核优化 agent。agent 可以在基准测试命令内标记计时敏感的关键区域，运行时只在标记区域内强制独占，通过阻塞新的竞争性 GPU 提交、排空在途工作、冻结兄弟进程并隔离 CPU 核来保护 GPU 执行与驱动测量的宿主线程，区域外则允许并发；同时用持久化 context 进程复用 GPU context，避免节点级串行化的创建。在 NVIDIA 与 AMD GPU 上，相对命令粒度独占基线最高取得 3.4 倍基准测试吞吐，p95 计时膨胀仅为 0.30%、1.58%、3.90%（分别对应核时长大于 10、1、0.1 ms）。

**为什么重要：** 内核优化智能体每轮迭代都要在真实 GPU 上测量候选性能，基准测试吞吐直接决定搜索效率。KREX 表明可以在几乎不牺牲测量保真度的情况下大幅提高共享 GPU 上的并行测量能力，对 Agent 型内核搜索工具链的规模化是重要支撑。

**涉及层级：** `runtime`、`benchmark`

**来源：** [原始来源 1](https://arxiv.org/abs/2609.30057v1)

**事件评分：** 61.5/100

## 值得继续观察

- KernelOPT 的验证级联与加速比在更大模型和更多后端上的泛化表现
- TileBench 是否扩展到更多 GPU 代际与编程模型
- KREX 区域粒度独占在更多工作负载下的测量偏差
- EAAC 编译器信号量分配最坏情况的优化进展
- MQSS-Selector 的端到端基准结果

## 数据与生成说明

- 采集窗口：`2026-09-24T08:00:00+08:00` 至 `2026-09-25T08:00:00+08:00`
- 生成模式：`deepseek`
- 文章由程序根据一手来源生成；性能结论应以链接中的原始配置为准。
- 原始条目：LLVM 1 条，NVIDIA 1 条，PyTorch 1 条，arXiv 7 条
