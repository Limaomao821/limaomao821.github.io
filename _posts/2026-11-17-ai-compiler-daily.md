---
layout: default
categories: [ai-compiler, daily]
title: "IREE v3.14.0 发布：量化卷积全面整数化并新增 MoE 专用算子"
date: 2026-11-17
generation_mode: "deepseek"
window_since: "2026-11-16T08:00:00+08:00"
cutoff_at: "2026-11-17T08:00:00+08:00"
---

# IREE v3.14.0 发布：量化卷积全面整数化并新增 MoE 专用算子

## 今日概览

今日编译动态聚焦 IREE v3.14.0 版本发布。量化卷积的 im2col 重写被统一为通用重写并移至 QDQ 转换之后，使卷积完全转为整数计算，4-bit 与 2-bit 常量量化权重改为 i4/i2 紧凑存储；新增 LinalgExt 算子 group_matmul 与 group_mmt4d 以支撑 MoE 模型。同时带来 RISC-V V 的 f16/bf16 inner_tiled MMA、Wasm SIMD128 打包 f32 布局、实验性 early data tiling 与可复现 .vmfb 构建，并修复了 attention 归一化操作数顺序、GPU 整数归约单位元等多处正确性问题。

## 重点变化

### 1. Release v3.14.0

IREE v3.14.0 正式发布。量化卷积不再依赖浮点路径：im2col 被统一为单一通用重写并移到 QDQ 转换之后执行，覆盖分组、空洞、一维/三维、动态形状与整数卷积；常量 4-bit 和 2-bit 量化权重打包为 i4/i2 存储，torchao 的 quantize_affine/dequantize_affine 也会转换到 LinalgExt。为支持 Mixture-of-Experts 模型，新增 LinalgExt 算子 group_matmul 和 group_mmt4d，对应 PyTorch 的 grouped_mm。后端方面，RISC-V V 增加 f16 与 bf16 的 inner_tiled MMA，Wasm SIMD128 支持打包 f32 布局。此外还引入实验性 early data tiling 与可复现 .vmfb 构建，并修复 attention 归一化操作数顺序错误、GPU 整数归约单位元错误、Flow 流水线 canonicalization/CSE 未生效等问题；iree_linalg_ext.sort 改为 ins()/outs() 语义，默认不再稳定，并以更快算法替换 O(n²) 冒泡排序。

**为什么重要：** 这些改动直接面向边缘与端侧推理的效率与可移植性：量化卷积全面整数化并配合 i4/i2 紧凑权重存储，可以降低内存占用与算力开销；group_matmul 等算子则为 MoE 这类依赖分组矩阵乘法的模型铺平了编译路径。正确性修复同样关键，attention 归一化操作数顺序与 GPU 整数归约单位元此前会产生错误结果，修复后这些优化路径才可放心启用。可复现的 .vmfb 构建也为 CI 缓存与发布流程提供了确定性基础。

**涉及层级：** `frontend`、`graph_ir`、`tensor_ir`、`optimization`、`kernel_codegen`、`hardware_backend`、`runtime`

**来源：** [原始来源 1](https://github.com/iree-org/iree/releases/tag/v3.14.0)

**事件评分：** 80.5/100

## 值得继续观察

- IREE 量化卷积整数化与 i4/i2 权重打包
- LinalgExt group_matmul/group_mmt4d 的 MoE 支持
- RISC-V V f16/bf16 inner_tiled MMA
- Wasm SIMD128 打包 f32 布局
- IREE 可复现 .vmfb 构建与 early data tiling

## 数据与生成说明

- 采集窗口：`2026-11-16T08:00:00+08:00` 至 `2026-11-17T08:00:00+08:00`
- 生成模式：`deepseek`
- 文章由程序根据一手来源生成；性能结论应以链接中的原始配置为准。
- 原始条目：LLVM 1 条，PyTorch 2 条，iree-org/iree 1 条
