---
title: Triton-Ascend 编译栈与差异化竞争力
date: 2026-09-08 09:00:00
categories:
  - agentdocs
tags:
  - compiler-stack
  - ai-systems
---

# Triton-Ascend 技术路线静态调研报告

> 调研日期：2026-09-07（Asia/Shanghai）  
> 仓库 URL：`https://github.com/triton-lang/triton-ascend.git`  
> 本地分支：`main`  
> 基线 commit：`b64287188046fe7bb0cfd424ee3e389f96a7affa`（`[ssbuffer](fix) fix vf operand substitution (#2033)`）  
> `git describe`：`v3.2.0-4022-g-b64287188`  
> AscendNPU-IR gitlink：`aea934a66646e837c54fea11e87db54d42eb3221`

## 0. 问题：本报告的证据范围、状态标记和限制是什么？

**答案：**

- **[已验证]** 本报告只依据上述本地 commit 的源码、CMake/Python 构建文件、仓库内文档和 Git 元数据；远端 `origin/main` 在调研时也指向该 commit。关键 Git 证据来自 `.git/config`、`.gitmodules`、`git show HEAD`、`git log` 和 `git diff 85400f8..HEAD`。
- **[已验证]** 工作树调研前为干净状态；本报告没有执行安装、构建或设备测试。原因之一是 `setup_ascend.py:CMakeBuild.run()` 会调用 `_apply_triton_ascend_patch()`，主动改写一组上游 Triton 文件并使工作树变脏，这与“仅修改报告文件”的要求冲突。
- **[已验证]**`third_party/ascend/AscendNPU-IR` 在本地是**未初始化的子模块**（`git submodule status` 前缀为 `-`），只能验证 `.gitmodules` 的 URL、gitlink commit 以及上层 CMake/源码对其公开库名和 dialect 名的引用，不能验证子模块内部实现。
- **[未验证]** 未验证真实 CANN/BiShengIR 工具版本的命令行行为、生成 ELF 的内部段、设备指令、性能和正确性，也未确认文档中的外部链接内容或远端是否存在更新 commit。
- 状态含义：**[已验证]**＝可由当前快照直接定位到文件、符号或 Git 记录；**[推断]**＝由多处已验证事实得出的合理解释；**[未验证]**＝当前快照或环境不足以确认。

---

## 1. 问题：Triton-Ascend 最具特色、最有竞争力的部分是什么？

**答案：** Triton-Ascend 的核心竞争力不是“把 Triton 换成 Ascend 后端”这么简单，而是保留 Triton 的 Python/TTIR 编程入口，同时在 TTIR 之后建立一条面向 Ascend 微架构的编译分叉：用结构化 tile、Linalg/memref、Cube/Vector、UB、pipe、同步和多缓冲等抽象，把通用 Triton 程序转成适合 Ascend 执行的代码。

| 竞争力 | 已验证实现 | 与普通 Triton 后端的差异 |
|---|---|---|
| TTIR 后的 Ascend 专属编译分叉 | `third_party/ascend/backend/compiler.py`：`make_ttir()`、`ttir_to_linalg()`、`ttir_to_npubin()` | 不照搬 NVIDIA 的 TTGIR/warp-layout 路径，转向 Ascend 的结构化 tile 与 NPU 编译链 |
| Ascend 异构硬件建模 | `third_party/ascend/include/Dialect/TritonAscend/IR/`、`third_party/ascend/include/Dialect/TritonStructured/IR/` | 显式表达 Cube/Vector、存储层级、pipe、同步和多缓冲，服务 Ascend NPU 微架构 |
| 编译设施复用而非完全重写 | `python/triton/`、标准 MLIR/BiShengIR 接缝、`third_party/ascend/bin/triton-mlir-opt.cpp` | 复用 Triton 前端和 MLIR 基础设施，在后端加入 Ascend dialect/pass 与 BiShengIR 适配 |
| 面向真实 NPU 性能的优化 | `third_party/ascend/backend/compiler.py` 中 DynamicCV、SSBuffer、multi-buffer 等配置 | 优化目标从 GPU warp/布置转为 Ascend 的 Cube/Vector 协同和片上存储流水 |

## 2. 问题：本报告应该优先回答哪些技术问题？

**答案：** 阅读顺序按技术竞争力排序，而不是按仓库目录顺序。

| 优先级 | 核心问题 | 报告对应章节 |
|---|---|---|
| 第一 | TTIR 之后如何分叉到 Ascend 编译链 | 第 3、5 章 |
| 第二 | Triton、MLIR、LLVM、BiShengIR 哪些被复用 | 第 4、7 章 |
| 第三 | Ascend 自研 dialect、Pass、layout 和硬件优化是什么 | 第 8 章 |
| 第四 | 这条路线与 NVIDIA Triton 的根本差异是什么 | 第 7 章 |
| 辅助 | 编译产物如何加载和执行 | 第 9 章 |

## 3. 问题：项目定位和目标是什么？

**答案：**

