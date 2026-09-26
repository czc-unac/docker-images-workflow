# 修复摘要

## 修复的问题
将 `HPC/lammps/1.5/24.03-lts-sp4/Dockerfile` 中不存在的上游 tag `stable_1.5` 修正为 LAMMPS 真实存在的稳定版 tag `stable_22Jul2025_update6`，使 Docker 构建时 `wget` 不再 404。

## 修改的文件
- `HPC/lammps/1.5/24.03-lts-sp4/Dockerfile`: 将 `ARG VERSION=1.5` 改为 `ARG VERSION=22Jul2025_update6`（仅此一行改动）。

## 修复逻辑
- 根因（分析报告模式02：下载 URL / 软件包版本不存在）：Dockerfile 用 `stable_${VERSION}` 拼接下载 URL，`VERSION=1.5` 得到 `stable_1.5.tar.gz`，上游 `lammps/lammps` 无此 tag，GitHub 302 → codeload 返回 404，`wget` 退出码 8，`docker build` 失败。
- 上游验证：通过 `git ls-remote --tags https://github.com/lammps/lammps` 及 GitHub API 核对，`lammps/lammps` **不存在** `stable_1.5`；实际使用的 tag 形如 `stable_29Aug2024`、`stable_22Jul2025_update6`。`1.5` 实际是 LAMMPS-GUI 的版本号（仓库内存在与之无关的 `lammps-gui-v1.5` tag，其指向 2023-11-21 的提交），属于自动升级工具的误识别，并非 LAMMPS 核心发布版本。
- 选择 `22Jul2025_update6` 的依据：它是上游最新的稳定发布（GitHub `/releases/latest` 返回 `stable_22Jul2025_update6`，发布时间 2026-09-03），且与同目录已有的 `HPC/lammps/22Jul2025/24.03-lts-sp4/Dockerfile` 使用的版本一致，是确定可用的 tag。
- 修复验证：`curl -sL -o /dev/null -w "%{http_code}" https://github.com/lammps/lammps/archive/refs/tags/stable_22Jul2025_update6.tar.gz` 返回 `200`（改动前 `stable_1.5` 返回 `404`）。解压目录名 `lammps-stable_22Jul2025_update6` 与 Dockerfile 中 `WORKDIR /opt/lammps-stable_${VERSION}` 匹配。
- 未修改 README.md / doc/image-info.yml / meta.yml：本次 CI 失败仅为下载 URL 404，这 3 个文件中的 `1.5-oe2403sp4` 是镜像 tag/目录名（与原目录 `HPC/lammps/1.5/` 一致），并非上游 tag；在无法重命名目录的前提下按最小化原则保持其不变，可避免与已有 `22Jul2025-oe2403sp4` 条目产生重复。

## 潜在风险
- 镜像 tag 仍为 `1.5-oe2403sp4`（目录名决定），而实际构建的是 LAMMPS 22Jul2025_update6。这与仓库既有做法一致（例如 `Others/mongoose/7.22`、`7.23` 目录的 Dockerfile 均下载 `7.21`），但会导致该 tag 所声明的版本名与实际上游版本不一致；如需彻底消除，需重命名版本目录并同步元数据（超出本次允许修改的文件范围）。
- 不涉及其他文件与功能，构建流程（解压、`make mpi`）与原有镜像完全一致，无新增风险。