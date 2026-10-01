---
layout: default
categories: [ai-compiler, daily]
title: "TileLang v0.1.15 新增 Ascend 950 原生后端与自动 warp 特化；研究用强化学习为 MLIR 选择编译 pass"
date: 2026-10-01
generation_mode: "deepseek"
window_since: "2026-09-30T08:00:00+08:00"
cutoff_at: "2026-10-01T08:00:00+08:00"
---

# TileLang v0.1.15 新增 Ascend 950 原生后端与自动 warp 特化；研究用强化学习为 MLIR 选择编译 pass

## 今日概览

2026年10月1日：TileLang 发布 v0.1.15，加入华为 Ascend 950 原生 NPU 后端、基于角色的自动 CUDA warp specialization、统一块缩放 GEMM 语义以及更丰富的 Python 编译期前端，并修复大量 codegen 问题；学术方面，MQSS-Selector 论文提出用强化学习统一处理设备选择、编译 pass 选择与作业队列调度，为 MLIR 编译管线引入多目标的学习式 pass 选择。

## 重点变化

### 1. v0.1.15

TileLang v0.1.15 发布，新增华为 Ascend 950 的原生 NPU 后端，支持 Cube 与 Vector 混合内核、自动调度与同步插入，并集成 Bisheng 编译流程；同时引入基于角色的 CUDA 自动 warp specialization 调度器，统一分块缩放 GEMM 语义并改进 SM100/SM120 指令选择，Python 前端支持编译期迭代与推导式，还包含大量 CUDA lowering 与 codegen 修复、ROCm/CPU/Metal 后端改进，以及运行时缓存改为 JSON 参数与原子发布。

**为什么重要：** 自动 warp specialization 可以省去手工划分 warp 组的调优负担，而 Ascend 后端让同一套 TileLang 前端扩展到国产 NPU 硬件；运行时改用 JSON 缓存参数并以原子方式发布不可变缓存目录，也有助于避免并发读取到部分发布的内核。

**涉及层级：** `frontend`、`tensor_ir`、`optimization`、`kernel_codegen`、`hardware_backend`、`runtime`、`benchmark`

**来源：** [原始来源 1](https://github.com/tile-ai/tilelang/releases/tag/v0.1.15)

**事件评分：** 87.8/100

### 2. MQSS-Selector: RL-Guided Pass Selection for an MLIR Compilation Pipeline

该论文提出 MQSS-Selector，一种基于强化学习与深度学习的统一编译选择器，将 MLIR 编译管线中的设备选择、编译 pass 优化与作业队列调度整合进同一个框架，可同时优化保真度、编译时间与调度延迟等多个目标，并随量子电路特征与设备状态动态调整。

**为什么重要：** 编译 pass 的选择与排序长期依赖手工经验，学习式选择器有望自动适配工作负载与硬件；将设备选择与队列调度纳入同一决策框架，也把优化目标从单纯的运行性能扩展到编译时间与调度延迟等多维权衡。

**涉及层级：** `optimization`、`runtime`

**来源：** [原始来源 1](https://arxiv.org/abs/2609.30104v2)

**事件评分：** 60.3/100

## 值得继续观察

- TileLang Ascend 950 NPU 后端
- 基于角色的自动 warp specialization
- MLIR pass 选择的强化学习方法
- 量子编译栈的设备与调度联合优化

## 数据与生成说明

- 采集窗口：`2026-09-30T08:00:00+08:00` 至 `2026-10-01T08:00:00+08:00`
- 生成模式：`deepseek`
- 文章由程序根据一手来源生成；性能结论应以链接中的原始配置为准。
- 原始条目：NVIDIA 3 条，PyTorch 2 条，arXiv 2 条，pytorch/pytorch 1 条，tile-ai/tilelang 1 条
