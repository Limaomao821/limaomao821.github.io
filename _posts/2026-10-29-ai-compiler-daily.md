---
layout: default
categories: [ai-compiler, daily]
title: "PyTorch 2.15 发布：NVGEMM 加速 Blackwell NVFP4 解码，NCCL 重写后端转正、Inductor 接入 AMD FlyDSL"
date: 2026-10-29
generation_mode: "deepseek"
window_since: "2026-10-28T08:00:00+08:00"
cutoff_at: "2026-10-29T08:00:00+08:00"
---

# PyTorch 2.15 发布：NVGEMM 加速 Blackwell NVFP4 解码，NCCL 重写后端转正、Inductor 接入 AMD FlyDSL

## 今日概览

今日焦点是 PyTorch 2.15 发布，发布说明与官方博客两个事件从不同侧面呈现同一个版本。编译侧，NVGEMM 将 NVFP4 反量化因子折叠进 GEMM 并启用 programmatic dependent launch 以加速 Blackwell 4-bit LLM 解码；Inductor 在 AMD gfx950/gfx1250 上自动调优 FlyDSL 与 Tensor Data Mover；Apple Silicon 新增 FP8 与带解码快速路径的 grouped_mm；Dynamo 支持 Python 3.15。语义侧，AOTAutograd 将 effectful operator 的 MUST_RECOMPUTE 策略改为编译时报错。分布式侧，重写版 NCCL 后端成为默认并配备 flight recorder，新增 NIXL one-sided transports，FSDP2 获得 CUDA graph 捕获与通信-计算重叠能力。

## 重点变化

### 1. PyTorch 2.15.0 Release

PyTorch 2.15.0 发布说明横跨前端、图 IR、内核代码生成与运行时各层：NVGEMM 通过更少、重叠更好的 kernel launches 加速 NVIDIA Blackwell 上 4-bit（NVFP4）LLM 解码；Apple Silicon 新增 float8_e4m3fn 与 _scaled_mm 的 FP8 支持，并为 MoE 解码引入带 gemv 快速路径的 grouped_mm（报告 1.32–1.34x tokens/s）。平台支持扩展到 AMD gfx950（FlyDSL kernels）、gfx1250/MI450（Tensor Data Mover）、ROCm 10.0 wheels、Intel GPU 的 SYCL kernel 编写与多 GPU profiling，以及 CUDA 13.2/13.4 wheels 和 Python 3.15 上的 torch.compile。语义层面，AOTAutograd 对 effectful operator 的 CheckpointPolicy.MUST_RECOMPUTE 改为编译时报错。分布式方面，重写的 NCCL backend 成为默认并提供 flight recorder 与 nccl-legacy 回退，同时引入 NIXL one-sided transports 与 CUDA-graph-capturable transfers；FSDP2 与流水线并行获得更细粒度的显存与通信控制。

**为什么重要：** 这是一次同时改动前端、codegen、runtime 与 distributed 的发布。对编译器使用者最直接的影响是 MUST_RECOMPUTE 的行为变更：effectful operator 的前向副作用无法接入反向 effect-token 链，2.15 将原本可能静默出错的重计算变成确定的编译时报错，把正确性问题前移到编译期。NCCL 后端的默认切换加上 flight recorder，则让分布式集合通信挂起从难以排查走向可事后诊断。

**涉及层级：** `frontend`、`graph_ir`、`kernel_codegen`、`hardware_backend`、`runtime`、`distributed`、`benchmark`

**来源：** [原始来源 1](https://github.com/pytorch/pytorch/releases/tag/v2.15.0)

**事件评分：** 92.3/100

### 2. PyTorch 2.15 Release Blog

官方发布博客从编译器视角展开同一版本：NVGEMM（CuTeDSL GEMM 后端）把 ModelOpt NVFP4 linear 层的标量反量化因子折叠进 GEMM，消除解码期每层一次 pointwise 启动，通过有界、shape-aware 候选策略覆盖 nvMatmulHeuristics 遗漏的 Blackwell NVFP4 解码形状，并用 programmatic dependent launch 让 Triton 消费者与 GEMM 尾部重叠，整体迁移到 CUTLASS Operators。Inductor 在 AMD gfx950 上自动调优 FlyDSL 模板（GEMM、ragged F.grouped_mm、MXFP4/MXFP8 scaled_mm），在 gfx1250/MI450 上用 Tensor Data Mover 生成 dense mm/addmm、pointwise、reduction 与 FlexAttention 代码；Apple Silicon MPS 获得 FP8/_scaled_mm、gemv 解码路径、grouped_mm 与公开 Stream API；Intel XPU 引入 SYCL free-function 内核编程模型；Dynamo 支持 Python 3.15。重写版 NCCL 成为所有 nccl process group 的默认后端并带 backend-agnostic flight recorder，FSDP2 可将完整梯度累积步捕获进单个 CUDA graph。

**为什么重要：** 博客揭示的关键趋势是 PyTorch 代码生成从以 Triton 为中心转向多 DSL 并存：CuTeDSL、FlyDSL 与 Tensor Data Mover 都已进入 Inductor 的自动调优或代码生成路径，编译栈开始按厂商硬件特性分化。对跟踪编译器演进的人而言，这意味着内核后端竞争格局正在变化，跨厂商对比 torch.compile 性能时需要更仔细地核对所用后端配置。

**涉及层级：** `frontend`、`optimization`、`kernel_codegen`、`hardware_backend`、`runtime`、`distributed`、`benchmark`

**来源：** [原始来源 1](https://pytorch.org/blog/pytorch-2-15-release-blog/)

**事件评分：** 82.2/100

## 值得继续观察

- PyTorch NVGEMM（CuTeDSL）在 Blackwell NVFP4 解码上的后续优化
- AMD FlyDSL 与 Tensor Data Mover 在 Inductor 中的覆盖扩展
- 重写版 NCCL backend 的稳定性与 flight recorder 实战表现
- AOTAutograd 对 effectful operator 重计算策略的收紧
- Apple Silicon MPS FP8 与 grouped_mm 的生态落地
- Intel XPU SYCL free-function 内核与 XPU torch.compile 支持

## 数据与生成说明

- 采集窗口：`2026-10-28T08:00:00+08:00` 至 `2026-10-29T08:00:00+08:00`
- 生成模式：`deepseek`
- 文章由程序根据一手来源生成；性能结论应以链接中的原始配置为准。
- 原始条目：PyTorch 1 条，pytorch/pytorch 1 条
