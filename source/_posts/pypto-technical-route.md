---
title: PyPTO 编译器技术路线与差异化竞争力
date: 2026-09-07 09:00:00
categories:
  - agentdocs
tags:
  - compiler-stack
  - ai-systems
---

# PyPTO 编译器技术路线与差异化竞争力调研

> 调研对象：https://gitcode.com/cann/pypto.git，本地 master
>
> 固定基线：commit 5d6afbe0dc470aaf692d23e4c3b73d5fa461acf5
>
> 基线时间：2026-09-07 20:00:38 +08:00
>
> git describe：v9.2.0-beta.2-322-g5d6afbe0d
>
> 调研方法：以当前仓库源码、配置和本地 Git 历史为证据，沿 Python 前端→IR→Pass/Lowering→CCE/PTO CodeGen→Bisheng→CANN/Ascend 的编译主链分析。Runtime 仅保留解释编译产物执行模型所必需的内容。

## 证据口径

- **[已验证]**：能够由本报告引用的当前源码、配置、测试或 commit 直接复核。
- **[推断]**：由多个已验证事实形成的技术判断，不代表项目官方定位或承诺。
- **[未验证]**：当前仓库没有足够证据，或依赖仓库外组件内部实现，不能下确定结论。

---

## 问题一：PyPTO 最核心的竞争力是什么？

**答案：把高层 Tensor 程序自动编译为适配 Ascend 多核、多流水、多级存储的 Tile 级程序，并把动态控制流与 AI Core 叶子计算协同编译；同时提供 PyPTO Pro 作为显式 Tile/Reg/SIMT 的性能专家路径。**

- **[已验证]** 主 pypto 路线不是简单的 Python 算子封装。它从 Python 控制流构建结构化 IR，再依次处理依赖 token、动态图、Tensor 切分、内存类型、子图划分、乱序调度、同步插入、内存复用和 CCE/PTO 代码生成。证据集中在 [compile_pipeline.py](python/pypto/pil/compile_pipeline.py)、[pass_manager.cpp](framework/src/passes/pass_mgr/pass_manager.cpp) 和 [codegen_npu.cpp](framework/src/codegen/npu/codegen_npu.cpp)。
- **[已验证]** 主路线最终形成 AICPU 控制流与多个 AI Core 叶子程序。Host 编译编排见 [CompileDyndevFunction](framework/src/machine/host/backend.cpp)，设备 ABI 和 AI Core 子函数调用见 [aicore_entry.h](framework/src/interface/machine/device/tilefwk/aicore_entry.h)。
- **[已验证]** PyPTO Pro 提供另一种路径：用户显式表达 Tile、寄存器、SIMT、Cube/Vector section，编译器从自研 IR 直接生成 CCE/PTO C++，再由 Bisheng JIT 编译。入口见 [KernelDef](python/pypto_pro/runtime/kernel.py)、[CCECodegen::GenerateSingle](framework/src/interface/pypto_pro/codegen/cce/cce_codegen.cpp) 和 [_run_bisheng](python/pypto_pro/runtime/jit.py)。
- **[推断]** 相比只做算子捕获或只生成单个 kernel 的 DSL，PyPTO 的差异化不在 Python 语法本身，而在“高层动态图语义→自动 Tile 化与硬件资源决策→设备侧 MPMD 调度”的纵向闭环。
- **[推断]** 两条路线覆盖了两个互补目标：主 pypto 追求生产力和自动优化，pypto_pro 追求性能可控性。它们共享基础 IR 和 CANN 底座，但还不是一个完全统一的后端。

核心路线可压缩为：

    高层 Tensor 路线：
    Python → PIL → pypto::ir → Tensor IR Pass
           → tile_fwk 多级图 → PVC2_OOO
           → CCE/PTO CodeGen → Bisheng
           → AICPU 控制流 + AI Core 叶子程序 → CANN Runtime

    显式 Pro 路线：
    Python Tile/Reg/SIMT → pypto::ir → ConvertToSSA
                         → CCECodegen → CCE/PTO C++
                         → Bisheng JIT .so → CANN Runtime

---

## 问题二：PyPTO 有哪两条特色编译路径，它们分别解决什么问题？

### 答案 A：主 pypto 是高层 Tensor 自动 lowering 路线

