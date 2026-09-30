# 修复摘要

## 修复的问题
修正 LAMMPS `2025.07.22` 镜像 Dockerfile 中非法的上游下载 tag：`ARG VERSION=2025.07.22` 拼出的 `stable_2025.07.22` 在上游 `github.com/lammps/lammps` 不存在（HTTP 404），改用真实存在的 `stable_22Jul2025`，使 Docker 构建可正常下载源码。

## 修改的文件
- `HPC/lammps/2025.07.22/24.03-lts-sp4/Dockerfile`:
  - 新增 `ARG LAMMPS_TAG=22Jul2025`，并将 `wget` / `tar` / 解压目录 / `WORKDIR` 中拼接 URL 与路径的 `${VERSION}` 全部改为 `${LAMMPS_TAG}`。
  - 保留 `ARG VERSION=2025.07.22` 作为镜像版本标识（与目录名 `2025.07.22/`、`meta.yml` key、`README.md` / `doc/image-info.yml` 条目一致）。
- `HPC/lammps/README.md`、`HPC/lammps/doc/image-info.yml`、`HPC/lammps/meta.yml`：检查无不一致，未改动。

> 说明：本修复在当前 fix 分支上已存在（上一轮迭代已提交，HEAD `64500a8c0`）。本次进入后对当前工作树做了复核与验证，确认修复正确、无需再产生新改动，故未追加代码变更。

## 修复逻辑
- 根因：`ARG VERSION=2025.07.22` 拼出上游 tag `stable_2025.07.22`，该 tag 在 `github.com/lammps/lammps` 不存在；`wget` 经 302 重定向到 codeload 后返回 HTTP 404（exit code: 8），构建在 `Dockerfile` 的 wget 步骤中断（对应分析报告 PID「模式02：下载 URL 版本路径错误 / 软件包版本不存在」）。
- 上游命名：LAMMPS stable tag 采用 `stable_<DDMonYYYY>` 形式（仓库既有 `29Aug2024`、`22Jul2025` 均如此），2025-07-22 发布版对应 `stable_22Jul2025`。分析报告「方向 1」要求以上游实际 tag 为准，本次据此修正。
- 采用「版本标识」与「上游 tag」解耦：`2025.07.22`（点分日期）是自动升级脚本生成的镜像版本标识，不能直接作为上游 tag；用独立的 `LAMMPS_TAG=22Jul2025` 构造下载地址，保留 `VERSION` 与 `meta.yml`/README/image-info.yml 的一致性。此模式与仓库其它镜像约定一致（如 `HPC/petsc` 使用独立的 `ARG TAG`、`HPC/moose` 使用独立的 `MOOSE_TAG`）。
- 未采用「方向 2 整体回退」：本流程只允许修改原始 PR 的 4 个文件、不允许新增/移动文件，且既有 `22Jul2025` 目录与 `2025.07.22` 目录需各自保留，回退会与目录/元数据布局冲突。

### 提交前验证（均已实测）
1. 上游 tag 列表：`git ls-remote --tags https://github.com/lammps/lammps` 确认存在 `refs/tags/stable_22Jul2025`，不存在 `stable_2025.07.22`。
2. 下载可达性：`stable_22Jul2025.tar.gz` → HTTP 200；`stable_2025.07.22.tar.gz` → HTTP 404。
3. 解压目录匹配：`stable_22Jul2025.tar.gz` 顶层目录为 `lammps-stable_22Jul2025/`，与 `WORKDIR /opt/lammps-stable_${LAMMPS_TAG}` 完全一致。
4. 本地完整构建：`docker build` 执行成功，`make mpi` 产出 `lmp_mpi`，镜像导出 DONE。
5. 记录证据：对应 Fix PR（head = 当前 HEAD `64500a8c0`）在 openEuler CI 上 `check_build` x86_64 与 aarch64 均为 SUCCESS。

## 潜在风险
- `HPC/lammps/doc/image-info.yml` 中的自动升级配置 `version_prefix: stable_` / `version_scheme: RPM` 仍可能把日期类 tag 解析为 `2025.07.22` 这类点分格式，从而持续产出需要手工修正 tag 的 PR。本次按最小化原则未改该配置，建议后续单独评估。
- 本镜像与既有 `22Jul2025-oe2403sp4`（内部使用 `22Jul2025_update6`）基于同一上游 `22Jul2025` 发布，内容可能高度重合，属自动升级产生的重复版本，建议人工确认是否保留该新版本目录。
- 本轮未新增代码改动（修复已存在于分支），CI 通过与否取决于该分支既有提交的构建结果。