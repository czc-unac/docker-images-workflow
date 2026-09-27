# CI 失败分析报告

## 基本信息
- PR: #4641 — 【自动升级】lammps容器镜像升级至1.5版本.
- 失败类型: build-error
- 置信度: 高
- 知识库匹配: 模式02（下载 URL 硬编码版本路径错误 / 软件包版本不存在）
- 新模式标题: (不适用，已匹配已有模式)
- 新模式症状关键词: (不适用)

## 根因分析

### 直接错误
```
#9 [4/8] RUN wget https://github.com/lammps/lammps/archive/refs/tags/stable_${VERSION}.tar.gz     && tar -zxvf stable_${VERSION}.tar.gz     && rm -f stable_${VERSION}.tar.gz
#9 0.066 --2026-09-27 08:06:14--  https://github.com/lammps/lammps/archive/refs/tags/stable_1.5.tar.gz
#9 0.153 HTTP request sent, awaiting response... 302 Found
#9 0.474 Location: https://codeload.github.com/lammps/lammps/tar.gz/refs/tags/stable_1.5 [following]
#9 0.542 HTTP request sent, awaiting response... 404 Not Found
#9 0.763 2026-09-27 08:06:15 ERROR 404: Not Found.
#9 ERROR: process "/bin/sh -c wget ..." did not complete successfully: exit code: 8
```

### 根因定位
- 失败位置: `HPC/lammps/1.5/24.03-lts-sp4/Dockerfile:13`（`RUN wget ... stable_${VERSION}.tar.gz` 步骤）
- 失败原因: 新增 Dockerfile 中 `ARG VERSION=1.5`，构造出的上游下载地址 `https://github.com/lammps/lammps/archive/refs/tags/stable_1.5.tar.gz` 在 LAMMPS 上游仓库中不存在（HTTP 404）。LAMMPS 的发布 tag 采用日期命名（如 `stable_29Aug2024`、`stable_22Jul2025`），并不存在 `stable_1.5` 这一 tag，因此 `wget` 返回 404 并使 Docker 构建（exit code 8）失败。

### 与 PR 变更的关联
直接相关。本次 PR 新增了 `HPC/lammps/1.5/24.03-lts-sp4/Dockerfile`，其中 `ARG VERSION=1.5` 及对应的下载 URL 模板由该 PR 引入。`meta.yml`、`README.md`、`doc/image-info.yml` 中新增的 `1.5-oe2403sp4` 条目同样指向该无效版本。失败步骤正是 Dockerfile 第 13 行的 wget 下载，日志末尾为 `Finished: FAILURE`，属于真实构建失败，非 infra-error。

## 修复方向

### 方向 1（置信度: 高）
使用 LAMMPS 上游实际存在的 tag（日期命名格式，如现有条目的 `29Aug2024`、`22Jul2025`）替换无效的 `1.5`。需确认目标 LAMMPS 版本对应的真实 `stable_<日期>` tag，并同步更新 Dockerfile 中的 `VERSION`、目录路径（`/opt/lammps-stable_${VERSION}`）以及 `meta.yml`、`README.md`、`doc/image-info.yml` 中的版本与链接，保持一致。

### 方向 2（置信度: 中）
若确需“1.5”这类语义化版本，需核实上游是否提供对应的其它下载路径/tag 命名（当前证据显示 `stable_1.5` 不存在），否则应放弃该版本命名，改用上游真实 tag。

## 需要进一步确认的点
- LAMMPS 上游仓库针对该 PR 期望的版本，实际可用的 tag 名称是什么（建议确认 GitHub tags 页面中是否存在非日期命名或 `stable_1.5` 形式的 tag）。
- 该 PR 是否为“自动升级”误将 LAMMPS 官方版本号解析成了错误的 `1.5`（`doc/image-info.yml` 中 `version_prefix: stable_`、`version_scheme: RPM`），需确认版本解析逻辑是否应产出日期型版本号。
- 修复后需保证 `Dockerfile` 中 `WORKDIR /opt/lammps-stable_${VERSION}` 与下载解压出的实际目录名一致，否则后续 `make mpi` 步骤仍会失败。

## 修复验证要求
无（本修复不涉及对第三方源文件的正则 patch，仅需修正 Dockerfile 中的下载 URL / VERSION 变量及配套元数据文件）。