- **[已验证]** [entry.py::jit](python/pypto/frontend/parser/entry.py) 默认 new_ir=True，通过 JitCallableWrapper 进入 [compile_new_ir](python/pypto/pil/compile_pipeline.py)。
- **[已验证]** [ast2pil](python/pypto/pil/parser.py) 将 Python AST 转为 PIL 的 Function、Block、Call、Value、Jump；[dispatch_block](python/pypto/pil/dispatcher.py) 解释执行 PIL，[ops.py](python/pypto/pil/ops.py) 中的语义实现将控制流和 Tensor 操作发射为 C++ 侧 pypto::ir 节点。
- **[已验证]** 新 IR 完成前置规范化后，[CreateRootFunctions](framework/src/interface/tensor/ir.cpp) 调用 [RootFunctionBuilder](framework/src/interface/tensor/ir_func_builder.cpp)，把 IR 分段为 dynamic root、path function 和 hidden leaf function，并建立 call、incast/outcast 与 slot 关系。
- **[已验证]** 后续通过 PVC2_OOO 进入 Tensor Graph、Tile Graph、Block/Execute Graph 的成熟 Pass 管线，最后生成 CCE/PTO 代码和设备程序。
- **[推断]** 这条路线的产品价值是让用户保留 Tensor 级表达，编译器承担 Tile 切分、异构核映射、存储层级、同步和调度等硬件化工作。

### 答案 B：pypto_pro 是显式 Tile/Reg/SIMT 直达 CCE 路线

- **[已验证]** [KernelDef::parse_target_program](python/pypto_pro/runtime/kernel.py) 按 Cube、Vector target 调用 [ASTParser::parse_function](python/pypto_pro/language/parser/_ast_parser.py)，构造包含 TensorType、TileType、PtrType、SectionStmt 和 Call 的 pypto::ir Program。
- **[已验证]** [CCECodegen::GenerateSingle](framework/src/interface/pypto_pro/codegen/cce/cce_codegen.cpp) 执行 ir::pass::ConvertToSSA()，处理 helper/SIMT function，随后直接生成 CCE/PTO C++。
- **[已验证]** [compile_config.py](python/pypto_pro/runtime/compile_config.py) 的 CCE_BACKEND 是当前唯一 JIT backend；_DEFAULT_CCE_JIT_COMPILE_CONFIG 配置 -xcce、--cce-aicore-arch、PTO include、runtime/profapi 链接和 -mllvm 后端参数。
- **[已验证]** [jit.py::_parse_and_codegen_targets](python/pypto_pro/runtime/jit.py) 分别生成 Cube/Vector 实现并组装 wrapper；_run_bisheng 编译为 call_kernel_<hash>.so。
- **[推断]** Pro 绕开 PVC2_OOO 的大部分自动图 lowering，用用户显式安排换取更短、可预测的生成路径，适合手工极致优化和新硬件能力快速暴露。

### 两条路线的关键差异

| 维度 | 主 pypto | pypto_pro |
|---|---|---|
| 输入抽象 | **[已验证]** Tensor 和结构化 Python 控制流 | **[已验证]** 显式 Tile/Reg/SIMT、Cube/Vector section |
| 核心优化 | **[已验证]** 新 IR Pass + PVC2_OOO 多级图 Pass | **[已验证]** AST/IR 规范化、ConvertToSSA、直接 CCE CodeGen |
| 硬件决策者 | **[推断]** 主要由编译器自动决定 | **[推断]** 主要由用户显式决定 |
| 执行模型 | **[已验证]** AICPU 调度不同 AI Core 叶子程序，MPMD | **[已验证]** 多 AI Core 执行同一 kernel，SPMD |
| 当前汇合点 | **[已验证]** 共享 pypto::ir 基础设施、Python binding、CANN/PTO/Bisheng 底座 | **[已验证]** 后半段 Pass、CodeGen 和运行模型仍分叉 |

---

## 问题三：Python 前端的特色在哪里，它怎样保留动态程序语义？

**答案：前端不是把 Python 直接翻译成算子列表，而是先构造 PIL，再通过 trace/解释执行将 Python 变量、分支和循环转换为结构化 IR 的 carried values、Yield 和 token。**

- **[已验证]** PIL 数据模型定义在 [pir.py](python/pypto/pil/pir.py)：Function/Block/Call/Value/Jump 保存控制流骨架，Scope.varmap 把 PIL Value 映射到 Python 值、SymbolicScalar 或 Tensor。
- **[已验证]** [parser.py](python/pypto/pil/parser.py) 的 visit_if、visit_for、visit_while 把分支和循环体编码为独立 Block；store_names 记录分支/循环中发生写入的变量，为后续 yield/carry 决策提供输入。
- **[已验证]** [dispatcher.py::dispatch_call](python/pypto/pil/dispatcher.py) 解析参数、调用语义实现，并在每次用户调用后通过 BuildContext.emit_tensor_stmts() 刷出 C++ 侧累积的 TensorOpStmt。
- **[已验证]** [ops.py](python/pypto/pil/ops.py) 的 if_else_impl 和 loop_impl 将变量更新转为 IfStmt/ForStmt 的 returnVars、iterArgs 和 Yield；Journal 用于隔离、回滚分支 trace 期间的属性写副作用。
- **[已验证]** 具体 Python bool 或可判定条件可以静态选择分支，符号条件则保留为 IR 控制流；动态 valid shape 发生分支差异时，前端会构造相应的 phi 标量并加入 yield。
- **[推断]** 这种“两步式前端”把 Python 语法处理与 Tensor IR 构造解耦，既保留动态控制流，又能在进入图优化前把 Python 副作用收敛到可验证的结构化表示，是主路线的重要编译设施。

