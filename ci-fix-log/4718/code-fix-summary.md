# 修复摘要

## 修复的问题
修复 rabitq-library 0.4.0 Dockerfile 中 `git checkout 0.4.0` 因缺少 `v` 前缀而找不到上游 tag 导致的构建失败。

## 修改的文件
- `Others/rabitq-library/0.4.0/24.03-lts-sp4/Dockerfile`: 将第 10 行的 `git checkout ${VERSION}` 改为 `git checkout v${VERSION}`。

## 修复逻辑
CI 分析报告指出失败根因是 `git checkout 0.4.0` 报 `pathspec '0.4.0' did not match any file(s) known to git`。对照仓库既有约定，0.3.6 / 0.3.8 / 0.3.9 的 Dockerfile 均使用 `git checkout v${VERSION}`（上游 tag 带 `v` 前缀）。已通过 `git ls-remote --tags https://github.com/VectorDB-NTU/RaBitQ-Library.git` 从上游确认实际标签为 `refs/tags/v0.4.0`（commit `90943fd046b2114b69fa4f2aa342d2f2148791c0`），不存在无前缀的 `0.4.0`。因此最小化地将 0.4.0 Dockerfile 与旧版本对齐，加上 `v` 前缀，使 `git checkout v0.4.0` 能正确检出。

## 潜在风险
无。修改仅影响 0.4.0 版本的构建检出步骤，与旧版本约定一致；`ARG VERSION=0.4.0` 保持不变，镜像 tag `0.4.0-oe2403sp4` 及 README/meta.yml/image-info.yml 均无需改动。