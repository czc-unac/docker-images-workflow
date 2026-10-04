# 修复摘要

## 修复的问题
自动升级 PR 使用了不存在 tarball 的开发快照版本 `2.42.9000`，导致 Dockerfile 中 `wget glibc-${VERSION}.tar.xz` 下载失败；已将其改为上游实际发布的正式版本 `2.43`，并同步更新镜像 Tag 与文档。

## 修改的文件
- `Others/glibc/2.42.9000/24.03-lts-sp4/Dockerfile`: `ARG VERSION=2.42.9000` → `ARG VERSION=2.43`
- `Others/glibc/meta.yml`: 注册条目 `2.42.9000-oe2403sp4` → `2.43-oe2403sp4`（path 保持不变）
- `Others/glibc/README.md`: 镜像表首行 Tag `2.42.9000-oe2403sp4` → `2.43-oe2403sp4`，说明文字 `glibc 2.42.9000` → `glibc 2.43`
- `Others/glibc/doc/image-info.yml`: 同上，Tag 与说明文字同步为 `2.43-oe2403sp4` / `glibc 2.43`

## 修复逻辑
修复了分析报告"修复方向 1"指出的根因：自动升级选出的 `2.42.9000` 是 glibc 开发快照版本（`.9000` 后缀），GNU 官方镜像站不提供该版本的 tarball。

已通过网络实测验证（对应分析报告要求的"从上游以 Dockerfile `ARG VERSION` 为准实际验证版本存在性"）：
- `https://mirrors.tuna.tsinghua.edu.cn/gnu/glibc/glibc-2.42.9000.tar.xz` → HTTP 404（该版本不存在）
- `https://mirrors.tuna.tsinghua.edu.cn/gnu/glibc/glibc-2.43.tar.xz` → HTTP 200，`content-length: 20297012`（可正常下载）
- 另外核对 `https://ftp.gnu.org/gnu/glibc/` 目录，正式发布版本为 2.40 / 2.41 / 2.42 / 2.43 / 2.44，确认无任何 `.9000` 开发快照 tarball。

因此将 `VERSION` 指向当前最新可用的正式发布版本（`2.43`，紧邻仓库既有的 `2.42`，符合"自动升级"语义），并同步更新 `meta.yml` 中的镜像 Tag 及 `README.md` / `image-info.yml` 文档，保证 Tag、镜像内容描述、构建源三者一致。

说明：受"只允许修改 `pr.changed_files` 中已存在文件、不允许新增文件/目录"的约束，构建目录保留为 `2.42.9000/24.03-lts-sp4/`，仅修改其中的 Dockerfile 内容，未做目录重命名。

## 潜在风险
- 目录名仍为 `2.42.9000`，而实际构建的是 glibc 2.43，Tag/目录命名存在轻微不一致；不影响构建与镜像功能，仅为可读性问题。
- 已将版本回退定位到实际存在的正式发布版，未改动 glibc 构建依赖（与仓库中可正常构建的 `2.42`/`2.41` Dockerfile 依赖完全一致），故构建流程无额外风险。