---

## 问题四：PyPTO IR 的关键抽象是什么？哪些是自研？

**答案：PyPTO 使用两层自研中间表示：通用不可变 pypto::ir，以及承载 Tensor/Tile 编译历史能力的 npu::tile_fwk 图体系。**

### pypto::ir：新通用 IR 基础

- **[已验证]** [core.h](framework/include/ir/core.h) 定义 IRNode 和 ObjectKind；[expr.h](framework/include/ir/expr.h)、[stmt.h](framework/include/ir/stmt.h)、[type.h](framework/include/ir/type.h) 定义表达式、结构化控制流、TensorOpStmt、Scalar/Token/LogicalTensor/MemRef/Tile 等类型。
- **[已验证]** IR 节点普遍由 shared_ptr<const T> 持有，变换通过 [IRVisitor/IRMutator](framework/include/ir/transforms/) 重建；GetFieldDescriptors() 是 visitor、mutator、结构等价、哈希、打印和 Python binding 的共享反射元数据。
- **[已验证]** [builder.h](framework/include/ir/builder.h) 提供 BeginIf/For/While/Section、Emit、iterArg/returnVar 等结构化构图接口；[verifier](framework/src/interface/ir/verifier/) 校验 SSA、use-before-def、控制流 yield 数量和类型约束。
- **[已验证]** Pass 抽象支持 required/produced/invalidated IRProperty，具体 factory 位于 [passes.h](framework/include/ir/transforms/passes.h)。

### Tensor 语义层：LogicalTensor、token、动态 shape、slot

- **[已验证]** [LogicalTensor](framework/src/interface/tensor/logical_tensor.h) 继承 ir::Var，关联 RawTensor、offset、shape、dynValidShape、storage、producer/consumer 和 read/write token。
- **[已验证]** [symbolic_scalar.h](framework/src/interface/tensor/symbolic_scalar.h) 表达动态 shape 和符号约束；CollectScalarVarRefs 会把 dynValidShape 中的标量视为真实 use。
- **[已验证]** TensorOpStmt 的 result_token、tokens 将内存访问顺序显式化；[InferTokenPass](framework/src/interface/ir/transforms/infer_token_pass.cpp) 和 RemoveRedundantTokenPass 分别建立、精简 WAW/WAR 等依赖。
- **[已验证]** [TensorSlotManager](framework/src/interface/tensor/tensor_slot.cpp) 与 RootFunctionBuilder 将 if/loop/call 两侧的逻辑 Tensor 绑定到一致运行时 slot，贯通结构化 IR 和动态设备程序。
- **[推断]** dynamic valid shape、内存 token 和 slot 的联合表达，是 PyPTO 为动态图与设备侧调度定制的核心 IR 语义，不只是通用 SSA 容器。

### tile_fwk 图体系：成熟 lowering 载体

- **[已验证]** [function.h](framework/src/interface/function/function.h) 的 FunctionType 和 GraphType 区分 dynamic、Tensor Graph、Tile Graph、Execute/Block/Leaf 等编译状态；Operation 还承载旧 Pass 使用的 producer/consumer、内存和调度元数据。
- **[已验证]** 新 IR 没有直接替换该体系，而是通过 CreateRootFunctions/RootFunctionBuilder 转入它。
- **[推断]** 当前是迁移期双 IR 架构：新 IR 改善前端结构化表示和验证，旧图体系继续承载已经成熟的自动 Tile 化、内存与调度 Pass。

---

## 问题五：最具竞争力的 Lowering 和 Pass 是怎样实现的？

**答案：前半段先把动态 Tensor 程序规范化并建立内存依赖，后半段用 PVC2_OOO 的 46 个 Pass 完成从 Tensor 图到硬件可生成图的自动映射。**

### 新 IR 前置流水

