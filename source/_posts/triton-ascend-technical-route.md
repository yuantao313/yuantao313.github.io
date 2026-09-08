---
title: Triton-Ascend 技术路线与差异化竞争力
date: 2026-09-08 09:00:00
categories:
  - agentdocs
tags:
  - compiler-stack
  - npu-backend
  - heterogeneous-computing
---

| Source | Version | Commit | Date |
| :--- | :--- | :--- | :--- |
| [triton-lang/triton-ascend](https://github.com/triton-lang/triton-ascend/tree/b64287188046fe7bb0cfd424ee3e389f96a7affa) | [3.6.0](https://github.com/triton-lang/triton-ascend/blob/b64287188046fe7bb0cfd424ee3e389f96a7affa/version.txt#L1) | [b64287188046fe7bb0cfd424ee3e389f96a7affa](https://github.com/triton-lang/triton-ascend/commit/b64287188046fe7bb0cfd424ee3e389f96a7affa) | 2026-09-07 |

# Triton-Ascend 技术路线

## 项目方向概览

Triton-Ascend 保留 Triton 的 Python DSL、JIT、specialization、cache 和 TTIR 前端，在 TTIR 之后建立 Ascend NPU 专属编译与执行体系：

```text
Python kernel → TTIR → structure/unstructure → Linalg/HIVM/HFusion
              → MLIR bytecode → BiShengIR → npubin → torch_npu + CANN
```

[ASTSource.make_ir()](https://github.com/triton-lang/triton-ascend/blob/b64287188046fe7bb0cfd424ee3e389f96a7affa/python/triton/compiler/compiler.py#L78) 生成 TTIR，[compile()](https://github.com/triton-lang/triton-ascend/blob/b64287188046fe7bb0cfd424ee3e389f96a7affa/python/triton/compiler/compiler.py#L226) 发现后端，[AscendBackend.add_stages()](https://github.com/triton-lang/triton-ascend/blob/b64287188046fe7bb0cfd424ee3e389f96a7affa/third_party/ascend/backend/compiler.py#L1388) 装配普通 `ttir → ttadapter → mlirbc → bcmlir → npubin` 路径；A5 pure-SIMT 则直接 `ttir → npubin`。项目实际覆盖语言扩展、NPU lowering/优化以及设备加载与发射，而非只替换 launcher。

## 工程结构

Triton-Ascend 保持 Triton 的前端入口和通用编译设施，在后端建立面向 Ascend 的分层实现：Python kernel 先进入 TTIR，再经过 Ascend 专属 lowering、结构化优化和硬件代码生成，最终由 `npubin`、`torch_npu` 与 CANN 完成加载和发射。

| Layer | Implementation | Role |
|:---|:---|:---|
| Frontend | [`python/triton/compiler/compiler.py`](https://github.com/triton-lang/triton-ascend/blob/b64287188046fe7bb0cfd424ee3e389f96a7affa/python/triton/compiler/compiler.py) | AST 到 TTIR 及编译阶段装配 |
| Backend | [`third_party/ascend/backend/compiler.py`](https://github.com/triton-lang/triton-ascend/blob/b64287188046fe7bb0cfd424ee3e389f96a7affa/third_party/ascend/backend/compiler.py) | Ascend 编译阶段和后端选项 |
| Dialect/IR | [`third_party/ascend/include/Dialect/TritonAscend/IR/`](https://github.com/triton-lang/triton-ascend/tree/b64287188046fe7bb0cfd424ee3e389f96a7affa/third_party/ascend/include/Dialect/TritonAscend/IR) | Ascend op、layout 和硬件抽象 |
| Lowering | [`third_party/ascend/lib/Conversion/`](https://github.com/triton-lang/triton-ascend/tree/b64287188046fe7bb0cfd424ee3e389f96a7affa/third_party/ascend/lib/Conversion) | TTIR/Linalg/结构化 IR 到 Ascend 表示 |
| Pipeline | [`third_party/ascend/lib/DynamicCVPipeline/`](https://github.com/triton-lang/triton-ascend/tree/b64287188046fe7bb0cfd424ee3e389f96a7affa/third_party/ascend/lib/DynamicCVPipeline) | Cube/Vector、UB、流水和多缓冲优化 |
| Launch | [`third_party/ascend/backend/driver.py`](https://github.com/triton-lang/triton-ascend/blob/b64287188046fe7bb0cfd424ee3e389f96a7affa/third_party/ascend/backend/driver.py) | 设备、ABI、launcher 和执行边界 |

```mermaid
flowchart LR
    A[Python Triton kernel] --> B[TTIR]
    B --> C[Ascend backend stages]
    C --> D[Ascend dialect and structured IR]
    D --> E[Cube / Vector / UB pipeline]
    E --> F[MLIR bytecode and BiShengIR]
    F --> G[npubin]
    G --> H[torch_npu and CANN launch]
```


## 项目核心竞争点

1. **TTIR 后精准分叉。** [AscendBackend.supports_target()](https://github.com/triton-lang/triton-ascend/blob/b64287188046fe7bb0cfd424ee3e389f96a7affa/third_party/ascend/backend/compiler.py#L1306) 只接管 NPU target，最大化复用 Triton 开发体验，同时避免硬套 NVIDIA warp/CTA 布局。
2. **结构化 SIMD 与 pure-SIMT 双路线。** [compile_mode](https://github.com/triton-lang/triton-ascend/blob/b64287188046fe7bb0cfd424ee3e389f96a7affa/third_party/ascend/backend/compiler.py#L1109) 提供 `simd`、默认 `simd_simt_template` 和 A5 专属 `simt_only`；[ttir_to_linalg()](https://github.com/triton-lang/triton-ascend/blob/b64287188046fe7bb0cfd424ee3e389f96a7affa/third_party/ascend/backend/compiler.py#L197) 负责结构化主链，[ttir_to_npubin()](https://github.com/triton-lang/triton-ascend/blob/b64287188046fe7bb0cfd424ee3e389f96a7affa/third_party/ascend/backend/compiler.py#L1213) 负责直编。
3. **把 Cube/Vector 协同做成编译问题。** [enable_dynamic_cv_pipeline](https://github.com/triton-lang/triton-ascend/blob/b64287188046fe7bb0cfd424ee3e389f96a7affa/third_party/ascend/backend/compiler.py#L1088) 驱动数据流拆分、依赖分析、多缓冲和同步规划；[InterCoreTransferAndSyncPass](https://github.com/triton-lang/triton-ascend/blob/b64287188046fe7bb0cfd424ee3e389f96a7affa/third_party/ascend/lib/DynamicCVPipeline/SplitDataflow/InterCoreTransferAndSync.cpp#L187) 处理 Vector↔Cube 传输与同步。
4. **显式硬件语义。** [CORE](https://github.com/triton-lang/triton-ascend/blob/b64287188046fe7bb0cfd424ee3e389f96a7affa/third_party/ascend/language/cann/extension/core.py#L91)、[PIPE](https://github.com/triton-lang/triton-ascend/blob/b64287188046fe7bb0cfd424ee3e389f96a7affa/third_party/ascend/language/cann/extension/core.py#L98)、[ascend_address_space_group](https://github.com/triton-lang/triton-ascend/blob/b64287188046fe7bb0cfd424ee3e389f96a7affa/third_party/ascend/language/cann/extension/core.py#L140)、[sync_block_set()](https://github.com/triton-lang/triton-ascend/blob/b64287188046fe7bb0cfd424ee3e389f96a7affa/third_party/ascend/language/cann/extension/core.py#L228) 和 [fixpipe()](https://github.com/triton-lang/triton-ascend/blob/b64287188046fe7bb0cfd424ee3e389f96a7affa/third_party/ascend/language/cann/extension/core.py#L331) 把核型、流水线、UB/L1/L0 与同步带到 DSL。
5. **编译—执行闭环。** [linalg_to_bc_by_triton_mlir_opt()](https://github.com/triton-lang/triton-ascend/blob/b64287188046fe7bb0cfd424ee3e389f96a7affa/third_party/ascend/backend/compiler.py#L316) 与 [bc_to_linalg_by_bishengir_opt()](https://github.com/triton-lang/triton-ascend/blob/b64287188046fe7bb0cfd424ee3e389f96a7affa/third_party/ascend/backend/compiler.py#L354) 交接 BiShengIR；[NPUUtils.load_binary()](https://github.com/triton-lang/triton-ascend/blob/b64287188046fe7bb0cfd424ee3e389f96a7affa/third_party/ascend/backend/driver.py#L100) 和 [make_launcher()](https://github.com/triton-lang/triton-ascend/blob/b64287188046fe7bb0cfd424ee3e389f96a7affa/third_party/ascend/backend/driver.py#L720) 消费 task type、workspace、同步锁等 metadata。

## 相比同类产品的实现差异

直接参照同一源码树中的 NVIDIA Triton 后端。

| 维度 | NVIDIA Triton | Triton-Ascend |
| :--- | :--- | :--- |
| 前端 | Python DSL → TTIR | 复用相同前端 |
| IR 主链 | [CUDABackend.add_stages()](https://github.com/triton-lang/triton-ascend/blob/b64287188046fe7bb0cfd424ee3e389f96a7affa/third_party/nvidia/backend/compiler.py#L537) 使用 TTGIR→LLVM IR→PTX→cubin | 绕过 TTGIR，采用结构化 Linalg/memref/HIVM/HFusion，或 pure-SIMT 直编 |
| 硬件模型 | warp、CTA、GPU layout、shared memory | Cube/Vector、pipe、UB/L1/L0、跨核 transfer/sync |
| codegen | LLVM/NVPTX 与 CUDA assembler | `triton-mlir-opt`、`bishengir-opt`、`bishengir-compile` |
| runtime | CUDA driver | torch_npu device/stream + CANN ACL/rt |

根本差异是硬件语义的承载位置：NVIDIA 将线程和布局集中在 TTGIR；Ascend 结构化路线以 Linalg/memref、核型、存储层级和同步 pass 表达异构 AICore。pure-SIMT 也不是复用 CUDA TTGIR，而是让 BiShengIR 直接接收 TTIR。

## 复用的公共组件

| 组件 | 复用内容 |
| :--- | :--- |
| Triton 前端 | [JITFunction.run()](https://github.com/triton-lang/triton-ascend/blob/b64287188046fe7bb0cfd424ee3e389f96a7affa/python/triton/runtime/jit.py#L361)、AST/semantic、TTIR、specialization 和 grid |
| 插件与编译框架 | [_discover_backends()](https://github.com/triton-lang/triton-ascend/blob/b64287188046fe7bb0cfd424ee3e389f96a7affa/python/triton/backends/__init__.py#L38)、[get_entry_points()](https://github.com/triton-lang/triton-ascend/blob/b64287188046fe7bb0cfd424ee3e389f96a7affa/setup.py#L757)、[CompiledKernel](https://github.com/triton-lang/triton-ascend/blob/b64287188046fe7bb0cfd424ee3e389f96a7affa/python/triton/compiler/compiler.py#L404) |
| 通用优化 | [make_ttir()](https://github.com/triton-lang/triton-ascend/blob/b64287188046fe7bb0cfd424ee3e389f96a7affa/third_party/ascend/backend/compiler.py#L166) 复用 inliner、canonicalizer、CSE、LICM、DCE、unroll |
| LLVM/MLIR | [根 CMake 配置](https://github.com/triton-lang/triton-ascend/blob/b64287188046fe7bb0cfd424ee3e389f96a7affa/CMakeLists.txt#L99) 复用 dialect、pass manager、TableGen、bytecode 与 LLVM dialect |
| 标准 IR | [buffer_ir.cc](https://github.com/triton-lang/triton-ascend/blob/b64287188046fe7bb0cfd424ee3e389f96a7affa/python/triton/extension/buffer/src/buffer_ir.cc#L45) 复用 memref 与 bufferization |
| 厂商组件 | [编译器工具发现](https://github.com/triton-lang/triton-ascend/blob/b64287188046fe7bb0cfd424ee3e389f96a7affa/third_party/ascend/backend/utils.py#L487) 连接 BiShengIR、CANN 与 torch_npu |

## 可复用但自行开发的部分及原因

- **结构化 lowering 取代 TTGIR 主链。** 既有 TTGIR 围绕 warp/CTA；项目另建 [TritonStructured dialect](https://github.com/triton-lang/triton-ascend/blob/b64287188046fe7bb0cfd424ee3e389f96a7affa/third_party/ascend/include/Dialect/TritonStructured/IR/TritonStructuredDialect.td#L41) 和 [TritonAscend dialect](https://github.com/triton-lang/triton-ascend/blob/b64287188046fe7bb0cfd424ee3e389f96a7affa/third_party/ascend/include/Dialect/TritonAscend/IR/TritonAscendDialect.td#L13)，因为 Ascend 必须表达 Cube/Vector、片上层级与 pipe。
- **在 memref 上另建 buffer DSL。** [buffer_type](https://github.com/triton-lang/triton-ascend/blob/b64287188046fe7bb0cfd424ee3e389f96a7affa/python/triton/extension/buffer/language/core.py#L78)、[alloc()](https://github.com/triton-lang/triton-ascend/blob/b64287188046fe7bb0cfd424ee3e389f96a7affa/python/triton/extension/buffer/language/core.py#L191) 和 [subview()](https://github.com/triton-lang/triton-ascend/blob/b64287188046fe7bb0cfd424ee3e389f96a7affa/python/triton/extension/buffer/language/core.py#L283) 将 shape、stride、address space 提升为 Python DSL 类型；标准 memref 本身不是 Triton 用户 API。
- **另建 NPU launcher。** [NPUDriver](https://github.com/triton-lang/triton-ascend/blob/b64287188046fe7bb0cfd424ee3e389f96a7affa/third_party/ascend/backend/driver.py#L241)、[NPULauncher](https://github.com/triton-lang/triton-ascend/blob/b64287188046fe7bb0cfd424ee3e389f96a7affa/third_party/ascend/backend/driver.py#L199) 和 [cann_register_kernel()](https://github.com/triton-lang/triton-ascend/blob/b64287188046fe7bb0cfd424ee3e389f96a7affa/third_party/ascend/backend/npu_utils.cpp#L86) 处理 Ascend binary magic、workspace、lock 与 task queue，CUDA ABI 无法复用。
- **另建 dialect 工具与补丁层。** [triton-mlir-opt.cpp](https://github.com/triton-lang/triton-ascend/blob/b64287188046fe7bb0cfd424ee3e389f96a7affa/third_party/ascend/bin/triton-mlir-opt.cpp#L1) 注册 Triton/Ascend/BiShengIR dialect；[setup_ascend._patch_module()](https://github.com/triton-lang/triton-ascend/blob/b64287188046fe7bb0cfd424ee3e389f96a7affa/setup_ascend.py#L377) 与 [_apply_ascend_patch()](https://github.com/triton-lang/triton-ascend/blob/b64287188046fe7bb0cfd424ee3e389f96a7affa/third_party/ascend/backend/__init__.py#L28) 补足现有插件接口未覆盖的 build、CodeGenerator、IR override 和 dot 语义。
- **另建 NPU 搜索层。** [AutoTilingTuner](https://github.com/triton-lang/triton-ascend/blob/b64287188046fe7bb0cfd424ee3e389f96a7affa/third_party/ascend/backend/runtime/autotuner.py#L254)、[UBTuner](https://github.com/triton-lang/triton-ascend/blob/b64287188046fe7bb0cfd424ee3e389f96a7affa/third_party/ascend/backend/runtime/ubtuner.py#L426) 和 [run_costmodel()](https://github.com/triton-lang/triton-ascend/blob/b64287188046fe7bb0cfd424ee3e389f96a7affa/third_party/ascend/backend/runtime/costmodel_runtime.py#L32) 纳入核型、UB、多缓冲和 tiling 约束，这是通用 autotuner 不具备的资源模型。

## 已做的发展方向

- 主线已合入上游 Triton [3.6.0](https://github.com/triton-lang/triton-ascend/blob/b64287188046fe7bb0cfd424ee3e389f96a7affa/version.txt#L1)，对应 [50c50cc7d](https://github.com/triton-lang/triton-ascend/commit/50c50cc7d)。
- A2/A3 已形成结构化 SIMD 主链，A5/950 已加入 pure-SIMT；[NPUOptions.__post_init__()](https://github.com/triton-lang/triton-ascend/blob/b64287188046fe7bb0cfd424ee3e389f96a7affa/third_party/ascend/backend/compiler.py#L1122) 固化架构相关模式和 metadata。
- Ascend DSL 已覆盖 core、pipe、sync、fixpipe、fractal dot；[Conv2dOp](https://github.com/triton-lang/triton-ascend/blob/b64287188046fe7bb0cfd424ee3e389f96a7affa/third_party/ascend/include/Dialect/TritonAscend/IR/TritonAscendOps.td#L594) 已进入 IR。
- DynamicCV 已覆盖数据流拆分、跨核传输、同步、多缓冲和 UB 检查；[GraphOptimizePass](https://github.com/triton-lang/triton-ascend/blob/b64287188046fe7bb0cfd424ee3e389f96a7affa/third_party/ascend/lib/TritonToGraph/GraphOptimizationPass.cpp#L95) 已在 TTIR 阶段匹配图模式。
- 运行链已传递 task type、workspace 和 lock layout，并兼容 CANN 新旧 API，演进见 [b1187ec82](https://github.com/triton-lang/triton-ascend/commit/b1187ec82)。自动 tiling、UB tuner、cost model cache、IR dump 和源码级调试亦已落地，源码 stepping 见 [c2d48ad24](https://github.com/triton-lang/triton-ascend/commit/c2d48ad24)。

## 正在做的发展方向

- **SSBuffer/DynamicCV 收敛。** 最近仍连续修复读写依赖、multi-region、buffer slot 与 VF operand substitution，并优化 SDPA，见 [8ba4ac4ce](https://github.com/triton-lang/triton-ascend/commit/8ba4ac4ce)、[54bb7c040](https://github.com/triton-lang/triton-ascend/commit/54bb7c040)、[6ff78229c](https://github.com/triton-lang/triton-ascend/commit/6ff78229c) 和基线 [b64287188046fe7bb0cfd424ee3e389f96a7affa](https://github.com/triton-lang/triton-ascend/commit/b64287188046fe7bb0cfd424ee3e389f96a7affa)。重点已从功能搭建转到复杂控制流、同步精度和真实算子性能。
- **SIMD/SIMT 覆盖扩展。** [d1f8e2fa4](https://github.com/triton-lang/triton-ascend/commit/d1f8e2fa4) 补强 shape-only pointer 的 indirect-load 分析，[a674a6f85](https://github.com/triton-lang/triton-ascend/commit/a674a6f85) 调整 SinOp 的 SIMT template 路由，持续扩大默认混合模式的自动适用面。
- **选项契约统一。** [NPUOptions](https://github.com/triton-lang/triton-ascend/blob/b64287188046fe7bb0cfd424ee3e389f96a7affa/third_party/ascend/backend/compiler.py#L1027) 已集中公开选项与内部 metadata，近期仍在处理废弃项兼容和 HFusion 开关传递，见 [452e8717e](https://github.com/triton-lang/triton-ascend/commit/452e8717e)、[e1314a246](https://github.com/triton-lang/triton-ascend/commit/e1314a246)、[47e291c05](https://github.com/triton-lang/triton-ascend/commit/47e291c05)。
- **可观测与可重复调优。** 图优化日志、autotune 磁盘缓存、L2 cache 清理和 debug-line 重写密集进入主线，见 [e93350318](https://github.com/triton-lang/triton-ascend/commit/e93350318)、[d78ae7718](https://github.com/triton-lang/triton-ascend/commit/d78ae7718)、[c47ebc7cb](https://github.com/triton-lang/triton-ascend/commit/c47ebc7cb)、[14ba391ad](https://github.com/triton-lang/triton-ascend/commit/14ba391ad)。
- **算子与数学语义补齐。** [conv2d()](https://github.com/triton-lang/triton-ascend/blob/b64287188046fe7bb0cfd424ee3e389f96a7affa/third_party/ascend/language/cann/extension/core.py#L681) 由 [6ff6d725a](https://github.com/triton-lang/triton-ascend/commit/6ff6d725a) 加入；libdevice 接口与精度仍持续修订，见 [7d0fd4533](https://github.com/triton-lang/triton-ascend/commit/7d0fd4533) 和 [ade29bb49](https://github.com/triton-lang/triton-ascend/commit/ade29bb49)。

## 技术路线结论

项目的技术价值集中在 TTIR 之后：前端最大化复用 Triton，后端以结构化 IR、Linalg/memref、HIVM/HFusion、Cube/Vector 数据流和 SSBuffer 映射 Ascend，再由 BiShengIR、torch_npu 与 CANN 完成 codegen、资源管理和发射。近期重心已从建立可工作的 NPU 后端，转向混合 Cube/Vector 与 SIMD/SIMT 路线在复杂控制流、性能、调试和调优上的持续收敛。
