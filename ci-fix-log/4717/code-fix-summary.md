# 修复摘要

## 修复的问题
QEMU 11.1.2 官方源码包尚未在 `download.qemu.org` 发布，下载返回 HTTP 404 导致镜像构建失败；将构建版本回退到当前确认可下载的 11.1.1。

## 修改的文件
- `Cloud/qemu/11.1.2/24.03-lts-sp4/Dockerfile`: 将 `ARG VERSION=11.1.2` 改为 `ARG VERSION=11.1.1`，使 `wget https://download.qemu.org/qemu-${VERSION}.tar.xz` 命中实际存在的源码包。

## 修复逻辑
对应分析报告的根因「下载 URL 硬编码版本路径错误 / 软件包版本不存在（模式02）」。

已联网核实（不涉及正则 patch 外部源文件，故无需正则验证）：
- `https://download.qemu.org/qemu-11.1.2.tar.xz` → **HTTP 404**（构建日志中亦为此错误，wget exit code 8）。
- `https://download.qemu.org/qemu-11.1.1.tar.xz` → **HTTP 200**，`Content-Length: 141888716`，`Content-Type: application/x-tar`，是真实可用的发布制品。
- 上游 GitHub tag `v11.1.2` 确实已存在（tagger date `2026-09-28`），但 `download.qemu.org` 目录索引当前最新仅到 `qemu-11.1.1.tar.xz`，说明官方发布 tarball 尚未同步。
- 下载文件名与解压命令格式保持匹配：仍为 `.tar.xz` + `tar xvJf`，无需改动。

因此将版本回退到分析报告建议的、确认可下载的 `11.1.1`，即可让 `check_build`（x86_64/aarch64）通过。

## 潜在风险
- 保留目录/tag 名 `11.1.2`，因此构建产物 `qemu:11.1.2-oe2403sp4` 实际内含 qemu 11.1.1；`README.md`、`meta.yml`、`doc/image-info.yml` 仍登记 11.1.2 条目。这是为保持改动最小、避免在 `meta.yml` 中产生重复键（`11.1.1-oe2403sp4` 已存在）而做的临时回退处理。
- 待上游 `download.qemu.org` 正式发布 `qemu-11.1.2.tar.xz` 后，应将 `ARG VERSION` 恢复为 `11.1.2`。