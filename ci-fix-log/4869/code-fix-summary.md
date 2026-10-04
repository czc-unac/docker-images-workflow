# 修复摘要

## 修复的问题
修正 PyRosetta 3.15 镜像 Dockerfile 中引用的、上游不存在的 git tag（`v3.15-dev62280`），改为上游真实存在的发布 tag `v3.15`，消除 `git clone --branch` 失败导致的镜像构建失败。

## 修改的文件
- `HPC/pyrosetta/3.15/24.03-lts-sp4/Dockerfile`: 将 `ARG VERSION=v3.15-dev62280` 改为 `ARG VERSION=v3.15`。

## 修复逻辑
CI 分析报告（build-error，置信度低、无日志）从 diff 推断出的首要风险点是「`ARG VERSION` 指定的上游分支/tag 不存在」。经向上游仓库实际核实，该推断成立：

- 对 `https://github.com/RosettaCommons/rosetta.git` 执行 `git ls-remote`，仅存在 `refs/tags/v3.15` 与 `refs/tags/v2026.29-dev62280`，**不存在** `refs/tags/v3.15-dev62280`。
- 通过 `raw.githubusercontent.com` 验证：`v3.15` 存在（HTTP 200），`v3.15-dev62280` 返回 HTTP 404。

即自动升级流程把日期式开发版本号（`-dev62280`）错误地拼接到了数字版本 `3.15` 上。`3.15` 属于数字发布线，其对应的上游 tag 就是 `v3.15`（tag commit 日期 2025-09-03），不带 dev 后缀。因此将 `VERSION` 修正为 `v3.15`。

同时确认该 tag 与 Dockerfile 构建命令兼容：
- 已从上游 `v3.15` 获取 `source/src/python/PyRosetta/build.py` 验证，其中包含 Dockerfile 使用的 `--create-package` 与 `--binder-llvm-options` 参数，构建参数有效。
- Dockerfile 中 `--binder-llvm-options` 使用 `multiarch="$(gcc -dumpmachine)"` 动态拼接 `/usr/include/c++/12/${multiarch}`，与已可正常构建的 `2026.29` 镜像保持一致，适配 amd64/arm64，无需改动。

该修复已存在于 fix 分支的 `HPC/pyrosetta/3.15/24.03-lts-sp4/Dockerfile`（当前分支工作区内容即为 `ARG VERSION=v3.15`，与 `pr-head` 相比已包含此修正），其余 README/doc/meta 元数据文件与修正后的路径/tag 一致，无需改动。

## 潜在风险
- 本修复仅改变版本 tag，不涉及构建参数或依赖，风险很低。
- 报告中其余推测项（`/usr/include/c++/12` 硬编码、Python 版本、版权头）经核对与已正常工作的 `2026.29` 镜像完全一致，且 openEuler 24.03-LTS-SP4 的 GCC 主版本为 12，因此未做改动；若后续 CI 日志显示其他根因，需另行分析，避免在无日志情况下盲目改动。
- 由于 3.15 为数字发布线，与日期式的 2026.29 属于不同发布线，两者共存不影响。