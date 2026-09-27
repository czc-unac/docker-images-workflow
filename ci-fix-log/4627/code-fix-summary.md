# 修复摘要

## 修复的问题
撤销无效的 ranger 2.9.0 升级：`2.9.0` 并非本镜像上游（ranger 文件管理器 `ranger/ranger`，最新版本为 v1.9.4）存在的版本，导致 `git clone -b v2.9.0` 报 `Remote branch v2.9.0 not found`（exit 128），同时修正被错误指向 `apache/ranger` 的上游元数据。

## 修改的文件
- `Bigdata/ranger/2.9.0/24.03-lts-sp4/Dockerfile`: 删除。该文件为本次自动升级新增，克隆 `ranger/ranger` 的 `v2.9.0`，该 tag 在上游不存在。
- `Bigdata/ranger/meta.yml`: 移除错误的 `2.9.0-oe2403sp4` 条目，仅保留有效的 `1.9.4-*` 条目。
- `Bigdata/ranger/README.md`: 移除 tags 表中 `2.9.0-oe2403sp4` 行，恢复文件末尾换行。
- `Bigdata/ranger/doc/image-info.yml`: 移除 `2.9.0-oe2403sp4` 行；将 `homepage`/`upstream` 从 `apache/ranger`、`version_prefix: release-`、`version_scheme: RPM` 修正回 `ranger/ranger`、`version_prefix: v`、`version_scheme: semantic`，并恢复文件末尾换行。

## 修复逻辑
根因是**版本号在上游不存在 + 上游元数据被污染**：

1. 本镜像构建的对象是 **ranger 文件管理器**（`ENTRYPOINT ["ranger"]`、README 描述为 "console file manager"、`homepage` 为 ranger.fm）。已通过上游仓库核实 `https://api.github.com/repos/ranger/ranger/tags`，其最新 tag 为 `v1.9.4`，**不存在 `v2.9.0`**，因此 Dockerfile 的 `git clone -b v2.9.0` 必然失败（CI 失败根因）。
2. `image-info.yml` 早前被错误标注为 `apache/ranger`（Apache Ranger 才是 2.x 版本系列的 Java 大数据项目，其 tag 为 `release-ranger-2.9.0`，需要 Maven/Java 构建，与本镜像的 `pip install`/`python setup.py` 构建方式完全不同）。自动升级逻辑据此误取了 `2.9.0` 套用到 ranger 文件管理器上。
3. 由于 `ranger/ranger` 不存在任何高于 1.9.4 的版本，本次"升级"本身无效，正确修复是**回退该升级**并**把上游元数据修正回 `ranger/ranger`**，与主分支既有状态保持一致。

验证：
- 已核实 `ranger/ranger` tags 列表（v1.9.4 为最新，无 v2.9.0）；`apache/ranger` 虽有 `release-ranger-2.9.0`，但其为不同的 Java 项目，非本镜像构建目标。
- 修复后的 3 个文件内容与主分支（及既有自动化修复 commit `7783b8af8`，其为同一失败 PR #4462 的修复）逐字节一致，`git diff master -- Bigdata/ranger/` 为空。

## 潜在风险
无。该修复是回退无效的自动升级并恢复主分支已确认的正确状态，不改变任何有效镜像（1.9.4）的构建，不影响其他镜像。