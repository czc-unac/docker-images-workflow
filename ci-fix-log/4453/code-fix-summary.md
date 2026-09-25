# 修复摘要

## 修复的问题
新增的 0.3.9 Dockerfile 在 `git checkout` 时未加 `v` 前缀，导致上游仓库中不存在 `0.3.9` ref，构建失败。

## 修改的文件
- `Others/rabitq-library/0.3.9/24.03-lts-sp4/Dockerfile`: 将 `git checkout ${VERSION}` 改为 `git checkout v${VERSION}`。

## 修复逻辑
CI 日志报错 `pathspec '0.3.9' did not match any file(s) known to git`。经 `git ls-remote --tags https://github.com/VectorDB-NTU/RaBitQ-Library.git` 验证，上游实际 tag 命名为 `v0.3.9`（对应 commit `f4be243ab7be4d79e56b0047622a9e0279c35de7`），即带 `v` 前缀。同目录历史版本 `0.3.6`、`0.3.8` 的 Dockerfile 均使用 `git checkout v${VERSION}`，新 PR 的 0.3.9 文件漏掉了 `v` 前缀。修复方向与分析报告"方向 1"一致：同步修正 checkout 目标为 `v0.3.9`，使 `git checkout` 能匹配到已发布 tag。已通过 `git ls-remote` 确认 `refs/tags/v0.3.9` 存在，checkout 目标可用。

## 潜在风险
无。改动仅恢复与历史版本一致的写法，README.md、doc/image-info.yml、meta.yml 中的版本号描述（0.3.9）保持不变，无需改动。