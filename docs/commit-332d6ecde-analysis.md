# 提交分析：`332d6ecde` — `[AMD] add veclib for llvm opt stage`

**作者：** anq | **日期：** 2026-04-04 | **变更文件：** 2个，新增 11 行

---

## 背景

AMDGCNLIB 是 AMD GPU 的向量数学函数库，提供高性能的向量化数学实现（如三角函数、指数、对数等）。本提交将其接入 Triton 的 LLVM 优化阶段，使 LLVM 自动向量化等优化 Pass 能够识别并调用 AMDGCNLIB 中的例程，替换标量或低效的数学调用。

---

## 变更详情

### 1. `include/triton/Tools/Sys/GetEnv.hpp`

将 `TRITON_USE_AMDGCNLIB` 加入 `CACHE_INVALIDATING_ENV_VARS` 集合。

**作用：** 该集合中的环境变量一旦变更，会使内核编译缓存失效。此处确保开启/关闭 veclib 编译的内核不会复用彼此的缓存条目，避免使用错误的二进制。

### 2. `python/src/llvm.cc`

**改动一：`createTargetMachine()` 函数**

```cpp
if (mlir::triton::tools::getBoolEnv("TRITON_USE_AMDGCNLIB"))
    opt.VecLib = llvm::VectorLibrary::AMDGCNLIB;
```

在构造 LLVM `TargetMachine` 时，将 `TargetOptions::VecLib` 设置为 `AMDGCNLIB`，告知目标机器该向量库可用。

**改动二：`init_triton_llvm()` 中的 `llvm_optimize` 绑定**

```cpp
if (targetMachine &&
    mlir::triton::tools::getBoolEnv("TRITON_USE_AMDGCNLIB")) {
  TargetLibraryInfoImpl TLII(targetMachine->getTargetTriple(),
                             targetMachine->Options.VecLib);
  fam.registerPass([TLII = std::move(TLII)] {
    return TargetLibraryAnalysis(TLII);
  });
}
```

在 LLVM Pass 管道的 `FunctionAnalysisManager` 中注册 `TargetLibraryAnalysis`，使后续优化 Pass（如自动向量化器）能够查询并使用 AMDGCNLIB 函数。

---

## 触发方式

```bash
TRITON_USE_AMDGCNLIB=1 python your_kernel.py
```

---

## 设计评价

| 维度 | 评估 |
|------|------|
| 正确性 | 缓存失效已处理，不会引入静默错误 |
| 侵入性 | 极低，完全 opt-in，不影响现有路径 |
| 作用范围 | 仅对 AMD GPU 有意义（AMDGCN target） |
| 完整性 | 缺少测试覆盖，无文档说明该 knob 的预期效果 |

**主要缺失：** `TRITON_USE_AMDGCNLIB` 未加入 `python/triton/knobs.py`（其他调试 env var 均在此统一管理），也未在 README 或文档中提及，后续维护者较难发现此功能。
