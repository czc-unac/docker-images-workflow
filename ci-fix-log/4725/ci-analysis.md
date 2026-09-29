# CI 失败分析报告

## 基本信息
- PR: #4725 — 【自动升级】lammps容器镜像升级至2025.07.22版本.
- 失败类型: build-error
- 置信度: 高（错误现象）/ 中（正确版本号取值）
- 知识库匹配: 模式02（下载 URL 版本路径/版本号不存在导致 404）
- 新模式标题: 无
- 新模式症状关键词: 无

## 根因分析

### 直接错误
```
#9 [4/8] RUN wget https://github.com/lammps/lammps/archive/refs/tags/stable_2025.07.22.tar.gz     && tar -zxvf stable_2025.07.22.tar.gz     && rm -f stable_2025.07.22.tar.gz
#9 0.071 --2026-09-29 08:24:02--  https://github.com/lammps/lammps/archive/refs/tags/stable_2025.07.22.tar.gz
#9 0.260 HTTP request sent, awaiting response... 302 Found
#9 0.704 Location: https://codeload.github.com/lammps/lammps/tar.gz/refs/tags/stable_2025.07.22 [following]
#9 0.845 HTTP request sent, awaiting response... 404 Not Found
#9 1.523 2026-09-29 08:24:03 ERROR 404: Not Found.
#9 ERROR: process "/bin/sh -c wget https://github.com/lammps/lammps/archive/refs/tags/stable_${VERSION}.tar.gz ..." did not complete successfully: exit code: 8
ERROR: failed to solve: process ... exit code: 8
```
（网络连通性正常：成功解析 github.com / codeload.github.com 并完成 302 跳转，说明失败不是网络问题，而是目标路径不存在。）

### 根因定位
- 失败位置: `HPC/lammps/2025.07.22/24.03-lts-sp4/Dockerfile:13-15`（源码下载 RUN 步骤；对应构建编号 `#9 [4/8]`）
- 失败原因: Dockerfile 使用 `ARG VERSION=2025.07.22`，并按模板拼接 URL `.../refs/tags/stable_${VERSION}.tar.gz`，最终请求到 `stable_2025.07.22.tar.gz`；该 tag 在上游 `lammps/lammps` 仓库中不存在，服务器返回 404，wget 以 exit code 8 退出，Docker 构建在该层中止。

### 与 PR 变更的关联
- 这是本 PR **新增文件**引起：diff 新增了 `HPC/lammps/2025.07.22/24.03-lts-sp4/Dockerfile`，其中 `ARG VERSION=2025.07.22` 与下载 URL 模板一起生成了不存在的 tag。
- 关键旁证：同仓库 README/image-info.yml 中已有旧条目 `22Jul2025-oe2403sp4`，其版本目录为 `22Jul2025/24.03-lts-sp4/`，说明 LAMMPS 上游同期 tag 采用的是 **`stable_22Jul2025` 这类“日-月-年”格式，而非 `stable_2025.07.22` 点分日期格式**。2025.07.22 与 22Jul2025 指向的是同一发布日（7 月 22 日），本 PR 很可能是自动升级工具把发布日渲染成了点分格式，导致 tag 模板展开错误。
- 其余改动（README.md / doc/image-info.yml / meta.yml 新增条目）为配套元数据，与本次失败无直接因果；失败仅由 Dockerfile 的下载 URL 触发。

## 修复方向

### 方向 1（置信度: 高）
修正新增 Dockerfile 中 `ARG VERSION` 的取值（或下载 URL 模板），使最终拼接出的 tag 与上游 `lammps/lammps` 实际存在的 tag 一致。依据本仓库既有同日期条目，目标 tag 应为 `stable_22Jul2025` 形式。若确认上游确用点分日期命名，则应为去掉多余的 `stable_` 前缀或按上游真实 tag 调整前缀/版本组合，最终以能 200 下载为准。
- 同步注意：若改动 `VERSION` 值，需一并核对目录名 `2025.07.22/`、README.md、doc/image-info.yml 与 meta.yml 中新增条目的版本标识是否保持一致，避免元数据与 Dockerfile 不匹配。

### 方向 2（置信度: 中）
若上游确实存在以 `2025.07.22` 为发布名的新版本，但 tag 命名规则与 `stable_` 前缀不再匹配（例如 tag 为 `2025.07.22` 或 `stable_20250722`），则应调整 URL 模板而非简单改版本号（例如改用不带 `stable_` 前缀、或去掉点号）。此方向必须先用上游实际 tag 列表验证后再改。

## 需要进一步确认的点
- LAMMPS 上游 `lammps/lammps` 仓库中与 2025.07.22（7 月 22 日）对应的真实 tag 名称：是 `stable_22Jul2025`、`stable_20250722`、`2025.07.22` 还是其他。日志已确证 `stable_2025.07.22` 不存在，但未提供上游 tag 列表，无法仅凭日志断定正确字符串。
- 该版本是否与既有 `22Jul2025` 镜像为同一上游制品（若相同则本 PR 属于重复升级，需确认是否应改为复用/修正既有条目，而不再新增 `2025.07.22` 目录）。
- 若 tag 修正后构建继续推进，需关注后续步骤是否还有次生问题（当前构建在第 4/8 步即中止，后续 `cp examples/melt/in.melt src/`、`make mpi` 均未执行，尚无法验证 `examples/melt/in.melt` 在新版本目录结构中是否仍然存在、`make mpi` 所需依赖是否齐全）。

## 修复验证要求
本修复不涉及对第三方源文件的正则 patch。但因“正确 tag 取值”无法由现有日志唯一确定（置信度为中），code-fixer 在提交前必须执行以下验证，不得直接假设修复方向一定正确：
1. 从上游 `lammps/lammps` 仓库（以 GitHub tags/refs 为准）确认 2025.07.22 对应的真实 tag，并验证 `https://github.com/lammps/lammps/archive/refs/tags/<真实tag>.tar.gz` 返回 200（可先用 `wget --spider` 验证）。
2. 核对本仓库已有 `22Jul2025/24.03-lts-sp4/Dockerfile` 的 `ARG VERSION` 与 URL 模板，保持与既有可用镜像一致的命名约定。
3. 修改后确认新增 Dockerfile 的 `VERSION`、目录名及各元数据文件条目版本标识一致，且下载 URL 可实际访问。