- **[已验证]** Triton-Ascend 是 Triton 的 Ascend NPU 适配与优化版本，目标是在尽量保留 Triton Python 编程模型、JIT、缓存和通用 IR 基础设施的同时，把 Triton kernel 编译为 Ascend 设备二进制并通过 CANN 运行时加载、发射。证据：`README_zh.md`；`docs/zh/installation_guide.md`；`docs/zh/architecture_design_and_core_features.md` 的 “Ascend language extension / compiler / driver” 三层划分。
- **[已验证]** 目标硬件在当前用户文档中是 Linux 上的 Atlas A2/A3/950 系列；后端 target 判定为 `GPUTarget.backend == "npu"`。证据：`README_zh.md`；`third_party/ascend/backend/compiler.py:AscendBackend.supports_target()`、`NPUDriver.get_current_target()`。
- **[已验证]** 项目不只是“换一个 launcher”：它同时增加 Ascend 语言扩展、Ascend/TritonStructured dialect、TTIR 到 Linalg/HIVM/HFusion 的转换、BiShengIR 编译器对接、CANN/ACL 运行时封装以及 NPU 专用测试。证据：`third_party/ascend/language/`、`third_party/ascend/include/`、`third_party/ascend/lib/`、`third_party/ascend/backend/`、`third_party/ascend/unittest/`。
- **[推断]** 技术路线可以概括为“**复用上游 Triton 前端和 MLIR 编译框架，在 TTIR 后分叉到面向 Ascend 的结构化/缓冲化 IR 与厂商编译器，再用 CANN runtime 发射**”，而不是复用 NVIDIA 的 TTGIR→LLVM→PTX 路线。

## 4. 问题：Python Triton 前端到 Ascend 后端的完整编译链是什么？

**答案：**

### 4.1 通用入口

- **[已验证]**`@triton.jit` 的 `JITFunction.run()` 形成参数签名、constexpr 和 grid 后进入通用编译器；`ASTSource.make_ir()` 调用 `ast_to_ttir()`，由 Python AST/语义构造 TTIR `ModuleOp`。证据：`python/triton/runtime/jit.py:JITFunction.run()`；`python/triton/compiler/compiler.py:ASTSource.make_ir(), compile()`；`python/triton/compiler/code_generator.py:CodeGenerator, ast_to_ttir()`。
- **[已验证]** 通用 `compile()` 通过 entry point 发现后端，按 target 选中 `AscendBackend`，调用 `parse_options()`、`load_dialects()`、`add_stages()`，逐 stage 执行并把 IR、二进制和 JSON metadata 写入 Triton cache。证据：`python/triton/backends/__init__.py:_discover_backends()`；`python/triton/compiler/compiler.py:compile(), make_backend()`；`setup.py:get_entry_points()`；`setup_ascend.py:_patch_module(), _build_setup_kwargs()`。
- **[已验证]** Ascend 构建把 `third_party/ascend/backend` 安装为 `triton.backends.ascend`，把 `third_party/ascend/language/cann` 安装为 `triton.language.extra.cann`。证据：`setup.py:BackendInstaller, get_package_dirs(), add_link_to_backends()`；`setup_ascend.py:_patch_module()`；`third_party/ascend/backend/name.conf`。

### 4.2 普通路径（`compile_mode=simd` 或默认 `simd_simt_template`）

**[已验证]** 实际 stage 顺序由 `AscendBackend.add_stages()` 明确定义：

```text
Python AST
  → TTIR ModuleOp                    ast_to_ttir
  → 优化后 TTIR                     make_ttir
  → ttadapter 文本 MLIR             ttir_to_linalg
  → MLIR bytecode                   linalg_to_bc_by_triton_mlir_opt
  → bcmlir 文本 MLIR                bc_to_linalg_by_bishengir_opt
  → npubin（kernel.o/reloc.o 字节）  linalg_to_bin_enable_npu_compile_*
  → Triton cache + metadata JSON
```

- **[已验证]**`make_ttir()` 复用通用 inliner、canonicalizer、CSE、LICM、symbol-DCE，以及 TTIR combine、tensor-descriptor rewrite、broadcast reorder、loop unroll；默认还调用 Ascend graph optimization。证据：`third_party/ascend/backend/compiler.py:make_ttir()`。
- **[已验证]**`ttir_to_linalg()` 的 Ascend pass 序列包括：control-flow-opt → structure → discrete-mask conversion → annotation → unstructure → TritonToHIVM → TritonToHFusion → TritonToLLVM → bubble-up → 再 structure → TritonToLinalg；随后可接 DynamicCV pipeline、buffer-count 设置和 coalescing metadata 导出。证据：`third_party/ascend/backend/compiler.py:ttir_to_linalg()`；各实现位于 `third_party/ascend/lib/TritonControlFlowOpt/`、`TritonToStructured/`、`DiscreteMaskAccessConversion/`、`TritonToUnstructure/`、`TritonToHIVM/`、`TritonToHFusion/`、`TritonToLLVM/`、`TritonToLinalg/`、`DynamicCVPipeline/`。
- **[已验证]**`triton-mlir-opt --emit-bytecode` 负责把同时含标准 MLIR 与 BishengIR dialect 的 ttadapter 文本序列化为 MLIR bytecode；`bishengir-opt` 再读 bytecode 并输出文本。这是序列化/兼容交接 stage，不是 LLVM bitcode。证据：`third_party/ascend/backend/compiler.py:linalg_to_bc_by_triton_mlir_opt(), bc_to_linalg_by_bishengir_opt()`；`third_party/ascend/bin/triton-mlir-opt.cpp`。
- **[已验证]** 最后按架构选择 `linalg_to_bin_enable_npu_compile_910_95()` 或 `linalg_to_bin_enable_npu_compile_A2_A3()`，调用 `bishengir-compile`（找不到时可由 `TRITON_NPU_COMPILER_PATH` 指定兼容工具），读取 `kernel.o` 或 `kernel_reloc.o`；若产生 `libkernel.so`，还读取 task type、workspace、同步锁布局等回调 metadata。证据：`third_party/ascend/backend/compiler.py` 同名函数；`third_party/ascend/backend/utils.py:_get_npucompiler_path(), _check_bishengir_api_change()`。

