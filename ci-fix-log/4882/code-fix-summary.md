# 修复摘要

## 修复的问题
将 openfoam Dockerfile 中不存在的版本 `20260907` 修正为上游真实存在的版本 `2606`，并适配 v2606 的 ThirdParty 归档扩展名。

## 修改的文件
- `HPC/openfoam/20260907/24.03-lts-sp4/Dockerfile`: `ARG VERSION=20260907` → `2606`；`ThirdParty-v${VERSION}.tgz` 的下载 URL、`tar` 解压与 `rm` 清理改为 `.tar.gz`（`OpenFOAM-v${VERSION}.tgz` 保持不变）。
- `HPC/openfoam/README.md`: 表项 tag `20260907-oe2403sp4` → `2606-oe2403sp4`，描述 `openfoam 20260907` → `openfoam 2606`（Dockerfile 链接路径保持不变）。
- `HPC/openfoam/doc/image-info.yml`: 同 README，tag 与描述改为 `2606-oe2403sp4` / `openfoam 2606`。
- `HPC/openfoam/meta.yml`: 条目名 `20260907-oe2403sp4` → `2606-oe2403sp4`（`path` 保持不变，仍指向 `20260907/24.03-lts-sp4/Dockerfile`）。

## 修复逻辑
- 根因对应分析报告的模式02/模式19：自动升级 PR 使用了形似日期（2026-09-07）而非 OpenFOAM 合法的 `YYMM` 版本号，上游不存在 `v20260907`，导致 `ThirdParty-v20260907.tgz` / `OpenFOAM-v20260907.tgz` 下载 404，Docker 构建在下载阶段失败。
- 已从上游 SourceForge（`https://sourceforge.net/projects/openfoam/files/`）核实：最新发布为 `v2606`（2026-07-30），其目录下只有 `OpenFOAM-v2606.tgz` 与 `ThirdParty-v2606.tar.gz`；实测 `ThirdParty-v2606.tgz` 返回 404，`ThirdParty-v2606.tar.gz` 与 `OpenFOAM-v2606.tgz` 返回 200。因此除修正版本号外，必须同步把 ThirdParty 的扩展名由 `.tgz` 改为 `.tar.gz`。
- 受修复流程约束（仅暂存 `pr.changed_files` 中的既有文件，不允许新增文件），物理目录 `HPC/openfoam/20260907/` 无法重命名，故保留既有 path，仅将镜像 tag 与版本描述同步为真实的 `2606`。该做法与本项目针对同一路径的历史修复 PR #4838 完全一致（已比对为逐字节相同）。
- 未涉及正则 patch 上游源文件。

## 潜在风险
- `meta.yml` / README / image-info 中的 tag 为 `2606-oe2403sp4`，而 Dockerfile 物理路径仍位于 `20260907/` 目录，目录名与 tag 不一致（仅命名层面）。因流程禁止重命名目录，此为可接受的最小化折衷；后续如有需要可在允许的变更范围内重命名目录。
- 未修改 `image-info.yml` 中的 `upstream.regex: (?i)v\d{4}\b`，该正则本身可匹配 `v2606`，无需调整。