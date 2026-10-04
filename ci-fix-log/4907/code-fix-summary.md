# 修复摘要

## 修复的问题
LAMMPS 自动升级 PR 的 Dockerfile 使用了上游不存在的 Git tag `stable_2026.09.30`，导致 `wget` 下载源码包返回 404、镜像构建失败；已改为上游真实存在的 tag `stable_30Sep2026`。

## 修改的文件
- `HPC/lammps/2026.09.30/24.03-lts-sp4/Dockerfile`: 将 `ARG VERSION=2026.09.30` 改为 `ARG VERSION=30Sep2026`（唯一改动，1 行）。

## 修复逻辑
- 分析报告根因：Dockerfile 第 13 行 `wget .../stable_${VERSION}.tar.gz` 中 `VERSION=2026.09.30` 构造出的 `stable_2026.09.30.tar.gz` 在上游不存在。LAMMPS 稳定版 tag 采用 `stable_<DD><Mon><YYYY>` 形式（如 `stable_22Jul2025`、`stable_29Aug2024`）。
- 上游验证（已实测）：
  - GitHub Tags API `https://api.github.com/repos/lammps/lammps/tags` 返回真实 tag `stable_30Sep2026`（2026.09.30 即 30 Sep 2026）。
  - `https://github.com/lammps/lammps/archive/refs/tags/stable_2026.09.30.tar.gz` → HTTP 404。
  - `https://github.com/lammps/lammps/archive/refs/tags/stable_30Sep2026.tar.gz` → HTTP 200。
  - 实际流式解包 `stable_30Sep2026.tar.gz`，顶层目录为 `lammps-stable_30Sep2026/`，与 Dockerfile 第 17/21 行 `WORKDIR /opt/lammps-stable_${VERSION}` 在 `VERSION=30Sep2026` 时完全匹配。
- 命名约定依据：仓库内既有的成功升级记录（PR #4280，`HPC/lammps/22Jul2025/24.03-lts-sp4/Dockerfile`）中，目录/镜像标签使用 `22Jul2025`，而 Dockerfile 内 `ARG VERSION=22Jul2025_update6`（真实上游 tag）。即目录标签与 Dockerfile 内实际上游 tag 可以不同，构建设计正是通过 `ARG VERSION` 指向真实 tag。
- 因此本次仅需修正 Dockerfile 内的 `VERSION`，目录名 `2026.09.30` 及 `README.md`、`doc/image-info.yml`、`meta.yml` 中的标签/路径引用均保持一致（路径 `2026.09.30/24.03-lts-sp4/Dockerfile` 物理存在），无需改动，符合最小化原则。
- 约束遵从：目录重命名会新增 `pr.changed_files` 之外的文件路径，被 CI 暂存逻辑拒绝，且违反“不新增文件/只改允许文件”约束，故不做重命名，仅改 Dockerfile。

## 潜在风险
无。改动仅修正上游下载 tag，目录结构、元数据标签、构建步骤（`make mpi` 等）均未变化；新 tag `stable_30Sep2026` 的源码目录结构与既有 `stable_22Jul2025*` 一致，`examples/melt/in.melt` 与 `src` 目录布局不变。镜像对外标签仍为 `2026.09.30-oe2403sp4`，与目录/元数据一致。