- **[已验证]** [compile_pipeline.py::_build_default_pipeline](python/pypto/pil/compile_pipeline.py) 当前依次运行：
  1. InferTokenPass；
  2. Canonicalize + AggressiveDCE 两轮；
  3. MergeStmtsIntoIf + Canonicalize；
  4. SimplifySymbolicScalar；
  5. RemoveRedundantTokenPass；
  6. CreateRootFunctions；
  7. FinalizeDynamicFunction。
- **[已验证]** InferToken/RemoveRedundantToken 把 Tensor alias 和读写危险转换为显式 token 边；Canonicalize/DCE 清理无效 carry 和死代码；MergeStmtsIntoIf 将后继语句下沉到分支并可用符号可满足性裁剪路径。
- **[推断]** 前置流水的目标不是完成硬件 lowering，而是把 Python 产生的动态图变成依赖明确、控制流闭合、适合分段进入旧图后端的 IR。

### 新旧 IR 的关键腰部

- **[已验证]** [RootFunctionBuilder](framework/src/interface/tensor/ir_func_builder.cpp) 为 dynamic root 中的 TensorOpStmt 段创建 hidden/path function，插入 CALL Operation，并建立 incast/outcast 与 Tensor slot。
- **[已验证]** [FinalizeDynamicFunction](framework/src/interface/tensor/ir.cpp) 完成动态图函数和编译任务提交。
- **[推断]** 这是当前技术路线最关键也最脆弱的接口：它让新前端复用成熟后端，但必须保持 SSA/token、alias、dynamic valid shape、控制流 carried value 与旧图 slot 语义等价。

### PVC2_OOO 多级图流水

- **[已验证]** [BuildPvc2OooPassEntries](framework/src/passes/pass_mgr/pass_manager.cpp) 固定注册 46 个 Pass，核心能力可归为：
  - 语义与格式：InferTensorFormat、AutoCast、RemoveRedundantReshape、InferMemoryConflict；
  - Tensor→Tile：ExpandFunction、SplitRawTensor、SplitReshape、GraphPartition、SubgraphToFunction；
  - 数据搬运与存储：AssignMemoryType、NBufferMerge、L1CopyInReuseMerge、GenerateMoveOp、PadLocalBuffer；
  - 融合与图优化：CommonOperationEliminate、ReduceCopyMerge、VFFusionClusterIdentify；
  - 调度与同步：OoOSchedule、TuneTileOpSeqForVF、InsertSync、TuneSyncForVF、MixSubgraphSplit；
  - 内存生命周期与生成准备：AddAlloc、RemoveAlloc、LastUseMark、GlobalMemoryReuse、CodegenPreproc。
- **[已验证]** ExpandFunction 是 Tensor Graph 阶段边界，SubgraphToFunction 是 Tile Graph 阶段边界；ExecuteGraph 另执行 DynAttrToStatic 和 LoopaxesProc。
- **[推断]** PyPTO 的主要编译护城河集中在这套 Pass：它把 Ascend 的 Cube/Vector 异构核、多级存储、搬运流水、同步和乱序执行约束编码成可组合的图变换，而不是把这些决策全部推给用户或外部后端。

---

## 问题六：CCE/PTO CodeGen、Bisheng 和 CANN/Ascend 的边界在哪里？

**答案：PyPTO 决定图结构、Tile 操作、内存、调度与同步，并生成 CCE/PTO 源码；Bisheng 把该源码编译为 AI Core 目标代码；CANN 提供工具链、运行库、通信和设备接口。**

| 层次 | 责任与证据 |
|---|---|
| PyPTO Pass | **[已验证]** 决定 format/cast、子图、memory type、alloc/reuse、TileOp 顺序、同步和 leaf function，见 framework/src/passes。 |
| PyPTO CodeGen | **[已验证]** [CodeGenFactory](framework/src/codegen/codegen_factory.h) 按 NPUArch 选择 CloudNPU/LiteNPU 实现；[CodeGenNPU::GenCode](framework/src/codegen/npu/codegen_npu.cpp) 把 Operation 转为 CCE/PTO TileOp 源码和编译任务。 |
| CCE/PTO 表达 | **[已验证]** CodeGen 输出 CCE C++ 与 PTO intrinsic/header；内存地址空间、Cube/Vector core type、Tile 指令和同步选择已经包含在生成源码中。 |
| Bisheng | **[已验证]** [CodeGenNPU::PrepareCmd](framework/src/codegen/npu/codegen_npu.cpp) 构造 bisheng -c -O3 -g -x cce 命令；Pro 的 [_run_bisheng](python/pypto_pro/runtime/jit.py) 用 -xcce、架构、PTO include 和 CANN link 参数生成共享库。 |
| CANN | **[已验证]** 顶层 [CMakeLists.txt](CMakeLists.txt) 通过 find_cann_package 依赖 acl_rt、ascend_dump、ascend_hal、hcomm、runtime、securec；adapter 层封装 ACL/Runtime/HAL/HCOMM/ADump/MsProf。 |
| Ascend 设备 | **[已验证]** 执行 Bisheng 产出的 AI Core 代码；主路线另有 AICPU 控制程序负责动态调用不同 leaf。 |

