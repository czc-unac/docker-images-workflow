# 修复摘要

## 修复的问题
修正 LAMMPS 2025.07.22 镜像 Dockerfile 中非法的下载 tag（`stable_2025.07.22` 不存在、HTTP 404），改用上游真实存在的 `stable_22Jul2025`，使 Docker 构建可正常下载源码。

## 修改的文件
- `HPC/lammps/2025.07.22/24.03-lts-sp4/Dockerfile`:
  - 新增 `ARG LAMMPS_TAG=22Jul2025`，并将 `wget` / `tar` / 解压目录 / `WORKDIR` 中拼接 URL 与路径的 `${VERSION}` 全部改为 `${LAMMPS_TAG}`。
  - `ARG VERSION` 恢复为镜像版本标识 `2025.07.22`（与目录名、`meta.yml` key、README/image-info.yml 路径保持一致）。

## 修复逻辑
- 根因：`ARG VERSION=2025.07.22` 拼出上游 tag `stable_2025.07.22`，该 tag 在 `github.com/lammps/lammps` 不存在，wget 经 302 重定向到 codeload 后返回 HTTP 404，构建 exit code 8。LAMMPS 上游 stable tag 采用 `stable_<DDMonYYYY>` 形式，2025-07-22 发布版对应 `stable_22Jul2025`。
- 采用分析报告「方向 2」：将镜像版本标识（`2025.07.22`，与 `meta.yml`/README/image-info.yml 一致）与上游 Git tag 解耦，单独用 `LAMMPS_TAG=22Jul2025` 构造下载 URL。这与仓库既有约定一致（如 `HPC/moose` 中版本号与 `MOOSE_TAG` 分离）。
- 上游验证（提交前已实测）：
  - `https://github.com/lammps/lammps/archive/refs/tags/stable_22Jul2025.tar.gz` → **HTTP 200**；
  - 该 tar 包顶层目录为 `lammps-stable_22Jul2025/`，与 `WORKDIR /opt/lammps-stable_${LAMMPS_TAG}` 完全匹配（已用 `tar -tzf` 实测）；
  - 对照原值 `stable_2025.07.22` → **HTTP 404**，确认原 tag 非法。
- 未采用「方向 1」（将目录/版本标识整体改为 `22Jul2025`）：仓库已存在 `HPC/lammps/22Jul2025` 目录及 `22Jul2025-oe2403sp4` 的 meta/README 条目（内容为 `22Jul2025_update6`），整体改名会产生目录冲突与 `meta.yml` 重复 key；同时本流程只允许修改原始 PR 的 4 个文件、不允许新增文件，无法移动/重命名目录。因此保留 `2025.07.22` 目录与版本标识，仅修正上游下载 tag。

## 潜在风险
- `HPC/lammps/doc/image-info.yml` 的自动升级配置 `version_prefix: stable_` / `version_scheme: RPM` 仍可能把日期类 tag 解析成 `2025.07.22` 这类点分格式，从而持续产出需要手工修正的 PR。本次按最小化原则未修改该配置（分析报告亦标注其适用性需进一步确认），建议后续单独评估。
- 本镜像与既有 `22Jul2025-oe2403sp4`（`22Jul2025_update6`）基于同一上游 `22Jul2025` 发布，内容可能高度重合，属自动升级产生的重复版本，建议人工确认是否保留该新版本目录。