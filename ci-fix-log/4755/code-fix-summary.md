# 修复摘要

## 修复的问题
自动升级单使用了上游不存在的版本号 `3.0`（keybinder 无 3.0 发布），导致 `wget` 下载 `keybinder-3.0-v3.0.tar.gz` 返回 404、x86-64 与 aarch64 构建均失败；已将其修正为上游实际存在的最新版本 `0.3.2`。

## 修改的文件
- `Others/binder/3.0/24.03-lts-sp4/Dockerfile`: `ARG VERSION=3.0` → `ARG VERSION=0.3.2`
- `Others/binder/meta.yml`: 条目 `3.0-oe2403sp4` → `0.3.2-oe2403sp4`（path 仍指向 `3.0/24.03-lts-sp4/Dockerfile`）
- `Others/binder/README.md`: 新增行 tag `3.0-oe2403sp4` → `0.3.2-oe2403sp4`，描述 `binder 3.0` → `binder 0.3.2`（链接路径保持不变）
- `Others/binder/doc/image-info.yml`: 与 README.md 同步修改

## 修复逻辑
1. **根因确认（实际日志）**：从 PR #4755 的 CI 评论表取得失败 job 链接，拉取真实构建日志：
   - x86-64 `check_build`（job #4866）与 aarch64 `check_build`（job #4962）均在 Dockerfile:18 失败：
     `wget .../keybinder-3.0-v3.0.tar.gz` → `HTTP request sent, awaiting response... 404 Not Found` → `exit code: 8`。
   - 即根因是分析报告"方向 2（上游 tag/下载 URL 不可用）"，且已被日志证实，并非基础设施问题。
2. **上游版本核对**：`git ls-remote --tags https://github.com/kupferlauncher/keybinder.git` 与 GitHub Releases API 确认，`keybinder-3.0` 系列仅存在 `keybinder-3.0-v0.3.0 / v0.3.1 / v0.3.2`，不存在 `keybinder-3.0-v3.0`。自动升级把包名后缀 `keybinder-3.0` 误当作版本号，实际最新发布版本为 `0.3.2`。
3. **验证修正后的构建链路**：
   - `https://github.com/kupferlauncher/keybinder/archive/refs/tags/keybinder-3.0-v0.3.2.tar.gz` 返回 HTTP 200；
   - GitHub 归档解压目录为 `keybinder-keybinder-3.0-v0.3.2`，与 `WORKDIR /opt/keybinder-keybinder-3.0-v${VERSION}`（VERSION=0.3.2）完全一致；
   - 已下载该 tarball，对 `Fix-gtk-doc-build-failure.patch` 执行 `patch -p1 --dry-run`，结果 `checking file docs/keybinder-docs.sgml` 通过（exit 0）；
   - 上游 `configure.ac` 中 `AC_INIT(keybinder-3.0, 0.3.2, ...)`，与目标版本一致。
4. 因本仓库既有同类修复（如 QEMU `11.0.2→11.0.1`、alluxio `2.9.6→2.9.5`、redis `5.4.1→8.6.4`）均同步更新 Dockerfile 与三个元数据文件（tag/描述），本次沿用该约定；目录名 `3.0/` 保持不变（约束不允许新增/重命名文件），仅修正其中的版本号引用。

## 潜在风险
- `meta.yml` 中会出现两个 `0.3.2-oe2403sp4` 键（分别指向 `3.0/...` 与已存在的 `0.3.2/...`），README/image-info 也会出现两条同名 tag 行。这与历史修复（如 libvirt `12.7.0-oe2403sp4` 重复条目）一致，YAML 解析与 CI 构建均可通过；但由于两者构建内容相同，属于冗余条目。
- 目录名 `3.0/` 与其中的 `VERSION=0.3.2` 语义不一致，属自动升级流程遗留问题，可在后续清理 PR 中将目录重命名为 `0.3.2/` 彻底消除；不影响本次构建与最终镜像产物。