### 4.3 Ascend 950 纯 SIMT 路径

- **[已验证]**`compile_mode="simt_only"` 只允许 A5 target（`Ascend910_95*`/`Ascend950*`），`AscendBackend.add_stages()` 直接注册 `ttir → npubin`，跳过 ttadapter/Linalg/bytecode stages。证据：`third_party/ascend/backend/compiler.py:_normalize_compile_mode(), NPUOptions.__post_init__(), AscendBackend.add_stages(), ttir_to_npubin()`。
- **[已验证]** 此路径向厂商编译器传 `--enable-triton-ir-compile --pure-simt --num-warps --threads-per-warp` 等参数，并可设置 SIMT stack、dynamic shared memory、reorder、FMA 和 auto-blockify。证据：`third_party/ascend/backend/compiler.py:ttir_to_npubin()`。

## 5. 问题：TTIR、TTGIR、LLVM IR/MLIR 在本项目中是什么关系？

**答案：**

- **[已验证] TTIR** 是 Triton dialect 的 MLIR，是 Python AST lowering 后的首个编译器 IR；Ascend 普通路径和纯 SIMT 路径都从它出发。证据：`ASTSource.ext="ttir"`、`ASTSource.make_ir()`；`AscendBackend.add_stages()`。
- **[已验证] TTGIR**（TritonGPU IR）仍存在于继承的上游源码，例如 `include/triton/Conversion/TritonToTritonGPU/`、`include/triton/Dialect/TritonGPU/` 和 NVIDIA/AMD 后端，也能被通用 `IRSource` 解析；但 Ascend `add_stages()`**没有注册 `ttgir` stage**。因此它不是当前 Ascend 正常编译链的一环。
- **[未验证]** 当前 Ascend 后端是否支持用户直接以 `.ttgir` 文件作为可工作的编译入口。通用解析器认识扩展名不等于 Ascend pipeline 能继续处理；当前 `add_stages()` 的 stage 表没有 TTGIR，不能据此宣称支持。
- **[已验证] MLIR** 是总的 IR 基础设施：TTIR、标准 `arith/scf/tensor/memref/linalg/bufferization` dialect、MLIR LLVM dialect、Ascend dialect，以及子模块提供的 HIVM/HFusion/HACC 等 BishengIR dialect，都共存在 MLIR module/bytecode 中。证据：根 `CMakeLists.txt` 的 `find_package(MLIR REQUIRED)` 与 MLIR link targets；`third_party/ascend/bin/triton-mlir-opt.cpp`；Ascend 各 pass 的 CMake link targets。
- **[已验证]**`TritonToLLVM` 在 ttadapter lowering 中把部分 TTIR 操作转成 **MLIR LLVM dialect**；`third_party/ascend/include/Dialect/TritonAscend/IR/TritonAscendDialect.td` 也声明依赖 `mlir::LLVM::LLVMDialect`。这仍然是 MLIR，不等同于已经导出的文本 LLVM IR。
- **[已验证]** Ascend 的 stage 表没有上游 GPU 后端常见的 `llir` stage，也没有在 Python 侧调用 LLVM target machine 生成 PTX/HSACO。最终设备二进制由 `bishengir-compile` 接管。
- **[未验证]** BiShengIR 内部是否、何时把混合 MLIR 转为原生 LLVM IR，以及最终是否经过 LLVM 后端生成设备 ISA；本地未展开的 AscendNPU-IR 子模块和外部 `bishengir-compile` 才掌握该部分。仓库中的调试变量 `LLVM_IR_ENABLE_DUMP` 只能证明工具链可涉及 LLVM IR，不能补足内部实现证据。

## 6. 问题：项目是否直接依赖 MLIR/LLVM？哪些来自 Triton 上游，哪些是 Ascend 适配？

**答案：**

