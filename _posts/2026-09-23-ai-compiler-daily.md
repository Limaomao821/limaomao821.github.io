---
layout: default
categories: [ai-compiler, daily]
title: "vLLM 引入树内硬件无关层，前沿 flat 模型与 torch.compile 路径正式分流"
date: 2026-09-23
generation_mode: "deepseek"
window_since: "2026-09-22T08:00:00+08:00"
cutoff_at: "2026-09-23T08:00:00+08:00"
---

# vLLM 引入树内硬件无关层，前沿 flat 模型与 torch.compile 路径正式分流

## 今日概览

2026 年 9 月 23 日，vLLM 公布并合入树内硬件无关层（model_executor/hw_agnostic），目标是在前沿 flat 模型放弃 torch.compile、转向硬件特定融合与内核的背景下，继续为 OOT 加速器与较老 GPU 保留 fullgraph torch.compile 可编译路径。新层遵循 Compilable、Extensible、Isolated、Portable 四原则，transformers 后端重接线已指向该层并可用 USE_HW_AGNOSTIC=1 启用，在 H100 上三模型几何平均总 token 吞吐与原生实现差距控制在 3.4% 以内。

> ⚠️ 本期数据不完整：
>
> - 数据源 arxiv-ai-compiler 采集失败：HTTP 406 while requesting export.arxiv.org

## 重点变化

### 1. Hardware-Agnostic Models in vLLM

vLLM 在其主干分支引入树内硬件无关层（model_executor/hw_agnostic），以保持 fullgraph torch.compile 的可编译性并服务 OOT 加速器与老 GPU。新层遵循可编译（全图 torch.compile）、可扩展（CustomOp/PluggableLayer）、隔离、可移植（原生 PyTorch 或 Triton/Helion）四项原则；transformers 后端的重接线已改指向该层，通过 USE_HW_AGNOSTIC=1 启用，并已在 IBM Spyre 等插件上验证 Gemma 4、Qwen3、Granite 4.2 等模型。与此同时，追求极致性能的前沿 flat 模型正放弃 torch.compile，改用模型与硬件特定的融合及内核，现有层未来可能被重构为与 torch.compile 不兼容。在 H100 上，三模型几何平均总 token 吞吐与原生实现差距在 3.4% 以内。

**为什么重要：** 这标志着 vLLM 内部出现两条分化路径：前沿模型走向定制内核，而可移植性继续依赖 torch.compile。对编译器生态而言，TorchDynamo/TorchInductor 在 OOT 加速器（如 IBM Spyre）路径上仍是关键组件；新层能否长期维持 fullgraph 可编译性并控制与原生实现的性能差距，将直接影响非英伟达硬件接近原生吞吐的难度，也决定了依赖 torch.compile 做图追踪与下放（lowering）的第三方插件能否跟上 vLLM 演进。

**涉及层级：** `frontend`、`optimization`、`kernel_codegen`、`hardware_backend`、`runtime`、`benchmark`

**来源：** [原始来源 1](https://pytorch.org/blog/hardware-agnostic-models-in-vllm/)

**事件评分：** 80.2/100

## 值得继续观察

- vLLM model_executor/hw_agnostic 层的覆盖范围与相对原生实现的性能差距
- flat 模型重构对既有层及 torch.compile 兼容性的后续影响
- IBM Spyre 等 OOT 插件在新路径上的适配进展
- USE_HW_AGNOSTIC=1 在更多模型与硬件上的验证结果

## 数据与生成说明

- 采集窗口：`2026-09-22T08:00:00+08:00` 至 `2026-09-23T08:00:00+08:00`
- 生成模式：`deepseek`
- 文章由程序根据一手来源生成；性能结论应以链接中的原始配置为准。
- 原始条目：NVIDIA 4 条，PyTorch 2 条，llvm/llvm-project 1 条
