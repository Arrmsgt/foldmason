# Foldmason 龙芯 loongarch64 适配记录

> Foldmason 是**多蛋白结构比对（MSA）工具**（Foldseek 的扩展），用于快速准确地对海量
> 蛋白结构做渐进式多序列比对。本次结论：**复用 Foldseek 的适配方案**，编译一次通过，
> 功能验证正确。分类：结构比对类（同 Foldseek）。

## 一、环境信息

| 项目 | 值 |
|------|-----|
| 架构 | loongarch64（Loongnix Server 23.1，龙芯 3A6000，128 核）|
| 机器 | `10.71.13.47`，容器 `tuyi_test` |
| 分支 | `adapt/sdaa`|
| cmake / gcc | 3.26.3 / 12.3.0 |

## 二、架构分析

### 2.1 Foldmason 是什么

渐进式比对器（progressive aligner），做**多蛋白结构的序列比对（MSA）**。核心命令 `easy-msa`，
内部依赖 Foldseek 的 3Di 结构字母表 + MMseqs2 搜索框架。

### 2.2 结构：自己代码 + 完整 vendored Foldseek

```
Foldmason
├── src/                     ← 自己的代码（~20 文件）
│   ├── commons/FMStructureSmithWaterman.cpp   ← 结构 Smith-Waterman（含 8 处 _mm_）
│   ├── workflow/EasyMSA.cpp                  ← easy-msa 入口
│   └── strucclustutils/refinemsa.cpp 等
└── lib/foldseek/            ← 完整 vendored 的 Foldseek 仓库（2671 文件）
    └── lib/mmseqs, lib/3di, lib/gemmi, lib/prostt5 等
```

CMakeLists 里 `add_subdirectory(lib/foldseek)` + `FOLDSEEK_FRAMEWORK_ONLY=1`（foldseek 作为库编译）。

### 2.3 自定义算子 / SIMD

- 156 个 `.cu` 全在 `lib/foldseek/`（ggml-cuda + libmarv），**`ENABLE_CUDA` 默认 0，不编译**
- 自己的 `FMStructureSmithWaterman.cpp` 的 8 处 `_mm_` 走 SIMDe 中转，升级 SIMDe 后自动用 LSX

## 三、适配改动（2 处，完全复用 Foldseek 方案）

因为 `lib/foldseek` 就是 Foldseek 本体，之前 Foldseek 适配的两个坑在这里原样存在，改动相同：

### 3.1 `lib/foldseek/lib/gemmi/third_party/stb_sprintf.h` — 64 位指针修复

```c
// 64 位架构宏列表末尾加 __loongarch64
defined(__s390x__)  →  defined(__s390x__) || defined(__loongarch64)
```

### 3.2 `lib/foldseek/lib/mmseqs/lib/simde/simde/` — 升级 SIMDe

旧版 SIMDe 不认识 loongarch64（会标量 fallback），升级到支持 LSX/LASX 的版本。
本适配直接从已适配的 Foldseek 仓库（`/data01/tuyilist/foldseek`）复制升级后的 SIMDe，无需重新下载。

## 四、编译

```bash
cd /data01/tuyilist/foldmason
mkdir build && cd build
cmake -DCMAKE_BUILD_TYPE=Release ..
make -j 32
# → build/src/foldmason（21.5MB）
```

## 五、测试结果（功能正确）

```bash
./build/src/foldmason easy-msa d1asha_ d1b0ba_ d1cg5a_ d1cg5b_ example.fasta tmpFolder --report-mode 1
```

输出：
- 氨基酸 MSA（4 结构正确对齐，gap 正确插入）
- Newick 树 `((d1asha_,d1b0ba_),(d1cg5a_,d1cg5b_))` ← d1cg5a/b（同蛋白两链）正确聚一起
- `example.fasta_3di.fa`（3Di 比对）+ `example.fasta.html`（交互式报告）

## 六、踩坑

| # | 坑 | 解决 |
|---|----|------|
| 1 | lib/foldseek 复用 Foldseek 的两个坑（stb_sprintf 64位 + SIMDe 旧版）| 原样复用 Foldseek 的两处改动 |

## 七、归类

与 Foldseek 同类（结构比对/搜索），modelzoo 归 `05_interaction_analysis/` 或结构搜索类目录。
