---
layout: default
categories: [ai-compiler, daily]
title: "浏览器 GPU 推理与单主机 816B MoE：ExecuTorch WebGPU 后端与 TPU v7 上的 Pallas megakernel"
date: 2026-11-11
generation_mode: "deepseek"
window_since: "2026-11-10T08:00:00+08:00"
cutoff_at: "2026-11-11T08:00:00+08:00"
---

# 浏览器 GPU 推理与单主机 816B MoE：ExecuTorch WebGPU 后端与 TPU v7 上的 Pallas megakernel

## 今日概览

今日两条进展分别落在推理部署光谱的两端：ExecuTorch 新增 WebGPU 后端，让 PyTorch 模型导出后直接在浏览器 GPU 上运行；PyTorch 团队则展示在单台 TPU v7 主机上以 SGLang-TorchTPU 和 Pallas megakernel 服务 816B 参数的 Muse Spark Open 模型。两者共同点是编译与代码生成层面的优化——前者做跨平台着色器翻译与 Wasm 运行时桥接，后者做整模型 kernel 融合与流水线调度。

## 重点变化

### 1. ExecuTorch WebGPU: PyTorch Models on the GPU, in the Browser

ExecuTorch 推出 WebGPU 硬件后端：PyTorch 模型经 torch.export 导出为 .pte 文件即可在浏览器 GPU 上运行，无需转换到其他框架格式。后端使用 WGSL compute shaders 与 WebGPUPartitioner，动态形状通过 allocate-at-max buffers 与 per-dispatch workgroup 重算支持 batched prefill；单一 .pte 工件可跨平台运行，WGSL 按目标平台翻译为 MSL、SPIR-V 或 HLSL。浏览器部署用 Emscripten 把 C++ 运行时编译为 WebAssembly，由 emdawnwebgpu 桥接浏览器的 WebGPU JavaScript API。在 Apple M1 Pro 的 Chrome 上，prefill 比 ExecuTorch Wasm(XNNPACK) CPU 路径快 4–7 倍，decode 快 2–10 倍，并已覆盖 Llama、Qwen、Gemma 等 LLM 及 SmolVLM、Depth Anything V2、SAM 等视觉模型。

**为什么重要：** 它把 PyTorch 生态直接带入浏览器 GPU 推理，省去框架格式转换，降低客户端与边缘部署摩擦；同时给出一个以单一中间表示、多目标代码生成加 Wasm 桥接为核心的跨平台后端范式，并带有与自身 CPU 路径对照的量化基准，为后续在浏览器端做本地模型服务提供了可复用的工程参考。

**涉及层级：** `frontend`、`kernel_codegen`、`hardware_backend`、`runtime`、`benchmark`

**来源：** [原始来源 1](https://pytorch.org/blog/executorch-webgpu-pytorch-models-on-the-gpu-in-the-browser/)

**事件评分：** 75.8/100

### 2. Optimizing Muse Spark Open on TPU v7 with SGLang-TorchTPU

PyTorch 博客介绍在单台 TPU v7（四芯片）主机上用 SGLang-TorchTPU 服务 Meta 816B 参数 MoE 模型 Muse Spark Open 的优化工作：核心是把 decode/verify 步骤的全部 62 层合入单个 Pallas megakernel，使层间权重加载、KV 读取与跨设备传输全部流水化；配合 collective folding（按切片到达顺序消费 all-gather，避免等待完整 gather）、sequence-on-lane KV 布局与 L2 HiCache（session-aware eviction）、DFlash2 投机解码、fused MoE prefill kernel 与 sequence parallelism。在 AgentX 1M-context 基准上，每请求吞吐从 433 tokens/s（每芯片 6,101 tokens/s）延伸至 25,906 tokens/s（每请求 95 tokens/s）。

**为什么重要：** 它展示编译器层面的 kernel 融合与流水线调度如何消除多 kernel 与多 collective 边界带来的停顿——verify 步骤从原生路径的 2.9 倍权重流式时间降至 1.5–1.7 倍，使 816B MoE 在单主机上可服务。这对分布式大模型推理的软硬件协同设计（megakernel、collective folding、KV cache 布局）提供了可量化的参考点。

**涉及层级：** `optimization`、`kernel_codegen`、`hardware_backend`、`runtime`、`distributed`、`benchmark`

**来源：** [原始来源 1](https://pytorch.org/blog/optimizing-muse-spark-open-on-tpu-v7-with-sglang-torchtpu/)

**事件评分：** 69.5/100

## 值得继续观察

- ExecuTorch WebGPU backend 后续迭代与算子系统覆盖
- WGSL 到 MSL/SPIR-V/HLSL 的跨平台翻译质量
- SGLang-TorchTPU 在 TPU v7 的 Pallas megakernel 路线
- collective folding 与 L2 HiCache 在其他后端的复现

## 数据与生成说明

- 采集窗口：`2026-11-10T08:00:00+08:00` 至 `2026-11-11T08:00:00+08:00`
- 生成模式：`deepseek`
- 文章由程序根据一手来源生成；性能结论应以链接中的原始配置为准。
- 原始条目：NVIDIA 4 条，PyTorch 2 条，arXiv 1 条
