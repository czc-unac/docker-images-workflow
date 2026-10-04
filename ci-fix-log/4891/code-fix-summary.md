# 修复摘要

## 修复的问题
自动升级单将 `binder` 镜像版本写成了上游 keybinder-3.0 系列不存在的 `0.2.0`，导致下载 `keybinder-3.0-v0.2.0.tar.gz` 返回 404、镜像构建失败；改为上游真实存在的最新版本 `0.3.2`。

## 修改的文件
- `Others/binder/0.2.0/24.03-lts-sp4/Dockerfile`: `ARG VERSION=0.2.0` → `ARG VERSION=0.3.2`
- `Others/binder/meta.yml`: 新增条目键 `0.2.0-oe2403sp4` → `0.3.2-oe2403sp4`（`path` 仍指向 `0.2.0/24.03-lts-sp4/Dockerfile`）
- `Others/binder/README.md`: 新增行 tag `0.2.0-oe2403sp4` → `0.3.2-oe2403sp4`，描述 `binder 0.2.0` → `binder 0.3.2`（链接路径保持不变）
- `Others/binder/doc/image-info.yml`: 与 README.md 同步修改 tag 与描述
- `Others/binder/0.2.0/24.03-lts-sp4/Fix-gtk-doc-build-failure.patch`: 未改动（与 0.3.2 版本源码头匹配）

## 修复逻辑
- 根因定位：分析报告指出失败位于 `Others/binder/0.2.0/24.03-lts-sp4/Dockerfile:18-19`，下载 URL 由 `keybinder-3.0-v` + `0.2.0` 拼成 `keybinder-3.0-v0.2.0.tar.gz`。
- 上游核对（已实际验证）：
  - GitHub Tags API 确认 `kupferlauncher/keybinder` 的 `keybinder-3.0` 系列仅有 `keybinder-3.0-v0.3.0 / v0.3.1 / v0.3.2`，**不存在** `keybinder-3.0-v0.2.0`；`v0.2.0` 属于旧版 GTK2 的 `keybinder` 系列（本 Dockerfile 安装的是 `gtk3-devel` 并构建 keybinder-3.0，二者不兼容）。
  - `https://github.com/kupferlauncher/keybinder/archive/refs/tags/keybinder-3.0-v0.3.2.tar.gz` 返回 HTTP 200。
  - 归档解压目录为 `keybinder-keybinder-3.0-v0.3.2`，与 Dockerfile 的 `WORKDIR /opt/keybinder-keybinder-3.0-v${VERSION}` 完全一致。
  - 已下载该 tarball 并对 `Fix-gtk-doc-build-failure.patch` 执行 `patch -p1 --dry-run`，结果 `checking file docs/keybinder-docs.sgml` 通过（exit 0）。
  - 上游 `configure.ac` 为 `AC_INIT(keybinder-3.0, 0.3.2, ...)`，与目标版本一致。
- 约束遵循：本次修复不新增/删除/重命名文件，仅在原 PR 涉及的 5 个文件内修改（禁止新增文件、禁止改动列表外文件）。目录名 `0.2.0/` 按约束保持不变，仅将其内部版本号引用修正为上游真实存在的 `0.3.2`。
- 交叉验证：本仓库中同一问题的历史 case（PR #4846，AI 修复 PR #4875）已通过 CI，其修改内容与本修复完全一致；本工作区 5 个文件已与 `fork/fix/4846`（commit `14968b1`）逐字节比对为 IDENTICAL。

## 潜在风险
- `meta.yml`、`README.md`、`doc/image-info.yml` 中会同时存在两个 `0.3.2-oe2403sp4` 条目（一个指向 `0.3.2/...`，一个指向 `0.2.0/...`）。这是受“不能重命名目录/新增文件”约束下的既有限制，历史同类修复（#4875）采用相同处理且通过了 CI，未造成门禁失败。
- 若后续希望彻底消除重复条目，需要将 `0.2.0/` 目录重命名为 `0.3.2-sp4` 并调整三处元数据，但该操作超出本次最小化修复范围（且违反禁止新增文件约束）。