- **[已验证]** 项目在**构建时直接依赖 LLVM/MLIR**：根 `CMakeLists.txt` 通过 `MLIR_DIR` 执行 `find_package(MLIR REQUIRED CONFIG)`，包含 `AddLLVM/AddMLIR`，链接 `MLIRIR`、`MLIRPass`、`MLIRLLVMDialect`、`MLIRTargetLLVMIRExport`、`LLVMPasses` 等；大量 C++ 源码直接 `#include "mlir/..."` 和 `#include "llvm/..."`。
- **[已验证]** 当前 main 将 LLVM 源码 revision 固定为 `f6ded0be897e2878612dd903f7e8bb85448269e5`（`cmake/llvm-hash.txt`），Ascend 打包入口又把 `third_party/ascend/patch/llvm_patch_f6ded0b.patch` 的 hash 纳入预编译包名，并从 Ascend OBS 下载定制 LLVM 包。证据：`setup_ascend.py:_get_ascend_llvm_package_info()`。
- **[已验证] 上游复用部分**：`python/triton/` 的 JIT/compiler/cache/runtime，`include/triton/`、`lib/` 的 Triton/TTGIR/LLVM 通用 dialect 与 pass 基础设施，根 CMake、`setup.py`，以及 `third_party/nvidia/`、`third_party/amd/` 均来自或持续同步上游 Triton。
- **[已验证] Ascend 适配部分**：`third_party/ascend/` 的 backend/language/dialect/pass/runtime/test、`setup_ascend.py`、Ascend LLVM patch、`python/triton/extension/buffer/`，以及构建时施加的 `triton-ascend-3.6.0.patch`/dev patch。
- **[已验证]**`setup_ascend.py:_get_triton_ascend_patch_file()` 明列构建时需要修改的上游文件（含根 CMake、Triton attrs/traits、Python IR binding、compiler、language semantic/standard/math、runtime、工具注册等）；`third_party/ascend/backend/__init__.py:_apply_ascend_patch()` 还在运行时 monkey-patch `CodeGenerator`、IR override parser 和 `TritonSemantic.dot`。这说明当前边界是“插件目录为主，但仍有受控的上游补丁”，不是完全零侵入插件。
- **[已验证]** Git 元数据与 `docs/zh/release_note.md` 把 main 的上游 Triton 基准标为 `85400f80bf859a34ad7a746ffda877faf80312ab`（Triton 3.6.0）；历史中存在 merge commit `50c50cc7d`，提交信息为合入 `triton-lang/triton:3.6.0(85400f)`。该 commit 是当前 HEAD 的祖先。
- **[推断]** “直接依赖 LLVM/MLIR”主要指编译器自身和插件的构建/IR 基础设施；用户运行已构建 wheel 时，LLVM/MLIR 实现通常已进入 `libtriton`/工具二进制，而设备最终 codegen 另依赖 CANN/BiShengIR 工具链。

## 7. 问题：Ascend 后端的 lowering、代码生成、编译器调用和运行时分别怎样实现？

**答案：**

- **[已验证] Lowering 层**：`make_ttir()` 做通用 TTIR 优化；`ttir_to_linalg()` 负责 Ascend 转换和优化。核心方向是把规则/非规则指针与 mask、控制流、算术、reduction、view、dot 等变成更适合厂商工具链的 Linalg、memref、HIVM、HFusion、LLVM dialect 组合。证据：`third_party/ascend/backend/compiler.py`；`third_party/ascend/lib/TritonToLinalg/TritonToLinalgPass.cpp` 及 converter 文件；`docs/zh/architecture_design_and_core_features.md`。
- **[已验证] 优化层**：存在 graph/layout-memory optimization、AutoBlockify、DynamicCV pipeline、multi-buffer/SSBuffer、Cube/Vector block planning、同步注入与求解、HFusion/VF fusion 等 NPU 专用 pass 或 compiler option。证据：`third_party/ascend/lib/TritonToGraph/`、`AutoBlockify/`、`DynamicCVPipeline/`；`NPUOptions`。
- **[已验证] Codegen 交接层**：本仓先用 `triton-mlir-opt` 做 MLIR bytecode 序列化，再用外部 `bishengir-opt` 重读，最后用 `bishengir-compile` 生成 `kernel.o`/`kernel_reloc.o`；纯 SIMT 则直接把 TTIR 文件交给 `bishengir-compile`。证据：`AscendBackend.add_stages()` 及对应函数。
- **[已验证] 编译 metadata**：从 lowered IR 正则提取 `mix_mode`、`parallel_mode`、kernel name、tensor kinds、bitcode；从 `libkernel.so` callback 提取 `bs_task_type`、`workspace_size`、`sync_block_lock_layout`、`lock_init_val`，供 launcher 决定资源和 ABI。证据：`third_party/ascend/backend/compiler.py:_parse_linalg_metadata(), __get_metadata_attr_by_callback()`。
- **[已验证] Runtime 层**：通用 `CompiledKernel` 懒加载 npubin；Ascend 构建 patch 将调用改成 `load_binary(kernel_name, bytes, shared, device, mix_mode)`。`NPUUtils` 动态编译/缓存 `npu_utils.cpp` 扩展并注册二进制；`NPULauncher` 按 kernel signature 动态生成/编译/缓存 C++ launcher。证据：`python/triton/compiler/compiler.py:CompiledKernel`；`third_party/ascend/patch/triton-ascend-3.6.0.patch`；`third_party/ascend/backend/driver.py:NPUUtils, NPULauncher, make_launcher()`。
- **[已验证]** CANN 9.1+ 分支用 `aclrtBinaryLoadFromData()`、`aclrtBinaryGetFunction()` 和 `aclrtLaunchKernelWithHostArgs()`；旧兼容分支用 `rtDevBinaryRegister()`、`rtFunctionRegister()`、`rtKernelLaunch[WithFlagV2]()`。证据：`third_party/ascend/backend/npu_utils.cpp:cann_register_kernel()`；`third_party/ascend/backend/driver.py:generate_npu_header_src()`。

## 8. 问题：CANN、GE、ACL、torchair、torch_npu 的边界在哪里？

**答案：**

