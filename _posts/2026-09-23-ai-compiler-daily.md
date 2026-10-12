---
layout: default
categories: [ai-compiler, daily]
title: "vLLM 引入硬件无关层，为 OOT 加速器保留全图编译路径"
date: 2026-09-23
generation_mode: "deepseek"
window_since: "2026-09-22T08:00:00+08:00"
cutoff_at: "2026-09-23T08:00:00+08:00"
---

# vLLM 引入硬件无关层，为 OOT 加速器保留全图编译路径

## 今日概览

本期聚焦 vLLM 的一次重要架构调整：项目在转向硬件特定 flat 模型定义的同时，新增了一组硬件无关层，以维持 OOT 加速器经由 torch.compile 获得性能的路径，并在 NVIDIA H100 上验证了 3.4% 以内的吞吐差距。

## 重点变化

### 1. Hardware-Agnostic Models in vLLM

vLLM 正在将其模型定义迁移到硬件特定的 flat 模型，这类模型使用自定义融合算子及其他硬件专属优化，并放弃 fullgraph torch.compile 兼容性。为应对由此带来的兼容性风险，项目在 model_executor/hw_agnostic 下新增了一组硬件无关层，遵循可编译、可扩展、隔离、可移植四项设计原则，使 IBM Spyre 等 OOT 加速器仍可通过 TorchDynamo tracing 与 TorchInductor lowering 获得性能。transformers 后端的 rewiring 已改为指向这些新层，可通过 USE_HW_AGNOSTIC=1 启用，并在 Gemma 4、Qwen3、Granite 4.2 等模型上借助 Spyre 插件完成验证。在 NVIDIA H100 上，硬件无关层的 token 吞吐量与原生实现差距在 3.4% 以内。后续计划为每个 flat 模型提供基于这些层的 model.py，模型专属层如 DeepSeekV4FlashMLAAttention 也将遵循同样原则。

**为什么重要：** 这标志着 vLLM 在硬件特化与可移植性之间的重新权衡。flat 模型以自定义融合算子换取性能，但同时放弃了 torch.compile 兼容性和基于 CustomOp 的扩展性，可能将依赖编译优化的非 NVIDIA 加速器排除在外。新的硬件无关层为这类加速器保留了一条全图编译路径，并维持 CustomOp 与 PluggableLayer 的覆盖机制，避免了 OOT 硬件在 vLLM 演进中被边缘化。

**涉及层级：** `frontend`、`optimization`、`kernel_codegen`、`hardware_backend`、`runtime`、`benchmark`

**来源：** [原始来源 1](https://pytorch.org/blog/hardware-agnostic-models-in-vllm/)

**事件评分：** 76.2/100

## 值得继续观察

- vLLM 为每个 flat 模型提供基于 HW agnostic 层的 model.py 的落地进展
- model_executor/hw_agnostic 层覆盖范围的扩展及与原生实现的性能差距变化
- IBM Spyre 等 OOT 插件在新路径上的持续性能验证

## 数据与生成说明

- 采集窗口：`2026-09-22T08:00:00+08:00` 至 `2026-09-23T08:00:00+08:00`
- 生成模式：`deepseek`
- 文章由程序根据一手来源生成；性能结论应以链接中的原始配置为准。
- 原始条目：NVIDIA 4 条，PyTorch 2 条，llvm/llvm-project 1 条
