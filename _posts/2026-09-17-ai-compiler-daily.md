---
layout: default
categories: [ai-compiler, daily]
title: "PyTorch 大会编译专场预告 torch.compile 图捕获与自动调优新方向；Splyce 以选择性谓词加速稀疏协同迭代向量化"
date: 2026-09-17
generation_mode: "deepseek"
window_since: "2026-09-16T08:00:00+08:00"
cutoff_at: "2026-09-17T08:00:00+08:00"
---

# PyTorch 大会编译专场预告 torch.compile 图捕获与自动调优新方向；Splyce 以选择性谓词加速稀疏协同迭代向量化

## 今日概览

2026 年 9 月 17 日的两条进展都聚焦于攻克编译器面前的难优化模式：PyTorch 官方博客预告了 PyTorch Conference North America 2026 的编译与优化专场，涵盖 torch.compile 动态 shape 图捕获成本、Helion 异构后端与 LLM 引导自动调优等内容；学术侧则出现基于 MLIR 的 Splyce 框架，用双路径执行与选择性谓词消除稀疏协同迭代中的数据依赖分支，在合成内核上取得 1.96 至 2.86 倍加速。

## 重点变化

### 1. Open Research, Tooling & Optimization at PyTorch Conference North America 2026

PyTorch 博客提前公布了 PyTorch Conference North America 2026 的编译与优化专场安排，覆盖 torch.compile、TorchInductor、Helion、vLLM 与分布式训练等多条技术路线。torch.compile 方面，Dynamo 计划将嵌套 graph break 的成本从 O(N²) 次 frame trace 降到 O(N) 次；新的 C++ FakeTensor 实现在 aten.mm 上带来 30 倍加速，用于缩短冷启动编译时间；参数化动态 shape CUDA Graph 复用单一图，免去逐 shape 重录制；unbacked dynamic shapes 则面向 vLLM 和 export 等显式图捕获工作流。TorchInductor 集成了 AMD 的 FlyDSL MLIR 原生 GEMM 后端；Helion 新增 CuteDSL 与 Pallas 后端，并以 LFBO 与 LLM 引导搜索压缩 autotuning 时间。KernelAgent 加入硬件引导的多智能体优化层，自称较默认 torch.compile 快 1.56 倍、达到 H100 roofline 的 89%；vLLM 公布 Intel GPU 的 Triton 内核策略；FlexGEMM 提议用普通 PyTorch 函数编写 GEMM epilogue 并融合进 store path；IBM Research 的 Helion Paged Attention 后端报告延迟最高降 50%；Meta 还提出了分布式 SPMD 训练的 Precompile 抽象，实现编译一次、跨 rank 复用。

**为什么重要：** 这些是提前发布的会议议程，性能数字来自演讲者自述，最终效果需待演讲细节和基准公开后验证。但整份议程清晰勾勒出 PyTorch 编译栈近期的重心：压低动态 shape 图捕获与冷启动的开销、通过 Helion 这类 DSL 让单一内核源码覆盖 NVIDIA、AMD、TPU 等异构硬件、并用贝叶斯优化、LLM 与多智能体方法替代手写调优。它表明内核优化正在从人工工程转向自动化搜索，且编译产物复用开始从推理端延伸到分布式训练端。

**涉及层级：** `frontend`、`graph_ir`、`tensor_ir`、`optimization`、`kernel_codegen`、`hardware_backend`、`runtime`、`distributed`、`benchmark`

**来源：** [原始来源 1](https://pytorch.org/blog/open-research-tooling-optimization-at-pytorch-conference-north-america-2026/)

**事件评分：** 89.2/100

### 2. Splyce: SIMD Vectorization of Sparse Coiteration

Splyce 是一个基于 MLIR 的自动向量化框架，专门解决稀疏张量收缩中稀疏-稀疏协同迭代循环难以标准向量化的问题。它提出双路径执行模型，将坐标求交与指针管理解耦，再通过选择性谓词把数据依赖分支转化为可预测的数据流，从而消除不可预测的控制流，暴露可并发执行的独立计算，提升指令级并行与功能单元利用率。在合成稀疏张量内核上，Splyce 报告了 1.96 至 2.86 倍的性能提升，并在 SuiteSparse 集合的多数真实不规则数据集上保持了稳定加速。

**为什么重要：** 稀疏-稀疏协同迭代长期受困于数据依赖分支，传统循环向量化难以生效，而 Splyce 的谓词化变换不要求更换稀疏格式或算法，直接改写现有内核的控制流结构，让现代超标量处理器更充分地利用执行单元，对依赖稀疏计算的图网络与科学计算负载有实际价值。需要留意的是，其在真实数据集上的加速幅度仍有赖于后续公开的完整基准细节加以确认。

**涉及层级：** `tensor_ir`、`optimization`、`kernel_codegen`

**来源：** [原始来源 1](https://arxiv.org/abs/2609.19410v1)

**事件评分：** 67.8/100

## 值得继续观察

- torch.compile 嵌套 graph break 成本 O(N) 优化与 C++ FakeTensor 落地进展
- Helion CuteDSL/Pallas 异构后端及 LFBO、LLM 引导 autotuning 实测结果
- KernelAgent 硬件引导多智能体内核优化的 H100 roofline 数据
- Splyce 选择性谓词稀疏向量化在更多真实负载上的泛化表现
- Precompile for Training 与 FlexGEMM epilogue 融合前端的后续开源情况

## 数据与生成说明

- 采集窗口：`2026-09-16T08:00:00+08:00` 至 `2026-09-17T08:00:00+08:00`
- 生成模式：`deepseek`
- 文章由程序根据一手来源生成；性能结论应以链接中的原始配置为准。
- 原始条目：NVIDIA 3 条，PyTorch 2 条，arXiv 3 条
