# 修复摘要

## 修复的问题
修正新增 lammps Dockerfile 中错误的源码 tag（`stable_2025.07.22` 不存在导致 404），改为上游真实存在的 tag，使镜像构建可以继续。

## 修改的文件
- `HPC/lammps/2025.07.22/24.03-lts-sp4/Dockerfile`: 第 4 行 `ARG VERSION=2025.07.22` → `ARG VERSION=22Jul2025`，使下载 URL 拼接为 `.../refs/tags/stable_22Jul2025.tar.gz`。

> 说明：当前 `fix/4725` 分支工作区已包含该改动（前序修复迭代写入），本次核验后确认其为正确的最小修复，未再引入其他改动。README.md / doc/image-info.yml / meta.yml 中新增条目的镜像标识为 `2025.07.22-oe2403sp4`，与目录名 `2025.07.22/` 一致，属于镜像 Tag 标识而非上游源码版本，无需改动。

## 修复逻辑
- 根因：Dockerfile 下载步骤使用 `ARG VERSION=2025.07.22` 并按模板拼接 `stable_${VERSION}`，请求 `stable_2025.07.22.tar.gz` 返回 404（网络 302 跳转正常，属目标路径不存在），与 CI 分析「模式02（下载 URL 版本路径不存在导致 404）」一致。
- 上游 `lammps/lammps` 的 stable 发布 tag 采用「日-月-年」格式，2025 年 7 月 22 日对应的真实 tag 为 `stable_22Jul2025`。本仓库 `image-info.yml` 的 `version_filter: patch;update` 表示自动升级工具过滤掉 patch/update 版本，仅跟踪基础 stable 发布，因此正确取值就是 `22Jul2025`（而非 `_update6`）。
- 验证结果：
  1. 通过 GitHub API 拉取 `lammps/lammps` tag 列表，确认不存在 `stable_2025.07.22`，存在 `stable_22Jul2025` 与 `stable_22Jul2025_update6`。
  2. `https://github.com/lammps/lammps/archive/refs/tags/stable_22Jul2025.tar.gz` 返回 HTTP 200（`stable_22Jul2025_update6` 亦为 200）。
  3. 确认 tag `stable_22Jul2025` 下 `examples/melt/in.melt` 与 `src/MAKE/Makefile.mpi` 均存在，后续 `cp examples/melt/in.melt src/`、`make mpi` 步骤具备执行条件。
  4. 与既有同日期条目 `HPC/lammps/22Jul2025/24.03-lts-sp4/Dockerfile` 的 `stable_` 前缀命名约定保持一致。
- 本修复不涉及对第三方源文件的正则 patch。

## 潜在风险
- 本 PR 新增的 `2025.07.22` 镜像与仓库既有 `22Jul2025` 镜像为同一上游 stable 发布（22 Jul 2025）的重复条目，仅镜像 Tag/目录名不同；这属于自动升级工具的命名产物，不影响本次构建通过，但后续人工评审可能按重复升级处理。
- 其余改动（README.md / doc/image-info.yml / meta.yml）为配套元数据，与构建失败无因果，未做改动。