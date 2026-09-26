# 修复摘要

## 修复的问题
Alluxio 2.9.6 官方未发布二进制包，Dockerfile 下载地址 404，将构建版本修正为上游实际可用的最新稳定版 2.9.5。

## 修改的文件
- `Storage/alluxio/2.9.6/24.03-lts-sp4/Dockerfile`: `ARG VERSION` 由 `2.9.6` 改为 `2.9.5`，使下载 URL 指向真实存在的制品。
- `Storage/alluxio/meta.yml`: 镜像 tag `2.9.6-oe2403sp4` 改为 `2.9.5-oe2403sp4`（path 保持不变）。
- `Storage/alluxio/README.md`: 支持列表中的 tag 与 alluxio 版本描述由 `2.9.6` 改为 `2.9.5`。
- `Storage/alluxio/doc/image-info.yml`: tags 表中的 tag 与版本描述由 `2.9.6` 改为 `2.9.5`。

## 修复逻辑
- 分析报告根因定位为 `Storage/alluxio/2.9.6/24.03-lts-sp4/Dockerfile:16`：`VERSION=2.9.6` 展开后的下载地址 `https://downloads.alluxio.io/downloads/files/2.9.6/alluxio-2.9.6-bin.tar.gz` 返回 HTTP 404，`curl -fSL` 以 exit 22 退出。
- 上游核实结论：
  - 下载站目录 `https://downloads.alluxio.io/downloads/files/` 及 2.9.x 列表实际只到 `2.9.5`；`2.9.6` 目录与 `alluxio-2.9.6-bin.tar.gz` 均不存在（HTTP 404）。
  - GitHub 上仅存在 git tag `v2.9.6`，但没有对应 Release / 二进制制品；Maven 有 `alluxio-core:2.9.6`，但无对应的 bin 发行包。
  - 因此 2.9.6 只被打了 tag、未发布可下载的二进制，原 PR 的升级目标无制品可用。
- 采用分析报告“方向 1”的修复：将 `VERSION` 改为上游实际可用最新稳定版 `2.9.5`，并同步更新 README.md、doc/image-info.yml、meta.yml 中的版本描述。
- **URL 实际验证**：使用与 Dockerfile 完全一致展开后的地址执行 `curl -I`：
  - `https://downloads.alluxio.io/downloads/files/2.9.5/alluxio-2.9.5-bin.tar.gz` → **HTTP 200**
  - `https://downloads.alluxio.io/downloads/files/2.9.6/alluxio-2.9.6-bin.tar.gz` → HTTP 404（确认原版本无制品）
- **目录未重命名的说明**：本工作流的 code-fix 阶段只会暂存“原始 PR changed_files”中列出的文件（`scripts/stages/code-fix.py:196-208`），任何新建/移动的文件都会被移出暂存区。因此无法将物理目录 `2.9.6/24.03-lts-sp4/` 迁移为 `2.9.5/24.03-lts-sp4/`（否则会只删除旧文件而不添加新文件，导致仓库损坏）。据此保留物理路径 `2.9.6/`，仅修正用户可见的版本号/tag。
- 已确认 conf/、entrypoint.sh、libexec/ 内的辅助文件与 2.9.4 版本完全一致，不含版本号字符串，无需修改。

## 潜在风险
- 物理目录名仍为 `2.9.6/`，而 tag 与 `VERSION` 为 `2.9.5`，README/image-info 中的 Dockerfile 链接路径仍指向 `2.9.6/` 目录。因工作流限制无法重命名目录，属已知且有意保留的折中；对该镜像的构建与发布（tag=`2.9.5-oe2403sp4`、内容 alluxio 2.9.5）无功能影响。
- 2.9.5 与 2.9.6 之间的上游差异未做验证；但 2.9.5 为官方下载站可用的最新 2.9.x 稳定版，是当前唯一可行目标。