---
layout: default
categories: [ai-compiler, daily]
title: "编译器自动化三线推进：PGO 驱动的 PIM 卸载、LLM agent 的 RL 内核优化与 F₂ 代数学习的 SASS 编码器"
date: 2026-09-29
generation_mode: "deepseek"
window_since: "2026-09-28T08:00:00+08:00"
cutoff_at: "2026-09-29T08:00:00+08:00"
---

# 编译器自动化三线推进：PGO 驱动的 PIM 卸载、LLM agent 的 RL 内核优化与 F₂ 代数学习的 SASS 编码器

## 今日概览

本期三个事件分别落在编译优化管线的不同层次。Torch-PIM 把 profile-guided optimization 用在 progressive lowering 物化的 loop nest 上，自动决定 host 与存内计算之间的算子放置；KernelBraid 引入 optimization IR 支撑 LLM agent 对统一 RL 内核做有状态且 bitwise 一致的搜索优化；F2Asm 则以 F₂ 线性代数从反汇编与原始 CUBIN 配对中学习精确的 NVIDIA SASS 编码器，并首次支持 Rubin SM107。三者共同显示，自动化正从算子放置、内核搜索一路延伸到指令编码这一底层环节。

## 重点变化

### 1. Torch-PIM: Automated Profile-Guided PIM Offloading for PyTorch

Torch-PIM 在 PyTorch 的 progressive lowering 管线之上引入 profile-guided 决策：每个物化出的并行 loop nest 都进入 host 与 PIM 放置的候选空间，按工作量和 memory boundedness 两阶段评估，评估所需数据来自 host 端 profiling 或 MLIR，从而突破以往在 lowering 之前固定算子候选集的限制。在 32 至 128 核的 PIM 配置下，相对 CPU-only 执行，其卸载决策在 tensor operators、MLP、Attention、GPT-J-6B 和 LLaMA-7B 上分别取得最高 8.6x、2.9x、4.4x、5.1x 和 3.6x 加速。

**为什么重要：** 把卸载决策推迟到 lowering 之后并以实际物化的 loop nest 为决策对象，能让编译器看到更细粒度的并行结构，用真实 profiling 数据驱动放置，这对存内计算这类对数据移动成本敏感的异构后端尤为关键。其在主流模型算子上的加速说明 PGO 驱动的自动卸载具备实用潜力，但数字基于 CPU-only 基线，泛化性仍需更多硬件与对比实验验证。

**涉及层级：** `tensor_ir`、`optimization`、`hardware_backend`、`benchmark`

**来源：** [原始来源 1](https://arxiv.org/abs/2609.34657v1)

**事件评分：** 76.5/100

### 2. AReaL-TIK: Stateful Agentic Optimization of Unified RL Kernels through an Optimization IR

KernelBraid 以 hand-tuned、bitwise-consistent 的统一 RL 内核实现为起点，提出 optimization IR 把实现与修改关联到数值要求、workload 测量和推导历史，由 LLM agent 在此之上进行有状态的源代码搜索并保留已验证的中间结果；修改只有在通过 bitwise 正确性检查且改善 aggregate latency 时才会被采纳。在 H20 上 12 个端到端训练配置的平均吞吐为 AReaL（log-probability recomputation）的 1.10x，隔离层 profiling 在 15 个 model-GPU 对上取得 1.40x 平均 phase time 加速，unified-attention 搜索用 7M LLM tokens 取得 2.52x 加速，10 个算子在 A100/H20/H200 上均通过 bitwise 检查。

**为什么重要：** agentic 代码优化的常见痛点是搜索噪声大、验证成本高。KernelBraid 用 optimization IR 把数值约束、测量数据和推导历史结构化为可检索证据，使 agent 能在保证 bitwise 一致的前提下做增量式优化，为 RL 训练内核这类数值敏感又性能敏感的代码提供了可复现的自动化路径。不过端到端训练收益相对保守，说明隔离层加速尚未完全转化为训练吞吐。

**涉及层级：** `optimization`、`kernel_codegen`、`hardware_backend`、`benchmark`

**来源：** [原始来源 1](https://arxiv.org/abs/2609.35140v1)

**事件评分：** 72.0/100

### 3. Learning Exact NVIDIA SASS Encoders with $\mathbb{F}_2$ Linear Algebra

F2Asm 将 NVIDIA SASS 指令编码器建模为 F₂ 上的向量值仿射映射，从反汇编与原始 CUBIN 指令字配对中学习精确的 128 位编码器，用 F₂ 上的高斯消元增量构建紧凑基、检测不一致并拒绝超出学习范围的输入。作者使用 3,225 个 CUBIN 训练 Hopper SM90/SM90a、Blackwell SM100 和 Rubin SM107 编码器，反汇编-重汇编往返测试中可执行文本段与原始完全一致；联合训练得到覆盖五个 Blackwell SM 目标与三个 Rubin SM 目标的共享编码器，持续训练以 1,504 个新增基行将 Rubin 编码器扩展到 17,159 个此前不支持的 cuTile/GROMACS 查询。

**为什么重要：** NVIDIA 长期只提供 SASS 反汇编器而没有公开汇编器，限制了受控机器码改写、内核修补与底层性能研究。F2Asm 首次把编码学习形式化为 F₂ 线性代数问题，并成为首个支持 Rubin SM107 的开源 SASS 汇编器，其往返一致的还原能力与跨 SM 目标共享编码器的发现为反向工程和底层工具链提供了更严谨的方法。需要留意的是，学习式编码器受限于已学到的基空间，遇到新编码时仍需持续训练扩展。

**涉及层级：** `kernel_codegen`、`hardware_backend`

**来源：** [原始来源 1](https://arxiv.org/abs/2608.20532v3)

**事件评分：** 61.6/100

## 值得继续观察

- Torch-PIM
- PIM 卸载
- KernelBraid
- AReaL
- F2Asm
- SASS 汇编器
- Rubin SM107

## 数据与生成说明

- 采集窗口：`2026-09-28T08:00:00+08:00` 至 `2026-09-29T08:00:00+08:00`
- 生成模式：`deepseek`
- 文章由程序根据一手来源生成；性能结论应以链接中的原始配置为准。
- 原始条目：NVIDIA 3 条，arXiv 3 条
