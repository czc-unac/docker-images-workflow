# 修复摘要

## 修复的问题
自动升级 PR 将 OpenFOAM 版本号误写为日期串 `20260907`（上游不存在 `v20260907` 目录），导致 Docker 构建时源码包 wget 返回 404；本次将其修正为上游真实存在的版本 `2606`，并同步修正下载文件名。

## 修改的文件
- `HPC/openfoam/20260907/24.03-lts-sp4/Dockerfile`: `ARG VERSION=20260907` → `ARG VERSION=2606`；ThirdParty 源码包扩展名由 `.tgz` 修正为上游实际发布的 `.tar.gz`（wget/tar/rm 三处同步修改）。
- `HPC/openfoam/meta.yml`: 标签 `20260907-oe2403sp4` → `2606-oe2403sp4`（path 仍指向真实文件目录 `20260907/24.03-lts-sp4/Dockerfile`）。
- `HPC/openfoam/README.md`: 版本标签行 `[20260907-oe2403sp4]` → `[2606-oe2403sp4]`，展示版本改为 `openfoam 2606`。
- `HPC/openfoam/doc/image-info.yml`: 版本标签行同步由 `20260907-oe2403sp4` 修正为 `2606-oe2403sp4`，展示版本改为 `openfoam 2606`。

## 修复逻辑
分析报告根因：`ARG VERSION=20260907` 使下载 URL 展开为 `.../files/v20260907/OpenFOAM-v20260907.tgz`，该路径在上游不存在，导致 404（dependency-error，知识库模式 02）。OpenFOAM 版本号为 4 位 `YYMM` 形式，`20260907`（8 位日期）不符合格式。

已将版本号修正为当前上游真实发布的最新版本 `2606`，并确认扩展名差异。验证结果：
- 通过 WebFetch 获取上游目录：`https://sourceforge.net/projects/openfoam/files/`，列表中 `v2606` 存在（更新时间 2026-07-30，为最新版本）。
- 通过 WebFetch 获取 `https://sourceforge.net/projects/openfoam/files/v2606/`，确认文件清单为 `OpenFOAM-v2606.tgz` 与 `ThirdParty-v2606.tar.gz`（注意 ThirdParty 为 `.tar.gz`，而非旧版本沿用的 `.tgz`）。
- 通过 curl 实际请求上游 URL：`.../v2606/OpenFOAM-v2606.tgz` 返回 HTTP 200，`.../v2606/ThirdParty-v2606.tar.gz` 返回 HTTP 200，确认修复后不再 404。

因此除修正版本号外，必须将 ThirdParty 文件名由 `.tgz` 改为 `.tar.gz`，否则即使版本号正确仍会 404。

## 潜在风险
无。改动仅涉及版本号与下载文件名，作用范围限于 `HPC/openfoam` 本次新增条目。
说明：上游 v2606 真实目录名为 `2606`，而本 PR 新增文件目录名仍为 `20260907`；受“只允许修改 `pr.changed_files`、不允许新增文件/目录”的约束，未对目录重命名。`meta.yml`、`README.md`、`doc/image-info.yml` 中引用该文件的路径均已指向实际的 `20260907/24.03-lts-sp4/Dockerfile`，内部一致，不影响构建与链接有效性。