- **[已验证] CANN** 是外部软件栈边界：提供 Ascend toolkit 路径、编译器/`bishengir-*` 工具、runtime/ACL headers 与 `libruntime`、`libascendcl`，也提供设备运行环境。证据：`README_zh.md`、`docs/zh/installation_guide.md`；`third_party/ascend/backend/utils.py:_get_ascend_path(), _build_npu_ext()`；构建命令链接 `-lruntime -lascendcl`。
- **[已验证] ACL/aclrt** 是 CANN 的直接 runtime API 边界之一，负责设备选择、流、内存、二进制加载、函数句柄与 kernel launch；CANN 9.1 的宏分支切换到 aclrt，旧版保留 `rt*` 兼容层。证据：`third_party/ascend/backend/npu_utils.cpp`、`third_party/ascend/backend/driver.py:generate_npu_header_src()`、`third_party/ascend/backend/utils.py:cann_version_compile_args()`。
- **[已验证] torch_npu** 是当前固定的宿主框架/调度集成策略。`get_backend_func()` 固定 `backend_policy="torch_npu"`；`NPUDriver` 用 `torch.npu` 获取 device/property，用 `torch_npu._C._npu_getCurrentRawStream*` 获取原始流；`npu_utils.cpp` 链接 `libtorch_npu`，以 NPU workspace allocator 分配同步锁，并通过 `at_npu::native::OpCommand.SetCustomHandler()` 接入 task queue。证据：`third_party/ascend/backend/utils.py:get_backend_func()`；`backend_register.py`；`driver.py:NPUDriver`；`npu_utils.cpp`。
- **[推断]** torch_npu 主要管理 PyTorch tensor、device、stream、workspace 生命周期和异步调度；真正的设备二进制注册与发射仍落到 ACL/rt API。因此它不是 TTIR lowering/codegen 的实现者。
- **[已验证] GE**：在排除普通英文片段后，当前仓库没有发现 `ge::`、GE header、Graph Engine API 或 ACL graph 构建调用。launcher 中的 `MSPROF_GE_*` 只是 profiling 数据结构常量，不能证明 kernel 经 GE graph 编译/执行。
- **[未验证]** torch_npu 的 `OpCommand` 内部是否进一步经过 GE/task queue；这是外部 torch_npu 实现细节，当前仓库无法确认。
- **[已验证] torchair**：当前快照没有 `torchair` import、构建依赖或调用。
- **[推断]** 因而本仓可确认的主边界是 `Triton-Ascend → BiShengIR/CANN compiler → ACL/rt`，以及 `Triton-Ascend launcher ↔ torch_npu`；不能把 GE 或 torchair 画进已验证主链。

## 9. 问题：与 NVIDIA Triton 相比，哪些复用，在哪里分叉？

**答案：**

- **[已验证] 复用**：Python DSL 与装饰器、AST/semantic/code generator、TTIR、通用 compiler/cache/autotuner/runtime 抽象、MLIR pass 管理器、Triton core dialect、通用分析与工具，以及 backend plugin/entry-point 机制。证据集中在 `python/triton/`、`include/triton/`、`lib/`、`setup.py`。
- **[已验证] 源码仍保留** NVIDIA/AMD backend 与 TTGIR/TritonGPU→LLVM 代码（`third_party/nvidia/`、`third_party/amd/`、`lib/Conversion/TritonGPUToLLVM/`），但 `AscendBackend.supports_target()` 只匹配 `npu`，它们不是 Ascend codegen 的执行路径。
- **[已验证] 核心分叉点在 TTIR 之后**：NVIDIA 路线使用 GPU layout/warp/CTA 语义和 TTGIR，再生成 LLVM IR/PTX/cubin；当前 Ascend 普通路线绕过 TTGIR，转向 structure/unstructure、Linalg/memref、HIVM/HFusion 和 BiShengIR，产物是 npubin。Ascend 纯 SIMT 也不是走仓内 NVIDIA TTGIR，而是把 TTIR 交给厂商编译器的 pure-SIMT 路径。
- **[已验证] 运行时分叉**：NVIDIA 使用 CUDA driver/cubin launch；Ascend 使用 torch_npu 获取 device/stream、用 ACL/rt 注册 npubin 并 launch。证据：`third_party/ascend/backend/driver.py`、`npu_utils.cpp`，对照保留的 `third_party/nvidia/backend/driver.py`。
- **[已验证] 语义分叉**：Ascend 把 `tf32` 映射到 `hf32`，不支持 NVIDIA 语义的 imprecise accumulation；增加 Cube/Vector、UB/L1/L0、pipe/sync、fractral zN、buffer 等概念。证据：`third_party/ascend/backend/__init__.py:_apply_ascend_patch()`；Ascend language extension 与 dialect 定义。
- **[已验证]** 项目通过周期性上游同步而不是一次性复制后完全脱离：`docs/zh/release_note.md` 声明 main 跟踪上游；Git 历史含 Triton 3.6.0 merge 与多个 sync commit。但当前 HEAD 相对 `85400f8` 已有大量 Ascend 文件和后续提交，不能视为无差异镜像。

## 10. 问题：有哪些自定义 Ascend dialect、intrinsic，以及 tile/vector/memory/layout 抽象？

**答案：**

### 12.1 Dialect 与操作

