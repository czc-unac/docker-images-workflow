# CI 失败分析报告

## 基本信息
- PR: #4559 — 【自动升级】lammps容器镜像升级至1.5版本.
- 失败类型: build-error
- 置信度: 高
- 知识库匹配: 模式02（下载 URL / 软件包版本不存在）
- 新模式标题: -
- 新模式症状关键词: -

## 根因分析

### 直接错误
```
#9 [4/8] RUN wget https://github.com/lammps/lammps/archive/refs/tags/stable_1.5.tar.gz     && tar -zxvf stable_1.5.tar.gz     && rm -f stable_1.5.tar.gz
#9 0.084 --2026-09-26 07:32:45--  https://github.com/lammps/lammps/archive/refs/tags/stable_1.5.tar.gz
#9 0.278 HTTP request sent, awaiting response... 302 Found
#9 0.615 Location: https://codeload.github.com/lammps/lammps/tar.gz/refs/tags/stable_1.5 [following]
#9 0.756 HTTP request sent, awaiting response... 404 Not Found
#9 1.015 2026-09-26 07:32:46 ERROR 404: Not Found.
#9 ERROR: process "/bin/sh -c wget https://github.com/lammps/lammps/archive/refs/tags/stable_${VERSION}.tar.gz ..." did not complete successfully: exit code: 8
Dockerfile:13
  13 | >>> RUN wget https://github.com/lammps/lammps/archive/refs/tags/stable_${VERSION}.tar.gz \
ERROR: failed to solve: ... exit code: 8
Build step 'Execute shell' marked build as failure
Finished: FAILURE
```

### 根因定位
- 失败位置: `HPC/lammps/1.5/24.03-lts-sp4/Dockerfile`:13（`RUN wget .../stable_${VERSION}.tar.gz` 步骤）
- 失败原因: Dockerfile 中 `ARG VERSION=1.5`，下载 URL 拼接为 `stable_1.5.tar.gz`；上游 `lammps/lammps` 仓库不存在 `stable_1.5` 这个 Git tag，GitHub 302 跳转到 codeload 后返回 HTTP 404，wget 退出码 8，Docker build 失败。

### 与 PR 变更的关联
直接由本 PR 引起。PR 新增 `HPC/lammps/1.5/24.03-lts-sp4/Dockerfile`（`ARG VERSION=1.5`），并配套更新 `README.md`、`doc/image-info.yml`、`meta.yml` 新增 `1.5-oe2403sp4` 条目。该版本号 `1.5` 与 LAMMPS 实际的上游发布命名不一致（同目录已有镜像使用的是 `29Aug2024`、`22Jul2025` 这类 `stable_<日期>` 形式的版本，而非 `1.5`），因此构造出的 tag `stable_1.5` 在上游不存在。

## 修复方向

### 方向 1（置信度: 高）
将版本号/下载 tag 改为上游仓库 `lammps/lammps` 中真实存在的发布标识。LAMMPS 的发布 tag 命名惯例为 `stable_<日期>`（如 `stable_29Aug2024`、`stable_22Jul2025`），通常并不存在纯数字版本 `stable_1.5`。需先到上游 GitHub tags 列表确认与“1.5 版本”对应的实际 tag 名称，再同步修正 Dockerfile 中的 `VERSION` 以及 `README.md`、`doc/image-info.yml`、`meta.yml` 中引用的版本标识，保持三者一致。

### 方向 2（可选）
若“1.5”确为上游某个发布线（release branch）而非 tag，则应改用对应的 tag 或 branch 名构造 URL（例如改用 `refs/heads/...` 或正确的 `refs/tags/...`），确保 `wget` 目标地址返回 200。

## 需要进一步确认的点
1. LAMMPS 上游 GitHub 仓库中与“1.5 版本”对应的实际 tag 名称（`stable_` 后接什么），这是修复的关键依据。
2. 若该镜像版本命名需要遵循仓库既有规则（`stable_<日期>`），PR 标题/版本目录 `1.5` 是否应改为实际发布的日期化版本号。
3. `README.md`、`doc/image-info.yml`、`meta.yml` 中新增条目所引用的版本（`1.5-oe2403sp4`）与 Dockerfile `VERSION` 是否需要一并调整，避免元数据与构建内容再次不一致。

## 修复验证要求
本失败不涉及修改第三方/上游源文件的正则 patch 场景，无需额外上游文件正则校验。但 code-fixer 在提交前必须确认：所采用的上游 tag 在 `https://github.com/lammps/lammps` 中真实存在（例如通过 GitHub tags/releases 页面或 `git ls-remote --tags https://github.com/lammps/lammps` 校验），确保 `stable_<tag>.tar.gz` 下载地址返回 200 而非 404。
