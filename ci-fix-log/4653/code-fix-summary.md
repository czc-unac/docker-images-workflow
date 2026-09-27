# 修复摘要

## 修复的问题
将 blat 新版本 Dockerfile 中无效的克隆 ref `2.5.1` 修正为上游 `icebert/pblat-cluster` 实际存在的最新 tag `1.1`，并修正 `image-info.yml` 中被写错的上游仓库地址，解决 `git clone` 因「Remote branch 2.5.1 not found」导致的构建失败。

## 修改的文件
- `HPC/blat/2.5.1/24.03-lts-sp4/Dockerfile`: `ARG VERSION=2.5.1` → `ARG VERSION=1.1`（该值用于第 9 行 `git clone -b ${VERSION} https://github.com/icebert/pblat-cluster.git`）。
- `HPC/blat/doc/image-info.yml`: `upstream.version_url: icebert/blat-cluster` → `icebert/pblat-cluster`（此仓库此前被误写为不存在的 `icebert/blat-cluster`）。

## 修复逻辑
1. **直接根因（构建失败）**：Dockerfile 从 `icebert/pblat-cluster` 克隆，并以其 tag 作为 `${VERSION}`。经 `git ls-remote https://github.com/icebert/pblat-cluster.git` 验证，上游仓库的 tag 只有 `0.1 / 0.2 / 0.3 / 1.0 / 1.1`，**不存在 `2.5.1`**，因此 `git clone -b 2.5.1` 必然以 exit code 128 失败。将其改为上游最新可用 tag `1.1` 后，`clone` 可成功，其后所有构建步骤与已通过 CI 的 `HPC/blat/1.1/24.03-lts-sp4/Dockerfile` 完全一致，可直接复用其验证结果。对应分析报告「方向 1（高）」与「方向 2：改为上游最新可用版本」。

2. **源头根因（防止复发）**：`git log -p --follow HPC/blat/doc/image-info.yml` 显示，提交 `7ad36ce09`（Just blat）将 `version_url` 由 `icebert/pblat-cluster` 误改为不存在的 `icebert/blat-cluster`。经 `https://api.github.com/repos/icebert/blat-cluster` 验证该仓库返回 404。这使自动升级工具在解析 `blat` 时落到另一个项目 `icebert/pblat`（其最新 tag 恰为 `2.5.1`），从而生成了本次错误的升级 PR。将其恢复为 Dockerfile 实际克隆的 `icebert/pblat-cluster` 后，自动升级源与构建源一致，避免再次生成不存在的版本。

**上游验证**：已通过 `git ls-remote`/GitHub API 确认 `icebert/pblat-cluster` 存在 tag `1.1`（Dockerfile 使用的 ref），并确认 `icebert/blat-cluster` 不存在。本次修复不涉及正则 patch 第三方源文件，无需正则匹配验证。

## 潜在风险
- 新镜像 tag 仍为 `2.5.1-oe2403sp4`（目录/meta.yml/README 未改，遵循最小化原则），但其内容实际为 `pblat-cluster 1.1`，与既有 `1.1-oe2403sp4` 相同；这是自动升级 PR 的既有命名，未在本修复中扩大改动范围。若维护者希望彻底消除该重复版本，应另行关闭/回退该升级 PR 并清理 `HPC/blat/2.5.1/` 目录、`meta.yml`、`README.md`、`image-info.yml` 中的 2.5.1 条目。