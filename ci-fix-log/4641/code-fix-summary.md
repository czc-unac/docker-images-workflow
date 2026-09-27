# 修复摘要

## 修复的问题
将 `HPC/lammps/1.5/24.03-lts-sp4/Dockerfile` 中不存在的上游下载 tag `stable_1.5` 修正为 LAMMPS 上游真实存在的 `stable_22Jul2025_update6`，解决 Docker 构建 wget 404（exit code 8）失败。

## 修改的文件
- `HPC/lammps/1.5/24.03-lts-sp4/Dockerfile`: 第 4 行 `ARG VERSION=1.5` 改为 `ARG VERSION=22Jul2025_update6`。

## 修复逻辑
CI 报错根因（匹配知识库模式02：下载 URL 版本不存在）为：LAMMPS 上游 tag 采用日期命名（如 `stable_29Aug2024`、`stable_22Jul2025_update6`），并不存在 `stable_1.5`，因此 `wget https://github.com/lammps/lammps/archive/refs/tags/stable_1.5.tar.gz` 返回 404。

上游验证：
- 通过 GitHub releases/tags API 拉取 `lammps/lammps`，确认全部 `stable_*` tag 均为日期命名，无 `1.5`。
- 对候选 tag 做 HTTP HEAD 实测：`stable_22Jul2025_update6` → 200，`stable_22Jul2025` → 200，`stable_1.5` → 404。
- 仓库中现有的最新 LAMMPS 镜像目录 `HPC/lammps/22Jul2025/24.03-lts-sp4/Dockerfile` 使用的即为 `ARG VERSION=22Jul2025_update6`，本次修改与该既有约定保持一致。

修复只改动 `VERSION` 变量，`WORKDIR /opt/lammps-stable_${VERSION}`、`tar` 解压目录名均随变量同步生效，解压目录 `lammps-stable_22Jul2025_update6` 与 tarball 顶层目录一致，后续 `cp examples/melt/in.melt src/`、`make mpi` 可正常执行。

说明：`meta.yml`、`README.md`、`doc/image-info.yml` 中的 `1.5-oe2403sp4` 条目保持不变。该 `1.5` 为软件市场自动升级流程给出的“应用版本”标签（PR 来源分支 `lammps-软件市场自动升级2026-09-27`），镜像 Tag 由 `meta.yml` 的 key 决定，与 Dockerfile 内引用的上游源码 tag 相互独立；此处理方式与仓库此前的同类自动升级修复（如 `Others/mongoose/7.22` 仅将无效的 `ARG VERSION=7.22` 修正为有效 tag、不改动元数据）一致，符合最小化修复原则。

## 潜在风险
无。仅修正 Dockerfile 内的上游源码版本变量，构建产物与既有 `22Jul2025-oe2403sp4` 镜像内容相同，但镜像 Tag（`1.5-oe2403sp4`）不同，不会覆盖既有镜像。未新增/删除文件，未改动元数据与 CI 配置。