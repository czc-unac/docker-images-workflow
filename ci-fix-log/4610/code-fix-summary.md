# 修复摘要

## 修复的问题
修复 milc 镜像构建时因把 commit 短哈希 `6b9b8a0` 当作分支名传给 `git clone --branch` 而导致的 `fatal: Remote branch 6b9b8a0 not found in upstream origin`（exit 128）构建失败。

## 修改的文件
- `HPC/milc/6b9b8a0/24.03-lts-sp4/Dockerfile`: 将第 18 行的 `git clone --depth 1 --branch 6b9b8a0 ...` 改为普通 `git clone ...`，并在 `cd ${MILC_HOME}` 后新增 `git checkout ${VERSION}`，按 commit 检出目标版本。

## 修复逻辑
- 根因：`6b9b8a0` 是上游 `milc-qcd/milc_qcd` 的 commit 短哈希，不是分支名或 tag。`git clone --branch` / `git fetch` 只能解析分支或 tag，无法解析短哈希，因此失败（与知识库模式22/28一致）。
- 已从上游核实：`https://api.github.com/repos/milc-qcd/milc_qcd/commits/6b9b8a0` 返回完整 SHA `6b9b8a06eec5746187bbfd197eac2629ab8d8e72`，作者 Carleton DeTar，日期 2026-08-09；`git ls-remote` 显示该 commit 正是 `refs/heads/develop` 的当前 tip，且仓库当前无任何 tag（tags API 为空）。因此不存在等价的 official tag 可直接用于 `--branch`。
- 修复采用本仓库已有的 commit 版本构建约定（参见 `Others/multiwfn/cb37c53/24.03-lts-sp4/Dockerfile`、`Others/monolith/135c491/24.03-lts-sp3/Dockerfile`：完整 clone 后 `git checkout ${VERSION}`）：先完整克隆仓库，再用已有的 `ARG VERSION=6b9b8a0` 执行 `git checkout`。完整克隆包含该 commit，短哈希检出可正常工作；同时保留了变量化写法，便于后续自动升级复用。
- 验证（已实际执行）：
  1. 复现失败：`git clone --depth 1 --branch 6b9b8a0 https://github.com/milc-qcd/milc_qcd.git` → `fatal: Remote branch 6b9b8a0 not found in upstream origin`。
  2. 验证修复：在临时目录按修改后的命令执行完整 `git clone` + `git checkout 6b9b8a0`，成功检出，`git log` 显示 `6b9b8a06 Use FORSOMPARITY_OMP instead of FOREVENSITES_OMP`。
- `README.md`、`doc/image-info.yml`、`meta.yml` 的版本条目与目录/`ARG VERSION` 一致（`6b9b8a0-oe2403sp4`），且与本构建失败无因果关系，故未改动。

## 潜在风险
- 改为完整 `git clone`（不再 `--depth 1`）会比浅克隆多下载一些历史与工作区（实测约 346MB），构建时间/网络占用略有增加，但不影响功能正确性。仓库未启用 submodule 递归克隆，与原命令保持一致。
- `git checkout ${VERSION}` 依赖 `VERSION` 为有效 commit/分支/tag；若后续自动升级写入其他无效版本串，仍会失败，但这是版本来源问题，不属于本次最小修复范围。