- **[推断]** PyPTO 与外部后端的分界不是“高层图交给 CANN 自动优化”，而是“PyPTO 已完成大部分领域特定 lowering，再把具体 CCE/PTO 程序交给 Bisheng 做目标编译”。
- **[推断]** 这种边界使 PyPTO 能深度控制 Ascend 特有的 Tile、Cube/Vector、内存和同步策略，同时也造成对 PTO headers、Bisheng 参数、CANN ABI 和设备版本的强耦合。
- **[未验证]** Bisheng 如何在内部进一步 lower CCE/PTO、是否以及如何使用 LLVM/MLIR，不在当前仓库中，不能由 PyPTO 源码确认。

---

## 问题七：哪些编译设施是复用的，哪些是 PyPTO 自研的？

| 设施 | 归属 | 判断 |
|---|---|---|
| Python inspect/ast | Python 标准库 | **[已验证] 复用。** ast2pil 读取源码并遍历 Python AST。 |
| pybind11/Python extension | 第三方/C++ binding | **[已验证] 复用。** python/src/bindings 将 IR、Tensor、Pass、Runtime 暴露给 Python。 |
| Torch/torch_npu Tensor 接入 | PyTorch/Ascend 生态 | **[已验证] 复用。** JitCallableWrapper 和 runtime binding 接收 torch.Tensor/stream。 |
| PIL 前端表示与 dispatcher | PyPTO | **[已验证] 自研。** 位于 python/pypto/pil。 |
| pypto::ir、IRBuilder、Visitor/Mutator、Verifier | PyPTO | **[已验证] 自研。** 位于 framework/include/ir 与 framework/src/interface/ir。 |
| LogicalTensor、SymbolicScalar、token、slot | PyPTO | **[已验证] 自研领域抽象。** 位于 framework/src/interface/tensor。 |
| tile_fwk Function/Operation、多级图 Pass | PyPTO | **[已验证] 自研核心 lowering。** 位于 framework/src/interface 与 framework/src/passes。 |
| CCE/PTO CodeGen | PyPTO 生成逻辑 + CANN PTO 接口 | **[已验证] PyPTO 自研生成器复用外部 PTO 指令/header。** |
| Bisheng | CANN/Ascend 外部工具链 | **[已验证] 直接复用。** PyPTO 通过命令行调用，不包含其编译器实现。 |
| ACL、runtime、HAL、HCOMM、dump/profiling | CANN | **[已验证] 直接复用。** 由 find_cann_package 和 adapter 接入。 |
| LLVM 独立工具 | LLVM 工具集 | **[已验证] 辅助复用。** PvModelImpl.h 调用 llvm-objcopy；异常文档使用 llvm-symbolizer，不属于核心 lowering。 |
| LLVM/MLIR 编译框架 | 未发现直接依赖 | **[已验证] 当前 CMake/源码没有 find_package、include、link 或 IR lowering 证据。** |

**关于 MLIR/LLVM 的准确结论：**

- **[已验证]** PyPTO 当前不是 MLIR-based compiler；核心 IR、Pass、Verifier、多级图和 CCE/PTO CodeGen 均为仓库内自研实现。
- **[已验证]** 仓库有 PTO MLIR 术语痕迹，例如 [ptr_ops.py](python/pypto_pro/ir/op/ptr_ops.py)、[PtrType](framework/include/ir/type.h)、[codegen_base.h](framework/include/pypto_pro/codegen/codegen_base.h) 和 jit.py::_get_mlir_code，但当前生产路径实际调用 _codegen_target_cce/CCECodegen，未发现 MLIR backend。
- **[已验证]** Pro 通过 [JitCompileConfig](python/pypto_pro/runtime/compile_config.py) 向 Bisheng 透传 -mllvm 参数；这证明存在 LLVM 风格的外部后端参数接口，不证明 PyPTO 直接使用 LLVM API 或 LLVM IR。
- **[推断]** PyPTO IR 的 SSA、类型系统、MemRef、结构化控制流、Pass 和 Verifier 与现代编译器 IR 有概念相似性。
- **[未验证]** 当前没有官方设计说明证明 PyPTO IR 由 MLIR 派生、兼容 MLIR、以 MLIR 为模板，或与 MLIR 设计完全无关。因此“不是 MLIR”不能被扩大为“所有 IR 设计风格完全不一致”。

