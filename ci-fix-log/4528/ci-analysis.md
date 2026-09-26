# CI 失败分析报告

## 基本信息
- PR: #4528 — 【自动升级】milc容器镜像升级至6b9b8a0版本.
- 失败类型: dependency-error
- 置信度: 高
- 知识库匹配: 模式22（症状关键词完全匹配；根因机制另见模式28）
- 新模式标题: (不适用)
- 新模式症状关键词: (不适用)

## 根因分析

### 直接错误
```
#9 0.074 Cloning into '/opt/milc_qcd'...
#9 0.725 fatal: Remote branch 6b9b8a0 not found in upstream origin
#9 ERROR: process "/bin/sh -c git clone --depth 1 --branch 6b9b8a0 https://github.com/milc-qcd/milc_qcd.git ${MILC_HOME} && ..." did not complete successfully: exit code: 128
...
Dockerfile:18
  18 | >>> RUN git clone --depth 1 --branch 6b9b8a0 https://github.com/milc-qcd/milc_qcd.git ${MILC_HOME} && \
ERROR: failed to solve: ... exit code: 128
Build step 'Execute shell' marked build as failure
Finished: FAILURE
```
（日志末尾为 `Finished: FAILURE`，属真实失败，非编排层/下游架构 job 缺失场景。）

### 根因定位
- 失败位置: `HPC/milc/6b9b8a0/24.03-lts-sp4/Dockerfile:18`
- 失败原因: 新增 Dockerfile 中 `git clone --depth 1 --branch 6b9b8a0` 把版本号 `6b9b8a0` 直接当作远程 ref 传入，但该 ref 在上游仓库 `milc-qcd/milc_qcd` 中不存在（`6b9b8a0` 形如 commit 短哈希/版本号，git 的 `--branch` 只能接受分支名或标签名，不能接受 commit hash），克隆立即失败并返回 exit code 128。

### 与 PR 变更的关联
直接相关。本 PR 为自动升级，新增的 `HPC/milc/6b9b8a0/24.03-lts-sp4/Dockerfile` 第 18 行硬编码 `--branch 6b9b8a0`；`meta.yml` 同步新增 `6b9b8a0-oe2403sp4` 条目，将该版本纳入 CI 构建。构建在同一 `RUN` 层执行 `git clone && cmake`，clone 阶段即失败，后续 cmake 未执行。

## 修复方向

### 方向 1（置信度: 高）
确认 `milc-qcd/milc_qcd` 上游实际存在的 ref：若 `6b9b8a0` 是 commit hash，则不应作为 `git clone --branch` 参数。应改为克隆上游真实存在的分支/标签，或在获取完整历史（去掉 `--depth 1`，或使用 `git fetch origin <sha>`）后再 checkout 该 commit。注意 `--depth 1` 浅克隆无法直接 checkout 任意 commit（参见模式18）。

### 方向 2（置信度: 中）
若自动升级脚本本意是跟踪某分支/标签但错误将 commit 短哈希填入 `VERSION`，应修正生成逻辑，使 Dockerfile 使用上游发布的分支或 tag 名（如 `develop` 等）。

## 需要进一步确认的点
- `6b9b8a0` 在 https://github.com/milc-qcd/milc_qcd 中究竟是 commit hash、tag 还是 branch（日志仅显示 "Remote branch ... not found"，无法区分）。
- 若确为 commit hash，需确认其完整 40 位 SHA，以及上游是否支持按该 commit 拉取。
- 需确认自动升级工具生成 Dockerfile 时选用 ref 的规则，避免后续同类版本继续引用不存在的 ref。

## 修复验证要求
本修复不涉及正则 patch 第三方源文件，无需上游文件正则验证；但 Code Fixer 在提交前必须确认所引用的 ref 在上游 `milc-qcd/milc_qcd` 中实际存在（分支/标签名或完整 commit SHA），且与该版本目录 `6b9b8a0` 所声明的版本一致。