- **[已验证]** 本仓自定义 `ascend` dialect（C++ namespace `mlir::triton::ascend`），操作包括 `annotation`、`mod`、`index_put`、`gather_out_to_ub`、`scatter_ub_to_out`、`index_select_simd`、编译器内部 `indirect_load/store`、`stride_load/store`、`custom`、`flip`、`sort`、`conv1d`、`dot`、`conv2d`。证据：`third_party/ascend/include/Dialect/TritonAscend/IR/TritonAscendDialect.td`、`TritonAscendOps.td`。
- **[已验证]** 另有 `TritonStructured` dialect，核心 `tts.get_structured_state` 用来携带结构化值、offset 和 stride。证据：`third_party/ascend/include/Dialect/TritonStructured/IR/TritonStructuredDialect.td`。
- **[已验证]** HIVM、HFusion、HACC、Scope、Annotation 等 dialect 以 `BiShengIR*Dialect` 库链接进插件和 `triton-mlir-opt`，来源是 AscendNPU-IR/BiShengIR，而不是本仓 `TritonAscendOps.td`。证据：`third_party/ascend/CMakeLists.txt`、`third_party/ascend/bin/CMakeLists.txt`。
- **[未验证]** 上述 BishengIR dialect 的完整 op/type 设计与最终 lowering，因为本地 AscendNPU-IR 子模块未展开。

### 12.2 Language intrinsic/extension

- **[已验证]**`triton.language.extra.cann.extension` 暴露 Cube/Vector core、`PIPE_*`、SIMD/SIMT/MIX mode、iterator type、cross-core `sync_block_set/wait/all`、`fixpipe`、copy、conv/dot、自定义 op 注册、scope/compile hint 等。证据：`third_party/ascend/language/cann/extension/__init__.py`、`core.py`、`semantic.py`、`custom_op.py`、`scope.py`。
- **[已验证]**`cann.libdevice` 用 `@core.extern`/`extern_elementwise` 把 Python 数学函数映射到 `__hmf_*` 等设备符号，并可链接 `third_party/ascend/backend/lib/libdevice.10.bc`。证据：`third_party/ascend/language/cann/libdevice.py`；`NPUOptions.bisheng_options`、`get_libdevice()`。

### 12.3 Tile、vector、memory、layout

- **[已验证] Tile/Vector 抽象**：高层仍使用 Triton block tensor（shape 即单核 tile）和 `tl.arange/load/store/dot`；Ascend 文档建议用 `ncore/XBLOCK/XBLOCK_SUB` 做核间与核内 tiling。lowering 后，Linalg iterator、Cube/Vector core 与 pipe 描述计算和流水；`tile_mix_vector_loop`/`tile_mix_cube_loop` 控制混合 CV 切分。证据：`docs/zh/migration_guide/architecture_difference.md`；`NPUOptions`；`third_party/ascend/language/cann/extension/core.py:CORE, PIPE, IteratorType`。
- **[未验证]** 当前本仓不存在一个单独名为 “Ascend Tile dialect” 的本地定义；不能把文档中的 tiling 策略误写成独立 dialect。部分更底层 tile 语义可能在未展开的 BishengIR 子模块中。
- **[已验证] Memory 抽象**：`python/triton/extension/buffer/language` 定义 `buffer_type/buffer/alloc/subview/to_buffer/to_tensor`，底层用 MLIR memref 与 bufferization；Ascend address space 显式暴露 UB、L1、L0A/L0B/L0C，GM 则由 pointer/global-memory effect 表示。证据：`python/triton/extension/buffer/language/core.py`、`semantic.py`、`python/triton/extension/buffer/src/buffer_ir.cc`；`third_party/ascend/language/cann/extension/core.py:ascend_address_space_group`。
- **[已验证] Layout 抽象**：buffer type 可携带显式 strides，`subview` 派生布局，`to_tensor(target_shape=...)` 可触发 `create_convert_layout`；Ascend `dot` 还用 `fractal_a/b/c` 表达 zN/ND 与 L0C 16×16 分形布局。证据：上述 buffer 文件；`TritonAscendOps.td:DotOp`。
- **[推断]** 与 NVIDIA 用 TTGIR encoding 描述 warp/thread/shared-memory layout 不同，Ascend 普通 SIMD 路线更依赖“tensor tile → Linalg/memref + address space/stride + HIVM core/pipe”的组合表达。

## 11. 问题：设备执行模型和 kernel launch 是怎样的？

**答案：**