---

## 问题八：为理解编译产物，最少需要知道哪些 Runtime 内容？

**答案：只需区分主路线的动态 MPMD 产物和 Pro 的单 kernel SPMD 产物。**

### 主 pypto 产物

- **[已验证]** [CompileDyndevFunction](framework/src/machine/host/backend.cpp) 依次执行 ExecuteGraph、构建控制流/表达式表、编译 AICPU 控制程序、生成并编译 AI Core leaf、编码 DevAscendProgram。
- **[已验证]** [KernelModule](python/src/bindings/runtime.cpp) 与 [DeviceLauncher](framework/src/machine/runtime/launcher/device_launcher.cpp) 负责匹配参数、准备 workspace/slot/stream 并启动设备程序。
- **[已验证]** AICPU 根据控制流和 task ID 调用不同 AI Core 子函数，入口协议见 [aicore_entry.h](framework/src/interface/machine/device/tilefwk/aicore_entry.h)。
- **[推断]** Runtime 在主路线中不是普通 kernel launcher，而是编译器动态控制流语义的执行端；这解释了为什么 token、slot、path/hidden function 和 AICPU 编译属于编译链核心。

### pypto_pro 产物

- **[已验证]** Pro 生成一个内容 hash 命名的 call_kernel_<hash>.so，Python 用 ctypes 加载 call_kernel，并传入 stream、block_dim 和 ABI 参数。
- **[已验证]** 多个逻辑 AI Core 执行同一 kernel，通过 block index 分工，属于 SPMD。
- **[推断]** Pro Runtime 更接近传统 JIT kernel launcher，复杂性主要留在显式 DSL、CCECodegen 和 Bisheng 编译阶段。

除上述 ABI、控制流和启动模型外，workspace 分配细节、profiling、dump、分布式初始化和发布脚本不影响本报告对编译技术路线的判断，因此不展开。

---

## 问题九：这条技术路线相对同类编译器的差异化优势是什么？

### 1. 高层动态语义与底层 Tile 编译在一套系统内闭环

- **[已验证]** Python 分支/循环、SymbolicScalar、dynValidShape、token、slot、AICPU 控制流和 AI Core leaf 存在贯通实现。
- **[推断]** 这使 PyPTO 能处理“运行时变化，但仍需生成高性能 Tile 程序”的场景，区别于只能静态展开或只生成单个 SPMD kernel 的简单 DSL。

### 2. 硬件感知决策前移到领域 Pass

- **[已验证]** PVC2_OOO 显式处理 Cube/Vector、format、memory type、L1 reuse、N-buffer、VF fusion、OOO schedule、sync 和 global memory reuse。
- **[推断]** PyPTO 不完全依赖通用编译器后端猜测 Tensor 语义，而是在信息尚未丢失时完成 Ascend 领域优化，理论上更容易取得稳定性能。

### 3. 自动与显式两种生产力层级并存

- **[已验证]** 主 pypto 提供 Tensor 自动 lowering；pypto_pro 提供 Tile/Reg/SIMT 和 Cube/Vector section。
- **[推断]** 同一产品可覆盖算法开发者和性能专家，并有机会让 Pro kernel 成为主 Tensor 图的高性能叶子实现。

### 4. 与 CANN 垂直集成

- **[已验证]** 编译器直接掌握 PTO 源码、Bisheng 参数、CANN runtime、HCOMM、设备控制流和 profiling/dump 接口。
- **[推断]** 垂直集成缩短了新硬件能力到 DSL/Pass 的路径，但可移植性和工具链兼容成本高于后端中立方案。

---

## 问题十：核心编译路线目前最大的风险和未完成部分是什么？

### 1. 新旧 IR 桥接是最高风险点

- **[已验证]** 新 pypto::ir 经 CreateRootFunctions 转入旧 tile_fwk::Function/Operation；两侧分别维护 SSA/token 与 producer/consumer/GraphType/slot。
- **[推断]** alias、dynamic valid shape、控制流 carried value 或 slot 的任何不等价，都可能在较晚的 Pass、内存复用或设备执行阶段表现为精度问题。

### 2. 两条路线共享底座但后端尚未统一

- **[已验证]** 主路线走 PVC2_OOO 和 MPMD；Pro 走 ConvertToSSA、直接 CCECodegen 和 SPMD。
- **[推断]** 若 Tensor/Tile/MemorySpace/同步能力在两条路线中独立演进，可能产生概念重名、能力表重复和行为差异。

### 3. 新通用 IR 的完整 lowering 尚未落地

