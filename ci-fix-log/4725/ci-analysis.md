# CI 失败分析报告

## 基本信息
- PR: #4725 — 【自动升级】lammps容器镜像升级至2025.07.22版本.
- 失败类型: `build-error`
- 置信度: 高
- 知识库匹配: 模式02（下载 URL / 软件包版本不存在导致 404）
- 新模式标题: (不适用)
- 新模式症状关键词: (不适用)

## 根因分析

### 直接错误
```
#9 0.071 --2026-09-29 08:24:02--  https://github.com/lammps/lammps/archive/refs/tags/stable_2025.07.22.tar.gz
#9 0.260 HTTP request sent, awaiting response... 302 Found
#9 0.704 Location: https://codeload.github.com/lammps/lammps/tar.gz/refs/tags/stable_2025.07.22 [following]
#9 0.845 HTTP request sent, awaiting response... 404 Not Found
#9 1.523 2026-09-29 08:24:03 ERROR 404: Not Found.
#9 ERROR: process "/bin/sh -c wget https://github.com/lammps/lammps/archive/refs/tags/stable_${VERSION}.tar.gz     && tar -zxvf stable_${VERSION}.tar.gz     && rm -f stable_${VERSION}.tar.gz" did not complete successfully: exit code: 8
Dockerfile:13
> [4/8] RUN wget https://github.com/lammps/lammps/archive/refs/tags/stable_2025.07.22.tar.gz ...
404 Not Found
```

### 根因定位
- 失败位置: `HPC/lammps/2025.07.22/24.03-lts-sp4/Dockerfile:13`（`RUN wget ... stable_${VERSION}.tar.gz` 步骤）
- 失败原因: 该 Dockerfile 以 `ARG VERSION=2025.07.22` 构造下载 URL `stable_2025.07.22.tar.gz`，而 LAMMPS 上游发布 tag 采用 `DDMonYYYY` 格式（同日期的既有镜像条目为 `22Jul2025`，对应 tag `stable_22Jul2025`），不存在 `stable_2025.07.22` 这一 tag，GitHub 返回 404（exit code: 8）。

### 与 PR 变更的关联
- 本次 PR 新增文件 `HPC/lammps/2025.07.22/24.03-lts-sp4/Dockerfile`，其 `ARG VERSION=2025.07.22` 与 URL 模板 `stable_${VERSION}.tar.gz` 的组合直接导致下载 404，属于本次改动新增的失败。
- 值得注意的是：`meta.yml` 中已存在同日期条目 `22Jul2025-oe2403sp4`（path `22Jul2025/24.03-lts-sp4/Dockerfile`），二者指向同一上游发布（2025-07-22）。本次新增的 `2025.07.22` 版本号格式与上游 tag 命名规则不一致，疑为自动升级生成的版本串未转换为 LAMMPS 的 tag 格式。
- README.md / image-info.yml / meta.yml 的文档与索引改动本身不构成失败原因。

## 修复方向

### 方向 1（置信度: 高）
将 URL 中的版本串改为匹配上游 LAMMPS 实际存在的 tag。即 `stable_${VERSION}` 应展开为 `stable_22Jul2025`（`DDMonYYYY` 格式），而非 `stable_2025.07.22`。需确认目标镜像的版本标识与既有 `22Jul2025` 条目之间的关系，避免与已存在的同版本镜像重复。

### 方向 2（可选，置信度: 低）
若上游确实提供了以点分日期命名的 tag/归档路径，则应改用对应的 tag 命名规则或归档下载地址；但当前日志与知识库均未提供该命名存在的证据，需先核实。

## 需要进一步确认的点
- LAMMPS 上游仓库针对 2025-07-22 实际存在的 release tag 名称（`stable_22Jul2025` 或其它），以及是否存在 `stable_2025.07.22` 形式。
- 本次 PR 新增的 `2025.07.22` 与既有 `22Jul2025` 是否为同一上游版本；若相同，需确认是否应复用/合并而非新增重复镜像。
- `meta.yml`、`image-info.yml`、`README.md` 中的版本标识需与 Dockerfile 修正后的 `VERSION` 保持一致。

## 修复验证要求
code-fixer 在提交前，必须核实 LAMMPS 上游仓库（以 Dockerfile 中 `VERSION` 为准，参考 `https://github.com/lammps/lammps/tags`）中目标 release tag 的真实名称，确认 `stable_<正确tag>.tar.gz` 可正常下载（即不再 404）后再提交；不得假设 `2025.07.22` 形式一定可替换为 `22Jul2025` 而未经核实（本仓库中 `22Jul2025` 为强旁证，但仍需确认版本一致性）。
