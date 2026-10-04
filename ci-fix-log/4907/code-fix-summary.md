# 修复摘要

## 修复的问题
LAMMPS 自动升级 Dockerfile 使用了上游不存在的版本 tag（`stable_2026.09.30`，`wget` 404），已将其修正为上游真实存在的 tag `stable_30Sep2026`，使镜像构建能够正常下载源码。

## 修改的文件
- `HPC/lammps/2026.09.30/24.03-lts-sp4/Dockerfile`: 将 `ARG VERSION=2026.09.30` 改为 `ARG VERSION=30Sep2026`。该变量同时驱动源码包名、下载 URL（`stable_${VERSION}.tar.gz`）与解压后的工作目录，改一处即可全链路修正。
- `HPC/lammps/README.md`、`HPC/lammps/doc/image-info.yml`、`HPC/lammps/meta.yml`: 经核查，无需修改（见"修复逻辑"）。

## 修复逻辑
对应分析报告"方向 1"，根因为下载 URL 404：

1. **上游 tag 验证（已完成）**：通过 GitHub API 获取 `lammps/lammps` tag 列表并实测下载：
   - `https://github.com/lammps/lammps/archive/refs/tags/stable_30Sep2026.tar.gz` → **HTTP 200**（实际下载 178,883,256 字节，tag 提交日期 2026-09-30）
   - `https://github.com/lammps/lammps/archive/refs/tags/stable_2026.09.30.tar.gz` → **HTTP 404**
   - 对比既有可用 tag `stable_22Jul2025` → HTTP 200。
   结论：LAMMPS 稳定版 tag 采用 `stable_<DDMonYYYY>` 命名，原 `2026.09.30`（自动升级工具按 `version_scheme: RPM` 解析出的日期形式）并不存在，必须改为 `30Sep2026`。

2. **构建链路复核（方向 2 排除）**：对 `stable_30Sep2026` 验证构建依赖同样存在（均 HTTP 200）：
   - `src/MAKE/Makefile.mpi`、`src/Makefile`、`examples/melt/in.melt`，即 `RUN make mpi` 与 `cp examples/melt/in.melt src/` 可正常执行，故排除 MPI/构建依赖问题。

3. **元数据一致性复核**：`meta.yml` / `README.md` / `doc/image-info.yml` 中的镜像 tag 与目录路径 `2026.09.30/24.03-lts-sp4/Dockerfile` 保持一致（与 PR 目录命名相符），且 `meta.yml` 的 path 必须指向真实存在的 Dockerfile。由于约束限制不可新增/重命名文件，目录无法改为 `30Sep2026`，因此保持元数据与目录一致、仅修正 Dockerfile 中的上游下载 tag，是最小且不引入新问题的修复。`HPC/image-list.yml` 已存在 `lammps: lammps` 条目，无需补充。

4. **其他疑点排除**：
   - 模式17（Copyright/SPDX 头）：仓库中 0/429 个 README.md、0/268 个 HPC Dockerfile 含 SPDX 头，且同目录既有 `22Jul2025`、`29Aug2024` 文件均无该头并通过 CI，故该检查不适用于本目录，未添加。
   - YAML 合法性：`meta.yml`、`image-info.yml` 结构与既有版本条目一致，无格式错误。

5. **知识库佐证**：`docs/ci-failure-patterns.md` 模式42 已记录同一路径/同一版本的历史案例 PR #4861（"使用了不存在的上游 tag `stable_2026.09.30`，导致 Dockerfile 构建失败"），与本次修复方向一致。

## 潜在风险
无。修改仅涉及 Dockerfile 中的版本变量，不改变构建步骤、依赖与目录结构；修正后的 tag 已实测可下载。元数据/目录仍沿用 PR 的 `2026.09.30` 命名，与既有结构一致。若后续自动升级工具仍按 `version_scheme: RPM` 生成 ISO 日期形式版本号，同类问题可能再次出现，但这属于升级工具侧问题，超出本 PR 的修复范围。