- **[已验证]** 用户仍以 Triton 的 3D `grid` API 发射，launcher 将 `gridX*gridY*gridZ` 计算为 `blockNum` 并把三个 grid 值一并打包给 kernel；但导出的 `triton_launch_kernel` 注释明确写“Only 1D parallelization is supported for NPU”。证据：`python/triton/compiler/compiler.py:CompiledKernel.__getitem__()`；`third_party/ascend/backend/driver.py:make_launcher()`。
- **[已验证]** 文档定义的 Ascend 模型是物理核强绑定：Vector/Cube 核各执行 block；与 NVIDIA 上逻辑 CTA 由 SM 自动调度不同，NPU grid/core 数及拓扑更受硬件约束。证据：`docs/zh/migration_guide/architecture_difference.md`。
- **[已验证]** launcher 根据 `mix_mode` 选择 AIVector 或 AICore 物理核数；启用固定 auto-map policy 时把实际 `blockNum` clamp 到物理核数。编译端同步传 `--enable-auto-blockify-loop`，由编译期循环在每个物理 block 内覆盖多个逻辑 block。证据：`third_party/ascend/backend/driver.py:make_launcher()`；`compiler.py` 两个普通 codegen 函数及 `ttir_to_npubin()`；`utils.py:_is_auto_map_parallel_blocks_enabled()`。
- **[已验证]** 对 atomic、inline asm、volatile load、cache modifier 等黑名单模式，编译 metadata 禁用 auto-blockify，避免编译期映射与 runtime cap 不一致。证据：`third_party/ascend/backend/utils.py:AUTO_BLOCKIFY_BLACKLIST_RULES`；`compiler.py:ttir_to_linalg()`。
- **[已验证]** 二进制加载时根据 `mix_mode` 选择 vector-core 或 AI-core ELF magic；launch 时打包可选 FFTS 地址、同步锁、workspace、用户参数、grid、pure-SIMT scratch/device-print 指针，并调用 `aclrtLaunchKernelWithHostArgs` 或旧 `rtKernelLaunch*`。证据：`npu_utils.cpp:cann_register_kernel()`；`driver.py:generate_npu_header_src(), make_launcher()`。
- **[已验证]** stream 来自 torch_npu 当前 raw stream；task queue 默认开启时 launcher 把 launch closure 交给 `at_npu::native::OpCommand.SetCustomHandler()`，关闭时直接 launch 并同步 stream。证据：`NPUDriver.get_current_stream()`；`backend_register.py:async_launch()`；`npu_utils.cpp:triton_async_launch()`。
- **[未验证]** block 与具体芯片物理核/子核的最终一一映射、混合 Cube/Vector task 的固件调度规则，属于 CANN/BiShengIR/硬件内部实现。

## 12. 问题：版本、构建、测试和已知限制是什么？

**答案：**

### 12.1 版本与构建

- **[已验证]** 当前 main 开发线是 Triton **3.6.0**：`version.txt`、`setup.py:TRITON_VERSION`、`third_party/ascend/backend/__init__.py:__version__` 均为 3.6.0；`setup_ascend.py` 还声明安装依赖 `triton==3.6.0`。
- **[已验证]** README 的“最新稳定版”是 Triton-Ascend **3.2.2**，推荐 CANN 9.1.0、TorchNPU 2.7.1.post8；`docs/zh/release_note.md` 同时说明 main=3.6.0，而 release/3.2.2 基于上游 Triton 3.2.0。故 3.2.2 发布兼容矩阵不能直接当作当前 main 的精确矩阵。
- **[已验证]** main 的 LLVM pin 是 `f6ded0be…` 加 Ascend patch；BiShengIR 安装包清单为 `ascendnpu-ir_1.2.0_*_pr_2863.run`。证据：`cmake/llvm-hash.txt`、`third_party/ascend/patch/llvm_patch_f6ded0b.patch`、`cmake/bisheng-version.txt`。
- **[已验证]** Python build-system 要求 setuptools、CMake 3.20～<4、Ninja、pybind11；`setup.py` 当前声明 Python 3.10～3.14。文档中 main 映射写 3.9～3.13，与 setup 声明存在差异，应以实际目标 wheel/CI 验证为准。证据：`pyproject.toml`、`setup.py:MIN_PYTHON/MAX_PYTHON`、`docs/zh/release_note.md`。
- **[已验证]**`setup_ascend.py` 在上游 setup 流程上：加入 ascend backend、替换定制 LLVM 下载、应用 Triton/LLVM patch、追加 `LLVM_MAJOR_VERSION_22_COMPATIBLE`、复制 `triton-mlir-opt`、重写 wheel metadata。AscendNPU-IR 由 `third_party/ascend/CMakeLists.txt:add_subdirectory()` 编入，distributed 支持由 `TRITON_BUILD_TD` 可选开启。

### 12.2 测试

- **[已验证]** Ascend 测试包含 MLIR FileCheck/lit、C++/costmodel tests、NPU pytest、autotune/UB tuner/custom-op/device tests；`third_party/ascend/unittest/CMakeLists.txt` 定义 `check-triton-ascend-lit-tests`。
- **[已验证]** 当前 integration workflow 会在配置的 A3/A5（950）环境构建 wheel，运行 `third_party/ascend/unittest/pytest_ut`、`autotune_ut` 和 MLIR lit；kernel test 因间歇失败被注释禁用。证据：`.github/workflows/ci.yml`、`integration-tests-ascend.yml`。
- **[未验证]** 本报告没有在 NPU 上重跑这些测试，不能把 CI 文件的“计划执行”当作当前 commit 已通过的结果。

### 12.3 明确限制

