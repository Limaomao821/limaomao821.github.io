---
layout: default
categories: [ai-compiler, daily]
title: "KernelFoundry：进化搜索驱动 GPU kernel 生成，KernelBench 平均提速 2.3 倍"
date: 2026-09-04
generation_mode: "deepseek"
window_since: "2026-09-03T08:00:00+08:00"
cutoff_at: "2026-09-04T08:00:00+08:00"
---

# KernelFoundry：进化搜索驱动 GPU kernel 生成，KernelBench 平均提速 2.3 倍

## 今日概览

今日关注一项基于大模型的 GPU kernel 进化优化框架 KernelFoundry。该框架将 MAP-Elites 质量多样性搜索、meta-prompt 与 kernel 的共同进化、以及模板化参数优化相结合，同时生成 SYCL 与 CUDA kernel，在 KernelBench 上取得平均 2.3 倍加速，并通过分布式框架支持远程异构硬件的快速基准测试。

## 重点变化

### 1. KernelFoundry: Hardware-aware evolutionary GPU kernel optimization

KernelFoundry 是一个基于大模型的 GPU kernel 进化优化框架，通过 MAP-Elites 质量多样性搜索以 kernel 专属行为维度维持探索，让 meta-prompt 与 kernel 共同进化以发现任务特定的优化策略，并采用模板化参数优化使 kernel 适配具体输入与硬件。该框架生成 SYCL 与 CUDA kernel，在 KernelBench、robust-kbench 及自定义任务上评估，SYCL 实现在 KernelBench 上取得平均 2.3 倍加速，同时以分布式形式提供对多样异构硬件的远程访问以支持快速 benchmark。

**为什么重要：** GPU kernel 优化要求同时理解硬件架构、并行计算策略与 profiling 反馈，难度超出常规代码生成任务，而多数现有方法仅通过 profiling 反馈间接感知硬件。KernelFoundry 把硬件感知的进化搜索引入大模型 kernel 生成流程，质量多样性与 prompt 共进化有望发现任务特定的优化策略，其跨平台 SYCL 输出与分布式基准框架也为编译器后端及异构硬件自动调优提供了可复用的探索思路。

**涉及层级：** `kernel_codegen`、`hardware_backend`、`distributed`、`benchmark`

**来源：** [原始来源 1](https://arxiv.org/abs/2603.12440v2)

**事件评分：** 65.0/100

## 值得继续观察

- KernelFoundry 的进化搜索方法在 KernelBench 之外的真实负载上的泛化表现
- SYCL 跨平台 kernel 与 CUDA kernel 的生成质量及可移植性对比
- MAP-Elites 与 meta-prompt 共进化在编译器后端自动调优场景的进一步应用

## 数据与生成说明

- 采集窗口：`2026-09-03T08:00:00+08:00` 至 `2026-09-04T08:00:00+08:00`
- 生成模式：`deepseek`
- 文章由程序根据一手来源生成；性能结论应以链接中的原始配置为准。
- 原始条目：NVIDIA 2 条，arXiv 2 条
