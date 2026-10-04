# 修复摘要

## 修复的问题
修复新增 rabitq-library 0.5.2 Dockerfile 中 `git checkout` 未带 `v` 前缀导致检出上游 tag 失败的问题。

## 修改的文件
- `Others/rabitq-library/0.5.2/24.03-lts-sp4/Dockerfile`: 将 `git checkout ${VERSION}` 改为 `git checkout v${VERSION}`，与同目录所有历史版本保持一致。

## 修复逻辑
分析报告因缺少 CI 日志判定为证据不足，并将「`git checkout 0.5.2` 目标 tag 不存在」列为低置信度候选方向 2。本修复通过客观证据确认了该根因：

1. **上游 tag 核对**：执行 `git ls-remote --tags https://github.com/VectorDB-NTU/RaBitQ-Library.git`，上游实际 tag 均为带 `v` 前缀的形式，其中 0.5.2 对应 `refs/tags/v0.5.2`（`d929e30dbb7ded52db1830736a2356fbc7dba99a`），**不存在**裸 `0.5.2` tag。
2. **仓库内模式比对**：`0.3.6`/`0.3.8`/`0.3.9`/`0.4.0` 及 `7c2d0d7` 的 Dockerfile 均使用 `git checkout v${VERSION}`；只有本次新增的 0.5.2 Dockerfile 写成了 `git checkout ${VERSION}`，属复制/修改时的遗漏。

因此 `git checkout 0.5.2`（完整克隆后 tag 名不匹配）会返回非 0 退出码，使该构建层失败（amd64/arm64 均会失败）。补回 `v` 前缀即可检出 `v0.5.2`，从根本上消除构建失败。

关于报告方向 1（缺少 Copyright/SPDX 头）：同一目录下已有的 `0.3.6`、`0.3.8`、`0.3.9`、`0.4.0` 等 Dockerfile 同样没有该头且已通过既有 CI，故未做改动，避免引入超出根因的变更。

## 潜在风险
无。改动仅为在版本号前补回与上游 tag 及同目录历史文件一致的 `v` 前缀，不改变其它任何构建步骤。