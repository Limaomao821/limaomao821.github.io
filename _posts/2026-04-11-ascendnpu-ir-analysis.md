---
layout: default
title: "AscendNPU-IR Pass 分析"
date: 2026-04-11
categories: [compiler, mlir, ascend]
---

# Overview

整理分析了开源版本[AscendNPU-IR](https://gitcode.com/Ascend/AscendNPU-IR)中Pass的作用。

## 阅读说明

- 文中以 `.td` 结尾的一级标题表示“Pass 定义文件”。
- 每个 `.td` 标题下的各个 Pass 小节，均来自该文件中的声明与对应实现。
- 这种组织方式用于建立“定义文件 → Pass 列表与行为说明”的映射关系，直到下一个 `.td` 文件标题开始。

# Pass 定义文件：`Conversion_Passes.td`

## convert-arith-to-affine

作用概述

- 将索引类型上的 arith 运算转换为 affine 运算，便于后续基于 Affine 的分析与优化，如循环规范化、边界简化、Polyhedral 推理等
- 声明与构造器： bishengir/include/bishengir/Conversion/Passes.td:27-31
- 具体实现： bishengir/lib/Conversion/ArithToAffine/ArithToAffine.cpp:117-136 ，构造并应用模式完成局部转换
  转换内容
- 二元算术到 affine.apply ：
  - arith.addi → Affine 加法： ArithToAffine.cpp:94-99
  - arith.subi → Affine 加法 + 右操作数取负： ArithToAffine.cpp:48-53, 96-99
  - arith.muli → Affine 乘法： ArithToAffine.cpp:97-99
  - arith.ceildivsi → Affine 向上整除： ArithToAffine.cpp:98-99
  - arith.divsi → Affine 向下整除： ArithToAffine.cpp:99-100
  - arith.remsi → Affine 取模： ArithToAffine.cpp:100-101
- 取最值到 affine.min/max ：
  - arith.maxsi / arith.maxui → affine.max ： ArithToAffine.cpp:103-105, 79-82
  - arith.minsi / arith.minui → affine.min ： ArithToAffine.cpp:105-106, 82-84
    触发条件
- 仅当两个操作数均为 index 类型时才转换；否则保留为合法的 arith 运算
  - 动态合法性判定： bishengir/lib/Conversion/ArithToAffine/ArithToAffine.cpp:121-130
  - Affine 方言设为目标合法方言： ArithToAffine.cpp:120-121
    在管线中的位置
- HFusion 自动调度后进行循环简化时使用： bishengir/lib/Dialect/HFusion/Pipelines/HFusionPipelines.cpp:210-214
- HIVM 规范化管线中用于简化与 CSE 前： bishengir/lib/Dialect/HIVM/Pipelines/HIVMPipelines.cpp:36
  示例（教育性）

```
// 原始（两个操作数为 index）
%r = arith.addi %i, %j : index

// 转换后
%r = affine.apply (s0 + s1) (%i, %j)
```

关键点

- 该 pass 不改变计算语义，只改变表示方式，让索引表达式进入 Affine 体系
- 转换是“局部转换”，若某些 arith 不满足索引类型约束则不会强制改写

## convert-arith-to-hfusion

**作用概述**

- 将作用在张量上的 `arith` 元素级运算转换为 `HFusion` 或 `Linalg` 方言的命名元素算子，使计算更易于融合、调度与后续设备化降级
- 声明与构造器：`bishengir/include/bishengir/Conversion/Passes.td:37-41`
- 实现入口：`bishengir/lib/Conversion/ArithToHFusion/ArithToHFusion.cpp:442-469`

**转换范围**

- 转为 Linalg 元素算子
  - 二元运算：`add/sub/mul/div(si/ui)/max(si/ui)/min(si/ui)` → `linalg::ElemwiseBinaryOp`，函数属性用 `linalg::BinaryFn` 标记，见 `ArithToHFusion.cpp:374-395`
  - 一元运算：`negf` → `linalg::ElemwiseUnaryOp`，见 `ArithToHFusion.cpp:393-395`
  - 张量常量整值/浮值“一致填充值”(splat) → `linalg.fill`，见 `ArithToHFusion.cpp:336-372`
- 转为 HFusion 元素算子
  - 位运算：`and/or/xor` → `hfusion::ElemwiseBinaryOp`，见 `ArithToHFusion.cpp:400-403`
  - 浮点最值：`minnum/minimum/maxnum/maximum` → `hfusion::ElemwiseBinaryOp`，见 `ArithToHFusion.cpp:403-406`
  - 整除/取模/位移：`remsi/shli/shrsi/shrui/floordivsi/ceildivsi/ceildivui` → `hfusion::ElemwiseBinaryOp`，见 `ArithToHFusion.cpp:407-416`
  - 比较：`arith.cmpf/cmpi` → `hfusion::CompareOp`，谓词映射见 `ArithToHFusion.cpp:246-292, 304-311`
  - 选择：`arith.select` → `hfusion::SelectOp`，见 `ArithToHFusion.cpp:314-333`
  - 扩展乘：`arith.mulsi_extended/mului_extended` → `hfusion::MulExtOp`，见 `ArithToHFusion.cpp:130-147, 430-432`
  - 位宽转换：`truncf/extf/fptosi/fptoui/sitofp/uitofp/extsi/extui/trunc i/f` → `hfusion::CastOp`，并根据类型选择 `RoundMode`（`RINT/TRUNC/TRUNCWITHOVERFLOW`），见 `ArithToHFusion.cpp:185-241, 417-426`
  - 位解释：`arith.bitcast` → `hfusion::BitcastOp`，同时创建与目标元素类型一致的 `tensor.empty` 作为输出，见 `ArithToHFusion.cpp:151-170, 432`

**触发条件与合法性规则**

- 仅当“所有操作数为 `RankedTensorType`”时触发转换；非张量上的 `arith` 运算保持为合法，不做改写：`ArithToHFusion.cpp:39-42, 456-457`
- `arith.constant` 若是张量 `DenseElementsAttr` 且为 splat，需要改写为 `linalg.fill`；非 splat 常量保持合法：`ArithToHFusion.cpp:449-457, 336-372`
- 目标合法方言包含 `linalg/tensor/hfusion`，确保转换后 IR 可继续在融合与调度管线中处理：`ArithToHFusion.cpp:445-447`

**在管线中的位置**

- HFusion 预处理阶段与若干规范化后都会调用该转换，以便尽早将张量上的标量 `arith` 运算统一为元素算子，利于融合与后续优化：`bishengir/lib/Dialect/HFusion/Pipelines/HFusionPipelines.cpp:84-100, 127-129`
- 与 `convert-arith-to-affine` 区分：前者处理“张量上的元素算子”，后者处理“索引类型上的算术表达式”以进入 `affine` 体系

**实现要点**

- 目的张量采用 `tensor::getOrCreateDestinations` 获取或创建目标 `tensor.empty`，以保证 SSA/形状一致性：`ArithToHFusion.cpp:54-61, 76-83, 97-104, 118-126, 228-238, 298-311, 323-332`
- Cast 的舍入/溢出策略按输入/输出类型组合选择，避免非法数据行为：`ArithToHFusion.cpp:189-220`
- Bitcast 保持形状不变，仅更换元素类型：`ArithToHFusion.cpp:159-167`

**简例**

- `arith.addf`（张量）转换为 HFusion/Linalg 元素加法
  - 原始：`%r = arith.addf %a, %b : tensor<?xf32>`
  - 转换：`%dst = tensor.empty(...)`；`%r = hfusion.elemwise.binary %a, %b {fun = #hfusion.binary_fn<add>} -> %dst`
- `arith.constant` splat 张量转 `linalg.fill`：`ArithToHFusion.cpp:357-369`

## convert-math-to-hfusion

**作用概述**

- 将作用在张量上的 `math` 方言运算改写为 `HFusion` 或 `Linalg` 的元素算子，便于后续融合、调度与设备后端降级
- 声明与构造器：`bishengir\include\bishengir\Conversion\Passes.td:47-51`
- 实现入口与执行：`bishengir\lib\Conversion\MathToHFusion\MathToHFusion.cpp:200-216, 218-220`

**转换范围**

- Linalg 一元元素算子
  - `math.exp/log/absf/ceil/floor` → `linalg::ElemwiseUnaryOp`，函数属性为 `linalg::UnaryFn`：`MathToHFusion.cpp:168-173, 47-66`
- HFusion 二元/一元元素算子
  - 二元：`mathExt.ldexp` → `hfusion::BinaryFn::ldexp`，`math.powf` → `hfusion::BinaryFn::powf`：`MathToHFusion.cpp:173-175, 111-131`
  - 一元：`math.sqrt/rsqrt/tanh/atan/tan/sin/cos/absi/erf/log2/log10/log1p/exp2/expm1`、`mathExt.ilogb` → 对应 `hfusion::UnaryFn`：`MathToHFusion.cpp:175-190, 90-109`
- 复合算子展开
  - `math.fma` 展开为 `linalg` 的先 `mul` 再 `add` 两个元素算子组合：`MathToHFusion.cpp:133-159`

**触发条件与合法性**

- 仅当所有操作数为 `RankedTensorType` 时触发转换；否则保留为合法的 `math` 运算：`MathToHFusion.cpp:42-45, 208-210`
- 目标合法方言：`linalg`、`tensor`、`hfusion`，并允许在转换过程中产生的 `arith`：`MathToHFusion.cpp:203-207`

**在管线中的位置**

- HFusion 预处理阶段调用该转换，使张量上的标量 `math` 运算统一为元素算子以利融合：`bishengir\lib\Dialect\HFusion\Pipelines\HFusionPipelines.cpp:84-86`

**实现要点**

- 目的张量通过 `tensor::getOrCreateDestinations` 获取/创建输出 `tensor.empty`，保证 SSA 与形状一致性：`MathToHFusion.cpp:57-64, 79-86, 99-107, 121-129`
- `fma` 使用 `hfusion::createBinaryOp` 辅助构造 `linalg` 元素算子链：`MathToHFusion.cpp:146-156`

**示例（教育性）**

```mlir
%y = math.exp %x : tensor<?xf32>
%dst = tensor.empty(%n) : tensor<?xf32>
%y = hfusion.elemwise.unary %x {fun = #hfusion.unary_fn<exp>} -> %dst
```

**关联文件**

- Pass 定义：`bishengir\include\bishengir\Conversion\Passes.td:47-51`
- 模式与运行：`bishengir\lib\Conversion\MathToHFusion\MathToHFusion.cpp:165-191, 200-216`
- 接口声明：`bishengir\include\bishengir\Conversion\MathToHFusion\MathToHFusion.h:31-37`

## convert-linalg-to-hfusion

**作用概述**

- 将特定 `linalg` 运算改写为 `HFusion` 方言的命名算子或等价的元素算子，便于后续融合、自动调度以及面向设备后端的降级
- 声明与构造器：`bishengir\include\bishengir\Conversion\Passes.td:57-61`
- 实现入口与执行：`bishengir\lib\Conversion\LinalgToHFusion\LinalgToHFusion.cpp:444-470`

**转换内容**

- `linalg.map` 到 HFusion/Linalg 元素算子或专用算子：`bishengir\lib\Conversion\LinalgToHFusion\LinalgToHFusion.cpp:45-227`
  - 根据 `map` 区域中的 `func.call` 名称（以 `__hmf_` 前缀）映射到目标算子
  - 典型映射：
    - 一元元素算子：`relu/log1p/sqrt/rsqrt/tan/tanh/atan/ilogb` → `hfusion::ElemwiseUnaryOp`，`fabs/exp/log` → `linalg::ElemwiseUnaryOp`
    - 二元元素算子：`ldexp/powf/powi` → `hfusion::ElemwiseBinaryOp`
    - 判定算子：`isinf` → `hfusion::IsInfOp`，`isnan` → `hfusion::IsNanOp`
    - 倒数：`recipf/recipDh` 展开为构造常量 1 的 `linalg.fill` + `linalg::ElemwiseBinaryOp(div)` 组合
    - 取整：`roundf` → `hfusion::CastOp`，`RoundMode=ROUND`
- `linalg.generic` 的特殊模式
  - `arange` 识别：当 `generic` 满足“全并行迭代、identity 索引映射、index 语义、体内产出为 `index -> cast -> yield` 的一维张量”时，改写为 `hfusion::ArangeOp`：`LinalgToHFusion.cpp:229-271`
  - 原子更新：带 `GenericAtomicRMW` 属性的 `generic` 改写为
    - `CAS` → `hfusion::AtomicCasOp`
    - `XCHG` → `hfusion::AtomicXchgOp`
    - 其他（`add/max/min/and/xor/or`）→ `hfusion::StoreOp` 并设置 `atomic_kind`：`LinalgToHFusion.cpp:291-359`
- `linalg.reduce`（带索引的归约）
  - 当有 `reduce_mode` 属性为 `"max_with_index"` 或 `"min_with_index"` 时，改写为 `hfusion::ReduceWithIndexOp`，保留 `dimensions` 并映射 `tie_break_left` 属性；若存在注解 `UseIndexInput` 决定是否保留第二输入：`LinalgToHFusion.cpp:375-427`

**触发条件与合法性**

- 目标合法方言包括：`memref/linalg/bufferization/tensor/hfusion`，并允许转换过程中生成的 `arith/math`：`LinalgToHFusion.cpp:447-452`
- 将 `linalg.map` 与 `linalg.generic` 标记为非法，强制它们被上述模式覆盖：`LinalgToHFusion.cpp:458-460`
- 对 `linalg.reduce` 采用动态合法性：没有 `reduce_mode` 属性的保持合法，有该属性的必须转换：`LinalgToHFusion.cpp:452-456`
- 模式注册入口：`populateLinalgToHFusionConversionPatterns`：`LinalgToHFusion.cpp:430-435`

**在管线中的位置**

- HFusion 预处理阶段调用本转换，使上层 `linalg` 语义尽早落入 HFusion 体系，利于后续融合与调度：`bishengir\lib\Dialect\HFusion\Pipelines\HFusionPipelines.cpp:84-87`

**关键点**

- 该 pass 主要面向“张量层面的 linalg 算子表达”，与将张量上的 `arith`/`math` 元素算子改写的两个 pass（`convert-arith-to-hfusion`、`convert-math-to-hfusion`）互补
- 对 `map` 的处理依赖约定的函数名（`__hmf_*`），这是连接上层库调用与下层元素算子的桥梁
- 对原子与带索引归约的专门识别确保后端可用的语义化 HFusion 原语，而不是保留通用 `generic` 区域表示

**参考位置**

- Pass 声明与构造器：`bishengir/include/bishengir/Conversion/Passes.td:57-61`
- 转换模式与细节：`bishengir/lib/Conversion/LinalgToHFusion/LinalgToHFusion.cpp:45-227, 229-359, 375-427, 444-470`

## convert-gpu-to-hfusion

**作用概述**

- 将 `GPU` 方言中的同步原语改写为 `HFusion` 方言的等价操作，剔除 `GPU` 方言，使 IR 进入 HFusion 体系以便后续融合与设备化管线处理
- 声明与构造器：`bishengir\include\bishengir\Conversion\Passes.td:67-71`
- 具体实现：`bishengir\lib\Conversion\GPUToHFusion\GPUToHFusion.cpp:56-71`

**转换内容**

- `gpu.barrier` → `hfusion.barrier` 一对一映射
  - 模式定义：`bishengir/lib/Conversion/GPUToHFusion/GPUToHFusion.cpp:34-41`
  - 模式注册：`bishengir/lib/Conversion/GPUToHFusion/GPUToHFusion.cpp:44-47`

**合法性规则**

- 目标合法方言：`hfusion::HFusionDialect`
- 将 `gpu::GPUDialect` 标记为非法，确保转换后 IR 不含 GPU 方言：`bishengir/lib/Conversion/GPUToHFusion/GPUToHFusion.cpp:59-61`

**在管线中的使用**

- Triton 内核路径开启时，先进行 Triton 适配，再做 GPU→HFusion 转换：`bishengir\lib\Dialect\HFusion\Pipelines\HFusionPipelines.cpp:87-91`

**示例（教育性）**

```mlir
// 原始
gpu.barrier

// 转换后
hfusion.barrier
```

**注意事项**

- 目前实现聚焦于屏障同步的改写；如后续引入更多 GPU 方言操作，需要补充相应转换模式，否则因 `gpu` 方言被标为非法会触发部分转换失败提示，这样可以及时提醒扩展覆盖集。

## convert-hfusion-to-hivm

**作用概述**

- 将 `HFusion` 与部分 `Linalg` 张量运算系统性改写为 `HIVM` 方言原语，消除上层融合语义，使 IR 进入面向设备后端的 HIVM 体系，便于后续缓冲化、对齐、同步与硬件映射
- 声明与构造器：`bishengir/include/bishengir/Conversion/Passes.td:77-81`
- 主体实现与执行：`bishengir/lib/Conversion/HFusionToHIVM/HFusionToHIVM.cpp:1024-1072`

**转换覆盖**

- 元素算子（统一处理 linalg/hfusion 两类元素算子）
  - 一元/二元/三元：映射到 `hivm::V*` 系列，如 `VAdd/VMul/VSub/VDiv/VExp/VLn/VRelu/VSqrt/VRsqrt/VTanh/VSin/VCos/VSel` 等：`HFusionToHIVM.cpp:166-201, 203-229, 231-254, 264-267, 373-404`
  - 比较与类型转换：`hfusion.compare/cast` → `hivm::VCmp/VCast`，并映射谓词与舍入模式：`HFusionToHIVM.cpp:126-134, 156-164`
  - 标量参与的元素算子：交换（若可交换）或通过 `VBrc` 广播成向量再参与计算：`HFusionToHIVM.cpp:283-371, 453-474`
- 广播与填充
  - `linalg.fill` → `hivm::VBrc`：`HFusionToHIVM.cpp:453-474`
  - `linalg.broadcast` 先 `expand_shape` 再 `VBrc`，保证源/目标同秩：`HFusionToHIVM.cpp:479-515`
- 复制与转置
  - `linalg.copy` → `hivm::Copy`：`HFusionToHIVM.cpp:521-532`
  - `linalg.transpose` → `hivm::VTranspose`：`HFusionToHIVM.cpp:539-551`
- 加载/存储与原子
  - `hfusion.load/store` → `hivm::Load/Store`，并传递 `atomic_kind`：`HFusionToHIVM.cpp:612-623, 660-682`
  - `hfusion.atomic.cas/xchg` → `hivm::AtomicCas/AtomicXchg`：`HFusionToHIVM.cpp:892-907`
- 位解释、翻转、交织
  - `hfusion.bitcast` → `hivm::Bitcast`：`HFusionToHIVM.cpp:689-716`
  - `hfusion.flip` → `hivm::VFlip`：`HFusionToHIVM.cpp:855-869`
  - `hfusion.interleave/deinterleave` → `hivm::VInterleave/VDeinterleave`：`HFusionToHIVM.cpp:805-849`
- 排序与累计
  - `hfusion.sort` → `hivm::VSort`：`HFusionToHIVM.cpp:913-926`
  - `hfusion.cumsum/cumprod` → `hivm::VCumsum/VCumprod`：`HFusionToHIVM.cpp:874-887`
- 核心归约
  - `linalg.reduce` 与 `hfusion.reduce_with_index` → `hivm::VReduce`，自动识别 `sum/prod/max/min/and/or/xor/any/all` 等，并用 `expand_shape/collapse_shape` 保持形状一致：`bishengir/lib/Conversion/HFusionToHIVM/Reduction.cpp:47-93, 100-173, 178-184`
- Matmul 映射（两类）
  - 局部 MMAD（向量核 L1）：`linalg.matmul/batch_matmul` → `hivm::MmadL1/BatchMmadL1`，提取转置标志与初始化条件（从外层 `scf.for` 建模），并在外层 K-loop 前插入新的 init tensor：`bishengir/lib/Conversion/HFusionToHIVM/Matmul.cpp:48-99, 100-115, 205-225, 327-367`
  - 全局 Matmul（GM）：`linalg.matmul/*` 或 `hfusion.group_matmul` → `hivm::Matmul/MixMatmul/MixGroupMatmul`，同时
    - 从 `annotation::MarkOp` 提取并移除 `tiling_struct/workspace/post_vector_func` 参数：`Matmul.cpp:408-454`
    - 处理 `dummy` 调用以选取真实输出：`Matmul.cpp:456-474`
    - 保留后处理标记 `post_vector_func`：`Matmul.cpp:517-522`
- 屏障与调试
  - `hfusion.barrier` → `hivm::PipeBarrier`（管道全域）：`HFusionToHIVM.cpp:765-774`
  - `hfusion.print/assert` → `hivm::DebugOp`（类型 `print/assert`）：`HFusionToHIVM.cpp:723-739, 746-762`

**合法性与规则**

- 目标合法方言：`hivm/memref/bufferization/tensor/arith/affine/scf/func`：`HFusionToHIVM.cpp:1034-1038`
- 标记非法方言：`linalg/hfusion`，强制其被转换覆盖：`HFusionToHIVM.cpp:1039-1040`
- 额外重写：在函数内对 HIVM Op 应用补充规则（如属性转移、绑定子块映射）：`HFusionToHIVM.cpp:1056-1060, 929-987`

**选项与管线位置**

- `mmMapMode` 控制 Matmul 映射策略（`CoreOp` vs `MacroInstr`），由管线在编译 Triton 内核时选择：`bishengir/include/bishengir/Conversion/Passes.td:82-91`，`bishengir/lib/Dialect/HIVM/Pipelines/ConvertToHIVMPipeline.cpp:31-35`
- 在 “Convert to HIVM” 管线中首先执行该转换，再做张量到 HIVM、通用到 HIVM 的收敛：`ConvertToHIVMPipeline.cpp:29-41`

**实现要点**

- 统一的元素算子转换器封装 `DestinationStyleOpInterface`，保持 Dps 输入/初始化与结果类型一致：`HFusionToHIVM.cpp:70-105`
- 谓词/舍入模式从 HFusion 到 HIVM 的枚举映射：`HFusionToHIVM.cpp:107-134, 136-153`
- 形状一致性通过 `expand_shape/collapse_shape` 保障（广播与归约场景）：`HFusionToHIVM.cpp:499-515`，`Reduction.cpp:128-166`
- 标量参与元素运算的修正策略：
  - 交换操作数（可交换时）：`HFusionToHIVM.cpp:326-355`
  - 否则用 `VBrc` 将标量扩为向量：`HFusionToHIVM.cpp:314-324, 370-371`
- 将 HFusion 标注属性降级或改名为 HIVM 的对应属性（如 `multi_buffer`、`stride_align_*`）：`HFusionToHIVM.cpp:929-967`

这一步在整体编译流中承担“融合到设备原语”的关键桥接工作：向量/局部计算用 HIVM `V*`/`MMAD` 系列表达，内存/同步用 HIVM 的 `Load/Store/Copy/PipeBarrier` 等表达，归约/矩阵乘等复杂算子用专用 HIVM 原语，确保后续 HIVM 优化与缓冲化、对齐、同步求解能有效进行。

## convert-tensor-to-hfusion

**作用**

- 将 `tensor` 方言中的构造类操作规范化为 `Linalg/HFusion` 可融合的目标风格，目前重点是把 `tensor.splat` 改写为 `linalg.fill`，使后续 HFusion 规范化、融合、调度与设备映射流程统一处理
- 目标是“把张量生成/填充从纯 `tensor` 语义迁移到 destination-style 的 `linalg/hfusion` 语义”，与 HFusion 的 Dps 接口和广播/归约策略保持一致

**关键改写**

- `tensor.splat` → `linalg.fill`
  - 创建与结果同形状的 `tensor.empty` 作为目的缓冲，然后以输入常量 `fill` 到该空张量
  - 位置：`bishengir/lib/Conversion/TensorToHFusion/TensorToHFusion.cpp:34-47`
- 合法性/非法性设置
  - 合法方言：`tensor`, `linalg`, `hfusion`：`TensorToHFusion.cpp:65-66`
  - 标记非法 `tensor.splat`，强制以模式重写：`TensorToHFusion.cpp:67-68`
- 部分转换应用与失败处理：`applyPartialConversion` + `signalPassFailure`：`TensorToHFusion.cpp:69-73`
- Pass 构造器：`createTensorToHFusionConversionPass()`：`TensorToHFusion.cpp:75-77`
- 模式注册函数：`populateTensorToHFusionConversionPatterns`：`TensorToHFusion.cpp:50-53`

**管线位置**

- 在 HFusion 预处理阶段执行，位于 Arith/Math/Linalg→HFusion 之后，用于清理残留的 `tensor` 构造操作，使形状与数据生成进入可融合的 `linalg/hfusion` 体系：`bishengir/lib/Dialect/HFusion/Pipelines/HFusionPipelines.cpp:92-99`

**为什么需要**

- HFusion 的大多数优化与转换（广播、归约、元素算子、规范化）围绕 destination-style 的 `linalg/hfusion` 操作展开；直接保留 `tensor.splat` 会阻塞融合与形状变换的统一处理
- 将常量填充统一为 `linalg.fill` 能与后续 `VBrc` 广播、`expand/collapse_shape`、规范化等步骤自然协同，减少特殊路径与边界情况

**文件/符号**

- 定义：`bishengir/include/bishengir/Conversion/Passes.td:97-98`
- 实现：`bishengir/lib/Conversion/TensorToHFusion/TensorToHFusion.cpp`
- 模式头文件：`bishengir/include/bishengir/Conversion/TensorToHFusion/TensorToHFusion.h`

## convert-tensor-to-hivm

**作用**

- 将 `tensor` 方言中的拼接与填充类操作改写为 `HIVM` 的目标原语，清理剩余的高层张量构造，使 IR 进入设备后端的 HIVM 体系以便后续缓冲化、对齐和硬件映射
- 定义位置：`bishengir\include\bishengir\Conversion\Passes.td:108-109`

**关键改写**

- `tensor.concat` → `hivm::VConcatOp`
  - 通过 `reifyResultShapes` 推导输出形状，生成对应的 `tensor.empty` 作为目的缓冲，然后用 `VConcatOp` 替换：`bishengir\lib\Conversion\TensorToHIVM\TensorToHIVM.cpp:44-58`
  - 追踪并转移 `InsertSliceSourceIndex` 属性：自上游 `annotation::MarkOp` 递归查找并设到新 `VConcat` 上，随后删除标记：`TensorToHIVM.cpp:66-86`
- `tensor.pad` → `hivm::VPadOp`
  - 对每一维计算结果大小 `src + low + high`（支持动态维度，使用 `affine::makeComposedFoldedAffineApply` 组合符号表达式）：`TensorToHIVM.cpp:112-127`
  - 构造 `tensor.empty` 作为目的缓冲，创建 `VPadOp` 并携带常量填充值与动态/静态低高填充参数：`TensorToHIVM.cpp:134-139`
  - 若 HIVM 计算出的结果张量形状与期望类型不一致，插入 `tensor.cast` 保持类型一致：`TensorToHIVM.cpp:139-147`

**合法性与目标**

- 标记非法并强制重写的 `tensor` 操作：`tensor::ConcatOp`, `tensor::PadOp`：`TensorToHIVM.cpp:170`
- 合法方言集合：`hivm`, `func`, `tensor`, `arith`, `affine`，允许这些方言的结果 IR：`TensorToHIVM.cpp:167-169`
- Pass 构造与应用：`TensorToHIVM.cpp:163-175, 177-179`

**管线位置**

- 在“转换到 HIVM”总管线中，位于 `HFusion→HIVM` 之后，用于清理残留的 `tensor` 结构并用 HIVM 原语收敛：`bishengir\lib\Dialect\HIVM\Pipelines\ConvertToHIVMPipeline.cpp:39-41`

**为什么需要**

- 将拼接/填充统一为目的风格的 `HIVM` 原语，有助于后续缓冲化、对齐、同步与设备映射；形状计算显式化（含动态维）并绑定到 HIVM 操作的语义
- 属性追踪（如 `InsertSliceSourceIndex`）保证在跨方言重写中保留上游逻辑信息，减少优化阶段的信息丢失

**范围与支持**

- 当前覆盖 `tensor.concat` 与 `tensor.pad` 的改写，完整支持静态/动态形状场景及常量填充值；其它 `tensor` 构造类操作由前置的 HFusion/Linalg/Tensor→HFusion pass 或后续 HIVM 收敛 pass 共同完成

## lower-memeref-ext

**作用**

- 将 `memref_ext.alloc_workspace` 降低为标准 `memref.view`，按“每块（block）本地工作空间”模型把工作空间指针偏移计算展开到 IR，去除 `MemRefExt` 方言
- 面向 HIVM 后端的内存规划与同步流程，显式化工作空间地址计算，便于后续缓冲化、对齐与代码生成
- 声明位置：`bishengir\include\bishengir\Conversion\Passes.td:118-120`
- 实现入口：`bishengir\lib\Conversion\LowerMemRefExt\LowerMemRefExt.cpp:57-60`

**关键转换**

- `memref_ext.alloc_workspace(...)` → `memref.view(base, byte_shift=offset, dynamic_sizes=[])`
  - 计算“块内本地偏移”`localOffset`，若存在双缓冲（两个 offset），使用循环迭代计数对 2 取模选择当前偏移：`LowerMemRefExt.cpp:76-95`
  - 获取块 ID：`hivm.get_block_idx` 并转为 `index`：`LowerMemRefExt.cpp:98-103`
  - 用仿射式计算字节偏移：`viewOffset = blockIdx * localWorkSpaceSize + localOffset`：`LowerMemRefExt.cpp:105-112`
  - 构造 `memref.view` 指向目标类型（从 `memref<?xi8>` 视图到目标 `memref<...>`），替换原 op：`LowerMemRefExt.cpp:115-119`

**工作空间大小来源**

- 从模块内的 Host 函数（类型为 `infer_workspace_shape_function`）读取“每块本地工作空间大小”，要求该函数返回常量 `index`：`LowerMemRefExt.cpp:126-154`
- 该函数通常由前置管线注入：`bishengir\lib\Dialect\HIVM\Pipelines\HIVMPipelines.cpp:165-168`

**条件与限制**

- 仅支持带 `offset` 的 `alloc_workspace`，否则报错：`LowerMemRefExt.cpp:71-74`
- 偏移参数长度最多为 2（用于双缓冲切换）：`LowerMemRefExt.cpp:76-80`
- 当前实现目标为“每块（block）可见”的本地工作空间；跨块共享的全局工作空间未在此步处理（注释说明未来方向）：`LowerMemRefExt.cpp:156-160`
- 重写使用 `memref.view(..., dynamic_sizes=[])`，在实践中目标类型应具备静态尺寸或在其他阶段提供动态尺寸

**管线位置**

- HIVM 优化管线中，在规划工作空间与插入推断函数之后执行：`bishengir\lib\Dialect\HIVM\Pipelines\HIVMPipelines.cpp:149-169`

**相关文件**

- 方言定义与示例：`bishengir\include\bishengir\Dialect\MemRefExt\IR\MemRefExtOps.td:8-32, 35-73`
- Pass 实现与模式：`bishengir\lib\Conversion\LowerMemRefExt\LowerMemRefExt.cpp:62-124`
- 测试样例（展示生成 `get_block_idx`、`affine.apply` 和 `memref.view`）：`bishengir\test\Conversion\LowerMemRefExt\LowerMemRefExt.mlir:1-22`

## convert-torch-to-hfusion

**作用**

- 将可识别的 Torch 方言算子批量改写为 `Linalg` 或 `HFusion` 的“命名/目标风格”算子，建立“张量 on Linalg/HFusion”的后端契约，便于后续 HFusion 归约、融合、调度与设备映射
- 定义位置：`bishengir\include\bishengir\Conversion\Passes.td:134-135`
- 构造器与主体：`bishengir\lib\Conversion\TorchToHFusion\TorchToHFusion.cpp:84-93, 53-80`

**覆盖范围**

- 元素类算子
  - 二元/一元映射到 `linalg` 或 `hfusion` 命名元素算子，统一广播与类型提升
  - 例：`aten.add/sub` 带 `alpha` 展开为 `mul`+`add`（`linalg`）：`.../Elementwise.cpp:170-177`
  - 例：`aten.mul/div/max/min` 分别映射到签名/无符号版本：`.../Elementwise.cpp:208-216`
  - 比较与选择：`aten.lt/le/gt/ge/eq/ne` → `hfusion.compare`；`aten.where` → `hfusion.select`：`.../Elementwise.cpp:339-342, 420-422`
  - `aten.clamp` 分步用 `linalg.max_*`/`min_*` 实现：`.../Elementwise.cpp:491-502`
  - 类型转换：`aten.to.dtype` → `hfusion.cast`：`.../Elementwise.cpp:528-533`
  - 特殊激活：`aten.sigmoid` 以 `linalg.exp/add/div` 组合实现：`.../Elementwise.cpp:375-387`
- 归约类算子
  - `aten.sum/prod/any/all/max/min`（含 `dim`/`keepdim` 变体）：
    - 一般归约 → `linalg.reduce`，支持 `keepdim` 与动态形状：`.../Reduction.cpp:493-501, 503-508`
    - 带索引的极值（`max/min(dim)`）→ `hfusion.reduce_with_index`，初始化、索引类型与维度处理：`.../Reduction.cpp:192-200, 202-209, 211-219`
  - 归约初值推导（浮点/有符号/无符号/布尔）：`.../Reduction.cpp:74-106`
- 数据移动/形状
  - 置换：`aten.permute` → 通过助手生成 `linalg.transpose`，结果再 `tensor.cast`：`.../DataMovement.cpp:62-70`
  - 显式广播：`aten.broadcast_to` → 形状列表解析与目标形状广播，支持动态维与选项控制：`.../DataMovement.cpp:117-129, 146-149`
- 张量构造
  - 区间构造：`aten.arange_start_step`（要求 `step==1`）→ `hfusion.arange` 并计算结果长度：`.../TensorConstructors.cpp:92-100, 114-131`

**类型转换与合法性**

- 使用 Torch-MLIR 后端类型转换并登记依赖方言：`.../TorchToHFusion.cpp:47-51, 63-66`
- 合法方言：`linalg/func/cf/math/scf/sparse_tensor/tensor/arith/complex/hfusion`，并保留 `torch_conversion.get_next_seed`：`.../TorchToHFusion.cpp:55-62`
- 将涉及的 Torch 算子标记为非法以触发模式重写（见各 `populate*PatternsAndLegality` 实现）：`.../Elementwise.cpp:539-541`, `.../Reduction.cpp:326-329`, `.../DataMovement.cpp:143-149`, `.../TensorConstructors.cpp:143-145`, `.../Uncategorized.cpp:131-133`

**选项**

- `ensureNoImplicitBroadcast`：控制广播行为是否必须显式且形状匹配，避免隐式广播引入不确定性：`.../TorchToHFusion.cpp:67-69`, `.../DataMovement.cpp:146-149`

**管线集成**

- Torch 后端到命名算子管线中调用本 Pass，随后进一步转为 Linalg、规范化 reshape 等：`bishengir\lib\Dialect\Torch\Pipelines\TorchPipelines.cpp:71-78, 83-85`

**为什么需要**

- 统一将 Torch 高层语义转为 `Linalg/HFusion` 的目标风格（Destination-style、显式广播/归约），让 HFusion 的融合、归约规范化、算子调度以及后续 `HFusion→HIVM` 成为可能，减少跨方言语义分歧并提升可优化性

## convert-torch-to-symbol

**作用**

- 将 Torch 方言中的“符号”相关操作改写为 `Symbol` 方言的对应原语，统一动态形状与符号约束到 `symbol` 体系，便于后续形状传播、推断与规范化
- 目标是把 Torch 的 `SymbolicInt` 与“绑定符号形状”的语义显式化为 `symbol.symbolic_int` 与 `symbol.bind_symbolic_shape`，并完成必要的类型材质化（如 Torch int → `index`，value tensor → `ranked tensor`）

**具体转换**

- `torch.symbolic_int` → `symbol.symbolic_int`
  - 复用 Torch 符号名生成 `FlatSymbolRefAttr`，创建 `symbol::SymbolicIntOp`，保留 `min/max` 区间约束
  - 通过 Torch 后端类型转换器把 `symbolic_int` 的 `index` 结果材质化为 Torch 的整数结果类型，替换原 op
  - 位置：`bishengir\lib\Conversion\TorchToSymbol\TorchToSymbol.cpp:52-55, 62-65, 41-67`
- `torch.bind_symbolic_shape` → `symbol.bind_symbolic_shape`
  - 先检查待绑定的 Torch `vtensor` 能转为内建 `RankedTensorType`
  - 用类型转换器把每个形状符号材质化为 `index`，同时把 `vtensor` 材质化为 `ranked tensor`，创建 `symbol::BindSymbolicShapeOp` 并替换
  - 位置：`bishengir\lib\Conversion\TorchToSymbol\TorchToSymbol.cpp:83-87, 95-101, 101-104, 73-106`

**合法性与依赖**

- 标记非法 Torch 符号类操作：`Torch::SymbolicIntOp`, `Torch::BindSymbolicShapeOp`
- 合法方言：`symbol::SymbolDialect`, `arith::ArithDialect`
- 通过 Torch-MLIR 后端类型转换配置依赖与类型材质化：`bishengir\lib\Conversion\TorchToSymbol\TorchToSymbol.cpp:112-114, 134-139, 145-150`

**管线位置**

- 在 Torch 后端到命名算子管线中，先执行符号转换，再进行 HFusion/Linalg 转换与规范化
  - 调用顺序：`createConvertTorchToSymbolPass()` → `createConvertTorchToHFusionPass()` → `createConvertTorchToLinalgPass()`
  - 位置：`bishengir\lib\Dialect\Torch\Pipelines\TorchPipelines.cpp:71-78`

**文件/符号**

- 定义：`bishengir/include/bishengir/Conversion/Passes.td:157-158`
- 实现与构造器：`bishengir\lib\Conversion\TorchToSymbol\TorchToSymbol.cpp:141-150, 152-155`
- 模式注册接口：`bishengir\include\bishengir\Conversion\TorchToSymbol\TorchToSymbol.h:28-32, 38-39`

**为什么需要**

- 把动态形状与符号表达统一到 `symbol` 方言，有利于后续形状相关优化与跨方言协同；并提前完成 Torch 专用类型到 MLIR 内建类型的材质化，避免在 HFusion/Linalg 阶段处理 Torch 特有语义，提高转换稳定性与可优化性

# Pass 定义文件：`Dialect_Annotation_Transforms_Passes.td`

## annotation-lowering

- 作用概述
- 该 pass 用于将 Annotation 方言从 IR 中“下沉/剔除”，把仅用于分析或优化提示的标注操作移除，确保后续管线不再依赖 Annotation 方言。
- 工作方式
- 将整个 annotation::AnnotationDialect 标记为非法目标，要求转化后 IR 不含该方言中的任何操作： bishengir/lib/Dialect/Annotation/Transforms/Lowering.cpp:54
- 为 annotation::MarkOp 注册一个简单的转换模式：匹配到即删除，不改变数据/控制流： bishengir/lib/Dialect/Annotation/Transforms/Lowering.cpp:32-43
- 使用部分转换应用这些模式；若仍残留 Annotation 操作则报错： bishengir/lib/Dialect/Annotation/Transforms/Lowering.cpp:58-60
- Pass 声明与构造器位置： bishengir/include/bishengir/Dialect/Annotation/Transforms/Passes.td:23-26 ， bishengir/lib/Dialect/Annotation/Transforms/Lowering.cpp:64-66
- 何时运行
- 通常在下游后端/LLVM 降级前的清理阶段运行，确保 IR 只包含目标方言。例如 CPU Runner 管线中在内存/循环转换之前调用： bishengir/lib/ExecutionEngine/ExecutionEnginePipelines.cpp:65
- 背景与影响
- annotation::MarkOp 用于给值附加静态或动态属性（如缓冲大小、对齐、复用等），供上游优化与 HIVM/融合管线消费：参考其语义与折叠逻辑 bishengir/lib/Dialect/Annotation/IR/AnnotationOps.cpp:37-112
- 该 pass 不更改计算，仅移除标注；因此应在相关标注已被其它 pass 使用完毕后再执行，以免丢失优化信息。
- 目前实现只显式处理并删除 MarkOp ；若未来 Annotation 方言新增其它操作而未提供转换规则，因方言被设为非法会导致转换失败，从而提醒补充相应 lowering 模式。

# Pass 定义文件：`Dialect_HACC_Transforms_Passes.td`

## hacc-rename-func

该 pass 的作用是：根据函数上的 `hacc.rename_func` 属性，将该函数重命名为目标名，并在整个模块范围内把所有对该函数的符号引用统一更新。

- 触发条件：函数带有 `hacc.rename_func = #hacc.rename_func<@目标名>` 属性
- 核心逻辑：
  - 校验目标名非空
  - 校验模块内不存在同名函数，否则报错并使 pass 失败
  - 用符号表在模块内替换所有旧符号引用（包括 `func.call`、属性中的 `callee`、`SymbolRefAttr` 等）
  - 更新函数的 `sym_name`，并移除 `hacc.rename_func` 属性
- 适用范围：模块级遍历所有 `func.func`，一次性在模块根上完成符号替换
- 约束：目标名不能与现有函数重名；目标名必须是合法的 `FlatSymbolRefAttr`

代码参考

- 定义与说明：`bishengir\include\bishengir\Dialect\HACC\Transforms\Passes.td:23-58`
- 属性定义：`bishengir\include\bishengir\Dialect\HACC\IR\HACCAttrs.td:319-327`
- 实现逻辑：`bishengir\lib\Dialect\HACC\Transforms\RenameFunc.cpp:36-73`
- 测试示例：`bishengir\test\Dialect\HACC\rename-func.mlir:1-30`

使用示例

- 输入（带属性的待重命名函数与引用者）：
  ```
  func.func @bar() attributes {hacc.rename_func = #hacc.rename_func<@foo>} { ... }
  func.func @caller() { func.call @bar() : () -> (); ... }
  ```
- 运行后：
  ```
  func.func @foo() { ... }
  func.func @caller() { func.call @foo() : () -> (); ... }
  ```

命令行运行

- 在 Windows 上可以用：
  ```
  bishengir-opt.exe -hacc-rename-func path\to\input.mlir
  ```
- 若模块里已存在 `@foo`，pass 会报错并终止（见测试用例中的期望错误）。

## hacc-append-device-spec

**作用概述**

- 将目标 NPU 设备的规格信息以 `#hacc.target_device_spec<...>` 形式附加到模块的 DataLayout 中，用于后续编译/优化阶段查询硬件参数
- 设备来源优先级：优先使用 pass 选项 `target`，其次尝试模块上的 `hacc.target` 属性；两者都缺省或为 `Unknown` 时不做任何事
- 若模块已有设备规格，则发出覆盖警告并重新写入；写入后移除模块上的 `hacc.target` 属性

**行为细节**

- 选择最终设备：
  - 若同时指定了选项与 IR 属性且不一致，发出覆盖警告，按选项值执行
  - 两者都为 `Unknown` 则直接返回
- 规格内容：
  - 为全部 `DeviceSpec` 枚举键（如 `AI_CORE_COUNT`, `UB_SIZE`, `L1_SIZE` 等）构造 `#dlti.dl_entry<key, value>` 并汇聚到 `#hacc.target_device_spec<...>`
  - 规格数据来源于生成文件 `NPUTargetSpec.cpp.inc` 的查询接口
- 依赖：
  - 需要 `hacc::HACCDialect` 和 `mlir::DLTIDialect`（DataLayout 接口）
- 副作用：
  - 最终在模块级别设置设备规格属性；移除原始目标设备字符串属性 `#hacc.target<"...">`

**代码参考**

- Pass 定义与选项：`bishengir\include\bishengir\Dialect\HACC\Transforms\Passes.td:60-91`
- 设备规格属性定义：`bishengir\include\bishengir\Dialect\HACC\IR\HACCAttrs.td:268-288`
- 规格枚举键：`bishengir\include\bishengir\Dialect\HACC\IR\HACCAttrs.td:243-256`
- 规格构造与附加逻辑：`bishengir\lib\Dialect\HACC\Transforms\AppendDeviceSpec.cpp:50-67`、`71-108`
- 生成规格表来源：`bishengir\lib\Dialect\HACC\Transforms\AppendDeviceSpec.cpp:25`

**使用示例**

- 在模块上用属性指定设备（可选）：
  ```
  // 模块属性
  // module attributes { hacc.target = #hacc.target<"Ascend910B1"> }
  ```
- 命令行运行（Windows）：
  ```
  bishengir-opt.exe input.mlir -hacc-append-device-spec --target=Ascend910B1
  ```
  - 若同时存在模块属性与命令行选项且不一致，将按选项值覆盖，并发出警告
- 运行后模块会携带：
  ```
  #hacc.target_device_spec<
    #dlti.dl_entry<"UB_SIZE", ...>,
    #dlti.dl_entry<"AI_CORE_COUNT", ...>,
    ...
  >
  ```
  并移除 `hacc.target` 字符串属性

**适用场景**

- 需要在 IR 中显式嵌入目标设备规格，供后续 pass（如 tiling、资源分配、调度）读取统一的硬件参数集合
- 统一设备描述来源，避免下游对设备信息的分散解析与重复配置

# Pass 定义文件：`Dialect_HFusion_Transforms_Passes.td`

## hfusion-fuse-ops

**作用概述**

- 该条目在 `bishengir\include\bishengir\Dialect\HFusion\Transforms\Passes.td:23-59` 定义了一个模块级优化 pass：`hfusion-fuse-ops`。它的职责是根据 HFusion 的融合策略，把同一函数内的可融合的张量算子分组，并“外提”为一个或多个设备核函数，从而减少中间张量的物化、降低内存流量并提升执行效率。
- 该 pass 的实现位于 `bishengir\lib\Dialect\HFusion\Transforms\OpFusion.cpp:236-321`。核心流程是：对每个 `func.func`，根据函数上的 `hfusion.fusion_kind` 标注或推断到的融合模式，分析可融合的算子块，然后用 `FusibleBlockOutliner` 将其“外提”为设备核函数，并在原 host 函数中插入调用。
- 在 HFusion 总流水线中，这个 pass 通常在“标注融合类型”和若干规范化后执行，见 `bishengir\lib\Dialect\HFusion\Pipelines\HFusionPipelines.cpp:177-189`。

**融合范围**

- 可融合的算子类别由融合模式控制，判定逻辑见 `bishengir\lib\Dialect\HFusion\Transforms\OpFusion\FusibleHelper.cpp:364-433`：
  - `PURE_ELEMWISE`：元素级一元/二元算子链（以及零秩元素算子）
  - `ANY_PB`：元素级 + 广播
  - `LAST_AXIS_PBR/ANYPBR`：元素级 + 广播 + 规约（限制规约轴）
  - `SHALLOWVV/SHALLOWCV/MIXCV/MIXC2`：包含向量/矩阵类算子（如 `linalg.matmul`）的浅融合或混合融合
- 多核模式下会按固定顺序尝试多种融合模式，见 `bishengir\lib\Dialect\HFusion\Transforms\OpFusion.cpp:68-129`。

**关键选项**

- `output-mode`（默认 Multiple）：控制外提核函数的输出形式
  - `multi`：多个结果各自返回
  - `single`：把多个结果聚合为单返回（在 MIXCV 等场景中会强制 single）
  - `single-aggr`：更激进的单返回，允许为融合复制部分计算
- `fusion-mode`：融合模式枚举，通常由函数属性 `hfusion.fusion_kind` 提供；pass 会读取该标签并据此调整策略，见 `bishengir\lib\Dialect\HFusion\Transforms\OpFusion.cpp:56-66`
- `always-inline`：是否始终内联外提函数到调用点
- `move-out-to-param`（默认 true）：是否把核函数的结果改为“通过参数输出”（out-params），影响调用签名与资源管理
- `max-horizontal-fusion-size`：限制“水平融合”（彼此无依赖的并列算子）规模
- `multi-kernel`：允许在一个 host 函数中外提多个设备核函数，见 `bishengir\lib\Dialect\HFusion\Transforms\OpFusion.cpp:294-310`

**简单示例（元素级融合）**

- 输入（一个 Host 函数，设置 `PURE_ELEMWISE`，三段元素级链）：

```mlir
module {
  func.func @example(%a: tensor<1024xf32>, %b: tensor<1024xf32>, %c: tensor<1024xf32>)
      -> tensor<1024xf32>
      attributes {hacc.function_kind = #hacc.function_kind<HOST>,
                  hfusion.fusion_kind = #hfusion.fusion_kind<PURE_ELEMWISE>} {
    %tmp0 = tensor.empty() : tensor<1024xf32>
    %mul = linalg.elemwise_binary {fun = #linalg.binary_fn<mul>}
           ins(%a, %b : tensor<1024xf32>, tensor<1024xf32>)
           outs(%tmp0 : tensor<1024xf32>) -> tensor<1024xf32>
    %tmp1 = tensor.empty() : tensor<1024xf32>
    %add = linalg.elemwise_binary {fun = #linalg.binary_fn<add>}
           ins(%mul, %c : tensor<1024xf32>, tensor<1024xf32>)
           outs(%tmp1 : tensor<1024xf32>) -> tensor<1024xf32>
    return %add : tensor<1024xf32>
  }
}
```

- 应用 `hfusion-fuse-ops` 后的常见结果（外提为设备核函数，host 内用 `call`）：

```mlir
module {
  func.func @example_fused_0(%a: tensor<1024xf32>, %b: tensor<1024xf32>, %c: tensor<1024xf32>, %out: tensor<1024xf32>)
      -> tensor<1024xf32>
      attributes {hacc.entry, hacc.function_kind = #hacc.function_kind<DEVICE>,
                  hfusion.fusion_kind = #hfusion.fusion_kind<PURE_ELEMWISE>} {
    %tmp = linalg.elemwise_binary {fun = #linalg.binary_fn<mul>}
           ins(%a, %b : tensor<1024xf32>, tensor<1024xf32>)
           outs(%out : tensor<1024xf32>) -> tensor<1024xf32>
    %ret = linalg.elemwise_binary {fun = #linalg.binary_fn<add>}
           ins(%tmp, %c : tensor<1024xf32>, tensor<1024xf32>)
           outs(%out : tensor<1024xf32>) -> tensor<1024xf32>
    return %ret : tensor<1024xf32>
  }

  func.func @example(%a: tensor<1024xf32>, %b: tensor<1024xf32>, %c: tensor<1024xf32>)
      -> tensor<1024xf32>
      attributes {hacc.function_kind = #hacc.function_kind<HOST>} {
    %out = tensor.empty() : tensor<1024xf32>
    %r = call @example_fused_0(%a, %b, %c, %out)
         : (tensor<1024xf32>, tensor<1024xf32>, tensor<1024xf32>, tensor<1024xf32>)
           -> tensor<1024xf32>
    return %r : tensor<1024xf32>
  }
}
```

- 如果把选项改为 `--hfusion-fuse-ops="move-out-to-param=false,output-mode=single"`，核函数的签名将更倾向直接“返回”结果而不是通过 out 参数，调用也无需再传 `%out`。

**如何运行**

- 直接在命令行对 MLIR 文件应用该 pass（Windows 路径示例）：

```bat
bishengir-opt --hfusion-fuse-ops="output-mode=single,move-out-to-param=false" ^
  bishengir\test\Dialect\HFusion\OpFusion\test_fused.mlir ^
  -o tmp/out.mlir
```

- 若使用 HFusion 预置流水线，可将其包含在 `lower-hfusion-pipeline` 中，该流水线内部会构造 `HFusionOpFusionOptions` 并调用 `createHFusionOpFusionPass`，见 `bishengir\lib\Dialect\HFusion\Pipelines\HFusionPipelines.cpp:177-189`。

**相关代码位置**

- Pass 定义与选项：`bishengir\include\bishengir\Dialect\HFusion\Transforms\Passes.td:23-59`
- Pass 实现与调度策略：`bishengir\lib\Dialect\HFusion\Transforms\OpFusion.cpp:56-129, 236-321`
- 流水线集成：`bishengir\lib\Dialect\HFusion\Pipelines\HFusionPipelines.cpp:177-189`
- 可融合模式判定：`bishengir\lib\Dialect\HFusion\Transforms\OpFusion\FusibleHelper.cpp:364-433`

需要其他融合模式（如包含广播或规约）的示例，我也可以基于现有测试用例快速给出相应的前后 IR。

## hfusion-auto-schedule

**作用概述**

- 该条目在 `bishengir\include\bishengir\Dialect\HFusion\Transforms\Passes.td:62-90` 定义了模块级 pass：`hfusion-auto-schedule`，用于对前一步已“外提”的 HFusion 核函数进行自动调度（如分块/并行映射、多缓冲复用、DMA 缓冲管理、确定性计算等）。
- 实现位于 `bishengir\lib\Dialect\HFusion\Transforms\AutoSchedule\AutoScheduleBase.cpp:1191-1238`。该 pass 会遍历模块内的 `func.func`，按函数融合类型与用户选项配置调度参数，然后调用调度器执行具体的调度序列。
- 在流水线中的位置见 `bishengir\lib\Dialect\HFusion\Pipelines\HFusionPipelines.cpp:216-241`：它位于“Flatten/Cache/Canonicalize”之后，且在“Decompose/Pack/SCF 优化”等后续步骤之前。

**主要能力**

- 为不同融合类型选择合适的调度策略并应用到核函数（依赖 `transform::TransformDialect`），如对规约/矩阵/向量类算子进行分块、循环转换/映射到硬件 block。
- 资源与缓冲优化：支持多缓冲（pipeline overlap）、DMA 缓冲计数优化、主机侧资源管理提示。
- 确定性选项：启用确定性计算时避免非确定性并行复用路径。
- 针对特定融合类型的细化：例如 `MixCV`/`SingleCube` 会调整 `blockDim`（“cube 与 vector 1:2”），见 `AutoScheduleBase.cpp:1218-1226`。

**可用选项**

- 定义与说明见 `Passes.td:63-90`：
  - `block-dim`：调度块数量
  - `enable-auto-multi-buffer`：启用多缓冲自动化
  - `enable-deterministic-computing`：启用确定性计算
  - `max-buffer-count-tuning`：允许缓冲数量调优
  - `enable-count-buffer-dma-opt`：DMA 缓冲不与向量缓冲复用
  - `enable-manage-host-resources`：主机函数的资源管理
  - `cube-tiling-tuning`：cube 相关分块参数调优
  - `external-tiling-func-path`：外部分块函数
  - `enable-symbol-analysis`：符号分析辅助分块/融合

**管线集成**

- 在 HFusion 总流水线中，自动调度由 `createHFusionAutoSchedulePass` 添加，见 `HFusionPipelines.cpp:240-241`。其前后还会配合“规约/转置分解”“常量化/打包分块参数”“SCF 规范化”等，形成完整的调度与后处理链。

**简单示例（如何运行）**

- 示例 1：对已外提核函数进行自动调度（命令行）

```bat
:: 对 MLIR 输入执行自动调度（仅演示 AutoSchedule）
bishengir-opt --pass-pipeline="builtin.module(hfusion-auto-schedule)" ^
  bishengir\test\Dialect\HFusion\AutoSchedule\test-last-axis-pbr.mlir ^
  -o tmp/scheduled.mlir
```

- 示例 2：在完整 HFusion 流水线中包含自动调度（会自动创建选项并调用）

```bat
:: 使用预置流水线（包含外提+自动调度+后处理）
bishengir-opt -pass-pipeline="builtin.module(lower-hfusion-pipeline)" ^
  bishengir\test\Dialect\HFusion\OpFusion\test_fused.mlir ^
  -o tmp/lowered.mlir
```

- 如果启用调试打印（例如在测试里），你会在输出中看到调度序列（如 `transform.loop.tile`、`transform.loop.for_to_forall ... mapping = [#hivm.block]`），对应自动分块与并行映射，参见测试注释 `bishengir\test\Dialect\HFusion\AutoSchedule\test-last-axis-pbr.mlir:79`。

**效果预期（概念说明）**

- 对元素级链：可能标注或重排以适配硬件线程块，不一定产生显式循环。
- 对规约/矩阵类算子：会加入分块/循环映射（tile + forall/block），并按选项进行多缓冲与 DMA 复用控制。
- 后续的打包/常量化/SCF 规范化 pass 会把调度参数固化为常量、简化循环结构，见 `HFusionPipelines.cpp:200-214`。

**相关代码位置**

- 定义与说明：`bishengir/include/.../Passes.td:62-90`
- 流水线集成：`bishengir/lib/.../HFusionPipelines.cpp:216-241`
- 实现入口：`bishengir/lib/.../AutoScheduleBase.cpp:1191-1238`
- 调度解释器选项：`bishengir/lib/.../AutoScheduleInterpreter.cpp:143-174`

如果你需要某个具体融合类型（例如 `LAST_AXIS_PBR` 规约或 `MIX_CV` 混合）下的调度前后 IR 对比，我可以基于现有测试用例进一步给出对应的最小示例与运行指令。

## hfusion-add-ffts-addr

**作用概述**

- 该 pass 在 `ModuleOp` 级为设备入口核函数添加一条 `FFTS` 基址参数，并在整条调用链上同步更新调用点与上层函数的参数签名，用于后续运行时或编译链路（如 Triton 编译）统一传递基础地址。
- 定义与选项在 `bishengir/include/.../Passes.td:92-102`，实现位于 `bishengir/lib/.../Transforms/AddFFTSAddr.cpp:41-131`。在 HFusion 流水线中位于后处理阶段，见 `bishengir/lib/.../HFusionPipelines.cpp:263-269`。

**插入策略**

- 仅处理设备入口函数：通过 `hacc.entry` 判断，见 `AddFFTSAddr.cpp:120-126`。
- 插入位置与触发条件，见 `AddFFTSAddr.cpp:99-115`：
  - 显式选项 `force-add-ffts-addr>=0` 时，按指定位置插入（当前实现验证后固定插入到位置 `0`）。
  - 未显式指定时，若融合类型为 `ShallowCV/MixCV/MixC2`，默认在位置 `0` 插入。
- 参数类型与标注：插入一个 `i64` 参数并附加 `HACC` 的 kernel-arg 标注，类型为 `FFTSBaseAddr`，见 `AddFFTSAddr.cpp:83-90`。
- 递归更新调用链：为所有调用该被修改函数的上层函数也添加同位置参数，并重建 `call` 的操作数序列，将新参数在同索引处向下传递，见 `AddFFTSAddr.cpp:49-73`。

**简单示例**

- 输入（设备入口核函数 + Host 调用，不含 FFTS 基址）：

```mlir
module {
  func.func @kernel(%a: tensor<1024xf32>, %b: tensor<1024xf32>)
      -> tensor<1024xf32>
      attributes {hacc.entry, hacc.function_kind = #hacc.function_kind<DEVICE>,
                  hfusion.fusion_kind = #hfusion.fusion_kind<SHALLOW_CV>} {
    %0 = linalg.elemwise_binary {fun = #linalg.binary_fn<add>}
         ins(%a, %b : tensor<1024xf32>, tensor<1024xf32>)
         outs(%a : tensor<1024xf32>) -> tensor<1024xf32>
    return %0 : tensor<1024xf32>
  }

  func.func @host(%a: tensor<1024xf32>, %b: tensor<1024xf32>)
      -> tensor<1024xf32>
      attributes {hacc.function_kind = #hacc.function_kind<HOST>} {
    %r = call @kernel(%a, %b)
         : (tensor<1024xf32>, tensor<1024xf32>) -> tensor<1024xf32>
    return %r : tensor<1024xf32>
  }
}
```

- 应用 `hfusion-add-ffts-addr` 后（为入口核函数及其调用方统一插入 `i64` 基址参数并更新 `call`）：

```mlir
module {
  func.func @kernel(%ffts_base: i64, %a: tensor<1024xf32>, %b: tensor<1024xf32>)
      -> tensor<1024xf32>
      attributes {hacc.entry, hacc.function_kind = #hacc.function_kind<DEVICE>,
                  hfusion.fusion_kind = #hfusion.fusion_kind<SHALLOW_CV>} {
    %0 = linalg.elemwise_binary {fun = #linalg.binary_fn<add>}
         ins(%a, %b : tensor<1024xf32>, tensor<1024xf32>)
         outs(%a : tensor<1024xf32>) -> tensor<1024xf32>
    return %0 : tensor<1024xf32>
  }

  func.func @host(%ffts_base: i64, %a: tensor<1024xf32>, %b: tensor<1024xf32>)
      -> tensor<1024xf32>
      attributes {hacc.function_kind = #hacc.function_kind<HOST>} {
    %r = call @kernel(%ffts_base, %a, %b)
         : (i64, tensor<1024xf32>, tensor<1024xf32>) -> tensor<1024xf32>
    return %r : tensor<1024xf32>
  }
}
```

**运行命令**

- 单独运行该 pass（明确插入到第一个参数位置）：

```bat
bishengir-opt --hfusion-add-ffts-addr="force-add-ffts-addr=0" ^
  input.mlir -o tmp/out.mlir
```

- 在 HFusion 预置流水线中启用（当 `enableTritonKernelCompile` 时会设置 `force-add-ffts-addr=0`），见 `HFusionPipelines.cpp:263-269`：

```bat
bishengir-opt -pass-pipeline="builtin.module(lower-hfusion-pipeline)" ^
  input.mlir -o tmp/lowered.mlir
```

**相关代码位置**

- Pass 定义与选项：`bishengir/include/bishengir/Dialect/HFusion/Transforms/Passes.td:92-102`
- Pass 实现：`bishengir/lib/Dialect/HFusion/Transforms/AddFFTSAddr.cpp:41-131`
- 流水线集成：`bishengir/lib/Dialect/HFusion/Pipelines/HFusionPipelines.cpp:263-269`

## hfusion-convert-generic-to-named

**作用概述**

- 将可判定为“元素级计算”的 `linalg.generic` 转换为更语义化的“命名算子”，包括 `linalg.elemwise_unary/binary` 与 `hfusion.elemwise_unary/binary`，以便后续融合、调度和优化更精确。
- 定义位置：`bishengir\include\bishengir\Dialect\HFusion\Transforms\Passes.td:105-110`
- 实现位置：`bishengir\lib\Dialect\HFusion\Transforms\ConvertGenericToNamedOp.cpp:260-299`
- 流水线集成：该 pass 会在自动调度前后被使用，比如在 `HFusionPipelines.cpp:242-245` 里对自动调度后可能产生的 `generic` 再做一次转换。

**转换规则**

- 仅处理元素级的 `linalg.generic`（需要满足 `linalg::isElementwise(op)`），且 `yield` 中恰有一个“可识别”的计算原语（如 `arith.addf/mulf/subf/divf`、`math.sqrt` 等），见 `ConvertGenericToNamedOp.cpp:265-279`。
- 通过内置映射把 `yield` 的原语识别为一元或二元函数，并构造相应的 `fun` 属性：
  - Linalg 映射：`arith.addf/mulf/subf/divf` → `#linalg.binary_fn<...>`；`math.exp/absf/log` → `#linalg.unary_fn<...>`，见 `ConvertGenericToNamedOp.cpp:44-63`
  - HFusion 映射：`math.sqrt/rsqrt` → `#hfusion.unary_fn<...>`；特殊识别 `relu`（`arith.maximumf` + 常量 0）与 `rec`（`arith.divf` + 常量 1），见 `ConvertGenericToNamedOp.cpp:68-81, 149-159, 161-178`
- 输入/输出重构：根据一元/二元属性类型与 `generic` 输入数量，必要时重排/选择操作数，见 `ConvertGenericToNamedOp.cpp:195-226`。
- 生成目标算子：创建 `hfusion.elemwise_unary/binary` 或 `linalg.elemwise_unary/binary`，并以 `fun` 命名属性携带函数语义，见 `ConvertGenericToNamedOp.cpp:228-247`。

**简单示例**

- 示例 1：加法元素级 `generic` → Linalg 命名二元算子

```mlir
// 输入：元素级 generic，yield 为 arith.addf
%out = tensor.empty() : tensor<4xf32>
%r0 = linalg.generic
  { indexing_maps = [affine_map<(i)->(i)>, affine_map<(i)->(i)>, affine_map<(i)->(i)>],
    iterator_types = ["parallel"] }
  ins(%a, %b : tensor<4xf32>, tensor<4xf32>)
  outs(%out : tensor<4xf32>) {
    ^bb0(%xa: f32, %xb: f32, %yo: f32):
      %sum = arith.addf %xa, %xb : f32
      linalg.yield %sum : f32
  } -> tensor<4xf32>

// 输出：命名二元算子，fun = #linalg.binary_fn<add>
%r1 = linalg.elemwise_binary {fun = #linalg.binary_fn<add>}
       ins(%a, %b : tensor<4xf32>, tensor<4xf32>)
       outs(%out : tensor<4xf32>) -> tensor<4xf32>
```

- 示例 2：平方根元素级 `generic` → HFusion 命名一元算子

```mlir
// 输入：yield 为 math.sqrt
%out = tensor.empty() : tensor<4xf32>
%r0 = linalg.generic { ... } ins(%x : tensor<4xf32>) outs(%out : tensor<4xf32>) {
  ^bb0(%vx: f32, %yo: f32):
    %s = math.sqrt %vx : f32
    linalg.yield %s : f32
} -> tensor<4xf32>

// 输出：hfusion.elemwise_unary，fun = #hfusion.unary_fn<sqrt>
%r1 = hfusion.elemwise_unary {fun = #hfusion.unary_fn<sqrt>}
       ins(%x : tensor<4xf32>) outs(%out : tensor<4xf32>) -> tensor<4xf32>
```

- 示例 3：ReLU 识别（`maximumf(x, 0)`） → HFusion 命名一元算子

```mlir
// 输入：yield 为 arith.maximumf，且另一个操作数为常量 0
// 输出：fun = #hfusion.unary_fn<relu>
%r = hfusion.elemwise_unary {fun = #hfusion.unary_fn<relu>}
      ins(%x : tensor<4xf32>) outs(%out : tensor<4xf32>) -> tensor<4xf32>
```

- 示例 4：Reciprocal 识别（`divf(1, x)`） → HFusion 命名一元算子

```mlir
// 输入：yield 为 arith.divf，且分子为常量 1
// 输出：fun = #hfusion.unary_fn<rec>
%r = hfusion.elemwise_unary {fun = #hfusion.unary_fn<rec>}
      ins(%x : tensor<4xf32>) outs(%out : tensor<4xf32>) -> tensor<4xf32>
```

**使用方式**

- 仅运行该 pass（对函数级 IR）：

```bat
bishengir-opt --pass-pipeline="func.func(hfusion-convert-generic-to-named)" ^
  input.mlir -o tmp/out.mlir
```

- 在 HFusion 流水线中使用：自动调度后会再次调用该 pass，见 `bishengir\lib\Dialect\HFusion\Pipelines\HFusionPipelines.cpp:242-245`

**代码参考**

- Pass 定义与摘要：`bishengir/include/.../Passes.td:105-110`
- 映射与判定逻辑：`bishengir/lib/.../ConvertGenericToNamedOp.cpp:44-191, 195-247`
- 匹配与替换实现：`bishengir/lib/.../ConvertGenericToNamedOp.cpp:260-299`

## hfusion-flatten-ops

**作用概述**

- 将可“元素级”的 `linalg`/`hfusion` 算子在迭代维度上进行扁平化（flatten），把多维迭代合并为一维迭代，从而降低循环层级、利于后续融合与调度。
- 两种模式：
  - Greedy：基于模式重写，逐个把元素级算子扁平化（`isElementwise` 且非 `linalg.broadcast`），当 `numLoops>1` 时使用身份重关联 `[0,1,...]` 进行折叠，见 `lib/Dialect/HFusion/Transforms/FlattenOps.cpp:49-73, 88-99, 100-109`。
  - Tidy：对整函数做分析式扁平化（`Flattener`），可按需跳过 Host 函数、控制多动态形状处理，见 `FlattenOps.cpp:111-118`。
- 定义与选项：`bishengir/include/.../Passes.td:112-133`
  - `flatten-mode`：`greedy` 或 `tidy`
  - `skip-host`：在 `tidy` 模式下是否跳过 Host 函数
  - `multi-dynamic-shape`：是否在有多个动态维时也进行折叠
- 实现入口与模式注册：`FlattenOps.cpp:31-47, 88-99, 120-123`，注册了所有 Linalg/HFusion 结构化元素级算子以及 `linalg.generic`。

**关键逻辑**

- Greedy 模式中的元素级算子匹配与折叠：
  - 仅当 `numLoops>1`、`isElementwise(linalgOp)` 且不是 `linalg.broadcast`，才尝试折叠迭代维度，见 `FlattenOps.cpp:56-66`。
  - 通过 `collapseOpIterationDims` 把多维迭代变为一维，并保持结果类型不变；重写器会在算子周围插入 `tensor.collapse_shape`/`tensor.expand_shape` 来适配原始张量形状，见 `FlattenOps.cpp:66-71`。
- Tidy 模式使用 `Flattener` 对函数整体做扁平化分析，可避免贪心模式的局部最优导致的形状不一致，见 `FlattenOps.cpp:115-118`。
- 在 HFusion 流水线中，该 pass 位置靠前（Flatten→Canonicalize→Cache IO 等），见 `lib/Dialect/HFusion/Pipelines/HFusionPipelines.cpp:163-175`。

**简单示例**

- 输入（二维元素级加法）：

```mlir
%out = tensor.empty() : tensor<4x8xf32>
%r0 = linalg.elemwise_binary {fun = #linalg.binary_fn<add>}
      ins(%A, %B : tensor<4x8xf32>, tensor<4x8xf32>)
      outs(%out : tensor<4x8xf32>) -> tensor<4x8xf32>
```

- 应用 `hfusion-flatten-ops`（Greedy）后的常见结构（概念示意，保形）：

```mlir
%a_flat = tensor.collapse_shape %A [[0, 1]] : tensor<4x8xf32> into tensor<32xf32>
%b_flat = tensor.collapse_shape %B [[0, 1]] : tensor<4x8xf32> into tensor<32xf32>
%out_flat = tensor.empty() : tensor<32xf32>
%r1 = linalg.elemwise_binary {fun = #linalg.binary_fn<add>}
      ins(%a_flat, %b_flat : tensor<32xf32>, tensor<32xf32>)
      outs(%out_flat : tensor<32xf32>) -> tensor<32xf32>
%r0 = tensor.expand_shape %r1 [[0, 1]] output_shape [4, 8]
      : tensor<32xf32> into tensor<4x8xf32>
```

- 说明：
  - 扁平化把 2 维迭代合并为 1 维，算子内部循环层级变少。
  - 通过 `collapse_shape`/`expand_shape` 保持算子接口类型不变，便于后续融合与调度。

**运行命令**

- 仅扁平化（函数级）：

```bat
bishengir-opt --pass-pipeline="func.func(hfusion-flatten-ops)" ^
  input.mlir -o tmp/flattened.mlir
```

- 指定模式与选项（Greedy 或 Tidy）：

```bat
:: Greedy
bishengir-opt --hfusion-flatten-ops="flatten-mode=greedy" ^
  input.mlir -o tmp/flattened.mlir

:: Tidy，跳过 Host，关闭多动态形状折叠
bishengir-opt --hfusion-flatten-ops="flatten-mode=tidy,skip-host=true,multi-dynamic-shape=false" ^
  input.mlir -o tmp/flattened.mlir
```

**代码参考**

- 定义与选项：`bishengir\include\bishengir\Dialect\HFusion\Transforms\Passes.td:112-133`
- 模式重写与实现：`bishengir\lib\Dialect\HFusion\Transforms\FlattenOps.cpp:49-73, 88-123`
- 流水线调用位置：`bishengir\lib\Dialect\HFusion\Pipelines\HFusionPipelines.cpp:163-170`

## hfusion-tensor-results-to-out-params

**作用概述**

- 将函数的“张量返回值”改为“通过输出参数（out-params）写出”，并同步修复所有调用点的参数与结果使用，从而使设备核函数更贴近 in-place 写入模型，便于资源管理与后续优化。
- 定义与选项：`bishengir\include\bishengir\Dialect\HFusion\Transforms\Passes.td:136-150`
- 实现入口与核心流程：
  - 执行主体与签名修改：`bishengir\lib\Dialect\HFusion\Transforms\TensorResToOutParams.cpp:291-336`
  - 修复调用点：`bishengir\lib\Dialect\HFusion\Transforms\TensorResToOutParams.cpp:338-381`
  - 构造新增操作数规则：`bishengir\lib\Dialect\HFusion\Transforms\TensorResToOutParams.cpp:383-449`
  - 正式格式检查（输出参数数目与返回值数目一致）：`bishengir\lib\Dialect\HFusion\Transforms\TensorResToOutParams.cpp:486-538`

**关键行为**

- 为每个张量返回值插入对应的“输出参数”，并将目的风格算子（DestinationStyleOpInterface）的 `init` 写入目标改为这个新参数；若返回值有 `reshape/slice` 链，会反向重构以保持形状一致。
- 在所有调用点插入与新增输出参数匹配的实参：
  - 若调用结果未作为调用者的返回值使用，统一用 `tensor.empty` 作为输出缓冲。
  - 若调用结果被调用者返回：当调用者开启“管理主机资源”时仍使用 `tensor.empty`；否则为调用者也插入对应的输出参数并向下传递。
- 为新输出参数添加 `HACC` 属性（输出索引等），并在整个调用链中递归修复直到满足“输出参数数目 = 返回值数目”的格式。
- 选项：
  - `include-symbols`：仅对指定符号的函数执行
  - `enable-manage-host-resources`：调用者为 Host 且管理资源时，对动态形状不支持并强制使用 `tensor.empty`（参见 getResToOutCallOperand 逻辑）

**简单示例**

- 变换前（设备核返回一个张量，Host 直接接收）：

```mlir
module {
  func.func @kernel(%x: tensor<4xf32>) -> tensor<4xf32>
      attributes {hacc.function_kind = #hacc.function_kind<DEVICE>} {
    %out = tensor.empty() : tensor<4xf32>
    %r = linalg.elemwise_unary {fun = #linalg.unary_fn<exp>}
         ins(%x : tensor<4xf32>) outs(%out : tensor<4xf32>) -> tensor<4xf32>
    return %r : tensor<4xf32>
  }

  func.func @host(%x: tensor<4xf32>) -> tensor<4xf32>
      attributes {hacc.function_kind = #hacc.function_kind<HOST>} {
    %r = func.call @kernel(%x) : (tensor<4xf32>) -> tensor<4xf32>
    return %r : tensor<4xf32>
  }
}
```

- 变换后（设备核新增输出参数，调用点补齐输出缓冲；返回值仍保留，满足“个数一致”的要求）：

```mlir
module {
  func.func @kernel(%out0: tensor<4xf32>, %x: tensor<4xf32>) -> tensor<4xf32>
      attributes {hacc.function_kind = #hacc.function_kind<DEVICE>} {
    %r = linalg.elemwise_unary {fun = #linalg.unary_fn<exp>}
         ins(%x : tensor<4xf32>) outs(%out0 : tensor<4xf32>) -> tensor<4xf32>
    return %r : tensor<4xf32>
  }

  func.func @host(%x: tensor<4xf32>) -> tensor<4xf32>
      attributes {hacc.function_kind = #hacc.function_kind<HOST>} {
    %out0 = tensor.empty() : tensor<4xf32>
    %r = func.call @kernel(%out0, %x)
         : (tensor<4xf32>, tensor<4xf32>) -> tensor<4xf32>
    return %r : tensor<4xf32>
  }
}
```

- 说明：
  - 设备核的计算结果改为写入 `%out0`，但仍返回该结果以满足“输出参数数目 = 返回值数目”的格式检查。
  - Host 调用点为新增的输出参数准备了 `tensor.empty`；若 `enable-manage-host-resources=false` 且该结果被 Host 返回，也可为 Host 插入输出参数并把它向下传递。

**运行命令（Windows）**

```bat
:: 对全模块执行“张量返回值改为输出参数”并修复调用点
bishengir-opt --pass-pipeline="builtin.module(hfusion-tensor-results-to-out-params)" ^
  input.mlir -o tmp/out.mlir

:: 仅对指定符号函数执行，启用主机资源管理
bishengir-opt --hfusion-tensor-results-to-out-params="include-symbols=foo,enable-manage-host-resources=true" ^
  input.mlir -o tmp/out.mlir
```

**代码参考**

- 定义与摘要：`bishengir\include\bishengir\Dialect\HFusion\Transforms\Passes.td:136-150`
- 签名与 `init` 改写：`bishengir\lib\Dialect\HFusion\Transforms\TensorResToOutParams.cpp:291-336`
- 调用点修复与新增实参：`bishengir\lib\Dialect\HFusion\Transforms\TensorResToOutParams.cpp:338-381, 383-449`
- 统一格式校验：`bishengir\lib\Dialect\HFusion\Transforms\TensorResToOutParams.cpp:486-538`

## hfusion-inline-brc

**作用概述**

- 在函数级内，将“广播类”操作（`linalg.broadcast`、`linalg.fill`）若其输入是标量，直接内联到其后续的元素级计算中，避免先把标量扩成整张量再参与计算，减少中间张量与内存流量。
- 定义位置：`bishengir\include\bishengir\Dialect\HFusion\Transforms\Passes.td:152-155`
- 实现位置：`bishengir\lib\Dialect\HFusion\Transforms\InlineBrc.cpp:163-180`
- 流水线位置：
  - 早期阶段：`bishengir\lib\Dialect\HFusion\Pipelines\HFusionPipelines.cpp:107-115`（紧跟简化与规范化）
  - 后处理阶段：`bishengir\lib\Dialect\HFusion\Pipelines\HFusionPipelines.cpp:258-262`

**典型行为**

- 匹配 `linalg.broadcast` 或 `linalg.fill`，且输入为标量（含“scalar-like”的张量），将其结果在后续“元素级算子”里替换为该标量，消除广播/填充的中间张量：
  - 支持的使用者：`linalg.elemwise_unary/binary` 与 `hfusion.elemwise_unary/binary`，见 `InlineBrc.cpp:44-48`
  - 跨 `hfusion.cast`、`tensor.cast` 的使用链递归寻找可内联用户，见 `InlineBrc.cpp:69-87`
  - 若标量与使用点元素类型不同，会按元素类型选择合适的舍入模式执行 `hfusion.cast`，见 `InlineBrc.cpp:117-123`
  - 避免把“目的张量 init 本身”等价的输入在标量情形下替换（保持语义安全），见 `InlineBrc.cpp:59-67`

**简单示例**

- 示例 1：广播标量改为直接参与元素级计算

```mlir
// 变换前
%dst = tensor.empty() : tensor<8xf32>
%cst = arith.constant 2.0 : f32
%brc = linalg.broadcast ins(%cst : f32) outs(%dst : tensor<8xf32>) dimensions = [0]
%out = tensor.empty() : tensor<8xf32>
%add = linalg.elemwise_binary {fun = #linalg.binary_fn<add>}
       ins(%brc, %x : tensor<8xf32>, tensor<8xf32>)
       outs(%out : tensor<8xf32>) -> tensor<8xf32>

// 变换后（广播被内联为标量输入）
%out = tensor.empty() : tensor<8xf32>
%add = linalg.elemwise_binary {fun = #linalg.binary_fn<add>}
       ins(%cst, %x : f32, tensor<8xf32>)
       outs(%out : tensor<8xf32>) -> tensor<8xf32>
```

- 示例 2：填充标量改为直接参与元素级计算

```mlir
// 变换前
%dst = tensor.empty() : tensor<8xf32>
%cst = arith.constant 3.14 : f32
%filled = linalg.fill ins(%cst : f32) outs(%dst : tensor<8xf32>) -> tensor<8xf32>
%out = tensor.empty() : tensor<8xf32>
%mul = hfusion.elemwise_binary {fun = #hfusion.binary_fn<mul>}
       ins(%filled, %x : tensor<8xf32>, tensor<8xf32>)
       outs(%out : tensor<8xf32>) -> tensor<8xf32>

// 变换后（填充被内联为标量输入）
%out = tensor.empty() : tensor<8xf32>
%mul = hfusion.elemwise_binary {fun = #hfusion.binary_fn<mul>}
       ins(%cst, %x : f32, tensor<8xf32>)
       outs(%out : tensor<8xf32>) -> tensor<8xf32>
```

**运行命令**

```bat
:: 仅执行内联广播类算子（函数级）
bishengir-opt --pass-pipeline="func.func(hfusion-inline-brc)" ^
  input.mlir -o tmp/out.mlir

:: 放入 HFusion 早期流水线（可与规范化配合）
bishengir-opt --pass-pipeline="builtin.module(
  func.func(hfusion-simplify-ops),
  func.func(hfusion-inline-brc),
  func.func(hfusion-normalize-ops)
)" input.mlir -o tmp/out.mlir
```

**相关代码位置**

- 定义与摘要：`bishengir/include/.../Passes.td:152-155`
- 内联逻辑与模式实现：`bishengir/lib/.../InlineBrc.cpp:39-126, 128-161, 163-180`
- 流水线集成与规范化顺序：`bishengir/lib/.../HFusionPipelines.cpp:107-115, 258-262`

## hfusion-outline-single-op

**作用概述**

- 在 Host 函数内，把“单个可外提算子”独立外提为一个设备核函数，并在原函数里改为 `call` 调用，便于后续调度、资源管理和设备侧优化
- 只处理 Host 函数；若函数仅“直接返回”而没有可融合算子，会为所有返回值统一外提一个“直返”设备核函数
- 定义与选项：`bishengir\include\bishengir\Dialect\HFusion\Transforms\Passes.td:157-165`
  - `move-out-to-param`：是否把核函数结果改为通过参数输出（out-params）

**实现要点**

- 入口逻辑：`bishengir\lib\Dialect\HFusion\Transforms\OutlineSingleOp.cpp:47-63`
  - Host 函数且“仅返回”的情况：调用 `outlineAllReturnValue` 外提一个“直返”核函数，见 `OutlineSingleOp.cpp:65-94`
  - 其它情况：调 `outlineSingleFusedFuncs` 为每个可外提算子创建一个设备核函数，见 `bishengir\lib\Dialect\HFusion\Transforms\OpFusion.cpp:172-200`
- 外提设备核函数时，设置设备入口属性与融合类型（若能推断），见 `OutlineSingleOp.cpp:77-83`
- 流水线位置：在融合与规范化后作为补充外提步骤，见 `bishengir\lib\Dialect\HFusion\Pipelines\HFusionPipelines.cpp:190-194`

**简单示例（元素级加法）**

- 变换前（Host 内有一个元素级二元加法）：

```mlir
func.func @host_elemwise(%a: tensor<?xf32>, %b: tensor<?xf32>, %out: tensor<?xf32>)
    -> tensor<?xf32>
    attributes {hacc.function_kind = #hacc.function_kind<HOST>} {
  %r = linalg.elemwise_binary {fun = #linalg.binary_fn<add>}
       ins(%a, %b : tensor<?xf32>, tensor<?xf32>)
       outs(%out : tensor<?xf32>) -> tensor<?xf32>
  return %r : tensor<?xf32>
}
```

- 变换后（生成设备核函数并在 Host 中调用）：

```mlir
// 设备核函数（入口 + 融合类型标注）
func.func @host_elemwise_single_outlined_0(%a: tensor<?xf32>, %b: tensor<?xf32>, %out: tensor<?xf32>)
    -> tensor<?xf32>
    attributes {hacc.entry, hacc.function_kind = #hacc.function_kind<DEVICE>,
                hfusion.fusion_kind = #hfusion.fusion_kind<PURE_ELEMWISE>} {
  %r = linalg.elemwise_binary {fun = #linalg.binary_fn<add>}
       ins(%a, %b) outs(%out) -> tensor<?xf32>
  return %r : tensor<?xf32>
}

// Host 改为调用设备核
func.func @host_elemwise(%a: tensor<?xf32>, %b: tensor<?xf32>, %out: tensor<?xf32>)
    -> tensor<?xf32>
    attributes {hacc.function_kind = #hacc.function_kind<HOST>} {
  %r = func.call @host_elemwise_single_outlined_0(%a, %b, %out)
       : (tensor<?xf32>, tensor<?xf32>, tensor<?xf32>) -> tensor<?xf32>
  return %r : tensor<?xf32>
}
```

**简单示例（仅返回值的函数）**

- 变换前（Host 函数仅“返回形参”）：

```mlir
func.func @return_only(%x: tensor<?xf32>, %y: tensor<?x?xf32>)
    -> (tensor<?xf32>, tensor<?x?xf32>)
    attributes {hacc.function_kind = #hacc.function_kind<HOST>} {
  return %x, %y : tensor<?xf32>, tensor<?x?xf32>
}
```

- 变换后（外提“直返”设备核函数并在 Host 中调用）：

```mlir
func.func @return_only_single_outlined(%x: tensor<?xf32>, %y: tensor<?x?xf32>)
    -> (tensor<?xf32>, tensor<?x?xf32>)
    attributes {hacc.entry, hacc.function_kind = #hacc.function_kind<DEVICE>} {
  return %x, %y : tensor<?xf32>, tensor<?x?xf32>
}

func.func @return_only(%x: tensor<?xf32>, %y: tensor<?x?xf32>)
    -> (tensor<?xf32>, tensor<?x?xf32>)
    attributes {hacc.function_kind = #hacc.function_kind<HOST>} {
  %res:2 = func.call @return_only_single_outlined(%x, %y)
          : (tensor<?xf32>, tensor<?x?xf32>)
            -> (tensor<?xf32>, tensor<?x?xf32>)
  return %res#0, %res#1 : tensor<?xf32>, tensor<?x?xf32>
}
```

**运行命令（Windows）**

- 单独运行该 pass：

```bat
bishengir-opt --pass-pipeline="func.func(hfusion-outline-single-op)" ^
  input.mlir -o tmp/outlined.mlir
```

- 控制 out-params 行为：

```bat
bishengir-opt --hfusion-outline-single-op="move-out-to-param=false" ^
  input.mlir -o tmp/outlined.mlir
```

**代码参考**

- 定义与选项：`bishengir\include\bishengir\Dialect\HFusion\Transforms\Passes.td:157-165`
- 入口与外提逻辑：`bishengir\lib\Dialect\HFusion\Transforms\OutlineSingleOp.cpp:47-63, 65-94, 109-112`
- 单算子外提实现：`bishengir\lib\Dialect\HFusion\Transforms\OpFusion.cpp:172-200`
- 流水线集成：`bishengir\lib\Dialect\HFusion\Pipelines\HFusionPipelines.cpp:190-194`

## hfusion-simplify-ops

**作用概述**

- 对 HFusion/Linalg 等算子进行本地化简，移除冗余转换与恒等序列，做基础的代数化简，便于后续规范化、融合、自动调度与后端 Lowering。
- 核心覆盖：去冗余的 `cast` 链、循环内 `cast` 的位置优化、抵消的 `transpose` 链、二元逐元素算子的零/一化简。

**主要规则**

- Cast 链化简：在不跨浮点/整数类型的前提下，消除“往返”或冗余的 `hfusion.cast` 链（如 f32→f16→f32）并移除死 `cast`。见 `lib/Dialect/HFusion/Transforms/SimplifyOps.cpp:39-47, 49-99`
- 循环内 Cast 优化：对于 `scf.for` 中的简单 `cast`（`outs()` 由 `tensor.empty` 定义），把首个 `cast` 上移到循环前，把与之配对的 `cast` 下移到循环后，同时更新 `yield` 与迭代参数类型。见 `SimplifyOps.cpp:101-191`
- Transpose 抵消：若一串 `linalg.transpose` 的置换组合为恒等（例如在 2D 上连续两次 `[1,0]`），移除前导 `transpose`。见 `SimplifyOps.cpp:193-232`
- 代数化简（逐元素二元）：对 `add/sub/mul/div` 做常见恒等元消除
  - `x + 0 -> x`，`0 + x -> x`；`x - 0 -> x`；`x * 1 -> x`；`1 * x -> x`；`x / 1 -> x`
  - 同时识别来自 `fill`、`cast` 的常量一/零传播。见 `SimplifyOps.cpp:234-371`

**简单示例**

- 加法消零
  ```mlir
  // 之前
  %zero = arith.constant 0.0 : f32
  %empty = tensor.empty() : tensor<32xf32>
  %y = linalg.elemwise_binary {fun = #linalg.binary_fn<add>}
       ins(%x, %zero : tensor<32xf32>, f32)
       outs(%empty : tensor<32xf32>) -> tensor<32xf32>
  // 之后
  // %y 直接替换为 %x
  ```
- 乘法去一
  ```mlir
  // 之前
  %one = arith.constant 1.0 : f32
  %empty = tensor.empty() : tensor<32xf32>
  %y = linalg.elemwise_binary {fun = #linalg.binary_fn<mul>}
       ins(%x, %one : tensor<32xf32>, f32)
       outs(%empty : tensor<32xf32>) -> tensor<32xf32>
  // 之后
  // %y 直接替换为 %x
  ```
- Transpose 抵消
  ```mlir
  // 之前：对 2D 张量先 [1,0] 再 [1,0]，等价于恒等
  %e1 = tensor.empty() : tensor<2x3xf32>
  %t1 = linalg.transpose ins(%x : tensor<2x3xf32>)
         outs(%e1 : tensor<3x2xf32>) permutation = [1, 0]
  %e2 = tensor.empty() : tensor<3x2xf32>
  %t2 = linalg.transpose ins(%t1 : tensor<3x2xf32>)
         outs(%e2 : tensor<2x3xf32>) permutation = [1, 0]
  // 之后：消除链首 transpose
  // %t2 直接替换为 %x
  ```
- Cast 链化简
  ```mlir
  // 之前：f32 -> f16 -> f32 的往返
  %e16 = tensor.empty() : tensor<32xf16>
  %c1  = hfusion.cast ins(%x : tensor<32xf32>) outs(%e16 : tensor<32xf16>) // f32->f16
  %e32 = tensor.empty() : tensor<32xf32>
  %c2  = hfusion.cast ins(%c1 : tensor<32xf16>) outs(%e32 : tensor<32xf32>) // f16->f32
  // 之后：移除回向的 %c2，直接使用 %x，保留一次降精度 %c1（若仍有用）
  ```

**命令示例（Windows）**

- 运行化简：
  ```
  bishengir-opt.exe input.mlir -hfusion-simplify-ops
  ```

**管线中的位置**

- 在预处理/规范化阶段多次使用，以清理 IR 并为后续内核外提、Normalize、AutoSchedule 做准备
  - 见 `lib/Dialect/HFusion/Pipelines/HFusionPipelines.cpp:109-111, 139-141`

**代码参考**

- 规则与实现：`bishengir\lib\Dialect\HFusion\Transforms\SimplifyOps.cpp:39-47, 49-99, 101-191, 193-232, 234-371`
- Pass 声明：`bishengir\include\bishengir\Dialect\HFusion\Transforms\Passes.td:167-171`

## hfusion-normalize-ops

**作用概述**

- 将 HFusion/Linalg 运算规范化到稳定、易优化的形态，统一数据类型与形状表达，消除复杂或非标准算子组合，便于后续融合、AutoSchedule 与后端 Lowering。
- 典型归一化包括：把特殊函数改写为基础算子组合（如 `log1p`、`sin/cos`）、将“1 元素张量”转为标量参与计算、统一比较与选择的目标类型、在需要时将 f16 计算提升到 f32 后再回写。

**常见归一化**

- 函数改写
  - `hfusion.elemwise_unary{log1p}(x)` 归一化为 `log(x+1)`，见 `lib/Dialect/HFusion/Transforms/Normalize.cpp:790-842`
  - `hfusion.elemwise_unary{sin/cos}` 近似为泰勒展开，多步算子组合实现，并在 f16 输入时提升到 f32 计算后回写，见 `Normalize.cpp:315-390`、`401-486`
- 标量样式处理
  - 将仅含 1 个元素的 `dense` 张量/`memref` 常量转为纯标量，用于二元/一元/比较/选择/类型转换等算子，见 `Normalize.cpp:2591-2667`
  - `linalg.broadcast` 若输入是 1 元素常量，改写为 `linalg.fill`，见 `Normalize.cpp:2651-2667`
- 类型规范化
  - 比较算子：i8 比较统一转 f16；i32 比较（除 `veq/vne`）统一转 i64，比对后再按目标类型回写，见 `Normalize.cpp:2669-2716`
  - 结果类型回写：布尔、i8、i16 结果在替换后按目标类型安全回写（含溢出处理），见 `Normalize.cpp:2751-2808`
  - 精度提升：在若干一元/二元/累积类运算中，将 f16 输入转 f32 计算以提升数值稳定性，见 `Normalize.cpp:5618-5625`

**简单示例**

- 示例 1（log1p 归一化）
  - 之前：
    ```
    %empty = tensor.empty() : tensor<32xf32>
    %y = hfusion.elemwise_unary {fun = #hfusion.unary_fn<log1p>}
         ins(%x : tensor<32xf32>) outs(%empty : tensor<32xf32>) -> tensor<32xf32>
    ```
  - 之后（规范化为 ln(x+1)）：
    ```
    %empty1 = tensor.empty() : tensor<32xf32>
    %one = arith.constant 1.0 : f32
    %add = linalg.elemwise_binary {fun = #linalg.binary_fn<add>}
           ins(%x, %one : tensor<32xf32>, f32) outs(%empty1 : tensor<32xf32>) -> tensor<32xf32>
    %empty2 = tensor.empty() : tensor<32xf32>
    %y = linalg.elemwise_unary {fun = #linalg.unary_fn<log>}
         ins(%add : tensor<32xf32>) outs(%empty2 : tensor<32xf32>) -> tensor<32xf32>
    ```
- 示例 2（“1 元素张量”转标量）
  - 之前：
    ```
    %c1 = arith.constant dense<1.0> : tensor<1xf32>
    %empty = tensor.empty() : tensor<32xf32>
    %y = linalg.elemwise_binary {fun = #linalg.binary_fn<add>}
         ins(%x, %c1 : tensor<32xf32>, tensor<1xf32>) outs(%empty : tensor<32xf32>) -> tensor<32xf32>
    ```
  - 之后（改为标量参与二元运算）：
    ```
    %one = arith.constant 1.0 : f32
    %y = linalg.elemwise_binary {fun = #linalg.binary_fn<add>}
         ins(%x, %one : tensor<32xf32>, f32) outs(%empty : tensor<32xf32>) -> tensor<32xf32>
    ```

**管线位置**

- 在预处理阶段与 `inline-brc` 之后再次运行，用于将“标量-向量”模式规范为“向量-标量”，提升后续 Normalize/融合效果，见 `lib/Dialect/HFusion/Pipelines/HFusionPipelines.cpp:109-115`

**命令示例（Windows）**

- 运行规范化：
  ```
  bishengir-opt.exe input.mlir -hfusion-normalize-ops
  ```

**代码参考**

- Pass 声明：`bishengir\include\bishengir\Dialect\HFusion\Transforms\Passes.td:173-176`
- 入口与注册：`bishengir\lib\Dialect\HFusion\Transforms\Normalize.cpp:5674-5689`
- log1p 归一化：`bishengir\lib\Dialect\HFusion\Transforms\Normalize.cpp:790-842`
- sin/cos 归一化：`bishengir\lib\Dialect\HFusion\Transforms\Normalize.cpp:315-390`、`401-486`
- 标量样式归一化与 broadcast→fill：`bishengir\lib\Dialect\HFusion\Transforms\Normalize.cpp:2591-2667`
- 比较/类型回写/精度提升注册：`bishengir\lib\Dialect\HFusion\Transforms\Normalize.cpp:5618-5672`

## hfusion-pack-tiling-data

**作用概述**

- 将“动态算子调度产生的整型分块参数”从多个 `i64` 形态统一打包到一个 `memref<?xi64>` 结构中，便于在调用链中以一个参数传递，减少接口复杂度并提升主机-设备交互的稳定性
- 支持选择是否把“tiling key”也打包进结构；支持自动生成“查询打包结构大小”的主机函数以供外部系统/Host 侧调用
- 关键位置：`bishengir\include\bishengir\Dialect\HFusion\Transforms\Passes.td:189-213`，实现：`bishengir\lib\Dialect\HFusion\Transforms\PackTilingData.cpp`

**触发与选项**

- `includeSymbols`：只处理指定符号；为空则处理所有匹配的函数（Passes.td:197-203）
- `emitGetTilingStructSizeFunction`：为每个 kernel 生成一个 Host 函数返回 tiling struct 的长度，并把函数名标到对应 device 函数属性上（Passes.td:203-207，PackTilingData.cpp:950-990）
- `packTilingKey`：是否把 tiling key 也打包进 `memref`；关闭时，tiling key 作为 `!llvm.ptr` 直接写指针（Passes.td:208-212，PackTilingData.cpp:705-722, 662-679）

**处理流程**

- 收集“tiling 函数信息”和“device 函数映射”，以及基于注解的“调用者中的 tiling case 分组”信息（PackTilingData.cpp:372-458, 460-513）
- 对每个需要打包的 tiling 函数：
  - 重写 tiling 函数：新增 `tiling_struct` 参数，返回值清空；把原返回的每个 `i64` 写入该结构（PackTilingData.cpp:757-778）
  - 重写所有对应 device 函数：删除所有 `hacc.tiling_data` 实参，新增一个 `tiling_struct` 实参；在函数体开头将旧的参数读出并替换所有使用（PackTilingData.cpp:800-831）
  - 修复调用者（Host 或 Device 的上层）中“每个 tiling case 分组”的调用点：构造或插入 `tiling_struct` 值，统一修改分组内所有 `func.call` 的参数形态（PackTilingData.cpp:839-850, 852-948）
- 清理：删除空的 tiling 函数（仅有 `return` 且无结果）（PackTilingData.cpp:334-351，空判定见:50-60）

**关键函数**

- `collectTilingFuncInfo`：发现所有 tiling 函数、判空、推断结构大小、建立 tiling→device 映射并校验一致性（PackTilingData.cpp:372-458）
- `collectTilingCaseInfo`：通过 `annotation::MarkOp` 标记和 `scf.index_switch` 抽取“同一调用者内的 tiling case 分组”（PackTilingData.cpp:460-513）
- `analyzeTilingCaseCallSite`：检查每个 case 内的 device 调用是否一致，记录“资源是否由调用者管理”（PackTilingData.cpp:546-636）
- `packTilingForTilingFunc`：把所有返回的 `i64` 写入 `memref` 并清空返回（PackTilingData.cpp:757-778）
- `packTilingForDeviceFunc`：删除 tiling 参数，改为从 `tiling_struct` 中加载并替换（PackTilingData.cpp:800-831）
- `fixCallSitesOfTilingCaseInCaller`：两种资源管理场景下构造或插入 `tiling_struct`，并统一重写 case 内所有调用（PackTilingData.cpp:852-948）
- `doEmitGetTilingStructSizeFunction`：生成 `@get_tiling_struct_size_*` Host 函数并在 device 函数上设置属性（PackTilingData.cpp:950-990）

**输入/输出示例**

- 打包前（示意）：
  ```mlir
  // Host tiling 计算函数
  func.func @foo_tiling(%arg0: tensor<?x?xf16>) -> (i64, i64) {HOST}
    // ... 计算出 td0, td1
    return %td0, %td1 : i64, i64

  // 两个设备函数共享同一 tiling 函数
  func.func @foo_kernel_case0(%x: tensor<?x?xf16>, %td0: i64 {hacc.tiling_data}, %td1: i64 {hacc.tiling_data}) -> tensor<?x?xf16> {DEVICE, hacc.tiling_func = "foo_tiling"}
  func.func @foo_kernel_case1(%x: tensor<?x?xf16}, %td0: i64 {hacc.tiling_data}, %td1: i64 {hacc.tiling_data}) -> tensor<?x?xf16> {DEVICE, hacc.tiling_func = "foo_tiling"}

  // 调用者，按 tiling key 分派
  func.func @caller(%x: tensor<?x?xf16}, %td0: i64 {hacc.tiling_data}, %td1: i64 {hacc.tiling_data}) {HOST}
    %k = arith.index_castui %td0 : i64 to index
    %y = scf.index_switch %k -> tensor<?x?xf16>
    case 0 {
      %r = func.call @foo_kernel_case0(%x, %td0, %td1)
      scf.yield %r
    }
    case 1 {
      %r = func.call @foo_kernel_case1(%x, %td0, %td1)
      scf.yield %r
    }
    return %y
  ```
- 打包后（示意，`packTilingKey=true`）：
  ```mlir
  // Host tiling 计算函数：新增 memref 参数，返回清空
  func.func @foo_tiling(%arg0: tensor<?x?xf16}, %ts: memref<2xi64> {hacc.tiling_struct}) {HOST}
    // ... 计算出 td0, td1
    memref.store %td0, %ts[%c0]
    memref.store %td1, %ts[%c1]
    return

  // 设备函数：删除 i64 tiling 参数，新增 memref<2xi64> 参数
  func.func @foo_kernel_case0(%x: tensor<?x?xf16}, %ts: memref<2xi64> {hacc.tiling_struct}) -> tensor<?x?xf16> {DEVICE, hacc.tiling_func = "foo_tiling"}
  func.func @foo_kernel_case1(%x: tensor<?x?xf16}, %ts: memref<2xi64> {hacc.tiling_struct}) -> tensor<?x?xf16> {DEVICE, hacc.tiling_func = "foo_tiling"}

  // 调用者：统一传递 tiling_struct
  func.func @caller(%x: tensor<?x?xf16}, %ts: memref<2xi64> {hacc.tiling_struct}) {HOST}
    %k = memref.load %ts[%c0] : i64
    %k_idx = arith.index_castui %k : i64 to index
    %y = scf.index_switch %k_idx -> tensor<?x?xf16>
    case 0 {
      %r = func.call @foo_kernel_case0(%x, %ts)
      scf.yield %r
    }
    case 1 {
      %r = func.call @foo_kernel_case1(%x, %ts)
      scf.yield %r
    }
    return %y
  ```
- 关键位置参考：打包写入返回值（PackTilingData.cpp:757-769）、函数签名更新（771-778）、device 参数替换（819-829）、调用点重写（939-946）

**与其他 Pass 的关系**

- 与自动调度：单核/多核调度流程中在后置阶段调用本 Pass，用打包后的 `tiling_struct` 标记关键算子（SingleCubeSchedule.cpp:234-269）
- 与常量化：若部分 tiling 数据是常量，`hfusion-constantize-tiling-data` 会把常量内联并从参数中移除，减少结构长度与调用开销（ConstantizeTilingData.cpp:392-426；Passes.td:215-300）
- 与标注/识别：通过 `annotation::MarkOp` 和 `hacc::TilingFunctionAttr` 抽取 case 信息和函数映射（PackTilingData.cpp:460-474, 419-428）

**常见问题与排查**

- 设备函数与 tiling 函数不匹配：在 `collectTilingFuncInfo` 中校验失败（PackTilingData.cpp:454-456）
- 资源管理不一致：同一调用者内某些 case 从 BlockArg 取 tiling，而另一些从 `func.call` 返回取，触发一致性错误（PackTilingData.cpp:592-597）
- 嵌套 Host 调用暂不支持：当调用者仍被其他符号引用时，会报错提示不支持嵌套 Host（PackTilingData.cpp:907-913）
- 空 tiling 函数：仅 `return` 无结果，最终被删除（PackTilingData.cpp:334-351，判定：50-60）

**使用建议**

- Pass 顺序：在完成 kernel 选择与 case 分派后执行打包；如需减少结构长度，之后可接 `hfusion-constantize-tiling-data`
- 选项建议：
  - 多形状场景优先保留 `packTilingKey=true`，可确保调用者只需一个 `memref` 参数即可完成分派
  - 如外部框架需提前分配内存，打开 `emit-get-tiling-struct-size-function` 以生成 `@get_tiling_struct_size_*`（PackTilingData.cpp:950-990）
- 命令示例（Windows）：
  ```bat
  mlir-opt --hfusion-pack-tiling-data="emit-get-tiling-struct-size-function=true,pack-tiling-key=true" input.mlir -o output.mlir
  ```

**代码参考定位**

- 打包主流程入口：`bishengir\lib\Dialect\HFusion\Transforms\PackTilingData.cpp:290-351`
- tiling 函数打包实现：`PackTilingData.cpp:757-778`
- device 函数参数替换：`PackTilingData.cpp:800-831`
- 调用点重写（两种资源管理模式）：`PackTilingData.cpp:852-948`
- case 分组与注解解析：`PackTilingData.cpp:460-513`
- 生成查询结构大小函数：`PackTilingData.cpp:950-990`
- Pass 定义与选项：`bishengir\include\bishengir\Dialect\HFusion\Transforms\Passes.td:189-213`

如果你希望我进一步“走读实现细节（比如 IRMapping 的替换策略、tiling key 不打包时的指针读写路径）”或“给出一个可运行的小型 MLIR 片段并演示 pass 前后效果”，告诉我你的偏好，我可以补充对应的示例和说明。

## hfusion-constantize-tiling-data

**作用**

- 将计算出的“常量”tiling 数据从 `HOST` 侧的 tiling 函数下推到 `DEVICE` 侧内核函数中，直接以内联常量的形式使用
- 同步删掉这些常量对应的所有函数参数、返回值以及调用点，包含设备函数的所有调用者，沿调用链递归更新
- 保证设备函数形参中带 `hacc.tiling_data` 的参数顺序与 tiling 函数返回值顺序一致，从而能正确匹配并常量化
- 定义位置与示例可见 `bishengir\include\bishengir\Dialect\HFusion\Transforms\Passes.td:215-343`，构造器为 `mlir::hfusion::createConstantizeTilingDataPass()`，pass 名称为 `hfusion-constantize-tiling-data`

**关键假设**

- 共享同一 tiling 函数的所有设备函数，其 tiling 形参顺序完全一致
- 设备函数的 tiling 形参顺序与 tiling 函数的返回值顺序一一对应
- 仅当某个 tiling 返回值在 `HOST` 侧是编译期可解析的常量（如 `arith.constant`）时，才会被内联到 `DEVICE` 函数并在签名中移除

**简单示例（输入 → 输出）**

- 输入示例（tiling 返回一个变量和一个常量，设备核函数有两个 `hacc.tiling_data` 参数，`main` 传入两者）:

```mlir
func.func @tiling_func(%arg0: tensor<?x?xf16>) -> (i64, i64)
attributes {hacc.function_kind = #hacc.function_kind<HOST>} {
  %ret0 = "some_calc"() : () -> i64
  %ret1 = arith.constant 32 : i64
  return %ret0, %ret1 : i64, i64
}

func.func @device_kernel_tiling(%arg0: tensor<?x?xf16>,
                                %t0: i64 {hacc.tiling_data},
                                %t1: i64 {hacc.tiling_data}) -> tensor<?x?xf16>
attributes {hacc.function_kind = #hacc.function_kind<DEVICE>, hacc.tiling_func = "tiling_func"} {
  "use"(%t0) : (i64) -> ()
  "use"(%t1) : (i64) -> ()
  %out = "some_op"(%arg0) : (tensor<?x?xf16>) -> tensor<?x?xf16>
  return %out : tensor<?x?xf16>
}

func.func @main(%arg0: tensor<?x?xf16>,
                %t0: i64 {hacc.tiling_data},
                %t1: i64 {hacc.tiling_data}) -> tensor<?x?xf16>
attributes {hacc.function_kind = #hacc.function_kind<HOST>} {
  %res = func.call @device_kernel_tiling(%arg0, %t0, %t1)
       : (tensor<?x?xf16>, i64, i64) -> tensor<?x?xf16>
  return %res : tensor<?x?xf16>
}
```

- 运行 `hfusion-constantize-tiling-data` 后的输出（常量 32 被内联到设备函数；tiling 函数与设备函数签名及 `main` 调用统一缩减为一个 tiling 量）:

```mlir
func.func @tiling_func(%arg0: tensor<?x?xf16>) -> (i64)
attributes {hacc.function_kind = #hacc.function_kind<HOST>} {
  %ret0 = "some_calc"() : () -> i64
  return %ret0 : i64
}

func.func @device_kernel_tiling(%arg0: tensor<?x?xf16>,
                                %t0: i64 {hacc.tiling_data}) -> tensor<?x?xf16>
attributes {hacc.function_kind = #hacc.function_kind<DEVICE>, hacc.tiling_func = "tiling_func"} {
  "use"(%t0) : (i64) -> ()
  %t1_cst = arith.constant 32 : i64
  "use"(%t1_cst) : (i64) -> ()
  %out = "some_op"(%arg0) : (tensor<?x?xf16>) -> tensor<?x?xf16>
  return %out : tensor<?x?xf16>
}

func.func @main(%arg0: tensor<?x?xf16>,
                %t0: i64 {hacc.tiling_data}) -> tensor<?x?xf16>
attributes {hacc.function_kind = #hacc.function_kind<HOST>} {
  %res = func.call @device_kernel_tiling(%arg0, %t0)
       : (tensor<?x?xf16>, i64) -> tensor<?x?xf16>
  return %res : tensor<?x?xf16>
}
```

**定位参考**

- 概要行：`bishengir\include\bishengir\Dialect\HFusion\Transforms\Passes.td:217`
- 详细说明与完整示例：`bishengir\include\bishengir\Dialect\HFusion\Transforms\Passes.td:218-343`

## hfusion-infer-func-fusion-kind

**作用**

- 自动分析 `func::FuncOp` 的运算模式，推断该函数的融合类型，并写入 `hfusion.fusion_kind` 属性
- 若已有期望的融合类型（函数上已有 `hfusion.fusion_kind`），但与推断结果不同，会发出警告并用推断结果覆盖
- 主要用于为后续 `hfusion-fuse-ops` 按融合类型进行轮廓化/融合提供标签

**位置与接口**

- 定义：`bishengir\include\bishengir\Dialect\HFusion\Transforms\Passes.td:346-350`
- 实现：`bishengir\lib\Dialect\HFusion\Transforms\InferFuncFusionKind.cpp:112-140`
- 关键函数：`inferFuncFusionKind` 推断融合类型 `bishengir\lib\Dialect\HFusion\Transforms\InferFuncFusionKind.cpp:82-104`
- 依赖方言：`hfusion::HFusionDialect`

**实现要点**

- 仅分析“单基本块”函数；多基本块函数直接判为不可融合 `bishengir\lib\Dialect\HFusion\Transforms\InferFuncFusionKind.cpp:69-80`
- 收集“可外提”的重要算子（如元素算子、reduce、broadcast、matmul、transpose、extract\_slice 等），数量决定不同路径：
  - 无可外提算子 → 标记为 `AnyPB` `bishengir\lib\Dialect\HFusion\Transforms\InferFuncFusionKind.cpp:85-88`
  - 仅一个重要算子 → 按模式映射到融合类型：
    - 元素类/transpose/interleave → `PureElemwise`
    - 最后轴 reduce → `LastAxisPBR`
    - 其他 reduce → `AnyPBR`
    - matmul → `SingleCube`
    - extract\_slice / broadcast → `AnyPB`
      参考映射：`bishengir\lib\Dialect\HFusion\Transforms\OpFusion\FusibleHelper.cpp:40-73`
  - 多个重要算子 → 遍历所有融合类型，调用 `FusibleBlockAnalyzer::isFusible()` 验证；第一个可融合的类型即结果，否则 `Unknown` `bishengir\lib\Dialect\HFusion\Transforms\InferFuncFusionKind.cpp:96-104`
- 如函数已有 `hfusion.fusion_kind`，但与推断不一致，则发出警告并覆盖 `bishengir\lib\Dialect\HFusion\Transforms\InferFuncFusionKind.cpp:125-133`

**简单示例**

- 示例 1：纯元素算子链推断为 `PURE_ELEMWISE`

```mlir
func.func @model_0(%arg0: tensor<5x1xf32>) -> tensor<5x1xf32>
attributes {hacc.function_kind = #hacc.function_kind<HOST>} {
  %cst = arith.constant 1.0 : f32
  %tmp = tensor.empty() : tensor<5x1xf32>
  %a = linalg.elemwise_unary {fun = #linalg.unary_fn<negf>} ins(%arg0) outs(%tmp) -> tensor<5x1xf32>
  %b = linalg.elemwise_unary {fun = #linalg.unary_fn<exp>} ins(%a)    outs(%tmp) -> tensor<5x1xf32>
  %c = linalg.elemwise_binary {fun = #linalg.binary_fn<add>} ins(%b, %cst) outs(%tmp) -> tensor<5x1xf32>
  %d = linalg.elemwise_binary {fun = #linalg.binary_fn<div>} ins(%c, %cst)  outs(%tmp) -> tensor<5x1xf32>
  return %d : tensor<5x1xf32>
}
// 运行后：函数属性包含 `hfusion.fusion_kind = #hfusion.fusion_kind<PURE_ELEMWISE>`
```

参考测试：`bishengir\test\Dialect\HFusion\OpFusion\test_infer.mlir:1-13`

- 示例 2：仅广播，推断为 `ANY_PB`

```mlir
func.func @test_any_pb(%x: tensor<7x7x7xf32>, %y: tensor<7x7x7x7xf32>) -> tensor<7x7x7x7xf32>
attributes {hacc.function_kind = #hacc.function_kind<HOST>} {
  %out = linalg.broadcast ins(%x : tensor<7x7x7xf32>) outs(%y : tensor<7x7x7x7xf32>) dimensions = [3]
  return %out : tensor<7x7x7x7xf32>
}
// 运行后：`hfusion.fusion_kind = #hfusion.fusion_kind<ANY_PB>`
```

参考测试：`bishengir\test\Dialect\HFusion\OpFusion\test_infer.mlir:30-37`

- 示例 3：混合复杂但不可归类，推断为 `UNKNOWN`

```mlir
func.func @test_unknown(%a: tensor<?x?xf32>, %b: tensor<?x?xf32>) -> tensor<?x?xf32> {
  %tmp = tensor.empty(%c0, %c1) : tensor<?x?xf32>
  %e = linalg.matmul ins(%a, %b : tensor<?x?xf32>, tensor<?x?xf32>) outs(%tmp : tensor<?x?xf32>) -> tensor<?x?xf32>
  %f = linalg.transpose ins(%e) outs(%tmp) permutation = [0, 1]
  return %f : tensor<?x?xf32>
}
// 运行后：`hfusion.fusion_kind = #hfusion.fusion_kind<UNKNOWN>`
```

参考测试：`bishengir\test\Dialect\HFusion\OpFusion\test_infer.mlir:40-66`

**运行命令**

```bat
bishengir-opt --hfusion-infer-func-fusion-kind -split-input-file path\to\your.mlir
```

**补充说明**

- 通常应在 `hfusion-fuse-ops` 之前运行该推断，以便融合器依据 `hfusion.fusion_kind` 做出策略选择；`hfusion-fuse-ops` 的选项 `fusion-mode=Unknown` 时，会按函数标签决定融合策略 `bishengir\include\bishengir\Dialect\HFusion\Transforms\Passes.td:40-43`
- 某些后续优化也会根据融合类型进行参数插入等处理，例如 FFTS 基址插入在 `ShallowCV/MixCV/MixC2` 场景中优先靠前插入 `bishengir\lib\Dialect\HFusion\Transforms\AddFFTSAddr.cpp:102-115`

## adapt-triton-kernel

**作用**

- 将 Triton 适配器注入的辅助函数调用统一改写为 HFusion 原生算子，并标注 Triton 入口核的元数据，方便后续编译与调度
- 具体包括：设备入口标记、工作区/锁参数标记、循环并行绑定属性转换，以及 `print/assert/gather/cumsum/cumprod/sort` 等算子改写

**改写与标注**

- 入口核标记：若函数含 `global_kernel` 属性，则标记为设备入口并移除该属性，模块加 `hacc.triton_kernel` 元属性 bishengir\lib\Dialect\HFusion\Transforms\AdaptTritonKernel.cpp:359-375
- 参数标记：根据函数属性标记形参类型
  - `WorkspaceArgIdx` → 对应参数加 `hacc.arg_type=workspace` bishengir\lib\Dialect\HFusion\Transforms\AdaptTritonKernel.cpp:50-62
  - `SyncBlockLockArgIdx` → 对应参数加 `hacc.arg_type=sync_block_lock` bishengir\lib\Dialect\HFusion\Transforms\AdaptTritonKernel.cpp:64-76
- 算子改写（按函数名前缀匹配）
  - `triton_print(...)` → `hfusion.print`，携带 `prefix` 和 `hex` 属性 bishengir\lib\Dialect\HFusion\Transforms\AdaptTritonKernel.cpp:78-123
  - `triton_assert(x)` → `hfusion.assert`，读取 `msg` 字符串。若参数为 `tensor<i1>`，先做 `arith.extui` 到 `i8` bishengir\lib\Dialect\HFusion\Transforms\AdaptTritonKernel.cpp:125-176
  - `triton_gather(src, index, axis)` → `hfusion.gather`，`axis` 必须是 `arith.constant` bishengir\lib\Dialect\HFusion\Transforms\AdaptTritonKernel.cpp:178-220
  - `triton_cumsum(src, dim)` / `triton_cumprod(src, dim)` → `hfusion.cumsum`/`hfusion.cumprod`，`dim` 必须是 `arith.constant` bishengir\lib\Dialect\HFusion\Transforms\AdaptTritonKernel.cpp:222-269
  - `triton_sort(src, axis, descending)` → `hfusion.sort`，`axis/descending` 需为常量 bishengir\lib\Dialect\HFusion\Transforms\AdaptTritonKernel.cpp:271-312
- 并行绑定：`scf.for` 若带 `{bind_sub_block = true}`，规范化循环索引类型并移除该属性，改为在 `scf.for` 上加 `hfusion.bind_sub_block` bishengir\lib\Dialect\HFusion\Transforms\AdaptTritonKernel.cpp:314-342

**简单示例**

- 打印与采集改写

```mlir
// 输入
func.func private @triton_print(i32) attributes {hex = true, prefix = "PID: "}
func.func private @triton_gather(tensor<4x64xf32>, tensor<4x32xi32>, i32) -> tensor<4x32xf32>
func.func @kernel(%pid: i32, %a: memref<4x64xf32>, %idx: memref<4x32xi32>) {
  %t = bufferization.to_tensor %a : memref<4x64xf32>
  %i = bufferization.to_tensor %idx : memref<4x32xi32>
  func.call @triton_print(%pid) : (i32) -> ()
  %axis = arith.constant 1 : i32
  %g = func.call @triton_gather(%t, %i, %axis) : (...) -> tensor<4x32xf32>
  return
}

// 输出
func.func @kernel(%pid: i32, %a: memref<4x64xf32>, %idx: memref<4x32xi32>) {
  %t = bufferization.to_tensor %a : memref<4x64xf32>
  %i = bufferization.to_tensor %idx : memref<4x32xi32>
  hfusion.print "PID: " {hex = true} %pid : i32
  %init = tensor.empty() : tensor<4x32xf32>
  %g = hfusion.gather ins(%t, %i) outs(%init) axis = 1 -> tensor<4x32xf32>
  return
}
```

参考测试：打印 bishengir\test\Dialect\HFusion\hfusion-adapt-triton-kernel.mlir:27-43，采集 bishengir\test\Dialect\HFusion\hfusion-adapt-triton-kernel.mlir:46-71

- 并行绑定属性

```mlir
// 输入
scf.for %i = %c0 to %c2 step %c1 : i32 {bind_sub_block = true} {
  ...
}

// 输出（规范为 index，添加 hfusion.bind_sub_block）
scf.for %i = %c0_idx to %c2_idx step %c1_idx : index {
} {hfusion.bind_sub_block}
```

参考测试：bishengir\test\Dialect\HFusion\hfusion-adapt-triton-kernel.mlir:121-131

**运行命令**

```bat
bishengir-opt -adapt-triton-kernel path\to\your.mlir -split-input-file
```

**定位参考**

- 定义与摘要：bishengir\include\bishengir\Dialect\HFusion\Transforms\Passes.td:352-356
- 具体实现：bishengir\lib\Dialect\HFusion\Transforms\AdaptTritonKernel.cpp:41-75, 78-375

## hfusion-legalize-bf16

**作用**

- 在 `func::FuncOp` 内，将使用 `bf16` 元素类型的算子改写为在 `f32` 上计算，并在算子边界插回到 `bf16`，以提升数值稳定性或兼容性
- 仅对需要的算子进行改写；对部分算子保持原样，不做改写

**规则**

- 触发条件：操作数或结果的元素类型中出现 `bf16` bishengir\lib\Dialect\HFusion\Transforms\LegalizeBF16.cpp:41-48
- 不改写的算子：`hfusion.cast`、`linalg.fill`、`linalg.copy`、`linalg.matmul`、`linalg.batch_matmul`、`linalg.transpose`、`hfusion.load`、`hfusion.store`、`hfusion.bitcast`、`hfusion.select` bishengir\lib\Dialect\HFusion\Transforms\LegalizeBF16.cpp:50-56
- 改写方式（对可改写算子）:
  - 先将所有 `bf16` 操作数通过 `hfusion.cast` 转为 `f32` bishengir\lib\Dialect\HFusion\Transforms\LegalizeBF16.cpp:149-156
  - 克隆原算子，修改其 `bf16` 结果类型为 `f32` 并在区域内同步调整参数/内部指令类型 bishengir\lib\Dialect\HFusion\Transforms\LegalizeBF16.cpp:67-105, 107-138
  - 在新算子之后，将 `f32` 结果再通过 `hfusion.cast` 转回 `bf16`，并替换原算子 bishengir\lib\Dialect\HFusion\Transforms\LegalizeBF16.cpp:161-172
- 覆盖范围：为所有 Linalg 结构化算子、HFusion 结构化算子，以及部分额外算子（`hfusion.is_nan`/`is_inf`/`is_finite`、`hfusion.sort`、`tensor.concat`、`tensor.pad`）注册模式 bishengir\lib\Dialect\HFusion\Transforms\LegalizeBF16.cpp:203-218

**简单示例 1（元素算子）**

- 输入（在 `bf16` 上计算）:

```mlir
func.func @bf16_elemwise(%a: tensor<4x4xbf16>, %b: tensor<4x4xbf16>)
    -> tensor<4x4xbf16> {
  %dst = tensor.empty() : tensor<4x4xbf16>
  %r = linalg.elemwise_binary {fun = #linalg.binary_fn<add>}
       ins(%a, %b : tensor<4x4xbf16>, tensor<4x4xbf16>)
       outs(%dst : tensor<4x4xbf16>) -> tensor<4x4xbf16>
  return %r : tensor<4x4xbf16>
}
```

- 运行 `hfusion-legalize-bf16` 后（在 `f32` 上计算并在边界回写 `bf16`）:

```mlir
func.func @bf16_elemwise(%a: tensor<4x4xbf16>, %b: tensor<4x4xbf16>)
    -> tensor<4x4xbf16> {
  %a32 = hfusion.cast ins(%a) outs(tensor.empty() : tensor<4x4xf32>) -> tensor<4x4xf32>
  %b32 = hfusion.cast ins(%b) outs(tensor.empty() : tensor<4x4xf32>) -> tensor<4x4xf32>
  %dst32 = tensor.empty() : tensor<4x4xf32>
  %r32 = linalg.elemwise_binary {fun = #linalg.binary_fn<add>}
        ins(%a32, %b32 : tensor<4x4xf32>, tensor<4x4xf32>)
        outs(%dst32 : tensor<4x4xf32>) -> tensor<4x4xf32>
  %r = hfusion.cast ins(%r32) outs(tensor.empty() : tensor<4x4xbf16>) -> tensor<4x4xbf16>
  return %r : tensor<4x4xbf16>
}
```

**简单示例 2（HFusion 一元算子）**

- 输入:

```mlir
func.func @bf16_unary(%x: tensor<8xbf16>) -> tensor<8xbf16> {
  %dst = tensor.empty() : tensor<8xbf16>
  %y = hfusion.elemwise_unary {fun = #hfusion.unary_fn<exp>}
       ins(%x : tensor<8xbf16>) outs(%dst : tensor<8xbf16>) -> tensor<8xbf16>
  return %y : tensor<8xbf16>
}
```

- 输出:

```mlir
func.func @bf16_unary(%x: tensor<8xbf16>) -> tensor<8xbf16> {
  %x32 = hfusion.cast ins(%x) outs(tensor.empty() : tensor<8xf32>) -> tensor<8xf32>
  %dst32 = tensor.empty() : tensor<8xf32>
  %y32 = hfusion.elemwise_unary {fun = #hfusion.unary_fn<exp>}
        ins(%x32 : tensor<8xf32>) outs(%dst32 : tensor<8xf32>) -> tensor<8xf32>
  %y = hfusion.cast ins(%y32) outs(tensor.empty() : tensor<8xbf16>) -> tensor<8xbf16>
  return %y : tensor<8xbf16>
}
```

**运行命令**

```bat
bishengir-opt -hfusion-legalize-bf16 path\to\your.mlir -split-input-file
```

**定位参考**

- 定义与摘要：bishengir\include\bishengir\Dialect\HFusion\Transforms\Passes.td:358-361
- 改写排除列表：bishengir\lib\Dialect\HFusion\Transforms\LegalizeBF16.cpp:50-56
- 改写核心逻辑：bishengir\lib\Dialect\HFusion\Transforms\LegalizeBF16.cpp:141-172
- 模式注册范围：bishengir\lib\Dialect\HFusion\Transforms\LegalizeBF16.cpp:203-218
- 构造器：`mlir::hfusion::createLegalizeBF16Pass()` bishengir\lib\Dialect\HFusion\Transforms\LegalizeBF16.cpp:228-230

## hfusion-legalize-bool

**作用**

- 统一设备入口函数的布尔类型接口：把函数签名中的 `i1`（包括张量元素 `i1`）转换为 `i8`，而在函数体内仍以 `i1` 进行逻辑计算
- 输入参数执行 `i8 -> i1` 转换后参与运算；返回值在 `return` 前执行 `i1 -> i8` 转换
- 同步更新调用链上的调用者函数签名与调用点的实参与结果类型为 `i8`
- 折叠多余的双重转换并清理中间标记

**实现要点**

- 作用域：遍历模块中所有设备入口函数并处理其调用者链 bishengir\lib\Dialect\HFusion\Transforms\LegalizeBool.cpp:315-336
- 修改签名：将所有输入/输出类型中元素 `i1` 改为 `i8`，同时更新入口基本块的形参类型 bishengir\lib\Dialect\HFusion\Transforms\LegalizeBool.cpp:109-141
- 输入转换：为原本是 `i1` 的参数在使用前插入 `hfusion.cast` 执行 `i8 -> i1`，并沿 `reshape` 链统一类型为 `i8` bishengir\lib\Dialect\HFusion\Transforms\LegalizeBool.cpp:170-216
- 输出转换：在 `func.return` 之前对 `i1` 返回值插入 `hfusion.cast` 执行 `i1 -> i8`，并调整相关 `reshape` 链的类型为 `i8` bishengir\lib\Dialect\HFusion\Transforms\LegalizeBool.cpp:243-279
- 调用者更新：将调用者函数及其 `call` 的操作数/结果中 `i1` 改为 `i8`，保证类型一致 bishengir\lib\Dialect\HFusion\Transforms\LegalizeBool.cpp:150-168
- 折叠与清理：
  - 折叠由合法化产生的连续 `cast(i8->i1)` 后接 `cast(i1->B)` 为一次 `cast(i8->B)` bishengir\lib\Dialect\HFusion\Transforms\LegalizeBool.cpp:48-72
  - 清除用于标记的 `annotation.mark` bishengir\lib\Dialect\HFusion\Transforms\LegalizeBool.cpp:74-92

**简单示例 1（设备核：参数和返回）**

- 输入

```mlir
func.func @kernel(%x: tensor<4xi1>) -> tensor<4xi1>
attributes {hacc.entry, hacc.function_kind = #hacc.function_kind<DEVICE>} {
  %dst = tensor.empty() : tensor<4xi1>
  %y = hfusion.compare {compare_fn = #hfusion.compare_fn<veq>}
       ins(%x, %x : tensor<4xi1>, tensor<4xi1>)
       outs(%dst : tensor<4xi1>) -> tensor<4xi1>
  return %y : tensor<4xi1>
}
```

- 运行 `hfusion-legalize-bool` 后

```mlir
func.func @kernel(%x: tensor<4xi8>) -> tensor<4xi8>
attributes {hacc.entry, hacc.function_kind = #hacc.function_kind<DEVICE>} {
  %x_i1 = hfusion.cast ins(%x) outs(tensor.empty() : tensor<4xi1>) -> tensor<4xi1>
  %dst_i1 = tensor.empty() : tensor<4xi1>
  %y_i1 = hfusion.compare {compare_fn = #hfusion.compare_fn<veq>}
           ins(%x_i1, %x_i1 : tensor<4xi1>, tensor<4xi1>)
           outs(%dst_i1 : tensor<4xi1>) -> tensor<4xi1>
  %y_i8 = hfusion.cast ins(%y_i1) outs(tensor.empty() : tensor<4xi8>) -> tensor<4xi8>
  return %y_i8 : tensor<4xi8>
}
```

**简单示例 2（调用者同步更新）**

- 输入

```mlir
func.func @kernel(%x: tensor<4xi1>) -> tensor<4xi1>
attributes {hacc.entry, hacc.function_kind = #hacc.function_kind<DEVICE>} {
  %dst = tensor.empty() : tensor<4xi1>
  %y = hfusion.elemwise_unary {fun = #hfusion.unary_fn<vnot>}
       ins(%x : tensor<4xi1>) outs(%dst : tensor<4xi1>) -> tensor<4xi1>
  return %y : tensor<4xi1>
}

func.func @main(%x: tensor<4xi1>) -> tensor<4xi1>
attributes {hacc.function_kind = #hacc.function_kind<HOST>} {
  %r = func.call @kernel(%x) : (tensor<4xi1>) -> tensor<4xi1>
  return %r : tensor<4xi1>
}
```

- 运行 `hfusion-legalize-bool` 后

```mlir
func.func @kernel(%x: tensor<4xi8>) -> tensor<4xi8>
attributes {hacc.entry, hacc.function_kind = #hacc.function_kind<DEVICE>} {
  %x_i1 = hfusion.cast ins(%x) outs(tensor.empty() : tensor<4xi1>) -> tensor<4xi1>
  %dst_i1 = tensor.empty() : tensor<4xi1>
  %y_i1 = hfusion.elemwise_unary {fun = #hfusion.unary_fn<vnot>}
           ins(%x_i1 : tensor<4xi1>) outs(%dst_i1 : tensor<4xi1>) -> tensor<4xi1>
  %y_i8 = hfusion.cast ins(%y_i1) outs(tensor.empty() : tensor<4xi8>) -> tensor<4xi8>
  return %y_i8 : tensor<4xi8>
}

func.func @main(%x: tensor<4xi8>) -> tensor<4xi8>
attributes {hacc.function_kind = #hacc.function_kind<HOST>} {
  %r = func.call @kernel(%x) : (tensor<4xi8>) -> tensor<4xi8>
  return %r : tensor<4xi8>
}
```

**运行命令**

```bat
bishengir-opt -hfusion-legalize-bool path\to\your.mlir -split-input-file
```

**定位参考**

- 定义与摘要：bishengir\include\bishengir\Dialect\HFusion\Transforms\Passes.td:364-367
- 签名与调用点改写：bishengir\lib\Dialect\HFusion\Transforms\LegalizeBool.cpp:109-168
- 输入/输出插入转换：bishengir\lib\Dialect\HFusion\Transforms\LegalizeBool.cpp:170-216, 243-279
- 折叠和清理：bishengir\lib\Dialect\HFusion\Transforms\LegalizeBool.cpp:48-92, 337-347
- 构造器：`hfusion::createLegalizeBoolPass()` bishengir\lib\Dialect\HFusion\Transforms\LegalizeBool.cpp:350-352

## hfusion-reorder-ops

**作用**

- 在单个 `func::FuncOp` 的每个 `Block` 内，按数据依赖的广度优先顺序对操作进行重排，保证先放置无依赖的生产者（常量、`tensor.empty`、外部 live-in 等），再层层就近放置其直接消费者的计算（以 `linalg::LinalgOp` 及包含 Linalg 的复合 op 为“计算”）
- 不改变语义，仅变更物理顺序，增强可读性并提供稳定、确定的操作序
- 对带 Region 的复合操作，会综合其外部 live-in 与操作数计算“入度”，确保类型与支配关系正确

**机制**

- 计算“入度”：对普通 op 用唯一化后的操作数个数；对带 Region 的复合 op 还把 Region 内部对外部值的 live-in 加入入度 bishengir\lib\Dialect\HFusion\Transforms\ReorderOpsByBFS.cpp:176-196, 198-228
- 选取起点：把入度为 0 的操作入队，并作为锚点位置；随后从块参数与外部 live-in 值出发，按用户的原始出现顺序（稳定排序）推进遍历 bishengir\lib\Dialect\HFusion\Transforms\ReorderOpsByBFS.cpp:245-275
- 排序策略：每当某个 op 的入度归零，就将其移动到锚点之后；非“计算”op（非 Linalg 且内部不含 Linalg）只作为过渡，继续通过其结果值向下层消费者推进，直到排到计算 op bishengir\lib\Dialect\HFusion\Transforms\ReorderOpsByBFS.cpp:89-129, 131-174
- 稳定性：为所有 `Operation*` 建立原始遍历索引，作为并列项的稳定比较依据 bishengir\lib\Dialect\HFusion\Transforms\ReorderOpsByBFS.cpp:283-305

**简单示例**

- 输入（两个互不相关的计算链，先写 `b` 链，再写 `a` 链）：

```mlir
func.func @bfs(%a: tensor<4xf32>, %b: tensor<4xf32>)
  -> (tensor<4xf32>, tensor<4xf32>) {
  %ea = tensor.empty() : tensor<4xf32>
  %eb = tensor.empty() : tensor<4xf32>

  %b_log  = linalg.elemwise_unary {fun = #linalg.unary_fn<log>}
             ins(%b : tensor<4xf32>) outs(%eb : tensor<4xf32>) -> tensor<4xf32>
  %b_sqrt = linalg.elemwise_unary {fun = #linalg.unary_fn<sqrt>}
             ins(%b_log : tensor<4xf32>) outs(%eb : tensor<4xf32>) -> tensor<4xf32>

  %a_exp  = linalg.elemwise_unary {fun = #linalg.unary_fn<exp>}
             ins(%a : tensor<4xf32>) outs(%ea : tensor<4xf32>) -> tensor<4xf32>
  %a_abs  = linalg.elemwise_unary {fun = #linalg.unary_fn<abs>}
             ins(%a_exp : tensor<4xf32>) outs(%ea : tensor<4xf32>) -> tensor<4xf32>

  return %a_abs, %b_sqrt : tensor<4xf32>, tensor<4xf32>
}
```

- 运行 `hfusion-reorder-ops` 后（按块参数顺序从 `%a` 开始 BFS，重排为 `a` 链在前，`b` 链在后；语义不变）：

```mlir
func.func @bfs(%a: tensor<4xf32>, %b: tensor<4xf32>)
  -> (tensor<4xf32>, tensor<4xf32>) {
  %ea = tensor.empty() : tensor<4xf32>
  %eb = tensor.empty() : tensor<4xf32>

  %a_exp  = linalg.elemwise_unary {fun = #linalg.unary_fn<exp>}
             ins(%a : tensor<4xf32>) outs(%ea : tensor<4xf32>) -> tensor<4xf32>
  %a_abs  = linalg.elemwise_unary {fun = #linalg.unary_fn<abs>}
             ins(%a_exp : tensor<4xf32>) outs(%ea : tensor<4xf32>) -> tensor<4xf32>

  %b_log  = linalg.elemwise_unary {fun = #linalg.unary_fn<log>}
             ins(%b : tensor<4xf32>) outs(%eb : tensor<4xf32>) -> tensor<4xf32>
  %b_sqrt = linalg.elemwise_unary {fun = #linalg.unary_fn<sqrt>}
             ins(%b_log : tensor<4xf32>) outs(%eb : tensor<4xf32>) -> tensor<4xf32>

  return %a_abs, %b_sqrt : tensor<4xf32>, tensor<4xf32>
}
```

- 更复杂的内含 `tensor.extract_slice`、`hfusion.load/broadcast/store` 的块也会被按“生产者→直接消费者→下游消费者”层层推进重排，参考真实用例：bishengir\test\Dialect\HFusion\reorder-ops.mlir

**运行命令**

```bat
bishengir-opt -hfusion-reorder-ops path\to\your.mlir -split-input-file
```

**定位参考**

- 定义与摘要：bishengir\include\bishengir\Dialect\HFusion\Transforms\Passes.td:370-375
- 核心流程：
  - BFS 排序入口：bishengir\lib\Dialect\HFusion\Transforms\ReorderOpsByBFS.cpp:307-314
  - 构建稳定索引与块遍历：bishengir\lib\Dialect\HFusion\Transforms\ReorderOpsByBFS.cpp:283-305
  - 入队与锚点插入：bishengir\lib\Dialect\HFusion\Transforms\ReorderOpsByBFS.cpp:104-129
  - 用户遍历与入度归零触发：bishengir\lib\Dialect\HFusion\Transforms\ReorderOpsByBFS.cpp:131-174
  - 入度计算与 live-in 汇总：bishengir\lib\Dialect\HFusion\Transforms\ReorderOpsByBFS.cpp:176-196, 198-228

## hfusion-downgrade-fp64

**作用**

- 将函数体中的 `f64` 常量统一降级为 `f32` 常量，避免在后续计算中出现不必要的 `arith.truncf` 转换
- 以 `f32` 近似值替换原 `f64` 常量（存在可控的精度损失），更贴近 HFusion 侧主要在 `f16/bf16/f32` 上运行的数值域
- 专注于常量和由常量驱动的截断转换；不会更改一般运算的精度或类型

**实现要点**

- 模式 1（常量降级）：匹配 `arith.constant : f64`，按 `static_cast<float>` 生成对应的 `f32` 值 bishengir\lib\Dialect\HFusion\Transforms\DowngradeFP64CstOp.cpp:37-60
- 模式 2（去冗余截断）：匹配 `arith.truncf` 的输入是常量且已表达为 `f32` 时，用 `f32` 常量直接替换，移除对 `f64` 的依赖 bishengir\lib\Dialect\HFusion\Transforms\DowngradeFP64CstOp.cpp:62-84
- 作用域：`func::FuncOp` bishengir\lib\Dialect\HFusion\Transforms\DowngradeFP64CstOp.cpp:100-106
- 构造器：`hfusion::createDowngradeFP64CstOpPass()` bishengir\lib\Dialect\HFusion\Transforms\DowngradeFP64CstOp.cpp:110-112
- 管线位置：在“Outline 后处理”阶段调用 bishengir\lib\Dialect\HFusion\Pipelines\HFusionPipelines.cpp:143-151
- 定义与摘要：bishengir\include\bishengir\Dialect\HFusion\Transforms\Passes.td:377-378

**简单示例 1（移除 f64→f32 的截断）**

- 输入

```mlir
%c64 = arith.constant 1.6276041666666666E-4 : f64
%cf32 = arith.truncf %c64 : f64 to f32
%dst = tensor.empty() : tensor<24x32xf32>
%filled = linalg.fill ins(%cf32 : f32) outs(%dst : tensor<24x32xf32>) -> tensor<24x32xf32>
```

- 运行 `hfusion-downgrade-fp64` 后

```mlir
%c32 = arith.constant 1.62760422E-4 : f32
%dst = tensor.empty() : tensor<24x32xf32>
%filled = linalg.fill ins(%c32 : f32) outs(%dst : tensor<24x32xf32>) -> tensor<24x32xf32>
```

- 对应测试用例：bishengir\test\Dialect\HFusion\downgrade-fp64.mlir:4-7, 38-40

**简单示例 2（常量参与 bf16 路径）**

- 输入

```mlir
%c64 = arith.constant 9.9999999999999995E-21 : f64
%cbf16 = arith.truncf %c64 : f64 to bf16
%cf32  = arith.extf %cbf16 : bf16 to f32
%sum = linalg.elemwise_binary {fun = #linalg.binary_fn<add>}
       ins(%x, %cf32 : tensor<4096xf32>, f32)
       outs(%y : tensor<4096xf32>) -> tensor<4096xf32>
```

- 运行 `hfusion-downgrade-fp64` 后

```mlir
%cf32 = arith.constant 9.99999968E-21 : f32
%sum = linalg.elemwise_binary {fun = #linalg.binary_fn<add>}
       ins(%x, %cf32 : tensor<4096xf32>, f32)
       outs(%y : tensor<4096xf32>) -> tensor<4096xf32>
```

- 对应测试用例：bishengir\test\Dialect\HFusion\downgrade-fp64.mlir:71-79, 85-90

**运行命令**

```bat
bishengir-opt -hfusion-downgrade-fp64 path\to\your.mlir -split-input-file
```

**定位参考**

- 降级模式：bishengir\lib\Dialect\HFusion\Transforms\DowngradeFP64CstOp.cpp:37-60
- 截断清理：bishengir\lib\Dialect\HFusion\Transforms\DowngradeFP64CstOp.cpp:62-84
- 入口实现：bishengir\lib\Dialect\HFusion\Transforms\DowngradeFP64CstOp.cpp:100-106
- 测试样例：bishengir\test\Dialect\HFusion\downgrade-fp64.mlir
- 传输管线：bishengir\lib\Dialect\HFusion\Pipelines\HFusionPipelines.cpp:143-151

## hfusion-infer-out-shapes

**作用**

- 为每个设备函数生成对应的“输出形状推导函数”，用于在运行前根据输入张量的动态维度推导内核输出的各结果形状
- 在设备函数上记录指向该形状函数的绑定属性 `hacc.infer_output_shape_function`，供运行时或后续编排调用
- 形状函数被标记为主机函数，类型为 `hacc.host_func_type = infer_output_shape_function`，并且只保留输入参数（移除输出参数）
- 每个返回值的形状以 `tensor<rank x index>` 表示，内部通过 `tensor.dim` 和 `tensor.from_elements` 组合得到各维度值

**实现要点**

- 克隆设备函数为形状函数，命名为 `原函数名_infer_output_shape_function` bishengir\lib\Dialect\HFusion\Transforms\InferOutShapes.cpp:89-106
- 对设备函数的每个返回值，优先在其“根产出操作”上调用 `reifyResultShapes` 接口获得维度；若根操作满足 `SameOperandsAndResultShape`，也尝试其操作数的生产者以提升命中率 bishengir\lib\Dialect\HFusion\Transforms\InferOutShapes.cpp:154-167, 169-179
- 将 `OpFoldResult` 转换为 `Value`：常量维度用 `arith.constant`，动态维度用已有 SSA 值；最终组装为 `tensor.from_elements` 返回 bishengir\lib\Dialect\HFusion\Transforms\InferOutShapes.cpp:50-61, 183-193, 200-206
- 在原设备函数上写入绑定属性 `hacc.infer_output_shape_function` 指向形状函数符号名 bishengir\lib\Dialect\HFusion\Transforms\InferOutShapes.cpp:195-199 与 bishengir\include\bishengir\Dialect\HACC\IR\HACCAttrs.td:178-185
- 形状函数标注为 Host，设置 `hacc.host_func_type = infer_output_shape_function`，并删除输出参数 bishengir\lib\Dialect\HFusion\Transforms\InferOutShapes.cpp:208-214, 228-258
- 简化形状函数中的 `tensor.dim` 等规范化模式 bishengir\lib\Dialect\HFusion\Transforms\InferOutShapes.cpp:217-226

**简单示例 1（元素二元运算，SameOperandsAndResultShape）**

- 输入（设备函数）

```mlir
func.func @test_elemwise_binary(%a: tensor<?x?xf32>, %b: tensor<?x?xf32>, %out: tensor<?x?xf32>)
  -> tensor<?x?xf32> attributes {hacc.function_kind = #hacc.function_kind<DEVICE>} {
  %r = linalg.elemwise_binary {fun = #linalg.binary_fn<add>}
       ins(%a, %b : tensor<?x?xf32>, tensor<?x?xf32>)
       outs(%out : tensor<?x?xf32>) -> tensor<?x?xf32>
  return %r : tensor<?x?xf32>
}
```

- 运行 `hfusion-infer-out-shapes` 后
  - 原设备函数新增绑定属性：
    `hacc.infer_output_shape_function = #hacc.infer_output_shape_function<@test_elemwise_binary_infer_output_shape_function>`
  - 生成形状函数（Host）：

```mlir
func.func @test_elemwise_binary_infer_output_shape_function(%a: tensor<?x?xf32>, %b: tensor<?x?xf32>, %out: tensor<?x?xf32>)
  -> tensor<2xindex>
  attributes {hacc.function_kind = #hacc.function_kind<HOST>,
              hacc.host_func_type = #hacc.host_func_type<infer_output_shape_function>} {
  %c0 = arith.constant 0 : index
  %c1 = arith.constant 1 : index
  %d0 = tensor.dim %a, %c0 : tensor<?x?xf32>
  %d1 = tensor.dim %a, %c1 : tensor<?x?xf32>
  %shape = tensor.from_elements %d0, %d1 : tensor<2xindex>
  return %shape : tensor<2xindex>
}
```

- 对应测试：bishengir\test\Dialect\HFusion\infer-kernel-output-shapes.mlir:7-17

**简单示例 2（矩阵乘，跨多层取维度）**

- 输入（设备函数）

```mlir
func.func @test_matmul(%a0: tensor<?x?xf32>, %a1: tensor<?x?xf32>,
                       %o0: tensor<?x?xf32>, %b0: tensor<?x?xf32>, %b1: tensor<?x?xf32>,
                       %o1: tensor<?x?xf32>, %out: tensor<?x?xf32>)
  -> tensor<?x?xf32> attributes {hacc.function_kind = #hacc.function_kind<DEVICE>} {
  %x = linalg.matmul_transpose_a ins(%a0, %a1) outs(%o0)
  %y = linalg.matmul_transpose_b ins(%b0, %b1) outs(%o1)
  %z = linalg.matmul ins(%x, %y) outs(%out)
  return %z : tensor<?x?xf32>
}
```

- 生成形状函数会从中间结果的“根产出操作”推导维度（例如取 `%o0` 的第 0 维和 `%o1` 的第 1 维）：

```mlir
func.func @test_matmul_infer_output_shape_function(...)
  -> tensor<2xindex> {
  %c0 = arith.constant 0 : index
  %c1 = arith.constant 1 : index
  %d0 = tensor.dim %o0, %c0
  %d1 = tensor.dim %o1, %c1
  %shape = tensor.from_elements %d0, %d1
  return %shape
}
```

- 对应测试：bishengir\test\Dialect\HFusion\infer-kernel-output-shapes.mlir:28-31

**简单示例 3（多结果 + 输出参数移除）**

- 输入（设备函数，含输出参数标记）

```mlir
func.func @test_out_param(%x: tensor<?x4096xf16>,
                          %w: tensor<6144x4096xf16>,
                          %y: tensor<?x6144xf16> {hacc.arg_type = #hacc.arg_type<output>, hacc.output_idx = #hacc.output_idx<0>})
  -> tensor<?x6144xf16> attributes {hacc.entry, hacc.function_kind = #hacc.function_kind<DEVICE>} {
  %r = linalg.matmul_transpose_b ins(%x, %w) outs(%y)
  return %r : tensor<?x6144xf16>
}
```

- 生成的形状函数只保留输入参数 `%x` 和 `%w`，返回输出形状：

```mlir
func.func @test_out_param_infer_output_shape_function(%x: tensor<?x4096xf16>, %w: tensor<6144x4096xf16>)
  -> tensor<2xindex>
  attributes {hacc.function_kind = #hacc.function_kind<HOST>,
              hacc.host_func_type = #hacc.host_func_type<infer_output_shape_function>} {
  %c0 = arith.constant 0 : index
  %d0 = tensor.dim %x, %c0
  %shape = tensor.from_elements %d0, %c6144
  return %shape : tensor<2xindex>
}
```

- 对应测试：bishengir\test\Dialect\HFusion\infer-kernel-output-shapes.mlir:94-98

**运行命令**

```bat
bishengir-opt --hfusion-infer-out-shapes path\to\your.mlir --split-input-file
```

**定位参考**

- 定义与摘要：bishengir\include\bishengir\Dialect\HFusion\Transforms\Passes.td:383-386
- 形状函数生成与绑定：bishengir\lib\Dialect\HFusion\Transforms\InferOutShapes.cpp:132-215
- 参数裁剪（移除输出）：bishengir\lib\Dialect\HFusion\Transforms\InferOutShapes.cpp:228-258
- `reifyResultShapes` 驱动推导：bishengir\lib\Dialect\HFusion\Transforms\InferOutShapes.cpp:63-75, 169-179
- 常量与维度值拼装：bishengir\lib\Dialect\HFusion\Transforms\InferOutShapes.cpp:50-61, 183-193
- 形状函数属性枚举与绑定属性：bishengir\include\bishengir\Dialect\HACC\IR\HACCAttrs.td:124-148, 178-185

## hfusion-compose-multi-reduce

**作用**

- 将同一块中多个互不依赖、形状/归约维兼容的 `linalg.reduce` 合并成一个“多路归约”操作，统一读入多个输入和初值、在一个 Region 中完成多路规约并一次性产出多个结果
- 降低归约的调度/遍历开销，提升融合和内存访问的局部性，为后续调度与转换提供更整洁的 IR
- 支持两种模式：
  - 严格模式：要求输入形状和 `dimensions` 完全一致
  - 激进模式：若形状可通过 `tensor.expand_shape`/`tensor.collapse_shape` 松弛匹配，则先对操作数做对齐再合并

**关键逻辑**

- 收集与分组
  - 收集函数内全部 `linalg.reduce` bishengir\lib\Dialect\HFusion\Transforms\ComposeMultiReduce.cpp:70-82
  - 仅处理静态形状；动态形状跳过 bishengir\lib\Dialect\HFusion\Transforms\ComposeMultiReduce.cpp:113-119
  - 将可兼容的 `reduce` 按组聚合，控制每组大小 `maxCompose` bishengir\lib\Dialect\HFusion\Transforms\ComposeMultiReduce.cpp:88-107, 126-143
- 兼容性判定
  - 严格：形状完全一致且归约维相同 bishengir\lib\Dialect\HFusion\Transforms\ComposeMultiReduce.cpp:197-208
  - 激进：尝试通过松弛重关联把较小维形状扩展到基准形状，验证维度映射一致 bishengir\lib\Dialect\HFusion\Transforms\ComposeMultiReduce.cpp:210-246, 248-262
- 依赖与距离约束
  - 组内任意两个 `reduce` 不得有数据依赖，且二者从共同祖先的最短路径差不超过 `maxDistDiff` bishengir\lib\Dialect\HFusion\Transforms\ComposeMultiReduce.cpp:265-286
- 激进模式的形状对齐
  - 对输入和初值分别按需要插入 `expand_shape`；克隆出对齐后的 `reduce`，产出后用 `collapse_shape` 收回并替换原结果 bishengir\lib\Dialect\HFusion\Transforms\ComposeMultiReduce.cpp:299-344, 355-383, 385-399, 420-431, 452-472
- 合成与替换
  - 将一组 `reduce` 的所有输入/初值汇总，创建一个新的 `linalg.reduce`，并把各原 `reduce` 的 Region 逻辑合并到一个块内，多值 `linalg.yield` 返回 bishengir\lib\Dialect\HFusion\Transforms\ComposeMultiReduce.cpp:499-523, 526-538
  - 用新结果替换旧结果并删除旧 `reduce`；新 op 标注 `hfusion.reduce_composed` 属性 bishengir\lib\Dialect\HFusion\Transforms\ComposeMultiReduce.cpp:521, 489-491；属性定义 bishengir\include\bishengir\Dialect\HFusion\IR\HFusionAttrs.td:73-77

**简单示例（严格模式）**

- 输入（两个同形状、同维度的归约）：

```mlir
func.func @compose(%A: tensor<4x5xf32>, %B: tensor<4x5xf32>,
                   %I0: tensor<4xf32>, %I1: tensor<4xf32>)
  -> (tensor<4xf32>, tensor<4xf32>) {
  %R0 = linalg.reduce ins(%A : tensor<4x5xf32>) outs(%I0 : tensor<4xf32>) dimensions = [1]
    (%in: f32, %init: f32) {
      %x = arith.addf %in, %init : f32
      linalg.yield %x : f32
    }
  %R1 = linalg.reduce ins(%B : tensor<4x5xf32>) outs(%I1 : tensor<4xf32>) dimensions = [1]
    (%in: f32, %init: f32) {
      %y = arith.mulf %in, %init : f32
      linalg.yield %y : f32
    }
  return %R0, %R1 : tensor<4xf32>, tensor<4xf32>
}
```

- 运行 `hfusion-compose-multi-reduce` 后：

```mlir
%RR:2 = linalg.reduce ins(%A, %B
        : tensor<4x5xf32>, tensor<4x5xf32>)
      outs(%I0, %I1 : tensor<4xf32>, tensor<4xf32>)
      dimensions = [1] {hfusion.reduce_composed = ""}
    (%in0: f32, %in1: f32, %init0: f32, %init1: f32) {
      %x = arith.addf %in0, %init0 : f32
      %y = arith.mulf %in1, %init1 : f32
      linalg.yield %x, %y : f32, f32
    }
return %RR#0, %RR#1 : tensor<4xf32>, tensor<4xf32>
```

**简单示例（激进模式）**

- 当两个归约在相同“批维”上归约但中间展开维不同（如 `4x5` 与 `4x6`），激进模式会先把较小维通过 `expand_shape` 对齐到基准，再合并为多路归约，结果经 `collapse_shape` 收回到各自原类型
- 对照测试样例：bishengir\test\Dialect\HFusion\hfusion-compose-multi-reduce.mlir:22-75（多组 4x5 与 4x6 的合并）以及 159-190（`aggressive=true` 时进一步合并）

**运行命令**

- 严格模式：

```bat
bishengir-opt --hfusion-compose-multi-reduce path\to\your.mlir --split-input-file
```

- 激进模式与上限控制：

```bat
bishengir-opt --hfusion-compose-multi-reduce="aggressive=true,max-compose=8,max-dist-diff=16" path\to\your.mlir --split-input-file
```

**定位参考**

- 定义与选项：bishengir\include\bishengir\Dialect\HFusion\Transforms\Passes.td:389-402
- 主流程入口：bishengir\lib\Dialect\HFusion\Transforms\ComposeMultiReduce.cpp:47-59
- 静态形状要求：bishengir\lib\Dialect\HFusion\Transforms\ComposeMultiReduce.cpp:113-119
- 分组与兼容性检查（严格/激进）：bishengir\lib\Dialect\HFusion\Transforms\ComposeMultiReduce.cpp:145-169, 197-208, 210-246, 248-262
- 依赖与距离约束：bishengir\lib\Dialect\HFusion\Transforms\ComposeMultiReduce.cpp:265-286
- 形状对齐与结果折叠：bishengir\lib\Dialect\HFusion\Transforms\ComposeMultiReduce.cpp:299-344, 355-383, 420-431, 452-472
- 合成新 `reduce`：bishengir\lib\Dialect\HFusion\Transforms\ComposeMultiReduce.cpp:499-523, 526-538
- 合成标记属性：bishengir\include\bishengir\Dialect\HFusion\IR\HFusionAttrs.td:73-77
- 管线调用位置：bishengir\lib\Dialect\HFusion\Pipelines\HFusionPipelines.cpp:116-141

## hfusion-decompose-multi

**作用**

- 将多结果的 `linalg.reduce` 和 `linalg.generic` 按输出拆分为多个“单结果”操作，简化 IR，便于后续调度、合并和 Codegen
- 针对每个可独立的结果，克隆原始 `linalg`，只保留它所需的输入/输出与 `region` 参数，修正 `yield`、`indexing_maps` 和 `operandSegmentSizes`
- 对不可独立的剩余结果，保留为一个“残余”操作，确保功能等价
- 定义与构造：`bishengir\include\bishengir\Dialect\HFusion\Transforms\Passes.td:404-406`，构造函数 `mlir::hfusion::createDecomposeMulti()` 在 `bishengir\lib\Dialect\HFusion\Transforms\DecomposeMulti.cpp:245-247`

**触发条件**

- 只处理有多个结果的 `linalg.reduce`/`linalg.generic`，单结果直接跳过
- 某个结果 i 的 `yield` 产出如果直接由一个“仅使用 block 参数且仅 1 次使用”的算术/元素级操作定义，则该结果可被抽取
  - 定义在 `bishengir\lib\Dialect\HFusion\Transforms\DecomposeMulti.cpp:65-101`
  - 要求该定义操作 `definer->hasOneUse()`，其所有操作数都是 `BlockArgument` 且 `hasOneUse()`，否则标记为不可抽取
- 抽取时会构造待保留的参数索引集，将对应输入和该输出的 init/result 搭成一个新 `linalg`，并更新 `yield`、删除未用的 `region` 参数
  - 克隆及修正逻辑见 `bishengir\lib\Dialect\HFusion\Transforms\DecomposeMulti.cpp:132-225`
  - 对 `linalg::GenericOp` 额外修正 `indexing_maps` 与 `operandSegmentSizes`，见 `DecomposeMulti.cpp:205-223`
- 模式注册与应用：`ReduceOp` 和 `GenericOp` 两类，`runOnOperation` 在 `DecomposeMulti.cpp:228-240`

**简单示例（linalg.generic）**
输入（一个 `linalg.generic` 同时产生两个独立输出，第二个不依赖第一个）：

```mlir
module {
  func.func @two_out_generic(%A: tensor<4xf32>, %B: tensor<4xf32>)
    -> (tensor<4xf32>, tensor<4xf32>) {
    %O1 = tensor.empty() : tensor<4xf32>
    %O2 = tensor.empty() : tensor<4xf32>
    %res:2 = linalg.generic
      {indexing_maps = [
         affine_map<(d0) -> (d0)>,  // A
         affine_map<(d0) -> (d0)>,  // B
         affine_map<(d0) -> (d0)>,  // O1
         affine_map<(d0) -> (d0)>   // O2
       ],
       iterator_types = ["parallel"]}
      ins(%A, %B : tensor<4xf32>, tensor<4xf32>)
      outs(%O1, %O2 : tensor<4xf32>, tensor<4xf32>) {
      ^bb0(%a: f32, %b: f32, %o1: f32, %o2: f32):
        %add = arith.addf %a, %o1 : f32
        %mul = arith.mulf %a, %b  : f32
        linalg.yield %add, %mul : f32, f32
    } -> (tensor<4xf32>, tensor<4xf32>)
    return %res#0, %res#1 : tensor<4xf32>, tensor<4xf32>
  }
}
```

输出（被拆成两个单结果 `linalg.generic`，各自只保留所需的参数与映射）：

```mlir
%O1 = tensor.empty() : tensor<4xf32>
%O2 = tensor.empty() : tensor<4xf32>
%out1 = linalg.generic
  {indexing_maps = [
     affine_map<(d0) -> (d0)>,  // A
     affine_map<(d0) -> (d0)>   // O1
   ],
   iterator_types = ["parallel"]}
  ins(%A : tensor<4xf32>) outs(%O1 : tensor<4xf32>) {
  ^bb0(%a: f32, %o1: f32):
    %add = arith.addf %a, %o1 : f32
    linalg.yield %add : f32
} -> tensor<4xf32>

%out2 = linalg.generic
  {indexing_maps = [
     affine_map<(d0) -> (d0)>,  // A
     affine_map<(d0) -> (d0)>,  // B
     affine_map<(d0) -> (d0)>   // O2
   ],
   iterator_types = ["parallel"]}
  ins(%A, %B : tensor<4xf32>, tensor<4xf32>) outs(%O2 : tensor<4xf32>) {
  ^bb0(%a: f32, %b: f32, %o2: f32):
    %mul = arith.mulf %a, %b : f32
    linalg.yield %mul : f32
} -> tensor<4xf32>

return %out1, %out2 : tensor<4xf32>, tensor<4xf32>
```

**简单示例（linalg.reduce）**

- 输入：一个 `linalg.reduce` 产生两个结果，分别用不同的 `block` 参数组合（例如第一个做加和，第二个做最大）
- 输出：拆分为两个独立的 `linalg.reduce`，保留相同 `dimensions = [...]`，每个只含其需要的 `ins/outs` 与对应的 `yield` 值
- 可参考测试文件的 `@reduction_tile_2`，在 `bishengir\test\Dialect\HFusion\hfusion-decompose-multi.mlir:4-36`，CHECK 显示多个 `linalg.reduce` 被生成并调整返回次序

**运行方式**

- 运行该 pass：

```bat
bishengir-opt -hfusion-decompose-multi -split-input-file bishengir\test\Dialect\HFusion\hfusion-decompose-multi.mlir
```

- 测试示例中的 `@generic_tile` 会被拆成多个 `linalg.generic`，见 `bishengir\test\Dialect\HFusion\hfusion-decompose-multi.mlir:42-59`

**相关代码**

- 定义：`bishengir\include\bishengir\Dialect\HFusion\Transforms\Passes.td:404-406`
- 模式匹配与抽取判定：`bishengir\lib\Dialect\HFusion\Transforms\DecomposeMulti.cpp:44-101`
- 拆分与克隆、修正 `yield`/参数/死代码清理：`bishengir\lib\Dialect\HFusion\Transforms\DecomposeMulti.cpp:132-225`
- 处理 `linalg.generic` 的 `indexing_maps` 与 `operandSegmentSizes`：`bishengir\lib\Dialect\HFusion\Transforms\DecomposeMulti.cpp:205-223`
- 注册与运行：`bishengir\lib\Dialect\HFusion\Transforms\DecomposeMulti.cpp:228-240`
- 构造：`bishengir\lib\Dialect\HFusion\Transforms\DecomposeMulti.cpp:245-247`

**补充说明**

- 该 pass 的“抽取”是安全前提下进行的：产出值必须是单一定义操作，且仅依赖对应的 `region` 参数；跨结果依赖或复用度高的产出将留在“残余” op 中
- 与 `hfusion-compose-multi-reduce`（多归约合并）相反，该 pass 是“拆分”，二者通常在不同阶段配合以获得更稳定的 IR 形态与优化空间

## hfusion-cache-io

**作用**

- 在 `func::FuncOp` 内对“函数输入”和“函数返回值”做显式的缓存包装，使 IO 语义稳定、可分析、可调度：
  - 为张量参数插入 `hfusion.load`，把函数参数读入到显式的中间缓冲
  - 为返回值插入 `hfusion.store`，把计算结果写入显式的缓冲并用其作为最终返回
- 位置选择与安全性：
  - 输入侧：沿着 reshape/slice 链向后定位到更合适的位置再插入 `hfusion.load`
  - 输出侧：沿着 reshape 链向前回溯到返回值的真实生产者，在该值之后插入 `hfusion.store`；同一个值被多次返回时，会复制到各自独立的 use 路径，避免多个返回指向同一 `store` 的错误
- 与后续优化的关系：统一 IO 形态，便于自动调度、重缓存（如 `hfusion-recache-io`）、内存规划以及后续多缓冲等分析

**实现要点**

- 入口与工厂：`bishengir\include\bishengir\Dialect\HFusion\Transforms\Passes.td:410-412`，构造 `::mlir::hfusion::createCacheIO()`
- 执行逻辑：
  - 函数参数缓存读取：`bishengir\lib\Dialect\HFusion\Transforms\CacheIO.cpp:136-168`
    - 跳过已打 `hacc.cached_io` 标记的参数
    - 利用 `ReshapeAnalyzer` 跟踪参数的 reshape 后继，插入 `hfusion.load ins(arg) outs(empty)`
  - 函数返回缓存写入：`bishengir\lib\Dialect\HFusion\Transforms\CacheIO.cpp:83-134`
    - 跳过已打 `hacc.cached_io` 标记的结果
    - 回溯 reshape 链至真实生产者，插入 `hfusion.store ins(result) outs(init或empty)`，并为重复返回场景复制 use 路径保证各返回唯一
  - `runOnOperation`：`bishengir\lib\Dialect\HFusion\Transforms\CacheIO.cpp:182-189`
- 关键工具函数：
  - `createCacheRead` 插入 `hfusion.load` 并替换后续使用，`bishengir\lib\Dialect\HFusion\Utils\Utils.cpp:826-837`
  - `createCacheWrite` 插入 `hfusion.store`，支持“写回 init”或写入新 `tensor.empty`，并复制 use 树确保重复返回的唯一性，`bishengir\lib\Dialect\HFusion\Utils\Utils.cpp:966-1037`

**简单示例 1（单算子）**

- 输入

```mlir
func.func @test_single_op(%src: tensor<6x4xbf16>, %dst: tensor<6x4xbf16>) -> tensor<6x4xbf16> {
  %res = hfusion.elemwise_unary {fun = #hfusion.unary_fn<sqrt>}
         ins(%src : tensor<6x4xbf16>) outs(%dst : tensor<6x4xbf16>) -> tensor<6x4xbf16>
  return %res : tensor<6x4xbf16>
}
```

- 经过 `hfusion-cache-io` 后要点
  - 在 `%src` 之后插入 `hfusion.load`，把输入显式读到中间缓冲
  - 在 `%res` 之后插入 `hfusion.store`，用存入缓冲后的值作为返回
- 验证参考：`bishengir\test\Dialect\HFusion\hfusion-cache-io.mlir:3-13`

**简单示例 2（含 reshape）**

- 输入

```mlir
func.func @test_reshape(%arg0: tensor<?x256xf32>) -> tensor<?x1x1x256xf32> {
  %c0 = arith.constant 0 : index
  %dim = tensor.dim %arg0, %c0 : tensor<?x256xf32>
  %expanded = tensor.expand_shape %arg0 [[0,1,2],[3]] output_shape [%dim,1,1,256]
               : tensor<?x256xf32> into tensor<?x1x1x256xf32>
  %init = tensor.empty(%dim) : tensor<?x1x1x256xf32>
  %ret = hfusion.elemwise_unary ins(%expanded) outs(%init) -> tensor<?x1x1x256xf32>
  return %ret : tensor<?x1x1x256xf32>
}
```

- 经过 `hfusion-cache-io` 后要点
  - `hfusion.load` 会插在 `%expanded` 之后（沿 reshape 后继跟踪），再驱动后续计算
  - `hfusion.store` 会插在真实生产者之后（沿 reshape 生产者回溯），返回值改为 `store` 的结果
- 验证参考：`bishengir\test\Dialect\HFusion\hfusion-cache-io.mlir:17-27`

**重复返回的处理（示意）**

- 当同一个结果被多次返回（或经不同 reshape 后重复返回），该 pass会复制至各自独立的 use 路径，并为每个返回插入各自的 `hfusion.store`，避免多个返回共享同一 `store`。
- 验证参考：
  - 重复返回 reshape：`bishengir\test\Dialect\HFusion\hfusion-cache-io.mlir:30-54`
  - 重复返回含 cast：`bishengir\test\Dialect\HFusion\hfusion-cache-io.mlir:58-85`

**使用方式**

- 运行命令（Windows）：

```bat
bishengir-opt -hfusion-cache-io -split-input-file bishengir\test\Dialect\HFusion\hfusion-cache-io.mlir
```

**补充**

- 跳过策略：已打 `hacc.cached_io` 的参数或返回会被跳过，避免重复缓存，见 `CacheIO.cpp:91-96, 145-148`
- 动态形状：创建 `tensor.empty` 时，会从 `init` 或通过 `ReifyRankedShapedTypeOpInterface` 重建形状，减少不必要的 `tensor.dim` 传播，见 `Utils.cpp:1009-1016`
- 相关配套 pass：
  - `hfusion-cache-io-for-return-arg`：专门为“返回即参数”的场景做缓存，并在函数类型上标注 `hacc.cached_io`，测试见 `bishengir\test\Dialect\HFusion\hfusion-cache-io-for-return-arg.mlir`
  - `hfusion-recache-io`：当一个 `hfusion.load` 之后被多个 `extract_slice` 分片使用时，重分配到各分片上以减少共享源竞争，测试见 `bishengir\test\Dialect\HFusion\hfusion-recache-io.mlir`

## hfusion-cache-io-for-return-arg

**作用**

- 针对函数里“直接返回某个输入参数”的情况，将该返回路径显式改写为一条 `hfusion.load` → `hfusion.store` 的 IO 缓存链，并在函数参数与对应返回值上标记 `hacc.cached_io` 属性
- 若该参数在函数中还参与了 `tensor.expand_shape` / `tensor.collapse_shape` 等变形，沿“直接返回”的支路补齐逆向变形，保证最终返回值的形状与原先一致
- 目的：把“直通返回”的参数统一纳入 IO 规划，便于后续缓冲分析、调度与多缓冲优化，避免与计算结果的返回路径不一致

**触发与行为**

- 触发条件：遍历 `func.return`，只要返回列表中包含 `BlockArgument`（即函数参数），就对这些“直接返回的参数”进行改写；如果所有返回值都是参数，则跳过（不做任何改写）
- 主要步骤：
  - 追踪该参数的其它使用路径上的 `expand/collapse` 变形，在“直接返回”的支路插入相应的逆向变形，直到遇到“重要算子”（如 elementwise、matmul、transpose 等）
  - 在被追踪到的值之后插入 `tensor.empty`，然后插入 `hfusion.load ins(arg) outs(empty)`，接着插入 `hfusion.store ins(loadRes) outs(empty)`，用 `store` 的结果替换 `func.return` 的对应操作数
  - 给该参数和对应返回值都设置 `hacc.cached_io` 属性；并在 `hfusion.store` 上写入 `hfusion.return_operand_num` 属性以保持唯一性、防止 CSE 合并
- 运行位置：该 pass 在 `lower-hfusion-pipeline` 的 `flattenAndFold` 阶段调用，见 `bishengir\lib\Dialect\HFusion\Pipelines\HFusionPipelines.cpp:171-175`

**简单示例（无变形）**
输入 IR（第二个返回值是参数直通）：

```mlir
func.func @f(%arg0: tensor<16x16xf32>, %arg1: tensor<16x16xf32>)
  -> (tensor<16x16xf32>, tensor<16x16xf32>) {
  %0 = linalg.elemwise_unary ins(%arg0 : tensor<16x16xf32>)
       outs(%arg1 : tensor<16x16xf32>) -> tensor<16x16xf32>
  func.return %0, %arg1 : tensor<16x16xf32>, tensor<16x16xf32>
}
```

应用 `hfusion-cache-io-for-return-arg` 后：

```mlir
func.func @f(%arg0: tensor<16x16xf32>, %arg1: tensor<16x16xf32> {hacc.cached_io})
  -> (tensor<16x16xf32>, tensor<16x16xf32> {hacc.cached_io}) {
  %empty0 = tensor.empty() : tensor<16x16xf32>
  %ld0 = hfusion.load ins(%arg1 : tensor<16x16xf32>)
         outs(%empty0 : tensor<16x16xf32>) -> tensor<16x16xf32>
  %empty1 = tensor.empty() : tensor<16x16xf32>
  %st0 = hfusion.store ins(%ld0 : tensor<16x16xf32>)
        outs(%empty1 : tensor<16x16xf32>) -> tensor<16x16xf32>
  func.return %0, %st0 : tensor<16x16xf32>, tensor<16x16xf32>
}
```

**简单示例（带变形）**
输入 IR（参数既被 `expand/collapse` 用于计算，又被直接返回）：

```mlir
func.func @g(%arg0: tensor<16x16xf32>)
  -> (tensor<64x4xf32>, tensor<16x16xf32>) {
  %asu = tensor.empty() : tensor<16x4x4xf32>
  %v1 = tensor.expand_shape %arg0 [[0], [1, 2]] output_shape [16, 4, 4]
        : tensor<16x16xf32> into tensor<16x4x4xf32>
  %v2 = linalg.elemwise_unary ins(%v1 : tensor<16x4x4xf32>)
        outs(%asu : tensor<16x4x4xf32>) -> tensor<16x4x4xf32>
  %v3 = tensor.collapse_shape %v2 [[0, 1], [2]]
        : tensor<16x4x4xf32> into tensor<64x4xf32>
  func.return %v3, %arg0 : tensor<64x4xf32>, tensor<16x16xf32>
}
```

应用后（在直通返回支路插入逆向变形与 load/store，保持最终返回形状一致）：

```mlir
func.func @g(%arg0: tensor<16x16xf32> {hacc.cached_io})
  -> (tensor<64x4xf32>, tensor<16x16xf32> {hacc.cached_io}) {
  %asu = tensor.empty() : tensor<16x4x4xf32>
  %v1 = tensor.expand_shape %arg0 [[0], [1, 2]] output_shape [16, 4, 4]
        : tensor<16x16xf32> into tensor<16x4x4xf32>
  %v2 = linalg.elemwise_unary ins(%v1 : tensor<16x4x4xf32>)
        outs(%asu : tensor<16x4x4xf32>) -> tensor<16x4x4xf32>
  %v3 = tensor.collapse_shape %v2 [[0, 1], [2]]
        : tensor<16x4x4xf32> into tensor<64x4xf32>
  %empty0 = tensor.empty() : tensor<16x4x4xf32>
  %ld0 = hfusion.load ins(%v1 : tensor<16x4x4xf32>)
         outs(%empty0 : tensor<16x4x4xf32>) -> tensor<16x4x4xf32>
  %empty1 = tensor.empty() : tensor<16x4x4xf32>
  %st0 = hfusion.store ins(%ld0 : tensor<16x4x4xf32>)
        outs(%empty1 : tensor<16x4x4xf32>) -> tensor<16x4x4xf32>
  %collapsed = tensor.collapse_shape %st0 [[0, 1], [2]]
               : tensor<16x4x4xf32> into tensor<16x16xf32>
  func.return %v3, %collapsed : tensor<64x4xf32>, tensor<16x16xf32>
}
```

**如何运行**

- Windows 命令行示例：

```bat
bishengir-opt.exe -hfusion-cache-io-for-return-arg input.mlir -cse -canonicalize-ext -o output.mlir
```

**代码引用**

- 定义与摘要：`bishengir\include\bishengir\Dialect\HFusion\Transforms\Passes.td:417-422`
- 具体实现与改写逻辑：`bishengir\lib\Dialect\HFusion\Transforms\CacheIOForReturnArg.cpp:117-206`
- 逆向变形插入逻辑：`bishengir\lib\Dialect\HFusion\Transforms\CacheIOForReturnArg.cpp:44-115`
- 管线集成位置：`bishengir\lib\Dialect\HFusion\Pipelines\HFusionPipelines.cpp:171-175`
- 行为测试样例：`bishengir\test\Dialect\HFusion\hfusion-cache-io-for-return-arg.mlir`

## hfusion-recache-io

**作用**

- 将某些 `hfusion.load` 的“切片使用场景”改写为“先对源数据切片，再对切片单独加载”，以：
  - 避免对 UB（片上缓冲）进行不对齐访问（默认按 32 字节对齐）
  - 避免多个互不重叠的切片在同一个已加载的大张量中发生越界或相互干扰
- 简单说，就是把“对已加载张量再切片”的模式，重写为“先对 GM 源张量切片，再对每个切片各自 `hfusion.load`”，从而更安全、更符合硬件访问约束

**触发与改写规则**

- 仅针对 `hfusion.load` 且“有多个使用者”的情况处理，单一使用者直接跳过
- 两类模式会触发改写：
  - 不对齐切片用户：只要某个 `tensor.extract_slice` 对 `hfusion.load` 的结果进行切片，其偏移按 32 字节换算不对齐，就把该切片改写为“对 `load` 的源直接切片”，并在其后插入新的 `hfusion.load` 读取该切片
  - 互斥切片用户：多组切片在至少一个维度范围上互不重叠，且其它维度范围一致，则对这些互斥切片做同样的下推改写
- 动态切片（动态 offset 或 size）不参与改写
- 改写细节：
  - 将 `extract_slice` 的 `source` 改为原始 `load` 的输入（GM 源）
  - 在该 `extract_slice` 之后插入一个新的 `hfusion.load`，其输入为该切片结果，输出为 `tensor.empty` 新建的目标张量
  - 用这个新的 `hfusion.load` 结果替换原切片的下游使用者（保留当前创建的 `load` 自身）

**示例：不对齐切片**
原始 IR（先整体 `load`，对结果两次切片，其中 `[1]` 不是 32 字节对齐的元素偏移）：

```mlir
%ld = hfusion.load ins(%src : tensor<2048xi32>) outs(%empty : tensor<2048xi32>) -> tensor<2048xi32>
%sl0 = tensor.extract_slice %ld[1] [2047] [1] : tensor<2048xi32> to tensor<2047xi32>
%sl1 = tensor.extract_slice %ld[0] [2047] [1] : tensor<2048xi32> to tensor<2047xi32>
... uses %sl0, %sl1 ...
```

改写后（将不对齐的 `%sl0` 下推到源，给每个切片各自 `load`）：

```mlir
%ld = hfusion.load ins(%src : tensor<2048xi32>) outs(%empty : tensor<2048xi32>) -> tensor<2048xi32>
%sl0_src = tensor.extract_slice %src[1] [2047] [1] : tensor<2048xi32> to tensor<2047xi32>
%ld0 = hfusion.load ins(%sl0_src : tensor<2047xi32>) outs(%empty0 : tensor<2047xi32>) -> tensor<2047xi32>
%sl1 = tensor.extract_slice %ld[0] [2047] [1] : tensor<2048xi32> to tensor<2047xi32>
... uses %ld0, %sl1 ...
```

参考测试：`bishengir\test\Dialect\HFusion\hfusion-recache-io.mlir:3-27`

**示例：互斥切片**
原始 IR（把一个 4096x36864 的张量均分为两个不重叠的 4096x18432 切片）：

```mlir
%ld = hfusion.load ins(%arg0 : tensor<4096x36864xbf16>) outs(%empty) -> tensor<4096x36864xbf16>
%slA = tensor.extract_slice %ld[0, 0] [4096, 18432] [1, 1]  : tensor<4096x36864xbf16> to tensor<4096x18432xbf16>
%slB = tensor.extract_slice %ld[0, 18432] [4096, 18432] [1, 1] : tensor<4096x36864xbf16> to tensor<4096x18432xbf16>
... uses %slA, %slB ...
```

改写后（对源各自切片并各自 `load`）：

```mlir
%slA_src = tensor.extract_slice %arg0[0, 0] [4096, 18432] [1, 1]  : tensor<4096x36864xbf16> to tensor<4096x18432xbf16>
%ldA = hfusion.load ins(%slA_src : tensor<4096x18432xbf16>) outs(%emptyA) -> tensor<4096x18432xbf16>
%slB_src = tensor.extract_slice %arg0[0, 18432] [4096, 18432] [1, 1] : tensor<4096x36864xbf16> to tensor<4096x18432xbf16>
%ldB = hfusion.load ins(%slB_src : tensor<4096x18432xbf16>) outs(%emptyB) -> tensor<4096x18432xbf16>
... uses %ldA, %ldB ...
```

参考测试：`bishengir\test\Dialect\HFusion\hfusion-recache-io.mlir:31-48`

**不改写的场景**

- `hfusion.load` 只有一个使用者
- 切片的 `offset/size` 含动态值
- 所有切片在每个维度上范围都完全相同（没有互斥维度）
- 切片部分重叠但不满足“至少一个维度互斥，其它维度相同”的条件

**如何运行**

- Windows 命令行示例：

```bat
bishengir-opt.exe -hfusion-recache-io input.mlir -o output.mlir
```

**代码引用**

- 定义与构造器：`bishengir\include\bishengir\Dialect\HFusion\Transforms\Passes.td:424-430`
- 改写核心：
  - 不对齐切片模式：`bishengir\lib\Dialect\HFusion\Transforms\ReCacheIO.cpp:69-100`
  - 互斥切片模式：`bishengir\lib\Dialect\HFusion\Transforms\ReCacheIO.cpp:114-242`
  - 切片改写和新 `load` 插入：`bishengir\lib\Dialect\HFusion\Transforms\ReCacheIO.cpp:40-55`
  - pass 入口：`bishengir\lib\Dialect\HFusion\Transforms\ReCacheIO.cpp:244-261`
- 自动调度中调用位置：`bishengir\lib\Dialect\HFusion\Transforms\AutoSchedule\AutoScheduleBase.cpp:1136-1142`

## hfusion-hoist-tensor-empty

**作用**

- 将函数体内的所有 `tensor.empty` 输出缓冲统一“上提”为函数参数，用一个“工作区间”参数集中承载它们的内存
- 对每个原先的 `tensor.empty`，以该工作区间参数的不同子切片替代，并还原为与原 `empty` 相同形状的张量供后续运算使用
- 为动态大小的工作区间生成并注册一个 Host 侧“工作区大小推断函数”，在调用点先推断大小、再分配工作区并传入被调用函数
- 适用范围仅限设备入口函数，且融合类型为 `ShallowCV`、`ShallowVV` 或 `MixCV`

**触发与条件**

- 仅处理设备入口函数，且函数具有 `hfusion.fusion_kind` 属性为 `ShallowCV`/`ShallowVV`/`MixCV`，见 `bishengir\lib\Dialect\HFusion\Transforms\HoistTensorEmpty.cpp:499-511`
- 要求所有 `tensor.empty` 的元素类型一致；否则报错，见 `HoistTensorEmpty.cpp:194-205`

**关键改写**

- 复制共享的 `tensor.empty`：对拥有多处使用的 `tensor.empty` 先克隆，使每处使用独立，便于后续替换，见 `HoistTensorEmpty.cpp:75-93`
- 计算统一“工作区”大小：
  - 静态维度：按元素类型字节数统一尺度，累加每个 `empty` 的元素总数，见 `HoistTensorEmpty.cpp:180-192`
  - 动态维度：乘积计算总元素数后再按类型字节数缩放，见 `HoistTensorEmpty.cpp:132-168`
- 在被处理函数末尾新增一个工作区参数：类型为 `memref<workspaceSize x elementType>`，并标注 `hacc.kernel_arg_type = workspace`，见 `HoistTensorEmpty.cpp:207-224`
- 用工作区子视图替代每个 `tensor.empty`：
  - 为每个 `empty` 依次切出一段 `memref.subview`，`memref.reinterpret_cast` 成合适类型后 `bufferization.to_tensor` 转为张量，必要时 `tensor.expand_shape` 恢复多维形状
  - 将原 `empty` 的结果全部替换为该张量，累加偏移继续处理下一个 `empty`，见 `HoistTensorEmpty.cpp:293-321`
- 生成并挂接 Host 侧“推断工作区大小”函数：
  - 追踪并记录用于计算总大小/偏移的 IR，克隆为一个新的 `func`，标注为 Host 且类型 `InferWorkspaceShapeFunction`，并在原函数上写入引用属性，见 `HoistTensorEmpty.cpp:379-395`
  - 校验该推断函数不含 `linalg` 与 `func.call`，见 `HoistTensorEmpty.cpp:397-409`
- 修补调用点：
  - 对所有调用该函数的 `func.call`，静态大小则直接 `memref.alloc`；动态大小则先调用推断函数得到大小，再 `memref.alloc`，最后把新分配的工作区作为额外参数插入调用操作数，见 `HoistTensorEmpty.cpp:422-458`
- 擦除所有已替换且无用的 `tensor.empty`，见 `HoistTensorEmpty.cpp:486-493`

**简单示例（静态形状）**
原始 IR（函数体中用两个 `tensor.empty` 承接输出）：

```mlir
func.func @kernel(%a: tensor<8x8xf32>) -> (tensor<8x8xf32>, tensor<4x4xf32>) {
  %out0 = tensor.empty() : tensor<8x8xf32>
  %out1 = tensor.empty() : tensor<4x4xf32>
  %r0 = linalg.elemwise_unary ins(%a) outs(%out0) -> tensor<8x8xf32>
  %r1 = linalg.elemwise_unary ins(%a) outs(%out1) -> tensor<4x4xf32>
  return %r0, %r1 : tensor<8x8xf32>, tensor<4x4xf32>
}
```

应用 `hfusion-hoist-tensor-empty` 后（概念示意）：

```mlir
func.func @kernel(
  %a: tensor<8x8xf32>,
  %workspace: memref<96xf32> {hacc.kernel_arg_type = "workspace"}
) -> (tensor<8x8xf32>, tensor<4x4xf32>) {
  %off0 = arith.constant 0 : index
  %sub0 = memref.subview %workspace[%off0] [64] [1] : memref<96xf32> to memref<64xf32>
  %t0   = bufferization.to_tensor %sub0 ...
  %expanded0 = tensor.expand_shape %t0 [[0,1]] output_shape [8,8] : tensor<64xf32> into tensor<8x8xf32>
  %r0 = linalg.elemwise_unary ins(%a) outs(%expanded0) -> tensor<8x8xf32>

  %off1 = arith.constant 64 : index
  %sub1 = memref.subview %workspace[%off1] [16] [1] : memref<96xf32> to memref<16xf32>
  %t1   = bufferization.to_tensor %sub1 ...
  %expanded1 = tensor.expand_shape %t1 [[0,1]] output_shape [4,4] : tensor<16xf32> into tensor<4x4xf32>
  %r1 = linalg.elemwise_unary ins(%a) outs(%expanded1) -> tensor<4x4xf32>

  return %r0, %r1 : tensor<8x8xf32>, tensor<4x4xf32>
}
```

调用点需新增工作区分配并传参（静态大小直接分配，动态大小先调用推断函数获得尺寸后再分配）。

**运行方法**

- Windows 命令行：

```bat
bishengir-opt.exe -hfusion-hoist-tensor-empty input.mlir -o output.mlir
```

- 在完整管线中，该 pass 位于后处理阶段，见 `bishengir\lib\Dialect\HFusion\Pipelines\HFusionPipelines.cpp:269`

**代码引用**

- 定义与描述：`bishengir\include\bishengir\Dialect\HFusion\Transforms\Passes.td:432-442`
- 核心实现入口：`bishengir\lib\Dialect\HFusion\Transforms\HoistTensorEmpty.cpp:496-524`
- 复制共享 `tensor.empty`：`HoistTensorEmpty.cpp:75-93`
- 合并到工作区参数与替换逻辑：`HoistTensorEmpty.cpp:272-327`
- 推断函数的生成与校验：`HoistTensorEmpty.cpp:379-409`
- 修补调用点并按需分配工作区：`HoistTensorEmpty.cpp:422-458`

## hfusion-wrap-host-func

**作用**

- 为与设备核相关的“宿主侧函数”自动生成包装函数，并将设备核上的引用更新为这些包装函数
- 包装的目标包括：`tiling_function`、`infer_output_shape_function`、`infer_workspace_shape_function`、`get_tiling_struct_size_function`、`infer_sync_block_lock_num_function`、`infer_sync_block_lock_init_function`
- 包装函数的参数对齐宿主入口函数的输入，返回值沿用原被包装函数的返回类型；函数体会回溯调用点上需要的实参计算，将其克隆进包装函数，然后在包装函数内调用原始宿主函数
- 可选移除包装函数中“未使用的形参”（`removeUnusedArguments` 选项），以收紧包装函数签名

**触发与流程**

- 遍历模块，寻找设备入口函数，定位其宿主侧调用点，见 `bishengir/lib/Dialect/HFusion/Transforms/WrapHostFunc.cpp:283-305`
- 若设备函数标注了上述宿主函数属性，则：
  - 生成包装函数名：以“宿主入口函数”名为前缀 + 宿主函数类型后缀，见 `WrapHostFunc.cpp:183-190`
  - 选择需要“回溯克隆”的调用实参：依据 `hacc.arg_type`（input/output 等）及其 `input_idx/output_idx` 映射，与设备端形参位置匹配，见 `WrapHostFunc.cpp:81-146`
  - 克隆用于构造这些实参的 IR 到新包装函数入口块，见 `WrapHostFunc.cpp:56-79, 221-229`
  - 在包装函数中调用原始宿主函数并返回其结果；为包装函数设置宿主属性与 `HostFuncType`，并把设备函数上的原引用改成包装函数名，见 `WrapHostFunc.cpp:226-243`
  - 若启用移除未使用参数，则删除未用的块参数并收紧函数类型，见 `WrapHostFunc.cpp:148-170`
- 集成位置：在完整管线的后段，对单核情况默认启用，见 `bishengir/lib/Dialect/HFusion/Pipelines/HFusionPipelines.cpp:250-253`

**简单示例**
假设设备核 `@kernel` 需要一个宿主侧“推断输出形状”函数 `@infer_shape_impl`，宿主入口为 `@host_entry`。

原始 IR（概念示例）：

```mlir
func.func @infer_shape_impl(%x: tensor<?x1xf16>, %y: tensor<1x?xf16>)
  -> tensor<2xindex>
  attributes {hacc.function_kind = #hacc.function_kind<HOST>,
              hacc.host_func_type = #hacc.host_func_type<infer_output_shape_function>} {
  %c0 = arith.constant 0 : index
  %c1 = arith.constant 1 : index
  %d0 = tensor.dim %x, %c0 : tensor<?x1xf16>
  %d1 = tensor.dim %y, %c1 : tensor<1x?xf16>
  %shape = tensor.from_elements %d0, %d1 : tensor<2xindex>
  return %shape : tensor<2xindex>
}

func.func @kernel(%x: tensor<?x1xf16>, %y: tensor<1x?xf16>, %out: tensor<?x?xf16>)
  attributes {hacc.infer_output_shape_function = #hacc.infer_output_shape_function<@infer_shape_impl>} {
  ...
}

func.func @host_entry(%x: tensor<?x1xf16>, %y: tensor<1x?xf16>, %out: tensor<?x?xf16>) attributes {hacc.entry, hacc.function_kind = #hacc.function_kind<HOST>} {
  %res = func.call @kernel(%x, %y, %out) : (...)
  return %res : tensor<?x?xf16>
}
```

应用 `hfusion-wrap-host-func` 后（概念示例）：

```mlir
func.func @host_entry_infer_output_shape_function(
  %x: tensor<?x1xf16>, %y: tensor<1x?xf16>) -> tensor<2xindex>
  attributes {hacc.function_kind = #hacc.function_kind<HOST>,
              hacc.host_func_type = #hacc.host_func_type<infer_output_shape_function>} {
  %c0 = arith.constant 0 : index
  %c1 = arith.constant 1 : index
  %d0 = tensor.dim %x, %c0 : tensor<?x1xf16>
  %d1 = tensor.dim %y, %c1 : tensor<1x?xf16>
  %shape = func.call @infer_shape_impl(%x, %y) : (...) -> tensor<2xindex>
  return %shape : tensor<2xindex>
}

func.func @kernel(...) attributes {
  hacc.infer_output_shape_function
    = #hacc.infer_output_shape_function<@host_entry_infer_output_shape_function>
}
```

- 包装函数参数对齐宿主入口 `@host_entry` 的输入集，内部克隆并重用必要的计算（此例为 `tensor.dim` 等），最终调用原 `@infer_shape_impl`
- 设备核的属性改为指向新包装函数，确保调度/调用的一致性
- 若某些包装函数形参在函数体中未被使用且设置了 `removeUnusedArguments=true`，将被移除

可参考内置测试样例：`bishengir\test\Dialect\HFusion\hfusion-wrap-host-func.mlir`，其中演示了 `infer_output_shape_function`、`tiling_function` 等包装与调用的关系。

**如何运行**

- Windows 命令行示例：

```bat
bishengir-opt.exe -hfusion-wrap-host-func input.mlir -o output.mlir
```

**代码引用**

- Pass 定义与选项：`bishengir\include\bishengir\Dialect\HFusion\Transforms\Passes.td:444-459`
- 入口与调度：`bishengir\lib\Dialect\HFusion\Transforms\WrapHostFunc.cpp:283-305`
- 参数匹配与实参回溯：`WrapHostFunc.cpp:81-146`
- 包装函数生成与属性设置：`WrapHostFunc.cpp:172-243`
- 移除未用参数：`WrapHostFunc.cpp:148-170`
- 管线集成：`bishengir\lib\Dialect\HFusion\Pipelines\HFusionPipelines.cpp:250-253`

## hfusion-fold-symbolic-dim

**作用**

- 将 `tensor.dim` 的结果和值替换为具名的 `hfusion.symbolic_dim` 索引，以统一维度语义并便于后续融合、规范化和符号分析
- 同步处理 `tensor.empty` 的动态形参：若某一维是动态并且类型上携带符号维编码，则把该动态形参替换为对应的 `hfusion.symbolic_dim`
- 与之相对的“展开”pass 会把 `hfusion.symbolic_dim` 还原为原来的 `tensor.dim`，在管线中一对配合使用

**触发与行为**

- 对 `tensor.dim`：
  - 仅当 `dim` 的轴索引是常量时进行替换
  - 从源张量类型的 `encoding` 中读取对应维的符号名，插入 `hfusion.symbolic_dim @SymName : index`，并用其替换所有原 `tensor.dim` 的使用者
- 对 `tensor.empty`：
  - 遍历所有动态维的 `mixedSizes`，若该维的类型 `encoding` 提供了符号名且当前操作数不是 `hfusion.symbolic_dim`，则生成并替换为符号维
- 插入点选择在函数首块开头，保证符号索引的可见性与可复用性

**简单示例**

- 折叠 `tensor.dim` 为符号维

```mlir
// 之前：从第1维读 size
func.func @f(%arg0: tensor<?x?xf32, encoding = [@S0, @S1]>) -> index {
  %c1 = arith.constant 1 : index
  %d1 = tensor.dim %arg0, %c1 : tensor<?x?xf32, encoding = [@S0, @S1]>
  return %d1 : index
}

// 之后：直接引用具名符号维
func.func @f(%arg0: tensor<?x?xf32, encoding = [@S0, @S1]>) -> index {
  %sd1 = hfusion.symbolic_dim @S1 : index
  return %sd1 : index
}
```

- 折叠 `tensor.empty` 的动态形参

```mlir
// 之前：两个动态维由 SSA 值传入
%sz0 = arith.constant 128 : index
%sz1 = arith.constant 64 : index
%empty = tensor.empty(%sz0, %sz1)
         : tensor<?x?xf32, encoding = [@S0, @S1]>

// 之后：用符号维驱动空张量形状
%sd0 = hfusion.symbolic_dim @S0 : index
%sd1 = hfusion.symbolic_dim @S1 : index
%empty = tensor.empty(%sd0, %sd1)
         : tensor<?x?xf32, encoding = [@S0, @S1]>
```

**管线位置与配套**

- 在推断与外提阶段，先折叠符号维，后在轮廓化等处理完成后再展开回 `tensor.dim`，最后丢弃符号：
  - 折叠：`bishengir\lib\Dialect\HFusion\Pipelines\HFusionPipelines.cpp:179`
  - 展开：`bishengir\lib\Dialect\HFusion\Pipelines\HFusionPipelines.cpp:195`
  - 去符号：`bishengir\lib\Dialect\HFusion\Pipelines\HFusionPipelines.cpp:197`

**注意点**

- 仅当源类型为 `RankedTensorType` 且其 `encoding` 是 `ArrayAttr`，每个维都对应一个 `SymbolRefAttr` 时才会生效
- `tensor.dim` 的轴必须是常量；非常量轴不处理
- 若 `encoding` 中缺少该维的符号或类型不匹配，则跳过该替换

**如何运行**

- Windows 命令行：

```bat
bishengir-opt.exe -hfusion-fold-symbolic-dim input.mlir -o output.mlir
```

- 若需要还原：

```bat
bishengir-opt.exe -hfusion-unfold-symbolic-dim input.mlir -o output.mlir
bishengir-opt.exe -hfusion-drop-symbols input.mlir -o output.mlir
```

**代码引用**

- 定义与摘要：`bishengir\include\bishengir\Dialect\HFusion\Transforms\Passes.td:461-464`
- 核心实现：
  - `tensor.dim` 替换：`bishengir\lib\Dialect\HFusion\Transforms\FoldSymbolicDim.cpp:79-105`
  - `tensor.empty` 动态形参替换：`bishengir\lib\Dialect\HFusion\Transforms\FoldSymbolicDim.cpp:48-76`
  - Pass 入口：`bishengir\lib\Dialect\HFusion\Transforms\FoldSymbolicDim.cpp:112-135`
- 反向展开：`bishengir\lib\Dialect\HFusion\Transforms\UnfoldSymbolicDim.cpp:77-95`
- 符号维操作定义：`bishengir\include\bishengir\Dialect\HFusion\IR\HFusionOps.td:88-100`
- 类型符号读取工具：`bishengir\lib\Dialect\HFusion\Utils\Utils.cpp:1140-1149`

## hfusion-unfold-symbolic-dim

**作用**

- 将 `hfusion.symbolic_dim @SymName : index` 恢复为对应函数参数上的 `tensor.dim` 读取，按符号维在参数类型 `encoding` 中出现的位置选择轴索引
- 与“折叠”pass 相反，用实际的 `tensor.dim` 替换符号索引，便于后续标准化、下游转换或不再依赖符号体系时的处理

**触发与行为**

- 遍历函数体内所有 `hfusion.symbolic_dim`：
  - 扫描父 `func.func` 的每个形参类型的 `encoding`（`ArrayAttr`），寻找与该 `symbolic_dim` 的 `symbolName` 完全匹配的符号
  - 找到后，在函数首块插入 `tensor.dim %arg, <matched-index>`，用其替换当前 `symbolic_dim` 的所有使用者
  - 若任何一个 `symbolic_dim` 未找到匹配参数与维度，整个 pass 失败
- 插入位置：函数首块起始处，保证新建的 `dim` 值可全函数复用

**简单示例**

- 将符号维还原为 `tensor.dim`

```mlir
// 之前：以符号维引用动态维度
func.func @f(%arg0: tensor<?x?xf32, encoding = [@S0, @S1]>) -> index {
  %sd1 = hfusion.symbolic_dim @S1 : index
  return %sd1 : index
}

// 之后：从对应参数读取具体维度
func.func @f(%arg0: tensor<?x?xf32, encoding = [@S0, @S1]>) -> index {
  %d1 = tensor.dim %arg0, 1 : tensor<?x?xf32, encoding = [@S0, @S1]>
  return %d1 : index
}
```

- 与“折叠”配合的往返示例

```mlir
// 折叠后（由 FoldSymbolicDim 得到）
%sd0 = hfusion.symbolic_dim @S0 : index
%sd1 = hfusion.symbolic_dim @S1 : index
%empty = tensor.empty(%sd0, %sd1)
         : tensor<?x?xf32, encoding = [@S0, @S1]>

// 展开后（由 UnfoldSymbolicDim 还原）
%d0 = tensor.dim %arg0, 0 : tensor<?x?xf32, encoding = [@S0, @S1]>
%d1 = tensor.dim %arg0, 1 : tensor<?x?xf32, encoding = [@S0, @S1]>
%empty = tensor.empty(%d0, %d1)
         : tensor<?x?xf32, encoding = [@S0, @S1]>
```

**管线与配套**

- 典型流程（推断/轮廓阶段）：
  - 先折叠符号维：`hfusion-fold-symbolic-dim`
  - 中间进行融合/轮廓
  - 再展开符号维：`hfusion-unfold-symbolic-dim`
  - 最后丢弃符号：`hfusion-drop-symbols`
- 调用顺序参考：`bishengir\lib\Dialect\HFusion\Pipelines\HFusionPipelines.cpp:179,195,197`

**运行方法**

- Windows 命令行：

```bat
bishengir-opt.exe -hfusion-unfold-symbolic-dim input.mlir -o output.mlir
```

**代码引用**

- 定义与摘要：`bishengir\include\bishengir\Dialect\HFusion\Transforms\Passes.td:466-469`
- 展开实现（查找匹配参数维并替换）：`bishengir\lib\Dialect\HFusion\Transforms\UnfoldSymbolicDim.cpp:48-71`
- pass 入口与遍历逻辑：`bishengir\lib\Dialect\HFusion\Transforms\UnfoldSymbolicDim.cpp:77-95`
- 相关折叠实现（供对比理解）：`bishengir\lib\Dialect\HFusion\Transforms\FoldSymbolicDim.cpp:79-105,48-76`
- 类型符号读取工具：`bishengir\lib\Dialect\HFusion\Utils\Utils.cpp:1140-1149`

## hfusion-drop-symbols

**作用**

- 删除 `RankedTensorType` 上的符号维编码（类型 `encoding` 中保存的 `SymbolRefAttr` 列表），使张量类型回落为普通的有形状类型
- 统一在整个函数范围内执行：函数参数、块参数、所有操作结果的类型都会去除符号编码；同时更新函数签名的输入/输出类型
- 典型用法是在“折叠/展开符号维”（Fold/Unfold）之后，彻底移除符号痕迹，便于后续标准化和下游转换

**行为细节**

- 仅处理 `RankedTensorType`；维度形状与元素类型保持不变，只去掉类型的 `encoding`
- 处理范围：
  - 函数参数与其块参数的类型
  - 函数体内所有操作结果类型
  - 函数类型签名（输入/输出）
- 不改写 IR 逻辑，只是类型层面的元数据清理

**简单示例**

- 输入 IR（带符号编码）：

```mlir
func.func @f(%arg0: tensor<?x?xf32, encoding = [@S0, @S1]>)
  -> tensor<?x?xf32, encoding = [@S0, @S1]> {
  %0 = "some.op"(%arg0) : (tensor<?x?xf32, encoding = [@S0, @S1]>)
                          -> tensor<?x?xf32, encoding = [@S0, @S1]>
  return %0 : tensor<?x?xf32, encoding = [@S0, @S1]>
}
```

- 应用 `hfusion-drop-symbols` 后：

```mlir
func.func @f(%arg0: tensor<?x?xf32>) -> tensor<?x?xf32> {
  %0 = "some.op"(%arg0) : (tensor<?x?xf32>) -> tensor<?x?xf32>
  return %0 : tensor<?x?xf32>
}
```

**管线位置**

- 在 HFusion 主流程中与符号维配套使用：
  - 展开符号维：`hfusion-unfold-symbolic-dim`
  - 丢弃符号：`hfusion-drop-symbols`
- 参考：`bishengir\lib\Dialect\HFusion\Pipelines\HFusionPipelines.cpp:195-197`

**如何运行**

- Windows 命令行：

```bat
bishengir-opt.exe -hfusion-drop-symbols input.mlir -o output.mlir
```

**代码引用**

- Pass 实现与入口：`bishengir\lib\Dialect\HFusion\Transforms\DropSymbols.cpp:91-100`
- 类型去符号逻辑（值/类型/函数）：
  - `dropSymbolsFromType`：`bishengir\lib\Dialect\HFusion\Transforms\DropSymbols.cpp:47-54`
  - 应用于值与结果：`bishengir\lib\Dialect\HFusion\Transforms\DropSymbols.cpp:63-70, 75-83`
- 定义与摘要：`bishengir\include\bishengir\Dialect\HFusion\Transforms\Passes.td:471-474`

## hfusion-eliminate-duplicate-funcs

**作用**

- 删除模块中“内容完全相同”的函数副本，统一到一个“规范函数”，并更新宿主侧调用点的符号引用
- 同时清理与设备函数绑定的宿主辅助函数（如 `tiling_function`、`infer_output_shape_function` 等），避免保留孤立副本

**判等规则**

- 函数签名一致：比较 `FunctionType` 与属性集合的结构一致性
- 属性值一致，但忽略以下属性的具体值：`hacc::InferOutputShapeFunctionAttr`、`hacc::TilingFunctionAttr`、`hacc::InferWorkspaceShapeFunctionAttr`、`hacc::GetTilingStructSizeFunctionAttr`、`hacc::InferSyncBlockLockNumFunctionAttr`、`hacc::InferSyncBlockLockInitFunctionAttr`、以及一般的 `mlir::StringAttr`（仅要求同名）
- 函数体逐条操作等价：使用 `OperationEquivalence::isEquivalentTo`，忽略 SSA 值编号和位置信息（locations）

代码参考：

- 判等与替换流程：`bishengir\lib\Dialect\HFusion\Transforms\EliminateDuplicateFuncs.cpp:55-103, 106-126, 129-145`
- 绑定宿主辅助函数的清理：`EliminateDuplicateFuncs.cpp:166-193`
- 模块入口：`EliminateDuplicateFuncs.cpp:223-233`
- 管线集成：`bishengir\lib\Dialect\HFusion\Pipelines\HFusionPipelines.cpp:197-198`

**行为步骤**

- 收集设备函数与宿主函数，分别构建“重复函数替换表”（旧函数→规范函数）
- 在宿主函数内，将所有对“旧函数”的符号引用替换为“规范函数”
- 删除重复的设备函数；并依据设备函数上的宿主绑定属性，额外删除对应的宿主辅助函数副本

**简单示例 1：设备函数去重**
输入（两个设备核完全相同，宿主主函数各调用一次）：

```mlir
module {
  func.func @kernel_A(%x: tensor<4xf32>) attributes {hacc.function_kind = #hacc.function_kind<DEVICE>} {
    %e = tensor.empty() : tensor<4xf32>
    %r = linalg.elemwise_unary ins(%x) outs(%e) -> tensor<4xf32>
    func.return %r : tensor<4xf32>
  }

  func.func @kernel_B(%x: tensor<4xf32>) attributes {hacc.function_kind = #hacc.function_kind<DEVICE>} {
    %e = tensor.empty() : tensor<4xf32>
    %r = linalg.elemwise_unary ins(%x) outs(%e) -> tensor<4xf32>
    func.return %r : tensor<4xf32>
  }

  func.func @host(%x: tensor<4xf32>) attributes {hacc.entry, hacc.function_kind = #hacc.function_kind<HOST>} {
    %r0 = func.call @kernel_A(%x) : (tensor<4xf32>) -> tensor<4xf32>
    %r1 = func.call @kernel_B(%x) : (tensor<4xf32>) -> tensor<4xf32>
    func.return %r0 : tensor<4xf32>
  }
}
```

应用后（将 `kernel_B` 的调用改为 `kernel_A`，并删除 `kernel_B`）：

```mlir
module {
  func.func @kernel_A(...){ ... }
  func.func @host(%x: tensor<4xf32>) attributes {hacc.entry, hacc.function_kind = #hacc.function_kind<HOST>} {
    %r0 = func.call @kernel_A(%x) : (tensor<4xf32>) -> tensor<4xf32>
    %r1 = func.call @kernel_A(%x) : (tensor<4xf32>) -> tensor<4xf32>
    func.return %r0 : tensor<4xf32>
  }
}
```

**简单示例 2：宿主辅助函数去重**
若两个设备核完全一致且都绑定到它们各自的 `infer_output_shape_function`，当设备核被去重时，会同时根据属性名删除重复的宿主函数副本（保持一个统一包装/推断函数），见属性处理模板：`EliminateDuplicateFuncs.cpp:158-186`

**注意点**

- 仅在函数体严格等价时去重；常量不同、维度索引不同或额外操作均会阻止合并
- 对属性的“值忽略”仅限上述特定类型及 `StringAttr`；其它属性值需要完全一致
- 替换调用只在宿主函数范围进行；之后直接删除重复的设备/宿主函数

**如何运行**

- Windows 命令行：

```bat
bishengir-opt.exe -hfusion-eliminate-duplicate-funcs input.mlir -o output.mlir
```

## hfusion-decompose

**作用**

- 拆解实现了 `BiShengIRAggregatedOpInterface` 的“聚合算子”，把一个复杂操作分解为一系列更基础、易调度的操作；目前在 HFusion 中用于把“多轴” `linalg.transpose` 拆成若干只交换两轴的二元 `transpose`。
- 通过 `DecomposePhase` 精确控制分解的时机；默认 `NO_CONSTRAINT` 表示全阶段可分解，也可指定 `AFTER_HFUSION_FLATTEN` 仅在整洁扁平化后生效。

**实现要点**

- Pass 定义与选项：`bishengir\include\bishengir\Dialect\HFusion\Transforms\Passes.td:481-495`
  - 名称 `hfusion-decompose`，目标 `func::FuncOp`
  - 选项 `--hfusion-decompose-phase={no-constraint|after-hfusion-flatten}`
  - 构造函数 `mlir::hfusion::createDecomposePass()`
- 模式执行逻辑：`bishengir\lib\Dialect\HFusion\Transforms\Decompose.cpp:45-78,87-95,97-100`
  - 使用 `OpInterfaceRewritePattern<BiShengIRAggregatedOpInterface>` 匹配实现接口的 op
  - 仅当 `op.getDecomposePhase()` 与传入的 phase 一致或为 `NO_CONSTRAINT` 时分解
  - 调用 `op.decomposeOperation(rewriter)` 获取新结果，`rewriter.replaceOp(op, newResults)`
- 具体接口实现（以多轴 transpose 为例）：`bishengir\lib\Dialect\HFusion\Transforms\DecomposeOpInterfaceImpl.cpp:35-136,139-144`
  - 给 `linalg::TransposeOp` 附加外部模型，实现：
    - 判定是否需要分解：排列与恒等 `[0,1,...,n-1]` 的“偏差”大于 2 轴时才分解
    - 计算最少交换对（最小 swap 序列）
    - 对每个交换步骤：构造中间 `tensor.empty`（动态维用 `tensor.dim`），执行一次只交换两轴的 `linalg.transpose`

**管线位置**

- 预处理阶段（无约束，清理早期复杂聚合）：`bishengir\lib\Dialect\HFusion\Pipelines\HFusionPipelines.cpp:105`
- 自动调度前（在扁平化之后专门拆解 transpose）：`bishengIR\lib\Dialect\HFusion\Pipelines\HFusionPipelines.cpp:225`
- 后处理（再次确保多轴 transpose 已拆解）：`bishengir\lib\Dialect\HFusion\Pipelines\HFusionPipelines.cpp:276`

**简单示例**

- 输入（多轴 transpose，需拆解）

```mlir
# 初始：tensor<2x16x8x4x3xf32> 经过多轴转置到 tensor<2x3x4x8x16xf32>
%empty = tensor.empty() : tensor<2x3x4x8x16xf32>
%r = linalg.transpose ins(%arg0 : tensor<2x16x8x4x3xf32>)
                       outs(%empty)
                       permutation = [0, 4, 3, 2, 1]
```

- 输出（拆解为两次只交换两轴的 transpose）

```mlir
# 第一步：交换 1 与 4 轴 -> [0, 4, 2, 3, 1]
%empty1 = tensor.empty() : tensor<2x3x8x4x16xf32>
%t1 = linalg.transpose ins(%arg0) outs(%empty1)
      permutation = [0, 4, 2, 3, 1]

# 第二步：交换 2 与 3 轴 -> [0, 1, 3, 2, 4]
%empty2 = tensor.empty() : tensor<2x3x4x8x16xf32>
%t2 = linalg.transpose ins(%t1) outs(%empty2)
      permutation = [0, 1, 3, 2, 4]

# 最终结果：%t2 等价于原始的多轴转置 %r
```

- 动态形状时，`tensor.empty` 的动态维由 `tensor.dim` 提供，接口实现已处理：`bishengir\lib\Dialect\HFusion\Transforms\DecomposeOpInterfaceImpl.cpp:91-105`

**何时不会分解**

- 二元 transpose（排列与恒等仅两处不同）将被判定为“无需分解”：`bishengir\lib\Dialect\HFusion\Transforms\DecomposeOpInterfaceImpl.cpp:119-123`
- `getDecomposePhase()` 与传入 phase 不匹配且非 `NO_CONSTRAINT` 时跳过：`bishengir\lib\Dialect\HFusion\Transforms\Decompose.cpp:60-64`

**扩展建议**

- 若需对其他聚合算子进行拆解，给该 op 附加 `BiShengIRAggregatedOpInterface` 的 ExternalModel，并实现：
  - `decomposeOperation(OpBuilder&) -> FailureOr<SmallVector<Value>>`
  - `getDecomposePhase() -> DecomposePhase`（决定在管线中的拆解时机）
- 参考 transpose 的实现与注册方法：`bishengir\lib\Dialect\HFusion\Transforms\DecomposeOpInterfaceImpl.cpp:139-144`

# Pass 定义文件：`Dialect_HIVM_Transforms_Passes.td`

## hivm-infer-func-core-type

​       **作用**

- 在 `ModuleOp` 上为每个设备侧 `func.func` 推断核心类型并写入 `hivm.func_core_type` 属性，取值为 `AIC`(Cube)、`AIV`(Vector) 或 `MIX`。
- 汇总整个模块的函数核心类型，给 `module` 写入 `hivm.module_core_type` 属性，若模块内存在不同核心类型则为 `MIX`。
- 遵循已有显式标注：若函数已带 `hivm.func_core_type`，推断不会覆盖。
- 跳过宿主侧函数（Host）；仅对设备侧函数（Device）推断。
- 若出现无法解析的间接调用、未能推断的核心类型等情况，会标记失败并使 pass 失败。

**判定依据**

- 构建调用图并按后序遍历，使被调函数先于调用者完成推断，结合调用关系综合约束，得到调用者的核心类型约束。参考 `InferFuncCoreType.cpp:55-140`。
- 对函数体内实现了 `CoreTypeInterface` 的 HIVM 操作查询其核心类型并合并约束，例如：
  - `mmadL1` 等矩阵乘操作 → `CUBE`，见 `InferCoreTypeInterface/InferCoreType.cpp:30-77`、矩阵相关实现；
  - `vbrc` 广播到 UB → `VECTOR`，见 `InferCoreTypeInterface/InferCoreType.cpp:306-322`；
  - `copy` 根据源/目的地址空间对判断 → 可能是 `VECTOR` 或 `CUBE`，见 `InferCoreTypeInterface/InferCoreType.cpp:253-277`。
- 合并规则：只出现 Cube → `AIC`，只出现 Vector → `AIV`，混合出现 → `MIX`。显式标注直接取值。无约束时当前实现暂时默认 `AIV`（Vector），见 `InferFuncCoreType.cpp:109-115`。
- 在管线中的位置：添加于后续需要核心类型信息的优化之前，见 `HIVMPipelines.cpp:152-154`。

**简单示例**

- 输入 MLIR（不带任何核心类型标注）：

```mlir
module {
  func.func @CUBE() {
    %mc = memref.alloc() : memref<256x256xf32>
    %start = arith.constant 0 : index
    %end = arith.constant 1024 : index
    %step = arith.constant 128 : index
    %c256 = arith.constant 256 : index
    %c128 = arith.constant 128 : index
    scf.for %i = %start to %end step %step {
      %ma = memref.alloc() : memref<256x128xf16>
      %mb = memref.alloc() : memref<128x256xf16>
      %init = arith.cmpi eq, %i, %start : index
      hivm.hir.mmadL1 ins(%ma, %mb, %init, %c256, %c128, %c256
        : memref<256x128xf16>, memref<128x256xf16>, i1, index, index, index)
        outs(%mc : memref<256x256xf32>)
    }
    return
  }

  func.func @VEC() {
    %src = memref.alloc() : memref<1x10xi32>
    %dst = memref.alloc() : memref<3x10xi32>
    %tmp = memref.alloc() : memref<16xi32>
    hivm.hir.vbrc ins(%src : memref<1x10xi32>)
                  outs(%dst : memref<3x10xi32>)
                  temp_buffer(%tmp : memref<16xi32>)
                  broadcast_dims = [0]
    return
  }

  func.func @F5() {
    func.call @CUBE() : () -> ()
    func.call @VEC()  : () -> ()
    return
  }
}
```

- 运行后效果：
  - `@CUBE` 被打上 `hivm.func_core_type = #hivm.func_core_type<AIC>`
  - `@VEC` 被打上 `hivm.func_core_type = #hivm.func_core_type<AIV>`
  - `@F5` 调用两者，得到 `hivm.func_core_type = #hivm.func_core_type<MIX>`
  - `module` 汇总得到 `hivm.module_core_type = #hivm.module_core_type<MIX>`

**运行命令**

```bash
bishengir-opt -hivm-infer-func-core-type input.mlir
```

- 项目内已有更完整的用例与校验，参见 `bishengir/test/Dialect/HIVM/hivm-infer-func-core-type.mlir`，可直接运行：

```bash
bishengir-opt -hivm-infer-func-core-type bishengir/test/Dialect/HIVM/hivm-infer-func-core-type.mlir
```

**更多参考**

- 定义：`bishengir/include/bishengir/Dialect/HIVM/Transforms/Passes.td:23`
- 实现：`bishengir/lib/Dialect/HIVM/Transforms/InferFuncCoreType.cpp:55-140`
- 单个操作的核心类型推断：`bishengir/lib/Dialect/HIVM/IR/InferCoreTypeInterface/InferCoreType.cpp`，如 `VBrcOp::inferCoreType` 在 `.../InferCoreType.cpp:306`
- 管线中用法：`bishengir/lib/Dialect/HIVM/Pipelines/HIVMPipelines.cpp:152-154`

## convert-to-hivm-op

**作用**

- 将非 HIVM 方言中的通用操作转换为对应的 HIVM 硬件感知操作，便于后续基于管线、同步与内存布局的优化。
- 在设备侧函数中进行模式重写：识别拷贝语义并生成 `hivm.hir.load`、`hivm.hir.store` 或 `hivm.hir.copy`。
- 跳过宿主侧函数，避免把 Host 逻辑转换为设备端 HIVM 操作。
- 在转换时尽可能补全有用的属性信息（如左侧填充、Pad 模式、隐式转置标记），以便下游优化与代码生成。

**转换规则**

- `memref.copy src -> dst`
  - 若 `src` 溯源自 GM 空间，转为 `hivm.hir.load ins(src) outs(dst)`
  - 若 `dst` 溯源自 GM 空间，转为 `hivm.hir.store ins(src) outs(dst)`
  - 否则，转为纯张量/本地拷贝 `hivm.hir.copy ins(src) outs(dst)`
- `bufferization.materialize_in_destination source in writable dest`
  - 若 `dest` 溯源自 GM 空间，转为 `hivm.hir.store ins(source) outs(dest)`
- 附加属性与内联处理
  - 从 `memref.subview` 偏移提取左侧填充数，写入 `load` 的 `leftPaddingNum`
    - 参考 `bishengir/lib/Dialect/HIVM/Transforms/ConvertToHIVMOp.cpp:66-79`
  - 若同一 `alloc` 的唯一初始化由 `vbrc` 或 `scf.if` 条件控制，`load` 内联初始化并设置 `init_out_buffer` 与 `init_condition`
    - 参考 `.../ConvertToHIVMOp.cpp:101-125, 134-151`
  - 传播 `MayImplicitTransposeWithLastAxis` 标注到 `load/store`
    - 参考 `.../ConvertToHIVMOp.cpp:156-163, 200-205`
- 仅处理设备侧函数
  - 参考 `.../ConvertToHIVMOp.cpp:246-250`
- 入口
  - Pass 实现与注册：`.../ConvertToHIVMOp.cpp:237-260`
  - TableGen 定义：`.../Transforms/Passes.td:29-35`
  - 在转换管线中调用：`.../Pipelines/ConvertToHIVMPipeline.cpp:39-41`

**简单示例**

- 示例 1：一般拷贝被归一化为 HIVM 纯拷贝

```mlir
module {
  func.func @copy_local() attributes {hacc.function_kind = #hacc.function_kind<DEVICE>} {
    %src = memref.alloc() : memref<16xf32>
    %dst = memref.alloc() : memref<16xf32>
    memref.copy %src, %dst : memref<16xf32> to memref<16xf32>
    return
  }
}
```

运行后（关键语义）：

```mlir
module {
  func.func @copy_local() attributes {hacc.function_kind = #hacc.function_kind<DEVICE>} {
    %src = memref.alloc() : memref<16xf32>
    %dst = memref.alloc() : memref<16xf32>
    hivm.hir.copy ins(%src : memref<16xf32>) outs(%dst : memref<16xf32>)
    return
  }
}
```

- 示例 2：GM → 本地的拷贝识别为 Load

```mlir
module {
  func.func @gm_to_local() attributes {hacc.function_kind = #hacc.function_kind<DEVICE>} {
    %src_gm = func.arg @gm_buffer : memref<16xf32, #hivm.address_space<gm>>
    %dst    = memref.alloc() : memref<16xf32>
    memref.copy %src_gm, %dst : memref<16xf32, #hivm.address_space<gm>> to memref<16xf32>
    return
  }
}
```

运行后（关键语义）：

```mlir
module {
  func.func @gm_to_local() attributes {hacc.function_kind = #hacc.function_kind<DEVICE>} {
    %src_gm = func.arg @gm_buffer : memref<16xf32, #hivm.address_space<gm>>
    %dst    = memref.alloc() : memref<16xf32>
    hivm.hir.load ins(%src_gm : memref<16xf32, #hivm.address_space<gm>>) outs(%dst : memref<16xf32>)
    return
  }
}
```

- 示例 3：将 materialize\_in\_destination 转换为 Store

```mlir
module {
  func.func @materialize_to_gm(%t: tensor<16xf32>, %dst_gm: memref<16xf32, #hivm.address_space<gm>>)
    attributes {hacc.function_kind = #hacc.function_kind<DEVICE>} {
    bufferization.materialize_in_destination %t in writable %dst_gm :
      (tensor<16xf32>, memref<16xf32, #hivm.address_space<gm>>) -> ()
    return
  }
}
```

运行后（关键语义）：

```mlir
module {
  func.func @materialize_to_gm(%t: tensor<16xf32>, %dst_gm: memref<16xf32, #hivm.address_space<gm>>)
    attributes {hacc.function_kind = #hacc.function_kind<DEVICE>} {
    hivm.hir.store ins(%t : tensor<16xf32>) outs(%dst_gm : memref<16xf32, #hivm.address_space<gm>>)
    return
  }
}
```

**运行命令**

```bash
# 单独运行此 pass
bishengir-opt -convert-to-hivm-op input.mlir

# 或使用完整转换管线（包含 HFusion/Tensor 到 HIVM 等）
bishengir-opt -convert-to-hivm-pipeline input.mlir
```

**参考代码位置**

- Pass 定义：`bishengir/include/bishengir/Dialect/HIVM/Transforms/Passes.td:29-35`
- Pass 实现与规则：`bishengir/lib/Dialect/HIVM/Transforms/ConvertToHIVMOp.cpp:237-260, 183-211, 213-231, 127-165, 171-181`
- 管线集成：`bishengir/lib/Dialect/HIVM/Pipelines/ConvertToHIVMPipeline.cpp:29-41`

## hivm-normalize-matmul

**作用**

- 规范化 HIVM 的矩阵乘操作，统一表达“初始化/偏置/真实维度”等语义，便于后续优化、布局推断与代码生成。
- 将“矩阵乘 + 加法偏置”的不同写法折叠为一致的 `mmad` 形式：
  - 非零初值的累加改写为“先清零再 `mmad`，随后 `vadd`”；或
  - 通道偏置折叠进 `mmad` 的 `perChannelBias` 输入（含 split‑K 场景）。
- 自动填充 `mmad` 的真实 `M/K/N` 尺寸，来源于 `tensor` 形状或 `memref.subview`/`alloc` 的实际切片。
- 根据提示与数据情况优化填充：若只需补 `K` 维且 `load` 的 `padValue` 为 0，则去掉不必要的 L1 填充，提高效率。
- 采用自顶向下遍历确保更好的融合顺序（如“mad + mad”保留在 L0C 累加，不被过早拆成“mad + add”）。

**关键规则**

- 非零初值累加
  - 原始 `mmad` 对非零初值 `C` 累加，改写为“`C` 保留，`mmad` 写入空 `C`，再对 `C` 和 `mmad` 结果执行 `vadd`”，见 `NormalizeMatmul.cpp:236-270`。
- 每通道偏置折叠
  - 若偏置通过 `vbrc` 扩成与输出同形，再与 `mmad` 输出相加，则直接把偏置折叠为 `mmad` 的 `perChannelBias`，见 `NormalizeMatmul.cpp:289-325`。
  - split‑K 加法的特殊形态也能识别并折叠，见 `NormalizeMatmul.cpp:357-386`。
- 真实 M/K/N 尺寸
  - 从 `tensor` 或 `memref.subview/alloc` 推导真实 `M/K/N`，考虑 `A/B` 是否转置，写入 `mmad` 的 `real_m/k/n` 可变字段，见 `NormalizeMatmul.cpp:130-178, 180-211, 213-234`。
- 填充优化
  - 若源值带 `dot_pad_only_k` 提示且 `load` 的 `padValue` 为 0，则移除多余的 L1 填充与初始化相关字段，见 `NormalizeMatmul.cpp:98-122`。
- 实施位置
  - Pass 类与入口：`NormalizeMatmul.cpp:124-129, 423-451`
  - 定义：`Passes.td:37-43`

**简单示例**

- 示例 1：非零初值累加的规范化（元素加）

```mlir
// 输入（mmad 直接对非零初值 %C 累加）
%C    = tensor.empty() : tensor<16x32xf32>
%mad  = hivm.hir.mmadL1 ins(%A, %B : tensor<16xKxf32>, tensor<Kx32xf32>)
                     outs(%C : tensor<16x32xf32>) -> tensor<16x32xf32>

// 规范化后（清零 + mmad，再对原 %C 与结果做 vadd）
%zero = tensor.empty() : tensor<16x32xf32>
%mmad = hivm.hir.mmadL1 ins(%A, %B) outs(%zero) -> tensor<16x32xf32>
%tmp  = tensor.empty() : tensor<16x32xf32>
%add  = hivm.hir.vadd ins(%C, %mmad) outs(%tmp) -> tensor<16x32xf32>
```

- 示例 2：每通道偏置折叠到 `mmad`

```mlir
// 输入（偏置先 vbrc，再与输出相加）
%bias_1xN = tensor.empty() : tensor<1x32xf32>
%dst      = tensor.empty() : tensor<16x32xf32>
%brc      = hivm.hir.vbrc ins(%bias_1xN) outs(%dst) broadcast_dims = [0]
%C        = tensor.empty() : tensor<16x32xf32>
%mad      = hivm.hir.mmadL1 ins(%A, %B) outs(%C) -> tensor<16x32xf32>

// 规范化后（偏置作为 perChannelBias 直接进入 mmad）
%C        = tensor.empty() : tensor<16x32xf32>
%mad_norm = hivm.hir.mmadL1 ins(%A, %B, bias = %bias_1xN)
                     outs(%C) -> tensor<16x32xf32>
```

- 示例 3：自动填充真实 `M/K/N`
  - 若 `A`/`B` 来自 `memref.subview` 或 `tensor` 的动态形状，Pass 会读取真实尺寸，结合 `A/B` 是否转置，写入 `mmad` 的 `real_m/k/n` 字段，供后续数据布局与块大小选择使用（语义参考 `NormalizeMatmul.cpp:130-178, 180-211, 213-234`）。

**运行命令**

```bash
# 仅运行规范化 pass
bishengir-opt -hivm-normalize-matmul input.mlir
```

**参考代码位置**

- 定义：`bishengir/include/bishengir/Dialect/HIVM/Transforms/Passes.td:37-43`
- 实现（核心逻辑与模式）：
  - `非零初值累加`：`bishengir/lib/Dialect/HIVM/Transforms/NormalizeMatmul.cpp:236-270`
  - `每通道偏置折叠`：`.../NormalizeMatmul.cpp:289-325`
  - `split‑K 偏置折叠`：`.../NormalizeMatmul.cpp:357-386`
  - `真实 M/K/N 提取与设置`：`.../NormalizeMatmul.cpp:130-178, 180-211, 213-234`
  - `填充优化`：`.../NormalizeMatmul.cpp:98-122`
  - `Pass 类与运行`：`.../NormalizeMatmul.cpp:124-129, 423-451`

## triton-global-kernel-args-to-hivm-op

**作用**

- 将 Triton 全局 kernel 的网格参数规范化为 HIVM 的块索引获取方式，并整理函数签名与属性：
  - 用 `hivm.get_block_idx` 替换 Triton 的 3D `program_id` 参数表达，计算并回填每轴的 `program_id_{0,1,2}`。
  - 标注逻辑块数乘积以便后续优化（在临时乘积值上打 `annotation` 标记）。
  - 删除已被替换的 `program_id` 形式参数，使函数签名更简洁。
  - 根据形参类型，生成 `func_dyn_memref_args` 属性标记哪些 `memref` 是动态形状，便于后续描述符适配。

**处理逻辑**

- 验证最后 6 个 `i32` 形参是否存在：3 个 `program_id` 和 3 个 `program_num`，见 `TritonGlobalKernelArgsToHIVMOp.cpp:79-95`。
- 使用 `hivm.get_block_idx : i64` 获取线性块索引，截断为 `i32` 后按 3D 网格 `[x, y, z]` 计算每轴 `program_id`：
  - `pid2 = idx // 1 mod z`
  - `pid1 = idx // z mod y`
  - `pid0 = idx // (y*z) mod x`
  - 参考 `.../TritonGlobalKernelArgsToHIVMOp.cpp:97-131`
- 将上述计算结果替换原有的 3 个 `program_id` 参数的使用，并删除它们，见 `...:134-141`。
- 统计形参中动态 `memref`，设置 `func_dyn_memref_args` 属性为布尔向量，见 `...:143-156`。
- 适用范围：设备侧入口函数（`hacc::utils::isDeviceEntry`），见 `...:163-165`。
- 定义与注册：
  - Pass 定义：`bishengir/include/bishengir/Dialect/HIVM/Transforms/Passes.td:46-50`
  - 实现：`bishengir/lib/Dialect/HIVM/Transforms/TritonGlobalKernelArgsToHIVMOp.cpp`

**简单示例**

- 输入（Triton 风格，末尾 6 个 `i32` 形参为 `[pid0, pid1, pid2, x, y, z]`）：

```mlir
func.func @kernel(%a: memref<?xf32>, %b: memref<?xf32>, %pid0: i32, %pid1: i32, %pid2: i32,
                  %x: i32, %y: i32, %z: i32)
    attributes {hacc.function_kind = #hacc.function_kind<DEVICE>, hacc.entry} {
  // 使用 %pid0/%pid1/%pid2 进行索引计算...
  return
}
```

- 运行后效果（核心变化）：
  - 在入口块创建 `hivm.get_block_idx : i64`，并按上式计算 3 个维度的 `program_id` 值替换所有使用；
  - 删除 3 个 `program_id` 形参，保留 `[x, y, z]`；
  - 写入 `func_dyn_memref_args = dense<[true, true, ...]> : vector<...xi1>`，标记动态 `memref`；
  - 在内部插入对逻辑块数乘积的标注：

```mlir
%block_idx = hivm.hir.get_block_idx -> i64
%block_i32 = arith.trunci %block_idx : i64 to i32
%mul = arith.muli %x, %y
%logical_blocks = arith.muli %mul, %z
annotation.mark %logical_blocks {annotation.logical_block_num} : i32
// 计算 pid2/pid1/pid0 的表达式并替换原使用点
```

**使用命令**

```bash
bishengir-opt -triton-global-kernel-args-to-hivm-op input.mlir
# 或在转换到 HIVM 的管线中启用
bishengir-opt -convert-to-hivm-pipeline='enable-triton-kernel-compile=true' input.mlir
```

**参考位置**

- 定义：`bishengir/include/bishengir/Dialect/HIVM/Transforms/Passes.td:46-50`
- 实现：`bishengir/lib/Dialect/HIVM/Transforms/TritonGlobalKernelArgsToHIVMOp.cpp:57-173`
- 管线集成（可选）：`bishengir/lib/Dialect/HIVM/Pipelines/ConvertToHIVMPipeline.cpp:31-41`

## hivm-infer-mem-scope

**作用**

- 推断并传播 HIVM 程序中的 `memref` 地址空间（内存作用域），为后续数据布局、同步与代码生成提供准确的内存层级信息。
- 设备侧函数：
  - 为 `mmadL1` 相关缓冲区赋予正确作用域：`A/B` → `L1`，`C` → `L0C`，`bias`（若存在）→ `L1`。
  - 将设备函数的形参统一标记为 `GM`，并更新函数类型签名。
  - 根据指针转换注解传播作用域，确保 `PointerCast` 体现的 `GM` 语义正确传导。
  - 根据函数核心类型选择默认 `alloc` 作用域：
    - `AIC`（Cube）函数：剩余 `memref.alloc` 默认设为 `L1`
    - `AIV`（Vector）函数：剩余 `memref.alloc` 默认设为 `UB`
- 调用点修复：
  - 按被调函数签名将 `func.call` 的实参与返回值的 `memref` 作用域同步、修正。
- 宿主侧函数：
  - 用已传播的块参数与返回值类型更新函数签名。

**关键路径**

- 推断与传播机制通过更新 `BaseMemRefType` 的 `memorySpace` 并级联更新使用者：
  - `scf.yield` / `scf.for` / `scf.while` 的迭代参数与结果
  - 单结果的视图/转置/位解释等可传播操作
  - 对 `func.call`，不直接跨函数体内传播，但在调用点按签名修复类型
- 入口与执行：
  - Pass 定义：`bishengir/include/bishengir/Dialect/HIVM/Transforms/Passes.td:52-56`
  - 实现主流程：`bishengir/lib/Dialect/HIVM/Transforms/InferHIVMMemScope.cpp:422-478`
  - `mmadL1` 专属规则：`.../InferHIVMMemScope.cpp:177-252`
  - 设备函数参数/返回传播：`.../InferHIVMMemScope.cpp:358-387`
  - `PointerCast` GM 传播：`.../InferHIVMMemScope.cpp:390-405`
  - `alloc` 默认作用域按核心类型：`.../InferHIVMMemScope.cpp:454-466`

**简单示例**

- 示例 1：`mmadL1` 的 A/B/C 作用域设置与传播

```mlir
func.func @cube_kernel(%A: memref<256x128xf16>, %B: memref<128x256xf16>, %C: memref<256x256xf32>)
  attributes {hacc.function_kind = #hacc.function_kind<DEVICE>} {
  // 初始未带作用域
  hivm.hir.mmadL1 ins(%A, %B : memref<256x128xf16>, memref<128x256xf16>)
                  outs(%C : memref<256x256xf32>)
  return
}
```

运行 `-hivm-infer-mem-scope` 后，类型被补充并向使用者传播：

- `%A` → `memref<..., #hivm.address_space<cbuf>>`（L1）
- `%B` → `memref<..., #hivm.address_space<cbuf>>`（L1）
- `%C` → `memref<..., #hivm.address_space<cc>>`（L0C）
- 若随后有 `memref.subview`、`memref.transpose`、`hivm.bitcast` 等使用，它们的结果类型也会同步携带该作用域。
- 示例 2：设备函数形参统一设为 GM，并修复调用点

```mlir
module {
  func.func @device(%arg0: memref<16xf32>, %arg1: memref<16xf32>)
    attributes {hacc.function_kind = #hacc.function_kind<DEVICE>} {
    // ... 操作 ...
    return
  }

  func.func @main() attributes {hacc.function_kind = #hacc.function_kind<HOST>} {
    %a = memref.alloc() : memref<16xf32>
    %b = memref.alloc() : memref<16xf32>
    // 调用前，形参尚未带 GM
    func.call @device(%a, %b) : (memref<16xf32>, memref<16xf32>) -> ()
    return
  }
}
```

运行后：

- `@device` 的函数类型更新为 `memref<16xf32, #hivm.address_space<gm>>` 形参；
- `@main` 里的 `func.call` 实参沿着调用点传播，参数与可能的返回值携带 GM 作用域；
- 宿主侧 `@main` 的函数签名按块参数与返回值的已传播类型进行更新。

**运行命令**

```bash
bishengir-opt -hivm-infer-mem-scope input.mlir
```

**提示**

- 建议在 `NormalizeMatmul`、拷贝/加载等语义明确后执行本 Pass，以获得更准确的 `mmad` 相关作用域。
- 与函数核心类型推断协作：已推断为 `AIC` 的函数默认 `alloc` 倾向 `L1`，`AIV` 倾向 `UB`。

## hivm-mark-multi-buffer

**作用**

- 在设备侧函数内，自动为满足条件的缓冲区标记多缓冲属性 `hivm.multi_buffer`，以启用后续的多缓冲优化（流水并行、隐藏访存等）。
- 支持两类场景：
  - 本地缓冲（如 `alloc` 链接到 `load/store/fixpipe/ND2NZ`）处于循环中时自动标记；
  - 工作区缓冲（`memref_ext.alloc_workspace` 或其视图/张量化后）在循环中被写入时，根据选项标记指定数量的多缓冲。
- 避免宿主侧函数；混核 `MIX` 函数可根据策略限制仅标记 Cube 或 Vector 一侧的缓冲。

**标记规则**

- 仅在纯缓冲状态下进行（操作具有纯 buffer 语义），并且源/目的操作数已带地址空间信息。
- 向后溯源到 `alloc` 或 `alloc_workspace`，要求该缓冲在 `scf.for` 循环层次内；其他循环类型当前不支持。
- 若缓冲已存在 `annotation.mark` 且含 `hivm.multi_buffer`，不会重复标记。
- 默认标记数为 2，可通过选项调整工作区标记数。

**选项与策略**

- `enable-auto`：开启自动标记（默认 false）
- 限制本地与工作区：
  - `limit-auto-multi-buffer-only-for-local-buffer`：为 true 时只标记本地缓冲，不标记工作区
- 混核函数限制：
  - `limit-auto-multi-buffer-of-local-buffer`：对本地缓冲的策略，如 `CUBE_NO_L0C` 不在 L0C 标记
  - `limit-mix-auto-multi-buffer-buffer`：在 `MIX` 函数内只标记 Cube 或 Vector 一侧（`ONLY_CUBE`/`ONLY_VECTOR`/`NO_LIMIT`）
- `set-workspace-multibuffer`：工作区多缓冲数，默认 2

定义位置与实现：

- 定义：`bishengir/include/bishengir/Dialect/HIVM/Transforms/Passes.td:58-93`
- 实现：`bishengir/lib/Dialect/HIVM/Transforms/MarkMultiBuffer.cpp`

**简单示例**

- 示例 1：在循环中对本地缓冲进行加载与存储，自动标记为多缓冲

```mlir
func.func @kernel(%gm: memref<1024xf32, #hivm.address_space<gm>>)
  attributes {hacc.function_kind = #hacc.function_kind<DEVICE>} {
  %local = memref.alloc() : memref<256xf32>          // 未带作用域，前序 pass 可能已推断为 UB/L1
  %out   = memref.alloc() : memref<256xf32>
  %c0 = arith.constant 0 : index
  %c256 = arith.constant 256 : index
  %c64 = arith.constant 64 : index
  scf.for %i = %c0 to %c256 step %c64 {
    hivm.hir.load  ins(%gm : memref<1024xf32, #hivm.address_space<gm>>) outs(%local : memref<256xf32>)
    // ... 计算 ...
    hivm.hir.store ins(%local : memref<256xf32>) outs(%out : memref<256xf32>)
  }
  return
}
```

运行 `-hivm-mark-multi-buffer='enable-auto'` 后：

- 追溯到 `%local`/`%out` 的 `alloc`，若位于 `scf.for` 层次下且满足策略，就在其后插入 `annotation.mark`，带 `hivm.multi_buffer = 2`。
- 示例 2：工作区缓冲在循环中被写入，按选项标记为多缓冲

```mlir
func.func @kernel(%gm: memref<1024xf32, #hivm.address_space<gm>>)
  attributes {hacc.function_kind = #hacc.function_kind<DEVICE>} {
  %ws = memref_ext.alloc_workspace() : memref<256xf32>
  %tmp = tensor.empty() : tensor<256xf32>
  scf.for %i = %c0 to %c256 step %c64 {
    // 写入工作区（通过 store 或 fixpipe 等）
    hivm.hir.store ins(%tmp : tensor<256xf32>) outs(%ws : memref<256xf32>)
  }
  return
}
```

运行 `-hivm-mark-multi-buffer='enable-auto limit-auto-multi-buffer-only-for-local-buffer=false set-workspace-multibuffer=4'` 后：

- 追溯 `%ws` 到 `alloc_workspace` 且确认位于循环中，插入 `annotation.mark`，`hivm.multi_buffer = 4`。

**运行命令**

```bash
# 默认本地缓冲标记
bishengir-opt -hivm-mark-multi-buffer='enable-auto' input.mlir

# 在混核函数中仅标注 Vector 一侧，并对工作区设置多缓冲为 4
bishengir-opt -hivm-mark-multi-buffer='enable-auto limit-mix-auto-multi-buffer-buffer=only-vector set-workspace-multibuffer=4' input.mlir
```

**提示**

- 建议在内存作用域已推断（`hivm-infer-mem-scope`）且循环结构稳定后运行本 Pass，避免误标或漏标。
- 标记仅添加 `annotation.mark` 属性，不执行实际的缓冲重写；实际开启多缓冲需要后续 `EnableMultiBuffer` Pass 生效。

## hivm-enable-multi-buffer

**作用**

- 将已通过标注或前序 pass 识别的多缓冲机会在 IR 中真正展开为可选择的多个缓冲版本，从而打破迭代间依赖，支持流水并行。
- 主要面向 `hivm.hir.pointer_cast` 形成的临时缓冲；在循环内依据迭代计数选择不同缓冲，实现如“双缓冲”切换。
- 要求在 `scf.for` 循环环境下；宿主侧函数被跳过。

**核心改写**

- 针对循环中的 `hivm.pointer_cast`，当其 `addrs` 数量大于 1 且不为 GM 指针：
  - 在函数入口创建多个等价的 `pointer_cast` 版本，各自绑定不同 `addr`；
  - 将原 `annotation.mark` 的属性复制到新建的各版本；
  - 构造选择索引：通过 `affine.apply` 计算 `(iv floordiv step) mod factor`，再用 `arith.select` 在每次迭代选择对应缓冲；
  - 替换原使用并删除旧的 `pointer_cast`。

参考实现位置：

- 定义：`bishengir/include/bishengir/Dialect/HIVM/Transforms/Passes.td:95-103`
- 实现：`bishengir/lib/Dialect/HIVM/Transforms/EnableMultiBuffer.cpp`

**简单示例**

- 输入（循环内的指针转换，带多缓冲标记，或有多个 `addrs`）：

```mlir
func.func @kernel()
  attributes {hacc.function_kind = #hacc.function_kind<DEVICE>} {
  %addr0 = arith.constant 0 : i64
  %addr1 = arith.constant 4096 : i64
  %tmp   = memref.alloc() : memref<4x128xf32>
  %c0 = arith.constant 0 : index
  %c16 = arith.constant 16 : index
  %c4 = arith.constant 4 : index
  scf.for %iv = %c0 to %c16 step %c4 {
    %pc = hivm.hir.pointer_cast(%addr0, %addr1) [] : memref<4x128xf32>
    annotation.mark %pc {hivm.multi_buffer = 2 : i32}
    "use"(%pc) : (memref<4x128xf32>) -> ()
  }
  return
}
```

- 运行 `-hivm-enable-multi-buffer` 后的关键变化：
  - 在函数入口创建两个 `pointer_cast` 版本：`%pc0` 绑定 `addr0`，`%pc1` 绑定 `addr1`；
  - 复制原标注到新版本；
  - 构建选择逻辑：
    - `#map = affine_map<()[s0] -> ((s0 floordiv 4) mod 2)>`
    - `%sel = affine.apply #map()[%iv]`
    - 将 `%sel` 转为条件并用 `arith.select` 在 `%pc0/%pc1` 中选取当前迭代的缓冲；
  - 用选择结果替换原 `%pc` 的所有使用，删除旧 `%pc`。

**配合使用**

- 建议与 `MarkMultiBuffer` 配套：由前者自动标注可多缓冲的资源，后者再将其展开为实际多缓冲结构。
- 管线中通常先运行 `hivm-mark-multi-buffer`，随后 `hivm-enable-multi-buffer`，再进行内存规划和下游优化。

**运行命令**

```bash
bishengir-opt -hivm-enable-multi-buffer input.mlir
```

**说明**

- 当前实现强调双缓冲和 `scf.for`，其他循环类型或多于 2 的缓冲因子尚不覆盖。
- 在生成的 IR 中，`pointer_cast` 的地址来源需要为常量，以便在函数入口安全地创建静态版本。

## hivm-add-ffts-to-syncblocksetop

**作用**

- 为设备侧函数中的 `hivm.hir.sync_block_set` 操作自动填充 FFTS 基地址参数，使同步块集合在运行时具备正确的基址配置。
- 从包围的设备函数的形参中检索 FFTS 基地址（被标注为 `kFFTSBaseAddr` 的 kernel 参数），并赋值到 `SyncBlockSetOp` 的 `ffts_base_addr` 字段。
- 跳过宿主侧函数。

**处理逻辑**

- 查找 `SyncBlockSetOp` 所在的包围 `func.func`。
- 在该函数的参数列表中检查是否存在被标记为 `hacc::KernelArgType::kFFTSBaseAddr` 的参数；若存在，将其作为 `ffts_base_addr` 写入 `SyncBlockSetOp`。
- 若 `SyncBlockSetOp` 已包含 `ffts_base_addr`，不做修改；若找不到对应函数或参数，报错。
- 入口与实现：
  - Pass 定义：`bishengir/include/bishengir/Dialect/HIVM/Transforms/Passes.td:105-113`
  - 实现：`bishengir/lib/Dialect/HIVM/Transforms/AddFFTSToSyncBlockSetOp.cpp:63-104`

**简单示例**

- 输入（设备侧函数未显式为 `sync_block_set` 提供基地址）：

```mlir
func.func @kernel(%ffts_base: i64 {hfusion.ffts_base_address}, %arg0: memref<1xi64>)
  attributes {hacc.function_kind = #hacc.function_kind<DEVICE>} {
  // 未设置 ffts_base_addr 的 sync 操作
  hivm.hir.sync_block_set lock_var(%arg0 : memref<1xi64>)
  return
}
```

- 运行 `-hivm-add-ffts-to-syncblocksetop` 后：

```mlir
func.func @kernel(%ffts_base: i64 {hfusion.ffts_base_address}, %arg0: memref<1xi64>)
  attributes {hacc.function_kind = #hacc.function_kind<DEVICE>} {
  // 自动填充 ffts_base_addr
  hivm.hir.sync_block_set lock_var(%arg0 : memref<1xi64>) ffts_base_addr = %ffts_base : i64
  return
}
```

**运行命令**

```bash
bishengir-opt -hivm-add-ffts-to-syncblocksetop input.mlir
```

**提示**

- 需要函数形参中存在 FFTS 基地址（通常由上游管线或编译前端按约定注入并标注）。若缺失，该 Pass 会报错提示。

## hivm-memref-alloc-to-alloca

**作用**

- 将位于本地内存空间的 `memref.alloc` 转换为 `memref.alloca`，从堆分配改为栈分配，以减少运行时分配开销并匹配期望的生命周期。
- 仅转换带有 HIVM 地址空间且不为 `GM` 的分配；`GM`（全局内存）上的 `alloc` 保持不变。
- 依赖前序内存作用域推断，使本地缓冲（如 `UB`/`L1`）已携带地址空间属性。

实现位置：

- 定义：`bishengir/include/bishengir/Dialect/HIVM/Transforms/Passes.td:115-121`
- 逻辑：`bishengir/lib/Dialect/HIVM/Transforms/AllocToAlloca.cpp:33-55, 64-79`

**转换规则**

- 检查 `memref.alloc` 的类型：
  - 必须是 `BaseMemRefType` 且 `memorySpace` 为 `hivm.address_space`；
  - 若为 `GM`，不转换；
  - 若为非 `GM`（如 `UB`、`L1`、`L0A/B/C`），替换为等价的 `memref.alloca`，保留动态尺寸、符号操作数与对齐属性。

**简单示例**

- 输入（已推断为 UB/L1 的本地缓冲）：

```mlir
func.func @kernel()
  attributes {hacc.function_kind = #hacc.function_kind<DEVICE>} {
  %local = memref.alloc() : memref<256xf32, #hivm.address_space<ub>>
  %tile  = memref.alloc() : memref<128x128xf16, #hivm.address_space<cbuf>>
  // ... 使用 ...
  return
}
```

- 运行 `-hivm-memref-alloc-to-alloca` 后：

```mlir
func.func @kernel()
  attributes {hacc.function_kind = #hacc.function_kind<DEVICE>} {
  %local = memref.alloca() : memref<256xf32, #hivm.address_space<ub>>
  %tile  = memref.alloca() : memref<128x128xf16, #hivm.address_space<cbuf>>
  // ... 使用 ...
  return
}
```

**运行命令**

```bash
bishengir-opt -hivm-memref-alloc-to-alloca input.mlir
```

**提示**

- 建议在 `hivm-infer-mem-scope` 之后运行，使得 `alloc` 已正确带上地址空间属性，从而精准选择可转换的本地分配。
- `alloca` 的栈分配需注意栈大小与生命周期，过大的缓冲或跨越复杂控制流可能不适合栈分配。

## hivm-clone-tensor-empty

**作用**

- 在设备侧函数中，将 `hivm` 结构化操作和 `scf.for` 的初值中直接使用的 `tensor.empty` 克隆为就地创建的独立实例，避免多个使用点共享同一个空张量，从而防止隐式别名和不期望的依赖。
- 针对 `hivm` 结构化操作的 `DPS init`（输出目标）和 `scf.for` 的 `init_args`，如果是 `tensor.empty`，在操作附近克隆一份并替换原引用。

实现位置：

- 定义：`bishengir/include/bishengir/Dialect/HIVM/Transforms/Passes.td:123-128`
- 逻辑：`bishengir/lib/Dialect/HIVM/Transforms/CloneTensorEmpty.cpp`

**处理规则**

- 对所有实现了 `HIVMStructuredOpInterface` 的操作，以及常见的 HIVM 栈操作（`copy/load/store/fixpipe/mmadL1`）：
  - 遍历其 `getDpsInits()`；若某个 init 的定义是 `tensor.empty`，在当前操作处克隆该 `empty`，并将操作使用从原 `empty` 改为克隆结果。
- 对 `scf.for`：
  - 检查 `init_args` 中的 `tensor.empty`，在 `scf.for` 处克隆并替换对应的 `init_args`。

**简单示例**

- 输入（`mmadL1` 直接使用共享的 `tensor.empty` 作为输出 init）：

```mlir
func.func @kernel()
  attributes {hacc.function_kind = #hacc.function_kind<DEVICE>} {
  %init = tensor.empty() : tensor<16x32xf32>
  %res  = hivm.hir.mmadL1 ins(%A, %B : tensor<16xKxf32>, tensor<Kx32xf32>)
                    outs(%init : tensor<16x32xf32>) -> tensor<16x32xf32>
  // 其他操作也可能使用 %init
  return
}
```

- 运行 `-hivm-clone-tensor-empty` 后（关键语义）：
  - 在 `mmadL1` 的位置克隆一份 `tensor.empty`，将其作为 `outs` 的目标；
  - 原 `%init` 可继续被其他操作使用，避免与 `mmadL1` 的写入共享。
- 示例（`scf.for` 的 `init_args` 含 `tensor.empty`）：

```mlir
%init = tensor.empty() : tensor<16x32xf32>
%for = scf.for %i = %c0 to %cN step %c1 iter_args(%acc = %init) -> (tensor<16x32xf32>) {
  // ...
  scf.yield %acc : tensor<16x32xf32>
}
```

- 运行后：
  - 在 `scf.for` 处克隆 `%init`，用克隆结果替换 `iter_args` 中的空张量，确保每个循环构造点的初值与外部不共享。

**运行命令**

```bash
bishengir-opt -hivm-clone-tensor-empty input.mlir
```

**提示**

- 此 Pass 只在设备侧函数生效（宿主侧函数跳过）。
- 它不改变语义，仅避免共享 `tensor.empty` 引发的后续优化或缓冲化问题，常与后续的布局推断、规范化等 Pass 协同工作。

## hivm-infer-data-layout

**作用**

- 推断并统一 HIVM 程序中各个 `memref` 值的“数据布局”属性（如 `ND`、`zN`、`nZ`、`DOTA_ND/DOTB_ND/DOTC_ND`），并在必要时插入或折叠布局转换，使生产者与使用者的布局一致。
- 针对矩阵乘链路和加载路径，自动生成或批处理生成 `ND2NZ` 等布局转换；必要时插入 `hivm.convert_layout`，或与 HIVM 操作进行融合以减少冗余转换。
- 只在设备侧函数上执行。

**关键规则**

- 支持的转换对：
  - `DOT{A|B|C}_ND -> zN/nZ`
  - `ND -> zN/nZ` 与其逆向回退（`zN/nZ -> ND`）
  - 转换根据目标布局的分块大小 `fractalSizes` 以及是否需要转置来计算目标形状与偏移
- `ND2NZ` 注入：
  - 直接 `ND -> nZ/zN` 的转换通过创建 `hivm.nd2nz` 完成
  - 若第一维为批维，支持批处理的 `ND2NZ`，遍历批次创建子视图并逐批转换
- 转换折叠与最小化：
  - 对实现了 `OpWithLayoutInterface` 的操作，优先通过操作内部的布局规则协调输入输出布局
  - 若仍不一致，插入 `hivm.convert_layout` 进行转换，并记录映射，尽量避免重复插入
- 针对 `memref.collapse_shape` 的约束：
  - 仅允许折叠“批维为 1”的情况，否则报错；随后继续做 `ND2NZ` 转换或其他布局处理

实现位置：

- 定义：`bishengir/include/bishengir/Dialect/HIVM/Transforms/Passes.td:130-134`
- 主逻辑：`bishengir/lib/Dialect/HIVM/Transforms/InferHIVMDataLayout.cpp`
  - 支持转换表：`66-87`
  - ND2NZ 构造与批处理：`95-133, 134-147, 965-970`
  - 形状/偏移计算：`216-256, 298-334, 336-372, 374-432`
  - 插入 `convert_layout`：`433-452, 194-205, 198-205, 1001-1014`
  - 过约束与报错：`913-915`

**简单示例**

- 示例 1：为从 GM 加载的 `ND` 数据转换到 `nZ` 布局

```mlir
func.func @kernel(%src: memref<256x256xf32, #hivm.address_space<gm>>)
  attributes {hacc.function_kind = #hacc.function_kind<DEVICE>} {
  %dst = memref.alloc() : memref<256x256xf32, #hivm.address_space<ub>>
  // 需要将 ND -> nZ，以进入后续矩阵乘或向量路径
  hivm.hir.load ins(%src : memref<256x256xf32, #hivm.address_space<gm>>) outs(%dst : memref<256x256xf32, #hivm.address_space<ub>>)
  // 后续使用 %dst 进行计算
  return
}
```

运行 `-hivm-infer-data-layout` 后（关键语义）：

- 在 `load` 后插入 `hivm.nd2nz`，根据目标 `nZ` 的分块大小计算目标形状/偏移；
- 若某处需要 `zN`，则根据使用者接口插入 `hivm.convert_layout` 或直接折叠到相关 HIVM 操作。
- 示例 2：批维为 1 的 `memref.collapse_shape` 后的布局转换

```mlir
%collapsed = memref.collapse_shape %arg [[0, 1]] : memref<1x65536xf32> into memref<65536xf32>
// 继续将其 ND -> zN
```

运行后：

- 验证批维为 1，否则报错；
- 对 `collapsed` 插入 `hivm.nd2nz` 到 `zN`，并按 `fractalSizes` 计算形状与偏移。

**运行命令**

```bash
bishengir-opt -hivm-infer-data-layout input.mlir
```

**提示**

- 最佳效果通常在 `NormalizeMatmul`、`ConvertToHIVMOp`、`InferHIVMMemScope` 等 pass 之后运行，使内存作用域、偏置折叠已稳定；该 pass 可据此更精确地选择 `nZ/zN` 等目标布局并插入最少的转换。

## hivm-plan-memory

**作用**

- 为设备侧函数执行“内存地址规划”，根据缓冲的生命周期、作用域和可重用性，计算每个本地缓冲或工作区缓冲的偏移地址，并把地址落入 IR：
  - 本地内存规划：为 `UB/L1/L0C` 等作用域中的 `memref.alloc` 计算偏移并转换为 `hivm.hir.pointer_cast`，支持多缓冲展开与复用。
  - 工作区规划：为 `memref_ext.alloc_workspace` 写入偏移索引，按函数工作区参数进行聚合与复用。
- 综合分析控制流（`scf.for/while/if`）、缓冲别名、多缓冲标记和操作类型，生成尽可能复用的内存布局；空间不足时提供降级策略与失败信息。

实现位置：

- 定义：`bishengir/include/bishengir/Dialect/HIVM/Transforms/Passes.td:136-143`
- 主逻辑：`bishengir/lib/Dialect/HIVM/Transforms/PlanMemory.cpp`
  - 生命周期与别名分析：`135-223, 251-418`
  - 迭代器与控制流处理：`270-356, 380-398, 400-418, 321-334`
  - 多缓冲信息收集：`440-458`
  - 原位复用判定：`824-858, 860-900`
  - 规划算法（本地/工作区）：`916-931, 939-1007, 1039-1057, 1074-1109, 1120-1136, 1162-1306`
  - 失败与回滚策略：`1909-2024, 1926-1971`
  - 将偏移写回 IR：`2049-2066, 2153-2161` 调用 `AllocToPointerCast.cpp:35-55` 和 `58-82`

**处理流程**

- 构建线性操作序列与缓冲生命周期，追踪别名与多缓冲数；识别可原位复用的操作与额外缓冲。
- 按作用域聚合 `StorageEntry`，计算所需容量，若足够则顺序布局；若不足，采用分级规划与回滚、拆分轮廓、考虑多缓冲展开。
- 生成每个缓冲的偏移列表并写回：
  - 本地内存：把 `memref.alloc` 替换为 `hivm.pointer_cast(addrs=[const offsets])`；
  - 工作区：把 `alloc_workspace` 的 `offset` 字段填入常量索引数组。
- 修正循环中的多缓冲 `pointer_cast` 位置，去重类似的 cast 实例。

**简单示例（本地内存）**

- 输入：

```mlir
func.func @kernel()
  attributes {hacc.function_kind = #hacc.function_kind<DEVICE>} {
  %A = memref.alloc() : memref<1024xf16, #hivm.address_space<ub>>
  %B = memref.alloc() : memref<1024xf16, #hivm.address_space<ub>>
  hivm.hir.vadd ins(%A, %B : memref<1024xf16, #hivm.address_space<ub>>,
                             memref<1024xf16, #hivm.address_space<ub>>)
                 outs(%A : memref<1024xf16, #hivm.address_space<ub>>)
  return
}
```

- 运行 `-hivm-plan-memory` 后（关键语义）：
  - 构建生命周期并判断 `vadd` 可原位复用；
  - 计算偏移，例如 `%A` 偏移 `0`、`%B` 偏移 `1024`（示例值）；
  - 将 `memref.alloc` 替换为：
    - `%A = hivm.hir.pointer_cast(0) [] : memref<1024xf16, #hivm.address_space<ub>>`
    - `%B = hivm.hir.pointer_cast(1024) [] : memref<1024xf16, #hivm.address_space<ub>>`

**简单示例（工作区）**

- 输入：

```mlir
func.func @kernel(%ws: memref<?xi8, #hivm.address_space<gm>>)
  attributes {hacc.function_kind = #hacc.function_kind<DEVICE>} {
  %buf0 = memref_ext.alloc_workspace %ws, %dyn0 : memref<4096xi8, #hivm.address_space<gm>>
  %buf1 = memref_ext.alloc_workspace %ws, %dyn1 : memref<2048xi8, #hivm.address_space<gm>>
  // 使用 %buf0, %buf1
  return
}
```

- 运行后：
  - 为 `%buf0/%buf1` 规划不重叠或可复用的偏移；
  - 把偏移写回 `alloc_workspace` 的 `offset` 列表为常量索引，如 `offset=[0]`, `offset=[4096]`。

**运行命令**

```bash
bishengir-opt -hivm-plan-memory input.mlir
```

**提示**

- 建议在 `hivm-mark-multi-buffer`、`hivm-enable-multi-buffer`、`hivm-infer-mem-scope` 和布局相关 pass 之后运行，使地址空间、布局和多缓冲信息稳定。
- 若出现“overflow”错误，表示某作用域需求超过容量；可以减少块大小、关闭或限制多缓冲、开启更多复用或拆分管线以缓解。

## hivm-inject-sync

**作用**

- 在设备侧函数内，根据内存依赖与执行管线自动插入同步原语，确保数据生产与消费的顺序正确，避免不同管线或核间的数据竞争。
- 支持两种模式：
  - `NORMAL`：进行依赖分析、事件分配与去冗余优化，仅插入必要的同步；
  - `BARRIERALL`：在每个 HIVM 指令（以及 `func.return`）前插入 `hivm.pipe_barrier`，用于调试或强制串行。

实现位置：

- 定义：`bishengir/include/bishengir/Dialect/HIVM/Transforms/Passes.td:160-168`
- 逻辑：`bishengir/lib/Dialect/HIVM/Transforms/InjectSync/InjectSync.cpp`
  - 全量屏障注入：`47-60`
  - 正常模式分析与注入：`62-107`
  - Pass 入口：`109-126`

**处理流程（NORMAL）**

- 翻译 IR 到同步分析 IR，收集缓冲与管线的依赖关系；
- 进行同步规划，生成候选同步操作；
- 进行状态移动和去冗余清理；
- 分配事件 ID；
- 生成代码：在需要的位置插入 `hivm.pipe_barrier` 等同步指令。

**简单示例**

- 输入（两个不同管线的生产与消费存在跨步依赖）：

```mlir
func.func @kernel()
  attributes {hacc.function_kind = #hacc.function_kind<DEVICE>} {
  // 管线 A 生产
  %dst = memref.alloc() : memref<1024xf16, #hivm.address_space<ub>>
  hivm.hir.load ins(%src_gm : memref<1024xf16, #hivm.address_space<gm>>) outs(%dst : memref<1024xf16, #hivm.address_space<ub>>)
  // 管线 B 消费
  hivm.hir.vmul ins(%dst, %w : memref<1024xf16, #hivm.address_space<ub>>,
                              memref<1024xf16, #hivm.address_space<ub>>)
                 outs(%dst : memref<1024xf16, #hivm.address_space<ub>>)
  return
}
```

- 运行 `-hivm-inject-sync`（`NORMAL`）后：
  - 在 `vmul` 前插入必要的同步，如 `hivm.pipe_barrier #pipe_all`，以保证 `load` 完成并可见；
  - 若同一管线内无跨核/跨缓存可见性问题，则不会插入冗余同步。
- 调试模式（`BARRIERALL`）：
  - 对每个 HIVM 指令和 `return` 前注入 `hivm.pipe_barrier`，便于定位问题或验证依赖。

**运行命令**

```bash
bishengir-opt -hivm-inject-sync input.mlir
# 可选：设置同步模式（根据管线集成）
# NORMAL：默认智能注入
# BARRIERALL：全量屏障，适合调试
```

**提示**

- 建议在内存规划、多缓冲展开与布局推断之后运行，以便同步分析具有稳定的缓冲地址与生命周期信息。
- 如遇到过度同步导致性能退化，可通过选项切换模式或调整管线，结合 SyncDebug 输出检查插入点。

## hivm-inject-block-sync

**作用**

- 面向“混合核（MIX）”函数，在块级别插入同步指令，协调 Cube 与 Vector 两种核心管线的生产-消费次序，避免跨核数据竞争。
- 三种注入策略：
  - 全量块同步：在特定 HIVM 指令后统一插入 `sync_block` + 成对的 `sync_block_set/wait`，用于强制同步或调试；
  - 浅层 CV：对浅融合（`ShallowCV`）场景，在关键算子（如 `matmul/mixmatmul`）与 `func.call` 之后插入必要的块同步；
  - 混合 CV 自动分析：构建块同步 IR，分析复合元素与内存依赖，去冗余、分配事件并生成最小化同步代码。

实现位置：

- 定义：`bishengir/include/bishengir/Dialect/HIVM/Transforms/Passes.td:182-192`
- 逻辑：`bishengir/lib/Dialect/HIVM/Transforms/InjectBlockSync.cpp`
  - 基地址设置：`55-77`
  - 工作区使用合法性检查：`79-99`
  - 核类型查询与标志生成：`101-134`
  - 同步原语生成：`135-170, 172-199`
  - 浅层同步插入：`201-222, 471-489`
  - 块同步 IR 翻译与分析：`224-402, 430-469`
  - Pass 主入口：`491-521`（随后分支到 `InjectAllBlockSync` 或 `InjectBlockMixSync`）

**处理流程**

- 仅在设备侧且函数核心类型为 `MIX` 时生效；否则跳过。
- 读取函数形参中 `FFTSBaseAddr` 并在函数首部插入 `hivm.set_ffts_base_addr`。
- 根据选项选择模式：
  - `blockAllSync`：对 `hivm.fixpipe` 和 `hivm.store` 等关键点注入 `sync_block` 以及 `sync_block_set/wait` 成对指令；
  - `ShallowCV`：对 `matmul/mixmatmul` 与 `func.call` 后插入块同步；
  - 自动分析：构建同步 IR，执行依赖分析、移动/去冗余、事件分配、同步代码生成。

**简单示例（ShallowCV）**

- 输入（MIX 核心函数，存在 Cube→Vector 的跨核依赖）：

```mlir
func.func @kernel(%ffts_base: i64 {hfusion.ffts_base_address})
  attributes {hacc.function_kind = #hacc.function_kind<DEVICE>, hivm.func_core_type = #hivm.func_core_type<MIX>} {
  %C = memref.alloc() : memref<128x128xf16, #hivm.address_space<ub>>
  %V = memref.alloc() : memref<128x128xf16, #hivm.address_space<ub>>
  hivm.hir.matmul ins(%A, %B) outs(%C)  // Cube 管线
  // 需要在此处与后续 Vector 消费之间插入块同步
  hivm.hir.vadd   ins(%C, %V) outs(%V)  // Vector 管线
  return
}
```

- 运行 `-hivm-inject-block-sync` 后（核心语义）：
  - 在 `matmul` 后生成块同步：
    - `hivm.hir.sync_block(mode=ALL_CUBE, flag_id=…)`
    - `hivm.hir.sync_block_set(core=CUBE, pipe=PIPE_FIX, flag_id=…)`
    - `hivm.hir.sync_block_wait(core=VECTOR, pipe=PIPE_FIX, flag_id=…)`
  - 若无跨核依赖，则可能仅插入 `sync_block` 作为轻量标记或不插。

**运行命令**

```bash
bishengir-opt -hivm-inject-block-sync input.mlir
```

**提示**

- 仅对 `MIX` 核心类型函数启用；需要函数形参包含 `FFTSBaseAddr` 并且工作区参数仅被 `alloc_workspace` 使用。
- 与 `hivm-inject-sync` 不同，该 pass 专注块级同步（Cube/Vector 核间），并提供浅层/全量/自动三种策略以权衡性能与安全性。

## hivm-graph-sync-solver

**作用**

- 在设备侧函数上进行图级同步求解：基于线性化的指令发生序列与内存冲突分析，为整个计算图推导最小化且无死锁的同步方案，插入必要的 `SetFlag/WaitFlag/PipeBarrier` 等同步指令。
- 与普通的指令级同步注入不同，采用图搜索与可行性检查，避免局部最优导致的全局死锁或过度同步；支持事件复用与回退到屏障策略。

实现位置：

- 定义：`bishengir/include/bishengir/Dialect/HIVM/Transforms/Passes.td:200-208`
- 逻辑：`bishengir/lib/Dialect/HIVM/Transforms/GraphSyncSolver/GraphSyncSolver.cpp`
  - Pass 主体：`70-130`
  - 测试模式：`75-77`
  - Solver 构建与选项：`88-93`
  - 调试输出：`96-108, 117-123`
  - 结果物化：`125-130`

**处理流程**

- 跳过宿主函数；为设备函数构建内部 `funcIr` 和线性化的 `syncIr`。
- 进行冲突检测、图可行性检查、事件分配与复用；必要时选择 `PipeBarrier` 回退。
- 将同步结果插入 MLIR：`hivm.set_flag`、`hivm.wait_flag`、`hivm.pipe_barrier` 等。

**简单示例**

- 输入（两条数据路径在不同管线产生交叉读写，需要图级协调）：

```mlir
func.func @kernel()
  attributes {hacc.function_kind = #hacc.function_kind<DEVICE>} {
  %A = memref.alloc() : memref<1024xf16, #hivm.address_space<ub>>
  %B = memref.alloc() : memref<1024xf16, #hivm.address_space<ub>>
  // 路径1：产生 %A 后使用 %B
  hivm.hir.load ins(%gm1) outs(%A)
  hivm.hir.vadd ins(%A, %B) outs(%B)
  // 路径2：产生 %B 后使用 %A
  hivm.hir.load ins(%gm2) outs(%B)
  hivm.hir.vmul ins(%B, %A) outs(%A)
  return
}
```

- 运行 `-hivm-graph-sync-solver` 后（核心语义）：
  - 求解器在线性化发生序列上检测到双向依赖冲突；
  - 插入 `set_flag/wait_flag` 或必要的 `pipe_barrier`，确保两路径间的生产-消费顺序无环且可进展；
  - 复用事件 ID 并去冗余，避免过多同步。

**运行命令**

```bash
bishengir-opt -hivm-graph-sync-solver input.mlir
```

**提示**

- 适合复杂图或多路径交叉的数据依赖场景，常在局部同步注入之后使用以做全局一致性修正。
- 可通过选项启用“unit-flag”特性改变代码生成策略；调试模式下还可开启测试驱动以验证求解器正确性。

## hivm-decompose-op

**作用**

- 将部分复杂或宏式的 HIVM 指令分解为更基础、可优化的序列，便于后续流水、内存规划与标量/矢量化降级处理。
- 典型分解包括：
  - 复杂类型转换 `VCastOp` 的级联拆分（如 f32→i8 拆为 f32→f16→i8；i8/i16→i1 拆为转换+比较）
  - 多轴广播 `VBrcOp` 拆分为按单轴逐级广播
  - 低精度倒数 `VRecOp` 分解为常量 1 的广播 + `VDivOp`
  - 比较 `VCmpOp` 的 i1 结果路径通过 i8 中间缓冲重写
  - 去混合出块同步宏 `SyncBlockOp` 为具体 `sync_block_set/wait` 或 `pipe_barrier`
  - `VDeinterleaveOp` 的全通道拆成逐通道版本
  - 原子存储/交换/比较-交换在不支持硬件原子时分解为锁+load/eltwise/store 的软件方案

实现位置：

- 定义：`bishengir/include/bishengir/Dialect/HIVM/Transforms/Passes.td:209-217`
- 逻辑：`bishengir/lib/Dialect/HIVM/Transforms/HIVMDecomposeOp.cpp`
  - `VCast` 分解：`54-82, 84-113, 115-156`
  - `VBrc` 多轴分解：`158-238`
  - 倒数高精度化：`243-346`
  - 块同步宏降解：`350-411`
  - `VCmp` 重写：`413-469`
  - 反交错全通道拆分：`922-947`
  - 原子软件实现：`950-1110`

**简单示例**

- 示例 1（多轴广播分解）：

```mlir
// 输入
%src = memref.alloc() : memref<1x1x10x1xi16>
%dst = memref.alloc() : memref<16x8x10x32xi16>
hivm.hir.vbrc ins(%src) outs(%dst) broadcast_dims = [0, 1, 3]
```

- 运行 `-hivm-decompose-op` 后：
  - 生成三次单轴 `vbrc`，按内到外顺序依次扩维，可能引入临时缓冲以承接中间结果。
- 示例 2（f32→i8 的级联转换）：

```mlir
// 输入
%src = memref.alloc() : memref<1024xf32, #hivm.address_space<ub>>
%dst = memref.alloc() : memref<1024xi8,  #hivm.address_space<ub>>
hivm.hir.vcast ins(%src) outs(%dst) round_mode = nearest
```

- 运行后：
  - 拆成 `vcast` f32→f16，再 `vcast` f16→i8，必要时保留转置/广播属性到第二步。
- 示例 3（同步宏降解）：

```mlir
hivm.hir.sync_block mode = ALL_VECTOR, tcube_pipe = #PIPE_FIX, tvector_pipe = #PIPE_MTE3, flag_id = 3
```

- 运行后：
  - 展开为成对的 `sync_block_set` 与 `sync_block_wait`（或 `pipe_barrier`），并删除宏指令。

**运行命令**

```bash
bishengir-opt -hivm-decompose-op input.mlir
```

**提示**

- Pass 仅在设备侧运行；常作为归一化与降级步骤，方便后续调度与优化。
- 某些分解会引入临时缓冲与锁；结合内存规划与同步求解 pass 使用，保证正确性与性能。

## hivm-aggregated-decompose-op

**作用**

- 针对实现了 `BiShengIRAggregatedOpInterface` 的“聚合操作”，按指定分解阶段将其解构为基础 HIVM 原语或更细粒度的子操作，或直接移除，保证复杂宏式/组合算子在合适时机被展开。
- 分解阶段受 `DecomposePhase` 控制：仅在操作自身声明的阶段与传入的阶段一致，或操作阶段为 `NO_CONSTRAINT` 时执行分解；分解结果可为空（表示该聚合操作被移除）。

实现位置：

- 定义：`bishengir/include/bishengir/Dialect/HIVM/Transforms/Passes.td:224-231`
- 逻辑：`bishengir/lib/Dialect/HIVM/Transforms/HIVMAggregatedDecomposeOp.cpp`
  - Pass 主体：`49-57, 99-106, 108-111`
  - 聚合接口匹配与分解：`59-95`

**处理流程**

- 仅在设备侧运行；
- 遍历所有实现 `BiShengIRAggregatedOpInterface` 的操作：
  - 若 `op.getDecomposePhase()` 与传入的 `decomposePhase` 相同或为不限阶段，调用 `op.decomposeOperation(rewriter)`；
  - 返回空结果则删除该操作；否则以新结果替换。

**简单示例**

- 假设有一个聚合矩阵操作 `hivm.agg.matmul_plus_bias` 实现了 `BiShengIRAggregatedOpInterface`，在阶段 `AFTER_NORMALIZE` 分解为普通 `matmul` 加 `add`：

```mlir
// 输入（聚合算子）
%out = hivm.hir.agg.matmul_plus_bias %A, %B, %bias
```

- 运行 `-hivm-aggregated-decompose-op` 且 `decomposePhase=AFTER_NORMALIZE` 后：
  - 调用接口 `decomposeOperation` 返回两步序列：
    - `hivm.hir.matmul ins(%A, %B) outs(%tmp)`
    - `hivm.hir.vadd   ins(%tmp, %bias) outs(%out)`
  - 用新结果替换原聚合操作；若该聚合在此阶段应直接消除，则删除它。

**运行命令**

```bash
bishengir-opt -hivm-aggregated-decompose-op input.mlir
```

**提示**

- 与 `hivm-decompose-op` 的固定模式分解不同，本 Pass 由每个聚合操作的接口决定具体展开形式与时机，适合控制复杂宏算子的生命周期。

## hivm-lower-to-loops

**作用**

- 对实现了 `ImplByScalarOpInterface` 的 HIVM 操作，在其声明需要降级的场景将其“按标量循环”实现：用标量指令的 `scf.for`/`affine.for` 等循环序列替代高阶矢量/块操作。
- 仅在 `op.shouldLowerToScalarLoops()` 返回真时触发；调用 `op.lowerToLoops(rewriter)` 生成替代 IR，并对性能发出警告。

实现位置：

- 定义：`bishengir/include/bishengir/Dialect/HIVM/Transforms/Passes.td:260-266`
- 逻辑：`bishengir/lib/Dialect/HIVM/Transforms/HIVMLowerToLoops.cpp`
  - 匹配接口与条件：`53-62`
  - 调用降级实现与替换：`63-77`
  - Pass 主入口：`82-89, 91-93`

**简单示例**

- 假设某 `hivm.vop` 在特定形状或类型下不支持硬件路径并实现了该接口：

```mlir
// 输入（设备侧函数）
func.func @kernel()
  attributes {hacc.function_kind = #hacc.function_kind<DEVICE>} {
  %A = memref.alloc() : memref<1024xf32, #hivm.address_space<ub>>
  %B = memref.alloc() : memref<1024xf32, #hivm.address_space<ub>>
  hivm.hir.vmul ins(%A, %B) outs(%A)
  return
}
```

- 当 `vmul` 的接口实现判定需降级时，运行 `-hivm-lower-to-loops`：
  - 将 `vmul` 替换为基于 `scf.for` 的逐元素乘法循环，可能形如：
    - `scf.for %i = %c0 to %c1024 step %c1 { %a_i = memref.load %A[%i]; %b_i = memref.load %B[%i]; %c_i = arith.mulf %a_i, %b_i; memref.store %c_i, %A[%i]; }`
  - 同时在 IR 中对该操作发出效率低的警告，提醒后续优化或规避。

**运行命令**

```bash
bishengir-opt -hivm-lower-to-loops input.mlir
```

**提示**

- 此 Pass 依赖各 HIVM 操作的接口实现来决定降级条件与生成的循环体；适合将小规模或不受支持的形态降至标量路径，保证可执行性。

## hivm-recognize-deinterleave-op

**作用**

- 识别源为全局内存（GM）且末维非连续、目标为本地缓冲（UB）且末维连续的访问模式，将原始 `load/copy` 重写为“先拷贝到中间连续缓冲，再用 `vdeinterleave` 重排”的等价序列，并自动标注步幅对齐以保证硬件可执行。
- 适用于 1D/2D 非连续到连续的通道解交错；不支持 i64 元素或 3D 以上形状。

实现位置：

- 定义：`bishengir/include/bishengir/Dialect/HIVM/Transforms/Passes.td:267-275`
- 逻辑：`bishengir/lib/Dialect/HIVM/Transforms/RecognizeDeinterleaveOp.cpp`
  - 模式检测：`141-163`
  - 追溯静态 `alloc` 与 `subview`：`35-58, 217-235, 286-304`
  - 中间缓冲与替换：`241-258, 311-327`
  - 步幅对齐标注：`165-175, 260-264, 329-333`
  - Pass 主入口：`344-359`

**处理流程**

- 对 `hivm.load` 与 `hivm.copy`：
  - 判断源末维“非单位步幅”且目标末维连续；源在 GM，目标在 UB；
  - 追溯目标是否源自静态 `alloc`（支持单层 `subview`）且偏移为零；
  - 复用旧 `alloc` 作为 `vdeinterleave` 的目标，新建同型 `alloc` 作为 `load/copy` 的目标并作为 `vdeinterleave` 的源；
  - 调整原 `load/copy` 的目标为新中间缓冲，在其后插入 `hivm.vdeinterleave`；
  - 为中间缓冲末维添加步幅对齐标注，使用硬件对齐字节数，使 `vdeinterleave` 能正确执行。

**简单示例**

- 输入（GM→UB，源不连续、目标连续）：

```mlir
func.func @kernel(%gm: memref<256x128xf32, #hivm.address_space<gm>>)
  attributes {hacc.function_kind = #hacc.function_kind<DEVICE>} {
  %ub = memref.alloc() : memref<256x128xf32, #hivm.address_space<ub>>
  hivm.hir.load ins(%gm) outs(%ub)
  return
}
```

- 若检测到 `%gm` 的末维为非连续、`%ub` 为连续，运行 `-hivm-recognize-deinterleave-op` 后：
  - 新建中间缓冲 `%ub_tmp`，将 `load` 的 `outs` 改为 `%ub_tmp`；
  - 在 `load` 后插入 `hivm.vdeinterleave ins(%ub_tmp) outs(%ub) channel_num = (align_bytes / elem_bytes) mode = channel0`；
  - 在 `%ub_tmp` 上标注步幅对齐以满足硬件约束。

**运行命令**

```bash
bishengir-opt -hivm-recognize-deinterleave-op input.mlir
```

**提示**

- 目前仅支持单层 `subview` 的静态 `alloc` 追溯且偏移为零；复杂视图链或动态形状需在前序 pass 中规范化或对齐。

## hivm-opt-single-point

**Pass 作用**

- 这个 Pass 是在函数级别运行的优化（`func::FuncOp`），名称为 `hivm-opt-single-point`，定义位置在 `bishengir/include/bishengir/Dialect/HIVM/Transforms/Passes.td:277-284`。
- 作用是识别“单点”场景（目标 `memref` 的各维度都是 1，只存一个元素），把一批 HIVM 的向量算子用标量算子替换，并把数据通过单元素的 `memref.load`/`memref.store` 来读写，从而避免整向量/大块内存的开销。
- 具体覆盖：
  - Elementwise：`VAdd`, `VSub`, `VMul`, `VDiv`, `VAbs`, `VSqrt`, `VMax`, `VMin` 映射到 `arith`/`math` 的标量算子（实现见 `bishengir/lib/Dialect/HIVM/Transforms/HIVMOptSinglePoint.cpp:261-270`）。
  - 广播：`VBrc` 简化为对单元素的加载和存储（`memref.load`/`memref.store`）（位置 `HIVMOptSinglePoint.cpp:125-157`）。
  - 拷贝/加载：`hivm.copy` 和 `hivm.load` 在满足安全条件时替换为单元素 `load/store`（位置 `HIVMOptSinglePoint.cpp:194-249`）。注意：`hivm.store` 不会被优化（位置 `HIVMOptSinglePoint.cpp:204-207`）。
- 只在设备函数上运行，跳过 Host 函数（`bishengir/lib/Dialect/HIVM/Transforms/HIVMOptSinglePoint.cpp:254-255`）。
- 在默认管线中也会自动调用：`canonicalizationHIVMPipeline` 包含该 Pass（`bishengir/lib/Dialect/HIVM/Pipelines/HIVMPipelines.cpp:42`）。
- 工具函数：标量值提取与单点读写由 `utils::getScalarValue`、`utils::createSinglePointLoad/Store` 完成（声明见 `bishengir/include/bishengir/Dialect/Utils/Util.h:271-285`）。

**触发条件**

- 纯缓冲语义（`hasPureBufferSemantics`），且只存在一个 `init`（`getDpsInits().size() == 1`）。
- 目标为 `memref`，且其每个维度都是 1（形状检查，见 `HIVMOptSinglePoint.cpp:85-98`）。
- 元素类型限制：目前只对 `f32` 或 `i64` 做标量替换（`HIVMOptSinglePoint.cpp:99-105`），这是硬件支持约束。
- 对 `copy/load` 的安全性检查：
  - 目标形状为单元素（`HIVMOptSinglePoint.cpp:230-233`）。
  - 源/目的都带 HIVM 的内存空间属性（`HIVMOptSinglePoint.cpp:226-228`）。
  - 源 GM 的使用链中不能出现写或未知用户（BFS 检查，`HIVMOptSinglePoint.cpp:159-192`）。
  - 对 `hivm.load` 仅在函数有 `no_alias` 属性时优化（`HIVMOptSinglePoint.cpp:210-216`，属性判断见 `bishengir/lib/Dialect/HACC/Utils/Utils.cpp:69-71`）。

**简单示例（前后对比）**

- Elementwise 加法（`f32` 单元素）

```mlir
// 运行前
func.func @kernel(%a: memref<1xf32, #hivm.address_space<UB>>,
                  %b: memref<1xf32, #hivm.address_space<UB>>,
                  %dst: memref<1xf32, #hivm.address_space<UB>>) {
  hivm.hir.vadd ins(%a, %b) outs(%dst)
  return
}

// 运行后（核心逻辑）
%v0 = memref.load %a[0] : memref<1xf32, #hivm.address_space<UB>>
%v1 = memref.load %b[0] : memref<1xf32, #hivm.address_space<UB>>
%sum = arith.addf %v0, %v1
memref.store %sum, %dst[0] : memref<1xf32, #hivm.address_space<UB>>
return
```

- 广播到单元素

```mlir
// 运行前
func.func @brc(%src: f32, %dst: memref<1xf32, #hivm.address_space<UB>>) {
  hivm.hir.vbrc ins(%src) outs(%dst)
  return
}

// 运行后（核心逻辑）
memref.store %src, %dst[0] : memref<1xf32, #hivm.address_space<UB>>
return
```

- GM→UB 的单元素拷贝（满足安全约束时）

```mlir
// 运行前
func.func @copy(%gm: memref<1xf32, #hivm.address_space<GM>>,
                %ub: memref<1xf32, #hivm.address_space<UB>>) attributes {hacc.no_io_alias} {
  hivm.hir.copy ins(%gm) outs(%ub)
  return
}

// 运行后（核心逻辑）
%val = memref.load %gm[0] : memref<1xf32, #hivm.address_space<GM>>
memref.store %val, %ub[0] : memref<1xf32, #hivm.address_space<UB>>
return
```

**如何运行**

- 独立使用命令行：

```bash
mlir-opt -hivm-opt-single-point input.mlir -o output.mlir
```

- 在默认管线中，它已被包含在 `canonicalizationHIVMPipeline`（参考 `bishengir/lib/Dialect/HIVM/Pipelines/HIVMPipelines.cpp:42`），因此按项目提供的管线入口即可触发。

**代码参考**

- 定义与简介：`bishengir/include/bishengir/Dialect/HIVM/Transforms/Passes.td:277-284`
- Pass 类与注册：`bishengir/lib/Dialect/HIVM/Transforms/HIVMOptSinglePoint.cpp:42-45, 276-278`
- Elementwise 重写规则：`bishengir/lib/Dialect/HIVM/Transforms/HIVMOptSinglePoint.cpp:70-123, 261-270`
- 广播重写：`bishengir/lib/Dialect/HIVM/Transforms/HIVMOptSinglePoint.cpp:125-157`
- 拷贝/加载重写与安全检查：`bishengir/lib/Dialect/HIVM/Transforms/HIVMOptSinglePoint.cpp:194-249`
- 主入口（设备函数约束）：`bishengir/lib/Dialect/HIVM/Transforms/HIVMOptSinglePoint.cpp:252-255`
- 管线集成：`bishengir/lib/Dialect/HIVM/Pipelines/HIVMPipelines.cpp:42`

## hivm-alloc-extra-buffer

**Pass 作用**

- 名称与位置：`AllocExtraBuffer` 在 `bishengir/include/bishengir/Dialect/HIVM/Transforms/Passes.td:286`，作用于 `func::FuncOp`。
- 作用：为需要“临时缓冲区”的 HIVM 运算分配额外的 `memref.alloc` 缓冲，并填充到运算的可选 `temp_buffer` 操作数中，以满足硬件指令或内联广播等实现需求。
- 工作机制：
  - 遍历函数内所有实现了 `ExtraBufferOpInterface` 的 HIVM 运算，调用 `allocExtraBuffersIfPossible()`。
  - 若该运算尚无 `temp_buffer`，则依据 `getExtraBufferSize()` 或特定规则计算大小，插入 `memref.alloc`，并通过 `getTempBufferMutable().assign(...)` 绑定到运算上。
  - 跳过 Host 函数，仅在设备函数上生效。
- 关键实现：`bishengir/lib/Dialect/HIVM/Transforms/AllocExtraBuffer.cpp:48-64`，接口定义在 `bishengir/include/bishengir/Dialect/HIVM/Interfaces/ExtraBufferOpInterface.td`。

**适用运算与分配策略**

- 自动支持的运算包含大量向量一元/二元算子及特殊算子，例如 `VAdd`, `VMul`, `VSub`, `VDiv`, `VAbs`, `VSqrt`, `VRelu`, `VExp`, `VLn`, `VNot`, `VMax`, `VMin`, `VShL`, `VShR`，以及 `VBrc`, `VCast`, `VGather`, `VInterleave`, `VMulextended`, `VPow`, `VReduce`, `VSel`, `VSort`, `VTranspose`, `VXor` 等。分配逻辑散见：
  - 通用宏实现：`ExtraBufferAllocations.cpp:46-83` 为多数 Vector ops 直接根据 `getExtraBufferSize()` 分配。
  - 特例：
    - `VBrcOp`：广播维度需先分解为单维；随后按内联广播的块对齐计算缓冲（`ExtraBufferAllocations.cpp:89-108`，大小计算函数参见 `GetExtraBuffers.cpp:117-140, 164-215`）。
    - `VCastOp`：不同类型转换有不同缓冲策略，例如 i32→i8、i16→i8 需要额外转置缓冲；i32→i16、i64→i32 在特定 RoundMode 下分配“大小为 0 的占位缓冲”（为统一接口）（`ExtraBufferAllocations.cpp:114-163`）。
    - `VGatherOp`：缓冲大小由 `indices` 的分配最大尺寸决定（`ExtraBufferAllocations.cpp:169-186`）。
    - `VInterleaveOp`：按照块/重复对齐计算（`ExtraBufferAllocations.cpp:192-211`）。
    - `VMulextendedOp`：按 32 位块对齐、三倍缓冲（`ExtraBufferAllocations.cpp:217-236`）。
    - `VPowOp`（仅 i32 需要）：组合若干中间缓冲（条件、两份源大小、布尔视图、reduce 用临时）并做对齐求和（`ExtraBufferAllocations.cpp:242-285`）。
    - `VReduceOp`：保持与源/目的相同秩的临时缓冲（`ExtraBufferAllocations.cpp:291-307`）。
    - `VSelOp`：依据类型与标量参与情况，计算每重复块需要的元素数并分配（`ExtraBufferAllocations.cpp:314-371`）。
    - `VSortOp`：按排序维长度与方向、是否输出索引来决定（`ExtraBufferAllocations.cpp:377-392` 与 `GetExtraBuffers.cpp:319-364`）。
    - `VTransposeOp`：当元素为 i64 且最后轴参与转置时，分配两块固定大小的临时缓冲（总计 512 元素）（`ExtraBufferAllocations.cpp:398-423`）。
    - `VXorOp`：大小等于源的分配最大尺寸（`ExtraBufferAllocations.cpp:429-444`）。

**简单示例**

- 将 `VAddOp` 的向量标量混合（vs）场景分配额外缓冲，用于内联广播或硬件不支持的 vs 指令：

```mlir
// 运行前
func.func @kernel(%src: memref<16xf32, #hivm.address_space<UB>>,
                  %scalar: f32, 
                  %dst: memref<16xf32, #hivm.address_space<UB>>) {
  // 可能需要额外缓冲做内联广播或支持 vs
  hivm.hir.vadd ins(%src, %scalar) outs(%dst)
  return
}

// 运行后（核心逻辑，插入临时缓冲）
// 根据 getExtraBufferSize() 计算的元素数，生成：
// %tmp = memref.alloc() : memref<?xf32, #hivm.address_space<UB>>
hivm.hir.vadd ins(%src, %scalar) outs(%dst) temp_buffer(%tmp)
return
```

- 广播运算 `VBrcOp` 在最后轴内联广播的情形，分配一块按块对齐的临时缓冲：

```mlir
// 运行前
func.func @brc(%src: memref<1x128xf16, #hivm.address_space<UB>>,
               %dst: memref<64x128xf16, #hivm.address_space<UB>>) {
  hivm.hir.vbrc ins(%src) outs(%dst) broadcast = [0] // 最后一轴内联
  return
}

// 运行后（核心逻辑）
%tmp = memref.alloc() : memref<?xf16, #hivm.address_space<UB>>
hivm.hir.vbrc ins(%src) outs(%dst) temp_buffer(%tmp) broadcast = [0]
return
```

- 类型转换 `VCastOp` i32→i8 需要转置中间结果的缓冲：

```mlir
// 运行前
func.func @cast(%src: memref<1024xi32, #hivm.address_space<UB>>,
                %dst: memref<1024xi8, #hivm.address_space<UB>>) {
  hivm.hir.vcast ins(%src) outs(%dst) round_mode = #hivm.round_mode<trunc>
  return
}

// 运行后（核心逻辑）
%tmp = memref.alloc() : memref<?xi32, #hivm.address_space<UB>> // 大小取决于 src 分配最大尺寸的两倍等规则
hivm.hir.vcast ins(%src) outs(%dst) temp_buffer(%tmp) round_mode = #hivm.round_mode<trunc>
return
```

**何时运行**

- 独立使用命令行：

```bash
mlir-opt -hivm-alloc-extra-buffer input.mlir -o output.mlir
```

- 也可在自定义 Pass 管线中在需要阶段插入该 Pass；其构造函数为 `mlir::hivm::createAllocExtraBufferPass()`。

**代码参考**

- Pass 定义：`bishengir/include/bishengir/Dialect/HIVM/Transforms/Passes.td:286-290`
- Pass 实现：`bishengir/lib/Dialect/HIVM/Transforms/AllocExtraBuffer.cpp:41-64`
- 接口与分配逻辑：`bishengir/include/bishengir/Dialect/HIVM/Interfaces/ExtraBufferOpInterface.td`、`bishengir/lib/Dialect/HIVM/IR/ExtraBufferOpInterface/ExtraBufferAllocations.cpp`、`bishengir/lib/Dialect/HIVM/IR/ExtraBufferOpInterface/GetExtraBuffers.cpp`

## hivm-constantize-buffer-size

**Pass 作用**

- 名称与位置：`ConstantizeBufferSize` 在 `bishengir/include/bishengir/Dialect/HIVM/Transforms/Passes.td:293`，作用于 `func::FuncOp`。
- 作用：把动态大小的本地缓冲（`memref.alloc`/`memref.alloca`）尽可能“常量化”。它为原本带动态维度的分配，计算每个动态维度的常量上界，如果能得到全部维度的上界，则将其替换为一个按总字节数分配的静态 `memref`，并通过 `memref.view` 还原到原始形状接口。
- 与标注交互：
  - 如果存在 `annotation.mark` 对该缓冲设置了 `buffer_size_in_byte`，常量化后的总大小不能超过该值。
  - 成功常量化后，移除所有关联的 `buffer_size_in_byte` 标注。

**实现要点**

- 入口：`bishengir/lib/Dialect/HIVM/Transforms/ConstantizeBufferSize.cpp`
  - 仅在设备函数上运行（`isHost` 跳过）。
  - 对 `memref::AllocOp` 和 `memref::AllocaOp` 应用同样的重写模式（`ConstantizeAllocLikeOp`）。
- 常量化流程（`ConstantizeAllocLikeOp::matchAndRewrite`）：
  - 读取动态维度；使用 `ValueBoundsConstraintSet::computeConstantBound(UB)` 为每个动态维度计算常量上界。
  - 若所有维度均已确定（原始静态 + 成功求得的上界），计算总位数与字节数。
  - 构造新的静态 `memref<total_bytes x i8, space>` 分配；以 `memref.view` 把它视为原始的 `memref<...>` 类型，替换旧分配。
  - 若存在 `buffer_size_in_byte` 标注，校验总大小不超标并删除这些标注。
- 失败路径：
  - 分配已全静态，或无法为某些维度求出上界，则保持不变。
  - 有标注但常量化后超过上限，也保持不变。

**简单示例（前后对比）**

- 动态形状缓冲常量化

```mlir
// 运行前
%alloc = memref.alloc(%n) : memref<?x16xf32, #hivm.address_space<UB>>

// 运行后（假设能求出 %n 的常量上界 64）
%bytes = memref.alloc() : memref<4096xi8, #hivm.address_space<UB>> // 64*16*4 = 4096B
%zero = arith.constant 0 : index
%view = memref.view %bytes[%zero][%n, 16] : memref<?x16xf32, #hivm.address_space<UB>>
```

- 与标注的约束

```mlir
// 运行前：存在设置的最大缓冲大小标注（单位：字节）
%alloc = memref.alloc(%n) : memref<?xf32, #hivm.address_space<UB>>
%mark = annotation.mark %alloc { buffer_size_in_byte = 1024 : i64 }

// 若能常量化且总大小 <= 1024：常量化并删除标注
// 若常量化后的总大小 > 1024：保持不变，避免超出限制
```

**使用方式**

- 独立运行：

```bash
mlir-opt -hivm-constantize-buffer-size input.mlir -o output.mlir
```

- 在管线中插入该 Pass 的入口函数是 `mlir::hivm::createConstantizeBufferSizePass()`。

**代码参考**

- 定义与说明：`bishengir/include/bishengir/Dialect/HIVM/Transforms/Passes.td:292-303`
- 实现：`bishengir/lib/Dialect/HIVM/Transforms/ConstantizeBufferSize.cpp:38-155`
- 标注工具：`buffer_size_in_byte` 相关处理见 `ConstantizeBufferSize.cpp:117-137` 与自动标注 Pass `AutoInferBufferSize.cpp`（可为未标注的动态分配注入推断值，便于后续常量化）。

## hivm-opt-func-output

**Pass 作用**

- 名称与位置：`HIVMOptFuncOutput` 在 `bishengir/include/bishengir/Dialect/HIVM/Transforms/Passes.td:305`，作用于 `ModuleOp`。
- 作用：在缓冲化之后，优化函数的返回值列表，移除“没有必要返回”的缓冲类型返回值。具体是检测函数的 `return` 操作数中那些其实就是函数的某个形参（`memref` 的 `BlockArgument`）且没有必要通过返回值再传出，此时：
  - 更新函数签名，删除这些冗余的返回值类型。
  - 更新所有调用点，将原调用的对应返回值使用替换为调用参数（也就是该 `memref` 的实参）。
  - 更新函数体内的 `return`，只保留真正需要返回的值。

**实现细节**

- 入口与逻辑：`bishengir/lib/Dialect/HIVM/Transforms/HIVMOptFuncOutput.cpp`
  - 遍历模块中的每个 `func::FuncOp`，收集所有调用点（`SymbolTable::getSymbolUses`）。
  - 在被调用函数内，遍历 `func.return` 的操作数，找出那些是 `memref` 且是当前函数块参数的返回项（认为是冗余）。
  - 对每个调用点：
    - 构建新的 `func.call`，返回仅包含“仍需要的返回值”。
    - 对被删除的返回项，把旧的调用结果替换为该调用的对应输入参数。
    - 对保留的返回项，把旧调用结果替换为新调用的相应结果。
    - 删除旧调用。
  - 更新被调用函数的 `return` 操作数和函数类型。
- 仅处理设备无关的函数签名层面，不涉及具体 HIVM 运算。

**简单示例（前后对比）**

- 函数返回中包含冗余的缓冲地址

```mlir
// 运行前
func.func @callee(%buf0: memref<128xf32>, %x: i32) -> (memref<128xf32>, i32) {
  // 返回第一个值为输入缓冲参数 %buf0（冗余）
  return %buf0, %x : memref<128xf32>, i32
}

func.func @caller(%arg0: memref<128xf32>, %arg1: i32) {
  %r0, %r1 = func.call @callee(%arg0, %arg1) : (memref<128xf32>, i32)
  // 使用 %r0 的地方会被替换为 %arg0
  // 使用 %r1 保留
  return
}

// 运行后（核心变化）
func.func @callee(%buf0: memref<128xf32>, %x: i32) -> (i32) {
  return %x : i32
}

func.func @caller(%arg0: memref<128xf32>, %arg1: i32) {
  %r1 = func.call @callee(%arg0, %arg1) : (i32)
  // 原来使用 %r0 的位置，直接使用 %arg0
  return
}
```

**使用方式**

- 独立运行：

```bash
mlir-opt -hivm-opt-func-output input.mlir -o output.mlir
```

- 在管线中插入该 Pass 的入口函数为 `mlir::hivm::createHIVMOptFuncOutputPass()`。

**代码参考**

- 定义与说明：`bishengir/include/bishengir/Dialect/HIVM/Transforms/Passes.td:305-310`
- 实现：`bishengir/lib/Dialect/HIVM/Transforms/HIVMOptFuncOutput.cpp:35-133`

## hivm-split-mix-kernel

**Pass 作用**

- 名称与位置：`SplitMixKernel` 在 `bishengir/include/bishengir/Dialect/HIVM/Transforms/Passes.td:312`，作用于 `ModuleOp`。
- 作用：将标记为 Mix 内核的设备函数拆分为两个独立函数，分别针对 AICube 与 AIVector。通过核心类型属性识别和过滤，将同一函数中的“Cube 管线”与“Vector 管线”运算分离，生成：
  - AIC 版本函数：仅保留 Cube 运算，删除 Vector 运算
  - AIV 版本函数：仅保留 Vector 运算，删除 Cube 运算
- 同时，为 Host 调用场景生成 Mix 函数的声明（`func` 声明），并标注必要属性，使后续编译/调用正确处理。

**实现要点**

- 入口与整体逻辑：`bishengir/lib/Dialect/HIVM/Transforms/SplitMixKernel.cpp`
  - 遍历模块，跳过 Host 函数，仅处理设备函数（`runOnOperation`）。
  - 只对具备 `tcore_type = MIX` 的函数生效（`splitMixKernel` 检查 `TFuncCoreTypeAttr`）。
  - 先生成 Mix 函数的声明（仅当存在 Host 调用者时），并标注：
    - `HACCFuncType = DEVICE`
    - `TFuncCoreType = MIX`
    - 若原函数是 Device Entry，则在声明上加 `MIX_ENTRY` 标记
  - 复制原函数，得到两份：
    - 原函数重命名为 `<name>_mix_aic`，标注 `TFuncCoreType = AIC`
    - 克隆函数重命名为 `<name>_mix_aiv`，标注 `TFuncCoreType = AIV`
    - 两者都设置 `TPartOfMix` 属性以标识为 Mix 的分片
  - 过滤阶段：
    - 在 AIC 函数中删除 Vector 运算；在 AIV 函数中删除 Cube 运算（`filterMixFunc`）
    - 为被删除运算的输入做注记传播，以维护数据流可用性（`annotateOpOperand`），并将结果使用替换为 `init`/输出操作数（`replaceResultWithInitOperand`）
    - 针对循环（`scf.for`）若该循环属于被过滤的核心类型，直接把上界设为下界，使循环空转
  - 后处理：
    - AIC 函数：应用模式替换以修复 `tensor.extract` 等，并清理标记过的 `to_tensor`/`load`/`alloc`（`postProcessCubeFunc`）
    - AIV 函数：清理标记过的 `annotation.mark` 和新增的 `tensor.extract`（`postProcessVectorFunc`）

**简单示例（前后对比）**

- 输入函数（Mix）

```mlir
// 输入：Mix 内核函数，既有 Cube 又有 Vector 运算
func.func @kernel(%workspace: memref<...>) attributes {
  hivm.tfunc_core_type = #hivm.tcore_type<MIX>
} {
  %t0 = hivm.hir.cube_op ins() outs(%workspace)
  %t1 = hivm.hir.vector_op ins(%t0) outs(%workspace)
  return
}
```

- 拆分结果

```mlir
// 生成声明（若存在 Host 调用者）
func.func private @kernel() attributes {
  hacc.func_type = #hacc.func_type<DEVICE>,
  hivm.tfunc_core_type = #hivm.tcore_type<MIX>,
  // 若是 Device Entry：添加 mix_entry 标记
}

// AIC 版本，仅保留 Cube 运算
func.func @kernel_mix_aic(%workspace: memref<...>) attributes {
  hivm.tfunc_core_type = #hivm.tcore_type<AIC>,
  hivm.part_of_mix
} {
  %t0 = hivm.hir.cube_op ins() outs(%workspace)
  // vector_op 被删除；其使用替换为对应 init 操作数
  return
}

// AIV 版本，仅保留 Vector 运算
func.func @kernel_mix_aiv(%workspace: memref<...>) attributes {
  hivm.tfunc_core_type = #hivm.tcore_type<AIV>,
  hivm.part_of_mix
} {
  // cube_op 被删除；其输出改由 workspace 直接提供或通过标注传播
  %t1 = hivm.hir.vector_op ins(%workspace) outs(%workspace)
  return
}
```

**管线位置与约束**

- 放置于缓冲化之前运行（依赖张量 SSA 性质）：`bishengir/lib/Dialect/HIVM/Pipelines/HIVMPipelines.cpp:204-214`
- 目前只支持 Host 侧对 Mix 函数的调用；如果有设备函数调用 Mix 内核，会报错并终止（`generateMixKernelDecl` 检查调用者类型）。

**代码参考**

- 定义与说明：`bishengir/include/bishengir/Dialect/HIVM/Transforms/Passes.td:312-349`
- 实现入口：`bishengir/lib/Dialect/HIVM/Transforms/SplitMixKernel.cpp:143-356`
- 管线集成：`bishengir/lib/Dialect/HIVM/Pipelines/HIVMPipelines.cpp:204-214`

## hivm-mark-real-core-type

**Pass 作用**

- 名称与位置：`MarkRealCoreType` 在 `bishengir/include/bishengir/Dialect/HIVM/Transforms/Passes.td:351`，作用于 `ModuleOp`。
- 作用：为实际会在特定核心管线执行的“标量/内存访问类”操作标注真实的核心类型属性（`hivm.tcore_type`），用于后续的跨核心同步分析与插入。例如为 `memref.load/store`、`affine.load/store`、`tensor.extract/insert`、`hivm.load` 等打上 `CUBE`、`VECTOR` 或 `CUBE_AND_VECTOR`。
- 清理模式：该 Pass 支持一个选项 `removeCoreTypeAttrs`，为真时会移除这些属性，作为后续阶段的清理。

**实现思路**

- 入口与实现：`bishengir/lib/Dialect/HIVM/Transforms/MarkRealCoreType.cpp`
  - 标注对象类型筛选函数：`isOpTypeToBeMarked`，包括 `memref.load/store`、`affine.load/store`、`tensor.extract/insert`、`hivm.load` 等。
  - 若 `removeCoreTypeAttrs` 为真，遍历模块删除这些操作上的 `TCoreTypeAttr`，并直接返回。
  - 否则：
    - 克隆整个 `ModuleOp`，在克隆中为每个操作打一个自增的指令计数标记属性（`instruction-marker`），并记录克隆操作到原操作的映射。
    - 对克隆模块运行 `SplitMixKernel` 与标准规范化管线，借此让函数级核心类型（AIC 或 AIV）落地，并据此遍历克隆函数内的操作，反推每条指令的核心类型。
      - 如果操作只出现在 AIC 函数，则标注为 `CUBE`；只出现在 AIV，则标注为 `VECTOR`；若同时出现在两者则标注为 `CUBE_AND_VECTOR`。
    - 回到原模块，根据映射给对应的原始操作设置 `hivm.tcore_type` 属性。

**简单示例**

- 在跨核心同步（如插入块级同步）前，为标量/内存访问打核心类型，后续同步逻辑即可识别跨核心的读写冲突：

```mlir
// 运行前
%v = memref.load %gm[%i] : memref<?xf32, #hivm.address_space<GM>>
memref.store %v, %ub[%j] : memref<?xf32, #hivm.address_space<UB>>

// 运行后（核心类型标注）
%v = memref.load %gm[%i] : memref<?xf32, #hivm.address_space<GM>> 
    attributes { hivm.tcore_type = #hivm.tcore_type<VECTOR> }  // 例如推断在 Vector 管线侧
memref.store %v, %ub[%j] : memref<?xf32, #hivm.address_space<UB>>
    attributes { hivm.tcore_type = #hivm.tcore_type<CUBE> }    // 例如推断在 Cube 管线侧
```

- 当清理阶段需要移除这些辅助属性：

```bash
mlir-opt -hivm-mark-real-core-type="remove-core-type-attrs=true" input.mlir -o output.mlir
```

**在管线中的作用**

- 在跨核心同步管线中先标注、后注入、再清理：
  - 标注：`createMarkRealCoreTypePass()` 放在注入块同步前（`bishengir/lib/Dialect/HIVM/Pipelines/HIVMPipelines.cpp:73-89`）
  - 插入：随后运行 `createInjectBlockSyncPass(...)`，识别不同核心类型的访存冲突并插入同步
  - 清理：再次运行 `createMarkRealCoreTypePass(removeCoreTypeAttrs=true)` 删除属性，避免污染后续阶段

**代码参考**

- 定义与说明：`bishengir/include/bishengir/Dialect/HIVM/Transforms/Passes.td:351-359`
- 实现与选项：`bishengir/lib/Dialect/HIVM/Transforms/MarkRealCoreType.cpp:40-148`
- 管线集成：`bishengir/lib/Dialect/HIVM/Pipelines/HIVMPipelines.cpp:73-89`

## hivm-set-buffer-size

**Pass 作用**

- 名称与位置：`SetBufferSize` 在 `bishengir/include/bishengir/Dialect/HIVM/Transforms/Passes.td:361`，作用于 `func::FuncOp`。
- 作用：读取 `annotation.mark` 上的 `buffer_size_in_byte` 标注，把相关的动态形状 `memref.alloc`/`memref.alloca` 转换成具有固定总字节数的静态分配，并用 `memref.view` 恢复成原来的逻辑形状接口。完成后移除这些标注属性。
- 约束与冲突检测：
  - 若同一分配点（同一个 `alloc`/`alloca`）被标注了不同的缓冲大小，触发错误并终止。
  - 若分配本身已是静态形状，仅移除标注，不做转换。

**实现要点**

- 入口与核心逻辑：`bishengir/lib/Dialect/HIVM/Transforms/SetBufferSize.cpp`
  - 跳过 Host 函数，仅在设备侧运行（`isHost`）。
  - 遍历所有 `annotation::MarkOp`，寻找带 `buffer_size_in_byte` 的标注；经 `utils::tracebackMemRef` 追溯到根分配（`memref.alloc/alloca`），对每个分配汇总一个一致的缓冲大小。
  - 对每个需要设置的分配调用 `utils::createAllocWithSettingBufferSize`：
    - 新建 `memref<total_bytes x i8, space>` 的静态分配
    - 插入一个 `memref.view`，以原始形状/类型视图该字节缓冲
    - 替换原分配为 `view` 结果
  - 在成功设置时，移除相应标注的 `buffer_size_in_byte` 属性。

**简单示例（前后对比）**

- 为动态形状分配设置固定总字节大小

```mlir
// 运行前：动态维度的分配，随后有标注写入固定字节数信息
%alloc = memref.alloc(%n) : memref<?x16xf32, #hivm.address_space<UB>>
annotation.mark %alloc { buffer_size_in_byte = 4096 : i64 }

// 运行后：将分配转换为固定总字节数的静态分配 + view，还原逻辑形状
%bytes = memref.alloc() : memref<4096xi8, #hivm.address_space<UB>>
%zero = arith.constant 0 : index
%view = memref.view %bytes[%zero][%n, 16] : memref<?x16xf32, #hivm.address_space<UB>>
// 删除标注属性
```

- 已是静态形状的分配，仅清理标注：

```mlir
%alloc = memref.alloc() : memref<64x16xf32, #hivm.address_space<UB>>
annotation.mark %alloc { buffer_size_in_byte = 4096 : i64 }

// 运行后：保持分配不变，仅移除标注属性
```

**使用方式**

- 独立运行：

```bash
mlir-opt -hivm-set-buffer-size input.mlir -o output.mlir
```

- 在管线中可以配合 `AutoInferBufferSize`（自动为未标注的分配推断字节数）与 `ConstantizeBufferSize`（基于上界推断常量化）使用，以获得更好的内存确定性。

**代码参考**

- 定义与说明：`bishengir/include/bishengir/Dialect/HIVM/Transforms/Passes.td:361-364`
- 实现：`bishengir/lib/Dialect/HIVM/Transforms/SetBufferSize.cpp:39-104`
- 相关工具函数：`utils::tracebackMemRef` 与 `utils::createAllocWithSettingBufferSize`（声明在 `bishengir/include/bishengir/Dialect/Utils/Util.h`）

## hivm-map-forall-to-blocks

**Pass 作用**

- 名称与位置：`HIVMMapForallToBlocks` 在 `bishengir/include/bishengir/Dialect/HIVM/Transforms/Passes.td:366`，作用于 `func::FuncOp`。
- 作用：把顶层的 `scf.forall` 并行循环映射到 HIVM 的“块”执行模型。每个 `scf.forall` 的迭代被转为对应的 HIVM block 上的执行，并把 `scf.forall` 的归纳变量改写为 HIVM 的 block 索引操作。
- 限制：仅处理不嵌套的顶层 `scf.forall`；Host 函数跳过。

**实现细节**

- 入口与逻辑：`bishengir/lib/Dialect/HIVM/Transforms/HIVMMapForallToBlocks.cpp`
  - 模式 `ForallToBlocksPattern` 匹配顶层 `scf.forall`，调用 `mapForallToBlocksImpl` 完成实际映射。
  - 在 Pass 中还引入了 `affine::populateAffineExpandIndexOpsPatterns`，用于展开/规范索引相关操作，配合块映射后的索引替换。
  - Host 函数直接返回，不作处理。

**简单示例（概念示意）**

```mlir
// 运行前：顶层并行 forall，二维并行
scf.forall (%i, %j) in (0 to %M, 0 to %N) {
  // 在每个并行迭代中执行某些 HIVM op
  ...
}

// 运行后（示意）：映射到 HIVM blocks，并将归纳变量替换为 block 索引
hivm.block (%bi, %bj) {
  // %i -> %bi, %j -> %bj；其余并行体逻辑保持
  ...
}
```

- 如果原 `forall` 中存在 `affine.delinearize_index` 或类似索引重建操作，Pass 会加模式展开这些索引以适配块映射。

**使用方式**

- 独立运行：

```bash
mlir-opt -hivm-map-forall-to-blocks input.mlir -o output.mlir
```

- 入口构造函数：`mlir::hivm::createHIVMMapForallToBlocksPass()`。

**代码参考**

- 定义与说明：`bishengir/include/bishengir/Dialect/HIVM/Transforms/Passes.td:366-375`
- 实现：`bishengir/lib/Dialect/HIVM/Transforms/HIVMMapForallToBlocks.cpp:43-86`

## hivm-flatten-ops

**Pass 作用**

- 名称与位置：`HIVMFlattenOps` 在 `bishengir/include/bishengir/Dialect/HIVM/Transforms/Passes.td:377`，作用于 `func::FuncOp`。
- 作用：对实现了 `FlattenInterface` 的 HIVM 结构化算子进行“维度折叠”规范化。它根据每个算子的 `getFlattened(...)` 结果，使用 `memref.collapse_shape` 折叠输入和输出的形状，并克隆该算子到折叠后的维度空间；随后调用 `adjustTargetDimensions(...)` 调整目标维度，使算子在更低维、连贯的布局上工作。
- 限制：Host 函数不处理；若算子返回的是“身份折叠”（不需要折叠），则跳过该算子。

**实现细节**

- 入口与流程：`bishengir/lib/Dialect/HIVM/Transforms/FlattenOps.cpp`
  - 匹配所有 `HIVMStructuredOp`，强转为 `FlattenInterface`，调用 `getFlattened(FlattenOptions())` 获取折叠计划。
  - 对每个 `memref` 类型的操作数，按计划生成 `memref.collapse_shape`，建立 `IRMapping`。
  - 克隆原算子，替换为折叠后的操作数；调用其 `adjustTargetDimensions` 方法更新目标维度。
  - 将原算子替换为新算子。
- 依赖：算子需实现 `FlattenInterface`，并提供 `getFlattened`、`adjustTargetDimensions`。

**简单示例（概念示意）**

```mlir
// 运行前：某 HIVM 结构化算子在高维张量上操作
%src = memref.alloc() : memref<2x8x16xf32>   // 形状示例
%dst = memref.alloc() : memref<2x8x16xf32>
%op = hivm.hir.some_structured_op ins(%src) outs(%dst)

// 运行后：折叠维度为更低维（例如把 2x8 折叠成 16）
%src_c = memref.collapse_shape %src [[0, 1], [2]]
          : memref<2x8x16xf32> into memref<16x16xf32>
%dst_c = memref.collapse_shape %dst [[0, 1], [2]]
          : memref<2x8x16xf32> into memref<16x16xf32>
%op_flat = hivm.hir.some_structured_op ins(%src_c) outs(%dst_c)
// 算子内部通过 `adjustTargetDimensions(...)` 更新目标维度元数据
```

- 如果某个算子报告 `isIdentityCollapse()`，说明无需折叠，Pass 会跳过该算子。

**使用方式**

- 独立运行：

```bash
mlir-opt -hivm-flatten-ops input.mlir -o output.mlir
```

- 入口构造函数：`mlir::hivm::createFlattenOpsPass()`。

**代码参考**

- 定义与说明：`bishengir/include/bishengir/Dialect/HIVM/Transforms/Passes.td:377-381`
- 实现：`bishengir/lib/Dialect/HIVM/Transforms/FlattenOps.cpp:45-103`

## hivm-align-alloc-size

**Pass 作用**

- 名称与位置：`AlignAllocSize` 在 `bishengir/include/bishengir/Dialect/HIVM/Transforms/Passes.td:383`，作用于 `func::FuncOp`。
- 作用：为特定 HIVM 运算的局部缓冲分配（`memref.alloc`）进行“大小对齐”。当某些运算的访问要求按硬件单位对齐（如转置最后维、部分类型转换、排序），Pass 会：
  - 标注需要对齐的维度及字节数信息到相关操作数；
  - 向上游传播对齐信息直到根分配点（`memref.alloc`）；
  - 依据标注调整分配的形状（将维度提升到满足对齐的大小），并用 `memref.subview` 恢复原逻辑视图。

**实现流程**

- 入口与核心逻辑：`bishengir/lib/Dialect/HIVM/Transforms/AlignBuffer/HIVMAlignAllocSize.cpp`
  - 步骤1（标注）：扫描函数，针对以下操作生成对齐标注：
    - `VTransposeOp` 且为“最后维转置”，且未显式禁用对齐（属性 `disableAlign`）
    - `VCastOp` 的 `i32→i8`、`i16→i8` 情形
    - `VSortOp`
    - 对每个匹配操作，调用 `alignAllocSize(...)` 产生标注（维度索引与字节数），使用属性名 `hivm.alloc_align_dims` 与 `hivm.alloc_align_value_in_byte`。
  - 步骤2（传播）：运行传播模式，将对齐标注向上追踪到根 `memref.alloc`。
  - 步骤3（重写）：对带有对齐标注的 `memref.alloc`：
    - 计算每维按元素字节数换算后的最小对齐单位；
    - 以 `AlignUp` 将各维度提升到对齐大小；
    - 生成新的 `memref.alloc`（对齐后的形状）；
    - 生成 `memref.subview` 视图到原逻辑形状；
    - 替换旧的 `alloc`，并移除对齐标注。

**简单示例（概念示意）**

- 最后维转置需要对齐

```mlir
// 运行前
%a = memref.alloc(%m, %n) : memref<?x?xf16, #hivm.address_space<UB>>
%t = hivm.hir.vtranspose ins(%a) outs(%b) permutation = [0, 1] // 最后一维参与

// 运行后（核心逻辑）
%a_aligned = memref.alloc(%m_aligned, %n_aligned) : memref<?x?xf16, #hivm.address_space<UB>>
%a_view = memref.subview %a_aligned[%0, %0][%m, %n][1, 1] : memref<?x?xf16, ...>
// 原使用 %a 的位置替换为 %a_view
```

- 类型转换 `i32→i8` 需要对齐

```mlir
// 运行前
%src = memref.alloc(%n) : memref<?xi32, #hivm.address_space<UB>>
%dst = memref.alloc(%n) : memref<?xi8,  #hivm.address_space<UB>>
hivm.hir.vcast ins(%src) outs(%dst) round_mode = ...

// 运行后（核心逻辑）
%dst_aligned = memref.alloc(%n_aligned) : memref<?xi8, #hivm.address_space<UB>>
%dst_view    = memref.subview %dst_aligned[%0][%n][1] : memref<?xi8, ...>
// 使用处改用 %dst_view；`alloc_align_*` 标注被清理
```

**使用方式**

- 独立运行：

```bash
mlir-opt -hivm-align-alloc-size input.mlir -o output.mlir
```

- 构造函数：`mlir::hivm::createAlignAllocSizePass()`。

**代码参考**

- 定义与说明：`bishengir/include/bishengir/Dialect/HIVM/Transforms/Passes.td:383-395`
- 实现：`bishengir/lib/Dialect/HIVM/Transforms/AlignBuffer/HIVMAlignAllocSize.cpp:69-228`

## hivm-mark-stride-align

**Pass 作用**

- 名称与位置：`MarkStrideAlign` 在 `bishengir/include/bishengir/Dialect/HIVM/Transforms/Passes.td:397`，作用于 `func::FuncOp`。
- 作用：为 HIVM 结构化算子的本地缓冲（`UB` 空间）操作数自动标注“存储对齐”信息（维度与字节数），以便后续 `EnableStrideAlign` Pass 依据该标注对内存分配与布局进行实际对齐。主要解决当最后维不是连续访问时的步幅对齐需求。
- 条件与限制：
  - 仅处理实现了 `FlattenInterface` 且具备纯缓冲语义的 HIVM 结构化算子。
  - 仅针对本地缓冲（`UB`）的操作数标注。
  - 若是最后维转置（`VTransposeOp` 且 last-dim transpose），该场景通过 `AlignAllocSize` 处理，`MarkStrideAlign` 不再重复标注。

**实现细节**

- 入口与逻辑：`bishengir/lib/Dialect/HIVM/Transforms/AlignBuffer/MarkStrideAlign.cpp`
  - 跳过 Host 函数。
  - 对每个 `HIVMStructuredOp`：
    - 检查纯缓冲语义；过滤掉非 UB 操作数、所有 rank 为 0 的情况。
    - 调用 `getFlattened(FlattenOptions{checkMarkStride=true})` 获取“连续性”关联信息与折叠后的类型。
    - 计算“最后一个非连续维”（`getLastUnContinuousDim`）；并针对每个目标空间操作数（UB）进行维度调整（`adjustAlignDim`）后，标注该维的对齐字节数（硬件对齐字节，`getHWAlignBytes(memorySpace)`）。
    - 标注通过 `createAlignMarkOp(...)` 完成，最终写入对齐维和字节的属性。

**简单示例（概念示意）**

```mlir
// 运行前：最后维不是连续访问（stride != 1），需要存储对齐
%ub = memref.alloc() : memref<16x15xf32, #hivm.address_space<UB>>
%gm = memref.alloc() : memref<16x15xf32, #hivm.address_space<GM>>
%op = hivm.hir.vadd ins(%gm, %ub) outs(%ub)

// 运行后：为 %ub 标注存储对齐维和字节数（例如最后维）
annotation.mark %ub {
  hivm.alloc_align_dims = [1],
  hivm.alloc_align_value_in_byte = [<HW对齐字节>]
}
hivm.hir.vadd ins(%gm, %ub) outs(%ub)
```

- 随后 `EnableStrideAlign` 会读取这些标注，重分配缓冲并确保相应维度满足对齐。

**使用方式**

- 独立运行：

```bash
mlir-opt -hivm-mark-stride-align input.mlir -o output.mlir
```

- 通常与 `AlignAllocSize`、`EnableStrideAlign` 配合：先标注，再传播到根分配并执行对齐。

**代码参考**

- 定义与说明：`bishengir/include/bishengir/Dialect/HIVM/Transforms/Passes.td:397-407`
- 实现：`bishengir/lib/Dialect/HIVM/Transforms/AlignBuffer/MarkStrideAlign.cpp:43-190`

## hivm-enable-stride-align

**Pass 作用**

- 名称与位置：`EnableStrideAlign` 在 `bishengir/include/bishengir/Dialect/HIVM/Transforms/Passes.td:409`，作用于 `func::FuncOp`。
- 作用：启用并实现“步幅对齐”（stride align）标注的实际效果。它读取由 `MarkStrideAlign` 添加的对齐信息，规范并在整个函数内进行传播（向下到使用点、在操作数之间合并、向上到根分配），最终在 `memref.alloc/alloca` 处调整分配形状，必要时通过 `memref.subview` 保持原逻辑视图，然后清理标注。
- 处理范围：针对本地缓冲（`UB`）的操作数与其根分配；跳过 Host 函数。

**实现流程**

- 入口与核心逻辑：`bishengir/lib/Dialect/HIVM/Transforms/AlignBuffer/EnableStrideAlign.cpp`
  - 规范标注：`NormalizeAlignInfoPattern` 校验并排序 `hivm.stride_align_dims` 与 `hivm.stride_align_value_in_byte`，确保一致性。
  - 向上传播：`populatePropagateAlignUpToRootAllocationPattern(...)` 将对齐标注层层传回到根 `alloc`。
  - 向下传播：`AddAlignAnnotationMarkForAlloc` 在 `alloc` 结果上插入 `annotation.mark`；`PropagateAlignDownToLeafOperandsPattern` 将对齐标注沿 SSA 使用链传播到叶操作数。
  - 操作数间合并：`PropagateAlignAmongOperationOperands` 在一个算子的多个操作数之间做信息融合与补充，统一维度和字节的对齐需求。
  - 迭代收敛：循环执行“下传播→操作数合并→上传播”，比较前后 `alloc` 的对齐信息，直到稳定或达到迭代上限。
  - 实施对齐：`EnableAlignAllocation<AllocLikeOp>` 重写带对齐标注的分配：
    - 计算各维的对齐单元（按元素大小换算字节 → 元素数单位）
    - 生成对齐后的静态或半静态形状 `alloc`
    - 创建 `memref.subview` 恢复原逻辑子形状
    - 替换原分配并移除标注；若无需改变形状则只移除标注

**简单示例（概念示意）**

- 结合 `MarkStrideAlign` 标注后，启用对齐

```mlir
// 运行前（已由 MarkStrideAlign 添加了步幅对齐标注）
%ub = memref.alloc(%m, %n) : memref<?x?xf32, #hivm.address_space<UB>>
annotation.mark %ub {
  hivm.stride_align_dims = [1],
  hivm.stride_align_value_in_byte = [64]   // 举例：硬件要求最后维按 64B 对齐
}
%op = hivm.hir.vadd ins(%ub, %ub2) outs(%ub3)

// 运行后（EnableStrideAlign 应用重写）
%ub_aligned = memref.alloc(%m, %n_aligned) : memref<?x?xf32, #hivm.address_space<UB>>
%ub_view    = memref.subview %ub_aligned[%0, %0][%m, %n][1, 1] : memref<?x?xf32, ...>
%op = hivm.hir.vadd ins(%ub_view, %ub2) outs(%ub3)
// 标注被清理
```

- 若同一 `alloc` 被不同位置标注相同维但不同字节数，Pass 会统一、合并、排序；若冲突不可合并将报错并中止。

**使用方式**

- 独立运行：

```bash
mlir-opt -hivm-enable-stride-align input.mlir -o output.mlir
```

- 常与 `MarkStrideAlign`、`AlignAllocSize` 配合：先标注，迭代传播与收敛，最后在分配处实施对齐。

**代码参考**

- 定义与说明：`bishengir/include/bishengir/Dialect/HIVM/Transforms/Passes.td:409-424`
- 实现与关键模式：`bishengir/lib/Dialect/HIVM/Transforms/AlignBuffer/EnableStrideAlign.cpp:55-597`

## hivm-lift-lowest-stride

**Pass 作用**

- 名称与位置：`LiftLowestStride` 在 `bishengir/include/bishengir/Dialect/HIVM/Transforms/Passes.td:421`，作用于 `func::FuncOp`。
- 作用：当 HIVM 算子的操作数在最后一维不是连续（stride≠1）且也不是 size=1 时，通过“提升一个维度”来消除最低层面的非连续步幅问题。具体做法是对相关 `memref` 操作数追加一个大小为 1 的新维，并在步幅元数据中追加 1，使最后维变为连续，然后克隆或重建该算子以适配新的维度与属性。
- 适用场景：`VBrc`（广播）、`VReduce`（规约）、`VTranspose`（转置）、累积类（`VCumsum`/`VCumprod`）、`VMulextended`、`Copy` 以及一般元素级向量算子（通过统一模式注册）在 UB/GM 上的缓冲语义下。
- 限制：Host 函数不处理；零维 `memref` 跳过；若所有相关操作数的最后维已连续或为 1，则不改写。

**实现细节**

- 入口与逻辑：`bishengir/lib/Dialect/HIVM/Transforms/LiftLowestStride.cpp`
  - `createLiftedOperand` 使用 `memref.extract_strided_metadata` + `memref.reinterpret_cast` 为操作数追加一个 size=1 且 stride=1 的新维；静态形状时直接拼接形状与步幅，动态形状时从元数据抽取并拼接。
  - 各算子专属重写模式：
    - `VBrcOpLiftLowestStridePattern`：对 `src/dst` 追加维度并重建 `vbrc`，维持 `broadcast_dims`。
    - `VReduceOpLiftLowestStridePattern`：对 `src/dst`（以及可选 `indices`）追加维度并重建 `vreduce`，维持 `reduce_dims` 与算子属性。
    - `VTransposeOpLiftLowestStridePattern`：追加维度，并在 `permutation` 末尾追加新维索引。
    - `CopyOpLiftLowestStridePattern`：追加维度，并在存在 `collapse_reassociation` 时同步在末尾追加一个新分组。
    - `VMulextendedOp`、`VCumsumOp`、`VCumprodOp` 等：类似地对相关操作数追加维度并替换。
    - 通用元素级算子通过模板模式统一处理；若算子带有 `transpose` 属性，则在属性末尾追加新维索引。
  - 所有模式在“最后维已连续或为 1”时直接拒绝匹配。

**简单示例（概念示意）**

- 针对 `vreduce` 的操作数最后维非连续，提升维度以规整步幅

```mlir
// 运行前：最后维 stride != 1
%src = memref.alloc() : memref<?x32xf16, strided<[?, 16]>>
%dst = memref.alloc() : memref<?x1xf16,  strided<[?, 16]>>
hivm.hir.vreduce <sum> ins(%src) outs(%dst) reduce_dims = [1]

// 运行后：对 src/dst 都追加一个 size=1、stride=1 的新维
%src_lift = memref.reinterpret_cast %src to
             memref<?x32x1xf16, strided<[?, 16, 1]>>
%dst_lift = memref.reinterpret_cast %dst to
             memref<?x1x1xf16,  strided<[?, 16, 1]>>
hivm.hir.vreduce <sum> ins(%src_lift) outs(%dst_lift) reduce_dims = [1]
```

- 针对 `vtranspose`，同步扩展 `permutation`

```mlir
// 运行前
%a = memref.alloc() : memref<2x8xf32, strided<[?, 4]>>
%b = memref.alloc() : memref<8x2xf32, strided<[?, 4]>>
hivm.hir.vtranspose ins(%a) outs(%b) permutation = [1, 0]

// 运行后：追加新维，更新 permutation = [1, 0, 2]
%a_lift = memref.reinterpret_cast %a to memref<2x8x1xf32, strided<[?, 4, 1]>>
%b_lift = memref.reinterpret_cast %b to memref<8x2x1xf32, strided<[?, 4, 1]>>
hivm.hir.vtranspose ins(%a_lift) outs(%b_lift) permutation = [1, 0, 2]
```

**使用方式**

- 独立运行：

```bash
mlir-opt -hivm-lift-lowest-stride input.mlir -o output.mlir
```

- 入口构造函数：`mlir::hivm::createLiftLowestStridePass()`。

**代码参考**

- 定义与说明：`bishengir/include/bishengir/Dialect/HIVM/Transforms/Passes.td:421-431`
- 实现与模式：`bishengir/lib/Dialect/HIVM/Transforms/LiftLowestStride.cpp:43-472`
- 测试用例：`bishengir/test/Dialect/HIVM/hivm-lift-lowest-stride.mlir:118-141`

## hivm-reduce-rank-subview

**Pass 作用**

- 名称与位置：`ReduceRankSubview` 在 `bishengir/include/bishengir/Dialect/HIVM/Transforms/Passes.td:438`，作用于 `func::FuncOp`。
- 作用：当某些维度在所有相关操作数上都是长度为 1 时，使用 `memref.subview` 将这些维度“折叠”并降低秩（rank）。它生成带秩降低后的 `subview`，同时维持正确的步幅布局，并对算子的维度属性（如 `broadcast_dims`、`reduce_dims`）进行相应调整。
- 适用场景：元素级多元算子（`isElemwiseNaryOp()`）、`CopyOp`、`LoadOp`、`StoreOp`、`VBrcOp`（广播）、`VReduceOp`（规约），且必须为缓冲语义（`memref`）而非张量语义。

**实现细节**

- 入口与逻辑：`bishengir/lib/Dialect/HIVM/Transforms/ReduceRankSubview.cpp`
  - 判定可降秩维度：
    - 通用元素算子：收集所有操作数的 `memref` 形状；若某维在所有操作数上都是 1，并且不是“全部维都将被删除”的退化情况，则该维可删除。
    - 专用算子：
      - `VBrcOp`：在避免删除 `broadcast_dims` 的前提下，删除源与目的同时为 1 的维度，并调整新的 `broadcast_dims`。
      - `VReduceOp`：在避免删除 `reduce_dims` 的前提下，删除源与目的同时为 1 的维度，并调整新的 `reduce_dims`；如有 `indices`，同步处理。
  - 构造降秩视图：
    - 利用 `memref.extract_strided_metadata` 获取偏移/尺寸/步幅；对可删除维设置 `subview` 的 `size=1`，并构造降秩后的 `MemRefType`，同时删除对应的步幅分量。
    - 替换操作数为新的 `subview` 结果，克隆或重写算子，并更新维度属性。
- 零维或秩为 1 的情况直接跳过。

**简单示例（概念示意）**

- 对元素级算子降秩

```mlir
// 运行前：所有操作数在维度1上都是 size=1
%a = memref.alloc() : memref<16x1x32xf32, strided<[?, 4, 1]>>
%b = memref.alloc() : memref<16x1x32xf32, strided<[?, 4, 1]>>
%c = memref.alloc() : memref<16x1x32xf32, strided<[?, 4, 1]>>
hivm.hir.vadd ins(%a, %b) outs(%c)

// 运行后：删除维度1，使用 subview 降秩，并保持 strides 正确
%a_r = memref.subview %a [0, 0, 0] [16, 1, 32] [1, 1, 1]
       : memref<16x1x32xf32, ...> to memref<16x32xf32, strided<[?, 1]>>
%b_r = memref.subview %b [...] : ... to memref<16x32xf32, strided<[?, 1]>>
%c_r = memref.subview %c [...] : ... to memref<16x32xf32, strided<[?, 1]>>
hivm.hir.vadd ins(%a_r, %b_r) outs(%c_r)
```

- 对 `vbrc` 的降秩与属性更新

```mlir
// 运行前：某些非广播维都是 size=1
%src = memref.alloc() : memref<1x32x1xi16, strided<[?, 16, 1]>>
%dst = memref.alloc() : memref<1x32x1xi16, strided<[?, 16, 1]>>
hivm.hir.vbrc ins(%src) outs(%dst) broadcast_dims = [0]

// 运行后：删除维度2（size=1）并调整 broadcast_dims
%src_r = memref.subview %src [...] : ... to memref<1x32xi16, strided<[?, 16]>>
%dst_r = memref.subview %dst [...] : ... to memref<1x32xi16, strided<[?, 16]>>
hivm.hir.vbrc ins(%src_r) outs(%dst_r) broadcast_dims = [0]
```

- 对 `vreduce` 的降秩与属性更新

```mlir
// 运行前：所有相关维都是 size=1，且不在 reduce_dims
%src = memref.alloc() : memref<1x32x1xf16, strided<[?, 16, 1]>>
%dst = memref.alloc() : memref<1x1x1xf16, strided<[?, 16, 1]>>
hivm.hir.vreduce <sum> ins(%src) outs(%dst) reduce_dims = [1]

// 运行后：删除维度2，reduce_dims 无需变化或按需重新映射
%src_r = memref.subview %src [...] : ... to memref<1x32xf16, strided<[?, 16]>>
%dst_r = memref.subview %dst [...] : ... to memref<1x1xf16, strided<[?, 16]>>
hivm.hir.vreduce <sum> ins(%src_r) outs(%dst_r) reduce_dims = [1]
```

**使用方式**

- 独立运行：

```bash
mlir-opt -hivm-reduce-rank-subview input.mlir -o output.mlir
```

- 入口构造函数：`mlir::hivm::createReduceRankSubviewPass()`。

**代码参考**

- 定义与说明：`bishengir/include/bishengir/Dialect/HIVM/Transforms/Passes.td:438-448`
- 实现与模式：`bishengir/lib/Dialect/HIVM/Transforms/ReduceRankSubview.cpp:42-397`

## hivm-inline-otf-broadcast

**Pass 作用**

- 名称与位置：`InlineOTFBroadcast` 在 `bishengir/include/bishengir/Dialect/HIVM/Transforms/Passes.td:445`，作用于 `func::FuncOp`。
- 作用：将“在线广播”（OTF，on-the-fly）`hivm.hir.vbrc` 的结果直接内联到其下游可支持广播的算子中，移除单独的 `vbrc` 依赖，并把广播维度信息转移为下游算子的 `broadcast` 属性。这能减少中间张量、简化图结构，并利于后续融合。
- 条件与限制：
  - 仅处理张量语义（`hasPureTensorSemantics()`）的 `vbrc`。
  - 只支持单一广播轴（`broadcast_dims.size() == 1`）。
  - 目标元素类型不能是 `i64` 或 `i1`。
  - 下游用户需满足广播支持白名单或具备 `BroadcastableOTF` trait（且为二元元素级算子、`HIVMStructuredOp`）。

**实现细节**

- 入口与逻辑：`bishengir/lib/Dialect/HIVM/Transforms/InlineOTFBroadcast.cpp`
  - 检查 `vbrc` 的 `broadcast_dims`，计算轴类别（最后轴或非最后轴）。
  - 对 `vbrc` 的所有用户进行筛选：
    - 最后轴时，要求用户在白名单中（如 `VAdd/VMul/VMax/VMin/VSub/VDiv/VAnd/VOr/VNot/VAbs/VLn/VRelu/VExp/VRsqrt/VSqrt`；`VAbs` 对 `i16/i32` 不支持）。
    - 非最后轴时，要求用户具有 `BroadcastableOTF` trait，且为二元元素级且 `HIVMStructuredOp`。
  - 对符合条件的用户：
    - 将用户的 `broadcast` 属性添加/合并该轴
    - 将用户中使用 `vbrc`结果的位置替换为 `vbrc` 的源操作数
  - 若至少一个用户被处理，返回成功；否则不改写。

**简单示例（概念示意）**

```mlir
// 运行前：vbrc 的结果被用作二元元素级算子的输入
%brc = hivm.hir.vbrc ins(%a : tensor<1x128xf32>) outs(%b : tensor<5x128xf32>) broadcast_dims = [0]
%ret = hivm.hir.vmul ins(%brc, %c : tensor<5x128xf32>, tensor<5x128xf32>) outs(%d : tensor<5x128xf32>)

// 运行后：将广播内联到 vmul，去除对 %brc 的依赖
%ret = hivm.hir.vmul ins(%a, %c : tensor<1x128xf32>, tensor<5x128xf32>) outs(%d : tensor<5x128xf32>)
  attributes { broadcast = [0] }
```

- 对最后轴广播的白名单算子会直接内联；其他满足 trait 的算子也会内联并合并其已有的 `broadcast` 属性。

**使用方式**

- 独立运行：

```bash
mlir-opt -hivm-inline-otf-broadcast input.mlir -o output.mlir
```

- 构造函数：`mlir::hivm::createInlineOTFBroadcastPass()`。

**代码参考**

- 定义与说明：`bishengir/include/bishengir/Dialect/HIVM/Transforms/Passes.td:445-454`
- 实现：`bishengir/lib/Dialect/HIVM/Transforms/InlineOTFBroadcast.cpp:85-163`

## hivm-inline-load-copy

**Pass 作用**

- 名称与位置：`InlineLoadCopy` 在 `bishengir/include/bishengir/Dialect/HIVM/Transforms/Passes.td:452`，作用于 `func::FuncOp`。
- 作用：当一个 `hivm.copy` 的源缓冲恰好来自同一个 `hivm.load` 写入，且该源仅被两个用户使用（该 `load` 与该 `copy`），则将这对“先 load 到中间 UB，再 copy 到目标 UB”的模式内联为一次直接 `hivm.load` 到目标缓冲，移除中间缓冲与 `copy`，同时保留 `load` 的填充与边界属性。
- 限制：仅在缓冲语义（`memref`）下处理；必须能唯一匹配到写入 `copy` 源的那个 `LoadOp`，且源值的使用数等于 2。

**实现细节**

- 入口与逻辑：`bishengir/lib/Dialect/HIVM/Transforms/InlineLoadCopy.cpp`
  - 模式匹配：
    - 找到 `copyOp.getSrc()` 的所有用户，统计使用次数并定位 `LoadOp`。
    - 要求该 `LoadOp` 的目标 `dst` 正是 `copy` 的源缓冲（保证确实是“把 GM load 到这个 UB，再从这个 UB copy 到另一个 UB”）。
    - 要求源值只有两个用户，确保该 UB 不被其他操作使用。
  - 改写：
    - 用 `LoadOp` 替换 `CopyOp`，将 `load` 的 `src` 与 `copy` 的 `dst` 组合为“直接加载到目标缓冲”，并继承 `load` 的 `pad_mode`、`pad_value`、左右 padding 等属性。
    - 删除原 `LoadOp`。

**简单示例（概念示意）**

```mlir
// 运行前：GM->UB 的 load，随后 UB->UB 的 copy
%ub1 = memref.alloc() : memref<32x32xf16, #hivm.address_space<UB>>
%ub2 = memref.alloc() : memref<32x32xf16, #hivm.address_space<UB>>
hivm.hir.load  ins(%gm : memref<32x32xf16, #hivm.address_space<GM>>)
               outs(%ub1 : memref<32x32xf16, #hivm.address_space<UB>>) pad_mode = ...
hivm.hir.copy  ins(%ub1) outs(%ub2)

// 运行后：直接把 load 的结果写入目标缓冲 %ub2，移除中间缓冲写入与 copy
hivm.hir.load  ins(%gm : memref<32x32xf16, #hivm.address_space<GM>>)
               outs(%ub2 : memref<32x32xf16, #hivm.address_space<UB>>) pad_mode = ...
```

- 注意：若 `copy` 源还有其他用户或找不到匹配的 `LoadOp`，Pass 不会改写。

**使用方式**

- 独立运行：

```bash
mlir-opt -hivm-inline-load-copy input.mlir -o output.mlir
```

- 构造函数：`mlir::hivm::createInlineLoadCopyPass()`。

**代码参考**

- 定义与说明：`bishengir/include/bishengir/Dialect/HIVM/Transforms/Passes.td:452-460`
- 实现：`bishengir/lib/Dialect/HIVM/Transforms/InlineLoadCopy.cpp:45-97`

## hivm-init-entry-kernel

**Pass 作用**

- 名称与位置：`InitEntryKernel` 在 `bishengir/include/bishengir/Dialect/HIVM/Transforms/Passes.td:459`，作用于 `func::FuncOp`。
- 作用：对设备端入口函数（device entry kernel）进行初始化，在函数首部插入 `hivm.set_mask_norm` 指令以设定面罩规范（mask normalization），确保后续 HIVM 运算在统一的掩码状态下执行。
- 条件：仅当函数被识别为设备入口（`isDeviceEntry(funcOp)`）时生效；Host 或非入口函数不处理。

**实现细节**

- 入口与逻辑：`bishengir/lib/Dialect/HIVM/Transforms/InitEntryKernel.cpp`
  - 判定入口：使用 `hacc::utils::isDeviceEntry(funcOp)`。
  - 插入位置：设置插入点为入口函数首个基本块的起始处。
  - 插入指令：`hivm::SetMaskNormOp`，无操作数与结果，作为设备端执行时的状态初始化。

**简单示例（概念示意）**

```mlir
// 运行前：设备入口函数尚未做掩码初始化
func.func @kernel_entry(%arg0: memref<...>) attributes {hacc.device_entry} {
  // 原始主体
  ...
}

// 运行后：在入口函数首部插入 hivm.set_mask_norm
func.func @kernel_entry(%arg0: memref<...>) attributes {hacc.device_entry} {
  hivm.hir.set_mask_norm
  ...
}
```

**使用方式**

- 独立运行：

```bash
mlir-opt -hivm-init-entry-kernel input.mlir -o output.mlir
```

- 构造函数：`mlir::hivm::createInitEntryKernelPass()`。

**代码参考**

- 定义与说明：`bishengir/include/bishengir/Dialect/HIVM/Transforms/Passes.td:459-462`
- 实现：`bishengir/lib/Dialect/HIVM/Transforms/InitEntryKernel.cpp:44-56`

## hivm-inline-fixpipe

**Pass 作用**

- 名称与位置：`InlineFixpipe` 在 `bishengir/include/bishengir/Dialect/HIVM/Transforms/Passes.td:465`，作用域为模块或函数级（定义中未限定 `func::FuncOp`，实际按实现在函数级运行）。
- 作用：围绕 `hivm.fixpipe` 与矩阵乘加（`hivm.mmadL1`/`hivm.batch_mmadL1`）结果的常见后处理链进行内联与插入：
  - 插入：在满足条件的 `mmad` 结果链路上自动插入 `fixpipe`，将后续的量化、激活、存储、切片等处理统一融合到 `fixpipe`。
  - 内联：当已有 `fixpipe` 时，尝试与后续操作内联，如融合量化（`vcast`）、激活（`vrelu`）、存储（`store`），并对 `extract_slice`/`insert_slice`、`scf.for` 中的 `yield` 做结构调整，使 `fixpipe` 出现在更合适的位置。
  - 打印路径：对设备端打印（`hivm.print`）且打印的是 `mmad` 结果并在 `scf.for` 中被 `yield` 的情况，自动在打印前插入 `fixpipe`。

**实现细节**

- 入口与逻辑：`bishengir/lib/Dialect/HIVM/Transforms/InlineFixpipe.cpp`
  - 插入模式 `InsertFixpipeOpPattern<MmadL1Op/BatchMmadL1Op>`：
    - 若 `mmad` 将分解为“`mmad` + `vadd`”则暂不插入（等待分解后再插）。
    - 若沿单链路追踪用户是“本地 matmul 初始化并唯一用户”，则无需插入（直接在本地使用）。
    - 通过 `getInsertPoint` 找到在 `scf.for` 链路中合适的插入点（可能递归追踪到 `yield` 的父），在其后插入 `fixpipe` 并用空 `tensor.empty` 初始化目标，随后替换原值用户。
  - 内联模式 `InlineFixpipeOpPattern`：
    - 匹配 `FixpipeOp` 且结果仅有一个用户，尝试：
      - 与 `vcast` 量化融合：根据 `f32→f16/bf16`、`i32→i8` 等确定 `pre_quant` 模式，生成新的 `fixpipe` 并替换 `cast`。
      - 与 `vrelu` 激活融合：设置 `pre_relu` 属性并移除 `relu`。
      - 与 `store` 融合：将 `fixpipe` 的结果落地到目标 `memref`，替换 `store` 为 `fixpipe`。
      - 与 `tensor.extract_slice`/`insert_slice` 位置交换：调整顺序以更好地融合。
      - 从 `scf.for` 中抽出：当 `fixpipe` 结果被 `yield`，将其移出循环，在循环结果上插入新的 `fixpipe`。
  - 打印路径 `InsertFixpipeForDevicePrint`：
    - 当 `hivm.debug` 的类型为 `print`，且打印的值溯源到 `mmad`/`batch_mmadL1`，并且该值被 `scf.for` 的 `yield` 使用，插入 `fixpipe` 并令打印使用 `fixpipe` 的结果。

**简单示例（概念示意）**

- 在 `mmadL1` 之后插入并融合量化

```mlir
// 运行前
%init = tensor.empty() : tensor<16x16xf32>
%mm = hivm.hir.mmadL1 ins(...) outs(%init)
%cast = hivm.hir.vcast ins(%mm : tensor<16x16xf32>) outs(%dst : tensor<16x16xf16>)

// 运行后：插入 fixpipe 并内联量化
%fix_init = tensor.empty() : tensor<16x16xf16>
%fix = hivm.hir.fixpipe ins(%mm) outs(%fix_init) pre_quant = F322F16
// 移除 vcast，所有使用者改用 %fix
```

- 与 `store` 融合为直接落地到 `memref`

```mlir
// 运行前
%t = hivm.hir.fixpipe ins(%mm) outs(%tmp) ...
hivm.hir.store ins(%t) outs(%ub) pad_mode = ...

// 运行后
hivm.hir.fixpipe ins(%mm) outs(%ub) ...  // 直接输出到目标
```

- 从循环中移出以避免在循环体内重复执行

```mlir
%init = tensor.empty()
%res = scf.for iter_arg(%arg = %init) {
  %t = hivm.hir.mmadL1 ins(...) outs(%arg)
  scf.yield %t
}
%fix_init = tensor.empty()
%fix = hivm.hir.fixpipe ins(%res) outs(%fix_init) ...
```

**使用方式**

- 独立运行：

```bash
mlir-opt -hivm-inline-fixpipe input.mlir -o output.mlir
```

- 构造函数：`mlir::hivm::createInlineFixpipePass()`。

**代码参考**

- 定义与说明：`bishengir/include/bishengir/Dialect/HIVM/Transforms/Passes.td:465-473`
- 实现与关键模式：`bishengir/lib/Dialect/HIVM/Transforms/InlineFixpipe.cpp:52-547`

## hivm-tile-batchmm-into-loop

**Pass 作用**

- 名称与位置：`TileBatchMMIntoLoop` 在 `bishengir/include/bishengir/Dialect/HIVM/Transforms/Passes.td:472`，作用于 `func::FuncOp`。
- 作用：将批量矩阵乘（`hivm.batch_mmadL1`）及其紧随的后处理链（尤其 `hivm.fixpipe` 与特定 `tensor.extract_slice` 形状操作）重写为“外层批次循环 + 非批 `hivm.mmadL1` + 对应的 `fixpipe`”。通过沿用提取-插入切片，按批次迭代处理，从而降低单次算子的维度复杂度并获得更好的调度与融合机会。
- 限制与匹配条件：
  - 当前仅支持输出为 3D 张量（形如 `[batch, M, N]`）的 `batch_mmadL1`，且与后续 `fixpipe` 在同一基本块。
  - 在 `batch_mmadL1` 到 `fixpipe` 之间仅允许存在满足约束的 `tensor.extract_slice` 链（保持批维在首维、offset=0/stride=1/size=batch\_dim）；不满足则拒绝改写。

**实现细节**

- 入口与逻辑：`bishengir/lib/Dialect/HIVM/Transforms/TileBatchMMIntoLoop.cpp`
  - 收集使用链：`getBatchSingleUseChain` 验证 `batch_mmadL1` 的唯一用户链条是否由合法的 `extract_slice` 序列连接到 `fixpipe`。
  - 提取/插入批次切片：
    - `extractTensorValueWithoutBatch` 从 `[b, m, n]` 的张量中按循环索引提取单批的 `[m, n]` 切片；
    - `insertTensorValueWithoutBatch` 将每次迭代的结果插入到累积张量的对应批索引位置。
  - 重写核心：
    - 迭代批次索引，替换为非批 `hivm.mmadL1`（`RankedTensorType` 去掉批维），并按原链路形状调整（`rewriteMatrixCShapeChange` 对后续 `extract_slice` 进行匹配性重建）。
    - 对应地重写 `fixpipe`：
      - 若目标是 `memref`，使用 `subview + collapse_shape` 取得非批目标视图，并在迭代内执行 `fixpipe`；
      - 若目标是张量，将循环迭代参数作为目标，并在每次迭代中执行 `fixpipe` 后用 `insert_slice` 汇总。
  - 最终替换：若原 `fixpipe` 返回张量，循环结果替代原返回；随后删除旧的 `batch_mmadL1` 与相关链路。

**简单示例（概念示意）**

```mlir
// 运行前：批量矩阵乘 + 形状处理 + fixpipe
%C = tensor.empty() : tensor<4xM x N xf16>
%bm = hivm.hir.batch_mmadL1 ins(%A: tensor<4xKxMxf16>, %B: tensor<4xKxNxf16>) outs(%C) ...
%sl = tensor.extract_slice %bm[0, 0, 0] [4, M, N] [1, 1, 1] : tensor<4xMxNxf16> to tensor<4xMxNxf16>
%fx = hivm.hir.fixpipe ins(%sl) outs(%C) ...

// 运行后：外层批次循环 + 非批次 mmad + 对应 fixpipe
%init = tensor.empty() : tensor<4xMxNxf16>
scf.for %i = %c0 to %c4 step %c1 iter_args(%acc = %init) {
  %A_i = tensor.extract_slice %A[%i, 0, 0] [1, K, M] [1, 1, 1] : ... to tensor<KxMxf16>
  %B_i = tensor.extract_slice %B[%i, 0, 0] [1, K, N] [1, 1, 1] : ... to tensor<KxNxf16>
  %C_i = tensor.empty() : tensor<MxNxf16>
  %mm = hivm.hir.mmadL1 ins(%A_i, %B_i) outs(%C_i) ...
  %fix = hivm.hir.fixpipe ins(%mm) outs(%C_i) ...
  %acc2 = tensor.insert_slice %fix into %acc[%i, 0, 0] [1, M, N] [1, 1, 1]
  scf.yield %acc2
}
```

**使用方式**

- 独立运行：

```bash
mlir-opt -hivm-tile-batchmm-into-loop input.mlir -o output.mlir
```

- 构造函数：`mlir::hivm::createTileBatchMMIntoLoopPass()`。

**代码参考**

- 定义与说明：`bishengir/include/bishengir/Dialect/HIVM/Transforms/Passes.td:472-476`
- 实现：`bishengir/lib/Dialect/HIVM/Transforms/TileBatchMMIntoLoop.cpp:67-392`

## hivm-lift-zero-rank

**Pass 作用**

- 名称与位置：`LiftZeroRank` 在 `bishengir/include/bishengir/Dialect/HIVM/Transforms/Passes.td:478`，作用于 `func::FuncOp`。
- 作用：当 HIVM 结构化算子的某些缓冲操作数是零维 `memref`（rank=0）时，将其“提升”为一维大小为 1 的 `memref`，使算子形状处理统一并避免零维特殊分支。实现方式是用 `memref.expand_shape` 将 `memref<?>` 扩展为 `memref<1x?>`（对于标量缓冲则变为 `memref<1xT>`）。
- 适用场景：仅在缓冲语义（`memref`）下运行；张量语义不处理。

**实现细节**

- 入口与逻辑：`bishengir/lib/Dialect/HIVM/Transforms/LiftZeroRank.cpp`
  - 遍历每个 `HIVMStructuredOp`，收集其操作数中 rank=0 的 `memref`。
  - 对每个零维操作数调用 `memref.expand_shape`，目标形状为 `[1]`，并将操作数替换为新的展开结果。
  - 若不存在零维操作数则跳过该算子。

**简单示例（概念示意）**

```mlir
// 运行前：元素级加法的一个操作数为零维 memref（标量缓冲）
%scalar = memref.alloc() : memref<f32>            // rank=0
%vec    = memref.alloc() : memref<16xf32>         // rank=1
hivm.hir.vadd ins(%scalar, %vec) outs(%vec)

// 运行后：将零维缓冲提升为 1 维大小为 1 的缓冲，便于统一广播/步幅处理
%scalar1d = memref.expand_shape %scalar [[0]] : memref<f32> into memref<1xf32>
hivm.hir.vadd ins(%scalar1d, %vec) outs(%vec)
```

- 对其他 HIVM 结构化算子也同理；一旦提升，后续 Pass（如广播、步幅对齐等）能更一致地处理这些操作数。

**使用方式**

- 独立运行：

```bash
mlir-opt -hivm-lift-zero-rank input.mlir -o output.mlir
```

- 构造函数：`mlir::hivm::createLiftZeroRankPass()`。

**代码参考**

- 定义与说明：`bishengir/include/bishengir/Dialect/HIVM/Transforms/Passes.td:478-481`
- 实现：`bishengir/lib/Dialect/HIVM/Transforms/LiftZeroRank.cpp:36-99`

## hivm-insert-load-store-for-mix-cv

**Pass 作用**

- 名称与位置：`InsertLoadStoreForMixCV` 在 `bishengir/include/bishengir/Dialect/HIVM/Transforms/Passes.td:484`，作用于 `func::FuncOp`。
- 作用：在“混合 Cube/Vector（Mix CV）”函数中，保证数据在不同核心类型之间流动时具备明确的缓冲边界，通过插入 `hivm.store` 与 `hivm.load` 将数据在 GM/UB 间落地与再加载，避免隐式跨核心或非连续内存访问导致的问题。具体包含：
  - 在“store-like → vector/cube”之间插入 `load`；
  - 在“vector → load”之间插入 `store`；
  - 在“vector ↔ cube”之间同时插入 `store` 和 `load`；
  - 针对特殊场景（`scf.for` 的 gather/load、`bufferization.to_tensor` 的隐式最后轴转置、`tensor.collapse_shape` 的潜在非连续 reshape）插入合适的 `store/load` 序列；
  - 对 `scf.for` 的 `yield` 值在张量路径上插入 `store`，以确保下一步 `load` 有明确来源。

**实现细节**

- 入口与逻辑：`bishengir/lib/Dialect/HIVM/Transforms/InsertLoadStoreForMixCV.cpp`
  - 通用插入器：`insertLoadStoreOp(...)` 接收一组待替换的操作数位置，按模式选择插入 `load`、`store`、或 `store+load`，并将消费者操作数指向新值。
  - 模式集：
    - `InsertLoadOpBetweenStoreLikeAndVectorOrCube<OpType>`：若某算子的操作数追溯到 `FixpipeOp` 或 `StoreOp`，在该操作数后插入 `load`。
    - `InsertStoreOpBetweenVectorAndLoad<OpType>`：若 `LoadOp` 的操作数追溯到某向量算子 `OpType`，在该操作数前插入 `store`。
    - `InsertLoadStoreOpBetweenVectorAndCube<OpType>`：若 `MmadL1Op` 的某操作数追溯到 `OpType`（向量算子或特殊 IR），在两者之间插入 `store+load`。
    - 特化场景：
      - `scf.for` 带 `ExtractedLoadOrStore`：循环实现离散加载，循环后对接 `mmadL1` 时插入 `store+load`。
      - `bufferization.to_tensor` 带 `MayImplicitTransposeWithLastAxis` 或 `gather_load` 属性：在接 `mmadL1` 前插入 `store+load`，让转置在向量侧发生。
      - `tensor.collapse_shape` 带 `maybeUnCollapsibleReshape` 属性：折叠可能引入非连续，接 `mmadL1` 前插入 `store+load`。
    - `InsertStoreForSCFYield`：若 `scf.for` 的 `yield` 值被 `LoadOp` 消费且不是 `Fixpipe/Store` 的结果，在 `yield` 前插入 `store`，并以循环迭代参数作为 `insertInit`。

**简单示例（概念示意）**

- 在向量算子和后续加载之间插入 `store`

```mlir
// 运行前
%tmp = hivm.hir.vadd ins(%a, %b) outs(%c)            // 产生 UB 数据
%x   = hivm.hir.load ins(%c) outs(%t)                // 后续将要加载

// 运行后
%tmp = hivm.hir.vadd ins(%a, %b) outs(%c)
%c_gm = hivm.hir.store ins(%c) outs(%gm_tmp)         // 显式落地
%x    = hivm.hir.load ins(%gm_tmp) outs(%t)          // 再加载
```

- 在 `mmadL1` 前对来自向量路径的数据插入 `store+load`

```mlir
// 运行前
%v = hivm.hir.vtranspose ins(%u) outs(%v2)
%mm = hivm.hir.mmadL1 ins(%v2, %B) outs(%C)

// 运行后
%v = hivm.hir.vtranspose ins(%u) outs(%v2)
%gm_tmp = hivm.hir.store ins(%v2) outs(%gm_buf)      // GM/合适空间落地
%l1_tmp = hivm.hir.load  ins(%gm_buf) outs(%ub_buf)  // 以 UB 连续视图再加载
%mm = hivm.hir.mmadL1 ins(%l1_tmp, %B) outs(%C)
```

- 对 `scf.for` 的 `yield` 插入 `store`

```mlir
%res = scf.for iter_args(%arg = %init) {
  %x = hivm.hir.vadd ins(%arg, %b) outs(%tmp)
  // 运行后：在 yield 前插入 store，让下一步的 load 有来源
  %tmp_gm = hivm.hir.store ins(%tmp) outs(%gm_tmp)
  scf.yield %gm_tmp
}
%y = hivm.hir.load ins(%res) outs(%ub)
```

**使用方式**

- 独立运行：

```bash
mlir-opt -hivm-insert-load-store-for-mix-cv input.mlir -o output.mlir
```

- 构造函数：`mlir::hivm::createInsertLoadStoreForMixCVPass()`。

**代码参考**

- 定义与说明：`bishengir/include/bishengir/Dialect/HIVM/Transforms/Passes.td:484-495`
- 实现与关键模式：`bishengir/lib/Dialect/HIVM/Transforms/InsertLoadStoreForMixCV.cpp:55-495`

## hivm-insert-infer-workspace-size-func

**Pass 作用**

- 名称与位置：`InsertInferWorkSpaceSizeFunc` 在 `bishengir/include/bishengir/Dialect/HIVM/Transforms/Passes.td:490`，作用于 `func::FuncOp`。
- 作用：在设备函数旁生成一个 Host 侧“工作空间大小推断”回调函数。该函数返回该设备函数所需的总工作空间字节数，便于运行时在调用前正确分配和管理工作空间内存。
- 前提：函数中已存在 `memref_ext.alloc_workspace` 的工作空间分配（通常由“计划工作空间”阶段生成）。

**实现细节**

- 入口与逻辑：`bishengir/lib/Dialect/HIVM/Transforms/InsertInferWorkSpaceSizeFunc.cpp`
  - 收集：遍历函数内的 `bishengir::memref_ext::AllocWorkspaceOp`。
  - 计算：对每个工作空间片段，检查静态 `offset` 和静态形状，按元素位宽/字节求片段大小，并加上偏移，取所有片段末端最大值作为总字节数。
  - 生成：创建一个新的 `func.func`（Host 函数），类型为 `() -> index`，返回常量的总字节数。
  - 标注：将该函数标记为 Host，并设置函数类型为 `kInferWorkspaceShapeFunction`；函数名通过 `hacc::constructHostFunctionName` 基于设备函数名派生。

**简单示例（概念示意）**

```mlir
// 运行前：设备函数包含工作空间片段分配
func.func @kernel_dev(...) {
  %ws0 = memref_ext.alloc_workspace [%c0] : memref<1024xi8>
  %ws1 = memref_ext.alloc_workspace [%c512] : memref<2048xi8>
  ...
}

// 运行后：生成 Host 回调，返回总工作空间字节数
func.func @kernel_dev(...) { ... }

func.func @kernel_dev__infer_workspace_size() attributes {hacc.host, hacc.host_func_type = "infer_workspace_size"} {
  %size = arith.constant 2560 : index  // 例：max(offset+size) = max(0+1024, 512+2048) = 2560
  return %size : index
}
```

- 该回调通常在 Triton 编译启用且自动工作空间管理开启时插入，供运行时查询并分配工作空间。

**使用方式**

- 独立运行：

```bash
mlir-opt -hivm-insert-infer-workspace-size-func input.mlir -o output.mlir
```

- 构造函数：`mlir::hivm::createInsertInferWorkSpaceSizeFuncPass()`。

**代码参考**

- 定义与说明：`bishengir/include/bishengir/Dialect/HIVM/Transforms/Passes.td:490-497`
- 实现：`bishengir/lib/Dialect/HIVM/Transforms/InsertInferWorkSpaceSizeFunc.cpp:66-175`

## hivm-bind-workspace-arg

**Pass 作用**

- 名称与位置：`BindWorkSpaceArg` 在 `bishengir/include/bishengir/Dialect/HIVM/Transforms/Passes.td:503`，作用于 `func::FuncOp`。
- 作用：将函数的“工作空间”形参绑定到函数体内的每个 `memref_ext.alloc_workspace` 操作上，使这些工作空间分配点明确地引用同一个入口工作空间参数。这确保运行时分配的工作空间能在设备函数内部被所有相关分配正确使用。
- 前提：函数具备一个工作空间参数（`hacc::KernelArgType::kWorkspace`）；若不存在且有需要绑定的分配点则报错。

**实现细节**

- 入口与逻辑：`bishengir/lib/Dialect/HIVM/Transforms/BindWorkSpaceArg.cpp`
  - 读取函数的工作空间 `BlockArgument`（通过 `hacc::utils::getBlockArgument(funcOp, kWorkspace)`）。
  - 遍历所有 `bishengir::memref_ext::AllocWorkspaceOp`：
    - 若其尚未设置 `workspace_arg`，则将函数的工作空间形参写入该 op 的可变属性；
    - 若函数没有工作空间形参且需要绑定，报错并中止。
  - 成功则不改变 IR 结构，仅补全关联。

**简单示例（概念示意）**

```mlir
// 运行前：工作空间参数未绑定到分配点
func.func @kernel(%ws: memref<?xi8> {hacc.workspace}, %a: memref<...>) {
  %w0 = memref_ext.alloc_workspace [%c0] : memref<1024xi8>   // 未绑定 workspace_arg
  %w1 = memref_ext.alloc_workspace [%c512] : memref<2048xi8> // 未绑定 workspace_arg
  ...
}

// 运行后：每个分配点都绑定到入口工作空间参数 %ws
func.func @kernel(%ws: memref<?xi8> {hacc.workspace}, %a: memref<...>) {
  %w0 = memref_ext.alloc_workspace [%c0] workspace(%ws) : memref<1024xi8>
  %w1 = memref_ext.alloc_workspace [%c512] workspace(%ws) : memref<2048xi8>
  ...
}
```

- 若函数没有 `%ws` 但存在需要绑定的 `alloc_workspace`，Pass 会在该 op 上报错并终止。

**使用方式**

- 独立运行：

```bash
mlir-opt -hivm-bind-workspace-arg input.mlir -o output.mlir
```

- 构造函数：`mlir::hivm::createBindWorkSpaceArgPass()`。

**代码参考**

- 定义与说明：`bishengir/include/bishengir/Dialect/HIVM/Transforms/Passes.td:503-510`
- 实现：`bishengir/lib/Dialect/HIVM/Transforms/BindWorkSpaceArg.cpp:61-85`

## hivm-bind-sync-block-lock-arg

**Pass 作用**

- 将函数形参中标注为 `hacc.arg_type<sync_block_lock>` 的参数绑定到每个未绑定的 `hivm.hir.create_sync_block_lock` 操作的可选输入 `lockArg`，使其形如 `from %arg0 : from memref<?xi8> to memref<1xi64>`。
- 如果函数没有该形参，或者某个 `create_sync_block_lock` 已经显式设置了 `lockArg`，则不做任何修改。
- 为后续的降低步骤做准备：后续 `hivm-lower-create-sync-block-lock` 需要这个绑定，否则会报错并中断降低。
- 依赖 `memref::MemRefDialect` 与 `hivm::HIVMDialect`。

**代码位置**

- Pass 声明与意图：`bishengir/include/bishengir/Dialect/HIVM/Transforms/Passes.td:512-518`
- 具体实现：`bishengir/lib/Dialect/HIVM/Transforms/BindSyncBlockLockArg.cpp:45-65`
- 相关 IR 语法定义（操作形态与语法）：`bishengir/include/bishengir/Dialect/HIVM/IR/HIVMSynchronizationOps.td:180-203`、`:205-225`
- 后续降低会强制要求已绑定：`bishengir/lib/Dialect/HIVM/Transforms/LowerCreateSyncBlockLock.cpp:56-61`

**为什么需要绑定**

- `CreateSyncBlockLockOp` 结果类型通常是单槽锁 `memref<1xi64>`；而函数传入的是一个统一的锁工作区 `memref<?xi8>`。绑定后，降低会把每个 `create_sync_block_lock` 转成对该工作区的 `memref.view`，并依据创建顺序计算偏移，得到不同的锁槽。
- 未绑定时，降低直接报错：“failed to bind sync block lock argument”。

**简单示例**

- 输入（运行前，`create_sync_block_lock` 尚未绑定到形参）：

```mlir
func.func @kernel(
  %lock_ws: memref<?xi8> {hacc.arg_type = #hacc.arg_type<sync_block_lock>},
  %x: memref<16xi32>) {
  %lock0 = hivm.hir.create_sync_block_lock : memref<1xi64>
  %lock1 = hivm.hir.create_sync_block_lock : memref<1xi64>
  return
}
```

- 运行本 Pass 后（为每个未绑定的 `create_sync_block_lock` 填充 `from %lock_ws`）：

```mlir
func.func @kernel(
  %lock_ws: memref<?xi8> {hacc.arg_type = #hacc.arg_type<sync_block_lock>},
  %x: memref<16xi32>) {
  %lock0 = hivm.hir.create_sync_block_lock from %lock_ws
            : from memref<?xi8> to memref<1xi64>
  %lock1 = hivm.hir.create_sync_block_lock from %lock_ws
            : from memref<?xi8> to memref<1xi64>
  return
}
```

- 后续（说明，仅示例使用场景）：降低 Pass 会把上面的两个 `create_sync_block_lock` 转为对 `%lock_ws` 不同偏移的 `memref.view`，随后可被 `hivm.hir.sync_block_lock` / `hivm.hir.sync_block_unlock` 使用，配合块间顺序执行。

**如何运行**

- 单独运行该 Pass：

```bash
bishengir-opt --hivm-bind-sync-block-lock-arg input.mlir -o output.mlir
```

- 在管线中的位置（该 Pass 会被加入函数级管线）：`bishengir/lib/Dialect/HIVM/Pipelines/HIVMPipelines.cpp:244`

如果你希望，我可以继续展示绑定后的 IR 经 `hivm-lower-create-sync-block-lock` 降低为 `memref.view` 的样例，或演示如何与原子存储分解产生的 `sync_block_lock/unlock` 搭配使用。

## hivm-insert-infer-sync-block-lock-num-and-init-func

**Pass 作用**

- 在每个 `func::FuncOp` 中统计 `hivm.hir.create_sync_block_lock` 的数量，并插入两个宿主侧回调函数：
  - `infer_sync_block_lock_num_function`：返回锁槽总数（`i64` 常量）
  - `infer_sync_block_lock_init_function`：返回锁初始值（恒为 `0`，`i64` 常量）
- 这两个函数被标记为“host function”，供宿主代码在运行前分配并初始化用于块级锁的工作区。
- 当函数里没有 `create_sync_block_lock` 时不做任何修改。
- 依赖 `memref::MemRefDialect` 与 `hivm::HIVMDialect`。

**示例**

- 运行前（函数中创建了 3 个锁槽）：

```mlir
func.func @kernel(
  %ffts: i64 {hacc.arg_type = #hacc.arg_type<ffts_base_address>},
  %lock_ws: memref<?xi8> {hacc.arg_type = #hacc.arg_type<sync_block_lock>}
) {
  hivm.hir.create_sync_block_lock from %lock_ws : from memref<?xi8> to memref<1xi64>
  hivm.hir.create_sync_block_lock from %lock_ws : from memref<?xi8> to memref<1xi64>
  hivm.hir.create_sync_block_lock from %lock_ws : from memref<?xi8> to memref<1xi64>
  return
}
```

- 运行该 Pass 后将新增两个宿主函数（函数名基于原始符号名拼接）：

```mlir
func.func @kernel_infer_sync_block_lock_num_function() -> i64 {
  %c3 = arith.constant 3 : i64
  return %c3 : i64
}
func.func @kernel_infer_sync_block_lock_init_function() -> i64 {
  %c0 = arith.constant 0 : i64
  return %c0 : i64
}
```

- 原函数体保持不变，这两段回调供宿主侧查询需要的锁槽数量与初始值。

**如何运行**

- 命令行：

```bash
bishengir-opt --hivm-insert-infer-sync-block-lock-num-and-init-func input.mlir -o output.mlir
```

**代码参考**

- 声明与说明：`bishengir/include/bishengir/Dialect/HIVM/Transforms/Passes.td:520-530`
- 实现与插入逻辑：
  - 收集锁创建操作：`bishengir/lib/Dialect/HIVM/Transforms/InsertInferSyncBlockLockNumAndInitFunc.cpp:45-50`
  - 插入两个宿主函数：`bishengir/lib/Dialect/HIVM/Transforms/InsertInferSyncBlockLockNumAndInitFunc.cpp:74-85`, `108-118`
  - 总体执行：`bishengir/lib/Dialect/HIVM/Transforms/InsertInferSyncBlockLockNumAndInitFunc.cpp:134-154`
- 测试示例：`bishengir/test/Dialect/HIVM/insert-infer-sync-block-lock-num-and-init-func.mlir`

## hivm-lower-create-sync-block-lock

**Pass 作用**

- 将每个 `hivm.hir.create_sync_block_lock` 降低为对锁工作区 `memref<?xi8>` 的 `memref.view`，以得到具体的单槽锁 `memref<1xi64>` 视图，供随后 `sync_block_lock` / `sync_block_unlock` 使用。
- 要求该 `create_sync_block_lock` 已绑定函数形参（`hacc.arg_type<sync_block_lock>`）；未绑定时直接报错并中断降低。
- 为每个锁创建按顺序计算字节偏移：`perOffset = ceil(lock_result_bitwidth / lock_arg_bitwidth)`，并以累计偏移生成视图，确保每个锁使用不同槽位。
- 仅在设备函数上运行；宿主函数跳过。

**示例**

- 运行前（已通过绑定 Pass 将 `from %lock_ws` 填充到 `create_sync_block_lock`）：

```mlir
func.func @kernel(%lock_ws: memref<?xi8> {hacc.arg_type = #hacc.arg_type<sync_block_lock>}) {
  %lock0 = hivm.hir.create_sync_block_lock from %lock_ws
           : from memref<?xi8> to memref<1xi64>
  %lock1 = hivm.hir.create_sync_block_lock from %lock_ws
           : from memref<?xi8> to memref<1xi64>
  hivm.hir.sync_block_lock   lock_var(%lock0 : memref<1xi64>)
  hivm.hir.sync_block_unlock lock_var(%lock0 : memref<1xi64>)
  return
}
```

- 运行该 Pass 后（假设 `lock_arg` 元素是 `i8`，目标锁是 `i64`，则单槽偏移为 `ceil(64/8)=8` 字节）：

```mlir
func.func @kernel(%lock_ws: memref<?xi8> {hacc.arg_type = #hacc.arg_type<sync_block_lock>}) {
  %c0 = arith.constant 0 : index
  %lock0_view = memref.view %lock_ws[%c0] : memref<1xi64>
  %c8 = arith.constant 8 : index
  %lock1_view = memref.view %lock_ws[%c8] : memref<1xi64>
  hivm.hir.sync_block_lock   lock_var(%lock0_view : memref<1xi64>)
  hivm.hir.sync_block_unlock lock_var(%lock0_view : memref<1xi64>)
  return
}
```

- 说明：
  - 每个 `create_sync_block_lock` 被替换为一个 `memref.view`，从统一的锁工作区切出不同槽位。
  - `sync_block_lock/unlock` 继续使用得到的 `memref<1xi64>` 视图，不需要改动。

**如何运行**

- 命令行：

```bash
bishengir-opt --hivm-lower-create-sync-block-lock input.mlir -o output.mlir
```

**代码参考**

- 声明：`bishengir/include/bishengir/Dialect/HIVM/Transforms/Passes.td:532-538`
- 降低规则与偏移计算：`bishengir/lib/Dialect/HIVM/Transforms/LowerCreateSyncBlockLock.cpp:56-79`
- 绑定前置条件（必须有 `lockArg`）：`bishengir/lib/Dialect/HIVM/Transforms/BindSyncBlockLockArg.cpp:48-65`
- 操作校验（仅静态形状）：`bishengir/lib/Dialect/HIVM/IR/HIVMSynchronizationOps.cpp:258-265`

## hivm-auto-infer-buffer-size

**Pass 作用**

- 自动为未标注大小的动态 `memref.alloc` 推断缓冲区字节大小，并插入 `annotation.mark` 标注属性 `buffer_size_in_byte`。
- 推断依据是函数内已存在的某个 `annotation.mark` 提供的统一总字节数：先从任一 `annotation.mark` 的 `buffer_size_in_byte` 读取总字节数，计算统一的元素个数 `numOfElements = buffer_bits / element_bits`，再对其它未标注、且形状动态的 `memref.alloc`，根据其元素位宽反推字节数并插入标注。
- 只有当函数具备属性 `utils::kEnableAutoMarkBufferSize` 时才生效；如果没有该属性或未找到有效标注则不做修改。
- 依赖 `annotation::AnnotationDialect`。

**行为细节**

- 读取统一基准：遍历函数中的 `annotation.mark`，找到首个携带 `buffer_size_in_byte` 的标注，计算 `numOfElements`；若没有，或算出的元素数是 1，则直接返回不处理。
- 处理目标：仅对 `memref.alloc` 的结果值类型为动态形状的情况生效；静态形状不处理；若该 `alloc` 的结果已有 `annotation.mark` 且包含 `buffer_size_in_byte` 则跳过。
- 插入方式：在 `alloc` 之后插入 `annotation.mark %alloc_result` 并设置整数属性 `buffer_size_in_byte`。

**简单示例**

- 输入（函数已开启自动标注，并提供一个基准标注为 4096 字节；其它动态 `alloc` 尚未标注）：

```mlir
func.func @foo() attributes {enable_auto_mark_buffer_size = true} {
  %dyn = memref.alloc() : memref<?xf32>
  %dyn2 = memref.alloc() : memref<?xi8>
  %base = memref.alloc() : memref<?xf32>
  // 基准：告知统一缓冲区总字节数为 4096
  annotation.mark %base {buffer_size_in_byte = 4096 : i64}
  return
}
```

- 运行该 Pass 后（假设用 `%base` 的标注推得元素总数为 `4096 * 8 / 32 = 1024` 元；则为其它动态 `alloc` 推断并插入标注）：

```mlir
func.func @foo() attributes {enable_auto_mark_buffer_size = true} {
  %dyn = memref.alloc() : memref<?xf32>
  // 插入：f32 每元素 32 bit => 1024 * 32 / 8 = 4096 字节
  annotation.mark %dyn {buffer_size_in_byte = 4096 : i64}

  %dyn2 = memref.alloc() : memref<?xi8>
  // 插入：i8 每元素 8 bit => 1024 * 8 / 8 = 1024 字节
  annotation.mark %dyn2 {buffer_size_in_byte = 1024 : i64}

  %base = memref.alloc() : memref<?xf32>
  annotation.mark %base {buffer_size_in_byte = 4096 : i64}
  return
}
```

**如何运行**

- 命令行：

```bash
bishengir-opt --hivm-auto-infer-buffer-size input.mlir -o output.mlir
```

**代码参考**

- 声明：`bishengir/include/bishengir/Dialect/HIVM/Transforms/Passes.td:540-546`
- 实现：`bishengir/lib/Dialect/HIVM/Transforms/AutoInferBufferSize.cpp:41-103`

## insert-workspace-for-mix-cv

**Pass 作用**

- 为 Mix CV 场景在需要的地方把“写入 GM 后再读回”的空目标替换为“本地工作区”以避免不必要的 GM 往返。
- 具体做法：匹配 GM 读（`hivm.hir.load`）或某些 `hivm.hir.debug` 的输入链路，向后回溯其来源，若发现该来源是将计算结果写到一个 `tensor.empty` 的“空目标”（通过 `store` 或 `fixpipe` 的 `outs` 看到），则在那个 `tensor.empty` 的位置改为分配一个“本地工作区张量”，并把所有使用该空目标的地方替换成工作区张量。
- 仅在 Mix CV 组合中设计的连接点生效（注释中称 CC/CV/VC/VV 交汇），实现里以 `load` 和 `debug` 为入口进行回溯替换。
- 依赖 `hivm::HIVMDialect`、`bishengir::memref_ext::MemRefExtDialect`、`bufferization::BufferizationDialect`。

**示例**

- 输入（负例抽象化为“写 GM→读 GM”链路，`empty` 作为 outs 目标）：

```mlir
func.func @mix_cv_case(%src: tensor<16x16xf32>) {
  %dst_empty = tensor.empty() : tensor<16x16xf32>
  %fp = hivm.hir.fixpipe {enable_nz2nd}
          ins(%src : tensor<16x16xf32>)
          outs(%dst_empty : tensor<16x16xf32>) -> tensor<16x16xf32>
  // 下游（例如 load 或 debug）将从 GM 读取 fp 的结果
  hivm.hir.debug {debugtype = "print"} %fp : tensor<16x16xf32>
  return
}
```

- 运行该 Pass 后（把 `empty` 替换成“本地工作区张量”，避免将结果写回 GM 再读回）：

```mlir
func.func @mix_cv_case(%src: tensor<16x16xf32>) {
  // 在 empty 位置插入本地工作区张量（形状与元素类型保持一致）
  %workspace = /* 本地工作区张量，形状 [16,16]，元素 f32 */
  %fp = hivm.hir.fixpipe {enable_nz2nd}
          ins(%src : tensor<16x16xf32>)
          outs(%workspace : tensor<16x16xf32>) -> tensor<16x16xf32>
  hivm.hir.debug {debugtype = "print"} %fp : tensor<16x16xf32>
  return
}
```

- 说明：
  - 工作区张量的创建由 `getLocalWorkSpaceTensor` 完成，形状与原 `empty` 一致，元素类型保持一致。
  - 所有对原 `empty` 结果的使用都会被替换为工作区张量，从而实现就地、局部化的数据流。

**如何运行**

- 命令行：

```bash
bishengir-opt --insert-workspace-for-mix-cv input.mlir -o output.mlir
```

**代码参考**

- 声明：`bishengir/include/bishengir/Dialect/HIVM/Transforms/Passes.td:548-555`
- 实现整体：`bishengir/lib/Dialect/HIVM/Transforms/InsertWorkSpaceForMixCV.cpp:49-117`
- 模式与替换逻辑：
  - 匹配入口 `LoadOp` 与 `DebugOp`：`:101-105`
  - 回溯到 GM 写入源，找到 `tensor.empty` 并在其位置创建工作区张量替换：`:62-99`
- 测试样例：`bishengir/test/Dialect/HIVM/insert-workspace-for-mix-cv.mlir`

## hivm-normalize-loop-iterator

**Pass 作用**

- 规范循环迭代变量与 yield 值的关系，修复一种不利于内存规划的 IR 形态：当循环迭代参数 `iterArg` 与循环体中产出的 `yieldVal` 是不同的缓冲区，而且 `yieldVal` 在循环内有“首次写入初始化”，随后 IR 又在该初始化之后读取了 `iterArg`，这会导致结果回传与别名使用混乱。
- 针对上述情况，在循环终止处插入一次 `hivm.hir.copy`，将 `yieldVal` 的内容复制到 `iterArg`，并把循环的该位置的 `yieldedValues` 改为 `iterArg`，从而使迭代值与回传值一致，避免后续内存别名分析出现冲突。
- 仅处理单区域单基本块的循环样式；仅处理非 GM 地址空间的 `memref` 迭代值；若 `iterArg == yieldVal` 或不是 `memref`，则跳过。

**触发条件简述**

- 循环具备单区域单块。
- 存在索引 `i`，满足：
  - `iterArg[i] != yieldVal[i]` 且 `iterArg[i]` 类型是 `MemRefType`。
  - `yieldVal[i]` 的根分配在循环内（首次写入初始化在循环内）。
  - 在首次初始化之后，IR 还读取了 `iterArg[i]`（存在“初始化后使用迭代值”的情况）。
- 满足后，将在循环终止指令前：
  - 插入 `hivm.hir.copy ins(yieldVal[i]) outs(iterArg[i])`。
  - 将 `yieldedValues[i]` 改为 `iterArg[i]`。

**示例**

- 输入（循环中产出了新缓冲区 `y`，但循环的迭代参数是另一个缓冲区 `x`，且在首次对 `y` 写入后还读取了 `x`）：

```mlir
scf.for %iv = %lb to %ub step %step {
  %x = memref.alloc() : memref<?xf32>
  %y = memref.alloc() : memref<?xf32>
  // ... 对 %y 的首次写入初始化发生在这里（例如 store 到 %y） ...
  %r = memref.load %x[%c0] : memref<?xf32>  // 在首次初始化之后读取了 iterArg(%x)
  scf.yield %y : memref<?xf32>              // yield 与 iterArg 不一致
}
```

- 运行该 Pass 后（在循环终止处插入复制，并使 yield 与 iterArg 对齐）：

```mlir
scf.for %iv = %lb to %ub step %step {
  %x = memref.alloc() : memref<?xf32>
  %y = memref.alloc() : memref<?xf32>
  // ... 首次写入 %y ...
  %r = memref.load %x[%c0] : memref<?xf32>
  // 在循环终止点前插入
  hivm.hir.copy ins(%y : memref<?xf32>) outs(%x : memref<?xf32>)
  scf.yield %x : memref<?xf32>              // yield 改为 iterArg
}
```

- 说明：
  - 复制保障了下一次迭代的 `iterArg` 与当前产出的 `yieldVal` 同步，消除迭代与结果的偏离。
  - 仅当首次初始化在循环内且存在“初始化后读取迭代值”的情况才会触发。

**如何运行**

- 命令行：

```bash
bishengir-opt --hivm-normalize-loop-iterator input.mlir -o output.mlir
```

**代码参考**

- 声明：`bishengir/include/bishengir/Dialect/HIVM/Transforms/Passes.td:557-562`
- 实现与条件判断：
  - 查找 yield 值的首次写入初始化：`yieldMemoryInitialization` `bishengir/lib/Dialect/HIVM/Transforms/NormalizeLoopIterator.cpp:72-111`
  - 检查初始化后是否读取了迭代参数：`existIterArgUseAfterYieldValInit` `:113-139`
  - 修复与重写：`NormalizeIterUseAfterYieldInit::matchAndRewrite` `:146-199`
- Pass 入口：`runOnOperation` `:216-226`

## hivm-inline-otf-load-store

**Pass 作用**

- 针对“`vconcat` 后马上 `store`”且拼接轴为最后一维、并且该维未满足块对齐的场景，内联重写 `store` 源为一系列 `tensor.insert_slice` 逐段插入，从而消除对整块拼接结果的依赖，避免非对齐导致的额外缓冲与搬运。
- 判断条件：
  - `store` 的输入由 `hivm.hir.vconcat` 定义，且是纯张量语义；
  - 拼接维度是最后一维；
  - 若该维是静态且已按块大小对齐（`utils::INTR_BYTES_PER_BLOCK`），则不改写；否则执行内联插片。
- 额外处理：若 `vconcat` 上存在用于缓冲大小的 `annotation.mark {buffer_size_in_byte}`，在完成内联后移除这些标注，使 `vconcat` 可以被进一步消除。

**示例**

- 输入（`vconcat` 在最后一维拼接若干子张量，随后 `store` 其结果；最后一维未按块对齐）：

```mlir
%init = tensor.empty() : tensor<4x128xf32>
%a = tensor.empty() : tensor<4x30xf32>
%b = tensor.empty() : tensor<4x50xf32>
%c = tensor.empty() : tensor<4x48xf32>
%concat = hivm.hir.vconcat dim = 1
           ins(%a, %b, %c : tensor<4x30xf32>, tensor<4x50xf32>, tensor<4x48xf32>)
           outs(%init : tensor<4x128xf32>) -> tensor<4x128xf32>
%out = tensor.empty() : tensor<4x128xf32>
%store = hivm.hir.store ins(%concat : tensor<4x128xf32>) outs(%out : tensor<4x128xf32>) -> tensor<4x128xf32>
```

- 运行该 Pass 后（把 `store` 源替换为逐段 `insert_slice`，消除对整体拼接的依赖；并清理多余标注）：

```mlir
%off = arith.constant 0 : index
%ins1 = tensor.insert_slice %a into %init [0, %off] [4, 30] [1, 1]
%off2 = arith.addi %off, %c30   // 逐段累加偏移
%ins2 = tensor.insert_slice %b into %ins1 [0, %off2] [4, 50] [1, 1]
%off3 = arith.addi %off2, %c50
%ins3 = tensor.insert_slice %c into %ins2 [0, %off3] [4, 48] [1, 1]
hivm.hir.store ins(%ins3 : tensor<4x128xf32>) outs(%out : tensor<4x128xf32>) -> tensor<4x128xf32>
// 针对 %concat 的 buffer_size 标注（若存在）将被移除
```

- 说明：
  - 偏移、大小、步长按最后一维累加计算，确保完整覆盖。
  - 若最后一维静态且拼接后正好满足块对齐，则保持原状不内联。

**如何运行**

- 命令行：

```bash
bishengir-opt --hivm-inline-otf-load-store input.mlir -o output.mlir
```

**代码参考**

- 声明：`bishengir/include/bishengir/Dialect/HIVM/Transforms/Passes.td:564-570`
- 实现与判定逻辑：`bishengir/lib/Dialect/HIVM/Transforms/HIVMInlineOTFLoadStore.cpp:66-129, 137-147`

## hivm-bind-sub-block

**Pass 作用**

- 在混合核（Mix CV）且向量核（AIV）函数上，对整个函数体按“子块”进行包裹、切片和绑定，以实现每个子块处理其局部数据，从而减少跨块写回与读回的冲突并便于后续内存计划。
- 核心步骤：
  - 将原函数体整体包裹到一个 `scf.for` 子块循环（当前子块数 `kSubBlockDim=2`），并设置绑定属性（`mapping = DimX`），形成每个子块独立执行的结构。
  - 在每个 `hivm.hir.store` 前插入 `tensor.extract_slice`/`memref.subview`，按子块维度切分输出，从而把写入限制在当前子块的局部范围；同时在循环体中为动态结果插入 `annotation.mark buffer_size_in_byte`，按每子块的字节数标注。
  - 若变换失败，回退到“只允许子块 0 执行 store”（为 `store` 包裹 `scf.if (sub_block_id == 0)`），确保功能正确但不做切片绑定。
- 仅对满足条件的函数生效：函数 `hivm.func_core_type` 为 `AIV` 且带有 `hivm.part_of_mix` 标记。

**示例**

- 输入（简化版 AIV + Mix 的向量函数，直接将结果写回整块数据）：

```mlir
func.func @aiv_mix(%src: tensor<128x14336xf32>, %dst: tensor<128x14336xf32>)
    attributes {hivm.func_core_type = #hivm.func_core_type<AIV>, hivm.part_of_mix} {
  %res = hivm.hir.vadd ins(%src : tensor<128x14336xf32>) outs(%dst : tensor<128x14336xf32>) -> tensor<128x14336xf32>
  return
}
```

- 运行该 Pass 后（概念化展示）：
  - 包裹整个函数体到子块循环，并在 `store` 前对 `outs` 与源结果做切片，使每个子块只处理各自切片。
  - 为动态结果插入 `annotation.mark buffer_size_in_byte = ceil((总位数/子块数)/8)`。

```mlir
func.func @aiv_mix(...) attributes {...} {
  scf.for %sb = %c0 to %c2 step %c1 {  // 子块循环, 2 个子块
    // 计算当前子块的 offset/size/stride
    %slice_src = tensor.extract_slice %src[...] : tensor<64x14336xf32>
    %slice_dst = tensor.extract_slice %dst[...] : tensor<64x14336xf32>
    %res = hivm.hir.vadd ins(%slice_src) outs(%slice_dst) -> tensor<64x14336xf32>
    // 对动态结果标注每子块字节数
    annotation.mark %res {buffer_size_in_byte = ... : i64} : tensor<64x14336xf32>
  }
  return
}
```

- 如果无法完成切片绑定（例如存在动态形状 store 等限制），会回退为：

```mlir
scf.if (%sub_block_id == 0) {
  // 仅子块 0 执行写回，避免多子块同时写导致冲突
  %res = hivm.hir.vadd ... -> ...
  scf.yield ...
} else {
  scf.yield %dst
}
```

**如何运行**

- 命令行：

```bash
bishengir-opt --hivm-bind-sub-block input.mlir -o output.mlir
```

**代码参考**

- 声明与依赖：`bishengir/include/bishengir/Dialect/HIVM/Transforms/Passes.td:572-578`
- 主逻辑：
  - 包裹函数体到子块循环：`createSubBlockLoop` 与 `attemptBindSubBlock` `/TileAndBindSubBlock.cpp:346-367, 455-520`
  - 为 `store` 插入切片并标注 buffer 大小：`TileAndSliceStore`、`modifyStoreToSliced`、`setBufferSizeInLoopOp` `:207-257, 165-203, 127-163`
  - 回退策略：`limitUniqueSubBlockToStore` `:274-334`
- 适用函数筛选：`runOnOperation` `:542-547`

## hivm-bubble-up-extract-slice

**Pass 作用**

- 将标记为需要上浮的 `tensor.extract_slice` 或 `memref.subview` 往其依赖链上“冒泡”，把切片操作提前到更靠近数据源的位置，使下游 `hivm` 运算直接消费切片后的数据。典型用途是配合“子块绑定”流水线，将在 `store` 前插入的切片沿着数据流向上移动，使计算在每个子块范围内就地进行，减小跨子块数据交叉和内存压力。
- 仅对已标记的切片执行（通过内部标记，比如 `toBeBubbleUpSlice`），且要求切片为单位步长且静态大小；遇到不安全场景（动态切片、多重使用、层级不匹配）会保守跳过。
- 针对不同源操作提供专门策略，包括广播 `VBrcOp`、归约 `VReduceOp`、`tensor.expand_shape`、连续的 `extract_slice` 或 `insert_slice` 链等，分别计算对应的新偏移/大小并重建上游操作，使功能等价但数据范围缩小到子块。

**工作方式概览**

- 判断切片的父子关系与使用情况，确认可冒泡并选择策略。
- 计算子块维度上的 `offset/size/stride`，用该信息在父操作的输入位置创建新的切片（把切片“提到上游”）。
- 重建父操作使其输入改为新的切片输出，替换原有下游切片，保证结果语义一致。
- 对 `memref.subview` 的场景，若父子 `subview` 来源于切片化（由子块切分创建），以相同流程提升。

**简单示例**

- 输入（在 `vbrc` 后做切片，意味着对广播后的大张量切片，开销更大）：

```mlir
%init = tensor.empty() : tensor<1x24x256x128xf32>
%b = hivm.hir.vbrc ins(%src : tensor<1x24x1x128xf32>) outs(%init : tensor<1x24x256x128xf32>) -> tensor<1x24x256x128xf32>
%slice = tensor.extract_slice %b[0,0,77,0] [1,24,256,128] [1,1,1,1] : tensor<1x24x256x128xf32> to tensor<1x24x256x128xf32>
%mul = linalg.elemwise_binary {fun = #linalg.binary_fn<mul>} ins(%slice, %x) outs(%init2) -> tensor<1x24x256x128xf32>
```

- 运行该 Pass 后（将切片上浮到 `vbrc` 的输入侧，避免对广播结果大张量切片）：

```mlir
// 在 vbrc 的输入 %src 上创建与子块一致的 extract_slice
%src_slice = tensor.extract_slice %src[...] : tensor<1x24x1x128xf32> to tensor<1x24x1x128xf32>
// 对 init 也同步切片
%init_slice = tensor.extract_slice %init[...] : tensor<1x24x256x128xf32> to tensor<1x24x256x128xf32>
// 用切片后的输入重建 vbrc，使其直接产出子块范围
%b_tiled = hivm.hir.vbrc ins(%src_slice) outs(%init_slice) -> tensor<1x24x256x128xf32>
// 下游直接消费 %b_tiled，不再需要额外切片
%mul = linalg.elemwise_binary ... ins(%b_tiled, %x) outs(%init2_slice) -> tensor<...>
```

- 注：
  - 对 `vreduce`、`expand_shape`、连续 `extract_slice`、`insert_slice` 等都有相应的冒泡计算与重建逻辑。
  - 冒泡过程中会对 `init` 侧同步切片，保持形状与语义一致。

**如何运行**

- 命令行：

```bash
bishengir-opt --hivm-bubble-up-extract-slice input.mlir -o output.mlir
```

**代码参考**

- 声明：`bishengir/include/bishengir/Dialect/HIVM/Transforms/Passes.td:580-586`
- 广播策略：`bishengir/lib/Dialect/HIVM/Transforms/BubbleUpExtractSlice/Pattern.cpp:251-322`
- 归约策略：`:324-387`
- 连续 `extract_slice` 场景与秩降低处理：`:460-607`
- `insert_slice` 场景与秩降低处理：`:609-659`
- `memref.subview` 冒泡：`:213-243`
- 辅助计算（偏移/大小、子块单 tile 尺寸等）：`bishengir/lib/Dialect/HIVM/Transforms/BubbleUpExtractSlice/MoveUpAffineMap.cpp` 和 `TileAndBindSubBlock/Helper.h`

## hivm-insert-init-and-finish-for-debug

**Pass 作用**

- 为存在 `hivm.hir.debug` 的函数自动插入调试初始化与结束指令：
  - 在函数入口插入一次 `hivm.hir.init_debug`
  - 在每个 `hivm.hir.debug` 之后插入 `hivm.hir.finish_debug`
- 如果函数中没有 `debug`，或已存在 `init_debug`，则不做任何改动。
- 对每个 `debug` 只插入一次 `finish_debug`，通过内部标记属性避免重复。

**示例**

- 输入（包含一个调试打印）：

```mlir
func.func @kernel(%t: tensor<16x16xf32>) {
  hivm.hir.debug {debugtype = "print", hex = false} %t : tensor<16x16xf32>
  return
}
```

- 运行该 Pass 后：

```mlir
func.func @kernel(%t: tensor<16x16xf32>) {
  hivm.hir.init_debug
  hivm.hir.debug {debugtype = "print", hex = false} %t : tensor<16x16xf32>
  hivm.hir.finish_debug
  return
}
```

- 说明：
  - `init_debug` 放在函数首条指令位置，初始化调试环境；
  - 每个 `debug` 紧随其后插入 `finish_debug`，标记属性确保不会二次插入。

**如何运行**

- 命令行：

```bash
bishengir-opt --hivm-insert-init-and-finish-for-debug input.mlir -o output.mlir
```

**代码参考**

- 声明：`bishengir/include/bishengir/Dialect/HIVM/Transforms/Passes.td:588-592`
- 实现：`bishengir/lib/Dialect/HIVM/Transforms/InsertInitAndFinishForDebug.cpp:98-121`

## hivm-mark-disable-load

**Pass 作用**

- 标记可能引发数据缓存一致性问题的 `memref.load`，在其操作上添加属性 `disableDCache`，提示后续代码生成使用“设备侧加载”（如 `ld_dev`）或禁用缓存路径，避免与写端竞争造成脏读。
- 判定逻辑：
  - 溯源该 `load` 的 `memref` 到函数形参（经由 `view` 类操作链回溯）；
  - 收集该形参的“写端终端使用者”（最终写内存的操作，包含经由 `view` 的间接使用）；
  - 若存在写端终端使用者，则认为该 `load` 与其他写者共享同一底层存储，需标记 `disableDCache`。
- 同一 `load` 只处理一次，通过内部标记属性避免重复。

**示例**

- 输入（同一函数实参 `%buf` 被写入同时也被读取）：

```mlir
func.func @kernel(%buf: memref<?xf32>) {
  memref.store %v, %buf[%i] : memref<?xf32>
  %x = memref.load %buf[%j] : memref<?xf32>
  return
}
```

- 运行该 Pass 后（为 `load` 标注禁用缓存）：

```mlir
func.func @kernel(%buf: memref<?xf32>) {
  memref.store %v, %buf[%i] : memref<?xf32>
  %x = memref.load %buf[%j] : memref<?xf32> {disableDCache = 0 : i32}
  return
}
```

- 说明：
  - 属性值 `0 : i32` 只是占位；核心是有该属性以供后端识别并选择合适的访存指令或路径。
  - 若溯源不到函数形参，或该形参没有任何写端终端使用者，则不标记。

**如何运行**

- 命令行：

```bash
bishengir-opt --hivm-mark-disable-load input.mlir -o output.mlir
```

**代码参考**

- 声明：`bishengir/include/bishengir/Dialect/HIVM/Transforms/Passes.td:594-598`
- 实现与判定：`bishengir/lib/Dialect/HIVM/Transforms/MarkDisableLoad.cpp:88-117, 120-132`

## hivm-insert-nz2nd-for-debug

**Pass 作用**

- 在遇到 `hivm.hir.mmadL1` 的输入张量且这些输入被 `hivm.hir.debug` 使用时，为调试友好插入 `hivm.hir.nz2nd` 操作，将张量从 NZ 格式转换为 ND 格式，再把 `debug` 的参数改为转换后的结果，从而以更直观的 ND 布局进行打印或检查。
- 仅针对具有定义操作的张量输入生效；如果该输入已存在 `nz2nd` 用户则不会重复插入。
- 转换的目标缓冲通过 `getLocalWorkSpaceTensor` 分配本地工作区张量，形状与元素类型与原输入一致。

**示例**

- 输入（`mmadL1` 的输入 `%a` 同时被 `debug` 使用）：

```mlir
%a = ... : tensor<64x64xf16>
%b = ... : tensor<64x64xf16>
%out = tensor.empty() : tensor<64x64xf32>
%mm = hivm.hir.mmadL1 ins(%a, %b, %cond, %m, %k, %n : ...) outs(%out : tensor<64x64xf32>) -> tensor<64x64xf32>
hivm.hir.debug {debugtype = "print"} %a : tensor<64x64xf16>
```

- 运行该 Pass 后（在 `%a` 的定义后插入 `nz2nd` 并更新 `debug` 参数）：

```mlir
%ws = /* 本地工作区张量，形状 [64,64], 元素 f16 */
%a_nd = hivm.hir.nz2nd ins(%a : tensor<64x64xf16>) outs(%ws : tensor<64x64xf16>) -> tensor<64x64xf16>
hivm.hir.debug {debugtype = "print"} %a_nd : tensor<64x64xf16>
%mm = hivm.hir.mmadL1 ins(%a, %b, ...) outs(%out) -> tensor<64x64xf32>
```

- 说明：
  - 若 `%a` 已有 `nz2nd` 用户，则不再插入。
  - 仅当 `%a` 是张量类型、且存在定义操作与被 `debug` 使用时才处理。

**如何运行**

- 命令行：

```bash
bishengir-opt --hivm-insert-nz2nd-for-debug input.mlir -o output.mlir
```

**代码参考**

- 声明：`bishengir/include/bishengir/Dialect/HIVM/Transforms/Passes.td:600-608`
- 实现：`bishengir/lib/Dialect/HIVM/Transforms/InsertNZ2NDForDebug.cpp:57-112`

## cv-pipelining

**Pass 作用**

- 面向 Mix-CV 场景的流水化改造：基于块内标注的多缓冲计数（`annotation.mark {MultiBuffer}`），将同一块中的相关 `hivm` 张量/存储操作分组为多个“工作项”，在外层循环上按多缓冲数进行展开，并为每个工作项构造内层循环与迭代参数，使数据在子缓冲维度上分片处理，从而降低互相依赖、提升并行与吞吐。
- 关键步骤：
  - 发现带 `MultiBuffer` 标注的块，提取其中向量核或 CUBE 核相关操作，构建工作项集合；
  - 自动或启发式地平衡各工作项的开销；
  - 外层 `scf.for` 按多缓冲数进行展开（重构 IV 与上界），同时扩展工作区 `alloc_workspace` 的形状，在首维增加多缓冲维度；
  - 为每个工作项包裹一层内 `scf.for`，将其相关操作迁移到循环体，调整 `dps init` 采用 `tensor.extract_slice`，并将局部输出以 `tensor.insert_slice` 聚合到迭代参数或工作区；
  - 对工作区和局部输出的下游使用者，替换为从多缓冲张量中提取相应切片的使用。

**简单示例**

- 输入（块中有 `MultiBuffer=3` 的标注，含 `store` 和一些向量核操作；工作区为 `alloc_workspace`）：

```mlir
// block 内有标注 {MultiBuffer = 3}
%ws = hivm.hir.alloc_workspace ... : memref<16x16xf16>
%t = bufferization.to_tensor %ws : tensor<16x16xf16>
%vec = some_vector_op ins(%t, ...) outs(%init) -> tensor<16x16xf16>
hivm.hir.store ins(%vec) outs(%dst) -> tensor<16x16xf16>
```

- 运行该 Pass 后（概念化结果，省略大量细节）：

```mlir
// 工作区扩展为多缓冲首维
%ws_mb = hivm.hir.alloc_workspace ... : memref<3x16x16xf16>
// 每个工作项包裹内层 for，并在迭代参数中累积局部输出
scf.for %iv = 0 to %ub step 1 {
  // 根据 %iv 选择多缓冲切片
  %ws_slice = memref.subview %ws_mb[%iv, 0, 0] [1, 16, 16] [1, 1, 1]
  // 迁移的原始计算，改造输出 init 为 extract_slice
  %t_mb = bufferization.to_tensor %ws_slice : tensor<16x16xf16>
  %vec_mb = some_vector_op ins(%t_mb, ...) outs(%init_slice) -> tensor<16x16xf16>
  // 局部输出以 insert_slice 汇入迭代参数或工作区
  %acc = tensor.insert_slice %vec_mb into %acc_prev [%iv, 0, 0] [1, 16, 16] [1, 1, 1]
  scf.yield %acc
}
// 外部使用者统一改为从多缓冲聚合张量中按 %iv 或最后一轮索引提取切片使用
```

- 说明：
  - 扩展工作区：将 `memref<16x16xf16>` 扩成 `memref<3x16x16xf16>`，并更新相关标注；
  - DPS 输出的 `init` 由原始张量改为 `extract_slice` 到当前多缓冲位置；
  - 局部结果以 `insert_slice` 聚合到迭代参数；外部使用改为 `extract_slice` 对应位置；
  - 若仅形成 2 个工作项，流水化会让位于双缓冲优化。

**如何运行**

- 命令行：

```bash
bishengir-opt --cv-pipelining input.mlir -o output.mlir
```

**代码参考**

- 声明：`bishengir/include/bishengir/Dialect/HIVM/Transforms/Passes.td:610-617`
- 标注解析与多缓冲计数：`bishengir/lib/Dialect/HIVM/Transforms/CVPipelining.cpp:378-404, 1641-1647`
- 工作区扩展与标注更新：`:404-430`
- 分组与成本估计、平衡：`:1219-1237, 1282-1327`
- 循环重构与切片插入/提取：`:1330-1422, 1456-1524, 1526-1565, 1567-1631`
- 入口与主流程：`:1633-1711`

## auto-blockify-parallel-loop

**Pass 作用**

- 当逻辑块数量（由标注 `annotation.mark {logical_block_num}` 给出）大于物理块数量（依据设备规格推断）时，自动在函数入口外层包裹一个循环，将原先依赖 `hivm.hir.get_block_idx` 的块索引改为 `min(iv * physical_block_num + block_idx, logical_block_num)`，从而按批次遍历所有逻辑块，避免一次性超出物理并行度。
- 保留必须在循环外的依赖项（例如用于计算 `logical_block_num` 的依赖链），其余非 `return` 指令搬移到循环体中执行。

**示例**

- 输入（逻辑块数通过 `annotation.mark` 提供，函数中使用 `get_block_idx` 取得当前块号）：

```mlir
func.func @entry(%n: i32) {
  %logic = annotation.mark {logical_block_num} %n : i32
  %bid = hivm.hir.get_block_idx : i64
  // use %bid 作为块级索引访问或计算
  ...
  return
}
```

- 运行该 Pass 后（在外层插入 `for`，重写块索引为按批次遍历的最小值）：

```mlir
func.func @entry(%n: i32) {
  %logic = annotation.mark {logical_block_num} %n : i32
  %phys = arith.constant <device_core_count> : i32
  %ub = arith.ceildivsi %logic, %phys : i32
  scf.for %iv = %c0 to %ub step %c1 {
    %bid = hivm.hir.get_block_idx : i64
    %bid_i32 = arith.trunci %bid : i64 to i32
    %mix = arith.addi (arith.muli %iv, %phys), %bid_i32 : i32
    %mix_clamp = arith.minsi %mix, %logic : i32
    %bid_new = arith.extsi %mix_clamp : i32 to i64
    // 用 %bid_new 替换原先 %bid 的使用
    ...
  }
  return
}
```

- 说明：
  - 物理块数量通过设备规格接口查询：向量核用 `VECTOR_CORE_COUNT`，CUBE 核用 `CUBE_CORE_COUNT`；
  - 循环上界为 `ceildiv(logical_block_num, physical_block_num)`；
  - 对 `get_block_idx` 的所有使用替换为扩展后的 `bid_new`，保证索引落在逻辑块范围内。

**如何运行**

- 命令行：

```bash
bishengir-opt --auto-blockify-parallel-loop input.mlir -o output.mlir
```

**代码参考**

- 声明：`bishengir/include/bishengir/Dialect/HIVM/Transforms/Passes.td:621-628`
- 设备核心数获取与块索引替换：`bishengir/lib/Dialect/HIVM/Transforms/AutoBlockifyParallelLoop.cpp:76-95, 96-111`
- 循环构造与指令迁移：`:112-163, 166-179`

## compose-collapse-expand

**Pass 作用**

- 将连续出现的 `memref.collapse_shape` 与 `memref.expand_shape` 进行合并重写，直接用一个等价的 `collapse_shape` 或 `expand_shape` 替换，减少不必要的形状转换链路，保持内存布局为默认（identity）时的优化。
- 关键点：
  - 仅在参与的 `memref` 类型均为默认布局（无非标布局）时进行；
  - 会检查动态维度是否“符号等价”：通过 `memref.dim` 与 `symbol.bind_symbolic_shape` 追踪，确认两端动态维度绑定到同一符号再允许合并；
  - 若输入/输出 rank 不同，计算合适的 `reassociation indices` 以实现折叠或展开的合成。

**示例**

- 输入（先折叠两个维度再展开为另一组合，形状兼容且动态维度符号一致）：

```mlir
%1 = memref.collapse_shape %m [[0, 1], [2]] : memref<?x?x32xf32> into memref<?x32xf32>
%2 = memref.expand_shape %1 [[0], [1, 2]] : memref<?x32xf32> into memref<?x?x32xf32>
```

- 运行该 Pass 后（合成成一次折叠或展开，示例为保持最终结果的等价操作）：

```mlir
%2 = memref.collapse_shape %m [[0], [1, 2]] : memref<?x?x32xf32> into memref<?x32xf32>
// 或在另一方向：根据 rank 与 reassociation 计算结果，生成等价的 expand/collapse
```

- 说明：
  - 当源 rank 大于结果 rank 时倾向生成新的 `collapse_shape`；
  - 否则生成新的 `expand_shape`，并正确传递 `output_shape` 的动态尺寸。

**如何运行**

- 命令行：

```bash
bishengir-opt --compose-collapse-expand input.mlir -o output.mlir
```

**代码参考**

- 声明：`bishengir/include/bishengir/Dialect/HIVM/Transforms/Passes.td:627-632`
- 合并与符号检查逻辑：`bishengir/lib/Dialect/HIVM/Transforms/ComposeCollapseExpand.cpp:69-116, 118-272`

## tile-cube-vector-loop

**Pass 作用**

- 面向 CUBE/向量核混合场景，对标记为管线循环的 `scf.for` 进行子块化（sub-tiling）与环路融合，目标是在本地缓冲上按指定“分块次数”（trip count）切分操作，提高吞吐与内存对齐。
- 处理两类循环：
  - 向量循环：分析每个 `hivm.hir.store` 与 `scf.yield` 的张量结果，确定可切分维度；必要时在 `yield` 路径上插入“占位 store”（dummy store）以便统一按 store 进行切分与融合，后续再清理该占位。
  - CUBE 循环：定位唯一的 `hivm.hir.fixpipe`，以其目标张量大小与设备 `L0C_SIZE` 估算是否需要切分并选择 M/N 较大维度进行子块化；沿 `mmadL1` 输入链标记生产者以实现融合。
- 变换流程：
  - 预处理：将 `memref` 上的 `hivm.hir.load` 提升为张量版，以便在张量域做切分；
  - 收集循环信息与生产者标记；构造 Transform 序列执行 `tile`、`fuse` 等变换；若失败则回滚；
  - 后处理：移除占位 `store`；收缩 `memref.alloc` 以匹配实际 `subview` 大小，避免 UB 侧非连续布局与空间浪费。

**简单示例**

- 输入（向量循环，`store` 与 `yield` 结果需要在某维切分）：

```mlir
scf.for %i = %c0 to %c64 step %c1 {
  %tmp = hivm.hir.vop ins(...) outs(%init : tensor<1x1024xf16>) -> tensor<1x1024xf16>
  %st = hivm.hir.store ins(%tmp) outs(%dst : tensor<1x1024xf16>) -> tensor<1x1024xf16>
  scf.yield %tmp : tensor<1x1024xf16>
}
```

- 运行该 Pass 后（概念化，核心变化）：
  - 选定可切分维度（如最后一维 1024），根据 trip count 计算 tile size；
  - 通过 Transform 序列对 `store` 和沿 `yield` 产生的“占位 store”进行 `tile`，并将相关生产者融合进切分后的环路；
  - 清理占位 store，缩小不必要的 `alloc`。
- 输入（CUBE 循环，含 `mmadL1` 与 `fixpipe`）：

```mlir
scf.for %k = %c0 to %c128 step %c1 {
  %a_t = bufferization.to_tensor %a_mem
  %b_t = bufferization.to_tensor %b_mem
  %mm = hivm.hir.mmadL1 ins(%a_t, %b_t, ...) outs(%c_init) -> tensor<64x64xf16>
  hivm.hir.fixpipe ins(%mm) outs(%dst : tensor<64x64xf16>) -> tensor<64x64xf16>
}
```

- 运行该 Pass 后（概念化，核心变化）：
  - 评估 `fixpipe` 目标大小与 `L0C_SIZE`，必要时按 M/N 较大维度子块化；
  - 标记沿 `mmadL1` 输入链的生产者，执行 `tile` 并融合至包含环路；
  - 后处理同上。

**如何运行**

- 命令行：

```bash
bishengir-opt --tile-cube-vector-loop input.mlir -o output.mlir
```

**代码参考**

- 声明：`bishengir/include/bishengir/Dialect/HIVM/Transforms/Passes.td:632-641`
- 提升 `load` 到张量域与收缩 `alloc`：`bishengir/lib/Dialect/HIVM/Transforms/TileCubeVectorLoop.cpp:237-301, 342-347, 1209-1224`
- 向量循环信息收集与占位 store：`:741-771, 781-854, 990-1068`
- CUBE 循环信息与 `fixpipe` 子块化计算：`:1083-1127, 894-934, 1114-1127`
- 变换应用与回滚、清理：`:1130-1219, 1200-1224`

## convert-non-contiguous-reshape-to-copy

**Pass 作用**

- 将可能导致非连续内存访问的 `memref.collapse_shape` 重写为“先拷贝到连续缓冲，再折叠”的安全版本，避免后续在向量核/UB侧出现步进不一致或不连续布局引发的精度和性能问题。
- 触发条件：
  - 目标结果值上存在标注 `annotation.mark {maybeUnCollapsibleReshape}`；
  - 函数核心类型不是 AIC（CUBE 核），即通常在向量侧使用；
- 改写逻辑：
  - 为 `collapse` 的源 `memref` 分配一个相同形状与元素类型的连续临时缓冲；
  - 通过 `hivm.hir.copy` 从原 `memref` 拷贝到该连续缓冲，并设置“完全折叠”的重关联属性；
  - 将 `collapse` 的输入替换为该连续缓冲，随后执行 `collapse_shape`，达到“先规整再重排”的效果；
  - 在进入改写前移除触发标注，避免重复处理。

**示例**

- 输入（带有“可能不可折叠”的标注）：

```mlir
%src = ... : memref<?x?x32xf32>
%mark = annotation.mark {maybeUnCollapsibleReshape} %res : memref<?x32xf32>
%collapsed = memref.collapse_shape %src [[0, 1], [2]] : memref<?x?x32xf32> into memref<?x32xf32>
```

- 运行该 Pass 后（先复制到连续缓冲，再折叠）：

```mlir
%tmp = memref.alloc() : memref<?x?x32xf32>   // 连续缓冲
hivm.hir.copy ins(%src : memref<?x?x32xf32>) outs(%tmp : memref<?x?x32xf32>) {collapse_reassociation = [[0,1,2]]}
%collapsed = memref.collapse_shape %tmp [[0, 1], [2]] : memref<?x?x32xf32> into memref<?x32xf32>
```

- 说明：
  - 标注 `maybeUnCollapsibleReshape` 会被移除；
  - 在 AIC 核函数中不会改写（认为不需要此保护）。

**如何运行**

- 命令行：

```bash
bishengir-opt --convert-non-contiguous-reshape-to-copy input.mlir -o output.mlir
```

**代码参考**

- 声明：`bishengir/include/bishengir/Dialect/HIVM/Transforms/Passes.td:655-660`
- 实现与触发标注处理：`bishengir/lib/Dialect/HIVM/Transforms/ConvertNonContiguousReshapeToCopy.cpp:43-94, 98-108`

# Pass 定义文件：`Dialect_MemRef_Transforms_Passes.td`

## fold-alloc-reshape

**Pass 作用**

- 把 `memref.alloc` 后紧跟的 `memref.expand_shape` 折叠在一起：直接按展开后的形状进行分配，移除中间的 `expand_shape`。
- 典型模式是 `memref.alloc -> memref.expand_shape`，折叠成一个新的 `memref.alloc`，其结果类型与 `expand_shape` 的结果类型一致。
- 好处：减少 IR 中的“视图”操作，提高后续分析与优化（如拷贝消除、死代码消除）的机会。

**触发条件**

- 仅在 `expand_shape` 的输入是由 `memref.alloc` 定义时触发。
- 原始 `alloc` 必须只有一个用户（避免改变类型影响到其他用法）。
- 参考实现：
  - 匹配 `expand_shape` 源为 `alloc` 且 `alloc->hasOneUse()`：`bishengir/lib/Dialect/MemRef/Transforms/FoldAllocReshape.cpp:47-55`
  - 在 `expand_shape` 之后插入新的 `memref.alloc`，类型取 `expand_shape` 的结果类型，动态维度取 `expand_shape` 提供的输出形状值：`FoldAllocReshape.cpp:56-60`
  - 模式注册与应用：`FoldAllocReshape.cpp:65-72`
  - 入口与构造器定义：`bishengir/include/bishengir/Dialect/MemRef/Transforms/Passes.td:23-30`

**重写规则**

- 原 IR：
  - `%view = memref.expand_shape %alloc ... : ... into memref<...>`
- 重写为：
  - `%alloc_expanded = memref.alloc(...dynamic sizes...) : memref<...>`
  - 所有对 `%view` 的使用改为使用 `%alloc_expanded`，`expand_shape` 被删除。

**如何使用**

- 在 MLIR 工具链中启用该 Pass 的参数名为 `-fold-alloc-reshape`。
- 示例命令：
  - `mlir-opt input.mlir -fold-alloc-reshape`
- 代码构造器为 `mlir::memref::createFoldAllocReshapePass()`，可在自定义管线中加入：`bishengir/lib/Dialect/HIVM/Pipelines/HIVMPipelines.cpp` 等处通常以类似方式注册调用。

**简单示例**

- 静态形状示例（1D 展开为 2D）
  - 折叠前：
    ```mlir
    %alloc = memref.alloc() : memref<12xf32>
    %view  = memref.expand_shape %alloc [[0, 1]]
             : memref<12xf32> into memref<3x4xf32>
    use(%view)
    ```
  - 折叠后：
    ```mlir
    %alloc_expanded = memref.alloc() : memref<3x4xf32>
    use(%alloc_expanded)
    ```
- 动态形状示例（把动态维度透传给新的 alloc）
  - 折叠前：
    ```mlir
    %n = arith.constant 12 : index
    %alloc = memref.alloc(%n) : memref<?xf32>
    %view  = memref.expand_shape %alloc [[0, 1]]
             : memref<?xf32> into memref<?x?xf32>
    use(%view)
    ```
  - 折叠后（新的 `alloc` 接受 `expand_shape` 提供的动态维度）：
    ```mlir
    %alloc_expanded = memref.alloc(%n0, %n1) : memref<?x?xf32>
    use(%alloc_expanded)
    ```
  - 注：具体维度值来源于 `expand_shape` 的输出形状计算；实现里通过 `op.getOutputShape()` 传给新的 `memref.alloc`（`FoldAllocReshape.cpp:56-60`）。

**相关代码**

- Pass 声明与注册标识：`bishengir/include/bishengir/Dialect/MemRef/Transforms/Passes.td:23-30`
- 实现类与模式：
  - `FoldAllocReshapeOp` 定义与 `runOnOperation`：`bishengir/lib/Dialect/MemRef/Transforms/FoldAllocReshape.cpp:33-37,65-72`
  - 匹配与重写逻辑：`bishengir/lib/Dialect/MemRef/Transforms/FoldAllocReshape.cpp:41-63`
- 工厂函数：`bishengir/lib/Dialect/MemRef/Transforms/FoldAllocReshape.cpp:74-76`

## memref-dse

**Pass 作用**

- 消除“死写入”和“死分配”，并进行负载前向替换：把紧邻、可证明最近一次 `memref.store` 的值直接替换后续的 `memref.load`，同时移除未被读取即被覆盖的写入以及无用的 `alloc/dealloc`。
- 工作范围是函数级别的 `func::FuncOp`，启用参数为 `-memref-dse`。
- 代码依据：
  - 声明与构造器：`bishengir/include/bishengir/Dialect/MemRef/Transforms/Passes.td:33-36`
  - 实现入口：`bishengir/lib/Dialect/MemRef/Transforms/DeadStoreElimination.cpp:161-176`
  - 负载前向替换：`DeadStoreElimination.cpp:143-154`
  - 同层级预计算“最后写入”并匹配索引：`DeadStoreElimination.cpp:73-101`

**关键逻辑**

- 建立值到根 `alloc` 的依赖映射，以便跨 `subview/reshape` 统一按源缓冲跟踪：`DeadStoreElimination.cpp:133-140, 156-158`。
- 同层级扫描，将每个 `memref.store` 记录为该缓冲区的“最新写入”，遇到 `memref.load` 时若索引一致或缓冲为标量样式（0 维），即可用最新写入的值替换 `load`：`DeadStoreElimination.cpp:80-101, 143-154`。
- 对任意“写内存”的操作（实现 `MemoryEffectOpInterface`）或函数调用参数，会清空缓存，避免跨副作用错误前向：`DeadStoreElimination.cpp:50-71, 101-115`。
- 在完成前向替换后，调用 `memref::eraseDeadAllocAndStores` 移除死写和死分配：`DeadStoreElimination.cpp:175`。

**触发条件**

- 负载前向替换要求同层级、同索引（非标量时）且期间没有其他会写该缓冲的副作用。
- 死写消除在“写后从未被读取即再次被写或释放”的场景触发；死分配消除在缓冲不再被使用时触发。

**简单示例**

- 示例 1：负载前向替换（同索引）
  - 优化前：
    ```mlir
    func.func @f(%i: index) -> f32 {
      %buf = memref.alloc() : memref<10xf32>
      %cst = arith.constant 1.0 : f32
      memref.store %cst, %buf[%i] : memref<10xf32>
      %x = memref.load %buf[%i] : memref<10xf32>
      return %x : f32
    }
    ```
  - 负载前向后：
    ```mlir
    func.func @f(%i: index) -> f32 {
      %buf = memref.alloc() : memref<10xf32>
      %cst = arith.constant 1.0 : f32
      memref.store %cst, %buf[%i] : memref<10xf32>
      return %cst : f32
    }
    ```
  - 死写/死分配清理后（若 `%buf` 无后续用途）：
    ```mlir
    func.func @f(%i: index) -> f32 {
      %cst = arith.constant 1.0 : f32
      return %cst : f32
    }
    ```
- 示例 2：死写入消除（覆盖未读）
  - 优化前：
    ```mlir
    %buf = memref.alloc() : memref<4xf32>
    memref.store %a, %buf[%i] : memref<4xf32>
    memref.store %b, %buf[%i] : memref<4xf32>
    %y = memref.load %buf[%i] : memref<4xf32>
    return %y : f32
    ```
  - 消除后（第一条 `store` 死写被移除；负载前向到 `%b`）：
    ```mlir
    %buf = memref.alloc() : memref<4xf32>
    memref.store %b, %buf[%i] : memref<4xf32>
    return %b : f32
    ```
  - 若 `%buf` 不再使用，进一步清理：
    ```mlir
    return %b : f32
    ```
- 示例 3：标量样式缓冲（0 维）
  - 优化前：
    ```mlir
    %buf = memref.alloc() : memref<f32>
    memref.store %v, %buf[] : memref<f32>
    %x = memref.load %buf[] : memref<f32>
    return %x : f32
    ```
  - 优化后：
    ```mlir
    return %v : f32
    ```

**使用方式**

- 命令行启用：`mlir-opt input.mlir -memref-dse`
- 代码管线中构造器：`mlir::memref::createDeadStoreEliminationPass()`（`bishengir/include/bishengir/Dialect/MemRef/Transforms/Passes.h:34`）

**相关代码引用**

- 声明行：`bishengir/include/bishengir/Dialect/MemRef/Transforms/Passes.td:33-39`
- 实现入口与清理：`bishengir/lib/Dialect/MemRef/Transforms/DeadStoreElimination.cpp:161-180`
- 前向替换条件：`DeadStoreElimination.cpp:88-101, 143-154`

## memref-remove-redundant-copy

**Pass 作用**

- 去除冗余的拷贝操作：识别 `CopyOpInterface`（如 `memref.copy`）在不同场景下不需要的拷贝，并用更直接的值或缓冲替代。
- 同步清理相关 `alloc/dealloc`：当拷贝被去除后，如果目标或源缓冲的定义/释放已无用，则一并删除。
- 工作在 `func::FuncOp` 上，命令行参数为 `-memref-remove-redundant-copy`。

**核心规则**

- 子视图写重定向：当源是外层块分配的缓冲，内层块中存在唯一一次拷贝到 `memref.subview`，且该拷贝是内层块对源的最后用户时，直接把内层块中对源的使用替换为子视图，删除该拷贝，并把 `subview` 提前到第一个使用之前。参考 `bishengir/lib/Dialect/MemRef/Transforms/RemoveRedundantCopy.cpp:138-175`。
- 目标为分配的缓冲：若目标是 `alloc/alloca` 且在拷贝前没有对目标的其他使用，同时在拷贝到源的释放或块末尾之间源没有被再次使用，则用源替换目标的所有使用，删除目标的 `alloc/dealloc` 与该拷贝。参考 `RemoveRedundantCopy.cpp:178-245, 219-243, 229-235`。
- 目标为函数参数（或外层定义的值）：如果目标无定义（如 `BlockArgument`），且源的定义位于目标父 op 的祖先范围内，则用目标替换源的所有使用，删除源的 `alloc/dealloc` 与该拷贝。参考 `RemoveRedundantCopy.cpp:215-258`。
- 安全性检查：通过路径扫描避免“区间内仍有使用”的情况，保证替换合法；使用“最后且唯一用户”约束确保重定向不会改变语义。参考 `RemoveRedundantCopy.cpp:76-95, 97-111`。
- 该 Pass只遍历拷贝接口的 op 并执行业务逻辑；整体入口见 `RemoveRedundantCopy.cpp:262-274`。

**简单示例**

- 示例 1：子视图重定向（删拷贝）
  - 优化前：
    ```mlir
    %A = memref.alloc() : memref<16xf32>           // 外层块
    // 进入内层块
    %sub = memref.subview %A[0] [16] [1] : memref<16xf32> to memref<16xf32>
    %src = memref.alloc() : memref<16xf32>
    write_to(%src)
    memref.copy %src, %sub : memref<16xf32> to memref<16xf32>
    use(%src)
    ```
  - 优化后：
    ```mlir
    %A = memref.alloc() : memref<16xf32>
    // 进入内层块
    %sub = memref.subview %A[0] [16] [1] : memref<16xf32> to memref<16xf32>
    write_to(%sub)
    use(%sub)
    ```
  - 行为依据：`RemoveRedundantCopy.cpp:154-175`
- 示例 2：目标为分配的缓冲（用源替换目标）
  - 优化前：
    ```mlir
    %src = memref.alloc() : memref<4xf32>
    %dst = memref.alloc() : memref<4xf32>
    write_to(%src)
    memref.copy %src, %dst : memref<4xf32> to memref<4xf32>
    return %dst : memref<4xf32>
    ```
  - 优化后：
    ```mlir
    %src = memref.alloc() : memref<4xf32>
    write_to(%src)
    return %src : memref<4xf32>
    ```
  - 行为依据：`RemoveRedundantCopy.cpp:237-245`
- 示例 3：目标为函数参数（用目标替换源）
  - 优化前：
    ```mlir
    func.func @f(%dst: memref<4xf32>) {
      %src = memref.alloc() : memref<4xf32>
      write_to(%src)
      memref.copy %src, %dst : memref<4xf32> to memref<4xf32>
      memref.dealloc %src : memref<4xf32>
      return
    }
    ```
  - 优化后：
    ```mlir
    func.func @f(%dst: memref<4xf32>) {
      write_to(%dst)
      return
    }
    ```
  - 行为依据：`RemoveRedundantCopy.cpp:247-258`

**使用方式**

- 命令行：`mlir-opt input.mlir -memref-remove-redundant-copy`
- C++ 管线：调用 `mlir::memref::createRemoveRedundantCopyPass()`。

**相关代码**

- 声明与注册：`bishengir/include/bishengir/Dialect/MemRef/Transforms/Passes.td:41-47`
- 入口与遍历：`bishengir/lib/Dialect/MemRef/Transforms/RemoveRedundantCopy.cpp:262-274`
- 子视图重定向：`RemoveRedundantCopy.cpp:138-175`
- 目标为分配的缓冲替换：`RemoveRedundantCopy.cpp:237-245`
- 目标为函数参数替换：`RemoveRedundantCopy.cpp:247-258`

# Pass 定义文件：`Dialect_SCF_Transforms_Passes.td`

## map-for-to-forall

**Pass 作用**

- 将带有标记的 `scf.for` 映射为 `scf.forall`，用于并行化循环到设备块维度。
- 要求源 `scf.for` 上存在属性 `map_for_to_forall`（`UnitAttr`）。可选地读取或注入设备映射属性 `mapping`（默认使用 `hivm::HIVMBlockMappingAttr`）。
- 核心转换通过工具函数 `scf::utils::mapForToForallImpl` 完成，并用新生成的 `scf.forall` 替换原有 `scf.for`。
- 代码依据：
  - 声明：`bishengir/include/bishengir/Dialect/SCF/Transforms/Passes.td:24-31`
  - 实现入口与匹配条件：`bishengir/lib/Dialect/SCF/Transforms/MapForToForall.cpp:47-75, 77-87`

**触发条件与行为**

- 仅转换具备 `map_for_to_forall` 属性的 `scf.for`。
- 若已有 `ArrayAttr` 类型的 `mapping` 属性，则使用之；否则注入默认的块映射属性。
- 转换生成的 `scf.forall` 支持并行语义，便于后续并行代码生成或设备映射。

**简单示例**

- 示例 1：最简并行映射
  - 转换前：
    ```mlir
    // scf.for 带标记
    %lb = arith.constant 0 : index
    %ub = arith.constant 1024 : index
    %step = arith.constant 1 : index
    scf.for (%i) = (%lb) to (%ub) step (%step) attributes {map_for_to_forall} {
      // loop body using %i
      scf.yield
    }
    ```
  - 转换后（概念化展示，实际 `scf.forall` 需携带结果/资源等语义）：
    ```mlir
    %lb = arith.constant 0 : index
    %ub = arith.constant 1024 : index
    %step = arith.constant 1 : index
    scf.forall (%i) in (%lb to %ub step %step) attributes {mapping = [#hivm.block]} {
      // parallel body using %i
      scf.in_parallel {
        // reductions or side effects aggregation if any
      }
    }
    ```
  - 说明：默认注入 `mapping = [hivm::HIVMBlockMappingAttr]`，用于设备块级并行。
- 示例 2：自定义设备映射
  - 转换前：
    ```mlir
    scf.for (%i) = (%lb) to (%ub) step (%step)
      attributes {map_for_to_forall, mapping = [#hivm.block, #hivm.thread]} {
      body
      scf.yield
    }
    ```
  - 转换后：
    ```mlir
    scf.forall (%i) in (%lb to %ub step %step)
      attributes {mapping = [#hivm.block, #hivm.thread]} {
      body
      scf.in_parallel { }
    }
    ```

**使用方式**

- 命令行：`mlir-opt input.mlir -map-for-to-forall`
- C++ 管线：调用 `mlir::scf::createMapForToForallPass()`。

**相关代码**

- Pass 定义与注册：`bishengir/include/bishengir/Dialect/SCF/Transforms/Passes.td:21-31`
- 转换逻辑与属性处理：`bishengir/lib/Dialect/SCF/Transforms/MapForToForall.cpp:53-75`

## scf-remove-redundant-loop-init

**Pass 作用**

- 移除 `scf.for` 的冗余迭代初值，当初值内容不会影响循环结果时，用 `tensor.empty` 替换初值以减少不必要的初始化与数据依赖。
- 针对迭代参数是 `tensor` 类型的情形，识别并优化“按步覆盖整块”的 `tensor.insert_slice` 模式。
- 代码依据：
  - 声明：`bishengir/include/bishengir/Dialect/SCF/Transforms/Passes.td:33-40`
  - 实现：`bishengir/lib/Dialect/SCF/Transforms/RemoveRedundantLoopInit.cpp:226-286`

**判定冗余的关键条件**

- 初值未在循环体中被直接使用：`RemoveRedundantLoopInit.cpp:159-164`
- 迭代参数唯一使用为 `tensor.insert_slice`，其结果直接作为该迭代参数的 `yield`：`RemoveRedundantLoopInit.cpp:173-193`
- 插入切片沿某唯一维度覆盖整个目标 `tensor`：
  - 步长为 1：`RemoveRedundantLoopInit.cpp:123-131`
  - 插入偏移等于归纳变量：`RemoveRedundantLoopInit.cpp:201-205`
  - 插入大小等于循环步长：`RemoveRedundantLoopInit.cpp:207-211`
  - 下界为 0，上界等于目标张量该维度大小：`RemoveRedundantLoopInit.cpp:212-221`

**变换效果**

- 将该迭代参数的初值替换为 `tensor.empty`，消除无意义的初始数据填充：`RemoveRedundantLoopInit.cpp:259-266`

**简单示例**

- 优化前：
  ```mlir
  %sz = arith.constant 8 : index
  %step = arith.constant 1 : index
  %init = some_tensor_value : tensor<8xf32>  // 内容不影响最终结果
  scf.for (%i) = (%c0) to (%sz) step (%step)
      iter_args(%t = %init) -> (tensor<8xf32>) {
    %upd = tensor.insert_slice %slice into %t[%i][1][1]
           : tensor<1xf32> into tensor<8xf32>
    scf.yield %upd : tensor<8xf32>
  }
  ```
- 优化后：
  ```mlir
  %empty = tensor.empty() : tensor<8xf32>
  scf.for (%i) = (%c0) to (%sz) step (%step)
      iter_args(%t = %empty) -> (tensor<8xf32>) {
    %upd = tensor.insert_slice %slice into %t[%i][1][1]
           : tensor<1xf32> into tensor<8xf32>
    scf.yield %upd : tensor<8xf32>
  }
  ```

**使用方式**

- 命令行：`mlir-opt input.mlir -scf-remove-redundant-loop-init`
- C++ 管线：`mlir::scf::createRemoveRedundantLoopInitPass()`。

**相关代码引用**

- Pass 声明行：`bishengir/include/bishengir/Dialect/SCF/Transforms/Passes.td:33-40`
- 判定与替换逻辑：`bishengir/lib/Dialect/SCF/Transforms/RemoveRedundantLoopInit.cpp:156-223, 259-266`

## scf-canonicalize-iter-arg

**Pass 作用**

- 规范化并精简 `scf` 循环的迭代参数（iter args）：
  - 当某迭代参数在所有迭代中保持不变时，用其初值替换相关别名与结果，去除不必要的数据传递。
  - 当某迭代参数的对应循环结果未被使用，并且 `yield` 对该迭代参数的生成与循环体独立（无副作用、仅依赖体内可删除算子且不逃逸）时，删除该迭代参数与相关循环体内计算，重建一个不包含该迭代参数的新 `scf.for`。
  - 同时应用常见子表达式消除与 `tensor.empty` 折叠以帮助进一步简化。
- 适用范围：`func::FuncOp` 内的 `scf.for` 与 `scf.while`。
- 代码依据：
  - 声明：`bishengir/include/bishengir/Dialect/SCF/Transforms/Passes.td:42-48`
  - 实现：`bishengir/lib/Dialect/SCF/Transforms/CanonicalizeIterArg.cpp:372-391`

**关键逻辑**

- 不变迭代参数检测：通过在循环与条件分支内做等价性追踪，若 `yield` 的值始终等价于 `init`，则替换所有别名为初值：`CanonicalizeIterArg.cpp:73-140, 244-268`。
- 死迭代参数删除：当某循环结果无用且其 `yield` 计算仅依赖体内的无副作用算子，并且这些算子只用于该 `yield` 链，则标记为可删除，构造一个移除该迭代参数的新循环并替换原循环：`CanonicalizeIterArg.cpp:162-224, 273-369`。
- 额外优化：消除常见子表达式、折叠 `tensor.empty` 模式以简化 IR：`CanonicalizeIterArg.cpp:239-243, 378-383`.

**简单示例**

- 示例 1：不变迭代参数折叠
  - 优化前：
    ```mlir
    %init = arith.constant 42 : i32
    scf.for (%i) = (%c0) to (%c10) step (%c1) iter_args(%x = %init) -> (i32) {
      // %x 始终被原样 yield
      scf.yield %x : i32
    }
    %res = ... use(%x_result) ...
    ```
  - 优化后：
    ```mlir
    %init = arith.constant 42 : i32
    // %x_result 的所有使用替换为 %init
    %res = ... use(%init) ...
    // 若循环结果仅为 %x_result 且不再使用，可进一步重写或删除该结果
    ```
- 示例 2：删除无用迭代参数与其生成链
  - 优化前：
    ```mlir
    scf.for (%i) = (%c0) to (%N) step (%c1)
      iter_args(%t = %init_tensor) -> (tensor<64xf32>, i32) {
      %upd = some_pure_ops(%t, %i)
      scf.yield %upd, %i : tensor<64xf32>, i32
    }
    // 只使用第二个结果 i32，tensor 结果未使用
    ```
  - 优化后（移除未使用的 tensor 迭代参数与生成链）：
    ```mlir
    scf.for (%i) = (%c0) to (%N) step (%c1)
      iter_args(/* 只保留 i32 或其他仍需的迭代参数 */) -> (i32) {
      // 删除与 tensor 结果相关的纯算子
      scf.yield %i : i32
    }
    ```

**使用方式**

- 命令行：`mlir-opt input.mlir -scf-canonicalize-iter-arg`
- C++ 管线：`mlir::scf::createCanonicalizeIterArgPass()`。

**相关代码引用**

- 不变性检测与替换：`bishengir/lib/Dialect/SCF/Transforms/CanonicalizeIterArg.cpp:73-140, 244-268`
- 死迭代参数删除与循环重建：`bishengir/lib/Dialect/SCF/Transforms/CanonicalizeIterArg.cpp:162-224, 303-369`
- Pass 入口与模式注册：`bishengir/lib/Dialect/SCF/Transforms/CanonicalizeIterArg.cpp:372-386`

# Pass 定义文件：`Dialect_Symbol_Transforms_Passes.td`

## propagate-symbol

**Pass 作用**

- 在函数级别传播与统一符号化的动态形状信息：
  - 为具有动态维的张量结果绑定符号形状，自动从可重构形状接口的重化信息中生成并插入 `symbol.bind_symbolic_shape`。
  - 将 `tensor.dim` 等索引运算替换为已绑定的符号（`symbol.symbolic_int`），使动态维度通过符号贯穿表达式链。
  - 把动态尺寸上的算术索引（如用于 `tensor.empty`、`tensor.expand_shape`、`tensor.extract_slice` 的 `arith` 常量/计算）符号化为 `symbol.symbolic_int` 并替换使用点。
  - 对具有相同或部分等价维度约束的运算（如广播、归约、拼接），统一各参与值的符号，使同一维的符号在 IR 中一致，减少冗余符号与形状不一致。
- 目标：让动态维的依赖通过符号一致表达，便于后续优化、融合、以及静态检查。
- 代码依据：
  - 声明行：`bishengir/include/bishengir/Dialect/Symbol/Transforms/Passes.td:23`
  - 主要实现与模式：
    - 重化结果形状并绑定符号：`bishengir/lib/Dialect/Symbol/Transforms/PropagateSymbol.cpp:98-189`
    - 用已绑定符号替换 `tensor.dim`：`PropagateSymbol.cpp:221-266`
    - 符号化动态形状索引（empty/expand/extract\_slice）：`PropagateSymbol.cpp:270-321`
    - 统一符号（同形状操作/部分维等价）：`PropagateSymbol.cpp:323-448, 450-560`

**触发与行为细节**

- 对实现 `ReifyRankedShapedTypeOpInterface` 的操作，获取输出形状的重化信息，针对动态维生成绑定（可能基于 `tensor.dim` 或 `affine.apply`），插入 `symbol.bind_symbolic_shape`。
- 若某维为动态，且存在已绑定符号，则在使用 `tensor.dim` 获取该维时直接替换为符号值。
- 对初始化动态形状张量的 `index` 型操作数（若由 `arith` 运算产生），创建 `symbol.symbolic_int` 并替换，使尺寸成为符号源。
- 在具有相同形状的操作数与结果间，逐维收集符号并统一为块内最靠前的符号，消除重复符号。
- 对广播、归约、拼接等存在部分维映射的操作，依据维映射仅统一等价维的符号。

**简单示例**

- 示例 1：绑定与传播动态形状符号
  - 变换前：
    ```mlir
    %arg0: tensor<?x640xf16>
    %dim0 = tensor.dim %arg0, %c0
    %empty = tensor.empty(%dim0, %c640) : tensor<?x640xf16>
    %add = linalg.elemwise_binary ins(%arg0, %arg1) outs(%empty)
    ```
  - 变换后（符号 `S0` 表示第一维）：
    ```mlir
    %S0 = symbol.symbolic_int @S0
    symbol.bind_symbolic_shape %arg0, [%S0], affine_map<()[s0] -> (s0, 640)>
    %empty = tensor.empty(%S0, %c640) : tensor<?x640xf16>
    %add = linalg.elemwise_binary ins(%arg0, %arg1) outs(%empty)
    symbol.bind_symbolic_shape %add, [%S0], affine_map<()[s0] -> (s0, 640)>
    ```
  - 行为依据：`PropagateSymbol.cpp:98-189, 221-266`
- 示例 2：符号化 `arith` 尺寸并替换使用
  - 变换前：
    ```mlir
    %n = arith.addi %a, %b : index
    %empty = tensor.empty(%n, %c640) : tensor<?x640xf32>
    ```
  - 变换后：
    ```mlir
    %Sx = symbol.symbolic_int @Sx(%n)  // 从 %n 派生的符号
    %empty = tensor.empty(%Sx, %c640) : tensor<?x640xf32>
    ```
  - 行为依据：`PropagateSymbol.cpp:270-321`
- 示例 3：统一同形状操作数与结果的符号
  - 变换前：
    ```mlir
    // %A, %B, %C 形状皆为 tensor<?x640xf16>，各自绑定了不同符号 S0/S1/S2
    %out = linalg.elemwise_binary ins(%A, %B) outs(%C)
    ```
  - 变换后（统一为块内最靠前的符号，如 S0）：
    ```mlir
    // 将 S1、S2 的使用替换为 S0
    // 所有与第一维相关的绑定与使用统一到 S0
    ```
  - 行为依据：`PropagateSymbol.cpp:323-348, 386-426`

**使用方式**

- 命令行：`mlir-opt input.mlir -propagate-symbol`
- C++ 管线：构造器在生成的头文件中，通常以 `mlir::symbol::createPropagateSymbolPass()` 命名；该 Pass 作用域为 `func::FuncOp`。

**相关测试与参考**

- 测试覆盖绑定与传播：`bishengir/test/Dialect/Symbol/propagate.mlir`
- 符号操作定义：`bishengir/lib/Dialect/Symbol/IR/SymbolOps.cpp`

## erase-symbol

**Pass 作用**

- 移除符号相关 IR（如 `symbol.symbolic_int`、`symbol.bind_symbolic_shape` 等）及其影响，将函数体恢复为不依赖符号机制的形式。
- 适用于在符号分析/传播阶段结束后，需要清理符号层的场景；或在下游不支持符号方言时作为过渡。
- 声明与构造器：
  - `Pass` 名称：`erase-symbol`
  - 构造函数：`mlir::symbol::createEraseSymbolPass()`
  - 位置：`bishengir/include/bishengir/Dialect/Symbol/Transforms/Passes.td:81-87`

**典型行为**

- 删除 `symbol.bind_symbolic_shape`，并把下游使用的维度索引恢复为原始 `tensor.dim` 或常量/算术表达式（具体恢复策略取决于前置 Pass 的输出与实现）。
- 移除 `symbol.symbolic_int`，并用可计算的维度表达式或 `tensor.dim` 替换其所有使用点。
- 保证在移除过程中不破坏类型与形状一致性；若存在无法确定的符号使用，通常保留必要的 `tensor.dim` 重化或发出诊断。

**简单示例**

- 清理前（含符号层）：
  ```mlir
  %S0 = symbol.symbolic_int @S0 : index
  symbol.bind_symbolic_shape %arg0, [%S0], affine_map<()[s0] -> (s0, 640)>
  %empty = tensor.empty(%S0, %c640) : tensor<?x640xf16>
  %add = linalg.elemwise_binary ins(%arg0, %arg1) outs(%empty)
  symbol.bind_symbolic_shape %add, [%S0], affine_map<()[s0] -> (s0, 640)>
  ```
- 清理后（移除符号与绑定，回退到 `tensor.dim` 或常量尺寸）：
  ```mlir
  %c0 = arith.constant 0 : index
  %dim0 = tensor.dim %arg0, %c0 : tensor<?x640xf16>
  %empty = tensor.empty(%dim0, %c640) : tensor<?x640xf16>
  %add = linalg.elemwise_binary ins(%arg0, %arg1) outs(%empty)
  ```

**使用方式**

- 命令行：`mlir-opt input.mlir -erase-symbol`
- C++ 管线：`mlir::symbol::createEraseSymbolPass()`。

**补充说明**

- 与 `propagate-symbol`、`symbol-to-encoding`、`encoding-to-symbol`、`unfold-symbolic-int` 等 Pass 配合使用，通常流程为：传播/统一符号 → 编码或展开 → 清理符号。
- 若 IR 中的符号承载了复杂 `affine` 映射或跨值统一后的合并符号，清理时需确保降级后的表达仍可被后续 Pass 正确消费。

## symbol-to-encoding

**Pass 作用**

- 把符号绑定（`symbol.bind_symbolic_shape`）转换为张量类型上的编码信息，从而不再需要符号方言的运行时绑定，动态维以类型属性形式携带。
- 典型目标是将符号形状从操作级绑定迁移到类型层面（如 `tensor` 的 `encoding`），便于下游仅基于类型信息进行推理与优化。
- 声明与构造器：
  - `Pass` 名称：`symbol-to-encoding`
  - 构造函数：`mlir::symbol::createSymbolToEncodingPass()`
  - 定义位置：`bishengir/include/bishengir/Dialect/Symbol/Transforms/Passes.td:89-97`

**直观行为**

- 遍历模块，找到所有 `symbol.bind_symbolic_shape`，读取其绑定的符号列表与 `affine_map`。
- 将这些信息编码进对应张量类型的 `encoding`，并删除原有 `bind_symbolic_shape`。
- 若存在 `symbol.symbolic_int` 仅用于形状绑定，也可通过该转换间接实现“使用点不再依赖符号”，配合其他 Pass（如 `erase-symbol` 或 `unfold-symbolic-int`）共同完成符号层的剥离。

**简单示例**

- 转换前（使用符号绑定表达动态维）：
  ```mlir
  %S0 = symbol.symbolic_int @S0 : index
  %empty = tensor.empty(%S0, %c640) : tensor<?x640xf16>
  symbol.bind_symbolic_shape %empty, [%S0], affine_map<()[s0] -> (s0, 640)>
  ```
- 转换后（将绑定转为类型编码，移除符号绑定）：
  ```mlir
  // 伪示例：类型的 encoding 携带了对第一维的符号/仿射表达
  %empty = tensor.empty(%S0, %c640) : tensor<?x640xf16, encoding = <affine_map<()[s0] -> (s0, 640)>>>
  // 原来的 symbol.bind_symbolic_shape 被删除
  ```
- 注：具体 `encoding` 的语法取决于实现与注册的类型属性格式，示例用法展示意图而非精确语法。

**与其它 Pass 的关系**

- 上游：`propagate-symbol` 将动态维符号化并绑定到值；`symbol-to-encoding` 将绑定迁移到类型。
- 下游：`encoding-to-symbol` 可将类型编码反向还原为符号绑定；`erase-symbol` 可在迁移完成后清理符号方言；`unfold-symbolic-int` 可把符号替换为可计算的 `tensor.dim` 等。

**使用方式**

- 命令行：`mlir-opt input.mlir -symbol-to-encoding`
- C++ 管线：`mlir::symbol::createSymbolToEncodingPass()`。

## encoding-to-symbol

**Pass 作用**

- 将类型上的 `encoding` 信息转换回符号绑定形式：把 `RankedTensorType` 的 `encoding` 属性还原为 `symbol.bind_symbolic_shape`，并移除类型上的 `encoding`，恢复纯张量类型。
- 目的：在需要基于符号进行进一步分析或与依赖符号的 Pass 协作时，从类型编码回到显式的符号绑定表示。
- 声明与构造器：
  - `Pass` 名称：`encoding-to-symbol`
  - 构造函数：`mlir::symbol::createEncodingToSymbolPass()`
  - 定义位置：`bishengir/include/bishengir/Dialect/Symbol/Transforms/Passes.td:99-107`
- 实现位置与关键模式：`bishengir/lib/Dialect/Symbol/Transforms/SymbolAndEncodingConverter.cpp:317-348`

**关键逻辑**

- 遍历模块，找到带 `encoding` 的 `RankedTensorType`：
  - 对函数参数：生成对应的 `symbol.bind_symbolic_shape`，并修改函数签名，将参数类型的 `encoding` 去除，同时更新块参数类型以匹配新签名。参考 `HandleFuncOp`：`SymbolAndEncodingConverter.cpp:226-266`。
  - 对一般操作的结果：在结果值之后插入 `symbol.bind_symbolic_shape`，并将结果类型改为去掉 `encoding` 的张量类型。参考 `ConvertTensorEncodeToBindSymbolicShape`：`SymbolAndEncodingConverter.cpp:267-296`。
- 绑定生成规则：
  - `encoding` 是一个 `ArrayAttr`，其中每个维度编码要么是常量（`IntegerAttr`），要么是符号名（`FlatSymbolRefAttr`）。
  - 为每个符号名在当前模块中查找对应的 `symbol.symbolic_int` 定义，并用其结果值作为 `bind_symbolic_shape` 的 `shapeSymbols`。
  - 构造 `AffineMap`，常量维对应 `AffineConstantExpr`，符号维对应递增的 `AffineSymbolExpr`，并插入 `bind_symbolic_shape`。参考 `addBindSymbolicShapeOp`：`SymbolAndEncodingConverter.cpp:190-224`。
- 类型一致性：
  - 对 `func.call` 和 `func.return` 插入必要的 `tensor.cast`，保证调用实参与返回值在类型变化后仍匹配函数签名与使用点。参考 `InsertCastForEncoding` 与 `HandleReturnOp`：`SymbolAndEncodingConverter.cpp:68-107, 165-188`。

**简单示例**

- 转换前（类型带 `encoding`）：
  ```mlir
  func.func @f(%arg0: tensor<?x640xf16, encoding = [@S0, 640]>) -> tensor<?x640xf16, encoding = [@S0, 640]> {
    %0 = "some_op"(%arg0) : (tensor<?x640xf16, encoding = [@S0, 640]>) -> tensor<?x640xf16, encoding = [@S0, 640]>
    func.return %0 : tensor<?x640xf16, encoding = [@S0, 640]>
  }
  ```
- 转换后（生成绑定，去除 `encoding`）：
  ```mlir
  // 假定模块内已有：%S0 = symbol.symbolic_int @S0 : index
  func.func @f(%arg0: tensor<?x640xf16>) -> tensor<?x640xf16> {
    // 参数绑定
    symbol.bind_symbolic_shape %arg0, [%S0], affine_map<()[s0] -> (s0, 640)> : tensor<?x640xf16>
    %0 = "some_op"(%arg0) : (tensor<?x640xf16>) -> tensor<?x640xf16>
    // 结果绑定
    symbol.bind_symbolic_shape %0, [%S0], affine_map<()[s0] -> (s0, 640)> : tensor<?x640xf16>
    func.return %0 : tensor<?x640xf16>
  }
  ```
- 若调用点类型不匹配，会自动插入 `tensor.cast` 以桥接：
  ```mlir
  %x = func.call @f(%y) : (tensor<?x640xf16, encoding = [@S0, 640]>) -> tensor<?x640xf16, encoding = [@S0, 640]>
  // 转换后在调用点插入 cast 以适配新签名
  ```

**使用方式**

- 命令行：`mlir-opt input.mlir -encoding-to-symbol`
- C++ 管线：`mlir::symbol::createEncodingToSymbolPass()`。

**配套 Pass**

- 与 `symbol-to-encoding` 相反方向；可配合 `propagate-symbol` 先将动态维符号化，再按需切换表示。

## unfold-symbolic-int

**Pass 作用**

- 将符号表示展开为具体的计算与维度查询：
  - 用 `affine.apply` 展开 `symbol.symbolic_int` 的仿射表达，将其所有使用替换为具体的仿射计算结果。
  - 用 `tensor.dim` 展开 `symbol.bind_symbolic_shape` 的符号绑定，将绑定的符号替换为来自源张量的维度查询，并删除绑定。
- 适用范围：`func::FuncOp`，Pass 名称 `-unfold-symbolic-int`。
- 代码依据：
  - 声明与入口：`bishengir/include/bishengir/Dialect/Symbol/Transforms/Passes.td:109-157`
  - 展开实现：`bishengir/lib/Dialect/Symbol/Transforms/UnfoldSymbolicInt.cpp:53-167`

**关键逻辑**

- 对 `symbol.symbolic_int`：
  - 若其 `int_expressions` 非空（仿射映射不为空），在符号位置插入 `affine.apply`，操作数为其 `int_symbols`，并替换所有使用；随后删除该 `symbolic_int`：`UnfoldSymbolicInt.cpp:83-99`。
- 对 `symbol.bind_symbolic_shape`：
  - 遍历其 `shape_expressions` 的每个结果，仅处理符号或常量（要求映射为“符号或常量”形式，通常等价于恒等映射）；
  - 对对应的符号输入，若来源为 `symbol.symbolic_int`，就在被绑定的源值之后插入 `tensor.dim %src, %idx`，用它替换该符号所有使用，并删除该 `symbolic_int`；
  - 最后删除该 `bind_symbolic_shape`：`UnfoldSymbolicInt.cpp:104-167`。

**简单示例**

- 展开符号仿射表达
  - 展开前：
    ```mlir
    %S1 = symbol.symbolic_int @S1 [%S0], affine_map<()[s0] -> (s0 + 1)> : index
    %empty = tensor.empty(%S1) : tensor<?xf32>
    ```
  - 展开后：
    ```mlir
    %applied = affine.apply ()[%S0] -> (s0 + 1)
    %empty = tensor.empty(%applied) : tensor<?xf32>
    ```
- 展开绑定为维度查询
  - 展开前：
    ```mlir
    %S0 = symbol.symbolic_int @S0 : index
    symbol.bind_symbolic_shape %arg0, [%S0], affine_map<()[s0] -> (s0, 640)> : tensor<?x640xf32>
    %empty = tensor.empty(%S0, %c640) : tensor<?x640xf32>
    ```
  - 展开后：
    ```mlir
    %c0 = arith.constant 0 : index
    %dim0 = tensor.dim %arg0, %c0 : tensor<?x640xf32>
    %empty = tensor.empty(%dim0, %c640) : tensor<?x640xf32>
    ```
  - 若 `bind_symbolic_shape` 中符号来源不是 `symbol.symbolic_int`（例如直接来自 `tensor.dim` 或 `affine.apply`），则该符号输入不在此 Pass 展开范围内，会跳过；绑定本身在处理结束时被删除。

**使用方式**

- 命令行：`mlir-opt input.mlir -unfold-symbolic-int`
- C++ 管线：`mlir::symbol::createUnfoldSymbolicIntPass()`。

**注意事项**

- 要求 `bind_symbolic_shape` 的 `shape_expressions` 结果为“符号或常量”（常见于恒等映射）；复杂表达式需先通过其他 Pass（如 `propagate-symbol`）约化。
- 展开后会删除符号相关操作，IR 将回退为标准的 `tensor.dim` 与 `affine.apply` 表示，适合后续不依赖符号方言的优化与代码生成。

# Pass 定义文件：`Dialect_Tensor_Transforms_Passes.td`

## canonicalize-tensor-reshape

**Pass 作用**

- 对 `tensor.reshape` 进行规范化与折叠：
  - 将某些 `tensor.reshape` 规范化为更基础的 `tensor.expand_shape` 或 `tensor.collapse_shape`，使重排关系显式、便于后续优化。
  - 当源/目标形状在静态乘积与动态维数量上可推断等价时，把一次 `reshape` 分解为“先 `expand_shape` 后 `collapse_shape`”，并替换原有 `reshape`。
- 适用范围：`func::FuncOp`，Pass 名称 `-canonicalize-tensor-reshape`。
- 代码依据：
  - 声明：`bishengir/include/bishengir/Dialect/Tensor/Transforms/Passes.td:24-31`
  - 实现：`bishengir/lib/Dialect/Tensor/Transforms/CanonicalizeTensorReshape.cpp:39-205`

**核心规则**

- 静态 shape 参数时的规范化：
  - 若 `shape` 是静态并且源为 1 维，则把 `reshape` 规范化为 `expand_shape`：`CanonicalizeTensorReshape.cpp:103-109, 73-86`。
  - 若 `shape` 的维度是 1 且值为 1（目标一维展开），则把 `reshape` 规范化为 `collapse_shape`（把多维源折叠为 1 维）：`CanonicalizeTensorReshape.cpp:109-115, 47-71`。
- 可推断等价时的折叠：
  - 当源/目标 `RankedTensorType` 动态维数量相同且静态乘积一致时，计算松散重关联关系，生成一对兼容的 `expand_shape` 与 `collapse_shape`，并用它们替代 `reshape`：`CanonicalizeTensorReshape.cpp:117-191`。

**简单示例**

- 示例 1：源为 1 维，规范化为 `expand_shape`
  - 变换前：
    ```mlir
    %shape = arith.constant dense<[2, 3]> : tensor<2xi64>
    %r = tensor.reshape %v, %shape
         : (tensor<6xf32>, tensor<2xi64>) -> tensor<2x3xf32>
    ```
  - 变换后：
    ```mlir
    %r = tensor.expand_shape %v [[0, 1]]
         : tensor<6xf32> into tensor<2x3xf32>
    ```
- 示例 2：目标为 1 维（值为 1），规范化为 `collapse_shape`
  - 变换前：
    ```mlir
    %shape = arith.constant dense<[1]> : tensor<1xi64>
    %r = tensor.reshape %v, %shape
         : (tensor<2x3xf32>, tensor<1xi64>) -> tensor<1xf32>
    ```
  - 变换后：
    ```mlir
    %r = tensor.collapse_shape %v [[0, 1]]
         : tensor<2x3xf32> into tensor<1xf32>
    ```
- 示例 3：折叠为 expand+collapse
  - 变换前：
    ```mlir
    %r = tensor.reshape %v, %shape
         : (tensor<2x?xf32>, tensor<2xi64>) -> tensor<?x2xf32>
    ```
  - 条件满足（动态维数量一致且静态乘积相等）后，变换为：
    ```mlir
    %e = tensor.expand_shape %v [[0], [1]] : tensor<2x?xf32> into tensor<2x?xf32>
    %c = tensor.collapse_shape %e [[0, 1]]  : tensor<2x?xf32> into tensor<?x2xf32>
    // 用 %c 替换原 %r
    ```

**使用方式**

- 命令行：`mlir-opt input.mlir -canonicalize-tensor-reshape`
- C++ 管线：`mlir::tensor::createCanonicalizeTensorReshapePass()`。

**相关代码引用**

- 规范化判定与重写：`bishengir/lib/Dialect/Tensor/Transforms/CanonicalizeTensorReshape.cpp:88-115`
- expand+collapse 分解与替换：`CanonicalizeTensorReshape.cpp:117-191`
- Pass 注册与入口：`CanonicalizeTensorReshape.cpp:193-205`

## fold-tensor-empty

**Pass 作用**

- 折叠和简化围绕 `tensor.empty` 的构造与使用模式，移除无用的空张量创建或将其合并到更简洁的形式，减少 IR 噪音并为下游优化创造条件。
- 工作在 `func::FuncOp` 上，Pass 名称为 `-fold-tensor-empty`。
- 代码依据：
  - 声明：`bishengir/include/bishengir/Dialect/Tensor/Transforms/Passes.td:32-40`
  - 实现入口：`bishengir/lib/Dialect/Tensor/Transforms/FoldTensorEmpty.cpp:41-51`
  - 模式来源：`mlir::tensor::populateFoldTensorEmptyPatterns(patterns)`（`FoldTensorEmpty.cpp:44`）

**典型优化**

- 将 `tensor.empty` 直接内联到使用者的初始化路径，例如 `linalg.fill`、`linalg.elemwise_*` 的输出初始化，避免单独的空张量创建。
- 对 `tensor.pack/unpack`、`tensor.pad` 等操作，如果空张量仅作为中间容器且能从上下文推断整形，则消除或合并空张量创建。
- 对等价尺寸的空张量重复创建，统一其使用，减少冗余。

**简单示例**

- 示例 1：将空张量折叠到填充操作中
  - 优化前：
    ```mlir
    %empty = tensor.empty() : tensor<16x16xf32>
    %filled = linalg.fill ins(%cst : f32) outs(%empty : tensor<16x16xf32>) -> tensor<16x16xf32>
    ```
  - 优化后：
    ```mlir
    %filled = linalg.fill ins(%cst : f32) outs(tensor.empty() : tensor<16x16xf32>) -> tensor<16x16xf32>
    ```
  - 进一步的上游 Pass 可能会消除内联的 `tensor.empty`，直接生成更规范的填充。
- 示例 2：折叠 `tensor.pack` 的空目标
  - 优化前：
    ```mlir
    %dest = tensor.empty() : tensor<8x16x8x32xf32>
    %packed = tensor.pack %src
      padding_value(%pad : f32)
      outer_dims_perm = [1, 0] inner_dims_pos = [0, 1]
      inner_tiles = [8, 32]
      into %dest : tensor<63x127xf32> -> tensor<8x16x8x32xf32>
    ```
  - 优化后（移除显式 `%dest` 并内联或规范化其创建）：
    ```mlir
    %packed = tensor.pack %src
      padding_value(%pad : f32)
      outer_dims_perm = [1, 0] inner_dims_pos = [0, 1]
      inner_tiles = [8, 32]
      into tensor.empty() : tensor<8x16x8x32xf32>
      : tensor<63x127xf32> -> tensor<8x16x8x32xf32>
    ```

**使用方式**

- 命令行：`mlir-opt input.mlir -fold-tensor-empty`
- C++ 管线：`mlir::tensor::createFoldTensorEmptyPass()`。

**补充**

- 该 Pass依赖 upstream 的 `tensor` 变换模式集合；如果你需要具体的哪些折叠在你的 IR 触发，可参考本仓库中的测试用例如 `bishengir/test/Dialect/Tensor/canonicalize.mlir` 中的相关场景（例如 pack/pad/cast 场景）。

## normalize-tensor-ops

**作用**

- 统一并优化常见 `tensor` 运算模式，减少冗余与便于后续转换
- 核心包含 5 类重写：
  - 折叠“insert\_slice 后再 pad”为一次 `pad`（要求目的张量为常值或 `linalg.fill` 且 pad 值一致）
  - 折叠连续两次 `pad` 为一次（要求两次 `pad` 的低/高填充量为静态并且填充值相同）
  - 将最后一维上的 `tensor.concat` 规范化为 `hfusion.interleave`（当前仅支持通道数为 2，且被拼接的最后维度大小均为 1）
  - 将常量体的 `tensor.generate` 变换为 `linalg.fill`
  - 把“高维静态负数 padding”的 `tensor.pad`改写为 `tensor.extract_slice`（仅当所有 high 为非正且源形状静态、low 为全 0 时）

实现位置：

- 定义：`bishengir/include/bishengir/Dialect/Tensor/Transforms/Passes.td:40`
- 代码：`bishengir/lib/Dialect/Tensor/Transforms/NormalizeTensorOps.cpp`

**简单示例**

- insert\_slice + pad 折叠
  - 之前：
    ```mlir
    %fill = linalg.fill ins(%cst : f32) outs(%dst : tensor<6140xf32>)
    %ins  = tensor.insert_slice %src into %fill[2046] [2047] [1]
             : tensor<2047xf32> into tensor<6140xf32>
    %pad  = tensor.pad %ins low[0] high[-2047] {pad_value = %cst}
             : tensor<6140xf32> to tensor<4093xf32>
    ```
  - 之后：
    ```mlir
    %pad = tensor.pad %src low[2046] high[0] {pad_value = %cst}
           : tensor<2047xf32> to tensor<4093xf32>
    ```
- 双 pad 折叠
  - 之前：
    ```mlir
    %p0 = tensor.pad %src low[2046] high[2047] {pad_value = %cst}
          : tensor<2047xf32> to tensor<6140xf32>
    %p1 = tensor.pad %p0  low[0]    high[-2047] {pad_value = %cst}
          : tensor<6140xf32> to tensor<4093xf32>
    ```
  - 之后：
    ```mlir
    %p = tensor.pad %src low[2046] high[0] {pad_value = %cst}
         : tensor<2047xf32> to tensor<4093xf32>
    ```
- 最后一维 concat → interleave（通道数为 2 且最后维度为 1）
  - 之前：
    ```mlir
    %out = tensor.concat dim(2) %A, %B
           : (tensor<8x8x1xf32>, tensor<8x8x1xf32>) -> tensor<8x8x2xf32>
    ```
  - 之后：
    ```mlir
    %out = hfusion.interleave %A, %B : (tensor<8x8x1xf32>, tensor<8x8x1xf32>)
    ```
- generate 常量体 → fill
  - 之前：
    ```mlir
    %g = tensor.generate %shape {  // 单语句 body：yield %cst
      ^bb0(%i0: index): 
        tensor.yield %cst : f32
    } : tensor<?xf32>
    ```
  - 之后：
    ```mlir
    %e = tensor.empty(%dyn) : tensor<?xf32>
    %f = linalg.fill ins(%cst) outs(%e) : f32, tensor<?xf32>
    ```
- 高维静态负数 high padding 折叠为切片
  - 之前：
    ```mlir
    %p = tensor.pad %src low[0,0] high[0,-2] {pad_value = %cst}
         : tensor<16x10xf32> to tensor<16x8xf32>
    ```
  - 之后：
    ```mlir
    %slice = tensor.extract_slice %src[0,0] [16,8] [1,1]
             : tensor<16x10xf32> to tensor<16x8xf32>
    ```

**限制与提示**

- interleave 目前仅支持 2 个输入；concat 轴必须是最后一维且各输入该维大小为 1
- pad 折叠要求静态 low/high 与静态偏移，且填充值一致
- 负数 high 折叠不支持混合正负或动态大小的情况（文件中标注 TODO）

## narrow-tensor-ops

**作用**

- 缩小 `tensor` 运算的存活范围，尤其是把“在循环体内才被使用”的 `tensor.empty` 和 `tensor.extract_slice` 从循环外部移动到循环内部，减少不必要的生命周期与占用
- 仅当这些值的所有使用者都在某个循环内，且值本身不在该循环内时才生效；否则保持不变
- 关键实现：
  - 为 `tensor::EmptyOp` 和 `tensor::ExtractSliceOp` 添加重写（`bishengir/lib/Dialect/Tensor/Transforms/NarrowTensorOp.cpp:84-86`）
  - 找到所有使用者的共同祖先循环并在其循环体开头克隆该运算（`NarrowTensorOpPattern::matchAndRewrite`，`bishengir/lib/Dialect/Tensor/Transforms/NarrowTensorOp.cpp:47-59`）
  - 仅当“存在候选循环且该运算不在该循环内”时执行（`bishengir/lib/Dialect/Tensor/Transforms/NarrowTensorOp.cpp:49-51`）；若任何使用者不在循环内，直接放弃（`bishengir/lib/Dialect/Tensor/Transforms/NarrowTensorOp.cpp:63-79`）
  - 旧值的所有使用被替换为克隆后的新值，旧值通常由后续 DCE 清理（`bishengir/lib/Dialect/Tensor/Transforms/NarrowTensorOp.cpp:58`）
- Pass 定义：`bishengir/include/bishengir/Dialect/Tensor/Transforms/Passes.td:48`，构造器为 `mlir::tensor::createNarrowTensorOpPass()`；入口实现：`bishengir/lib/Dialect/Tensor/Transforms/NarrowTensorOp.cpp:81-93`

**简单示例**

- 将循环内才使用的 `tensor.empty` 下沉到循环体
  - 之前：
    ```mlir
    %n = arith.constant 128 : index
    %c0 = arith.constant 0 : index
    %c1 = arith.constant 1 : index
    %zero = arith.constant 0.0 : f32
    %dst = tensor.empty(%n) : tensor<?xf32>

    scf.for %i = %c0 to %n step %c1 {
      %filled = linalg.fill ins(%zero) outs(%dst) : f32, tensor<?xf32>
      // 使用 %filled ...
    }
    ```
  - 之后：
    ```mlir
    %n = arith.constant 128 : index
    %c0 = arith.constant 0 : index
    %c1 = arith.constant 1 : index
    %zero = arith.constant 0.0 : f32

    scf.for %i = %c0 to %n step %c1 {
      %dst_local = tensor.empty(%n) : tensor<?xf32>
      %filled = linalg.fill ins(%zero) outs(%dst_local) : f32, tensor<?xf32>
      // 使用 %filled ...
    }
    ```
- 将循环内才使用的 `tensor.extract_slice` 下沉到循环体
  - 之前：
    ```mlir
    %big = "make_tensor"() : () -> tensor<128xf32>
    %off = arith.constant 0 : index
    %sz  = arith.constant 64 : index
    %slice = tensor.extract_slice %big[%off] [%sz] [1]
             : tensor<128xf32> to tensor<64xf32>

    scf.for %i = %off to %sz step %c1 {
      %init = tensor.empty() : tensor<64xf32>
      %out  = linalg.elemwise_unary ins(%slice) outs(%init)
              -> tensor<64xf32>
      // 使用 %out ...
    }
    ```
  - 之后：
    ```mlir
    %big = "make_tensor"() : () -> tensor<128xf32>
    %off = arith.constant 0 : index
    %sz  = arith.constant 64 : index

    scf.for %i = %off to %sz step %c1 {
      %slice_local = tensor.extract_slice %big[%off] [%sz] [1]
                     : tensor<128xf32> to tensor<64xf32>
      %init = tensor.empty() : tensor<64xf32>
      %out  = linalg.elemwise_unary ins(%slice_local) outs(%init)
              -> tensor<64xf32>
      // 使用 %out ...
    }
    ```

**注意点**

- 若有任何使用者不在循环中，或该值本身已在该循环体内，Pass 不会改变（`NarrowTensorOp.cpp:49-52, 63-69`）
- 此 Pass 仅负责“下沉并替换使用”，不改变循环迭代参数或返回值的语义；多余的旧值由后续 DCE 清理
- 目标 Dialect 依赖：`tensor` 与 `scf`（`Passes.td:51-53`）

## propagate-reshape

**定位**

- 该 Pass 定义为 `PropagateReshape`，位置在 `bishengir/include/bishengir/Dialect/Tensor/Transforms/Passes.td:56-83`
- 摘要与详细说明见 `Passes.td:57-72`
- 选项 `forHIVM` 定义于 `Passes.td:74-77`，构造器与依赖见 `Passes.td:78-83`

**作用**

- 识别当 `reshape` 的输入由某个 `linalg` 元素级运算产生时，将该运算“穿透”到 `reshape` 之前
- 将 `reshape` 下推到该运算的所有输入，然后在重塑后的形状上重新执行该元素级运算
- 用新的计算结果替代原来 `reshape` 的所有使用，从而消除中间的 `reshape`
- 主要收益：减少不必要的 `reshape`、简化数据流，利于后续融合、向量化与代码生成

**触发条件**

- `reshape` 的来源是 `linalg` 的元素级运算（如 `linalg.generic` 的逐元素计算）
- 形状变换保持元素数不变，且能为该运算在新形状上构造一致的索引映射
- 若目标后端有特殊行为（HIVM），可通过选项进行相应的传播策略调整

**简单示例（前后对比）**

- 重塑从二维 `tensor<2x3xf32>` 折叠为一维 `tensor<6xf32>`，将 `reshape` 下推到输入，再在一维上做逐元素加法

原始形态（`reshape` 在运算之后）：

```mlir
func.func @propagate_example(%x: tensor<2x3xf32>, %y: tensor<2x3xf32>) -> tensor<6xf32> {
  %out0 = tensor.empty() : tensor<2x3xf32>
  %sum = linalg.generic
      { indexing_maps = [affine_map<(i,j)->(i,j)>,
                         affine_map<(i,j)->(i,j)>,
                         affine_map<(i,j)->(i,j)>],
        iterator_types = ["parallel", "parallel"] }
      ins(%x, %y : tensor<2x3xf32>, tensor<2x3xf32>)
      outs(%out0 : tensor<2x3xf32>) {
    ^bb0(%a: f32, %b: f32, %acc: f32):
      %c = arith.addf %a, %b : f32
      linalg.yield %c : f32
  } -> tensor<2x3xf32>

  %r = tensor.reshape %sum [[0, 1]]
       : tensor<2x3xf32> into tensor<6xf32>
  return %r
}
```

传播后形态（`reshape` 下推到输入，消除中间重塑）：

```mlir
func.func @propagate_example(%x: tensor<2x3xf32>, %y: tensor<2x3xf32>) -> tensor<6xf32> {
  %rx = tensor.reshape %x [[0, 1]]
        : tensor<2x3xf32> into tensor<6xf32>
  %ry = tensor.reshape %y [[0, 1]]
        : tensor<2x3xf32> into tensor<6xf32>

  %out1 = tensor.empty() : tensor<6xf32>
  %sum1d = linalg.generic
      { indexing_maps = [affine_map<(i)->(i)>,
                         affine_map<(i)->(i)>,
                         affine_map<(i)->(i)>],
        iterator_types = ["parallel"] }
      ins(%rx, %ry : tensor<6xf32>, tensor<6xf32>)
      outs(%out1 : tensor<6xf32>) {
    ^bb0(%a: f32, %b: f32, %acc: f32):
      %c = arith.addf %a, %b : f32
      linalg.yield %c : f32
  } -> tensor<6xf32>

  return %sum1d
}
```

**使用方式**

- 命令行运行（已注册文本名为 `propagate-reshape`）：

```bash
mlir-opt --propagate-reshape input.mlir
# 或使用管线语法（在 func 上运行）
mlir-opt -pass-pipeline='func(propagate-reshape)' input.mlir
```

- 在 C++ 中加入到 `PassManager`：

```cpp
mlir::PassManager pm(&context);
pm.addPass(mlir::tensor::createPropagateReshapePass());
```

**选项与依赖**

- 选项：`for-hivm`（默认 `false`），用于有 HIVM 特殊行为的场景，可通过管线语法开启：

```bash
mlir-opt -pass-pipeline='func(propagate-reshape{for-hivm=true})' input.mlir
```

- 依赖方言：`func::FuncDialect`、`tensor::TensorDialect`（见 `Passes.td:79-83`）

## trickle-concat-down

**定位**

- Pass 名称与位置：`TrickleConcatDown`，位于 `bishengir/include/bishengir/Dialect/Tensor/Transforms/Passes.td:85`
- 实现文件：`bishengir/lib/Dialect/Tensor/Transforms/TrickleConcatDown.cpp`
  - 元素一元传播规则：`bishengir/lib/Dialect/Tensor/Transforms/TrickleConcatDown.cpp:42-117`
  - Slice 传播规则：`bishengir/lib/Dialect/Tensor/Transforms/TrickleConcatDown.cpp:119-213`
  - Pass 入口：`bishengir/lib/Dialect/Tensor/Transforms/TrickleConcatDown.cpp:215-237`

**作用**

- 将 `tensor.concat` “下推”越过其唯一使用者，使 `concat` 发生在更靠后的位置：
  - 当使用者是元素级一元、且为 destination-style 的算子时，把该一元算子应用到每个 `concat` 输入，再对结果执行 `concat`，消除原一元算子
  - 当使用者是 `tensor.extract_slice`（静态切片参数）时，在每个输入上计算对应的切片，再将这些切片按同一维度重新 `concat`，替换原先对 `concat` 结果的切片
- 目的：减少在大张量上的操作，把计算尽量分摊到各输入，便于后续融合、布局优化与代码生成

**关键条件**

- 仅处理 `tensor.concat` 的“单一使用”场景（`hasOneUse`）`bishengir/lib/Dialect/Tensor/Transforms/TrickleConcatDown.cpp:58-63,129-132`
- 元素一元规则：
  - 使用者需被标记为元素级一元算子，且为 `DestinationStyleOpInterface` `TrickleConcatDown.cpp:62-63,76-88`
  - 仅在 `concat` 维度为“最后一维”时触发 `TrickleConcatDown.cpp:64-66`
  - 若各输入最后一维的大小都“32 对齐”，则不触发（针对“最后维不对齐”的场景）`TrickleConcatDown.cpp:67-75`
- Slice 规则：
  - 使用者为 `tensor.extract_slice`，且 `concat` 维度上的 offsets/sizes/strides 为静态 `TrickleConcatDown.cpp:136-154`
  - 所有输入在 `concat` 维度上必须是静态大小 `TrickleConcatDown.cpp:136-141`
  - 计算每个输入的切片区间并保证总切片长度能被完全覆盖，否则失败 `TrickleConcatDown.cpp:156-185`

**示例 1（元素一元算子下推）**
原始形态（先 `concat`，后一元运算）：

```mlir
func.func @unary_after_concat(%x: tensor<4x8xf32>, %y: tensor<4x6xf32>) -> tensor<4x14xf32> {
  %c = tensor.concat dim(1) %x, %y : (tensor<4x8xf32>, tensor<4x6xf32>) -> tensor<4x14xf32>
  %out = tensor.empty() : tensor<4x14xf32>
  %r = linalg.elemwise_unary {fun = #linalg.unary_fn<exp>}
       ins(%c : tensor<4x14xf32>) outs(%out : tensor<4x14xf32>) -> tensor<4x14xf32>
  return %r : tensor<4x14xf32>
}
```

传播后（先对每个输入做一元运算，再 `concat`，去掉原一元算子）：

```mlir
func.func @unary_after_concat(%x: tensor<4x8xf32>, %y: tensor<4x6xf32>) -> tensor<4x14xf32> {
  %ox = tensor.empty() : tensor<4x8xf32>
  %ux = linalg.elemwise_unary {fun = #linalg.unary_fn<exp>}
        ins(%x : tensor<4x8xf32>) outs(%ox : tensor<4x8xf32>) -> tensor<4x8xf32>
  %oy = tensor.empty() : tensor<4x6xf32>
  %uy = linalg.elemwise_unary {fun = #linalg.unary_fn<exp>}
        ins(%y : tensor<4x6xf32>) outs(%oy : tensor<4x6xf32>) -> tensor<4x6xf32>
  %r = tensor.concat dim(1) %ux, %uy : (tensor<4x8xf32>, tensor<4x6xf32>) -> tensor<4x14xf32>
  return %r : tensor<4x14xf32>
}
```

对应实现逻辑参考：

- 条件检查与元素一元判定：`bishengir/lib/Dialect/Tensor/Transforms/TrickleConcatDown.cpp:62-76`
- 克隆使用者到各输入并替换 `concat` 操作数：`bishengir/lib/Dialect/Tensor/Transforms/TrickleConcatDown.cpp:81-113`
- 用新的 `concat` 结果替换旧使用者并删除之：`bishengir/lib/Dialect/Tensor/Transforms/TrickleConcatDown.cpp:114-116`

**示例 2（切片下推）**
原始形态（对 `concat` 结果切片）：

```mlir
func.func @slice_after_concat(%x: tensor<4x8xf32>, %y: tensor<4x6xf32>) -> tensor<4x4xf32> {
  %c = tensor.concat dim(1) %x, %y : (tensor<4x8xf32>, tensor<4x6xf32>) -> tensor<4x14xf32>
  %s = tensor.extract_slice %c [0, 5] [4, 4] [1, 1]
       : tensor<4x14xf32> to tensor<4x4xf32>
  return %s : tensor<4x4xf32>
}
```

传播后（先在各输入上切片，再 `concat` 拼接切片结果，移除原 `concat`）：

```mlir
func.func @slice_after_concat(%x: tensor<4x8xf32>, %y: tensor<4x6xf32>) -> tensor<4x4xf32> {
  %sx = tensor.extract_slice %x [0, 5] [4, 3] [1, 1]
        : tensor<4x8xf32> to tensor<4x3xf32>
  %sy = tensor.extract_slice %y [0, 0] [4, 1] [1, 1]
        : tensor<4x6xf32> to tensor<4x1xf32>
  %r = tensor.concat dim(1) %sx, %sy
       : (tensor<4x3xf32>, tensor<4x1xf32>) -> tensor<4x4xf32>
  return %r : tensor<4x4xf32>
}
```

对应实现逻辑参考：

- 静态参数检查与输入维度静态性检查：`bishengir/lib/Dialect/Tensor/Transforms/TrickleConcatDown.cpp:136-154`
- 逐输入计算 offsets/sizes 并构造切片：`bishengir/lib/Dialect/Tensor/Transforms/TrickleConcatDown.cpp:156-203`
- 以切片集合重新构造 `concat` 替换原切片，并删除旧 `concat`：`bishengir/lib/Dialect/Tensor/Transforms/TrickleConcatDown.cpp:205-210`

**使用方式**

- 命令行：

```bash
mlir-opt --trickle-concat-down input.mlir
mlir-opt -pass-pipeline='func(trickle-concat-down)' input.mlir
```

- 在 C++ 中加入到 `PassManager`：

```cpp
mlir::PassManager pm(&context);
pm.addPass(mlir::tensor::createTrickleConcatDownPass());
```

**注意事项**

- 仅处理单一使用者的 `concat`，多处使用需要先做简化或拷贝分裂
- 元素一元场景限定在“最后一维”且“非 32 对齐”的输入形状，意在优化末维不对齐的数据路径
- Slice 场景要求相关切片参数与输入尺寸在目标维度上为静态，便于按输入分配切片范围并保证总覆盖量一致

## bubble-pad-up

**定位**

- Pass 名称与位置：`BubblePadUp`，位于 `bishengir/include/bishengir/Dialect/Tensor/Transforms/Passes.td:93-99`
- 实现文件与入口：
  - 重写规则：`bishengir/lib/Dialect/Tensor/Transforms/BubblePadUp.cpp:48-98`
  - 执行入口：`bishengir/lib/Dialect/Tensor/Transforms/BubblePadUp.cpp:105-114`
  - 构造器：`bishengir/lib/Dialect/Tensor/Transforms/BubblePadUp.cpp:118-120`

**作用**

- 将 `tensor.pad` 沿数据流“上浮”（bubble up）到其源运算之前：当 `pad` 的输入是“元素级一元”运算（如 `linalg.elemwise_unary`）的结果时，把同样的 `pad` 施加到该一元运算的所有相关输入/输出操作数上，再在“已填充”的形状上执行该一元运算
- 主要收益：
  - 避免对大张量结果做后置填充，将填充分摊到更小的输入上
  - 有利于后续融合与调度，尤其是最后一维不对齐场景的布局与代码生成优化

**触发条件**

- `pad` 源是被标记的“元素级一元”算子 `bishengir/lib/Dialect/Tensor/Transforms/BubblePadUp.cpp:61-64`
- 仅处理“最后一维被填充”的情况 `bishengir/lib/Dialect/Tensor/Transforms/BubblePadUp.cpp:66-71`
- 最后一维的低位填充量不得为 32 对齐，否则不触发 `bishengir/lib/Dialect/Tensor/Transforms/BubblePadUp.cpp:72-76`
- 重写时会克隆 `pad` 的 region，保持填充值与语义一致 `bishengir/lib/Dialect/Tensor/Transforms/BubblePadUp.cpp:88-91`

**示例（前后对比）**

- 设输入为 `tensor<16x27xf32>`，在末维做低 3 高 2 的填充，填充值为常量 0

原始形态（先做一元运算，再对结果填充）：

```mlir
func.func @bubble_pad_up(%A: tensor<16x27xf32>) -> tensor<16x32xf32> {
  %out0 = tensor.empty() : tensor<16x27xf32>
  %u = linalg.elemwise_unary {fun = #linalg.unary_fn<exp>}
       ins(%A : tensor<16x27xf32>)
       outs(%out0 : tensor<16x27xf32>) -> tensor<16x27xf32>

  %c0 = arith.constant 0.0 : f32
  %padded = tensor.pad %u low[0, 3] high[0, 2] {
    ^bb0(%i0: index, %i1: index):
      tensor.yield %c0 : f32
  } : tensor<16x27xf32> to tensor<16x32xf32>

  return %padded : tensor<16x32xf32>
}
```

传播后形态（将 `pad` 上浮到一元运算的输入与输出操作数，再在填充后的形状上计算）：

```mlir
func.func @bubble_pad_up(%A: tensor<16x27xf32>) -> tensor<16x32xf32> {
  %c0 = arith.constant 0.0 : f32

  %A_pad = tensor.pad %A low[0, 3] high[0, 2] {
    ^bb0(%i0: index, %i1: index):
      tensor.yield %c0 : f32
  } : tensor<16x27xf32> to tensor<16x32xf32>

  %out0 = tensor.empty() : tensor<16x27xf32>
  %out_pad = tensor.pad %out0 low[0, 3] high[0, 2] {
    ^bb0(%i0: index, %i1: index):
      tensor.yield %c0 : f32
  } : tensor<16x27xf32> to tensor<16x32xf32>

  %u2 = linalg.elemwise_unary {fun = #linalg.unary_fn<exp>}
        ins(%A_pad : tensor<16x32xf32>)
        outs(%out_pad : tensor<16x32xf32>) -> tensor<16x32xf32>

  return %u2 : tensor<16x32xf32>
}
```

说明：上述“上浮”会在所有与源形状一致的操作数上插入同样的 `pad`，并保持原 `pad` 的填充值 region。

**使用方式**

- 命令行：

```bash
mlir-opt --bubble-pad-up input.mlir
mlir-opt -pass-pipeline='func(bubble-pad-up)' input.mlir
```

- C++ 集成：

```cpp
mlir::PassManager pm(&context);
pm.addPass(mlir::tensor::createBubblePadUpPass());
```

**依赖**

- 依赖方言：`tensor::TensorDialect`（见 `/include/.../Passes.td:96-99`）
- 模式注册与执行：`applyPatternsGreedily` 在 `func::FuncOp` 上运行 `bishengir/lib/Dialect/Tensor/Transforms/BubblePadUp.cpp:105-114`

## normalize-last-dim-unaligned-tensor-op

**定位**

- 位置与定义：`bishengir/include/bishengir/Dialect/Tensor/Transforms/Passes.td:101-149`
- 详细描述与示例：`Passes.td:102-143`
- 主要实现：`bishengir/lib/Dialect/Tensor/Transforms/NormalizeLastDimUnalignedTensorOp.cpp`
  - Concat 归一化：`NormalizeLastDimUnalignedTensorOp.cpp:221-264`
  - Pad 归一化：`NormalizeLastDimUnalignedTensorOp.cpp:349-375`
  - 1-D 特例处理（扩维再还原）：`NormalizeLastDimUnalignedTensorOp.cpp:318-341`（concat）、`NormalizeLastDimUnalignedTensorOp.cpp:113-140`（pad）
  - 转置与映射辅助：`NormalizeLastDimUnalignedTensorOp.cpp:46-73,142-152,295-306`

**作用**

- 规范化“最后一维不对齐”的 `tensor.concat` 与 `tensor.pad`：
  - 若 `concat` 沿最后一维，且该维不满足对齐约束，则对输入与输出执行转置，使拼接轴变为首维，再在首维上 `concat`，最后转置回原布局
  - 若 `pad` 在最后一维进行填充，且不满足对齐约束，则将最后一维转置到首维，在首维上进行 `pad`，再转置回原布局
- 目标：避免在末维不对齐场景下的硬件不友好访问，改善后续融合、调度与代码生成

**触发条件**

- Concat：拼接维为最后一维，且被判定为“不对齐”（例如输入的末维尺寸不满足 32 对齐）`NormalizeLastDimUnalignedTensorOp.cpp:223-233,266-276`
- Pad：最后一维存在静态填充（动态填充暂不处理），且低位填充量不满足对齐约束 `NormalizeLastDimUnalignedTensorOp.cpp:351-365`
- 1-D 情况：通过广播扩展到 2-D，再进行转置与归一化，最后用切片还原 1-D `NormalizeLastDimUnalignedTensorOp.cpp:318-341,113-140`

**示例（Concat 归一化）**
原始（沿最后一维拼接）：

```mlir
%0 = tensor.concat dim(1) %A, %B
     : (tensor<16x32xf32>, tensor<16x64xf32>) -> tensor<16x96xf32>
```

归一化后（将拼接轴移到首维，再转回）：

```mlir
%tmpA = tensor.empty() : tensor<32x16xf32>
%tA = linalg.transpose ins(%A : tensor<16x32xf32>)
                      outs(%tmpA : tensor<32x16xf32>) permutation = [1, 0]
%tmpB = tensor.empty() : tensor<64x16xf32>
%tB = linalg.transpose ins(%B : tensor<16x64xf32>)
                      outs(%tmpB : tensor<64x16xf32>) permutation = [1, 0]
%cat0 = tensor.concat dim(0) %tA, %tB
        : (tensor<32x16xf32>, tensor<64x16xf32>) -> tensor<96x16xf32>
%tmpO = tensor.empty() : tensor<16x96xf32>
%out = linalg.transpose ins(%cat0 : tensor<96x16xf32>)
                     outs(%tmpO : tensor<16x96xf32>) permutation = [1, 0]
```

**示例（Pad 归一化）**
原始（最后一维填充）：

```mlir
%p = tensor.pad %A low[0, 3] high[0, 2] { ... }
     : tensor<16x27xf32> to tensor<16x32xf32>
```

归一化后（将填充轴移到首维，再转回）：

```mlir
%tmpA = tensor.empty() : tensor<27x16xf32>
%tA = linalg.transpose ins(%A : tensor<16x27xf32>)
                      outs(%tmpA : tensor<27x16xf32>) permutation = [1, 0]
%p2 = tensor.pad %tA low[3, 0] high[2, 0] { ... }
      : tensor<27x16xf32> to tensor<32x16xf32>
%tmpO = tensor.empty() : tensor<16x32xf32>
%out = linalg.transpose ins(%p2 : tensor<32x16xf32>)
                     outs(%tmpO : tensor<16x32xf32>) permutation = [1, 0]
```

## bubble-up-extract-slice

**定位**

- 定义位置：`bishengir/include/bishengir/Dialect/Tensor/Transforms/Passes.td:151-162`
- 实现入口：`bishengir/lib/Dialect/Tensor/Transforms/BubbleUpExtractSlice.cpp:40-48`
- 选项定义：`Passes.td:156-161`（`aggressive`）
- 构造器：`bishengir/lib/Dialect/Tensor/Transforms/BubbleUpExtractSlice.cpp:50-53`

**作用**

- 将对结果的 `tensor.extract_slice` “上浮”（bubble up）到其产生者之前，在输入侧进行切片，直接计算小片段的结果
- 典型效果：把“大张量上计算→再切片”的模式改为“先切片输入→在小张量上计算”，减少不必要的数据处理与内存访问
- 同时应用 `tensor::populateFoldTensorEmptyPatterns`，简化与空张量相关的构造

**触发条件**

- 使用了上游 `linalg` 的 Bubble-Up 规则，适用于多种 `linalg` 算子（尤其是逐元素或索引映射为置换的场景）
- 默认仅当切片的源值使用次数较少或可安全上浮；开启 `aggressive` 时，即使源值有多个使用也会尝试上浮（可能导致更多克隆）

**示例 1：逐元素一元运算**
原始（先算后切）：

```mlir
func.func @bubble_unary(%A: tensor<8x16xf32>) -> tensor<8x8xf32> {
  %out = tensor.empty() : tensor<8x16xf32>
  %r = linalg.elemwise_unary {fun = #linalg.unary_fn<exp>}
       ins(%A : tensor<8x16xf32>) outs(%out : tensor<8x16xf32>) -> tensor<8x16xf32>
  %s = tensor.extract_slice %r [0, 4] [8, 8] [1, 1]
       : tensor<8x16xf32> to tensor<8x8xf32>
  return %s : tensor<8x8xf32>
}
```

上浮后（先切输入，再算小片段）：

```mlir
func.func @bubble_unary(%A: tensor<8x16xf32>) -> tensor<8x8xf32> {
  %A_s = tensor.extract_slice %A [0, 4] [8, 8] [1, 1]
         : tensor<8x16xf32> to tensor<8x8xf32>
  %out_s = tensor.empty() : tensor<8x8xf32>
  %r_s = linalg.elemwise_unary {fun = #linalg.unary_fn<exp>}
        ins(%A_s : tensor<8x8xf32>) outs(%out_s : tensor<8x8xf32>) -> tensor<8x8xf32>
  return %r_s : tensor<8x8xf32>
}
```

**示例 2：逐元素二元运算**
原始：

```mlir
%out = tensor.empty() : tensor<4x10xf32>
%sum = linalg.generic
  { indexing_maps = [affine_map<(i,j)->(i,j)>,
                     affine_map<(i,j)->(i,j)>,
                     affine_map<(i,j)->(i,j)>],
    iterator_types = ["parallel","parallel"] }
  ins(%X, %Y : tensor<4x10xf32>, tensor<4x10xf32>)
  outs(%out : tensor<4x10xf32>) {
  ^bb0(%a: f32, %b: f32, %acc: f32):
    %c = arith.addf %a, %b : f32
    linalg.yield %c : f32
} -> tensor<4x10xf32>
%s = tensor.extract_slice %sum [0, 3] [4, 4] [1, 1]
     : tensor<4x10xf32> to tensor<4x4xf32>
```

上浮后：

```mlir
%X_s = tensor.extract_slice %X [0, 3] [4, 4] [1, 1]
       : tensor<4x10xf32> to tensor<4x4xf32>
%Y_s = tensor.extract_slice %Y [0, 3] [4, 4] [1, 1]
       : tensor<4x10xf32> to tensor<4x4xf32>
%out_s = tensor.empty() : tensor<4x4xf32>
%sum_s = linalg.generic
  { indexing_maps = [affine_map<(i,j)->(i,j)>,
                     affine_map<(i,j)->(i,j)>,
                     affine_map<(i,j)->(i,j)>],
    iterator_types = ["parallel","parallel"] }
  ins(%X_s, %Y_s : tensor<4x4xf32>, tensor<4x4xf32>)
  outs(%out_s : tensor<4x4xf32>) {
  ^bb0(%a: f32, %b: f32, %acc: f32):
    %c = arith.addf %a, %b : f32
    linalg.yield %c : f32
} -> tensor<4x4xf32>
```

**使用方式**

- 命令行：

```bash
mlir-opt --bubble-up-extract-slice input.mlir
mlir-opt -pass-pipeline='func(bubble-up-extract-slice{aggressive=true})' input.mlir
```

- C++ 集成：

```cpp
mlir::PassManager pm(&context);
mlir::tensor::BubbleUpExtractSliceOptions opts;
opts.aggressive = true; // 可选
pm.addPass(mlir::tensor::createBubbleUpExtractSlicePass(opts));
```

**选项**

- `aggressive`（默认 `false`）：`Passes.td:156-161`
  - `false`：谨慎上浮，避免影响有多重使用的值
  - `true`：更激进，上浮切片即便源值存在多处使用，可能增加克隆与后续融合机会

**实现细节参考**

- 模式来源：`linalg::populateBubbleUpExtractSliceOpPatterns` `bishengir/lib/.../BubbleUpExtractSlice.cpp:43-46`
- 同步折叠：`tensor::populateFoldTensorEmptyPatterns` `bishengir/lib/.../BubbleUpExtractSlice.cpp:46`
- 应用策略：贪心重写 `applyPatternsGreedily` `bishengir/lib/.../BubbleUpExtractSlice.cpp:47`

## merge-consecutive-insert-extract-slice

**定位**

- 定义位置：`bishengir/include/bishengir/Dialect/Tensor/Transforms/Passes.td:164-170`
- 实现入口：`bishengir/lib/Dialect/Tensor/Transforms/MergeConsecutiveInsertExtractSlice.cpp:37-47`
- 模式来源：`tensor::populateMergeConsecutiveInsertExtractSlicePatterns` `MergeConsecutiveInsertExtractSlice.cpp:40`

**作用**

- 合并或消除“连续的切片插入/提取”链：
  - 将 `tensor.extract_slice` 后接 `tensor.insert_slice` 等等价重组为更简单的单次操作，或直接替换为原值
  - 典型场景：先从大张量提取子片段，接着把该片段插回到原张量同位置；或多个连续提取/插入在同一范围上进行，存在冗余
- 目标：减少无效数据搬运、降低 IR 噪声，便于后续融合和代码生成

**示例 1：提取后原位插入（消除）**
原始：

```mlir
%big = ... : tensor<4x8xf32>
%sub = tensor.extract_slice %big [0, 2] [4, 4] [1, 1]
       : tensor<4x8xf32> to tensor<4x4xf32>
%res = tensor.insert_slice %sub into %big [0, 2] [4, 4] [1, 1]
       : tensor<4x4xf32> into tensor<4x8xf32>
```

合并后：

```mlir
%res = %big : tensor<4x8xf32>
```

解释：提取出的子片段被原位插入回同一位置，无净效应，直接替换为原值。

**示例 2：连续同区间提取（折叠）**
原始：

```mlir
%a = tensor.extract_slice %big [0, 2] [4, 4] [1, 1]
     : tensor<4x8xf32> to tensor<4x4xf32>
%b = tensor.extract_slice %a [0, 1] [4, 2] [1, 1]
     : tensor<4x4xf32> to tensor<4x2xf32>
```

合并后（计算合并后的偏移与大小，直接一次提取）：

```mlir
%b = tensor.extract_slice %big [0, 3] [4, 2] [1, 1]
     : tensor<4x8xf32> to tensor<4x2xf32>
```

**使用方式**

- 命令行：

```bash
mlir-opt --merge-consecutive-insert-extract-slice input.mlir
mlir-opt -pass-pipeline='func(merge-consecutive-insert-extract-slice)' input.mlir
```

- C++ 集成：

```cpp
mlir::PassManager pm(&context);
pm.addPass(mlir::tensor::createMergeConsecutiveInsertExtractSlicePass());
```

**实现细节**

- 采用 `applyPatternsGreedily` 在 `func::FuncOp` 上应用预置的模式集合
- 具体规则集来自 `mlir::tensor` 的标准变换库，涵盖等价折叠与范围合并等场景

## decompose-tensor-concat

**定位**

- 定义位置：`bishengir/include/bishengir/Dialect/Tensor/Transforms/Passes.td:172-176`
- 实现文件：`bishengir/lib/Dialect/Tensor/Transforms/DecomposeTensorConcat.cpp`
  - 模式填充：`DecomposeTensorConcat.cpp:35-39`
  - 构造器：`DecomposeTensorConcat.cpp:43-45`
- 上游补丁（后移逻辑到 `ConcatOp::decomposeOperation`）：`/build-tools/patches/llvm-project/0041-[Backport]-Move-concat-operation-decomposition-as-a-method-of-th.patch:23-70, 75-137`

**作用**

- 将一个 `tensor.concat` 分解为一串 `tensor.insert_slice` 到一个 `tensor.empty` 的序列，行为等价：
  - 先计算输出形状（在拼接维度上依次累加各输入的尺寸）
  - 创建目标空张量 `tensor.empty`
  - 按拼接维度的偏移逐个把输入通过 `tensor.insert_slice` 插入到空张量中
  - 必要时在末尾进行 `tensor.cast` 以匹配原 `concat` 的结果类型
- 价值：把复杂的 `concat` 变为更基础的切片插入，便于后续优化（如切片推移、冗余合并、调度与代码生成）

**示例 1：二维静态拼接**
原始：

```mlir
%0 = tensor.concat dim(1) %A, %B
     : (tensor<8x4xf32>, tensor<8x6xf32>) -> tensor<8x10xf32>
```

分解后：

```mlir
%c0 = arith.constant 0 : index
%c1 = arith.constant 1 : index
%empty = tensor.empty() : tensor<8x10xf32>
%t0 = tensor.insert_slice %A into %empty [0, 0] [8, 4] [1, 1]
      : tensor<8x4xf32> into tensor<8x10xf32>
%t1 = tensor.insert_slice %B into %t0 [0, 4] [8, 6] [1, 1]
      : tensor<8x6xf32> into tensor<8x10xf32>
```

**示例 2：含动态尺寸的拼接（需要** **`cast`）**
原始：

```mlir
%0 = tensor.concat dim(1) %A, %B
     : (tensor<8x4xf32>, tensor<?x?xf32>) -> tensor<?x?xf32>
```

分解后（根据补丁中的实现）：

```mlir
%c0 = arith.constant 0 : index
%c1 = arith.constant 1 : index
%d0 = tensor.dim %B, %c0 : tensor<?x?xf32>
%d1 = tensor.dim %B, %c1 : tensor<?x?xf32>
%sum = affine.apply affine_map<()[s0] -> (s0 + 4)>()[%d1]
%empty = tensor.empty(%d0, %sum) : tensor<8x?xf32>
%s0 = tensor.insert_slice %A into %empty [0, 0] [8, 4] [1, 1]
      : tensor<8x4xf32> into tensor<8x?xf32>
%s1 = tensor.insert_slice %B into %s0 [0, 4] [%d0, %d1] [1, 1]
      : tensor<?x?xf32> into tensor<8x?xf32>
%res = tensor.cast %s1 : tensor<8x?xf32> to tensor<?x?xf32>
```

**使用方式**

- 命令行：

```bash
mlir-opt --decompose-tensor-concat input.mlir
mlir-opt -pass-pipeline='func(decompose-tensor-concat)' input.mlir
```

- C++ 集成：

```cpp
mlir::PassManager pm(&context);
pm.addPass(mlir::tensor::createDecomposeTensorConcatPass());
```

**实现细节**

- 模式集合由 `tensor::populateDecomposeTensorConcatPatterns` 提供
- 在本工程中已经回迁上游补丁，将分解逻辑作为 `ConcatOp` 的方法实现，匹配时直接调用 `concatOp.decomposeOperation` 完成替换

## optimize-dps-op-with-yielded-insert-slice

**定位**

- 定义位置：`bishengir/include/bishengir/Dialect/Tensor/Transforms/Passes.td:178-187`
- 实现文件：`bishengir/lib/Dialect/Tensor/Transforms/OptimizeDpsOpWithYieldedInsertSlice.cpp`
  - 规则类与说明：`OptimizeDpsOpWithYieldedInsertSlice.cpp:52-66, 70-117`
  - 支配性检查与抽取切片：`OptimizeDpsOpWithYieldedInsertSlice.cpp:90-110`
  - 将 `dps` 初始化改为切片：`OptimizeDpsOpWithYieldedInsertSlice.cpp:111-115`
  - Pass 注册与执行：`OptimizeDpsOpWithYieldedInsertSlice.cpp:121-132`

**作用**

- 优化“destination-style”算子在循环中产出的局部结果被 `tensor.insert_slice` 插入并 `yield` 的模式
- 将该算子的 `init` 操作数，从原来的 `tensor.empty` 改为对迭代参数（整体结果）的对应区域做 `tensor.extract_slice`
- 目的：为一体化缓冲化（one-shot-bufferize）创造“先抽取再插入”的成对访问，使该区域可原位复用，避免额外拷贝与缓冲

**触发条件**

- `tensor.insert_slice` 的 `source` 来自 `DestinationStyleOpInterface`（如 `linalg.generic`、`linalg.fill` 等）`OptimizeDpsOpWithYieldedInsertSlice.cpp:77-82`
- 该 `dps` 的绑定 `init` 是 `tensor.empty`，没有既有写入目标 `OptimizeDpsOpWithYieldedInsertSlice.cpp:86-89`
- `insert_slice` 的 offsets/sizes/strides 的定义在支配关系上不晚于该 `dps`（可安全在其前构造 `extract_slice`）`OptimizeDpsOpWithYieldedInsertSlice.cpp:90-103`

**简单示例（前后对比）**
原始形态（先算到空张量，再插入并 `yield`）：

```mlir
func.func @opt_example(%init: tensor<8x8xf32>, %X: tensor<4x4xf32>, %Y: tensor<4x4xf32>) -> tensor<8x8xf32> {
  %c0 = arith.constant 0 : index
  %c1 = arith.constant 1 : index
  %res = scf.for %i = %c0 to %c1 step %c1 iter_args(%acc = %init) -> tensor<8x8xf32> {
    %out = tensor.empty() : tensor<4x4xf32>
    %sum = linalg.generic
      { indexing_maps = [affine_map<(i,j)->(i,j)>,
                         affine_map<(i,j)->(i,j)>,
                         affine_map<(i,j)->(i,j)>],
        iterator_types = ["parallel","parallel"] }
      ins(%X, %Y : tensor<4x4xf32>, tensor<4x4xf32>)
      outs(%out : tensor<4x4xf32>) {
      ^bb0(%a: f32, %b: f32, %accv: f32):
        %c = arith.addf %a, %b : f32
        linalg.yield %c : f32
    } -> tensor<4x4xf32>
    %acc2 = tensor.insert_slice %sum into %acc [0, 0] [4, 4] [1, 1]
            : tensor<4x4xf32> into tensor<8x8xf32>
    scf.yield %acc2 : tensor<8x8xf32>
  }
  return %res : tensor<8x8xf32>
}
```

优化后（将 `init` 改为对迭代参数的抽取切片，实现原位写入该片段）：

```mlir
func.func @opt_example(%init: tensor<8x8xf32>, %X: tensor<4x4xf32>, %Y: tensor<4x4xf32>) -> tensor<8x8xf32> {
  %c0 = arith.constant 0 : index
  %c1 = arith.constant 1 : index
  %res = scf.for %i = %c0 to %c1 step %c1 iter_args(%acc = %init) -> tensor<8x8xf32> {
    %slice = tensor.extract_slice %acc [0, 0] [4, 4] [1, 1]
             : tensor<8x8xf32> to tensor<4x4xf32>
    %sum = linalg.generic
      { indexing_maps = [affine_map<(i,j)->(i,j)>,
                         affine_map<(i,j)->(i,j)>,
                         affine_map<(i,j)->(i,j)>],
        iterator_types = ["parallel","parallel"] }
      ins(%X, %Y : tensor<4x4xf32>, tensor<4x4xf32>)
      outs(%slice : tensor<4x4xf32>) {
      ^bb0(%a: f32, %b: f32, %accv: f32):
        %c = arith.addf %a, %b : f32
        linalg.yield %c : f32
    } -> tensor<4x4xf32>
    %acc2 = tensor.insert_slice %sum into %acc [0, 0] [4, 4] [1, 1]
            : tensor<4x4xf32> into tensor<8x8xf32>
    scf.yield %acc2 : tensor<8x8xf32>
  }
  return %res : tensor<8x8xf32>
}
```

**使用方式**

- 命令行：

```bash
mlir-opt --optimize-dps-op-with-yielded-insert-slice input.mlir
mlir-opt -pass-pipeline='func(optimize-dps-op-with-yielded-insert-slice)' input.mlir
```

- C++ 集成：

```cpp
mlir::PassManager pm(&context);
pm.addPass(mlir::tensor::createOptimizeDpsOpWithYieldedInsertSlicePass());
```

**注意**

- 若 `dps` 的初始化不是 `tensor.empty`（例如已绑定到某个真实缓冲），该优化不会生效
- 需要满足支配性约束，常量或早前定义的 `offsets/sizes/strides` 更易通过检查
- 该优化与后续 bufferization 强关联，可显著减少循环迭代体中的复制开销

# Pass 定义文件：`Dialect_Torch_Transforms_Passes.td`

## literal-data-type-cast

**定位**

- 定义位置：`bishengir/include/bishengir/Dialect/Torch/Transforms/Passes.td:23-30`
- 实现文件：`bishengir/lib/Dialect/Torch/Transforms/LiteralDataTypeCast.cpp`
  - 模式类：`LiteralDataTypeCast.cpp:46-95`
  - Pass 主体：`LiteralDataTypeCast.cpp:98-121`
  - 依赖注册：`LiteralDataTypeCast.cpp:101-104`

**作用**

- 将 `torch.value_tensor.literal` 中的元素类型从 `f64` 转换为 `f32`
- 适用于 `DenseIntOrFPElementsAttr` 的浮点常量，保持张量形状不变，仅更改元素位宽
- 用途：统一数值类型到 `f32`，减少不必要的高精度常量带来的计算/带宽成本，利于与下游 `linalg` 类型兼容

**触发条件**

- 目标操作为 `Torch.ValueTensorLiteralOp`，且其 `value` 属性为 `DenseIntOrFPElementsAttr`
- 元素类型为 `Float64Type`；非浮点或非 `f64` 情况不处理
- 验证与上游 `linalg` 类型兼容性通过 `verifyLinalgCompatibleTypes` `LiteralDataTypeCast.cpp:51-54`

**示例（前后对比）**
原始（`f64` 常量）：

```mlir
func.func @literal_fp64_to_fp32() -> !torch.vtensor<[2, 2], f64> {
  %lit = torch.value_tensor.literal [1.0, 2.0, 3.0, 4.0]
         : !torch.vtensor<[2, 2], f64>
  return %lit : !torch.vtensor<[2, 2], f64>
}
```

转换后（元素改为 `f32`，形状不变）：

```mlir
func.func @literal_fp64_to_fp32() -> !torch.vtensor<[2, 2], f32> {
  %lit = torch.value_tensor.literal [1.0, 2.0, 3.0, 4.0]
         : !torch.vtensor<[2, 2], f32>
  return %lit : !torch.vtensor<[2, 2], f32>
}
```

说明：Pass 会构造 `DenseFPElementsAttr` 的 `f32` 版本，更新 `op` 的 `value` 属性和结果类型。

**使用方式**

- 命令行：

```bash
mlir-opt --literal-data-type-cast input.mlir
mlir-opt -pass-pipeline='func(literal-data-type-cast)' input.mlir
```

- C++ 集成：

```cpp
mlir::PassManager pm(&context);
pm.addPass(mlir::torch::createLiteralDataTypeCastPass());
```

**依赖**

- 依赖方言：`tensor::TensorDialect`（Pass 的 `getDependentDialects` 明确注册）
- 需要 Torch Dialect 与 `torch-mlir` 上游支持的 `ValueTensorLiteralOp` 类型及工具函数

# Pass 定义文件：`ExecutionEngine_Passes.td`

## execution-engine-create-host-main

**定位**

- 定义位置：`bishengir/include/bishengir/ExecutionEngine/Passes.td:23-79`
- 实现文件：`bishengir/lib/ExecutionEngine/CreateHostMain.cpp`
  - 获取 host entry：`CreateHostMain.cpp:281-312`
  - 构造 wrapper `main`：`CreateHostMain.cpp:349-384`
  - 打印/读入数据的库调用封装：`CreateHostMain.cpp:80-114, 147-168, 239-279`
  - 形状和类型适配（tensor/memref、ranked/unranked）：`CreateHostMain.cpp:170-237`

**作用**

- 为模块中唯一的 host 入口函数自动生成一个包装函数（默认名 `main`），用于测试与运行：
  - 根据 kernel 的参数类型生成输入内存（`memref.alloc`），并通过占位库函数填充数据
  - 调用 kernel 并收集结果，包括带输出语义的入参以及显式返回值
  - 将输入和输出通过占位库函数打印到文件句柄，便于调试与验证
  - 自动声明所需外部函数签名（如 `getFileHandle`, `closeFileHandle`, `printData<Type>`, `getData<Type>`），并标注 `emit_c_interface` 以便后续生成 C 包装
- 约束：
  - 模块中必须且仅有一个 `hacc.function_kind = HOST` 且 `hacc.host_func_type = host_entry` 的函数
  - 该 host entry 的参数与返回必须是 `tensor` 或 `memref`（可有动态维）
  - 包装函数名可通过选项 `wrapper-name` 修改，默认 `main`

**流程概要**

- 确认并查找 host entry；若不存在或多于一个，报错 `CreateHostMain.cpp:339-343, 295-304`
- 在 module 中创建 `func @main`，插入基本块并设置插入点
- 解析 kernel 形状元数据，扩展 `main` 的参数以承载动态维度大小 `CreateHostMain.cpp:116-145`
- 为每个输入分配 `memref`，并调用 `getData<Type>` 初始化数据 `CreateHostMain.cpp:253-279`
- 将输入打印到文件：`getFileHandle` → 多次 `printData<Type>` → `closeFileHandle` `CreateHostMain.cpp:147-168, 150-167`
- 类型适配：ranked ↔ unranked、tensor ↔ memref 的双向转换 `CreateHostMain.cpp:200-237`
- 调用 kernel，收集结果（含输出语义的入参和值返回）并打印 `CreateHostMain.cpp:368-381, 314-335`
- 返回空 `func.return` 结束 `main` `CreateHostMain.cpp:383`

**简单示例（效果示意）**
输入 IR（单一 host 入口）：

```mlir
func.func @kernel(%A: tensor<4x?xf32>, %B: memref<4x?xf32>)
  attributes {hacc.function_kind = #hacc.function_kind<HOST>,
              hacc.host_func_type = #hacc.host_func_type<host_entry>} {
  %0 = arith.constant 0.0 : f32
  %ret = tensor.empty() : tensor<4x?xf32>
  // ... compute ...
  return %ret : tensor<4x?xf32>
}
```

生成的包装函数（简化示意）：

```mlir
func.func @main(%dyn0: index, %dyn1: index) {
  // 声明并调用库函数初始化与打印
  // 分配输入 memrefs，根据动态维传入大小
  %A_mem = memref.alloc(%dyn0) : memref<4x?xf32>
  %A_unrank = memref.cast %A_mem : memref<4x?xf32> to memref<*xf32>
  call @getDataF32(%A_unrank)  attributes {llvm.emit_c_interface}
  // 打印输入
  %fh_in = call @getFileHandle(%path_in)
  call @printDataF32(%fh_in, %A_unrank) attributes {llvm.emit_c_interface}
  call @closeFileHandle(%fh_in)

  // 形态适配后调用 kernel
  %A_t = bufferization.to_tensor %A_mem : tensor<4x?xf32>
  %B_unrank = ... ; // 同理
  %ret = call @kernel(%A_t, %B_unrank)

  // 打印输出
  %fh_out = call @getFileHandle(%path_out)
  call @printDataF32(%fh_out, %ret_unrank)
  call @closeFileHandle(%fh_out)

  return
}
```

**使用方式**

- 命令行：

```bash
mlir-opt --execution-engine-create-host-main --wrapper-name=main input.mlir
```

- C++ 集成：

```cpp
mlir::PassManager pm(&context);
mlir::execution_engine::ExecutionEngineHostMainCreatorOptions opts;
opts.wrapperName = "main"; // 可选
pm.addPass(mlir::execution_engine::createCreateHostMainPass(opts));
```

**依赖**

- 依赖方言：`arith`, `bufferization`, `func`, `LLVM`, `memref`, `tensor`（见 `Passes.td:67-74`）
- 需要 `hacc` 相关属性标识 host 入口与参数语义，用于判定与结果收集

## execution-engine-convert-hivm-to-upstream

**定位**

- 定义与说明：`bishengir/include/bishengir/ExecutionEngine/Passes.td:81-106`
- 入口实现：`bishengir/lib/ExecutionEngine/ConvertHIVMToUpstream.cpp`
  - Pass 宏定义：`ConvertHIVMToUpstream.cpp:42-45`
  - 关键改写（广播/转置内联）：`ConvertHIVMToUpstream.cpp:162-207, 216-246`
  - 结构化元素级算子改写到 `linalg`/`hfusion`：`ConvertHIVMToUpstream.cpp:366-388, 284-297, 300-321`
  - 归约改写到 `linalg.reduce` 或带索引归约：`ConvertHIVMToUpstream.cpp:392-516`

**作用**

- 将 HIVM 方言中的运算统一转换为上游方言（主要是 `linalg`/`tensor`/`arith` 等）可低到 LLVM 的等价形式
- 同步相关操作视为 NoOp；删除 HIVM 专有类型属性（如内存空间）并使用上游类型
- 对元素级、广播、转置、归约等模式进行语义等价的替换，以便后续使用上游标准管线进行优化与代码生成

**改写要点**

- 类型归一化与适配：移除 HIVM 内存空间等属性，必要时在 `tensor` 与 `memref` 间、`ranked` 与 `unranked` 间插入 `cast`/`bufferization` 转换（见 `ConvertHIVMToUpstream.cpp:184-237`）
- 广播/转置内联：将 HIVM 的广播/转置需求内联为等价的上游操作序列（见 `inlineBroadcast` 与 `inlineTranspose`）
- 元素级运算：用 `linalg.map` 或上游的 `elemwise_unary/binary` 进行替换，并保持函数属性（如 `fun = #linalg.unary_fn<...>`）（`ConvertHIVMToUpstream.cpp:306-316, 366-388`）
- 归约：
  - 普通归约改为 `linalg.reduce`，内部区域用 `arith.add/mul/max/min/...` 表达（`ConvertHIVMToUpstream.cpp:452-516`）
  - “带索引的最值归约”改为 `hfusion.reduce_with_index`（`ConvertHIVMToUpstream.cpp:426-449`）

**简单示例**

- 示例 1：HIVM 元素级二元“加法”改写为 `linalg.map`

```mlir
# 原始（示意）
%out = tensor.empty() : tensor<4x8xf32>
%z = hivm.velemwise_add ins(%x, %y : tensor<4x8xf32>, tensor<4x8xf32>)
                      outs(%out : tensor<4x8xf32>) -> tensor<4x8xf32>

# 改写后（等价）
%out = tensor.empty() : tensor<4x8xf32>
%z = linalg.map ins(%x, %y : tensor<4x8xf32>, tensor<4x8xf32>) outs(%out) {
  ^bb0(%a: f32, %b: f32):
    %c = arith.addf %a, %b : f32
    linalg.yield %c : f32
} -> tensor<4x8xf32>
```

- 示例 2：HIVM 位运算（浮点按位或）改写为 Bitcast + `arith.ori` + Bitcast 回浮点（见 `RewriteVBitwiseOp`）

```mlir
# 原始（示意）
%out = tensor.empty() : tensor<4x8xf32>
%z = hivm.vori ins(%x, %y : tensor<4x8xf32>, tensor<4x8xf32>) outs(%out) -> tensor<4x8xf32>

# 改写后
%out = tensor.empty() : tensor<4x8xf32>
%z = linalg.map ins(%x, %y) outs(%out) {
  ^bb0(%a: f32, %b: f32):
    %ai = arith.bitcast %a : f32 to i32
    %bi = arith.bitcast %b : f32 to i32
    %ri = arith.ori %ai, %bi : i32
    %rf = arith.bitcast %ri : i32 to f32
    linalg.yield %rf : f32
} -> tensor<4x8xf32>
```

- 示例 3：HIVM 归约改写为 `linalg.reduce`（sum）

```mlir
# 原始（示意）
%init = tensor.empty() : tensor<4xf32>
%r = hivm.vreduce ins(%x : tensor<4x8xf32>) outs(%init : tensor<4xf32>)
                  reduce_dims = [1] arith = #hivm.reduce_op<sum> -> tensor<4xf32>

# 改写后
%init = tensor.empty() : tensor<4xf32>
%r = linalg.reduce ins(%x : tensor<4x8xf32>) outs(%init : tensor<4xf32>) dimensions = [1] {
  ^bb0(%a: f32, %acc: f32):
    %sum = arith.addf %a, %acc : f32
    linalg.yield %sum : f32
} -> tensor<4xf32>
```

- 示例 4：移除 HIVM 内存空间属性

```mlir
# 原始
%buf = memref.alloc() : memref<4x8xf32, 3>

# 改写后（默认空间）
%buf = memref.alloc() : memref<4x8xf32>
```

**使用方式**

- 命令行：

```bash
mlir-opt --execution-engine-convert-hivm-to-upstream input.mlir
```

- C++ 集成：

```cpp
mlir::PassManager pm(&context);
pm.addPass(mlir::execution_engine::createConvertHIVMToUpstreamPass());
```

**约束与注意**

- 仅处理 `tensor` 或 `memref` 语义；同步操作视为 NoOp；不支持 PointerCast（见 `Passes.td:92-96`）
- 可能在转换中插入必要的 `tensor.cast` / `memref.cast` 与 `bufferization` 适配（`ConvertHIVMToUpstream.cpp:200-237`）
- 带索引的最值归约会转成 `hfusion.reduce_with_index`，其余归约转到 `linalg.reduce`（`ConvertHIVMToUpstream.cpp:426-449, 452-516`）

# Pass 定义文件：`Transforms_Passes.td`

## canonicalize-module

**定位**

- 定义与说明：`bishengir/include/bishengir/Transforms/Passes.td:24-28`
- 实现文件：`bishengir/lib/Transforms/CanonicalizeModule.cpp`
  - 空模块消除模式：`CanonicalizeModule.cpp:32-45`
  - 嵌套模块展开与属性转移：`CanonicalizeModule.cpp:47-55, 70-103`
  - 构造器：`CanonicalizeModule.cpp:108-111`

**作用**

- 对 `ModuleOp` 进行规范化：
  - 删除空的顶层模块
  - 若顶层模块仅包含一个嵌套的 `module`，则将内层模块的内容提升到外层，并转移其属性；重复此过程直到不再是“完美嵌套”的模块结构
- 目标：清理冗余模块层级、统一模块属性，简化整体 IR 结构，便于后续管线处理

**示例 1：消除空模块**
原始：

```mlir
module {
}
```

规范化后：

```
// 空模块被删除（顶层 IR 为空）
```

**示例 2：展开单层嵌套模块并转移属性**
原始：

```mlir
module attributes { a = 1 } {
  module attributes { b = 2 } {
    func.func @foo() { return }
  }
}
```

规范化后：

```mlir
module attributes { a = 1, b = 2 } {
  func.func @foo() { return }
}
```

说明：内层模块的属性被转移到外层模块，内层 `module` 本身被移除；若仍是“单模块包含单个内层模块”，Pass 会递归继续展开。

## hivm-lower-to-cpu-backend

**定位**

- 定义与说明：`bishengir/include/bishengir/Transforms/Passes.td:30-51`
- 构造器：`bishengir/include/bishengir/Transforms/Passes.td:43`
- 依赖方言：`memref`, `hivm`, `func`, `LLVM`（`Passes.td:44-46`）

**作用**

- 在“已将 HIVM 转换为标准/上游 IR”的上下文中，进一步剥离剩余的 HIVM 特殊性，使模块可被 CPU 后端接受
- 具体目标：
  - 移除类型上的 HIVM 特性，如 `memref` 的 HIVM 地址空间、内存 scope 等
  - 清除或替换仍然残留的 HIVM 特有操作或属性，保证 IR 仅包含标准/上游可降级到 LLVM 的内容
- 用途：让包含少量 HIVM 特性的模块在 CPU 端运行或验证，便于对内核与管线进行纯 CPU 验证

**典型处理项**

- 类型归一化：`memref<... , #hivm.address_space<ub>>` → `memref<...>`
- 删除 HIVM 专用属性与标注以符合 CPU runner 要求
- 保留标准方言操作，移除/替换 HIVM Intrinsics（若仍存在）

**简单示例**

- 示例 1：移除 HIVM 地址空间属性
  原始：

```mlir
%buf = memref.alloc() : memref<4x8xf32, #hivm.address_space<ub>>
```

处理后：

```mlir
%buf = memref.alloc() : memref<4x8xf32>
```

- 示例 2：清理残留的 HIVM Intrinsic（示意）
  原始：

```mlir
%tmp = hivm.hir.pointer_cast(%c123_i64) : memref<32xi8, #hivm.address_space<ub>>
```

处理后：

```mlir
// 指针语义用上游等价或移除；最终不含 hivm.hir.* 残留
```

**使用方式**

- 命令行：

```bash
mlir-opt --hivm-lower-to-cpu-backend input.mlir
```

- C++ 集成：

```cpp
mlir::PassManager pm(&context);
pm.addPass(bishengir::createLowerToCPUBackendPass());
```

**注意事项**

- 本 Pass 假设前置管线已完成 “hivm-to-std” 的主体转换；这里只做残留清理与属性归一化
- 如仍存在不支持的 HIVM 操作，需在前置管线完成转换或在本 Pass 中扩展规则以确保消除
- 可选项 `--enable-triton-kernel-compile` 辅助确定 launch 维度源信息（影响某些 CPU runner 下游流程）

## dead-func-elimination

**定位**

- 定义与说明：`bishengir/include/bishengir/Transforms/Passes.td:53-64`
- 实现文件：`bishengir/lib/Transforms/DeadFunctionElimination.cpp`
  - 核心逻辑：`DeadFunctionElimination.cpp:32-47`
  - Pass 构造与运行：`DeadFunctionElimination.cpp:51-61, 68-71`

**作用**

- 删除模块中所有“无引用”的函数类操作（`FunctionOpInterface`），包括可能是 `public` 的符号
- 可配置过滤器决定哪些函数符合删除条件，默认策略由 `DeadFunctionEliminationOptions::filterFn` 提供
- 与上游 `SymbolDCE` 的区别：
  - 仅针对函数类操作
  - 更激进（可删除 `public` 函数）
  - 提供可定制的过滤机制

**判定规则**

- 使用 `SymbolTable::symbolKnownUseEmpty(funcLikeOp, module)` 判断符号在模块中是否没有已知使用
- 通过 `options.filterFn(funcLikeOp)` 进行二次筛选，返回 `true` 才删除

**简单示例**

- 示例 1：删除未使用的 `func.func`
  原始：

```mlir
module {
  func.func @dead() { return }
  func.func @live() { return }
  func.func @caller() {
    call @live() : () -> ()
    return
  }
}
```

运行 Pass 后：

```mlir
module {
  func.func @live() { return }
  func.func @caller() {
    call @live() : () -> ()
    return
  }
}
```

说明：`@dead` 无任何使用，被删除；`@live` 被 `@caller` 调用，保留。

- 示例 2：带过滤器（只删除以 `_tmp` 结尾的未用函数）
  伪代码调用：

```cpp
bishengir::DeadFunctionEliminationOptions opts;
opts.filterFn = [](mlir::FunctionOpInterface func) {
  return func.getName().ends_with("_tmp");
};
pm.addPass(bishengir::createDeadFunctionEliminationPass(opts));
```

效果：仅删除未使用且名称以 `_tmp` 结尾的函数，其余未使用函数保留。

**使用方式**

- 命令行：

```bash
mlir-opt --dead-func-elimination input.mlir
```

- C++ 集成：

```cpp
mlir::PassManager pm(&context);
bishengir::DeadFunctionEliminationOptions opts; // 可选配置
pm.addPass(bishengir::createDeadFunctionEliminationPass(opts));
```

## canonicalize-ext

**定位**

- 定义与构造器：`bishengir/include/bishengir/Transforms/Passes.td:66-68`
- 实现文件：`bishengir/lib/Transforms/ExtendedCanonicalizer.cpp`
  - 继承自上游 `Canonicalizer` 基类：`ExtendedCanonicalizer.cpp:36-55`
  - 扩展模式收集：`ExtendedCanonicalizer.cpp:58-71`
  - 执行与配置：`ExtendedCanonicalizer.cpp:73-85`
  - 构造器：`ExtendedCanonicalizer.cpp:90-93`

**作用**

- 基于上游 `Canonicalizer`，进行“扩展版”规范化：
  - 收集所有已加载方言与已注册操作的 canonicalization 模式
  - 注入额外扩展模式：`memref` 与 `linalg` 的增强规范化集合
  - 支持禁用/启用特定模式、遍历方向、区域简化、迭代次数等高级配置
- 目标：更强力度地折叠等价 IR、消除冗余、统一表达形式，为后续优化与降级铺路

**示例**

- 示例 1：折叠冗余 `tensor.cast`（取决于方言提供的 canonicalization）
  原始：

```mlir
%a = tensor.cast %x : tensor<4x8xf32> to tensor<4x8xf32>
```

规范化后：

```mlir
%a = %x : tensor<4x8xf32>
```

- 示例 2：规范化 `memref.subview` 的组合（依赖扩展 `memref` 模式）
  原始：

```mlir
%sv1 = memref.subview %buf[0, 0][4, 8][1, 1] : memref<4x8xf32> to memref<4x8xf32>
%sv2 = memref.subview %sv1[0, 0][4, 8][1, 1] : memref<4x8xf32> to memref<4x8xf32>
```

规范化后（合并为一次等效子视图或直接去除）：

```mlir
%sv = memref.subview %buf[0, 0][4, 8][1, 1] : memref<4x8xf32> to memref<4x8xf32>
```

- 示例 3：`linalg` 逐元素模式归一（依赖扩展 `linalg` 模式）
  原始：

```mlir
%out = tensor.empty() : tensor<4x8xf32>
%r = linalg.generic
  { indexing_maps = [#id, #id, #id], iterator_types = ["parallel","parallel"] }
  ins(%x, %y)
  outs(%out) {
  ^bb0(%a: f32, %b: f32, %acc: f32):
    %c = arith.addf %a, %b : f32
    linalg.yield %c : f32
} -> tensor<4x8xf32>
```

规范化后（可能提升为 `linalg.elemwise_binary` 或简化区域结构）：

```mlir
%out = tensor.empty() : tensor<4x8xf32>
%r = linalg.elemwise_binary {fun = #linalg.binary_fn<add>}
     ins(%x, %y) outs(%out) -> tensor<4x8xf32>
```

**使用方式**

- 命令行：

```bash
mlir-opt --canonicalize-ext input.mlir
# 或配置选项
mlir-opt --canonicalize-ext="max-iterations=10,enable-region-simplification=true" input.mlir
```

- C++ 集成：

```cpp
mlir::PassManager pm(&context);
mlir::CanonicalizerOptions opts;
opts.topDownProcessingEnabled = true;
opts.enableRegionSimplification = true;
pm.addPass(bishengir::createExtendedCanonicalizerPass(opts));
```

**配置能力**

- 支持上游 `CanonicalizerOptions` 的参数：禁用/启用模式、遍历方向、最大迭代次数、最大重写数、区域简化开关等
- 适合在大型管线中作为通用清理与归一化步骤，提升 IR 的优化空间与可读性