- **[已验证]** 硬件/环境：用户文档只支持 Linux Ascend NPU，产品为 A2/A3/950；`simt_only` 仅 A5；若无 CANN/BiSheng compiler，`NPUDriver.is_active()` 返回 false。证据：README、`_normalize_compile_mode()`、`NPUDriver.is_active()`。
- **[已验证]** 内存：UB/L1/L0 容量需要手工或 autotune tiling 控制；buffer address-space 大小由用户自行保证，UB overflow 是明确问题类别。证据：`docs/zh/migration_guide/architecture_difference.md`、`docs/zh/debug_guide/ub_overflow.md`、`docs/zh/triton_api_extension/bl/alloc.md`。
- **[已验证]** Alias：外部 pointer 参数默认互不别名；把同一 tensor 作为多个指针实参可能使优化失效或结果异常。证据：`docs/zh/FAQ.md` 第 8 节。
- **[已验证]** 控制流：不同来源 pointer/block pointer 在分支后合并再访存、复杂嵌套控制流中反复更新 pointer 并 store/read-after-write 的支持不完备。证据：`docs/zh/FAQ.md` 第 8 节；`docs/zh/architecture_design_and_core_features.md:TritonToStructured` 局限。
- **[已验证]** API/数据类型仍有局部限制，例如 buffer copy 的 tensor 形式暂不支持、部分 Ascend op 限定 fp16/bf16/fp32、部分向量指令会退化为标量。证据：`docs/zh/triton_api_extension/al/al.copy.md`；`TritonAscendOps.td`；`docs/zh/migration_guide/performance_guidelines.md`。
- **[推断]** 文档里“支持全部 Triton Ops”是版本亮点口径，不能覆盖上述显式局限，也不能证明与 NVIDIA 后端逐 API/逐 dtype/逐 corner case 等价。

## 13. 问题：一句话结论是什么？

**答案：**

**[已验证+推断] Triton-Ascend 的主路线是复用上游 Triton 3.6 的 Python/TTIR/MLIR 基座，在 TTIR 后绕开 NVIDIA TTGIR/PTX 链，转入 Ascend 的结构化 Linalg/memref + HIVM/HFusion/CV 优化与 BiShengIR codegen，并以 torch_npu 管理宿主资源、ACL/rt 直接加载和发射 npubin；最深层设备 IR/ISA 生成因外部编译器及未展开子模块而仍属未验证黑盒。**

---

## Sources / 证据索引

| 编号 | 证据文件或 Git 元数据 | 主要用途 |
|-|-|-|
| S1 | `README_zh.md` | 项目定位、硬件、稳定版、CANN/TorchNPU 安装关系 |
| S2 | `docs/zh/architecture_design_and_core_features.md` | 官方架构、TTIR→Linalg→AscendNPU、pass/option 说明 |
| S3 | `docs/zh/release_note.md` | main/upstream/LLVM 映射、发布与兼容矩阵 |
| S4 | `docs/zh/migration_guide/architecture_difference.md` | GPU/NPU 执行模型、grid、tiling、auto-blockify |
| S5 | `docs/zh/FAQ.md` | pointer alias 与控制流限制 |
| S6 | `python/triton/runtime/jit.py:JITFunction.run()` | JIT 调用入口 |
| S7 | `python/triton/compiler/code_generator.py:CodeGenerator, ast_to_ttir()` | Python AST→TTIR |
| S8 | `python/triton/compiler/compiler.py:ASTSource, compile(), CompiledKernel` | 通用 pipeline、缓存、加载与 launch |
| S9 | `python/triton/backends/__init__.py:_discover_backends()` | backend discovery |
| S10 | `third_party/ascend/backend/compiler.py:AscendBackend, NPUOptions, make_ttir(), ttir_to_linalg(), ttir_to_npubin(), linalg_to_bin_enable_npu_compile_*()` | Ascend 完整编译链与选项 |
| S11 | `third_party/ascend/backend/utils.py` | 工具发现、CANN 版本、编译扩展、auto-blockify policy |
| S12 | `third_party/ascend/backend/driver.py:NPUUtils, NPUDriver, NPULauncher, make_launcher()` | device/stream、launcher、ABI、ACL/rt launch |
| S13 | `third_party/ascend/backend/npu_utils.cpp` | npubin 注册、ACL/rt API、torch_npu workspace/task queue |
| S14 | `third_party/ascend/backend/backend_register.py` | 固定 torch_npu 策略及宿主框架边界 |
| S15 | `third_party/ascend/include/Dialect/TritonAscend/IR/*.td` | Ascend dialect/op/fractal layout 定义 |
| S16 | `third_party/ascend/include/Dialect/TritonStructured/IR/TritonStructuredDialect.td` | 结构化状态 dialect |
| S17 | `third_party/ascend/lib/*` 与各 `CMakeLists.txt` | Ascend lowering/pass 及 MLIR/BiShengIR 链接证据 |
| S18 | `third_party/ascend/language/cann/extension/*.py`、`libdevice.py` | Ascend builtin、core/pipe/sync/custom op、设备函数 |
| S19 | `python/triton/extension/buffer/` | buffer/memref/address-space/stride/layout 抽象 |
| S20 | `third_party/ascend/bin/triton-mlir-opt.cpp` | 标准 MLIR + BishengIR dialect 的 bytecode 工具 |
| S21 | 根 `CMakeLists.txt`、`pyproject.toml`、`setup.py`、`setup_ascend.py` | LLVM/MLIR 直接依赖、打包、上游 patch 与版本 |
| S22 | `cmake/llvm-hash.txt`、`cmake/bisheng-version.txt`、`third_party/ascend/patch/*.patch` | 工具链 pin 和定制补丁 |
| S23 | `.gitmodules` 与 gitlink `aea934a…` | AscendNPU-IR 来源和本地未初始化状态 |
| S24 | `.github/workflows/ci.yml`、`integration-tests-ascend.yml`、`third_party/ascend/unittest/CMakeLists.txt` | CI/test 范围及临时禁用项 |
| S25 | Git commits `b6428718…`、`85400f80…`、`50c50cc7…` | 本次基线、上游 3.6 基准、合并历史 |