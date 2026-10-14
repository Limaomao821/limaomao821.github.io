---
layout: default
categories: [ai-compiler, daily]
title: "IBM 将 Spyre 编译器迁至 Inductor 扩展，代码量缩至七分之一；MLIR 新管线借 OpenMP 卸载量子电路"
date: 2026-10-14
generation_mode: "deepseek"
window_since: "2026-10-13T08:00:00+08:00"
cutoff_at: "2026-10-14T08:00:00+08:00"
---

# IBM 将 Spyre 编译器迁至 Inductor 扩展，代码量缩至七分之一；MLIR 新管线借 OpenMP 卸载量子电路

## 今日概览

本期两条进展分别落在传统加速器后端与量子计算编译上。IBM 团队公开了 torch-spyre 从自研 sendnn 栈迁移到扩展 Inductor 的实现细节，通过专用分解、lowering 与挂接在 GraphLowering.codegen() 上的自定义预调度 pass，在 LoopLevelIR 层完成 stick 布局、tiling、work division 与 scratchpad 规划，编译器体量约为原来的七分之一，并正与上游合作形式化相关扩展点。另一项工作 OpenMQPI 提出基于 MLIR 的 OpenMP 扩展，以 QPIR 方言表示量子电路并借助 OpenMP target offloading 将其卸载到量子硬件，在 VQE、QFT 与 Deutsch-Jozsa 的小规模评测中相比 Qiskit 和 CUDA Quantum 各有胜负。

## 重点变化

### 1. Extending Inductor for the IBM Spyre Accelerator

IBM 团队将 torch-spyre 的 torch.compile 后端从自研 sendnn 栈迁移到基于 Inductor 的 backend=inductor：注册 Spyre 专用算子分解与 lowering，并通过 monkey-patch GraphLowering.codegen() 挂接 CustomPreSchedulingPasses，在 LoopLevelIR 层完成 stick 布局分配、tiling、work division 与 scratchpad allocation 等图级优化。由于 Spyre 是 32 核、每核 2MB scratchpad、以 128 字节 stick 为数据移动与计算基本单位的静态数据流加速器，编译器需把索引算术改为按 stick 推理、在 stick 不兼容处插入 restickify 搬移算子、把 stick 维度分配建模为最小化数据搬移成本的全局优化问题，并在编译期静态决定数据搬移与 work division。整体编译器代码量约为原栈的七分之一；对 batched matmul 改变 tiling 使 kernel 时间在不同 batch 与形状下下降 29% 到 69%。

**为什么重要：** 这是一次对 Inductor 扩展性的现实检验：out-of-tree 硬件后端不必再造整套编译栈，就能复用 Dynamo、AOTAutograd 与 Inductor 的图基础设施，显著降低编译器维护成本。文中也坦诚指出 CustomPreSchedulingPasses 目前依赖 monkey-patch 而非一等扩展钩子，与上游合作形式化这些扩展点的进展值得持续关注，因为它将决定此类后端能否长期稳定跟随 PyTorch 演进。

**涉及层级：** `frontend`、`graph_ir`、`optimization`、`kernel_codegen`、`hardware_backend`

**来源：** [原始来源 1](https://pytorch.org/blog/extending-inductor-for-the-ibm-spyre-accelerator/)

**事件评分：** 94.8/100

### 2. OpenMQPI: An MLIR-based Pipeline to Offload Quantum Circuits via OpenMP

OpenMQPI 提出基于 MLIR 的 OpenMP 扩展，引入表示量子电路的轻量方言 QPIR，构建执行量子专用优化并将电路 lowering 为兼容 OpenMP offloading 可执行文件的编译流水线；运行时由 MQSS 插件经 OpenMP target offloading 模型将量子电路卸载至量子硬件。在 5、10、15、20 量子位的评测中，VQE 在 20 量子位耗时 5.18 秒，低于 Qiskit 的 5.83 秒与 CUDA Quantum 的 7.39 秒；QFT 与 Deutsch-Jozsa 的耗时与 Qiskit 相当、低于 CUDA Quantum。

**为什么重要：** 该工作展示了把量子计算接入现有 HPC 卸载生态的一条低摩擦路径：直接复用 OpenMP 成熟的 target offloading 基础设施，而不是另建一套运行时。不过其优势主要体现在小规模、仿真类评测中的 VQE 场景，量子位规模与真实量子硬件上的表现仍有待进一步验证。

**涉及层级：** `frontend`、`graph_ir`、`optimization`、`hardware_backend`、`runtime`、`benchmark`

**来源：** [原始来源 1](https://arxiv.org/abs/2610.18056v1)

**事件评分：** 66.3/100

## 值得继续观察

- PyTorch 上游对 CustomPreSchedulingPasses 等 Inductor 扩展点的形式化进展
- OpenMQPI 在更大规模量子电路及真实量子硬件上的评测结果
- torch-spyre 后端在更多模型与更大 batch 下的性能与稳定性表现

## 数据与生成说明

- 采集窗口：`2026-10-13T08:00:00+08:00` 至 `2026-10-14T08:00:00+08:00`
- 生成模式：`deepseek`
- 文章由程序根据一手来源生成；性能结论应以链接中的原始配置为准。
- 原始条目：NVIDIA 2 条，PyTorch 1 条，arXiv 3 条
