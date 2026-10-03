# 修复摘要

## 修复的问题
LAMMPS 自动升级 PR 使用了不存在的上游 tag `stable_2026.09.30`，导致 Dockerfile 中 `wget` 拉取源码包返回 404、镜像构建失败；已将其修正为上游真实存在的稳定 tag `stable_30Sep2026`。

## 修改的文件
- `HPC/lammps/2026.09.30/24.03-lts-sp4/Dockerfile`: 将 `ARG VERSION=2026.09.30` 改为 `ARG VERSION=30Sep2026`（第 4 行），其余步骤不变。

## 修复逻辑
- CI 分析报告因 `ci.logs` / `ci.run_info` 缺失，被归类为 `infra-error（证据不足）`，并在“修复方向-方向 2”及“需要进一步确认的点 #2”中指出：**需核实上游是否存在 `stable_2026.09.30` tag**。
- 本次未依赖日志，而是直接向上游做确定性验证（以 Dockerfile 中 `ARG VERSION=` 对应的 tag 为准）：
  - `https://github.com/lammps/lammps/archive/refs/tags/stable_2026.09.30.tar.gz` → **404（不存在）**
  - `https://github.com/lammps/lammps/archive/refs/tags/stable_30Sep2026.tar.gz` → **200（存在）**
  - GitHub Releases API 最新稳定版为 `stable_30Sep2026`（“Stable release 30 September 2026”，published 2026-09-30T23:21:07Z），与 PR 标题中的“2026.09.30”为同一版本，但上游 tag 采用 `DDMonYYYY` 格式。
- 已确认该 tag 下构建所需文件存在：
  - `examples/melt/in.melt` → 200（`cp examples/melt/in.melt src/` 步骤不会因路径变更失败）
  - `src/Makefile` → 200（`make mpi` 步骤可用）
- 因此根因是自动升级流程把上游 `DDMonYYYY` 日期 tag 规范化成了 `YYYY.MM.DD`，而 Dockerfile 直接以 `stable_${VERSION}` 拼接 tag，导致 404。最小修复即还原为上游真实 tag `30Sep2026`。
- 未改动 `README.md`、`doc/image-info.yml`、`meta.yml`：这三处的版本号/目录路径（`2026.09.30`）是仓库自身的镜像 tag 与目录命名，`meta.yml` 中 `path: 2026.09.30/24.03-lts-sp4/Dockerfile` 指向的目录真实存在；改动它们需要重命名目录并新增路径，超出本次最小修复范围且违反“不新增文件”的约束。构建失败与这些元数据无关。

## 潜在风险
- 镜像对外 tag 仍为 `2026.09.30-oe2403sp4`，但内部打包的 LAMMPS 上游版本 tag 为 `30Sep2026`（同一版本，仅命名格式不同），属命名层面的不一致，不影响构建与运行。
- 未验证 `make mpi` 在 `stable_30Sep2026` 下的实际编译结果（本地无法执行完整 arm64/amd64 Docker 构建），但所需 `src/Makefile` 与示例文件均已确认存在。