---
layout: default
categories: [ai-compiler, daily]
title: "ExecuTorch-TensorRT 达到生产级：PyTorch 机器人策略以接近 TensorRT 的速度跑在 Jetson 上"
date: 2026-11-25
generation_mode: "deepseek"
window_since: "2026-11-24T08:00:00+08:00"
cutoff_at: "2026-11-25T08:00:00+08:00"
---

# ExecuTorch-TensorRT 达到生产级：PyTorch 机器人策略以接近 TensorRT 的速度跑在 Jetson 上

## 今日概览

11 月 25 日的重点是 PyTorch 生态在边缘部署上的更新。Torch-TensorRT 2.15 与 ExecuTorch 1.6 推出生产级 ExecuTorch-TensorRT 工作流：模型用 torch.export 捕获而非经过 ONNX，TensorRT 编译可转换的算子，其余部分由 ExecuTorch CUDA backend（AOTInductor）编译，最终生成无需 libtorch 即可运行的 .pte 程序；在 NVIDIA Jetson Orin Nano 上其性能与最快的 TensorRT 方案相差 4% 以内。

## 重点变化

### 1. ExecuTorch-TensorRT: PyTorch Robot Policies at NVIDIA TensorRT Speed

Torch-TensorRT 2.15 与 ExecuTorch 1.6 带来了生产级 ExecuTorch-TensorRT 工作流。它跳过 ONNX，直接用 torch.export 捕获 PyTorch 模型，并由 TensorRT 编译可转换部分；对于 TensorRT 无法转换的算子，则交给 ExecuTorch CUDA backend 用 AOTInductor 编译，两者在同一程序、同一 CUDA stream 上运行，最终产出无需 libtorch 即可从 C++ 或 Python 运行的 .pte 程序。该工作流新增不依赖 PyTorch 的 Python runtime、经 torch.library.triton_op 注册的 Triton 自定义算子编译，以及 use_cuda_graphs=True 时将 engine 的 GPU 工作录制为 CUDA graph 并单次 launch 重放；输入输出和 backend 间张量可保持 GPU 驻留，无需经 CPU 往返。在 NVIDIA Jetson Orin Nano 上，其性能与最快的 TensorRT 方案相差 4% 以内，3.6B 参数的策略能在 8GB 板上运行，而 PyTorch eager 模式会内存耗尽。

**为什么重要：** 这一变化把 TensorRT 的算子优化与 ExecuTorch 的端侧运行时组合起来，扭转了此前算子无法被 TensorRT 转换就构建失败的处境，使 PyTorch 模型能以接近纯 TensorRT 的性能部署到 Jetson 这类边缘设备，为机器人策略等大模型上板提供了可行路径，并显著降低了部署端对 PyTorch 运行时的依赖。

**涉及层级：** `frontend`、`graph_ir`、`optimization`、`kernel_codegen`、`hardware_backend`、`runtime`、`benchmark`

**来源：** [原始来源 1](https://pytorch.org/blog/executorch-tensorrt-pytorch-robot-policies-at-nvidia-tensorrt-speed/)

**事件评分：** 78.7/100

## 值得继续观察

- ExecuTorch 1.6
- Torch-TensorRT 2.15
- 算子级分区与 CUDA graph 重放
- Triton 自定义算子编译

## 数据与生成说明

- 采集窗口：`2026-11-24T08:00:00+08:00` 至 `2026-11-25T08:00:00+08:00`
- 生成模式：`deepseek`
- 文章由程序根据一手来源生成；性能结论应以链接中的原始配置为准。
- 原始条目：NVIDIA 4 条，PyTorch 1 条
