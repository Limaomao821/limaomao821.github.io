---
layout: default
categories: [ai-compiler, daily]
title: "Compile-Time Heuristics Make Helion Kernels Fast by Default; SGLang Reaches IBM Spyre in Under a Week"
date: 2026-11-13
generation_mode: "deepseek"
window_since: "2026-11-12T08:00:00+08:00"
cutoff_at: "2026-11-13T08:00:00+08:00"
---

# Compile-Time Heuristics Make Helion Kernels Fast by Default; SGLang Reaches IBM Spyre in Under a Week

## 今日概览

Two practical advances in ML compiler and runtime portability stand out today. PyTorch Helion now derives kernel launch configurations directly from IR and dataflow analysis at compile time, classifying kernels as matmul, reduction, or pointwise and emitting block size, warp count, and pipeline stage settings without per-kernel-per-shape autotuning; reported H100 numbers beat hand-written Triton and vLLM CUDA kernels while seeding autotuning when it is enabled. Separately, IBM brought SGLang to its Spyre accelerator in under a week with roughly 1,500 lines of plugin code and AI coding agent assistance, registering Spyre as a non-Triton attention backend and reusing operator implementations from the vLLM Spyre integration to deliver roughly 3.6x the throughput of the torch-spyre reference implementation.

## 重点变化

### 1. Fast by Default: Compile-Time Heuristics for Helion Kernels

PyTorch Helion adds compile-time heuristics that analyze a kernel's IR and dataflow to pick launch settings without autotuning. The system classifies kernels as matmul, reduction, or pointwise and routes each to a corresponding heuristic, backed by liveness analysis and resource estimation for registers, shared memory, and Blackwell tensor memory to infer occupancy and register pressure. Matmul kernels get block size, num_warps, and num_stages from dynamic work and instruction selection (wgmma or tcgen05), with conflicting multi-dot kernels ranked by loop-carried accumulators, dynamic work, and output area. Reduction kernels add persistence reasoning, such as keeping rows resident in RMSNorm to avoid a second memory pass. On H100 the computed configs beat hand-written Triton Flash Linear Attention by 1.18x forward, outperform Helion's previous no-autotune default by 2.13x, and beat vLLM's handwritten CUDA/C++ kernels by 1.27x across production kernels. When autotuning is enabled, helper heuristics generate 5-20 seed configs, and kernels with unknown extents fall back to the old default.

**为什么重要：** Default compilation paths in ML kernel DSLs have often relied on autotuning, which costs search time and can still miss good configs. Emitting near-autotuned settings directly from IR analysis makes the no-autotuning path competitive by default, and seeding the autotuner with 5-20 heuristic configs can shorten searches when tuning is warranted. The approach also illustrates how compiler-level resource estimation and liveness analysis, rather than runtime trials, can drive occupancy and register pressure decisions on modern hardware such as Hopper and Blackwell.

**涉及层级：** `frontend`、`optimization`、`kernel_codegen`、`hardware_backend`、`benchmark`

**来源：** [原始来源 1](https://pytorch.org/blog/fast-by-default-compile-time-heuristics-for-helion-kernels/)

**事件评分：** 83.7/100

### 2. Integrating SGLang into Spyre in Under a Week with AI

IBM integrated SGLang into its Spyre accelerator in under a week using AI coding agents, producing roughly a 1,500-line plugin built on torch-spyre, which registers Spyre as a PyTorch device connected to torch.compile. Four patches made SGLang's execution paths available without CUDA-specific dependencies and registered Spyre as a supported device with an attention backend that does not use Triton. Existing Spyre implementations for RMSNorm, SiLU, GeLU, rotary embeddings, and linear layers were connected to SGLang's operator dispatch, and attention code and KV cache allocation were reused from the vLLM Spyre integration. SGLang delivered about 3.6x the throughput of the torch-spyre reference inference implementation and was on par with vLLM on measured workloads for Granite 3.3 8B and Gemma 4 26B A4B.

**为什么重要：** This port shows how a serving framework can reach a non-CUDA accelerator quickly when device abstraction is already in place through torch.compile and when an existing integration, here vLLM on Spyre, provides reusable operator implementations. It also demonstrates AI coding agents accelerating accelerator bring-up, and the device-agnostic dispatch changes may help other accelerators adopt SGLang while the registration work remains Spyre-specific.

**涉及层级：** `hardware_backend`、`runtime`、`benchmark`

**来源：** [原始来源 1](https://pytorch.org/blog/integrating-sglang-into-spyre-in-under-a-week-with-ai/)

**事件评分：** 61.4/100

## 值得继续观察

- PyTorch Helion compile-time config heuristics and whether they extend beyond matmul, reduction, and pointwise kernels
- Helion heuristic-seeded autotuning compared with traditional autotuning search
- Non-CUDA and non-Triton attention backends in SGLang for accelerators like IBM Spyre
- AI-assisted porting workflows for accelerator bring-up

## 数据与生成说明

- 采集窗口：`2026-11-12T08:00:00+08:00` 至 `2026-11-13T08:00:00+08:00`
- 生成模式：`deepseek`
- 文章由程序根据一手来源生成；性能结论应以链接中的原始配置为准。
- 原始条目：PyTorch 3 条，arXiv 3 条，cuTile 1 条
