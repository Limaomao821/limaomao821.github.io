---
layout: default
categories: [ai-compiler, daily]
title: "vLLM 新加坡 Meetup：更快 prefill、AI agent 手写 kernel 与 Vera Rubin 首日支持"
date: 2026-11-26
generation_mode: "deepseek"
window_since: "2026-11-25T08:00:00+08:00"
cutoff_at: "2026-11-26T08:00:00+08:00"
---

# vLLM 新加坡 Meetup：更快 prefill、AI agent 手写 kernel 与 Vera Rubin 首日支持

## 今日概览

2026-11-26 的 vLLM 新加坡 meetup 展示了开源推理栈多个方向的进展：pipeline parallelism 让 prefill 更快且无需手工调优，FlashInfer 针对 SM12x 的专用 kernel 与 dispatch 修复带来吞吐提升，vLLM-Omni 的 AR-Diffusion 引擎实现视频实时生成，AI agent loop 能以低成本生成 bit-exact kernel 支撑无 clipping 的异步 RL，而 vLLM 已在 NVIDIA Vera Rubin 上实现首日运行并探索 locality-domain 优化。

## 重点变化

### 1. vLLM Singapore Meetup: Faster Prefill, Agent-Written Kernels and the Road to Vera Rubin

本次 meetup 覆盖 vLLM 的 pipeline parallelism（PP）prefill 优化：按实测时间而非层数平衡 stage、限制 step size 以缩短 pipeline fill/drain、将 activation 传输移至 copy engines（p50 TTFT 降低 13–20% 并在 NVLink 上默认开启），以及无需调优即可匹配最佳 step size 的 PP-aware scheduler。FlashInfer 为 SM12x（RTX PRO / DGX Spark）新增专用 kernel，并借 dispatch 修复（+17.7% 吞吐）和 NVFP4 线性层改用 W4A4（+23.8%）提升性能。vLLM-Omni 新增 AR-Diffusion 引擎，复用 paged KV cache 逐 chunk 生成视频，将 5.17 秒视频生成从 213.4 秒降至 4.92 秒。此外，通过 agent loop 生成与 rollout 逐层 bit-for-bit 一致的 exact kernel，成本约为 rollout 速度的 10%，消除了 RL 训练中的 KL mismatch；面向 Vera Rubin 的 locality-domain 感知 kernel（如 double-K MMA）与 MoE expert 本地化使 memory-bound decode kernel 提升 8–29%。

**为什么重要：** 这些进展表明开源推理社区在多个瓶颈上同时推进：PP 优化让多卡 prefill 在接近饱和时显著降低延迟并默认可用，agent 生成 exact kernel 使异步 RL 能以约 2.4 倍速度更新且无需 clipping，AR-Diffusion 让视频世界模型达到实时交互，而 Vera Rubin 的首日支持与 locality 优化为下一代硬件提供了性能路线图。

**涉及层级：** `optimization`、`kernel_codegen`、`hardware_backend`、`distributed`、`runtime`、`benchmark`

**来源：** [原始来源 1](https://pytorch.org/blog/vllm-singapore-meetup-faster-prefill-agent-written-kernels-and-the-road-to-vera-rubin/)

**事件评分：** 73.4/100

## 值得继续观察

- vLLM pipeline parallelism prefill 优化
- FlashInfer SM12x 专用 kernel 与 dispatch
- AR-Diffusion 实时视频生成
- AI agent 生成 bit-exact kernel
- NVIDIA Vera Rubin locality-domain 优化
- REINFORCE Zero 异步 RL

## 数据与生成说明

- 采集窗口：`2026-11-25T08:00:00+08:00` 至 `2026-11-26T08:00:00+08:00`
- 生成模式：`deepseek`
- 文章由程序根据一手来源生成；性能结论应以链接中的原始配置为准。
- 原始条目：NVIDIA 1 条，PyTorch 1 条，arXiv 2 条
