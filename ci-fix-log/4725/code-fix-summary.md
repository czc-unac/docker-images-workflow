# 修复摘要

## 修复的问题
修正新增 lammps Dockerfile 中错误的源码下载 tag（`stable_2025.07.22` 上游不存在导致 404），改为 LAMMPS 上游真实存在的 `stable_22Jul2025`，使镜像构建可以继续。本次核验确认该修复已存在于 `fix/4725` 分支工作区；当前唯一残留的 aarch64 构建失败属 CI 基础设施问题，无需代码修改。

## 修改的文件
- `HPC/lammps/2025.07.22/24.03-lts-sp4/Dockerfile`: 第 4 行 `ARG VERSION=2025.07.22` → `ARG VERSION=22Jul2025`，使下载 URL 拼接为 `.../refs/tags/stable_22Jul2025.tar.gz`（该改动已在本分支前序修复提交中落盘，本次核验后无需再改）。

> 说明：`README.md` / `doc/image-info.yml` / `meta.yml` 中新增条目的镜像标识为 `2025.07.22-oe2403sp4`，与目录名 `2025.07.22/` 一致，属于镜像 Tag 展示版本，非上游源码 tag，无需改动。

## 修复逻辑
- 根因（对应分析报告「模式02：下载 URL 版本路径不存在导致 404」）：Dockerfile 下载步骤用 `ARG VERSION=2025.07.22` 拼接 `stable_${VERSION}`，请求 `stable_2025.07.22.tar.gz`。网络 302 跳转正常，但目标 tag 在上游 `lammps/lammps` 不存在，codeload 返回 404，wget exit code 8。
- LAMMPS 上游 stable tag 采用「日-月缩写-年」命名。已通过 GitHub Tags API 拉取全量 tag 列表核对：不存在 `stable_2025.07.22`；存在 `stable_22Jul2025` 与 `stable_22Jul2025_update6`。
- 本仓库 `image-info.yml` 的 `version_scheme: RPM` 会把上游 tag 规范化为点分日期展示版本 `2025.07.22`；`version_filter: patch;update` 表示自动升级工具过滤掉 patch/update 版本，只跟踪基础 stable 发布，因此对应 2025-07-22 的基础 tag 为 `22Jul2025`（而非 `_update6`）。修复即区分「展示/目录版本 `2025.07.22`」与「上游源码 tag `22Jul2025`」。
- 提交前验证结果（满足分析报告的验证要求）：
  1. `curl -I -L https://github.com/lammps/lammps/archive/refs/tags/stable_22Jul2025.tar.gz` 返回 **HTTP 200**；`stable_22Jul2025_update6` 亦为 200。
  2. 上游 tag `stable_22Jul2025` 的 tar 包解压目录为 `lammps-stable_22Jul2025`，与 Dockerfile 中 `WORKDIR /opt/lammps-stable_${VERSION}` 一致，后续 `cp examples/melt/in.melt src/` 与 `make mpi` 具备执行条件。
  3. 与既有同日期条目 `HPC/lammps/22Jul2025/24.03-lts-sp4/Dockerfile` 的 `stable_` 前缀命名约定保持一致。
- 本次修复不涉及对第三方源文件的正则 patch。
- 关于当前 CI 状态（已核实 fix PR #4746 的构建结果）：
  - `x86_64 check_build` = **SUCCESS**，证明上述 tag 修复有效；
  - `aarch64 check_build` = FAILED，但失败发生在 `#13 exporting to image` / `exporting layers` 阶段（Docker 已成功编译出 `lmp_mpi`，`#12 DONE`），报错为 `ERROR: failed to receive status: rpc error: code = Unavailable ... closing transport ... graceful_stop` 与 `failed to remove one or more builders`，属于 BuildKit builder 通信中断/被回收的**基础设施问题**，与 Dockerfile 内容无关，不需要代码修改，重跑 CI 即可。

## 潜在风险
- 本 PR 新增的 `2025.07.22` 镜像与仓库既有 `22Jul2025` 镜像源自同一上游 stable 发布（22 Jul 2025），仅镜像 Tag/目录名不同（自动升级工具把上游 tag 规范化为点分日期所致）。这不影响构建通过，但后续人工评审可能按「重复升级」处理。
- 其余元数据文件（README.md / doc/image-info.yml / meta.yml）与构建失败无因果关系，未做改动。