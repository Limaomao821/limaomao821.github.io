---
layout: default
categories: [ai-compiler, daily]
title: "TileLang v0.1.16 发布：新增华为 Ascend 950 PTO 后端与 T.vmi 自动向量化，CuTeDSL 覆盖 SM120/SM121 块缩放 MMA"
date: 2026-11-28
generation_mode: "deepseek"
window_since: "2026-11-27T08:00:00+08:00"
cutoff_at: "2026-11-28T08:00:00+08:00"
---

# TileLang v0.1.16 发布：新增华为 Ascend 950 PTO 后端与 T.vmi 自动向量化，CuTeDSL 覆盖 SM120/SM121 块缩放 MMA

## 今日概览

2026 年 11 月 28 日的重点动态来自 TileLang v0.1.16 版本发布。该版本首次将华为 Ascend 950 的 PTO 后端纳入主线，并默认经 tvm_ffi 执行，同时引入 T.vmi 虚拟指令和 T.Parallel 在 T.SimdVF 内的自动向量化；CuTeDSL 后端扩展到集群 TMA、SM120/SM121 块缩放 MMA 与 FP8 稀疏 WGMMA；在 tvm_ffi 下新增 NVRTC 设备编译、进程级并行编译（par_compile）与 TILELANG_CACHE_DIR 多根缓存，并对 CUDA、ROCm、CPU 后端做了归约、GEMM、WMMA 与 OpenMP 并行等多方面增强。

## 重点变化

### 1. v0.1.16

TileLang 发布 v0.1.16。新增华为 Ascend 950 PTO 后端（target="pto"，默认经 tvm_ffi 执行，保留 execution_backend="pto" 供旧版 Cython launcher 使用），并加入 T.vmi 虚拟机器指令，使 T.Parallel 循环可在 T.SimdVF 内自动向量化，支持独立 AIV 执行、共享标量存储和精确片上缓冲放置。CuTeDSL 后端新增集群 TMA（multicast、gather4、scatter4）、SM120/SM121 块缩放 MMA、FP8 算术与稀疏 WGMMA、Philox4x32-10 RNG，并支持 sm_103a 与 sm_107a 目标。在 tvm_ffi 下支持 NVRTC 设备编译（tl.cuda_compiler="nvrtc"）、进程级并行编译（par_compile executor="process"）与 TILELANG_CACHE_DIR 多根有序缓存查找。CUDA 侧新增 host-mapped 零拷贝缓冲、L1/L2 预取与基于 redux.sync 的单轮跨 warp AllReduce；ROCm 新增 gfx1200 FP16/BF16 WMMA 线性共享布局、pip-SDK ROCm 10 支持与 E8M0/FP4 cast；CPU 支持 OpenMP grid-loop 并行（tl.cpu_parallel），并附有大量编译正确性修复。

**为什么重要：** 这是 TileLang 首次以正式版本把华为昇腾 950 的 PTO 后端推上默认执行路径，配合 T.vmi 自动向量化，可能明显降低在昇腾硬件上编写高性能 kernel 的门槛。CuTeDSL 对 SM120/SM121 块缩放 MMA 与 FP8 稀疏 WGMMA 的支持，使 TileLang 能面向下一代 NVIDIA GPU 的低精度与稀疏算子提前完成编译路径验证。NVRTC 设备编译、进程级并行编译和多根缓存则主要改善大规模工程场景下的编译时延、工具链兼容性与团队间的缓存共享。

**涉及层级：** `frontend`、`graph_ir`、`tensor_ir`、`optimization`、`kernel_codegen`、`hardware_backend`、`runtime`

**来源：** [原始来源 1](https://github.com/tile-ai/tilelang/releases/tag/v0.1.16)

**事件评分：** 82.8/100

## 值得继续观察

- TileLang Ascend 950 PTO 后端与 T.vmi 自动向量化在 FlashMLA 等真实模型上的性能与稳定性
- SM120/SM121 块缩放 MMA、FP8 稀疏 WGMMA 及集群 TMA 的算子覆盖范围与后续正确性收敛
- tvm_ffi 下 NVRTC 设备编译与进程级 par_compile 带来的实际编译时延收益

## 数据与生成说明

- 采集窗口：`2026-11-27T08:00:00+08:00` 至 `2026-11-28T08:00:00+08:00`
- 生成模式：`deepseek`
- 文章由程序根据一手来源生成；性能结论应以链接中的原始配置为准。
- 原始条目：tile-ai/tilelang 1 条
