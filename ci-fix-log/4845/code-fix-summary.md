# 修复摘要

## 修复的问题
新增的 rabitq-library 0.5.1 Dockerfile 在 `git checkout` 时缺少上游 tag 的 `v` 前缀，导致 `git checkout 0.5.1` 找不到对应 tag，镜像构建失败。

## 修改的文件
- `Others/rabitq-library/0.5.1/24.03-lts-sp4/Dockerfile`: 将 `git checkout ${VERSION}` 修正为 `git checkout v${VERSION}`。

## 修复逻辑
CI 分析报告因 `ci.logs` 缺失，将失败归类为"证据不足"，但同时指出疑似模式 02 / 模式 28（`git checkout <tag>` 目标 tag 不存在）。

经核查上游仓库 `VectorDB-NTU/RaBitQ-Library` 的实际 tag 列表（`git ls-remote --tags`），确认所有版本 tag 均带 `v` 前缀：
- `refs/tags/v0.5.1`（对应本次 `ARG VERSION=0.5.1`）
- `refs/tags/v0.4.0`、`refs/tags/v0.3.9`、`refs/tags/v0.3.6` ...

并无不带前缀的 `0.5.1` tag。同目录历史版本的 Dockerfile（0.3.6 / 0.3.8 / 0.3.9 / 0.4.0）均使用 `git checkout v${VERSION}`，仅 `7c2d0d7`（commit hash 版本）使用 `${VERSION}`，符合预期。因此本次新增的 0.5.1 Dockerfile 属回归性遗漏 `v` 前缀。

已从上游仓库验证：`refs/tags/v0.5.1` 存在（commit 44f8a607741e9e54f6dda3b38c325071e808aa45），修复后 `git checkout v0.5.1` 可正确检出。README.md、image-info.yml、meta.yml 中新增的 0.5.1 条目与目录结构、版本号一致，无需改动。

## 潜在风险
无。改动仅为恢复与历史版本一致的 tag 检出写法，不影响其他镜像或文件。