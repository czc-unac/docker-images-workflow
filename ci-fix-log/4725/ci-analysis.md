# CI 失败分析报告

## 基本信息
- PR: #4725 — 【自动升级】lammps容器镜像升级至2025.07.22版本.
- 失败类型: build-error
- 置信度: 高
- 知识库匹配: 模式02（下载 URL 版本路径错误 / 软件包版本不存在），与模式27（GitHub Release URL 404）同源
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
#9 ERROR: process "/bin/sh -c wget https://github.com/lammps/lammps/archive/refs/tags/stable_${VERSION}.tar.gz ..." did not complete successfully: exit code: 8
------
Dockerfile:13
  13 | >>> RUN wget https://github.com/lammps/lammps/archive/refs/tags/stable_${VERSION}.tar.gz \
```

### 根因定位
- 失败位置: `HPC/lammps/2025.07.22/24.03-lts-sp4/Dockerfile:13`
- 失败原因: `ARG VERSION=2025.07.22` 拼接出的上游 tag 为 `stable_2025.07.22`，该 tag 在 `github.com/lammps/lammps` 不存在，wget 经 302 重定向到 codeload 后返回 HTTP 404，Docker 构建失败（exit code 8）。LAMMPS 官方 stable tag 采用 `stable_<DDMonYYYY>` 命名（如既有镜像使用的 `29Aug2024`、`22Jul2025`），并不存在形如 `stable_2025.07.22` 的“点分年月日”tag。

### 与 PR 变更的关联
本 PR 的 diff 新增了 `HPC/lammps/2025.07.22/24.03-lts-sp4/Dockerfile`（`new_file: true`，共 25 行），其中第 13 行的 wget 下载逻辑是本失败的直接触发点；同 PR 的 `README.md`、`doc/image-info.yml`、`meta.yml` 仅登记该新版本。CI 失败由新 Dockerfile 的版本号/tag 构造方式直接引起，与既有 22Jul2025、29Aug2024 镜像无关。日志末尾为 `Finished: FAILURE`，失败发生在本次提供的构建 job 内，日志与状态一致，非下钻架构 job 缺失的问题。

## 修复方向

### 方向 1（置信度: 高）
修正 LAMMPS 版本/tag 的取值与命名，使下载 URL 指向真实存在的上游 tag。对应 2025-07-22 发布的 LAMMPS 版本，上游 stable tag 实为 `stable_22Jul2025`（与仓库中已有的 `22Jul2025` 镜像一致）。应将 Dockerfile 中 `VERSION`/目录版本标识改为上游真实 tag 形式，并同步更新 README.md、doc/image-info.yml、meta.yml 中的版本标识与路径，确保 wget 构造出 `stable_22Jul2025.tar.gz`。

### 方向 2（置信度: 中）
若坚持使用“点分日期”作为镜像目录版本号（`2025.07.22`），则需在 Dockerfile 中显式指定真实的上游 tag（例如单独引入 `LAMMPS_TAG=22Jul2025`）用于拼接 URL，而不要用版本目录名直接拼 tag；同时需核对 `doc/image-info.yml` 的 `version_prefix: stable_` / `version_scheme: RPM` 配置是否能从上游正确解析出该 tag，避免自动升级再次生成非法 tag。

## 需要进一步确认的点
1. 从上游确认 LAMMPS 2025-07-22 发布版对应的准确 Git tag 名称（预期为 `stable_22Jul2025`，需以 GitHub releases/tags 实际值为准）。
2. 确认仓库版本目录命名约定：历史 lammps 镜像使用 `22Jul2025` 形式，本次自动升级生成了 `2025.07.22` 形式，需确认是否应同时统一目录名、meta.yml 的 key 与 README 链接路径。
3. 确认 `doc/image-info.yml` 中 `version_filter: patch;update` 与 `version_scheme: RPM` 是否适用于 LAMMPS 的日期式 tag，若否则自动升级会持续产出不存在的 tag。

## 修复验证要求
本失败属于上游 tag 命名不匹配（下载 404），修复方向不涉及对第三方/上游源文件做正则 patch，无需按“正则 patch”流程验证；但 code-fixer 在提交前必须从上游仓库核实所选 tag 确实存在，即确认 `https://github.com/lammps/lammps/archive/refs/tags/stable_<修正后的tag>.tar.gz` 可正常下载（HTTP 200），再提交版本标识与路径的统一修改。
