# CI 失败分析报告

## 基本信息
- PR: #4869 — 【自动升级】pyrosetta容器镜像升级至3.15版本.
- 失败类型: build-error（推断；日志缺失，无法最终确认）
- 置信度: 低
- 知识库匹配: 模式42（日志缺失无法定位） / 模式19（证据不足）
- 新模式标题: (不适用，命中已有模式)
- 新模式症状关键词: (不适用)

## 前置检查结论（日志与状态一致性）
- 上下文 `ci.logs` = `(not available — analyze based on PR diff only)`，`ci.run_info` = `(not available)`。
- 未提供任何 CI 日志，**不满足"日志显示成功但 PR 失败"的情形**，但也**完全无法从日志中定位最早错误**。
- 因此本报告只能基于 `pr.diff` 进行推断，所有"根因"均为**未经验证的假设**，不得作为确定性结论。

## 根因分析

### 直接错误
无可用日志，无法摘录任何真实报错信息。

```
（ci.logs 为空，无法获取最早 error / traceback / Docker build 失败步骤）
```

### 根因定位
- 失败位置: 未知（无日志）。
- 变更内容: 新增 `HPC/pyrosetta/3.15/24.03-lts-sp4/Dockerfile`（42 行，全新文件），并同步更新 `README.md`、`doc/image-info.yml`、`meta.yml`。
- 失败原因: 无法从现有材料确定。仅能从 diff 提出若干**待验证假设**。

### 与 PR 变更的关联
PR 属于"自动升级"类改动，核心是新增 pyrosetta 3.15 的 Dockerfile。若 CI 失败，最可能发生在新 Dockerfile 的构建阶段（多阶段构建的 builder 阶段）。但没有任何日志佐证，无法排除元数据校验或触发层问题。

## 基于 diff 的假设（均需日志验证，非结论）

1. **上游版本/分支不存在（与模式02、模式19 同类）**
   - Dockerfile 中 `ARG VERSION=v3.15-dev62280`，随后
     `git clone --depth 1 --branch ${VERSION} ... https://github.com/RosettaCommons/rosetta.git`。
   - 若 `v3.15-dev62280` 在 `RosettaCommons/rosetta` 中并非有效 tag/分支，将报 `fatal: Remote branch ... not found in upstream origin` / `couldn't find remote ref`（exit code 128）。自动升级类 PR 使用不存在的上游版本是本仓库高频失败模式（参见模式02/19/42 案例）。

2. **硬编码 C++ 头文件路径与基础镜像 GCC 版本不符（与模式10/35 同类）**
   - `--binder-llvm-options "-isystem /usr/include/c++/12 ..."` 硬编码了 `c++/12`。
   - `openeuler/openeuler:24.03-lts-sp4` 默认 GCC 版本若不为 12，则该 include 路径不存在，clang/LLVM 编译 binder 时可能报头文件缺失或 `cstdlib` 等标准库报错。需确认该基础镜像 `gcc --version` 与 `gcc -dumpmachine` 的实际输出。

3. **构建依赖缺失（与模式10 同类）**
   - 首段 `dnf install` 未显式安装 `sqlite-devel`、`libxml2-devel`、`bzip2-devel`、`boost-devel`、`python3-wheel` 等 PyRosetta/build.py 常见依赖；若上游构建脚本要求而缺失，会在配置/编译阶段报错。需以实际日志确认。

4. **元数据一致性（与模式11 同类，可能性较低）**
   - `meta.yml` 新增 `3.15-oe2403sp4`，`image-info.yml`/`README.md` 同步新增 `[3.15-oe2403sp4]` 条目，三处 tag 名称一致，路径 `3.15/24.03-lts-sp4/Dockerfile` 为两级结构，表面符合规范。若无从日志确认为元数据校验失败，不列入主要方向。

## 修复方向

### 方向 1（置信度: 低）
优先核对上游 `RosettaCommons/rosetta` 是否存在与 `v3.15-dev62280` 完全对应的 tag/分支；自动升级脚本可能拼装了不存在的版本号。若不存在，需改用上游真实可用的 3.15 release 标识。

### 方向 2（置信度: 低）
核对 `openeuler/openeuler:24.03-lts-sp4` 实际 GCC 版本与 `gcc -dumpmachine` 输出，确认 `--binder-llvm-options` 中 `-isystem /usr/include/c++/12/...` 路径真实存在；否则需按实际 GCC 版本调整。

### 方向 3（置信度: 低）
核对 PyRosetta `source/src/python/PyRosetta/build.py` 在 3.15 分支上的依赖要求，确认首段 `dnf install` 是否遗漏必需 `-devel` 包。

> ⚠️ 以上方向均基于 diff 推测，**未获日志验证**。在取得真实失败日志前，不应直接按某一方向提交修改。

## 需要进一步确认的点
- **必须获取失败 job 的真实日志**：
  - 该镜像为 amd64/arm64 双架构（README/image-info 标注），失败很可能发生在某一架构专属的下游构建 job（如 `/job/x86-64/...`、`/job/aarch64/...`）。需要该 job 的完整日志，尤其是最早出现的 `error` / `fatal` / `ERROR: process ... did not complete successfully` 行。
- 确认 `RosettaCommons/rosetta` 仓库中 `v3.15-dev62280`（以及 `v3.15`）是否为有效 ref。
- 确认 `openeuler/openeuler:24.03-lts-sp4` 的 GCC 版本及对应 `/usr/include/c++/<ver>` 路径。
- 确认失败发生在 Docker `builder` 阶段哪一步（git clone / build.py / pip install），还是元数据预检阶段。
- 确认是否有 `ci.run_info` 中的 job 名称、架构、退出码信息。

## 修复验证要求
- 本 PR 的修复**不涉及**正则 patch 第三方/上游源文件，无正则验证要求。
- 但因**置信度为低**，code-fixer 在提交任何修复前**必须**先取得失败 job 日志；在日志缺失情况下**不得**仅凭 diff 假设直接修改 Dockerfile。
