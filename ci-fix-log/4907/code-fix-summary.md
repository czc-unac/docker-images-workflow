# 修复摘要

## 修复的问题
本 PR 的真实根因（Dockerfile 使用上游不存在的 tag `stable_2026.09.30`）已在上一轮修复中修正为上游真实存在的 tag `stable_30Sep2026`；本轮核查确认当前代码正确，**无需新增代码修改**。当前 fix 分支的 x86_64 CI 失败属于基础设施错误（拉取 `build.sh` 时 HTTP 429 限流），与代码无关。

## 修改的文件
- `HPC/lammps/2026.09.30/24.03-lts-sp4/Dockerfile`: 无新增修改。当前第 4 行已是 `ARG VERSION=30Sep2026`（由上一轮修复提交 `dcb359ef2` 完成）。
- `HPC/lammps/README.md`、`HPC/lammps/doc/image-info.yml`、`HPC/lammps/meta.yml`: 无需修改，四处版本条目与目录/镜像 tag `2026.09.30-oe2403sp4` 保持一致。

## 修复逻辑
本报告中的 CI 分析为“日志缺失、置信度低”的旧分析，且引用的 `ARG VERSION=2026.09.30` 在当前分支已不存在。为保证结论确定性，本轮直接做了两级证据核查：

1. **上游 tag 确定性验证（已完成）**
   - `https://github.com/lammps/lammps/archive/refs/tags/stable_2026.09.30.tar.gz` → **HTTP 404**（不存在）
   - `https://github.com/lammps/lammps/archive/refs/tags/stable_30Sep2026.tar.gz` → **HTTP 200**，下载得到有效 gzip 包（178,883,256 字节）
   - 解压顶层目录为 `lammps-stable_30Sep2026`，与 `WORKDIR /opt/lammps-stable_${VERSION}` 一致；`examples/melt/in.melt`、`src/Makefile` 均存在。
   - 结论：LAMMPS stable tag 采用 `stable_<DDMonYYYY>`，`2026.09.30` 是自动升级工具按 `version_scheme: RPM` 规范化出的日期形式，并非上游真实 tag。当前 `ARG VERSION=30Sep2026` 已指向真实 tag，Dockerfile 修复正确。

2. **当前 fix 分支 CI 失败定性：infra-error（已从真实日志确认）**
   - 通过 GitCode API 定位到修复 PR #4918（head `fix/4907`）的门禁结果：**x86_64 FAILED / aarch64 SUCCESS**。
   - 从门禁日志服务拉取 x86_64 失败构建 #5037 的真实控制台日志，失败原因是：
     ```
     curl: (22) The requested URL returned error: 429
     chmod: cannot access 'build.sh': No such file or directory
     ./build.sh: No such file or directory
     Build step 'Execute shell' marked build as failure
     ```
   - 该构建 `durationMs=1277`（约 1.3 秒），构建根本未开始；对照成功的构建 #4985 `durationMs≈6,643,487`（约 110 分钟）。这是 Jenkins 拉取构建脚本被限流（HTTP 429）的临时性基础设施故障，与仓库代码无关。

3. **同类先例佐证**
   - 完全相同的补丁（Dockerfile blob `e211599c`，四个文件 blob 完全一致）在 PR #4872（fix #4861）的 x86_64 与 aarch64 均 **ci_successful**。
   - 说明 `stable_30Sep2026` 这一修复本身可通过门禁，PR #4918 的 x86_64 失败为可重试的偶发限流，不是代码问题。

综上：根因修复已到位且经上游实测 + 同补丁历史通过记录双重验证；本次 CI 失败为 infra-error，按约束**不强行改代码**，重跑门禁即可。

## 潜在风险
无。当前分支未做任何新增改动；对外的镜像 tag 仍为 `2026.09.30-oe2403sp4`，内部打包的上游版本 tag 为 `30Sep2026`（同一版本、命名格式不同），与既有 `22Jul2025` 目录沿用仓库自身命名、Dockerfile 使用精确上游 tag 的约定一致，不影响构建与运行。