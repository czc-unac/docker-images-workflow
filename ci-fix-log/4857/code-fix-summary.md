# 修复摘要

## 修复的问题
将 rdkit 2026.09.1 镜像 Dockerfile 中引用 conda-forge 上尚不存在的版本 `2026.09.1` 修正为 conda-forge 实际存在的最新版本 `2026.03.6`，并同步更新 README、image-info.yml、meta.yml 中的版本标识。

## 修改的文件
- `HPC/rdkit/2026.09.1/24.03-lts-sp4/Dockerfile`: `ARG VERSION=2026.09.1` → `ARG VERSION=2026.03.6`
- `HPC/rdkit/README.md`: 新增镜像条目由 `2026.09.1-oe2403sp4` / `rdkit 2026.09.1` 改为 `2026.03.6-oe2403sp4` / `rdkit 2026.03.6`（链接路径按仓库既有修复惯例保留原 `2026.09.1/` 目录）
- `HPC/rdkit/doc/image-info.yml`: 同步将 tags 表中的 `2026.09.1` 标识改为 `2026.03.6`
- `HPC/rdkit/meta.yml`: 新增条目 key 由 `2026.09.1-oe2403sp4` 改为 `2026.03.6-oe2403sp4`，path 保持不变

## 修复逻辑
分析报告指出：本次自动升级使用 `conda install -c conda-forge rdkit==2026.09.1`，若 conda-forge 上无该版本，则构建会因 `PackagesNotFoundError` 失败；报告要求先验证目标版本是否真实存在，若不存在则改用上游真实存在的版本号。

验证结果（报告指定的验证途径）：
- `conda search -c conda-forge --override-channels "rdkit=2026.09"` → 无任何结果；
- `conda search -c conda-forge --override-channels "rdkit=2026.03.6"` → 存在（py310~py314 多个 build）；
- `https://api.anaconda.org/package/conda-forge/rdkit` 返回的版本列表中最高为 `2026.03.6`，不含任何 `2026.09.x`；
- PyPI `rdkit` 包同样最高为 `2026.3.6`，无 `2026.09.1`。

即上游 GitHub 虽发布了 `Release_2026_09_1`（2026-10-03），但 conda-forge / PyPI 尚未产出对应制品，属"升级目标在包源上尚不存在"。因此按报告要求改用包源中真实存在的版本 `2026.03.6`，可消除构建时的依赖解析失败。

改动方式遵循本仓库同类自动修复的既有惯例（如 libvirt `2026.77159→12.7.0`、alluxio `2.9.6→2.9.5`、qemu `11.1.2→11.1.1`）：仅在 Dockerfile 及各元数据中替换版本标识，保留原目录路径不变。

## 潜在风险
- `meta.yml` 与 README/image-info.yml 中会存在与既有 `2026.03.6-oe2403sp4` 重复的条目（与仓库历史修复 libvirt 的最终状态一致），若后续 CI 对重复 tag/key 做唯一性校验，可能触发校验告警；如需彻底避免，应在上游 conda-forge 发布 `2026.09.1` 后重新升级。
- 上游 conda-forge 一旦发布 `2026.09.1`，应重新发起升级以取回 2026.09.1 镜像。