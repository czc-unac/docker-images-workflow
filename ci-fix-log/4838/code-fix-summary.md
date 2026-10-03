# 修复摘要

## 修复的问题
将 openfoam Dockerfile 中不存在的版本 `20260907` 修正为上游真实存在的版本 `2606`，并同步修正 `ThirdParty` 压缩包扩展名（`.tgz` → `.tar.gz`），修复 wget 404 导致的镜像构建失败。

## 修改的文件
- `HPC/openfoam/20260907/24.03-lts-sp4/Dockerfile`: `ARG VERSION=20260907` → `ARG VERSION=2606`；`ThirdParty-v${VERSION}.tgz` 的下载、解压、清理三处均改为 `ThirdParty-v${VERSION}.tar.gz`（`OpenFOAM-v${VERSION}.tgz` 保持 `.tgz`）。
- `HPC/openfoam/README.md`: 新增条目的 Tag/版本由 `20260907-oe2403sp4` / `openfoam 20260907` 修正为 `2606-oe2403sp4` / `openfoam 2606`（链接路径仍指向 `20260907/24.03-lts-sp4/Dockerfile`）。
- `HPC/openfoam/doc/image-info.yml`: 同上，Tag/版本修正为 `2606`。
- `HPC/openfoam/meta.yml`: 条目 key 由 `20260907-oe2403sp4` 修正为 `2606-oe2403sp4`，path 保持不变。

## 修复逻辑
1. **根因确认（已获取真实 CI 日志）**：分析报告称日志缺失、无法定位根因，但通过 openEuler 门禁评论中的构建 URL 拉取到 x86_64（job `4952`）与 aarch64（job `5048`）的完整日志，两者均在 `[4/5] RUN wget .../v20260907/ThirdParty-v20260907.tgz` 步骤失败：
   ```
   --2026-10-03 06:18:17--  https://sourceforge.net/projects/openfoam/files/v20260907/ThirdParty-v20260907.tgz
   HTTP request sent, awaiting response... 404 Not Found
   ERROR 404: Not Found.
   process ... did not complete successfully: exit code: 8
   ```
2. **上游核验**：从 SourceForge 获取 OpenFOAM 实际发布列表，`v20260907` 目录不存在（404），OpenFOAM 使用 `vYYMM` 版本命名（`v2412`/`v2506`/`v2512`/`v2606`），当前最新稳定版为 `v2606`。逐个验证下载 URL：
   - `v2606/OpenFOAM-v2606.tgz` → HTTP 200
   - `v2606/ThirdParty-v2606.tar.gz` → HTTP 200（`ThirdParty-v2606.tgz` 为 404）
   据此选择 `2606`，并因其 ThirdParty 制品为 `.tar.gz` 而同步修正扩展名。
3. **遵循仓库既有修复约定**：与本仓库同类修复（如 `!2671 fix: redis 5.4.1`）一致——当自动升级带入的版本不存在时，将 Dockerfile 的 `ARG VERSION` 改为上游真实版本并更新 README/image-info/meta 中的版本与 Tag，同时保留原目录路径不变（受“禁止新增/重命名文件”约束，`20260907/` 目录名保持不变）。

## 潜在风险
- 仅将版本修正为 `v2606`，`Allwmake` 全量编译在 24.03-LTS-SP4 上是否完全通过无法在本地验证；现有 `yum` 依赖列表沿用自 `2506`，若 `2606` 引入新的编译依赖，后续 CI 可能仍失败并需追加依赖。
- 由于目录名 `20260907` 无法在本次修复中重命名，Tag/目录存在 `2606-oe2403sp4` 与路径 `20260907/...` 的不一致，此为本仓库同类修复（redis `5.4.1` 目录承载 `8.6.4` 版本）的既有约定，非本次引入。