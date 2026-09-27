# 修复摘要

## 修复的问题
上游仓库 RaBitQ-Library 的发布 tag 带 `v` 前缀（`v0.4.0`），而 Dockerfile 使用了 `git checkout 0.4.0`，导致 pathspec 匹配失败、容器构建中断。

## 修改的文件
- `Others/rabitq-library/0.4.0/24.03-lts-sp4/Dockerfile`: 将 `git checkout ${VERSION}` 改为 `git checkout v${VERSION}`。

## 修复逻辑
分析报告根因指出 `git checkout 0.4.0` 无法解析到上游 ref。通过 `git ls-remote --tags https://github.com/VectorDB-NTU/RaBitQ-Library.git` 验证，上游实际存在的 tag 为 `v0.4.0`（同时存在 v0.1.1 ~ v0.3.9、v0.4.0），即 tag 统一带 `v` 前缀。本仓库已有的 `0.3.6`、`0.3.8` 条目均使用 `git checkout v${VERSION}` 的写法，说明这是既有约定。因此将新条目对齐为 `git checkout v${VERSION}`，与上游实际 ref 及仓库既有写法一致。

验证结果：已从上游 `v0.4.0` 实际 clone 验证（`git clone --depth 1 --branch v0.4.0`），tag 存在且包含 `include/rabitqlib` 目录，`cp -r include ...` 步骤可正常执行。

## 潜在风险
无。修改仅影响该新增条目的 ref 解析，与仓库内其他版本条目互不影响。