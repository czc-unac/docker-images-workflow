# CI 失败分析报告

## 基本信息
- PR: #4452 — 【自动升级】cps_public容器镜像升级至5.2.5版本.
- 失败类型: `build-error`
- 置信度: 高
- 知识库匹配: 模式22（Git分支名构造错误，症状完全匹配；根因亦与模式02「软件包版本不存在」同源）
- 新模式标题: (不适用)
- 新模式症状关键词: (不适用)

## 根因分析

### 直接错误
```
#9 [4/5] RUN git clone --depth 1 --branch v5.2.5 https://github.com/RBC-UKQCD/CPS_public.git ...
#9 0.061 Cloning into 'CPS_public'...
#9 0.635 fatal: Remote branch v5.2.5 not found in upstream origin
#9 ERROR: process "/bin/sh -c git clone --depth 1 --branch v${VERSION} https://github.com/RBC-UKQCD/CPS_public.git ..." did not complete successfully: exit code: 128
ERROR: failed to solve: process ... did not complete successfully: exit code: 128
```
日志末尾为 `Finished: FAILURE`，与 PR 失败状态一致，前置一致性检查通过，失败可定位到构建阶段。

### 根因定位
- 失败位置: `HPC/cps_public/5.2.5/24.03-lts-sp4/Dockerfile:13`（`RUN git clone --depth 1 --branch v${VERSION} ...`）
- 失败原因: `ARG VERSION=5.2.5`（第 4 行）展开后构造出的分支名 `v5.2.5` 在上游仓库 `RBC-UKQCD/CPS_public` 中不存在，`git clone --branch v5.2.5` 报 `fatal: Remote branch v5.2.5 not found in upstream origin`，退出码 128。

### 与 PR 变更的关联
该 PR 新增 `HPC/cps_public/5.2.5/24.03-lts-sp4/Dockerfile`，其中第 4 行 `ARG VERSION=5.2.5` 与第 13 行 `--branch v${VERSION}` 组合直接生成被克隆的分支名 `v5.2.5`。这是本 PR 新增内容，失败由本次改动直接触发。注意：仓库中此前已存在同一上游版本的 `HPC/cps_public/5_2_5/24.03-lts-sp4/Dockerfile`（目录名用下划线 `5_2_5`），提示上游对 5.2.5 的实际 tag 命名可能与新增文件采用的 `v5.2.5` 不同。

## 修复方向

### 方向 1（置信度: 高）
核实上游仓库 `RBC-UKQCD/CPS_public` 实际存在的 tag/branch 命名（5.2.5 对应的真实名称），使 Dockerfile 第 13 行克隆的 ref 与该真实名称一致；即修正 `ARG VERSION` 的值或 `git clone --branch` 前缀的构造方式，使展开结果命中上游真实 ref。

### 方向 2（置信度: 中）
若上游确实不存在 5.2.5 对应的任何 tag/branch，则本次「自动升级至 5.2.5」的目标版本无效，应改为上游真实存在且有意义的版本，或确认该自动升级 PR 是否应被放弃。

## 需要进一步确认的点
- `RBC-UKQCD/CPS_public` 上游仓库的 tags 列表中 5.2.5 的真实 ref 名称（是 `5.2.5`、`v5_2_5`、`CPS_public_5_2_5` 还是其他）。
- 参照已合并的 `HPC/cps_public/5_2_5/24.03-lts-sp4/Dockerfile` 中 `git clone` 使用的 ref 形式，确认同版本此前的正确拉取方式。
- 确认 5.2.5 版本在上游是否真实发布，避免重复/命名不一致的自动升级。

## 修复验证要求
code-fixer 在提交前，必须从上游仓库 `RBC-UKQCD/CPS_public` 获取实际的 tag/branch 列表，确认修正后的 ref 能被 `git clone --branch <ref>` 成功解析后，再提交 Dockerfile 修改；不得凭猜测直接替换分支名。