- **[已验证]** [passes.cpp](framework/src/interface/ir/transforms/passes.cpp) 中 InitMemRef、BasicMemoryReuse、AllocateMemoryAddr、OutlineIncoreScopes、ConvertTensorToBlockOps、FlattenCallExpr、NormalizeStmtStructure、FlattenSingleStmt 当前返回 identity/no-op。
- **[推断]** 新 IR 目前不能独立承担完整 Tensor→设备 lowering，主路线仍依赖旧图后端。

### 4. 默认路径与部分文档仍处迁移期

- **[已验证]** entry.py 的 jit 默认 new_ir=True，而 [developer_doc_zh.md](python/pypto/frontend/developer_doc_zh.md) 仍重点描述旧 doc-AST/Parser 路线；new_ir=False 仍保留兼容入口。
- **[推断]** 开发者可能在错误的前端层实现功能，或只验证其中一条路径。

### 5. 编译 session 和工具链耦合限制工程扩展

- **[已验证]** IRContext::Get()、Program::GetInstance()、ConfigManager、HostMachine 等存在进程级状态；Python 批量测试要求 --forked 隔离。
- **[已验证]** 生成与运行依赖 CANN、PTO headers、Bisheng、driver 和具体 NPUArch。
- **[推断]** 多线程并行 JIT、跨版本复现、社区无设备开发和跨后端移植成本较高。

---

## 问题十一：Git 历史显示技术路线正在怎样演进？

**答案：2026 年 7 月以后，主线明显同时推进“新 IR 接入成熟后端”和“Pro 显式直达 CCE”两条方向。**

| commit | 日期 | 已验证的路线信号 |
|---|---|---|
| 062c219f2881116a83d916fe65805f5c1a2921f7 | 2026-07-07 | refactor(frontend): Wire new IR compile pipeline and add jit flag；新 IR 正式接入 JIT。 |
| 7964a2a4f71a4eca147c2a460e75d2e9f345f873 | 2026-07-08 | feat(ir): Add host_machine to compile_pipeline；新 IR 接到既有 HostMachine。 |
| c944f45988e94a86b28ee8b1620f6e524cd8a9fb | 2026-07-18 | feat(pypto_pro): Add pypro_pro codegen and frontend；Pro 前端和 codegen 落地。 |
| 50a7132d9577e1b70177b3a6e961dd8045a10742 | 2026-08-13 | feat(ir): Infer read and write tokens in tensor graphs；新 IR 补强内存依赖。 |
| a55a702506bd190f1dc2f570540f557dcb69f2a2 | 2026-08-20 | feat(ir): Add remove_redundant_token_pass；token 管线继续完善。 |
| d71f7ed16ed2c6b25c805510deb7e2a6df21d128 | 2026-08-21 | feat(pypto_pro): Support pypto_pro SIMT；Pro 扩展到显式 SIMT。 |
| e23f9ee7dedd1ca2980418547ed9b9dc128abceb | 2026-08-28 | feat(pypto_pro): Remove pl.function and pl.program；Pro API/IR 仍在快速收敛。 |

- **[已验证]** 当前基线为 5d6afbe0d，git describe 为 v9.2.0-beta.2-322-g5d6afbe0d；version.cmake 的 CANN 产品包版本是 9.2.0，pyproject.toml 的 Python distribution 版本是 0.2.1。
- **[推断]** 主路线采取的是“新前端/新 IR 渐进替换、成熟后端继续复用”，而不是一次性重写全部编译器；Pro 则作为更短的新路径快速迭代。
- **[未验证]** 仓库没有给出两条路线最终是否合并、何时移除旧前端/旧图、是否引入 MLIR backend 的正式路线图。

---

## 问题十二：后续演进应优先做什么，才能强化核心竞争力？

1. **[建议] 把 RootFunctionBuilder 变成有显式契约的 lowering 边界。** 在桥接前后验证 alias/raw-memory ID、token、dynValidShape、control-flow carry、incast/outcast 和 slot 等价，并建立最小差分 golden。
2. **[建议] 渐进迁移成熟 Pass，而不是重写 PVC2_OOO。** 先选择 token/sync、format/type inference、简单 elementwise Tile lowering 等可独立验证环节，在新 IR 和旧图上做差分。
3. **[建议] 统一两条路线的硬件语义事实源。** MemorySpace、layout、Tile 能力、dtype 约束、Cube/Vector/SIMT、同步与 SoC feature 应由同一套 schema/query 提供。
4. **[建议] 明确主 Tensor 图调用 Pro leaf 的 ABI。** 若将 Pro 作为手工优化叶子，需要定义 shape specialization、workspace、stream、异常、profiling、cache key 和调度接口。
5. **[建议] 收口未实现 Pass。** 对 identity/no-op factory 标注 experimental/deprecated，或为其定义 owner、IRProperty、里程碑和端到端测试。
6. **[建议] 引入显式 CompilationSession。** 逐步封装 Program、IRContext、配置、ID generator 和缓存，减少全局状态对并行 JIT 和可复现性的限制。
7. **[建议] 建立 Python→PIL→IR→Graph→CCE→device task 的统一追踪 ID。** 让源码 span、IR stage、Function hash、kernel name 和设备 task ID 可关联，降低跨层定位成本。

