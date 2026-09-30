# 修复摘要

## 修复的问题
0.5.0 Dockerfile 中 `git checkout ${VERSION}`（即 `0.5.0`）无法匹配上游 tag，导致镜像构建失败；已改为使用 `v` 前缀。

## 修改的文件
- `Others/rabitq-library/0.5.0/24.03-lts-sp4/Dockerfile`: 将 `git checkout ${VERSION}` 改为 `git checkout v${VERSION}`

## 修复逻辑
CI 分析报告置信度为"低"且无日志，但方向 2 明确指出需验证上游 `VectorDB-NTU/RaBitQ-Library` 的 tag 命名（`0.5.0` vs `v0.5.0`）。通过 GitHub API 拉取上游 tag 列表（`https://api.github.com/repos/VectorDB-NTU/RaBitQ-Library/tags`）确认：仓库所有 tag 均带 `v` 前缀，最新为 `v0.5.0`，**不存在** `0.5.0` tag。原 Dockerfile 执行 `git checkout 0.5.0` 会报 `pathspec '0.5.0' did not match any file(s) known to git` 并退出非 0，导致 `RUN` 层构建失败。

同时确认仓库内历史版本 Dockerfile（0.3.6 / 0.3.8 / 0.3.9 / 0.4.0）全部使用 `git checkout v${VERSION}`，本次 0.5.0 是唯一漏掉 `v` 前缀的版本，属于自动升级脚本生成时的回归。修复为与历史版本一致即可。

另已验证 `v0.5.0` tag 下存在 `include` 目录（GitHub contents API 返回 `include` dir），因此 `cp -r include /usr/local/include/rabitq` 路径有效，无需其他改动。README.md、doc/image-info.yml、meta.yml 内容与历史版本格式一致，未发现需要同步修改之处。

## 潜在风险
无。改动仅恢复与历史版本一致的 tag 前缀，不改变任何镜像内容或路径。