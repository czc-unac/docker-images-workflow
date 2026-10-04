# 修复摘要

## 修复的问题
修正 `HPC/pyrosetta/3.15/24.03-lts-sp4/Dockerfile` 中不存在的上游 git 引用，避免 `git clone --branch` 因 `Remote branch not found` 而失败。

## 修改的文件
- `HPC/pyrosetta/3.15/24.03-lts-sp4/Dockerfile`: 将 `ARG VERSION=v3.15-dev62280` 改为 `ARG VERSION=v3.15`。

## 修复逻辑
分析报告虽因 `ci.logs` 缺失而归类为 `infra-error`，但明确将本 PR diff 中最高风险的排查点列为：
`ARG VERSION=v3.15-dev62280` 随后用于 `git clone --branch ${VERSION} https://github.com/RosettaCommons/rosetta.git`，
若上游不存在该 ref 则构建必然失败。报告同时要求“若确认上游 ref 不存在，则修正 `ARG VERSION`”。

修复前已通过 `git ls-remote` 直接向实上游仓库核验（离线日志缺失，故改用比日志更直接的实证）：

- `refs/tags/v2026.29-dev62280` 存在（现有 2026.29 镜像引用的 ref）；
- 3.15 系列在上游仅有 `refs/tags/v3.15`，**不存在** `v3.15-dev62280`（`git ls-remote | grep 3.15` 仅返回 `refs/tags/v3.15`，全部 388 个 tag 中仅 `v2026.29-dev62280` 带 `dev62280` 后缀）。

因此本 PR 新增的 3.15 Dockerfile 在 `git clone --branch v3.15-dev62280` 处必然报 `Remote branch v3.15-dev62280 not found`。
同时通过 WebFetch 核验 `https://raw.githubusercontent.com/RosettaCommons/rosetta/v3.15/source/src/python/PyRosetta/build.py` 可正常获取，
确认 `v3.15` tag 中存在 Dockerfile 第 20-24 行调用的 `build.py`，替换后构建路径有效。

改动仅一行，未触碰 `pr.changed_files` 之外的文件，未新增文件。

## 潜在风险
- `v3.15` 为 Rosetta 源码 tag，与原有 `dev62280` 构建号语义不同；若 CI 期望的是某个带 `dev` 后缀的 3.15 构建 ref，需上游实际发布后再调整。当前以上游实际存在的 ref 为准。
- 其余报告提及的低置信风险（GCC 12 头文件路径、license 头、文件末尾换行）均无日志证实，且不在本次最小化修复范围内，未做改动。