- **[推断]** 最有价值的演进方向不是增加更多外围 API，而是让两条编译路线共享更多硬件语义、提高新旧 IR 桥接可证明性，并逐步形成“高层自动编译可调用低层专家 kernel”的组合能力。

---

## 关键证据索引

### Python 前端与新 IR

- [python/pypto/frontend/parser/entry.py](python/pypto/frontend/parser/entry.py)：JitCallableWrapper、compile_new、jit(new_ir=True)
- [python/pypto/pil/parser.py](python/pypto/pil/parser.py)：ast2pil、visit_if/for/while
- [python/pypto/pil/pir.py](python/pypto/pil/pir.py)：PIL 数据结构、BuildContext、Scope、Journal
- [python/pypto/pil/dispatcher.py](python/pypto/pil/dispatcher.py)：dispatch_block、dispatch_call
- [python/pypto/pil/ops.py](python/pypto/pil/ops.py)：if_else_impl、loop_impl、carriable_names
- [python/pypto/pil/compile_pipeline.py](python/pypto/pil/compile_pipeline.py)：默认新 IR pipeline

### IR、桥接和 Pass

- [framework/include/ir/](framework/include/ir/)：core/expr/stmt/type/builder/function/program、transforms、verifier 接口
- [framework/src/interface/ir/](framework/src/interface/ir/)：通用 IR 实现与 Pass
- [framework/src/interface/tensor/](framework/src/interface/tensor/)：LogicalTensor、SymbolicScalar、token、slot、RootFunctionBuilder
- [framework/src/interface/function/function.h](framework/src/interface/function/function.h)：FunctionType、GraphType
- [framework/src/passes/pass_mgr/pass_manager.cpp](framework/src/passes/pass_mgr/pass_manager.cpp)：PVC2_OOO、ExecuteGraph
- [framework/src/passes/](framework/src/passes/)：Tensor/Tile/Block Graph Pass 实现

### CodeGen、外部工具链与最小 Runtime

- [framework/src/codegen/codegen_factory.h](framework/src/codegen/codegen_factory.h)：CodeGenFactory
- [framework/src/codegen/npu/codegen_npu.cpp](framework/src/codegen/npu/codegen_npu.cpp)：CodeGenNPU、PrepareCmd
- [framework/src/machine/host/backend.cpp](framework/src/machine/host/backend.cpp)：CompileDyndevFunction
- [framework/src/machine/compile/aicore_compiler.cpp](framework/src/machine/compile/aicore_compiler.cpp)：AI Core 编译
- [framework/src/machine/runtime/launcher/device_launcher.cpp](framework/src/machine/runtime/launcher/device_launcher.cpp)：DeviceLauncher
- [python/pypto_pro/runtime/kernel.py](python/pypto_pro/runtime/kernel.py)：KernelDef
- [python/pypto_pro/language/parser/_ast_parser.py](python/pypto_pro/language/parser/_ast_parser.py)：ASTParser
- [framework/src/interface/pypto_pro/codegen/cce/cce_codegen.cpp](framework/src/interface/pypto_pro/codegen/cce/cce_codegen.cpp)：CCECodegen::GenerateSingle
- [python/pypto_pro/runtime/compile_config.py](python/pypto_pro/runtime/compile_config.py)：CCE_BACKEND、JitCompileConfig
- [python/pypto_pro/runtime/jit.py](python/pypto_pro/runtime/jit.py)：_codegen_target_cce、_run_bisheng
- [CMakeLists.txt](CMakeLists.txt)：CANN package 依赖

## 调研边界

- **[已验证]** 本报告只分析固定 commit 5d6afbe0d 的当前仓库内容和本地 Git 元数据。
- **[已验证]** 本次未修改仓库源代码；报告文件是唯一目标制品。
- **[未验证]** 本机没有配置 CANN/NPU/Bisheng，未执行完整构建或上板测试；“存在某能力”的结论来自源码、测试和配置，不等价于本机动态验证。
- **[未验证]** 对同类编译器的差异化判断没有引入外部竞品仓库做逐项基准，因此本报告只说明 PyPTO 自身实现形成的可观察差异，不声称性能领先。
