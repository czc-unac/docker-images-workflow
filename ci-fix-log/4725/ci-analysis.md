# CI 失败分析报告

## 基本信息
- PR: #4725 — 【自动升级】lammps容器镜像升级至2025.07.22版本.
- 失败类型: build-error
- 置信度: 高
- 知识库匹配: 模式02（下载 URL 版本/路径构造错误，伴随 404）；亦与模式27（上游下载 URL 404）同源
- 新模式标题: 不适用（匹配已有模式）
- 新模式症状关键词: 不适用

## 根因分析

### 直接错误
```
#9 [4/8] RUN wget https://github.com/lammps/lammps/archive/refs/tags/stable_2025.07.22.tar.gz     && tar -zxvf stable_2025.07.22.tar.gz     && rm -f stable_2025.07.22.tar.gz
#9 0.260 HTTP request sent, awaiting response... 302 Found
#9 0.704 Location: https://codeload.github.com/lammps/lammps/tar.gz/refs/tags/stable_2025.07.22 [following]
#9 0.845 HTTP request sent, awaiting response... 404 Not Found
#9 1.523 2026-09-29 08:24:03 ERROR 404: Not Found.
#9 ERROR: process "/bin/sh -c wget https://github.com/lammps/lammps/archive/refs/tags/stable_${VERSION}.tar.gz ..." did not complete successfully: exit code: 8
------
Dockerfile:13
  13 | >>> RUN wget https://github.com/lammps/lammps/archive/refs/tags/stable_${VERSION}.tar.gz \
ERROR: failed to solve: process "..." did not complete successfully: exit code: 8
Build step 'Execute shell' marked build as failure
Finished: FAILURE
```

### 根因定位
- 失败位置: `HPC/lammps/2025.07.22/24.03-lts-sp4/Dockerfile:13`
- 失败原因: 新增 Dockerfile 以 `ARG VERSION=2025.07.22` 拼接上游下载地址 `stable_${VERSION}.tar.gz`，得到 `stable_2025.07.22.tar.gz`，该 Git tag 在 `lammps/lammps` 上游仓库不存在，GitHub 重定向到 `codeload.github.com` 后返回 HTTP 404（wget exit code 8），Docker 构建在该层失败。

### 与 PR 变更的关联
直接相关。本 PR 新增 `HPC/lammps/2025.07.22/24.03-lts-sp4/Dockerfile`，第 13 行的源码下载 URL 使用了 `2025.07.22` 这一版本字符串，而 LAMMPS 上游 stable tag 的命名并非 `YYYY.MM.DD` 形式（仓库中已存在的同类条目为 `22Jul2025`、`29Aug2024`，即 `stable_<日><月><年>` 形式，参见相邻目录与 `image-list.yml`/`meta.yml`）。`image-info.yml` 中声明 `version_scheme: RPM`，说明展示版本被规范化为 `2025.07.22`，但该规范化版本被直接用于构造 Git tag，导致 tag 名与上游不一致并 404。注意 `2025.07.22` 与既有 `22Jul2025` 疑似为同一发布日期（2025 年 7 月 22 日），存在重复升级/版本命名不一致的可能。

## 修复方向

### 方向 1（置信度: 高）
修正 Dockerfile 中下载 URL 的 tag 构造，使其与 LAMMPS 上游实际存在的 Git tag 一致，而非使用 RPM 展示版本 `2025.07.22`。即区分"展示/目录版本（`2025.07.22`）"与"上游 tag 版本（形如 `22Jul2025`）"，用正确的上游 tag 生成 `stable_<tag>.tar.gz` 下载地址。

### 方向 2（可选）
若 `2025.07.22` 确为上游新的发布版本，则需确认其真实 tag 命名规则后再构造 URL；如上游仅提供 Release 制品或 tag 命名变更，可改用其归档地址/镜像源下载。同时确认本次是否为对既有 `22Jul2025` 的重复升级（两者日期相同），避免重复提交。

## 需要进一步确认的点
1. LAMMPS 上游是否存在对应 2025-07-22 的发布，及其确切的 Git tag 名称（`stable_2025.07.22` 已被证伪，需确认是 `stable_22Jul2025` 还是其他命名）。
2. `HPC/lammps/2025.07.22/` 与现有 `HPC/lammps/22Jul2025/` 是否为同一版本；若为同一发布，应确认是否应合并/去重。
3. `image-info.yml` 的 `version_scheme: RPM` 与上游 tag 命名之间的关系（展示版本与下载版本是否应由不同字段/变量承载）。

## 修复验证要求
code-fixer 在提交前，必须访问 LAMMPS 上游仓库（https://github.com/lammps/lammps/tags）确认与 `2025.07.22` 对应的真实 tag 名称，并验证 `https://github.com/lammps/lammps/archive/refs/tags/stable_<实际tag>.tar.gz` 返回 200（而非 404），再据此修正 Dockerfile 中的 URL 构造；不得假设 `stable_2025.07.22` 或其他未验证的 tag 可用。
