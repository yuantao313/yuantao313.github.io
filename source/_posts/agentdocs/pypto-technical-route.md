---
title: PyPTO 技术路线与差异化竞争力
date: 2026-09-08 09:00:00
description: 解析 PyPTO 从 Tensor 级 Python 程序到硬件感知 Tile 调度、代码生成和昇腾设备执行的双路线架构。
categories:
  - agentdocs
tags:
  - compiler-architecture
  - tensor-programming
  - ascend-ai
---

| Field | Value |
|:---|:---|
| Source | [cann/pypto](https://gitcode.com/cann/pypto/tree/5d6afbe0dc470aaf692d23e4c3b73d5fa461acf5) |
| Version | [PyPTO 0.2.1](https://gitcode.com/cann/pypto/blob/5d6afbe0dc470aaf692d23e4c3b73d5fa461acf5/pyproject.toml) / [CANN package 9.2.0](https://gitcode.com/cann/pypto/blob/5d6afbe0dc470aaf692d23e4c3b73d5fa461acf5/version.cmake) / [`v9.2.0-beta.2-322-g5d6afbe0d`](https://gitcode.com/cann/pypto/commit/5d6afbe0dc470aaf692d23e4c3b73d5fa461acf5) |
| Commit | [5d6afbe0dc470aaf692d23e4c3b73d5fa461acf5](https://gitcode.com/cann/pypto/commit/5d6afbe0dc470aaf692d23e4c3b73d5fa461acf5) |
| Date | 2026-09-07 20:00:38 +08:00 |

# PyPTO 技术路线与差异化竞争力

## 项目方向概览

PyPTO 是面向昇腾 AI 处理器的高性能编程框架。它要解决的不是单个算子的 Python 包装，而是把 Tensor 级程序逐层编译成硬件感知的 Tile 程序：在 Tensor 语义仍然完整时决定切分、数据布局、存储层级、搬运、融合、乱序调度和同步，生成 CCE/PTO 源码，再交给 Bisheng 和 CANN 完成目标编译与执行。

项目形成了两条互补路线：

| 路线 | 面向用户 | 输入与决策方式 | 编译和执行路径 |
|:---|:---|:---|:---|
| 主 PyPTO | 算法开发者、整图/融合算子开发者 | Tensor 与 Python 结构化控制流；Tile、内存和调度主要由编译器决定 | Python → PIL → `pypto::ir` → `tile_fwk` 多级图 → CCE/PTO → AICPU 控制流 + AI Core 叶子程序，形成 MPMD 调度 |
| PyPTO Pro | 性能专家、硬件能力开发者 | 显式 Tile、Reg、SIMT 以及 Cube/Vector section；用户直接控制更多硬件细节 | Python DSL → `pypto::ir` → SSA → CCE/PTO C++ → JIT 共享库，以 SPMD kernel 方式启动 |

主路线的默认入口由 [`jit(new_ir=True)`](https://gitcode.com/cann/pypto/blob/5d6afbe0dc470aaf692d23e4c3b73d5fa461acf5/python/pypto/frontend/parser/entry.py) 进入 [`compile_new_ir`](https://gitcode.com/cann/pypto/blob/5d6afbe0dc470aaf692d23e4c3b73d5fa461acf5/python/pypto/pil/compile_pipeline.py)。Python 源码先经 [`ast2pil`](https://gitcode.com/cann/pypto/blob/5d6afbe0dc470aaf692d23e4c3b73d5fa461acf5/python/pypto/pil/parser.py) 建立 PIL 控制流，再由 [`dispatch_block`](https://gitcode.com/cann/pypto/blob/5d6afbe0dc470aaf692d23e4c3b73d5fa461acf5/python/pypto/pil/dispatcher.py) 执行语义发射。随后 [`_build_default_pipeline`](https://gitcode.com/cann/pypto/blob/5d6afbe0dc470aaf692d23e4c3b73d5fa461acf5/python/pypto/pil/compile_pipeline.py) 依次完成 token 推导、规范化与消除无效语句、分支合并、符号标量简化、冗余 token 消除、root function 构建和动态图终结。

新 IR 并未直接替代成熟后端。[`CreateRootFunctions`](https://gitcode.com/cann/pypto/blob/5d6afbe0dc470aaf692d23e4c3b73d5fa461acf5/framework/src/interface/tensor/ir.cpp) 调用 [`RootFunctionBuilder::Build`](https://gitcode.com/cann/pypto/blob/5d6afbe0dc470aaf692d23e4c3b73d5fa461acf5/framework/src/interface/tensor/ir_func_builder.cpp)，把结构化 IR 分段为 dynamic root、path function 和 hidden leaf function，并建立 call、incast/outcast 与运行时 slot；[`FinalizeDynamicFunction`](https://gitcode.com/cann/pypto/blob/5d6afbe0dc470aaf692d23e4c3b73d5fa461acf5/framework/src/interface/tensor/ir_finalize.cpp) 再把叶子函数提交给既有编译链。因此当前主线的实际架构是“新前端与新通用 IR + 成熟的 `tile_fwk` lowering 后端”。

PyPTO Pro 则由 [`KernelDef::parse_target_program`](https://gitcode.com/cann/pypto/blob/5d6afbe0dc470aaf692d23e4c3b73d5fa461acf5/python/pypto_pro/runtime/kernel.py) 和 [`ASTParser::parse_function`](https://gitcode.com/cann/pypto/blob/5d6afbe0dc470aaf692d23e4c3b73d5fa461acf5/python/pypto_pro/language/parser/_ast_parser.py) 建图，经 [`CCECodegen::GenerateSingle`](https://gitcode.com/cann/pypto/blob/5d6afbe0dc470aaf692d23e4c3b73d5fa461acf5/framework/src/interface/pypto_pro/codegen/cce/cce_codegen.cpp) 执行 [`ConvertToSSA`](https://gitcode.com/cann/pypto/blob/5d6afbe0dc470aaf692d23e4c3b73d5fa461acf5/framework/src/interface/ir/transforms/convert_to_ssa_pass.cpp) 并直接生成 CCE/PTO C++。[`JitCompileConfig`](https://gitcode.com/cann/pypto/blob/5d6afbe0dc470aaf692d23e4c3b73d5fa461acf5/python/pypto_pro/runtime/compile_config.py) 集中描述 A2/A3、A5 的核型、内存模型、Bisheng 参数和运行库依赖，JIT 最终由 [`_run_bisheng`](https://gitcode.com/cann/pypto/blob/5d6afbe0dc470aaf692d23e4c3b73d5fa461acf5/python/pypto_pro/runtime/jit.py) 产出可加载的共享库。

<!-- more -->

## 工程结构

PyPTO 的工程边界按“Python 编程入口 → 多级 IR 与图抽象 → Pass/调度 → CCE/PTO CodeGen → Bisheng/CANN 工具链 → 设备执行”组织。关键层不是简单的 Python 绑定，而是由 Tensor/Tile/Block 图逐步形成硬件感知的执行图。

| Layer | Implementation | Role |
|:---|:---|:---|
| Frontend | [`python/pypto/pil/compile_pipeline.py`](https://gitcode.com/cann/pypto/blob/5d6afbe0dc470aaf692d23e4c3b73d5fa461acf5/python/pypto/pil/compile_pipeline.py) | Python 程序进入编译管线 |
| IR | [`framework/src/interface/tensor/`](https://gitcode.com/cann/pypto/tree/5d6afbe0dc470aaf692d23e4c3b73d5fa461acf5/framework/src/interface/tensor) | LogicalTensor、SymbolicScalar、token 和 slot 抽象 |
| Pass | [`framework/src/passes/pass_mgr/pass_manager.cpp`](https://gitcode.com/cann/pypto/blob/5d6afbe0dc470aaf692d23e4c3b73d5fa461acf5/framework/src/passes/pass_mgr/pass_manager.cpp) | 图级 Pass、依赖处理和调度管线 |
| CodeGen | [`framework/src/codegen/npu/codegen_npu.cpp`](https://gitcode.com/cann/pypto/blob/5d6afbe0dc470aaf692d23e4c3b73d5fa461acf5/framework/src/codegen/npu/codegen_npu.cpp) | 生成 NPU 侧 CCE/PTO 代码 |
| Toolchain | [`framework/src/machine/compile/aicore_compiler.cpp`](https://gitcode.com/cann/pypto/blob/5d6afbe0dc470aaf692d23e4c3b73d5fa461acf5/framework/src/machine/compile/aicore_compiler.cpp) | AI Core 编译和外部工具链衔接 |
| Execution | [`framework/src/machine/runtime/launcher/device_launcher.cpp`](https://gitcode.com/cann/pypto/blob/5d6afbe0dc470aaf692d23e4c3b73d5fa461acf5/framework/src/machine/runtime/launcher/device_launcher.cpp) | 设备侧加载、参数和启动边界 |

```mermaid
flowchart LR
    A["Python/PIL frontend"] --> B["Tensor and symbolic IR"]
    B --> C["Tensor / Tile / Block graph"]
    C --> D["Pass manager and scheduling"]
    D --> E["CCE/PTO NPU CodeGen"]
    E --> F["Bisheng and CANN toolchain"]
    F --> G["Device launcher"]
```


## 项目核心竞争点

### 1. 动态 Tensor 语义贯通到设备侧 MPMD

PIL 不把 Python 简化成线性算子列表。[`visit_if`、`visit_for` 和 `visit_while`](https://gitcode.com/cann/pypto/blob/5d6afbe0dc470aaf692d23e4c3b73d5fa461acf5/python/pypto/pil/parser.py) 把控制流编码为独立 block，[`if_else_impl` 与 `loop_impl`](https://gitcode.com/cann/pypto/blob/5d6afbe0dc470aaf692d23e4c3b73d5fa461acf5/python/pypto/pil/ops.py) 将变量更新转换为 return variables、iteration arguments 和 yield。符号条件、动态 valid shape、跨分支值和 Tensor 副作用由 [`SymbolicScalar`](https://gitcode.com/cann/pypto/blob/5d6afbe0dc470aaf692d23e4c3b73d5fa461acf5/framework/src/interface/tensor/symbolic_scalar.h)、[`LogicalTensor`](https://gitcode.com/cann/pypto/blob/5d6afbe0dc470aaf692d23e4c3b73d5fa461acf5/framework/src/interface/tensor/logical_tensor.h)、读写 token 与 [`TensorSlotManager`](https://gitcode.com/cann/pypto/blob/5d6afbe0dc470aaf692d23e4c3b73d5fa461acf5/framework/src/interface/tensor/tensor_slot.cpp) 联合承载。

这套表达最终不是被静态抹平：[`CompileDyndevFunction`](https://gitcode.com/cann/pypto/blob/5d6afbe0dc470aaf692d23e4c3b73d5fa461acf5/framework/src/machine/host/backend.cpp) 编译设备控制程序与 AI Core 叶子程序，[`aicore_entry.h`](https://gitcode.com/cann/pypto/blob/5d6afbe0dc470aaf692d23e4c3b73d5fa461acf5/framework/src/interface/machine/device/tilefwk/aicore_entry.h) 定义设备 ABI。竞争力因此来自一条完整链路：动态控制决策在设备侧执行，而叶子计算仍然保持硬件专用 Tile 程序。

### 2. 领域决策前移的多级图编译

[`BuildPvc2OooPassEntries`](https://gitcode.com/cann/pypto/blob/5d6afbe0dc470aaf692d23e4c3b73d5fa461acf5/framework/src/passes/pass_mgr/pass_manager.cpp) 固定编排 46 个 pass，覆盖四组关键决策：

- Tensor 语义规范化：format 推导、automatic cast、reshape/view 清理、内存冲突推导；
- Tensor 到 Tile：raw tensor 切分、reshape 切分、图分区与子图函数化；
- 数据与存储：memory type、N-buffer、L1 copy-in 复用、move 生成、本地 buffer padding、全局内存复用；
- 执行映射：Cube/Vector 相关融合、OOO schedule、同步插入、TileOp 顺序调优、last-use 标记和 codegen 预处理。

[`ExpandFunction`](https://gitcode.com/cann/pypto/blob/5d6afbe0dc470aaf692d23e4c3b73d5fa461acf5/framework/src/passes/tensor_graph_pass/expand_function.h) 是 Tensor Graph 向 Tile Graph 展开的关键转换点，[`SubgraphToFunction`](https://gitcode.com/cann/pypto/blob/5d6afbe0dc470aaf692d23e4c3b73d5fa461acf5/framework/src/passes/tile_graph_pass/subgraph_to_function.h) 把子图变为可生成函数。Tensor、Tile、Block/Execute、Leaf 等状态由 [`FunctionType` 与 `GraphType`](https://gitcode.com/cann/pypto/blob/5d6afbe0dc470aaf692d23e4c3b73d5fa461acf5/framework/src/interface/function/function.h) 明确区分。这使布局、片上存储、异构计算单元和同步等信息在语义尚未丢失时就进入优化，而不是全部留给通用目标后端猜测。

### 3. 自动编译与专家编程共存

主 PyPTO 把硬件决策交给编译器；PyPTO Pro 暴露 Tile、TileGroup、Reg、Ptr、Cube/Vector section 和 SIMT。两条路线共同覆盖“高层生产力”和“底层性能可控性”：前者适合把复杂 Tensor 子图自动降到设备程序，后者适合快速暴露新指令、新数据布局和精细流水控制。

二者共享 [`pypto::ir`](https://gitcode.com/cann/pypto/tree/5d6afbe0dc470aaf692d23e4c3b73d5fa461acf5/framework/include/ir)、Python binding、PTO/Bisheng 与 CANN 底座，但没有强行共用同一种 lowering 和启动模型。这种“双轨共底座”比单一路线更能覆盖不同性能工程层级，也是项目当前最鲜明的产品形态。

### 4. 面向 Ascend 的垂直闭环

[`CodeGenNPU::GenCode`](https://gitcode.com/cann/pypto/blob/5d6afbe0dc470aaf692d23e4c3b73d5fa461acf5/framework/src/codegen/npu/codegen_npu.cpp) 已把图决策具体化为 CCE/PTO TileOp 源码，[`CodeGenNPU::PrepareCmd`](https://gitcode.com/cann/pypto/blob/5d6afbe0dc470aaf692d23e4c3b73d5fa461acf5/framework/src/codegen/npu/codegen_npu.cpp) 构造 Bisheng 编译命令；顶层 [`CMakeLists.txt`](https://gitcode.com/cann/pypto/blob/5d6afbe0dc470aaf692d23e4c3b73d5fa461acf5/CMakeLists.txt) 接入 ACL Runtime、HAL、HCOMM、dump、profiling 与 securec。PyPTO 因而控制从 Python 语义到设备任务的主要环节，新硬件能力可以沿 DSL、IR、pass、codegen 和 runtime 同步落地。

## 相比同类产品的实现差异

下表比较的是编译实现形态，而非 API 外观：

| 对比对象 | 常见实现重心 | PyPTO 的实现差异 |
|:---|:---|:---|
| Triton 类 kernel DSL | 以单 kernel、blocked program 和 SPMD 实例映射为中心，程序员定义 kernel 范围 | 主 PyPTO 能从 Tensor 图继续分解出多个 AI Core leaf，并由 AICPU 承载动态控制形成 MPMD；PyPTO Pro 才更接近显式 SPMD kernel DSL，但额外显式区分 Cube/Vector、Tile/Reg/SIMT |
| TVM 类张量编译栈 | 以可扩展的通用 Tensor IR、schedule 和多后端 lowering 为中心 | PyPTO 不以跨后端通用性为中心，而把 Ascend 的多级存储、Cube/Vector、PTO 指令、设备控制流和 CANN ABI 固化进多级图与 pass；优势是硬件协同深，代价是后端耦合强 |
| MLIR 型编译基础设施 | 复用 dialect、operation、rewrite、pass manager 和 LLVM lowering 生态 | PyPTO 自行实现不可变 [`IRNode`](https://gitcode.com/cann/pypto/blob/5d6afbe0dc470aaf692d23e4c3b73d5fa461acf5/framework/include/ir/core.h)、类型与结构化语句、[`IRVisitor/IRMutator`](https://gitcode.com/cann/pypto/tree/5d6afbe0dc470aaf692d23e4c3b73d5fa461acf5/framework/include/ir/transforms)、反射、verifier 和 [`Pass`](https://gitcode.com/cann/pypto/blob/5d6afbe0dc470aaf692d23e4c3b73d5fa461acf5/framework/include/ir/transforms/passes.h)，再桥接自有 `tile_fwk` 图，而非建立 MLIR dialect |
| 图框架的算子捕获/融合 | 捕获现有算子图，选择或生成融合 kernel，复杂控制通常停留在 host | PyPTO 的 Python 控制流、符号 shape、Tensor alias/token、slot 与设备动态函数是同一编译对象，不只是在已定 kernel 范围内做融合 |
| Ascend C/手写 CCE | 开发者直接管理核内存储、搬运、同步和指令组合 | 主 PyPTO 用 Tensor→Tile→Execute 多级 pass 自动承担这些决策；Pro 保留显式能力，但仍提供结构化 IR、SSA 转换、统一 codegen 和 JIT 装载 |

PyPTO 的差异化并不是“另一个 Python kernel 语法”，而是把 Tensor 图自动 lowering、设备侧动态 MPMD 和显式专家 kernel 放在一套 Ascend 垂直栈中。它选择用硬件专用性换取对性能决策和执行模型的控制力。

## 复用的公共组件

| 公共组件 | 使用位置 | 在 PyPTO 中承担的职责 |
|:---|:---|:---|
| Python `ast`、`inspect` | [`parser.py`](https://gitcode.com/cann/pypto/blob/5d6afbe0dc470aaf692d23e4c3b73d5fa461acf5/python/pypto/pil/parser.py)、[`entry.py`](https://gitcode.com/cann/pypto/blob/5d6afbe0dc470aaf692d23e4c3b73d5fa461acf5/python/pypto/frontend/parser/entry.py) | 取得 Python 函数源码并构造前端控制流，不自行开发 Python 语法解析器 |
| PyTorch、torch_npu | [`entry.py`](https://gitcode.com/cann/pypto/blob/5d6afbe0dc470aaf692d23e4c3b73d5fa461acf5/python/pypto/frontend/parser/entry.py)、[`jit.py`](https://gitcode.com/cann/pypto/blob/5d6afbe0dc470aaf692d23e4c3b73d5fa461acf5/python/pypto_pro/runtime/jit.py) | Tensor、dtype、stream、device 与分布式生态入口 |
| pybind11 | [`python/src/bindings`](https://gitcode.com/cann/pypto/tree/5d6afbe0dc470aaf692d23e4c3b73d5fa461acf5/python/src/bindings) | 把 C++ IR、pass、Tensor 和 runtime 能力暴露给 Python |
| CMake、setuptools | [`CMakeLists.txt`](https://gitcode.com/cann/pypto/blob/5d6afbe0dc470aaf692d23e4c3b73d5fa461acf5/CMakeLists.txt)、[`setup.py`](https://gitcode.com/cann/pypto/blob/5d6afbe0dc470aaf692d23e4c3b73d5fa461acf5/setup.py) | C++/Python 混合工程的配置、构建与分发 |
| PTO headers、Bisheng | [`codegen_npu.cpp`](https://gitcode.com/cann/pypto/blob/5d6afbe0dc470aaf692d23e4c3b73d5fa461acf5/framework/src/codegen/npu/codegen_npu.cpp)、[`compile_config.py`](https://gitcode.com/cann/pypto/blob/5d6afbe0dc470aaf692d23e4c3b73d5fa461acf5/python/pypto_pro/runtime/compile_config.py) | 承接 PyPTO 已完成领域 lowering 的 CCE/PTO 源码，生成 AI Core 目标代码 |
| CANN ACL/Runtime/HAL/HCOMM/ADump/MsProf | [`framework/src/adapter`](https://gitcode.com/cann/pypto/tree/5d6afbe0dc470aaf692d23e4c3b73d5fa461acf5/framework/src/adapter) | 设备、内存、流、通信、诊断和性能数据接口 |
| LLVM 独立工具和 Bisheng `-mllvm` 接口 | [`PvModelImpl.h`](https://gitcode.com/cann/pypto/blob/5d6afbe0dc470aaf692d23e4c3b73d5fa461acf5/framework/src/cost_model/simulation_pv/PvModelImpl.h)、[`_DEFAULT_CCE_JIT_COMPILE_CONFIG`](https://gitcode.com/cann/pypto/blob/5d6afbe0dc470aaf692d23e4c3b73d5fa461acf5/python/pypto_pro/runtime/compile_config.py) | 使用 `llvm-objcopy` 及向外部编译器透传 LLVM 风格参数；PyPTO 本身没有 LLVM/MLIR IR lowering 依赖 |

公共组件的复用集中在语言宿主、Tensor 生态、binding、工程系统和目标工具链；直接决定 PyPTO 编程语义与性能策略的部分仍由项目掌握。

## 可复用但自行开发的部分及原因

| 自行开发部分 | 原本可复用的通用能力 | 选择自研形成的实际价值 |
|:---|:---|:---|
| PIL 与 dispatcher | Python AST 框架、通用 tracing/graph capture | [`PIL Function/Block/Call/Value/Jump`](https://gitcode.com/cann/pypto/blob/5d6afbe0dc470aaf692d23e4c3b73d5fa461acf5/python/pypto/pil/pir.py) 专门保留分支、循环、变量写入和 Tensor 发射时序，可直接构造动态 Tensor IR |
| `pypto::ir` | MLIR、LLVM IR 或其他编译 IR | 自有 [`Expr/Stmt/Type`](https://gitcode.com/cann/pypto/tree/5d6afbe0dc470aaf692d23e4c3b73d5fa461acf5/framework/include/ir) 能把 `LogicalTensorType`、`TileType`、`TokenType`、结构化控制流和 Python binding 放入同一对象模型，并同时服务主路线与 Pro |
| Tensor 领域语义 | 通用 SSA、alias analysis、shape expression | [`InferTokenPass`](https://gitcode.com/cann/pypto/blob/5d6afbe0dc470aaf692d23e4c3b73d5fa461acf5/framework/src/interface/ir/transforms/infer_token_pass.cpp) 把 Tensor 读写顺序显式化；dynamic valid shape、RawTensor 版本、slot 与 incast/outcast 则对应 PyPTO 的动态设备程序，不是通用 SSA 可以直接替代的概念 |
| `tile_fwk` 多级图及 46-pass 流水 | 通用 graph compiler、自动调度框架 | [`PVC2_OOO`](https://gitcode.com/cann/pypto/blob/5d6afbe0dc470aaf692d23e4c3b73d5fa461acf5/framework/src/passes/pass_mgr/pass_manager.cpp) 直接编码 Ascend 的 Tensor→Tile 展开、存储层级、Cube/Vector、搬运、融合、同步和 OOO 规则，是自动性能路线的核心资产 |
| CCE/PTO CodeGen | 通用 C++ emitter 或现成 kernel backend | [`CodeGenFactory`](https://gitcode.com/cann/pypto/blob/5d6afbe0dc470aaf692d23e4c3b73d5fa461acf5/framework/src/codegen/codegen_factory.h) 按 NPU 架构选择生成器，自有 emitter 可以保留 pass 已决定的地址空间、核型、TileOp 与同步语义 |
| 动态 device machine | 通用 host launcher、传统 kernel runtime | [`framework/src/machine`](https://gitcode.com/cann/pypto/tree/5d6afbe0dc470aaf692d23e4c3b73d5fa461acf5/framework/src/machine) 把控制流、表达式表、leaf binary 和 Tensor slot 编成设备可消费的程序，支撑主路线特有的 AICPU + AI Core MPMD |
| Pro DSL、CCECodegen 与 JIT | 现成 kernel DSL/JIT | 自有路线可直接表达 Ascend Cube/Vector、PTO Tile、寄存器与 SIMT，并让 [`resolve_kernel_target`](https://gitcode.com/cann/pypto/blob/5d6afbe0dc470aaf692d23e4c3b73d5fa461acf5/python/pypto_pro/runtime/compile_config.py) 同时约束编译目标和 launch geometry，避免 ABI 信息分散 |

这里的共同原因不是拒绝公共生态，而是现成组件无法直接覆盖“动态 Tensor 图—Ascend 多级图—设备 MPMD”这组联合语义。PyPTO 复用宿主生态与目标工具链，把自研投入集中在决定抽象层级、自动优化质量和设备执行方式的中间层。

## 特色架构：双 IR、双路线、一个硬件底座

当前架构包含两个“并存”而非简单重复的维度：

1. **双 IR。** 新 [`pypto::ir`](https://gitcode.com/cann/pypto/tree/5d6afbe0dc470aaf692d23e4c3b73d5fa461acf5/framework/include/ir) 提供不可变节点、结构化控制流、SSA、类型、反射与 verifier；旧 [`tile_fwk::Function/Operation`](https://gitcode.com/cann/pypto/blob/5d6afbe0dc470aaf692d23e4c3b73d5fa461acf5/framework/src/interface/function/function.h) 保留成熟的 Tensor/Tile/Execute 图和调度、内存元数据。连接点是 [`RootFunctionBuilder`](https://gitcode.com/cann/pypto/blob/5d6afbe0dc470aaf692d23e4c3b73d5fa461acf5/framework/src/interface/tensor/ir_func_builder.cpp)。
2. **双编程路线。** 主路线通过桥接复用 `tile_fwk` 自动 lowering；Pro 复用新 IR，却跳过主路线多级图，直接经 SSA 和 CCECodegen 生成 kernel。
3. **一个硬件底座。** 两条路线最终都使用 PTO/CCE、Bisheng、CANN runtime 和 Ascend AI Core，只是在“由编译器还是用户决定硬件细节”以及“MPMD 还是 SPMD”上分工。

这一架构解释了 PyPTO 的演进方式：新 IR 可以先改善前端结构、控制流和公共类型系统，而不必重写全部成熟 pass；Pro 又可以借新 IR 快速建立短路径。与此同时，两套 lowering 暂时保留各自适合的执行模型。

## 已做的发展方向

### 从框架骨架到可用 Tensor 编程栈

项目已经历 [v0.1.0 初始发布、v0.1.1 新前端、v0.1.2 集群与编译性能、v0.2.0 前端表达重构](https://gitcode.com/cann/pypto/blob/5d6afbe0dc470aaf692d23e4c3b73d5fa461acf5/README.md)，当前 Python distribution 为 [`0.2.1`](https://gitcode.com/cann/pypto/blob/5d6afbe0dc470aaf692d23e4c3b73d5fa461acf5/pyproject.toml)。Tensor API、动态 shape、多级图、MPMD、工具链和集群能力已经组成完整产品轮廓。

### 新前端和新 IR 接入成熟后端

- [`062c219f`](https://gitcode.com/cann/pypto/commit/062c219f2881116a83d916fe65805f5c1a2921f7) 将新 IR 编译流水接入 JIT 并加入路径开关；
- [`7964a2a4`](https://gitcode.com/cann/pypto/commit/7964a2a4f71a4eca147c2a460e75d2e9f345f873) 把新流水接到既有 HostMachine；
- [`50a7132d`](https://gitcode.com/cann/pypto/commit/50a7132d9577e1b70177b3a6e961dd8045a10742) 推导 Tensor 图读写 token，[`a55a7025`](https://gitcode.com/cann/pypto/commit/a55a702506bd190f1dc2f570540f557dcb69f2a2) 加入冗余 token 消除；
- [`46c71146`](https://gitcode.com/cann/pypto/commit/46c711463a9f57add9164bc97c2a6f7431f50915) 继续让成熟 pass 适配新 IR 特性。

这条路线已经完成默认入口、结构化 IR、token 依赖和旧后端桥接的主骨架，采用渐进替换而非整栈重写。

### PyPTO Pro 从新路径扩展到显式硬件编程

- [`c944f459`](https://gitcode.com/cann/pypto/commit/c944f45988e94a86b28ee8b1620f6e524cd8a9fb) 落地 Pro 前端和 codegen；
- [`d71f7ed1`](https://gitcode.com/cann/pypto/commit/d71f7ed16ed2c6b25c805510deb7e2a6df21d128) 加入 SIMT；
- [`5b25e40b`](https://gitcode.com/cann/pypto/commit/5b25e40b4d3045de1124e706d658572c3d3d35c8) 加入 Tile/TileGroup 零拷贝 reinterpret；
- [`852afc47`](https://gitcode.com/cann/pypto/commit/852afc47758a9ecac33193399859201e366743c6) 扩展 GM NZ，[`9c264475`](https://gitcode.com/cann/pypto/commit/9c2644752c7aeff0a22b783275b2bd6f3a7a61b8) 扩展 A5 SIMT gather/index-put；
- [`e23f9ee7`](https://gitcode.com/cann/pypto/commit/e23f9ee7dedd1ca2980418547ed9b9dc128abceb) 移除早期 `pl.function`、`pl.program` 抽象，表明 API 已从原型继续收敛。

### 面向 A5、Vector Fusion 与设备调度深化

截至基线前，项目已连续扩展 A5 dtype、layout 与 SIMT 能力；[`fe20d726`](https://gitcode.com/cann/pypto/commit/fe20d7268e99a748ea649909e3a71524ac188981) 加入 Vector Fusion cluster 识别，[`10a0fc5a`](https://gitcode.com/cann/pypto/commit/10a0fc5a8d6fe9b3d39b29720fa201efcf962fd5) 让 OOO schedule 适配 VF，[`f87a7060`](https://gitcode.com/cann/pypto/commit/f87a70601f84005117cadcaa82feb28ec5785f72) 继续增强 VF 的同步与 loop-axis 处理。设备侧则已有控制流 cache、early launch、任务队列与异常诊断等连续建设。

## 正在做的发展方向

“正在做”由基线附近连续提交共同呈现，重点不是新增独立子系统，而是让已落地的双路线在正确性、性能和工程可用性上继续闭环。

### 1. 让 token/alias 成为调度与内存优化的硬约束

[`af3848f3`](https://gitcode.com/cann/pypto/commit/af3848f37c47eb64474bf218f6210ad540451390) 修复原地操作 token 保留，[`d434158c`](https://gitcode.com/cann/pypto/commit/d434158c6e7e387a74fd975da16f1e63dd11dbde) 用有序 WAW 保证跨循环 atomic-add 的确定性。基线之后紧邻的 [`af153920`](https://gitcode.com/cann/pypto/commit/af1539200b459c22979d658e4b590b0cc91f72ef) 继续保留 shared view 的 overlap dependency，[`ead80e92`](https://gitcode.com/cann/pypto/commit/ead80e92310a1b60e001d71799400ed936b20469) 让 OOO schedule 遵循 token dependency。方向已经从“IR 中存在 token”转为“后续调度、融合和内存判断一致消费 token”。

### 2. 新旧 IR 桥接处持续补齐等价语义

近期提交集中在 dynamic valid shape、assemble/outcast alias、slot 生命周期和依赖刷新：[`59b692c2`](https://gitcode.com/cann/pypto/commit/59b692c22b923cc948c9422f25c3444951c77110) 按分支条件选择合并 valid shape，[`ddeb250e`](https://gitcode.com/cann/pypto/commit/ddeb250e1e332c861a637fc6f9f95cb2bc83d666) 在重排中保留 outcast alias，[`358dab96`](https://gitcode.com/cann/pypto/commit/358dab9618ebccb7f5719100593427f2217ba051) 延迟分配跨图 outcast slot，[`bf599774`](https://gitcode.com/cann/pypto/commit/bf599774e2b38de2c8139cebeca7563865f35d1c) 在删除 operation 后刷新变量依赖。它们共同指向 [`RootFunctionBuilder`](https://gitcode.com/cann/pypto/blob/5d6afbe0dc470aaf692d23e4c3b73d5fa461acf5/framework/src/interface/tensor/ir_func_builder.cpp) 周围的新旧语义一致性建设。

### 3. Vector Fusion 与 OOO 流水继续协同

[`0d17f3e2`](https://gitcode.com/cann/pypto/commit/0d17f3e2c31cd9b69ce977ad01128642e190973f) 按 HLF 模式控制 VF cluster 识别，[`724e0c74`](https://gitcode.com/cann/pypto/commit/724e0c74317531ff9180b53979d307854c3370c8) 收紧 VF fusion prefix 兼容性，[`1aeae00c`](https://gitcode.com/cann/pypto/commit/1aeae00cc957ce8435f0928d25921e51d0f940f3) 将 OOO spill 重构为表驱动流程。可见的目标是把 VF 从孤立融合能力接入调度、同步、spill 和图合法性判断，成为稳定的执行图优化阶段。

### 4. PyPTO Pro API、自动流水与 A5 能力快速收敛

[`ac2e5899`](https://gitcode.com/cann/pypto/commit/ac2e58991b9345869e814ec35839955499869929) 重构 Pro auto pipeline；相邻提交持续增加 load/store layout 校验、Tile address capacity、sync path、GM NZ、SIMT 和高精度函数能力。其技术方向是保持显式编程可控性的同时，把目标选择、ABI、参数合法性和常用流水规则前置到 [`JitCompileConfig`](https://gitcode.com/cann/pypto/blob/5d6afbe0dc470aaf692d23e4c3b73d5fa461acf5/python/pypto_pro/runtime/compile_config.py) 与前端，使错误更早暴露、生成路径更一致。

### 5. 设备动态调度向低开销和可恢复演进

[`22f9f88f`](https://gitcode.com/cann/pypto/commit/22f9f88fb54e3439ea93e61b183aecc33d13a62c) 支持 host control-flow cache，[`2af4f3f6`](https://gitcode.com/cann/pypto/commit/2af4f3f6ad66a46c43384901f389ae564620f687) 与 [`9d7444c5`](https://gitcode.com/cann/pypto/commit/9d7444c5ef1521b27125080f9e83301ce05f1bc0) 推进 early launch，[`b1324ce3`](https://gitcode.com/cann/pypto/commit/b1324ce34fdb645986f12213f9b4da7fe074a7e6) 处理 early-launch slot 复用同步，[`33f14b4f`](https://gitcode.com/cann/pypto/commit/33f14b4fecc48a769158b2fe97a770ad6cb7dbf0) 修复 control-flow cache。主线在保留设备动态性的同时，持续压缩控制开销，并补齐 cache 恢复、slot 生命周期和异常路径。

## 技术路线总结

PyPTO 的核心资产可以归纳为三层：上层用 PIL 与 `pypto::ir` 保留 Python 动态 Tensor 语义；中层用 `tile_fwk` 多级图和 PVC2_OOO 把 Tensor 程序变成 Ascend 硬件感知的 Tile/Execute 图；下层用 CCE/PTO、Bisheng、CANN 和设备 machine 形成 MPMD 执行闭环。PyPTO Pro 在同一硬件底座上提供显式 Tile/Reg/SIMT 的短路径。

相比通用 Tensor 编译器、单 kernel DSL 或手写 CCE，这条路线最独特的组合是：**高层动态控制、自动多级 Tile lowering、设备侧 MPMD 与专家级 SPMD kernel 同时存在**。当前演进主线也围绕这套组合展开——强化 token/alias 对后端的约束，补齐新旧 IR 桥接语义，将 VF 与 OOO/同步协同，并让 Pro 和设备调度走向更稳定、更低开销的实现。
