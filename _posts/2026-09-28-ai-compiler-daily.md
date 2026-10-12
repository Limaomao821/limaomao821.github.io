---
layout: default
categories: [ai-compiler, daily]
title: "KernelZero 用 Proposer 与 Coder 协同进化生成 GPU kernel，7B 模型在 CUDA 与 Triton 上超越更大的前沿模型"
date: 2026-09-28
generation_mode: "deepseek"
window_since: "2026-09-27T08:00:00+08:00"
cutoff_at: "2026-09-28T08:00:00+08:00"
---

# KernelZero 用 Proposer 与 Coder 协同进化生成 GPU kernel，7B 模型在 CUDA 与 Triton 上超越更大的前沿模型

## 今日概览

本期关注 KernelZero 框架：它让一个 Proposer 模型生成 Torch modules，再由一个 Coder 模型翻译为 CUDA 或 Triton kernel，并以前沿驱动的模块生成和正确性感知的群组相对策略优化（CA-GRPO）交替训练两者，形成自动课程。KernelZero-7B 在 CUDA 上超越 Claude-4.5-Sonnet，在 Triton 上超越 DeepSeek-V4-Pro，KernelBench 一级和二级 CUDA pass@1 分别达 75.8% 和 69.6%。

## 重点变化

### 1. KernelZero: Co-Evolving Proposer and Coder for Continuously Improved GPU Kernel Generation

KernelZero 提出一个协同进化框架，用两个专用模型持续改进 GPU kernel 生成：Proposer 从 API sets 生成 Torch modules，Coder 将其翻译为 CUDA 或 Triton kernel。它引入 frontier-driven module generation，依据 Coder 当前的弱点持续产出能力对齐的训练模块；CA-GRPO 则只在正确性足够可靠后才优化性能。交替优化 Proposer 与 Coder 形成自动课程，使能力提升更具定向性和训练效率。实验显示 KernelZero-7B 在 CUDA 上超越 Claude-4.5-Sonnet、在 Triton 上超越 DeepSeek-V4-Pro，KernelBench 一级和二级的 CUDA pass@1 分别为 75.8% 和 69.6%，Triton 上为 77.2% 和 72.5%，且 pass@10 接近满分。

**为什么重要：** 一个 7B 规模的小模型在 GPU kernel 生成上超过通用大模型，说明把任务拆成模块生成与底层翻译两个角色、并让两者互为课程地交替训练，是比单纯扩大通用模型更数据高效的编译代码生成路线。先保证正确性、再优化性能的 CA-GRPO 设计也直指 LLM 生成 kernel 的可信度痛点，可能影响未来针对编译器后端的强化学习训练方法。

**涉及层级：** `kernel_codegen`、`optimization`

**来源：** [原始来源 1](https://arxiv.org/abs/2609.33074v1)

**事件评分：** 78.2/100

## 值得继续观察

- KernelZero 论文与代码的后续发布情况
- CA-GRPO 与 frontier-driven 训练机制在更多后端如 ROCm/HIP、MLIR 上的推广
- 小模型 kernel 生成能力在 KernelBench 之外真实工作负载上的验证
- Proposer 与 Coder 双模型协同模式在其他编译代码生成任务中的复用

## 数据与生成说明

- 采集窗口：`2026-09-27T08:00:00+08:00` 至 `2026-09-28T08:00:00+08:00`
- 生成模式：`deepseek`
- 文章由程序根据一手来源生成；性能结论应以链接中的原始配置为准。
- 原始条目：arXiv 1 条
