---
layout: default
categories: [ai-compiler, daily]
title: "IREE v3.12.0 发布 HAL 0.7 破坏性升级并新增多款硬件目标；PyTorch 加速器工作组推进跨仓库 CI 与 PrivateUse1 参考后端"
date: 2026-10-06
generation_mode: "deepseek"
window_since: "2026-10-05T08:00:00+08:00"
cutoff_at: "2026-10-06T08:00:00+08:00"
---

# IREE v3.12.0 发布 HAL 0.7 破坏性升级并新增多款硬件目标；PyTorch 加速器工作组推进跨仓库 CI 与 PrivateUse1 参考后端

## 今日概览

2026 年 10 月 6 日：IREE 发布 v3.12.0，这是一次带破坏性变更的大版本——HAL 模块升至 0.7 使旧 VMFB 失效，codegen pipeline 名称改为 backend attribute 语法，同时重写 Vulkan 驱动、新增 WebGPU、NVIDIA Ada/Blackwell、AMD CDNA5/RDNA3 等目标，并为 ROCm 引入 skinny GEMM 优化。PyTorch 加速器集成工作组公布 2026 H1 进展，通过跨仓库 CI Relay、去设备化测试重构与 hw_classification 分类、OpenReg profiler stub 等基础设施，降低第三方加速器接入门槛。

## 重点变化

### 1. Release v3.12.0

IREE 发布 v3.12.0，累计约 720 个 PR。HAL 模块版本升至 0.7，使旧 VMFB 与 3.12 runtime 不兼容，需要重新编译；信号量系统改为由 async proactor 驱动的共享 iree_async_semaphore_t；Vulkan HAL 驱动重写并要求 Vulkan 1.3（timelineSemaphore、scalarBlockLayout、synchronization2）。codegen pipeline 改用 backend attribute 命名（如 #iree_gpu.pipeline<TileAndFuse>、#iree_cpu.pipeline<Default>），旧关键字不再解析。新增 WebGPU WGSL 目标（经 SPIR-V 与 Tint，通过 IREE_TARGET_BACKEND_WEBGPU_SPIRV 开启）、NVIDIA Ada (sm_89) 与 Blackwell (sm_120/sm_121)、AMD CDNA5/MI455X 与 RDNA3，以及 Broadcom VideoCore VII、Arm Mali 5th-gen、llvmpipe 等 Vulkan 目标。ROCm codegen 为 M≤8 的 skinny GEMM 引入虚拟稠密 MFMA，在 gfx950 上最高约 24% 提速，并在 gfx950 默认启用 F16/BF16 GEMM 的 DMA；iree-dialects 项目与旧 transform.iree.match_callback 风格 ops 被移除。

**为什么重要：** 这是一次破坏性版本升级：HAL 0.7 的 VMFB 不兼容意味着已部署模型包需重新编译，pipeline 关键字改为 attribute 语法也要求用户更新编译脚本，迁移成本集中在前端与集成环节。同时硬件目标大幅扩展——WebGPU、Blackwell、CDNA5 等扩大了可部署范围——而 MFMA 与 DMA 默认开启则为 ROCm 平台上的 GEMM 工作负载带来实际性能提升。

**涉及层级：** `frontend`、`optimization`、`kernel_codegen`、`hardware_backend`、`runtime`

**来源：** [原始来源 1](https://github.com/iree-org/iree/releases/tag/v3.12.0)

**事件评分：** 90.8/100

### 2. PyTorch Hardware Enablement: Updates from the Accelerator Integration Working Group

PyTorch 加速器集成工作组发布 2026 H1 进展：引入跨仓库 CI Relay（CRCR），通过 webhook、GitHub OIDC 回调与分层 allowlist 回传跨仓库 CI 状态；测试套件去设备化重构迁移了 276+ 测试文件，并落地 hw_classification 分类元数据（GENERIC、DEVICE_GENERIC、CUDA、XPU、MPS）、-hw-classification 标志与 HW_CLASSIFICATION linter 护栏；基于 REGISTER_PRIVATEUSE1_PROFILER 实现 OpenReg 参考 profiler stub 栈，覆盖 session 生命周期、activity 类型与 correlation ID；整体工作流明确包含 compiler backend integration，OpenReg 作为 PyTorch 树内 PrivateUse1 参考后端持续扩展 device registration、operator dispatch、streams、events 等集成模式。

**为什么重要：** 对第三方加速器厂商而言，这些基础设施降低了把私有后端接入 PyTorch 生态的成本：CRCR 打通跨仓库 CI 状态可见性，hw_classification 让测试按硬件自动分派，OpenReg profiler stub 提供了可复制的 Kineto 集成范例，而 compiler backend integration 工作流则为后端编译器如何与 PyTorch 协同指明了路径。

**涉及层级：** `hardware_backend`、`runtime`

**来源：** [原始来源 1](https://pytorch.org/blog/pytorch-hardware-enablement-updates-from-the-acceleration-integration-working-group/)

**事件评分：** 62.7/100

## 值得继续观察

- IREE v3.12：HAL 0.7 导致旧 VMFB 不兼容，需重新编译
- IREE pipeline 关键字改为 backend attribute 语法
- Vulkan HAL 重写并要求 Vulkan 1.3
- IREE WebGPU WGSL 目标（SPIR-V + Tint，opt-in）
- ROCm skinny GEMM 虚拟稠密 MFMA 与 F16/BF16 DMA 默认开启
- PyTorch CRCR 跨仓库 CI 与 hw_classification 测试分类
- OpenReg PrivateUse1 profiler stub 参考实现

## 数据与生成说明

- 采集窗口：`2026-10-05T08:00:00+08:00` 至 `2026-10-06T08:00:00+08:00`
- 生成模式：`deepseek`
- 文章由程序根据一手来源生成；性能结论应以链接中的原始配置为准。
- 原始条目：LLVM 1 条，PyTorch 2 条，arXiv 4 条，iree-org/iree 1 条
